# 第 5 章　与模型对话：`ModelClient` 与 Responses API

上一章的 `run_turn` 把活儿交给了一句 `run_sampling_request`，而它背后真正与模型对话的，是 `core/src/client.rs` 里的 `ModelClient` 和 `ModelClientSession`。这一章我们拆开 Codex 与 OpenAI 模型之间的这条管道：请求长什么样、流式响应如何被解析成一个个事件、WebSocket 为什么要预热、以及当网络抖动时它怎么重试。

这是整个 Agent 里"与外部世界对话"最关键的一条边界——所有不确定性（网络、模型、限流）都在这里被工程化地驯服。

---

## 5.1 两级客户端：`ModelClient` 与 `ModelClientSession`

Codex 把模型客户端分成两级，对应"会话级"与"turn 级"两种生命周期。

`ModelClient` 是**会话级**的，内部只有 `state: Arc<ModelClientState>` 和一个可选的 `prompt_cache_key_override`。它通过 `new()` 构造、`new_session()` 派生出 turn 级会话。它代表"这个线程用哪个 provider、哪个模型、什么认证"这些相对稳定的东西。

`ModelClientSession` 是**turn 作用域**的——这一点上一章已经埋了伏笔，这里源码注释讲得极清楚。它"惰性地建立一条 Responses WebSocket 连接，并在一个 turn 内的多次请求间复用"。它缓存两样关键的 per-turn 状态：

- **上一次完整请求**（`last_request`），这样后续调用在"当前请求是上一次的增量扩展"时，可以只发增量的 WebSocket payload，而非整包重传。
- **`x-codex-turn-state` sticky-routing token**，存在一个 `turn_state: Arc<OnceLock<String>>` 里。

这个 `turn_state` 的契约值得专门讲。注释说：服务器在 turn 开始时通过 `x-codex-turn-state` **响应头**下发这个值，客户端一旦收到就存进 `OnceLock`，之后这个 turn 内的**所有**请求（重试、增量追加、续写）都要把它原样塞回 `x-codex-turn-state` **请求头**，以维持 sticky routing（黏性路由——让同一 turn 的请求落到同一后端）。**但绝不能跨 turn 复用**——注释警告，跨 turn 复用会把上一轮的路由 token 带进下一轮，违反客户端/服务端契约、引发路由 bug。这就是为什么第 4 章强调"每个 turn 必须新建一个 `ModelClientSession`"。

`WebsocketSession` 这个内部结构则保存连接本身（`connection`）、`last_request`、`last_response_rx`，以及一个 `connection_reused` 标志。它让"复用连接还是新建"成为一个可观测的状态。

---

## 5.2 请求是什么：`Prompt` 与 Responses API

发给模型的请求由 `core/src/client_common.rs` 的 `Prompt` 结构承载。它的字段精准地框定了"一次模型调用包含什么"：

- `input: Vec<ResponseItem>`——对话上下文条目（历史消息、工具结果等）。
- `tools: Vec<ToolSpec>`——本次可用的工具，注释强调**包括来自外部 MCP server 的工具**（第 14 章会看到 MCP 工具如何混进这里）。
- `parallel_tool_calls: bool`——是否允许并行工具调用。
- `base_instructions: BaseInstructions`——基础指令（System Prompt 的核心，第 12 章详解）。
- `personality`、`output_schema`、`output_schema_strict`——可选的人格与结构化输出约束。

注意 `Prompt::default()` 里 `output_schema_strict` 默认为 `true`：当你要求结构化输出时，Codex 默认让 Responses API **严格校验** schema。这是一个典型的"默认即关闭/默认即严格"姿态——宁可严，不可松。

请求最终打到哪个端点？`client.rs` 里定义了几个常量：主端点 `RESPONSES_ENDPOINT = "/responses"`，压缩端点 `RESPONSES_COMPACT_ENDPOINT = "/responses/compact"`，记忆摘要端点 `MEMORIES_SUMMARIZE_ENDPOINT = "/memories/trace_summarize"`。ChatGPT 登录态下的 base URL 是 `CHATGPT_CODEX_BASE_URL = "https://chatgpt.com/backend-api/codex"`（定义在 `model-provider-info`）。

这里有一个值得记住的工程决策：**Codex 已经彻底抛弃了 Chat Completions API，只支持 Responses API。** `model-provider-info/src/lib.rs` 里有一个 `CHAT_WIRE_API_REMOVED_ERROR` 常量，当用户配置 `wire_api = "chat"` 时直接报错，并给出可操作的修复建议："set `wire_api = \"responses\"` in your provider config"，还附上讨论链接。这正是"错误即文档"原则的范本——错误信息本身教你怎么改对。

---

## 5.3 流式响应：`ResponseEvent` 事件流

模型的回复不是一次性返回的，而是流式的。Codex 把这条流抽象成一串 `ResponseEvent`（定义在 `codex-api/src/common.rs`）。读懂这个枚举，就读懂了"模型在流里到底会说什么"：

| 事件变体 | 含义 |
|---|---|
| `Created` | 响应已创建 |
| `OutputItemAdded` / `OutputItemDone(ResponseItem)` | 一个输出条目开始 / 完成（消息、工具调用等） |
| `OutputTextDelta(String)` | 文本增量（逐字流出的可见回复） |
| `ToolCallInputDelta { item_id, call_id, delta }` | 工具调用参数的增量 |
| `ReasoningSummaryDelta` / `ReasoningContentDelta` | 推理摘要/内容的增量 |
| `ServerModel(String)` | 服务端实际使用的模型——注释说它**可能与请求的模型不同**，当后端安全路由生效时 |
| `ModelVerifications(...)` | 服务端建议追加账户验证 |
| `TurnModerationMetadata(...)` | 审核元数据 |
| `ServerReasoningIncluded(bool)` | 服务端已计入历史推理 token，客户端不应重复估算 |
| `Completed { response_id, token_usage, end_turn }` | 本次响应结束，带 token 用量与"模型是否主动结束 turn" |
| `RateLimits(RateLimitSnapshot)` | 限流快照 |

几个细节透露了 Codex 对真实世界复杂性的处理。`ServerModel` 的存在承认了"你请求 A 模型，服务端可能因安全路由给你跑 B 模型"——客户端必须诚实地反映这一点。`Completed` 里的 `end_turn: Option<bool>` 带注释说"有些 provider 不设这个值，所以我们依赖 fallback 逻辑"——这正是第 4 章 `run_turn` 里 `needs_follow_up` 判断要那么小心的原因：不能假设模型一定会明确告诉你"我说完了"。`ServerReasoningIncluded` 则解决了 token 计费的一个微妙问题：避免客户端把服务端已经算过的推理 token 又估一遍。

这条事件流通过 `ResponseStream`（`client_common.rs`）传递，`client.rs` 里的 `RESPONSE_STREAM_CHANNEL_CAPACITY = 1600` 设定了流通道的缓冲容量，`STREAM_DROPPED_REASON` 则是流在收到 provider 终止事件前被丢弃时的诊断信息。

---

## 5.4 WebSocket 预热与请求头

为了降低首字延迟（TTFT），Codex 在 WebSocket 上做了**预热**。`ModelClientSession` 提供 `preconnect_websocket` 和 `prewarm_websocket` 两个方法——在 turn 真正开始前就把连接建好。第 4 章 `RegularTask::run` 里消费的 "startup prewarm" 正是这套机制的产物：用户还没等模型回复，连接已经热着了。`WEBSOCKET_CONNECT_TIMEOUT` 和 `DEFAULT_WEBSOCKET_CONNECT_TIMEOUT_MS = 15_000`（15 秒）框定了连接超时。WebSocket 协议版本则由 beta 头 `RESPONSES_WEBSOCKETS_V2_BETA_HEADER_VALUE = "responses_websockets=2026-02-06"` 协商。

每次请求都带一组 `x-codex-*` 请求头，它们构成了客户端与服务端之间的"带外元数据通道"：

- `X_CODEX_INSTALLATION_ID_HEADER = "x-codex-installation-id"`——安装标识。
- `X_CODEX_TURN_STATE_HEADER = "x-codex-turn-state"`——上文讲的 sticky routing token。
- `X_CODEX_TURN_METADATA_HEADER = "x-codex-turn-metadata"`——turn 元数据（第 4 章 `turn_metadata_state` 生成的就是它）。
- `X_CODEX_PARENT_THREAD_ID_HEADER`、`X_CODEX_WINDOW_ID_HEADER`、`X_OPENAI_SUBAGENT_HEADER`——父线程、窗口、子 agent 标识。

这些头让服务端能感知"这是哪个安装、哪个 turn、哪个窗口、是不是子 agent 发来的"，从而做路由、计费、遥测。

---

## 5.5 重试与退避：把网络不确定性关进笼子

与外部服务对话，失败是常态而非例外。Codex 在 `model-provider-info` 里为重试设了清晰的默认值与硬上限：

- `DEFAULT_REQUEST_MAX_RETRIES = 4`——单次请求最多重试 4 次。
- `DEFAULT_STREAM_MAX_RETRIES = 5`——流式响应最多重试 5 次。
- `DEFAULT_STREAM_IDLE_TIMEOUT_MS = 300_000`——流空闲 5 分钟即超时。
- `MAX_REQUEST_MAX_RETRIES = MAX_STREAM_MAX_RETRIES = 100`——即便用户在配置里调高，也被硬封顶在 100。

`request_max_retries()` 和 `stream_max_retries()` 这两个 getter 用 `unwrap_or(DEFAULT_*)` 兜底——用户没配就用默认值。这种"默认值 + 用户覆盖 + 硬上限"的三层结构，是 Codex 处理所有可配置数值的惯用范式：给合理默认、允许调整、但不许调到危险区间。

具体的重试逻辑在 `core/src/responses_retry.rs` 的 `handle_retryable_response_stream_error`——它判断一个流错误是否可重试，并通过 `log_retry` 记录每次重试，便于事后从遥测里看清"为什么重试了、退避了多久"。`client.rs` 里还有专门的 `PendingUnauthorizedRetry` 与 `AuthRequestTelemetryContext`，处理认证失效（401）这种特殊的可恢复错误——这与第 11 章的令牌刷新机制相连。

---

## 本章小结

- 模型客户端分两级：`ModelClient`（会话级，持有 `Arc<ModelClientState>`，`new_session()` 派生）与 `ModelClientSession`（turn 级，惰性建 WebSocket 并在 turn 内复用）。
- `ModelClientSession` 缓存 `last_request`（支持增量 payload）与 `x-codex-turn-state` sticky token（存 `OnceLock`）；token **turn 内必传、跨 turn 严禁复用**，否则路由出错。
- 请求由 `Prompt` 承载：`input`/`tools`（含 MCP 工具）/`parallel_tool_calls`/`base_instructions`/`output_schema`。`output_schema_strict` 默认 `true`（默认即严格）。
- Codex **只支持 Responses API**：端点 `/responses`、`/responses/compact`；`wire_api="chat"` 直接报错（`CHAT_WIRE_API_REMOVED_ERROR`，错误即文档）。
- 流式响应抽象为 `ResponseEvent` 枚举：`OutputTextDelta`、`ToolCallInputDelta`、`Reasoning*Delta`、`Completed{end_turn}`、`ServerModel`（实际模型可能≠请求模型）、`RateLimits` 等。
- WebSocket 预热（`prewarm_websocket`，超时 15s，beta 头 `responses_websockets=2026-02-06`）降低 TTFT；一组 `x-codex-*` 请求头传递安装/turn/窗口/子agent 元数据。
- 重试默认值：请求 4 次、流 5 次、流空闲超时 5 分钟，用户可调但硬封顶 100；逻辑在 `responses_retry.rs`，401 走 `PendingUnauthorizedRetry`。

## 动手实验

1. **实验一：读懂 turn_state 契约** — 阅读 `client.rs` 中 `ModelClientSession.turn_state` 字段上的长注释，用自己的话复述：这个值从哪来、什么时候必须发回、为什么绝不能跨 turn 复用。
2. **实验二：列出所有流事件** — 打开 `codex-api/src/common.rs` 的 `pub enum ResponseEvent`，把每个变体分成"可见输出""推理""控制/元数据""计费/限流"四类。思考 `Completed.end_turn` 为何是 `Option<bool>`。
3. **实验三：验证"错误即文档"** — 在 `model-provider-info/src/lib.rs` 搜索 `CHAT_WIRE_API_REMOVED_ERROR` 和 `OLLAMA_CHAT_PROVIDER_REMOVED_ERROR`，读它们的错误文本，体会 Codex 如何用错误信息直接告诉用户修复步骤。
4. **实验四：找出重试三层结构** — 在 `model-provider-info/src/lib.rs` 找到 `DEFAULT_REQUEST_MAX_RETRIES`、`MAX_REQUEST_MAX_RETRIES` 与 `request_max_retries()` 方法，画出"默认值 → 用户覆盖 → 硬上限"的取值逻辑。

> **下一章预告**：模型在流里吐出了 `ToolCallInputDelta`——它想调一个工具。下一章我们进入工具系统：`ToolRouter` 如何把模型的工具调用解析、路由，`ToolRegistry` 如何让内置工具、MCP 工具、Skill 被一视同仁地调度，以及一次工具调用从请求到结果回灌的完整链路。
