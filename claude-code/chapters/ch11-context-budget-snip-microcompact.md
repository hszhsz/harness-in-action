# 第 11 章　上下文瘦身（上）：预算、snip 与 microcompact

> 第 4 章拆解主循环时，我们看到每个 turn 在调模型**之前**都要经过一条流水线：`applyToolResultBudget → snip → microcompact → autocompact`。这一章和下一章把这条流水线拆到字节级。本章讲前三段——它们的共同特点是**不动语义、只省空间**：要么给单条工具结果上预算、要么按标记裁掉旧消息、要么纯按 id 清空旧工具结果。真正会「读懂并改写历史」的破坏式摘要（autocompact）留到第 12 章。

## 11.1 为什么需要瘦身：窗口有限，对话无限

模型的上下文窗口是有上限的，但 Agent 会越聊越长——每一轮的工具调用、文件内容、命令输出都堆进历史。如果不加干预，迟早会撞上 413（prompt too long）。CCB 的应对不是「等撞墙再处理」，而是在每个 turn 调模型前主动瘦身。而瘦身有个铁律：**能不丢语义就不丢**。所以流水线被设计成从「最无损」到「最有损」的梯度——预算和 snip、microcompact 都属于无损或近无损的前段，autocompact 这种「读懂内容再摘要」的破坏式操作是最后手段。

## 11.2 工具结果预算：三层闸门防止单点撑爆

`src/constants/toolLimits.ts` 里有一组层层递进的尺寸常量，它们构成了工具结果的三层防线：

```ts
export const DEFAULT_MAX_RESULT_SIZE_CHARS = 50_000      // 单工具默认上限
export const MAX_TOOL_RESULT_TOKENS = 100_000           // 单结果 token 硬顶(~400KB)
export const MAX_TOOL_RESULTS_PER_MESSAGE_CHARS = 200_000 // 单条消息聚合预算
export const BYTES_PER_TOKEN = 4                         // token 估算系数
```

**第一层是单工具上限**。`DEFAULT_MAX_RESULT_SIZE_CHARS = 50_000` 是系统级默认，注释说明个别工具可以声明更低的 `maxResultSizeChars`（第 7 章见过：Bash 30K、Grep 20K），但这个常量「acts as a system-wide cap regardless of what tools declare」——是无论工具怎么声明都不能突破的天花板。超过就把结果存到磁盘文件，模型只收到一段预览 + 文件路径。

**第二层是单结果 token 硬顶**。`MAX_TOOL_RESULT_TOKENS = 100_000`（约 400KB），换算成字节就是 `MAX_TOOL_RESULT_BYTES`。

**第三层最有意思——单条消息的聚合预算** `MAX_TOOL_RESULTS_PER_MESSAGE_CHARS = 200_000`。它防的是一个第 7 章并发批的副作用：模型一轮并行调了 N 个工具，每个都没超单工具上限，但**加起来**能把一条 user 消息撑爆。注释举例：「N parallel tools from each hitting the per-tool max and collectively producing e.g. 10 × 40K = 400K in one turn's user message」。所以预算是**按消息独立评估**的——「a 150K result in one turn and a 150K result in the next are both untouched」，这一轮和下一轮各算各的，互不牵连。这个限额可被 GrowthBook 的 `tengu_hawthorn_window` 在运行时覆盖（`getPerMessageBudgetLimit`）。

## 11.3 enforceToolResultBudget：冻结历史决策、保住缓存

执行预算的是 `enforceToolResultBudget`（`src/utils/toolResultStorage.ts` 第 769 行），它的设计核心是一个看似与「省空间」无关、实则至关重要的目标：**别破坏 prompt 缓存**。

它把每条 API 级消息分组独立处理，对每组里的候选工具结果用 `partitionByPriorDecision` 切成三类：

- **`mustReapply`（必须重放）**：之前已经决定要替换的，这次纯 Map 查表重放——「No file I/O, byte-identical, cannot fail」。
- **`frozen`（冻结）**：模型已经原样看过的内容，决策被冻住，永不再替换。
- **`fresh`（新鲜）**：本轮新增消息里的候选，才需要跑预算检查。

为什么要这么大费周章地区分「以前见过」和「这次新增」？注释点破了关键：模型已经看过的内容如果这一轮突然被替换成预览，会导致 prompt 缓存命中失败（cache miss），白白浪费已缓存的 token。所以 CCB 的原则是：**历史决策一旦做出就冻结，新预算只作用于新消息**。一个中途的 flag 变更也只影响 fresh 消息，「prior decisions are frozen via seenIds/replacements」。

只有当一组的 `frozenSize + freshSize > limit` 时，才调 `selectFreshToReplace` 挑出最大的几个 fresh 结果落盘替换。落盘是并发的（`Promise.all`），且有一处偏执的原子性处理：注释解释 `seenIds.add` 必须和 `replacements.set` 在 await 之后**原子地**配对，否则并发读者（未来 subagent 共享状态时）可能看到「X 在 seenIds 里但不在 replacements 里」，把 X 误判为 frozen 而发送完整内容——主线程发预览、它发全文，又是 cache miss。每一次替换都打 `tengu_tool_result_persisted_message_budget` 埋点记录省了多少字节。

`applyToolResultBudget`（第 925 行）是给主循环用的薄封装：`state` 为 `undefined`（feature 关闭）就直接返回原消息（no-op）；否则执行并对新替换触发可选的 transcript 写回回调。注释说明这个回调只对「resume 时会读回记录」的 querySource（`repl_main_thread*`、`agent:*`）传入，临时的 forked agent（compact、sessionMemory、`/btw`）传 `undefined`——又一处「按场景精确控制副作用」的设计。

## 11.4 snip：按标记裁剪，绝不读内容

第二段是 snip（`src/services/compact/snipCompact.ts`），由 `feature('HISTORY_SNIP')` 门控。它和预算的区别在于：预算处理「单条结果太大」，snip 处理「整段历史太长」。但它同样**不读消息内容**，只认一个标记——`snip_boundary`。

`snipCompactIfNeeded`（第 83 行）从后往前扫消息，找最后一个 `subtype === 'snip_boundary'` 的系统消息：

```ts
for (let i = messages.length - 1; i >= 0; i--) {
  const msg = messages[i]!
  if (msg.type === 'system' && msg.subtype === 'snip_boundary') {
    boundaryIdx = i
    removedUuids = msg.snipMetadata?.removedUuids
    break
  }
}
if (boundaryIdx === -1) return { messages, executed: false, tokensFreed: 0 }
```

找到边界后，按边界里记录的 `removedUuids` 列表过滤消息——凡 UUID 在列表里的就丢掉、累计 `tokensFreed`，其余保留。如果边界没带 `removedUuids` 元数据，则保守回退：保留边界及其之后的一切（`messages.slice(boundaryIdx)`）。

注意这里的职责分工：snip 边界本身是由 snip 工具（用户主动 `/force-snip` 或模型调用）写进消息流的，`snipCompactIfNeeded` 只负责**消费**这个标记、执行裁剪。决定「裁哪些」的智能在写入边界时就完成了，执行端只是机械地按 UUID 删除。配套的 `projectSnippedView`（`snipProjection.ts`）做同样的过滤，供不同消费方（query.ts 的模型面消息、QueryEngine 的 `snipReplay`）共用。还有 `SNIP_NUDGE_THRESHOLD = 30`——当历史超过 30 条消息时，注入 `SNIP_NUDGE_TEXT` 提示模型「考虑用 /force-snip 压缩老消息」。**瘦身不全是自动的，CCB 也会提醒模型自己动手。**

## 11.5 microcompact：纯按 tool_use_id 清空旧工具结果

第三段 microcompact（`src/services/compact/microCompact.ts`）是这条无损流水线里最精巧的一段。第 4 章引用过它的关键注释：它「never inspects」消息内容，纯按 `tool_use_id` 操作。它的目标是：把**老的、可压缩工具**的结果内容清空，但保留消息结构。

哪些工具可压缩？`COMPACTABLE_TOOLS` 集合列得很明确：

```ts
const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME, ...SHELL_TOOL_NAMES, GREP_TOOL_NAME,
  GLOB_TOOL_NAME, WEB_SEARCH_TOOL_NAME, WEB_FETCH_TOOL_NAME,
  FILE_EDIT_TOOL_NAME, FILE_WRITE_TOOL_NAME,
])
```

全是「结果可重新获取」的工具——读文件、跑命令、搜索、抓网页。它们的旧结果清掉后，模型若真需要，重新读一遍即可，所以清空是安全的、近无损的。被清空的内容替换成一句 `TIME_BASED_MC_CLEARED_MESSAGE = '[Old tool result content cleared]'`。

为什么是「按 id 而非按内容」？因为按 id 操作意味着 microcompact 完全不需要理解工具结果说了什么，只需知道「这个 tool_use_id 对应的是个可压缩工具、且足够老」。这让它极快、极安全、且不可能因为「误读内容」而出错——这是第 12 章破坏式 autocompact 永远做不到的。

### 时间触发与缓存触发

microcompact 有两条触发路径。第一条是**时间触发**（`maybeTimeBasedMicrocompact` + `timeBasedMCConfig.ts`）。它的逻辑很妙：当距离上一条助手消息的时间超过阈值时触发。注释解释了为什么用时间：

```ts
/** Trigger when (now − last assistant timestamp) exceeds this many minutes.
 *  60 is the safe choice: the server's 1h cache TTL is guaranteed expired */
```

服务端的 prompt 缓存 TTL 是 1 小时——如果对话停了超过 1 小时，缓存反正已经失效了，这时做 microcompact 不会额外损失缓存命中（因为本来就要 cache miss）。所以「缓存反正要失效」的时刻，正是「免费瘦身」的最佳时机。默认 `keepRecent: 5`（保留最近 5 个）。这是把「缓存经济学」算进瘦身时机的精算设计。

第二条是 **cached microcompact**（`cachedMicrocompactPath`，由 `feature('CACHED_MICROCOMPACT')` 门控，ant-only）。它更进一步——**不修改本地消息内容**，而是通过缓存编辑 API（`cache_edits`）在 API 层删除工具结果，从而「without invalidating the cached prefix」（不破坏已缓存的前缀）。它用基于计数的触发/保留阈值（`triggerThreshold` / `keepRecent`），把待删工具的 id 收集进 `pendingCacheEdits` 交给 API 层处理。这是把瘦身这件事下沉到协议层、连本地消息都不碰的极致优化。

## 11.6 三段流水线的共同哲学

把这三段串起来看，它们有一个共同的克制：**都不去「读懂」对话讲了什么**。

- 预算：按字节大小决定哪些结果落盘，不关心结果内容。
- snip：按 UUID 列表删除消息，不关心消息内容。
- microcompact：按 tool_use_id 清空可压缩工具结果，「never inspects」内容。

这种「不理解、只搬运」的设计带来三个好处：快（无需模型调用）、安全（不可能误读误删）、可缓存（操作确定性强，易于保持 prompt 缓存）。它们能省下的空间是有限的——只能清工具结果、删已标记消息——但代价极低。只有当这些无损手段都不够、上下文仍然超窗时，CCB 才会动用第 12 章那个真正「读懂历史再摘要」的破坏式武器：autocompact。

## 本章小结

- 每个 turn 调模型前的瘦身流水线 `预算 → snip → microcompact → autocompact` 按「从无损到有损」排列；本章的前三段共同特点是**不动语义、只省空间**。
- 工具结果三层尺寸闸（`toolLimits.ts`）：单工具上限 `DEFAULT_MAX_RESULT_SIZE_CHARS = 50_000`（系统级硬顶，工具可声明更低但不能更高）、单结果 token 顶 `MAX_TOOL_RESULT_TOKENS = 100_000`（~400KB）、单消息聚合预算 `MAX_TOOL_RESULTS_PER_MESSAGE_CHARS = 200_000`。
- 聚合预算防并发批副作用（10 个工具各 40K 合成 400K），**按消息独立评估**，互不牵连；可被 `tengu_hawthorn_window` 运行时覆盖。
- `enforceToolResultBudget` 把候选切成 `mustReapply`/`frozen`/`fresh`，核心目标是**不破坏 prompt 缓存**：历史决策冻结、新预算只作用 fresh 消息；`seenIds.add` 与 `replacements.set` 原子配对防并发误判导致 cache miss。
- `applyToolResultBudget` 是薄封装：state 为 undefined 即 no-op；transcript 写回回调只对 resume 时读回记录的 querySource 传入。
- snip（`feature('HISTORY_SNIP')`）按 `snip_boundary` 标记的 `removedUuids` 删消息，**不读内容**；无元数据则保守保留边界之后全部；`SNIP_NUDGE_THRESHOLD = 30` 提醒模型主动 `/force-snip`。
- microcompact 纯按 `tool_use_id` 清空 `COMPACTABLE_TOOLS`（Read/Shell/Grep/Glob/WebSearch/WebFetch/Edit/Write，结果可重获取）的旧结果，替换为 `[Old tool result content cleared]`，「never inspects」内容。
- microcompact 时间触发以「服务端 1h 缓存 TTL 必然已失效」为安全点（默认 `keepRecent: 5`）；cached microcompact 通过 `cache_edits` API 在协议层删除、不碰本地消息、不破坏缓存前缀。
- 三段共同哲学「不理解、只搬运」带来快、安全、可缓存三重好处，代价极低；无损手段不够才动用破坏式 autocompact。

## 动手实验

1. **实验一：算清三层预算** — 打开 `src/constants/toolLimits.ts`，记录三个尺寸常量。假设模型一轮并行调了 6 个 Bash（各产出 38K 字符），单工具上限没破，但这条 user 消息会触发哪一层预算？算一算 6×38K 与 `MAX_TOOL_RESULTS_PER_MESSAGE_CHARS` 的关系。
2. **实验二：理解「冻结历史决策」** — 在 `enforceToolResultBudget` 里找到 `partitionByPriorDecision` 的三分类（mustReapply/frozen/fresh）。用自己的话解释：为什么模型已经原样看过的内容这一轮不能被替换成预览？这和 prompt 缓存有什么关系？
3. **实验三：对比 snip 与 microcompact** — 分别读 `snipCompactIfNeeded`（按 UUID）和 microcompact（按 tool_use_id）。列出它们各自删除/清空的对象、触发条件、以及「都不读内容」这一共性带来的好处。
4. **实验四：读懂时间触发的缓存经济学** — 在 `timeBasedMCConfig.ts` 找到「60 minutes」那段注释。论证：为什么选在「距上条助手消息超过 1 小时」时做 microcompact 是「免费」的？如果改成每 5 分钟就 microcompact 一次，会损失什么？

> **下一章预告**：无损手段省不出足够空间时，CCB 才动用破坏式摘要。第 12 章我们拆 `autoCompact.ts` 与 `compact.ts`——它如何把一长段历史真正「读懂并摘要」成一份九段式结构化总结、压缩边界 `getMessagesAfterCompactBoundary` 如何工作、以及 `hasAttemptedReactiveCompact` 那道「防止压缩→仍超长→报错→再压缩」死循环的闸门。
