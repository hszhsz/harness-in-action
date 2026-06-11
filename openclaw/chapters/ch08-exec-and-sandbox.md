# 第八章 exec 与沙箱：最危险的工具如何被关进笼子

上一章我们把工具系统拆成了三层：描述符内核负责声明、策略管线负责裁决、`beforeToolCall` 闸门负责拦截。那一章的结尾我们留下了一个伏笔——`SandboxToolPolicy` 这个反复出现的类型名,到底对应着怎样一套真实的隔离边界?这一章就来兑现这个承诺。

我们要把视线从「工具怎么被管理」彻底转向「工具怎么被执行」。而要谈执行,就绕不开 `exec` 这个工具——它能在宿主机上跑任意 shell 命令,是整个 Agent 工具箱里**最强大、也最危险**的一个。一个能写文件的工具最多破坏几个文件,一个能跑 `exec` 的工具可以 `rm -rf /`、可以读走你的 SSH 私钥、可以把内网当跳板。所以 OpenClaw 在 `exec` 周围堆了一整套防御工事:目标解析、安全等级、审批策略、环境变量净化、脚本预检,以及最重的那一层——把命令丢进 Docker 沙箱里跑。

这一章我们会沿着一条 `exec` 调用的生命周期往下走,看每一道关卡是怎么设计的,以及它们为什么以现在这种方式存在。

## 8.1 exec 工具的全貌:一次调用要穿过多少道门

`exec` 工具的实现集中在 `src/agents/bash-tools.exec.ts` 里,光这一个文件就有近两千行。但它的骨架其实清晰:`createExecTool` 是一个工厂函数,接受一组 `ExecToolDefaults`(运行时默认值、审批策略、沙箱句柄等),返回一个带有 `execute` 方法的工具对象。

```ts
export function createExecTool(
  defaults?: ExecToolDefaults,
): AgentToolWithMeta<typeof execSchema, ExecToolDetails> {
  // ...解析 backgroundMs、timeoutSec、safeBins、elevated 等默认值...
  return {
    name: "exec",
    label: "exec",
    parameters: execSchema,
    execute: async (_toolCallId, args, signal, onUpdate) => {
      // ...一长串关卡...
    },
  };
}
```

工具的入参类型 `ExecToolArgs` 揭示了 `exec` 的表达力:

```ts
type ExecToolArgs = Record<string, unknown> & {
  command: string;
  workdir?: string;
  env?: Record<string, string>;
  yieldMs?: number;
  background?: boolean;
  timeout?: number;
  pty?: boolean;
  elevated?: boolean;
  host?: string;
  security?: string;
  ask?: string;
  node?: string;
};
```

注意这里有三个看起来很「危险」的参数:`host`(在哪台机器上跑)、`security`(以什么安全等级跑)、`ask`(要不要先问人)。模型(也就是 LLM)在调用工具时是可以填这些参数的。一个自然的担忧立刻浮现:**如果模型自己把 `security` 填成 `full`、`ask` 填成 `off`,岂不是可以绕过所有审批?**

这正是本章要反复回答的核心问题。OpenClaw 的答案是一个贯穿始终的原则:**模型的请求只能让执行变得更严格,永远不能让它变得更宽松**。我们后面会看到这个原则在代码里是怎么落地的。

先把 `execute` 的主流程拎出来。一次 `exec` 调用大致要依次穿过这些门:

1. **目标解析**——决定命令最终在哪里执行(`gateway` 宿主机 / `node` 远程节点 / `sandbox` 容器)。
2. **安全等级与审批策略求交**——把配置侧的策略和模型请求的参数合并,取最严的那个。
3. **危险控制命令拦截**——拒绝 `exec` 去跑 `/approve`、`openclaw channels login` 这类会破坏控制流的命令。
4. **环境变量净化**——剥掉 `PATH`、危险变量,防止通过环境变量做攻击。
5. **脚本预检**——在真正执行前,检查 Python/JS 脚本里有没有 shell 语法泄漏。
6. **进程启动**——根据目标走宿主机执行、远程节点执行,或沙箱执行。
7. **前台 / 后台移交**——短命令同步返回结果,长命令在 `yieldMs` 后转入后台会话。

这一章我们重点看 1、2、3、4、5 和沙箱执行。

## 8.2 目标解析:host 参数为什么不能想填就填

`exec` 的第一道门是「这条命令到底在哪里跑」。这由 `bash-tools.exec-runtime.ts` 里的 `resolveExecTarget` 决定:

```ts
export function resolveExecTarget(params: {
  configuredTarget?: ExecTarget;
  requestedTarget?: ExecTarget | null;
  elevatedRequested: boolean;
  sandboxAvailable: boolean;
}) {
  const configuredTarget = params.configuredTarget ?? "auto";
  const requestedTarget = params.requestedTarget ?? null;
  if (
    requestedTarget &&
    !isRequestedExecTargetAllowed({
      configuredTarget,
      requestedTarget,
      sandboxAvailable: params.sandboxAvailable,
    })
  ) {
    // ...抛出 "exec host not allowed" 错误...
  }
  const selectedTarget = requestedTarget ?? configuredTarget;
  const resolvedTarget = params.elevatedRequested
    ? selectedTarget === "node"
      ? "node"
      : "gateway"
    : selectedTarget;
  const effectiveHost =
    resolvedTarget === "auto" ? (params.sandboxAvailable ? "sandbox" : "gateway") : resolvedTarget;
  return { configuredTarget, requestedTarget, selectedTarget: resolvedTarget, effectiveHost };
}
```

这段逻辑里藏着几个关键设计:

**第一,模型请求的 host 要先经过 `isRequestedExecTargetAllowed` 校验。** 配置侧通过 `tools.exec.host` 声明了「允许在哪里跑」。如果运维把 host 锁死在 `sandbox`,而模型却请求 `host=gateway`(想跳出沙箱去宿主机跑),校验会失败,直接抛出一个相当啰嗦但很有用的错误:

> `exec host not allowed (requested gateway; configured host is sandbox; set tools.exec.host=... to allow this override).`

这个错误信息不只是说「不行」,还告诉运维要改哪个配置键才能放行。这是 OpenClaw 一贯的「错误即文档」风格——把修复方法直接写进异常里。

**第二,`auto` 的语义是「有沙箱就用沙箱」。** 当 `effectiveHost` 计算到 `resolvedTarget === "auto"` 时,只要这个会话有可用的沙箱运行时(`sandboxAvailable`),就解析成 `sandbox`,否则才退回 `gateway`。换句话说,**沙箱是默认的、安全的选择,跳出沙箱才需要显式声明**。这是一个典型的「安全默认值」(secure by default)设计。

**第三,提权(elevated)会强制把目标拉回 `gateway` 或 `node`。** 在沙箱里 `sudo` 没有意义——沙箱本来就是用来限制权限的。所以一旦请求了提权,目标就不可能是 `sandbox`。这条规则把「提权」和「隔离」这两个互斥的概念在类型层面就分开了。

`createExecTool` 里对这三个目标分别有不同的执行路径:`host === "node"` 走 `executeNodeHostCommand`(远程节点),`host === "gateway"` 走 `processGatewayAllowlist` + 本地进程,`host === "sandbox"` 则把命令交给沙箱句柄。我们这一章主要看 gateway 和 sandbox 这两条线。

## 8.3 安全地板:模型只能加锁,不能开锁

确定了在哪里跑,下一道门是「以什么安全等级跑」。这里的代码不长,但它是整个 `exec` 安全模型的灵魂,值得逐行读。

OpenClaw 定义了三个安全等级,在 `infra/exec-approvals.js` 里:`deny`(拒绝执行)、`allowlist`(只允许白名单内的命令)、`full`(放行任意命令)。还有一个 `ask` 维度:`off`(不问)、`on-miss`(白名单未命中时才问)、`always`(每次都问)。

关键在于这些值是从**多个来源**汇集的:配置文件 `tools.exec.security/ask`、独立的审批文件 `exec-approvals.json`、以及模型在工具调用里填的 `security/ask` 参数。它们怎么合并?看 `createExecTool` 里这几行:

```ts
const modePolicy = resolveExecModePolicy({
  mode: defaults?.mode,
  security: configuredSecurity,
  ask: defaults?.ask ?? "off",
});
// ...
let security = minSecurity(
  modePolicy.security,
  approvalPolicy?.security ?? modePolicy.security,
);
```

注意这个 `minSecurity`——它取的是**更严的**那个安全等级(`deny` < `allowlist` < `full`,取最小即最严)。配置说 `full`、审批文件说 `deny`,结果就是 `deny`。再看 `ask`:

```ts
let ask = maxAsk(hostAsk, trustedAsk ?? hostAsk);
```

`maxAsk` 取的是**更频繁询问**的那个(`off` < `on-miss` < `always`,取最大即最常问)。配置说 `off`、模型说 `always`,结果是 `always`。

把这两个函数放在一起看,设计意图就一目了然了:**安全等级取最严,询问频率取最高**。无论模型怎么填参数,它都只能让执行变得更受约束。

这套机制在测试文件 `bash-tools.exec.security-floor.test.ts` 里被一个一个验证过。这些测试的名字本身就是规格说明书:

- `"ignores model-supplied allowlist security when configured security is full"`——配置是 `full`,模型填 `allowlist`,结果仍按 `full` 执行(模型想收紧到白名单,但这里 `full` 已经是配置定的地板,测试验证的是模型的 `allowlist` 不会意外改变 `full` 的放行行为)。
- `"enforces configured allowlist security when model also passes allowlist"`——配置是 `allowlist`,命令不在白名单里,即使模型也填 `allowlist`,照样抛 `exec denied: allowlist miss`。
- `"ignores model-supplied full security when configured security is deny"`——配置是 `deny`,模型填 `full` 想放行,结果抛 `exec denied`。
- `"ignores model-supplied ask overrides when configured ask is off"`——这个有意思:当存在 `messageProvider`(也就是命令来自某个聊天渠道)且配置的 `ask=off` 时,模型填 `ask=always` 会被忽略。

最后这条揭示了一个微妙之处。看 `createExecTool` 里的代码:

```ts
const trustedAsk = defaults?.messageProvider && hostAsk === "off" ? undefined : requestedAsk;
```

为什么来自聊天渠道时要忽略模型的 `ask=always`?因为对于来自渠道的请求,运维已经通过配置明确表态「这条链路信任、不必每次问」。如果允许模型自己把 `ask` 拉高到 `always`,模型就能利用审批流程做一些奇怪的事(比如反复触发审批通知来骚扰用户)。所以在「可信渠道 + 配置不问」的组合下,模型对 `ask` 的请求被丢弃。

但反过来——`"honors per-call ask hardening for trusted callers without messageProvider"`——如果**没有** `messageProvider`(请求不来自渠道,而是来自更底层的可信调用方),模型填 `ask=always` 是被尊重的,这时审批会真的触发。差别就在那个 `messageProvider` 条件上。

这种「同一个参数,在不同信任上下文里有不同待遇」的设计,正是真实安全系统的复杂之处:它不是非黑即白的开关,而是一张根据来源、配置、请求三方协商出来的策略表。

## 8.4 控制命令拦截:exec 不能用来攻击 exec 自己

确定了安全等级,下一道门是 `rejectUnsafeControlShellCommand`。它解决的是一类特殊威胁:**用 `exec` 去运行那些会扰乱 OpenClaw 自身控制流的命令**。

```ts
function rejectUnsafeControlShellCommand(command: string): void {
  const rawCommand = command.trim();
  const analysis = analyzeShellCommand({ command: rawCommand });
  const candidates = analysis.ok
    ? analysis.segments.flatMap((segment) => buildCommandPayloadCandidates(segment.argv))
    : /* 退化为按行拆分 */;
  for (const candidate of candidates) {
    if (parseExecApprovalShellCommand(candidate)) {
      throw new Error("exec cannot run /approve commands. ...");
    }
    if (parseOpenClawChannelsLoginShellCommand(candidate)) {
      throw new Error("exec cannot run interactive OpenClaw channel login commands. ...");
    }
  }
}
```

它拦截两类命令:

**第一类是 `/approve`。** 回想上一章的审批闸门——审批是「人在回路」的最后一道保险。如果模型能通过 `exec` 跑 `/approve <id> allow-always`,它就能**给自己批准审批请求**,整套人工审批机制瞬间形同虚设。所以 `exec` 必须明确拒绝运行 `/approve`。错误信息也给出了正确做法:「把 `/approve` 命令作为聊天文本展示给用户,或者走审批命令处理器,而不是走 shell 执行」。

注意 `parseExecApprovalShellCommand` 用的正则相当宽松:

```ts
/^\/approve(?:@[^\s]+)?\s+([A-Za-z0-9][A-Za-z0-9._:-]*)\s+(allow-once|allow-always|always|deny)\b/i
```

它能识别 `/approve@botname`、各种大小写、`allow-once`/`allow-always`/`always`/`deny` 等所有变体。这种「宁可拦宽一点」的态度在安全代码里是对的——拦错了顶多是少跑一条命令,漏拦了就是审批被绕过。

**第二类是 `openclaw channels login`。** 这是个交互式登录命令,会要求输入凭据。如果让 `exec` 去跑,要么卡住(等不到交互输入),要么可能把凭据流程暴露在不该暴露的上下文里。`parseOpenClawChannelsLoginShellCommand` 还很贴心地处理了各种包管理器前缀——`pnpm openclaw`、`npx openclaw`、`pnpm exec openclaw` 都能被识破:

```ts
function stripOpenClawPackageRunner(argv: string[]): string[] {
  const commandName = normalizeCommandBaseName(argv[0]);
  if (commandName === "openclaw") return argv;
  if ((commandName === "pnpm" || ...) && normalizeCommandBaseName(argv[1]) === "openclaw") {
    return argv.slice(1);
  }
  // ...处理 npx/bunx 及其 -p/--package 选项...
}
```

这种对「同一个命令的 N 种写法」的穷举,是 shell 命令安全检查里最磨人也最必要的活儿——攻击者总会尝试用各种等价写法绕过简单的字符串匹配。

## 8.5 环境变量净化:别让 PATH 成为后门

命令本身检查完了,还有一个常被忽视的攻击面:**环境变量**。一条 `echo hello` 看起来人畜无害,但如果攻击者能同时塞一个 `PATH=/tmp/evil:$PATH`,那么 `echo` 解析到的可能是 `/tmp/evil/echo`——一个恶意替身。

`createExecTool` 在宿主机执行路径上调用 `sanitizeHostExecEnvWithDiagnostics` 来净化环境:

```ts
const hostEnvResult =
  host === "sandbox"
    ? null
    : sanitizeHostExecEnvWithDiagnostics({
        baseEnv: inheritedBaseEnv,
        overrides: requestedEnv,
        blockPathOverrides: true,
      });
```

`blockPathOverrides: true` 这个参数就是专门挡 `PATH` 的。如果模型或插件试图覆盖 `PATH`,会被拒绝,并抛出一个措辞严厉的错误:

```ts
if (pathBlocked && blockedKeys.length === 1 && invalidKeys.length === 0) {
  throw new Error(
    "Security Violation: Custom 'PATH' variable is forbidden during host execution.",
  );
}
```

除了 `PATH`,代码还区分了两类被拒的键:`rejectedOverrideBlockedKeys`(危险键,如 `LD_PRELOAD` 之类能劫持动态链接的变量)和 `rejectedOverrideInvalidKeys`(非法/不可移植的键名)。错误信息会把这两类分别列出来,方便排查。

值得注意的是,这套净化**只在宿主机执行时生效**(`host === "sandbox"` 时直接 `null`)。为什么沙箱里就不管 `PATH` 了?因为沙箱本身就是隔离的——即便 `PATH` 被改坏,影响也被关在容器里,出不来。沙箱场景下走的是另一套环境构造逻辑 `buildSandboxEnv`,它从一个干净的 `DEFAULT_PATH` 起步,而不是继承宿主机环境。这又是一个「隔离边界决定防护强度」的例子:边界越强,边界内部就可以越宽松。

来自插件的环境变量还要额外过一道 `filterPluginExecEnv`:

```ts
function filterPluginExecEnv(rawEnv: Record<string, string>): Record<string, string> | undefined {
  const env: Record<string, string> = {};
  for (const [rawKey, value] of Object.entries(rawEnv)) {
    const key = normalizeHostOverrideEnvVarKey(rawKey);
    if (!key) continue;
    const upperKey = key.toUpperCase();
    if (
      upperKey === "PATH" ||
      upperKey === OPENCLAW_CLI_ENV_VAR ||
      isDangerousHostEnvVarName(upperKey) ||
      isDangerousHostEnvOverrideVarName(upperKey)
    ) {
      continue;
    }
    env[key] = value;
  }
  return Object.keys(env).length > 0 ? env : undefined;
}
```

插件通过 `resolve_exec_env` 钩子可以为命令注入环境变量(比如临时凭据),但 `PATH`、OpenClaw 自身的 CLI 标识变量、以及所有危险变量名都会被无情过滤掉。插件能加东西,但不能加危险的东西。

## 8.6 脚本预检:防止 shell 语法漏进 Python/JS

这一节要讲的关卡,初看会觉得「这也要管?」,但读懂之后会发现它解决的是一个真实、高频、且代价不菲的失败模式。

场景是这样的:模型经常会先写一个 Python 脚本文件,再用 `exec` 跑 `python script.py`。但 LLM 有一个顽固的坏习惯——它会把 shell 的语法不小心写进 Python 代码里。最典型的就是把 shell 的环境变量引用 `$HOME`、`$API_KEY` 直接写进 Python 源码,而 Python 根本不认这种语法,会原样当成字符串或者直接报错。

如果这种错误发生在一个 cron 定时任务的循环里,后果是:每次循环都跑一个注定失败的命令,每次都消耗 token,无声无息地烧钱。`validateScriptFileForShellBleed` 就是为了在执行**之前**把这类错误拦下来:

```ts
async function validateScriptFileForShellBleed(params: {
  command: string;
  workdir: string;
}): Promise<void> {
  const target = extractScriptTargetFromCommand(params.command);
  // ...如果识别出 `python <file>.py` 或 `node <file>.js`...
  // 读取脚本文件内容(限制在 workdir 内、512KB 以内、非阻塞打开)
  // 检测 shell 变量注入
  const envVarRegex = /\$[A-Z_][A-Z0-9_]{1,}/g;
  const first = envVarRegex.exec(content);
  if (first) {
    throw new Error(
      `exec preflight: detected likely shell variable injection (${token}) in ${target.kind} script: ...`,
    );
  }
}
```

如果脚本里出现了 `$UPPERCASE_VAR` 这样的全大写环境变量引用,预检会抛错,并贴心地告诉模型正确的写法:

> In Python, use `os.environ.get("HOME")` instead of raw `$HOME`.
> In Node.js, use `process.env["HOME"]` instead of raw `$HOME`.

为什么只匹配全大写下划线变量?注释解释得很清楚:「我们故意只匹配全大写/下划线变量,以避免和 JS 里把 `$` 当作标识符的合法写法产生误报」。jQuery 时代留下的 `$` 习惯让 JS 里 `$something` 太常见了,所以只盯全大写——这恰好是 shell 环境变量的命名惯例。

针对 Node 还有一条额外检查:如果 `.js` 文件的第一行非空内容以 `NODE` 开头,直接判定为「这看起来是 shell 命令,不是 JavaScript」并报错。

但这个预检最精彩的部分,是它对「无法解析的复杂命令」的**失败闭合**(fail-closed)处理。`extractScriptTargetFromCommand` 是一个快速解析器,它能处理 `python foo.py` 这种直白的命令。但如果命令复杂到快速解析器搞不定怎么办?比如命令里夹杂了管道、命令替换 `$(...)`、进程替换 `<(...)`、或者把脚本内容通过 shell 转一道手?这时候 `shouldFailClosedInterpreterPreflight` 会出场:

```ts
const {
  hasInterpreterInvocation,
  hasComplexSyntax,
  hasProcessSubstitution,
  hasInterpreterSegmentScriptHint,
  hasInterpreterPipelineScriptHint,
  isDirectInterpreterCommand,
} = shouldFailClosedInterpreterPreflight(params.command);
if (
  hasInterpreterInvocation &&
  hasComplexSyntax &&
  (hasInterpreterSegmentScriptHint ||
    hasInterpreterPipelineScriptHint ||
    (hasProcessSubstitution && isDirectInterpreterCommand))
) {
  throw new Error(
    "exec preflight: complex interpreter invocation detected; refusing to run without script preflight validation. " +
      "Use a direct `python <file>.py` or `node <file>.js` command.",
  );
}
```

它的逻辑用大白话说就是:「如果我检测到这是一次解释器(Python/Node)调用,而且命令结构复杂到我没法安全地验证脚本内容,那我就**拒绝执行**,要求你改用直白的 `python <file>.py` 写法」。

代码注释把威胁模型写得明明白白:「当解释器驱动的脚本执行有歧义时,失败闭合;否则攻击者就能把脚本内容路由进我们的快速解析器无法验证的形式里」。这是安全工程里的黄金原则——**当你无法确定一个东西是否安全时,默认它不安全**。预检这道门宁可错杀复杂命令,也不放过一个可能藏着 shell 注入的脚本。

为了识别各种刁钻的命令形态,这个文件里堆了一大堆解析辅助函数:`extractUnquotedShellText`(剥掉引号看真实结构)、`splitShellSegmentsOutsideQuotes`(在引号外按 `;`、`&&`、`||`、`|` 拆分)、`extractShellWrappedCommandPayload`(从 `bash -c "..."` 里掏出真正要跑的命令)、`hasUnescapedSequence`(检测未转义的危险序列)。这些函数共同构成了一个微型的 shell 语法分析器——不是为了执行,而是为了**审查**。

不过预检也不是铁板一块。`shouldSkipExecScriptPreflight` 给出了一个豁免条件:

```ts
function shouldSkipExecScriptPreflight(params: {
  host: ExecHost; security: ExecSecurity; ask: ExecAsk;
}): boolean {
  return params.host === "gateway" && params.security === "full" && params.ask === "off";
}
```

当 host 是 gateway、安全等级是 full、且 ask 是 off 时——也就是说,这是一个**已经被运维完全信任、不设防的执行链路**——预检会被跳过。逻辑是:既然你都已经允许任意命令无审批执行了,那再做这种「防手滑」的预检就没意义了,纯属增加延迟。预检本质上是一层「兜底防护」,而不是「访问控制」,在完全信任的场景下可以省掉。

## 8.7 沙箱:把执行关进 Docker 容器

前面六节讲的都是「在宿主机上跑命令时的层层防护」。但最彻底的隔离手段,是干脆**不在宿主机上跑**——把命令丢进一个 Docker 容器里执行。这就是 `host === "sandbox"` 这条线。

### 后端中立的句柄抽象

OpenClaw 没有把沙箱写死成 Docker。它定义了一个**后端中立**(backend-neutral)的抽象层,让 Docker、SSH、以及未来可能的其他沙箱提供方都实现同一组接口。这套契约在 `sandbox/backend-handle.types.ts` 里:

```ts
export type SandboxBackendHandle = {
  id: SandboxBackendId;
  runtimeId: string;
  runtimeLabel: string;
  workdir: string;
  env?: Record<string, string>;
  capabilities?: { browser?: boolean };
  buildExecSpec(params: {
    command: string;
    workdir?: string;
    env: Record<string, string>;
    usePty: boolean;
  }): Promise<SandboxBackendExecSpec>;
  runShellCommand(params: SandboxBackendCommandParams): Promise<SandboxBackendCommandResult>;
  createFsBridge?: (params: { sandbox: SandboxFsBridgeContext }) => SandboxFsBridge;
};
```

文件头的注释一句话点明了设计意图:「Docker、SSH,以及未来的沙箱提供方实现这些命令、执行和文件系统桥接接口」。

这个句柄最核心的方法是 `buildExecSpec`——它接收一条命令,返回一个 `SandboxBackendExecSpec`(包含 `argv`、`env`、`stdinMode`)。注意它返回的是「怎么启动这个进程的规格」,而不是直接执行。**真正的进程启动逻辑被收拢在 `exec` 工具的通用代码里,后端只负责告诉它「该用什么 argv 启动」**。这种「后端造规格、核心管执行」的分工,让前台/后台移交、超时处理、输出聚合这些复杂逻辑只需要写一遍,所有后端共享。

### Docker 后端的实现

Docker 后端在 `sandbox/docker-backend.ts` 里。它的 `buildExecSpec` 实现非常薄:

```ts
async buildExecSpec({ command, workdir, env, usePty }) {
  return {
    argv: [
      "docker",
      ...buildDockerExecArgs({
        containerName: params.containerName,
        command,
        workdir: workdir ?? params.workdir,
        env,
        tty: usePty,
      }),
    ],
    env: process.env,
    stdinMode: usePty ? "pipe-open" : "pipe-closed",
  };
}
```

说白了,沙箱执行就是把你的命令包进一个 `docker exec <container> ...` 调用里。容器本身由 `ensureSandboxContainer` 负责创建或复用——这个函数会处理「容器是否已存在」「配置是否变了需要重建」等逻辑。

后端还提供了一个 `runShellCommand`,专门给文件系统桥接(fs-bridge)和探针用。注意它的实现里塞了一个有意思的 `argv0`:

```ts
const dockerArgs = ["exec", "-i", params.containerName, "sh", "-c", params.script, "openclaw-sandbox-fs", ...];
```

`sh -c <script>` 后面那个 `openclaw-sandbox-fs` 是 `$0`(脚本名),后续的 `params.args` 则成为 `$1`、`$2`...。用一个固定的、可识别的 `$0` 名字,既方便在进程列表里辨认这是 OpenClaw 沙箱的 fs 操作,也避免了把用户参数误当成脚本路径。

### 后端注册表:可插拔的隔离层

那么 `exec` 工具怎么知道该用 Docker 还是 SSH 后端?答案是一个进程级的**后端注册表**,在 `sandbox/backend.ts` 里。它的结构和我们前几章见过的工具注册表、沙箱运行时注册表如出一辙:

```ts
const SANDBOX_BACKEND_FACTORIES_STATE_KEY = Symbol.for("openclaw.sandboxBackendFactories");

export function registerSandboxBackend(
  id: string,
  registration: SandboxBackendRegistration,
): () => void {
  const normalizedId = normalizeSandboxBackendId(id);
  const resolved = typeof registration === "function" ? { factory: registration } : registration;
  const factories = getSandboxBackendFactories();
  const previous = factories.get(normalizedId);
  factories.set(normalizedId, resolved);
  return () => { /* 恢复 previous,或删除 */ };
}
```

几个值得注意的设计:

**第一,注册表挂在 `globalThis` 上,用 `Symbol.for` 做键。** 这保证了即便模块被多次加载(在测试或复杂打包场景下很常见),整个进程也只有一份注册表。这是 OpenClaw 处理「进程级单例」的标准手法。

**第二,`registerSandboxBackend` 返回一个恢复回调。** 注册时会保存 `previous`,恢复回调会把它还原回去。这让测试和插件可以临时替换某个后端,用完再无痛恢复——注释里直接写明了这个用途:「测试和插件可以安装临时工厂,而核心仍然自动注册内置的 Docker 和 SSH 后端」。

**第三,文件末尾自动注册了两个内置后端:**

```ts
registerSandboxBackend("docker", {
  factory: createDockerSandboxBackend,
  manager: dockerSandboxBackendManager,
});
registerSandboxBackend("ssh", {
  factory: createSshSandboxBackend,
  manager: sshSandboxBackendManager,
});
```

当配置里 `sandbox.backend` 找不到对应后端时,`requireSandboxBackendFactory` 会抛出又一个「错误即文档」式的异常:「沙箱后端 `xxx` 未注册。加载提供它的插件,或者设置 `agents.defaults.sandbox.backend=docker`」。

每个后端除了 `factory`(造句柄),还可以提供一个 `manager`(管生命周期),实现 `describeRuntime`(报告运行时状态)和 `removeRuntime`(清理运行时)。Docker 的 manager 会去 `docker inspect` 真实容器,对比当前镜像和配置镜像是否一致——这为「配置变了要不要重建容器」提供了判断依据。

这套「注册表 + 工厂 + 管理器 + 恢复回调」的模式,我们在前面讲工具系统、讲运行时时已经反复见过。OpenClaw 用同一种结构性语法表达「可插拔的能力」,读多了会形成肌肉记忆——看到 `Symbol.for` 的全局注册表,就知道这又是一个可被插件扩展、可被测试替换的扩展点。

## 8.8 沙箱安全校验:容器配置里的雷区

把命令关进容器,听起来已经很安全了。但容器的**配置本身**就是一个攻击面。如果有人(或者被诱导的模型)能控制容器的挂载、网络、安全 profile,那么沙箱的隔离就可能被从配置层面凿穿。`sandbox/validate-sandbox-security.ts` 就是专门守这道门的,它在 `docker.ts` 创建容器之前被调用:

```ts
// docker.ts 第 444 行附近
validateSandboxSecurity({
  ...params.cfg,
  allowedSourceRoots: params.bindSourceRoots,
  allowSourcesOutsideAllowedRoots:
    params.allowSourcesOutsideAllowedRoots ??
    params.cfg.dangerouslyAllowExternalBindSources === true,
  allowReservedContainerTargets:
    params.allowReservedContainerTargets ??
    params.cfg.dangerouslyAllowReservedContainerTargets === true,
  dangerouslyAllowContainerNamespaceJoin:
    params.allowContainerNamespaceJoin ??
    params.cfg.dangerouslyAllowContainerNamespaceJoin === true,
});
```

注意那几个 `dangerouslyAllow*` 配置键——它们的命名本身就是一种警告。要绕过这些安全检查,你必须显式地写一个带 `dangerously` 前缀的配置,这个名字会在 code review 和配置审计时刺眼地跳出来。

校验文件头的注释定义了威胁模型:「本地可信配置,但要防范脚枪(foot-guns)和配置注入。在创建沙箱容器时于运行时强制执行」。也就是说,它假设配置作者不是恶意的,但要防止两件事:**手滑**(把危险路径误挂进去)和**配置注入**(通过某种途径篡改了配置)。

### 挂载黑名单

最核心的是 `validateBindMounts`。它维护了两张黑名单:

```ts
const BLOCKED_HOST_PATHS = [
  "/etc", "/private/etc", "/proc", "/sys", "/dev", "/root", "/boot",
  // 常见的 Docker socket 所在(或别名)目录
  "/run", "/var/run", "/private/var/run",
  "/var/run/docker.sock", "/private/var/run/docker.sock", "/run/docker.sock",
];

const BLOCKED_HOME_SUBPATHS = [
  ".aws", ".cargo", ".config", ".docker", ".gnupg", ".netrc", ".npm", ".ssh",
] as const;
```

第一张表挡的是系统目录和 Docker socket。这里 Docker socket 是重中之重——`/var/run/docker.sock` 一旦被挂进容器,容器内的进程就能控制宿主机的 Docker 守护进程,等于直接拿到宿主机 root。所以这个路径(及其各种别名和父目录)被反复列举,确保任何形式都挡得住。

第二张表挡的是用户主目录下的敏感子目录:云凭据(`.aws`)、配置(`.config`)、GPG 密钥(`.gnupg`)、SSH 密钥(`.ssh`)、npm token(`.npm`)等等。这些一旦被挂进沙箱,沙箱里跑的命令就能读走你所有的凭据——沙箱就成了「凭据泄漏器」。`getBlockedHostPaths` 会把这些子目录拼到所有可能的 home 目录路径上(`HOME`、`OPENCLAW_HOME`、`USERPROFILE`、`os.homedir()` 等),生成完整的黑名单。

校验逻辑 `getBlockedReasonForSourcePath` 不只匹配精确路径,还匹配**子路径**:

```ts
if (sourceKey === blockedKey || sourceKey.startsWith(`${blockedKey}/`)) {
  return { kind: "targets", blockedPath: blocked };
}
```

所以挂 `/etc` 不行,挂 `/etc/passwd` 也不行,挂 `/` 更不行(`getBlockedReasonForSourcePath` 里专门有一条 `sourceNormalized === "/"` 返回 `covers`,因为挂载根目录「永远不安全」)。

### 符号链接逃逸加固

但黑名单有个经典漏洞:**符号链接**。假如攻击者在一个允许的目录里建一个软链接 `~/myproject/sneaky -> /etc`,然后挂 `~/myproject/sneaky`,字符串检查会以为这是个合法路径,但实际挂进去的是 `/etc`。

`validateBindMounts` 用「双重检查」堵住了这个洞:

```ts
const sourceRaw = parseBindSourcePath(bind);
const sourceNormalized = normalizeHostPath(sourceRaw);
enforceSourcePathPolicy({ bind, sourcePath: sourceNormalized, ... });

// 符号链接逃逸加固:通过已存在的祖先目录解析真实路径,再检查一遍
const sourceCanonical = resolveSandboxHostPathViaExistingAncestor(sourceNormalized);
enforceSourcePathPolicy({ bind, sourcePath: sourceCanonical, ... });
```

第一遍检查字符串形式的路径,第二遍通过 `resolveSandboxHostPathViaExistingAncestor` 把符号链接解析成真实路径(canonical path)再检查一遍。注释说得很直白:「通过已存在的祖先做一次符号链接/realpath 处理,这样不存在的叶子路径也无法绕过 source-root 和 blocked-path 检查」。这是防御符号链接逃逸的标准做法——**永远基于解析后的真实路径做安全决策,而不是用户给的字符串**。

### 网络与安全 profile

除了挂载,校验还覆盖了三个维度:

**网络模式**(`validateNetworkMode`)——拦截 `host` 模式(「绕过容器网络隔离」)和 `container:*` 模式(「加入另一个容器的命名空间,绕过沙箱网络隔离」)。后者默认拦截,除非显式设置 `dangerouslyAllowContainerNamespaceJoin=true`。

**seccomp profile**(`validateSeccompProfile`)——拦截 `unconfined`,因为「禁用 seccomp 会移除系统调用过滤,削弱沙箱隔离」。

**apparmor profile**(`validateApparmorProfile`)——拦截 `unconfined`,因为「禁用 AppArmor 会移除强制访问控制」。

这三个加上挂载校验,由顶层的 `validateSandboxSecurity` 一次性串起来:

```ts
export function validateSandboxSecurity(cfg: {...} & ValidateBindMountsOptions): void {
  validateBindMounts(cfg.binds, cfg);
  validateNetworkMode(cfg.network, {
    allowContainerNamespaceJoin: cfg.dangerouslyAllowContainerNamespaceJoin === true,
  });
  validateSeccompProfile(cfg.seccompProfile);
  validateApparmorProfile(cfg.apparmorProfile);
}
```

四道检查,覆盖了「容器能看到宿主机什么文件」「容器能访问什么网络」「容器内能做什么系统调用」三个层面。这是典型的**纵深防御**(defense in depth)——即便某一层被绕过,还有其他层兜底。

### 还有一道:保留目标路径

最后一个细节:`validateBindMounts` 还会检查挂载的**目标路径**(容器内路径),拦截那些会覆盖 OpenClaw 自己沙箱挂载点的路径:

```ts
const RESERVED_CONTAINER_TARGET_PATHS = ["/workspace", SANDBOX_AGENT_WORKSPACE_MOUNT];
```

`/workspace` 是沙箱里工作区的标准挂载点。如果允许用户把别的东西挂到 `/workspace`,就可能**遮蔽**(shadow)OpenClaw 真正的工作区挂载,造成混乱甚至安全问题。所以这些路径被保留,除非显式开启 `dangerouslyAllowReservedContainerTargets`。

## 8.9 串起来看:一条命令的安全之旅

让我们用一个具体的例子把这一章串起来。假设模型发起了这样一次 `exec` 调用,配置是 `host=auto`、有可用沙箱、`mode=auto`:

```json
{ "command": "python analyze.py", "security": "full", "ask": "off" }
```

这条命令会依次经历:

1. **目标解析**(`resolveExecTarget`)——`host=auto` + 有沙箱 → `effectiveHost=sandbox`。模型没填 `host`,所以没有越权请求。
2. **安全等级求交**——host 是 sandbox,`configuredSecurity` 默认是 `deny`(沙箱里默认不需要宿主机审批,因为隔离已经够强),但 `mode=auto` 经过 `resolveExecModePolicy` 规范化。模型填的 `security=full` 不会让它变宽松——`minSecurity` 保证取最严。
3. **控制命令拦截**(`rejectUnsafeControlShellCommand`)——`python analyze.py` 不是 `/approve` 也不是渠道登录,放行。
4. **环境净化**——host 是 sandbox,跳过宿主机净化,改用 `buildSandboxEnv` 从干净的 `DEFAULT_PATH` 构造环境。
5. **脚本预检**(`validateScriptFileForShellBleed`)——识别出 `python analyze.py`,读取 `analyze.py`,扫描里面有没有 `$UPPERCASE` 的 shell 变量泄漏。如果有,直接报错并教模型用 `os.environ.get`。
6. **沙箱执行**——通过 Docker 后端的 `buildExecSpec` 把命令包成 `docker exec <container> python analyze.py`。而这个 `<container>` 在创建时已经过 `validateSandboxSecurity` 的四道校验,确保它没挂载危险路径、没用危险网络、没禁用 seccomp/apparmor。
7. **前台/后台移交**——如果 `analyze.py` 在 `yieldMs` 内跑完,同步返回结果;否则转入后台会话,模型后续用 `process` 工具轮询。

七道门,层层设防。任何一道都不是「一刀切」的开关,而是根据 host、配置、请求三方协商出来的结果。而最外层的那道门——沙箱——一旦启用,就把前面那些「在宿主机上要小心翼翼」的防护需求,大部分都化解在了隔离边界之内。

这就是 `SandboxToolPolicy` 这个我们从上一章带过来的类型名,在运行时真正的样子:它不是一个抽象概念,而是一套由目标解析、安全地板、命令拦截、环境净化、脚本预检、容器隔离、配置校验共同构成的、可观测、可配置、可插拔的隔离边界。

## 本章小结

这一章我们沿着一次 `exec` 调用的生命周期,拆解了 OpenClaw 是怎么驯服「最危险的工具」的:

- **目标解析**(`resolveExecTarget`)决定命令在哪里跑。`auto` 默认优先沙箱,跳出沙箱需要显式声明,提权会强制回到宿主机——隔离与提权在类型层面互斥。模型请求的 host 必须通过配置校验,越权请求会得到「错误即文档」式的拒绝。
- **安全地板**是整个模型的灵魂:`minSecurity` 取最严的安全等级,`maxAsk` 取最高的询问频率。**模型只能加锁,不能开锁**。`security-floor.test.ts` 里十几个测试逐一钉死了这个不变量,包括「可信渠道下忽略模型的 ask 提升」这种微妙的上下文相关行为。
- **控制命令拦截**(`rejectUnsafeControlShellCommand`)防止 `exec` 用来攻击 OpenClaw 自己——拒绝跑 `/approve`(否则模型能自我批准审批)和交互式渠道登录,并穷举了各种包管理器前缀写法。
- **环境变量净化**挡住 `PATH` 劫持和危险变量,但只在宿主机执行时生效——沙箱场景因为隔离够强,改用干净的 `buildSandboxEnv`。
- **脚本预检**(`validateScriptFileForShellBleed`)拦截 LLM 高频犯的「shell 语法漏进 Python/JS」错误,并对无法安全解析的复杂命令**失败闭合**——不确定就拒绝。
- **沙箱**是最彻底的隔离:后端中立的 `SandboxBackendHandle` 抽象让 Docker/SSH/未来后端实现同一组接口;进程级的后端注册表(`Symbol.for` 全局单例 + 恢复回调)让隔离层可插拔、可替换。
- **沙箱安全校验**(`validateSandboxSecurity`)守住容器配置这个攻击面:挂载黑名单(系统目录、Docker socket、凭据目录)+ 符号链接逃逸加固(基于 realpath 而非字符串决策)+ 网络模式/seccomp/apparmor 校验 + 保留目标路径保护,构成纵深防御。所有绕过开关都带 `dangerously` 前缀,在审计时刺眼可见。

贯穿这一章的几个原则,其实是整本书反复出现的主题在安全领域的具体体现:**安全默认值**(auto 优先沙箱)、**失败闭合**(不确定就拒绝)、**纵深防御**(多层独立校验)、**错误即文档**(异常里写明修复键)、**可插拔抽象 + 注册表**(后端中立)。读懂了 `exec` 的防护,你就读懂了 OpenClaw 对待「能力越大、约束越严」这件事的全部诚意。

## 动手实验

下面的实验都基于 OpenClaw 的真实代码,建议你 clone 仓库(`git clone https://github.com/openclaw/openclaw.git`)后动手验证。

**实验一:验证安全地板「只能加锁」。**
打开 `src/agents/bash-tools.exec.security-floor.test.ts`,通读 `describe("exec security floor")` 里的每一个 `it`。试着只看测试名,自己预测每个测试的断言结果(比如「配置 full + 模型填 deny」应该怎样)。然后对照 `createExecTool` 里 `minSecurity`/`maxAsk` 的调用,解释为什么 `"ignores model-supplied ask overrides when configured ask is off"` 和 `"honors per-call ask hardening for trusted callers without messageProvider"` 这两个测试的结果相反——关键差异在哪个变量上?

**实验二:给挂载黑名单加一条规则。**
在 `src/agents/sandbox/validate-sandbox-security.ts` 的 `BLOCKED_HOME_SUBPATHS` 里,假设你想再禁掉 `.kube`(Kubernetes 凭据目录)。加上之后,运行 `validate-sandbox-security.test.ts`,看现有测试是否仍通过。然后写一个新测试:挂载 `~/.kube/config` 应该被 `validateBindMounts` 拒绝。思考:为什么这张表用 home 子目录的形式,而不是写死绝对路径?(提示:看 `getBlockedHostPaths` 怎么处理多个 home 来源。)

**实验三:复现符号链接逃逸的防御。**
读懂 `validateBindMounts` 里「双重检查」的逻辑后,构造一个思想实验:如果只保留第一遍 `sourceNormalized` 的检查、删掉第二遍 `sourceCanonical` 的检查,攻击者要怎么用符号链接把 `/etc` 挂进沙箱?画出这个攻击的路径,并指出 `resolveSandboxHostPathViaExistingAncestor` 在哪一步把它挡下。

**实验四:追踪一条命令的完整旅程。**
以 `{ "command": "node build.js && rm -rf dist", "host": "gateway" }` 为例,在纸上(或注释里)走一遍 8.9 节的七道门,标注每一道门会对这条命令做什么。重点:第 5 步的脚本预检会不会因为命令里有 `&&` 而触发 `shouldFailClosedInterpreterPreflight`?读 `shouldFailClosedInterpreterPreflight` 的返回值条件,判断这条命令是「直接解释器命令」还是「复杂语法」,以及它最终会不会被失败闭合拦下。

---

下一章,我们会从「单个工具的执行安全」上升到「多个 Agent 的协作」——看 OpenClaw 是怎么让一个主 Agent 派生子 Agent、子 Agent 又如何在受限的工具策略下完成委派任务的。上一章提到的 `SUBAGENT_TOOL_DENY_ALWAYS` 那张拒绝清单,到那时会显示出它真正的分量。
