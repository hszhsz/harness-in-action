# 第 9 章　Capability Seams：能力接缝三件套

> dsh 最值得抄的设计：**Capability Seam（能力接缝）**。一个可替换能力由三个角色构成——Service Definition（服务定义）、Service Provider（服务提供者）、Consumer（消费者，通常是模型面对的工具）。这一章讲透这个模式，以及它如何让"换一个 Provider 就换掉整个产品"成为现实。

## 9.1　Service Definition / Service Provider / Consumer：一个完整 seam

`docs/architecture.md` 的定义：

> A **seam** is a swappable capability with three roles: a **Service Definition** declaring the interface, a **Service Provider** implementing it, and a **Consumer** using it, commonly a model-facing tool. A package may combine roles, but one role alone is not a seam; adding a capability means designing all three.

翻译：一个 seam 是**三个角色**的完整设计。一个包可以兼多个角色，但**只有一个角色不算 seam**——加一个能力意味着把三个角色一起设计。

dsh 的 `AGENTS.md` 把它设为红线：

> **A capability seam comprises Service Definition / Service Provider / Consumer roles.** It is complete, never one role; split only when roles evolve independently.

**Seam 全景表**（从 `docs/architecture.md` + `docs/subsystems/README.md` 整理）：

| Seam | Service Definition | Service Provider(s) | Consumer(s) |
|---|---|---|---|
| LLM | `llm/llm`（`ctx.llm`） | `llm-deepseek`、`llm-pi-ai` | 主循环 |
| Filesystem | `fs/fs`（`ctx.fs`） | `fs-local`、`fs-e2b` | `tool-fs`、`tool-str-replace-editor`、`tool-fs-search` |
| Shell | `shell/shell`（`ctx.shell`） | `bash-local`、`pwsh-local` | `tool-bash`、`tool-pwsh`、`tool-bash-persistent` |
| Subprocess | `subprocess/subprocess`（`ctx.subprocess`） | `subprocess-local`（进程树） | 工具、shell、LSP |
| Sandbox | `sandbox/sandbox`（`ctx.sandbox`） | `sandbox-local`（bwrap/Landlock/Seatbelt/ACL） | `bash-sandbox`、`pwsh-sandbox` |
| Code Runtime | `code-runtime/code-runtime`（`ctx.codeRuntime`） | `code-runtime-worker-thread` | `run_code`（Code Mode） |
| Subagent | `subagent/subagent`（`ctx.subagents`） | in-process / fork / ACP / Codex / Claude Code / dsh-sdk | `tool-subagent`、`tool-subagent-control`、`tool-subagent-report` |
| Workflow | `workflow/workflow`（`ctx.workflowEngine`） | `workflow-worker-thread` | `tool-workflow`、`tool-ralph` |
| Web | `web/web`（`ctx.web`） | web search/fetch providers | `tool-web`（`web_search`/`web_fetch`） |
| Spill | `spill/spill`（`ctx.spillStore`） | `spill-local` | `spill-policy`（`tools/post-execute` 政策） |
| Skill | `skill/skill`（`ctx.skills`） | `skill-filesystem`、`skill-badge` | `tool-skill` |
| Terminal | `terminal/terminal`（`ctx.terminals`） | 各 PTY 后端 | `tool-terminal`、`tool-bash-persistent` |
| Compaction | `compaction/compaction` | `compaction-basic`、`tool-result-pruner` | 主循环上下文 |
| LSP | `lsp/lsp`（`ctx.lsp`） | `lsp-stdio` | `tool-lsp` |
| Persistence | `session/session-persistence` | JSONL / SQLite | 主循环、session-query |

## 9.2　为什么换 Provider 就换掉整个产品

`docs/architecture.md` 给出了最惊艳的例子：

> Seams are why one provider swap changes the whole product. Filesystem and subprocess providers share one execution world, so pointing them at a remote sandbox moves Bash, PTY, and LSP with them, with no provider forks.

**Filesystem 和 subprocess provider 共享同一个执行世界**——把它们指向远程沙箱，Bash、PTY、LSP 全部跟着走，不需要任何 provider 分叉。这就是 `e2b/` 包在做的 POC：`fs-e2b`、`subprocess-e2b` 是 E2B 的 sandbox + FS/subprocess adapter。一个产品，本地跑是一套后端，远程跑是另一套后端，**模型看到的工具 schema 完全不变**。

Subagent 同样：`docs/subsystems/subagent.md` 说 provider "vary just as widely behind one interface, from a fresh child agent to a delegated turn in another product"——从全新子 Agent 到"委托给另一个产品的 turn"（ACP/Codex/Claude Code 后端）都藏在同一个接口后面。

## 9.3　Scope 与 scoped registration：per-agent 可见性与生命周期

Capability seam 的"作用域"维度由 `packages/core/scope` 提供。核心原语（`docs/subsystems/scope.md`）：

```ts
type ScopeKey = object                          // 不透明对象身份
type Scoped<T extends object> = object & { readonly [ScopedBrand]: T }  // 编译期 brand
```

`ScopeKey` 是不透明对象身份——发货的 loop 用 live `Agent` 对象本身当 key，但原语从不 inspect 这个对象。`scopeTarget(base, key)` 返回带 brand 的路由接收器。

**`ScopeLayer` 与 `ScopedLayers`** 是作用域注册表的核心：

```ts
interface ScopeLayer {
  isEmpty(): boolean   // 层完全为空，可回收
}
```

`ScopedLayers<L>` 拥有 eager 全局层 + 惰性创建的精确 scope 层。**读不创建层**（`peek(undefined)` 表示无覆盖），`merge()` 物化"插入序全局命名条目 + scoped shadows"。注册用同一个上下文做可见性和 Cordis effect 所有权，收集一个同步 undo，返回 Cordis 的精确 disposer；只有完整 `ScopeLayer` 为空时才回收 scoped 层。

`NamedEntries<V>` 提供插入序查找与 live 迭代（调用者拥有的重复错误）；`AnonymousEntries<V>` 给每个 append 唯一身份，相等值也保持独立。

**这对产品意味着什么**：工具注册表、skill 注册表、prompt sections 都支持"per-agent 可见性"——一个 agent preset 挂载的插件只影响那个 agent 的组成，不污染全局。`docs/subsystems/tools.md` 提到 `dsh-scope` 的形状被 `dsh-skills` 直接沿用（"the shape the tools registry established over dsh-scope"）。这是"一切皆插件"的最后一环：**插件不但可替换，还可以按 agent 作用域隔离**。

## 本章小结

Capability Seam 三件套是 dsh 对"可替换性"的完整回答：

1. **三角色完整**：Definition 声明接口、Provider 实现、Consumer 使用——只有一角不算 seam；
2. **换骨不换面**：换 Provider 换掉整个产品（本地 ↔ 远程沙箱、子 Agent 跨产品委托），模型看到的 schema 不变；
3. **作用域隔离**：`scope` 原语让注册可以 per-agent 可见，插件树可以局部重写；
4. **可逆**：所有注册都是 effect，卸载自动回卷。

## 动手实验

1. 读 `packages/e2b/` 的 `fs-e2b` 和 `subprocess-e2b`，理解 E2B 如何实现与 `fs-local` 相同的 seam 契约。
2. 在 `docs/subsystems/subagent.md` 里读 `SubagentCapabilities` 的四个 flag（outputSchema / depthLimit / toolFilter / persona），理解"启动时能力检查、拒绝要大声"。
3. 读 `packages/core/scope/src/store.ts` 的 `ScopedLayers`，画出全局层 + scope 层的合并顺序。
4. 找一个同时扮演多角色的包（比如 `spill-policy` 既是 Consumer 又是政策），分析它为什么这样设计。
