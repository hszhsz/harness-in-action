# 第 11 章　会话即一棵树：JSONL v3 与投影

> 会话文件不是「聊天记录」，而是一份可回放的操作日志；「当前对话」只是这份日志上的一条路径。

第 5 章我们看到 `AgentSession` 如何在每一轮请求前调用 `buildSessionProjection()`，把持久化状态变成模型可见的消息。这一章把镜头对准那个被投影的源头：`SessionManager` 与它的 JSONL 文件。

pi 的会话模型有三个反直觉之处：

1. **文件是追加式的**，永远只在末尾写新行，不修改旧行；
2. **结构是树**，每条 entry 都带 `parentId`，分支只是「把 leaf 指针挪回去」；
3. **leaf 不落盘**，它是运行期指针，进程重启后由「文件里最后一条带 id 的 entry」重建。

把这三件事想通，分支、回溯、压缩、重试为什么都能用同一套机制实现，就一目了然了。

---

## 11.1 文件长什么样：一行 header + N 行 entry

一个会话文件（`~/.pi/agent/sessions/--<cwd>--/<timestamp>_<id>.jsonl`）的第一行是 header：

```json
{"type":"session","version":3,"id":"...","timestamp":"2026-10-18T...","cwd":"/path/to/project"}
```

之后每一行是一条 entry：

```json
{"type":"message","id":"a1","parentId":null,"timestamp":"...","message":{"role":"user","content":[...]}}
{"type":"message","id":"b2","parentId":"a1","timestamp":"...","message":{"role":"assistant","content":[...]}}
{"type":"model_change","id":"c3","parentId":"b2","timestamp":"...","provider":"anthropic","modelId":"claude-sonnet-4-5"}
```

写入逻辑只有一句（[session-manager.ts:1191](file:///Users/heshaozhong/Code/pi/packages/coding-agent/src/core/session-manager.ts#L1191)）：

```ts
private _appendEntry(entry: SessionEntry): void {
	this.fileEntries.push(entry);
	this.byId.set(entry.id, entry);
	this.leafId = entry.id;
	this._persist(entry);
}
```

追加、索引、移动 leaf、落盘，四件事原子地发生在同一个方法里。读取时则相反：逐行 `JSON.parse`，坏行直接跳过（`parseSessionEntryLine`），保证一个被截断的文件仍能恢复到最后一个完整 entry。

这就是为什么 pi 选 JSONL 而不是 SQLite：

| 维度 | JSONL 树 | SQLite |
|---|---|---|
| 崩溃恢复 | 截断到最后一个完整行即可 | 依赖 WAL / 事务恢复 |
| 可调试性 | `cat`、`jq`、`grep` 直接看 | 需要工具与 schema |
| 并发写 | 单进程追加，天然简单 | 需要处理锁与并发 |
| 分支 | 移动指针 + 追加 | 需要额外的树表与查询 |
| 代价 | 全量加载、无索引 | 大文件与复杂查询更优 |

pi 的会话规模通常在一次任务内，全量加载完全可接受；换来的是「文件本身就是数据库」的透明度。

---

## 11.2 十一种 entry，一棵树

`SessionEntry` 是一个 11 元联合类型（[session-manager.ts:183](file:///Users/heshaozhong/Code/pi/packages/coding-agent/src/core/session-manager.ts#L183)）：

| 类型 | 作用 | 进入模型上下文？ |
|---|---|---|
| `message` | 用户 / 助手 / 工具结果消息 | 是 |
| `thinking_level_change` | 思考等级变更 | 否（只改状态） |
| `model_change` | 切换模型 | 否（只改状态） |
| `usage` | 按模型归因的用量记录 | 否 |
| `compaction` | 压缩检查点 + 摘要 | 是（摘要 + 系统提示词快照） |
| `branch_summary` | 被放弃分支的摘要 | 是（作为摘要消息） |
| `custom` | 扩展私有状态 | 否 |
| `custom_message` | 扩展注入的消息 | 是（转成 custom 消息） |
| `context_edit` | 对早期 entry 的上下文改写 / 省略 | 间接（改变目标 entry 的投影） |
| `label` | 给某条 entry 打标签 | 否 |
| `session_info` | 会话显示名等元数据 | 否 |

注意「进入上下文」这一列的分裂：**树里的 entry 不等于模型看到的消息**。状态类 entry（切模型、改思考等级、打标签）只影响重建时的设置；`custom` 只是扩展的储物柜；`context_edit` 自己不出现在上下文里，但它会改变别人。

这正是第 5 章说的「投影」：`SessionManager` 负责保存全部事实，`buildSessionProjection()` 负责选出「这一次请求该看到什么」。

---

## 11.3 leaf 是运行期指针，不是一条记录

很多树形会话系统会持久化一个 `LeafEntry`，显式记录「当前在哪个节点」。pi 不这么做。

`SessionManager` 里只有一个字段：

```ts
private leafId: string | null = null;
```

它有三个行为：

- **追加时前进**：`_appendEntry()` 把 `leafId` 设为新 entry 的 id；
- **分支时跳回**：`branch(targetId)` 只做 `this.leafId = branchFromId`，不写文件、不改旧数据；
- **重启时重建**：`_buildIndex()` 顺序扫描所有 entry，把 `leafId` 设为**最后一个带 id 的 entry**（[session-manager.ts:1103](file:///Users/heshaozhong/Code/pi/packages/coding-agent/src/core/session-manager.ts#L1103)）。

```ts
for (const entry of this.fileEntries) {
	if (entry.type === "session") continue;
	this.byId.set(entry.id, entry);
	this.leafId = entry.id;
	// ...
}
```

为什么可以这么简单？因为文件是**单进程顺序追加**的：最后写入的那条 entry，天然就是「上一次操作结束时的位置」。不需要额外的一致性记录，也不存在「leaf 记录与文件不同步」的窗口。

`branch()` 的注释把语义说得很清楚：

> Moves the leaf pointer to the specified entry. The next appendXXX() call will create a child of that entry, forming a new branch. Existing entries are not modified or deleted.

于是「回溯并重新提问」的实现是：把 leaf 挪到某条用户消息的父节点，再追加一条新的用户消息。旧分支原封不动留在文件里，只是当前路径不再经过它。

---

## 11.4 从树到上下文：三层函数

从「文件里的所有 entry」到「模型看到的消息」，pi 分了三层，每层只做一件事。

### 第一层：`buildSessionPath()` —— 从 leaf 走到根

给一个 leaf（缺省用最后一条 entry），沿 `parentId` 一路向上，再反转。得到的是一条从根到叶的链。分支在这里被天然排除：不在这条链上的 entry 不参与本次上下文。

### 第二层：`buildContextEntries()` —— 应用压缩检查点

如果路径上存在 `compaction` entry，只保留**最新的那一个**作为检查点，然后：

- 从 `firstKeptEntryId` 开始，保留其后的 entry；
- 丢弃更早的、已被摘要覆盖的 entry；
- 保留压缩 entry 之后的所有 entry。

```ts
const contextEntries: SessionEntry[] = [compaction];
let foundFirstKept = false;
for (let i = 0; i < compactionIdx; i++) {
	const entry = path[i];
	if (entry.id === compaction.firstKeptEntryId) foundFirstKept = true;
	if (foundFirstKept && !(entry.type === "message" && entry.message.role === "system")) {
		contextEntries.push(entry);
	}
}
contextEntries.push(...path.slice(compactionIdx + 1));
```

这里有个容易忽略的细节：**被保留区间里的旧 `system` 消息会被跳过**。因为压缩 entry 自己带了 `systemMessage` 快照，再保留旧的系统消息会导致重复声明提示词和工具。

### 第三层：`buildSessionProjection()` —— 应用 context_edit，保留来源

这一层做两件事：

1. 收集路径上所有 `context_edit`，以 `targetId` 为键（后写覆盖先写）；
2. 把每条 context entry 转成消息时，套用对应的编辑。

返回值不是裸消息数组，而是带来源的投影：

```ts
export interface ProjectedSessionEntry {
	sourceEntry: SessionEntry;   // 这条消息来自哪条原始 entry
	messages: AgentMessage[];    // 编辑后模型可见的消息（可能为空）
}
```

这个 `sourceEntry` 是后面很多能力的基石：UI 要把消息映射回可回滚的节点、重试要判断「失败响应是否已被省略」、边界事件要计算「如果提交这些草稿，上下文会变成什么样」，都需要知道消息的出处。

`buildSessionContext()` 只是投影的薄包装，返回 `{ messages, thinkingLevel, model }`。

---

## 11.5 `context_edit`：不改历史，也能改上下文

传统做法里，「删掉一条消息」意味着重写文件。pi 的选择是：**追加一条编辑记录**。

```ts
export interface ContextEditEntry extends SessionEntryBase {
	type: "context_edit";
	targetId: string;
	/** Null omits the target from model context. A value replaces only its content. */
	replacement: { content: ContextEditableContent } | null;
}
```

它有两个语义：

- `replacement: null`：目标 entry 从模型上下文中**完全消失**（但仍在树里、仍在 UI 里）；
- `replacement: { content }`：只替换内容，保留消息的角色、时间戳、工具调用 id 等元数据。

投影时的实现非常克制（`projectContextEntry`）：

```ts
if (replacement === null) return [];
return messages.map((message) => {
	if (message.role !== "user" && message.role !== "assistant"
		&& message.role !== "toolResult" && message.role !== "custom") return message;
	return { ...message, content: replacement.content } as AgentMessage;
});
```

写入前有一组校验（`appendContextEdit`）：

- 目标必须存在，且**在当前分支上**（不能编辑另一条分支的历史）；
- 目标必须是 `message`（user / assistant / toolResult）或 `custom_message`；
- 字符串替换对 assistant / toolResult 会被规范化为一个 text block。

谁在用这个能力？两个真实场景：

1. **自动重试时省略失败响应**。模型 API 报错或返回空响应后，`AgentSession` 会把那条失败的 assistant 消息（以及它的工具结果）用 `context_edit(null)` 从投影里省略，再重试。这样既保留了「发生过一次失败」的审计事实，又不会让下一次请求看到半截响应。
2. **边界事件的草稿**。扩展在 `turn_end` 里可以返回 `ContextEditEntryDraft`，和其他草稿一起原子提交——这是第 5 章讲的 boundary 机制。

**为什么必须是追加式？** 因为并发与崩溃。如果编辑是就地重写，一次崩溃可能留下「改了一半」的历史；追加式编辑则保证任何时刻文件都是合法前缀，最坏情况只是最后一条编辑丢失。

---

## 11.6 分支、标签与「另存一条路径」

### `branchWithSummary()`：离开时留一封说明书

直接 `branch()` 会让被放弃的分支从上下文里消失——模型不知道自己刚才做过什么。`branchWithSummary()` 在挪动 leaf 之前，先记录旧 leaf：

```ts
const fromId = this.leafId ?? "root";
this.leafId = branchFromId;
const entry: BranchSummaryEntry = {
	type: "branch_summary",
	id: generateId(this.byId),
	parentId: branchFromId,   // 挂在新分支的起点上
	fromId,                   // 指向被放弃分支的末端
	summary,
	// ...
};
```

于是新路径上会多出一条摘要消息，模型看到的是「你之前尝试过 X，结论是 Y，现在从 Z 继续」。`fromId` 同时保留了「摘要的是哪条分支」这一溯源信息。

`AgentSession.navigateTree()` 把这套流程包装成用户操作：找到旧 leaf 与目标的共同祖先、收集要摘要的 entry、调用摘要模型、再 `branchWithSummary()`。它还处理了两个工程细节：流式输出/压缩期间禁止导航，以及把目标若是用户消息时的原文回填到编辑器（方便改写后重发）。

### `createBranchedSession()`：把一条路径导出成新会话

分支会话文件里可能同时躺着好几条路径。`createBranchedSession(leafId)` 只取「根 → 指定 leaf」这一条，写成一个新文件。

难点在标签：`label` 本身也是树上的 entry，可能有别的 entry 以它为父节点。直接删掉标签会让后代成为孤儿。pi 的做法是先把标签从路径里摘出来，记录「哪个标签 id 被谁替换」，再把剩余 entry 重新串成一条链，最后按解析后的标签映射重建标签 entry。这个细节说明了一件事：**在树模型里，任何「删除节点」的操作都必须回答「它的孩子怎么办」。**

### 标签与元数据

`label` 是用户书签，`session_info` 是会话显示名。两者都作为普通 entry 追加，因此：

- 标签可以随时改（追加一条 `label: undefined` 即取消）；
- 会话改名不影响历史路径；
- 二者都不进入模型上下文。

---

## 11.7 版本迁移：v1 → v2 → v3

`CURRENT_SESSION_VERSION = 3`。加载旧文件时，`migrateToCurrentVersion()` 会就地升级并重写文件：

| 版本 | 变化 |
|---|---|
| v1 → v2 | 给每条 entry 补 `id`/`parentId`（按文件顺序串成链）；压缩的 `firstKeptEntryIndex` 转成 `firstKeptEntryId` |
| v2 → v3 | 消息角色 `hookMessage` 重命名为 `custom` |

迁移策略是「读时升级 + 立即重写」，而不是保留多套读取逻辑。这让运行时代码只需要面对一种形状，代价是旧文件在第一次打开时会被改写——所以 pi 会先完整解析，确认是合法会话后才动文件；空文件会补 header，非空但无法解析的文件则直接报错、不修改。

---

## 11.8 这套设计的取舍

| 收益 | 代价 |
|---|---|
| 分支/回溯/重试共用一套追加机制 | 文件只增不减，长会话需要压缩来控制体积 |
| 崩溃安全：永远是合法前缀 | 全量加载，超大会话启动变慢 |
| 上下文可重建、可审计 | 投影逻辑有三层，理解成本高于「直接存消息数组」 |
| `context_edit` 保留历史又改变上下文 | 同一 entry 可能有多个编辑，需按顺序覆盖 |
| 来源可追溯（`sourceEntry`） | 每条消息多一层映射，UI 与重试逻辑要维护 id 关系 |

如果你要借鉴这套设计，最小实现只需要：**一条链 + leaf 指针 + 追加式编辑**。树、标签、压缩都可以后面再加；但「永不就地改写历史」这条原则最好一开始就守住，否则分支和恢复会立刻变成一堆特判。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/coding-agent/src/core/session-manager.ts` | 全部：header、11 种 entry、追加/索引/leaf、投影、分支、迁移 |
| `packages/coding-agent/src/core/agent-session.ts` | 消费投影；重试时用 `context_edit` 省略失败响应；`navigateTree()` |
| `packages/coding-agent/src/core/agent-session-runtime.ts` | 换会话、fork、import 时的文件切换 |
| `packages/coding-agent/src/core/messages.ts` | `createCompactionSummaryMessage` / `createBranchSummaryMessage` / `createCustomMessage` |
| `packages/coding-agent/src/core/extensions/types.ts` | `ContextEditEntryDraft` 与 boundary 草稿类型 |

## 动手实验

1. 跑一个包含两轮对话的会话，`cat` 对应的 `.jsonl`，画出 `id`/`parentId` 链。然后用 `/tree` 回到第一条消息前，再问一个不同的问题，观察文件末尾新增了哪些行、旧行是否被修改。
2. 手动构造一个带 `compaction` 的文件（或跑一次 `/compact`），用 `buildContextEntries()` 的思路手算：哪些 entry 会被保留？如果 `firstKeptEntryId` 指向一条 `system` 消息，会发生什么？
3. 找到一次触发自动重试的失败请求（或人为让模型报错），检查会话文件里是否出现 `context_edit` 且 `replacement: null`。思考：为什么 pi 不直接删掉那条 assistant 消息？
