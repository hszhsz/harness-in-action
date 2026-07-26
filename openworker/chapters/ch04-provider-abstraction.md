# 第 4 章　模型 Provider 抽象：让十四家供应商可插拔

## 4.1　设计目标

OpenWorker 的 README 列出了 14+ 家模型供应商：OpenAI、Anthropic、Google Gemini、Z AI (GLM)、DeepSeek、Kimi (Moonshot)、MiniMax、Qwen (Alibaba)、xAI (Grok)、Mistral、Meta (Muse Spark)、Together AI、Fireworks AI，以及通过 Ollama 运行的本地模型。

让这么多供应商可插拔的核心抽象是 `providers/base.py` 中的 `ProviderClient` ABC。

## 4.2　类型契约

### ProviderClient ABC

```python
class ProviderClient(ABC):
    @abstractmethod
    def complete(self, *, model, messages, tools=None, **settings) -> AssistantTurn: ...

    @abstractmethod
    def capabilities(self, model: str) -> ModelCapabilities: ...

    def stream(self, *, model, messages, tools=None, **settings):
        yield StreamChunk(turn=self.complete(...))  # 默认：无流式
```

**关键决策：阻塞式 API**。`complete()` 和 `stream()` 都是同步阻塞调用，TurnEngine 用 `asyncio.to_thread` 包装它们。这个决策的代价是 provider SDK 必须支持同步调用（OpenAI/Anthropic/Gemini SDK 都支持），收益是 TurnEngine 的事件循环不被 provider 的实现细节污染。

### AssistantTurn

```python
@dataclass
class AssistantTurn:
    text: Optional[str] = None
    tool_calls: list[ToolCall] = field(default_factory=list)
    finish_reason: Optional[str] = None
    raw: Any = field(default=None, repr=False, compare=False)
    reasoning: Optional[str] = None  # 思考文本（display-only）
    extras: dict[str, Any] = field(default_factory=dict)  # provider 私有 sidecar
```

`extras` 字段解决了一个跨 provider 的工程难题：某些 provider 需要在 assistant 消息上持久化私有数据（如 Gemini 的 thought signatures），但其他 provider 不应该看到这些数据。契约是：拥有者 provider 在 `complete()`/`stream()` 时重新附加自己的 key，其他 provider 在 `_outbound_messages()` 中剥离所有下划线前缀的 key。

### ModelCapabilities

```python
@dataclass(frozen=True)
class ModelCapabilities:
    tools: bool = True
    vision: bool = False
    pdf: bool = False
    parallel_tool_calls: bool = True
    streaming: bool = True
```

这个 frozen dataclass 驱动了 OpenWorker 的**优雅降级**机制：
- 没有 `vision` 的模型看到的是 `[image attachment - not viewable by this model]` 占位符
- 没有 `pdf` 的模型看到的是本地提取的文本/页面图片（`pdf_support.adapt_content()`）
- 这些适配在 `_outbound_messages()` 中**每次调用都重新决定**，所以 mid-session 切换模型总是做对

## 4.3　三种 Provider 实现路径

### 路径 1：原生 SDK（OpenAI / Anthropic / Gemini）

- `OpenAIProvider`：用 OpenAI SDK 的 `chat.completions`，支持自定义 endpoint（Azure / OpenRouter / vLLM）
- `AnthropicProvider`：用 Anthropic SDK 的 Messages API，带 extended thinking 支持（`DEFAULT_THINKING_BUDGET`）
- `GeminiProvider`：用 Google GenAI SDK，原生支持 thought summaries

### 路径 2：OpenAI 兼容 API（Z AI / DeepSeek / Kimi / MiniMax / Qwen / xAI / Mistral / Together / Fireworks）

`_openai_compat(vendor, default_base_url, env_key)` 是一个工厂函数，为每家 OpenAI 兼容的厂商生成一个 builder。关键设计：

```python
def build(profile, secrets):
    base_url = profile.get("base_url") or default_base_url
    api_key = profile.get("api_key") or os.environ.get(env_key, "")
    if not api_key:
        raise RuntimeError(f"No {vendor} API key configured")
    return OpenAIProvider(api_key=api_key, base_url=base_url)
```

**关键安全决策**：每家厂商用自己的 key profile，**不回退到 OpenAI 的 env/SecretStore**。这防止了一个配置好的 OpenAI key 被静默发送到另一个厂商的 endpoint。

### 路径 3：本地模型（Ollama）

```python
def _build_ollama(profile, secrets):
    base_url = _normalize_ollama_url(profile.get("base_url"))
    return OpenAIProvider(api_key="ollama", base_url=base_url)
```

Ollama 的 OpenAI 兼容 endpoint 忽略 key，但 SDK 需要非空字符串。`_normalize_ollama_url()` 确保 URL 以 `/v1` 结尾。

## 4.4　ProviderDescriptor + Registry：声明式配置

`providers/registry.py` 用 `ProviderDescriptor` 把每家厂商的 UI 配置字段、builder 函数、推荐模型和环境变量名打包成一个声明式描述符：

```python
@dataclass(frozen=True)
class ProviderDescriptor:
    name: str
    title: str
    needs_key: bool
    fields: list[ProviderField]
    build: Callable[[dict, Any], ProviderClient]
    recommended_model: Optional[str] = None
    env_key: Optional[str] = None
    blurb: str = ""
```

`DESCRIPTORS` 列表是所有 provider 的注册表，GUI 用它动态渲染设置页面（每家厂商的 fields 不同）。`build_provider_client(name, profile, secrets)` 是工厂入口。

## 4.5　ProviderRouter：按 `provider:` 前缀路由

```python
class ProviderRouter:
    """按模型字符串的 `provider:` 前缀选择 descriptor，缓存 client。"""
```

当用户选择 `anthropic:claude-fable-5` 时，Router 提取 `anthropic` 前缀，找到 `AnthropicProvider` 的 descriptor，从 SecretStore 取出对应的 profile，调用 `build()` 构建（并缓存）一个 `AnthropicProvider` 实例。如果模型字符串没有前缀，默认用 `openai`。

## 4.6　模型切换：mid-session 的复杂性

`TurnEngine.switch_model(model)` 处理运行中切换模型。核心逻辑：

1. 写入 `self.model = model`
2. 检查历史中是否有图片附件
3. 如果新模型不支持 vision，追加一条 notice：`"Model switched to X - earlier images can't be read by this model"`
4. 这条 notice 被 `_outbound_messages()` 在每次 provider 调用时重新评估--所以图片附件不会被丢失，只是变成占位符

`_outbound_messages()` 中的 PDF 适配也是 per-turn 的：`pdf_support.adapt_content(content, caps)` 根据**当前**模型的 capabilities 决定是发送原生 PDF 还是文本提取。

## 4.7　凭证验证

`verify_provider_key(name, *, api_key, base_url, timeout)` 用一次只读 HTTP 调用验证凭证：

- Anthropic：`GET https://api.anthropic.com/v1/models`（带 `x-api-key` header）
- Gemini：`GET https://generativelanguage.googleapis.com/v1beta/models?key=...`
- Ollama：`GET <base_url>/models`
- 其他：`GET <base_url>/models`（Bearer token）

永不抛异常，返回 `{ok: bool, error?: str}`。这和连接器的 token 验证模式一致。

## 本章小结

OpenWorker 的 Provider 抽象用"阻塞 ABC + 线程桥接 + per-turn 降级"的简洁设计，处理了 14+ 家供应商的能力差异。最值得借鉴的是：`ModelCapabilities` frozen dataclass 驱动的优雅降级（vision/pdf/streaming 都在每次调用时重新决定），以及 OpenAI 兼容厂商的 key 隔离设计（每家厂商用自己的 profile，不回退到 OpenAI）。

## 动手实验

1. 找到 `providers/registry.py` 中的 `DESCRIPTORS` 列表，数一下有多少个 `ProviderDescriptor`。
2. 打开 `providers/openai_provider.py`，找到 `stream()` 方法，看它如何把 SDK 的流式 chunk 转成 `StreamChunk`。
3. 在 `engine.py` 中找到 `_outbound_messages()` 方法，追踪一个 image_url 在 vision 模型和非 vision 模型下的不同处理路径。
4. 思考：为什么 `_openai_compat` 的 builder 不回退到 OpenAI 的 env key？
