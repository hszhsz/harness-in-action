# 第 5 章 Session：事件溯源的对话容器

上一章 Loader 启动完成，插件树稳定了。用户输入的 `"帮我写一个 hello.py"` 通过 `provideCmdline` 注入到了树中。现在，headless profile 的应用插件会读取这个命令行参数，创建一个 Session，然后把用户的输入变成第一条 `user/message` 事件。让我们看看 `packages/core/session/src/` 中发生了什么。

## 请求到达这里时发生了什么

Session 是 DeepSeek Harness 的核心数据结构——一个**事件溯源的、只追加的日志**。它不存储"当前对话状态"，而是存储一系列不可变的事件。对话状态从事件日志中派生。

在 `packages/core/session/src/types.ts` 中，`SessionEventMap` 定义了所有可能的事件类型：

```typescript
export interface SessionEventMap {
  'turn/start': { turn: number }
  'turn/end': { turn: number; reason: TurnEndReason }
  'step/start': { turn: number; step: number }
  'step/end': { turn: number; step: number }
  'user/message': UserMessage
  'assistant/chunk': { turn: number; step: number; chunk: StreamChunk }
  'assistant/message': { turn: number; step: number; message: AssistantMessage; usage?: TokenUsage }
  'tool/call': { turn: number; step: number; callId: CallId; name: string; arguments: string }
  'tool/result': { turn: number; step: number; message: ToolResultMessage; error?: ...; meta?: JsonValue }
  'request/header': { header: EpochHeader; reason: RequestHeaderReason }
  'request/context': RequestContext
  'session/end-seed': Record<string, never>
  // ... 更多
}
```

对于我们的请求，第一个被追加的事件是 `turn/start`，然后是 `user/message`。

## 关键代码路径

### Session 类：只追加的日志

`Session` 在 `packages/core/session/src/index.ts` 中定义。它的核心是一个私有数组：

```typescript
export class Session {
  private log: SessionEvent[] = []
  private readonly surfaceManager = new SurfaceManager(this.log)
  // ...
}
```

`append` 方法是写入日志的唯一入口：

```typescript
append<T extends SessionEventType>(
  type: T,
  data: SessionEventMap[T],
  ...opts: T extends SurfaceEventType ? [opts: SurfaceIntent] : []
): SessionEvent<T> {
```

它做了以下事情：

1. **快照数据**：`snapshotJsonValue(data)` 对数据做一次无损 JSON 序列化+反序列化，确保数据是纯 JSON 且与调用者解耦
2. **验证**：检查事件信封格式、请求头格式、消息结构
3. **Surface 验证**：`this.surfaceManager.validateNext(event)` 检查 surface 元数据
4. **冻结**：`deepFreeze` 使事件完全不可变
5. **追加到日志**：`this.log.push(event)`
6. **通知观察者**：通过 `session/event` 事件通知持久化插件

### 事件信封

每个 `SessionEvent` 都有标准信封：

```typescript
{
  type: K,        // 事件类型
  seq: number,    // 在日志中的序号（从 0 开始，连续递增）
  time: number,   // Unix 毫秒时间戳
  data: SessionEventMap[K],  // 事件负载
  ignorable?: true,  // 是否可忽略（用于前向兼容）
}
```

`seq` 的连续性是整个系统的基石——它保证日志可以可靠地序列化、传输和重建。

### Surface 事件与普通事件

并非所有事件都产生模型可见的消息。`SurfaceEventType` 只有三种：

```typescript
export type SurfaceEventType =
  | 'user/message'
  | 'assistant/message'
  | 'tool/result'
```

这三种事件携带 `surfaceOp` 元数据，决定它们如何进入有序的对话表面：
- `'append'`：追加到末尾
- `{ op: 'replace', start, end }`：替换指定范围的节点（用于 compaction）

其他事件（`turn/start`、`step/end`、`assistant/chunk`）是"日志专用"——它们不产生模型消息，但记录了对话的结构。

### SessionStore：内存中的 Session 管理器

`SessionStore` 是一个 Cordis Service，注册为 `ctx.sessions`。它管理所有活跃的 Session：

```typescript
export class SessionStore extends Service {
  private store = new Map<SessionId, SessionEntry>()
  // ...
}
```

创建 Session 的标准路径是 `ctx.sessions.create(id, options)`，它：

1. 调用 `prepare` 构造 Session 对象
2. 通过 `ctx.effect` 注册一个效果（effect），在其中调用 `enter` 安装发布钩子
3. 调用 `announce` 发射 `session/created` 事件

`enter` 方法安装了模块私有的发布钩子——这就是为什么 `Session.append` 内部能通过 `attachments` WeakMap 找到 store entry 并通知观察者。

### SessionHeader：不可变的存储元数据

```typescript
export interface SessionHeader {
  readonly version: number
  readonly id: SessionId
  readonly createdAt: number
  readonly cwd?: string
  readonly parentSession?: SessionId
  readonly seedLength?: number
  readonly origin?: 'subagent'
  readonly delegationDepth?: number
  readonly agentPreset?: string
}
```

Header 在事件日志之外——它是存储关注点，不是可重放的对话状态。`version` 固定为 `0`（未发布阶段），用于持久化后端在加载时做版本检查。

## 调用链：从这里到下一步

当 headless 应用插件收到命令行参数后，它会：

1. 调用 `ctx.sessions.create(sessionId, { meta: { cwd } })` 创建 Session
2. Session 的 `append` 被调用两次：先 `'turn/start'`，再 `'user/message'`
3. 然后创建一个 Agent（ReactLoopAgent），它会接管 Session 的驱动

**下一步：请求将流向 Agent Loop——`ReactLoopAgent` 如何接管 Session，开始一个"turn"，调用 LLM，处理工具调用。**

## 关键设计决策

1. **事件溯源而非状态存储**：不存储"当前对话"，而是存储事件序列。好处是：完美重放、支持 fork（从某个 seq 分叉）、支持 compaction（用 replace 替换旧节点）。

2. **append 是唯一的写入口**：所有验证、快照、冻结都在 append 内部完成。外部代码无法绕过这些保护。

3. **seq 必须连续**：`seq = log.length`。这个不变量让持久化后端可以简单地检查连续性来检测损坏。

4. **Surface 与日志分离**：Surface 是从日志派生的有序视图，只包含消息产生的事件。这避免了在构建 LLM 请求时过滤非消息事件。

5. **深冻结（deepFreeze）**：一旦事件进入日志，它就是完全不可变的。这消除了"谁在什么时候修改了什么"的竞态问题。

## 本章小结

- Session 是事件溯源的只追加日志，`append` 是唯一的写入口
- 事件有标准信封：type、seq（连续）、time、data
- SurfaceEventType（user/message、assistant/message、tool/result）携带 surfaceOp 决定对话表面的位置
- SessionStore 管理活跃 Session，通过 WeakMap 的发布钩子连接 Session 的 append 与事件通知
- SessionHeader 是不可变的存储元数据，存在于事件日志之外

## 阅读思考题

1. 为什么 `snapshotJsonValue` 要做一次序列化+反序列化，而不是直接 `Object.freeze`？这防御了什么攻击场景？
2. `surfaceOp` 的 `replace` 操作在什么场景下使用？为什么需要 `sourceEventSeqs` 字段？
3. `session/end-seed` 事件的作用是什么？为什么它是空的 Record？

> **下一章预告**：Session 创建好了，`user/message` 事件已经追加。现在 `ReactLoopAgent` 要接管——它会打开一个 turn，组装 prompt，调用 LLM。下一章我们进入 Agent Loop 的状态机。
