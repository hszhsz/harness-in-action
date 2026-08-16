# 第 16 章　Workflow 与 Code Runtime：让模型写编排

> 这一章讲 dsh 最"未来感"的两个能力：Workflow（模型写一个编排脚本、脚本开子 Agent）与 Code Runtime（worker-thread 里跑模型写的程序）。外加 ralph 工具与 Claude Code / Codex hook 桥。

## 16.1　Workflow seam：模型写脚本、脚本开子 Agent

`ctx.workflowEngine` 让 Agent 运行一个**模型写的编排脚本**，脚本启动子 Agent。像 bash 一样，它**每个上下文只允许一个引擎实现**——没有命名 provider 注册表（"a second engine replaces the first through plugin configuration rather than running beside it"）。[[docs/subsystems/workflow.md]](https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/subsystems/workflow.md)

```ts
interface WorkflowStartRequest {
  script: string                 // plain-JS 脚本体（顶层 await 可用，以 return <json-value> 结尾）
  meta: WorkflowMeta             // 身份块，纯 JSON 数据（引擎先 shape 校验）
  args?: unknown                 // 可选输入，脚本里作为 args 全局变量原样可见
  subagentProvider?: string      // 可选 run 级 child-provider 覆盖
  maxTotalAgents?: number        // 可选每 run 总 child 上限
  parent: Agent                  // 必填：脚本启动的每个 child 都归因给它
  signal?: AbortSignal
}
```

**`meta` 和 `args` 是纯 JSON 数据**——引擎先按 schema 校验 `meta` 并大声拒绝，然后才跑任何东西（"no script text is ever evaluated to obtain it"）。**`parent` 必填**：每个 child 都归因给这个 live Agent，cwd、lineage、depth 穿过 subagent seam。

`WorkflowMeta` 的字段词汇与 **Claude Code dynamic-workflows meta block 一致**（name/description/whenToUse/phases）——dsh 直接对齐了 Anthropic 的约定。`phases` 只是进度词汇：`phase()` 调用匹配标题给观察者，不暗示执行结构。

**引擎**：`workflow-worker-thread`（`node:worker_threads`，每 run 一个 worker，脚本的 vm context 在 worker 里）。**Consumer**：`tool-workflow`（`workflow` 工具）。设计记录：`2026-07-05-dynamic-workflows.md`。

**脚本全局 API**（tool-catalog 记录）：`agent(prompt, opts?)`（跑一个 subagent 到完成；带 `opts.schema` 时返回校验过的对象）、`pipeline(items, ...stages)`（每 item 独立跑各阶段，无阶段间屏障）、`parallel(thunks)`（并发跑零参函数并等全部）、`phase(title)`、`log(message)`、`args`。

## 16.2　Code Runtime：worker-thread 里跑模型写的程序

`ctx.codeRuntime` 跑一个**模型写的程序**，绑定 host 提供的 async bindings，报告打印了什么、返回了什么。Code execution 是**一个可选能力，不是 agent-loop spine 的一部分**。[[docs/subsystems/code-runtime.md]](https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/subsystems/code-runtime.md)

```ts
interface CodeRunRequest {
  program: string                  // async function 的函数体：顶层 await 与 return 可用
  bindings: CodeBindingNamespace[] // 暴露给程序的 host 函数，每命名空间一个全局对象
  signal?: AbortSignal             // 中止：硬停（即使 mid-loop），以 kind 'abort' 失败解决
}
```

**错误是结果的字段，不是 `run()` 的 rejection**——"reporting a failed program is the caller's job, not an exception path"（与 `ShellExecutor.run` 的 resolve-on-failure 契约一致）：

```ts
interface CodeRunResult {
  value?: CodeJsonValue   // 程序完成值（顶层 return），越过 lossless-JSON 边界
  logs: string[]          // 程序按序输出的文本
  error?: CodeRunFailure  // 失败时存在；见失败分类学
}
```

Provider 是 `code-runtime-worker-thread`；Consumer 是 `run_code`（Code Mode）。**Code Mode 有两个变体**（`DSH_TOOLS_MODE=native|code|both` 环境变量 opt-in）：native 模式是普通工具，code 模式下 `run_code` 是注册表唯一的 wire 贡献，其他可见能力由加载 runtime 语言里的生成 SDK 段声明，程序通过 bindings 调用它们——子调用重入完整守卫工具管线，记 `tool/code-dispatch`，每嵌套执行链接到这个外层结果。

## 16.3　ralph 工具与动态 workflow

`tool-ralph`（`ralph` 工具）：一个固定前台 workflow，每轮启动一个新鲜的结构化 child；模型只选不可变 objective 和可选轮次上限（"The model selects only the immutable objective and an optional round cap"）。这是"重复多轮子 Agent 研究"的封装。

## 16.4　Hooks：Claude Code / Codex hook 桥

`packages/hooks`：`hook-protocol`（wire 协议库）+ `hooks-claude-code` + `hooks-codex`。它们把 Claude Code / Codex 的 hook 桥接进 dsh——log-only 的 `hook/invoked` / `hook/result` 记录（第 5.1 节提过它们不是 `SurfaceEventType`）。这让用惯了 Claude Code hooks 的团队可以把既有 hook 脚本直接接进 dsh 管线。

## 本章小结

dsh 的"模型写代码"能力族：

1. **Workflow**：模型写编排脚本开子 Agent，`meta` 先校验后执行，field 词汇对齐 Claude Code；
2. **Code Runtime**：worker-thread 里跑模型写的程序，错误是结果字段；Code Mode 下 `run_code` 是唯一 wire 贡献，子调用重入完整工具管线；
3. **ralph**：固定前台多轮结构化子 Agent 研究；
4. **Hooks**：Claude Code / Codex hook 桥，既有资产可复用。

## 动手实验

1. 读 `packages/workflow/workflow-worker-thread`，看 worker thread 里如何创建 vm context 跑脚本。
2. 读 `packages/code-runtime/code-runtime/src/types.ts` 的 `CodeRunFailure` 分类学。
3. 在 `docs/tool-catalog.md` 里读 `ralph` 与 `workflow` 的对比行，确认两者差异。
4. 读 `packages/hooks/hook-protocol`，理解 `hook/invoked`/`hook/result` 如何进 session 日志。
