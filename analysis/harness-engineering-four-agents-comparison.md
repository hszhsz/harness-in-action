# 同一个心脏，四种活法：OpenClaw / Hermes / OpenCode / Codex 的 Harness 工程对比

> Harness（执行骨架）是包裹在大模型外面的那层工程——它决定模型"能调什么工具、看到什么上下文、在什么边界上被打断、出错后怎么恢复、副作用能不能落地"。同一个 LLM，套上不同的 harness，就长成性格迥异的 Agent。本文从十个工程维度切开 OpenClaw、Hermes、OpenCode、Codex 四个 Agent，看它们在同一道道工程难题前各自给出的答案——以及答案背后的取舍。

读这四套代码，最强烈的感受是：**它们解决的是同一组问题，但每一套都把自己的"工程性格"刻进了最微小的常量与边界里**。OpenClaw 偏执，OpenCode 严谨，Codex 保守诚实，Hermes 务实粗粝。下面逐维度展开。

---

## 一、整体工程性格：一句话画像

| Agent | 语言/栈 | 一句话性格 | 最能代表它的设计 |
|---|---|---|---|
| **OpenClaw** | TypeScript | **偏执型 fail-closed**——"模型只能加锁，不能开锁" | 工具策略流水线 `minSecurity/maxAsk`，权限是单调收紧的 |
| **OpenCode** | TypeScript + Effect | **严谨型**——"宁可受限也不冒险"，把不变量焊进类型 | Effect 的 `Effect<A,E>` 把失败穷举进类型、事件溯源 + SQLite |
| **Codex** | Rust | **保守诚实型**——"在每个边界上选择诚实与保守" | 多后端沙箱 + Responses-only + apply_patch 上下文匹配 |
| **Hermes** | Python | **务实粗粝型**——"为无人值守、多平台、长时运行而生" | 凭证池轮换 + 10+ IM 网关 + cron 无人值守 |

这四种性格不是风格偏好，而是**部署形态倒逼出来的**：OpenClaw/OpenCode 面向开发者本地交互，所以把"别让模型在我机器上闯祸"做到极致；Codex 要在云端大规模跑不可信代码，所以沙箱是一等公民；Hermes 要 7×24 挂在十个 IM 平台上无人值守，所以容错和凭证轮换压倒一切。

---

## 二、主循环与控制流：自主性的物理来源

四个 Agent 的"心脏"都是同一个抽象——**生成 → 调工具 → 把结果回灌 → 再生成**，循环直到模型不再调工具。但实现风格分四路：

- **OpenClaw** 用最朴素的双层 `while(true)`。它的自主性来自一行 `currentContext.messages.push(result)`——把工具结果推回消息数组，下一轮模型就看见了。三层 API（`agentLoop`/`runAgentLoop`/`runLoop`）分别面向不同调用者，`executeToolCalls` 按 `executionMode` 决定并行还是串行，`beforeToolCall`/`afterToolCall`/`shouldStopAfterTurn` 三个钩子让外部插桩。

- **Hermes** 是个自由函数 `run_conversation(agent, ...)`，把 `AIAgent` 显式传入（便于独立测试）。循环条件最有料：`(api_call_count < max_iterations and iteration_budget.remaining > 0) or _budget_grace_call`——次数上限、独立预算、外加一个"宽限调用"让 Agent 在预算耗尽时还能留一句遗言。它的三道 `< 3` 重试防线（压缩、长度续写、截断工具调用）是失败恢复"有界"哲学的教科书。

- **OpenCode** 最结构化：三层嵌套循环——**轮次**（模型这轮要不要调工具）/**步**（`for step < MAX_STEPS=25`，工具回灌后还续不续）/**活动**（还有没有 queue 输入）。撞上 25 步不是悄悄停，而是显式抛 `StepLimitExceededError(sessionID, limit)`。更精巧的是用类型化的 `TurnTransition` 信号（`RebuildPreparedTurn`/`ContinueAfterOverflowCompaction`）配 `catchDefect` 递归驱动"重来"，而不是嵌套一堆 `if (shouldRetry) continue`。

- **Codex** 用 Rust 的状态机：`CodexThread` → `SessionTask`/`RegularTask` → `run_turn`。它有独特的"准入拒绝"枚举 `TryStartTurnIfIdleRejectionReason`（PendingTriggerTurn/PlanMode/Busy），把"现在能不能开新回合"显式建模。

**共性**：都有硬步数/迭代上限防失控；都把工具结果当一等公民回灌。**分歧**：OpenClaw 把自主性藏在一行 push 里（隐式），OpenCode 把每种"还没完"拆成独立循环层（显式）；Hermes 把恢复策略堆在重试计数器上，OpenCode/Codex 把它们提升成类型化信号/状态。

---

## 三、准入与执行分离：OpenCode 的独门设计

这是四者里差异最大的一维。OpenClaw/Hermes/Codex 基本是"收到 prompt 就同步跑"，而 **OpenCode 把"准入"和"执行"刻意拆开**：

`SessionV2.prompt` 先把 prompt 写成一行持久化的 `session_input` 记录（准入），再调度一次 advisory `wake`（执行）。落到模型执行那一层永远是有序、可恢复的。三个协作者各司其职：`SessionRunner`（只认 sessionID、从持久化历史排空活）、`SessionExecution`（wake/resume/interrupt 遥控器）、`SessionRunCoordinator`（保证每 Session 串行、跨 Session 并发的交通警）。

这套机制专门解决三类竞态：**执行中插话（steering）、进程崩溃恢复、并发戳同一会话**。Coordinator 的并发纪律——`run` 压倒 `wake`、多次 wake 被 `coalesce` 折叠、用单调 `seq` 划定中断边界、用 `uninterruptibleMask` 保证状态转换原子——与 LLM 完全无关，却决定了这个 Agent 在真实并发下"可信"还是"能跑"。

对比之下，其他三家的 steering 实现更轻量：OpenClaw/Hermes 靠在循环顶部检查待处理消息，Codex 靠 `TryStartTurnIfIdle` 的拒绝原因。**OpenCode 是唯一把这件事提升为一等领域问题、用事件溯源兜底的**——代价是复杂度，收益是崩溃后不丢活。

---

## 四、工具系统：从"函数表"到"策略流水线"

工具是 harness 与外部世界的接口，四家的注册与治理思路分化明显：

- **OpenClaw**：`ToolDescriptor` + 策略流水线 + exec 包装三层。核心是 `createExecTool`（约 2000 行）的 `minSecurity`/`maxAsk`——**模型只能把安全级别往上加锁，永远不能往下解锁**。这是"偏执"二字的源头。

- **OpenCode**：`Tool.make` 用 `Object.freeze` + `WeakMap` 把工具定义冻成不可变。`bash.ts` 的常量值得抄作业：`DEFAULT_TIMEOUT_MS=2min`/`MAX_TIMEOUT_MS=10min`/`MAX_CAPTURE_BYTES=1MB`。16 种 `LLMEvent` 类型把流式事件穷举进类型。

- **Codex**：工具暴露有阈值——`DIRECT_MCP_TOOL_EXPOSURE_THRESHOLD=100`，超过 100 个 MCP 工具就不再直接平铺。最有特色的是 `apply_patch`：用 Lark 文法解析、**基于上下文而非行号匹配**（`seek_sequence` 四级 fallback），这让补丁在文件轻微变动后仍能命中。

- **Hermes**："导入即注册"——用 AST 扫描在导入时自动收集工具，不需要手写注册表。命令安全分两档：`HARDLINE_PATTERNS`（12 条无条件拦截）vs `DANGEROUS_PATTERNS`（约 47 条需确认）。`delegate_task` 支持子 Agent 委派。

**共性**：都在工具层做安全治理，都支持 MCP。**分歧**：OpenClaw 把治理做成单调收紧的流水线（最严），OpenCode 把工具做成不可变值（最干净），Codex 在补丁工具上下了最多功夫（最适配代码场景），Hermes 用模式匹配做黑名单（最务实）。

---

## 五、执行与沙箱：谁把"不可信代码"当头号敌人

这一维 Codex 和 Hermes 远超另外两家——因为它们要跑别人/自己生成的、不可信的代码。

- **Codex** 把沙箱做成跨平台一等公民：`SandboxType` 枚举 None / **MacosSeatbelt** / **LinuxSeccomp**（seccomp+Landlock）/ **WindowsRestrictedToken**（受限令牌 + WFP）。网络**默认关闭**。审批 `AskForApproval` 分 5 个等级，配 `execpolicy` crate 和 `BANNED_PREFIX_SUGGESTIONS`。这是为"在云端跑陌生代码"而生的纵深防御。

- **Hermes** 有 **6 种沙箱后端**（统一在 `BaseEnvironment` ABC 下），从本地到容器到远程一应俱全——因为它要适配各种无人值守部署环境。

- **OpenClaw/OpenCode** 偏本地交互，沙箱较轻，主要靠工具层权限（`minSecurity`、`permission.ts` 的 all-deny）兜底，而非 OS 级隔离。

**这维度最能说明"部署形态决定 harness"**：面向云端不可信负载 → OS 级多后端沙箱（Codex/Hermes）；面向开发者本机 → 工具层权限闸门（OpenClaw/OpenCode）。

---

## 六、权限与审批：fail-closed 的四种姿势

四家都信奉 fail-closed（"不确定时拒绝"），但落点不同：

- **OpenClaw**：`minSecurity/maxAsk` 单调收紧 + 插件"能扩展不能削弱核心"。
- **OpenCode**：`permission.ts`（329 行）的 `missingAgentPermissions` **全部默认 deny**；`SystemContext.InitializationBlocked` 体现"宁可阻塞这一轮，也不持久化一个不完整的基线"。
- **Codex**：5 级审批 + execpolicy + 网络默认关。
- **Hermes**：`HARDLINE_PATTERNS` 12 条无条件拦截 + 凭证配对 `PairingStore` 用 `secrets.compare_digest`（防时序攻击）。

这是四者**最趋同**的一维——大家都同意：Agent 安全的默认值必须是"关"，开口子要显式。

---

## 七、上下文压缩与记忆：留结构、去内容

**压缩**的共识是"从便宜到昂贵、从无损到有损"：先修剪/截断工具输出，再无损保护头尾，最后才动用昂贵的 LLM 摘要。

- **Hermes**：`ContextEngine(ABC)` 可插拔；默认 `threshold_percent` 从 0.75 下调到 0.50（早压勤压）、`protect_last_n` 6→20、`MINIMUM_CONTEXT_LENGTH=64_000` 地板、抗抖动开关（连续两次省空间 <10% 就停）、五步流水线。
- **OpenCode**：`DEFAULT_KEEP_TOKENS=8000` 从尾往头划折叠线、`SUMMARY_TEMPLATE` 7 个固定章节、`Compaction.Ended` 事件触发纪元替换、渲染成 `<conversation-checkpoint>`（标注"历史上下文，非新指令"）。增量压缩把上次摘要当 `<previous-summary>` 锚点更新。
- **Codex**：`COMPACT_USER_MESSAGE_MAX_TOKENS=20_000`、`remove_first_item` 时刻意保留前缀以命中 prefix cache。
- **OpenClaw**：`preemptive-compaction.ts` 预压缩 + `compactWithSafetyTimeout` 带超时兜底。

**记忆**分两层：**声明式**（Hermes 的 `MEMORY.md` 2200 字符/`USER.md` 1375 字符、Codex 的 `AGENTS.md` 32 KiB 上限）和**程序式**（skills）。Hermes 还有 `honcho` 外部记忆（fail-open——记忆服务挂了不阻塞主流程）。共同的安全细节：记忆加载时做净化（`scan_for_threats`），防止"记忆投毒"变成提示注入。

**`_PRUNED_TOOL_PLACEHOLDER` / `<conversation-checkpoint>` / `(none)` 空章节**——这些都是同一原则的微观实例：**压缩丢掉冗长叙述，死保 Agent 续跑所需的硬事实**。

---

## 八、持久化：从"内存数组"到"事件溯源"

差异最悬殊的一维：

- **OpenCode** 是唯一把会话做成**事件溯源 + SQLite** 的：Drizzle ORM、6 个 PRAGMA（`WAL`/`synchronous=NORMAL`/`busy_timeout=5000`/`cache_size=-64000`/`foreign_keys=ON`）、`uniqueIndex` 做乐观并发。每次状态变更都是可重放、可恢复、可审计的持久记录。压缩后旧历史**仍留库审计，只是离开模型视野**——审计完整与上下文精简兼得。
- **Hermes**：`hermes_state.py` 用 SQLite + **双 FTS5**（其中 trigram 专为中文全文检索）。
- **Codex/OpenClaw**：相对轻量，以会话文件/内存状态为主。

OpenCode 的事件溯源不是炫技——它是"准入/执行分离"和"崩溃恢复"能成立的底座。**没有持久化的事件流，steering 和崩溃续跑都无从谈起**。

---

## 九、扩展体系、网关与可观测性

- **扩展**：四家都支持 MCP + 插件/skills。OpenClaw 的红线是"插件能扩展不能削弱核心，MCP 是前门"；Codex 用阈值控制 MCP 工具平铺。skills 普遍用渐进式披露（progressive disclosure）省 token。

- **网关/路由**：**Hermes 独占鳌头**——10+ IM 平台消息网关 + `CredentialPool`（4 种轮换策略，按错误码定制冷却 TTL：`EXHAUSTED_TTL_401=5min`/`429=1hr`）+ cron 无人值守（60s tick、at-most-once、`wakeAgent` 门控）。这是"7×24 多平台长时运行"性格的集中体现。OpenClaw 用 `callGateway` 单入口 + `operator.*` scopes。

- **可观测性与脱敏**：Codex 用 OTEL + Statsig；Hermes 用 `RedactingFormatter` 日志脱敏；OpenClaw 用 `DEFAULT_REDACT_MODE="tools"`。大家都默认在日志里抹掉敏感信息——又一个 fail-closed 的延伸。

---

## 十、测试与可信度：数字会说话

| Agent | 测试规模 | 信号 |
|---|---|---|
| **OpenClaw** | **3628 个单元测试文件，约 80 个 vitest 分片** | 偏执也体现在测试覆盖上 |
| **Hermes** | **1489 个测试文件** | 长时运行靠测试兜底 |
| **Codex** | **425+ 测试文件** | Rust 类型系统分担了一部分 |
| **OpenCode** | **128 个测试文件** | Effect 类型 + 事件溯源把不变量前移到编译期/架构层 |

这张表有意思：**测试文件数与"靠类型/架构保证正确性的程度"成反比**。OpenClaw 用海量测试堆出信心；OpenCode 用 `Effect<A,E>` 把失败穷举进类型、用事件溯源把状态变更变可重放，于是需要的运行时测试反而少。两条路通向同一个目的地——"可信度即流水线"。

---

## 结语：harness 是 Agent 真正的差异化所在

四个 Agent 调的可能是同一批底层模型，但它们是四个完全不同的产品，差异**几乎全在 harness 里**：

- 想要**本地交互、绝不在我机器上闯祸** → 学 OpenClaw 的单调收紧策略流水线；
- 想要**崩溃不丢活、并发可信、审计完整** → 学 OpenCode 的准入/执行分离 + 事件溯源；
- 想要**云端跑不可信代码、补丁稳健** → 学 Codex 的多后端沙箱 + apply_patch 上下文匹配；
- 想要**无人值守、多平台、长时运行** → 学 Hermes 的凭证池轮换 + IM 网关 + cron。

而贯穿四者的几条共识，可能才是 harness 工程最该被记住的"硬通货"：**失败恢复必须有界**（Hermes 三道 `<3`、OpenCode 的 `MAX_STEPS=25`、Codex 只允许压缩恢复一次）；**默认即关闭**（全员 fail-closed）；**留结构、去内容**（压缩占位符、conversation-checkpoint）；**错误即文档**（把诊断信息塞进错误对象/类型）；**可信度建在流水线里**（类型、测试、事件溯源各择其一）。

一句话：**模型决定 Agent 聪不聪明，harness 决定 Agent 可不可信、能不能在真实世界里活下来。**
