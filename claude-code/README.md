# 解构 Agent：Claude Code（CCB 还原版）

> 一本面向 Agent 开发者的开源技术书 —— 通过逐层拆解 [claude-code-best/claude-code](https://github.com/claude-code-best/claude-code)（一个对 Anthropic 官方 Claude Code CLI 的逆向还原实现，简称 CCB）的真实源码，从 **harness engineering（执行骨架工程）** 的视角，理解一个生产级编码 Agent 是怎么被工程化的。

## 关于本书

市面上谈 Agent 的内容大多停在「Prompt 技巧」和「Demo 拼装」。但当你真要把一个 Agent 做成 **可约束、可观测、可恢复、可迭代** 的生产系统时，会发现真正的难点全在模型之外——也就是包裹在大模型外面的那层工程：**Harness（驾驭层 / 执行骨架）**。它决定模型能调什么工具、看到什么上下文、在什么边界上被拦下、出错后怎么恢复、副作用能不能落地。

本书以 CCB 为标本。它不是一个玩具：一个 TypeScript / Bun monorepo，`src/` 下 1700+ 个 `.ts` 文件、约 42 万行代码，把官方 Claude Code 的工程骨架几乎完整复刻了一遍——双层 agent 主循环、统一工具计划、AST 级别的命令安全门、多级上下文压缩、自动记忆、子 agent 委派、MCP / Skill / 插件 / Hook 四套扩展体系、JSONL 会话持久化、`tengu_` 遥测。正因为它是「还原」而非「原创」，它把官方那些隐藏在压缩产物里的工程决策重新摊开成了可读的源码——这是研究 Claude Code 内部机制难得的标本。

本书的方法论只有一条铁律：**以真实源码为唯一真理来源**。每一个技术论断都来自实际读过的文件，引用确切的函数名、常量名、字面量与默认值。读不到的不写，拿不准的回去读代码。

**读完本书你将掌握**：一个生产级编码 Agent 的完整骨架，以及每一个工程决策背后的「为什么」。

## 读者对象

- 正在或准备开发 Agent / Agent 框架的工程师
- 想理解 Claude Code / Codex 这类编码 Agent CLI 内部原理的开发者
- 需要为团队搭建可落地 Harness 的技术负责人

**前置知识**：熟悉 TypeScript / Node.js，了解 LLM API 的基本概念（消息、工具调用、流式输出、上下文窗口）。

## 阅读主线

本书顺着「一次真实请求穿过整个系统」的链路自外向内展开：

```
进程入口 (src/entrypoints/cli.tsx)
  → 双驱动分流 (交互 REPL / headless QueryEngine)
    → Agent 主循环 (src/query.ts 的 queryLoop 双层 while)
      → 模型流式抽象 (src/services/api/claude.ts + provider 抽象)
        → 统一工具计划 (packages/agent-tools + src/Tool.ts + src/tools.ts)
          → 权限审批与命令安全 (src/utils/permissions + BashTool AST 安全门)
            → 上下文工程与压缩 (snip / microcompact / autocompact)
              → 记忆系统 (CLAUDE.md 层级 + memdir 自动记忆)
                → 子 agent 与扩展 (AgentTool / Skill / MCP / Hook / 插件)
                  → 配置·状态·可观测性 (settings 级联 / JSONL / tengu_ 遥测)
                    → 测试与可信度 (DI deps / bun test / 隔离基建)
```

## 目录

### 第一部分　导论

- [**第 1 章　从 Prompt 到 Harness：为什么要解构 Claude Code**](./chapters/ch01-from-prompt-to-harness.md)
- [**第 2 章　CCB 全景：一个逆向还原的 Claude Code 长什么样**](./chapters/ch02-ccb-panorama.md)

### 第二部分　执行内核

- [**第 3 章　从进程到运行时：启动入口与双驱动分流**](./chapters/ch03-entry-and-runtime.md)
- [**第 4 章　Agent 主循环：query.ts 的双层 while 是怎么跳动的**](./chapters/ch04-agent-loop.md)
- [**第 5 章　与模型对话：Provider 抽象、流式、重试与 Fallback**](./chapters/ch05-model-gateway.md)

### 第三部分　让 Agent 动手：工具系统

- [**第 6 章　统一工具计划：三层契约与 fail-closed 默认**](./chapters/ch06-tool-system.md)
- [**第 7 章　内置工具与并发执行：从 partition 到 StreamingToolExecutor**](./chapters/ch07-builtin-tools-and-concurrency.md)

### 第四部分　安全与护栏

- [**第 8 章　权限与审批流：在动手前踩刹车**](./chapters/ch08-permissions.md)
- [**第 9 章　命令安全与沙箱：AST 级别的 fail-closed 安全门**](./chapters/ch09-command-safety-and-sandbox.md)
- [**第 10 章　Auto 模式与 YOLO 分类器：把放行权交给一个会拒绝的模型**](./chapters/ch10-auto-mode-classifier.md)

### 第五部分　记住一切：上下文与记忆

- [**第 11 章　上下文工程（上）：非破坏式裁剪——budget / snip / microcompact**](./chapters/ch11-context-trimming.md)
- [**第 12 章　上下文工程（下）：破坏式摘要——autocompact 与九段压缩**](./chapters/ch12-compaction.md)
- [**第 13 章　记忆系统：CLAUDE.md 层级、memdir 与相关性预取**](./chapters/ch13-memory.md)

### 第六部分　可扩展性

- [**第 14 章　子 Agent 与任务委派：AgentTool 与工具子集分层**](./chapters/ch14-subagents.md)
- [**第 15 章　Skill 与 MCP：把外部能力接进统一工具计划**](./chapters/ch15-skills-and-mcp.md)
- [**第 16 章　Hook 与插件：让扩展成为权限管线的一等公民**](./chapters/ch16-hooks-and-plugins.md)

### 第七部分　工程化收尾

- [**第 17 章　配置·状态·可观测性·测试，与可迁移的工程原则**](./chapters/ch17-config-state-observability-testing.md)

## 配套说明

- 本书引用的代码路径均来自 CCB 公开仓库，建议边读边对照源码。
- 每章末尾提供「本章小结」与 4 个「动手实验」，引导你亲自验证书中结论。
- CCB 更新很快，部分行号 / 常量值可能随版本漂移；遇到对不上时，以你本地 clone 的源码为准。

## License

本书内容采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可；涉及的 CCB 代码版权归其原作者所有。
