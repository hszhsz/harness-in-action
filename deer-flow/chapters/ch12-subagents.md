# 第 12 章　子 agent 系统：任务委派、隔离与并发封顶

记忆让 agent 跨会话变聪明，但单个 agent 的「脑容量」终究有限。面对一个需要并行调研十个子问题、或一次跑几十条命令把主上下文搅成一锅粥的大任务，super agent 的答案是**分身**——派出一队子 agent，每个各管一摊、在自己独立的上下文里干活、干完只把一句结论交回来。DeerFlow 把这套能力收束在一个叫 `task` 的工具里。这一章我们拆开 `tools/builtins/task_tool.py`、`subagents/` 整个分区，看它怎么派生子 agent、怎么用「背景执行 + 轮询」不阻塞主流程、子 agent 为什么默认禁掉 `task` 工具（防止无限套娃）、bash 子 agent 为什么沿用第 8 章的 fail-closed、以及主 agent 怎样用一道中间件把「一次派太多分身」的冲动按回去。

## task 工具:一次委派的完整生命周期

子 agent 对模型而言只有一个入口——`task` 工具（`task_tool.py`），签名很克制：

```python
@tool("task", parse_docstring=True)
async def task_tool(
    runtime: Runtime,
    description: str,
    prompt: str,
    subagent_type: str,
    tool_call_id: Annotated[str, InjectedToolCallId],
) -> str:
```

`description` 是给日志/前端卡片看的「3~5 词短描述」，`prompt` 是真正交给子 agent 的任务，`subagent_type` 选哪种分身。`tool_call_id` 是 LangChain 注入的——它后面会被**复用成子 agent 的 task_id**，让「这次工具调用」和「这个后台任务」共享同一个标识，全链路可追溯。

内置两种子 agent 类型：`general-purpose`（复杂多步、又探索又动手）和 `bash`（命令执行专家）。`subagent_type` 传了个不认识的名字，工具不会默默失败，而是把所有可用类型列出来回给模型:`Error: Unknown subagent type '{subagent_type}'. Available: {available}`——又一次「错误即文档」。

## bash 子 agent:把第 8 章的 fail-closed 再拦一道

`task_tool` 里有一段直接呼应第 8 章的安全策略:

```python
if subagent_type == "bash":
    host_bash_allowed = is_host_bash_allowed(runtime_app_config) if runtime_app_config is not None else is_host_bash_allowed()
    if not host_bash_allowed:
        return f"Error: {LOCAL_BASH_SUBAGENT_DISABLED_MESSAGE}"
```

也就是说——当用的是不安全的本地沙箱、且没显式开 `sandbox.allow_host_bash`，请求一个 bash 子 agent 会被直接拒掉，返回那句 `LOCAL_BASH_SUBAGENT_DISABLED_MESSAGE`（原因 + 两条出路）。这正是第 8 章说的「同一条安全策略在工具暴露层、子 agent 层、执行层各拦一次」里的**子 agent 层**。而且更上游的 `registry.py` 还有一道 `get_available_subagent_names`：host bash 不允许时，干脆把 `bash` 从「可用子 agent 名单」里摘掉——模型连「有个 bash 子 agent」都看不到。先在名单层隐藏、再在调用层兜底，纵深防御。

## 背景执行 + 轮询:主流程不被子 agent 卡住

`task_tool` 最核心的设计是**永远异步**。它不调 `executor.execute()`(同步阻塞),而是 `executor.execute_async(prompt, task_id=tool_call_id)`——把子 agent 丢进后台线程池立刻返回一个 task_id，然后自己在一个轮询循环里等结果:

```python
max_poll_count = (config.timeout_seconds + 60) // 5
...
while True:
    result = get_background_task_result(task_id)
    ...
    await asyncio.sleep(5)
    poll_count += 1
    if poll_count > max_poll_count:
        ...
```

每 5 秒查一次后台任务状态,轮询上限是「执行超时 + 60 秒缓冲」换算成的轮数。为什么要在后台轮询而不是让 LLM 自己反复调「查状态」工具?注释点明了:`Poll for task completion in backend (removes need for LLM to poll)`——把轮询沉到后端,模型只看到一次 `task` 调用、最后拿到一句结果,不必为「催进度」浪费一轮轮推理。

轮询期间它还通过 `get_stream_writer()` 往前端推一串 SSE 事件:开始时 `task_started`,子 agent 每产出一条新 AI 消息就 `task_running`(带 `message_index`/`total_messages` 让前端显示进度),终态分别是 `task_completed`/`task_failed`/`task_cancelled`/`task_timed_out`。这样用户能实时看到分身在干什么,而主 agent 的上下文里只会留下最终那一句。

五种正常返回字符串(`Task Succeeded. Result:`、`Task failed. Error:`、`Task cancelled by user.`、`Task timed out. Error:`、`Task polling timed out...`)不是随手写的——它们和 `status_contract.py` 的前缀表严格对应(下节详述)。

## 两层超时:线程池超时 + 轮询超时兜底

子 agent 的超时其实有**两道**。第一道在 `executor.execute_async` 的后台线程里:`execution_future.result(timeout=self.config.timeout_seconds)` 一旦超时,就 set `cancel_event`、把任务标成 `TIMED_OUT`、cancel 掉 future。第二道是 `task_tool` 轮询侧的 `max_poll_count` ——注释说得很直白,这是「安全网」,防的是「线程池超时万一没生效、后台任务卡死」的边缘情况。两道都触发不了的极端情况下,轮询超时会 `request_cancel_background_task` 发协作取消信号、再 `_schedule_deferred_subagent_cleanup` 排一个延迟清理,而不是把卡死的任务条目永远留在内存里。这种「主超时 + 兜底超时」的双保险,是对「后台异步任务可能以各种方式卡住」这一边界的清醒认知。

## 协作式取消:不能强杀线程,就在迭代边界停

子 agent 跑在线程池里,而 Python 线程**没法被强行 kill**。DeerFlow 的取消是**协作式**的:`request_cancel_background_task` 只是 set 一个 `threading.Event`(`cancel_event`),真正的停止发生在 `_aexecute` 的 `agent.astream()` 循环里——每吐出一个 chunk 就检查一次 `if result.cancel_event.is_set()`,是就停。注释诚实地标注了它的局限:

> cancellation is only detected at astream iteration boundaries, so long-running tool calls within a single iteration will not be interrupted until the next chunk is yielded.

也就是说,一个正在跑的长命令不会被立刻打断,得等它这一步结束、下一个 chunk 出来才停。把「能取消到什么粒度」摊开写清楚,而不是假装取消是即时的。

当主 agent 自己被取消(`asyncio.CancelledError`)时,`task_tool` 的处理尤其讲究:它先发协作取消信号,然后用 `await asyncio.shield(_await_subagent_terminal(...))` **被保护地**等子 agent 走到终态——为的是「在父 worker 持久化 `get_completion_data()` 之前,把子 agent 的最终 token 用量快照报上去」。哪怕自己正在被取消,也要先把账算清。这与第 11 章「best-effort 但要尽量不丢」的可靠性分级一脉相承。

## 结构化状态契约:别再靠字符串前缀猜状态

`status_contract.py` 是一段「踩坑后立的规矩」(issue #3146)。早先前端是靠**字符串匹配** `task` 工具结果的开头文字来决定子任务卡片的状态的——后端任何一次措辞改动都会悄悄弄坏卡片生命周期。新方案改成在 `ToolMessage.additional_kwargs` 里塞一个结构化的小契约:`subagent_status`(五个枚举值之一)+ 可选的 `subagent_error`。

后端的 `extract_subagent_status` 用一张**最具体优先**排序的前缀表把结果文字映射成状态:

```python
_PREFIX_TO_STATUS = (
    ("Task Succeeded. Result:", "completed"),
    ("Task polling timed out", "polling_timed_out"),
    ("Task timed out", "timed_out"),
    ("Task cancelled by user", "cancelled"),
    ("Task failed.", "failed"),
    ("Error", "failed"),
)
```

排序「最具体优先」是因为有些前缀是另一些的子串(`Task timed out` vs `Task polling timed out`)——顺序排错就会把 polling 超时误判成普通超时。`make_subagent_additional_kwargs` 还守了一条:status 不在合法枚举里直接 `raise ValueError`——「打错一个字也宁可炸,也不要悄悄退化回老的前缀 fallback」。共享的 `contracts/subagent_status_contract.json` 是前后端共同的「单一事实源」,两边的测试都加载它做断言。这是把一个脆弱的「文本形契约」升级成「结构化契约 + 共享 fixture」的范例。

## 隔离一:子 agent 默认禁 task,防止无限套娃

子 agent 最危险的失控是**递归自我派生**——子 agent 再派子 agent、子子 agent 再派……DeerFlow 用两道闸堵死它。第一道在 `task_tool` 构建子 agent 工具时:

```python
available_tools_kwargs = {
    "model_name": effective_model,
    "groups": parent_tool_groups,
    "subagent_enabled": False,
}
tools = get_available_tools(**available_tools_kwargs)
```

`subagent_enabled=False` 意味着子 agent 拿到的工具列表里**根本没有 `task` 工具**(呼应第 7 章 `get_available_tools` 的按开关挂载)。第二道是兜底:每个子 agent 配置的 `disallowed_tools` 默认就含 `"task"`(`SubagentConfig` 的 `field(default_factory=lambda: ["task"])`,以及两个内置子 agent的 `disallowed_tools=["task", ...]`)。一道在「不暴露」、一道在「显式拒绝」,双保险。

## 隔离二:工具组继承与 skill allowlist 求交

子 agent 不是凭空获得一套全新权限,而是**继承并收窄**父 agent 的权限边界:

- **工具组继承**:`groups=parent_tool_groups`——子 agent 沿用父 agent 的 `tool_groups` 限制,父 agent 不能用的工具组,子 agent 也别想碰。
- **skill allowlist 求交**:`_merge_skill_allowlists(parent, child)` 把父 agent 的可用 skill 集合与子 agent 自己声明的 skill 白名单**取交集**:

```python
def _merge_skill_allowlists(parent, child):
    if parent is None:
        return child
    if child is None:
        return list(parent)
    parent_set = set(parent)
    return [skill for skill in child if skill in parent_set]
```

逻辑是:父没限制就用子的、子没限制就用父的、两个都有就求交——子 agent 永远**不可能**获得比父 agent 更宽的 skill 权限。这正是第 15 章会反复出现的「约束只会收紧、不会放宽」原则在子 agent 边界上的体现。

`SubagentExecutor` 自己还有一层工具过滤 `_filter_tools`(先 allowlist 再 denylist),以及按 `config.skills` 加载 skill 内容并以 `<skill>` SystemMessage 注入——子 agent 在自己的会话里独立加载 skill,而不是从父 agent 那里继承一份。

## 隔离三:独立模型、独立上下文、不落 checkpoint

`SubagentExecutor._create_agent` 用 `create_agent(...)` 起一个**全新**的 agent 实例,几个隔离细节值得记:

- **模型可独立**:`resolve_subagent_model_name` 解析模型名——`config.model` 不是 `"inherit"` 就用它自己的,否则继承父 model,再否则取配置里第一个 model。子 agent 可以用一个比主 agent 更便宜的模型干粗活。
- **`thinking_enabled=False`**:子 agent 默认关掉思考模式,省 token。
- **`checkpointer=False`**:子 agent **不持久化** checkpoint——它是一次性的、用完即弃,状态不该污染主线程的持久化存储(呼应 `test_subagent_checkpointer_isolation`)。
- **复用同一套中间件**:`build_subagent_runtime_middlewares` 让子 agent 和主 agent 共享同一条中间件组合(循环检测、工具预算、安全终止那一整套防失控都在),`lazy_init=True` 懒初始化沙箱。
- **共享父沙箱/线程数据**:`sandbox_state`/`thread_data` 从父 agent 透传过来——子 agent 在同一个沙箱里操作文件,这样它写的文件父 agent 能看到。
- **deferred tool search 一致**:子 agent 同样在「策略过滤之后」装配 `tool_search`(第 7 章),绝不裸绑 MCP schema。

## 跨 event-loop 的执行:一条长命的隔离循环

子 agent 的执行路径藏着一个和第 11 章记忆系统同源的并发坑。`execute_async` 把任务提交到一个**专属的持久化 event loop**(`_get_isolated_subagent_loop`,跑在一个 daemon 线程里、`run_forever`),而不是每次执行都 `asyncio.new_event_loop()` 起一个临时 loop。注释解释:复用一条长命 loop,避免「把共享的异步资源(比如 httpx 客户端)绑到一个用完就关的短命 loop 上」——这正是第 11 章 `model.invoke()` 同步路径要规避的同一类「跨 loop 复用连接」诡异 bug。

提交时还用 `copy_context()` + `context.run(...)` 把父线程的 ContextVar 状态**显式打包**带进隔离 loop——又一次第 11 章讲过的「线程/loop 边界会吃掉 ContextVar,跨边界的数据必须显式带走」。

`try_set_terminal` 则用一把 `_state_lock` 保证「终态只设一次」:后台超时、协作取消、执行 worker 三方可能同时往同一个 `SubagentResult` 写终态,第一个写成功的赢,迟到的终态写入一律被忽略——避免「已经标成 COMPLETED 又被迟到的 TIMED_OUT 覆盖」这种竞态。

## token 用量回流:子 agent 的账记到父 agent 头上

子 agent 烧的 token 不能凭空消失。`SubagentTokenCollector` 是一个挂在子 agent 执行上的 `BaseCallbackHandler`,在 `on_llm_end` 时按 `run_id` 去重累计 input/output/total token。子 agent 结束后,`task_tool._report_subagent_usage` 把这些记录通过 `_find_usage_recorder(runtime)` 找到父 agent 的 RunJournal、调 `record_external_llm_usage_records` 报上去——而且用 `usage_reported` 标志守着「每个子任务只报一次」。

`_find_usage_recorder` 处理 callbacks 可能是 `None`/`list`/`BaseCallbackManager` 三种形态(异步工具运行时是 manager,得先 `.handlers` 解包),其它形态一律当「没有 recorder」而不抛错——对「框架可能用三种形状传同一个东西」这一边界的清醒处理。另外 `_subagent_usage_cache` 按 `tool_call_id` 缓存用量,好让 `TokenUsageMiddleware` 把它写回触发那次调用的 AIMessage 的 `usage_metadata`。一次委派的账,既进了父 RunJournal(审计),又回写进触发它的消息(归因)。

## 并发封顶:一次最多派几个分身

最后一道闸是 `SubagentLimitMiddleware`。模型一时兴起,在一条响应里并排生成十个 `task` 调用怎么办?这道中间件挂在 `after_model`,把超出上限的 `task` 调用**截断**掉:

```python
task_indices = [i for i, tc in enumerate(tool_calls) if tc.get("name") == "task"]
if len(task_indices) <= self.max_concurrent:
    return None
indices_to_drop = set(task_indices[self.max_concurrent:])
truncated_tool_calls = [tc for i, tc in enumerate(tool_calls) if i not in indices_to_drop]
...
updated_msg = clone_ai_message_with_tool_calls(last_msg, truncated_tool_calls)
return {"messages": [updated_msg]}
```

它只保留前 `max_concurrent` 个 `task` 调用、丢掉多出来的,然后用 `clone_ai_message_with_tool_calls`(同 id 触发替换)重建 AIMessage。注意它**只截 `task` 调用**,其它工具调用原样保留。上限默认 `MAX_CONCURRENT_SUBAGENTS=3`,且被 `_clamp_subagent_limit` 钳在 `[2, 4]` 区间——配低于 2 或高于 4 都会被拉回边界。注释说得直白:`This is more reliable than prompt-based limits`——与其在提示里求模型「最多派 3 个」(它经常不听),不如在中间件层硬截。这又是 harness「用代码约束模型,而不是用提示央求模型」的典型姿态。

## 本章小结

- 子 agent 唯一入口是 `task` 工具(`task_tool.py`):`description`/`prompt`/`subagent_type` + 注入的 `tool_call_id`(复用为 task_id);内置 `general-purpose` 与 `bash` 两型,未知类型返回可用列表(错误即文档)。
- bash 子 agent 沿用第 8 章 fail-closed:`is_host_bash_allowed` 为假时 `task_tool` 返回 `LOCAL_BASH_SUBAGENT_DISABLED_MESSAGE`,`registry.get_available_subagent_names` 更在名单层就摘掉 bash——名单隐藏 + 调用兜底两道。
- 永远异步:`execute_async` 丢后台线程池 + `task_tool` 后端轮询(每 5s,上限 `(timeout+60)//5`),免去 LLM 自己催进度;轮询期推 `task_started`/`task_running`/`task_completed/failed/cancelled/timed_out` SSE 事件。
- 双超时:后台 `Future.result(timeout)` 主超时 + 轮询 `max_poll_count` 兜底;取消是协作式(set `cancel_event`,只在 `astream` 迭代边界生效);父被取消时 `asyncio.shield` 等子 agent 终态以先报齐 token 账。
- 结构化状态契约(`status_contract.py`,issue #3146):`subagent_status` 五枚举写进 `additional_kwargs`,`_PREFIX_TO_STATUS` 最具体优先映射结果文字,非法值 `raise`;`contracts/subagent_status_contract.json` 为前后端单一事实源。
- 三重隔离:① 子 agent 禁 `task`(`subagent_enabled=False` 不暴露 + `disallowed_tools` 含 `task` 兜底)防套娃;② 工具组继承 + `_merge_skill_allowlists` 求交(权限只收窄);③ 独立 model/`thinking_enabled=False`/`checkpointer=False`/复用防失控中间件/透传父沙箱。
- 跨 loop 隔离:复用一条持久 `_isolated_subagent_loop`(避免短命 loop 关掉共享 httpx 客户端,同第 11 章),`copy_context` 显式带 ContextVar;`try_set_terminal` 加锁保证终态只设一次(三方竞态)。
- token 回流:`SubagentTokenCollector` 按 run_id 去重累计 → `_report_subagent_usage` 报给父 RunJournal(`usage_reported` 守一次)+ `_subagent_usage_cache` 回写触发 AIMessage 的 usage_metadata。
- `SubagentLimitMiddleware`(after_model)只截超额 `task` 调用、保留其余,`clone_ai_message_with_tool_calls` 同 id 替换;`MAX_CONCURRENT_SUBAGENTS=3` 钳在 `[2,4]`——代码硬截胜过提示央求。

## 动手实验

1. **实验一:推演一次 bash 子 agent 请求** — 阅读 `task_tool.py` 的 bash 分支与 `registry.py` 的 `get_available_subagent_names`。假设部署用 `LocalSandboxProvider` 且未设 `allow_host_bash`,模型仍调 `task(subagent_type="bash", ...)`。问:`get_available_subagent_names` 返回的名单里有没有 `bash`?`task_tool` 最终返回什么字符串?为什么要在「名单层」和「调用层」各拦一次?
2. **实验二:算一次轮询超时** — 看 `task_tool` 的 `max_poll_count = (config.timeout_seconds + 60) // 5` 与轮询循环。假设某子 agent `timeout_seconds=900`,推演:`max_poll_count` 是多少?如果后台线程池的主超时失效、任务卡死,轮询侧第几次会触发 `Task polling timed out`?触发后它做了哪三件事(提示:取消信号、延迟清理、返回值)?
3. **实验三:验证 skill 权限只会收窄** — 阅读 `_merge_skill_allowlists`。给定父 agent `available_skills=["a","b","c"]`、子 agent 配置 `skills=["b","c","d"]`,推演合并结果。再分别推演「父为 None」「子为 None」两种情况。能否构造一组输入让子 agent 拿到父 agent 没有的 skill?为什么不能?
4. **实验四:核对并发封顶的截断** — 阅读 `SubagentLimitMiddleware._truncate_task_calls`。假设某 AIMessage 的 `tool_calls` 依次是 `[task, web_search, task, task, task, write_file]`,`max_concurrent=3`。推演:哪几个 tool_call 被丢、哪几个保留?`web_search` 和 `write_file` 会被影响吗?为什么用 `clone_ai_message_with_tool_calls` 而不是新建一条消息?

> **下一章预告**:子 agent 让能力横向扩展,但每个 agent(无论主从)真正「会干什么专业活」靠的是 skill——一份份写在 `/mnt/skills` 下、按需加载的操作手册。下一章我们走进 skill 系统,看 DeerFlow 如何发现、加载、按 allowlist 裁剪 skill,skill 自带的 `allowed-tools` 如何收窄工具权限,以及 `skill_manage_tool` 怎样让 agent 在运行中「进化」出新技能——又如何防止这种自我修改变成失控的后门。
