# 第 11 章　记忆系统：异步队列、置信度阈值与上传脱敏

防失控三件套管的是「单次会话内」的失控。但一个真正好用的 agent，得记住**跨会话**的东西：你偏好用什么风格、上次纠正过它什么、你的工作背景是什么。这些不能塞在每轮对话的上下文里（太贵），得抽出来、存成长期记忆，下次对话开始时再注回去。DeerFlow 的记忆系统是一条独立的管线，挂在 `MemoryMiddleware` 上。这一章我们看它怎么在对话结束后**异步、防抖、绝不阻塞主流程**地更新记忆，怎么用置信度阈值和去重守住记忆质量，以及它为什么要专门把「上传文件」的痕迹从记忆里擦掉。

## after_agent 排队：主流程一秒都不等

`MemoryMiddleware`（`agents/middlewares/memory_middleware.py`）挂在 `after_agent` 钩子上——整个 agent 跑完之后才动。它做的事极轻：把这轮对话**塞进一个队列**就返回 `None`（不改任何 state），真正的「读对话、调 LLM 总结、写文件」全部交给队列在后台异步做。这个设计的第一原则就是：**记忆更新绝不能让用户多等一秒**。

塞队列前它做了几道过滤：

1. `filter_messages_for_memory` 只留「用户输入 + 最终助手回复」，丢掉所有工具调用消息——记忆要记的是「聊了什么、得出什么结论」，不是中间的工具往返。
2. 必须至少有一条 user 消息**和**一条 assistant 消息才入队，否则没有「值得记的对话」。
3. 顺手跑 `detect_correction` / `detect_reinforcement`，识别这轮里有没有「纠正信号」或「正向强化信号」（下面详述）。

有一个容易被忽略却很关键的细节，注释专门点出来了：

```python
# threading.Timer fires on a different thread where ContextVar values are not
# propagated, so we must store user_id explicitly in ConversationContext.
user_id = get_effective_user_id()
```

记忆更新最终在 `threading.Timer` 的回调线程里执行，而 ContextVar（比如「当前请求是哪个用户」）**不会跨线程传播**。所以必须在「请求上下文还活着」的入队时刻就把 `user_id` 抓出来、显式存进 `ConversationContext`，否则后台线程更新记忆时就不知道该写到谁的文件里了。这是「边界不信任」的一个微妙现身——线程边界会吃掉 ContextVar，所以跨边界的数据必须显式打包带走。

## 防抖队列：30 秒窗口里的多轮对话合并成一次

`MemoryUpdateQueue`（`agents/memory/queue.py`）的核心是**防抖（debounce）**。你连续发了三条消息、agent 连续回了三轮，如果每轮都立刻调一次 LLM 更新记忆，既浪费又容易打架。队列的解法是：每次 `add` 都重置一个 `threading.Timer`，默认 `debounce_seconds=30`——只有当你**安静 30 秒**之后，才真正处理队列。窗口期内的多轮对话被攒在一起、一次性处理。

队列用 `(thread_id, user_id, agent_name)` 三元组做「防抖身份」。同一身份在窗口期内再次入队时，`_enqueue_locked` 会把旧的那条**替换**掉（只保留最新的消息列表），但会把 `correction_detected` / `reinforcement_detected` 这两个信号**合并**（`or` 逻辑）——也就是说，哪怕纠正信号出现在被替换掉的那一轮，它也不会丢。这是「保留结构、移除内容」的又一次体现：替换掉冗余的消息，但保住有价值的信号位。

处理时 `_process_queue` 还有一道并发自保：如果发现已经有一个 worker 在处理（`self._processing`），它不会重入，而是重新排一个 0 延迟的 timer 稍后再试——保住「立即冲刷」的语义又不撞车。批量处理多条时，每条之间 `time.sleep(0.5)` 避免把记忆模型的限流打爆。

`flush_nowait` / `daemon=True` 的注释也很诚实：守护线程意味着进程退出时，没处理完的队列可能丢——「对 best-effort 的记忆更新而言这是可接受的」。又一次「核心稳、增强降级」：记忆是锦上添花，丢一次不该拖垮主进程退出。

## 纠正与强化：从对话里嗅出「该重点记什么」

`message_processing.py` 里的 `detect_correction` / `detect_reinforcement` 是两个正则匹配器，扫最近 6 条用户消息：

- **纠正信号**：`that's wrong`、`you misunderstood`、`try again`、`redo`，以及中文的「不对」「你理解错了」「重试」「换一种」「改用」等。
- **强化信号**：`yes, exactly`、`perfect`、`that's exactly what I wanted`、`keep doing that`，以及中文的「对，就是这样」「完全正确」「正是我想要的」「继续保持」等。

这两个信号会变成给记忆 LLM 的额外提示（`_build_correction_hint`）：检测到纠正，就提示模型「特别注意用户纠正了什么，把正确做法记成 category=correction、confidence≥0.95 的 fact」；检测到强化，就提示「把被确认的做法/风格/偏好记成 category=preference 或 behavior、confidence≥0.9 的 fact」。这是一个很聪明的设计——**用户的纠正和肯定是最高质量的记忆信号**，系统主动去捕捉它们，并指示记忆模型给这类 fact 打高置信度。注意纠正和强化是互斥优先的（先检测纠正，没有纠正才看强化），避免一条「不对，应该这样……对了就是这样」的消息把两个信号都点亮。

## 记忆的结构：分层摘要 + 带置信度的 facts

记忆的数据结构定义在 `storage.py` 的 `create_empty_memory`，分三大块：

- `user`：`workContext` / `personalContext` / `topOfMind`——用户的工作背景、个人背景、当下关注点，每块是一段 `summary`。
- `history`：`recentMonths` / `earlierContext` / `longTermBackground`——按时间分层的历史摘要。
- `facts`：一个数组，每条 fact 带 `content`、`category`、`confidence`（0~1）、`createdAt`、`source` 等。

记忆更新（`MemoryUpdater._apply_updates`）的逻辑很见功力：

- **摘要分块更新**：每个 section 只有当 LLM 返回 `shouldUpdate=true` 且有非空 `summary` 时才覆盖——没新信息就不动旧摘要。
- **置信度阈值**：新 fact 只有 `confidence >= fact_confidence_threshold`（默认 0.7）才被收下。低于阈值的「猜测」被直接丢弃——宁缺毋滥。
- **内容去重**：用 `_fact_content_key`（`casefold` 归一化）比对，同样内容的 fact 不重复添加。
- **总量封顶**：facts 超过 `max_facts`（默认 100）时，**按置信度排序保留 top N**——记忆满了就淘汰最不确定的，而不是最旧的。

这套「阈值 + 去重 + 按置信度淘汰」的组合，本质是给「自动生成的记忆」装了几道质量闸，防止低质量噪声慢慢污染长期记忆。

## 解析模型输出：容错但不修复

记忆更新要让 LLM 返回一个 JSON。但模型常常在 JSON 外面裹一层思考痕迹、散文或 markdown 代码围栏。`_parse_memory_update_response` 的策略是「**安全提取、但不修复**」：用 `json.JSONDecoder().raw_decode` 从每一个 `{` 位置尝试解码，找到第一个**包含全部必需 top-level key**（`user`/`history`/`newFacts`/`factsToRemove`）的对象就用它，截断或畸形的 JSON 一律不尝试修补、直接放弃这次更新。

`_normalize_memory_update_data` 里还藏着一个 fail-safe 不变量：

```python
if normalized_facts_to_remove and dropped_new_fact:
    raise json.JSONDecodeError("Unsafe partial memory update: factsToRemove with malformed newFacts", ...)
```

意思是——如果这次更新**既要删 fact、又有畸形的 newFacts 被丢弃**，那就整体放弃这次更新而不是「只删不增」。因为「删了旧的、却没成功加上新的」会留下一个比更新前更糟的残缺状态。**宁可不更新，也不留半残的记忆**，这与第 3 章持久化的可靠性思路一脉相承。解析失败只 `logger.warning` 返回 False，绝不抛给主流程。

## 上传脱敏：会话级的东西不该进长期记忆

`_strip_upload_mentions_from_memory` 是一个非常具体、却很能说明「边界感」的处理。问题是：用户这次会话上传了一个文件（路径在 `/mnt/user-data/uploads/`），对话里自然会提到它。如果这被记进长期记忆，**下次会话**里 agent 就会去找一个早已不存在的文件——因为上传是会话级的。

所以记忆在落盘前会用一个精心设计的正则 `_UPLOAD_SENTENCE_RE` 把「描述上传**事件**」的句子从所有摘要和 facts 里擦掉。注释强调这个正则**刻意收得很窄**——它匹配「uploaded a file」「file upload」「/mnt/user-data/uploads/」「`<uploaded_files>`」这类，但**绝不**误伤「User works with CSV files」「prefers PDF export」这种正当的、与文件相关的长期事实。「该删的精确删掉、不该碰的一个字不改」——这种克制和第 8 章「只对 agent 写过的文件做反向解析」、第 10 章安全终止「只记名字不记参数」是同一种边界自觉。

## 写盘的原子性与隔离

`FileMemoryStorage`（`storage.py`）几个稳健性细节值得记：

- **原子写**：`save` 先写一个带 uuid 后缀的临时文件，再 `temp_path.replace(file_path)` 原子替换——避免写到一半崩溃留下半个损坏的 JSON。
- **写前不污染缓存/入参**：先 `{**memory_data, "lastUpdated": ...}` 浅拷贝，保证文件写成功前不更新缓存引用、也不改调用方的 dict。
- **per-user / per-agent 隔离**：记忆文件路径按 `user_id` / `agent_name` 分开，`_validate_agent_name` 用 `AGENT_NAME_PATTERN` 校验防路径穿越。
- **mtime 缓存**：`load` 按文件修改时间缓存，文件没变就走缓存。
- **storage 可插拔**：`get_memory_storage` 用 `storage_class` 配置反射加载 provider，加载失败**降级**回内置 `FileMemoryStorage`——又一次配置驱动 + 失败降级。

`MemoryUpdater` 还有一个跨 event-loop 的坑值得一提（issue #2615）：记忆更新走的是**同步** `model.invoke()` 而非异步，并在「已在事件循环里」时用一个专属线程池 `_SYNC_MEMORY_UPDATER_EXECUTOR` 离线执行。注释解释得很硬核——用同步路径就不会创建第二个 event loop，也就绝不会碰到 langchain 全局缓存的异步 httpx 连接池（那个和主 agent 共享的），从而杜绝「跨 loop 复用连接」的诡异 bug。又一次：把一个踩过的并发坑，连同它的根因，钉死在代码注释里。

最后说回**注入**侧：`MemoryConfig` 有 `injection_enabled`（默认 True）和 `max_injection_tokens`（默认 2000）——记忆在下一次对话开始时被注入回上下文（由第 5 章提过的 `DynamicContextMiddleware` 注入进首条 HumanMessage 以利前缀缓存），且有 token 预算封顶。`token_counting` 还给了 `char` 这个「不联网」选项，因为 tiktoken 首次用时可能要从公网下载 BPE 数据、在受限网络里会长时间卡住（issue #3402/#3429）——连「数 token 这一步会不会卡」都考虑到了。

## 本章小结

- `MemoryMiddleware` 挂 `after_agent`，只做「过滤 + 入队」就返回，记忆更新全部异步后台执行，主流程零等待；入队时显式抓 `user_id` 因为 `threading.Timer` 回调线程不传播 ContextVar。
- `filter_messages_for_memory` 只留 user 输入 + 最终 assistant 回复（丢工具消息），且需至少各一条才入队。
- `MemoryUpdateQueue` 防抖 `debounce_seconds=30`：同 `(thread_id,user_id,agent_name)` 身份在窗口内替换消息但 `or` 合并 correction/reinforcement 信号；`_processing` 防重入；daemon 线程丢失可接受（best-effort）。
- `detect_correction`/`detect_reinforcement` 中英双语正则扫最近 6 条用户消息，转成给记忆 LLM 的高置信度提示（correction≥0.95、preference/behavior≥0.9）；纠正优先于强化。
- `_apply_updates` 守质量：摘要按 `shouldUpdate` 分块更新、新 fact 过 `fact_confidence_threshold=0.7`、`casefold` 去重、超 `max_facts=100` 按置信度淘汰最低的。
- `_parse_memory_update_response` 容错提取首个含全部必需 key 的 JSON 但不修复截断；`factsToRemove` 与畸形 `newFacts` 并存时整体放弃更新（不留半残记忆）；失败只 warning。
- `_strip_upload_mentions_from_memory` 用刻意收窄的正则擦掉「上传事件」句子（会话级），但不误伤「常处理 CSV」这类长期事实；`FileMemoryStorage` 原子写（temp+replace）、per-user/agent 隔离、storage 可插拔且失败降级。
- 记忆更新走同步 `model.invoke()` + 专属线程池避开跨 event-loop 连接复用 bug（#2615）；注入侧 `max_injection_tokens=2000`，`token_counting` 提供 `char` 免联网选项（#3402/#3429）。

## 动手实验

1. **实验一：跟踪一条对话进队列** — 阅读 `MemoryMiddleware.after_agent` 与 `MemoryUpdateQueue.add`。假设用户在 10 秒内连发两条消息、agent 各回一轮，且第一轮里用户说了「不对，重做」。推演：队列里最终留几条 `ConversationContext`？`correction_detected` 是什么值？为什么 `user_id` 必须在 `after_agent` 里抓而不能在后台线程里取？
2. **实验二：算一次 facts 淘汰** — 看 `_apply_updates` 的「Add new facts」与「Enforce max facts limit」分支。假设当前有 99 条 fact、LLM 返回 3 条新 fact（confidence 分别 0.6/0.75/0.9）、`fact_confidence_threshold=0.7`、`max_facts=100`。推演：哪几条被收下？最终留几条?如果总数超 100，谁会被淘汰？
3. **实验三：验证「不留半残记忆」** — 阅读 `_normalize_memory_update_data` 的 `raise json.JSONDecodeError(...)` 分支。构造一个会触发它的 LLM 输出（提示：`factsToRemove` 非空 + 某条 `newFacts` 缺 `content`）。再说明：如果去掉这道守卫、改成「只删不增」，长期记忆会陷入什么坏状态？
4. **实验四：测试上传脱敏的边界** — 通读 `_UPLOAD_SENTENCE_RE`。对下面四句分别判断会不会被擦掉，并解释为什么：①「User uploaded a report.pdf for analysis.」②「User works with CSV files daily.」③「The file at /mnt/user-data/uploads/x.csv was processed.」④「User prefers PDF export over Excel.」

> **下一章预告**：记忆让 agent 跨会话变聪明，但单个 agent 的「脑容量」终究有限。面对一个需要并行调研十个子问题的大任务，super agent 的答案是**分身**——派出一队子 agent 各管一摊。下一章我们走进 `task_tool` 与子 agent 系统，看 DeerFlow 如何派生、隔离、限流这些分身，子 agent 为什么默认禁 bash，以及主 agent 怎样收束它们的结果而不让并发把状态搅乱。
