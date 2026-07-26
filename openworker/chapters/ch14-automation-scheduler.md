# 第 14 章　自动化调度：Cron、自唤醒与重叠保护

## 14.1　自动化系统的角色

OpenWorker 的自动化系统让 Agent 能在用户不在时主动工作：

- **定时任务**：每天早上 8 点生成一份简报
- **频道监听**：监听 #support 频道，有新消息时触发 Agent
- **自唤醒**：Agent 可以挂起自己，设定在某个事件发生后恢复

`coworker/automation/` 目录包含调度器的核心：

| 文件 | 职责 |
|---|---|
| `models.py` | `ScheduledTask` / `TaskRun` 数据类 |
| `store.py` | `TaskStore`：SQLite 持久化 |
| `scheduler.py` | `Scheduler`：定时循环 + 重叠保护 |
| `tools.py` | 暴露给 Agent 的调度工具 |

## 14.2　ScheduledTask 模型

```python
@dataclass
class ScheduledTask:
    id: str
    name: str
    cron: str            # cron 表达式
    prompt: str          # 触发时发给 Agent 的 prompt
    workspace: str       # 工作目录
    agent: str = "cowork"  # 使用的 Agent
    model: str = ""      # 使用的模型
    mode: str = "auto"   # 运行模式
    enabled: bool = True
    run_count: int = 0
    last_run: Optional[str] = None
    last_status: Optional[str] = None
    next_run: Optional[str] = None
    # 来源标记
    origin: dict = field(default_factory=dict)
    # Task-scoped standing rules（§25）
    targets: dict[str, list[str]] = field(default_factory=dict)
```

`targets` 字段是 standing rules 的种子：自动化任务可以声明"允许给 #general 频道发消息"，这样无人值守运行时不需要每次审批。

## 14.3　Scheduler：定时循环

```python
class Scheduler:
    def __init__(self, store: TaskStore, runner: Runner, *, tick_seconds: float = 30.0):
        self.tick_seconds = tick_seconds
        self._running_ids: set[str] = set()  # 重叠保护

    async def _loop(self) -> None:
        await self._tick(trigger="catchup")  # 启动时追赶
        while True:
            await asyncio.sleep(self.tick_seconds)
            await self._tick(trigger="schedule")
```

### 两个关键策略

1. **run-once-catch-up**：启动时的第一次 tick 用 `trigger="catchup"`，把宕机期间错过的任务补跑一次。不是补跑每一次（如果一个每小时运行的任务宕机了 3 小时，只补跑一次，不是三次）。

2. **skip-on-overlap**：`_running_ids` 集合防止同一个任务的重叠运行。如果一个任务还在跑（可能挂在审批上），下一次 tick 会跳过它。

### spawn 而非 await

```python
async def _tick(self, *, trigger: str) -> None:
    for task in self.store.due():
        spawned = asyncio.create_task(self.run_task(task, trigger=trigger))
        self._spawned.add(spawned)
        spawned.add_done_callback(self._spawned.discard)
```

任务被 `asyncio.create_task` spawn 而非 await --一个被审批挂起的任务不会阻塞调度器或其他到期任务。

## 14.4　Self-wake：Agent 的自暂停与恢复

`coworker/selfwake.py` 是 OpenWorker 独特的设计：Agent 可以**挂起自己**，设定在某个条件满足时恢复。

### 三种唤醒触发器

1. **timer**：定时恢复（如"30 分钟后叫我"）
2. **on-completion**：某个操作完成后恢复（如"等 CI 跑完"）
3. **on-event**：某个事件发生时恢复（如"有新的 Slack 消息"）

### 工具

```python
# selfwake.py
def selfwake_tools(wake_store, session_id):
    return [
        suspend_self,       # 挂起当前会话
        schedule_wake,      # 安排一个唤醒
        cancel_wake,         # 取消唤醒
    ]
```

### 与调度器的关系

Scheduler 的 `extra_tick` 回调每 30 秒检查一次是否有到期的 wake：

```python
# server/manager.py 中
scheduler = Scheduler(
    store=task_store,
    runner=runner,
    extra_tick=wake_store.check_due,
)
```

到期的 wake 触发会话的 `resume()`，TurnEngine 从挂起点继续执行。

## 14.5　自动化工具

`automation/tools.py` 暴露了以下工具给 Agent：

| 工具 | 功能 |
|---|---|
| `create_task` | 创建一个定时任务 |
| `list_tasks` | 列出当前工作区的任务 |
| `update_task` | 修改任务（cron、prompt 等） |
| `delete_task` | 删除任务 |
| `toggle_task` | 启用/禁用任务 |

### 限制

自动化工具只对 `knowledge` family 的 Agent 注册：

```python
if task_store is not None and ws is not None and agent.family == "knowledge":
    registry.register_all(scheduling_tools(task_store, origin=origin, ...))
```

Code agent 不暴露这些工具--它通过 `explorer_tools` 做只读子 Agent 委派，不需要定时任务。

## 14.6　无人值守自动化任务的审批

当一个自动化任务在 unattended 模式下运行时，审批通过 Inbox 处理：

1. 任务触发，Agent 开始工作
2. Agent 调用一个需要审批的工具（如 `send_message`）
3. `inbox_approver` 创建一个 `visibility=VIS_INBOX` 的 Inbox item
4. Agent 挂起等待
5. 用户在 Inbox 中回答（可能从手机、Slack 等）
6. Agent 继续

如果 standing rules 覆盖了这个调用（`targets` 声明了允许这个目标），则跳过审批直接执行。这是 §25 standing rules 的设计意图--让无人值守任务在不牺牲安全的前提下顺畅运行。

## 14.7　TaskRun 与历史记录

```python
@dataclass
class TaskRun:
    task_id: str
    status: str          # "ok" / "error" / "interrupted"
    error: Optional[str] = None
    trigger: str         # "schedule" / "catchup"
    started_at: str
    finished_at: Optional[str] = None
    session_id: str      # 创建的会话 ID
```

每次运行都被记录在 `TaskStore` 中，用户可以在 UI 中查看运行历史和 transcript。

## 本章小结

OpenWorker 的自动化系统用 "Scheduler 定时循环 + selfwake 自唤醒 + standing rules 精确授权" 的三层设计，实现了无人值守的安全自动化。最值得借鉴的是：run-once-catch-up（补跑一次而非多次）、skip-on-overlap（防止任务重叠）、spawn 而非 await（一个挂起的任务不阻塞其他任务）、以及 standing rules 的精确目标绑定（让无人值守的"always allow"不再危险）。

## 动手实验

1. 在 `automation/scheduler.py` 中找到 `_loop()` 方法，追踪一个 due 任务的执行路径。
2. 在 `selfwake.py` 中找到 `selfwake_tools()`，看 Agent 如何安排自己的唤醒。
3. 在 `automation/models.py` 中找到 `ScheduledTask.targets`，思考它如何被转化为 `PermissionEngine.task_rules`。
4. 创建一个每 5 分钟运行一次的自动化任务，切换到 unattended 模式，观察审批如何进入 Inbox。
