# 第 7 章　权限引擎：四种模式与五重门控

## 7.1　权限系统的三层架构

OpenWorker 的权限系统由三个组件构成：

1. **`risk.py`**：工具的内在风险分类（READ / WRITE_LOCAL / EXEC / EXTERNAL）-- 第 5 章已详述
2. **`permissions.py`**：`PermissionEngine` -- 基于模式 + 风险 + 允许列表做决策
3. **`engine.py` 的 `_authorize()`**：把决策结果路由到审批/执行

## 7.2　五种会话模式

```python
class Mode(str, Enum):
    DISCUSS = "discuss"      # 只读对话：不能编辑、不能 plan
    PLAN = "plan"            # 只读 + 计划工作流：探索 -> propose_plan -> 执行
    INTERACTIVE = "interactive"  # 默认：读取自动允许，写入/命令需审批
    AUTO = "auto"            # 全自动
    CUSTOM = "custom"         # interactive + 自动允许配置的 auto_allow 工具
```

### 模式的行为差异

| 行为 | DISCUSS | PLAN | INTERACTIVE | AUTO | CUSTOM |
|---|---|---|---|---|---|
| 读工具 | ✅ | ✅ | ✅ | ✅ | ✅ |
| 写本地文件 | ❌ | ❌ | 审批 | ✅ | 视配置 |
| 执行命令 | ❌ | ❌ | 审批 | ✅ | 审批 |
| 外部副作用 | ❌ | ❌ | 审批 | ✅ | 审批 |
| propose_plan | 无意义 | ✅ 出口 | 无意义 | 无意义 | 无意义 |

`READ_ONLY_MODES = frozenset({Mode.DISCUSS, Mode.PLAN})` -- 两种只读模式共享同一个门控，区别只在 intent。

## 7.3　决策流程：`evaluate()` 的五重门控

`PermissionEngine.evaluate()` 是所有工具调用的决策入口，按顺序检查五道门：

```python
def evaluate(self, tool_name, arguments, metadata=None) -> Decision:
    # 门 1: 只读模式拒绝 consequential
    if self.mode in READ_ONLY_MODES and consequential:
        return Decision(False, f"{self.mode.value} mode is read-only")

    # 门 2: 写操作的路径约束
    if is_write:
        path = arguments.get("path")
        if path is not None and not self._under_writable_root(path):
            return Decision(False, f"path is not in a writable directory: {path}")

    # 门 3: 非 consequential 工具始终允许
    if not consequential:
        return Decision(True, "low risk")

    # 门 4: AUTO 模式全开
    if self.mode is Mode.AUTO:
        return Decision(True, "full access")

    # 门 5: 允许列表 + standing rules + custom auto-allow
    # 5a: Shell 命令 allowlist
    if is_shell:
        if self._command_allowed(command): return Decision(True, "command on allowlist")
        if command in self.session_allow_commands: return Decision(True, "command allowed for session")
    # 5b: 工具 session allowlist（排除 connector）
    if tool_name in self.session_allow_tools and not is_connector:
        return Decision(True, "tool allowed for session")
    # 5c: Task-scoped standing rules（§25）
    if tool_name in self.task_rules:
        target = standing_rule_candidate(...)
        if target and target in self.task_rules[tool_name]:
            return Decision(True, f"allowed by standing rule: {rule}")
    # 5d: Custom 模式的 auto_allow
    if self.mode is Mode.CUSTOM and tool_name in self.auto_allow_tools:
        return Decision(True, "auto-allowed by config")

    # 默认：需要用户审批
    return Decision(False, "requires approval", needs_user=True)
```

## 7.4　Decision 数据类

```python
@dataclass
class Decision:
    allowed: bool
    reason: str = ""
    needs_user: bool = False  # True -> 触发审批 prompt
    rule: str = ""  # standing rule 的来源标记
```

`needs_user=True` 是关键信号：TurnEngine 的 `_authorize()` 看到它就发出 `PERMISSION_REQUIRED` 事件，然后 `await self.approver(...)` 挂起等待用户回答。

## 7.5　ApprovalOutcome：用户回答的四种结果

```python
class ApprovalOutcome(str, Enum):
    ONCE = "once"            # 允许这一次
    ALWAYS_TOOL = "always_tool"    # 这个 session 里这个工具总是允许
    ALWAYS_COMMAND = "always_command"  # 这个 session 里这个命令总是允许
    DENY = "deny"            # 拒绝
```

当用户选择"Always allow"时：
- `ALWAYS_TOOL`：调用 `permissions.allow_tool_for_session(tool_name)` 加入 `session_allow_tools`
- `ALWAYS_COMMAND`：调用 `permissions.allow_command_for_session(command)` 加入 `session_allow_commands`

这些 session 级别的允许列表在会话存续期间有效，并在 `SessionRecord.grants` 中持久化（重启后恢复）。

## 7.6　Standing Rules：自动化场景的精确授权

`standing_rule_candidate()` 函数为"Always allow"按钮计算可绑定的精确目标：

```python
def standing_rule_candidate(tool_name, arguments, metadata, overrides) -> Optional[str]:
    if classify(tool_name, metadata, overrides) is not RiskClass.EXTERNAL:
        return None  # 只有 external 风险才 eligible
    arg = target_arg_for(tool_name)
    if arg is None:
        return None
    value = str(arguments.get(arg) or "").strip()
    return value or None
```

**设计意图**：Shell 工具永远不 eligible（每次都要审批），连接器工具可以绑定到精确目标（如"允许给 #general 频道发消息"）。这是自动化场景的安全关键--一个无人值守的 Agent 不应该被 blanket-allow 到所有外部操作。

## 7.7　Shell 命令 Allowlist 的安全设计

`_command_allowed()` 有三层保护：

1. **操作符黑名单**：`_has_shell_operators(command)` 检查 `; & | > < \` $( ( \n \r`--有任何一个就拒绝
2. **Token 级前缀匹配**：用 `shlex.split` 解析命令和 allowlist 条目为 argv 数组，要求 allowlist 条目的 argv 是命令 argv 的精确前缀
3. **空命令拒绝**：`shlex.split` 失败或返回空列表时拒绝

这防止了所有已知的命令注入向量：
- `git status && rm -rf ~` -- `&&` 被操作符黑名单拦截
- `git status$(rm -rf ~)` -- `$(` 被拦截
- `git status; rm -rf ~` -- `;` 被拦截

## 7.8　plan 模式的特殊工作流

plan 模式不只是"只读"，它驱动一个特定的工作流：

1. Agent 探索代码（读工具可用）
2. Agent 调用 `propose_plan(plan="...")` 提出方案
3. TurnEngine 发出 `PLAN_PROPOSED` 事件
4. `plan_approver` 回调让用户审批
5. **批准**：`PermissionEngine.mode` 切换到 `INTERACTIVE` 或 `AUTO`（由审批结果决定），同一会话继续执行
6. **拒绝**：保持 plan 模式，用户反馈作为工具结果返回给 Agent

```python
# engine.py: _handle_plan_proposal()
if result.get("approved"):
    self.permissions.mode = Mode(str(result.get("mode", "interactive")))
```

**精妙之处**：模式切换是 in-session 的，Agent 的探索上下文（已经读过的文件、已经搜过的代码）全部保留。不需要开新会话。

## 本章小结

OpenWorker 的权限引擎用"四模式 + 五门控 + 四种审批结果"的矩阵，实现了细粒度的工具安全控制。最值得借鉴的设计是：standing rules 的精确目标绑定（让自动化的"always allow"不再危险），Shell 命令的三层防护（操作符黑名单 + token 级匹配 + 空命令拒绝），以及 plan 模式的 in-session 切换（保留上下文的同时解锁执行权限）。

## 动手实验

1. 在 `permissions.py` 中找到 `evaluate()` 方法，追踪一个 `run_shell(command="ls -la")` 在 INTERACTIVE 模式下的完整决策路径。
2. 修改一个会话的模式为 DISCUSS，尝试让 Agent 写文件，观察被拒绝的理由。
3. 在 `risk.py` 中找到 `standing_rule_candidate()`，思考为什么 EXEC 风险的工具不 eligible。
4. 找到 `_SHELL_OPERATORS` 元组，构造 5 个会被拦截的命令字符串，验证每个被哪个操作符触发。
