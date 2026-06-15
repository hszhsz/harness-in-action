# 第 6 章　模型网关：provider 抽象、patched_* 与流式超时

中间件链装好了，模型节点要真的去调 LLM 了。但「调哪个模型、用哪个 SDK 类、出问题怎么补」并不是 `create_agent` 关心的事——LangChain 只要一个 `BaseChatModel` 实例。把「配置里写的一行 `use:`」翻译成一个可用的 chat model 实例，是 DeerFlow `models/` 分区的活儿。这一章我们钻进 `models/factory.py` 的 `create_chat_model`，看 DeerFlow 如何抽象不同 provider、为什么要给一堆模型打 `patched_*` 补丁、以及那个把流式超时默认设成 240 秒的决定背后藏着什么。

## provider 抽象：配置里的一行 `use:` 决定一切

DeerFlow 不在代码里 `if model == "deepseek": import ChatDeepSeek`。它把「用哪个类」彻底外包给配置。`create_chat_model` 的核心只有一行：

```python
model_class = resolve_class(model_config.use, BaseChatModel)
```

`model_config.use` 是 `config.yaml` 里每个模型条目的一个字符串字段，形如 `langchain_openai:ChatOpenAI` 或 `deerflow.models.patched_deepseek:PatchedChatDeepSeek`。`resolve_class`（`reflection/resolvers.py`）把这个 `模块路径:类名` 的字符串动态 import 成真正的类，并校验它是 `BaseChatModel` 的子类。这就是 DeerFlow 的 provider 抽象：**新增一个 provider 不需要改框架代码，只要在配置里写一行 `use:` 指向某个 `BaseChatModel` 子类**。`community/` 的可插拔搜索 provider 是这个思路在工具层的翻版，这里是它在模型层的现身。

`resolve_class` 底下是 `resolve_variable`：先 `module_path.rsplit(":", 1)` 切出模块和变量名，import 模块，`getattr` 取类，最后做类型校验。每一步失败都抛带提示的 `ImportError`——连「这个字符串根本不像一个变量路径」都给你举了个正确格式的例子：`Example: parent_package_name.sub_package_name.module_name:variable_name`。

## 缺依赖时，错误信息就是安装命令

provider 抽象有个天然的坑：配置里写了 `langchain_google_genai:...`，但用户没装那个包。普通的 `ModuleNotFoundError` 只会甩给用户一个光秃秃的「No module named 'langchain_google_genai'」。DeerFlow 不接受这种体验。`resolve_variable` 捕获 import 失败后调 `_build_missing_dependency_hint`：

```python
MODULE_TO_PACKAGE_HINTS = {
    "langchain_google_genai": "langchain-google-genai",
    "langchain_anthropic": "langchain-anthropic",
    "langchain_openai": "langchain-openai",
    "langchain_deepseek": "langchain-deepseek",
}
```

它把「import 用的下划线模块名」映射到「pip 用的横杠包名」，拼出一句可以直接复制粘贴的命令：`Missing dependency 'langchain_google_genai'. Install it with `uv add langchain-google-genai` (or `pip install langchain-google-genai`), then restart DeerFlow.` 注释里还点出一个隐蔽细节：它优先用「已知集成的包名提示」，**即使**报错是被某个传递依赖（比如 `google`）触发的——因为对用户来说，装 `langchain-google-genai` 才是对的解法，而不是去装那个底层的 `google`。这又是「错误即文档」原则的一次现身：报错不只说哪里错了，还告诉你下一步敲什么命令。

## thinking 模式：一套配置，两种开关姿势

`create_chat_model` 接受一个 `thinking_enabled` 布尔参数。但「开思考」这件事在不同 provider 上的姿势完全不同：原生 `langchain_anthropic` 把 `thinking` 当构造参数；OpenAI 兼容网关把它塞进 `extra_body.thinking.type`；vLLM 则靠 `extra_body.chat_template_kwargs` 里的 `enable_thinking`。DeerFlow 用配置字段 `when_thinking_enabled` / `when_thinking_disabled`（外加一个 `thinking` 快捷字段）把这些差异吸收掉。

开启路径相对直接：若 `thinking_enabled` 且模型声明了 `supports_thinking`，就把 `when_thinking_enabled` 的设置合并进构造参数；若模型没声明 `supports_thinking` 却被要求开思考，直接抛错——错误信息甚至告诉你去 `config.yaml` 里把 `supports_thinking` 设成 true。

真正啰嗦的是**关闭**路径，因为「关掉思考」要按 provider 分三种写法：

```python
elif has_thinking_settings and effective_wte.get("extra_body", {}).get("thinking", {}).get("type"):
    # OpenAI 兼容网关：thinking 嵌在 extra_body 下
    model_settings_from_config["extra_body"] = _deep_merge_dicts(..., {"thinking": {"type": "disabled"}})
    model_settings_from_config["reasoning_effort"] = "minimal"
elif has_thinking_settings and (disable_chat_template_kwargs := _vllm_disable_chat_template_kwargs(...)):
    # vLLM：用 chat template kwargs 切换
    ...
elif has_thinking_settings and effective_wte.get("thinking", {}).get("type"):
    # 原生 langchain_anthropic：thinking 是直接构造参数
    model_settings_from_config["thinking"] = {"type": "disabled"}
```

这段「丑陋」的分支正是 harness 工程的质地——它不优雅，但每一支都对应一个真实 provider 的真实写法。值得记下的是合并用的是 `_deep_merge_dicts`（递归合并、不改原字典），保证用户在 `extra_body` 里的其它设置不会被一句 `{"thinking": {"type": "disabled"}}` 整个覆盖掉。

## patched_* 补丁：上游 SDK 丢字段，DeerFlow 捡回来

`models/` 目录里有一排 `patched_*.py`：`patched_deepseek`、`patched_openai`、`patched_mimo`、`patched_minimax`、`patched_stepfun`。它们都干同一类事：**上游 LangChain 的某个 chat model 在序列化请求时丢掉了某个 provider 必需的字段，DeerFlow 子类化它、在 `_get_request_payload` 里把字段捡回来**。

最典型的两个：

**`PatchedChatDeepSeek`**（`patched_deepseek.py`）。开了 thinking 的 DeepSeek API 要求**所有** assistant 消息上都带 `reasoning_content`，但原版 `ChatDeepSeek` 把 `reasoning_content` 存进 `additional_kwargs` 后，下一轮请求时并不会把它序列化回去——多轮对话就会因为缺这个字段报错。补丁重写 `_get_request_payload`，调父类拿到基础 payload 后，用 `restore_assistant_payloads(...)` 把 `reasoning_content` 注回每条 assistant 消息。

**`PatchedChatOpenAI`**（`patched_openai.py`）。当你用 OpenAI 兼容网关跑开了 thinking 的 Gemini，API 要求每个 tool-call 对象上的 `thought_signature` 必须原样回传。但 `langchain_openai.ChatOpenAI` 只序列化 `id`/`type`/`function` 三个标准字段，把 `thought_signature` 默默丢了，导致 HTTP 400 `INVALID_ARGUMENT`：「function call ... is missing a `thought_signature`」。补丁同样重写 `_get_request_payload`，用 `_restore_tool_call_signatures` 把签名按 `id`（缺失则按位置）匹配回每个 tool-call——还贴心地兼容了网关可能用的 `thought_signature` 或 `thoughtSignature` 两种命名。

这两个补丁共用一段基础设施 `assistant_payload_replay.py`：`restore_assistant_payloads` 负责把「序列化后的 payload 消息」和「原始 AIMessage」对齐——长度一致就按位置 zip，不一致就退化成「按内容+tool_call_id 签名匹配，匹配不上再按序数兜底」。这段对齐逻辑被抽出来共享，各 provider 只需传入「要恢复哪个字段」的回调。一句话：**provider 各自决定捡哪个字段，怎么把字段对回正确的消息是公共的**。

## 240 秒：给「会思考的模型」留出沉默的时间

`_apply_stream_chunk_timeout_default` 里藏着一个很有人情味的常量：

```python
_DEFAULT_STREAM_CHUNK_TIMEOUT_SECONDS: float = 240.0
```

注释解释得很透：`langchain-openai` 在「多少秒收不到一个 chunk 就抛 `StreamChunkTimeoutError`」上的自带默认是 60 秒，但这对推理模型太狠了——DeepSeek-R1、Doubao-thinking、GPT-5 这类模型「第一个 chunk 合法地就能花 90~150 秒」（它在闷头思考，还没开始吐字)。DeerFlow 把默认提到 240 秒，让流式层「极少因为长时间的思考停顿而误判」；万一真的卡死，`LLMErrorHandlingMiddleware` 还会重试（budget=2）。用户也能在 `config.yaml` 里按模型覆盖。

这个 240 不是拍脑袋的魔法数字，而是一次对「会思考的模型」的体谅：**不要因为它沉默就以为它死了**。但补丁也很克制——`stream_chunk_timeout` 是 `langchain_openai:ChatOpenAI` 独有的 kwarg，别的 provider 构造器收到会抛 `TypeError`。所以这个函数对非 OpenAI 路径直接 `pop` 掉这个键，绝不把它转发给不认识它的构造器。「只给认识的人塞参数」是又一处边界自觉。

## stream_usage：让 token 统计不再为空

类似的「补默认」还有 `_enable_stream_usage_by_default`。LangChain 只在「没配自定义 base_url/client」时才自动给 OpenAI 模型开 `stream_usage`，但 DeerFlow 大量用的是 OpenAI 兼容网关——一旦设了 `base_url`，`stream_usage` 就静默关了，流式响应里就没有 token 用量，`TokenUsageMiddleware` 也就无从记录。所以 DeerFlow 在「path 是 `langchain_openai:ChatOpenAI`、用户没显式配 `stream_usage`、且配了 `base_url`」时主动把它打开。`create_chat_model` 末尾还有一层更通用的兜底：只要模型类的 `model_fields` 里有 `stream_usage` 字段且没人配过，就默认 True。**可观测性不能因为换了个网关就瞎掉**——这个细节呼应第 16 章的遥测主题。

## 两个特例：Codex 与 MindIE

provider 抽象再干净，也总有装不进通用流程的特例，DeerFlow 选择在 `create_chat_model` 里显式打补丁而非藏着掖着：

- **Codex**（`CodexChatModel`，走 ChatGPT Codex 的 Responses API）：该端点会拒绝 `max_tokens`，所以 pop 掉；它也不吃布尔式 thinking，而是用 `reasoning_effort`——于是把 `thinking_enabled` 映射成 `none`/`low`/`medium`/`high`/`xhigh`，未开思考时强制 `none`，开了但没指定则给 `medium`。这个 provider 还能从 `~/.codex/auth.json` 自动加载 Codex CLI 的 OAuth 凭据。
- **MindIE**（`MindIEChatModel`）：强制 `max_retries` 默认为 1，注释说是「防止级联超时」——一个不让重试雪崩的保守默认。

这两段特判都用 `issubclass` / `__name__` 判断模型类，把「这个 provider 有个怪脾气」明明白白写在工厂里，而不是让用户在运行时撞见一个莫名其妙的 400。

## tracing：装在图根，模型层闭嘴

最后一个参数 `attach_tracing` 看似不起眼，却是踩过坑才有的设计。`create_chat_model` 默认 `attach_tracing=True`，给模型实例挂上 Langfuse/LangSmith 回调。但文档字符串里有一句加粗级别的警告：已经在图根（`make_lead_agent`、图内的 `TitleMiddleware`）挂了 tracing 的调用方，**必须**传 `attach_tracing=False`，否则同一次 LLM 调用会发出两份 span（一份挂图根、一份挂模型），而且 `session_id`/`user_id` 元数据会因为模型变成嵌套观测而丢失。像 `MemoryUpdater`、临时工具这类「在图外独立调模型」的调用方才保留默认 True。这是「在哪一层挂遥测」的精确取舍——挂错层不会报错，只会悄悄污染你的 trace。

## 本章小结

- DeerFlow 的 provider 抽象是「配置驱动」：`create_chat_model` 用 `resolve_class(model_config.use, BaseChatModel)` 把 `config.yaml` 里一行 `模块:类名` 动态解析成 chat model 类，新增 provider 不改框架代码。
- `resolve_variable`/`resolve_class` 对每种失败都给带提示的 `ImportError`；`_build_missing_dependency_hint` 把缺失模块映射成可直接复制的 `uv add` / `pip install` 命令。
- thinking 模式按 provider 有三种开关姿势（Anthropic 直接参数、OpenAI 兼容网关 `extra_body.thinking`、vLLM `chat_template_kwargs`），DeerFlow 用 `when_thinking_enabled`/`when_thinking_disabled` 配置 + `_deep_merge_dicts` 递归合并统一吸收。
- 一排 `patched_*` 补丁子类化上游 chat model、重写 `_get_request_payload` 捡回被丢字段：`PatchedChatDeepSeek` 保 `reasoning_content`（多轮思考必需），`PatchedChatOpenAI` 保 Gemini 的 `thought_signature`（防 400）；对齐逻辑共享在 `assistant_payload_replay.py`。
- `_DEFAULT_STREAM_CHUNK_TIMEOUT_SECONDS = 240.0` 把流式 chunk 超时从上游的 60s 放宽到 240s，体谅推理模型 90~150s 的首 chunk 沉默；该 kwarg 仅对 `ChatOpenAI` 注入，其它 provider 直接 pop 防 `TypeError`。
- `_enable_stream_usage_by_default` 在 OpenAI 兼容网关上主动开 `stream_usage`，避免换网关后 token 统计变空、`TokenUsageMiddleware` 无米下锅。
- Codex（pop `max_tokens`、thinking→`reasoning_effort`、自动加载 OAuth 凭据）与 MindIE（`max_retries` 默认 1 防级联超时）作为特例在工厂里显式处理。
- `attach_tracing` 默认 True，但已在图根挂 tracing 的调用方必须传 False，否则同一次调用发重复 span 且丢元数据。

## 动手实验

1. **实验一：跟踪一行 `use:` 的解析** — 打开 `reflection/resolvers.py`，手推 `resolve_class("deerflow.models.patched_deepseek:PatchedChatDeepSeek", BaseChatModel)` 的执行：它在哪一步 import、哪一步 `getattr`、哪一步做 `issubclass` 校验？再故意传一个不含 `:` 的字符串，看它命中哪条报错分支。
2. **实验二：复现一次缺依赖提示** — 阅读 `_build_missing_dependency_hint` 与 `MODULE_TO_PACKAGE_HINTS`，回答：若配置写 `langchain_google_genai:ChatGoogleGenerativeAI` 而该包未装，用户会看到的完整报错是什么？为什么注释强调「即使报错由传递依赖 `google` 触发，也优先提示装 `langchain-google-genai`」？
3. **实验三：读懂一个 patched 补丁** — 通读 `patched_openai.py` 的 `_restore_tool_call_signatures` 和 `assistant_payload_replay.py` 的 `restore_assistant_payloads`。画出「payload 消息」与「原始 AIMessage」对齐的两条路径（长度相等 vs 不等），并说明 `thought_signature` 在什么情况下会按 `id` 匹配、什么情况下退化成按位置匹配。
4. **实验四：核对超时与用量的默认** — 在 `factory.py` 找到 `_apply_stream_chunk_timeout_default` 与 `_enable_stream_usage_by_default`，确认它们都只对 `model_use_path == "langchain_openai:ChatOpenAI"` 生效。思考：如果把 `stream_chunk_timeout=240` 误传给一个非 OpenAI 的 provider 构造器，会发生什么？DeerFlow 用哪一行代码防住了这件事？

> **下一章预告**：模型能调了，但它「能调什么工具」又是另一套工程。下一章我们走进 `tools/` 分区，看 DeerFlow 如何注册工具、如何用 deferred search（延迟搜索）把昂贵的工具调用推迟到真正需要时，以及 `community/` 下那 12 个可插拔搜索/抓取 provider 怎样被统一成一套工具契约。
