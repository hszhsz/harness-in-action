# 第 8 章　Inbox 系统：可持久化的审批队列

## 8.1　问题：用户可能不在

传统的 Agent 审批是同步的：工具需要审批 -> 弹一个对话框 -> 用户点击 -> 继续。但当 Agent 运行在桌面上时，用户可能已经切走了、锁屏了、甚至关了电脑。

OpenWorker 用 **Inbox 系统**解决这个问题：审批不必当场回答，可以排队、可以持久化、可以从任何 surface 回答。

## 8.2　InboxItem：统一的人类注意力单元

```python
@dataclass
class InboxItem:
    id: str
    session_id: str
    kind: str  # approval / question / notification / directory / plan
    title: str
    body: str = ""
    state: str = STATE_PENDING  # pending -> resolved
    resolution: Optional[str] = None  # "allow" / "deny" / "always" / 自由文本
    visibility: str = VIS_INBOX  # inline（attended）vs inbox（unattended）
    tool_call_id: Optional[str] = None  # 幂等键
    options: list[str] = field(default_factory=list)  # 问题的选项
    data: dict[str, Any] = field(default_factory=dict)  # kind-specific payload
```

### 五种 kind

| kind | 来源 | 用途 |
|---|---|---|
| `approval` | 权限引擎的 `needs_user` | "Run `write_file`?" |
| `question` | `ask_user` 工具 | "Which PR should I review first?" |
| `notification` | 系统事件 | "Automation run completed" |
| `directory` | `request_directory` 工具 | "Grant access to /projects/foo?" |
| `plan` | `propose_plan` 工具 | "Approve this plan?" |

### 两种 visibility

| visibility | 何时使用 | 用户体验 |
|---|---|---|
| `inline` | 会话是 attended（用户在场） | 内联显示在会话 composer 中 |
| `inbox` | 会话是 unattended（用户离开） | 进入跨会话 Inbox 队列 |

**关键设计**：无论是 inline 还是 inbox，底层是**同一个持久化的、awaitable 的、resolve-from-anywhere 的记录**，只是可见性不同。

## 8.3　状态机：resolve-once, first-responder-wins

```
pending ──resolve()──> resolved
    │                      │
    │  (不可逆)              │  (幂等：后续 resolve 返回 False)
    │                      │
    └──wait()──────────────┘
```

`resolve(item_id, resolution)` 的实现：

```python
def resolve(self, item_id: str, resolution: str) -> bool:
    with self._lock:
        item = self._items.get(item_id)
        if item is None or item.state == STATE_RESOLVED:
            return False  # 幂等：first-responder-wins
        item.state = STATE_RESOLVED
        item.resolution = resolution
        item.resolved_at = _now()
        self._save()
    waiter = self._waiters.get(item_id)
    if waiter is not None:
        waiter.set()  # 唤醒 await 的 agent
    return True
```

**线程安全**：`threading.Lock` 保护状态转换，`asyncio.Event` 唤醒等待者。从任何 surface（GUI 内联、Inbox 列表、Slack 回复、重启后恢复）回答都是安全的。

## 8.4　幂等设计：tool_call_id

```python
def add(self, session_id, kind, title, *, tool_call_id=None, ...) -> InboxItem:
    if tool_call_id:
        existing = self.for_tool_call(session_id, tool_call_id)
        if existing is not None:
            return existing  # 复用已有的（可能已 resolved）
```

**为什么需要幂等**：当 TurnEngine 的 `resume()` 方法处理 durable resume 时，它会重新处理未回答的 trailing tool_calls。这些 tool_calls 已经有一个 Inbox item（在崩溃前创建），`resume()` 通过 `for_tool_call()` 找到它并直接 `await wait()`，如果它已经被回答了就立即返回，不会重新创建。

## 8.5　inbox_approver：把审批变成 Inbox item

`inbox.py` 中的 `inbox_approver()` 函数是一个 `Approver` 适配器：

```python
def inbox_approver(store: InboxStore, session_id: str, *, inbox: str = "default"):
    async def approve(request: PermissionRequest) -> ApprovalOutcome:
        item = store.add_approval(
            session_id,
            title=f"Run `{request.tool_name}`?",
            body=request.reason or "",
            inbox=inbox,
        )
        resolution = await store.wait(item.id)
        if resolution == "always":
            return ApprovalOutcome.ALWAYS_TOOL
        if resolution == "allow":
            return ApprovalOutcome.ONCE
        return ApprovalOutcome.DENY
    return approve
```

这把 TurnEngine 的 `await self.approver(...)` 变成了一个可持久化的挂起：Agent 的循环在 `store.wait(item.id)` 上挂起，即使进程重启，只要 Inbox 数据还在，`resume()` 就能重建这个挂起。

## 8.6　持久化

```python
class InboxStore:
    def _save(self) -> None:
        self.path.write_text(
            json.dumps({"items": [asdict(i) for i in self._items.values()]}, indent=2),
            encoding="utf-8",
        )
```

简单的 JSON 文件持久化。每次 `add()` 和 `resolve()` 都写一次文件。没有 SQLite，没有 WAL--因为 Inbox 的写入频率很低（一个会话一天可能几十次审批），JSON 的简单性胜过数据库的并发性能。

## 8.7　reconcile_on_resume：用户回来的恢复流程

当用户 resume 一个 unattended 会话时：

```python
def reconcile_on_resume(self, session_id: str) -> dict:
    pending = self.pending(session_id)
    recap = [i for i in self.list(session_id=session_id, state=STATE_RESOLVED)]
    return {"pending": [...], "recap": [...]}
```

- **pending**：还有待回答的审批/问题 -- 内联展示给用户
- **recap**：用户不在时已经被回答（由自动化规则或其他人）的条目 -- 展示回顾

这保证了用户回来后有一个统一的入口看到"我离开时发生了什么"。

## 8.8　resolve_session：会话删除时的清理

```python
def resolve_session(self, session_id: str, resolution: str = "session deleted") -> int:
    closed = 0
    for item in self.pending(session_id):
        if self.resolve(item.id, resolution):
            closed += 1
    return closed
```

当会话被删除时，所有 pending 的 Inbox item 都被 resolve 为 "session deleted"。这防止了孤儿审批永远挂在队列里。

## 本章小结

Inbox 系统是 OpenWorker 最独特的设计之一。它把审批从"同步对话框"变成了"持久化队列"，让 Agent 能在用户不在时继续工作（或优雅挂起）。核心设计是：resolve-once 状态机（first-responder-wins）、tool_call_id 幂等（durable resume 安全）、inline/inbox 双可见性（attended vs unattended 的同一记录）。

## 动手实验

1. 在 `inbox.py` 中找到 `InboxStore.add()` 方法，追踪一个重复的 tool_call_id 如何被去重。
2. 在 `engine.py` 中找到 `resume()` 方法，看它如何重建 durable resume 的挂起状态。
3. 打开 `inbox_approver()` 函数，思考为什么它不直接创建 Inbox item 而是先检查 `for_tool_call()`。
4. 调用 `reconcile_on_resume(session_id)` API，观察返回的 pending 和 recap 结构。
