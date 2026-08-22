# 第 14 章 Subagent：任务委派与多 Agent 协作

上一章 `python hello.py` 执行成功。在我们的简单例子中，任务已经完成。但在更复杂的场景中——比如"帮我写一个完整的 Web 应用"——一个 Agent 可能需要将子任务委派给其他 Agent。`packages/subagent/` 实现了这种多 Agent 协作模式。

## 请求到达这里时发生了什么

Subagent 系统允许一个 Agent（父 Agent）创建子 Agent 来处理子任务。子 Agent 拥有独立的 Session、独立的上下文窗口、但共享同一个工作区。

## 关键代码路径

### 子 Agent 的创建

当父 Agent 决定委派任务时，它调用 `agentLoop.createAgent()`：

```typescript
const handle = await ctx.agentLoop.createAgent(parentCtx, {
  sessionId: childSessionId,
  agentOptions: { provider: 'deepseek', model: 'deepseek-chat' },
  meta: { cwd: session.header.cwd },
  setup: async (agentCtx) => {
    // 配置子 Agent 的工具集、提示词等
  },
  signal: parentAbortSignal,
})
```

子 Agent 的 Session 通过 `SessionStore.fork()` 从父 Session 创建：

```typescript
fork(source, boundary?, childSessionId?): Session {
  const seed = this._forkSeed(liveSource, boundary)
  return this.create(childSessionId, {
    seed,
    meta: {
      cwd: liveSource.header.cwd,
      parentSession: liveSource.id,
      seedLength: seed.length,
    },
  })
}
```

`boundary` 参数决定了从父 Session 的哪个 seq 位置分叉。子 Session 的 `seedLength` 记录了继承了多少事件——这让 resume 时能区分"父的历史"和"子的工作"。

### 委派深度

`SessionHeader.delegationDepth` 记录了委派深度：

- 顶层 Session：`delegationDepth` 缺省（即 0）
- 子 Agent Session：`delegationDepth = 父.depth + 1`

这个深度被持久化，用于递归预算控制——防止无限嵌套的子 Agent。

### prepareContinuable

子 Agent 的一个关键特性是**可延续性**（continuable）——子 Session 可以被暂停、序列化、然后在另一个进程中恢复。`prepareContinuable` 函数准备了延续所需的所有资源：

```typescript
interface ContinuableState {
  sessionId: SessionId
  seed: SessionEvent[]
  header: SessionHeader
  // 可以安全序列化的状态
}
```

### 子 Agent 的工具集

子 Agent 通常拥有与父 Agent 不同的工具集。比如：
- 父 Agent 有 `delegate` 工具来创建子 Agent
- 子 Agent 有 `write_file`、`shell` 等执行工具
- 子 Agent 没有 `delegate` 工具（防止无限委派）

这通过 Cordis 的 `isolate` 机制实现——子 Agent 运行在独立的作用域中，只能看到被注入的工具。

### 结果收集

子 Agent 的结果通过以下方式传回父 Agent：

1. **Session fork**：子 Session 的事件包含了完整的执行历史
2. **工具结果**：`delegate` 工具的 `ToolResult` 包含子 Agent 的最终输出
3. **事件通知**：父 Agent 可以订阅子 Agent 的 `agent/status` 事件

## 调用链：从这里到下一步

在我们的简单例子中，没有子 Agent 的参与。`hello.py` 已经创建并执行成功，Agent Loop 的 turn 结束。

但在更复杂的场景中，如果任务涉及多个独立子任务的编排，模型可能会生成一个脚本来协调它们。

**下一步：请求将流向 Workflow 系统——`packages/workflow/`。当模型编写脚本来编排多个操作时，workflow 如何运行这些脚本、管理状态、处理错误？**

## 关键设计决策

1. **fork 而非 copy**：子 Session 从父 Session fork 而非创建全新 Session。这让子 Agent 能看到父的上下文历史。

2. **delegationDepth 持久化**：深度信息写入 SessionHeader，跨进程恢复后仍然有效。

3. **工具集隔离**：子 Agent 运行在独立的 Cordis 作用域中，只能看到被注入的工具。这防止了权限泄露。

4. **可延续性**：子 Session 可以被序列化和恢复，支持长时间运行的任务。

## 本章小结

- Subagent 允许父 Agent 委派子任务给子 Agent
- 子 Session 通过 `SessionStore.fork()` 从父 Session 创建
- `delegationDepth` 持久化在 SessionHeader 中，防止无限嵌套
- 子 Agent 的工具集通过 Cordis isolate 机制隔离
- 子 Session 支持可延续性（序列化和恢复）

## 阅读思考题

1. `fork` 的 `boundary` 参数如何决定子 Session 继承多少父历史？如果 `boundary` 是 undefined 会怎样？
2. 子 Agent 没有 `delegate` 工具，这如何防止无限委派？如果父 Agent 通过其他方式间接创建子 Agent 呢？
3. `prepareContinuable` 为什么需要与普通的 fork 分开？它们的使用场景有什么不同？

> **下一章预告**：在我们的例子中，模型直接完成了任务。但有时候模型会编写脚本来编排多个操作。下一章我们看 Workflow 系统——如何运行模型生成的编排脚本。
