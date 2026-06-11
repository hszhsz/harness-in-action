# 第 9 章 子智能体与多智能体协作——一棵会自我繁殖的树如何不失控

> 上一章我们看到 OpenClaw 如何把最危险的工具(exec)关进笼子:七道防御闸门、Docker 沙箱、配置校验,层层设防。但还有一种"危险"是结构性的、不属于任何单一工具——当一个智能体可以**生出另一个智能体**,而那个智能体又可以继续生下去时,系统就从"一个进程"变成了"一棵会自我繁殖的树"。这棵树如果不加约束,会在三个维度上失控:深度(无限递归)、广度(无限并发)、权限(子代继承甚至放大父代能力)。本章解构 OpenClaw 的子智能体(subagent)子系统,看它如何用**角色分层、五道生成闸门、能力收敛、可观测的生命周期**这四件事,让一棵自我繁殖的树始终长在可控的范围内。

## 9.1 为什么需要子智能体:从"一个人干活"到"一个团队干活"

一个串行的智能体面对复杂任务时只有一种节奏:读一点、想一点、做一点,再读一点。当任务可以拆分成互不依赖的子任务时——比如"分别调研三个竞品""并行重构五个模块"——串行执行就成了瓶颈。

OpenClaw 的答案是允许一个**主智能体(main agent)**通过 `sessions_spawn` 工具派生出**子智能体(subagent)**,把子任务分发下去并行执行,最后汇总结果。这本质上是把"一个人干活"升级成"一个团队干活"。但团队协作引入了三个新问题,而这三个问题恰好对应本章的三条主线:

- **层级问题**:谁能派生?派生出来的还能不能再派生?(角色与深度)
- **资源问题**:一个智能体能同时养多少个孩子?会不会派生出无限多个进程把机器拖垮?(广度闸门)
- **权限问题**:子智能体能用主智能体的所有工具吗?它会不会偷偷给老板发消息、自己批准自己的危险操作?(能力收敛)

让我们从最基础的"角色"开始。

## 9.2 三种角色与深度:main / orchestrator / leaf

OpenClaw 给会话树上的每个节点定义了三种角色,定义在 `src/agents/subagent-capabilities.ts`:

```typescript
export type SubagentSessionRole = "main" | "orchestrator" | "leaf";
```

角色不是配置出来的,而是由节点在树中的**深度(depth)**推导出来的。推导逻辑只有三行,却定义了整棵树的形状:

```typescript
function resolveSubagentRoleForDepth(params: {
  depth: number;
  maxSpawnDepth?: number;
}): SubagentSessionRole {
  const depth = resolveNonNegativeIntegerOption(params.depth, 0);
  const maxSpawnDepth = resolveIntegerOption(
    params.maxSpawnDepth,
    DEFAULT_SUBAGENT_MAX_SPAWN_DEPTH,
    { min: 1 },
  );
  if (depth <= 0) {
    return "main";
  }
  return depth < maxSpawnDepth ? "orchestrator" : "leaf";
}
```

翻译成人话:

- **depth = 0 → main**:树根。这是直接面对用户的主智能体。
- **0 < depth < maxSpawnDepth → orchestrator(协调者)**:中间节点。它是某个智能体派生出来的,但它自己还没到深度上限,所以**还能继续派生孙子**。
- **depth >= maxSpawnDepth → leaf(叶子)**:最底层的工人。它已经到了深度上限,**不能再派生**,只能埋头干活。

角色一旦确定,"控制范围(control scope)"也随之确定:

```typescript
function resolveSubagentControlScopeForRole(role: SubagentSessionRole): SubagentControlScope {
  return role === "leaf" ? "none" : "children";
}
```

叶子节点的控制范围是 `none`——它管不了任何人(因为它本来就没有孩子);main 和 orchestrator 的控制范围是 `children`——它们可以列出、终止、引导自己派生的子代。最终,`resolveSubagentCapabilities` 把这些推导汇总成一组布尔能力:

```typescript
export function resolveSubagentCapabilities(params: { depth: number; maxSpawnDepth?: number }) {
  const depth = resolveNonNegativeIntegerOption(params.depth, 0);
  const role = resolveSubagentRoleForDepth(params);
  const controlScope = resolveSubagentControlScopeForRole(role);
  return {
    depth,
    role,
    controlScope,
    canSpawn: role === "main" || role === "orchestrator",
    canControlChildren: controlScope === "children",
  };
}
```

这里有一个很关键的设计哲学:**能力不是开关,而是位置的函数**。你不能通过配置"赋予某个叶子节点派生权限",因为 `canSpawn` 直接由 `role` 决定,而 `role` 直接由 `depth` 决定。深度是树本身的结构属性,无法伪造。这就把"递归深度失控"这个最致命的问题,从"运行时要不要拦截"前移成了"结构上根本不可能"。

### 9.2.1 一个微妙的细节:从持久化信封恢复角色

上面的推导是"理想情况"——假设我们总能从会话存储里读到准确的深度。但有一类会话不那么听话:ACP(Agent Client Protocol)运行时派生的子智能体,它可能被**恢复(resume)**,而恢复出来的会话键里不一定带着深度信息。

`resolveStoredSubagentCapabilities` 处理了这种情况。它先尝试读取持久化"信封(envelope)"里显式存储的 `subagentRole` 和 `subagentControlScope`,如果信封里没有,再沿着 `spawnedBy`(谁派生了我)这条链一路往上追溯父节点的身份:

```typescript
const storedRole = normalizeSubagentRole(entry?.subagentRole);
const storedControlScope = normalizeSubagentControlScope(entry?.subagentControlScope);
const fallback = resolveSubagentCapabilities({ depth, maxSpawnDepth });
const role = storedRole ?? fallback.role;
const controlScope = storedControlScope ?? resolveSubagentControlScopeForRole(role);
```

追溯父链时用了一个 `visited` 集合来防止环:

```typescript
if (!normalizedSessionKey || visited.has(normalizedSessionKey)) {
  return false;
}
visited.add(normalizedSessionKey);
```

如果某个被篡改或损坏的信封形成了 A→B→A 的循环引用,`visited` 会在第二次访问 A 时立刻返回 `false`,而不是无限递归把栈撑爆。**这是"防御性编程"的一个典型:即使输入数据已经损坏到形成环,代码也只是放弃追溯,而不是崩溃。**

## 9.3 五道生成闸门:`spawnSubagentDirect`

角色分层解决了"结构上的递归失控",但真正的派生动作发生在 `src/agents/subagent-spawn.ts` 的 `spawnSubagentDirect` 函数里。这是整个子系统的核心执行器(1700 多行),也是把所有约束落地的地方。每一次派生请求,都要依次通过五道闸门。这一节我们逐一拆解。

### 闸门一:agentId 必须合法

子智能体可以指定运行在哪个"智能体身份(agentId)"下。这个 ID 会被用来构造会话键、工作目录路径。如果不校验,一个恶意或错误的 ID 就可能制造出幽灵工作目录、甚至级联的 cron 循环:

```typescript
if (requestedAgentId && !isValidAgentId(requestedAgentId)) {
  return {
    status: "error",
    error: `Invalid agentId "${requestedAgentId}". agentId must match [a-z0-9][a-z0-9_-]{0,63}`,
  };
}
```

合法的 agentId 必须匹配 `[a-z0-9][a-z0-9_-]{0,63}`:以小写字母或数字开头,后面只能是小写字母、数字、下划线、连字符,总长不超过 64。源码注释明确点出这道闸门要防的是什么(对应内部 issue #31311):防止用奇怪字符制造出幽灵工作目录和级联 cron 循环。**这又是"输入即攻击面"原则的体现——任何会变成路径或会话键的字符串,都要先过正则。**

### 闸门二:深度限制

```typescript
const callerDepth = getSubagentDepthFromSessionStore(requesterInternalKey, { cfg });
if (callerDepth >= maxSpawnDepth) {
  return {
    status: "forbidden",
    error: `Subagent spawning is not allowed at this depth (current depth: ${callerDepth}, max: ${maxSpawnDepth})`,
  };
}
```

注意这里检查的是**调用者(caller)**的深度,而不是即将派生的子代深度。逻辑是:如果你自己已经在深度上限了(你是叶子),你就根本没资格发起派生。这与 9.2 的 `canSpawn` 是一致的——叶子不能生孩子,这里是运行时的二次确认。两层防御:第一层是工具压根不暴露给叶子(见 9.6 的 deny list),第二层是即使工具被调到了,执行器也会拒绝。

### 闸门三:并发子代数量限制

深度管住了"树有多高",广度则要管住"树有多宽":

```typescript
const activeChildren = countActiveRunsForSession(requesterInternalKey);
if (activeChildren >= maxChildren) {
  return {
    status: "forbidden",
    error: `Subagent spawning blocked: reached max active children (${activeChildren}/${maxChildren})`,
  };
}
```

`countActiveRunsForSession`(定义在 `subagent-registry.ts`)统计当前会话还有多少个活跃的子运行。如果达到 `maxChildrenPerAgent` 配置的上限,新的派生就被拒绝。**这道闸门的存在,意味着一个智能体即使逻辑出错、反复调用 `sessions_spawn`,也无法把机器的进程/内存吃光——它最多养这么多孩子。**

错误信息里特意带上了 `(${activeChildren}/${maxChildren})` 这个比值。这是本系列反复强调的"错误即文档":智能体看到 `(8/8)` 就立刻知道是配额满了,可能需要等已有的孩子完成;看到 `(0/8)` 配额却被拒就知道是别的问题。错误信息本身就是给 LLM 看的调试线索。

### 闸门四:目标策略(allowAgents)

默认情况下,一个智能体只能派生**自己身份**的子代(self-spawn)。但有时你希望主智能体能派生不同身份的专家子代(比如一个 "researcher" 身份、一个 "coder" 身份)。这由 `allowAgents` 配置控制,校验逻辑在 `src/agents/subagent-target-policy.ts`:

```typescript
export function resolveSubagentTargetPolicy(params: {
  requesterAgentId: string;
  targetAgentId: string;
  requestedAgentId?: string;
  allowAgents?: readonly string[];
  configuredAgentIds?: readonly string[];
}): SubagentTargetPolicyResult {
  const requesterAgentId = normalizeAgentId(params.requesterAgentId);
  const targetAgentId = normalizeAgentId(params.targetAgentId);
  if (!params.requestedAgentId?.trim() && targetAgentId === requesterAgentId) {
    return { ok: true };
  }
  // ... 否则必须命中 allowAgents 允许列表
}
```

第一行 fast-path 很关键:**如果你没有显式指定 agentId,且目标恰好就是你自己,直接放行**。这保证了最常见的"自己派生自己"场景零配置可用。只有当你想派生**别的身份**时,才需要 `allowAgents` 授权。

`allowAgents` 的语义还有一层精细设计。看 `resolveSubagentAllowedTargetIds`:

```typescript
if (policy.allowAny) {
  const configuredIds = Array.from(normalizeConfiguredAgentIds(params.configuredAgentIds));
  // ...
  return { allowAny: true, allowedIds: sortUniqueStrings(configuredIds) };
}
return {
  allowAny: false,
  allowedIds: filterConfiguredAllowedIds({
    allowedIds: policy.allowedIds,
    configuredAgentIds: params.configuredAgentIds,
  }).toSorted((a, b) => a.localeCompare(b)),
};
```

即使配置了 `allowAgents: ["*"]`(允许任意),允许列表也会和 `configuredAgentIds`(真实存在的、配置过的智能体)做**交集**。换句话说,`*` 不是"允许凭空捏造任何 ID",而是"允许任何**已经存在**的智能体"。这避免了通配符配置变成派生未注册幽灵智能体的后门。

而且错误信息区分了两种失败情形:

```typescript
if (allowed.allowAny || policy.allowedIds.includes(targetAgentId)) {
  return {
    ok: false,
    error: `agentId "${targetAgentId}" is not in the configured agent registry (allowed: ${allowedText})`,
  };
}
return {
  ok: false,
  error: `agentId is not allowed for sessions_spawn (allowed: ${allowedText})`,
};
```

第一种是"你这个 ID 被策略允许了,但它根本没在注册表里";第二种是"这个 ID 压根不在你的允许列表里"。两种诊断,对应两种不同的修复动作(去注册 vs 去改 allowAgents)。

### 闸门五:沙箱继承不可降级

最后一道闸门处理沙箱的继承关系。回想第 8 章,沙箱是把危险操作关进笼子的机制。如果一个被沙箱关着的智能体能派生出一个**不在沙箱里**的子代,那它就等于越狱了——把自己想干的危险事交给一个自由的孩子去干:

```typescript
if (!childRuntime.sandboxed && (requesterRuntime.sandboxed || sandboxMode === "require")) {
  return {
    status: "forbidden",
    error: requesterRuntime.sandboxed
      ? "Sandboxed sessions cannot spawn unsandboxed subagents."
      : 'sandbox="require" needs a sandboxed target runtime.',
  };
}
```

规则非常干脆:**被沙箱关着的会话,不能派生不在沙箱里的子代**;如果显式要求 `sandbox="require"`,目标运行时也必须是沙箱化的。这条规则保证了沙箱是**单向收敛**的——子代的安全级别只能等于或严于父代,绝不可能更松。这与第 8 章 exec 的 `minSecurity` 哲学完全一致:**约束只能收紧,不能放松**。

五道闸门全部通过后,执行器才会生成子代的会话键:

```typescript
const childSessionKey = `agent:${targetAgentId}:subagent:${crypto.randomUUID()}`;
```

会话键的结构本身就编码了血缘:`agent:<身份>:subagent:<随机UUID>`。`subagent` 这个段是身份证——后续任何代码看到这个键,都能立刻判断"这是个子智能体"(`isSubagentSessionKey`),从而施加子智能体专属的工具约束。

## 9.4 隔离还是分叉:isolated vs fork

派生时还有一个重要选择:子智能体要不要看到父智能体的对话历史?OpenClaw 提供两种**上下文模式(context mode)**:

- **isolated(隔离,默认)**:子代是一张白纸,只收到你给它的任务描述,看不到父代的任何历史。
- **fork(分叉)**:子代会**复制**父代当前的对话历史作为起点。

默认 isolated 是有道理的:大多数子任务是自包含的("去调研竞品 X"),给它一堆无关的父代历史只会污染上下文、浪费 token。只有当子代确实需要父代的完整对话背景才能理解任务时,才用 fork。

fork 有一个硬约束——只能在**同一个智能体身份**内分叉:

```typescript
// context="fork" currently requires the same target agent
```

道理也很直白:对话历史里可能包含父智能体特定的系统提示、工具调用记录,把它原样塞给一个不同身份的智能体会造成语义混乱。所以分叉只在"自己分叉给自己"时允许。

此外,在 ACP 运行时下,fork 和 `lightContext`(轻量引导上下文)都不被支持,会直接抛错:

```typescript
if (runtime === "acp" && lightContext) {
  throw new Error("lightContext is only supported for runtime='subagent'.");
}
if (runtime === "acp" && context === "fork") {
  throw new Error('context="fork" is only supported for runtime="subagent".');
}
```

这是"用类型和早期校验把不支持的组合拦在门外"的做法——与其让一个无意义的参数组合在深处产生诡异行为,不如在入口处直接说清楚"这个组合不支持"。

## 9.5 运行模式与生命周期:run vs session,以及"谁拥有谁"

派生还有最后一个维度:**运行模式(spawn mode)**。

- **run(一次性)**:派生一个子代,让它跑完一个任务就结束。这是默认。
- **session(持久会话)**:子代是一个长期存在的会话,可以多轮交互。但 session 模式有前置条件——必须绑定到一个聊天线程(`thread=true`):

```typescript
if (spawnMode === "session" && !requestThreadBinding) {
  return { status: "error", error: /* session 模式需要线程绑定 */ };
}
```

`sessions_spawn` 工具的 schema 里也体现了这个联动:`thread=true` 会让 `mode` 默认变成 `"session"`。

### 9.5.1 把孩子记进账本:`registerSubagentRun`

派生成功后,执行器会通过网关发起一个 `"agent"` 调用真正启动子代的运行,然后把这次运行登记进**注册表(registry)**:

```typescript
registerSubagentRun({
  runId,
  childSessionKey,
  controllerSessionKey,
  requesterSessionKey,
  task,
  taskName,
  cleanup,
  spawnMode,
  // ...
});
```

注册表是整个子系统的"账本",它记录了每一次运行的归属关系。这里有两个不同的"所有者"概念,初看容易混淆,但区分得很清楚:

- **controllerSessionKey(控制者)**:谁有权终止、引导这个子代。
- **requesterSessionKey(请求者)**:子代完成后,结果应该汇报给谁。

大多数情况两者相同,但在 cron 任务、hook 触发等场景下,"发起派生的会话"和"应该接收完成通知的会话"可能不是同一个——把它们拆成两个字段,系统就能灵活地把控制权和汇报路径分开路由。

### 9.5.2 谁能控制谁:所有权检查

注册表登记的归属关系,在 `src/agents/subagent-control.ts` 里被严格执行。任何"终止/引导/给子代发消息"的操作,都要先过 `ensureControllerOwnsRun`:

```typescript
function ensureControllerOwnsRun(params: {
  controller: ResolvedSubagentController;
  entry: SubagentRunRecord;
}) {
  const owner = params.entry.controllerSessionKey?.trim() || params.entry.requesterSessionKey;
  if (owner === params.controller.controllerSessionKey) {
    return undefined;
  }
  return "Subagents can only control runs spawned from their own session.";
}
```

一句话:**你只能控制你自己派生的子代**。你不能终止别人的孩子,也不能给别人的孩子发指令。

而叶子节点连"控制"这个动作本身都被禁止:

```typescript
if (params.controller.controlScope !== "children") {
  return {
    status: "forbidden",
    error: "Leaf subagents cannot control other sessions.",
  };
}
```

还有一个细节防止了自我引用的死结——子代不能引导自己:

```typescript
if (params.controller.callerSessionKey === params.entry.childSessionKey) {
  return {
    status: "forbidden",
    error: "Subagents cannot steer themselves.",
  };
}
```

### 9.5.3 级联终止:杀死一个节点等于杀死整棵子树

当你终止一个子代时,如果它自己又派生了孙子,这些孙子也必须被清理,否则就成了孤儿进程。`cascadeKillChildren` 用递归实现了这一点,并用 `seenChildSessionKeys` 集合防止重复访问:

```typescript
const cascade = await cascadeKillChildren({
  cfg: params.cfg,
  parentChildSessionKey: childKey,
  cache: params.cache,
  seenChildSessionKeys,
});
killed += cascade.killed;
labels.push(...cascade.labels);
```

终止操作的返回信息也很贴心地告诉你连带杀了多少后代:

```typescript
const cascadeText =
  cascade.killed > 0 ? ` (+ ${cascade.killed} descendant${cascade.killed === 1 ? "" : "s"})` : "";
```

让用户看到 `killed researcher (+ 3 descendants)`,而不是默默地杀掉一片进程却只报告杀了一个。**可观测性贯穿了整个生命周期:从派生(注册)、到运行(状态)、到终止(级联报告)。**

### 9.5.4 完成靠"推",不靠"轮询":sessions_yield

多智能体协作里有个经典的反模式:父代派生了孩子之后,陷入一个 `while (没完成) { sleep; 查状态 }` 的轮询循环,白白烧掉一轮又一轮的推理。OpenClaw 用**推送式完成(push-based completion)**根除了这个模式。

机制的核心是 `sessions_yield` 工具(`src/agents/tools/sessions-yield-tool.ts`),它的描述只有一句话:

> "End current turn. Use after spawning subagents; results arrive as next message."

它做的事情极简——记录"我要让出本轮"的意图,然后由运行时真正执行暂停:

```typescript
// The runtime owns the actual pause/end-turn behavior; this tool records intent.
await opts.onYield(message);
return jsonResult({ status: "yielded", message });
```

派生完孩子后,父代调用 `sessions_yield` 主动结束当前回合并进入等待。当孩子完成时,它的结果会作为一条**新的用户消息**自动"宣告(announce)"回父代,从而把父代唤醒。这就把"父代主动轮询"翻转成了"孩子完成时主动推送"。

子智能体的系统提示(`src/agents/subagent-system-prompt.ts`)里反复强调这条纪律:

> "Auto-announce is push-based. After spawning children, do NOT call sessions_list, sessions_history, exec sleep, or any polling tool."
>
> "If required completions have not arrived yet and `sessions_yield` is available, call it to end the turn and wait for completion events as user messages."

甚至连 `subagents` 工具(列出子代状态)的描述都在劝阻轮询:

> "If sessions_yield exists, use it for completion; do not poll wait loops."

**这是一个值得学习的设计:与其相信 LLM 会"聪明地不去轮询",不如从工具描述、系统提示、运行时机制三个层面同时把推送式完成钉死。** 提示词负责引导,机制负责兜底。

## 9.6 能力收敛:子智能体不能做的事

现在到了本章承接第 8 章预告的核心——子智能体的**工具权限收敛**。一个子智能体绝不能拥有和主智能体一模一样的工具集,否则前面所有的层级和闸门都会被绕过。约束定义在 `src/agents/agent-tools.policy.ts`,分成两张拒绝清单。

### 第一张清单:无论深度,永远禁止

```typescript
const SUBAGENT_TOOL_DENY_ALWAYS = [
  // System admin - dangerous from subagent
  "gateway",
  "agents_list",
  // Status/scheduling - main agent coordinates
  "session_status",
  "cron",
  // Direct session sends - subagents communicate through announce chain
  "sessions_send",
];
```

这五个工具,**任何深度的子智能体都用不了**。每一个禁令都有清晰的理由(源码注释直接写出来了):

- `gateway` / `agents_list`:系统级管理工具,子智能体握有它们太危险——一个子代不该能直接操作网关或枚举所有智能体。
- `session_status` / `cron`:状态查询和定时调度是**主智能体的协调职责**,子代越俎代庖会破坏协调一致性。
- `sessions_send`:子智能体之间不该直接互发消息,**它们只能通过"宣告链(announce chain)"沟通**——也就是 9.5.4 讲的"完成时自动汇报给父代"。禁掉直接发送,就堵死了子代之间私下串联、绕过父代协调的可能。

### 第二张清单:叶子节点额外禁止

```typescript
const SUBAGENT_TOOL_DENY_LEAF = [
  "subagents",
  "sessions_list",
  "sessions_history",
  "sessions_spawn",
];
```

这四个工具只对**叶子节点**额外禁止。它们都是"管理子代"用的——既然叶子没有孩子,这些工具对它毫无意义,干脆拿掉。最关键的是 `sessions_spawn`:**叶子节点连"派生"这个工具都看不到**。这就是 9.3 闸门二"深度限制"的第一层防御——工具压根不暴露,根本调不到。

合并逻辑由 `resolveSubagentDenyList` 完成:

```typescript
function resolveSubagentDenyList(depth: number, maxSpawnDepth: number): string[] {
  const isLeaf = depth >= Math.max(1, Math.floor(maxSpawnDepth));
  if (isLeaf) {
    return [...SUBAGENT_TOOL_DENY_ALWAYS, ...SUBAGENT_TOOL_DENY_LEAF];
  }
  // Orchestrator sub-agent: only deny the always-denied tools.
  return [...SUBAGENT_TOOL_DENY_ALWAYS];
}
```

逻辑一目了然:叶子节点拿到"永久禁止 + 叶子禁止"两张清单的并集;协调者(orchestrator)只受"永久禁止"约束,因为它还需要 `sessions_spawn`、`subagents` 这些工具来管理自己的孩子。

### 一个值得注意的逃生口

拒绝清单不是死的。看 `resolveSubagentToolPolicy` 里的合并逻辑:

```typescript
const explicitAllow = new Set(
  [...(allow ?? []), ...(alsoAllow ?? [])].map((toolName) => normalizeToolName(toolName)),
);
const deny = [
  ...baseDeny.filter((toolName) => !explicitAllow.has(normalizeToolName(toolName))),
  ...(Array.isArray(configured?.deny) ? configured.deny : []),
];
```

如果运维在配置里**显式 allow** 了某个本来被默认拒绝的工具,这个工具就会从默认 deny 列表里被**过滤掉**(`!explicitAllow.has(...)`)。这是一个有意保留的"逃生口":默认安全,但允许运维在明确知道风险的前提下放开特定工具。这与第 8 章 exec 的设计哲学呼应——**安全默认值(secure defaults)+ 显式覆盖(explicit override)**,而不是一刀切的硬编码禁止。

把这一切拼起来,子智能体的权限就是一个**逐层收敛的漏斗**:主智能体拥有完整工具集 → 协调者砍掉五个系统级工具 → 叶子再砍掉四个管理工具 → 加上沙箱继承、目标策略、深度/广度闸门。每往下一层,能力只减不增。**这正是把"自我繁殖的树"关进笼子的最后一道、也是最根本的一道锁:不是限制树长多大,而是限制每个节点能做什么,且越往下越受限。**

## 9.7 边界守护:sessions_spawn 工具不接受什么

最后值得一看的是 `sessions_spawn` 工具本身(`src/agents/tools/sessions-spawn-tool.ts`)如何在入口处拒绝那些"看起来合理但其实越界"的参数。

它维护了一张**不支持的参数清单**,凡是涉及"频道投递"的参数一律拒绝:

```typescript
const UNSUPPORTED_SESSIONS_SPAWN_PARAM_KEYS = [
  "target", "transport", "channel", "to",
  "threadId", "thread_id", "replyTo", "reply_to",
] as const;
```

碰到这些参数,直接报错并指明正路:

```typescript
throw new ToolInputError(
  `sessions_spawn does not support "${unsupportedParam}". Use "message" or "sessions_send" for channel delivery.`,
);
```

这道防线的意义在于**职责分离**:`sessions_spawn` 的职责是"派生一个干活的子代",而"往某个频道/某个人发消息"是另一套工具(`message` / `sessions_send`)的职责。如果允许在派生时夹带投递参数,子代就可能被用作绕过权限的投递跳板。把这些参数硬性拒绝,就让 `sessions_spawn` 保持了单一、可审计的语义。

类似地,它还拒绝了 per-call 的超时参数,引导你去改配置:

```typescript
throw new ToolInputError(
  `sessions_spawn does not support per-call "${providedTimeoutParam}". ` +
  `Configure agents.defaults.subagents.runTimeoutSeconds instead.`,
);
```

**这又是"错误即文档":报错不只是说"不行",而是说"不行,你应该用 X"。** 对一个 LLM 调用者来说,这种带修复指引的错误信息,意味着它下一轮就能自我纠正,而不是反复撞同一堵墙。

## 本章小结

本章我们解构了 OpenClaw 的子智能体子系统,看它如何让"一棵会自我繁殖的树"始终长在可控范围内。核心思路可以归纳成四层防御:

1. **角色由结构决定,不由配置决定**。`main / orchestrator / leaf` 三种角色完全由节点深度推导(`depth <= 0 → main`,`< maxSpawnDepth → orchestrator`,否则 `leaf`),`canSpawn` 和 `canControlChildren` 是角色的函数。深度是树的结构属性,无法伪造,所以"递归失控"在结构上就不可能。

2. **派生要过五道闸门**。`spawnSubagentDirect` 依次校验:agentId 合法性(防幽灵目录)、深度限制(防无限递归)、并发子代数(防无限并发)、目标策略(`allowAgents` 与已注册智能体取交集)、沙箱继承(子代安全级别只能等于或严于父代)。

3. **生命周期全程可观测**。注册表用 `controllerSessionKey`(谁能控制)和 `requesterSessionKey`(结果汇报给谁)区分两种归属;控制操作必须通过所有权检查;终止会级联清理整棵子树并如实报告杀了多少后代;完成靠 `sessions_yield` 的推送式机制,而非轮询。

4. **能力逐层收敛**。`SUBAGENT_TOOL_DENY_ALWAYS`(gateway / agents_list / session_status / cron / sessions_send)对所有子代生效;`SUBAGENT_TOOL_DENY_LEAF`(subagents / sessions_list / sessions_history / sessions_spawn)对叶子额外生效。每往下一层,工具只减不增;同时保留"显式 allow 可覆盖默认 deny"的逃生口,守住"安全默认 + 显式覆盖"的底线。

贯穿全章的设计哲学,与前面几章一脉相承:**约束只能收紧不能放松**(沙箱继承、深度上限);**错误即文档**(`(8/8)` 配额比值、带修复指引的拒绝信息);**可观测的生命周期**(注册-运行-级联终止);**安全默认值加显式逃生口**(deny list 可被配置覆盖)。OpenClaw 没有试图"相信智能体会乖",而是把每一条纪律同时钉在工具描述、系统提示和运行时机制三个层面上——提示词负责引导,机制负责兜底。

下一章,我们将深入 OpenClaw 的**网关(gateway)与会话路由**——那个在本章里被反复提及、却一直站在幕后的角色:它是如何把"agent"调用、in-process 派发、admin scope 收敛(还记得 9.3 那个被拒绝的 `gateway` 工具吗?)串成一套统一的请求总线的。

## 动手实验

> 以下实验基于本章解读的源码逻辑设计,帮助你把"读懂"变成"验证"。建议在本地克隆 OpenClaw 仓库后进行。

**实验一:推导角色边界。** 不运行代码,纯靠 `resolveSubagentRoleForDepth` 的三行逻辑,手算当 `maxSpawnDepth = 3` 时,depth 为 0、1、2、3、4 各对应什么角色和 `controlScope`。然后找到 `subagent-capabilities.test.ts`,看官方测试是否和你的推导一致。重点验证:depth 恰好等于 `maxSpawnDepth` 时是 orchestrator 还是 leaf(注意 `<` 是严格小于)。

**实验二:复现五道闸门的错误信息。** 阅读 `spawnSubagentDirect`,把五道闸门(agentId 校验、深度、并发、目标策略、沙箱继承)各自的错误信息字符串抄写下来,对每一条回答:这条错误是给**人**看的还是给 **LLM** 看的?它有没有包含"修复指引"?然后对照 `subagent-spawn.depth-limits.test.ts`,看测试是如何断言这些错误的。

**实验三:画出权限收敛漏斗。** 基于 `SUBAGENT_TOOL_DENY_ALWAYS` 和 `SUBAGENT_TOOL_DENY_LEAF`,画一张表:横轴是 main / orchestrator / leaf,纵轴列出 `gateway`、`cron`、`sessions_send`、`sessions_spawn`、`subagents`、`sessions_list` 这几个工具,在每个格子里标注"可用 / 禁用"。再在 `resolveSubagentToolPolicy` 里找到"显式 allow 覆盖默认 deny"的那行 `filter`,思考:如果运维把 `sessions_spawn` 显式 allow 给了 leaf,会发生什么?这是否绕过了深度限制?(提示:结合 9.3 闸门二——工具可见 ≠ 闸门放行。)

**实验四:验证推送式完成的纪律。** 通读 `subagent-system-prompt.ts` 里 `canSpawn` 为 true 时追加的那段提示,数一数它一共用了几种不同的措辞来禁止"轮询"。再打开 `sessions-yield-tool.ts`,注意它的 `execute` 自己**并不执行**暂停,而是调用 `opts.onYield` 把控制权交还运行时。思考:为什么把"记录意图"和"真正暂停"拆成工具层和运行时层两件事?(提示:工具应该是无状态、可测试的,真正的回合控制属于运行时。)
