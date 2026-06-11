# 解构 Agent：OpenCode

> 一本面向 Agent 开发者的开源技术书 —— 通过逐层拆解 [OpenCode](https://github.com/anomalyco/opencode) 的真实代码，理解一个生产级 AI 编码 Agent 是如何被工程化的。

## 关于本书

市面上关于 Agent 的内容大多停留在「Prompt 技巧」和「Demo 拼装」层面。但当你真正要把一个 Agent 做成**可约束、可观测、可复现、可迭代**的生产系统时，会发现真正的难点在于模型之外的那一整套工程系统——也就是 **Harness（驾驭层）**。

本书以 OpenCode 这个成熟的开源 AI 编码 Agent（Bun + TypeScript monorepo，基于 [Effect](https://effect.website/) 构建，内含 `core` / `opencode` / `cli` / `llm` / `tui` 等 20 多个 workspace 包）为标本，**自外向内、顺着一次真实请求的执行链路**，把 Agent 的每一个子系统逐一拆开：从进程入口、Session 运行时、模型协议适配，到统一工具系统、权限审批、System Context 装配、压缩与持久化，最后到插件/MCP 扩展与多端服务接入。

OpenCode 有一个特别值得研究的地方：它正处在一次**从 v1 到 v2 的架构迁移**中。`packages/opencode/src` 是仍在服役的 v1 运行时，而 `packages/core/src` 是基于 Effect 重写、带有严格领域语言定义（见 `CONTEXT.md`）的 v2 内核。把这两套实现对照着读，你能看到同一批工程问题在两种范式下的不同解法——这是单看一份"定稿"代码学不到的。

**读完本书你将掌握**：一个生产级 Agent 的完整骨架，以及每一个工程决策背后的「为什么」。

## 本书方法论：以真实代码为唯一真理来源

本书所有技术论断都来自对 OpenCode 公开仓库源码的**实际阅读**，引用确切的文件名、函数名、常量与字面量值。我们不写"Agent 一般怎么做"的常识脑补——拿不准就回去读代码。建议你 clone 仓库边读边对照：

```bash
git clone https://github.com/anomalyco/opencode.git
```

> 注：OpenCode 默认分支是 `dev`（见 `AGENTS.md`）。代码持续演进，本书引用以撰写时的快照为准；如有出入，**以你本地读到的源码为准**。

## 读者对象

- 正在或准备开发 Agent / Agent 框架的工程师
- 想理解 Claude Code / Codex 这类编码 Agent CLI 内部原理的开发者
- 想学习 Effect、领域驱动设计在真实大型 TypeScript 项目中如何落地的开发者

**前置知识**：熟悉 TypeScript，了解 LLM API 基本概念（消息、工具调用、流式输出）。Effect 相关概念会在用到时随文解释。

## 阅读主线

本书遵循一条「跟随一次请求穿过整个系统」的主线：

```
进程入口 (bin/opencode → src/index.ts, yargs)
  → CLI 命令与运行时 (cli/cmd/*, framework/runtime)
    → Session 运行时 (session/session.ts, processor.ts, runner)
      → 模型协议适配 (packages/llm: protocols / providers)
        → 统一工具系统 (core/src/tool: registry, tool.ts, builtins)
          → 命令执行与沙箱 (tool/bash.ts, pty, shell)
            → 权限与审批 (core/src/permission, agent/subagent-permissions)
              → System Context 装配与压缩 (system-context, session/compaction)
                → 状态持久化 (core/src/database, session/sql.ts, SQLite)
                  → 扩展体系 (core/src/plugin, mcp, skill)
                    → 多端服务接入 (server, acp, tui, sdk)
```

## 目录

### 第一部分　导论

- **第 1 章　从 Prompt 到 Harness：Agent 工程范式的演进**
- **第 2 章　OpenCode 全景：一个生产级编码 Agent 长什么样**

### 第二部分　骨架：从进程到 Session 内核

- **第 3 章　从进程到命令：一次请求的起点**
- **第 4 章　Session 运行时：对话状态的中枢**
- **第 5 章　主循环与 Processor：心脏是如何跳动的**
- **第 6 章　System Context：决定模型看到什么**

### 第三部分　与模型对话

- **第 7 章　LLM 抽象层：让多家模型协议可插拔**

### 第四部分　让 Agent 动手

- **第 8 章　统一工具系统：注册、描述与调度**
- **第 9 章　命令执行与沙箱：bash / pty 的护栏**

### 第五部分　安全与可控

- **第 10 章　权限与审批：在动手前踩刹车**

### 第六部分　记住一切：上下文与状态

- **第 11 章　压缩与上下文纪元：在有限窗口里走得更远**
- **第 12 章　状态持久化：SQLite 与领域 SQL**

### 第七部分　可扩展性与多端

- **第 13 章　插件、Skill 与 MCP：核心精简、能力外挂**
- **第 14 章　服务与多端接入：server / acp / tui / sdk**

### 第八部分　工程化收尾

- **第 15 章　测试与可信度：怎么知道改好了**
- **第 16 章　可迁移的工程原则：从 OpenCode 学到了什么**

> 实际章节随阅读推进可能微调。每章末尾提供「本章小结」与「动手实验」。

## License

本书内容采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可；涉及的 OpenCode 代码版权归其原作者所有（MIT License）。
