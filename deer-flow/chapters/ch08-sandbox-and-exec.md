# 第 8 章　沙箱与命令执行：Local vs Aio 与 fail-closed host bash

工具系统里最危险的一类，是「执行命令、读写文件」——它直接动宿主机的文件系统和进程。一个不设防的 agent，一条 `rm -rf /` 或一句 `cat ~/.ssh/id_rsa` 就能闯大祸。这一章我们钻进 `sandbox/` 分区，看 DeerFlow 用什么隔离边界约束这些副作用、为什么当部署用「不安全的本地沙箱」时它默认禁掉 host bash 又留了哪个显式开关、以及那些把文件操作钉死在虚拟路径里的层层防御。这是「默认即关闭」原则最密集的一章。

## 两层抽象：SandboxProvider 与 Sandbox

DeerFlow 把沙箱拆成两个抽象基类。`SandboxProvider`（`sandbox/sandbox_provider.py`）管「环境的生命周期」——`acquire`（拿一个沙箱、返回 id）、`get`、`release`。`Sandbox`（`sandbox/sandbox.py`）管「在一个沙箱里干活」——`execute_command`、`read_file`、`write_file`、`download_file`、`list_dir`、`glob`、`grep`、`update_file`。这套分层让「用哪种隔离」和「怎么操作」解耦。

provider 同样是配置驱动的——`get_sandbox_provider` 用 `resolve_class(config.sandbox.use, SandboxProvider)` 从配置一行 `use:` 解析出 provider 类，和模型、工具如出一辙。两个内置实现：`LocalSandboxProvider`（直接在宿主机上跑，靠路径映射做约束）和 `AioSandboxProvider`（`community/aio_sandbox`，真正的隔离沙箱）。

`SandboxProvider` 还有个体贴的细节 `acquire_async`：大多数 provider 的生命周期 API 是同步阻塞的（本地 Docker/provisioner 操作天生阻塞），所以异步运行时应调 `acquire_async`，它用 `asyncio.to_thread(self.acquire, ...)` 把阻塞操作丢进工作线程，不卡事件循环。这呼应第 5 章 `SandboxMiddleware(lazy_init=True)` 的懒初始化——沙箱不在装配时创建，真要用时才拉起，而且拉起的过程不阻塞主循环。

provider 是个带缓存的单例（`get_sandbox_provider` 返回缓存实例），`reset_sandbox_provider` / `shutdown_sandbox_provider` 分别负责「清缓存」和「优雅关闭释放所有沙箱」。`reset` 的注释专门解释为什么 provider 要能 override 它——`LocalSandboxProvider` 缓存了一个 `LocalSandbox` 单例，不 reset 的话配置/挂载变更在下一次 acquire 时不生效。

## fail-closed 的核心：host bash 默认禁

这一章最重要的一段代码，在 `sandbox/security.py`。它定义了一条判断：「当前用的是本地沙箱 provider 吗？」「host bash 被允许了吗？」

```python
def is_host_bash_allowed(config=None) -> bool:
    sandbox_cfg = getattr(config, "sandbox", None)
    if sandbox_cfg is None:
        return False
    if not uses_local_sandbox_provider(config):
        return True
    return bool(getattr(sandbox_cfg, "allow_host_bash", False))
```

逻辑读出来是：**如果用的不是本地 provider（即真正隔离的 Aio 沙箱），host bash 默认允许；如果用的是本地 provider，则默认禁止，除非用户显式设了 `sandbox.allow_host_bash: true`**。注意 `sandbox_cfg is None` 时直接返回 False——连「没配沙箱」都按「不允许」处理。这是教科书级的 fail-closed：安全边界缺失时，默认走最严的那条路。

配套的 `LOCAL_HOST_BASH_DISABLED_MESSAGE` 把「为什么禁、怎么解」写成一句完整的人话：

> Host bash execution is disabled for LocalSandboxProvider because it is not a secure sandbox boundary. Switch to AioSandboxProvider for isolated bash access, or set `sandbox.allow_host_bash: true` only in a fully trusted local environment.

它先说「为什么禁」（本地 provider 不是安全边界），再给两条出路（换 Aio，或在完全可信环境下显式开开关）。这正是「错误即文档」——错误信息本身就是修复指南，连「只在完全可信环境下开」这种使用边界都点明了。

这个判断在多处被消费，构成一张前后一致的防护网：第 7 章 `get_available_tools` 用它把 host-bash 工具从给模型的工具列表里剔掉；`task_tool.py`（子 agent）和 `subagents/registry.py` 用它在「bash 子 agent」被请求时返回 `LOCAL_BASH_SUBAGENT_DISABLED_MESSAGE` 而非真的拉起；`sandbox/tools.py` 在执行层再兜一道。**同一条安全策略在工具暴露层、子 agent 层、执行层各拦一次**——纵深防御，不靠单点。

## 虚拟路径：把文件操作钉进笼子

`LocalSandbox` 直接在宿主机跑，没有内核级隔离，那它靠什么不让 agent 乱摸文件？答案是**路径映射 + 虚拟路径前缀**。`LocalSandbox` 持有一组 `PathMapping`（`container_path` → `local_path`，带 `read_only` 标志），所有路径操作都先经过 `_resolve_path_with_mapping` 把「容器视角的虚拟路径」翻译成「宿主机真实路径」，并在翻译时做逃逸检查：

```python
try:
    resolved_path.relative_to(local_root)
except ValueError as exc:
    raise PermissionError(errno.EACCES, "Access denied: path escapes mounted directory", path_str) from exc
```

用 `Path.resolve()` 把 `..` 之类的相对跳转折叠掉后，再确认结果仍在挂载根目录之内——一旦 `relative_to` 抛 `ValueError`，说明路径逃出了笼子，直接拒。`download_file` 还加了一道更硬的前缀检查：路径必须在 `VIRTUAL_PATH_PREFIX`（即 `/mnt/user-data`）之下，否则 `Refused download outside allowed directory`，并配 100MB 的下载上限。

几个考究的细节值得记：

- **错误信息隐藏内部路径**：`read_file`/`download_file` 捕获 `OSError` 后用**原始 path**（虚拟路径）重新抛，而不是泄露内部 resolved 的宿主机真实路径——既给用户清晰报错，又不暴露内部目录结构。这是第 11 章「诊断导出的隐私」主题的预演。
- **只对 agent 写过的文件做反向解析**：`read_file` 只在 `resolved_path in self._agent_written_paths` 时才把内容里的真实路径反解回虚拟路径。注释引用 PR #1935：用户上传的文件、外部工具输出不应被「悄悄改写」，只有 agent 自己写的内容才该做这层路径回译。边界感极强——不该碰的内容一个字都不改。
- **read-only 挂载选最具体的映射**：`_is_read_only_path` 在多个映射嵌套命中时，选 `local_path` 最长前缀的那个（最具体的挂载），避免父挂载的 read-only 标志误伤子挂载。skills 目录默认 read-only。
- **TOCTOU 自觉**：`download_file` 注释承认 `getsize()` 和 `read()` 之间文件可能变大（TOCTOU），并说明「在受控沙箱环境里这是可接受的权衡」——把已知的弱点摊开写在注释里，而不是假装它不存在。

## 命令里的路径也要查

光约束文件 API 还不够——`execute_command` 跑的是任意 bash 命令，命令字符串里可以塞绝对路径。所以 DeerFlow 在执行前对命令做路径校验。`validate_local_bash_command_paths`（`sandbox/tools.py`）扫描命令里的绝对路径，凡是不在允许范围（`/mnt/user-data`、`/mnt/skills`、`/mnt/acp-workspace`、配置的自定义挂载、以及一个小的系统路径白名单如 `/bin/sh`、`/dev/null`）内的，一律 `Unsafe absolute paths in command`。它还专门堵 `file://` URL——因为这种写法能绕过绝对路径正则，却能读本地文件搞数据外泄。`cd` 到笼外目录、命令替换里的危险路径，也各有专门的 `PermissionError`。

但 DeerFlow 对这层防御的定位**异常诚实**。`validate_local_bash_command_paths` 的文档字符串开宗明义：

> This validation is only a best-effort guard for the explicit `sandbox.allow_host_bash: true` opt-in. It is not a secure sandbox boundary and must not be treated as isolation from the host filesystem.

也就是说——**即使你开了 host bash，这套路径校验也只是「尽力而为的护栏」，绝不是真正的隔离边界**。想要真隔离，请用 `AioSandboxProvider`。这种「不把自己的防御吹成银弹」的克制，比任何安全宣传都更可信。`execute_command` 本身用 `subprocess.run(..., shell=False, timeout=600)`——`shell=False` 避免 shell 注入，600 秒超时防止命令挂死。

## 进出命令的路径双向翻译

`execute_command` 还做了一件让 agent 体验顺滑的事：执行前用 `_resolve_paths_in_command` 把命令里的虚拟路径换成真实路径，执行后用 `_reverse_resolve_paths_in_output` 把输出里的真实路径换回虚拟路径。这样模型始终活在「虚拟路径」这套干净的坐标系里，永远看不到宿主机的真实目录结构——既是体验，也是安全(不泄露内部布局)。

## 本章小结

- 沙箱分两层抽象：`SandboxProvider`（生命周期 acquire/get/release，配置驱动经 `resolve_class`）与 `Sandbox`（操作 execute_command/read/write/glob/grep 等）；provider 是带缓存单例，`acquire_async` 用 `asyncio.to_thread` 不阻塞事件循环。
- 两个内置 provider：`LocalSandboxProvider`（宿主机直跑、靠路径映射约束、非安全边界）与 `AioSandboxProvider`（真正隔离）。
- 核心 fail-closed：`is_host_bash_allowed` 在本地 provider 下默认禁 host bash，仅 `sandbox.allow_host_bash: true` 显式开启；`sandbox_cfg is None` 也返回 False。该策略在工具暴露层、子 agent 层、执行层各拦一次（纵深防御）。
- 禁用消息 `LOCAL_HOST_BASH_DISABLED_MESSAGE` / `LOCAL_BASH_SUBAGENT_DISABLED_MESSAGE` 同时给出原因和两条出路，是「错误即文档」的范例。
- `LocalSandbox` 用 `PathMapping` + `VIRTUAL_PATH_PREFIX`（`/mnt/user-data`）把文件操作钉进笼子：`_resolve_path_with_mapping` 用 `Path.resolve()` + `relative_to` 防逃逸，`download_file` 强制前缀且限 100MB；错误只暴露虚拟路径不泄露真实路径；只对 `_agent_written_paths` 做反向解析（PR #1935）。
- 命令执行前 `validate_local_bash_command_paths` 扫绝对路径、堵 `file://`、查 `cd`/命令替换；但文档明确声明它「只是 best-effort 护栏，不是安全隔离边界」——要真隔离用 Aio。`execute_command` 用 `shell=False` + 600s 超时，并对命令做虚拟↔真实路径双向翻译。

## 动手实验

1. **实验一：推演 host bash 门禁** — 阅读 `sandbox/security.py` 的 `is_host_bash_allowed` 与 `uses_local_sandbox_provider`，列出 4 种组合（本地/非本地 provider × allow_host_bash 真/假）各返回什么。再加上「`sandbox` 配置缺失」这一例，确认 fail-closed 默认走哪条。
2. **实验二：找出三道防线** — 用 `grep -rn "is_host_bash_allowed"` 在 harness 包里找到所有调用点，确认 host bash 策略分别在「工具暴露」（`tools/tools.py`）、「子 agent」（`task_tool.py` / `subagents/registry.py`）、「执行层」（`sandbox/tools.py`）各拦了一次。为什么单点拦截不够？
3. **实验三：构造一次路径逃逸** — 阅读 `_resolve_path_with_mapping`，给定一个挂载 `container_path=/mnt/user-data`、`local_path=/data/u1`，推演访问 `/mnt/user-data/../../etc/passwd` 时 `Path.resolve()` + `relative_to` 在哪一步抛 `PermissionError`。
4. **实验四：评估 best-effort 护栏的边界** — 通读 `validate_local_bash_command_paths` 的 docstring 与实现，回答：它能挡住 `cat /etc/passwd` 吗？能挡住 `cat $(echo /etc/passwd)` 吗？为什么 DeerFlow 仍坚持说它「不是安全隔离边界」？要获得真隔离应改用哪个 provider？

> **下一章预告**：副作用被关进笼子了，但还有一个会悄悄膨胀、最终拖垮一切的东西——上下文。下一章我们进入上下文工程的第一站，看 `SummarizationMiddleware` 如何在上下文撑爆前把它压缩、压缩时优先牺牲谁、又为什么要给 skill 的内容留一笔专门的保留预算。
