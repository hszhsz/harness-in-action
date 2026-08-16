# 第 3 章　Cordis：vendored 插件框架的五块基石

> dsh 没有自己的"框架层"——它直接把 Cordis 的源码 vendoring 进了 `vendor/`。要读懂 dsh，先读懂这五块基石：插件、上下文、依赖注入、类型化事件、可逆 effect。

## 3.1　插件（Plugin）与上下文（Context）

Cordis 的第一概念：**插件是一个实现了 Service 的对象**。它可以是一个带可选 `inject` 和 `apply(ctx)` 字段的函数，也可以是一个 `Service` 子类——Cordis 会把它的生命周期挂载进当前上下文。[[docs/cordis-primer.md]](https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/cordis-primer.md)

第二概念：**上下文（Context）是服务的仓库**。一个服务从上下文认领一个稳定的 `ctx.<key>`（比如 `ctx.tools`、`ctx.llm`、`ctx.sessions`），其他插件通过 key 找服务，而不是 import 具体实现。这彻底解耦了插件之间的编译期依赖——两个插件只需要在"服务契约"层面达成一致，不需要在代码层面互相认识。

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

注意 `core/scope` 是唯一不是服务的包——它是零依赖库（`createScope`/`scopeOf`/`scopeTarget`），坐在模块图最底层，`session/` 和 `system-prompt/` 都要消费它但不能反向依赖它。

## 3.2　依赖注入：`inject` 声明取代手动启动排序

Cordis 的第三个概念：**通过 `inject` 声明服务依赖**。一个插件如果 `inject` 了所需服务，Cordis 会等这些服务存在后才激活它——**加载顺序由服务需求表达，而不是手动编排启动顺序**。

这在 dsh 里随处可见。看 `packages/bundle/headless/cordis.patch.yml`：

```yaml
- id: headless-runner
  name: '@deepseek-ai/dsh-headless'
  inject: [headlessStartup]
  config:
    task: !!js ctx.headlessStartup.task
```

`headless-runner` 声明 `inject: [headlessStartup]`，它的 `config.task` 从 `ctx.headlessStartup.task` 取——`headless-startup` 插件（解析命令行任务）必须先激活，`headless-runner` 才会被激活。这就是"以服务需求表达加载顺序"的活例子。

更妙的是 `web-app` bundle 里的用法：

```yaml
- id: web-startup
  name: '@deepseek-ai/dsh-web-startup'
```

配置表达式 `!!js ctx.webStartup.xxx` 只有在服务存在后才会被 Loader 解析——`dsh --profile web --help` 不提供该服务，所以服务器不会绑定端口。**同一个配置文件，根据注入的服务存在与否，决定激活哪些行为。**

## 3.3　类型化事件与四种派发模式

Cordis 的第四个概念：**类型化事件**。服务通过 TypeScript 声明合并（declaration merging）声明事件名，然后用 `emit`、`waterfall`、`parallel`、`serial` 四种方式派发：

| 模式 | 是否 await | 派发顺序 | 有返回值？ |
|---|---|---|---|
| `emit` | 否 | 监听者按注册顺序观察 | 无 |
| `waterfall` | 否 | 监听者按注册顺序观察 | 有 |
| `parallel` | 是 | 所有监听者并行观察 | 无 |
| `serial` | 是 | 监听者按注册顺序观察 | 有 |

派发模式是事件的**公开契约**：dsh 要求新事件用 `@mode` JSDoc 标注，生成的 catalog 会检查声明与派发点是否一致。

**waterfall 是"around 中间件"**，这是 dsh 拦截体系的核心原语：

> `ctx.waterfall` is around-middleware. A listener receives `(...args, next)`. Call `next()` to delegate the possibly wrapped result to the next service; return without `next()` to short-circuit.

dsh 的 `AGENTS.md` 有一条铁律与之配套：

> **Waterfall listeners MUST call `next()`** to delegate; returning without it short-circuits the chain.

在 dsh 里，`agent/pre-step`、`agent/request`、`llm/stream`、`tools/pre-execute`、`tools/execute`、`tools/post-execute` 全部是 waterfall——监听者可以改写请求、拒绝请求、包裹执行，但必须调用 `next()` 把控制权传给链上的下一个监听者；不调用就是短路（短路对"决策型"监听者是设计，对"观察型"监听者是 bug）。而 `agent/turn-stopping` 是 serial——按序运行，没有 `next()`。

## 3.4　可逆 effect：注册即副作用，卸载即回卷

Cordis 的第五个概念：**注册是可逆的 effect**。

> Registrations are reversible effects. Prompt sections, tool schemas, adapters, providers, and listeners are installed through `ctx.effect()` or `ctx.on()` so reload and teardown unwind them predictably.

dsh 的 `AGENTS.md` 把它上升为红线：

> **Registrations are effects**: every contribution goes through `ctx.effect()` / `ctx.on()`; a registry's `register()` returns the disposer.

这个机制支撑了 dsh 最激进的能力：**插件可以在运行时被安装、替换、卸载，而系统状态保持一致**。`extensions/` 组（`cordis-host-runner`、`tool-cordis`）甚至让模型自己通过 `cordis_define` / `cordis_run` / `cordis_stop` 工具动态定义和运行 Cordis 包——"agent inspects/mounts its own plugins"（自修改）。这在其他 Agent 里是禁区，在 dsh 里是设计。

## 3.5　`ctx.effect()` / `ctx.on()` 与插件生命周期

一个 Cordis 插件的生命周期大致是：**装载（mount）→ 注入等待（inject 的服务全部就绪）→ 激活（apply）→ 运行 → 卸载（dispose）**。在卸载时，所有通过 `ctx.effect()` 注册的贡献按注册逆序回卷，所有 `ctx.on()` 注册的监听器被移除。

dsh 的 `app-boot` 在启动失败时利用这一点做了**部分上下文回卷**（partial-context disposal）：

```ts
} catch (cause) {
  // Root-fiber disposal contains cleanup failures per observer (Cordis
  // fiber.ts hardening) and a repeated call returns the settled single-shot
  // result, so this await cannot reject and replace `cause`.
  await ctx.fiber.dispose()
  const detail = cause instanceof Error ? cause.message : String(cause)
  ...
}
```

启动任何一步失败，`boot()` 都会 `ctx.fiber.dispose()` 回卷整个插件树——包括已经挂载的 surface 的终端恢复（raw mode、bracketed paste、键盘协议）。这就是"注册即 effect"在工程上的直接收益：**失败路径和成功路径共享同一套清理逻辑**。

## 本章小结

Cordis 的五块基石构成了 dsh 的全部语法：

1. **插件 = 实现 Service 的对象**，上下文 = 服务仓库；
2. **`inject` 声明依赖**，加载顺序由服务需求表达；
3. **类型化事件**四种派发模式，waterfall 是 around 中间件；
4. **注册是可逆 effect**，卸载自动回卷；
5. **生命周期**（mount → activate → dispose）由框架管理，失败路径共享清理逻辑。

理解了这五块基石，第 4 章的组合机制（profile/bundle/patch）和第 5-8 章的主循环与工具管线就都是"在 Cordis 语法上写业务"了。

## 动手实验

1. 读 `docs/cordis-primer.md` 全文（很短），对照 3.2 节的 headless 例子理解 `inject`。
2. 在 dsh 源码里搜 `ctx.waterfall`，找出至少 5 个 waterfall 事件，确认每个监听者都调用了 `next()`。
3. 搜 `ctx.effect(`，看一个注册函数的返回值——确认它返回 disposer。
4. 在 `packages/core/scope/src/store.ts` 里看 `ScopedLayers` 如何实现"全局层 + 按需 scope 层"的合并——这是第 9 章 scope 机制的地基。
