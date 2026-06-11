# 第 4 章　Agent 主循环：心脏是如何跳动的

> 如果整本书只能读一章，读这一章。
> 因为一个 Agent 之所以是 Agent——而不是一次性的 Chat 调用——全部秘密都藏在这个循环里。

第 3 章我们把请求一路送到了 `packages/agent-core` 门口。现在我们推门进去，看那颗心脏第一次跳动。本章会精读 `agent-core` 这个包，尤其是它 851 行的 `agent-loop.ts`。读完你将彻底理解一件事：**所谓"Agent 自主完成任务"，在代码层面，不过是一个被精心设计的 `while(true)` 循环，反复做着"生成 → 调工具 → 把结果喂回去"这三件事，直到没有更多工具要调。**

---

## 4.1 为什么 agent-core 要独立成包

在看循环之前，先理解一个架构决策：OpenClaw 为什么要把 Agent 核心抽成一个独立的 `packages/agent-core`？

第 3 章我们已经见过它的接缝——`runtime-deps.ts` 只有 45 行，定义了一个极简契约：

```ts
/** Runtime functions injected by host packages so agent-core stays provider-agnostic. */
export interface AgentCoreRuntimeDeps {
  /** Streaming completion implementation used for normal agent turns. */
  streamSimple: StreamFn;
  /** Non-streaming completion implementation used by summarization helpers. */
  completeSimple: CompleteSimpleFn;
}
```

注释里那句 **"so agent-core stays provider-agnostic"**（让 agent-core 保持与 provider 无关）是整个包的设计灵魂。`agent-core` 不知道 Anthropic、不知道 OpenAI，它只知道：有人会给我一个 `streamSimple` 函数，我把模型、上下文、选项传进去，它就会吐回一个事件流。

它甚至把"解析依赖"这件事做得很谨慎——`resolveAgentCoreStreamFn` 允许调用方传入一个显式的 `streamFn` 覆盖注入的 runtime，找不到就抛出一个**信息明确的错误**："runtime dependency 'streamSimple' is not configured."

这种独立性带来三个好处，每一个对 Agent 开发者都很重要：

| 好处 | 含义 |
|---|---|
| **可复用** | 同一套循环可被 CLI、Web 服务、测试 harness 复用，宿主各自注入不同的模型实现 |
| **可测试** | 测试时注入一个假的 `streamSimple`（返回预设事件流），就能在不联网、不烧 token 的情况下测整个循环逻辑 |
| **边界清晰** | 循环逻辑的演进与模型层的演进互不干扰，呼应第 2 章"核心精简"红线 |

> **给 Agent 开发者的启示**：把"Agent 的控制流"和"模型怎么调"分到两个包里，是让 Agent 工程可维护的第一步。如果你的循环逻辑里直接 `import OpenAI from "openai"`，那你已经把两件本该正交的事耦死了。

---

## 4.2 三层 API：从便利到底层

`agent-loop.ts` 对外暴露了一组层次分明的 API，理解它们的关系能帮你快速定位代码：

- **`agentLoop(prompts, ...)`** / **`agentLoopContinue(context, ...)`**：最外层的便利封装。它们返回一个 `EventStream`，内部 `void` 调用下一层并把事件 push 进流、把最终结果 `end` 进流。调用方只需 `for await` 这个流。
- **`runAgentLoop(...)`** / **`runAgentLoopContinue(...)`**：中间层。不返回流，而是接受一个 `emit` 回调（`AgentEventSink`），让调用方自己拥有事件的处理方式。
- **`runLoop(...)`**：私有的核心实现，被上面两者共享。**真正的 `while(true)` 在这里。**

`agentLoop` 和 `agentLoopContinue` 的区别很关键，体现了 Agent 的两种启动姿态：

- **`agentLoop`**：用一条**新 prompt** 启动。它会把 prompt 加进上下文，并为之 emit 消息事件。这是"用户发来一句新话"的场景。
- **`agentLoopContinue`**：**不加新消息**，直接从当前上下文继续。代码里有一道严格校验——上下文最后一条消息不能是 `assistant` 角色，否则抛错。注释解释得很清楚：因为 LLM provider 要求最后一条消息必须能转换成 `user` 或 `toolResult`，否则会拒绝请求。这是"重试"或"工具结果已就位、让模型接着说"的场景。

> 这种"新起 vs 继续"的二分，是 Agent 状态机的基础。很多 Agent 框架的 bug 就出在没分清这两者，导致重试时重复注入消息或丢失上下文。

---

## 4.3 心脏解剖：runLoop 的双层循环

现在进入本书技术含量最高的一段代码。`runLoop` 是一个**双层嵌套循环**，初看费解，但一旦理解它的设计意图，会觉得无比精巧。先看骨架：

```ts
async function runLoop(...) {
  let pendingMessages = (await config.getSteeringMessages?.()) || [];

  // 外层循环：当一轮本该结束、却又来了 follow-up 消息时，继续
  while (true) {
    let hasMoreToolCalls = true;

    // 内层循环：处理工具调用与 steering 消息
    while (hasMoreToolCalls || pendingMessages.length > 0) {
      // 1. 注入 pending 消息
      // 2. 流式生成 assistant 回复
      // 3. 抽取并执行 tool calls
      // 4. 把 tool 结果喂回上下文
      // 5. 询问是否该停
      // 6. 重新拉取 steering 消息
    }

    // 内层退出后，看有没有 follow-up 消息要起新一轮
    const followUpMessages = (await config.getFollowUpMessages?.()) || [];
    if (followUpMessages.length > 0) { pendingMessages = followUpMessages; continue; }
    break;
  }
}
```

### 为什么需要两层循环？

这是设计的精髓。两层循环对应两种"任务还没完"的情形：

- **内层循环**回答的是："模型这一轮调了工具，所以还得再转一圈把工具结果给它。" 只要 `hasMoreToolCalls` 为真，或者用户在等待时插了话（`pendingMessages`），内层就继续转。
- **外层循环**回答的是："Agent 本来已经打算收工了，但这时又来了一条新的后续消息（follow-up），不该让它白白结束。" 此时把 follow-up 当作新的 pending，`continue` 回到内层，重新开一轮。

这两层分别处理了 Agent 的两种"连续性"：**工具驱动的连续性**（内层）和**消息驱动的连续性**（外层）。把它们分开，事件顺序才能保持干净。

### 一轮内层循环到底做了什么

我们逐步拆解内层循环体（对应上面骨架的 6 步）：

**第 1 步——注入 pending 消息（Steering）。** 如果队列里有"用户在模型生成时插入的话"（steering message），先把它们 emit 出去并推入上下文：

```ts
if (pendingMessages.length > 0) {
  for (const message of pendingMessages) {
    await emit({ type: "message_start", message });
    await emit({ type: "message_end", message });
    currentContext.messages.push(message);
    newMessages.push(message);
  }
}
```

**Steering（操舵）是一个高级但极其实用的能力**：用户不必等 Agent 干完一整件事，可以在它执行过程中随时插话纠偏。把这些消息在"下一次 assistant 回复之前"注入，模型就能立刻看到新指令。

**第 2 步——流式生成 assistant 回复。** 调用 `streamAssistantResponse`（下一节细讲），拿到模型这一轮的完整输出 `message`，推入 `newMessages`。

**第 3 步——错误/中止的早退。** 如果这条消息的 `stopReason` 是 `error` 或 `aborted`，立刻 emit `turn_end` 与 `agent_end` 并 `return`。循环不会在出错时空转。

**第 4 步——抽取并执行工具调用。** 这是"Agent 能做事"的关键一行：

```ts
const toolCalls = message.content.filter((c) => c.type === "toolCall");
```

模型的输出是一个 `content` 数组，里面混着文本、思考（thinking）、以及工具调用（toolCall）。这里把 `toolCall` 过滤出来。如果有，就交给 `executeToolCalls`（4.5 节），拿回一批 `ToolResultMessage`，并据此更新 `hasMoreToolCalls`：

```ts
const executedToolBatch = await executeToolCalls(currentContext, message, config, signal, emit);
toolResults.push(...executedToolBatch.messages);
hasMoreToolCalls = !executedToolBatch.terminate;
for (const result of toolResults) {
  currentContext.messages.push(result);   // 关键：工具结果被推回上下文
  newMessages.push(result);
}
```

**注意 `currentContext.messages.push(result)` 这一行——这就是"把工具结果喂回模型"的物理动作。** 下一轮 `streamAssistantResponse` 时，模型就会在上下文里看到这些工具结果，从而决定下一步。整个 Agent 的"自主性"就源于这个回灌动作。

**第 5 步——回合钩子与是否停止。** emit `turn_end` 后，循环给宿主两个干预点：

- `config.prepareNextTurn?.(...)`：宿主可以在下一轮前**换模型、调整推理强度（thinkingLevel）、甚至替换整个上下文**。这是动态模型切换、上下文压缩注入的接口。
- `config.shouldStopAfterTurn?.(...)`：宿主可以判断"虽然还有工具结果，但该停了"，主动结束。

**第 6 步——重新拉取 steering 消息**，进入下一次内层迭代。

### 终止语义：terminate 与 stopReason

这里有个容易混淆的点，值得单列。循环的"停"有几种来源：

| 停止来源 | 含义 |
|---|---|
| `message.stopReason === "error"/"aborted"` | 模型层出错或被中止，立即早退 |
| `hasMoreToolCalls === false`（即 `executedToolBatch.terminate === true`） | 工具批次主动要求终止（见 4.5） |
| `shouldStopAfterTurn` 返回 `true` | 宿主主动叫停 |
| 没有更多工具、也没有 follow-up | 自然结束 |

把"为什么停"显式建模成多个明确来源，而不是一个含糊的布尔值，是这个循环健壮性的来源之一。

---

## 4.4 streamAssistantResponse：把消息变成流，再把流收回消息

`streamAssistantResponse` 是循环与模型层的接触面。它的工作可以概括为一句话：**把 `AgentMessage[]` 翻译成 LLM 认识的格式，发起流式请求，再把流式事件实时回放、最后收敛成一条完整的 `AssistantMessage`。**

几个关键设计：

**① 上下文变换与格式转换是两个独立步骤。**

```ts
let messages = context.messages;
if (config.transformContext) {
  messages = await config.transformContext(messages, signal);   // AgentMessage[] → AgentMessage[]
}
const llmMessages = await config.convertToLlm(messages);          // AgentMessage[] → Message[]
```

`transformContext` 是"在发给模型前，对消息做最后加工"的钩子（比如裁剪、注入摘要）；`convertToLlm` 则负责把 OpenClaw 内部的 `AgentMessage` 格式转成各 provider 通用的 `Message` 格式。两者分开，意味着"上下文工程"（第 12–13 章）和"格式适配"互不干扰。

**② API Key 在每次请求前重新解析。** 这是一个容易被忽略但很重要的细节：

```ts
const resolvedApiKey =
  (config.getApiKey ? await config.getApiKey(config.model.provider) : undefined) || config.apiKey;
```

代码注释直言其目的——"important for expiring tokens"（对会过期的 token 很重要）。OAuth token 会过期，如果在循环开始时解析一次就一直用，长任务跑到一半就会因 token 失效而崩。**每轮重新解析**保证了长时运行的健壮性。

**③ 流式事件被实时回放成 message_update。** 拿到流后，循环 `for await` 它的每个事件——`text_delta`、`thinking_delta`、`toolcall_delta` 等——并把不断增长的"部分消息"（partial message）通过 `message_update` 事件 emit 出去：

```ts
case "text_delta": case "toolcall_delta": /* ... */ {
  if (partialMessage) {
    partialMessage = event.partial;
    context.messages[context.messages.length - 1] = event.partial;  // 原地更新最后一条
    await emit({ type: "message_update", assistantMessageEvent: event, message: { ...event.partial } });
  }
  break;
}
```

这就是你在 TUI 里看到文字"一个字一个字蹦出来"的底层机制。注意它**原地替换上下文里的最后一条消息**——上下文始终持有"当前最新的部分消息"，直到 `done`/`error` 事件到来，用 `response.result()` 得到的最终消息收尾。

> **启示**：流式不只是"为了好看"。把 partial message 实时放进上下文与事件流，意味着上层（TUI、日志、中止逻辑）能在模型还没说完时就开始响应——这是低延迟交互与可中断性的基础。

---

## 4.5 executeToolCalls：并行还是串行，谁说了算

模型说"我要调这几个工具"之后，`executeToolCalls` 负责把它们真正执行掉。这里有几个对 Agent 工程很关键的设计。

### 并行 vs 串行：由工具自己声明

```ts
const hasSequentialToolCall = toolCalls.some(
  (tc) => currentContext.tools?.find((t) => t.name === tc.name)?.executionMode === "sequential",
);
if (config.toolExecution === "sequential" || hasSequentialToolCall) {
  return executeToolCallsSequential(...);
}
return executeToolCallsParallel(...);
```

逻辑很清晰：**默认并行执行**（更快），但只要批次里有任何一个工具声明了 `executionMode === "sequential"`，或全局配置要求串行，就走串行路径。

为什么要这个开关？因为工具的副作用性质不同：读文件、搜网页这类**只读/幂等**操作并行跑没问题；而像"编辑同一个文件"这类有**写副作用、有顺序依赖**的操作，并行会互相踩踏。让工具自己声明执行模式，是把"安全约束"下放到最了解情况的地方。

### 一次工具调用的生命周期：prepare → validate → gate → execute → finalize

无论并行还是串行，每个工具调用都经历 `prepareToolCall` 的把关。这段代码是第 1 章"约束与恢复"职责在循环内的第一道闸门，值得细看：

```ts
async function prepareToolCall(...) {
  const tool = currentContext.tools?.find((t) => t.name === toolCall.name);
  if (!tool) {
    return { kind: "immediate", result: createErrorToolResult(`Tool ${toolCall.name} not found`), isError: true };
  }
  const preparedToolCall = prepareToolCallArguments(tool, toolCall);   // 1. 参数预处理
  const validatedArgs = validateToolArguments(tool, preparedToolCall); // 2. 参数校验（schema）
  if (config.beforeToolCall) {                                          // 3. 前置钩子（审批/拦截）
    const beforeResult = await config.beforeToolCall({ assistantMessage, toolCall, args: validatedArgs, context }, signal);
    if (beforeResult?.block) {
      return { kind: "immediate", result: createErrorToolResult(beforeResult.reason || "Tool execution was blocked"), isError: true };
    }
  }
  return { kind: "prepared", toolCall, tool, args: validatedArgs };
}
```

这条流水线把四道关卡串在一起：

1. **工具存在性检查**：模型可能幻觉出一个不存在的工具名，直接返回错误结果（而不是崩溃）。
2. **参数校验** `validateToolArguments`：模型给的参数可能不符合 schema，先验证。
3. **前置钩子** `beforeToolCall`：这是**最重要的扩展点**。宿主在这里实现权限审批、危险操作拦截——返回 `{ block: true, reason }` 就能否决一次工具调用，模型会收到一条"被拦截"的错误结果。第 10 章的审批流，正是挂在这个钩子上。
4. **中止检查**：每一步后都检查 `signal?.aborted`，保证用户随时能叫停。

注意一个优雅的设计：被拦截/出错的调用返回 `kind: "immediate"`（带一个错误结果），通过的返回 `kind: "prepared"`（待执行）。**错误不是异常抛出，而是被建模成一种正常的工具结果**——因为"工具失败"本身就是 Agent 必须能消化的信息：模型看到错误结果后，往往会自己调整策略重试。这是"反馈循环"职责的微观体现。

### terminate：工具如何让整个循环停下来

执行完一批工具后，`shouldTerminateToolBatch` 决定是否终止：

```ts
function shouldTerminateToolBatch(finalizedCalls): boolean {
  return finalizedCalls.length > 0 &&
    finalizedCalls.every((finalized) => finalized.result.terminate === true);
}
```

只有当**这一批所有工具结果都标记了 `terminate === true`** 时，整个工具批次才算"要求终止"，进而让 `hasMoreToolCalls` 变 false。这给了工具一种"我执行完后，Agent 应该收工了"的表达能力（比如一个"完成任务并提交"的终结性工具）。

### afterToolCall：执行后的修正机会

`AgentLoopConfig` 还提供了 `afterToolCall` 钩子，让宿主在工具执行后、结果回灌前，对结果做覆盖或加工。结合 `beforeToolCall`，宿主就拥有了一对完整的"工具调用前后拦截器"——这正是搭建 Harness 的标准接缝。

---

## 4.6 Agent 类与配置钩子：循环的"可编程性"

到这里你可能已经注意到，`runLoop` 本身几乎不包含任何 OpenClaw 特有的策略。它把所有"业务决策"都委托给了 `AgentLoopConfig` 上的一组可选钩子。把它们集中看一遍，你就握住了"如何编程这个循环"的全部把手：

| 钩子 | 触发时机 | 典型用途 |
|---|---|---|
| `convertToLlm` | 每次请求前（必填） | AgentMessage → provider Message 格式转换 |
| `transformContext` | convertToLlm 之前 | 上下文裁剪、摘要注入 |
| `getApiKey` | 每轮请求前 | 解析会过期的 token |
| `beforeToolCall` | 工具执行前 | **权限审批、危险操作拦截** |
| `afterToolCall` | 工具执行后 | 结果加工、覆盖 |
| `prepareNextTurn` | 每个 turn 结束后 | 动态换模型、调推理强度、替换上下文 |
| `shouldStopAfterTurn` | 每个 turn 结束后 | 宿主主动叫停 |
| `getSteeringMessages` | 每轮迭代前 | 拉取用户中途插入的话 |
| `getFollowUpMessages` | 内层循环退出后 | 拉取后续消息以开新一轮 |

而 `agent.ts`（612 行）里的 `Agent` 类，则是把这些钩子、队列模式、生命周期组织成一个更易用的面向对象封装——还记得第 3 章吗？`src/agents/runtime` 正是 `extends` 了这个 `Agent` 类，并注入了 runtime deps。

**这套"核心循环 + 可插拔钩子"的设计，是本章最该带走的工程范式。** 它让 `agent-core` 成为一个"机制"（mechanism），而把"策略"（policy）留给宿主。机制稳定、策略灵活——这正是一个能长期演进的 Agent 内核应有的样子。

---

## 本章小结

- `agent-core` 通过一个 45 行的 `runtime-deps` 契约（`streamSimple`/`completeSimple`）保持 provider 无关，换来可复用、可测试、边界清晰三大好处。
- 对外提供三层 API：便利的 `agentLoop`/`agentLoopContinue`、回调式的 `runAgentLoop*`、以及核心的私有 `runLoop`；"新起"与"继续"两种姿态是 Agent 状态机的基础。
- `runLoop` 的双层循环分别处理"工具驱动的连续性"（内层）与"消息驱动的连续性"（外层）；一轮内层做六件事，核心是"流式生成 → 抽取工具调用 → 执行 → **把结果 push 回上下文** → 判断是否停"。
- `streamAssistantResponse` 负责格式转换、每轮重解析 API Key（防 token 过期）、把流式事件实时回放成 `message_update`，最终收敛成完整消息。
- `executeToolCalls` 默认并行、按工具声明转串行；每个调用经过 prepare→validate→`beforeToolCall` 闸门→execute→finalize，错误被建模成正常工具结果而非异常；`terminate` 让工具能终结循环。
- 整个循环是"机制"，所有策略通过 `AgentLoopConfig` 的钩子下放给宿主——这套机制/策略分离，是值得每个 Agent 开发者借鉴的内核范式。

下一章，我们上升一层，看 `src/harness/agent-harness.ts` 如何把这个单轮循环编排成"会话、阶段、压缩、Skill 调用"俱全的完整任务流程。

---

> **动手实验（建议在读第 5 章前完成）**
> 1. 打开 `packages/agent-core/src/agent-loop.test.ts`，看测试是如何注入一个假的 `streamFn` 来驱动整个循环的。这会让你彻底明白 4.1 节"可测试"的含义——并教你如何为自己的 Agent 循环写单测。
> 2. 在 `runLoop` 里找到 `currentContext.messages.push(result)` 那一行，用一句话向同事解释："为什么把这一行删掉，Agent 就退化成了一个只会调一次工具的玩具？"
