# 第 13 章　自扩展的 Extension 系统：不改核心，装上任意能力

> 一个真正可复用的内核，不是「功能全」，而是「留得住口子」。pi 把核心做到极薄，再开出一整面墙的扩展点——于是「加功能」这件事，从「改源码、等发版」变成了「往一个目录里丢一个 `.ts` 文件」。

前十二章我们反复看到 pi 的同一句口头禅:**核心只做机制,策略全部外包。** 前面那些「外包」散落在各处——权限外包给容器(第 10 章)、执行后端外包给 `BashOperations`(第 9 章)、压缩策略可被替换(第 12 章)。这一章我们看的是**承接这些外包的总闸门**:Extension 系统。它是 pi 「自扩展」能力的核心——用户不碰一行核心代码,就能给 Agent 装上工具、命令、快捷键、UI 组件,甚至接管权限和上下文。

---

## 13.1 一个扩展长什么样:导出一个工厂函数

先看最小的一个扩展。pi 自带的 `tps` 扩展(`.pi/extensions/tps.ts`),功能是「每轮结束后算一下生成速度(tokens/秒)并提示」,全文不到 50 行:

```ts
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function (pi: ExtensionAPI) {
	let agentStartMs: number | null = null;

	pi.on("agent_start", () => { agentStartMs = Date.now(); });

	pi.on("agent_end", (event, ctx) => {
		if (!ctx.hasUI) return;
		if (agentStartMs === null) return;
		const elapsedMs = Date.now() - agentStartMs;
		// ...遍历 event.messages 累加 usage.output...
		const tokensPerSecond = output / (elapsedMs / 1000);
		ctx.ui.notify(`TPS ${tokensPerSecond.toFixed(1)} tok/s ...`, "info");
	});
}
```

这就是一个完整的扩展。它揭示了 pi 扩展模型的三个基本约定:

1. **一个扩展 = 一个默认导出的工厂函数**,签名是 `(pi: ExtensionAPI) => void`。
2. **工厂函数在「加载时」运行一次**,它的职责是「**注册**」——通过 `pi.on(...)` 注册事件处理器,通过 `pi.registerTool(...)` 注册工具,等等。
3. **真正的逻辑在回调里**,回调被调用时会拿到 `(event, ctx)`——`event` 是这次事件的数据,`ctx` 是「当前时刻」的 `ExtensionContext`(能访问 UI、会话、模型……)。

注意 `tps` 用一个闭包变量 `agentStartMs` 在两个事件之间传递状态。扩展就是普通的 JS 模块,闭包、import、任何 npm 包都能用——它不是一种受限的「插件 DSL」,而是**完整的代码**。

## 13.2 加载机制:用 jiti 直接吃 TypeScript

扩展是 `.ts` 文件,但 pi 可以是编译进 Bun 的单文件二进制——它怎么在运行时加载一个用户随手写的 TypeScript?答案是 `jiti`(`core/extensions/loader.ts`):

```ts
const jiti = createJiti(import.meta.url, {
	moduleCache: false,
	...(isBunBinary
		? { virtualModules: VIRTUAL_MODULES, tryNative: false }  // 二进制:用打包进去的虚拟模块
		: { alias: getAliases() }),                              // 开发态:别名指到 node_modules
});
const module = await jiti.import(extensionPath, { default: true });
const factory = module as ExtensionFactory;
```

`jiti` 是一个「即时 TypeScript 加载器」——它能在运行时直接 import `.ts`,无需预编译。这里最值得玩味的是 `virtualModules`/`alias` 这对设计:

- 扩展里会写 `import type { ExtensionAPI } from "@earendil-works/pi-coding-agent"`,还可能 import `typebox`、`pi-ai`、`pi-tui`。
- 但扩展是被宿主进程动态加载的,它没有自己的 `node_modules`。这些 import 怎么解析?
- pi 把这些「扩展可能用到的包」**静态打包进自己**(`VIRTUAL_MODULES`),再通过 jiti 的虚拟模块机制喂给扩展。于是扩展 `import "pi-ai"` 拿到的,正是**宿主自己用的那一份**——版本一致,实例共享,不会出现「两个 pi-ai 互不认识」的诡异 bug。

> **给 Agent 开发者的启示**:让用户写扩展,最大的工程坑不是「怎么执行用户代码」(jiti 一行搞定),而是「**用户代码怎么 import 到你的内部类型和工具**」。pi 的解法——把可被扩展依赖的包静态打包 + 虚拟模块注入——保证了「宿主与扩展共享同一份依赖实例」。这是动态插件系统能稳定工作的隐形地基。

扩展从哪发现?`discoverAndLoadExtensions` 按固定顺序扫三处:**项目本地 `./.pi/extensions/`**、**全局 `~/.pi/extensions/`**、以及配置里**显式指定的路径**。目录里 `*.ts`/`*.js` 直接当扩展;子目录有 `index.ts` 或带 `pi` 字段的 `package.json` 也能被识别成一个扩展包。「往目录里丢个文件就生效」的体验,就是这套发现规则给的。

## 13.3 `ExtensionAPI`:注册方法 vs 动作方法

工厂函数拿到的 `pi: ExtensionAPI`,是扩展能做的一切的总表面。`createExtensionAPI`(`loader.ts`)把它的方法分成泾渭分明的两类。

**第一类:注册方法(registration)——把能力「登记」到扩展对象上。** 它们在加载时调用,往扩展的几张 Map 里写东西:

```ts
on(event, handler)            // 注册事件处理器 → extension.handlers
registerTool(tool)            // 注册一个工具 → extension.tools
registerCommand(name, opts)   // 注册一个 slash 命令 → extension.commands
registerShortcut(key, opts)   // 注册一个键盘快捷键 → extension.shortcuts
registerFlag(name, opts)      // 注册一个 CLI flag → extension.flags
registerMessageRenderer(t, r) // 注册自定义消息的渲染器
registerProvider(name, config)// 注册一个模型 provider
```

注意 `registerTool` 一调用就 `runtime.refreshTools()`——新工具能在运行中热插上去。这正是第 8 章 `ToolDefinition` 的归宿:**内置的 7 个工具和扩展注册的工具,走的是同一套 `ToolDefinition` 形状**,扩展工具因此和原生工具完全平权(一样能渲染、能写提示词、能被 `beforeToolCall` 拦)。

**第二类:动作方法(action)——运行时主动「操作」会话。** 它们不是登记,而是真去干事:

```ts
sendMessage(msg) / sendUserMessage(content)  // 替扩展往会话里发消息
appendEntry(customType, data)                // 往会话树 append 一个 CustomEntry(第 11 章)
setModel(model) / setThinkingLevel(level)    // 切模型、切思考强度(都是 append 节点)
setActiveTools(names) / getActiveTools()     // 动态改激活工具集
setSessionName(name) / setLabel(id, label)   // 改会话名、打书签
exec(command, args, opts)                    // 跑一条外部命令
```

这两类的实现差异很说明问题:注册方法直接写扩展自己的 Map;**动作方法全部 `runtime.xxx()` 转发给一个共享的 `ExtensionRuntime`**。为什么?因为加载扩展时,真正的会话还没就绪——`createExtensionRuntime` 给所有动作方法装的是「抛错的桩」:

```ts
const notInitialized = () => { throw new Error("Extension runtime not initialized..."); };
```

等会话准备好,`ExtensionRunner.bindCore()` 才把这些桩**替换成真实现**。于是 pi 优雅地强制了一条规矩:**加载期只能注册,不能动作**。你想在工厂函数顶层就 `pi.sendMessage(...)`?直接抛错——因为那个时刻根本没有会话可发。

## 13.4 事件墙:从 `agent_start` 到 `tool_call` 的二十多个钩子

`pi.on(event, handler)` 里的 `event`,是一个有二十多个成员的联合类型 `ExtensionEvent`。把它们按「Agent 一次运行的生命周期」铺开,就是 pi 暴露给扩展的全部可观测/可干预的点:

| 阶段 | 事件 | 能干什么 |
|---|---|---|
| 项目启动 | `project_trust` / `resources_discover` | 决定是否信任项目、贡献技能/提示词/主题路径 |
| 一轮开始前 | `before_agent_start` | 改系统提示词、往本轮注入消息 |
| 运行中 | `agent_start` / `turn_start` / `turn_end` / `agent_end` | 计时、统计、监控(`tps` 用的就是这对) |
| 消息流 | `message_start` / `message_update` / `message_end` | 观测/改写助手消息(`message_end` 可改写) |
| **工具准入** | **`tool_call`** | **拦截或改写工具调用(第 10 章的「权限」)** |
| 工具结果 | `tool_result` | 改写工具返回的内容/details/isError |
| 工具执行 | `tool_execution_start/update/end` | 观测工具执行进度 |
| 上下文 | `context` / `before_provider_request` | 发给模型前最后改写消息列表 / 改写原始请求体 |
| 用户交互 | `input` / `user_bash` | 拦截/转换用户输入、接管 `!` 命令 |
| 会话操作 | `session_before_switch/fork/compact/tree` | 在切换/分支/压缩/跳转前确认或取消 |

这面「事件墙」是 pi 全部可扩展性的来源。值得专门拆开看的是**不同事件的「结果语义」并不相同**——`ExtensionRunner` 为它们写了各自的 `emitXxx`:

- **`tool_call`(可拦截)**:`emitToolCall` 顺序过所有处理器,**任意一个返回 `{ block: true }` 就立即短路返回**——这就是第 10 章「软约束」的执行点。注释还点明:想改参数不要返回新值,而是**原地 mutate `event.input`**,后面的处理器能看到改动。
- **`message_end` / `tool_result`(可改写)**:`emitMessageEnd` / `emitToolResult` 把消息/结果**穿过处理器链**,每个处理器的输出喂给下一个,实现链式改写。
- **`context`(可改写整个列表)**:`emitContext` 让扩展在「发给模型前」拿到完整消息数组并替换它——做「注入项目规则」「脱敏」「裁剪」都在这。
- **`session_before_*`(可取消)**:返回 `{ cancel: true }` 就把这次切换/分支/压缩**叫停**——`confirm-destructive` 扩展就靠它在清空会话前弹一句确认。
- **普通事件(纯观测)**:`agent_start` 这类返回值被忽略,扩展只是「看一眼」。

> 这套「事件 + 分化的结果语义」是 pi 扩展系统的精髓:**不是所有钩子都平等**。有的只能看(观测),有的能改(转换),有的能拦(准入),有的能取消(确认)。pi 没有用一个万能的「中间件」糊弄过去,而是为每种干预强度设计了精确的契约——这让扩展作者一眼就知道「我这个钩子到底有多大权力」。

## 13.5 `ExtensionContext`:回调里那个「当前时刻」的 ctx

每个事件处理器的第二个参数 `ctx`,是 `ExtensionContext`。它和 `ExtensionAPI` 的区别要分清:**`pi`(API)是「注册期」的能力总表;`ctx`(Context)是「回调期」那一刻的运行时快照。**

`createContext`(`runner.ts`)用的全是 **getter**,而非提前算好的值:

```ts
createContext(): ExtensionContext {
	return {
		get ui() { runner.assertActive(); return runner.uiContext; },
		get model() { runner.assertActive(); return getModel(); },
		get cwd() { ... },
		isIdle: () => runner.isIdleFn(),
		compact: (options) => runner.compactFn(options),
		getContextUsage: () => runner.getContextUsageFn(),
		// ...
	};
}
```

为什么坚持用 getter?因为 ctx 必须反映「**读取这一刻**」的真实状态——当前用哪个模型、UI 在不在、会话有没有空闲。如果提前把值快照进对象,扩展拿到的就是「创建 ctx 那一刻」的陈旧值了。`ctx.ui.notify(...)`、`ctx.compact()`、`ctx.getContextUsage()`(查当前 token 用量,连回第 12 章)、`ctx.abort()`——这些都是扩展在回调里「此时此刻」操作 Agent 的把手。

还有一个细节体现 pi 的严谨:`assertActive()`。当会话被替换或重载后,旧的 ctx 会被 `invalidate` 标记为「陈旧」,之后任何对它的访问都直接抛错,并附上一长串解释「不要在 `newSession()`/`fork()` 之后继续用捕获的旧 ctx」。这是在帮扩展作者**尽早暴露「拿着过期上下文乱用」这类隐蔽 bug**。

## 13.6 一个完整的例子:用扩展实现「危险操作确认」

把前面的拼起来,看 `confirm-destructive` 这个官方示例扩展——它用 `session_before_*` 事件在破坏性操作前弹确认:

```ts
export default function (pi: ExtensionAPI) {
	pi.on("session_before_switch", async (event, ctx) => {
		if (!ctx.hasUI) return;          // 没有 UI(如 print 模式)就放行
		if (event.reason === "new") {
			const ok = await ctx.ui.confirm("Clear session?", "This will delete all messages...");
			if (!ok) { ctx.ui.notify("Clear cancelled", "info"); return { cancel: true }; }
		}
	});
	pi.on("session_before_fork", async (event, ctx) => {
		const choice = await ctx.ui.select(`Fork from entry ${event.entryId.slice(0,8)}?`, [...]);
		if (choice !== "Yes, create fork") return { cancel: true };
	});
}
```

短短二十行,集齐了所有要素:工厂函数、`pi.on` 注册、回调拿 `(event, ctx)`、用 `ctx.hasUI` 判环境、用 `ctx.ui.confirm/select` 与用户交互、返回 `{ cancel: true }` 取消操作。**这个「危险确认」能力,在很多 Agent 里是写死在核心里的弹窗逻辑;在 pi 里,它只是一个可装可卸的二十行扩展。** 这就是「约束外包」哲学在体验层的兑现——连「要不要二次确认」这种产品决策,pi 都不替你定,而是留成一个口子。

至此我们看清了 pi 自扩展系统的骨架:工厂函数注册、jiti 加载、虚拟模块共享依赖、注册/动作两类 API、二十多个分化语义的事件、getter 式的运行时 ctx。下一章我们看另一种「轻量扩展」——不写代码,只写 Markdown 的 **Skill 与 Slash Command**:如何用一个 `SKILL.md` 文件就给 Agent 注入一套领域知识。

---

## 本章小结

- 一个 pi 扩展 = 一个默认导出的工厂函数 `(pi: ExtensionAPI) => void`;工厂在加载时运行一次,职责是「注册」,真正逻辑在事件回调里(拿 `event` + 当前时刻的 `ctx`)。
- 加载用 `jiti` 在运行时直接 import TypeScript;关键是用 `virtualModules`/`alias` 把宿主自己的 `pi-ai`/`pi-tui`/`typebox` 等共享给扩展,保证「宿主与扩展同一份依赖实例」。扩展从 `./.pi/extensions/`、全局目录、显式配置路径三处发现。
- `ExtensionAPI` 分两类:注册方法(`on`/`registerTool`/`registerCommand`/`registerShortcut`/`registerFlag`...)写扩展自己的 Map;动作方法(`sendMessage`/`setModel`/`appendEntry`...)转发给共享 `ExtensionRuntime`。加载期动作方法是抛错桩,`bindCore()` 后才变真——强制「加载期只注册、不动作」。
- 二十多个事件构成「事件墙」,且结果语义分化:`tool_call` 可拦截(`{block}` 短路、mutate `event.input` 改参数)、`message_end`/`tool_result`/`context` 可链式改写、`session_before_*` 可取消、普通事件纯观测。
- `ExtensionContext` 全用 getter,反映「读取那一刻」的运行时状态(`ui`/`model`/`compact`/`getContextUsage`...);会话替换后旧 ctx 被 `invalidate`,再用即抛错,帮作者尽早暴露「用过期 ctx」的 bug。
- 「危险操作确认」这类在别的 Agent 里写死在核心的逻辑,在 pi 里只是一个二十行的可装卸扩展——「约束外包」延伸到了产品体验层。

下一章,看不写代码、只写 Markdown 的轻量扩展——Skill 与 Slash Command。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/coding-agent/src/core/extensions/loader.ts` | `createExtensionAPI`(注册/动作两类方法)、`createExtensionRuntime`(抛错桩)、jiti 加载 + `VIRTUAL_MODULES`、`discoverAndLoadExtensions` 发现规则 |
| `packages/coding-agent/src/core/extensions/runner.ts` | `ExtensionRunner`:`bindCore` 注入真实现、`createContext`(getter 式 ctx)、各 `emitXxx`(`emitToolCall` 短路、`emitContext`/`emitMessageEnd` 链式、`session_before_*` 取消) |
| `packages/coding-agent/src/core/extensions/types.ts` | `ExtensionEvent` 联合(二十多个事件)、`ExtensionAPI`/`ExtensionContext`/`ToolCallEventResult` 等接口 |
| `.pi/extensions/tps.ts` | 最小扩展示例(`agent_start`/`agent_end` 算 tokens/秒) |
| `packages/coding-agent/examples/extensions/confirm-destructive.ts` | 用 `session_before_*` + `{cancel:true}` 实现破坏性操作确认 |

## 动手实验

1. 照着 `tps.ts` 写一个最小扩展:在 `agent_end` 时用 `ctx.ui.notify` 打印「本轮发了几条消息」。把它丢进 `./.pi/extensions/`,验证「丢文件即生效」。再故意在工厂函数顶层调用 `pi.sendMessage(...)`,观察它为什么会抛 "runtime not initialized"——对照 13.3 的「注册/动作」二分解释原因。
2. 写一个 `tool_call` 扩展:拦截所有 `bash` 命令里含 `rm` 的调用并 `{ block: true, reason }`。再改成「不拦截而是 mutate `event.input.command` 给 `rm` 加 `-i`」。对照 `emitToolCall` 的源码,解释「为什么 block 会短路、而 mutate 能被后续处理器看到」。
3. 读 `createContext` 的所有 getter,列出 `ExtensionContext` 暴露了哪些「当前时刻」的能力。重点回答:为什么这些必须是 getter 而非提前算好的值?设想一个扩展捕获了旧 ctx、在 `ctx.newSession()` 之后又用它——`assertActive()` 会发生什么,它为你挡掉了什么 bug?
