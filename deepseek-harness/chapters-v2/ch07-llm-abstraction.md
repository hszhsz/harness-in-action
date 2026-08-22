# 第 7 章　LLM 流式抽象：Message / ContentBlock / StreamChunk

dsh 的 LLM 层在 `packages/llm/llm`：它定义整个系统公用的对话词汇（`Message`/`ContentBlock`）、完整的模型请求、原始流块协议 `StreamChunk`、每个 adapter 必须实现的契约。这一章把这层讲透。核心源码在 `packages/llm/llm/src/types.ts`。

## 内容块：text / reasoning / image / tool-call / tool-result

dsh 的对话模型：一个 `Message` 是一组**类型化 content blocks**。块联合从 merge-extensible map 派生：

```ts
export interface ContentBlockMap {
  'text': TextBlock
  'reasoning': ReasoningBlock
  'image': ImageBlock
  'tool-call': ToolCallBlock
  'tool-result': ToolResultBlock
}

export type ContentBlockType = keyof ContentBlockMap
export type ContentBlock = ContentBlockMap[ContentBlockType]
```

每种块的结构（源码原文）：

- `TextBlock`：`{ type: 'text', text: string }` —— 普通文本。
- `ReasoningBlock`：`{ type: 'reasoning', text: string }` —— 模型思维过程，与可见文本分离。
- `ImageBlock`：`{ type: 'image', attachment: ImageAttachmentRef }` —— 持久的图片附件（dsh 有独立的 `attachment` seam 管理图片身份与元数据）。
- `ToolCallBlock`：`{ type: 'tool-call', id: CallId, name: string, arguments: string }` —— 工具调用请求，`arguments` 是原始 JSON 字符串。
- `ToolResultBlock`：`{ type: 'tool-result', toolCallId: CallId, content: ContentBlock[], isError?: boolean }` —— 工具调用结果，嵌套 `content` 允许富文本结果。

**新增 modality 的门槛**：源码注释说"A new modality belongs in the merge-extensible map only when its adapter, UI, compaction, and durable replay paths honor it"——一个新块类型要进 map，必须同时有 adapter、UI、compaction、durable replay 四条路径支持。

## Message 与 provenance

`Message` 本身是不可变值：稳定 id + 角色 + 块 + 来源。`packages/llm/llm/src/message.ts` 里 assistant 消息携带 provenance：

```ts
export interface AssistantProvenance {
  provider: string
  model: string
  replayState?: unknown   // adapter-private replay data
}
```

`replayState` 是 adapter 私有的重放数据——`LlmRuntime` 只在"目标 adapter 同时拥有历史 provider 和目标 provider"时把它暴露给目标 adapter。这是"重放保真"的机制核心：换模型后，历史消息依然可以被原 adapter 重放。

## 模型请求组装与 `request/header` 日志

每个 step 的模型请求由三部分组装：**prompt sections（`ctx.systemPrompt` 按序拼接）+ tool schemas（`ctx.tools` 的 allowlist 投影）+ 从日志派生的历史（`deriveMessages()`）**。

`ReactLoopAgent.buildRequest()` 方法（`packages/core/agent-loop/src/agent.ts`）展示了完整的请求构建流程：

```ts
private async buildRequest(turn, step, tools, system, boundaryMessages, signal) {
  const persistedHeader = session.requestHeader()
  const seedConfig = deepFreeze(structuredClone(
    this.requestHeaderLogged
      ? requestProposal(persistedHeader!)
      : { ...route, ...reasoningEffort === undefined ? {} : { reasoningEffort }, ...maxTokens === undefined ? {} : { maxTokens } },
  ))
  const proposedConfig = await this.dispatch.waterfall(
    'agent/request', { turn, step, signal },
    () => Promise.resolve(seedConfig),
  )
  // ...
  const header = canonicalHeader({ config, ...adapterDefaults, ...system, ...tools })
  if (!this.requestHeaderLogged) {
    this.session.append('request/header', { header, reason: baseline === undefined ? 'initial' : 'resume' })
    this.requestHeaderLogged = true
  } else if (baseline === undefined || !headerEquals(baseline, header)) {
    this.session.append('request/header', { header, reason: 'change' })
  }
}
```

请求头在 step 内、派发前作为 `request/header` 事件**记入日志**。`headerEquals` 做相等性检查——如果两次请求头相同，跳过日志记录。这是 KV cache 友好的设计。

## Adapter 契约：`LlmAdapter` 与多 Provider 注册

Adapter 层是 dsh 的一个 **capability seam**：Service Definition 在 `llm/llm`（`ctx.llm`），Provider 是各个 adapter 包（`llm-deepseek`、`llm-pi-ai`），Consumer 是主循环。`docs/architecture.md` 的扩展点地图：

> Add a model provider → register its adapter on `ctx.llm`.

adapter 契约包括：把 `Message`/`ContentBlock` 转成 provider 的 wire 格式、实现 `StreamChunk` 流、报告 usage。

## `BlockAssembler` 与流式组装

`BlockAssembler`（`llm/llm` 包）把原始 `StreamChunk` 流组装成完整的 `AssistantMessage`。在 `ReactLoopAgent.step()` 里可以看到它的用法：

```ts
const assembler = new BlockAssembler()
const chunkSeqs: number[] = []
try {
  const stream = preparedCall?.stream(request) ?? this.loopCtx.llm.stream(request)
  for await (const chunk of stream) {
    chunkSeqs.push(this.session.append('assistant/chunk', { turn, step, chunk }).seq)
    assembler.push(chunk)
  }
} catch (error: unknown) {
  if (signal.aborted) {
    const content = assembler.interruptedBlocks()
    if (content.length > 0) {
      this.session.append('assistant/message', {
        turn, step,
        message: createAssistantMessage({ content, source: { provider, model } }),
        interrupted: true,
        ...assembler.usage === undefined ? {} : { usage: assembler.usage },
      }, { surfaceOp: 'append', sourceEventSeqs: chunkSeqs })
    }
  }
  throw error
}
```

关键设计：
- **chunks 边流边记 `assistant/chunk` 事件**——token 级重放保真度。
- **中断时**：如果流在中途被中止，`assembler.interruptedBlocks()` 返回已收集的部分内容，记为 `assistant/message` with `interrupted: true`。
- **usage 随消息走**：`assembler.usage` 直接附加到 `assistant/message` 事件，没有独立的 usage 记录。

## FinishReason 与错误恢复

```ts
export interface FinishReasonMap {
  'stop': { kind: 'stop' }
  'tool-calls': { kind: 'tool-calls' }
  'max-tokens': { kind: 'max-tokens' }
  'aborted': { kind: 'aborted'; failure: LlmFailure }
  'error': { kind: 'error'; failure: LlmFailure }
}
```

`LlmFailure` 是 provider-neutral 的失败事实：

```ts
export interface LlmFailure {
  readonly message: string
  readonly code: string
  readonly status?: number
  readonly providerRetryAfterMs?: number
  readonly requestId?: ProviderRequestId
}
```

`step()` 方法在 `assembler.finish` 为 `error` 或 `aborted` 时进入 `agent/request-error` waterfall——监听者返回 retry action 或保留原始错误。

## TokenUsage：disjoint 计数

```ts
export interface TokenUsage {
  inputTokens: number
  outputTokens: number
  cacheReadTokens?: number
  cacheWriteTokens?: number
  reasoningTokens?: number
}
```

源码注释："Counts are DISJOINT: `inputTokens` is uncached input only; cached input is reported separately as `cacheReadTokens`/`cacheWriteTokens`"。计数是不相交的——`inputTokens` 只是非缓存输入，缓存输入单独报告。Adapters whose providers fold cache hits into a total prompt count 需要自己减出来。

## 架构速览

```
LLM 层词汇
┌─────────────────────────────────────────┐
│  ContentBlockMap (merge-extensible)     │
│  ├── text      TextBlock                │
│  ├── reasoning ReasoningBlock           │
│  ├── image     ImageBlock               │
│  ├── tool-call ToolCallBlock            │
│  └── tool-result ToolResultBlock        │
├─────────────────────────────────────────┤
│  Message = id + role + ContentBlock[]   │
│  AssistantProvenance = provider + model │
│                         + replayState?  │
├─────────────────────────────────────────┤
│  StreamChunk → BlockAssembler           │
│  → assistant/chunk (token级)            │
│  → assistant/message (组装后)           │
├─────────────────────────────────────────┤
│  LlmAdapter seam: ctx.llm              │
│  Provider: llm-deepseek / llm-pi-ai    │
│  Consumer: 主循环                        │
└─────────────────────────────────────────┘
```

## 关键设计决策

1. **`replayState` 支撑跨 adapter 重放**：adapter 私有的重放数据让换模型后历史消息依然可以被原 adapter 重放。这是"重放保真"的机制核心。

2. **`ToolCallBlock.arguments` 是原始 JSON 字符串**：不预解析，保留模型原话。这与 `SessionEventMap` 的 `tool/call` 设计一致——"exactly as the model produced it (unparsed)"。

3. **usage 随消息走，无独立记录**：`assistant/message` 事件携带 step 的 `usage`，避免两处记账漂移。这是"模型可见即日志"在 token 计量层面的体现。

4. **新 modality 必须四路齐备**：adapter、UI、compaction、durable replay 四条路径都支持才能加入 `ContentBlockMap`。这保证了每种内容类型在所有下游路径都能正确处理。

## 本章小结

- LLM 层是 dsh 的"对话词汇表 + 适配器契约"。
- `ContentBlockMap` 是 merge-extensible 的块类型联合，新 modality 必须四路齐备。
- `Message` 不可变，assistant 消息携带 provenance（`replayState` 支撑跨 adapter 重放）。
- 请求 = sections + schemas + 派生历史，请求头作为 log-only 事件入日志。
- `BlockAssembler` 边流边记 `assistant/chunk`，中断时记 `interrupted: true` 的 `assistant/message`。

## 阅读思考题

1. `replayState` 是 adapter 私有的重放数据。如果用户从 DeepSeek 切换到另一个 provider，历史消息的 `replayState` 还有意义吗？dsh 如何处理这种情况？
2. `BlockAssembler` 在流中断时用 `interruptedBlocks()` 收集已有的部分内容。如果中断发生在 reasoning block 的中间，组装出的消息会是什么样子？
3. `TokenUsage` 的计数是 disjoint 的。如果一个 adapter 的 provider 把 cache hits 折叠进 `prompt_tokens`，adapter 需要怎么做才能报告正确的 disjoint 计数？

> **下一章预告**：第 8 章将拆开 `packages/core/tools`——工具系统是 Agent 的"手"。`ToolDefinition` 的六段守卫管线是 dsh 工程哲学的缩影。
