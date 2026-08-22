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
