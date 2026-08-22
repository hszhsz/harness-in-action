# 第 13 章　持久化与崩溃恢复：日志如何变耐久

第 5 章讲了内存中的事件日志，这一章讲它如何变耐久：`SessionPersistence` seam、JSONL/SQLite 两个后端、flush checkpoint、崩溃恢复（孤儿 turn 的 synthetic `interrupted` 封口）、spill 存储。读完后，你应该能理解为什么"不截断"是崩溃恢复的正确策略，以及 synthetic `interrupted` 如何保持日志的平衡性。

## `SessionPersistence` seam：JSONL 与 SQLite 两个后端

持久化是**另一个 capability seam**：`packages/session/session-persistence/src/index.ts` 定义了 `SessionPersistence` 抽象服务：

```ts
export abstract class SessionPersistence extends Service {
  constructor(ctx: Context) {
    super(ctx, 'sessionPersistence')
  }

  abstract locate(meta: SessionHeader): SessionLocation | undefined
  // ...
}
```

两个可互换后端实现同一契约：**没有平行的持久化事件类型**——持久化直接存现有的 `SessionEvent`。JSONL 后端把每个 session 存成一个 transcript 文件（`SessionLocation.kind === 'jsonl'` 返回绝对路径）；SQLite 后端所有 session 共享一个数据库（返回 `undefined`）。

`SessionHeader` 是**日志旁边的元数据**：

```ts
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

格式版本、cwd、lineage、seed boundary——存储关切，不是对话事件，不进 `SessionEventMap`，永不进 `deriveMessages()`。

## Flush checkpoint：`session/flush` 与有界写批处理

`session/event` 是**同步**通知；持久化插件把事件拷进 per-session 控制器，**不阻塞生产者**。然后：

> The first pending event starts a fixed batching window, and later events join without resetting its deadline. Expiry starts one durable batch; events admitted during that write receive their own deadline and form a follow-up batch.

第一个 pending 事件启动固定批处理窗口，后续事件加入而不重置截止时间。到期写一批；写入期间新到的事件获得自己的截止时间，形成后续批次。**`session/flush` 取消等待并排干到静止**——主循环在认领下一个普通 turn 前用它作为排序和错误观察检查点。

一个被拒绝的后台写入保留事件并暂停自动重试；新事件启动新窗口，显式 flush 立即重试并通过 `agent/error` 和 logger 报告失败——**绝不作为关闭 turn 之后的 session event**。

## 崩溃恢复：孤儿 turn 的 synthetic `interrupted` 封口

崩溃恢复是最精彩的设计。一个后端重载一个 mid-turn 崩溃的日志，发现一个没有 `turn/end` 的孤儿 `turn/start`。**它不截断**——一个 turn 可能巨大（多 step、大工具输出），那些事件在崩溃前已经持久追加了。相反：

> …it closes the orphaned turn with a synthetic `turn/end { reason: { kind: 'interrupted' } }`, keeping the interrupted execution balanced without changing any standalone events before or after it. `interrupted` is the one `TurnEndReason` no loop emits.

用一个人造的 `turn/end { reason: { kind: 'interrupted' } }` 封口，保持中断执行"平衡"（balanced），不改动前后任何独立事件。`interrupted` 是**唯一一个主循环永远不会自己发出的 `TurnEndReason`**——所以重放时可以精确区分"真实结束"和"崩溃恢复"。

修复只适用于冷 session。对于 live id，`SessionPersistence.load(id)` 等待权威内存快照持久化，只在平衡时返回；开放的 live turn 拒绝而不是接收 synthetic 中断边界。

## Spill 存储：超大工具输出溢出到文件

`ctx.spillStore` 把工具的**超大文本**持久化，返回模型面对的定位符 + 检索指引。`packages/spill/spill` 定义了：

```ts
interface SaveTextSpill {
  owner: SpillOwner        // 保存时命名空间（session id）
  source: SpillSource      // 工具与调用身份
  suggestedName: string    // 建议名，后端净化成单个安全路径段
  content: string          // 完整文本（UTF-8）
}
```

`spill-local` 是私有 session-scoped 文件；`spill-policy` 是 `tools/post-execute` 政策（Consumer）。**fork 继承已存在的定位符但不复制/重拥有产物**；fork 后产生的 spill 用子 session id。

消费者例子：`tool-fs-search` 的 capped 结果、超长工具输出。`SaveTextSpill` 返回不透明定位符 + 后端提供的检索提示 + 精确字节数。

## Settings / Credentials / Workspace：其他持久化域

- **Settings**（`settings/`）：`SettingsNamespace` 注册、分层解析（defaults → composition `base` → user document）、owner scopes、hot commits。
- **Credentials**（`credentials/`）：`CredentialRef` 引用（**永远不是值**）进配置、按操作解析、UI-safe `CredentialInfo`。源码注释："Managed credentials live separately in `.credentials.yaml`; a credential left in either `.env` remains a lower-priority fallback"。
- **Workspace**（`workspace/`）：`WorkspaceId` 是 generated uuid，**绝不是路径**（路径规范化会重写路径，引用锚必须稳定）。`realpathNormalize`（`fs.realpath`；尾斜杠、`..`、symlink 全部解析）是唯一唯一性准则。

## 架构速览

```
持久化与恢复体系
┌─────────────────────────────────────────────┐
│  SessionPersistence seam                    │
│  Definition: session-persistence            │
│  Provider: JSONL (每session一个文件)          │
│           SQLite (所有session共享一个DB)      │
├─────────────────────────────────────────────┤
│  写批处理                                     │
│  session/event → 控制器 (不阻塞)              │
│  固定窗口 → 到期写一批 → session/flush 排干   │
├─────────────────────────────────────────────┤
│  崩溃恢复                                     │
│  孤儿 turn/start → synthetic turn/end        │
│  reason: { kind: 'interrupted' }             │
│  不截断, 保持平衡, 主循环永不自己发出           │
├─────────────────────────────────────────────┤
│  Spill 存储                                   │
│  ctx.spillStore → 超大文本溢出到文件           │
│  返回不透明定位符 + 检索指引                   │
├─────────────────────────────────────────────┤
│  其他域                                       │
│  Settings: 分层解析                           │
│  Credentials: 只存引用, 不存值                 │
│  Workspace: uuid, 不是路径                    │
└─────────────────────────────────────────────┘
```

## 关键设计决策

1. **崩溃恢复不截断**：用 synthetic `interrupted` 封口而不是截断孤儿 turn。一个 turn 可能巨大（多 step、大工具输出），截断会丢失已持久化的有价值事件。`interrupted` 是唯一主循环不发的 `TurnEndReason`，重放时精确区分真实结束和崩溃恢复。

2. **有界写批处理**：第一个 pending 事件启动固定窗口，后续事件加入而不重置截止时间。这让写入吞吐量和延迟之间取得平衡——低流量时一个事件一批，高流量时多个事件一批。

3. **Credentials 只存引用**：`CredentialRef` 引用永远不是值本身。值在 `.credentials.yaml` 或 `.env` 里，通过引用解析。这避免了敏感值被记入 session 日志或配置文件。

4. **WorkspaceId 是 uuid 不是路径**：路径规范化会重写路径，引用锚必须稳定。uuid 不受路径规范化影响，是一个稳定的标识符。

## 本章小结

- 持久化是 seam：`SessionPersistence` 抽象 + JSONL/SQLite 两个后端，无平行事件类型。
- 有界写批处理：`session/flush` 作为主循环的排序/错误检查点。
- 崩溃恢复不截断：synthetic `interrupted` 封口保持平衡，是唯一主循环不发的终止原因。
- Spill 溢出：超大输出持久化 + 定位符，模型只见指引。
- 其他域：Settings 分层、Credentials 只存引用、Workspace 用 uuid 不用路径。

## 阅读思考题

1. `interrupted` 是 synthetic 的 `TurnEndReason`。如果一个 turn 在第一个 step 的 assistant/chunk 流式传输中途崩溃，恢复后的日志里会有哪些事件？`deriveMessages()` 如何处理这个不完整的 assistant 消息？
2. JSONL 后端每个 session 一个文件，SQLite 后端所有 session 共享一个数据库。在什么场景下你会选择 JSONL 而不是 SQLite？反过来呢？
3. `SessionHeader` 的 `seedLength` 字段记录了"通过 seed 继承的前导事件数量"。在 fork 场景下，这个字段如何帮助区分"父的历史"和"子的工作"？

> **下一章预告**：第 14 章将讲 dsh 的上下文组装、压缩与记忆——prompt sections 的按序拼接、compaction seam、以及 skills/goals/schedule。
