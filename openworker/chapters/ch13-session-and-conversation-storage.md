# 第 13 章　会话与对话存储：JSONL + SQLite 双层

## 13.1　存储分层

OpenWorker 的会话存储分为两层：

1. **SQLite 索引**（`coworker.db`）：会话元数据（ID、workspace、model、mode、title、更新时间等）
2. **JSONL 消息日志**（`conversations/<id>.jsonl`）：每个会话一个文件，append-only

这个设计的核心思想是：**元数据需要查询（用 SQLite），消息需要追加（用 JSONL）**。

## 13.2　ConversationStore

`coworker/conversations.py` 是存储层的入口：

```python
class ConversationStore:
    def __init__(self, base_dir: str | Path) -> None:
        self.base = Path(base_dir).expanduser()
        self.conv_dir = self.base / "conversations"
        self.db_path = self.base / "coworker.db"
        self._conn = sqlite3.connect(self.db_path, check_same_thread=False)
```

### SQLite Schema

```sql
CREATE TABLE sessions (
    session_id TEXT PRIMARY KEY,
    workspace TEXT,
    model TEXT,
    mode TEXT,
    title TEXT,
    agent TEXT DEFAULT 'code',
    n_msgs INTEGER DEFAULT 0,
    messages TEXT,  -- 已迁移到 JSONL，此列保留用于兼容
    extra_roots TEXT,  -- JSON array
    grants TEXT,  -- JSON: {"tools": [...], "commands": [...]}
    pinned INTEGER DEFAULT 0,
    archived INTEGER DEFAULT 0,
    origin TEXT,  -- 来源（如 "slack"）
    origin_label TEXT,
    auto_title TEXT,
    renamed INTEGER DEFAULT 0,
    updated_at TEXT DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE workspaces (
    path TEXT PRIMARY KEY,
    last_used TEXT DEFAULT CURRENT_TIMESTAMP
);
```

### 追加写入

```python
def _append(self, sid: str, messages: list[dict]) -> None:
    with open(self._file(sid), "a", encoding="utf-8") as f:
        for m in messages:
            f.write(json.dumps(m) + "\n")
```

`save()` 只追加新消息：

```python
def save(self, record: SessionRecord) -> None:
    existing = self._count(sid)
    if len(record.messages) > existing:
        self._append(sid, record.messages[existing:])
```

## 13.3　SessionRecord

```python
@dataclass
class SessionRecord:
    session_id: str
    workspace: str
    model: str
    mode: str
    messages: list[dict[str, Any]] = field(default_factory=list)
    title: Optional[str] = None
    agent: str = "code"
    message_count: int = 0
    updated_at: Optional[str] = None
    extra_roots: list[dict[str, Any]] = field(default_factory=list)
    grants: dict[str, Any] = field(default_factory=dict)
    pinned: bool = False
    archived: bool = False
    origin: Optional[str] = None       # 来源平台
    origin_label: Optional[str] = None  # 来源显示名
```

`origin` 和 `origin_label` 是 Slack/Telegram 等消息平台触发的会话的来源标记。它们在 spawn 时设置一次，后续 `save()` 不会覆盖（pinned 机制）。

## 13.4　标题管理：三层优先级

```python
def _display_title(row: sqlite3.Row) -> Optional[str]:
    if row["renamed"]:
        return row["title"]  # 用户手动命名
    return row["auto_title"] or row["title"]  # 自动生成 或 首条消息
```

三层优先级：
1. **手动命名**（`renamed=1`）：用户在 UI 中重命名，最终
2. **自动生成**（`auto_title`）：LLM 生成的标题
3. **首条消息**（`title`）：保存时的 `title_from(messages)`

`set_auto_title()` 只在 `renamed=0` 时写入，不影响 `updated_at`（所以标题更新不会重排会话列表）。

## 13.5　工作区管理

```python
def touch_workspace(self, path: str) -> None:
    self._conn.execute(
        "INSERT INTO workspaces (path, last_used) VALUES (?, CURRENT_TIMESTAMP) "
        "ON CONFLICT(path) DO UPDATE SET last_used = CURRENT_TIMESTAMP",
        (path,),
    )
```

`recent_workspaces()` 返回最近使用的工作区列表。`canonicalize_workspaces()` 把路径规范化（`os.path.realpath`），消除符号链接导致的重复。

## 13.6　数据迁移：从 inline 到 JSONL

`_backfill_counts()` 是一个一次性迁移函数：

```python
def _backfill_counts(self) -> None:
    for row in rows:
        sid = row["session_id"]
        if jsonl.exists() and row["title"] and row["n_msgs"]:
            continue  # 已迁移
        if row["messages"]:  # 旧的 inline blob
            messages = json.loads(row["messages"])
            self._append(sid, messages)  # 写入 JSONL
            self._conn.execute("UPDATE sessions SET messages = NULL ...")  # 清空 inline
```

这是一个"启动时检查并迁移"的 lazy migration 模式--不需要一次性迁移所有数据，每次启动时检查并迁移未处理的行。

## 13.7　JSONL 的优势

选择 JSONL 而非 SQLite 存储消息的原因：

1. **Append-only 天然契合**：每轮只是追加新消息，不修改旧消息
2. **人类可读**：可以直接 `cat conversations/<id>.jsonl` 查看对话
3. **无锁竞争**：追加写入不阻塞读取
4. **简单备份**：复制文件即完成备份

缺点是**没有查询能力**--但会话内的查询需求很少（大部分是按 session_id 整体加载）。SQLite 索引提供了跨会话的查询能力（列表、过滤、排序）。

## 13.8　与 Inbox 的关系

Inbox 有自己的持久化（JSON 文件，不是 SQLite），因为：
- 写入频率更低（每次审批才写一次）
- 数据量更小（一个会话可能只有几十条 Inbox item）
- 需要被独立加载（Inbox 列表不应该依赖会话存储）

## 本章小结

OpenWorker 的会话存储用 "SQLite 索引 + JSONL 消息日志" 的双层设计，把元数据查询和消息追加分离到最适合各自的存储引擎。最值得借鉴的是：append-only 写入策略（只追加新消息）、三层标题优先级（手动 > 自动 > 首条消息）、lazy migration 模式（启动时检查并迁移）、以及 `origin`/`origin_label` 的 pinned 机制（来源标记一次设置不被覆盖）。

## 动手实验

1. 在 `conversations.py` 中找到 `save()` 方法，追踪一个新消息从 `TurnEngine.messages` 到 JSONL 的路径。
2. 找到 `_backfill_counts()`，理解 inline blob 到 JSONL 的迁移逻辑。
3. 在 `sessions.py` 中找到 `SessionRecord` 的 `origin` 和 `origin_label` 字段，在 `conversations.py` 中找到 `set_origin()`，思考为什么它在 `save()` 中不被覆盖。
4. 调用 `GET /v1/sessions` API，观察返回的会话列表是如何排序的。
