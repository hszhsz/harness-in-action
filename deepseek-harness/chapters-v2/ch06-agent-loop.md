# 第 6 章 ReactLoopAgent：Agent Loop 的状态机

上一章 Session 已经创建，`user/message` 事件已追加。现在 `ReactLoopAgent` 接管了——它是驱动整个对话循环的核心状态机。从这里开始，请求进入了"模型思考 → 输出 → 工具调用 → 模型继续思考"的循环。

## 请求到达这里时发生了什么

`ReactLoopAgent` 在 `packages/core/agent-loop/src/agent.ts` 中定义。它的构造函数接收：

```typescript
constructor(
  private loopCtx: Context,
  public readonly id: SessionId,
  public readonly options: AgentOptions,
  public readonly session: Session,
) {
  this.dispatch = agentEvents(loopCtx, this)
  this.inbox = new Inbox(session, { ... })
  const lastTurn = session.events.findLast(event => event.type === 'turn/start')?.data.turn ?? 0
  this.phase = { kind: 'idle', lastTurn }
  this.scope = createScope(loopCtx, this)
  this.ctx = this.scope.ctx.extend({ agent: this })
  this.runtimeContext = new RuntimeContextProjection(this.ctx, session)
}
```

关键组件：
- **Inbox**：消息队列，管理待处理的用户输入
- **phase**：状态机，三种状态——`idle`、`maintenance`、`running`
- **scope**：Agent 级别的 Cordis 作用域，生命周期与 Agent 绑定
- **RuntimeContextProjection**：运行时上下文注入

## 关键代码路径

### 状态机：Phase

```typescript
type Phase =
  | { kind: 'idle'; lastTurn: number }
  | { kind: 'maintenance'; abort: AbortController; lastTurn: number; wakeRequested: boolean }
  | { kind: 'running'; abort: AbortController; turn: number; step: number; wakeRequested: boolean }
```

- **idle**：没有活跃工作，等待唤醒
- **running**：正在执行一个或多个 turn
- **maintenance**：正在执行维护任务（如 compaction）

当 `send(message, target, wakeup)` 被调用时，如果 `wakeup=true`，它会调用 `wakeDriver()` 启动驱动循环。

### 驱动循环：kick → turn → step

`wakeDriver()` 从 idle 转到 running，然后调用 `kick()`：

```typescript
private async kick(): Promise<void> {
  try {
    while (await this.turn()) {}
  } catch (_error) {
    // 报告的失败在驱动边界被遏制
  } finally {
    if (this.phase.kind === 'running') {
      const { turn, wakeRequested } = this.phase
      this.setPhase({ kind: 'idle', lastTurn: turn })
      if (wakeRequested && this.inbox.hasPending) this.wakeDriver()
    }
  }
}
```

`kick` 循环调用 `turn()`，直到没有更多工作。每个 `turn()` 代表一个完整的交互回合。

### turn：打开回合，循环 step

```typescript
private async turn(): Promise<boolean> {
  const turn = phase.turn + 1
  this.session.append('turn/start', { turn })
  phase.turn = turn
  let target: InboxTarget = 'next-turn'
  
  while (true) {
    signal.throwIfAborted()
    const step = phase.step + 1
    const decision = await this.preStep(target, { turn, step })
    if (decision.kind === 'reject') {
      turnEnds = { kind: 'blocked' }
      return false
    }
    // ... 执行 step ...
  }
}
```

一个 turn 内部可以有多个 step。每个 step 是一次模型调用加上它请求的所有工具执行。

### preStep：组装 Prompt，决定是否进入

```typescript
private async preStep(target: InboxTarget, position): Promise<PreparedStep> {
  const claimed = this.inbox.claim(target, position.turn)
  const assembly = await this.loopCtx.systemPrompt.assemble(assembleContextFor(this, signal))
  const sections = renderContextSections(assembly)
  const context = this.runtimeContext.project(joinContextSections(sections), sections)
  
  const decision = await this.dispatch.waterfall(
    'agent/pre-step', { messages: claimed, ...position, signal },
    (): Promise<PreStepDecision> => Promise.resolve({
      kind: 'enter',
      messages: context === undefined ? claimed : [...claimed, context],
    }),
  )
  return decision
}
```

`preStep` 做了三件事：
1. 从 Inbox 中 claim 本 step 应该处理的消息
2. 组装系统提示词（system prompt）
3. 通过 `agent/pre-step` waterfall 让插件有机会拦截（比如拒绝进入、修改消息）

### buildRequest：组装 LLM 请求

```typescript
private async buildRequest(
  turn, step, tools, system, boundaryMessages, signal
): Promise<{ request: GenerateOptions; preparedCall?: PreparedLlmCall }> {
```

这个方法：
1. 从 Session 的 `requestHeader()` 获取之前保存的请求配置
2. 通过 `agent/request` waterfall 让插件修改配置
3. 调用 `llm.prepareCall(proposedConfig, signal)` 解析适配器
4. 比较新旧 header，如果变化了就追加 `request/header` 事件
5. 追加 `request/context` 事件（路由元数据）
6. 组装最终的 `GenerateOptions`（包含 messages、system、tools）

### step 执行：流式调用 LLM + 处理工具

step 的核心循环（在 `turn()` 方法内部）：

```typescript
// 追加 step/start 事件
this.session.append('step/start', { turn, step })

// 构建请求
const { request, preparedCall } = await this.buildRequest(...)

// 流式调用 LLM
const stream = this.loopCtx.llm.generate(request)
const assembler = new BlockAssembler()

for await (const chunk of stream) {
  this.session.append('assistant/chunk', { turn, step, chunk })
  assembler.accumulate(chunk)
}

// 组装 assistant/message 事件
const message = createAssistantMessage(assembler)
this.session.append('assistant/message', { turn, step, message, usage })

// 如果有工具调用，执行它们
if (toolCalls.length > 0) {
  const results = await executeToolCalls(this.ctx, session, turn, step, toolCalls, signal)
  // 追加 tool/result 事件
  // 下一个 step 的 target 变为 'next-step'
  target = 'next-step'
} else {
  // 没有工具调用，turn 结束
  this.session.append('turn/end', { turn, reason: { kind: 'completed' } })
  return true // 继续下一个 turn
}
```

## 调用链：从这里到下一步

当 LLM 返回工具调用（比如 `write_file`）时，`executeToolCalls` 函数被调用。

**下一步：请求将流向工具系统——`ToolDefinition` 如何定义工具，六段守卫管线如何校验和执行工具调用。**

## 关键设计决策

1. **Inbox 的 target 机制**：`next-turn` 消息在当前 turn 结束后才被 claim，`next-step` 消息在当前 step 结束后立即可用。这让 `steer()`（用户中途修改）和 `followup()`（turn 结束后的新输入）有不同语义。

2. **waterfall 拦截点**：`agent/pre-step` 和 `agent/request` 都用 waterfall 模式，让插件可以拦截、修改、甚至拒绝。这是 extensibility 的关键。

3. **流式处理**：`assistant/chunk` 事件逐 token 记录，保证了完美的重放保真度。`assistant/message` 是从 chunks 组装的摘要。

4. **AbortSignal 传播**：每个 turn 和 step 都有 signal，取消请求可以精确传播到正在进行的 LLM 调用和工具执行。

## 本章小结

- `ReactLoopAgent` 是三层循环：kick（驱动）→ turn（回合）→ step（步）
- Phase 状态机有三种状态：idle、running、maintenance
- `preStep` 组装系统提示词并决定是否进入 step
- `buildRequest` 通过 waterfall 让插件修改 LLM 请求配置
- step 核心循环：流式调用 LLM → 记录 chunks → 组装 message → 执行工具调用

## 阅读思考题

1. 为什么一个 turn 可以包含多个 step？什么场景下会发生多 step turn？
2. `wakeRequested` 标志的作用是什么？如果去掉它会丢失什么场景？
3. `agent/request` waterfall 中，如果一个插件返回了没有 provider/model 的配置会怎样？

> **下一章预告**：模型决定调用 `write_file` 工具。现在请求进入工具系统——工具如何定义、注册、校验和执行？六段守卫管线是什么？
