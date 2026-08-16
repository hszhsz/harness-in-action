# 第 6 章　Agent 主循环：ReactLoopAgent 是如何跳动的

> 这一章拆开 dsh 的心脏：`packages/core/agent-loop/src/agent.ts` 里的 `ReactLoopAgent`。它是 `Agent` 公共契约的唯一默认实现——但记住，它只是"默认"，换掉它不需要改任何消费者代码。

## 6.1　`Agent` 接口：send / followup / steer / inject / cancel / whenIdle

`Agent` 是每个插件（UI、hooks、orchestrators）编程对向的表面。`ctx.agents.get(id)` 返回它；具体实现是 `agent-loop` 包的内部细节，**loop 之外的任何代码都不依赖它**。[[docs/subsystems/core.md]](https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/subsystems/core.md)

```ts
interface Agent {
  readonly id: SessionId
  readonly options: AgentOptions
  readonly session: Session
  readonly inbox: Inbox
  readonly status: AgentStatus
  readonly ctx: Context

  cancel(cause: AgentCancelCause, options?: CancelOptions): void
  whenIdle(): Promise<void>
  runMaintenance<T>(task: (signal: AbortSignal) => Promise<T>): Promise<T>
  send(message: UserMessage, target: InboxTarget, wakeup: boolean): void
  followup(message: UserMessage): void
  steer(message: UserMessage): void
  inject(message: UserMessage): void
}
```

四个投递方法的分工非常精确（源码 JSDoc）：

- **`send`** —— 统一入口：目标（next-turn 或 next-step inbox 边界）+ 是否唤醒驱动。唤醒输入在活动取消后提交会被排队到下一 turn；`disposed` 取消后留在原地。
- **`followup`** —— 排队一个普通后续 turn 并唤醒驱动，成为自己 turn 的唯一普通消息。
- **`steer`** —— 为**最近的 step** 提交转向（steering）：idle 驱动开一个 turn；运行中的驱动在下一个 step 边界消费它。被拒绝的 step 会让 steering 留在 inbox 等下一次唤醒。
- **`inject`** —— 排队模型可见上下文给下一个 pre-step，**不唤醒**驱动。运行中的驱动在最近的后续 step 边界认领；idle 驱动会一直挂起直到别的消息唤醒。取消或销毁可能丢弃 pending 上下文。

- **`cancel(cause)`** —— 清空排队的和 steering 的工作（除非 `keepInbox`），中止活动中的 turn 或 turn 间任务。第一个 cause 胜出。没有活动时取消是 no-op，不会武装后续工作。
- **`whenIdle()`** —— 等整个 agent 活动到达静止（quiescence）。会跟随替换工作，但不标识任何特定消息的结算。
- **`runMaintenance(task)`** —— 从真正的 idle 相位跑一个非 turn 维护任务。任务同步认领 idle 相位；后来的唤醒输入留在 inbox 直到任务结算，公共状态保持 `idle`。

## 6.2　Inbox 与 wakeup：输入如何到达驱动

输入通过**一个 inbox** 到达驱动。`docs/architecture.md` 说：

> Input reaches the driver through one inbox. Some messages wake it immediately; injected context waits in the inbox until another message does.

inbox 是 agent 持有的"排队的持久工作的投影"（"The agent-owned projection of durable pending work"）。wakeup 是投递时的一个布尔参数：`followup` 和 `steer` 默认唤醒，`inject` 不唤醒。唤醒输入在取消后提交会排队等下一 turn；idle 时提交的唤醒输入**总是**开它的 turn 边界（即使消息在驱动认领前被清掉——`cancel-convergence wake latch`，见 Agent Note `2026-08-07-cancel-convergence-wake-latch.md`）。

## 6.3　Turn 与 Step 边界：一次模型请求 + 它调用的工具

`ReactLoopAgent` 用 `Phase` 联合类型管理自己的生命周期状态（`agent.ts` 源码）：

```ts
type Phase =
  | { kind: 'idle'; lastTurn: number }
  | { kind: 'maintenance'; abort: AbortController; lastTurn: number; wakeRequested: boolean }
  | { kind: 'running'; abort: AbortController; turn: number; step: number; wakeRequested: boolean }
```

三种相位：idle / maintenance / running。`maintenance` 与 `running` 互斥（`runMaintenance` 在 turn-driving 或另一个维护任务占用时同步抛错）。

**Step 的准备**：

```ts
type PreparedStep =
  | { kind: 'reject' }
  | { kind: 'enter'; messages: UserMessage[]; assembly: PromptAssembly }
```

`agent/pre-step` waterfall 决定模型看到什么：监听者可以改写认领的消息，或直接拒绝。被拒绝或空的首次认领仍然关闭一个花了零 step 的持久 turn（第 5.4 节）。每个 step 读取插件注册的 prompt sections 和 tool schemas。

**每步的模型请求派生**：`requestProposal(header)` 有个精巧细节——从 adapter 派生的值（`adapterDefaults.reasoningEffort` / `adapterDefaults.maxTokens`）在插件提议下一个请求配置前被删掉，让插件可以重新提议，而不是被 adapter 的默认值锁死。

## 6.4　`agent/*` 事件族：pre-step / request / turn-stopping 的拦截语义

`agent/*` 事件携带 live `Agent`，分两类语义：

- **waterfall（必须 `next()`）**：`agent/pre-step`（改写/拒绝认领的输入）、`agent/request`（改写模型请求）、`agent/request-error`（错误恢复决策）；
- **serial（无 `next()`）**：`agent/turn-stopping`——turn 停止前的终端检查点，监听者按序运行，任何一个都可以阻止 turn 停止。

`docs/architecture.md` 的拦截地图：

| Goal | Mechanism |
|---|---|
| Intercept a request | `agent/request` |
| Intercept a tool call | `tools/*` 事件 |
| Stop a turn | `agent/turn-stopping` |
| Add model-facing context | `agent.inject()` |

## 6.5　取消与错误恢复：`agent/request-error` 与重试

模型请求失败时，驱动进入错误恢复路径：记录 `step/end`，然后走 `agent/request-error` waterfall——监听者返回 retry action 或保留原始错误。

```mermaid
alt final adapter or terminal in-band request failure
  Driver->>Session: step/end
  Driver->>Hooks: agent/request-error waterfall
  Hooks-->>Driver: return retry action or preserve the original error
```

错误恢复与取消的关系：`cancel(cause)` 的 cause 由活动操作的 signal 携带（"the stable caller intent carried by the active operation signal"）。工具的 `exec.signal` 同样来自取消链——工具体必须观察或转发 `exec.signal`，并在自己的工作到达静止后才 settle（"Async work must observe or forward `exec.signal` and settle only after its owned work reaches quiescence"）。注册表保留调用者取消（around-dispatch signal replacement），但不能硬杀同进程代码——"it cannot hard-kill same-process code"。

`AgentHandle` 的销毁路径也是分层的：`dispose()` 停止 loop、await 退出、注销 agent、从 store 移除 session、最后回卷它的 scoped world。provider 卸载会 stop and drain 它制造的每个 live handle。

## 本章小结

ReactLoopAgent 是 dsh 的默认心脏，但它的设计刻意"不特殊"：

1. **公共契约**：`Agent` 接口是消费者唯一依赖，实现是包内部细节；
2. **四投递**：send（通用）/ followup（普通 turn）/ steer（step 转向）/ inject（不唤醒的上下文）——分工精确到"唤醒与否"；
3. **相位机**：idle / maintenance / running 三态，维护任务与 turn 驱动互斥；
4. **事件拦截**：pre-step 决定模型看什么，request-error 决定失败怎么办，turn-stopping 决定何时能停；
5. **取消纪律**：cause 由 signal 携带，工具必须配合到达静止，注册表保留调用者取消。

## 动手实验

1. 读 `packages/core/agent-loop/src/agent.ts` 的 `ReactLoopAgent` 类，画出 `Phase` 状态转移图。
2. 找 `agent/pre-step` 的所有监听者（`docs/event-producer-consumer.md` 有生产者/消费者表），确认谁改写、谁拒绝。
3. 在 `packages/core/agent/src/types.ts` 里读 `AgentEventDispatch`，理解 fused dispatcher 为什么"built once in the constructor so hot-path dispatches never allocate"。
4. 模拟一次 `steer`：读 `Agent.steer` 的 JSDoc，确认它何时开 turn、何时在 step 边界消费、被拒绝后去哪。
