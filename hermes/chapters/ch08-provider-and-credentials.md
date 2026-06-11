# 第 8 章　模型 Provider 抽象与凭证池

第 1 章我们就看到，Hermes 的卖点之一是"用任何你想用的模型"——Nous Portal、OpenRouter、NovitaAI、NVIDIA NIM、小米 MiMo、z.ai、Kimi、MiniMax、Hugging Face、OpenAI，或你自己的 endpoint，一句 `hermes model` 就能切换，不改一行代码。这种自由的背后，必须有一层把"几十个 provider 的鉴权差异、API 格式差异、能力差异"统统抹平的抽象。

更进一步，当你在生产环境跑一个长时间任务时，单把 API key 随时可能撞上限流（429）或被封（401）。一个可靠的 Agent 不能因为一把 key 出问题就整个停摆。Hermes 的答案是**凭证池**——管理多把 key，按错误类型智能轮换与冷却。这一章我们解构这两层：provider 抽象（`providers/` 与 `agent/*_adapter.py`）和凭证池（`agent/credential_pool.py`）。

## 8.1 `ProviderProfile`：用数据描述一个 provider

抹平 provider 差异的核心，是 `providers/base.py` 约第 39 行的 `@dataclass class ProviderProfile`。它的设计哲学是**用数据而非代码描述一个 provider**——新增一个 provider，理想情况下只需填一份 profile，而不是写一个类。

它的字段几乎把"接入一个 LLM 服务需要知道的一切"都覆盖了：`name`、`api_mode="chat_completions"`（API 风格，默认走 chat completions）、`aliases=()`（别名）、`display_name`、`description`、`signup_url`、`env_vars=()`（需要的环境变量）、`base_url`、`models_url`、`auth_type="api_key"`、`supports_health_check=True`、`supports_vision=False`、`supports_vision_tool_messages=True`、`fallback_models=()`、`hostname`、`default_headers`、`fixed_temperature`、`default_max_tokens`、`default_aux_model`。

其中 `auth_type` 的合法取值尤其值得注意：`api_key | oauth_device_code | oauth_external | copilot | aws_sdk`。这五种鉴权方式覆盖了从最简单的 API key、到 OAuth 设备码流程、到 GitHub Copilot 专有鉴权、再到 AWS SDK 签名——Hermes 把这些天差地别的鉴权机制收敛成一个枚举字段，上层代码只需看 `auth_type` 就知道该走哪条鉴权路径。

profile 还预留了一组可被覆盖的钩子方法：`get_hostname()`、`prepare_messages()`（消息预处理，因为不同 provider 对消息格式要求不同）、`build_extra_body()`、`build_api_kwargs_extras()`、`get_max_tokens(model)`、`fetch_models()`。**默认用数据描述、特殊情况用方法覆盖**——这是 dataclass + 钩子方法的经典组合，既让常见 provider 零代码接入，又给特殊 provider 留了逃生口。

## 8.2 provider 注册表与插件化接入

provider 的注册管理在 `providers/__init__.py`，结构上和第 4 章的工具注册表如出一辙：模块全局 `_REGISTRY: dict[str, ProviderProfile]`、`_ALIASES: dict[str, str]`、`_discovered` 标志。API 有 `register_provider(profile)`（约第 53 行）、`get_provider_profile(name)`（约第 65 行，会解析别名）、`list_providers()`（约第 76 行）、`_discover_providers()`（约第 140 行）。

最值得学习的是 provider 的**插件化**。provider 们自己在导入时注册（又一次"导入即注册"模式），而且分两个来源：bundled 插件在 `plugins/model-providers/`（`_BUNDLED_PLUGINS_DIR`），用户自定义插件在 `<hermes_home>/plugins/model-providers`。`_import_plugin_dir` 会导入每个插件的 `__init__.py`。这意味着**用户可以不改 Hermes 源码、只在自己的家目录里丢一个插件，就接入一个全新的 provider**。这种"核心提供机制、能力靠插件扩展"的模式，我们在第 1 章（extras）、第 4 章（工具）都见过，它是 Hermes 一以贯之的扩展哲学。

## 8.3 适配器：把统一接口翻译成各家 API

profile 描述"这个 provider 是什么"，适配器则负责"如何真正和它通信"。`agent/` 目录下有一批 `*_adapter.py`，处理各家 API 的具体差异。一个有意思的观察是：这些适配器**大多是函数式的，而非类式的**——一组处理特定转换的函数，而不是一个大类。

- `anthropic_adapter.py` 有一串判定与处理函数：`_is_claude_model`、`_get_anthropic_max_output`、`_resolve_anthropic_messages_max_tokens`、`_supports_adaptive_thinking`、`_is_oauth_token`、`_requires_bearer_auth`，还有 `_is_kimi_family_endpoint`、`_is_deepseek_anthropic_endpoint`、`_base_url_needs_context_1m_beta`。这些函数名透露了一个现实：很多 provider（Kimi、DeepSeek）其实**复用 Anthropic 的 API 格式**，所以 Anthropic 适配器要识别这些"Anthropic 兼容"的 endpoint 并做相应处理。
- `bedrock_adapter.py`（AWS Bedrock）有 `stream_converse_with_callbacks`（约第 725 行）、`call_converse`（约第 934 行）、`call_converse_stream`（约第 975 行）——Bedrock 用的是它自己的 Converse API。
- `codex_responses_adapter.py` 处理 OpenAI Responses API 风格，有 `_chat_messages_to_responses_input`（把 chat 消息转成 responses 输入）、`_responses_tools`、`_normalize_codex_response`、`_deterministic_call_id` 等。
- `gemini_cloudcode_adapter.py` 是少见的类式适配器，有 `GeminiCloudCodeClient`（约第 591 行）、`_GeminiChatCompletions`（约第 578 行）等——它把 Gemini 包装成一个"看起来像 chat completions"的客户端。

这一堆适配器加起来，就是 Hermes "用任何模型"承诺的工程代价：**每接入一种 API 格式，就要有一个适配器把它翻译成内部统一的消息/工具/流式接口**。这活儿又脏又累，但正是它让上层的主循环、工具系统、上下文引擎能完全不关心"底下到底是 Claude 还是 Gemini 还是 Bedrock"。

## 8.4 凭证池：把"封号"降级成"切换"

现在来到这一章的重头戏——`agent/credential_pool.py`。它解决的问题是：当你有多把 API key（无论是同一 provider 的多个账号，还是为了规避单 key 限流），如何智能地在它们之间调度，让单把 key 的故障不至于让 Agent 停摆。

核心是 `class CredentialPool`（约第 449 行），池里每个凭证是一个 `@dataclass class PooledCredential`（约第 130 行）。池支持四种轮换策略，定义为常量：`STRATEGY_FILL_FIRST = "fill_first"`（用满一把再换下一把）、`STRATEGY_ROUND_ROBIN = "round_robin"`（轮流）、`STRATEGY_RANDOM = "random"`（随机）、`STRATEGY_LEAST_USED = "least_used"`（优先用得最少的），合法集合是 `SUPPORTED_POOL_STRATEGIES`，由 `get_pool_strategy(provider)`（约第 430 行）按 provider 决定。不同策略适配不同需求：`fill_first` 适合有主备之分的场景，`round_robin` / `least_used` 适合均摊负载。

## 8.5 按错误码区分的冷却 TTL

凭证池最精妙的设计，是它**根据错误类型给凭证设不同长度的冷却时间**。三个 TTL 常量：

- `EXHAUSTED_TTL_401_SECONDS = 5 * 60`（401 鉴权失败，冷却 5 分钟）
- `EXHAUSTED_TTL_429_SECONDS = 60 * 60`（429 限流，冷却 1 小时）
- `EXHAUSTED_TTL_DEFAULT_SECONDS = 60 * 60`（默认 1 小时）

由 `_exhausted_ttl(error_code)` 按错误码选择。这个区分体现了对错误语义的精准理解：**401（鉴权问题）冷却时间短，因为它可能是临时的（token 短暂失效、时钟偏移），过几分钟再试或许就好了；而 429（限流）冷却时间长，因为限流窗口通常以小时计，太快重试只会再次撞墙、甚至加重限流**。如果对所有错误一视同仁地用同一个冷却时间，要么对 401 太保守（白白浪费一把还能用的 key），要么对 429 太激进（在限流窗口内反复无效重试）。

凭证的关键方法包括 `select()`（约第 1223 行，选一把可用凭证）、`current()`、`mark_exhausted_and_rotate()`（约第 1382 行，把当前凭证标记为耗尽并轮换到下一把）、属性 `has_capacity`（约第 463 行，池里还有没有可用凭证）。还有一类被标记为 "DEAD" 的凭证——它们会被排除在轮换之外，直到重新鉴权才复活。

这套机制和第 2 章的错误分类器（`ClassifiedError.should_rotate_credential`）严丝合缝地咬合在一起：主循环遇到错误 → 错误分类器判断该不该轮换凭证 → 凭证池执行 `mark_exhausted_and_rotate` 并按错误码设置冷却 → `select` 下一把可用 key 继续。**于是"一把 key 被封"这件原本会让 Agent 崩溃的事，被降级成了一次几乎无感的内部切换**。对一个要在 serverless 上无人值守跑几个小时的 Agent 来说，这是能不能用的分水岭。

## 8.6 解析 provider 的"何时恢复"提示

provider 在返回限流错误时，常会附带"什么时候能恢复"的信息，但格式五花八门。凭证池里有一组解析器专门对付这种混乱：`_parse_absolute_timestamp`（解析绝对时间戳）、`_extract_retry_delay_seconds`（提取重试延迟秒数），后者用正则匹配各种表达，比如 `quotaResetDelay`、`retry after Ns`、甚至自然语言的 `resets in 4hr 5min`。

把"resets in 4hr 5min"这种人话也纳入正则解析，是个很实在的细节——provider 的错误信息并不总是结构化的 JSON，有时就是给人看的英文句子。Hermes 选择主动去解析这些非结构化提示，从而能更精确地设置冷却时间（用 provider 告诉你的真实恢复时间，而不是粗暴地套用默认 1 小时）。这又是一处"边界不信任、但尽力理解外部输入"的体现。配套的凭证来源与持久化由 `agent/credential_sources.py` 和 `agent/credential_persistence.py` 承担，还有一个 `CUSTOM_POOL_PREFIX = "custom:"` 用于标识自定义池。

## 本章小结

- provider 差异由 `@dataclass ProviderProfile`（`providers/base.py` 约第 39 行）抹平，用数据描述 provider；`auth_type` 枚举 `api_key|oauth_device_code|oauth_external|copilot|aws_sdk` 覆盖五种鉴权，特殊情况用 `prepare_messages` 等钩子方法覆盖。
- provider 注册表结构同工具注册表，支持插件化接入：bundled 在 `plugins/model-providers/`、用户自定义在 `<hermes_home>/plugins/model-providers`，"导入即注册"，无需改源码即可加新 provider。
- `agent/*_adapter.py` 多为函数式适配器，把各家 API 翻译成内部统一接口；Anthropic 适配器还识别 Kimi/DeepSeek 等"Anthropic 兼容"endpoint，Bedrock 走 Converse API，Codex 走 Responses API。
- `CredentialPool` 管理多把 key，支持四种轮换策略（`fill_first` / `round_robin` / `random` / `least_used`），由 `get_pool_strategy(provider)` 选择。
- 冷却 TTL 按错误码区分：401 冷却 5 分钟（可能临时）、429 冷却 1 小时（限流窗口以小时计），由 `_exhausted_ttl` 选择——避免对 401 太保守、对 429 太激进。
- 凭证池与第 2 章错误分类器的 `should_rotate_credential` 咬合：错误 → 判断轮换 → `mark_exhausted_and_rotate` + 设冷却 → `select` 下一把，把"key 被封"降级成几乎无感的内部切换。
- `_extract_retry_delay_seconds` 用正则解析 provider 的恢复提示，连自然语言 `resets in 4hr 5min` 都能解析，以设置更精确的冷却——"尽力理解外部输入"。

## 动手实验

1. **实验一：用 profile 接入一个假想 provider** —— `Read` `providers/base.py` 的 `ProviderProfile`（约第 39 行）。假设要接入一个走 chat completions、用 API key 鉴权、支持 vision 的新服务，列出你需要填哪些字段、哪些可以用默认值。再用 Grep 在 `plugins/model-providers/` 找一个真实 provider 插件对照。

2. **实验二：理解适配器为何函数式** —— 用 Grep 在 `agent/anthropic_adapter.py` 列出所有以 `_is_` / `_get_` / `_resolve_` 开头的函数。论证：为什么这些 endpoint 判定逻辑写成独立函数、而不是塞进一个大类的方法？识别出哪些函数揭示了"其他 provider 复用 Anthropic 格式"这一事实。

3. **实验三：推演一次凭证轮换** —— `Read` `agent/credential_pool.py` 的三个 TTL 常量与 `_exhausted_ttl`、`mark_exhausted_and_rotate`（约第 1382 行）。模拟：一把 key 连续遇到 429，会冷却多久？另一把遇到 401 又冷却多久？如果池里所有 key 都进入冷却，`has_capacity` 会怎样、Agent 会发生什么？

4. **实验四：解析恢复提示** —— 找到 `_extract_retry_delay_seconds` 的正则，构造三种 provider 返回（一个 JSON 里的 `retry after 30s`、一个 `quotaResetDelay`、一个自然语言 `resets in 4hr 5min`），逐一验证能否被正确解析成秒数。思考：如果解析失败，会回退到哪个默认 TTL？

> **下一章预告**：模型能稳定调用了，但 Hermes 不只活在终端里——它能从 Telegram、Discord、Slack、WhatsApp、Signal、飞书等十余个平台接收消息。第 9 章将解构 `gateway/` 这个单进程多平台网关：`PlatformEntry` 如何统一异构平台、`PairingStore` 如何用 8 位码和哈希存储安全地配对设备、以及 `DeliveryRouter` 如何把超长回复按 `MAX_PLATFORM_OUTPUT = 4000` 切块投递。
