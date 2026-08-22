# 第 15 章 Workflow：脚本编排与代码运行时

上一章我们看了 Subagent 如何委派任务。另一种常见的模式是模型直接编写脚本来编排多个操作——比如"先读取文件 A，然后根据内容修改文件 B，最后运行测试"。`packages/workflow/` 实现了这种脚本编排能力。

## 请求到达这里时发生了什么

Workflow 系统允许模型生成一段脚本（通常是 TypeScript/JavaScript），由 dsh 在受控环境中执行。这不是简单的 `eval`——它是一个有状态管理、错误处理、和可观测性的运行时。

## 关键代码路径

### Workflow Seam

Workflow 通过"接缝"（seam）模式接入工具系统。一个 workflow 工具（如 `run_workflow`）接受模型生成的脚本，在受控运行时中执行它。

```typescript
interface WorkflowRunRequest {
  script: string              // 脚本内容
  language: 'typescript' | 'javascript'
  timeout?: number
  context?: Record<string, unknown>  // 传给脚本的上下文
}
```

### 代码运行时

脚本在受限的 Node.js VM 中执行：

```typescript
const vm = require('node:vm')
const context = vm.createContext({
  // 注入的 API
  readFile: (path) => fs.readFile(path, 'utf8'),
  writeFile: (path, content) => fs.writeFile(path, content),
  exec: (cmd) => execInSandbox(cmd),
  log: (...args) => captureOutput(args),
  // 受限的全局对象
  console: { log: captureLog, error: captureError },
})

const result = vm.runInContext(script, context, {
  timeout: request.timeout ?? 30000,
})
```

关键限制：
- **没有 `require`**：脚本不能导入任意模块
- **没有 `process`**：脚本不能访问进程
- **超时强制**：`timeout` 参数保证脚本不能无限运行
- **注入的 API**：只能通过注入的 `readFile`、`writeFile`、`exec` 等函数操作外部世界

### 状态管理

Workflow 运行时维护执行状态：

```typescript
interface WorkflowState {
  status: 'running' | 'completed' | 'failed' | 'cancelled'
  steps: WorkflowStep[]
  output: unknown
  error?: Error
}

interface WorkflowStep {
  name: string
  status: 'pending' | 'running' | 'completed' | 'failed'
  result?: unknown
  duration?: number
}
```

每个步骤的状态和结果都被记录，让 UI 可以展示执行进度。

### 与 Agent Loop 的集成

Workflow 执行是 Agent Loop 的一个工具调用。当模型返回 `run_workflow` 工具调用时：

1. 工具系统通过守卫管线（包括审批）
2. Workflow 运行时执行脚本
3. 脚本的结果被封装为 `ToolResult`
4. `tool/result` 事件追加到 Session

### 错误处理

脚本执行可能失败。Workflow 运行时捕获所有错误并封装：

```typescript
try {
  const result = vm.runInContext(script, context, { timeout })
  return { content: [{ type: 'text', text: JSON.stringify(result) }] }
} catch (error) {
  return {
    content: [{ type: 'text', text: `Workflow failed: ${error.message}` }],
    error: { name: error.name, code: 'WORKFLOW_ERROR' },
  }
}
```

### tool-workflow 事件

Workflow 的执行过程会产生专门的 Session 事件：

```
tool-workflow/run-start   → Workflow 开始执行
tool-workflow/agent-start → 脚本中的子任务开始
tool-workflow/agent-end   → 脚本中的子任务结束
tool-workflow/run-end     → Workflow 执行完毕
```

这些事件是 `ignorable` 的——旧版本的 dsh 可以安全地跳过它们。

## 调用链：从这里到下一步

在我们的简单例子中，没有 Workflow 的参与。模型直接用 `write_file` 和 `shell` 完成了任务。

但在分布式场景中，dsh 可能运行在远程服务器上，通过 Web UI 与用户交互。

**下一步：请求将流向远程调用与可观测系统——`packages/api/`。dsh 的 Typert RPC 协议如何工作？Web UI 如何实时观察 Agent 的行为？**

## 关键设计决策

1. **VM 隔离而非 eval**：用 `node:vm` 创建受限上下文，脚本不能访问任意 Node API。

2. **注入式 API**：脚本只能通过注入的函数操作外部世界，这些函数本身经过安全检查。

3. **超时强制**：防止恶意或错误的脚本消耗无限资源。

4. **ignorable 事件**：workflow 事件标记为可忽略，保证前向兼容性。

## 本章小结

- Workflow 允许模型生成脚本来编排多个操作
- 脚本在受限的 `node:vm` 上下文中执行，有超时限制
- 注入式 API（readFile、writeFile、exec）控制脚本的能力
- 执行状态和步骤被记录，支持 UI 展示
- workflow 事件标记为 ignorable，保证前向兼容

## 阅读思考题

1. 为什么用 `node:vm` 而不是在子进程中执行脚本？两者的安全模型有什么不同？
2. 如果脚本试图访问 `process.env` 或 `require('fs')`，会发生什么？
3. Workflow 的 `timeout` 和 Shell 的 `timeout` 有什么不同的实现机制？

> **下一章预告**：dsh 不仅可以在本地运行，还可以通过远程 API 服务。下一章我们看 Typert RPC 协议和 Web UI 的可观测性——如何从另一个进程或浏览器实时观察 Agent 的行为。
