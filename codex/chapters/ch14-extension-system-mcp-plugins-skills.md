# 第 14 章　扩展体系：MCP、插件与技能

第 13 章把会话的"记忆"落了盘——rollout 与 SQLite 让一次中断的编码之旅能从断点续上。但 Codex 真正的威力,不只在于它记得住,更在于它**长得出**：内置工具(`shell`、`apply_patch`、`read_file`……)只是它能力的底座,真正让它从"一个会写代码的终端助手"变成"能接通整个外部世界"的,是它的**扩展体系**——挂接 **MCP 服务器**接入第三方工具,加载**插件**把一组能力打包分发,调用**技能**(skill)把领域知识与工作流程注入上下文。

这一章拆这三层扩展如何被统一抽象、如何安全地暴露给模型、又如何顺着第 12 章那条"装配总线"投递到正确的位置。代码主要在 `plugin/`、`core-plugins/`、`core-skills/`、`rmcp-client/` 这几个 crate,以及 `core/src/` 下的 `mcp_tool_exposure.rs`、`mcp_tool_call.rs`、`mcp_skill_dependencies.rs`。

---

## 14.1 一张能力总账：插件打包技能、MCP 服务器、apps 与 hooks

理解扩展体系的入口,是 `plugin/src/lib.rs` 里那个 `PluginCapabilitySummary`——它是一个插件"能带来什么"的总账:

```rust
pub struct PluginCapabilitySummary {
    pub config_name: String,
    pub display_name: String,
    pub description: Option<String>,
    pub has_skills: bool,
    pub mcp_server_names: Vec<String>,
    pub app_connector_ids: Vec<AppConnectorId>,
}
```

这个结构体一字道破整套设计:**一个插件是一个捆绑包**,里面可以同时装着技能(`has_skills`)、若干 MCP 服务器(`mcp_server_names`)、若干 app 连接器(`app_connector_ids`),外加 hooks(`PluginHookSource`)。换句话说,MCP、技能、app、hook 不是四套各自为政的机制,而是**同一个分发单元下的四类贡献**。插件是"壳",技能/MCP/app/hook 是"瓤"。这让 Codex 能用一条统一的安装、启停、遥测管线(`PluginTelemetryMetadata`)去管理形态各异的扩展,而不必为每种能力单独造一套生命周期。

后面三节正是沿着这三类"瓤"展开:技能(14.2–14.4)、MCP(14.5–14.7)、以及把它们绑在一起的插件清单与安全边界(14.8)。

---

## 14.2 技能是什么：一份带元数据的 `SKILL.md`

技能是扩展体系里最"轻"的一种——它不是代码,而是**一份本地指令文件**。`core-skills/src/render.rs` 里给模型的介绍一句话说清:"A skill is a set of local instructions to follow that is stored in a `SKILL.md` file."(技能是一组写在 `SKILL.md` 里、供模型遵循的本地指令。)

每个技能在内存里是一个 `SkillMetadata`(`core-skills/src/model.rs`):

```rust
pub struct SkillMetadata {
    pub name: String,
    pub description: String,
    pub short_description: Option<String>,
    pub interface: Option<...>,
    pub dependencies: Option<SkillDependencies>,
    pub policy: Option<SkillPolicy>,
    pub path_to_skills_md: AbsolutePathBuf,
    pub scope: SkillScope,
    pub plugin_id: Option<...>,
}
```

几个字段后文会反复用到:`scope` 标明这条技能从哪个层级来(`SkillScope` 四态:`System`/`Admin`/`Repo`/`User`),`plugin_id` 把它回指到所属插件,`dependencies` 声明它运行时需要哪些工具(尤其是 MCP 工具,见 14.7),`policy` 则控制它**能不能被模型自动调用**。

关键的"渐进式披露"(progressive disclosure)思想就藏在介绍文本里:模型在上下文里看到的只是技能的**名字 + 描述 + 路径**,真正的指令正文留在磁盘上的 `SKILL.md`。`SKILLS_HOW_TO_USE_WITH_ABSOLUTE_PATHS` 明确要求:决定使用某技能后,"the main agent must open and read its `SKILL.md` completely before taking task actions"(主 agent 必须在动手前完整读完 `SKILL.md`,被截断/分页就读到 EOF)。这是一种刻意的**两段式加载**:列表廉价地告诉模型"有什么",用到时才花 token 去读"怎么做"——上下文窗口因此不必为每个技能的全文买单。

---

## 14.3 隐式调用策略与产品门控：技能不是默认就能被自动触发的

技能虽轻,却不是"放进去就任模型随便用"。`SkillPolicy` 给每条技能装了两道闸:

```rust
pub struct SkillPolicy {
    pub allow_implicit_invocation: Option<bool>,
    pub products: Vec<Product>,
}
```

第一道是**隐式调用开关**。`allows_implicit_invocation()` 读 `policy.allow_implicit_invocation`,**缺省时默认为 `true`**——即技能默认允许模型在"任务匹配描述"时自行触发,但作者可以显式关掉它,把某条技能变成"只有用户用 `$SkillName` 点名才生效"。这道闸直接决定了一条技能会不会出现在 14.4 渲染给模型的"可用技能列表"里:`build_available_skills` 第一步就是 `outcome.allowed_skills_for_implicit_invocation()`,把不允许隐式调用的技能挡在列表之外。

第二道是**产品门控**。`products` 列表为空时该技能对所有产品可见;非空时只对 `matches_product_restriction` 命中的产品可见。`filter_skill_load_outcome_for_product` 在加载阶段就按当前运行的产品(CLI / IDE / ...)裁掉不适用的技能——**同一份技能库,在不同形态的 Codex 里能看到不同的子集**。

这两道闸合起来,体现的是扩展体系一以贯之的姿态:**能力默认收敛、显式放开**。技能作者既能把高频技能设成"自动触发",也能把危险或场景专属的技能锁成"点名才用"。

---

## 14.4 把技能塞进上下文：2% 预算下的"逐字符分配"

第 12 章讲 `build_initial_context` 时提到,可用技能由 `AvailableSkillsInstructions` 渲染进 developer 篮子,且受 `default_skill_metadata_budget(context_window)` 约束。这一节把那个预算的算法拆开——它是 `core-skills/src/render.rs` 里相当精巧的一段工程。

**预算从哪来**:`default_skill_metadata_budget` 默认取上下文窗口的 **2%**(`SKILL_METADATA_CONTEXT_WINDOW_PERCENT = 2`)作为 token 预算;拿不到窗口大小时,退回字符预算 `DEFAULT_SKILL_METADATA_CHAR_BUDGET = 8_000`。代码里的测试把这点钉死:`default_skill_metadata_budget(Some(200_000))` 得到 `Tokens(4_000)`(即 20 万的 2%)。**技能元数据再多,也只准占上下文窗口的一个零头**——这是对"技能列表撑爆 prompt"的硬约束。

**预算怎么花**:`render_skill_lines_from_lines` 分三档处理:

1. 全量放得下(`full_cost <= limit`)→ 每条技能都带完整描述渲染。
2. 全量放不下、但"最小行"(只有 `- name: (file: path)`、没描述)放得下 → 进入**描述预算分配**:把剩余预算"一个字符一个字符"地分给各技能的描述。注释说得很形象——"Distribute description space one character at a time across skills. Short descriptions naturally drop out, so their unused share can go to longer descriptions"(逐字符分配,短描述自然先满,腾出的份额流向长描述),避免按固定配额切分时短描述浪费、长描述饿死。
3. 连最小行都放不下 → `render_minimum_skill_lines_until_budget` 按优先级顺序逐条塞,塞不下的计入 `omitted_count`。

**谁先被保留**:排序由 `prompt_scope_rank` 决定——`System(0)` > `Admin(1)` > `Repo(2)` > `User(3)`,同级再按名字。预算紧张时,系统级技能比用户级技能更值得留。

**省 token 的别名表**:当技能来自插件缓存目录(路径极长且共享前缀),`build_aliased_available_skills` 会建一张别名表,把长根路径压成 `r0`、`r1`,正文里写 `r0/skill-0/SKILL.md`。只有当别名版"能装下更多技能或截断更少描述"时才采用(`aliased_render_is_better`)——**为省 token 而引入的复杂度,本身也要先证明它真的省了**。

**诚实的降级告警**:描述被截到平均超过 `SKILL_DESCRIPTION_TRUNCATION_WARNING_THRESHOLD_CHARS = 100` 字符,或有技能被完全省略,就发一条 `warning_message`,直白告诉用户"技能描述被压缩了""禁用不用的技能/插件能腾出空间"——这与第 12 章 compaction"压缩有损就告诉用户"是同一种诚实。

最后,被选中的技能正文怎么进上下文?`skill_instructions.rs` 的 `SkillInstructions` 实现了 `ContextualUserFragment`,角色为 `"user"`,标记是 `("<skill>", "</skill>")`,body 把 `name`/`path`/`contents` 包进去。这正是第 12 章 `context_contributors`/`PromptSlot` 那套"统一契约、多态实现"的落地——**技能贡献的 fragment,顺着同一条装配总线,被投递到 contextual user 篮子**。

---

## 14.5 MCP 客户端：四种传输与"连接中/就绪/关闭"状态机

技能是把"知识"注入模型,MCP(Model Context Protocol)则是把"工具"接进 Codex。`rmcp-client` crate 实现了 MCP 客户端,核心是 `RmcpClient`。它对外屏蔽了"工具到底跑在哪"——`TransportRecipe` 枚举出四种连接方式:

```rust
enum PendingTransport {
    InProcess { ... },                 // 进程内(测试/内置)
    Stdio { ... },                     // 子进程,走 stdin/stdout
    StreamableHttp { ... },            // 远端 HTTP
    StreamableHttpWithOAuth { ... },   // 远端 HTTP + OAuth 鉴权
}
```

连接的生命周期由 `ClientState` 三态机管理:`Connecting { transport }`(握手中,持有待启动的传输)→ `Ready { service, oauth }`(就绪,持有运行中的 rmcp 服务和可选的 OAuth 持久器)→ `Closed`。这种"延迟到就绪"的设计,让客户端能把建连接的开销推迟到真正要用时。

**stdio 服务器**怎么起?`stdio_server_launcher.rs` 的 `LocalStdioServerLauncher` 先用 `program_resolver::resolve` 把命令名解析成可执行路径(带上环境变量与 cwd),再 `spawn` 出子进程,rmcp 通过子进程的 stdin/stdout 与之对话。除了本地启动器,还有 `ExecutorStdioServerLauncher` 把启动委托给一个 exec 后端——**同一套 `StdioServerLauncher` trait,本地直起或受控代理两种实现**,呼应第 8 章沙箱执行的多后端模型。

**远端 HTTP** 的握手不靠运气。`streamable_http_retry.rs` 定义了 `STREAMABLE_HTTP_RETRY_DELAYS_MS: [u64; 2] = [250, 1_000]`——握手失败按 250ms、1000ms 两段退避重试,最多 `len() + 1 = 3` 次尝试。网络抖动不至于让一个 MCP 服务器一连不上就彻底放弃。

还有一处并发细节:`ElicitationPauseState`。MCP 服务器可以反向向用户"征询"(elicitation),征询进行时需要暂停某些活动——它用一个 `AtomicUsize` 引用计数 + `watch::channel<bool>`,第一个征询进入时置 `paused = true`,全部退出时恢复,用 RAII guard(`ElicitationPauseGuard`)保证配对。

---

## 14.6 工具暴露的阀门：100 个工具就转入"延迟披露"

MCP 服务器一接,可能瞬间涌入成百上千个工具定义。**全塞进 prompt 会把上下文淹掉**。`core/src/mcp_tool_exposure.rs` 的 `build_mcp_tool_exposure` 是这道阀门:

```rust
pub(crate) const DIRECT_MCP_TOOL_EXPOSURE_THRESHOLD: usize = 100;
```

逻辑是:先把所有"模型可见"(`tool_is_model_visible`)的 MCP 工具收集起来——非 `CODEX_APPS_MCP_SERVER_NAME` 的常规工具直接收,codex-apps 的工具则要其 `connector_id` 在已启用的连接器白名单里、且 `codex_app_tool_is_enabled` 为真才收。然后判断要不要**延迟披露**(defer):当搜索工具已启用,且(开了 `Feature::ToolSearchAlwaysDeferMcpTools` **或** 工具数 `>= 100`)时,就不把工具直接列进 prompt,而是放进 `deferred_tools`——模型需要时通过搜索工具按需检索,而不是开局就背下全部清单。

这与第 12 章技能的 2% 预算是同一种哲学的两个变体:**上下文是稀缺资源,海量能力清单必须"按需展开"而非"全量预载"**。技能用逐字符截断 + 别名压缩,MCP 用阈值触发的延迟披露,殊途同归。

`McpToolExposure` 的返回也很干净:要么 `direct_tools` 有内容、`deferred_tools` 为 `None`(全部直接暴露),要么反过来(全部延迟)——非此即彼,不会两头都给,避免模型看到重复或矛盾的工具视图。

---

## 14.7 命名即契约：`mcp__{transport}__{identifier}` 与技能的 MCP 依赖

MCP 工具进了 Codex,得有个**不会撞车的名字**。命名规则在 `mcp_skill_dependencies.rs` 的 `canonical_mcp_key`:

```rust
fn canonical_mcp_key(transport: &str, identifier: &str, fallback: &str) -> String {
    let identifier = identifier.trim();
    if identifier.is_empty() {
        fallback.to_string()
    } else {
        format!("mcp__{transport}__{identifier}")
    }
}
```

也就是 `mcp__<传输>__<标识>`——`stdio` 传输用启动命令作标识(`canonical_mcp_key("stdio", command, name)`),`streamable_http` 传输用 URL 作标识。**用"传输 + 来源"而非仅仅服务器名做 key**,意味着同名服务器只要传输/命令/URL 不同就不会冲突,这是个稳健的去重锚点。

这套命名还服务于 14.2 提到的**技能 MCP 依赖**。一条技能可以在 `SkillDependencies.tools` 里声明"我需要某个 MCP 工具":`canonical_mcp_dependency_key` 把依赖项(默认传输 `streamable_http`)归一成同样的 `mcp__...` key。当用户启用一个声明了 MCP 依赖的技能,而对应的 MCP 服务器**还没配置**时,Codex 能检测出 `missing` 依赖(`format_missing_mcp_dependencies` 列出缺失服务器名),并通过 `filter_prompted_mcp_dependencies` 避免对同一个服务器反复打扰用户(已 prompt 过的用 `canonical_mcp_server_key` 去重)。**技能与 MCP 不是孤岛——技能能声明它依赖的工具,系统据此把缺失的依赖补齐或提示安装**。

至于 MCP 工具被调用时的**审批**,逻辑在 `mcp_tool_call.rs`:它复用了第 9 章的审批/guardian 框架,审批结果可以"本次会话持久"(`PERSIST_SESSION`)或"永久"(`PERSIST_ALWAYS`),也支持按上下文自动批准(`mcp_permission_prompt_is_auto_approved`)。**第三方工具被纳入了和内置工具同一套权限模型**,而不是在权限体系外开后门——这正是本章开篇说的"统一抽象"的要害。

---

## 14.8 插件清单与安全边界：`plugin.json` 与 `./`-only 路径

回到 14.1 的"壳"。一个插件靠 `plugin.json` 清单描述自己,解析在 `core-plugins/src/manifest.rs`。清单里 `PluginManifestPaths` 用相对路径指向插件内的 `skills`、`mcp_servers`、`apps`、`hooks` 四类资源。

这里有一道贯穿整个清单解析的**路径逃逸防御**——`resolve_manifest_path`。它的注释开宗明义:"`plugin.json` paths are required to be relative to the plugin root"。具体三重校验:

1. **必须以 `./` 开头**:不以 `./` 起头的路径直接 `warn!` 并丢弃("path must start with `./` relative to plugin root")。绝对路径、裸相对路径一律拒收。
2. **绝不含 `..`**:逐个 `Component` 检查,遇到 `Component::ParentDir` 立即拒绝("path must not contain '..'");遇到任何非 `Normal` 组件(如根、前缀)也拒("path must stay within the plugin root")。
3. **最终落在插件根内**:把规范化后的相对路径 `join` 到 `plugin_root`,转成 `AbsolutePathBuf`。

这三道合起来,确保一个插件**绝无可能通过清单里的路径读到自己目录之外的文件**——它读不到 `../../etc/passwd`,也读不到用户主目录下的别的秘密。这与第 12 章 `AGENTS.md` 绝不越过项目根、第 11 章诊断导出防路径逃逸是同一类边界纪律:**对一切来自外部、可被第三方控制的路径,默认不信任,逐组件验明正身**。

清单里还有别的约束反映"克制"的设计取向:默认提示(default prompts)最多 `MAX_DEFAULT_PROMPT_COUNT = 3` 条,每条最长 `MAX_DEFAULT_PROMPT_LEN = 128` 字符,超了就拒——**插件可以贡献能力,但不能借机往用户的 prompt 空间里灌私货**。

---

## 本章小结

- **统一抽象**:`PluginCapabilitySummary` 揭示扩展体系的核心——一个**插件**是捆绑包,同时承载**技能**(`has_skills`)、**MCP 服务器**(`mcp_server_names`)、**app 连接器**(`app_connector_ids`)与 **hooks**;四类能力共用一套安装/启停/遥测管线。
- **技能 = 带元数据的 `SKILL.md`**(`core-skills`):`SkillMetadata`(name/description/scope/plugin_id/dependencies/policy)+ 渐进式披露(列表只给名字+描述+路径,用到才完整读 `SKILL.md` 到 EOF)。
- **两道闸**:`SkillPolicy.allow_implicit_invocation`(缺省 `true`,可关成"点名才用")决定是否进可用列表;`products` 产品门控让同一技能库在不同形态 Codex 里呈现不同子集(`filter_skill_load_outcome_for_product`)。
- **技能渲染预算**(`render.rs`):默认上下文窗口 **2%**(`SKILL_METADATA_CONTEXT_WINDOW_PERCENT`),无窗口大小退回 `8_000` 字符;三档渲染(全量→逐字符分配描述→最小行截断),按 `System/Admin/Repo/User` 优先级保留,长路径用 `r0/r1` 别名表压缩(仅当更优才用),截断/省略发诚实告警;选中技能作为 `<skill>` `ContextualUserFragment` 顺装配总线进 contextual user 篮子。
- **MCP 客户端**(`rmcp-client`):四种传输(InProcess/Stdio/StreamableHttp/StreamableHttpWithOAuth),`ClientState` 三态(Connecting/Ready/Closed);stdio 经 `program_resolver` 解析后 spawn 子进程(本地或 executor 后端);HTTP 握手按 `[250, 1_000]` ms 退避重试(共 3 次);elicitation 用引用计数 + watch 暂停。
- **工具暴露阀门**(`mcp_tool_exposure.rs`):`DIRECT_MCP_TOOL_EXPOSURE_THRESHOLD = 100`——搜索工具开启且工具数 ≥ 100(或开 `ToolSearchAlwaysDeferMcpTools`)时转入延迟披露,模型按需检索而非全量预载;codex-apps 工具需连接器白名单 + enabled 才暴露。
- **命名即契约**:`mcp__{transport}__{identifier}`(stdio 用命令、streamable_http 用 URL 作标识)做去重锚点;技能可在 `SkillDependencies` 声明 MCP 依赖,缺失时检测并去重提示;MCP 工具调用复用第 9 章 guardian/审批模型(可 `PERSIST_SESSION`/`PERSIST_ALWAYS`)。
- **插件安全边界**(`manifest.rs`):`plugin.json` 的 `resolve_manifest_path` 强制路径以 `./` 开头、禁含 `..`、逐组件校验、最终须落在 plugin root 内——彻底封死路径逃逸;默认提示限 `MAX_DEFAULT_PROMPT_COUNT = 3` 条 × `MAX_DEFAULT_PROMPT_LEN = 128` 字符。

## 动手实验

1. **实验一:追踪技能的两道闸** — 阅读 `core-skills/src/model.rs` 的 `allows_implicit_invocation` 与 `SkillPolicy`。说明 `allow_implicit_invocation` 缺省时的取值,以及一条把它设成 `false` 的技能在 `build_available_skills`(`render.rs`)里会发生什么。再读 `filter_skill_load_outcome_for_product`,解释 `products` 列表为空与非空时的可见性差异。
2. **实验二:复算技能预算** — 找到 `SKILL_METADATA_CONTEXT_WINDOW_PERCENT`、`DEFAULT_SKILL_METADATA_CHAR_BUDGET` 与 `default_skill_metadata_budget`。给定上下文窗口 200_000,预算是多少 token?为什么测试里 `Some(99)` 会得到 `Tokens(1)`?再读 `render_lines_with_description_budget`,用自己的话解释"逐字符分配"为什么比"每技能固定配额"更省。
3. **实验三:画出 MCP 工具暴露的判定** — 阅读 `mcp_tool_exposure.rs` 的 `build_mcp_tool_exposure`。列出 `should_defer` 为真的两个条件,并说明 `direct_tools` 与 `deferred_tools` 为什么是"非此即彼"。`DIRECT_MCP_TOOL_EXPOSURE_THRESHOLD` 的值是多少?它和技能的 2% 预算在设计哲学上有什么共通?
4. **实验四:验证插件路径逃逸防御** — 阅读 `core-plugins/src/manifest.rs` 的 `resolve_manifest_path`。对下列四个清单路径,各自判断会被接受还是被 `warn!` 丢弃,并给出原因:`"./skills"`、`"skills"`、`"./a/../../etc"`、`"/abs/skills"`。再说明 `MAX_DEFAULT_PROMPT_COUNT`/`MAX_DEFAULT_PROMPT_LEN` 防的是什么滥用。

> **下一章预告**:扩展体系让第三方能力安全地汇入同一套工具调度与权限模型,但能力越多、链路越长,"线上到底发生了什么"就越难看清。下一章我们进入**可观测性、遥测与诊断**:Codex 如何给日志分级与脱敏、如何用 OTEL 指标量化 compaction/技能/降级这些关键路径、以及导出诊断信息时怎样守住隐私与路径安全的底线。
