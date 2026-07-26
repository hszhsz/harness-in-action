# 第 2 章　OpenWorker 全景：一个三层架构的全貌

## 2.1　项目定位与设计哲学

OpenWorker 的 README 一句话定义了自己：

> **AI that gets your everyday tasks done.** OpenWorker is an open-source AI coworker that lives on your desktop and delivers **finished work**, not just chat.

三个关键词：**desktop**（桌面端原生应用）、**finished work**（交付完成品而非建议）、**open-source**（MIT 许可，基于 aisuite 构建）。

设计哲学可以归纳为四条：

1. **Local-first**：所有数据（对话、记忆、连接器令牌、模型 key）存在本地 SQLite/JSONL，唯一的云端组件是一个 OAuth 中继服务。
2. **Bring your own model**：14+ 家供应商 + Ollama 本地模型，不锁定任何一家。
3. **Ask before acting**：写入、发送、Shell 命令一律审批门控。
4. **Built on aisuite**：引擎层不自己造轮子， aisuite 提供统一的 chat-completions API + tools/toolkits 层。

## 2.2　三层架构全景

```
┌────────────────────────────────────────────────┐
│              OpenWorker desktop app            │  Tauri shell (Rust) + React UI
├────────────────────────────────────────────────┤
│           local agent server (Python)          │  FastAPI + TurnEngine + tools + connectors
├───────────────┬────────────────┬───────────────┤
│  your files   │   your tools   │  your model   │  everything runs with your keys
│  & terminal   │ 25+ connectors │  any provider │  on your machine
└───────────────┴────────────────┴───────────────┘
```

### 目录组织

| 目录 | 职责 |
|---|---|
| `coworker/` | Python 后端：Agent 引擎、模型 Provider、连接器、MCP 客户端、记忆、自动化 |
| `surfaces/gui/` | 桌面应用：React UI + Tauri 壳（管着 server 进程） |
| `stt/` | 语音转文字侧车（Rust） |
| `packaging/` | 安装包构建（macOS DMG、Windows）、自动更新清单、开发引导 |
| `docs/` | 设计文档和决策日志 |
| `tests/` | 后端测试套件 |

### `coworker/` 内部结构

| 子目录/文件 | 职责 |
|---|---|
| `engine.py` | **TurnEngine**：Agent 主循环（核心） |
| `agent.py` | `build_engine()`：从 Agent 定义组装 TurnEngine |
| `permissions.py` | 权限引擎：模式 + 风险评估 + 允许列表 |
| `risk.py` | 工具风险四档分类：read / write_local / exec / external |
| `inbox.py` | 跨会话审批队列：resolve-once 状态机 |
| `audit.py` | 本地审计日志（SQLite） |
| `events.py` | 事件模型（Engine → Surface 的契约） |
| `sessions.py` | 会话记录数据类 |
| `conversations.py` | 会话存储：SQLite 索引 + JSONL 消息日志 |
| `config.py` | 多层配置加载与合并 |
| `secrets.py` | 本地密钥存储 |
| `workspace_trust.py` | 工作区信任边界 |
| `roots.py` | 多根目录管理 |
| `mentions.py` | Slack/Telegram @提及路由 |
| `unattended.py` | 无人值守模式 |
| `selfwake.py` | 自唤醒工具（定时器 / 事件触发） |
| `overrides.py` | 用户级风险覆盖 |
| `cloud.py` | OpenWorker Cloud 签到（可选） |
| `tools/` | 内置工具：shell、files、git、plan、ask、todo、subagent、search |
| `providers/` | 模型 Provider：OpenAI、Anthropic、Gemini + 14 兼容厂商 |
| `connectors/` | 25+ 消息/集成连接器 |
| `mcp/` | MCP 客户端集成 |
| `memory/` | 持久化记忆（SQLite） |
| `skills/` | Skill 加载器 |
| `automation/` | 定时任务：scheduler + store + models |
| `server/` | FastAPI 应用 + SessionManager |
| `agents/` | Agent 定义：code、chat、cowork、myhelper |
| `personas/` | Persona 系统（可安装的 Agent 模板） |
| `web/` | Web 搜索/抓取工具 |

## 2.3　四条工程红线

贯穿整个代码库的四条设计原则：

### 红线 1：本地优先，云端可选
- `SecretStore` 管所有密钥（API key、连接器 token），存在 `~/.config/coworker/` 下
- 唯一的云端组件是 OAuth broker（用于连接器的托管登录）
- `cloud.py` 的所有功能都是可选的（`cloud_status()` 返回 `signed_in: false` 不影响任何核心功能）

### 红线 2：审批是持久化的，不是内存中的
- `InboxStore` 把每个审批写入 JSON 文件（`_save()`）
- 即使进程崩溃，重启后 `resume()` 能重建挂起的审批
- 每个审批 item 有 `tool_call_id`，幂等去重

### 红线 3：风险分级驱动一切
- `risk.py` 的四档分类（`RiskClass.READ / WRITE_LOCAL / EXEC / EXTERNAL`）是权限引擎、审批路由、并行执行策略的**唯一输入**
- 不是按工具名做特判，而是按内在副作用分类
- 用户可以通过 `overrides.py` 覆盖（主要用来放松 MCP 的保守默认）

### 红线 4：Sidecar 模式，不是进程内
- Python 后端是一个独立的 FastAPI server（`server/app.py`）
- 桌面 UI 通过 WebSocket 连接（`/ws/sessions/{id}`）
- 也暴露 OpenAI 兼容的 `/v1/chat/completions` 端点
- Tauri 壳管着 server 进程的生命周期

## 2.4　如何高效阅读这套代码

推荐阅读顺序：

1. **入口**：`server/app.py` → `server/manager.py`（SessionManager 是所有状态的拥有者）
2. **引擎**：`agent.py`（`build_engine` 看组装）→ `engine.py`（`TurnEngine._loop` 看心跳）
3. **安全**：`risk.py` → `permissions.py` → `inbox.py`（三者构成完整的审批链路）
4. **工具**：`tools/registry.py` → `tools/shell.py` → `connectors/tools.py` → `mcp/tools.py`
5. **状态**：`conversations.py` → `sessions.py` → `memory/sqlite_store.py`
6. **自动化**：`automation/scheduler.py` → `automation/models.py` → `unattended.py`

## 本章小结

OpenWorker 的架构是一个经典的"三层 + 侧车"桌面应用：Tauri 壳管进程、React UI 管交互、Python 后端管 Agent 循环。它的工程性格可以概括为**"务实守门型"**--务实在于用 aisuite 不造轮子、用 FastAPI 不自研框架、用 SQLite 不引入数据库；守门在于把审批做成了持久化队列、把风险做成了四档分类、把路径做成了根目录约束。

## 动手实验

1. 按 README 的步骤从源码运行 OpenWorker，观察 Tauri 壳如何启动 Python server。
2. 在 `coworker/` 目录下运行 `grep -r "class.*Engine" --include="*.py"` 找到所有引擎类。
3. 打开 `coworker/agent.py` 的 `build_engine()` 函数，列出它注册的所有工具类别。
4. 找到 `coworker/server/app.py` 中的 `create_app()` 函数，数一下有多少个路由端点。
