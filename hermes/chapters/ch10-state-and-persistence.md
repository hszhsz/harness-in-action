# 第 10 章　状态与持久化：SQLite、FTS5 与文件系统的现实

网关收发的每一条消息、Agent 跑过的每一个回合，如果只活在内存里，那么进程一重启就全部蒸发——更别提"跨会话回忆""恢复一个三天前挂起的对话"这类能力了。Hermes 把所有会话与消息持久化进一个 SQLite 数据库，承载这件事的是近 20 万字节的 `hermes_state.py`（实测 `197263` 字节）。这一章我们读它如何设计 schema、如何用 FTS5 做全文检索（还专门处理了中文）、如何在写竞争下不卡死，以及——最能体现工程成熟度的——如何面对 NFS/SMB 这类"WAL 不友好"的文件系统优雅降级。

`hermes_state.py` 的文件头注释把它的设计目标写得很直白：`WAL mode for concurrent readers + one writer (gateway multi-platform)`——多平台网关意味着多个读者并发、一个写者，这正是 SQLite WAL 模式的典型场景。但现实中的文件系统不总是配合，这一章的大半篇幅，其实都在讲 Hermes 怎么和"不配合的现实"打交道。

## 10.1 数据模型：sessions 与 messages 两张核心表

打开 `hermes_state.py` 约第 436 行的 `SCHEMA_SQL`，整个状态层的骨架一目了然。核心是两张表。

`CREATE TABLE IF NOT EXISTS sessions`（约第 440 行）描述一次会话，字段密度相当高：除了 `id TEXT PRIMARY KEY`、`source`、`user_id`、`model`、`started_at`/`ended_at`/`end_reason` 这些基本信息，还有一整组计量字段——`message_count`、`tool_call_count`、`input_tokens`、`output_tokens`、`cache_read_tokens`、`cache_write_tokens`、`reasoning_tokens`、`api_call_count`——把每次会话的 token 消耗和工具调用次数都记进去。再往后是一组计费字段 `billing_provider`、`billing_mode`、`estimated_cost_usd`、`actual_cost_usd`、`cost_status`、`pricing_version`，意味着 Hermes 在数据库层面就为"这次对话花了多少钱"留好了账。值得注意的还有 `parent_session_id TEXT` 配 `FOREIGN KEY (parent_session_id) REFERENCES sessions(id)`——会话之间能形成父子关系（回想第 7 章的子 Agent 委派），以及 `handoff_state`/`handoff_platform`/`handoff_error`（对应第 9 章网关的跨平台交接）、`rewind_count`、`archived` 这些状态字段。一张表把会话的身份、计量、计费、谱系、生命周期状态全收纳了。

`CREATE TABLE IF NOT EXISTS messages`（约第 477 行）描述一条消息，`id INTEGER PRIMARY KEY AUTOINCREMENT` 自增主键，`session_id TEXT NOT NULL REFERENCES sessions(id)` 外键挂回会话。除了 `role`、`content`、`timestamp` 这些常规字段，它把模型交互的全部细节都拆开存了：`tool_call_id`、`tool_calls`、`tool_name`、`finish_reason`，以及一组 reasoning 相关列 `reasoning`、`reasoning_content`、`reasoning_details`、`codex_reasoning_items`、`codex_message_items`——不同 provider（回想第 8 章）返回的推理内容格式各异，这里用多列分别承接。末尾的 `observed INTEGER DEFAULT 0` 和 `active INTEGER NOT NULL DEFAULT 1` 是两个软标记：`active` 让消息可以被"逻辑删除"而不真正删行（压缩、rewind 时用得上），`observed` 标记消息是否已被处理。

另外两张小表也各有用途：`state_meta`（约第 498 行）是一个简单的 key-value 元数据表；`compression_locks`（约第 503 行）带 `holder`、`acquired_at`、`expires_at`，是一把**带过期时间的分布式锁**——多个 Hermes 进程共享同一个 state.db 时，用它协调"谁在压缩哪个会话"，避免重复压缩。索引方面，sessions 按 source、parent、started_at 建了多个索引，messages 按 `(session_id, timestamp)` 建索引。

这里有个容易被忽略但很见功力的细节：`DEFERRED_INDEX_SQL`（约第 521 行）把 `idx_messages_session_active` 这个引用了 `active` 列的索引**单独拎出来延后创建**。注释解释得很清楚：`SCHEMA_SQL` 由 `executescript` 一次性跑完，但老数据库可能还没有 `active` 列（它是后加的），如果索引和建表混在一起跑，老库会因为 "no such column: active" 整体失败。把依赖新列的索引推迟到列补齐之后再建，是为了**让 schema 演进对存量数据库平滑**——这是"约束只能收紧、但不能弄坏老数据"思路在持久化层的体现。

## 10.2 全文检索：FTS5 与中文的麻烦

光能存还不够，Hermes 要支持"翻找我几天前聊过的某件事"，这需要全文检索。`hermes_state.py` 用了 SQLite 的 FTS5 扩展，但它做的远不止"建个虚拟表"那么简单。

基础的 `FTS_SQL`（约第 527 行）建了 `CREATE VIRTUAL TABLE IF NOT EXISTS messages_fts USING fts5(content)`，再配三个触发器 `messages_fts_insert`/`_delete`/`_update`，在 messages 表增删改时自动同步索引。注意触发器里索引的内容不只是 `content`，而是 `COALESCE(new.content, '') || ' ' || COALESCE(new.tool_name, '') || ' ' || COALESCE(new.tool_calls, '')`——把消息正文、工具名、工具调用参数拼在一起索引，这样搜索时连"我让它执行过哪个工具"也能被检索到。

但默认的 FTS5 用的是 unicode61 分词器，对中文是失效的。源码约第 553 行的注释把问题点得很透：默认分词器会把每个 CJK 字符切成单独的 token，于是"大别山项目"被拆成"大 AND 别 AND 山 AND 项 AND 目"，既产生误报又匹配不到精确短语。Hermes 的解法是再建一张 trigram 表：`FTS_TRIGRAM_SQL`（约第 557 行）的 `CREATE VIRTUAL TABLE IF NOT EXISTS messages_fts_trigram USING fts5(content, tokenize='trigram')`——trigram 分词器把文本切成重叠的 3 字节序列，让任何文字系统（CJK、泰文等）的子串查询都能原生工作。

查询时的路由逻辑（约第 3205 行起）更见细致。`_contains_cjk(query)` 判断查询是否含中文；如果含，再数 CJK 字符数。这里有个微妙的边界：trigram 需要 ≥9 个 UTF-8 字节、也就是 ≥3 个 CJK 字符才能匹配。所以 1–2 个中文字的短查询，trigram 索引根本搜不到，代码回退到 `LIKE`。更精细的是源码注释里提到的 issue #20494——像"广西 OR 桂林 OR 漓江"这种查询，总 CJK 字符数是 6（≥3），但**每个单独的 token 只有 2 个字**，trigram 仍会返回 0。所以 Hermes 不看总数,而是逐 token 检查：只要有任何一个非操作符的 CJK token 不足 3 个字,就整个路由到 `LIKE`。把"中文全文检索"这件看似简单的事做对,需要同时备好 trigram 表和 LIKE 回退,并按 token 粒度判断该走哪条路——这是从踩坑里长出来的代码。

FTS5 本身还可能在某些 Python 构建里缺失。`_sqlite_supports_fts5`（约第 716 行）用了一个干净的探针:`CREATE VIRTUAL TABLE temp._hermes_fts5_probe USING fts5(x)` 建个临时表再删掉,能建成功就说明支持 FTS5。一旦发现不支持,`_warn_fts5_unavailable`(约第 707 行)把 `_fts_enabled` 设为 False 并打一条 WARNING,提示用户 `Run hermes update`——全文检索降级关闭,但会话存储本身照常工作。**核心功能不因可选能力缺失而崩溃**,这是典型的优雅降级。

## 10.3 写竞争：短超时 + 应用层抖动重试

`class SessionDB`（约第 583 行)的类文档写明了它的并发模型:`Thread-safe for the common gateway pattern (multiple reader threads, single writer via WAL mode)`。但当多个 Hermes 进程(网关 + 若干 CLI 会话 + worktree 里的 agent)共享同一个 state.db 时,WAL 写锁竞争会造成 TUI 可见的卡顿。

Hermes 的应对策略写在类常量的注释里,很有意思。SQLite 内置的 busy handler 用的是确定性的睡眠时间表,高并发下会产生"车队效应"(convoy effect)——所有写者按同样的节奏退避、又同时醒来抢锁。Hermes 的对策是**绕开内置 busy handler**:把 SQLite 的 `timeout` 设得很短(`timeout=1.0`,见 `__init__` 约第 636 行),改在应用层做重试。常量 `_WRITE_MAX_RETRIES = 15`、`_WRITE_RETRY_MIN_S = 0.020`、`_WRITE_RETRY_MAX_S = 0.150`(约第 600 行)定义了重试策略。

具体落在 `_execute_write`(约第 801 行起):每次写事务用 `BEGIN IMMEDIATE` 在事务开始时就抢 WAL 写锁(而非等到第一条写语句),失败时 `random.uniform` 在 20–150ms 之间随机退避再重试(约第 839 行)。**随机抖动让竞争的写者自然错开,打破车队模式**——这是处理锁竞争的经典手法,Hermes 把它从理论落到了每一次写。此外每成功写 `_CHECKPOINT_EVERY_N_WRITES = 50` 次,触发一次 WAL checkpoint(`_try_wal_checkpoint`,约第 852 行,用 `PRAGMA wal_checkpoint(TRUNCATE)`),把 WAL 文件截断回收,避免它无限膨胀。

## 10.4 和文件系统的现实和解:WAL 降级与畸形 schema 自愈

这一节是整章最能体现"工程成熟度"的部分。理想中 SQLite WAL 模式很美好,但现实里 state.db 经常被放在 NFS、SMB/CIFS、某些 FUSE 挂载或 WSL1 上——这些文件系统对 WAL 依赖的共享内存(mmap)和 fcntl 字节范围锁支持不可靠。源码约第 39 行起的一大段注释直接引用了 SQLite 官方文档说明这一点[[SQLite WAL 文档]](https://www.sqlite.org/wal.html)。

核心函数是 `apply_wal_with_fallback`(约第 157 行)。它的逻辑层层设防:先用只读探针 `PRAGMA journal_mode` 看看当前是不是已经是 wal(是就直接返回,不重复设置,避免 WAL-init 去 unlink 别的连接正持有的文件);然后尝试 `PRAGMA journal_mode=WAL`;如果抛 `OperationalError`,就检查错误消息里是否含 `_WAL_INCOMPAT_MARKERS`(约第 54 行)里的标记——`"locking protocol"`(NFS/SMB 上的 SQLITE_PROTOCOL)、`"not authorized"`(某些 FUSE 挂载直接禁掉 WAL pragma)等。**只有匹配这些已知标记才降级**,无关的 OperationalError 照常抛出、绝不静默吞掉。降级前还有最后一道保险:`_on_disk_journal_mode` 检查磁盘上的 DB 头,如果别的进程已经把它设成 WAL,就不降级(避免互相打架)。确认要降级,才回退到 `PRAGMA journal_mode=DELETE`——这是 WAL 出现之前的默认模式,在 NFS 上能正常工作,代价是并发读会被写阻塞。

降级时的 WARNING 也做了精细处理:`_log_wal_fallback_once`(约第 207 行)用 `_wal_fallback_warned_paths` 集合按 db_label 去重,**每个进程每个数据库只警告一次**。注释解释了为什么——kanban 每次操作都开新连接,不去重的话 NFS 用户的 errors.log 一小时会被几百条相同警告刷爆。一个 warning 也要考虑刷屏问题,可见这段代码是被真实运维场景反复打磨过的。

另一类现实问题是 schema 畸形。约第 234 行起的注释描述了一种棘手情况:sqlite_master 里出现重复的对象定义(比如两条 `messages_fts` 的 `CREATE VIRTUAL TABLE`),SQLite 在准备语句、解析整个 schema 时就会抛 "malformed database schema",连 `PRAGMA integrity_check` 都跑不了。`SessionDB.__init__`(约第 606 行)对此有专门的自愈路径:`_connect_and_init` 第一次失败若是 `is_malformed_db_error`,就调 `repair_state_db_schema`(约第 336 行)——先备份,再 `DELETE FROM sqlite_master WHERE name LIKE 'messages_fts%'` 删掉畸形的 FTS 对象 + `VACUUM`(约第 407 行,canonical 的 sessions/messages 数据保留),然后重开一次。注释点明这正是让 Desktop/Dashboard 能**自愈**、而不是默默显示"no sessions"的关键。把"数据库坏了"也当成一种要优雅处理的常态而非崩溃理由,这是把可靠性做到骨子里。

把这一章串起来看,`hermes_state.py` 的设计哲学是:**理想路径要快,但现实路径不能崩**。WAL 给并发、trigram 给中文检索、抖动重试给写竞争——这些是"理想路径"的优化;而 WAL→DELETE 降级、FTS5 缺失降级、畸形 schema 自愈——这些是"现实路径"的兜底。一个能在 NFS、缺 FTS5 的 Python、被写坏的 DB 文件上都不丢数据、不崩溃的状态层,才配做一个"活在你每台设备上"的 Agent 的记忆。

## 本章小结

- Hermes 用单个 SQLite 数据库(`hermes_state.py`,实测 `197263` 字节)持久化所有会话与消息,设计目标是"WAL 多读单写"匹配多平台网关。
- 核心两表:`sessions`(约第 440 行)收纳身份/计量(token 各类计数)/计费(cost_*)/谱系(`parent_session_id` 自引用外键)/生命周期状态;`messages`(约第 477 行)用多列承接不同 provider 的 reasoning 格式,`active`/`observed` 做软标记支持逻辑删除。
- `DEFERRED_INDEX_SQL` 把依赖新列 `active` 的索引延后创建,让 schema 演进对老数据库平滑、不因 "no such column" 整体失败。
- 全文检索建两张 FTS5 表:基础 `messages_fts(content)` + 触发器自动同步(索引内容含 content/tool_name/tool_calls);中文专用 `messages_fts_trigram` 用 `tokenize='trigram'` 解决 unicode61 拆字问题。
- 查询路由按 token 粒度判断:CJK 且每个 token ≥3 字走 trigram,否则回退 `LIKE`(issue #20494 的教训);FTS5 缺失时用临时表探针检测并优雅降级关闭检索,会话存储照常。
- 写竞争靠"短超时(`timeout=1.0`)+ 应用层抖动重试"(`_WRITE_MAX_RETRIES=15`,`random.uniform` 20–150ms)绕开 SQLite busy handler 的车队效应;`BEGIN IMMEDIATE` 抢锁,每 50 次写做一次 `wal_checkpoint(TRUNCATE)`。
- `apply_wal_with_fallback` 在 NFS/SMB/FUSE 上按已知错误标记(`"locking protocol"` 等)降级 `WAL→DELETE`,降级前查磁盘头避免打架,WARNING 按 db_label 去重每进程一次。
- `repair_state_db_schema` 对畸形 sqlite_master(重复 FTS 对象)先备份再删 `messages_fts%` + VACUUM 后重开,实现自愈而非崩溃。

## 动手实验

1. **实验一:读懂会话表** —— `Read` `hermes_state.py` 约第 440 行的 `CREATE TABLE sessions`。把字段按"身份 / 计量 / 计费 / 谱系 / 生命周期"五类分组列出。思考:为什么 token 计数要分 `input`/`output`/`cache_read`/`cache_write`/`reasoning` 五种?这对应第 8 章 provider 返回的哪些信息?

2. **实验二:验证中文检索的路由** —— `Read` `hermes_state.py` 约第 3205 行起的 CJK 路由逻辑。回答:查询"大别山项目"(4 个连续中文字)走 trigram 还是 LIKE?查询"广西 OR 桂林"呢?为什么后者尽管总字数够、却仍要走 LIKE?把 issue #20494 的边界条件用自己的话复述一遍。

3. **实验三:模拟 WAL 降级** —— `Read` `apply_wal_with_fallback`(约第 157 行)与 `_WAL_INCOMPAT_MARKERS`(约第 54 行)。推演:如果 `PRAGMA journal_mode=WAL` 抛出 `OperationalError("disk I/O error")`(不在标记列表里),代码会降级还是抛出?如果抛 `"locking protocol"` 但磁盘头已是 wal 呢?各对应什么真实场景。

4. **实验四:理解抖动重试** —— `Read` `_execute_write`(约第 801 行)和类常量 `_WRITE_RETRY_MIN_S`/`_WRITE_RETRY_MAX_S`(约第 600 行)。解释:为什么用 `random.uniform` 随机退避、而不是固定间隔重试?用"车队效应"这个词论证随机抖动如何打破多写者同步退避的死循环。

> **下一章预告**:存下了会话与消息只是"记得发生过什么",真正让 Agent 变聪明的是从这些历史里**学习**。第 11 章将解构 Hermes 的闭环学习系统——它如何把对话沉淀成记忆(memory)、如何让 Agent 自己写技能(skills)、以及 honcho 这类外部记忆服务如何接进来,让 Hermes 不只是"能回忆",而是"会成长"。
