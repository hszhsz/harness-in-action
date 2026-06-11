# 第 3 章　从进程到运行时：一次请求的起点

> 我们终于要碰代码了。
> 这一章追踪一个最朴素的问题：当你在终端敲下 `openclaw` 回车，到"Agent 主循环开始转动"之前，这中间发生了什么？

第二部分的标题是"骨架：Agent 的执行内核"。但在解剖心脏（第 4 章的主循环）之前，我们得先看清楚血液是从哪里流进来的。本章走完三段路：**启动器 `openclaw.mjs` → 入口 `src/entry.ts` → 运行时门面 `src/agents/runtime`**，再补上"消息驱动"这条平行入口。读完你会明白一件对 Agent 工程很重要的事：**一个生产级 Agent 的"开机"过程，本身就是一套精心设计的工程。**

---

## 3.1 启动器 openclaw.mjs：被低估的第一道工序

很多人以为 CLI 工具的入口就是"导入主程序、跑起来"。但 OpenClaw 的启动器 `openclaw.mjs`（一个 661 行的纯 `.mjs` 文件，刻意不依赖任何 TypeScript 编译产物）做了远比这多的事。它是整个系统里**唯一一段不能假设"环境已就绪"的代码**——因为它运行时，连 TypeScript 运行时都还没起来。

我们按它的执行顺序拆解它的四项职责。

### 职责一：Node 版本守卫——把环境问题挡在最前面

文件开头就是一道硬门槛：

```js
const MIN_NODE_MAJOR = 22;
const MIN_NODE_MINOR = 19;
// ...
ensureSupportedNodeVersion();
```

如果当前 Node 版本低于 22.19，启动器立刻打印一段**带 nvm 修复指引**的错误信息并 `process.exit(1)`。

这是一个朴素但深刻的工程原则：**把环境前置条件检查放在最早、最便宜的地方**。如果让 TypeScript 运行时先加载、跑到一半才因为某个新语法在旧 Node 上崩溃，错误信息会面目全非、难以排查。OpenClaw 选择在第一行就用最原始的 JS 把这道关守住。

> **给 Agent 开发者的启示**：Agent 的"第一公里"（环境/依赖/版本）失败，是新用户流失的最大杀手之一。OpenClaw 把 "Setup reliability and first-run UX" 列为项目优先级，启动器的版本守卫就是这条优先级的物化。

### 职责二：版本/帮助快速通道——为常见命令省掉冷启动

紧接着，启动器检查一类特殊调用：

```js
if (tryOutputLauncherVersion(process.argv)) {
  process.exit(0);
}
```

如果用户只是敲 `openclaw --version`（或 `-v`/`-V`），启动器会**直接从 `package.json` / `dist/build-info.json` / git HEAD 里读出版本号打印出来，然后退出**——根本不加载庞大的 TypeScript 运行时。类似地，对 `--help` 和少数子命令（browser/secrets/nodes）的帮助，它会尝试读取预计算好的 `dist/cli-startup-metadata.json` 直接输出。

为什么值得为这点小事单独写代码？因为**冷启动成本**。一个有 142 个扩展、要初始化插件系统、连接 Gateway 的运行时，启动是有可观开销的。而 `--version`、`--help` 这类查询是高频调用（脚本探测、CI、用户试探）。把它们做成"快速通道"，绕开整个运行时，是体感性能的关键优化。

> 这其实是一个普适的架构思想：**识别"重路径"上的"轻请求"，给它们开一条旁路。** 你的 Agent 里也一定有这种"不需要唤醒整个大脑就能回答"的请求。

### 职责三：源码/产物分流与编译缓存

这是启动器最"硬核"的部分。OpenClaw 既可能以**源码 checkout**（开发者本地，有 `.git` 和 `src/entry.ts`）运行，也可能以**打包产物**（npm 安装，只有 `dist/`）运行。启动器要分辨自己处在哪种形态：

```js
const isSourceCheckoutLauncher = () =>
  existsSync(new URL("./.git", import.meta.url)) ||
  existsSync(new URL("./src/entry.ts", import.meta.url));
```

基于这个判断，它对 **Node 的编译缓存**（`module.enableCompileCache`）做精细管理：

- **打包产物**模式：启用编译缓存，缓存目录按"包版本 + 安装标记"分桶（`resolvePackagedCompileCacheDirectory`），让二次启动更快。
- **源码**模式：反而要**禁用**编译缓存（`respawnWithoutCompileCacheIfNeeded`），因为源码经常变动，缓存会帮倒忙。

更有意思的是，当发现当前进程的编译缓存配置"不对"时，启动器会**用正确的环境变量重新 spawn 一个子进程**（respawn），然后把信号转发、退出码透传都管理起来——`runRespawnedChild` 里那一大段 `SIGTERM`/`SIGKILL` 的优雅关闭逻辑就是为此服务。

> **启示**：分发形态的多样性（源码 / npm / Docker / 各平台）是真实工具必须面对的复杂度。把"我现在是什么形态"这个判断**集中在启动器一处**，下游代码就能假设环境已规整。这又是一次"把脏活集中处理"的体现，和第 2 章那条"配置只认当前 Schema"的红线异曲同工。

### 职责四：最终入口分发

走完上面所有前置处理，启动器才真正把控制权交出去。注意它**永远只加载打包产物**：

```js
await installProcessWarningFilter();
if (await tryImport("./dist/entry.js")) {
  // OK
} else if (await tryImport("./dist/entry.mjs")) {
  // OK
} else {
  throw new Error(await buildMissingEntryErrorMessage());
}
```

如果连 `dist/entry.js` 都找不到，它会调用 `buildMissingEntryErrorMessage()` 给出**有温度的报错**——它甚至能识别出"你这是一个没构建的源码树或 GitHub 源码包"，并精确建议 `pnpm install && pnpm build` 或改用 `npm install -g openclaw@latest`。

**小结**：`openclaw.mjs` 本身不包含任何 Agent 逻辑。它是一道"工序前处理"——校验环境、开快速通道、规整运行形态、给出友好报错，最后才把接力棒交给 `entry.ts`。这种"启动器与运行时分离"的设计，让运行时代码得以在一个干净、已规整的环境里开始工作。

---

## 3.2 入口 src/entry.ts：从规整环境到 CLI 分发

控制权来到 `src/entry.ts`（309 行）。如果说 `openclaw.mjs` 是"开机自检"，那 `entry.ts` 就是"操作系统引导"——它在 TypeScript 运行时里做最终的环境规整，然后把请求分发给 CLI。

### 一个容易被忽略的陷阱：主模块守卫

`entry.ts` 开头有一段非常关键的防御：

```ts
if (
  !isMainModule({
    currentFile: fileURLToPath(import.meta.url),
    wrapperEntryPairs: [...ENTRY_WRAPPER_PAIRS],
  })
) {
  // Imported as a dependency — skip all entry-point side effects.
} else {
  // ... 真正的入口逻辑
}
```

为什么需要它？因为打包器可能把 `entry.js` 当作 `dist/index.js` 的共享依赖来导入。如果没有这道守卫，顶层代码会被执行第二次，**启动两个 Gateway**，第二个会因为端口/锁冲突而崩溃。代码注释把这个坑解释得很清楚。

> **启示**：在 Node 生态里，"这个文件是被直接运行，还是被当作模块导入"是一个真实存在、且会造成诡异 bug 的区别。任何有"运行时副作用"的入口文件都应该有这样一道守卫。

### 入口逻辑的执行顺序

进入 `else` 分支后，`entry.ts` 按顺序做了这些事（每一步都对应一个真实的工程考量）：

1. **再做一次编译缓存检查/respawn**（`respawnWithoutOpenClawCompileCacheIfNeeded`）——注意这和启动器里的逻辑"有意重叠"，代码注释明确说"keep the respawn supervision behavior in sync until the launcher can share TS code"。即：因为启动器不能用 TS，所以这段逻辑在两处各写了一遍，靠注释提醒维护者同步。
2. **设置进程标题** `process.title = "openclaw"`、安装警告过滤器、规整环境变量 `normalizeEnv()`。
3. **特殊命令的环境预设**：比如检测到 `secrets audit` 就强制 `OPENCLAW_AUTH_STORE_READONLY=1`（安全默认值的又一例）；检测到 `--no-color` 就设好 `NO_COLOR`。
4. **CLI respawn 计划**（`ensureCliRespawnReady`）——某些情况下 CLI 需要在调整后的参数/环境下重启自己。
5. **解析容器目标与 profile 参数**（`--container` / `--profile` / `--dev`），并处理它们的互斥关系。
6. **版本快速通道兜底**（`tryHandleRootVersionFastPath`），最后调用 `runMainOrRootHelp(process.argv)`。

### 真正的分发点：runMainOrRootHelp

这个函数是入口的终点，也是运行时的起点：

```ts
async function runMainOrRootHelp(argv: string[]): Promise<void> {
  if (await tryHandleRootHelpFastPath(argv)) return;
  if (await tryHandlePrecomputedCommandHelpFastPath(argv)) return;
  try {
    const { runCli } = await gatewayEntryStartupTrace.measure(
      "run-main-import",
      () => import("./cli/run-main.js"),
    );
    await runCli(argv);
  } catch (error) {
    // ... 格式化友好的失败信息后 process.exit(1)
  }
}
```

注意三个细节：

- **帮助快速通道在这里又出现了**——这次是带"实时配置感知"的版本（因为插件可能改变帮助内容），与启动器里的"预计算"版本形成两级。
- **`runCli` 是动态 `import` 的**（`await import("./cli/run-main.js")`）。这不是随意为之：动态导入意味着只有在真正需要跑命令时，才加载庞大的 CLI 模块图，进一步压缩冷启动。
- **启动耗时被 `gatewayEntryStartupTrace.measure` 包裹**——OpenClaw 在入口处就埋了启动性能追踪（受 `OPENCLAW_GATEWAY_STARTUP_TRACE` 环境变量控制）。这是第 1 章"可观测性"职责在最早期的体现：连"开机有多慢"都要可观测。

到这里，命令被分发到 `src/cli/`，由 Commander 风格的命令系统接管。对于一次真正要"跑 Agent"的调用（而非 `--help`、`doctor` 这类），控制权会进一步流向运行时。

---

## 3.3 运行时门面 src/agents/runtime：依赖注入的接缝

现在我们到了本章最该停下来细看的地方。OpenClaw 把"可复用的 Agent 核心"抽成了独立 package `packages/agent-core`（下一章的主角），而 `src/agents/runtime/` 的职责，就是**把这个通用核心"接"到 OpenClaw 自己的世界里**。

打开 `src/agents/runtime/index.ts`，它出奇地短，但每一行都关键：

```ts
import { Agent as CoreAgent, type AgentOptions as CoreAgentOptions }
  from "../../../packages/agent-core/src/agent.js";
import type { AgentCoreRuntimeDeps } from "../../../packages/agent-core/src/runtime-deps.js";
import type { CompleteSimpleFn, StreamFn } from "../../../packages/llm-core/src/index.js";
import { completeSimple, streamSimple } from "../../plugin-sdk/llm.js";

export const openClawAgentCoreRuntime = {
  completeSimple: completeSimple as unknown as CompleteSimpleFn,
  streamSimple: streamSimple as unknown as StreamFn,
} satisfies AgentCoreRuntimeDeps;

export class Agent extends CoreAgent {
  constructor(options: CoreAgentOptions = {}) {
    super({ runtime: openClawAgentCoreRuntime, ...options });
  }
}
```

这段代码是**依赖注入（Dependency Injection）**的教科书式范例，值得 Agent 开发者反复琢磨：

- `packages/agent-core` 里的 `CoreAgent` **不知道**自己将来用哪个模型 provider。它只声明了一个契约 `AgentCoreRuntimeDeps`：给我两个函数——`streamSimple`（流式生成）和 `completeSimple`（一次性补全）——我就能转动起来。
- `src/agents/runtime` 这个"门面（facade）"负责把 OpenClaw 真正的 LLM 实现（来自 `plugin-sdk/llm.js`）**注入**进去，然后导出一个绑定好依赖的 `Agent` 子类。
- 于是 OpenClaw 的其余代码只需 `import { Agent } from "agents/runtime"`，拿到的就是一个"已经接好模型能力"的 Agent。

**为什么这个接缝如此重要？** 因为它把"Agent 的执行逻辑"和"模型怎么调"彻底解耦了：

| 这一侧（agent-core） | 那一侧（OpenClaw runtime） |
|---|---|
| 关心：怎么转循环、怎么调工具、怎么管上下文 | 关心：用哪个 provider、怎么流式、怎么计费 |
| 完全不知道 Anthropic/OpenAI 的存在 | 通过 `plugin-sdk/llm.js` 对接真实 provider |
| 可被任何宿主复用、可单独测试 | 可独立替换模型层而不动核心 |

这正是第 2 章"核心精简"红线在架构层面的落地：**通用核心保持纯净，宿主通过注入提供具体能力。** 你将在后续章节反复看到这种"core 定义契约、宿主注入实现"的模式。

### 运行时选择：openclaw / auto

官方架构文档点明了运行时是可选择的：默认内置运行时 id 是 `openclaw`；插件 harness 可以注册额外的运行时 id；而 `auto` 会在存在合适的插件 harness 时选它、否则回落到内置的 OpenClaw 运行时[[Agent runtime architecture]](https://github.com/openclaw/openclaw/blob/main/docs/agent-runtime-architecture.md)。换句话说，连"用哪套运行时"本身都是一个可插拔的决策。

### OpenClaw 专属驱动：embedded-agent-runner

`agent-core` 提供的是"裸核心"。真正把它驱动起来、并补上 OpenClaw 特有能力的，是 `src/agents/embedded-agent-runner/`。官方文档对它的定义是："built-in agent attempt loop, provider stream adapters, compaction, model selection, and session wiring"（内置 Agent 尝试循环、provider 流式适配、压缩、模型选择、会话接线）[[Agent runtime architecture]](https://github.com/openclaw/openclaw/blob/main/docs/agent-runtime-architecture.md)。

这个目录里光是和"压缩（compaction）"相关的文件就有十几个（`compact.ts`、`compaction-hooks.ts`、`compaction-safety-timeout.ts`……）——这预示了第 13 章会有大量内容。现在你只需记住它的定位：**它是 agent-core 与 OpenClaw 真实世界之间的"驱动层"。**

---

## 3.4 另一条入口：消息驱动路径

到目前为止我们走的是"用户在终端敲命令"这条路。但别忘了 OpenClaw 的定位是"in your channels"——它同样能被一条 Discord/Telegram 消息**驱动**。这条平行入口的核心在 `src/auto-reply/reply/agent-runner.ts`。

这个文件相当大（约 85KB），它的开头一句注释道出了职责：

> "Orchestrates reply agent execution, payload building, and delivery callbacks."（编排回复 Agent 的执行、载荷构建与投递回调。）

它和命令行入口的关键区别在于：命令行是"一次性请求—响应"，而消息驱动要处理**会话连续性、投递回执、排队**等问题。从它的 import 就能看出端倪——它牵涉 `resolveSessionAgentId`（会话→Agent 解析）、`queueEmbeddedAgentMessageWithOutcomeAsync`（把消息排队进嵌入式 Agent）、`normalizeUsage`（用量归一）、`loadSessionStore`（加载会话存储）等一大批模块。

我们暂时不深入它的细节（投递与渠道是第 20 章的内容）。这里要建立的认知是：

> **同一个 Agent 内核，可以被多种"入口"驱动。** 命令行是同步的、一次性的；消息渠道是异步的、持续的。它们最终都汇流到同一套运行时（`agents/runtime` + `embedded-agent-runner`）。一个设计良好的 Agent，应该把"怎么被触发"（入口/渠道）和"被触发后怎么干活"（运行时/循环）清晰地分开。

---

## 3.5 把这一章连起来：一次请求的完整起点

让我们把本章追踪的链路画成一张图。这是全书执行链路的"第一段"：

```
                     ┌─────────────────────────────────────────┐
  $ openclaw ...      │  openclaw.mjs (纯 JS 启动器, 不依赖 TS)    │
        │             │  1. Node 版本守卫                          │
        └────────────▶│  2. version/help 快速通道                  │
                      │  3. 源码/产物分流 + 编译缓存 (+respawn)     │
                      │  4. import dist/entry.js                  │
                      └───────────────────┬─────────────────────┘
                                          ▼
                      ┌─────────────────────────────────────────┐
                      │  src/entry.ts (TS 运行时入口)              │
                      │  · isMainModule 守卫 (防双 Gateway)        │
                      │  · normalizeEnv / 进程标题 / 安全默认值      │
                      │  · profile/container 解析                  │
                      │  · runMainOrRootHelp → import run-main.js  │
                      │  · runCli(argv)  ← 启动性能被 trace 包裹    │
                      └───────────────────┬─────────────────────┘
                                          ▼
        命令行路径                  ┌──────────────┐        消息驱动路径
  (src/cli/* 命令系统) ───────────▶│  agents/runtime │◀─── (auto-reply/reply/
                                  │   (DI 门面)      │       agent-runner.ts)
                                  │ 注入 streamSimple│
                                  │   /completeSimple│
                                  └───────┬──────────┘
                                          ▼
                              ┌────────────────────────┐
                              │ embedded-agent-runner    │
                              │ (OpenClaw 驱动层)         │
                              │ → packages/agent-core    │
                              │   的主循环 (第 4 章)       │
                              └────────────────────────┘
```

这张图里藏着三个贯穿全书的工程思想，请记牢：

1. **分层规整环境**：从纯 JS 启动器到 TS 入口，每一层只负责把环境向"更规整"推进一步，让下游能做更强的假设。
2. **依赖注入解耦**：`agents/runtime` 用一个极小的门面，把"通用核心"和"具体模型能力"焊在一起，却互不知道对方的实现。
3. **入口与内核分离**：命令行和消息渠道是两条入口，但共享同一套运行时内核。

下一章，我们将推开 `agents/runtime` 这道门，走进 `packages/agent-core`——去看那个被注入了模型能力之后、终于开始转动的 **Agent 主循环**。

---

## 本章小结

- `openclaw.mjs` 是一个不依赖 TS 的纯 JS 启动器，承担四项职责：Node 版本守卫、version/help 快速通道、源码/产物分流与编译缓存管理（含 respawn）、最终入口分发，并提供有温度的报错。
- `src/entry.ts` 在 TS 运行时里完成最终环境规整，关键是 `isMainModule` 守卫（防止重复启动 Gateway）与动态 `import` 的 `runCli`，并在入口处就埋了启动性能追踪。
- `src/agents/runtime/index.ts` 是依赖注入的核心接缝：把通用的 `packages/agent-core` 与 OpenClaw 的真实 LLM 实现（`plugin-sdk/llm.js`）焊接成一个绑定好依赖的 `Agent`，实现"核心不知道 provider、宿主注入实现"的解耦。
- `embedded-agent-runner` 是 agent-core 之上的 OpenClaw 专属驱动层（循环、流式适配、压缩、模型选择、会话接线）。
- 命令行与消息渠道是两条平行入口，但都汇流到同一套运行时内核——"入口与内核分离"是值得借鉴的架构原则。

---

> **动手实验（建议在读第 4 章前完成）**
> 1. 在 clone 的仓库里运行 `node openclaw.mjs --version`，再运行 `OPENCLAW_GATEWAY_STARTUP_TRACE=1 node openclaw.mjs gateway ...`（参考其帮助），观察启动追踪输出，体会"快速通道"与"完整运行时"两条路径的差异。
> 2. 阅读 `src/agents/runtime/index.ts` 全文（只有 ~30 行），然后到 `packages/agent-core/src/runtime-deps.ts` 看 `AgentCoreRuntimeDeps` 的类型定义。用你自己的话回答：如果要把 agent-core 移植到一个完全不同的宿主（比如一个 Web 服务），你最少需要注入哪几个函数？
