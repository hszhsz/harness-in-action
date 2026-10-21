# 第 8 章　统一工具模型：ToolDefinition、负载与暴露

> 工具系统要解决的不是「怎么执行」，而是「模型此刻应该看到什么、能调用什么」。

一个成熟的 Agent 工具系统至少要回答四个问题：

1. 工具长什么样（schema / 描述）？
2. 模型此刻能看到哪些工具？
3. 哪些工具可以被别的工具调用？
4. 执行结果如何稳定地返回给模型与 UI？

pi 用 `ToolDefinition`、**工具负载（loadout）**、**暴露模式（exposure）** 和 `ToolResult` 四套机制分别回答。本章逐一拆开。

---

## 8.1 八个内置工具与三套预设

`packages/coding-agent/src/core/tools/index.ts:95`：

```ts
export type ToolName = "read" | "bash" | "powershell" | "edit" | "write" | "grep" | "find" | "ls";
```

- **默认编码预设**：`read` / `bash` / `edit` / `write`；
- **只读预设**：`read` / `grep` / `find` / `ls`；
- **全集**：八个（含 `powershell`，Windows 原生 shell 工具）。

每个工具都有两套构造器：

- `createXxxToolDefinition(cwd, options)` → `ToolDefinition`（产品层完整定义，含提示词片段、渲染器）；
- `createXxxTool(cwd, options)` → `AgentTool`（内核可执行的最小定义）。

两者通过 `wrapToolDefinition()` 连接。

## 8.2 `ToolDefinition` 是 `AgentTool` 的超集

内核只认识 `AgentTool`：名字、参数 schema、执行函数、结果。产品层需要更多东西：

```ts
export interface ToolDefinition<TParams, TDetails, TState> {
	name: string;
	label: string;
	description: string;

	// 系统提示词集成
	promptSnippet?: string;
	promptGuidelines?: string[];

	parameters: TParams;
	outputSchema?: TSchema;
	prepareArguments?: (args: unknown) => Static<TParams>;
	constrainedSampling?: false | ConstrainedSamplingConfig;

	// 暴露与负载
	exposure?: ToolExposure;          // 默认 "direct"
	namespace?: ToolNamespace;        // 例如 mcp__docs
	reachesTools?: (input) => "all" | readonly string[];
	defaultActive?: boolean;
	prepareLoadout?: (loadout: ToolLoadout) => ToolLoadoutChanges | undefined;

	// 执行与渲染
	executionMode?: ToolExecutionMode;
	execute(toolCallId, params, signal, onUpdate, ctx: ExtensionToolContext): Promise<AgentToolResult<TDetails>>;
	renderShell?: "default" | "self";
	renderCall?(args, theme, context): Component;
	renderResult?(result, options, theme, context): Component;
}
```

注意 `execute` 的第五个参数 `ctx: ExtensionToolContext`：工具上下文由 `AgentSession` 注入，提供 `executeTool()`（嵌套调用）、`getSystemPrompt()`、事件发送等能力。**这是工具从「函数」变成「系统公民」的关键。**

`wrapToolDefinition()` 做的是字段映射，并保留一个可选的 `ctxFactory`：

```ts
export function wrapToolDefinition(definition, ctxFactory?): AgentTool {
	return {
		name: definition.name,
		label: definition.label,
		description: definition.description,
		parameters: definition.parameters,
		outputSchema: definition.outputSchema,
		constrainedSampling: definition.constrainedSampling,
		prepareArguments: definition.prepareArguments,
		executionMode: definition.executionMode,
		execute: (id, params, signal, onUpdate, ctx?) =>
			definition.execute(id, params, signal, onUpdate, ctx ?? ctxFactory?.(id, signal)),
	};
}
```

反过来，`createToolDefinitionFromAgentTool()` 能把一个裸 `AgentTool` 提升成最小 `ToolDefinition`，让注册表始终以「定义优先」的方式工作。

## 8.3 暴露模式：五种「模型能不能看见/调用」

`ToolExposure` 是 1.3 工具系统里最重要的概念。它把「声明给模型」与「可被执行」拆成两个维度：

| 模式 | 声明给模型 | 可被其他工具调用 | 典型用途 |
|---|---|---|---|
| `direct` | 激活时声明 | 激活时可调用 | 内置工具、默认扩展工具 |
| `model-only` | 激活时声明 | **永不**可调用 | 交互式工具、编排型工具 |
| `codemode` | 不主动声明，除非显式激活 | 注册后始终可调用 | MCP 工具的默认模式 |
| `deferred` | 由 `tool_search` 加载后声明 | 始终可调用 | 大量低频工具 |
| `hidden` | 永不 | 永不 | 内部实现、禁用工具 |

两个容易混淆的点：

1. **激活集合 ≠ 可调用集合**。`getActiveToolNames()` 是「声明给模型的工具」；`_getCallableTools()` 还会包含所有 `codemode` / `deferred` 工具。所以一个工具可以「模型看不见，但 codemode 脚本能调」。
2. **`model-only` 不是「只给模型看」**。它恰恰相反：模型能看见并直接调用，但其他工具不能通过 `executeTool()` 调它。这样可防止编排工具绕过人类交互。

### `defaultActive`

`direct` 与 `model-only` 工具在注册时默认激活；`codemode` / `deferred` 默认不激活。`defaultActive: false` 可以让一个 `direct` 工具也默认关闭，之后通过 `--tools`、`defaultTools` 设置或 `setActiveTools()` 打开。

### 工具负载（loadout）

一组工具被激活后，`_applyToolLoadout()` 会构建一个 `ToolLoadout`：

```ts
{
	declared,        // 将要声明给模型的工具
	callable,        // 可被 executeTool 调用的工具
	registered,      // 所有已注册工具
	getExposure,
	getNamespace,
	getPromptGuidelines,
}
```

每个带 `prepareLoadout` 的工具都能看到这份快照，并返回：

- `descriptions`：改写在本次请求中使用的工具描述；
- `hiddenDeclarations`：临时隐藏某些工具的声明。

这就是 codemode / tool_search 能让「自己的描述里列出当前可调用工具」的机制：它们不是硬编码列表，而是**在每次负载变化时动态生成**。

> **设计启示**：工具系统不应该只有一个「启用/禁用」开关。pi 把「声明」「可调用」「已注册」拆开，才支撑得起 MCP 的延迟加载、codemode 的脚本调用、以及「工具编排工具」的复杂场景。

## 8.4 `prepareArguments`：在 schema 校验前修复模型输出

模型给出的工具参数经常有瑕疵：JSON 里多一层包装、数字被写成字符串、必填字段名拼错。`prepareArguments` 是 schema 校验前的**兼容层**：

```ts
prepareArguments?: (args: unknown) => Static<TParams>;
```

`agent-loop.ts` 的 `prepareToolCallArguments()` 会先调用它，再走 `validateToolArguments()`。如果修复失败，校验错误会变成错误工具结果，模型有机会重试。

这与「把 schema 放宽」是两回事：schema 仍然是严格契约，修复只是把常见的模型方言翻译成契约。**不要把业务校验塞进 `prepareArguments`**——它应该只做形状修复。

## 8.5 结果契约：让 UI、模型、程序化调用方各取所需

`AgentToolResult` 的字段各有明确受众：

| 字段 | 受众 | 说明 |
|---|---|---|
| `content` | 模型 | 文本/图片，最终进入 toolResult 消息 |
| `details` | UI / 日志 | 任意结构化数据，不进模型 |
| `structuredContent` | 程序化调用方 / codemode | 符合 `outputSchema` 的机器可读结果 |
| `usage` | 成本统计 | 工具自身的 token/费用（通常用于子模型调用） |
| `isError` | 循环 / UI | 失败标记；模型会看到错误内容 |
| `terminate` | 循环 | 建议本批结束后停止 |
| `durationMs` | UI | 由运行时测量，工具无需填写 |

两个约定：

- **声明了 `outputSchema` 的工具，就应当总是提供 `structuredContent`**。codemode 脚本拿到的就是它，而不是拼好的文本。
- **失败要标记 `isError`，不要只在文本里写「失败了」**。否则 UI 无法区分正常输出与错误，重试策略也无法判断。

## 8.6 工具注册与来源追踪

`AgentSession` 维护一个 `_toolDefinitions: Map<string, { definition, sourceInfo }>`。每个工具都带着来源：

- 内置工具（`builtin:` 前缀）；
- 扩展注册的工具（扩展文件路径）；
- MCP 工具（服务器名与作用域）；
- 从会话转录恢复的工具。

`sourceInfo` 不只用于展示。它决定了：

- 项目不受信任时，来自项目的工具是否允许注册；
- 扩展重载时，哪些工具需要被替换；
- 冲突时，谁覆盖谁；
- `/tools` 面板如何分组。

**工具不是凭空出现的函数，而是有来源、有生命周期、有信任级别的资源。**

## 8.7 工具与系统提示词

工具定义里的 `promptSnippet` 与 `promptGuidelines` 会被 `system-prompt.ts` 用来生成：

- `Available tools` 区：每个工具一行摘要；
- `Guidelines` 区：工具作者建议的使用规则。

这意味着「工具描述」实际上有两份：

1. **provider 看到的 schema description**：决定模型是否/如何调用；
2. **系统提示词里的摘要与规则**：决定模型在什么场景下想到它。

两者由同一个 `ToolDefinition` 驱动，但渲染位置不同。第 14 章会看到系统提示词如何按工具负载增量刷新。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/coding-agent/src/core/tools/index.ts` | 八个工具、三套预设、构造器 |
| `packages/coding-agent/src/core/extensions/types.ts` | `ToolDefinition`、`ToolExposure`、`ToolLoadout` |
| `packages/coding-agent/src/core/tools/tool-definition-wrapper.ts` | `ToolDefinition` ↔ `AgentTool` |
| `packages/coding-agent/src/core/agent-session.ts` | 工具注册表、负载计算、激活集合 |
| `packages/agent/src/types.ts` | `AgentTool`、`AgentToolResult` |
| `packages/coding-agent/src/core/system-prompt.ts` | 工具摘要与规则渲染 |

## 动手实验

1. 写一个 `ToolDefinition`，设置 `exposure: "codemode"` 与 `defaultActive: false`。注册后验证：它不出现在模型的工具声明里，但可以通过 `ctx.executeTool()` 调用。
2. 给一个工具加 `prepareLoadout`，让它在描述里列出当前所有 `direct` 工具。切换激活集合，观察描述是否随请求变化。
3. 故意让模型传入一个「多包了一层 `{ arguments: ... }`」的参数，分别测试有/无 `prepareArguments` 时的行为。
4. 阅读 `_applyToolLoadout` 与 `_getCallableTools`，回答：一个 `hidden` 工具能否被 codemode 调用？为什么？
