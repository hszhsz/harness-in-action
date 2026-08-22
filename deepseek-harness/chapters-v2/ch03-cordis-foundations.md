# 第 3 章　Cordis：vendored 插件框架的五块基石

dsh 没有自己的"框架层"——它直接把 Cordis 的源码 vendoring 进了 `vendor/cordis/`。要读懂 dsh，先读懂这五块基石：插件、上下文、依赖注入、类型化事件、可逆 effect。这一章的源码全部来自 `vendor/cordis/src/` 下的六个文件：`context.ts`、`service.ts`、`events.ts`、`fiber.ts`、`registry.ts`、`reflect.ts`。

## 插件（Plugin）与上下文（Context）

Cordis 的第一概念：**插件是一个实现了 Service 的对象**。`vendor/cordis/src/service.ts` 里的 `Service` 抽象类是所有 Cordis 服务的基类：

```ts
export abstract class Service<out T = never> {
  static readonly init: unique symbol = symbols.init
  static readonly check: unique symbol = symbols.check
  static readonly config: unique symbol = symbols.config
  static readonly invoke: unique symbol = symbols.invoke
  static readonly extend: unique symbol = symbols.extend
  static readonly tracker: unique symbol = symbols.tracker

  constructor(protected ctx: Context, name: string) {
    name ??= this.constructor['provide'] as string
    // ...
    self.ctx.reflect.provide(name, self, this[symbols.check])
    return self
  }
}
```

构造函数的关键一行：`self.ctx.reflect.provide(name, self, ...)` —— 服务在构造时就把自己注册到上下文中。子类调用 `super(ctx, 'myServiceName')`，Cordis 会把它的生命周期挂载进当前上下文。

第二概念：**上下文（Context）是服务的仓库**。`vendor/cordis/src/context.ts` 里的 `Context` 类是一个 Proxy——普通属性读取经过服务解析器，`extend()`、`isolate()`、`intercept()` 创建子上下文而不修改父上下文：

```ts
export class Context {
  static readonly effect: unique symbol = symbols.effect
  static readonly filter: unique symbol = symbols.filter
  static readonly isolate: unique symbol = symbols.isolate
  static readonly intercept: unique symbol = symbols.intercept

  constructor() {
    this[symbols.isolate] = Object.create(null)
    this[symbols.intercept] = Object.create(null)
    const self = new Proxy<this>(this, ReflectService.handler)
    this.root = self
    this.fiber = new Fiber(self, {}, Object.create(null), null, () => [])
    // ...
    return self
  }
}
```

构造函数返回 `new Proxy(this, ...)` —— 这意味着 `ctx.xxx` 的读取会经过 `ReflectService.handler` 的拦截逻辑，实现服务的懒解析和作用域隔离。一个服务从上下文认领一个稳定的 `ctx.<key>`（比如 `ctx.tools`、`ctx.llm`、`ctx.sessions`），其他插件通过 key 找服务，而不是 import 具体实现。

在 dsh 里，这些 `ctx.<key>` 就是整个产品的 API 表面。`docs/architecture.md` 给了核心表：

| Package | Owns | `ctx` key |
|---|---|---|
| `core/session` | append-only `SessionEvent` 日志与内存 store | `ctx.sessions` |
| `core/system-prompt` | prompt-section 与 tool-schema 组装 | `ctx.systemPrompt` |
| `core/tools` | 作用域工具注册表与守卫执行管线 | `ctx.tools` |
| `core/agent` | `Agent` 接口、live registry、`agent/*` 事件 | `ctx.agents` |
| `core/agent-loop` | 实现该接口的默认驱动 | `ctx.agentLoop` |
| `core/scope` | per-agent scoped-registration 原语 | library，无 key |
| `llm/llm` | 消息与流词汇 + adapter seam | `ctx.llm` |

## 依赖注入：`inject` 声明取代手动启动排序

Cordis 的第三个概念：**通过 `inject` 声明服务依赖**。一个插件如果 `inject` 了所需服务，Cordis 会等这些服务存在后才激活它——**加载顺序由服务需求表达，而不是手动编排启动顺序**。

这在 dsh 的 bundle 配置里随处可见。看 `packages/bundle/headless/cordis.patch.yml`：

```yaml
- id: headless-runner
  name: '@deepseek-ai/dsh-headless'
  inject: [headlessStartup]
  config:
    task: !!js ctx.headlessStartup.task
```

`headless-runner` 声明 `inject: [headlessStartup]`，它的 `config.task` 从 `ctx.headlessStartup.task` 取——`headless-startup` 插件必须先激活，`headless-runner` 才会被激活。这就是"以服务需求表达加载顺序"的活例子。

更妙的是 `!!js` 表达式的延迟求值：配置表达式 `!!js ctx.webStartup.xxx` 只有在服务存在后才会被 Loader 解析——如果该服务不存在（比如 `dsh --profile web --help` 不提供该服务），表达式不会被求值，服务器不会绑定端口。**同一个配置文件，根据注入的服务存在与否，决定激活哪些行为。**

## 类型化事件与四种派发模式

Cordis 的第四个概念：**类型化事件**。`vendor/cordis/src/events.ts` 定义了五种派发模式：

```ts
export type DispatchMode = 'emit' | 'parallel' | 'serial' | 'bail' | 'waterfall'
```

| 模式 | 是否 await | 派发顺序 | 有返回值？ |
|---|---|---|---|
| `emit` | 否 | 监听者按注册顺序观察 | 无 |
| `waterfall` | 否 | 监听者按注册顺序观察 | 有 |
| `parallel` | 是 | 所有监听者并行观察 | 无 |
| `serial` | 是 | 监听者按注册顺序观察 | 有 |
| `bail` | 否 | 监听者按注册顺序，第一个 bail 值停止 | 有 |

`events.ts` 里每种模式都有对应的 `ctx` 方法声明。关键的是 **waterfall**——它是 dsh 拦截体系的核心原语：

```ts
/**
 * Dispatch an event whose last argument is a `next` continuation.
 *
 * Each listener wraps the rest of the chain: calling `next()` invokes the
 * next listener (finally the built-in behavior); not calling it vetoes.
 */
waterfall<K extends keyof Events>(name: K, ...args: Parameters<Events[K]>): ReturnType<Events[K]>
```

waterfall 的语义是 **around-middleware**：监听者接收 `(...args, next)`，调用 `next()` 把控制权传给链上的下一个监听者；不调用就是短路。在 dsh 里，`agent/pre-step`、`agent/request`、`llm/stream`、`tools/pre-execute`、`tools/execute`、`tools/post-execute` 全部是 waterfall。`AGENTS.md` 的铁律：

> **Waterfall listeners MUST call `next()`** to delegate; returning without it short-circuits the chain.

## 可逆 effect：注册即副作用，卸载即回卷

Cordis 的第五个概念：**注册是可逆的 effect**。`vendor/cordis/src/fiber.ts` 定义了 `Fiber`（插件运行时实例）和 effect 机制：

```ts
export type Disposable<T = any> = () => T

export type Effect<T = any> =
  | SyncEffect<T>
  | AsyncEffect<T>

type SyncEffect<T = any> =
  | Disposable<T>
  | Iterable<Disposable<T>, void, void>

type AsyncEffect<T = any> =
  | Promise<Disposable<T>>
  | AsyncIterable<Disposable<T>, void, void>
```

`ctx.effect()` 接受一个 effect body，body 返回 disposer（或多个 disposer 的可迭代）。当拥有该 effect 的 fiber 卸载时，所有 disposer 按注册逆序运行——**这就是"注册即 effect，卸载即回卷"的完整机制**。

dsh 的 `AGENTS.md` 把它上升为红线：

> **Registrations are effects**: every contribution goes through `ctx.effect()` / `ctx.on()`; a registry's `register()` returns the disposer.

这个机制支撑了 dsh 最激进的能力：**插件可以在运行时被安装、替换、卸载，而系统状态保持一致**。`packages/extensions/` 甚至让模型自己通过 `cordis_define` / `cordis_run` / `cordis_stop` 工具动态定义和运行 Cordis 包——"agent inspects/mounts its own plugins"（自修改）。这在其他 Agent 里是禁区，在 dsh 里是设计。

## 插件生命周期：mount → inject → activate → dispose

一个 Cordis 插件的生命周期：**装载（mount）→ 注入等待（inject 的服务全部就绪）→ 激活（apply）→ 运行 → 卸载（dispose）**。`packages/boot/app-boot/src/index.ts` 的 `boot()` 函数在启动失败时利用这一点做了**部分上下文回卷**：

```ts
} catch (cause) {
  await ctx.fiber.dispose()
  const detail = cause instanceof Error ? cause.message : String(cause)
  // ...
}
```

启动任何一步失败，`boot()` 都会 `ctx.fiber.dispose()` 回卷整个插件树——包括已经挂载的 surface 的终端恢复（raw mode、bracketed paste、键盘协议）。**失败路径和成功路径共享同一套清理逻辑**，这是"注册即 effect"在工程上的直接收益。

## 架构速览

```
Cordis 五块基石
┌──────────────┐
│   Plugin     │  实现 Service 的对象
├──────────────┤
│   Context    │  Proxy + 服务仓库 + 子上下文继承
├──────────────┤
│   Inject     │  声明依赖 → 按需激活，不手动排序
├──────────────┤
│   Events     │  emit/waterfall/parallel/serial/bail
├──────────────┤
│   Effect     │  ctx.effect() → disposer → 逆序回卷
└──────────────┘
     ↓
  Fiber (插件运行时实例) 管理生命周期
```

## 关键设计决策

1. **Context 是 Proxy**：`new Context()` 返回 `new Proxy(this, ReflectService.handler)`。这让 `ctx.xxx` 的读取可以经过服务解析器，实现懒解析和作用域隔离。代价是每次属性访问都有额外开销，收益是插件不需要手动管理服务查找。

2. **waterfall 是 around-middleware**：不是简单的"通知"，而是"拦截+委托"。监听者可以改写参数、短路链、包裹执行。这正是 dsh 的 `agent/pre-step`、`tools/pre-execute` 等拦截点的基础设施。

3. **Fiber 管理 effect 的生命周期**：每个插件有自己的 Fiber，Fiber 持有该插件注册的所有 disposer。卸载时按逆序回卷——这保证了"最后注册的最先清理"，避免了资源泄漏和依赖倒置。

4. **`!!js` 表达式的延迟求值**：配置文件里的 `!!js ctx.xxx.yyy` 只在服务存在后才被求值。这让同一个配置文件可以根据环境动态决定激活哪些行为，而不需要条件分支。

## 本章小结

- **插件 = 实现 Service 的对象**，Context 是 Proxy + 服务仓库，`ctx.<key>` 是产品的 API 表面。
- **`inject` 声明依赖**，加载顺序由服务需求表达，不手动编排。
- **类型化事件**五种派发模式，waterfall 是 around-middleware——dsh 的所有拦截点都建立在 waterfall 上。
- **注册是可逆 effect**，`ctx.effect()` 返回 disposer，Fiber 卸载时按逆序回卷。
- **生命周期**（mount → activate → dispose）由框架管理，失败路径共享清理逻辑。

## 阅读思考题

1. `vendor/cordis/src/context.ts` 的构造函数里 `return self` 返回的是 Proxy 而不是 `this`。这意味着 `new Context()` 的调用者拿到的是 Proxy。如果某个插件绕过 Proxy 直接访问 `this` 上的属性，会发生什么？
2. waterfall 的"短路"语义意味着不调用 `next()` 就能阻止后续监听者运行。在 dsh 的 `agent/pre-step` 里，谁是"最后一个必须调 `next()` 的监听者"？如果没有人调 `next()`，主循环会怎样？
3. `Fiber.dispose()` 按注册逆序回卷 disposer。如果一个 disposer 是异步的（比如需要 flush 数据），Fiber 会等它完成吗？这对 dsh 的崩溃恢复有什么影响？

> **下一章预告**：第 4 章将拆开 dsh 的启动组装线——从 `apps/cli/src/bin.ts` 的三种模式分发，到 `app-boot` 的 `boot()` 函数，再到 profile / bundle / patch 三层组合。
