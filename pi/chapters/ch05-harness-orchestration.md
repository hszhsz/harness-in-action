# 第 5 章　AgentSession：把循环变成产品级会话

> `Agent` 让一次任务跑起来；`AgentSession` 让一万次任务都跑得下去。

前两章的内核是「干净」的：没有文件、没有压缩、没有扩展。真正把这些能力缝在一起的是 `packages/coding-agent/src/core/agent-session.ts` —— 一个 **4428 行**的类。它替代了旧版的 `AgentHarness`，是 1.3 架构变化最大的地方。

本章不逐行讲代码，而是回答一个问题：**一个产品级会话，必须在循环外面再加哪些东西？**

---

## 5.1 先纠正一个旧认知：`AgentHarness` 已经不在了

上一版电子书花了整整一章讲 `packages/agent/src/harness/agent-harness.ts` 的「阶段机」：`idle / turn / compaction / branch_summary / retry`、`pendingSessionWrites`、8 个入口方法。

在 1.3 的代码里，这些都不存在了：

- `AgentHarness` 类已删除；
- 阶段机被拆成 `AgentSession` 的多个独立状态与控制器（`_compactionAbortController`、`_retryAbortController`、`_branchSummaryAbortController`、`_autoCompactionAbortController`）；
- `pendingSessionWrites` 被「先写会话树、再刷新投影」的同步路径取代；
- 会话树、压缩、扩展、工具都不再属于 `pi-agent-core`，而是 `coding-agent` 的职责。

这不是简单的重命名，而是**职责重新划线**：内核只保留可复用的循环与工具管线，所有产品语义上移。结果是 `pi-agent-core` 更小、更容易嵌入，而 `AgentSession` 更厚、更贴近真实需求。

## 5.2 一个类，六个装配点

`AgentSession` 的构造函数在 515–526 行做了六次「安装」：

```ts
this._installAgentToolHooks();
this._installAgentNextTurnRefresh();
this._installAgentRequestProjection();
this._installAgentBoundaryHooks();
this._installHiddenDeclarationsProjection();
this._installAgentForcedPromptProjection();
```

它们分别改写 `Agent` 的不同钩子：

| 安装项 | 改写的钩子 | 作用 |
|---|---|---|
| Tool hooks | `beforeToolCall` / `afterToolCall` | 把扩展的 `tool_call` / `tool_result` 事件接进工具管线，并做图片归一化 |
| Next-turn refresh | `prepareNextTurnWithContext` | 请求前压缩、重建系统提示词与工具负载、生成增量系统消息 |
| Request projection | `prepareRequest` | 用会话树投影替换上下文、解析虚拟模型、按路由后的模型检查压缩阈值 |
| Boundary hooks | `finishTurn` | 派发扩展的 `turn_end` 边界事件，并合并「继续」决策 |
| Hidden declarations | `convertToLlm` | 处理不暴露给模型但需要参与会话的工具/消息 |
| Forced prompt projection | 请求前 | 把强制提示词与运行时状态同步进转录 |

它持有的核心协作者包括：

```
AgentSession
├── Agent                    // 低层循环 + 队列
├── SessionManager           // JSONL 树、投影、分支
├── ModelRuntime             // 模型目录、认证、虚拟模型路由
├── ExtensionRunner          // 扩展加载与事件派发
├── SettingsManager          // 压缩、重试、UI、项目信任等设置
├── ResourceLoader           // skills / prompts / themes / packages
└── ToolRegistry             // 内置工具 + 扩展工具 + MCP 工具
```

> **设计启示**：`AgentSession` 没有把「压缩」「重试」「扩展」写成循环里的 `if`，而是全部通过**改写 `Agent` 的钩子**接入。这样内核完全不知道这些概念，产品层却能在正确的时机介入。

## 5.3 请求投影：模型看到的，必须就是会话树里的

这是 `AgentSession` 最重要的不变量：

> `agent.state.messages` 是运行期缓存；`SessionManager.buildSessionProjection()` 才是模型可见上下文的唯一真相。

`_installAgentRequestProjection` 用 `prepareRequest` 强制落实这一点：

```ts
const projection = this.sessionManager.buildSessionProjection();
const canonicalContext = {
	...request.context,
	messages: projection.messages,
	tools: this.agent.state.tools.slice(),
};
```

为什么要覆盖 `request.context.messages`？因为低层循环可能持有尚未投影的临时消息（流式中的 partial、刚追加但未刷新的事务）。如果直接发给模型，就会出现「模型看到了会话树里没有的内容」，压缩、回溯、重放都会失真。

`tools` 则相反：投影只负责「声明」，可执行实现仍来自 `agent.state.tools`。这对应第 3 章的 `declareToolChanges`：**转录声明工具，运行时持有实现**。

### 5.3.1 虚拟模型：选择是一个，请求是另一个

pi 支持「虚拟模型」——用户选中的可能是一个路由器，而不是具体模型。投影钩子在 `prepareRequest` 里解析路由：

```ts
const route = await this._modelRuntime.resolveModel(model, convertToLlm(context.messages), {
	reason: failed ? "retry" : userTurn ? "user" : "continuation",
	thinkingLevel,
	signal,
	failed,
	state,
});
```

路由结果只影响**这一次请求**，不会改写 `agent.state.model`。如果路由器返回了新状态，`AgentSession` 会追加一条 `custom` entry（`VIRTUAL_MODEL_STATE_ENTRY`）持久化它。这样恢复会话时，路由器能从上次的状态继续。

路由之后，如果新模型认为上下文超过阈值，会立刻触发一次 `threshold` 压缩，然后**重新做一遍投影**——这就是 `prepare` 函数被调用两次的原因。

### 5.3.2 系统提示词与工具负载的增量刷新

`_installAgentNextTurnRefresh` 在每轮开始前：

1. 视情况压缩（`_compactBeforeNextAssistantResponse`）；
2. 重新计算 `selectedTools`、`toolSnippets`、`toolGuidelines`；
3. 调用 `_preparePromptAndToolLoadout`，生成一条**增量系统消息**（只包含变化的 sections / toolsAdded / toolsRemoved）；
4. 把它作为 `messages` 返回，由低层循环按正常消息流程写入转录。

这样模型在长会话里不会反复收到完整提示词，缓存命中率更高，转录也更易读。

## 5.4 `prompt()`：一句话进入系统前的五道工序

`AgentSession.prompt()` 不是一个简单的 `agent.prompt()` 包装。它按顺序处理：

```
扩展命令 (/command)  → 若命中，直接执行并返回
  → input 事件（扩展可拦截/改写输入）
    → /skill:name 展开
      → 提示词模板展开（$1 / $@ / ${N:-default}）
        → 若正在流式输出：steer 或 followUp 入队
          → 校验模型与认证
            → _startRun：before_agent_start、注入 nextTurn 消息、强制提示词投影
```

几个容易忽略的行为：

- **扩展命令优先于一切**：即使 Agent 正在流式输出，`/my-command` 也会立即执行，因为它自己管理 LLM 交互。
- **流式期间必须显式指定行为**：调用 `prompt(text)` 而不传 `streamingBehavior` 会直接报错，避免「悄悄排队」造成的用户体验歧义。
- **认证在入口校验**：OAuth 过期会提示 `/login`，而不是等到流式请求里才报错。
- **输入事件在展开之前**：扩展看到的是用户原文，不是展开后的技能文本，便于做权限或审计。

## 5.5 工具钩子：扩展如何真正影响执行

`_beforeToolCall` 只做一件事：把调用转成扩展事件。

```ts
return await runner.emitToolCall({
	type: "tool_call",
	toolName: toolCall.name,
	toolCallId: toolCall.id,
	...(parentToolCallId ? { parentToolCallId } : {}),
	input: args as Record<string, unknown>,
});
```

`_afterToolCall` 则多做一步：先让扩展改写结果，再对图片做归一化（缩放、格式转换），最后返回字段级覆盖。注意图片归一化放在扩展之后，意味着扩展注入的图片也会被处理。

`parentToolCallId` 是 1.3 新增的：当工具通过 `ctx.executeTool()` 发起嵌套调用时，扩展能看到「这个调用是谁发起的」，从而对整个工具链做统一策略（第 15 章）。

## 5.6 边界事件：`turn_end` 不只是通知

1.3 的扩展事件墙引入了「边界事件」的概念。`_installAgentBoundaryHooks` 把 `finishTurn` 包了一层：

```ts
this.agent.finishTurn = async (turn, signal) => {
	this._boundaryDispatchedMessages.add(turn.message);
	const extensionContinue = await this._dispatchTurnEndBoundary(turn.message, turn.toolResults);
	const previousDecision = await previousFinishTurn?.(turn, signal);
	if (previousDecision?.action === "end") return previousDecision;
	if (extensionContinue || previousDecision?.action === "continue") return { action: "continue" };
	return undefined;
};
```

`_dispatchTurnEndBoundary` 会：

1. 找到助手消息与工具结果对应的**已持久化 entry ID**；
2. 以 `emitBoundary` 派发 `turn_end`，并提供一个「构建边界上下文」的回调；
3. 把扩展提交的 drafts（新的 custom / custom_message / context_edit 等）写入会话树；
4. 如果扩展要求继续，再检查边界上下文是否允许继续。

这比旧的「after turn 钩子」强得多：扩展可以**在回合边界安全地改写会话**，而不是只能发通知。`agent_before_settle`、`session_before_tree` 等事件也遵循同样的边界协议。

## 5.7 自动压缩与自动重试：两条独立的恢复链

### 压缩

`_checkCompaction` 在每次运行结束后检查三种情况：

1. **溢出且可重试**：上下文溢出错误或可恢复的长度截断 → 移除失败的助手消息 → 压缩 → `agent.continue()` 重试一次。
2. **溢出但已完成**：响应成功但超出上下文窗口 → 只压缩，不重试（无法从完成的 assistant 消息继续）。
3. **阈值触发**：正常响应但上下文 token 超过阈值 → 压缩，不重试。

溢出恢复有「只尝试一次」的硬限制（`_overflowRecoveryAttempted`）。第二次仍然溢出不循环重试，而是明确报错并建议换模型或减少上下文——避免无限压缩循环。

### 重试

自动重试走另一条链：`_handlePostAgentRun` 发现可重试错误 → `_prepareRetry` 指数退避 → 把失败响应**从投影中省略**（`_omitRecoveryAttempt`）→ 下一轮带 `failed` 标记重新请求。

「保留在原始历史、但从模型投影中移除」是这里的关键设计：会话记录保留完整失败现场，供审计与 UI 展示；模型不会因为看到自己的失败响应而反复道歉或复读。

两条链的优先级：**可重试错误先重试，重试耗尽后再考虑压缩**。因为瞬时网络错误压缩上下文没有意义。

## 5.8 `AgentSessionRuntime`：会话之上的会话

`AgentSession` 管理「一个会话的生命周期」；`AgentSessionRuntime`（447 行）管理「在多个会话之间切换」。它负责：

- `switchSession()`：保存当前会话、拆除服务、加载目标会话、重建服务；
- `newSession()`：以当前工作目录为上下文创建新会话；
- `fork()`：从指定 entry 派生一个新会话，并派发 `session_before_fork`；
- `importFromJsonl()`：导入外部会话文件；
- `session_before_switch` 钩子与 teardown/rebind 生命周期。

它解决的是一类很容易出错的问题：**扩展、工具、模型运行时都绑定在某个 cwd 和会话上**。切换会话时如果只换 SessionManager 而不重建这些服务，就会出现「新会话却用着旧扩展状态」的幽灵 bug。Runtime 把这些绑定显式化。

## 5.9 事件面：UI 与扩展看到的是同一套事实

`AgentSession` 向外发出两类事件：

- **Agent 事件**：`agent_start`、`message_*`、`turn_*`、`tool_execution_*`、`agent_end`；
- **会话事件**：`compaction_start/end`、`auto_retry_start/end`、`entry_appended`、`session_info_changed`、`queue_update` 等。

扩展通过 `ExtensionRunner` 订阅同名事件，但两者的语义并不完全相同：扩展事件有「结果返回值」，可以改写行为；会话事件更多是状态通知。第 13 章会把这张「事件墙」完整摊开。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/coding-agent/src/core/agent-session.ts` | 产品级会话编排（4428 行） |
| `packages/coding-agent/src/core/agent-session-runtime.ts` | 多会话切换、fork、导入 |
| `packages/coding-agent/src/core/session-manager.ts` | 会话树与投影 |
| `packages/coding-agent/src/core/compaction/compaction.ts` | 压缩算法 |
| `packages/coding-agent/src/core/extensions/runner.ts` | 事件派发与边界提交 |
| `packages/coding-agent/src/core/model-runtime.ts` | 模型、认证、虚拟路由 |

## 动手实验

1. 在 `agent-session.ts` 里搜索 `prepareRequest`、`prepareNextTurnWithContext`、`finishTurn`，分别列出它们改写了什么、在什么时机运行。画一张与第 4 章时间线对应的「产品层注入图」。
2. 构造一个超过上下文阈值的会话（或用设置把 `reserveTokens` 调大），观察 `compaction_start` / `compaction_end` 事件的 `reason` 是 `threshold` 还是 `overflow`，以及 `willRetry` 的取值。
3. 把 `getRetrySettings().maxRetries` 设为 1，用一个必定返回 529 的假 provider 跑一次，观察 `auto_retry_start` → `auto_retry_end` 的事件序列，并检查失败消息是否仍在 JSONL 里、是否出现在下一轮请求中。
4. 阅读 `AgentSessionRuntime.switchSession`，列出它在切换前后分别重建了哪些服务。思考：如果漏掉 `ExtensionRunner` 的重绑，会出现什么症状？
