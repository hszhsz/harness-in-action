# 第 11 章　会话即一棵树：append-only 的 JSONL 存储

> 大多数 Agent 把会话当成一个不断变长的数组。pi 把它当成一棵只增不改的树——于是「回溯」「分支」「压缩」从「危险的数组手术」变成了「安全的指针移动」。

第 5 章我们讲 `AgentHarness` 时埋过一个伏笔:会话被持久化成一棵 JSONL 树,每条消息、每次模型切换、每次压缩都是树上的一个节点。这一章我们把这棵树彻底挖开:它的节点长什么样、为什么要 append-only、「当前对话」如何从树里重建出来,以及这套设计如何让回溯和分支变得廉价而安全。这是 pi 第二个独到的架构决策(第一个是上一章的「无权限」)。

---

## 11.1 不是数组,是一棵 append-only 的树

先破除一个直觉。大多数人想象「会话历史」时,脑子里是一个数组:`[user, assistant, user, assistant, ...]`,新消息往后 push,要回退就 pop 或 splice。

pi 不这么干。它的会话是一棵**只追加(append-only)的树**:

- 每一个事件——一条消息、一次模型切换、一次压缩、一个书签——都是树上的一个**节点(entry)**,带唯一 `id` 和指向父节点的 `parentId`。
- 树永远只增长,**从不修改、从不删除**已有节点。
- 「当前对话」不是「整棵树」,而是从某个**叶子(leaf)节点回溯到根**的一条**路径**。

这个模型来自 `SessionTreeEntryBase`(`harness/types.ts`),所有节点的共同基类:

```ts
export interface SessionTreeEntryBase {
	type: string;
	id: string;
	parentId: string | null;   // ← 指向父节点,串成树
	timestamp: string;
}
```

一个 `parentId` 字段,就把线性的 JSONL 文件(每行一个 JSON 节点)组织成了一棵树。文件物理上是顺序追加的(这就是「JSONL」——JSON Lines),但逻辑上,靠 `parentId` 的指向构成树形结构。

> **给 Agent 开发者的启示**:「append-only + 用 parentId 串树」是一个被低估的存储模式。它的威力在于——**任何「修改历史」的操作,都不必真的去改历史,只需在树上长出新枝、再把「当前叶子」指过去。** 历史永远不可变,于是永远可追溯、可回放、可对比。这与 Git 的对象模型同源。

## 11.2 十一种节点:会话里发生的一切都是一个 entry

pi 把「会话里能发生的事」穷举成了一个 `SessionTreeEntry` 联合类型,共十一种:

```ts
export type SessionTreeEntry =
	| MessageEntry              // 一条消息(user/assistant/toolResult)
	| ThinkingLevelChangeEntry  // 切换思考强度
	| ModelChangeEntry          // 切换模型
	| ActiveToolsChangeEntry    // 改变激活的工具集
	| CompactionEntry           // 一次压缩(含摘要)
	| BranchSummaryEntry        // 一次分支跳转的摘要
	| CustomEntry               // 扩展自定义数据(不进上下文)
	| CustomMessageEntry        // 扩展自定义消息(进上下文)
	| LabelEntry                // 书签/标记
	| SessionInfoEntry          // 会话名等元信息
	| LeafEntry;                // 记录「当前叶子是谁」
```

把它们分一下类,设计意图就清楚了:

| 类别 | 节点 | 作用 |
|---|---|---|
| **对话内容** | `MessageEntry` / `CustomMessageEntry` | 真正进入模型上下文的东西 |
| **状态变更** | `ModelChangeEntry` / `ThinkingLevelChangeEntry` / `ActiveToolsChangeEntry` | 记录「从这点起,模型/思考强度/工具集变了」 |
| **历史管理** | `CompactionEntry` / `BranchSummaryEntry` | 压缩、分支跳转留下的摘要节点(第 12 章细说) |
| **元数据** | `LabelEntry` / `SessionInfoEntry` / `CustomEntry` | 书签、会话名、扩展私有数据 |
| **指针** | `LeafEntry` | 记录「当前活跃叶子」——决定哪条路径是「当前对话」 |

最值得玩味的是「状态变更」这一类。注意 pi 不是「改一个全局的 model 变量」,而是**往树上追加一个 `ModelChangeEntry` 节点**。为什么?因为 append-only——状态变更本身也是历史的一部分。于是「这条消息是用哪个模型生成的」「从哪一步起切到了便宜模型」,全都能从树上精确回溯出来。这也呼应第 6 章 `cost` 做成一等公民:状态变更可追溯,成本核算才能精确到「每条消息用的哪个模型、什么价」。

`Session` 类(`session/session.ts`)为每种节点提供了一个 `appendXxx` 方法,而它们的实现**长得一模一样**——都是「创建 id、把当前 leaf 当 parentId、追加」:

```ts
async appendMessage(message: AgentMessage): Promise<string> {
	return this.appendTypedEntry({
		type: "message",
		id: await this.storage.createEntryId(),
		parentId: await this.storage.getLeafId(),   // ← 挂到当前叶子下
		timestamp: new Date().toISOString(),
		message,
	} satisfies MessageEntry);
}
```

每一次 append,都是「在当前叶子下面长一个新节点,新节点成为新叶子」。这就是树的生长方式。

## 11.3 从树到「当前对话」:`buildSessionContext` 的重建

树是持久化的形态,但模型要的是一个**线性的消息列表**。从树到列表的转换,是 `buildSessionContext`(`session/session.ts`)干的活,也是这棵树设计里最精妙的一段。

第一步,拿到「当前路径」。`getBranch` → `getPathToRoot(leafId)` 从当前叶子一路回溯到根,得到一个 `SessionTreeEntry[]`——这就是「当前对话」对应的那条路径。

第二步,`buildSessionContext` 遍历这条路径,干两件事:

**1. 折叠「状态变更」节点为最终状态。** 遍历时,后出现的 `model_change` 覆盖先前的、最后一个 `thinking_level_change` 生效……于是一堆历史状态变更被「折叠」成「当前应该用哪个模型、什么思考强度、哪套工具」:

```ts
for (const entry of pathEntries) {
	if (entry.type === "model_change") model = { provider: entry.provider, modelId: entry.modelId };
	else if (entry.type === "thinking_level_change") thinkingLevel = entry.thinkingLevel;
	else if (entry.type === "active_tools_change") activeToolNames = [...entry.activeToolNames];
	else if (entry.type === "compaction") compaction = entry;
	// assistant 消息也会更新 model(记录它实际用的模型)
}
```

**2. 把「内容」节点收集成消息列表。** `MessageEntry` 直接取出消息,`CustomMessageEntry` 重建成自定义消息,`BranchSummaryEntry` 重建成摘要消息——而 `CustomEntry`/`LabelEntry`/`SessionInfoEntry` 这些**不进上下文**,被自然跳过:

```ts
const appendMessage = (entry) => {
	if (entry.type === "message") messages.push(entry.message);
	else if (entry.type === "custom_message") messages.push(createCustomMessage(...));
	else if (entry.type === "branch_summary" && entry.summary) messages.push(createBranchSummaryMessage(...));
	// 其余类型不进上下文
};
```

注意这里的分层:**「树里存的」是全集(连书签、扩展私有数据都存),「喂给模型的」是子集**。同一棵树,既是完整的审计日志,又能投影出一个精简的模型上下文。这是 append-only 树的又一个红利——存得多不等于喂得多。

第三步,**压缩感知的重建**。如果路径上有 `CompactionEntry`,重建逻辑会拐个弯:先放摘要,再只放「被保留的尾部消息」,中间被折叠的部分用摘要替代:

```ts
if (compaction) {
	messages.push(createCompactionSummaryMessage(compaction.summary, ...));
	// 找到压缩点,从 firstKeptEntryId 开始才收集消息
	let foundFirstKept = false;
	for (let i = 0; i < compactionIdx; i++) {
		if (pathEntries[i].id === compaction.firstKeptEntryId) foundFirstKept = true;
		if (foundFirstKept) appendMessage(pathEntries[i]);
	}
	// 压缩点之后的全收
	for (let i = compactionIdx + 1; i < pathEntries.length; i++) appendMessage(pathEntries[i]);
}
```

这正是第 5 章那句话的代码兑现:**「压缩不是删掉旧消息,而是在树上插一个摘要节点,重建上下文时用摘要替代被折叠的部分」**——原始消息节点一条都还在树上,只是 `buildSessionContext` 重建时跳过了它们。第 12 章我们会专门讲压缩怎么算「保留谁、摘要谁」。

## 11.4 回溯与分支:为什么这棵树让它们变廉价

理解了「当前对话 = 从叶子到根的路径」,回溯和分支就变成了一件极其简单的事:**移动叶子指针。**

看 `moveTo`(`session/session.ts`):

```ts
async moveTo(entryId: string | null, summary?: {...}): Promise<string | undefined> {
	if (entryId !== null && !(await this.storage.getEntry(entryId)))
		throw new SessionError("not_found", ...);
	await this.storage.setLeafId(entryId);   // ← 就这一行:把叶子指过去
	if (!summary) return undefined;
	// 可选:为这次跳转留一个 branch_summary 节点
	return this.appendTypedEntry({ type: "branch_summary", ... });
}
```

**回溯**:想回到三轮之前的某条消息?把 leaf 指向那个节点的 id,`buildSessionContext` 自然就只重建到那里为止的路径——后面那些消息节点还在树上,只是不在「当前路径」上了。

**分支**:在回溯后的那个点上接着对话,新消息会以那个节点为 `parentId` 长出来——于是从那个分叉点,树长出了第二条枝。原来那条枝完好无损。你随时能把 leaf 切回去,在两条平行的对话历史间来回跳。

对比一下数组模型:回溯要 `splice` 掉后面的元素(数据就此丢失),分支几乎无法表达(一个数组只有一条线)。而树模型里,**这两个操作都不破坏任何数据,只是改变「当前看哪条路径」**。这就是 append-only 的核心价值:

> 把「修改」变成「新增 + 改指针」,你就得到了免费的撤销、免费的分支、免费的审计。代价只是存储——而文本节点的存储成本,在今天几乎可以忽略。

`moveTo` 还能顺手记一个 `BranchSummaryEntry`:跳转时让模型(或扩展)生成一段「我刚才在那条分支上做了什么」的摘要,作为新枝的第一个节点。这样即便你切换到一条全新的分支,模型也能带着「另一条线的成果」继续——这是第 12 章分支摘要的内容,这里先点到。

## 11.5 存储是可插拔的:`SessionStorage` 接口

最后,呼应 pi 一以贯之的「机制与策略分离」。`Session` 类本身不碰磁盘——它依赖一个 `SessionStorage` 接口(`harness/types.ts`):

```ts
export interface SessionStorage<TMetadata> {
	getLeafId(): Promise<string | null>;
	setLeafId(leafId: string | null): Promise<void>;
	createEntryId(): Promise<string>;
	appendEntry(entry: SessionTreeEntry): Promise<void>;
	getEntry(id: string): Promise<SessionTreeEntry | undefined>;
	getPathToRoot(leafId: string | null): Promise<SessionTreeEntry[]>;
	getEntries(): Promise<SessionTreeEntry[]>;
	// ...
}
```

`Session` 的所有树操作(append、回溯、分支)都只通过这个接口表达。默认实现是 JSONL 文件(`JsonlSessionMetadata` 带 `cwd`/`path`),但你完全可以实现一个内存版(测试用)、一个数据库版、一个远程版——`Session` 一行不改。这又是第 2 章那条接缝的同款套路:**核心定义接口,实现可替换。** pi 自己就有一个内存仓库实现,专门用于测试,不落任何磁盘。

至此,「会话是一棵树」的骨架就清楚了:十一种 append-only 节点 + parentId 串成树 + leaf 指针选路径 + `buildSessionContext` 投影出模型上下文 + 可插拔存储。下一章,我们专门攻克这棵树上最复杂的两种节点——`CompactionEntry` 和 `BranchSummaryEntry`,看 pi 如何在上下文窗口快满时,优雅地「做梦」(压缩记忆)。

---

## 本章小结

- pi 的会话不是数组,而是一棵 append-only 的 JSONL 树:每个事件是一个带 `id`/`parentId` 的节点,只增不改;「当前对话」是从某个叶子(leaf)回溯到根的一条路径。
- 十一种 `SessionTreeEntry`(消息/状态变更/历史管理/元数据/指针)穷举了会话里能发生的一切;连「切模型」「改思考强度」都是追加节点而非改全局变量,使一切可回溯。
- `buildSessionContext` 把树投影成线性消息列表:折叠状态变更为最终状态、收集内容节点为消息、跳过不进上下文的节点;遇到 `CompactionEntry` 则「摘要 + 保留尾部」重建——证明压缩不删原始消息。
- 回溯与分支被简化成「移动 leaf 指针」(`moveTo` 一行 `setLeafId`):不破坏任何数据,得到免费的撤销、分支与审计;`moveTo` 可顺手记 `BranchSummaryEntry`。
- 存储通过 `SessionStorage` 接口可插拔(JSONL / 内存 / 数据库 / 远程),`Session` 的树逻辑与存储后端解耦。

下一章,深入这棵树上最复杂的两种节点——压缩(`CompactionEntry`)与分支摘要(`BranchSummaryEntry`)。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/agent/src/harness/types.ts` | `SessionTreeEntry` 十一种节点、`SessionStorage` 接口、`SessionContext` |
| `packages/agent/src/harness/session/session.ts` | `Session` 类:各 `appendXxx`、`getBranch`、`moveTo`;`buildSessionContext` 重建逻辑 |
| `packages/agent/src/harness/messages.ts` | `createCompactionSummaryMessage`/`createBranchSummaryMessage`/`createCustomMessage` |

## 动手实验

1. 在纸上画一棵会话树:3 条消息后,把 leaf 移回第 1 条消息,再发 2 条新消息。标出此时的「当前路径」和「被遗忘但仍在树上」的那两条旧消息。验证「回溯 = 改 leaf 指针,数据不丢」。
2. 读 `buildSessionContext`,列出哪些 `SessionTreeEntry` 类型会进入 `messages`、哪些被跳过。回答:为什么 `LabelEntry`(书签)和 `CustomEntry` 存在树里却不进模型上下文?
3. 跟踪 `moveTo` 的 `summary` 分支:它在跳转后追加了一个什么类型的节点、`parentId` 指向谁?设想「切到一条新分支但希望模型记得旧分支的结论」,说明 `BranchSummaryEntry` 如何实现这一点。
