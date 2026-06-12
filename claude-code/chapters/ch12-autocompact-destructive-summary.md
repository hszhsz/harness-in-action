# 第 12 章　上下文瘦身（下）：autocompact 破坏式摘要

> 第 11 章那三段流水线（预算、snip、microcompact）有一个共同的克制——**都不读懂对话讲了什么**，只按字节、UUID、tool_use_id 机械搬运。但它们能省的空间是有限的：清不掉的是对话本身的语义内容。当无损手段都用尽、上下文仍然逼近窗口上限时，CCB 才动用最后的武器——autocompact。它会真正请一个模型把一长段历史**读懂、再摘要**成一份结构化总结，然后用总结替换掉原始历史。这是破坏式的（lossy），所以 CCB 在它周围筑了一圈精算的阈值、防死循环的闸门、以及保住 prompt 缓存的设计。

## 12.1 有效窗口：先给摘要本身留出预算

autocompact 的所有阈值都建立在一个基准上——**有效上下文窗口**（`getEffectiveContextWindowSize`，`autoCompact.ts` 第 33 行）。它不是模型的物理窗口，而是物理窗口**减去给摘要输出预留的 token**：

```ts
const MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000  // 基于 p99.99 摘要输出 17,387 tokens
```

注释点出这个 20K 是怎么定的：「Based on p99.99 of compact summary output being 17,387 tokens」——统计了真实摘要输出的 p99.99 分位数，约 1.7 万 token，向上取整留到 2 万。换句话说，CCB 不假设「摘要很短」，而是按几乎最坏情况给摘要本身留出生成空间。如果不留这块，压缩请求自己就可能因为「输入快撑满 + 还要生成长摘要」而撞上 prompt-too-long——压缩动作本身把自己压死。`CLAUDE_CODE_AUTO_COMPACT_WINDOW` 环境变量还能把窗口再调小，方便测试。

## 12.2 四道阈值：警告、错误、触发、阻断

有了有效窗口，`calculateTokenWarningState`（第 122 行）在它之下划出一组层层逼近的水位线，对应不同的缓冲常量：

```ts
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000
export const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000
export const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000
export const MANUAL_COMPACT_BUFFER_TOKENS = 3_000
```

- **autocompact 触发线** = 有效窗口 − autocompact buffer。这是「该自动压缩了」的水位。
- **warning / error 线** = 触发线再往下 20K，用于 UI 给用户「上下文快满了」的提示。
- **阻断线（blocking limit）** = 物理有效窗口 − `MANUAL_COMPACT_BUFFER_TOKENS`(3K)。这是最后的硬墙——逼近它时连手动操作的余量都快没了。

值得注意的是 autocompact buffer 不是固定值。`getAutocompactBufferTokens`（第 77 行）按窗口大小分档放大：

```ts
if (effectiveWindow >= 800_000) return 50_000
if (effectiveWindow >= 400_000) return 30_000
return AUTOCOMPACT_BUFFER_TOKENS  // 13_000
```

注释解释了为什么大窗口要留更多 headroom：「a single turn can produce proportionally more tokens」——窗口越大，单个 turn 能产出的 token（更长的模型输出 + 更大的工具结果）也越多，所以预留的安全垫必须按比例加厚。配套的 `estimateMaxTurnGrowth` 还把「单 turn 最大增量」估成 `maxOutput + TOOL_RESULT_GROWTH_ESTIMATE(15_000)`，用于在 API 调用前做预测性触发判断。这是把「一个 turn 能膨胀多少」量化进阈值的精算。

## 12.3 shouldAutoCompact：先排除会自我毁灭的场景

`shouldAutoCompact`（第 189 行）在真正触发前，先用一串守卫排除掉「绝对不能在这里压缩」的场景，每一条都对应一个会出事的具体原因：

- **递归守卫**：`querySource === 'session_memory' || querySource === 'compact'` 直接返回 false。注释说明这两个是 forked agent——如果压缩动作自己又触发压缩，会**死锁**。
- **context-collapse 模式**（`feature('CONTEXT_COLLAPSE')`，ant-only）：当上下文坍缩系统接管时，autocompact 必须让位。注释算得很细：autocompact 在有效窗口 −13K（约 93%）触发，正好夹在 collapse 的「90% 开始提交 / 95% 阻断」之间，「would race collapse and usually win, nuking granular context that collapse was about to save」——会和 collapse 抢跑、把 collapse 本来要细粒度保存的上下文一把炸掉。
- **reactive-only 模式**（`tengu_cobalt_raccoon`）：抑制主动压缩，把 413 留给反应式压缩去兜。

只有全部守卫放过，才计算 `tokenCountWithEstimation(messages) - snipTokensFreed` 和触发线比较。这里有个细节：snip 删了消息，但存活的助手消息里记录的 usage 仍反映 snip 前的上下文，token 计数器看不到这部分节省，所以要手动减去 snip 已经算出的 `snipTokensFreed`——又是一处「不同瘦身阶段之间要对账」的工程细节。

## 12.4 九段式结构化摘要：把「读懂历史」钉成模板

autocompact 的核心是 `compactConversation`（`compact.ts` 第 411 行）。它请一个 forked agent 把历史读懂、产出一份摘要。但「摘要」不是让模型自由发挥——`prompt.ts` 里的 `BASE_COMPACT_PROMPT` 把摘要钉成了**固定的九个段落**：

1. **Primary Request and Intent**（主要请求与意图）——捕捉用户所有显式请求
2. **Key Technical Concepts**（关键技术概念）
3. **Files and Code Sections**（涉及的文件与代码段，附完整代码片段）
4. **Errors and fixes**（遇到的错误与修复，特别留意用户的纠正反馈）
5. **Problem Solving**（已解决的问题与进行中的排查）
6. **All user messages**（**所有**非工具结果的用户消息——「critical for understanding the users' feedback and changing intent」）
7. **Pending Tasks**（待办任务）
8. **Current Work**（紧接摘要前正在做的工作）
9. **Optional Next Step**（下一步——并要求附上最近对话的**逐字引用**，「verbatim to ensure there's no drift in task interpretation」）

这个九段结构本身就是一种工程不变量：它强制摘要覆盖「意图、技术、文件、错误、用户原话、待办、现场、下一步」这八九个维度，**让一份破坏式摘要尽可能不丢关键信息**。第 6 段「列出所有用户消息」和第 9 段「逐字引用最近对话」尤其关键——它们防的是摘要漂移：模型在概括时容易把用户的原始诉求改写走样，强制保留原话能把这种漂移压到最低。（你正在读的这本书的写作过程，每次上下文耗尽时收到的「This session is being continued…」摘要，就是这个九段模板的产物。）

prompt 开头还有一段 `NO_TOOLS_PREAMBLE`，反复强调「只输出文本、不要调用任何工具」。注释解释了原因：cache-sharing 的 fork 路径继承了父会话的完整工具集（为了匹配缓存 key），Sonnet 4.6+ 的自适应思考模型有时会忍不住调工具，而 `maxTurns: 1` 下一次被拒的工具调用就意味着这一轮没有文本产出、白白浪费。所以 preamble 放在最前面、把「调工具=失败」说死。摘要产出后，`formatCompactSummary` 会把模型用于打草稿的 `<analysis>` 块剥掉（「a drafting scratchpad... has no informational value once the summary is written」），只留 `<summary>` 的正文。

## 12.5 压缩边界：getMessagesAfterCompactBoundary 如何切割历史

摘要生成后，CCB 不是简单地「删掉旧消息」，而是插入一个**压缩边界标记**（`createCompactBoundaryMessage`，`subtype === 'compact_boundary'`），然后靠这个标记来切割模型面的历史。

`getMessagesAfterCompactBoundary`（`messages.ts` 第 5062 行）就是切割器：

```ts
export function getMessagesAfterCompactBoundary<T>(messages: T[], options?): T[] {
  const boundaryIndex = findLastCompactBoundaryIndex(messages)
  const sliced = boundaryIndex === -1 ? messages : messages.slice(boundaryIndex)
  if (!options?.includeSnipped && feature('HISTORY_SNIP')) {
    return projectSnippedView(sliced as Message[]) as T[]
  }
  return sliced
}
```

`findLastCompactBoundaryIndex` 从后往前扫，找**最后一个**压缩边界——这意味着可以多次压缩，每次只认最新的那道边界，边界之前的原始历史对模型不可见。注意两个设计：第一，无边界时返回全部消息（boundaryIndex === −1）；第二，它还顺手套了第 11 章的 snip 过滤（`projectSnippedView`）——因为 REPL 为了 UI 滚动会保留完整历史，**模型面的路径必须同时应用「压缩切片」和「snip 过滤」**，两层瘦身在同一个出口叠加。这正是第 11 章提到的「snipProjection 供多个消费方共用」的落点。

边界本身是 system 消息，会被 `normalizeMessagesForAPI` 过滤掉，不进 API。压缩后的历史结构是固定的：`boundaryMarker → summaryMessages → messagesToKeep → attachments → hookResults`（`buildPostCompactMessages`）。

## 12.6 压缩之后：不是清空，而是「重建现场」

破坏式摘要最危险的副作用是**丢失工作现场**——模型刚才读过的文件、用过的 skill、正在执行的 plan，压缩后全没了。CCB 在压缩成功后做了一连串「重建现场」的补偿，每一项都有预算上限：

- **恢复最近读过的文件**（`createPostCompactFileAttachments`）：从 `readFileState` 里挑最近访问的文件重新读回，上限 `POST_COMPACT_MAX_FILES_TO_RESTORE = 5` 个、`POST_COMPACT_TOKEN_BUDGET = 50_000` token、每文件 `POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000`。但会跳过「保留尾部已经能看到的文件」（避免重复注入，最多省 25K token），也跳过 CLAUDE.md 和 plan 文件。
- **恢复用过的 skill**（`createSkillAttachmentIfNeeded`）：把本会话调用过的 skill 内容截断后重注入，预算 `POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000`、每 skill 上限 5K。注释解释为什么是「按 skill 截断」而非「整个丢掉」：「instructions at the top of a skill file are usually the critical part」——skill 文件开头的安装/使用说明通常最关键，截断保头部比整体丢弃更好。
- **恢复 plan / plan 模式 / 异步 agent 状态**：让模型压缩后仍知道自己在 plan 模式、有哪些后台 agent 在跑。
- **重新播报工具与 MCP 指令**（delta attachment）：压缩吃掉了之前的工具声明 delta，要从当前状态重新对空播报一遍。

压缩前还会用 `stripImagesFromMessages` 把图片块换成 `[image]` 文本标记——因为「图片对生成摘要没用，反而可能让压缩请求自己撞上 prompt-too-long」，尤其图片多的会话。这又是「压缩动作要防止自己把自己压死」的同一类偏执。

## 12.7 保住缓存：fork 复用前缀，且不设 maxOutputTokens

压缩是一次额外的模型调用，最怕的是它破坏主会话的 prompt 缓存。`streamCompactSummary`（`compact.ts` 第 1170 行）默认走 **cache-sharing 的 forked agent 路径**（`tengu_compact_cache_prefix`，默认 true）——通过发送和主线程**完全一致的缓存 key 参数**（system、tools、model、消息前缀、thinking 配置）来复用主会话已缓存的前缀。注释里有一段极其精细的警告：

```ts
// DO NOT set maxOutputTokens here. ... Setting maxOutputTokens would clamp
// budget_tokens via Math.min(budget, maxOutputTokens-1) in claude.ts,
// creating a thinking config mismatch that invalidates the cache.
```

fork 路径**绝不能设 maxOutputTokens**——因为它会通过 `Math.min(budget, maxOutputTokens-1)` 改写 thinking 的 budget_tokens，造成 thinking 配置和主线程不一致，缓存 key 一变就 cache miss。而 fallback 的普通流式路径不共享缓存，反而可以安全地设 `maxOutputTokensOverride`。一个「设不设上限」的细节，背后是「缓存 key 必须逐字节一致」的硬约束。实验数据（Jan 2026）显示 false 路径 98% cache miss，约占全机队 cache_creation 的 0.76%（约 380 亿 token/天）——所以默认开启共享，GB gate 只作为 kill-switch 保留。

压缩成功后还要调 `notifyCompaction` 重置缓存读取基线——否则压缩造成的上下文骤降会被「缓存中断检测」误报为一次 cache break。BQ 数据：漏掉这一步会让 20% 的 `tengu_prompt_cache_break` 事件变成误报。

## 12.8 三道防死循环的闸门

破坏式压缩最阴险的失败模式是**死循环**：压缩 → 仍超长 → 报错 → 再压缩 → 再超长……每一轮都烧 token 却永远跳不出。CCB 为此设了三道相互独立的闸：

**闸一：连续失败熔断**（`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`）。`autoCompactIfNeeded` 把连续失败次数串在 `tracking.consecutiveFailures` 里，超过 3 次就直接 `return { wasCompacted: false }`，不再尝试。注释附了触目惊心的数据：「BQ 2026-03-10: 1,279 sessions had 50+ consecutive failures (up to 3,272) in a single session, wasting ~250K API calls/day globally」——上下文不可恢复地超限时，没有这道闸的会话会在每个 turn 都发起注定失败的压缩，全球一天白烧 25 万次 API 调用。成功一次就把计数清零。

**闸二：压缩请求自身的 PTL 重试**（`truncateHeadForPTLRetry`，最多 `MAX_PTL_RETRIES = 3` 次）。当压缩请求**自己**撞上 prompt-too-long（CC-1180）时，不是直接放弃，而是按 API-round 分组从**最旧的头部**丢掉若干组再重试——丢多少由 `getPromptTooLongTokenGap` 算出的 token 缺口决定，算不出就丢 20%。它至少保留一组「有东西可摘要」。这是「压缩本身太大」的逃生舱：丢最旧的上下文虽有损，但能让用户脱困。

**闸三：反应式压缩的 hasAttemptedReactiveCompact**（`query.ts`）。当一个 turn 的 API 调用返回 413（prompt-too-long），主循环走的是「反应式压缩」——`tryReactiveCompact`。它的第一个参数就是 `hasAttempted`：

```ts
if (hasAttempted || aborted) return null
```

注释把死循环场景说得很清楚：如果超大的内容恰好落在压缩后**保留的尾部**，那么压缩后的下一个 turn 会**再次** media-error/PTL——「hasAttemptedReactiveCompact prevents a spiral and the error surfaces」。这个 flag 一旦在某个 turn 置为 true，本轮就不再二次反应式压缩，而是直接把错误抛给用户。代码里还特意说明：失败时**不能 fall through 到 stop hooks**，否则「error → hook blocking → retry → error」会形成另一种死亡螺旋（hook 每轮还往里灌 token）。

三道闸防的是同一类病但作用在不同层：闸一防「跨 turn 的主动压缩反复失败」、闸二防「单次压缩请求自己太大」、闸三防「反应式压缩后保留尾部仍超长」。它们共同的逻辑是：**压缩不是万灵药，必须能识别「压了也没用」并及时止损，把错误老实交给用户，而不是无限重试。**

## 12.9 partial compact：可选的前缀/后缀保留

除了「全量压缩」，CCB 还有 `partialCompactConversation`——围绕用户选中的某条消息做局部压缩，由 `direction` 控制两种语义：

- **`'from'`**（摘要选中点之后、保留更早的）：更早的消息原样保留在前，prompt 缓存**得以保住**。
- **`'up_to'`**（摘要选中点之前、保留更晚的）：摘要排在保留消息之前，所以缓存**被失效**（前缀变了）。注释还说明 `'up_to'` 必须剥掉旧的压缩边界/摘要，否则向后扫描的 `findLastCompactBoundaryIndex` 会被陈旧边界误导而丢掉新摘要。

这对应 `prompt.ts` 里三套不同的 prompt 模板（BASE、PARTIAL、PARTIAL_UP_TO），其中 up_to 版本第 8、9 段从「Current Work / Next Step」改成「Work Completed / Context for Continuing Work」——因为它的摘要会**放在续接会话的开头**，语义上是「为后续工作铺垫」而非「描述当前现场」。同一个九段骨架，按摘要在历史中的位置微调措辞，可见这套 prompt 是按场景精调过的。

## 本章小结

- autocompact 是瘦身流水线最后、唯一**破坏式**的一段：请模型把历史读懂再摘要，用摘要替换原始历史；前提是第 11 章无损手段都不够用。
- `getEffectiveContextWindowSize` = 物理窗口 − `MAX_OUTPUT_TOKENS_FOR_SUMMARY(20_000)`（基于摘要输出 p99.99=17,387），先给摘要本身留预算，防压缩请求自我压死。
- 四道水位线：autocompact 触发线（有效窗口 − buffer）、warning/error 线（再往下 20K）、阻断线（有效窗口 − `MANUAL_COMPACT_BUFFER_TOKENS` 3K）；buffer 按窗口分档放大（≥800K→50K、≥400K→30K、否则 `AUTOCOMPACT_BUFFER_TOKENS` 13K）。
- `shouldAutoCompact` 守卫：`session_memory`/`compact` querySource 防 forked agent 死锁、context-collapse 让位（防抢跑炸掉细粒度上下文）、reactive-only 抑制主动压缩；并扣除 `snipTokensFreed` 对账。
- 摘要被钉成**九段结构**（主要意图/技术概念/文件代码/错误修复/问题解决/所有用户消息/待办/当前工作/下一步），第 6 段「列出所有用户消息」+第 9 段「逐字引用」专防摘要漂移；`NO_TOOLS_PREAMBLE` 防 fork 误调工具浪费唯一一轮；`formatCompactSummary` 剥掉 `<analysis>` 草稿。
- `getMessagesAfterCompactBoundary` 按最后一个 `compact_boundary` 切片，无边界返回全部，并叠加 snip 过滤（模型面双层瘦身）；压缩后结构固定 `boundary→summary→keep→attachments→hooks`。
- 压缩后「重建现场」：恢复最近 5 个文件（预算 50K/每文件 5K，跳过已保留与 CLAUDE.md）、用过的 skill（25K，按 skill 截断保头部）、plan/plan 模式/异步 agent、重播工具与 MCP delta；压缩前 `stripImagesFromMessages` 防图片撑爆压缩请求。
- 保缓存：默认走 cache-sharing forked agent 复用主会话前缀，**绝不设 maxOutputTokens**（否则改写 thinking budget→缓存 key 不一致→miss）；压缩后 `notifyCompaction` 重置缓存基线防误报 cache break。
- 三道防死循环闸：连续失败熔断（`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES=3`，防全球日烧 25 万次空压缩）、压缩请求自身 PTL 重试（`truncateHeadForPTLRetry` 丢最旧组，≤3 次）、反应式压缩 `hasAttemptedReactiveCompact`（防保留尾部仍超长的螺旋，且不 fall through 到 stop hooks）。
- `partialCompactConversation` 两种方向：`from`（保前缀、缓存保住）/`up_to`（保后缀、缓存失效、需剥旧边界），对应三套 prompt 模板，up_to 末两段改为「Work Completed / Context for Continuing Work」。

## 动手实验

1. **实验一：算清有效窗口与触发线** — 打开 `autoCompact.ts`，假设模型物理窗口 200K、有效窗口落在 400K 以下档。算出 `getEffectiveContextWindowSize`（减 20K）、autocompact 触发线（再减 13K）、阻断线（有效窗口减 3K）。再换成 800K 窗口，看 buffer 如何跳到 50K，并解释「大窗口要更厚 headroom」的理由。
2. **实验二：通读九段模板** — 读 `prompt.ts` 的 `BASE_COMPACT_PROMPT`，逐一列出九个段落。重点解释：为什么第 6 段要「列出所有用户消息」、第 9 段要「逐字引用最近对话」？它们各自防的是什么信息损失？
3. **实验三：追压缩边界的切割** — 读 `getMessagesAfterCompactBoundary` 和 `findLastCompactBoundaryIndex`。论证：为什么用「最后一个」边界而不是第一个？为什么模型面路径要在切片之外再叠一层 `projectSnippedView`？无边界时返回什么？
4. **实验四：拆三道防死循环闸** — 分别在 `autoCompactIfNeeded`（`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES`）、`truncateHeadForPTLRetry`、`query.ts`（`hasAttemptedReactiveCompact`）找到三道闸。为每一道写出：它防的具体死循环是什么、触发条件、以及失败后把控制权交还给谁。重点解释闸三为什么「不能 fall through 到 stop hooks」。

> **下一章预告**：上下文工程除了「压缩已有历史」，还有「主动注入正确的记忆」。第 13 章我们转向 CCB 的记忆系统——`CLAUDE.md` 分层记忆如何加载与合并、`getMemoryPath` 的各级记忆类型、记忆文件为何在压缩时被排除出「重建现场」的恢复名单，以及相关记忆是怎么在每个 turn 被挑选并注入的。
