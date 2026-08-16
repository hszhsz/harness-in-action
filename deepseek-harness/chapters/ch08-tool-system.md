# 第 8 章　工具系统：ToolDefinition 与守卫管线

> 工具系统是 Agent 的"手"。dsh 的 `ctx.tools` 是一个作用域工具注册表 + 守卫执行管线：模型只能看到 allowlist 投影出的 schema，任何工具调用都要穿过六段守卫。这一章拆开 `packages/core/tools`。

## 8.1　`ToolDefinition`：schema + execute + 纯投影三件套

一个注册的工具是 `ToolDefinition`：模型可见的 `ToolSchema` + 强制的规范输出声明 + `execute` 函数 + 可选的最终内容回调 + 可选 UI 呈现器。[[docs/subsystems/tools.md]](https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/subsystems/tools.md)

```ts
interface ToolDefinition extends ToolSchema {
  readonly output: ToolOutputDefinition   // JSON Schema + render + presentationMeta
  execute(args: unknown, exec: ToolRunContext): Promise<unknown>
  finalizeContent?(exec, result): ContentBlock[] | undefined
  timeoutMs?: number
  isConcurrencySafe?(args: unknown): boolean
  presentCall?(args: unknown): ToolCallView | undefined
  presentResult?(args: unknown, result: ToolResult): ToolResultView | undefined
}
```

关键设计：

- **`output` 是强制的规范输出声明**：`ToolOutputDefinition` 含 JSON Schema（`output.schema`，对每个成功的规范值强制校验）、纯投影 `render`（从校验过的参数和值投影到 Native/model content）、可选 `presentationMeta`（纯可重放呈现投影，只对顶层调用计算）。
- **`execute` 返回唯一规范值**："Run one accepted call and return only its canonical lossless-JSON value"。工具不能直接往模型对话里塞东西——它只能返回值，由管线决定如何呈现。这是"模型可见即日志"不变量在工具层的体现。
- **`finalizeContent` 是同步的最后一英里变换**：注册表在 execute 开始时快照此回调，对每个规范化结果（包括绕过 `tools/post-execute` 的管线失败）恰好调用一次，在 lossless 物化前立即执行。返回 `undefined` 保留内容。**必须 total，不许 throw**。
- **`timeoutMs` 是协作式超时预算**：由 `@deepseek-ai/dsh-tool-call-timeout-policy`（一个 `tools/execute` wrapper）强制执行，**绝不发给模型**——`schemas()` 只 whitelist name/description/parameters。声明它等于断言"本工具会把 `exec.signal` 转发给一个协作式实现"。
- **`isConcurrencySafe` 是纯同步分类器**：只有 `true` 才 opt-in 并行。opt-in 的执行**不得修改 parent-owned state**；共享状态必须容忍并发派发。这个元数据也绝不模型可见。

## 8.2　JSON-value schema DSL：一套词汇描述参数与输出

插件作者用一套词汇描述类型化参数和类型化输出值。`ValueSchemaSpec` 支持 `string` / `number` / `integer` / `boolean` / `null` / `array` / `object` / 作者专用 `json` / 恰好一个 `oneOf`。标量 `enum`/`const` 必须匹配节点类型；显式 object 节点必须声明 `additionalProperties: true | false`；参数定义是隐式开放 object property map，必填属性挂 `required: true`。

```ts
type ValueSchemaSpec =
  | StringValueSchemaSpec | NumberValueSchemaSpec | IntegerValueSchemaSpec
  | BooleanValueSchemaSpec | NullValueSchemaSpec | ArrayValueSchemaSpec
  | ObjectValueSchemaSpec | JsonValueSchemaSpec | OneOfValueSchemaSpec
```

为什么限制这么严？因为 `output.schema` 要对**每个成功的规范值**做 JSON Schema 校验——DSL 的封闭性保证 schema 一定可执行、可校验、可投影。

## 8.3　六段守卫管线：pre-execute → guards → execute → post-execute → finalizeContent → tools/result

`docs/tool-execution-pipeline.md` 给出了完整流程图（Mermaid）。核心顺序：

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
  → 工具自己的 session 事件（todo/write、fs/observed、hook/invoked、hook/result、tool/code-dispatch）
  → tools/post-execute waterfall（accept、block、replace、add context）
  → 注册表外层规范化（pipeline/result 快照 throw 变成 isError）
  → ToolDefinition.finalizeContent（最后的 content-only 不变量）
  → tools/result 同步通知（冻结的权威结果）
  → 记 session 事件 tool/result（单一模型可见结果）
  → presentResult(args, result) 渲染 UI 完成卡
```

几个关键点：

1. **`tool/call` 在 execute 之前就记日志**——"logged before execution"。执行失败也有记录。
2. **`tools/pre-execute` / `tools/execute` / `tools/post-execute` 是三个 waterfall**，可能转换调用；monotonic guards 在它们之后跑，deny 或 abstain，identity 受保护（owner policy 不能被重排序，必须注册为 guard）。
3. **`ctx.approval` 在 guards 之前解析**——先问人，再跑 owner 政策。
4. **denied 的调用也走 post-execute**——拒绝不是无声消失，政策监听者能看到结果。
5. **`tools/result` 观察不可变、lossless-JSON 结果**——"This lets hooks span tool families without coupling the tools to one policy service"。
6. **Code Mode** 把保留的 `run_code` transport 和序列化的子调用都送进这条管线——子调用携带 parent token、记 `tool/code-dispatch`、denial 作为 binding rejection 返回、省略 `additionalContexts` 以保持 call/result 相邻。

这条管线的设计目标：**policy、hooks、sandboxing、filesystem guards、result rewriting、final-outcome observation、UI rendering 全部在不动主循环的前提下挂上去**。

## 8.4　并行工具调用：`isConcurrencySafe` 与有界滚动池

`docs/agent-lifecycle.md` 的序列图展示了工具批次的执行模型：

```
Driver->>Tools: classify pending call by executionMode
loop barriers and bounded rolling pool, reclassify before start
  opt call starts
    Driver->>Session: tool/call
    Driver->>Tools: ordered pre, concurrent execute
  opt next model-order result ready
    Driver->>Tools: ordered post
    Driver->>Session: tool/result
```

并行模型是"**有界滚动池**"（bounded rolling pool）：按 executionMode 分类，符合 `isConcurrencySafe` 的调用可以并发 execute，但 pre 按序、post 按模型顺序（"ordered pre, concurrent execute; ordered post"）。结果按模型顺序结算，保证日志顺序与模型预期一致。

## 8.5　UI 呈现：presentCall / presentResult 的纯函数设计

工具的 UI 呈现意图是**设计的一部分**（`AGENTS.md`："A tool's UI render intent is part of its design, decided up front (generic/terminal/diff, locations)"）。`presentCall(args)` 渲染 pending 状态，`presentResult(args, result)` 渲染完成状态，都返回 `ToolCallView`/`ToolResultView`（`card`-tagged render intent）。

两者都**必须纯且无副作用**——因为 UI 可能在 live streaming 期间**和** session-log 重放期间调用它们（"a UI may call it during live streaming AND a session-log replay, so it must depend only on args"）。这是"可重放"哲学在 UI 层的延伸：同一个纯函数，live 和 replay 输出一致。

## 本章小结

dsh 的工具系统是它工程哲学的缩影：

1. **工具只返回规范值**，呈现由管线负责——模型可见即日志的不变量延伸；
2. **六段守卫管线**让 policy/hooks/sandbox/approval 全部在不动主循环的前提下插入；
3. **`isConcurrencySafe` opt-in 并行**，有界滚动池按模型顺序结算；
4. **UI 呈现是纯函数**，live 与 replay 一致；
5. **timeout/sandbox/approval 元数据绝不模型可见**——模型看到的永远是干净的 schema。

## 动手实验

1. 读 `packages/core/tools/src/index.ts`（1946 行），画出 `ToolDefinition` → `schemas()` → 执行管线的完整类型链。
2. 在 `docs/tool-catalog.md` 里挑一个工具（比如 `dsh-tool-fs` 的 `edit`），对照它的 schema 描述读 `packages/fs/tool-fs` 的实现。
3. 找 `tools/post-execute` 的监听者（spill-policy、timeout-policy 等），理解每个如何"不动循环地加政策"。
4. 读 `packages/core/tools/src/schema.ts` 的 `ValueSchemaSpec`，确认 object 节点必须声明 `additionalProperties`。
