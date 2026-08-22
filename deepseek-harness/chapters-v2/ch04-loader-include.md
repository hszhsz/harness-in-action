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
