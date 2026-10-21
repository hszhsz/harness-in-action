# 第 3 章　Agent 主循环：心脏如何跳动

> 一个 Agent 的「聪明」来自模型，但它的「可靠」来自循环。

上一章我们站在高处看完了 pi 的地形。从这一章开始，我们进入 `packages/agent`，看一个任务是如何被一轮轮推进的。本章的主线只有一条：**`runLoop` 如何把「生成 → 调工具 → 回灌」跑稳，并在 1.3 的转录原生模型下保持一致。**

---

## 3.1 三层 API：从事件流到可等待函数

`packages/agent/src/agent-loop.ts` 对外暴露两层，内部还有一层：

| API | 形态 | 适用场景 |
|---|---|---|
| `agentLoop(prompts, context, config, signal, streamFn)` | 返回 `EventStream<AgentEvent, AgentMessage[]>` | 需要边跑边消费事件的调用方 |
| `runAgentLoop(prompts, context, config, emit, signal, streamFn)` | `Promise<AgentMessage[]>` | 已有自己的事件分发，或写测试 |
| `runLoop(...)`（私有） | 真正的主循环 | 只被上面两个函数复用 |

`agentLoop` 只是把 `runAgentLoop` 的事件转发进一个流，并在结束时 `stream.end(messages)`：

```ts
export function agentLoop(
	prompts: AgentMessage[],
	context: AgentContext,
	config: AgentLoopConfig,
	signal: AbortSignal | undefined,
	streamFn: StreamFn,
): EventStream<AgentEvent, AgentMessage[]> {
	const stream = createAgentStream();

	void runAgentLoop(prompts, context, config, async (event) => {
		stream.push(event);
	}, signal, streamFn).then((messages) => {
		stream.end(messages);
	});

	return stream;
}
```

`runAgentLoop` 负责「开场」：先把新提示词通过 `declareToolChanges` 声明进去，发出 `agent_start`、`turn_start`，并逐条发出 `message_start` / `message_end`，然后交给 `runLoop`。`runAgentLoopContinue` 则不加新消息，直接从现有上下文继续——这正是自动重试和溢出恢复要用的入口。

## 3.2 转录原生：系统提示词为什么进了 `messages`

这是 1.3 相比旧版最大的结构变化，值得单独一节。

旧版的 `Context` 长这样（概念上）：

```ts
{ systemPrompt: string, messages: Message[], tools: Tool[] }
```

系统提示词和工具是**旁路字段**，每次请求时由调用方临时拼进去。问题在于：一旦会话被压缩、被分支、被重放，「当时用的到底是哪版提示词、哪套工具」就只能靠外部状态猜测。

1.3 把所有东西都放进转录：

```ts
export interface SystemMessage {
	role: "system";
	content: string | TextContent[];
	sections?: Record<string, string | null>;
	toolsAdded?: Tool[];
	toolsRemoved?: ToolReference[];
	timestamp: number;
}
```

- `content`：首条系统消息承载基础提示词；后续系统消息承载追加指令。
- `sections`：具名有序的提示词分区。后续消息可以按名字替换或删除分区，从而实现**增量更新**，而不是整段重发。
- `toolsAdded` / `toolsRemoved`：在这条消息处，工具集合发生了哪些变化。

`runLoop` 在每次请求前都会调用 `declareToolChanges`，把「运行时可执行的工具集合」与「转录里已经声明过的工具」做差，生成一条新的系统消息（或改写待发送的系统消息）：

```ts
const changes = getToolStateChanges(
	getCurrentTools([...context.messages, ...baseline]),
	(context.tools ?? []).map(toToolDeclaration),
);
```

这样带来三个直接收益：

1. **可重放**：只要按顺序回放转录，模型在第 N 步看到的提示词与工具集合就是确定的。
2. **可增量**：工具负载变化只发一条 small delta，不必每轮重发完整声明。
3. **可压缩**：压缩条目可以连同当时的 `SystemMessage` 一起快照（第 11 章），恢复时不丢上下文。

代价是：**任何改变模型可见状态的东西，都必须变成转录事件**。这也解释了为什么 `AgentSession` 那么重视「请求投影」——它必须保证会话树投影出的消息，就是 provider 真正收到的东西。

## 3.3 双层循环：内层干活，外层等待

`runLoop` 的结构可以浓缩成下面这段伪代码：

```ts
while (true) {                       // 外层：follow-up
  let hasMoreToolCalls = true;
  while (hasMoreToolCalls || pendingMessages.length > 0) {  // 内层：工具 + steering
    if (lastCompletedTurn) {
      prepareNextTurn?.(lastCompletedTurn);   // 压缩、刷新提示词、换模型
      pendingMessages = await getSteeringMessages?.();
      emit(turn_start);
    }
    // 1. 把 prepared/pending 消息逐条声明并写入转录
    // 2. prepareRequest?.()：最后一次改写 context/model/thinkingLevel 的机会
    // 3. streamAssistantResponse：流式生成并发出事件
    // 4. 错误/中止 → finishTurn → turn_end → agent_end，直接返回
    // 5. 有工具调用 → 执行并把 toolResult 写回转录
    // 6. finishTurn?.()：扩展/编排层决定继续还是结束
    // 7. 轮询 steering
  }
  // 内层退出：轮询 follow-up；有则继续外层
  if (followUpMessages.length > 0) { pendingMessages = followUpMessages; continue; }
  if (explicitContinuation) { explicitContinuation = false; continue; }
  break;
}
emit(agent_end);
```

两个循环的语义差异是理解 pi 调度模型的关键：

| 队列 | 何时被读取 | 语义 |
|---|---|---|
| **steering** | 每轮工具执行完、下一次请求前 | 「插话」：改变当前任务的方向，但仍在本轮任务内 |
| **follow-up** | 内层即将退出、Agent 本来要停下时 | 「追加任务」：当前任务结束后再做一件事 |
| **prepared messages** | `prepareNextTurn` 返回时 | 编排层注入的消息（如压缩通知、系统刷新），优先于队列消息 |

注意 1.3 的一个细节：`prepareNextTurn` 可能很慢（比如触发压缩），所以它返回后**会再轮询一次 steering**；而且只有先前没有待处理消息时才轮询，避免 `one-at-a-time` 模式一次投递两条。

## 3.4 一次 turn 的完整步骤

把镜头再拉近，一次「有工具调用」的 turn 依次发生：

1. **准备下一轮**（非首轮）：`prepareNextTurn` 可以替换 `context`、`messages`、`model`、`thinkingLevel`。
2. **声明待发消息**：`declareToolChanges` 计算工具增删，必要时插入系统消息；每条消息发出 `message_start` / `message_end` 并写入 `currentContext.messages`。
3. **请求前钩子**：`prepareRequest` 拿到 `{ context, model, thinkingLevel }`，可以返回替换值。与 `prepareNextTurn` 的区别是：它在**每一次** provider 请求前都会调用，包括第一轮。
4. **流式生成**：`transformContext` → `convertToLlm` → `normalizeContext` → `getApiKey` → `streamFunction`。事件被转换成 `message_start` / `message_update` / `message_end`。
5. **错误短路**：`stopReason` 为 `"error"` 或 `"aborted"` 时，调用 `finishTurn`，发 `turn_end` 与 `agent_end`，**不再执行工具**。
6. **执行工具**：抽取所有 `toolCall` 内容块，按执行模式运行；结果以 `ToolResultMessage` 逐条写回。
7. **回合决策**：`finishTurn` 返回 `{ action: "end" }` 就结束；返回 `{ action: "continue" }` 则保证下一轮至少发一次请求；不返回则按正常调度。
8. **发 `turn_end`**，轮询 steering，决定内层是否继续。

这里有一个容易忽略的细节：**`finishTurn` 在错误路径上也会被调用**。它不只是「回合结束的回调」，而是编排层最后一次参与决策的机会——`AgentSession` 正是在这里派发扩展的 `turn_end` 边界事件（第 5 章）。

## 3.5 工具执行：先预检，再并发

### 3.5.1 预检是串行的

无论并行还是串行模式，工具调用的**准备阶段**都按模型给出的顺序逐个进行（`prepareToolCall`）：

```
查找工具 → prepareArguments?.() → validateToolArguments() → beforeToolCall?.()
```

任何一步失败都会变成一个**错误工具结果**，而不是异常：

- 工具不存在 → `Tool X not found`
- 参数不合法 → 校验错误文本
- `beforeToolCall` 返回 `{ block: true }` → 阻止执行，可附带 `terminate`

这一步是 pi 的「准入闸门」。它只做机制：调用钩子、尊重结果；至于钩子里写什么策略（弹窗、白名单、审计），内核不关心。

### 3.5.2 执行可以并行

`executeToolCalls` 决定模式：

```ts
const hasSequentialToolCall = toolCalls.some(
	(tc) => currentContext.tools?.find((t) => t.name === tc.name)?.executionMode === "sequential",
);
if (config.toolExecution === "sequential" || hasSequentialToolCall) {
	return executeToolCallsSequential(...);
}
return executeToolCallsParallel(...);
```

只要**任何一个**被调用的工具声明了 `executionMode: "sequential"`，整批工具都会串行执行——这是有意为之的保守策略：一个需要独占资源的工具，不应该和同批的其他工具竞争。

并行模式的顺序保证值得记住：

- `tool_execution_start`：按模型给出的顺序发出。
- 工具的 `execute()`：并发运行。
- `tool_execution_end`：按**完成顺序**发出。
- `ToolResultMessage`：在全部完成后，按**模型给出的原始顺序**写回转录。

这样 UI 能尽早看到「某个工具完成了」，而模型看到的上下文仍然稳定有序。

### 3.5.3 `terminate`：整批一致才生效

一个工具结果可以带 `terminate: true`，表示「本批做完就停」。但循环不会因为单个工具想停就停：

```ts
function shouldTerminateToolBatch(finalizedCalls) {
	return finalizedCalls.length > 0 && finalizedCalls.every((finalized) => finalized.result.terminate === true);
}
```

必须**整批所有结果**都同意终止，才会阻止下一轮。这避免了并行工具中某个「拒绝执行」的结果误伤其他仍需完成的工作。

### 3.5.4 被截断的工具调用一律不执行

当模型因为输出 token 上限而 `stopReason: "length"` 时，流式解析器可能仍然「抢救」出一些看似合法、实际被截断的 JSON 参数。pi 的选择非常保守：

```ts
const executedToolBatch =
	message.stopReason === "length"
		? await failToolCallsFromTruncatedMessage(toolCalls, emit)
		: await executeToolCalls(...);
```

所有工具调用都被标记为错误，提示模型「响应触发了输出上限，参数可能不完整，请重新发起」。**宁可让模型重做，也不执行半截命令。**

## 3.6 停下来的四种方式

| 方式 | 触发 | 结果 |
|---|---|---|
| 没有工具调用且没有后续消息 | 内层条件不满足，外层也没有 follow-up | 正常结束，发 `agent_end` |
| `stopReason: "error"` / `"aborted"` | 模型请求失败或用户中止 | 硬退出，不执行工具，发 `turn_end` + `agent_end` |
| `finishTurn` 返回 `{ action: "end" }` | 编排层主动结束 | 立即发 `agent_end` |
| `terminate: true` 且整批一致 | 工具要求停止 | 内层不再因工具继续；是否还有 follow-up 由外层决定 |

注意 `stopReason: "length"` 不是硬退出：它会让本批工具失败，然后模型有机会在下一轮重新发起。这是「可恢复的长度截断」与「不可恢复的请求错误」的区别。

## 3.7 嵌套工具调用：同一套管线，不同的入口

1.3 新增了 `runToolCall()`（`agent-loop.ts:811`），允许工具在**自己的执行过程中**调用另一个工具，而不经过模型。它复用同一套 `prepareToolCall` / `beforeToolCall` / `afterToolCall` / 结果规范化逻辑，只是由调用方直接指定工具和参数。

这为 codemode、子 Agent、组合工具打开了门：工具链可以嵌套，但**钩子与准入规则不会因为绕过模型而失效**。第 15 章会看到 codemode 如何用它把脚本里的调用伪装成 `parentToolCallId` 关联的嵌套调用。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/agent/src/agent-loop.ts` | 主循环、工具执行、流式消息处理 |
| `packages/agent/src/types.ts` | `AgentLoopConfig`、`AgentTool`、事件类型 |
| `packages/agent/src/agent.ts` | 有状态的 `Agent` 包装（下一章） |
| `packages/ai/src/utils/transcript.ts` | `normalizeContext`、`getCurrentTools`、`getCurrentSystemMessage` |
| `packages/ai/src/types.ts` | `SystemMessage`、`TranscriptContext`、`AssistantMessageEvent` |

## 动手实验

1. 在 `packages/agent/src/agent-loop.ts` 里找到 `declareToolChanges`，构造一个最小场景：初始工具为 `[read]`，第二轮变成 `[read, bash]`。手工推导会插入几条系统消息、每条消息的 `toolsAdded` / `toolsRemoved` 是什么。
2. 写一个假的 `StreamFn`，返回一个带两个 `toolCall` 的 `AssistantMessage`，其中一个工具声明 `executionMode: "sequential"`。用 `runAgentLoop` 跑一遍，打印事件顺序，验证「预检串行、执行串行、结果按源顺序」。
3. 把 `stopReason` 改成 `"length"` 再跑一次，观察工具是否被执行、模型收到什么错误文本。思考：为什么这里不选择「尽力执行未截断的工具」？
