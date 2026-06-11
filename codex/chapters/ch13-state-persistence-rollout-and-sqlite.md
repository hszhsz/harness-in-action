# 第 13 章　状态持久化：Rollout 与 SQLite

第 12 章把上下文工程讲透了——Codex 在每一轮精心装配 base instructions、`AGENTS.md`、环境上下文，让模型"懂"你的项目。但这一切都活在内存里。终端一关、进程一退，对话历史、每一次工具调用、模型每一段推理，会不会就此烟消云散?

不会。Codex 有一套**双层持久化**:每个会话的完整事件流以 **JSONL** 形式落盘成一份**可回放的 rollout 文件**(单一事实来源),另有一套 **SQLite** 状态库作为可被随时重建的**查询索引**,让"列出最近会话""按 cwd/关键词检索""恢复某个历史会话"这类操作不必每次都去扫文件系统。这一章拆这两层:rollout 怎么写、写什么、怎么读回来,SQLite 索引怎么和 rollout 对账、怎么在不可用时优雅降级,以及"恢复一个会话"时这些落盘记录如何被重新装回内存。代码主要在 `rollout/` 这个独立 crate:`recorder.rs`、`policy.rs`、`session_index.rs`、`state_db.rs`、`compression.rs`,以及 `core/src/session/rollout_reconstruction.rs`。

---

## 13.1 Rollout 是什么:一行一事件的 JSONL

Rollout 是 Codex 的**会话黑匣子**。`recorder.rs` 文件头一句就点题:"Persist Codex session rollouts (.jsonl) so sessions can be replayed or inspected later." 它把会话里发生的每一件"规范事件"按时间顺序写成 JSONL——每行一个 JSON 对象,既能被 `jq`/`fx` 这类工具直接检视,也能被程序逐行解析回放。

落盘的单位是 `RolloutItem`(`protocol/src/protocol.rs`),一个五态枚举:

```rust
pub enum RolloutItem {
    SessionMeta(SessionMetaLine),
    ResponseItem(ResponseItem),
    Compacted(CompactedItem),
    TurnContext(TurnContextItem),
    EventMsg(EventMsg),
}
```

每个 item 写盘前会被包成一行带时间戳的 `RolloutLine`——`JsonlWriter::write_rollout_item` 用 `#[serde(flatten)]` 把 `timestamp` 字段和 item 自身拍平到同一个 JSON 对象里:

```rust
#[derive(serde::Serialize)]
struct RolloutLineRef<'a> {
    timestamp: String,
    #[serde(flatten)]
    item: &'a RolloutItem,
}
```

这五态各有分工:`SessionMeta` 是会话的"出生证"(每个文件的第一行);`ResponseItem` 是模型与工具的原始往返(消息、推理、函数调用及其输出);`EventMsg` 是 harness 侧的事件(用户消息、补丁应用、token 计数等);`Compacted` 记录第 12 章讲过的压缩检查点;`TurnContext` 记录某一轮的上下文快照(模型、cwd、compaction hash 等)。注释把这条设计哲学说得很直白——持久化这些 "Codex executive markers" 是为了"日后能分析整条流程(比如 compaction、API turn)"。

---

## 13.2 文件布局:按日期分桶,文件名自带时间戳与 ID

Rollout 文件存在哪、叫什么,`precompute_log_file_info` 说了算。路径是 `~/.codex/sessions/YYYY/MM/DD/` 的**日期分桶**结构(`SESSIONS_SUBDIR = "sessions"`,月、日都补零成两位),文件名形如:

```
rollout-2025-05-07T17-24-21-5973b6c0-94b8-487b-a530-2aeb6098ae0e.jsonl
```

也就是 `rollout-<YYYY-MM-DDThh-mm-ss>-<conversation_id>.jsonl`。这里藏着一个跨平台的小心思——时间戳里用 `-` 而非 `:` 分隔时分秒,源码注释写明原因:"Use `-` instead of `:` for compatibility with filesystems that do not allow colons in filenames"(Windows 文件名不允许冒号)。

把**时间戳和会话 ID 直接编码进文件名**是个关键决策:它让"按时间排序""按 ID 定位"这类操作连文件都不用打开,光读目录项的文件名就能完成。`parse_timestamp_uuid_from_filename` 正是据此从文件名解析出 `(created_at, id)`,用于排序与游标分页。归档会话则另存于 `archived_sessions/`(`ARCHIVED_SESSIONS_SUBDIR`)。

---

## 13.3 写入器:后台单写者 + 缓冲重试

写盘绝不能阻塞主循环。`RolloutRecorder` 的做法是:用一个有界 channel(`mpsc::channel::<RolloutCmd>(256)`)把写请求投递给一个**独占文件句柄的后台 Tokio 任务** `rollout_writer`,所有真正的异步 I/O 都在那条任务里发生。注释解释了为什么用有界通道:缓冲满了 `send` future 会让出执行权,"这没关系——我们只要保证不在调用者线程上做*阻塞* I/O 即可"。

写者接受四种命令:`AddItems`(追加)、`Persist`(物化文件并落盘所有缓冲项)、`Flush`(确保此前写入都已提交)、`Shutdown`(排空后停止)。这里有一处值得玩味的**延迟物化**设计:新建会话时,`new()` 并不立刻创建文件,而是把路径和元数据"预计算"好(`deferred_log_file_info`),把建文件/开文件**推迟到第一次显式 `persist()`**。`is_deferred()` 判断的就是"还没物化"这个状态。好处是:一个开了头却没产生任何内容的会话,不会在磁盘上留下空壳文件。

更见功力的是**失败可重试**的缓冲语义。`RolloutWriterState` 维护一个 `pending_items` 队列,`write_pending_items_once` 逐项写,**只在成功写入后才从队列里 `drain` 掉**;一旦某项写失败,立即 `break`,未写的后缀原样留在队列里。`write_pending_with_recovery` 则给了一次"丢句柄重开再试"的机会——第一次失败就 `enter_recovery_mode`(把 `writer` 置 `None`)并 `warn!`,然后重试一次;两次都失败才真正返回错误。注释点明设计意图:"I/O failures drop the file handle but keep the unwritten suffix so the next barrier can reopen the file and retry"——**磁盘抖动不丢数据,下一个屏障点(flush/persist/shutdown)还能补救**。还有一处贴心细节:`enter_recovery_mode` 用 `last_logged_error` 去重,避免同一个错误把日志刷爆。

至于哪些 item 真正落盘,由 `policy.rs` 的 `is_persisted_rollout_item` 把关——`Compacted`/`TurnContext`/`SessionMeta` 永远持久化;`ResponseItem` 与 `EventMsg` 则各有一张**白名单**(`should_persist_response_item` / `should_persist_event_msg`)。比如 `ResponseItem::CompactionTrigger` 和 `ResponseItem::Other` 明确**不**落盘;海量的流式增量事件(`AgentMessageContentDelta`、`ReasoningRawContentDelta`、`ExecCommandOutputDelta` 等)也一律不写——**rollout 记的是"已成形的事实",不是"过程噪声"**。一个精巧的例外是 `EventMsg::ItemCompleted`:只有当它承载的是 `TurnItem::Plan` 时才持久化,注释解释 plan 项"派生自流式标签、不属于原始 ResponseItem 历史",所以单独持久化它的完成态,以便 resume 时重放、又不至于让 rollout 被每个 item 的生命周期事件撑肿。

---

## 13.4 SessionMeta:会话的出生证

每个 rollout 文件的第一行是一条 `SessionMeta`,由 `write_session_meta` 在首次写入时落盘。它是会话的"出生证",字段相当丰富:会话 `id`、`forked_from_id`/`parent_thread_id`(派生/父子关系)、`timestamp`、`cwd`、`originator`、`cli_version`(`env!("CARGO_PKG_VERSION")`)、`source`(CLI/VSCode/...)、`model_provider`、`base_instructions`、`dynamic_tools`、`memory_mode`,等等。

特别值得一提的是 **git 信息的采集时机**:`write_session_meta` 会先用 `get_git_repo_root(cwd)` 判断 cwd 是否在 git 仓库内,**只有在仓库里**才异步 `collect_git_info` 采集 commit hash、分支、远端 URL,拼成 `SessionMetaLine { meta, git }`。这让恢复会话列表时能显示"这个会话当时在哪个分支、哪个 commit 上干活"——把代码现场的版本坐标也一并冻进了黑匣子。

为什么 SessionMeta 这么重要?因为它是**恢复会话的元数据总源**。第 12 章提过 base instructions 是编译期常量,但恢复一个**老会话**时,模型该用的是那次会话当时的 base instructions——`InitialHistory::get_base_instructions()` 正是从历史里的 `SessionMeta.base_instructions` 把它捞回来,`dynamic_tools`、`multi_agent_version` 同理。**出生证不仅记录身份,还冻结了那次会话赖以运行的关键配置**。

---

## 13.5 读回来:`load_rollout_items` 与"宽容解析"

恢复会话从读文件开始。`load_rollout_items` 逐行解析 rollout,把每行还原成 `RolloutItem`,同时返回**首个** `SessionMeta` 的 id 作为这条线程的规范 thread id(注释:"Use the FIRST SessionMeta encountered ... as the canonical thread id")。

这个读取器的姿态是**宽容但不静默**——和第 12 章 `AGENTS.md` 那套"边界不信任、但要包容"一脉相承:

- 空行直接跳过;若整个文件**没有任何非空行**,返回 `IoError::other("empty session file")`。
- 单行 JSON 解析失败不会让整个恢复崩掉——`warn!` 记一条,`parse_errors` 计数加一,继续读下一行。一个损坏的行不该毁掉整次恢复。
- 老格式兼容:`strip_legacy_ghost_snapshot_rollout_line` 识别并丢弃历史遗留的 `ghost_snapshot` 行——**rollout 是长期存在的磁盘格式,必须能读得动旧版本写下的文件**,这是任何持久化格式都逃不掉的向后兼容责任。

读完后,`get_rollout_history` 把结果包成 `InitialHistory`——一个四态枚举:`New`(全新)、`Cleared`(清空)、`Resumed(ResumedHistory)`(恢复)、`Forked(Vec<RolloutItem>)`(派生)。`ResumedHistory` 带着 `conversation_id`、完整的 `history` 和 `rollout_path`。这个枚举之上挂了一串便捷方法(`forked_from_id`、`session_cwd`、`get_base_instructions`、`get_dynamic_tools`...),让上层从"一堆 rollout 项"里按需提取出恢复所需的各种元数据,而不必关心它们散落在哪几行。

---

## 13.6 从落盘记录到内存历史:`reconstruct_history_from_rollout`

读回 `RolloutItem` 只是把磁盘上的字节变回结构体。但要让模型**接着上次往下聊**,还得把这串异构事件重新熔炼成一份连贯的 `Vec<ResponseItem>` 内存历史——这是 `core/src/session/rollout_reconstruction.rs` 的活,也是整条持久化链路里最烧脑的一环。

它的核心算法是**逆序回放**:`rollout_items.iter().enumerate().rev()` 从最新往最旧扫,把事件累积进"当前进行中的轮段"(`ActiveReplaySegment`),直到撞见它对应的 `TurnStarted` 才 `finalize` 这一段。为什么要倒着扫?注释讲明了:一旦找到**最新一个幸存的 replacement-history 检查点**(即第 12 章 compaction 写下的压缩历史),"更旧的 rollout 项就不再影响重建的历史了"——压缩检查点本身就是一份完整的历史基底,逆序扫能最早地命中它,从而只需正向重放那之后的"幸存尾巴",省去重放整条历史。

这里要处理几种纠缠的语义:

- **Compaction 清基线**:逆向看,一个 `Compacted` 会"清除"任何更旧的基线(`TurnReferenceContextItem::Cleared`),除非同段里有更新的 `TurnContext` 已经重建了基线——这正是第 12 章"压缩摘要必须是历史最后一项"在恢复侧的镜像。
- **Thread rollback**:`ThreadRolledBack` 事件意味着"丢弃最新 N 个用户轮"。在逆序扫描里,它被翻译成 `pending_rollback_turns`——"跳过接下来要 finalize 的 N 个用户轮段"。
- **承接上一轮设置**:`previous_turn_settings`(模型、comp_hash、realtime 状态)取自"最新一个幸存的、确立了这些设置的用户轮",好让恢复后的会话沿用上次的模型选择等。

整套逻辑用 `TurnReferenceContextItem` 的 `NeverSet`/`Cleared`/`Latest` 三态精确区分"从没设过基线"和"曾有基线但被压缩作废"——只有后者需要在恢复时显式发一段"清基线"信号。**持久化的难点从来不是写,而是把碎片化的历史无损地拼回一个语义正确的现场。**

---

## 13.7 SQLite 状态库:可重建的查询索引

光有 rollout 文件,"列出最近 20 个会话"就得扫一堆目录、打开一堆文件读头部——会话一多就慢。于是 Codex 加了第二层:**SQLite 状态库**(`state_db.rs`,底层是 `codex_state::StateRuntime`)。它是 rollout 的**查询索引**,缓存每个线程的元数据(cwd、git 信息、首条用户消息、预览、source、provider、时间戳等),让列表、过滤、搜索走数据库而非文件系统。

关键在于这个索引的定位——**它是派生的、可重建的,rollout 文件才是事实来源**。这条原则贯穿整个 `state_db.rs`:

- **启动回填(backfill)**:`try_init_with_roots_inner` 初始化运行时后,会 `wait_for_backfill_gate` 等待回填完成——把文件系统里 SQLite 还没收录的 rollout 扫进库。回填有超时(`STARTUP_BACKFILL_WAIT_TIMEOUT`,非测试下 30 秒)和轮询间隔(`STARTUP_BACKFILL_POLL_INTERVAL`,1 秒),超时则报错;久等不决还会 `emit_startup_warning` 告诉用户"正在等待回填"。
- **读修复(read-repair)与对账(reconcile)**:列表时采取"文件系统优先、顺手修库"的策略。`read_repair_rollout_path` 有快慢两路——快路在 metadata 行已存在时**原地**更新 rollout 路径,且"当读修复算出来没有有效变更时就跳过写"(`if repaired == metadata { return; }`),避免无谓写库;慢路在行缺失/不可读时,从 rollout 内容重建元数据再 `reconcile_rollout` 写回。
- **陈旧行自愈**:`list_threads_db` 拿到 DB 结果后,会逐条用 `existing_rollout_path` 校验文件是否还在;若 DB 指向一个已不存在的 rollout 路径,就 `warn!` 记一条 "stale_db_path_dropped" 并 `delete_thread` 把这条陈旧行删掉——**索引可以错,但不能把错带给用户**。

---

## 13.8 降级哲学:DB 不可用就回退文件系统

既然 SQLite 只是索引,那它**坏了、缺了、对不上**的时候,绝不能让整个"列出会话"功能瘫痪。`list_threads_with_db_fallback` 把这套降级逻辑写得淋漓尽致:

- `state_db_ctx.is_none()`(库不可用)→ `record_fallback("list_threads", "db_unavailable", ...)`,直接返回文件系统扫描结果,注释明说"Keep legacy behavior when SQLite is unavailable"。
- DB 查询返回 `None`(出错)→ 同样 `record_fallback(... "db_error" ...)`,回退文件系统页。
- 带元数据过滤的列表 → 以**文件系统页为准**,再用 SQLite 补齐缺失字段(`fill_missing_thread_item_metadata_from_state_db`)。
- 兜底分支甚至直接 `tracing::error!("Falling back on rollout system")`——大白话告诉运维"正在退回 rollout 系统"。

注意每一条降级路径都调用了 `codex_state::record_fallback(操作, 原因, ...)`:`db_unavailable`、`db_error`、`metadata_filter`、`missing_row`、`backfill_incomplete`……**降级不是悄悄发生的,每一次回退都打了带原因的遥测点**(第 15 章),团队能据此量化"SQLite 这层到底有多可靠、多久退一次"。这正是"派生缓存"应有的工程姿态:**索引尽力加速,但永远不是关键路径上的单点故障**。

---

## 13.9 会话名索引与冷文件压缩:两个轻量旁路

除了主 rollout 和 SQLite,还有两个轻量的旁路设施值得一提。

**会话名索引**(`session_index.rs`):用户给会话起的名字单独记在 `session_index.jsonl` 里,而且是**追加式**(append-only)的——`append_thread_name` 每次改名都追加一条 `SessionIndexEntry { id, thread_name, updated_at }`,"最新的一条胜出"(注释:"the most recent entry wins")。读取时反着读才高效:`scan_index_from_end` 用 `READ_CHUNK_SIZE = 8192` 的块**从文件尾部往前**逐块读、逐行解析,一碰到匹配就停——因为最新的改名在文件末尾,从尾扫能最快命中。追加写时用一把进程内 `Mutex`(`SESSION_INDEX_LOCK`)串行化,避免并发写交错。删除则走"读全文件 → 过滤掉目标行 → 写临时文件 → 原子 rename"的经典安全替换。

**冷文件压缩**(`compression.rs`):老 rollout 会被后台 worker 用 zstd 压成 `.jsonl.zst`(`COMPRESSED_SUFFIX = ".zst"`),省磁盘。这个 worker 是"尽力而为、失败只记日志、不阻塞启动"的:只压**冷**文件——`MIN_ROLLOUT_AGE = 7 * 24 * 60 * 60`(7 天)内的不动;用一个 `codex_home` 下的 run marker 防止多次运行重叠或过频(`RUN_MARKER_STALE_AFTER = 6 小时`)。而读取与续写对这层压缩**完全透明**:`open_rollout_line_reader` 自动识别 `.jsonl` 和 `.jsonl.zst` 两种表示;要往压缩文件追加时,`materialize_rollout_for_append` 会先把它解压回 `.jsonl`。`open_rollout_line_reader` 甚至处理了"表示形态正在切换"的竞态——文件一时找不到时按 `MAX_NOT_FOUND_RETRIES = 3` 短暂重试,让调用方完全不必关心磁盘上到底是压缩还是明文。

---

## 本章小结

- **Rollout = 会话黑匣子**:每个会话的事件流以 JSONL 落盘(`recorder.rs`),单位是五态 `RolloutItem`(`SessionMeta`/`ResponseItem`/`Compacted`/`TurnContext`/`EventMsg`),每行经 `RolloutLineRef` 用 `#[serde(flatten)]` 拍平成带 `timestamp` 的 `RolloutLine`,可被 `jq`/`fx` 直接检视。
- **文件布局**:`~/.codex/sessions/YYYY/MM/DD/rollout-<时间戳>-<id>.jsonl`,时分秒用 `-` 而非 `:` 以兼容不允许冒号的文件系统;时间戳与 ID 直接编码进文件名,排序/定位不必打开文件;归档存 `archived_sessions/`。
- **后台单写者**:`RolloutRecorder` 经有界 channel(容量 256)把写请求交给独占文件句柄的 `rollout_writer` 任务;延迟物化(没内容不建空文件);`pending_items` 缓冲 + "写成功才 drain" + 一次"丢句柄重开重试" → 磁盘抖动不丢数据。
- **持久化白名单**(`policy.rs`):`Compacted`/`TurnContext`/`SessionMeta` 必存;`ResponseItem`/`EventMsg` 各有白名单;流式增量、`CompactionTrigger`、`Other` 一律不写;`ItemCompleted` 仅 `Plan` 例外。
- **SessionMeta 出生证**:首行记录 id、父子/派生关系、cwd、cli_version、source、base_instructions、dynamic_tools 等;仅当 cwd 在 git 仓库内才采集 git 信息;恢复老会话时 base instructions/dynamic_tools 从这里捞回。
- **读回与重建**:`load_rollout_items` 宽容解析(空行跳过、坏行计数不崩、丢弃 legacy `ghost_snapshot`、空文件报错),取首个 `SessionMeta` 为规范 thread id;`InitialHistory` 四态(New/Cleared/Resumed/Forked);`reconstruct_history_from_rollout` 逆序回放、命中最新 replacement-history 检查点即止,用 `TurnReferenceContextItem` 三态处理 compaction 清基线与 rollback。
- **SQLite 索引**(`state_db.rs`):派生的、可重建的查询索引;启动回填(超时 30s/轮询 1s)、读修复(无变更则跳过写)、对账、陈旧行自愈(`delete_thread`)。
- **降级哲学**:DB 不可用/出错一律回退文件系统扫描,每条降级路径都 `record_fallback(操作, 原因)` 打遥测——索引加速但绝非单点故障。
- **轻量旁路**:`session_index.jsonl` 追加式会话名索引(最新胜出、从尾扫描、Mutex 串行、原子 rename 删除);zstd 冷文件压缩(7 天阈值、6 小时 run marker、读写透明、找不到重试 3 次)。

## 动手实验

1. **实验一:解剖一行 rollout** — 阅读 `recorder.rs` 的 `JsonlWriter::write_rollout_item` 与 `RolloutLineRef`,说明 `#[serde(flatten)]` 如何把 `timestamp` 与 `RolloutItem` 合进一行 JSON。再对照 `protocol.rs` 的 `RolloutItem` 五态,各举一个会出现在 rollout 里的具体例子。
2. **实验二:验证写入器的失败重试** — 阅读 `RolloutWriterState::write_pending_with_recovery` 与 `write_pending_items_once`。解释为什么"只在成功写入后才 `drain`"能保证磁盘临时故障不丢数据;`enter_recovery_mode` 把 `writer` 置 `None` 又起了什么作用?延迟物化(`is_deferred`)解决了什么问题?
3. **实验三:跟踪持久化白名单** — 阅读 `policy.rs`,列出 `should_persist_response_item` 中**不**持久化的两个变体,并解释为什么海量流式 `*Delta` 事件不落盘。`EventMsg::ItemCompleted` 为什么唯独对 `TurnItem::Plan` 持久化?
4. **实验四:画出 SQLite 降级路径** — 阅读 `state_db.rs` 的 `list_threads_with_db_fallback`,枚举它在哪些情形下会回退到文件系统扫描,以及每种情形调用 `record_fallback` 时传入的 `reason` 字符串。结合 `list_threads_db` 里的"陈旧行自愈",解释为什么把 SQLite 设计成"可重建的索引而非事实来源"是更稳健的选择。

> **下一章预告**:持久化让会话能断点续上,但 Codex 的能力边界远不止内置工具——它还能挂接 MCP 服务器、加载插件、调用技能,把第三方能力纳入同一套工具调度与权限模型。下一章我们进入**扩展体系**:Codex 如何统一抽象 MCP/插件/技能,如何把它们的工具安全地暴露给模型,以及扩展贡献的上下文 fragment 如何顺着第 12 章那条"装配总线"投递到正确的位置。
