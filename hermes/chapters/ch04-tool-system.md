# 第 4 章　工具系统：注册与分发

到目前为止，我们解构的都是 Agent 的"内部循环"——怎么转、怎么压缩上下文。但 Agent 真正改变世界的方式，是**调用工具**：读文件、跑命令、搜网页、生成图片、委派子任务。一个 Agent 的能力边界，几乎就等于它工具集的边界。所以工具系统的工程质量，直接决定了这个 Agent 是个玩具还是个生产力工具。

Hermes 的工具系统要回答几个尖锐的问题：几十上百个工具如何注册而不互相打架？模型吐出一个工具名，怎么安全地翻译成真实的函数调用？不同场景（科研、开发、纯终端、不受信的 webhook）如何启用不同的工具子集？这一章我们就从注册、分发、分组三个层面把它读透。

## 4.1 导入即注册：去中心化的注册模型

Hermes 的工具注册采用一种"去中心化"的模式：**每个工具模块在被导入时，自己调用 `registry.register(...)` 把自己登记进去**。这意味着没有一个集中的"工具清单"文件需要手动维护——你写一个新工具模块、在里面调一次 register，它就自动成为系统的一部分。

但这种模式有个陷阱：如果在启动时 eager 地导入所有工具模块，冷启动会很慢、还会把一堆可能用不到的依赖都拉起来。Hermes 的 `tools/__init__.py` 因此**刻意不 eager 导入**任何具体工具，而是由调用方按需导入具体子模块。

为了在"去中心化注册"和"按需加载"之间架桥，`tools/registry.py` 提供了 `discover_builtin_tools()`（约第 57 行）。它的做法很巧妙：用 **AST 静态扫描**各模块，检测模块顶层是否存在一个 `registry.register(...)` 调用（靠 `_is_registry_register_call`、`_module_registers_tools` 这两个辅助函数判断），**只导入那些确实会注册工具的模块**。这样既保留了"模块自注册"的便利，又避免了"为了发现工具而导入一堆无关模块"的开销。用静态分析代替运行时导入来做发现，是一个很漂亮的工程取舍。

## 4.2 `ToolRegistry`：单例、锁与 generation 计数器

注册表本身是 `class ToolRegistry`（约第 151 行），一个全局单例。它的内部状态揭示了它要应对的复杂性：`_tools: Dict[str, ToolEntry]` 是工具名到工具条目的映射；`_toolset_checks`、`_toolset_aliases` 管理工具集层面的检查与别名；一把 `threading.RLock` 保证并发安全（Agent 会并发执行多个工具，注册表必须线程安全）；还有一个单调递增的 `_generation` 计数器，**每次发生变更就 +1**。

这个 generation 计数器是个很实用的设计。当工具集合发生动态变化（比如加载了新插件、启用了某个 MCP server）时，依赖工具列表的缓存就需要失效。有了 generation，下游只要比较"我缓存时记的 generation"和"现在的 generation"是否一致，就能 O(1) 地判断"工具表变没变"，而不必去逐一 diff 整张表。

每个工具用一个 `ToolEntry`（约第 77 行，用 `__slots__` 省内存）描述，字段包括：`name`、`toolset`（所属工具集）、`schema`（给模型看的 JSON Schema）、`handler`（真正的处理函数）、`check_fn`（可用性检查）、`requires_env`（依赖的环境变量）、`is_async`（是否异步）、`description`、`emoji`、`max_result_size_chars`（结果大小上限）、`dynamic_schema_overrides`（动态 schema 覆盖）。`__slots__` 的使用是个小信号：当一个对象会被大量实例化时（工具可能有上百个），用 `__slots__` 砍掉每实例的 `__dict__` 能省下可观的内存。

## 4.3 注册的防冲突规则

`register(...)`（约第 234 行）的签名很长，但它最值得关注的是一条**防冲突规则**：如果一次注册会覆盖掉一个来自**不同工具集**的已有同名工具，注册会被拒绝，并打印日志 `Tool registration REJECTED`——除非显式传 `override=True`。

这条规则防的是什么？想象插件 A 注册了一个叫 `search` 的工具，插件 B 也想注册 `search`。如果允许后者静默覆盖前者，用户调用 `search` 时会得到无法预期的行为，且毫无察觉。Hermes 的选择是：**默认拒绝跨工具集的同名覆盖，强迫冲突显式化**——要么改名，要么明确写 `override=True` 表示"我知道我在覆盖"。

唯一的例外是 MCP 工具之间的覆盖：当新旧工具的 toolset 都 `startswith("mcp-")` 时，允许覆盖。这是因为 MCP server 可能动态重连、重新注册它的工具集，这种"同源刷新"是正常行为，不该被当成冲突。

`check_fn` 的结果会被 TTL 缓存：`_CHECK_FN_TTL_SECONDS = 30.0`，通过 `_check_fn_cached` 实现（检查时抛异常一律当 False 处理，即"不可用"），`invalidate_check_fn_cache()` 可手动清缓存。为什么要缓存可用性检查？因为 `check_fn` 可能很贵（比如探测某个外部服务是否在线），而工具列表会被频繁查询，30 秒缓存能避免重复探测。注意"异常即 False"的处理——**检查工具是否可用时，任何异常都保守地判定为不可用**，这是典型的 fail-closed 心态。

## 4.4 分发：从工具名到执行结果

模型生成的工具调用，最终通过 `dispatch(name, args, **kwargs) -> str`（约第 390 行）落地执行。这个函数的每一处错误处理都体现了"对模型输出不信任"的边界思维：

- 未知工具名 → 不抛异常崩溃，而是返回 `json.dumps({"error": f"Unknown tool: {name}"})`。模型可能幻觉出一个不存在的工具名，dispatch 必须优雅地把这个错误**作为数据返回给模型**，让模型有机会自我纠正。
- 异步 handler → 通过 `model_tools._run_async` 桥接到同步上下文执行。
- **所有异常都被 catch**，格式化成 `"Tool execution failed: {type}: {e}"`，再经 `model_tools._sanitize_tool_error` 净化（去掉可能泄露的敏感信息），最后同样以 `{"error": ...}` 的形式返回。

这套"把一切失败都转成结构化 error 返回"的设计至关重要：**工具执行失败绝不能让 Agent 主循环崩溃**，而应该变成一条模型能读懂、能据此调整下一步的反馈。一个工具炸了，Agent 应该能说"这个工具失败了，我换个方法"，而不是整个进程退出。

结果大小通过 `get_max_result_size(name, default=None)` 控制，缺省回退到 `tools.budget_config.DEFAULT_RESULT_SIZE_CHARS`。这道闸防止某个工具返回海量输出（比如读了一个巨大的文件）直接撑爆上下文——和第 3 章的压缩是同一类"上下文卫生"措施，只不过这里是在源头限制。

在主循环层面，工具调用的实际执行由 `agent/tool_executor.py` 承担，它提供两种模式：`execute_tool_calls_concurrent(...)`（约第 243 行，并发执行）和 `execute_tool_calls_sequential(...)`（约第 770 行，顺序执行）。并发能让多个独立工具调用同时进行、缩短回合时延；顺序则用于有依赖或需要确定性的场景。两个辅助函数 `tool_error(message, **extra)`（约第 563 行）与 `tool_result(data=None, **kwargs)`（约第 577 行）统一了工具返回值的结构。

## 4.5 工具集分组：`toolsets.py` 的能力编排

有了几十个工具，下一个问题是怎么管理它们的"启用组合"。`toolsets.py` 负责把零散的工具编排成有意义的分组。

最基础的是 `_HERMES_CORE_TOOLS`（约第 31 行）——一份规范的全量工具清单，囊括了 `web_search`、`web_extract`、`terminal`、`process`、`read_terminal`、`read_file`、`write_file`、`patch`、`search_files`、`vision_analyze`、`image_generate`、`skills_list` / `skill_view` / `skill_manage`、`browser_*` 系列、`text_to_speech`、`todo`、`memory`、`session_search`、`clarify`、`execute_code`、`delegate_task`、`cronjob`、`send_message`、`kanban_*`、`computer_use` 等。把这份清单读一遍，你就掌握了 Hermes 的完整能力图谱：它能上网、能操作文件、能跑终端、能看图生图、能管理自己的技能与记忆、能搜索历史会话、能委派子任务、能定时、能跨平台发消息、甚至能直接操作计算机界面。

与之形成鲜明对比的是 `_HERMES_WEBHOOK_SAFE_TOOLS`（约第 81 行），它**故意只有四个**：`["web_search", "web_extract", "vision_analyze", "clarify"]`。这是为"处理来自不受信来源（webhook）的输入"专门准备的极简、只读、无副作用的工具集。当输入可能被攻击者构造时，Agent 能动用的工具被收窄到"只能查、不能改"——查网页、抽内容、看图、追问澄清，仅此而已。这是一个教科书级别的**最小权限**实践：**信任级别越低，能力面越窄**。

`TOOLSETS = {...}`（约第 91 行）则把工具组织成命名分组，每组形如 `{"description", "tools": [...], "includes": [...]}`，例如 `web` → `[web_search, web_extract]`、`terminal` → `[terminal, process]`、`file` → `[read_file, write_file, patch, search_files]`、`delegation` → `[delegate_task]`、`code_execution` → `[execute_code]`。`includes` 字段支持组合复用——一个工具集可以"包含"另一个，避免重复列举。

## 4.6 分布预设：`toolset_distributions.py`

再往上一层是 `toolset_distributions.py` 的 `DISTRIBUTIONS = {...}`（约第 29 行），它把工具集进一步打包成面向场景的预设，每个预设形如 `{"description", "toolsets": {...}}`。预设名直接表达了使用场景：`default`、`image_gen`、`research`、`science`、`development`、`safe`、`balanced`、`minimal`、`terminal_only`、`terminal_web`、`creative`、`reasoning`、`browser_use`、`browser_only`、`browser_tasks`、`terminal_tasks`、`mixed_tasks`。

配套 API 有 `get_distribution(name)`、`list_distributions()`，以及一个有趣的 `sample_toolsets_from_distribution(...)`——**从一个分布中采样工具集**。这个 "sample" 字眼暴露了 Hermes 的研究基因：它不仅是个产品，还是个生成训练数据的平台。通过从不同分布里随机采样工具组合来跑 Agent、收集轨迹，可以为训练下一代工具调用模型生成多样化的数据。这条线索我们在第 2 章的 `TrajectoryCompressor` 见过，这里又一次浮现。

把四层结构连起来看——单个工具（`ToolEntry`）→ 工具集（`TOOLSETS`）→ 分布预设（`DISTRIBUTIONS`）→ 采样——Hermes 的工具系统呈现出清晰的层次：底层是去中心化注册的原子工具，往上是按功能分组，再往上是按场景预设，最顶层还能为研究目的做随机采样。这种分层让"给这次任务启用哪些工具"既可以精确指定，也可以按场景一键切换。

## 本章小结

- 工具采用"导入即注册"的去中心化模型：每个模块自调用 `registry.register(...)`；`tools/__init__.py` 刻意不 eager 导入，`discover_builtin_tools()` 用 AST 静态扫描只导入确实会注册工具的模块，兼顾便利与冷启动。
- `ToolRegistry` 单例用 `threading.RLock` 保证并发安全，用单调递增的 `_generation` 计数器让下游 O(1) 判断"工具表是否变化"以失效缓存。
- `ToolEntry` 用 `__slots__` 省内存，字段涵盖 `schema`/`handler`/`check_fn`/`requires_env`/`max_result_size_chars` 等完整契约。
- `register` 默认拒绝跨工具集的同名覆盖（打印 `Tool registration REJECTED`），强迫冲突显式化，仅 `override=True` 或 MCP-到-MCP 同源刷新例外。
- `check_fn` 结果 TTL 缓存 30 秒（`_CHECK_FN_TTL_SECONDS = 30.0`），且"检查抛异常一律当不可用"，体现 fail-closed。
- `dispatch` 把一切失败都转成结构化 `{"error": ...}` 返回（未知工具、执行异常均如此），经 `_sanitize_tool_error` 净化，确保工具失败不会崩溃主循环；`tool_executor.py` 提供并发与顺序两种执行模式。
- `toolsets.py` 用 `_HERMES_CORE_TOOLS` 定义全量能力，用故意只含四个只读工具的 `_HERMES_WEBHOOK_SAFE_TOOLS` 践行"信任越低、能力面越窄"的最小权限。
- 四层结构（工具 → 工具集 → 分布预设 → 采样）让能力可精确指定也可按场景切换；`sample_toolsets_from_distribution` 暴露了 Hermes 生成训练轨迹的研究基因。

## 动手实验

1. **实验一：追一个工具的注册全过程** —— 以 `tools/file_tools.py` 为例，用 Grep 找到它末尾的 `registry.register(...)` 调用（约第 1583–1586 行），确认 `read_file` / `write_file` / `patch` / `search_files` 都注册在 `"file"` 工具集下、共享 `check_fn=_check_file_reqs`、`max_result_size_chars=100_000`。再回到 `discover_builtin_tools` 解释这个模块是如何被发现并导入的。

2. **实验二：制造一次注册冲突** —— `Read` `register(...)`（约第 234 行）的防冲突分支。设想你写了一个插件，注册一个名为 `terminal` 但 toolset 不同的工具，推演会发生什么、日志会打印什么。再思考 MCP-到-MCP 例外为什么是合理的。

3. **实验三：验证失败的优雅降级** —— `Read` `dispatch`（约第 390 行）的异常处理路径。构造三种失败：调用不存在的工具、工具 handler 抛异常、异步工具。分别说明每种情况下 dispatch 返回什么结构，以及为什么"返回 error 数据"比"抛异常"更适合 Agent 场景。

4. **实验四：对比两个工具集的权限差** —— 把 `_HERMES_CORE_TOOLS` 和 `_HERMES_WEBHOOK_SAFE_TOOLS` 两份清单并排列出，标出后者剔除了哪些有副作用的工具（如 `terminal`、`write_file`、`execute_code`、`delegate_task`）。论证：为什么处理 webhook 输入时必须把这些工具拿掉？这和第 5 章的命令审批是同一思路的不同层次吗？

> **下一章预告**：工具能被调用了，但有些工具——尤其是终端命令——危险得多。`rm -rf /`、`mkfs`、`curl | sh` 该怎么拦？第 5 章将解剖 `tools/approval.py` 的两级阻断名单（`HARDLINE_PATTERNS` 与 `DANGEROUS_PATTERNS`）、YOLO 模式的边界、以及 `agent/file_safety.py` 如何守住那些绝不能被写入的控制面文件。
