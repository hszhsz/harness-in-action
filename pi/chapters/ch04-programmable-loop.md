# 第 4 章　可编程的循环：钩子、队列与错误即数据

> 机制与策略的分界线，不在文档里，而在函数签名里。

第 3 章我们看到 `runLoop` 如何跑；这一章看它如何**被外部改写**。pi 把「循环该不该继续、消息何时插入、工具前后做点什么」全部抽成 `AgentLoopConfig` 上的钩子。读懂这些钩子的调用时机，你就读懂了大半个产品层。

---

## 4.1 `AgentLoopConfig` 的十个钩子

`packages/agent/src/types.ts:193` 定义了配置。除去模型、工具执行模式等普通字段，真正参与控制流的有十个：

| 钩子 | 调用时机 | 能做什么 |
|---|---|---|
| `convertToLlm` | 每次请求前 | `AgentMessage[]` → LLM 可理解的 `Message[]`；过滤 UI 专用消息 |
| `transformContext` | 每次请求前，早于 `convertToLlm` | 在 AgentMessage 层剪枝、注入外部上下文 |
| `getApiKey` | 每次请求前 | 解析短期 token；每轮重新取，避免过期 |
| `prepareNextTurn` | `turn_end` 之后、下一轮开始前 | 替换 context/model/thinkingLevel，注入准备消息（压缩、刷新提示词） |
| `prepareRequest` | **每一次** provider 请求前（含首轮） | 最后一次改写 context/model/thinkingLevel |
| `finishTurn` | 助手消息 + 工具结果都落盘后、`turn_end` 前 | 返回 `end` / `continue`，决定循环去向 |
| `getSteeringMessages` | 每轮工具后、内层继续前 | 取「插话」消息 |
| `getFollowUpMessages` | Agent 本来要停时 | 取「追加任务」消息 |
| `beforeToolCall` | 参数校验后、执行前 | 准入：放行 / 阻止 / 建议终止整批 |
| `afterToolCall` | 工具执行后、结果落盘前 | 按字段覆盖结果：content/details/isError/usage/terminate/structuredContent |

三个「准备类」钩子的分工最容易混淆，用一张时间线记住：

```
turn_end
  └─ prepareNextTurn ── 可以换上下文/模型、注入 messages
       └─ [写入 prepared + steering 消息]
            └─ prepareRequest ── 每次请求前的最后改写
                 └─ streamAssistantResponse
                      └─ 工具执行（before/afterToolCall）
                           └─ finishTurn ── end/continue
                                └─ turn_end ── 回到顶部
```

### 4.1.1 `finishTurn`：不是「清理」，而是「调度决策」

旧版的循环只根据「有没有工具调用 / 有没有队列消息」自然决定是否继续。1.3 增加了 `finishTurn`，让编排层显式表达意图：

- `{ action: "end" }`：**硬退出**。不再轮询队列，直接 `agent_end`。
- `{ action: "continue" }`：保证下一轮至少发一次 provider 请求。如果恰好有工具结果、steering 或 follow-up 要发送，这次请求会被它们「顺带满足」，不会多花一次调用。
- `undefined`：保持默认调度。

`AgentSession` 用它在 `turn_end` 边界派发扩展事件，并根据扩展是否要求继续来覆盖默认决策（第 5 章）。

### 4.1.2 `prepareRequest`：面对「真实模型」的最后一道口子

`prepareRequest` 与 `prepareNextTurn` 的区别不只是调用频率：

- `prepareNextTurn` 是**回合级**的，适合做压缩、系统提示词刷新、模型切换。
- `prepareRequest` 是**请求级**的，适合做虚拟模型路由、按请求注入 header、按当前上下文选择物理模型。

`AgentSession` 正是在 `prepareRequest` 里做「会话树投影 + 虚拟模型解析 + 阈值压缩」这三件事。注意它拿到的是 `request.context`，但会用 `SessionManager.buildSessionProjection()` 重建一份规范上下文——因为 `agent.state.messages` 可能包含尚未投影的临时状态。

### 4.1.3 `afterToolCall`：字段级覆盖，不做深合并

返回值是 `AfterToolCallResult`，语义是「按字段替换」：

```ts
{
	content?: (TextContent | ImageContent)[];
	details?: unknown;
	structuredContent?: JsonValue;
	isError?: boolean;
	usage?: Usage;
	terminate?: boolean;
}
```

两个容易踩的坑：

1. 如果你返回了 `content` 但没有返回 `structuredContent`，原有的结构化结果会被**丢弃**——因为它可能已经与新的文本不一致。
2. 没有深合并。`details` 要么整体替换，要么保持原样。

这种「宁可丢，也不拼」的策略，避免了扩展把两个不同来源的结果拼成一个自相矛盾的对象。

## 4.2 队列：steering、follow-up 与消费时机

`Agent` 内部有两个 `PendingMessageQueue`，默认模式都是 `"one-at-a-time"`：

```ts
class PendingMessageQueue {
	private messages: AgentMessage[] = [];
	public mode: QueueMode;

	peek(): AgentMessage[] {
		if (this.mode === "all") return this.messages.slice();
		const first = this.messages[0];
		return first ? [first] : [];
	}

	drain(): AgentMessage[] {
		const drained = this.peek();
		this.messages = this.messages.slice(drained.length);
		return drained;
	}
}
```

`one-at-a-time` 的意义是**保序且可打断**：用户连续发三条消息，Agent 不会把三条一次性塞进同一轮，而是在每个可插入点消费一条，让模型有时间对第一条作出反应。

两者的语义差异：

| 队列 | 消费点 | 典型用途 |
|---|---|---|
| steering | 工具批次结束后、下一次请求前 | 「先别改那个文件，先跑测试」 |
| follow-up | 内层循环即将退出时 | 「做完这个，再把文档也更新了」 |

`Agent` 还提供了 `peekQueuedMessages()`：steering 优先，其次 follow-up。UI 用它显示「下一条将发送什么」，但不会消费队列。

## 4.3 `Agent` 类：把循环包成对象

`packages/agent/src/agent.ts` 的 `Agent` 是低层循环的有状态外壳。它解决三个问题：**状态归谁管、事件发给谁、并发如何约束**。

### 4.3.1 状态初始化：系统提示词就是首条消息

```ts
function createMutableAgentState(initialState?: AgentInitialState): MutableAgentState {
	let tools = initialState?.tools?.slice() ?? [];
	let messages = initialState?.messages?.slice() ?? [];
	const initialMessage = createInitialSystemMessage(
		initialState?.systemPrompt,
		tools.map(toToolDeclaration),
	);
	if (messages[0]?.role !== "system" && initialMessage) messages.unshift(initialMessage);
	// ...
}
```

`AgentInitialState.systemPrompt` 只是「种子」：它会被转换成一条 `SystemMessage` 放进 `messages`。之后读取 `state.systemPrompt` 时，是从转录里回放出来的（`getCurrentSystemPrompt(messages)`），而不是一个独立字段。这与第 3 章的转录原生模型完全一致。

### 4.3.2 一次运行的完整生命周期

`prompt()` 与 `continue()` 都走 `runWithLifecycle`：

```ts
private async runWithLifecycle(executor: (signal: AbortSignal) => Promise<void>): Promise<void> {
	if (this.activeRun) throw new Error("Agent is already processing.");

	const abortController = new AbortController();
	// 建立 activeRun...
	this._state.isStreaming = true;
	this._state.errorMessage = undefined;

	try {
		await executor(abortController.signal);
	} catch (error) {
		await this.handleRunFailure(error, abortController.signal.aborted);
	} finally {
		this.finishRun();
	}
}
```

`activeRun` 保证同一时刻只有一次运行；`abort()` 中止当前 controller；`waitForIdle()` 等待 `activeRun.promise`，而该 promise 只有在 `agent_end` 的所有监听器都结算后才 resolve。

事件处理在 `processEvents`：先归约内部状态（`streamingMessage`、`pendingToolCalls`、`errorMessage`），再按订阅顺序**依次 await** 监听器。也就是说，一个慢的 UI 监听器会拖慢运行——这是有意的背压设计，而不是缺陷。

### 4.3.3 `continue()` 的边界条件

`continue()` 不是「再跑一次」的万能按钮。它要求：

- 当前没有运行；
- 最后一条消息不是 `assistant`（除非队列里有 steering / follow-up，可以借机消费）。

如果最后一条是 assistant 且队列为空，它会抛出 `Cannot continue from message role: assistant`。这是为了守住 provider 的协议约束：下一轮必须以 user 或 toolResult 结尾。

## 4.4 错误即数据：三层兜底

pi 的错误哲学可以概括为：**能在数据类型里表达的，就不要用异常表达。** 具体分三层：

### 第一层：模型流

`StreamFn` 的契约要求「不要 throw」，失败要编码成流事件，最终产出 `stopReason: "error" | "aborted"` 的 `AssistantMessage`。这样循环永远能拿到一条完整的消息，继续走 `finishTurn` / `turn_end` / `agent_end` 的正常路径。

### 第二层：工具

工具执行失败有两种表达：

- 抛异常 → 被 `executePreparedToolCall` 捕获，转换成 `createErrorToolResult(message)`；
- 返回 `{ isError: true }` → 保留 `details` 与 `structuredContent`，同时让模型看到错误文本。

后者更好，因为 UI 和程序化调用方仍然能拿到结构化信息。pi 在类型注释里明确要求：**不要只在 `content` 里描述失败，要用 `isError` 标记。**

### 第三层：运行器

如果 `executor` 本身抛了未捕获异常，`handleRunFailure` 会合成一条 assistant 错误消息，并补齐完整的事件序列：

```ts
await this.processEvents({ type: "message_start", message: failureMessage });
await this.processEvents({ type: "message_end", message: failureMessage });
await this.processEvents({ type: "turn_end", message: failureMessage, toolResults: [] });
await this.processEvents({ type: "agent_end", messages: [failureMessage] });
```

这样 UI 不会卡在「流式输出中」，扩展也能收到 `agent_end` 做清理。**异常被降级成一条消息，但生命周期仍然完整。**

## 4.5 何时用 `Agent`，何时直接用循环

| 场景 | 选择 |
|---|---|
| 嵌入式 Agent、需要自己管理消息与队列 | `Agent` |
| 一次性脚本、已有事件总线 | `runAgentLoop` |
| 需要完全自定义生命周期 | 直接用 `runLoop`（需要复制或 fork） |
| 产品级会话（压缩、重试、扩展、持久化） | 不要直接用 `Agent`，用 `AgentSession` |

`Agent` 是「可用的最小内核」，不是「产品」。它不负责持久化、不负责压缩、不负责扩展——这些都在 `pi-coding-agent` 的 `AgentSession` 里。下一章我们就进入那一层。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/agent/src/agent.ts` | `Agent` 状态机、队列、事件分发、失败兜底 |
| `packages/agent/src/types.ts` | 十个钩子与结果类型 |
| `packages/agent/src/proxy.ts` | 把 `StreamFn` 代理到远端的最小例子 |
| `packages/coding-agent/src/core/agent-session.ts` | 钩子最复杂的真实使用者 |

## 动手实验

1. 用 `Agent` 起一个不联网的会话：注入一个 `streamFn`，第一次返回带 `toolCall` 的 assistant 消息，第二次返回纯文本。在 `subscribe` 里打印事件顺序，确认 `turn_end` 与 `agent_end` 的位置。
2. 在运行中调用 `steer()` 与 `followUp()` 各一次，观察它们分别在什么事件之后被消费。把 `steeringMode` 改成 `"all"` 再试一次，对比差异。
3. 写一个 `beforeToolCall`，对名字为 `bash` 的调用返回 `{ block: true, reason: "blocked by policy" }`。确认模型收到的是错误工具结果，而不是进程崩溃。
4. 故意让 `streamFn` 抛异常，观察 `handleRunFailure` 合成的事件序列是否仍然以 `agent_end` 收尾。
