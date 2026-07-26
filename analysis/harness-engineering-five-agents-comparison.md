# 同一个心脏，五种活法：OpenClaw / Hermes / OpenCode / Codex / OpenWorker 的 Harness 工程对比

> Harness（执行骨架）是包裹在大模型外面的那层工程--它决定模型"能调什么工具、看到什么上下文、在什么边界上被打断、出错后怎么恢复、副作用能不能落地"。同一个 LLM，套上不同的 harness，就长成性格迥异的 Agent。本文从十个工程维度切开 OpenClaw、Hermes、OpenCode、Codex、OpenWorker 五个 Agent，看它们在同一道道工程难题前各自给出的答案--以及答案背后的取舍。

读这五套代码，最强烈的感受是：**它们解决的是同一组问题，但每一套都把自己的"工程性格"刻进了最微小的常量与边界里**。OpenClaw 偏执，OpenCode 严谨，Codex 保守诚实，Hermes 务实粗粝，OpenWorker 务实守门。下面逐维度展开。

> **关于本文**：本文是 [四种 Agent 对比文章](./harness-engineering-four-agents-comparison.md) 的扩展版，加入了 OpenWorker 作为第五个 Agent。原有的四个 Agent 对比内容保持不变，在每个维度末尾新增 OpenWorker 的分析和五方对比。选择新建文章而非扩展原文，是为了保留原文作为四 Agent 对比的独立参考。

---

## 一、整体工程性格：一句话画像

| Agent | 语言/栈 | 一句话性格 | 最能代表它的设计 |
|---|---|---|---|
| **OpenClaw** | TypeScript | **偏执型 fail-closed**--"模型只能加锁，不能开锁" | 工具策略流水线 `minSecurity/maxAsk`，权限是单调收紧的 |
| **OpenCode** | TypeScript + Effect | **严谨型**--"宁可受限也不冒险"，把不变量焊进类型 | Effect 的 `Effect<A,E>` 把失败穷举进类型、事件溯源 + SQLite |
| **Codex** | Rust | **保守诚实型**--"在每个边界上选择诚实与保守" | 多后端沙箱 + Responses-only + apply_patch 上下文匹配 |
| **Hermes** | Python | **务实粗粝型**--"为无人值守、多平台、长时运行而生" | 凭证池轮换 + 10+ IM 网关 + cron 无人值守 |
| **OpenWorker** | Python | **务实守门型**--"在桌面上交付完成品，每一步都审批门控" | Inbox 持久化审批 + 四档风险分类 + 25+ 连接器 |

这五种性格不是风格偏好，而是**部署形态倒逼出来的**：OpenClaw/OpenCode 面向开发者本地交互，所以把"别让模型在我机器上闯祸"做到极致；Codex 要在云端大规模跑不可信代码，所以沙箱是一等公民；Hermes 要 7×24 挂在十个 IM 平台上无人值守，所以容错和凭证轮换压倒一切；OpenWorker 是桌面端原生应用，用户可能随时离开，所以审批必须持久化、可跨 surface 回答。

---

## 二、主循环与控制流：自主性的物理来源

五个 Agent 的"心脏"都是同一个抽象--**生成 -> 调工具 -> 把结果回灌 -> 再生成**，循环直到模型不再调工具。但实现风格分五路：

- **OpenClaw** 用最朴素的双层 `while(true)`。它的自主性来自一行 `currentContext.messages.push(result)`--把工具结果推回消息数组，下一轮模型就看见了。三层 API（`agentLoop`/`runAgentLoop`/`runLoop`）分别面向不同调用者，`executeToolCalls` 按 `executionMode` 决定并行还是串行，`beforeToolCall`/`afterToolCall`/`shouldStopAfterTurn` 三个钩子让外部插桩。

- **Hermes** 是个自由函数 `run_conversation(agent, ...)`，把 `AIAgent` 显式传入（便于独立测试）。循环条件最有料：`(api_call_count < max_iterations and iteration_budget.remaining > 0) or _budget_grace_call`--次数上限、独立预算、外加一个"宽限调用"让 Agent 在预算耗尽时还能留一句遗言。它的三道 `< 3` 重试防线（压缩、长度续写、截断工具调用）是失败恢复"有界"哲学的教科书。

- **OpenCode** 最结构化：三层嵌套循环--**轮次**（模型这轮要不要调工具）/**步**（`for step < MAX_STEPS=25`，工具回灌后还续不续）/**活动**（还有没有 queue 输入）。撞上 25 步不是悄悄停，而是显式抛 `StepLimitExceededError(sessionID, limit)`。更精巧的是用类型化的 `TurnTransition` 信号（`RebuildPreparedTurn`/`ContinueAfterOverflowCompaction`）配 `catchDefect` 递归驱动"重来"，而不是嵌套一堆 `if (shouldRetry) continue`。

- **Codex** 用 Rust 的状态机：`CodexThread` -> `SessionTask`/`RegularTask` -> `run_turn`。它有独特的"准入拒绝"枚举 `TryStartTurnIfIdleRejectionReason`（PendingTriggerTurn/PlanMode/Busy），把"现在能不能开新回合"显式建模。

- **OpenWorker** 的 `TurnEngine._loop()` 是一个 `while True` 循环，`max_iterations=12` 为默认上限。它的独特之处在于**阻塞 provider + 异步引擎**的桥接设计：provider 的 `stream()` 是同步生成器，跑在一个线程里通过 `asyncio.Queue` 把 chunk 推回事件循环。工具执行也是同步的（`asyncio.to_thread`），但低风险工具可以并发（`asyncio.gather`）。中断设计特别细致：`_interruptible()` 方法让审批等待也可以被 cancel 打断，且每个未执行的工具都得到一个 `interrupted` 结果（不留孤儿）。

**共性**：都有硬步数/迭代上限防失控；都把工具结果当一等公民回灌。**分歧**：OpenClaw 把自主性藏在一行 push 里（隐式），OpenCode 把每种"还没完"拆成独立循环层（显式）；Hermes 把恢复策略堆在重试计数器上，OpenCode/Codex 把它们提升成类型化信号/状态；OpenWorker 用"阻塞桥接 + 不留孤儿中断"的简洁设计实现了同等能力。

---

## 三、准入与执行分离：OpenCode 的独门设计

这是五者里差异最大的一维。OpenClaw/Hermes/Codex/OpenWorker 基本是"收到 prompt 就同步跑"，而 **OpenCode 把"准入"和"执行"刻意拆开**：

`SessionV2.prompt` 先把 prompt 写成一行持久化的 `session_input` 记录（准入），再调度一次 advisory `wake`（执行）。落到模型执行那一层永远是有序、可恢复的。

对比之下，其他四家的 steering 实现更轻量：OpenClaw/Hermes 靠在循环顶部检查待处理消息，Codex 靠 `TryStartTurnIfIdle` 的拒绝原因，OpenWorker 用 `_steering` 列表在两处检查点注入（模型无工具调用时和工具调用处理完毕后）。**OpenCode 是唯一把这件事提升为一等领域问题、用事件溯源兜底的**--代价是复杂度，收益是崩溃后不丢活。

OpenWorker 的 steering 注入虽然简单，但有一个独特设计：steering 消息携带 `source` sidecar（连接器来源信息），被持久化到消息历史中但被 `_outbound_messages()` 在发送给 provider 之前剥离。

---

## 四、工具系统：从"函数表"到"策略流水线"

工具是 harness 与外部世界的接口，五家的注册与治理思路分化明显：

- **OpenClaw**：`ToolDescriptor` + 策略流水线 + exec 包装三层。核心是 `createExecTool`（约 2000 行）的 `minSecurity`/`maxAsk`--**模型只能把安全级别往上加锁，永远不能往下解锁**。这是"偏执"二字的源头。

- **OpenCode**：`Tool.make` 用 `Object.freeze` + `WeakMap` 把工具定义冻成不可变。`bash.ts` 的常量值得抄作业：`DEFAULT_TIMEOUT_MS=2min`/`MAX_TIMEOUT_MS=10min`/`MAX_CAPTURE_BYTES=1MB`。16 种 `LLMEvent` 类型把流式事件穷举进类型。

- **Codex**：工具暴露有阈值--`DIRECT_MCP_TOOL_EXPOSURE_THRESHOLD=100`，超过 100 个 MCP 工具就不再直接平铺。最有特色的是 `apply_patch`：用 Lark 文法解析、**基于上下文而非行号匹配**（`seek_sequence` 四级 fallback），这让补丁在文件轻微变动后仍能命中。

- **Hermes**："导入即注册"--用 AST 扫描在导入时自动收集工具，不需要手写注册表。命令安全分两档：`HARDLINE_PATTERNS`（12 条无条件拦截）vs `DANGEROUS_PATTERNS`（约 47 条需确认）。`delegate_task` 支持子 Agent 委派。

- **OpenWorker**：`ToolRegistry` 极简（注册/查询/执行三操作），schema 生成复用 aisuite。核心创新是 **`risk.py` 的四档分类**（READ / WRITE_LOCAL / EXEC / EXTERNAL）--不是按工具名做特判，而是按内在副作用分类。`_parallel_safe()` 让低风险工具并发执行、高风险工具串行。Shell 命令有三层防护：`_SHELL_OPERATORS` 操作符黑名单（`; & | > < \` $( (`）+ `shlex.split` token 级前缀匹配 + 空命令拒绝。

**共性**：都在工具层做安全治理，都支持 MCP。**分歧**：OpenClaw 把治理做成单调收紧的流水线（最严），OpenCode 把工具做成不可变值（最干净），Codex 在补丁工具上下了最多功夫（最适配代码场景），Hermes 用模式匹配做黑名单（最务实），OpenWorker 用四档分类法做分类驱动（最清晰）。

---

## 五、执行与沙箱：谁把"不可信代码"当头号敌人

这一维 Codex 和 Hermes 远超另外三家--因为它们要跑别人/自己生成的、不可信的代码。

- **Codex** 把沙箱做成跨平台一等公民：`SandboxType` 枚举 None / **MacosSeatbelt** / **LinuxSeccomp**（seccomp+Landlock）/ **WindowsRestrictedToken**（受限令牌 + WFP）。网络**默认关闭**。审批 `AskForApproval` 分 5 个等级，配 `execpolicy` crate 和 `BANNED_PREFIX_SUGGESTIONS`。

- **Hermes** 有 **6 种沙箱后端**（统一在 `BaseEnvironment` ABC 下），从本地到容器到远程一应俱全。

- **OpenClaw/OpenCode/OpenWorker** 偏本地交互，沙箱较轻，主要靠工具层权限兜底，而非 OS 级隔离。OpenWorker 的路径约束（`_under_writable_root`）和 Shell 命令黑名单是工具层防护的代表。

**这维度最能说明"部署形态决定 harness"**：面向云端不可信负载 -> OS 级多后端沙箱（Codex/Hermes）；面向开发者/用户本机 -> 工具层权限闸门（OpenClaw/OpenCode/OpenWorker）。

---

## 六、权限与审批：fail-closed 的五种姿势

五家都信奉 fail-closed（"不确定时拒绝"），但落点不同：

- **OpenClaw**：`minSecurity/maxAsk` 单调收紧 + 插件"能扩展不能削弱核心"。
- **OpenCode**：`permission.ts`（329 行）的 `missingAgentPermissions` **全部默认 deny**；`SystemContext.InitializationBlocked` 体现"宁可阻塞这一轮，也不持久化一个不完整的基线"。
- **Codex**：5 级审批 + execpolicy + 网络默认关。
- **Hermes**：`HARDLINE_PATTERNS` 12 条无条件拦截 + 凭证配对 `PairingStore` 用 `secrets.compare_digest`（防时序攻击）。
- **OpenWorker**：四模式（DISCUSS / PLAN / INTERACTIVE / AUTO）+ 四档风险分类 + 五重门控 + Inbox 持久化审批。**独特设计**：审批是可持久化的 `InboxItem`，有 `tool_call_id` 幂等键，resolve-once 状态机让"从任何 surface 回答都安全"。standing rules 允许"always allow this tool for this target"的精确授权（不是 blanket allow）。Shell 命令的 `_SHELL_OPERATORS` 黑名单 + token 级前缀匹配是命令安全的微观最佳实践。

这是五者**最趋同**的一维--大家都同意：Agent 安全的默认值必须是"关"，开口子要显式。**但 OpenWorker 的 Inbox 系统把审批从"同步对话框"变成了"持久化队列"**，这是五者中独一无二的。

---

## 七、上下文压缩与记忆：留结构、去内容

**压缩**的共识是"从便宜到昂贵、从无损到有损"：先修剪/截断工具输出，再无损保护头尾，最后才动用昂贵的 LLM 摘要。

- **Hermes**：`ContextEngine(ABC)` 可插拔；默认 `threshold_percent` 从 0.75 下调到 0.50（早压勤压）、`protect_last_n` 6->20、`MINIMUM_CONTEXT_LENGTH=64_000` 地板、抗抖动开关。
- **OpenCode**：`DEFAULT_KEEP_TOKENS=8000`、`SUMMARY_TEMPLATE` 7 个固定章节、`Compaction.Ended` 事件触发纪元替换。
- **Codex**：`COMPACT_USER_MESSAGE_MAX_TOKENS=20_000`、`remove_first_item` 时刻意保留前缀以命中 prefix cache。
- **OpenClaw**：`preemptive-compaction.ts` 预压缩 + `compactWithSafetyTimeout` 带超时兜底。
- **OpenWorker**：目前没有显式的上下文压缩机制（`max_iterations=12` 是主要的上下文增长限制）。`_outbound_messages()` 在每次 provider 调用时剥离 sidecar（`source`、`_display`、`ts`、`reasoning`）和 `notice` 消息，减少了发给模型的内容量，但这不是真正的压缩。记忆系统通过 system prompt 注入（`format_memories`），不参与压缩。

**记忆**分两层：**声明式**（Hermes 的 `MEMORY.md`、Codex 的 `AGENTS.md`、OpenWorker 的三级 MemoryStore）和**程序式**（skills）。OpenWorker 的记忆用 SQLite 持久化，三级作用域（GLOBAL / WORKSPACE / SESSION），每条记忆前缀 `[#id]` 让 Agent 可以精确修改/删除。

---

## 八、持久化：从"内存数组"到"事件溯源"

差异最悬殊的一维：

- **OpenCode** 是唯一把会话做成**事件溯源 + SQLite** 的：Drizzle ORM、6 个 PRAGMA、`uniqueIndex` 做乐观并发。
- **Hermes**：`hermes_state.py` 用 SQLite + **双 FTS5**（trigram 专为中文全文检索）。
- **Codex/OpenClaw**：相对轻量，以会话文件/内存状态为主。
- **OpenWorker**：**SQLite 索引 + JSONL 消息日志** 双层。SQLite 存会话元数据（`coworker.db`），JSONL 存消息（`conversations/<id>.jsonl`，append-only）。设计思想是"元数据需要查询（用 SQLite），消息需要追加（用 JSONL）"。Inbox 用 JSON 文件（数据量小，全量重写可接受）。grants（session 级审批授权）持久化在 `SessionRecord.grants` 中，重启后恢复到 `PermissionEngine` 的 session allow lists。

OpenWorker 的 `notice` 消息设计值得注意：error/interrupted/model_switch 标记被持久化到 JSONL（重载后用户可见），但在 `_outbound_messages()` 中被剥离（模型看不到）。这解决了"显示事件需要持久化但不能污染模型上下文"的矛盾。

---

## 九、扩展体系、网关与可观测性

- **扩展**：五家都支持 MCP + 插件/skills。OpenClaw 的红线是"插件能扩展不能削弱核心，MCP 是前门"；Codex 用阈值控制 MCP 工具平铺。OpenWorker 有三层扩展：Skill（渐进式披露）、Persona（可安装的 Agent 模板，需要 consent）、Subagent（只读探索）。skills 普遍用渐进式披露省 token。

- **网关/路由**：**Hermes 独占鳌头**--10+ IM 平台消息网关 + `CredentialPool`（4 种轮换策略）。OpenWorker 有 25+ 连接器 + Slack/GitHub 托管中继 + 自动化调度（`Scheduler` 30s tick + run-once-catch-up + skip-on-overlap）+ selfwake（Agent 自暂停与恢复）。OpenClaw 用 `callGateway` 单入口 + `operator.*` scopes。

- **可观测性与脱敏**：Codex 用 OTEL + Statsig；Hermes 用 `RedactingFormatter`；OpenClaw 用 `DEFAULT_REDACT_MODE="tools"`。**OpenWorker** 用 `AuditStore`（SQLite 审计日志，记录每个工具调用的完整生命周期）+ `_sanitize_args()`（`_SECRET_KEYS` + `_BODY_KEYS` + `browser_type` 特殊处理）+ `Event` 系统（15 种 EventType 通过 WebSocket 实时推送）。

---

## 十、测试与可信度：数字会说话

| Agent | 测试规模 | 信号 |
|---|---|---|
| **OpenClaw** | **3628 个单元测试文件，约 80 个 vitest 分片** | 偏执也体现在测试覆盖上 |
| **Hermes** | **1489 个测试文件** | 长时运行靠测试兜底 |
| **Codex** | **425+ 测试文件** | Rust 类型系统分担了一部分 |
| **OpenCode** | **128 个测试文件** | Effect 类型 + 事件溯源把不变量前移到编译期/架构层 |
| **OpenWorker** | **后端 pytest + 前端 hermetic e2e** | 桌面应用需要端到端测试覆盖 Tauri+Python 全链路 |

这张表有意思：**测试文件数与"靠类型/架构保证正确性的程度"成反比**。OpenClaw 用海量测试堆出信心；OpenCode 用 `Effect<A,E>` 把失败穷举进类型、用事件溯源把状态变更变可重放，于是需要的运行时测试反而少。OpenWorker 选择了 hermetic e2e（端到端密封测试），因为它的三层架构（Tauri + Python + 连接器）的集成点比单层应用多。

---

## 结语：harness 是 Agent 真正的差异化所在

五个 Agent 调的可能是同一批底层模型，但它们是五个完全不同的产品，差异**几乎全在 harness 里**：

- 想要**本地交互、绝不在我机器上闯祸** -> 学 OpenClaw 的单调收紧策略流水线；
- 想要**崩溃不丢活、并发可信、审计完整** -> 学 OpenCode 的准入/执行分离 + 事件溯源；
- 想要**云端跑不可信代码、补丁稳健** -> 学 Codex 的多后端沙箱 + apply_patch 上下文匹配；
- 想要**无人值守、多平台、长时运行** -> 学 Hermes 的凭证池轮换 + IM 网关 + cron；
- 想要**桌面端交付完成品、审批可持久化** -> 学 OpenWorker 的 Inbox 系统 + 四档风险分类 + 25+ 连接器。

而贯穿五者的几条共识，可能才是 harness 工程最该被记住的"硬通货"：**失败恢复必须有界**（Hermes 三道 `<3`、OpenCode 的 `MAX_STEPS=25`、Codex 只允许压缩恢复一次、OpenWorker 的 `max_iterations=12` + 不留孤儿中断）；**默认即关闭**（全员 fail-closed）；**留结构、去内容**（压缩占位符、conversation-checkpoint、notice 消息剥离）；**错误即文档**（把诊断信息塞进错误对象/类型）；**可信度建在流水线里**（类型、测试、事件溯源各择其一）。

一句话：**模型决定 Agent 聪不聪明，harness 决定 Agent 可不可信、能不能在真实世界里活下来。**
