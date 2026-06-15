# 第 10 章　防失控：循环检测、工具输出预算与安全终止

上一章我们让上下文「太长」时能压下去。但 super agent 还有另外几种「失控」姿态：模型把同一个工具翻来覆去调二十遍（陷进死循环）、一次工具调用吐出一坨几十万字符的输出把窗口瞬间撑爆、或者模型供应商出于安全原因半路掐断了响应、却还留着半截畸形的 `tool_calls` 等着被执行。这三类失控各有各的引爆点，DeerFlow 用三个独立的中间件分别拦它们：`LoopDetectionMiddleware`、`ToolOutputBudgetMiddleware`、`SafetyFinishReasonMiddleware`。这一章把它们摆在一起看——它们共享同一套「改写消息、剥掉 tool_calls」的机制，却各有不同的触发条件，而且都在「该不该出手」上极其克制，生怕误杀正常行为。

## 循环检测：两层探测，警告在前、强停在后

`LoopDetectionMiddleware`（`agents/middlewares/loop_detection_middleware.py`，600 多行）是这一章里最复杂的一个。它的配置 `LoopDetectionConfig`（`config/loop_detection_config.py`）默认就是开的（`enabled=True`，呼应第 5 章「防失控能力默认在」），核心阈值是四组数：

```python
warn_threshold: int = 3       # 攒够 3 次相同调用集，注入警告
hard_limit: int = 5           # 攒够 5 次，强制停
window_size: int = 20         # 每线程只追踪最近 20 个调用集
max_tracked_threads: int = 100  # 内存里最多保留 100 个线程的历史
```

它的探测分**两层**，各管一种「重复」：

- **哈希层**：检测「一模一样的调用集」。`_hash_tool_calls` 把这一轮所有 tool_call 的「稳定 key」排序后算 md5 取前 12 位——排序让它对调用顺序不敏感（先调 A 后调 B、和先 B 后 A，算出同一个哈希）。同一个哈希攒够 `warn_threshold` 次就警告，攒够 `hard_limit` 次就强停。
- **频次层**：检测「同一类工具被调太多次」，哪怕参数每次都不同。默认 `tool_freq_warn=30`、`tool_freq_hard_limit=50`——同一个工具名调用 30 次警告、50 次强停。

频次层还有一个很体贴的逃生口 `tool_freq_overrides`：

```python
class ToolFreqOverride(BaseModel):
    warn: int = Field(ge=1)
    hard_limit: int = Field(ge=1)
```

注释举的例子是 RNA-seq 流水线这种「批处理工作流」——它合法地就要调 bash 几百次。给 bash 配一个更高的 override，就能在不削弱其它工具防护的前提下放行这类高频场景。这是「约束只会收紧」原则的一个反向补充：默认从严，但给真实业务留一个**显式、按工具**的松绑开关，而不是让用户为了一个工具把全局阈值调垮。两组阈值都有 `model_validator` 守着「hard_limit 必须 ≥ warn」，配反了直接在加载期就报错。

## 稳定 key：怎么判断「这是同一个调用」

两层探测都依赖一个关键函数 `_stable_tool_key`——把一个 tool_call 归一成一个可比较的字符串。它的精细之处在于**按工具类型分别处理**：

- `read_file`：用「路径 + 200 行的桶」。也就是说，读同一个文件的第 1~200 行算同一个 key，读第 201~400 行算另一个 key——这样「翻页式地连续读一个大文件」不会被误判成死循环，但「反复读同一段」会。
- `write_file` / `str_replace`：对**完整参数**算哈希。因为写操作是内容敏感的，同一个文件写两次不同内容是完全正常的，必须把内容纳入 key。
- 其它工具：取一组「显著字段」（`path`/`url`/`query`/`command`/`pattern`/`glob`/`cmd`）做 key，忽略无关的细枝末节。

这套「按工具语义定制 key」的设计，正是「保留结构、移除噪声」在循环检测里的体现：要识别「真正的重复」，就得懂每种工具「什么样算重复」。

## 警告为什么注入在 wrap_model_call 而不是 after_model

这是 `LoopDetectionMiddleware` 里最考究的一处工程决策。直觉上，检测完循环、要给模型一句警告，最自然的地方是 `after_model`（模型刚输出完）。但 DeerFlow 偏偏把警告注入放在 `wrap_model_call`，注释解释得很硬核：在 `after_model` 往消息里插东西，会破坏「tool_call 必须紧跟配对 ToolMessage」的结构——OpenAI / Moonshot 会直接报「tool_call_ids did not have response messages」，而 Anthropic 不允许在流中间塞 SystemMessage。所以警告用的是 `HumanMessage`、并打上 `name="loop_warning"`，在下一次模型调用前注入，绕开了所有这些校验器的雷区。这呼应第 4 章 `DanglingToolCallMiddleware` 和第 7 章的同一关切——**OpenAI 兼容校验器对消息结构的要求，是 harness 里反复出现的隐形约束**。

强停的机制则是「剥掉 tool_calls」。`_build_hard_stop_update` 把 AIMessage 上的 `tool_calls`/`function_call` 清空，并把 `finish_reason` 从 `tool_calls` 改成 `stop`——模型本想调工具，但这一轮的工具调用被整个抹掉，循环就此断开。警告消息和强停消息都有专门的常量（`_WARNING_MSG` 开头是 `[LOOP DETECTED]`、`_HARD_STOP_MSG` 是 `[FORCED STOP]`，频次层另有 `_TOOL_FREQ_WARNING_MSG` / `_TOOL_FREQ_HARD_STOP_MSG`），又一次「错误即文档」——连给模型看的提示都写成清晰的人话。

还有几个稳健性细节：`max_tracked_threads=100` 配 LRU 淘汰，防止长期运行的服务把所有线程历史攒在内存里；待发警告是**瞬态**的（`_MAX_PENDING_WARNINGS_PER_RUN=4` 封顶），在 `after_agent` 里被丢弃，不会泄漏到下一个 run。

## 工具输出预算：一次返回不能撑爆窗口

第二类失控是「单次工具输出过大」。一条 `bash` 命令 `cat` 了一个几十万行的日志，或者一次 web 抓取拉回整个网页——这一坨内容会瞬间吃掉上下文。`ToolOutputBudgetMiddleware`（`tool_output_budget_middleware.py`）给每个工具结果设了预算。配置 `ToolOutputConfig` 默认也是开的，关键阈值：

```python
externalize_min_chars: int = 12_000   # 超过 1.2 万字符就外置到磁盘
preview_head_chars: int = 2_000       # 预览保留头部 2000 字符
preview_tail_chars: int = 1_000       # 预览保留尾部 1000 字符
fallback_max_chars: int = 30_000      # 磁盘不可用时的硬截断上限
```

它的处理策略是**优先外置、退而截断**：

1. 当输出超过 `externalize_min_chars`，把**全文写到磁盘**（`_externalize` 写宿主机、或 `_externalize_to_sandbox` 写进沙箱文件系统），然后用 `_build_preview` 把返回给模型的内容换成「头部 2000 字符 + 一句文件引用 + 尾部 1000 字符」。那句引用写得很贴心：`[Full {tool_name} output saved to {virtual_path} ({total} chars, ~{total // 4} tokens). Use read_file with start_line and end_line to access specific sections...]`——它告诉模型「完整内容存这儿了，想看哪段用 read_file 翻」。
2. 如果磁盘持久化不可用（写失败），退化成 `_build_fallback` 的「头 + 尾截断」，并保证结果绝不超过 `fallback_max_chars`。

几个值得记的工程细节：

- **外置到沙箱时要验证落地**（issue #3416）：远程 AIO 沙箱的 `write_file` 不会自动建父目录，所以先 `mkdir -p`，写完还要 `test -s` 确认文件真在那儿——因为 `execute_command` 失败时是把 `"Error: ..."` 当 stdout 返回而非抛异常，不能靠异常传播判断成败。「拒绝给模型一个读不出来的 read_file 路径」是边界自觉的又一例。
- **read_file 自身豁免**（`exempt_tools` 默认含 `read_file`）：防止「外置→模型用 read_file 读→又触发外置」的死循环。这正好和上一节循环检测的 read_file 分桶逻辑呼应——读操作要特殊对待。
- **快速预扫描**：`_needs_budget` / `_tool_message_over_budget` 先做一次廉价检查，只有真的超预算才进入「分配新列表、可能离线写盘」的重路径——小输出的常见情况下零开销。异步路径 `awrap_tool_call` 还把真正的写盘 I/O 用 `asyncio.to_thread` 丢进工作线程，不卡事件循环。
- **历史消息也兜底**：`wrap_model_call` 会扫一遍历史里的 ToolMessage，万一有漏网的大输出（比如外置时机错过），在喂给模型前再做一次 inline 截断。

## 安全终止：供应商掐断时，别执行半截工具调用

第三类失控最隐蔽。`SafetyFinishReasonMiddleware`（`safety_finish_reason_middleware.py`，背景是 issue #3028）处理的是：模型供应商出于安全原因**半路停止**生成，却仍返回了**部分成形**的 `tool_calls`。LangChain 的工具路由器只要看到 AIMessage 带非空 `tool_calls` 就「去执行吧」，于是一个停在句子中间的 `write_file` 就被当成完整调用派发出去——模型看到截断的文件，想修，又被过滤，于是死循环。

解法是：检测到安全终止信号 **且** 这条 AIMessage 带 tool_calls 时，把 tool_calls 整个剥掉（结构化的和原始 provider payload 都剥），附一句给用户的解释，并在 `additional_kwargs.safety_termination` 里留下可观测字段。

怎么识别「安全终止」？这部分被抽成独立的 `safety_termination_detectors.py`，定义了一个策略接口 `SafetyTerminationDetector` 和三个内置探测器——因为**不同供应商用不同字段、不同值**来表达「我为安全停了」：

- `OpenAICompatibleContentFilterDetector`：看 `finish_reason == "content_filter"`，覆盖 OpenAI、Azure、Moonshot/Kimi、DeepSeek、vLLM、Qwen 等一众 OpenAI 兼容网关。
- `AnthropicRefusalDetector`：Anthropic 用的是 `stop_reason == "refusal"`，是另一个字段。
- `GeminiSafetyDetector`：Gemini 用 `finish_reason` 但是一套大写枚举（`SAFETY`/`BLOCKLIST`/`PROHIBITED_CONTENT`/`SPII`/`RECITATION` 以及图像安全的几个）。

`GeminiSafetyDetector` 的注释是一份「克制」的范本：它明确列出**故意不纳入**默认集的枚举值——`STOP`（正常结束）、`MAX_TOKENS`（长度截断不是安全问题，issue #3028 把它划走了）、`MALFORMED_FUNCTION_CALL`（工具协议错误，跟安全过滤是不同类别，「为了让可观测记录诚实」要用专门的探测器处理）、`OTHER`（太宽泛，要用就显式 opt-in）。这种「只认我确定的、拿不准的宁可漏」的态度，避免把正常终止误判成安全终止。三个探测器都被要求「side-effect free 且容忍缺失/畸形元数据」——`_get_metadata_value` 同时查 `response_metadata` 和 `additional_kwargs`、只接受非空字符串，绝不在脏输入上抛错。

## 两个剥 tool_calls 的中间件，谁先看到模型输出

`SafetyFinishReasonMiddleware` 挂在 `after_model`，和 `LoopDetectionMiddleware` 共享「剥 tool_calls」这套机制，但触发条件不同。它的文件头注释专门讲了**注册顺序**：必须注册在 `LoopDetectionMiddleware` **之后**。原因是 LangChain 工厂按列表**逆序**接 `after_model` 边——最后注册的中间件最先看到模型输出。所以「Safety 注册在 Loop 之后」意味着 Safety 先看到原始响应、先清掉（被安全过滤的）tool_calls，然后 Loop 再针对「已清理的消息」做计数。这种「谁先看到」的精确排序，正是第 5 章中间件链顺序学问的延续——顺序不是摆设，它决定了两个中间件协作的语义。

它的可观测性也分两层：`_emit_event` 给实时 SSE 消费者（Web UI）发一个 `safety_termination` 事件，好让前端把已经流出去的「tool starting...」占位符收回；`_record_audit_event` 往 RunJournal 写一条持久审计记录，让运维能用一句 SQL 查「今天哪些 run 被安全压制了」。两处都刻意**不记录工具参数**——那些正是被供应商过滤掉的内容，持久化它们等于绕过安全过滤。只记名字、数量、id，够审计就行。所有可观测写入都包在 try/except 里、失败只 debug 日志——又一次「审计绝不能拖垮主流程」的可靠性分级（呼应第 3 章）。

## 本章小结

- 三类失控、三个中间件：`LoopDetectionMiddleware`（死循环）、`ToolOutputBudgetMiddleware`（单次大输出）、`SafetyFinishReasonMiddleware`（供应商安全终止留半截 tool_calls）；三者共享「改写消息 / 剥 tool_calls」机制但触发条件各异。
- 循环检测两层探测：哈希层（`_hash_tool_calls` 排序后 md5[:12]，对顺序不敏感，`warn_threshold=3`/`hard_limit=5`）+ 频次层（同工具类型，`tool_freq_warn=30`/`tool_freq_hard_limit=50`，可按工具 `tool_freq_overrides` 松绑如 bash 批处理）。
- `_stable_tool_key` 按工具语义定制：read_file 用「路径 + 200 行桶」（容忍翻页连续读），write_file/str_replace 哈希全参数（内容敏感），其它取显著字段。
- 警告注入在 `wrap_model_call`（用 `HumanMessage name="loop_warning"`）而非 `after_model`——避免破坏 tool_call↔ToolMessage 配对、避开 Anthropic 禁中途 SystemMessage；强停用 `_build_hard_stop_update` 清空 tool_calls 并把 finish_reason 改为 stop。
- 工具输出预算 `externalize_min_chars=12000`：超限则外置全文到磁盘/沙箱 + 返回「头 2000 + 文件引用 + 尾 1000」预览；磁盘不可用退化为 `fallback_max_chars=30000` 的头尾截断。沙箱外置要 `test -s` 验证落地（#3416），read_file 豁免防 persist→read→persist 循环，快速预扫描避免小输出开销。
- 安全终止探测器策略化（`safety_termination_detectors.py`）：OpenAI `content_filter` / Anthropic `refusal` / Gemini 大写枚举三个内置；Gemini 探测器刻意排除 `STOP`/`MAX_TOKENS`/`MALFORMED_FUNCTION_CALL`/`OTHER` 以免误判。
- `SafetyFinishReasonMiddleware` 须注册在 Loop **之后**（after_model 逆序接边，最后注册者最先看到输出），让 Safety 先清 tool_calls、Loop 再对清理后消息计数；审计只记工具名/数量/id，绝不记被过滤的参数。

## 动手实验

1. **实验一：推演两层探测的触发** — 阅读 `LoopDetectionConfig` 与 `loop_detection_middleware.py` 的 `_hash_tool_calls`。构造一段历史：模型连续 4 轮都调用 `{web_search(query="X"), web_fetch(url="Y")}`（顺序在第 3 轮颠倒）。问：哈希层会在第几轮注入警告？颠倒顺序会不会让它算成不同的调用集？为什么？
2. **实验二：理解 read_file 的分桶** — 看 `_stable_tool_key` 对 `read_file` 的处理。给定模型依次读 `a.log` 的第 1–200 行、201–400 行、1–200 行、1–200 行，列出每次的稳定 key，判断哪几次会被算作「相同调用」、循环计数到几。再说明为什么 write_file 不能用同样的分桶策略。
3. **实验三：跟踪一次输出外置** — 通读 `_budget_content` 与 `_build_preview`。假设 `bash` 返回 5 万字符、`externalize_min_chars=12000`、沙箱可用且非 thread-data 挂载，推演：内容会走哪条外置分支（`_externalize` 还是 `_externalize_to_sandbox`）？为什么外置后还要执行 `test -s`？模型最终收到的 ToolMessage 大致长什么样？
4. **实验四：核对安全终止的边界与顺序** — 阅读 `GeminiSafetyDetector` 的 docstring 与 `safety_finish_reason_middleware.py` 头部的「Placement」注释。回答：Gemini 返回 `finish_reason="MAX_TOKENS"` 且带 tool_calls 时，这个中间件会不会剥掉 tool_calls？为什么 `MAX_TOKENS` 被排除在默认集外？再说明：若把 Safety 注册在 Loop **之前**会发生什么后果？

> **下一章预告**：到这里，「防失控」三件套齐活了。但 agent 要真正聪明，光靠单次会话内的上下文还不够——它得**记住**跨会话的东西：用户偏好、过往结论、长期事实。下一章我们走进 `MemoryMiddleware` 与记忆更新管线，看 DeerFlow 如何在对话结束后异步地把「值得记的」抽出来、存进长期记忆，又如何在下一次对话开始时把它注入回上下文——以及它为什么把记忆更新做成「排队、降级、绝不阻塞主流程」。
