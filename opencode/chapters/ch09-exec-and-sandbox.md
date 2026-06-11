# 第 9 章　命令执行与沙箱：让 `bash` 跑起来又不被它反噬

> 工具系统里最危险的一个，是 `bash`——它的描述里明明白白写着"with the host user's filesystem, process, and network authority"。
> 一条 shell 命令拥有宿主用户的全部权柄：它能删文件、能联网、能 fork 出永不退出的子进程。本章看 OpenCode 怎么把这头猛兽关进笼子：超时、捕获上限、工作目录边界、外部目录授权、进程组清理。

第 8 章我们把工具系统的骨架读完了——注册、物化、结算、输出上限。本章聚焦那个最需要被约束的叶子：`tool/bash.ts`（207 行）以及它背后的进程运行时 `process.ts`（`AppProcess`）和路径边界服务 `location-mutation.ts`。读完你会看到，v2 对"执行一条命令"这件事的态度是彻底的防御性——**每一个能失控的维度（时间、内存、路径、子进程生命周期）都被一个明确的常量或一道明确的闸门兜住。** 而它对自己"还没做到的部分"也毫不掩饰：`bash.ts` 里排着十几条 `TODO`，把与 v1 的"能力欠债"明明白白挂在墙上。

## 9.1 四个常量：把"能失控的维度"先钉死

`bash.ts` 顶部就把三个最关键的安全边界声明成常量：

```ts
export const DEFAULT_TIMEOUT_MS = 2 * 60 * 1_000   // 默认 2 分钟
export const MAX_TIMEOUT_MS = 10 * 60 * 1_000      // 最长 10 分钟
export const MAX_CAPTURE_BYTES = 1024 * 1024       // 输出捕获上限 1MB
```

这三个数字分别管住两类失控：**时间**（命令跑太久）和**内存**（输出刷太多）。而且 timeout 不是软约束——它被编织进了 input schema：

```ts
timeout: PositiveInt.check(Schema.isLessThanOrEqualTo(MAX_TIMEOUT_MS))
  .pipe(Schema.optional)
```

模型想传一个超过 10 分钟的 timeout，**根本通不过 schema 校验**（回想第 8 章 `settle` 那道 `decodeUnknownEffect`，非法 input 直接被挡成 `Invalid tool input`）。这是第 1 章"约束只收紧"原则的微观样本：上限不是运行时才检查的，是在类型边界上就拒绝的——模型连"请求一个过大的值"都做不到。

## 9.2 工作目录边界：相对路径不许逃逸，绝对路径要授权

`bash` 的第一道实质防御，是它怎么确定"在哪个目录里跑"。它不直接信任模型传来的 `workdir`，而是交给 `LocationMutation.resolve`：

```ts
const target = yield* mutation.resolve({ path: input.workdir ?? ".", kind: "directory" })
const external = target.externalDirectory
if (external)
  yield* permission.assert({
    ...LocationMutation.externalDirectoryPermission(external),
    sessionID: context.sessionID, agent: context.agent, source,
  })
```

`location-mutation.ts` 这个服务的模块注释把规则讲得斩钉截铁：

> Relative paths must stay inside the active Location. Absolute paths outside it require separate `external_directory` approval.

它的 `resolve` 把路径分成两类处理，每一类都有对应的 `PathError`：

- **相对路径**：必须留在当前 Location 内。算出绝对路径后，若 `!FSUtil.contains(location.directory, absolute)`，直接 `relative_escape` 报错——`../../../etc` 这种企图用相对路径逃出沙箱的，当场拦下。
- **绝对路径**：允许指向 Location 外，但会被标记成 `externalDirectory`，**触发一道额外的 `external_directory` 权限断言**。换句话说，命令想在沙箱外的目录跑，得先过一遍用户授权（第 10 章的主角）。

最精妙的是它对**符号链接**的处理。`resolve` 里有两道 `contains` 检查：先做"词法上"的判断（`lexicallyInternal`，纯字符串层面在不在 Location 内），再用 `fs.realPath` 解析出真实路径后，对**内部路径**再查一次：

```ts
if (lexicallyInternal && !FSUtil.contains(locationRoot, resolved.canonical)) {
  return yield* new PathError({ path: input.path, reason: "location_escape" })
}
```

为什么"看起来在内部"的路径还要再查一遍真实路径？因为 Location 内可能有一个符号链接指向外部——词法检查看它是内部的，`realPath` 解开后才暴露它真正落在外面。这正是第 8 章 `read.ts` 里那道"两次校验"在执行侧的同款防御：**字符串路径不可信，符号链接解析后的真实路径才算数。** `resolvePath` 还能处理"目标文件还不存在"的情况——它沿父目录向上爬，找到第一个真实存在的祖先目录，若祖先不是目录就报 `non_directory_ancestor`。

## 9.3 命令参数里的外部目录：advisory 警告

除了 `workdir`，命令字符串本身的参数里也可能藏着外部路径——比如 `cat /etc/hosts`。`bash.ts` 用 `externalCommandDirectories` 做了一道**尽力而为的扫描**：

```ts
const externalCommandDirectories = (command, cwd) => {
  const directories = new Set<string>()
  for (const token of shellTokens(command)) {
    const value = unquote(token).replace(/[;,|&]+$/, "")
    if (!path.isAbsolute(value)) continue
    const resolved = FSUtil.resolve(value)
    if (FSUtil.contains(cwd, resolved)) continue
    directories.add(FSUtil.resolve(path.dirname(resolved)))
  }
  return [...directories]
}
```

它把命令切成 token，挑出绝对路径、且落在 cwd 外的，收集成一组"外部目录"，然后生成警告文本。但这里 v2 非常诚实地标注了它的**局限性**——警告里直接写着"this scan is advisory only"（仅供参考），而且代码顶部的 TODO 也承认：

```ts
// TODO: Replace token-based command-argument external-directory advisories with parser-based detection.
```

为什么只是 advisory？因为 shell 语法极其复杂（变量展开、子命令、管道），用正则切 token 根本不可能准确识别所有路径参数。v2 的态度是：**与其假装能完美拦截、给人虚假的安全感，不如老实承认"这只是个尽力而为的提示"。** 真正的硬边界是 `workdir` 的 `external_directory` 授权——那个是会阻塞执行的；命令参数扫描只是锦上添花的告警。这种"区分硬边界与软提示、并诚实标注哪个是哪个"的克制，本身就是一种工程诚实。

扫描完外部目录、（必要时）过完外部授权后，`bash` 才对命令本身做权限断言：

```ts
yield* permission.assert({
  action: name,                  // "bash"
  resources: [input.command],    // 整条命令作为资源
  save: [input.command],
  sessionID: context.sessionID, agent: context.agent, source,
})
```

整条命令字符串作为权限资源送去断言——这是第 10 章会展开的"命令级批准"。

## 9.4 进程怎么真正被 spawn：detached + forceKillAfter

权限过了、目录确认是目录了（`fs.stat(target.canonical)).type !== "Directory"` 会拦下"工作目录不是目录"的情况），`bash` 才真正构造子进程：

```ts
const command = ChildProcess.make(input.command, [], {
  cwd: target.canonical,
  shell,
  stdin: "ignore",
  detached: process.platform !== "win32",
  forceKillAfter: Duration.seconds(3),
})
```

这里每个选项都对应一个真实风险：

- **`stdin: "ignore"`**：不给命令接 stdin。一个交互式命令（比如等输入的 `read`）不会因为没人喂输入而把整个 agent 挂死。
- **`detached: process.platform !== "win32"`**：在 POSIX 上让子进程**独立成组**（new process group）。这是为了能在超时/中断时把**整个进程组**一起杀掉——否则一个 `npm install` 派生的孙子进程可能在父进程死后变成孤儿继续跑。
- **`forceKillAfter: Duration.seconds(3)`**：先发温和的终止信号，3 秒内不退就强杀。给进程一个干净退出的机会，又不允许它赖着不走。

shell 本身也是可配置的——先读 config 里的 `shell`，没配才回退到 `defaultShell()`（POSIX 上 `/bin/sh`，Windows 上 `COMSPEC` 或 `cmd.exe`）。

## 9.5 输出捕获：边读边截，1MB 封顶

子进程跑起来后，`AppProcess.run` 负责收集它的输出，并把 `MAX_CAPTURE_BYTES` 作为上限传进去：

```ts
const result = yield* appProcess.run(command, {
  timeout: Duration.millis(timeout),
  maxOutputBytes: MAX_CAPTURE_BYTES,
  maxErrorBytes: MAX_CAPTURE_BYTES,
})
```

`process.ts` 里的 `collectStream` 是这道上限的执行者。它用 `Stream.runFold` 边读边累加，**一旦累计字节超过上限，就只保留前面够数的部分、把 `truncated` 标记置位**，但仍然继续把流读完（消费掉剩余字节，避免管道阻塞）：

```ts
const remaining = maxOutputBytes - acc.bytes
if (remaining > 0) acc.chunks.push(remaining >= chunk.length ? chunk : chunk.slice(0, remaining))
acc.bytes += chunk.length
acc.truncated = acc.truncated || acc.bytes > maxOutputBytes
```

这是一道**内存安全网**：一个疯狂刷屏的命令（`yes` 或 `cat 大文件`）不会把 agent 进程的内存吃光——最多在内存里留 1MB，多的边读边丢。

**注意这里有两道独立的输出上限，第 8 章已埋下伏笔，这里看清全貌：**

| 上限 | 在哪 | 管什么风险 | 值 |
|---|---|---|---|
| `MAX_CAPTURE_BYTES` | `bash.ts` / `AppProcess` | 别让子进程刷爆 **agent 进程内存** | 1MB |
| `MAX_BYTES` / `MAX_LINES` | `tool-output-store.ts`（第 8 章） | 别让结果刷爆 **模型上下文** | 50KB / 2000 行 |

`bash` 工具捕获到最多 1MB，但这 1MB 再交给注册表结算时，还会被 `tool-output-store` 进一步压到 50KB 喂给模型。`tool/AGENTS.md` 那句"Bash keeps `AppProcess.maxOutputBytes`... but it does not run model-output truncation"说的就是这个分工——**捕获上限保护内存、结算上限保护上下文，两道闸门串联。** 截断发生时，`captureNotice` 会生成一条明确的提示文本（`[stdout capture truncated at the in-memory safety limit]`），让模型知道"你看到的不是全部"。

## 9.6 超时不是错误，是一种正常结局

`bash` 对超时的处理体现了一个细腻的设计取舍。`AppProcess.run` 内部用 `Effect.timeoutOrElse`，超时会抛一个 `cause` 为 `"Timed out"` 的 `AppProcessError`。但 `bash` 把这个特定错误**接住、转成一个正常的返回值**：

```ts
.pipe(
  Effect.catchTag("AppProcessError", (error) =>
    isTimeout(error) ? Effect.succeed(undefined) : Effect.fail(error),
  ),
)
if (!result) {
  return {
    command: input.command, cwd: target.canonical,
    output: `Command exceeded timeout of ${timeout} ms. Retry with a larger timeout if the command is expected to take longer.`,
    truncated: false, timedOut: true,
    ...(warnings.length ? { warnings } : {}),
  }
}
```

为什么超时不当成失败抛出去，而是返回一个 `timedOut: true` 的结构化结果？因为**对 agent 来说，"命令超时了"是一条有用的信息，不是一个该终止整轮的异常**。模型拿到这个结果，能读懂提示"Retry with a larger timeout if the command is expected to take longer"，自己决定要不要加大 timeout 重试。这是第 1 章"错误即文档"的又一面——把"可恢复的失败"翻译成模型能理解、能据此行动的反馈，而不是一个让它无从应对的栈。而真正的意外错误（非超时的 `AppProcessError`）仍然 `Effect.fail` 抛出去，最后被统一 `mapError` 成 `ToolFailure`。

## 9.7 墙上的欠债清单：把"还没做"挂出来

`bash.ts` 最有性格的一段，是它顶部那排 `TODO` 注释——足足十几条，诚实地列出了 v2 core shell 相对 v1 的所有"能力欠债"。摘几条：

```ts
// TODO: Port tree-sitter bash / PowerShell parser-based approval reduction.
// TODO: Port BashArity reusable command-prefix approvals.
// TODO: Restore PowerShell and cmd-specific invocation/path handling on Windows.
// TODO: Re-add model-facing background launch only with owner-bound get/wait/cancel tools and completion delivery.
// TODO: Persist background job status and define restart recovery before exposing remote observation.
```

它的整体注释定了调子："Minimal V2 core shell boundary. Keep parity debt visible without pulling the legacy shell runtime into core."——**保持一个最小的 shell 边界，把欠债摆在明处，而不是把 v1 那套庞大的 shell 运行时整个拖进 core。** 这呼应第 5 章 runner 里那份 `[x]/[ ]` 蓝图：v2 反复用"显式列出未完成项"的方式，拒绝假装完美。

尤其值得玩味的是关于**后台任务**的几条 TODO。v2 目前**砍掉了**面向模型的后台命令启动能力，并把"重新加回来"的前置条件写得清清楚楚：必须先有"owner-bound 的 get/wait/cancel 工具""持久化的任务状态""重启恢复""授权定义"。这是 fail-closed 的极致体现——**一个还没想清楚安全模型的能力，宁可暂时不提供，也不带着隐患上线。** 与其先放出一个能 spawn 失控后台进程的 `bash &`、再回头补窟窿，不如先关掉它，等配套的可观测、可取消、可授权机制齐了再开。

## 本章小结

- `bash` 工具用四个常量钉死可失控维度：`DEFAULT_TIMEOUT_MS`(2min)、`MAX_TIMEOUT_MS`(10min)、`MAX_CAPTURE_BYTES`(1MB)；timeout 上限直接编进 input schema，模型连"请求超额值"都做不到——约束只收紧。
- 工作目录经 `LocationMutation.resolve` 把关：相对路径逃逸报 `relative_escape`、Location 内符号链接指向外部报 `location_escape`（realPath 二次校验）、绝对外部路径触发 `external_directory` 授权——字符串路径不可信，真实路径才算数。
- 命令参数里的外部目录只做 advisory 扫描并诚实标注"advisory only"（正则切 token 无法准确解析 shell），真正硬边界是 `workdir` 授权——区分硬边界与软提示是一种工程诚实。
- 子进程以 `stdin:"ignore"`、POSIX 下 `detached`（独立进程组便于整组清理）、`forceKillAfter:3s`（先温和后强杀）spawn；shell 可配置，回退 `/bin/sh`/`cmd.exe`。
- 两道串联的输出上限：`AppProcess` 的 1MB 捕获上限保护进程内存（`collectStream` 边读边截、置 `truncated`、仍读完流防阻塞），`tool-output-store` 的 50KB/2000 行保护模型上下文；截断有明确 notice。
- 超时被 `catchTag` 接住转成 `timedOut:true` 的结构化结果（含"加大 timeout 重试"提示），而非抛异常——把可恢复失败翻译成模型能据此行动的反馈。
- 顶部十余条 `TODO` 把相对 v1 的能力欠债挂在明处；后台任务能力被刻意砍掉，并把"加回来"的前置安全条件写清——还没想清安全模型的能力宁可不上线（fail-closed）。

## 动手实验

1. **实验一：验证 timeout 的类型级约束** — 在 `bash.ts` 找到 `timeout` 的 schema 定义 `PositiveInt.check(Schema.isLessThanOrEqualTo(MAX_TIMEOUT_MS))`。假设模型传 `timeout: 999999999`（约 11.5 天），它会在哪一步被拦下？是运行时检查还是 schema 解码？对照第 8 章 `settle` 管道说说错误信息会是什么。
2. **实验二：手推一次路径逃逸** — 在 `location-mutation.ts` 的 `resolve` 里，假设 Location 是 `/work`，模型传 `workdir: "../etc"`（相对）和 `workdir: "/etc"`（绝对）。分别推演：前者命中哪个 `PathError`？后者会不会报错、还是走 `externalDirectory` 授权？
3. **实验三：算两道输出上限** — 一条命令输出 3MB 文本。手推它经过 `AppProcess.collectStream`（1MB 上限）后内存里留多少、`truncated` 是什么；再经过第 8 章 `tool-output-store.bound`（50KB）后，模型最终看到多少、有几条截断提示？画出这条"3MB→1MB→50KB"的衰减链。
4. **实验四：读懂"砍掉后台任务"的取舍** — 在 `bash.ts` 顶部找到关于 background launch 的几条 TODO。列出 v2 给"重新加回后台启动"开出的前置条件（owner-bound 工具、持久状态、重启恢复、授权）。用一句话向同事解释：为什么"先砍掉、等条件齐了再加"比"先上线再补窟窿"更符合 fail-closed？

> **下一章预告**：本章两次提到 `permission.assert`——无论是外部目录还是命令本身，最终都要过这道权限断言。第 10 章进入权限系统，读 `core/src/permission/` 与子 agent 权限，看 OpenCode 如何用一套 allow/ask/deny 规则 + 通配符匹配 + "记住这次选择"的机制，决定一个工具调用到底能不能落地，以及子 agent 的权限为什么"只能比父 agent 更窄"。
