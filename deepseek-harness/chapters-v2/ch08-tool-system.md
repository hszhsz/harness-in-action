# 第 8 章　工具系统：ToolDefinition 与守卫管线

工具系统是 Agent 的"手"。dsh 的 `ctx.tools` 是一个作用域工具注册表 + 守卫执行管线：模型只能看到 allowlist 投影出的 schema，任何工具调用都要穿过六段守卫。这一章拆开 `packages/core/tools/src/index.ts`。读完后，你应该能理解工具从注册到执行的完整生命周期，以及为什么"工具只返回规范值"是"模型可见即日志"在工具层的延伸。

## `ToolDefinition`：schema + execute + 纯投影三件套

一个注册的工具是 `ToolDefinition`：模型可见的 `ToolSchema` + 强制的规范输出声明 + `execute` 函数 + 可选的最终内容回调 + 可选 UI 呈现器。`packages/core/tools/src/index.ts` 导出了 `defineTool` 和相关的 schema DSL：

```ts
export {
  defineTool,
  valueSchemaSpecToJsonSchema,
  parameterSchemaSpecToJsonSchema,
  validateArgs,
  ToolArgsError,
  type ValueSchemaAnnotations,
  type StringValueSchemaSpec,
  type NumberValueSchemaSpec,
  type IntegerValueSchemaSpec,
  type BooleanValueSchemaSpec,
  type NullValueSchemaSpec,
  type ArrayValueSchemaSpec,
  type ObjectValueSchemaSpec,
  type JsonValueSchemaSpec,
  type OneOfValueSchemaSpec,
  type ValueSchemaSpec,
  type ParameterPropertySpec,
  type ParameterSchemaSpec,
  type ParameterJsonSchema,
  type InferValue,
  type InferArgs,
  type DefineToolOptions,
} from './schema.ts'
```

关键设计：

- **`output` 是强制的规范输出声明**：含 JSON Schema（对每个成功的规范值强制校验）、纯投影 `render`（从校验过的参数和值投影到 Native/model content）。
- **`execute` 返回唯一规范值**：工具不能直接往模型对话里塞东西——它只能返回值，由管线决定如何呈现。这是"模型可见即日志"不变量在工具层的体现。
- **`finalizeContent` 是同步的最后一英里变换**：注册表在 execute 开始时快照此回调，对每个规范化结果恰好调用一次。**必须 total，不许 throw**。
- **`timeoutMs` 是协作式超时预算**：由 `timeout-policy` 工具守卫强制执行，**绝不发给模型**——`schemas()` 只 whitelist name/description/parameters。
- **`isConcurrencySafe` 是纯同步分类器**：只有 `true` 才 opt-in 并行。这个元数据也绝不模型可见。

## JSON-value schema DSL：一套词汇描述参数与输出

插件作者用一套词汇描述类型化参数和类型化输出值。`ValueSchemaSpec` 支持 `string` / `number` / `integer` / `boolean` / `null` / `array` / `object` / 作者专用 `json` / 恰好一个 `oneOf`。标量 `enum`/`const` 必须匹配节点类型；显式 object 节点必须声明 `additionalProperties: true | false`。

为什么限制这么严？因为 `output.schema` 要对**每个成功的规范值**做 JSON Schema 校验——DSL 的封闭性保证 schema 一定可执行、可校验、可投影。`packages/core/tools/src/json-schema.ts` 导出了 `assertSupportedJsonSchema` 和 `validateJsonSchemaValue`，在注册时和执行后分别校验。

## 六段守卫管线

`docs/tool-execution-pipeline.md` 给出了完整流程。核心顺序：

```
model 产出 tool-call block
  → 记 session 事件 tool/call（执行前先记日志！）
  → presentCall(args) 渲染 UI pending 卡
  → tools/pre-execute waterfall（hooks、permission、sandbox）
  → 注册的 monotonic guards（deny or abstain；identity protected）
  → ctx.approval 一次性审批（无应答=deny）
  → tools/execute waterfall（timeout、retry、metrics——around dispatch）
  → 注册的 execute() 本体
  → fs/write-intent 或 fs/edit-intent（仅 tool-fs 突变）
  → 工具自己的 session 事件
  → tools/post-execute waterfall（accept、block、replace、add context）
  → 注册表外层规范化（pipeline/result 快照 throw 变成 isError）
  → ToolDefinition.finalizeContent
  → tools/result 同步通知（冻结的权威结果）
  → 记 session 事件 tool/result
  → presentResult(args, result) 渲染 UI 完成卡
```

几个关键点：

1. **`tool/call` 在 execute 之前就记日志**——"logged before execution"。执行失败也有记录。
2. **三个 waterfall**（`tools/pre-execute`、`tools/execute`、`tools/post-execute`）让 policy/hooks/sandbox 全部在不动主循环的前提下插入。
3. **denied 的调用也走 post-execute**——拒绝不是无声消失，政策监听者能看到结果。
4. **`tools/result` 观察不可变、lossless-JSON 结果**——"This lets hooks span tool families without coupling the tools to one policy service"。

## Code Mode：`run_code` 与 SDK 渲染

`packages/core/tools/src/code-mode.ts` 导出了 `createRunCodeTool` 和 `RUN_CODE_NAME`。Code Mode 有两个变体（`DSH_TOOLS_MODE=native|code|both`）：

- **native 模式**：普通工具注册。
- **code 模式**：`run_code` 是注册表唯一的 wire 贡献，其他可见能力由加载 runtime 语言里的生成 SDK 段声明。

`packages/core/tools/src/index.ts` 里有一个 `CODE_ONLY_INSTRUCTION` 常量：

```ts
const CODE_ONLY_INSTRUCTION = `\`${RUN_CODE_NAME}\` is the only tool you can call directly — a tool call naming any other tool fails. Reach every tool the SDK declares below from inside the program.`

const COLLAPSE_SECTION_ORDER = 99
```

`COLLAPSE_SECTION_ORDER = 99`——在 persona（order 0）之后、per-tool guidance（100-199）之前，让模型先读到哪些工具可以调，再读每个工具的用途。

SDK 渲染器支持 TypeScript 和 Python：

```ts
const SDK_RENDERERS: Record<string, (schemas: ToolSdkSchema[]) => string> = {
  typescript: renderToolsSdk,
  python: renderToolsSdkPy,
} satisfies Record<CodeSdkLanguage, (schemas: ToolSdkSchema[]) => string>
```

## 并行工具调用：`isConcurrencySafe` 与有界滚动池

并行模型是"**有界滚动池**"：按 `executionMode` 分类，符合 `isConcurrencySafe` 的调用可以并发 execute，但 pre 按序、post 按模型顺序（"ordered pre, concurrent execute; ordered post"）。结果按模型顺序结算，保证日志顺序与模型预期一致。

## UI 呈现：presentCall / presentResult 的纯函数设计

工具的 UI 呈现意图是**设计的一部分**。`presentCall(args)` 渲染 pending 状态，`presentResult(args, result)` 渲染完成状态。两者都**必须纯且无副作用**——因为 UI 可能在 live streaming 期间**和** session-log 重放期间调用它们（"a UI may call it during live streaming AND a session-log replay, so it must depend only on args"）。

## 架构速览

```
工具执行管线（六段守卫）
┌─────────────────────────────────────────┐
│ 1. tool/call → session event (先记日志)  │
│ 2. tools/pre-execute waterfall          │
│ 3. monotonic guards (deny / abstain)    │
│ 4. ctx.approval 一次性审批               │
│ 5. tools/execute waterfall → execute()  │
│ 6. tools/post-execute waterfall         │
│    → normalize → finalizeContent        │
│    → tools/result → tool/result event   │
└─────────────────────────────────────────┘

ToolDefinition
├── ToolSchema (name, description, parameters)
├── output: ToolOutputDefinition (schema + render)
├── execute(args, exec) → canonical value
├── finalizeContent?(exec, result) → ContentBlock[]
├── timeoutMs? (协作式, 不发给模型)
└── isConcurrencySafe? (opt-in 并行)
```

## 关键设计决策

1. **工具只返回规范值**：呈现由管线负责。这保证了"模型可见即日志"——工具的输出必须经过管线的规范化和记录，才能到达模型。

2. **`tool/call` 在 execute 之前记日志**：执行失败也有记录。这让崩溃恢复能精确知道"模型请求了什么工具、参数是什么"，即使工具还没执行完。

3. **`finalizeContent` 必须 total，不许 throw**：它是管线的最后一环，任何异常都会破坏整个管线的封闭性。源码注释："must be total, never throw"。

4. **UI 呈现是纯函数**：live 和 replay 输出一致。这是"可重放"哲学在 UI 层的延伸——同一个纯函数，无论在实时流还是日志重放中调用，输出相同。

## 本章小结

- 工具系统是 dsh 工程哲学的缩影：**工具只返回规范值，呈现由管线负责**。
- 六段守卫管线让 policy/hooks/sandbox/approval 全部在不动主循环的前提下插入。
- `isConcurrencySafe` opt-in 并行，有界滚动池按模型顺序结算。
- Code Mode 下 `run_code` 是唯一 wire 贡献，子调用重入完整守卫工具管线。
- UI 呈现是纯函数，live 与 replay 一致。

## 阅读思考题

1. `tool/call` 在 execute 之前就记日志。如果一个工具的 `execute` 函数永远不返回（比如死循环），日志里会有什么？主循环会怎样？
2. `isConcurrencySafe` 只有 `true` 才 opt-in 并行。如果一个工具声明了 `isConcurrencySafe: true` 但实际上修改了共享状态，会引入什么问题？dsh 有什么机制能发现这种错误？
3. Code Mode 下 `run_code` 是唯一注册的工具。如果模型尝试调用其他工具，会发生什么？这种"collapse"设计的好处和代价是什么？

> **下一章预告**：第 9 章将讲 dsh 最值得抄的设计——Capability Seam 三件套：Service Definition / Service Provider / Consumer。
