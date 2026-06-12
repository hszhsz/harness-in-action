# 第 5 章　与模型对话：Provider 抽象、流式、重试与 Fallback

> 主循环在 `deps.callModel` 处把控制权交了出去。交给谁？交给一层把「跟大模型说话」这件看似简单的事，包装得极其复杂的网关。
> 这一章我们解构 `src/services/api/`——它要同时伺候四五家不同的模型 provider、要把响应流式吐回来、要在限流和故障时不动声色地重试甚至换模型，还要保证主循环对这一切几乎无感。

## 5.1 一个网关，多家 provider

`deps.callModel` 指向的是 `src/services/api/claude.ts` 的 `queryModelWithStreaming`（第 775 行，一个 `async function*` 生成器）。但「claude.ts」这个文件名有点误导——它并不只伺候 Anthropic。决定到底打给谁的，是 `src/utils/model/providers.ts` 里的 `getAPIProvider()`：

```ts
export type APIProvider =
  | 'firstParty' | 'bedrock' | 'vertex' | 'foundry'
  | 'openai' | 'gemini' | 'grok'
```

七种 provider。`getAPIProvider()` 的判定顺序本身就是一套优先级协议：先看 settings 里的 `modelType`（`openai`/`gemini`/`grok` 直接命中），再依次查环境变量 `CLAUDE_CODE_USE_BEDROCK` → `CLAUDE_CODE_USE_VERTEX` → `CLAUDE_CODE_USE_FOUNDRY` → `CLAUDE_CODE_USE_OPENAI` → `CLAUDE_CODE_USE_GEMINI` → `CLAUDE_CODE_USE_GROK`，全都没命中才回落到 `firstParty`（Anthropic 官方 API）。

```ts
if (isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK)) return 'bedrock'
if (isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX)) return 'vertex'
...
return 'firstParty'
```

这个「settings 优先于环境变量、环境变量有固定先后、最后兜底 firstParty」的排序，就是网关的第一层抽象——**无论底下接的是 AWS Bedrock、GCP Vertex、Azure Foundry，还是 OpenAI/Gemini/Grok，主循环只管调 `callModel`，provider 的差异被收进网关内部**。

在 `claude.ts` 里，这个差异通过一连串 `if (getAPIProvider() === 'xxx')` 分派落地：OpenAI 兼容 provider 委托给 `src/services/api/openai/` 的适配层，Gemini 走 `gemini/`，Grok 走 `grok/`，Bedrock/Vertex/Foundry 则在请求构造时走各自的 header 与签名分支。每家 provider 的消息格式、缓存控制、beta header 都不一样，网关的职责就是把它们抹平成统一的「消息进、流式 token 出」。

## 5.2 流式是默认，不是选项

`queryModelWithStreaming` 是生成器，意味着模型响应是**逐块流出来的**，而不是等整段生成完再返回。这对 Agent 至关重要：用户能看到打字机效果、思考过程（thinking 块）能边出边显示、而且——如第 4 章所见——**工具调用能在流式过程中就被检测到并开始执行**，不必等模型把整条消息说完。

文件里同时存在 `queryModelWithoutStreaming`（第 732 行）和 `executeNonStreamingRequest`（第 841 行），用于少数确实需要一次性拿到完整响应的场景，但主循环走的是流式那条。流式响应里混杂着文本增量、thinking 块、tool_use 块、usage 统计等多种 part，网关用 `switch (part.type)` 把它们逐类解析、归一化成内部消息类型再 yield 出去。

网关还要处理一些「脏活」。比如 `stripExcessMediaItems`（第 978 行）会在必要时剥掉过多的图片/媒体——这正是第 4 章提到的「media-size 恢复」在网关侧的配合：当一条消息塞了太多图导致超限，网关有能力裁掉多余媒体后重试。又比如 `getPromptCachingEnabled`、`getCacheControl`——prompt 缓存的开关和缓存断点检测（`promptCacheBreakDetection.ts`）也在这一层，因为不同 provider 的缓存语义不同。

## 5.3 withRetry：把抖动和限流挡在主循环之外

网关最硬核的部分是 `src/services/api/withRetry.ts`。模型 API 会超时、会 5xx、会限流（429）、会容量过载（529），如果每个错误都冒泡到主循环，Agent 就会脆得没法用。`withRetry` 是一个把这些瞬时故障吞掉、自动重试的生成器包装器。几个写死的关键常量定义了它的策略：

- `DEFAULT_MAX_RETRIES = 10`：默认最多重试 10 次。
- `MAX_529_RETRIES = 3`：529（容量过载）特殊对待，只重试 3 次——注释解释了原因：容量级联时每次重试都是 3-10 倍的网关放大，盲目重试会火上浇油。
- `BASE_DELAY_MS = 500`：退避基准 500 毫秒。

退避算法在 `getRetryDelay`（第 530 行）：

```ts
export function getRetryDelay(attempt, retryAfter, maxDelayMs = 32000) {
  const baseDelay = Math.min(BASE_DELAY_MS * 2 ** (attempt - 1), maxDelayMs)
  const jitter = Math.random() * 0.25 * baseDelay
  return baseDelay + jitter
}
```

这是教科书式的**指数退避 + 抖动（jitter）**：延迟随重试次数指数增长（500ms、1s、2s、4s……），封顶 32 秒，再叠加最多 25% 的随机抖动以避免「惊群效应」（所有客户端同时重试）。如果服务端返回了 `Retry-After` 头，则优先尊重它，但 `withRetry` 会再加一道 6 小时的封顶，注释明确说明——防止一个「病态的 header」让客户端无界等待。这是第 8 章「边界不信任外部输入」原则在网络层的体现：连服务端返回的重试头都不全信。

## 5.4 FallbackTriggeredError：换一台发动机

重试解决的是「同一个模型再试一次」，但有时候问题是「这个模型彻底不行了」。这时 `withRetry` 不再原地重试，而是抛出 `FallbackTriggeredError`（第 160 行，一个自定义 Error 子类）：

```ts
export class FallbackTriggeredError extends Error {
  constructor(...) {
    super(...)
    this.name = 'FallbackTriggeredError'
  }
}
```

最典型的触发点是 529 连续失败：当 `consecutive529Errors >= MAX_529_RETRIES` 时，`withRetry` 直接 `throw new FallbackTriggeredError(...)`（第 347 行）。这个异常会一路冒泡回第 4 章讲过的主循环内层 `while (attemptWithFallback)`——`catch (innerError)` 捕获到它，就把 `currentModel` 切成 `fallbackModel`、`attemptWithFallback = true`，清空缓冲后用备用模型重跑整个请求。

注意这里有个职责分工的精妙之处：**网关只负责「判定该 fallback 了」并抛异常，真正的「换模型重试」动作由主循环完成。** 网关不持有「下一个该用哪个模型」的状态，那是主循环的事。这样网关保持无状态、可测试，而 fallback 的编排逻辑集中在主循环一处。`onStreamingFallback` 回调（`Options` 类型第 705 行）则是网关通知主循环「流到一半发生了 fallback」的信号，对应第 4 章那段 tombstone 清理孤儿消息的逻辑。

还有一个 `CannotRetryError`（第 144 行）表示「这个错误根本不该重试」（比如鉴权失败、请求格式错误），它会直接冒泡而不浪费重试次数。`is529Error`（第 607 行）等判定函数则负责把五花八门的 provider 错误归类到「可退避重试 / 该 fallback / 不可重试」三档。

## 5.5 模型选择与请求装配的旁支

网关还要决定「这一次到底用哪个模型 id」。`getMainLoopModel()`（`src/utils/model/model.js`）按「显式 override → feature flag → `ANTHROPIC_MODEL` 环境变量 → settings → 默认值」的优先级链选出主循环模型——又是一条「越显式越优先」的级联，和 `getAPIProvider()` 一脉相承。

请求装配上还有不少 provider 相关的细节散落在 `claude.ts`：`getExtraBodyParams`（第 281 行）注入额外 body 参数，`getAPIMetadata`（第 498 行）拼装请求元数据，`configureTaskBudgetParams`（第 474 行）处理任务 token 预算，Bedrock 专属的 adapter（`providerUsage/adapters/bedrock.js`）统计用量。这些都是「让一个统一接口适配多家后端」必须付出的胶水代价。

读到这里你会发现一个反复出现的模式：**主循环对模型的全部认知，就是 `callModel` 这一个生成器函数。** provider 是谁、重试了几次、退避多久、有没有偷偷换了模型——这些复杂度全被 `src/services/api/` 这层网关吸收了。这正是分层抽象的价值：上层只关心「给我流式的模型输出」，下层独自面对真实世界的混乱。

## 本章小结

- 主循环的 `deps.callModel` 指向 `src/services/api/claude.ts` 的 `queryModelWithStreaming`（生成器）；尽管文件名叫 claude.ts，它伺候七种 provider。
- `getAPIProvider()` 定义了 `firstParty/bedrock/vertex/foundry/openai/gemini/grok` 七种，判定遵循「settings.modelType 优先 → 环境变量有固定先后 → 兜底 firstParty」的优先级。
- provider 差异通过 `if (getAPIProvider() === 'xxx')` 分派：OpenAI/Gemini/Grok 各有适配层，Bedrock/Vertex/Foundry 在 header 与签名上分支；网关把它们抹平成「消息进、流式 token 出」。
- 流式是默认路径，逐块产出文本/thinking/tool_use/usage，使打字机效果、思考可见、工具流式执行成为可能；`stripExcessMediaItems`、prompt 缓存控制也在这一层。
- `withRetry.ts` 用 `DEFAULT_MAX_RETRIES = 10`、`MAX_529_RETRIES = 3`、`BASE_DELAY_MS = 500` 定义重试策略；529 因容量级联放大而被特殊限制为 3 次。
- `getRetryDelay` 是指数退避 + 25% 抖动，封顶 32 秒；尊重服务端 `Retry-After` 但再加 6 小时封顶防止病态 header 导致无界等待（边界不信任）。
- `FallbackTriggeredError` 在连续 529 等场景抛出，由主循环内层 `while` 捕获并切换 `fallbackModel`——网关只判定「该 fallback」，换模型动作归主循环，保持网关无状态可测试。
- `getMainLoopModel()` 以「override → flag → ANTHROPIC_MODEL → settings → 默认」选模型，与 provider 选择同样遵循「越显式越优先」。
- 整层网关的价值在于：主循环对模型的全部认知收敛为 `callModel` 一个函数，真实世界的 provider 差异、限流、退避、fallback 全被这层吸收。

## 动手实验

1. **实验一：读懂 provider 优先级** — 打开 `src/utils/model/providers.ts` 的 `getAPIProvider()`，按代码顺序写出七种 provider 的判定优先级。思考：同时设置 `modelType: 'openai'` 和 `CLAUDE_CODE_USE_BEDROCK=1` 时，最终走哪个 provider？为什么？
2. **实验二：手算退避序列** — 用 `getRetryDelay` 的公式 `min(500 * 2^(attempt-1), 32000) + jitter`，算出第 1、3、5、7、10 次重试的基准延迟（不含 jitter）。确认它在第几次触到 32 秒封顶。
3. **实验三：追一次 fallback 的完整链路** — 在 `withRetry.ts` 找到 `throw new FallbackTriggeredError`（第 347 行附近）和 `consecutive529Errors >= MAX_529_RETRIES` 的判定；再回到 `src/query.ts` 找到 `catch` 捕获 `FallbackTriggeredError` 的位置。画出「529→529→529→抛异常→主循环切模型重试」的完整调用链。
4. **实验四：辨认三档错误** — 在 `withRetry.ts` 里找到 `CannotRetryError`、`FallbackTriggeredError`、`is529Error` 三者。归纳：哪些错误「原地退避重试」、哪些「触发 fallback 换模型」、哪些「直接冒泡不重试」？为每一档各举一个真实错误类型的例子。

> **下一章预告**：模型说它要调一个工具，但「工具」在 CCB 里到底是什么？第 6 章我们将解构统一工具计划的三层契约——`packages/agent-tools` 的协议层 `CoreTool`、宿主侧 `src/Tool.ts` 的超集，以及 `buildTool()` 工厂那套 fail-closed 的默认值，看一个工具从定义到被主循环接纳要满足哪些约束。
