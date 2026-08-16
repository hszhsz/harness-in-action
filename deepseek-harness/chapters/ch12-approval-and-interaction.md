# 第 12 章　审批与交互：在动手前踩刹车

> Agent 在用户机器上发出的每个副作用都需要用户同意。dsh 的审批体系分四层：一次性审批（approval）、人机问答（user-questions）、直接命令（commands）、权限预设（permission-presets）。这一章拆开 `packages/interaction`。

## 12.1　Approval seam：`ApprovalOutcome` 四值与 fail-closed

`ctx.approval` 回答一个问题：**这个具体动作可以继续吗？**（"may this specific action proceed?"）。[[docs/subsystems/approval.md]](https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/subsystems/approval.md)

```ts
type ApprovalOutcome = 'allowed-once' | 'rejected' | 'cancelled' | 'unavailable'
```

**封闭且 fail-closed**：`allowed-once` 只授予被问到的那个动作；`rejected`（显式拒绝）、`cancelled`（请求撤回）、`unavailable`（应答者缺失/非拥有/抛错/不合规）全部 deny。**"A missing, non-owning, throwing, or non-conforming answerer becomes `unavailable` rather than opening the gate"**——应答者出任何问题，门都是关的，而不是开的。

每个请求拿到一个新鲜的 `ApprovalRequestId`（branded），把 `approval/asked` 和 `approval/decided` 审计事件配对，同时避免审批 id 与工具调用 id / agent / session id 混用。

## 12.2　Per-session policy：ask / never

```ts
type ApprovalPolicy = 'ask' | 'never'
```

- **`ask`**（默认）：委托给组合的应答者链，无应答默认 `unavailable`；
- **`never`**：永不询问任何人，每个 ask 确定性返回 `rejected`——"The strict headless stance (CI, unattended runs) and the policy whose outcome is knowable without asking"。

有效政策是 session 日志里最后一个 `approval/policy` 事件的值，回退到服务配置。**唯一写路径是 `setApprovalPolicy(session, policy)`**——重放可以重建覆盖。两个政策都把它们的完整当前含义贡献给 cache-safe runtime-context 快照；改变审批状态会追加一个新的完整快照，而不是重写请求头的系统提示词。

## 12.3　User-questions seam：工具需要人回答时

`ctx.userQuestions` 是工具或权限插件需要人先回答才能继续时的 provider-neutral 词汇。[[docs/subsystems/user-questions.md]](https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/subsystems/user-questions.md)

```ts
interface AskUserQuestionOption {
  label: string          // 用户可见选项文本，也是模型可见的选中值
  description?: string   // 可选 UI 帮助文本
}
```

`AskUserQuestionIntent` 可选声明一个已知决策类型（tagged on `kind`）：`approve` 命名肯定选项而不是依赖选项顺序。**intent 只改呈现，不改答案字段**——认出 tag 的 UI 用特定方式呈现，但回答的仍是同样的选项 label。`ask()` 拒绝两个类型无法承载的断言：`approve` 命名了非本问题选项、无 detail 的问题带 intent。

消费者例子：`tool-ask-user`（`ask_user_question` 工具暂停工具调用直到活跃 UI provider 返回人类答案）、`exit_plan_mode`（呈现计划让人审阅）。

## 12.4　Commands：不经模型的直接人机通道

`ctx.commands` 是人类命令注册表。交互适配器用它发现并直接执行插件拥有的命令——**不创建模型消息**（"directly execute plugin-owned commands for an exact agent without creating a model message"）。[[docs/subsystems/commands.md]](https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/subsystems/commands.md)

```ts
interface CommandDefinition {
  name: string        // 小写命令名，无前导斜杠
  summary: string     // 发现 UI 用的人类可读摘要
}
```

例子：`/plan [off|message]`（plan-mode 插件注册）、`/export`（session-log-download）、`/compact`（command-compact）。命令是"绕过模型 turn 的确定性操作"——输入 hint、执行、结果都直接由适配器处理。

## 12.5　Permission presets 与 plan mode 的软约束

**Permission presets**（`interaction/permission-presets`）：`PresetSpec`/`PresetOption` + 派生的 `custom` 状态 + log-only `permission/preset` 事件。预设让部署把"审批策略组合"打包成可切换的模式。

**Plan mode**（`packages/plan/plan-mode`）是**软约束**：

> Plan mode is **soft guidance**. [Sandbox mode] and [approval policy] enforce restrictions independently; neither reads or writes plan state.

`plan/mode` 是 log-only 的整值替换 session 事件（`{ active: boolean }`）——持久、可重放、永不在模型 transcript 里。激活时，部署拥有的 guidance section 作为 `plan:policy` prompt section（order 50）包含进每个模型请求。`exit_plan_mode` 工具在非 plan 模式下也保持注册（进出 plan mode 只变 prompt section，不变工具目录），但执行时非 plan 模式会拒绝。退出需要完整 markdown 计划（`#` 开头）+ user-questions 审阅；批准记录一个 silent pending exit，在下一个被接受的 in-turn pre-step 追加。

## 本章小结

dsh 的"刹车"体系分四层，各司其职：

1. **Approval**：一次性动作审批，四值封闭、fail-closed，per-session 政策 ask/never；
2. **User-questions**：工具需要人回答的 provider-neutral 词汇，intent 只改呈现；
3. **Commands**：不经模型的确定性人机通道（`/plan`、`/export`、`/compact`）；
4. **Presets + Plan mode**：审批组合预设 + 软约束指导，硬约束（沙箱/审批）独立强制。

## 动手实验

1. 读 `packages/interaction/user-approval/src/index.ts` 的 `ApprovalOutcome` 与 `ApprovalPolicy`，确认 fail-closed 的所有路径。
2. 读 `packages/plan/plan-mode` 的 `plan/mode` 事件处理，理解"pending selection 在下一个被接受的 in-turn pre-step 追加"。
3. 在 `docs/tool-catalog.md` 里找 `ask_user_question` 与 `exit_plan_mode`，对照它们的执行路径。
4. 读 `packages/interaction/commands` 的 `/plan` 注册逻辑（`ctx.commands` 消费者），理解命令如何通过 `agent.steer()` 提交消息。
