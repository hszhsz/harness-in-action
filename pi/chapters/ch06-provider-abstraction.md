# 第 6 章　Models 运行时：Provider 不再是一张全局表

> 旧版 pi 用一张全局 API 注册表找 provider；1.3 把「谁拥有模型、谁负责认证」收回到对象里。

上一版电子书讲的 `api-registry.ts`、`registerApiProvider()`、`streamSimple()` 门面，在 1.3 已经被 `Models` 运行时取代。这一章讲清楚新的三件套：`Provider`、`Models`、`ModelRuntime`，以及它们如何支撑 pi 的多模型、多认证、动态目录。

---

## 6.1 三个对象，各管一段

| 对象 | 位置 | 职责 |
|---|---|---|
| `Provider` | `packages/ai/src/models.ts:157` | 一个 provider 的全部行为：模型列表、认证、流式实现、图片/分类能力 |
| `Models` | `packages/ai/src/models.ts:257` | provider 集合 + 认证解析 + 请求归一化 + 便捷方法 |
| `ModelRuntime` | `packages/coding-agent/src/core/model-runtime.ts:173` | 产品层扩展：实现 `Models`，叠加虚拟模型、用户自定义 provider、目录刷新 |

依赖方向是：`Agent` 依赖一个 `StreamFn`；`AgentSession` 依赖 `ModelRuntime`；`ModelRuntime` 实现 `Models`；`Models` 依赖 `Provider`。**越往下越不知道业务，越往上越不知道循环。**

## 6.2 `Provider`：认证是 provider 自己的事

这是 1.3 最重要的所有权变化。旧版把 API Key 从环境变量或注册表里「注入」给 provider；新版要求每个 provider **自带认证语义**：

```ts
export interface Provider<TApi extends Api = Api> {
	readonly id: string;
	readonly name: string;
	readonly baseUrl?: string;
	readonly headers?: ProviderHeaders;
	readonly auth: ProviderAuth;

	getModels(): readonly Model<TApi>[];
	getAllModels?(): readonly ProviderModel<TApi>[];
	refreshModels?(context: RefreshModelsContext): Promise<void>;
	filterModels?(models: readonly Model<TApi>[], credential: Credential | undefined): readonly Model<TApi>[];

	stream<T extends TApi>(model: Model<T>, context: TranscriptContext, options?): AssistantMessageEventStream;
	streamSimple(model: Model<TApi>, context: TranscriptContext, options?): AssistantMessageEventStream;
	// generateImages / classify / deferred ...
}
```

几个刻意的约束：

- **`auth` 是必填的**。即使是只读环境变量的 provider、AWS profile、ADC 文件，甚至本地免密钥服务，也必须提供 `apiKey` 形式的认证描述，其 `resolve()` 报告「是否已配置」。这样 `Models.getAuth()` 永远有统一答案，不需要到处写特判。
- **`getModels()` 是同步的**。静态 provider 返回内置目录；动态 provider 返回「上次刷新的结果」，刷新前为空。不能抛异常——抛了就被当作没有模型。
- **`refreshModels()` 是动态 provider 的增量覆盖**。它负责恢复缓存、在凭据有效时拉新列表；失败必须保留旧列表。
- **`filterModels()` 允许凭据相关的可用性策略**。例如某些模型只对特定订阅开放。

这带来的直接好处是：**认证不再是一个旁路的 `withEnvApiKey` 工具函数**，而是 provider 契约的一部分。扩展注册自定义 provider 时，也必须想清楚「怎么登录、怎么刷新、哪些模型可用」。

## 6.3 `Models`：集合、认证与归一化

`createModels()`（`models.ts:1012`）返回一个 `MutableModels`：

```ts
const models = createModels();
models.setProvider(openaiProvider());
models.setProvider(anthropicProvider());
```

`Models` 的职责可以分成四块：

### 6.3.1 目录读取

```ts
getModel(provider, id)
getModels(provider?)
getModelsOfType(type, provider?)
getAllModels(provider?)
```

未限定类型的方法只返回 chat 模型；`*OfType` 返回指定类型（image / classifier）；`getAll*` 返回全部。动态 provider 的列表类型是 `Model<Api>`，需要用 `hasApi()` 收窄。

### 6.3.2 可用性与认证

```ts
checkAuth(providerId)
getAvailable(providerId?)
getAuth(providerId | model)
login(providerId, type, interaction)
logout(providerId)
refresh({ providers? })
```

`getAvailable()` 会先确认 provider 认证完整，再应用 `filterModels`。UI 的模型选择器就只列这些。

`getAuth()` 可以按 provider 或按 model 解析。按 model 时会把模型自带的 headers 合并进去。失败会抛 `ModelsError`：

- `code: "oauth"`：token 刷新失败，已存凭据保留，重新登录可修复；
- `code: "auth"`：API Key 解析或凭据存储失败。

请求路径会把这些拒绝转成流错误，而不是让进程崩溃。

### 6.3.3 请求归一化

`Models.stream()` 在派发给 provider 之前做两件事：

```ts
const transcript = normalizeContext(context);
// ...
const { requestModel, requestOptions } = await this.applyAuth(model, options);
return provider.stream(requestModel, transcript, requestOptions);
```

- `normalizeContext()` 把调用方的 `Context` 转成 `TranscriptContext`。这是**唯一**能构造该类型的入口，provider 代码拿不到未归一化的上下文。
- `applyAuth()` 解析认证、合并 baseUrl 与 headers，并把 `transformHeaders` 应用到最终请求头。

`ModelsRequestTransforms` 只提供一个 `transformHeaders`，但它的位置很关键：**在模型、认证、provider 默认头全部合并完之后**。这让代理、审计、灰度等基础设施可以在最后一刻改 header，而不用改 provider。

### 6.3.4 便捷方法

`complete()` / `completeSimple()` / `streamDeferred()` / `fetchDeferred()` / `cancelDeferred()` / `generateImages()` / `classify()` 都是薄封装。几个契约细节：

- `generateImages` / `classify` **永不 reject**：未知 provider、未配置认证、provider 不支持，都会返回一个错误结果对象。
- `stream` / `streamSimple` 返回 `AssistantMessageEventStream`；请求准备阶段的失败通过 `lazyStream` 延迟成流错误。
- `complete` 是 `stream(...).result()` 的语法糖，但错误仍会被编码进最终消息。

## 6.4 `createProvider()`：内置 provider 与自定义 provider 共用一条路

`packages/ai/src/providers/*.ts` 里每个内置 provider 都通过 `createProvider()` 构造：

```ts
export function createProvider<TApi extends Api>(input: CreateProviderOptions<TApi>): Provider<TApi> {
	// 合并静态模型 + 动态覆盖
	// 按 model.api 选择实现
	// 组装 auth / getModels / refreshModels / filterModels / stream ...
}
```

`CreateProviderOptions` 的关键字段：

| 字段 | 作用 |
|---|---|
| `models` | 静态基线目录（可含 chat / image / classifier） |
| `fetchModels` | 动态目录覆盖，事务性发布 |
| `api` | 单个流实现，或按 `model.api` 分派的映射 |
| `images` / `classifiers` | 对应能力的实现映射 |
| `filterModels` / `filterAllModels` | 凭据相关的可用性过滤 |

`createProvider` 会校验「至少有一个实现」，空映射直接报错。一个 provider 可以同时支持多种 API：例如同一个网关既有 `openai-completions` 又有 `anthropic-messages` 模型，就传一个按 `model.api` 分派的 map。**模型找不到对应实现时，返回一个流错误，而不是静默失败。**

目前 `providers/all.ts` 汇总了 **42 个** provider 工厂。它们共享 `createProvider` 的行为，但各自的认证、目录、请求方言独立。

## 6.5 模型目录：生成、缓存、叠加

pi 的模型目录不是手写的一张常量表，而是分层构建：

1. **生成阶段**：`packages/ai/scripts/generate-models.ts` 从模型数据源生成 `models.generated.ts`；
2. **目录层**：`model-catalog.ts` 把内置目录与远端目录合并；
3. **运行时层**：`ModelsStore` 持久化动态 provider 的刷新结果；
4. **产品层**：`coding-agent` 的 `models-store.ts` 在启动时恢复缓存，并在后台尝试从 pi.dev 叠加新数据。

根目录的 `npm run check:model-data`、`check:model-catalog`、`check:catalog-compat` 是这套机制的护栏：模型数据或 catalog 结构变化会导致 CI 失败，避免「代码能跑但模型列表悄悄缺项」。

> **给 Agent 开发者的启示**：模型目录不是配置，而是**需要版本化、可校验、可离线降级的数据产品**。pi 把它拆成生成、缓存、叠加三层，就是为了让「没有网络」和「provider 新增模型」都不至于让 Agent 瘫痪。

## 6.6 `ModelRuntime`：产品层为什么还要再包一层

`packages/coding-agent/src/core/model-runtime.ts` 里的 `ModelRuntime implements Models`，它不是重复造轮子，而是补三类产品需求：

1. **虚拟模型**：用户选择的 `provider/model` 可能是一个路由器。`resolveModel()` 根据请求原因（user / continuation / retry / direct）、思考级别和持久化状态，解析出真正的物理模型。
2. **用户自定义 provider**：从 `models.json` 读取 OpenAI/Anthropic 兼容端点，构造 provider 并注册进 `Models`。
3. **认证与目录的产品化**：`/login`、`/logout`、OAuth 刷新、模型选择器、启动时的目录刷新，都需要比 `Models` 更上层的状态。

因此你会看到 `AgentSession` 的认证调用是 `this._modelRuntime.getAuth(model)`，而不是直接操作 `Models`。**`Models` 是可复用的库，`ModelRuntime` 是可运行的产品。**

## 6.7 一次请求的认证与路由路径

把这一章串起来，一次模型请求的完整路径是：

```
AgentSession.prepareRequest
  └─ ModelRuntime.resolveModel（若是虚拟模型）
       └─ AgentSession._getRequiredRequestAuth
            └─ ModelRuntime.getAuth → Provider.auth.resolve
                 └─ Agent.streamFunction(model, TranscriptContext, options)
                      └─ Models.streamSimple
                           ├─ normalizeContext
                           ├─ applyAuth（合并 baseUrl / headers）
                           ├─ transformHeaders
                           └─ Provider.streamSimple
                                └─ Provider.stream → 具体 API 实现
```

每一层都只做一件事，而且都能被替换：路由可以换、认证可以换、header 变换可以换、provider 实现可以换。**这就是「模型可插拔」在工程上的真实含义。**

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/ai/src/models.ts` | `Provider` / `Models` / `createModels` / `createProvider` |
| `packages/ai/src/providers/all.ts` | 42 个内置 provider 工厂的汇总 |
| `packages/ai/src/model-catalog.ts` | 目录合并与查询 |
| `packages/ai/src/models.generated.ts` | 生成的内置模型数据 |
| `packages/ai/src/models-store.ts` | 动态目录持久化 |
| `packages/coding-agent/src/core/model-runtime.ts` | 产品层 `Models` 实现、虚拟模型 |
| `packages/coding-agent/src/core/virtual-models.ts` | 虚拟模型定义与状态 |
| `packages/coding-agent/docs/models.md` | 面向用户的模型配置说明 |

## 动手实验

1. 用 `createModels()` + `createProvider()` 手写一个最小 provider：`getModels()` 返回一个假模型，`streamSimple()` 返回一段固定文本。把它接到 `Agent` 上跑通一轮，不联网。
2. 给这个 provider 加一个只允许特定 `credential.plan` 的 `filterModels`，调用 `getAvailable()` 验证过滤结果。
3. 阅读 `ModelsImpl.stream()` 与 `applyAuth()`，画出「model headers、provider headers、auth headers、transformHeaders」的合并顺序。思考：为什么 `transformHeaders` 要放在最后？
4. 在 `providers/all.ts` 里数一遍 provider 工厂，并找出至少两个「同一 provider 支持多种 api」的例子。
