# 第 7 章　流式协议、消息转换与认证

> 模型 API 之间的差异，不应该由循环来背。

上一章讲的是「找谁说话」；这一章讲「怎么把话说到位」。pi 面对的是一个高度碎片化的世界：Anthropic 有 thinking 签名，OpenAI 有 encrypted reasoning，不同 API 对工具调用 ID 的格式要求不同，图片支持能力也各不相同。本章看 pi 如何把这些差异收进 `pi-ai` 一层，并保持 Agent 循环的无感。

---

## 7.1 `AssistantMessageEvent`：一条严格的流协议

`packages/ai/src/types.ts:806` 定义了流事件。可以分成三段：

```ts
// 开场
{ type: "start"; partial: AssistantMessage }

// 增量（按内容块）
{ type: "text_start" | "text_delta" | "text_end"; contentIndex; partial }
{ type: "thinking_start" | "thinking_delta" | "thinking_end"; contentIndex; partial }
{ type: "toolcall_start" | "toolcall_delta"; contentIndex; partial }
{ type: "toolcall_end"; contentIndex; toolCall; partial }

// 收尾（二选一）
{ type: "done"; reason: "stop" | "length" | "toolUse" | "deferred"; message }
{ type: "error"; reason: "aborted" | "error"; error }
```

几条契约值得逐字读：

1. **`start` 必须最先出现**。没有 `start` 就不该有增量，也不该有 `done`。
2. **`partial` 是「累积快照的共享对象」，不是事件时点的不可变副本**。文本在 `text_start` 时为空，随 delta 增长，到 `text_end` 才权威。消费方如果要保存快照，必须自己 clone。
3. **`toolcall_start` 的参数是 provider 相关的**。有的 provider 会在 start 时就给出部分参数，有的要到 delta 才给。不要假设。
4. **`done` 与 `error` 是终止事件**，之后不应再有事件。
5. **`error` 的 `error` 字段是一条完整的 `AssistantMessage`**，包含 `stopReason` 与 `errorMessage`。错误也是消息。

`Agent` 的 `streamAssistantResponse` 只处理这组事件：`start` 推入 partial，增量替换最后一条消息并发 `message_update`，`done`/`error` 替换为最终消息并发 `message_end`。它不关心 provider 的方言——方言在更下层已经被吸收。

## 7.2 为什么需要 `transformMessages`

不同 provider 对「同一个对话」的合法性要求不同。把 A 模型的会话直接发给 B 模型，常见爆炸点有：

| 问题 | 典型触发 |
|---|---|
| thinking 块带签名 | 从 Anthropic 切到 OpenAI，签名对新模型无意义 |
| encrypted reasoning | OpenAI 的推理块只能在原模型重放 |
| 工具调用 ID 格式 | OpenAI Responses 的 ID 可能超长且含 `\|`，Anthropic 只接受 `^[a-zA-Z0-9_-]+$` 且 ≤64 字符 |
| 孤立工具调用 | 历史被裁剪后，assistant 有 toolCall 但没有对应 toolResult |
| 图片不被支持 | 新模型是纯文本模型，但历史里有图片 |
| 错误/中止的 assistant 消息 | 包含不完整推理或截断工具调用，重放会触发 API 错误 |

`packages/ai/src/api/transform-messages.ts` 的 `transformMessages(messages, model, normalizeToolCallId?)` 统一处理这些情况。它被 Anthropic、OpenAI completions、OpenAI responses、Google、Bedrock、Mistral 等所有主流 API 实现共同调用。

### 第一遍：按目标模型改写内容

- **图片降级**：目标模型 `input` 不含 `image` 时，用户图片和工具图片都被替换为占位文本。
- **thinking 块**：只有「同 provider + 同 api + 同 model」才保留；跨模型时丢弃可读 thinking。带 `redacted` 或 `thinkingSignature` 的块即使文本为空也保留，因为它们参与重放。
- **text 块**：跨模型时重建为纯文本块，去掉 provider 特有的附加字段。
- **toolCall**：跨模型时去掉 `thoughtSignature`；若提供了 `normalizeToolCallId`，则把 ID 映射成目标 API 可接受的格式，并记录映射。
- **toolResult**：按映射表同步改写 `toolCallId`。

### 第二遍：修复工具调用配对

这一遍是「结构修复」：

- 遇到新的 assistant 消息时，先为上一批未收到结果的 toolCall 补一条 `"No result provided"` 的错误 toolResult；
- 遇到 user 消息打断工具流时，同样补齐；
- 会话结束时仍未闭合的 toolCall，也补齐；
- **跳过 `stopReason` 为 `error` / `aborted` 的 assistant 消息**，避免重放不完整回合；
- 系统消息如果落在「toolCall 与 toolResult 之间」，会先被暂存，等结果补齐后再发出，且不会造成重复结果。

这套处理让「随便切换模型」在协议层变得可行。注意它只保证**协议合法**，不保证**语义最优**——跨模型丢失 thinking 是必然的代价。

## 7.3 Provider 方言：适配器在 `api/` 层

`packages/ai/src/api/` 下是一组「API 方言适配器」：

| 文件 | 方言 |
|---|---|
| `anthropic-messages.ts` | Anthropic Messages（含 thinking、cache control、tool use） |
| `openai-completions.ts` | OpenAI Chat Completions 及所有兼容端点 |
| `openai-responses-shared.ts` / `openai-responses.ts` | OpenAI Responses（含 encrypted reasoning） |
| `openai-codex-responses.ts` | Codex 专用 Responses 变体 |
| `google-shared.ts` / `google-generative-ai.ts` | Gemini |
| `bedrock-converse-stream.ts` | AWS Bedrock Converse |
| `mistral-conversations.ts` | Mistral |
| ... | 以及各 provider 的流式包装 |

每个适配器做三件事：

1. 把统一的 `TranscriptContext` 翻译成该 API 的请求体；
2. 把该 API 的流事件翻译成 `AssistantMessageEvent`；
3. 处理方言特有的错误、usage、缓存字段。

以 Anthropic 为例，`anthropic-messages.ts:1176` 先调用 `transformMessages`，再构造 messages、system、tools。**转换发生在适配器内部，而不是循环里**，所以新增一个 provider 不需要改内核一行代码。

## 7.4 失败重试：在「消息」层而不是「异常」层

`packages/ai/src/utils/retry.ts` 提供了 `retryAssistantCall`：

```ts
export async function retryAssistantCall(
	produce: () => Promise<AssistantMessage>,
	policy?: RetryPolicy,
	signal?: AbortSignal,
	callbacks?: RetryCallbacks,
): Promise<AssistantMessage>
```

它的判断依据不是 HTTP 状态码，而是最终 `AssistantMessage`：

- `isRetryableAssistantError(message)` 决定是否重试；
- 非重试错误（配额、鉴权、参数错误）立即返回；
- 中止立即返回；
- 退避时长由 `retryDelayMs(policy, attempt)` 计算，并受 `maxAgentDelayMs` 限制。

这层重试主要服务**摘要生成**（压缩、分支摘要）这类一次性调用：流突然断开时，不应该让整个压缩失败。Agent 主循环的重试在 `AgentSession` 层（第 5 章），那层还要处理「失败消息从投影中省略」等会话语义。**两层重试，职责不同，不要混为一谈。**

## 7.5 认证：provider 拥有流程，运行时负责解析

`ProviderAuth` 只有两个可选字段：

```ts
export interface ProviderAuth {
	apiKey?: ApiKeyAuth;
	oauth?: OAuthAuth;
}
```

但它的语义比看起来重：

- `apiKey` 是**必选兜底**。即使是纯环境变量、AWS profile、ADC、免密钥本地服务，也要提供一个 `resolve()` 能报告「是否已配置」的 apiKey auth。
- `oauth` 负责登录、刷新、凭据持久化。OAuth 流程按 provider 实现，pi 内置了 Anthropic、OpenAI ChatGPT、OpenAI Codex、GitHub Copilot、Kimi Coding、OpenRouter、xAI 等流程，PKCE 实现在 `auth/oauth/pkce.ts`。
- `resolve()` 可以访问 `AuthContext`（环境变量、凭据存储、fetch 等），返回 `{ apiKey, headers, baseUrl, env }`。

`Models.getAuth()` 是统一入口：

```ts
getAuth(providerId): Promise<AuthResult | undefined>
getAuth(model): Promise<AuthResult | undefined>
```

返回 `undefined` 表示未配置；抛 `ModelsError` 表示配置了但解析失败（OAuth 刷新失败 / 凭据存储失败）。请求路径会把后者转成流错误。

### 为什么每轮都要重新解析？

因为 OAuth token 会过期，而一次 Agent 任务可能跑几十分钟。`AgentLoopConfig.getApiKey` 在**每次请求前**调用，`AgentSession` 则通过 `ModelRuntime.getAuth` 拿到最新的 key/headers/baseUrl。这样即使用户在任务中途刷新了登录，下一轮也能无缝继续。

### 凭据来源的优先级

`coding-agent` 的文档与实现一致：

1. 运行时 `--api-key`；
2. 存储的 `auth.json`；
3. `models.json` 中的 `apiKey`；
4. provider 的环境变量或云环境凭据。

后两者由 provider 自己的 `resolve()` 实现，因此新增 provider 时可以自定义顺序，而不必改运行时。

## 7.6 `faux` provider：把「模型」变成可编程对象

`packages/ai/src/providers/faux.ts` 是 pi 测试体系的基石。它提供：

```ts
fauxText("hello")
fauxThinking("...")
fauxToolCall("read", { path: "a.txt" }, { id: "call-1" })
fauxAssistantMessage(content, { stopReason, errorMessage })
```

以及一个按脚本返回响应的 `FauxProvider`。你可以精确控制：

- 每条消息的内容块（文本、thinking、toolCall）；
- `stopReason`（`stop` / `length` / `toolUse` / `error` / `aborted`）；
- usage 与错误信息；
- 流式增量的节奏（用于测试 UI 与中断）。

这带来的测试能力非常关键：

- **确定性**：同样的输入永远得到同样的输出，CI 不会因为模型波动而红；
- **可构造边界**：可以精确制造 `length` 截断、孤立 toolCall、thinking 签名等真实 API 很难复现的情况；
- **零成本**：不联网、不烧 token，整个循环可以在毫秒级跑完。

> **工程建议**：如果你在搭自己的 Agent，先把 faux provider 写出来，再写真实 provider。它会强迫你把「请求/响应的数据形状」想清楚。

## 7.7 一次流式请求的完整时序

```
AgentSession.prepareRequest
  → Agent.streamFunction
    → Models.streamSimple(model, Context, options)
      → normalizeContext
      → applyAuth（provider.auth.resolve）
      → transformHeaders
      → Provider.streamSimple
        → 方言适配器（如 anthropic-messages）
          → transformMessages（跨模型归一化 + 工具配对修复）
          → HTTP / SDK 请求
          → 原始流事件
        ← AssistantMessageEvent（start/delta/done/error）
      ← AssistantMessageEventStream
    ← Agent 消费事件，发 message_start/update/end
  ← 最终 AssistantMessage 进入工具执行或回合结束
```

理解这条链路后，你就能定位绝大多数「模型明明返回了，但 Agent 行为不对」的问题：是转换丢了内容？是认证解析出了错？是流事件没有按契约收尾？还是上层把 partial 当成了最终消息？

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/ai/src/types.ts` | `AssistantMessageEvent`、`Usage`、消息类型 |
| `packages/ai/src/api/transform-messages.ts` | 跨 provider 消息归一化与工具配对修复 |
| `packages/ai/src/api/*.ts` | 各方言适配器 |
| `packages/ai/src/auth/` | 凭据解析、OAuth、PKCE、凭据存储 |
| `packages/ai/src/utils/retry.ts` | 摘要/一次性调用的重试 |
| `packages/ai/src/providers/faux.ts` | 确定性测试 provider |
| `packages/coding-agent/docs/providers.md` | 各 provider 的环境变量与配置 |

## 动手实验

1. 构造一段包含 Anthropic thinking 块和 OpenAI 风格长工具 ID 的对话，分别用 `transformMessages` 以 Anthropic 模型和 OpenAI 模型为目标转换，打印结果，验证 thinking 与 ID 的差异。
2. 造一个「assistant 有 toolCall、但没有 toolResult、后面紧跟 user 消息」的历史，跑 `transformMessages`，观察合成的错误结果插入位置。
3. 用 `fauxAssistantMessage` 构造 `stopReason: "length"` 的响应，接到 `runAgentLoop`，验证工具不会执行、模型收到什么样的错误提示。
4. 阅读 `auth/resolve.ts` 的 `resolve()` 分支，列出 API Key 与 OAuth 两条路径在「未配置 / 已配置 / 刷新失败」三种状态下的返回值与异常。
