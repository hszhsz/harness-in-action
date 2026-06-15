# 解构 Agent：DeerFlow 2.0

> 一本面向 Agent 开发者的开源技术书 —— 通过逐层拆解 [bytedance/deer-flow](https://github.com/bytedance/deer-flow)（DeerFlow 2.0，一个基于 LangGraph 的「super agent harness」）的真实源码，从 **harness engineering（执行骨架工程）** 的视角，理解一个生产级通用 Agent 是怎么被工程化的。

## 关于本书

市面上谈 Agent 的内容大多停在「Prompt 技巧」和「Demo 拼装」。但当你真要把一个 Agent 做成 **可约束、可观测、可恢复、可扩展** 的生产系统时，会发现真正的难点全在模型之外——也就是包裹在大模型外面的那层工程：**Harness（驾驭层 / 执行骨架）**。它决定模型能调什么工具、看到什么上下文、在什么边界上被拦下、出错后怎么恢复、副作用能不能落地。

本书以 **DeerFlow 2.0** 为标本。它不是一个玩具：2026 年 2 月随 2.0 版本发布登顶 GitHub Trending，是一次「从零重写、与 1.x 不共享代码」的工程实践。与大多数自己撸 agent loop 的项目不同，DeerFlow 把执行内核直接架在 **LangChain / LangGraph** 之上——`make_lead_agent` 调 `langchain.agents.create_agent`，所有 DeerFlow 自己的工程决策都浓缩进一条 **14 个中间件（middleware）组成的有序链**：沙箱基建、悬挂工具调用修补、护栏、工具错误处理、上下文摘要、待办、标题、记忆、视觉、子 agent 限流、循环检测、澄清拦截。读懂这条链，就读懂了 DeerFlow 的「性格」。

本书的方法论只有一条铁律：**以真实源码为唯一真理来源**。每一个技术论断都来自实际读过的文件，引用确切的函数名、常量名、字面量与默认值。读不到的不写，拿不准的回去读代码。

**读完本书你将掌握**：一个建立在 LangGraph 之上的生产级通用 Agent 的完整骨架，以及每一个工程决策背后的「为什么」——尤其是「如何在不自己造主循环的前提下，用中间件把约束、安全、可观测性焊进框架」。

## 读者对象

- 正在或准备开发 Agent / Agent 框架的工程师
- 想理解 LangGraph `create_agent` + 中间件这套范式如何支撑生产 Agent 的开发者
- 需要为团队搭建可落地 Harness 的技术负责人

**前置知识**：熟悉 Python，了解 LLM API 的基本概念（消息、工具调用、流式输出、上下文窗口），对 LangChain / LangGraph 有基础认识更佳。

## 阅读主线

本书顺着「一次真实请求穿过整个系统」的链路自外向内展开：

```
进程入口 (langgraph.json → deerflow.agents:make_lead_agent)
  → 运行时调度 (RunManager / RunWorker / RunJournal)
    → Agent 主循环 (langchain.agents.create_agent + ThreadState)
      → 中间件链 (build_middlewares 的 14 段有序装配)
        → 模型网关 (models/factory.py provider 抽象 + patched_* + 流式超时)
          → 工具系统 (tools/tools.py 注册 + deferred tool search)
            → 沙箱与命令执行 (LocalSandbox vs AioSandbox + fail-closed host bash)
              → 上下文工程 (summarization / loop_detection / tool_output_budget)
                → 记忆系统 (MemoryMiddleware + updater 异步队列)
                  → 子 agent 与扩展 (task_tool / SubagentExecutor / Skill / MCP)
                    → 配置·持久化·可观测性 (AppConfig / SQL / tracing)
                      → 测试与可信度 (DI / blocking_io 门禁 / fixtures)
```

## 目录

### 第一部分　导论

- [**第 1 章　从 Prompt 到 Harness：为什么要解构 DeerFlow**](./chapters/ch01-from-prompt-to-harness.md)
- [**第 2 章　DeerFlow 2.0 全景：一个架在 LangGraph 上的 super agent harness**](./chapters/ch02-deerflow-panorama.md)

### 第二部分　执行内核

- [**第 3 章　从进程到运行时：langgraph.json、RunManager 与运行生命周期**](./chapters/ch03-process-to-runtime.md)
- [**第 4 章　Agent 主循环：create_agent、ThreadState 与中间件钩子**](./chapters/ch04-agent-loop.md)
- [**第 5 章　中间件链：14 段装配顺序与 @Next/@Prev 定位**](./chapters/ch05-middleware-chain.md)

### 第三部分　与模型对话

- [**第 6 章　模型网关：provider 抽象、patched_* 与流式超时**](./chapters/ch06-model-gateway.md)

### 第四部分　让 Agent 动手：工具系统

- [**第 7 章　工具系统：注册、分组与 deferred tool search**](./chapters/ch07-tool-system.md)
- [**第 8 章　沙箱与命令执行：Local vs Aio 与 fail-closed host bash**](./chapters/ch08-sandbox-and-exec.md)

### 第五部分　上下文工程

- [**第 9 章　上下文压缩：摘要中间件与 skill 保留策略**](./chapters/ch09-context-summarization.md)
- [**第 10 章　防失控：循环检测、工具输出预算与安全终止**](./chapters/ch10-loop-and-budget.md)
- [**第 11 章　长期记忆：MemoryMiddleware 与异步更新队列**](./chapters/ch11-memory-system.md)

### 第六部分　扩展体系

- [**第 12 章　子 Agent：task_tool、SubagentExecutor 与隔离事件循环**](./chapters/ch12-subagents.md)
- [**第 13 章　Skill 系统：解析、权限、安全扫描与激活**](./chapters/ch13-skills.md)
- [**第 14 章　MCP 集成：第三方能力的接入、隔离与延迟暴露**](./chapters/ch14-mcp.md)

### 第七部分　状态与运维

- [**第 15 章　配置系统：一棵校验过的树，默认即安全、热加载有边界**](./chapters/ch15-config.md)
- [**第 16 章　持久化与可观测性：状态落盘、事件留痕、记得多但不外泄**](./chapters/ch16-persistence-observability.md)

### 第八部分　质量与收束

- [**第 17 章　收尾综合：从 DeerFlow 提炼出的可迁移工程原则**](./chapters/ch17-engineering-principles.md)

## 一句话画像

> **DeerFlow 2.0** —— 务实的「借壳」型：不自己造主循环，而是把 LangGraph `create_agent` 当心脏，所有工程意志都焊进一条 14 段中间件链；默认即关闭（host bash 默认禁、危险路径默认拦），约束在配置合并时只收紧，错误信息本身就是修复指南。

## 方法论铁律

1. **零虚构**：每个技术论断都来自实际读过的源码文件，引用真实的函数名、常量名、字面量与默认值。
2. **以源码为证**：宁可少写一个机制，也不写一个没读过源码的机制。
3. **保留原文**：函数名、常量名、路径、配置键全部保留英文原文。

> 本书是 [Harness in Action](../README.md) 合集的一部分。
