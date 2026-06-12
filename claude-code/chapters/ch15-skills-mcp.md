# 第 15 章　外部能力：Skill 动态发现与 MCP 工具接入

> 第 14 章讲的子 agent 是把**能力委派出去**——开一个隔离上下文跑完收一份报告。本章讲反方向：把**外部能力引进来**。CCB 有两套互补的「引入」系统：**Skill** 把领域知识与脚本按需注入对话(声明式 markdown + 可选 bash 注入),**MCP** 把外部工具服务器接进工具池(`mcp__server__tool` 命名 + 连接状态机)。两者共同的工程命题是同一条:**外部能力必须在不污染主上下文、不破坏缓存、不绕过权限的前提下被纳入**。这一章拆 `SkillTool` 怎么列出/加载/执行 skill、skill 怎么动态发现与条件激活、`clearInvokedSkillsForAgent` 那条第 14 章埋下的清理线、以及 MCP 的工具命名、`requiredMcpServers` 就绪等待、instructions delta 三处缓存偏执。

## 15.1 列表只用 1% 窗口:发现与加载分离

Skill 系统的第一个设计决策藏在预算里。模型要「知道有哪些 skill 可用」,但每个 skill 的完整内容(可能上千行 markdown)不该全塞进上下文。CCB 把这件事拆成**发现**(listing)和**加载**(invoke)两步,只给 listing 留极小预算(`prompt.ts` 第 19 行):

```ts
// Skill listing gets 1% of the context window (in characters)
export const SKILL_BUDGET_CONTEXT_PERCENT = 0.01
export const CHARS_PER_TOKEN = 4
export const DEFAULT_CHAR_BUDGET = 8_000 // Fallback: 1% of 200k × 4
```

listing 只占上下文窗口的 **1%**(按字符算),`getCharBudget` 用 `contextWindowTokens × 4 × 0.01` 算出预算,200K 窗口约 8000 字符。每条 listing 还有 per-entry 硬上限 `MAX_LISTING_DESC_CHARS = 1536`(第 28 行)。注释把账说得很直白:「The listing is for discovery only — the Skill tool loads full content on invoke, so verbose whenToUse strings waste turn-1 cache_creation tokens without improving match rate」——listing 只为「让模型挑」,完整内容在 invoke 时才加载,所以冗长的 `whenToUse` 是在白烧第一轮的 cache_creation token,对匹配率毫无帮助。

`formatCommandsWithinBudget`(第 70 行)在预算内的截断策略也分层:先试全量描述,超预算时把 **bundled skill**(CCB 自带的)圈出来**永不截断**,只截非 bundled 的描述。这是「自带能力优先保真、第三方能力按预算压缩」的取舍——和第 14 章「三层裁剪只让工具变少」是同一种「按可信度分配资源」的思路。

## 15.2 Skill 加载:目录约定、frontmatter、realpath 去重

skill 的加载在 `getSkillDirCommands`(`loadSkillsDir.ts` 第 638 行,`memoize` 缓存)。它从四个来源**并行**读取(第 679 行起):managed(`policySettings`)、user(`~/.claude/skills`)、project(从 cwd 上溯各级 `.claude/skills`)、以及 `--add-dir` 附加目录,外加 legacy `/commands/` 目录。`/skills/` 目录**只认目录格式** `skill-name/SKILL.md`(第 424 行注释:「Single .md files are NOT supported」),单个 `.md` 文件被忽略。

frontmatter 的解析集中在 `parseSkillFrontmatterFields`(第 185 行),它抽出 `description`/`whenToUse`/`allowed-tools`/`model`/`effort`/`context`/`agent`/`paths`/`hooks`/`disable-model-invocation`/`user-invocable` 等字段。几个边界值得记:`model: 'inherit'` 被当成 `undefined`(继承主线程模型);`user-invocable` 缺省为 `true`;非法 `effort` 值记 debug 日志后降级为 `undefined`——又是「未知/非法输入优雅降级,不报错中断」的边界处理。

去重用的是 `realpath` 而非 inode(第 118 行 `getFileIdentity`)。注释解释为什么:某些虚拟/容器/NFS 文件系统报告的 inode 不可靠(inode 0),ExFAT 还有精度损失(引 issue #13893),所以靠 `realpath` 解析符号链接到规范路径来识别「同一个文件经不同路径访问」。去重是**先并行算所有文件身份、再同步按 first-wins 去重**(第 728–763 行),保证顺序确定性(managed > user > project 的来源优先级)。

## 15.3 动态发现与条件激活:skill 也能「按你碰的文件」出现

并非所有 skill 都在启动时加载。CCB 有两套「运行中才让 skill 出现」的机制,都和第 13 章的条件规则同源。

**动态目录发现**(`discoverSkillDirsForPaths`,第 861 行):当模型读/写某个文件时,从该文件父目录**向上走到 cwd(但不含 cwd)**,沿途找 `.claude/skills` 目录。两处精算:第一,`dynamicSkillDirs` 集合记录「已检查过的路径(命中或未命中)」,避免每次 Read/Write 都对不存在的目录重复 stat(常见情况就是不存在);第二,找到 skills 目录后还要查 `isPathGitignored`——被 gitignore 的目录(如 `node_modules/pkg/.claude/skills`)不静默加载,但注释强调「Fails open outside a git repo」「the invocation-time trust dialog is the actual security boundary」:git 检查只是第一道筛,真正的安全边界是调用时的信任弹窗。

**条件 skill**(`paths` frontmatter,第 997 行 `activateConditionalSkillsForPaths`):带 `paths` glob 的 skill 启动时**不进可用列表**,而是存进 `conditionalSkills` map,等到模型操作的文件匹配了 glob 才用 `ignore()` 库激活(移进 `dynamicSkills`)。这和第 13 章 `.claude/rules/*.md` 的条件规则用的是同一套 gitignore 风格匹配——`parseSkillPaths`(第 159 行)同样剥掉 `/**` 后缀、把纯 `**` 当成「无约束」。激活过的名字记在 `activatedConditionalSkillNames`,**跨缓存清理仍存活**(同一会话内不会反复激活)。

## 15.4 两种执行:inline 展开 vs fork 子 agent

`SkillTool.call()`(`SkillTool.ts` 第 584 行)按 skill 的 `context` 字段分流出两种执行语义:

**inline(默认)**:走 `processPromptSlashCommand` 把 skill 的 markdown 展开成 prompt,作为 `newMessages` 注入**当前**对话,模型在主线程里接着处理。它通过 `contextModifier`(第 779 行)把 skill 声明的 `allowedTools`、`model`、`effort` 临时叠加到上下文里。这里有一处第 12/13 章呼应的细节(第 813 行):若 skill 指定 `model: opus`,要把会话原有的 `[1m]` 后缀带过去,否则「an opus[1m] session drops the effective window to 200K and trips autocompact」——模型覆盖不能误把有效窗口砍小,否则触发自动压缩。

**fork**(`context: 'fork'`,第 626 行 `executeForkedSkill`):skill 在一个**隔离子 agent**里跑(自己的 token 预算),和第 14 章的子 agent 同源——它直接调 `runAgent`,`isAsync: false`,跑完用 `extractResultText` 把消息流压成一段结果文本回主线程。这是「重的、会产生大量中间步骤的 skill」的隔离方案:不让它的中间过程污染主上下文,只收一份结果。

两种执行都会把 skill 注册进 `addInvokedSkill`(`state.ts` 第 1514 行),key 是 `${agentId ?? ''}:${skillName}`。这个注册的目的在第 12 章已埋下伏笔:**让 skill 内容在 autocompact 后能被「重建现场」恢复**(`createSkillAttachmentIfNeeded`,按 skill 截断保头部,预算 25K)。

## 15.5 clearInvokedSkillsForAgent:第 14 章埋下的清理线

`addInvokedSkill` 往一个**全局 map**(`STATE.invokedSkills`)里塞内容,如果只进不出,长会话里它会无限累积。CCB 为此设了按 agent 粒度的清理(`state.ts` 第 1561 行):

```ts
export function clearInvokedSkillsForAgent(agentId: string): void {
  for (const [key, skill] of STATE.invokedSkills) {
    if (skill.agentId === agentId) {
      STATE.invokedSkills.delete(key)
    }
  }
}
```

这正是第 14 章 14.7 提到的子 agent `finally` 收尾不变量之一——子 agent 跑完(无论成功/失败/abort),要清掉**本 agent**调用过的 skill,防止全局 map 随子 agent 数量累积。fork skill 自己也在 `finally` 里调它(`SkillTool.ts` 第 291 行)。粒度设计很讲究:`clearInvokedSkillsForAgent(agentId)` 只删某个 agent 的,主线程的 skill(`agentId === null`)不受影响;而 `clearInvokedSkills(preservedAgentIds)`(第 1547 行)反过来——清掉**除保留集之外**的所有,用于更粗的会话级清理。一个 map,两种清理粒度,对应「子 agent 退出」和「会话重置」两个不同的生命周期事件。

## 15.6 bash 注入与 MCP skill 的安全边界

本地 skill 的 markdown 支持两种动态替换(`loadSkillsDir.ts` 第 344 行 `getPromptForCommand`):`${CLAUDE_SKILL_DIR}` 换成 skill 自己的目录(让 skill 能引用 bundled 脚本)、`${CLAUDE_SESSION_ID}` 换成会话 ID,以及 `!`...`` 形式的 inline bash 注入(`executeShellCommandsInPrompt`)。

但**MCP skill 是远程、不可信的**,所以这里有一道硬边界(第 371–374 行):

```ts
// Security: MCP skills are remote and untrusted — never execute inline
// shell commands (!`…` / ```! … ```) from their markdown body.
// ${CLAUDE_SKILL_DIR} is meaningless for MCP skills anyway.
if (loadedFrom !== 'mcp') {
  finalContent = await executeShellCommandsInPrompt(...)
}
```

`loadedFrom === 'mcp'` 的 skill **绝不执行内联 shell**——远程 markdown 里的 `!`rm -rf`` 不会被当成命令跑。这是「边界处不信任输入」(第 7 章透镜)在 skill 系统的直接体现:本地文件是用户自己写的(可信),MCP server 推来的 markdown 是外部的(不可信),同样的语法在两种来源下被区别对待。

权限层也有对应设计。`SkillTool.checkPermissions`(第 436 行)先查 deny/allow 规则,然后有一道 `skillHasOnlySafeProperties` 自动放行(第 533 行):只有当 skill 的属性**全部**落在 `SAFE_SKILL_PROPERTIES` 白名单(第 879 行)内才自动允许,否则要弹窗确认。注释点出这是**allowlist 而非 blocklist**:「new properties added in the future default to requiring permission」——将来给 skill 加的任何新属性,默认都要确认,直到被显式审查并加进白名单。又一处「fail-closed」:未知能力默认收紧。

## 15.7 MCP 工具命名:mcp__server__tool 与它的已知裂缝

MCP server 的工具接进工具池后,命名遵循固定格式 `mcp__server__tool`。`buildMcpToolName`(`mcpStringUtils.ts`)和它的逆操作 `mcpInfoFromString` 是一对:

```ts
export function buildMcpToolName(serverName, toolName) {
  return `${getMcpPrefix(serverName)}${normalizeNameForMCP(toolName)}`
}
```

server 名先过 `normalizeNameForMCP`(`normalization.ts`):把不符合 API pattern `^[a-zA-Z0-9_-]{1,64}$` 的字符(含点、空格)全换成下划线;claude.ai 前缀的 server 还要折叠连续下划线、剥首尾下划线,**防止干扰 `__` 这个分隔符**。

但这套命名有一道诚实标注的裂缝(`mcpInfoFromString` 注释):「If a server name contains `__`, parsing will be incorrect」——server 名里若含双下划线,`mcp__my__server__tool` 会被错误解析成 server=`my`、tool=`server__tool`。CCB 没有强行修复,而是把限制写进注释并说明「rare in practice」。这是「把已知限制当文档」(第 7 章「错误即文档」透镜)的典型:与其搞一套复杂的转义,不如明说边界、靠 normalize 把常见情况兜住。权限匹配也专门用全限定名(`getToolNameForPermissionCheck`),避免 deny 规则 `Write` 误伤共享显示名的 MCP 替换工具。

## 15.8 连接状态机与 requiredMcpServers 就绪等待

MCP server 的连接是一个**五态状态机**(`types.ts` 第 221 行):`connected`(已连且认证,带 capabilities/instructions)、`failed`、`needs-auth`、`pending`(连接中,带 reconnectAttempt)、`disabled`。关键不变量:**只有 `connected` 且认证过的 server 才会有工具**——一个连上但没认证的 server,`mcp.tools` 里没有它的任何工具。

这就引出第 14 章提到的 `requiredMcpServers` 就绪等待(`AgentTool.tsx` 第 475 行)。某些 agent 定义声明「我需要某些 MCP server」,但 agent 可能在 server 还在 `pending` 时就被调用——这是个竞态。CCB 的处理是**轮询等待**:

```ts
const MAX_WAIT_MS = 30_000
const POLL_INTERVAL_MS = 500
const deadline = Date.now() + MAX_WAIT_MS
while (Date.now() < deadline) {
  await sleep(POLL_INTERVAL_MS)
  currentAppState = toolUseContext.getAppState()
  // Early exit: 任一 required server 已 failed → 不再等
  if (hasFailedRequiredServer) break
  if (!stillPending) break
}
```

最多等 30 秒、每 500ms 轮询一次,但有**提前退出**:只要任一 required server 已经 `failed`,就不必再等其他 pending 的——「the check will fail regardless」。等待结束后,从 `mcp.tools` 反推「真正有工具的 server」(即认证成功的),若仍缺失就抛错并提示用户 `/mcp` 去配置认证。这是「在能力可能尚未就绪时,给一个有界的等待窗口,而非无限等或立即失败」的工程权衡。

## 15.9 instructions delta:让 MCP 指令不每 turn 炸缓存

MCP server 可以在握手时(`InitializeResult`)带一段 `instructions`,告诉模型怎么用它。这段指令要注入系统提示,但**晚连接的 server**会带来一个缓存问题:如果每个 turn 都把当前所有 server 的指令重新拼进系统提示,一旦有 server 中途连上,系统提示的字节就变了,prompt cache 整个失效。

`mcpInstructionsDelta.ts` 用「**增量 attachment**」解决。`getMcpInstructionsDelta`(第 55 行)把「当前连接的、有指令的 server」和「本对话已经播报过的」做 diff,只产出**新增/移除**的部分,写成持久化的 `mcp_instructions_delta` attachment。几个精算点:

- **按 server NAME 而非内容 diff**(第 52 行注释):「Instructions are immutable for the life of a connection (set once at handshake)」——指令在连接生命周期内不变,所以只需比名字,不必比内容。
- **重建用扫描历史**:`announced` 集合通过扫历史里所有 `mcp_instructions_delta` attachment 重建(`addedNames` 加、`removedNames` 减),无状态可恢复。
- **gate 切换形态**:`isMcpInstructionsDeltaEnabled`(第 37 行)为 false 时回退到 `DANGEROUS_uncachedSystemPromptSection`——注释直接点名「rebuilt every turn; cache-busts on late connect」。delta 路径正是为了消灭这个 cache-bust 而生。

这和第 13 章「age 字符串预计算防 `Date.now()` 漂移 cache bust」、第 12 章「fork 不设 maxOutputTokens 防 thinking 配置不一致 cache miss」是同一套偏执的第三次复现:**任何会进 API 请求前缀的内容,只要它的字节可能随时间/连接状态变动,就必须想办法让它「一次确定、之后不变」,否则就是在持续烧 cache_creation**。

## 15.10 MCP skill:server 也能贡献 skill

MCP 不止贡献工具,还能贡献 skill。`fetchMcpSkillsForClient`(`mcpSkills.ts` 第 30 行)在 server 暴露 `skill://` 前缀的 resource 时,把每个 resource 读出来、解析 frontmatter、转成一个和本地 `.md` skill 一样可索引可调用的 Command。命名同样走 `mcp__<server>__<rawName>` 保证跨 server 唯一。内容先过 `recursivelySanitizeUnicode` 清洗(外部输入不信任),且如 15.6 所述,这类 skill 的 `loadedFrom: 'mcp'` 会让它在执行时**跳过 inline bash**。这个函数按 server 名 `memoizeWithLRU`(缓存 20 个),连接生命周期内重复调用走缓存。

还有一道相关的收口(`SkillTool.ts` 第 81 行 `getAllCommands`):SkillTool 只把 `loadedFrom === 'mcp'` 的 MCP **skill** 纳入可调用范围,普通 MCP prompt 被滤掉。注释说明动机:在这个过滤之前,模型若猜中 `mcp__server__prompt` 名字就能调用 MCP prompt——「they weren't discoverable but were technically reachable」,可发现性和可达性不一致是个洞,这道过滤把两者对齐。

## 本章小结

- CCB 把「外部能力引入」分两套:**Skill**(声明式 markdown + 可选 bash 注入,注入对话)与 **MCP**(外部工具服务器,接入工具池);共同命题是「不污染主上下文、不破坏缓存、不绕过权限」。
- Skill listing 只用 **1% 上下文窗口**(`SKILL_BUDGET_CONTEXT_PERCENT`,per-entry cap `MAX_LISTING_DESC_CHARS=1536`)——发现与加载分离,完整内容在 invoke 时才读;超预算时 bundled skill 永不截断。
- `getSkillDirCommands` 四来源并行加载(managed/user/project/add-dir),`/skills/` 只认 `skill-name/SKILL.md`;`realpath` 去重(防不可靠 inode,issue #13893);frontmatter 非法值优雅降级。
- 两套「运行中才出现」:`discoverSkillDirsForPaths`(向上走到 cwd、缓存已查路径、gitignore 第一道筛)与条件 skill(`paths` frontmatter,`ignore()` 匹配,与第 13 章条件规则同源)。
- `SkillTool.call` 两种执行:inline(展开进当前对话,`contextModifier` 叠加 allowedTools/model/effort,带 `[1m]` 后缀防误砍窗口)与 fork(`context:'fork'` 隔离子 agent,只收结果);都 `addInvokedSkill` 以便压缩后「重建现场」。
- `clearInvokedSkillsForAgent` 是第 14 章子 agent `finally` 埋下的清理线,按 agentId 删本 agent 的 skill;`clearInvokedSkills(preservedAgentIds)` 反向清保留集之外的——一个 map 两种粒度。
- 安全边界:MCP skill(`loadedFrom==='mcp'`)**绝不执行内联 shell**(远程不可信);`skillHasOnlySafeProperties` 是 allowlist(新属性默认要确认,fail-closed)。
- MCP 工具命名 `mcp__server__tool`(`normalizeNameForMCP` 把非法字符换下划线);`mcpInfoFromString` 诚实标注「server 名含 `__` 会解析错」的已知裂缝;权限用全限定名防误伤。
- MCP 五态状态机(connected/failed/needs-auth/pending/disabled),只有 connected+认证才有工具;`requiredMcpServers` 就绪等待(30s/500ms 轮询,任一 failed 提前退出)。
- `getMcpInstructionsDelta` 按 server NAME diff(指令握手后不可变)、产出持久化增量 attachment、无状态扫历史重建——消灭「晚连接 server 每 turn 重拼系统提示 cache-bust」,与第 12/13 章的缓存偏执同源。
- `fetchMcpSkillsForClient` 让 server 经 `skill://` resource 贡献 skill(`memoizeWithLRU`、Unicode 清洗);`getAllCommands` 只纳入 MCP skill、滤掉可达但不可发现的 MCP prompt。

## 动手实验

1. **实验一:算清 skill listing 预算** — 读 `prompt.ts`(`SKILL_BUDGET_CONTEXT_PERCENT`、`getCharBudget`、`formatCommandsWithinBudget`)。给定 200K 窗口,算出 listing 的字符预算。再构造一个「全量描述超预算」的场景:论证 bundled skill 为何永不截断、非 bundled 如何被压缩,并解释「listing 只为发现、完整内容 invoke 时才加载」如何省下第一轮 cache_creation。
2. **实验二:跟踪条件 skill 的激活** — 读 `loadSkillsDir.ts` 的 `parseSkillPaths`、`conditionalSkills`/`activatedConditionalSkillNames`、`activateConditionalSkillsForPaths`。构造一个带 `paths: ["src/api/**"]` 的 skill:它启动时进哪个 map?模型编辑 `src/api/foo.ts` 时如何被激活?为什么 `activatedConditionalSkillNames` 要「跨缓存清理存活」?这和第 13 章的条件规则有何同构?
3. **实验三:论证 MCP skill 的安全边界** — 读 `loadSkillsDir.ts` 第 371–374 行与 `mcpSkills.ts`。回答:为什么 `loadedFrom==='mcp'` 的 skill 要跳过 `executeShellCommandsInPrompt`?一个本地 skill 和一个 MCP skill 写了同样的 `!`whoami``,行为差在哪?再看 `skillHasOnlySafeProperties` 为什么用 allowlist 而非 blocklist?
4. **实验四:拆 instructions delta 的缓存账** — 读 `mcpInstructionsDelta.ts`。论证:为什么按 server NAME 而非内容 diff 是安全的?`announced` 集合如何从历史 attachment 无状态重建?`isMcpInstructionsDeltaEnabled` 为 false 时回退到的 `DANGEROUS_uncachedSystemPromptSection` 会在什么时刻 cache-bust?把它和第 13 章的 age 预计算、第 12 章的 fork 不设 maxOutputTokens 并列,提炼出共同的缓存不变量。

> **下一章预告**:Skill 和 MCP 是「把能力接进来」,但接进来之后,用户还想在**关键节点插入自己的逻辑**——提交前跑 lint、工具调用前做校验、会话结束时归档。第 16 章我们拆 CCB 的 Hooks 与 plugin 系统:hook 的事件类型与执行时机、hook 输出如何反馈进主循环(以及第 12 章为何警惕「error → hook blocking → retry」的死亡螺旋)、plugin 如何打包 skill/agent/hook/MCP 配置成一个可分发单元,看「用户自定义逻辑」如何在不破坏 harness 不变量的前提下被安全编织进运行时。
