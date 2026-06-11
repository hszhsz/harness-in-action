# 第 2 章　OpenCode 全景：一个生产级编码 Agent 长什么样

第 1 章我们建立了"Agent = Model + Harness"的心智模型。现在我们打开 OpenCode 的仓库，做一次全景俯瞰。本章的目标不是深入任何一个子系统，而是先在脑子里画出一张**地图**：这套代码怎么切分成包、v1 与 v2 如何并存、哪几条工程纪律贯穿始终、以及如何高效地阅读它。有了这张地图，后面每一章的"深入"才不会迷路。

## 2.1 一个 Bun + Effect 的 monorepo

OpenCode 的根 `package.json` 第一行就交代了它的技术底色：`"packageManager": "bun@1.3.14"`，`"type": "module"`。它是一个用 **Bun** 作为运行时与包管理器的 monorepo，通过 `workspaces` 把代码切成 `packages/*`、`packages/console/*`、`packages/stats/*` 等多个工作区。根目录还配了 `turbo.json`（用 Turborepo 编排构建）和一个庞大的 `catalog`——把所有依赖的版本号集中声明在一处，子包用 `catalog:` 引用。

在这个 catalog 里，有一个依赖的地位高于其他所有：`"effect": "4.0.0-beta.74"`。**Effect 是理解 OpenCode v2 内核的钥匙。** 它是一个把"副作用"（IO、错误、依赖、并发）都用类型系统显式表达的函数式框架。你会在 `packages/core/src` 里反复看到 `Effect.Effect<成功类型, 错误类型>` 这样的签名——错误类型被写进类型签名，意味着"这个操作可能失败、会以哪几种方式失败"在编译期就是可见的。比如第 4 章我们会读到的 `SessionRunner` 就声明了一长串联合错误类型 `RunError = LLMError | SessionRunnerModel.Error | MessageDecodeError | ...`。这是 OpenCode v2 把"可信度"焊进类型系统的第一处体现，呼应了我们方法论里"魔鬼在细节"的判断。

> 不熟悉 Effect 也不用怕。本书会在用到具体特性时随文解释，你只需先记住一个直觉：**在 Effect 里，一段逻辑的"会做什么副作用、会怎么失败、依赖什么服务"都写在类型里**，这让大型系统的约束变得可被编译器检查。

## 2.2 Monorepo 全景：每个包是干什么的

`packages/` 下有二十多个包。我们不必逐个细看，但要建立一个分层认知。按照第 1 章的六类职责，可以把核心的几个包归类如下：

| 包 | 角色 | 对应职责 |
|---|---|---|
| `opencode`（即 npm 上的 `opencode`） | v1 主运行时 + CLI 入口，源码在 `packages/opencode/src` | 编排 / 入口 |
| `@opencode-ai/core` | v2 内核，基于 Effect 重写，源码在 `packages/core/src` | 编排 / 上下文 / 状态 |
| `@opencode-ai/llm` | 模型协议适配层，多家 provider 的统一抽象 | 接入 |
| `@opencode-ai/cli` | v2 的 CLI 框架（`framework/runtime.ts`、命令 handler） | 入口 |
| `@opencode-ai/tui` | 终端 UI | 多端 |
| `@opencode-ai/server` | HTTP 服务、路由、SSE 事件 | 多端 |
| `@opencode-ai/sdk` | 自动生成的 JS SDK | 接入 |
| `@opencode-ai/plugin` | 插件 SDK 公共契约 | 扩展 |
| `@opencode-ai/enterprise` | 企业能力 | 扩展 |
| `app` / `web` / `desktop` / `console` | 各类前端与控制台 | 多端 |
| `effect-drizzle-sqlite` / `effect-sqlite-node` | 把 SQLite（经 Drizzle ORM）接入 Effect 的基础设施 | 状态 |

这张表透露了一个重要信号：**OpenCode 把"状态只用 SQLite"这件事郑重其事地做成了两个专门的基础设施包**（`effect-drizzle-sqlite`、`effect-sqlite-node`）。一个项目愿意为"如何优雅地在 Effect 里用 SQLite"单独抽包，说明持久化在它的工程版图里是头等公民。这一点我们留到第 12 章细看。

本书聚焦的是 Harness 内核，所以接下来的篇幅几乎都围着 `opencode`、`core`、`llm`、`cli`、`server` 这五个包转，其余前端包只在第 14 章谈多端接入时点到为止。

## 2.3 v1 与 v2：同一个项目里的两套实现

这是 OpenCode 最值得研究、也最容易让初次读者困惑的地方：**`packages/opencode/src` 和 `packages/core/src` 里有两套高度相似又彼此不同的实现。**

打开两个 `session` 目录对比，差异一目了然：

- v1：`packages/opencode/src/session` 下是 `session.ts`（1119 行）、`processor.ts`（1084 行）、`prompt.ts`（1704 行）、`compaction.ts`、`message-v2.ts` 等——一套已经服役、相当庞大的运行时。
- v2：`packages/core/src/session` 下出现了一批 v1 没有的新概念文件：`context-epoch.ts`、`projector.ts`、`run-coordinator.ts`、`runner/`、`execution/`。

v2 不是简单的重构，而是带着一套**全新领域语言**的重写。证据就是仓库根目录那份 `CONTEXT.md`——它像一份正式的领域词典，逐条定义 **System Context**、**Session History**、**Context Source**、**Context Epoch**、**Safe Provider-Turn Boundary** 等术语，甚至给出"该用哪个词、不该用哪个词"的规范（比如定义 System Context 时明确标注 `_Avoid_: System prompt`）。它还有一段"Example dialogue"，用一问一答的形式固化设计决策。**一个团队愿意为内部概念写这么严肃的领域词典，本身就说明 v2 是奔着"长期可演进"去的。**

`AGENTS.md` 末尾的"V2 Session Core"一节进一步把 v2 的硬约束写成了铁律，例如："Keep durable prompt admission separate from model execution"（把持久化的 prompt 准入与模型执行分离）、"Preserve one explicit `llm.stream(request)` call per provider turn"（每个 provider turn 只保留一次显式的 `llm.stream` 调用）、"Do not bridge through legacy `SessionPrompt.loop(...)`"（不要再走 v1 的 `SessionPrompt.loop`）。这些约束告诉我们：**v2 在刻意纠正 v1 的一些架构债**——尤其是把"接收用户输入"和"调用模型"这两件事彻底解耦。这正是我们在第 4、5 章要重点拆解的。

> **怎么读这两套？** 本书的策略是：讲清楚某个子系统时，**优先读结构更清晰、约束更显式的 v2（`core`）**，在 v2 尚未覆盖或 v1 仍是主力的地方（比如完整的工具实现、CLI 命令）再回到 v1（`opencode`）。两套对照，恰恰能让你看清"一个设计为什么要演进成这样"。

## 2.4 四条工程纪律

读一套大型代码前，先了解它的"家规"，能事半功倍。OpenCode 的 `AGENTS.md` 里写满了风格约束，其中几条直接影响你阅读代码的预期：

1. **早返回、不写 `else`**：`AGENTS.md` 明确要求 "Avoid `else` statements. Prefer early returns"。所以你在源码里会看到大量"校验失败就立刻 return/抛错，主流程一路平铺到底"的写法。读函数时，**把开头那一连串 early return 当成"前置条件清单"来读**，剩下的才是 happy path。
2. **happy path 在上，辅助函数在下**：约束 "make the main function read as the happy path and move supporting details into small helpers below it"。所以读一个文件时，先读顶部导出的主函数抓主线，细节助手函数在下方备查。
3. **不用 `any`、少用 `try/catch`、依赖类型推断**："Avoid using the `any` type"、"Avoid `try`/`catch` where possible"。在 v2 里，错误不靠 `try/catch` 满天飞，而是用 Effect 的类型化错误来表达——这也是为什么 `RunError` 那样的联合错误类型会出现在签名里。
4. **持久化用 snake_case 的 Drizzle schema**：约束要求表字段直接用 snake_case 命名，"so column names don't need to be redefined as strings"。所以读到 `sqliteTable("session", { project_id: text().notNull(), created_at: integer().notNull() })` 这种代码不要意外——字段名即列名。

还有一条隐藏的元规则值得记住：`AGENTS.md` 开篇就说 **"The default branch in this repo is `dev`"**，并提醒"Local `main` ref may not exist"。这意味着 OpenCode 的主线开发发生在 `dev` 分支上——本书引用的代码以撰写时的快照为准，你本地 clone 后看到的可能更新，**永远以你读到的源码为准**。

## 2.5 如何高效阅读这套代码

最后给一套实用的阅读姿势，本书后续章节都会沿用：

- **从入口顺流而下**：从 `packages/opencode/src/index.ts`（yargs CLI 定义）开始，跟着一个具体命令（比如 `run`）往下追，比孤立地读某个文件更容易建立全局观。第 3 章我们就这么做。
- **用领域词典当字典**：读 v2 代码遇到 `Context Epoch`、`Safe Provider-Turn Boundary` 这类术语，回 `CONTEXT.md` 查定义，能省下大量猜测。
- **对照 v1/v2 看演进**：同一个子系统两套实现并存时，对照读能看出设计意图。
- **盯住类型签名**：在 Effect 项目里，一个函数的 `Effect.Effect<A, E>` 签名几乎是一份微型规格说明——成功返回什么、可能怎么失败，一目了然。

带着这张地图和这套姿势，我们就可以下潜了。

## 本章小结

- OpenCode 是一个以 Bun 为运行时、用 Turborepo 编排、依赖集中在 `catalog` 的 monorepo；其 v2 内核构建在 Effect 之上，把副作用与错误类型显式写进签名。
- `packages/` 下二十多个包按职责分层，本书聚焦 `opencode`(v1 运行时)、`core`(v2 内核)、`llm`(协议适配)、`cli`、`server` 五个核心包。
- 项目专门为持久化抽了 `effect-drizzle-sqlite`、`effect-sqlite-node` 两个基础设施包，说明"状态只用 SQLite"是被认真对待的工程决策。
- OpenCode 正处于 v1→v2 迁移：v2 带着一份正式的领域词典 `CONTEXT.md` 和 `AGENTS.md` 里的"V2 Session Core"硬约束，核心是把"持久化 prompt 准入"与"模型执行"解耦。
- 四条可观察的工程纪律：早返回不写 `else`、happy path 在上助手在下、不用 `any` 少用 `try/catch`（错误走类型化表达）、Drizzle schema 用 snake_case。
- 高效阅读姿势：从 CLI 入口顺流而下、用领域词典查术语、对照 v1/v2 看演进、盯住 Effect 类型签名当规格说明。

## 动手实验

1. **实验一：核对你的代码地图** — 回到第 1 章实验一你画的"包→职责"猜想，对照本章 2.2 的表格批改。哪些猜对了？`effect-drizzle-sqlite` 你之前归到了哪一格？
2. **实验二：量化 v1 的体量** — 在仓库里运行 `wc -l packages/opencode/src/session/*.ts | sort -rn | head`，看看 v1 的 session 子系统里最大的几个文件有多少行（提示：`prompt.ts` 超过 1700 行）。体会一下为什么团队要启动 v2 重写。
3. **实验三：读懂一条类型化错误** — 打开 `packages/core/src/session/runner/index.ts`，找到 `RunError` 这个联合类型，把它包含的每一种错误成员列出来。对照 `AGENTS.md` 里"Avoid `try/catch`"那条，想一想：这种把错误写进类型的做法，相比到处 `try/catch`，对"可信度"有什么好处？
4. **实验四：验证一条家规** — 在 `packages/core/src` 里随便挑三个 `.ts` 文件，数一数 `else` 关键字出现了几次。再看看函数开头是不是大量用 early return。亲手验证 `AGENTS.md` 的"Avoid `else`"纪律是否被真正执行。

> **下一章预告**：地图已经在手。第 3 章我们就从最外层切入——跟随一条 `opencode run "..."` 命令，看进程是如何启动的：`bin/opencode` 启动器、`src/index.ts` 的 yargs 命令树、以及一条命令如何被路由到对应的 handler，最终唤醒 Session 运行时。
