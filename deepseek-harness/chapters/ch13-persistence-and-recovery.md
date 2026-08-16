# 第 13 章　持久化与崩溃恢复：日志如何变耐久

> 第 5 章讲了内存中的事件日志，这一章讲它如何变耐久：`SessionPersistence` seam、JSONL/SQLite 两个后端、flush checkpoint、崩溃恢复（孤儿 turn 的 synthetic `interrupted` 封口）、spill 存储，以及其他持久化域。

## 13.1　`SessionPersistence` seam：JSONL 与 SQLite 两个后端

持久化是**另一个 capability seam**：一个抽象服务（`dsh-session-persistence`，`ctx.sessionPersistence`）+ 两个可互换后端实现同一契约。[[docs/subsystems/persistence.md]](https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/subsystems/persistence.md)

> …no parallel persisted event type—and two interchangeable backends implementing the same contract.

**没有平行的持久化事件类型**——持久化直接存现有的 `SessionEvent`。JSONL 后端把每个 session 存成一个 transcript 文件（`SessionLocation.kind === 'jsonl'` 返回绝对路径）；SQLite 后端所有 session 共享一个数据库（返回 `undefined`，因为"the sessions share one database"）。

`SessionHeader` 是**日志旁边的元数据**：格式版本、cwd、lineage、seed boundary——存储关切，不是对话事件，不进 `SessionEventMap`，永不进 `deriveMessages()`。

## 13.2　Flush checkpoint：`session/flush` 与有界写批处理

`session/event` 是**同步**通知；持久化插件把事件拷进 per-session 控制器，**不阻塞生产者**。然后：

> The first pending event starts a fixed batching window, and later events join without resetting its deadline. Expiry starts one durable batch; events admitted during that write receive their own deadline and form a follow-up batch.

第一个 pending 事件启动固定批处理窗口，后续事件加入而不重置截止时间。到期写一批；写入期间新到的事件获得自己的截止时间，形成后续批次。**`session/flush` 取消等待并排干到静止**——主循环在认领下一个普通 turn 前用它作为排序和错误观察检查点（"the loop still uses it as the ordering and error-observation checkpoint before claiming the next ordinary turn"）。

一个被拒绝的后台写入保留事件并暂停自动重试；新事件启动新窗口，显式 flush 立即重试并通过 `agent/error` 和 logger 报告失败——**绝不作为关闭 turn 之后的 session event**。配置的最大值只限制有意的批处理等待，不限制事件循环调度或后端持久化延迟（决策：`2026-08-08-bounded-session-persistence-write-batching.md`）。

## 13.3　崩溃恢复：孤儿 turn 的 synthetic `interrupted` 封口

崩溃恢复是最精彩的设计。一个后端重载一个 mid-turn 崩溃的日志，发现一个没有 `turn/end` 的孤儿 `turn/start`。**它不截断**——一个 turn 可能巨大（多 step、大工具输出），那些事件在崩溃前已经持久追加了。相反：

> …it closes the orphaned turn with a synthetic `turn/end { reason: { kind: 'interrupted' } }`, keeping the interrupted execution balanced without changing any standalone events before or after it. `interrupted` is the one `TurnEndReason` no loop emits.

用一个人造的 `turn/end { reason: { kind: 'interrupted' } }` 封口，保持中断执行"平衡"（balanced），不改动前后任何独立事件。`interrupted` 是**唯一一个主循环永远不会自己发出的 `TurnEndReason`**——所以重放时可以精确区分"真实结束"和"崩溃恢复"。

修复只适用于冷 session。对于 live id，`SessionPersistence.load(id)` 等待权威内存快照持久化，只在平衡时返回；开放的 live turn 拒绝而不是接收 synthetic 中断边界。`inspect(id)` 构建不可变逻辑 Session 而不发布或写恢复；协调器后端在受限 LRU 中保留精确的冷未发布 Session，重复历史读取和后续 `prepare(id)` 共享一次读/解压/校验/冻结/Session 构建。

## 13.4　Spill 存储：超大工具输出溢出到文件

`ctx.spillStore` 把工具的**超大文本**持久化，返回模型面对的定位符 + 检索指引。[[docs/subsystems/spill.md]](https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/subsystems/spill.md)

```ts
interface SaveTextSpill {
  owner: SpillOwner        // 保存时命名空间（session id）
  source: SpillSource      // 工具与调用身份（命名/检查用，非访问控制）
  suggestedName: string    // 建议名，后端净化成单个安全路径段——是提示，绝不是路径
  content: string          // 完整文本（UTF-8）
}
```

`spill-local` 是私有 session-scoped 文件；`spill-policy` 是 `tools/post-execute` 政策（Consumer）。**fork 继承已存在的定位符但不复制/重拥有产物**；fork 后产生的 spill 用子 session id。

消费者例子：`tool-fs-search` 的 capped 结果、超长工具输出。`SaveTextSpill` 返回不透明定位符 + 后端提供的检索提示 + 精确字节数。`SpillLocator` 是 branded。

## 13.5　Settings / Credentials / Workspace：其他持久化域

- **Settings**（`settings/`）：`SettingsNamespace` 注册、分层解析（defaults → composition `base` → user document）、owner scopes、hot commits。
- **Credentials**（`credentials/`）：`CredentialRef` 引用（**永远不是值**）进配置、按操作解析、UI-safe `CredentialInfo`、provider 源层（env / .env / .credentials.yaml）。`docs/architecture.md` 说 "Managed credentials live separately in `.credentials.yaml`; a credential left in either `.env` remains a lower-priority fallback"。
- **Workspace**（`workspace/`）：`WorkspaceId` 是 generated uuid，**绝不是路径**（路径规范化会重写路径，引用锚必须稳定）。`realpathNormalize`（`fs.realpath`；尾斜杠、`..`、symlink 全部解析）是唯一唯一性准则。membership 要求 id 在账户里**且** session header 的 canonical cwd 等于 workspace 路径。
- **Storage**（`storage/`）：`StorageBackend` 契约 + `StorageForms` + `DomainSpec`/`Domain` + `domain/changed`——workspace、投影缓存等域 KV 的地基。

## 本章小结

dsh 的持久化把"事件日志"变成了"可恢复的真相源"：

1. **持久化是 seam**：`SessionPersistence` 抽象 + JSONL/SQLite 两个后端，无平行事件类型；
2. **有界写批处理**：`session/flush` 作为主循环的排序/错误检查点；
3. **崩溃恢复不截断**：synthetic `interrupted` 封口保持平衡，是唯一主循环不发的终止原因；
4. **Spill 溢出**：超大输出持久化 + 定位符，模型只见指引；
5. **其他域**：Settings 分层、Credentials 只存引用、Workspace 用 uuid 不用路径。

## 动手实验

1. 读 `packages/session/session-persistence`，确认两个后端（JSONL/SQLite）实现同一契约。
2. 搜 `kind: 'interrupted'`，确认它只在持久化恢复路径出现。
3. 读 `docs/persistence-catalog.md` 的 crash recovery 部分，对照第 5.4 节的 turn 封闭语义。
4. 读 `packages/workspace/workspace`，确认 `realpathNormalize` 的实现与 uniqueness 语义。
