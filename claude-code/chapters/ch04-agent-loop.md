# 第 4 章　Agent 主循环：query.ts 的双层 while 是怎么跳动的

> 如果说整个 Harness 是一台机器，那么 `src/query.ts` 就是它的曲轴。所有的工具、压缩、权限、记忆，最终都被这根曲轴串成连续的运动。
> 这一章我们逐圈拆解它的两层 `while`——外层决定「这一轮要不要再来一轮」，内层决定「这一次模型调用要不要换台发动机重试」。

## 4.1 两个函数：`query()` 是门面，`queryLoop()` 是引擎

打开 `src/query.ts`，会发现「主循环」其实由两个函数组成。`query()`（第 276 行）是对外的异步生成器门面，签名是 `AsyncGenerator<..., Terminal>`——它产出一串消息/事件，最终 return 一个 `Terminal`。它的职责很轻：建立 Langfuse trace（用于可观测），处理 trace 的所有权（作为子 agent 被调用时复用上层 trace，而不是新建），然后把真正的活儿委托给 `queryLoop()`：

```ts
const ownsTrace = !params.toolUseContext.langfuseTrace
const langfuseTrace =
  params.toolUseContext.langfuseTrace ??
  (isLangfuseEnabled() ? createTrace({...}) : ...)
```

真正的引擎是 `queryLoop()`（第 393 行）。它的核心是第 460 行那句朴素的 `while (true)`，一直转到某个分支 `return` 一个 `Terminal` 为止。这就是「Agent」区别于「Chat」的那根曲轴——Chat 调一次模型就结束，Agent 会因为「模型这一轮调了工具」而**自己决定再转一圈**。

## 4.2 一个 turn 的解剖：从压上下文到点火

`while (true)` 的每一圈就是一个 turn。CCB 在一圈里做的事有严格顺序，理解这个顺序，就理解了整个 Harness 的数据流。按代码行号串起来：

1. **`yield { type: 'stream_request_start' }`（第 495 行）**：告诉前端「我要开始一次模型请求了」，REPL 借此显示 spinner。
2. **`getMessagesAfterCompactBoundary(messages)`（第 523 行）**：取压缩边界之后的消息——之前被 autocompact 摘要掉的历史不再重复送进去。
3. **`applyToolResultBudget(...)`（第 553 行）**：给工具结果上预算，超大的工具输出在这里被裁掉（第 11 章详解）。
4. **snip（第 577 行，`feature('HISTORY_SNIP')` 门控）**：非破坏式裁剪，`snipCompactIfNeeded`。
5. **microcompact（第 588 行，`deps.microcompact(...)`）**：基于 `tool_use_id` 清理可压缩工具的旧结果，注释强调它「never inspects」消息内容，纯按 id 操作。
6. **autocompact（第 638 行附近）+ 阻塞限额预判（第 821 行）+ 预测式 autocompact（第 838 行）**：破坏式摘要，第 12 章详解。如果触到阻塞上限，这里直接 `return { reason: 'blocking_limit' }`（第 830 行）。

这一整段——**预算 → snip → microcompact → autocompact**——是每个 turn 在调模型**之前**必经的「上下文工程流水线」。它的存在是因为模型上下文窗口有限，而 Agent 会越聊越长。把它放在调模型之前，是为了保证「送进模型的永远是被精心裁剪过的、不超窗的上下文」。第 11、12 章会把这条流水线拆到字节级。

做完这些预处理，才轮到点火——调模型。

## 4.3 内层 while：模型调用与 Fallback 重试

第二层循环在第 880 行：`while (attemptWithFallback)`。它包裹的是**一次模型流式调用**，存在的唯一理由是 **Fallback**——主模型不可用时切到备用模型重试。

循环体一开始就把标志位清掉（`attemptWithFallback = false`），默认只跑一次。真正的模型调用是 `deps.callModel({...})`（第 886 行），注意它走的是 `deps` 而非直接 import——这是依赖注入（DI），`src/query/deps.ts` 把 `callModel`、`microcompact`、`autocompact`、`uuid` 四个 I/O 依赖收进 `QueryDeps` 类型，测试时可以注入假实现。`deps.ts` 的注释写得很坦白：这四个依赖「each spied in 6-8 test files today」，抽成 DI 是为了省掉每个测试文件重复的 spy 样板。这是第 17 章「可信度即流水线」原则的一个具体落点——**主循环天生为可测试而设计**。

模型以流式返回，`for await (const message of deps.callModel(...))` 逐块消费。这里有两个关键机制：

**其一，流式 Fallback 的「墓碑」清理。** 如果流到一半触发了 fallback（`streamingFallbackOccured`），那些已经收到的 assistant 消息（尤其是带签名的 thinking 块）会变成孤儿——它们的签名无效，若留着会引发 API 报错。CCB 的处理是给每条孤儿消息 `yield { type: 'tombstone', message: msg }`，让前端和 transcript 把它们删掉，再清空缓冲区重来。`catch (innerError)` 里捕获 `FallbackTriggeredError`，切到 `fallbackModel`、把 `attemptWithFallback` 重新置 `true`，触发内层循环再转一圈。

**其二,工具调用的检测与流式执行。** 流过来的 assistant 消息里，凡是 `content.type === 'tool_use'` 的块都被收集进 `toolUseBlocks`，并把一个布尔量置位：

```ts
if (msgToolUseBlocks.length > 0) {
  toolUseBlocks.push(...msgToolUseBlocks)
  needsFollowUp = true
}
```

`needsFollowUp` 就是「跨过 Chat 那条线」的代码级分界（第 1 章实验二点名让你找的就是它）。它的语义极简：**这一轮模型调了工具，所以处理完工具结果后还得再让模型看一眼**。同时，工具不是等模型全部输出完再跑——`streamingToolExecutor.addTool(...)` 让工具在流式过程中就开始并发执行（第 7 章详解 `StreamingToolExecutor`），完成的结果随流 `yield` 回去。

## 4.4 turn 末尾的两个岔路口：继续还是终止

模型流结束后，`queryLoop` 走到本 turn 最重要的判决——以 `needsFollowUp` 为界分成两半。

**`if (!needsFollowUp)`（第 1335 行）：模型没调工具，可能要收工。** 但「没调工具」不等于「立刻结束」，CCB 在这里塞进了一连串恢复与门控逻辑：

- **prompt-too-long / media-size 恢复**：如果最后一条消息是被「withhold」的 413（上下文太长）或媒体超限错误，先尝试 collapse drain（便宜、保留细粒度），再尝试 reactive compact（完整摘要），各只试一次。对应 `Continue` 的 `collapse_drain_retry` / `reactive_compact_retry`。这里有个被注释明确标注的防死循环设计：`hasAttemptedReactiveCompact` 一旦置位就不重置，否则会陷入「compact → 还是太长 → 报错 → stop hook 阻塞 → 又 compact」烧掉成千上万次 API 调用的死循环。
- **Stop hooks**：`handleStopHooks(...)`（第 1543 行）。如果 hook 要求阻止结束（`preventContinuation`）→ `return { reason: 'stop_hook_prevented' }`；如果 hook 抛出阻塞错误，则把错误塞回消息、以 `transition: { reason: 'stop_hook_blocking' }` 继续转一圈。这是第 16 章「Hook 是权限管线一等公民」的体现——hook 能直接左右主循环的生死。
- **Token budget**（`feature('TOKEN_BUDGET')`）：检查 token 预算，未超则注入一条 nudge 消息以 `token_budget_continuation` 继续。

如果以上都不触发，模型确实干净地说完了话，最终走到 `return { reason: 'completed' }`——这是最常见的正常终止。

**`needsFollowUp` 为真：必然再来一轮。** turn 计数 `nextTurnCount = turnCount + 1`。先检查上限：

```ts
if (maxTurns && nextTurnCount > maxTurns) {
  yield createAttachmentMessage({ type: 'max_turns_reached', ... })
  return { reason: 'max_turns', turnCount: nextTurnCount }
}
```

没到上限，就构造下一圈的 `State`——把 `messagesForQuery`、本轮 assistant 消息、工具结果拼成新的消息历史，`transition: { reason: 'next_turn' }`，赋给 `state` 后 `continue`，外层 `while` 进入下一圈。曲轴转过一整圈。

## 4.5 用类型把「为什么转/为什么停」固化下来

CCB 没有用零散的布尔量来表达循环的去留，而是用 `src/query/transitions.ts` 里两个**判别联合类型**把所有可能性穷举并锁死：

```ts
export type Terminal =
  | { reason: 'completed' } | { reason: 'blocking_limit' }
  | { reason: 'image_error' } | { reason: 'model_error'; error?: unknown }
  | { reason: 'aborted_streaming' } | { reason: 'aborted_tools' }
  | { reason: 'prompt_too_long' } | { reason: 'stop_hook_prevented' }
  | { reason: 'hook_stopped' } | { reason: 'max_turns'; turnCount: number }

export type Continue =
  | { reason: 'collapse_drain_retry'; committed: number }
  | { reason: 'reactive_compact_retry' }
  | { reason: 'max_output_tokens_escalate' }
  | { reason: 'max_output_tokens_recovery'; attempt: number }
  | { reason: 'stop_hook_blocking' }
  | { reason: 'token_budget_continuation' }
  | { reason: 'next_turn' }
```

`Terminal` 有 10 个终止理由，`Continue` 有 7 个继续理由。每一次 `return` 都返回一个带 `reason` 的 `Terminal`，每一次 `continue` 之前都给 `state.transition` 写一个 `Continue`。这是第 17 章「不变量固化进类型」原则的范本——**循环的所有出口和入口都被编译器盯着**，新增一种结束/继续的理由就必须在类型里登记，漏处理会编译报错。读懂了这 17 个 `reason`，就读懂了 CCB 主循环的全部可能命运。

还有几个写死的恢复常量值得记住：`MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3`（输出 token 不够时最多恢复 3 次），`ESCALATED_MAX_TOKENS = 64_000`（升级后的最大输出 token）。它们对应 `Continue` 里的 `max_output_tokens_recovery` 和 `max_output_tokens_escalate`——当模型输出被 `max_tokens` 截断时，主循环会先 escalate 到 64K，仍不够则有限次恢复，而不是直接失败。

## 本章小结

- 主循环由两个函数组成：`query()`（第 276 行）是异步生成器门面，负责 Langfuse trace 所有权；`queryLoop()`（第 393 行）是真正引擎，核心是第 460 行的 `while (true)`。
- 外层 `while (true)` 的每一圈是一个 turn；内层 `while (attemptWithFallback)`（第 880 行）包裹单次模型流式调用，只为 Fallback 重试存在。
- 每个 turn 在调模型**前**必经一条上下文工程流水线：`applyToolResultBudget` → snip → microcompact → autocompact，保证送进模型的上下文永不超窗。
- 模型调用走 `deps.callModel`（DI 注入，`QueryDeps` 含 callModel/microcompact/autocompact/uuid），主循环天生为可测试设计。
- `needsFollowUp` 是「Agent 区别于 Chat」的代码级分界：只要模型本轮产出 `tool_use` 块就置真，意味着处理完工具结果还要再转一圈。
- 工具通过 `StreamingToolExecutor` 在模型流式输出过程中就并发执行，完成即随流 yield。
- 流式 Fallback 会用 `tombstone` 消息清理签名失效的孤儿 thinking 块，再清空缓冲重试，避免「thinking blocks cannot be modified」API 错误。
- turn 末尾以 `needsFollowUp` 分岔：为假则走恢复/stop-hook/token-budget 门控，最常见终点是 `return { reason: 'completed' }`；为真则检查 `maxTurns` 后以 `next_turn` 继续。
- 多处防死循环设计（如 `hasAttemptedReactiveCompact` 不重置）避免「压缩→仍超长→报错→重试」烧光 API 配额。
- `transitions.ts` 用 `Terminal`（10 个 reason）与 `Continue`（7 个 reason）两个判别联合类型把循环所有出入口固化进编译期；恢复常量 `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3`、`ESCALATED_MAX_TOKENS = 64_000`。

## 动手实验

1. **实验一：定位那根曲轴** — 打开 `src/query.ts`，找到第 460 行的 `while (true)` 和第 880 行的 `while (attemptWithFallback)`。在两层循环体里各找一处 `return`/`continue`，确认外层管 turn、内层管模型重试。
2. **实验二：跟踪 `needsFollowUp` 的一生** — 在 `query.ts` 里搜索 `needsFollowUp` 的每一处赋值。记录它在哪被置 `true`（模型产出 tool_use）、在哪被重置为 `false`（fallback/streaming 重试），最后被 `if (!needsFollowUp)` 读取决定去留。这就是 Chat→Agent 的分界线。
3. **实验三：穷举循环的 17 个命运** — 打开 `src/query/transitions.ts`，抄下 `Terminal` 的 10 个 reason 和 `Continue` 的 7 个 reason。回到 `query.ts` 用 grep 找到每个 reason 对应的 `return`/`transition` 位置，理解每种结束/继续分别由什么条件触发。
4. **实验四：验证 DI 的可测试性设计** — 打开 `src/query/deps.ts`，记录 `QueryDeps` 的 4 个字段和 `productionDeps()` 的真实实现。思考：如果要写一个「模型总是返回固定文本、从不调工具」的测试，你需要注入哪个假实现？据此理解为什么主循环要把 `callModel` 做成可注入依赖。

> **下一章预告**：主循环在 `deps.callModel` 处把控制权交给了模型网关。第 5 章我们深入 `src/services/api/claude.ts`——多 provider（Anthropic/OpenAI/Gemini/Grok）如何被统一抽象、流式如何逐块吐出、`withRetry` 的指数退避与 529 限流如何处理、以及 `FallbackTriggeredError` 是怎么让主循环切换备用模型的。
