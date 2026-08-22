# 第 16 章　Workflow 与 Code Runtime：让模型写编排

这一章讲 dsh 最"未来感"的两个能力：Workflow（模型写一个编排脚本、脚本开子 Agent）与 Code Runtime（worker-thread 里跑模型写的程序）。外加 ralph 工具与 Claude Code / Codex hook 桥。读完后，你应该能理解 `WorkflowMeta` 如何对齐 Claude Code 的约定、Code Mode 的"collapse"设计、以及 hooks 如何复用既有资产。

## Workflow seam：模型写脚本、脚本开子 Agent

`ctx.workflowEngine` 让 Agent 运行一个**模型写的编排脚本**，脚本启动子 Agent。像 bash 一样，它**每个上下文只允许一个引擎实现**——没有命名 provider 注册表（"a second engine replaces the first through plugin configuration rather than running beside it"）。

`packages/workflow/workflow/src/types.ts` 定义了核心类型：

```ts
export interface WorkflowMeta {
  name: string
  description: string
  whenToUse?: string
  phases?: WorkflowPhase[]
}

export interface WorkflowPhase {
  title: string
  detail?: string
  provider?: string
  model?: string
}

export type WorkflowStopReason = 'completed' | 'cancelled' | 'error'

export interface WorkflowResult {
  value: unknown
  stopReason: WorkflowStopReason
  error?: string
}
```

**`WorkflowMeta` 的字段词汇与 Claude Code dynamic-workflows meta block 一致**（name/description/whenToUse/phases）——dsh 直接对齐了 Anthropic 的约定。`phases` 只是进度词汇：`phase()` 调用匹配标题给观察者，不暗示执行结构。

**`WorkflowRunId`** 是 branded 的：

```ts
export type WorkflowRunId = Branded<'WorkflowRunId'>

export function WorkflowRunId(id: string): WorkflowRunId {
  return id as WorkflowRunId
}
```

**脚本全局 API**：`agent(prompt, opts?)`（跑一个 subagent 到完成；带 `opts.schema` 时返回校验过的对象）、`pipeline(items, ...stages)`（每 item 独立跑各阶段，无阶段间屏障）、`parallel(thunks)`（并发跑零参函数并等全部）、`phase(title)`、`log(message)`、`args`。

**引擎**：`workflow-worker-thread`（`node:worker_threads`，每 run 一个 worker，脚本的 vm context 在 worker 里）。**Consumer**：`tool-workflow`（`workflow` 工具）。

## Code Runtime：worker-thread 里跑模型写的程序

`ctx.codeRuntime` 跑一个**模型写的程序**，绑定 host 提供的 async bindings，报告打印了什么、返回了什么。Code execution 是**一个可选能力，不是 agent-loop spine 的一部分**。

`packages/code-runtime/code-runtime/src/types.ts` 定义了：

```ts
export interface CodeRunRequest {
  program: string                  // async function 的函数体
  bindings: CodeBindingNamespace[] // 暴露给程序的 host 函数
  signal?: AbortSignal             // 中止：硬停
}

export interface CodeRunResult {
  value?: CodeJsonValue   // 程序完成值
  logs: string[]          // 程序按序输出的文本
  error?: CodeRunFailure  // 失败时存在
}
```

**错误是结果的字段，不是 `run()` 的 rejection**——"reporting a failed program is the caller's job, not an exception path"。这与 `ShellExecutor.run` 的 resolve-on-failure 契约一致。

Provider 是 `code-runtime-worker-thread`；Consumer 是 `run_code`（Code Mode）。**Code Mode 有两个变体**（`DSH_TOOLS_MODE=native|code|both`）：native 模式是普通工具，code 模式下 `run_code` 是注册表唯一的 wire 贡献。

第 8 章讲过 `CODE_ONLY_INSTRUCTION`：

```ts
const CODE_ONLY_INSTRUCTION = `\`${RUN_CODE_NAME}\` is the only tool you can call directly — a tool call naming any other tool fails. Reach every tool the SDK declares below from inside the program.`
```

程序通过 bindings 调用能力——子调用重入完整守卫工具管线，记 `tool/code-dispatch`，每嵌套执行链接到这个外层结果。

## ralph 工具与动态 workflow

`tool-ralph`（`ralph` 工具）：一个固定前台 workflow，每轮启动一个新鲜的结构化 child；模型只选不可变 objective 和可选轮次上限。这是"重复多轮子 Agent 研究"的封装。

## Hooks：Claude Code / Codex hook 桥

`packages/hooks`：`hook-protocol`（wire 协议库）+ `hooks-claude-code` + `hooks-codex`。它们把 Claude Code / Codex 的 hook 桥接进 dsh——log-only 的 `hook/invoked` / `hook/result` 记录。这让用惯了 Claude Code hooks 的团队可以把既有 hook 脚本直接接进 dsh 管线。

## 架构速览

```
Workflow 与 Code Runtime
┌─────────────────────────────────────────────┐
│  Workflow seam (ctx.workflowEngine)         │
│  模型写编排脚本 → 脚本开子 Agent              │
│  WorkflowMeta 对齐 Claude Code 约定          │
│  全局 API: agent/pipeline/parallel/phase    │
│  引擎: worker-thread (每 run 一个 worker)    │
│  Consumer: tool-workflow                    │
├─────────────────────────────────────────────┤
│  Code Runtime seam (ctx.codeRuntime)        │
│  模型写程序 → worker-thread 里跑              │
│  错误是结果字段, 不是 rejection               │
│  Code Mode: run_code 是唯一 wire 贡献        │
│  子调用重入完整工具管线                        │
├─────────────────────────────────────────────┤
│  ralph (tool-ralph)                         │
│  固定前台 workflow, 多轮结构化子 Agent 研究     │
├─────────────────────────────────────────────┤
│  Hooks (packages/hooks)                     │
│  hook-protocol + hooks-claude-code          │
│  + hooks-codex                              │
│  桥接既有 hook 脚本进 dsh 管线                 │
└─────────────────────────────────────────────┘
```

## 关键设计决策

1. **`WorkflowMeta` 对齐 Claude Code 约定**：字段词汇（name/description/whenToUse/phases）直接复用 Anthropic 的 dynamic-workflows meta block。这让从 Claude Code 迁移到 dsh 的团队可以复用已有的 workflow 定义。

2. **错误是结果字段，不是 rejection**：`CodeRunResult.error` 而不是 `run()` 抛错。这让调用者可以统一处理成功和失败——"reporting a failed program is the caller's job, not an exception path"。

3. **Code Mode 的"collapse"设计**：`run_code` 是唯一注册的工具，其他能力通过 SDK 段在脚本内部暴露。这让模型看到的工具列表极度精简，但脚本内部可以调用所有能力。

4. **子调用重入完整工具管线**：Code Mode 里脚本通过 bindings 调用工具时，每个子调用都经过完整的六段守卫管线。这让安全策略在"模型写代码"的场景下依然生效。

## 本章小结

- Workflow：模型写编排脚本开子 Agent，`meta` 先校验后执行，field 词汇对齐 Claude Code。
- Code Runtime：worker-thread 里跑模型写的程序，错误是结果字段；Code Mode 下 `run_code` 是唯一 wire 贡献。
- ralph：固定前台多轮结构化子 Agent 研究。
- Hooks：Claude Code / Codex hook 桥，既有资产可复用。

## 阅读思考题

1. `WorkflowMeta` 对齐了 Claude Code 的约定。如果 Claude Code 更新了它的 meta block 格式（比如加了新字段），dsh 需要做什么来保持对齐？
2. Code Mode 下 `run_code` 是唯一注册的工具。如果模型在脚本里尝试调用一个不存在的 SDK 函数，会发生什么？错误信息对模型友好吗？
3. 子调用重入完整工具管线意味着每个嵌套调用都经过 approval 和 sandbox。如果一个脚本发起了 100 次子调用，用户会被审批 100 次吗？dsh 有什么机制优化这种体验？

> **下一章预告**：第 17 章将拆开 dsh 的远程调用体系——Typert 类型图生成器 + `@Remote` 方法 + API Gateway + Web UI。
