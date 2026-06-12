# 第 12 章　压缩与分支摘要：上下文窗口的「做梦」机制

> 模型的上下文窗口是有限的，但任务是无限长的。压缩，就是 Agent 在「快记不下了」的时候，把旧记忆梳理成一段摘要、腾出空间继续干活——像人睡觉时把一天的经历整理进长期记忆。

第 11 章我们看清了会话树，并埋下两颗最复杂的节点：`CompactionEntry`（压缩）和 `BranchSummaryEntry`（分支摘要）。这一章把它们彻底讲透。压缩是几乎所有长任务 Agent 都绕不开的工程难题——做不好会丢上下文、会割裂正在进行的工作、会把成本算错。pi 的 `compaction.ts`（757 行）给出了一套相当严谨的答案。

---

## 12.1 何时压缩：`shouldCompact` 的阈值判断

压缩的触发判断极简，就一行核心逻辑（`compaction.ts`）：

```ts
export function shouldCompact(contextTokens, contextWindow, settings): boolean {
	if (!settings.enabled) return false;
	return contextTokens > contextWindow - settings.reserveTokens;
}
```

翻译：**当前上下文 token 数 > （窗口上限 − 预留 token）时，就该压缩了。** `reserveTokens` 是给「压缩本身」留的余量——因为压缩要调一次模型生成摘要，那次调用的输入输出也得占地方，不能等窗口塞满了才动手。默认设置（`DEFAULT_COMPACTION_SETTINGS`）是：

```ts
export const DEFAULT_COMPACTION_SETTINGS = {
	enabled: true,
	reserveTokens: 16384,    // 给摘要生成留 16K
	keepRecentTokens: 20000, // 压缩后保留约 20K 的近期上下文
};
```

那「当前用了多少 token」怎么算？pi 不傻算字符——它优先信任 provider 的真实用量。`estimateContextTokens` 的策略是：**找到最后一条带 usage 的助手消息，用它报告的真实 token 数作为基线，再加上它之后那些消息的估算值**：

```ts
const usageTokens = calculateContextTokens(usageInfo.usage);   // 模型回报的真实用量
let trailingTokens = 0;
for (let i = usageInfo.index + 1; i < messages.length; i++) {
	trailingTokens += estimateTokens(messages[i]);   // 之后的消息才估算
}
return { tokens: usageTokens + trailingTokens, ... };
```

这个「真实基线 + 尾部估算」的混合策略很聪明：模型回报的用量是精确的（厂商按这个收费），但它只覆盖到上一轮；上一轮之后新产生的消息没有官方用量，只能用字符启发式（`estimateTokens` 大致按「字符数 / 4」算）估。两者相加，得到一个既准确又及时的当前用量。第 6 章说「成本是一等公民」，这里就是它的下游——`Usage` 一路流到压缩判断里。

## 12.2 压缩的灵魂：保留谁、摘要谁——`findCutPoint`

压缩最难的不是「生成摘要」，而是**在哪里下刀**：把会话切成「要摘要的旧部分」和「要原样保留的近期部分」。切点选错了，要么丢掉关键近况，要么把正在进行的一步腰斩。

pi 用 `findCutPoint` 解决这个问题，核心思路是「**从最近往回数，累积到 `keepRecentTokens` 为止，那一刀就是切点**」：

```ts
let accumulatedTokens = 0;
for (let i = endIndex - 1; i >= startIndex; i--) {   // 从最新往回扫
	const entry = entries[i];
	if (entry.type !== "message") continue;
	accumulatedTokens += estimateTokens(entry.message);
	if (accumulatedTokens >= keepRecentTokens) {      // 攒够 20K 了
		// 在这个位置附近找一个「合法切点」
		for (let c = 0; c < cutPoints.length; c++) {
			if (cutPoints[c] >= i) { cutIndex = cutPoints[c]; break; }
		}
		break;
	}
}
```

但「合法切点」不是随便哪都能切。`findValidCutPoints` 规定了**只能在「turn 边界」下刀**——即 user / assistant / 自定义消息这类节点，而**绝不能切在 `toolResult` 之前**：

```ts
case "toolResult":
	break;   // ← toolResult 不能作为切点
```

为什么？因为第 7 章讲过，所有厂商都要求「每个工具调用必须跟着工具结果」。如果切点落在「助手发出工具调用」和「工具结果返回」之间，保留下来的部分就会出现一个**孤儿工具调用**（有 call 没 result），模型直接报错。只在完整的 turn 边界切，才能保证保留部分自洽。

> **给 Agent 开发者的启示**：压缩的切点选择，是「token 预算」和「消息完整性」的双重约束求解。先按 token 预算定个大致位置，再吸附到最近的合法 turn 边界——pi 这套「预算定位 + 边界吸附」是个值得直接抄走的范式。

## 12.3 split-turn：连「半个 turn」都要照顾到

还有一个更刁钻的情况：**如果想保留的近期部分，本身就落在一个超大 turn 的中间怎么办？** 比如某一轮 Agent 干了很多事（调了二十个工具），这一个 turn 就超过了 `keepRecentTokens`，切点不得不落在这个 turn 内部。

pi 没有粗暴地把这个 turn 整个扔进「要摘要」或整个留下，而是引入了 **split-turn（劈开的 turn）** 处理（`CutPointResult.isSplitTurn`）。它把这个被劈开的 turn 分成两半：

- **前缀（prefix）**：turn 开头到切点之间——这部分会被**单独**摘要（用专门的 `TURN_PREFIX_SUMMARIZATION_PROMPT`）。
- **后缀（suffix）**：切点到 turn 结尾——这部分**原样保留**。

于是 `compact` 在 split-turn 时会**并行**生成两段摘要，再拼起来：

```ts
if (isSplitTurn && turnPrefixMessages.length > 0) {
	const [historyResult, turnPrefixResult] = await Promise.all([
		generateSummary(messagesToSummarize, ...),       // 旧历史的常规摘要
		generateTurnPrefixSummary(turnPrefixMessages, ...), // 被劈 turn 的前缀摘要
	]);
	summary = `${historyResult.value}\n\n---\n\n**Turn Context (split turn):**\n\n${turnPrefixResult.value}`;
}
```

前缀摘要的提示词专门交代了它的处境：「This is the PREFIX of a turn that was too large to keep. The SUFFIX (recent work) is retained.」——它告诉摘要模型「你在总结一个被劈开的 turn 的前半段，后半段会原样留着」，于是摘要会聚焦「理解保留的后半段所需的背景」。这种对边界情况的细腻处理，是 pi 工程严谨度的一个缩影。

## 12.4 摘要长什么样：结构化的「上下文检查点」

pi 的压缩摘要不是「随便概括两句」，而是一份**严格结构化的检查点文档**（`SUMMARIZATION_PROMPT`）。它要求摘要模型按固定格式输出：

```
## Goal              （用户想达成什么）
## Constraints & Preferences  （约束与偏好）
## Progress
  ### Done / In Progress / Blocked  （已完成/进行中/受阻）
## Key Decisions     （关键决策 + 理由）
## Next Steps        （有序的下一步）
## Critical Context  （继续工作所需的数据/引用）
```

并且反复强调「Preserve exact file paths, function names, and error messages」——精确保留文件路径、函数名、错误信息。这个格式的设计目标很明确：**让另一个 LLM 能仅凭这份摘要无缝接上工作。** 它本质是把「一段冗长的对话」无损降维成「一份任务状态快照」。

更进一步，压缩支持**迭代更新**。如果会话之前已经压缩过一次，这次压缩不会从零重写摘要，而是用 `UPDATE_SUMMARIZATION_PROMPT` 把新发生的事**增量合并**进上一份摘要：「PRESERVE all existing information... ADD new progress... move items from In Progress to Done」。`prepareCompaction` 会找到上一个压缩节点、取出它的 `previousSummary` 传进来：

```ts
if (prevCompactionIndex >= 0) {
	const prevCompaction = pathEntries[prevCompactionIndex];
	previousSummary = prevCompaction.summary;   // 上次的摘要，用于增量更新
	// 这次压缩只处理「上次保留点之后」的历史
	boundaryStart = firstKeptEntryIndex >= 0 ? firstKeptEntryIndex : prevCompactionIndex + 1;
}
```

注意 `boundaryStart`——第二次压缩只从「上一次的保留起点」开始算，不重复处理已经被上次摘要覆盖的部分。这样多次压缩像「滚动的检查点」，每次在前一份基础上前进，而非推倒重来。

pi 还顺手在摘要里附上**文件操作清单**（`CompactionDetails`：读过哪些文件、改过哪些文件），从被摘要的消息里提取出来，拼到摘要末尾。这样即便对话细节被折叠了，「这个任务碰过哪些文件」这个对编码 Agent 极重要的信息也不会丢。

## 12.5 压缩落地：append 一个节点，原始消息不删

讲完「怎么算」，回到第 5、11 章反复强调的那句话，现在能完全证实它了：**压缩不是删消息，是往树上 append 一个 `CompactionEntry` 节点。**

`compact` 的产物是一个 `CompactionResult`：

```ts
return ok({
	summary,                  // 生成的摘要文本
	firstKeptEntryId,         // 从哪个节点起原样保留
	tokensBefore,             // 压缩前的 token 数（用于展示「省了多少」）
	details: { readFiles, modifiedFiles },
});
```

这个结果被 `Session.appendCompaction`（第 11 章见过）写成一个 `CompactionEntry` 节点挂到树上。然后下一次 `buildSessionContext` 重建上下文时，第 11 章 3 节那段「压缩感知重建」就会生效：放摘要 + 从 `firstKeptEntryId` 起保留尾部，跳过中间被折叠的原始消息。

**关键是：被折叠的那些 `MessageEntry` 节点，一个都没从树上删除。** 它们还在 JSONL 文件里、还在树的那条路径上，只是 `buildSessionContext` 重建模型上下文时选择性地跳过了它们。这意味着：

- 你随时可以**回溯**到压缩点之前，看到完整的原始对话（第 11 章的 leaf 指针移动）。
- 压缩是**有损于「喂给模型的上下文」、无损于「存储的历史」**的——审计、调试、复盘时，原始记录永远在。

这正是第 11 章「append-only 树」设计的回报兑现：因为从不真删，压缩这种「看起来很危险的破坏性操作」，实现起来其实是个安全的「新增节点 + 重建时跳过」。

## 12.6 分支摘要：换条路也能带着记忆走

最后是孪生节点 `BranchSummaryEntry`。第 11 章讲过 `moveTo` 可以在跳转到树的另一条分支时，顺手记一个分支摘要。它和压缩共享同一套摘要基础设施（`branch-summarization.ts` 与 `compaction.ts` 同目录），但语义不同：

| | 压缩（Compaction） | 分支摘要（Branch Summary） |
|---|---|---|
| 触发 | 上下文快满，自动/手动 | 在树上跳到另一条分支时 |
| 目的 | 腾出窗口空间 | 把「离开的那条分支的成果」带到新分支 |
| 节点 | `CompactionEntry` | `BranchSummaryEntry` |
| 重建时 | 替代被折叠的旧消息 | 作为新分支的起始上下文 |

场景是这样的：你在分支 A 上探索了一个方案，做了不少工作；现在想回到分叉点、在分支 B 上试另一个方案，但又不想丢掉 A 的结论。`moveTo(分叉点, { summary })` 会把 leaf 切到分叉点、并在新分支起点 append 一个 `BranchSummaryEntry`，里面是「我在 A 上做了什么、得到什么结论」。于是 B 分支的模型一开口就带着 A 的记忆——**换条路走，但没失忆。**

至此第五部分（状态与记忆）讲完。会话树（第 11 章）给了我们可分支、可回溯的存储骨架，压缩与分支摘要（第 12 章）则让 Agent 能在有限窗口里跑无限长的任务、在多条探索路径间自由穿梭而不丢记忆。下一章进入第六部分，我们看 pi 如何用一套「自扩展」系统，让用户在不改核心的前提下给它装上任意能力。

---

## 本章小结

- `shouldCompact` 用「当前 token > 窗口 − `reserveTokens`」触发；当前 token 用「最后一条 usage 真实基线 + 其后消息估算」的混合策略算（默认 reserve 16K、keepRecent 20K）。
- `findCutPoint` 用「从最近往回累积到 `keepRecentTokens`、再吸附到最近的合法 turn 边界」选切点；切点绝不落在 `toolResult` 前，以免产生孤儿工具调用。
- split-turn 处理超大 turn：把被劈 turn 的前缀单独摘要、后缀原样保留，两段摘要并行生成再拼接。
- 摘要是结构化的「上下文检查点」（Goal/Progress/Key Decisions/Next Steps...），目标是让另一个 LLM 无缝接手；支持基于 `previousSummary` 的增量更新，多次压缩像滚动检查点；附带文件操作清单。
- 压缩落地 = 往树上 append 一个 `CompactionEntry`，原始消息节点一条不删；重建时跳过被折叠部分——有损于模型上下文、无损于存储历史，可随时回溯。
- `BranchSummaryEntry` 与压缩共享摘要设施但语义不同：在分支跳转时把「离开分支的成果」带到新分支，实现「换路不失忆」。

下一章，进入第六部分，看 pi 的自扩展 Extension 系统如何让用户不改核心就装上任意能力。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/agent/src/harness/compaction/compaction.ts` | `shouldCompact`/`findCutPoint`/`prepareCompaction`/`compact`、摘要提示词、`DEFAULT_COMPACTION_SETTINGS` |
| `packages/agent/src/harness/compaction/branch-summarization.ts` | 分支摘要生成 |
| `packages/agent/src/harness/compaction/utils.ts` | `serializeConversation`、文件操作提取/格式化 |
| `packages/agent/src/harness/session/session.ts` | `appendCompaction`、`buildSessionContext` 的压缩感知重建（第 11 章） |

## 动手实验

1. 顺着 `findCutPoint` 读 `findValidCutPoints`，列出「哪些节点类型可以当切点、哪些不行」。重点回答：为什么 `toolResult` 被显式排除？如果允许切在它前面，重建出的上下文会出什么错（联系第 7 章的「孤儿工具调用」）？
2. 构造一个「单轮就超过 keepRecentTokens」的超大 turn，跟踪 `prepareCompaction` 里 `isSplitTurn` 为真的分支：`messagesToSummarize`、`turnPrefixMessages` 各装了哪些消息？最终摘要是怎么拼出来的？
3. 对一个会话连续压缩两次，检查第二次 `prepareCompaction` 里 `previousSummary` 和 `boundaryStart` 的取值，解释「为什么第二次不会重复摘要第一次已经覆盖的历史」。再到 JSONL 文件里确认：两次压缩后，最早那批被折叠的原始消息节点是否仍然存在？
