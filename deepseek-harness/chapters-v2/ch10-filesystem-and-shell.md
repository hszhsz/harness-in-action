# 第 10 章　文件系统与 Shell：本地能力的两大支柱

一个编码 Agent 的两只手：读写文件（FS seam）和执行命令（Shell + Subprocess seam）。这一章拆开 dsh 的三个能力 seam——filesystem、shell、subprocess——以及它们如何共享同一个"执行世界"。读完后，你应该能理解 read-before-write 策略如何通过事件门实现、`resolve(request): Spec` 的显式默认设计、以及 `DSH_*` 环境变量词汇的作用。

## FS seam：`FsTarget` / 读写编辑 / `fs/observed` 观察状态

`ctx.fs` 是文件系统能力 seam 的服务定义（`packages/fs/fs`）。它提供 `FsTarget`（路径目标）、读/写/编辑结果、observed-file 状态、`FsErrorCode`。

消费者是：
- `tool-fs`（`edit`/`read`/`read_image`/`write` 四个工具）
- `tool-str-replace-editor`（独立的 view/create/unique literal replace/line insert 工具）
- `tool-fs-search`（`glob`/`grep`，spawn 打包的 `@vscode/ripgrep`）

一个有趣的设计：`read_image` 只有在 `ctx.attachments` 存在时才注册——schema 与路由无关，执行时拒绝除非确切路由的模型声明支持图像输入。这是"能力与条件分离"的例子：工具的 schema 始终注册，但执行时根据运行时条件决定是否放行。

## read-before-write 策略与 `fs/*` 事件门

dsh 的 read-before-write/edit 政策由 `@deepseek-ai/dsh-fs-observation-policy` 提供——一个 **`fs/*` 事件门插件**。

机制：tool-fs 的突变发出 `fs/write-intent` 或 `fs/edit-intent` 事件；观察政策监听这些事件，检查该文件是否已经被 read 过（`fs/observed` 状态）；没读过就不放行。

关键：**政策挂在事件上，不挂在工具上**——工具不 import 政策服务，政策也不改工具 schema。这正是第 8 章管线哲学的延伸：工具只管能力，政策通过事件门插入。

## Shell seam：`ShellExecRequest/Spec` 的显式分离

Shell 能力 seam 的服务定义在 `shell/shell`（`ctx.shell`）。dsh 的 `AGENTS.md` 有一条"Explicit > implicit at package boundaries"红线：

> **Explicit > implicit at package boundaries**: defaulting is an explicit `resolve(request): Spec` step in the owning implementation, never a hidden `?? default` inside `run()`.

`ShellExecRequest`（调用者想要什么）先经过一个显式的 `resolve(request): ShellExecSpec` 步骤变成 `ShellExecSpec`（实际执行什么）——默认值在 resolve 里显式落定，绝不藏在 `run()` 里用 `?? default`。

这在 `packages/shell/shell` 的类型定义里可以看到：`ShellExecRequest` 是调用者提供的请求，`ShellExecSpec` 是解析后的完整规格。两者分离让"部署 A 的超时 30s、部署 B 的超时 120s"成为配置差异，而不是代码差异。

提供者：`bash-local`（本机 bash）、`pwsh-local`（Windows PowerShell）；消费者：`tool-bash`、`tool-pwsh`、`tool-bash-persistent`（持久 PTY）。

`tool-bash` 的 `run_in_background` 通过 `ctx.jobs` 注册后台任务，`enableRunInBackground` 配置（默认 true）关掉时直接从 schema 里移除该参数——不是"执行时拒绝"，而是"schema 里就不存在"。

## Subprocess seam：全显式 spawn spec 与 `DSH_*` 环境词汇

`ctx.subprocess` 是进程派发层（`packages/subprocess/subprocess`）：`SubprocessSpawnSpec`（全显式 spawn 参数）、基于 offset 的输出读取器、未分类的 `SubprocessOutcome`、受管理的 `DSH_*` 环境词汇。

**`DSH_*` 环境变量词汇**是 dsh 的一个巧妙设计：`@deepseek-ai/dsh-shell-env` 给子进程注入一组受管理的环境变量（来源、工作区根、session 上下文等），子进程和工具通过这组词汇互相沟通，而不需要隐式约定。

`glob`/`grep` 工具 spawn 打包的 ripgrep 二进制（`@vscode/ripgrep`）就是走 `ctx.subprocess` 的普通前台调用——"no host `rg` install and no shell layer"。

## 持久终端（terminal seam）与 PTY

`ctx.terminals` 是持久终端 seam：`terminal_open`/`terminal_send`/`terminal_read`/`terminal_close`/`terminal_list`/`terminal_signal` 六个工具（`tool-terminal`）。终端 id 是 branded，后端/会话契约明确。

`tool-bash-persistent` 是一个 owner-isolated 的持久 bash 工具——部署组合提供 PTY 后端。与 `tool-bash` 的一次性执行不同，持久终端保持会话状态（环境变量、工作目录等）。

## 架构速览

```
本地能力三大支柱
┌─────────────────────────────────────────────┐
│  FS seam (ctx.fs)                           │
│  Definition: fs/fs                          │
│  Provider: fs-local                         │
│  Consumer: tool-fs / tool-fs-search         │
│  政策: fs-observation-policy (事件门)        │
├─────────────────────────────────────────────┤
│  Shell seam (ctx.shell)                     │
│  Definition: shell/shell                    │
│  Provider: bash-local / pwsh-local          │
│  Consumer: tool-bash / tool-pwsh            │
│  resolve(request): Spec ← 显式默认          │
├─────────────────────────────────────────────┤
│  Subprocess seam (ctx.subprocess)           │
│  Definition: subprocess/subprocess          │
│  Provider: subprocess-local                 │
│  Consumer: 工具 / shell / LSP               │
│  DSH_* 环境词汇                              │
├─────────────────────────────────────────────┤
│  Terminal seam (ctx.terminals)              │
│  Definition: terminal/terminal              │
│  Consumer: tool-terminal / tool-bash-persistent │
└─────────────────────────────────────────────┘

FS + Subprocess 共享执行世界
→ 换远程沙箱全部跟着走
```

## 关键设计决策

1. **能力与政策分离**：工具只管能力，read-before-write 挂在 `fs/*` 事件门上。工具不 import 政策服务，政策也不改工具 schema。这是"通过事件解耦"的经典模式。

2. **`resolve(request): Spec` 的显式默认**：默认值在 resolve 里显式落定，绝不藏在 `run()` 里用 `?? default`。这让部署差异变成配置差异，而不是代码差异。

3. **`DSH_*` 环境变量词汇**：受管理的环境变量让子进程与工具通过约定沟通，而不是隐式约定。这比"随便设环境变量"更安全、更可审计。

4. **`enableRunInBackground` 控制 schema**：关掉时直接从 schema 里移除 `run_in_background` 参数，而不是"执行时拒绝"。模型看到的 schema 始终是准确的——不存在"模型请求了一个不支持的功能"的情况。

## 本章小结

- FS / Shell / Subprocess 是 dsh 本地能力的三大支柱，共享一个设计基因。
- 能力与政策分离：工具只管能力，read-before-write 挂在 `fs/*` 事件门上。
- 显式默认：`resolve(request): Spec` 把默认值显式落定，部署差异变配置差异。
- 共享执行世界：FS 和 subprocess provider 同源，换远程沙箱全部跟着走。
- `DSH_*` 环境变量让子进程与工具通过约定沟通。

## 阅读思考题

1. `fs-observation-policy` 通过监听 `fs/write-intent` 事件来阻止未读写入。如果一个工具绕过了 `tool-fs` 直接调用 `ctx.fs` 的写入接口，这个政策还能生效吗？
2. `resolve(request): Spec` 的设计把默认值从 `run()` 里移出来。如果一个 Provider 的 `resolve` 函数和另一个 Provider 的 `resolve` 函数对同一个字段有不同的默认值，会引入什么问题？
3. `DSH_*` 环境变量是"受管理的"——只有 `dsh-shell-env` 插件能设置它们。如果一个恶意插件也设置了 `DSH_*` 变量，会引入什么安全风险？dsh 有什么机制防止这种情况？

> **下一章预告**：第 11 章将拆开 dsh 的沙箱系统——它不是"隔离环境"，而是一个"文件效应政策"。
