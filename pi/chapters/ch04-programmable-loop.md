# 第 4 章　可编程的循环：钩子、队列与三种消息

> 一个好的内核，应该像一颗主板：自己不决定任何业务，但每一个引脚都能插上不同的策略。

第 3 章我们看清了 `runLoop` 的机械结构——双层循环、turn 的五个步骤、串并行与停止语义。但你可能注意到一件事：`runLoop` 自己几乎不做任何「业务决策」。要不要换模型？要不要拦截某个工具？用户中途插的话怎么排队？这些它统统不管，而是委托给 `AgentLoopConfig` 上的一组钩子，以及上层对消息队列的编排。

这一章，我们就来拆这套「可编程性」——它是 pi「机制与策略分离」哲学最直接的体现。

---

## 4.1 `AgentLoopConfig` 的九个钩子：机制与策略的分界线

打开 `packages/agent/src/types.ts` 里的 `AgentLoopConfig`，你会看到一组可选回调。把它们集中看一遍，你就握住了「如何编程这个循环」的全部把手：

| 钩子 | 触发时机 | 典型用途 |
|---|---|---|
| `convertToLlm`（必填） | 每次请求前 | `AgentMessage[]` → provider `Message[]`，过滤掉 UI-only 消息 |
| `transformContext` | `convertToLlm` 之前 | 上下文裁剪、外部上下文注入 |
| `getApiKey` | 每轮请求前 | 解析会过期的短期 token（如 OAuth） |
| `beforeToolCall` | 工具执行前 | **权限审批、危险操作拦截** |
| `afterToolCall` | 工具执行后 | 结果加工、覆盖、注入终止信号 |
| `prepareNextTurn` | 每个 turn 结束后 | 动态换模型、调推理强度、替换上下文 |
| `shouldStopAfterTurn` | 每个 turn 结束后 | 宿主主动叫停（如上下文快满了） |
| `getSteeringMessages` | 每轮迭代前 | 拉取用户中途插入的话 |
| `getFollowUpMessages` | 内层循环退出后 | 拉取后续消息以开新一轮 |

这九个钩子有一个共同的契约，写在每个钩子的注释里：**「must not throw or reject」**——绝不能抛异常或返回 rejected promise，出错时要返回一个安全的回退值。原因很硬核：这些钩子被 `runLoop` 在精心编排的事件序列中调用，一旦抛异常，就会**绕过正常的事件序列**直接中断底层循环，让 UI 和会话状态陷入不一致。

> **设计要点**：`runLoop` 是「机制」（mechanism），这九个钩子是「策略」（policy）的注入点。机制稳定、策略灵活——这正是一个能长期演进的 Agent 内核应有的样子。你想让 pi「胆小」，就在 `beforeToolCall` 里加审批；想让它「省钱」，就在 `prepareNextTurn` 里把贵模型换成便宜模型。内核一行都不用改。

我们重点看 `prepareNextTurn` 在 `runLoop` 里的真实用法（第 3 章见过它，这里展开）：

```ts
const nextTurnSnapshot = await config.prepareNextTurn?.(nextTurnContext);
if (nextTurnSnapshot) {
	currentContext = nextTurnSnapshot.context ?? currentContext;
	config = {
		...config,
		model: nextTurnSnapshot.model ?? config.model,
		reasoning:
			nextTurnSnapshot.thinkingLevel === undefined
				? config.reasoning
				: nextTurnSnapshot.thinkingLevel === "off"
					? undefined
					: nextTurnSnapshot.thinkingLevel,
	};
}
```

注意 `config` 是用 `{ ...config, ... }` 重新赋值的——也就是说，**模型、推理强度、上下文都能在两个 turn 之间被热替换**，而循环本身浑然不觉。这是「动态降智/升智」「中途换模型」这类高级编排能力的底层支点。

## 4.2 三种消息队列：steering / follow-up / next-turn 的语义差异

pi 对「用户在 Agent 干活时插进来的消息」做了非常细致的区分。这不是过度设计——任何做过交互式 Agent 的人都知道，「什么时候让用户的话生效」是体验的关键。

三种消息，三种时机：

| 消息类型 | 注入时机 | 语义 |
|---|---|---|
| **steering**（操舵） | 当前 turn 的工具执行完后、下一次模型生成前 | 「等等，调整一下方向」——尽快生效 |
| **follow-up**（后续） | Agent 本要停下时 | 「等你忙完这个，再做这个」——排队等候 |
| **next-turn**（下一轮） | 由上层在 idle 时注入会话 | 「下次启动时带上这条」——延迟到下次 |

在 `runLoop` 里，steering 和 follow-up 的位置泾渭分明（回顾第 3 章的骨架）：steering 在内层循环每轮开头通过 `getSteeringMessages` 拉取并立即注入；follow-up 在内层循环**退出之后**通过 `getFollowUpMessages` 拉取，有就重新进内层。next-turn 则不属于 `runLoop` 的职责，而是 `AgentHarness` 在 idle 阶段把消息直接 append 进会话（第 5 章细说）。

队列还有一个「排空模式」（`QueueMode`），定义在 `agent.ts` 的 `PendingMessageQueue` 里：

```ts
drain(): AgentMessage[] {
	if (this.mode === "all") {
		const drained = this.messages.slice();
		this.messages = [];
		return drained;
	}
	// "one-at-a-time"：只取最老的一条
	const first = this.messages[0];
	if (!first) return [];
	this.messages = this.messages.slice(1);
	return [first];
}
```

- `"all"`：到了排空点，把队列里所有消息一次性注入。
- `"one-at-a-time"`：每次只注入最老的一条，其余留到下一个排空点。

`Agent` 类的默认值是 `"one-at-a-time"`（构造函数里 `options.steeringMode ?? "one-at-a-time"`）。为什么默认逐条？因为它让用户的多条插话能被模型逐一消化，而不是一股脑塞进去造成混乱。

## 4.3 `beforeToolCall` / `afterToolCall`：在动手前后插一脚

这两个钩子值得单独拎出来，因为它们是 pi「约束外包」哲学的技术落点——**pi 核心不内置权限系统，但它在工具执行前后各留了一个口子，让上层把权限策略插进来。**

回顾第 3 章 `prepareToolCall` 里 `beforeToolCall` 的位置：参数校验通过之后、真正执行之前。它的返回值能直接「掐断」执行：

```ts
if (beforeResult?.block) {
	return {
		kind: "immediate",
		result: createErrorToolResult(beforeResult.reason || "Tool execution was blocked"),
		isError: true,
	};
}
```

注意拦截的结果不是抛异常，而是**合成一条「带错误内容的工具结果」喂回给模型**——模型会看到「这个操作被拦截了，原因是 XXX」，于是它可能换个做法。这又一次印证了「错误即数据」。

`afterToolCall` 则在工具执行后、结果落地前给一次「修正机会」（`finalizeExecutedToolCall`）。它的合并语义是「字段级覆盖、不深合并」：

```ts
result = {
	content: afterResult.content ?? result.content,
	details: afterResult.details ?? result.details,
	terminate: afterResult.terminate ?? result.terminate,
};
isError = afterResult.isError ?? isError;
```

典型用途：脱敏工具输出、给结果加注释、或者在某个工具成功后设 `terminate: true` 让循环停下。

## 4.4 `Agent` 类与 `prompt` 队列模式：把循环包成对象

`agent-loop.ts` 是函数式的、无状态的。但真实应用需要一个有状态的对象来「拥有」当前会话、管理队列、广播事件。这就是 `packages/agent/src/agent.ts` 里的 `Agent` 类（557 行）。

它做的事可以归纳为三类：

**1. 拥有状态。** `Agent` 持有 `_state`（系统提示词、模型、工具、消息、`isStreaming` 等）。这里有一个值得学的小设计——`tools` 和 `messages` 用 getter/setter 实现，赋值时自动 `.slice()` 复制顶层数组：

```ts
set messages(nextMessages: AgentMessage[]) {
	messages = nextMessages.slice();
}
```

这避免了「外部传进来一个数组，之后又在外面改它，结果污染了 Agent 内部状态」的经典 bug。

**2. 管理队列与单飞约束。** `Agent` 内部有两个 `PendingMessageQueue`（steering 和 follow-up），并通过 `steer()` / `followUp()` 暴露入队接口。它还强制「同一时刻只能有一个 run」：

```ts
async prompt(...): Promise<void> {
	if (this.activeRun) {
		throw new Error(
			"Agent is already processing a prompt. Use steer() or followUp() to queue messages, or wait for completion.",
		);
	}
	// ...
}
```

这条约束很关键：它把「正在跑的时候用户又发了消息」这个并发问题，强制收敛成「要么入队、要么等待」，而不是允许两个循环同时跑。

**3. 翻译事件、广播给监听者。** `processEvents` 把底层 `AgentEvent` 翻译成对内部状态的更新（比如 `tool_execution_start` 时把 toolCallId 加进 `pendingToolCalls`），然后**按订阅顺序 await 每个监听者**。注释特别强调：`agent_end` 是最后一个事件，但 Agent 要等到所有 `agent_end` 监听者都 settle 之后才真正变回 idle。这种「监听者也是 run 结算的一部分」的设计，保证了 UI 渲染、会话落盘这些副作用都完成后，才允许下一个 prompt 进来。

## 4.5 错误即数据：失败为什么不抛异常而编码进流

最后收束一个贯穿前几章的主题。pi 在好几个层次上都坚持「失败不抛异常，而是编码成数据」：

- **模型请求失败** → `StreamFn` 契约要求把失败编码进流，最终落成 `stopReason: "error"` 的 `AssistantMessage`（第 2 章）。
- **工具执行失败** → 被 `createErrorToolResult` 包成正常工具结果喂回（第 3 章）。
- **`beforeToolCall` 拦截** → 同样包成错误工具结果（本章 4.3）。
- **整个 run 失败兜底** → `Agent.handleRunFailure` 合成一条 `stopReason` 为 `"error"`/`"aborted"` 的失败消息，并依次 emit `message_start` / `message_end` / `turn_end` / `agent_end`：

```ts
private async handleRunFailure(error: unknown, aborted: boolean): Promise<void> {
	const failureMessage = { /* ... stopReason: aborted ? "aborted" : "error" ... */ };
	await this.processEvents({ type: "message_start", message: failureMessage });
	await this.processEvents({ type: "message_end", message: failureMessage });
	await this.processEvents({ type: "turn_end", message: failureMessage, toolResults: [] });
	await this.processEvents({ type: "agent_end", messages: [failureMessage] });
}
```

为什么如此执着？因为对一个交互式 Agent 来说，**异常冒泡 = 事件序列断裂 = UI 卡死、会话不一致**。把失败建模成「一条正常的、只是 stopReason 不同的消息」，意味着所有下游消费者（UI、会话、扩展）都能用同一套代码路径处理成功和失败——它们甚至不需要写 try/catch。这是一个对「可观测性」极其友好的决策。

---

## 本章小结

- `AgentLoopConfig` 的九个钩子是「策略」的注入点，共同契约是「绝不抛异常」；`prepareNextTurn` 能在 turn 之间热替换模型/上下文/推理强度。
- 三种消息队列对应三种生效时机：steering（尽快）、follow-up（忙完再做）、next-turn（下次启动）；排空模式有 `"all"` 和 `"one-at-a-time"`，默认逐条。
- `beforeToolCall` / `afterToolCall` 是「约束外包」的技术落点：前者能拦截执行、后者能修正结果，且都以「错误即数据」的方式工作。
- `Agent` 类给无状态的循环套上「拥有状态、管理队列、单飞约束、广播事件」的外壳；getter/setter 复制数组、`activeRun` 强制串行是两处值得借鉴的细节。
- pi 在四个层次坚持「失败编码成数据而非异常」，换来一个永不断裂的事件序列和极简的下游处理逻辑。

下一章，我们再上升一层，看 `AgentHarness` 如何把这个「可编程的循环」编排成「会话、阶段、压缩、Skill」俱全的完整任务流程。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/agent/src/types.ts` | `AgentLoopConfig` 九钩子、`QueueMode`、`AfterToolCallResult` 等契约 |
| `packages/agent/src/agent.ts` | `Agent` 类、`PendingMessageQueue`、`handleRunFailure` 兜底 |
| `packages/agent/src/agent-loop.ts` | 钩子的真实调用点（`prepareToolCall`、`finalizeExecutedToolCall`） |

## 动手实验

1. 给一个 `Agent` 实例注册一个 `beforeToolCall`，对所有 `bash` 工具调用返回 `{ block: true, reason: "演示拦截" }`。观察模型收到错误结果后的反应——它会道歉、换工具、还是放弃？这就是「约束外包」最小可用的样子。
2. 把 `Agent` 的 `steeringMode` 分别设为 `"all"` 和 `"one-at-a-time"`，在一次长任务里连续 `steer()` 三条消息，对比两种模式下消息进入上下文的节奏差异。
3. 阅读 `prepareNextTurn` 的合并逻辑，思考：如果你想实现「前 3 个 turn 用便宜模型，第 4 个 turn 起换贵模型」，你会在这个钩子里读什么状态来做判断？（提示：钩子的 context 里有 `newMessages`。）
