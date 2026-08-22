# 第 1 章　从 Prompt 到 Harness：插件化 Agent 的工程范式

在拆开任何一个零件之前，先回答一个问题：为什么同一个 LLM 套上不同的工程外壳，会变成性格迥异的 Agent？这一章定义"Harness"这个概念，说明 DeepSeek Harness 为什么值得当教材，以及本书的方法论。读完后，你应该能在脑中画出 dsh 的设计哲学三角：一切皆插件、注册即 effect、模型可见即日志。

## Agent = Model + Harness

当一个 LLM 被包裹上一层工程系统后，它就不再是"聊天机器人"，而是"Agent"。这层工程系统决定了模型能调什么工具、看到什么上下文、在什么边界上被打断、出错后怎么恢复、副作用能不能落地——这就是 **Harness（驾驭层）**。

DeepSeek Harness（`dsh`）是一个特别有研究价值的标本，因为它的定位很极端：**整个产品就是一层 Harness，而且这层 Harness 本身又被拆成了无数可替换的插件。** 它的 `README.zh.md` 第一句就说：

> DeepSeek Harness（`dsh`）是由 DeepSeek AI 开发的开源 agent harness。它采用**一切皆插件**的架构，并由 Cordis 驱动。

注意这里的措辞：不是"基于 Cordis 构建"，而是"**由 Cordis 驱动**"（driven by Cordis）。Cordis 不是被调用的一个库，而是整个系统的血管和骨架——`vendor/` 目录里直接塞了一份 Cordis 的源码拷贝（pinned source copy，`vendor/cordis/` 下有完整的 `src/`、`lib/`、`package.json`，`vendor/README.md` 记录了 manifest 与上游 SHA）。dsh 团队宁可 vendoring 也要把这个框架焊死在自己手里，可见它对这一层有多依赖。

## "一切皆插件"：为什么 DeepSeek 敢把主循环也做成插件

绝大多数 Agent 框架都有一个"核心"：主循环、工具注册表、状态存储是特权代码，插件只能挂在边缘。dsh 反着来——**每一块都是插件**。`docs/architecture.md` 的原文：

> Cordis is the framework under dsh: plugins contribute services, typed events, and reversible effects to a shared context. Every part of the product is a plugin, including the model adapter, the tool registry, the session log, and the agent loop itself, so every part is replaceable from configuration.

这句话值得逐字拆解：

- **模型适配器是插件** —— `ctx.llm` 上的 adapter 注册，换模型就是换插件。
- **工具注册表是插件** —— `ctx.tools` 本身是一个 Cordis 服务（`packages/core/tools` 导出的 `ToolRegistryService` 继承自 `Service`），注册工具就是往这个服务上挂条目。
- **会话日志是插件** —— `ctx.sessions` 是 append-only 事件日志服务（`packages/core/session`）。
- **连 Agent 主循环本身都是插件** —— `ctx.agentLoop` 是 `agent-loop` 包提供的默认驱动实现；`Agent` 接口（`packages/core/agent/src/types.ts`）是公共契约，而 `ReactLoopAgent`（`packages/core/agent-loop/src/agent.ts`）只是它的"默认产品循环"。

"没有特权核心"的直接推论是：**你扩展 dsh 的方式不是在某个固定的扩展点上写代码，而是在别的插件旁边再挂一个插件。** 而因为 Cordis 的注册都是可逆 effect（`ctx.effect()` 返回 disposer，卸载时按注册逆序回卷），插件卸载时它会自动回卷——热更新、按 profile 组合、用户 patch 覆盖，全都建立在这个机制上。

`docs/architecture.md` 把这个设计总结为一句话：

> There is no privileged core to patch: you extend dsh by mounting a plugin beside the others, and registrations are effects that unwind when their plugin unloads.

## Harness 的六个核心职责在 dsh 中的映射

| 职责 | 定义 | DeepSeek Harness 实现 |
|---|---|---|
| **上下文** | 决定模型看到什么 | `ctx.systemPrompt` 的 prompt sections 按序拼接 + `agent.inject()` 注入 + 从 session 日志 `deriveMessages()` 派生历史 |
| **工具** | 模型能调什么 | `ctx.tools` 作用域工具注册表 + `ToolDefinition` + 六段守卫管线 |
| **编排** | 单轮循环如何串成任务 | `ctx.agentLoop`（`ReactLoopAgent`）+ turn/step 边界 + `agent/*` 事件 |
| **状态** | 出错后怎么恢复 | `ctx.sessions` append-only 事件日志 + `ctx.sessionPersistence`（JSONL/SQLite）+ 崩溃恢复 |
| **观测** | 发生了什么可查 | `session/event` 广播 + `session-query` + `session-projection` + telemetry |
| **约束** | 在什么边界上被打断 | `ctx.approval` + `ctx.sandbox` + permission presets + 工具守卫 |

一个有意思的差异：很多 Agent 框架把"状态"理解为快照（SQLite 存 session 状态），而 dsh 把"状态"理解为**事件日志**——模型可见的一切都必须能从日志重建。`packages/core/session/src/types.ts` 里的 `SESSION_FORMAT_VERSION = 0` 这个常量（注释写着"While the harness is unreleased it is pinned at `0`: no compatibility is implied"）就是这条设计哲学的直接体现：日志格式说换就换，因为日志本身就是真相源。

## 三条工程红线

dsh 的 `AGENTS.md` 写了 40+ 条开发约定，但有三条决定了整个系统的形态：

**红线一：一切皆插件，插件不是循环的补丁，循环本身是插件。** `AGENTS.md` 原文："Plugins, not loop changes: new behavior goes on documented extension points; changing `agent-loop` requires updating docs/architecture.md." 新行为必须挂到文档化的扩展点上；要改主循环，必须先改架构文档。

**红线二：Model-visible ⟺ logged（模型可见 = 已记日志）。** `AGENTS.md` 原文："Model-visible ⟺ logged: anything that reaches a model request must be reconstructable from the session log; a new model-visible input requires a session event." 任何到达模型请求的东西必须能从会话日志重建。反过来：一个新的模型可见输入，必须先新增一个 session event。

**红线三：Registrations are effects（注册即副作用）。** `AGENTS.md` 原文："Registrations are effects: every contribution goes through `ctx.effect()` / `ctx.on()`; a registry's `register()` returns the disposer." 每一个贡献都必须走 `ctx.effect()` 或 `ctx.on()`，注册返回 disposer——卸载插件时全部自动回卷。

## 本书方法论：以真实代码为唯一真理来源

本书所有技术论断都来自对 DeepSeek Harness 公开仓库源码的**实际阅读**，引用确切的文件名、函数名、常量与字面量值。dsh 仓库自带一套异常高质量的文档（`docs/architecture.md`、`docs/subsystems/*.md`、生成的 `config-catalog` / `tool-catalog` / `persistence-catalog`），但本书坚持一条铁律：**文档只作索引，论断以源码为准**。

`docs/` 里的类型声明标着 `type-equiv`，由 `pnpm run verify-type-equiv` 漂移检查保证与源码一致——这意味着读 dsh 的文档几乎等于读源码。`scripts/` 下的生成器（`gen-cordis-catalog.ts` 等）保证文档与源码永不脱节。这是它比大多数开源项目更适合当教材的原因。

建议 clone 仓库边读边对照：

```bash
git clone https://github.com/deepseek-ai/deepseek-harness
cd deepseek-harness
pnpm install
```

## 架构速览

```
┌─────────────────────────────────────────────────┐
│                  dsh 产品                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │  bundle  │  │  bundle  │  │  bundle  │       │
│  │  (base)  │  │(web-app) │  │(headless)│       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
│       │              │              │             │
│  ┌────▼──────────────▼──────────────▼─────────┐  │
│  │          Cordis 插件框架 (vendored)          │  │
│  │  Context → Service → Events → Fiber → Effect│  │
│  └────────────────────────────────────────────┘  │
│                                                   │
│  核心 ctx keys:                                   │
│  ctx.sessions │ ctx.tools │ ctx.llm              │
│  ctx.agents   │ ctx.agentLoop │ ctx.systemPrompt │
│  ctx.approval │ ctx.sandbox │ ctx.sessionPersistence │
└─────────────────────────────────────────────────┘
```

## 关键设计决策

1. **Vendoring Cordis 而非依赖**：`vendor/cordis/` 是一份 pinned source copy，团队宁可维护 fork 也要掌控框架层。这保证了 Cordis 的 API 表面与 dsh 的需求完全对齐，代价是同步上游时需要手动 merge。

2. **`SESSION_FORMAT_VERSION = 0`**：预览阶段的日志格式版本号钉死在 0，不承诺兼容。这释放了重构自由——磁盘格式说换就换，SQLite schema 用单调递增的 `SCHEMA_VERSION` 管理。只有全新项目才配拥有这种自由。

3. **没有特权核心**：主循环、工具注册表、会话日志、模型适配器全部是 Cordis 插件。扩展 = 在旁边挂插件，不是打补丁。这把"架构漂移"变成了一个文档门禁——改主循环必须先改架构文档。

## 本章小结

- DeepSeek Harness 的核心设计哲学是**一切皆插件，无特权核心**，被 Cordis 的可逆 effect 机制支撑。
- 三条工程红线（插件优先、模型可见即日志、注册即 effect）决定了系统的一切形态。
- Harness 的六个核心职责（上下文、工具、编排、状态、观测、约束）全部映射到 `ctx.*` 服务上。
- `vendor/` 里的 Cordis 不是可选依赖，而是系统的骨架——"由 Cordis 驱动"是字面意思。
- 本书以源码为唯一真理来源，文档只作索引。

## 阅读思考题

1. dsh 把主循环也做成插件（`ctx.agentLoop`），而大多数框架把它当特权核心。这种设计的代价是什么？在什么场景下你会选择不这样做？
2. "Model-visible ⟺ logged"这条不变量意味着"给模型加一种新输入"的唯一正确做法是扩展 `SessionEventMap`。如果有一个需求是"让模型看到实时文件变更通知"，你会怎么设计这个 session event？
3. Cordis 的注册都是可逆 effect——卸载时自动回卷。如果一个插件注册了一个 waterfall 监听器，卸载时回卷会怎样影响正在排队的 waterfall 调用？

> **下一章预告**：第 2 章将画出 dsh 的全景地图——54 个包的 monorepo 疆域、五组包的职责划分、以及三条工程红线如何在代码中落地。
# 第 2 章　DeepSeek Harness 全景：一个插件树的疆域

在拆开任何一个零件之前，先看清整台机器的疆域。这一章给你一张地图——读完它，你应该能在脑中标出"如果我要找主循环/工具定义/沙箱策略，该去哪个目录"。我们从 monorepo 的顶层结构开始，一路细到 `packages/` 的五个分组。

## 项目定位与设计哲学

DeepSeek Harness 用一句话定义自己：它是一个由 Cordis 驱动的、一切皆插件的开源 agent harness。目前处于**开发者预览**阶段（`0.1.0-rc.5`），官方明确警告"未来将出现破坏兼容性的变更"。`AGENTS.md` 开篇就写：

> **Pre-release stance: foundation over blast radius.** With no external consumers, prefer the correct foundation over compatibility shims: rename or repackage freely and update every reference together.

翻译过来：**没有外部消费者，所以选"正确的基础"而不是"兼容的权宜"**。想改名就改名，想重构就重构，磁盘格式说换就换。这种只有全新项目才配拥有的自由，解释了为什么 dsh 的代码如此"干净"——它没有历史包袱。

**为什么是 TypeScript？** 因为这是一个编排系统——核心是 prompts、tools、protocols、integrations，而不是重计算。整个 monorepo 用 `pnpm@11.7.0` 管理，`packages/` 下 54 个顶层包，TypeScript 源码约 20 万行。

## Monorepo 全景：疆域地图

先看**顶层目录**，这是全书的"总索引"：

| 顶层目录 | 一句话职责 | 对应 Harness 职责 / 后续章节 |
|---|---|---|
| `vendor/` | vendored Cordis 源码拷贝（`vendor/cordis/`、`vendor/cosmokit/` 等） | 第 3 章 |
| `packages/` | `@deepseek-ai/dsh-<pkg>` 工作区：core / llm / shell / fs / sandbox / subagent / workflow / bundle / … | 贯穿全书 |
| `apps/` | 两个应用：`cli`（`dsh` 命令入口 `apps/cli/src/bin.ts`）+ `web`（Vite 前端） | 第 4、17 章 |
| `native/` | `@deepseek-ai/node-addon-landlock-run`：Linux Landlock 的 node addon | 第 11 章 |
| `python/` | Python SDK 与 bundled runtime | 附录 |
| `examples/` | 可运行的 `cordis.yml` 叶子：acp-agent / headless-agent / jsonrpc-agent 等 | 第 4 章 |
| `docs/` | 架构文档 + 逐子系统参考 + 生成的 catalog + postmortem | 全书索引 |
| `scripts/` | 仓库门禁（gates）与生成器（`run-gates.ts`、`gen-cordis-catalog.ts` 等） | 第 18 章 |
| `.agents/` | Agent 工作流与 Agent Notes（`notes/`，含 implemented/proposed 分类） | 第 18 章 |

## packages/ 的五个组

`packages/` 下的 54 个包按职责分为五组：

**第一组：core（产品 API 脊柱）**——六个包构成整个系统的骨架：
- `core/session`（`ctx.sessions`）：append-only `SessionEvent` 日志与内存 store
- `core/system-prompt`（`ctx.systemPrompt`）：prompt-section 与 tool-schema 组装
- `core/tools`（`ctx.tools`）：作用域工具注册表与守卫执行管线
- `core/agent`（`ctx.agents`）：`Agent` 接口、live registry、`agent/*` 事件
- `core/agent-loop`（`ctx.agentLoop`）：实现该接口的默认驱动（`ReactLoopAgent`）
- `core/scope`：per-agent scoped-registration 原语（**无 ctx key**，是零依赖库）

注意 `core/scope` 是唯一不是服务的包——它坐在模块图最底层，`session/` 和 `system-prompt/` 都要消费它但不能反向依赖它。`docs/module-graph.md` 说它"sits below session and system-prompt precisely so they can consume it without a cycle"。

**第二组：能力 seam 组**——每个 seam 拆成 Definition + Provider + Consumer 三包：
- `fs/`（`fs`/`fs-local`/`tool-fs`/`fs-sandbox`/`fs-observation-policy`）
- `shell/`（`shell`/`bash-local`/`bash-sandbox`/`tool-bash`/`pwsh-local`/`tool-pwsh`）
- `sandbox/`（`sandbox`/`sandbox-local`）
- `subagent/`（9 个包，含 in-process / fork / ACP / Codex / Claude Code / dsh-sdk）
- `workflow/`（4 个包）
- `code-runtime/`、`lsp/`、`web/`、`skill/`、`spill/`、`terminal/`、`compaction/`

**第三组：交互与治理**——`interaction/`（`user-approval`/`user-questions`/`commands`/`permission-presets`）、`guard/`（`timeout-policy`/`repeat-tool-reminder`）、`plan/`、`goal/`、`schedule/`、`credentials/`、`settings/`

**第四组：会话与数据**——`session/`（`session-persistence`/`session-query`/`session-projection`/`session-telemetry`/`session-title`）、`compaction/`、`storage/`、`workspace/`

**第五组：运行时与远程**——`boot/`（`app-boot`/`cmdline`）、`bundle/`（`base`/`web-app`/`headless`）、`api/`（gateway/remotes）、`typert/`、`sdk/`、`acp/`、`hooks/`（`hook-protocol`/`hooks-claude-code`/`hooks-codex`）、`extensions/`（自修改插件）、`llm/`（`llm`/`llm-deepseek`/`llm-pi-ai`/`llm-retry`/`token-meter`）、`client/`（Web 前端模块）、`host/`（webserver）

## 架构速览

```
packages/ 54 包
├── core/ (6包) ← 产品 API 脊柱
│   ├── session          ctx.sessions
│   ├── system-prompt    ctx.systemPrompt
│   ├── tools            ctx.tools
│   ├── agent            ctx.agents
│   ├── agent-loop       ctx.agentLoop
│   └── scope            (无 key, 零依赖库)
│
├── 能力 seam 组 (20+包)
│   ├── fs/              ctx.fs
│   ├── shell/           ctx.shell
│   ├── sandbox/         ctx.sandbox
│   ├── subagent/        ctx.subagents
│   ├── workflow/        ctx.workflowEngine
│   ├── code-runtime/    ctx.codeRuntime
│   └── ...
│
├── 交互与治理 (10+包)
│   ├── interaction/     ctx.approval / ctx.userQuestions / ctx.commands
│   ├── guard/           timeout-policy / repeat-tool-reminder
│   ├── plan/            plan-mode
│   └── goal/ / schedule/ / credentials/ / settings/
│
├── 会话与数据 (8+包)
│   ├── session/         ctx.sessionPersistence / session-query / projection
│   ├── compaction/      ctx.compaction
│   └── storage/ / workspace/
│
└── 运行时与远程 (15+包)
    ├── boot/            app-boot / cmdline
    ├── bundle/          base / web-app / headless
    ├── api/             api-gateway / remotes
    ├── typert/          远程调用基础设施
    ├── llm/             ctx.llm / llm-deepseek / llm-pi-ai
    └── client/ / host/ / web/
```

## 如何高效阅读这套代码

dsh 的文档体系是它最被低估的资产。按这个顺序读最高效：

1. **`docs/architecture.md`** —— 总纲。讲清 Cordis、profiles/bundles、事件分类、turn flow、capability seams。
2. **`docs/cordis-primer.md`** —— Cordis 五概念入门（插件/上下文/注入/事件/effect）+ 四种派发模式。**读 dsh 源码前必读**。
3. **`docs/subsystems/README.md`** —— 每个子系统的索引表；每个子系统一页。
4. **生成的 catalog**：`docs/config-catalog.md`、`docs/tool-catalog.md`、`docs/persistence-catalog.md`、`docs/event-producer-consumer.md`、`docs/module-graph.md`。
5. **`AGENTS.md`** —— 开发约定全集。每条约定都解释了一个工程决策。

## 关键设计决策

1. **54 包的拆分粒度**：每个 capability seam 拆成 Definition / Provider / Consumer 三包，甚至更细（`subagent/` 有 9 个包）。代价是包间依赖图复杂，收益是每个包可以独立替换——换一个 Provider 不需要碰其他任何包。

2. **`core/scope` 是零依赖库**：它不注册任何 `ctx` key，不依赖任何其他 core 包，坐在模块图最底层。这让 `session/` 和 `system-prompt/` 可以消费 scope 机制而不形成循环依赖——"sits below session and system-prompt precisely so they can consume it without a cycle"。

3. **预览阶段的激进自由**：`SESSION_FORMAT_VERSION = 0`、`AGENTS.md` 的"foundation over blast radius"、磁盘格式说换就换。这种自由塑造了代码的干净程度——没有向后兼容的包袱，每个设计都可以是"正确的"而不是"妥协的"。

4. **文档与源码同源**：`docs/subsystems/*.md` 里的 Cordis API 段由 `scripts/gen-cordis-catalog.ts` 从源码生成，`pnpm run verify-cordis-catalog` 检查新鲜度；类型声明用 `type-equiv` 标记，`pnpm run verify-type-equiv` 做漂移检查。文档和源码永远不会脱节。

## 本章小结

- DeepSeek Harness 是一个 54 包、约 20 万行 TypeScript 的插件化 Agent 系统。
- `packages/` 按职责分五组：core 脊柱、能力 seam、交互与治理、会话与数据、运行时与远程。
- `core/scope` 是唯一无 ctx key 的零依赖库，坐在模块图最底层避免循环依赖。
- 文档体系是 dsh 最被低估的资产：生成的 catalog + type-equiv 漂移检查保证文档永不过期。
- 预览阶段的"foundation over blast radius"哲学让代码保持了惊人的干净度。

## 阅读思考题

1. dsh 把 `subagent/` 拆成 9 个包（Definition + 6 个 Provider + 3 个 Consumer），而 `shell/` 只拆了 6 个。拆分的粒度依据是什么？什么时候"一个包兼多个角色"是合理的？
2. `core/scope` 没有 `ctx` key，不是 Service，只是一个库。为什么它不能是 Service？如果把它变成 Service 会引入什么循环？
3. `docs/module-graph.md` 里说 `core/scope` "sits below session and system-prompt precisely so they can consume it without a cycle"。在 monorepo 的包依赖图中，"cycle"意味着什么？pnpm workspace 如何处理循环依赖？

> **下一章预告**：第 3 章将深入 vendored Cordis 框架的五块基石——插件、上下文、依赖注入、类型化事件、可逆 effect——这是读懂 dsh 所有源码的前提。
# 第 3 章　Cordis：vendored 插件框架的五块基石

dsh 没有自己的"框架层"——它直接把 Cordis 的源码 vendoring 进了 `vendor/cordis/`。要读懂 dsh，先读懂这五块基石：插件、上下文、依赖注入、类型化事件、可逆 effect。这一章的源码全部来自 `vendor/cordis/src/` 下的六个文件：`context.ts`、`service.ts`、`events.ts`、`fiber.ts`、`registry.ts`、`reflect.ts`。

## 插件（Plugin）与上下文（Context）

Cordis 的第一概念：**插件是一个实现了 Service 的对象**。`vendor/cordis/src/service.ts` 里的 `Service` 抽象类是所有 Cordis 服务的基类：

```ts
export abstract class Service<out T = never> {
  static readonly init: unique symbol = symbols.init
  static readonly check: unique symbol = symbols.check
  static readonly config: unique symbol = symbols.config
  static readonly invoke: unique symbol = symbols.invoke
  static readonly extend: unique symbol = symbols.extend
  static readonly tracker: unique symbol = symbols.tracker

  constructor(protected ctx: Context, name: string) {
    name ??= this.constructor['provide'] as string
    // ...
    self.ctx.reflect.provide(name, self, this[symbols.check])
    return self
  }
}
```

构造函数的关键一行：`self.ctx.reflect.provide(name, self, ...)` —— 服务在构造时就把自己注册到上下文中。子类调用 `super(ctx, 'myServiceName')`，Cordis 会把它的生命周期挂载进当前上下文。

第二概念：**上下文（Context）是服务的仓库**。`vendor/cordis/src/context.ts` 里的 `Context` 类是一个 Proxy——普通属性读取经过服务解析器，`extend()`、`isolate()`、`intercept()` 创建子上下文而不修改父上下文：

```ts
export class Context {
  static readonly effect: unique symbol = symbols.effect
  static readonly filter: unique symbol = symbols.filter
  static readonly isolate: unique symbol = symbols.isolate
  static readonly intercept: unique symbol = symbols.intercept

  constructor() {
    this[symbols.isolate] = Object.create(null)
    this[symbols.intercept] = Object.create(null)
    const self = new Proxy<this>(this, ReflectService.handler)
    this.root = self
    this.fiber = new Fiber(self, {}, Object.create(null), null, () => [])
    // ...
    return self
  }
}
```

构造函数返回 `new Proxy(this, ...)` —— 这意味着 `ctx.xxx` 的读取会经过 `ReflectService.handler` 的拦截逻辑，实现服务的懒解析和作用域隔离。一个服务从上下文认领一个稳定的 `ctx.<key>`（比如 `ctx.tools`、`ctx.llm`、`ctx.sessions`），其他插件通过 key 找服务，而不是 import 具体实现。

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

## 依赖注入：`inject` 声明取代手动启动排序

Cordis 的第三个概念：**通过 `inject` 声明服务依赖**。一个插件如果 `inject` 了所需服务，Cordis 会等这些服务存在后才激活它——**加载顺序由服务需求表达，而不是手动编排启动顺序**。

这在 dsh 的 bundle 配置里随处可见。看 `packages/bundle/headless/cordis.patch.yml`：

```yaml
- id: headless-runner
  name: '@deepseek-ai/dsh-headless'
  inject: [headlessStartup]
  config:
    task: !!js ctx.headlessStartup.task
```

`headless-runner` 声明 `inject: [headlessStartup]`，它的 `config.task` 从 `ctx.headlessStartup.task` 取——`headless-startup` 插件必须先激活，`headless-runner` 才会被激活。这就是"以服务需求表达加载顺序"的活例子。

更妙的是 `!!js` 表达式的延迟求值：配置表达式 `!!js ctx.webStartup.xxx` 只有在服务存在后才会被 Loader 解析——如果该服务不存在（比如 `dsh --profile web --help` 不提供该服务），表达式不会被求值，服务器不会绑定端口。**同一个配置文件，根据注入的服务存在与否，决定激活哪些行为。**

## 类型化事件与四种派发模式

Cordis 的第四个概念：**类型化事件**。`vendor/cordis/src/events.ts` 定义了五种派发模式：

```ts
export type DispatchMode = 'emit' | 'parallel' | 'serial' | 'bail' | 'waterfall'
```

| 模式 | 是否 await | 派发顺序 | 有返回值？ |
|---|---|---|---|
| `emit` | 否 | 监听者按注册顺序观察 | 无 |
| `waterfall` | 否 | 监听者按注册顺序观察 | 有 |
| `parallel` | 是 | 所有监听者并行观察 | 无 |
| `serial` | 是 | 监听者按注册顺序观察 | 有 |
| `bail` | 否 | 监听者按注册顺序，第一个 bail 值停止 | 有 |

`events.ts` 里每种模式都有对应的 `ctx` 方法声明。关键的是 **waterfall**——它是 dsh 拦截体系的核心原语：

```ts
/**
 * Dispatch an event whose last argument is a `next` continuation.
 *
 * Each listener wraps the rest of the chain: calling `next()` invokes the
 * next listener (finally the built-in behavior); not calling it vetoes.
 */
waterfall<K extends keyof Events>(name: K, ...args: Parameters<Events[K]>): ReturnType<Events[K]>
```

waterfall 的语义是 **around-middleware**：监听者接收 `(...args, next)`，调用 `next()` 把控制权传给链上的下一个监听者；不调用就是短路。在 dsh 里，`agent/pre-step`、`agent/request`、`llm/stream`、`tools/pre-execute`、`tools/execute`、`tools/post-execute` 全部是 waterfall。`AGENTS.md` 的铁律：

> **Waterfall listeners MUST call `next()`** to delegate; returning without it short-circuits the chain.

## 可逆 effect：注册即副作用，卸载即回卷

Cordis 的第五个概念：**注册是可逆的 effect**。`vendor/cordis/src/fiber.ts` 定义了 `Fiber`（插件运行时实例）和 effect 机制：

```ts
export type Disposable<T = any> = () => T

export type Effect<T = any> =
  | SyncEffect<T>
  | AsyncEffect<T>

type SyncEffect<T = any> =
  | Disposable<T>
  | Iterable<Disposable<T>, void, void>

type AsyncEffect<T = any> =
  | Promise<Disposable<T>>
  | AsyncIterable<Disposable<T>, void, void>
```

`ctx.effect()` 接受一个 effect body，body 返回 disposer（或多个 disposer 的可迭代）。当拥有该 effect 的 fiber 卸载时，所有 disposer 按注册逆序运行——**这就是"注册即 effect，卸载即回卷"的完整机制**。

dsh 的 `AGENTS.md` 把它上升为红线：

> **Registrations are effects**: every contribution goes through `ctx.effect()` / `ctx.on()`; a registry's `register()` returns the disposer.

这个机制支撑了 dsh 最激进的能力：**插件可以在运行时被安装、替换、卸载，而系统状态保持一致**。`packages/extensions/` 甚至让模型自己通过 `cordis_define` / `cordis_run` / `cordis_stop` 工具动态定义和运行 Cordis 包——"agent inspects/mounts its own plugins"（自修改）。这在其他 Agent 里是禁区，在 dsh 里是设计。

## 插件生命周期：mount → inject → activate → dispose

一个 Cordis 插件的生命周期：**装载（mount）→ 注入等待（inject 的服务全部就绪）→ 激活（apply）→ 运行 → 卸载（dispose）**。`packages/boot/app-boot/src/index.ts` 的 `boot()` 函数在启动失败时利用这一点做了**部分上下文回卷**：

```ts
} catch (cause) {
  await ctx.fiber.dispose()
  const detail = cause instanceof Error ? cause.message : String(cause)
  // ...
}
```

启动任何一步失败，`boot()` 都会 `ctx.fiber.dispose()` 回卷整个插件树——包括已经挂载的 surface 的终端恢复（raw mode、bracketed paste、键盘协议）。**失败路径和成功路径共享同一套清理逻辑**，这是"注册即 effect"在工程上的直接收益。

## 架构速览

```
Cordis 五块基石
┌──────────────┐
│   Plugin     │  实现 Service 的对象
├──────────────┤
│   Context    │  Proxy + 服务仓库 + 子上下文继承
├──────────────┤
│   Inject     │  声明依赖 → 按需激活，不手动排序
├──────────────┤
│   Events     │  emit/waterfall/parallel/serial/bail
├──────────────┤
│   Effect     │  ctx.effect() → disposer → 逆序回卷
└──────────────┘
     ↓
  Fiber (插件运行时实例) 管理生命周期
```

## 关键设计决策

1. **Context 是 Proxy**：`new Context()` 返回 `new Proxy(this, ReflectService.handler)`。这让 `ctx.xxx` 的读取可以经过服务解析器，实现懒解析和作用域隔离。代价是每次属性访问都有额外开销，收益是插件不需要手动管理服务查找。

2. **waterfall 是 around-middleware**：不是简单的"通知"，而是"拦截+委托"。监听者可以改写参数、短路链、包裹执行。这正是 dsh 的 `agent/pre-step`、`tools/pre-execute` 等拦截点的基础设施。

3. **Fiber 管理 effect 的生命周期**：每个插件有自己的 Fiber，Fiber 持有该插件注册的所有 disposer。卸载时按逆序回卷——这保证了"最后注册的最先清理"，避免了资源泄漏和依赖倒置。

4. **`!!js` 表达式的延迟求值**：配置文件里的 `!!js ctx.xxx.yyy` 只在服务存在后才被求值。这让同一个配置文件可以根据环境动态决定激活哪些行为，而不需要条件分支。

## 本章小结

- **插件 = 实现 Service 的对象**，Context 是 Proxy + 服务仓库，`ctx.<key>` 是产品的 API 表面。
- **`inject` 声明依赖**，加载顺序由服务需求表达，不手动编排。
- **类型化事件**五种派发模式，waterfall 是 around-middleware——dsh 的所有拦截点都建立在 waterfall 上。
- **注册是可逆 effect**，`ctx.effect()` 返回 disposer，Fiber 卸载时按逆序回卷。
- **生命周期**（mount → activate → dispose）由框架管理，失败路径共享清理逻辑。

## 阅读思考题

1. `vendor/cordis/src/context.ts` 的构造函数里 `return self` 返回的是 Proxy 而不是 `this`。这意味着 `new Context()` 的调用者拿到的是 Proxy。如果某个插件绕过 Proxy 直接访问 `this` 上的属性，会发生什么？
2. waterfall 的"短路"语义意味着不调用 `next()` 就能阻止后续监听者运行。在 dsh 的 `agent/pre-step` 里，谁是"最后一个必须调 `next()` 的监听者"？如果没有人调 `next()`，主循环会怎样？
3. `Fiber.dispose()` 按注册逆序回卷 disposer。如果一个 disposer 是异步的（比如需要 flush 数据），Fiber 会等它完成吗？这对 dsh 的崩溃恢复有什么影响？

> **下一章预告**：第 4 章将拆开 dsh 的启动组装线——从 `apps/cli/src/bin.ts` 的三种模式分发，到 `app-boot` 的 `boot()` 函数，再到 profile / bundle / patch 三层组合。
# 第 4 章　启动与组合：从空列表到完整产品

dsh 最反直觉的一点：一个功能完整的 Agent 产品，不是"启动时加载一堆模块"，而是"**从空 entry 列表开始，一层层 patch 出来的**"。这一章拆开这条组装线——从 `apps/cli/src/bin.ts` 的三种模式分发，到 `app-boot` 的 `boot()` 函数，再到 profile / bundle / patch 三层组合。

## `dsh` CLI：`apps/cli/src/bin.ts` 的按需动态导入

`dsh` 命令的入口是 `apps/cli/src/bin.ts`。它只有三个逻辑分支，按模式动态导入：

```ts
const invocation = parseDshArgs(process.argv.slice(2), readVersion())

switch (invocation.mode) {
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

源码注释说得很清楚："Dynamic imports per mode keep unrelated modes out of each dispatch path"——按模式动态导入，无关模式的代码不进派发路径。CLI 本身不解析业务 flag：**启动器只解析自己的 flag，第一个不认识的 token 之后全部交给 booted profile 里的 app 插件解析**。

## Profile / Bundle / Patch 三层组合

一个运行中的 dsh 是一棵插件树，从有序层组合而来。`docs/architecture.md` 的定义：

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

后一层覆盖前一层，同一个 id 最后写入者胜。**注意 patch 是整行替换，不是深合并**——`app-boot` 的 README 明说 "A user patch replaces the whole matched config"。这意味着每个 patch 都要完整重述它要保留的字段，代价是啰嗦，收益是**任何一行配置的最终形态都可以从 patch 链精确推导**，没有隐式合并的惊喜。

## `app-boot` 的 `boot()`：fail loud 与 partial-context 回卷

`packages/boot/app-boot/src/index.ts` 的 `boot()` 是组装线的总控：

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
    // ...
  }
}
```

关键设计点：

1. **两个失败标签**：`stage` 在 `prepare` 前后切换——prepare 失败是"host setup"问题，插件树装载失败是"plugin tree"问题，错误信息因此能准确分类。

2. **Loader 可能随 surface 一起消失**：`await ctx.get('loader')?.await()` 之后要**重新检查** `ctx.get('loader') === undefined`——因为 surface 可能在启动完成前就按用户要求退出了，此时直接返回而不是抛错。

3. **`assertEntriesActivated`**：装载 settle 后，检查每个 enabled entry 是否真的激活了——有 entry 没有 fiber（插件名解析失败）就抛错，把每个失败插件的原始 stack 拼进启动错误。**"Misconfiguration fails loud"**。

4. **fail loud 的收尾**：启动失败 = `ctx.fiber.dispose()` 回卷整个插件树 + 一行带标签的 stderr + `exit(1)`。如果 bin 拥有终端，先恢复终端（raw mode、bracketed paste）再退出，避免把用户的 shell 弄坏。

## `dsh-base` → `dsh-web-app` / `dsh-headless`：模板 profile 的解剖

**`dsh-base`**（`packages/bundle/base/cordis.patch.yml`）是每个 profile 的第一层：一个覆盖空根的 `insert`，插入模型适配器（`llm`）、会话（`session`）、工具（`tools`）、持久化、沙箱与审批策略、settings、credentials、telemetry、typert、session-title、user-questions、agent、system-prompt……还有 `timer` 和 `hmr`（Cordis 自带的定时器与热重载服务）。

**`dsh-web-app`** 在 base 之上加浏览器表面：`code-runtime`（worker-thread）、`storage` + `storage-json` + `storage-domain`、`message-feedback`、`session-log-download`、`workspace`、`session-projection-cache`，以及 web host 行。它还故意把 base 的某些行**整行重述**——比如 `system-prompt` 的 persona 换成 "You are a coding agent powered by the {{model}} model. Your working directory is {{cwd}}."。

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

**同一个 `dsh-base`，一个 bundle 变成浏览器应用，另一个 bundle 变成一次性 runner——这就是分层组合的威力。**

## 配置即代码：`!!js` 表达式与 HMR

Cordis Loader 把 `!!js` 解析为表达式节点。dsh 在 `cordis.yml` 里用它做**配置时计算**：

```yaml
- id: tools
  config:
    mode: !!js process.env.DSH_TOOLS_MODE
```

`config`（在声明的注入激活后求值）和 `disabled`（每次挂载决策时求值）支持 `!!js`，其他元数据保持字面量。`cordis.yml` 只允许 `!!js`（绝不 `!js`）。

**用户 patch 是热更新的**：`watchUserPatches` 把用户 patch 文件注册到 Cordis 的 HMR 服务，每次增删改都事务性地重新组合完整 patch 列表。一个 rejected 的读/解析/候选会让**最后一棵好树继续跑**，并广播 `hmr/config-update-failed`——配置坏了不会杀死正在运行的 Agent。

## 架构速览

```
空 entry 列表
  │
  ▼
dsh-base/cordis.patch.yml (insert → core + llm + tools + ...)
  │
  ▼
dsh-web-app/cordis.patch.yml (insert → web host + code-runtime + ...)
  │  或
  ▼
dsh-headless/cordis.patch.yml (insert → headless-startup + headless-runner)
  │
  ▼
~/.dsh/profiles/web/cordis.patch.yml (用户覆盖)
  │
  ▼
$DSH_HOME/cordis.patch.yml (home 级覆盖)
  │
  ▼
--patch overlays (命令行覆盖)
  │
  ▼
完整插件树 → boot() → assertEntriesActivated → 运行
```

## 关键设计决策

1. **整行替换而非深合并**：patch 按 id 定位并替换整行配置，不深合并。代价是啰嗦（每个 patch 要完整重述保留字段），收益是可推导性——任何一行配置的最终形态都可以从 patch 链精确推导，没有隐式合并的惊喜。

2. **`!!js` 表达式的延迟求值**：配置表达式只在注入的服务存在后才被求值。这让同一个配置文件可以根据环境动态决定激活哪些行为，而不需要条件分支。

3. **HMR 让配置变更不断服**：用户 patch 文件的增删改触发事务性重组，rejected 的变更让最后一棵好树继续跑。这让"在运行中的 Agent 旁边修改配置"成为安全操作。

4. **`--dump-config` 与 `boot()` 共享组合逻辑**：`renderConfigDump` 用 include 自己的 parser 和 patch 算法离线组合配置，结果与 `boot()` 装载的完全一致——"composition, flag derivation, and config dumps cannot drift from what boots"。

## 本章小结

- dsh 的组合机制：**产品 = 空列表 + 有序 patch 层**。
- CLI 只做模式分发（profile / plugin / dump-config），`boot()` 只做装载与审计。
- bundle 层决定"是什么产品"，用户 patch 层决定"我的定制"。
- `!!js` 表达式让配置能感知环境，HMR 让配置变更不断服。
- `dsh-base` 是所有 profile 的第一层；`dsh-web-app` 和 `dsh-headless` 在其上叠加不同的表面。

## 阅读思考题

1. `assertEntriesActivated` 在装载 settle 后检查每个 enabled entry 是否真的激活了。如果一个 entry 的插件名解析失败（比如拼写错误），它会报什么错？这种"fail loud"策略在什么场景下可能过于严格？
2. patch 是整行替换而非深合并。如果一个用户只想修改某个 entry 的一个 config 字段，他们需要怎么做？这种设计在什么规模的配置下会变得不可维护？
3. `dsh-headless` 关掉了 HMR（`hmr: disabled: true`）。为什么一次性任务模式不需要热更新？如果 headless 模式也启用 HMR，会引入什么复杂性？

> **下一章预告**：第 5 章将深入 `packages/core/session`——dsh 的 append-only 事件日志系统。它是"模型可见即日志"这条硬不变量的物理载体。
# 第 5 章　Session 事件日志：append-only 的唯一真相源

绝大多数 Agent 把"会话状态"存成快照，dsh 把它存成**事件日志**。这一章讲清楚：为什么日志是唯一真相源，模型历史为什么是日志的投影，以及"Model-visible ⟺ logged"这条硬不变量。核心源码在 `packages/core/session/src/types.ts`。

## `SessionEventMap`：事件词汇表

dsh 的核心会话模型在 `packages/core/session/src/types.ts`：一个 `Session` 是**类型化 `SessionEvent` 的 append-only 日志**。LLM 消息历史从日志**派生**，绝不单独存储。

事件类型通过 **merge-extensible map**（声明合并）组织：`SessionEventMap` 是核心词汇表，插件可以往里面加自己的事件。核心事件（源码原文）：

```ts
export interface SessionEventMap {
  'turn/start': { turn: number }
  'turn/end': { turn: number; reason: TurnEndReason }
  'step/start': { turn: number; step: number }
  'step/end': { turn: number; step: number }
  'user/message': UserMessage
  'assistant/chunk': { turn: number; step: number; chunk: StreamChunk }
  'assistant/message': { turn: number; step: number; message: AssistantMessage; usage?: TokenUsage; interrupted?: true }
  'tool/call': { turn: number; step: number; callId: CallId; name: string; arguments: string }
  'tool/result': { turn: number; step: number; message: ToolResultMessage; error?: {...}; meta?: JsonValue }
  'request/header': { header: EpochHeader; reason: RequestHeaderReason }
  'request/context': RequestContext
  'session/end-seed': Record<string, never>
}
```

几个值得注意的细节：

- **`assistant/chunk` 是原始流块**——token 级重放保真度。`assistant/chunk` 事件在流式传输过程中逐个记入日志，`BlockAssembler` 边流边组装。源码注释："raw stream chunk — token-level replay fidelity"。

- **`tool/call` 存的是模型原始输出的未解析 JSON 字符串**（`arguments: string`）——源码注释："the raw `arguments` JSON string exactly as the model produced it (unparsed)"。不丢模型的原话。

- **`assistant/message` 携带 usage**——源码注释："the model output and its accounting travel together (there is no separate usage record)"。没有独立的 usage 记录，避免两处记账漂移。

- **`assistant/message` 的 `interrupted: true`**——源码注释："A turn cancelled mid-stream finalizes its delivered text/reasoning prefix as this event with `interrupted: true`; undispatched tool calls are absent." 这是崩溃恢复的关键标记。

- **`request/header` 是 log-only**——为下一次请求准备的完整请求头（系统提示词等），只记最新快照，不参与历史派生。

## "Model-visible ⟺ logged"：为什么新输入必须先有新事件

dsh 最硬的一条不变量，`AGENTS.md` 原文：

> **Model-visible ⟺ logged**: anything that reaches a model request must be reconstructable from the session log; a new model-visible input requires a session event.

推论链是这样的：

1. 模型历史 `deriveMessages()` 从日志派生；
2. 所以如果某条输入没进日志，它永远不会出现在模型历史里；
3. 所以"给模型加一种新输入"（比如 cron 通知、文件变更提示、skill 目录）的唯一正确做法是**扩展 `SessionEventMap` 并定义渲染**——而不是在别处偷偷塞消息。

这个设计有一个美妙的副产品：**fork、resume、transcript、telemetry、持久化全部从同一条事件流派生**。

## `deriveMessages()`：模型历史是日志的投影

模型历史不是一份独立保存的聊天记录，而是**每次请求时从日志重新投影**出来的。`packages/core/session` 的 `deriveMessages()` 把日志里的 `user/message`、`assistant/message`、`tool/result` 等按顺序投影成模型请求的消息数组。

这样做的好处：
- **无冗余**：日志是唯一存储，模型历史是计算产物；
- **可重放**：任何时刻的历史都可以从日志精确重建；
- **可恢复**：崩溃后从持久化日志重放，就能恢复到崩溃前的完整上下文。

代价：每次请求都要重新投影（性能换正确性）。dsh 用 `request/header` 等 log-only 快照事件做部分缓存——请求头只记最新值，避免每次重算。

## Turn / Step / TurnEndReason：执行封闭

dsh 的时间结构分三层：**session → turn → step**。`docs/architecture.md` 的精确定义：

> A **step** is one model request plus the tools it calls. A **turn** is zero or more steps: it opens before its first input is claimed and closes once nothing is owed.

`TurnEndReason` 通过 merge-extensible map 定义（`packages/core/session/src/types.ts`）：

```ts
export interface TurnEndReasonMap {
  'completed': { kind: 'completed' }
  'max-tokens': { kind: 'max-tokens' }
  'blocked': { kind: 'blocked' }
  'aborted': { kind: 'aborted'; reason: AgentCancelCause }
  'error': { kind: 'error'; error: LlmFailure }
}

export type TurnEndReason = TurnEndReasonMap[keyof TurnEndReasonMap]
```

关键语义：

- **每个 session event 都被 turn 封闭**——`user/message` 一定出现在某个 turn 里。
- **turn 可以花零个 step**：pre-step 拒绝或输入为空时，turn 仍然打开又关闭，日志记录这次"尝试"。
- **`interrupted` 是唯一一个主循环永远不会自己发出的终止原因**——它只在持久化崩溃恢复路径出现（第 13 章详述）。

## `SESSION_FORMAT_VERSION`：日志格式的版本管理

`packages/core/session/src/types.ts` 里的这个常量值得单独讲：

```ts
/**
 * The on-disk session format version, stamped into every newly-written
 * {@link SessionHeader} and enforced by every persistence backend on load.
 * While the harness is unreleased it is pinned at `0`: no compatibility is
 * implied, incompatible logs are rejected, and no migration is provided.
 */
export const SESSION_FORMAT_VERSION = 0
```

注释说得很清楚：**版本 0 = 不承诺兼容**。这个常量是一个单调递增的整数，没有 major/minor 拆分。是否需要 bump 由**写入端**决定，不由更新的读取端能接受什么决定。只有结构性变更才达到 bump 的门槛：header 形状、`SessionEvent` envelope、核心事件语义、或 surface 机制（`SurfaceEventType` 集合和 `SurfaceOp` 变体）。添加普通事件类型不需要 bump——per-event 的 `SessionEvent.ignorable` guard 覆盖了词汇增长。

## 事件三域分类

`docs/architecture.md` 把事件分为三个域：

- **Session events**（`turn/*`、`step/*`、`user/message`、`assistant/*`、`tool/*`）——持久的可重建事实，通过 `session/event` 广播；
- **Agent events**（`agent/*`）——携带 live `Agent`：inbox、step、status、request、validation、continuation——观察或拦截进行中的工作；
- **Capability events**——把 policy 和 adapter 挂到 seam 上（`fs/*`、`tools/*`、`telemetry/*`），不 import 主循环。

## 架构速览

```
Session (append-only 日志)
├── turn/start { turn }
│   ├── step/start { turn, step }
│   │   ├── user/message { content, source }
│   │   ├── assistant/chunk { turn, step, chunk }  ← token 级
│   │   ├── assistant/chunk { ... }
│   │   ├── assistant/message { turn, step, message, usage }  ← 组装后
│   │   ├── tool/call { turn, step, callId, name, arguments }
│   │   └── tool/result { turn, step, message, error?, meta? }
│   └── step/end { turn, step }
├── request/header { header, reason }  ← log-only 快照
├── request/context { provider, model, contextWindow }  ← log-only
└── turn/end { turn, reason }

deriveMessages() → 从日志投影出 Message[]
```

## 关键设计决策

1. **日志是唯一真相源，模型历史是投影**：`deriveMessages()` 每次请求从日志重新投影。代价是性能，收益是无冗余、可重放、可恢复。fork、resume、transcript、telemetry 全部从同一流派生。

2. **`tool/call` 存未解析的 JSON 字符串**：`arguments: string` 保留模型原话，不预解析。这让重放能精确还原模型的原始输出，即使 JSON 格式有微妙差异。

3. **`SESSION_FORMAT_VERSION = 0`**：预览阶段不承诺兼容。只有结构性变更才 bump 版本号。这让团队可以自由重构日志格式，代价是用户升级时可能需要删除旧日志。

4. **`interrupted` 是人造的 TurnEndReason**：主循环永远不会自己发出它，只有持久化崩溃恢复路径会合成它。这让重放能精确区分"真实结束"和"崩溃恢复"。

## 本章小结

- Session 事件日志是 dsh 的心脏：append-only 的类型化事件日志，模型历史是它的投影。
- **硬不变量**：Model-visible ⟺ logged——新输入必须先有新事件，否则模型永远看不到。
- `deriveMessages()` 每次请求从日志重新投影模型历史，无冗余、可重放、可恢复。
- `SESSION_FORMAT_VERSION = 0`：预览阶段不承诺兼容，只有结构性变更才 bump。
- 三域事件：session events（持久）/ agent events（live）/ capability events（seam 政策）。

## 阅读思考题

1. `deriveMessages()` 每次请求都从日志重新投影。如果一个 session 有 10000 个事件，这个投影的性能瓶颈在哪里？dsh 用什么机制缓解？
2. `tool/call` 存的是未解析的 JSON 字符串，而不是解析后的对象。如果模型输出了一个格式不正确的 JSON，工具执行会怎样？这种设计是"宽容"还是"严格"？
3. `request/header` 是 log-only 快照，不参与历史派生。为什么请求头需要记入日志？如果不记，崩溃恢复后会发生什么？

> **下一章预告**：第 6 章将拆开 `packages/core/agent-loop/src/agent.ts` 里的 `ReactLoopAgent`——dsh 的默认心脏。它是 `Agent` 公共契约的唯一默认实现，但记住，它只是"默认"。
# 第 6 章　Agent 主循环：ReactLoopAgent 是如何跳动的

这一章拆开 dsh 的心脏：`packages/core/agent-loop/src/agent.ts` 里的 `ReactLoopAgent`。它是 `Agent` 公共契约的唯一默认实现——但记住，它只是"默认"，换掉它不需要改任何消费者代码。读完后，你应该能画出它的三态相位机、理解四个投递方法的精确分工、以及 `agent/*` 事件族的拦截语义。

## `Agent` 接口：send / followup / steer / inject / cancel / whenIdle

`Agent` 是每个插件（UI、hooks、orchestrators）编程对向的表面。`ctx.agents.get(id)` 返回它；具体实现是 `agent-loop` 包的内部细节，**loop 之外的任何代码都不依赖它**。

`packages/core/agent-loop/src/agent.ts` 里 `ReactLoopAgent` 的构造函数展示了它如何初始化：

```ts
constructor(
  private loopCtx: Context,
  public readonly id: SessionId,
  public readonly options: AgentOptions,
  public readonly session: Session,
) {
  this.dispatch = agentEvents(loopCtx, this)
  this.inbox = new Inbox(session, {
    inserted: (message) => { this.dispatch.emit('agent/inbox/inserted', { message }) },
    discarded: (message) => { this.dispatch.emit('agent/inbox/discarded', { message }) },
    claimed: (message, turn) => { this.dispatch.emit('agent/inbox/claimed', { message, turn }) },
  })
  const lastTurn = session.events.findLast(event => event.type === 'turn/start')?.data.turn ?? 0
  this.phase = { kind: 'idle', lastTurn }
  this.scope = createScope(loopCtx, this)
  this.ctx = this.scope.ctx.extend({ agent: this })
  this.runtimeContext = new RuntimeContextProjection(this.ctx, session)
}
```

注意几个关键点：
- **`dispatch` 在构造时一次性构建**——源码注释："Fused dispatcher, built once in the constructor so hot-path dispatches never allocate." 热路径零分配。
- **`Inbox` 的回调**：inserted / discarded / claimed 三个事件，让外部观察者可以追踪消息的生命周期。
- **`createScope(loopCtx, this)`**：agent 拥有自己的 scoped registration boundary，卸载时回卷。

四个投递方法的分工非常精确：

```ts
send(message: UserMessage, target: InboxTarget, wakeup: boolean): void {
  const wakingAfterAbort = wakeup && this.phase.kind !== 'idle' && this.phase.abort.signal.aborted
  const resolvedTarget = wakingAfterAbort ? 'next-turn' : target
  this.inbox.splice(resolvedTarget, Infinity, 0, [message])
  if (wakeup) this.wakeDriver(wakingAfterAbort)
}

followup(input: UserMessage): void {
  this.send(input, 'next-turn', true)
}

steer(input: UserMessage): void {
  this.send(input, 'next-step', true)
}

inject(input: UserMessage): void {
  this.send(input, 'next-step', false)
}
```

- **`send`** —— 统一入口：目标（next-turn 或 next-step）+ 是否唤醒。唤醒输入在活动取消后提交会被排队到下一 turn（`wakingAfterAbort` 逻辑）。
- **`followup`** —— 排队一个普通后续 turn 并唤醒驱动。
- **`steer`** —— 为最近的 step 提交转向：idle 驱动开一个 turn；运行中的驱动在下一个 step 边界消费。
- **`inject`** —— 排队模型可见上下文给下一个 pre-step，**不唤醒**驱动。idle 驱动会一直挂起直到别的消息唤醒。

## Inbox 与 wakeup：输入如何到达驱动

输入通过**一个 inbox** 到达驱动。`docs/architecture.md` 说：

> Input reaches the driver through one inbox. Some messages wake it immediately; injected context waits in the inbox until another message does.

`send` 方法里的 `wakingAfterAbort` 逻辑值得细看：唤醒输入在取消后提交时，target 被强制改为 `'next-turn'`——它不能加入已中止的活动，必须排队等下一 turn。这是"取消纪律"的一部分。

## 相位机：idle / maintenance / running

`ReactLoopAgent` 用 `Phase` 联合类型管理自己的生命周期状态：

```ts
type Phase =
  | { kind: 'idle'; lastTurn: number }
  | { kind: 'maintenance'; abort: AbortController; lastTurn: number; wakeRequested: boolean }
  | { kind: 'running'; abort: AbortController; turn: number; step: number; wakeRequested: boolean }
```

三种相位：idle / maintenance / running。`maintenance` 与 `running` 互斥——`runMaintenance` 在 turn-driving 或另一个维护任务占用时同步抛错：

```ts
runMaintenance<T>(job: (signal: AbortSignal) => Promise<T>): Promise<T> {
  if (this.phase.kind !== 'idle') throw new Error(`agent "${this.id}" already has active work`)
  // ...
}
```

`wakeDriver` 方法展示了相位转换的核心逻辑：

```ts
private wakeDriver(wakeAfterAbort = false): void {
  if (this.phase.kind !== 'idle') {
    const reason = this.phase.abort.signal.reason as AgentCancelCause | undefined
    if (reason?.kind !== 'disposed' && (this.phase.kind === 'maintenance' || wakeAfterAbort)) {
      this.phase.wakeRequested = true
    }
    return
  }
  const driver = Promise.withResolvers<void>()
  this.activityDone = driver.promise
  this.setPhase({
    kind: 'running',
    abort: new AbortController(),
    turn: this.phase.lastTurn,
    step: 0,
    wakeRequested: false,
  })
  this.loopCtx.agents.withInitiator(this, () => this.kick()).then(driver.resolve, driver.reject)
}
```

非 idle 时不直接启动新驱动，而是设置 `wakeRequested` 标志——当前驱动结束后检查这个标志，如果有待处理消息就启动新驱动。`disposed` 取消后不设置标志（teardown 不会 latch wake）。

## Turn 与 Step 边界

`turn()` 方法是主循环的核心：

```ts
private async turn(): Promise<boolean> {
  // ...
  const turn = phase.turn + 1
  this.session.append('turn/start', { turn })
  phase.turn = turn
  let turnEnds: TurnEndReason | null = null
  try {
    while (true) {
      const step = phase.step + 1
      const decision = await this.preStep(target, { turn, step })
      if (decision.kind === 'reject') {
        turnEnds = { kind: 'blocked' }
        return false
      }
      if (phase.step === 0 && decision.messages.length === 0) {
        turnEnds = { kind: 'completed' }
        return false
      }
      this.session.append('step/start', { turn, step })
      // ...
      const stepEnd = await this.step(decision.assembly)
      if (turnEnds === null || turnEnds.kind !== 'max-tokens') turnEnds = stepEnd
      this.session.append('step/end', { turn, step })
      // ...
    }
  } finally {
    this.session.append('turn/end', { turn, reason: turnEnds! })
  }
}
```

关键语义：
- **pre-step 拒绝 → turn 以 `blocked` 结束**，不开 step。
- **空首次认领 → turn 以 `completed` 结束**，花零个 step。
- **max-tokens 是 sticky 的**：一旦某个 step 达到上限，后续正常完成的 step 不会降级 turn 结果。
- **turn/end 在 finally 块里**——无论成功还是失败，turn 总是被封闭。

## `agent/*` 事件族：pre-step / request / turn-stopping 的拦截语义

`agent/*` 事件携带 live `Agent`，分两类语义：

- **waterfall（必须 `next()`）**：`agent/pre-step`（改写/拒绝认领的输入）、`agent/request`（改写模型请求）、`agent/request-error`（错误恢复决策）；
- **serial（无 `next()`）**：`agent/turn-stopping`——turn 停止前的终端检查点，监听者按序运行，任何一个都可以阻止 turn 停止。

`preStep` 方法展示了 `agent/pre-step` waterfall 的用法：

```ts
private async preStep(target: InboxTarget, position: { turn: number; step: number }): Promise<PreparedStep> {
  const claimed = this.inbox.claim(target, position.turn)
  const assembly = await this.loopCtx.systemPrompt.assemble(assembleContextFor(this, signal))
  const sections = renderContextSections(assembly)
  const context = this.runtimeContext.project(joinContextSections(sections), sections)
  const decision = await this.dispatch.waterfall(
    'agent/pre-step', { messages: claimed, ...position, signal },
    (): Promise<PreStepDecision> => Promise.resolve<PreStepDecision>({
      kind: 'enter',
      messages: context === undefined ? claimed : [...claimed, context],
    }),
  )
  return decision.kind === 'reject' ? decision : { ...decision, assembly }
}
```

`requestProposal` 函数有一个精巧细节——从 adapter 派生的值在插件提议前被删掉：

```ts
function requestProposal(header: EpochHeader): LlmCallConfig {
  if (header.adapterDefaults === undefined) return header.config
  const proposal = { ...header.config }
  if (header.adapterDefaults.reasoningEffort === true) delete proposal.reasoningEffort
  if (header.adapterDefaults.maxTokens === true) delete proposal.maxTokens
  return proposal
}
```

这让插件可以重新提议 `reasoningEffort` 和 `maxTokens`，而不是被 adapter 的默认值锁死。

## 架构速览

```
ReactLoopAgent 相位机
┌─────────┐  wake   ┌───────────┐
│  idle   │ ──────→ │  running  │
│(lastTurn)│         │(turn,step)│
└─────────┘ ←────── └───────────┘
     ↑        done        ↑
     │                    │
     │    ┌───────────┐   │
     └──→ │maintenance│ ←─┘ (互斥)
          └───────────┘

turn() 循环:
  turn/start → [preStep → step/start → step → step/end]* → turn/end
```

## 关键设计决策

1. **热路径零分配**：`dispatch` 在构造时一次性构建 fused dispatcher，热路径 dispatches 不分配内存。这是性能敏感的 Agent 主循环的必要优化。

2. **四种投递方法的精确分工**：send（通用）/ followup（普通 turn）/ steer（step 转向）/ inject（不唤醒的上下文）——分工精确到"唤醒与否"和"目标 inbox 边界"。

3. **max-tokens 是 sticky 的**：一旦某个 step 达到 token 上限，后续正常完成的 step 不会降级 turn 结果。这保证了 turn 的终止原因反映了最严重的问题。

4. **`requestProposal` 删掉 adapter 默认值**：让插件可以重新提议 `reasoningEffort` 和 `maxTokens`，而不是被 adapter 的默认值锁死。这体现了"插件可以覆盖一切"的设计哲学。

## 本章小结

- `ReactLoopAgent` 是 dsh 的默认心脏，但设计刻意"不特殊"——`Agent` 接口是消费者唯一依赖。
- 三态相位机：idle / maintenance / running，维护任务与 turn 驱动互斥。
- 四投递：send / followup / steer / inject，分工精确到唤醒与否。
- 事件拦截：pre-step 决定模型看什么，request-error 决定失败怎么办，turn-stopping 决定何时能停。
- 取消纪律：cause 由 signal 携带，工具必须配合到达静止。

## 阅读思考题

1. `send` 方法里的 `wakingAfterAbort` 逻辑把 target 强制改为 `'next-turn'`。为什么唤醒输入不能加入已中止的活动？如果允许它加入，会破坏什么不变量？
2. `runMaintenance` 在非 idle 时同步抛错。为什么维护任务不能和 turn 驱动并行？如果允许并行，会引入什么竞态条件？
3. `agent/turn-stopping` 是 serial 事件，没有 `next()`。如果一个监听者在 `turn-stopping` 里做了耗时操作（比如等待用户确认），主循环会怎样？

> **下一章预告**：第 7 章将讲 dsh 的 LLM 层——`packages/llm/llm` 里的 `Message` / `ContentBlock` / `StreamChunk` 词汇表，以及 adapter 契约。
# 第 7 章　LLM 流式抽象：Message / ContentBlock / StreamChunk

dsh 的 LLM 层在 `packages/llm/llm`：它定义整个系统公用的对话词汇（`Message`/`ContentBlock`）、完整的模型请求、原始流块协议 `StreamChunk`、每个 adapter 必须实现的契约。这一章把这层讲透。核心源码在 `packages/llm/llm/src/types.ts`。

## 内容块：text / reasoning / image / tool-call / tool-result

dsh 的对话模型：一个 `Message` 是一组**类型化 content blocks**。块联合从 merge-extensible map 派生：

```ts
export interface ContentBlockMap {
  'text': TextBlock
  'reasoning': ReasoningBlock
  'image': ImageBlock
  'tool-call': ToolCallBlock
  'tool-result': ToolResultBlock
}

export type ContentBlockType = keyof ContentBlockMap
export type ContentBlock = ContentBlockMap[ContentBlockType]
```

每种块的结构（源码原文）：

- `TextBlock`：`{ type: 'text', text: string }` —— 普通文本。
- `ReasoningBlock`：`{ type: 'reasoning', text: string }` —— 模型思维过程，与可见文本分离。
- `ImageBlock`：`{ type: 'image', attachment: ImageAttachmentRef }` —— 持久的图片附件（dsh 有独立的 `attachment` seam 管理图片身份与元数据）。
- `ToolCallBlock`：`{ type: 'tool-call', id: CallId, name: string, arguments: string }` —— 工具调用请求，`arguments` 是原始 JSON 字符串。
- `ToolResultBlock`：`{ type: 'tool-result', toolCallId: CallId, content: ContentBlock[], isError?: boolean }` —— 工具调用结果，嵌套 `content` 允许富文本结果。

**新增 modality 的门槛**：源码注释说"A new modality belongs in the merge-extensible map only when its adapter, UI, compaction, and durable replay paths honor it"——一个新块类型要进 map，必须同时有 adapter、UI、compaction、durable replay 四条路径支持。

## Message 与 provenance

`Message` 本身是不可变值：稳定 id + 角色 + 块 + 来源。`packages/llm/llm/src/message.ts` 里 assistant 消息携带 provenance：

```ts
export interface AssistantProvenance {
  provider: string
  model: string
  replayState?: unknown   // adapter-private replay data
}
```

`replayState` 是 adapter 私有的重放数据——`LlmRuntime` 只在"目标 adapter 同时拥有历史 provider 和目标 provider"时把它暴露给目标 adapter。这是"重放保真"的机制核心：换模型后，历史消息依然可以被原 adapter 重放。

## 模型请求组装与 `request/header` 日志

每个 step 的模型请求由三部分组装：**prompt sections（`ctx.systemPrompt` 按序拼接）+ tool schemas（`ctx.tools` 的 allowlist 投影）+ 从日志派生的历史（`deriveMessages()`）**。

`ReactLoopAgent.buildRequest()` 方法（`packages/core/agent-loop/src/agent.ts`）展示了完整的请求构建流程：

```ts
private async buildRequest(turn, step, tools, system, boundaryMessages, signal) {
  const persistedHeader = session.requestHeader()
  const seedConfig = deepFreeze(structuredClone(
    this.requestHeaderLogged
      ? requestProposal(persistedHeader!)
      : { ...route, ...reasoningEffort === undefined ? {} : { reasoningEffort }, ...maxTokens === undefined ? {} : { maxTokens } },
  ))
  const proposedConfig = await this.dispatch.waterfall(
    'agent/request', { turn, step, signal },
    () => Promise.resolve(seedConfig),
  )
  // ...
  const header = canonicalHeader({ config, ...adapterDefaults, ...system, ...tools })
  if (!this.requestHeaderLogged) {
    this.session.append('request/header', { header, reason: baseline === undefined ? 'initial' : 'resume' })
    this.requestHeaderLogged = true
  } else if (baseline === undefined || !headerEquals(baseline, header)) {
    this.session.append('request/header', { header, reason: 'change' })
  }
}
```

请求头在 step 内、派发前作为 `request/header` 事件**记入日志**。`headerEquals` 做相等性检查——如果两次请求头相同，跳过日志记录。这是 KV cache 友好的设计。

## Adapter 契约：`LlmAdapter` 与多 Provider 注册

Adapter 层是 dsh 的一个 **capability seam**：Service Definition 在 `llm/llm`（`ctx.llm`），Provider 是各个 adapter 包（`llm-deepseek`、`llm-pi-ai`），Consumer 是主循环。`docs/architecture.md` 的扩展点地图：

> Add a model provider → register its adapter on `ctx.llm`.

adapter 契约包括：把 `Message`/`ContentBlock` 转成 provider 的 wire 格式、实现 `StreamChunk` 流、报告 usage。

## `BlockAssembler` 与流式组装

`BlockAssembler`（`llm/llm` 包）把原始 `StreamChunk` 流组装成完整的 `AssistantMessage`。在 `ReactLoopAgent.step()` 里可以看到它的用法：

```ts
const assembler = new BlockAssembler()
const chunkSeqs: number[] = []
try {
  const stream = preparedCall?.stream(request) ?? this.loopCtx.llm.stream(request)
  for await (const chunk of stream) {
    chunkSeqs.push(this.session.append('assistant/chunk', { turn, step, chunk }).seq)
    assembler.push(chunk)
  }
} catch (error: unknown) {
  if (signal.aborted) {
    const content = assembler.interruptedBlocks()
    if (content.length > 0) {
      this.session.append('assistant/message', {
        turn, step,
        message: createAssistantMessage({ content, source: { provider, model } }),
        interrupted: true,
        ...assembler.usage === undefined ? {} : { usage: assembler.usage },
      }, { surfaceOp: 'append', sourceEventSeqs: chunkSeqs })
    }
  }
  throw error
}
```

关键设计：
- **chunks 边流边记 `assistant/chunk` 事件**——token 级重放保真度。
- **中断时**：如果流在中途被中止，`assembler.interruptedBlocks()` 返回已收集的部分内容，记为 `assistant/message` with `interrupted: true`。
- **usage 随消息走**：`assembler.usage` 直接附加到 `assistant/message` 事件，没有独立的 usage 记录。

## FinishReason 与错误恢复

```ts
export interface FinishReasonMap {
  'stop': { kind: 'stop' }
  'tool-calls': { kind: 'tool-calls' }
  'max-tokens': { kind: 'max-tokens' }
  'aborted': { kind: 'aborted'; failure: LlmFailure }
  'error': { kind: 'error'; failure: LlmFailure }
}
```

`LlmFailure` 是 provider-neutral 的失败事实：

```ts
export interface LlmFailure {
  readonly message: string
  readonly code: string
  readonly status?: number
  readonly providerRetryAfterMs?: number
  readonly requestId?: ProviderRequestId
}
```

`step()` 方法在 `assembler.finish` 为 `error` 或 `aborted` 时进入 `agent/request-error` waterfall——监听者返回 retry action 或保留原始错误。

## TokenUsage：disjoint 计数

```ts
export interface TokenUsage {
  inputTokens: number
  outputTokens: number
  cacheReadTokens?: number
  cacheWriteTokens?: number
  reasoningTokens?: number
}
```

源码注释："Counts are DISJOINT: `inputTokens` is uncached input only; cached input is reported separately as `cacheReadTokens`/`cacheWriteTokens`"。计数是不相交的——`inputTokens` 只是非缓存输入，缓存输入单独报告。Adapters whose providers fold cache hits into a total prompt count 需要自己减出来。

## 架构速览

```
LLM 层词汇
┌─────────────────────────────────────────┐
│  ContentBlockMap (merge-extensible)     │
│  ├── text      TextBlock                │
│  ├── reasoning ReasoningBlock           │
│  ├── image     ImageBlock               │
│  ├── tool-call ToolCallBlock            │
│  └── tool-result ToolResultBlock        │
├─────────────────────────────────────────┤
│  Message = id + role + ContentBlock[]   │
│  AssistantProvenance = provider + model │
│                         + replayState?  │
├─────────────────────────────────────────┤
│  StreamChunk → BlockAssembler           │
│  → assistant/chunk (token级)            │
│  → assistant/message (组装后)           │
├─────────────────────────────────────────┤
│  LlmAdapter seam: ctx.llm              │
│  Provider: llm-deepseek / llm-pi-ai    │
│  Consumer: 主循环                        │
└─────────────────────────────────────────┘
```

## 关键设计决策

1. **`replayState` 支撑跨 adapter 重放**：adapter 私有的重放数据让换模型后历史消息依然可以被原 adapter 重放。这是"重放保真"的机制核心。

2. **`ToolCallBlock.arguments` 是原始 JSON 字符串**：不预解析，保留模型原话。这与 `SessionEventMap` 的 `tool/call` 设计一致——"exactly as the model produced it (unparsed)"。

3. **usage 随消息走，无独立记录**：`assistant/message` 事件携带 step 的 `usage`，避免两处记账漂移。这是"模型可见即日志"在 token 计量层面的体现。

4. **新 modality 必须四路齐备**：adapter、UI、compaction、durable replay 四条路径都支持才能加入 `ContentBlockMap`。这保证了每种内容类型在所有下游路径都能正确处理。

## 本章小结

- LLM 层是 dsh 的"对话词汇表 + 适配器契约"。
- `ContentBlockMap` 是 merge-extensible 的块类型联合，新 modality 必须四路齐备。
- `Message` 不可变，assistant 消息携带 provenance（`replayState` 支撑跨 adapter 重放）。
- 请求 = sections + schemas + 派生历史，请求头作为 log-only 事件入日志。
- `BlockAssembler` 边流边记 `assistant/chunk`，中断时记 `interrupted: true` 的 `assistant/message`。

## 阅读思考题

1. `replayState` 是 adapter 私有的重放数据。如果用户从 DeepSeek 切换到另一个 provider，历史消息的 `replayState` 还有意义吗？dsh 如何处理这种情况？
2. `BlockAssembler` 在流中断时用 `interruptedBlocks()` 收集已有的部分内容。如果中断发生在 reasoning block 的中间，组装出的消息会是什么样子？
3. `TokenUsage` 的计数是 disjoint 的。如果一个 adapter 的 provider 把 cache hits 折叠进 `prompt_tokens`，adapter 需要怎么做才能报告正确的 disjoint 计数？

> **下一章预告**：第 8 章将拆开 `packages/core/tools`——工具系统是 Agent 的"手"。`ToolDefinition` 的六段守卫管线是 dsh 工程哲学的缩影。
# 第 8 章　工具系统：ToolDefinition 与守卫管线

工具系统是 Agent 的"手"。dsh 的 `ctx.tools` 是一个作用域工具注册表 + 守卫执行管线：模型只能看到 allowlist 投影出的 schema，任何工具调用都要穿过六段守卫。这一章拆开 `packages/core/tools/src/index.ts`。读完后，你应该能理解工具从注册到执行的完整生命周期，以及为什么"工具只返回规范值"是"模型可见即日志"在工具层的延伸。

## `ToolDefinition`：schema + execute + 纯投影三件套

一个注册的工具是 `ToolDefinition`：模型可见的 `ToolSchema` + 强制的规范输出声明 + `execute` 函数 + 可选的最终内容回调 + 可选 UI 呈现器。`packages/core/tools/src/index.ts` 导出了 `defineTool` 和相关的 schema DSL：

```ts
export {
  defineTool,
  valueSchemaSpecToJsonSchema,
  parameterSchemaSpecToJsonSchema,
  validateArgs,
  ToolArgsError,
  type ValueSchemaAnnotations,
  type StringValueSchemaSpec,
  type NumberValueSchemaSpec,
  type IntegerValueSchemaSpec,
  type BooleanValueSchemaSpec,
  type NullValueSchemaSpec,
  type ArrayValueSchemaSpec,
  type ObjectValueSchemaSpec,
  type JsonValueSchemaSpec,
  type OneOfValueSchemaSpec,
  type ValueSchemaSpec,
  type ParameterPropertySpec,
  type ParameterSchemaSpec,
  type ParameterJsonSchema,
  type InferValue,
  type InferArgs,
  type DefineToolOptions,
} from './schema.ts'
```

关键设计：

- **`output` 是强制的规范输出声明**：含 JSON Schema（对每个成功的规范值强制校验）、纯投影 `render`（从校验过的参数和值投影到 Native/model content）。
- **`execute` 返回唯一规范值**：工具不能直接往模型对话里塞东西——它只能返回值，由管线决定如何呈现。这是"模型可见即日志"不变量在工具层的体现。
- **`finalizeContent` 是同步的最后一英里变换**：注册表在 execute 开始时快照此回调，对每个规范化结果恰好调用一次。**必须 total，不许 throw**。
- **`timeoutMs` 是协作式超时预算**：由 `timeout-policy` 工具守卫强制执行，**绝不发给模型**——`schemas()` 只 whitelist name/description/parameters。
- **`isConcurrencySafe` 是纯同步分类器**：只有 `true` 才 opt-in 并行。这个元数据也绝不模型可见。

## JSON-value schema DSL：一套词汇描述参数与输出

插件作者用一套词汇描述类型化参数和类型化输出值。`ValueSchemaSpec` 支持 `string` / `number` / `integer` / `boolean` / `null` / `array` / `object` / 作者专用 `json` / 恰好一个 `oneOf`。标量 `enum`/`const` 必须匹配节点类型；显式 object 节点必须声明 `additionalProperties: true | false`。

为什么限制这么严？因为 `output.schema` 要对**每个成功的规范值**做 JSON Schema 校验——DSL 的封闭性保证 schema 一定可执行、可校验、可投影。`packages/core/tools/src/json-schema.ts` 导出了 `assertSupportedJsonSchema` 和 `validateJsonSchemaValue`，在注册时和执行后分别校验。

## 六段守卫管线

`docs/tool-execution-pipeline.md` 给出了完整流程。核心顺序：

```
model 产出 tool-call block
  → 记 session 事件 tool/call（执行前先记日志！）
  → presentCall(args) 渲染 UI pending 卡
  → tools/pre-execute waterfall（hooks、permission、sandbox）
  → 注册的 monotonic guards（deny or abstain；identity protected）
  → ctx.approval 一次性审批（无应答=deny）
  → tools/execute waterfall（timeout、retry、metrics——around dispatch）
  → 注册的 execute() 本体
  → fs/write-intent 或 fs/edit-intent（仅 tool-fs 突变）
  → 工具自己的 session 事件
  → tools/post-execute waterfall（accept、block、replace、add context）
  → 注册表外层规范化（pipeline/result 快照 throw 变成 isError）
  → ToolDefinition.finalizeContent
  → tools/result 同步通知（冻结的权威结果）
  → 记 session 事件 tool/result
  → presentResult(args, result) 渲染 UI 完成卡
```

几个关键点：

1. **`tool/call` 在 execute 之前就记日志**——"logged before execution"。执行失败也有记录。
2. **三个 waterfall**（`tools/pre-execute`、`tools/execute`、`tools/post-execute`）让 policy/hooks/sandbox 全部在不动主循环的前提下插入。
3. **denied 的调用也走 post-execute**——拒绝不是无声消失，政策监听者能看到结果。
4. **`tools/result` 观察不可变、lossless-JSON 结果**——"This lets hooks span tool families without coupling the tools to one policy service"。

## Code Mode：`run_code` 与 SDK 渲染

`packages/core/tools/src/code-mode.ts` 导出了 `createRunCodeTool` 和 `RUN_CODE_NAME`。Code Mode 有两个变体（`DSH_TOOLS_MODE=native|code|both`）：

- **native 模式**：普通工具注册。
- **code 模式**：`run_code` 是注册表唯一的 wire 贡献，其他可见能力由加载 runtime 语言里的生成 SDK 段声明。

`packages/core/tools/src/index.ts` 里有一个 `CODE_ONLY_INSTRUCTION` 常量：

```ts
const CODE_ONLY_INSTRUCTION = `\`${RUN_CODE_NAME}\` is the only tool you can call directly — a tool call naming any other tool fails. Reach every tool the SDK declares below from inside the program.`

const COLLAPSE_SECTION_ORDER = 99
```

`COLLAPSE_SECTION_ORDER = 99`——在 persona（order 0）之后、per-tool guidance（100-199）之前，让模型先读到哪些工具可以调，再读每个工具的用途。

SDK 渲染器支持 TypeScript 和 Python：

```ts
const SDK_RENDERERS: Record<string, (schemas: ToolSdkSchema[]) => string> = {
  typescript: renderToolsSdk,
  python: renderToolsSdkPy,
} satisfies Record<CodeSdkLanguage, (schemas: ToolSdkSchema[]) => string>
```

## 并行工具调用：`isConcurrencySafe` 与有界滚动池

并行模型是"**有界滚动池**"：按 `executionMode` 分类，符合 `isConcurrencySafe` 的调用可以并发 execute，但 pre 按序、post 按模型顺序（"ordered pre, concurrent execute; ordered post"）。结果按模型顺序结算，保证日志顺序与模型预期一致。

## UI 呈现：presentCall / presentResult 的纯函数设计

工具的 UI 呈现意图是**设计的一部分**。`presentCall(args)` 渲染 pending 状态，`presentResult(args, result)` 渲染完成状态。两者都**必须纯且无副作用**——因为 UI 可能在 live streaming 期间**和** session-log 重放期间调用它们（"a UI may call it during live streaming AND a session-log replay, so it must depend only on args"）。

## 架构速览

```
工具执行管线（六段守卫）
┌─────────────────────────────────────────┐
│ 1. tool/call → session event (先记日志)  │
│ 2. tools/pre-execute waterfall          │
│ 3. monotonic guards (deny / abstain)    │
│ 4. ctx.approval 一次性审批               │
│ 5. tools/execute waterfall → execute()  │
│ 6. tools/post-execute waterfall         │
│    → normalize → finalizeContent        │
│    → tools/result → tool/result event   │
└─────────────────────────────────────────┘

ToolDefinition
├── ToolSchema (name, description, parameters)
├── output: ToolOutputDefinition (schema + render)
├── execute(args, exec) → canonical value
├── finalizeContent?(exec, result) → ContentBlock[]
├── timeoutMs? (协作式, 不发给模型)
└── isConcurrencySafe? (opt-in 并行)
```

## 关键设计决策

1. **工具只返回规范值**：呈现由管线负责。这保证了"模型可见即日志"——工具的输出必须经过管线的规范化和记录，才能到达模型。

2. **`tool/call` 在 execute 之前记日志**：执行失败也有记录。这让崩溃恢复能精确知道"模型请求了什么工具、参数是什么"，即使工具还没执行完。

3. **`finalizeContent` 必须 total，不许 throw**：它是管线的最后一环，任何异常都会破坏整个管线的封闭性。源码注释："must be total, never throw"。

4. **UI 呈现是纯函数**：live 和 replay 输出一致。这是"可重放"哲学在 UI 层的延伸——同一个纯函数，无论在实时流还是日志重放中调用，输出相同。

## 本章小结

- 工具系统是 dsh 工程哲学的缩影：**工具只返回规范值，呈现由管线负责**。
- 六段守卫管线让 policy/hooks/sandbox/approval 全部在不动主循环的前提下插入。
- `isConcurrencySafe` opt-in 并行，有界滚动池按模型顺序结算。
- Code Mode 下 `run_code` 是唯一 wire 贡献，子调用重入完整守卫工具管线。
- UI 呈现是纯函数，live 与 replay 一致。

## 阅读思考题

1. `tool/call` 在 execute 之前就记日志。如果一个工具的 `execute` 函数永远不返回（比如死循环），日志里会有什么？主循环会怎样？
2. `isConcurrencySafe` 只有 `true` 才 opt-in 并行。如果一个工具声明了 `isConcurrencySafe: true` 但实际上修改了共享状态，会引入什么问题？dsh 有什么机制能发现这种错误？
3. Code Mode 下 `run_code` 是唯一注册的工具。如果模型尝试调用其他工具，会发生什么？这种"collapse"设计的好处和代价是什么？

> **下一章预告**：第 9 章将讲 dsh 最值得抄的设计——Capability Seam 三件套：Service Definition / Service Provider / Consumer。
# 第 9 章　Capability Seams：能力接缝三件套

dsh 最值得抄的设计：**Capability Seam（能力接缝）**。一个可替换能力由三个角色构成——Service Definition（服务定义）、Service Provider（服务提供者）、Consumer（消费者，通常是模型面对的工具）。这一章讲透这个模式，以及它如何让"换一个 Provider 就换掉整个产品"成为现实。读完后，你应该能在源码里找到 seam 的三个角色，并理解为什么"只有一个角色不算 seam"。

## Service Definition / Service Provider / Consumer：一个完整 seam

`docs/architecture.md` 的定义：

> A **seam** is a swappable capability with three roles: a **Service Definition** declaring the interface, a **Service Provider** implementing it, and a **Consumer** using it, commonly a model-facing tool. A package may combine roles, but one role alone is not a seam; adding a capability means designing all three.

翻译：一个 seam 是**三个角色**的完整设计。一个包可以兼多个角色，但**只有一个角色不算 seam**——加一个能力意味着把三个角色一起设计。

`AGENTS.md` 把它设为红线：

> **A capability seam comprises Service Definition / Service Provider / Consumer roles.** It is complete, never one role; split only when roles evolve independently.

## Seam 全景表

从 `docs/architecture.md` + `docs/subsystems/README.md` 整理的完整 seam 表：

| Seam | Service Definition | Service Provider(s) | Consumer(s) |
|---|---|---|---|
| LLM | `llm/llm`（`ctx.llm`） | `llm-deepseek`、`llm-pi-ai` | 主循环 |
| Filesystem | `fs/fs`（`ctx.fs`） | `fs-local`、`fs-e2b` | `tool-fs`、`tool-str-replace-editor`、`tool-fs-search` |
| Shell | `shell/shell`（`ctx.shell`） | `bash-local`、`pwsh-local` | `tool-bash`、`tool-pwsh`、`tool-bash-persistent` |
| Subprocess | `subprocess/subprocess`（`ctx.subprocess`） | `subprocess-local` | 工具、shell、LSP |
| Sandbox | `sandbox/sandbox`（`ctx.sandbox`） | `sandbox-local`（bwrap/Landlock/Seatbelt/ACL） | `bash-sandbox`、`pwsh-sandbox` |
| Code Runtime | `code-runtime/code-runtime`（`ctx.codeRuntime`） | `code-runtime-worker-thread` | `run_code` |
| Subagent | `subagent/subagent`（`ctx.subagents`） | in-process / fork / ACP / Codex / Claude Code / dsh-sdk | `tool-subagent`、`tool-subagent-control`、`tool-subagent-report` |
| Workflow | `workflow/workflow`（`ctx.workflowEngine`） | `workflow-worker-thread` | `tool-workflow`、`tool-ralph` |
| Web | `web/web`（`ctx.web`） | web search/fetch providers | `tool-web` |
| Spill | `spill/spill`（`ctx.spillStore`） | `spill-local` | `spill-policy` |
| Skill | `skill/skill`（`ctx.skills`） | `skill-filesystem`、`skill-badge` | `tool-skill` |
| Terminal | `terminal/terminal`（`ctx.terminals`） | 各 PTY 后端 | `tool-terminal`、`tool-bash-persistent` |
| Compaction | `compaction/compaction` | `compaction-basic`、`tool-result-pruner` | 主循环上下文 |
| LSP | `lsp/lsp`（`ctx.lsp`） | `lsp-stdio` | `tool-lsp` |
| Persistence | `session/session-persistence` | JSONL / SQLite | 主循环、session-query |

## 为什么换 Provider 就换掉整个产品

`docs/architecture.md` 给出了最惊艳的例子：

> Seams are why one provider swap changes the whole product. Filesystem and subprocess providers share one execution world, so pointing them at a remote sandbox moves Bash, PTY, and LSP with them, with no provider forks.

**Filesystem 和 subprocess provider 共享同一个执行世界**——把它们指向远程沙箱，Bash、PTY、LSP 全部跟着走，不需要任何 provider 分叉。`packages/e2b/` 就是 POC：`fs-e2b`、`subprocess-e2b` 是 E2B 的 sandbox + FS/subprocess adapter。一个产品，本地跑是一套后端，远程跑是另一套后端，**模型看到的工具 schema 完全不变**。

Subagent 同样：provider "vary just as widely behind one interface, from a fresh child agent to a delegated turn in another product"——从全新子 Agent 到"委托给另一个产品的 turn"（ACP/Codex/Claude Code 后端）都藏在同一个接口后面。

## Scope 与 scoped registration：per-agent 可见性与生命周期

Capability seam 的"作用域"维度由 `packages/core/scope` 提供。`packages/core/scope/src/store.ts` 定义了核心原语：

```ts
export interface ScopeLayer {
  isEmpty(): boolean   // 层完全为空，可回收
}
```

`ScopedLayers<L>` 拥有 eager 全局层 + 惰性创建的精确 scope 层。**读不创建层**（`peek(undefined)` 表示无覆盖），`merge()` 物化"插入序全局命名条目 + scoped shadows"。

`NamedEntries<V>` 提供插入序查找与 live 迭代：

```ts
export class NamedEntries<V> implements EntryValues<V> {
  private data = new Map<string, V>()

  insert(name: string, value: V): () => void {
    const data = this.data
    if (data.has(name)) throw this.duplicateError(name)
    data.set(name, value)
    let active = true
    return () => {
      if (!active) return
      active = false
      data.delete(name)
      if (data.size === 0 && this.data === data) this.data = new Map()
    }
  }
  // ...
}
```

`insert` 返回一个 idempotent undo——注册即 effect，卸载时回卷。注意当表清空时，`this.data = new Map()` 释放旧 Map 的引用——这是内存友好的设计。

`AnonymousEntries<V>` 给每个 append 唯一身份，相等值也保持独立——用于不需要命名的注册（比如 prompt sections）。

**这对产品意味着什么**：工具注册表、skill 注册表、prompt sections 都支持"per-agent 可见性"——一个 agent preset 挂载的插件只影响那个 agent 的组成，不污染全局。

## 架构速览

```
Capability Seam 三件套
┌─────────────────────────────────────────────┐
│  Service Definition (接口声明)                │
│  例: fs/fs → ctx.fs                         │
│  声明: FsTarget, read/write/edit 接口        │
├─────────────────────────────────────────────┤
│  Service Provider (实现)                      │
│  例: fs-local (本地FS), fs-e2b (远程FS)      │
│  实现: 同一接口, 不同后端                      │
├─────────────────────────────────────────────┤
│  Consumer (使用, 通常是模型工具)               │
│  例: tool-fs (edit/read/write/read_image)    │
│  使用: ctx.fs 的接口, 不关心 Provider         │
└─────────────────────────────────────────────┘

换 Provider → 换整个产品 (本地 ↔ 远程)
模型看到的 schema 不变
```

## 关键设计决策

1. **三角色完整设计**：只有一角不算 seam。加一个能力意味着把 Definition / Provider / Consumer 一起设计。这避免了"先写工具，再补接口"的倒置设计。

2. **共享执行世界**：FS 和 subprocess provider 同源，换远程沙箱全部跟着走。这是"换骨不换面"的极致——产品形态完全由 Provider 决定，Consumer 不需要任何改动。

3. **scope 原语的惰性创建**：`ScopedLayers` 的读不创建层，只有写才创建。这让"没有 per-agent 覆盖"的场景零开销。

4. **`NamedEntries.insert` 返回 idempotent undo**：注册即 effect，卸载时精确回卷。这与 Cordis 的 `ctx.effect()` 机制一致——所有注册都可逆。

## 本章小结

- Capability Seam 三件套是 dsh 对"可替换性"的完整回答：Definition / Provider / Consumer 一起设计。
- 换 Provider 换掉整个产品（本地 ↔ 远程沙箱、子 Agent 跨产品委托），模型看到的 schema 不变。
- `scope` 原语让注册可以 per-agent 可见，插件树可以局部重写。
- 所有注册都是 effect，卸载自动回卷。

## 阅读思考题

1. `docs/architecture.md` 说"one role alone is not a seam"。如果一个包只实现了 Service Definition 和 Consumer（没有独立的 Provider），它算 seam 吗？在什么场景下这种"内置 Provider"是合理的？
2. `NamedEntries.insert` 返回的 undo 是 idempotent 的——调用两次只删一次。如果一个插件意外调用了两次 undo，会发生什么？为什么 idempotent 是必要的？
3. `ScopedLayers` 的读不创建层。如果一个 agent preset 没有覆盖任何工具，它的 scope 层在内存中存在吗？这种"零开销"设计在什么场景下可能失效？

> **下一章预告**：第 10 章将拆开本地能力的两大支柱——文件系统 seam 和 Shell seam——以及它们如何共享同一个执行世界。
# 第 10 章　文件系统与 Shell：本地能力的两大支柱

一个编码 Agent 的两只手：读写文件（FS seam）和执行命令（Shell + Subprocess seam）。这一章拆开 dsh 的三个能力 seam——filesystem、shell、subprocess——以及它们如何共享同一个"执行世界"。读完后，你应该能理解 read-before-write 策略如何通过事件门实现、`resolve(request): Spec` 的显式默认设计、以及 `DSH_*` 环境变量词汇的作用。

## FS seam：`FsTarget` / 读写编辑 / `fs/observed` 观察状态

`ctx.fs` 是文件系统能力 seam 的服务定义（`packages/fs/fs`）。它提供 `FsTarget`（路径目标）、读/写/编辑结果、observed-file 状态、`FsErrorCode`。

消费者是：
- `tool-fs`（`edit`/`read`/`read_image`/`write` 四个工具）
- `tool-str-replace-editor`（独立的 view/create/unique literal replace/line insert 工具）
- `tool-fs-search`（`glob`/`grep`，spawn 打包的 `@vscode/ripgrep`）

一个有趣的设计：`read_image` 只有在 `ctx.attachments` 存在时才注册——schema 与路由无关，执行时拒绝除非确切路由的模型声明支持图像输入。这是"能力与条件分离"的例子：工具的 schema 始终注册，但执行时根据运行时条件决定是否放行。

## read-before-write 策略与 `fs/*` 事件门

dsh 的 read-before-write/edit 政策由 `@deepseek-ai/dsh-fs-observation-policy` 提供——一个 **`fs/*` 事件门插件**。

机制：tool-fs 的突变发出 `fs/write-intent` 或 `fs/edit-intent` 事件；观察政策监听这些事件，检查该文件是否已经被 read 过（`fs/observed` 状态）；没读过就不放行。

关键：**政策挂在事件上，不挂在工具上**——工具不 import 政策服务，政策也不改工具 schema。这正是第 8 章管线哲学的延伸：工具只管能力，政策通过事件门插入。

## Shell seam：`ShellExecRequest/Spec` 的显式分离

Shell 能力 seam 的服务定义在 `shell/shell`（`ctx.shell`）。dsh 的 `AGENTS.md` 有一条"Explicit > implicit at package boundaries"红线：

> **Explicit > implicit at package boundaries**: defaulting is an explicit `resolve(request): Spec` step in the owning implementation, never a hidden `?? default` inside `run()`.

`ShellExecRequest`（调用者想要什么）先经过一个显式的 `resolve(request): ShellExecSpec` 步骤变成 `ShellExecSpec`（实际执行什么）——默认值在 resolve 里显式落定，绝不藏在 `run()` 里用 `?? default`。

这在 `packages/shell/shell` 的类型定义里可以看到：`ShellExecRequest` 是调用者提供的请求，`ShellExecSpec` 是解析后的完整规格。两者分离让"部署 A 的超时 30s、部署 B 的超时 120s"成为配置差异，而不是代码差异。

提供者：`bash-local`（本机 bash）、`pwsh-local`（Windows PowerShell）；消费者：`tool-bash`、`tool-pwsh`、`tool-bash-persistent`（持久 PTY）。

`tool-bash` 的 `run_in_background` 通过 `ctx.jobs` 注册后台任务，`enableRunInBackground` 配置（默认 true）关掉时直接从 schema 里移除该参数——不是"执行时拒绝"，而是"schema 里就不存在"。

## Subprocess seam：全显式 spawn spec 与 `DSH_*` 环境词汇

`ctx.subprocess` 是进程派发层（`packages/subprocess/subprocess`）：`SubprocessSpawnSpec`（全显式 spawn 参数）、基于 offset 的输出读取器、未分类的 `SubprocessOutcome`、受管理的 `DSH_*` 环境词汇。

**`DSH_*` 环境变量词汇**是 dsh 的一个巧妙设计：`@deepseek-ai/dsh-shell-env` 给子进程注入一组受管理的环境变量（来源、工作区根、session 上下文等），子进程和工具通过这组词汇互相沟通，而不需要隐式约定。

`glob`/`grep` 工具 spawn 打包的 ripgrep 二进制（`@vscode/ripgrep`）就是走 `ctx.subprocess` 的普通前台调用——"no host `rg` install and no shell layer"。

## 持久终端（terminal seam）与 PTY

`ctx.terminals` 是持久终端 seam：`terminal_open`/`terminal_send`/`terminal_read`/`terminal_close`/`terminal_list`/`terminal_signal` 六个工具（`tool-terminal`）。终端 id 是 branded，后端/会话契约明确。

`tool-bash-persistent` 是一个 owner-isolated 的持久 bash 工具——部署组合提供 PTY 后端。与 `tool-bash` 的一次性执行不同，持久终端保持会话状态（环境变量、工作目录等）。

## 架构速览

```
本地能力三大支柱
┌─────────────────────────────────────────────┐
│  FS seam (ctx.fs)                           │
│  Definition: fs/fs                          │
│  Provider: fs-local                         │
│  Consumer: tool-fs / tool-fs-search         │
│  政策: fs-observation-policy (事件门)        │
├─────────────────────────────────────────────┤
│  Shell seam (ctx.shell)                     │
│  Definition: shell/shell                    │
│  Provider: bash-local / pwsh-local          │
│  Consumer: tool-bash / tool-pwsh            │
│  resolve(request): Spec ← 显式默认          │
├─────────────────────────────────────────────┤
│  Subprocess seam (ctx.subprocess)           │
│  Definition: subprocess/subprocess          │
│  Provider: subprocess-local                 │
│  Consumer: 工具 / shell / LSP               │
│  DSH_* 环境词汇                              │
├─────────────────────────────────────────────┤
│  Terminal seam (ctx.terminals)              │
│  Definition: terminal/terminal              │
│  Consumer: tool-terminal / tool-bash-persistent │
└─────────────────────────────────────────────┘

FS + Subprocess 共享执行世界
→ 换远程沙箱全部跟着走
```

## 关键设计决策

1. **能力与政策分离**：工具只管能力，read-before-write 挂在 `fs/*` 事件门上。工具不 import 政策服务，政策也不改工具 schema。这是"通过事件解耦"的经典模式。

2. **`resolve(request): Spec` 的显式默认**：默认值在 resolve 里显式落定，绝不藏在 `run()` 里用 `?? default`。这让部署差异变成配置差异，而不是代码差异。

3. **`DSH_*` 环境变量词汇**：受管理的环境变量让子进程与工具通过约定沟通，而不是隐式约定。这比"随便设环境变量"更安全、更可审计。

4. **`enableRunInBackground` 控制 schema**：关掉时直接从 schema 里移除 `run_in_background` 参数，而不是"执行时拒绝"。模型看到的 schema 始终是准确的——不存在"模型请求了一个不支持的功能"的情况。

## 本章小结

- FS / Shell / Subprocess 是 dsh 本地能力的三大支柱，共享一个设计基因。
- 能力与政策分离：工具只管能力，read-before-write 挂在 `fs/*` 事件门上。
- 显式默认：`resolve(request): Spec` 把默认值显式落定，部署差异变配置差异。
- 共享执行世界：FS 和 subprocess provider 同源，换远程沙箱全部跟着走。
- `DSH_*` 环境变量让子进程与工具通过约定沟通。

## 阅读思考题

1. `fs-observation-policy` 通过监听 `fs/write-intent` 事件来阻止未读写入。如果一个工具绕过了 `tool-fs` 直接调用 `ctx.fs` 的写入接口，这个政策还能生效吗？
2. `resolve(request): Spec` 的设计把默认值从 `run()` 里移出来。如果一个 Provider 的 `resolve` 函数和另一个 Provider 的 `resolve` 函数对同一个字段有不同的默认值，会引入什么问题？
3. `DSH_*` 环境变量是"受管理的"——只有 `dsh-shell-env` 插件能设置它们。如果一个恶意插件也设置了 `DSH_*` 变量，会引入什么安全风险？dsh 有什么机制防止这种情况？

> **下一章预告**：第 11 章将拆开 dsh 的沙箱系统——它不是"隔离环境"，而是一个"文件效应政策"。
# 第 11 章　沙箱：进程的文件效应政策

Agent 最危险的能力是执行代码。dsh 的沙箱设计有一个独特的视角：**沙箱不是"隔离环境"，而是一个"文件效应政策"**。它只管文件效应（`SandboxMode`），网络与进程可见性在词汇之外。这一章拆开 `packages/sandbox/sandbox/src/index.ts`。

## `SandboxMode`：read-only / workspace-write / danger-full-access

`ctx.sandbox` 把同世界子进程的 argv 包裹进一个文件效应政策，而不把消费者耦合到平台 runner。`packages/sandbox/sandbox/src/index.ts` 定义了核心类型：

```ts
export type SandboxMode = 'read-only' | 'workspace-write' | 'danger-full-access'
```

- **`read-only`**：拒绝写。POSIX runner 额外授予 shell 需要的 `/dev/null` sink；Windows ACL runner 不授予显式可写根。
- **`workspace-write`**：允许在工作区根和后端承诺的 temp 区域下写。
- **`danger-full-access`**：绕过约束。

**只有前两种模式可以发给 provider**。源码注释说得很清楚："A `danger-full-access` consumer spawns its original argv and does not call `ctx.sandbox`"——`danger-full-access` 的消费者直接 spawn 原始 argv，根本不调用 `ctx.sandbox`。

## `SandboxExecutionPolicy`：每次调用携带的完整政策

```ts
export interface SandboxExecutionPolicy {
  mode: SandboxMode
  workspaceRoot: string
  sessionId?: SessionId
}
```

`workspaceRoot` 从调用 session 的不可变 cwd 派生。`sessionId` 是 branded 的 `SessionId`——后端用它做 per-session 状态管理（比如 Windows ACL 给每个 session/workspace 对分配随机私有 temp 目录和 SID）。

源码注释："What one confined execution is allowed to touch — carried PER CALL, not fixed on the provider"。政策是每次调用携带的，不是固定在 provider 上的——两个消费者可以同时请求不同边界（bash 用 `read-only`，子 agent 用 `workspace-write`），而不改变 provider 状态。

## 三平台后端：Linux bwrap / Landlock、macOS Seatbelt、Windows ACL

`dsh-sandbox-local` 提供三个平台后端：

- **Linux**：bwrap（bubblewrap，用户命名空间沙箱）+ Landlock（内核 LSM）。Landlock 还有一个专门的 node addon：`native/landlock-run`（`@deepseek-ai/node-addon-landlock-run`）。
- **macOS**：Seatbelt（sandbox-exec profile）。
- **Windows**：ACL restricted-token 后端。

`dsh-bash-sandbox` 和 `dsh-pwsh-sandbox` 是消费者——它们包裹 bash/pwsh 的 argv 后 spawn。

## Enforcement：`full` vs `partial` 的诚实报告

dsh 沙箱最诚实的设计：

```ts
export type SandboxEnforcement = 'full' | 'partial'
```

- `full`：后端治理模式承诺的每个文件效应；
- `partial`：活跃后端或旧内核 ABI 只治理子集——**要求绝对边界的调用者不能把它当 `full`**。

源码注释："callers requiring an absolute boundary must not treat it as `full`"。当前 partial 的案例：旧 Landlock ABI、Windows ACL runner 的 Everyone/hard-link 边界。**"partial 就是 partial"**——沙箱不会假装比实际更安全。

## 沙箱的隔离哲学

`docs/subsystems/sandbox.md` 明确区分了沙箱和容器/VM：

> Containers, microVMs, and remote execution are sibling implementations of whole capability seams, not providers of `ctx.sandbox`.

**容器、microVM 和远程执行是"完整能力 seam 的兄弟实现"，不是 `ctx.sandbox` 的 provider**。这个区分很重要：dsh 不把"隔离"降维成"一个包裹函数"，而是让每个执行域（本地/容器/远程）都成为完整的 seam 组合。`packages/e2b/` 就是远程执行的完整 seam 实现。

## 架构速览

```
沙箱体系
┌─────────────────────────────────────────────┐
│  SandboxMode (文件效应政策)                   │
│  read-only → workspace-write → danger-full  │
├─────────────────────────────────────────────┤
│  SandboxExecutionPolicy (每次调用携带)        │
│  mode + workspaceRoot + sessionId           │
├─────────────────────────────────────────────┤
│  三平台后端                                   │
│  Linux: bwrap + Landlock                    │
│  macOS: Seatbelt                            │
│  Windows: ACL                               │
├─────────────────────────────────────────────┤
│  SandboxEnforcement (诚实报告)                │
│  full → 完全治理                             │
│  partial → 部分治理, 不假装更安全              │
├─────────────────────────────────────────────┤
│  消费者: bash-sandbox / pwsh-sandbox         │
│  包裹 argv → 交给后端                        │
│  danger-full-access → 不调用 ctx.sandbox     │
└─────────────────────────────────────────────┘
```

## 关键设计决策

1. **只管文件效应**：`SandboxMode` 三值，网络与进程可见性不在词汇内。这让沙箱的关注点极度聚焦——"文件效应"是一个可精确描述、可精确验证的约束。

2. **`danger-full-access` 消费者不调用 `ctx.sandbox`**：绕过约束的方式是"不进入沙箱系统"，而不是"在沙箱系统里请求全权限"。这避免了沙箱后端需要处理"全权限"这个特殊情况。

3. **诚实报告 `full` vs `partial`**：沙箱不会假装比实际更安全。这是一个工程诚信问题——`partial` 意味着"某些文件效应可能逃逸"，调用者需要自己决定是否接受这个风险。

4. **远程是完整 seam，不是沙箱 provider 变体**：E2B 是 FS + subprocess + sandbox 三个 seam 的完整实现，不是 `ctx.sandbox` 的一个 provider。这让远程执行和本地执行在架构上是对等的。

## 本章小结

- dsh 的沙箱是一个"文件效应政策 + 诚实报告"系统。
- `SandboxMode` 三值：read-only / workspace-write / danger-full-access。
- 三平台后端：bwrap/Landlock、Seatbelt、Windows ACL。
- 诚实报告：`full` vs `partial`，绝不假装比实际安全。
- 远程是完整 seam：E2B 是兄弟实现，不是沙箱 provider 的变体。

## 阅读思考题

1. `SandboxMode` 只管文件效应，网络与进程可见性不在词汇内。如果一个被沙箱限制的进程通过网络发送了敏感数据，`ctx.sandbox` 能阻止吗？这种"只管文件"的设计是局限还是有意为之？
2. `SandboxEnforcement = 'partial'` 意味着"某些文件效应可能逃逸"。在什么场景下你会拒绝接受 `partial` enforcement？在什么场景下 `partial` 已经足够？
3. dsh 把容器/VM/远程执行定位为"完整能力 seam 的兄弟实现"，而不是沙箱 provider 的变体。如果把远程执行实现为 `ctx.sandbox` 的一个 provider，会引入什么架构问题？

> **下一章预告**：第 12 章将拆开 dsh 的审批体系——在 Agent 动手之前的四层"刹车"。
# 第 12 章　审批与交互：在动手前踩刹车

Agent 在用户机器上发出的每个副作用都需要用户同意。dsh 的审批体系分四层：一次性审批（approval）、人机问答（user-questions）、直接命令（commands）、权限预设（permission-presets）。这一章拆开 `packages/interaction`。读完后，你应该能理解为什么"fail-closed"是审批系统的第一原则，以及 plan mode 如何作为"软约束"与硬约束（沙箱/审批）共存。

## Approval seam：`ApprovalOutcome` 四值与 fail-closed

`ctx.approval` 回答一个问题：**这个具体动作可以继续吗？** `packages/interaction/user-approval/src/index.ts` 定义了核心类型：

```ts
export type ApprovalOutcome = 'allowed-once' | 'rejected' | 'cancelled' | 'unavailable'
```

**封闭且 fail-closed**：
- `allowed-once`：只授予被问到的那个动作。
- `rejected`：显式拒绝。
- `cancelled`：请求撤回。
- `unavailable`：应答者缺失/非拥有/抛错/不合规——**全部 deny**。

源码注释："A missing, non-owning, throwing, or non-conforming answerer becomes `unavailable` rather than opening the gate"——应答者出任何问题，门都是关的，而不是开的。

每个请求拿到一个新鲜的 `ApprovalRequestId`（branded），把 `approval/asked` 和 `approval/decided` 审计事件配对：

```ts
declare module '@deepseek-ai/dsh-session/types' {
  interface SessionEventMap {
    'approval/asked': {
      id: ApprovalRequestId
      toolName: string
      callId?: CallId
      reason?: string
    }
    'approval/decided': {
      id: ApprovalRequestId
      outcome: ApprovalOutcome
    }
    'approval/policy': {
      policy: ApprovalPolicy
      source?: 'delegation'
    }
  }
}
```

注意 `approval/policy` 事件的 `source?: 'delegation'`——标记从父 agent 继承的审批策略覆盖。这在子 agent 委派时很重要。

## Per-session policy：ask / never

```ts
export type ApprovalPolicy = 'ask' | 'never'
```

- **`ask`**（默认）：委托给组合的应答者链，无应答默认 `unavailable`。
- **`never`**：永不询问任何人，每个 ask 确定性返回 `rejected`——"The strict headless stance (CI, unattended runs) and the policy whose outcome is knowable without asking"。

有效政策是 session 日志里最后一个 `approval/policy` 事件的值，回退到服务配置。**唯一写路径是 `setApprovalPolicy(session, policy)`**——重放可以重建覆盖。

## User-questions seam：工具需要人回答时

`ctx.userQuestions` 是工具或权限插件需要人先回答才能继续时的 provider-neutral 词汇。`packages/interaction/user-questions` 定义了：

```ts
interface AskUserQuestionOption {
  label: string          // 用户可见选项文本，也是模型可见的选中值
  description?: string   // 可选 UI 帮助文本
}
```

`AskUserQuestionIntent` 可选声明一个已知决策类型：`approve` 命名肯定选项而不是依赖选项顺序。**intent 只改呈现，不改答案字段**——认出 tag 的 UI 用特定方式呈现，但回答的仍是同样的选项 label。

消费者例子：`tool-ask-user`（`ask_user_question` 工具暂停工具调用直到活跃 UI provider 返回人类答案）、`exit_plan_mode`（呈现计划让人审阅）。

## Commands：不经模型的直接人机通道

`ctx.commands` 是人类命令注册表。交互适配器用它发现并直接执行插件拥有的命令——**不创建模型消息**（"directly execute plugin-owned commands for an exact agent without creating a model message"）。

```ts
interface CommandDefinition {
  name: string        // 小写命令名，无前导斜杠
  summary: string     // 发现 UI 用的人类可读摘要
}
```

例子：`/plan [off|message]`（plan-mode 插件注册）、`/export`（session-log-download）、`/compact`（command-compact）。命令是"绕过模型 turn 的确定性操作"——输入 hint、执行、结果都直接由适配器处理。

## Permission presets 与 plan mode 的软约束

**Permission presets**（`interaction/permission-presets`）：`PresetSpec`/`PresetOption` + 派生的 `custom` 状态 + log-only `permission/preset` 事件。预设让部署把"审批策略组合"打包成可切换的模式。

**Plan mode**（`packages/plan/plan-mode`）是**软约束**：

> Plan mode is **soft guidance**. [Sandbox mode] and [approval policy] enforce restrictions independently; neither reads or writes plan state.

`plan/mode` 是 log-only 的整值替换 session 事件（`{ active: boolean }`）——持久、可重放、永不在模型 transcript 里。激活时，部署拥有的 guidance section 作为 `plan:policy` prompt section（order 50）包含进每个模型请求。

`exit_plan_mode` 工具在非 plan 模式下也保持注册（进出 plan mode 只变 prompt section，不变工具目录），但执行时非 plan 模式会拒绝。退出需要完整 markdown 计划（`#` 开头）+ user-questions 审阅。

## 架构速览

```
审批与交互四层体系
┌─────────────────────────────────────────────┐
│ 1. Approval (一次性审批)                      │
│    ctx.approval → ApprovalOutcome (4值)       │
│    fail-closed: 缺应答者 = unavailable        │
│    per-session policy: ask / never            │
├─────────────────────────────────────────────┤
│ 2. User-questions (人机问答)                  │
│    ctx.userQuestions → AskUserQuestionOption  │
│    intent 只改呈现, 不改答案字段                │
├─────────────────────────────────────────────┤
│ 3. Commands (直接命令)                        │
│    ctx.commands → CommandDefinition           │
│    不经模型, 确定性操作                        │
├─────────────────────────────────────────────┤
│ 4. Presets + Plan mode (预设 + 软约束)        │
│    permission-presets: 审批组合预设             │
│    plan-mode: 软约束指导 (不读写沙箱/审批)      │
└─────────────────────────────────────────────┘
```

## 关键设计决策

1. **fail-closed 是第一原则**：应答者出任何问题（缺失、非拥有、抛错、不合规），门都是关的。`unavailable` 和 `rejected` 的效果相同——deny。这让审批系统在任何异常情况下都是安全的。

2. **intent 只改呈现，不改答案字段**：`AskUserQuestionIntent` 的 `approve` 标签让 UI 可以用特定方式呈现"批准"选项，但回答的仍是同样的选项 label。这让同一套答案逻辑在不同 UI 下都能工作。

3. **plan mode 是软约束**：sandbox mode 和 approval policy 独立强制，不读写 plan state。plan mode 只通过 prompt section 引导模型行为，不通过硬机制阻止。这让 plan mode 可以在不破坏现有安全机制的情况下提供额外指导。

4. **`approval/policy` 事件支持 delegation source**：子 agent 继承父 agent 的审批策略时，用 `source: 'delegation'` 标记。这让重放能精确区分"用户设置的策略"和"委派继承的策略"。

## 本章小结

- dsh 的"刹车"体系分四层，各司其职。
- **Approval**：一次性动作审批，四值封闭、fail-closed，per-session 政策 ask/never。
- **User-questions**：工具需要人回答的 provider-neutral 词汇，intent 只改呈现。
- **Commands**：不经模型的确定性人机通道。
- **Presets + Plan mode**：审批组合预设 + 软约束指导，硬约束独立强制。

## 阅读思考题

1. `ApprovalOutcome` 的 `unavailable` 和 `rejected` 效果相同（deny），但它们的语义不同。在什么场景下区分这两者是有价值的？比如审计日志、错误诊断、还是 UI 呈现？
2. `exit_plan_mode` 工具在非 plan 模式下保持注册但执行时拒绝。为什么不直接注销这个工具？这种"注册但拒绝"的设计在什么场景下比"注销"更好？
3. plan mode 是"软约束"——它只通过 prompt section 引导模型，不通过硬机制阻止。如果模型无视 plan mode 的指导直接执行了副作用，谁来负责？这种设计是缺陷还是有意为之？

> **下一章预告**：第 13 章将讲 dsh 的持久化与崩溃恢复——日志如何变耐久，以及孤儿 turn 的 synthetic `interrupted` 封口。
# 第 13 章　持久化与崩溃恢复：日志如何变耐久

第 5 章讲了内存中的事件日志，这一章讲它如何变耐久：`SessionPersistence` seam、JSONL/SQLite 两个后端、flush checkpoint、崩溃恢复（孤儿 turn 的 synthetic `interrupted` 封口）、spill 存储。读完后，你应该能理解为什么"不截断"是崩溃恢复的正确策略，以及 synthetic `interrupted` 如何保持日志的平衡性。

## `SessionPersistence` seam：JSONL 与 SQLite 两个后端

持久化是**另一个 capability seam**：`packages/session/session-persistence/src/index.ts` 定义了 `SessionPersistence` 抽象服务：

```ts
export abstract class SessionPersistence extends Service {
  constructor(ctx: Context) {
    super(ctx, 'sessionPersistence')
  }

  abstract locate(meta: SessionHeader): SessionLocation | undefined
  // ...
}
```

两个可互换后端实现同一契约：**没有平行的持久化事件类型**——持久化直接存现有的 `SessionEvent`。JSONL 后端把每个 session 存成一个 transcript 文件（`SessionLocation.kind === 'jsonl'` 返回绝对路径）；SQLite 后端所有 session 共享一个数据库（返回 `undefined`）。

`SessionHeader` 是**日志旁边的元数据**：

```ts
export interface SessionHeader {
  readonly version: number
  readonly id: SessionId
  readonly createdAt: number
  readonly cwd?: string
  readonly parentSession?: SessionId
  readonly seedLength?: number
  readonly origin?: 'subagent'
  readonly delegationDepth?: number
  readonly agentPreset?: string
}
```

格式版本、cwd、lineage、seed boundary——存储关切，不是对话事件，不进 `SessionEventMap`，永不进 `deriveMessages()`。

## Flush checkpoint：`session/flush` 与有界写批处理

`session/event` 是**同步**通知；持久化插件把事件拷进 per-session 控制器，**不阻塞生产者**。然后：

> The first pending event starts a fixed batching window, and later events join without resetting its deadline. Expiry starts one durable batch; events admitted during that write receive their own deadline and form a follow-up batch.

第一个 pending 事件启动固定批处理窗口，后续事件加入而不重置截止时间。到期写一批；写入期间新到的事件获得自己的截止时间，形成后续批次。**`session/flush` 取消等待并排干到静止**——主循环在认领下一个普通 turn 前用它作为排序和错误观察检查点。

一个被拒绝的后台写入保留事件并暂停自动重试；新事件启动新窗口，显式 flush 立即重试并通过 `agent/error` 和 logger 报告失败——**绝不作为关闭 turn 之后的 session event**。

## 崩溃恢复：孤儿 turn 的 synthetic `interrupted` 封口

崩溃恢复是最精彩的设计。一个后端重载一个 mid-turn 崩溃的日志，发现一个没有 `turn/end` 的孤儿 `turn/start`。**它不截断**——一个 turn 可能巨大（多 step、大工具输出），那些事件在崩溃前已经持久追加了。相反：

> …it closes the orphaned turn with a synthetic `turn/end { reason: { kind: 'interrupted' } }`, keeping the interrupted execution balanced without changing any standalone events before or after it. `interrupted` is the one `TurnEndReason` no loop emits.

用一个人造的 `turn/end { reason: { kind: 'interrupted' } }` 封口，保持中断执行"平衡"（balanced），不改动前后任何独立事件。`interrupted` 是**唯一一个主循环永远不会自己发出的 `TurnEndReason`**——所以重放时可以精确区分"真实结束"和"崩溃恢复"。

修复只适用于冷 session。对于 live id，`SessionPersistence.load(id)` 等待权威内存快照持久化，只在平衡时返回；开放的 live turn 拒绝而不是接收 synthetic 中断边界。

## Spill 存储：超大工具输出溢出到文件

`ctx.spillStore` 把工具的**超大文本**持久化，返回模型面对的定位符 + 检索指引。`packages/spill/spill` 定义了：

```ts
interface SaveTextSpill {
  owner: SpillOwner        // 保存时命名空间（session id）
  source: SpillSource      // 工具与调用身份
  suggestedName: string    // 建议名，后端净化成单个安全路径段
  content: string          // 完整文本（UTF-8）
}
```

`spill-local` 是私有 session-scoped 文件；`spill-policy` 是 `tools/post-execute` 政策（Consumer）。**fork 继承已存在的定位符但不复制/重拥有产物**；fork 后产生的 spill 用子 session id。

消费者例子：`tool-fs-search` 的 capped 结果、超长工具输出。`SaveTextSpill` 返回不透明定位符 + 后端提供的检索提示 + 精确字节数。

## Settings / Credentials / Workspace：其他持久化域

- **Settings**（`settings/`）：`SettingsNamespace` 注册、分层解析（defaults → composition `base` → user document）、owner scopes、hot commits。
- **Credentials**（`credentials/`）：`CredentialRef` 引用（**永远不是值**）进配置、按操作解析、UI-safe `CredentialInfo`。源码注释："Managed credentials live separately in `.credentials.yaml`; a credential left in either `.env` remains a lower-priority fallback"。
- **Workspace**（`workspace/`）：`WorkspaceId` 是 generated uuid，**绝不是路径**（路径规范化会重写路径，引用锚必须稳定）。`realpathNormalize`（`fs.realpath`；尾斜杠、`..`、symlink 全部解析）是唯一唯一性准则。

## 架构速览

```
持久化与恢复体系
┌─────────────────────────────────────────────┐
│  SessionPersistence seam                    │
│  Definition: session-persistence            │
│  Provider: JSONL (每session一个文件)          │
│           SQLite (所有session共享一个DB)      │
├─────────────────────────────────────────────┤
│  写批处理                                     │
│  session/event → 控制器 (不阻塞)              │
│  固定窗口 → 到期写一批 → session/flush 排干   │
├─────────────────────────────────────────────┤
│  崩溃恢复                                     │
│  孤儿 turn/start → synthetic turn/end        │
│  reason: { kind: 'interrupted' }             │
│  不截断, 保持平衡, 主循环永不自己发出           │
├─────────────────────────────────────────────┤
│  Spill 存储                                   │
│  ctx.spillStore → 超大文本溢出到文件           │
│  返回不透明定位符 + 检索指引                   │
├─────────────────────────────────────────────┤
│  其他域                                       │
│  Settings: 分层解析                           │
│  Credentials: 只存引用, 不存值                 │
│  Workspace: uuid, 不是路径                    │
└─────────────────────────────────────────────┘
```

## 关键设计决策

1. **崩溃恢复不截断**：用 synthetic `interrupted` 封口而不是截断孤儿 turn。一个 turn 可能巨大（多 step、大工具输出），截断会丢失已持久化的有价值事件。`interrupted` 是唯一主循环不发的 `TurnEndReason`，重放时精确区分真实结束和崩溃恢复。

2. **有界写批处理**：第一个 pending 事件启动固定窗口，后续事件加入而不重置截止时间。这让写入吞吐量和延迟之间取得平衡——低流量时一个事件一批，高流量时多个事件一批。

3. **Credentials 只存引用**：`CredentialRef` 引用永远不是值本身。值在 `.credentials.yaml` 或 `.env` 里，通过引用解析。这避免了敏感值被记入 session 日志或配置文件。

4. **WorkspaceId 是 uuid 不是路径**：路径规范化会重写路径，引用锚必须稳定。uuid 不受路径规范化影响，是一个稳定的标识符。

## 本章小结

- 持久化是 seam：`SessionPersistence` 抽象 + JSONL/SQLite 两个后端，无平行事件类型。
- 有界写批处理：`session/flush` 作为主循环的排序/错误检查点。
- 崩溃恢复不截断：synthetic `interrupted` 封口保持平衡，是唯一主循环不发的终止原因。
- Spill 溢出：超大输出持久化 + 定位符，模型只见指引。
- 其他域：Settings 分层、Credentials 只存引用、Workspace 用 uuid 不用路径。

## 阅读思考题

1. `interrupted` 是 synthetic 的 `TurnEndReason`。如果一个 turn 在第一个 step 的 assistant/chunk 流式传输中途崩溃，恢复后的日志里会有哪些事件？`deriveMessages()` 如何处理这个不完整的 assistant 消息？
2. JSONL 后端每个 session 一个文件，SQLite 后端所有 session 共享一个数据库。在什么场景下你会选择 JSONL 而不是 SQLite？反过来呢？
3. `SessionHeader` 的 `seedLength` 字段记录了"通过 seed 继承的前导事件数量"。在 fork 场景下，这个字段如何帮助区分"父的历史"和"子的工作"？

> **下一章预告**：第 14 章将讲 dsh 的上下文组装、压缩与记忆——prompt sections 的按序拼接、compaction seam、以及 skills/goals/schedule。
# 第 14 章　上下文、压缩与记忆

模型窗口有限，Agent 的上下文必须被精心组装、压缩和遗忘。这一章拆开 dsh 的上下文体系：system-prompt assembly、`agent.inject()`、compaction seam、skills、goals/schedule/todo。读完后，你应该能理解 prompt sections 的 order 约定、compaction 如何通过 log-only 事件工作、以及 skills 的 host+per-scope 分层注册表。

## System-prompt assembly：prompt sections 的按序拼接

`ctx.systemPrompt` 拥有 prompt-section 与 tool-schema 组装。`packages/core/system-prompt` 定义了 `PromptSection` 接口和组装逻辑。

**Sections 按升序拼接**。约定：
- `-100`：harness identity opener
- `-99`：`harness:source`（`addHarnessSourceSection` 注册，告诉 agent dsh 实现 checkout 在哪、警告不要从路径推断 cwd）
- `0`：deployment persona
- `50`：plan policy
- `99`：Code Mode 的 collapse statement（`COLLAPSE_SECTION_ORDER`）
- `100-199`：per-tool guidance band

`AssembleContext` 是 merge-extensible：`dsh-agent` 加可选 live `agent` 字段，`assembleContextFor(agent, signal)` 一起设置显式字段。`ToolProviderResult` 提供模型可见 schema 集 + 预限制名字宇宙（`knownNames`——区分"配置名字打错了"和"故意隐藏的已知工具"）。

**协作组装**（cooperative assembly）：sections 的贡献者通过 waterfall 协作，一个有效的 `complete` section 成为唯一 prompt section。

## `agent.inject()` 与 per-request context

第 6.1 节讲过 `agent.inject()`：排队模型可见上下文给下一个 pre-step，**不唤醒**。注入的上下文会成为 `user/message` 事件（`source` 区分：file-change notices、subdir AGENTS.md、skill content、cron notifications、goal continuation round）。源码注释说："All three project their `content` verbatim; `source` tells them apart"。

`packages/context` 组还有 request-context 插件，提供更细粒度的上下文注入。

## Compaction seam：`compaction/*` 事件与 tool-result pruner

compaction 是另一个 capability seam：`packages/compaction/compaction/src/index.ts` 定义了 `CompactionEngine` 抽象服务：

```ts
export abstract class CompactionEngine extends Service {
  constructor(ctx: Context) {
    super(ctx, 'compaction')
  }
  // ...
}
```

compaction 通过声明合并给 `SessionEventMap` 加 `compaction/start` / `compaction/summary` / `compaction/end`——**不是 `SurfaceEventType`**（无 `surfaceOp`）。这意味着 compaction 事件不参与表面事件的投影，只在日志里记录压缩的发生。

Provider：
- `compaction-basic`：基本摘要压缩
- `tool-result-pruner`：剪枝工具结果（spill 化的大输出）

Consumer：`command-compact` 是 `/compact` 命令消费者。

`CompactionTrigger` 定义了触发原因：

```ts
export type CompactionTrigger = 'pressure' | 'context-overflow'
```

`ManualCompactionError` 是手动压缩的失败分类：

```ts
export class ManualCompactionError extends Error {
  constructor(
    readonly code: ManualCompactionErrorCode,  // 'busy' | 'cancelled' | 'changed' | 'summary' | 'commit' | 'persistence'
    message: string,
    options?: ErrorOptions,
  ) { super(message, options) }
}
```

## Skills：注册表、目录注入与 `skill` 工具

skill 能力族（`packages/skill`）：Definition（`dsh-skill`，`ctx.skills`）+ Provider（`skill-filesystem` 本地、`skill-badge` 打包徽章）+ Consumer（`tool-skill`）。

**注册表是 host+per-scope 分层的**（沿用 tools registry 在 `dsh-scope` 上建立的形状）：host 行和 repo 插件进全局层，agent preset 挂载的插件进该 preset 的层——provider 名**每层唯一**而不是进程级唯一。读合并全局层 + 查看 scope 的链，最近层的条目在重名时直接胜出。

**Skills 是可选指令，不是 session 事件**——词汇在 skill 页而不是 core.md。这意味着 skill 内容不会自动进入 session 日志，而是在需要时通过 `agent.inject()` 注入。

## Goals / Schedule / Todo：同会话目标与定时记忆

**Goals**（`goal/`）：持久化的同会话目标域。
- `GoalId` branded，`GoalRef`（id + revision）CAS 变更。
- `GoalPhase` = active / paused / blocked / complete。
- blocked 是唯一的"被问题卡住"持久状态，带稳定 lower-kebab-case `code`（机器路由）+ 人类可读 `message`。
- 工具 `create_goal`/`get_goal`/`update_goal`：create/edit/pause/resume 要求直接人类 root authority；complete/blocked 也接受当前目标轮次；默认 blocked 下界是 3 个已承认轮次。

**Schedule**（`schedule/`）：session-local 提醒。
- `schedule_create`/`schedule_delete`/`schedule_list`
- v1 接受 after_seconds、显式绝对 at、有界固定速率 every_seconds
- 披露 session-local delivery
- 只在 opt-in Schedule 插件加载后创建的 live root Agent scopes 内注册

**Todo**（`todo/`）：`todo_write` 是 session-owned 状态；UI 渲染最新的 `todo/write` 事件为 checklist。`allowParallelInProgress` 无默认必须显式声明——"so the catalog states its choice: true, whose description invites several in_progress items"。

## 架构速览

```
上下文与记忆体系
┌─────────────────────────────────────────────┐
│  System-prompt assembly                     │
│  sections 按 order 升序拼接                  │
│  -100 identity → 0 persona → 50 plan → ... │
├─────────────────────────────────────────────┤
│  agent.inject()                             │
│  不唤醒, 排队模型可见上下文                    │
│  source 区分: file-change / AGENTS.md / ... │
├─────────────────────────────────────────────┤
│  Compaction seam                            │
│  Definition: compaction/compaction           │
│  Provider: compaction-basic / pruner         │
│  Consumer: command-compact                   │
│  事件: compaction/start/summary/end (log-only)│
├─────────────────────────────────────────────┤
│  Skills                                     │
│  host+per-scope 分层注册表                    │
│  可选指令, 不是 session 事件                   │
├─────────────────────────────────────────────┤
│  Goals / Schedule / Todo                    │
│  Goals: 持久化 CAS 状态机                     │
│  Schedule: session-local 提醒                │
│  Todo: log-only 状态                         │
└─────────────────────────────────────────────┘
```

## 关键设计决策

1. **sections 按升序拼接**：order 数值越小越靠前。这让 identity（-100）始终在最前面，persona（0）紧随其后，per-tool guidance（100-199）在最后。每个插件只需要知道自己的 order 值，不需要关心其他插件的存在。

2. **Skills 是可选指令，不是 session 事件**：skill 内容不自动进入日志，只在需要时通过 `agent.inject()` 注入。这让 skill 内容不会污染 session 日志，也不会被 `deriveMessages()` 自动投影到模型历史。

3. **compaction 事件是 log-only**：`compaction/start/summary/end` 不是 `SurfaceEventType`，不参与表面事件的投影。这让压缩的发生被记录在日志里（可审计），但不影响模型看到的历史。

4. **Goals 的 blocked 下界是 3 个已承认轮次**：一个目标不能在被承认之前就从 blocked 转移到其他状态。这防止了模型过早地认为问题已解决。

## 本章小结

- System-prompt assembly 按 order 升序拼接 sections，`-100` identity → `0` persona → `50` plan policy。
- `agent.inject()` 不唤醒地排队模型可见上下文，落进下一个被承认的请求。
- compaction seam 加 log-only 事件，basic 摘要 + tool-result pruner 剪枝。
- Skills 是 host+per-scope 分层注册表，可选指令不是事件。
- Goals 持久化 CAS 状态机、Schedule session-local 提醒、Todo log-only 状态。

## 阅读思考题

1. prompt sections 的 order 是编译时常量。如果两个插件注册了相同 order 的 section，会发生什么？dsh 有什么机制防止 order 冲突？
2. compaction 的 `compaction/summary` 事件记录了压缩后的摘要。如果压缩丢失了关键信息（比如一个重要的工具调用结果），模型还能恢复它吗？
3. Goals 的 `blocked` 状态带一个 `code` 字段用于机器路由。如果一个 goal 的 `code` 在不同版本间变了（比如从 `rate-limited` 变成 `throttled`），会破坏什么？

> **下一章预告**：第 15 章将拆开 dsh 的子 Agent 体系——按名注册的多 Provider、一次性 vs 可续接两种形态、以及后台 job 运行时。
# 第 15 章　子 Agent 与多 Agent 委派

dsh 的子 Agent 体系是它最丰富的 seam 之一：**按名注册的多 Provider**（in-process / fork / ACP / Codex / Claude Code / dsh-sdk），一次性 vs 可续接两种形态，外加控制工具与后台 job 运行时。这一章拆开 `packages/subagent`。读完后，你应该能理解"能力即方法"的 TypeScript narrowing 设计，以及 `ctx.jobs` 如何让 bash/subagent/PTY 共用一套生命周期。

## Subagent seam：按名注册的多 Provider

`ctx.subagents` 与 bash 不同：**多个 provider 实现共存于一个上下文，按名注册**。`packages/subagent/subagent/src/types.ts` 定义了 Provider 契约。

> It differs from the other capability seams because **multiple provider implementations coexist** in one context, registered by name (`ctx.subagents`), while bash allows only one executor.

Provider 全家：
- `subagent-spawn-in-process`（同进程子 Agent）
- `subagent-fork-in-process`（fork 后端）
- `subagent-acp`（ACP 自动化协议）
- `subagent-codex`（Codex）
- `subagent-claude-code`（Claude Code）
- `subagent-dsh-sdk`（dsh SDK）

Consumer 全家：`tool-subagent`（按 provider 的委派工具）、`tool-subagent-control`（全局 `send_message`/`interrupt_agent`/`list_agents`）、`tool-subagent-report`（child-scoped `report` 返回通道）。

## 启动时能力检查：`SubagentCapabilities`

provider 在一个静态描述符上声明**启动时**特性，服务在一次性 run 存在前检查。`packages/subagent/subagent/src/types.ts` 定义了：

```ts
export interface SubagentCapabilities {
  readonly outputSchema: boolean
  readonly depthLimit: boolean
  readonly toolFilter: boolean
  readonly persona: boolean
}
```

需要 provider 没有的能力的请求被大声拒绝（`SubagentError('UNSUPPORTED_CAPABILITY')`）——"never accepted-then-ignored"。源码注释说："a request that needs a capability the chosen provider lacks is rejected with a typed error rather than accepted-then-ignored (the 'fail loud, no silent degradation' rule)"。

`SubagentStartRequest` 带必填 `parent`（提供 session cwd、lineage、delegation depth）：

```ts
export interface SubagentRunInfo {
  readonly runId: SubagentRunId
  readonly provider: string
  readonly id: SessionId
  readonly local: boolean
}
```

## 一次性 vs 可续接子 Agent：`prepareContinuable` 即能力

子 Agent 分两种形态：

- **一次性（one-shot）**：`start()` 路径，provider 组合 child。
- **可续接（continuable）**：由 continuation manager 自己组合，**gated by 一个可选方法，方法存在即能力**（"gated by one optional method whose presence IS the capability, with TS narrowing as the discovery mechanism"）——`SubagentProvider.prepareContinuable`。

`tool-subagent` 的默认配置很讲究：`subagent` 是 `continuable`，默认省略调用进后台并自动结算投递；`subagent_fork` 保持 `one-shot`，默认前台——"the two shipped schemas are not identical"。

## 控制工具：send_message / interrupt_agent / list_agents / report

- **`send_message`** 与 **`interrupt_agent`**：全局注册一次。
- **`list_agents`**：从 `/list-agents` 插件单独加载，用 sessionProjections 和 live Agent 注册表。
- **`report`**：per continuable in-process child 注册（不是全局）——所以这个 schema 只在这样的 child 内部可见，并且**能穿过全局 toolFilter** 存活。

## 后台 job 运行时：`ctx.jobs` 与 job_* 工具

`ctx.jobs` 是通用长任务运行时。`packages/subagent` 的 job 模块定义了：

```ts
interface JobKindMap {
  bash: 'bash'
  subagent: 'subagent'
}
```

`JobKind` 从 merge-extensible map 派生，注册表把 kind 当不透明 id 命名空间。`JobStatus` = `'running' | 'stopping' | 'completed' | 'killed' | 'failed'`。

**Producer 契约**：`JobStart` 声明身份和 starter。运行时在调用 `run()` 前完成 preflight（access + cleanup），之后不再有可失败的步骤——"The runtime finishes preflight before calling `run()` and commits without a later failable step"。

**`JobId` 生成是 `<kind>-N`**；访问控制依赖 owner 授权而非 id 保密（"Access control relies on owner authorization, not id secrecy"）。

`job_kill`/`job_list`/`job_output` 是 kind-agnostic 控制器：后台 bash、PTY sends、subagents 用同一套三个工具读/列/杀。后台完成通知通过 `user/message via agent.inject()` 投递。

## 架构速览

```
子 Agent 体系
┌─────────────────────────────────────────────┐
│  Subagent seam (ctx.subagents)              │
│  多 Provider 按名注册, 共存于一个上下文         │
│  ┌─────────────────────────────────────┐    │
│  │ in-process │ fork │ ACP │ Codex     │    │
│  │ Claude Code │ dsh-sdk              │    │
│  └─────────────────────────────────────┘    │
├─────────────────────────────────────────────┤
│  能力检查 (启动时)                            │
│  SubagentCapabilities: 4 个 boolean flag     │
│  缺能力 → UNSUPPORTED_CAPABILITY (大声拒绝)   │
├─────────────────────────────────────────────┤
│  一次性 vs 可续接                             │
│  start() → 一次性                            │
│  prepareContinuable → 可续接 (方法存在即能力)  │
├─────────────────────────────────────────────┤
│  控制工具                                     │
│  send_message / interrupt_agent (全局)        │
│  list_agents (独立加载)                       │
│  report (child-scoped, 穿过 toolFilter)      │
├─────────────────────────────────────────────┤
│  ctx.jobs (后台 job 运行时)                   │
│  JobKind: bash / subagent (merge-extensible) │
│  job_kill / job_list / job_output            │
│  完成通知 via agent.inject()                  │
└─────────────────────────────────────────────┘
```

## 关键设计决策

1. **多 Provider 按名注册**：与 bash 的单 executor 不同，subagent 允许多个 provider 共存。这让"本地子 agent"和"委托给 Claude Code"可以在同一个上下文中并存，由 tool schema 的参数选择 provider。

2. **能力即方法**：`prepareContinuable` 的存在本身就是能力声明。TypeScript 的类型 narrowing 让服务可以自动发现 provider 是否支持可续接，不需要额外的 flag。

3. **启动时大声拒绝**：`SubagentCapabilities` 在 start 之前检查，缺少能力就拒绝。这比"接受后忽略"更安全——不会产生一个功能不完整的子 agent。

4. **`ctx.jobs` 统一后台任务**：bash、PTY、subagent 共用一套生命周期（running → stopping → completed/killed/failed）和控制工具（kill/list/output）。这让"后台任务管理"不需要为每种任务类型写不同的 UI。

## 本章小结

- 多 Provider 共存：按名注册，in-process / fork / ACP / Codex / Claude Code / dsh-sdk 一个接口。
- 能力即方法：`prepareContinuable` 存在即可续接，TS narrowing 做发现。
- 启动时大声拒绝：`SubagentCapabilities` 前置检查，绝不 accept-then-ignored。
- 控制工具分层：委派工具 per-provider，控制工具全局，report 工具 child-scoped。
- 后台 job 统一：`ctx.jobs` 让 bash/subagent/PTY 共用一套生命周期与控制器。

## 阅读思考题

1. `prepareContinuable` 的存在本身就是能力声明。如果一个 provider 后来移除了这个方法（比如重构），已有的可续接子 agent 会怎样？
2. `ctx.jobs` 的 `JobId` 是 `<kind>-N` 格式（比如 `bash-1`、`subagent-3`）。这种生成方式在什么场景下可能产生冲突？为什么"访问控制依赖 owner 授权而非 id 保密"？
3. `report` 工具是 child-scoped 的，能穿过全局 toolFilter。如果一个恶意子 agent 通过 `report` 工具泄露了敏感信息，谁来负责？这种"穿透"设计是安全风险还是必要功能？

> **下一章预告**：第 16 章将讲 dsh 最"未来感"的两个能力——Workflow（模型写编排脚本）与 Code Runtime（worker-thread 里跑模型写的程序）。
# 第 16 章　Workflow 与 Code Runtime：让模型写编排

这一章讲 dsh 最"未来感"的两个能力：Workflow（模型写一个编排脚本、脚本开子 Agent）与 Code Runtime（worker-thread 里跑模型写的程序）。外加 ralph 工具与 Claude Code / Codex hook 桥。读完后，你应该能理解 `WorkflowMeta` 如何对齐 Claude Code 的约定、Code Mode 的"collapse"设计、以及 hooks 如何复用既有资产。

## Workflow seam：模型写脚本、脚本开子 Agent

`ctx.workflowEngine` 让 Agent 运行一个**模型写的编排脚本**，脚本启动子 Agent。像 bash 一样，它**每个上下文只允许一个引擎实现**——没有命名 provider 注册表（"a second engine replaces the first through plugin configuration rather than running beside it"）。

`packages/workflow/workflow/src/types.ts` 定义了核心类型：

```ts
export interface WorkflowMeta {
  name: string
  description: string
  whenToUse?: string
  phases?: WorkflowPhase[]
}

export interface WorkflowPhase {
  title: string
  detail?: string
  provider?: string
  model?: string
}

export type WorkflowStopReason = 'completed' | 'cancelled' | 'error'

export interface WorkflowResult {
  value: unknown
  stopReason: WorkflowStopReason
  error?: string
}
```

**`WorkflowMeta` 的字段词汇与 Claude Code dynamic-workflows meta block 一致**（name/description/whenToUse/phases）——dsh 直接对齐了 Anthropic 的约定。`phases` 只是进度词汇：`phase()` 调用匹配标题给观察者，不暗示执行结构。

**`WorkflowRunId`** 是 branded 的：

```ts
export type WorkflowRunId = Branded<'WorkflowRunId'>

export function WorkflowRunId(id: string): WorkflowRunId {
  return id as WorkflowRunId
}
```

**脚本全局 API**：`agent(prompt, opts?)`（跑一个 subagent 到完成；带 `opts.schema` 时返回校验过的对象）、`pipeline(items, ...stages)`（每 item 独立跑各阶段，无阶段间屏障）、`parallel(thunks)`（并发跑零参函数并等全部）、`phase(title)`、`log(message)`、`args`。

**引擎**：`workflow-worker-thread`（`node:worker_threads`，每 run 一个 worker，脚本的 vm context 在 worker 里）。**Consumer**：`tool-workflow`（`workflow` 工具）。

## Code Runtime：worker-thread 里跑模型写的程序

`ctx.codeRuntime` 跑一个**模型写的程序**，绑定 host 提供的 async bindings，报告打印了什么、返回了什么。Code execution 是**一个可选能力，不是 agent-loop spine 的一部分**。

`packages/code-runtime/code-runtime/src/types.ts` 定义了：

```ts
export interface CodeRunRequest {
  program: string                  // async function 的函数体
  bindings: CodeBindingNamespace[] // 暴露给程序的 host 函数
  signal?: AbortSignal             // 中止：硬停
}

export interface CodeRunResult {
  value?: CodeJsonValue   // 程序完成值
  logs: string[]          // 程序按序输出的文本
  error?: CodeRunFailure  // 失败时存在
}
```

**错误是结果的字段，不是 `run()` 的 rejection**——"reporting a failed program is the caller's job, not an exception path"。这与 `ShellExecutor.run` 的 resolve-on-failure 契约一致。

Provider 是 `code-runtime-worker-thread`；Consumer 是 `run_code`（Code Mode）。**Code Mode 有两个变体**（`DSH_TOOLS_MODE=native|code|both`）：native 模式是普通工具，code 模式下 `run_code` 是注册表唯一的 wire 贡献。

第 8 章讲过 `CODE_ONLY_INSTRUCTION`：

```ts
const CODE_ONLY_INSTRUCTION = `\`${RUN_CODE_NAME}\` is the only tool you can call directly — a tool call naming any other tool fails. Reach every tool the SDK declares below from inside the program.`
```

程序通过 bindings 调用能力——子调用重入完整守卫工具管线，记 `tool/code-dispatch`，每嵌套执行链接到这个外层结果。

## ralph 工具与动态 workflow

`tool-ralph`（`ralph` 工具）：一个固定前台 workflow，每轮启动一个新鲜的结构化 child；模型只选不可变 objective 和可选轮次上限。这是"重复多轮子 Agent 研究"的封装。

## Hooks：Claude Code / Codex hook 桥

`packages/hooks`：`hook-protocol`（wire 协议库）+ `hooks-claude-code` + `hooks-codex`。它们把 Claude Code / Codex 的 hook 桥接进 dsh——log-only 的 `hook/invoked` / `hook/result` 记录。这让用惯了 Claude Code hooks 的团队可以把既有 hook 脚本直接接进 dsh 管线。

## 架构速览

```
Workflow 与 Code Runtime
┌─────────────────────────────────────────────┐
│  Workflow seam (ctx.workflowEngine)         │
│  模型写编排脚本 → 脚本开子 Agent              │
│  WorkflowMeta 对齐 Claude Code 约定          │
│  全局 API: agent/pipeline/parallel/phase    │
│  引擎: worker-thread (每 run 一个 worker)    │
│  Consumer: tool-workflow                    │
├─────────────────────────────────────────────┤
│  Code Runtime seam (ctx.codeRuntime)        │
│  模型写程序 → worker-thread 里跑              │
│  错误是结果字段, 不是 rejection               │
│  Code Mode: run_code 是唯一 wire 贡献        │
│  子调用重入完整工具管线                        │
├─────────────────────────────────────────────┤
│  ralph (tool-ralph)                         │
│  固定前台 workflow, 多轮结构化子 Agent 研究     │
├─────────────────────────────────────────────┤
│  Hooks (packages/hooks)                     │
│  hook-protocol + hooks-claude-code          │
│  + hooks-codex                              │
│  桥接既有 hook 脚本进 dsh 管线                 │
└─────────────────────────────────────────────┘
```

## 关键设计决策

1. **`WorkflowMeta` 对齐 Claude Code 约定**：字段词汇（name/description/whenToUse/phases）直接复用 Anthropic 的 dynamic-workflows meta block。这让从 Claude Code 迁移到 dsh 的团队可以复用已有的 workflow 定义。

2. **错误是结果字段，不是 rejection**：`CodeRunResult.error` 而不是 `run()` 抛错。这让调用者可以统一处理成功和失败——"reporting a failed program is the caller's job, not an exception path"。

3. **Code Mode 的"collapse"设计**：`run_code` 是唯一注册的工具，其他能力通过 SDK 段在脚本内部暴露。这让模型看到的工具列表极度精简，但脚本内部可以调用所有能力。

4. **子调用重入完整工具管线**：Code Mode 里脚本通过 bindings 调用工具时，每个子调用都经过完整的六段守卫管线。这让安全策略在"模型写代码"的场景下依然生效。

## 本章小结

- Workflow：模型写编排脚本开子 Agent，`meta` 先校验后执行，field 词汇对齐 Claude Code。
- Code Runtime：worker-thread 里跑模型写的程序，错误是结果字段；Code Mode 下 `run_code` 是唯一 wire 贡献。
- ralph：固定前台多轮结构化子 Agent 研究。
- Hooks：Claude Code / Codex hook 桥，既有资产可复用。

## 阅读思考题

1. `WorkflowMeta` 对齐了 Claude Code 的约定。如果 Claude Code 更新了它的 meta block 格式（比如加了新字段），dsh 需要做什么来保持对齐？
2. Code Mode 下 `run_code` 是唯一注册的工具。如果模型在脚本里尝试调用一个不存在的 SDK 函数，会发生什么？错误信息对模型友好吗？
3. 子调用重入完整工具管线意味着每个嵌套调用都经过 approval 和 sandbox。如果一个脚本发起了 100 次子调用，用户会被审批 100 次吗？dsh 有什么机制优化这种体验？

> **下一章预告**：第 17 章将拆开 dsh 的远程调用体系——Typert 类型图生成器 + `@Remote` 方法 + API Gateway + Web UI。
# 第 17 章　远程调用与可观测性：Typert RPC 与 Web UI

dsh 的 Web UI 不是"挂在主循环旁边的网页"，而是一个完整的远程调用体系：Typert 类型图生成器 + `@Remote` 方法 + API Gateway + ACP 自动化服务器 + client modules。这一章拆开 `packages/typert`、`packages/api`、`packages/acp`、`packages/client` 与 `apps/web`。读完后，你应该能理解 Typert 的 Host 对象 ↔ wire 身份分离、session query 的只读工具体系、以及 Web UI 的声明式组合。

## Typert：类型图生成器与 `@Remote` 方法

Typert 是 dsh 的远程调用基础设施：一个**类型图生成器、loader 和运行时注册表**（`packages/typert`）。`packages/typert/protocol/src/types.ts` 定义了核心类型：

```ts
export interface TypertLookup<Host, Wire> {
  readonly [LOOKUP_HOST]: Host
  readonly [LOOKUP_WIRE]: Wire
}

export interface TypertContext<Wire> {
  readonly [CONTEXT_WIRE]: Wire
}

export interface TypertLookupMap {}
export interface TypertContextMap {}
export interface TypertRemoteMap {}
export interface TypertRemoteScopeMap {}
```

这些是 **merge-extensible 空 map**——插件通过声明合并往里面加条目。`TypertLookupMap` 声明 Host 对象类型与 wire 身份的关联，`TypertContextMap` 声明 scoped Context kind 与 wire 身份的关联。

业务服务用 `@Remote` 或 `@RemoteScope` 装饰器选择暴露给 Client 的方法——**未标记的方法不进生成的 Client 类型、不进运行时贡献、不能通过 `ctx.remote` 调用**。

**复杂 Host 对象不能直接过 wire**：业务包必须通过 `TypertLookupMap` 声明它们与 wire 身份的关联，并注册默认解析 provider（`ctx.typert.lookups`）。例如 Host 签名里的 `agent: Agent` 参数产生 `agentId` wire 字段，Gateway 在调用业务方法前把 id 解析回 Host 对象。

注册表在 resolver 卸载后**保留 lookup 的 wire 声明**——SRC 发现继续把参数分类为 lookup，失败时报 unavailable，而不是把 wire 值当普通业务对象接受。

`RemoteResult` 是每个 Remote 方法的标准返回类型：

```ts
export type RemoteResult<T> =
  | { readonly ok: true; readonly value: T }
  | { readonly ok: false; readonly error: RemoteFailure }
```

## API Gateway 与 Lookup/Context 声明

`packages/api`：`api-gateway`（`ctx.typert` 的 Host Gateway）+ remotes。

- **unary Remote 方法**：业务服务声明 → 构建生成 Host 和 Client 契约 → 调用复用 Connection RPC 和 `/api` route。
- **流式协议在本文档范围外**：session events、增量数据等可能用同一个 Connection，但不用 Remote 方法描述符。

## Session query / projection / telemetry：回放与上报

**Session query**（`session-query/`）：逻辑记录、有界精确事件读、关系 trace、语义过滤/文档、全文结果页。工具：`session_event_read`/`session_event_search`/`session_event_trace`/`session_search`/`session_trace`——五个只读工具把 provider cursor 藏起来，从不可变 calling agent session 授权每个结果。

**Projection**（`session-projection/`）：`SessionProjectionMap`、纯 `ProjectionDefinition` 单元、`ProjectionSnapshot` 的一致切面、change feed。

**Telemetry**（`session-telemetry/`）：出站 session 上报 seam：`SessionTelemetryRecord`/`SessionTelemetrySeverity`、`SessionTelemetrySink` 契约、`session-telemetry/record` redact waterfall。

**Session title**：持久标题快照 + cited source-message seqs + 异步 provider 契约。base bundle 里配置可见：`fallbackMaxWords: 5` / `maxTitleBytes: 80`。

## ACP（Agent Client Protocol）自动化服务器

`packages/acp` 是 **automation-only 的 Agent Client Protocol server**——自动化场景的机器决策。ACP provider 在 subagent 体系里是"委托给另一个产品"的后端之一。

## Web UI：`apps/web` + client modules + HMR

- **`apps/web`**：Vite 构建，`apps/web/src/main.ts` 是入口，dist 由 `apps/cli` 的 `dsh web` 服务。
- **`packages/client`**：Web 前端模块——`web`（shell 库）、`web-react`、`connection`（RPC）、`hmr`、`locale`、`modules`（`dsh.client` 声明：浏览器名册）、`runtime`、`schema-form`，以及几十个 `ui-*` 模块。
- **`client-modules`**：`dsh.client` 声明 + `WebBootGraph` wire 组合 + bundle route + index tap（`tapIndex` 把 boot manifest 注入 index.html）。
- **Web server**（`host/webserver`）：单 `node:http` 插件提供 `ctx.webServer`：named-route 注册表（exact 先、最长 prefix 次、fallback 最后）。**无 TLS、无 auth、无 origin policy**——非 loopback 绑定就把服务器暴露给那个网络。
- **Electron**：加载 built 文件走 `file://`，fetch 请求走 IPC bridge（"It serves browsers only"）。

## 架构速览

```
远程调用与可观测性
┌─────────────────────────────────────────────┐
│  Typert (类型图生成器 + 远程调用)              │
│  @Remote / @RemoteScope 装饰器               │
│  Host对象 ↔ wire身份 分离                    │
│  TypertLookupMap / TypertContextMap          │
│  (merge-extensible 空 map)                  │
├─────────────────────────────────────────────┤
│  API Gateway                                │
│  unary Remote → Connection RPC + /api route │
├─────────────────────────────────────────────┤
│  Session query / projection / telemetry     │
│  5个只读工具 (session_event_read/search/...) │
│  projection: 一致切面 + change feed          │
│  telemetry: redact waterfall                │
├─────────────────────────────────────────────┤
│  ACP (自动化服务器)                           │
│  automation-only 机器决策                    │
├─────────────────────────────────────────────┤
│  Web UI                                     │
│  apps/web (Vite) + packages/client          │
│  client-modules 声明式组合                    │
│  webserver: exact → prefix → fallback       │
│  无 TLS / 无 auth / 无 origin policy         │
└─────────────────────────────────────────────┘
```

## 关键设计决策

1. **Host 对象 ↔ wire 身份分离**：`TypertLookupMap` 让 Host 对象（如 `Agent`）通过 branded id（如 `agentId`）过 wire。Gateway 在调用业务方法前把 id 解析回 Host 对象。这让远程调用的序列化和反序列化是类型安全的。

2. **未标记的方法不可达**：只有 `@Remote` 或 `@RemoteScope` 装饰的方法才暴露给 Client。这让业务服务可以有内部方法（不暴露）和远程方法（暴露），而不需要在运行时做权限检查。

3. **Session query 是只读的**：5 个只读工具（`session_event_read`/`search`/`trace`/`session_search`/`trace`）把 provider cursor 藏起来。这让 UI 可以查询 session 数据，但不能修改它。

4. **Web server 无 TLS / 无 auth**：默认绑定 `127.0.0.1`（loopback），故意暴露时才绑 `0.0.0.0`。这是一个有意识的安全取舍——简化本地开发，但非 loopback 绑定就把服务器暴露给网络。

## 本章小结

- Typert：装饰器驱动的远程方法，Host 对象 ↔ wire 身份分离，未标记方法不可达。
- API Gateway：unary Remote 复用 Connection RPC 与 `/api` route。
- 查询/投影/遥测：session-query 只读工具、projection 一致切面、telemetry redact waterfall。
- ACP：自动化场景的机器决策服务器。
- Web UI：client modules 声明式组合 + webserver 严格路由。

## 阅读思考题

1. `TypertLookupMap` 在 resolver 卸载后保留 wire 声明。如果一个业务包被卸载了，但 Client 仍然尝试调用它的 Remote 方法，会发生什么？
2. Session query 的 5 个只读工具从 calling agent session 授权每个结果。如果一个子 agent 尝试查询父 agent 的 session 数据，会被允许吗？
3. Web server 无 TLS / 无 auth。如果用户在 `0.0.0.0` 上启动 dsh web，同一网络的任何人都能完全控制 Agent。dsh 有什么机制提醒用户这个风险？

> **下一章预告**：第 18 章将讲 dsh 的测试与可信度体系——keyless snapshot 测试、运行时不变量注册表、每文件 100% 覆盖率门禁。
# 第 18 章　测试与可信度：怎么知道改好了

dsh 在"可信度"上的投入可能是所有已解读 Agent 里最重的：keyless snapshot 测试、运行时不变量注册表、每文件 100% 覆盖率门禁、防御模式文档、几十个 verify-* 门禁脚本。这一章讲它的工程可信度体系。读完后，你应该能理解为什么"行为变更必须带可重放证据"是 dsh 测试哲学的核心，以及从 dsh 可以借鉴的最小清单。

## 分层测试：vitest 单测、e2e、keyless snapshot

dsh 的测试体系（`package.json` scripts）：

- **`test`**：vitest 单测。
- **`test:coverage`**：CI 覆盖率门禁——**`packages/*/*/src` 每文件 100% 覆盖率**。这是极端严苛的门槛，迫使每个分支都被测试覆盖。
- **`test:e2e`**：真实 API 测试，无 `DEEPSEEK_API_KEY` 自动跳过。
- **`test:snapshot`**：**keyless** ACP/headless 重放对比期望输出——不需要 API key 的模型行为测试。`test:snapshot:record` 重新录制（需要 key），`test:snapshot:refresh` 刷新。
- **`test:web` / `test:web:perf` / `test:web:stress` / `test:gui`**：Web/性能/压力/GUI 分层。
- **`check:all` / `check:ci`**：`scripts/run-gates.ts` 的完整门禁编排。

**关键测试哲学**（`AGENTS.md`）：

> **Tests describe behavior, not correctness.** Change obsolete behavior with its tests; explain why in the PR.

> Every non-trivial model- or product-user-visible behavior change adds or updates a keyless snapshot through a real runnable example in the same PR; package tests, e2e-only assertions, and mock-only fixtures do not substitute for the assembled application transcript. Fixtures must replay on macOS/Linux; fix fixtures, not normalizers.

**keyless snapshot 是硬要求**：任何非平凡的模型/产品用户可见行为变更，必须带一个真实可运行的 example + keyless snapshot——mock-only fixtures 不算数。"Fix fixtures, not normalizers"——修 fixture，不修 normalizer（防止掩盖真实差异）。

## 运行时不变量注册表（invariants）

`docs/subsystems/invariants.md`：运行时 invariant 注册表——选择 `Config`、`InvariantInstaller`/`InvariantFailure`、empty-companion 契约。

`AGENTS.md` 的核心规则：

> **Runtime invariants assert owned relationships.** Check authoritative event streams or mutable data, not service or method presence, plugin metadata or effects, or fixed pure examples. Without a plausible relationship, an explained empty companion is correct.

**不变量断言"拥有的关系"**——检查权威事件流或可变数据，而不是检查"服务/方法存在"、"插件元数据/effect"或固定纯例子。没有合理关系时，解释过的空同伴是正确的。

最典型的运行时不变量就是 "Model-visible ⟺ logged"——任何到达模型请求的东西必须能从 session 日志重建。这类不变量被写进代码、在运行时断言，而不是只停留在文档里。

## 防御模式（defensive patterns）与 fail loud 哲学

`docs/defensive-patterns.md` 是生命周期、并发、subprocess、teardown 工作的必读。`AGENTS.md` 的防御规则精选：

- **Misconfiguration fails loud**：装载期自包含的错立刻报；否则在最早可解析点报。**绝不静默跳过缺失的引用**。
- **Opaque cross-boundary ids are branded**：跨边界的 id 用 `Branded<B>`，绝不用裸 `string`。`packages/core/session/src/types.ts` 里的 `SessionId`、`packages/subagent/subagent/src/types.ts` 里的 `SubagentRunId`、`packages/workflow/workflow/src/types.ts` 里的 `WorkflowRunId` 全部是 branded。
- **Trust TypeScript at typed same-process boundaries**：不在静态接口要求处加运行时校验/fallback/敌对输入测试；在 parser/config、queued、model/tool JSON、durable/file、worker、process、wire 边界校验。
- **An empty `catch` names what it swallows**：空 catch 必须说明吞掉了什么、为什么没有别的东西能到达它；`try` 只包一个语句。
- **Fail loud 的启动**：`installFailLoud`——启动失败 = 一行带标签 stderr + exit(1)，先恢复终端再退出。
- **Approval fail-closed**：应答者出任何问题都 `unavailable`，门是关的。
- **Sandbox 诚实报告**：`partial` 就是 `partial`，不假装更安全。

## 从 DeepSeek Harness 借鉴：插件化 Harness 的最小清单

把 dsh 的方法论压缩成一份可抄的清单：

1. **没有特权核心**：主循环、工具、日志、适配器全部是插件；扩展 = 在旁边挂插件，不是打补丁。
2. **注册即 effect**：每个贡献返回 disposer；卸载自动回卷——热更新与失败清理共享一套逻辑。
3. **Model-visible ⟺ logged**：模型看到的一切必须能从事件日志重建；新输入先加新事件。
4. **Capability Seam 三件套**：Definition / Provider / Consumer 一起设计，换 Provider 换产品。
5. **工具只返回规范值**：呈现由管线负责；policy/hooks/sandbox 通过事件瀑布插入，不改工具、不改循环。
6. **显式默认**：`resolve(request): Spec` 把默认值显式落定，部署差异 = 配置差异。
7. **fail loud + fail closed**：配置错大声报，审批/沙箱缺应答者时关门。
8. **诚实报告**：沙箱 enforcement `full`/`partial`、快照可重放、UI 呈现纯函数。
9. **keyless snapshot + 每文件 100% 覆盖**：行为变更必须带可重放证据。
10. **文档与源码同源**：生成的 catalog + type-equiv 漂移检查，文档永不过期。

## 架构速览

```
可信度三件套
┌─────────────────────────────────────────────┐
│  测试                                        │
│  vitest 单测 + 每文件 100% 覆盖率             │
│  keyless snapshot (真实 example 重放)         │
│  e2e (需要 API key) + web/perf/stress/gui    │
├─────────────────────────────────────────────┤
│  不变量                                      │
│  运行时注册表, 断言拥有的关系                   │
│  Model-visible ⟺ logged (核心)               │
│  检查权威事件流, 不检查存在性                   │
├─────────────────────────────────────────────┤
│  防御模式                                     │
│  fail loud (配置错大声报)                      │
│  fail closed (审批/沙箱缺应答者关门)            │
│  branded ids (跨边界 id 品牌化)                │
│  边界校验 (parser/config/wire/...)            │
│  诚实报告 (沙箱 enforcement)                   │
└─────────────────────────────────────────────┘
```

## 关键设计决策

1. **每文件 100% 覆盖率门禁**：这是极端严苛的门槛。它迫使每个分支都被测试覆盖，包括错误路径和边界条件。代价是开发速度变慢，收益是代码的可靠性极高。

2. **keyless snapshot 是硬要求**：mock-only fixtures 不算数。行为变更必须带真实可运行的 example + 重放对比。这让"改了什么"有可验证的证据，而不是"我相信它没坏"。

3. **不变量断言关系，不检查存在性**：检查"服务是否注册"不如检查"事件流是否一致"有意义。这让不变量能发现真正的问题（数据不一致），而不是误报（服务暂时不可用）。

4. **branded ids 是编译期安全**：`Branded<'SessionId'>` 在运行时就是 `string`，但在 TypeScript 的类型系统里是不同的类型。这让"把 SessionId 传给需要 SubagentRunId 的函数"成为编译错误，而不是运行时 bug。

## 本章小结

- dsh 的可信度体系是三件套：测试 + 不变量 + 防御模式。
- 每文件 100% 覆盖率门禁 + keyless snapshot（真实 example 重放）。
- 运行时不变量断言拥有的关系，"Model-visible ⟺ logged"是核心。
- fail loud、fail closed、branded ids、边界校验、诚实报告。
- 一句话总结 dsh 的工程性格：**激进插件化 + 偏执的可信度**。

## 阅读思考题

1. 每文件 100% 覆盖率门禁在什么场景下可能过于严格？比如一个只有一行的 re-export 文件，也需要 100% 覆盖吗？dsh 用 `v8 ignore` 注释处理这种情况——这种做法是"务实"还是"作弊"？
2. keyless snapshot 要求"Fixtures must replay on macOS/Linux; fix fixtures, not normalizers"。如果一个测试在 macOS 上通过但在 Linux 上失败（比如路径分隔符不同），dsh 的做法是"修 fixture"而不是"加 normalizer"。为什么？
3. dsh 的可信度清单有 10 条。如果只能选 3 条应用到你的项目，你会选哪 3 条？为什么？

> **全书完**。这 18 章从"什么是 Harness"开始，一路穿过 Cordis 的五块基石、启动组装线、事件日志、主循环、LLM 抽象、工具管线、能力接缝、沙箱、审批、持久化、上下文、子 Agent、Workflow、远程调用，最后收束于可信度体系。dsh 的工程哲学可以浓缩为一句话：**一切皆插件，注册即 effect，模型可见即日志**。
