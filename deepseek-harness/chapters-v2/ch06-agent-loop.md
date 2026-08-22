# 第 6 章　Agent 主循环：ReactLoopAgent 是如何跳动的

这一章拆开 dsh 的心脏：`packages/core/agent-loop/src/agent.ts` 里的 `ReactLoopAgent`。它是 `Agent` 公共契约的唯一默认实现——但记住，它只是"默认"，换掉它不需要改任何消费者代码。读完后，你应该能画出它的三态相位机、理解四个投递方法的精确分工、以及 `agent/*` 事件族的拦截语义。

## `Agent` 接口：send / followup / steer / inject / cancel / whenIdle

`Agent` 是每个插件（UI、hooks、orchestrators）编程对向的表面。`ctx.agents.get(id)` 返回它；具体实现是 `agent-loop` 包的内部细节，**loop 之外的任何代码都不依赖它**。

`packages/core/agent-loop/src/agent.ts` 里 `ReactLoopAgent` 的构造函数展示了它如何初始化：

```ts
constructor(
  private loopCtx: Context,
  public readonly id: SessionId,
  public readonly options: AgentOptions,
  public readonly session: Session,
) {
  this.dispatch = agentEvents(loopCtx, this)
  this.inbox = new Inbox(session, {
    inserted: (message) => { this.dispatch.emit('agent/inbox/inserted', { message }) },
    discarded: (message) => { this.dispatch.emit('agent/inbox/discarded', { message }) },
    claimed: (message, turn) => { this.dispatch.emit('agent/inbox/claimed', { message, turn }) },
  })
  const lastTurn = session.events.findLast(event => event.type === 'turn/start')?.data.turn ?? 0
  this.phase = { kind: 'idle', lastTurn }
  this.scope = createScope(loopCtx, this)
  this.ctx = this.scope.ctx.extend({ agent: this })
  this.runtimeContext = new RuntimeContextProjection(this.ctx, session)
}
```

注意几个关键点：
- **`dispatch` 在构造时一次性构建**——源码注释："Fused dispatcher, built once in the constructor so hot-path dispatches never allocate." 热路径零分配。
- **`Inbox` 的回调**：inserted / discarded / claimed 三个事件，让外部观察者可以追踪消息的生命周期。
- **`createScope(loopCtx, this)`**：agent 拥有自己的 scoped registration boundary，卸载时回卷。

四个投递方法的分工非常精确：

```ts
send(message: UserMessage, target: InboxTarget, wakeup: boolean): void {
  const wakingAfterAbort = wakeup && this.phase.kind !== 'idle' && this.phase.abort.signal.aborted
  const resolvedTarget = wakingAfterAbort ? 'next-turn' : target
  this.inbox.splice(resolvedTarget, Infinity, 0, [message])
  if (wakeup) this.wakeDriver(wakingAfterAbort)
}

followup(input: UserMessage): void {
  this.send(input, 'next-turn', true)
}

steer(input: UserMessage): void {
  this.send(input, 'next-step', true)
}

inject(input: UserMessage): void {
  this.send(input, 'next-step', false)
}
```

- **`send`** —— 统一入口：目标（next-turn 或 next-step）+ 是否唤醒。唤醒输入在活动取消后提交会被排队到下一 turn（`wakingAfterAbort` 逻辑）。
- **`followup`** —— 排队一个普通后续 turn 并唤醒驱动。
- **`steer`** —— 为最近的 step 提交转向：idle 驱动开一个 turn；运行中的驱动在下一个 step 边界消费。
- **`inject`** —— 排队模型可见上下文给下一个 pre-step，**不唤醒**驱动。idle 驱动会一直挂起直到别的消息唤醒。

## Inbox 与 wakeup：输入如何到达驱动

输入通过**一个 inbox** 到达驱动。`docs/architecture.md` 说：

> Input reaches the driver through one inbox. Some messages wake it immediately; injected context waits in the inbox until another message does.

`send` 方法里的 `wakingAfterAbort` 逻辑值得细看：唤醒输入在取消后提交时，target 被强制改为 `'next-turn'`——它不能加入已中止的活动，必须排队等下一 turn。这是"取消纪律"的一部分。

## 相位机：idle / maintenance / running

`ReactLoopAgent` 用 `Phase` 联合类型管理自己的生命周期状态：

```ts
type Phase =
  | { kind: 'idle'; lastTurn: number }
  | { kind: 'maintenance'; abort: AbortController; lastTurn: number; wakeRequested: boolean }
  | { kind: 'running'; abort: AbortController; turn: number; step: number; wakeRequested: boolean }
```

三种相位：idle / maintenance / running。`maintenance` 与 `running` 互斥——`runMaintenance` 在 turn-driving 或另一个维护任务占用时同步抛错：

```ts
runMaintenance<T>(job: (signal: AbortSignal) => Promise<T>): Promise<T> {
  if (this.phase.kind !== 'idle') throw new Error(`agent "${this.id}" already has active work`)
  // ...
}
```

`wakeDriver` 方法展示了相位转换的核心逻辑：

```ts
private wakeDriver(wakeAfterAbort = false): void {
  if (this.phase.kind !== 'idle') {
    const reason = this.phase.abort.signal.reason as AgentCancelCause | undefined
    if (reason?.kind !== 'disposed' && (this.phase.kind === 'maintenance' || wakeAfterAbort)) {
      this.phase.wakeRequested = true
    }
    return
  }
  const driver = Promise.withResolvers<void>()
  this.activityDone = driver.promise
  this.setPhase({
    kind: 'running',
    abort: new AbortController(),
    turn: this.phase.lastTurn,
    step: 0,
    wakeRequested: false,
  })
  this.loopCtx.agents.withInitiator(this, () => this.kick()).then(driver.resolve, driver.reject)
}
```

非 idle 时不直接启动新驱动，而是设置 `wakeRequested` 标志——当前驱动结束后检查这个标志，如果有待处理消息就启动新驱动。`disposed` 取消后不设置标志（teardown 不会 latch wake）。

## Turn 与 Step 边界

`turn()` 方法是主循环的核心：

```ts
private async turn(): Promise<boolean> {
  // ...
  const turn = phase.turn + 1
  this.session.append('turn/start', { turn })
  phase.turn = turn
  let turnEnds: TurnEndReason | null = null
  try {
    while (true) {
      const step = phase.step + 1
      const decision = await this.preStep(target, { turn, step })
      if (decision.kind === 'reject') {
        turnEnds = { kind: 'blocked' }
        return false
      }
      if (phase.step === 0 && decision.messages.length === 0) {
        turnEnds = { kind: 'completed' }
        return false
      }
      this.session.append('step/start', { turn, step })
      // ...
      const stepEnd = await this.step(decision.assembly)
      if (turnEnds === null || turnEnds.kind !== 'max-tokens') turnEnds = stepEnd
      this.session.append('step/end', { turn, step })
      // ...
    }
  } finally {
    this.session.append('turn/end', { turn, reason: turnEnds! })
  }
}
```

关键语义：
- **pre-step 拒绝 → turn 以 `blocked` 结束**，不开 step。
- **空首次认领 → turn 以 `completed` 结束**，花零个 step。
- **max-tokens 是 sticky 的**：一旦某个 step 达到上限，后续正常完成的 step 不会降级 turn 结果。
- **turn/end 在 finally 块里**——无论成功还是失败，turn 总是被封闭。

## `agent/*` 事件族：pre-step / request / turn-stopping 的拦截语义

`agent/*` 事件携带 live `Agent`，分两类语义：

- **waterfall（必须 `next()`）**：`agent/pre-step`（改写/拒绝认领的输入）、`agent/request`（改写模型请求）、`agent/request-error`（错误恢复决策）；
- **serial（无 `next()`）**：`agent/turn-stopping`——turn 停止前的终端检查点，监听者按序运行，任何一个都可以阻止 turn 停止。

`preStep` 方法展示了 `agent/pre-step` waterfall 的用法：

```ts
private async preStep(target: InboxTarget, position: { turn: number; step: number }): Promise<PreparedStep> {
  const claimed = this.inbox.claim(target, position.turn)
  const assembly = await this.loopCtx.systemPrompt.assemble(assembleContextFor(this, signal))
  const sections = renderContextSections(assembly)
  const context = this.runtimeContext.project(joinContextSections(sections), sections)
  const decision = await this.dispatch.waterfall(
    'agent/pre-step', { messages: claimed, ...position, signal },
    (): Promise<PreStepDecision> => Promise.resolve<PreStepDecision>({
      kind: 'enter',
      messages: context === undefined ? claimed : [...claimed, context],
    }),
  )
  return decision.kind === 'reject' ? decision : { ...decision, assembly }
}
```

`requestProposal` 函数有一个精巧细节——从 adapter 派生的值在插件提议前被删掉：

```ts
function requestProposal(header: EpochHeader): LlmCallConfig {
  if (header.adapterDefaults === undefined) return header.config
  const proposal = { ...header.config }
  if (header.adapterDefaults.reasoningEffort === true) delete proposal.reasoningEffort
  if (header.adapterDefaults.maxTokens === true) delete proposal.maxTokens
  return proposal
}
```

这让插件可以重新提议 `reasoningEffort` 和 `maxTokens`，而不是被 adapter 的默认值锁死。

## 架构速览

```
ReactLoopAgent 相位机
┌─────────┐  wake   ┌───────────┐
│  idle   │ ──────→ │  running  │
│(lastTurn)│         │(turn,step)│
└─────────┘ ←────── └───────────┘
     ↑        done        ↑
     │                    │
     │    ┌───────────┐   │
     └──→ │maintenance│ ←─┘ (互斥)
          └───────────┘

turn() 循环:
  turn/start → [preStep → step/start → step → step/end]* → turn/end
```

## 关键设计决策

1. **热路径零分配**：`dispatch` 在构造时一次性构建 fused dispatcher，热路径 dispatches 不分配内存。这是性能敏感的 Agent 主循环的必要优化。

2. **四种投递方法的精确分工**：send（通用）/ followup（普通 turn）/ steer（step 转向）/ inject（不唤醒的上下文）——分工精确到"唤醒与否"和"目标 inbox 边界"。

3. **max-tokens 是 sticky 的**：一旦某个 step 达到 token 上限，后续正常完成的 step 不会降级 turn 结果。这保证了 turn 的终止原因反映了最严重的问题。

4. **`requestProposal` 删掉 adapter 默认值**：让插件可以重新提议 `reasoningEffort` 和 `maxTokens`，而不是被 adapter 的默认值锁死。这体现了"插件可以覆盖一切"的设计哲学。

## 本章小结

- `ReactLoopAgent` 是 dsh 的默认心脏，但设计刻意"不特殊"——`Agent` 接口是消费者唯一依赖。
- 三态相位机：idle / maintenance / running，维护任务与 turn 驱动互斥。
- 四投递：send / followup / steer / inject，分工精确到唤醒与否。
- 事件拦截：pre-step 决定模型看什么，request-error 决定失败怎么办，turn-stopping 决定何时能停。
- 取消纪律：cause 由 signal 携带，工具必须配合到达静止。

## 阅读思考题

1. `send` 方法里的 `wakingAfterAbort` 逻辑把 target 强制改为 `'next-turn'`。为什么唤醒输入不能加入已中止的活动？如果允许它加入，会破坏什么不变量？
2. `runMaintenance` 在非 idle 时同步抛错。为什么维护任务不能和 turn 驱动并行？如果允许并行，会引入什么竞态条件？
3. `agent/turn-stopping` 是 serial 事件，没有 `next()`。如果一个监听者在 `turn-stopping` 里做了耗时操作（比如等待用户确认），主循环会怎样？

> **下一章预告**：第 7 章将讲 dsh 的 LLM 层——`packages/llm/llm` 里的 `Message` / `ContentBlock` / `StreamChunk` 词汇表，以及 adapter 契约。
