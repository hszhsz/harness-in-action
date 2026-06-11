# 第 2 章　Agent 主循环与回合生命周期

如果说第 1 章解决的是"进程怎么活起来"，那么这一章要解决的是 Agent 的心脏问题：**一次用户消息，如何被驱动成一轮又一轮"调用模型—执行工具—再调用模型"的循环，直到任务收束？** 这个循环就是 Agent 之所以是 Agent、而不是一个单次问答机器人的根本原因。

Hermes 把这套机制放在 `agent/conversation_loop.py` 里。它不是教科书里那种二十行的 while 循环——它要处理流式输出、工具并发执行、上下文超长压缩、模型因长度截断后的续写、被截断的工具调用重试、预算耗尽、以及插件钩子。我们这一章就把这台精密机器逐层拆开，并且特别留意那些写死在源码里的常量——它们是设计意图最诚实的证词。

## 2.1 入口：`run_conversation` 与回合前奏

主循环的入口是 `run_conversation(agent, user_message, system_message=None, conversation_history=None, task_id=None, stream_callback=None, persist_user_message=None)`（约第 371 行）。注意它的第一个参数就是 `agent`——也就是上一章末尾提到的 `AIAgent` 实例。循环本身不是 `AIAgent` 的方法，而是一个接受 agent 作为参数的自由函数。这种"把状态对象显式传进来"的写法，让循环逻辑更容易被单独测试。

每一回合真正开干之前，有一段繁重的"前奏"被抽到了 `agent/turn_context.py` 的 `build_turn_context(...)` 里。这一步做的事情多得惊人：守护标准输入输出、重置各类重试计数器、对输入做净化、恢复或重建系统提示词、为应对崩溃而提前持久化、执行**预压缩（preflight compression）**、触发 `pre_llm_call` 插件钩子、以及预取外部记忆。它返回一个 `_ctx` 对象，循环再把其中的 `messages`、`active_system_prompt`、`turn_id` 等局部变量读回来。

把这么重的前奏单独抽成一个函数，是一个值得学习的结构决策：**主循环只负责"循环"，所有"一回合开始前要准备的状态"都收敛在一处**。否则 `run_conversation` 会膨胀成无法阅读的怪物。

紧接着，循环初始化一组关键计数器（约第 436–445 行）：`api_call_count = 0`、`length_continue_retries = 0`、`truncated_tool_call_retries = 0`、`compression_attempts = 0`、`_turn_exit_reason = "unknown"`。记住这几个名字，本章后半段它们每一个都对应一类失败恢复策略。

## 2.2 循环条件：迭代预算与"宽限调用"

循环的主条件写在约第 461 行：

```python
while (api_call_count < agent.max_iterations
       and agent.iteration_budget.remaining > 0) or agent._budget_grace_call:
```

这一行包含三个值得玩味的设计。第一，`api_call_count < agent.max_iterations` 是硬性的迭代次数上限——防止 Agent 陷入无限的"调工具—看结果—再调工具"死循环。第二，`agent.iteration_budget.remaining > 0` 引入了一个独立于次数的**预算**概念，预算可以按 token、成本等维度耗尽。第三，也是最微妙的——末尾那个 `or agent._budget_grace_call`：即使次数用完、预算耗尽，只要"宽限调用"标志为真，循环仍会再跑一次。这是给 Agent 一个"临终遗言"的机会，比如在预算刚好用尽时还能输出一段对用户的交代，而不是粗暴地戛然而止。

每进入一次迭代，`api_call_count += 1`，并调用 `agent._touch_activity(...)` 刷新活跃时间戳（这关系到后续的空闲判定与后台维护），如果设置了 `agent.step_callback`，还会回调 `agent.step_callback(api_call_count, prev_tools)` 让外部观察每一步。

这里还埋了一条岔路：如果 `agent.api_mode == "codex_app_server"`（约第 452 行），整个回合会被交给 `agent._run_codex_app_server_turn(...)` 处理，绕过默认路径。这说明 Hermes 的主循环本身是可被替换的运行时之一——它支持不止一种"回合执行模型"。

## 2.3 三道 `< 3` 防线：失败恢复的克制美学

Agent 在和真实模型 API 打交道时，会遇到三类典型的"非致命异常"，Hermes 为每一类都设计了带上限的重试。耐人寻味的是，**这三个上限不约而同都是 3**——既给了恢复机会，又坚决不允许无限重试。

第一道是**上下文超长压缩**。`max_compression_attempts = 3`（约第 803 行）。当请求因上下文过长（典型的 413 类错误）失败时，循环会调用 `agent._compress_context(...)` 压缩历史消息后重试；每成功压缩一次，`compression_attempts` 归零；一旦尝试次数超过 3，就返回硬错误：`"Context length exceeded: max compression attempts (3) reached."`。这个错误信息本身就在向开发者解释"我已经尽力压了三次还是装不下"。

第二道是**长度截断续写**。当模型因为输出达到长度上限（`finish_reason == "length"`，即第 1 章见过的 `FINISH_REASON_LENGTH`）而被截断时，循环会自动发起续写请求，但 `length_continue_retries` 被限制在 `< 3`（约第 1413–1444 行）。

第三道是**被截断的工具调用重试**。如果模型吐出的工具调用 JSON 因长度被截断、无法解析，循环会重试，`truncated_tool_call_retries` 同样封顶 `< 3`（约第 1477–1499 行）。这里有个精巧的细节：重试时会给一个 token 提升量 `_tc_boost = _tc_boost_base * (truncated_tool_call_retries + 1)`——**每多重试一次，就给模型更多的输出预算**，因为工具调用被截断往往意味着 token 不够用，单纯重试而不加预算只会再次截断。

把三道防线放在一起看，能读出 Hermes 的一种哲学：**失败恢复要有，但必须有界**。无界重试会把一次小故障放大成无限循环、烧光预算；而把上限统一设在一个小数字（3），既覆盖了绝大多数瞬时抖动，又能让真正的硬故障快速暴露而不是被无限掩盖。

## 2.4 压缩重启：当上下文实在装不下

当 413 / 上下文超长错误发生且决定压缩时（约第 2500–2520 行），循环做的不只是"压缩后接着跑"，而是一套完整的**重启编排**：调用 `agent._compress_context(messages, system_message, approx_tokens=..., task_id=...)`，把 `conversation_history` 置为 `None`（避免旧的未压缩历史再次被带入），`time.sleep(2)` 短暂退避，设置 `_retry.restart_with_compressed_messages = True`，然后 `break` 跳出当前迭代去用压缩后的消息重启。

为什么是"重启"而不是"继续"？因为压缩会重写整个消息序列——把中间一大段对话替换成摘要。这相当于换了一份新的输入，干净地从压缩后的状态重新发起，比在原循环里打补丁要可靠得多。压缩的具体算法是第 3 章的主题，这里我们只需记住：**主循环把"压缩"当成一个可以触发循环重启的一等事件来处理**。

## 2.5 错误分类：把异常翻译成恢复动作

主循环之所以能对不同异常采取不同策略，靠的是 `agent/error_classifier.py` 这个"翻译官"。它的核心是一个 `@dataclass ClassifiedError`（约第 69 行），字段包括 `reason`（一个 `FailoverReason` 枚举，约第 24 行）、`status_code`、`provider`、`model`、`message`、`error_context`，以及一组**恢复提示**：`retryable=True`、`should_compress=False`、`should_rotate_credential=False`、`should_fallback=False`，外加一个 `is_auth` 属性。

主入口函数 `classify_api_error(...)`（约第 441 行）通过三条子路径协同判断：`_classify_by_status`（按 HTTP 状态码）、`_classify_by_error_code`（按错误码）、`_classify_by_message`（按错误文本）。例如上下文超长 / 413 类错误会被标成 `retryable=True, should_compress=True`；而无法归类的默认走 `FailoverReason.unknown`，仍然是可重试的。主循环则在约第 2010 行读取 `classified.retryable` 与 `classified.should_compress`，据此决定是直接重试、还是先压缩再重试、还是轮换凭证、还是降级到备用模型。

这是"错误即文档、错误即指令"的典范：**异常不是被笼统地 catch 然后打印栈，而是被结构化地分类成"接下来该做什么"的决策数据**。第 8 章讲凭证池轮换、备用模型降级时，我们会再回到这个分类器，看 `should_rotate_credential` 和 `should_fallback` 如何被消费。

## 2.6 周边压缩设施：分工明确的几个模块

主循环本身不实现压缩细节，而是协同好几个分工明确的模块。`agent/conversation_compression.py` 里有 `compress_context(...)`（约第 271 行）、`check_compression_model_feasibility`（约第 64 行，判断当前模型是否适合做压缩这件事）、以及 `try_shrink_image_parts_in_messages`（约第 634 行，当上下文超长时优先缩小图片部分而不是文本）。独立的 `trajectory_compressor.py` 则提供了面向"训练数据/轨迹"的 `TrajectoryCompressor`（约第 332 行），带 `compress_trajectory` / `compress_trajectory_async` 与一个 `CompressionConfig` 数据类——这条线服务于 Hermes 作为研究平台的另一面（批量生成轨迹用于训练下一代工具调用模型）。

把这些模块名记下来，你会更清楚"压缩"在 Hermes 里不是一个函数，而是一个**有层次的子系统**：实时对话压缩、图片瘦身、可行性检查、训练轨迹压缩各司其职。第 3 章会聚焦其中最核心的"实时对话压缩引擎"。

## 本章小结

- 主循环入口是自由函数 `run_conversation(agent, ...)`（约第 371 行），把 `AIAgent` 实例显式传入，便于独立测试。
- 每回合的繁重前奏（stdio 守护、计数器重置、系统提示重建、预压缩、`pre_llm_call` 钩子、记忆预取）被抽到 `agent/turn_context.py:build_turn_context`，主循环只管"循环"。
- 循环条件 `(api_call_count < max_iterations and iteration_budget.remaining > 0) or _budget_grace_call` 同时受次数上限与独立预算约束，并保留"宽限调用"让 Agent 能在预算耗尽时留遗言。
- 三类非致命异常各有一道 `< 3` 的重试上限：压缩 `max_compression_attempts = 3`、长度续写 `length_continue_retries < 3`、截断工具调用 `truncated_tool_call_retries < 3`，后者每重试一次还递增 token 预算 `_tc_boost`。
- 失败恢复"有界"是核心哲学：无界重试会放大故障、烧光预算，统一封顶在 3 既容错又能快速暴露硬故障。
- 上下文超长时走完整"压缩重启"：压缩消息、清空 `conversation_history`、退避 2 秒、置 `restart_with_compressed_messages=True` 后 break 重启。
- `agent/error_classifier.py` 的 `classify_api_error` 把异常翻译成结构化的 `ClassifiedError`（带 `retryable`/`should_compress`/`should_rotate_credential`/`should_fallback`），主循环据此选择恢复动作——"错误即指令"。
- 压缩是一个分层子系统：实时对话压缩、图片瘦身、可行性检查（`conversation_compression.py`）与训练轨迹压缩（`trajectory_compressor.py`）各司其职。

## 动手实验

1. **实验一：拆解循环条件** —— `Read` `agent/conversation_loop.py` 约第 461 行的 `while` 条件，逐项解释 `api_call_count < max_iterations`、`iteration_budget.remaining > 0`、`_budget_grace_call` 三者的关系。构造一个思想实验：在什么状态组合下循环会"次数没到但提前退出"？又在什么状态下会"次数到了还多跑一次"？

2. **实验二：复盘三道重试防线** —— 用 Grep 在 `conversation_loop.py` 中分别定位 `max_compression_attempts`、`length_continue_retries`、`truncated_tool_call_retries` 三个变量的全部出现位置，画出每一类失败的"触发条件 → 重试动作 → 上限 → 超限后果"四段式流程。重点解读 `_tc_boost` 为什么要随重试次数递增。

3. **实验三：跟踪一次压缩重启** —— 阅读约第 2500–2520 行的压缩重启分支，列出它依次修改了哪些状态（`conversation_history`、`_retry.restart_with_compressed_messages` 等），并解释为什么这里选择 `break` 重启而不是原地继续。把它和第 3 章的压缩算法联系起来思考"重启"的代价与收益。

4. **实验四：读懂错误分类器** —— `Read` `agent/error_classifier.py` 的 `ClassifiedError` 数据类与 `classify_api_error`。自己挑三种真实错误（401 鉴权失败、429 限流、413 上下文超长），推演每一种会被设成怎样的 `retryable` / `should_compress` / `should_rotate_credential` / `should_fallback` 组合，再回到主循环约第 2010 行验证这些字段如何被消费。

> **下一章预告**：本章我们多次提到"压缩"，却一直把它当黑盒。第 3 章将打开这个黑盒，解剖 `agent/context_engine.py` 的抽象基类与 `agent/context_compressor.py` 的默认实现——看清 `threshold_percent`、`protect_first_n` / `protect_last_n`、`MINIMUM_CONTEXT_LENGTH = 64_000` 这些常量如何决定"保留什么、丢弃什么、摘要什么"，以及那个防止反复无效压缩的"抗抖动"开关。
