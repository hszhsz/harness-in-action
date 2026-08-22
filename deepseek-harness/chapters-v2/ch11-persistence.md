# 第 11 章 事件持久化：JSONL 与 SQLite 的存储引擎

上一章 `write_file` 执行完毕，`tool/result` 事件追加到了 Session。但这些事件只是在内存中的数组里——如果进程崩溃，所有对话历史都会丢失。`packages/persistence/` 解决了这个问题：它将事件安全地持久化到磁盘。

## 请求到达这里时发生了什么

持久化系统通过 Cordis 的事件机制工作。当 `Session.append` 被调用时，它同步通知所有通过 `session/event` 注册的观察者。持久化插件是这些观察者之一——它接收每个新事件并将其缓冲到内存中，等待 flush 时机写入磁盘。

## 关键代码路径

### 两种后端

dsh 提供两种持久化后端：

**JSONL 后端**：每行一个 JSON 对象，追加写入
```
{"type":"turn/start","seq":0,"time":1692691200000,"data":{"turn":1}}
{"type":"user/message","seq":1,"time":1692691201000,"data":{...}}
{"type":"tool/call","seq":2,"time":1692691202000,"data":{...}}
```

**SQLite 后端**：结构化存储，支持查询

JSONL 的优势是简单和追加友好——写入只需一次 `appendFile` 调用，不需要事务。SQLite 的优势是支持查询（如"找到所有包含某个工具调用的 session"）。

### Session 头与事件分离

持久化存储包含两部分：
- **Session Header**：存储在单独的文件/表中，包含 `version`、`id`、`createdAt`、`cwd` 等元数据
- **事件序列**：存储在主文件/表中，按 `seq` 顺序排列

这种分离是有意的——Header 是存储关注点，不是可重放的对话状态。

### Flush 策略

事件不是每追加一个就写入磁盘。持久化插件维护一个缓冲区，在以下时机 flush：

1. **turn 边界**：一个 turn 结束时
2. **session/flush 事件**：显式请求刷新
3. **空闲时**：Agent 没有活跃工作时
4. **关闭时**：进程退出前

`SessionStore.flush(session)` 方法触发 `session/flush` 事件：

```typescript
async flush(session: Session): Promise<boolean> {
  const { carrier } = this.liveEntryFor(session)
  const callbacks = collectSessionCallbacks(this.ctx, [carrier, 'session/flush', session])
  const results = await Promise.allSettled(callbacks.map(callback => callback(...)))
  const failure = results.find(result => result.status === 'rejected')
  if (failure !== undefined) throw failure.reason
  return callbacks.length > 0
}
```

`session/flush` 用 `parallel` 模式分发——所有持久化后端并行刷新。

### 加载与恢复

当一个 session 被恢复时，持久化后端负责：

1. 读取 Session Header，验证版本
2. 读取所有事件，按 `seq` 顺序
3. 验证事件信封格式（`assertSessionEventEnvelope`）
4. 验证消息结构（`assertMessageEventShape`）
5. 通过 `Session.fromRestore` 创建 Session，转移所有权

```typescript
static fromRestore(id: SessionId, seed: readonly SessionEvent[], header: SessionHeader): Session {
  return new Session(id, seed, header, 'restore')
}
```

`restore` 模式与 `snapshot` 模式不同——它直接冻结接收到的对象（`freezeRestoredObject`），而不是重新序列化。这是性能优化，也是所有权转移的体现。

### 检查点与崩溃恢复

持久化系统需要处理进程崩溃的情况。JSONL 后端通过以下机制保证一致性：

- 每行是一个完整的 JSON 对象
- 最后一行可能不完整（崩溃时正在写入）→ 加载时忽略不能解析的尾行
- Header 中的 `seedLength` 记录了 fork 时继承了多少事件

### chunk 打包

`assistant/chunk` 事件逐 token 记录，数量可能很大。持久化系统可以选择将连续的 chunk 事件打包（`packChunkRuns`），减少存储空间。

## 调用链：从这里到下一步

事件被持久化后，我们的请求继续在 Agent Loop 中流转。如果对话变长（很多轮工具调用），上下文可能会超过模型的上下文窗口。

**下一步：请求将流向上下文压缩——`packages/core/system-prompt/src/` 的 compaction 机制。当对话变长时，如何压缩历史以适应模型的上下文窗口？**

## 关键设计决策

1. **Header 与事件分离**：元数据（Header）和对话历史（事件）存储在不同的位置，各司其职。

2. **缓冲而非同步写入**：每个事件追加到内存缓冲区，定期 flush。这平衡了性能和持久性。

3. **restore 模式的所有权转移**：从持久化加载的对象直接冻结使用，不重新序列化。这避免了不必要的深拷贝。

4. **并行 flush**：多个持久化后端可以同时工作（如同时写 JSONL 和 SQLite），互不干扰。

## 本章小结

- 持久化通过 `session/event` 和 `session/flush` 事件机制工作
- 两种后端：JSONL（追加友好）和 SQLite（支持查询）
- Header 与事件分离存储
- Flush 在 turn 边界、session/flush、空闲时、关闭时触发
- restore 模式转移对象所有权，避免深拷贝

## 阅读思考题

1. 为什么 `session/flush` 用 `parallel` 而不是 `serial` 模式？如果一个后端失败了会怎样？
2. JSONL 后端如何处理进程崩溃时的半写行？这个机制有什么局限性？
3. `packChunkRuns` 打包 chunk 事件后，重放时如何恢复原始的逐 token 流？

> **下一章预告**：事件被安全地持久化了。但随着对话变长，模型的上下文窗口可能装不下所有历史。下一章我们看上下文压缩——如何在保留关键信息的同时缩减对话长度。
