# 第 10 章　文件系统与 Shell：本地能力的两大支柱

> 一个编码 Agent 的两只手：读写文件（FS seam）和执行命令（Shell + Subprocess seam）。这一章拆开 dsh 的三个能力 seam——filesystem、shell、subprocess——以及它们如何共享同一个"执行世界"。

## 10.1　FS seam：`FsTarget` / 读写编辑 / `fs/observed` 观察状态

`ctx.fs` 是文件系统能力 seam 的服务定义（`packages/fs/fs`）。它提供 `FsTarget`（路径目标）、读/写/编辑结果、observed-file 状态、`FsErrorCode`。[[docs/subsystems/filesystem.md]](https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/subsystems/filesystem.md)

消费者是 `tool-fs`（`edit`/`read`/`read_image`/`write` 四个工具）、`tool-str-replace-editor`（独立的 view/create/unique literal replace/line insert 工具）、`tool-fs-search`（`glob`/`grep`，spawn 打包的 `@vscode/ripgrep`）。

工具注册与模型 schema 的耦合点：`read_image` 只有在 `ctx.attachments` 存在时才注册（schema 与路由无关，执行时拒绝除非确切路由的模型声明支持图像输入）。`tool-fs-search` 的 capped 结果通过 `ctx.spillStore` 保存完整列表（第 13.4 节）。

## 10.2　read-before-write 策略与 `fs/*` 事件门

dsh 的 read-before-write/edit 政策由 `@deepseek-ai/dsh-fs-observation-policy` 提供——一个 **`fs/*` 事件门插件**（"an `fs/*` event-gate plugin, no schema change"）。

机制：tool-fs 的突变发出 `fs/write-intent` 或 `fs/edit-intent` 事件；观察政策监听这些事件，检查该文件是否已经被 read 过（`fs/observed` 状态）；没读过就不放行。部署加载这些工具时被期望同时加载这个政策（"a deployment that loads these tools is expected to also load it"）。

关键：**政策挂在事件上，不挂在工具上**——工具不 import 政策服务，政策也不改工具 schema。这正是第 8 章管线哲学的延伸：工具只管能力，政策通过事件门插入。

## 10.3　Shell seam：`ShellExecRequest/Spec` 的显式分离

Shell 能力 seam 的服务定义在 `shell/shell`（`ctx.shell`）。dsh 的 `AGENTS.md` 有一条"Explicit > implicit at package boundaries"红线，`dsh-shell` 的 request/spec 分离就是模板：

> **Explicit > implicit at package boundaries**: defaulting is an explicit `resolve(request): Spec` step in the owning implementation, never a hidden `?? default` inside `run()`.

`ShellExecRequest`（调用者想要什么）先经过一个显式的 `resolve(request): ShellExecSpec` 步骤变成 `ShellExecSpec`（实际执行什么）——默认值在 resolve 里显式落定，绝不藏在 `run()` 里用 `?? default`。这让"部署 A 的超时 30s、部署 B 的超时 120s"成为配置差异，而不是代码差异。

提供者：`bash-local`（本机 bash）、`pwsh-local`（Windows PowerShell）；消费者：`tool-bash`、`tool-pwsh`、`tool-bash-persistent`（持久 PTY）。`tool-bash` 的 `run_in_background` 通过 `ctx.jobs` 注册后台任务（第 15.4 节），`enableRunInBackground` 配置（默认 true）关掉时直接从 schema 里移除该参数。

## 10.4　Subprocess seam：全显式 spawn spec 与 `DSH_*` 环境词汇

`ctx.subprocess` 是进程派发层（`packages/subprocess/subprocess`）：`SubprocessSpawnSpec`（全显式 spawn 参数）、基于 offset 的输出读取器、未分类的 `SubprocessOutcome`、受管理的 `DSH_*` 环境词汇。[[docs/subsystems/subprocess.md]](https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/subsystems/subprocess.md)

**`DSH_*` 环境变量词汇**是 dsh 的一个巧妙设计：`@deepseek-ai/dsh-shell-env` 给子进程注入一组受管理的环境变量（来源、工作区根、session 上下文等），子进程和工具通过这组词汇互相沟通，而不需要隐式约定。

`glob`/`grep` 工具 spawn 打包的 ripgrep 二进制（`@vscode/ripgrep`）就是走 `ctx.subprocess` 的普通前台调用——"no host `rg` install and no shell layer"。

## 10.5　持久终端（terminal seam）与 PTY

`ctx.terminals` 是持久终端 seam：`terminal_open`/`terminal_send`/`terminal_read`/`terminal_close`/`terminal_list`/`terminal_signal` 六个工具（`tool-terminal`）。终端 id 是 branded，后端/会话契约明确，send readiness、bounded reads、owner-visible snapshots 都有定义。`tool-bash-persistent` 是一个 owner-isolated 的持久 bash 工具——部署组合提供 PTY 后端。

## 本章小结

FS / Shell / Subprocess 是 dsh 本地能力的三大支柱，共享一个设计基因：

1. **能力与政策分离**：工具只管能力，read-before-write 挂在 `fs/*` 事件门上；
2. **显式默认**：`resolve(request): Spec` 把默认值显式落定，部署差异变配置差异；
3. **共享执行世界**：FS 和 subprocess provider 同源，换远程沙箱全部跟着走；
4. **环境词汇受管理**：`DSH_*` 让子进程与工具通过约定沟通；
5. **持久终端独立 seam**：一次性 shell 与持久 PTY 分开设计。

## 动手实验

1. 读 `packages/fs/fs-observation-policy`，确认它监听哪些 `fs/*` 事件、如何阻止未读写入。
2. 读 `packages/shell/shell` 的 `resolve(request): Spec` 签名，找一处默认值落定。
3. 读 `packages/subprocess/subprocess` 的 `SubprocessSpawnSpec`，数一数它显式声明了多少字段。
4. 在 `packages/shell/tool-bash` 里找 `run_in_background` 的 schema 移除逻辑（`enableRunInBackground`）。
