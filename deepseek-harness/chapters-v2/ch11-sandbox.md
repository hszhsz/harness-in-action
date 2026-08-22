# 第 11 章　沙箱：进程的文件效应政策

Agent 最危险的能力是执行代码。dsh 的沙箱设计有一个独特的视角：**沙箱不是"隔离环境"，而是一个"文件效应政策"**。它只管文件效应（`SandboxMode`），网络与进程可见性在词汇之外。这一章拆开 `packages/sandbox/sandbox/src/index.ts`。

## `SandboxMode`：read-only / workspace-write / danger-full-access

`ctx.sandbox` 把同世界子进程的 argv 包裹进一个文件效应政策，而不把消费者耦合到平台 runner。`packages/sandbox/sandbox/src/index.ts` 定义了核心类型：

```ts
export type SandboxMode = 'read-only' | 'workspace-write' | 'danger-full-access'
```

- **`read-only`**：拒绝写。POSIX runner 额外授予 shell 需要的 `/dev/null` sink；Windows ACL runner 不授予显式可写根。
- **`workspace-write`**：允许在工作区根和后端承诺的 temp 区域下写。
- **`danger-full-access`**：绕过约束。

**只有前两种模式可以发给 provider**。源码注释说得很清楚："A `danger-full-access` consumer spawns its original argv and does not call `ctx.sandbox`"——`danger-full-access` 的消费者直接 spawn 原始 argv，根本不调用 `ctx.sandbox`。

## `SandboxExecutionPolicy`：每次调用携带的完整政策

```ts
export interface SandboxExecutionPolicy {
  mode: SandboxMode
  workspaceRoot: string
  sessionId?: SessionId
}
```

`workspaceRoot` 从调用 session 的不可变 cwd 派生。`sessionId` 是 branded 的 `SessionId`——后端用它做 per-session 状态管理（比如 Windows ACL 给每个 session/workspace 对分配随机私有 temp 目录和 SID）。

源码注释："What one confined execution is allowed to touch — carried PER CALL, not fixed on the provider"。政策是每次调用携带的，不是固定在 provider 上的——两个消费者可以同时请求不同边界（bash 用 `read-only`，子 agent 用 `workspace-write`），而不改变 provider 状态。

## 三平台后端：Linux bwrap / Landlock、macOS Seatbelt、Windows ACL

`dsh-sandbox-local` 提供三个平台后端：

- **Linux**：bwrap（bubblewrap，用户命名空间沙箱）+ Landlock（内核 LSM）。Landlock 还有一个专门的 node addon：`native/landlock-run`（`@deepseek-ai/node-addon-landlock-run`）。
- **macOS**：Seatbelt（sandbox-exec profile）。
- **Windows**：ACL restricted-token 后端。

`dsh-bash-sandbox` 和 `dsh-pwsh-sandbox` 是消费者——它们包裹 bash/pwsh 的 argv 后 spawn。

## Enforcement：`full` vs `partial` 的诚实报告

dsh 沙箱最诚实的设计：

```ts
export type SandboxEnforcement = 'full' | 'partial'
```

- `full`：后端治理模式承诺的每个文件效应；
- `partial`：活跃后端或旧内核 ABI 只治理子集——**要求绝对边界的调用者不能把它当 `full`**。

源码注释："callers requiring an absolute boundary must not treat it as `full`"。当前 partial 的案例：旧 Landlock ABI、Windows ACL runner 的 Everyone/hard-link 边界。**"partial 就是 partial"**——沙箱不会假装比实际更安全。

## 沙箱的隔离哲学

`docs/subsystems/sandbox.md` 明确区分了沙箱和容器/VM：

> Containers, microVMs, and remote execution are sibling implementations of whole capability seams, not providers of `ctx.sandbox`.

**容器、microVM 和远程执行是"完整能力 seam 的兄弟实现"，不是 `ctx.sandbox` 的 provider**。这个区分很重要：dsh 不把"隔离"降维成"一个包裹函数"，而是让每个执行域（本地/容器/远程）都成为完整的 seam 组合。`packages/e2b/` 就是远程执行的完整 seam 实现。

## 架构速览

```
沙箱体系
┌─────────────────────────────────────────────┐
│  SandboxMode (文件效应政策)                   │
│  read-only → workspace-write → danger-full  │
├─────────────────────────────────────────────┤
│  SandboxExecutionPolicy (每次调用携带)        │
│  mode + workspaceRoot + sessionId           │
├─────────────────────────────────────────────┤
│  三平台后端                                   │
│  Linux: bwrap + Landlock                    │
│  macOS: Seatbelt                            │
│  Windows: ACL                               │
├─────────────────────────────────────────────┤
│  SandboxEnforcement (诚实报告)                │
│  full → 完全治理                             │
│  partial → 部分治理, 不假装更安全              │
├─────────────────────────────────────────────┤
│  消费者: bash-sandbox / pwsh-sandbox         │
│  包裹 argv → 交给后端                        │
│  danger-full-access → 不调用 ctx.sandbox     │
└─────────────────────────────────────────────┘
```

## 关键设计决策

1. **只管文件效应**：`SandboxMode` 三值，网络与进程可见性不在词汇内。这让沙箱的关注点极度聚焦——"文件效应"是一个可精确描述、可精确验证的约束。

2. **`danger-full-access` 消费者不调用 `ctx.sandbox`**：绕过约束的方式是"不进入沙箱系统"，而不是"在沙箱系统里请求全权限"。这避免了沙箱后端需要处理"全权限"这个特殊情况。

3. **诚实报告 `full` vs `partial`**：沙箱不会假装比实际更安全。这是一个工程诚信问题——`partial` 意味着"某些文件效应可能逃逸"，调用者需要自己决定是否接受这个风险。

4. **远程是完整 seam，不是沙箱 provider 变体**：E2B 是 FS + subprocess + sandbox 三个 seam 的完整实现，不是 `ctx.sandbox` 的一个 provider。这让远程执行和本地执行在架构上是对等的。

## 本章小结

- dsh 的沙箱是一个"文件效应政策 + 诚实报告"系统。
- `SandboxMode` 三值：read-only / workspace-write / danger-full-access。
- 三平台后端：bwrap/Landlock、Seatbelt、Windows ACL。
- 诚实报告：`full` vs `partial`，绝不假装比实际安全。
- 远程是完整 seam：E2B 是兄弟实现，不是沙箱 provider 的变体。

## 阅读思考题

1. `SandboxMode` 只管文件效应，网络与进程可见性不在词汇内。如果一个被沙箱限制的进程通过网络发送了敏感数据，`ctx.sandbox` 能阻止吗？这种"只管文件"的设计是局限还是有意为之？
2. `SandboxEnforcement = 'partial'` 意味着"某些文件效应可能逃逸"。在什么场景下你会拒绝接受 `partial` enforcement？在什么场景下 `partial` 已经足够？
3. dsh 把容器/VM/远程执行定位为"完整能力 seam 的兄弟实现"，而不是沙箱 provider 的变体。如果把远程执行实现为 `ctx.sandbox` 的一个 provider，会引入什么架构问题？

> **下一章预告**：第 12 章将拆开 dsh 的审批体系——在 Agent 动手之前的四层"刹车"。
