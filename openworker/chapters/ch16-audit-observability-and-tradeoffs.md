# 第 16 章　审计、可观测性与工程权衡

## 16.1　AuditStore：本地审计日志

`coworker/audit.py` 的 `AuditStore` 是一个 SQLite 审计日志，记录每个工具调用的完整生命周期。

### Schema

```sql
CREATE TABLE audit_events (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp TEXT DEFAULT CURRENT_TIMESTAMP,
    session_id TEXT,
    agent TEXT,
    workspace TEXT,
    connector TEXT,
    tool TEXT,
    stage TEXT,         -- proposed / started / auto_allowed / approval_requested / approval_resolved / finished / filtered
    status TEXT,        -- ok / error / denied / interrupted
    approval TEXT,      -- once / always_tool / always_command / deny
    args TEXT,          -- sanitized JSON
    result_preview TEXT,
    reason TEXT,
    resource TEXT       -- url / owner / repo / issue_key / page_id 等
)
```

### 审计事件

TurnEngine 的 `_audit()` 方法在每个工具调用的关键节点写入审计事件：

```python
def _audit(self, tool_call, **event) -> None:
    payload = {
        **self.audit_context,  # session_id, agent, workspace
        "tool": tool_call.name,
        "arguments": tool_call.arguments,
        **event,  # stage, status, approval, reason, result_preview
    }
    self.audit_sink(payload)
```

调用点：
- `proposed`：工具被提出（TOOL_PROPOSED 时）
- `auto_allowed`：被 standing rule 自动允许
- `approval_requested`：需要用户审批
- `approval_resolved`：用户已回答
- `started`：工具开始执行
- `finished`：工具执行完成
- `filtered`：结果被隐私过滤器处理

### 敏感数据脱敏

`_sanitize_args()` 函数在写入前对参数进行脱敏：

```python
_SECRET_KEYS = ("token", "secret", "password", "api_key", "access_token", "bot_token", "app_token", "raw")
_BODY_KEYS = ("body", "content", "html")

def _sanitize_args(tool, args):
    for key, value in args.items():
        lk = str(key).lower()
        if any(s in lk for s in _SECRET_KEYS):
            out[key] = "[redacted]"
        elif tool == "browser_type" and lk == "text":
            out[key] = "[redacted input]"
        elif any(b == lk for b in _BODY_KEYS):
            out[key] = "[redacted body]"
        else:
            out[key] = _summarize(value)
```

**两层脱敏**：key 名匹配（`_SECRET_KEYS`）+ 值截断（`_truncate` 500 字符）。`browser_type` 工具的 `text` 参数被特殊处理（防止用户的表单输入被审计日志记录）。

### Resource 提取

`_resource()` 函数从参数或结果中提取一个"资源标识符"：

```python
def _resource(tool, args, result) -> str:
    for key in ("url", "owner", "repo", "issue_key", "page_id", "ticket_id", "calendar_id", "message_id"):
        if args.get(key):
            return str(args[key])
```

这让审计日志可以按资源（如某个 GitHub repo 或某个 Jira ticket）过滤，而不仅仅是按工具名。

## 16.2　Event 系统：Engine 与 Surface 的契约

`coworker/events.py` 定义了 TurnEngine 与所有 surface（GUI/TUI/IDE/消息平台）之间的事件契约：

```python
class EventType(str, Enum):
    TURN_START = "turn_start"
    ASSISTANT_DELTA = "assistant_delta"       # 流式文本 delta
    REASONING_DELTA = "reasoning_delta"       # 思考文本 delta
    ASSISTANT_MESSAGE = "assistant_message"   # 完整 assistant 消息
    TOOL_PROPOSED = "tool_proposed"           # 工具被提出
    PERMISSION_REQUIRED = "permission_required"
    DIRECTORY_REQUESTED = "directory_requested"
    QUESTION_REQUESTED = "question_requested"
    PLAN_PROPOSED = "plan_proposed"
    TOOL_STARTED = "tool_started"
    TOOL_FINISHED = "tool_finished"
    ITERATION_END = "iteration_end"
    TURN_END = "turn_end"
    ERROR = "error"
    INTERRUPTED = "interrupted"
```

### 事件流

一个完整的 turn 的事件流：

```
TURN_START
  -> ASSISTANT_DELTA (x N)         # 流式文本
  -> REASONING_DELTA (x N)         # 思考文本（如果有）
  -> ASSISTANT_MESSAGE              # 完整消息
  -> TOOL_PROPOSED (x N)           # 工具调用被提出
  -> PERMISSION_REQUIRED?          # 需要审批时
  -> TOOL_STARTED                  # 工具开始执行
  -> TOOL_FINISHED                 # 工具执行完成
  -> ITERATION_END                 # 一轮迭代结束
  -> [回到 ASSISTANT_DELTA]        # 下一轮迭代
  -> TURN_END                      # 回合结束
```

### WebSocket 推送

事件通过 WebSocket 实时推送到 GUI：

```python
# server/app.py
@app.websocket("/ws/sessions/{session_id}")
async def ws_session(ws, session_id):
    async for event in engine.run(user_input):
        await ws.send_json({"type": event.type.value, "data": event.data})
```

GUI 根据事件类型渲染不同的 UI 元素：流式文本、工具卡片、审批提示等。

## 16.3　可观测性的层次

OpenWorker 的可观测性分为四层：

| 层 | 机制 | 用途 |
|---|---|---|
| **实时事件** | Event + WebSocket | 用户实时看到 Agent 在做什么 |
| **审计日志** | AuditStore (SQLite) | 事后审查 Agent 做了什么 |
| **对话历史** | JSONL + SQLite | 完整的对话记录 |
| **Inbox 记录** | JSON (InboxStore) | 审批/问题的完整记录 |

## 16.4　工程权衡

OpenWorker 在整个设计中有几个值得讨论的工程权衡：

### 权衡 1：阻塞 Provider vs 异步引擎

**选择**：Provider API 是阻塞的，引擎用 `asyncio.to_thread` 桥接。
**代价**：每个 provider 调用占用一个线程池线程。
**收益**：引擎的事件循环不被 provider SDK 的实现细节污染；provider SDK 不需要支持 async。
**结论**：合理。provider 调用是 IO 密集的，线程池的开销远小于一次 LLM 调用的延迟。

### 权衡 2：JSONL vs SQLite for 消息

**选择**：消息存 JSONL，元数据存 SQLite。
**代价**：两个存储需要同步（`_backfill_counts`）；消息内查询能力有限。
**收益**：追加写入极简（不需要 SQL）；人类可读；无锁竞争。
**结论**：合理。会话内的查询需求很少（大部分是整体加载）。

### 权衡 3：JSON vs SQLite for Inbox

**选择**：Inbox 存 JSON 文件。
**代价**：每次 add/resolve 都全量重写整个 JSON；没有查询能力（只能遍历）。
**收益**：实现极简（< 100 行代码）；不需要初始化数据库。
**结论**：合理。Inbox 的数据量很小（一个会话几十条），全量重写的开销可以忽略。

### 权衡 4：四档风险分类 vs 细粒度权限

**选择**：四档（read/write_local/exec/external）而非每工具独立配置。
**代价**：同档工具被同等对待（如所有 EXTERNAL 工具走同一个审批路径）。
**收益**：分类逻辑极简（一个 `classify()` 函数）；用户认知负担低（四档而非每工具配置）。
**结论**：合理。standing rules 的精确目标绑定弥补了粒度不足的问题。

### 权衡 5：没有 OS 级沙箱

**选择**：不像 Codex 那样做 Landlock/Seatbelt 沙箱，而是靠路径约束 + 命令黑名单 + 审批。
**代价**：Agent 能执行的代码没有 OS 级隔离。
**收益**：桌面应用不需要虚拟化/容器，用户体验更流畅。
**结论**：**桌面端定位决定的**。Codex 要在云端跑不可信代码，OpenWorker 在用户本机跑，用户可以随时看屏幕/按 Stop。

### 权衡 6：aisuite 依赖

**选择**：基于 aisuite 而非自己造框架。
**代价**：aisuite 的 API 设计影响 OpenWorker 的工具接口；aisuite 的演进速度影响 OpenWorker。
**收益**：不重复造轮子；aisuite 的 provider 覆盖直接可用。
**结论**：合理。OpenWorker 的 README 明确说"如果你要建自己的 harness，从 aisuite 开始"。

## 16.5　与其他 Agent 的对比

| 维度 | OpenWorker | OpenClaw | Hermes | Codex | OpenCode |
|---|---|---|---|---|---|
| 语言 | Python | TypeScript | Python | Rust | TypeScript+Effect |
| 定位 | 桌面端 | CLI | 7×24 服务器 | CLI | CLI |
| 沙箱 | 工具层 | 工具层 | 6 种后端 | OS 级多后端 | 工具层 |
| 审批 | Inbox 持久化 | 策略流水线 | 模式匹配 | 5 级 + execpolicy | permission.ts |
| 持久化 | SQLite+JSONL | SQLite | SQLite+FTS5 | SQLite | SQLite+Event Sourcing |
| 记忆 | 三级作用域 | AGENTS.md | MEMORY.md | AGENTS.md | system-context |
| 自动化 | Scheduler+selfwake | heartbeat/cron | cron | 无 | 无 |
| 连接器 | 25+ | channels | 10+ IM | 无 | 无 |
| MCP | ✅ | ✅ | ✅ | ✅ | ✅ |

## 本章小结

OpenWorker 的可观测性用"实时事件 + 审计日志 + 对话历史 + Inbox 记录"四层覆盖了 Agent 运行的所有侧面。工程权衡的核心是"务实"--每一步都选择最简单够用的方案（JSONL for 消息、JSON for Inbox、四档 for 风险分类），但在关键安全点（审批持久化、路径约束、Shell 命令黑名单）上绝不含糊。

## 动手实验

1. 在 `audit.py` 中找到 `_sanitize_args()`，看 `browser_type` 工具的 `text` 参数为什么被特殊处理。
2. 在 `events.py` 中找到 `EventType` 枚举，画出一个包含审批的完整 turn 的事件时序图。
3. 在 `engine.py` 中找到所有调用 `_audit()` 的位置，列出每个 stage 的含义。
4. 调用 `GET /v1/inbox` API，观察 Inbox 中的 item 结构，找到 `tool_call_id` 字段。
