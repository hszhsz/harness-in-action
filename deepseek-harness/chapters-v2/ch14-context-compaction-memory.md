# 第 14 章　上下文、压缩与记忆

模型窗口有限，Agent 的上下文必须被精心组装、压缩和遗忘。这一章拆开 dsh 的上下文体系：system-prompt assembly、`agent.inject()`、compaction seam、skills、goals/schedule/todo。读完后，你应该能理解 prompt sections 的 order 约定、compaction 如何通过 log-only 事件工作、以及 skills 的 host+per-scope 分层注册表。

## System-prompt assembly：prompt sections 的按序拼接

`ctx.systemPrompt` 拥有 prompt-section 与 tool-schema 组装。`packages/core/system-prompt` 定义了 `PromptSection` 接口和组装逻辑。

**Sections 按升序拼接**。约定：
- `-100`：harness identity opener
- `-99`：`harness:source`（`addHarnessSourceSection` 注册，告诉 agent dsh 实现 checkout 在哪、警告不要从路径推断 cwd）
- `0`：deployment persona
- `50`：plan policy
- `99`：Code Mode 的 collapse statement（`COLLAPSE_SECTION_ORDER`）
- `100-199`：per-tool guidance band

`AssembleContext` 是 merge-extensible：`dsh-agent` 加可选 live `agent` 字段，`assembleContextFor(agent, signal)` 一起设置显式字段。`ToolProviderResult` 提供模型可见 schema 集 + 预限制名字宇宙（`knownNames`——区分"配置名字打错了"和"故意隐藏的已知工具"）。

**协作组装**（cooperative assembly）：sections 的贡献者通过 waterfall 协作，一个有效的 `complete` section 成为唯一 prompt section。

## `agent.inject()` 与 per-request context

第 6.1 节讲过 `agent.inject()`：排队模型可见上下文给下一个 pre-step，**不唤醒**。注入的上下文会成为 `user/message` 事件（`source` 区分：file-change notices、subdir AGENTS.md、skill content、cron notifications、goal continuation round）。源码注释说："All three project their `content` verbatim; `source` tells them apart"。

`packages/context` 组还有 request-context 插件，提供更细粒度的上下文注入。

## Compaction seam：`compaction/*` 事件与 tool-result pruner

compaction 是另一个 capability seam：`packages/compaction/compaction/src/index.ts` 定义了 `CompactionEngine` 抽象服务：

```ts
export abstract class CompactionEngine extends Service {
  constructor(ctx: Context) {
    super(ctx, 'compaction')
  }
  // ...
}
```

compaction 通过声明合并给 `SessionEventMap` 加 `compaction/start` / `compaction/summary` / `compaction/end`——**不是 `SurfaceEventType`**（无 `surfaceOp`）。这意味着 compaction 事件不参与表面事件的投影，只在日志里记录压缩的发生。

Provider：
- `compaction-basic`：基本摘要压缩
- `tool-result-pruner`：剪枝工具结果（spill 化的大输出）

Consumer：`command-compact` 是 `/compact` 命令消费者。

`CompactionTrigger` 定义了触发原因：

```ts
export type CompactionTrigger = 'pressure' | 'context-overflow'
```

`ManualCompactionError` 是手动压缩的失败分类：

```ts
export class ManualCompactionError extends Error {
  constructor(
    readonly code: ManualCompactionErrorCode,  // 'busy' | 'cancelled' | 'changed' | 'summary' | 'commit' | 'persistence'
    message: string,
    options?: ErrorOptions,
  ) { super(message, options) }
}
```

## Skills：注册表、目录注入与 `skill` 工具

skill 能力族（`packages/skill`）：Definition（`dsh-skill`，`ctx.skills`）+ Provider（`skill-filesystem` 本地、`skill-badge` 打包徽章）+ Consumer（`tool-skill`）。

**注册表是 host+per-scope 分层的**（沿用 tools registry 在 `dsh-scope` 上建立的形状）：host 行和 repo 插件进全局层，agent preset 挂载的插件进该 preset 的层——provider 名**每层唯一**而不是进程级唯一。读合并全局层 + 查看 scope 的链，最近层的条目在重名时直接胜出。

**Skills 是可选指令，不是 session 事件**——词汇在 skill 页而不是 core.md。这意味着 skill 内容不会自动进入 session 日志，而是在需要时通过 `agent.inject()` 注入。

## Goals / Schedule / Todo：同会话目标与定时记忆

**Goals**（`goal/`）：持久化的同会话目标域。
- `GoalId` branded，`GoalRef`（id + revision）CAS 变更。
- `GoalPhase` = active / paused / blocked / complete。
- blocked 是唯一的"被问题卡住"持久状态，带稳定 lower-kebab-case `code`（机器路由）+ 人类可读 `message`。
- 工具 `create_goal`/`get_goal`/`update_goal`：create/edit/pause/resume 要求直接人类 root authority；complete/blocked 也接受当前目标轮次；默认 blocked 下界是 3 个已承认轮次。

**Schedule**（`schedule/`）：session-local 提醒。
- `schedule_create`/`schedule_delete`/`schedule_list`
- v1 接受 after_seconds、显式绝对 at、有界固定速率 every_seconds
- 披露 session-local delivery
- 只在 opt-in Schedule 插件加载后创建的 live root Agent scopes 内注册

**Todo**（`todo/`）：`todo_write` 是 session-owned 状态；UI 渲染最新的 `todo/write` 事件为 checklist。`allowParallelInProgress` 无默认必须显式声明——"so the catalog states its choice: true, whose description invites several in_progress items"。

## 架构速览

```
上下文与记忆体系
┌─────────────────────────────────────────────┐
│  System-prompt assembly                     │
│  sections 按 order 升序拼接                  │
│  -100 identity → 0 persona → 50 plan → ... │
├─────────────────────────────────────────────┤
│  agent.inject()                             │
│  不唤醒, 排队模型可见上下文                    │
│  source 区分: file-change / AGENTS.md / ... │
├─────────────────────────────────────────────┤
│  Compaction seam                            │
│  Definition: compaction/compaction           │
│  Provider: compaction-basic / pruner         │
│  Consumer: command-compact                   │
│  事件: compaction/start/summary/end (log-only)│
├─────────────────────────────────────────────┤
│  Skills                                     │
│  host+per-scope 分层注册表                    │
│  可选指令, 不是 session 事件                   │
├─────────────────────────────────────────────┤
│  Goals / Schedule / Todo                    │
│  Goals: 持久化 CAS 状态机                     │
│  Schedule: session-local 提醒                │
│  Todo: log-only 状态                         │
└─────────────────────────────────────────────┘
```

## 关键设计决策

1. **sections 按升序拼接**：order 数值越小越靠前。这让 identity（-100）始终在最前面，persona（0）紧随其后，per-tool guidance（100-199）在最后。每个插件只需要知道自己的 order 值，不需要关心其他插件的存在。

2. **Skills 是可选指令，不是 session 事件**：skill 内容不自动进入日志，只在需要时通过 `agent.inject()` 注入。这让 skill 内容不会污染 session 日志，也不会被 `deriveMessages()` 自动投影到模型历史。

3. **compaction 事件是 log-only**：`compaction/start/summary/end` 不是 `SurfaceEventType`，不参与表面事件的投影。这让压缩的发生被记录在日志里（可审计），但不影响模型看到的历史。

4. **Goals 的 blocked 下界是 3 个已承认轮次**：一个目标不能在被承认之前就从 blocked 转移到其他状态。这防止了模型过早地认为问题已解决。

## 本章小结

- System-prompt assembly 按 order 升序拼接 sections，`-100` identity → `0` persona → `50` plan policy。
- `agent.inject()` 不唤醒地排队模型可见上下文，落进下一个被承认的请求。
- compaction seam 加 log-only 事件，basic 摘要 + tool-result pruner 剪枝。
- Skills 是 host+per-scope 分层注册表，可选指令不是事件。
- Goals 持久化 CAS 状态机、Schedule session-local 提醒、Todo log-only 状态。

## 阅读思考题

1. prompt sections 的 order 是编译时常量。如果两个插件注册了相同 order 的 section，会发生什么？dsh 有什么机制防止 order 冲突？
2. compaction 的 `compaction/summary` 事件记录了压缩后的摘要。如果压缩丢失了关键信息（比如一个重要的工具调用结果），模型还能恢复它吗？
3. Goals 的 `blocked` 状态带一个 `code` 字段用于机器路由。如果一个 goal 的 `code` 在不同版本间变了（比如从 `rate-limited` 变成 `throttled`），会破坏什么？

> **下一章预告**：第 15 章将拆开 dsh 的子 Agent 体系——按名注册的多 Provider、一次性 vs 可续接两种形态、以及后台 job 运行时。
