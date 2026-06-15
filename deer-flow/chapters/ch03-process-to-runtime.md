# 第 3 章　从进程到运行时：langgraph.json、RunManager 与运行生命周期

第 2 章我们看到，一次请求从 HTTP 网关进来，最终汇聚到唯一的 `lead_agent` 图。但「图」只是模型转圈的舞台，舞台之外还有一层调度：谁把请求登记成一个「运行（run）」？run 跑到一半用户关掉浏览器怎么办？同一个会话里用户又发了一条消息，是排队、打断、还是拒绝？进程崩溃重启后，那些「正在跑」的 run 会变成什么？这一章我们钻进 `runtime/` 分区，看 DeerFlow 如何把一次模型调用包装成一个有状态、可中断、能熬过重启的 run。

## run 是什么：一条可变记录

一切从 `runtime/runs/manager.py` 的 `RunRecord` 开始。它是一个 dataclass，记录了一个 run 的全部运行时状态。核心字段包括 `run_id`、`thread_id`、`status`、`on_disconnect`、`multitask_strategy`，以及一组 token 统计（`total_input_tokens`、`lead_agent_tokens`、`subagent_tokens`、`middleware_tokens` 等）。两个字段特别关键，它们是「可中断」能力的物理基础：

```python
task: asyncio.Task | None = field(default=None, repr=False)
abort_event: asyncio.Event = field(default_factory=asyncio.Event, repr=False)
```

`task` 是承载这次执行的 `asyncio.Task`，`abort_event` 是一个事件开关。后面我们会看到，所有的「打断」最终都归结为一个动作：`record.abort_event.set()`。

run 的状态空间定义在 `runtime/runs/schemas.py` 的 `RunStatus` 枚举里，只有六个取值：`pending`、`running`、`success`、`error`、`timeout`、`interrupted`。同一个文件里还有 `DisconnectMode`，只有两个值：`cancel`（SSE 消费者断连就取消）和 `continue_`（断连后继续跑）。状态空间小而封闭，这本身就是一种工程克制——run 的生命周期不会演化出一堆模糊的中间态。

## RunManager：内存注册表 + 可选持久化

`RunManager` 是 run 的中央注册表。它在内存里维护两个数据结构：`_runs`（`run_id → RunRecord`）和一个二级索引 `_runs_by_thread`（`thread_id → 有序的 run_id 集合）。后者用「dict 当有序集合」实现，目的是让「查某个 thread 下的所有 run」从 O(全部内存 run) 降到 O(该 thread 的 run)。所有对这两个结构的修改都由一把 `asyncio.Lock` 保护，并且注释反复强调一条不变量：索引和 `_runs` 必须「lockstep」（同步修改，两次写之间不能有 `await`），这样任何持锁者看到的两者都是一致的。

`_thread_records_locked` 的注释把这个不变量讲得很透彻：它用 `self._runs.get` 做了一层「防御性过滤」，能丢掉「还在索引里但已从 `_runs` 删除」的陈旧 id，但它**救不了**反方向的错误（在 `_runs` 里却不在索引里的 run 会被静默漏掉）。换句话说，DeerFlow 清楚地知道这层防御只守一个方向，并把这件事写进了注释——这是「不变量靠注释 + 代码双重固化」的典型笔法。

`RunManager` 可以只在内存里工作，也可以挂一个 `RunStore` 做持久化。当 store 存在时，run 的元数据会同步落库，从而「run 历史能熬过进程重启」。

## 持久化的两种态度：创建必须成功，更新尽力而为

DeerFlow 对「持久化失败」的处理透露了它的可靠性分级。看 `_persist_new_run_to_store` 的文档：

> Initial run creation is part of the run visibility boundary: callers should not observe a run in memory unless its backing store row exists. ... failures are propagated so the caller can treat creation as failed.

也就是说，**新建 run 的落库失败会向上抛**——因为「调用方不应该看到一个内存里有、但 store 里没有的 run」。这是一条可见性边界。但对比 `_persist_status`、`_persist_to_store` 这些后续更新，它们都标着 `best-effort`，失败只 `logger.warning` 不抛错。一个生产 harness 必须在「强一致」和「可用性」之间做取舍，DeerFlow 的取舍是：**run 的诞生要么对所有人可见、要么干脆失败；run 诞生之后的状态更新则尽力而为，不让一次写库抖动拖垮整个执行**。

而「写库抖动」本身也被专门处理了。`_call_store_with_retry` 实现了一个有界重试：默认 `PersistenceRetryPolicy(max_attempts=5, initial_delay=0.05, max_delay=1.0, backoff_factor=2.0)`，指数退避。但它只对「可重试」的错误退避——`_is_retryable_persistence_error` 会递归地穿过异常链（`orig`、`__cause__`、`__context__`），专门识别 SQLite 的 `database is locked` / `SQLITE_BUSY` / `SQLITE_LOCKED`。注释说得很准：「protects run status finalization from transient writer pressure without hiding permanent failures forever.」——只扛短暂的写压力，绝不无限期掩盖真正的失败。

## worker：让图在一个 Task 里转动

run 的实际执行在 `runtime/runs/worker.py` 的 `run_agent`。它做的事可以概括为：构造运行时上下文、把 run 状态置为 `running`、驱动图流式执行、然后根据结束方式落定终态。

驱动图用的是 LangGraph 的 `agent.astream(...)`。worker 的模块文档专门解释了一个取舍：网关层**不支持** `events` 这个 stream_mode，因为它需要 `graph.astream_events()`，而后者无法同时产出 `values` 模式的全量状态快照——「JS 版开源 LangGraph API server 靠内部 checkpoint 回调绕过，但 Python 公共 API 没暴露这个」。这是一个诚实记录框架能力边界的好例子：不是不想做，是上游 API 不支持，于是注释把原因写清楚。

流式循环的每一跳都检查中断信号：

```python
async for chunk in agent.astream(graph_input, config=runnable_config, stream_mode=single_mode):
    if record.abort_event.is_set():
        logger.info("Run %s abort requested — stopping", run_id)
        break
```

循环结束后，worker 根据 `abort_event` 和 `abort_action` 决定终态。`abort_action` 有三种语义：普通 `interrupt`（终态置 `interrupted`）、`rollback`（终态置 `error`，错误信息 "Rolled back by user"，并尝试回滚）。如果没有被中断、且无错误，才置为 `success`。这套逻辑把「正常完成 / 被打断 / 被回滚 / 出错」四种结局清清楚楚地映射到了那六个 RunStatus 上。

## 多任务策略：reject / interrupt / rollback

当同一个 thread 已经有 run 在跑，用户又发来新消息，怎么办？`RunManager.create_or_reject` 给出三种策略，定义在 `_supported_strategies = ("reject", "interrupt", "rollback")`，默认是 `reject`：

- `reject`：thread 有在飞行（inflight）的 run，就抛 `ConflictError`，新请求被拒。
- `interrupt`：把在飞行的 run 打断（`r.abort_action = "interrupt"; r.abort_event.set()`），再起新的。
- `rollback`：把在飞行的 run 打断并回滚，再起新的。

如果传入一个不在支持列表里的策略，会直接抛 `UnsupportedStrategyError`，错误信息里还列出所有支持的策略名。这是第 17 章「错误即文档」原则的又一处现身——报错不只说「不支持」，还告诉你「支持哪些」。

## 熬过重启：孤儿 run 的和解

进程崩溃重启后，store 里可能留着一批状态还是 `running` 的 run——但承载它们的 `asyncio.Task` 早已随进程消失。这些就是「孤儿（orphaned）run」。`RunManager.reconcile_orphaned_inflight_runs` 就是用来收拾它们的：进程重启后，把这些再也不会有人驱动的 run 从「假装在跑」的状态里解救出来，置成一个确定的终态。没有这一步，前端会永远看到一个转圈、却永远不会结束的 run。这正是「run 历史能熬过重启」承诺背后的清道夫。

## 本章小结

- 一个 run 由 `RunRecord` 表示，其 `task`（`asyncio.Task`）与 `abort_event`（`asyncio.Event`）是「可中断」能力的物理基础；所有打断最终都归结为 `abort_event.set()`。
- run 的状态空间封闭：`RunStatus` 只有 `pending/running/success/error/timeout/interrupted` 六态，`DisconnectMode` 只有 `cancel/continue_` 两态。
- `RunManager` 用 `_runs` + `_runs_by_thread` 二级索引加速按 thread 查询，并以「lockstep 不变量」保证两者一致，连防御性过滤只守一个方向都写进了注释。
- 持久化分级：新建 run 落库失败会**抛错**（可见性边界），后续状态更新是 **best-effort**；写库用有界指数退避重试，但只对 SQLite 的 `locked/BUSY` 这类瞬时错误退避，绝不掩盖永久失败。
- worker 用 `agent.astream` 驱动图，每跳检查 `abort_event`；终态按 `abort_action`（interrupt/rollback）和有无错误映射到六个 RunStatus 之一。网关不支持 `events` 流模式，原因（上游 Python API 限制）写在注释里。
- 多任务策略 `reject/interrupt/rollback` 默认 `reject`；不支持的策略抛错并列出可选值。
- `reconcile_orphaned_inflight_runs` 在重启后和解孤儿 run，避免前端看到永不结束的「假在跑」。

## 动手实验

1. **实验一：枚举状态空间** — 打开 `runtime/runs/schemas.py`，抄下 `RunStatus` 的六个值和 `DisconnectMode` 的两个值。再去 `worker.py` 里搜索 `set_status`，把每一处「在什么条件下置成哪个状态」列成一张状态转移表。
2. **实验二：追踪一次「打断」** — 在 `manager.py` 里搜索 `abort_event.set()`，找出所有调用点（`cancel`、`create_or_reject`、`shutdown`）。再到 `worker.py` 看 `abort_event.is_set()` 在哪被检查。把「谁触发打断 → 信号怎么传 → worker 怎么响应」连成一条完整链路。
3. **实验三：验证持久化的分级态度** — 对比 `_persist_new_run_to_store`（创建）与 `_persist_status`（更新）两个方法的实现和文档字符串，确认前者失败会向上 `raise`、后者只 `logger.warning`。思考：如果把创建也改成 best-effort，会出现什么 bug？
4. **实验四：读懂重试的「只扛瞬时」** — 阅读 `_is_retryable_persistence_error`，弄清它如何递归穿过 `orig` / `__cause__` / `__context__` 异常链，以及它认哪些 SQLite 错误为「可重试」。再看 `PersistenceRetryPolicy` 的四个默认值，手算一下最坏情况下总共会等待多久（0.05 + 0.1 + 0.2 + 0.4… 受 max_delay=1.0 封顶）。

> **下一章预告**：run 的舞台和调度已经清楚，下一章我们走上舞台中央，看 `make_lead_agent` 如何把模型、工具、系统提示和那条中间件链组装成一张 LangGraph 图——以及 DeerFlow 为什么宁可借 `create_agent` 的主循环，也要在它外面包十几层中间件钩子。
