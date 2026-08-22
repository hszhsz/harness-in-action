# 第 5 章　Session 事件日志：append-only 的唯一真相源

绝大多数 Agent 把"会话状态"存成快照，dsh 把它存成**事件日志**。这一章讲清楚：为什么日志是唯一真相源，模型历史为什么是日志的投影，以及"Model-visible ⟺ logged"这条硬不变量。核心源码在 `packages/core/session/src/types.ts`。

## `SessionEventMap`：事件词汇表

dsh 的核心会话模型在 `packages/core/session/src/types.ts`：一个 `Session` 是**类型化 `SessionEvent` 的 append-only 日志**。LLM 消息历史从日志**派生**，绝不单独存储。

事件类型通过 **merge-extensible map**（声明合并）组织：`SessionEventMap` 是核心词汇表，插件可以往里面加自己的事件。核心事件（源码原文）：

```ts
export interface SessionEventMap {
  'turn/start': { turn: number }
  'turn/end': { turn: number; reason: TurnEndReason }
  'step/start': { turn: number; step: number }
  'step/end': { turn: number; step: number }
  'user/message': UserMessage
  'assistant/chunk': { turn: number; step: number; chunk: StreamChunk }
  'assistant/message': { turn: number; step: number; message: AssistantMessage; usage?: TokenUsage; interrupted?: true }
  'tool/call': { turn: number; step: number; callId: CallId; name: string; arguments: string }
  'tool/result': { turn: number; step: number; message: ToolResultMessage; error?: {...}; meta?: JsonValue }
  'request/header': { header: EpochHeader; reason: RequestHeaderReason }
  'request/context': RequestContext
  'session/end-seed': Record<string, never>
}
```

几个值得注意的细节：

- **`assistant/chunk` 是原始流块**——token 级重放保真度。`assistant/chunk` 事件在流式传输过程中逐个记入日志，`BlockAssembler` 边流边组装。源码注释："raw stream chunk — token-level replay fidelity"。

- **`tool/call` 存的是模型原始输出的未解析 JSON 字符串**（`arguments: string`）——源码注释："the raw `arguments` JSON string exactly as the model produced it (unparsed)"。不丢模型的原话。

- **`assistant/message` 携带 usage**——源码注释："the model output and its accounting travel together (there is no separate usage record)"。没有独立的 usage 记录，避免两处记账漂移。

- **`assistant/message` 的 `interrupted: true`**——源码注释："A turn cancelled mid-stream finalizes its delivered text/reasoning prefix as this event with `interrupted: true`; undispatched tool calls are absent." 这是崩溃恢复的关键标记。

- **`request/header` 是 log-only**——为下一次请求准备的完整请求头（系统提示词等），只记最新快照，不参与历史派生。

## "Model-visible ⟺ logged"：为什么新输入必须先有新事件

dsh 最硬的一条不变量，`AGENTS.md` 原文：

> **Model-visible ⟺ logged**: anything that reaches a model request must be reconstructable from the session log; a new model-visible input requires a session event.

推论链是这样的：

1. 模型历史 `deriveMessages()` 从日志派生；
2. 所以如果某条输入没进日志，它永远不会出现在模型历史里；
3. 所以"给模型加一种新输入"（比如 cron 通知、文件变更提示、skill 目录）的唯一正确做法是**扩展 `SessionEventMap` 并定义渲染**——而不是在别处偷偷塞消息。

这个设计有一个美妙的副产品：**fork、resume、transcript、telemetry、持久化全部从同一条事件流派生**。

## `deriveMessages()`：模型历史是日志的投影

模型历史不是一份独立保存的聊天记录，而是**每次请求时从日志重新投影**出来的。`packages/core/session` 的 `deriveMessages()` 把日志里的 `user/message`、`assistant/message`、`tool/result` 等按顺序投影成模型请求的消息数组。

这样做的好处：
- **无冗余**：日志是唯一存储，模型历史是计算产物；
- **可重放**：任何时刻的历史都可以从日志精确重建；
- **可恢复**：崩溃后从持久化日志重放，就能恢复到崩溃前的完整上下文。

代价：每次请求都要重新投影（性能换正确性）。dsh 用 `request/header` 等 log-only 快照事件做部分缓存——请求头只记最新值，避免每次重算。

## Turn / Step / TurnEndReason：执行封闭

dsh 的时间结构分三层：**session → turn → step**。`docs/architecture.md` 的精确定义：

> A **step** is one model request plus the tools it calls. A **turn** is zero or more steps: it opens before its first input is claimed and closes once nothing is owed.

`TurnEndReason` 通过 merge-extensible map 定义（`packages/core/session/src/types.ts`）：

```ts
export interface TurnEndReasonMap {
  'completed': { kind: 'completed' }
  'max-tokens': { kind: 'max-tokens' }
  'blocked': { kind: 'blocked' }
  'aborted': { kind: 'aborted'; reason: AgentCancelCause }
  'error': { kind: 'error'; error: LlmFailure }
}

export type TurnEndReason = TurnEndReasonMap[keyof TurnEndReasonMap]
```

关键语义：

- **每个 session event 都被 turn 封闭**——`user/message` 一定出现在某个 turn 里。
- **turn 可以花零个 step**：pre-step 拒绝或输入为空时，turn 仍然打开又关闭，日志记录这次"尝试"。
- **`interrupted` 是唯一一个主循环永远不会自己发出的终止原因**——它只在持久化崩溃恢复路径出现（第 13 章详述）。

## `SESSION_FORMAT_VERSION`：日志格式的版本管理

`packages/core/session/src/types.ts` 里的这个常量值得单独讲：

```ts
/**
 * The on-disk session format version, stamped into every newly-written
 * {@link SessionHeader} and enforced by every persistence backend on load.
 * While the harness is unreleased it is pinned at `0`: no compatibility is
 * implied, incompatible logs are rejected, and no migration is provided.
 */
export const SESSION_FORMAT_VERSION = 0
```

注释说得很清楚：**版本 0 = 不承诺兼容**。这个常量是一个单调递增的整数，没有 major/minor 拆分。是否需要 bump 由**写入端**决定，不由更新的读取端能接受什么决定。只有结构性变更才达到 bump 的门槛：header 形状、`SessionEvent` envelope、核心事件语义、或 surface 机制（`SurfaceEventType` 集合和 `SurfaceOp` 变体）。添加普通事件类型不需要 bump——per-event 的 `SessionEvent.ignorable` guard 覆盖了词汇增长。

## 事件三域分类

`docs/architecture.md` 把事件分为三个域：

- **Session events**（`turn/*`、`step/*`、`user/message`、`assistant/*`、`tool/*`）——持久的可重建事实，通过 `session/event` 广播；
- **Agent events**（`agent/*`）——携带 live `Agent`：inbox、step、status、request、validation、continuation——观察或拦截进行中的工作；
- **Capability events**——把 policy 和 adapter 挂到 seam 上（`fs/*`、`tools/*`、`telemetry/*`），不 import 主循环。

## 架构速览

```
Session (append-only 日志)
├── turn/start { turn }
│   ├── step/start { turn, step }
│   │   ├── user/message { content, source }
│   │   ├── assistant/chunk { turn, step, chunk }  ← token 级
│   │   ├── assistant/chunk { ... }
│   │   ├── assistant/message { turn, step, message, usage }  ← 组装后
│   │   ├── tool/call { turn, step, callId, name, arguments }
│   │   └── tool/result { turn, step, message, error?, meta? }
│   └── step/end { turn, step }
├── request/header { header, reason }  ← log-only 快照
├── request/context { provider, model, contextWindow }  ← log-only
└── turn/end { turn, reason }

deriveMessages() → 从日志投影出 Message[]
```

## 关键设计决策

1. **日志是唯一真相源，模型历史是投影**：`deriveMessages()` 每次请求从日志重新投影。代价是性能，收益是无冗余、可重放、可恢复。fork、resume、transcript、telemetry 全部从同一流派生。

2. **`tool/call` 存未解析的 JSON 字符串**：`arguments: string` 保留模型原话，不预解析。这让重放能精确还原模型的原始输出，即使 JSON 格式有微妙差异。

3. **`SESSION_FORMAT_VERSION = 0`**：预览阶段不承诺兼容。只有结构性变更才 bump 版本号。这让团队可以自由重构日志格式，代价是用户升级时可能需要删除旧日志。

4. **`interrupted` 是人造的 TurnEndReason**：主循环永远不会自己发出它，只有持久化崩溃恢复路径会合成它。这让重放能精确区分"真实结束"和"崩溃恢复"。

## 本章小结

- Session 事件日志是 dsh 的心脏：append-only 的类型化事件日志，模型历史是它的投影。
- **硬不变量**：Model-visible ⟺ logged——新输入必须先有新事件，否则模型永远看不到。
- `deriveMessages()` 每次请求从日志重新投影模型历史，无冗余、可重放、可恢复。
- `SESSION_FORMAT_VERSION = 0`：预览阶段不承诺兼容，只有结构性变更才 bump。
- 三域事件：session events（持久）/ agent events（live）/ capability events（seam 政策）。

## 阅读思考题

1. `deriveMessages()` 每次请求都从日志重新投影。如果一个 session 有 10000 个事件，这个投影的性能瓶颈在哪里？dsh 用什么机制缓解？
2. `tool/call` 存的是未解析的 JSON 字符串，而不是解析后的对象。如果模型输出了一个格式不正确的 JSON，工具执行会怎样？这种设计是"宽容"还是"严格"？
3. `request/header` 是 log-only 快照，不参与历史派生。为什么请求头需要记入日志？如果不记，崩溃恢复后会发生什么？

> **下一章预告**：第 6 章将拆开 `packages/core/agent-loop/src/agent.ts` 里的 `ReactLoopAgent`——dsh 的默认心脏。它是 `Agent` 公共契约的唯一默认实现，但记住，它只是"默认"。
