# 第 15 章　扩展体系：Skill、Persona 与子 Agent

## 15.1　三层扩展机制

OpenWorker 有三种扩展 Agent 能力的机制，从轻到重：

1. **Skill**：Markdown 文件，渐进式披露的操作指南
2. **Persona**：可安装的 Agent 模板，定义 Agent 的身份、工具集和连接器
3. **Subagent**：只读子 Agent，由父 Agent 委派探索任务

## 15.2　Skill 系统

### Skill 的形式

Skill 是 Markdown 文件，存放在两个目录：
- `<state_dir>/skills/`：全局 skill（`~/.config/coworker/skills/`）
- `<workspace>/.coworker/skills/`：项目级 skill

每个 skill 是一个目录，包含 `SKILL.md` 和可选的辅助文件。

### SkillLoader

```python
# skills/base.py
class SkillLoader:
    def __init__(self, dirs: list[Path]):
        self.dirs = dirs

    def load_all(self) -> list[SkillManifest]:
        # 扫描所有目录，解析 frontmatter
        ...
```

### 渐进式披露

Skill 的设计是**渐进式披露**（progressive disclosure）：

1. **Catalog**：所有 skill 的名称和一句话描述被注入 system prompt
2. **Load**：Agent 可以用 `load_skill(name)` 工具读取某个 skill 的完整内容
3. **Execute**：skill 内容作为上下文指导 Agent 执行

这避免了把所有 skill 的完整内容都塞进 system prompt（浪费 token），而是只在需要时加载。

```python
# agent.py
catalog = skill_catalog_text(skill_loader)
if catalog:
    instructions = f"{instructions}\n\n{catalog}"
```

### Skill 工具

```python
# skills/base.py
def skill_tools(loader: SkillLoader):
    return [
        load_skill_tool,      # 读取一个 skill 的完整内容
        list_skills_tool,     # 列出所有可用 skill
    ]
```

## 15.3　Persona 系统

Persona 是 OpenWorker 独有的扩展机制。如果说 Skill 是"教 Agent 做事的方法"，Persona 是"定义 Agent 是谁"。

### Persona Manifest

`coworker/personas/manifest.py` 定义了 persona 的清单格式。一个 persona 包含：

- **identity**：名称、描述、图标
- **agent**：使用的 Agent 类型（code/chat/cowork/myhelper）
- **connections**：默认启用的连接器
- **instructions**：注入到 system prompt 的额外指令
- **tools**：额外注册的工具

### 安装来源

Persona 可以从三个来源安装：

```python
# server/app.py: install_persona
if body.get("git_url"):
    summaries = reg.install_from_git(str(body["git_url"]))
elif body.get("dir"):
    summaries = reg.install_from_dir(str(body["dir"]))
elif body.get("gallery_slug"):
    # 从 OpenWorker Cloud gallery 下载
    manifest = cloud.gallery_manifest(...)
    # 验证 hash
    digest = "sha256:" + hashlib.sha256(markdown.encode()).hexdigest()
    if manifest.get("manifest_hash") != digest:
        return {"ok": False, "error": "manifest hash mismatch"}
```

### 同意流程

Persona 安装后是 **disabled** 状态，需要用户显式批准：

```python
# server/app.py: install_persona 返回
{"ok": True, "consent": summaries, "personas": reg.list_all()}
```

每个 persona 的 consent summary 列出了它要做什么、需要什么权限、会注册什么工具。用户在 UI 中审阅后决定是否启用。

### 安全约束

- **No self-grant**：Persona 不能在自己的 instructions 中给自己授权
- **Lands disabled**：安装后不自动启用，需要用户显式 action
- **Hash 验证**：gallery 安装时验证 manifest hash，防止传输篡改

## 15.4　Subagent：只读探索

`coworker/tools/subagent.py` 提供了 `explore` 工具，让 code-family 的 Agent 能 fan out 只读探索任务：

```python
# agent.py
if agent.family == "code" and ws is not None:
    registry.register_all(
        explorer_tools(
            workspace=ws,
            provider=provider,
            model=model,
            model_settings=model_settings,
        )
    )
```

### 子 Agent 的特点

1. **只读**：子 Agent 没有 write/shell 权限，只能读文件和搜索
2. **独立 context**：子 Agent 有自己的消息历史，不污染父 Agent 的上下文
3. **共享 provider**：子 Agent 用父 Agent 的 provider 和 model
4. **结果汇总**：子 Agent 的最终回答作为工具结果返回给父 Agent

### 设计哲学

OpenWorker 没有像 OpenClaw 那样做完整的 ACP（Agent Client Protocol）和子 Agent 委派系统。它的子 Agent 更简单--只是一个只读的探索工具，不是一个完整的 Agent 实例。这反映了 OpenWorker 的"务实"工程性格：**用最简单的方式解决问题，不提前优化**。

## 15.5　Agent 类型

`coworker/agents/` 目录定义了四种内置 Agent：

| Agent | family | 用途 |
|---|---|---|
| `code` | code | 编码：读写文件、跑命令、Git 操作 |
| `chat` | code | 轻量聊天：不绑定 workspace |
| `cowork` | knowledge | 知识工作：多根目录、连接器、调度 |
| `myhelper` | knowledge | 个人助手：消息平台接入 |

每种 Agent 的 `build_tools(context)` 返回不同的工具集。Agent 的 `family` 属性决定了它注册哪些额外工具（如 knowledge family 才有 scheduling 和 selfwake）。

## 15.6　与 aisuite 的关系

OpenWorker 基于 [aisuite](https://github.com/andrewyng/aisuite) 构建。aisuite 提供了：

- 统一的 `chat.completions` API 跨 provider
- `ToolMetadata` 携带 `risk_level`、`requires_approval`、`category` 等字段
- `Tools` 类自动从函数签名生成 JSON schema
- toolkit 概念（一组工具的集合）

OpenWorker 在此之上构建了：
- TurnEngine（aisuite 没有主循环）
- PermissionEngine（aisuite 没有权限系统）
- Inbox（aisuite 没有审批队列）
- 连接器系统（aisuite 有但不完整）
- 自动化调度（aisuite 没有）

README 中说：

> If you want to build your own agent harness rather than use ours, start there; this repo is a working reference for what aisuite can carry.

## 本章小结

OpenWorker 的扩展体系用 "Skill（操作指南）+ Persona（Agent 模板）+ Subagent（只读委派）" 的三层设计，提供了从轻到重的扩展路径。最值得借鉴的是：Skill 的渐进式披露（catalog + load 而非全量注入）、Persona 的 consent 流程（安装后 disabled，用户显式批准）、以及子 Agent 的"只做只读探索"的克制设计。

## 动手实验

1. 在 `skills/base.py` 中找到 `SkillLoader`，看它如何扫描两个目录。
2. 在 `personas/manifest.py` 中找到 persona manifest 的格式定义。
3. 在 `tools/subagent.py` 中找到 `explorer_tools()`，看子 Agent 的权限约束。
4. 创建一个自定义 Skill（在 `.coworker/skills/` 下放一个 SKILL.md），让 Agent 加载它。
