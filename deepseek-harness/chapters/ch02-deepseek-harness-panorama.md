# 第 2 章　DeepSeek Harness 全景：一个插件树的疆域

> 在拆开任何一个零件之前，先看清整台机器的疆域。这一章给你一张地图——读完它，你应该能在脑中标出"如果我要找主循环/工具定义/沙箱策略，该去哪个目录"。

## 2.1　项目定位与设计哲学

DeepSeek Harness 用一句话定义自己：

> DeepSeek Harness（`dsh`）是由 DeepSeek AI 开发的开源 agent harness（智能体框架）。它采用**一切皆插件**的架构，并由 Cordis 驱动。[[README.zh.md]](https://github.com/deepseek-ai/deepseek-harness/blob/main/README.zh.md)

它目前的阶段是**开发者预览**（`0.1.0-rc.5`），官方明确警告"未来将出现破坏兼容性的变更"。这一点非常重要——它塑造了仓库里几乎每一个工程决策。`AGENTS.md` 开篇就写：

> **Pre-release stance: foundation over blast radius.** With no external consumers, prefer the correct foundation over compatibility shims: rename or repackage freely and update every reference together. Backends reject old on-disk formats. SQLite uses monotonic `SCHEMA_VERSION`; `dsh-session` keeps `SESSION_FORMAT_VERSION` at `0` with no compatibility promise.

翻译过来：**没有外部消费者，所以选"正确的基础"而不是"兼容的权宜"**。想改名就改名，想重构就重构，磁盘格式说换就换，SQLite schema 用单调递增的 `SCHEMA_VERSION` 管理，session 格式版本号永远停在 `0`（不承诺兼容）。这是一种只有全新项目才配拥有的自由——也解释了为什么 dsh 的代码如此"干净"：它没有历史包袱。

**为什么是 TypeScript？** 与 OpenClaw 的理由类似：这是一个编排系统——核心是 prompts、tools、protocols、integrations，而不是重计算。但它比 OpenClaw 更进一步：**连编排框架本身（Cordis）都被 vendoring 进 `vendor/`**，整个团队围绕一个自己掌控的插件框架设计产品。

## 2.2　Monorepo 全景：疆域地图

在写作时，dsh 是一个 pnpm monorepo（`pnpm@11.7.0`），**`packages/` 下 54 个顶层包**，`packages/` 内 TypeScript 源码约 **19.8 万行**。这个规模足以暴露真实 Agent 系统的全部工程难题。

我们先看**顶层目录**。下面这张表是全书的"总索引"：

| 顶层目录 | 一句话职责 | 对应 Harness 职责 / 后续章节 |
|---|---|---|
| `vendor/` | vendored Cordis 源码拷贝（manifest + 同步流程） | 第 3 章 |
| `packages/` | `@deepseek-ai/dsh-<pkg>` 工作区：core / llm / shell / fs / sandbox / subagent / workflow / bundle / … | 贯穿全书 |
| `apps/` | 两个应用：`cli`（`dsh` 命令）+ `web`（Vite 前端入口） | 第 4、17 章 |
| `native/` | `@deepseek-ai/node-addon-landlock-run`：Linux Landlock 的 node addon | 第 11 章 |
| `python/` | Python SDK 与 bundled runtime（`python/sdk` + `python/sdk-runtime`） | 附录 |
| `examples/` | 可运行的 `cordis.yml` 叶子：acp-agent / headless-agent / jsonrpc-agent / mcp-memory / web-cordis / web-schedule | 第 4 章 |
| `docs/` | 架构文档 + 逐子系统参考 + 生成的 catalog + postmortem | 全书索引 |
| `website/` | VitePress 对双语文档的投影 | — |
| `scripts/` | 仓库门禁（gates）与生成器 | 第 18 章 |
| `.agents/` | Agent 工作流与 Agent Notes（`notes/`，含 implemented/proposed 分类） | 第 18 章 |

**`packages/` 的五个组**（`packages/README.md` 有官方分组说明）：

1. **core**（产品 API 脊柱）：`session`（事件日志）、`system-prompt`（提示词组装）、`tools`（工具注册表）、`agent`（Agent 接口与事件）、`agent-loop`（默认主循环实现）、`scope`（per-agent 作用域原语）。
2. **能力 seam 组**：每个 seam 拆成 Service Definition + Provider + Consumer 三包——`fs`（`fs`/`fs-local`/`tool-fs`/`fs-sandbox`）、`shell`（`shell`/`bash-local`/`bash-sandbox`/`tool-bash`）、`sandbox`（`sandbox`/`sandbox-local`）、`subagent`（9 个包）、`workflow`（4 个包）、`code-runtime`、`lsp`、`web`、`skill`、`spill` 等。
3. **交互与治理**：`interaction/`（`user-approval`/`user-questions`/`commands`/`permission-presets`）、`guard/`（`timeout-policy`/`repeat-tool-reminder`）、`plan/`、`goal/`、`schedule/`、`credentials/`、`settings/`。
4. **会话与数据**：`session/`（`session-persistence`/`session-query`/`session-projection`/`session-telemetry`/`session-title`）、`compaction/`、`storage/`、`workspace/`。
5. **运行时与远程**：`boot/`（`app-boot`/`cmdline`）、`bundle/`（`base`/`web-app`/`headless`）、`api/`（gateway/remotes）、`typert/`、`sdk/`、`acp/`、`hooks/`（`hook-protocol`/`hooks-claude-code`/`hooks-codex`）、`extensions/`（自修改插件）、`llm/`（`llm`/`llm-deepseek`/`llm-pi-ai`/`llm-retry`/`token-meter`）、`client/`（Web 前端模块）、`host/`（webserver 等）。

注意 `packages/examples/` 里还有 demo bundles（`agent-spine-demo` / `acp-demo` / `jsonrpc-demo`）——它们是"默认组合"的参考实现。

## 2.3　三条工程红线

dsh 的 `AGENTS.md` 写了 40+ 条开发约定，但有三条决定了整个系统的形态，值得单独拎出来：

**红线一：一切皆插件，插件不是循环的补丁，循环本身是插件。**

> **Plugins, not loop changes**: new behavior goes on documented extension points; changing `agent-loop` requires updating docs/architecture.md.

新行为必须挂到文档化的扩展点上；要改主循环，必须先改架构文档。这把"架构漂移"变成了一个文档门禁。

**红线二：Model-visible ⟺ logged（模型可见 = 已记日志）。**

> **Model-visible ⟺ logged**: anything that reaches a model request must be reconstructable from the session log; a new model-visible input requires a session event.

任何到达模型请求的东西必须能从会话日志重建。反过来：一个新的模型可见输入，必须先新增一个 session event。这个不变量甚至被写进了运行时断言（第 5 章详述）。

**红线三：Registrations are effects（注册即副作用）。**

> **Registrations are effects**: every contribution goes through `ctx.effect()` / `ctx.on()`; a registry's `register()` returns the disposer.

每一个贡献（prompt section、工具 schema、适配器、provider、监听器）都必须走 `ctx.effect()` 或 `ctx.on()`，注册返回 disposer——卸载插件时全部自动回卷。这是"没有特权核心"能成立的地基：因为所有注册都可逆，所以任何插件都可以被安全地替换、重载、移除。

## 2.4　如何高效阅读这套代码

dsh 的文档体系是它最被低估的资产，按这个顺序读最高效：

1. **`docs/architecture.md`** —— 总纲。讲清 Cordis、profiles/bundles、事件分类、turn flow、capability seams、扩展点地图。
2. **`docs/cordis-primer.md`** —— Cordis 五概念入门（插件/上下文/注入/事件/effect）+ 四种派发模式。**读 dsh 源码前必读**。
3. **`docs/subsystems/README.md`** —— 每个子系统的索引表；每个子系统一页，包含从源码生成的 Cordis API 参考。
4. **生成的 catalog**：`docs/config-catalog.md`（配置项全集）、`docs/tool-catalog.md`（工具全集）、`docs/persistence-catalog.md`（持久化事件全集）、`docs/event-producer-consumer.md`（事件生产者/消费者图）、`docs/module-graph.md`（模块图）。
5. **`AGENTS.md`** —— 开发约定全集。如果你要贡献代码，这是必读；如果只是读代码，也强烈建议扫一遍，因为每条约定都解释了一个工程决策。

**生成器哲学**：`docs/subsystems/*.md` 里那些 "Cordis API" 段是 `scripts/gen-cordis-catalog.ts` 从源码生成的，有 `pnpm run verify-cordis-catalog` 检查新鲜度；类型声明用 `ts type-equiv` 标记，由 `pnpm run verify-type-equiv` 做漂移检查。这意味着**文档和源码永远不会脱节**——这在开源项目里极其罕见。

## 本章小结

DeepSeek Harness 是一个 54 包、约 20 万行 TypeScript 的插件化 Agent 系统：vendored Cordis 当骨架，core 五包当脊柱，几十个 capability seam 当肌肉，profile/bundle/patch 分层当组装线。它的工程性格是**激进插件化**——没有特权核心，一切皆可替换，连主循环本身都只是一个"默认实现"。三条红线（插件优先、模型可见即日志、注册即 effect）决定了它的一切形态。

## 动手实验

1. 运行 `pnpm dsh --profile web --dump-config`，把输出存下来，数一数这棵树有多少个 entry、分属哪些 bundle。
2. 打开 `docs/module-graph.md`，找到 `core/scope` 的模块位置——官方说它" sits below session and system-prompt precisely so they can consume it without a cycle"，验证一下这个说法。
3. 在 `packages/` 里找一个 seam（比如 `spill`），列出它的 Definition / Provider / Consumer 三个包，对照第 9 章的三件套模型。
4. 读 `vendor/README.md`，看看 Cordis 的 vendoring manifest 长什么样、同步流程是什么。
