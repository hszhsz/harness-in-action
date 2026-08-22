# 第 10 章 用户审批：fail-closed 的安全策略

上一章沙箱在 OS 层面做了最后一道检查。但对于文件写入、shell 执行等高风险操作，dsh 还有一个更上层的安全机制——用户审批。在 `packages/interaction/user-approval/src/` 中，审批系统实现了"默认拒绝"的哲学。

## 请求到达这里时发生了什么

当工具的守卫管线到达审批阶段时，如果工具声明了 `requiresApproval: true`，系统会暂停执行，等待用户做出决定。审批的结果只有两种：

```typescript
type ApprovalOutcome = 'approved' | 'denied'
```

在 headless 模式下，没有交互式终端，审批系统的行为取决于配置——可以通过预设策略自动批准或拒绝。

## 关键代码路径

### fail-closed 策略

审批系统的核心设计是 **fail-closed**：当审批界面不可用、超时、或出现任何异常时，默认行为是**拒绝**而非允许。

```typescript
async function requestApproval(request: ApprovalRequest): Promise<ApprovalOutcome> {
  try {
    const handler = ctx.get('approvalHandler')
    if (handler === undefined) {
      // 没有审批处理器 → 默认拒绝
      return 'denied'
    }
    return await handler(request)
  } catch (error) {
    // 任何异常 → 默认拒绝
    return 'denied'
  }
}
```

这个设计意味着：
- 在交互式终端（TUI）中，用户会看到审批提示
- 在 headless 模式下，如果没有配置自动审批策略，操作会被拒绝
- 如果审批处理器崩溃，操作会被拒绝
- 如果审批超时，操作会被拒绝

### ApprovalRequest：审批的上下文

```typescript
interface ApprovalRequest {
  tool: string           // 工具名
  description: string    // 操作描述
  params: unknown        // 工具参数
  risk: 'low' | 'medium' | 'high'  // 风险等级
}
```

风险等级由工具自己声明，帮助用户做出决策。`write_file` 通常是 `medium`，`shell` 执行任意命令是 `high`。

### 审批策略

在实际使用中，用户通常不想每个操作都手动审批。dsh 支持配置审批策略：

- **always-approve**：自动批准所有操作（仅限低风险环境）
- **always-deny**：自动拒绝所有操作
- **tool-specific**：按工具名配置（如自动批准 `read_file`，但要求审批 `write_file`）
- **interactive**：每次都问用户

这些策略通过 Cordis 插件注入，不同的 profile 可以有不同的默认策略。

### 审批与 Session 事件

审批过程会被记录到 Session 日志中：

```typescript
session.append('approval/asked', { tool, description, params })
const outcome = await requestApproval(request)
session.append('approval/decided', { tool, outcome })
```

这保证了完整的审计轨迹——我们可以回放一个 session 并看到每一次审批请求和决定。

## 调用链：从这里到下一步

审批通过后，`write_file` 操作实际执行。执行结果（包括 contextual diff）被封装为 `ToolResult`。回到 Agent Loop 的 step 循环，`tool/result` 事件被追加到 Session。

如果模型认为还需要更多操作（比如执行 `python hello.py`），它会返回另一个工具调用，step 循环继续。如果模型认为任务完成，它返回纯文本响应，turn 结束。

**下一步：请求将流向事件持久化——`packages/persistence/`。所有这些事件（turn/start、user/message、tool/result）如何被持久化到磁盘？JSONL 和 SQLite 后端有什么区别？**

## 关键设计决策

1. **fail-closed 而非 fail-open**：安全系统在不确定时应该拒绝而非允许。这可能会导致一些合法操作被拒绝，但避免了危险操作被误允许。

2. **审批事件记录到 Session**：完整的审计轨迹，支持回放和调试。

3. **策略可配置**：不同场景有不同的安全需求。headless 生产环境可能需要更严格的策略，开发环境可能更宽松。

4. **风险等级由工具声明**：工具最了解自己的风险。系统根据风险等级决定是否需要审批。

## 本章小结

- 审批系统采用 fail-closed 策略：无处理器、超时、异常都默认拒绝
- `ApprovalOutcome` 只有 `'approved'` 和 `'denied'` 两种
- 审批策略可配置：always-approve、always-deny、tool-specific、interactive
- 审批过程记录为 `approval/asked` 和 `approval/decided` 事件
- 风险等级由工具自己声明

## 阅读思考题

1. 为什么审批系统用 fail-closed 而不是 fail-open？举一个 fail-open 可能导致的安全问题。
2. `approval/asked` 和 `approval/decided` 事件是 `ignorable` 的还是必需的？为什么？
3. 如果一个工具调用在审批阶段被拒绝，Agent Loop 如何处理？模型会看到什么？

> **下一章预告**：`write_file` 执行完毕，`tool/result` 事件追加到了 Session。但这些事件只是在内存中——如果进程崩溃怎么办？下一章我们看事件持久化系统如何把事件安全地写入磁盘。
