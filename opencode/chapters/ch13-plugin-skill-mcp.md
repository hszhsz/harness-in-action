# 第 13 章 可扩展性边界:插件、技能与 MCP 如何安全地接入运行时

一个 agent 的生命力,很大程度上取决于它能不能在**不改核心**的前提下长出新能力。OpenCode 在这件事上有三条并行的扩展通道:**插件(plugin)**让外部代码挂进运行时的关键接缝、**技能(skill)**让一段 Markdown 指令按需注入对话、**MCP(Model Context Protocol)**把外部服务器接成工具。

这一章我们沿着"外部能力如何进入运行时"这条线,逐个拆开三套机制。你会发现一个共同的母题:每一处"让外部东西进来"的接缝,OpenCode 都配了一道信任边界——作用域隔离、路径逃逸防御、权限过滤。扩展性和安全性在这里不是对立的,而是被同一套设计同时满足。

## 13.1 插件系统:带作用域生命周期的钩子注册表

插件的核心定义在 `packages/core/src/plugin.ts`。一个插件就是一个 `{ id, effect }`:

```ts
export function define<R>(input: { id: ID; effect: Effect.Effect<HookFunctions | void, never, R> }) {
  return input
}
```

`effect` 跑完后可以返回一组**钩子函数(HookFunctions)**。钩子的类型是封死的,定义在 `HookSpec` 里,目前有四个:`catalog.transform`(改写模型目录)、`account.switched`(账号切换)、`aisdk.language` 和 `aisdk.sdk`(在 AI SDK 这一层介入模型构造)。这是一个**显式枚举的扩展点清单**——插件不能随便 hook 任意位置,只能挂进核心预先开放的这几个接缝。扩展面被刻意收窄,这是可控性的第一道保证。

`PluginV2.Service` 的 `add` 实现是整套机制最讲究的地方:

```ts
add: Effect.fn("Plugin.add")(function* (input) {
  yield* locks.withLock(input.id)(
    Effect.gen(function* () {
      const existing = hooks.find((item) => item.id === input.id)
      if (existing) yield* Scope.close(existing.scope, Exit.void).pipe(Effect.ignore)
      const childScope = yield* Scope.fork(scope)
      const result = yield* input.effect.pipe(
        Scope.provide(childScope),
        Effect.withSpan("Plugin.load", { ... }),
        Effect.onExit((exit) => (Exit.isFailure(exit) ? Scope.close(childScope, exit) : Effect.void)),
      )
      hooks = [...hooks.filter((item) => item.id !== input.id), { id: input.id, hooks: result ?? {}, scope: childScope }]
      yield* events.publish(Event.Added, { id: input.id })
    }),
  )
}),
```

这里有四个值得拆解的设计决策:

1. **每个 id 一把锁**:`KeyedMutex.makeUnsafe<ID>()` 给每个插件 id 配独立的互斥锁,`locks.withLock(input.id)` 保证同一个插件的加载/卸载串行化——重复加载同名插件不会产生竞态。
2. **每个插件一个子作用域**:`Scope.fork(scope)` 给插件 effect 一个独立的 `childScope`,插件注册的所有资源(文件句柄、订阅、定时器)都挂在这个子作用域上。
3. **热替换**:如果同名插件已存在,先 `Scope.close(existing.scope)` 把旧实例连同它的全部资源干净地关掉,再装新的。这让插件可以被安全地重新加载。
4. **加载失败即清理**:`Effect.onExit` 里那个判断——如果插件 effect 以失败退出,立刻 `Scope.close(childScope, exit)` 回收已经申请的资源。一个加载到一半崩掉的插件,不会留下泄漏的句柄。

钩子的触发在 `triggerFor` 里,用了 immer 的 draft 机制做"可变输出":核心把 `output` 对象转成 draft 交给每个钩子修改,所有钩子跑完后 `finishDraft` 收成最终值。这意味着多个插件可以**依次**对同一个输出(比如 `aisdk.language` 的 `language` 字段)叠加修改,而核心拿到的始终是不可变的结果。监听者之间通过 draft 协作,但谁也改不了别人已经定稿的东西。

## 13.2 插件启动:有序装配与依赖注入

插件不会凭空运行,它们的服务依赖由 `packages/core/src/plugin/boot.ts` 统一注入。`PluginBoot` 的 `add` 函数把一长串服务用 `Effect.provideService` 一次性喂给插件 effect:

```ts
const add = Effect.fn("PluginBoot.add")(function* (input: Plugin) {
  yield* plugin.add({
    id: input.id,
    effect: input.effect.pipe(
      Effect.provideService(Catalog.Service, catalog),
      Effect.provideService(CommandV2.Service, commands),
      Effect.provideService(Auth.Service, accounts),
      ... // 共 13 个服务
      Effect.provideService(PluginV2.Service, plugin),
    ),
  })
})
```

这是依赖注入在插件边界上的具体落地:插件代码声明它需要哪些服务(通过 effect 的 `R` 类型参数),启动器统一供给。插件拿到的是同一批共享的服务单例,而不是各自新建——`catalog`、`config`、`skill` 这些都是 `yield*` 出来的同一个实例。

`boot` 函数里的注册顺序是刻意的:

```ts
yield* add(EnvPlugin)
yield* add(AccountPlugin)
yield* add(AgentPlugin.Plugin)
yield* add(CommandPlugin.Plugin)
yield* add(SkillPlugin.Plugin)
for (const item of ProviderPlugins) { yield* add(item) }
yield* add(ModelsDevPlugin)
yield* add(ConfigProviderPlugin.Plugin)
yield* add(ConfigAgentPlugin.Plugin)
yield* add(ConfigSkillPlugin.Plugin)
...
```

注意 `SkillPlugin`(内置 skill 来源)先于 `ConfigSkillPlugin`(从用户配置发现 skill)。这个顺序决定了能力叠加的层次:先铺内置,再叠用户自定义。

整个 boot 流程被 `Effect.forkScoped` 派到后台运行,结果通过一个 `Deferred` 暴露:

```ts
yield* boot.pipe(Effect.exit, Effect.flatMap((exit) => Deferred.done(done, exit)), Effect.forkScoped)
return Service.of({ wait: () => Deferred.await(done) })
```

谁需要确保插件全部装配完毕,就 `yield* boot.wait()`。我们马上会在 skill 工具里看到这个 `wait()` 的用途——它是"工具运行前,扩展必须就绪"的同步点。

## 13.3 技能系统:三种来源、按需加载、权限过滤

技能(skill)是 OpenCode 给 LLM 的"可调用说明书":一段带 frontmatter 的 Markdown,描述某类任务该怎么做,以及配套的脚本/文件在哪。定义在 `packages/core/src/skill.ts`。

技能有三种来源(`Source`),用标签联合区分:

```ts
DirectorySource  // type: "directory", path
UrlSource        // type: "url", url
EmbeddedSource   // type: "embedded", skill: Info
```

`DirectorySource` 指向本地目录,`UrlSource` 指向远程技能仓库,`EmbeddedSource` 是直接内嵌在代码里的技能(比如 13.1 没提到的、`plugin/skill.ts` 里那个 `customize-opencode` 内置技能,专门用来辅助用户配置 OpenCode 自身)。

加载逻辑 `load` 对目录/远程来源走文件系统扫描:

```ts
const files = yield* fs.glob("{*.md,**/SKILL.md}", { cwd: directory, absolute: true, include: "file", symlink: true, dot: true })
  .pipe(Effect.catch(() => Effect.succeed([] as string[])))
for (const filepath of files.toSorted()) {
  const content = yield* fs.readFileStringSafe(filepath).pipe(Effect.catch(() => Effect.succeed(undefined)))
  if (!content) continue
  const markdown = ConfigMarkdown.parseOption(content)
  if (!markdown) continue
  const frontmatter = decodeFrontmatter(markdown.data).valueOrUndefined
  if (!frontmatter) continue
  ...
}
```

glob 模式 `{*.md,**/SKILL.md}` 透露了两种约定:目录顶层的任意 `.md` 文件,或任意子目录里名为 `SKILL.md` 的文件。每一步都用 `Effect.catch` 兜底——读不到文件、解析不出 frontmatter,都只是 `continue` 跳过这一个,绝不让单个坏 skill 文件炸掉整个加载流程。技能名的推断也有讲究:有 frontmatter 的 `name` 就用它;没有,且文件就在目录顶层,则用文件名(去掉 `.md`)。

加载结果进 `cache`(一个 `Map<string, Info[]>`,按 `Source.key` 缓存),`list()` 把所有来源的技能汇总、按名去重。源码里还留了一个诚实的 `QUESTION(Dax)` 注释,问本地技能要不要随文件监听失效——说明这套缓存的失效策略仍是开放问题。

最关键的安全接缝是 `available`:

```ts
export const available = (skills: ReadonlyArray<Info>, agent: AgentV2.Info) =>
  skills.filter((skill) => PermissionV2.evaluate("skill", skill.name, agent.permissions).effect !== "deny")
```

技能不是"加载了就能用"。一个 agent 能看到哪些技能,要经过第 10 章那套权限引擎过滤:对每个技能,用 `("skill", skill.name)` 去 `evaluate`,只要结果是 `deny` 就从可用列表里剔除。这就是为什么第 10 章里 compaction/title/summary 这些 all-deny 的子 agent 根本看不到任何技能——权限系统在这里二次发挥作用。

## 13.4 skill 工具:加载即一次权限断言

技能最终要被 LLM **调用**进来。这条路径在 `packages/core/src/tool/skill.ts` 里实现为一个名叫 `skill` 的工具。它的 `execute` 先做两件准备,再做一次关键的权限断言:

```ts
yield* boot.wait()   // 等插件全部装配完毕
...
const current = yield* skills.list()
const skill = current.find((skill) => skill.name === input.name)
if (!skill) return yield* unableToLoad(input.name)
yield* permission.assert({
  action: name,
  resources: [skill.name],
  save: [skill.name],
  sessionID: context.sessionID,
  agent: context.agent,
  source: { type: "tool", messageID: context.assistantMessageID, callID: context.toolCallID },
})
```

`boot.wait()` 正是 13.2 埋下的同步点:在 skill 工具真正干活前,确保所有提供 skill 来源的插件都已经注册完毕,否则 `skills.list()` 可能漏掉刚要加载的技能。

`permission.assert` 带了 `save: [skill.name]`——这意味着用户对某个技能授权时可以选"always",授权会被持久化(回顾第 10 章的项目级 saved 规则)。下次再用同一个技能,就不必重复确认。

工具的输出 `toModelOutput` 把技能内容包成一段结构化的 `<skill_content>`,并附上技能所在目录和一份**采样过的**文件清单:

```ts
const files = path.basename(skill.location) === "SKILL.md"
  ? (yield* fs.glob("**/*", { cwd: directory, ... })).filter((file) => path.basename(file) !== "SKILL.md").toSorted().slice(0, FILE_LIMIT)
  : []
```

`FILE_LIMIT = 10`——只列前 10 个附属文件,输出里也明确标注 `Note: file list is sampled`。这是对上下文预算的克制:技能可能带几十个脚本,但塞给 LLM 的清单有上限,避免一次 skill 加载就吃掉大量 token。这与第 11 章 `TOOL_OUTPUT_MAX_CHARS` 的思路一脉相承——**任何可能膨胀的工具输出都要有硬上限**。

## 13.5 远程技能发现:把路径逃逸防御做到偏执

`UrlSource` 允许从一个远程 URL 拉取整个技能仓库。这是把**外部内容下载到本地文件系统**——典型的高危操作。`packages/core/src/skill/discovery.ts` 对此的防御称得上偏执,值得专门一节。

`pull` 先抓远程的 `index.json`(一个 `{ skills: [{ name, files }] }` 结构),然后对每个技能、每个文件做层层校验后才下载。两个守卫函数是核心:

`isSafeSegment` 校验技能名,拒绝空串、`.`、`..`、含 `/`、`\`、`\0` 的名字——技能名会变成本地目录名,任何能跳出目录的字符都被挡掉。

`isSafeRelativePath` 校验每个文件路径,更严格:

```ts
function isSafeRelativePath(value: string) {
  const segments = value.split("/")
  return (
    value.length > 0 && !value.includes("\\") && !value.includes("\0") &&
    !value.includes("?") && !value.includes("#") && !URL.canParse(value) &&
    !path.posix.isAbsolute(value) && !path.win32.isAbsolute(value) &&
    segments.every((segment) => {
      try {
        const decoded = decodeURIComponent(segment)
        return decoded.length > 0 && decoded !== "." && decoded !== ".." &&
          !decoded.includes("/") && !decoded.includes("\\") && !decoded.includes("\0")
      } catch { return false }
    })
  )
}
```

注意它不只检查原始字符串,还对每个路径段 `decodeURIComponent` **解码后再查一遍**——防的是 `%2e%2e`(URL 编码的 `..`)这种绕过。posix 和 win32 的绝对路径判断都做,意味着无论运行在哪个平台都不会被另一平台的路径语法钻空子。

光有字符串校验还不够,落地时还有一道**实际路径包含检查**:

```ts
const root = path.resolve(sourceRoot, skill.name)
if (!FSUtil.contains(sourceRoot, root) || root === sourceRoot) return []
...
const destination = path.resolve(root, file)
if (!FSUtil.contains(root, destination) || destination === root) return undefined
```

`FSUtil.contains` 在 `path.resolve` 之后确认最终的目标路径真的落在预期目录内。字符串层面的校验是第一道闸,解析后的路径包含检查是第二道——两道独立的防线叠加,这正是第 12 章和第 10 章反复出现的"边界不信任"原则在文件下载场景的体现。此外还有一道来源校验:`if (resource.origin !== source.origin) return undefined`,确保每个文件 URL 与索引同源,不会被索引诱导去别的域名拉东西。

下载本身受并发限制(`skillConcurrency = 4`、`fileConcurrency = 8`)、HTTP 客户端配了指数退避重试(`Schedule.exponential(200).pipe(Schedule.jittered)`,重试 2 次),且 `download` 前先查 `fs.exists` 跳过已下载的文件——幂等、克制、有节流。

## 13.6 技能从哪来:配置驱动的来源装配

技能来源不是写死的,而是由 `packages/core/src/config/plugin/skill.ts` 这个配置插件从用户配置里发现。它对每个配置目录,自动添加 `skill/` 和 `skills/` 两个子目录作为来源:

```ts
for (const directory of directories) {
  editor.source(new SkillV2.DirectorySource({ type: "directory", path: AbsolutePath.make(path.join(directory, "skill")) }))
  editor.source(new SkillV2.DirectorySource({ type: "directory", path: AbsolutePath.make(path.join(directory, "skills")) }))
}
```

对配置文档里显式列出的 `skills` 条目,它会区分:能解析成 `http(s)` URL 的当远程来源,`~/` 开头的展开到用户主目录,其余按相对/绝对路径解析成本地目录来源。这套逻辑让用户既能用约定俗成的 `skills/` 目录,也能在配置里点名引入远程技能库。

## 13.7 MCP:把外部服务器声明成工具来源

第三条扩展通道是 MCP。在 v2 里,MCP 的配置 schema 定义在 `packages/core/src/config/mcp.ts`,区分两类服务器:

```ts
class Local {   // type: "local"
  command: string[]              // 启动命令
  environment?: Record<string, string>
  disabled?: boolean
  timeout?: PositiveInt
}
class Remote {  // type: "remote"
  url: string
  headers?: Record<string, string>
  oauth?: OAuth | false
  disabled?: boolean
  timeout?: PositiveInt
}
```

`Local` 是本地子进程式的 MCP 服务器(用 `command` 数组启动),`Remote` 是远程 HTTP 式的,带 `headers` 和可选的 `OAuth` 配置。`OAuth` schema 里那个 `callback_port` 用 `Schema.isBetween({ minimum: 1, maximum: 65535 })` 把端口约束在合法范围——又一处"把约束写进类型"的细节。两类服务器都有 `disabled` 开关和 `timeout`,让用户能精细控制每个外部依赖的启停与超时。

值得注意的是 v2 的核心代码里,MCP 工具的**运行时注册**是和静态内置工具刻意分开的。`packages/core/src/tool/builtins.ts` 顶部的注释把这个边界说得很清楚:

> Dynamic MCP and plugin tools later use separate scoped canonical registrations, while provider/model filtering belongs to a future materialization phase rather than this static list. ... Keep MCP and plugin transforms separate from this static built-in list.

也就是说:`builtins.ts` 里 `Layer.mergeAll(...)` 合并的是那批随版本出厂、位置固定的工具(bash/edit/glob/grep/read/skill/write 等);而 MCP 和插件带来的工具走**独立的、带作用域的注册**路径。这个分离不是偶然——内置工具的生命周期跟随进程,而外部工具(MCP 子进程、插件)可能随时上线下线,把它们放进独立作用域,才能在某个 MCP 服务器崩溃或被禁用时,干净地回收它的工具而不影响内置工具。这与 13.1 里"每个插件一个子作用域"的设计如出一辙。

## 13.8 三套机制,一个共同的母题

回头看插件、技能、MCP 这三条扩展通道,表面上各管各的,底下却共享同一套设计哲学:

- **扩展点显式枚举**:插件只能挂进 `HookSpec` 里那四个钩子,不能 hook 任意位置。
- **作用域隔离生命周期**:每个插件、每个动态工具来源都活在独立的 `Scope` 里,失败即回收、卸载即清理。
- **权限引擎二次把关**:技能加载要过 `available` 的 deny 过滤,skill 工具调用要过 `permission.assert`——能力进来了,但用不用得了还得问权限系统。
- **边界处偏执地不信任外部输入**:远程技能下载做了字符串校验 + 解码后再校验 + 路径包含检查 + 同源检查四重防线;MCP 端口被约束进合法区间。

扩展性给了 OpenCode 生长的空间,而这些边界保证了生长不会失控。一个能装任意插件、拉任意远程技能、连任意外部服务器的 agent,如果没有这些闸门,就是一个巨大的攻击面。OpenCode 的做法是:**每开一扇门,就在门口配一道锁**。

## 本章小结

- OpenCode 有三条不改核心即可扩展的通道:插件(挂进运行时钩子)、技能(按需注入 Markdown 指令)、MCP(把外部服务器接成工具)。
- 插件系统 `PluginV2` 用 `HookSpec` 显式枚举四个扩展点;`add` 用 `KeyedMutex` 串行化、给每个插件 fork 独立 `Scope`、支持热替换、加载失败即 `Scope.close` 回收;`triggerFor` 用 immer draft 让多个钩子协作修改输出。
- `PluginBoot` 统一注入 13 个服务依赖、按刻意顺序装配(内置 skill 先于配置 skill),并通过 `Deferred` 暴露 `wait()` 作为"扩展就绪"同步点。
- 技能 `SkillV2` 支持 directory/url/embedded 三种来源,加载时用 glob `{*.md,**/SKILL.md}` 扫描、逐文件 `Effect.catch` 容错、按 `Source.key` 缓存;`available` 用第 10 章权限引擎的 `evaluate("skill", name)` 过滤掉 deny 的技能。
- skill 工具加载前 `boot.wait()`,加载时 `permission.assert(save: [name])` 支持持久化授权,输出用 `FILE_LIMIT = 10` 采样附属文件,呼应第 11 章的输出上限思路。
- 远程技能发现做了四重路径逃逸防御:`isSafeSegment`/`isSafeRelativePath` 字符串校验、`decodeURIComponent` 解码后再校验、`FSUtil.contains` 路径包含检查、同源检查;配并发限制、指数退避重试、按 exists 幂等跳过。
- MCP 配置区分 Local/Remote,端口用 `isBetween(1, 65535)` 约束;MCP 与插件工具走独立作用域注册,与静态内置工具刻意分离,以便动态工具能被干净回收。

## 动手实验

1. **写一个最小内置技能**:在你的项目里建一个 `skills/hello/SKILL.md`,frontmatter 写 `name: hello` 和一句 `description`,正文写几句指令。启动 OpenCode 后,在系统上下文里确认这个技能出现在可用列表中,然后让 agent 调用 `skill` 工具加载它。对照 `tool/skill.ts` 的 `toModelOutput`,观察注入的 `<skill_content>` 结构和那份采样文件清单。

2. **验证权限对技能的过滤**:给某个 agent 配一条 `{action: "skill", resource: "hello", effect: "deny"}` 的权限规则,重启后确认这个 agent 的可用技能列表里**没有** hello。然后对照 `skill.ts` 的 `available` 函数,理解 `PermissionV2.evaluate("skill", skill.name, ...)` 是怎么把它过滤掉的。

3. **复现路径逃逸防御**:阅读 `skill/discovery.ts` 的 `isSafeRelativePath`,在本地写个小脚本,分别用 `../etc/passwd`、`%2e%2e/secret`、`/abs/path`、`a/../../b` 这几个输入调用它(把函数逻辑抄出来),观察哪些被拒、为什么。重点体会"解码后再校验"这一步挡住了哪类攻击。

4. **画出插件作用域的生命周期**:阅读 `plugin.ts` 的 `add` 函数,画一张图标出:`Scope.fork` 在哪创建子作用域、`Effect.onExit` 在什么条件下 `Scope.close`、重复 `add` 同名插件时旧作用域如何被关闭。然后思考:如果一个插件在 `effect` 里 `addFinalizer` 注册了清理逻辑,这个 finalizer 会在插件被热替换时被触发吗?

---

下一章我们把视角拉到最外层:第 14 章将看 OpenCode 的**服务端与多端架构**,读 server 相关代码,看 CLI、TUI、HTTP 这些不同的"端"如何共享同一套 core 运行时——以及第 12 章那条"历史事件 + 实时 PubSub"的事件流,如何成为让多个端实时同步同一个会话状态的骨架。
