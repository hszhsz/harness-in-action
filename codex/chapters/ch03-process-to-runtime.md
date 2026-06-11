# 第 3 章　从进程到运行时：npm 启动器、`arg0` 多态与 CLI 分发

当你在终端敲下 `codex` 并回车，到模型开始为你改代码之间，发生了一连串你看不见的事。这一章顺着这条最外层的链路走一遍：npm 壳如何拉起 Rust 二进制、为什么"同一个可执行文件能扮演好几种角色"、以及 clap 如何把你的命令路由到正确的子命令。理解了进程模型，你才知道后面所有子系统是被"谁"、在"什么上下文"里调起来的。

---

## 3.1 npm 壳：`codex.js` 的平台分流

上一章已经点过，`codex-cli/bin/codex.js` 是一层 Node 启动脚本。它的逻辑很直接：读取 `process.platform` 和 `process.arch`，拼出当前平台的 target triple（如 `x86_64-unknown-linux-musl`），再通过 `PLATFORM_PACKAGE_BY_TARGET` 映射表找到对应的平台包（如 `@openai/codex-linux-x64`），最后 `spawn` 出那个包里的原生二进制，把命令行参数、stdio 原样转交。

这种"JS 壳 + 平台二进制"的分发模式，让 `npm install -g @openai/codex` 这一条命令在任何平台上都能装到正确的原生程序，而用户无需关心自己是 Intel Mac 还是 Apple Silicon。从这一刻起，控制权就交给了 Rust。

---

## 3.2 `arg0` 多态：一个二进制，多种人格

进入 Rust 世界，你会遇到 Codex 进程模型里最巧妙的设计——**arg0 trick**。它的目标，用源码注释里的原话说，是"我们想把 Codex CLI 部署成单个可执行文件以求简单，但又想把它的一部分功能暴露成独立的 CLI"。解决办法是：**根据进程被调用时的名字（argv[0]，即 arg0），决定这个二进制此刻该扮演谁。**

这套逻辑集中在 `arg0/src/lib.rs` 的 `arg0_dispatch` 函数里。它先取出 `argv[0]` 的文件名，然后逐一比对：

- 如果名字是 `CODEX_LINUX_SANDBOX_ARG0`（即 `codex-linux-sandbox`），就**直接**调用 `codex_linux_sandbox::run_main()`——注释标注它 "never returns"，进程就此变成一个纯粹的 Linux 沙箱执行器。
- 如果名字是 `APPLY_PATCH_ARG0`（`"apply_patch"`）或拼错的 `MISSPELLED_APPLY_PATCH_ARG0`（`"applypatch"`），就调用 `codex_apply_patch::main()`，进程变成补丁工具。
- 如果第一个参数 `argv[1]` 是 `CODEX_FS_HELPER_ARG1`，进程变成文件系统 helper（`codex_exec_server::run_fs_helper_main()`）。
- 如果 `argv[1]` 是 `CODEX_CORE_APPLY_PATCH_ARG1`，进程会读取后面的 PATCH 参数，构造一个 current-thread 的 Tokio runtime，调用 `codex_apply_patch::apply_patch(...)` 应用补丁后退出。
- 在 Unix 上，如果名字是 `EXECVE_WRAPPER_ARG0`（`"codex-execve-wrapper"`），它会跑 `run_shell_escalation_execve_wrapper`，这是 shell 提权的包装器。

为什么要这么设计？因为 Codex 需要在运行时**重新执行自己**来完成某些隔离任务——比如要在 Linux 沙箱里跑一条命令，它不会去找另一个二进制，而是用一个叫 `codex-linux-sandbox` 的名字重新 exec 当前这个二进制。注释里也坦白了局限："在 Mac 和 Linux 上可以这样模拟多个可执行文件，但 Windows 不行。"

走完这些特殊分支后，正常的 CLI 路径才开始：`arg0_dispatch` 调用 `load_dotenv()` 加载 `.env`（注释强调这必须在创建任何线程/Tokio runtime **之前**做，因为修改环境变量不是线程安全的），然后通过 `prepare_path_env_var_with_aliases` 把 Codex 的别名（如 `apply_patch`）注入到一个临时 PATH 目录里——这样模型在 shell 里直接调 `apply_patch` 时，命中的就是当前二进制的别名。这个 PATH 入口由 `Arg0PathEntryGuard` 持有，靠一个 `.lock` 文件锁定，存活整个进程生命周期。

还有一个容易忽略的细节：`TOKIO_WORKER_STACK_SIZE_BYTES` 被设为 `16 * 1024 * 1024`（16 MiB）。Codex 显式调大了 Tokio worker 线程的栈，因为它的某些同步递归逻辑（如补丁/diff 处理）需要更深的栈空间。

---

## 3.3 `MultitoolCli`：clap 如何路由你的命令

通过了 arg0 多态分发，真正的 CLI 解析在 `cli/src/main.rs` 里展开。Codex 用 [clap](https://docs.rs/clap) 定义了一个名为 `MultitoolCli` 的顶层解析器。它的 `#[clap(...)]` 属性透露了几个设计决策：

- `subcommand_negates_reqs = true`：如果给了子命令，就忽略默认（交互式）参数的必填要求。
- `bin_name = "codex"`：即便二进制实际叫 `codex-x86_64-unknown-linux-musl`，help 输出里也永远显示成用户真正敲的 `codex`。
- `override_usage`：手写了用法行 `codex [OPTIONS] [PROMPT]` 与 `codex [OPTIONS] <COMMAND> [ARGS]`，明确表达"不带子命令时，剩余参数当作给交互式 TUI 的 PROMPT"。

`MultitoolCli` 用 `#[clap(flatten)]` 把几组横切选项摊平进来：`CliConfigOverrides`（配置覆盖，第 10 章）、`FeatureToggles`（特性开关）、`InteractiveRemoteOptions`、以及 `TuiCli`（交互式 TUI 的参数）。最后是一个 `Option<Subcommand>`——这正是"不带子命令默认进 TUI"的实现基础。

`Subcommand` 这个 enum 几乎就是 Codex 的功能全景图，值得通读。挑几个与本书主线相关的：

| 子命令 | 作用 | 本书章节 |
|---|---|---|
| `Exec`（别名 `e`） | 非交互式运行 Codex | 第 16 章 |
| `Review` | 非交互式跑代码评审 | — |
| `Login` / `Logout` | 管理登录凭据 | 第 11 章 |
| `Mcp` / `McpServer` | 管理外部 MCP server / 把 Codex 自己当 MCP server | 第 14 章 |
| `Plugin` | 管理 Codex 插件 | 第 14 章 |
| `Doctor` | 诊断本地安装、配置、认证、运行时健康 | 第 15 章 |
| `Sandbox` | 在 Codex 提供的沙箱内跑命令 | 第 8 章 |
| `Apply`（别名 `a`） | 把 Codex 产出的最新 diff 用 `git apply` 应用到工作区 | 第 7 章 |
| `Resume` / `Fork` / `Archive` / `Delete` | 恢复、分叉、归档、删除历史会话 | 第 13 章 |
| `Execpolicy` | 执行策略工具（`#[clap(hide = true)]` 隐藏） | 第 9 章 |

注意几个带 `#[clap(hide = true)]` 的内部子命令：`ResponsesApiProxy`、`StdioToUds`、`Execpolicy`。它们不在 help 里展示，是供 Codex 内部自我调用的——这与 arg0 多态是同一种"一个二进制干多件事"的思路，只不过一个走文件名分发，一个走子命令分发。

`main.rs` 顶部那一长串 `use` 也是一张缩小版的依赖地图：`codex_login::AuthManager`（认证）、`codex_core::config::ConfigBuilder`（配置）、`codex_core::build_models_manager`（模型管理）、`codex_tui::Cli`（TUI）、`codex_exec::Cli`（exec）……几乎本书后续每一章的主角，都在这里被 `cli` crate 当作零件组装起来。`cli` 本身几乎不含业务逻辑，它的职责就是**把各个 crate 的能力按子命令编排成一个统一入口**。

---

## 3.4 启动序列总览

把这一章的链路串起来，一次 `codex "帮我修复这个 bug"` 的启动序列是：

1. **npm 壳**：`codex.js` 按平台 spawn 原生二进制，转交参数。
2. **arg0 分发**：`arg0_dispatch` 检查 argv[0]/argv[1]，若是沙箱/apply_patch 等特殊角色则直接进入对应 `run_main` 并不再返回；否则 `load_dotenv()` + 注入 PATH 别名，返回 `Arg0PathEntryGuard`。
3. **clap 解析**：`MultitoolCli` 解析参数；无子命令 → 走交互式 TUI 路径（把剩余参数当 PROMPT），有子命令 → 路由到 `Subcommand` 对应分支。
4. **构建运行时**：选定路径后，`cli` 用 `ConfigBuilder` 加载配置、用 `AuthManager` 准备认证、用 `build_models_manager` 准备模型，最终把控制权交给 `core` 里的线程与主循环——这正是下一章的主题。

这个序列体现了一个清晰的分层：**`arg0` 管"我是谁"，`cli` 管"我要干什么"，`core` 管"具体怎么干"。** 三层各司其职，互不越界。

---

## 本章小结

- `codex-cli/bin/codex.js` 是 npm 壳，按 `process.platform`/`arch` 经 `PLATFORM_PACKAGE_BY_TARGET` 选择平台二进制并 spawn。
- **arg0 trick**（`arg0/src/lib.rs::arg0_dispatch`）让单个二进制按 argv[0]/argv[1] 扮演多种角色：`codex-linux-sandbox`、`apply_patch`/`applypatch`、`codex-execve-wrapper`、fs-helper、core-apply-patch。这是 Codex 在运行时安全地"重新执行自己"的基础。
- 走完特殊分支后，`arg0_dispatch` 在建线程前 `load_dotenv()`，并用 `Arg0PathEntryGuard`（靠 `.lock` 锁定）把 `apply_patch` 等别名注入临时 PATH。
- `TOKIO_WORKER_STACK_SIZE_BYTES = 16 MiB`：显式调大 worker 栈以容纳深递归逻辑。
- `cli/src/main.rs` 用 clap 定义 `MultitoolCli`：`bin_name="codex"`、`subcommand_negates_reqs=true`、无子命令默认进 TUI。
- `Subcommand` enum 是功能全景图；`#[clap(hide=true)]` 的 `ResponsesApiProxy`/`StdioToUds`/`Execpolicy` 是内部自调用子命令。
- 分层清晰：`arg0` 管"我是谁"、`cli` 管"干什么"、`core` 管"怎么干"。

## 动手实验

1. **实验一：追踪平台分流** — 阅读 `codex-cli/bin/codex.js`，找到 `PLATFORM_PACKAGE_BY_TARGET` 与 `switch (platform)` 逻辑，画出从 `process.platform`/`arch` 到最终 spawn 哪个二进制的决策树。
2. **实验二：列出所有 arg0 角色** — 在 `codex-rs/arg0/src/lib.rs` 中搜索所有形如 `*_ARG0`、`*_ARG1` 的常量（如 `CODEX_LINUX_SANDBOX_ARG0`、`APPLY_PATCH_ARG0`、`CODEX_FS_HELPER_ARG1`），列出每个名字对应进程会变成什么角色。
3. **实验三：读懂"为什么 load_dotenv 必须在建线程前"** — 阅读 `arg0_dispatch` 中 `load_dotenv()` 调用处的注释，结合 `unsafe { std::env::set_var(...) }` 的注释，解释为什么修改环境变量必须在进程还是单线程时完成。
4. **实验四：通读功能全景** — 打开 `cli/src/main.rs` 的 `enum Subcommand`，逐条读 doc 注释。找出所有 `#[clap(hide = true)]` 的子命令，思考它们为什么对用户隐藏而对 Codex 自己可见。

> **下一章预告**：进程已经启动、命令已经路由、配置与认证已经就绪，控制权终于交到了 `core`。下一章我们进入 Agent 的心脏——`CodexThread` 与 `ThreadManager`，看看一次 turn 是如何从你的输入开始、经过模型流式响应与工具调用、再把结果回灌的，"心跳"究竟是怎么跳的。
