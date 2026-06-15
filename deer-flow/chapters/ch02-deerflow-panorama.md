# 第 2 章　DeerFlow 2.0 全景：一个架在 LangGraph 上的 super agent harness

上一章我们确立了 DeerFlow 最核心的工程选择——不自己造主循环。这一章把镜头拉远，俯瞰整个 2.0 仓库：它的目录怎么分、核心的 `deerflow` harness 包内部怎么切块、一次请求从哪进、到哪出。读完这一章，你会拿到一张全景坐标系，后面每一章深入的子系统，都能在这张图上找到自己的位置。

## 一个 monorepo，前后端分治

克隆下来的仓库根目录，按职责清晰地分成几块：`backend/`（Python 后端，也是本书的主战场）、`frontend/`（Next.js 前端）、`docker/`（部署）、`docs/`（文档）、`skills/`（内置技能库）、`contracts/`（跨语言契约）、`scripts/` 与 `tests/`。前端是一个标准的 Next.js 应用（根目录有 `next.config.js`、`pnpm-workspace.yaml`、`playwright.config.ts`），与后端通过 HTTP 通信；本书聚焦 harness 工程，因此从第 3 章起几乎只谈 `backend/`。

`backend/` 内部又分两层：`app/` 是对外的 HTTP 网关层，`packages/` 是可独立复用的核心库。打开 `backend/pyproject.toml` 能看到这个分层在依赖关系上的体现——`app` 依赖一个名为 `deerflow-harness` 的 workspace 包，并在 `[tool.uv.sources]` 里把它指向 `packages/harness`：

```toml
[tool.uv.sources]
deerflow-harness = { workspace = true }

[tool.uv.workspace]
members = ["packages/harness"]
```

这是一个有意的解耦：`app/`（FastAPI 网关、IM 渠道接入、鉴权）是「应用」，`packages/harness/deerflow/`（agent 装配、中间件、沙箱、模型、工具）是「库」。库不知道自己被谁调用——这一点在第 17 章谈测试与可信度时会反复出现：`agents/factory.py` 的文档明确说「the factory assembly itself is config-free」，纯参数进、agent 出。

## 网关层：17 个路由器与一张图

`backend/app/gateway/routers/` 下有 17 个路由器文件，名字本身就是 DeerFlow 对外能力的清单：`threads.py`、`thread_runs.py`、`runs.py`（会话与运行）、`skills.py`、`mcp.py`、`agents.py`（扩展）、`uploads.py`、`artifacts.py`（文件）、`memory.py`、`feedback.py`、`channels.py` 与 `channel_connections.py`（IM 渠道）、`models.py`、`suggestions.py`、`auth.py`、`assistants_compat.py`。这些 HTTP 端点处理的是「一次请求的外壳」——鉴权、分页、SSE 流式输出——但它们最终都汇聚到同一个地方：那张唯一的 `lead_agent` 图。

回顾 `backend/langgraph.json`，除了 `graphs` 指向 `make_lead_agent`，它还声明了两个关键的可插拔点：

```json
"auth": { "path": "./app/gateway/langgraph_auth.py:auth" },
"checkpointer": { "path": "./packages/harness/deerflow/runtime/checkpointer/async_provider.py:make_checkpointer" }
```

鉴权挂在 `app` 层，checkpointer（决定运行状态怎么持久化）挂在 `harness` 层。这条配置就是「应用 vs 库」分层的缩影——对外的安全在外圈，对内的状态在内核。

## harness 包：十一个分区

`backend/packages/harness/deerflow/` 是本书真正要解剖的对象。它内部按子系统切成十几个分区，每个分区对应后面的一到两章：

| 分区 | 文件数（约） | 职责 | 对应章节 |
|---|---|---|---|
| `agents/` | 49 | agent 装配、中间件链、记忆 | 第 4、5、11 章 |
| `middlewares/` | 23 | 14 段中间件的实现 | 第 5、9、10 章 |
| `models/` | 15 | provider 抽象、patched_* 适配 | 第 6 章 |
| `tools/` | 28 | 工具注册、deferred search | 第 7 章 |
| `sandbox/` | 22 | 文件/命令执行隔离 | 第 8 章 |
| `subagents/` | 10 | 子 agent 执行引擎 | 第 12 章 |
| `skills/` | 13 | 技能解析、权限、扫描 | 第 13 章 |
| `mcp/` | — | MCP 会话池、缓存、OAuth | 第 14 章 |
| `config/` | 31 | AppConfig 级联与校验 | 第 15 章 |
| `persistence/` + `runtime/` | 23 + 31 | SQL 存储、run 调度、journal | 第 3、16 章 |
| `community/` | 28（12 个 provider） | 第三方搜索/抓取集成 | 第 7 章 |

`community/` 这个分区值得单独点一句：它下面挂着 12 个独立的 provider 目录——`tavily`、`serper`、`searxng`、`jina_ai`、`exa`、`brave`、`firecrawl`、`browserless`、`ddg_search`、`image_search`、`infoquest`、`aio_sandbox`。这印证了 DeerFlow「super agent」的定位：它不绑死某一家搜索/抓取服务，而是把它们都做成可插拔的社区集成。

## 一次请求的生命线

把上面的分区串起来，一次真实请求穿过系统的路径大致是：

1. HTTP 请求打到 `app/gateway/routers/thread_runs.py` 之类的路由器，经过 `langgraph_auth.py` 鉴权。
2. 网关创建一个 run，交给 `runtime/runs/manager.py` 的 `RunManager` 登记、`runtime/runs/worker.py` 的 worker 执行（第 3 章详解）。
3. worker 调用 `make_lead_agent` 构造出 `lead_agent` 图——这一步完成模型解析、工具组装、以及那条 14 段中间件链的装配（第 4、5 章）。
4. 图开始转动：LangGraph 的主循环驱动「模型节点 → 工具节点」，每一跳都被中间件钩子层层包裹。
5. 全程由 `runtime/journal.py` 的 `RunJournal`（一个 LangChain 回调）旁路捕获事件、累加 token，写入可插拔的 `RunEventStore`（第 16 章）。
6. 结果以 SSE 流式回吐给前端，run 状态最终落到持久化层。

## 贯穿全程的不变量：ThreadState

如果说中间件链是 DeerFlow 的「神经」，那 `agents/thread_state.py` 里的 `ThreadState` 就是它的「血液」——所有节点和中间件共享的那份状态。它继承自 LangChain 的 `AgentState`，在 `messages` 之外又加了若干 DeerFlow 自己的字段：`sandbox`、`thread_data`、`title`、`artifacts`、`todos`、`uploaded_files`、`viewed_images`、`promoted`。

这里藏着一个值得玩味的设计细节。LangGraph 里多个节点可能并发写同一个状态键，所以每个共享键都要声明一个 **reducer**（合并函数）。看 `sandbox` 字段的 reducer `merge_sandbox`：当两个节点都尝试写入 sandbox_id 时，如果两者相同就幂等通过，如果**不同**，它不是随便选一个，而是直接抛错：

```python
if existing_id == new_id:
    return existing
raise ValueError(f"Conflicting sandbox state updates: {existing_id!r} != {new_id!r}")
```

注释说得很直白：「Different sandbox ids in the same thread indicate a lifecycle/isolation bug, so fail closed instead of choosing one silently.」（同一线程里出现不同的 sandbox id 意味着生命周期/隔离 bug，所以宁可失败关闭也不静默选一个。）这正是第 17 章会提炼的「默认即关闭」原则在状态层的第一次现身——连一个看似无害的状态合并，DeerFlow 都选择了「出错就炸，别蒙混」。

其他 reducer 各有性格：`merge_artifacts` 合并去重、`merge_todos` 取最后一次非 None 的写入、`merge_promoted` 按 `catalog_hash` 分桶——当工具目录的哈希变了，旧的 promoted 工具名会被整体丢弃，注释解释这是为了「防止一个持久化的裸名字在目录漂移后指向了另一个工具」。这些 reducer 单独看都不起眼，但它们共同构成了 DeerFlow 在并发状态下的安全网。

## 本章小结

- DeerFlow 2.0 是一个前后端分治的 monorepo，后端 `backend/` 又分 `app/`（HTTP 网关）与 `packages/harness/`（核心库），通过 `deerflow-harness` workspace 包解耦。
- `agents/factory.py` 明确声明工厂装配「config-free」：纯参数进、agent 出，这是后续可测试性的基础。
- 网关层有 17 个路由器，覆盖会话、运行、技能、MCP、记忆、IM 渠道等能力，但全部汇聚到唯一的 `lead_agent` 图。
- `langgraph.json` 把 `auth` 挂在 app 层、`checkpointer` 挂在 harness 层，体现「对外安全在外圈、对内状态在内核」的分层。
- `community/` 下挂着 12 个可插拔的第三方 provider，印证了 DeerFlow「不绑死单一服务」的 super agent 定位。
- 全局共享状态 `ThreadState` 继承自 LangChain `AgentState`，每个共享键都有 reducer；`merge_sandbox` 在 sandbox_id 冲突时直接抛错，是「默认即关闭」原则在状态层的首次现身。

## 动手实验

1. **实验一：画出仓库分区图** — 用 `ls -d */` 列出仓库根目录，再用 `ls -d backend/packages/harness/deerflow/*/` 列出 harness 包的分区。对照本章的分区表，把每个分区标注上「它负责什么 / 对应哪一章」，贴在你手边。
2. **实验二：验证 app 与 harness 的依赖方向** — 打开 `backend/pyproject.toml`，找到 `[tool.uv.sources]` 与 `[tool.uv.workspace]`，确认 `app` 依赖 `deerflow-harness` 而非反过来。再用 `grep -rn "from app" backend/packages/harness/` 验证 harness 包**没有**反向 import app——这是分层是否干净的试金石。
3. **实验三：清点对外能力** — 列出 `backend/app/gateway/routers/` 下所有文件，给每个路由器写一句话它对外提供什么能力。哪些是「会话核心」，哪些是「扩展」，哪些是「IM 渠道」？这份清单就是 DeerFlow 的产品功能地图。
4. **实验四：读懂一个 reducer 的 fail-closed** — 打开 `agents/thread_state.py`，逐个读 `merge_sandbox`、`merge_artifacts`、`merge_todos`、`merge_promoted` 四个 reducer。找出哪一个选择「冲突即抛错」、哪一个选择「合并去重」、哪一个选择「后写覆盖」。思考：为什么唯独 `sandbox` 用了最严格的抛错策略？

> **下一章预告**：全景坐标系已经就位。下一章我们钻进运行时内核，看一次请求如何被 `RunManager` 登记成 run、被 worker 拉起执行、在断连时如何处置——以及 DeerFlow 如何用一个带 SQLite 重试策略的注册表，让运行状态熬过进程重启。
