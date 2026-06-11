# 第 1 章　全景与运行时入口

每一本解构源码的书，第一刀都应该切在"进程是怎么活起来的"。在你读懂 Agent 主循环、上下文压缩、工具审批这些精巧的内部机制之前，得先回答一个朴素的问题：当用户在终端敲下 `hermes` 并回车，到底发生了什么？哪一行代码先被执行？参数怎么被解析？子命令怎么被分发？这一章我们就从 Hermes Agent 的最外层——打包声明、可执行入口、命令分发表、全局常量——一路读到 Agent 真正开始一次对话之前的那一刻。

Hermes Agent 是一个体量惊人的项目。仅看仓库根目录，`cli.py` 超过 60 万字节，`run_agent.py` 接近 24 万字节，`hermes_state.py` 近 20 万字节。它不是那种"几百行讲清楚一个 Agent"的教学项目，而是一个长期演进、承载真实用户的生产系统。正因如此，理解它的"骨架结构"——哪些是入口、哪些是核心模块、哪些是可选扩展——就成了读懂全书的前提。

## 1.1 身份与打包：从 `pyproject.toml` 读出工程纪律

一个项目的 `pyproject.toml` 往往是它工程价值观的第一张名片。Hermes 的 `pyproject.toml` 里，`[project]` 段声明了 `name = "hermes-agent"`、`version = "0.16.0"`，作者 Nous Research，许可证 MIT。

真正值得停下来看的是 Python 版本约束：`requires-python = ">=3.11,<3.14"`。下界 3.11 是为了用上较新的类型与异步特性，而**上界 `<3.14` 是刻意写死的**——源码注释解释，若不封顶，`uv` 会去拉 3.14，而那时 `pydantic-core` 还没有 cp314 的预编译 wheel，会导致安装失败。这是一种典型的"防御性约束"：与其让用户在未来某天突然装不上，不如现在就把已知不兼容的版本挡在门外。

更能说明问题的是依赖的钉法。Hermes 的核心依赖大量采用**精确钉版**（`==X.Y.Z`），例如 `openai==2.24.0`、`fire==0.7.1`、`httpx[socks]==0.28.1`、`rich==14.3.3`、`pydantic==2.13.4`、`prompt_toolkit==3.0.52`、`croniter==6.0.0`、`Pillow==12.2.0`。源码注释直言这是出于**供应链安全**考量——精确钉版能把"某个依赖被投毒"的爆炸半径限制住，注释里甚至点名了曾经的 "Mini Shai-Hulud worm" 与 mistralai 的问题版本作为前车之鉴。与此同时，那些重量级、与具体能力强相关的依赖（`anthropic`、`modal`、`daytona`、`slack`、`matrix`、`messaging` 等）被拆进了可选 extras，只有用到对应功能时才装。**核心精简、能力按需加载**——这条线索会贯穿全书。

## 1.2 三个入口：`console_scripts` 暴露的命令面

Hermes 不是只有一个入口。`[project.scripts]` 里声明了三个 console script，它们对应三种截然不同的使用姿势：

- `hermes = "hermes_cli.main:main"` —— 面向人类的交互式 CLI，是绝大多数用户的入口。
- `hermes-agent = "run_agent:main"` —— 直接运行单次 Agent 任务的脚本式入口。
- `hermes-acp = "acp_adapter.entry:main"` —— ACP（Agent Client Protocol）适配器入口，供外部客户端按协议接入。

`[tool.setuptools]` 的 `py-modules` 还列出了一批顶层单文件模块：`run_agent`、`model_tools`、`toolsets`、`batch_runner`、`trajectory_compressor`、`toolset_distributions`、`cli`、`hermes_bootstrap`、`hermes_constants`、`hermes_state`、`hermes_time`、`hermes_logging`、`utils`、`mcp_serve`。把这串名字读一遍，你其实已经对全书的章节地图有了模糊预感：`toolsets` 是工具分组（第 4 章）、`hermes_state` 是状态持久化（第 10 章）、`trajectory_compressor` 是轨迹压缩（第 3 章）、`hermes_logging` 是可观测性（第 13 章）。

## 1.3 启动序列：从 `hermes` 脚本到 `main()`

让我们顺着 `hermes = "hermes_cli.main:main"` 这条主入口往里走。

仓库根目录有一个名为 `hermes` 的可执行脚本，shebang 是 `#!/usr/bin/env python3`，在 `if __name__ == "__main__":` 下做的事极其简单：`from hermes_cli.main import main; main()`。它只是一个轻量转发器。

在真正进入 `main()` 之前，`hermes_bootstrap.py` 会先处理一个跨平台的老大难问题——Windows 的编码。`apply_windows_utf8_bootstrap()` 在模块导入时（约第 129 行）被调用：它 `os.environ.setdefault("PYTHONUTF8", "1")`、设置 `PYTHONIOENCODING="utf-8"`，再对 stdout/stderr/stdin 调用 `reconfigure(encoding="utf-8", errors="replace")`。在 POSIX 上它是个 no-op，且用模块级全局 `_bootstrap_applied` 保证只跑一次。这是一个很小但很典型的细节：**把平台差异收敛在最外层处理掉，让内部代码可以假装世界永远是 UTF-8**。

真正的总指挥是 `hermes_cli/main.py:main()`（约第 10847 行）。它的开场动作是一连串"环境净化"：`_set_process_title()` 设置进程标题，`configure_windows_stdio()` 再次确认 Windows 标准流，`_cleanup_quarantined_exes()` 清理被隔离的可执行文件，`_recover_from_interrupted_install()` 尝试从一次被中断的安装中恢复（但当 `"update" in sys.argv[1:]` 时会跳过，避免和更新流程打架）。随后是 Termux（Android 终端）的两条快速路径 `_try_termux_fast_tui_launch()` 与 `_try_termux_fast_cli_launch()`——在资源受限的移动环境里走捷径直接拉起界面。

走完这些前置，`main()` 调用 `build_top_level_parser()`（来自 `hermes_cli._parser`）构建顶层 argparse 解析器并注册所有子命令解析器，然后 `parser.parse_args(...)`。

## 1.4 命令分发：`func` 约定与默认聊天

Hermes 的子命令分发用的是 argparse 里一个经典而优雅的模式。每个子命令在构建自己的子解析器时，都通过 `set_defaults(func=...)` 把自己的处理函数挂到解析结果上。于是分发逻辑可以写得极简（约第 11880 行）：

```python
if hasattr(args, "func"):
    args.func(args)
else:
    parser.print_help()
```

而当用户什么子命令都不带、直接敲 `hermes` 时（`args.command is None`，约第 11863 行），代码会补齐若干属性默认值并直接调用 `cmd_chat(args)`——也就是说，**裸 `hermes` 的语义就是"开始聊天"**。这种"无参即进入最常用功能"的设计，降低了新用户的认知门槛。

子命令本身则通过一系列 `build_*_parser(subparsers, cmd_*=...)` 从 `hermes_cli/subcommands/*.py` 里导入并注册，覆盖了 `model`、`fallback`、`gateway`、`setup`、`config`、`cron`、`doctor`、`security`、`backup`、`skills`、`plugins`、`memory`、`tools`、`mcp`、`pairing`、`update` 等数十个命令。这串命令名几乎就是 Hermes 全部能力的索引。

这里还藏着一个值得学习的工程手法。`main.py` 里定义了一个 `_BUILTIN_SUBCOMMANDS` frozenset（约第 10368 行），把所有内置子命令名（`acp`、`auth`、`config`、`cron`、`gateway`、`model`、`skills`、`tools`、`memory`、`chat` 等等）冻结成一个集合。它的用途是**短路插件发现**：当用户敲的是一个已知的内置命令时，根本不需要去扫描、加载用户安装的插件子命令，从而省下启动开销。只有当命令不在这个集合里时，才走动态插件发现的慢路径。用一个 frozenset 把"快路径"和"慢路径"分开，是大型 CLI 控制冷启动延迟的常见做法。

## 1.5 另一条入口：`run_agent.py` 的 Fire 式脚本运行

如果说 `hermes` 是给人用的，那么 `hermes-agent`（即 `run_agent.py:main`，约第 5145 行）就是给脚本和自动化用的。它的分发方式完全不同——基于 Google 的 `fire` 库：`if __name__ == "__main__": fire.Fire(main)`（约第 5361 行）。`fire` 会把函数签名自动映射成命令行参数，于是 `main()` 的默认参数就直接定义了脚本接口：`model=""`、`max_turns=10`、`log_prefix_chars=20`，外加 `enabled_toolsets` / `disabled_toolsets`、`list_tools`、`save_trajectories`、`save_sample`、`verbose` 等。文档里给出的默认模型是经由 OpenRouter 访问的 `anthropic/claude-sonnet-4.6`（base URL `https://openrouter.ai/api/v1`）。这条路径的核心类是 `AIAgent`（约第 320 行）——它正是我们下一章要解剖的主角。

两个入口、两套参数解析哲学（argparse 之于人，fire 之于脚本），却复用同一套 Agent 内核。这种"多入口、单内核"的结构，是让一个 Agent 既能交互又能批处理、还能被协议接入的关键。

## 1.6 全局常量：`hermes_constants.py` 里的契约

最后看一眼 `hermes_constants.py`，它定义了一批贯穿全系统的常量与基础工具函数，相当于全工程的"宪法条款"。

推理强度的取值被收敛成一个元组：`VALID_REASONING_EFFORTS = ("minimal", "low", "medium", "high", "xhigh")`，而 `parse_reasoning_effort()` 还额外接受 `"none"`，把它解析为 `{"enabled": False}`。流式相关有 `PARTIAL_STREAM_STUB_ID = "partial-stream-stub"` 和 `FINISH_REASON_LENGTH = "length"`（后者在第 2、3 章处理"模型因长度截断"时会反复出现）。默认接入点是 `OPENROUTER_BASE_URL = "https://openrouter.ai/api/v1"`。

更关键的是一组关于"家目录"的函数。`get_hermes_home()` 决定 Hermes 把配置、技能、状态都放在哪里：POSIX 默认 `~/.hermes`，Windows 默认 `%LOCALAPPDATA%\hermes`；它尊重 `HERMES_HOME` 环境变量，也支持一个上下文局部的覆盖机制（`set_hermes_home_override`）。配套的 `secure_parent_dir()` 会把目录权限 chmod 成 `0o700`，但**拒绝对 `/` 或路径段少于 3 的目录动手**——这是一道防止"误把根目录权限改掉"的安全闸。此外还有一组环境探测器：`is_termux()`、`is_wsl()`（读 `/proc/version` 找 "microsoft" 字样）、`is_container()`（检查 `/.dockerenv`、`/run/.containerenv`、`/proc/1/cgroup`）。这些探测结果会在后续的安装、沙箱、网关行为里被反复用到。

把这一节的常量连起来看，你会发现 Hermes 的一个底层习惯：**把"什么是合法值"用数据结构（元组、frozenset）固化下来，把"路径与权限"用带防御分支的函数封装起来**。常量不是散落各处的魔法字符串，而是有名字、有边界、有校验的契约。

## 本章小结

- Hermes Agent 是一个 `version = "0.16.0"` 的大型 Python 工程，核心单文件模块体量巨大（`cli.py` 60 万+字节），核心精简、能力按 extras 与插件按需加载是其基本结构哲学。
- `requires-python = ">=3.11,<3.14"` 的上界是刻意写死的，为的是规避 `pydantic-core` 缺少 cp314 wheel；核心依赖精确钉版（如 `openai==2.24.0`）出于供应链安全，把投毒爆炸半径限制住。
- 项目暴露三个 console script 入口：`hermes`（交互 CLI）、`hermes-agent`（脚本式 Fire 入口）、`hermes-acp`（协议适配器），多入口共享同一 Agent 内核。
- 启动序列为 `hermes` 脚本 → `hermes_bootstrap` 处理 Windows UTF-8 → `hermes_cli/main.py:main()` 做环境净化 → 构建 argparse → 分发。
- 命令分发依赖 `set_defaults(func=...)` 约定与 `if hasattr(args, "func")` 调用；裸 `hermes` 默认进入 `cmd_chat`；`_BUILTIN_SUBCOMMANDS` frozenset 用于短路插件发现以控制冷启动延迟。
- `run_agent.py` 用 `fire.Fire(main)` 提供脚本接口，默认 `max_turns=10`、默认模型经 OpenRouter 访问 `anthropic/claude-sonnet-4.6`，核心类是 `AIAgent`。
- `hermes_constants.py` 把合法取值（`VALID_REASONING_EFFORTS`）、接入点、家目录解析（`get_hermes_home`）与带防御分支的权限收紧（`secure_parent_dir` 拒绝 `/`）固化为全工程契约。

## 动手实验

1. **实验一：画出启动调用链** —— 在本地 clone 仓库，从 `hermes` 这个可执行脚本读起，依次 `Read` `hermes_bootstrap.py` 的 `apply_windows_utf8_bootstrap()` 与 `hermes_cli/main.py` 的 `main()`。把从"敲下 hermes"到"调用 args.func"之间的每一步函数调用按顺序记成一张时序图，特别标注哪些步骤在 POSIX 上是 no-op。

2. **实验二：验证默认聊天行为** —— 在 `hermes_cli/main.py` 中用 Grep 定位 `args.command is None` 的分支（约第 11863 行），读懂它如何补默认值并调用 `cmd_chat`。再对照 `if hasattr(args, "func")` 的分发逻辑，解释为什么"裸 hermes"和"hermes chat"最终走到同一处。

3. **实验三：统计命令面** —— 用 Grep 在 `hermes_cli/subcommands/` 下统计所有 `build_*_parser` 的数量，并把它们与 `_BUILTIN_SUBCOMMANDS` frozenset 里的名字做差集。思考：哪些子命令在 frozenset 里、哪些不在？不在的那些为什么可以走插件慢路径？

4. **实验四：读懂家目录的安全边界** —— `Read` `hermes_constants.py` 里的 `get_hermes_home()` 与 `secure_parent_dir()`。尝试回答：如果有人把 `HERMES_HOME` 设成 `/`，`secure_parent_dir()` 会发生什么？这道"路径段少于 3 则拒绝"的判断防住了哪种事故？

> **下一章预告**：进程已经活起来、命令已经分发到 `cmd_chat`，接下来真正的戏肉登场——`AIAgent` 如何把一次用户消息变成一轮又一轮的"思考-调用工具-观察"循环。第 2 章我们将解剖 `agent/conversation_loop.py:run_conversation`，看清一个回合（turn）的完整生命周期，以及那些写死的 `< 3` 重试上限背后的设计意图。
