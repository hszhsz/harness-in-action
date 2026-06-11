# 第 3 章　上下文引擎与压缩

上一章我们反复把"压缩"当黑盒一笔带过。但对任何长时间运行的 Agent 来说，上下文压缩都不是可有可无的优化，而是**生死攸关的核心能力**。模型的上下文窗口是有限的，而一个真实任务可能跑几十上百轮、塞进海量工具输出。没有压缩，Agent 跑不了几轮就会撞上上下文上限然后崩溃。

Hermes 把这件事抽象成一个叫"上下文引擎（Context Engine）"的可插拔子系统：定义一个抽象基类规定接口，再提供一个默认实现 `compressor`。这一章我们就读透这两层——抽象层规定了"一个合格的上下文引擎必须回答哪些问题"，实现层则给出 Hermes 默认的答案，而那些写死的阈值常量，正是这些答案最精确的表达。

## 3.1 抽象层：`ContextEngine` 规定的契约

`agent/context_engine.py` 定义了 `class ContextEngine(ABC)`。把它当作一份"接口规范"来读，能立刻看清 Hermes 认为"上下文管理"由哪几个职责构成。

抽象方法有四个：`name`（引擎名）、`update_from_response(usage)`（根据模型返回的用量更新内部 token 账本）、`should_compress(prompt_tokens=None)`（判断现在该不该压缩）、`compress(messages, current_tokens=None, focus_topic=None)`（执行压缩）。这四个方法把"管理上下文"拆成了两个核心动词：**判断**（该不该压）与**执行**（怎么压），中间靠一个不断更新的 token 账本连接。

类级别的默认值定义了引擎的基础行为：`threshold_percent = 0.75`（用量达到上下文窗口的 75% 就考虑压缩）、`protect_first_n = 3`（保护开头 3 条消息不被压）、`protect_last_n = 6`（保护结尾 6 条消息不被压）。这三个数字背后是一个朴素但重要的洞察：**对话的"头"和"尾"信息密度最高**——头部通常是系统提示与任务定义，尾部是最近的上下文，中间那一大段历史才是最适合被摘要压缩的部分。

除了四个抽象方法，基类还预留了一组生命周期钩子：`on_session_start`、`on_session_end`、`on_session_reset`（重置 token 计数器与 `compression_count`），以及一组可选的精细化开关：`should_compress_preflight`（默认 False，是否在请求发出前就预判压缩）、`should_defer_preflight_to_real_usage`（默认 False，是否把预判推迟到拿到真实用量后）、`has_content_to_compress`（默认 True）、`get_tool_schemas` / `handle_tool_call`（让引擎自己也能暴露工具）。还有一个 `update_model(...)`，它会在切换模型时重算 `threshold_tokens = int(context_length * threshold_percent)`——因为不同模型的上下文窗口不同，阈值必须跟着模型走。引擎由 config.yaml 的 `context.engine` 选择，默认就是 `"compressor"`。

这套抽象的价值在于：**它把"何时压、压多少、保护哪些"全部参数化、钩子化**，于是你可以替换成完全不同的策略（比如基于向量检索的引擎），而主循环代码一行都不用改。这正是第 2 章里主循环能把压缩当黑盒调用的底气。

## 3.2 默认实现：`ContextCompressor` 的参数选择

`agent/context_compressor.py` 里的 `class ContextCompressor(ContextEngine)`（约第 522 行）是默认引擎，`name == "compressor"`。它的 `__init__`（约第 600 行）做了几处关键的"覆盖默认值"，每一处覆盖都是一个深思熟虑的工程判断。

最显眼的是：`threshold_percent` 从基类的 0.75 **下调到了 0.50**。也就是说默认引擎更激进——用量刚到上下文窗口一半就开始压缩，而不是等到 75%。为什么？因为压缩本身要消耗一次 LLM 调用、也需要一点缓冲空间，等到 75% 才动手风险更高（这一轮的新输出可能直接顶破窗口）。把阈值压到一半，是用"早压、勤压"换"绝不撞墙"。

同时 `protect_last_n` 从 6 **上调到 20**——保护更多的近期消息。这同样合理：尾部是 Agent 当前正在推进的工作上下文，丢了它等于失忆。`protect_first_n` 保持 3。还有一个 `summary_target_ratio = 0.20`（被夹在 `max(0.10, min(ratio, 0.80))` 之间），意思是摘要的目标长度约为原文的 20%。以及 `abort_on_summary_failure = False`——摘要失败时不中止，而是降级处理，体现"压缩失败也不能让整个回合崩"的容错取向。

阈值的最终计算是：`threshold_tokens = max(int(context_length * threshold_percent), MINIMUM_CONTEXT_LENGTH)`。这里 `MINIMUM_CONTEXT_LENGTH = 64_000`（定义在 `agent/model_metadata.py` 约第 133 行）是一道地板：**即使某个模型自报的上下文窗口很小，阈值也不会低于 6.4 万 token**。这避免了在小窗口模型上过度频繁压缩到无法工作。`tail_token_budget = int(threshold_tokens * summary_target_ratio)` 则决定了尾部保护区按 token 计的预算。

## 3.3 一串常量背后的工程经验

`context_compressor.py` 的模块级常量值得逐个品读，它们几乎是把"做对话压缩会踩的坑"全列了出来：

- `_MIN_SUMMARY_TOKENS = 2000`、`_SUMMARY_TOKENS_CEILING = 12_000`：摘要长度有下限也有上限。太短会丢信息，太长则压缩没意义——把摘要锁在 2000～12000 token 之间。
- `_SUMMARY_RATIO = 0.20`：与 `summary_target_ratio` 呼应的目标压缩比。
- `_CHARS_PER_TOKEN = 4`：字符到 token 的粗略换算系数，用于不调用 tokenizer 时的快速估算。
- `_IMAGE_TOKEN_ESTIMATE = 1600`、`_IMAGE_CHAR_EQUIVALENT = _IMAGE_TOKEN_ESTIMATE * _CHARS_PER_TOKEN`：一张图片粗估值 1600 token。Agent 的上下文里图片很贵，必须单独估算，否则按文本长度估会严重低估。
- `_SUMMARY_FAILURE_COOLDOWN_SECONDS = 600`：摘要失败后有 10 分钟冷却，避免反复触发昂贵且会失败的压缩。
- `_FALLBACK_SUMMARY_MAX_CHARS = 8_000`、`_FALLBACK_TURN_MAX_CHARS = 700`：当正常摘要不可用时的降级截断长度。
- `_PRUNED_TOOL_PLACEHOLDER = "[Old tool output cleared to save context space]"`：被清理掉的旧工具输出会被这个占位符替换——保留"这里曾有工具输出"的结构信息，只抹掉内容。这正是第 15 章会提炼的"留结构、去内容"原则的一个微观实例。
- `LEGACY_SUMMARY_PREFIX = "[CONTEXT SUMMARY]:"`：历史摘要的前缀标记，用于识别"这条消息本身是一段摘要"。
- `_IMAGE_PART_TYPES = frozenset({"image_url", "input_image", "image"})`：识别消息中图片部分的类型集合。

这串常量本身就是一份微缩的"对话压缩设计备忘录"。它告诉你：压缩不是简单地"截断旧消息"，而要分别处理文本与图片、要给摘要设上下限、要为失败留冷却、还要用占位符保住结构。

## 3.4 判断逻辑：`should_compress` 与抗抖动

`should_compress()`（约第 744 行）的逻辑短小却精悍。第一步很直白：如果当前 `tokens < self.threshold_tokens`，返回 False——还没到阈值，不压。

真正精彩的是它的**抗抖动（anti-thrashing）机制**：如果 `_ineffective_compression_count >= 2`（即最近两次压缩每次省下的空间都 `< 10%`），就跳过压缩并发出警告。这道防线解决的是一个真实而棘手的问题——**当对话已经被压无可压时，再压也省不出空间，但每压一次都要烧一次 LLM 调用**。如果没有这个开关，一个已经塞满不可压缩内容（比如大量近期工具输出）的会话，会陷入"压缩→没效果→还是超长→再压缩"的昂贵死循环。Hermes 的做法是：连续两次压缩都没什么效果，就认定"压不动了"，干脆不压，把问题交给上层（比如第 2 章里那个 `max_compression_attempts = 3` 的硬上限，或者提示用户开新会话）。

这是一个很能体现成熟度的细节：**新手关心"功能能不能跑通"，老手关心"功能在退化场景下会不会变成灾难"**。抗抖动开关就是为退化场景准备的安全阀。

## 3.5 压缩算法：五步流水线

`ContextCompressor` 的压缩算法在其文档字符串里被清晰地描述成一条五步流水线，我们顺着读：

1. **修剪旧工具结果**（`_prune_old_tool_results`，**不调用 LLM**）：先做最便宜的清理——把陈旧的工具输出替换成前面那个 `_PRUNED_TOOL_PLACEHOLDER` 占位符。工具输出往往是上下文里最臃肿的部分（一次文件读取、一次搜索可能就是几千 token），优先清它们性价比最高，而且这一步纯本地操作、零 LLM 成本。
2. **保护头部**：系统提示 + 第一轮交互（对应 `protect_first_n`）原样保留。
3. **按 token 预算保护尾部**：用大约 `tail_token_budget` 的预算保住最近的消息（对应 `protect_last_n = 20`）。
4. **LLM 摘要中段**：把头尾之间的那一大段历史交给 LLM 压成摘要——这是唯一需要模型参与、也是最贵的一步，所以被放在便宜手段都用尽之后。
5. **迭代更新 `_previous_summary`**：在后续多次压缩中，把上一次的摘要也纳入更新，避免"摘要套摘要"导致信息无限衰减。

这条流水线的排序逻辑非常清晰：**从便宜到昂贵、从无损到有损**。先做零成本的工具输出修剪，再做无损的头尾保护，最后才动用昂贵且有损的 LLM 摘要。这种"能用便宜手段解决就绝不动用昂贵手段"的分层处理，是资源敏感系统的通用智慧。

## 3.6 预压缩与图片瘦身：两条辅助路径

除了主流水线，还有两条辅助路径值得一提。

**预压缩（preflight）** 在 `agent/turn_context.py`（约第 254 行）：它用 `estimate_request_tokens_rough(...)` 在请求真正发出前粗估 token 数，再咨询引擎的 `should_defer_preflight_to_real_usage(...)` 决定是现在就压、还是等拿到真实用量再说。只有在不推迟的情况下，才提前更新 `_compressor.last_prompt_tokens`。这条路径让 Agent 有机会"在撞墙之前先低头"，而不是非要等 API 返回 413 才被动压缩——这正好和第 2 章主循环里那条"被动压缩重启"形成主动/被动的互补。

**图片瘦身** 在 `agent/conversation_compression.py` 的 `try_shrink_image_parts_in_messages`（约第 634 行）：当上下文超长时，优先缩小消息里的图片部分。结合前面 `_IMAGE_TOKEN_ESTIMATE = 1600` 的估算，你能理解为什么图片要被单独拎出来处理——一张图顶得上几百字文本，先压图片往往比压文本更划算，且对文本语义无损。`compress_context(...)`（约第 271 行）与 `check_compression_model_feasibility`（约第 64 行）则分别承担实际压缩与"当前模型是否适合做压缩"的可行性判断。

至此，第 2 章里那个被反复调用的"压缩黑盒"已经被完全打开：它是一个由抽象引擎接口、激进的默认阈值、一串经验常量、抗抖动安全阀、五步流水线、以及主动预压缩共同组成的精密子系统。

## 本章小结

- 上下文管理被抽象为可插拔的 `ContextEngine(ABC)`，四个抽象方法 `name` / `update_from_response` / `should_compress` / `compress` 把职责拆成"判断"与"执行"两个动词，由 token 账本连接。
- 基类默认 `threshold_percent = 0.75`、`protect_first_n = 3`、`protect_last_n = 6`，背后是"对话头尾信息密度最高、中段最适合摘要"的洞察；引擎由 `context.engine` 选择，默认 `compressor`。
- 默认实现 `ContextCompressor` 把 `threshold_percent` 下调到 0.50（早压勤压防撞墙）、`protect_last_n` 上调到 20（保住工作上下文）、`summary_target_ratio = 0.20`。
- `threshold_tokens = max(int(context_length * 0.50), MINIMUM_CONTEXT_LENGTH)`，其中 `MINIMUM_CONTEXT_LENGTH = 64_000` 是地板，防止小窗口模型上过度频繁压缩。
- 一串模块常量构成"压缩设计备忘录"：摘要 2000–12000 token 上下限、图片粗估 1600 token、失败 600 秒冷却、`_PRUNED_TOOL_PLACEHOLDER` 占位符保结构去内容。
- `should_compress` 的抗抖动开关：连续两次压缩省空间都 `< 10%`（`_ineffective_compression_count >= 2`）就停止压缩，避免"压不动还反复压"的昂贵死循环。
- 压缩是一条"从便宜到昂贵、从无损到有损"的五步流水线：零成本修剪工具输出 → 无损保护头尾 → 昂贵的 LLM 摘要中段 → 迭代更新历史摘要。
- 预压缩（preflight）让 Agent 主动在撞墙前压缩，图片瘦身优先缩小高成本的图片部分，二者与主循环的被动压缩重启互补。

## 动手实验

1. **实验一：对比抽象层与实现层的默认值** —— `Read` `agent/context_engine.py` 的类级默认值与 `agent/context_compressor.py:__init__`（约第 600 行）的覆盖值，列一张对照表（`threshold_percent`、`protect_first_n`、`protect_last_n`）。针对每一处差异，写一句话解释默认引擎为什么要这样调。

2. **实验二：解读地板常量** —— 用 Grep 在 `agent/model_metadata.py` 找到 `MINIMUM_CONTEXT_LENGTH = 64_000`，再回到 `threshold_tokens` 的计算式。构造一个反例：假设某模型上下文窗口只有 32000 token，算出 `threshold_tokens` 实际是多少，并解释这个地板防住了什么问题。

3. **实验三：复现抗抖动判断** —— `Read` `should_compress()`（约第 744 行），把它的两个返回 False 的条件（未达阈值、连续两次无效压缩）各写成一个最小判断伪代码。思考：如果删掉 `_ineffective_compression_count >= 2` 这一支，一个塞满近期工具输出的会话会发生什么？

4. **实验四：给五步流水线排序找理由** —— 对照压缩算法的五个步骤，论证为什么"修剪旧工具结果"必须排在"LLM 摘要中段"之前。再用 Grep 找到 `_prune_old_tool_results` 与 `_PRUNED_TOOL_PLACEHOLDER`，确认被清理的工具输出是被删除还是被占位符替换，并解释这对后续诊断的意义。

> **下一章预告**：上下文准备好了，模型也该调用工具了。第 4 章我们转向 Hermes 的工具系统——`tools/registry.py` 如何在导入时注册工具、`ToolRegistry` 单例如何用 generation 计数器追踪变更、`dispatch` 如何把模型吐出的工具名翻译成真实执行，以及 `toolsets.py` 怎样把几十个工具编排成可按场景启用的分组。
