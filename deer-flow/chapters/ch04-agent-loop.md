# 第 4 章　Agent 主循环：create_agent、ThreadState 与中间件钩子

前两章我们反复说 DeerFlow「不自己造主循环，借了 LangChain `create_agent` 的心脏」。这一章我们正面回答两个问题：这颗借来的心脏到底怎么跳？DeerFlow 又是怎么在不改心脏的前提下，在它每一次跳动的间隙塞进自己的逻辑的？答案的关键词是 **中间件钩子（middleware hooks）**——理解了这套钩子，你就理解了 DeerFlow 全部工程能力的「挂载点」。

## 借来的心脏：create_agent 的节拍

回到 `agents/lead_agent/agent.py` 的 `_make_lead_agent`。它最终的返回语句是：

```python
return create_agent(
    model=create_chat_model(name=model_name, thinking_enabled=thinking_enabled, ...),
    tools=final_tools,
    middleware=build_middlewares(config, model_name=model_name, ...),
    system_prompt=apply_prompt_template(...),
    state_schema=ThreadState,
)
```

五个参数，五件事：用哪个模型、能调哪些工具、套哪条中间件链、系统提示是什么、状态结构是什么。`create_agent` 拿到这些，编译出一张 LangGraph 状态图。这张图的核心循环很朴素——LangChain 内部把它实现成「模型节点」和「工具节点」两个节点之间的往返：模型节点请求一次 LLM；如果模型返回了 tool_calls，就跳到工具节点执行；工具结果回填进消息历史，再跳回模型节点；直到模型不再要求调工具，循环结束。

这套「模型 ↔ 工具」的往返就是 agent loop 的本质，DeerFlow 完全没碰它。它把全部精力放在了：**在每一跳进出节点的瞬间，插入自己的钩子。**

## 钩子的五种形态

LangChain 的 `AgentMiddleware` 提供了一组生命周期钩子，DeerFlow 的 14 段中间件全都是它的子类。grep 一遍 `agents/middlewares/` 目录，能看到这些钩子方法被反复实现：

- `before_agent` / `after_agent`：整个 agent 调用的最外圈，进入图之前、退出图之后各跑一次。
- `before_model` / `after_model`：每一次模型调用的前后。
- `wrap_model_call`：把模型调用整个「包」起来——可以在调用前改请求、调用后改响应，甚至决定要不要真的调。
- `wrap_tool_call`：同理，把单个工具调用包起来。
- 每个钩子都有一个 `a` 前缀的异步孪生（`abefore_model`、`aafter_agent`、`aafter_model`…），用于 async 执行路径。

这套钩子的强大之处在于「包裹」语义。以 `before_model`（在模型看到消息前修改状态）和 `wrap_model_call`（完全接管这次模型调用）为例，DeerFlow 在两者之间做过精确的取舍。看 `DanglingToolCallMiddleware` 的模块注释，它解释了为什么用 `wrap_model_call` 而不是 `before_model`：

> Uses wrap_model_call instead of before_model to ensure patches are inserted at the correct positions (immediately after each dangling AIMessage), not appended to the end of the message list as before_model + add_messages reducer would do.

这句话信息量很大：`before_model` 返回的状态更新会经过 `add_messages` reducer，而那个 reducer 只会把新消息**追加到末尾**；但修补「悬挂工具调用」需要把合成的 `ToolMessage` 插到**每个出问题的 AIMessage 紧后面**，位置不能错。所以这里必须用 `wrap_model_call` 拿到完整的请求、自己重排消息列表。**钩子选型本身就是工程决策**——选错钩子，逻辑就会被 reducer 默默搬到错误的位置。

## 一个具体的钩子：修补悬挂的工具调用

`DanglingToolCallMiddleware`（`agents/middlewares/dangling_tool_call_middleware.py`，约 205 行）是理解钩子价值的最佳样本，也是第 5 章里标着 `(always)` 的不可关闭中间件之一。

它解决的问题很真实：当一个 `AIMessage` 带了 `tool_calls`，但历史里没有对应的 `ToolMessage`（比如用户中途打断、请求被取消），消息序列就「悬挂」了。严格的 OpenAI 兼容校验器会因为这个不完整的格式直接返回 400。这个中间件在模型调用前扫描历史，给每一个悬挂的 tool_call 插入一条合成的、带错误标记的 `ToolMessage`，让模型收到的对话始终是良构的。

细节里有两处 fail-safe 值得记下。其一，它连 LangChain 存在 `invalid_tool_calls` 里的「畸形工具调用」都一并当作悬挂处理——因为 provider 适配器可能把这些调用的 id/name 序列化进下一次请求，让严格校验器期待一个匹配的 ToolMessage。注释把这个隐蔽的坑写清楚了：「Treat them as dangling calls so the next model request stays well-formed and the model sees a recoverable tool error instead of another provider 400.」其二，它定义了 `_MAX_RECOVERY_ERROR_DETAIL_LEN = 500`，把合成 ToolMessage 里的错误详情截断到 500 字符——注释解释这是为了应对 issue #2894：畸形的 `write_file` 调用可能在无效参数里塞进巨大的 Markdown 负载，不截断就会把这堆垃圾原样回吐给模型。

一个不足 210 行的中间件，背后是两个真实 issue 的疤痕。这就是 harness 工程的质地：它不优雅，但每一道补丁都有来历。

## 状态怎么流动：ThreadState 是钩子之间的总线

钩子能改什么？答案是 `ThreadState`——第 2 章我们已经见过它。每个钩子方法的签名都是 `(self, state, runtime)`，返回一个 `dict | None`。返回的 dict 会被 LangGraph 按各字段的 reducer 合并回 `ThreadState`。这就是中间件之间通信的方式：上游中间件往 state 里写（比如 `ThreadDataMiddleware` 写 `workspace_path`、`uploads_path`、`outputs_path`），下游中间件读。

这解释了第 5 章会详谈的装配顺序为什么至关重要。`build_middlewares` 的注释里写着：「ThreadDataMiddleware must be before SandboxMiddleware to ensure thread_id is available」「UploadsMiddleware should be after ThreadDataMiddleware to access thread_id」。中间件之间是有数据依赖的——A 写进 state 的东西，B 才能读到。顺序错了，下游就读到空值。中间件链不是一堆独立插件的随意堆叠，而是一条有严格上下游关系的流水线。

## 静态系统提示 + 动态注入：为了 prefix cache

还有一个钩子用法体现了 DeerFlow 对成本的敏感。`build_middlewares` 里挂了一个 `DynamicContextMiddleware`，注释解释它的职责：

> Always inject current date (and optionally memory) as `<system-reminder>` into the first HumanMessage to keep the system prompt fully static for prefix-cache reuse.

也就是说，当前日期、记忆这类「每次都变」的动态内容，DeerFlow **不**塞进 system prompt，而是注入到第一条 `HumanMessage` 里。为什么？因为 system prompt 一旦带了变量就每次都不同，模型提供方的 prefix cache（前缀缓存）就失效了。把 system prompt 钉死成静态，动态内容旁路注入，就能持续命中前缀缓存、省下真金白银。这是一个纯粹由「省 token」驱动的架构决策，而它的落地点，又是一个中间件钩子。

## 本章小结

- `_make_lead_agent` 用五个参数（model / tools / middleware / system_prompt / state_schema）调用 `create_agent`，编译出一张 LangGraph 图；agent loop 的「模型 ↔ 工具」往返完全由 LangChain 提供，DeerFlow 不碰。
- DeerFlow 全部工程能力都挂在 LangChain 的中间件钩子上：`before/after_agent`、`before/after_model`、`wrap_model_call`、`wrap_tool_call`，每个都有 `a` 前缀的异步孪生。
- 钩子选型是工程决策：`DanglingToolCallMiddleware` 必须用 `wrap_model_call` 而非 `before_model`，因为后者经过 `add_messages` reducer 只能追加到末尾，无法把合成 ToolMessage 插到精确位置。
- 该中间件是 `(always)` 不可关闭的，它把畸形的 `invalid_tool_calls` 也当悬挂处理，并用 `_MAX_RECOVERY_ERROR_DETAIL_LEN = 500` 截断错误详情——两处补丁分别对应真实 provider 400 和 issue #2894。
- 中间件之间通过 `ThreadState` 通信：钩子返回的 dict 经各字段 reducer 合并回 state；因此中间件有上下游数据依赖，装配顺序不能乱（如 ThreadData 必须在 Sandbox 之前）。
- `DynamicContextMiddleware` 把当前日期、记忆等动态内容注入第一条 HumanMessage 而非 system prompt，目的是保持 system prompt 静态以命中 provider 的 prefix cache，省 token。

## 动手实验

1. **实验一：清点钩子方法** — 在 `agents/middlewares/` 目录用 `grep -rn "def before_model\|def after_model\|def wrap_model_call\|def wrap_tool_call\|def before_agent\|def after_agent"` 统计每种钩子各被多少个中间件实现。哪种钩子用得最多？据此推测 DeerFlow 最常在 agent loop 的哪个位置插手。
2. **实验二：复现一次悬挂修补** — 通读 `dangling_tool_call_middleware.py`，画出它的算法：如何扫描出「带 tool_calls 但无对应 ToolMessage」的 AIMessage，又如何把合成 ToolMessage 插到正确位置。然后回答：如果改用 `before_model` 实现，会插错到哪里？
3. **实验三：验证状态依赖** — 在 `agents/lead_agent/agent.py` 的 `build_middlewares` 上方读那段顺序注释，列出至少三条「A 必须在 B 之前」的依赖，并到对应中间件源码里找出 A 写了哪个 state 字段、B 读了哪个，确认依赖真实存在。
4. **实验四：理解 prefix-cache 优化** — 找到 `DynamicContextMiddleware` 的实现，确认它把日期/记忆注入到第一条 HumanMessage 而非 system prompt。然后思考：如果你的项目里 system prompt 包含了 `今天是 {date}`，会对模型提供方的前缀缓存命中率造成什么影响？如何改造？

> **下一章预告**：钩子机制已经清楚，但 14 段中间件**为什么是这个顺序**？下一章我们逐段拆解 `_assemble_from_features` 与 `build_middlewares` 的装配逻辑，看 `@Next`/`@Prev` 定位器如何让用户安全地往这条流水线里插队，以及为什么 `ClarificationMiddleware` 无论如何都要被钉在队尾。
