# 第 13 章　Skill 系统：发现、加载、allowed-tools 收窄与自我进化

子 agent 让能力横向扩展,但每个 agent——无论主从——真正「会干什么专业活」靠的是 **skill**:一份份写在 `skills/` 目录下、用 Markdown + YAML front-matter 描述的操作手册。「做 PPT」「解析 PDF」「调某个内部 API」这些专门知识不该硬编码进框架,而是按需加载的 skill。这一章我们走进 `skills/` 分区,看 DeerFlow 怎么发现并解析 skill、怎么用 skill 自带的 `allowed-tools` 反过来**收窄**这个 agent 能用的工具、怎么把 public 内置 skill 和 custom 用户 skill 隔离开,以及最危险也最有趣的一环——`skill_manage_tool` 如何让 agent 在运行中**进化出新技能**,又怎样用一道 LLM 安全扫描把这种「自我修改」关进笼子。

## skill 的数据结构:一份手册的元数据

skill 的核心定义在 `types.py` 的 `Skill` dataclass。它带 `name`、`description`、`license`、路径信息、`category`、`allowed_tools`、`enabled` 等字段。`category` 是个 `StrEnum`,只有两个值,注释把区别讲死了:

```python
class SkillCategory(StrEnum):
    PUBLIC = "public"   # 平台自带的内置 skill,只读
    CUSTOM = "custom"   # 用户写的 skill,可编辑可删除
```

这条「public 只读 / custom 可写」的分界,是整个 skill 系统权限模型的地基——后面所有「能不能改这个 skill」的判断都回到它。`Skill` 还有一组 `get_container_path` 之类的方法,把宿主机上的 skill 目录翻译成沙箱里的 `/mnt/skills/{category}/{name}` 路径——模型永远只看到容器视角的虚拟路径(呼应第 8 章)。

## 解析 SKILL.md:容错但不放水的 front-matter

每个 skill 是一个目录,里头有一个 `SKILL.md`,顶部是 `---` 围起来的 YAML front-matter。`parse_skill_file`(`parser.py`)负责把它解析成 `Skill`。它的容错姿态很克制:

- front-matter 用正则 `^---\s*\n(.*?)\n---\s*\n` 提取,提不出来就返回 `None`(不是这个格式就不当 skill)。
- `name` 和 `description` 是**必填且必须是非空字符串**,缺一个就返回 `None`——没有名字或描述的「手册」没意义。
- YAML 解析失败不是默默吞掉,而是调 `_format_yaml_error` 渲染一份**对作者友好**的报错:它能算出出错的**文件行号**(注释解释 `mark.line + 2` 是怎么把「front-matter 内部 0-based 行号」换算成「作者编辑器里看到的 1-based 行号、且算上被正则剥掉的 `---` 围栏」的),还会针对最常见的错误(值里含 `: ` 却没加引号)给一句精准的 `hint`。这是「错误即文档」做到极致——连 YAML 报错都帮作者定位到行、给出修复示范。

`parse_allowed_tools` 单独处理 `allowed-tools` 字段:省略返回 `None`(传统的「允许所有」),是个字符串列表就收下(空列表也合法,表示「这个 skill 不需要任何工具」),格式不对就 `raise ValueError`。注意 `None` 和 `[]` 的语义差别——这在下一节的工具收窄里是关键。

## allowed-tools:skill 反过来收窄 agent 的工具

这是 skill 系统最体现「harness 约束模型」的一手。常识里 skill 是「增加能力」的,但 DeerFlow 让 skill 同时能**减少**能力——一个 skill 可以声明「我这套流程只该用这几个工具」,从而把 agent 的工具面收窄到一个安全子集。逻辑在 `tool_policy.py`:

```python
def allowed_tool_names_for_skills(skills):
    if not skills:
        return None
    allowed = set()
    has_explicit_declaration = False
    for skill in skills:
        if skill.allowed_tools is None:
            continue
        has_explicit_declaration = True
        allowed.update(skill.allowed_tools)
    if not has_explicit_declaration:
        return None
    return allowed
```

它的精妙在于处理「混用」的语义:**只要有任何一个加载的 skill 声明了 `allowed-tools`,就进入「显式限制」模式**——此时没声明该字段的传统 skill **不贡献任何工具**,而不是用它的「允许所有」把别的 skill 的限制冲垮。注释说得很清楚:legacy skill 不该「disable the explicit restrictions from other skills」。返回 `None` 只在「没有任何 skill 声明 allowed-tools」时发生,意味着回退到「允许所有」。`filter_tools_by_skill_allowed_tools` 拿这个集合去过滤工具列表——这正是第 12 章 `SubagentExecutor._apply_skill_allowed_tools` 调用的那个函数。**取并集、但一旦有人显式声明就以显式为准**,这套规则保证了「约束只会收紧」:加载一个声明了窄 allowed-tools 的 skill,只会让 agent 的工具面变窄,绝不会因为同时加载了一个「宽松」的 legacy skill 而被放宽。

## 加载与启用状态:每次都重读,跨进程一致

`SkillStorage.load_skills`(`skill_storage.py`)是个模板方法:先 `_iter_skill_files` 遍历 `public/` 和 `custom/` 两个目录下所有 `SKILL.md`、逐个 `parse_skill_file`,再**合并启用状态**。启用状态不存在 skill 文件里,而是来自 `ExtensionsConfig`:

```python
extensions_config = ExtensionsConfig.from_file()
for skill in skills:
    skill.enabled = extensions_config.is_skill_enabled(skill.name, skill.category)
```

注释强调「re-read every call so changes made by another process are picked up immediately」——每次加载都从磁盘重读启用配置,因为 Gateway API 可能在另一个进程里改了「哪些 skill 启用」。这与第 7 章 MCP 工具用 `ExtensionsConfig.from_file()` 重读是同一个边界自觉:**配置可能被另一个进程改动,所以不缓存、每次读最新**。`_iter_skill_files` 遍历时 `dir_names[:]` 过滤掉 `.` 开头的隐藏目录(比如 `.history`),并 `sorted` 保证确定性顺序;`enabled_only=True` 时只留启用的;最后按 name 排序。

## 路径安全:skill 名校验、相对路径不许逃逸

`SkillStorage` 里一连串静态校验,把「agent 写 skill」这件危险事钉在笼子里:

- **skill 名格式**:`_SKILL_NAME_PATTERN = ^[a-z0-9]+(?:-[a-z0-9]+)*$`——只允许小写字母、数字、连字符(hyphen-case),且 ≤64 字符。这堵死了用 skill 名做路径穿越(`../`)的可能。
- **相对路径不许逃逸**:`validate_relative_path` 把相对路径 join 到 base_dir 后 `.resolve()`,再用 `relative_to` 确认仍在 base 之内——和第 8 章 `LocalSandbox` 的逃逸检查同款。
- **支持文件只能放白名单子目录**:`ensure_safe_support_path` 规定支持文件只能落在 `{references, templates, scripts, assets}` 四个子目录之一,且不许 `..`、不许绝对路径、必须带文件名。
- **front-matter 的 name 必须和请求的 name 一致**:`validate_skill_markdown_content` 解析内容里的 name,和调用方传的 name 不符就拒——防止「声称改 A 实际写 B」。

`custom` 可写而 `public` 只读这条铁律由 `ensure_custom_skill_is_editable` 守:要改的 skill 在 custom 里就放行;如果它其实是个 public 内置 skill,报一句明确的指引——「这是内置 skill,要定制请在 `skills/custom/` 下新建一个同名 skill」;都不是就 `FileNotFoundError`。原子写沿用第 11 章的套路:`tempfile` 写临时文件再 `replace`。

## skill_manage_tool:让 agent 在运行中进化技能

`skill_manage_tool`(`skill_manage_tool.py`)是「skill 自我进化」的入口——agent 可以在跑的过程中**创建、编辑、打补丁、删除**custom skill。它支持六个 action:`create` / `edit` / `patch` / `delete` / `write_file` / `remove_file`。这是一种很激进的能力:agent 学会一套新流程后,能把它固化成一个 skill,下次直接复用。但「让模型修改自己的能力定义」显然是个潜在后门,所以这个工具被一层层守卫包着。

第一层守卫是**默认关闭**。它只有在 `skill_evolution.enabled` 为真时才挂载给模型(第 7 章 `get_available_tools` 的按开关挂),而 `SkillEvolutionConfig.enabled` **默认 `False`**:

```python
class SkillEvolutionConfig(BaseModel):
    enabled: bool = Field(default=False, ...)
    moderation_model_name: str | None = Field(default=None, ...)
```

「能自我修改」这种高危能力,默认就是不给的——又一次「默认即关闭」。

第二层是**按 skill 名加锁**:`_skill_locks` 用 `WeakValueDictionary` 给每个 skill 名分配一把 `asyncio.Lock`,所有写操作 `async with lock`——防止同一个 skill 被并发的两次修改撞坏。第三层是**每次写都记 history**:每个 action 都 `append_history` 一条 JSONL 记录到 `custom/.history/<name>.jsonl`,记下 action、author=`agent`、thread_id、改动前后内容、扫描结果——agent 对 skill 的每一次自我修改都可审计、可回溯。

## LLM 安全扫描:写盘前先过一道「安全审查员」

最关键的守卫是 `_scan_or_raise` → `scan_skill_content`(`security_scanner.py`):**任何内容写进 skill 之前,先让一个 LLM 当「安全审查员」扫一遍**。它给模型一份 rubric,要求把内容分类成 `allow` / `warn` / `block`——明确要 block 掉 prompt 注入、系统角色覆盖、提权、数据外泄、不安全的可执行代码。返回必须是单行 JSON。

它的失败处理是教科书级的 **fail-closed**:

```python
if model_responded:
    return ScanResult("block", "Security scan produced unparseable output; manual review required.")
if executable:
    return ScanResult("block", "Security scan unavailable for executable content; manual review required.")
return ScanResult("block", "Security scan unavailable for skill content; manual review required.")
```

也就是说——扫描模型返回了但**解析不出有效 JSON**?block。扫描模型**根本没调通**(异常)?block。无论哪种「拿不准」的情况,默认结论都是 block,而不是「扫不了就放行」。`_scan_or_raise` 还对**可执行内容**加码:写到 `scripts/` 下的文件被标记 `executable=True`,此时扫描结果**必须是 `allow`**(连 `warn` 都不行)才放行,decision 是 `block` 或非 `allow` 一律 `raise ValueError`。脚本能直接跑,所以审查标准比纯文本更严。这层「自我修改前先过 LLM 安检、且安检失败默认拒绝」的设计,把「agent 进化」这件危险事的风险压到了最低。

写成功后,`create`/`edit`/`patch`/`delete` 都会 `refresh_skills_system_prompt_cache_async()` 刷新系统提示里的 skill 列表缓存——新进化出的 skill 立刻对模型可见。

## 安装外部 skill:zip 炸弹、符号链接与路径穿越

除了 agent 自己进化,skill 也能从 `.skill` 归档(本质是 zip)安装。`safe_extract_skill_archive`(`installer.py`)把解压这件危险事做足了防御:

- **拒绝绝对路径和 `..` 穿越**:`is_unsafe_zip_member` 检出就 `raise`。
- **跳过符号链接条目**:不 materialise 符号链接(防止用软链指到笼外),只 `logger.warning` 跳过。
- **zip 炸弹防御**:`max_total_size = 512 * 1024 * 1024`(512MB)硬上限,累计解压超过就拒。

安装流程也很讲究边界:解压、校验在 worker 线程跑(`asyncio.to_thread`),但**逐文件安全扫描是 async LLM 调用,必须留在事件循环上**;先解压到临时目录、`_validate_skill_frontmatter` 校验、`_scan_skill_archive_contents_or_raise` 逐个扫描所有文本和脚本文件(脚本按 `executable=True` 扫)、再 `_move_staged_skill_into_reserved_target` 用「先 `mkdir` 占坑、失败则清理」的方式原子地移进目标位置。`finally` 里的临时目录清理还包了 `asyncio.wait_for(..., timeout=5)`——防止卡死的文件系统(比如 NFS)把安装结果的传播挡在 finally 块里。连「清理临时目录会不会卡」都考虑到了。

## 存储可插拔:又一个配置驱动

和模型、工具、沙箱一样,skill 存储也是配置驱动的。`SkillsConfig.use` 默认 `deerflow.skills.storage.local_skill_storage:LocalSkillStorage`,可以换成别的后端。`get_skills_path` 的解析顺序也很周到:显式 `path` > `DEER_FLOW_SKILLS_PATH` 环境变量 > 项目根下的 `skills/` > monorepo 兼容的 legacy 位置 > 兜底返回项目根默认(即使不存在也给一个稳定的「无 skill」位置而不抛错)。`container_path` 默认 `/mnt/skills`——和沙箱、解析器里的默认值前后一致。

## 本章小结

- `Skill`(`types.py`)核心字段 + `SkillCategory` 二值枚举:`PUBLIC` 内置只读 / `CUSTOM` 用户可写,是整个权限模型的地基;路径经 `get_container_path` 翻译成 `/mnt/skills/{category}/{name}` 虚拟路径。
- `parse_skill_file` 容错但不放水:front-matter 提不出/`name`/`description` 缺失返回 `None`;YAML 错误用 `_format_yaml_error` 给出文件行号 + 加引号 hint(错误即文档);`parse_allowed_tools` 区分 `None`(允许所有)与 `[]`(无工具)。
- `allowed-tools` 让 skill 反向收窄工具:`allowed_tool_names_for_skills` 取并集,但**任一 skill 显式声明即进入限制模式**,未声明的 legacy skill 不再贡献「允许所有」——保证约束只收紧不放宽;被第 12 章子 agent 复用。
- `load_skills` 每次都 `ExtensionsConfig.from_file()` 重读启用状态(跨进程一致,同第 7 章 MCP);路径安全靠 `_SKILL_NAME_PATTERN`(hyphen-case ≤64)、`validate_relative_path` 防逃逸、`ensure_safe_support_path` 限白名单子目录、front-matter name 必须匹配;`ensure_custom_skill_is_editable` 守 public 只读。
- `skill_manage_tool` 让 agent 运行中进化 skill(create/edit/patch/delete/write_file/remove_file),四层守卫:① `skill_evolution.enabled` 默认 False;② 按 skill 名 `asyncio.Lock`;③ 每次写记 JSONL history(author=agent、含前后内容);④ 写盘前 `scan_skill_content` LLM 安检。
- 安全扫描 fail-closed:解析不出 / 模型没调通 / 不可用一律 `block`;`scripts/` 下可执行内容必须 `allow`(连 `warn` 都拒)。
- 外部 `.skill` 安装防御:`safe_extract_skill_archive` 拒绝绝对路径/`..`、跳过符号链接、512MB zip 炸弹上限;逐文件 LLM 扫描留在事件循环、文件操作走 worker 线程;临时目录清理 `wait_for(timeout=5)` 防卡。
- skill 存储配置驱动:`SkillsConfig.use` 可插拔,`get_skills_path` 多级回退,`container_path` 默认 `/mnt/skills` 全局一致。

## 动手实验

1. **实验一:推演 allowed-tools 的混用语义** — 阅读 `allowed_tool_names_for_skills`。给定三个加载的 skill:A 声明 `allowed-tools: [read_file, write_file]`、B 声明 `allowed-tools: []`、C 没声明该字段(`None`)。推演:函数返回什么集合?agent 最终能用哪些工具?如果把 A、B 都去掉只留 C,返回又是什么?为什么「任一 skill 显式声明即进入限制模式」能保证约束只收紧?
2. **实验二:跟踪一次 skill 进化的安检** — 通读 `skill_manage_tool` 的 `create` 分支与 `_scan_or_raise`、`scan_skill_content`。假设 agent 调 `create` 写一个 `SKILL.md`,但安全扫描模型这次返回了一坨无法解析成 JSON 的散文。推演:`scan_skill_content` 返回什么 `decision`?`_scan_or_raise` 会怎样?skill 会被写盘吗?再假设是写一个 `scripts/run.sh` 且扫描返回 `warn`,结果有何不同?
3. **实验三:验证 public/custom 隔离** — 阅读 `ensure_custom_skill_is_editable` 与 `SkillCategory`。假设存在一个 public 内置 skill 叫 `pdf`,agent 调 `skill_manage(action="edit", name="pdf", ...)`。推演会发生什么、报什么错、给出的指引是什么?为什么 DeerFlow 不允许直接改 public skill,而要求「新建同名 custom skill」?
4. **实验四:核对归档安装的三道防御** — 通读 `safe_extract_skill_archive`。分别构造三个会被拒/被特殊处理的归档条目:① 一个成员路径是 `/etc/passwd`;② 一个成员是指向 `/etc/shadow` 的符号链接;③ 一个解压后累计 600MB 的归档。推演每种各走哪条分支(`raise` / `warning` 跳过 / `raise`)。为什么解压必须在 zip 解析阶段就拦,而不能等写盘时再查?

> **下一章预告**:skill、子 agent、工具都能由外部提供方注入,而把这些「外部能力」接进来的统一通道是 MCP(Model Context Protocol)与扩展机制。下一章我们走进 MCP 集成,看 DeerFlow 如何发现并连接 MCP server、如何把它们暴露的工具纳入第 7 章的 deferred tool search、扩展配置如何热加载,以及它怎样在「接入第三方能力」与「不被第三方拖垮」之间设防。
