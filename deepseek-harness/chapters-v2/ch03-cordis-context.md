# 第 3 章 Cordis 核心：Context、Service 和依赖注入

上一章 `composeProfile` 组装好了补丁列表，`runProfile` 将它传给了 `boot()`。现在请求到达了 `packages/boot/app-boot/src/index.ts` 的 `boot()` 函数——这里将创建 Cordis 的根 Context 并启动整个 Loader。但在深入 `boot()` 之前，我们需要先理解 Cordis 的核心概念：Context 是什么，Service 怎么工作，依赖注入如何运转。

## 请求到达这里时发生了什么

`boot()` 函数在 `packages/boot/app-boot/src/index.ts` 第 215 行左右，签名如下：

```typescript
export async function boot(
  binName: string,
  absoluteConfigPath: string,
  patches?: PatchOptions[],
  prepare?: (ctx: Context) => Promise<void> | void,
  bareModuleBaseUrl?: string,
): Promise<Context>
```

它的第一步极其简单：

```typescript
const ctx = new Context()
```

这个 `new Context()` 在 `vendor/cordis/src/context.ts` 中定义，让我们看看它的构造函数做了什么。

## 关键代码路径

### Context：一个 Proxy 包装的依赖容器

```typescript
constructor() {
  this[symbols.isolate] = Object.create(null)
  this[symbols.intercept] = Object.create(null)
  const self = new Proxy<this>(this, ReflectService.handler)
  this.root = self
  this.baseUrl = undefined
  this.fiber = new Fiber(self, {}, Object.create(null), null, () => [])
  this.reflect = new ReflectService(self)
  this.registry = new RegistryService(self)
  this.events = new EventsService(self)
  this.logger = new LoggerService(self)
  this.fiber._disposables.clear()
  return self
}
```

关键点：**构造函数返回的是一个 Proxy**，不是原始对象。所有属性读取都经过 `ReflectService.handler` 拦截——当你写 `ctx.sessions` 时，Proxy 会通过依赖注入系统解析 `'sessions'` 这个名字，找到对应的 Service 实例。

根 Context 的 `fiber` 是一个特殊的 Fiber——`runtime` 为 `null`，state 直接是 `FiberState.ACTIVE`。它不对应任何插件，是整棵树的根节点。

### Service：在 Context 上注册能力

`Service` 基类在 `vendor/cordis/src/service.ts` 中。每个 Service 子类通过构造函数注册自己：

```typescript
constructor(protected ctx: Context, name: string) {
  name ??= this.constructor['provide'] as string
  let self = this
  const tracker: Tracker = { associate: name, property: 'ctx' }
  if (self[symbols.invoke]) {
    self = createCallable(name, joinPrototype(...), tracker)
  }
  self.ctx = ctx
  self.name = name
  self.ctx.reflect.provide(name, self, this[symbols.check])
  return self
}
```

`ctx.reflect.provide(name, self, check)` 将这个 Service 实例注册到 Context 的反射层。之后任何子 Context 读取 `ctx[name]` 都能拿到这个实例。当 Service 所在的 Fiber 卸载时，它会自动注销。

### Fiber：插件的运行时实例

`Fiber` 在 `vendor/cordis/src/fiber.ts` 中，是每个插件的运行时容器。它的生命周期状态机：

```
PENDING → LOADING → ACTIVE → UNLOADING → DISPOSED
                   ↘ FAILED ↗
```

`PENDING` 等待依赖的 Service 就绪；`LOADING` 是插件回调正在运行；`ACTIVE` 表示正常服务中。当所有依赖的 Service 都可用时，Fiber 自动从 PENDING 转到 LOADING。

Fiber 的 `_reload` 方法执行实际的插件启动：

```typescript
private async _reload() {
  this.store = { ...this._store }
  const oldEpoch = this._runner.epoch
  try {
    await Promise.resolve()
    if (this._runner.epoch === oldEpoch) {
      this.config = this._resolveConfig(this._config)
      await this._execute(this._runner)
      this._error = undefined
    }
  } catch (reason) {
    this.ctx.logger.error(reason)
    this._error = reason
    this._runner.epoch = INACTIVE
  }
  // ...
}
```

`_execute` 调用插件的回调函数。如果回调是构造函数（`isConstructor`），就 `new` 它；否则直接调用。返回值可以是一个 disposer 函数、一个 Promise、或一个迭代器——这些都是"效果"（effect）的合法形态。

### EventsService：五种分发模式

Cordis 的事件总线在 `vendor/cordis/src/events.ts` 中，支持五种分发模式：

- **emit**：同步触发，不等待返回值
- **parallel**：并发等待所有监听器
- **serial**：按序等待，直到某个返回 bail 值
- **bail**：同步运行，第一个返回非 null/false/undefined 的值中断
- **waterfall**：每个监听器包装下一个，形成调用链

这五种模式在后续的 Agent Loop、Session、Tool 执行中各有用途。

## 调用链：从这里到下一步

回到 `boot()` 函数，Context 创建后：

```typescript
ctx.baseUrl = pathToFileURL(dirname(absoluteConfigPath)).href + '/'
ctx.provide('dshHomePath', dshHomePath)
await ctx.plugin(Loader)
await prepare?.(ctx)
await mountRootInclude(ctx, absoluteConfigPath, patches, bareModuleBaseUrl)
await ctx.get('loader')?.await()
```

`ctx.plugin(Loader)` 将 Loader 安装为根 Context 的插件。`mountRootInclude` 创建一个 `cordis:include` 条目，指向 profile 的根配置文件——这就是整棵插件树的入口。

**下一步：请求将流向 `mountRootInclude`——Loader 如何从一个 YAML 文件长出一棵完整的插件树。**

## 关键设计决策

1. **Context 是 Proxy**：属性读取走反射层，这让依赖注入对使用者完全透明——`ctx.sessions` 看起来像普通属性访问，实际是运行时解析。

2. **Service 自动生命周期管理**：Service 注册到 Fiber，Fiber 卸载时自动注销。不需要手动清理。

3. **Fiber 的 epoch 机制**：每次依赖变化都生成新的 epoch 字符串。如果 epoch 在 await 期间变了，说明状态已经过期，当前操作应该中止。这避免了复杂的竞态条件。

4. **事件分发模式的区分**：不同场景需要不同的并发语义。`session/event` 用 emit（fire-and-forget），`session/flush` 用 parallel（等待全部完成），`agent/request` 用 waterfall（可拦截修改）。

## 本章小结

- `Context` 是 Proxy 包装的依赖容器，属性读取走反射层
- `Service` 通过构造函数自动注册到 Context，随 Fiber 卸载自动注销
- `Fiber` 是插件的运行时实例，有完整的生命周期状态机
- 事件总线支持五种分发模式：emit、parallel、serial、bail、waterfall
- 根 Context 的 Fiber 是特殊的——runtime 为 null，直接 ACTIVE

## 阅读思考题

1. Context 的 Proxy 机制和 JavaScript 的 `with` 语句有什么相似和不同之处？
2. 为什么 Fiber 的 `_reload` 方法要在 `await Promise.resolve()` 之后再检查 epoch？如果去掉这个 checkpoint 会怎样？
3. `waterfall` 模式和 `serial` 模式的核心区别是什么？为什么 `agent/request` 用 waterfall 而不是 serial？

> **下一章预告**：Context 创建了，Loader 也安装了。现在 `mountRootInclude` 要把 profile 的根配置文件挂载为 Loader 的入口。Loader 怎么从一个 YAML 文件解析出插件列表，又怎么逐个加载它们？
