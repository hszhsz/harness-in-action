# 第 11 章　沙箱：进程的文件效应政策

> Agent 最危险的能力是执行代码。dsh 的沙箱设计有一个独特的视角：**沙箱不是"隔离环境"，而是一个"文件效应政策"**。它只管文件效应（SandboxMode），网络与进程可见性在词汇之外。这一章拆开 `packages/sandbox`。

## 11.1　`SandboxMode`：read-only / workspace-write / danger-full-access

`ctx.sandbox` 把同世界子进程的 argv 包裹进一个文件效应政策，而不把消费者耦合到平台 runner。[[docs/subsystems/sandbox.md]](https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/subsystems/sandbox.md)

```ts
type SandboxMode = 'read-only' | 'workspace-write' | 'danger-full-access'
```

- **`read-only`**：拒绝写。POSIX runner 额外授予 shell 需要的 `/dev/null` sink；Windows ACL runner 不授予显式可写根，报告其环境 ACL 间隙的部分执行。
- **`workspace-write`**：允许在工作区根和后端承诺的 temp 区域下写。
- **`danger-full-access`**：绕过约束。

**只有前两种模式可以发给 provider**。`danger-full-access` 的消费者直接 spawn 原始 argv，根本不调用 `ctx.sandbox`（"A `danger-full-access` consumer spawns its original argv and does not call `ctx.sandbox`"）。

## 11.2　三平台后端：Linux bwrap / Landlock、macOS Seatbelt、Windows ACL

`dsh-sandbox-local` 提供三个平台后端：

- **Linux**：bwrap（bubblewrap，用户命名空间沙箱）+ Landlock（内核 LSM）。Landlock 还有一个专门的 node addon：`native/landlock-run`（`@deepseek-ai/node-addon-landlock-run`，"source of record"）。
- **macOS**：Seatbelt（sandbox-exec profile）。
- **Windows**：ACL restricted-token 后端。

`dsh-bash-sandbox` 和 `dsh-pwsh-sandbox` 是消费者——它们包裹 bash/pwsh 的 argv 后 spawn。

## 11.3　Enforcement：`full` vs `partial` 的诚实报告

dsh 沙箱最诚实的设计：**强制是报告的事实，不是假设**。

```ts
type SandboxEnforcement = 'full' | 'partial'
```

- `full`：后端治理模式承诺的每个文件效应；
- `partial`：活跃后端或旧内核 ABI 只治理子集——**要求绝对边界的调用者不能把它当 `full`**（"callers requiring an absolute boundary must not treat it as `full`"）。

当前 partial 的案例：旧 Landlock ABI、Windows ACL runner 的 Everyone/hard-link 边界。**"partial 就是 partial"**——沙箱不会假装比实际更安全。

## 11.4　沙箱消费者：bash-sandbox / pwsh-sandbox 如何包裹 argv

`SandboxExecutionPolicy` 是每次能力调用解析并携带的完整执行政策（含 `danger-full-access`，所以消费者可以先解析政策再决定是否绕过）：

```ts
interface SandboxExecutionPolicy {
  mode: SandboxMode
  workspaceRoot: string
  sessionId?: SessionId
}
```

`workspaceRoot` 从调用 session 的不可变 cwd 派生；根用文件系统语义规范化（先 canonicalize 再 lexical normalize，所以 `symlink/..` 的 cwd 指向 spawn 进程实际运行的目录）。`ctx.sandboxPolicy.resolve()` 接受活跃 session 和（对已批准重试的）显式 mode——服务拥有优先级和根回退，bash 和 fs 不用重复实现。

只有受约束的执行才到达 `ctx.sandbox`；provider 政策收窄 mode 但保留同一根。这允许并发 session、消费者和一次性升级重试向同一 provider 请求不同边界而不改变 provider 状态。

## 11.5　远程执行：E2B POC 与 capability seam 的换骨

第 9.2 节讲过：FS 和 subprocess provider 共享同一个执行世界，把它们指向远程沙箱，Bash/PTY/LSP 全部跟着走。`e2b/` 组就是 POC：`e2b`（sandbox）、`fs-e2b`（远程 FS）、`subprocess-e2b`（远程 subprocess）。

**容器、microVM 和远程执行是"完整能力 seam 的兄弟实现"，不是 `ctx.sandbox` 的 provider**（"Containers, microVMs, and remote execution are sibling implementations of whole capability seams, not providers of `ctx.sandbox`"）。这个区分很重要：dsh 不把"隔离"降维成"一个包裹函数"，而是让每个执行域（本地/容器/远程）都成为完整的 seam 组合。

## 本章小结

dsh 的沙箱是一个"文件效应政策 + 诚实报告"系统：

1. **只管文件效应**：SandboxMode 三值，网络与进程可见性不在词汇内；
2. **三平台后端**：bwrap/Landlock、Seatbelt、Windows ACL，各有 native 支持；
3. **诚实报告**：`full` vs `partial`，绝不假装比实际安全；
4. **消费者解析政策**：`SandboxExecutionPolicy` 每次调用携带，`danger-full-access` 消费者根本不进 `ctx.sandbox`；
5. **远程是完整 seam**：E2B 是兄弟实现，不是沙箱 provider 的变体。

## 动手实验

1. 读 `packages/sandbox/sandbox/src/index.ts` 的 `SandboxMode` 与 `SandboxEnforcement`，确认三值语义。
2. 读 `packages/shell/bash-sandbox`，看它如何把 `ConfinedArgv` 交给沙箱后端。
3. 读 `native/landlock-run` 的 README，理解 Landlock node addon 的用途与限制。
4. 读 `packages/e2b/`，对照第 9 章的 seam 三件套，确认 E2B 实现了哪些角色的哪些契约。
