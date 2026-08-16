# 第 7 章　LLM 流式抽象：Message / ContentBlock / StreamChunk

> dsh 的 LLM 层在 `packages/llm`：它定义整个系统公用的对话词汇（`Message`/`ContentBlock`）、完整的模型请求、原始流块协议 `StreamChunk`、每个 adapter 必须实现的契约，以及共享的组装器。这一章把这层讲透。

## 7.1　内容块（content blocks）：text / reasoning / image / tool-call / tool-result

dsh 的对话模型：一个 `Message` 是一组**类型化 content blocks**。块联合从 merge-extensible map 派生：[[docs/subsystems/llm-streaming.md]](https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/subsystems/llm-streaming.md)

```ts
interface ContentBlockMap {
  'text': TextBlock
  'reasoning': ReasoningBlock
  'image': ImageBlock
  'tool-call': ToolCallBlock
  'tool-result': ToolResultBlock
}
```

- `TextBlock`：`text`；
- `ReasoningBlock`：thinking（与可见文本分离——模型思维过程独立存放）；
- `ImageBlock`：一个持久的图片附件（dsh 有独立的 `attachment` seam 管理图片身份与元数据）；
- `ToolCallBlock`：`id: CallId` + `name` + 原始 JSON `arguments`；
- `ToolResultBlock`：`toolCallId` + 嵌套 `content: ContentBlock[]` + `isError?`。

**新增 modality 的门槛**：一个新块类型要进 map，必须同时有 adapter、UI、compaction、durable replay 四条路径支持（"A new modality belongs in the merge-extensible map only when its adapter, UI, compaction, and durable replay paths honor it"）。

`Message` 本身是不可变值：稳定 id + 角色 + 块 + 来源。**assistant 消息携带 provenance**：

```ts
interface AssistantProvenance {
  provider: string
  model: string
  replayState?: unknown   // adapter-private replay data
}
```

`replayState` 是 adapter 私有的重放数据——`LlmRuntime` 只在"目标 adapter 同时拥有历史 provider 和目标 provider"时把它暴露给目标 adapter。这是"重放保真"的机制核心：换模型后，历史消息依然可以被原 adapter 重放。

## 7.2　模型请求组装与 `request/header` 日志

每个 step 的模型请求由三部分组装：**prompt sections（`ctx.systemPrompt` 按序拼接）+ tool schemas（`ctx.tools` 的 allowlist 投影）+ 从日志派生的历史（`deriveMessages()`）**。

请求头（`EpochHeader`）在 step 内、派发前作为 `request/header` 事件**记入日志**（第 5.1 节讲过它是 log-only 快照）。`request/context` 只在路由或容量变化时记。`requestProposal` 会把 adapter 派生的值（`reasoningEffort`/`maxTokens`）从提议里删掉，让插件可以重新提议（第 6.3 节）。

请求头的相等性有专门检查（`headerEquals`）：如果两次请求头相同，系统可以跳过不必要的日志/缓存失效——这就是 KV cache 友好的设计（`docs/architecture.md` 提到 prompt-section 变更会 invalidate cache，所以 dsh 把 "harness:source" 这类稳定行放在请求头前部，避免每 turn 变化）。

## 7.3　Adapter 契约：`LlmAdapter` 与多 Provider 注册

Adapter 层是 dsh 的一个 **capability seam**：Service Definition 在 `llm/llm`（`ctx.llm`），Provider 是各个 adapter 包（`llm-deepseek`、`llm-pi-ai`），Consumer 是主循环。`docs/architecture.md` 的扩展点地图：

> Add a model provider → register its adapter on `ctx.llm`.

`llm-deepseek` 是 DeepSeek 官方 adapter，`llm-pi-ai` 是 pi-ai 的。**adapter 注册是普通插件行为**——`dsh-base` bundle 里 `llm` 行就挂着这些。adapter 契约包括：把 `Message`/`ContentBlock` 转成 provider 的 wire 格式、实现 `StreamChunk` 流、报告 usage。

## 7.4　`BlockAssembler` 与重试：llm-retry 与 token-meter

- **`BlockAssembler`**（`llm/llm` 包）：把原始 `StreamChunk` 流组装成完整的 `AssistantMessage`。chunks 边流边记 `assistant/chunk` 事件（token 级重放保真），组装完记 `assistant/message`（含 usage）。
- **`llm-retry`**：模型请求的重试封装。
- **`token-meter`**：token 计量——`docs/subsystems/token-meter.md` 描述它是"immutable scalar and positional replay measurements with consumed-log revisions"（不可变标量与位置重放测量，带已消费日志修订）。

**usage 随消息走**：`assistant/message` 事件携带 step 的 `usage`（adapter 报告时）——"the model output and its accounting travel together (there is no separate usage record)"。没有独立的 usage 记录，避免两处记账漂移。

## 本章小结

LLM 层是 dsh 的"对话词汇表 + 适配器契约"：

1. **ContentBlockMap**：merge-extensible 块类型，新 modality 必须四路齐备；
2. **Message 不可变**：稳定 id + provenance，`replayState` 支撑跨 adapter 重放；
3. **请求 = sections + schemas + 派生历史**，请求头作为 log-only 事件入日志；
4. **Adapter 是 seam**：注册到 `ctx.llm`，换模型就是换插件；
5. **usage 随消息走**，无独立记录，杜绝漂移。

## 动手实验

1. 读 `packages/llm/llm/src/types.ts` 的 `StreamChunk` 协议，画出 `assistant/chunk` 事件与 `BlockAssembler` 的关系。
2. 读 `packages/llm/llm-deepseek`，看 adapter 如何把 `Message[]` 转成 DeepSeek API 的请求格式。
3. 搜 `assembleContextFor`，理解 `dsh-agent` 如何给 assembly 加上 live `agent` 字段。
4. 读 `docs/subsystems/token-meter.md`，理解"consumed-log revisions"的定位。
