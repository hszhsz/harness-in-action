# 第 9 章　Capability Seams：能力接缝三件套

dsh 最值得抄的设计：**Capability Seam（能力接缝）**。一个可替换能力由三个角色构成——Service Definition（服务定义）、Service Provider（服务提供者）、Consumer（消费者，通常是模型面对的工具）。这一章讲透这个模式，以及它如何让"换一个 Provider 就换掉整个产品"成为现实。读完后，你应该能在源码里找到 seam 的三个角色，并理解为什么"只有一个角色不算 seam"。

## Service Definition / Service Provider / Consumer：一个完整 seam

`docs/architecture.md` 的定义：

> A **seam** is a swappable capability with three roles: a **Service Definition** declaring the interface, a **Service Provider** implementing it, and a **Consumer** using it, commonly a model-facing tool. A package may combine roles, but one role alone is not a seam; adding a capability means designing all three.

翻译：一个 seam 是**三个角色**的完整设计。一个包可以兼多个角色，但**只有一个角色不算 seam**——加一个能力意味着把三个角色一起设计。

`AGENTS.md` 把它设为红线：

> **A capability seam comprises Service Definition / Service Provider / Consumer roles.** It is complete, never one role; split only when roles evolve independently.

## Seam 全景表

从 `docs/architecture.md` + `docs/subsystems/README.md` 整理的完整 seam 表：

| Seam | Service Definition | Service Provider(s) | Consumer(s) |
|---|---|---|---|
| LLM | `llm/llm`（`ctx.llm`） | `llm-deepseek`、`llm-pi-ai` | 主循环 |
| Filesystem | `fs/fs`（`ctx.fs`） | `fs-local`、`fs-e2b` | `tool-fs`、`tool-str-replace-editor`、`tool-fs-search` |
| Shell | `shell/shell`（`ctx.shell`） | `bash-local`、`pwsh-local` | `tool-bash`、`tool-pwsh`、`tool-bash-persistent` |
| Subprocess | `subprocess/subprocess`（`ctx.subprocess`） | `subprocess-local` | 工具、shell、LSP |
| Sandbox | `sandbox/sandbox`（`ctx.sandbox`） | `sandbox-local`（bwrap/Landlock/Seatbelt/ACL） | `bash-sandbox`、`pwsh-sandbox` |
| Code Runtime | `code-runtime/code-runtime`（`ctx.codeRuntime`） | `code-runtime-worker-thread` | `run_code` |
| Subagent | `subagent/subagent`（`ctx.subagents`） | in-process / fork / ACP / Codex / Claude Code / dsh-sdk | `tool-subagent`、`tool-subagent-control`、`tool-subagent-report` |
| Workflow | `workflow/workflow`（`ctx.workflowEngine`） | `workflow-worker-thread` | `tool-workflow`、`tool-ralph` |
| Web | `web/web`（`ctx.web`） | web search/fetch providers | `tool-web` |
| Spill | `spill/spill`（`ctx.spillStore`） | `spill-local` | `spill-policy` |
| Skill | `skill/skill`（`ctx.skills`） | `skill-filesystem`、`skill-badge` | `tool-skill` |
| Terminal | `terminal/terminal`（`ctx.terminals`） | 各 PTY 后端 | `tool-terminal`、`tool-bash-persistent` |
| Compaction | `compaction/compaction` | `compaction-basic`、`tool-result-pruner` | 主循环上下文 |
| LSP | `lsp/lsp`（`ctx.lsp`） | `lsp-stdio` | `tool-lsp` |
| Persistence | `session/session-persistence` | JSONL / SQLite | 主循环、session-query |

## 为什么换 Provider 就换掉整个产品

`docs/architecture.md` 给出了最惊艳的例子：

> Seams are why one provider swap changes the whole product. Filesystem and subprocess providers share one execution world, so pointing them at a remote sandbox moves Bash, PTY, and LSP with them, with no provider forks.

**Filesystem 和 subprocess provider 共享同一个执行世界**——把它们指向远程沙箱，Bash、PTY、LSP 全部跟着走，不需要任何 provider 分叉。`packages/e2b/` 就是 POC：`fs-e2b`、`subprocess-e2b` 是 E2B 的 sandbox + FS/subprocess adapter。一个产品，本地跑是一套后端，远程跑是另一套后端，**模型看到的工具 schema 完全不变**。

Subagent 同样：provider "vary just as widely behind one interface, from a fresh child agent to a delegated turn in another product"——从全新子 Agent 到"委托给另一个产品的 turn"（ACP/Codex/Claude Code 后端）都藏在同一个接口后面。

## Scope 与 scoped registration：per-agent 可见性与生命周期

Capability seam 的"作用域"维度由 `packages/core/scope` 提供。`packages/core/scope/src/store.ts` 定义了核心原语：

```ts
export interface ScopeLayer {
  isEmpty(): boolean   // 层完全为空，可回收
}
```

`ScopedLayers<L>` 拥有 eager 全局层 + 惰性创建的精确 scope 层。**读不创建层**（`peek(undefined)` 表示无覆盖），`merge()` 物化"插入序全局命名条目 + scoped shadows"。

`NamedEntries<V>` 提供插入序查找与 live 迭代：

```ts
export class NamedEntries<V> implements EntryValues<V> {
  private data = new Map<string, V>()

  insert(name: string, value: V): () => void {
    const data = this.data
    if (data.has(name)) throw this.duplicateError(name)
    data.set(name, value)
    let active = true
    return () => {
      if (!active) return
      active = false
      data.delete(name)
      if (data.size === 0 && this.data === data) this.data = new Map()
    }
  }
  // ...
}
```

`insert` 返回一个 idempotent undo——注册即 effect，卸载时回卷。注意当表清空时，`this.data = new Map()` 释放旧 Map 的引用——这是内存友好的设计。

`AnonymousEntries<V>` 给每个 append 唯一身份，相等值也保持独立——用于不需要命名的注册（比如 prompt sections）。

**这对产品意味着什么**：工具注册表、skill 注册表、prompt sections 都支持"per-agent 可见性"——一个 agent preset 挂载的插件只影响那个 agent 的组成，不污染全局。

## 架构速览

```
Capability Seam 三件套
┌─────────────────────────────────────────────┐
│  Service Definition (接口声明)                │
│  例: fs/fs → ctx.fs                         │
│  声明: FsTarget, read/write/edit 接口        │
├─────────────────────────────────────────────┤
│  Service Provider (实现)                      │
│  例: fs-local (本地FS), fs-e2b (远程FS)      │
│  实现: 同一接口, 不同后端                      │
├─────────────────────────────────────────────┤
│  Consumer (使用, 通常是模型工具)               │
│  例: tool-fs (edit/read/write/read_image)    │
│  使用: ctx.fs 的接口, 不关心 Provider         │
└─────────────────────────────────────────────┘

换 Provider → 换整个产品 (本地 ↔ 远程)
模型看到的 schema 不变
```

## 关键设计决策

1. **三角色完整设计**：只有一角不算 seam。加一个能力意味着把 Definition / Provider / Consumer 一起设计。这避免了"先写工具，再补接口"的倒置设计。

2. **共享执行世界**：FS 和 subprocess provider 同源，换远程沙箱全部跟着走。这是"换骨不换面"的极致——产品形态完全由 Provider 决定，Consumer 不需要任何改动。

3. **scope 原语的惰性创建**：`ScopedLayers` 的读不创建层，只有写才创建。这让"没有 per-agent 覆盖"的场景零开销。

4. **`NamedEntries.insert` 返回 idempotent undo**：注册即 effect，卸载时精确回卷。这与 Cordis 的 `ctx.effect()` 机制一致——所有注册都可逆。

## 本章小结

- Capability Seam 三件套是 dsh 对"可替换性"的完整回答：Definition / Provider / Consumer 一起设计。
- 换 Provider 换掉整个产品（本地 ↔ 远程沙箱、子 Agent 跨产品委托），模型看到的 schema 不变。
- `scope` 原语让注册可以 per-agent 可见，插件树可以局部重写。
- 所有注册都是 effect，卸载自动回卷。

## 阅读思考题

1. `docs/architecture.md` 说"one role alone is not a seam"。如果一个包只实现了 Service Definition 和 Consumer（没有独立的 Provider），它算 seam 吗？在什么场景下这种"内置 Provider"是合理的？
2. `NamedEntries.insert` 返回的 undo 是 idempotent 的——调用两次只删一次。如果一个插件意外调用了两次 undo，会发生什么？为什么 idempotent 是必要的？
3. `ScopedLayers` 的读不创建层。如果一个 agent preset 没有覆盖任何工具，它的 scope 层在内存中存在吗？这种"零开销"设计在什么场景下可能失效？

> **下一章预告**：第 10 章将拆开本地能力的两大支柱——文件系统 seam 和 Shell seam——以及它们如何共享同一个执行世界。
