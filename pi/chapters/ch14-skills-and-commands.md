# 第 14 章　Skill、Slash Command 与提示词模板

> 不是所有能力都需要写代码。有些能力，一段写得足够好的 Markdown 就够了。

第 13 章的扩展是「可执行代码」。但很多工作流的本质是**知识与步骤**，不是新工具：怎么审一个 PR、怎么迁移一版数据库、怎么按团队规范写 release note。为这些事写一个 TypeScript 工具，既重又不灵活。

pi 为此提供了两个「纯文本能力」通道：

- **Skill**：按需加载的完整操作手册，可以带脚本、参考资料、模板；
- **Prompt Template**：把一段常用提示词变成带参数的 `/` 命令。

再加上内置与扩展注册的命令，它们共同组成 pi 的 Slash Command 体系。

---

## 14.1 三选一：扩展、Skill、模板

先给一张决策表，避免「用错容器」：

| 需求 | 选择 | 原因 |
|---|---|---|
| 需要执行代码、注册工具/命令、监听事件 | Extension | 唯一能进入进程执行任意逻辑的通道 |
| 需要一套较长的流程说明，可能附带脚本与资料 | Skill | 渐进式披露，不占常驻上下文 |
| 只需要复用一段提示词，可能带参数 | Prompt Template | 最轻，展开后就是普通输入 |
| 需要用户显式触发、不希望模型自动选用 | Skill + `disable-model-invocation` 或模板 | 两者都能变成显式命令 |

一个常见误区是把 Skill 当成「更长的系统提示词」。不是。系统提示词每轮都在上下文里；Skill 只有名字和描述常驻，正文要等模型主动读取（或被 `/skill:name` 强制加载）。

---

## 14.2 Skill：一个目录 + 一份 `SKILL.md`

### 最小结构

```text
pdf-tools/
├── SKILL.md
├── scripts/
│   └── extract.sh
├── references/
│   └── formats.md
└── assets/
    └── template.json
```

`SKILL.md` 以 frontmatter 开头：

```markdown
---
name: pdf-tools
description: Extract text and tables from PDF files. Use when reading, converting, or inspecting PDFs.
---

# PDF tools

Read `references/formats.md` before converting a document. Run scripts relative to this skill directory.
```

`description` 不是给人看的摘要，而是**路由信号**：模型靠它判断「当前任务要不要加载这个 Skill」。所以它必须同时回答「做什么」和「什么时候用」。官方文档点名了反例：`Helps with PDFs` 提供不了任何路由信息。

### frontmatter 契约

pi 实现的是 [Agent Skills 规范](https://agentskills.io/specification)，支持字段：

| 字段 | 作用 |
|---|---|
| `name` | 命令名与显示名 |
| `description` | 给模型的路由描述 |
| `license` | 许可证名称或文件 |
| `compatibility` | 环境要求 |
| `metadata` | 额外键值 |
| `allowed-tools` | 实验性的预批准工具列表 |
| `disable-model-invocation` | 从模型自动选择中隐藏 |

几条硬校验（`skills.ts`）：

- `name` 只能是小写字母、数字、连字符，不能以连字符开头/结尾或连续出现，最长 **64** 字符；
- `description` 必填，最长 **1024** 字符；
- `disable-model-invocation` 采用严格布尔判断：只有 `=== true` 才生效，写 `"true"` 字符串没用；
- 格式错误或缺少 description 的 `SKILL.md` **不会加载**；
- 重名时**先发现的胜出**，并产生一条警告。

pi 不强制 `name` 与父目录名一致，但其他 Agent Skills 实现可能会，所以「保持一致」是可移植性的最佳实践。

### 发现顺序

`loadSkills()` 的默认来源顺序是：

1. `~/.pi/agent/skills/`
2. `<cwd>/.pi/skills/`
3. `~/.agents/skills/`
4. 从 cwd 向上直到仓库根的 `.agents/skills/`

再加上 settings 里显式配置的 `skillPaths`、Pi Package 里的 skills、以及命令行指定的路径。目录是递归扫描的，任何包含 `SKILL.md` 的子目录都会被识别。

最终以 **realpath 去重 + 先到先得** 合并。这意味着：用户级 Skill 可以覆盖同名项目级 Skill——这个优先级是刻意的，让个人习惯不被仓库绑架；但也意味着排查「为什么我改的 Skill 没生效」时，要先检查是不是被更高优先级的同名项挡住了。

### 渐进式披露是怎么实现的

启动时，`formatSkillsForPrompt()` 只把三样东西写进系统提示词：

```xml
<available_skills>
  <skill>
    <name>pdf-tools</name>
    <description>Extract text and tables from PDF files. ...</description>
    <location>/home/me/.pi/agent/skills/pdf-tools/SKILL.md</location>
  </skill>
</available_skills>
```

外加两条使用说明：

- 任务匹配描述时，用 `read`（或 `bash`）读取该文件；
- Skill 内部的相对路径，要相对**Skill 目录**解析，并在工具命令里使用绝对路径。

正文、脚本、参考资料都不进上下文。这就是「渐进式披露」：常驻成本只有几十个 token，完整内容按需付费。

注意 `formatSkillsForPrompt` 的第三个参数 `fileReadTool`：如果当前没有 `read` 工具但注册了 `bash`，提示会改成「用 bash 读取」；如果文件读取工具只能通过其他工具间接到达，则提示不点名任何工具。**提示词会跟着工具集合变形**，而不是假设某个工具一定存在。

### 强制加载：`/skill:name`

模型可能漏掉一个本该用的 Skill。此时可以显式触发：

```text
/skill:pdf-tools extract report.pdf
```

`AgentSession._expandSkillCommand()` 做的事很简单（[agent-session.ts:2177](file:///Users/heshaozhong/Code/pi/packages/coding-agent/src/core/agent-session.ts#L2177)）：

```ts
const content = readFileSync(skill.filePath, "utf-8");
const body = stripFrontmatter(content).trim();
const skillBlock = `<skill name="${skill.name}" location="${skill.filePath}">\nReferences are relative to ${skill.baseDir}.\n\n${body}\n</skill>`;
return args ? `${skillBlock}\n\n${args}` : skillBlock;
```

三个细节值得注意：

1. **去掉 frontmatter**，只把正文交给模型；
2. **显式告诉模型 baseDir**，否则它会把 `references/formats.md` 解析到 cwd；
3. **把参数附在 Skill 后面**，而不是替换正文——参数是「这次的具体要求」，正文是「通用的做法」。

`disable-model-invocation: true` 的 Skill 不会出现在 `<available_skills>` 里，但 `/skill:name` 仍然可用。这正是「危险或高成本操作只允许人触发」的实现方式。

是否把 Skill 注册成命令，还受 `enableSkillCommands` 设置控制；但即使关闭命令补全，手输 `/skill:name` 依然有效。

---

## 14.3 Prompt Template：把提示词变成命令

### 一个文件就是一个命令

`~/.pi/agent/prompts/review.md`：

```markdown
---
description: Review staged git changes
argument-hint: "[focus]"
---
Review the staged changes. Focus on ${1:-correctness, security, and error handling}.
```

文件名决定命令名：这个文件就是 `/review`。`description` 出现在命令补全里；省略时取第一行非空文本。`argument-hint` 可选，用 `<尖括号>` 表示必填、`[方括号]` 表示可选。

### 参数替换

模板支持一套 bash 风格的语法：

| 语法 | 含义 |
|---|---|
| `$1`, `$2`, … | 第 N 个位置参数 |
| `$@` / `$ARGUMENTS` | 所有参数，空格连接 |
| `${1:-default}` | 第一个参数，缺省用默认值 |
| `${@:-default}` | 所有参数，为空时用默认值 |
| `${@:N}` | 从第 N 个开始的所有参数 |
| `${@:N:L}` | 从第 N 个开始的 L 个参数 |

参数遵循 shell 风格引号：`/review "API compatibility"` 会得到一个含空格的参数，而不是两个。

实现上有一个重要约束：**替换不会递归**。如果参数内容本身包含 `$1` 或 `$@`，它不会被再次展开。这避免了「用户输入被当成模板执行」的注入问题。

### 展开发生在什么时候

一次用户输入的完整处理顺序（`AgentSession`）：

```text
input 事件（扩展可改写/吞掉）
  → 如果是扩展命令，直接交给扩展，不进入模型
    → /skill:name 展开
      → prompt template 展开
        → 进入 steering / followUp / 正常 prompt 队列
```

注意扩展命令优先于模板：同名时，扩展的 `input` 处理器或注册命令会先拿到控制权。这也是「扩展是更强的通道」的体现。

项目级模板放在 `.pi/prompts/`，同样要等项目信任通过才加载。

---

## 14.4 Slash Command 的三个来源

`SlashCommandInfo.source` 只有三个值：

```ts
export type SlashCommandSource = "extension" | "prompt" | "skill";
```

| 来源 | 命名 | 执行方式 |
|---|---|---|
| 内置 | `/model`、`/tree`、`/compact`、`/reload` 等 | 由 CLI / TUI 直接处理 |
| 扩展 | `pi.registerCommand(name, ...)` | 交给扩展 handler，不进入模型 |
| 提示词模板 | 文件名，如 `/review` | 展开成文本，进入模型 |
| Skill | `/skill:<name>` | 读取 `SKILL.md`，包成 `<skill>` 块进入模型 |

内置命令列表定义在 `slash-commands.ts`，包含会话管理（`new`/`resume`/`fork`/`tree`）、配置（`model`/`thinking`/`settings`）、维护（`compact`/`reload`/`export`）等。它们不是「特殊提示词」，而是真正的控制面操作。

一个值得注意的边界：**扩展命令不能被排队成 steering/follow-up**。`_queueUserInput` 里有一句 `_throwIfExtensionCommand(text)`——因为扩展命令是同步控制操作，把它塞进消息队列没有语义。Skill 与模板则可以，因为展开后就是普通用户文本。

---

## 14.5 系统提示词：分区、增量与强制替换

Skill 能出现在提示词里，靠的是系统提示词的**分区结构**。

### 分区

`buildSystemPromptSections()` 产出一个有序的 `Record<string, string>`：

| 分区 | 内容 |
|---|---|
| `preamble` | 身份与总体职责（唯一不带 XML 标签的分区） |
| `tools` | 当前声明给模型的工具及一句话说明 |
| `rules` | 工具贡献的规则 + 全局规则 |
| `docs` | pi 自身文档的位置索引 |
| `addendum` | 用户 `APPEND_SYSTEM.md` 追加的内容 |
| `project_context` | `AGENTS.md` 等项目指令 |
| `skills` | `<available_skills>` 清单 |
| `cwd` | 工作目录 |
| 自定义 | 扩展通过 `sections` 注入的任意分区 |

除 `preamble` 外，每个分区都会被包成同名 XML 标签。分区名必须匹配 `^[a-z][a-z0-9_-]*$`，且不能叫 `preamble`。

这些分区最终成为转录里 `SystemMessage.sections` 的字段——这正是第 3 章「转录原生」的落点：提示词不是一个拼好的大字符串，而是**可独立替换的块**。

### 增量更新

`diffSystemPromptSections(previous, current)` 比较模型当前持有的分区与期望分区，只返回变化的键；被删除的分区值为 `null`。

于是「加了一个 Skill」「切了一个工具」「cwd 变了」都只需要在转录里追加一个小的 `SystemMessage` 补丁，而不是重发整个系统提示词。这既省 token，也让 prompt caching 更有效。

### 强制替换

扩展在 `before_agent_start` 里可以返回 `systemPrompt` 或设置 `forceSystemPrompt`，把整个提示词换掉。此时：

- `SystemMessage.content` 存放完整文本，`sections` 为空；
- 转录仍然记录结构化的 sections（用于后续增量），但 provider 收到的是强制文本作为开头系统提示。

这是一个「逃生舱」：能力强，但会失去分区带来的增量与可观测性，非必要不用。

---

## 14.6 实践建议

1. **先问「这是知识还是代码」**。需要执行、需要状态、需要拦截事件 → 扩展；只是一套做法 → Skill；只是一段话 → 模板。
2. **Skill 的 description 当路由规则写**。写清楚触发条件、输入形态、产出物；不要写成营销文案。
3. **Skill 正文保持短，重内容放 `references/`**。模型是「按需读文件」，不是「一次性背完整本书」。
4. **危险流程用 `disable-model-invocation: true`**。例如「发布到生产」「轮换密钥」，只允许人显式 `/skill:...`。
5. **模板不要承载长流程**。超过一屏的步骤说明应该升级成 Skill，否则每次调用都重复占用上下文。
6. **项目级资源永远假设不可信**。Skill 可以指示模型执行脚本，模板可以诱导模型泄露信息；项目信任只是加载闸，不是内容审查。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/coding-agent/src/core/skills.ts` | frontmatter 校验、发现与去重、`formatSkillsForPrompt` |
| `packages/coding-agent/src/core/prompt-templates.ts` | 模板解析、bash 风格参数替换 |
| `packages/coding-agent/src/core/slash-commands.ts` | 内置命令表、`SlashCommandSource` |
| `packages/coding-agent/src/core/system-prompt.ts` | 分区构建、`diffSystemPromptSections`、强制提示词 |
| `packages/coding-agent/src/core/agent-session.ts` | `/skill:name` 展开、模板展开、输入处理顺序 |
| `packages/coding-agent/docs/skills.md` | 官方 Skill 指南 |
| `packages/coding-agent/docs/prompt-templates.md` | 官方模板指南 |

## 动手实验

1. 写一个带 `references/` 与脚本的 Skill，故意在正文里用相对路径引用脚本。分别用「模型自动加载」和 `/skill:name` 触发，观察模型是否正确解析 baseDir。
2. 写一个模板，测试 `$1`、`${1:-default}`、`${@:2:2}` 与带引号参数，确认替换结果；再在参数里塞一个 `$1`，验证不会递归展开。
3. 用 `disable-model-invocation: true` 隐藏一个 Skill，确认它不出现在 `<available_skills>` 中，但 `/skill:name` 仍能加载；然后打开 `enableSkillCommands: false`，观察命令补全与手动输入的区别。
