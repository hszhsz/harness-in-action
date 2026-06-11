# 解构 Hermes Agent

> 一本面向 Agent 开发者的开源技术书 —— 通过逐层拆解 [Hermes Agent](https://github.com/NousResearch/hermes-agent) 的真实源码，理解一个**自我改进型**生产级 Agent 是如何被工程化的。

## 关于本书

市面上关于 Agent 的内容大多停留在「Prompt 技巧」与「Demo 拼装」。但当你要把一个 Agent 做成**可约束、可观测、可复现、可持续学习**的生产系统时，真正的难点都在模型之外的那一整套工程系统里。

本书以 Nous Research 开源的 **Hermes Agent** 为标本。它不是一个玩具：一个以 Python 单体为核心、辅以 TypeScript TUI 与 Web 前端的大型工程，自带**闭环学习循环**（从经验中创建技能、使用中自我改进、跨会话回忆），可运行在六种终端后端（local / Docker / SSH / Singularity / Modal / Daytona）之上，并通过单一网关进程接入 Telegram、Discord、Slack、WhatsApp、Signal、飞书等十余个平台。

本书坚持一条铁律：**每一个技术论断都来自实际读过的源码**——确切的文件名、函数名、常量名、字面量值。我们不写"Agent 一般怎么做"，只写"Hermes 在代码里怎么做"。

## 读者对象

- 正在或准备开发 Agent / Agent 框架的工程师
- 想理解生产级 Agent CLI 内部原理的开发者
- 需要为团队搭建可落地、可信赖 Harness 的技术负责人

**前置知识**：熟悉 Python，了解 LLM API 基本概念（消息、工具调用、流式输出）。

## 阅读主线

本书顺着"一次请求穿过整个系统、再加上一条后台自我改进的暗线"展开：

```
进程入口 (hermes → hermes_cli.main:main)
  → Agent 主循环 (agent/conversation_loop.py:run_conversation)
    → 上下文引擎与压缩 (agent/context_engine, context_compressor)
      → 工具系统 (tools/registry + toolsets)
        → 审批与命令安全 (tools/approval, agent/file_safety)
          → 沙箱与六种终端后端 (tools/environments)
            → 子 Agent 委派 (tools/delegate_tool)
              → 模型 Provider 抽象与凭证池 (providers, agent/*_adapter, credential_pool)
                → 消息网关与多平台 (gateway/)
                  → 状态持久化 (hermes_state.py: SQLite + FTS5)
                    → 闭环学习 (background_review / curator / honcho)
                      → 定时自动化 (cron/)
                        → 可观测性与脱敏 (hermes_logging, agent/redact)
                          → 测试与可信度 (tests/)
```

## 目录

1. [第 1 章　全景与运行时入口](chapters/ch01-panorama-and-entry.md)
2. [第 2 章　Agent 主循环与回合生命周期](chapters/ch02-agent-loop.md)
3. [第 3 章　上下文引擎与压缩](chapters/ch03-context-engine.md)
4. [第 4 章　工具系统：注册与分发](chapters/ch04-tool-system.md)
5. [第 5 章　审批与命令安全](chapters/ch05-approval-and-safety.md)
6. [第 6 章　沙箱与六种终端后端](chapters/ch06-sandbox-backends.md)
7. [第 7 章　子 Agent 与任务委派](chapters/ch07-subagents-and-delegation.md)
8. [第 8 章　模型 Provider 抽象与凭证池](chapters/ch08-provider-and-credentials.md)
9. [第 9 章　消息网关与多平台接入](chapters/ch09-messaging-gateway.md)
10. [第 10 章　状态持久化：SQLite 与 FTS5](chapters/ch10-state-and-persistence.md)
11. [第 11 章　闭环学习：记忆与技能](chapters/ch11-learning-loop.md)
12. [第 12 章　定时自动化：Cron 调度](chapters/ch12-cron-automation.md)
13. [第 13 章　可观测性：日志与脱敏](chapters/ch13-observability-and-redaction.md)
14. [第 14 章　测试与可信度](chapters/ch14-testing-and-trustworthiness.md)
15. [第 15 章　可迁移的工程原则](chapters/ch15-engineering-principles.md)

## 方法论

本书写作方法论参考了《[解构 Agent：OpenClaw](https://github.com/hszhsz/deconstructing-agent-openclaw)》：以真实代码为唯一真理来源，按"由外到内、由启动到收束"的顺序逐章拆解，每章都包含本章小结与四个动手实验。

## 版本说明

本书基于 Hermes Agent `pyproject.toml` 中记录的 `version = "0.16.0"` 源码快照写成。源码以高频迭代著称，阅读时请以你本地 clone 的实际代码为准。

## License

本书内容遵循 MIT License。Hermes Agent 本身亦为 [MIT License](https://github.com/NousResearch/hermes-agent/blob/main/LICENSE)。
