# 第 17 章　远程调用与可观测性：Typert RPC 与 Web UI

dsh 的 Web UI 不是"挂在主循环旁边的网页"，而是一个完整的远程调用体系：Typert 类型图生成器 + `@Remote` 方法 + API Gateway + ACP 自动化服务器 + client modules。这一章拆开 `packages/typert`、`packages/api`、`packages/acp`、`packages/client` 与 `apps/web`。读完后，你应该能理解 Typert 的 Host 对象 ↔ wire 身份分离、session query 的只读工具体系、以及 Web UI 的声明式组合。

## Typert：类型图生成器与 `@Remote` 方法

Typert 是 dsh 的远程调用基础设施：一个**类型图生成器、loader 和运行时注册表**（`packages/typert`）。`packages/typert/protocol/src/types.ts` 定义了核心类型：

```ts
export interface TypertLookup<Host, Wire> {
  readonly [LOOKUP_HOST]: Host
  readonly [LOOKUP_WIRE]: Wire
}

export interface TypertContext<Wire> {
  readonly [CONTEXT_WIRE]: Wire
}

export interface TypertLookupMap {}
export interface TypertContextMap {}
export interface TypertRemoteMap {}
export interface TypertRemoteScopeMap {}
```

这些是 **merge-extensible 空 map**——插件通过声明合并往里面加条目。`TypertLookupMap` 声明 Host 对象类型与 wire 身份的关联，`TypertContextMap` 声明 scoped Context kind 与 wire 身份的关联。

业务服务用 `@Remote` 或 `@RemoteScope` 装饰器选择暴露给 Client 的方法——**未标记的方法不进生成的 Client 类型、不进运行时贡献、不能通过 `ctx.remote` 调用**。

**复杂 Host 对象不能直接过 wire**：业务包必须通过 `TypertLookupMap` 声明它们与 wire 身份的关联，并注册默认解析 provider（`ctx.typert.lookups`）。例如 Host 签名里的 `agent: Agent` 参数产生 `agentId` wire 字段，Gateway 在调用业务方法前把 id 解析回 Host 对象。

注册表在 resolver 卸载后**保留 lookup 的 wire 声明**——SRC 发现继续把参数分类为 lookup，失败时报 unavailable，而不是把 wire 值当普通业务对象接受。

`RemoteResult` 是每个 Remote 方法的标准返回类型：

```ts
export type RemoteResult<T> =
  | { readonly ok: true; readonly value: T }
  | { readonly ok: false; readonly error: RemoteFailure }
```

## API Gateway 与 Lookup/Context 声明

`packages/api`：`api-gateway`（`ctx.typert` 的 Host Gateway）+ remotes。

- **unary Remote 方法**：业务服务声明 → 构建生成 Host 和 Client 契约 → 调用复用 Connection RPC 和 `/api` route。
- **流式协议在本文档范围外**：session events、增量数据等可能用同一个 Connection，但不用 Remote 方法描述符。

## Session query / projection / telemetry：回放与上报

**Session query**（`session-query/`）：逻辑记录、有界精确事件读、关系 trace、语义过滤/文档、全文结果页。工具：`session_event_read`/`session_event_search`/`session_event_trace`/`session_search`/`session_trace`——五个只读工具把 provider cursor 藏起来，从不可变 calling agent session 授权每个结果。

**Projection**（`session-projection/`）：`SessionProjectionMap`、纯 `ProjectionDefinition` 单元、`ProjectionSnapshot` 的一致切面、change feed。

**Telemetry**（`session-telemetry/`）：出站 session 上报 seam：`SessionTelemetryRecord`/`SessionTelemetrySeverity`、`SessionTelemetrySink` 契约、`session-telemetry/record` redact waterfall。

**Session title**：持久标题快照 + cited source-message seqs + 异步 provider 契约。base bundle 里配置可见：`fallbackMaxWords: 5` / `maxTitleBytes: 80`。

## ACP（Agent Client Protocol）自动化服务器

`packages/acp` 是 **automation-only 的 Agent Client Protocol server**——自动化场景的机器决策。ACP provider 在 subagent 体系里是"委托给另一个产品"的后端之一。

## Web UI：`apps/web` + client modules + HMR

- **`apps/web`**：Vite 构建，`apps/web/src/main.ts` 是入口，dist 由 `apps/cli` 的 `dsh web` 服务。
- **`packages/client`**：Web 前端模块——`web`（shell 库）、`web-react`、`connection`（RPC）、`hmr`、`locale`、`modules`（`dsh.client` 声明：浏览器名册）、`runtime`、`schema-form`，以及几十个 `ui-*` 模块。
- **`client-modules`**：`dsh.client` 声明 + `WebBootGraph` wire 组合 + bundle route + index tap（`tapIndex` 把 boot manifest 注入 index.html）。
- **Web server**（`host/webserver`）：单 `node:http` 插件提供 `ctx.webServer`：named-route 注册表（exact 先、最长 prefix 次、fallback 最后）。**无 TLS、无 auth、无 origin policy**——非 loopback 绑定就把服务器暴露给那个网络。
- **Electron**：加载 built 文件走 `file://`，fetch 请求走 IPC bridge（"It serves browsers only"）。

## 架构速览

```
远程调用与可观测性
┌─────────────────────────────────────────────┐
│  Typert (类型图生成器 + 远程调用)              │
│  @Remote / @RemoteScope 装饰器               │
│  Host对象 ↔ wire身份 分离                    │
│  TypertLookupMap / TypertContextMap          │
│  (merge-extensible 空 map)                  │
├─────────────────────────────────────────────┤
│  API Gateway                                │
│  unary Remote → Connection RPC + /api route │
├─────────────────────────────────────────────┤
│  Session query / projection / telemetry     │
│  5个只读工具 (session_event_read/search/...) │
│  projection: 一致切面 + change feed          │
│  telemetry: redact waterfall                │
├─────────────────────────────────────────────┤
│  ACP (自动化服务器)                           │
│  automation-only 机器决策                    │
├─────────────────────────────────────────────┤
│  Web UI                                     │
│  apps/web (Vite) + packages/client          │
│  client-modules 声明式组合                    │
│  webserver: exact → prefix → fallback       │
│  无 TLS / 无 auth / 无 origin policy         │
└─────────────────────────────────────────────┘
```

## 关键设计决策

1. **Host 对象 ↔ wire 身份分离**：`TypertLookupMap` 让 Host 对象（如 `Agent`）通过 branded id（如 `agentId`）过 wire。Gateway 在调用业务方法前把 id 解析回 Host 对象。这让远程调用的序列化和反序列化是类型安全的。

2. **未标记的方法不可达**：只有 `@Remote` 或 `@RemoteScope` 装饰的方法才暴露给 Client。这让业务服务可以有内部方法（不暴露）和远程方法（暴露），而不需要在运行时做权限检查。

3. **Session query 是只读的**：5 个只读工具（`session_event_read`/`search`/`trace`/`session_search`/`trace`）把 provider cursor 藏起来。这让 UI 可以查询 session 数据，但不能修改它。

4. **Web server 无 TLS / 无 auth**：默认绑定 `127.0.0.1`（loopback），故意暴露时才绑 `0.0.0.0`。这是一个有意识的安全取舍——简化本地开发，但非 loopback 绑定就把服务器暴露给网络。

## 本章小结

- Typert：装饰器驱动的远程方法，Host 对象 ↔ wire 身份分离，未标记方法不可达。
- API Gateway：unary Remote 复用 Connection RPC 与 `/api` route。
- 查询/投影/遥测：session-query 只读工具、projection 一致切面、telemetry redact waterfall。
- ACP：自动化场景的机器决策服务器。
- Web UI：client modules 声明式组合 + webserver 严格路由。

## 阅读思考题

1. `TypertLookupMap` 在 resolver 卸载后保留 wire 声明。如果一个业务包被卸载了，但 Client 仍然尝试调用它的 Remote 方法，会发生什么？
2. Session query 的 5 个只读工具从 calling agent session 授权每个结果。如果一个子 agent 尝试查询父 agent 的 session 数据，会被允许吗？
3. Web server 无 TLS / 无 auth。如果用户在 `0.0.0.0` 上启动 dsh web，同一网络的任何人都能完全控制 Agent。dsh 有什么机制提醒用户这个风险？

> **下一章预告**：第 18 章将讲 dsh 的测试与可信度体系——keyless snapshot 测试、运行时不变量注册表、每文件 100% 覆盖率门禁。
