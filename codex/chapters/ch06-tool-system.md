# 第 6 章　统一工具计划：`ToolRouter`、`ToolRegistry` 与工具调度

第 5 章模型在流里吐出了 `ToolCallInputDelta`——它想调一个工具。这一章我们解构 Codex 的工具系统：模型的工具调用如何被解析成结构、如何被路由到正确的处理器、内置工具与 MCP 工具与扩展工具如何被**同一套机制一视同仁**地调度。这套"统一工具计划"是 Codex 能力可扩展、却又安全可控的关键。

工具系统的代码集中在 `core/src/tools/`，外加一个独立的 `codex-tools` crate 提供公共 trait。我们顺着"契约 → 注册 → 路由 → 调度"四步拆开。

---

## 6.1 工具的契约：`ToolExecutor` 与 `CoreToolRuntime`

Codex 把"什么是一个工具"抽象成两层 trait。最底层的公共契约是 `ToolExecutor`，定义在独立的 `codex-tools` crate 里，`tools/registry.rs` 通过 `pub use codex_tools::ToolExecutor` 重新导出。把它单拎出来成一个 crate，意味着工具执行的契约是稳定的、可被多处复用的——这与第 2 章看到的"核心精简、契约独立"理念一致。

在 `ToolExecutor` 之上，`core` 定义了 `CoreToolRuntime` trait（`tools/registry.rs`）。它的 doc 注释写得很清楚：这是"本地执行工具的类型化运行时契约"，实现者在 `ToolExecutor` 的共享行为之外，再提供 core 私有的元数据——用于 **hook、遥测、工具搜索、参数 diff**。

`CoreToolRuntime` 的几个默认方法揭示了一个工具能参与哪些横切关注点：

- `matches_kind(payload)`：判断这个 runtime 能否处理某种 `ToolPayload`，默认匹配 `Function` 和 `ToolSearch` 两种。
- `waits_for_runtime_cancellation()`：默认 `false`——但某些工具（如正在写文件的）需要在取消时先完成清理再返回 aborted，可以覆盖成 `true`。这呼应了第 4 章"随时能叫停，但要叫停得干净"。
- `pre_tool_use_payload` / `post_tool_use_payload`：生成"工具调用前/后"的 hook 负载，让外部 hook 能在工具执行前后介入（甚至改写输入）。
- `with_updated_hook_input(...)`：允许 hook **改写**工具的输入参数后再执行——这是一个强大的可编程拦截点，注释强调实现者要"反转"它在 `pre_tool_use_payload` 里暴露的稳定契约。
- `create_diff_consumer()`：返回一个可选的 `ToolArgumentDiffConsumer`，用于消费**流式的工具参数增量**（还记得第 5 章的 `ToolCallInputDelta` 吗？就是喂给它的），并据此发出协议事件。这让 UI 能在工具参数还在流式生成时就实时展示，比如 `apply_patch` 的 diff 可以边生成边渲染。

这套 trait 设计的精妙之处在于：**工具不只是"一个会执行的函数"，而是一个能参与 hook、遥测、取消、流式 diff 的完整生命周期参与者。** 每个能力都有默认实现，工具只需覆盖自己关心的部分。

---

## 6.2 注册：`ToolRegistry`

所有工具被收进 `tools/registry.rs` 的 `ToolRegistry`。它通过 `from_tools(tools: impl IntoIterator<Item = Arc<dyn CoreToolRuntime>>)` 构造——接收一组 `Arc<dyn CoreToolRuntime>`，即一组实现了核心运行时契约的工具。注意这里用的是 trait object（`dyn CoreToolRuntime`）：内置工具、MCP 工具、扩展工具，只要实现了 `CoreToolRuntime`，在 registry 眼里就是同一种东西。**这就是"一视同仁"的技术根基——多态。**

`ToolRegistry` 提供按工具名查询各种属性的方法：`supports_parallel_tool_calls(name)`、`waits_for_runtime_cancellation(name)`、`tool_exposure(name)`、`create_diff_consumer(name)`。它还有一个有趣的 `ExposureOverride` 类型，实现了 `ToolExecutor` 和 `CoreToolRuntime`，用于覆盖某个工具的暴露策略（比如临时禁用、或改变它对模型的可见性）——这是工具系统能动态调整"模型此刻能看到哪些工具"的开关。

`tools/handlers/` 目录则是工具的实现仓库，每个工具一个（或一组）文件，且大多配有 `*_spec.rs`（工具的 schema 定义）和 `*_tests.rs`。扫一眼就能看出 Codex 内置了多少能力：`apply_patch.rs`（改代码，第 7 章）、`shell/`（shell 命令）、`unified_exec/`（统一执行）、`mcp.rs`（MCP 工具桥接）、`plan.rs`（计划）、`multi_agents.rs`（多 agent）、`request_permissions.rs`（请求权限）、`request_user_input.rs`（向用户提问）、`view_image.rs`（看图）、`web_search`、`tool_search.rs`（工具搜索）等。每个 handler 文件配一个 spec 文件的模式，正是第 2 章"模块限 500 行、职责单一"纪律的体现。

---

## 6.3 路由：`ToolRouter` 与 `build_tool_call`

`ToolRegistry` 是静态的工具仓库，`ToolRouter`（`tools/router.rs`）则是 turn 期的动态调度器。它持有 `registry: ToolRegistry` 和 `model_visible_specs: Vec<ToolSpec>`——后者是"本轮告诉模型有哪些工具可用"的清单。

`ToolRouter` 由 `from_turn_context(turn_context, params)` 构建，参数 `ToolRouterParams` 把本轮要纳入的各类工具汇聚到一起：

- `mcp_tools` / `deferred_mcp_tools`：来自 MCP server 的工具。
- `discoverable_tools`：可被发现的工具。
- `extension_tool_executors: Vec<Arc<dyn ToolExecutor<ExtensionToolCall>>>`：扩展工具。
- `dynamic_tools: &[DynamicToolSpec]`：动态工具。

这个参数结构本身就是一张"工具来源全景图"——内置、MCP、扩展、动态，全部在构建路由时被融合进同一个 registry。`model_visible_specs()` 则把融合后的工具列表暴露给模型（最终进入第 5 章 `Prompt.tools`）。

模型回复里的工具调用，先经 `build_tool_call(item: ResponseItem)` 解析。它接收一个 `ResponseItem`（来自模型流），匹配其中的 `FunctionCall` 等变体，产出一个 `ToolCall` 结构——包含 `tool_name: ToolName`、`call_id: String`、`payload: ToolPayload`。这一步把"模型说的话"翻译成"系统能执行的结构化调用"。注意它返回 `Result<Option<ToolCall>>`：不是所有 `ResponseItem` 都是工具调用（`None`），且解析可能失败（`FunctionCallError`）——边界处的输入永远不被假定为合法，这是第 1 章"边界不信任外部输入"的又一处体现。

`ToolRouter` 还提供两个查询方法：`tool_supports_parallel(call)` 判断该工具是否支持并行调用，`tool_waits_for_runtime_cancellation(call)` 判断取消时是否要等它清理完。两者都 `unwrap_or(false)`——**默认不并行、默认不等待**，这是保守的安全默认。

---

## 6.4 调度：`dispatch_tool_call_with_terminal_outcome`

解析出 `ToolCall` 后，真正执行它的是 `dispatch_tool_call_with_terminal_outcome`（以及面向 code-mode 的 `dispatch_tool_call_with_code_mode_result`）。调度流程在 registry 侧的 `dispatch_any_with_terminal_outcome` 落地，它负责：

1. 根据 `tool_name` 在 registry 里找到对应的 `CoreToolRuntime`。
2. 触发 `pre_tool_use` hook（可能改写输入）。
3. 走审批/沙箱判定（第 8、9 章详解）——这一步对**所有**工具一视同仁，无论它是内置 shell 还是外部 MCP 工具。
4. 调用工具的 `ToolExecutor` 执行。
5. 触发 `post_tool_use` hook，记录遥测。
6. 把结果包装成 `AnyToolResult`，最终转成 `FunctionCallOutput` 形态的 `ResponseItem`，回灌进对话历史——这就接回了第 4 章 `run_turn` 循环里"工具结果回灌"的那一环。

整个调度的"终态"（terminal outcome）概念很重要：一次工具调用无论成功、失败、被取消、被拒绝，都会产出一个确定的终态结果回灌给模型。模型永远会收到"你调的那个工具发生了什么"的反馈——哪怕是"它被审批拒绝了"。这种"必有回灌"的契约，是 Agent 循环不会卡死的保证：模型不会陷入"我调了工具但没等到结果"的悬空状态。

并行调度由 `tools/parallel.rs` 与 `orchestrator.rs` 协调：支持并行的工具（`tool_supports_parallel` 为真）可以同时执行，不支持的则串行。这让"读三个文件"这类无副作用操作能并行加速，而"改代码""跑命令"这类有副作用操作保持串行安全。

---

## 6.5 一视同仁：为什么内置工具和 MCP 工具走同一条路

回过头看，Codex 工具系统最重要的设计决策是：**内置工具、MCP 工具、扩展工具、动态工具，全部实现统一契约（`CoreToolRuntime`/`ToolExecutor`），被同一个 `ToolRegistry` 收纳、同一个 `ToolRouter` 路由、同一套审批与沙箱约束覆盖。**

这带来三个好处。其一，**安全无死角**：你无法通过"包装成 MCP 工具"来绕过审批——因为 MCP 工具和内置 shell 走的是同一条调度链（第 14 章会验证这一点）。其二，**可观测一致**：所有工具调用都经过同一套 hook 和遥测，`codex.tool.call` 这个指标（第 15 章）覆盖每一个工具。其三，**扩展零特权**：新增一个工具，无论来自哪里，都不会获得比内置工具更高的权限——能力是平权的。

这正是第 1 章所说"让会动手的 Agent 值得信任"在工具层的落地：信任不是靠"相信工具本身安全"，而是靠"所有工具都被同一道闸门约束"。

---

## 本章小结

- 工具契约分两层：公共的 `ToolExecutor`（独立 `codex-tools` crate）+ core 私有的 `CoreToolRuntime`（`tools/registry.rs`），后者附加 hook/遥测/工具搜索/参数 diff 元数据。
- `CoreToolRuntime` 让工具成为完整生命周期参与者：`pre/post_tool_use_payload`（hook 介入）、`with_updated_hook_input`（改写输入）、`create_diff_consumer`（消费流式参数增量）、`waits_for_runtime_cancellation`（取消时清理）。
- `ToolRegistry::from_tools` 接收 `Arc<dyn CoreToolRuntime>` 集合——用 **trait object 多态**让各来源工具"一视同仁"；`ExposureOverride` 可动态调整工具可见性。
- `handlers/` 是工具实现仓库，每工具配 `*_spec.rs`（schema）+ `*_tests.rs`，体现"模块单一职责"纪律。
- `ToolRouter::from_turn_context` 用 `ToolRouterParams` 融合内置/MCP/扩展/动态工具；`build_tool_call` 把模型的 `ResponseItem` 解析成 `ToolCall`（返回 `Result<Option<_>>`，边界不信任输入）。
- 默认保守：`tool_supports_parallel`、`tool_waits_for_runtime_cancellation` 均 `unwrap_or(false)`。
- 调度走 `dispatch_*_with_terminal_outcome`：查找 runtime → pre-hook → 审批/沙箱 → 执行 → post-hook/遥测 → 包成终态结果回灌。**必有回灌**保证循环不悬空。
- 核心思想：所有工具走同一调度链 + 同一审批沙箱 → 安全无死角、可观测一致、扩展零特权。

## 动手实验

1. **实验一：清点内置工具** — 列出 `core/src/tools/handlers/` 下所有不带 `_spec`/`_tests` 后缀的 `.rs` 文件，对照每个文件名猜测它提供什么能力，再打开其中两三个验证你的猜测。
2. **实验二：读懂"一视同仁"的多态根基** — 阅读 `tools/registry.rs` 的 `ToolRegistry::from_tools` 签名，解释为什么 `Arc<dyn CoreToolRuntime>` 是让内置工具与 MCP 工具被同等对待的关键。
3. **实验三：跟踪一次工具调用的解析** — 阅读 `tools/router.rs::build_tool_call`，找出它从 `ResponseItem` 的哪个变体中提取 `tool_name`/`call_id`/`payload`，并解释它为何返回 `Result<Option<ToolCall>>` 而非 `ToolCall`。
4. **实验四：验证流式参数 diff** — 在 `tools/registry.rs` 找到 `ToolArgumentDiffConsumer` trait 与 `create_diff_consumer` 方法，结合第 5 章的 `ToolCallInputDelta`，说明工具参数是如何"边流式生成、边渲染给用户"的。

> **下一章预告**：在所有工具里，`apply_patch` 是编码 Agent 的灵魂——它让模型能真正改你的代码。下一章我们专门拆这个工具：它的补丁格式长什么样、如何被解析和应用、改动如何被记进 `TurnDiffTracker`，以及 Codex 用哪些手段保证"改代码"这件高危操作的安全。
