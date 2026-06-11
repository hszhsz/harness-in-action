# 第 10 章 网关与会话路由——把每一次调用都收敛到同一条总线上

> 上一章我们说,子智能体是"一棵会自我繁殖的树"。我们也埋下了几个伏笔:9.3 节那个被列入 `SUBAGENT_TOOL_DENY_ALWAYS` 的 `gateway` 工具,为什么子智能体连碰都不许碰?那些"admin-only 方法被钉死在 `ADMIN_SCOPE` 上"的说法,到底是钉在哪里?子智能体把结果"自动推送回父级",这条推送又是走的哪条路?
>
> 这一章,我们就来拆掉幕布后面的那个角色:**网关(gateway)**。它是 OpenClaw 里所有"跨进程/跨边界请求"的统一入口——CLI 命令、远程节点、插件方法、会话操作,最终都收敛成同一种东西:**一次带着 scope 的网关方法调用**。理解了网关,你就理解了 OpenClaw 是如何用"一条总线 + 一张权限表"把一个本可以四处蔓延的系统约束成可审计、可拒绝、默认安全的形态。

本章我们会沿着一次请求的生命周期走完整条链路:

1. 调用方如何进入网关(`callGateway` 这个唯一入口);
2. 每个方法需要什么权限(operator scope 模型与那张核心方法表);
3. 权限是怎么判定的(默认拒绝、admin 永远放行、最小权限解析);
4. 不启动服务器时,本地路径如何"嵌入式"复用网关方法;
5. 会话键(session key)如何把一次请求路由到正确的会话;
6. 传输层的安全硬化(把 URL override 当作不可信重定向、把错误当文档)。

---

## 10.1 为什么需要一条总线

设想一个没有网关的世界。CLI 想列出会话,就直接读会话存储;Web 控制台想发消息,就直接调 agent loop;远程节点想上报事件,就直接写运行时状态;插件想暴露一个自定义动作,就自己挂一个 HTTP handler。每个调用方各走各的路,于是:

- **权限是散落的**。"谁能改配置"这个问题,要去 N 个调用点分别检查,漏掉一个就是一个提权漏洞。
- **可观测性是缺失的**。没有统一入口,就没有统一的审计点、限流点、startup 可用性判定点。
- **边界是模糊的**。本地调用和远程调用混在一起,无法回答"这个请求来自一个可信的本机操作者,还是一个刚配对上的远程节点"。

OpenClaw 的答案是把所有这些请求都规约成同一个抽象:**网关方法(gateway method)**。每个方法有一个名字(如 `sessions.list`、`config.set`、`agent`),一份权限策略(它需要哪个 scope),一个 handler(真正干活的函数),以及若干元数据(是否在启动期可用、是否算控制面写操作、是否对外广播)。调用方不再直接碰业务逻辑,而是统一通过 `callGateway` 发起一次"方法调用",由网关来认证、鉴权、派发。

这就是"总线"的含义:**所有请求汇入同一条管道,在同一个地方被分类、被鉴权、被派发。** 本章接下来的每一节,都是在拆这条管道的一段。

---

## 10.2 唯一入口:`callGateway`

调用方进入网关的门只有一扇:`src/gateway/call.ts` 里的 `callGateway`。它的职责不是"干活",而是"决定这次调用走哪条路、带什么权限",然后把真正的工作交给下游。它的分流逻辑大致是这样的:

```typescript
const callerMode = opts.mode ?? GATEWAY_CLIENT_MODES.BACKEND;
const callerName = opts.clientName ?? GATEWAY_CLIENT_NAMES.GATEWAY_CLIENT;
// CLI 路径单独处理:它有自己的默认 scope 策略
if (callerMode === CLI || callerName === CLI) {
  return callGatewayCli(opts);
}
// 调用方显式给了 scopes —— 尊重它,按给定 scope 调用
if (Array.isArray(opts.scopes)) {
  return callGatewayWithScopes({ ...opts, mode, clientName }, opts.scopes);
}
// 既不是 CLI、也没给 scopes —— 走最小权限解析
return callGatewayLeastPrivilege({ ...opts, mode, clientName });
```

这里有三条分支,对应三种调用方画像:

- **CLI 模式**:命令行操作者。它走 `callGatewayCli`,有一套专门的默认 scope 兜底逻辑(见下文 10.3 的 `CLI_DEFAULT_OPERATOR_SCOPES`)。
- **显式 scopes**:调用方明确知道自己需要哪些权限,直接传进来。网关尊重它——但注意,这只是"声明我要用这些 scope 去敲门",最终能不能进,还要看服务端那张权限表。
- **最小权限(least-privilege)**:既没声明也不是 CLI,网关替它**算出**这次调用最少需要哪些 scope,只带那么多去敲门。这是默认行为,也是"最小权限原则"在代码里的落地。

`call.ts` 是本章最核心的文件(一千多行),但它的复杂度几乎全在于**把不同来源、不同模式、不同认证方式的调用收敛成同一条请求**,而不是在于业务逻辑。它定义了三类专门的错误,本身就说明了它关心什么:

- `GatewayTransportError`(传输错误,`kind` 区分 `"closed"` / `"timeout"`)——连接层面的失败;
- `GatewayCredentialsRequiredError`——缺凭证;
- `GatewayExplicitAuthRequiredError`——需要**显式**认证(后面 10.6 会讲为什么"显式"很重要)。

> **设计观察:入口收敛是一切安全约束的前提。** 如果有第二扇门,那张权限表就只能保护一半的请求。OpenClaw 把"唯一入口"做成硬约束——这也解释了第 9 章里那个谜题:为什么 `gateway` 工具要被钉进 `SUBAGENT_TOOL_DENY_ALWAYS`。`gateway` 工具本质上是让模型能直接发起任意网关方法调用,等于把这扇唯一的、本应由代码精确控制权限的门,交到了一个不可信的子智能体手里。堵死它,总线才仍然是总线。

---

## 10.3 权限模型:六种 operator scope

网关的鉴权建立在一个**封闭的权限集合**之上。`src/gateway/operator-scopes.ts` 把它定义得极其克制——只有六个常量:

```typescript
export const ADMIN_SCOPE = "operator.admin" as const;
export const READ_SCOPE = "operator.read" as const;
export const WRITE_SCOPE = "operator.write" as const;
export const APPROVALS_SCOPE = "operator.approvals" as const;
export const PAIRING_SCOPE = "operator.pairing" as const;
export const TALK_SECRETS_SCOPE = "operator.talk.secrets" as const;
```

注意它们的命名空间是 `operator.*`——这是"操作者"权限,与"节点(node)"角色是两套体系(节点方法用单独的 `node` 哨兵 scope,不在这六个里)。这六个 scope 各管一摊:

| Scope | 含义 | 典型方法 |
|---|---|---|
| `operator.read` | 只读:看状态、看会话、看日志 | `health`、`status`、`sessions.list`、`logs.tail` |
| `operator.write` | 改运行时状态:发消息、调工具、跑会话 | `sessions.send`、`tools.invoke`、`agent`、`chat.send` |
| `operator.admin` | 管控制面:改配置、增删会话/智能体、重启 | `config.set`、`sessions.delete`、`agents.create` |
| `operator.approvals` | 审批流:exec/plugin 审批请求与决议 | `exec.approval.request`、`plugin.approval.resolve` |
| `operator.pairing` | 配对:节点/设备的配对、令牌轮换与吊销 | `node.pair.approve`、`device.token.revoke` |
| `operator.talk.secrets` | 语音/talk 子系统的敏感凭据 | (talk 相关的敏感调用) |

最关键的一处设计在 `isOperatorScope`:

```typescript
const KNOWN_OPERATOR_SCOPES: ReadonlySet<OperatorScope> = new Set(KNOWN_OPERATOR_SCOPE_VALUES);

/** Narrows untrusted auth-token scope entries to the gateway's closed scope set. */
export function isOperatorScope(value: unknown): value is OperatorScope {
  return typeof value === "string" && KNOWN_OPERATOR_SCOPES.has(value as OperatorScope);
}
```

它的注释一针见血:**把不可信的认证令牌里的 scope 条目,收窄到网关那个封闭集合里。** 也就是说,即便一个令牌声称自己拥有 `"operator.superadmin"` 或任何花哨的字符串,只要不在这六个之中,就被当作不存在。**权限集合是封闭的、不可扩展的**——这是"约束只能收紧、不能放松"这条贯穿全书的原则在 scope 层面的体现。

### CLI 的默认 scope 兜底

`method-scopes.ts` 里有一个特殊常量:

```typescript
/** Default scopes granted to CLI/operator clients when no narrower local policy is known. */
export const CLI_DEFAULT_OPERATOR_SCOPES: OperatorScope[] = [
  ADMIN_SCOPE, READ_SCOPE, WRITE_SCOPE,
  APPROVALS_SCOPE, PAIRING_SCOPE, TALK_SECRETS_SCOPE,
];
```

这看起来像"全部放开",似乎与最小权限相悖。但要理解它的适用场景:**一个独立的 CLI/工具调用方,可能正在和一个本地进程里没有完整插件注册表的网关对话**。当 CLI 在本地算不出某个方法的精确 scope 需求时,与其"算少了"导致一个合法调用被错误拒绝,不如对**本机操作者**这种本就高度可信的身份给出完整默认集,把精确的鉴权判断留给服务端。注意这个兜底只在 CLI 路径、且本地无法分类该方法时才生效——它不是对所有调用方的后门。

---

## 10.4 方法表:策略与实现的唯一对账点

权限要落地,先得知道"每个方法需要哪个 scope"。OpenClaw 把这件事做成了一张**集中的、声明式的表**:`src/gateway/methods/core-descriptors.ts` 里的 `CORE_GATEWAY_METHOD_SPECS`。每一行就是一个核心方法的完整策略:

```typescript
export const CORE_GATEWAY_METHOD_SPECS = [
  { name: "health", scope: "operator.read" },
  { name: "config.get", scope: "operator.read" },
  { name: "config.set", scope: "operator.admin" },
  { name: "config.apply", scope: "operator.admin", controlPlaneWrite: true },
  { name: "sessions.list", scope: "operator.read", startup: true },
  { name: "sessions.send", scope: "operator.write", startup: true },
  { name: "sessions.delete", scope: "operator.admin" },
  { name: "agent", scope: "operator.write" },
  { name: "plugins.sessionAction", scope: "dynamic" },
  { name: "skills.bins", scope: "node" },
  // ……约两百行
] as const;
```

这张表的注释定义了它的纪律:

> This is the canonical core method policy table: every core handler must appear here so listing, authorization, startup availability, and write throttling stay in sync.

也就是说,**列出方法、鉴权、启动期可用性、写操作限流——四件事共用同一张表**。这条纪律不是口号,它被一段代码强制执行。`createCoreGatewayMethodDescriptors` 在把 handler 映射成可派发的描述符时,会做一次双向核对:

```typescript
for (const name of Object.keys(handlers)) {
  if (!specNames.has(name)) {
    // Unclassified core handlers would bypass scope/startup/write metadata, so fail before the
    // dispatcher can expose a method with missing policy.
    throw new Error(`gateway method handler is missing a descriptor: ${name}`);
  }
}
```

**任何一个核心 handler 如果在表里找不到对应的策略条目,系统直接抛错、拒绝启动。** 这堵死了一类最隐蔽的安全退化:有人新增了一个 handler,却忘了给它配权限——在很多系统里,这样的方法会以"无人鉴权 = 谁都能调"的形式悄悄上线。OpenClaw 把它变成一个**启动期的硬失败**:没有策略,就不许存在。这正是"默认拒绝"在工程层面的极致——连"忘记声明"都被默认拒绝。

### 三种特殊 scope 哨兵

表里除了那六个 operator scope,还出现了两个特殊值,它们定义在 `descriptor.ts`:

```typescript
/** Scope marker for methods that only authenticated node clients may call. */
export const NODE_GATEWAY_METHOD_SCOPE = "node" as const;
/** Scope marker for methods whose handler derives the required operator scope at runtime. */
export const DYNAMIC_GATEWAY_METHOD_SCOPE = "dynamic" as const;
```

- **`node`**:这个方法只给认证过的**节点客户端**用,不属于 operator 体系。像 `node.pending.pull`、`node.invoke.result`、`skills.bins` 都是节点角色专用——所以 `resolveCoreOperatorGatewayMethodScope` 会显式把 `node` 和 `dynamic` 排除,绝不让它们混进 operator 的 scope 解析。
- **`dynamic`**:这个方法的所需 scope **由 handler 在运行时根据参数算**,而不是静态钉死。典型例子是 `plugins.sessionAction`——一个插件注册的会话动作可能要 read、可能要 write,得看具体是哪个动作。

### 还有一些方法是"隐藏"的

注意表尾那些 `advertise: false` 的方法:`sessions.get`、`sessions.resolve`、`connect`、`chat.inject`、`nativeHook.invoke`、`web.login.start` 等等。它们仍然是合法的、有完整 scope 策略的方法,但**不会出现在对外广播的方法清单里**(由 `listCoreAdvertisedGatewayMethodNames` 过滤掉)。这是"能力存在但不张扬"的取舍:内部/兼容用途的方法不必暴露给外部客户端的方法发现机制,减少攻击面,同时不影响内部调用。

### `startup` 标记:启动期的诚实

表里还有 `startup: true`(如 `sessions.list`、`sessions.send`、`models.list`、`agent.wait`)。这些方法被收集进 `STARTUP_UNAVAILABLE_GATEWAY_METHODS`,在 sidecar 还没就绪时它们会**尽早出现在方法清单里、但返回可重试的"启动期不可用"错误**。这是"错误即文档"的又一次应用:与其让客户端在一个不存在的方法上困惑,不如告诉它"这个方法存在、只是还没准备好,稍后重试"。

---

## 10.5 鉴权判定:admin 永远放行,未分类一律拒绝

有了 scope 模型和方法表,判定逻辑就水到渠成。`method-scopes.ts` 里的 `authorizeOperatorScopesForMethod` 是真正下判决的地方,它的骨架是:

```typescript
export function authorizeOperatorScopesForMethod(method, scopes, params) {
  // 1. admin 是万能钥匙:有 admin 就放行,不必逐方法判断
  if (scopes.includes(ADMIN_SCOPE)) {
    return { allowed: true };
  }
  // 2. dynamic 方法:根据 params 找到插件动作的精确 scope 需求再判
  if (isDynamicOperatorGatewayMethod(method)) {
    /* ……解析 pluginId/actionId,取该动作的 requiredScopes …… */
  }
  // 3. 普通方法:未分类则要求 admin(= 默认拒绝)
  const requiredScope = resolveRequiredOperatorScopeForMethod(method) ?? ADMIN_SCOPE;
  // 4. read 可以被 write 满足(write 比 read 强)
  if (requiredScope === READ_SCOPE) {
    if (scopes.includes(READ_SCOPE) || scopes.includes(WRITE_SCOPE)) {
      return { allowed: true };
    }
    return { allowed: false, missingScope: READ_SCOPE };
  }
  // 5. 其余:必须精确持有所需 scope
  if (scopes.includes(requiredScope)) {
    return { allowed: true };
  }
  return { allowed: false, missingScope: requiredScope };
}
```

这段代码里藏着三条值得拎出来的设计决策:

**第一,`ADMIN_SCOPE` 是顶层万能钥匙。** 持有 admin 就直接放行,不再逐方法核对。这呼应了第 9 章的那条线索——某些 admin-only 方法被"钉死在 `ADMIN_SCOPE` 上"。在这套体系里,"钉死在 admin"有两层含义:方法表里它的 `scope` 写的就是 `operator.admin`(如 `config.set`、`sessions.delete`);而 admin 这把钥匙又能开所有锁。换句话说,admin 是权限格(lattice)的顶。

**第二,未分类方法默认要求 admin。** 看那个 `?? ADMIN_SCOPE`:如果一个方法在所有策略来源里都查不到 scope,它的所需权限就被当成 `ADMIN_SCOPE`——也就是"几乎没人能调"。这是**默认拒绝(default-deny)** 的精确实现。与之配套的是最小权限解析侧的另一半:

```typescript
export function resolveLeastPrivilegeOperatorScopesForMethod(method, params) {
  if (isDynamicOperatorGatewayMethod(method)) { /* …… */ }
  const requiredScope = resolveRequiredOperatorScopeForMethod(method);
  if (requiredScope) {
    return [requiredScope];
  }
  // Default-deny for unclassified methods.
  return [];
}
```

未分类方法解析出来的最小 scope 是**空数组**——调用方什么权限都凑不齐,自然进不去。一边"算不出权限就给空",一边"查不到策略就要 admin",两头夹击,未分类方法在 OpenClaw 里寸步难行。

**第三,`read` 可由 `write` 满足,但反过来不行。** 这是 scope 之间唯一的"向下兼容"关系:能写的人当然能读。除此之外没有任何隐式提权——approvals 不能当 write 用,pairing 不能当 admin 用。权限格是精心设计的,不是想当然的层级。

### scope 是怎么解析出来的:三级优先级

`requiredScope` 从哪来?`resolveScopedMethod` 给出了一条**有明确优先级的解析链**:

```typescript
function resolveScopedMethod(method: string): OperatorScope | undefined {
  // 1. 核心描述符最权威
  const explicitScope = resolveCoreOperatorGatewayMethodScope(method);
  if (explicitScope) return explicitScope;
  // 2. 其次是保留命名空间策略
  const reservedScope = resolveReservedGatewayMethodScope(method);
  if (reservedScope) return reservedScope;
  // 3. 最后才是活跃插件注册的描述符
  const pluginDescriptor = getPluginRegistryState()?.activeRegistry
    ?.gatewayMethodDescriptors?.find((d) => d.name === method);
  const pluginScope = pluginDescriptor?.scope;
  // node/dynamic 哨兵不算 operator scope
  return pluginScope === "node" || pluginScope === "dynamic" ? undefined : pluginScope;
}
```

**核心 > 保留命名空间 > 插件**。核心方法表说了算,插件只能在核心没占用的名字上声明 scope。这条优先级保证了**插件无法重定义核心方法的权限**——它顶多在自己的命名空间里玩。

### 保留命名空间:把危险前缀强制拉回 admin

第二级的"保留命名空间策略"值得单独看。`src/shared/gateway-method-policy.ts` 定义了一组**保留给 admin 的方法前缀**:

```typescript
const RESERVED_ADMIN_GATEWAY_METHOD_PREFIXES = [
  "exec.approvals.", "config.", "wizard.", "update.",
] as const;
```

任何以这些前缀开头的方法都被强制要求 `operator.admin`。更关键的是 `normalizePluginGatewayMethodScope`——它会把插件**自己声明的** scope 在这些保留前缀上**强行纠偏**回 admin:

```typescript
export function normalizePluginGatewayMethodScope(method, scope) {
  const reservedScope = resolveReservedGatewayMethodScope(method);
  if (!reservedScope || !scope || scope === reservedScope) {
    return { scope, coercedToReservedAdmin: false };
  }
  // 插件想用更弱的 scope 占用保留前缀?不行,拉回 admin。
  return { scope: reservedScope, coercedToReservedAdmin: true };
}
```

设想一个恶意或粗心的插件注册了一个叫 `config.backdoor` 的方法,并声明它只需要 `operator.read`。如果没有这道防线,任何只读操作者都能调它。有了 `normalizePluginGatewayMethodScope`,这个声明会被**悄悄但坚决地纠正**为 `operator.admin`,并打上 `coercedToReservedAdmin: true` 的标记。**插件可以扩展系统,但不能借扩展之名削弱核心的安全边界**——这又是一次"约束只能收紧"的实践。

---

## 10.6 不启动服务器也能用:嵌入式网关上下文

到这里你可能会问:网关听起来像是一个跑在某个端口上的服务器进程。那当我在本地直接跑一条 CLI 命令、或者一个本地 agent 路径想调 `sessions.list` 时,难道每次都要先起一个完整的网关服务器?

不需要。`src/gateway/local-request-context.ts` 给出了**嵌入式(embedded)**方案。它的文件头注释说得很清楚:

> Lets local agent paths reuse Gateway server methods without starting a server.

`createLocalGatewayRequestContext` 会构造一个**最小可用的 `GatewayRequestContext`**——只填上本地执行真正需要的字段(配置读取、模型目录加载、会话事件订阅、运行缓冲区等),让本地路径能直接复用那些 server 方法 handler,而不必走网络、不必监听端口。这意味着**同一套方法 handler、同一张权限表,既服务于远程 WebSocket 客户端,也服务于本地嵌入式调用**——逻辑只有一份,行为完全一致。

但"最小"不等于"假装支持一切"。注意它对 cron 子系统的处理:

```typescript
function cronUnavailable(): never {
  throw new Error("Cron is unavailable in local embedded agent gateway context.");
}
const unavailableCron: CronServiceContract = {
  start: async () => { cronUnavailable(); },
  list: async () => cronUnavailable(),
  add: async () => cronUnavailable(),
  // ……每个方法都显式抛错
};
```

它的注释解释了为什么要这么"凶":

> Unsupported subsystems fail loudly so local command paths do not silently enqueue cron/channel work.

嵌入式上下文里没有真正的 cron 调度器。与其给一个**静默无操作(silent no-op)**的桩——让调用方以为自己排了一个定时任务、实际上石沉大海——不如让每一次调用都**大声抛错**。这是"快速失败 / 大声失败"原则的典范:**一个会骗人的成功,比一个诚实的失败危险得多。** 这也呼应了第 9 章里子智能体的工具 deny list——子智能体的 `cron` 工具被禁,而这里嵌入式上下文的 cron 直接抛错,两道防线指向同一个判断:某些能力在某些上下文里就是不该存在,要禁就禁得干净利落。

---

## 10.7 会话路由:从 run id 到 session key

网关把请求派发到了某个方法 handler,但很多方法(发消息、看历史、续跑)需要回答一个更具体的问题:**这次请求到底作用在哪一个会话上?** 这就是会话路由要解决的事。

OpenClaw 的会话标识有一套结构化的格式。`src/routing/session-key.ts` 定义了它的形状。最规范的形态是 **agent 键**:`agent:<agentId>:<rest>`,比如一个智能体的主会话是:

```typescript
export function buildAgentMainSessionKey(params: { agentId: string; mainKey?: string }): string {
  const agentId = normalizeAgentId(params.agentId);
  const mainKey = normalizeMainKey(params.mainKey);
  return `agent:${agentId}:${mainKey}`;  // 例如 "agent:main:main"
}
```

而 `classifySessionKeyShape` 会把任意一个会话键归入四种形态之一:

```typescript
export type SessionKeyShape = "missing" | "agent" | "legacy_or_alias" | "malformed_agent";
```

- `agent`:合法的 `agent:<id>:<rest>` 结构;
- `legacy_or_alias`:不带 agent 前缀的老格式或别名(会被 `scopeLegacySessionKeyToAgent` 补上 agent 作用域);
- `malformed_agent`:**号称**是 agent 键(以 `agent:` 开头)却解析失败的——这是要警惕的畸形输入;
- `missing`:空。

注意 `agentId` 的合法性约束:

```typescript
const VALID_ID_RE = /^[a-z0-9][a-z0-9_-]{0,63}$/i;
```

这个正则你应该眼熟——它和第 9 章里校验子智能体 `agentId` 的那条规则**一模一样**。原因很简单:agentId 会进入会话键、会进入文件路径。`normalizeAgentId` 的注释直说了:"Keep it path-safe + shell-friendly."(保持路径安全、shell 友好)。一个能进入文件系统路径的标识符,如果允许任意字符,就是一个潜在的路径穿越/注入入口。**同一条命名规则在子智能体生成和会话路由两处复用,不是巧合,而是同一个安全不变量在不同子系统的两次落地。**

### run id ↔ session key 的桥接

活跃的 agent 运行用 `runId` 标识,而持久化的会话用 `sessionKey` 标识。两者之间需要一座桥,这就是 `src/gateway/server-session-key.ts` 里 `resolveSessionKeyForRun` 的活儿:给一个 run id,找出对应的、面向调用方的 session key。

它的实现里有一处很能体现工程成熟度的细节——**带 TTL 的负缓存**:

```typescript
const RUN_LOOKUP_MISS_TTL_MS = 1_000;
// ……
if (sessionKey === null) {
  // Negative caching avoids repeated full-store scans while still allowing
  // a just-created run/session pair to appear shortly after the first lookup.
  const missExpiresAt = resolveExpiresAtMsFromDurationMs(RUN_LOOKUP_MISS_TTL_MS);
  // ……
}
```

正命中(找到了)可以**长期缓存**——run 和 session 的对应关系一旦建立就是稳定的。但**未命中(没找到)只缓存 1 秒**。为什么?因为一个刚刚创建的 run/session 对,可能在你第一次查的瞬间还没写进存储;如果把"没找到"也长期缓存,这个新会话就会在缓存过期前一直"查无此人"。短 TTL 的负缓存,既避免了每次未命中都全量扫描会话存储(性能),又保证了新会话能在一秒内变得可见(正确性)。这是缓存设计里"正负命中区别对待"的经典权衡。

还有一处边界处理也值得一提——它返回的不是原始存储键,而是**面向调用方的请求键**:

```typescript
function resolveRunSessionKeyForCaller(storeKey: string) {
  return toAgentRequestSessionKey(storeKey) ?? storeKey;
}
```

注释解释了原因:HTTP/RPC 客户端会拿这个返回值在后续的会话操作里复用,所以必须返回它们认得、用得上的形态,而不是内部存储用的规范化键。**内部表示和对外契约的分离**,在这里体现得很干净。

---

## 10.8 传输层硬化:把 override 当作不可信重定向

最后,我们回到 `call.ts`,看它在连接建立这一层做了什么安全防护。这里有一个非常值得学习的安全思维样本——`ensureExplicitGatewayAuth`,它的注释把威胁模型讲得明明白白:

> URL overrides are untrusted redirects and can move WebSocket traffic off the intended host. Never allow an override to silently reuse implicit credentials or device token fallback.

翻译过来:**URL override 是不可信的重定向,它可能把 WebSocket 流量引到一个非预期的主机上。绝不允许一个 override 静默地复用隐式凭证或设备令牌兜底。**

想象一下攻击场景:你的隐式凭证(比如本地配置里存的设备令牌)本来是用来连接你信任的那台网关的。如果某处的 URL override 能在你不知情的情况下把目标主机改掉,同时又**自动带上**你的隐式凭证——那你的凭证就被送到了攻击者的主机上。OpenClaw 的防线是:**只要目标地址是被 override 的(无论来自 CLI 参数还是环境变量),就必须有显式认证**。CLI override 要求显式的 token/password;环境变量 override 要求已解析出的认证信息。凑不齐?抛出 `GatewayExplicitAuthRequiredError`,宁可让这次调用失败,也绝不静默地把凭证送往一个未经显式确认的地方。

这就是 10.2 里那个 `GatewayExplicitAuthRequiredError` 错误类的来历。它不是一个普通的"认证失败",而是一个**专门针对"重定向 + 隐式凭证"这一组合威胁**的拒绝。

### 错误即文档:code 1006 的诊断提示

`call.ts` 还有一处很有"人味"的设计——`formatGatewayCloseError`。当 WebSocket 以 code `1006`(异常关闭,通常意味着连接根本没握上手)断开时,它不会只甩给你一个干巴巴的错误码,而是附上排障线索:网关可能还没就绪、TLS 配置可能不匹配、网关可能崩了——并建议你 "Run `openclaw doctor`"。

这是全书反复出现的"**错误即文档(error-as-documentation)**"理念在传输层的又一次落地。一个 1006 对新手来说几乎不可解读;但一条"这通常意味着 A/B/C,试试运行诊断命令"的提示,直接把用户从困惑里捞出来。**好的错误信息不是事后追责,而是即时的、上下文相关的帮助。**

---

## 本章小结

这一章我们拆开了 OpenClaw 的请求总线——网关,以及它背后的会话路由。把零散的细节收成几条主线:

1. **唯一入口收敛一切。** 所有跨边界请求都经由 `callGateway`(`call.ts`)进入,按 CLI / 显式 scopes / 最小权限三条路径分流。唯一入口是一切安全约束得以成立的前提——这也是第 9 章里 `gateway` 工具必须被子智能体禁用的根本原因。
2. **权限集合是封闭的六个 scope。** `operator-scopes.ts` 把权限钉死在 read/write/admin/approvals/pairing/talk.secrets 六个常量上,`isOperatorScope` 把任何不在集合里的令牌 scope 收窄掉。约束只能收紧,不能放松。
3. **一张表对账四件事。** `core-descriptors.ts` 的 `CORE_GATEWAY_METHOD_SPECS` 让"列方法、鉴权、启动可用性、写限流"共用同一份声明;缺策略的 handler 会在启动期硬失败。
4. **默认拒绝,admin 封顶。** 未分类方法解析出空 scope、判定时要求 admin,两头夹击。admin 是权限格的顶;read 可由 write 满足,此外无隐式提权。保留命名空间(`config.`/`wizard.`/`update.`/`exec.approvals.`)还会把插件的弱声明强制纠偏回 admin。
5. **嵌入式上下文复用同一套逻辑。** `local-request-context.ts` 让本地路径无需起服务器即可复用 server 方法;不支持的子系统(如 cron)选择大声抛错而非静默无操作——会骗人的成功比诚实的失败更危险。
6. **会话路由用结构化的 session key。** `agent:<id>:<rest>` 格式 + 与子智能体共享的 `agentId` 命名规则(路径安全),配合 run→session 的带 TTL 负缓存桥接,把每次请求精确路由到目标会话。
7. **传输层把 override 当敌人。** `ensureExplicitGatewayAuth` 拒绝让重定向静默复用隐式凭证;`formatGatewayCloseError` 把 1006 翻译成可行动的排障提示。

如果说第 9 章讲的是"能力如何安全地繁殖",那么这一章讲的就是"请求如何安全地收敛"。两者其实是同一枚硬币的两面:一个系统要既强大又可控,就必须让能力的扩张走在收敛的轨道上——每一次调用,无论来自 CLI、远程节点、插件还是子智能体,最终都汇入同一条带着权限表的总线,被同一套规则分类、鉴权、派发。

---

## 动手实验

> 以下实验基于本章引用的源码文件,目的是把"读懂"变成"验证过"。建议在 OpenClaw 仓库(`git clone https://github.com/openclaw/openclaw.git`)的工作副本里进行。

### 实验一:给核心方法表做一次"权限普查"

打开 `src/gateway/methods/core-descriptors.ts`,统计 `CORE_GATEWAY_METHOD_SPECS` 里每种 scope 各有多少个方法。

- 用 `rg` 或脚本数一数 `scope: "operator.read"`、`"operator.write"`、`"operator.admin"` 各出现几次。
- 思考:为什么 `operator.read` 的方法数量远多于 `operator.admin`?这反映了怎样的权限分布哲学?
- 找出所有 `controlPlaneWrite: true` 的方法(如 `config.apply`、`update.run`、`gateway.restart.request`),它们有什么共性?为什么这些方法需要额外的"控制面写"标记?

### 实验二:亲手触发"缺策略 = 启动失败"

阅读 `createCoreGatewayMethodDescriptors` 的双向核对逻辑(`core-descriptors.ts` 末尾)。

- 在脑中(或在一个本地分支里)模拟:如果你新增一个 handler 叫 `mySecret.dump`,但**不**在 `CORE_GATEWAY_METHOD_SPECS` 里加对应条目,会发生什么?
- 找到那句 `throw new Error(\`gateway method handler is missing a descriptor: ${name}\`)`,确认它会在启动期阻止这个无策略方法上线。
- 反向实验:如果你只加了 spec 条目却**没有**提供 handler,代码走的是 `continue`(跳过),不会报错——想想为什么"有 handler 没策略"是致命的,而"有策略没 handler"是可容忍的?

### 实验三:验证保留命名空间的"强制纠偏"

阅读 `src/shared/gateway-method-policy.ts` 的 `normalizePluginGatewayMethodScope`。

- 写一个小测试场景(纸上推演即可):一个插件注册方法 `config.evil`,声明 `scope: "operator.read"`。追踪它经过 `normalizePluginGatewayMethodScope` 后的返回值。
- 确认返回的 `scope` 被改成了 `"operator.admin"`,且 `coercedToReservedAdmin` 为 `true`。
- 再试 `exec.approvals.bypass`、`wizard.hijack`、`update.sneaky`——验证四个保留前缀都会触发纠偏。
- 思考:为什么这个纠偏是"悄悄但坚决"(不报错、只打标记)而不是直接拒绝注册?(提示:`coercedToReservedAdmin` 这个标记的存在,意味着上层可以选择记录/告警,而不必让整个插件加载失败。)

### 实验四:观察 run→session 负缓存的 TTL 行为

阅读 `src/gateway/server-session-key.ts` 的 `resolveSessionKeyForRun` 与 `setResolvedSessionKeyCache`。

- 画一条时间线:t=0 查询一个尚未持久化的 run id(返回 `undefined`,负缓存写入,TTL=1s);t=0.5s 再查(命中负缓存,仍返回 `undefined`,**不**扫描存储);t=1.5s 再查(负缓存已过期,删除后重新全量扫描,此时若会话已写入则正命中并长期缓存)。
- 思考:如果把 `RUN_LOOKUP_MISS_TTL_MS` 设成 0(等于不做负缓存)会怎样?设成 60000(60 秒)又会带来什么问题?
- 进阶:把这套"正命中长缓存、负命中短缓存"的模式,对照你自己项目里任何一处"查不到就反复打后端"的代码——能不能用同样的思路省掉一批无谓的扫描?

---

> 下一章,我们将转向 OpenClaw 的**配置系统与运行时重载**——这条总线上跑的方法,有相当一部分(还记得那些 `config.*` 和 `controlPlaneWrite: true` 的方法吗?)就是在读写系统的"活配置"。我们会看到 OpenClaw 如何让一个长期运行的进程在不重启的前提下安全地吞下新配置:diff 出哪些变了、哪些子系统需要重启、哪些可以热替换,以及当一份坏配置进来时,系统如何回滚到上一个已知良好的状态——又一次"快速失败 + 可恢复"的演练。
