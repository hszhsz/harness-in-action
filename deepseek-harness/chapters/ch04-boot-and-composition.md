# 第 4 章　启动与组合：从空列表到完整产品

> dsh 最反直觉的一点：一个功能完整的 Agent 产品，不是"启动时加载一堆模块"，而是"**从空 entry 列表开始，一层层 patch 出来的**"。这一章拆开这条组装线。

## 4.1　`dsh` CLI：`apps/cli/src/bin.ts` 的按需动态导入

`dsh` 命令的入口是 `apps/cli/src/bin.ts`。它只有三个逻辑分支，按模式动态导入：

```ts
const invocation = parseDshArgs(process.argv.slice(2), readVersion())

switch (invocation.mode) {
  case 'profile': {
    const { runProfile } = await import('./profile-boot.ts')
    await runProfile({ environment: loadLayeredEnv('dsh'), profile: invocation.profile, patchFiles: invocation.patches, args: invocation.args })
    break
  }
  case 'plugin': {
    const { runPlugin } = await import('./plugin.ts')
    process.exit(runPlugin(invocation.profile, invocation.args))
    break
  }
  case 'dump-config': {
    const { runDumpConfig } = await import('./dump-config.ts')
    runDumpConfig(invocation.profile, invocation.defaultOnly, invocation.patches)
    break
  }
  default:
    invocation satisfies never
    throw new Error(`dsh: unhandled invocation mode ${JSON.stringify(invocation)}`)
}
```

三种模式：
- **`dsh --profile <name>`**：启动指定 profile（`dsh web` 是 `--profile web` 的别名；`dsh --profile headless "job"` 跑一次性任务后退出）；
- **`dsh plugin --profile <name> <pnpm args>`**：在 profile 目录里转发给 pnpm 管理插件；
- **`dsh --dump-config` / `--dump-default-config`**：不启动，打印组合后的配置树。

注意注释："Dynamic imports per mode keep unrelated modes out of each dispatch path"——按模式动态导入，无关模式的代码不进派发路径。CLI 本身不解析业务 flag：**启动器只解析自己的 flag，第一个不认识的 token 之后全部交给 booted profile 里的 app 插件解析**（`packages/boot/cmdline` 提供共享的不可变参数快照）。

## 4.2　Profile / Bundle / Patch 三层组合

一个运行中的 dsh 是一棵插件树，从有序层组合而来。[[docs/architecture.md]](https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/architecture.md)

- **Profile（画像）**：Harness home（`$DSH_HOME`，默认 `~/.dsh`）下 `profiles/<name>/` 里的命名组合。它列出自己堆叠的 bundles、装着 out-of-tree 插件，还有用户自己的 `cordis.patch.yml`。`web` 和 `headless` 作为模板随发布提供。
- **Bundle（束）**：Cordis 配置行 + 它们装载的代码的分发格式。每个 bundle 在自己的 `package.json` 的 `dsh` 字段里声明：`dsh.profile` 列出 profile 的 bundles，`dsh.bundle` 指向 bundle 的 patch 文件。
- **Patch（补丁）**：按 id 定位一行并替换它的整个 config，或插入新行。

**层级顺序**（对空 entry 列表依次应用）：

```
每个 bundle 的 patch（按 profile 列出的顺序）
  → profile 的 cordis.patch.yml
    → home 级 $DSH_HOME/cordis.patch.yml
      → --patch overlays
```

后一层覆盖前一层，同一个 id 最后写入者胜。**注意 patch 是整行替换，不是深合并**——`app-boot` 的 README 明说 "A user patch replaces the whole matched config: an id-targeted patch does not deep-merge, so a profile override restates the bundle fields it keeps." 这意味着每个 patch 都要完整重述它要保留的字段，代价是啰嗦，收益是**任何一行配置的最终形态都可以从 patch 链精确推导**，没有隐式合并的惊喜。

## 4.3　`app-boot` 的 `boot()`：fail loud 与 partial-context 回卷

`packages/boot/app-boot/src/index.ts` 的 `boot()` 是组装线的总控（第 3.5 节看过它的回卷逻辑）。完整流程：

```ts
export async function boot(binName, absoluteConfigPath, patches?, prepare?, bareModuleBaseUrl?): Promise<Context> {
  const ctx = new Context()
  let stage = 'host preparation failed'
  try {
    ctx.baseUrl = pathToFileURL(dirname(absoluteConfigPath)).href + '/'
    ctx.provide('dshHomePath', dshHomePath)
    await ctx.plugin(Loader)
    await prepare?.(ctx)
    stage = 'plugin tree failed to load'
    await mountRootInclude(ctx, absoluteConfigPath, patches, bareModuleBaseUrl)
    await ctx.get('loader')?.await()
    if (ctx.get('loader') === undefined) return ctx
    await assertEntriesActivated(ctx, binName)
    return ctx
  } catch (cause) {
    await ctx.fiber.dispose()
    ...
  }
}
```

关键设计点：

1. **两个失败标签**：`stage` 在 `prepare` 前后切换——prepare 失败是"host setup"问题，插件树装载失败是"plugin tree"问题，错误信息因此能准确分类。
2. **Loader 可能随 surface 一起消失**：`await ctx.get('loader')?.await()` 之后要**重新检查** `ctx.get('loader') === undefined`——因为 surface 可能在启动完成前就按用户要求退出了，此时直接返回而不是抛错。
3. **`assertEntriesActivated`**：装载 settle 后，检查每个 enabled entry 是否真的激活了——有 entry 没有 fiber（插件名解析失败）就抛错，把每个失败插件的原始 stack 拼进启动错误。**"Misconfiguration fails loud"**。
4. **fail loud 的收尾**：`installFailLoud` 把未处理的 boot 期 rejection 变成一行带标签的 stderr + `exit(1)`；如果 bin 拥有终端，`release` 回调先恢复终端（raw mode、bracketed paste）再退出，避免把用户的 shell 弄坏。

还有 `--dump-config` 的实现哲学：`renderConfigDump` 用 include 自己的 parser 和 patch 算法离线组合配置，**结果与 `boot()` 装载的完全一致**（"so composition, flag derivation, and config dumps cannot drift from what boots"）。

## 4.4　`dsh-base` → `dsh-web-app` / `dsh-headless`：模板 profile 的解剖

**`dsh-base`**（`packages/bundle/base/cordis.patch.yml`）是每个 profile 的第一层：一个覆盖空根的 `insert`，插入模型适配器（`llm`）、会话（`session`）、工具（`tools`）、持久化、沙箱与审批策略、settings、credentials、telemetry、typert（`typert-registry`/`typert-loader`/`api-gateway`）、session-title、user-questions、agent、system-prompt……还有 `timer` 和 `hmr`（Cordis 自带的定时器与热重载服务）。

**`dsh-web-app`** 在 base 之上加浏览器表面：`code-runtime`（worker-thread）、`storage` + `storage-json` + `storage-domain`、`message-feedback`、`session-log-download`（导出）、`workspace`、`session-projection-cache`，以及 web host 行（webserver、client modules、前端静态托管）。它还故意把 base 的某些行**整行重述**——比如 `system-prompt` 的 persona 换成 "You are a coding agent powered by the {{model}} model. Your working directory is {{cwd}}."。

**`dsh-headless`** 更精简：一次性任务模式，**不挂任何 Host、HTTP server、Web runtime 或 browser 插件**。它关掉共享 HMR（`hmr: disabled: true`），插入 `code-runtime` 和两个核心插件：

```yaml
- id: headless-startup
  name: '@deepseek-ai/dsh-headless/startup'
- id: headless-runner
  name: '@deepseek-ai/dsh-headless'
  inject: [headlessStartup]
  config:
    task: !!js ctx.headlessStartup.task
```

一个 provider 插件注入 `cmdlineArgs`、解析位置参数（`dsh --profile headless "<task>"`），direct driver 通过 core registry 创建 Agent 并打印它的 durable result。**同一个 `dsh-base`，一个 bundle 变成浏览器应用，另一个 bundle 变成一次性 runner——这就是分层组合的威力。**

## 4.5　配置即代码：`!!js` 表达式与 HMR

Cordis Loader 把 `!!js` 解析为表达式节点。dsh 在 `cordis.yml` 里用它做**配置时计算**：

```yaml
- id: tools
  config:
    mode: !!js process.env.DSH_TOOLS_MODE   # web bundle: 环境变量决定工具模式
```

`config`（在声明的注入激活后、针对该插件上下文求值）和 `disabled`（每次挂载决策时、针对 loader 上下文求值）支持 `!!js`，其他元数据保持字面量。`cordis.yml` 只允许 `!!js`（绝不 `!js`）。

**用户 patch 是热更新的**：`watchUserPatches` 把用户 patch 文件注册到 Cordis 的 HMR 服务，每次增删改都事务性地重新组合完整 patch 列表。一个 rejected 的读/解析/候选会让**最后一棵好树继续跑**，并广播 `hmr/config-update-failed`——配置坏了不会杀死正在运行的 Agent。

## 本章小结

dsh 的组合机制可以一句话概括：**产品 = 空列表 + 有序 patch 层**。CLI 只做模式分发，`boot()` 只做装载与审计，bundle 层决定"是什么产品"，用户 patch 层决定"我的定制"，`!!js` 表达式让配置能感知环境，HMR 让配置变更不断服。这一整套设计让"换掉主循环""换掉模型适配器""换掉整个表面"都只是改几行配置的事——这正是第 2 章"一切皆插件"红线的落地。

## 动手实验

1. 运行 `pnpm dsh --profile web --dump-config > /tmp/web-tree.yml`，数一数 entry 总数，按来源（base/web-app/用户层）分类。
2. 读 `packages/bundle/base/cordis.patch.yml` 全文，找出所有 `inject` 行，理解每个依赖关系。
3. 修改 `~/.dsh/profiles/web/cordis.patch.yml` 加一行覆盖某个 entry 的 config，跑 `--dump-config` 看层级生效——再删掉恢复。
4. 读 `apps/cli/src/profile-boot.ts`，看 `runProfile` 如何把 profile 解析与 `boot()` 接起来。
