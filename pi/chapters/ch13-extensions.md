# 第 13 章　扩展系统与 Pi Packages

> 核心不长大，能力却要长大——唯一的办法是把「长大」这件事交给进程内的插件。

如果说前十二章讲的是 pi「已经有什么」，从这一章开始讲 pi「允许你加什么」。

扩展（Extension）是 pi 最重要的可扩展面：它在 pi 进程内运行，拿到的权限和 pi 自己一样，能注册工具、命令、快捷键、CLI 参数、模型 provider、MCP server、渲染器，还能监听几乎每一个生命周期事件并改写其中的数据。

代价也很直接：**扩展是任意代码执行**。所以这一章除了讲「怎么写」，也会讲清楚「它什么时候被加载、能改什么、不能改什么」。

---

## 13.1 一个扩展的最小形态

扩展是一个 TypeScript 模块，默认导出一个工厂函数：

```ts
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function (pi: ExtensionAPI) {
	pi.registerCommand("hello", {
		description: "Show a greeting",
		handler: async (name, ctx) => {
			ctx.ui.notify(`Hello, ${name || "world"}!`, "info");
		},
	});
}
```

不需要编译。pi 用 jiti 直接加载 TypeScript，开发时甚至可以：

```bash
pi --extension ./hello.ts
```

扩展的放置位置有三类：

| 位置 | 路径 | 是否受项目信任约束 |
|---|---|---|
| 个人级 | `~/.pi/agent/extensions/` | 否 |
| 项目级 | `.pi/extensions/` | 是 |
| 显式加载 | `-e/--extension <file>` | 否 |

单文件扩展直接放 `.ts`；多文件扩展放一个目录，入口为 `index.ts`。依赖写在旁边的 `package.json` 里。

---

## 13.2 加载：jiti、虚拟模块与两种运行时

### 为什么不是普通 `import()`

普通 Node 无法直接执行 TypeScript（除非用 strip-types 且不涉及类型擦除以外的语法），也解析不了扩展里对 `@earendil-works/pi-*` 的裸导入——尤其是在 pi 被编译成单文件二进制、根本没有 `node_modules` 的时候。

pi 的解法是 jiti + 虚拟模块（`jiti-loader.ts` / `virtual-modules.ts`）：

```ts
export const VIRTUAL_MODULES: Record<string, unknown> = {
	typebox: bundledTypebox,
	"@earendil-works/pi-agent-core": bundledPiAgentCore,
	"@earendil-works/pi-tui": bundledPiTui,
	"@earendil-works/pi-ai": bundledPiAiCompat,
	"@earendil-works/pi-coding-agent": bundledPiCodingAgent,
	// 旧包名映射，兼容历史扩展
	"@mariozechner/pi-ai": bundledPiAiCompat,
	// ...
};
```

也就是说：扩展写 `import { Type } from "typebox"` 或 `import type { ExtensionAPI } from "@earendil-works/pi-coding-agent"` 时，拿到的不是磁盘上的包，而是 **pi 进程自己已经加载的那一份实例**。

这不只是方便。它解决了一个关键问题：**单例一致性**。如果扩展加载了第二份 `pi-ai` 或 `typebox`，就会出现两份类、两个注册表、两套初始化状态——`instanceof` 失败、类型校验行为不一致、provider 注册丢失。虚拟模块让宿主与扩展共享同一份运行时。

### 两种加载路径

`loader.ts` 会根据运行形态选择策略：

- **Node 构建**：使用 dist 别名 + `tryNative`，让 jiti 尽量走原生 ESM；
- **编译二进制**：使用 `virtualModules`，并把 `tryNative` 关掉；
- **源码开发**：启用 `tsconfigPaths`，尊重仓库内的路径映射。

同一份扩展代码，在三种形态下都能加载——这是「扩展生态」能成立的前提。

### 生命周期约束

工厂函数可以是 async，pi 会等它完成再继续启动。但文档明确要求：

> Do not start processes, sockets, watchers, or timers in the factory … Start long-lived resources from `session_start` … Close session-scoped resources from an idempotent `session_shutdown` handler.

原因很实际：**有些调用会加载扩展但不启动会话**（例如 `pi config`、只列命令）。在工厂里开定时器，就会留下没人关闭的资源和副作用。

`reload` 更严格：它替换整个扩展运行时，`await ctx.reload()` 之后的代码仍属于旧运行时，不能再碰旧运行时的会话状态。

---

## 13.3 `ExtensionAPI` 能做什么

把 API 按用途分组，比逐个罗列方法更好记：

| 能力 | API | 典型用途 |
|---|---|---|
| 监听 / 改写事件 | `pi.on(event, handler)` | 审批、注入上下文、观测 |
| 注册工具 | `pi.registerTool(tool)` | 让模型调用新能力 |
| 注册命令 | `pi.registerCommand(name, opts)` | `/deploy`、`/review` |
| 快捷键 / CLI 参数 | `registerShortcut` / `registerFlag` / `getFlag` | 交互与自动化开关 |
| 发消息 | `sendMessage` / `sendUserMessage` | 注入提示、排队 steering |
| 持久化私有状态 | `appendEntry(customType, data)` | 跨会话恢复扩展状态 |
| 会话控制 | `setModel` / `setThinkingLevel` / `setActiveTools` / `setSessionName` / `setLabel` | 动态调整运行时 |
| 模型接入 | `registerProvider` / `unregisterProvider` | 代理、私有网关、OAuth |
| MCP | `registerMcpServer` / `unregisterMcpServer` | 动态挂载工具服务器 |
| 虚拟模型路由 | `registerVirtualModel` | 按请求特征选模型（见第 5 章） |
| 渲染 | `registerToolRenderer` / `registerMessageRenderer` / `registerMarkdownTransformer` | 自定义 TUI 呈现 |
| 跨扩展通信 | `pi.events` | 扩展间事件总线 |
| 执行命令 | `pi.exec()` | 受控地跑子进程 |

几个容易混淆的点：

- **`appendEntry` 与 `sendMessage` 不是一回事**。前者写 `custom` entry，**不进模型上下文**，只用于扩展自己恢复状态；后者写 `custom_message`，会进上下文。
- **`registerTool` 的工具描述符是 `ToolDefinition`**，比核心的 `AgentTool` 多出 `promptSnippet`、`promptGuidelines`、`exposure`、`namespace`、`defaultActive`、`prepareLoadout`、渲染器等字段（第 8 章）。注册后由 `wrapToolDefinition()` 适配成核心认识的形状。
- **`setActiveTools` 只影响「声明给模型」的集合**。`codemode` / `deferred` 暴露的工具即使不在 active 集合里，仍然可以被脚本或工具搜索调用。

---

## 13.4 事件墙：六种组合方式

pi 的事件很多，但它们的**组合语义**只有六种。理解这张表，比背事件名有用得多（`docs/extensions.md`）：

| 模式 | 行为 | 代表事件 |
|---|---|---|
| 通知 | 返回值被忽略，只用于观察 | `agent_start`、`tool_execution_start`、`provider_stream_event` |
| 就地修改 | 后续 handler 看到被改过的字段 | `tool_call.input`、`before_provider_headers.headers` |
| 链式变换 | 后一个 handler 收到前一个的返回值 | `input`、`context`、`message_end`、`tool_result`、`before_provider_request` |
| 闸门 | 一旦处理/拦截，就停止默认路径 | `tool_call` 的 block、`user_bash` 的 handled |
| 可操作边界 | 看到「目前已提议的 entry 与继续决策」，可追加草稿 | `turn_end`、`agent_before_settle` |
| 最后覆盖 | 最后一个返回动作的 handler 生效 | `cache_warming_decision` |

### 一次模型请求的事件顺序

```text
context
  → context_with_system
    → before_provider_headers
      → before_provider_request
        → after_provider_response
          → provider_stream_event / message_start / message_update / message_end
            → turn_end
```

`context` 与 `context_with_system` 的分工值得单独说：

- `context` 只拿到**对话消息**，看不到系统提示词和工具声明。pi 在它返回后会恢复提示词与工具状态，所以它不能借机篡改系统消息。
- `context_with_system` 拿到**完整转录**，返回值原样使用。代价是你必须自己保证「系统消息仍在索引 0」，否则 provider 会收到非法请求。

这是典型的「默认安全、显式越权」设计：想让扩展碰系统提示词，必须选那个名字更长、语义更危险的钩子。

### 工具执行的事件顺序

```text
tool_execution_start
  → tool_call          （可改参数、可 block）
    → tool_execution_update（执行中的流式进度）
      → tool_result    （可改写结果）
        → tool_execution_end
```

注意 `tool_call` 在 `tool_execution_start` 之后、真正执行之前。这看起来只是顺序问题，实际决定了 UI 可以先画出「正在调用」，再被扩展拦下来改成失败——用户看到的是完整过程，而不是「调用凭空消失」。

### 边界事件：`turn_end` 与 `agent_before_settle`

这两个事件不是「通知」，而是**提交点**。它们的 handler 可以：

- 追加 `custom` / `custom_message` / `context_edit` / `compaction` 草稿；
- 返回 `continue: true` 请求再进行一次模型请求。

pi 会把所有 handler 的草稿累积起来，在边界处原子提交，然后用提交后的投影决定是否继续。`turn_end` 适合「这一轮做完了，补一条记录/改一条上下文」；`agent_before_settle` 适合「整个 run 即将结束，还有最后一件事」。

文档给了一条硬警告：**无条件 `continue` 会死循环**。边界事件给了扩展「再来一轮」的权力，也把「什么时候停」的责任交给了扩展。

---

## 13.5 自定义工具：从声明到执行

一个最小工具（`examples/extensions/hello.ts`）：

```ts
const helloTool = defineTool({
	name: "hello",
	label: "Hello",
	description: "A simple greeting tool",
	parameters: Type.Object({
		name: Type.String({ description: "Name to greet" }),
	}),
	async execute(_toolCallId, params, _signal, _onUpdate, _ctx) {
		return {
			content: [{ type: "text", text: `Hello, ${params.name}!` }],
			details: { greeted: params.name },
		};
	},
});

export default function (pi: ExtensionAPI) {
	pi.registerTool(helloTool);
}
```

`execute` 的五个参数是 1.3 的契约：`toolCallId`、参数、`signal`、`onUpdate`、`ExtensionToolContext`。最后一个参数让工具能访问会话能力（执行嵌套工具、读设置、发消息等），也是 codemode 与 MCP 资源工具能复用同一套钩子的基础。

几条来自官方文档的硬约束：

1. **失败就抛异常**。抛出的错误会变成 `isError: true` 的工具结果；返回对象不会自动标记为错误。
2. **要报错但仍有数据，返回 `isError: true` + `structuredContent`**。模型看到错误，脚本仍能拿到结构化数据。
3. **声明 `outputSchema` 时，`structuredContent` 必须匹配**。模型仍然读 `content`；codemode 脚本优先拿 `structuredContent`。
4. **共享可变状态就声明 `executionMode: "sequential"`**。否则同一批工具会并行执行。
5. **文件读改写必须用 `withFileMutationQueue()` 包住整个过程**。
6. **大输出要截断**，并告诉模型去哪里读完整内容。
7. **嵌套模型调用的 usage 要放进结果**，否则会话账本会漏计。

这些不是风格建议，而是「工具在真实系统里不闯祸」的最低要求。

---

## 13.6 状态、重载与信任

### 扩展状态放哪

- **`appendEntry(customType, data)`**：写进会话文件，重载后可扫描恢复；不进模型上下文。
- **`sendMessage(...)`**：写 `custom_message`，进上下文，可选择是否显示在 TUI。
- **内存变量**：只在当前运行时有效，`reload` 后消失。

判断标准很简单：**模型需要知道吗？** 需要 → `sendMessage`；只是扩展自己要记住 → `appendEntry`；两者都不需要 → 内存。

### 项目信任与扩展

项目级扩展是「克隆一个仓库就可能被执行」的典型风险，所以它被项目信任严格闸住（第 10 章）：

- 只有**个人级和命令行显式加载**的扩展，才能参与 `project_trust` 事件；
- 项目扩展只有在信任通过后才加载；
- 项目包（Pi Packages）同样如此，且包里的 skill 还可能指示模型去执行程序。

文档的措辞很直白：**扩展以 pi 进程的操作系统权限运行，能看提示词、工具调用、文件、凭证和会话历史。只加载你信任的来源。**

---

## 13.7 Pi Packages：把能力打包分发

扩展、Skill、提示词模板、主题可以作为一个整体分发，这就是 Pi Package。它本质上是一个普通目录或 npm 包：

```text
my-pi-package/
├── package.json
├── extensions/
├── skills/
├── prompts/
└── themes/
```

没有 `pi` 清单时，pi 按约定目录发现资源；需要精确控制时，用清单：

```json
{
  "name": "my-pi-package",
  "keywords": ["pi-package"],
  "pi": {
    "extensions": ["./src/extension.ts"],
    "skills": ["./resources/skills"],
    "prompts": ["./resources/prompts/*.md"],
    "themes": ["./resources/themes/*.json"]
  }
}
```

### 三种来源

| 来源 | 示例 | 行为 |
|---|---|---|
| npm | `pi install npm:@example/pi-tools@1.0.0` | 安装到 pi 的 npm 目录，版本固定 |
| git | `pi install git:github.com/example/pi-tools@v1` | 克隆并对齐到指定 ref，ref 不随更新移动 |
| 本地 | `pi install ./local-package` | 直接按路径加载，不复制 |

个人安装写入 `~/.pi/agent/settings.json`；加 `--local` 写入 `.pi/settings.json`，后者要等项目信任通过才读取。

### 依赖规则：宿主提供的必须用 peerDependencies

pi 向扩展提供 `pi-ai`、`pi-agent-core`、`pi-coding-agent`、`pi-tui`、`typebox`。包作者必须把它们列在 `peerDependencies`（`"*"`），**不能列进 `dependencies`，也不能打包**。

原因还是单例：如果包里带了一份物理副本，在编译 ESM 下会绕过模块映射，产生重复的类与注册表。pi 检测到这种 manifest 会直接告警。

其他 Pi 包作为依赖时，必须真的打进发布产物，并通过各自的 `node_modules` 路径引用；不要假设两个包共享同一个依赖实例。

### 资源筛选

settings 里的对象形式可以只加载包的一部分：

```json
{
  "packages": [{
    "source": "npm:@example/pi-tools",
    "extensions": ["extensions/*.ts", "!extensions/legacy.ts"],
    "skills": [],
    "prompts": ["prompts/review.md"]
  }]
}
```

语义：省略 = 全加载；`[]` = 全不加载；`!pattern` = 排除；`+path` / `-path` = 精确包含/排除。筛选只能收窄包自己声明的范围，不能凭空扩大。

---

## 13.8 一个完整的观察型扩展

`examples/extensions/status-line.ts` 展示了「不改任何数据，只观察」的写法：

```ts
export default function (pi: ExtensionAPI) {
	let turnCount = 0;

	pi.on("session_start", async (_event, ctx) => {
		ctx.ui.setStatus("status-demo", ctx.ui.theme.fg("dim", "Ready"));
	});

	pi.on("turn_start", async (_event, ctx) => {
		turnCount++;
		ctx.ui.setStatus("status-demo", ctx.ui.theme.fg("accent", `● Turn ${turnCount}...`));
	});

	pi.on("turn_end", async (_event, ctx) => {
		ctx.ui.setStatus("status-demo", ctx.ui.theme.fg("success", `✓ Turn ${turnCount} complete`));
	});
}
```

它揭示了扩展系统的两个性格：

- **UI 是扩展的一等公民**。`ctx.ui` 提供状态栏、通知、选择框、编辑器等能力，扩展不是只能躲在后台。
- **观察型扩展很便宜**。不返回结果、不改数据，就不会影响主流程；这正是「事件墙」把通知与变换分开的价值。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/coding-agent/src/core/extensions/types.ts` | `ExtensionAPI`、全部事件与结果类型、`ToolDefinition` |
| `packages/coding-agent/src/core/extensions/loader.ts` | 发现、加载、jiti 配置、模块解析 |
| `packages/coding-agent/src/core/extensions/jiti-loader.ts` / `jiti-static-loader.ts` | Node 构建与编译二进制两种加载路径 |
| `packages/coding-agent/src/core/extensions/virtual-modules.ts` | 宿主模块映射与旧包名兼容 |
| `packages/coding-agent/src/core/extensions/runner.ts` | 事件分发、组合语义、工具注册、边界的草稿提交 |
| `packages/coding-agent/docs/extensions.md` | 官方扩展指南（事件顺序、组合模式、工具约束） |
| `packages/coding-agent/docs/packages.md` | Pi Packages 的清单、来源、依赖与筛选 |
| `packages/coding-agent/examples/extensions/` | 可直接运行的真实样例 |

## 动手实验

1. 写一个 `turn_end` 扩展：每轮结束时用 `appendEntry` 记录本轮工具调用次数，并注册一个 `/stats` 命令读出来。验证它**没有**进入模型上下文。
2. 把同一个扩展改成在 `turn_end` 返回 `continue: true`，观察循环行为；再加一个「最多继续一次」的计数器，体会文档为什么警告无条件继续。
3. 写一个扩展，在 `context` 里尝试修改系统提示词，确认无效；再改用 `context_with_system`，并保证系统消息仍在索引 0。解释这两者的安全差异。
