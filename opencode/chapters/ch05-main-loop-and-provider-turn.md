# 第 5 章　主循环与 Provider 轮次：心脏是如何跳动的

> 如果整本书只能读一章，读这一章。
> 因为一个 Agent 之所以是 Agent——而不是一次性的 Chat 调用——全部秘密都藏在这个循环里。

第 4 章我们看清了"谁来执行、怎么调度"。本章推门进去，看 `SessionRunner` 真正跑起来后那颗心脏怎么跳。我们精读 v2 的 `session/runner/llm.ts`（404 行）——它实现了 `SessionRunner.Service`，把"生成 → 调工具 → 把结果喂回去 → 再生成"这套自主循环，建在 Effect 之上。读完你会彻底明白：**所谓"Agent 自主完成任务"，在代码层面，就是一个被 `MAX_STEPS` 上限保护、由"是否还需要续写"驱动的多层循环，反复做着"跑一个 provider 轮次、结算工具、判断要不要再来一轮"这几件事。**

## 5.1 一份写在注释里的施工蓝图

`llm.ts` 顶部有一段罕见的、长达 50 行的施工蓝图注释。它不只是文档，更是 v2 团队对"一个可信循环该做什么"的清单式自省。开头第一句就划了红线：

> Keep this as orchestration over smaller collaborators rather than rebuilding the legacy `SessionPrompt` monolith.

——把这里保持成"对小协作者的编排"，而不是重建 v1 那个庞大的 `SessionPrompt` 单体。还记得第 2 章我们看到 v1 的 `session/prompt.ts` 有 1704 行吗？v2 把它视为"要避免重蹈的覆辙"，刻意让 runner 只做编排，把脏活分给 compaction、tool registry、system context 等独立协作者。

注释用 `[x]`/`[ ]` 勾选框把工作切成可审查的小片，这本身就泄露了大量设计意图。摘几条已完成（`[x]`）的：

- "Coordinate one local active drain per Session; explicit resumes join and prompt wakeups coalesce." —— 正是第 4 章 Coordinator 干的事。
- "Bound model steps." —— 给模型步数设上限（下面的 `MAX_STEPS`）。
- "Stream exactly one `llm.stream(request)` provider turn." —— 每个 provider 轮次只发一次流式请求。
- "Durably record each tool call before side effects begin." —— **工具调用在产生任何副作用之前，先持久化记录下来**。
- "Reload projected history and start the next explicit provider turn after local tool results." —— 工具结果出来后，重新加载历史再开下一轮。

还有几条故意留着 `[ ]` 没做：多节点集群所有权、限制 provider 重试与重复的相同工具调用、把状态（busy/retrying/idle）持久化……**把"还没做什么"也明明白白写出来**，是这份蓝图最可贵的地方——它告诉读者 v2 是一个诚实地"在路上"的系统，而不是假装完美的定稿。

紧接着是全章最重要的一个常量，还带着一句作者自己的疑问：

```ts
// QUESTION: Did this exist previously, or did we add this limit? Does it make sense?
const MAX_STEPS = 25
```

**`MAX_STEPS = 25`**：一个 Agent 在单次"活动"里最多连续跑 25 个 provider 轮次。这是防止 Agent 陷入无限工具循环、烧光预算的硬护栏。那句 `// QUESTION:` 注释也很真实——连作者都在自问这个值合不合理。对 Agent 开发者来说，这正是个值得记住的细节：**步数上限是必须的，但具体取多少是经验值，没有银弹。**

## 5.2 一次 provider 轮次：`runTurnAttempt` 做了什么

循环的最小单元是 `runTurnAttempt`——"跑一个 provider 轮次的一次尝试"。它的 happy path 顺序大致是这样（对照第 2 章"主函数读作 happy path"的纪律，它读起来确实像一条直线）：

**第 1 步——位置一致性检查。** 一开头就做防御：

```ts
const session = yield* getSession(sessionID)
if (session.location.directory !== location.directory ||
    session.location.workspaceID !== location.workspaceID)
  return yield* Effect.interrupt
```

如果会话的 Location 和当前运行时的 Location 对不上（比如会话被并发地搬走了），立刻 `Effect.interrupt`——这呼应第 4 章的 Location 作用域设计：一个旧位置的 runner 不许去跑一个已经搬走的会话。

**第 2 步——选 agent、初始化 System Context 纪元。** 选出当前生效的 agent，然后初始化"上下文纪元"（Context Epoch，第 6 章主角）：

```ts
const agent = yield* agents.select(session.agent)
const initialized = yield* SessionContextEpoch.initialize(
  db, loadSystemContext(agent), session.id, session.location, agent.id,
).pipe(retryAgentMismatch(promotion))
```

`loadSystemContext` 用 `concurrency: "unbounded"` 并发加载三个来源——系统上下文、skill 指引、reference 指引——再 `SystemContext.combine` 合起来。并发加载、有序合并，这是 v2 上下文装配的标准姿势。

**第 3 步——提升持久化输入为可见消息。** 这是第 4 章"准入/执行分离"在循环里的兑现点：

```ts
if (promotion) {
  const cutoff = yield* SessionInput.latestSeq(db, session.id)
  if (promotion === "steer") yield* SessionInput.promoteSteers(db, events, session.id, cutoff)
  if (promotion === "queue") {
    yield* SessionInput.promoteNextQueued(db, events, session.id)
    yield* SessionInput.promoteSteers(db, events, session.id, cutoff)
  }
}
```

之前被"准入"进数据库的 `session_input` 行，现在在这个安全边界上被**提升（promote）**成模型能看到的用户消息。`steer`（插话）和 `queue`（排队）两种投递方式在这里有不同的提升策略——`steer` 默认合并进当前活动，`queue` 则一次开一个未来活动。这正是 `AGENTS.md` 说的"Prompts steer by default... Explicit `queue` inputs open FIFO future activities one at a time"。

**第 4 步——组装请求。** 拿到模型、加载历史、物化工具，拼出一个 `LLM.request`：

```ts
const model = yield* models.resolve(session)
const entries = yield* SessionHistory.entriesForRunner(db, session.id, system.baselineSeq)
const context = entries.map((entry) => entry.message)
const toolMaterialization = yield* tools.materialize(agent.info?.permissions)
const promptCacheKey = /^ses_[0-9a-f]{64}$/.test(session.id) ? session.id.slice(4) : session.id
const request = LLM.request({
  model,
  providerOptions: { openai: { promptCacheKey } },
  system: [agent.info?.system, system.baseline].filter(/* 非空 */).map(SystemPart.make),
  messages: toLLMMessages(context, model),
  tools: toolMaterialization.definitions,
})
```

注意 `promptCacheKey`：它用一个正则 `/^ses_[0-9a-f]{64}$/` 校验会话 ID 格式，符合就剥掉 `ses_` 前缀当缓存键。这是为了让 OpenAI 的 prompt caching 在同一会话内命中——又一个"魔鬼在细节"的成本优化。还有 `tools.materialize(agent.info?.permissions)`：**工具是按当前 agent 的权限被物化的**，权限不够的工具根本不会进入这次请求（第 8、10 章详谈）。

**第 5 步——按需压缩。** 组装好请求后，先问一句要不要压缩：

```ts
if (yield* compaction.compactIfNeeded({ sessionID: session.id, entries, model, request }))
  return yield* Effect.die(rebuildPreparedTurn())
```

如果压缩发生了，就抛一个 `rebuildPreparedTurn` 转换信号，让外层从持久化状态**重建这一轮**——因为压缩改变了历史，之前组装的 request 已经过期了。

## 5.3 流式消费与"边收边结算"工具

第 6 步是循环的高潮——发起那唯一一次 `llm.stream(request)`，并**边流式接收、边结算工具**：

```ts
const providerStream = llm.stream(request).pipe(
  Stream.runForEach((event) =>
    Effect.gen(function* () {
      // ……处理 provider error / overflow
      yield* publish(event)
      if (event.type !== "tool-call" || event.providerExecuted) return
      needsContinuation = true
      const assistantMessageID = yield* publisher.assistantMessageID(event.id)
      yield* Effect.uninterruptibleMask((restore) =>
        restore(toolMaterialization.settle({ sessionID, agent, assistantMessageID, call: event }))
          .pipe(Effect.flatMap((settlement) =>
            publish(LLMEvent.toolResult({ id: event.id, name: event.name,
              result: settlement.result, output: settlement.output }), settlement.outputPaths ?? []),
          )),
      ).pipe(FiberSet.run(toolFibers))
    }),
  ),
  Effect.ensuring(withPublication(publisher.flush())),
)
```

这段代码浓缩了好几个关键设计：

**① 每个事件都被 `publish` 持久化。** 文本增量、推理、工具调用事件，一边从模型流出来一边就被发布成 `SessionEvent` 并落库。蓝图里那条 `[x] Persist assistant text and usage events incrementally as they arrive` 就在这里兑现。这意味着即便进程下一秒崩溃，已经流出来的内容也不会丢——可恢复性是设计进流式消费里的，不是事后补的。

**② 工具调用"一出现就开跑"。** 一旦遇到 `tool-call` 事件（且不是 provider 自己执行的），立刻把 `needsContinuation` 置为 `true`（意味着"这一轮之后还得再来一轮，把工具结果给模型看"），然后通过 `toolMaterialization.settle(...)` 启动工具执行。`FiberSet.run(toolFibers)` 把每个工具结算丢进一个 fiber 集合里——**多个工具调用是并发结算的**，对应蓝图的 "Start each recorded local call eagerly and await all settlements before continuation"：急切启动、最后统一等待。

**③ 工具结算包在 `uninterruptibleMask` 里。** 工具一旦开始产生副作用（写文件、跑命令），就不能被从中间粗暴打断——所以 `restore(...settle...)` 外面套了不可中断掩码，保证"执行 + 发布结果"是一个完整区域。这正是第 4 章那条原则的延续：耗时副作用要么完整发生、要么干净取消，不能留半截。

**④ 上下文溢出被特殊对待。** 如果在 assistant 还没开始输出时就遇到 `isContextOverflowFailure`，它不会立刻报错，而是记下 `overflowFailure` 留待后面触发压缩恢复。这是为长任务准备的安全网。

流跑完后，`runTurnAttempt` 进入一大段 `uninterruptibleMask` 包裹的收尾，处理各种失败与中断的组合：provider 失败了要 `failUnsettledTools("Provider did not return a tool result")`、被中断了要 `clear(toolFibers)` 并标记工具中断、用户拒绝了一个 question（`isQuestionRejected`）要直接 `Effect.interrupt` 停掉整个循环。这段代码把"一次轮次可能以多少种方式结束"穷举得密不透风——它最终的返回值是一个布尔：`!publisher.hasProviderError() && needsContinuation`，即"没出 provider 错、且还有工具调用待续写"。**这个布尔，就是驱动外层循环转不转的信号。**

## 5.4 三层循环：步、轮次、活动

把镜头拉到最外层，看 `run` 方法——它是一个三层嵌套结构，每一层回答一种不同的"还没完"：

```ts
const run = Effect.fn("SessionRunner.run")(function* (input) {
  const hasSteer = yield* SessionInput.hasPending(db, input.sessionID, "steer")
  const hasQueue = hasSteer ? false : yield* SessionInput.hasPending(db, input.sessionID, "queue")
  if (input.force !== true && !hasSteer && !hasQueue) return   // 没活也没强制，直接收工
  yield* failInterruptedTools(input.sessionID)                 // 先清理上次中断遗留的半截工具
  let promotion = hasSteer ? "steer" : hasQueue ? "queue" : undefined
  let openActivity = input.force === true || hasSteer || hasQueue
  while (openActivity) {                                        // 第三层：活动循环
    let needsContinuation = true
    for (let step = 0; step < MAX_STEPS; step++) {              // 第二层：步循环（受 25 步上限保护）
      needsContinuation = yield* runTurn(input.sessionID, promotion)  // 第一层：一个 provider 轮次
      promotion = "steer"
      if (!needsContinuation) needsContinuation = yield* SessionInput.hasPending(db, input.sessionID, "steer")
      if (!needsContinuation) break
    }
    if (needsContinuation)
      return yield* new StepLimitExceededError({ sessionID: input.sessionID, limit: MAX_STEPS })
    openActivity = yield* SessionInput.hasPending(db, input.sessionID, "queue")
    promotion = openActivity ? "queue" : undefined
  }
})
```

三层各司其职：

| 层 | 循环结构 | 回答的问题 | 终止条件 |
|---|---|---|---|
| **轮次（turn）** | `runTurn(...)` 一次调用 | 模型这一轮要不要调工具 | provider 流结束 |
| **步（step）** | `for step < MAX_STEPS` | 工具结果回灌后，模型还想继续吗 | `needsContinuation === false` 或撞上 25 步上限 |
| **活动（activity）** | `while (openActivity)` | 还有排队（queue）的输入要处理吗 | 没有 pending 的 queue 输入 |

几个值得品味的细节：

- **每跑完一轮，`promotion` 就被设成 `"steer"`**。这意味着第一轮之后，循环会持续检查"用户有没有在执行过程中插话（steer）"，有就把它提升进来。这就是 steering——用户不必等 Agent 干完，可以随时插话纠偏，下一轮模型立刻看到。第 4 章的"准入/执行分离"在这里结出了交互体验的果实。
- **即便模型说"我不调工具了"（`needsContinuation` 为假），循环还要再查一次有没有 steer**。`if (!needsContinuation) needsContinuation = ...hasPending("steer")`——只要用户插了话，就不让循环停。
- **撞上 `MAX_STEPS` 不是悄悄停，而是显式报错** `StepLimitExceededError`，带上 `sessionID` 和 `limit`。Agent 跑飞了，用户会得到一个明确的"超过 25 步上限"提示，而不是莫名其妙地卡住。又一次"错误即文档"。
- **活动循环让 queue 输入一个接一个跑**。`steer` 是合并进当前活动，`queue` 是排队等当前活动结束后再开新的——两种连续性被清晰地分到了不同的循环层。

## 5.5 用"转换信号"代替深层重试：`TurnTransition`

最后看一个 Effect 风格的精巧设计。`runTurnAttempt` 在需要"重来"时，不是写一堆嵌套重试，而是抛出一个**类型化的转换信号**：

```ts
type TurnTransition =
  | { readonly _tag: "RebuildPreparedTurn"; readonly promotion?: SessionInput.Delivery }
  | { readonly _tag: "ContinueAfterOverflowCompaction" }

class TurnTransitionError extends Error {
  constructor(readonly transition: TurnTransition) { super() }
}
```

- `RebuildPreparedTurn`：当请求准备过程中观察到并发的 Session 变化（比如压缩发生了、agent 被切换了），需要从持久化状态**重建这一轮**。
- `ContinueAfterOverflowCompaction`：溢出压缩完成后，重新走一遍（但这次不再处理溢出，防止无限套娃）。

外层的 `runTurn` 和 `runAfterOverflowCompaction` 用 `Effect.catchDefect` 捕获这些信号，`yield* Effect.yieldNow` 让一下出栈、再递归重来：

```ts
const runTurn = Effect.fnUntraced(function* (sessionID, promotion) {
  return yield* runTurnAttempt(sessionID, promotion, compaction.compactAfterOverflow).pipe(
    Effect.catchDefect(Effect.fnUntraced(function* (defect) {
      if (!(defect instanceof TurnTransitionError)) return yield* Effect.die(defect)
      yield* Effect.yieldNow
      if (defect.transition._tag === "ContinueAfterOverflowCompaction")
        return yield* runAfterOverflowCompaction(sessionID, undefined)
      return yield* runTurn(sessionID, defect.transition.promotion)
    })),
  )
})
```

注意 `runAfterOverflowCompaction` 里有一道防递归的闸门：如果压缩后的尝试又想触发一次溢出压缩，直接 `Effect.die("Post-compaction provider attempt cannot recover another overflow")`——**只允许压缩恢复一次，绝不无限重试**。这是 fail-closed 思想在控制流里的体现：与其冒险无限循环，不如明确地失败。

> **给 Agent 开发者的启示**：把"需要重来"建模成一组有限的、带 `_tag` 的转换信号，比在循环里塞满 `if (shouldRetry) continue` 要清晰得多。每种"重来"的原因都有名字、有携带的数据、有专门的处理路径。这套"用类型化信号驱动状态转换"的范式，是 Effect 项目里管理复杂控制流的标志性手法，也是让一个 851 行级别的循环还能被人读懂的关键。

## 本章小结

- v2 的 `session/runner/llm.ts` 把 Agent 循环建成"对小协作者的编排"，刻意避免重建 v1 那个 1704 行的 `SessionPrompt` 单体；顶部 50 行 `[x]/[ ]` 蓝图诚实地列出已做与未做。
- `MAX_STEPS = 25` 是防止 Agent 陷入无限工具循环的硬护栏，连作者都在注释里自问这个值是否合理——步数上限必须有，但取值是经验值。
- 一次 provider 轮次（`runTurnAttempt`）的 happy path：位置一致性检查 → 选 agent、初始化上下文纪元 → 在安全边界提升持久化输入 → 组装 request（含 promptCacheKey、按权限物化工具）→ 按需压缩 → 发起唯一一次 `llm.stream`。
- 流式消费"边收边结算"：每个事件即时 publish 落库（可恢复性内建）、工具调用一出现就并发结算并包在 `uninterruptibleMask` 里、上下文溢出留待压缩恢复。
- `run` 是三层循环：轮次（要不要调工具）、步（受 25 步上限保护，工具回灌后还续不续）、活动（还有没有 queue 输入）；每轮后把 promotion 设为 steer 以支持执行中插话。
- 撞上步数上限会显式抛 `StepLimitExceededError`（带 sessionID/limit），不是悄悄卡住——错误即文档。
- 用类型化的 `TurnTransition` 信号（`RebuildPreparedTurn` / `ContinueAfterOverflowCompaction`）配合 `catchDefect` 递归驱动"重来"，并设防递归闸门只允许压缩恢复一次——fail-closed 体现在控制流里。

## 动手实验

1. **实验一：读懂施工蓝图** — 打开 `packages/core/src/session/runner/llm.ts` 顶部那段长注释，把所有 `[x]`（已做）和 `[ ]`（未做）分两列抄下来。挑一条未做项（如"Bound provider retries and repeated identical tool calls"），说说它如果不做、可能在真实使用中导致什么问题。
2. **实验二：定位三层循环** — 在 `run` 方法里，用笔标出 `while (openActivity)`、`for step < MAX_STEPS`、`runTurn(...)` 三层，并在每层旁边写一句话："这一层在什么情况下会再转一圈？"
3. **实验三：找到 steering 的物理位置** — 在 `run` 里找到 `promotion = "steer"` 那行，再到 `runTurnAttempt` 里找到 `promoteSteers`。用一句话向同事解释："为什么把这两处删掉，用户就无法在 Agent 干活时插话纠偏了？"
4. **实验四：验证可恢复性** — 在流式消费段里找到 `yield* publish(event)`。思考并写下：如果把"边流式边 publish"改成"等整轮跑完再一次性 publish"，在进程中途崩溃的场景下会损失什么？这和蓝图里的 "Persist ... incrementally as they arrive" 是什么关系？

> **下一章预告**：本章我们多次提到"组装 System Context""初始化 Context Epoch"却没展开。第 6 章就专门解剖 v2 那套被 `CONTEXT.md` 严肃建模的上下文系统——System Context、Context Source、Context Epoch、Safe Provider-Turn Boundary，看 OpenCode 如何精确地决定"模型每一轮到底看到什么"。
