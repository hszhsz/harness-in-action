# 第 17 章　远程调用与可观测性：Typert RPC 与 Web UI

> dsh 的 Web UI 不是"挂在主循环旁边的网页"，而是一个完整的远程调用体系：Typert 类型图生成器 + `@Remote` 方法 + API Gateway + ACP 自动化服务器 + client modules。这一章拆开 `packages/typert`、`packages/api`、`packages/acp`、`packages/client` 与 `apps/web`。

## 17.1　Typert：类型图生成器与 `@Remote` 方法

Typert 是 dsh 的远程调用基础设施：一个**类型图生成器、loader 和运行时注册表**（`packages/typert`）。[[docs/api-gateway.md]](https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/api-gateway.md)

业务服务用 `@Remote` 或 `@RemoteScope` 装饰器选择暴露给 Client 的方法——**未标记的方法不进生成的 Client 类型、不进运行时贡献、不能通过 `ctx.remote` 调用**。

```ts
export class GoalService extends TypertRemoteService {
  constructor(ctx: Context) {
    super(ctx, 'goals')
  }

  @Remote('create')
  createForClient(agent: Agent, request: CreateGoalRequest, signal: AbortSignal): CreateGoalResult {
    // …
  }
}
```

**复杂 Host 对象不能直接过 wire**：业务包必须通过 `TypertLookupMap` 声明它们与 wire 身份的关联，并注册默认解析 provider（`ctx.typert.lookups`）。例如 Host 签名里的 `agent: Agent` 参数产生 `agentId` wire 字段，Gateway 在调用业务方法前把 id 解析回 Host 对象。`@RemoteScope(key)` 先把身份解析成 scoped Context，再从那个 Context 拿服务调方法。

**两个 merge-extensible 空 map**：`TypertLookupMap`（Host 对象类型 ↔ wire 身份）、`TypertContextMap`（scoped Context kind ↔ wire 身份）。注册表在 resolver 卸载后**保留 lookup 的 wire 声明**——SRC 发现继续把参数分类为 lookup，失败时报 unavailable，而不是把 wire 值当普通业务对象接受。

## 17.2　API Gateway 与 Lookup/Context 声明

`packages/api`：`api-gateway`（`ctx.typert` 的 Host Gateway）+ remotes。`docs/api-gateway.md` 描述编程模型：

- **unary Remote 方法**：业务服务声明 → 构建生成 Host 和 Client 契约 → 调用复用 Connection RPC 和 `/api` route；
- **流式协议在本文档范围外**：session events、增量数据等可能用同一个 Connection，但不用 Remote 方法描述符。

## 17.3　Session query / projection / telemetry：回放与上报

- **Session query**（`session-query/`）：逻辑记录、有界精确事件读、关系 trace、语义过滤/文档、全文结果页。工具：`session_event_read`/`session_event_search`/`session_event_trace`/`session_search`/`session_trace`——五个只读工具把 provider cursor 藏起来，从不可变 calling agent session 授权每个结果。
- **Projection**（`session-projection/`）：`SessionProjectionMap`、纯 `ProjectionDefinition` 单元、`ProjectionSnapshot` 的一致切面、change feed。
- **Telemetry**（`session-telemetry/`）：出站 session 上报 seam：`SessionTelemetryRecord`/`SessionTelemetrySeverity`、`SessionTelemetrySink` 契约、`session-telemetry/record` redact waterfall。
- **Session title**：持久标题快照 + cited source-message seqs + 异步 provider 契约（`session-title`，base bundle 里 `fallbackMaxWords: 5` / `maxTitleBytes: 80` 等配置可见）。

## 17.4　ACP（Agent Client Protocol）自动化服务器

`packages/acp` 是 **automation-only 的 Agent Client Protocol server**——自动化场景的机器决策（"The ACP automation bridge provides one-shot machine decisions for its own agents"）。`dsh-acp-demo` 是 demo bin。ACP provider 在 subagent 体系里是"委托给另一个产品"的后端之一（第 15.1 节）。

## 17.5　Web UI：`apps/web` + client modules + HMR

- **`apps/web`**：`@deepseek-ai/dsh-web-frontend`（Vite 构建，`apps/web/src/main.ts` 是入口），dist 由 `apps/cli` 的 `dsh web` 服务。
- **`packages/client`**：Web 前端模块——`web`（shell 库）、`web-react`、`connection`（RPC）、`hmr`、`locale`、`modules`（`dsh.client` 声明：浏览器名册）、`runtime`、`schema-form`，以及几十个 `ui-*` 模块（conversation / tool / subagent / workflow-run / goal / jobs / plan / settings / skill / trajectory / user-questions / workspace / deliverables / feedback / message-feedback / permission-presets / model-selection / directory-picker / theme / layout / sidebar / slots / input-trigger / commands / attachment / session-query …）。
- **`client-modules`**（`docs/subsystems/client-modules.md`）：`dsh.client` 声明 + `WebBootGraph` wire 组合 + bundle route + index tap（`tapIndex` 把 boot manifest 注入 index.html）。
- **Web server**（`host/webserver`）：单 `node:http` 插件提供 `ctx.webServer`：named-route 注册表（exact 先、最长 prefix 次、fallback 最后）、index.html transform 回调、可认领的 fallback seat。`host` 只接受 `127.0.0.1`（默认）和 `0.0.0.0`（故意暴露）；**无 TLS、无 auth、无 origin policy**——非 loopback 绑定就把服务器暴露给那个网络。前端静态（`host/frontend-static`）认领 fallback seat：非 GET/HEAD 405、dist 根外 traversal 403、SPA miss 回退 `index.html` 200。
- **Electron**：加载 built 文件走 `file://`，fetch 请求走 IPC bridge 而不是这个 HTTP server（"It serves browsers only"）。

## 本章小结

dsh 的远程与可观测体系：

1. **Typert**：装饰器驱动的远程方法，Host 对象 ↔ wire 身份分离，未标记方法不可达；
2. **API Gateway**：unary Remote 复用 Connection RPC 与 `/api` route；
3. **查询/投影/遥测**：session-query 只读工具、projection 一致切面、telemetry redact waterfall；
4. **ACP**：自动化场景的机器决策服务器；
5. **Web UI**：client modules 声明式组合 + webserver 严格路由 + Electron file:// 与 HTTP 双通道。

## 动手实验

1. 读 `packages/typert/protocol/src/types.ts` 的 `TypertLookupMap` / `TypertContextMap`，理解 merge-extensible 声明。
2. 读 `packages/api/gateway`，看 `ctx.typert.lookups.configure()` 如何覆盖 lookup 解析政策。
3. 读 `packages/host/webserver` 的 route match order，确认 exact → prefix → fallback。
4. 在 `apps/web/src/main.ts` 里找 boot manifest 的注入点（`window.__DSH_BOOT__`）。
