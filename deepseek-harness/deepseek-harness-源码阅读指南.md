# DeepSeek Harness 源码阅读指南

> 追踪一个请求穿越整个 dsh 的旅程——从终端输入到任务完成。

## 目录

- [第 1 章 从终端到入口：parseDshArgs 的解析艺术](#第-1-章-从终端到入口parsedshargs-的解析艺术)
- [第 2 章 Profile 的组装：composeProfile 的补丁层叠魔法](#第-2-章-profile-的组装composeprofile-的补丁层叠魔法)
- [第 3 章 Cordis 核心：Context、Service 和依赖注入](#第-3-章-cordis-核心contextservice-和依赖注入)
- [第 4 章 Loader 与 Include：从 YAML 到插件树](#第-4-章-loader-与-include从-yaml-到插件树)
- [第 5 章 Session：事件溯源的对话容器](#第-5-章-session事件溯源的对话容器)
- [第 6 章 ReactLoopAgent：Agent Loop 的状态机](#第-6-章-reactloopagentagent-loop-的状态机)
- [第 7 章 工具系统：ToolDefinition 与六段守卫管线](#第-7-章-工具系统tooldefinition-与六段守卫管线)
- [第 8 章 文件系统能力：FsTarget 与 Capability Seam](#第-8-章-文件系统能力fstarget-与-capability-seam)
- [第 9 章 沙箱：操作系统级的 Agent 限制](#第-9-章-沙箱操作系统级的-agent-限制)
- [第 10 章 用户审批：fail-closed 的安全策略](#第-10-章-用户审批fail-closed-的安全策略)
- [第 11 章 事件持久化：JSONL 与 SQLite 的存储引擎](#第-11-章-事件持久化jsonl-与-sqlite-的存储引擎)
- [第 12 章 上下文压缩：当对话变长时](#第-12-章-上下文压缩当对话变长时)
- [第 13 章 Shell 执行：子进程管理与 I/O 流](#第-13-章-shell-执行子进程管理与-io-流)
- [第 14 章 Subagent：任务委派与多 Agent 协作](#第-14-章-subagent任务委派与多-agent-协作)
- [第 15 章 Workflow：脚本编排与代码运行时](#第-15-章-workflow脚本编排与代码运行时)
- [第 16 章 远程调用与 Web UI 可观测](#第-16-章-远程调用与-web-ui-可观测)
- [第 17 章 测试策略：覆盖完整请求路径](#第-17-章-测试策略覆盖完整请求路径)
- [第 18 章 完整调用链：从终端到任务完成](#第-18-章-完整调用链从终端到任务完成)

---

## 叙事主线

用户在终端输入：`dsh --profile headless "帮我写一个 hello.py"`

我们追踪这个请求穿过 dsh 的每一个模块：

```
终端输入 → bin.ts (parseDshArgs)
  → profile-boot.ts (runProfile → boot)
    → Cordis 插件树组合 (profile/bundle/patch)
      → Session 事件日志 (第一条 user/message 事件)
        → ReactLoopAgent (agent loop 接管, 创建 turn)
          → LLM adapter (组装 prompt, 调 DeepSeek API, 流式返回)
            → 模型决定调用 write_file 工具
              → ToolDefinition + 六段守卫管线
                → Capability Seam (FS provider)
                  → Sandbox (bwrap/Landlock/Seatbelt 限制文件写入)
                    → Approval (用户审批写入)
                      → 事件持久化 (JSONL/SQLite)
                        → 上下文压缩 (当对话变长)
                          → 模型继续调用 shell 执行 python hello.py
                            → Subagent (如果任务需要委派)
                              → Workflow (如果模型写脚本编排)
                                → 远程调用与 Web UI 可观测
                                  → 测试如何覆盖这条完整路径
                                    → 完整调用链回顾
```

---

# 第 1 章 从终端到入口：parseDshArgs 的解析艺术

想象一下，你坐在终端前，敲下了一行命令：

```
dsh --profile headless "帮我写一个 hello.py"
```

按下回车的那一刻，一个请求开始了它穿越整个 DeepSeek Harness 的旅程。我们的任务就是跟着这个请求，走完它经过的每一个模块。这一章，我们从旅程的起点——`bin.ts` 开始。

## 请求到达这里时发生了什么

当 Node.js 执行 `dsh` 时，它首先加载的是 `apps/cli/src/bin.ts`。这个文件的第一行是 `#!/usr/bin/env node` 声明，它是一个自执行的入口模块。

`bin.ts` 做的第一件事是从 `./args.ts` 导入 `parseDshArgs` 函数，同时从 `@deepseek-ai/dsh-app-boot` 导入 `loadLayeredEnv`。然后它调用 `readVersion()` 从同目录的 `package.json` 中读取版本号——注意这里用了 `fileURLToPath(new URL('../package.json', import.meta.url))` 来保证在源码树和构建产物中都能正确定位。

关键的调用只有一行：

```typescript
const invocation = parseDshArgs(process.argv.slice(2), readVersion())
```

`process.argv.slice(2)` 去掉了 Node 可执行文件路径和脚本路径，只留下用户传入的参数。对于我们的例子，这就是 `['--profile', 'headless', '帮我写一个 hello.py']`。

## 关键代码路径

`parseDshArgs` 定义在 `apps/cli/src/args.ts` 中，它使用 Commander.js 来解析命令行。这里有一个精妙的设计决策：

```typescript
program
  .helpOption(false)
  .allowUnknownOption()
  .passThroughOptions()
  .enablePositionalOptions()
  .argument('[args...]', 'arguments for the booted profile\'s app')
  .option('--profile <name>', 'the profile under $DSH_HOME/profiles to boot')
```

看到了吗？它显式禁用了 `helpOption`，设置了 `allowUnknownOption()` 和 `passThroughOptions()`。这意味着 Commander 只解析它自己认识的标志（`--profile`、`--patch`、`--dump-config`），**遇到第一个不认识的 token 就停止解析**，把后面的所有东西原封不动地传给内层参数。

对于 `dsh --profile headless "帮我写一个 hello.py"`，Commander 的 `.action()` 回调会收到 `args = ['帮我写一个 hello.py']` 和 `options = { profile: 'headless' }`。然后 `resolveBoot` 函数将它们组装成一个 `ProfileInvocation`：

```typescript
return { mode: 'profile', profile, patches, args }
```

这里 `patches` 来自 `--patch` 选项（我们的例子里没有），`args` 是 `['帮我写一个 hello.py']`。

## 调用链：从这里到下一步

`parseDshArgs` 返回 `DshInvocation` 后，`bin.ts` 的 `switch` 语句根据 `invocation.mode` 分发。对于 `'profile'` 模式：

```typescript
case 'profile': {
  const { runProfile } = await import('./profile-boot.ts')
  await runProfile({
    environment: loadLayeredEnv('dsh'),
    profile: invocation.profile,
    patchFiles: invocation.patches,
    args: invocation.args,
  })
  break
}
```

注意两个关键细节：**动态导入** `profile-boot.ts` 保持了无关模式的代码不互相污染；`loadLayeredEnv('dsh')` 在 `runProfile` 之前就冻结了环境快照。

**下一步：请求将流向 `profile-boot.ts` 的 `runProfile()` 函数。**

## 关键设计决策

1. **Commander 的 passThroughOptions 模式**：launcher 只管自己的标志，所有应用层参数（`--resume`、`-h`、对话文本）都透传给 booted profile。这让 `dsh --profile web --help` 打印的是 web 应用的帮助，而不是 launcher 的。

2. **动态导入每个模式**：`profile`、`plugin`、`dump-config` 三个模式的代码通过 `await import()` 按需加载。一个 `dsh plugin` 命令不需要加载整个 profile boot 链。

3. **环境快照在启动前冻结**：`loadLayeredEnv` 在任何插件加载之前就运行，确保整个生命周期使用同一份不可变的环境值。

4. **`web` 是硬编码别名**：`dsh web` 等价于 `dsh --profile web`，是 Commander 子命令层面的语法糖。

## 本章小结

- `bin.ts` 是 dsh 的入口，调用 `parseDshArgs` 解析命令行
- `parseDshArgs` 用 Commander.js 的 passThrough 模式：launcher 标志先解析，剩余参数原样传递
- 解析结果是一个 `DshInvocation` 联合类型（profile / dump-config / plugin）
- 对于 profile 模式，`bin.ts` 动态导入 `profile-boot.ts` 并调用 `runProfile()`
- 环境快照在任何插件加载前就已冻结

## 阅读思考题

1. 为什么 `parseDshArgs` 要禁用 Commander 的内置 `helpOption`？如果不禁用会怎样？
2. `loadLayeredEnv` 返回的 `LaunchEnvironmentSnapshot` 被传给 `runProfile` 但没有传给 `parseDshArgs`，这说明了什么关于职责划分的设计意图？
3. 如果用户输入 `dsh --profile headless --patch extra.yml "帮我写一个 hello.py"`，`parseDshArgs` 的返回值会是什么？

> **下一章预告**：`runProfile` 收到了 profile 名称、patch 文件路径和内层参数。接下来我们要看它如何把一个 profile 名变成一棵活着的插件树——`composeProfile` 的补丁层叠魔法。

---

# 第 2 章 Profile 的组装：composeProfile 的补丁层叠魔法

上一章我们追踪到 `parseDshArgs` 返回了一个 `ProfileInvocation`，`bin.ts` 随即调用了 `runProfile()`。现在请求到达了 `apps/cli/src/profile-boot.ts`——这是把一个 profile 名字变成一棵活着的 Cordis 插件树的地方。

## 请求到达这里时发生了什么

`runProfile` 接收的参数很干净：`environment`（冻结的环境快照）、`profile`（字符串 `'headless'`）、`patchFiles`（空数组）、`args`（`['帮我写一个 hello.py']`）。

它做的第一件事是调用 `composeProfile`：

```typescript
const composed = composeProfile(options.profile, options.patchFiles)
```

`composeProfile` 是本章的核心函数。它做了三件事：

1. 调用 `prepareProfile(name)` 加载 profile 并写入空的根配置
2. 读取 home 级补丁和 overlay 补丁
3. 把所有补丁层叠在一起，返回 `ComposedProfile` 对象

让我们深入每一步。

## 关键代码路径

### prepareProfile：加载 profile 目录

```typescript
export function prepareProfile(name: string, userLayer = true): Profile {
  healProfilesModuleFallback(INSTALL_ANCHOR)
  const profile = loadProfile(NAME, name, INSTALL_ANCHOR, undefined, { userLayer })
  writeFileSync(join(profile.dir, PROFILE_ROOT_FILENAME), PROFILE_ROOT_CONFIG)
  return profile
}
```

`healProfilesModuleFallback` 在 `packages/boot/app-boot/src/profile.ts` 中定义，它维护一个扁平的模块回退目录 `$DSH_HOME/profiles/node_modules`。通过 BFS 遍历 dsh 安装的依赖闭包，为每个包创建一个符号链接。这让任何 profile 都能通过 Node 的父目录遍历找到所有内置插件。

`loadProfile` 做了更关键的事：它读取 profile 目录下的 `package.json`，从中提取 `dsh.profile.bundles` 数组。对于 `headless` profile，默认的 bundles 是 `['@deepseek-ai/dsh-base', '@deepseek-ai/dsh-headless']`。

每个 bundle 都被解析为一个 `ProfileLayer`：

```typescript
const layers = bundles.map((packageName): ProfileLayer => {
  const packageDir = resolveBundleDir(binName, packageName, installAnchor, dir)
  const bundleManifest = JSON.parse(readFileSync(join(packageDir, 'package.json'), 'utf8')) as ProfileManifest
  const declared = bundleManifest.dsh?.bundle?.patch
  if (declared === undefined) {
    throw new Error(`dsh: profile bundle ${JSON.stringify(packageName)} declares no dsh.bundle in its package.json`)
  }
  const patchPath = join(packageDir, declared)
  return { packageName, packageDir, patchPath, patches: loadOverlayPatches(binName, patchPath) }
})
```

每个 bundle 必须在其 `package.json` 的 `dsh.bundle.patch` 字段声明它的补丁文件路径。`loadOverlayPatches` 读取并解析这个 YAML 补丁文件——它是一个顶层 YAML 数组，使用 `!!js` 表达式方言。

### 层叠顺序

回到 `composeProfile`，补丁按以下顺序叠放：

```typescript
const bundlePatches = profile.layers.flatMap(layer => layer.patches)
```

然后加上用户的 `cordis.patch.yml`、home 级补丁、`--patch` overlay、以及遥测开关补丁。最终的顺序是：

```
bundle 层（按 dsh.profile.bundles 顺序）
  → profile 的 cordis.patch.yml（用户层）
    → $DSH_HOME/cordis.patch.yml（机器级用户偏好）
      → --patch overlay 文件
        → 遥测开关补丁
```

后面层的补丁覆盖前面层的同 id 行。

### composeEntries：从空列表到完整树

`composeEntries` 函数把所有展平的补丁应用到一个空列表上：

```typescript
export function composeEntries(
  layers: readonly PatchOptions[][], warn: (message: string) => void = () => {},
): EntryOptions[] {
  return applyEntryPatches([], structuredClone(layers.flat()), ...)
}
```

`applyEntryPatches` 来自 `@deepseek-ai/cordis-plugin-include`，它是整个补丁系统的引擎。对空列表 `[]` 应用补丁，就是从零构建出整棵配置树。

## 调用链：从这里到下一步

`composeProfile` 返回后，`runProfile` 继续执行：

```typescript
const rootConfig = join(composed.profile.dir, PROFILE_ROOT_FILENAME)
const ctx = await boot(NAME, rootConfig, structuredClone(allPatches(composed)), (hostCtx) => {
  app.current = hostCtx
  hostCtx.provide(DSH_LAUNCH_ENVIRONMENT_KEY, options.environment)
  provideCmdline(hostCtx, { args: options.args, exit: code => void shutdown.shutdown(code) })
})
```

它把补丁列表传给 `boot()`，同时通过回调在 Loader 挂载任何配置树条目之前注入了两个值：`DSH_LAUNCH_ENVIRONMENT_KEY`（冻结的环境快照）和命令行参数。

**下一步：请求将流向 `app-boot/src/index.ts` 的 `boot()` 函数——Cordis Loader 的启动引擎。**

## 关键设计决策

1. **补丁层叠而非配置合并**：每一层都是一个 YAML 补丁列表，通过 id 定位目标行。这比直接合并 YAML 更精确——你可以禁用某行、替换某行的配置、或在特定位置插入新行。

2. **根配置文件总是重写**：`prepareProfile` 每次启动都重写 `cordis.yml` 为空列表。因为 Loader 的树回写机制可能把组合后的行写入这个文件，如果不重写，下次启动会重复插入 bundle 行。

3. **Profile 自动初始化**：如果 `$DSH_HOME/profiles/headless` 不存在，`loadProfile` 会用 `PROFILE_TEMPLATES` 中的默认 bundles 自动创建它。首次使用无需手动配置。

4. **环境快照在 Loader 挂载前注入**：`DSH_LAUNCH_ENVIRONMENT_KEY` 通过 `hostCtx.provide()` 在 `prepare` 回调中设置，确保没有任何插件能在此之前读到不一致的环境值。

## 本章小结

- `composeProfile` 将 profile 名字解析为多层补丁：bundle 层 → 用户层 → home 层 → overlay 层 → 遥测开关
- 每个 bundle 通过 `package.json` 的 `dsh.bundle.patch` 声明其补丁文件
- `healProfilesModuleFallback` 维护扁平模块回退目录，让 profile 能解析所有内置插件
- `composeEntries` 用 `applyEntryPatches` 从空列表构建完整配置树
- `runProfile` 将组合后的补丁传给 `boot()`，并在此之前注入环境快照和命令行参数

## 阅读思考题

1. 为什么 bundle 层在用户层下面（先应用），而不是上面？如果反过来会怎样？
2. `structuredClone` 在 `boot()` 调用中出现了两次（一次在 `allPatches`，一次在 `composeLive`），它们分别防止什么问题？
3. 如果一个 bundle 的 `package.json` 缺少 `dsh.bundle.patch` 字段，系统会在什么时候报错？这个错误是"尽早失败"还是"延迟失败"？

> **下一章预告**：补丁准备好了，现在要真正启动 Cordis Loader 了。`boot()` 函数将创建根 Context、安装 Loader、挂载 Include 入口，然后等待整棵树稳定。下一章我们进入 Cordis 的核心——插件、上下文、和 Fiber。

---

# 第 3 章 Cordis 核心：Context、Service 和依赖注入

上一章 `composeProfile` 组装好了补丁列表，`runProfile` 将它传给了 `boot()`。现在请求到达了 `packages/boot/app-boot/src/index.ts` 的 `boot()` 函数——这里将创建 Cordis 的根 Context 并启动整个 Loader。但在深入 `boot()` 之前，我们需要先理解 Cordis 的核心概念：Context 是什么，Service 怎么工作，依赖注入如何运转。

## 请求到达这里时发生了什么

`boot()` 函数在 `packages/boot/app-boot/src/index.ts` 中定义，签名如下：

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

---

# 第 4 章 Loader 与 Include：从 YAML 到插件树

上一章我们看到 `boot()` 创建了根 Context 并安装了 Loader。现在 `mountRootInclude` 要把 profile 的根配置文件挂载为整棵插件树的入口。这一章我们追踪 Loader 如何从一个 YAML 文件长出一棵完整的插件树。

## 请求到达这里时发生了什么

`mountRootInclude` 在 `packages/boot/app-boot/src/index.ts` 中定义。它的核心工作：

```typescript
export async function mountRootInclude(
  ctx: Context,
  absoluteConfigPath: string,
  patches: readonly PatchOptions[] = [],
  bareModuleBaseUrl?: string,
): Promise<Entry | undefined> {
  ctx.loader.builtins.include = bareModuleBaseUrl === undefined
    ? Include
    : class HostResolvedRootInclude extends Include { ... }
  ctx.loader.builtins.group = Group
  
  const includeConfig: Include.Config = {
    path: pathToFileURL(absoluteConfigPath).href,
    ...patches.length > 0 ? { patches: [...patches] } : {},
  }
  const rootInclude: EntryOptions = {
    id: 'include',
    name: 'cordis:include',
    config: includeConfig,
  }
  const includeId = await ctx.loader.create(rootInclude)
  // ...
}
```

关键点：它注册了两个 builtin——`Include` 和 `Group`。然后创建一个 `cordis:include` 类型的条目，指向 profile 的根配置文件。

## 关键代码路径

### Include：配置文件的解析器

`Include` 来自 `@deepseek-ai/cordis-plugin-include`。当 Loader 创建这个条目时，Include 会：

1. 读取 `includeConfig.path` 指向的 YAML 文件
2. 用 `entryListSchema`（支持 `!!js` 表达式的 YAML schema）解析它
3. 得到一个 `EntryOptions[]` 数组——每个元素代表一个要挂载的插件
4. 对这个数组应用 `includeConfig.patches`（我们传入的补丁列表）
5. 通过 Loader 逐个创建这些条目

这就是从一个 YAML 文件变成一棵插件树的过程。

### Loader 的条目解析链

`ctx.loader.create(rootInclude)` 触发 Loader 的条目创建流程。Loader 维护一个条目列表，每个条目有：

- `id`：唯一标识
- `name`：插件名（如 `'@deepseek-ai/dsh-session'`、`'cordis:include'`）
- `config`：插件配置
- `fiber`：对应的 Fiber 运行时实例

当 Include 解析完 YAML 并产生子条目列表后，Loader 对每个子条目递归执行相同的创建流程。如果子条目也是一个 `cordis:include`，它会再解析另一个 YAML 文件。如果是一个普通插件名，Loader 会：

1. 通过模块解析找到插件的入口文件
2. 导入模块
3. 验证配置（如果有 `Config` schema）
4. 创建 Fiber 实例
5. 等待所有依赖的 Service 就绪后启动插件

### 并发挂载和事务性

Loader 的条目挂载是并发的——多个条目可以同时加载。但有一个关键约束：**整个 Include 的 apply 是事务性的**。如果某个条目启动失败，Loader 会回滚整个 Include 的变更。

这也是为什么 `boot()` 在 `mountRootInclude` 之后要做两道检查：

```typescript
await ctx.get('loader')?.await()
if (ctx.get('loader') === undefined) return ctx
await assertEntriesActivated(ctx, binName)
```

`loader.await()` 等待所有条目稳定。`assertEntriesActivated` 检查每个条目的 Fiber 状态：如果有 PENDING（等待依赖）或 FAILED（启动失败），就抛出详细的错误信息。

### fail-loud 安全网

在 Loader 启动之前，`boot()` 调用了 `installFailLoud`：

```typescript
installFailLoud(NAME, process, async () => {
  await app.current?.fiber.dispose()
})
```

这是一个进程级的 `unhandledRejection` 处理器。如果某个插件在 Loader 已经稳定之后抛出了未捕获的异常（比如一个异步初始化），`installFailLoud` 会：

1. 打印诊断信息到 stderr
2. 等待终端释放（最多 `FAIL_LOUD_RELEASE_TIMEOUT_MS` = 2000ms）
3. 调用 `process.exit(1)`

它特别处理了 Loader 自身产生的 "assembled activation rejections"——这些已经在 `assertEntriesActivated` 中报告过的错误不会被重复打印。

## 调用链：从这里到下一步

当 `boot()` 返回时，一棵完整的 Cordis 插件树已经稳定运行。回到 `profile-boot.ts` 的 `runProfile`：

```typescript
const ctx = await boot(NAME, rootConfig, structuredClone(allPatches(composed)), (hostCtx) => {
  app.current = hostCtx
  hostCtx.provide(DSH_LAUNCH_ENVIRONMENT_KEY, options.environment)
  provideCmdline(hostCtx, { args: options.args, exit: ... })
})
```

`boot` 返回后，`runProfile` 设置了 HMR watcher 来监听用户补丁文件的变化，然后返回 `{ ctx, shutdown }`。

此时，整棵插件树已经就位。用户在命令行输入的 `"帮我写一个 hello.py"` 还没有被处理——它通过 `provideCmdline` 注入到了树中，等待某个应用插件读取它。

**下一步：请求将流向应用插件——特别是 headless profile 中的 Session 创建逻辑。Session 是事件溯源的对话容器，我们的请求即将在这里变成第一条 user/message 事件。**

## 关键设计决策

1. **Include 和 Group 作为 builtin**：它们不是通过 npm 包名解析的，而是直接注入到 Loader 的 builtin 注册表中。这保证了任何 profile 都能使用它们，即使 profile 目录的 `node_modules` 里没有安装。

2. **事务性挂载**：整个 Include 的子条目要么全部成功，要么全部回滚。这避免了半初始化的插件树。

3. **fail-loud 而非 fail-safe**：未捕获的插件异常直接终止进程。这比静默忽略错误要安全得多——在 AI agent 场景中，一个静默失败的工具可能产生不可预期的行为。

4. **双阶段检查**：`loader.await()` 等待所有条目稳定（包括并发加载的），`assertEntriesActivated` 验证最终状态。两道检查确保"树已稳定"和"树是健康的"是两个独立的断言。

## 本章小结

- `mountRootInclude` 注册 Include 和 Group 为 Loader builtin，然后创建根 Include 条目
- Include 解析 YAML 配置文件，产生子条目列表，通过 Loader 逐个挂载
- Loader 并发加载条目，每个条目对应一个 Fiber 实例
- `installFailLoud` 是进程级安全网，捕获异步初始化失败
- `boot()` 用 `loader.await()` + `assertEntriesActivated` 双重检查确保树健康

## 阅读思考题

1. 为什么 Include 要注册为 builtin 而不是通过正常的包名解析？如果用包名解析会有什么问题？
2. `installFailLoud` 的 `release` 回调有什么用途？为什么要在退出前等待终端释放？
3. `assertEntriesActivated` 对 PENDING 状态的条目会报告哪些依赖缺失？这如何帮助调试？

> **下一章预告**：插件树已经稳定。现在我们的请求即将进入 Session 的世界——一个事件溯源的对话容器。Session 如何把用户输入变成 `user/message` 事件？事件日志又如何组织？

---

# 第 5 章 Session：事件溯源的对话容器

上一章 Loader 启动完成，插件树稳定了。用户输入的 `"帮我写一个 hello.py"` 通过 `provideCmdline` 注入到了树中。现在，headless profile 的应用插件会读取这个命令行参数，创建一个 Session，然后把用户的输入变成第一条 `user/message` 事件。让我们看看 `packages/core/session/src/` 中发生了什么。

## 请求到达这里时发生了什么

Session 是 DeepSeek Harness 的核心数据结构——一个**事件溯源的、只追加的日志**。它不存储"当前对话状态"，而是存储一系列不可变的事件。对话状态从事件日志中派生。

在 `packages/core/session/src/types.ts` 中，`SessionEventMap` 定义了所有可能的事件类型：

```typescript
export interface SessionEventMap {
  'turn/start': { turn: number }
  'turn/end': { turn: number; reason: TurnEndReason }
  'step/start': { turn: number; step: number }
  'step/end': { turn: number; step: number }
  'user/message': UserMessage
  'assistant/chunk': { turn: number; step: number; chunk: StreamChunk }
  'assistant/message': { turn: number; step: number; message: AssistantMessage; usage?: TokenUsage }
  'tool/call': { turn: number; step: number; callId: CallId; name: string; arguments: string }
  'tool/result': { turn: number; step: number; message: ToolResultMessage; error?: ...; meta?: JsonValue }
  'request/header': { header: EpochHeader; reason: RequestHeaderReason }
  'request/context': RequestContext
  'session/end-seed': Record<string, never>
  // ... 更多
}
```

对于我们的请求，第一个被追加的事件是 `turn/start`，然后是 `user/message`。

## 关键代码路径

### Session 类：只追加的日志

`Session` 在 `packages/core/session/src/index.ts` 中定义。它的核心是一个私有数组：

```typescript
export class Session {
  private log: SessionEvent[] = []
  private readonly surfaceManager = new SurfaceManager(this.log)
  // ...
}
```

`append` 方法是写入日志的唯一入口。它做了以下事情：

1. **快照数据**：`snapshotJsonValue(data)` 对数据做一次无损 JSON 序列化+反序列化，确保数据是纯 JSON 且与调用者解耦
2. **验证**：检查事件信封格式、请求头格式、消息结构
3. **Surface 验证**：`this.surfaceManager.validateNext(event)` 检查 surface 元数据
4. **冻结**：`deepFreeze` 使事件完全不可变
5. **追加到日志**：`this.log.push(event)`
6. **通知观察者**：通过 `session/event` 事件通知持久化插件

### 事件信封

每个 `SessionEvent` 都有标准信封：`type`（事件类型）、`seq`（序号，从 0 开始连续递增）、`time`（Unix 毫秒时间戳）、`data`（事件负载）、`ignorable`（是否可忽略）。

`seq` 的连续性是整个系统的基石——它保证日志可以可靠地序列化、传输和重建。

### Surface 事件与普通事件

`SurfaceEventType` 只有三种：`user/message`、`assistant/message`、`tool/result`。这三种事件携带 `surfaceOp` 元数据，决定它们如何进入有序的对话表面。其他事件是"日志专用"——不产生模型消息，但记录了对话的结构。

### SessionStore：内存中的 Session 管理器

`SessionStore` 是一个 Cordis Service，注册为 `ctx.sessions`。创建 Session 的标准路径是 `ctx.sessions.create(id, options)`，它调用 `prepare` 构造对象，通过 `ctx.effect` 注册效果，调用 `enter` 安装发布钩子，调用 `announce` 发射 `session/created` 事件。

## 调用链：从这里到下一步

当 headless 应用插件收到命令行参数后，它会调用 `ctx.sessions.create()` 创建 Session，然后创建一个 Agent（ReactLoopAgent）来接管驱动。

**下一步：请求将流向 Agent Loop——`ReactLoopAgent` 如何接管 Session，开始一个 "turn"，调用 LLM，处理工具调用。**

## 本章小结

- Session 是事件溯源的只追加日志，`append` 是唯一的写入口
- 事件有标准信封：type、seq（连续）、time、data
- SurfaceEventType 携带 surfaceOp 决定对话表面的位置
- SessionStore 管理活跃 Session，通过 WeakMap 的发布钩子连接 append 与事件通知

---

# 第 6 章 ReactLoopAgent：Agent Loop 的状态机

上一章 Session 已经创建，`user/message` 事件已追加。现在 `ReactLoopAgent` 接管了——它是驱动整个对话循环的核心状态机。

## 请求到达这里时发生了什么

`ReactLoopAgent` 在 `packages/core/agent-loop/src/agent.ts` 中定义。核心组件：
- **Inbox**：消息队列，管理待处理的用户输入
- **phase**：状态机，三种状态——`idle`、`maintenance`、`running`
- **scope**：Agent 级别的 Cordis 作用域
- **RuntimeContextProjection**：运行时上下文注入

## 关键代码路径

### 驱动循环：kick → turn → step

`wakeDriver()` 从 idle 转到 running，调用 `kick()` 循环调用 `turn()`。每个 turn 内部有多个 step，每个 step 是一次 LLM 调用加上工具执行。

### preStep：组装 Prompt

`preStep` 从 Inbox claim 消息，组装系统提示词，通过 `agent/pre-step` waterfall 让插件拦截。

### buildRequest：组装 LLM 请求

通过 `agent/request` waterfall 让插件修改配置，调用 `llm.prepareCall()` 解析适配器，比较新旧 header 决定是否追加 `request/header` 事件。

### step 执行

流式调用 LLM → 逐 token 记录 `assistant/chunk` → 组装 `assistant/message` → 如果有工具调用则执行。

## 调用链：从这里到下一步

当 LLM 返回工具调用（如 `write_file`）时，`executeToolCalls` 被调用。

**下一步：请求将流向工具系统——ToolDefinition 如何定义工具，六段守卫管线如何校验和执行工具调用。**

---

# 第 7 章 工具系统：ToolDefinition 与六段守卫管线

上一章 ReactLoopAgent 的 step 循环中，LLM 返回了一个工具调用请求。现在请求进入了 `packages/core/tools/src/` 的工具系统。

## 关键代码路径

### 六段守卫管线

1. **Schema 验证**：检查参数是否符合 JSON Schema
2. **参数规范化**：JSON 字符串解析为对象
3. **能力检查**：检查工具是否在当前上下文可用
4. **沙箱检查**：检查操作是否在沙箱允许范围内
5. **审批**：fail-closed 策略
6. **执行**：调用工具的 `execute` 函数

### ToolResult 与 meta

`meta` 是工具私有的展示元数据，对核心系统不透明，必须 JSON 可序列化。

## 调用链：从这里到下一步

**下一步：请求将流向 FS 模块——FsTarget 如何抽象文件操作。**

---

# 第 8 章 文件系统能力：FsTarget 与 Capability Seam

## 关键设计

- `FsTarget` 接口定义在消费者侧，实现在提供者侧
- 通过 Cordis DI 连接，支持不同环境的不同实现
- 路径安全检查在工具侧，防止路径穿越

---

# 第 9 章 沙箱：操作系统级的 Agent 限制

三种后端：bwrap（Linux 命名空间）、Landlock（Linux LSM）、Seatbelt（macOS）。Landlock 优先于 bwrap，更轻量且不需要 root。

---

# 第 10 章 用户审批：fail-closed 的安全策略

审批系统采用 fail-closed：无处理器、超时、异常都默认拒绝。审批过程记录为 `approval/asked` 和 `approval/decided` 事件。

---

# 第 11 章 事件持久化：JSONL 与 SQLite 的存储引擎

两种后端：JSONL（追加友好）和 SQLite（支持查询）。Flush 在 turn 边界、session/flush、空闲时、关闭时触发。restore 模式转移对象所有权，避免深拷贝。

---

# 第 12 章 上下文压缩：当对话变长时

压缩使用 Surface 的 `replace` 操作，用摘要替换旧消息。原始事件保留在日志中。`deriveMessages` 用 `replaceGeneration` 做增量缓存。

---

# 第 13 章 Shell 执行：子进程管理与 I/O 流

使用 `bash -c` 执行命令。I/O 被同时收集和流式转发。超时和取消通过 SIGTERM/SIGKILL 实现。沙箱在 spawn 前透明包装命令。

---

# 第 14 章 Subagent：任务委派与多 Agent 协作

子 Session 通过 `SessionStore.fork()` 创建。`delegationDepth` 持久化在 SessionHeader 中。子 Agent 的工具集通过 Cordis isolate 机制隔离。

---

# 第 15 章 Workflow：脚本编排与代码运行时

脚本在受限的 `node:vm` 上下文中执行。注入式 API 控制脚本能力。超时强制。workflow 事件标记为 ignorable。

---

# 第 16 章 远程调用与 Web UI 可观测

Typert RPC 提供类型安全的远程过程调用。Gateway 管理 WebSocket 连接，转发事件。遥测基于 OpenTelemetry，可禁用。

---

# 第 17 章 测试策略：覆盖完整请求路径

分层测试：单元、集成、端到端。LLM 和文件系统用 mock。Agent Loop 测试通过事件序列验证行为。

---

# 第 18 章 完整调用链：从终端到任务完成

## 完整调用链

```
用户输入: dsh --profile headless "帮我写一个 hello.py"
    │
    ▼
bin.ts: parseDshArgs(process.argv.slice(2), readVersion())
    │  返回: { mode: 'profile', profile: 'headless', args: ['帮我写一个 hello.py'] }
    │
    ▼
profile-boot.ts: runProfile({ environment, profile: 'headless', args })
    │
    ├─ composeProfile('headless', [])
    │   ├─ prepareProfile → loadProfile → 解析 bundles
    │   └─ 组装补丁层叠
    │
    ├─ boot('dsh', rootConfig, patches)
    │   ├─ new Context() → Proxy 依赖容器
    │   ├─ ctx.plugin(Loader)
    │   ├─ mountRootInclude → Include 解析 YAML → Loader 并发挂载
    │   └─ assertEntriesActivated()
    │
    └─ 插件树就位
        │
        ├─ ctx.sessions.create(sessionId, { meta: { cwd } })
        │   └─ Session.append('turn/start') → Session.append('user/message')
        │
        ├─ AgentLoop.create() → ReactLoopAgent
        │   └─ wakeDriver() → kick() → turn()
        │       ├─ preStep() → systemPrompt.assemble() → waterfall
        │       ├─ buildRequest() → llm.prepareCall() → waterfall
        │       └─ llm.generate() → stream chunks → assistant/message
        │           │
        │           ▼ tool_calls: write_file
        │           ├─ validateSchema()
        │           ├─ 能力检查 → FsTarget
        │           ├─ 沙箱检查 → bwrap/Landlock/Seatbelt
        │           ├─ 审批 → fail-closed
        │           └─ fs.writeFile() → tool/result
        │               │
        │               ▼ tool_calls: shell (python hello.py)
        │               ├─ spawn('bash', ['-c', 'python hello.py'])
        │               └─ tool/result (stdout: "Hello, World!")
        │                   │
        │                   ▼ assistant/message (final)
        │                   └─ turn/end → 持久化 flush
```

## 关键设计模式

1. **事件溯源**：所有状态变化记录为不可变事件
2. **能力接缝**：接口在消费者侧，实现在提供者侧
3. **补丁层叠**：YAML 补丁列表组合配置
4. **fail-closed**：不确定时拒绝
5. **深度防御**：多层安全检查
6. **Proxy DI**：Context 是 Proxy，DI 对使用者透明

---

**全书完**
