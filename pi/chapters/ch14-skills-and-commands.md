# 第 14 章　Skill 与 Slash Command：不写代码也能扩展

> 第 13 章的扩展要写 TypeScript——那是给工程师的。但「教会 Agent 干一件特定的事」常常不需要代码,只需要一段说明书。pi 为此留了两条「零代码」的扩展通道:Skill(技能)和 Prompt Template(提示词模板)。它们都只是一个 Markdown 文件。

上一章的 Extension 系统威力巨大,但门槛也实在——你得会写代码、懂事件、理解 `ExtensionContext`。可现实里大量的「扩展需求」根本是知识性的:「碰到 React 组件就按我们团队的规范写」「部署流程是这五步」。这类需求的本质是**给模型注入一段领域知识或操作流程**,不该逼用户去写 jiti 扩展。pi 的答案是两种纯 Markdown 的轻量机制:**Skill** 和 **Prompt Template**。这一章把它们讲透,顺带厘清 pi 里「斜杠命令」到底有几种来源。

---

## 14.1 Skill：一个带 frontmatter 的 `SKILL.md`

一个 Skill 就是一个 `SKILL.md` 文件,顶部一段 YAML frontmatter 声明元信息,下面是 Markdown 正文(具体的指令/知识)。pi 解析它的结构定义在 `core/skills.ts`:

```ts
export interface SkillFrontmatter {
	name?: string;
	description?: string;
	"disable-model-invocation"?: boolean;
	[key: string]: unknown;
}
```

一个典型的 `SKILL.md` 长这样:

```markdown
---
name: pdf-processing
description: Extract text and tables from PDF files. Use when the user
  asks to parse, read, or analyze a PDF document.
---

# PDF 处理

1. 用 `pdftotext` 提取纯文本……
2. 表格用 `camelot` ……
（这里是给模型看的详细操作说明）
```

pi 对 frontmatter 的两个字段有**严格的校验**(对齐 Agent Skills 规范):

- **`name`**:只能 `a-z0-9-`、不超过 64 字符、不能以连字符开头/结尾、不能有连续连字符(`validateName`)。缺省时回退用父目录名。
- **`description`**:**必填**、不超过 1024 字符(`validateDescription`)。缺了它,这个 Skill 直接不加载——`if (!frontmatter.description...) return { skill: null }`。

为什么 `description` 这么关键、宁可不加载也不能缺?因为它是**模型决定「要不要用这个 Skill」的唯一依据**。下一节就会看到,description 是唯一进系统提示词的内容——写不好,模型就永远不会在对的时机想起这个技能。

## 14.2 渐进式披露：description 进提示词，正文按需才读

这是 Skill 机制最精妙的设计,也是它和「把知识一股脑塞进系统提示词」的根本区别。看 `formatSkillsForPrompt` 如何把 Skill 注入系统提示词:

```ts
const lines = [
	"The following skills provide specialized instructions for specific tasks.",
	"Use the read tool to load a skill's file when the task matches its description.",
	"<available_skills>",
];
for (const skill of visibleSkills) {
	lines.push("  <skill>");
	lines.push(`    <name>${escapeXml(skill.name)}</name>`);
	lines.push(`    <description>${escapeXml(skill.description)}</description>`);
	lines.push(`    <location>${escapeXml(skill.filePath)}</location>`);  // ← 只给路径,不给正文
	lines.push("  </skill>");
}
```

注意:**进系统提示词的只有 `name` + `description` + `location`(文件路径),Skill 的正文一个字都没进去。** 提示词里那句话点破了机制:「**Use the read tool to load a skill's file when the task matches its description.**」——当任务匹配某个 description 时,模型自己用 `read` 工具去读那个 `location` 路径,把正文加载进来。

这就是 **渐进式披露(progressive disclosure)**:

1. **平时**:每个 Skill 只花掉「一行 description」的上下文预算。哪怕你装了 50 个 Skill,系统提示词也只多了 50 行简介。
2. **需要时**:模型判断「这个任务匹配 pdf-processing 的描述」,才主动 `read` 那个文件,把完整指令拉进上下文。

> **给 Agent 开发者的启示**:Skill 机制的本质,是把「知识的索引」和「知识的正文」分离——索引(description)常驻、廉价、用于触发;正文(SKILL.md body)按需加载、昂贵、用于执行。这完美复用了 pi 已有的 `read` 工具,**没有为「加载技能」发明任何新机制**。模型本来就会读文件,Skill 不过是给了它一张「该读哪个文件」的菜单。这种「用已有原语拼出新能力」的克制,是优雅扩展系统的标志。

`disable-model-invocation: true` 是这套机制的一个开关:置真的 Skill 会被 `formatSkillsForPrompt` 过滤掉(`filter((s) => !s.disableModelInvocation)`),即不进菜单、模型无从自动触发——它**只能被用户显式调用**(下一节的 `/skill:name`)。适合那些「我想手动点名才用、不希望模型自作主张」的技能。

## 14.3 Skill 的发现规则与冲突处理

`loadSkills` 从三处加载 Skill:**全局 `~/.pi/skills/`**、**项目 `./.pi/skills/`**、以及**显式配置的路径**。目录扫描规则(`loadSkillsFromDirInternal`)有讲究:

- 一个目录里有 `SKILL.md`,就**把这个目录当作一个技能根,不再往下递归**——`SKILL.md` 同目录的其他文件被视为这个技能的资源(正文里可以相对引用)。
- 没有 `SKILL.md` 的目录,则加载其下直接的 `.md` 文件各算一个技能,并继续递归子目录找 `SKILL.md`。
- 尊重 `.gitignore`/`.ignore`/`.fdignore`,跳过 `node_modules` 和点开头的目录。

加载时还做两件「卫生」工作:

1. **符号链接去重**:用 `canonicalizePath`(realpath)算真实路径,同一个文件经由不同符号链接被发现多次时,只装一次。这和第 9 章 `withFileMutationQueue` 用 realpath 做 key 是同一种「逻辑同一性」思维。
2. **同名冲突检测**:两个 Skill 重名时,**先加载的赢**,后来的产生一条 `collision` 诊断(不静默吞掉,让用户知道有冲突)。

注意 Skill 正文里若引用相对路径,要相对 `SKILL.md` 所在目录(`baseDir`)解析——`formatSkillsForPrompt` 的提示词专门叮嘱了模型这一点。这让一个 Skill 可以是「一个目录」:`SKILL.md` + 配套脚本/模板/数据,自成一个可移植的能力包。

## 14.4 三种斜杠命令：extension / prompt / skill

用户在输入框敲 `/xxx`,在 pi 里可能命中**三种完全不同来源**的东西。`slash-commands.ts` 把来源类型枚举得很清楚:

```ts
export type SlashCommandSource = "extension" | "prompt" | "skill";
```

再加上一组**内置命令**(`BUILTIN_SLASH_COMMANDS`:`/model`、`/compact`、`/fork`、`/tree`、`/resume`、`/new`、`/quit` ……),一共四类:

| 斜杠命令 | 来源 | 本质 | 例 |
|---|---|---|---|
| 内置 | 核心硬编码 | 直接操作会话/UI | `/compact`(连第 12 章)、`/tree`(连第 11 章) |
| 扩展命令 | `registerCommand`(第 13 章) | 跑一段扩展代码 | 任意自定义 |
| Prompt 模板 | `~/.pi/prompts/*.md` | 展开成一段文本喂给模型 | `/review-pr` |
| Skill 调用 | `/skill:name` | 注入某个 Skill 的正文 | `/skill:pdf-processing` |

后两类是「零代码」的,正是本章的主角。它们都不执行代码,只是**把用户的一行 `/命令` 展开成一段要发给模型的文本**。

## 14.5 Prompt Template：带参数替换的文本片段

Prompt Template(`core/prompt-templates.ts`)是最简单的一种:`~/.pi/prompts/` 或 `./.pi/prompts/` 下的 `.md` 文件,文件名即命令名。用户敲 `/review-pr 123`,pi 就把 `review-pr.md` 的正文展开、替换掉里面的参数占位符,当作用户消息发出去。

它的参数替换(`substituteArgs`)直接学 bash:

```ts
// $1, $2 ...        位置参数
// $@ / $ARGUMENTS   全部参数
// ${1:-default}     第1个参数,缺省时用 default
// ${@:2}            从第2个起的所有参数
// ${@:2:3}          从第2个起的3个参数
```

于是一个 `deploy.md` 模板可以写成:

```markdown
---
description: 部署到指定环境
argument-hint: <env> [version]
---
请把服务部署到 ${1:-staging} 环境,版本 ${2:-latest}。步骤:
1. ……
```

`expandPromptTemplate` 在用户输入以 `/` 开头时匹配模板名、解析参数、做替换,**返回展开后的文本**——它不发给模型、不执行,只负责「把 `/deploy prod v2` 变成一段完整的指令文本」。Prompt 模板因此是「可复用的提问」:把你反复输入的长 prompt 存成文件,用一行斜杠命令唤起。

## 14.6 Skill 调用：`/skill:name` 如何展开成 skill block

Skill 除了「让模型自动 read」,也能被用户**显式调用**:`/skill:pdf-processing 把这个文件转成文本`。展开逻辑在 `agent-session.ts` 的 `_expandSkillCommand`:

```ts
private _expandSkillCommand(text: string): string {
	if (!text.startsWith("/skill:")) return text;
	const skillName = ...; const args = ...;
	const skill = this.resourceLoader.getSkills().skills.find((s) => s.name === skillName);
	if (!skill) return text;
	const body = stripFrontmatter(readFileSync(skill.filePath, "utf-8")).trim();
	const skillBlock = `<skill name="${skill.name}" location="${skill.filePath}">\n` +
		`References are relative to ${skill.baseDir}.\n\n${body}\n</skill>`;
	return args ? `${skillBlock}\n\n${args}` : skillBlock;
}
```

它把 Skill 正文包进一个 `<skill name=... location=...>...</skill>` 块,再把用户附带的话(args)接在后面,作为一条用户消息发出。和「自动触发」相比,显式调用是**强制注入**——不依赖模型判断,用户点名了就一定把正文塞进上下文。

这个 `<skill>` 块还有 UI 上的呼应。`parseSkillBlock` 用正则把它解析回 `{name, location, content, userMessage}`,`SkillInvocationMessageComponent`(第 15 章的 TUI 组件)据此把它渲染成一个可折叠的 `[skill] 名字` 行——默认折叠(不让大段技能正文刷屏),按键展开看全文。**注入给模型的是全文,展示给用户的是一行**——这又是第 11 章「树里存全集、喂模型子集」那种「内容与呈现分层」思维在 UI 层的再现。

至此,pi 的「扩展光谱」就完整了:重量级的代码扩展(第 13 章 Extension)、轻量级的知识扩展(本章 Skill)、最轻的文本扩展(本章 Prompt Template)。三者覆盖了从「我要改 Agent 行为」到「我只想存一段常用 prompt」的全部需求,且**没有一个需要改 pi 核心**。下一章我们转向另一个 pi 不肯将就的地方——它为什么放着成熟的终端 UI 库不用,要自研一套 `pi-tui`。

---

## 本章小结

- Skill = 一个带 YAML frontmatter 的 `SKILL.md`;`name`(`a-z0-9-`、≤64)、`description`(必填、≤1024)被严格校验,缺 description 直接不加载——因为 description 是模型决定是否启用该技能的唯一依据。
- 核心设计是**渐进式披露**:`formatSkillsForPrompt` 只把 name+description+location 注入 `<available_skills>`,正文不进;提示词指示模型「任务匹配 description 时用 `read` 工具加载 location」。50 个技能也只花 50 行预算,正文按需加载——复用已有 `read` 工具,不发明新机制。
- Skill 从全局/项目/显式路径三处发现:含 `SKILL.md` 的目录即一个技能根(不再递归);用 realpath 去重、同名冲突「先到先得」并产出 collision 诊断;`disable-model-invocation` 让技能退出自动菜单、只能显式调用。
- 斜杠命令有四类来源:内置(`/compact`/`/tree`...)、扩展命令(`registerCommand`)、Prompt 模板、`/skill:name`。后两类零代码,只把 `/命令` 展开成发给模型的文本。
- Prompt Template 是 `.md` 文本片段 + bash 风格参数替换(`$1`/`$@`/`${1:-default}`/`${@:2:3}`),`expandPromptTemplate` 展开后当用户消息发出——「可复用的提问」。
- `/skill:name` 经 `_expandSkillCommand` 把正文包成 `<skill>` 块强制注入(不依赖模型判断);UI 用 `parseSkillBlock` + `SkillInvocationMessageComponent` 渲染成可折叠的一行——注入全文、展示一行。

下一章,看 pi 为什么要自研终端 UI 库 `pi-tui`。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/coding-agent/src/core/skills.ts` | `SkillFrontmatter`/`Skill`、`validateName`/`validateDescription`、`loadSkills`(发现/去重/冲突)、`formatSkillsForPrompt`(渐进式披露) |
| `packages/coding-agent/src/core/prompt-templates.ts` | `loadPromptTemplates`、`substituteArgs`(bash 风格参数)、`expandPromptTemplate` |
| `packages/coding-agent/src/core/slash-commands.ts` | `SlashCommandSource`(extension/prompt/skill)、`BUILTIN_SLASH_COMMANDS` |
| `packages/coding-agent/src/core/agent-session.ts` | `parseSkillBlock`、`_expandSkillCommand`(`/skill:name` → `<skill>` 块) |
| `packages/coding-agent/src/modes/interactive/components/skill-invocation-message.ts` | `SkillInvocationMessageComponent`:可折叠渲染 skill 块 |

## 动手实验

1. 写一个最小 Skill:在 `./.pi/skills/hello/SKILL.md` 里放 frontmatter(`name: hello`、一句 `description`)+ 一段正文。启动 pi,确认系统提示词的 `<available_skills>` 里**只有** description 而没有正文。再问一个匹配 description 的问题,观察模型是否主动 `read` 了 `SKILL.md`——这就是渐进式披露。
2. 故意把那个 Skill 的 `description` 删掉,重启,验证它**不再被加载**(对照 `loadSkillFromFile` 里 `return { skill: null }` 的分支)。再把 `name` 改成 `Hello_World`(含大写和下划线),看 `validateName` 产生了什么诊断。
3. 写一个 Prompt 模板 `./.pi/prompts/commit.md`,正文用 `${1:-WIP}` 接收一个可选的提交信息。分别敲 `/commit` 和 `/commit "fix bug"`,对照 `substituteArgs` 解释两次展开出的文本差异。再对比:`/skill:name` 和 `/prompt` 展开后的去向有何不同(一个包成 `<skill>` 块、一个直接当文本)?
