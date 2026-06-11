# 第 12 章　上下文工程：base instructions、AGENTS.md 与 compaction

第 11 章解决了"你是谁"——认证让 Codex 知道用哪个账户、哪份额度跑模型。但凭证只决定"能不能调模型"，不决定"模型能不能把活干好"。一个连不上你项目脉络的模型，纵使再聪明，也只会写出风格不符、违反约定、重复造轮子的代码。

于是有了**上下文工程**：在每一轮把请求发给模型之前，Codex 要精心组装一份"上下文"——它是谁（base instructions）、它在哪（环境上下文）、这个项目有什么规矩（`AGENTS.md`）、有哪些工具与技能可用、之前聊到哪了。这些信息不是随手拼起来的字符串，而是一套有**来源、有顺序、有预算、有边界**的装配流水线。而当对话越来越长、逼近模型上下文窗口上限时，第 4 章提过的 **compaction** 又要决定"丢什么、留什么"。这一章拆这条流水线。代码主要在 `core/src/agents_md.rs`、`core/src/session/mod.rs` 的 `build_initial_context`、`core/src/context/` 下的一众 fragment、以及 `core/src/compact.rs`。

---

## 12.1 三层指令：base / developer / contextual user

发给模型的指令不是一锅烩，而是分了**角色层级**。最底层是 `BaseInstructions`（`protocol/src/models.rs`）——一个朴素到只有一个字段的结构体：

```rust
pub const BASE_INSTRUCTIONS_DEFAULT: &str =
    include_str!("prompts/base_instructions/default.md");

pub struct BaseInstructions {
    pub text: String,
}

impl Default for BaseInstructions {
    fn default() -> Self {
        Self { text: BASE_INSTRUCTIONS_DEFAULT.to_string() }
    }
}
```

`default.md` 是 Codex 的"人格与宪法"，第一句就给模型定了位："You are a coding agent running in the Codex CLI, a terminal-based coding assistant... You are expected to be precise, safe, and helpful." 它用 `include_str!` 在**编译期**嵌进二进制——base instructions 不是运行时可被随意篡改的外部文件，而是和程序一起发布的常量。这本身就是一种安全姿态：模型的根指令不暴露在文件系统里任人改写。

`default.md` 里还藏着一份**给模型读的 `AGENTS.md` 规范**——它明确告诉模型：repo 里可能有 `AGENTS.md`，其作用域是"包含它的目录子树"，"更深层的 `AGENTS.md` 在冲突时优先"，"直接的 system/developer/user 指令优先于 `AGENTS.md`"，并且"从 CWD 到根目录沿途的 `AGENTS.md` 已经随 developer message 一起给你了，不必重读"。这句"不必重读"是理解整章的钥匙：**`AGENTS.md` 的内容是被 harness 预先注入的，不是模型自己去读的**——这正是 12.2 要拆的装配过程。

base 之上是 **developer** 层和 **contextual user** 层，由 `build_initial_context` 在每轮装配。前者放策略与能力（权限说明、可用工具/技能/插件），后者放"贴着用户身份"的上下文（`AGENTS.md` 用户指令、环境上下文）。三层各司其职，对应 Responses API 的不同角色——这与第 5 章"结构化输出"的思路一脉相承：连指令本身都讲究角色归属。

---

## 12.2 AGENTS.md 发现：从项目根到 cwd 的逐级拼接

`AGENTS.md` 是 Codex 上下文工程的灵魂——它让模型"懂你的项目"。发现逻辑全在 `core/src/agents_md.rs`，文件头注释把规则讲得一清二楚：

1. **定位项目根**：从当前工作目录**向上**走，直到撞见一个 `project_root_markers` 标记。标记默认是 `.git`（`default_project_root_markers` → `DEFAULT_PROJECT_ROOT_MARKERS = &[".git"]`）。找不到标记就只看 cwd；标记列表为空则禁用向上遍历。
2. **逐级收集**：从项目根**向下**到 cwd（含两端），把沿途每个 `AGENTS.md` 按顺序拼接。
3. **绝不越过项目根**——不会爬到你仓库外面去读别人的 `AGENTS.md`。

文件名也有讲究。`candidate_filenames` 的优先顺序是：`LOCAL_AGENTS_MD_FILENAME = "AGENTS.override.md"`（本地覆盖优先）、`DEFAULT_AGENTS_MD_FILENAME = "AGENTS.md"`，再加上用户配的 `project_doc_fallback_filenames`。`.override.md` 排在最前，意味着你能在不动团队共享的 `AGENTS.md` 的前提下，用一个本地覆盖文件加自己的私货——这是给"团队约定 vs 个人偏好"留的口子。

除了项目级，还有**全局级**：`load_global_instructions` 从 `$CODEX_HOME` 里同样按 `AGENTS.override.md` → `AGENTS.md` 的顺序找一份用户级指令。于是最终模型看到的指令栈是：全局用户级 `AGENTS.md` + 项目根到 cwd 沿途的项目级 `AGENTS.md`——**越靠近你正在编辑的目录，约定越具体、越靠后**。这与第 10 章配置分层"越靠近用户越优先"是同一种就近直觉。

---

## 12.3 预算与边界：32 KiB 的字节配额

`AGENTS.md` 是用户能任意写的文件——它可能很大，甚至大到把上下文窗口撑爆。所以 `read_agents_md` 给它设了一道**字节预算**：`project_doc_max_bytes`，默认 `DEFAULT_PROJECT_DOC_MAX_BYTES = 32 * 1024`（32 KiB，`core/src/config/mod.rs` 里别名为 `AGENTS_MD_MAX_BYTES`）。

预算的执行很克制：维护一个 `remaining` 计数器，逐个文件读取，**读一个扣一个**；当某个文件的大小超过剩余预算时，`data.truncate(remaining)` 把它截断，并 `tracing::warn!` 记一条"超预算被截断"的日志；`remaining == 0` 时直接停止收集。注意一个边界处理——`max_total == 0` 时 `read_agents_md` 直接返回 `Ok(None)`：把预算设成 0 就等于**彻底关闭** `AGENTS.md` 注入。给企业/特殊场景一个干净的总开关。

还有一道**编码边界**：`warn_invalid_utf8` 在读到非法 UTF-8 时不会崩、也不会静默，而是往 `startup_warnings` 里推一条明确的警告——"`AGENTS.md` 含非法 UTF-8，无效字节序列已被替换"，随后用 `String::from_utf8_lossy` 容错解码。这又是第 1 章"边界不信任、但要包容"的体现：用户写的文件可能编码不干净，Codex 既不信任它一定是合法 UTF-8，又不因此拒绝服务。

---

## 12.4 来源即结构：`InstructionProvenance` 与 project-doc 分隔符

收集来的指令不是一堆裸字符串，每一条都带**来源**。`LoadedAgentsMd` 里装的是 `InstructionEntry { contents, provenance }`，而 `provenance` 是个三态枚举：

```rust
enum InstructionProvenance {
    User(AbsolutePathBuf),     // 用户级，通常来自 CODEX_HOME
    Project(AbsolutePathBuf),  // 工作区里发现的项目级 AGENTS.md
    Internal,                  // 无文件来源的内部指令
}
```

为什么要记来源？因为拼接时**来源决定分隔符**。`LoadedAgentsMd::text()` 在把各条指令连起来时有一处精巧的判断：只有当从 `User`/`Internal` **过渡到** `Project` 时，才插入特殊分隔符 `AGENTS_MD_SEPARATOR = "\n\n--- project-doc ---\n\n"`；其余过渡用普通的 `"\n\n"`。注释解释得很清楚：这个 `--- project-doc ---` 标记是**告诉模型"工作区级别的指令从这里开始"**，所以只在"用户/内部 → 项目"这个边界上才需要。模型因此能分清哪些是用户全局偏好、哪些是这个仓库特有的约定——**来源不是元数据，它直接塑造了模型看到的文本结构**。

`Internal` 来源也有用武之地：当 `Feature::ChildAgentsMd` 开启时，`user_instructions` 会往末尾追加一条 `HIERARCHICAL_AGENTS_MESSAGE`（`prompts/templates/agents/hierarchical.md`），provenance 标为 `Internal`——它是 Codex 自己注入的层级化 agent 指引，不来自任何文件，但和用户写的 `AGENTS.md` 走同一条拼接管线。

---

## 12.5 装配总线：`build_initial_context`

把 base、developer、contextual user 三层真正拼成一组 `ResponseItem` 的，是 `session/mod.rs` 里那个体量很大的 `build_initial_context`。它像一条**装配总线**，按固定顺序往三个篮子里塞内容：`developer_sections`、`contextual_user_sections`、`separate_developer_sections`。

塞进 developer 篮子的（策略与能力，每一项都受一个 `include_*` 配置开关控制）：

- **模型切换提示**（`build_model_instructions_update_item`）——上一轮换了模型就告诉模型。
- **权限说明**（`PermissionsInstructions`）——把当前权限 profile、审批策略、execpolicy 规则渲染给模型看，让它知道自己能干什么、要不要请示（呼应第 9 章）。
- **developer 指令 / 协作模式指令 / realtime 更新 / 人格规格**。
- **可用 apps / 技能 / 插件**（`AppsInstructions`、`AvailableSkillsInstructions`、`AvailablePluginsInstructions`）——告诉模型有哪些扩展能力（第 14 章）。技能列表还受一个 `default_skill_metadata_budget(context_window)` 的预算约束，并在重复构建时发警告。
- **扩展贡献的 fragment**：`context_contributors` 让扩展按 `PromptSlot`（`DeveloperPolicy`/`DeveloperCapabilities`/`ContextualUser`/`SeparateDeveloper`）把内容投递到正确的篮子——又一处"统一契约、多态实现"。
- **token 预算上下文**（`TokenBudgetContext`，受 `Feature::TokenBudget` 控制）。

塞进 contextual user 篮子的（贴着用户身份的上下文）：

- **`AGENTS.md` 用户指令**：12.2 收集到的 `user_instructions` 在这里被 `UserInstructions` 包装渲染。它的标记很有意思——`("# AGENTS.md instructions for ", "</INSTRUCTIONS>")`，body 是 `"{directory}\n\n<INSTRUCTIONS>\n{text}\n"`，**把"这份指令针对哪个目录"和指令正文一起交给模型**。
- **环境上下文**（`EnvironmentContext`，见 12.6）。

最后由 `build_developer_update_item` / `build_contextual_user_message` 把各篮子聚合成 `ResponseItem`，guardian 策略提示则被刻意拆成**独立的** developer item（注释说为了"让 guardian 子 agent 看到一个清晰、易审计的指令块"）。整条总线的设计哲学是：**每一块上下文都有明确的角色归属和开关，能装也能关，可审计、可灰度。**

---

## 12.6 环境上下文：把"它在哪"结构化成 XML

模型要写好命令，得知道自己跑在什么环境里。`EnvironmentContext`（`core/src/context/environment_context.rs`）把这些信息渲染成一段 `<environment_context>` XML，塞进 contextual user 篮子。它包含：cwd、shell、`current_date`、`timezone`、网络约束（`<network>` 下的 allowed/denied 域名）、文件系统权限（`<filesystem>` 下的 workspace roots 与 permission profile）、以及子 agent 列表。

几个工程细节值得记：

- **多环境支持**：`EnvironmentContextEnvironments` 是 `None`/`Single`/`Multiple` 三态——Codex 支持多个工作区，渲染时单环境直接出 `<cwd>`/`<shell>`，多环境则出带 `id` 的 `<environments>` 列表。这与第 7 章 `TurnDiffTracker` 用 `environment_id` 区分多环境同名文件是同一套多环境模型。
- **XML 转义**：`push_xml_escaped_text` 把 `&`/`<`/`>`/`"`/`'` 一律转义成实体。为什么？因为 cwd、域名这些值来自真实环境、可能含特殊字符，不转义就会破坏 XML 结构、甚至让模型误解边界——**注入给模型的结构化文本也要防"注入"**。
- **轮间 diff**：`equals_except_shell` 和 `diff_from_turn_context_item` 让 Codex 能只在环境**变化时**重新告知模型，而非每轮重复整段——稳态下不重发全量元数据，省 token。这呼应了 base instructions 里那句"已经给你了，不必重读"。
- **权限即上下文**：`<filesystem>` 把第 8 章的 `WorkspaceWrite`、只读飞地等沙箱约束直接渲染给模型，连 `access="deny"` 的条目都标上 `escalatable="false"`——让模型**事先知道**哪些路径碰不得，而不是撞了沙箱南墙才知道。决策层、隔离层的约束在这里变成了模型可读的上下文。

---

## 12.7 Compaction：上下文逼近上限时丢什么、留什么

对话越长，历史越占 token。当上下文逼近窗口上限，Codex 触发 **compaction**——把一长段历史压成一份摘要，腾出空间继续干活。逻辑在 `core/src/compact.rs`。

核心思路是"**让另一个 LLM 写交接摘要**"。`SUMMARIZATION_PROMPT`（`prompts/templates/compact/prompt.md`）开宗明义："You are performing a CONTEXT CHECKPOINT COMPACTION. Create a handoff summary for another LLM that will resume the task." 它要求摘要覆盖：当前进度与关键决策、重要约束/用户偏好、待办的清晰下一步、以及继续工作所需的关键数据与引用。摘要生成后，前面会拼上 `SUMMARY_PREFIX`（`summary_prefix.md`）那段"另一个语言模型产出了它的思考摘要，请基于已完成的工作继续、避免重复劳动"的引导——**compaction 不是简单截断，而是一次"工作交接"**。

"丢什么、留什么"的策略在 `build_compacted_history` 里：

- **保留最近的用户消息**：从最新往旧遍历（`.iter().rev()`），在 `COMPACT_USER_MESSAGE_MAX_TOKENS = 20_000` 的 token 预算内尽量多留，超预算的那条用 `truncate_text` 截断——**离当下越近的用户意图越值得留**。
- **摘要垫底**：把生成的 summary 作为一条 user 消息追加到压缩历史末尾（空摘要则填 `"(no summary available)"`）。
- **初始上下文重注入**：`InitialContextInjection` 两态——pre-turn/手动压缩用 `DoNotInject`（清空引用、下一轮常规 turn 会重新注入完整初始上下文）；mid-turn 压缩必须用 `BeforeLastUserMessage`，因为模型被训练成"把压缩摘要看作历史最后一项"，于是 `insert_initial_context_before_last_real_user_or_summary` 精确地把初始上下文插在**最后一条真实用户消息之前**，保证摘要仍在最后。

还有两处兜底见功力。其一，压缩本身也可能撞 `ContextWindowExceeded`——这时 `history.remove_first_item()` **从最旧的一端**删历史项后重试。注释点明为什么从头删："Trim from the beginning to preserve cache (prefix-based)"——模型的前缀缓存依赖历史开头稳定，从尾巴删会让近期消息错位、缓存失效，从头删才能既腾空间又尽量保住缓存。其二，压缩成功后会发一条 `Warning`，老实告诉用户："长线程和多次压缩会让模型变得不那么准确，尽量开新线程"——**连"压缩有损"这件事都诚实地写给用户看**。

整个 compaction 全程进遥测：`CompactionAnalyticsAttempt` 记录压缩前后的 token、trigger（Auto/Manual）、reason、phase、status、耗时——团队能据此判断线上压缩到底省了多少、失败率多高（第 15 章）。

---

## 本章小结

- **三层指令**：`BaseInstructions`（`include_str!` 编译期嵌入的 `default.md`，是模型的"人格与宪法"，含一份给模型读的 `AGENTS.md` 规范）+ developer 层（策略/能力）+ contextual user 层（贴用户身份的上下文）。
- **AGENTS.md 发现**（`core/src/agents_md.rs`）：从 cwd 向上找项目根（默认标记 `.git`），再从根向下到 cwd 逐级拼接，**绝不越过项目根**；文件名优先级 `AGENTS.override.md` → `AGENTS.md` → 用户 fallback；全局级从 `$CODEX_HOME` 同序加载。
- **预算与边界**：`project_doc_max_bytes` 默认 32 KiB，逐文件扣减、超额截断并告警，设 0 即彻底关闭；非法 UTF-8 不崩不静默，告警 + `from_utf8_lossy` 容错。
- **来源即结构**：`InstructionProvenance`（User/Project/Internal）决定拼接分隔符——仅在"用户/内部 → 项目"过渡处插入 `--- project-doc ---` 标记，让模型分清全局偏好与仓库约定；`Feature::ChildAgentsMd` 开时追加 `Internal` 的层级化指引。
- **装配总线 `build_initial_context`**：按固定顺序把权限说明、协作模式、apps/技能/插件、扩展 fragment（按 `PromptSlot` 分流）、token 预算、`AGENTS.md` 用户指令、环境上下文塞进三个篮子，每项受 `include_*` 开关控制——能装能关、可审计可灰度。
- **环境上下文**（`EnvironmentContext`）：渲染 `<environment_context>` XML（cwd/shell/date/timezone/network/filesystem/subagents），多环境三态、XML 转义防注入、轮间 diff 省 token、把第 8 章沙箱约束作为模型可读上下文（`escalatable="false"`）。
- **Compaction**（`core/src/compact.rs`）：用 `SUMMARIZATION_PROMPT` 让 LLM 写"交接摘要"；`build_compacted_history` 在 `COMPACT_USER_MESSAGE_MAX_TOKENS = 20_000` 预算内从新到旧保留用户消息、摘要垫底；`InitialContextInjection` 两态决定是否重注入初始上下文；撞窗口上限时 `remove_first_item` 从头删以保前缀缓存；压缩有损会发 Warning 提醒；全程进遥测。

## 动手实验

1. **实验一：跟踪 AGENTS.md 发现** — 阅读 `core/src/agents_md.rs` 的 `agents_md_paths`，说明它如何从 cwd 向上找到项目根（默认标记是什么？在哪定义？），再从根向下收集。假设你在 `repo/a/b/` 编辑、`repo/`、`repo/a/`、`repo/a/b/` 各有一个 `AGENTS.md`，列出它们被拼接的顺序。
2. **实验二：验证字节预算** — 找到 `DEFAULT_PROJECT_DOC_MAX_BYTES`（值是多少？在哪个 crate？）与 `read_agents_md` 里的 `remaining` 扣减逻辑。解释当 `project_doc_max_bytes = 0` 时会发生什么，以及一个超大 `AGENTS.md` 会被如何处理、用户会看到什么日志。
3. **实验三：读懂 project-doc 分隔符** — 阅读 `LoadedAgentsMd::text()` 与 `InstructionProvenance`，解释为什么 `--- project-doc ---` 分隔符只在"User/Internal → Project"的过渡处出现、而不是每条指令之间都插。这个标记是给谁看的、起什么作用？
4. **实验四：画出 compaction 的取舍** — 阅读 `core/src/compact.rs` 的 `build_compacted_history` 与 `COMPACT_USER_MESSAGE_MAX_TOKENS`。说明它为什么从最新用户消息往旧保留、摘要为什么垫在最后；再看 `ContextWindowExceeded` 分支的 `remove_first_item`，解释"从最旧一端删"为什么能保住模型的前缀缓存。

> **下一章预告**：上下文工程让模型"懂"了你的项目，但一旦关掉终端，这一切会不会烟消云散？下一章我们进入状态持久化：Codex 如何用 **rollout** 把每一轮对话、每一次工具调用落盘成可回放的记录，如何用 SQLite 管理会话索引，以及"恢复一个历史会话"时这些记录如何被重新装回内存——让一次中断的编码之旅可以从断点续上。
