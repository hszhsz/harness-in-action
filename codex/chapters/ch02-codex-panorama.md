# 第 2 章　Codex 全景：一个 Rust monorepo 长什么样

上一章我们确立了"Agent = Model + Harness"，并说明本书要拆的是 Codex 这套 Harness。这一章带你俯瞰全貌：Codex 不是一个文件、一个 crate，而是一个有 **137 个 workspace 成员**的庞大 Rust monorepo。在你一头扎进 `agent loop` 之前，先建立一张地图，知道每个子系统住在哪里，后面读起来才不会迷路。

我们会回答三个问题：仓库顶层是怎么分的？`codex-rs` 这个 workspace 内部如何分层？以及——面对 100 多个 crate，怎么高效地找到你要读的那一行。

---

## 2.1 顶层结构：npm 壳 + Rust 核

把 Codex 仓库 clone 下来，顶层目录里真正重要的只有两个：`codex-cli/` 和 `codex-rs/`。

`codex-cli/` 是一层薄薄的 **npm 包外壳**。它的核心是 `codex-cli/bin/codex.js`——一个 Node 启动脚本，作用是根据当前平台挑选正确的预编译二进制。脚本顶部维护了一张 `PLATFORM_PACKAGE_BY_TARGET` 映射表，把 `x86_64-unknown-linux-musl`、`aarch64-apple-darwin`、`x86_64-pc-windows-msvc` 等 target triple 映射到对应的平台包（如 `@openai/codex-linux-x64`、`@openai/codex-darwin-arm64`）。换句话说，当你 `npm install -g @openai/codex` 时，装的是这个 JS 壳；真正干活的是它根据你的 `process.platform` 和 `process.arch` 拉起来的那个 Rust 二进制。

`codex-rs/` 才是本书的主战场——这里住着用 Rust 写的整个 Agent。除此之外，顶层还有 `docs/`（面向用户的文档，如 `config.md`、`sandbox.md`、`authentication.md`）、`sdk/`（对外 SDK）、`scripts/` 等辅助目录，以及 `AGENTS.md`——这是一份写给 Codex（也写给所有贡献者）的工程规约，我们稍后会看到它如何成为读代码的钥匙。

值得注意的是，根目录同时存在 `BUILD.bazel`、`MODULE.bazel` 和 Cargo 的 `Cargo.toml`：Codex 既能用 Cargo 构建，也接入了 Bazel。`AGENTS.md` 里专门叮嘱：改了 Rust 依赖要跑 `just bazel-lock-update` 刷新 `MODULE.bazel.lock`，否则 CI 会因 lockfile 漂移而失败。这透露出一个事实——这是一个有严格 CI 守卫的工程化项目，而非随手开源的 demo。

---

## 2.2 `codex-rs` workspace：按职责切分的 137 个 crate

打开 `codex-rs/Cargo.toml`，你会看到 `[workspace]` 下列着 137 个成员，统一使用 `resolver = "2"` 和 `edition = "2024"`。`AGENTS.md` 第一条规约就点明了命名约定：**所有 crate 名都以 `codex-` 为前缀**——比如 `core/` 文件夹下的 crate 叫 `codex-core`，`cli/` 下的叫 `codex-cli`。

137 个 crate 听起来吓人，但它们是按职责清晰切分的。我们把与本书主线相关的 crate 归成几组，建立一张速查地图：

| 分组 | 代表 crate | 职责 | 本书章节 |
|---|---|---|---|
| **入口与分发** | `arg0`、`cli`、`exec` | 多态可执行文件、CLI 子命令、无头执行 | 第 3、16 章 |
| **核心引擎** | `core` | 线程、turn 循环、工具、安全、会话——最大的 crate | 第 4–9、12 章 |
| **模型接入** | `model-provider`、`model-provider-info`、`models-manager`、`ollama`、`lmstudio` | provider 抽象、本地模型 | 第 5 章 |
| **工具与改代码** | `apply-patch`、`tools`、`file-search` | 补丁应用、工具实现 | 第 6–7 章 |
| **沙箱与安全** | `linux-sandbox`、`bwrap`、`execpolicy`、`sandboxing`、`windows-sandbox-rs`、`process-hardening` | 多平台隔离、执行策略 | 第 8–9 章 |
| **配置与认证** | `config`、`login`、`chatgpt`、`keyring-store`、`secrets`、`aws-auth` | 配置加载合并、登录鉴权 | 第 10–11 章 |
| **状态与持久化** | `rollout`、`rollout-trace`、`thread-store`、`state`、`message-history` | 会话记录、SQLite 状态库 | 第 13 章 |
| **扩展体系** | `mcp-server`、`rmcp-client`、`plugin`、`core-plugins`、`skills`、`core-skills`、`ext/*` | MCP、插件、Skill | 第 14 章 |
| **运维与观测** | `otel`、`analytics`、`feedback`、`cli/src/doctor` | 遥测、诊断、反馈 | 第 15 章 |
| **前端** | `tui`、`exec`、`app-server` | 终端 UI、无头模式、应用服务 | 第 16 章 |

这张表就是本书的"内存映射"。注意 `core` 这个 crate 的份量——它一个人就装下了线程管理、turn 循环、工具路由、命令执行、安全判定、会话状态等几乎所有"内核"逻辑。后面好几章都会在 `core/src/` 里打转。

`ext/` 目录值得单独一提：它下面挂着 `ext/goal`、`ext/guardian`、`ext/memories`、`ext/web-search`、`ext/image-generation`、`ext/mcp`、`ext/skills` 等一系列扩展 crate。这是 Codex"核心精简、能力外挂"理念的体现——把可选能力拆成独立 crate，而不是堆进 `core`。

---

## 2.3 工程红线：`AGENTS.md` 与构建守卫

如果你想真正理解一个团队怎么维护这么大的代码库，`AGENTS.md` 是必读的。它不是产品文档，而是一份**写给 Agent 和人类贡献者共同遵守的工程纪律**，每一条背后都是踩过的坑。

几条特别能体现工程姿态的规约：

**安全相关代码"碰都不要碰"。** `AGENTS.md` 明确写道："Never add or modify any code related to `CODEX_SANDBOX_NETWORK_DISABLED_ENV_VAR` or `CODEX_SANDBOX_ENV_VAR`。" 它解释说，Codex 自己运行在一个会设置 `CODEX_SANDBOX_NETWORK_DISABLED=1` 的沙箱里；许多测试正是靠检测这个环境变量来"提前退出"那些在沙箱里跑不了的用例。这是一种自指的优雅——Codex 知道自己会被自己改代码，于是提前给关键安全开关画了红线。

**模块不许长肥。** 规约要求"Rust 模块尽量控制在 500 行以内（不含测试）"，"超过约 800 行的文件，新功能应放进新模块而非继续往里堆"。这解释了为什么 `core/src/` 下文件如此之多、切得如此之细——`turn_metadata.rs`、`turn_timing.rs`、`turn_diff_tracker.rs` 各管一摊，而不是塞进一个巨型 `turn.rs`。

**配置改动必须同步 schema。** "如果你改了 `ConfigToml` 或嵌套配置类型，运行 `just write-config-schema` 更新 `codex-rs/core/config.schema.json`。" 配置即契约，契约必须随代码同步——这是第 10 章会反复见到的"约束固化"思想的雏形。

构建侧，根目录的 `justfile` 是日常命令的入口。它把工作目录设为 `codex-rs`，提供了 `just codex`（等价 `cargo run --bin codex`）、`just exec`、`just fmt`、`just write-config-schema`、`just bazel-lock-check` 等一系列任务别名。读代码时，`justfile` 本身就是一份"这个项目都能干什么"的清单。

---

## 2.4 怎么高效地读这套代码

面对 137 个 crate，硬啃是不现实的。本书推荐的路径就是后面各章的顺序：**跟着一次请求的执行链路走**。但在战术层面，有几个屡试不爽的技巧：

第一，**从 `lib.rs` 的 `pub use` 读起**。每个 crate 的 `lib.rs` 顶部那一堆 `pub use` 和 `pub mod`，就是它对外暴露的 API 目录。比如 `core/src/lib.rs` 里你能一眼看到 `pub use codex_thread::CodexThread`、`pub use thread_manager::ThreadManager`、`pub use rollout::RolloutRecorder`——这就是 `core` 这个 crate 的"门面"，告诉你它最重要的几个类型叫什么。

第二，**用 `grep` 追确切的名字**。Codex 命名极其一致：turn 相关的都带 `turn_`，sandbox 相关的都带 `sandbox`，approval 相关的都带 `approval`。想找审批逻辑，`grep -r "ExecApprovalRequirement"` 立刻定位。

第三，**测试文件就在源文件旁边**。Codex 习惯把测试拆成同名的 `*_tests.rs`（如 `exec_policy.rs` 配 `exec_policy_tests.rs`、`safety.rs` 配 `safety_tests.rs`）。当你读不懂某个函数的契约时，去读它的 `_tests.rs`——测试里的断言往往是最精确的"它该怎么用"的说明书。本书第 17 章会专门讲这一点。

---

## 本章小结

- Codex 仓库 = 薄薄的 npm 壳（`codex-cli/`，靠 `codex.js` 按平台选二进制）+ 厚重的 Rust 核（`codex-rs/`）。
- `codex-rs` 是一个有 **137 个成员**的 Cargo workspace，`resolver = "2"`、`edition = "2024"`，所有 crate 以 `codex-` 前缀命名。
- crate 按职责清晰分组：入口分发、核心引擎（`core` 一家独大）、模型接入、工具、沙箱安全、配置认证、状态持久化、扩展、运维、前端。
- `ext/*` 体现"核心精简、能力外挂"：goal/guardian/memories/web-search 等可选能力都是独立 crate。
- `AGENTS.md` 是工程纪律的真理来源：安全开关画红线、模块限 500 行、配置改动同步 `config.schema.json`，背后都是工程姿态。
- 高效阅读三法：从 `lib.rs` 的 `pub use` 读门面、用一致命名 `grep` 追名字、读同名 `*_tests.rs` 反推契约。

## 动手实验

1. **实验一：数清 workspace 成员** — 打开 `codex-rs/Cargo.toml`，统计 `[workspace] members` 下的条目数（应为 137）。再运行 `ls codex-rs/ | wc -l` 对比目录数，理解"crate ≈ 目录"的组织方式。
2. **实验二：读懂 npm 壳如何选二进制** — 阅读 `codex-cli/bin/codex.js` 顶部的 `PLATFORM_PACKAGE_BY_TARGET` 映射表，找出你当前平台对应的 target triple 和包名。思考：为什么要把二进制拆成多个平台包分发？
3. **实验三：从 `pub use` 认识 `core` 的门面** — 打开 `codex-rs/core/src/lib.rs`，列出所有 `pub use codex_thread::*` 和 `pub use thread_manager::*` 导出的类型名。这些就是第 4 章要拆的主角。
4. **实验四：读工程红线** — 在 `AGENTS.md` 中搜索 `CODEX_SANDBOX_NETWORK_DISABLED_ENV_VAR`，读懂为什么这段代码"碰都不要碰"。再搜索 `500`，找到"模块行数上限"的规约，对照 `core/src/` 下文件的细碎程度验证它确实被执行了。

> **下一章预告**：地图已经在手。下一章我们从最外层的进程入口切入——当你在终端敲下 `codex` 那一刻，npm 壳如何拉起 Rust 二进制，`arg0` 多态分发如何让同一个可执行文件扮演 CLI、沙箱、`apply_patch` 等多种角色，一次请求的旅程就此开始。
