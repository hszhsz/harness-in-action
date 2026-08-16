# 第 15 章　子 Agent 与多 Agent 委派

> dsh 的子 Agent 体系是它最丰富的 seam 之一：**按名注册的多 Provider**（in-process / fork / ACP / Codex / Claude Code / dsh-sdk），一次性 vs 可续接两种形态，外加控制工具与后台 job 运行时。这一章拆开 `packages/subagent`。

## 15.1　Subagent seam：按名注册的多 Provider

`ctx.subagents` 与 bash 不同：**多个 provider 实现共存于一个上下文，按名注册**。[[docs/subsystems/subagent.md]](https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/subsystems/subagent.md)

> It differs from the other capability seams because **multiple provider implementations coexist** in one context, registered by name (`ctx.subagents`), while bash allows only one executor.

Provider 全家：
- `subagent-spawn-in-process`（同进程子 Agent）
- `subagent-fork-in-process`（fork 后端）
- `subagent-acp`（ACP 自动化协议）
- `subagent-codex`（Codex）
- `subagent-claude-code`（Claude Code）
- `subagent-dsh-sdk`（dsh SDK）

Consumer 全家：`tool-subagent`（按 provider 的委派工具）、`tool-subagent-control`（全局 `send_message`/`interrupt_agent`/`list_agents`）、`tool-subagent-report`（child-scoped `report` 返回通道）。

**启动时能力检查**：provider 在一个静态描述符上声明**启动时**特性，服务在一次性 run 存在前检查：

```ts
interface SubagentCapabilities {
  readonly outputSchema: boolean
  readonly depthLimit: boolean
  readonly toolFilter: boolean
  readonly persona: boolean
}
```

需要 provider 没有的能力的请求被大声拒绝（`SubagentError('UNSUPPORTED_CAPABILITY')`）——"never accepted-then-ignored"。

## 15.2　一次性 vs 可续接子 Agent：`prepareContinuable` 即能力

子 Agent 分两种形态：

- **一次性（one-shot）**：`start()` 路径，provider 组合 child；
- **可续接（continuable）**：由 continuation manager 自己组合，**gated by 一个可选方法，方法存在即能力**（"gated by one optional method whose presence IS the capability, with TS narrowing as the discovery mechanism"）——`SubagentProvider.prepareContinuable`。

`SubagentStartRequest` 带必填 `parent`（提供 session cwd、lineage、delegation depth）；可选 output schema、depth、tool filter、persona 需要匹配的能力 flag。不支持 schema 在 start 时失败；in-process 后端把 filter 和 persona scope 到 child 创建，并用强制 capture 工具实现受支持的对象根 schema。

**tool-subagent 的默认配置很讲究**（`packages/bundle/base/cordis.patch.yml` + `examples/acp-agent/cordis.yml`）：`subagent` 是 `continuable`，默认省略调用进后台并自动结算投递；`subagent_fork` 保持 `one-shot`，默认前台——"the two shipped schemas are not identical"。

## 15.3　控制工具：send_message / interrupt_agent / list_agents / report

- **`send_message`** 与 **`interrupt_agent`**：全局注册一次（provider-bound 的 tool-subagent 实例注册不同的委派工具，而这个包注册全局控制工具）；
- **`list_agents`**：从 `/list-agents` 插件单独加载，用 sessionProjections 和 live Agent 注册表；
- **`report`**：per continuable in-process child 注册（不是全局）——所以这个 schema 只在这样的 child 内部可见，并且**能穿过全局 toolFilter** 存活（"survives its global toolFilter"）。

## 15.4　后台 job 运行时：`ctx.jobs` 与 job_* 工具

`ctx.jobs` 是通用长任务运行时。[[docs/subsystems/jobs.md]](https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/subsystems/jobs.md)

```ts
interface JobKindMap {
  bash: 'bash'
  subagent: 'subagent'
}
```

`JobKind` 从 merge-extensible map 派生，注册表把 kind 当不透明 id 命名空间。`JobStatus` = `'running' | 'stopping' | 'completed' | 'killed' | 'failed'`；producer 特有事实放 `JobSnapshot.detail`。

**Producer 契约**：`JobStart` 声明身份和 starter。运行时在调用 `run()` 前完成 preflight（access + cleanup），之后不再有可失败的步骤——"The runtime finishes preflight before calling `run()` and commits without a later failable step. Producers own execution resources; the runtime owns identity, access, and lifecycle state."

**`JobId` 生成是 `<kind>-N`**；访问控制依赖 owner 授权而非 id 保密（"Access control relies on owner authorization, not id secrecy"）。

`job_kill`/`job_list`/`job_output` 是 kind-agnostic 控制器：后台 bash、PTY sends、subagents 用同一套三个工具读/列/杀。加载插件会附加控制器，武装生产者的 `ctx.jobs.start()`。后台完成通知通过 `user/message via agent.inject()` 投递。

## 本章小结

dsh 的多 Agent 体系：

1. **多 Provider 共存**：按名注册，in-process / fork / ACP / Codex / Claude Code / dsh-sdk 一个接口；
2. **能力即方法**：`prepareContinuable` 存在即可续接，TS narrowing 做发现；
3. **启动时大声拒绝**：`SubagentCapabilities` 前置检查，绝不 accept-then-ignored；
4. **控制工具分层**：委派工具 per-provider，控制工具全局，report 工具 child-scoped；
5. **后台 job 统一**：`ctx.jobs` 让 bash/subagent/PTY 共用一套生命周期与控制器。

## 动手实验

1. 读 `packages/subagent/subagent/src/types.ts` 的 `SubagentStartRequest` 与 `SubagentCapabilities`，画出 start 前的校验流。
2. 读 `packages/subagent/subagent-acp`，看 ACP provider 如何"委托给另一个产品"。
3. 读 `packages/subagent/tool-subagent-control`，确认 `send_message`/`interrupt_agent` 的全局注册与 `list_agents` 的独立加载。
4. 在 `docs/tool-catalog.md` 里读 `subagent` vs `subagent_fork` 的对比行，确认两个 shipped schema 的差异。
