# 第 4 章　Session 运行时：对话状态的中枢

上一章，一条 prompt 通过 `client.session.prompt({...})` 离开了 CLI，进入会话系统。本章我们解剖 OpenCode v2 的 Session 运行时——这是整个 Agent 的状态中枢。这里有一个贯穿 v2 设计的核心理念，第 2 章已经预告过，现在我们正面拆开它：**"接收一条 prompt"和"调用一次模型"是两件被刻意分开的事。** 理解了这个分离，你就理解了 v2 为什么要引入 `SessionRunner`、`SessionExecution`、`SessionRunCoordinator` 这一组看似复杂的协作者。

## 4.1 为什么要把"准入"和"执行"分开

`AGENTS.md` 的"V2 Session Core"一节开宗明义：

> Keep durable prompt admission separate from model execution. `SessionV2.prompt(...)` admits one durable `session_input` row before scheduling advisory `SessionExecution.wake(sessionID)` unless `resume: false` requests admit-only behavior.

翻译成人话：当你发一条 prompt，系统**先把它作为一行持久化的 `session_input` 记录写进数据库**（这叫"准入 / admission"），然后才**调度一次"唤醒"**（`SessionExecution.wake`）去真正驱动模型。这两步之间是解耦的。

为什么要这么麻烦？想想一个 Agent 在真实世界里会遇到的麻烦事：

- 用户在模型还在输出时又插了一句话（steering）；
- 进程在模型跑到一半时崩了；
- 同一个会话被两个地方同时戳了一下。

如果"收消息"和"跑模型"是同一个同步调用，上面每一种情况都会变成竞态灾难。v2 的解法是：**把所有进来的 prompt 先变成数据库里一行不可丢失的记录，再由一个串行化的执行器在"安全的边界"上慢慢消化它们。** 这样无论外界怎么并发地戳，落到模型执行那一层永远是有序、可恢复的。

`CONTEXT.md` 里给这个"安全的边界"起了个正式名字——**Safe Provider-Turn Boundary（安全的 provider 轮次边界）**："The point immediately before a provider call, after durable input promotion and any required tool settlement, where context changes may be admitted chronologically." 也就是：在每次调用模型之前、在已持久化的输入被"提升"为可见用户消息之后、在必要的工具结算之后——这个点，才是允许上下文变化进场的安全时刻。

## 4.2 三个协作者：Runner、Execution、Coordinator

v2 把 Session 的"执行"拆成了三个职责清晰的角色，它们都用 Effect 的 `Context.Service` 定义为可注入服务：

- **`SessionRunner`**（`session/runner/index.ts`）：**真正干活的人**。它的接口注释写得很准："Runs one local continuation from already-recorded Session history."——从已经记录下来的 Session 历史里，跑一次本地续写。它的 `run` 方法签名是：

  ```ts
  readonly run: (input: {
    readonly sessionID: SessionSchema.ID
    readonly force?: boolean
  }) => Effect.Effect<void, RunError>
  ```

  注意它**只认 `sessionID`**，不接收 prompt 内容——因为 prompt 早已被持久化，Runner 要做的是去库里把"该干的活"捞出来干掉。注释强调："Drains eligible durable work. Explicit runs perform one provider attempt even when no work is eligible." 显式的 `run`（`force: true`）即便没有待办，也会强制做一次 provider 尝试。

- **`SessionExecution`**：**对外的遥控器**。它暴露 `wake`（唤醒，告知"可能有新活了"）、`resume`（显式恢复执行）、`interrupt`（中断）三个动作。第 3 章 CLI 投递 prompt 后调度的那个"advisory wake"，戳的就是它。

- **`SessionRunCoordinator`**（`session/run-coordinator.ts`）：**调度交通警**。它保证"每个 Session 同一时刻最多只有一条执行链在跑，而不同 Session 之间可以并发"。这是整个 v2 并发模型的核心，下一节细看。

三者的装配关系，在 `session/execution/local.ts` 里一目了然——`SessionExecution` 的本地实现，就是把 Coordinator 的三个方法转接出去：

```ts
return SessionExecution.Service.of({
  interrupt: coordinator.interrupt,
  resume: coordinator.run,
  wake: coordinator.wake,
})
```

而 Coordinator 的 `drain`（每次真正排空一个 Session 的活）回调，则委托给 `SessionRunner.run`：

```ts
drain: Effect.fnUntraced(function* (sessionID, mode) {
  const session = yield* store.get(sessionID)
  if (!session) return yield* Effect.die(`Session not found: ${sessionID}`)
  return yield* SessionRunner.Service.use((runner) =>
    runner.run({ sessionID, force: mode === "run" }),
  ).pipe(Effect.provide(locations.get(session.location)))
})
```

这段代码还透露了一个重要约束：Runner 是 **Location-scoped（位置作用域）** 的。`Effect.provide(locations.get(session.location))` 意味着每个 Session 的执行，是在它所属的 Location 的服务环境里跑的。`AGENTS.md` 对此有明确规定："Keep `SessionRunner`, model resolution, tool registry, permissions, and filesystem Location-scoped." 模型解析、工具注册表、权限、文件系统——全都绑定在 Session 的 Location 上。这是 v2 为将来"把会话搬到别的位置/机器上跑"预留的接缝（文件里那句 "Future remote placement belongs here" 就是明证）。

## 4.3 交通警的心法：每 Session 一条执行链

`SessionRunCoordinator` 是本章技术含量最高的一段代码，也是 v2 并发正确性的基石。它的核心数据结构注释把意图说得很清楚：

> Runs at most one drain chain per key while allowing different keys to drain concurrently.

这里的 `key` 就是 `sessionID`。它维护一张 `active: Map<Key, Entry>`，每个 Entry 是"一个 Session 的进程内执行车道"。注释画出了状态机：

```
idle --run/wake--> draining --run/wake--> draining + one coalesced rerun --> idle
```

理解它的关键是分清两种"需求"（Demand）：

```ts
type Demand = { readonly _tag: "run" } | { readonly _tag: "wake"; readonly seq?: number }
```

- **`run`（显式运行）**：用户明确要求"现在就跑一次"。
- **`wake`（唤醒）**：advisory（建议性）的，"有持久化的活可能就绪了"。

两者优先级不同，体现在 `coalesce` 合并函数里——**run 永远压倒 wake**：

```ts
const coalesce = (left, right) => {
  if (left?._tag === "run" || right._tag === "run") return { _tag: "run" }
  return { _tag: "wake", seq: maxSeq(left?.seq, right.seq) }
}
```

它的并发纪律可以归纳成几条，每条都直接对应一类竞态：

**① 同一 Session 串行，不同 Session 并发。** 当一个 Session 正在 draining 时，新来的 `run`/`wake` 不会另起一条链，而是被记成**至多一个**待合并的 `pending` 后继。多次 `wake` 会被 `coalesce` 折叠成一个——注释说"Repeated wakes collapse together"。这避免了"用户狂点、模型被重复触发"的灾难。

**② run 加入正在跑的链并升级语义。** 看 `run` 函数里的逻辑：如果当前正跑着一个 `wake`，新来的 `run` 会把 pending 升级成 run，并挂一个 `explicitWaiter`——这样发起 `run` 的调用方能拿到"显式运行"的结果语义，而不是被动等一个 advisory wake 的结果：

```ts
if (entry.current._tag === "wake") {
  entry.pending = coalesce(entry.pending, { _tag: "run" })
  entry.explicitWaiter ??= Deferred.makeUnsafe<A, E>()
  return restore(awaitRun(entry.explicitWaiter))
}
```

**③ 中断有序号、可比较。** `interrupt(key, seq)` 用一个单调递增的 `seq` 来界定"中断边界"。`interruptSeq` 这张表记住每个 key 最后一次中断的序号，`isAfterInterrupt` 则保证**中断边界之前的 advisory wake 被抑制、之后的才放行**：

```ts
function isAfterInterrupt(key, seq) {
  const latest = interruptSeq.get(key)
  return latest === undefined || (seq !== undefined && seq > latest)
}
```

这解决了一个非常微妙的竞态：用户按下 Ctrl-C 中断后，那些"在中断之前就排队的旧 wake"不该再把会话唤醒；但中断之后的新输入应该正常生效。用序号比较来划清这条时间线，比用布尔标志健壮得多。

**④ 用 `uninterruptibleMask` 守护状态转换。** `run` 整个包在 `Effect.uninterruptibleMask` 里，确保"检查 active、设置 entry、启动 owner"这一串内存状态转换不会被中途打断——注释明说"Every in-memory transition is synchronous"。所有 Map 的读写都是同步完成的，异步只发生在真正 `drain`（跑模型）那一步。这是一种经典的并发设计：**把决策做成同步原子的，把耗时操作留到决策之后**。

> **给 Agent 开发者的启示**：很多 Agent 的并发 bug——重复响应、中断不干净、崩溃后丢活——根源都在于把"收输入"和"跑模型"耦在一起，再用临时的布尔锁去补救。OpenCode v2 的做法是把这件事**提升为一个一等公民的领域问题**：先持久化、再串行排空、用序号管中断、用 coalesce 管合并。这套机制本身与 LLM 无关，但它决定了你的 Agent 在真实并发下"可信"还是"能跑"。

## 4.4 Session 是什么：一条会话的身份与账本

跑了半天"执行"，那被执行的"Session"本身长什么样？看 `session/schema.ts` 里的 `SessionSchema.Info` 类，它就是一条会话的完整身份与账本：

```ts
export class Info extends Schema.Class<Info>("SessionV2.Info")({
  id: ID,                                  // ses_ 前缀的品牌化 ID
  parentID: ID.pipe(optionalOmitUndefined), // 父会话（支持 fork / 子会话）
  projectID: ProjectV2.ID,
  agent: AgentV2.ID.pipe(Schema.optional),  // 当前生效的 agent
  model: ModelV2.Ref.pipe(Schema.optional), // 选定的模型
  cost: Schema.Finite,
  tokens: Schema.Struct({
    input, output, reasoning,
    cache: Schema.Struct({ read, write }),
  }),
  time: Schema.Struct({ created, updated, archived }),
  title: Schema.String,
  location: Location.Ref,                    // 4.2 说的 Location 作用域
  subpath: RelativePath.pipe(Schema.optional),
}) {}
```

几个细节值得品味：

- **ID 是品牌化（branded）且自带语义的**。`ID = Schema.String.check(Schema.isStartsWith("ses"))` 强制所有 Session ID 以 `ses_` 开头，并用 `Schema.brand("SessionID")` 让它在类型系统里与普通字符串不可混用。生成时用 `"ses_" + Identifier.descending()`——`descending` 暗示 ID 是**时间倒序可排**的，这样按 ID 排序就等于按新旧排序。这是"不变量固化进类型"原则的一个微观样本：一个会话 ID 不会被误当成别的 ID 用，编译器替你把关。
- **token 账本是结构化的**，不只记总数，而是细分 `input / output / reasoning / cache.read / cache.write`。能把 cache 的读写分开记，说明 OpenCode 认真对待 prompt caching 的成本核算（第 7 章我们会看到 provider 侧如何产生这些数字）。
- **`parentID` 让会话能成树**。fork 一个会话、派生子会话，都靠这个字段。第 3 章 `run` 命令里那个 `--fork` flag，最终就落在这里。

## 4.5 一次执行可能怎么失败：把错误写进类型

回到 `runner/index.ts`，它声明的 `RunError` 联合类型，本身就是一份"这次执行可能以哪些方式失败"的清单：

```ts
export type RunError =
  | LLMError
  | SessionRunnerModel.Error
  | MessageDecodeError
  | ContextSnapshotDecodeError
  | StepLimitExceededError
  | SystemContext.InitializationBlocked
  | SessionContextEpoch.AgentReplacementBlocked
  | ToolOutputStore.Error
```

对照第 2 章讲的工程纪律"Avoid `try/catch`"，这里就是它的正面体现：失败不是靠满天飞的异常，而是被**穷举进类型**。任何调用 Runner 的代码，编译器都会提醒它"你得处理这八种失败之一"。其中：

- `StepLimitExceededError`：带着 `sessionID` 和 `limit` 两个字段——它不只说"超限了"，还告诉你"哪个会话、限额是多少"。这正是"错误即文档"：错误对象本身携带可操作的诊断信息。
- `SystemContext.InitializationBlocked` / `AgentReplacementBlocked`：这两个错误对应第 6 章会讲的"上下文初始化被阻塞""跨 agent 替换被阻塞"。`CONTEXT.md` 里有一条规则解释了它们存在的意义："unavailable initial context blocks the turn instead of persisting an incomplete baseline"——宁可阻塞这一轮，也不肯持久化一个不完整的基线。这是"默认即关闭 / fail-closed"原则：不确定时，选更安全的"拒绝"而非"将就着跑"。

`SessionRunnerModel`（`runner/model.ts`）则负责把"选哪个模型"翻译成"用哪套协议"。它的 `fromCatalogModel` 是一组干净的分支：`@ai-sdk/openai` → OpenAI Responses 协议、`@ai-sdk/anthropic` → Anthropic Messages 协议、`@ai-sdk/openai-compatible` → OpenAI 兼容 Chat 协议；都不匹配就返回一个带 `providerID/modelID/api` 三个字段的 `UnsupportedApiError`——又是一次"错误即文档"。这里只是 LLM 抽象层的入口，完整的协议适配留到第 7 章。

## 本章小结

- v2 的核心理念是把"准入持久化 prompt"与"执行模型"分离：`SessionV2.prompt` 先写一行 `session_input` 记录，再调度 advisory `wake`；落到模型执行的永远是有序、可恢复的。
- 这个分离让 steering（执行中插话）、崩溃恢复、并发戳同一会话从"竞态灾难"变成"在 Safe Provider-Turn Boundary 上有序消化"。
- 执行被拆成三个可注入服务：`SessionRunner`（只认 sessionID、从持久化历史里排空活）、`SessionExecution`（wake/resume/interrupt 遥控器）、`SessionRunCoordinator`（保证每 Session 串行、跨 Session 并发的交通警）。
- Runner、模型解析、工具注册、权限、文件系统都是 Location-scoped，为将来的远程放置预留接缝。
- Coordinator 的并发纪律：同 Session 串行/异 Session 并发、`run` 压倒 `wake`、多次 wake 被 coalesce 折叠、用单调 `seq` 划定中断边界、用 `uninterruptibleMask` 保证状态转换原子——这套机制与 LLM 无关却决定了并发可信度。
- `SessionSchema.Info` 用品牌化且时间倒序可排的 `ses_` ID、结构化 token 账本（含 cache 读写分离）、`parentID` 会话树，把一条会话的身份与账本固化进类型。
- `RunError` 把八种失败穷举进类型（呼应"避免 try/catch"）；`InitializationBlocked` 等错误体现 fail-closed——宁可阻塞也不持久化不完整基线。

## 动手实验

1. **实验一：追踪一条 prompt 的去向** — 从 `packages/core/src/session` 出发，依次打开 `execution.ts`、`execution/local.ts`、`run-coordinator.ts`、`runner/index.ts`，画一张箭头图：`wake` 是怎么一步步变成 `SessionRunner.run` 的。
2. **实验二：手推一次并发** — 假设对同一个 `sessionID` 在极短时间内连续发生 `wake(seq=1)`、`wake(seq=2)`、`run`。按 `coalesce` 和 `run` 函数的逻辑，推演 `active` 里那个 Entry 的 `current` 和 `pending` 最终是什么。验证"run 压倒 wake、wake 折叠"。
3. **实验三：理解中断序号** — 在 `run-coordinator.ts` 里读 `interrupt`、`isAfterInterrupt`、`acceptsWake` 三个函数。用一句话解释：为什么"中断之前排队的 wake"必须被抑制，而单纯用一个布尔 `interrupted` 标志做不到这件事。
4. **实验四：清点失败模式** — 打开 `runner/index.ts`，把 `RunError` 的每个成员对应到"它在什么情况下发生"。再找到 `StepLimitExceededError` 的定义，列出它携带的字段，说说这些字段对排查问题有什么帮助。

> **下一章预告**：本章我们看清了"谁来执行、怎么调度执行"。第 5 章进入执行内部——Runner 真正跑起来后，那个"生成→调工具→把结果喂回去"的循环长什么样。我们会对照 v1 的 `session/processor.ts` 与 v2 的 runner，看 Agent 的心脏究竟是怎么一轮轮跳动的。
