# 第 1 章　从 Prompt 到 Harness：桌面端 Agent 的工程范式

## 1.1　Agent = Model + Harness

当一个 LLM 被包裹上一层工程系统后，它就不再是"聊天机器人"，而是"Agent"。这层工程系统决定了模型能调什么工具、看到什么上下文、在什么边界上被打断、出错后怎么恢复、副作用能不能落地--这就是 **Harness（驾驭层）**。

OpenWorker 是一个特别好的标本来理解这件事，因为它的定位很独特：**桌面端、本地优先、交付完成品**。它不是 CLI（像 OpenClaw / Codex），不是 7×24 服务器进程（像 Hermes），而是一个跑在用户桌面上的原生应用--Tauri 壳管着 Python 后端，Python 后端管着 Agent 循环、25+ 连接器、本地文件和终端。

## 1.2　桌面端 Agent 的独特约束

桌面端 Agent 面对的工程约束与前几种 Agent 都不同：

- **用户可能随时离开**：不像 CLI 那样用户一直在终端前，桌面应用的用户可能切走、锁屏、关盖子。所以 OpenWorker 发明了 Inbox 系统--审批不一定要当场回答。
- **本地文件是一等公民**：不像云端 Agent 通过 API 访问数据，OpenWorker 直接读写用户磁盘上的文件。所以路径安全（`_under_writable_root`）和信任边界（`WorkspaceTrustStore`）是核心关切。
- **模型完全由用户选择**：14+ 家供应商，从 OpenAI 到 Ollama 本地模型。所以 Provider 抽象必须容忍巨大的能力差异（vision / pdf / streaming / parallel_tool_calls 全部是 per-model 的）。
- **交付的是完成品**：不是"建议你这样做"，而是"已经帮你做好了一个文档/发了一条 Slack/更新了日历"。所以审批门控是硬需求--模型在用户机器上发出的每一个副作用都需要用户同意。

## 1.3　Harness 的六个核心职责在 OpenWorker 中的映射

| 职责 | 定义 | OpenWorker 实现 |
|---|---|---|
| **上下文** | 决定模型看到什么 | `engine.py` 的 `context_provider` + `_outbound_messages` 侧车剥离 |
| **工具** | 模型能调什么 | `tools/registry.py` + `risk.py` 四档分类 + 25+ 连接器 |
| **编排** | 单轮循环如何串成任务 | `engine.py` 的 `_loop` + steering 注入 + `propose_plan` |
| **状态** | 出错后怎么恢复 | `conversations.py` JSONL + `inbox.py` 持久化审批 |
| **观测** | 发生了什么可查 | `audit.py` SQLite 审计日志 + `events.py` 事件流 |
| **约束** | 在什么边界上被打断 | `permissions.py` 四模式 + `_SHELL_OPERATORS` 黑名单 |

## 1.4　本书方法论：以真实代码为唯一真理来源

本书所有技术论断都来自对 OpenWorker 公开仓库源码的**实际阅读**。引用确切的文件名、函数名、常量与字面量值。建议读者 clone 仓库边读边对照：

```bash
git clone https://github.com/andrewyng/openworker
```

## 本章小结

OpenWorker 的 Harness 工程围绕一个核心设计哲学展开：**本地优先 + 审批门控 + 交付完成品**。这个哲学被刻进了每一个子系统的设计里--从 `PermissionEngine` 的四模式到 `InboxStore` 的 resolve-once 状态机，从 `risk.py` 的四档分类到 `audit.py` 的本地审计日志。接下来的章节将逐一拆开这些子系统。

## 动手实验

1. Clone OpenWorker 仓库，运行 `python -c "import coworker"` 确认依赖就绪。
2. 找到 `coworker/engine.py` 中的 `TurnEngine` 类，数一下它的 `__init__` 参数有多少个--每个参数都是一个工程决策。
3. 打开 `coworker/permissions.py`，找到 `Mode` 枚举的五个值，思考每种模式适合什么使用场景。
4. 在 `coworker/events.py` 中找到 `EventType` 枚举，画出从 `TURN_START` 到 `TURN_END` 的事件流。
