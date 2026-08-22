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
