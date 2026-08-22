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

3. **环境快照在启动前冻结**：`loadLayeredEnv` 在任何插件挂载之前就运行，确保整个生命周期使用同一份不可变的环境值。

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
