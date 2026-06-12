# 第 6 章　多 Provider 抽象：让模型可插拔

> 一个 Agent 想活得久，第一件事是别把自己绑死在某一家模型厂商上。

第二部分我们把执行内核拆透了。但你应该记得，内核自始至终只认一个抽象——`StreamFn`。它从不知道屏幕另一端是 Anthropic 还是 OpenAI。那么「真正跟模型说话」这件事，到底由谁、怎么完成？答案是 `packages/ai`（`@earendil-works/pi-ai`）这个最底层的包。

这一章我们进入第三部分，从类型契约开始，看 pi 如何把「调模型」做成一件可插拔、可替换、可测试的事。

---

## 6.1 类型契约 `packages/ai/src/types.ts`：`Model` / `Context` / `Api`

`pi-ai` 的一切都建立在几个核心类型上。先看 `Api`——它是「用哪套协议跟模型对话」的标识：

```ts
export type Api = KnownApi | (string & {});
```

`KnownApi` 是 pi 内置认识的几套协议（`anthropic-messages`、`openai-completions`、`openai-responses`、`google-generative-ai`、`mistral-conversations` 等），而 `(string & {})` 这个技巧让 `Api` 在保留已知值自动补全的同时，**允许任意字符串**——也就是说，第三方可以注册一套全新的 API 协议，类型系统不会拦你。这是「可插拔」在类型层的第一个信号。

再看 `Model`，它描述一个具体模型的全部元数据：

```ts
export interface Model<TApi extends Api> {
	id: string;
	name: string;
	api: TApi;            // 用哪套协议
	provider: Provider;   // 哪家厂商
	baseUrl: string;
	reasoning: boolean;   // 是否支持思考
	thinkingLevelMap?: ThinkingLevelMap;  // pi 的 thinking level → 厂商特定值
	input: ("text" | "image")[];
	cost: { input; output; cacheRead; cacheWrite };  // $/百万 token
	contextWindow: number;
	maxTokens: number;
	headers?: Record<string, string>;
	compat?: ...;  // OpenAI 兼容 API 的覆盖项
}
```

注意 `Model` 用泛型 `<TApi>` 参数化，而 `compat` 字段会**根据 `TApi` 的具体值变成不同的兼容性配置类型**（`openai-completions` 对应 `OpenAICompletionsCompat`，`anthropic-messages` 对应 `AnthropicMessagesCompat`……）。这种「类型随协议变化」的设计，让 pi 能在一套统一接口下，精确表达每套协议特有的兼容性开关。

`cost` 直接写进 Model，意味着**成本核算是一等公民**——每个模型自带价格表，第 7 章我们会看到 token 用量如何顺着这个价格表一路核算到底。

最后是 `Context`——一次请求要发给模型的全部内容：系统提示词、消息列表、工具列表。它和第 3 章 `streamAssistantResponse` 里组装的 `llmContext` 一一对应。

## 6.2 注册表 `api-registry.ts`：`registerApiProvider` 与 `sourceId` 卸载

「可插拔」的核心机制是一张注册表。`packages/ai/src/api-registry.ts` 用一个 `Map` 维护「Api → Provider 实现」的映射：

```ts
const apiProviderRegistry = new Map<string, RegisteredApiProvider>();

export function registerApiProvider<TApi, TOptions>(
	provider: ApiProvider<TApi, TOptions>,
	sourceId?: string,
): void {
	apiProviderRegistry.set(provider.api, {
		provider: {
			api: provider.api,
			stream: wrapStream(provider.api, provider.stream),
			streamSimple: wrapStreamSimple(provider.api, provider.streamSimple),
		},
		sourceId,
	});
}
```

每个 Provider 必须提供两个函数：`stream`（底层流式）和 `streamSimple`（带 reasoning 的统一流式）。注册时它们会被 `wrapStream` 包一层，里面做一件防御性的事——**校验 model 的 api 与注册的 api 匹配**：

```ts
function wrapStream(api, stream): ApiStreamFunction {
	return (model, context, options) => {
		if (model.api !== api) {
			throw new Error(`Mismatched api: ${model.api} expected ${api}`);
		}
		return stream(model, context, options);
	};
}
```

注册表还有两个容易被忽略但很重要的能力：

- **`sourceId`**：注册时可以打一个「来源标记」。配套的 `unregisterApiProviders(sourceId)` 能把某个来源注册的所有 provider 一次性卸载。这对**扩展系统**至关重要——一个扩展可以注册自己的 provider，卸载扩展时干净地撤销，不留垃圾。
- **`clearApiProviders()`**：清空所有注册，主要用于测试隔离。

> **给 Agent 开发者的启示**：用「注册表 + sourceId」管理可插拔组件，比用一堆 `if (provider === "anthropic")` 的硬编码分支优雅得多。新增一家厂商 = 写一个 provider + 注册一次，内核与其他 provider 完全无感。

那么内置的几家厂商在哪注册？看 `providers/register-builtins.ts`——它惰性地（lazy）导入并注册 Anthropic、OpenAI（completions/responses）、Google、Mistral、Bedrock 等。而这个文件在 `stream.ts` 的第一行就被 `import`，确保任何人用 `streamSimple` 前，内置 provider 已经就位。

## 6.3 门面 `stream.ts`：四个对外函数

回到第 2 章那条最关键的接缝。`packages/ai/src/stream.ts` 是整个 `pi-ai` 对外的门面，只有四个函数，但内核（`StreamFn`）借用的就是其中的 `streamSimple`：

```ts
export function streamSimple<TApi extends Api>(
	model: Model<TApi>,
	context: Context,
	options?: SimpleStreamOptions,
): AssistantMessageEventStream {
	const provider = resolveApiProvider(model.api);
	return provider.streamSimple(model, context, withEnvApiKey(model, options));
}
```

它做的事极简：**根据 `model.api` 从注册表里查出对应 provider，调它的 `streamSimple`。** 一行路由，把「跟谁说话」的决策完全数据化了。

四个函数的关系是两个维度的笛卡尔积：

| | 流式（返回 EventStream） | 非流式（返回最终消息） |
|---|---|---|
| **底层**（`StreamOptions`） | `stream` | `complete` |
| **统一**（带 `reasoning`） | `streamSimple` | `completeSimple` |

`complete` / `completeSimple` 不过是「订阅流、等最终结果」的便利封装：

```ts
export async function completeSimple<TApi>(model, context, options?): Promise<AssistantMessage> {
	const s = streamSimple(model, context, options);
	return s.result();
}
```

还记得第 5 章 `AgentHarness` 压缩时调模型生成摘要吗？那种「只要最终结果、不需要流式 UI」的场景，用的就是 `completeSimple` 这类非流式函数。

## 6.4 环境变量 API Key 的隐式注入（`withEnvApiKey`）

`streamSimple` 里那个 `withEnvApiKey(model, options)` 是个贴心的小设计。它的逻辑是：**如果调用方没显式传 apiKey，就尝试从环境变量里按 provider 取一个填进去**：

```ts
function withEnvApiKey<TOptions>(model, options): TOptions | undefined {
	if (hasExplicitApiKey(options?.apiKey)) return options;  // 显式优先
	const apiKey = getEnvApiKey(model.provider);
	if (!apiKey) return options;
	return { ...options, apiKey } as TOptions;
}
```

优先级很清楚：**显式传入 > 环境变量 > 没有**。这让「本地随手跑一下」（靠 `ANTHROPIC_API_KEY` 等环境变量）和「生产里动态注入 key」（显式传 apiKey、甚至用第 4 章的 `getApiKey` 钩子每轮刷新）两种姿态共用一套代码。

注意这里有个层次：`withEnvApiKey` 是**一次性**的兜底（请求构造时取一次环境变量），而第 4 章讲的 `getApiKey` 钩子是**每轮**重新解析（应对会过期的 OAuth token）。两者不冲突——`getApiKey` 解析出的 key 会作为「显式 apiKey」传进来，从而短路掉 `withEnvApiKey` 的环境变量兜底。这套分层我们在第 7 章讲 OAuth 时还会回来。

至此，「模型可插拔」的骨架就清楚了：统一的类型契约（`Model`/`Context`/`Api`）+ 一张注册表 + 一个一行路由的门面。下一章，我们钻进 provider 内部，看流式协议、消息转换、重试和认证这些「与模型对话的脏活累活」是怎么干的。

---

## 本章小结

- `Api = KnownApi | (string & {})` 在类型层就为「第三方注册新协议」留了门；`Model<TApi>` 用泛型让 `compat` 随协议变化，并把成本（`cost`）做成一等公民。
- `api-registry.ts` 用 `Map` 维护「Api → Provider」映射，`registerApiProvider` 注册时校验 api 匹配；`sourceId` 支持按来源批量卸载（扩展系统的关键）。
- `stream.ts` 的四个门面函数是两个维度（流式/非流式 × 底层/统一）的笛卡尔积；`streamSimple` 的实现就是「按 `model.api` 路由」一行。
- `withEnvApiKey` 实现「显式 key > 环境变量 > 无」的优先级兜底，与第 4 章每轮刷新的 `getApiKey` 钩子分层协作。

下一章，深入 provider 内部，看流式协议、`transform-messages`、重试与 OAuth。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/ai/src/types.ts` | `Api`/`Model`/`Context`/`SimpleStreamOptions` 等核心契约 |
| `packages/ai/src/api-registry.ts` | provider 注册表、`sourceId` 卸载 |
| `packages/ai/src/stream.ts` | 对外门面：`stream`/`complete`/`streamSimple`/`completeSimple` |
| `packages/ai/src/providers/register-builtins.ts` | 内置厂商的惰性注册 |

## 动手实验

1. 在 `api-registry.ts` 里跟踪 `sourceId` 的用法，设想一个场景：你写了一个扩展，注册了一个自定义 provider。卸载扩展时，你会怎么用 `unregisterApiProviders` 确保不留残留？
2. 写一段最小代码：构造一个 `Model`（api 用 `faux`），调用 `completeSimple`，打印返回的 `AssistantMessage`。对照 6.3 的表格，理解「非流式 = 流式 + `.result()`」。
3. 把 `ANTHROPIC_API_KEY` 环境变量设好，再分别「传 apiKey」和「不传 apiKey」调用 `streamSimple`，用断点验证 `withEnvApiKey` 的优先级逻辑。
