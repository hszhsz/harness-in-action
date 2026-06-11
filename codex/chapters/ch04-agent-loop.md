# 第 4 章　Agent 主循环：`CodexThread` 与一次 turn 的生命周期

上一章我们把控制权交到了 `core`。这一章进入 Agent 的心脏：当你的输入抵达，Codex 如何把它变成一段"模型思考 → 调工具 → 看结果 → 再思考"的循环，直到任务收束。我们会拆三层：**线程**（`CodexThread` / `ThreadManager`）是会话的容器，**任务**（`SessionTask` / `RegularTask`）是一次 turn 的承载体，**`run_turn`** 才是真正跳动的那颗心。

读完你会明白：Codex 的"主循环"不是一个简单的 `while(true)`，而是一套精心分层、处处考虑取消/压缩/钩子的状态机。

---

## 4.1 线程层：`CodexThread` 与 `ThreadManager`

Codex 把一次完整的对话称为一个 **thread**（线程，不要和 OS 线程混淆）。它的句柄是 `core/src/codex_thread.rs` 里的 `CodexThread`。从 `core/src/lib.rs` 的 `pub use codex_thread::CodexThread` 可以看出，这是 `core` 对外暴露的头号类型。

`CodexThread` 的方法名几乎自解释了它的职责：`submit()` 提交一个操作、`try_start_turn_if_idle()` 在空闲时尝试开启一轮、`inject_if_running()` 向正在跑的 turn 注入模型可见项、`shutdown_and_wait()` 优雅关闭。它内部持有一个 `ThreadConfigSnapshot`——一份当前线程的配置快照，字段包括 `model`、`model_provider_id`、`approval_policy: AskForApproval`、`permission_profile`、`environments`、`workspace_roots`、`reasoning_effort` 等。把配置"快照"进线程，意味着一轮对话期间的行为是确定的，不会因为外部配置变动而中途漂移。

`try_start_turn_if_idle` 还有一个值得玩味的设计：它返回一个 `TryStartTurnIfIdleRejectionReason` 枚举来解释**为什么拒绝**自动开启 turn。三个变体——`PendingTriggerTurn`（用户触发的工作已排队，优先级更高）、`PlanMode`（处于 Plan 模式，自动空闲工作不得开启新的模型 turn）、`Busy`（已有 turn 在跑）——把"什么时候不该自动干活"显式编码进了类型。这正是第 1 章说的"会动手的 Agent 要克制"的微观体现：宁可拒绝，也不要在不该动的时候乱动。

线程之上是 `thread_manager.rs` 里的 `ThreadManager`，负责创建与维护多个线程：`start_thread()` 启动、`fork_thread()` 分叉（返回 `ForkSnapshot`）、`shutdown_all_threads_bounded()` 有界关闭。它还提供 `build_models_manager` 和 `thread_store_from_config`——把模型管理与状态存储从配置里装配出来。`NewThread`、`StartThreadOptions`、`ThreadShutdownReport` 这些类型勾勒出了线程的完整生命周期。

---

## 4.2 任务层：`SessionTask` 与 `RegularTask`

线程是容器，真正"跑起来"的是**任务**。Codex 在 `core/src/tasks/` 下定义了一个 `SessionTask` trait（`tasks/mod.rs`），它是所有可在后台 Tokio 任务上执行的工作的统一抽象。`tasks/` 目录里能看到它的几种实现，每一种对应一类 turn：`regular.rs`（常规对话）、`compact.rs`（压缩）、`review.rs`（代码评审）、`user_shell.rs`（用户 shell 命令）、`lifecycle.rs`（生命周期）。

最核心的是 `tasks/regular.rs` 里的 `RegularTask`。它实现 `SessionTask`，`kind()` 返回 `TaskKind::Regular`，`span_name()` 返回 `"session_task.turn"`（这个名字会出现在 tracing 里，便于排障）。它的 `run()` 方法是一切的起点：

1. 先 inline 发出一个 `TurnStarted` 事件（`EventMsg::TurnStarted`），注释解释这样做是为了"首轮生命周期不必等待 startup prewarm 解析"——即让用户尽快看到"turn 开始了"的反馈。
2. 尝试消费 startup prewarm（预热的模型连接）：`consume_startup_prewarm_for_regular_turn` 返回 `Ready`/`Unavailable`/`Cancelled`，若被取消则直接 `return None`。
3. 进入一个 `loop`，反复调用核心函数 `run_turn(...)`，直到没有待处理输入为止。

这里有个清晰的分层：`RegularTask::run` 管"任务级"的循环（一个任务里可能跑多轮 `run_turn`，比如用户在模型说话时又插了一句），而 `run_turn` 管"单轮"与模型的完整交互。`SessionTaskContext` 把 session、turn 扩展数据等上下文打包传递，`CancellationToken` 则贯穿始终——任何一层都能响应取消，这是"会动手的 Agent 必须随时能被叫停"的工程保证。

---

## 4.3 心跳：`run_turn` 的一轮究竟做了什么

真正的"心脏"在 `core/src/session/turn.rs` 的 `run_turn` 函数里。它接收 session、`TurnContext`、扩展数据、输入、可选的预热 `ModelClientSession`、以及取消令牌。我们顺着它的执行流走一遍——这是理解整个 Agent 最关键的一段代码。

**第一步：开轮前的准备。** `run_turn` 先拿到一个 `ModelClientSession`（优先复用预热的那个，否则 `model_client.new_session()` 新建）。注释特别说明，`ModelClientSession` 是**turn 作用域**的，缓存了 WebSocket 连接和 sticky 路由状态，所以**一轮内的多次重试会复用同一个实例**——这是性能与一致性的权衡。

接着它调用 `run_pre_sampling_compact`：在真正向模型采样之前，先判断是否需要压缩历史。如果失败，发一个 turn 错误生命周期事件并 `return None`。然后 `record_context_updates_and_set_reference_context_item` 记录上下文更新，`build_skills_and_plugins` 把本轮要注入的 skill / plugin 项准备好。一系列 hook（`run_pending_session_start_hooks`、`run_hooks_and_record_inputs`）在此插入——任一 hook 返回需要中断，就提前结束。

**第二步：建立本轮的 diff 追踪器。** 注释里有一句很关键的话："从 `codex.rs` 的视角看，`TurnDiffTracker` 的生命周期是一个包含多个 turn 的 Task；但从用户视角看，它是一个单独的 turn。" 于是它用一个 `Arc<Mutex<TurnDiffTracker>>` 包裹整轮的文件改动追踪，第 7 章会看到 `apply_patch` 如何往这里记账。

**第三步：进入 `loop`——采样与回灌的循环。** 这才是"主循环"本体：

- **收集待处理输入**：`pending_input` 是用户在模型运行时通过 UI 又输入的内容。注释提醒"UI 可能支持这种插话，但模型未必"，所以是否在本轮 drain 取决于 `can_drain_pending_input` 标志。
- **构造采样输入**：`clone_history().for_prompt(...)` 把历史按模型的 `input_modalities` 投影成 `Vec<ResponseItem>`。
- **附加 turn 元数据头**：`turn_metadata_state.current_header_value_for_model_request(...)` 生成本次模型请求的元数据头（与第 5 章的请求头机制呼应）。
- **采样**：调用 `run_sampling_request(...)`，这一步真正与模型对话、执行其中的工具调用，返回 `SamplingRequestResult { needs_follow_up, last_agent_message }`。

**第四步：根据采样结果决定去留。** 这是循环里最体现工程克制的部分：

- 计算 `needs_follow_up = model_needs_follow_up || has_pending_input`——只要模型还想继续、或还有用户插话没处理，就要再转一圈。
- 记账 token 预算（`maybe_record_token_budget_remaining_context`）。
- 如果开了新的上下文窗口（`maybe_start_new_context_window`）且还要继续，`continue`。
- **如果 token 上限到了且还要继续**，触发 `run_auto_compact`（自动压缩）。注释里有一句颇为自信的话："只要压缩能把我们压到远低于上限，就不必担心死循环。" 压缩用 `CompactionReason::ContextLimit`、`CompactionPhase::MidTurn` 标记原因与阶段。
- **如果不需要继续**（`!needs_follow_up`），跑停止钩子 `run_turn_stop_hooks`。停止钩子可以"否决"停止：若 `stop_outcome.should_block`，且有 continuation 提示，就把提示作为对话项记下、置 `stop_hook_active = true` 并 `continue`——相当于"钩子拦住了收尾，要求模型再干点活"。这让外部扩展能在 Agent 想停的时候强制它继续，是一种强大的可编程收束控制。

整个 `run_turn` 没有"魔法"，它就是一个被取消令牌、压缩安全网、钩子、token 预算层层包裹的循环。每一个 `continue` 和 `return None` 背后，都是一个明确回答了"此刻该不该继续"的判断。

---

## 4.4 取消、压缩与钩子：贯穿循环的三条横切线

读完 `run_turn`，你会发现三个概念像红线一样贯穿始终：

**取消（Cancellation）。** `CancellationToken` 从 `RegularTask::run` 一路传到 `run_turn`，再 `child_token()` 传给 `run_sampling_request`。任何一层检测到取消都能干净退出。`tasks/mod.rs` 里还有 `abort_all_tasks`、`abort_turn_if_active`、`handle_task_abort` 等一整套中止机制。对一个会执行 shell 命令的 Agent 来说，"随时能叫停"不是锦上添花，而是安全底线。

**压缩（Compaction）。** turn 内有两处压缩点：开轮前的 `run_pre_sampling_compact` 和轮中 token 超限时的 `run_auto_compact`。压缩用 `CompactionReason` 和 `CompactionPhase` 把"为什么压、在哪个阶段压"显式记录，便于事后分析。这是 Codex 在有限上下文窗口里"走得更远"的核心机制。

**钩子（Hooks）。** session start hook、input record hook、stop hook 在循环的不同位置插入。尤其是 stop hook 那个"否决停止"的设计，让 Codex 的收束行为可以被扩展程序化地干预——这是它扩展体系（第 14 章）的一个伏笔。

---

## 本章小结

- Codex 把一次对话称为 **thread**，句柄是 `CodexThread`（`core/src/codex_thread.rs`），持有 `ThreadConfigSnapshot` 配置快照，保证一轮对话行为确定、不中途漂移。
- `try_start_turn_if_idle` 用 `TryStartTurnIfIdleRejectionReason`（`PendingTriggerTurn`/`PlanMode`/`Busy`）显式编码"何时不该自动开 turn"——会动手的 Agent 要克制。
- `ThreadManager` 管多线程：`start_thread`/`fork_thread`（返回 `ForkSnapshot`）/`shutdown_all_threads_bounded`。
- 任务层 `SessionTask` trait 有多种实现：`RegularTask`(常规)/`compact`/`review`/`user_shell`/`lifecycle`；`RegularTask::run` 跑任务级循环，`span_name="session_task.turn"`。
- **心脏是 `session/turn.rs::run_turn`**：开轮前压缩 → 记录上下文/skill/plugin → 建 `TurnDiffTracker` → 进入"采样-回灌"`loop` → 按 `needs_follow_up` 决定 continue/停止。
- `ModelClientSession` 是 turn 作用域、缓存 WebSocket 与 sticky 路由，一轮内重试复用同一实例。
- 三条横切线贯穿循环：取消（`CancellationToken` + `abort_*`）、压缩（`run_pre_sampling_compact` / `run_auto_compact`，带 `CompactionReason`/`CompactionPhase`）、钩子（尤其 stop hook 可"否决停止"强制继续）。

## 动手实验

1. **实验一：画出三层调用栈** — 从 `tasks/regular.rs::RegularTask::run` 出发，找到它对 `run_turn` 的调用；再进入 `session/turn.rs::run_turn`，找到它对 `run_sampling_request` 的调用。画出"任务循环 → turn 循环 → 采样"的三层嵌套。
2. **实验二：理解"拒绝自动开 turn"** — 阅读 `codex_thread.rs` 里 `TryStartTurnIfIdleRejectionReason` 的三个变体与各自的 doc 注释。设想三个具体场景，分别会命中哪个变体。
3. **实验三：定位两处压缩点** — 在 `session/turn.rs` 中搜索 `run_pre_sampling_compact` 和 `run_auto_compact`，对比它们触发的时机（开轮前 vs 轮中超限）与传入的 `CompactionReason`/`CompactionPhase`。
4. **实验四：读懂 stop hook 的"否决停止"** — 在 `run_turn` 中找到 `run_turn_stop_hooks` 的调用，跟踪 `stop_outcome.should_block` 为真时的分支。解释一个外部 hook 是如何让"想停下来的 Agent"被迫再转一圈的。

> **下一章预告**：`run_turn` 里那句 `run_sampling_request` 把活儿交给了模型客户端。下一章我们钻进 `ModelClient` 与 OpenAI Responses API：流式响应如何被解析、WebSocket 为什么要预热、重试与退避怎么做，以及一次模型请求究竟带了哪些请求头。
