# 第 3 章　从进程到运行时：启动入口与双驱动分流

> 一个 CLI Agent 的第一行用户体验，不是模型说的第一句话，而是你按下回车后那几十毫秒里发生的事。
> 这一章我们走进第一道门：从 `claude` 这个命令被 shell 拉起，到主循环被点燃之前，CCB 做了哪些事，又是怎么把「人坐在终端前聊天」和「脚本批量跑」这两种截然不同的使用方式从同一个入口分流出去的。

## 3.1 入口的第一性原理：把启动开销压到最小

CCB 的真正入口是 `src/entrypoints/cli.tsx`。它的文件头就是一行 shebang：`#!/usr/bin/env bun`——这是个 Bun 脚本，不是 Node。紧接着的第一条 import 不是任何业务代码，而是一句注释极重的性能补丁：

```ts
// Performance shim MUST be the first import — it replaces globalThis.performance
// with a JS-backed implementation before React/OTel capture the native reference.
import '../utils/performanceShim.js';
```

注释解释了原因：如果不在 React 和 OTel 捕获原生 `performance` 引用之前替换掉它，JSC（JavaScriptCore，Bun 的引擎）底层的 C++ Vector 会在长会话里无限增长。一个 Agent 会跑几个小时甚至几天，这种内存泄漏在 Demo 里看不出来，在生产里是致命的。**入口文件的第一行，就已经是「为长时运行而设计」的工程决策。**

接下来 `cli.tsx` 里 `main()` 函数顶部的注释道出了整个入口的设计哲学：

```
* Bootstrap entrypoint - checks for special flags before loading the full CLI.
* All imports are dynamic to minimize module evaluation for fast paths.
* Fast-path for --version has zero imports beyond this file.
```

**所有 import 都是动态的（`await import(...)`），目的是把模块求值压到最小。** 这是因为 CCB 是一个巨型应用——静态 import 整棵依赖树会让每一次 `claude --version` 都付出加载几千个模块的代价。所以入口被设计成一个「分诊台」：先用极轻的逻辑判断你想干什么，再只动态加载那条路径真正需要的代码。

最极端的例子就是 `--version`：

```ts
if (args.length === 1 && (args[0] === '--version' || args[0] === '-v' || args[0] === '-V')) {
  console.log(`${MACRO.VERSION} (Claude Code)`);
  return;
}
```

`MACRO.VERSION` 是构建期内联进来的字面量（第 17 章会讲 `MACRO.*` 和 feature flag 的构建期注入），所以这条路径**除了入口文件本身零 import**——按回车立刻返回版本号，不付任何加载代价。

## 3.2 一层层分诊：fast-path 的瀑布

`main()` 的主体是一长串 `if` 分支，每一个都对应一种「特殊启动模式」，命中就动态加载对应模块、跑完、`return`，绝不往下走。按代码顺序，这些 fast-path 包括：

- `--version` / `-v` / `-V`：零 import 返回版本号。
- `--dump-system-prompt`（受 `feature('DUMP_SYSTEM_PROMPT')` 门控，仅内部构建）：渲染并打印系统提示词后退出，给 prompt 敏感性评测用。
- `--claude-in-chrome-mcp` / `--chrome-native-host` / `--computer-use-mcp`（`feature('CHICAGO_MCP')`）：把自己当成一个 MCP server 跑（浏览器/桌面控制）。
- `--acp`（`feature('ACP')`）：以 Agent Client Protocol agent 模式跑在 stdio 上。
- `weixin` 子命令：微信渠道 CLI。
- `--daemon-worker=<kind>`（`feature('DAEMON')`）：supervisor 派生出来的精简 worker，注释特别说明这层「no enableConfigs(), no analytics sinks」——worker 要尽量瘦。
- `remote-control` / `rc` / `remote` / `sync` / `bridge`（`feature('BRIDGE_MODE')`）：把本机当作 bridge 环境。
- `daemon`、`environment-runner`、`self-hosted-runner` 等子命令各自的入口。

这一长串分支里反复出现一个组合拳：**`feature('X')` 在前，运行时 gate（如 `isBridgeEnabled()`）在后**。`cli.tsx` 里有段注释把这个分工讲得很清楚——`feature()` 必须留在内联位置以便构建期 dead-code elimination，而 GrowthBook 那类运行时 gate 负责动态开关。这是 CCB 处理「编译期能力裁剪」与「运行期灰度」的标准范式，贯穿全书。

每一个分支前都有一句 `profileCheckpoint('cli_xxx_path')`。CCB 用 `startupProfiler` 在启动链路上埋了一连串检查点，专门用来量化「冷启动到底慢在哪条路径」。这本身就是「可观测性从启动第一刻就在线」的体现。

当所有特殊 flag 都没命中，瀑布走到底部，才真正加载完整 CLI：

```ts
const { startCapturingEarlyInput } = await import('../utils/earlyInput.js');
startCapturingEarlyInput();
const { main: cliMain } = await import('../main.jsx');
await cliMain();
```

注意 `startCapturingEarlyInput()`——在加载笨重的 `main.jsx` 之前，先开始捕获用户的早期键盘输入，这样用户在启动那一两秒里敲的字不会丢。又是一个「为真实终端体验打磨」的细节。

## 3.3 main.tsx：Commander 接管，preAction 做重活

`cli.tsx` 把控制权交给 `src/main.tsx` 的 `main()`（定义在第 743 行，整个文件 5600+ 行）。这里换了一套机制——用 `commander` 构建完整的命令行界面：

```ts
const program = new CommanderCommand()
  .configureHelp(createSortedHelpConfig())
  .enablePositionalOptions();
```

关键设计是 `program.hook('preAction', ...)`：真正的初始化重活（MDM 检查、`init()`、analytics sink、migrations、远程 settings、settings 同步）都挂在 commander 的 `preAction` 钩子里，而不是在 `main()` 顶部无条件跑。注释点明了原因——只有在「确实要执行某个命令」时才初始化。如果你只是 `claude --help`，这些昂贵的初始化根本不会触发。这是 fast-path 哲学在第二层入口的延续。

`main.tsx` 注册了海量 options：`--print`/`-p`（headless）、`--worktree`、`--permission-mode`、`--model`、`--proactive`、`--brief`、`--assistant`，以及大量 `hideHelp()` 的内部选项（`--agent-id`、`--team-name`、`--parent-session-id` 等，用于多 agent 协同）。这些 option 的存在本身就勾勒出 CCB 的能力边界。

## 3.4 双驱动分流：REPL 还是 Headless

走到默认命令的 `.action(async (prompt, options) => {...})` 处理器，CCB 面临本章最核心的分叉——**这是一次人机交互会话，还是一次脚本化的批处理？** 判据就是 `-p` / `--print`：

```ts
if (process.argv.includes('-p') || process.argv.includes('--print')) { ... }
```

两条路通向两套完全不同的驱动：

**驱动一：交互式 REPL。** 默认（无 `-p`）走 `launchRepl(...)`（`src/replLauncher.tsx`）。它动态 import `./screens/REPL.js`，把 `<REPL>` 这个 React 组件挂载到 Ink（终端 React 渲染器）上：

```ts
export async function launchRepl(root, replProps, renderAndRun) {
  const { REPL } = await import('./screens/REPL.js');
  await renderAndRun(root, <SentryErrorBoundary name="RootREPLBoundary"><REPL {...replProps} /></SentryErrorBoundary>);
}
```

REPL 是一个有状态的 TUI：它管理输入框、消息流渲染、权限弹窗、中断（Ctrl-C），并在用户每次回车时驱动一轮 `query()`。**它是「人在回路」的驱动器**——权限确认要弹窗等人点、中断要响应键盘、消息可以中途插队（注释提到 REPL 输入默认 `'next'` 优先级，会在工具调用之间排空）。

**驱动二：Headless 引擎。** 命中 `-p` 时走 `runHeadless(...)`（`src/cli/print.ts`，`runHeadless` 定义在第 458 行）。它没有 TUI，从 stdin/参数拿 prompt，把结果以文本或结构化（NDJSON）流式吐到 stdout。注释里明确：headless 模式不支持 SSH 会话（`claude ssh` 需要本地 REPL 来驱动中断和权限）。

Headless 路径背后是 `src/QueryEngine.ts`（1365 行）。它的类注释写道「One QueryEngine per conversation. Each submitMessage() call starts a new...」——每个会话一个 `QueryEngine`，核心是异步生成器 `async *submitMessage(...)`（第 217 行）。它是 Agent SDK 的适配层：把外部「发一条消息、流式收回结果」的接口，翻译成内部 `query()` 主循环的调用。这里还有个细节——结构化输出重试次数由 `process.env.MAX_STRUCTURED_OUTPUT_RETRIES || '5'` 控制，默认 5 次。

## 3.5 两条路、一个内核

无论 REPL 还是 Headless，它们的分歧只在「**怎么和外界交换输入输出**」：一个对接终端 UI 和人的实时操作，一个对接 stdin/stdout 和脚本。再往里走，两者最终都汇聚到同一个内核——`src/query.ts` 的 `queryLoop`。REPL 在用户回车时调它，QueryEngine 在 `submitMessage` 里调它。

这正是好的 Agent 架构的标志：**驱动层（怎么和人/脚本打交道）和编排层（agent 怎么思考、调工具、压上下文）彻底解耦**。换一种前端——Web、IDE 插件、ACP、微信——只需要再写一个驱动，内核一行不用动。CCB 里那么多入口（`--acp`、`weixin`、`bridge`、`daemon-worker`）能共存，根因就在这层解耦。下一章，我们就推开内核的门，看 `queryLoop` 那两层 `while` 到底是怎么跳动的。

## 本章小结

- CCB 的真正入口是 `src/entrypoints/cli.tsx`，一个 `#!/usr/bin/env bun` 脚本；第一条 import 是 `performanceShim.js`，为长会话避免 JSC C++ Vector 无限增长——入口第一行就是「为长时运行设计」。
- 入口的核心哲学是 **all imports are dynamic**：用极轻逻辑分诊，只加载该路径需要的模块。`--version` 走零 import 快速路径，直接打印构建期内联的 `MACRO.VERSION`。
- `main()` 是一串 fast-path 瀑布：`--dump-system-prompt`、`--acp`、`weixin`、`--daemon-worker`、`remote-control`、`daemon` 等各自命中即 `return`；普遍采用「`feature('X')` 构建期裁剪 + 运行时 gate 灰度」的组合拳。
- 每条路径都有 `profileCheckpoint('cli_*_path')`，启动可观测性从第一刻在线；加载完整 CLI 前先 `startCapturingEarlyInput()` 防丢键。
- `src/main.tsx` 用 commander 接管，把昂贵的初始化挂在 `preAction` 钩子里——只有真要执行命令时才初始化，`--help` 不付代价。
- 双驱动分流以 `-p`/`--print` 为判据：默认走交互式 REPL（`launchRepl` → Ink 挂载 `<REPL>`，人在回路），`-p` 走 headless（`runHeadless` → `QueryEngine.submitMessage` 异步生成器，脚本在回路）。
- 两条驱动只在 I/O 交换方式上分叉，最终都汇聚到同一个内核 `src/query.ts` 的 `queryLoop`——驱动层与编排层彻底解耦，是 CCB 能支持多种前端的根因。

## 动手实验

1. **实验一：给 fast-path 计时** — 在 shell 里分别 `time claude --version` 和 `time claude --help`（或你本地等价命令），对比两者耗时。再打开 `cli.tsx` 数一数 `--version` 分支到 `return` 之间有几次 `await import`（答案是零），理解「零 import 快速路径」为何这么快。
2. **实验二：列出所有特殊启动模式** — 在 `src/entrypoints/cli.tsx` 的 `main()` 里搜索所有 `return;` 语句，逐个记录它前面的 `if` 条件（flag 名 + 是否被 `feature()` 门控）。你会得到一张完整的「CCB 能以哪些形态启动」清单。
3. **实验三：追一次交互启动的 checkpoint 链** — 在 `cli.tsx` 和 `main.tsx` 里搜索 `profileCheckpoint(` 和 `preAction`，按代码顺序列出一次默认（无 flag）启动会依次经过哪些检查点，画出从 `cli_entry` 到主循环点燃前的时间线。
4. **实验四：对比两条驱动的入口签名** — 打开 `src/replLauncher.tsx` 的 `launchRepl` 和 `src/cli/print.ts` 的 `runHeadless`，对比它们的参数与返回方式（一个挂载 React 组件，一个返回流）。再在 `src/QueryEngine.ts` 找到 `async *submitMessage`，确认 headless 最终也是在调主循环——验证 3.5 节「一个内核」的论断。

> **下一章预告**：门已推开，内核就在眼前。第 4 章我们将逐行解构 `src/query.ts` 的 `queryLoop`——那两层 `while(true)` 各自负责什么、一个 turn 的生命周期里上下文压缩与模型调用以什么顺序发生、`Terminal` 与 `Continue` 两种状态转移如何决定「再转一圈还是收工」。
