# 第 7 章　工具系统：注册、分组与 deferred tool search

模型能调用了，但它「能调什么」是 harness 的另一项核心权力。一个 super agent 动辄挂着几十上百个工具——内置的文件读写、沙箱命令、子 agent，加上一堆 MCP server 暴露的工具。但把上百个工具的完整 schema 全塞进模型的上下文，既烧 token 又让模型挑花眼。这一章我们走进 `tools/` 分区，看 DeerFlow 如何注册工具、如何按能力裁剪、以及它怎么用 **deferred tool search（延迟工具发现）** 把「工具太多」这个问题优雅地化解掉。

## 工具也是配置驱动的

和模型一样，工具的「用哪个实现」也外包给配置。`get_available_tools`（`tools/tools.py`）的核心同样是一行 `resolve_variable`：

```python
loaded_tools_raw = [(cfg, resolve_variable(cfg.use, BaseTool)) for cfg in tool_configs]
```

`config.tools` 里每个条目有 `use`（`模块:变量` 路径）、`name`、`group` 字段。`resolve_variable` 把 `use` 解析成一个 `BaseTool` 实例。这意味着 `community/` 下那 12 个搜索/抓取 provider（`tavily`、`serper`、`searxng`、`jina_ai`、`exa`、`brave`、`firecrawl`、`browserless`、`ddg_search`、`image_search`、`infoquest`、`aio_sandbox`）都不是硬编码进框架的——它们是配置里指向某个 `BaseTool` 的一行 `use`。这与第 6 章模型层的 provider 抽象、第 2 章「不绑死单一服务」的 super agent 定位一脉相承。

## 名字不一致：一个 issue 喂出来的告警

工具有两个「名字」：配置里写的 `cfg.name`，和工具对象自己的 `.name` 属性。两者本该一致，但用户配错时会出问题。`get_available_tools` 专门检查并告警：

```python
for cfg, loaded in loaded_tools_raw:
    if cfg.name != loaded.name:
        logger.warning("Tool name mismatch: config name %r does not match tool .name %r ...")
```

注释把根因写得很清楚：这是 issue #1803 的病灶——LLM 在它的 tool schema 里收到一个名字，但运行时路由器认的是另一个名字，结果模型调用时报「not a valid tool」。DeerFlow 的处置是「以工具对象自己的 `.name` 为准做绑定」，并把不一致告警出来。又一次「错误即文档」：它不默默用一个，而是把潜在的坑喊给你听。

同一个 issue #1803 还催生了末尾的**按名去重**逻辑：把 config 工具、内置工具、MCP 工具、ACP 工具拼成一个大列表后，按 `name` 去重（config 工具优先），因为重复的工具名会让 LLM 收到「歧义的或被拼接的」函数 schema。去重时遇到撞名也会告警，提示去查 `config.yaml` 和 MCP server 注册。

## 按能力裁剪：工具不是有什么挂什么

`get_available_tools` 不是无脑把所有工具都给模型，而是按多个维度条件裁剪——这是 harness「约束模型能力」的直接体现：

- **按 group 过滤**：`groups` 参数可只取某些分组的工具。
- **host bash 默认禁**：`if not is_host_bash_allowed(config)` 时，把所有 host-bash 执行面（`group == "bash"` 或 `use == "deerflow.sandbox.tools:bash_tool"`）从列表里剔掉。这是第 8 章「fail-closed host bash」的入口——当用的是不安全的本地沙箱时，宿主机 bash 默认不暴露给模型。
- **vision 工具按模型能力挂**：只有 `model_config.supports_vision` 为真才追加 `view_image_tool`——和第 5 章 `ViewImageMiddleware` 的守卫遥相呼应，能力裁剪在中间件层和工具层都做了一遍。
- **subagent 工具按开关挂**：只有 `subagent_enabled` 为真才追加 `task_tool`。
- **skill 演化工具按配置挂**：`skill_evolution.enabled` 为真才追加 `skill_manage_tool`。
- **MCP / ACP 工具按配置挂**：有启用的 MCP server 才去取缓存的 MCP 工具，有配置的 ACP agent 才挂 `invoke_acp_agent` 工具。

值得一提的是 MCP 工具读取用的是 `ExtensionsConfig.from_file()` 而非缓存的 `config.extensions`——注释解释这是为了「总是从磁盘读最新配置」，因为 Gateway API 在另一个进程里改了配置，要能立刻反映出来。这是对「配置可能被另一个进程改动」这一边界的清醒认知。

还有个小而稳健的细节 `_ensure_sync_invocable_tool`：给那些「只有 `coroutine` 没有 `func`」的异步工具补一个同步包装器，让同步的 agent 调用方也能用。harness 要同时伺候同步和异步两条执行路径，这种适配处处都是。

## deferred tool search：工具太多就先只给名字

真正体现工程巧思的是 `tools/builtins/tool_search.py`。问题很现实：MCP server 可能暴露几十上百个工具，把它们的完整 JSON schema 全绑给模型，既炸上下文又拖慢推理。DeerFlow 的解法是**延迟**：模型一开始只看到这些工具的**名字**，需要某个工具时，先用一个叫 `tool_search` 的元工具去「捞」出它的完整 schema，捞到之后它才变得可调用。

机制的关键词是「source-agnostic」——一个工具是否「可延迟」，只看它有没有带 `deerflow_mcp` 元数据标签（`tag_mcp_tool` 写、`is_mcp_tool` 读，定义在 `mcp_metadata.py` 这个零依赖的叶子模块里，保证任何模块都能 import 它而不产生循环依赖）。也就是说，延迟机制天然只作用于 MCP 来源的工具，内置工具始终是直接可调的。

核心数据结构 `DeferredToolCatalog` 是一个 `frozen=True` 的 dataclass，持有一组延迟工具，提供纯搜索、不可变。它的 `search` 支持三种查询语法，直接写在 `tool_search` 工具的 docstring 里教模型用：

```
- "select:Read,Edit"  -- 按确切名字取这几个工具
- "notebook jupyter"  -- 关键词搜索，返回最多 5 个最佳匹配
- "+slack send"       -- 要求名字含 "slack"，再按剩余词排序
```

`MAX_RESULTS = 5` 限制每次搜索最多返回 5 个工具——避免一次捞回太多又把上下文撑爆。还有一处防御性细节：搜索词来自模型，可能是非法正则（比如括号不配对），`_compile_catalog_regex` 捕获 `re.error` 后**退化成字面子串匹配**而不是抛错——「模型给的输入一定要能容错」是边界自觉的又一例。

那个 `frozen=True` 的 dataclass 上还有一条**警告级注释**值得记：「frozen=True without slots=True keeps `__dict__`」——正是 `__dict__` 让 `@cached_property`（`names`、`hash`）能缓存（它们写进实例的 `__dict__`，绕过 frozen 的 `__setattr__`）。注释明令「不要加 slots=True，否则 hash/names 在运行时会坏掉」。这是把一个微妙的语言特性陷阱钉死在代码注释里，防止后来者好心办坏事。

## promotion：被捞出的工具记在 per-thread 状态里

模型调 `tool_search` 捞到工具后，这些工具怎么从「只有名字」变成「真能调」？答案藏在 `build_tool_search_tool` 返回的 `Command` 里：

```python
return Command(update={
    "promoted": {"catalog_hash": catalog_hash, "names": names},
    "messages": [ToolMessage(content=content, ...)],
})
```

它往 graph state 的 `promoted` 字段写入「这个 catalog（用 hash 标识）下，这几个名字被提升（promoted）了」。这正好接上第 2 章讲过的 `merge_promoted` reducer——它按 `catalog_hash` 分桶，当工具目录的哈希变了，旧的 promoted 名字会被整体丢弃，防止「一个持久化的裸名字在目录漂移后指向了另一个工具」。`catalog.hash` 用工具名+schema 算 SHA256 取前 16 位，正是目录的「指纹」。整套机制**不用 ContextVar**，提升状态完全活在 per-thread 的 graph state 里——干净、可序列化、随线程隔离。

## fail-closed：宁可报错也不裸绑 MCP schema

`assemble_deferred_tools` 把这套机制收束成一个所有 agent 构建路径（lead、嵌入式 client、subagent）共用的入口，并在这里钉了一条 fail-closed 不变量：

```python
if enabled and not deferred_setup.deferred_names and any(is_mcp_tool(t) for t in filtered_tools):
    raise RuntimeError("tool_search enabled and MCP tools survived policy filtering, but no deferred set was recovered - refusing to bind MCP schemas (fail-closed).")
```

意思是：如果延迟开着、且确实有 MCP 工具通过了策略过滤，但延迟集合却没建起来——这是个矛盾状态，与其「默默把这些 MCP 工具的完整 schema 裸绑给模型」（既浪费上下文，也可能暴露不该暴露的工具），不如直接抛错。注释强调这个保证「从一个地方」统一供给所有 agent 路径。这又是「默认即关闭」原则的现身：出现说不通的状态时，炸，别蒙混。

还有一个时序约束反复出现在注释里：`build_deferred_tool_setup` / `assemble_deferred_tools` **必须在工具策略过滤之后**调用——这样延迟目录就绝不会暴露一个当前 agent 无权使用的工具。catalog 的内容是「策略过滤后的结果」，提升只能在这个安全子集里发生。

## 提示里的 `<available-deferred-tools>`

模型怎么知道有哪些延迟工具？`get_deferred_tools_prompt_section` 把延迟工具的名字（仅名字，不含 schema）渲染成系统提示里的一段：

```
<available-deferred-tools>
{names}
</available-deferred-tools>
```

只列名字，让模型知道「存在这些工具」，需要时用 `tool_search` 去加载。这个函数刻意和「产生 deferred_names 的装配逻辑」放在同一个文件，让每条 agent 构建路径都用同样的方式渲染这一段，而不必反向耦合到 `lead_agent.prompt`。一句话总结整个机制:**让模型先看见目录、用时再翻页**——这是 super agent 在「工具规模」与「上下文成本」之间找到的平衡点。

## 本章小结

- 工具与模型一样配置驱动：`get_available_tools` 用 `resolve_variable(cfg.use, BaseTool)` 把配置里一行 `use` 解析成工具实例，`community/` 下 12 个搜索/抓取 provider 都是可插拔配置项。
- config 的 `name` 与工具对象 `.name` 不一致会告警（issue #1803 根因），绑定以工具自身 `.name` 为准；最终按名去重以防 LLM 收到歧义 schema。
- 工具按多维度条件裁剪：group 过滤、host bash 默认禁（`is_host_bash_allowed`）、vision 工具按 `supports_vision` 挂、subagent/skill_evolution/MCP/ACP 工具按开关挂；MCP 配置用 `ExtensionsConfig.from_file()` 读磁盘最新值（跨进程一致性）。
- deferred tool search 让模型先只看到 MCP 工具的名字（`<available-deferred-tools>`），用 `tool_search` 元工具按 `select:` / 关键词 / `+required` 三种语法捞回完整 schema 后才可调用；`MAX_RESULTS=5`，非法正则退化为字面匹配。
- 「是否可延迟」只看 `deerflow_mcp` 元数据标签（`mcp_metadata.py` 零依赖叶子模块），机制 source-agnostic；`DeferredToolCatalog` 用 `frozen=True` 且刻意不加 `slots=True` 以保住 `@cached_property` 的缓存能力。
- 提升（promotion）写入 graph state 的 `promoted` 字段，由 `merge_promoted` reducer 按 `catalog_hash`（工具名+schema 的 SHA256 前 16 位）分桶；不用 ContextVar。
- `assemble_deferred_tools` 是所有 agent 路径共用的入口，钉了 fail-closed 不变量：延迟开着却没建起延迟集合时宁可 `raise` 也不裸绑 MCP schema；且必须在工具策略过滤之后调用。

## 动手实验

1. **实验一：追踪一个工具的加载** — 打开 `tools/tools.py` 的 `get_available_tools`，按代码顺序列出工具被加入最终列表的所有来源（config 工具、BUILTIN_TOOLS、subagent、skill_evolution、vision、MCP、ACP），并标注每个来源的「开启条件」。哪些是无条件加入的？
2. **实验二：手推三种搜索语法** — 阅读 `DeferredToolCatalog.search`，对一个假想的延迟工具集（含 `slack_send`、`slack_read`、`notion_create`）分别推演 `select:slack_send,notion_create`、`notebook`、`+slack send` 三种查询各返回什么、按什么排序。
3. **实验三：验证 fail-closed 守卫** — 阅读 `assemble_deferred_tools` 的 `raise RuntimeError(...)` 分支，构造一个会触发它的场景（提示：`enabled=True` + 有 MCP 工具通过过滤 + deferred_names 为空）。再思考：如果去掉这个守卫，模型会多看到什么、坏处是什么？
4. **实验四：理解 promotion 的目录漂移防护** — 结合本章的 `catalog.hash` 与第 2 章的 `merge_promoted` reducer，回答：当 MCP server 增删了一个工具导致 catalog hash 变化时，之前 promoted 的工具名会怎样？为什么这种「按 hash 分桶丢弃」能防止裸名字指向错误的工具？

> **下一章预告**：工具里最危险的一类是「执行命令、读写文件」。下一章我们钻进 `sandbox/` 分区，看 DeerFlow 如何用 `LocalSandbox` 与 `AioSandbox` 两种隔离策略约束副作用，为什么当部署用「不安全的本地沙箱」时它默认禁掉 host bash、又留了哪个显式开关，以及那些把文件操作钉死在虚拟路径里的防御。
