# 第 13 章 Shell 执行：子进程管理与 I/O 流

上一章上下文压缩让模型可以继续工作。模型现在决定执行 `python hello.py` 来验证它写的文件。它返回了一个 `shell` 工具调用。请求进入了 `packages/shell/`——Shell 执行模块。

## 请求到达这里时发生了什么

Shell 工具是 dsh 中风险最高的工具之一——它可以执行任意命令。因此它的审批等级是 `high`，守卫管线会特别严格。

Shell 执行的核心是 `ShellExecRequest`：

```typescript
interface ShellExecRequest {
  command: string           // 要执行的命令
  cwd?: string             // 工作目录
  env?: Record<string, string>  // 环境变量
  timeout?: number         // 超时毫秒数
  background?: boolean     // 是否后台运行
}
```

## 关键代码路径

### 子进程创建

Shell 工具使用 Node.js 的 `child_process` 模块创建子进程：

```typescript
async function spawnProcess(request: ShellExecRequest): Promise<ProcessResult> {
  const child = spawn('bash', ['-c', request.command], {
    cwd: request.cwd ?? session.cwd,
    env: { ...process.env, ...request.env },
    stdio: ['pipe', 'pipe', 'pipe'],
  })
  // ...
}
```

使用 `bash -c` 而不是直接执行命令，保证了 shell 特性（管道、重定向、变量展开）的可用性。

### I/O 捕获

子进程的 stdout 和 stderr 被管道捕获：

```typescript
const stdout: string[] = []
const stderr: string[] = []

child.stdout.on('data', (chunk) => {
  stdout.push(chunk.toString())
  // 实时流式输出到 UI（如果有的话）
})

child.stderr.on('data', (chunk) => {
  stderr.push(chunk.toString())
})
```

输出被同时收集（用于结果）和流式转发（用于实时 UI 显示）。

### 超时处理

如果指定了 `timeout`，系统会在超时后终止子进程：

```typescript
if (request.timeout) {
  timer = setTimeout(() => {
    child.kill('SIGTERM')
    // 给 5 秒优雅关闭，然后 SIGKILL
    setTimeout(() => child.kill('SIGKILL'), 5000)
  }, request.timeout)
}
```

### 取消传播

Shell 执行支持通过 `AbortSignal` 取消。当 Agent 被取消时，信号传播到正在运行的子进程：

```typescript
signal.addEventListener('abort', () => {
  child.kill('SIGTERM')
})
```

### 沙箱集成

如第 9 章所述，如果启用了沙箱，命令会被包装：

```typescript
function wrapCommand(command: string, sandbox: Sandbox): SpawnArgs {
  switch (sandbox.mode) {
    case 'bwrap':
      return { executable: 'bwrap', args: ['--ro-bind', '/usr', '/usr', ...command] }
    case 'landlock':
      return { executable: command, landlockRules: sandbox.rules }
    case 'seatbelt':
      return { executable: command, seatbeltProfile: sandbox.profile }
    default:
      return { executable: 'bash', args: ['-c', command] }
  }
}
```

### 结果封装

子进程结束后，结果被封装为 `ToolResult`：

```typescript
{
  content: [{
    type: 'text',
    text: `Exit code: ${exitCode}\n\nStdout:\n${stdout.join('')}\n\nStderr:\n${stderr.join('')}`
  }],
  meta: {
    exitCode,
    stdout: stdout.join(''),
    stderr: stderr.join(''),
    duration: endTime - startTime,
  }
}
```

`meta` 携带了结构化的执行信息，供 UI 展示。

### 后台执行

如果 `background: true`，Shell 工具会立即返回一个进程 ID，命令在后台继续运行。Agent 可以后续查询进程状态。

## 调用链：从这里到下一步

`python hello.py` 执行完毕，结果（stdout 输出 "Hello, World!"，exit code 0）被封装为 `tool/result` 事件。回到 Agent Loop 的 step 循环，模型看到了执行结果。

如果模型认为任务完成，它会返回纯文本响应（比如"已创建 hello.py 并成功执行"），然后 turn 结束。

但在更复杂的场景中，任务可能需要委派给子 Agent（subagent）。

**下一步：请求将流向子 Agent 系统——`packages/subagent/`。当任务太复杂需要委派时，subagent 如何创建、通信、返回结果？**

## 关键设计决策

1. **bash -c 包装**：用 `bash -c` 而非直接 spawn，保证 shell 特性的完整性。

2. **双通道 I/O**：输出同时被收集和流式转发，兼顾了结果完整性和实时反馈。

3. **优雅关闭**：先 SIGTERM，等待 5 秒，再 SIGKILL。给子进程清理资源的机会。

4. **沙箱透明集成**：Shell 工具不需要知道沙箱的存在，`wrapCommand` 在 spawn 前处理。

## 本章小结

- Shell 工具使用 `bash -c` 执行命令，支持完整的 shell 特性
- I/O 被同时收集和流式转发
- 超时和取消通过 SIGTERM/SIGKILL 实现
- 沙箱在 spawn 前透明包装命令
- 结果包含 exit code、stdout、stderr 和执行时长

## 阅读思考题

1. 为什么用 `bash -c` 而不是直接 spawn 用户的命令？如果用户想用 zsh 或 fish 怎么办？
2. 后台执行的进程如何与 Agent 的生命周期关联？如果 Agent 被取消，后台进程会怎样？
3. Shell 工具的 `timeout` 和 Agent 的 `AbortSignal` 有什么关系？它们如何协作？

> **下一章预告**：`python hello.py` 执行成功了。但在更复杂的场景中，任务可能需要委派给子 Agent。下一章我们看 Subagent 系统——如何创建子 Agent、传递上下文、收集结果。
