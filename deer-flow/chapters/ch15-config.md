# 第 15 章　配置系统:一棵校验过的树,默认即安全、热加载有边界

前面十四章里,我们几乎每讲一个子系统都撞见同一句话:「从配置里读、按配置驱动、配置错就在加载期炸」。沙箱 provider 从 `sandbox.use` 读类路径、模型从 `models` 列表读、MCP server 从 `extensions_config.json` 读、skill 从 `skills` 读、子 agent、记忆、循环检测、断路器……几十个开关全挂在配置上。这一章我们终于走进 `config/` 分区本身,看这几十个子配置怎么被拼成一棵 `AppConfig` 树、Pydantic 校验器怎么在**加载期**挡下「注释掉就崩」「环境变量没设」「版本过旧」这些现实裂缝、热加载怎么区分「改完下一条消息就生效」和「必须重启进程才生效」,以及贯穿全书的那条原则——**默认即安全,危险能力默认关、出错默认拦**——在配置层是怎么一行一行落地的。

## 一棵树:AppConfig 把几十个子配置组合起来

`AppConfig`(`app_config.py`)是一个普通的 Pydantic `BaseModel`,但它的字段列表读起来像一张 DeerFlow 全功能清单:`log_level`、`token_usage`、`models`、`sandbox`、`tools`、`tool_groups`、`skills`、`skill_evolution`、`extensions`、`tool_output`、`tool_search`、`title`、`summarization`、`memory`、`agents_api`、`acp_agents`、`subagents`、`guardrails`、`circuit_breaker`、`channel_connections`、`loop_detection`、`safety_finish_reason`、`database`、`run_events`、`checkpointer`、`stream_bridge`……每个字段的类型都是另一个独立模块定义的子配置类(`ModelConfig`、`SandboxConfig`、`GuardrailsConfig`……),`AppConfig` 只是把它们**组合**成一棵树。

这种「一个根、几十棵子树、每棵子树独立定义」的结构有一个直接的工程好处:**每个子系统的配置 schema 和它的代码住在一起**(`sandbox_config.py` 挨着 sandbox、`memory_config.py` 挨着 memory),改一个子系统不必动 `AppConfig`,而 `AppConfig` 只负责声明「我有这么一棵子树、它的默认值是什么」。绝大多数子配置都给了 `default_factory`——`skills`、`memory`、`guardrails`、`subagents` 全都「不写也有一份合理的默认」,唯独 `sandbox` 是**没有默认值的必填字段**(`sandbox: SandboxConfig = Field(description=...)`,没有 `default`/`default_factory`),因为「在哪执行命令」这件事不该有隐式默认,必须用户显式声明。

注意根模型开了 `model_config = ConfigDict(extra="allow")`——和前几章 `McpServerConfig`、`SandboxConfig` 一样,**允许配置里出现 schema 没定义的额外顶层键**,不因为多了个未来字段或第三方扩展键就拒绝整份配置。

## 加载期挡下现实裂缝:三个校验/兜底各防一种「人会犯的错」

`AppConfig.from_file` 是配置的总入口。它的步骤顺序本身就是一串防御:`resolve_config_path` 找文件 → `yaml.safe_load` → `_check_config_version` 比版本 → `resolve_env_variables` 替换 `$VAR` → `_apply_database_defaults` 补持久化默认 → 单独加载 `extensions` → `model_validate` 真正校验 → 没有模型就 warning → `_apply_singleton_configs` 同步单例。这里头有三处兜底,每一处都对应一个「用户真的会踩」的坑。

**其一:把「注释光了的 section」当空列表,而不是当 `None` 崩掉。** DeerFlow 出厂的 `config.example.yaml` 里,`models:` 下面是一排注释示例(全注释掉)。用户照文档 `cp config.example.yaml config.yaml` 后第一次启动,PyYAML 会把这个「有键、无值」的 `models:` 解析成 `None`——而 `models` 字段类型是 `list`,Pydantic 会抛一个晦涩的 `Input should be a valid list`。`_coerce_null_list_sections` 这个 `field_validator(mode="before")` 专门把这种 `None` 强转成 `[]`:

```python
@field_validator("models", "tools", "tool_groups", mode="before")
@classmethod
def _coerce_null_list_sections(cls, value: Any) -> Any:
    return [] if value is None else value
```

注释把来龙去脉写得很直白:这是为了让「文档里那条 `cp` 首次启动流程」不崩。**对「按官方文档操作却第一步就报错」这种最伤人的体验主动补一道桥**,和第 14 章 `_accept_transport_alias` 补的别名兼容是同一种自觉。

**其二:环境变量没设时直接 `raise`,而不是吞掉。** `resolve_env_variables` 递归地把 `$OPENAI_API_KEY` 这样的字符串替换成环境变量值,让密钥不必硬写进 YAML:

```python
if config.startswith("$"):
    env_value = os.getenv(config[1:])
    if env_value is None:
        raise ValueError(f"Environment variable {config[1:]} not found for config value {config}")
    return env_value
```

这里有一个值得专门停下来看的**刻意的不一致**:第 14 章里 `extensions_config.py` 的同名函数遇到没设的变量是**返回空字符串**(「一个没设的可选 MCP 密钥不该把整套扩展搞崩」),而 `AppConfig` 这里是**直接 raise**。区别在于定位:`AppConfig` 承载的是主路径的核心配置(模型密钥、数据库 URL),「你引用了一个不存在的环境变量」几乎一定是部署错误,**早炸早好**;而 extensions 是可选增强,缺一个密钥应当降级而非拖垮主 agent。**同一个机制,在「核心」和「增强」两层故意采用相反的失败姿态**——这正是全书反复出现的可靠性分级。

**其三:版本过旧只 warning,不阻断。** `_check_config_version` 把用户 `config.yaml` 的 `config_version`(缺失视为 0)和向上最多五层目录里找到的 `config.example.yaml` 的版本比较,旧了就 warning「Run `make config-upgrade`」。注意它通篇用 `try/except ... return` 兜底:找不到 example、读取失败、版本字段不是数字——任何异常都**静默跳过**,绝不让「版本检查」这个辅助功能反过来阻断配置加载。版本落后是「提醒」级别的问题,不该升级成「启动失败」。

`_apply_database_defaults` 则是另一种兜底:`database` section 缺失时塞进 `CONFIG_FILE_DATABASE_DEFAULTS`(`backend: sqlite, sqlite_dir: .deer-flow/data`),用 `setdefault` 逐键补——已有的键不覆盖,只填没写的。**给「没写持久化配置」一个能用的单机默认,而不是逼用户从零配一遍。**

## 默认即安全:危险能力默认关、出错默认拦

如果把各子配置的默认值横着看一遍,会发现一条清晰的安全基线:**所有「放大权限/放大风险」的开关,默认值都是最保守的那一端。**

- `SandboxConfig.allow_host_bash` 默认 `False`,字段描述直言「Dangerous; intended only for fully trusted local environments」——宿主机直接执行 bash 这种最危险的能力,**必须用户显式打开**,默认状态下连这个 tool 都不会暴露(回顾第 8 章 `is_host_bash_allowed` 的过滤)。
- `GuardrailsConfig.fail_closed` 默认 `True`——当工具调用前置鉴权的 provider **自己出错**时,默认是**拦住这次调用**(fail-closed),而不是放行。把「鉴权器挂了」解释成「拒绝」而非「通过」,是安全系统的正确缺省。
- `DatabaseConfig.backend` 默认 `"memory"`——默认不落盘、重启即清,适合开发;要持久化必须显式选 `sqlite`/`postgres`。这条不是「安全」而是「不意外」:默认不偷偷往磁盘写数据。
- `GuardrailsConfig.enabled` 默认 `False`、`circuit_breaker` 默认 `failure_threshold=5`/`recovery_timeout_sec=60`——可选增强默认关,但断路器这种「保护性」机制默认就带着一组合理阈值在岗。

这条「默认即安全、覆盖只能更严」的原则,在配置层的体现就是:**你什么都不配,得到的是一个能跑但权限收得最紧的系统;要放开任何一道闸,都得自己在 YAML 里写明白。** 危险从不靠隐式默认溜进来。

`DatabaseConfig` 的文档注释还顺手记录了一个很扎实的并发取舍:sqlite 模式下 checkpointer 和应用数据共享同一个 `deerflow.db`,靠 **WAL 日志模式**(并发读 + 单写不互阻)加 **5 秒 busy timeout**(写竞争时等待而非立即失败)让「单文件双负载」安全。默认值不只是「保守」,还得「在保守的前提下真的能用」。

## 热加载有边界:per-run 热生效 vs startup-only 须重启

配置改了之后,什么时候生效?DeerFlow 给的答案不是「全热加载」也不是「全要重启」,而是**按字段分级**,这套边界由 `reload_boundary.py` 当**唯一真源**(issue #3144)。

先看「热」的一面。`get_app_config()` 是大量未迁移代码路径取配置的入口,它的逻辑是:

```python
runtime_override = _current_app_config.get()
if runtime_override is not None:
    return runtime_override
...
should_reload = _app_config is None or _app_config_path != resolved_path or _app_config_mtime != current_mtime
if should_reload:
    _load_and_cache_app_config(str(resolved_path))
```

三层依次让路:**ContextVar 运行时覆盖**(`push/pop_current_app_config`,给单次执行临时换一份配置,测试和多租户都靠它)优先级最高;其次是 `set_app_config` 注入的**自定义实例**(`_app_config_is_custom`,测试用,不会被磁盘覆盖);最后才是**按 mtime 自动重载**的磁盘缓存——文件改过(mtime 变了)就重新 `from_file`。Gateway 每个请求都走 `get_app_config()`,所以「per-run 字段」(模型参数、循环检测阈值、summarization 开关……)改完**下一条消息就生效**,不用重启。这和第 14 章 MCP 工具用 `from_file()` 对抗「另一个进程改了配置」是同一条边界自觉。

但有些字段**不能**这样热生效——它们在进程启动时被「快照」进了某个单例或长连接里。`STARTUP_ONLY_FIELDS` 这个 dict 把它们登记在册,而且**每个字段的「原因」不是泛泛的『需重启』,而是点名哪段代码在启动时抓走了快照**:

```python
STARTUP_ONLY_FIELDS: dict[str, str] = {
    "database": "init_engine_from_config() runs once during langgraph_runtime() startup; the SQLAlchemy engine holds the connection pool and is not rebuilt on config.yaml edits.",
    "checkpointer": "make_checkpointer() binds the persistent checkpointer once at startup...",
    "sandbox": "get_sandbox_provider() caches the provider singleton ...; a different sandbox.use class path only takes effect on next process start.",
    "log_level": "apply_logging_level() runs only during app.py startup ...",
    ...
}
```

`database`、`checkpointer`、`run_events`、`stream_bridge`、`sandbox`、`log_level` 这些**基础设施**字段——引擎连接池、持久化 checkpointer、SQLAlchemy engine、沙箱 provider 单例、日志 handler——都是「启动时建一次、之后不再按 config 重建」的,改了它们必须重启进程。注册表还收了一个**不在 `AppConfig` schema 里**的顶层 section `channels`(IM 渠道凭据由 `start_channel_service()` 在 lifespan 启动时消费一次),因为它没法在 schema 层标注,注册表就是它唯一的规范位置。

这套设计最精巧的是**让边界「自己防漂移」**。`format_field_description` 把注册表里的原因拼成标准前缀 `startup-only: ...`,再供 `AppConfig` 的 `Field(description=...)` 使用——于是 IDE 悬停在 `database` 字段上,看到的就是「为什么改它要重启 + 该重启哪个子系统」。而这个函数对**没登记的字段直接 `raise KeyError`**:

```python
reason = STARTUP_ONLY_FIELDS[field_path]   # 未注册 → KeyError
```

注释点明这是**故意的**——「silently returning a placeholder would let a typo bypass the drift coverage」。配套的 `test_reload_boundary` 双向校验:注册表里的字段必须在 schema 里用这个前缀,schema 里用了这个前缀的字段也必须在注册表里。**「哪些字段要重启」这件事只在一处声明,schema 描述、IDE 提示、漂移测试三方都从这一处派生**,杜绝了「文档说要重启、代码其实热生效」这类最坑运维的不一致。还要注意边界是**per-section 而非 per-leaf**:`is_startup_only_field("database")` 认,`"database.url"` 不认——整段基础设施配置要么一起热、要么一起冷,不在叶子级别切。

## 改了 checkpointer,顺手重建派生单例

`_apply_singleton_configs` 是 `from_file` 的最后一步,把那些「还在用全局单例、没迁移到显式传 `AppConfig`」的子系统(title、summarization、memory、agents_api、subagents、tool_search、guardrails、checkpointer、stream_bridge、acp)各自 `load_..._from_dict` 同步一遍。这里有一处关键的级联处理:

```python
previous_checkpointer_config = get_checkpointer_config()
...
if previous_checkpointer_config != config.checkpointer:
    from deerflow.runtime.checkpointer import reset_checkpointer
    from deerflow.runtime.store import reset_store
    reset_checkpointer()
    reset_store()
```

`checkpointer` 和 `store` 这两个运行时单例的 backend 是从 checkpointer 配置派生的,所以一旦检测到 checkpointer 配置**变了**,就连带把它们 reset 掉、强制下次按新配置重建——和第 14 章 `reset_mcp_tools_cache` 配置变了就 `close_all_sync` 重建连接是同一个套路:**配置是派生物的真源,真源变了,派生物必须失效。** 注释还特意说明 import 放在函数内是为了避免循环依赖(两个 provider 都 import `get_app_config`),这是配置模块作为「被所有人依赖的根」时绕不开的现实约束。

## 本章小结

- `AppConfig`(`app_config.py`)是一棵 Pydantic 树:几十个字段各是独立模块定义的子配置类(`ModelConfig`/`SandboxConfig`/`GuardrailsConfig`…),根模型 `extra="allow"` 容纳额外键;绝大多数子配置有 `default_factory`,唯 `sandbox` 是**无默认必填**(「在哪执行命令」不该有隐式默认)。
- 加载期三道兜底各防一种人会犯的错:`_coerce_null_list_sections`(`field_validator(before)`)把注释光的 `models:`(PyYAML 解析成 `None`)转 `[]`,救活 `cp config.example.yaml` 首次启动;`resolve_env_variables` 对未设的 `$VAR` **直接 raise**(与 extensions 返回空串相反——核心早炸、增强降级);`_check_config_version` 版本旧只 warning 且通篇 `try/except` 静默兜底。`_apply_database_defaults` 用 `setdefault` 补单机持久化默认。
- 默认即安全:`allow_host_bash` 默认 `False`(Dangerous,须显式开)、`guardrails.fail_closed` 默认 `True`(鉴权器出错则拦)、`database.backend` 默认 `memory`(不偷偷落盘)、`guardrails.enabled` 默认 `False`、`circuit_breaker` 默认 `5`/`60s`——危险能力默认关、出错默认拦,放闸都得在 YAML 里写明。
- 热加载分级:`get_app_config()` 三层让路(ContextVar 运行时覆盖 > `set_app_config` 自定义实例 > 按 mtime 自动重载磁盘),per-run 字段改完下一条消息生效;`reload_boundary.py` 的 `STARTUP_ONLY_FIELDS` 登记必须重启的基础设施字段(database/checkpointer/run_events/stream_bridge/sandbox/log_level + 非 schema 的 channels),原因点名启动时哪段代码抓快照;`format_field_description` 对未注册字段 `raise KeyError`(故意,防 typo 绕过漂移测试),schema 描述/IDE 提示/`test_reload_boundary` 三方同源;边界 per-section 非 per-leaf。
- `_apply_singleton_configs` 同步各全局单例,并在 checkpointer 配置变化时 `reset_checkpointer()`+`reset_store()` 重建派生单例——配置是真源,真源变则派生物失效。

## 动手实验

1. **实验一:复现「注释光了就崩」并验证兜底** — 阅读 `_coerce_null_list_sections` 与 `from_file` 的步骤顺序。设想一份 `config.yaml` 只写了 `models:`(下面全是注释)。推演:PyYAML 把 `models` 解析成什么?若没有这个 validator,`model_validate` 会抛什么错?validator(`mode="before"`)在校验链的哪一步介入、把值改成什么?为什么它只管 `models`/`tools`/`tool_groups` 三个字段?
2. **实验二:对比两个 `resolve_env_variables` 的失败姿态** — 并排阅读 `app_config.py` 和(第 14 章读过的)`extensions_config.py` 里的 `resolve_env_variables`。给定配置值 `$MISSING_VAR`(环境变量未设)。推演两者各返回/抛出什么?为什么核心配置选择 `raise`、扩展配置选择返回 `""`?把两者的姿态对调会各自引发什么后果?
3. **实验三:跟踪一次 per-run 热加载** — 通读 `get_app_config()` 与 `_load_and_cache_app_config`。假设进程已缓存了配置,运维把 `config.yaml` 里 `loop_detection` 的阈值改大(文件 mtime 变化)。推演:下一次 `get_app_config()` 怎么察觉变化?`should_reload` 的三个条件哪个为真?若同一时刻有一个 `push_current_app_config` 的运行时覆盖在生效,返回的会是磁盘新值还是覆盖值?为什么?
4. **实验四:核对 startup-only 边界与漂移防护** — 阅读 `reload_boundary.py` 的 `STARTUP_ONLY_FIELDS`、`format_field_description`、`is_startup_only_field`,以及 `AppConfig` 里 `sandbox`/`database` 字段的 `Field(description=...)`。推演:把 `sandbox.use` 从 Local 改成 Aio 后,不重启进程会生效吗?为什么(提示:`get_sandbox_provider()` 的单例)?如果有人新增一个 startup-only 字段却忘了登记进注册表,`format_field_description` 和 `test_reload_boundary` 分别会怎样把这个疏漏暴露出来?为什么边界是 per-section 而非 per-leaf?

> **下一章预告**:配置把「持久化用哪个 backend」「checkpointer 怎么建」声明清楚之后,真正把一次次运行**存下来、可回放、可观测**的是另一套机制。下一章我们走进持久化与可观测性层,看 LangGraph checkpointer 怎么把 graph state 落盘、run-event store 怎么记录一次运行的事件流、以及遥测/日志在「记得够多」和「不泄露敏感数据」之间是怎么用脱敏和分级把住关口的——又一次「核心必稳、增强可降级」的可靠性分级在持久化层的落地。
