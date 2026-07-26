# 第 11 章　MCP 集成：把外部协议接进工具计划

## 11.1　MCP 在 OpenWorker 中的定位

OpenWorker 支持 [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) 作为一等公民扩展机制。任何 MCP server 暴露的工具都可以被注册到 ToolRegistry 中，和内置工具享受同等的权限门控和审计。

`coworker/mcp/` 目录包含四个文件：

| 文件 | 职责 |
|---|---|
| `config.py` | MCP server 配置管理（增删改查） |
| `client.py` | MCP 客户端：连接、发现工具、调用 |
| `tools.py` | 把 MCP 工具包装成 registry 可注册的函数 |
| `oauth.py` | MCP server 的 OAuth 浏览器流程 |

## 11.2　MCP Server 配置

MCP server 的配置存储在 SecretStore 中，每个 server 有：

- **name**：唯一标识符
- **config**：连接配置（command/args for stdio，URL for SSE/HTTP）
- **auth**：认证方式（none / oauth）
- **status**：connected / disconnected / needs_auth / authorizing

API 端点：
- `GET /v1/mcp` -- 列出所有已配置的 server
- `POST /v1/mcp` -- 添加新 server
- `PATCH /v1/mcp/{name}` -- 更新配置
- `DELETE /v1/mcp/{name}` -- 删除
- `GET /v1/mcp/{name}/tools` -- 列出 server 暴露的工具
- `POST /v1/mcp/{name}/connect` -- 连接（OAuth 的可能需要浏览器流程）
- `POST /v1/mcp/reload` -- 重载所有 server

## 11.3　MCP 工具的注册

`mcp/tools.py` 的核心职责是把 MCP server 暴露的工具转换成 Python 可调用函数，注册到 ToolRegistry：

1. 连接到 MCP server
2. 调用 `tools/list` 获取工具清单
3. 每个 MCP 工具被包装成一个 Python 函数：
   - 函数名 = MCP 工具名
   - 函数签名 = 从 MCP 工具的 inputSchema 生成
   - 函数体 = 调用 `client.call_tool(name, arguments)`
4. metadata 标记为 `requires_approval=True`（默认 EXTERNAL 风险）

这个设计让 MCP 工具和内置工具在 TurnEngine 的眼中没有区别--它们都经过同一个 `_authorize()` 流程，都受同一个 PermissionEngine 管控。

## 11.4　OAuth 浏览器流程

`mcp/oauth.py` 处理需要 OAuth 认证的 MCP server。流程：

1. `POST /v1/mcp/{name}/connect` 启动连接
2. 如果 server 配置了 `auth: oauth`，打开系统浏览器
3. 用户在浏览器中完成 OAuth 授权
4. MCP server 重定向到 `http://localhost:<port>/mcp/oauth/callback`
5. `deliver_callback(code, state)` 把结果传回等待的 client
6. 工具列表被刷新，server 状态变为 connected

回调页面用和连接器相同的 `_browser_page()` 渲染--一个离线可渲染的 HTML 卡片。

## 11.5　连接器与 MCP 的关系

OpenWorker 的连接器系统有两种集成路径：

1. **原生路径**：`connectors/base.py` 的 BasePlatformAdapter + 手写工具
2. **MCP 路径**：通过 `mcp_url` 字段，连接器可以声明自己有一个 MCP endpoint

```python
# connectors/descriptors.py
@dataclass(frozen=True)
class ConnectorDescriptor:
    ...
    mcp_url: str = ""  # 非空 -> 支持 MCP 连接
```

当一个连接器有 `mcp_url` 时，`POST /v1/connectors/{name}/mcp-connect` 会通过 MCP 路径连接它。这让连接器可以渐进地从"手写工具"迁移到"MCP server"。

## 11.6　MCP 工具的风险覆盖

MCP 工具默认被分类为 `EXTERNAL`（因为 `requires_approval=True`）。但如果用户信任某个 MCP server 的某些工具，可以用 `overrides.py` 把它的风险降级：

```python
# coworker/overrides.py
class RiskOverrideStore:
    def resolver(self) -> RiskOverrides:
        # 返回一个 callable: tool_name -> Optional[RiskClass]
        # 用户可以把特定 MCP 工具从 EXTERNAL 降到 READ
```

`build_engine()` 中加载这个 store：

```python
risk_overrides = RiskOverrideStore(state_dir() / "risk_overrides.json").resolver()
permissions = PermissionEngine(..., risk_overrides=risk_overrides)
```

**安全约束**：override 只能放松（从 EXTERNAL 降级），不能收紧（不能把 READ 升级为 EXEC）。但 Phase 2 的设计意图是"主要用来放松 MCP 的保守默认"--因为 MCP 工具的 `requires_approval` 是 server 声明的，不一定反映真实风险。

## 本章小结

OpenWorker 的 MCP 集成把外部协议工具无缝地纳入了内部的"风险分级 + 权限门控 + 审计"体系。MCP 工具和内置工具在 TurnEngine 眼中没有区别。最值得借鉴的是：OAuth 浏览器流程的统一回调处理、连接器的 MCP 渐进迁移路径、以及用户级风险覆盖的 override 机制。

## 动手实验

1. 在 `mcp/tools.py` 中找到工具包装逻辑，追踪一个 MCP 工具从 server 暴露到注册到 registry 的路径。
2. 在 `mcp/oauth.py` 中找到 `deliver_callback()`，看它如何把浏览器回调结果传回等待的 client。
3. 在 `overrides.py` 中找到 `RiskOverrideStore`，思考为什么 override 只能放松不能收紧。
4. 配置一个 stdio 类型的 MCP server（如 filesystem），观察它的工具如何出现在工具列表中。
