# 第 9 章　无人值守与持久化恢复

## 9.1　无人值守模式：Agent 的"夜间值班"

OpenWorker 的 `unattended.py` 管理一种特殊模式：**用户离开，但 Agent 继续工作**。这和 Hermes 的 7×24 运行模式类似，但 OpenWorker 的实现更精细--它不是"永远运行"，而是"在特定会话中切换到无人值守"。

### 切换逻辑

```python
# server/app.py
@app.post("/v1/sessions/{session_id}/unattended")
def set_unattended(session_id: str, body: dict) -> dict:
    on = bool(body.get("unattended"))
    manager.unattended.set(session_id, on)
    return {"ok": True, "session_id": session_id, "unattended": on}
```

当切换到 unattended 时：
1. 新的审批请求从 `VIS_INLINE` 变为 `VIS_INBOX`
2. 这些审批进入跨会话 Inbox 队列
3. 用户可以从 Inbox 列表（或 Slack/通知）回答

### 会话级 vs 全局

`UnattendedStore` 是会话级别的：每个会话独立切换。这意味着用户可以让一个自动化任务无人值守，同时在另一个会话中正常交互。

## 9.2　持久化恢复：durable resume 的完整链路

OpenWorker 的 durable resume 是一个精心设计的链路，涉及多个组件：

### 步骤 1：持久化（运行时）

`ConversationStore.save()` 在每轮结束后把消息追加到 `.jsonl` 文件：

```python
def save(self, record: SessionRecord) -> None:
    existing = self._count(sid)
    if len(record.messages) > existing:
        self._append(sid, record.messages[existing:])  # 只追加新消息
```

Inbox item 也在每次 `add()` / `resolve()` 时写入 JSON 文件。

### 步骤 2：重建（重启后）

`SessionManager` 在启动时加载所有会话。当一个会话被打开时，`build_engine()` 用持久化的 messages 构建一个新的 `TurnEngine`。

### 步骤 3：发现未完成的工具调用

`TurnEngine.resume()` 的 `_unanswered_trailing_tool_calls()` 方法从持久化的消息历史中找出最后一条 assistant 消息中**没有对应 tool 结果**的 tool_calls：

```python
def _unanswered_trailing_tool_calls(self) -> list[ToolCall]:
    answered = {m.get("tool_call_id") for m in self.messages if m.get("role") == "tool"}
    for msg in reversed(self.messages):
        if msg.get("role") == "user":
            return []  # 遇到 user 消息就停
        if msg.get("role") == "assistant" and msg.get("tool_calls"):
            out = []
            for tc in msg["tool_calls"]:
                if tc.get("id") in answered:
                    continue  # 已有结果，跳过
                out.append(ToolCall(...))
            return out
    return []
```

### 步骤 4：重建挂起的审批

对于每个未回答的 tool_call，TurnEngine 检查 Inbox 是否有对应的 item：

```python
# inbox.py
def for_tool_call(self, session_id, tool_call_id) -> Optional[InboxItem]:
    for i in self._items.values():
        if i.session_id == session_id and i.tool_call_id == tool_call_id:
            return i
    return None
```

如果 item 已经被 resolve（比如用户在手机上回答了），`inbox_approver` 的 `await store.wait(item.id)` 立即返回结果。如果还没回答，TurnEngine 重新挂起等待。

### 步骤 5：继续执行

```python
async def resume(self) -> AsyncIterator[Event]:
    pending = self._unanswered_trailing_tool_calls()
    if not pending:
        return
    self._cancel.clear()
    yield Event(EventType.TURN_START, {"input": "(resumed)"})
    async for event in self._handle_tool_calls(pending):
        yield event
    yield Event(EventType.ITERATION_END, {"iteration": 0})
    if not self._cancel.is_set():
        async for event in self._loop():
            yield event
```

先处理完挂起的工具调用，然后继续主循环。对用户来说，就像什么都没发生过一样。

## 9.3　retry：错误后的重试

```python
async def retry(self) -> AsyncIterator[Event]:
    if not self._tail_is_retriable_error():
        return
    self._cancel.clear()
    yield Event(EventType.TURN_START, {"input": ""})
    async for event in self._loop():
        yield event
```

`_tail_is_retriable_error()` 检查消息历史尾部是否是 error notice（跳过 model_switch notice）。这保证了 retry 只在确实出错时才重跑，不会在已完成的会话上画蛇添足。

**model_switch + retry 的组合**：用户可以先切换模型（追加 model_switch notice），再 retry（跳过 model_switch notice 找到 error notice）。这是"切换模型后重试"的恢复路径。

## 9.4　notice 消息：持久化但不可见

`_append_notice()` 创建的 notice 消息是 OpenWorker 的独特设计：

- **持久化**：写入 JSONL，重启后仍然可见
- **display-only**：`_outbound_messages()` 在每次 provider 调用时剥离所有 `role == "notice"` 的消息
- **kind 标记**：`error`、`interrupted`、`model_switch` 三种

这解决了"错误/中断/模型切换这些事件需要被持久化到对话历史中（否则重载后用户看不到发生了什么），但不能被模型当作上下文"的矛盾。

## 9.5　grants 的持久化

Session 级别的审批授权（"always allow this tool"）被持久化在 `SessionRecord.grants` 中：

```python
@dataclass
class SessionRecord:
    grants: dict[str, Any] = field(default_factory=dict)
    # 结构：{"tools": [...], "commands": [...]}
```

在 `build_engine()` 时，这些 grants 被恢复到 `PermissionEngine` 的 `session_allow_tools` 和 `session_allow_commands` 中。所以用户重启后不需要重新授权。

## 本章小结

OpenWorker 的无人值守和持久化恢复构成了一个完整的"桌面 Agent 7×24"方案：unattended 模式让审批自动进入跨会话队列、durable resume 让崩溃/重启后能续跑、notice 消息让显示事件持久化但不污染模型上下文、grants 持久化让授权在重启后保持。整个链路的核心是 `tool_call_id` 的幂等设计和 Inbox 的 resolve-once 状态机。

## 动手实验

1. 在 `engine.py` 中找到 `resume()` 方法，追踪一个崩溃后的恢复路径。
2. 在 `conversations.py` 中找到 `_append()` 方法，观察 JSONL 的追加写入。
3. 在 `sessions.py` 中找到 `grants` 字段，在 `build_engine()` 中搜索它被恢复到 PermissionEngine 的代码路径。
4. 切换一个会话到 unattended 模式，触发一个审批，观察它如何出现在 Inbox 中。
