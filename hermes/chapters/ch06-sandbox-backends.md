# 第 6 章　沙箱与六种终端后端

第 5 章我们解决了"哪些命令不许跑、哪些需要先问"。这一章要回答下一个问题：通过了审批的命令，**到底在哪里、以什么方式执行**？答案出乎意料地丰富——Hermes 支持六种截然不同的"终端后端"：本地进程、Docker 容器、SSH 远程主机、Singularity（HPC 集群常用的容器）、Modal（serverless）、Daytona（serverless）。同一个 Agent，可以把命令跑在你的笔记本上，也可以跑在一个闲时几乎零成本、用时自动唤醒的云端沙箱里。

要支撑这种灵活性，又不让上层代码为每种后端写一套逻辑，关键在于一个干净的抽象。这一章我们就读 `tools/environments/` 这个目录：先看统一所有后端的抽象基类，再看命令是如何被"包裹"成一段可隔离、可复现的脚本，最后看后端是如何被选择和切换的。

## 6.1 抽象基类：`BaseEnvironment`

`tools/environments/` 目录的文件清单已经把全貌摆了出来：`base.py`、`local.py`、`docker.py`、`ssh.py`、`singularity.py`、`modal.py`、`managed_modal.py`、`modal_utils.py`、`daytona.py`、`file_sync.py`、`__init__.py`。六种后端，外加文件同步和工具模块。

统一它们的是 `base.py` 约第 288 行的 `class BaseEnvironment(ABC)`（用 `from abc import ABC, abstractmethod`）。构造函数是 `__init__(self, cwd, timeout, env=None)`——三个最基本的执行上下文：工作目录、超时、环境变量。类属性 `_snapshot_timeout: int = 30` 给"环境快照"操作设了 30 秒上限。

有意思的是，整个基类**只有一个 `@abstractmethod`**：`cleanup(self)`（约第 342–343 行）。也就是说，对一个后端的最低要求只有一条——你必须能清理自己（关掉容器、断开 SSH、释放 serverless 资源）。其余的执行逻辑则由一个共享的具体方法 `execute(self, command, cwd="", *, timeout=None, stdin_data=None, rewrite_compound_background=True)`（约第 829 行）统一提供，它返回一个朴素的字典 `{"output": str, "returncode": int}`，并用 `effective_timeout = timeout or self.timeout` 决定本次超时。各后端真正要各自实现的是 `_run_bash(self, cmd_string, *, login=False, timeout=120, ...)`——把一段 bash 脚本送到自己的执行环境里去跑。

这个抽象的精妙在于**职责切分的位置**。它没有让每个后端从零实现 `execute`，而是把"如何把命令包装成脚本、如何处理超时、如何捕获结果"这些通用逻辑放在基类的 `execute` 里，只把"这段脚本具体在哪儿跑"（`_run_bash`）和"怎么收尾"（`cleanup`）留给子类。新增一个后端，你只需回答两个问题：怎么跑一段 bash、怎么清理。这就是为什么 Hermes 能支持六种后端而代码不爆炸。

## 6.2 六种后端：从本地到 serverless

逐一看这六个具体类，每一个都对应一种部署形态：

- `LocalEnvironment(BaseEnvironment)`（`local.py` 约第 495 行），默认 `timeout=60`。最简单的后端，命令直接在宿主机进程里跑。
- `DockerEnvironment(BaseEnvironment)`（`docker.py` 约第 503 行）。命令在 Docker 容器里执行，提供进程级隔离。
- `SSHEnvironment(BaseEnvironment)`（`ssh.py` 约第 36 行），默认 `port=22`、`key_path=""`。把命令送到远程主机执行。
- `SingularityEnvironment(BaseEnvironment)`（`singularity.py` 约第 158 行）。Singularity（现 Apptainer）是高性能计算集群里常用的无 root 容器方案，这个后端让 Hermes 能跑在 HPC 环境。
- `ModalEnvironment(BaseEnvironment)`（`modal.py` 约第 164 行），把 `_snapshot_timeout` 覆盖为 60。Modal 是 serverless 平台。
- `ManagedModalEnvironment(BaseModalExecutionEnvironment)`（`managed_modal.py` 约第 36 行），其基类 `BaseModalExecutionEnvironment(BaseEnvironment)` 在 `modal_utils.py` 约第 58 行。这是 Modal 的"托管"变体。
- `DaytonaEnvironment(BaseEnvironment)`（`daytona.py` 约第 30 行）。它的实现里有一句很诚实的 docstring："Shell timeout wrapper preserved (SDK timeout unreliable)"——因为 Daytona SDK 自带的超时不可靠，所以 Hermes 自己又包了一层 shell 超时（调用 `sandbox.process.exec(shell_cmd, timeout=timeout)`）。这种"不信任第三方 SDK 的承诺、自己再兜一层"的态度，正是生产系统对待外部依赖的典型姿势。

Modal 和 Daytona 这两个 serverless 后端是 Hermes 的一大卖点：环境在闲置时休眠、几乎零成本，用时自动唤醒并保持状态。这让"在云端养一个随时待命的 Agent"在经济上变得可行——你不必为一台 7×24 空转的 VM 付钱。

## 6.3 后端选择：环境变量驱动的工厂

后端的创建与选择集中在 `tools/terminal_tool.py`。工厂函数是 `_create_environment(env_type, image, cwd, timeout, ...)`（约第 1199 行），它根据 `env_type` 实例化对应的后端类。

选哪个后端，由环境变量 `TERMINAL_ENV` 决定，默认 `"local"`（`terminal_tool.py` 约第 569、1067 行读取 `os.getenv("TERMINAL_ENV", "local")`）。Modal 还有一个子模式开关 `TERMINAL_MODAL_MODE`（默认 `"auto"`，经 `coerce_modal_mode(...)` 归一化，约第 1137 行），由 `_get_modal_backend_state()`（约第 1190 行）决定走"托管 Modal"还是"直连 Modal"。

值得一提的是，当 `TERMINAL_ENV` 是个无法识别的值时，代码会抛出一个清晰的错误（约第 2550 行）：`"Unknown TERMINAL_ENV '%s'. Use one of: local, docker, singularity, ..."`。这条错误信息直接告诉你"合法的取值有哪些"——又一个"错误即文档"的实例。配置写错时，用户立刻知道该怎么改，而不是对着一个 `KeyError` 发懵。**用环境变量切换整个执行后端、且把合法值写进错误信息**，让部署 Hermes 到不同基础设施只是改一个环境变量的事。

## 6.4 命令包裹：`_wrap_command` 的脚本工程

这一章最值得细读的，是 `base.py` 约第 417 行的 `_wrap_command(self, command, cwd)`。它把用户的一条命令包装成一段精心设计的 bash 脚本，解决了一连串"在隔离环境里反复执行命令"才会遇到的棘手问题。按源码，这段脚本依次做：

1. **恢复环境快照**：`source <snap> >/dev/null 2>&1 || true`——把上一次命令结束时保存的环境变量重新加载进来。`|| true` 保证即使快照不存在也不报错。
2. **切换工作目录**：`builtin cd -- <quoted_cwd> || exit 126`——用 `builtin cd` 确保用的是内建命令而非被改写的别名，`--` 防止目录名被当成选项，失败则以 126 退出。
3. **执行真正的命令**：`eval '<escaped>'`——用 `eval` 执行转义后的命令。
4. **捕获退出码**：`__hermes_ec=$?`——立刻把命令的退出码存进一个带前缀的私有变量，防止被后续操作覆盖。
5. **重新导出环境**：`export -p > <snap>`——把命令执行后的环境变量状态写回快照文件。
6. **回写工作目录与标记**：用 `pwd -P` 记录真实工作目录，写出一个 stdout 标记（`self._cwd_marker`）。
7. **以原命令退出码退出**：`exit $__hermes_ec`——保证包裹脚本的退出码就是原命令的退出码。

这套设计解决的核心痛点是**状态的跨命令持久化**。在一个隔离环境里，每次执行如果都是全新的 shell，那么 `cd` 到某个目录、`export` 一个环境变量，下一条命令就全忘了——用户会困惑"我明明 cd 过去了，怎么又回来了"。Hermes 通过"执行前 source 快照、执行后 dump 快照"的手法，让环境变量和工作目录能在一条条独立命令之间延续，模拟出"同一个会话"的体验。

转义处理也一丝不苟：命令里的单引号通过 `command.replace("'", "'\\''")` 转义（这是 shell 里嵌入单引号的经典手法），路径则用 `shlex.quote` 安全引用。**对每一个要拼进 shell 脚本的字符串都做正确转义**，是防止命令注入和语法破坏的基本功——而这恰恰是手写 shell 拼接最容易翻车的地方。

执行时，`execute()` 还会调用 `_rewrite_compound_background`（来自 `terminal_tool.py`）来修正像 `A && B &` 这种复合后台命令的语义。sudo 命令则由 `_prepare_command` → `_transform_sudo_command` 处理（用到第 5 章提过的 `SUDO_PASSWORD`）。

## 6.5 托管 Modal 的超时矩阵与输出截断

`managed_modal.py` 里有一组针对网络不稳定性精心设计的超时常量，全部可经 `_request_timeout_env` 用环境变量覆盖：`_CONNECT_TIMEOUT_SECONDS = 1.0`（连接超时，对应 `TERMINAL_MANAGED_MODAL_CONNECT_TIMEOUT_SECONDS`）、`_POLL_READ_TIMEOUT_SECONDS = 5.0`（轮询读取）、`_CANCEL_READ_TIMEOUT_SECONDS = 5.0`（取消读取）、`_client_timeout_grace_seconds = 10.0`（客户端超时宽限）。终态集合是 `{"completed", "failed", "cancelled", "timeout"}`。把不同阶段的超时拆成独立可调的常量，而不是用一个笼统的全局超时，是和远程服务打交道的细致之处——连接慢和执行慢是两回事，应该分别设限。

输出侧也有卫生措施（在 `terminal_tool.py`）：`FOREGROUND_MAX_TIMEOUT`（约第 107 行，可经环境变量解析）规定前台命令的最大超时，超过它的命令会被拒绝并提示改用 `background=true`——避免一条长命令把前台卡死。`MAX_OUTPUT_CHARS` 控制输出截断，且截断方式很讲究：**保留头部 40%、尾部 60%**（约第 2403 行）。为什么尾部留得更多？因为命令输出的尾部往往是结论（错误信息、最终结果），比中间的过程日志更重要。这又呼应了第 3 章"头尾信息密度高"的洞察。

此外，`tools/code_execution_tool.py`（约第 465 行）会剥离一组终端专属参数 `_TERMINAL_BLOCKED_PARAMS = {"background", "pty", "notify_on_complete", "watch_patterns"}`——因为代码执行场景不该使用这些终端交互特性。

## 6.6 Docker 构建产物：一瞥容器化

`tools/environments/docker.py` 只是后端的一面，另一面是真正的镜像构建。仓库根的 `Dockerfile` 用了多阶段构建，基础镜像包括 `uv:0.11.6-python3.13-trixie`、`node:22-bookworm-slim`、`debian:13.4`——Python、Node、以及最终的 Debian 运行层。`docker/` 目录里还有一套启动脚本：`entrypoint.sh`、`main-wrapper.sh`、`stage2-hook.sh`、`hermes-exec-shim.sh`，以及 `cont-init.d/` 和 `s6-rc.d/`（s6 是轻量的进程监督系统，用于容器内多进程管理）。配合根目录的 `docker-compose.yml` 与 `docker-compose.windows.yml`，Hermes 提供了开箱即用的容器化部署。这套容器基建本身就值得一本运维手册，本书不展开，只指出它的存在，让你知道"Docker 后端"背后有一整套精心维护的镜像工程在支撑。

## 本章小结

- Hermes 支持六种终端后端（local / Docker / SSH / Singularity / Modal / Daytona），同一 Agent 可把命令跑在本地或 serverless 云沙箱上。
- 所有后端被 `BaseEnvironment(ABC)` 统一，构造参数为 `cwd` / `timeout` / `env`；基类**只有一个抽象方法 `cleanup`**，通用的 `execute` 在基类实现，子类只需实现 `_run_bash`——这是六后端不爆炸的关键。
- 后端选择由 `TERMINAL_ENV`（默认 `local`）驱动，工厂 `_create_environment` 负责实例化；非法值会抛出把"合法取值"写进信息里的错误（错误即文档）；Modal 另有 `TERMINAL_MODAL_MODE` 子模式。
- `DaytonaEnvironment` 自己再包一层 shell 超时，因为 SDK 超时不可靠——体现"不轻信第三方 SDK 承诺"的生产姿态。
- `_wrap_command` 把命令包成 bash 脚本：source 环境快照 → `builtin cd` → `eval` 执行 → 捕获 `__hermes_ec` 退出码 → `export -p` 回写快照 → 记录 `pwd -P` 与 cwd 标记 → 以原退出码退出，从而让环境变量与工作目录跨命令持久化。
- 转义一丝不苟：单引号用 `'\\''` 经典手法、路径用 `shlex.quote`，防命令注入与语法破坏。
- 托管 Modal 把连接/轮询/取消/宽限超时拆成独立可调常量（`_CONNECT_TIMEOUT_SECONDS = 1.0` 等）；输出截断保留头 40%/尾 60%，因为尾部多为结论。
- Docker 后端背后是多阶段 `Dockerfile`（uv/Python、Node、Debian）与 `docker/` 下基于 s6 的启动脚本，构成完整的容器化部署基建。

## 动手实验

1. **实验一：数清抽象的最小契约** —— `Read` `tools/environments/base.py` 约第 288–343 行，确认 `BaseEnvironment` 究竟有几个 `@abstractmethod`。论证：为什么只把 `cleanup` 设为抽象、而把 `execute` 做成共享具体方法？这对"新增一个后端"的成本意味着什么？

2. **实验二：逐行翻译 `_wrap_command`** —— `Read` `base.py` 约第 417 行的 `_wrap_command`，把它生成的 bash 脚本逐行用中文注释。重点解释第 1 步的 `source <snap> || true` 和第 5 步的 `export -p > <snap>` 如何协作，使得 `cd` 和 `export` 能在多条独立命令之间保持。

3. **实验三：切换后端** —— 用 Grep 在 `tools/terminal_tool.py` 找到 `os.getenv("TERMINAL_ENV", "local")` 与那条 "Unknown TERMINAL_ENV" 错误（约第 2550 行）。列出全部合法取值，并描述：如果把 `TERMINAL_ENV` 设成一个拼错的值，用户会看到什么、该如何修正。

4. **实验四：理解输出截断策略** —— 用 Grep 找到 `MAX_OUTPUT_CHARS` 的截断逻辑（约第 2403 行），确认是"头 40% + 尾 60%"。构造一个会超长的命令输出（如打印一个大文件），推演哪部分会被保留、哪部分被丢弃，并解释为什么尾部权重更高。

> **下一章预告**：单个 Agent 跑在沙箱里只是起点。当任务需要并行推进多条工作流时，Hermes 会派出"子 Agent"。第 7 章将解构 `tools/delegate_tool.py` 的 `delegate_task` 工具——它如何携带目标、上下文、工具集与迭代上限派生出隔离的子 Agent，又如何把多个子任务并行化以"零上下文成本"折叠多步流水线。
