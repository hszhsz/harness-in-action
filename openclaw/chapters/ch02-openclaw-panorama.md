# 第 2 章　OpenClaw 全景：一个生产级 Agent 长什么样

> 在拆开任何一个零件之前，先看清整台机器的疆域。
> 这一章给你一张地图——读完它，你应该能在脑中标出"如果我要找审批逻辑/工具定义/状态存储，该去哪个目录"。

第 1 章我们论证了一件事：Agent 的主战场在 Model 之外的 Harness 里。这一章我们就把 OpenClaw 这套 Harness 的"全身骨架"摊开来看一遍。注意，本章的目标**不是**把每个文件讲清楚（那是后面二十章的事），而是建立三样东西：**整体疆域感**、**设计哲学**、以及那几条决定了它一切形态的**工程红线**。

---

## 2.1 项目定位与设计哲学

OpenClaw 用一句话定义自己：

> **OpenClaw is the AI that actually does things. It runs on your devices, in your channels, with your rules.**
> （OpenClaw 是一个真正能做事的 AI，运行在你的设备上、你的渠道里、按你的规则。）[[OpenClaw VISION]](https://github.com/openclaw/openclaw/blob/main/VISION.md)

这句话每一个词都有分量，值得逐一拆解，因为它直接决定了代码的形状：

- **"actually does things"（真正能做事）** —— 它不是一个聊天机器人，而是一个能在真实计算机上执行真实任务的 Agent。这意味着它必须有强大的工具系统（第 8–9 章）和兜底的安全护栏（第 10–11 章）。
- **"on your devices"（在你的设备上）** —— 它是本地优先的，因此有了 macOS / iOS / Android 等 6 个 companion app（`apps/` 目录）和一个终端优先的形态。
- **"in your channels"（在你的渠道里）** —— 它能接入 Discord、Telegram 等多种消息渠道，于是有了 channel 抽象（`src/channels/`）和 Gateway 层（第 20 章）。
- **"with your rules"（按你的规则）** —— 安全与可控是第一性的。OpenClaw 把 "Security and safe defaults" 列为项目第一优先级[[OpenClaw VISION]](https://github.com/openclaw/openclaw/blob/main/VISION.md)。

它的来历也值得一提：OpenClaw 起步于一个"学习 AI、做点真正有用的东西"的个人项目，经历了 Warelay → Clawdbot → Moltbot → OpenClaw 的数次更名与重塑[[OpenClaw VISION]](https://github.com/openclaw/openclaw/blob/main/VISION.md)。这段演化史解释了为什么它今天既有"个人助理"的轻盈基因，又有"生产系统"的工程纪律。

**为什么是 TypeScript？** 这是很多人会问的问题。OpenClaw 给出的理由很清醒：它本质上是一个**编排系统**（orchestration system）——核心是 prompts、tools、protocols、integrations，而不是重计算。选 TypeScript 是为了"keep OpenClaw hackable by default"：它流行、迭代快、易读易改易扩展[[OpenClaw VISION]](https://github.com/openclaw/openclaw/blob/main/VISION.md)。

> **给 Agent 开发者的第一个启示**：在动手写第一行代码前，先用一句话定义你的 Agent"是什么、为谁、按什么规则"。OpenClaw 这句定位语，之后会被你在代码里反复"撞见"——每一个目录都是它的某个分句的物化。

---

## 2.2 Monorepo 全景：疆域地图

OpenClaw 是一个 TypeScript 的 pnpm monorepo。在本书写作时（版本 `2026.6.2`），它的体量是：**21 个可复用 package、142 个扩展（extensions）、58 个 Skill、`src/` 下 69 个子模块**。这个规模足以暴露真实 Agent 系统的全部工程难题。

我们先看**顶层目录**。下面这张表是全书的"总索引"，建议折页：

| 顶层目录 | 一句话职责 | 对应 Harness 职责 / 后续章节 |
|---|---|---|
| `src/` | 核心运行时（TypeScript）：Agent、工具、会话、配置、安全、LLM、插件 | 贯穿全书 |
| `packages/` | 可复用库，有稳定契约：`agent-core`（循环）、`llm-core`/`llm-runtime`（模型）、`plugin-sdk` 等 | 第 4、6、15 章 |
| `apps/` | 各端 companion app：android / ios / macos / macos-mlx-tts / shared / swabble | 第 19–20 章背景 |
| `extensions/` | 142 个插件（内部名 extensions，对用户称 plugins）：模型 provider、channel、browser 等 | 第 15–17 章 |
| `skills/` | 58 个随包发布的 Skill（SKILL.md 单元） | 第 16 章 |
| `ui/` | Vite Web 前端，经 Gateway 连到运行时 | 第 20 章 |
| `docs/` | 源文档（发布到 docs.openclaw.ai），含架构文档 | 资料源 |
| `config/` | 仓库工具链配置（tsconfig / lint 等） | 第 22 章 |
| `security/` | 随包发布的 OpenGrep 安全规则包 | 第 11 章 |
| `qa/` | 评估场景库（agents / models / memory / security 等 16 类） | 第 21 章 |
| `scripts/` | 构建/检查/基准脚本（含 66 个 `check-*` 守卫） | 第 21–22 章 |
| `test/` | 跨切面边界测试（架构坏味道、导入边界） | 第 21 章 |
| `deploy` / `patches` / `git-hooks` | 部署、依赖补丁、Git 钩子 | 工程配套 |

再往里看 **`src/` 的 69 个子模块**。不必现在记住每一个，但请感受一下它的"分区逻辑"——它几乎是按 Harness 的六职责 + 多端接入来切分的：

- **执行内核**：`agents/`（OpenClaw 专属 Agent 驱动）、`auto-reply/`（消息驱动入口）、`flows/`、`tasks/`、`commitments/`
- **模型与工具**：`llm/`（provider 实现）、`model-catalog/`、`provider-runtime/`、`tools/`（统一工具计划）、`talk/`（子 Agent 咨询）
- **上下文与状态**：`context-engine/`、`memory/` + `memory-host-sdk/`、`sessions/`、`state/`、`transcripts/`、`trajectory/`
- **安全与可控**：`security/`、`secrets/`、`process/`（exec 沙箱）、`node-host/`
- **扩展体系**：`plugins/`（加载器）、`plugin-sdk/`、`plugin-state/`、`skills/`、`mcp/`、`acp/`
- **多端接入**：`tui/`（终端）、`gateway/`、`channels/`、`web/`、`chat/`、`interactive/`
- **能力模块**：`media-generation/`、`image-generation/`、`web-search/`、`web-fetch/`、`link-understanding/`、`tts/`、`realtime-transcription/` 等
- **基础设施**：`config/`、`bootstrap/`、`cli/`、`commands/`、`daemon/`、`cron/`、`routing/`、`logging/`、`infra/`、`i18n/`、`utils/`、`shared/`

> **如何用这张地图**：当你在后续章节读到某个文件路径时，先回到这里定位它属于哪一区。比如读到 `src/security/dangerous-tools.ts`，你立刻知道这是"安全与可控"区，对应职责⑥。空间感建立起来，知识就不会是散点。

---

## 2.3 四条工程红线：约束即设计

OpenClaw 最值得 Agent 开发者偷师的，其实不是某个具体实现，而是它**用一组严苛的工程纪律来约束整个系统的形状**。这些纪律写在根目录的 `AGENTS.md`（"Root rules only. Skills own workflows; root owns hard policy and routing."[[OpenClaw VISION]](https://github.com/openclaw/openclaw/blob/main/AGENTS.md)）和 `VISION.md` 里，并由 66 个 `check-*` CI 脚本强制守卫。

我把其中最核心的四条提炼为"四条红线"。**每一条红线背后，都是一个值得你借鉴的架构决策。**

### 红线一：核心精简，能力插件化（Core stays lean）

> "Core stays lean; optional capability should usually ship as plugins. We are generally slimming down core while expanding what plugins can do."[[OpenClaw VISION]](https://github.com/openclaw/openclaw/blob/main/VISION.md)

OpenClaw 的设计原则是：核心只保留"必须在核心里的东西"，一切可选能力都尽量做成插件。它甚至明确列出了"为了维持精简而拒绝合并"的清单（What We Will Not Merge），包括能放到 ClawHub 的新 Skill、重复造轮子的编排层等[[OpenClaw VISION]](https://github.com/openclaw/openclaw/blob/main/VISION.md)。

插件还分两种风格，优先级很明确：

- **Bundle 插件**（首选）：打包 Skill、MCP server、配置等外部稳定面，接口更小、安全边界更好。
- **Code 插件**：当能力需要运行时 hook、provider、channel、tool 等进程内扩展点时才用。

**借鉴价值**：决定"什么进核心、什么做成插件"是 Agent 架构最重要的早期决策之一。核心越臃肿，迭代越慢、攻击面越大。OpenClaw 把"加东西进核心的门槛"刻意设得很高。

### 红线二：配置只认当前 Schema，迁移交给 doctor

> "Runtime reads canonical config only. No silent compat for old/malformed config keys. If a config change invalidates existing files, add a matching `openclaw doctor --fix` migration."[[OpenClaw VISION]](https://github.com/openclaw/openclaw/blob/main/AGENTS.md)

这条红线很反直觉，却极有价值。大多数项目会在运行时代码里堆满"如果是旧字段就这样、如果是新字段就那样"的兼容分支，久而久之运行时代码被历史包袱拖垮。OpenClaw 的做法是：

- **运行时只读当前规范形态**，绝不在运行时做静默兼容。
- 配置变更若让旧文件失效，**必须**配一个 `openclaw doctor --fix` 迁移：它负责检测旧形态、解释、备份、改写成规范格式。
- 历史/退役形态的归一化代码**只活在 doctor/migration 里**，运行时没有任何 shim、别名或 fallback reader。

它对新增配置项的门槛也极高：在加一个配置或环境变量前，必须先证明"现有行为、provider 选择、默认值或 doctor 迁移都解决不了它"[[OpenClaw VISION]](https://github.com/openclaw/openclaw/blob/main/AGENTS.md)。

**借鉴价值**：把"向后兼容"这件脏活集中到一个专门的迁移层，让运行时代码永远只面对"一种世界"。代价是每次破坏性变更都要写迁移；收益是运行时代码长期保持干净、可推理。

### 红线三：状态只用 SQLite

> "Storage default: SQLite only. Do not add JSON/JSONL/TXT/sidecar files for OpenClaw-owned runtime state, caches, queues, registries, indexes, cursors, checkpoints, or plugin scratch data."[[OpenClaw VISION]](https://github.com/openclaw/openclaw/blob/main/AGENTS.md)

OpenClaw 对状态存储做了一个干净利落的统一决策——**所有运行时状态一律进 SQLite**，禁止散落的 JSON/JSONL/sidecar 文件。具体落地为：

- **全局状态库** `state/openclaw.sqlite`：全局运行时状态与插件 KV 数据。
- **per-agent 库** `agents/<agentId>/agent/openclaw-agent.sqlite`：Agent 作用域的状态/缓存。
- 运行时访问 SQLite 用 Kysely helper 而非裸 SQL 字符串（DDL/迁移/底层 bootstrap 除外）。
- 文件存储只能是"具名产品产物"（导入导出、用户附件、日志、备份），凡是 app 状态或缓存，一律归 SQLite。

**借鉴价值**：状态存储的碎片化是 Agent 系统腐化的常见根源——这里一个 JSON、那里一个缓存文件，最后没人说得清状态在哪、怎么迁移。一个统一的、事务性的、可查询的存储后端，是"可复现"这一 Harness 特征的地基（这是职责④，详见第 14 章）。

### 红线四：拒绝默认的多 Agent 层级

> 在 "What We Will Not Merge" 清单中明确列出：**"Agent-hierarchy frameworks (manager-of-managers / nested planner trees) as a default architecture"** 与 "Heavy orchestration layers that duplicate existing agent and tool infrastructure"[[OpenClaw VISION]](https://github.com/openclaw/openclaw/blob/main/VISION.md)。

这是一个非常清醒、也非常"逆潮流"的决策。在"多 Agent"概念被热炒的当下，OpenClaw 明确表态：**不把"经理管经理"的嵌套规划树作为默认架构。** 它支持子 Agent 委派（ACP，第 18 章），但刻意把它约束在有限范围内，而不是默认就搭一套层层下派的编排框架。

**借鉴价值**：多 Agent 不是越多越好。每多一层委派，就多一层不可控、不可观测、成本翻倍。OpenClaw 的态度提醒我们：在没有被真实问题逼到非用不可之前，优先把单 Agent 的 Harness 做扎实。

> 这四条红线不是"法律"，OpenClaw 自己也说这是 "a roadmap guardrail, not a law of physics"——强需求加强技术理由可以改变它[[OpenClaw VISION]](https://github.com/openclaw/openclaw/blob/main/VISION.md)。但作为一套被实践检验过的工程纪律，它们值得每一个 Agent 开发者认真思考"我要不要也立几条这样的红线"。

---

## 2.4 如何高效阅读这套代码

最后，给你一套读这套代码的实用方法，它来自 OpenClaw 自己的 `AGENTS.md` 约定。

**第一，理解 `AGENTS.md` 的层级机制。** 根目录 `AGENTS.md` 只放"硬策略与路由"，采用 telegraph 风格（极简电报体）。每个子目录可以有自己的 scoped `AGENTS.md`，进入子树工作前应先读它[[OpenClaw VISION]](https://github.com/openclaw/openclaw/blob/main/AGENTS.md)。这其实就是 OpenClaw 自己的"上下文工程"——它把给 AI 的规则也做了分层。（顺带一提，根目录的 `CLAUDE.md` 是 `AGENTS.md` 的软链接，二者内容相同。）

**第二，记住 66 个 `check-*` 脚本是"活的架构文档"。** 当你想知道"这个项目到底有哪些不可违反的约束"时，`scripts/check-*.mjs` 比任何文档都诚实。例如：

- `check-architecture-smells.mjs` —— 架构坏味道检测
- `check-deadcode-unused-files.mjs` —— 死代码检测
- `check-cli-startup-memory.mjs` —— 启动内存检测
- `check-codex-app-server-protocol.ts` —— 协议一致性检测

这些脚本把前面那些"红线"从口头约定变成了 CI 里的硬门禁——这正是第 1 章所说"用工程手段让错误不可能再发生"的活例子。

**第三，善用项目自带的导航工具。** `AGENTS.md` 提到用 `pnpm docs:list` 列出文档再按需读取[[OpenClaw VISION]](https://github.com/openclaw/openclaw/blob/main/AGENTS.md)；架构层面优先读 `docs/agent-runtime-architecture.md` 和 `docs/openclaw-agent-runtime.md`，它们是理解运行时的官方入口。

**第四，跟着本书的执行链路读，而不是目录字母序。** 重申第 1 章的主线：从 `openclaw.mjs` 入口出发，顺着请求穿过运行时、主循环、模型层、工具层、安全层、上下文层、状态层、扩展层，最后到多端接入。下一章我们就正式踏上这条链路的第一站——**进程入口**。

---

## 本章小结

- OpenClaw 用一句话定义自己——"真正能做事的 AI，在你的设备上、渠道里、按你的规则"——这句话物化成了整个目录结构。
- 它是一个 TypeScript pnpm monorepo（21 个 package、142 个扩展、58 个 Skill、`src/` 下 69 个模块），规模足以代表"真实生产级 Agent"。
- 顶层目录与 `src/` 子模块大体按 Harness 六职责 + 多端接入来分区；本章的两张表是全书的总索引。
- 四条工程红线——核心精简、配置只认当前 Schema、状态只用 SQLite、拒绝默认多 Agent 层级——是 OpenClaw 最值得偷师的部分，每条背后都是一个架构决策。
- 读这套代码的钥匙是 `AGENTS.md`（分层硬策略）与 66 个 `check-*` 脚本（把红线变成 CI 门禁）。

下一章起进入第二部分，我们将从进程入口 `openclaw.mjs` 开始，一步步走到 Agent 的执行内核——心脏第一次跳动的地方。

---

> **动手实验（建议在读第 3 章前完成）**
> 1. 在 clone 下来的仓库里运行 `ls src/` 与 `ls packages/`，对照本章 2.2 的分区表，把每个目录归到六职责中的某一类。遇到归不进去的（比如 `crestodian/`、`swabble`），记下来——后面章节会揭晓。
> 2. 打开 `scripts/` 目录，挑 3 个 `check-*` 脚本读一读它们的注释与报错信息，反推它们在守卫哪一条"红线"。这是理解一个项目"真实约束"最快的方式。
