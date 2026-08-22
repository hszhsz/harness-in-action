# 第 12 章 上下文压缩：当对话变长时

上一章事件被安全地持久化了。但在真实的 Agent 交互中，一个复杂任务可能产生数十轮工具调用，对话历史很快就会超过模型的上下文窗口。`packages/core/system-prompt/src/` 中的压缩（compaction）机制就是为了解决这个问题。

## 请求到达这里时发生了什么

上下文压缩是 Agent Loop 在构建 LLM 请求时的一个关键环节。在 `ReactLoopAgent.buildRequest` 中，系统提示词被组装并传给模型。当对话历史太长时，压缩机制会用摘要替换旧的对话片段。

## 关键代码路径

### 系统提示词的组装

`systemPrompt.assemble()` 方法收集所有注册的 prompt section，按 `order` 排序后拼接：

```typescript
const assembly = await this.loopCtx.systemPrompt.assemble(assembleContextFor(this, signal))
const sections = renderContextSections(assembly)
```

一个 `PromptAssembly` 包含多个 section，每个 section 有：
- `name`：标识符
- `order`：排序权重
- `text`：文本内容

dsh 的系统提示词由多个 section 组成：harness 身份、agent 预设、工具说明、当前工作目录等。

### RuntimeContextProjection

`RuntimeContextProjection` 在 `packages/core/agent-loop/src/runtime-context.ts` 中定义，它负责将运行时上下文注入到对话中：

```typescript
class RuntimeContextProjection {
  project(sections: string, contextSections: ContextSection[]): UserMessage | undefined {
    // 将系统提示词的 sections 投射为一条 user/message
  }
}
```

当系统提示词变化时（比如用户切换了工作目录），它会生成一条新的上下文消息注入到对话中。

### 压缩的触发

压缩在以下情况被触发：

1. **Token 计数超限**：当对话的总 token 数接近模型的上下文窗口
2. **显式请求**：用户或插件通过 `compaction/start` 事件请求压缩
3. **Agent Loop 内部**：在构建请求时检测到历史太长

### Surface 的 replace 操作

压缩的核心机制是 Surface 的 `replace` 操作。回忆第 5 章，`SurfaceOp` 可以是：

```typescript
type SurfaceOp =
  | 'append'
  | { op: 'replace'; start: number; end: number }
```

压缩时，系统会：

1. 选择要压缩的旧消息范围（比如前 20 条消息）
2. 用 LLM 生成这些消息的摘要
3. 追加一个 `compaction/summary` 事件，携带摘要文本
4. 这个事件的 `surfaceOp` 是 `{ op: 'replace', start: 5, end: 24 }`——替换第 5 到 24 条表面节点

这样，`deriveMessages()` 方法在计算模型可见的历史时，会跳过被替换的节点，只看到摘要。

### 压缩事件序列

一次完整的压缩产生以下事件：

```
compaction/start    → 标记压缩开始
compaction/prune    → 被修剪的节点信息
compaction/summary  → 摘要内容（带 surfaceOp: replace）
compaction/end      → 标记压缩结束
```

这些事件都是日志专用的（不在 Surface 上），但 `compaction/summary` 携带的 `surfaceOp` 会影响 Surface 的结构。

### deriveMessages 的缓存机制

`Session.deriveMessages()` 维护一个增量缓存：

```typescript
deriveMessages(): Message[] {
  const surface = this.surface
  const nodes = surface.nodes
  const generation = surface.replaceGeneration
  
  if (generation !== this.derivedGeneration) {
    this.derived = []
    this.derivedNodes = 0
    this.derivedGeneration = generation
  }
  
  for (const seq of nodes.slice(this.derivedNodes)) {
    const msg = this.deriveEventMessage(this.log[seq]!)
    if (msg) this.derived.push(msg)
  }
  this.derivedNodes = nodes.length
  return [...this.derived]
}
```

当 Surface 发生 `replace` 时，`replaceGeneration` 递增，缓存被清空重建。这保证了压缩后的历史是正确的。

## 调用链：从这里到下一步

压缩完成后，对话历史缩短了，模型可以继续处理。如果模型决定还需要调用更多工具（比如执行 `python hello.py`），step 循环继续。

**下一步：请求将流向 Shell 执行——`packages/shell/`。当模型决定执行 `python hello.py` 时，shell 工具如何创建子进程、管理输出、处理超时？**

## 关键设计决策

1. **Surface replace 而非删除**：压缩不是删除旧事件，而是用摘要替换表面节点。原始事件保留在日志中，支持完整重放。

2. **增量缓存**：`deriveMessages` 只投影新节点，压缩触发时才清空重建。这优化了正常情况下的性能。

3. **压缩事件的可审计性**：`compaction/start`、`compaction/prune`、`compaction/summary`、`compaction/end` 形成完整的审计轨迹。

4. **generation 计数器**：`replaceGeneration` 用一个简单的计数器检测 Surface 是否被修改过，避免了复杂的脏检查。

## 本章小结

- 上下文压缩在对话超过模型窗口时触发
- 压缩使用 Surface 的 `replace` 操作，用摘要替换旧消息
- 原始事件保留在日志中，Surface 只是跳过被替换的节点
- 压缩产生 `compaction/start`、`compaction/prune`、`compaction/summary`、`compaction/end` 事件
- `deriveMessages` 用 `replaceGeneration` 做增量缓存

## 阅读思考题

1. 为什么压缩保留原始事件而不删除它们？这与事件溯源的哲学有什么关系？
2. 如果压缩的摘要本身也很长（比如被压缩的消息包含大量代码），会发生什么？
3. `replace` 操作的 `sourceEventSeqs` 字段在压缩场景中包含什么？

> **下一章预告**：对话历史被压缩到合适的长度了。如果模型决定执行 `python hello.py`，它会调用 `shell` 工具。下一章我们看 Shell 执行如何创建子进程、管理 I/O、处理超时和取消。
