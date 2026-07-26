# 第 3 章　TurnEngine：心脏是如何跳动的

## 3.1　TurnEngine 的定位

`coworker/engine.py` 中的 `TurnEngine` 是 OpenWorker 的心脏。它的核心职责很纯粹：

> 生成 → 调工具 → 把结果回灌 → 再生成，循环直到模型不再调工具、触发了上限、或被用户中断。

关键设计决策：**它是异步的，但模型/工具调用是阻塞的**。TurnEngine 用 `asyncio` 管理事件循环和并发，但 provider 的 `stream()` 和工具的 `execute()` 是同步阻塞调用，被包在 `asyncio.to_thread()` 里。这样既保持了 UI 响应（流式 delta 实时推送），又不需要 provider SDK 支持 async。

## 3.2　主循环拆解：`_loop()`

```python
async def _loop(self) -> AsyncIterator[Event]:
    iterations = 0
    while True:
        if iterations >= self.max_iterations:
            yield Event(EventType.TURN_END, {"status": "max_iterations_exceeded", ...})
            return
        iterations += 1

        # 1. 流式获取模型响应
        turn: Optional[AssistantTurn] = None
        async for chunk in self._astream():
            if chunk.reasoning_delta:
                yield Event(EventType.REASONING_DELTA, ...)
            if chunk.text_delta:
                yield Event(EventType.ASSISTANT_DELTA, ...)
            if chunk.turn is not None:
                turn = chunk.turn

        # 2. 中断检查（流中被 stop）
        if self._cancel.is_set() and turn is None:
            self._append_notice("interrupted")
            yield Event(EventType.INTERRUPTED, ...)
            return

        # 3. 持久化 assistant 消息
        self.messages.append(_assistant_message(turn))

        # 4. 没有工具调用 → 回合结束
        if not turn.tool_calls:
            if self._steering:
                self._inject_steering()
                continue
            yield Event(EventType.TURN_END, {"status": "completed", ...})
            return

        # 5. 处理工具调用
        async for event in self._handle_tool_calls(turn.tool_calls):
            yield event

        # 6. 迭代结束检查
        if self._cancel.is_set():
            self._append_notice("interrupted")
            yield Event(EventType.INTERRUPTED, ...)
            return
        if self._steering:
            self._inject_steering()
```

### 关键常量与边界

- `max_iterations: int = 12`（默认 12 轮迭代上限，可在 config 中覆盖）
- `_cancel: asyncio.Event`（中断信号，`request_interrupt()` 设置）
- `_steering: list[tuple[str, Optional[dict]]]`（用户在回合进行中追加的消息）

## 3.3　流式桥接：`_astream()`

provider 的 `stream()` 是一个同步生成器。`_astream()` 在一个线程里跑它，通过 `asyncio.Queue` 把 chunk 推回事件循环：

```python
def produce():
    for chunk in provider.stream(model=..., messages=..., tools=...):
        if self._cancel.is_set():
            break
        loop.call_soon_threadsafe(queue.put_nowait, ("chunk", chunk))
    # ...
loop.run_in_executor(None, produce)
```

**中断设计**：queue 的消费者同时 `asyncio.wait` 两个 future--queue.get 和 `self._cancel.wait()`。如果 cancel 先就绪，直接返回（producer 下一个 chunk 检查到 cancel flag 也会退出）。这保证了即使 provider 的流式接口不支持取消，OpenWorker 也能在用户按 Stop 时尽快停止。

## 3.4　工具调用处理：`_handle_tool_calls()`

这是 OpenWorker 与其他 Agent 差异最大的地方之一。工具调用不是"直接执行"，而是经过一个精心设计的两阶段流程：

### 阶段 1：全量授权（sequential）

```python
for tool_call in tool_calls:
    if self._cancel.is_set():
        yield self._interrupted_tool(tool_call)  # 仍给一个 tool-error 结果
        continue
    yield Event(EventType.TOOL_PROPOSED, ...)
    # 三个特殊工具跳过权限路径：
    if tool_call.name == "request_directory": ...
    if tool_call.name == "propose_plan": ...
    if tool_call.name == "ask_user": ...
    # 普通工具走权限引擎
    async for item in self._authorize(tool_call):
        ...
    if allowed:
        cleared.append(tool_call)
```

### 阶段 2：分风险执行

```python
concurrent = [tc for tc in cleared if self._parallel_safe(tc)]
serial = [tc for tc in cleared if tc not in concurrent]

if concurrent:
    outcomes = await asyncio.gather(
        *[asyncio.to_thread(self._execute_sync, tc) for tc in concurrent]
    )

for tool_call in serial:
    result, status = await asyncio.to_thread(self._execute_sync, tool_call)
```

`_parallel_safe()` 检查的是 `metadata.risk_level == "low"` 且 `not metadata.requires_approval`--只有声明为低风险且不需要审批的工具（读文件、搜索等）才能并发执行。写入、Shell 和未标注的工具一律串行。

## 3.5　中断的完整性

OpenWorker 对中断的处理非常细致，核心原则是**"不留孤儿"**：

- **流中断**：`_astream()` 中的 cancel wait + producer 的 cancel flag 检查
- **工具中断**：`_interrupted_tool()` 给每个未执行的工具一个 `{"error": "tool call not executed", "reason": "interrupted by user"}` 结果，持久化到消息历史中
- **审批中断**：`_interruptible()` 方法让审批 await 可以被 cancel 打断，返回 `ApprovalOutcome.DENY`
- **中断后恢复**：`resume()` 方法处理 durable resume--从持久化消息中找出未回答的 tool_calls，重新执行

`_interruptible()` 的实现值得注意：

```python
async def _interruptible(self, coro, interrupted):
    task = asyncio.ensure_future(coro)
    cancel_wait = asyncio.ensure_future(self._cancel.wait())
    done, _ = await asyncio.wait({task, cancel_wait}, return_when=asyncio.FIRST_COMPLETED)
    if task in done:
        return task.result()
    task.cancel()
    return interrupted
```

## 3.6　Steering：回合中插话

`queue_steering(text, source)` 让用户在回合进行中追加消息。这些消息在两个检查点被注入：
1. 模型没有工具调用时（`_inject_steering()` 后 `continue` 再跑一轮模型）
2. 工具调用处理完毕后（`_inject_steering()` 后进入下一轮迭代）

注入的 steering 消息以 `{"role": "user", "content": text, "ts": time.time()}` 的形式追加到 `self.messages`。

## 3.7　Notice 消息：display-only 的持久化标记

OpenWorker 发明了一种特殊的消息类型--`notice`：

```python
def _append_notice(self, kind: str, text: Optional[str] = None) -> None:
    notice = {"role": "notice", "kind": kind, "ts": time.time()}
    if text:
        notice["text"] = text
    self.messages.append(notice)
```

`notice` 消息被持久化到对话历史中（所以重载后仍然可见），但 `_outbound_messages()` 在发送给 provider 之前会把它们全部剥离（`if msg.get("role") != "notice"`）。这解决了"错误/中断/模型切换"这些显示事件需要被持久化但不能被模型当作上下文的问题。

## 本章小结

TurnEngine 的设计可以用一句话概括：**阻塞的 provider + 异步的引擎 + 不留孤子的中断**。它不追求 OpenCode 那种类型化的信号系统，也不追求 Hermes 那种多重重试策略，而是用一个简洁的 `while True` + `asyncio.to_thread` 桥接 + 精心的 `_interruptible` 包装，实现了完整的异步循环、并发工具执行、和可恢复的中断。

## 动手实验

1. 在 `engine.py` 中找到 `_handle_tool_calls` 方法，追踪一个工具调用从 `TOOL_PROPOSED` 到 `TOOL_FINISHED` 的完整路径。
2. 修改 `max_iterations` 为 3，观察 Agent 在复杂任务中触发上限时的行为。
3. 找到 `_interruptible` 方法，思考它为什么用 `asyncio.wait` 而不是 `asyncio.gather`。
4. 在 `_outbound_messages` 中找到 `notice` 消息被剥离的那行代码，思考为什么 model_switch notice 不能被 provider 看到。
