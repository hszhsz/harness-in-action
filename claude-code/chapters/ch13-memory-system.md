# 第 13 章　记忆系统：分层 CLAUDE.md 与按需召回的 memdir

> 前面四章（第 10–12 章）讲的都是「**怎么把上下文减下来**」——预算、snip、microcompact、autocompact。但上下文工程还有另一半：**怎么把正确的记忆主动加进去**。CCB 的记忆分两套互补的系统：一套是声明式的、每个 turn 都全量注入的 `CLAUDE.md` 分层指令；另一套是文件式的、按查询相关性临时召回的 `memdir`（auto memory）。前者解决「项目和用户的硬规则必须始终在场」，后者解决「历史积累的知识不能全塞进上下文、得按需挑」。这两套系统在「注入什么、什么时候注入、注入多少」上做的每一个决定，都是上下文预算和召回精度之间的权衡。

## 13.1 CLAUDE.md 的四层加载顺序：优先级倒序排列

`claudemd.ts` 文件开头的注释把整套加载顺序钉得很死（第 1–10 行）：

```
1. Managed memory (/etc/claude-code/CLAUDE.md)   - 所有用户的全局指令
2. User memory (~/.claude/CLAUDE.md)             - 跨项目的私有全局指令
3. Project memory (CLAUDE.md, .claude/CLAUDE.md, .claude/rules/*.md) - 签入代码库
4. Local memory (CLAUDE.local.md)                - 项目私有指令
```

关键设计在第 9 行：「Files are loaded in **reverse order of priority**, i.e. the latest files are highest priority」——文件按**优先级倒序**加载，越靠后加载的优先级越高。Managed（机构强制）排最前但优先级最低，Local（用户当下项目的私有覆盖）排最后但优先级最高。这背后是对 LLM 注意力的一个经验假设：模型「pays more attention to」靠后的内容，所以把最该被遵守的指令放在最末尾。

项目级记忆的发现方式是**从当前目录向根目录逐级上溯**（第 14–16 行），每一级目录都检查 `CLAUDE.md`、`.claude/CLAUDE.md`、以及 `.claude/rules/` 下的所有 `.md`。越靠近当前目录的文件优先级越高（加载越晚）。这套规则的内部一致性在于：无论是「四层之间」还是「同一层的不同目录之间」，排序逻辑都是同一条——**越具体、越贴近当下，越靠后、越高优**。

记忆字符串的最终拼装在 `getClaudeMds`（第 1152 行），它给每一份记忆加上类型描述后拼成一段，最前面冠以 `MEMORY_INSTRUCTION_PROMPT`（第 88 行）：

```ts
const MEMORY_INSTRUCTION_PROMPT =
  'Codebase and user instructions are shown below. Be sure to adhere to these instructions. IMPORTANT: These instructions OVERRIDE any default behavior and you MUST follow them exactly as written.'
```

这句话把 CLAUDE.md 的语义级别抬到了「**OVERRIDE any default behavior**」——记忆不是建议，是凌驾于默认行为之上的硬约束。

## 13.2 @include：带深度上限与循环防护的文件拼接

CLAUDE.md 支持用 `@` 把别的文件拉进来（第 18–26 行注释）：`@path`、`@./relative`、`@~/home`、`@/absolute` 四种写法，其中无前缀的 `@path` 等同于 `@./path`（相对路径）。这套机制有几个精算过的边界：

- **只在叶子文本节点生效**（"Works in leaf text nodes only, not inside code blocks or code strings"）——靠 `marked` 词法器解析，代码块里的 `@foo` 不会被误当成 include 指令。这是「保结构、不误伤」的一贯偏执。
- **被包含文件排在包含它的文件之前**（"Included files are added as separate entries before the including file"）——被引用的内容先出现，引用方后出现，符合「依赖在前、主体在后」的阅读直觉。
- **深度上限 `MAX_INCLUDE_DEPTH = 5`**（第 536 行）：`processInclude` 在 `depth >= MAX_INCLUDE_DEPTH` 时直接停（第 629 行），防止深层嵌套把上下文撑爆。
- **循环引用防护**：`processedPaths` 集合记录已处理过的路径，再次遇到同一个 `normalizedPath` 就跳过（第 629 行 `processedPaths.has(normalizedPath)`），A→B→A 不会无限递归。
- **二进制文件拦截**：`TEXT_FILE_EXTENSIONS`（第 95 行起）是一个白名单——只有 `.md`/`.txt` 等文本扩展名能被 include，图片、PDF 等二进制文件被挡在外面，避免把无意义的字节灌进记忆。
- **不存在的文件静默忽略**（"Non-existent files are silently ignored"）——include 一个不存在的路径不报错，方便可选引用。

整套记忆还有一条总闸：`MAX_MEMORY_CHARACTER_COUNT = 40000`（第 91 行）。超出这个字符数的记忆文件会被 `getLargeMemoryFiles`（第 1132 行 `f.content.length > MAX_MEMORY_CHARACTER_COUNT`）挑出来给用户提示——记忆是要每个 turn 都全量进上下文的，太大就是在持续烧预算。

## 13.3 条件规则：用 frontmatter glob 把记忆绑定到文件路径

不是所有项目记忆都该「永远在场」。`.claude/rules/*.md` 支持在 frontmatter 里写 glob，让某条规则**只在当前操作的文件匹配时才注入**——这就是条件规则（conditional rules）。`parseFrontmatterPaths` 从 frontmatter 抽出 glob 列表，`processConditionedMdRules` 用 `ignore()` 把这些 glob 匹配到 `targetPath`：命中才把规则加进记忆，不命中就略过。

注释里有两个细节值得记：第一，glob 的 `/**` 后缀会被剥掉（统一成目录匹配语义）；第二，单独一个 `**` 等于「match-all」，会被当成「没有 glob 约束」处理——即无条件注入。这套机制把「记忆」从「全有或全无」细化到「按你正在碰的文件动态裁剪」，是上下文预算的又一处精细化：编辑前端文件时不必背着后端规约，反之亦然。

## 13.4 stripHtmlComments：注释要去掉，但代码里的不能动

CLAUDE.md 经常用 HTML 注释 `<!-- ... -->` 写给人看的旁注，这些不该进模型上下文。`stripHtmlComments` 负责剥掉它们，但做得很克制：

- **块级解析**：同样借 `marked` 词法器按 block 切，**保留代码块/代码 span 内的 `<!-- -->`**——因为那可能是文档里正在讲解的示例，不是真注释。
- **未闭合注释原样保留**：遇到没有结束标记的 `<!--`，不贪婪地吞掉后面所有内容，而是留在原地。这是「拿不准就不动」的 fail-safe——宁可漏删一个畸形注释，也不能把正文误删。

这又是第 11 章「保结构、删内容」那条透镜在记忆系统里的复现：删的是确定无用的（规范的注释），保的是可能有用的（代码里的字面内容）。

## 13.5 memdir：从「全量注入」到「按需召回」

CLAUDE.md 是声明式的——你写下的规则每个 turn 都全量进上下文。但还有一类知识不适合这样：用户的偏好、历史反馈、项目的来龙去脉，这些会随时间积累到几十上百条，全塞进上下文既烧预算又稀释注意力。CCB 为此造了第二套系统——**memdir（auto memory）**，核心思路是：**记忆以独立文件存在磁盘上，每个 turn 只按当前查询的相关性临时召回最多 5 条。**

memdir 的目录由 `getAutoMemPath`（`paths.ts` 第 223 行）解析，默认是 `<memoryBase>/projects/<sanitized-git-root>/memory/`。注意它用 `findCanonicalGitRoot`（第 204 行）取规范 git 根——这样同一个 repo 的所有 worktree **共享同一个 auto-memory 目录**（注释引 issue #24382），避免每个 worktree 各记一份。这个函数是 `memoize` 的，因为渲染路径上的调用会「per tool-use message per Messages re-render」高频触发，每次 miss 都要读四遍 settings 文件。

是否启用由 `isAutoMemoryEnabled`（第 30 行）按一条优先级链判定，**默认开启**：

```
1. CLAUDE_CODE_DISABLE_AUTO_MEMORY 环境变量（1/true→关，0/false→开）
2. CLAUDE_CODE_SIMPLE (--bare) → 关
3. CCR 无持久存储（没有 CLAUDE_CODE_REMOTE_MEMORY_DIR）→ 关
4. settings.json 的 autoMemoryEnabled（支持项目级 opt-out）
5. 默认开启
```

## 13.6 记忆类型与「不该存什么」：把记忆约束成封闭分类

memdir 的记忆被钉成**封闭的四类**（`memoryTypes.ts` 第 14 行）：

```ts
export const MEMORY_TYPES = ['user', 'feedback', 'project', 'reference'] as const
```

- **user**：用户的角色、目标、偏好、知识——用来定制行为
- **feedback**：用户对「怎么做事」的指导（该避免什么、该坚持什么），要求结构化成「规则 + **Why:** + **How to apply:**」
- **project**：进行中的工作、目标、bug、事故等**无法从代码或 git 历史推导**的信息（要求把相对日期转成绝对日期）
- **reference**：指向外部系统的指针（Linear、Slack、Grafana 等）

比「存什么」更关键的是 `WHAT_NOT_TO_SAVE_SECTION`（第 98 行）那张**禁存清单**：代码模式/架构/文件路径（可由读代码推导）、git 历史（`git log`/`git blame` 才权威）、调试解法（修复在代码里）、CLAUDE.md 已有的内容、临时任务细节。它的判据是一条清晰的不变量——**凡是能从当前项目状态推导出来的，都不该存成记忆**。注释里甚至说明：「These exclusions apply even when the user explicitly asks you to save」——连用户明说要存的也照样过滤，如果用户要存一份 PR 列表，应反问「哪里令人意外或不显然」，只留那部分。这道闸是 eval 验证过的（case 3，0/2→3/3），防的是「存本周 PR 列表」变成活动日志噪音。

`parseMemoryType`（第 28 行）对未知类型返回 `undefined` 而非报错——没有 `type` 字段的旧文件继续可用，未知类型优雅降级。这是「向后兼容优先」的边界处理。

## 13.7 findRelevantMemories：用一次 sideQuery 选出最相关的 5 条

召回的核心是 `findRelevantMemories`（`findRelevantMemories.ts` 第 40 行）。它不把记忆全文喂给模型挑，而是分两步：

**第一步——扫 header**。`scanMemoryFiles`（`memoryScan.ts` 第 35 行）递归读 memdir 下所有 `.md`（排除 `MEMORY.md`，它已在系统提示里），但**每个文件只读前 30 行 frontmatter**（`FRONTMATTER_MAX_LINES = 30`），取出 `description` 和 `type`，按 mtime 倒序、上限 `MAX_MEMORY_FILES = 200`。注释点出一个工程优化：`readFileInRange` 内部已经 stat 拿到 mtime，所以是「读了再排」而非「stat→排→读」，常见情况（N≤200）下省掉一轮 stat。

**第二步——让 Sonnet 选**。`selectRelevantMemories`（第 80 行）把所有 header 拼成 manifest（`[type] filename (timestamp): description` 每行一条），连同用户 query 发给一次 `sideQuery`，约束输出为 JSON schema（`selected_memories: string[]`），`max_tokens: 256`，`querySource: 'memdir_relevance'`，`optional: true`。系统提示（`SELECT_MEMORIES_SYSTEM_PROMPT`，第 19 行）反复强调「**最多 5 条**」「不确定就别选」「宁缺毋滥」。

这里有两个精细设计：

- **`recentTools` 去噪**：如果某个工具正在被频繁使用，它的 API 文档类记忆就是噪音（对话里已有实际用法）。系统提示明确「do not select memories that are usage reference or API documentation for those tools」——但**警告/坑/已知问题类记忆仍要选**，因为「active use is exactly when those matter」。这是把「相关」从「关键词重叠」细化到「此刻是否真的有用」。
- **`alreadySurfaced` 预过滤**：前几轮已经露出过的记忆，在 sideQuery **之前**就被滤掉（第 48–50 行），让选择器把 5 个名额花在新候选上，而不是反复挑会被调用方丢弃的旧文件。

失败处理很克制：`sideQuery` 抛错时，若是 abort 就返回空，否则记一条 warn 日志后返回空数组——召回失败绝不阻塞主流程，这正是 `optional: true` 的语义。

## 13.8 异步预取与去重注入：召回不能拖慢这一个 turn

召回要调一次模型，是有延迟的。CCB 不让它阻塞主 turn——`relevant_memories` 已经**从同步 attachment 路径移到异步预取**（`attachments.ts` 第 925 行注释：「relevant_memories moved to async prefetch」）。`startRelevantMemoryPrefetch`（第 2419 行）在 `query.ts`（第 454 行）里用 `using` 绑定，turn 开始时非阻塞地发起搜索，到了消费点轮询 `settledAt`（从不阻塞），`using` 在生成器所有退出路径上自动 dispose。注释还点出为什么不能 per-iteration 触发：那会「ask sideQuery the same question N times」——同一个问题问 N 遍纯属浪费。

选出的记忆在注入前还要再过两道去重（第 2285–2288 行）：`readFileState.has(m.path)`（模型已经用 FileRead 读过的就不重复注入）和 `alreadySurfaced.has(m.path)`，最后 `.slice(0, 5)` 封顶。注释自嘲这是「belt-and-suspenders」（双保险）——多目录结果可能重新引入一个在别的目录被滤掉的路径。

注入出去的 `relevant_memories` attachment 还带一个**预计算的 header**（第 503–516 行）。注释里有一处极典型的缓存偏执：age 字符串（"saved 3 days ago"）必须在 attachment 创建时算好、之后跨 turn 保持不变——因为 `memoryAge(mtimeMs)` 内部调 `Date.now()`，如果在渲染时重算，「3 天前」过一天变成「4 天前」，**渲染字节一变就 prompt cache bust**。同样的，`memoryAge`（`memoryAge.ts` 第 15 行）把时间戳渲染成「today / yesterday / N days ago」而非 ISO 时间戳，注释解释：「Models are poor at date arithmetic」——模型不擅长日期算术，「47 days ago」比一个原始时间戳更能触发它的「这条可能过时了」推理。

## 13.9 信任但要核实：记忆的「保鲜」机制

memdir 的 prompt 花了大量篇幅教模型**怎么对待召回的记忆**，因为破坏式风险不在「记不住」，而在「把过时的记忆当成事实」。三段递进的指导：

- **`MEMORY_DRIFT_CAVEAT`**（第 116 行）：记忆是「某个时间点为真」的快照，回答前要读当前文件/资源核实；若记忆与现状冲突，**信现在看到的**，并更新或删除过时记忆。
- **`TRUSTING_RECALL_SECTION`**（第 155 行，标题「Before recommending from memory」）：一条命名了具体函数/文件/flag 的记忆，只是「写下时它存在」的声明，可能已被改名/删除/从未合并。推荐前要核实——命名文件就检查文件存在、命名函数就 grep。这段的标题措辞是 eval 调出来的：「Before recommending」（决策点的动作线索）比抽象的「Trusting what you recall」效果好得多（3/3 vs 0/3，正文一字未改，只换了标题）。
- **`memoryFreshnessText`**（`memoryAge.ts` 第 33 行）：对 >1 天的记忆生成一句明文 staleness 警告（≤1 天返回空串，因为对新记忆告警是噪音）。动机是真实用户反馈：过时的 `file:line` 引用被当成事实断言——而**引用反而让过时的说法显得更权威**，所以要主动加保鲜标记。

`WHEN_TO_ACCESS_SECTION`（第 131 行）里还有一条 eval 调出来的反模式修复：用户说「忽略关于 X 的记忆」时，模型容易「acknowledge then override」（嘴上说忽略、实际还加一句「而非 Y，如记忆所述」）。这条 bullet 明确要求——**当作 MEMORY.md 为空，不引用、不比对、不提及**。

## 13.10 MEMORY.md 索引与压缩时的取舍

memdir 用 `MEMORY.md` 作为**索引**而非记忆本体（`memdir.ts` `buildMemoryLines`）：每条记忆写成独立文件，再在 `MEMORY.md` 里加一行指针 `- [Title](file.md) — one-line hook`。`MEMORY.md` 每个 turn 全量进上下文，所以有双重上限——`MAX_ENTRYPOINT_LINES = 200` 行 + `MAX_ENTRYPOINT_BYTES = 25_000` 字节（第 35–38 行）。`truncateEntrypointContent`（第 57 行）先按行截、再按字节在最后一个换行处截（不切半行），并在末尾追加一句指明是哪个上限触发的警告。字节上限专门防的是「行数没超但单行超长」的索引（注释记了 p100 实测：197KB 却在 200 行内）。

一个值得回看第 12 章的细节：autocompact 压缩后「重建现场」时，恢复最近读过的文件**会跳过 CLAUDE.md 和 plan 文件**。原因正是本章的逻辑——CLAUDE.md 是声明式记忆，下一个 turn 会**重新全量注入**，根本不需要靠「重建现场」去抢救；把名额留给那些不会自动回来的工作文件才是对的。记忆系统和压缩系统在这里对齐：**凡是每 turn 都会重新注入的，压缩就不必恢复。**

`tengu_moth_copse`（`memdir.ts` 第 422 行 `skipIndex`）这个 GB flag 进一步改变注入形态：开启时 `findRelevantMemories` 的预取通过 attachment 露出相关文件，`MEMORY.md` 索引**不再注入系统提示**——召回从「全量索引 + 按需读」转向「纯按需露出」，又省下一块固定预算。

## 本章小结

- CCB 记忆分两套互补系统：**声明式的 CLAUDE.md**（每 turn 全量注入的硬规则）与**文件式的 memdir/auto memory**（按查询相关性临时召回，最多 5 条）。
- CLAUDE.md 四层（Managed→User→Project→Local）按**优先级倒序**加载，越靠后越高优（模型更注意靠后内容）；项目记忆从当前目录上溯发现，越近越高优；`MEMORY_INSTRUCTION_PROMPT` 把记忆抬到「OVERRIDE any default behavior」。
- `@include` 带四种路径写法、深度上限 `MAX_INCLUDE_DEPTH=5`、`processedPaths` 循环防护、`TEXT_FILE_EXTENSIONS` 二进制拦截、不存在文件静默忽略；总闸 `MAX_MEMORY_CHARACTER_COUNT=40000`。
- 条件规则用 frontmatter glob（`parseFrontmatterPaths`/`processConditionedMdRules`）把规则绑定到文件路径，命中才注入；`stripHtmlComments` 删注释但保代码块内字面、未闭合注释原样留。
- memdir 默认开启（`isAutoMemoryEnabled` 优先级链），目录用 `findCanonicalGitRoot` 让 worktree 共享；记忆被钉成封闭四类（user/feedback/project/reference），禁存「可由项目状态推导」的内容（连用户明说要存也过滤）。
- `findRelevantMemories` 两步召回：`scanMemoryFiles` 只读前 30 行 frontmatter（上限 200 文件），再用一次 `sideQuery`（max 5、optional、JSON schema）让 Sonnet 选；`recentTools` 去工具文档噪音但保留警告类、`alreadySurfaced` 预过滤省名额。
- 召回走**异步预取**（`startRelevantMemoryPrefetch`，`using` 绑定、不阻塞 turn），注入前用 `readFileState`+`alreadySurfaced` 双重去重；header 的 age 字符串预计算以防 `Date.now()` 漂移导致 prompt cache bust，age 渲染成「N days ago」因模型不擅长日期算术。
- 「信任但核实」三段式：`MEMORY_DRIFT_CAVEAT`（冲突时信现状）、`TRUSTING_RECALL_SECTION`（推荐前 grep/查文件）、`memoryFreshnessText`（>1 天加保鲜警告）；标题措辞均为 eval 调优。
- `MEMORY.md` 是索引（200 行/25KB 双上限，`truncateEntrypointContent` 不切半行）；autocompact「重建现场」跳过 CLAUDE.md 正因它每 turn 自动重注入——**会自动回来的就不必恢复**；`tengu_moth_copse` 进一步把索引从系统提示移到按需露出。

## 动手实验

1. **实验一：画出 CLAUDE.md 的加载与优先级** — 读 `claudemd.ts` 开头注释（第 1–26 行）与 `getClaudeMds`（第 1152 行）。在一个三级嵌套目录里分别放 CLAUDE.md，再加一个 `~/.claude/CLAUDE.md` 和 `/etc/claude-code/CLAUDE.md`，写出最终注入顺序。重点解释：为什么「优先级倒序」加载（最高优放最后）能让模型更遵守 Local 覆盖？
2. **实验二：构造 @include 的三个边界** — 读 @include 相关逻辑（`MAX_INCLUDE_DEPTH`、`processedPaths`、`TEXT_FILE_EXTENSIONS`）。分别构造：(a) 6 层嵌套 include，(b) A→B→A 循环，(c) `@logo.png`。预测每种情况的行为，并解释为什么 include 一个不存在的文件要「静默忽略」而非报错。
3. **实验三：跟踪一次相关性召回** — 读 `findRelevantMemories`、`scanMemoryFiles`、`selectRelevantMemories`。论证三个问题：为什么只读前 30 行 frontmatter 而非全文？`recentTools` 为什么要「过滤工具文档但保留警告类记忆」？`alreadySurfaced` 在 sideQuery 之前过滤和注入时再过滤一次，哪个是主、哪个是双保险？
4. **实验四：定位「会自动回来的就不必恢复」** — 结合第 12 章的「重建现场」（跳过 CLAUDE.md）和本章 13.10。论证：为什么 CLAUDE.md 在 autocompact 后不需要恢复，而最近读过的工作文件需要？再看 `tengu_moth_copse`（`memdir.ts` 第 422 行 `skipIndex`），说明它如何把 `MEMORY.md` 从「系统提示固定注入」改为「按需露出」，省下的是哪一块预算？

> **下一章预告**：记忆是「给主 agent 注入正确的上下文」，而当任务太大、太杂时，CCB 会把它**委派给子 agent**。第 14 章我们拆 `AgentTool`——子 agent 如何拿到一个**裁剪过的工具子集**（而非父会话的全集）、它的上下文如何与主会话隔离、结果如何收束回主循环，以及「工具子集分层」背后那条「最小授权」的安全逻辑。
