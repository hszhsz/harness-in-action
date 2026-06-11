# Harness in Action

四个开源 AI 编码 Agent 的源码解读合集，外加一篇从 **harness engineering（执行骨架工程）** 视角出发的深度对比文章。

> Harness 是包裹在大模型外面的那层工程——它决定模型能调什么工具、看到什么上下文、在什么边界上被打断、出错后怎么恢复、副作用能不能落地。同一个 LLM，套上不同的 harness，就长成性格迥异的 Agent。本仓库把四个 Agent 的逐章解读汇总到一处，方便横向对照。

## 目录结构

| 目录 | 内容 |
|---|---|
| [`analysis/`](./analysis/) | 四 Agent 的 harness 工程深度对比文章 |
| [`openclaw/`](./openclaw/) | OpenClaw（TypeScript）逐章解读（17 章） |
| [`hermes/`](./hermes/) | Hermes（Python）逐章解读（15 章） |
| [`opencode/`](./opencode/) | OpenCode（TypeScript + Effect）逐章解读（16 章） |
| [`codex/`](./codex/) | Codex（Rust）逐章解读（17 章） |

## 核心对比文章

📄 **[同一个心脏，四种活法：OpenClaw / Hermes / OpenCode / Codex 的 Harness 工程对比](./analysis/harness-engineering-four-agents-comparison.md)**

从十个工程维度切开四个 Agent：整体工程性格、主循环与控制流、准入与执行分离、工具系统、执行与沙箱、权限与审批、上下文压缩与记忆、持久化、扩展体系/网关/可观测性、测试与可信度。

## 四个 Agent 的一句话画像

| Agent | 语言/栈 | 工程性格 |
|---|---|---|
| **OpenClaw** | TypeScript | 偏执型 fail-closed——"模型只能加锁，不能开锁" |
| **OpenCode** | TypeScript + Effect | 严谨型——"宁可受限也不冒险"，把不变量焊进类型 |
| **Codex** | Rust | 保守诚实型——"在每个边界上选择诚实与保守" |
| **Hermes** | Python | 务实粗粝型——"为无人值守、多平台、长时运行而生" |

## 解读来源

各 Agent 的逐章解读分别来自以下仓库：

- OpenClaw: <https://github.com/hszhsz/deconstructing-agent-openclaw>
- Hermes: <https://github.com/hszhsz/deconstructing-hermes-agent>
- OpenCode: <https://github.com/hszhsz/deconstructing-agent-opencode>
- Codex: <https://github.com/hszhsz/deconstructing-agent-codex>
