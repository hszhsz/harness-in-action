# 第 7 章 工具系统：从描述符到那道审批闸门

在第 4 章里，我们追踪了 Agent 的主循环：模型吐出一次 tool call，运行时执行它，把结果塞回上下文，再喂给模型。当时我们刻意跳过了两个问题——模型**看得到哪些工具**，以及在工具真正执行之前，有没有一道闸门可以拦下它。第 6 章结尾我还埋了一个钩子，承诺本章交代清楚：工具是怎么定义的、上百个工具如何被策略筛选与分组、以及第 4 章 4.5 节惊鸿一瞥的那道 `beforeToolCall` 审批闸门,在工具侧到底长什么样。

这一章我们就来还这笔债。和前几章一样,所有结论都建立在我实际读过的源码上——`src/tools/` 这个新的描述符内核、`src/agents/tool-catalog.ts` 这本核心工具目录、`src/agents/tool-policy*.ts` 这一组策略管线,以及那个长达 1383 行、把每次工具调用都包了一层的 `agent-tools.before-tool-call.ts`。

读完这一章,你会发现 OpenClaw 的工具系统其实是三层结构:**描述符内核**负责"一个工具长什么样、什么条件下可见";**策略管线**负责"在这次会话里,这上百个工具最终保留哪些";**执行包装层**负责"工具真要跑之前,谁有权说不"。三层各管一段,互不越界。

---

## 7.1 描述符内核:工具是一份声明,不是一段代码

很多 Agent 框架里,"工具"就是一个挂着 `name`、`description`、`execute` 的对象。OpenClaw 的新内核走得更远一步:它把工具的**定义**和工具的**执行**彻底拆开,中间隔着一层叫"描述符(descriptor)"的纯数据。

打开 `src/tools/types.ts`,核心类型 `ToolDescriptor` 是这样的:

```ts
export type ToolDescriptor = {
  readonly name: string;
  readonly title?: string;
  readonly description: string;
  readonly inputSchema: JsonObject;
  readonly outputSchema?: JsonObject;
  readonly owner: ToolOwnerRef;
  readonly executor?: ToolExecutorRef;
  readonly availability?: ToolAvailabilityExpression;
  readonly annotations?: JsonObject;
  readonly sortKey?: string;
};
```

注意几个细节。第一,这是一份**只读**(`readonly`)声明,它描述的是"这个工具是什么",而不是"它怎么跑"。第二,`owner` 和 `executor` 是两个独立字段——前者回答"谁定义了这个工具",后者回答"运行时该把调用派给谁"。第三,`executor` 是**可选的**;一个工具可以有描述符却没有执行器(后面会看到这背后的设计意图)。

`owner` 和 `executor` 都是封闭的可辨识联合(discriminated union),都覆盖同样四个"家族":

```ts
export type ToolOwnerRef =
  | { readonly kind: "core" }
  | { readonly kind: "plugin"; readonly pluginId: string }
  | { readonly kind: "channel"; readonly channelId: string; readonly pluginId?: string }
  | { readonly kind: "mcp"; readonly serverId: string };
```

这四个家族——核心(core)、插件(plugin)、通道(channel)、MCP 服务器——是整个 OpenClaw 工具生态的全部来源。一个工具要么是 OpenClaw 自带的核心工具,要么来自某个插件,要么是某个消息通道注册的动作,要么由一台 MCP 服务器提供。`types.ts` 顶部的注释把这个意图写得很直白:这套类型存在的目的,就是让"core、plugins、channels 和 MCP servers 共用同一份计划(share one plan)"。

把"定义"做成纯数据有什么好处?最直接的一条:无论工具来自哪个家族,planner 都能用同一套逻辑处理它们。core 工具和某台 MCP 服务器上的工具,在 planner 眼里没有本质区别,都是一份 `ToolDescriptor`。`src/tools/descriptors.ts` 里那两个 `defineToolDescriptor` / `defineToolDescriptors` 函数甚至什么都不做,只是把传入的对象原样返回:

```ts
export function defineToolDescriptor(descriptor: ToolDescriptor): ToolDescriptor {
  return descriptor;
}
```

它存在的唯一价值是"声明点的类型校验"——让作者在写描述符的地方就被 TypeScript 卡住,而不是等到运行时才发现字段写错。这是一种把契约前移到编译期的小技巧。

---

## 7.2 可用性表达式:用声明回答"这个工具现在能不能用"

`ToolDescriptor` 里最有意思的字段是 `availability`。它回答的问题是:这个工具在**当前运行环境**下到底该不该出现在模型面前?

一个浏览器工具,如果浏览器插件没启用,就不该让模型看见;一个需要某个 API key 的工具,如果环境变量缺失,展示给模型只会换来一次注定失败的调用。OpenClaw 没有把这套判断写成一堆 `if`,而是设计了一套**声明式的可用性表达式**。

最小单位是"信号(signal)",定义在 `types.ts`:

```ts
export type ToolAvailabilitySignal =
  | { readonly kind: "always" }
  | { readonly kind: "auth"; readonly providerId: string }
  | { readonly kind: "config"; readonly path: readonly string[];
      readonly check?: "exists" | "non-empty" | "available"; }
  | { readonly kind: "env"; readonly name: string }
  | { readonly kind: "plugin-enabled"; readonly pluginId: string }
  | { readonly kind: "context"; readonly key: string; readonly equals?: JsonPrimitive };
```

六种原子条件:永远可用、需要某个鉴权 provider、需要某个配置路径有值、需要某个环境变量、需要某个插件启用、需要某个上下文键匹配。信号之上再用布尔组合:

```ts
export type ToolAvailabilityExpression =
  | ToolAvailabilitySignal
  | { readonly allOf: readonly ToolAvailabilityExpression[] }
  | { readonly anyOf: readonly ToolAvailabilityExpression[] };
```

`allOf` 是"全部满足才可用",`anyOf` 是"任一满足即可用"。这种结构可以无限嵌套,足以表达"需要插件 X 启用,**并且**(环境变量 Y 存在 **或** 配置路径 Z 有值)"这类复合条件。

评估这套表达式的是 `src/tools/availability.ts`。它的入口 `evaluateToolAvailability` 把一个描述符跟运行时上下文做比对,返回一组"诊断(diagnostic)"——如果返回空数组,工具可用;如果非空,每一条诊断都解释了它**为什么**不可用。这里有几个值得记下的设计决策。

**默认永远可用。** 如果描述符没写 `availability`,代码会兜底成 `{ kind: "always" }`:

```ts
const availability = params.descriptor.availability ?? { kind: "always" };
```

也就是说,不声明任何条件 = 无条件可见。这是个友好的默认值,大多数核心工具不需要写可用性。

**`anyOf` 的诊断保留是反直觉但正确的。** 看 `evaluateExpression` 里 `anyOf` 这一支:

```ts
const diagnostics = expression.anyOf.map((entry) => evaluateExpression(entry, context));
// anyOf is available when at least one branch has no diagnostics; otherwise preserve all reasons.
return diagnostics.some((entries) => entries.length === 0) ? [] : diagnostics.flat();
```

只要有**任意一个**分支评估出空诊断,整个 `anyOf` 就算通过、返回空。但只有当**所有**分支都失败时,它才把每个分支的失败原因**全部**拼起来返回。这意味着当一个工具因为 `anyOf` 不满足而被隐藏时,诊断信息会告诉你"这几条路你一条都没走通",而不是只报第一条。对排障来说这非常关键——你能一眼看到所有候选条件。

**空组被当作错误。** 空的 `allOf` 和空的 `anyOf` 都会返回一条 `unsupported-signal` 诊断。这是在防御一类"写错了的描述符":一个空的 `anyOf` 在逻辑上意味着"没有任何路径能让它可用",与其让它静默地永远隐藏,不如明确报成契约错误。

`config` 信号还有个 `check` 子选项体现了同样的细致。`exists` 只看路径上有没有值;`non-empty` 会进一步判断字符串非空白、数组非空、对象有键;而 `available` 最特别——它把语义判断**委托**给上下文里的回调:

```ts
if ((signal.check ?? "exists") === "available") {
  // "available" delegates semantic checks, for example provider auth that is configured but stale.
  return params.context.isConfigValueAvailable?.({ value, path: signal.path, signal }) === true;
}
```

注释举的例子很传神:某个 provider 的鉴权"配置好了但已经过期"——配置路径上确实有值(`exists` 会通过),但它实际上不可用。`available` 这一档就是为这种"有值≠可用"的情况留的口子,把最终裁决权交给运行时。

---

## 7.3 planner:把上百份描述符,确定性地切成"可见"与"隐藏"

有了描述符和可用性评估,下一步就是把它们变成一份**计划(plan)**——告诉模型"这次你能用这些工具",同时把不可用的工具连同原因归档起来。这件事由 `src/tools/planner.ts` 的 `buildToolPlan` 完成,整个文件只有 67 行,但每一行都在为"确定性"服务。

```ts
export function buildToolPlan(options: BuildToolPlanOptions): ToolPlan {
  const descriptors = options.descriptors.toSorted(compareDescriptors);
  assertUniqueNames(descriptors);

  const visible: ToolPlanEntry[] = [];
  const hidden: HiddenToolPlanEntry[] = [];

  for (const descriptor of descriptors) {
    const diagnostics = [
      ...evaluateToolAvailability({ descriptor, context: options.availability }),
    ];
    if (diagnostics.length > 0) {
      hidden.push({ descriptor, diagnostics });
      continue;
    }
    if (!descriptor.executor) {
      throw new ToolPlanContractError({
        code: "missing-executor",
        toolName: descriptor.name,
        message: `Visible tool descriptor has no executor ref: ${descriptor.name}`,
      });
    }
    visible.push({ descriptor, executor: descriptor.executor });
  }

  return { visible, hidden };
}
```

第一步是**排序**。`toSorted` 用 `compareDescriptors`,优先按 `sortKey`(没有就用 `name`),再按 `name` 兜底:

```ts
function compareDescriptors(left, right) {
  return (
    (left.sortKey ?? left.name).localeCompare(right.sortKey ?? right.name) ||
    left.name.localeCompare(right.name)
  );
}
```

为什么要排序?因为工具呈现给模型的顺序需要稳定。同样一组输入,无论它们当初是以什么顺序注册进来的,planner 都给出同样的输出顺序。这就是"确定性"的第一层含义——它把"注册顺序"这个隐藏变量从结果里彻底消除了。

第二步是**唯一性校验**。`assertUniqueNames` 遍历一遍,只要发现重名就抛 `ToolPlanContractError`,错误码 `duplicate-tool-name`。

第三步是**逐个评估可用性**。有诊断的进 `hidden`,连同它为什么不可用的全部原因;没诊断的进入下一关。

第四步是那个意味深长的检查:**可见的工具必须有 executor**。还记得 7.1 里说 `executor` 是可选的吗?planner.ts 第 56 行的注释把这个设计的两面性点破了:

> Hidden tools may omit executors; visible tools must be callable after planning.

隐藏的工具可以没有执行器——反正它也不会被调用;但凡是要展示给模型的工具,**必须**是可调用的。一个"看得见却跑不了"的工具是一种契约违反,planner 选择直接抛错而不是放它进去。这道检查把"工具可见"和"工具可执行"在类型层面绑成了一体。

这里有个容易被忽略的区分:planner 在两种情况下抛错(`ToolPlanContractError`,定义于 `diagnostics.ts`,错误码只有 `duplicate-tool-name` 和 `missing-executor`),但在工具仅仅"不可用"时**不抛错**——它只是把工具归到 hidden。`diagnostics.ts` 的注释解释了这个边界:契约错误是"程序员的错误(programmer errors)",而可用性诊断是"有意隐藏的工具(intentionally hidden tools)",两者必须能被调用方区分开。一个是"你的工具注册写坏了",一个是"这个工具此刻本来就该藏起来",处理方式天差地别。

planner 的输出 `ToolPlan` 最后会经 `src/tools/protocol.ts` 转成发给模型的协议载荷。这一步极简,只取 `name` / `description` / `inputSchema` 三个字段:

```ts
export function toToolProtocolDescriptor(entry: ToolPlanEntry): ToolProtocolDescriptor {
  return {
    name: entry.descriptor.name,
    description: entry.descriptor.description,
    inputSchema: entry.descriptor.inputSchema,
  };
}
```

注释特意提醒:"Model/provider adapters still own schema normalization."——这里只负责挑出共享的那三个字段,各家模型/provider 的 schema 归一化仍然是它们自己的事。描述符内核管到协议边界为止,再往下就交给适配层。

---

## 7.4 核心工具目录:OpenClaw 自带哪些工具

描述符内核是通用机制,而 `src/agents/tool-catalog.ts` 是 OpenClaw 自己往这套机制里填的"内容"——它定义了 OpenClaw 出厂自带的全部核心工具。读这个文件,等于拿到了一份 Agent 能力的清单。

核心工具按"区段(section)"组织,文件里 `CORE_TOOL_SECTION_ORDER` 列出的顺序是:Files(文件)、Runtime(运行时)、Web(网络)、Memory(记忆)、Sessions(会话)、UI、Messaging(消息)、Automation(自动化)、Nodes(节点)、Agents(智能体)、Media(媒体)。每个工具挂在一个区段下,带着 id、标签、描述,以及它属于哪些"画像(profile)"。把 `CORE_TOOL_DEFINITIONS` 里的条目梳理一下,核心工具大致是这样一张图:

| 区段 | 代表性工具 |
| --- | --- |
| Files | `read`、`write`、`edit`、`apply_patch` |
| Runtime | `exec`、`process`、`code_execution` |
| Web | `web_search`、`web_fetch`、`x_search` |
| Memory | `memory_search`、`memory_get` |
| Sessions | `sessions_list`、`sessions_history`、`sessions_send`、`sessions_spawn`、`sessions_yield`、`subagents`、`session_status` |
| UI | `browser`、`canvas` |
| Messaging | `message` |
| Automation | `heartbeat_respond`、`cron`、`gateway` |
| Nodes | `nodes` |
| Agents | `agents_list`、`get_goal`、`create_goal`、`update_goal`、`update_plan`、`skill_workshop` |
| Media | `image`、`image_generate`、`music_generate`、`video_generate`、`tts` |

这张图本身就值得停留片刻。它告诉你 OpenClaw 心目中一个"完整的 Agent"应该有哪些手脚:读写文件、跑命令、上网、查记忆、开子会话、控浏览器、发消息、定时任务、管目标和计划、生成图音视频。第 4 章我们讲主循环时只用 `read` / `exec` 举例,现在你能看到那只是冰山一角。

### 画像:四套预设的能力组合

不是每个场景都需要全部工具。`tool-catalog.ts` 定义了四个"画像":

```ts
export type ToolProfileId = "minimal" | "coding" | "messaging" | "full";
```

每个工具定义里都有一个 `profiles` 数组,声明它属于哪些画像。画像策略由这两段代码生成:

```ts
const CORE_TOOL_PROFILES: Record<ToolProfileId, ToolProfilePolicy> = {
  minimal: { allow: listCoreToolIdsForProfile("minimal") },
  coding: { allow: [...listCoreToolIdsForProfile("coding"), "bundle-mcp"] },
  messaging: { allow: [...listCoreToolIdsForProfile("messaging"), "bundle-mcp"] },
  full: { allow: ["*"] },
};
```

四个画像的取向一目了然:`minimal` 几乎什么都不给(翻一遍定义,只有 `session_status` 标了 `minimal`);`coding` 是面向编码的全家桶,把文件、运行时、网络、记忆、会话、媒体一股脑收进来,还额外加上 `bundle-mcp`(让打包的 MCP 工具可用);`messaging` 偏重消息和会话;`full` 直接用通配符 `*` 放行一切。

值得注意的是 `full` 用 `["*"]` 而不是去枚举所有工具 id。这是个聪明的偷懒——通配符意味着"以后新增的核心工具自动包含在内",不用每加一个工具就回来改画像。

### 分组:`group:` 前缀与 `group:openclaw`

画像之外,还有"分组"。`buildCoreToolGroupMap` 把工具按区段自动归成 `group:fs`、`group:runtime`、`group:web` 这类组,同时拼出一个特殊的 `group:openclaw`:

```ts
const openclawTools = CORE_TOOL_DEFINITIONS
  .filter((tool) => tool.includeInOpenClawGroup)
  .map((tool) => tool.id);
return {
  "group:openclaw": openclawTools,
  ...Object.fromEntries(sectionToolMap.entries()),
};
```

每个工具定义里那个 `includeInOpenClawGroup` 布尔决定它是否进入 `group:openclaw`。这些组的意义在下一节会显现:用户在配置里写 `group:web`,策略管线会自动把它展开成该组下的所有具体工具 id。分组是用户书写策略时的"语法糖",省得一个个列工具名。

---

## 7.5 策略管线:上百个工具,如何层层筛到最终那几个

planner 决定了"在当前环境下哪些工具技术上可用",但这还不够。一个企业可能想规定"这个 Agent 只准用文件和网络工具",一个群聊可能想禁掉所有能发外部消息的工具,一个子 Agent 不该有权限再开新会话。这些是**策略**层面的诉求,由 `src/agents/tool-policy*.ts` 这一组文件处理。

### 匹配规则:deny 永远赢,空 allow 等于全放行

最底层的匹配器在 `tool-policy-match.ts`,逻辑简单到可以背下来:

```ts
return (name: string) => {
  const normalized = normalizeToolName(name);
  if (matchesAnyGlobPattern(normalized, deny)) {
    return false;          // 命中 deny,直接拒
  }
  if (allow.length === 0) {
    return true;           // 没有 allow 列表 = 默认全放行
  }
  if (matchesAnyGlobPattern(normalized, allow)) {
    return true;           // 命中 allow,放行
  }
  if (normalized === "apply_patch" && matchesAnyGlobPattern("write", allow)) {
    return true;           // 特例:宽泛的 write 也覆盖 apply_patch
  }
  return false;            // 有 allow 列表但没命中 = 拒
};
```

三条规则的优先级一定要记牢:**deny 永远先判,命中即拒,没有任何 allow 能救回来**;只有不在 deny 里的工具,才轮到 allow 来裁决;而空的 allow 列表被特意定义成"全放行"(文件头注释:"an empty allow list means 'allow everything not denied'")。最后那个 `apply_patch` 特例很贴心:`apply_patch` 是具体的写文件工具,如果用户在 allow 里只写了宽泛的 `write`,匹配器会让它也覆盖到 `apply_patch`,免得用户被工具命名的细节绊倒。

匹配用的是 glob 模式(`compileGlobPatterns` / `matchesAnyGlobPattern`),所以 allow/deny 里可以写通配符。而无论 allow 还是 deny,都会先经过 `expandToolGroups` 展开分组、再经 `normalizeToolName` 归一化——这就是上一节那些 `group:` 前缀和工具别名生效的地方。

说到别名,`tool-policy-shared.ts` 里有一张小表:

```ts
const TOOL_NAME_ALIASES: Record<string, string> = {
  bash: "exec",
  "apply-patch": "apply_patch",
};
```

用户在配置里写 `bash`,系统知道你指的是 `exec`;写 `apply-patch`(连字符),归一化成 `apply_patch`(下划线)。这些细节都在 `normalizeToolName` 一处统一处理,保证 allow 和 deny 两条路径用的是同一套归一化逻辑——文件头注释把这个意图写得很清楚:"Keeps aliases, groups, profile expansion, and prefix matching consistent across allow/deny paths."

### 分层管线:八道策略层叠加

单个匹配器只是积木。真正的复杂度在于,一次会话的工具集要经过**多层策略**层层过滤。`tool-policy-pipeline.ts` 的 `buildDefaultToolPolicyPipelineSteps` 列出了默认的八道层,按运行时解析顺序是:

1. `tools.profile` —— 画像策略
2. `tools.byProvider.profile` —— 按 provider 区分的画像
3. `tools.allow` —— 全局策略
4. `tools.byProvider.allow` —— 按 provider 区分的全局策略
5. `agents.<id>.tools.allow` —— 单个 Agent 的策略
6. `agents.<id>.tools.byProvider.allow` —— 该 Agent 按 provider 区分的策略
7. `group tools.allow` —— 群组策略
8. `tools.toolsBySender` —— 按发送者区分的策略

`applyToolPolicyPipeline` 把工具列表依次穿过这八道层。每一层不为空就过滤一次,然后产生一条审计记录(`auditToolPolicyFilter`):

```ts
let filtered = params.tools;
for (const step of params.steps) {
  if (!step.policy) continue;
  // ...(分析 + 展开插件分组)...
  const before = filtered;
  filtered = filterToolsByPolicy(before, expanded);
  auditToolPolicyFilter({ stepLabel: step.label, policy: expanded, before, after: filtered, ... });
}
return filtered;
```

这是个典型的**漏斗**结构。工具集从全集开始,每过一层只会变窄不会变宽(每层都是 allow/deny 过滤)。层的顺序代表了优先级语义:画像是底盘,然后是全局,再是单 Agent,再是群组,最后是按发送者——越靠后越具体,越能针对特定场景收紧。

### 诊断:配错了 allow,系统会提醒你

策略管线还有一段相当用心的"配置容错"逻辑。如果用户在 allow 里写了一个**既不是核心工具、也不是任何已知插件工具**的名字,系统不会静默忽略,而是发一条警告。`analyzeAllowlistByToolType` 负责把 allow 里的条目分类成"核心工具""插件工具""未知",`tool-policy-pipeline.ts` 再根据分类拼出有针对性的提示。

提示文案分了好几种情况(见 `describeUnknownAllowlistSuffix`)。如果 allow 里全是插件条目、一个核心工具都没有,会提醒"你的 allowlist 只含插件条目,核心工具将不可用";如果某个条目是 OpenClaw 出厂的核心工具但当前运行时/provider/模型/配置下不可用,会单独说明;两种情况混在一起还有混合文案。这些警告通过一个有上限(256 条)的去重缓存(`rememberToolPolicyWarning`)发出,保证同一条警告不会刷屏。

这个设计的价值在于:工具策略是用户手写的配置,极易拼错工具名或漏掉插件启用。与其让用户对着"为什么这个工具用不了"抓瞎,不如在加载策略时就主动把可疑条目指出来。这是把"可观测性"延伸到了配置层。

### 子 Agent 的硬性禁令

`agent-tools.policy.ts` 里有一组特别值得一提的策略:针对子 Agent 的**深度感知禁令**。有两张写死的 deny 表:

```ts
const SUBAGENT_TOOL_DENY_ALWAYS = [
  "gateway", "agents_list",        // 系统级 / 管理类,子 Agent 绝不该碰
  "session_status", "cron",        // 状态/调度,由主 Agent 协调
  "sessions_send",                 // 直接发会话,子 Agent 应走 announce 链
];
const SUBAGENT_TOOL_DENY_LEAF = [
  "subagents", "sessions_list", "sessions_history", "sessions_spawn",
];
```

`SUBAGENT_TOOL_DENY_ALWAYS` 对所有子 Agent 一律封禁——这些是系统管理、状态调度、直接发消息这类"只有主 Agent 才该协调"的能力。`SUBAGENT_TOOL_DENY_LEAF` 则只对"叶子(leaf)"子 Agent 额外封禁,封的是再开子会话、再生成子 Agent 的能力。`resolveSubagentDenyList` 根据深度决定用哪套:

```ts
function resolveSubagentDenyList(depth: number, maxSpawnDepth: number): string[] {
  const isLeaf = depth >= Math.max(1, Math.floor(maxSpawnDepth));
  if (isLeaf) {
    return [...SUBAGENT_TOOL_DENY_ALWAYS, ...SUBAGENT_TOOL_DENY_LEAF];
  }
  return [...SUBAGENT_TOOL_DENY_ALWAYS];
}
```

逻辑很清楚:深度还没到上限的"协调型"子 Agent,允许它管理自己的孩子(可以用 `sessions_spawn` 等);一旦到了最大深度成了叶子,就连开新会话的能力也一并收走。这是在防止子 Agent 无限递归生成,把生成树的深度焊死在配置允许的范围内。这一层禁令是用户配置无法轻易绕过的——它是系统给递归 Agent 设的安全护栏。

---

## 7.6 还债:那道 `beforeToolCall` 审批闸门

前面五节铺垫的都是"静态"决策:哪些工具存在、哪些可见、哪些被策略保留。但第 4 章 4.5 节我们瞥见的那道闸门是**动态**的——它在每一次具体的工具调用即将执行的那一刻介入,有权改写参数、要求审批、甚至直接否决。现在我们打开 `src/agents/agent-tools.before-tool-call.ts`,看它到底怎么运作。

### 它是一层包装,不是一个分支

最关键的认知:`beforeToolCall` 不是主循环里的一个 `if`,而是把每个工具的 `execute` 方法整个包了一层。核心函数是 `wrapToolWithBeforeToolCallHook`:

```ts
export function wrapToolWithBeforeToolCallHook(tool, ctx?, options = {}): AnyAgentTool {
  const execute = tool.execute;
  if (!execute) return tool;
  // ...
  const wrappedTool: AnyAgentTool = {
    ...tool,
    execute: async (toolCallId, params, signal, onUpdate) => {
      // 1. 跑 before_tool_call 钩子链
      const outcome = await runBeforeToolCallHook({ toolName, params: hookParams, ... });
      // 2. 如果被拦,要么抛错(failure),要么返回 blocked 结果(veto)
      if (outcome.blocked) { /* ... */ }
      // 3. 否则用(可能被改写的)参数跑真正的 execute
      const result = await execute(toolCallId, executeParams, signal, onUpdate);
      // 4. 发诊断事件 + 记录循环检测
      return result;
    },
  };
  // 打上标记,避免重复包装
  Object.defineProperty(wrappedTool, BEFORE_TOOL_CALL_WRAPPED, { value: true, enumerable: true });
  return wrappedTool;
}
```

包装模式的好处是**透明**。主循环只管调 `tool.execute`,它根本不知道(也不需要知道)这个 `execute` 已经被换成了带闸门的版本。闸门的全部逻辑——钩子、审批、循环检测、诊断——都藏在这层包装里,对调用方完全无感。代码里还用 `BEFORE_TOOL_CALL_WRAPPED` 这个 Symbol 标记打了戳,`isToolWrappedWithBeforeToolCallHook` 可以检查一个工具是否已被包装,避免重复套娃。

### 闸门里发生了什么:`runBeforeToolCallHook` 的四道关卡

真正的决策逻辑在 `runBeforeToolCallHook`(将近 300 行)。它依次过四类检查:

**第一关:循环检测。** 如果有 sessionKey,先调 `detectToolCallLoop`。如果检测到"卡死(stuck)"且级别是 `critical`,直接否决:

```ts
if (loopResult.stuck) {
  if (loopResult.level === "critical") {
    return { blocked: true, kind: "veto", deniedReason: "tool-loop", reason: loopResult.message, params };
  }
  // warning 级别只记日志、发警告,不拦截
}
```

这是在防 Agent 陷入"同一个工具用同样参数反复调"的死循环。critical 级别直接拦,warning 级别只提醒。这正是第 4 章我们讨论"约束与恢复"时所说的护栏之一,落在了工具调用这一层。

**第二关:可信工具策略(trusted tool policies)。** 这些是插件注册的、被宿主信任的策略(`runTrustedToolPolicies`)。它们可以 `block`(否决)、可以 `requireApproval`(要求审批)、也可以 `params`(改写参数):

```ts
if (trustedPolicyResult?.block) {
  return { blocked: true, kind: "veto", deniedReason: "plugin-before-tool-call",
           reason: trustedPolicyResult.blockReason || "...", params };
}
```

**第三关:审批。** 无论是可信策略还是后面的插件钩子,只要返回了 `requireApproval`,就走 `resolveBeforeToolCallApprovalOutcome`。这里有三种模式(`approvalMode`):

```ts
if (params.approvalMode === "defer") { /* 把审批推迟到后续执行边界 */ }
if (params.approvalMode === "report") { /* 直接当成拒绝上报 */ }
// 默认 request:真的去请求审批
return await requestPluginToolApproval({ approval, toolName, ... });
```

`request` 模式下,`requestPluginToolApproval` 会通过 gateway 发起一次真正的审批请求,然后**等待用户决策**。等待的同时它监听 abort 信号,这样如果整个 Agent run 被取消,用户不会被晾在那里干等到审批超时:

```ts
const abortPromise = new Promise<never>((_, reject) => {
  if (params.signal!.aborted) { reject(...); return; }
  onAbort = () => reject(...);
  params.signal!.addEventListener("abort", onAbort, { once: true });
});
waitResult = await Promise.race([waitPromise, abortPromise]);
```

决策回来后,`allow_once` / `allow_always` 放行,`deny` 否决,超时则看 `timeoutBehavior`(默认 `deny`,也可配成 `allow`)。这里还有个值得称道的失败处理:如果 gateway 请求本身出错了,代码选择**拦截而非放行**——"plugin approval gateway request failed; blocking tool call"。审批系统坏了的时候,默认是"宁可拦错,不可放错",这是安全系统应有的失败方向(fail closed)。

**第四关:before_tool_call 插件钩子。** 最后跑全局钩子运行器的 `runBeforeToolCall`。和可信策略对称,它同样可以 block、requireApproval、改写 params:

```ts
const hookResult = await hookRunner.runBeforeToolCall({ toolName, params: hookEventParams, ... }, ctx);
if (hookResult?.block) {
  return { blocked: true, kind: "veto", deniedReason: "plugin-before-tool-call", ... };
}
if (hookResult?.params) {
  finalParams = mergeParamsWithApprovalOverrides(finalParams, hookResult.params);
}
```

注意 `hookResult.params` 这个能力——钩子不仅能放行或拦截,还能**改写**即将传给工具的参数。比如一个安全插件可以在 `exec` 真正执行前,往命令里注入额外的沙箱约束。改写后的参数会被 `recordAdjustedParamsForToolCall` 记下来,供后续适配层执行时取用。

### veto 与 failure:两种"拦"的不同归宿

闸门拦截的结果分两种 `kind`,在包装层的处理截然不同:

```ts
if (outcome.blocked) {
  if (outcome.kind !== "veto") {
    throw new Error(outcome.reason);   // failure:抛异常
  }
  // veto:返回一个结构化的 blocked 结果,而不是抛错
  const blockedResult = buildBlockedToolResult({ reason: outcome.reason, deniedReason: ... });
  return blockedResult;
}
```

`failure`(比如审批 gateway 挂了)会**抛异常**,因为这是个意外故障,应该沿调用栈往上冒。而 `veto`(比如循环检测拦截、用户明确拒绝)会返回一个**结构化的 blocked 结果**:

```ts
export function buildBlockedToolResult(params) {
  return {
    content: [{ type: "text", text: params.reason }],
    details: { status: "blocked", deniedReason: ..., reason: params.reason },
  };
}
```

这个区别很重要。veto 是"系统有意识地说不",它需要把"为什么不"作为一条正常的工具结果喂回给模型——模型看到 `status: "blocked"` 和原因文本,就能理解"这条路被拦了,我换个思路"。如果用抛异常处理 veto,模型就只会看到一次失败而不知道为什么,可能傻乎乎地重试。把"有意的拒绝"做成结构化结果而非异常,是让 Agent 能从约束中**优雅恢复**的关键——这又一次呼应了第 4 章"约束与恢复"那一支柱。

### 顺带做的事:诊断与遥测

闸门除了"拦/放/改",还在工具执行的前后发一系列诊断事件:`tool.execution.started`、`tool.execution.completed`(带 `durationMs`)、`tool.execution.error`(带错误分类和 HTTP 状态码)、被拦时的 `tool.execution.blocked`。它甚至能识别"这次工具调用是不是在使用某个 Skill"(`findSkillUsageMatch`),命中就发一条 `skill.used` 遥测。这些事件就是第 5 章 lifecycle 那一层、第 6 章 ContextEngine 各处都在消费的可观测性信号的源头之一——工具调用这个最频繁的动作,在这里被密密地埋了点。

---

## 7.7 三层合一:一次工具调用的完整旅程

把这一章的三层串起来,一个工具从"被定义"到"被执行"的完整旅程是这样的:

1. **定义阶段(描述符内核)。** 某个家族(core/plugin/channel/mcp)用一份 `ToolDescriptor` 声明工具,包括它的 schema、owner、executor,以及可选的 `availability` 表达式。
2. **规划阶段(planner)。** `buildToolPlan` 把所有描述符排序、查重,逐个评估可用性,切成 `visible` 和 `hidden`。可见的工具必须有 executor,否则抛契约错误。可见集再经 `protocol.ts` 转成发给模型的 schema。
3. **策略阶段(策略管线)。** 在工具进入这次具体会话之前,`applyToolPolicyPipeline` 用八道层(画像→全局→Agent→群组→发送者)层层过滤,deny 永远优先,空 allow 等于全放行;子 Agent 还要额外吃深度感知的硬禁令。
4. **执行阶段(beforeToolCall 包装)。** 模型选了一个工具发起调用,被包装过的 `execute` 先跑闸门:循环检测→可信策略→审批→插件钩子。期间可能改写参数、要求审批,或者以 veto/failure 两种方式拦下。放行了才跑真正的 `execute`,前后再撒一圈诊断事件。

这四个阶段对应着本章开头说的三层职责,而它们又精确落在了第 4 章那六根支柱里:描述符内核与 planner 是"工具系统"支柱的骨架;策略管线和 beforeToolCall 闸门则横跨"约束与恢复"和"评估与可观测性"两根支柱。第 4 章我们说 Agent = 模型 + Harness,而 Harness 的六根支柱里,"工具"这一根,到这一章才算被我们彻底拆开看清。

下一章,我们会把视线从"工具怎么被管理"转向"工具怎么被执行"——尤其是 `exec` 这个最危险也最强大的工具背后的沙箱机制。本章反复出现的"沙箱工具策略"`SandboxToolPolicy`,到那时会从一个类型名变成一套真实的隔离边界。

---

## 本章小结

- OpenClaw 的工具系统是**三层结构**:描述符内核(定义+可见性)、策略管线(会话级筛选)、执行包装层(调用级闸门),各管一段、互不越界。
- **描述符内核**(`src/tools/`)把工具的"定义"做成只读纯数据 `ToolDescriptor`,`owner` 和 `executor` 分离,统一支持 core/plugin/channel/mcp 四个家族,让所有来源的工具共用同一套规划逻辑。
- **可用性表达式**用六种原子信号(always/auth/config/env/plugin-enabled/context)加 `allOf`/`anyOf` 组合,声明式地回答"工具现在能不能用";`anyOf` 全失败时保留所有原因,`config` 的 `available` 档把"有值≠可用"的语义判断委托给运行时。
- **planner** 的 `buildToolPlan` 追求确定性:排序消除注册顺序影响、查重防冲突、按可用性切成 visible/hidden;可见工具必须有 executor,否则抛 `ToolPlanContractError`。契约错误(程序员的错)与可用性诊断(有意隐藏)被刻意区分。
- **核心工具目录**(`tool-catalog.ts`)列出 OpenClaw 自带的全部核心工具,按 11 个区段组织,并用 `minimal`/`coding`/`messaging`/`full` 四个画像和 `group:` 分组,作为用户书写策略的预设与语法糖。
- **策略管线** 的匹配规则是"deny 永远先判、命中即拒、空 allow 等于全放行";默认八道层(画像→全局→Agent→群组→发送者)层层收窄,配错 allow 会触发去重警告;子 Agent 受深度感知的硬性禁令约束,防止无限递归生成。
- **beforeToolCall 闸门** 通过包装每个工具的 `execute` 实现,依次过循环检测、可信策略、审批、插件钩子四关;能改写参数、要求审批或拦截。拦截分 veto(返回结构化 blocked 结果,让模型优雅恢复)和 failure(抛异常)两种,审批 gateway 故障时默认 fail closed。

## 动手实验

1. **画出可用性诊断树。** 阅读 `src/tools/availability.ts` 的 `evaluateExpression`,手写一个嵌套表达式 `{ allOf: [{ kind: "plugin-enabled", pluginId: "browser" }, { anyOf: [{ kind: "env", name: "API_KEY" }, { kind: "config", path: ["browser", "token"] }] }] }`,在纸上推演:当 browser 插件启用、但 API_KEY 缺失且 config 也没有 token 时,最终返回的诊断数组里会有几条、分别是什么 reason?验证你对 `anyOf` "全失败才保留全部原因"的理解。

2. **追踪一次策略漏斗。** 在 `tool-policy-pipeline.ts` 的 `applyToolPolicyPipeline` 里给 `filtered = filterToolsByPolicy(...)` 之后加一行 `console.log(step.label, before.length, "->", filtered.length)`(仅本地实验),构造一个同时配置了 profile、全局 allow 和群组 deny 的场景,观察工具数量如何一层层收窄。注意 deny 优先和空 allow 全放行这两条规则分别在哪一层生效。

3. **复现 veto 与 failure 的分叉。** 阅读 `wrapToolWithBeforeToolCallHook` 中 `outcome.blocked` 的处理分支,对照 `runBeforeToolCallHook` 里所有 `return { blocked: true, ... }` 的地方,列一张表:哪些拦截是 `kind: "veto"`、哪些是 `kind: "failure"`?思考为什么"用户明确拒绝(deny)"被归为 failure 而"循环检测 critical"被归为 veto——它们喂回给模型的形态有何不同,对模型的后续行为各有什么影响?

4. **数一数子 Agent 的能力剥夺。** 对照 `tool-catalog.ts` 的核心工具全集与 `agent-tools.policy.ts` 的 `SUBAGENT_TOOL_DENY_ALWAYS` / `SUBAGENT_TOOL_DENY_LEAF`,列出:一个深度为 1 的"协调型"子 Agent 比主 Agent 少了哪些工具?一个叶子子 Agent 又比协调型子 Agent 进一步少了哪些?从这两张禁令表反推 OpenClaw 对"子 Agent 应该做什么、不应该做什么"的设计假设。
