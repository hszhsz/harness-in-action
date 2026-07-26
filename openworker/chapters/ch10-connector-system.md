# 第 10 章　连接器系统：25+ 集成的统一抽象

## 10.1　连接器的角色

OpenWorker 的连接器（connector）是它与外部 SaaS 平台的桥梁。README 列出了 25+ 集成：Slack、GitHub、Jira、Notion、Linear、HubSpot、Outlook、monday.com、Gmail、Google Calendar 等。每个连接器既是**消息平台**（可以收发消息）又是**工具提供者**（可以执行 API 操作）。

## 10.2　BasePlatformAdapter：平台无关的连接器契约

`connectors/base.py` 定义了所有连接器的 ABC：

```python
class BasePlatformAdapter(ABC):
    platform: str = "base"

    @abstractmethod
    async def connect(self) -> bool: ...

    @abstractmethod
    async def disconnect(self) -> None: ...

    @abstractmethod
    async def send(self, chat_id: str, text: str, *, thread_id=None) -> SendResult: ...

    async def handle_message(self, event: MessageEvent) -> None:
        if self._handler is not None:
            await self._handler(event)
```

### 核心数据类型

**SessionSource**：消息的来源标识

```python
@dataclass
class SessionSource:
    platform: str       # "slack", "telegram", ...
    chat_id: str        # 平台内的聊天 ID
    user_id: str        # 发送者 ID
    chat_type: str = "dm"  # dm / group / channel
    thread_id: str      # 线程 ID（Slack）
    team_id: str        # 工作区 ID（多工作区 Slack）
```

**MessageSource**：消息的显示 sidecar

```python
@dataclass
class MessageSource:
    connector: str      # 平台 ID
    kind: str            # channel / dm
    channel_id: str
    channel_name: str    # 解析后的显示名
    sender_id: str
    sender_name: str     # 解析后的显示名
    ts: float            # 时间戳
    text: str            # RAW 原文
```

`MessageSource` 被附加到 user 消息上作为 `source` 字段，但 `_outbound_messages()` 在发送给 provider 之前会剥离它。这是和 OpenClaw 的 `MessageSource` 类似的设计。

## 10.3　target token：统一的外部回复地址

```python
def format_target(platform: str, chat_id: str, thread_id: Optional[str] = None) -> str:
    base = f"{platform}:{chat_id}"
    return f"{base}:{thread_id}" if thread_id else base
```

`target` token（如 `slack:C0BD7KZ1AH5:P123456`）是连接器系统对外的统一地址。Agent 不需要知道每个平台的地址格式，只需要在 `send_message(target=...)` 中传入这个 token。

## 10.4　ConnectorDescriptor：声明式连接器配置

和 ProviderDescriptor 类似，每个连接器有一个声明式描述符：

```python
@dataclass(frozen=True)
class ConnectorDescriptor:
    name: str           # "slack"
    title: str          # "Slack"
    fields: list[Field]  # 配置字段（token、bot_token 等）
    tools: list[ToolDef] # 提供的工具
    managed: bool       # 是否支持 OAuth 托管登录
    mcp_url: str        # MCP 连接路径（如果支持）
```

`connectors/descriptors.py` 中注册了所有 25+ 连接器。GUI 用 `to_dict()` 方法动态渲染配置界面。

## 10.5　连接器工具的生成

`connectors/tools.py` 的 `make_integration_tools()` 为每个 enabled 连接器创建工具函数：

```python
def make_integration_tools(secrets, *, enabled_connectors, enabled_tools, roots=None):
    tools = []
    for name in enabled_connectors:
        descriptor = get_descriptor(name)
        for tool_def in descriptor.tools:
            if tool_def.name in enabled_tools:
                func = _make_tool_func(name, tool_def, secrets, roots)
                tools.append(func)
    return tools
```

每个连接器工具的 metadata 携带 `category="connector"`，这影响了 session_allow_tools 的逻辑（连接器工具有单独的 standing rule 路径）。

## 10.6　多账号支持

某些连接器支持多账号（Gmail、Google Calendar、HubSpot、Slack 多工作区）：

- `gmail_accounts.py`：每个邮箱是独立的 profile（`gmail:account:user@gmail.com`）
- `gcal_accounts.py`：每个日历账户独立
- `hubspot_portals.py`：每个 portal 独立
- `slack_addr.py`：每个 team 独立

每个多账号连接器有一个 "default" 指针，指向当前活跃的账号。API 提供了 `set_default` 和 `disconnect_account` 端点。

## 10.7　隐私过滤器

Gmail 和 HubSpot 连接器有隐私过滤器：

- Gmail：`gmail_accounts.set_filters()` 设置 "Never show agents" 的 sender 列表和 label 列表
- HubSpot：`hubspot_portals.set_hidden_fields()` 设置要从结果中剥离的字段列表

过滤是**在工具层**做的：工具执行后，结果先被过滤（移除被隐藏的条目/字段），然后被剥离的结果数（`hidden_by_filters`、`hidden_fields`）作为 `_display` sidecar 挂在消息上。模型看到的是过滤后的结果，用户在 tool card 上看到"3 result(s) hidden by privacy filters"。

```python
# engine.py: _record_result()
if isinstance(result, dict) and "_display" in result:
    display = result.get("_display")
    result = {k: v for k, v in result.items() if k != "_display"}
message = _tool_result_message(tool_call, result)
if display:
    message["_display"] = display
```

## 10.8　Slack/GitHub 托管中继

OpenWorker Cloud 提供了一种托管中继模式（managed relay），用于 Slack 和 GitHub：

- **Slack**：不需要用户自己配置 Socket Mode，云中继代为监听消息并转发到本地
- **GitHub**：通过 GitHub App installation，云中继把 webhook 事件转发到本地

这解决了"桌面应用不能 7×24 监听 webhook"的问题。`github_relay.py` 和 `slack_directory.py` 管理这些中继连接。

## 本章小结

OpenWorker 的连接器系统用 "BasePlatformAdapter ABC + Descriptor 声明 + 工具自动生成" 的三层架构，把 25+ SaaS 集成统一到一个框架下。最值得借鉴的是：target token 统一了外部回复地址、`_display` sidecar 让隐私过滤器对模型透明、多账号支持让一个用户管理多个 Slack 工作区/Gmail 邮箱。Slack/GitHub 的托管中继模式解决了桌面应用监听 webhook 的难题。

## 动手实验

1. 在 `connectors/base.py` 中找到 `SessionSource` 类，看它的 `target` 属性是如何生成的。
2. 在 `connectors/tools.py` 中找到 `make_integration_tools()`，追踪一个 Slack 工具从 descriptor 到注册到 registry 的路径。
3. 在 `engine.py` 中找到 `_record_result()`，看 `_display` sidecar 如何从工具结果中被剥离。
4. 配置一个 Gmail 连接器，设置一个 sender 过滤器，观察 tool card 上的 "hidden by filters" 提示。
