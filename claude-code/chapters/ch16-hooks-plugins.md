# 第 16 章　用户自定义逻辑：Hooks 事件系统与 Plugin 分发单元

> 第 15 章把外部能力**引进来**——Skill 注入领域知识、MCP 接入工具服务器。但「引进来的能力」终究是别人写好的;用户还想在 harness 的**关键节点插入自己的逻辑**:提交前跑 lint、工具调用前做校验、会话结束时归档。这就是本章的两个主题:**Hooks**(在固定事件点回调用户脚本,可阻断、可改写、可注入)与 **Plugin**(把 skill/agent/hook/MCP 配置打包成一个可分发、可信任的单元)。两者共同的工程命题和前几章一脉相承:**让用户自定义逻辑在不破坏 harness 不变量、不绕过权限、不引发死循环的前提下被安全编织进运行时**。这一章拆 27 种 hook 事件的分类与时机、4 种 hook 类型的 schema、hook 的超时/阻断协议(退出码 2)、hook 输出如何反馈进主循环(以及第 12 章警惕的「error → hook blocking → retry」死亡螺旋如何被堵住)、plugin 的 manifest 与打包内容、marketplace 的同形字防伪、以及 `strictPluginOnlyCustomization` 那道把四个定制面锁死到 plugin-only 的策略闸。

## 16.1 27 种 hook 事件:把运行时生命周期钉成枚举

Hook 系统的第一块基石是一张**封闭的事件枚举**。`HOOK_EVENTS`(`coreSchemas.ts` 第 362 行,在 `coreTypes.ts` 第 25 行、`agentSdkTypes.js` 第 4 行三处镜像同步)列了整整 **27 个**事件:

```ts
export const HOOK_EVENTS = [
  'PreToolUse', 'PostToolUse', 'PostToolUseFailure',
  'Notification', 'UserPromptSubmit',
  'SessionStart', 'SessionEnd', 'Stop', 'StopFailure',
  'SubagentStart', 'SubagentStop', 'PreCompact', 'PostCompact',
  'PermissionRequest', 'PermissionDenied', 'Setup',
  'TeammateIdle', 'TaskCreated', 'TaskCompleted',
  'Elicitation', 'ElicitationResult', 'ConfigChange',
  'WorktreeCreate', 'WorktreeRemove',
  'InstructionsLoaded', 'CwdChanged', 'FileChanged',
] as const
```

这张表本身就是一份**运行时生命周期地图**:它把整本书前面拆过的每个关键节点都暴露成一个可挂载点——工具调用前后(`PreToolUse`/`PostToolUse`,呼应第 6–8 章的工具系统与权限)、压缩前后(`PreCompact`/`PostCompact`,呼应第 12 章)、子 agent 起止(`SubagentStart`/`SubagentStop`,呼应第 14 章)、权限请求与拒绝(`PermissionRequest`/`PermissionDenied`,呼应第 8 章)、会话起止(`SessionStart`/`SessionEnd`)。事件被钉成 `as const` 枚举的好处和第 13 章「记忆类型封闭四类」同源:**枚举之外的事件名根本无法通过 schema 校验**,用户写错事件名会在配置加载时就被 Zod 拦下,而不是静默不触发。

事件命名上有个一致的「成对」规律:很多事件是**前后成对**或**正常/失败成对**的——`PreToolUse`/`PostToolUse` 夹住工具执行、`Stop`/`StopFailure` 区分正常停止与异常停止、`PostToolUse`/`PostToolUseFailure` 区分工具成功与失败后的回调。这让用户能精确地只在「失败路径」上挂逻辑(比如工具失败时发告警),而不必在成功路径上空跑。

`hookEvents.ts`(第 18 行)还圈出一个子集 `ALWAYS_EMITTED_HOOK_EVENTS = ['SessionStart', 'Setup']`——这两个事件**无条件触发**,哪怕没有任何匹配的 hook 也要走一遍事件发射流程。配套的 `MAX_PENDING_EVENTS = 100`(第 20 行)给挂起事件队列封了顶,防止事件积压把内存撑爆——又是一处「任何队列都要有上限」的工程偏执。

## 16.2 四种 hook 类型:从 shell 到 LLM 到 HTTP 到 agent

事件回答「**什么时候**触发」,hook 类型回答「**触发了做什么**」。`buildHookSchemas`(`schemas/hooks.ts` 第 31 行)定义了四种 hook,用 Zod discriminated union 按 `type` 字段分流:

- **`command`(`BashCommandHookSchema`,第 32 行)**:最常见的——跑一条 shell 命令。字段有 `command`、`shell`(bash/powershell)、`timeout`、`statusMessage`、`once`、`async`、`asyncRewake`。
- **`prompt`(`PromptHookSchema`,第 67 行)**:用一段 prompt 调小模型评估,`$ARGUMENTS` 占位符会被替换成 hook 输入的 JSON;不指定 `model` 就用「默认的 small fast model」。
- **`http`(`HttpHookSchema`,第 97 行)**:把 hook 输入 JSON POST 到一个 URL;支持 `headers` 和 `allowedEnvVars`(显式列出哪些环境变量可被插值进 header,**没列的一律解析成空串**——又是 allowlist)。
- **`agent`(`AgentHookSchema`,第 126 行)**:一个「agentic 验证器」,用一段 prompt 描述「要验证什么」(例:「Verify that unit tests ran and passed.」),不指定 `model` 就用 Haiku、`timeout` 默认 60 秒。

四种类型共享一个关键的过滤字段 `if`(`IfConditionSchema`,第 19 行):它用**权限规则语法**(如 `Bash(git *)`、`Read(*.ts)`)在**spawn 之前**就过滤掉不匹配的调用——注释说得很直白:「Avoids spawning hooks for non-matching commands」。这是性能上的精算:与其每次工具调用都起一个 hook 进程再让它自己判断「这事跟我有没有关系」,不如在起进程前就用一个轻量的模式匹配筛掉——和第 15 章 skill 的 `paths` glob、第 13 章条件规则是同一种「按匹配条件懒触发」的思路。

`AgentHookSchema` 上有一段血泪注释(第 128–135 行)值得记:**绝不能在这个 schema 上加 `.transform()`**。因为这个 schema 会被 `parseSettingsFile` 用,而 `updateSettingsForSource` 会把解析结果 round-trip 过 `JSON.stringify`——一个被 transform 成函数的值在序列化时会被**静默丢弃**,结果就是用户写在 `settings.json` 里的 prompt 凭空消失(gh-24920、CC-79)。这是「schema 不只是校验、还要能安全往返序列化」的一处真实教训。

最后,`HooksSchema`(`schemas/hooks.ts`)把事件和 hook 数组绑起来:

```ts
export const HooksSchema = lazySchema(() =>
  z.partialRecord(z.enum(HOOK_EVENTS), z.array(HookMatcherSchema())))
export type HooksSettings = Partial<Record<HookEvent, HookMatcher[]>>
```

每个事件名映射到一组 `HookMatcher`,每个 matcher 带一个 `matcher` 字符串(匹配工具名等)和一组 `hooks`。这就是用户在 `settings.json` 里写 hook 的最终结构。

## 16.3 两套超时:工具 hook 给 10 分钟,SessionEnd 给 1.5 秒

Hook 是用户代码,可能慢、可能挂死,所以每条执行都有超时。`hooks.ts` 里有两个量级悬殊的超时常量,差异本身就是设计:

```ts
const TOOL_HOOK_EXECUTION_TIMEOUT_MS = 10 * 60 * 1000   // 600_000,十分钟
const SESSION_END_HOOK_TIMEOUT_MS_DEFAULT = 1500        // 一点五秒
```

工具相关的 hook 给到 **10 分钟**——因为它可能要跑完整的测试套件、lint、构建,这些天然耗时。但 `SessionEnd` hook 只给 **1.5 秒**(第 176 行),注释解释得很清楚:「SessionEnd hooks run during shutdown/clear and need a much tighter bound」——会话结束 hook 跑在关闭/清理流程里,用户已经在等着退出了,不能让一个归档脚本把退出卡上十分钟。这个值还兼任「整体 AbortSignal 上限」(hook 并行跑,一个值同时管单 hook 超时和总超时),并可通过 `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` 给 teardown 脚本放宽。**同一个机制(超时),按它所处的生命周期阶段给两个数量级不同的预算**——这和第 12 章「不同窗口档位给不同 autocompact buffer」是同构的按场景分档。

## 16.4 退出码协议:0 成功、2 阻断、其它只是噪音

Hook 怎么把结果告诉 harness?最基础的通道是 **shell 退出码**,CCB 把它钉成一套三态协议(`hooks.ts` 第 2785 行附近):

- **退出码 0**:成功。stdout 作为 `hook_success` attachment 注入(第 2765 行),内容能被模型看到。
- **退出码 2**:**阻断反馈**。产出一个 `blockingError`(`outcome: 'blocking'`,第 2796–2804 行),把 stderr 作为阻断原因——这是 hook 说「不行,别继续」的方式。
- **其它任何非零退出码**:「non-critical error that should just be shown to the user」(第 2807 行)——只展示给用户看,不阻断、不喂给模型当指令。

退出码 2 这个「阻断魔数」是整个 hook 系统的承重点:`PreToolUse` hook 退出码 2 就能拦下一次工具调用、`Stop` hook 退出码 2 就能阻止 agent 停下来继续干活。把「阻断」编码成一个特定退出码(而非任意非零),好处是和「脚本本身崩了」(其它非零码)区分开——脚本意外崩溃不应该被误解成「用户想阻断」。

更复杂的反馈走 **JSON stdout 协议**,由 `processHookJSONOutput`(第 569 行)解析:

- `continue: false` → `preventContinuation`,可带 `stopReason`(第 601 行)。
- `decision: 'approve'` → 权限 `allow`;`decision: 'block'` → 权限 `deny` 并产出 `blockingError`(第 608–619 行);**未知 decision 直接 `throw`**(第 622 行)——又是「未知输入不静默吞、报错暴露」。
- `PreToolUse` 专属的 `hookSpecificOutput.permissionDecision`(第 634 行):`allow`/`deny`/`ask` 三态,直接接管第 8 章的权限判定——hook 可以**代替用户**说「这个工具调用允许/拒绝/还是要问」。同样,未知 permissionDecision 也 `throw`(第 654 行)。

JSON 协议比退出码表达力强得多:退出码只能说「成/阻/错」,JSON 能改写权限决策、能给出结构化原因、能注入 systemMessage。但两者的语义底座是一致的——**hook 是 harness 主动让出的一个决策位,用户代码可以在这个位上同意、否决、或要求人工介入**。

## 16.5 asyncRewake:后台 hook 也能把模型「叫醒」

普通 hook 是同步阻塞的——harness 等它跑完拿到退出码再继续。但有些 hook 天然慢(跑一套集成测试),又不该卡住主流程,于是有了 `async` 和更进一步的 `asyncRewake`(`BashCommandHookSchema` 第 55–64 行)。

`asyncRewake` 的语义很精妙(`hooks.ts` 第 206–246 行):hook 在后台跑,**完全绕过 async 注册表**,跑完时如果退出码是 2(阻断错误),就把结果作为 **task-notification** 入队——通过 `enqueuePendingNotification`(第 238 行)在模型空闲时经 `useQueueProcessor` 唤醒它、或在模型忙时经 `queued_command` attachment 中途注入。换句话说:**一个后台 hook 跑完发现有问题,能反过来把已经在干别的事的模型「叫醒」并告诉它「停一下,这里出问题了」。**

实现上有一处和第 14 章「后台 agent 不挂主线程 abort」同源的克制(第 213–218 行注释):它**故意不调** `shellCommand.background()`,因为那会触发 `spillToDisk()` 破坏内存里的 stdout/stderr 捕获;abort handler 对 `'interrupt'`(用户提交新消息)no-op、对硬取消(Escape)才真杀。和后台 agent 一样——**后台 hook 要能扛住新的用户输入,但用户明确按 Escape 时必须能被杀掉**。

## 16.6 死亡螺旋的三道堵口:hook 不能在错误路径上无限重试

第 12 章 12.8 反复警告过一个最阴险的失败模式:**error → hook blocking → retry → error → …**,每一轮还往上下文里灌更多 token。Hook 系统正是这个螺旋的潜在源头——`Stop` hook 阻断会让 agent 重试,如果重试又因为 prompt-too-long / API 错误失败,而失败又触发 Stop hook……就转不出来了。`query.ts` 用三处守卫把这个螺旋堵死:

**堵口一:prompt-too-long 时不 fall through 到 stop hooks**(第 1441–1448 行)。当模型因为上下文超长根本没产出有效响应时,代码直接 `return`,注释写得斩钉截铁:

```ts
// No recovery — surface the withheld error and exit. Do NOT fall
// through to stop hooks: the model never produced a valid response,
// so hooks have nothing meaningful to evaluate. Running stop hooks
// on prompt-too-long creates a death spiral: error → hook blocking
// → retry → error → … (the hook injects more tokens each cycle).
```

逻辑很硬:模型压根没产出真正的响应,stop hook 没有任何**有意义的东西**可评估,这时候跑 hook 纯属火上浇油。注意它仍调 `executeStopFailureHooks`——`StopFailure` 是「失败专用」事件,让用户能在失败时挂通知逻辑,但**不会**走会触发重试的常规 `Stop` hook。

**堵口二:API 错误时同样跳过 stop hooks**(第 1531–1541 行)。当最后一条消息是 API 错误(限流、prompt-too-long、认证失败)时,`if (lastMessage?.isApiErrorMessage)` 直接 return,注释同款:模型从没产出真实响应,hook 评估它会造死亡螺旋。

**堵口三:保留 `hasAttemptedReactiveCompact` 不重置**(第 1568–1573 行)。当 stop hook 确实产出了阻断错误、需要把错误塞回去重试时,新 state **刻意保留** `hasAttemptedReactiveCompact` 而不清零,注释记了一次真实的无限循环事故:

```ts
// Resetting to false here caused an infinite loop: compact → still
// too long → error → stop hook blocking → compact → … burning
// thousands of API calls.
```

如果这里把反应式压缩守卫重置成 false,就会变成「压缩 → 仍超长 → 错误 → stop hook 阻断 → 又压缩」的循环,白烧上千次 API 调用。三道堵口防的是同一类病在不同触发点的复发,共同的不变量是:**hook 是给「模型产出了真实响应」准备的决策位;当模型根本没产出有效响应(超长/API 错误)时,绝不能让 hook 把控制权又导回重试。**

## 16.7 registerSkillHooks:第 15 章 skill 也能挂 hook

第 15 章讲 skill 的 frontmatter 字段时列过一个 `hooks`。它的落点就在 `registerSkillHooks`(`registerSkillHooks.ts` 第 20 行):skill 可以在自己的 frontmatter 里声明 hook,激活时这些 hook 被注册成**会话级 hook**(session-scoped),活到会话结束。

最精巧的是 `once: true` 的处理(第 36–43 行):带 `once` 的 hook 注册时挂一个 `onHookSuccess` 回调,**首次成功执行后自动把自己摘掉**(`removeSessionHook`)。这让 skill 能声明「我只想在被激活后跑一次」的一次性逻辑(比如初始化某个环境),跑完即焚,不会在后续每个事件点反复触发。它遍历 `HOOK_EVENTS`(同一张 27 事件枚举),把 skill frontmatter 里每个事件下的每个 matcher 的每个 hook 都 `addSessionHook` 进去。这条线把第 15 章的 skill 系统和本章的 hook 系统缝在了一起:**skill 不只是注入 prompt,还能在会话生命周期里挂载自己的回调。**

## 16.8 HTTP hook 的 SSRF 闸:能打本机,但打不了云元数据

`http` 类型的 hook 把数据 POST 到任意 URL,这是一个典型的 SSRF(服务端请求伪造)风险面——一个项目级配置的 hook 如果能请求 `169.254.169.254`,就能偷到云厂商的实例元数据(临时凭证)。`ssrfGuard.ts`(`isBlockedAddress`,第 42 行)是这道闸。

它的封禁清单(第 22–40 行注释)很全:`10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`(私网)、`169.254.0.0/16`(link-local,云元数据)、`100.64.0.0/10`(CGNAT,注释特意点名「Alibaba 100.100.100.200」也是元数据端点)、以及对应的 IPv6 私有段和 `::ffff:<v4>` 映射地址。但有一个**故意的放行**(第 12–14 行):

```
Loopback (127.0.0.0/8, ::1) is intentionally ALLOWED — local dev
policy servers are a primary HTTP hook use case.
```

环回地址放行,因为「本地开发的策略服务器」正是 HTTP hook 的主要用途。这是一处精确的边界拿捏:**封掉所有能横向打到内部基础设施的私有段,但放行本机**——既堵住 SSRF 偷凭证,又不误伤合法的本地策略服务器场景。注释还诚实标注了一个已知的绕过(第 15–17 行):当全局代理或沙箱网络代理在用时,DNS 解析由代理做,这个守卫对目标 host **实质上被绕过**,转而由沙箱代理自己的域名 allowlist 兜底。和第 15 章 `mcpInfoFromString` 的「server 名含 `__` 会解析错」一样——**把已知限制写进注释当文档**(第 7 章「错误即文档」透镜)。

## 16.9 Plugin manifest:一个目录打包所有定制面

Hook 是「单点回调」,plugin 则是「打包分发」。一个 plugin 就是一个目录,核心是 `.claude-plugin/plugin.json` 这份 manifest(`pluginLoader.ts` 第 1360 行,另兼容 legacy 的根目录 `plugin.json`)。`loadPlugin` 加载时做的事很能说明 plugin 能打包什么(第 1374–1387 行):它**并行**探测四个可选目录——`commands/`、`agents/`、`skills/`、`output-styles/`,manifest 没显式声明时就按目录约定自动发现。加上 hook(`hooks/hooks.json`)和 MCP server,一个 plugin 能把**前面整本书讲过的几乎所有定制面**装进一个可分发单元:

- **commands/**:自定义斜杠命令
- **agents/**:第 14 章的子 agent 定义
- **skills/**:第 15 章的 skill
- **hooks/hooks.json**:本章的 hook 配置
- **mcpServers**:第 15 章的 MCP server 配置
- **output-styles/**:输出风格

manifest 里 hook 文件的加载有一处 fail-loud 设计(第 1229–1231 行):如果 manifest 声明了 hooks 但 `hooks.json` 文件不存在,**直接报错**而非静默跳过——「If the manifest declares hooks, the file must exist」。声明了就必须有,缺失是配置错误,要让用户知道。

## 16.10 marketplace 与同形字防伪:不让假货冒充官方

Plugin 从哪来?从 **marketplace**。CCB 区分两个保留 marketplace:`BUILTIN_MARKETPLACE_NAME = 'builtin'`(`builtinPlugins.ts` 第 23 行,随 CLI 出厂)和 `OFFICIAL_MARKETPLACE_NAME = 'claude-plugins-official'`(`officialMarketplace.ts` 第 25 行,官方源)。围绕「官方」这个名号,有一整套防冒充机制(`plugins/schemas.ts`):

- **同形字攻击防护**(`NON_ASCII_PATTERN = /[^ -~]/`,第 79 行):marketplace 名只允许 ASCII。注释举的例子很具体——用西里尔字母 `а` 冒充拉丁 `a` 来伪装成 `anthropic`。`isBlockedOfficialName`(第 87 行)对任何含非 ASCII 字符的名字直接封禁,堵死「肉眼看着像官方、字节其实不同」的钓鱼。
- **保留名源校验**(`validateOfficialNameSource`,第 119 行 + `OFFICIAL_GITHUB_ORG = 'anthropics'`,第 107 行):保留名(官方 marketplace 名)只能来自官方 GitHub org——一个 marketplace 想叫官方保留名,它的 `repo` 必须以 `anthropics/` 开头,否则校验失败。

这是把「信任」从「名字看起来对」收紧到「名字 + 来源都对」。和第 15 章 MCP server 名 normalize、本章 SSRF allowlist 是同一类**边界处对外部输入不信任**的体现——marketplace 名是用户/第三方提供的,绝不能凭它自报的名号就授予「官方」信任。

## 16.11 plugin scoped 命名与环境注入

Plugin 贡献的 MCP server 要进第 15 章那个全局工具池,命名冲突怎么办?`addPluginScopeToServers`(`mcpPluginIntegration.ts` 第 341 行)给每个 server 名加 plugin 作用域前缀:

```ts
const scopedName = `plugin:${pluginName}:${name}`
```

这和第 15 章 MCP 工具的 `mcp__server__tool` 是同一种「用命名空间防冲突」的思路——`plugin:foo:bar` 保证不同 plugin 即便有同名 server 也不会撞车,scope 设为 `'dynamic'`(第 353 行)。

plugin 的 MCP server 进程启动时,CCB 还注入两个特殊环境变量(`resolvePluginMcpEnvironment`,第 512–513 行):`CLAUDE_PLUGIN_ROOT`(plugin 自己的目录,让 plugin 脚本能引用自带资源,和第 15 章 skill 的 `${CLAUDE_SKILL_DIR}` 同源)和 `CLAUDE_PLUGIN_DATA`(plugin 的数据目录)。env 变量插值同样走 allowlist——只有显式列出的变量会被解析,其余 `${VAR}` 引用保持原样,避免把宿主环境的敏感变量意外泄漏进 plugin 进程。

## 16.12 strictPluginOnlyCustomization:把四个定制面锁死到 plugin-only

最后一道、也是最能体现「fail-closed」的策略闸:`strictPluginOnlyCustomization`(`pluginOnlyPolicy.ts`)。企业管理员可以用这条 managed 策略,把**用户级和项目级**的定制面锁死,只允许 plugin 和 managed 源提供。受管的定制面恰好是 `CUSTOMIZATION_SURFACES = ['skills', 'agents', 'hooks', 'mcp']`(`types.ts` 第 241 行)——也就是这本书后三章讲的全部外部能力入口。

`isRestrictedToPluginOnly(surface)`(第 19 行)判定某个面是否被锁:策略为 `true` 锁全部四个面、数组形式只锁列出的、缺省不锁(默认开放)。一旦某个面被锁,**用户 `~/.claude/*`、项目 `.claude/*`、本地、flag、裸 mcp 等用户可控源全部被跳过**,只有 `ADMIN_TRUSTED_SOURCES`(第 40 行)放行:

```ts
const ADMIN_TRUSTED_SOURCES: ReadonlySet<string> = new Set([
  'plugin', 'policySettings', 'built-in', 'builtin', 'bundled',
])
```

每一条放行都有理由(第 30–38 行注释):`plugin` 由 `strictKnownMarketplaces` 单独把关、`policySettings` 本就是管理员控制的 managed 设置、`built-in`/`builtin`/`bundled` 随 CLI 出厂不是用户写的。`isSourceAdminTrusted`(第 58 行)用于逐项校验——比如注册 skill frontmatter 里的 hook 时,要先看这个 skill 的 source 是否 admin-trusted。

这道闸的安全语义是企业级的:在受管环境里,管理员不希望用户在自己的项目里塞任意 hook/MCP(那等于在受控机器上跑任意代码),于是把这四个面锁死,**只信任经过 marketplace 把关的 plugin 和管理员自己下发的 managed 配置**。这是第 7 章「fail-closed」「最小授权」透镜在分发层的终极体现——**默认开放方便个人用户,但管理员能一键收紧到「只跑可信来源的定制」。**

## 本章小结

- Hook 系统的基石是 27 个 `HOOK_EVENTS` 封闭枚举(三处镜像同步),覆盖工具/压缩/子 agent/权限/会话等全运行时生命周期;事件多为「前后成对/正常失败成对」;`ALWAYS_EMITTED_HOOK_EVENTS=['SessionStart','Setup']` 无条件触发,`MAX_PENDING_EVENTS=100` 给事件队列封顶。
- 四种 hook 类型(`command`/`prompt`/`http`/`agent`),按 `type` discriminated union 分流;共享 `if`(权限规则语法)在 spawn 前过滤、`once`/`timeout`/`statusMessage`;`AgentHookSchema` 绝不能加 `.transform()`(序列化会静默丢失用户 prompt,gh-24920);`HooksSchema = z.partialRecord(z.enum(HOOK_EVENTS), ...)`。
- 两套超时按生命周期分档:工具 hook `TOOL_HOOK_EXECUTION_TIMEOUT_MS=600_000`(10 分钟,要跑测试/lint),`SessionEnd` 仅 `1500ms`(用户在等退出,可经 env 放宽)。
- 退出码协议:0 成功(stdout 注入)、2 阻断(`blockingError`)、其它非零只展示给用户;JSON stdout 协议(`processHookJSONOutput`)更强:`continue:false`、`decision: approve/block`、`PreToolUse permissionDecision: allow/deny/ask` 直接接管第 8 章权限判定,未知类型一律 `throw`。
- `asyncRewake` 后台 hook 退出码 2 时经 `enqueuePendingNotification` 把模型「叫醒」(task-notification);故意不调 `background()` 以保内存 stdout 捕获,对 interrupt no-op、对硬取消才杀——和第 14 章后台 agent 同源。
- 三道死亡螺旋堵口(`query.ts`):prompt-too-long 不 fall through 到 stop hooks、API 错误同样跳过、stop hook 重试时保留 `hasAttemptedReactiveCompact` 不重置(曾致烧上千次 API 的无限循环);共同不变量:模型没产出有效响应时绝不让 hook 把控制权导回重试。
- `registerSkillHooks` 让第 15 章 skill 经 frontmatter 挂会话级 hook,`once:true` 经 `onHookSuccess` 首次成功后自摘;遍历同一张 27 事件枚举。
- HTTP hook 的 `ssrfGuard` 封私网/link-local/CGNAT/云元数据段,**故意放行环回**(本地策略服务器是主要用途);诚实标注代理在用时守卫被绕过、由沙箱域名 allowlist 兜底。
- Plugin = 一个目录 + `.claude-plugin/plugin.json` manifest,并行打包 commands/agents/skills/output-styles/hooks/mcpServers;manifest 声明了 hooks 但文件缺失则 fail-loud 报错。
- marketplace 防伪:`builtin` 与 `claude-plugins-official` 保留名,`NON_ASCII_PATTERN` 防同形字冒充、`validateOfficialNameSource` 要求保留名必须来自 `anthropics/` org。
- plugin scoped 命名 `plugin:${pluginName}:${name}` 防 MCP server 撞名,注入 `CLAUDE_PLUGIN_ROOT`/`CLAUDE_PLUGIN_DATA`(env 插值走 allowlist)。
- `strictPluginOnlyCustomization` 把 `CUSTOMIZATION_SURFACES=['skills','agents','hooks','mcp']` 四个面锁死到 plugin-only,跳过所有用户可控源,只放行 `ADMIN_TRUSTED_SOURCES`——fail-closed/最小授权在分发层的终极体现。

## 动手实验

1. **实验一:给 27 个事件归类** — 读 `coreSchemas.ts` 第 362 行的 `HOOK_EVENTS` 与 `hookEvents.ts` 的 `ALWAYS_EMITTED_HOOK_EVENTS`。把 27 个事件按「前后成对/正常失败成对/单点」三类归档,并对照本书前面章节标出每个事件挂载点对应哪一章的机制(如 `PreCompact`→第 12 章)。再论证:为什么 `SessionStart`/`Setup` 要无条件触发,而 `MAX_PENDING_EVENTS=100` 防的是什么?
2. **实验二:走一遍退出码与 JSON 双协议** — 读 `hooks.ts` 第 2785 行附近的退出码分支与 `processHookJSONOutput`(第 569 行)。构造三个 `PreToolUse` hook:(a) 退出码 2,(b) JSON 输出 `{"decision":"block"}`,(c) JSON 输出 `{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"ask"}}`。分别预测它们对一次工具调用的影响。再解释:为什么「未知 decision 类型」要 `throw` 而非忽略?
3. **实验三:复现并堵住死亡螺旋** — 读 `query.ts` 第 1441–1456、1531–1541、1568–1573 三处。为每一道堵口写出:它防的具体循环路径、触发条件、以及为什么「prompt-too-long / API 错误时不能 fall through 到常规 Stop hook,但仍调 `executeStopFailureHooks`」。把它和第 12 章 12.8 的三道防死循环闸并列,提炼共同的止损不变量。
4. **实验四:论证 plugin-only 策略的信任边界** — 读 `pluginOnlyPolicy.ts` 与 `types.ts` 第 241 行的 `CUSTOMIZATION_SURFACES`。回答:管理员设 `strictPluginOnlyCustomization: ['hooks','mcp']` 后,一个写在项目 `.claude/settings.json` 里的 hook 还会被加载吗?为什么 `plugin` 源能放行而 `projectSettings` 不能?再结合 16.10 的同形字防伪,说明「plugin 可信」这条链里 marketplace 把关扮演什么角色。

> **下一章预告**:到这里,从启动入口、主循环、工具系统、权限沙箱、子 agent、上下文工程、记忆、到 Skill/MCP/Hooks/Plugin,CCB 的运行时已被逐层拆开。第 17 章是收尾综合章——不再引入新代码,而是把贯穿全书的观察提炼成**与语言、框架无关、可迁移**的工程原则:fail-closed 默认、约束只收紧、错误即文档、边界处不信任、保结构删内容、不变量写进类型与测试、可信度是一条流水线。每一条都回指到前面的具体章节作为证据,看这些散落在 CCB 各处的设计决策如何收敛成一套可复用的 harness 工程方法论。
