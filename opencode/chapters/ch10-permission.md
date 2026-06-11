# 第 10 章　权限系统：allow / ask / deny 三态闸门与"记住这次选择"

> 第 8、9 章里反复出现的那句 `permission.assert(...)`——无论是 `read` 读文件、`bash` 跑命令、还是 `question` 向用户提问——最终都汇流到这一章的同一道闸门。
> OpenCode 用一套不到 330 行的 `permission.ts`，把"一个工具调用到底能不能落地"建模成三种效果（allow / ask / deny）、一条 `findLast` 匹配规则、外加一个"记住这次选择"的持久化机制。本章把这道闸门彻底拆开。

第 8 章我们说过一个关键区分：物化阶段的权限过滤是 **catalog 可见性**，不是 **execution authorization**。一个工具即便出现在模型的菜单里，它真正执行时还要再过一遍自己内部的 `permission.assert`。本章读的就是那道执行授权闸门——`packages/core/src/permission.ts`（329 行）、`permission/schema.ts`、`permission/saved.ts`，以及 `util/wildcard.ts` 和 `plugin/agent.ts` 里那些内置 agent 的权限配置。读完你会理解：v2 的权限系统不是一张静态的 ACL 表，而是一台**带"询问—回复—记忆"反馈回路的状态机**，并且它的每一个默认值都朝着 fail-closed 的方向倾斜。

## 10.1 三种效果，一条规则

权限系统的原子在 `permission/schema.ts` 里，小得令人意外：

```ts
Effect = Schema.Literals(["allow", "deny", "ask"])
Rule   = Schema.Struct({ action, resource, effect })   // 三个字段
Ruleset = Schema.Array(Rule)
```

一条规则就是"对哪个 **action**（动作，比如 `bash`、`read`、`edit`）、哪个 **resource**（资源，比如命令字符串、文件路径），施加哪种 **effect**"。三种 effect 穷尽了一个调用的全部命运：

- **allow**：直接放行，不打扰用户。
- **deny**：直接拒绝，抛 `DeniedError`。
- **ask**：停下来问用户——这是整个系统最有特色的一态，下面 10.3 节专讲。

而把"一组规则 + 一个具体调用"碾成"一条生效规则"的，是 `evaluate`：

```ts
export function evaluate(action: string, resource: string, ...rulesets: Ruleset[]): Rule {
  return (
    rulesets
      .flat()
      .findLast((rule) => Wildcard.match(action, rule.action) && Wildcard.match(resource, rule.resource)) ?? {
      action,
      resource: "*",
      effect: "ask",
    }
  )
}
```

两个细节决定了整套系统的性格：

**其一，用 `findLast` 而不是 `find`——后定义的规则赢。** 规则数组从前往后是"从通用到特例"的叠放：你可以先写一条 `{action:"read", resource:"*", effect:"allow"}` 放行所有读取，再追加一条 `{action:"read", resource:"*.env", effect:"ask"}` 把 `.env` 文件单独拎出来要求询问。`findLast` 保证后者覆盖前者。这与第 8 章 `whollyDisabled` 里的 `findLast`、第 7 章 provider-error 的"最后匹配优先"是同一种叠加语义——**规则是分层叠放的，越靠后越具体、优先级越高。**

**其二，匹配不到任何规则时，兜底是 `effect: "ask"`，不是 `allow`。** 这是第 1 章 fail-closed 原则在权限层最直接的体现：一个没有被任何规则明确放行的动作，**默认不是"放行"，而是"停下来问人"**。系统宁可多打扰用户一次，也绝不在规则的空白处擅自放行。

## 10.2 Wildcard：把 glob 翻译成正则

`evaluate` 里的 `Wildcard.match(action, rule.action)` 是规则匹配的引擎。`util/wildcard.ts` 把一个 glob 风格的 pattern 翻译成正则：

```ts
// 把 glob 元字符转义，再把 * 和 ? 翻译成正则
//   *  → .*
//   ?  → .
const regex = pattern
  .replace(/[.+^${}()|[\]\\]/g, "\\$&")   // 先转义正则特殊字符
  .replace(/\*/g, ".*")
  .replace(/\?/g, ".")
```

于是 `bash` 工具的命令 `git status` 可以被规则 `{action:"bash", resource:"git *"}` 命中——`git *` 翻译成正则 `git .*`。它还有两个边角处理：尾部的 ` .*`（空格加通配）被特化成 `( .*)?`，让 `git` 这样不带参数的裸命令也能匹配 `git *` 这条规则；在 Windows（win32）上匹配是大小写不敏感的。这正是命令级批准能"批准一类命令前缀"的底层支撑——用户批准 `git *`，之后所有 `git` 子命令都自动放行。

## 10.3 ask 的一生：从 assert 到 Deferred 再到 reply

`assert` 是这套系统对外的主入口，它的骨架是一个三分支判断，包在 `uninterruptibleMask` 里：

```ts
const assert = EffectRuntime.fn("PermissionV2.assert")((input) =>
  EffectRuntime.uninterruptibleMask((restore) =>
    EffectRuntime.gen(function* () {
      const result = yield* evaluateInput(input)
      if (result.effect === "deny") return yield* new DeniedError({ rules: relevant(input, result.rules) })
      if (result.effect === "allow") return
      const item = yield* create(request(input), input.agent)
      return yield* restore(Deferred.await(item.deferred)).pipe(
        EffectRuntime.ensuring(EffectRuntime.sync(() => { pending.delete(item.request.id) })),
      )
    }),
  ),
)
```

- **deny** → 立刻抛 `DeniedError`，并附上 `relevant(input, ...)`（所有匹配该 action 的规则）作为证据——又一次"错误即文档"，告诉调用方"你是被哪些规则拦下的"。
- **allow** → 直接 `return`，零延迟放行。
- **ask** → 这里是精髓：`create` 一个 pending 请求，发布一个 `permission.v2.asked` 事件（UI 据此弹出询问框），然后 **`Deferred.await`——挂起，直到用户回复**。

注意 `Deferred.await` 被 `restore(...)` 包着：外层是 `uninterruptibleMask`（保证创建 pending + 注册的过程不被打断），但等待用户回复这一段是**可中断的**——否则一个永远不回复的用户会把整个 fiber 永久焊死。`ensuring` 保证无论结局如何（回复、拒绝、中断），那条 pending 记录都会从 map 里清掉。

用户的回复通过 `reply` 进来，它只有三种 `Reply`：`once` / `always` / `reject`。

**reject 的级联效应**最值得玩味：

```ts
if (input.reply === "reject") {
  yield* Deferred.fail(existing.deferred, input.message ? new CorrectedError({ feedback: input.message }) : new RejectedError())
  pending.delete(input.requestID)
  for (const [id, item] of pending) {
    if (item.request.sessionID !== existing.request.sessionID) continue
    // ……发布 reject 事件、Deferred.fail、删除
  }
  return
}
```

用户拒绝**一个**请求时，**同一 session 里所有其它待决请求也被连带拒绝**。这是一个很人性化的设计判断：当你按下"拒绝"，往往意味着"我不想让它继续这条路线了"，把同会话其它排队的询问一起清掉，免得用户被连环弹窗轰炸。另外注意 reject 还区分两种失败:带 `message` 的变成 `CorrectedError`（用户给了纠正反馈），不带的就是纯 `RejectedError`——这两个错误在第 5 章主循环里有不同的处理路径（`isQuestionRejected` 会中断整个循环）。

## 10.4 "always"：把这次选择记进项目

`once` 是"就这一次放行"，回复后 `Deferred.succeed`、删除 pending，下次同样的调用还会再问。而 **`always`** 触发"记住这次选择"——但有一个前提：请求里必须带了 `save` 字段。

```ts
if (input.reply === "always" && existing.request.save?.length) {
  yield* saved.add({
    projectID: location.project.id,
    action: existing.request.action,
    resources: existing.request.save,
  })
}
```

回想第 9 章 `bash` 的 `permission.assert`——它传的是 `save: [input.command]`，也就是说"记住"记的是**整条命令**。保存动作落到 `permission/saved.ts` 的 `PermissionTable`，每条 save 的 resource 插入一行（`onConflictDoNothing` 去重）。关键在它的作用域：`projectID: location.project.id`——**保存的规则是项目级的**，绑定在当前项目上，而不是全局、也不是单次会话。换个项目，这些"永久放行"就不再生效。

保存的规则怎么重新参与判断？看 `savedRules`——它把保存的行统统翻译成 `effect: "allow"` 的规则：

```ts
const savedRules = function* () {
  return (yield* saved.list({ projectID: location.project.id })).map(
    (item): Rule => ({ action: item.action, resource: item.resource, effect: "allow" }),
  )
}
```

然后在 `evaluateInput` 里，**保存的规则被叠在 agent 配置规则的后面**（因此优先级更高）：

```ts
const evaluateInput = function* (input) {
  const rules = yield* configured(input.sessionID, input.agent)
  if (denied(input, rules)) return { effect: "deny" as const, rules }   // ← agent 的 deny 是硬否决
  const all = [...rules, ...(yield* savedRules())]                       // ← 再叠上 saved
  const effects = input.resources.map((resource) => evaluate(input.action, resource, all).effect)
  const effect = effects.includes("deny") ? "deny" : effects.includes("ask") ? "ask" : "allow"
  return { effect, rules: all }
}
```

这里有一道**重要的安全顺序**：`denied(input, rules)` **先单独用 agent 配置规则查一遍 deny**，命中就直接判 deny，**根本不给 saved 规则翻盘的机会**。换句话说——**用户"记住的允许"永远无法覆盖 agent 配置里的明确拒绝。** 一个 agent 如果配了 `{action:"edit", resource:"*", effect:"deny"}`，用户哪怕之前对某次编辑点过"always allow"，也照样会被拦下。saved 规则只能把 ask 升级成 allow，不能把 deny 降级成 allow。这是 fail-closed 的又一处体现：**记忆只能放宽"询问"，不能突破"禁止"。**

还有一个细腻的收尾：当一次 `always` 回复保存了新规则后，`reply` 会**回头扫一遍同 session 里其它 pending 请求**，凡是被这条新规则变成"全 allow"的，就一并自动放行（发 `always` 事件、`Deferred.succeed`）。用户批准一次 `git *`，那些正排队等着问 `git push`、`git commit` 的请求当场被一起放行——回复的收益立刻惠及所有等价的待决请求。

## 10.5 fail-closed 的源头：缺失 agent 即"全拒绝"

`evaluateInput` 第一步调的 `configured`，回答的是"这次调用该用哪套规则"：

```ts
const configured = function* (sessionID, agentID?) {
  const session = yield* sessions.get(sessionID)
  if (!session) return yield* new SessionV2.NotFoundError({ sessionID })
  const agent = yield* agents.resolve(agentID ?? session.agent)
  return agent?.permissions ?? missingAgentPermissions
}
```

规则的来源是**当前生效 agent 的 `permissions` 字段**。但万一 agent 解析不出来（`agent` 为 undefined）呢？兜底值 `missingAgentPermissions` 是整章最该记住的一行：

```ts
const missingAgentPermissions: Ruleset = [{ action: "*", resource: "*", effect: "deny" }]
```

**找不到 agent，就用"对所有动作、所有资源、一律拒绝"兜底。** 这是 fail-closed 思想最纯粹的一句代码：当系统对"该用谁的权限"这个问题失去把握时，它的选择不是"放行"也不是"问用户"，而是**彻底关死**。一个权限上下文残缺的调用，绝不允许侥幸落地。对比 10.1 节"匹配不到规则默认 ask"——那是规则集**存在但没覆盖**的情况，可以问;而这里是规则集**根本取不到**的情况，只能拒。两种缺失、两种 fail-closed 的强度，区分得很清楚。

## 10.6 关于"子 agent 权限只能更窄"：澄清一个直觉

第 9 章末尾的预告里有一句话需要在这里诚实地校正：子 agent 的权限是否"只能比父 agent 更窄"？**读代码后的答案是：v2 并没有一个"用父 agent 规则去交集裁剪子 agent 规则"的运行时机制。** 每个 agent——无论 primary 还是 subagent——都携带**自己独立的一整套 `permissions` 规则**，互不派生。`plugin/agent.ts` 里那一串内置 agent 的配置把这件事讲得很清楚。

先看共用的 `defaults`（默认基线），它本身就内建了不少 fail-closed 的细节：

```ts
const defaults: PermissionV2.Ruleset = [
  { action: "*", resource: "*", effect: "allow" },          // 基线：默认放行
  ...readonlyExternalDirectory,                              // 外部目录默认 ask（白名单除外）
  { action: "question", resource: "*", effect: "deny" },     // 默认禁止提问
  { action: "plan_enter", resource: "*", effect: "deny" },
  { action: "plan_exit", resource: "*", effect: "deny" },
  { action: "read", resource: "*", effect: "allow" },
  { action: "read", resource: "*.env", effect: "ask" },      // 读 .env 要问
  { action: "read", resource: "*.env.*", effect: "ask" },
  { action: "read", resource: "*.env.example", effect: "allow" },  // 但 .env.example 放行
]
```

注意那三条 `.env` 规则的叠放顺序：先 `read *` 全放行，再把 `*.env`/`*.env.*` 升级成 ask，最后又把 `*.env.example`（不含真实密钥的模板）降回 allow。`findLast` 让这种"通用放行 + 敏感收紧 + 例外放宽"的精细分层成为可能——**默认放行读取，但读密钥文件必须问一声。**

然后每个 agent 在 `defaults` 之上**追加（push）**自己的特化规则。这里"追加"是字面意义的 `Array.push`——`PermissionV2.merge` 就是 `rulesets.flat()`，纯拼接，**不做任何交集或裁剪**：

- **default / plan**（primary）：在 defaults 上放开 `question`，plan 额外用 `{action:"edit", resource:"*", effect:"deny"}` 禁掉所有编辑（只留 plan 目录可写）。
- **explore**（subagent）：先 `{action:"*", resource:"*", effect:"deny"}` **整体关死**，再逐条放开 `grep`/`glob`/`webfetch`/`websearch`/`read`——一个**严格只读**的搜索 agent。
- **general**（subagent）：defaults 基础上只 deny 掉 `todowrite`。
- **compaction / title / summary**（hidden primary）：统统 `{action:"*", resource:"*", effect:"deny"}`——这些内部 agent 只负责生成文本，**不需要任何工具权限，于是一律关死**。

所以"子 agent 更窄"在**实践效果**上是成立的——`explore` 确实比 default 窄得多——但它窄是因为**它的配置里写死了 deny**，而不是因为有个引擎拿父 agent 去裁它。这个区分很重要：v2 选择了**显式配置每个 agent 的完整能力边界**，而非**隐式继承再裁剪**。前者的好处是每个 agent 的权限是"所见即所得"的一张完整清单，读配置就能确知它能干什么，不必在脑子里做集合运算。这正是第 1 章"约束只收紧、且收紧得显式可见"原则在 agent 编排层的落地——**每个 agent 的笼子有多大，都明明白白写在它自己的规则里。**

## 10.7 question 工具：权限与交互的合流

`tool/question.ts` 是个很好的收束样本，它让"权限 assert"和"向用户提问"两条线在一个工具里合流：

```ts
execute: (input, context) =>
  permission.assert({
    action: "question",
    resources: ["*"],
    sessionID: context.sessionID,
    agent: context.agent,
    source: { type: "tool", messageID: context.assistantMessageID, callID: context.toolCallID },
  }).pipe(
    Effect.mapError(() => new ToolFailure({ message: "Permission denied: question" })),
    Effect.andThen(question.ask({ ... }).pipe(Effect.orDie)),
    Effect.map((answers) => ({ answers })),
  )
```

它**先 assert 一道 `question` 权限**，过了才真正调 `question.ask` 弹出问题。回想 10.6 的 defaults——`question` 默认是 `deny` 的，只有 default/plan 这种 primary agent 显式放开了它。于是 `explore`、`compaction` 这类 agent 即便想调 `question` 工具，也会在这道 assert 上被 `DeniedError` 拦下，翻译成 `Permission denied: question`。**一个 subagent 不能擅自打断用户去提问**——这条交互纪律，靠的正是权限闸门来兜底。注意它构造的 `source` 字段严格按 schema（`type:"tool"` + messageID + callID）填，让每个权限请求都能回溯到是哪条 assistant 消息的哪个工具调用发起的——审计链路的一环。

## 本章小结

- 权限系统三态：`allow`（放行）/ `deny`（抛 `DeniedError` 并附匹配规则）/ `ask`（挂起问用户）；规则只有 `{action, resource, effect}` 三字段。
- `evaluate` 用 `findLast` + `Wildcard.match` 选出生效规则——**后定义的规则赢**（通用→特例叠放）；匹配不到任何规则时兜底 `ask`，而非 allow（fail-closed）。
- `Wildcard.match` 把 glob 翻成正则（`*`→`.*`、`?`→`.`，尾部 ` .*`→`( .*)?`，win32 大小写不敏感），支撑"批准一类命令前缀"。
- `assert` 在 `uninterruptibleMask` 里三分支：deny 抛错、allow 直返、ask 创建 pending + 发 `asked` 事件 + 可中断地 `Deferred.await`；`reply` 收 `once`/`always`/`reject`。
- `reject` 级联拒绝同 session 所有待决请求（带 message 变 `CorrectedError`，否则 `RejectedError`）；`always`+`save` 把规则写进**项目级** `PermissionTable`，并回扫放行其它等价 pending。
- 安全顺序：`denied` 先单用 agent 配置规则查 deny、saved 规则无权翻盘——**记忆只能把 ask 升级成 allow，不能把 deny 降级**；agent 解析失败兜底 `missingAgentPermissions`（全 deny）。
- 子 agent 没有"父规则交集裁剪"机制——每个 agent 携带独立完整 ruleset，在 `defaults` 上 `push` 追加特化（explore 先全 deny 再逐条放开读，compaction/title/summary 全 deny）；"更窄"是显式配置的结果，不是隐式继承。
- `question` 工具先 assert `question` 权限再提问，而 `question` 默认 deny——subagent 无法擅自打断用户。

## 动手实验

1. **实验一：手推一次 `evaluate`** — 给定规则数组 `[{action:"read",resource:"*",effect:"allow"}, {action:"read",resource:"*.env",effect:"ask"}, {action:"read",resource:"*.env.example",effect:"allow"}]`。对 `read config.json`、`read .env`、`read .env.example` 三个调用分别推演 `findLast` 命中哪条、最终 effect 是什么。再把数组顺序倒过来，结果会怎么变？由此解释"为什么规则顺序是从通用到特例"。
2. **实验二：验证"记忆不能突破禁止"** — 在 `permission.ts` 读 `evaluateInput` 里 `denied(input, rules)` 那道先行检查。假设 agent 配了 `{action:"edit",resource:"*",effect:"deny"}`，而用户之前对某次 edit 点过 `always`（saved 里有一条 edit 的 allow）。手推 `evaluateInput` 的返回 effect，并用一句话解释这道顺序为什么是安全关键。
3. **实验三：读懂 explore 的笼子** — 在 `plugin/agent.ts` 找到 `explore` agent 的 permissions。它先 push 了一条什么规则把自己"整体关死"？又逐条放开了哪几个动作?推演：explore 调 `bash ls`、`grep foo`、`write a.txt` 分别会得到 allow / ask / deny 中的哪个？
4. **实验四：追一次 reject 级联** — 在 `reply` 的 `reject` 分支里，假设同一 session 有 3 个 pending 请求、另一 session 有 1 个。用户对其中一个点 reject。手推：哪几个 Deferred 会被 fail、哪个不受影响?再说说为什么"拒绝一个就连带拒绝同会话其余"在交互上是合理的。

> **下一章预告**：本章和第 5、6 章都反复提到一个词——"压缩"（compaction）。当一次长对话把上下文撑到模型窗口上限，OpenCode 怎么在不丢关键信息的前提下把历史"折叠"起来、重建一个新基线、再让循环无缝续跑？第 11 章进入上下文压缩，读 `session/compaction.ts` 与它和 Context Epoch 的协作，看 OpenCode 如何用一个专门的 `compaction` agent（还记得它的权限是"全 deny"吗）把溢出的历史安全地收束成一段锚定摘要。
