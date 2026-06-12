# 第 5 章　Harness 编排层：把单轮循环串成完整任务

> 循环是心脏，编排层是中枢神经——它决定什么时候跳得快、什么时候休息、什么时候做梦（压缩记忆）。

第 3、4 章我们把 `agent-loop.ts` 和 `Agent` 类拆透了。但它们还停留在「跑一轮对话」的层次。真实的 Agent 任务远不止于此：它要把会话持久化成可恢复的树、要在上下文快满时压缩、要支持 Skill 调用、要在恰当的时机把扩展的钩子串进来。这些「跨越多轮、贯穿整个会话」的编排，由 `packages/agent/src/harness/agent-harness.ts`（1064 行）里的 `AgentHarness` 类完成。

这一章是「执行内核」部分的封顶——读完你将看到一次请求从「用户输入」到「会话落盘」的完整中枢。

---

## 5.1 `AgentHarness` 的全局视角：1064 行里有什么

如果说 `Agent` 类是「拥有一段对话」，那 `AgentHarness` 就是「拥有一个会话（session）的完整生命周期」。它在 `Agent`/`runLoop` 之上，多管了四件大事：

1. **会话持久化**：每条消息、每次模型切换、每次压缩，都写进一棵 JSONL 树（第 11 章细说）。
2. **阶段管理**：用一个 `phase` 状态机保证「同一时刻只做一件事」——不能边跑对话边压缩。
3. **资源编排**：管理 Skill、PromptTemplate、工具集，并把它们组织进系统提示词。
4. **中间件式钩子**：在请求发出前、payload 组装前、各种会话事件处都开了口子，让扩展插入。

构造它需要一组资源（节选自构造函数）：

```ts
constructor(options: AgentHarnessOptions<TSkill, TPromptTemplate, TTool>) {
	this.env = options.env;           // 执行环境（文件系统、shell）
	this.session = options.session;   // 会话存储
	this.resources = options.resources ?? {};  // skills / promptTemplates
	this.systemPrompt = options.systemPrompt;
	// ... 工具去重校验、模型、思考强度、队列模式 ...
}
```

注意它用泛型 `<TSkill, TPromptTemplate, TTool>` 参数化——这让上层应用（`pi-coding-agent`）能用自己的 Skill / 工具类型，而 harness 本身保持中立。这是第 2 章「核心精简」红线的又一次体现。

## 5.2 阶段机（phases）：idle / turn / compaction / branch_summary / retry

`AgentHarness` 的核心是一个极简但极重要的状态机。`phase` 的取值定义在 `harness/types.ts`：

```ts
export type AgentHarnessPhase = "idle" | "turn" | "compaction" | "branch_summary" | "retry";
```

每一个对外的「重型操作」入口，第一件事都是检查并切换 phase。看 `prompt` 和 `compact` 的开头，模式一模一样：

```ts
async prompt(text: string, options?: { images?: ImageContent[] }): Promise<AssistantMessage> {
	if (this.phase !== "idle") throw new AgentHarnessError("busy", "AgentHarness is busy");
	this.phase = "turn";
	// ... try { ... } finally { this.phase = "idle"; }
}

async compact(...): Promise<...> {
	if (this.phase !== "idle") throw new AgentHarnessError("busy", "compact() requires idle harness");
	this.phase = "compaction";
	// ...
}
```

这个状态机解决的是一个看似简单、实则极易出错的问题：**互斥**。一个 Agent 会话里，「跑对话」「压缩历史」「在会话树上跳转」这几件事**绝不能并发**——压缩到一半又来了新消息，会话树就乱了。pi 不用锁、不用信号量，只用一个 `phase` 字段 + 「非 idle 就抛 busy」的约定，就把所有重型操作串行化了。简单、可读、不会死锁。

而 `steer` / `followUp` 的检查方向相反——它们要求**必须不在 idle**（因为操舵只有在「正在跑」的时候才有意义）：

```ts
async steer(text: string, ...): Promise<void> {
	if (this.phase === "idle") throw new AgentHarnessError("invalid_state", "Cannot steer while idle");
	this.steerQueue.push(createUserMessage(text, options?.images));
	await this.emitQueueUpdate();
}
```

`nextTurn` 则两种状态都接受——idle 时它直接 append 进会话，非 idle 时进 `pendingSessionWrites` 队列（见 5.3）。这正是第 4 章「三种消息」在编排层的落地。

## 5.3 中间件式事件钩子与 `pendingSessionWrites`

`AgentHarness` 把扩展的介入点组织成「中间件」风格——在关键节点 emit 事件，收集 handler 的返回值，用它影响后续行为。看两个最有代表性的：

**请求前钩子。** 每次要调模型前，harness 给扩展一次「改写请求选项」的机会（`emitBeforeProviderRequest`）；payload 组装好、真正发出前，再给一次「改写 payload」的机会（`emitBeforeProviderPayload`）。`emitHook` 的合并语义是「后注册的 handler 的非 undefined 返回值覆盖前者」：

```ts
private async emitHook<TType extends keyof AgentHarnessEventResultMap>(event): Promise<...> {
	const handlers = this.getHandlers(event.type);
	if (!handlers || handlers.size === 0) return undefined;
	let lastResult;
	for (const handler of handlers) {
		const result = await handler(event);
		if (result !== undefined) lastResult = result;
	}
	return lastResult;
}
```

这套钩子是 `pi-coding-agent` 扩展系统（第 13 章）的底层支柱——`before_provider_request`、`session_before_compact`、`session_before_tree` 等扩展事件，最终都落到这里。

**`pendingSessionWrites`：阶段安全的写入队列。** 这是 `AgentHarness` 一个精巧的细节。会话写入（append 消息、append 压缩节点）必须在「安全的时刻」落盘，不能在循环跑到一半时穿插进去打乱树结构。所以非 idle 时的写入会先进一个队列：

```ts
async appendMessage(message: AgentMessage): Promise<void> {
	if (this.phase === "idle") {
		await this.session.appendMessage(message);   // idle 时直接落盘
	} else {
		this.pendingSessionWrites.push({ type: "message", message });  // 否则排队
	}
}
```

然后在循环的安全点（每个 turn 之间、run 结束时）调用 `flushPendingSessionWrites()` 统一排空。这保证了**会话树的写入永远是串行、有序、原子的**——即使有扩展在中途想往会话里塞东西。

## 5.4 八个入口方法：编排的对外接口

`AgentHarness` 对外暴露的方法，每一个都对应一种「会话级动作」。把它们摆在一起，就是 pi 编排层的完整 API：

| 方法 | phase 要求 | 作用 |
|---|---|---|
| `prompt(text)` | idle → turn | 用一句新话启动一轮完整任务 |
| `skill(name, extra?)` | idle → turn | 调用一个 Skill（把 SKILL.md 内容格式化成调用文本） |
| `promptFromTemplate(name, args)` | idle → turn | 用一个 PromptTemplate 启动 |
| `steer(text)` | 非 idle | 中途插话（操舵） |
| `followUp(text)` | 非 idle | 排队后续消息 |
| `nextTurn(text)` | 任意 | 下次启动时带上 |
| `compact(instructions?)` | idle → compaction | 主动压缩历史 |
| `navigateTree(targetId, opts?)` | idle → branch_summary | 在会话树上跳转，可选生成分支摘要 |
| `abort()` | — | 中止当前 run |

注意 `skill` 的实现：它并不是什么神秘机制，本质就是「把 Skill 的内容格式化成一段文本，当成 prompt 喂进去」：

```ts
async skill(name: string, additionalInstructions?: string): Promise<AssistantMessage> {
	// ...
	const skill = (turnState.resources.skills ?? []).find((c) => c.name === name);
	if (!skill) throw new AgentHarnessError("invalid_argument", `Unknown skill: ${name}`);
	return await this.executeTurn(turnState, formatSkillInvocation(skill, additionalInstructions));
}
```

这呼应了第 14 章的核心观点：**Skill 不是代码，是「教模型做事的 Markdown」**——调用一个 Skill，就是把它的内容注入对话。

## 5.5 何时继续、何时压缩、何时停止

把前面所有线索串起来，我们追踪一次 `prompt()` 调用的完整编排流程：

1. **入闸**：检查 `phase === "idle"`，切到 `"turn"`，开 run promise。
2. **建 turn 状态**：`createTurnState()` 从会话重建上下文、组装系统提示词（含 Skill 广告）、准备工具集。
3. **跑循环**：把 turn 状态喂给 `runLoop`（通过 `createLoopConfig` 把 harness 的钩子桥接成 `AgentLoopConfig` 的钩子），开始第 3、4 章讲的那套双层循环。
4. **turn 之间**：每个 turn 结束后，`flushPendingSessionWrites()` 落盘、`createTurnState()` 重建下一轮状态——这让扩展在上一轮注入的东西能在下一轮生效。
5. **压缩判断**：上层（`pi-coding-agent`）通常会在 `shouldStopAfterTurn` 或独立调用 `compact()` 里，用 `shouldCompact()`（第 12 章）判断上下文是否逼近窗口上限，逼近就压缩。
6. **收尾**：run 结束，最后一次 `flushPendingSessionWrites()`，phase 切回 `"idle"`。

关于「何时压缩」，看 `compact()` 的真实流程能学到 pi 的严谨：它先 `prepareCompaction` 算出「保留哪些、摘要哪些」，再发 `session_before_compact` 钩子让扩展有机会取消或自带摘要，然后才真正调模型生成摘要、把摘要作为一个 `compaction` 节点 append 进会话树。**压缩不是「删掉旧消息」，而是「在树上插入一个摘要节点，重建上下文时用摘要替代被折叠的部分」**——原始消息一条都没丢，只是不再进入模型上下文。这是第 12 章的核心，这里先埋下伏笔。

至此，「执行内核」三章（3、4、5）讲完。你已经握住了 pi 的骨架：底层是 provider 无关的 `runLoop`，中层是把循环包成对象的 `Agent`，上层是管理会话全生命周期的 `AgentHarness`。接下来的第三部分，我们往下钻一层，看这个骨架是如何跟形形色色的模型对话的。

---

## 本章小结

- `AgentHarness` 在 `Agent`/`runLoop` 之上，多管会话持久化、阶段管理、资源编排、中间件钩子四件事，并用泛型保持对上层类型中立。
- `phase` 状态机（idle/turn/compaction/branch_summary/retry）用「非 idle 就抛 busy」的极简约定，把所有重型操作串行化，无锁、不死锁。
- `pendingSessionWrites` + `flushPendingSessionWrites` 保证会话树的写入永远在安全点串行落盘，即使扩展中途想插入。
- `emitHook` 的「后者覆盖前者」语义是扩展系统的底层支柱；`before_provider_request` 等扩展事件最终都落到这里。
- 八个入口方法构成编排层的完整 API；`skill` 的本质是「把 Markdown 格式化成 prompt」，`compact` 的本质是「往树上插摘要节点」而非删消息。

下一章，进入第三部分，从 `pi-ai` 的类型契约开始，看 pi 如何让模型可插拔。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/agent/src/harness/agent-harness.ts` | 本章主角：阶段机、八个入口、钩子、`pendingSessionWrites` |
| `packages/agent/src/harness/types.ts` | `AgentHarnessPhase`、`AgentHarnessOptions`、各事件结果类型 |
| `packages/agent/src/harness/session/session.ts` | 会话树的 append/导航 API（第 11 章细看） |
| `packages/agent/src/harness/compaction/compaction.ts` | `prepareCompaction`/`compact`（第 12 章细看） |

## 动手实验

1. 在 `agent-harness.ts` 里找出所有 `if (this.phase !== "idle")` 和 `if (this.phase === "idle")` 的检查，列一张「哪个方法要求什么 phase」的表，验证 5.4 节的表格。思考：为什么 `nextTurn` 是唯一对 phase 没有硬性要求的方法？
2. 顺着 `compact()` 读一遍，找出「原始消息没有被删除」的证据——压缩到底往会话里 append 了什么类型的节点？（提示：`session.appendCompaction`。）
3. 给 harness 注册一个 `session_before_compact` 钩子，让它返回 `{ cancel: true }`，验证「扩展能取消压缩」。再让它返回一个自带的 `compaction`，验证「扩展能替模型生成摘要」。
