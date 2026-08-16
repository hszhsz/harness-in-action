# 第 1 章　从 Prompt 到 Harness：插件化 Agent 的工程范式

> 在拆开任何一个零件之前，先回答一个问题：为什么同一个模型，套上不同的外壳，会变成性格迥异的 Agent？

## 1.1　Agent = Model + Harness

当一个 LLM 被包裹上一层工程系统后，它就不再是"聊天机器人"，而是"Agent"。这层工程系统决定了模型能调什么工具、看到什么上下文、在什么边界上被打断、出错后怎么恢复、副作用能不能落地——这就是 **Harness（驾驭层）**。

DeepSeek Harness（`dsh`）是一个特别有研究价值的标本，因为它的定位很极端：**整个产品就是一层 Harness，而且这层 Harness 本身又被拆成了无数可替换的插件。** 它的 README 第一句就说：

> DeepSeek Harness（`dsh`）是由 DeepSeek AI 开发的开源 agent harness。它采用**一切皆插件**的架构，并由 [Cordis](https://github.com/cordiverse/cordis) 驱动。[[README.zh.md]](https://github.com/deepseek-ai/deepseek-harness/blob/main/README.zh.md)

注意这里的措辞：不是"基于 Cordis 构建"，而是"**由 Cordis 驱动**"（driven by Cordis）。Cordis 不是被调用的一个库，而是整个系统的血管和骨架——`vendor/` 目录里直接塞了一份 Cordis 的源码拷贝（pinned source copy，见 `vendor/README.md` 的 manifest 与上游 SHA）。dsh 团队宁可 vendoring 也要把这个框架焊死在自己手里，可见它对这一层有多依赖。

## 1.2　「一切皆插件」：为什么 DeepSeek 敢把主循环也做成插件

绝大多数 Agent 框架（包括本书已解读的 OpenClaw、Codex、Hermes）都有一个"核心"：主循环、工具注册表、状态存储是特权代码，插件只能挂在边缘。dsh 反着来——**每一块都是插件**：

> Cordis 是 dsh 之下的框架：插件向共享上下文贡献服务、类型化事件和可逆 effect。产品的每一部分都是插件，包括模型适配器、工具注册表、会话日志，以及 agent loop 本身——所以每一部分都可以从配置上被替换。[[docs/architecture.md]](https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/architecture.md)

这句话值得逐字拆解：

- **模型适配器是插件** —— `ctx.llm` 上的 adapter 注册，换模型就是换插件。
- **工具注册表是插件** —— `ctx.tools` 本身是一个 Cordis 服务，注册工具就是往这个服务上挂条目。
- **会话日志是插件** —— `ctx.sessions` 是 append-only 事件日志服务。
- **连 Agent 主循环本身都是插件** —— `ctx.agentLoop` 是 `agent-loop` 这个包提供的默认驱动实现；`Agent` 接口是公共契约，而 `ReactLoopAgent` 只是它的"默认产品循环"。

"没有特权核心"的直接推论是：**你扩展 dsh 的方式不是在某个固定的扩展点上写代码，而是在别的插件旁边再挂一个插件。** 而因为 Cordis 的注册都是可逆 effect，插件卸载时它会自动回卷——热更新、按 profile 组合、用户 patch 覆盖，全都建立在这个机制上。

> There is no privileged core to patch: you extend dsh by mounting a plugin beside the others, and registrations are effects that unwind when their plugin unloads.

## 1.3　Harness 的六个核心职责在 dsh 中的映射

| 职责 | 定义 | DeepSeek Harness 实现 |
|---|---|---|
| **上下文** | 决定模型看到什么 | `ctx.systemPrompt` 的 prompt sections 按序拼接 + `agent.inject()` 注入 + 从 session 日志 `deriveMessages()` 派生历史 |
| **工具** | 模型能调什么 | `ctx.tools` 作用域工具注册表 + `ToolDefinition` + 六段守卫管线 |
| **编排** | 单轮循环如何串成任务 | `ctx.agentLoop`（ReactLoopAgent）+ turn/step 边界 + `agent/*` 事件 |
| **状态** | 出错后怎么恢复 | `ctx.sessions` append-only 事件日志 + `ctx.sessionPersistence`（JSONL/SQLite）+ 崩溃恢复 |
| **观测** | 发生了什么可查 | `session/event` 广播 + `session-query` + `session-projection` + telemetry |
| **约束** | 在什么边界上被打断 | `ctx.approval` + `ctx.sandbox` + permission presets + 工具守卫 |

一个有意思的差异：前几个解读过的 Agent 把"状态"理解为快照（SQLite 存 session 状态），而 dsh 把"状态"理解为**事件日志**——模型可见的一切都必须能从日志重建，这就是下一节要讲的硬不变量。

## 1.4　本书方法论：以真实代码为唯一真理来源

本书所有技术论断都来自对 DeepSeek Harness 公开仓库源码的**实际阅读**，引用确切的文件名、函数名、常量与字面量值。dsh 仓库自带一套异常高质量的文档（`docs/architecture.md`、`docs/subsystems/*.md`、生成的 `config-catalog` / `tool-catalog` / `persistence-catalog`），但本书坚持一条铁律：**文档只作索引，论断以源码为准**。`docs/` 里的类型声明标着 `type-equiv`，由 `pnpm run verify-type-equiv` 漂移检查保证与源码一致——这意味着读 dsh 的文档几乎等于读源码，这是它比大多数开源项目更适合当教材的原因。

建议 clone 仓库边读边对照：

```bash
git clone https://github.com/deepseek-ai/deepseek-harness
cd deepseek-harness
pnpm install
```

写作时对应版本为 `0.1.0-rc.5`（2026-08-13 前后发布）。注意 dsh 官方声明：**当前是开发者预览，未来会出现破坏兼容性的变更**。读代码时留意版本，别把某个目录结构当成永久真理。

## 本章小结

DeepSeek Harness 的 Harness 工程围绕一个核心设计哲学展开：**一切皆插件，无特权核心**。这个哲学被刻进了每一个子系统的设计里——从 Cordis 的可逆 effect 到 profile/bundle 的分层组合，从 Session 事件日志到 Capability Seam 三件套。接下来的章节将逐一拆开这些子系统：第 2 章先看全景，第 3 章讲透 Cordis，第 4 章看系统如何从空列表被组合出来，然后顺着一次请求穿过主循环、LLM、工具、沙箱、持久化、多 Agent 与 Web UI。

## 动手实验

1. Clone 仓库，运行 `pnpm install && pnpm run build`，然后用 `pnpm dsh --profile headless "say hi"` 跑通第一个 headless 任务（需要 `DEEPSEEK_API_KEY`）。
2. 运行 `pnpm dsh --profile web --dump-config`，看看一棵真实的插件树长什么样——数一数有多少个 entry。
3. 打开 `docs/architecture.md`，对照"Events"一节的事件分类（session events / agent events / capability events），在源码里各找一个生产者。
4. 在 `packages/core/agent-loop/src/agent.ts` 中找到 `ReactLoopAgent` 类，数一下它的 `Phase` 联合类型有几种状态——每种状态都是一个工程决策。
