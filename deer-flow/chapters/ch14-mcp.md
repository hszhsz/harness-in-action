# 第 14 章　MCP 集成：第三方能力的接入、隔离与延迟暴露

skill、子 agent、工具都能由 DeerFlow 自己定义,但一个真正开放的 super agent 还得能接入**别人写的能力**——一个跑在另一个进程里的浏览器自动化服务、一个公司内部的检索网关、一个第三方的 Slack 工具集。把这些「外部能力」统一接进来的标准通道,就是 MCP(Model Context Protocol)。这一章我们走进 `mcp/` 分区,看 DeerFlow 怎么发现并连接 MCP server、怎么把它们暴露的工具纳入第 7 章讲过的 deferred tool search(延迟暴露)、扩展配置怎么跨进程热加载,以及最关键的一条主线:它怎样在「接入第三方能力」和「不被第三方拖垮」之间反复设防——一个连不上的 server、一个崩了的 session、一个写歪的配置,都不该让主 agent 跟着倒下。

## 配置即接口:三种 transport 与一个别名兼容

MCP server 的定义在 `extensions_config.py` 的 `McpServerConfig`。一个 server 由 `type`(transport,默认 `"stdio"`)、`command`/`args`/`env`(stdio 子进程怎么启动)、`url`/`headers`/`oauth`(sse/http 远程怎么连)这几组字段描述。`build_server_params`(`client.py`)按 transport 类型组装连接参数,并且**该报的错绝不含糊**:stdio 缺 `command`、sse/http 缺 `url`、或者给了个不认识的 transport,都直接 `raise ValueError` 并带上 server 名字——配置写错在加载期就炸,而不是等到运行时连接失败才一脸茫然。

`McpServerConfig` 有一个很体贴的 `model_validator`,处理的是「规范漂移」这种现实问题:

```python
@model_validator(mode="before")
@classmethod
def _accept_transport_alias(cls, data):
    if isinstance(data, dict):
        transport = data.get("transport")
        if transport and not data.get("type"):
            data = {**data, "type": transport}
    return data
```

官方 MCP 配置 schema 用的字段名是 `transport`,而 DeerFlow 早期只认 `type`——结果一个只写了 `transport: "http"` 的远程 server 会被默默当成默认的 `stdio` 处理、连都连不上。这个 validator 让两种拼写都能工作(`type` 优先),注释把来龙去脉讲得很清楚。**对「别人按标准写、我这边历史字段不一样」这种兼容性裂缝主动补一道桥**,是接入外部生态时绕不开的功夫。注意 `McpServerConfig` 和 `McpOAuthConfig` 都开了 `model_config = ConfigDict(extra="allow")`——允许配置里出现 schema 没定义的额外字段(比如 `mcpInterceptors`),不因为多了个键就拒绝整份配置。

## 配置解析:多级回退、环境变量替换、未解析占位符兜底

`ExtensionsConfig.resolve_config_path` 的解析顺序和第 13 章 `get_skills_path` 同款周到:显式 `config_path` 参数 > `DEER_FLOW_EXTENSIONS_CONFIG_PATH` 环境变量 > 项目根下的 `extensions_config.json`(再退一步找 legacy 的 `mcp_config.json`)> monorepo 兼容的 backend/repo-root 位置 > 都找不到就返回 `None`——「扩展是可选的」,没有配置文件不是错误,而是「没有扩展」这一合法状态(`from_file` 此时返回一份空的 `ExtensionsConfig`)。

`resolve_env_variables` 递归地把配置里 `$OPENAI_API_KEY` 这样的字符串替换成环境变量值——让 server 配置不必把密钥硬写进 JSON。它有一个很克制的兜底:

```python
env_value = os.getenv(config[1:])
if env_value is None:
    # 未解析的占位符 → 存空字符串,而不是把字面 "$VAR" 传下去
    return ""
return env_value
```

环境变量没设时,返回空字符串而不是把字面量 `"$VAR"` 当成真值传给下游 MCP server——避免「一个没设的密钥被当成了密钥本身」这种诡异 bug。

## 跨进程热加载:from_file 重读 vs 单例缓存 vs mtime 失效

这一章里「不缓存陈旧配置」的主题出现了三次,各在不同层次,值得摆在一起看。

`extensions_config.py` 同时提供两套读取入口:`get_extensions_config()` 返回**缓存的单例**(`reload_extensions_config` 显式刷新、`reset_extensions_config` 清缓存),而 `ExtensionsConfig.from_file()` 每次都从磁盘**重读**。`mcp/tools.py` 的 `get_mcp_tools` 和 `tools/tools.py` 装配工具时,**刻意用 `from_file()` 而非 `get_extensions_config()`**,注释解释得很直白:

> We use ExtensionsConfig.from_file() instead of get_extensions_config() to always read the latest configuration from disk. This ensures that changes made through the Gateway API (which runs in a separate process) are immediately reflected.

这正是第 7 章 MCP 工具开关、第 11 章记忆启用状态、第 13 章 skill 启用状态反复出现的同一条边界自觉:**Gateway API 可能在另一个进程里改了配置,所以读取方不信任内存缓存,每次读最新的磁盘版本**。

第三层在 `mcp/cache.py`——MCP 工具本身(连接、发现 schema)很贵,不能每次都重连,所以用 `_mcp_tools_cache` 缓存。但缓存又不能永远陈旧,于是它用**配置文件的 mtime**做失效判据:`initialize_mcp_tools` 记下当时的 `_config_mtime`,`get_cached_mcp_tools` 每次先 `_is_cache_stale()` 比对当前 mtime,文件被改过(`current_mtime > _config_mtime`)就 `reset_mcp_tools_cache()` 重建。一句话概括这三层:**配置对象每次重读(便宜)、但工具连接按 mtime 失效后才重连(贵的才缓存)**——区分「读配置」和「建连接」的成本,各用各的策略。

`reset_mcp_tools_cache` 还顺手 `close_all_sync()` 关掉所有持久 session 再 `reset_session_pool()`——配置变了,旧连接也得按新参数重建,不能留着连旧地址的僵尸 session。

## session 池:有状态 server 的连接复用与「同任务出入」铁律

`get_mcp_tools` 把每个 MCP 工具用 `_make_session_pool_tool` 包一层,让**同一个 thread 内**对同一个 server 的连续调用复用同一个 session。为什么要复用?注释举的例子是 Playwright 这种**有状态**的 server:打开的页面、填好的表单都是 server 端状态,如果每次工具调用都新建一个 session,这些状态就全丢了。session 池(`MCPSessionPool`)按 `(server_name, thread_id)` 做 scope key 隔离——不同 thread 各有各的 session,互不串台;`MAX_SESSIONS = 256` 满了按 LRU 淘汰。

但 session 池真正的工程难点不在「复用」,而在「**怎么安全地关掉**」。`session_pool.py` 开头的长注释记录了一个很深的坑(issue #3379):MCP 的 `ClientSession` 建在 `anyio` 的 task group 上,而 anyio 强制要求「cancel scope 必须从**进入它的那个 task**里退出」。同步工具路径(`make_sync_tool_wrapper`)每次调用都用一个全新的 `asyncio.run` event loop——于是「在回答 A 调用时进入的 session,可能在回答 B 调用时被退出」,跨了 task,直接 `RuntimeError`。

DeerFlow 的解法是**给每个 session 配一个专属的「所有者 task」**(`_run_session`):这个 task 进入 context manager、把 live session 通过 `ready` future 交回给调用方,然后自己 `await close_evt.wait()` 阻塞着不走。所有关闭路径都只是 `set` 那个 `close_evt`(或在卡在 `initialize()` 时 `cancel` 这个 task),真正的 `__aexit__` 永远由所有者 task 自己执行——这样「进入」和「退出」铁定发生在同一个 task 里,满足 anyio 的要求。`get_session` 那几百行的「Phase 1 锁内决策 / Phase 2 关闭被驱逐的 / Phase 3 等就绪 / Phase 4 提升为正式条目」、以及对「并发同 key 创建」「跨 loop 的旧 session」「创建中途被取消」种种竞态的处理,本质都是在为这条「同 task 出入」的铁律兜底。

还有一个针对 transport 的精细取舍:`get_mcp_tools` **只 pool stdio session,HTTP/SSE 工具原样返回不池化**。注释点明原因(issue #3203):HTTP/SSE transport 内部用 anyio TaskGroup,无法从另一个 async task 关闭,池化它们会在清理时 `RuntimeError`。哪种该池化、哪种不该,是按它能不能安全跨 task 关闭来定的。

## OAuth:远程 server 的鉴权,缓存 + 提前刷新 + 拦截器注入

远程(http/sse)MCP server 常需要 OAuth。`oauth.py` 的 `OAuthTokenManager` 管 token 的获取/缓存/刷新:`get_authorization_header` 先看缓存里的 token 还没过期就直接用,过期了就**按 server 名加锁**(`asyncio.Lock`),锁内二次检查(double-check)后才真的去 `_fetch_token`——防止 token 过期时一堆并发调用同时打 token 端点。`_is_expiring` 还留了 `refresh_skew_seconds`(默认 60 秒)的提前量:不等真过期,提前 60 秒就刷新,避免「刚检查没过期、用的时候过期了」的临界。`_fetch_token` 走 `httpx.AsyncClient(timeout=15.0)`——给鉴权这一步设了**明确超时**,不让一个迟迟不响应的 token 端点把整个工具加载卡死。

token 怎么注入到调用里?两条路:一是 `get_initial_oauth_headers` 在**建连接/发现 schema** 阶段把 `Authorization` 头塞进 server 配置;二是 `build_oauth_tool_interceptor` 返回一个**拦截器**,在**每次工具调用**前动态取最新 token 注入头部。拦截器机制本身是可扩展的——`get_mcp_tools` 还会读配置里的 `mcpInterceptors`(一组 `pkg.module:builder_func` 路径),`resolve_variable` 反射加载、`build()` 出自定义拦截器追加进链。注意每个拦截器都包在 try/except 里:**一个自定义拦截器加载失败只 warning 跳过,不拖垮整个 MCP 工具加载**。

## 接进第 7 章:MCP 工具默认延迟暴露,不裸绑 schema

接入外部 server 带来一个真实的上下文压力:一个 server 可能暴露几十个工具,几个 server 加起来上百个,把它们的完整 JSON schema 全 `bind_tools` 给模型,光工具定义就能吃掉巨量 token。DeerFlow 的答案是第 7 章讲过的 **deferred tool search(延迟暴露)**,而 MCP 工具正是它的主要服务对象。

衔接点设计得很干净。`tools/tools.py` 加载完 MCP 工具后,只做一件事:给每个工具打一个元数据标签 `tag_mcp_tool`。这个标签的定义独占一个叶子模块 `mcp_metadata.py`,只依赖 `BaseTool`,把魔法字符串 `"deerflow_mcp"` 收在一处——写在工具加载侧(`tools.py`),读在延迟装配侧(`tool_search.py`)和 agent 构建侧(`agent.py`),三处共用一个公开谓词 `is_mcp_tool` 而不是互相 import 私有 helper。

真正的延迟装配在 `assemble_deferred_tools`,而且注释反复强调一条时序铁律:**必须在 tool-policy 过滤(skill allowlist、agent denylist 等)之后调用**——这样延迟目录里绝不会出现一个当前 agent 根本无权使用的工具。它的逻辑是:从「策略过滤后的工具列表」里挑出所有带 MCP 标签的(`is_mcp_tool`),装进一个不可变的 `DeferredToolCatalog`,生成一个 `tool_search` 工具追加给 agent,并把这些 MCP 工具的名字作为 `deferred_names` 返回。

最能体现「接第三方但不松防线」的是它的 **fail-closed**:

```python
if enabled and not deferred_setup.deferred_names and any(is_mcp_tool(t) for t in filtered_tools):
    raise RuntimeError("tool_search enabled and MCP tools survived policy filtering, but no deferred set was recovered - refusing to bind MCP schemas (fail-closed).")
```

「延迟暴露开着、确实有 MCP 工具通过了过滤、却没能恢复出延迟集合」——这种自相矛盾的状态宁可 `raise`,也绝不退化成「把 MCP 工具的完整 schema 直接绑给模型」。这个守卫被 lead、embedded client、subagent 三条 agent 构建路径**共享**,让「不裸绑外部 schema」这条保证只在一处实现、处处生效(子 agent 那条正是第 12 章提到的「deferred tool search 一致」)。

## 延迟工具怎么被「发现并启用」:目录、提示段、promotion

延迟暴露后,模型一开始只看到工具的**名字**,看不到完整 schema,也就调不动。这套「先给名、按需取 schema」的循环由三块拼成:

- **提示段**:`get_deferred_tools_prompt_section` 在系统提示里渲染一段 `<available-deferred-tools>`,只列名字(排序后)。模型由此知道「有这些工具存在」。
- **检索工具**:`build_tool_search_tool` 构造的 `tool_search` 工具,接受三种 query 形态——`select:Read,Edit`(按名精确取)、`notebook jupyter`(关键词检索,`DeferredToolCatalog.search` 对名字和描述做大小写不敏感正则匹配、名字命中权重更高)、`+slack send`(要求名字含 `slack`、再按剩余词排序)。模型用它把感兴趣的工具的完整 schema 捞出来。query 来自模型,所以 `_compile_catalog_regex` 对非法正则(比如括号不配对)**降级成字面子串匹配**而不抛错。
- **promotion(提升)**:`tool_search` 命中后返回一个 `Command`,往 graph state 写入 `promoted = {"catalog_hash": ..., "names": [...]}`——记下「这些工具已经被发现、可以调用了」。

把这套串起来运行的是第 9 章见过的 `DeferredToolFilterMiddleware`:它在 `wrap_model_call` 时把「还没 promote 的延迟工具」从 `request.tools` 里**滤掉**(模型绑定时看不到它们的 schema),又在 `wrap_tool_call` 时**拦截**对「尚未 promote 的延迟工具」的调用,返回一句 `Error: Tool '...' is deferred and has not been promoted yet. Call tool_search first...`——错误即文档,直接告诉模型该先去 `tool_search`。注意 ToolNode 那边**始终持有全部工具**(包括延迟的)用于执行路由,被隐藏的只是「绑给模型的 schema」,不是执行能力。

这里藏着一个很要紧的安全细节:`catalog_hash`。promotion 是按 catalog 的 hash 来 scope 的。`DeferredToolCatalog.hash` 把所有工具的 name + OpenAI schema 规范化后取 sha256 前 16 位——**工具目录一旦变了(server 增减、工具改名、schema 漂移),hash 就变**。`thread_state.py` 的 `merge_promoted` reducer 据此处理:catalog_hash 变了就**整体替换、丢掉旧名字**;没变才并集去重累加。注释点明了它防的是什么——「一个被持久化的裸名字,在 catalog 漂移之后,可能指向一个不同的工具」。用 hash 给 promotion 绑定一个「目录版本」,杜绝了「上次会话 promote 的名字 X,这次 X 已经是另一个工具了却仍被当成已授权」的越权风险。`DeferredToolFilterMiddleware._promoted` 也做了同样的校验:state 里的 promoted 只有 `catalog_hash` 对得上才认。

## 失败隔离:外部能力出问题,主 agent 照常活着

通读 `mcp/` 这几个文件,一个一以贯之的姿态是「**任何外部 server 的问题都不该传播成主 agent 的崩溃**」,这条线索在每一层都能看到:

- `build_servers_config`:某个 server 配置组装失败,`logger.error` 记下后**跳过这一个**,继续配置其余的——一个坏 server 不连累其它 server。
- `get_mcp_tools`:`langchain-mcp-adapters` 没装就 warning 返回 `[]`(MCP 是可选依赖,没装等于「不用 MCP」);整个加载过程包在 try/except 里,**任何异常都 `logger.error` 后返回 `[]`**——MCP 工具加载彻底失败,agent 只是「没有 MCP 工具」,而不是起不来。
- `get_cached_mcp_tools`:懒初始化里对「已有 loop 在跑」「没有 loop」等情形分别处理,失败一律 `logger.exception` 返回 `[]`。
- session 关闭:`close_all_sync` 区分「owning loop 是当前 loop / 在别的线程跑 / 空闲」三种情况各用安全策略,`run_coroutine_threadsafe(...).result(timeout=SESSION_CLOSE_TIMEOUT)`(5 秒)给跨线程关闭设上限,绝不无限等一个卡死的 session;整个关闭循环里每个错误只 `logger.debug`。
- 工具名去重(`tools/tools.py`,issue #1803):config 工具 > 内置 > MCP > ACP 的优先级去重,重名的后来者被跳过并 warning——防止一个 MCP server 注册的工具名撞了内置工具,导致模型收到「拼接/歧义的 function schema」。

这套「外部能力全程降级、绝不上抛」的处理,和前面几章的可靠性分级是同一种思路:**核心路径(agent 能跑)必须稳,增强路径(MCP 工具)出问题就降级到「没有它」**。接入第三方生态最大的风险从来不是「连不上」,而是「连不上却把自己也拖垮」——DeerFlow 把这条风险按死在每一个 `except: return []` 里。

## 本章小结

- `McpServerConfig`(`extensions_config.py`)按 transport 描述 server(stdio 的 `command`/`args`/`env`,sse/http 的 `url`/`headers`/`oauth`);`build_server_params` 对缺字段/未知 transport `raise ValueError`(配置错在加载期炸);`_accept_transport_alias` 兼容官方 `transport` 与历史 `type`(`type` 优先);`extra="allow"` 容纳额外字段。
- 配置解析多级回退(显式 path > 环境变量 > 项目根 > legacy/monorepo > None「扩展可选」);`resolve_env_variables` 递归替换 `$VAR`,未设时返回空串而非字面量。
- 三层「不缓存陈旧」:`from_file()` 每次重读磁盘(跨进程一致,对抗 Gateway 改配置)、`get_extensions_config()` 单例缓存可显式 reload、`mcp/cache.py` 按配置文件 mtime 失效后才重连(贵的连接才缓存);reset 时 `close_all_sync` + `reset_session_pool` 重建。
- session 池按 `(server_name, thread_id)` 复用、`MAX_SESSIONS=256` LRU;核心难点是 anyio「cancel scope 须同 task 出入」(#3379)——每 session 配专属所有者 task `_run_session`,关闭只 `set close_evt`/`cancel`,`__aexit__` 永远由所有者执行;只 pool stdio,HTTP/SSE 不池化(#3203)。
- OAuth(`oauth.py`):token 缓存 + 按 server 加锁 double-check 刷新 + `refresh_skew_seconds=60` 提前量 + `httpx timeout=15s`;两条注入路径(初始连接头 + 每调用拦截器);`mcpInterceptors` 反射加载自定义拦截器,失败 warning 跳过。
- 接第 7 章:MCP 工具仅打 `tag_mcp_tool`(`mcp_metadata.py` 叶子模块单一真源);`assemble_deferred_tools` 在**策略过滤后**挑出 MCP 工具建 `DeferredToolCatalog`、追加 `tool_search`;fail-closed——有 MCP 工具却没恢复出延迟集合宁可 `raise`,不裸绑 schema;lead/embedded/subagent 三路共享。
- 延迟暴露循环:`<available-deferred-tools>` 提示段只列名 → `tool_search`(select/关键词/`+` 三种 query,非法正则降级字面匹配)→ promotion 写 graph state;`DeferredToolFilterMiddleware` 滤未 promote 的 schema + 拦未 promote 的调用(错误即文档);`catalog_hash`(name+schema 的 sha256[:16])scope promotion,目录漂移则 `merge_promoted` 整体替换防越权。
- 失败隔离贯穿全程:坏 server 跳过不连累其它、缺依赖/任何异常返回 `[]`、跨线程关闭 `timeout=5s`、重名工具按优先级去重(#1803)——外部能力全程降级,绝不上抛拖垮主 agent。

## 动手实验

1. **实验一:推演 transport 别名兼容** — 阅读 `McpServerConfig._accept_transport_alias` 与 `build_server_params`。给定一份 server 配置只写了 `{"transport": "http", "url": "https://x"}`(没有 `type`)。推演:validator 后 `type` 变成什么?`build_server_params` 走哪条分支、返回什么 params?如果改成 `{"type": "stdio", "transport": "http", ...}`,最终 transport 是哪个?为什么 `type` 优先?
2. **实验二:跟踪一次配置热加载** — 通读 `mcp/cache.py` 的 `_is_cache_stale`/`get_cached_mcp_tools`/`reset_mcp_tools_cache`。假设 Gateway 在另一个进程把 `extensions_config.json` 改了(新增一个 enabled server)。推演:下一次 `get_cached_mcp_tools` 怎么察觉到变化?它会做哪几步(提示:mtime 比对、reset、session 关闭、重连)?为什么 `tools.py` 装配时要用 `from_file()` 而不是 `get_extensions_config()`?
3. **实验三:验证延迟暴露的 fail-closed** — 阅读 `assemble_deferred_tools` 与 `build_deferred_tool_setup`。假设 `tool_search` enabled、经过策略过滤后还剩 2 个带 `deerflow_mcp` 标签的工具,但由于某种原因 `deferred_setup.deferred_names` 为空。推演:`assemble_deferred_tools` 会发生什么?为什么宁可 `raise` 也不把这 2 个工具的 schema 绑给模型?再假设 enabled 但过滤后没有任何 MCP 工具,结果有何不同?
4. **实验四:核对 catalog_hash 防越权** — 阅读 `DeferredToolCatalog.hash`、`merge_promoted`(`thread_state.py`)、`DeferredToolFilterMiddleware._promoted`。假设上次会话里模型 `tool_search` promote 了工具名 `slack_send`(catalog_hash=A),这次会话管理员改了 MCP server 配置导致目录变化(catalog_hash=B)。推演:旧的 `promoted={"catalog_hash":"A","names":["slack_send"]}` 在新会话里还作数吗?`merge_promoted` 和 `_promoted` 各自怎么处理 hash 不匹配?这道设计防住了什么具体的越权场景?

> **下一章预告**:MCP、skill、子 agent、工具、沙箱、模型……到这里 DeerFlow 几乎每个子系统我们都见过它「从配置里读、按配置驱动、配置错就在加载期炸」。这些配置本身是怎么组织、加载、校验、并在层层覆盖中始终「只收紧不放宽」的?下一章我们走进配置系统,看 `AppConfig` 如何把几十个子配置拼成一棵树、Pydantic 校验器如何在加载期挡下危险组合、以及「默认即安全、覆盖只能更严」这条贯穿全书的原则在配置层是怎么落地的。
