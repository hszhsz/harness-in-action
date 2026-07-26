# 第 12 章　记忆系统：三级作用域与 SQLite 持久化

## 12.1　记忆的定位

OpenWorker 的记忆系统不是"对话历史"（那由 ConversationStore 管理），而是**跨会话的持久化事实**：用户的偏好、项目的上下文、过去会话中学到的教训。

`coworker/memory/` 目录有三个文件：

| 文件 | 职责 |
|---|---|
| `base.py` | `MemoryStore` ABC + `MemoryItem` + `Scope` |
| `sqlite_store.py` | SQLite 实现 |
| `tools.py` | 暴露给 Agent 的记忆工具 |

## 12.2　三级作用域

```python
class Scope(str, Enum):
    GLOBAL = "global"       # 用户级：所有项目共享
    WORKSPACE = "workspace"  # 项目级：绑定到特定 workspace
    SESSION = "session"      # 会话级：临时记忆（当前未使用）
```

### 全局记忆 vs 工作区记忆

- **全局**：用户的通用偏好（"我更喜欢 Python 而不是 Ruby"）
- **工作区**：项目特定的上下文（"这个项目的测试用 pytest 跑"）

在 `build_engine()` 中，两个作用域的记忆都被加载：

```python
remembered = memory_store.list(scope=Scope.GLOBAL)
if ws is not None:
    remembered += memory_store.list(scope=Scope.WORKSPACE, workspace=str(ws))
block = format_memories(remembered)
if block:
    instructions = f"{instructions}\n\n{block}"
```

记忆被格式化为 system prompt 的一部分：

```python
def format_memories(items: list[MemoryItem]) -> str:
    lines = [f"- [#{item.id}] {item.content}" for item in items]
    return "Known memories (from earlier sessions):\n" + "\n".join(lines)
```

每条记忆前缀 `[#id]`，让 Agent 可以通过 `memory_update(id)` 或 `memory_forget(id)` 精确修改。

## 12.3　MemoryItem

```python
@dataclass
class MemoryItem:
    id: int
    scope: Scope
    content: str
    key: Optional[str] = None       # 可选的去重键
    workspace: Optional[str] = None  # WORKSPACE 作用域的工作区路径
    session_id: Optional[str] = None  # SESSION 作用域的会话 ID
    created_at: Optional[str] = None
```

## 12.4　SQLiteMemoryStore

```python
class SQLiteMemoryStore(MemoryStore):
    def __init__(self, path: str | Path) -> None:
        self._conn = sqlite3.connect(self.path, check_same_thread=False)
        self._conn.row_factory = sqlite3.Row
        self._conn.execute("""
            CREATE TABLE IF NOT EXISTS memories (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                scope TEXT NOT NULL,
                key TEXT,
                content TEXT NOT NULL,
                workspace TEXT,
                session_id TEXT,
                created_at TEXT DEFAULT CURRENT_TIMESTAMP
            )
        """)
```

**线程安全**：用 `threading.RLock` 串行化所有数据库操作。`check_same_thread=False` 因为 server 的 WS handler 和 store 的创建者可能在不同的线程。

### 增删改查

- `add(content, scope, key, workspace, session_id)` -> `MemoryItem`
- `get(item_id)` -> `Optional[MemoryItem]`
- `list(scope, workspace, session_id)` -> `list[MemoryItem]`
- `update(item_id, content)` -> `Optional[MemoryItem]`
- `delete(item_id)` -> `bool`

## 12.5　记忆工具

`memory/tools.py` 暴露了四个工具给 Agent：

| 工具 | 功能 | 风险分类 |
|---|---|---|
| `remember` | 保存一条新记忆 | WRITE_LOCAL（有副作用） |
| `memory_update` | 修改已有记忆 | WRITE_LOCAL |
| `memory_forget` | 删除记忆 | WRITE_LOCAL |
| `memory_list` | 列出当前记忆 | READ |

### 记忆指导原则

`agent.py` 中的 `_MEMORY_GUIDANCE` 被注入到 system prompt：

```
Memory:
- You have persistent memory across sessions. Use `remember` for durable facts:
  the user's corrections and stated preferences (include the why), and project
  context you couldn't rederive from the code. Don't save what the repo already
  records (code structure, git history, AGENTS.md) or details that only matter
  to the current task. Use absolute dates, never "yesterday".
- Before saving, check the known-memories list: if an entry already covers it,
  revise that entry with `memory_update` instead of adding a near-duplicate;
  retire wrong or obsolete entries with `memory_forget`.
- Memories reflect when they were written. If one names a file, flag, or URL,
  verify it still exists before relying on it.
```

这段指导防止了两种常见的记忆滥用：1）保存代码已经记录的信息（git 历史、AGENTS.md），2）保存只对当前任务有意义的临时信息。

## 12.6　记忆与 Skill 的区别

| 维度 | Memory | Skill |
|---|---|---|
| 形式 | 自由文本 | Markdown + frontmatter |
| 作用 | 注入到 system prompt | 渐进式披露（按需加载） |
| 谁写 | Agent 自己写 | 用户手写或 Agent 生成 |
| 何时用 | 跨会话事实 | 可复用的操作流程 |

Memory 是"Agent 的长期记忆"，Skill 是"Agent 的技能手册"。

## 12.7　记忆安全

记忆是从用户数据中提取的，但用户数据可能包含恶意内容（提示注入）。OpenWorker 的防御策略是**记忆指导原则**--明确告诉模型"记忆反映的是写入时的情况，如果它引用了文件/URL，先验证它还存在"。这比 Hermes 的 `scan_for_threats` 更轻量，但也更不安全。这是一个工程权衡。

## 本章小结

OpenWorker 的记忆系统用"三级作用域 + SQLite 持久化 + 四个工具"的简洁设计，实现了跨会话的持久化记忆。最值得借鉴的是：`[#id]` 前缀让 Agent 可以精确修改/删除特定记忆，`_MEMORY_GUIDANCE` 防止了常见的记忆滥用，以及 `key` 字段提供的可选去重能力。

## 动手实验

1. 在 `memory/base.py` 中找到 `format_memories()`，看记忆如何被格式化进 system prompt。
2. 在 `memory/sqlite_store.py` 中找到 `list()` 方法，看 scope 和 workspace 的过滤逻辑。
3. 在 `agent.py` 中找到 `_MEMORY_GUIDANCE` 常量，思考每条指导原则防止了什么滥用。
4. 让 Agent 保存一条记忆，重启会话，验证记忆是否被恢复到 system prompt 中。
