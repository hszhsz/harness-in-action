# 第 14 章　子 agent：AgentTool 与工具子集的三层裁剪

> 前面讲的记忆、压缩，都是在**给主 agent 维护一条尽可能干净的上下文**。但有些任务天生不该塞进主线程——「在整个代码库里找出所有调用点」会塞回来几十个文件，「设计一套实现方案」要反复试探。CCB 的答案是把这类任务**委派给子 agent**：开一个独立的 agent，给它一段提示、一套**裁剪过的工具子集**、一个隔离的上下文，让它跑完后只把**一份报告**收束回主线程。本章拆 `AgentTool`——它怎么选 agent、怎么给子 agent 装工具池、为什么工具要分三层裁剪、同步与异步/后台两种生命周期如何收尾,以及 fork 路径如何为了缓存一致性走一条特殊的「继承父上下文」分支。

## 14.1 一个工具,两种委派语义

`AgentTool`（`packages/builtin-tools/src/tools/AgentTool/AgentTool.tsx`,`buildTool` 于第 284 行）的入参 schema 很小但能表达多种委派（第 137 行 `baseInputSchema`）:

```ts
description: string         // 3-5 词的任务描述
prompt: string             // 交给子 agent 的任务
subagent_type?: string     // 用哪种专门 agent
model?: 'sonnet'|'opus'|'haiku'  // 覆盖模型
run_in_background?: boolean // 后台运行
```

`call()` 内部据此分流出几种执行路径(第 322 行起):**多 agent 团队 spawn**(`teamName && name`,走 `spawnTeammate`)、**远程隔离**(ant-only,`isolation === 'remote'`,走 CCR)、**fork 子 agent**(`subagent_type` 省略且 fork gate 开)、以及最常见的**普通同步/异步子 agent**。这一章聚焦后两者——它们体现了「委派」最核心的工程问题:**子 agent 该拿到什么、它的上下文怎么和主线程隔离、结果怎么回来**。

`isReadOnly()` 返回 `true`(第 1458 行),注释点出原因:「delegates permission checks to its underlying tools」——AgentTool 自己不做权限判断,把判断下推给子 agent 实际调用的每个工具。这是「委派」在权限层的体现:授权检查发生在叶子工具,不在委派入口。

## 14.2 子 agent 的工具池:独立组装,不继承父限制

最关键的设计在第 717–726 行——**子 agent 的工具池是独立组装的,不受父 agent 工具限制影响**:

```ts
// Workers always get their tools from assembleToolPool with their own
// permission mode, so they aren't affected by the parent's tool restrictions.
const workerPermissionContext = {
  ...appState.toolPermissionContext,
  mode: selectedAgent.permissionMode ?? 'acceptEdits',
}
const workerTools = assembleToolPool(workerPermissionContext, appState.mcp.tools)
```

注意 `mode: selectedAgent.permissionMode ?? 'acceptEdits'`——子 agent 默认跑在 `acceptEdits` 权限模式下,而不是继承主线程的模式。这是个深思熟虑的取舍:子 agent 是在执行一个**被明确委派、范围受限的子任务**,给它一个偏自动的权限模式能减少它卡在确认弹窗上(它也没有 UI 去问用户)。注释还说明为什么这块计算放在 AgentTool 而非 `runAgent`:避免 `runAgent` 反向 import `tools.ts` 造成循环依赖。

但「独立组装」不等于「想要什么都行」。子 agent 拿到的工具要先过**三层裁剪**,这是本章的骨架。

## 14.3 第一层:ALL_AGENT_DISALLOWED_TOOLS——有些工具子 agent 永远碰不到

`filterToolsForAgent`(`agentToolUtils.ts` 第 70 行)是第一道闸。它逐个过滤工具,核心是 `ALL_AGENT_DISALLOWED_TOOLS`(`src/constants/tools.ts` 第 44 行):

```ts
export const ALL_AGENT_DISALLOWED_TOOLS = new Set([
  TASK_OUTPUT_TOOL_NAME,
  EXIT_PLAN_MODE_V2_TOOL_NAME,
  ENTER_PLAN_MODE_TOOL_NAME,
  ...(process.env.USER_TYPE === 'ant' ? [] : [AGENT_TOOL_NAME]),  // 非 ant 禁套娃
  ASK_USER_QUESTION_TOOL_NAME,
  TASK_STOP_TOOL_NAME,
  ...(feature('WORKFLOW_SCRIPTS') ? [WORKFLOW_TOOL_NAME] : []),
  LOCAL_MEMORY_RECALL_TOOL_NAME,   // 跨会话用户笔记,只留主线程
  VAULT_HTTP_FETCH_TOOL_NAME,      // 触碰用户密钥,只留主线程
])
```

每一条禁用都对应一个具体风险:

- **`AGENT_TOOL_NAME`(非 ant 禁用)**:防止子 agent 再 spawn 子 agent 的**无限套娃**。外部用户的 agent 拿不到 Agent 工具;ant 内部放开以支持嵌套 agent。
- **`ASK_USER_QUESTION` / `TASK_STOP`**:子 agent 没有面向用户的交互通道,问用户、停任务都是主线程的抽象。
- **`LOCAL_MEMORY_RECALL` / `VAULT_HTTP_FETCH`**:第 13 章讲的跨会话记忆、以及触碰用户密钥的 vault fetch——注释写得明白:「Cross-session user notes shouldn't be siphoned by spawned subagents」「vault HTTP fetch is even more sensitive (touches user secrets)」。这是**最小授权**的直接体现:越敏感的能力,授权范围越收窄到主线程。

`CUSTOM_AGENT_DISALLOWED_TOOLS`(第 64 行)在此基础上再叠加——它直接 `...ALL_AGENT_DISALLOWED_TOOLS` 起步(目前等于全集),为「自定义 agent 比内置 agent 受更多限制」预留了扩展位。`filterToolsForAgent` 里 `if (!isBuiltIn && CUSTOM_AGENT_DISALLOWED_TOOLS.has(...))` 正是这层区分:不可信的用户自定义 agent 走更严的清单。

## 14.4 第二层:异步 agent 的工具白名单——从黑名单切到白名单

第二层只对**异步/后台 agent**生效,而且语义反转:前面是「禁用清单」(默认放行、列出的拦掉),这里是 `ASYNC_AGENT_ALLOWED_TOOLS`(第 71 行)**白名单**(默认拦掉、列出的才放行):

```ts
export const ASYNC_AGENT_ALLOWED_TOOLS = new Set([
  FILE_READ_TOOL_NAME, WEB_SEARCH_TOOL_NAME, TODO_WRITE_TOOL_NAME,
  GREP_TOOL_NAME, WEB_FETCH_TOOL_NAME, GLOB_TOOL_NAME,
  ...SHELL_TOOL_NAMES, FILE_EDIT_TOOL_NAME, FILE_WRITE_TOOL_NAME,
  NOTEBOOK_EDIT_TOOL_NAME, SKILL_TOOL_NAME, ...
])
```

`filterToolsForAgent` 里 `if (isAsync && !ASYNC_AGENT_ALLOWED_TOOLS.has(tool.name)) return false`——后台 agent 只能用这张白名单上的工具。为什么对异步收得更紧?因为后台 agent **脱离了主线程的生命周期**(用户按 ESC 取消主线程时它还活着,见 14.7),任何需要主线程状态的工具(Agent 防递归、TaskOutput、ExitPlanMode、TaskStop)在后台都是无意义甚至危险的——文件尾部的注释把每一条都列了原因。

这里有个精细的例外:**in-process teammate**(进程内队友)在 agent swarm 开启时,可以突破异步白名单拿到 `AGENT_TOOL_NAME`(spawn 同步子 agent)和 `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS`(任务协调、SendMessage、cron),因为队友需要协作能力。这是「白名单之上再开一道窄缝」,缝口被 `call()` 里的运行时校验兜住(队友不能 spawn 后台 agent / 不能 spawn 队友)。

## 14.5 第三层:agent 定义自己的 tools / disallowedTools

前两层是 harness 强加的硬边界,第三层是**每个 agent 定义自报**的工具范围。`resolveAgentTools`(第 122 行)处理:

- agent 定义里的 `disallowedTools` 转成 set,从可用工具里减掉;
- 若 `tools` 是 `undefined` 或 `['*']`,`hasWildcard = true`,放行(过滤掉 disallowed 之后的)全部;
- 否则按 `tools` 里列的名字逐个解析,命中的进 `validTools`,没命中的进 `invalidTools`。

三个内置 agent 正好示范了从宽到窄的谱系:

- **general-purpose**(`generalPurposeAgent.ts` 第 29 行):`tools: ['*']`——通配,拿到(裁剪后的)全部工具,因为它要处理开放式任务。
- **Explore**(`exploreAgent.ts` 第 63 行):`disallowedTools: [Agent, ExitPlanMode, FileEdit, FileWrite, NotebookEdit]`——一个**只读搜索** agent,显式禁掉所有写操作。它还设 `model: haiku`(外部用户,图快)和 `omitClaudeMd: true`,注释解释:只读搜索「doesn't need commit/PR/lint rules from CLAUDE.md」——连第 13 章的项目记忆都不注入,因为「The main agent has full context and interprets results」。
- **Plan**(`planAgent.ts` 第 85 行):`tools: EXPLORE_AGENT.tools`——直接复用 Explore 的工具集(同样只读),因为做计划也只需要读和搜。

这三层叠起来,语义是清晰的:**第一层 harness 划死不可逾越的红线(敏感工具/防递归),第二层按执行模式(异步)收紧,第三层让每个 agent 把自己的范围再收窄到任务所需**。三层都只会让工具**变少**,不会变多——这正是第 7 章「约束只收紧」那条透镜在子 agent 系统里的复现。

## 14.6 fork 路径:为缓存一致性而「继承父上下文」

普通子 agent 是「干净起步」——自己的系统提示、自己的工具池、一条全新的用户消息。但 CCB 还有一条 **fork 路径**(`isForkPath`,第 415 行),它反其道而行:**让子 agent 继承父会话的系统提示和完整工具数组**,为的是**缓存一致**。

第 777–786 行是这条路径的精华:

```ts
override: isForkPath ? { systemPrompt: forkParentSystemPrompt } : ...,
availableTools: isForkPath ? filterParentToolsForFork(toolUseContext.options.tools) : workerTools,
forkContextMessages: isForkPath ? toolUseContext.messages : undefined,
...(isForkPath && { useExactTools: true }),
```

注释解释了为什么 fork 不用 `workerTools`:`workerTools` 在 `bubble` 权限模式下重建,工具定义的序列化和父线程不同,**第一个不一致的工具就会击穿缓存**。所以 fork 必须传父线程的**精确工具数组**(`useExactTools: true`),让 API 请求前缀逐字节对齐——这和第 12 章 autocompact 的 cache-sharing fork 是同一套「缓存 key 必须一致」的偏执。

但「继承父工具全集」和「子 agent 禁用清单」会打架:`useExactTools: true` 让 `runAgent` **跳过** `resolveAgentTools`,于是第一层的 `ALL_AGENT_DISALLOWED_TOOLS` 被绕过了。CCB 用 `filterParentToolsForFork`(`agentToolFilter.ts` 第 21 行)补这个洞:

```ts
export function filterParentToolsForFork(parentTools: readonly Tool[]): Tool[] {
  return parentTools.filter(t => !ALL_AGENT_DISALLOWED_TOOLS.has(t.name))
}
```

注释把它叫「gate layer 2」——fork 路径绕过了 layer 1,所以在工具数组进入 fork **之前**手动套一遍同样的禁用清单。新 fork(AgentTool.tsx)和恢复 fork(resumeAgent.ts)两条路径都必须调它。这是一个很典型的「**为性能开了特例,就必须为特例补回安全闸**」——fork 为缓存绕过了正常裁剪,那就在旁路上重新加一道等价的闸,确保敏感工具(LocalMemoryRecall、VaultHttpFetch)在任何路径上都进不了子 agent。

fork 还有一道防递归:fork 子进程为了缓存一致**保留了 Agent 工具**,所以不能靠工具裁剪挡递归,改在 `call()` 里运行时拒绝(第 425–430 行)——主检查用 `querySource`(autocompact 重写消息后仍存活),fallback 扫消息(`isInForkChild`)。

## 14.7 同步与异步:两套生命周期,同一套收尾不变量

`shouldRunAsync`(第 709 行)决定走哪条路:`run_in_background`、agent 定义的 `background: true`、coordinator 模式、fork 实验(`forceAsync`)、assistant 模式等任一为真即异步。两条路径的差异和共性都值得记:

**异步路径**(第 829 行起):立即 `registerAsyncAgent` 注册后台任务,然后 `void runWithAgentContext(...)` 把整个执行**脱钩**到后台——注释特别说明它**不挂主线程的 abort controller**:「background agents should survive when the user presses ESC to cancel the main thread」。后台 agent 由 `chat:killAgents` 显式杀,而非随主线程取消。`call()` 立刻返回 `status: 'async_launched'` + outputFile 路径,主 agent 据此知道「任务在跑、结果会异步到」。

**同步路径**(第 913 行起):在一个循环里 `for await` 拉子 agent 的消息流,边收边 `onProgress` 转发进度。它有一个巧妙的「**运行中可转后台**」机制:用 `Promise.race` 同时等「下一条消息」和「backgroundSignal」(第 1042 行),用户随时可以把一个跑得太久的同步 agent 转后台——超过 `PROGRESS_THRESHOLD_MS = 2000` 还会弹出「可后台化」提示。一旦 backgrounded,剩余执行被搬进一个新的 `void` 闭包继续在后台跑完。

两条路径**收尾的不变量是一致的**:无论成功、失败、abort、还是中途转后台,`finally` 块都要 (1) 清前台任务注册、(2) `clearInvokedSkillsForAgent` 清掉本 agent 调用过的 skill(防全局 map 累积)、(3) `clearDumpState`、(4) `cleanupWorktreeIfNeeded` 清理 worktree。worktree 清理本身也很克制(第 797 行):**有改动就保留**(`hasWorktreeChanges`),没改动才删——子 agent 的劳动成果不能被收尾逻辑误删。

## 14.8 结果收束:把子 agent 压成一份报告回主线程

子 agent 跑完,`finalizeAgentTool` 把它的消息流压成一个 `agentResult`,再由 `mapToolResultToToolResultBlockParam`(第 1490 行)渲染成回主线程的 tool_result。这里有几处「把 token 花在刀刃上」的精算:

- **空输出兜底**(第 1554 行):子 agent 若没产出内容,tool_result 会变成只剩 agentId/usage 的元数据块,「Some models read that as 'nothing to act on' and end their turn」——所以显式塞一句「(Subagent completed but returned no output.)」给主 agent 一个可反应的锚点。
- **一次性内置 agent 省 trailer**(第 1568 行):`ONE_SHOT_BUILTIN_AGENT_TYPES = {Explore, Plan}`(`constants.ts`)这类「跑一次就返报告、绝不会被 SendMessage 续聊」的 agent,跳过 agentId/SendMessage/usage 尾巴。注释算了账:「~135 chars × 34M Explore runs/week ≈ 1-2 Gtok/week」——一个看似无害的尾部元数据块,放大到全机队就是每周 1-2 Gtok 的浪费。
- **可续聊 agent 带 SendMessage 提示**:非一次性 agent 的 tool_result 末尾附 `agentId` 和「use SendMessage with to: '...' to continue」,主 agent 据此能再唤醒它。
- **异步结果带防重叠指导**(第 1531 行):`async_launched` 的 tool_result 会提示主 agent「Do not duplicate this agent's work — avoid working with the same files or topics」——后台 agent 在跑时,主 agent 不该去碰同样的文件。

`checkPermissions`(第 1475 行)还有一处 ant-only 的 auto 模式钩子:auto 模式下 spawn 子 agent 走 `passthrough`(交给第 10 章的分类器判),其余模式一律 `allow`——又是「委派入口不自己判权限」的体现。

## 本章小结

- `AgentTool` 把大/杂任务**委派给子 agent**:独立上下文、裁剪过的工具子集、跑完只收束**一份报告**回主线程;`isReadOnly()=true` 因为它把权限判断**下推给子 agent 的叶子工具**。
- 子 agent 工具池**独立组装**(`assembleToolPool`,默认 `acceptEdits` 模式),不受父限制影响,但要过**三层裁剪**——三层都只让工具变少。
- 第一层 `ALL_AGENT_DISALLOWED_TOOLS`:防递归(非 ant 禁 Agent)、无用户通道(AskUserQuestion/TaskStop)、敏感能力只留主线程(LocalMemoryRecall/VaultHttpFetch);`CUSTOM_AGENT_DISALLOWED_TOOLS` 为自定义 agent 留更严扩展位。
- 第二层 `ASYNC_AGENT_ALLOWED_TOOLS`:对异步 agent **从黑名单切到白名单**(默认拦、列出才放),因后台 agent 脱离主线程生命周期;in-process teammate 可突破窄缝拿 Agent + 协调工具,缝口由运行时校验兜。
- 第三层 agent 自报 `tools`/`disallowedTools`(`resolveAgentTools`):general-purpose 用 `['*']`、Explore 只读(禁所有写 + haiku + `omitClaudeMd`)、Plan 复用 Explore 工具集。
- fork 路径为**缓存一致**继承父系统提示 + 父精确工具数组(`useExactTools` 跳过 `resolveAgentTools`),于是用 `filterParentToolsForFork`(gate layer 2)在旁路补回第一层禁用清单——**为性能开特例就必须为特例补安全闸**;fork 防递归改在 `call()` 运行时拒绝。
- 同步/异步两套生命周期:异步不挂主线程 abort(ESC 不杀后台 agent)、立即返 `async_launched`;同步 `for await` 流式收 + `Promise.race` 支持运行中转后台;两者 `finally` 收尾不变量一致(清任务/skill/dumpState/worktree),worktree **有改动才保留**。
- 结果收束精算:空输出塞占位句、一次性内置 agent(Explore/Plan)省 trailer(~1-2 Gtok/week)、可续聊 agent 带 SendMessage 提示、异步结果带「勿重叠工作」指导。

## 动手实验

1. **实验一:画出三层工具裁剪** — 读 `filterToolsForAgent`(`agentToolUtils.ts` 第 70 行)与 `src/constants/tools.ts` 的四个 Set。假设一个**自定义异步 agent**,`tools: ['*']`。逐层推演它最终能拿到哪些工具:先过 `ALL_`/`CUSTOM_` 黑名单,再过 `ASYNC_` 白名单,最后过 `resolveAgentTools` 的通配。哪些工具在哪一层被砍掉?
2. **实验二:论证 fork 的「绕过+补闸」** — 读 `AgentTool.tsx` 第 777–786 行与 `agentToolFilter.ts`。回答:为什么 fork 路径不能用 `workerTools` 而要传父线程精确工具数组?`useExactTools` 跳过了哪个函数,从而绕过了哪一层裁剪?`filterParentToolsForFork` 补的是哪一层?如果没有这道补闸,哪两个敏感工具会泄漏进 fork 子 agent?
3. **实验三:跟踪「同步转后台」** — 读同步路径(第 913 行起)的 `Promise.race`(第 1042 行)和 backgrounding 分支(第 1058 行起)。论证:用户在子 agent 跑到一半时把它转后台,剩余执行如何搬进新的 `void` 闭包?为什么这个闭包要用独立的 `stopBackgroundedSummarization` 而非共享前台的?转后台后 worktree 为什么**不能**在前台 `finally` 里清理?
4. **实验四:算一笔 trailer 的账** — 读 `mapToolResultToToolResultBlockParam`(第 1490 行)和 `ONE_SHOT_BUILTIN_AGENT_TYPES`(`constants.ts`)。解释:为什么 Explore/Plan 要跳过 agentId/SendMessage/usage 尾巴而普通 agent 不跳?注释里「~135 chars × 34M runs/week」的账怎么算?再看空输出兜底(第 1554 行)——为什么「返回空」比「返回一句占位」对主 agent 更危险?

> **下一章预告**:子 agent 是把**能力**委派出去,而 Skills 和 MCP 是把**能力引进来**——前者按需注入领域知识与脚本,后者把外部工具服务器接入工具池。第 15 章我们拆 CCB 的 Skill 系统(动态发现、按需加载、`clearInvokedSkillsForAgent` 那条本章已埋下的清理线)与 MCP 集成(服务器连接、工具命名 `mcp__server__tool`、`requiredMcpServers` 的就绪等待),看「外部能力」如何在不污染主上下文的前提下被纳入。
