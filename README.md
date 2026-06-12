# Harness in Action

开源 AI 编码 Agent 的源码解读合集，外加从 **harness engineering（执行骨架工程）** 视角出发的深度对比文章。本仓库持续更新——会不断收录新的 Agent 解读与对比分析。

> Harness 是包裹在大模型外面的那层工程——它决定模型能调什么工具、看到什么上下文、在什么边界上被打断、出错后怎么恢复、副作用能不能落地。同一个 LLM，套上不同的 harness，就长成性格迥异的 Agent。本仓库把各个 Agent 的逐章解读汇总到一处，方便横向对照。

## 当前收录

目前已收录 5 个 Agent 的完整逐章解读：

| 目录 | Agent | 语言/栈 | 篇幅 |
|---|---|---|---|
| [`openclaw/`](./openclaw/) | OpenClaw | TypeScript | 17 章 |
| [`hermes/`](./hermes/) | Hermes | Python | 15 章 |
| [`opencode/`](./opencode/) | OpenCode | TypeScript + Effect | 16 章 |
| [`codex/`](./codex/) | Codex | Rust | 17 章 |
| [`pi/`](./pi/) | pi | TypeScript | 16 章 |
| [`analysis/`](./analysis/) | — | 横向对比文章 | — |

## 核心对比文章

📄 **[同一个心脏，四种活法：OpenClaw / Hermes / OpenCode / Codex 的 Harness 工程对比](./analysis/harness-engineering-four-agents-comparison.md)**

从十个工程维度切开四个 Agent：整体工程性格、主循环与控制流、准入与执行分离、工具系统、执行与沙箱、权限与审批、上下文压缩与记忆、持久化、扩展体系/网关/可观测性、测试与可信度。

## 一句话画像

| Agent | 语言/栈 | 工程性格 |
|---|---|---|
| **OpenClaw** | TypeScript | 偏执型 fail-closed——"模型只能加锁，不能开锁" |
| **OpenCode** | TypeScript + Effect | 严谨型——"宁可受限也不冒险"，把不变量焊进类型 |
| **Codex** | Rust | 保守诚实型——"在每个边界上选择诚实与保守" |
| **Hermes** | Python | 务实粗粝型——"为无人值守、多平台、长时运行而生" |
| **pi** | TypeScript | 自扩展、把约束外包——核心只做精简执行骨架，权限/沙箱交给扩展与容器 |

## 仓库组织约定

为方便后续持续扩展，新增内容遵循以下约定：

- **新增 Agent 解读**：在根目录建一个以 Agent 命名的子目录，内含 `chapters/`（逐章解读）与该 Agent 的 `README.md`。
- **新增对比文章**：放入 [`analysis/`](./analysis/) 目录。
- **更新本 README**：在"当前收录"表与"一句话画像"表补上新条目，保持导航同步。
