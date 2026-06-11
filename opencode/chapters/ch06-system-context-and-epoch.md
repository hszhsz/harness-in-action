# 第 6 章　System Context 与 Context Epoch：模型每一轮到底看到什么

> 第 5 章里我们三次擦肩而过的两个词——"组装 System Context""初始化 Context Epoch"——本章正面拆开。
> 这是 OpenCode v2 最被认真对待的一块领域：它专门写了一份 `CONTEXT.md` 领域词典，逐条定义了十几个术语，只为精确回答一个问题：**模型每一轮到底看到什么。**

第 1 章我们说过，让模型"变可靠"的工程机制里，最隐蔽也最关键的一条是"它能看到什么"。一个玩具脚本会把整个仓库一股脑塞进上下文；一个生产级 Agent 则有一套精密的、可刷新的、可比较的上下文装配系统。本章读 `packages/core/src/system-context/` 这个目录（三个文件）和 `session/context-epoch.ts`（344 行），看 v2 如何把"上下文"建模成一组**可独立刷新的类型化来源**，又如何用一个带乐观并发控制的"纪元"机制，保证模型看到的基线在一段时间内稳定不变。

## 6.1 先读词典：为什么"System Context"不叫"System Prompt"

打开仓库根目录的 `CONTEXT.md`，"Language"一节第一条就给 **System Context** 下了定义，还特意标注了"不要用的词"：

> **System Context**: The structured collection of contextual facts presented to the model as initial instructions and chronological updates.
> _Avoid_: System prompt

为什么要刻意回避"system prompt"这个人人都懂的词？因为"prompt"暗示一段**静态的、一次性拼好的字符串**，而 v2 想表达的是一个**结构化的、随时间演进的事实集合**：它由若干"来源"组成，会在对话过程中产生"时间线上的更新"。这个命名上的较真，正是第 1 章说的"OpenCode 用一整份领域文档来定义模型看到的上下文"的微观证据——**当你决定不再把它当成一坨字符串，整套设计就被迫升级了。**

词典里还有一串环环相扣的概念，我们会在本章逐个对应到代码：

- **Context Source（上下文来源）**：System Context 里一个被独立观察的类型化值，有稳定的 key、JSON codec、不会失败的 loader、纯函数式的 baseline/update 渲染器，以及给动态来源用的可选 removal 渲染器。
- **Context Epoch（上下文纪元）**：一个生效 agent 初次渲染出的 System Context 保持不变的那段时间，在压缩或其他"替换基线"的转换时结束。
- **Baseline System Context（基线）**：一个纪元开始时完整渲染出的 System Context。
- **Context Snapshot（快照）**：模型看不到的、可覆写的 JSON 状态，用来把每个 Context Source 与"上一次喂给模型的值"做比较。
- **Mid-Conversation System Message（对话中系统消息）**：当某个来源变了，告诉模型"它现在的新状态"的一条持久化时间线指令。

记住这五个词，下面的代码就读得通了。

## 6.2 Context Source：一个能"观察、比较、渲染"的来源

`system-context/index.ts` 顶部的模块注释把整个抽象的意图说得很干净：

> `Source<A>` describes how to observe, compare, and render one value.

一个 `Source<A>` 就是"如何观察、比较、渲染某一个值"。看它的接口：

```ts
export interface Source<A> {
  readonly key: Key
  readonly codec: Schema.Codec<A, Schema.Json, never, never>
  readonly load: Effect.Effect<A | Unavailable>
  readonly baseline: (current: A) => string
  readonly update: (previous: A, current: A) => string
  readonly removed?: (previous: A) => string
}
```

逐个字段品味，每一个都对应词典里的一句话：

- **`key`** 是品牌化、带命名空间的稳定标识。它的正则 `/^[a-z0-9][a-z0-9._-]*\/[a-z0-9][a-z0-9._/-]*$/` 强制 key 必须形如 `core/environment`、`core/date`——**命名空间 + 名字**。重复的 key 会让组合直接失败（下面的 `combine` 会抛 `DuplicateKeyError`）。
- **`codec`** 既负责把值编码进快照存库，又负责比较两次观察是否相等（用 `Schema.toEquivalence` 派生出等价函数）。
- **`load`** 是"观察"这个来源当前的值。它的返回类型是 `A | Unavailable`——**注意它不会"失败"**（错误类型是 `never`），但可以返回一个特殊的 `unavailable` 符号，表示"这次暂时观察不到"。这是一个微妙却重要的区分，6.4 节细说。
- **`baseline` / `update` / `removed`** 是三个**纯函数渲染器**：`baseline` 渲染"初次出现时给模型看的文本"，`update` 渲染"这个值从 previous 变成 current 时给模型看的文本"，`removed`（可选）渲染"这个来源消失时给模型看的文本"。

`make<A>(source)` 这个工厂函数做的事很精巧：它**把类型参数 `A` 藏起来**，产出一个不透明的 `SystemContext`，让"日期来源"（A 是字符串）和未来可能的"任意结构化来源"（A 是别的类型）能用同一种方式组合。注释原话："`make` closes over `A`, producing an opaque `SystemContext` that composes uniformly with contexts built from other value types." 这是一种经典的存在类型（existential type）封装手法——**对外只暴露"我是一个上下文来源"，把"我承载什么类型的值"作为实现细节封进闭包。**

## 6.3 三个内置来源：environment、date、instructions

抽象讲完，看真实的来源长什么样。`system-context/builtins.ts` 注册了两个最基础的：

```ts
SystemContext.make({
  key: SystemContext.Key.make("core/environment"),
  codec: Schema.toCodecJson(Schema.String),
  load: Effect.succeed(environment),
  baseline: (environment) =>
    ["Here is some useful information about the environment you are running in:", environment].join("\n"),
  update: (_previous, environment) => ["The environment you are running in is now:", environment].join("\n"),
}),
```

`core/environment` 把工作目录、workspace 根、是否 git 仓库、平台拼成一段 `<env>...</env>`。注意它的 `baseline` 和 `update` 措辞不同——初次是"这是关于你运行环境的有用信息"，变化时是"你的运行环境现在是"。这正好呼应 `CONTEXT.md` 里那段"Example dialogue"：日期变了，对话中系统消息**只说新日期，不说旧日期**——"Emit the newly effective date so the agent can act on the current System Context."

`core/date` 来源更能体现这一点：

```ts
SystemContext.make({
  key: SystemContext.Key.make("core/date"),
  codec: Schema.toCodecJson(Schema.String),
  load: DateTime.nowAsDate.pipe(Effect.map((date) => date.toDateString())),
  baseline: (date) => `Today's date: ${date}`,
  update: (_previous, date) => `Today's date is now: ${date}`,
})
```

`load` 每次都读当前日期。如果一个会话从昨天活到今天，下一个安全边界上，date 来源的值变了，就会生成一条"Today's date is now: ..."的对话中系统消息——模型于是知道日期翻篇了。`CONTEXT.md` 还留了个伏笔："a configured user timezone may replace that default later"——目前用宿主机本地日历日期，将来可能换成用户配置的时区。

第三个来源在单独的 `instruction-context.ts` 里，它最能展示"动态来源"的全貌——加载项目里的 `AGENTS.md` 指令文件：

```ts
const source = (value) =>
  SystemContext.make({
    key,  // "core/instructions"
    codec: Schema.toCodecJson(Files),
    load: Effect.succeed(value),
    baseline: render,
    update: (_previous, current) =>
      `These instructions replace all previously loaded ambient instructions.\n\n${render(current)}`,
    removed: () => "Previously loaded instructions no longer apply.",
  })
```

它是唯一带 `removed` 渲染器的内置来源——因为指令文件可能被删掉，删掉后要明确告诉模型"之前加载的指令不再适用"。它的 `observe` 还藏着一道**路径逃逸防御**：先算出当前目录相对于项目根的相对路径，只有当目录确实在项目内（`insideProject`）时才向上扫 `AGENTS.md`，否则不扫。它还尊重 `OPENCODE_DISABLE_PROJECT_CONFIG` 这个开关——置位时只读全局指令、不扫项目目录。这对应 `CONTEXT.md` 那条："Ambient project instruction discovery honors `OPENCODE_DISABLE_PROJECT_CONFIG`; global instructions remain eligible."

## 6.4 Unavailable：观察失败，但不等于"被删除"

`instruction-context.ts` 里有一行特别值得停下来看：

```ts
if (files.some((file, index) => file === undefined && discovered.has(paths[index])))
  return SystemContext.unavailable
```

意思是：如果一个**已经被发现存在**的指令文件，这次却读不出来（可能是临时 IO 故障），就返回 `unavailable`，而**不是**返回"空指令列表"。这两者天差地别：

- 返回**空列表**，意味着"指令确实没了"——会触发 `removed` 渲染器，告诉模型"指令不再适用"。
- 返回 **`unavailable`**，意味着"我这次没看清，但请保留上一次的状态"——什么都不更新。

`CONTEXT.md` 把这条规则叫 **Unavailable Context**，并明确它用的是 **stale-while-revalidate（陈旧但重新验证）** 语义："the runtime retains its prior effective state and emits no update"。`system-context/index.ts` 的 `initialize` 把这条规则提到了更高的高度——**初始化时只要有任何一个来源 unavailable，整次初始化就被阻塞**：

```ts
export function initialize(value): Effect.Effect<Generation, InitializationBlocked> {
  return observe(value).pipe(
    Effect.flatMap((entries) => {
      const unavailable = entries.flatMap((entry) => (entry._tag === "Unavailable" ? [entry.key] : []))
      if (unavailable.length > 0) return new InitializationBlocked({ keys: unavailable })
      return Effect.succeed(initializeObservation(entries))
    }),
  )
}
```

还记得第 4 章那个 `InitializationBlocked` 错误吗？它就诞生在这里。`CONTEXT.md` 给的理由掷地有声："unavailable initial context blocks the turn instead of persisting an incomplete baseline"——**宁可阻塞这一轮，也绝不持久化一个不完整的基线。** 这是第 1 章那条"fail-closed（默认即关闭）"原则在上下文层最干净的体现：不确定时，选择"拒绝"而非"将就着跑"。一个不完整的基线会污染整个纪元（基线是要被 provider 缓存、跨进程重启复用的），代价远比"这一轮先等等"高。

## 6.5 reconcile：每个安全边界上做的那次"对账"

来源会变，但不能"一变就推送给模型"。`CONTEXT.md` 反复强调一条纪律：

> Context changes are sampled and admitted lazily at a Safe Provider-Turn Boundary, never pushed asynchronously when their source changes.

上下文变化是在**安全的 provider 轮次边界上被惰性采样**的，绝不在来源变化时异步推送。还有一条："Context source changes never wake idle sessions"——上下文变化绝不唤醒空闲会话。这与第 4 章 Coordinator 的设计一脉相承：只在已经安排好的边界上，顺手对一次账。

这次"对账"就是 `reconcile`。它观察一遍当前所有来源，与上一次的 `Snapshot` 比较，返回**恰好一个**下一步动作：

```ts
export type ReconcileResult = { readonly _tag: "Unchanged" } | Updated | ReplacementResult
```

- **Unchanged**：所有来源都没变，什么都不做。
- **Updated**：有来源变了（或新增了来源），把这些变化的 `update`/`baseline` 文本合并成**一条** Mid-Conversation System Message，同时推进快照。
- **Replace / ReplacementReady / ReplacementBlocked**：发生了"不兼容"的变化——比如某个来源的 codec 解不出旧快照（schema 变了），或者一个**没有 removal 渲染器**的来源消失了。这种情况没法用"增量更新"表达，只能整体替换基线。

`reconcileObservation` 函数里有两处细节值得点出。其一，多个来源的变化被**合并成一条**消息：`updates.push(...)` 收集所有变化文本，最后 `render(updates)`（用 `\n\n` 连接）成一条。对应 `CONTEXT.md`："Changes from multiple Context Sources admitted at one safe boundary combine into one Mid-Conversation System Message." 其二，遍历"上一次有、这一次没了"的来源时，如果它**没留 removal 文本**，就判定为不兼容、走 Replace；留了的，就 push 它的 removal 文本。这把"能优雅增量删除"和"必须整体重建"两种情况干净地分开了。

## 6.6 Context Epoch：用乐观并发把基线"钉"在数据库里

来源和渲染都搞清楚了，最后一块拼图是：这些基线和快照**存在哪、怎么防并发**。这就是 `session/context-epoch.ts` 的职责——它把 System Context 的抽象计算，落进 `SessionContextEpochTable` 这张表，并用一套乐观并发控制守护它。

词典对 **Context Epoch** 的定义是："一个生效 agent 初次渲染出的 System Context 保持不变的那段时间，在压缩或另一次替换基线的转换时结束。" 一张 epoch 行记录着：`baseline`（基线文本）、`agent`（拥有这个基线的 agent）、`snapshot`（快照）、`baseline_seq`（基线对应的输入序号）、`revision`（修订号）、可空的 `replacement_seq`。

**乐观并发的核心是 `revision` 这个修订号。** 几乎每个写操作都遵循"读出当前 revision → 带着 `expectedRevision` 去更新 → 更新条件里要求 `revision === expectedRevision` → 没更新到就抛 `RevisionMismatch`"的模式。看 `advance`（推进快照）：

```ts
const updated = yield* db
  .update(SessionContextEpochTable)
  .set({ snapshot, revision: expectedRevision + 1 })
  .where(and(
    eq(SessionContextEpochTable.session_id, sessionID),
    eq(SessionContextEpochTable.revision, expectedRevision),
    isNull(SessionContextEpochTable.replacement_seq),
  ))
  .returning({ revision: SessionContextEpochTable.revision })
  .get()
if (!updated) return yield* Effect.die(new RevisionMismatch())
```

如果两个并发的尝试同时想推进同一个 revision，只有一个的 `where revision = expectedRevision` 会命中，另一个的 `updated` 为空、抛 `RevisionMismatch`。而最外层用一个 `retryRevisionMismatch` 把这种冲突**捕获并重试**：

```ts
const retryRevisionMismatch = (attempt) =>
  attempt().pipe(
    Effect.catchDefect((defect) =>
      defect instanceof RevisionMismatch
        ? Effect.yieldNow.pipe(Effect.andThen(retryRevisionMismatch(attempt)))
        : Effect.die(defect),
    ),
  )
```

撞了就 `yieldNow`（让出一下）再重来，直到稳定。`CONTEXT.md` 给这套机制的注脚是："Context Epoch preparation retries until stable after optimistic revision mismatches so concurrent replacement requests cannot terminate an otherwise valid safe-boundary run."——**并发的替换请求不能搞死一次本来合法的安全边界运行。** 这与第 5 章那个 `TurnTransition` 信号是同一种哲学：把"需要重来"建模成显式的、可捕获的信号，而不是用锁去硬抗。

## 6.7 fence：把纪元"焊"在 Location 和 agent 上

`context-epoch.ts` 里还有一个低调但关键的函数 `fence`（栅栏）。它在两个场景下被调用：当 reconcile 结果是 `Unchanged`、或 agent 替换被阻塞时。它做的事是**校验当前纪元仍然属于预期的 agent 和 revision**：

```ts
const fence = Effect.fnUntraced(function* (db, sessionID, agent, expectedRevision) {
  const current = yield* db
    .select({ selected: SessionTable.agent, revision: SessionContextEpochTable.revision })
    .from(SessionContextEpochTable)
    .innerJoin(SessionTable, eq(SessionTable.id, SessionContextEpochTable.session_id))
    .where(eq(SessionContextEpochTable.session_id, sessionID))
    .get()
  if (!current || (current.selected !== null && current.selected !== agent))
    return yield* Effect.die(new AgentMismatch())
  if (current.revision !== expectedRevision) return yield* Effect.die(new RevisionMismatch())
})
```

为什么"什么都没变"的时候还要 fence 一下？因为"什么都没变"这个结论本身，是基于一个特定 revision 观察出来的。如果在你观察期间，另一个 fiber 切换了 agent 或推进了 revision，你这个"Unchanged"的结论就过期了——必须 `RevisionMismatch` 重来，或 `AgentMismatch` 报错。这呼应 `CONTEXT.md` 那条："Context Epoch initialization is fenced against the authoritative Session Location, so an old-Location runner cannot recreate source context after a concurrent move." 第 4 章我们说 Runner 是 Location-scoped 的，这里就是那个 Location 栅栏在上下文层的落点。

agent 切换还触发一条专门的规则。当 `replacingAgent` 为真且替换被阻塞（旧 agent 的特权基线还观察不全）时，直接抛 `AgentReplacementBlocked`：

```ts
if (result._tag === "ReplacementBlocked" && replacingAgent) {
  yield* fence(db, sessionID, agent, stored.revision)
  return yield* new AgentReplacementBlocked({ sessionID, previous: stored.agent, current: agent })
}
```

`CONTEXT.md` 解释了它存在的安全意义："A cross-agent replacement must complete before another provider turn; unavailable admitted context blocks that replacement instead of exposing the previous agent's privileged baseline."——**跨 agent 替换必须先完成；与其暴露上一个 agent 的特权基线，不如阻塞这次替换。** 又一次 fail-closed。

## 6.8 baselineSeq：历史投影的"切线"

最后把这一切和第 5 章接上。第 5 章 `runTurnAttempt` 组装请求时，调了 `SessionHistory.entriesForRunner(db, session.id, system.baselineSeq)`。这个 `baselineSeq` 就是 epoch 行里的 `baseline_seq`——它是历史投影的一条**切线**。

看 `session/history.ts` 的 `messageRows`：它在选消息时，对 `type === "system"` 的消息加了一个条件——只要 `seq > baselineSeq` 的系统消息。换句话说，**基线建立之前的那些旧"对话中系统消息"被排除在投影之外**了。这正是词典里"Baseline System Context remains separate provider-request state"的体现：基线本身作为独立的 provider 请求状态（system 部分）单独传，而历史投影里只保留基线之后产生的增量系统消息。压缩会重建基线、推进 baselineSeq，于是旧的增量系统消息自然离开模型视野，但作为审计历史仍然durably留在库里——"prior Mid-Conversation System Messages remain durable audit history but leave projected model history"。

至此，"模型每一轮到底看到什么"这个问题有了完整答案：**一段被 epoch 钉住的不可变基线（system 部分）+ 从 baselineSeq 和压缩切点之后投影出来的对话历史（messages 部分）+ 在每个安全边界上惰性对账产生的增量系统消息。** 三者由乐观并发和 Location/agent 栅栏共同守护，保证无论怎么并发，模型看到的都是一份自洽的上下文。

## 本章小结

- v2 为"模型看到什么"专门写了 `CONTEXT.md` 领域词典，刻意用 **System Context**（结构化、随时间演进的事实集合）取代"system prompt"（静态字符串）的心智。
- `Source<A>` 把一个上下文来源建模成"观察（load）+ 比较（codec）+ 渲染（baseline/update/removed）"；`make<A>` 用闭包藏起值类型，让异构来源能统一组合；key 必须命名空间化且唯一，重复即失败。
- 三个内置来源：`core/environment`、`core/date`（baseline 与 update 措辞不同，变化时只报新值）、`core/instructions`（唯一带 removed 渲染器，含路径逃逸防御与 `OPENCODE_DISABLE_PROJECT_CONFIG` 开关）。
- `Unavailable` 用 stale-while-revalidate 语义把"暂时观察不到"与"确实被删除"区分开；初始化时任一来源 unavailable 即 `InitializationBlocked`——宁可阻塞也不持久化不完整基线（fail-closed）。
- `reconcile` 在每个 Safe Provider-Turn Boundary 惰性对账，返回 Unchanged / Updated（多来源变化合并成一条系统消息）/ Replace 三选一；上下文变化绝不异步推送、绝不唤醒空闲会话。
- `context-epoch.ts` 用 `revision` 乐观并发 + `retryRevisionMismatch` 重试 + `fence`（Location/agent 栅栏）把基线和快照稳健地钉进数据库；跨 agent 替换被阻塞时抛 `AgentReplacementBlocked` 而非暴露旧 agent 特权基线。
- `baselineSeq` 是历史投影的切线：基线之前的旧系统消息离开模型视野但作为审计历史留库；模型每轮看到 = 不可变基线 + 切点之后的历史投影 + 安全边界上的增量系统消息。

## 动手实验

1. **实验一：给 System Context 加一个来源** — 阅读 `system-context/builtins.ts` 里 `core/environment` 来源的完整定义。仿照它，在纸上设计一个 `core/git-branch` 来源：写出它的 `key`、`load`（提示：读当前分支名）、`baseline` 与 `update` 文案。思考：它需要 `removed` 渲染器吗？为什么？
2. **实验二：追一次"对账"** — 在 `system-context/index.ts` 里读 `reconcileObservation`。假设某会话上一次快照里有 `core/date`，今天日期变了、`core/instructions` 文件被删（但它有 removed 渲染器）。手推一遍：函数会返回什么 `_tag`？最终那条合并消息里包含哪几段文本？
3. **实验三：理解 fail-closed** — 找到 `SystemContext.initialize` 里 `if (unavailable.length > 0) return new InitializationBlocked(...)` 这一行。用一句话向同事解释：如果把这里改成"跳过 unavailable 的来源、用剩下的来源建基线"，会破坏 `CONTEXT.md` 里哪条不变量？可能在真实使用中导致什么后果？
4. **实验四：验证乐观并发** — 在 `context-epoch.ts` 里读 `advance` 和 `retryRevisionMismatch`。假设两个 fiber 同时对 revision=3 的纪元调 `advance`，画出两者的执行时间线：谁成功、谁抛 `RevisionMismatch`、抛了之后会怎样？再说说为什么这里用乐观重试而不是悲观加锁。

> **下一章预告**：本章我们看清了"上下文怎么组装"。第 7 章转向另一条接缝——模型本身怎么接进来。我们会读 `packages/llm/src/protocols/` 下并排摆着的 anthropic-messages、openai-responses、openai-compatible-chat、gemini、bedrock-converse 五套协议适配器，看 OpenCode 如何把五家风格迥异的 provider API，收敛成一个统一的 `llm.stream(request)` 抽象。
