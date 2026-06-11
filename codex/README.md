# 解构 Agent：Codex

> 一本面向 Agent 开发者的开源技术书籍 —— 通过逐层拆解 [OpenAI Codex CLI](https://github.com/openai/codex) 的真实 Rust 代码，理解一个生产级编码 Agent 是如何被工程化的。

## 关于本书

市面上关于 Agent 的内容大多停留在「Prompt 技巧」和「Demo 拼装」层面。但当你真正要把一个 Agent 做成**可约束、可观测、可复现、可迭代**的生产系统时，会发现真正的难点在于模型之外的那一整套工程系统——也就是 **Harness（驾驭层）**。

本书以 OpenAI 官方开源的 **Codex CLI**（一个用 Rust 编写、拥有 100+ crate 的 Cargo workspace）为标本，**自底向上、顺着一次真实请求的执行链路**，把这个编码 Agent 的每一个子系统逐一拆开：从 npm 启动器与 `arg0` 多态可执行文件、`CodexThread` 主循环、`ModelClient` 与 Responses API 流式抽象、统一工具计划，到 `apply_patch` 安全改代码、Landlock / seccomp / Seatbelt 多平台沙箱、执行策略与审批流、多层配置合并、ChatGPT 设备码登录、Rollout + SQLite 状态持久化、MCP / 插件 / Skill 扩展体系，最后到 ratatui TUI 与无头 exec 模式。

**读完本书你将掌握**：一个生产级编码 Agent 的完整骨架，以及每一个工程决策背后的「为什么」。

## 读者对象

- 正在或准备开发 Agent / Agent 框架的工程师
- 想理解 Codex / Claude Code 这类编码 Agent CLI 内部原理的开发者
- 需要为团队搭建可落地 Harness 的技术负责人

**前置知识**：能读懂 Rust（所有权、trait、async/await、enum）即可；了解 LLM API 基本概念（消息、工具调用、流式输出）会更顺。

## 阅读主线

本书遵循一条「跟随一次请求穿过整个系统」的主线：

```
npm 启动器 (codex-cli/bin/codex.js)
  → arg0 多态分发 (codex-rs/arg0)
    → CLI 子命令 (codex-rs/cli) / 无头 exec (codex-rs/exec)
      → 线程与主循环 (core/src/thread_manager.rs · codex_thread.rs)
        → 模型客户端 (core/src/client.rs · Responses API)
          → 工具路由与执行 (core/src/tools)
            → apply_patch 改代码 + 命令执行 (core/src/apply_patch.rs · exec.rs)
              → 多平台沙箱 (sandboxing · landlock · seatbelt · windows_sandbox)
                → 执行策略与审批 (exec_policy.rs · sandboxing.rs · execpolicy crate)
                  → 配置 / 认证 (config crate · login crate)
                    → 上下文 / 状态持久化 (AGENTS.md · rollout · state SQLite)
                      → 扩展体系 (mcp · plugin · skills)
                        → 前端形态 (tui · exec) 与可观测性 (otel · analytics)
```

## 目录

### 第一部分　导论

- **第 1 章　从 Prompt 到 Harness：编码 Agent 的工程范式**
- **第 2 章　Codex 全景：一个 Rust monorepo 长什么样**

### 第二部分　骨架：进程与执行内核

- **第 3 章　从进程到运行时：npm 启动器、`arg0` 多态与 CLI 分发**
- **第 4 章　Agent 主循环：`CodexThread` 与一次 turn 的生命周期**
- **第 5 章　与模型对话：`ModelClient` 与 Responses API 流式抽象**

### 第三部分　让 Agent 动手：工具系统

- **第 6 章　统一工具计划：`ToolRouter`、`ToolRegistry` 与工具调度**
- **第 7 章　`apply_patch`：让 Agent 安全地改代码**

### 第四部分　安全与可控：Agent 的护栏

- **第 8 章　命令执行与多平台沙箱：Landlock / seccomp / Seatbelt / Windows**
- **第 9 章　执行策略与审批流：在动手前踩刹车**

### 第五部分　状态、配置与接入

- **第 10 章　配置系统：多层加载、合并与「约束只能收紧」**
- **第 11 章　认证与登录：ChatGPT 设备码、API Key 与令牌管理**
- **第 12 章　上下文工程：`AGENTS.md`、System Prompt 与 per-turn 投影**
- **第 13 章　状态持久化：Rollout、JSONL 与 SQLite 状态库**

### 第六部分　可扩展性与运维

- **第 14 章　扩展体系：MCP、插件与 Skill**
- **第 15 章　可观测性、遥测与诊断导出**
- **第 16 章　前端形态：ratatui TUI 与无头 `exec` 模式**

### 第七部分　工程化收尾

- **第 17 章　测试、可信度与可迁移的工程原则**

---

## 配套说明

- 本书引用的代码路径均来自 Codex 公开仓库，建议边读边对照源码。
- 每章末尾提供「本章小结」与「动手实验」。
- 欢迎通过 Issue / PR 参与勘误与共建。

## License

本书内容采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可；涉及的 Codex 代码版权归 OpenAI 及其原作者所有。
