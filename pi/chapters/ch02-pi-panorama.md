# 第 2 章　pi 全景：四个包如何各司其职

> 一个好的 monorepo 结构，本身就是一份架构文档。

上一章我们建立了「Agent = Model + Harness」的认知框架，并点明 pi 的独特立场——把约束外包、核心保持精简。这一章，我们把镜头拉到最高处，俯瞰 pi 的整体结构：四个包分别是什么、为什么这样切、它们之间最关键的那条接缝在哪里。理解了这张全景图，后面每一章你都能知道「自己现在站在系统的哪个位置」。

---

## 2.1 monorepo 结构：四个包，四种职责

pi 是一个 pnpm monorepo，`packages/` 下只有四个包。它们的职责切得非常干净：

| 包 | npm 名 | 职责 | 一句话定位 |
|---|---|---|---|
| `packages/ai` | `@earendil-works/pi-ai` | 多 Provider LLM 抽象 | 「怎么跟模型说话」 |
| `packages/agent` | `@earendil-works/pi-agent-core` | Agent 运行时与 Harness 子系统 | 「怎么把对话变成自主任务」 |
| `packages/coding-agent` | `@earendil-works/pi-coding-agent` | 编码 Agent CLI、内置工具、扩展系统 | 「怎么把内核变成一个能用的编码助手」 |
| `packages/tui` | `@earendil-works/pi-tui` | 自研终端 UI | 「怎么把这一切显示给人看」 |

这四个包的依赖关系是单向的、分层的：

```
pi-tui  ←──────────────┐
                        │
pi-ai  ←──  pi-agent-core  ←──  pi-coding-agent  ──→  pi-tui
（模型层）    （内核层）         （应用层）          （UI 层）
```

- `pi-ai` 在最底层，不依赖任何其他 pi 包。它只关心「如何把一次请求发给 Anthropic / OpenAI / Google，并把响应变成统一的流」。
- `pi-agent-core` 依赖 `pi-ai` 的**类型**，但**不依赖它的具体实现**——这是全书最重要的一条接缝，下一节细说。
- `pi-coding-agent` 是应用层，它把内核、工具、扩展、会话、配置组装成一个真正能跑的 CLI，并依赖 `pi-tui` 来渲染界面。
- `pi-tui` 是一个独立的、自研的终端 UI 库（不用 React/ink），任何包都能用它，但它不反向依赖业务逻辑。

> **给 Agent 开发者的启示**：很多 Agent 项目一开始就把「调模型」「跑循环」「画界面」糊在一个文件里。pi 用四个包强制划出了边界。这种边界不是洁癖，而是**可测试性**和**可复用性**的前提——你可以单独测 `pi-agent-core` 而完全不联网，因为它根本不知道真实的模型长什么样。

## 2.2 一条最关键的接缝：`streamSimple` 注入与 provider 无关

如果整本书只让你记住一个设计，请记住这一个：**`pi-agent-core` 不依赖任何具体的模型实现，它只依赖一个函数签名。**

打开 `packages/agent/src/types.ts` 开头，你会看到 Agent 循环要求的那个核心契约：

```ts
import type {
	// ...
	SimpleStreamOptions,
	streamSimple,
} from "@earendil-works/pi-ai";

/**
 * Stream function used by the agent loop.
 *
 * Contract:
 * - Must not throw or return a rejected promise for request/model/runtime failures.
 * - Must return an AssistantMessageEventStream.
 * - Failures must be encoded in the returned stream via protocol events and a
 *   final AssistantMessage with stopReason "error" or "aborted" and errorMessage.
 */
export type StreamFn = (
	...args: Parameters<typeof streamSimple>
) => ReturnType<typeof streamSimple> | Promise<ReturnType<typeof streamSimple>>;
```

注意这里的精妙：`StreamFn` 的类型直接「借用」了 `pi-ai` 的 `streamSimple` 函数的参数和返回值类型（`Parameters<>` / `ReturnType<>`），但 Agent 循环**真正运行时并不强制使用那个实现**。在 `agent-loop.ts` 里，每次需要调模型时是这样写的：

```ts
const streamFunction = streamFn || streamSimple;
```

也就是说，调用方可以传入任何一个**符合 `StreamFn` 契约**的函数来替换真实的模型调用。这条接缝带来三个直接好处：

| 好处 | 含义 |
|---|---|
| **provider 无关** | 内核不知道 Anthropic 还是 OpenAI，只知道「有人会给我一个能吐事件流的函数」 |
| **可测试** | 测试时注入一个返回预设事件流的假 `streamFn`，整个循环逻辑不联网、不烧 token 就能测 |
| **可远程化** | 模型调用可以被代理、被缓存、被重路由，内核完全无感 |

而那句注释里的契约更值得反复读：**「Must not throw ... Failures must be encoded in the returned stream」**——失败不许抛异常，必须编码进返回的流里，最终落成一个 `stopReason: "error"` 的消息。这是 pi 整个错误处理哲学的源头，我们在第 4 章会专门讲「错误即数据」。

## 2.3 设计红线：核心精简、约束外包、状态用 JSONL 树

俯瞰完结构，我们把 pi 的几条「工程红线」提炼出来。这些原则在后面每一章都会反复印证：

1. **核心精简（lean core）**。`pi-agent-core` 只做机制，不做策略。它不知道什么是「危险命令」，不内置任何工具，甚至不知道会话要存到哪。所有这些都通过参数、钩子、注入交给上层。

2. **约束外包（outsourced constraints）**。如第 1 章所述，pi 核心不内置权限系统。准入控制通过 `beforeToolCall` 钩子和扩展的 `tool_call` 事件实现，沙箱通过容器和可替换的工具执行后端（Operations）实现。

3. **状态用 JSONL 树（session as a tree）**。pi 不用 SQLite，会话被建模成一棵 append-only 的 JSONL 树：每条消息、每次模型切换、每次压缩，都是树上的一个节点；「当前对话」是从某个叶子（leaf）回溯到根的一条路径。这让分支、回溯、压缩都变成了树操作。详见第 11 章。

4. **错误即数据（errors as data）**。模型请求失败、工具执行失败，都不以异常的形式冒泡打断循环，而是被建模成正常的消息或工具结果，喂回给模型或交给上层决策。

5. **事件驱动观测（event-driven observability）**。循环内部的一切都通过 `AgentEvent` / 扩展事件以流的形式暴露，UI、日志、扩展都只是这些事件的消费者。

## 2.4 如何高效阅读这套代码

pi 的代码量不小，但有几个「抓手」能让你快速上手：

- **从 `packages/agent/src/index.ts` 看公共 API 表**。这个文件把 `pi-agent-core` 对外导出的所有东西列了一遍——`agent.ts`、`agent-loop.ts`、`harness/*`——相当于一张「内核功能目录」。
- **从 `types.ts` 入手而非实现**。pi 的类型文件注释极其详尽（很多注释本身就是设计文档，比如上一节那段 `StreamFn` 契约）。先读类型，再读实现，事半功倍。
- **`AGENTS.md` 是给「开发 pi 的人」（包括 Agent 自己）的规范**。pi 用自己来开发自己（dogfooding），所以这个文件既是人类贡献者的规范，也是 pi 跑在自己仓库里时读到的项目说明。提交信息遵循 `{feat,fix,docs}[(scope)]: msg` 格式——你在 `git log` 里能直接看到这种风格。
- **善用 `faux` provider**。`packages/ai/src/providers/faux.ts` 是一个假的模型实现，专门用于测试。想理解「一次模型调用进出的数据长什么样」，读它比读真实 provider 更快。

读代码的顺序，本书已经替你排好了：接下来的第二部分，我们直接推开内核的大门，看那颗心脏——`runLoop`——是如何第一次跳动的。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/{ai,agent,coding-agent,tui}/` | 四个包，对应模型层 / 内核层 / 应用层 / UI 层 |
| `packages/agent/src/types.ts` | `StreamFn` 契约——内核与模型解耦的接缝 |
| `packages/agent/src/index.ts` | `pi-agent-core` 公共 API 目录 |
| `packages/ai/src/stream.ts` | `streamSimple` 等门面函数的真实实现 |
| `packages/ai/src/providers/faux.ts` | 测试用假 provider，理解数据形态的捷径 |
| `AGENTS.md` | 开发规范，dogfooding 的证据 |

## 动手实验

1. 画出 pi 四个包的依赖关系图（可以用 `grep -r "from \"@earendil-works" packages/*/src` 看每个包 import 了哪些兄弟包来验证你的图）。确认 `pi-agent-core` 是否真的不 import `pi-ai` 的具体 provider 实现。
2. 打开 `packages/agent/src/types.ts`，把 `StreamFn` 那段注释里的「契约」翻译成你自己的话讲给同事听：为什么「失败必须编码进流而不是抛异常」对一个 Agent 循环如此重要？（提示：想想如果第 5 个工具调用时模型 API 超时了，你希望前 4 个工具的结果怎么办。）
