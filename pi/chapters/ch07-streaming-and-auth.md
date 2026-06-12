# 第 7 章　与模型对话的脏活累活：流、转换与认证

> 上一章我们站在门面看「调谁」，这一章钻进门后看「怎么调」——而真正难的，从来不是顺利的那条路，是出错、跨厂商、token 过期的那些岔路。

第 6 章我们看清了 `pi-ai` 的骨架：类型契约 + 注册表 + 一行路由的门面。但门面之后，每个 provider 内部都在干一堆「与模型对话的脏活累活」——把无尽的流式分片收敛成一条消息、把 pi 的统一消息翻译成各家厂商能懂的格式、在 token 过期时悄悄换一把新钥匙。这一章我们就把这些藏在 provider 内部的工程细节摊开。

---

## 7.1 `AssistantMessageEventStream`：把「错误编码进流」做成类型

第 2 章我们立下过一条红线：`StreamFn` 契约要求**失败不许抛异常，必须编码进返回的流**。这条契约在 `pi-ai` 里不是一句口号，而是被 `AssistantMessageEventStream` 这个类**用类型钉死的**。

先看它的基类 `EventStream<T, R>`（`packages/ai/src/utils/event-stream.ts`）。它是一个通用的「可异步迭代 + 有最终结果」的流：你能 `for await` 它逐个拿事件，也能 `await stream.result()` 等它的最终产物。它的精妙在构造函数的两个回调——「什么事件算结束」和「结束时怎么从那个事件里抽出最终结果」：

```ts
export class EventStream<T, R = T> implements AsyncIterable<T> {
	constructor(isComplete: (event: T) => boolean, extractResult: (event: T) => R) { ... }

	push(event: T): void {
		if (this.isComplete(event)) {
			this.done = true;
			this.resolveFinalResult(this.extractResult(event));
		}
		// ...派发给等待的消费者，或入队
	}
}
```

而 `AssistantMessageEventStream` 就是把这两个回调实例化成「模型响应」的语义：

```ts
export class AssistantMessageEventStream extends EventStream<AssistantMessageEvent, AssistantMessage> {
	constructor() {
		super(
			(event) => event.type === "done" || event.type === "error",   // 两种结束
			(event) => {
				if (event.type === "done") return event.message;            // 成功 → 最终消息
				else if (event.type === "error") return event.error;        // 失败 → 也是一条消息
				throw new Error("Unexpected event type for final result");
			},
		);
	}
}
```

**看清这里的设计：`done` 和 `error` 都是「合法的结束事件」，而且两者抽出来的最终结果都是一条 `AssistantMessage`。** 失败不是异常，是流里一个 `type: "error"` 的事件，它携带的 `error` 字段本身就是一条 `stopReason: "error"` 或 `"aborted"` 的助手消息。于是上层无论成功失败，`await stream.result()` 拿到的永远是同一种类型——一条消息。这就是「错误即数据」在类型层的兑现。

再看完整的事件联合 `AssistantMessageEvent`（`packages/ai/src/types.ts`），它描述了一次流式响应里能出现的所有「分片」：

```ts
export type AssistantMessageEvent =
	| { type: "start"; partial: AssistantMessage }
	| { type: "text_start" | "text_delta" | "text_end"; contentIndex; ... }
	| { type: "thinking_start" | "thinking_delta" | "thinking_end"; contentIndex; ... }
	| { type: "toolcall_start" | "toolcall_delta" | "toolcall_end"; contentIndex; ... }
	| { type: "done"; reason: "stop" | "length" | "toolUse"; message: AssistantMessage }
	| { type: "error"; reason: "aborted" | "error"; error: AssistantMessage };
```

注意几个细节：

- 每个增量事件都带 `contentIndex`——因为一条消息里可能有多个内容块（一段文本 + 一个思考块 + 两个工具调用），`contentIndex` 标明这个 delta 属于第几块。
- 文本、思考、工具调用都遵循 `start / delta / end` 三段式，UI 可以据此渲染「正在打字」「正在思考」「正在拼工具参数」的实时状态。
- 几乎每个事件都带一个 `partial: AssistantMessage`——这是「到目前为止累积的完整消息快照」。消费者不必自己拼分片，随时能拿到当前的全貌。

> **给 Agent 开发者的启示**：流式协议最容易写乱的地方，是「分片」和「完整结果」的关系。pi 的做法是让每个增量事件都背着一个 `partial` 全量快照，再用 `done`/`error` 给出终态——消费者既能要实时增量（for await），也能要最终结果（result()），两套姿态零成本共存。

## 7.2 `transform-messages.ts`：跨 Provider 的消息归一化

pi 的会话历史里存的是它自己的 `AssistantMessage`，里面可能混着 Anthropic 的「带签名的思考块」、OpenAI 的「加密 reasoning」、Google 的「thought signature」。当你**换一个模型**接着聊（pi 支持会话中途换模型），这些上一个模型留下的「私有痕迹」就成了麻烦——新模型既看不懂别家的加密思考，也可能因为工具调用 ID 格式不合法而直接拒绝请求。

`transformMessages`（`packages/ai/src/providers/transform-messages.ts`）就是发往模型前的最后一道「归一化」工序。它做三件事：

**1. 降级不支持的图片。** 如果目标模型不支持视觉（`model.input` 不含 `"image"`），就把消息里的图片块替换成占位文本，并合并连续占位符，避免一串 `(image omitted)` 刷屏：

```ts
function downgradeUnsupportedImages(messages, model) {
	if (model.input.includes("image")) return messages;  // 支持就原样放行
	// 否则把 user / toolResult 里的 image 块换成 "(image omitted...)" 文本
}
```

**2. 按「是否同一个模型」决定思考块的去留。** 这是最精妙的一段。它用 `provider + api + model.id` 三元组判断当前历史里的某条助手消息是否出自「当前正要调用的同一个模型」：

```ts
const isSameModel =
	assistantMsg.provider === model.provider &&
	assistantMsg.api === model.api &&
	assistantMsg.model === model.id;
```

- **加密的思考块（`redacted`）**：只有同模型才保留，跨模型直接丢弃——因为它是别家不透明的密文，replay 给新模型只会报错。
- **带签名的思考块**：同模型保留（replay 时需要签名），跨模型则降级成纯文本或丢弃。
- **工具调用的 `thoughtSignature`（Google 特有）**：跨模型时删掉。
- **工具调用 ID**：跨模型时通过 `normalizeToolCallId` 钩子归一化。注释点明了真实痛点——OpenAI Responses API 生成的 ID 长达 450+ 字符且含 `|`，而 Anthropic 要求 ID 匹配 `^[a-zA-Z0-9_-]+$` 且最长 64 字符。不归一化就会跨厂商崩。

**3. 补全「孤儿工具调用」的合成结果。** 第二趟扫描里，如果某个工具调用没有对应的工具结果（比如上一轮被中断了），就合成一条 `isError: true` 的空结果塞进去：

```ts
result.push({
	role: "toolResult",
	toolCallId: tc.id,
	content: [{ type: "text", text: "No result provided" }],
	isError: true,
	// ...
});
```

为什么必须补？因为几乎所有厂商的 API 都要求「每个 tool_call 必须跟着一个 tool_result」，否则报错。pi 宁可塞一条「没有结果」的假结果，也不让请求崩——又一次「错误即数据」。同理，errored/aborted 的助手消息会被整条跳过，避免把半截的 reasoning 重放给模型。

> 这一节是整本书「跨 Provider 抽象到底难在哪」的最佳答案：难的不是「发个 HTTP 请求」，而是**把 N 家厂商各自的私有格式、私有约束、私有错误条件，收敛成一套自己能稳定 replay 的统一历史**。`transform-messages.ts` 就是这层脏活的集中地。

## 7.3 四套协议的形态差异：同一个抽象，不同的「方言」

pi 内置认识好几套 API 协议（第 6 章的 `KnownApi`），每套对应 `providers/` 下一个文件。它们对外都满足同一个 `ApiProvider` 契约（`stream` + `streamSimple`），但内部「方言」差别很大：

| 协议（`api`） | 文件 | 形态要点 |
|---|---|---|
| `anthropic-messages` | `anthropic.ts` | 原生 SSE，`content_block_start/delta/stop`；思考块带 signature；支持 prompt caching |
| `openai-completions` | `openai-completions.ts` | 经典 Chat Completions；工具调用以 `tool_calls` 增量拼接；最广的兼容面（很多三方网关都仿它） |
| `openai-responses` | `openai-responses.ts` | 新的 Responses API；reasoning 以加密形式返回；工具调用 ID 超长 |
| `openai-codex-responses` | `openai-codex-responses.ts` | Codex 专用的 Responses 变体，配套 Codex OAuth |
| `google-generative-ai` | `google.ts` | Gemini；工具调用带 `thoughtSignature` |
| `mistral-conversations` | `mistral.ts` | Mistral 对话协议 |
| `bedrock` | `bedrock.ts` | AWS Bedrock 封装，复用 Anthropic 形态但走 AWS 认证 |

每个 provider 干的活本质一样：**把厂商的原生流式分片（通常是 SSE），翻译成 pi 统一的 `AssistantMessageEvent` 序列，push 进 `AssistantMessageEventStream`。** 这就是为什么内核能 provider 无关——所有「方言差异」都被各 provider 在这一层吸收掉了，吐出来的全是统一的事件。

第 6 章提过 `Model` 上的 `compat` 字段会随 `TApi` 变成不同类型。它的用武之地正在这里：面对「自称兼容 OpenAI、实则有细微差异」的三方网关，`compat` 让你在不写新 provider 的前提下，开关掉某些不兼容的特性（比如某些网关不支持 `tool_choice`，或者要求换一种 streaming 格式）。

## 7.4 `faux` provider：不联网、不烧 token 地测整个循环

`packages/ai/src/providers/faux.ts` 是一个「假模型」，它把第 2 章「可测试性」那条接缝落到了实处。它注册成 `api: "faux"` 的 provider，但内部不发任何网络请求，而是把你**预先编排好的内容**当成「模型的回答」吐成事件流。

它提供了一组顺手的构造器，让你像写剧本一样编排假回答：

```ts
fauxText("好的，我来帮你读文件")                          // 一段文本
fauxThinking("先想想该用哪个工具")                        // 一个思考块
fauxToolCall("read", { path: "a.txt" })                  // 一个工具调用
```

把这些拼成一个数组，faux 就会按 `start → delta → end → done` 的节奏把它们回放成标准事件流——和真实 provider 吐出来的形态一模一样。于是你可以这样测整个 Agent 循环：

> 第一次调用让 faux 返回「一个 read 工具调用」，第二次返回「一段纯文本」。用它驱动 `runAgentLoop`，你就能在**完全不联网、不花一分钱 token**的情况下，验证第 3 章那套双层循环「调工具 → 喂回结果 → 再生成 → 自然停下」的完整行为。

这正是第 3 章「动手实验 3」要你写的那个「最小假 `StreamFn`」——而 pi 把它做成了一等的、可复用的内置 provider。一个 Agent 内核能不能被认真测试，往往就看它有没有这样一个 faux 层。第 16 章我们还会回到它，谈「可信度工程」。

## 7.5 OAuth 与会过期的短期 token：`getApiKey` 每轮重解析

最后一块脏活是认证。最简单的认证是「一把静态 API Key」——第 6 章的 `withEnvApiKey` 已经覆盖了。但越来越多的场景用的是**会过期的短期 token**：Claude Pro/Max 的 OAuth、OpenAI Codex 的 OAuth、GitHub Copilot 的 token……这些 token 活不了多久，一次长任务跑到一半就可能失效。

pi 在 `packages/ai/src/utils/oauth/` 下为每家实现了完整的 OAuth 流程（`anthropic.ts`、`openai-codex.ts`、`github-copilot.ts`、外加通用的 `device-code.ts` 和 PKCE 工具）。以 Anthropic 为例，它走的是标准的「PKCE + 本地回调服务器」授权码流程：

```ts
const AUTHORIZE_URL = "https://claude.ai/oauth/authorize";
const TOKEN_URL = "https://platform.claude.com/v1/oauth/token";
// 起一个本地 http server 接 /callback，拿到 code 换 token
```

但 OAuth 的「拿到 token」只是开始，真正的工程难点是**token 会过期，得在合适的时机刷新**。这里就接回了第 4 章那个钩子 `getApiKey`——它的契约是**每轮请求前重新解析一次 key**。两者的分工再清晰不过：

| 机制 | 时机 | 适用 |
|---|---|---|
| `withEnvApiKey`（第 6 章） | 请求构造时**一次性**兜底 | 静态环境变量 key |
| `getApiKey` 钩子（第 4 章） | **每轮**请求前重解析 | 会过期的 OAuth token |

`getApiKey` 解析出的 key 会作为「显式 apiKey」传进 `streamSimple`，从而**短路掉** `withEnvApiKey` 的环境变量兜底（显式 > 环境变量）。于是一次长任务里，即便 OAuth token 在第 5 轮过期了，第 6 轮 `getApiKey` 也会拿到一把刷新后的新 key——循环本身对此毫无感知。这就是「机制稳定、认证灵活」：内核只管「每轮问一次 key」，至于这把 key 是静态的、刷新来的、还是远程注入的，全是策略。

至此，「与模型对话」这件事被我们从门面（第 6 章）一直拆到了流、转换、协议方言与认证（第 7 章）。从下一章起，我们离开 `pi-ai`，上升到应用层 `pi-coding-agent`，看 pi 如何把抽象的「工具」变成 read/bash/edit 这些真正能动手干活的能力。

---

## 本章小结

- `AssistantMessageEventStream` 用「`done`/`error` 都是合法结束事件、且都抽出一条 `AssistantMessage`」的设计，把「错误即数据」钉死在类型层；每个增量事件都背一个 `partial` 全量快照。
- `transform-messages.ts` 是发往模型前的归一化工序：降级不支持的图片、按 `isSameModel` 决定思考块去留、归一化超长工具调用 ID、为孤儿工具调用补合成结果——跨 Provider 抽象的难点集中地。
- 内置的多套协议（anthropic-messages / openai-completions / openai-responses / codex-responses / google / mistral / bedrock）对外满足同一契约，对内把各家「方言」翻译成统一事件流；`compat` 字段应对「自称兼容」的三方网关。
- `faux` provider 让整个循环可以不联网、不烧 token 地测试，是「可测试性」接缝的一等落地。
- OAuth（`utils/oauth/`）解决「拿到短期 token」，`getApiKey` 钩子解决「每轮重解析以应对过期」，与一次性的 `withEnvApiKey` 分层协作，内核对认证方式完全无感。

下一章，进入应用层 `pi-coding-agent`，看 `ToolDefinition` 与 `AgentTool` 如何把「工具」这个抽象变成能动手的能力。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/ai/src/utils/event-stream.ts` | `EventStream<T,R>` 与 `AssistantMessageEventStream`，「错误即数据」的类型落地 |
| `packages/ai/src/types.ts` | `AssistantMessageEvent` 事件联合（start/delta/end/done/error） |
| `packages/ai/src/providers/transform-messages.ts` | 跨 Provider 消息归一化：图片降级、思考块去留、ID 归一、孤儿补全 |
| `packages/ai/src/providers/faux.ts` | 测试用假 provider，`fauxText`/`fauxThinking`/`fauxToolCall` |
| `packages/ai/src/utils/oauth/` | 各厂商 OAuth 流程（anthropic / openai-codex / github-copilot / device-code / pkce） |

## 动手实验

1. 用 faux 编排一个「思考块 + 文本 + 一个工具调用」的假回答，`for await` 它的事件流，把每个事件的 `type` 和 `contentIndex` 打印出来。对照 7.1 的事件联合，理解 `start/delta/end` 三段式和 `partial` 快照的关系。
2. 在 `transform-messages.ts` 里给 `isSameModel` 那段加日志：构造一段「先用模型 A 聊、含一个带签名思考块，再换模型 B 继续」的历史，观察换模型时思考块是被保留、降级成文本、还是丢弃。
3. 阅读 `utils/oauth/anthropic.ts` 的回调服务器逻辑，再回看第 4 章 `getApiKey` 的契约，画一张时序图：一次长任务里 OAuth token 在第 N 轮过期，第 N+1 轮的 key 是从哪一步刷新出来、又是怎样短路掉 `withEnvApiKey` 的。
