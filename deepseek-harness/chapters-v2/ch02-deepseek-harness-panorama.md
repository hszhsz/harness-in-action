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
