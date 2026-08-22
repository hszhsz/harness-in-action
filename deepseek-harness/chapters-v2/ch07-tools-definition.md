# 第 7 章 工具系统：ToolDefinition 与六段守卫管线

上一章 ReactLoopAgent 的 step 循环中，LLM 返回了一个工具调用请求——它想调用 `write_file` 来创建 `hello.py`。现在请求进入了 `packages/core/tools/src/` 的工具系统。工具系统是 dsh 最精密的子系统之一，它不仅执行工具，还要在执行前通过多层守卫验证。

## 请求到达这里时发生了什么

当 `agent.ts` 中的 step 循环检测到 LLM 返回了工具调用时，它调用 `executeToolCalls`。这个函数定义在 `packages/core/agent-loop/src/tool-calls.ts` 中，它：

1. 从 `ctx.tools` 获取已注册的工具定义
2. 对每个工具调用，走一遍守卫管线（guard pipeline）
3. 执行实际的工具操作
4. 将结果封装为 `tool/result` 事件

## 关键代码路径

### ToolDefinition：工具的声明

每个工具通过 `ToolDefinition` 注册到 `ctx.tools`（一个 Cordis Service）。一个工具定义包含：

```typescript
interface ToolDefinition {
  name: string                    // 工具名，如 'write_file'
  description: string             // 给模型看的描述
  parameters: JsonSchema          // JSON Schema 参数定义
  execute: (params, context) => Promise<ToolResult>  // 执行函数
  // 守卫相关
  guards?: Guard[]                // 守卫列表
  requiresApproval?: boolean      // 是否需要用户审批
}
```

`parameters` 是标准的 JSON Schema，LLM 用它来理解工具接受什么参数。`execute` 是实际执行逻辑。

### 六段守卫管线

工具调用不是直接执行的。它要经过一个精心设计的守卫管线：

**第一段：Schema 验证**
```typescript
validateSchema(tool.parameters, args)
```
检查 LLM 传入的参数是否符合工具的 JSON Schema。类型错误、缺失必填字段都在这里被捕获。

**第二段：参数规范化**
将原始 JSON 字符串解析为对象，处理类型转换和默认值。

**第三段：能力检查（Capability Seam）**
检查工具是否在当前上下文中可用。比如 `write_file` 需要文件系统能力，如果当前 Agent 没有被授予 FS 能力，这里会拒绝。

**第四段：沙箱检查**
如果启用了沙箱（bwrap/Landlock/Seatbelt），检查操作是否在沙箱允许的范围内。

**第五段：审批（Approval）**
某些工具（特别是文件写入和 shell 执行）需要用户审批。审批系统采用 fail-closed 策略——如果没有审批界面，默认拒绝。

**第六段：执行**
通过所有守卫后，调用工具的 `execute` 函数。

### executeToolCalls：并行执行

```typescript
export async function executeToolCalls(
  ctx: Context,
  session: Session,
  turn: number,
  step: number,
  toolCalls: ToolCall[],
  signal: AbortSignal,
): Promise<ToolResult[]>
```

工具调用可以并行执行——`maxParallelToolCalls` 配置控制最大并发数。每个工具调用都会：

1. 追加 `tool/call` 事件到 Session
2. 通过守卫管线
3. 执行
4. 追加 `tool/result` 事件到 Session

### ToolResult 与 meta

```typescript
interface ToolResult {
  content: ContentBlock[]     // 给模型看的结果
  error?: { name: string; code: string }  // 内部错误标识
  meta?: JsonValue            // 工具私有的展示元数据
}
```

`meta` 是一个重要的设计——它是工具产出的、面向 UI 的元数据。比如 `dsh-tool-fs` 在写入文件后，会在 `meta` 中携带上下文差异（contextual diff）。`meta` 对核心系统是不透明的，但必须是 JSON 可序列化的。

## 调用链：从这里到下一步

对于 `write_file` 工具调用，守卫管线通过后，请求进入文件系统能力提供者（Capability Seam）。

**下一步：请求将流向 FS 模块——`packages/fs/fs/src/` 和 `packages/fs/tool-fs/src/`。文件系统能力如何抽象？Sandbox 如何限制文件写入？**

## 关键设计决策

1. **守卫管线而非单一检查**：每个守卫阶段独立、可组合。添加新的安全检查只需要插入一个新的守卫，不需要修改执行逻辑。

2. **fail-closed 审批策略**：没有审批界面时默认拒绝，而非默认允许。在 AI agent 场景中，误允许比误拒绝的后果严重得多。

3. **meta 字段的不透明性**：核心系统不理解 meta 的结构，只保证它可序列化。这让工具可以自由携带展示信息而不需要修改核心类型。

4. **并行执行与 maxParallelToolCalls**：多个独立的工具调用可以并行执行，提高效率。但并发数有上限，防止资源耗尽。

## 本章小结

- 工具通过 `ToolDefinition` 注册，包含名称、描述、参数 Schema 和执行函数
- 六段守卫管线：Schema 验证 → 参数规范化 → 能力检查 → 沙箱检查 → 审批 → 执行
- 审批采用 fail-closed 策略
- `ToolResult` 的 `meta` 字段是工具私有的展示元数据
- 工具调用可以并行执行，受 `maxParallelToolCalls` 限制

## 阅读思考题

1. 如果一个工具的 `parameters` Schema 与 LLM 实际传入的参数不匹配，错误会在管线的哪个阶段被捕获？
2. 为什么 `meta` 必须是 JSON 可序列化的？如果允许非 JSON 值（如 Map、Date）会有什么问题？
3. 六段守卫管线中，哪一段是最"深"的安全边界？为什么？

> **下一章预告**：`write_file` 工具通过了守卫管线，现在请求到达了文件系统层。FS 如何抽象文件操作？`FsTarget` 是什么？为什么文件系统需要一个专门的能力接口？
