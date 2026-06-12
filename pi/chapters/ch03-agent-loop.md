# 第 3 章　Agent 主循环：心脏是如何跳动的

> 如果整本书只能读一章，读这一章。
> 因为一个 Agent 之所以是 Agent——而不是一次性的 Chat 调用——全部秘密都藏在这个循环里。

第 2 章我们把请求一路送到了 `pi-agent-core` 的门口。现在推门进去，看那颗心脏第一次跳动。本章精读 `packages/agent/src/agent-loop.ts`（742 行）。读完你将彻底理解一件事：**所谓「Agent 自主完成任务」，在代码层面，不过是一个被精心设计的 `while (true)` 循环，反复做着「生成 → 调工具 → 把结果喂回去」这三件事，直到没有更多工具要调。**

---

## 3.1 `packages/agent` 为什么要独立成可复用包

在看循环之前，先理解第 2 章那条接缝在这里如何落地。`agent-loop.ts` 文件顶部的注释开门见山：

```ts
/**
 * Agent loop that works with AgentMessage throughout.
 * Transforms to Message[] only at the LLM call boundary.
 */
```

这句话点出了内核的一个关键设计：**循环内部自始至终操作的是 `AgentMessage`（pi 自己的消息抽象），只有在真正要调模型的那一瞬间，才把它转换成 LLM 能懂的 `Message[]`。** 这层「内部用自己的类型、边界处才转换」的设计，让内核可以承载各种「LLM 看不到的消息」（UI 通知、自定义状态等）而不污染模型上下文——具体怎么转，由调用方传入的 `convertToLlm` 决定（第 4 章细说）。

正因为内核只认 `AgentMessage` 和 `StreamFn` 这两个抽象，它才能被 CLI、Web 服务、测试 harness 共享。这就是它独立成包的根本原因。

## 3.2 三层 API：从便利到底层

`agent-loop.ts` 对外暴露了一组层次分明的 API，理解它们的关系能帮你快速定位代码：

- **`agentLoop(prompts, ...)`** / **`agentLoopContinue(context, ...)`**：最外层的便利封装。它们返回一个 `EventStream`，内部 `void` 调用下一层并把事件 push 进流、把最终结果 `end` 进流。调用方只需 `for await` 这个流。
- **`runAgentLoop(...)`** / **`runAgentLoopContinue(...)`**：中间层。不返回流，而是接受一个 `emit` 回调（`AgentEventSink`），让调用方自己拥有事件处理方式。
- **`runLoop(...)`**：私有的核心实现，被上面两者共享。**真正的 `while (true)` 在这里。**

`agentLoop` 和 `agentLoopContinue` 的区别很关键，体现了 Agent 的两种启动姿态：

- **`agentLoop`**：用一条**新 prompt** 启动。它把 prompt 加进上下文，并为之 emit 消息事件。这是「用户发来一句新话」的场景。
- **`agentLoopContinue`**：**不加新消息**，直接从当前上下文继续。代码里有一道严格校验：

```ts
if (context.messages[context.messages.length - 1].role === "assistant") {
	throw new Error("Cannot continue from message role: assistant");
}
```

注释解释得很清楚——因为继续运行时，上下文最后一条消息必须能转换成 `user` 或 `toolResult`，否则 LLM provider 会拒绝请求。这是「重试」或「工具结果已就位、让模型接着说」的场景。

> 这种「新起 vs 继续」的二分，是 Agent 状态机的基础。很多 Agent 框架的 bug 就出在没分清这两者，导致重试时重复注入消息或丢失上下文。

## 3.3 双层循环：工具驱动的内层 vs 消息驱动的外层

`runLoop` 的核心是两层嵌套的 `while`。把无关细节去掉，骨架长这样：

```ts
let pendingMessages = (await config.getSteeringMessages?.()) || [];

// 外层：当 agent 本要停下，却又来了 follow-up 消息时，继续
while (true) {
	let hasMoreToolCalls = true;

	// 内层：只要还有工具要调，或还有待注入的消息，就继续
	while (hasMoreToolCalls || pendingMessages.length > 0) {
		// 1. 注入待处理消息（steering）
		// 2. 流式生成助手响应
		// 3. 抽取并执行工具调用
		// 4. 把工具结果 push 回上下文
		// 5. 判断是否该停
		pendingMessages = (await config.getSteeringMessages?.()) || [];
	}

	// 内层退出后，检查是否有 follow-up 消息
	const followUpMessages = (await config.getFollowUpMessages?.()) || [];
	if (followUpMessages.length > 0) {
		pendingMessages = followUpMessages;
		continue;  // 回到内层再跑一轮
	}
	break;  // 真的没事干了，退出
}
```

**为什么需要两层？** 因为「Agent 该不该继续」有两种不同的连续性来源：

- **内层处理「工具驱动的连续性」**：模型这一轮调了工具 → 工具有结果 → 结果要喂回去让模型接着想 → 模型可能又调工具……这个链条只要不断，内层就一直转（`hasMoreToolCalls`）。
- **外层处理「消息驱动的连续性」**：当模型不再调工具、内层本该退出时，外层给一次「最后的机会」——看看队列里有没有用户排队等待的后续消息（follow-up）。有就拉进来，重新进内层。

这两层之间还夹着第三种消息：**steering**（中途插话）。它在内层每一轮的开头被拉取并注入——也就是说，用户可以在 Agent 干活的间隙插一句「等等，换个方向」，这句话会在下一次模型生成之前进入上下文。三种消息队列的语义差异，我们在第 4 章专门拆。

## 3.4 一轮 turn 究竟做了什么

放大内层循环的一轮（一个 turn），看那五件事的真实代码：

**第一步：注入待处理消息。** 把 steering 消息 push 进上下文，并 emit 生命周期事件：

```ts
if (pendingMessages.length > 0) {
	for (const message of pendingMessages) {
		await emit({ type: "message_start", message });
		await emit({ type: "message_end", message });
		currentContext.messages.push(message);
		newMessages.push(message);
	}
	pendingMessages = [];
}
```

**第二步：流式生成助手响应。** 这是 `streamAssistantResponse`（3.5 节细看），它把上下文转成 LLM 消息、调模型、把流式事件实时回放成 `message_update`，最终收敛成一条完整的 `AssistantMessage`。如果这条消息的 `stopReason` 是 `"error"` 或 `"aborted"`，循环立刻收尾退出：

```ts
if (message.stopReason === "error" || message.stopReason === "aborted") {
	await emit({ type: "turn_end", message, toolResults: [] });
	await emit({ type: "agent_end", messages: newMessages });
	return;
}
```

**第三步：抽取并执行工具调用。**

```ts
const toolCalls = message.content.filter((c) => c.type === "toolCall");
hasMoreToolCalls = false;
if (toolCalls.length > 0) {
	const executedToolBatch = await executeToolCalls(currentContext, message, config, signal, emit);
	toolResults.push(...executedToolBatch.messages);
	hasMoreToolCalls = !executedToolBatch.terminate;
	// ...
}
```

**第四步——也是整个循环的灵魂——把工具结果 push 回上下文：**

```ts
for (const result of toolResults) {
	currentContext.messages.push(result);
	newMessages.push(result);
}
```

> **这一行就是 Agent 与 Chat 的分水岭。** 如果删掉它，模型永远看不到自己工具调用的结果，Agent 就退化成一个「调一次工具就失忆」的玩具。正是「把结果喂回去」这个动作，让模型能在下一轮看到工具产出、据此决定下一步——自主性由此而生。

**第五步：turn 收尾与「下一轮该怎么走」的决策。** emit `turn_end` 后，循环依次调用三个可选钩子：`prepareNextTurn`（可换模型/换上下文/换推理强度）、`shouldStopAfterTurn`（宿主主动叫停）、`getSteeringMessages`（拉取新的中途插话）。这三个钩子是把循环「编程化」的把手，第 4 章详述。

## 3.5 串行还是并行：由工具自己声明 `executionMode`

一条助手消息可能一次性请求好几个工具调用。`executeToolCalls` 决定它们是并行还是串行执行，而**决定权交给了工具自己**：

```ts
const hasSequentialToolCall = toolCalls.some(
	(tc) => currentContext.tools?.find((t) => t.name === tc.name)?.executionMode === "sequential",
);
if (config.toolExecution === "sequential" || hasSequentialToolCall) {
	return executeToolCallsSequential(...);
}
return executeToolCallsParallel(...);
```

逻辑很清楚：**只要这一批里有任何一个工具声明了自己是 `"sequential"`，或者全局配置要求串行，整批就走串行；否则走并行。** 默认是并行（见 `types.ts` 里 `toolExecution` 的默认值 `"parallel"`）。

这是个很务实的安全网：读文件、grep 这类只读工具可以放心并发；而像 `edit`/`write` 这类会改文件、有副作用的工具，可以声明 `executionMode: "sequential"` 来避免并发写冲突（第 9 章会看到 `file-mutation-queue` 如何进一步串行化写操作）。

无论串行还是并行，一次工具调用都经历同样的生命周期：**prepare（找工具、修参数、校验、过 `beforeToolCall` 闸门）→ execute（真正执行，流式 update）→ finalize（过 `afterToolCall` 修正）**。注意并行模式下一个微妙的顺序设计：`tool_execution_end` 事件按工具**完成顺序**发出，但最终的 tool-result 消息按助手消息里的**原始顺序**排列——这样既能让 UI 实时反映「谁先跑完」，又能保证喂回模型的消息顺序是确定的。

## 3.6 `terminate` 与 `stopReason`：循环如何停下来

一个能自主推进的循环，必须有明确的「刹车」语义。pi 有两套，分工不同：

**`stopReason`——模型侧的停止。** 由模型响应决定。`"error"` / `"aborted"` 会让循环立刻收尾（见 3.4 第二步）；正常结束则看有没有工具调用决定是否继续。

**`terminate`——工具侧的停止。** 工具可以在结果里设置 `terminate: true`，主动请求「这批工具跑完后就停下整个循环」。但 pi 对此有一个严格的「全体一致」规则：

```ts
function shouldTerminateToolBatch(finalizedCalls: FinalizedToolCallOutcome[]): boolean {
	return finalizedCalls.length > 0 && finalizedCalls.every((finalized) => finalized.result.terminate === true);
}
```

也就是说，**只有当这一批里每一个工具结果都设了 `terminate: true`，整批才会终止循环**；只要有一个工具还想让模型继续，循环就继续。这避免了「一个工具擅自叫停、其他工具的产出被模型忽略」的问题。回到 3.4 第三步那行 `hasMoreToolCalls = !executedToolBatch.terminate`——`terminate` 为真时，`hasMoreToolCalls` 变假，内层循环条件不再满足，自然停下。

还有一个常被忽略的细节：**错误不是停止信号**。看 `prepareToolCall`/`executePreparedToolCall`，工具找不到、参数校验失败、执行抛异常，统统被包装成一条「带错误内容的正常工具结果」（`createErrorToolResult`），喂回给模型——模型看到错误后往往能自我纠正。这正是第 2 章「错误即数据」红线在工具层的体现。

---

## 本章小结

- `agent-loop.ts` 内部自始至终操作 `AgentMessage`，只在调模型的边界用 `convertToLlm` 转成 `Message[]`；这让内核能承载「LLM 看不到的消息」。
- 对外提供三层 API：便利的 `agentLoop`/`agentLoopContinue`、回调式的 `runAgentLoop*`、核心私有的 `runLoop`；「新起」与「继续」是 Agent 状态机的两种基本姿态。
- `runLoop` 的双层循环分别处理「工具驱动的连续性」（内层）与「消息驱动的连续性」（外层），中间还穿插 steering（中途插话）。
- 一轮 turn 做五件事，灵魂是「把工具结果 push 回上下文」——这一行是 Agent 与 Chat 的分水岭。
- 工具批的串/并行由工具自己用 `executionMode` 声明，「有一个要串行就整批串行」；并行模式下 end 事件按完成序、消息按源序。
- 两套停止语义：模型侧 `stopReason`、工具侧 `terminate`（且需全体一致）；错误不停止循环，而是被建模成工具结果喂回。

下一章，我们把这个循环「编程化」——看 `AgentLoopConfig` 的九个钩子和三种消息队列，是如何让同一套机制服务于千差万别的策略的。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/agent/src/agent-loop.ts` | 本章主角：`runLoop` 双层循环、`streamAssistantResponse`、`executeToolCalls*` |
| `packages/agent/src/types.ts` | `AgentMessage` / `AgentTool` / `AgentEvent` 等核心类型 |
| `packages/agent/src/agent.ts` | 把循环包成对象的 `Agent` 类（第 4 章细看） |

## 动手实验

1. 在 `runLoop` 里找到 `currentContext.messages.push(result)` 那一行，用一句话向同事解释：「为什么把这一行删掉，Agent 就退化成了一个只会调一次工具的玩具？」
2. 阅读 `executeToolCallsParallel`，找出它如何保证「`tool_execution_end` 按完成顺序、tool-result 消息按源顺序」。提示：看 `finalizedCalls` 数组里塞的是「值」还是「函数」，以及 `Promise.all` 之后那个二次循环。
3. 写一个最小的假 `StreamFn`：第一次调用返回一条带 `toolCall` 的消息，第二次返回一条纯文本消息。用它驱动 `runAgentLoop`，观察事件序列，验证「双层循环」如何在两轮后自然停下。
