# 解构 Agent：OpenWorker

> 一本面向 Agent 开发者的开源技术书籍 -- 通过逐层拆解 [OpenWorker](https://github.com/andrewyng/openworker) 的真实 Python 源码，理解一个**桌面端本地优先**的生产级 Agent 是如何被工程化的。

## 关于本书

市面上关于 Agent 的内容大多停留在「Prompt 技巧」和「Demo 拼装」层面。但当你真正要把一个 Agent 做成**可约束、可观测、可复现、可交付**的生产系统时，会发现真正的难点在于模型之外的那一整套工程系统--也就是 **Harness（驾驭层）**。

本书以 Andrew Ng 团队开源的 **OpenWorker** 为标本。它不是一个玩具：一个以 Python 后端（`coworker/`）+ React/Tauri 桌面壳（`surfaces/gui/`）+ Rust 语音侧车（`stt/`）组成的三层架构，基于 [aisuite](https://github.com/andrewyng/aisuite) 构建，内置 25+ 消息连接器（Slack / GitHub / Jira / Notion / Linear / HubSpot / Gmail / Google Calendar 等）、MCP 客户端、自动化调度器、持久化记忆、审批 Inbox、审计日志，以及完整的无人值守运行模式。它运行在用户本地桌面上，数据不离开机器（除用户选择的模型和集成），API key 自带，支持 14+ 家模型供应商。

OpenWorker 最值得研究的设计是它的**审批门控哲学**：不是简单的"什么都需要确认"，而是把每个工具调用按内在风险分为四档（read / write_local / exec / external），用四种会话模式（discuss / plan / interactive / auto）交叉管控，再把审批本身变成一个可持久化、可跨 surface 回答的 Inbox 系统。当用户离开时，审批自动进入跨会话的 Inbox 队列；当用户回来时，内联提示和离线回答汇合到同一个"resolve-once, first-responder-wins"的状态机。

本书坚持一条铁律：**每一个技术论断都来自实际读过的源码**--确切的文件名、函数名、常量名、字面量值。我们不写"Agent 一般怎么做"，只写"OpenWorker 在代码里怎么做"。

## 读者对象

- 正在或准备开发 Agent / Agent 框架的工程师
- 想理解桌面端 Agent 内部原理的开发者
- 需要为团队或个人搭建可落地、可信赖 Harness 的技术负责人
- 对 aisuite 框架的实际工程用法感兴趣的开发者

**前置知识**：熟悉 Python（async/await、dataclass、ABC），了解 LLM API 基本概念（消息、工具调用、流式输出）。

## 阅读主线

本书顺着"一次请求穿过整个系统、再加上一条无人值守的暗线"展开：

```
桌面应用入口 (Tauri shell -> local server)
  -> FastAPI 控制面 (server/app.py)
    -> 引擎组装 (agent.py: build_engine)
      -> TurnEngine 主循环 (engine.py: run -> _loop)
        -> 模型 Provider 抽象 (providers/base.py, registry.py, router.py)
          -> 工具注册与风险分级 (tools/registry.py, risk.py)
            -> 权限引擎与审批 (permissions.py, inbox.py)
              -> 连接器系统 (connectors/base.py, tools.py, descriptors.py)
                -> MCP 集成 (mcp/client.py, tools.py, config.py)
                  -> 记忆与状态 (memory/, conversations.py, sessions.py)
                    -> 自动化调度 (automation/scheduler.py, models.py, store.py)
                      -> 无人值守与 Inbox (unattended.py, inbox.py)
                        -> 审计与可观测性 (audit.py, events.py)
                          -> 测试与工程权衡 (tests/)
```

## 目录

### 第一部分　导论

- **第 1 章　从 Prompt 到 Harness：桌面端 Agent 的工程范式**
- **第 2 章　OpenWorker 全景：一个三层架构的全貌**

### 第二部分　骨架：Agent 执行内核

- **第 3 章　TurnEngine：心脏是如何跳动的**
- **第 4 章　模型 Provider 抽象：让十四家供应商可插拔**

### 第三部分　让 Agent 动手：工具与风险

- **第 5 章　工具注册与风险分级：四档分类法**
- **第 6 章　内置工具与 Shell 执行：本地能力的边界**

### 第四部分　安全与可控：审批的完整闭环

- **第 7 章　权限引擎：四种模式与五重门控**
- **第 8 章　Inbox 系统：可持久化的审批队列**
- **第 9 章　无人值守与持久化恢复**

### 第五部分　连接外部世界

- **第 10 章　连接器系统：25+ 集成的统一抽象**
- **第 11 章　MCP 集成：把外部协议接进工具计划**

### 第六部分　记住一切

- **第 12 章　记忆系统：三级作用域与 SQLite 持久化**
- **第 13 章　会话与对话存储：JSONL + SQLite 双层**

### 第七部分　自动化与扩展

- **第 14 章　自动化调度：Cron、自唤醒与重叠保护**
- **第 15 章　扩展体系：Skill、Persona 与子 Agent**

### 第八部分　工程化收尾

- **第 16 章　审计、可观测性与工程权衡**

---

## 配套说明

- 本书引用的代码路径均来自 OpenWorker 公开仓库，建议边读边对照源码。
- 每章末尾提供「本章小结」与「动手实验」。
- 欢迎通过 Issue / PR 参与勘误与共建。

## License

本书内容采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可；涉及的 OpenWorker 代码版权归其原作者所有（MIT License）。
