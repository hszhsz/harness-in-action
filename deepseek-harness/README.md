# 解构 Agent：DeepSeek Harness

> 一本面向 Agent 开发者的开源技术书籍 —— 通过逐层拆解 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（`dsh`）的真实代码，理解一个**「一切皆插件」**的生产级 Agent 系统是如何被工程化的。

## 关于本书

市面上关于 Agent 的内容大多停留在「Prompt 技巧」和「Demo 拼装」层面。但当你真正要把一个 Agent 做成**可约束、可观测、可复现、可迭代**的生产系统时，会发现真正的难点在于模型之外的那一整套工程系统——也就是 **Harness（驾驭层）**。

本书以 DeepSeek AI 开源的全新 Agent harness —— **DeepSeek Harness**（`dsh`，版本 `0.1.0-rc.5`，MIT）为标本。它不是又一个 CLI 壳子，而是一次激进的架构实验：**整个产品是一个跑在 vendored [Cordis](https://github.com/cordiverse/cordis) 插件框架上的插件树**——模型适配器是插件、工具注册表是插件、会话日志是插件、连 Agent 主循环本身都是插件。仓库里没有一块「特权核心」需要打补丁：你要扩展 dsh，就在别的插件旁边再挂一个插件，而所有注册都是可逆的 effect，插件卸载时自动回卷。

本书坚持一条铁律：**每一个技术论断都来自实际读过的源码**——确切的文件名、函数名、常量名、字面量值。我们不写"Agent 一般怎么做"，只写"DeepSeek Harness 在代码里怎么做"。

**读完本书你将掌握**：
- Cordis 插件模型的五个核心概念：插件、上下文、依赖注入、类型化事件、可逆 effect；
- 一个没有主循环特权的 Agent 系统如何用 **profile / bundle / patch 分层** 从空列表组合出完整产品；
- 为什么 **append-only 的 SessionEvent 日志**是唯一真相源，模型历史从日志派生而非单独存储（"Model-visible ⟺ logged" 不变量）；
- **Capability Seam（能力接缝）**三件套——Service Definition / Service Provider / Consumer——如何让换一个 Provider 就换掉整个产品（本地沙箱 ↔ E2B 远程沙箱）；
- 工具执行的六段守卫管线（pre-execute → guards → execute → post-execute → finalizeContent → tools/result）如何在不改主循环的前提下挂上审批、超时、spill、hook；
- 沙箱（bwrap / Landlock / Seatbelt / Windows ACL）、子 Agent 多 Provider、模型写脚本的 workflow、后台 job 运行时等现代 harness 工程的全套解法。

## 读者对象

- 正在或准备开发 Agent / Agent 框架的工程师
- 想理解「插件化 Agent 系统」如何做架构取舍的开发者
- 需要为团队搭建可落地、可替换、可审计 Harness 的技术负责人

**前置知识**：熟悉 TypeScript / Node.js，了解 LLM API 基本概念（消息、工具调用、流式输出）。有一点事件驱动或 DI 框架经验更好，但不是必需——第 3 章会把 Cordis 讲清楚。

## 阅读主线

本书遵循一条「从空列表到完整产品、再顺着一次请求穿过整个系统」的主线：

```
空 entry 列表
  → profile 组合（dsh-base → dsh-web-app / dsh-headless → 用户 cordis.patch.yml）
    → app-boot boot()（Loader 装载插件树，fail loud）
      → Cordis 运行时（ctx / services / typed events / reversible effects）
        → Session 事件日志（append-only SessionEvent，唯一真相源）
          → ReactLoopAgent 主循环（turn / step / inbox / steer / inject / cancel）
            → LLM 抽象与流式（Message / ContentBlock / StreamChunk / adapter seam）
              → 工具系统（ToolDefinition + 六段守卫管线）
                → Capability Seams（FS / Shell / Sandbox / Subagent / Workflow / Code Runtime…）
                  → 审批与交互（approval / user-questions / commands / presets）
                    → 持久化与崩溃恢复（JSONL / SQLite / flush checkpoint）
                      → 上下文、压缩与记忆（system-prompt / compaction / spill / skills / goals）
                        → 远程调用与可观测性（Typert RPC / events / telemetry / Web UI）
                          → 测试与工程可信度（snapshot / invariants / 防御模式）
```

---

## 目录

### 第一部分　导论

- **第 1 章　从 Prompt 到 Harness：插件化 Agent 的工程范式**
  - 1.1 `Agent = Model + Harness`：能力边界由谁决定
  - 1.2 「一切皆插件」：为什么 DeepSeek 敢把主循环也做成插件
  - 1.3 Harness 的六个核心职责在 dsh 中的映射
  - 1.4 本书方法论：以真实代码为唯一真理来源

- **第 2 章　DeepSeek Harness 全景：一个插件树的疆域**
  - 2.1 项目定位与设计哲学（开发者预览、foundation over blast radius）
  - 2.2 Monorepo 全景：`packages/` 54 个包、`vendor/`、`apps/`、`native/`、`python/`
  - 2.3 三条工程红线：一切皆插件、Model-visible ⟺ logged、Registrations are effects
  - 2.4 如何高效阅读这套代码（`AGENTS.md`、`docs/architecture.md`、generated catalogs）

### 第二部分　骨架：组合与运行时

- **第 3 章　Cordis：vendored 插件框架的五块基石**
  - 3.1 插件（Plugin）与上下文（Context）：`ctx.<key>` 服务仓库
  - 3.2 依赖注入：`inject` 声明取代手动启动排序
  - 3.3 类型化事件与四种派发模式：emit / waterfall / parallel / serial
  - 3.4 可逆 effect：注册即副作用，卸载即回卷
  - 3.5 `ctx.effect()` / `ctx.on()` 与插件生命周期

- **第 4 章　启动与组合：从空列表到完整产品**
  - 4.1 `dsh` CLI：`apps/cli/src/bin.ts` 的按需动态导入
  - 4.2 Profile / Bundle / Patch 三层组合（`dsh.profile` / `dsh.bundle` / `cordis.patch.yml`）
  - 4.3 `app-boot` 的 `boot()`：fail loud、partial-context 回卷、assertEntriesActivated
  - 4.4 `dsh-base` → `dsh-web-app` / `dsh-headless`：两个模板 profile 的解剖
  - 4.5 配置即代码：`--dump-config`、`!!js` 表达式、HMR 热更新 patch

### 第三部分　心脏：会话日志与主循环

- **第 5 章　Session 事件日志：append-only 的唯一真相源**
  - 5.1 `SessionEventMap`：turn / step / message / tool 的事件词汇表
  - 5.2 "Model-visible ⟺ logged"：为什么新输入必须先有新事件
  - 5.3 `deriveMessages()`：模型历史是日志的投影，不是独立存储
  - 5.4 Turn / Step / TurnEndReason：执行封闭（execution enclosure）

- **第 6 章　Agent 主循环：ReactLoopAgent 是如何跳动的**
  - 6.1 `Agent` 接口：send / followup / steer / inject / cancel / whenIdle
  - 6.2 Inbox 与 wakeup：输入如何到达驱动
  - 6.3 turn 与 step 边界：一次模型请求 + 它调用的工具
  - 6.4 `agent/*` 事件族：pre-step / request / turn-stopping 的拦截语义
  - 6.5 取消与错误恢复：`agent/request-error` 与重试

### 第四部分　与模型对话：LLM 抽象层

- **第 7 章　LLM 流式抽象：Message / ContentBlock / StreamChunk**
  - 7.1 内容块（content blocks）：text / reasoning / image / tool-call / tool-result
  - 7.2 模型请求组装与 `request/header` 日志
  - 7.3 Adapter 契约：`LlmAdapter` 与多 Provider 注册（DeepSeek / pi-ai）
  - 7.4 `BlockAssembler` 与重试：llm-retry 与 token-meter

### 第五部分　让 Agent 动手：工具与能力

- **第 8 章　工具系统：ToolDefinition 与守卫管线**
  - 8.1 `ToolDefinition`：schema + execute + 纯投影三件套
  - 8.2 JSON-value schema DSL：一套词汇描述参数与输出
  - 8.3 六段守卫管线：pre-execute → guards → execute → post-execute → finalizeContent → tools/result
  - 8.4 并行工具调用：`isConcurrencySafe` 与有界滚动池
  - 8.5 UI 呈现：presentCall / presentResult 的纯函数设计

- **第 9 章　Capability Seams：能力接缝三件套**
  - 9.1 Service Definition / Service Provider / Consumer：一个完整 seam
  - 9.2 为什么换 Provider 就换掉整个产品（FS / subprocess / sandbox 联动）
  - 9.3 Scope 与 scoped registration：per-agent 可见性与生命周期

- **第 10 章　文件系统与 Shell：本地能力的两大支柱**
  - 10.1 FS seam：`FsTarget` / 读写编辑 / `fs/observed` 观察状态
  - 10.2 read-before-write 策略与 `fs/*` 事件门
  - 10.3 Shell seam：`ShellExecRequest/Spec` 的显式分离
  - 10.4 Subprocess seam：全显式 spawn spec 与 `DSH_*` 环境词汇
  - 10.5 持久终端（terminal seam）与 PTY

### 第六部分　安全与可控：护栏

- **第 11 章　沙箱：进程的文件效应政策**
  - 11.1 `SandboxMode`：read-only / workspace-write / danger-full-access
  - 11.2 三平台后端：Linux bwrap / Landlock、macOS Seatbelt、Windows ACL
  - 11.3 Enforcement：`full` vs `partial` 的诚实报告
  - 11.4 沙箱消费者：bash-sandbox / pwsh-sandbox 如何包裹 argv
  - 11.5 远程执行：E2B POC 与 capability seam 的换骨

- **第 12 章　审批与交互：在动手前踩刹车**
  - 12.1 Approval seam：`ApprovalOutcome` 四值与 fail-closed
  - 12.2 Per-session policy：ask / never
  - 12.3 User-questions seam：工具需要人回答时
  - 12.4 Commands：不经模型的直接人机通道
  - 12.5 Permission presets 与 plan mode 的软约束

### 第七部分　记住一切：状态与持久化

- **第 13 章　持久化与崩溃恢复：日志如何变耐久**
  - 13.1 `SessionPersistence` seam：JSONL 与 SQLite 两个后端
  - 13.2 Flush checkpoint：`session/flush` 与有界写批处理
  - 13.3 崩溃恢复：孤儿 turn 的 synthetic `interrupted` 封口
  - 13.4 Spill 存储：超大工具输出溢出到文件
  - 13.5 Settings / Credentials / Workspace：其他持久化域

- **第 14 章　上下文、压缩与记忆**
  - 14.1 System-prompt assembly：prompt sections 的按序拼接
  - 14.2 上下文插件：`agent.inject()` 与 per-request context
  - 14.3 Compaction seam：`compaction/*` 事件与 tool-result pruner
  - 14.4 Skills：注册表、目录注入与 `skill` 工具
  - 14.5 Goals / Schedule / Todo：同会话目标与定时记忆

### 第八部分　多 Agent 与远程

- **第 15 章　子 Agent 与多 Agent 委派**
  - 15.1 Subagent seam：按名注册的多 Provider（in-process / fork / ACP / Codex / Claude Code）
  - 15.2 一次性 vs 可续接子 Agent：`prepareContinuable` 即能力
  - 15.3 控制工具：send_message / interrupt_agent / list_agents / report
  - 15.4 后台 job 运行时：`ctx.jobs` 与 job_* 工具

- **第 16 章　Workflow 与 Code Runtime：让模型写编排**
  - 16.1 Workflow seam：模型写脚本、脚本开子 Agent（worker_thread 引擎）
  - 16.2 Code Runtime：worker-thread 里跑模型写的程序
  - 16.3 ralph 工具与动态 workflow
  - 16.4 Hooks：Claude Code / Codex hook 桥

- **第 17 章　远程调用与可观测性：Typert RPC 与 Web UI**
  - 17.1 Typert：类型图生成器与 `@Remote` 方法
  - 17.2 API Gateway 与 Lookup/Context 声明
  - 17.3 Session query / projection / telemetry：回放与上报
  - 17.4 ACP（Agent Client Protocol）自动化服务器
  - 17.5 Web UI：`apps/web` + client modules + HMR

### 第九部分　工程化收尾

- **第 18 章　测试与可信度：怎么知道改好了**
  - 18.1 分层测试：vitest 单测、e2e、keyless snapshot
  - 18.2 运行时不变量注册表（invariants）
  - 18.3 防御模式（defensive patterns）与 fail loud 哲学
  - 18.4 从 DeepSeek Harness 借鉴：插件化 Harness 的最小清单

### 附录

- **附录 A　dsh 目录速查表**（顶层区域逐一索引）
- **附录 B　一次请求的完整事件时序**（从 followup 到 tool/result）
- **附录 C　Capability Seam 全景表**（每个 seam 的 Definition / Provider / Consumer）
- **附录 D　术语表**（Harness / Seam / Scope / Typert / Spill / ACP …）

---

## 一句话画像

> **DeepSeek Harness** —— 激进插件化型："没有特权核心，一切皆插件，连主循环都能换"。用 Cordis 的 typed events + reversible effects 把整个产品织成一张可替换的插件树，再用 profile/bundle/patch 分层把它从空列表组合出来。

| 维度 | DeepSeek Harness |
|---|---|
| 语言/栈 | TypeScript + vendored Cordis（pnpm monorepo，54 个包） |
| 核心哲学 | 一切皆插件、无特权核心、Model-visible ⟺ logged |
| 最值得读 | Session 事件日志、六段工具守卫管线、Capability Seam 三件套 |
| 状态 | 开发者预览（0.1.0-rc.5，破坏性变更随时可能） |

## 配套说明

- 本书引用的代码路径均来自 DeepSeek Harness 公开仓库（`v0.1.0-rc.5`，2026-08-13 附近），建议边读边对照源码。
- dsh 自带极高质量的 `docs/`（architecture.md + subsystems/ 逐子系统参考 + 生成的 config/tool/persistence catalog），本书是它的"中文导读+工程视角解读"，技术论断全部经源码核实。
- 欢迎通过 Issue / PR 参与勘误与共建。

## License

本书内容采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可；涉及的 DeepSeek Harness 代码版权归其原作者所有。
