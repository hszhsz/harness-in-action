# 第 5 章　工具注册与风险分级：四档分类法

## 5.1　ToolRegistry：最小的工具注册器

`coworker/tools/registry.py` 的 `ToolRegistry` 是 OpenWorker 工具系统的入口，设计极简：

```python
@dataclass
class ToolSpec:
    name: str
    schema: dict[str, Any]  # OpenAI 格式的 function tool schema
    func: Callable[..., Any]
    metadata: Any = None  # aisuite ToolMetadata 或 None

class ToolRegistry:
    def register(self, func, *, metadata=None, schema=None) -> ToolSpec: ...
    def schemas(self) -> list[dict]: return [spec.schema for spec in self._tools.values()]
    def execute(self, name, arguments) -> Any: return spec.func(**arguments)
```

**关键决策：schema 生成复用 aisuite**。`_schema_for(func)` 调用 `Tools([func]).tools(format="openai")[0]`，从函数签名和 docstring 自动提取 JSON schema。这避免了手写 schema 的维护负担。

**权限不在 registry 里**。`ToolRegistry` 只管"注册、查询、执行"三个操作，权限检查在 `PermissionEngine` 里，由 `TurnEngine._authorize()` 调用。这种分离让 registry 可以被独立测试，也让权限策略可以独立演进。

## 5.2　RiskClass：工具的内在风险分类

`coworker/risk.py` 是 OpenWorker 工具安全的核心抽象。每个工具被归入四档之一：

```python
class RiskClass(str, Enum):
    READ = "read"          # 无副作用，始终允许
    WRITE_LOCAL = "write_local"  # 修改工作区文件，受路径约束 + 模式门控
    EXEC = "exec"          # 执行命令，受模式门控
    EXTERNAL = "external"  # 对外部世界有副作用，是无人值守 Inbox 的路由依据
```

### 分类逻辑

```python
def classify(tool_name, metadata=None, overrides=None) -> RiskClass:
    if overrides is not None:
        ov = overrides(tool_name)
        if ov is not None:
            return ov
    base = _BASE.get(tool_name)
    if base is not None:
        return base
    if bool(getattr(metadata, "requires_approval", False)):
        return RiskClass.EXTERNAL
    return RiskClass.READ
```

三层优先级：
1. **用户覆盖**（`overrides`，Phase 2 特性，主要用来放松 MCP 的保守默认）
2. **按名字的内置表**（`WRITE_TOOLS = {"write_file", "replace_in_file", "apply_patch", "apply_unified_diff"}` 和 `SHELL_TOOL = "run_shell"`）
3. **aisuite metadata**（`requires_approval` -> EXTERNAL）
4. **默认 READ**

`is_consequential(risk)` 是一个简单的判断：`risk is not RiskClass.READ`。所有非 READ 的风险都需要权限引擎的关注。

## 5.3　从 Agent 到工具注册

`agent.py` 的 `build_engine()` 是工具注册的编排中心，按顺序注册：

1. **Agent 内置工具**：`agent.build_tools(context)` -- 根据 agent 类型不同（code/chat/cowork/myhelper）注册不同的基础工具
2. **extra_tools**（MCP / 连接器工具，由 manager 传入）
3. **send_message / send_file**（如果 agent 是 messaging 类型且有配置好的连接器）
4. **subscription_tools**（频道订阅，如果提供了 subscription_store）
5. **request_directory**（如果 agent 是 knowledge 类型且有多根工作区）
6. **connector 集成工具**：`make_integration_tools(secrets, ...)` -- 25+ 连接器的工具函数
7. **web_search / web_fetch**：研究工具（每个 agent 都注册）
8. **ask_user**：人机交互原语（如果提供了 question_asker 回调）
9. **explorer_tools**（如果 agent 是 code 类型，注册只读子 Agent 工具）
10. **scheduling_tools**（如果提供了 task_store 且 agent 是 knowledge 类型）
11. **selfwake_tools**（如果提供了 wake_store 且 agent 是 knowledge 类型）
12. **memory_tools**（如果提供了 memory_store）
13. **skill_tools**（渐进式披露的 Skill）
14. **propose_plan**（始终注册，plan 模式的出口）

## 5.4　metadata 驱动的行为

`ToolSpec.metadata`（aisuite 的 `ToolMetadata`）携带以下字段，驱动多个引擎行为：

- `risk_level: str` -> 低风险的工具可以并发执行（`_parallel_safe()`）
- `requires_approval: bool` -> 没有内置分类时，映射到 `EXTERNAL` 风险
- `category: str` -> `"connector"` 标记的工具有特殊的 session-allow 排除逻辑
- `target_arg` -> 用于 standing rule 候选（§25：允许"always allow this tool for this target"）

## 5.5　Shell 操作符黑名单

权限引擎中的 `_SHELL_OPERATORS` 是一个常量列表：

```python
_SHELL_OPERATORS = (";", "&", "|", ">", "<", "`", "$(", "(", "\n", "\r")
```

任何包含这些字符的命令都**不能**通过 allowlist 自动运行。这个设计防止了"`git status` 允许了 `git status && rm -rf ~`"这类命令注入。`_command_allowed()` 用 `shlex.split` 做 token 级前缀匹配，而非字符串前缀匹配。

## 5.6　连接器工具的注册

`connectors/tools.py` 的 `make_integration_tools()` 为每个 enabled 连接器创建工具函数。这些工具的 metadata 携带 `category="connector"` 和连接器特定的 risk 信息。

连接器工具还有一个特殊的 `_display` 机制：工具结果可以携带一个 `_display` 字典，包含用户可见但模型不可见的元数据（如隐私过滤器隐藏了多少结果）。这个字段在 `_record_result()` 中被从结果中剥离，作为 sidecar 挂在消息上，在 `_outbound_messages()` 中被剥离。

## 本章小结

OpenWorker 的工具系统可以用三层来概括：**Registry（注册）-> RiskClass（分类）-> PermissionEngine（决策）**。最值得借鉴的是四档风险分类法--它把工具安全的决策从"按名字特判"变成了"按内在副作用分类"，并用 `overrides` 机制提供了用户级别的覆盖能力。Shell 操作符黑名单和 token 级前缀匹配是命令安全的微观最佳实践。

## 动手实验

1. 在 `risk.py` 中找到 `_BASE` 字典，列出所有内置分类的工具名。
2. 打开 `agent.py` 的 `build_engine()`，数一下注册了多少类工具。
3. 在 `permissions.py` 中找到 `_has_shell_operators()` 函数，测试几个命令字符串看是否被正确拦截。
4. 思考：为什么连接器工具有 `category="connector"` 的标记？在 `_authorize()` 中搜索这个标记，看它影响了什么逻辑。
