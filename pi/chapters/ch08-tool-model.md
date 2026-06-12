# 第 8 章　统一工具模型：`ToolDefinition` 与 `AgentTool`

> 内核只认一个抽象 `AgentTool`，应用层却要把工具做得「能渲染、能写提示词、能被扩展覆盖」。这两者之间，差了一个 `ToolDefinition`。

第 3 章我们看过循环如何「抽取工具调用 → 执行 → 把结果喂回去」，但当时把工具当成了黑盒。这一章我们打开这个黑盒：pi 的工具到底长什么样、内核眼里的工具（`AgentTool`）和应用层眼里的工具（`ToolDefinition`）有何区别、七个内置工具怎么被工厂函数批量造出来、以及一个最容易被忽略却极其实用的字段——`prepareArguments`。

---

## 8.1 两种工具类型：`AgentTool`（内核） vs `ToolDefinition`（应用）

pi 里有两个长得很像的「工具」类型，分处两层，职责不同。

**`AgentTool`** 住在内核 `pi-agent-core`（`packages/agent/src/types.ts`）。它是循环执行工具时**唯一认识**的形状，只包含「跑起来」必需的字段：

```ts
interface AgentTool<TParams, TDetails> {
	name: string;
	label: string;
	description: string;
	parameters: TParams;             // TypeBox schema
	prepareArguments?: (args) => ...;
	executionMode?: ToolExecutionMode;
	execute(toolCallId, params, signal, onUpdate): Promise<AgentToolResult>;
}
```

注意它**没有任何「怎么显示」的字段**——内核不关心 UI，这是第 2 章「核心精简」红线的又一次贯彻。

**`ToolDefinition`** 住在应用层 `pi-coding-agent`（`core/extensions/types.ts`）。它是 `AgentTool` 的超集，多出一堆「应用层才在乎」的东西：

```ts
interface ToolDefinition<TParams, TDetails, TState> {
	name; label; description;
	promptSnippet?: string;       // 系统提示词「可用工具」段落的一行简介
	promptGuidelines?: string[];  // 该工具激活时追加到系统提示词的指引
	parameters: TParams;
	renderShell?: "default" | "self";
	prepareArguments?: (args) => Static<TParams>;
	executionMode?: ToolExecutionMode;
	execute(toolCallId, params, signal, onUpdate, ctx: ExtensionContext): ...;  // 多一个 ctx
	renderCall?: (...) => Component;     // 工具调用怎么画
	renderResult?: (...) => Component;   // 工具结果怎么画
}
```

差异一目了然：`ToolDefinition` 多了 **提示词元数据**（`promptSnippet` / `promptGuidelines`）、**UI 渲染器**（`renderCall` / `renderResult`），以及 `execute` 多收一个 **`ExtensionContext`**（让工具能访问 UI、会话、配置等应用层能力）。

> **给 Agent 开发者的启示**：把「执行契约」和「展示/集成契约」拆成两个类型，是 pi 保持内核纯净的关键手法。内核只依赖最小的 `AgentTool`，于是它可以被 CLI、Web、测试共享；而所有「怎么显示、怎么写进提示词、怎么拿应用上下文」的富信息，都压在应用层的 `ToolDefinition` 里，不向下污染。

## 8.2 `wrapToolDefinition`：从应用工具降维成内核工具

既然内核只认 `AgentTool`，那应用层这个胖胖的 `ToolDefinition` 怎么交给循环跑？答案是 `wrapToolDefinition`（`core/tools/tool-definition-wrapper.ts`）——一个「降维」适配器：

```ts
export function wrapToolDefinition(definition, ctxFactory?): AgentTool {
	return {
		name: definition.name,
		label: definition.label,
		description: definition.description,
		parameters: definition.parameters,
		prepareArguments: definition.prepareArguments,
		executionMode: definition.executionMode,
		execute: (toolCallId, params, signal, onUpdate) =>
			definition.execute(toolCallId, params, signal, onUpdate, ctxFactory?.()),
	};
}
```

它做的事极简：**只挑出 `AgentTool` 需要的字段，把渲染器、提示词元数据统统丢掉**；唯一的小魔法是 `execute` 那行——它用一个 `ctxFactory()` 工厂**惰性地**生成 `ExtensionContext`，再补成 `ToolDefinition.execute` 的第五个参数。为什么用工厂而非直接传一个 ctx 对象？因为 `ExtensionContext` 是「当前这一刻」的上下文（当前会话、当前 UI 状态），每次执行都该拿最新的，工厂保证了这一点。

反方向也有：`createToolDefinitionFromAgentTool` 能把一个裸 `AgentTool`「升维」成最小的 `ToolDefinition`（补一个不渲染、不带提示词的空壳）。注释点明了它的用途：让应用层的工具注册表「definition-first」——即便调用方塞进来一个裸 `AgentTool` 覆盖，注册表内部也能统一当 `ToolDefinition` 管理。

这一对正反适配器，就是第 2 章那条接缝在工具层的具体形态：**内核与应用各用各的类型，边界处转换。**

## 8.3 七个内置工具与工厂函数家族

pi-coding-agent 内置七个工具，名字定义在一处（`core/tools/index.ts`）：

```ts
export type ToolName = "read" | "bash" | "edit" | "write" | "grep" | "find" | "ls";
export const allToolNames = new Set(["read", "bash", "edit", "write", "grep", "find", "ls"]);
```

按「会不会改文件」可以分成两组：

| 组 | 工具 | 性质 |
|---|---|---|
| 只读 | `read` / `grep` / `find` / `ls` | 无副作用，可放心并发 |
| 会改 | `bash` / `edit` / `write` | 有副作用，需小心串行（见第 9 章 file-mutation-queue） |

每个工具都成对地提供两个工厂：`createXxxTool`（造 `AgentTool`）和 `createXxxToolDefinition`（造 `ToolDefinition`）。在此之上，`index.ts` 又封了一组「打包工厂」，按用途批量生产：

```ts
createCodingTools(cwd)      // read + bash + edit + write —— 能写代码的全套
createReadOnlyTools(cwd)    // read + grep + find + ls    —— 只读，适合「调研模式」
createAllTools(cwd)         // 七个全要，返回 Record<ToolName, Tool>
```

还有两个「单点工厂」`createTool(name, cwd)` / `createToolDefinition(name, cwd)`，用一个 `switch` 按名字造单个工具——典型用途是「用户在配置里只开了某几个工具」时按需实例化。

注意所有工厂的第一个参数都是 `cwd`（当前工作目录）。这不是偶然——工具是**绑定到一个工作目录**的：`read` 在哪读、`bash` 在哪跑、`edit` 改哪里的文件，都以这个 `cwd` 为根。把 `cwd` 做成构造参数而非全局变量，意味着同一个进程里可以有指向不同目录的多套工具（比如多会话、多工作区），互不干扰。

> 这套「成对工厂 + 打包工厂 + 单点工厂」的组织看似啰嗦，实则是把「我要哪些工具、它们绑哪个目录、要内核形态还是应用形态」这几个正交维度彻底解耦。新增一个工具 = 写一对 `createXxxTool`/`createXxxToolDefinition` + 在几个名单里登记一次。

## 8.4 `prepareArguments`：模型「手抖」时的兼容垫片

`prepareArguments` 是 `ToolDefinition` 和 `AgentTool` 共有的一个可选字段，注释把它的定位说得很准：

> Optional compatibility shim to prepare raw tool call arguments **before schema validation**. Must return an object conforming to TParams.

翻译过来：**它是 schema 校验之前的一道「参数预处理」。** 回顾第 3 章 `prepareToolCall` 的流程——找工具 → （`prepareArguments`）→ 校验参数 → 过 `beforeToolCall` 闸门 → 执行。`prepareArguments` 就插在「校验之前」，专门用来收拾模型「手抖」生成的、形状不太对但意图明确的参数。

典型场景：

- 模型把本该是数组的 `paths` 传成了单个字符串 → `prepareArguments` 把它包成 `[str]`。
- 模型多塞了一个 schema 里没有的字段 → 提前剔掉，免得严格校验直接拒绝。
- 不同 provider 对同一个工具的参数命名习惯有细微差异 → 在这里归一化。

它的价值在于：**与其让一个本可挽救的工具调用因为参数格式不完美而失败（虽然第 3 章说过失败会被包成错误结果喂回，模型可能自纠，但那要多花一轮），不如在校验前先把常见的格式问题悄悄补正。** 这是「对模型输出宽容一点」的工程务实——模型不是编译器，它的输出有噪声，`prepareArguments` 就是吸收这层噪声的垫片。

注意它的契约很严：**必须返回一个符合 `TParams` 的对象**。也就是说它只负责「把歪的掰正」，不负责「校验」——真正的校验仍由后面的 TypeBox schema 把关。垫片归垫片，关卡归关卡，职责不混。

## 8.5 工具的执行契约：`execute` 与流式 `onUpdate`

最后看每个工具真正干活的 `execute`。它的签名（以 `AgentTool` 版为例）是：

```ts
execute(
	toolCallId: string,
	params: Static<TParams>,               // 已校验的参数
	signal: AbortSignal | undefined,       // 中止信号
	onUpdate: AgentToolUpdateCallback,     // 流式进度回调
): Promise<AgentToolResult<TDetails>>
```

四个参数对应工具执行的四种需求：

- **`toolCallId`**：这次调用的唯一标识，用于把事件、结果对应回是哪一次调用（并行时尤其重要，见第 3 章「end 按完成序、消息按源序」）。
- **`params`**：已经过 `prepareArguments` 预处理 + schema 校验的干净参数。
- **`signal`**：`AbortSignal`，让长跑工具（比如 `bash` 跑一个慢命令）能响应用户的 `abort()`，及时停下。
- **`onUpdate`**：流式进度回调。工具不必等全部干完才返回——它可以边跑边 `onUpdate(...)` 推送中间进度（比如 `bash` 实时回显输出、`grep` 逐步报告匹配数），UI 据此实时刷新。返回值里的 `TDetails` 是工具特有的结构化细节（比如 `bash` 的退出码、`edit` 的 diff），供渲染器画出漂亮的结果。

这套契约把「工具」抽象成了一个**可中止、可流式汇报、返回结构化结果**的异步函数。它足够通用——无论是读文件这种瞬间完成的操作，还是跑测试这种几分钟的长任务，都能套进同一个形状。而正因为形状统一，第 3 章那套循环才能不关心工具具体是什么，只管「调 execute、拿结果、喂回去」。

至此，「工具是什么」这件事被我们彻底拆清：内核的 `AgentTool` 管执行、应用的 `ToolDefinition` 管展示与集成、`wrapToolDefinition` 在两者间转换、七个内置工具由工厂家族批量生产、`prepareArguments` 吸收模型输出的噪声。但有一个问题还悬着：`bash`、`read` 这些工具**真正动手的那一刻**——读哪个磁盘、在哪个 shell 跑命令——是怎么做到「可被替换、可被远程化」的?这就是下一章 `Operations` 的主题。

---

## 本章小结

- 内核的 `AgentTool` 只含执行必需字段（无 UI），应用的 `ToolDefinition` 是其超集，多出提示词元数据、渲染器、`ExtensionContext`——「执行契约」与「展示/集成契约」分离。
- `wrapToolDefinition` 把胖 `ToolDefinition` 降维成内核 `AgentTool`（丢渲染器、用 `ctxFactory` 惰性补 ctx）；`createToolDefinitionFromAgentTool` 反向升维，让应用注册表统一 definition-first。
- 七个内置工具（read/bash/edit/write/grep/find/ls）按「只读/会改」分组；每个工具成对提供 `createXxxTool`/`createXxxToolDefinition`，再由 `createCodingTools`/`createReadOnlyTools`/`createAllTools` 等打包工厂按用途批量生产；所有工厂以 `cwd` 为构造参数。
- `prepareArguments` 是 schema 校验之前的兼容垫片，专门掰正模型「手抖」生成的参数，对模型输出的噪声保持务实的宽容。
- `execute(toolCallId, params, signal, onUpdate)` 把工具抽象成可中止、可流式汇报、返回结构化 `TDetails` 的异步函数，让循环得以不关心工具具体是什么。

下一章，我们看工具「真正动手那一刻」背后的 `Operations` 抽象——pi 如何让 `bash`/文件操作变得可替换、可远程、可加固。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/coding-agent/src/core/tools/index.ts` | `ToolName`、七个工具的成对工厂、`createCodingTools`/`createReadOnlyTools`/`createAllTools` 等打包工厂 |
| `packages/coding-agent/src/core/tools/tool-definition-wrapper.ts` | `wrapToolDefinition`（降维）、`createToolDefinitionFromAgentTool`（升维） |
| `packages/coding-agent/src/core/extensions/types.ts` | `ToolDefinition` 完整接口（提示词元数据 + 渲染器 + ctx） |
| `packages/agent/src/types.ts` | 内核 `AgentTool`、`AgentToolResult`、`ToolExecutionMode` |

## 动手实验

1. 对照 8.1 的两个类型，列一张「`ToolDefinition` 比 `AgentTool` 多了哪些字段」的表，并对每个多出来的字段回答：它属于「提示词」「UI 渲染」还是「应用上下文」？验证「执行契约 vs 展示契约」的二分。
2. 给某个内置工具（比如 `read`）写一个 `prepareArguments`：当模型把 `path` 误传成 `{ file: "a.txt" }` 时，把它纠正成 `{ path: "a.txt" }`。再构造一个这样的「手抖」工具调用，验证有无 `prepareArguments` 两种情况下，校验是通过还是失败。
3. 跟踪 `createReadOnlyTools` 造出来的四个工具，确认它们都没有 `write`/`edit`/`bash`。设想一个「调研模式」的 Agent：为什么用这套只读工具集 + 默认并行执行，会比全套工具更安全也更快？
