# 第 16 章　持久化与可观测性:状态落盘、事件留痕、记得多但不外泄

第 15 章把「持久化用哪个 backend」「checkpointer 怎么建」在配置层声明清楚了。但配置只是写下了意图,真正把一次次运行**存下来、可回放、可观测**的,是另一套运行时机制。这一章我们走进 `runtime/` 分区:看 LangGraph checkpointer 怎么把 graph state 落盘、`RunJournal` 怎么把一次运行的事件流(消息、trace、token)记成账、事件 store 三种 backend 各自的取舍,以及最关键的一条主线——可观测性永远在「记得够多」和「不泄露/不撑爆」之间设防:一个超长的 trace、一张几 MB 的 base64 图、一个崩了的写盘任务,都不该污染主存储,也不该把数据漏到不该去的地方。

## 两套各管一半的持久化:checkpointer 存「状态」,RunJournal 存「过程」

DeerFlow 的持久化其实是**两条互不相同的线**,容易混淆,先分清楚:

- **checkpointer**(`runtime/checkpointer/provider.py`)存的是 **LangGraph 的 graph state**——也就是「这个 thread 当前的消息列表、promoted 工具集、各 middleware 写进 state 的字段」这份**可恢复的快照**。它的存在让一个 thread 能中断后继续、让 HITL 能挂起再恢复。
- **RunJournal**(`runtime/journal.py`)存的是 **一次运行的事件流**——每个 LLM 请求/响应、每个工具结果、每个 middleware 动作、token 用量,按时间顺序记成一条条 `RunEvent`。它喂的是回放、审计、用量统计、前端「运行进度」展示。

一个是「现在状态长什么样」(便于恢复),一个是「过程发生了什么」(便于观测)。两者 backend 可以不同,但在第 15 章 `database` 配置里被统一收口:sqlite 模式下两者共享同一个 `deerflow.db`(WAL + busy timeout 让单文件双负载安全)。

## checkpointer:三 backend、单例长连接、配缺则降级到内存

`provider.py` 的 `_sync_checkpointer_cm` 按 `config.type` 分三支建 checkpointer:`memory` → `InMemorySaver`(进程内、重启即失);`sqlite` → `SqliteSaver`(惰性 import,缺包则 `raise ImportError(SQLITE_INSTALL)` 带上明确的安装命令);`postgres` → `PostgresSaver`(缺 `connection_string` 直接 `raise ValueError`)。和前几章一样,**缺字段/缺依赖在建连接这一步就炸,且错误消息直接告诉你装哪个包**,而不是留一个 None 在运行时坑你。

`get_checkpointer()` 维护一个**单例 + 长开 context manager**:

```python
checkpointer_ctx = _sync_checkpointer_cm(config)
checkpointer = checkpointer_ctx.__enter__()
_checkpointer_ctx = checkpointer_ctx   # 把打开的 cm 存着,保持连接活着
_checkpointer = checkpointer
```

它把 context manager 显式 `__enter__` 后**把 cm 本身存进 `_checkpointer_ctx`**,而不是用 `with` 块——因为连接要在整个进程生命周期内一直活着,不能出块就关。`reset_checkpointer()` 则反过来 `__exit__(None,None,None)` 关掉连接、清空单例(回顾第 15 章:checkpointer 配置一变,`_apply_singleton_configs` 就调它强制重建)。这正解释了第 15 章为什么 `sandbox`/`checkpointer` 是 startup-only:**单例长连接在启动时建一次,改 config 不会自动重连。**

最能体现「默认即安全、配缺则降级」的是:**没配 checkpointer 时返回 `InMemorySaver` 而不是报错**。

```python
if config is None:
    _checkpointer = InMemorySaver()
    return _checkpointer
```

「不配持久化」是一个合法状态(开发态),系统降级到内存版照样能跑,只是重启丢状态——和第 14 章「没装 MCP 就当没有 MCP」、第 15 章「没写 database 就给单机默认」是同一种姿态:**可选能力缺失就降级,绝不让主路径起不来。** 注释还点了一个并发细节:`ensure_config_loaded()` 刻意放在 `_checkpointer_lock` 之外调用,避免「配置加载会 reset 持久化单例」和「持有 provider 锁」之间形成跨 provider 的锁序倒置。

## RunJournal:把回调流标准化成账,token 按调用方分桶

`RunJournal` 是一个 `BaseCallbackHandler`——它挂在 LangChain 的回调机制上,把 `on_chat_model_start`/`on_llm_end`/`on_tool_end`/`on_chain_*` 这些回调标准化成统一的 `RunEvent` 记录,写进可插拔的 `RunEventStore`。文件头的「Key design decisions」把几个关键取舍讲得很清楚,每一条都是踩过坑的结果:

- **不实现 `on_llm_new_token`**,只在 `on_llm_end` 记完整消息——逐 token 记账既贵又碎,只留完整响应。
- **在 `on_chat_model_start` 抓「第一条 human message」**而非 `on_chain_start`,因为前者只在真正的 LLM 调用时触发、消息结构完整、且不会被 checkpoint 裁剪压缩;后者每个 node 都触发,噪声大。
- **只有 `parent_run_id is None` 的根调用**才 emit `run.start`/`run.end`——嵌套的内部 graph node 也会触发 `on_chain_*`,但只有根链代表用户可见的运行生命周期,其余一律忽略。

最见功力的是 **token 用量的「分桶 + 去重」**。LangChain 可能对同一个 `run_id` 多次触发 `on_llm_end`,直接累加就会重复计数。`RunJournal` 用 `_counted_llm_run_ids` 这个 set 去重——**只有没数过的 run_id 才累加**:

```python
if total_tk > 0 and rid not in self._counted_llm_run_ids:
    self._counted_llm_run_ids.add(rid)
    self._total_tokens += total_tk
    if caller.startswith("subagent:"):
        self._subagent_tokens += total_tk
    elif caller.startswith("middleware:"):
        self._middleware_tokens += total_tk
    else:
        self._lead_agent_tokens += total_tk
```

而且 token 不是记一个总数,是**按调用方分桶**:lead agent、subagent、middleware 各有各的累加器。`_identify_caller` 从回调 tags 里识别身份(`subagent:{name}`/`middleware:{name}`/`lead_agent`),没 tag 就默认 `lead_agent`(主 agent graph 不注入 tag,子 agent 和 middleware 才显式标记自己)。这让事后能回答「这次运行的 token 到底花在主 agent 还是某个子 agent 上」——观测的价值不在「总数」,而在「能归因」。子 agent 的 token 还能通过 `record_external_llm_usage_records` 跨进程汇总进来,同样用 `_counted_external_source_ids` 去重。

`_record_message_summary` 里也藏着一个细节:`last_ai_message`(给持久化的 run row 用作「这次运行最后的回答」)只认 **lead agent 且非空** 的 AI 消息——middleware/子 agent 的模型调用、只带 tool_call 的空 AI 消息,都不准覆盖掉「最后一句有用的回答」。观测字段也要防「被噪声污染」。

## 写盘:缓冲、批量、失败回滚到 buffer、绝不阻塞事件循环

回调方法是**同步**的,但写 store 是**异步**的——这中间的阻抗匹配是 `RunJournal` 写盘逻辑的核心难点。它的策略是「攒批 + best-effort 异步刷」:

`_put` 把事件塞进 `self._buffer`,满 `_flush_threshold`(默认 20)就 `_flush_sync`。而 `_flush_sync` 极其克制:

```python
if self._pending_flush_tasks:
    return   # 已有 flush 在飞 — 避免多个 fire-and-forget 任务并发写同一个 SQLite 文件
try:
    loop = asyncio.get_running_loop()
except RuntimeError:
    return   # 没有事件循环 — 把事件留在 buffer 里,等之后异步 flush
```

两道防线:**有 flush 在飞就不再起新的**(防止并发写同一个 SQLite 文件冲突)、**没有事件循环就把事件留在 buffer**(同步回调可能在没有 loop 的上下文触发,这时不强写,留给 worker 的 `finally` 块里那个异步 `flush()` 兜底)。最关键的容错在 `_flush_async`:

```python
try:
    await self._store.put_batch(batch)
except Exception:
    logger.warning("Failed to flush %d events ... — returning to buffer", ...)
    self._buffer = batch + self._buffer   # 失败的事件放回 buffer 头部,下次重试
```

**写盘失败不丢事件、不抛异常拖垮主流程,而是把这批事件放回 buffer 头部等下次重试。** 这又是一条「观测是增强路径、出问题就降级」的体现:遥测写不进去,最多是这次没记上、下次补,绝不能让「记账失败」反过来打断「正在跑的 agent」。worker 的 `finally` 块里 `await journal.flush()` 做最后的强制刷盘——`flush()` 会先 `gather` 所有在飞的 flush 任务,再把残余 buffer 分批写完,失败则原样 `raise`(收尾阶段才允许把错误暴露出来)。

还有一条独立的「进度快照」线 `_schedule_progress_flush`:节流地(`progress_flush_interval` 默认 5 秒)把当前 token/消息累计推给 `progress_reporter`,让前端能看到**正在运行**的进度,而不必等运行结束。它同样全程 best-effort——推送失败只 warning,不影响主流程。

## 事件 store 三 backend:从开发到生产的取舍,与路径逃逸防御

`RunEventStore` 有三个实现,正好对应第 15 章 `run_events.backend` 的三个值,取舍写在各自的文件头:

- **memory**:进程内,重启即失,开发测试用。
- **db**(`db.py`):SQLAlchemy ORM,支持完整查询,生产用。它对 **trace 类内容按 `max_trace_content`(默认 10240 字节)截断**——`_truncate_trace` 按字节切(多字节字符可能被切断,故 `decode(errors="ignore")`),并在 metadata 里标记 `content_truncated=True` 和 `original_byte_length`。**一条超长的 trace 不该把数据库撑爆**,截断且留痕(标记原长),是「记得多」与「存得起」的平衡。
- **jsonl**(`jsonl.py`):追加写文件,单机轻量持久化。

`jsonl.py` 这个 backend 把「文件系统作为存储」的几个现实风险防得很仔细,值得细看:

**其一,路径逃逸防御。** 文件名直接由 `thread_id`/`run_id` 拼成路径(`.deer-flow/threads/{thread_id}/runs/{run_id}.jsonl`),如果 id 里带 `../` 或 `/` 就能写到目录外。`_validate_id` 用正则 `^[A-Za-z0-9_\-]+$` 把关,**任何不是字母数字短横下划线的 id 一律 `raise ValueError`**——这和后面章节会反复看到的「路径来自外部输入就必须先正则约束」是同一道防线。

**其二,并发写不交错。** 每个 thread 一把 `asyncio.Lock`,序列化同进程内对同一文件的写,防止多个 fire-and-forget 任务把 JSONL 行写得交错。文件头还**诚实地标注了边界**:in-memory seq 计数器是进程本地的,「多进程共享同一目录会产生重复或非单调的 seq——这种部署请用 `DbRunEventStore`」。把适用边界写进文档,而不是假装没有,是可靠组件的基本诚实。

**其三,I/O 不阻塞事件循环。** 所有文件读写都 `asyncio.to_thread` 丢到线程池;读单个 run 文件是快路径(`list_events`),而 `list_messages` 因为要跨多个 run 文件统一 seq 排序,只能扫全 thread 目录——这个慢路径的代价也被如实写在文件头。读取时遇到坏行(`json.JSONDecodeError`)只 `logger.debug` 跳过,不让一行损坏的 JSONL 毁掉整次读取。

## 记得多但不外泄:序列化层的脱敏与瘦身

可观测性的另一半是「数据出口」——把持久化的状态/消息返回给前端或 SSE 时,什么该送、什么不该送。`serialization.py` 是这道关口的唯一真源(注释明确:消费者是 worker 的 SSE 发布和 gateway 的 REST 响应),它做两件「减法」:

**其一,剥掉 LangGraph 内部键。** `serialize_channel_values` 跳过所有 `__pregel_*` 前缀和 `__interrupt__` 键——这些是 LangGraph 引擎内部状态,对前端无意义,且对齐了 LangGraph Platform API 的返回形态。

**其二,剥掉 base64 大图。** `strip_data_url_image_blocks` 是最能体现「记得多但不外泄/不撑爆」的一处。`ViewImageMiddleware` 会把完整的 base64 图片塞进 `hide_from_ui` 的 human message 作为模型上下文——这些数据**是给模型看的内部上下文,不该走线**(响应体巨大、对 UI 无价值)。它精准地只删 `hide_from_ui=True` 消息里 `image_url` 且 URL 以 `data:` 开头的 block,**文本块、`https://` 图、非隐藏消息一概不动**,保证消息顺序和数量不变:

```python
filtered = [block for block in content
    if not (isinstance(block, dict) and block.get("type") == "image_url"
        and isinstance(block.get("image_url"), dict)
        and str(block["image_url"].get("url", "")).startswith("data:"))]
```

`serialize_channel_values_for_api` 把这两步合成一个便捷包装,注释要求「所有返回 channel values 给前端的 REST 端点都用这个」——**让「不把 base64 漏出去」这条保证只在一处实现、处处调用**,和第 14 章 `is_mcp_tool`、第 15 章 `reload_boundary` 把规则收在单一真源是同一种工程纪律。

至于第三方 tracing(`tracing_config.py`),DeerFlow 接 LangSmith / Langfuse,但姿态克制:配置全从**环境变量**读(密钥不进 config.yaml),`enabled` 与 `is_configured` 分开——`enabled=True` 但缺密钥时,`validate()` 会 `raise` 明确报错(「enabled 了却没设 KEY」是部署错误,早炸),而 `is_configured` 则要求 enabled 且密钥齐全才算真正启用。**「想开但没配齐」和「真正生效」被严格区分**,不让一个半配的 tracing 静默地什么都没记。

## 本章小结

- 两套持久化各管一半:checkpointer(`runtime/checkpointer/provider.py`)存 LangGraph graph state(可恢复快照),RunJournal(`runtime/journal.py`)存运行事件流(可观测过程);sqlite 模式下共享 `deerflow.db`(WAL + busy timeout)。
- checkpointer 三 backend(memory/sqlite/postgres),缺包 `raise ImportError` 带安装命令、缺 `connection_string` `raise ValueError`;`get_checkpointer()` 单例 + 长开 context manager(显式 `__enter__`、存 cm 保持连接,`reset_checkpointer` 才 `__exit__`),解释了为何 checkpointer/sandbox 是 startup-only;**没配则降级到 `InMemorySaver` 不报错**;`ensure_config_loaded` 刻意置于 provider 锁外防锁序倒置。
- RunJournal 是 `BaseCallbackHandler`:不记逐 token(只 `on_llm_end` 记完整消息)、在 `on_chat_model_start` 抓首条 human message、只对 `parent_run_id is None` 根调用 emit run.start/end;token **按 caller 分桶**(lead/subagent/middleware)且用 `_counted_llm_run_ids`/`_counted_external_source_ids` set **去重**防重复计数;`last_ai_message` 只认 lead agent 非空 AI 消息防污染。
- 写盘:`_put` 攒批(`_flush_threshold=20`),`_flush_sync` 双防线(已有 flush 在飞则不起新的、防并发写同一 SQLite;无事件循环则留 buffer);`_flush_async` 失败把 batch 放回 buffer 头部重试、不抛错拖垮主流程;worker `finally` 调 `flush()` 强制收尾;独立的进度快照线节流 5s 推送、全程 best-effort。
- 事件 store 三 backend 对应 `run_events.backend`:memory(开发)、db(生产,trace 按 `max_trace_content=10240` 字节截断并标记 `content_truncated`)、jsonl(单机)。jsonl 防三类风险:`_validate_id` 正则 `^[A-Za-z0-9_\-]+$` 防路径逃逸、每 thread `asyncio.Lock` 防交错写、`asyncio.to_thread` 不阻塞 loop;诚实标注「多进程请用 db」的边界、坏行只 debug 跳过。
- 序列化层(`serialization.py`,单一真源)做两道减法:剥 `__pregel_*`/`__interrupt__` 内部键、`strip_data_url_image_blocks` 精准剥 `hide_from_ui` 消息里的 `data:` base64 图(文本/https 图/非隐藏消息不动);`serialize_channel_values_for_api` 合并供所有 REST 端点统一调用。tracing 从环境变量读密钥,`enabled` 与 `is_configured` 严格区分,半配则 `validate()` 报错。

## 动手实验

1. **实验一:推演 token 分桶与去重** — 阅读 `RunJournal.on_llm_end`、`_identify_caller`、`record_external_llm_usage_records`。假设一次运行里:lead agent 调了 2 次 LLM(其中 1 次 LangChain 对同一 run_id 触发了两遍 `on_llm_end`),一个 `subagent:researcher` 调了 1 次。推演:`_counted_llm_run_ids` 怎么防止那次重复计数?最终 `get_completion_data()` 里 `lead_agent_tokens`、`subagent_tokens`、`llm_call_count` 各是多少?为什么 token 要按 caller 分桶而不是只记总数?
2. **实验二:跟踪一次写盘失败的恢复** — 通读 `_put`/`_flush_sync`/`_flush_async`/`flush`。假设 buffer 攒到 20 条触发 `_flush_sync`,但这批 `put_batch` 因为 SQLite 锁竞争抛了异常。推演:这 20 条事件会丢吗?它们去了哪里、何时重试?如果此时根本没有运行的事件循环,`_flush_sync` 会怎么做?worker `finally` 里的 `flush()` 又如何兜底?为什么遥测写失败不能向上抛?
3. **实验三:验证 jsonl 的路径逃逸防御** — 阅读 `JsonlRunEventStore._validate_id`、`_run_file`、`_thread_dir`。假设某个 `thread_id` 传进来是 `"../../etc/cron.d/x"`。推演:`_validate_id` 会发生什么?正则 `^[A-Za-z0-9_\-]+$` 为什么能挡住它?如果去掉这道校验,`_run_file` 会把文件写到哪?再设想 `run_id` 合法但目录里有一行损坏的 JSON,`_read_run_events` 会整体失败还是跳过?
4. **实验四:核对「不外泄 base64」的单一真源** — 阅读 `strip_data_url_image_blocks` 与 `serialize_channel_values_for_api`。假设一个 thread 的消息里有:一条普通文本 human message、一条 `hide_from_ui=True` 且含 `data:image/png;base64,...` 的 message、一条含 `https://...` 图的可见 message。推演:经过 `serialize_channel_values_for_api` 后,这三条各自发生什么变化?消息总数和顺序变了吗?为什么这道剥离要放在序列化层「一处实现、处处调用」,而不是在每个 REST 端点各写一遍?

> **下一章预告**:到这里,DeerFlow 从启动入口、agent 主循环、中间件、模型网关、工具、沙箱、子 agent、记忆、skill、MCP、配置,到持久化与可观测性,几乎每个子系统我们都拆过一遍了。最后一章不再引入新代码,而是回头把这十六章里反复出现的同一批姿态——「默认即安全、覆盖只能更严」「核心必稳、增强可降级」「单一真源、加载期早炸」「外部输入先约束、失败要留痕」——提炼成一组与语言、框架无关、可迁移到任何 agent harness 的工程原则,每一条都回指到前面的具体章节作为证据。
