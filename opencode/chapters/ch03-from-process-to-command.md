# 第 3 章　从进程到命令：一次请求的起点

第 2 章我们鸟瞰了整个 monorepo。现在开始下潜，从最外层切入：当你在终端敲下 `opencode run "帮我修复这个 bug"` 并回车，到底发生了什么？本章顺着这条命令，把"进程怎么启动、命令怎么被解析、最终怎么唤醒会话"这条链路走一遍。读完你会发现一个让人意外的设计：**即便是本地单机运行，OpenCode 也在进程内起了一个 HTTP 服务，让 CLI 通过 SDK 客户端去调它自己。**

## 3.1 启动器 `bin/opencode`：一个平台分发器

用户安装后实际执行的，是 `packages/opencode/bin/opencode` 这个 Node 脚本（`#!/usr/bin/env node`）。但它本身几乎不含业务逻辑——它的全部职责是**找到并启动平台专属的原生二进制**。

脚本顶部就声明了平台与架构映射表：

```js
const platformMap = { darwin: "darwin", linux: "linux", win32: "windows" }
const archMap = { x64: "x64", arm64: "arm64", arm: "arm" }
```

接着拼出基础名 `const base = "opencode-" + platform + "-" + arch`，再根据 CPU 能力做更细的分流。最精彩的是 `supportsAvx2()` 这个函数——它会真的去读 `/proc/cpuinfo`（Linux）、跑 `sysctl -n hw.optional.avx2_0`（macOS）、或调 Windows 的 `IsProcessorFeaturePresent`，来判断当前 CPU 是否支持 AVX2 指令集。为什么一个 AI Agent 的启动器要关心 AVX2？因为它据此在"性能优化版"和"baseline 兼容版"二进制之间选择：

```js
const avx2 = supportsAvx2()
const baseline = arch === "x64" && !avx2
```

它还会探测 Linux 的 libc 实现——通过 `fs.existsSync("/etc/alpine-release")` 和跑 `ldd --version` 检查输出里有没有 `musl`，从而在 glibc 版和 musl 版二进制间选对的那个。最终它构造出一个**带优先级的候选名数组** `names`，比如在支持 AVX2 的 glibc x64 上是 `[base, base-baseline, base-musl, base-baseline-musl]`——优先用最优版本，逐级回退到最兼容版本。

`findBinary` 则从脚本所在目录开始**逐级向上遍历 `node_modules`**，按 `names` 的优先级找第一个存在的二进制：

```js
const resolved = envPath || (fs.existsSync(cached) ? cached : findBinary(scriptDir))
```

找到后用 `run(resolved)` 把当前进程的参数原样转发给子进程。注意 `run` 里有一段**信号转发**逻辑：它监听 `["SIGINT", "SIGTERM", "SIGHUP"]`，把这些信号 `child.kill(signal)` 转发给子进程，并在子进程退出时用相同信号杀掉自己（`process.kill(process.pid, signal)`）。

> **设计观察**：把"选哪个二进制"这件平台相关的脏活，从主程序里彻底剥离到一个纯 Node 启动器里，是一种干净的关注点分离。主程序可以假定自己跑在最合适的原生环境里，而所有"CPU 支不支持 AVX2、是不是 musl"的探测都被隔离在这 200 行启动器内。这呼应了第 2 章"happy path 与杂活分离"的纪律——只不过这次分离发生在进程边界上。

## 3.2 入口 `src/index.ts`：一棵 yargs 命令树

被启动的原生二进制，其逻辑源头是 `packages/opencode/src/index.ts`。它做的第一件事是用 [yargs](https://yargs.js.org/) 搭起一棵命令树：

```ts
const cli = yargs(args)
  .parserConfiguration({ "populate--": true })
  .scriptName("opencode")
  .wrap(100)
  .help("help", "show help")
  .version("version", "show version number", InstallationVersion)
```

然后挂上全局选项，再 `.command(...)` 注册了一长串子命令。把它们列出来，本身就是 OpenCode 能力边界的一份目录：

```ts
.command(AcpCommand)        // Agent Client Protocol
.command(McpCommand)        // MCP 服务
.command(TuiThreadCommand)  // 终端 UI
.command(RunCommand)        // 跑一条 prompt（本章主角）
.command(ServeCommand)      // 起 HTTP 服务
.command(WebCommand)        // Web 前端
.command(GithubCommand) / .command(PrCommand)  // GitHub 集成
.command(SessionCommand) / .command(ExportCommand) / .command(ImportCommand)
.command(PluginCommand) / .command(DbCommand) // 插件、数据库维护
// ……还有 agent / models / providers / stats / upgrade / uninstall 等
```

几个工程细节值得注意：

**① 全局 middleware 把 CLI flag 翻译成环境变量。** 在所有命令执行前，一段 `.middleware` 先跑：

```ts
.middleware(async (opts) => {
  if (opts.printLogs) process.env.OPENCODE_PRINT_LOGS = "1"
  if (opts.logLevel) process.env.OPENCODE_LOG_LEVEL = opts.logLevel
  if (opts.pure) process.env.OPENCODE_PURE = "1"
  Heap.start()
  process.env.AGENT = "1"
  process.env.OPENCODE = "1"
  process.env.OPENCODE_PID = String(process.pid)
})
```

其中 `--pure` → `OPENCODE_PURE=1` 是一个很有意思的开关，它的描述是 "run without external plugins"——一个**让 Agent 在不加载任何外部插件的"纯净模式"下运行**的逃生舱。当你怀疑某个第三方插件搞坏了行为时，`--pure` 能帮你快速二分定位。把它做成顶层全局 flag，说明"可被旁路的扩展"是 OpenCode 的核心安全假设之一（第 13 章再谈）。

**② 顶层错误处理分两层。** `.fail(...)` 钩子专门识别"参数错误"类消息（`Unknown argument`、`Not enough non-option arguments`、`Invalid values:`），对这类用户输入错误显示帮助而非堆栈；而真正的运行时异常则交给最外层的 `try/catch`，用 `FormatError(e)` 格式化后输出。这种"用户错误 vs 程序错误"的分流，正是第 1 章说的"错误即文档"理念的入口级体现。

**③ `finally` 里强制 `process.exit()`。** 最外层 `finally` 块有一段注释解释得很坦白：某些子进程（特别是基于 docker 容器的 MCP server）不能正确响应 SIGTERM，所以"Explicitly exit to avoid any hanging subprocesses"——显式退出，避免挂起的子进程拖住整个 CLI。这是一个被生产环境的真实痛点逼出来的防御。

## 3.3 `run` 命令：本地也走"服务器 + 客户端"

现在聚焦本章主角 `RunCommand`（`cli/cmd/run.ts`，888 行）。它的文件头注释直接交代了三种模式：

```
//   1. Non-interactive (default): sends a single prompt, streams events to
//      stdout, and exits when the session goes idle.
//   2. Interactive local (--interactive): boots the split-footer direct mode
//      with an in-process server (no external HTTP).
//   3. Interactive attach (--interactive --attach): connects to a running
//      opencode server and runs interactive mode against it.
```

最值得玩味的是第 1、2 种模式里反复出现的 **"in-process server"**。命令的元信息里写着：

```ts
// --attach connects to a remote server (no local instance needed); the
// default path runs an in-process server and needs the project instance.
instance: (args) => !args.attach,
```

也就是说：**除非你用 `--attach` 连到一个远程 server，否则本地 `run` 会在自己进程里起一个服务实例，CLI 自身则作为客户端去访问它。** 证据是 handler 里无论本地还是远程，最终都通过同一个 SDK 客户端发请求：

```ts
import { createOpencodeClient, type OpencodeClient } from "@opencode-ai/sdk/v2"
// ……
const result = await client.session.prompt({ /* ... */ })
```

为什么本地单机也要绕这么一圈，而不是直接函数调用？因为这样**一套核心运行时同时服务于所有前端**：CLI、TUI、Web、attach 远程，统统是同一个 `@opencode-ai/sdk` 客户端打到同一套 server 路由。本地模式只是把 server 起在同进程里、用一个随机端口（`--port` 的描述："defaults to random port if no value provided"）。这是一个非常重要的架构决策，我们会在第 14 章看到它的全貌——**OpenCode 的所有"脸"都长在同一具"身体"上，靠 HTTP/SDK 这层接缝连接。**

## 3.4 handler：用 Effect 编排的启动序列

`run` 的 handler 用 `Effect.fn("Cli.run")` 包裹，开头是一串**动态导入**：

```ts
handler: Effect.fn("Cli.run")(function* (args) {
  const { Agent } = yield* Effect.promise(() => import("@/agent/agent"))
  const { RuntimeFlags } = yield* Effect.promise(() => import("@/effect/runtime-flags"))
  const { InstanceRef } = yield* Effect.promise(() => import("@/effect/instance-ref"))
  const { ServerAuth } = yield* Effect.promise(() => import("@/server/auth"))
  const agentSvc = yield* Agent.Service
  const flags = yield* RuntimeFlags.Service
  const localInstance = yield* InstanceRef
  // ……
```

这段代码同时展示了 OpenCode 的两个工程习惯：

- **动态导入做启动优化**。还记得第 2 章引用的 `AGENTS.md` 纪律吗？"Prefer dynamic imports for heavy modules that are only needed in selected code paths, especially in startup-sensitive entrypoints"。`run` 是启动敏感的入口，所以它把 `Agent`、`ServerAuth` 这些重模块延迟到真正进入 handler 时才加载，缩短冷启动时间。
- **`yield* Service` 是 Effect 的依赖注入**。`yield* Agent.Service`、`yield* InstanceRef` 不是普通取值，而是从 Effect 的运行时环境里"要"一个服务实例。这套机制让核心逻辑不必关心服务怎么构造，只声明"我需要 Agent 服务"，由外层的 Layer 负责装配。

handler 接下来是一长串**输入校验**，几乎全是 early return / `die` 的组合，活脱脱是第 2 章"early return 当前置条件清单"纪律的范本：

```ts
if (args.interactive && args.command) die("--interactive cannot be used with --command")
if (args.demo && !args.interactive) die("--demo requires --interactive")
if (args.interactive && args.format === "json") die("--interactive cannot be used with --format json")
if (args.interactive && !process.stdout.isTTY) die("--interactive requires a TTY stdout")
```

每个 `die(message)` 都给出**人话级**的错误说明——告诉用户"这两个 flag 不能一起用"、"这个模式需要 TTY"，而不是抛一个晦涩的栈。这是"错误即文档"原则在 CLI 层的密集体现：错误信息本身就是用法教学。

校验通过后，handler 解析工作目录（注意它保留了"先 chdir 再解析文件"的传统顺序）、解析附件文件、构造 SDK 客户端，最终把用户的 prompt 通过 `client.session.prompt({...})` 投递进会话系统。**到这里，请求就正式离开 CLI 层，进入了第 4 章的主角——Session 运行时。**

> **一处呼应 v2 纪律的细节**：`session.prompt` 这个调用名不是随意的。第 2 章我们引用过 `AGENTS.md` 的"V2 Session Core"：`SessionV2.prompt(...)` 只负责**准入一条持久化的 prompt**，再调度执行，而不是同步地把模型跑完。也就是说，CLI 在这里做的是"投递"，真正的"执行"是异步在 Session 运行时里发生的。这种"投递与执行解耦"正是 v2 的核心设计，下一章见分晓。

## 本章小结

- `bin/opencode` 是一个纯 Node 启动器，职责是探测平台/CPU/libc（AVX2、musl）后，从带优先级的候选名数组里选出最合适的原生二进制并转发信号与参数。
- `src/index.ts` 用 yargs 搭起一棵命令树，注册了 run/serve/tui/mcp/acp/github/session/plugin/db 等二十余个子命令，全局 middleware 把 CLI flag 翻译成 `OPENCODE_*` 环境变量。
- `--pure`(`OPENCODE_PURE=1`) 提供"不加载外部插件"的纯净模式逃生舱，是 OpenCode"扩展可被旁路"安全假设的入口级体现。
- 顶层错误处理分两层：`.fail` 处理参数类用户错误并显示帮助，最外层 `try/catch` 处理运行时异常；`finally` 强制 `process.exit()` 以防 MCP 子进程挂起。
- `run` 命令的关键设计是"本地也走服务器+客户端"：除非 `--attach` 远程，否则在进程内起一个 server，CLI 作为 `@opencode-ai/sdk` 客户端访问它——让一套核心运行时同时服务 CLI/TUI/Web/远程。
- handler 用 Effect 编排：动态导入做启动优化、`yield* Service` 做依赖注入、一连串 early return + `die(人话错误)` 做输入校验，最后用 `client.session.prompt(...)` 把 prompt 投递进 Session 系统——投递与执行解耦正是 v2 核心设计。

## 动手实验

1. **实验一：读懂二进制选择逻辑** — 打开 `packages/opencode/bin/opencode`，找到 `names` 那个 IIFE。假设你在一台不支持 AVX2 的 glibc x64 Linux 上，手推一遍 `names` 数组的最终顺序，并解释为什么 baseline 版排在最前。
2. **实验二：清点能力边界** — 在 `packages/opencode/src/index.ts` 里数一数总共 `.command(...)` 注册了多少个子命令。挑三个你没见过的（如 `AcpCommand`、`PrCommand`、`DbCommand`），到 `packages/opencode/src/cli/cmd/` 下打开对应文件读它的 `describe` 字段，猜它的用途。
3. **实验三：验证"本地也起服务器"** — 在 `cli/cmd/run.ts` 里搜索 `createOpencodeClient` 和 `instance:`，确认默认路径（非 `--attach`）确实需要本地 instance。再思考：如果改成 CLI 直接函数调用核心逻辑，会失去什么（提示：TUI、Web、attach 三种前端的复用）？
4. **实验四：收集"错误即文档"样本** — 在 `cli/cmd/run.ts` 的 handler 里，把所有 `die("...")` 的错误消息抄下来。评价这些消息：它们是在"教用户怎么用"还是"甩给用户一个栈"？挑一条你觉得写得最好的，说说好在哪。

> **下一章预告**：prompt 已经通过 `session.prompt` 投递出去了。第 4 章我们进入 Session 运行时——看 v2 如何把"准入 prompt"与"执行模型"彻底解耦：`SessionRunner`、`SessionExecution`、`run-coordinator` 是如何协作，让一条投递进来的消息，在安全的边界上被提升为可见的用户消息并驱动一次模型调用的。
