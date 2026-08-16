# 第 5 章　Session 事件日志：append-only 的唯一真相源

> 绝大多数 Agent 把"会话状态"存成快照，dsh 把它存成**事件日志**。这一章讲清楚：为什么日志是唯一真相源，模型历史为什么是日志的投影，以及"Model-visible ⟺ logged"这条硬不变量。

## 5.1　`SessionEventMap`：事件词汇表

dsh 的核心会话模型在 `packages/core/session/src/types.ts`：一个 `Session` 是**类型化 `SessionEvent` 的 append-only 日志**——一个 Agent 全部交互历史的唯一真相源。LLM 消息历史从日志**派生**，绝不单独存储；重放就是重新从同一批事件派生。[[docs/subsystems/session.md]](https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/subsystems/session.md)

事件类型通过 **merge-extensible map**（声明合并）组织：`SessionEventMap` 是核心词汇表，插件可以往里面加自己的事件（比如 compaction seam 加 `compaction/start`/`compaction/summary`/`compaction/end`，hook 桥加 `hook/invoked`/`hook/result`）。核心事件（源码原文）：

```ts
interface SessionEventMap {
  'turn/start': { turn: number }
  'turn/end': { turn: number; reason: TurnEndReason }
  'step/start': { turn: number; step: number }
  'step/end': { turn: number; step: number }
  'user/message': UserMessage
  'assistant/chunk': { turn: number; step: number; chunk: StreamChunk }
  'assistant/message': { turn: number; step: number; message: AssistantMessage; usage?: TokenUsage }
  'tool/call': { turn: number; step: number; callId: CallId; name: string; arguments: string }
  'tool/result': { turn: number; step: number; message: ToolResultMessage; error?: {...}; meta?: JsonValue }
  'todo/write': { todos: TodoItem[] }
  'request/header': { header: EpochHeader; reason: RequestHeaderReason }
  'request/context': RequestContext
  'session/end-seed': Record<string, never>
}
```

几个值得注意的细节：

- **`assistant/chunk` 是原始流块**——token 级重放保真度（"raw chunks, so persistence can store the canonical log verbatim"）。
- **`tool/call` 存的是模型原始输出的未解析 JSON 字符串**（`arguments: string`，"exactly as the model produced it (unparsed)"）——不丢模型的原话。
- **`tool/result` 可以带 `meta`**：工具私有的呈现载荷（如 `dsh-tool-fs` 的结果时上下文 diff），但必须是 JSON 可序列化——`Session.append` 用 `isJsonValue` 运行时校验，非序列化的 `meta` 在源头就被拒绝。
- **`request/header` 是 log-only**：为下一次请求准备的完整请求头（系统提示词等），只记最新快照，不参与历史派生。
- **`todo/write` 也是 log-only**：UI 状态，不是模型历史（"Log-only UI state; never derived history"）。

## 5.2　"Model-visible ⟺ logged"：为什么新输入必须先有新事件

dsh 最硬的一条不变量，`AGENTS.md` 原文：

> **Model-visible ⟺ logged**: anything that reaches a model request must be reconstructable from the session log; a new model-visible input requires a session event.

翻译：**任何到达模型请求的东西必须能从会话日志重建；一个新的模型可见输入，必须先新增一个 session event。**

推论链是这样的：

1. 模型历史 `deriveMessages()` 从日志派生；
2. 所以如果某条输入没进日志，它永远不会出现在模型历史里；
3. 所以"给模型加一种新输入"（比如 cron 通知、文件变更提示、skill 目录）的唯一正确做法是**扩展 `SessionEventMap` 并定义渲染**——而不是在别处偷偷塞消息。

这就是 `AGENTS.md` 里 "A new model-visible input requires a new session event: extend `SessionEventMap` and render from the log" 的完整逻辑。它在运行时被断言：`packages/core/session` 的 invariant 检查事件是否可重建。

这个设计有一个美妙的副产品：**fork、resume、transcript、telemetry、持久化全部从同一条事件流派生**。`docs/architecture.md` 说：

> The session log is the source of the context the model sees. `deriveMessages()` projects model history from it, and raw `assistant/chunk` events preserve replay and UI fidelity. Fork, resume, transcripts, telemetry, and persistence all derive from this stream.

## 5.3　`deriveMessages()`：模型历史是日志的投影

模型历史不是一份独立保存的聊天记录，而是**每次请求时从日志重新投影**出来的。`deriveMessages()` 把日志里的 `user/message`、`assistant/message`（组装后的助手消息）、`tool/result` 等按顺序投影成模型请求的消息数组。

这样做的好处：
- **无冗余**：日志是唯一存储，模型历史是计算产物；
- **可重放**：任何时刻的历史都可以从日志精确重建；
- **可恢复**：崩溃后从持久化日志重放，就能恢复到崩溃前的完整上下文。

代价：每次请求都要重新投影（性能换正确性）。dsh 用 `request/header` 等 log-only 快照事件做部分缓存——请求头只记最新值，避免每次重算。

## 5.4　Turn / Step / TurnEndReason：执行封闭

dsh 的时间结构分三层：**session（整个会话）→ turn（一轮用户驱动的活动）→ step（一次模型请求 + 它调用的工具）**。`docs/architecture.md` 的精确定义：

> A **step** is one model request plus the tools it calls. A **turn** is zero or more steps: it opens before its first input is claimed and closes once nothing is owed.

Turn 的生命周期（事件层面）：

```
turn/start
  claim next-step input plus one queued message
  assemble prompt sections + tool schemas
  -> agent/pre-step                   reject | enter(messages)
     step/start
     append entered messages as user/message
     derive model history from the log
     agent/request -> llm/stream -> assistant/chunk* -> assistant/message
     tool/call* -> tools/pre-execute -> tools/execute -> tools/post-execute -> tool/result*
     step/end
     tools owe another request, or next-step input arrived -> claim -> next step
  -> agent/turn-stopping
turn/end
```

关键语义：

- **每个 session event 都被 turn 封闭**（"every session event is turn-enclosed"）——`user/message` 一定出现在某个 turn 里，这为持久化、恢复、fork 提供了干净的边界。
- **turn 可以花零个 step**：pre-step 拒绝或输入为空时，turn 仍然打开又关闭，日志记录这次"尝试"——"a rejected or empty first claim still closes a durable turn that spent no step"。
- **`TurnEndReason` 是封闭联合**：`completed` / `max-tokens` / 其他终止原因。持久化恢复时还有一个人造的 `interrupted`（第 13 章详述）——它是唯一一个主循环永远不会自己发出的终止原因。

**事件三域分类**（`docs/architecture.md`）：

- **Session events**（`turn/*`、`step/*`、`user/message`、`assistant/*`、`tool/*`）——持久的可重建事实，通过 `session/event` 广播；
- **Agent events**（`agent/*`）——携带 live `Agent`：inbox、step、status、request、validation、continuation——观察或拦截进行中的工作；
- **Capability events**——把 policy 和 adapter 挂到 seam 上（`fs/*`、`tools/*`、`telemetry/*`），不 import 主循环。

## 本章小结

Session 事件日志是 dsh 的心脏：

1. **唯一真相源**：append-only 的类型化事件日志，模型历史是它的投影；
2. **硬不变量**：Model-visible ⟺ logged——新输入必须先有新事件，否则模型永远看不到；
3. **三域事件**：session events（持久）/ agent events（live）/ capability events（seam 政策）；
4. **turn 封闭**：每个事件都有干净的 turn 边界，为持久化、恢复、fork 提供结构。

这个设计把"会话"从"一份聊天记录"变成了"一条可重放的事件流"——所有下游（UI 回放、telemetry、fork、持久化）都从同一流派生，杜绝了多副本漂移。

## 动手实验

1. 读 `packages/core/session/src/types.ts` 的 `SessionEventMap` 全文，找出所有带 `surfaceOp` 与不带 `surfaceOp` 的事件（后者是 log-only）。
2. 在 `packages/core/agent-loop/src/agent.ts` 里搜 `turn/start`，确认主循环在哪个时机打开 turn。
3. 搜 `deriveMessages`，看它的投影逻辑如何处理 `assistant/chunk`（raw chunks）与 `assistant/message`（组装消息）的关系。
4. 读 `docs/persistence-catalog.md`，数一数全部持久化事件（含插件合并的）有多少个。
