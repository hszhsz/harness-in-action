# 第 15 章　子 Agent 与多 Agent 委派

dsh 的子 Agent 体系是它最丰富的 seam 之一：**按名注册的多 Provider**（in-process / fork / ACP / Codex / Claude Code / dsh-sdk），一次性 vs 可续接两种形态，外加控制工具与后台 job 运行时。这一章拆开 `packages/subagent`。读完后，你应该能理解"能力即方法"的 TypeScript narrowing 设计，以及 `ctx.jobs` 如何让 bash/subagent/PTY 共用一套生命周期。

## Subagent seam：按名注册的多 Provider

`ctx.subagents` 与 bash 不同：**多个 provider 实现共存于一个上下文，按名注册**。`packages/subagent/subagent/src/types.ts` 定义了 Provider 契约。

> It differs from the other capability seams because **multiple provider implementations coexist** in one context, registered by name (`ctx.subagents`), while bash allows only one executor.

Provider 全家：
- `subagent-spawn-in-process`（同进程子 Agent）
- `subagent-fork-in-process`（fork 后端）
- `subagent-acp`（ACP 自动化协议）
- `subagent-codex`（Codex）
- `subagent-claude-code`（Claude Code）
- `subagent-dsh-sdk`（dsh SDK）

Consumer 全家：`tool-subagent`（按 provider 的委派工具）、`tool-subagent-control`（全局 `send_message`/`interrupt_agent`/`list_agents`）、`tool-subagent-report`（child-scoped `report` 返回通道）。

## 启动时能力检查：`SubagentCapabilities`

provider 在一个静态描述符上声明**启动时**特性，服务在一次性 run 存在前检查。`packages/subagent/subagent/src/types.ts` 定义了：

```ts
export interface SubagentCapabilities {
  readonly outputSchema: boolean
  readonly depthLimit: boolean
  readonly toolFilter: boolean
  readonly persona: boolean
}
```

需要 provider 没有的能力的请求被大声拒绝（`SubagentError('UNSUPPORTED_CAPABILITY')`）——"never accepted-then-ignored"。源码注释说："a request that needs a capability the chosen provider lacks is rejected with a typed error rather than accepted-then-ignored (the 'fail loud, no silent degradation' rule)"。

`SubagentStartRequest` 带必填 `parent`（提供 session cwd、lineage、delegation depth）：

```ts
export interface SubagentRunInfo {
  readonly runId: SubagentRunId
  readonly provider: string
  readonly id: SessionId
  readonly local: boolean
}
```

## 一次性 vs 可续接子 Agent：`prepareContinuable` 即能力

子 Agent 分两种形态：

- **一次性（one-shot）**：`start()` 路径，provider 组合 child。
- **可续接（continuable）**：由 continuation manager 自己组合，**gated by 一个可选方法，方法存在即能力**（"gated by one optional method whose presence IS the capability, with TS narrowing as the discovery mechanism"）——`SubagentProvider.prepareContinuable`。

`tool-subagent` 的默认配置很讲究：`subagent` 是 `continuable`，默认省略调用进后台并自动结算投递；`subagent_fork` 保持 `one-shot`，默认前台——"the two shipped schemas are not identical"。

## 控制工具：send_message / interrupt_agent / list_agents / report

- **`send_message`** 与 **`interrupt_agent`**：全局注册一次。
- **`list_agents`**：从 `/list-agents` 插件单独加载，用 sessionProjections 和 live Agent 注册表。
- **`report`**：per continuable in-process child 注册（不是全局）——所以这个 schema 只在这样的 child 内部可见，并且**能穿过全局 toolFilter** 存活。

## 后台 job 运行时：`ctx.jobs` 与 job_* 工具

`ctx.jobs` 是通用长任务运行时。`packages/subagent` 的 job 模块定义了：

```ts
interface JobKindMap {
  bash: 'bash'
  subagent: 'subagent'
}
```

`JobKind` 从 merge-extensible map 派生，注册表把 kind 当不透明 id 命名空间。`JobStatus` = `'running' | 'stopping' | 'completed' | 'killed' | 'failed'`。

**Producer 契约**：`JobStart` 声明身份和 starter。运行时在调用 `run()` 前完成 preflight（access + cleanup），之后不再有可失败的步骤——"The runtime finishes preflight before calling `run()` and commits without a later failable step"。

**`JobId` 生成是 `<kind>-N`**；访问控制依赖 owner 授权而非 id 保密（"Access control relies on owner authorization, not id secrecy"）。

`job_kill`/`job_list`/`job_output` 是 kind-agnostic 控制器：后台 bash、PTY sends、subagents 用同一套三个工具读/列/杀。后台完成通知通过 `user/message via agent.inject()` 投递。

## 架构速览

```
子 Agent 体系
┌─────────────────────────────────────────────┐
│  Subagent seam (ctx.subagents)              │
│  多 Provider 按名注册, 共存于一个上下文         │
│  ┌─────────────────────────────────────┐    │
│  │ in-process │ fork │ ACP │ Codex     │    │
│  │ Claude Code │ dsh-sdk              │    │
│  └─────────────────────────────────────┘    │
├─────────────────────────────────────────────┤
│  能力检查 (启动时)                            │
│  SubagentCapabilities: 4 个 boolean flag     │
│  缺能力 → UNSUPPORTED_CAPABILITY (大声拒绝)   │
├─────────────────────────────────────────────┤
│  一次性 vs 可续接                             │
│  start() → 一次性                            │
│  prepareContinuable → 可续接 (方法存在即能力)  │
├─────────────────────────────────────────────┤
│  控制工具                                     │
│  send_message / interrupt_agent (全局)        │
│  list_agents (独立加载)                       │
│  report (child-scoped, 穿过 toolFilter)      │
├─────────────────────────────────────────────┤
│  ctx.jobs (后台 job 运行时)                   │
│  JobKind: bash / subagent (merge-extensible) │
│  job_kill / job_list / job_output            │
│  完成通知 via agent.inject()                  │
└─────────────────────────────────────────────┘
```

## 关键设计决策

1. **多 Provider 按名注册**：与 bash 的单 executor 不同，subagent 允许多个 provider 共存。这让"本地子 agent"和"委托给 Claude Code"可以在同一个上下文中并存，由 tool schema 的参数选择 provider。

2. **能力即方法**：`prepareContinuable` 的存在本身就是能力声明。TypeScript 的类型 narrowing 让服务可以自动发现 provider 是否支持可续接，不需要额外的 flag。

3. **启动时大声拒绝**：`SubagentCapabilities` 在 start 之前检查，缺少能力就拒绝。这比"接受后忽略"更安全——不会产生一个功能不完整的子 agent。

4. **`ctx.jobs` 统一后台任务**：bash、PTY、subagent 共用一套生命周期（running → stopping → completed/killed/failed）和控制工具（kill/list/output）。这让"后台任务管理"不需要为每种任务类型写不同的 UI。

## 本章小结

- 多 Provider 共存：按名注册，in-process / fork / ACP / Codex / Claude Code / dsh-sdk 一个接口。
- 能力即方法：`prepareContinuable` 存在即可续接，TS narrowing 做发现。
- 启动时大声拒绝：`SubagentCapabilities` 前置检查，绝不 accept-then-ignored。
- 控制工具分层：委派工具 per-provider，控制工具全局，report 工具 child-scoped。
- 后台 job 统一：`ctx.jobs` 让 bash/subagent/PTY 共用一套生命周期与控制器。

## 阅读思考题

1. `prepareContinuable` 的存在本身就是能力声明。如果一个 provider 后来移除了这个方法（比如重构），已有的可续接子 agent 会怎样？
2. `ctx.jobs` 的 `JobId` 是 `<kind>-N` 格式（比如 `bash-1`、`subagent-3`）。这种生成方式在什么场景下可能产生冲突？为什么"访问控制依赖 owner 授权而非 id 保密"？
3. `report` 工具是 child-scoped 的，能穿过全局 toolFilter。如果一个恶意子 agent 通过 `report` 工具泄露了敏感信息，谁来负责？这种"穿透"设计是安全风险还是必要功能？

> **下一章预告**：第 16 章将讲 dsh 最"未来感"的两个能力——Workflow（模型写编排脚本）与 Code Runtime（worker-thread 里跑模型写的程序）。
