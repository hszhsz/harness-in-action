# 第 12 章　审批与交互：在动手前踩刹车

Agent 在用户机器上发出的每个副作用都需要用户同意。dsh 的审批体系分四层：一次性审批（approval）、人机问答（user-questions）、直接命令（commands）、权限预设（permission-presets）。这一章拆开 `packages/interaction`。读完后，你应该能理解为什么"fail-closed"是审批系统的第一原则，以及 plan mode 如何作为"软约束"与硬约束（沙箱/审批）共存。

## Approval seam：`ApprovalOutcome` 四值与 fail-closed

`ctx.approval` 回答一个问题：**这个具体动作可以继续吗？** `packages/interaction/user-approval/src/index.ts` 定义了核心类型：

```ts
export type ApprovalOutcome = 'allowed-once' | 'rejected' | 'cancelled' | 'unavailable'
```

**封闭且 fail-closed**：
- `allowed-once`：只授予被问到的那个动作。
- `rejected`：显式拒绝。
- `cancelled`：请求撤回。
- `unavailable`：应答者缺失/非拥有/抛错/不合规——**全部 deny**。

源码注释："A missing, non-owning, throwing, or non-conforming answerer becomes `unavailable` rather than opening the gate"——应答者出任何问题，门都是关的，而不是开的。

每个请求拿到一个新鲜的 `ApprovalRequestId`（branded），把 `approval/asked` 和 `approval/decided` 审计事件配对：

```ts
declare module '@deepseek-ai/dsh-session/types' {
  interface SessionEventMap {
    'approval/asked': {
      id: ApprovalRequestId
      toolName: string
      callId?: CallId
      reason?: string
    }
    'approval/decided': {
      id: ApprovalRequestId
      outcome: ApprovalOutcome
    }
    'approval/policy': {
      policy: ApprovalPolicy
      source?: 'delegation'
    }
  }
}
```

注意 `approval/policy` 事件的 `source?: 'delegation'`——标记从父 agent 继承的审批策略覆盖。这在子 agent 委派时很重要。

## Per-session policy：ask / never

```ts
export type ApprovalPolicy = 'ask' | 'never'
```

- **`ask`**（默认）：委托给组合的应答者链，无应答默认 `unavailable`。
- **`never`**：永不询问任何人，每个 ask 确定性返回 `rejected`——"The strict headless stance (CI, unattended runs) and the policy whose outcome is knowable without asking"。

有效政策是 session 日志里最后一个 `approval/policy` 事件的值，回退到服务配置。**唯一写路径是 `setApprovalPolicy(session, policy)`**——重放可以重建覆盖。

## User-questions seam：工具需要人回答时

`ctx.userQuestions` 是工具或权限插件需要人先回答才能继续时的 provider-neutral 词汇。`packages/interaction/user-questions` 定义了：

```ts
interface AskUserQuestionOption {
  label: string          // 用户可见选项文本，也是模型可见的选中值
  description?: string   // 可选 UI 帮助文本
}
```

`AskUserQuestionIntent` 可选声明一个已知决策类型：`approve` 命名肯定选项而不是依赖选项顺序。**intent 只改呈现，不改答案字段**——认出 tag 的 UI 用特定方式呈现，但回答的仍是同样的选项 label。

消费者例子：`tool-ask-user`（`ask_user_question` 工具暂停工具调用直到活跃 UI provider 返回人类答案）、`exit_plan_mode`（呈现计划让人审阅）。

## Commands：不经模型的直接人机通道

`ctx.commands` 是人类命令注册表。交互适配器用它发现并直接执行插件拥有的命令——**不创建模型消息**（"directly execute plugin-owned commands for an exact agent without creating a model message"）。

```ts
interface CommandDefinition {
  name: string        // 小写命令名，无前导斜杠
  summary: string     // 发现 UI 用的人类可读摘要
}
```

例子：`/plan [off|message]`（plan-mode 插件注册）、`/export`（session-log-download）、`/compact`（command-compact）。命令是"绕过模型 turn 的确定性操作"——输入 hint、执行、结果都直接由适配器处理。

## Permission presets 与 plan mode 的软约束

**Permission presets**（`interaction/permission-presets`）：`PresetSpec`/`PresetOption` + 派生的 `custom` 状态 + log-only `permission/preset` 事件。预设让部署把"审批策略组合"打包成可切换的模式。

**Plan mode**（`packages/plan/plan-mode`）是**软约束**：

> Plan mode is **soft guidance**. [Sandbox mode] and [approval policy] enforce restrictions independently; neither reads or writes plan state.

`plan/mode` 是 log-only 的整值替换 session 事件（`{ active: boolean }`）——持久、可重放、永不在模型 transcript 里。激活时，部署拥有的 guidance section 作为 `plan:policy` prompt section（order 50）包含进每个模型请求。

`exit_plan_mode` 工具在非 plan 模式下也保持注册（进出 plan mode 只变 prompt section，不变工具目录），但执行时非 plan 模式会拒绝。退出需要完整 markdown 计划（`#` 开头）+ user-questions 审阅。

## 架构速览

```
审批与交互四层体系
┌─────────────────────────────────────────────┐
│ 1. Approval (一次性审批)                      │
│    ctx.approval → ApprovalOutcome (4值)       │
│    fail-closed: 缺应答者 = unavailable        │
│    per-session policy: ask / never            │
├─────────────────────────────────────────────┤
│ 2. User-questions (人机问答)                  │
│    ctx.userQuestions → AskUserQuestionOption  │
│    intent 只改呈现, 不改答案字段                │
├─────────────────────────────────────────────┤
│ 3. Commands (直接命令)                        │
│    ctx.commands → CommandDefinition           │
│    不经模型, 确定性操作                        │
├─────────────────────────────────────────────┤
│ 4. Presets + Plan mode (预设 + 软约束)        │
│    permission-presets: 审批组合预设             │
│    plan-mode: 软约束指导 (不读写沙箱/审批)      │
└─────────────────────────────────────────────┘
```

## 关键设计决策

1. **fail-closed 是第一原则**：应答者出任何问题（缺失、非拥有、抛错、不合规），门都是关的。`unavailable` 和 `rejected` 的效果相同——deny。这让审批系统在任何异常情况下都是安全的。

2. **intent 只改呈现，不改答案字段**：`AskUserQuestionIntent` 的 `approve` 标签让 UI 可以用特定方式呈现"批准"选项，但回答的仍是同样的选项 label。这让同一套答案逻辑在不同 UI 下都能工作。

3. **plan mode 是软约束**：sandbox mode 和 approval policy 独立强制，不读写 plan state。plan mode 只通过 prompt section 引导模型行为，不通过硬机制阻止。这让 plan mode 可以在不破坏现有安全机制的情况下提供额外指导。

4. **`approval/policy` 事件支持 delegation source**：子 agent 继承父 agent 的审批策略时，用 `source: 'delegation'` 标记。这让重放能精确区分"用户设置的策略"和"委派继承的策略"。

## 本章小结

- dsh 的"刹车"体系分四层，各司其职。
- **Approval**：一次性动作审批，四值封闭、fail-closed，per-session 政策 ask/never。
- **User-questions**：工具需要人回答的 provider-neutral 词汇，intent 只改呈现。
- **Commands**：不经模型的确定性人机通道。
- **Presets + Plan mode**：审批组合预设 + 软约束指导，硬约束独立强制。

## 阅读思考题

1. `ApprovalOutcome` 的 `unavailable` 和 `rejected` 效果相同（deny），但它们的语义不同。在什么场景下区分这两者是有价值的？比如审计日志、错误诊断、还是 UI 呈现？
2. `exit_plan_mode` 工具在非 plan 模式下保持注册但执行时拒绝。为什么不直接注销这个工具？这种"注册但拒绝"的设计在什么场景下比"注销"更好？
3. plan mode 是"软约束"——它只通过 prompt section 引导模型，不通过硬机制阻止。如果模型无视 plan mode 的指导直接执行了副作用，谁来负责？这种设计是缺陷还是有意为之？

> **下一章预告**：第 13 章将讲 dsh 的持久化与崩溃恢复——日志如何变耐久，以及孤儿 turn 的 synthetic `interrupted` 封口。
