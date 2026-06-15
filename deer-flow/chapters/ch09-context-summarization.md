# 第 9 章　上下文压缩：摘要中间件与 skill 保留策略

副作用被关进笼子之后，还有一个会悄悄膨胀、最终拖垮一切的东西——上下文。每一轮「模型→工具→模型」都往消息历史里堆东西：工具输出、文件内容、思考过程。窗口总会满。满了之后怎么办？最朴素的做法是「砍掉旧消息」，但那会丢掉重要信息。DeerFlow 的答案是**摘要**——把旧对话压成一段总结、保留近期原文。但「压缩谁、保留谁」里藏着一个 super agent 特有的难题：刚加载进来的 skill 文件可能很大，又恰恰是模型当前最需要的，绝不能被一刀压掉。这一章我们拆开 `DeerFlowSummarizationMiddleware`，看它怎么在压缩的同时把 skill「捞」出来。

## 触发与保留：两套可配的阈值

摘要的开关和阈值定义在 `SummarizationConfig`（`config/summarization_config.py`）。它默认 `enabled=False`——呼应第 5 章说过的「`summarization=True` 会抛错，因为它需要 model 没法零配置」。配置的核心是两组 `ContextSize`，每组都能用三种单位表达：

```python
ContextSizeType = Literal["fraction", "tokens", "messages"]
```

- `trigger`：什么时候触发压缩。可以是「50 条消息」「4000 token」或「模型最大输入的 80%（fraction=0.8）」，支持配多个阈值，任一满足即触发。
- `keep`：压缩后保留多少近期历史。默认 `messages, 20`（保留最近 20 条）。

还有 `trim_tokens_to_summarize`（默认 4000）控制「喂给摘要 LLM 的消息最多保留多少 token」——连「给摘要本身的输入」都设了上限，防止摘要这一步自己又爆掉。用三种单位（fraction/tokens/messages）统一表达上下文尺寸，是个很灵活的设计：fraction 让阈值随模型窗口自适应，tokens/messages 给精确控制。

## 压缩的主流程：删全部，写摘要，接回保留

`DeerFlowSummarizationMiddleware` 继承自 LangChain 的 `SummarizationMiddleware`，把压缩逻辑挂在 `before_model` 钩子上（第 4 章讲过的钩子之一）。主流程 `_maybe_summarize` 读起来很清晰：

1. `token_counter(messages)` 数总 token，`_should_summarize` 判断是否触发，没到阈值就返回 None（什么都不做）。
2. `_determine_cutoff_index` 算出「切到哪」——切点之前的去摘要，之后的保留。
3. `_partition_with_skill_rescue` 按切点切分，并把近期 skill 捞回保留区（下节详解）。
4. `_preserve_dynamic_context_reminders` 把隐藏的动态上下文提醒挪出压缩区。
5. `_fire_hooks` 触发压缩前钩子。
6. `_create_summary` 调 LLM 生成摘要。
7. 返回新的消息列表。

返回的结构很有意思：

```python
return {"messages": [
    RemoveMessage(id=REMOVE_ALL_MESSAGES),
    *new_messages,      # 摘要消息
    *preserved_messages, # 保留的近期原文
]}
```

它先用 `RemoveMessage(id=REMOVE_ALL_MESSAGES)` **删掉全部**消息，再写入「摘要 + 保留区」。这是「整体替换」而非「增量删除」——把消息历史彻底重排成「一段摘要打头，后面跟近期原文」的干净结构。摘要消息被特意命名为 `name="summary"`（`_build_new_messages`），注释说明前端不显示它、但模型仍把它当上下文用——一个对用户隐藏、对模型可见的「记忆压缩块」。

## 不让摘要的 token 流污染前端

`__init__` 里有一段很细的工程处理，体现了「中间件里调 LLM」的隐藏坑。摘要这步要调一次 LLM，而这次调用发生在 LangGraph 中间件钩子里——它的 token 流会被「消息流回调」捕获，当成一条幻影 AI 消息广播给前端。DeerFlow 的解法是给摘要专用的 model 副本打 `TAG_NOSTREAM` 标签：

```python
existing_tags = list((getattr(self.model, "config", None) or {}).get("tags") or [])
merged_tags = [*existing_tags, TAG_NOSTREAM] if TAG_NOSTREAM not in existing_tags else existing_tags
self._summary_model = self.model.with_config(tags=merged_tags)
```

注释里有两处考究：其一，**不在实例级别替换 `self.model`**——因为 agent/中间件是跨并发 run 缓存复用的，临时替换会在 `await` 期间把 `RunnableBinding` 泄露给别的协程，破坏父类检查原始 model（`profile` / `_get_ls_params`）的逻辑。所以另起一个 `_summary_model` 副本。其二，合并 tags 时保留模型上已有的 tags（比如 `lead_agent/agent.py` 给摘要设的 `middleware:summarize`，用于 RunJournal 归因），因为 `with_config` 是浅合并，直接赋值会把已有 tags 整个覆盖。一个看似只是「加个标签」的动作，背后是并发安全和遥测归因两重考量。

## skill rescue：把近期加载的 skill 从压缩里捞出来

这是本章的核心，也是 super agent 特有的工程。问题是这样的：模型刚用 `read_file` 加载了一个 `/mnt/skills/...` 下的 skill 文件（可能几千 token），正打算照着它干活。如果这时上下文触发压缩，默认的「按切点切分」会把这个 skill 的内容连同旧消息一起压成一句摘要——模型刚拿到的「操作手册」就被撕掉了。

`_partition_with_skill_rescue` 解决这个问题：先按父类逻辑切分，然后在「本该被压缩」的那部分里，找出近期 skill 加载的「捆」（bundle），把它们捞回保留区。

`_find_skill_bundles` 负责识别这些 bundle：扫描消息，找「带 skill 文件读取 tool_call 的 AIMessage」+ 其紧随的配对 `ToolMessage`。判断一个 tool_call 是不是 skill 读取，靠 `_is_skill_tool_call`——工具名在 `skill_file_read_tool_names`（默认 `{"read_file", "read", "view", "cat"}`）里，且读的路径在 skills 根目录（默认 `/mnt/skills`）之下。每个 bundle 记下它占多少 token、对应哪个 skill、tool_call 的 id 等。

`_select_bundles_to_rescue` 决定捞哪些，遵守三道预算，**从最新往旧走**：

```python
for bundle in reversed(bundles):
    if kept >= self._preserve_recent_skill_count: break       # 数量上限（默认 5）
    if bundle.skill_key in seen_skill_keys: continue          # 同一 skill 去重
    if bundle.skill_tool_tokens > self._preserve_recent_skill_tokens_per_skill: continue  # 单 skill 上限（默认 5000）
    if total_tokens + bundle.skill_tool_tokens > self._preserve_recent_skill_tokens: continue  # 总预算（默认 25000）
    selected.append(bundle); ...
```

三道预算各司其职：`preserve_recent_skill_count`（最多捞几个）、`preserve_recent_skill_tokens_per_skill`（单个 skill 太大就不捞，免得一个巨型 skill 吃光预算）、`preserve_recent_skill_tokens`（总预算封顶）。还有 `seen_skill_keys` 去重——同一个 skill 被读多次只捞最新一次。这套「newest-first + 多重预算 + 去重」的选择算法，正是「保留结构、移除内容」原则的精细体现：要省 token，但要省得聪明，别把模型当前正依赖的操作手册省掉。

## 拆分 AIMessage：只捞 skill 那部分 tool_call

最精细的一手在 `_partition_with_skill_rescue` 的循环里。一个 AIMessage 可能同时带「读 skill 的 tool_call」和「别的 tool_call」。DeerFlow 不会因为它带了一个 skill 读取就整条保留，而是**把这条 AIMessage 一分为二**：

```python
rescued_tool_calls = [tc for tc in msg.tool_calls if tc.get("id") in bundle.skill_tool_call_ids]
remaining_tool_calls = [tc for tc in msg.tool_calls if tc.get("id") not in bundle.skill_tool_call_ids]
if rescued_tool_calls:
    rescued.append(_clone_ai_message(msg, rescued_tool_calls, content=""))
if remaining_tool_calls or msg.content:
    remaining.append(_clone_ai_message(msg, remaining_tool_calls))
```

skill 相关的 tool_call 克隆一份进保留区（content 清空，只留 tool_call 和对应结果），其余的 tool_call 和原 content 留在压缩区。这样保留区里只有「skill 读取这一动作 + 它的结果」，干干净净；模型既看到了 skill 内容，又不会因为消息结构断裂而困惑。这种「外科手术式」的消息拆分，背后是对「OpenAI 兼容校验器要求 tool_call 与 ToolMessage 配对」的深刻理解——呼应第 4 章 `DanglingToolCallMiddleware` 的关切。

## fail-safe：rescue 出错就退回默认切分

skill rescue 是个相对复杂的逻辑，万一出 bug 怎么办？`_partition_with_skill_rescue` 把整段 rescue 包在 try/except 里：

```python
try:
    bundles = self._find_skill_bundles(to_summarize, self._skills_container_path)
except Exception:
    logger.exception("Skill-preserving summarization rescue failed; falling back to default partition")
    return to_summarize, preserved
```

rescue 失败就记日志、退回父类的默认切分——压缩照常进行，只是不捞 skill。**一个增强功能的失败不该拖垮基础功能**：哪怕 skill 保留逻辑崩了，摘要本身还能正常压缩上下文、让 run 继续。这与第 3 章持久化「best-effort 更新只 warning 不抛错」的可靠性分级是同一种思路——核心路径必须稳，增强路径出错降级。

## 本章小结

- 摘要由 `DeerFlowSummarizationMiddleware`（继承 LangChain `SummarizationMiddleware`）挂在 `before_model` 实现；默认 `enabled=False`（需 model 参数，无零配置默认）。
- `SummarizationConfig` 用 `trigger`/`keep` 两组 `ContextSize` 控制触发与保留，单位可为 `fraction`/`tokens`/`messages`；`trim_tokens_to_summarize`（默认 4000）限制喂给摘要 LLM 的输入。
- 压缩是「整体替换」：`RemoveMessage(REMOVE_ALL_MESSAGES)` 删全部，再写入「摘要（`name="summary"`，前端隐藏、模型可见）+ 保留区」。
- 摘要专用 model 副本打 `TAG_NOSTREAM` 防止 token 流被当幻影消息广播；刻意不替换 `self.model`（并发安全），合并 tags 时保留已有 tags（遥测归因）。
- skill rescue 是核心：`_find_skill_bundles` 识别「skill 读取 AIMessage + 配对 ToolMessage」捆，`_select_bundles_to_rescue` 在 `preserve_recent_skill_count`（5）/`_per_skill`（5000）/总 `_tokens`（25000）三道预算下 newest-first 选择并去重。
- AIMessage 被外科手术式拆分：skill 相关 tool_call 进保留区（content 清空），其余留压缩区，保证 tool_call 与 ToolMessage 配对不破裂。
- fail-safe：rescue 抛异常则记日志退回默认切分——增强功能失败不拖垮基础压缩。

## 动手实验

1. **实验一：算一次触发阈值** — 阅读 `SummarizationConfig`，假设配 `trigger={'type':'fraction','value':0.8}`、模型最大输入 128000 token，求触发摘要的 token 数。再配 `keep={'type':'messages','value':20}`，说明压缩后保留什么。
2. **实验二：跟踪一次 skill rescue** — 通读 `_find_skill_bundles` 与 `_select_bundles_to_rescue`，构造一段含 3 次 skill 读取（其中一次 8000 token、超过 per-skill 上限）的历史，推演哪些 bundle 会被捞、哪些被压。
3. **实验三：理解消息拆分** — 看 `_partition_with_skill_rescue` 中拆分 AIMessage 的分支，回答：当一条 AIMessage 同时带「read_file /mnt/skills/x」和「bash ls」两个 tool_call 时，压缩后这条消息会被拆成几条、分别进哪个区？为什么 content 要清空？
4. **实验四：验证 fail-safe 降级** — 找到 `_partition_with_skill_rescue` 的 try/except，说明 rescue 抛异常时函数返回什么、摘要还能不能正常进行。对比第 3 章持久化的「best-effort」分级，总结 DeerFlow 的「核心稳、增强降级」模式。

> **下一章预告**：摘要管的是「上下文太长」，但还有一类失控是「模型陷进死循环」——把同一个工具调 20 次、或工具吐出一坨吞掉整个窗口的输出。下一章我们看 `LoopDetectionMiddleware` 如何发现并打断重复劳动、工具输出预算如何给单次输出封顶，以及 DeerFlow 怎样在「防失控」与「别误杀正常重试」之间取舍。
