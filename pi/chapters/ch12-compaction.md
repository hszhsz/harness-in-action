# 第 12 章　压缩、溢出恢复与分支摘要

> 上下文窗口是硬约束，但「记得什么」是软设计。压缩不是删历史，而是把历史换一种更贵的密度保存。

一个长会话迟早会撞墙：token 越积越多，直到模型返回 `context_length_exceeded`。多数 Agent 在这一刻失败；pi 则把这件事拆成了三条不同的恢复路径，并且区分了「主动压缩」和「事故恢复」。

这一章讲清楚四件事：

1. 什么时候该压缩（触发条件比你想的细）；
2. 从哪里下刀（`findCutPoint` 为什么允许切开一个 turn）；
3. 摘要长什么样、怎么增量更新；
4. 另一条独立路径——分支摘要，它解决的是「回溯后模型失忆」。

---

## 12.1 三种自动压缩，一种手动压缩

`AgentSession._checkCompaction()` 的注释直接列了三种自动场景（[agent-session.ts:2960](file:///Users/heshaozhong/Code/pi/packages/coding-agent/src/core/agent-session.ts#L2960)）：

| 场景 | 触发 | 是否重试当前轮 | 失败响应如何处理 |
|---|---|---|---|
| 1. 溢出 + 重试 | 上下文溢出错误，或可恢复的 `length` 截断 | 是，且只重试一次 | 用 `context_edit(null)` 永久省略 |
| 2. 溢出 + 不重试 | 响应成功完成，但用量已超过窗口 | 否 | 保留完整响应 |
| 3. 阈值 + 不重试 | 用量跨过 `contextWindow - reserveTokens` | 否 | 保留完整响应 |

外加一条手动路径：用户执行 `/compact`，走 `AgentSession.compact()`，可带自定义指令。

### 阈值怎么算

默认设置（`compaction.ts:126`）：

```ts
export const DEFAULT_COMPACTION_SETTINGS: CompactionSettings = {
	enabled: true,
	reserveTokens: 16384,
	keepRecentTokens: 20000,
};
```

判定只有一行：

```ts
return contextTokens > contextWindow - settings.reserveTokens;
```

`reserveTokens` 是留给「压缩这一次调用本身 + 下一轮输出」的余量。注意它是从窗口里**预留**，不是压缩后保留——后者由 `keepRecentTokens` 控制。

### 上下文 token 从哪来

pi 不靠纯字符估算，而是优先用**上一次模型返回的真实 usage**：

```ts
export function estimateContextTokens(messages: AgentMessage[]): ContextUsageEstimate {
	const usageInfo = getLastAssistantUsageInfo(messages);
	if (!usageInfo) { /* 全部按 chars/4 估算 */ }
	const usageTokens = calculateContextTokens(usageInfo.usage);
	let trailingTokens = 0;
	for (let i = usageInfo.index + 1; i < messages.length; i++) {
		trailingTokens += estimateTokens(messages[i]);
	}
	return { tokens: usageTokens + trailingTokens, ... };
}
```

也就是「真实 usage + 之后新增消息的估算」。但这里藏着一个坑：如果 usage 之后发生过 `context_edit` 或 `compaction`，那个 usage 已经不代表当前上下文了。`estimateProjectedContextTokens()` 专门处理这件事——只要发现 usage 对应的 entry 早于最近一次编辑/压缩，就丢弃它，改用全量估算。

**这是压缩系统里最容易出 bug 的地方**：拿一个「压缩前测得的用量」去判断「压缩后要不要再压缩」，会导致刚压缩完立刻又触发压缩。pi 在 `_checkCompaction` 里还额外做了两道防护：跳过早于最新压缩边界的 assistant 消息；估算 usage 来源时检查时间戳。

### 溢出为什么只重试一次

`_overflowRecoveryAttempted` 是一个一次性开关。第一次溢出：省略失败响应 → 压缩 → 重试。如果重试仍溢出，不再无限循环，而是明确报错：

> Context overflow recovery failed after one compact-and-retry attempt. Try reducing context or switching to a larger-context model.

这是很务实的工程判断：连续两次溢出说明「压缩根本救不了这个请求」（比如单条工具结果就超过窗口），继续重试只会烧钱。

---

## 12.2 从哪里下刀：`findCutPoint`

压缩需要回答一个问题：保留最近 N 个 token，前面的全部摘要掉。切点不能随便选。

### 合法切点

```ts
function isCutPointMessage(message: AgentMessage): boolean {
	switch (message.role) {
		case "user": case "assistant": case "bashExecution":
		case "custom": case "branchSummary": case "compactionSummary":
			return true;
		case "toolResult":
			return false;
	}
}
```

**永远不能切在 `toolResult` 上**——工具结果必须跟它的工具调用在一起，否则模型会看到「凭空出现的结果」。切在带工具调用的 assistant 上是安全的，因为它的结果都在后面，会被一起保留。

### 反向累加

算法很直白（`compaction.ts:446`）：从最新一条往前累加估算 token，累计到 `keepRecentTokens` 时停下，取「位于该位置或之后最近的合法切点」。

```ts
for (let i = endIndex - 1; i >= startIndex; i--) {
	accumulatedTokens += tokensOf(entries[i]);
	if (accumulatedTokens >= keepRecentTokens) {
		cutIndex = cutPoints.find((candidate) => candidate >= i) ?? cutPoints.at(-1);
		break;
	}
}
```

一个细节：如果尾部一串工具结果本身就超预算，`find` 会优先选它们前面的 assistant 工具调用，而不是一路退回到第一条消息。否则「保留最近 20k」会退化成「什么都不保留」。

### 切开一个 turn 怎么办

如果切点落在 assistant 消息上（不是用户消息），说明一个 turn 被切成了两半：前半段要摘要，后半段要保留。但模型看到「摘要 + 半个 turn」会困惑——它不知道这个 turn 的起点是什么。

pi 的解法是**两段式摘要**（`compact()` 的 split-turn 分支）：

1. 对切点之前的历史生成常规摘要（可带上一版摘要做增量合并）；
2. 对「被切开的这个 turn 的前半段」单独生成一份 `TURN_PREFIX_SUMMARIZATION_PROMPT` 摘要，只包含 `## Original Request`、`## Progress So Far`、`## Context Needed to Continue`；
3. 两份摘要用 `---` 和 `**Turn Context (split turn):**` 拼接成一条。

这样保留区里的后半段 turn 仍然能接上前半段的意图，而不用把整个 turn 都留下。

---

## 12.3 摘要不是「随便总结一下」

### 固定骨架

主摘要提示词要求模型严格按以下结构输出（`compaction.ts:507`）：

```markdown
## Goal
## Constraints & Preferences
## Progress
### Done
### In Progress
### Blocked
## Key Decisions
## Next Steps
## Critical Context
```

并反复强调「保留精确的文件路径、函数名、错误信息」。这不是审美选择，而是**让摘要可增量更新**的前提：结构固定，下一次才能把新信息合并进对应小节，而不是重写一份风格漂移的散文。

### 增量更新

如果已有上一版摘要，`generateSummary` 会换用 `UPDATE_SUMMARIZATION_PROMPT`，规则包括：

- 保留上一版的所有信息；
- 把 `In Progress` 里完成的项移到 `Done`；
- 根据进展更新 `Next Steps`；
- 已不相关的内容可以删除。

于是长会话的摘要会「滚雪球」式演进，而不是每次从零重写——后者会不断丢失早期决策。

### 文件操作清单

摘要正文之后，pi 还会追加一段机器可读的文件清单：

```xml
<read-files>
src/foo.ts
</read-files>

<modified-files>
src/bar.ts
</modified-files>
```

它来自对历史中工具调用的提取（`extractFileOperations`），并且在多次压缩之间累积。这解决了一个摘要天然会丢的信息：**「我到底改过哪些文件」**。对编码 Agent 来说，这往往比自然语言总结更关键。

### 摘要失败怎么办

摘要本身也是一次 LLM 调用，也会失败。pi 的处理很严格（`getSummarizationFailure`）：

- `stopReason === "error"` → 失败；
- `stopReason === "length"` → **也失败**，因为截断的摘要只是半篇文章，写进检查点会污染后续所有上下文。

所有摘要调用都经过 `completeSummarization()`：

```ts
const requestOptions = { ...options, cacheRetention: "none", sessionId: options.sessionId ?? uuidv7() };
return retryAssistantCall(produce, retry, requestOptions.signal, callbacks);
```

两个设计点：**关闭 prompt cache**（一次性摘要没必要写缓存），以及**复用统一的重试策略**（瞬时断流可重试，确定性错误立即返回）。

---

## 12.4 压缩结果如何落盘

压缩完成后，`SessionManager.appendCompaction()` 写入一条 `compaction` entry：

```ts
const systemMessage = getCurrentSystemMessage(this.buildSessionProjection().messages);
const entry: CompactionEntry = {
	type: "compaction",
	summary,
	firstKeptEntryId: firstKeptEntryId ?? id,
	tokensBefore,
	details,
	usage,
	fromHook,
	...(systemMessage ? { systemMessage: { ...systemMessage, timestamp: ... } } : {}),
};
```

四个字段各有用途：

- `firstKeptEntryId`：投影时从这里开始保留；
- `tokensBefore`：压缩前用量，用于 UI 展示与统计；
- `usage`：生成摘要那次调用的成本，计入账本；
- `systemMessage`：**压缩时刻的系统提示词与工具声明快照**。

最后一点是 1.3 的关键变化。因为系统提示词现在住在转录里（第 3 章），而压缩会丢弃早期 entry——如果不快照，压缩后模型就丢失了「自己有哪些工具、被要求怎么工作」。所以压缩 entry 同时扮演两个角色：**摘要检查点 + 提示词检查点**。

投影时（第 11 章）：

```ts
if (entry.type === "compaction") {
	const summary = createCompactionSummaryMessage(entry.summary, entry.tokensBefore, entry.timestamp);
	return entry.systemMessage ? [entry.systemMessage, summary] : [summary];
}
```

顺序是先系统消息、后摘要。如果路径里还留着更早的 `system` 消息，`buildContextEntries()` 会把它们过滤掉，避免重复声明。

### 扩展的介入点

压缩前后各有一个钩子：

- `session_before_compact`：可以取消压缩，或直接提供一份自定义 `CompactionResult`（`fromHook: true`）；
- `session_compact`：压缩完成后观察结果。

这让「用别的模型压缩」「按团队规范重写摘要」「压缩时额外写一份归档」都成为可能，而不需要改核心。

---

## 12.5 分支摘要：另一种「记忆」

压缩解决的是「向前走太久」，分支摘要解决的是「往回走太远」。

当你用 `/tree` 回到十轮之前重新提问，旧分支上的工作不会消失，但也不在当前路径上。如果不做任何事，模型会完全不知道自己刚才试过什么，很可能把踩过的坑再踩一遍。

### 收集与裁剪

`collectEntriesForBranchSummary()` 做三件事：

1. 从旧 leaf 沿 `parentId` 向上，直到与目标路径的**最深公共祖先**；
2. 收集这一段（按时间正序）；
3. 不因压缩边界而停止——压缩 entry 本身也会被当作上下文。

然后 `prepareBranchEntries()` 按 token 预算**从新到旧**裁剪：保留离现在最近的部分，丢弃更早的。`reserveTokens` 默认 16384，与压缩一致。

### 与压缩摘要的差异

分支摘要的提示词是另一套（`BRANCH_SUMMARY_PROMPT`），但骨架相同（Goal / Constraints / Progress / Key Decisions / Next Steps）。区别在语义：

- 压缩摘要面向「继续做同一件事」；
- 分支摘要面向「你可能以后回来参考」，所以更强调「做过什么、结论是什么、哪些路走不通」。

它同样会追加 `<read-files>` / `<modified-files>`，并且会**累积旧 branch_summary 里的文件操作**——多次分支后仍然知道整体改过哪些文件。

### 落点

摘要生成后，`branchWithSummary()` 在挪动 leaf 的同时写入一条 `branch_summary`：`parentId` 指向新分支起点，`fromId` 指向被放弃分支的末端。于是新路径上多了一条摘要消息，模型看到「此前有一条分支，结论如下」，而旧分支原样留在文件里，随时可以再切回去。

---

## 12.6 这套设计的取舍

| 决策 | 收益 | 代价 |
|---|---|---|
| 用真实 usage + 尾部估算 | 比纯估算准得多 | 需要处理「usage 过期」的失效判断 |
| 允许切开 turn + 双摘要 | 保留预算利用充分 | 多一次 LLM 调用，摘要结构更复杂 |
| 固定摘要骨架 + 增量更新 | 长会话信息衰减小 | 提示词约束强，模型偶尔仍会偏离格式 |
| 摘要 `length` 视为失败 | 不会写入半截检查点 | 极端情况下压缩直接失败，需要用户介入 |
| 压缩时快照系统提示词 | 压缩后工具与提示不丢失 | 系统提示词变更后，旧快照成为历史包袱 |
| 分支摘要与压缩分离 | 两种「记忆」各自最优 | 两套提示词、两套 token 预算，维护成本翻倍 |

一个可以直接借鉴的原则：**压缩不是「把聊天记录变短」，而是「生成一份让下一个模型能无损接手的交接文档」。** 判断摘要好坏的标准不是像不像原文，而是「新模型读完能不能不做重复劳动」。这也是为什么 pi 要固定小节、保留路径、单独列出改过的文件——这些都是在为「接手」服务。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/coding-agent/src/core/compaction/compaction.ts` | 默认设置、token 估算、`shouldCompact`、`findCutPoint`、摘要提示词、`compact()` |
| `packages/coding-agent/src/core/compaction/branch-summarization.ts` | 分支收集、预算裁剪、分支摘要提示词 |
| `packages/coding-agent/src/core/compaction/utils.ts` | 消息序列化、文件操作提取与 `<read-files>` 格式化 |
| `packages/coding-agent/src/core/agent-session.ts` | 三种自动压缩、溢出单次重试、手动 `compact()`、钩子 |
| `packages/coding-agent/src/core/session-manager.ts` | `appendCompaction()` 与系统提示词快照 |
| `packages/coding-agent/src/core/settings-defaults.ts` | `reserveTokens` / `keepRecentTokens` / `branchSummary` 默认值 |

## 动手实验

1. 造一个接近窗口上限的会话，分别触发阈值压缩和溢出压缩，对比 `compaction` entry 里的 `tokensBefore`、`firstKeptEntryId`、`usage`，以及压缩后第一次请求的 `contextTokens`。
2. 故意让摘要调用返回 `stopReason: "length"`（例如把模型输出上限调到极小），确认压缩失败且**没有**写入半截检查点。再打开重试，观察瞬时错误是否被 `retryAssistantCall` 接住。
3. 在一个 turn 中间制造切点（让保留预算刚好落在 assistant 消息上），检查摘要里是否出现 `**Turn Context (split turn):**` 段落，以及保留区是否从工具结果开始。
