# 第 16 章 远程调用与 Web UI 可观测

上一章 Workflow 系统在本地执行了编排脚本。但在很多场景中，dsh 运行在远程服务器上，用户通过浏览器与它交互。`packages/api/` 实现了远程通信协议，让 Web UI 可以实时观察和控制 Agent 的行为。

## 请求到达这里时发生了什么

dsh 的远程通信基于 **Typert RPC** 协议——一个类型安全的远程过程调用框架。它让 TypeScript 类型在客户端和服务端之间共享，保证了 API 的类型一致性。

## 关键代码路径

### Typert RPC

Typert 是 dsh 自研的 RPC 框架，在 `@deepseek-ai/dsh-typert-protocol` 中定义。核心概念：

```typescript
interface TypertLookup<T, Id> {
  resolve(id: Id): T | undefined
  // 类型安全的查找接口
}

interface TypertLookupMap {
  session: TypertLookup<Session, SessionId>
  // 其他注册的类型...
}
```

`TypertLookupMap` 是一个接口，各模块通过 `declare module` 扩展它来注册自己的可查找类型。比如 Session 模块注册了 `session` 查找。

### Gateway：API 网关

Web UI 通过 WebSocket 连接到 dsh 的 gateway 进程。Gateway 负责：

1. **连接管理**：接受 WebSocket 连接，维护客户端会话
2. **事件转发**：将 `session/event`、`agent/status` 等事件实时推送给客户端
3. **命令代理**：将客户端的 RPC 调用转发给对应的 Service
4. **认证**：验证客户端身份

### 实时事件流

Web UI 的核心能力是**实时观察** Agent 的行为。这通过事件订阅实现：

```typescript
// 客户端代码
const subscription = gateway.subscribe('session/event', { sessionId })
subscription.onEvent((event) => {
  if (event.type === 'assistant/chunk') {
    renderToken(event.data.chunk)  // 逐 token 渲染
  }
  if (event.type === 'tool/result') {
    renderToolResult(event.data)
  }
})
```

Gateway 使用 `session/event` 事件的 fire-and-forget 语义——服务端不等待客户端确认，保证了高吞吐。

### Session 的 Typert 查找

Session 模块通过 Typert 注册了自己的查找接口：

```typescript
declare module '@deepseek-ai/dsh-typert-protocol' {
  interface TypertLookupMap {
    session: TypertLookup<Session, SessionId>
  }
}
```

这允许远程客户端通过 SessionId 查找 Session，而不需要直接引用 SessionStore：

```typescript
const session = ctx.typert.lookups.resolve('session')(sessionId)
```

### Telemetry

dsh 内置了 OpenTelemetry 遥测支持。Session 事件可以被导出为 OTLP traces：

```typescript
session-telemetry-otel: {
  // 配置
}
```

遥测是可选的，通过 `DSH_TELEMETRY_DISABLED` 环境变量控制。第 2 章中 `resolveTelemetryPatch` 函数就是处理这个开关的。

### Web Profile

`dsh web` 是 `dsh --profile web` 的别名。Web profile 的 bundle 栈包含：
- `@deepseek-ai/dsh-base`：核心功能
- `@deepseek-ai/dsh-web-app`：Web UI 服务器

Web app 插件启动一个 HTTP 服务器，提供静态文件服务和 WebSocket 端点。

## 调用链：从这里到下一步

在我们的 `headless` profile 中，没有 Web UI。Agent 的行为通过 stdout 输出给用户。

但无论哪种 surface，测试都是保证代码质量的关键。

**下一步：请求将流向测试系统——如何测试这条完整的请求路径？端到端测试如何模拟从 CLI 输入到工具执行的全流程？**

## 关键设计决策

1. **Typert 的类型共享**：客户端和服务端共享 TypeScript 类型，编译时保证 API 一致性。

2. **事件驱动的实时性**：Web UI 通过事件订阅获得实时更新，不需要轮询。

3. **fire-and-forget 语义**：事件推送不等待客户端确认，保证了服务端的高吞吐。

4. **可选遥测**：遥测默认开启但可以禁用，隐私优先。

## 本章小结

- Typert RPC 提供类型安全的远程过程调用
- Gateway 管理 WebSocket 连接，转发事件和命令
- Web UI 通过事件订阅实时观察 Agent 行为
- Session 通过 TypertLookup 注册可远程查找的接口
- 遥测基于 OpenTelemetry，可通过环境变量禁用

## 阅读思考题

1. Typert 的类型共享如何在编译时保证客户端和服务端的 API 一致性？
2. 如果 WebSocket 连接断开，Gateway 如何处理？客户端重连后能看到断开期间的事件吗？
3. 为什么遥测用 `DSH_TELEMETRY_DISABLED` 而不是 `DSH_TELEMETRY_ENABLED`？这与第 2 章的 `resolveTelemetryPatch` 有什么关系？

> **下一章预告**：我们已经追踪了请求的完整旅程。但如何确保这条路径的每一步都是正确的？下一章我们看测试策略——如何用端到端测试覆盖从 CLI 到工具执行的完整路径。
