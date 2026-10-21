# 第 2 章　pi 全景：十一个 workspace，三条接缝

> 一个好的 monorepo 结构，本身就是一份架构文档。但当仓库长大时，先要确认你读的是哪一版「文档」。

上一版电子书写道：pi 是一个 pnpm monorepo，由四个包组成。这个描述在 0.79 时代成立，但在 1.3 已经过时。今天 pi 的 `packages/` 下有 **11 个 workspace**，使用 **npm workspaces + `package-lock.json`** 管理，所有包版本统一为 `1.3.0`。本章先把这张地图画准，再抽出贯穿全书的三个接缝。

---

## 2.1 从四个包到十一个 workspace

| 目录 | npm 包名 | 职责 | 一句话定位 |
|---|---|---|---|
| `packages/ai` | `@earendil-works/pi-ai` | 多 Provider LLM 运行时、模型目录、认证、流协议 | 「怎么跟模型说话」 |
| `packages/agent` | `@earendil-works/pi-agent-core` | Agent 循环、`Agent` 状态机、工具执行管线 | 「怎么把对话跑成任务」 |
| `packages/coding-agent` | `@earendil-works/pi-coding-agent` | 会话编排、工具、扩展、CLI / RPC / SDK | 「怎么变成一个产品」 |
| `packages/tui` | `@earendil-works/pi-tui` | 自研终端 UI、差分渲染、组件库 | 「怎么显示给人看」 |
| `packages/codemode` | `@earendil-works/pi-codemode` | QuickJS WASM 代码执行沙箱 | 「用写代码代替逐个调工具」 |
| `packages/mcp` | `@earendil-works/pi-mcp` | MCP 协议客户端（stdio / HTTP / OAuth） | 「接入外部工具生态」 |
| `packages/durable` | `@earendil-works/pi-durable` | 先提交后展示的持久化 Agent 运行时（实验） | 「进程挂了也能接着跑」 |
| `packages/chord` | `@earendil-works/chord` | 可热重载对象、契约、复制状态、RPC | 「跨进程的有状态对象」 |
| `packages/env` | `@earendil-works/pi-env` | SSH 远端执行环境 + Rust daemon | 「把工具执行搬到另一台机器」 |
| `packages/telemetry` | `@earendil-works/pi-telemetry` | 厂商中立的遥测契约与适配器 | 「可观测性不绑定后端」 |
| `packages/evals` | `@earendil-works/pi-evals`（private） | 评测与文档一致性测试 | 「用真实任务验收 Agent」 |

根 README 只列出了其中 7 个面向外部用户的包；`mcp`、`codemode` 是 `coding-agent` 的实现依赖，`env` 服务于 `durable`，`evals` 是私有评测工程。**读源码时不要被根 README 的表格限制**。

依赖关系（箭头表示「前者依赖后者」）：

```
telemetry ←── ai ←── agent ←── coding-agent ──→ tui
     ↑                          ├──→ codemode
     │                          └──→ mcp
     ├── chord ←── durable ←── env
     └──────────────┘
```

几个容易误判的点：

- `pi-agent-core` 只依赖 `pi-ai` 的类型与少量工具函数，**不依赖任何 provider 实现**，也不依赖 `coding-agent`。
- `pi-coding-agent` 是唯一的「组装厂」：模型运行时、工具、会话、扩展、TUI 都在这里汇合。
- `durable` 不复用 `coding-agent`，它有自己的 harness、会话和工具抽象；`env` 只服务 `durable`。**不要把 `durable` 当成「新版 AgentSession」**，这是两条独立的产品线，第 17 章会讲。

> **给 Agent 开发者的启示**：包的数量不是重点，重点是**依赖方向是否允许你替换任意一层**。pi 的切法保证了：模型层不知道会话，内核不知道存储，UI 不知道业务。你可以只拿走 `pi-ai` 做网关，也可以只拿走 `pi-agent-core` 做嵌入式循环。

## 2.2 接缝一：`StreamFn` —— 内核与模型的边界

如果整本书只让你记住一个设计，请记住这个：**Agent 循环不依赖 `Models`，只依赖一个函数签名。**

`packages/agent/src/types.ts:33`：

```ts
export type StreamFn = (
	model: Model<Api>,
	context: TranscriptContext,
	options?: SimpleStreamOptions,
) => AssistantMessageEventStream | Promise<AssistantMessageEventStream>;
```

相比旧版，这里有两个重要变化：

1. **不再用 `Parameters<typeof streamSimple>`「借用」类型**。`StreamFn` 是显式声明的独立契约，`pi-agent-core` 因此可以彻底不依赖具体门面函数。
2. **第二个参数是 `TranscriptContext`，而不是旧的 `Context`**。这是 1.3 最核心的架构变化：系统提示词和工具声明不再是 `Context.systemPrompt` / `Context.tools` 这样的旁路字段，而是作为 `SystemMessage` 存在于 `messages` 转录中。`normalizeContext()` 是唯一能构造 `TranscriptContext` 的入口，防止未归一化的上下文流入 provider。

那么默认实现从哪来？`packages/agent/src/stream-fn.ts` 只有一个模块级变量：

```ts
let defaultStreamFn: StreamFn | undefined;

export function setDefaultStreamFn(streamFn: StreamFn | undefined): void {
	defaultStreamFn = streamFn;
}

export function getDefaultStreamFn(): StreamFn {
	if (!defaultStreamFn) {
		throw new Error("No default stream function configured. Pass streamFn explicitly or call setDefaultStreamFn().");
	}
	return defaultStreamFn;
}
```

宿主（`pi-coding-agent` 或你自己的应用）在启动时把 `Models` 的流函数装进去；内核在调用方没传 `streamFn` 时使用它。这条接缝带来三个直接好处：

| 好处 | 含义 |
|---|---|
| **provider 无关** | 内核不知道 Anthropic 还是 OpenAI，只知道「有人会给我一个异步事件流」 |
| **可测试** | 注入 `faux` provider 的流函数，整个循环不联网、不烧 token |
| **可远程化** | `packages/agent/src/proxy.ts` 就是一个把模型调用转发到服务端的 `StreamFn`，内核无感 |

契约里还有一句必须背下来的话：**失败不许抛异常，必须编码进返回的流**，最终落成 `stopReason: "error"` 或 `"aborted"` 的 `AssistantMessage`。这是 pi 错误处理哲学的源头，第 4 章展开。

## 2.3 接缝二：`AgentTool.execute` —— 内核与工具的边界

内核不认识「读文件」「跑 shell」，只认识 `AgentTool`：

```ts
export interface AgentTool<TParameters extends TSchema = TSchema, TDetails = any> extends Tool<TParameters> {
	label: string;
	prepareArguments?: (args: unknown) => Static<TParameters>;
	outputSchema?: TSchema;
	execute: (
		toolCallId: string,
		params: Static<TParameters>,
		signal?: AbortSignal,
		onUpdate?: AgentToolUpdateCallback<TDetails>,
	) => Promise<AgentToolResult<TDetails>>;
	replay?: "never" | "safe";
	executionMode?: ToolExecutionMode;
}
```

一个工具只承诺四件事：参数 schema、执行函数、流式更新回调、结果形状。至于「参数从哪来」「结果怎么渲染」「执行后端是本地还是容器」，内核一概不管。`coding-agent` 在这一层之上又包了一层 `ToolDefinition`，增加了暴露策略、系统提示词片段、自定义渲染等产品化字段（第 8 章）。

## 2.4 接缝三：`SessionManager` —— 编排与存储的边界

`AgentSession` 不自己写文件，它持有一个 `SessionManager`。`SessionManager` 暴露的都是领域操作，而不是 `read/write`：

```ts
appendMessage(message)
appendCompaction(summary, firstKeptEntryId, tokensBefore, details, fromExtension, usage)
appendBranchSummary(...)
appendContextEdit(targetId, replacement)
branch(fromId)
getBranch(fromId?)
buildSessionProjection()
```

因此「会话存 JSONL 还是数据库」「压缩摘要长什么样」「分支如何复制」都可以独立演进，而编排层只面对「追加条目 + 投影出上下文」这两个动作。第 11 章会看到，这套抽象如何让**压缩**和**回溯**变成同一棵树上的两种操作。

## 2.5 四条设计红线

1. **核心精简（lean core）**。`pi-agent-core` 只做机制：循环、队列、工具管线、事件。它不内置工具，不知道权限，不知道会话文件。
2. **约束外包（outsourced constraints）**。准入通过 `beforeToolCall` 与扩展的 `tool_call` 事件；隔离通过容器、沙箱、项目信任。核心只提供挂点。
3. **转录即状态（transcript as state）**。系统提示词、工具增删、上下文编辑都是转录上的事件，而不是外部可变配置。这样「模型看到的」与「会话记录的」天然一致，重放才可靠。
4. **错误即数据（errors as data）**。模型失败、工具失败都变成消息或结果，继续参与循环，而不是用异常炸掉整条链路。

## 2.6 如何高效阅读这套代码

- **先读类型，再读实现**。`packages/agent/src/types.ts` 与 `packages/ai/src/types.ts` 的注释本身就是设计文档。
- **抓入口**：`packages/coding-agent/src/main.ts` 负责模式解析与运行时装配；`core/agent-session.ts` 是真正的「产品大脑」。
- **抓最小闭环**：`agent-loop.ts` 的 `runLoop` + `agent.ts` 的 `Agent` + `stream-fn.ts`，三份文件就能跑通一个不依赖 provider 的 Agent。
- **用 `faux` 验证理解**：`packages/ai/src/providers/faux.ts` 是确定性模型，适合观察请求/响应的真实数据形态。
- **读文档目录**：`packages/coding-agent/docs/` 有 40 余篇面向使用者的文档，`how-pi-works.md`、`session-format.md`、`compaction.md` 与源码高度对应。

下一章，我们推开内核的门，看 `runLoop` 如何第一次跳动。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/agent/src/types.ts` | `StreamFn`、`AgentLoopConfig`、`AgentTool` 等核心契约 |
| `packages/agent/src/stream-fn.ts` | 默认流函数的注入点 |
| `packages/agent/src/index.ts` | `pi-agent-core` 公共 API |
| `packages/ai/src/models.ts` | `createModels` / `Models` / `Provider` |
| `packages/coding-agent/src/core/agent-session.ts` | 产品级编排层 |
| `packages/coding-agent/src/core/session-manager.ts` | 会话树与投影 |
| `packages/coding-agent/docs/how-pi-works.md` | 官方架构说明，可与本章对照 |
| `AGENTS.md` | 开发规范；pi 用自己开发自己 |

## 动手实验

1. 用 `node -e` 遍历 `packages/*/package.json`，打印每个包的 `dependencies` 中出现的 `@earendil-works/*`，画出真实依赖图，并与 2.1 节的图对照。
2. 打开 `packages/agent/src/types.ts`，把 `StreamFn`、`AgentTool.execute`、`AgentToolResult` 三段注释翻译成自己的话。重点回答：为什么工具执行失败要返回 `isError: true` 的结果，而不是抛异常？
3. 在仓库根执行 `npm run check:entry-graphs`，看看 pi 如何用脚本约束包之间的入口依赖——这是「架构即测试」的一个小例子。
