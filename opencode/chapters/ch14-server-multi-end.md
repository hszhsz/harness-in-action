# 第 14 章 服务端与多端:一套 core,多个前端,事件流做骨架

到这里,我们已经把 OpenCode 的运行时从里到外拆了个遍:主循环、工具、权限、压缩、持久化、扩展。但有一个问题始终没正面回答——**这些能力是怎么被 CLI、TUI、桌面端、Web 端共享的?** 一个用户在终端里发起的会话,为什么能在另一个窗口里实时看到进度?

答案是 OpenCode 把整个 core 包成了一个 **HTTP 服务**,所有"端"都是这套服务的客户端。这一章我们读 `packages/server`,看它如何用 Effect 的 `HttpApi` 把 core 的能力暴露成路由、如何用一条 Server-Sent Events(SSE)流把第 12 章的事件流推给所有前端、又如何在请求边界上做认证和会话定位。你会看到,前面所有章节铺设的基础设施,在这里收束成一个清爽的服务层——core 几乎不知道 HTTP 的存在,server 只是薄薄一层适配。

## 14.1 一张声明式的 API:HttpApi 把路由组拼起来

服务的 API 表面定义在 `packages/server/src/api.ts`,用 Effect 的 `HttpApi.make` 声明式地拼装:

```ts
export const Api = HttpApi.make("server")
  .add(HealthGroup)
  .add(AgentGroup)
  .add(SessionGroup)
  .add(MessageGroup)
  .add(ModelGroup)
  .add(ProviderGroup)
  .add(PermissionGroup)
  .add(FileSystemGroup)
  .add(CommandGroup)
  .add(SkillGroup)
  .add(EventGroup)
  .add(QuestionGroup)
  .add(ReferenceGroup)
  .annotateMerge(OpenApi.annotations({ title: "opencode HttpApi", version: "0.0.1", ... }))
  .middleware(Authorization)
  .middleware(SchemaErrorMiddleware)
```

这个结构本身就是一份目录:OpenCode 的对外能力被切成十三个**组(group)**,每个组对应 core 的一块领域——session、message、model、provider、permission、文件系统、命令、技能、事件、问题(向用户提问)、引用。注意它们和前几章读过的 core 服务几乎一一对应:`SessionGroup` 背后是 `SessionV2.Service`,`PermissionGroup` 背后是第 10 章的权限系统,`SkillGroup` 背后是第 13 章的技能系统,`EventGroup` 背后是第 12 章的事件引擎。

server 包内部把每个组拆成两半:`groups/` 定义**契约**(端点的路径、入参出参 schema),`handlers/` 定义**实现**(收到请求后调用哪个 core 服务)。这种契约与实现分离,让同一份 API 定义既能驱动服务端、又能生成类型安全的客户端 SDK——`.annotateMerge(OpenApi.annotations(...))` 那行就是为了导出 OpenAPI 规范。

整个 API 最后挂了两个全局中间件:`Authorization`(认证)和 `SchemaErrorMiddleware`(把 schema 校验错误转成规范的 HTTP 错误)。中间件是 HttpApi 的横切关注点,所有路由共享。

## 14.2 handlers 装配:依赖注入在服务边界的又一次落地

光有契约和实现还不够,handlers 需要 core 的服务才能干活。`packages/server/src/handlers.ts` 把所有 handler 合并,再统一注入依赖:

```ts
export const handlers = Layer.mergeAll(
  HealthHandler, AgentHandler, SessionHandler, MessageHandler, ModelHandler,
  ProviderHandler, PermissionHandler, FileSystemHandler, CommandHandler,
  SkillHandler, EventHandler, QuestionHandler, ReferenceHandler,
).pipe(
  Layer.provide(sessionLocationLayer),
  Layer.provide(locationLayer),
  Layer.provide(SessionV2.defaultLayer),
  Layer.provide(SessionExecutionLocal.defaultLayer),
  Layer.provide(PermissionSaved.defaultLayer),
  Layer.provide(LocationServiceMap.layer),
)
```

这是我们在第 13 章 `PluginBoot` 里见过的同一种模式,只是换了个边界:所有 handler 被 `Layer.mergeAll` 合成一层,然后用 `Layer.provide` 把它们共同依赖的 core 服务(session、本地执行、已保存权限、位置映射)一次性供给。每个 handler 不必各自构造这些服务,它们共享同一批单例。

特别值得注意的是 `SessionExecutionLocal.defaultLayer`——它把"会话执行"绑定为**本地**实现。这暗示了架构的可替换性:执行后端是一个被注入的 Layer,本地直接跑是一种选择,理论上也能换成别的执行策略,而 handler 代码完全不用动。这正是 Effect 的依赖注入在架构层面的价值:**实现可换,契约不变**。

完整的路由组装在 `packages/server/src/routes.ts` 的 `createRoutes` 里,它把 API、handlers、认证、错误中间件,以及 `Database`、`EventV2`、HTTP 客户端等底层 Layer 全部 `provide` 进去,最终 `HttpRouter.toWebHandler` 产出一个标准的 Web handler。`createRoutes(password?)` 接一个可选密码参数——这是认证的入口,我们下一节就看。

## 14.3 认证:Basic Auth,默认关闭,开启即强制

认证逻辑分两处:策略在 `packages/server/src/auth.ts`,中间件在 `middleware/authorization.ts`。

配置来源很直接(`ServerAuth.Config.defaultLayer`):

```ts
password: EffectConfig.string("OPENCODE_SERVER_PASSWORD").pipe(EffectConfig.option),
username: EffectConfig.string("OPENCODE_SERVER_USERNAME").pipe(EffectConfig.withDefault("opencode")),
```

密码从环境变量 `OPENCODE_SERVER_PASSWORD` 读,是个 `option`(可能没有);用户名默认 `"opencode"`。是否需要认证由 `required` 判定:

```ts
export function required(config: Info) {
  return Option.isSome(config.password) && config.password.value !== ""
}
```

**只有当密码被设置且非空时,认证才生效**。没设密码,服务就是开放的——这对一个默认跑在本机、给本地前端用的 agent 服务来说是合理的默认值。但一旦你设了密码(比如要把服务暴露到网络),认证立刻变成强制。

中间件 `authorizationLayer` 据此分流:

```ts
if (!ServerAuth.required(config)) return Authorization.of((effect) => effect)
return Authorization.of((effect) =>
  Effect.gen(function* () {
    const request = yield* HttpServerRequest.HttpServerRequest
    const credential = yield* credentialFromRequest(request)
    if (ServerAuth.authorized(credential, config)) return yield* effect
    yield* HttpEffect.appendPreResponseHandler(...)  // 设 www-authenticate 头
    return yield* new UnauthorizedError({ message: "Authentication required" })
  }),
)
```

不需要认证时,中间件就是个透明的恒等函数 `(effect) => effect`,零开销。需要时,它从请求里取凭据、校验,失败就回 `UnauthorizedError` 并带上 `WWW-Authenticate: Basic realm="Secure Area"` 头。

凭据提取 `credentialFromRequest` 支持两种通道:URL 查询参数 `auth_token`,或标准的 `Authorization: Basic ...` 头。前者是为了 SSE/EventSource 这类**无法自定义请求头**的浏览器 API 准备的——EventSource 不能设 Authorization 头,只能把 token 塞进 URL。这是个为真实约束让路的务实设计。

两个安全细节值得点名:密码用 `Redacted.make` 包裹,意味着它在日志、错误、调试输出里都会被自动遮蔽,不会意外泄漏;`authorized` 的比较虽然是直接 `===`,但凭据全程以 Redacted 形式流转,降低了泄漏面。

## 14.4 会话定位:从 sessionID 反查它属于哪个目录

这是 server 层一个精妙但容易忽略的设计。OpenCode 是**多项目、多目录**的——同一个服务可能管理着好几个工作目录下的会话。当一个请求带着 `sessionID` 进来,server 怎么知道该在哪个目录、哪个工作区的上下文里执行?

答案是 `middleware/session-location.ts` 的 `SessionLocationMiddleware`。它在请求进入 handler 前,先用 sessionID 去数据库反查这个会话的位置:

```ts
const sessionID = yield* decodeSessionID(route.params.sessionID).pipe(
  Effect.mapError(() => new InvalidRequestError({ message: "Invalid session ID", field: "sessionID" })),
)
const row = yield* db
  .select({ directory: SessionTable.directory, workspaceID: SessionTable.workspace_id })
  .from(SessionTable)
  .where(eq(SessionTable.id, sessionID))
  .get()
  .pipe(Effect.orDie)
if (!row) return yield* new SessionNotFoundError({ sessionID, ... })

return yield* effect.pipe(
  Effect.provide(locations.get(Location.Ref.make({
    directory: AbsolutePath.make(row.directory),
    workspaceID: row.workspaceID ? WorkspaceV2.ID.make(row.workspaceID) : undefined,
  }))),
)
```

它做了三件事:先校验 sessionID 格式(解码失败 → `InvalidRequestError`),再查 `session` 表拿到这个会话的 `directory` 和 `workspace_id`(查不到 → `SessionNotFoundError`),最后——这是关键——用 `Effect.provide` 把对应的 `Location` 服务**注入到 handler 的执行环境里**。

这意味着 handler 本身完全不用关心"我在哪个目录"。它只管调 core 服务,而 `Location` 服务已经被中间件按 sessionID 精确地配置好了。位置这个上下文,通过 Effect 的依赖注入,从请求边界一路传到 core 内部。`locations.get(...)` 用一个 `LocationServiceMap` 按位置缓存/复用 Location 服务实例,避免每个请求都重建。

这又是一处"把横切关注点上提到中间件"的设计:会话定位这件事,所有需要 sessionID 的端点都要做,与其在每个 handler 里重复,不如做成一个 provides 类型的中间件,让类型系统保证拿到 sessionID 的 handler 一定有 Location 服务可用。

## 14.5 事件流:一条 SSE 把多端实时同步起来

终于到了多端架构的心脏。前端怎么实时知道"会话又产生了新消息"?靠轮询太笨。OpenCode 的答案是 `handlers/event.ts` 里那个 `event.subscribe` 端点——一条长连接的 Server-Sent Events 流。

它的实现把第 12 章的事件流接到了 HTTP 上:

```ts
const connected = {
  id: EventV2.ID.create(),
  type: "server.connected",
  location: new Location.Info({ directory: location.directory, workspaceID: location.workspaceID, project: location.project }),
  data: {},
}
return HttpServerResponse.stream(
  Stream.make(connected).pipe(
    Stream.concat(
      events.all().pipe(
        Stream.filter((event) =>
          event.location?.directory === location.directory &&
          event.location.workspaceID === location.workspaceID,
        ),
      ),
    ),
    Stream.map(eventData),
    Stream.pipeThroughChannel(Sse.encode()),
    Stream.encodeText,
  ),
  { contentType: "text/event-stream", headers: { "Cache-Control": "no-cache, no-transform", "X-Accel-Buffering": "no", ... } },
)
```

逐层拆解这条流水线:

1. **先发一个握手事件**:`Stream.make(connected)` 让流的第一帧是一个 `server.connected` 事件,带上当前连接的位置信息。前端一收到它就知道连接建立、自己绑定到哪个目录/工作区。
2. **接上全量事件流**:`Stream.concat(events.all())`——`events.all()` 正是第 12 章那个"历史 + 实时 PubSub"拼接而成的事件流。新连上的客户端能拿到完整事件,再无缝衔接实时更新。
3. **按位置过滤**:`Stream.filter(...)` 只放行 `directory` 和 `workspaceID` 都匹配当前连接位置的事件。这是多项目隔离的保证——一个绑定到项目 A 的前端,绝不会收到项目 B 的会话事件。
4. **编码成 SSE**:`Stream.map(eventData)` 把每个事件包成 `{ event: "message", data: JSON.stringify(...) }`,`Sse.encode()` 转成 SSE 线格式,`Stream.encodeText` 转字节。
5. **设对的响应头**:`text/event-stream` 是 SSE 的 content-type;`Cache-Control: no-cache, no-transform` 和 `X-Accel-Buffering: no` 是为了对抗中间代理(尤其 nginx)的缓冲——如果代理缓冲了流,实时性就没了,这两个头明确告诉代理"别缓冲、别变换"。

这条 SSE 流就是整个多端架构的骨架。任意多个前端(TUI、桌面、Web)同时订阅同一个位置,server 把 core 产生的每一个事件广播给所有订阅者。一个端发起的对话进度,通过事件流实时出现在所有端上。**前端之间不直接通信,它们通过共享同一条事件流达成一致**——这是典型的、健壮的事件驱动同步,而它能成立,完全建立在第 12 章那套"每个状态变更都是一个可广播事件"的基座之上。

## 14.6 错误的形状:把领域错误翻译成 HTTP 语义

server 层还有一个低调但重要的职责:把 core 抛出的**领域错误**翻译成前端能理解的 **HTTP 错误**。看 `handlers/session.ts` 里 `session.prompt` 的处理:

```ts
.pipe(
  Effect.catchTag("Session.NotFoundError", (error) =>
    Effect.fail(new SessionNotFoundError({ sessionID: error.sessionID, message: `Session not found: ${error.sessionID}` }))),
  Effect.catchTag("Session.PromptConflictError", (error) =>
    Effect.fail(new ConflictError({ message: `Prompt message ID conflicts with an existing durable record: ${error.messageID}`, resource: error.messageID }))),
)
```

`Effect.catchTag` 按错误的标签精确匹配:core 的 `Session.NotFoundError` 被翻译成 server 的 `SessionNotFoundError`(对应 404 语义),`Session.PromptConflictError` 翻译成 `ConflictError`(对应 409)。这是 Effect 类型化错误的威力——错误不是 `throw` 出来的不透明异常,而是带标签、可穷举匹配的值,server 能为每一种 core 错误给出恰当的 HTTP 响应。

`session.context` 的错误处理里还有一个细节值得学:遇到 `Session.MessageDecodeError`(内部数据损坏)时,它不把内部细节漏给客户端,而是生成一个短引用号、把详情记进日志、只回一个泛化的 `UnknownError`:

```ts
const ref = `err_${crypto.randomUUID().slice(0, 8)}`
return Effect.logError("failed to decode session message").pipe(
  Effect.annotateLogs({ ref, sessionID: error.sessionID, messageID: error.messageID }),
  Effect.andThen(Effect.fail(new UnknownError({ message: "Unexpected server error. Check server logs for details.", ref }))),
)
```

客户端只拿到 `ref`,运维凭 `ref` 在日志里查全貌。这是边界处的信息隐藏:**内部错误细节留在服务端,对外只暴露一个可追溯的引用号**。它既保护了内部实现(不泄漏 sessionID/messageID 给可能不可信的客户端),又保留了可调试性。这正是第 13 章"边界不信任"原则在错误传播方向上的镜像。

## 14.7 薄服务层的价值

读完整个 server 包(总共一千七百多行),最深的印象是它有多**薄**。它几乎不包含业务逻辑——所有真正的工作都在 core 里。server 干的是四件纯粹的边界工作:

- **把 core 能力声明成 HTTP 契约**(api.ts / groups);
- **在请求边界做认证和会话定位**(两个中间件);
- **把 core 事件流接成 SSE 推给多端**(event handler);
- **把领域错误翻译成 HTTP 错误**(handler 里的 catchTag)。

这种"薄"不是偷懒,而是分层的胜利。因为 core 被设计成不依赖任何传输层,server 才能这么薄;因为状态变更早被建模成可广播的事件(第 12 章),多端同步才能简化成"订阅一条流";因为错误是类型化的值(贯穿全书),错误翻译才能这么精确。CLI、TUI、桌面、Web 之所以能共享一套大脑,正是因为这套大脑(core)从一开始就和"嘴"(server)解耦了。

## 本章小结

- OpenCode 把整个 core 包成一个 HTTP 服务,CLI/TUI/桌面/Web 都是它的客户端;`api.ts` 用 `HttpApi.make` 把能力声明成十三个 group,每个 group 对应一块 core 领域,并挂 `Authorization` + `SchemaErrorMiddleware` 两个全局中间件。
- server 内部契约(`groups/`)与实现(`handlers/`)分离,既驱动服务端又能生成 OpenAPI / 类型化客户端;`handlers.ts` 用 `Layer.mergeAll` + `Layer.provide` 统一注入 core 服务,执行后端 `SessionExecutionLocal` 作为可替换 Layer 注入。
- 认证是 Basic Auth,密码从 `OPENCODE_SERVER_PASSWORD` 读取;**未设密码则开放、设了即强制**;凭据可走 `Authorization` 头或 `auth_token` 查询参数(为 EventSource 让路),密码用 `Redacted` 全程遮蔽。
- `SessionLocationMiddleware` 用 sessionID 反查 `session` 表拿到 directory/workspaceID,再用 `Effect.provide` 把对应 `Location` 服务注入 handler——位置上下文通过依赖注入从请求边界传到 core,handler 无需关心自己在哪个目录。
- 多端实时同步靠 `event.subscribe` 的一条 SSE 流:先发 `server.connected` 握手,再接第 12 章的 `events.all()`(历史+实时),按位置 `Stream.filter` 做项目隔离,设 `no-cache/X-Accel-Buffering: no` 对抗代理缓冲;前端通过共享事件流达成一致,不直接通信。
- 错误处理用 `Effect.catchTag` 把 core 领域错误精确翻译成 HTTP 错误(NotFound→404、Conflict→409);内部错误(如 MessageDecodeError)只回一个引用号 `ref`、详情记日志,不泄漏内部细节。
- server 层极薄(~1770 行),业务全在 core——这种薄是分层解耦的胜利:core 不依赖传输层、状态即事件、错误即类型,三者共同让多端共享一套大脑成为可能。

## 动手实验

1. **订阅事件流看多端同步**:本地启动 OpenCode 服务,用 `curl -N http://localhost:<port>/event` 订阅 SSE 流(`-N` 禁用缓冲)。然后在另一个终端发起一次会话对话,观察 curl 这边实时滚出的事件。对照 `handlers/event.ts`,确认第一帧是 `server.connected`,后续是被位置过滤后的会话事件。

2. **验证认证的"默认关闭、设了即强制"**:先不设 `OPENCODE_SERVER_PASSWORD` 启动,确认请求无需凭据即可访问。再设一个密码重启,确认不带凭据的请求收到 401 和 `WWW-Authenticate` 头,带正确 Basic 凭据则通过。对照 `auth.ts` 的 `required` 函数理解这个分界。

3. **追一次会话定位**:阅读 `middleware/session-location.ts`,用一个不存在的 sessionID 调用某个需要 sessionID 的端点,确认返回 `SessionNotFoundError`;再用一个格式非法的 sessionID,确认返回 `InvalidRequestError`。体会中间件如何在 handler 之前就把这两类错误挡掉,并为合法请求注入 Location 服务。

4. **观察错误翻译**:阅读 `handlers/session.ts` 的 `session.prompt` 和 `session.context`,列出它用 `Effect.catchTag` 捕获的每一种 core 错误以及对应的 server 错误。然后思考:`session.context` 为什么对 `MessageDecodeError` 用"引用号 + 日志"而不是直接把错误信息回给客户端?这种处理保护了什么、又保留了什么调试能力?

---

下一章我们转向一个常被忽视却决定项目能不能长期演进的维度:**可测试性与可信度**。第 15 章将读 OpenCode 的测试代码与测试基础设施,看它如何用依赖注入做隔离、如何用内存数据库做确定性测试、如何把"断言"当成防止回归的安全网——以及前面十四章反复出现的那些设计(类型化错误、Layer 注入、纯函数边界)是怎么同时也让这个系统变得可测的。
