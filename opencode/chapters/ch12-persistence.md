# 第 12 章 持久化与事件溯源：让每一次状态变更都可重放

前面十一章，我们一直在谈"运行时"——主循环怎么转、工具怎么调度、上下文怎么压缩。但有一个问题始终悬而未决：当进程退出、机器重启、甚至崩溃在半句话中间时，一个会话的全部状态去了哪里？为什么重新打开 OpenCode，上一次的对话、token 计费、待办列表都还原封不动地躺在那里？

答案是这一章的主题:OpenCode 的 v2 数据层不是简单地"把会话存进数据库"，而是把会话的每一次变更建模成**事件(event)**,先把事件持久化、再由**投影器(projector)**把事件回放成 SQL 表里的可查询状态。这是一套教科书式的**事件溯源(event sourcing)**架构,跑在 SQLite + Drizzle 之上,用 Effect 的事务和信号量把并发、幂等、可重放这些棘手性质牢牢钉死在类型与代码里。

我们从最底层的 SQLite 配置看起,逐层往上:数据库连接 → 迁移日志 → 事件引擎 → 状态投影。

## 12.1 SQLite 的六条 PRAGMA:把可靠性写进连接初始化

数据层的入口是 `packages/core/src/database/database.ts`。它用 Effect 的 `Layer.effect` 构造一个 `Database.Service`,服务初始化时第一件事不是建表,而是连续下发六条 PRAGMA:

```ts
yield* db.run("PRAGMA journal_mode = WAL")
yield* db.run("PRAGMA synchronous = NORMAL")
yield* db.run("PRAGMA busy_timeout = 5000")
yield* db.run("PRAGMA cache_size = -64000")
yield* db.run("PRAGMA foreign_keys = ON")
yield* db.run("PRAGMA wal_checkpoint(PASSIVE)")
yield* DatabaseMigration.apply(db)
```

这六行不是随手抄来的模板,每一条都对应一个具体的工程权衡:

- **`journal_mode = WAL`**(Write-Ahead Logging):这是整套架构能"边写边读"的前提。传统的回滚日志模式下,写操作会锁住整个数据库,读必须等写完成。WAL 模式把新的写追加到一个单独的 WAL 文件,读操作可以继续访问主库的旧快照——读写不再互相阻塞。对一个需要一边流式写入助手回复、一边让前端查询会话状态的 agent 来说,这几乎是必选项。
- **`synchronous = NORMAL`**:在 WAL 模式下,NORMAL 是官方推荐的同步级别。它不会在每次事务后都 fsync,而是只在 checkpoint 时同步,牺牲极端断电下"最后几个事务"的持久性,换来显著的写入吞吐。
- **`busy_timeout = 5000`**:当一个连接拿不到锁时,不立刻报 `SQLITE_BUSY`,而是自旋重试最多 5000 毫秒。这是多端并发(CLI、server、TUI 可能同时连同一个库)下避免偶发"数据库锁定"错误的护栏。
- **`cache_size = -64000`**:负数表示以 KB 为单位,即约 64 MB 页缓存。给热点查询留足内存。
- **`foreign_keys = ON`**:SQLite 默认**不**强制外键。这里显式打开,意味着后面我们会看到的所有 `references(...onDelete: "cascade")` 才真正生效——删一个 session,它的 message、part、todo 会被数据库自动级联删除,而不是靠应用代码逐个清理。
- **`wal_checkpoint(PASSIVE)`**:启动时做一次被动 checkpoint,把上次遗留的 WAL 内容合并回主库,避免 WAL 文件无限膨胀。PASSIVE 模式不会阻塞其他连接。

整个 `layer` 用 `.pipe(Effect.orDie)` 收尾——数据库初始化失败不是一个可以"优雅处理"的业务错误,而是一个应该让进程直接死掉的缺陷。这是我们在第 10 章 fail-closed 之外看到的另一种态度:**有些前置条件一旦不满足,继续运行只会掩盖问题**。

数据库文件路径由 `path()` 决定,它的逻辑也透露了发布渠道的设计:

```ts
if (Flag.OPENCODE_DB) { ... }
if (["latest", "beta", "prod"].includes(InstallationChannel) || ...)
  return join(Global.Path.data, "opencode.db")
return join(Global.Path.data, `opencode-${InstallationChannel.replace(/[^a-zA-Z0-9._-]/g, "-")}.db`)
```

正式渠道(latest/beta/prod)共用一个 `opencode.db`,而其他渠道(比如开发者自己的 dev 分支)会落到带渠道名后缀的独立文件里。这样开发版的脏数据不会污染你日常使用的正式库。文件名里那个 `replace(/[^a-zA-Z0-9._-]/g, "-")` 是一道朴素但必要的防御:渠道名最终会拼进文件路径,任何非白名单字符都被替换成连字符,杜绝路径注入。

## 12.2 迁移日志:用信号量保证"恰好一次"应用

库连好了,表结构怎么演进?答案在 `packages/core/src/database/migration.ts`。它没有用 Drizzle 自带的迁移命令,而是自己维护了一张迁移日志表,核心是一个**进程内信号量**:

```ts
const lock = Semaphore.makeUnsafe(1)

export function apply(db: Database) {
  return lock.withPermit(applyOnly(db, migrations))
}
```

`Semaphore.makeUnsafe(1)` 创建一个许可数为 1 的信号量,`withPermit` 保证同一时刻只有一个 fiber 能进入迁移流程。为什么需要它?因为多个 Layer 可能在启动期并发地拿到同一个 `Database.Service`,如果不串行化,两个 fiber 可能同时检查"迁移 X 没跑过"、然后都去跑它。信号量把这个竞态彻底消除。

`applyOnly` 的逻辑是经典的迁移日志模式,但有一个值得玩味的兼容性处理:

```ts
yield* db.run(
  sql`CREATE TABLE IF NOT EXISTS ${sql.identifier("migration")} (id TEXT PRIMARY KEY, time_completed INTEGER NOT NULL)`,
)
let completed = new Set(
  (yield* db.all<{ id: string }>(sql`SELECT id FROM ${sql.identifier("migration")}`)).map((row) => row.id),
)
if (completed.size === 0) {
  // Existing installs used Drizzle's migration journal. Seed the new
  // journal once so TypeScript migrations don't replay old SQL.
  if (yield* db.get(sql`SELECT name FROM sqlite_master WHERE type = 'table' AND name = ${"__drizzle_migrations"}`)) {
    yield* db.run(sql`
      INSERT OR IGNORE INTO ${sql.identifier("migration")} (id, time_completed)
      SELECT name, ${Date.now()}
      FROM ${sql.identifier("__drizzle_migrations")}
      WHERE name IS NOT NULL
    `)
    completed = new Set(...)
  }
}
```

这段代码讲了一个迁移系统自身演进的故事:OpenCode 早期用 Drizzle 的迁移机制,日志记在 `__drizzle_migrations` 表;后来切换到自定义的 TypeScript 迁移。如果直接上新系统,老库的 `migration` 表是空的,新系统会以为"什么都没迁过",于是把已经执行过的老 SQL 全部重放一遍——灾难。这里的处理是:当且仅当新日志为空、且检测到老的 `__drizzle_migrations` 表存在时,把老日志里的迁移 id **一次性播种**进新表。注释写得明明白白:`Seed the new journal once so TypeScript migrations don't replay old SQL`。

真正应用迁移的循环,每条迁移都裹在自己的事务里:

```ts
for (const migration of input) {
  if (completed.has(migration.id)) continue
  yield* db.transaction((tx) =>
    Effect.gen(function* () {
      if (!process.env.OPENCODE_SKIP_MIGRATIONS) yield* migration.up(tx)
      yield* tx.run(
        sql`INSERT INTO ${sql.identifier("migration")} (id, time_completed) VALUES (${migration.id}, ${Date.now()})`,
      )
    }),
  )
}
```

三层保险叠在一起:`completed.has(migration.id)` 跳过已完成的;`db.transaction` 保证"执行 SQL"和"写日志"要么都成功、要么都回滚,杜绝"SQL 跑了但日志没记"的悬空状态;`OPENCODE_SKIP_MIGRATIONS` 环境变量给测试一个只建日志、不动结构的逃生口。

## 12.3 事件表:每个聚合一条乐观序列

迁移搞定了表结构,我们终于可以看核心数据模型。事件溯源的两张表定义在 `packages/core/src/event/sql.ts`,只有二十几行,但密度极高:

```ts
export const EventSequenceTable = sqliteTable("event_sequence", {
  aggregate_id: text().notNull().primaryKey(),
  seq: integer().notNull(),
  owner_id: text(),
})

export const EventTable = sqliteTable("event", {
  id: text().$type<EventV2.ID>().primaryKey(),
  aggregate_id: text().notNull().references(() => EventSequenceTable.aggregate_id, { onDelete: "cascade" }),
  seq: integer().notNull(),
  type: text().notNull(),
  data: text({ mode: "json" }).$type<Record<string, unknown>>().notNull(),
}, (table) => [
  uniqueIndex("event_aggregate_seq_idx").on(table.aggregate_id, table.seq),
  index("event_aggregate_type_seq_idx").on(table.aggregate_id, table.type, table.seq),
])
```

两张表的分工很清楚:

- `event_sequence` 是每个**聚合(aggregate)**的"游标"。一个聚合通常就是一个会话(session),`aggregate_id` 是它的主键,`seq` 记录这个聚合当前已经写到第几号事件,`owner_id` 标记谁"拥有"这个聚合(后面讲并发控制时会用到)。
- `event` 是不可变的事件流本身。每条事件有全局唯一的 `id`、所属的 `aggregate_id`、在该聚合内的序号 `seq`、事件类型 `type`,以及一个 JSON 列 `data` 装载事件载荷。

最关键的是那个 `uniqueIndex("event_aggregate_seq_idx").on(aggregate_id, seq)`。它在数据库层面强制:**同一个聚合内,seq 不能重复**。这一个唯一索引,就是整套乐观并发控制的物理基石——我们马上会看到,事件引擎正是靠它来检测冲突。第二个索引 `(aggregate_id, type, seq)` 则是为了高效地按类型回放(比如"把这个会话所有 `Tool.Called` 事件按序拉出来")。

`EventTable.aggregate_id` 对 `EventSequenceTable` 设了 `onDelete: "cascade"` 外键——还记得 12.1 里打开的 `foreign_keys = ON` 吗?删一个聚合的序列行,它的全部事件会被数据库自动清掉。两处设计在这里咬合。

## 12.4 事件引擎:事务内提交、乐观序列、幂等重放

数据模型之上是 `packages/core/src/event.ts`(681 行)——EventV2 的引擎。它要解决的核心难题是:**多个端可能同时往一个会话写事件,如何保证写入有序、不丢、不重?**

引擎区分两类事件。**持久(durable)的同步事件**会真正落库;**非持久的临时事件**只在内存里通过 PubSub 广播给监听者,不进数据库。`define()` 在注册事件类型时,把同步事件登记进一个专门的 `syncRegistry`,这些事件的编解码不依赖任何运行时服务,因为它们要能在没有完整上下文的情况下被回放。

持久事件的提交走 `commitSyncEvent`,这是全章最精华的一段逻辑。它在**一个数据库事务里**依次完成:

1. 读出该 `aggregate_id` 在 `event_sequence` 里的最新 `seq`;
2. **期望**新事件的序号正好是 `latest + 1`。如果不是——说明在你读取和写入之间,有别人插了一条事件进来——直接抛 `InvalidSyncEventError` 让事务回滚。这就是乐观并发控制:不加悲观锁,而是赌"没人和我冲突",赌输了就重来;
3. 在同一个事务里运行所有 `commitGuards`(提交前校验)和所有 `projectors`(状态投影);
4. 最后把新的 `seq` 写回 `event_sequence`(用 `onConflictDoUpdate`),并把事件本身插入 `event` 表。

把投影器放进**同一个事务**是这套架构最重要的正确性保证:事件的写入和它对 SQL 状态表的影响是**原子**的。绝不会出现"事件写进了 event 表,但 session 表的 token 计数没更新"这种撕裂状态。要么整个变更可见,要么整个回滚。

幂等性靠两道关卡守住:重放一条已存在的事件时,引擎会用 `isDeepStrictEqual` 比较"存储的事件载荷"和"重新编码的载荷",一致就跳过,不重复施加副作用;同时 `strictOwner` 检查确保只有持有该聚合所有权的一方才能写入。

`publishEvent` 是对外的统一入口:durable 事件走 `commitSyncEvent` 落库再 `notify` 广播;非 durable 事件只 `notify`。读取侧,`aggregateEvents` / `streamEvents` 把"历史事件(从 event 表查出)"和"实时事件(从 PubSub 订阅)"拼接成一条连续的流——一个刚连上的客户端,可以先收到全部历史、再无缝衔接到实时更新。

引擎对监听者错误的处理也体现了边界隔离的思路:`notify` / `observe` 用 `catchCauseIf(cause => !Cause.hasInterrupts(cause))` 把单个监听者的异常吞掉、不让它污染整条广播链路——但**中断(interrupt)是例外**,中断必须向上传播,因为它代表"这个 fiber 该停了"而不是"这个监听者出错了"。这个细微的区分,正是第 5 章我们讨论过的"可中断性"在数据层的延续。

## 12.5 投影器:把事件翻译成可查询的 SQL 状态

事件流是"发生了什么"的真相来源(source of truth),但你不会想每次查会话标题都去重放几千条事件。所以需要**投影器**:订阅事件,把它们"投影"成扁平、可索引、可直接查询的 SQL 表。这层逻辑在 `packages/core/src/session/projector.ts`。

投影的目标表定义在 `session/sql.ts`,围绕一个会话铺开了一组表:`session`(会话主体)、`message` / `part`(v1 的消息与片段)、`session_message`(v2 的消息)、`session_input`(待提升的输入)、`todo`(待办)、`session_context_epoch`(上下文纪元,第 6 章的主角)。它们全部通过 `references(...onDelete: "cascade")` 挂在 `session` 上,删会话即删全部从属数据。

投影器本质上是一组"事件类型 → 表变更"的映射函数。比如 `sessionRow` 把一个 `SessionInfo` 摊平成 `session` 表的一行,逐字段对应:

```ts
function sessionRow(info: SessionV1.SessionInfo): typeof SessionTable.$inferInsert {
  return {
    id: info.id,
    project_id: info.projectID,
    workspace_id: info.workspaceID ?? null,
    ...
    cost: info.cost ?? 0,
    tokens_input: (info.tokens ?? { input: 0 }).input,
    ...
    time_created: info.time.created,
    time_updated: info.time.updated,
  }
}
```

注意大量的 `?? null` 和 `?? 0`——投影器不信任上游数据的完整性,对每一个可空字段都给了显式默认值。这是第 10 章"边界不信任"原则在数据投影边界的又一次出现。

`Created` 事件的投影器尤其能说明事件溯源的精妙:

```ts
const stored = yield* db
  .insert(SessionTable)
  .values(sessionRow(event.data.info))
  .onConflictDoNothing()
  .returning({ sessionID: SessionTable.id })
  .get()
  .pipe(Effect.orDie)
if (!stored) return yield* Effect.die(new SessionAlreadyProjected())
```

`onConflictDoNothing` + 检查返回值:如果这个 session 已经被投影过(重放场景),插入会冲突、`stored` 为空,于是抛 `SessionAlreadyProjected`。投影器必须能容忍同一个事件被重放——这正是 12.4 里幂等性在投影侧的体现。

最能体现"状态从事件累积而来"的是 token 计费。OpenCode 不在某处直接写"这个会话花了多少钱",而是每来一个带用量的事件,就**增量累加**:

```ts
function applyUsage(db, sessionID, value: Usage, sign = 1) {
  return db.update(SessionTable).set({
    cost: sql`${SessionTable.cost} + ${value.cost * sign}`,
    tokens_input: sql`${SessionTable.tokens_input} + ${value.tokens.input * sign}`,
    ...
  }).where(eq(SessionTable.id, sessionID)).run().pipe(Effect.orDie)
}
```

那个 `sign` 参数是点睛之笔。投影器对 `PartUpdated` 的处理是:先用旧值 `applyUsage(..., previous, -1)` 把上一版的用量**减掉**,再用新值 `applyUsage(..., next)` 加回来。`PartRemoved` / `MessageRemoved` 则只做减法。于是 `session` 表里的 token 总数,永远等于当前所有 part 用量之和——即使某个 part 被编辑或删除,账也能精确对平。计费状态不是"写"出来的,而是从事件流里**算**出来的。

回到第 11 章的闭环:`Compaction.Ended` 事件的投影器会调用 `SessionContextEpoch.requestReplacement(db, sessionID, seq)`,推进上下文纪元的 `baseline_seq`。压缩、纪元、事件投影,三者在这里汇成一个完整的故事——压缩产生一个事件,事件被投影,投影推进纪元基线,下一轮渲染据此裁剪历史。每一环都建立在"事件先落库、状态后投影"这一基座之上。

## 12.6 为什么值得这么复杂

读到这里你可能会问:存个会话而已,至于搞事件溯源、乐观锁、投影器吗?直接 `UPDATE session SET ...` 不香吗?

值得,因为它换来了三个直接 UPDATE 永远给不了的性质:

- **可重放(replayable)**:状态由事件算出来,意味着任何时候都能从事件流重建出完整状态。投影逻辑改了?删掉投影表、重放事件即可,不用数据迁移。
- **可恢复(recoverable)**:进程崩在半句助手回复中间,重启后从事件流恢复,因为每个有意义的状态变更都先成了一条落库的事件。
- **可审计(auditable)**:`event` 表是一条不可变的、带序号的变更日志。会话怎么一步步走到现在的状态,全程有据可查——这对调试一个本质上不确定的 AI 系统是无价的。

而把投影器塞进提交事务、用唯一索引 + 乐观序列做并发控制、用信号量串行化迁移、用 `?? 0` 在投影边界兜底——这些不是过度设计,而是把"可重放/可恢复/可审计"这些性质从"我们小心点就能保证"降级成"代码和数据库结构帮我们保证"。这正是这本书反复看到的那条主线:**把正确性钉进类型、约束和事务里,而不是寄希望于运行时的自觉**。

## 本章小结

- OpenCode v2 的数据层是一套跑在 SQLite + Drizzle 上的事件溯源架构:事件先落库,状态由投影器从事件算出。
- 数据库初始化下发六条 PRAGMA——`WAL` 让读写不互斥、`synchronous = NORMAL` 平衡持久与吞吐、`busy_timeout = 5000` 容忍并发抢锁、`foreign_keys = ON` 让级联删除真正生效——并以 `Effect.orDie` 表明初始化失败应让进程直接死掉。
- 迁移用 `Semaphore.makeUnsafe(1)` 串行化,维护独立的 `migration` 日志表,并一次性从老的 `__drizzle_migrations` 播种以避免重放历史 SQL;每条迁移裹在自己的事务里,三层保险保证"恰好一次"。
- `event_sequence` + `event` 两张表配合 `uniqueIndex(aggregate_id, seq)`,在数据库层面把乐观并发控制钉死:`commitSyncEvent` 期望新序号正好是 latest+1,否则回滚重来。
- 投影器和事件写入跑在**同一个事务**里,保证事件与状态的原子一致;通过 `onConflictDoNothing`、`isDeepStrictEqual` 幂等地容忍重放;token 计费用带 `sign` 的增量累加,使状态永远等于事件流的聚合结果。
- 这套复杂度换来了直接 UPDATE 给不了的三个性质:可重放、可恢复、可审计。

## 动手实验

1. **观察 WAL 文件**:找到你本地的 `opencode.db`(参考 `path()` 的逻辑,正式渠道在 `Global.Path.data/opencode.db`),发起一次会话对话,然后用 `ls -la` 查看同目录。你会看到 `opencode.db-wal` 和 `opencode.db-shm` 两个伴生文件。用 `sqlite3 opencode.db "PRAGMA journal_mode;"` 确认当前模式,再观察一次正常退出后 WAL 文件的大小变化,理解 PASSIVE checkpoint 的作用。

2. **读懂事件流**:用 `sqlite3 opencode.db "SELECT seq, type FROM event WHERE aggregate_id='<某会话id>' ORDER BY seq LIMIT 30;"` 把一个会话的前 30 条事件按序打印出来。对照第 4、5 章的事件类型(`Prompted`、`Step.Started`、`Tool.Called`、`Text.Started` 等),尝试只看事件序列就还原出"这一轮对话发生了什么"。这就是"可审计"的实感。

3. **验证乐观序列的唯一性**:用 `sqlite3 opencode.db ".schema event"` 查看建表语句,找到 `event_aggregate_seq_idx` 这个唯一索引。然后手动尝试 `INSERT` 一条 `(aggregate_id, seq)` 与已有记录重复的事件,观察 SQLite 报出的 `UNIQUE constraint failed`。这正是 `commitSyncEvent` 检测并发冲突所依赖的数据库级保证。

4. **追踪 token 是怎么"算"出来的**:在 `session/projector.ts` 里找到 `applyUsage` 和它在 `PartUpdated` 投影中的两次调用(`previous, -1` 与 `next`)。用 `sqlite3` 查询某会话的 `tokens_input`,再把该会话所有 part 的用量手动加总,验证两者相等。然后思考:如果把"先减旧值"那行删掉,编辑一条消息后计费会变成什么样?

---

下一章我们离开数据层,去看 OpenCode 的**可扩展性边界**:第 13 章将读 `plugin/` 与 skill、MCP 的相关代码,看这个 agent 如何在不改核心的前提下,让插件注册工具、让 skill 注入能力、让 MCP 把外部服务器接进工具系统——以及它在这些"外部代码进入运行时"的接缝处,又设了哪些信任边界。
