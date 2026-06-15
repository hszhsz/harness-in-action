# 第 5 章　中间件链：14 段装配顺序与 @Next/@Prev 定位

这是本书的「心脏切片」。前面我们反复说 DeerFlow 的工程意志都焊在中间件链里，这一章就把这条链拆开，一段一段看它怎么装配、为什么是这个顺序、用户怎么往里安全插队。读完这一章，你手里会有一张可以贴在工位上的「DeerFlow 中间件装配图」。

## 两套装配入口，同一套顺序

DeerFlow 有两个地方装配中间件链：`agents/factory.py` 的 `_assemble_from_features`（SDK 级，声明式 feature flag 驱动）和 `agents/lead_agent/agent.py` 的 `build_middlewares`（应用级，运行时配置驱动）。两者装出来的顺序刻意保持一致——`factory.py` 的文档甚至直接写「Middleware order matches `make_lead_agent` (14 middlewares)」。我们以 `_assemble_from_features` 的编号为骨架来读，因为它最清晰。

## 逐段拆解 14 段

```
0-2. Sandbox infrastructure (ThreadData → Uploads → Sandbox)
3.   DanglingToolCallMiddleware (always)
4.   GuardrailMiddleware (guardrail feature)
5.   ToolErrorHandlingMiddleware (always)
6.   SummarizationMiddleware (summarization feature)
7.   TodoMiddleware (plan_mode parameter)
8.   TitleMiddleware (auto_title feature)
9.   MemoryMiddleware (memory feature)
10.  ViewImageMiddleware (vision feature)
11.  SubagentLimitMiddleware (subagent feature)
12.  LoopDetectionMiddleware (loop_detection feature)
13.  ClarificationMiddleware (always last)
```

**[0-2] 沙箱基建三件套**：`ThreadDataMiddleware` → `UploadsMiddleware` → `SandboxMiddleware`，必须严格按这个序。原因在 `build_middlewares` 的顺序注释里：ThreadData 要先跑，才能让 `thread_id` 可用；Uploads 排在 ThreadData 之后，才能拿到 `thread_id`；Sandbox 最后，依赖前两者写进 state 的路径。这是第 4 章「中间件靠 state 通信、有上下游依赖」最直接的证据。注意 `ThreadDataMiddleware(lazy_init=True)` 和 `SandboxMiddleware(lazy_init=True)` 都用了懒初始化——沙箱不在装配时创建，而是真要用时才拉起。

**[3] DanglingToolCallMiddleware `(always)`**：第 4 章详解过，不可关闭，修补悬挂工具调用。它排在沙箱之后、护栏之前——要先把消息历史修良构，后面的环节才不会被畸形格式绊倒。

**[4] GuardrailMiddleware**：内容护栏。有意思的是，`factory.py` 里 `guardrail=True` 会直接抛错：「guardrail=True requires a custom AgentMiddleware instance (no built-in GuardrailMiddleware yet)」。它只接受 `False`（关）或一个自定义实例。这是「没有安全默认值就拒绝糊弄」的态度——护栏这种安全关键的东西，框架不提供一个似是而非的内置默认，逼你显式注入。

**[5] ToolErrorHandlingMiddleware `(always)`**：又一个不可关闭的，把工具抛出的异常转成 `ToolMessage`，让模型看到「工具报错了」而不是让整个图崩掉。它排在护栏之后、摘要之前。

**[6] SummarizationMiddleware**：上下文压缩（第 9 章详解）。同样地，`summarization=True` 也抛错——它需要一个 model 参数，没法零配置造一个默认实例。它排得很靠前，注释解释「SummarizationMiddleware should be early to reduce context before other processing」：先把上下文压下去，后面的处理才省 token。

**[7] TodoMiddleware**：由 `plan_mode` 参数控制，给模型一个 `write_todos` 工具做任务跟踪。

**[8] TitleMiddleware**：在首轮对话后自动生成会话标题。

**[9] MemoryMiddleware**：在 TitleMiddleware 之后，把对话排队送去更新长期记忆（第 11 章）。顺序注释专门说「after TitleMiddleware」。

**[10] ViewImageMiddleware**：视觉。`build_middlewares` 里它有个守卫——只有当 `model_config.supports_vision` 为真时才挂。给一个不支持视觉的模型挂视觉中间件没有意义，所以 DeerFlow 在装配时就按模型能力裁剪。

**[11] SubagentLimitMiddleware**：子 agent 并发限流（第 12 章），截断超量的并行 task 调用。

**[12] LoopDetectionMiddleware**：检测并打断重复工具调用循环（第 10 章），用 `LoopDetectionMiddleware.from_config(...)` 从配置构造。

**[13] ClarificationMiddleware `(always last)`**：永远在队尾，拦截模型的澄清请求。它的「永远在最后」不是说说而已——后面我们会看到框架用代码强制保证。

## ClarificationMiddleware 凭什么钉在队尾

`_assemble_from_features` 在装完内置链、插完用户的额外中间件后，有这样一段：

```python
if extra_middleware:
    _insert_extra(chain, extra_middleware)
    # Invariant: ClarificationMiddleware must always be last.
    # @Next(ClarificationMiddleware) could push it off the tail.
    clar_idx = next(i for i, m in enumerate(chain) if isinstance(m, ClarificationMiddleware))
    if clar_idx != len(chain) - 1:
        chain.append(chain.pop(clar_idx))
```

注释直白：用户的额外中间件如果用了 `@Next(ClarificationMiddleware)`，就会把 Clarification 从队尾顶掉。所以这里做了一次「不变量修复」——找到 ClarificationMiddleware，如果它不在最后，就把它 pop 出来重新 append 到末尾。这是一个把「ClarificationMiddleware 必须最后」从注释级约定提升为运行时强制的小动作。第 17 章会把这类「不变量固化进代码」单列为一条原则。

## @Next / @Prev：让用户安全插队

DeerFlow 允许用户通过 `extra_middleware` 往这条链里加自己的中间件。但「加在哪」是个问题——直接 append 到末尾会破坏 Clarification 的位置，乱插又可能破坏上下游依赖。DeerFlow 的解法是一对定位装饰器，定义在 `agents/features.py`：

```python
def Next(anchor):  # 放在 anchor 之后
    ...
    cls._next_anchor = anchor
def Prev(anchor):  # 放在 anchor 之前
    ...
    cls._prev_anchor = anchor
```

用户给自己的中间件类标上 `@Next(SomeMiddleware)` 或 `@Prev(SomeMiddleware)`，`_insert_extra` 就会把它插到锚点中间件的前后。这个机制比「给个数字索引」健壮得多——你声明的是**语义关系**（「我要在沙箱之后跑」），而不是脆弱的绝对位置（「我要在第 3 位」），后者一旦内置链增减就会错位。

`_insert_extra` 的实现处处是 fail-closed 的防御：

- 一个中间件不能同时有 `@Next` 和 `@Prev`：`raise ValueError(f"{...} cannot have both @Next and @Prev")`。
- 两个额外中间件锚同一个目标（同向或反向）会冲突报错，并提示「use cross-anchoring between extras instead」。
- 无锚点的额外中间件统一插到 ClarificationMiddleware 之前。
- 有锚点的迭代插入，支持「额外中间件互相锚定」（A 锚 B、B 锚内置 C）。
- 迭代若一轮下来没有任何进展，就检测是不是循环依赖（`anchor_types & remaining_types`），是就报「Circular dependency」，否则报「anchors not found」。

这套插入算法把所有可能出错的情况都变成了带可操作提示的报错，而不是默默插到一个错误的位置。错误信息本身就在教用户正确用法——又一次「错误即文档」。

## feature 的三态语义

最后看 `RuntimeFeatures`（`agents/features.py`）。每个 feature 字段不是简单的布尔，而是「布尔 + 实例」的联合类型，对应三种语义：

- `False`：跳过这段中间件。
- `True`：用内置默认中间件（但 `summarization` 和 `guardrail` 不支持 `True`，因为它们没有零配置默认）。
- 一个 `AgentMiddleware` 实例：用这个自定义实现替换。

默认值也透露了 DeerFlow 的取向：`sandbox=True`、`loop_detection=True` 默认开（安全相关默认开），而 `memory`、`subagent`、`vision`、`auto_title`、`summarization`、`guardrail` 默认关。沙箱和循环检测这两个「防失控」的能力是默认在的——这与第 2 章 `merge_sandbox` 的 fail-closed 一脉相承。

## 本章小结

- DeerFlow 有两个装配入口（`_assemble_from_features` 和 `build_middlewares`），刻意保持同一套 14 段顺序。
- 沙箱基建三件套 `ThreadData → Uploads → Sandbox` 必须严格按序，因为存在 `thread_id` / 路径的上下游 state 依赖；ThreadData 与 Sandbox 都用 `lazy_init=True` 懒初始化。
- `DanglingToolCallMiddleware` 与 `ToolErrorHandlingMiddleware` 标注 `(always)`，不可关闭，分别保证消息良构与工具异常转 ToolMessage。
- `guardrail=True` 和 `summarization=True` 都会抛错——这两个安全/上下文关键中间件没有零配置默认，框架逼用户显式注入自定义实例。
- `ViewImageMiddleware` 只在 `model_config.supports_vision` 时才挂，按模型能力裁剪。
- `ClarificationMiddleware` 标注 `(always last)`，并由 `_assemble_from_features` 用「pop 后重新 append」的代码强制钉在队尾，把约定提升为运行时不变量。
- `@Next`/`@Prev` 定位器让用户声明语义位置而非绝对索引；`_insert_extra` 对「同时标注、锚点冲突、循环依赖、锚点缺失」全部给出带提示的报错。
- `RuntimeFeatures` 字段是「False/True/实例」三态；`sandbox` 与 `loop_detection` 默认开，体现「防失控能力默认在」的取向。

## 动手实验

1. **实验一：抄写并验证 14 段顺序** — 打开 `agents/factory.py` 的 `_assemble_from_features`，把 14 段按编号抄下。再打开 `agents/lead_agent/agent.py` 的 `build_middlewares`，逐行对照，确认两者顺序一致（注意 `build_middlewares` 还多挂了 `DynamicContextMiddleware`、`SkillActivationMiddleware`、`TokenUsageMiddleware` 等应用级中间件，想想它们插在哪、为什么）。
2. **实验二：触发一次装配报错** — 阅读 `create_deerflow_agent` 开头的参数校验，找出三种会抛错的非法组合（如同时传 `middleware` 和 `features`）。再读 `_assemble_from_features` 里 `guardrail=True` / `summarization=True` 的分支，确认它们为什么必须传实例。
3. **实验三：手推一次 @Next 插入** — 假设你写了两个额外中间件 A（`@Next(SandboxMiddleware)`）和 B（`@Prev(ClarificationMiddleware)`），照着 `_insert_extra` 的算法在纸上推演它们最终插到链的哪个位置。再构造一个会触发「Circular dependency」报错的例子。
4. **实验四：核对 fail-closed 默认** — 打开 `agents/features.py` 的 `RuntimeFeatures`，列出每个 feature 的默认值。哪些默认 `True`、哪些默认 `False`？把「默认开」的那几个和它们解决的问题（安全？上下文？）对应起来，验证「防失控能力默认在」这个判断。

> **下一章预告**：中间件链装好了，模型节点要真的去调 LLM 了。下一章我们走进 `models/` 分区，看 DeerFlow 如何抽象不同的模型 provider、为什么要给一堆模型打 `patched_*` 补丁、以及那个把流式超时默认设成 240 秒的决定背后藏着什么对「会思考的模型」的体谅。
