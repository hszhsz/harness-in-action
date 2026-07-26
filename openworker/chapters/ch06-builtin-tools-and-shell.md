# 第 6 章　内置工具与 Shell 执行：本地能力的边界

## 6.1　内置工具清单

OpenWorker 的 `coworker/tools/` 目录包含以下内置工具：

| 文件 | 工具 | 功能 |
|---|---|---|
| `shell.py` | `run_shell` | 本地命令执行（通过 `LocalExecutor`） |
| `files.py` | `read_file` / `write_file` / `replace_in_file` / `apply_patch` / `apply_unified_diff` | 文件读写与补丁 |
| `directories.py` | `request_directory` | 请求用户授予一个文件夹 |
| `plan.py` | `propose_plan` | 计划模式：提出方案等用户审批 |
| `ask.py` | `ask_user` | 人机交互问答 |
| `todo.py` | `TodoList` | 任务清单管理 |
| `git.py` | git 相关工具 | Git 操作 |
| `search.py` | 搜索工具 | 本地搜索 |
| `subagent.py` | `explore` | 只读子 Agent 探索 |

## 6.2　LocalExecutor：Shell 执行的护栏

`tools/shell.py` 中的 `LocalExecutor` 是 `run_shell` 工具的核心。它的设计要点：

### 中断支持

```python
class LocalExecutor:
    def interrupt_now(self) -> None:
        """Kill the running foreground command."""
```

`LocalExecutor` 的 `interrupt_now()` 方法被注册为 TurnEngine 的 `interrupt_hooks` 之一（在 `build_engine()` 中：`interrupt_hooks=[executor.interrupt_now]`）。当用户按 Stop 时，不仅 TurnEngine 的循环停止，正在运行的 shell 命令也被实际 kill。

### 工作目录绑定

`LocalExecutor(cwd=ws)` 绑定到会话的工作目录。所有 shell 命令都在这个目录下执行。

## 6.3　文件工具的安全边界

文件工具（`write_file`、`replace_in_file`、`apply_patch`、`apply_unified_diff`）被分类为 `RiskClass.WRITE_LOCAL`。权限引擎对它们的额外约束是**路径必须在一个可写根目录下**：

```python
# permissions.py
if is_write:
    path = arguments.get("path")
    if path is not None and not self._under_writable_root(path):
        return Decision(False, f"path is not in a writable directory: {path}")
```

`_under_writable_root()` 的实现：

```python
def _under_writable_root(self, path: str) -> bool:
    candidate = self._candidate(path)
    for rp, writable in self._resolved_roots():
        if not writable:
            continue
        try:
            candidate.relative_to(rp)
            return True
        except ValueError:
            continue
    return False
```

这保证了 Agent 不能写到工作区以外的目录--即使用户批准了某个写入操作，路径也必须落在已声明的根目录下。

## 6.4　多根目录系统

OpenWorker 支持多根目录（`roots.py` 中的 `RootDir`）。一个会话可以有多个根：

```python
@dataclass
class RootDir:
    path: Path
    writable: bool
    label: str = ""
```

- **主工作区**：会话启动时指定的目录，始终可写
- **额外根**：用户在会话中通过 `request_directory` 工具授予的目录，可以是可写或只读
- **会话持久化**：额外根被存在 `SessionRecord.extra_roots` 中，重载后恢复

`_resolved_roots()` 在每次权限检查时重新解析根列表，所以运行时添加/移除文件夹**不需要重建 PermissionEngine**。

## 6.5　WorkspaceTrustStore：信任边界

`workspace_trust.py` 管理工作区的信任状态。一个工作区被标记为"trusted"后，它的配置（如 `allowed_commands`）才会被加载。这防止了在一个不信任的代码仓库中自动执行危险命令。

```python
class WorkspaceTrustStore:
    def is_trusted(self, path: Path) -> bool: ...
    def set_trusted(self, path: Path, trusted: bool) -> None: ...
```

在 `build_engine()` 中：

```python
workspace_trusted = bool(ws and WorkspaceTrustStore().is_trusted(ws))
config = load_config(ws, workspace_trusted=workspace_trusted)
```

## 6.6　request_directory：动态扩展工作区

`request_directory` 是一个特殊的工具：它不执行任何操作，而是**请求用户授予一个新的文件夹**。这个工具被 `TurnEngine._handle_directory_request()` 拦截：

1. 发出 `DIRECTORY_REQUESTED` 事件
2. 等待用户通过 `directory_requester` 回调授权
3. 授权结果包含路径和是否可写
4. 授权后，文件夹被添加到会话的 `roots` 列表中
5. 下一次权限检查就能通过这个文件夹的路径

这个工具只对 `knowledge` family 的 Agent 注册（`if agent.family == "knowledge" and root_list:`），因为它们是面向知识工作的，可能需要中途访问其他目录。

## 6.7　ask_user：人机交互原语

`ask_user` 工具把一个自然语言问题变成一个 Inbox item：

1. Agent 调用 `ask_user(question="Which PR should I review first?")`
2. TurnEngine 发出 `QUESTION_REQUESTED` 事件
3. `question_asker` 回调创建一个 Inbox item（kind=`question`）
4. 在 attended 模式下，问题内联显示在会话中
5. 在 unattended 模式下，问题进入跨会话 Inbox 队列
6. 用户回答后，答案作为工具结果返回给 Agent

这个工具的设计和 `request_directory` / `propose_plan` 一样：**把人机交互变成工具调用**，让 Agent 的主循环保持统一。

## 本章小结

OpenWorker 的内置工具围绕"本地能力"展开：Shell 执行有中断支持和目录绑定，文件工具有路径约束和多根目录管理，workspace trust 管理信任边界。三个特殊工具（`request_directory`、`propose_plan`、`ask_user`）把人机交互变成了工具调用，让 TurnEngine 的循环保持统一。

## 动手实验

1. 在 `tools/shell.py` 中找到 `LocalExecutor` 类，看它如何管理子进程的生命周期。
2. 在 `permissions.py` 中找到 `_under_writable_root()`，追踪一个 `write_file(path="/etc/passwd")` 调用是如何被拒绝的。
3. 在 `agent.py` 中找到 `request_directory` 工具的注册条件，思考为什么只对 knowledge family 注册。
4. 修改一个会话的 `extra_roots`，在重启后验证文件夹是否被恢复。
