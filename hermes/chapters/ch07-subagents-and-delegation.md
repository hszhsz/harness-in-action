# 第 7 章　子 Agent 与任务委派

一个 Agent 能力再强，单线程地一步步推进，效率也有天花板。真正复杂的任务往往可以拆解：调研三个竞品、并行跑五组实验、同时改十个文件里的同一处。如果主 Agent 亲自串行做完每一步，不仅慢，还会把所有中间产物——海量的工具输出、网页内容、文件 diff——全堆进自己的上下文，很快就撑爆窗口。

Hermes 的解法是**委派（delegation）**：主 Agent 派出隔离的"子 Agent"去执行子任务，子任务在自己独立的上下文里折腾，最后只把**结论**交回主 Agent。这一章我们解构 `tools/delegate_tool.py` 的 `delegate_task` 工具，看清子 Agent 如何被派生、如何被隔离、以及为什么委派能实现 README 里所说的"以零上下文成本折叠多步流水线"。

## 7.1 `delegate_task` 的注册与参数面

委派能力以一个普通工具的形式存在——第 4 章讲过的注册机制在这里同样适用。`tools/delegate_tool.py` 约第 2896 行注册了 `delegate_task` 工具，归属工具集 `"delegation"`（对应第 4 章 `TOOLSETS` 里的 `delegation` → `[delegate_task]`）。

它的 handler 是一个 lambda，从 kwargs 里解包出一组很能说明委派语义的参数：`goal`（子任务目标）、`context`（传给子 Agent 的上下文）、`toolsets`（子 Agent 能用哪些工具集）、`tasks`（支持一次派发多个任务）、`max_iterations`（子 Agent 的迭代上限）、`acp_command` / `acp_args`（通过 ACP 协议委派给外部 Agent）、`role`（子 Agent 扮演的角色），以及从 kwargs 透传进来的 `parent_agent`（父 Agent 引用）。

把这组参数读一遍，委派的设计意图就清楚了。`goal` + `context` 定义"做什么、带着什么信息做"；`toolsets` 让父 Agent 能**精确控制子 Agent 的能力面**——这直接复用了第 4 章的工具集分组，比如派一个只做调研的子 Agent，就只给它 `research` 工具集；`max_iterations` 给子 Agent 设独立的预算上限（呼应第 2 章主循环的迭代上限），防止子任务失控；`tasks`（复数）是并行化的入口；`acp_command` / `acp_args` 则把委派的边界从"进程内的子 Agent"扩展到"通过 ACP 协议调用的外部 Agent"。

## 7.2 隔离：子 Agent 为什么不污染主上下文

委派的核心价值是**上下文隔离**。子 Agent 在执行 `goal` 的过程中，会产生大量中间 token——它可能搜十次网、读二十个文件、跑五条命令。这些中间产物全都留在子 Agent 自己的上下文里，主 Agent 完全看不到，最后回到主 Agent 的只有一份凝练的结果。

这就是 README 里"collapsing multi-step pipelines into zero-context-cost turns"的真实含义：从主 Agent 的视角看，一次 `delegate_task` 调用就像一次普通工具调用——发出一个请求，收到一个结果，中间那几十步推理和工具调用的 token 开销，主 Agent 一个字都不用承担。这对第 3 章讲的上下文压缩是绝佳的互补：与其让主 Agent 塞满中间产物再费劲压缩，不如一开始就把会产生大量中间产物的子任务隔离出去。**最好的上下文管理，是让无关内容根本不进入上下文**。

`parent_agent` 这个透传参数则保证了子 Agent 能继承父 Agent 的必要配置（模型、凭证、运行时设置），而不是从零开始——隔离的是上下文，共享的是基础设施。

## 7.3 并行：`tasks` 复数参数的威力

`tasks` 参数支持一次派发多个子任务，这是 Hermes "delegates and parallelizes"卖点的代码落地。当多个子任务彼此独立时（比如分别调研三个不同的主题），它们可以并行跑——总耗时从"三个任务串行相加"压缩到"最慢的那个任务"。

这背后复用的是第 4 章提到的 `tool_executor.py` 的并发执行能力（`execute_tool_calls_concurrent`）。并行委派 + 上下文隔离这两件事叠加起来威力巨大：你既省了时间（并行），又省了上下文（隔离），还能给每个子 Agent 配不同的工具集与迭代预算（精细控制）。一个需要"同时干五件事、每件事都要折腾很多步"的复杂任务，通过委派可以被压缩成主 Agent 的一次并行调用。

## 7.4 委派的边界：从进程内到 ACP

`acp_command` / `acp_args` 参数把委派的概念边界推到了进程之外。ACP（Agent Client Protocol）是一套让不同 Agent 互相调用的协议，第 1 章我们见过 `hermes-acp = "acp_adapter.entry:main"` 这个入口，仓库里也有完整的 `acp_adapter/` 目录（`auth.py`、`edit_approval.py`、`events.py`、`permissions.py`、`provenance.py`、`server.py`、`session.py`、`tools.py`）和 `acp_registry/`。

这意味着 Hermes 的"委派"不局限于派生自己的副本，还能通过 ACP 把任务交给一个完全独立的、可能由别人实现的外部 Agent。`acp_adapter/permissions.py` 与 `edit_approval.py` 的存在说明：跨 Agent 委派时，权限与编辑审批同样要走一遍——你委派出去的任务，不会因为换了执行者就豁免第 5 章那套安全约束。`provenance.py`（来源追踪）则记录"这个动作是谁、经由哪条委派链发起的"，这在多 Agent 协作时对审计至关重要。

把这条线索连起来：Hermes 的委派是一个有层次的能力——进程内派生子 Agent（最常见）、并行派发多个子 Agent（提速）、通过 ACP 委派给外部 Agent（跨系统协作），而无论哪一层，安全约束和来源追踪都不缺席。

## 7.5 委派与主循环的关系

值得点明的是，子 Agent 本身也是一个完整的 Agent——它跑的是同一套第 2 章解构过的 `run_conversation` 主循环，受同样的迭代上限、预算、上下文压缩、错误分类约束，只不过它的 `max_iterations` 由父 Agent 通过 `delegate_task` 的参数指定。这种"子 Agent = 受限的完整 Agent"的设计，意味着 Hermes 不需要为委派单独维护一套简化的执行逻辑——委派复用了主循环的全部成熟机制。

这是一个很经济的架构决策。子 Agent 不是一个"阉割版"的特殊执行器，而是一个被赋予了特定 goal、特定工具集、特定迭代预算的标准 Agent。所有为主循环写的失败恢复、压缩、安全检查，子 Agent 自动全都有。**用同一套内核同时支撑主 Agent 和子 Agent**，既减少了代码重复，也保证了行为一致——你不会遇到"主 Agent 能拦住 `rm -rf /`、子 Agent 却拦不住"这种危险的不一致。

## 本章小结

- 委派能力以普通工具 `delegate_task` 的形式注册（`tools/delegate_tool.py` 约第 2896 行，工具集 `delegation`），复用第 4 章的注册与分组机制。
- 参数面完整表达委派语义：`goal` / `context`（做什么、带什么）、`toolsets`（精确控制子 Agent 能力面）、`max_iterations`（独立迭代预算）、`tasks`（并行多任务）、`acp_command` / `acp_args`（委派给外部 Agent）、`role`、透传的 `parent_agent`。
- 核心价值是上下文隔离：子任务的大量中间 token 留在子 Agent 自己的上下文，主 Agent 只收回结论，实现"零上下文成本折叠多步流水线"——是对第 3 章压缩的最佳互补（让无关内容根本不进上下文）。
- `tasks` 复数参数 + `tool_executor` 并发能力实现并行委派，把"多个独立子任务串行相加"压缩成"最慢的那个"。
- `acp_command` / `acp_args` 把委派边界扩展到进程外：通过 ACP 协议委派给外部 Agent，且 `acp_adapter/` 里的 `permissions.py` / `edit_approval.py` / `provenance.py` 保证跨 Agent 时安全约束与来源追踪不缺席。
- 子 Agent 复用同一套 `run_conversation` 主循环，是"受限的完整 Agent"而非阉割版执行器；用同一内核支撑主/子 Agent，既减少重复又保证行为一致（如安全拦截不会出现主子不一致）。

## 动手实验

1. **实验一：读懂委派参数面** —— `Read` `tools/delegate_tool.py` 约第 2896 行的注册与 handler lambda，逐个解释 `goal` / `context` / `toolsets` / `max_iterations` / `tasks` / `role` 的作用。设计一个真实场景（如"并行调研三个竞品"），写出你会怎么填这些参数。

2. **实验二：论证上下文隔离的收益** —— 假设一个子任务需要读 20 个文件、每个文件 2000 token。计算如果主 Agent 亲自做、这些内容会给主上下文增加多少 token；再说明通过 `delegate_task` 委派后主 Agent 实际承担多少。用这个对比解释"零上下文成本"。

3. **实验三：追踪并行执行** —— 用 Grep 在 `agent/tool_executor.py` 找到 `execute_tool_calls_concurrent`（约第 243 行），结合 `delegate_task` 的 `tasks` 参数，说明多个子任务是如何被并发调度的。思考：什么样的子任务适合并行、什么样的不适合？

4. **实验四：探查 ACP 委派的安全链** —— 浏览 `acp_adapter/` 目录，`Read` `permissions.py` 和 `provenance.py` 的开头。论证：当任务通过 ACP 委派给一个外部 Agent 时，第 5 章的命令审批是否依然生效？`provenance.py` 记录的"来源"在多 Agent 审计中能回答什么问题？

> **下一章预告**：无论主 Agent 还是子 Agent，最终都要调用大模型。但 Hermes 支持几十个 provider、每个 provider 鉴权方式不同、还会遇到限流和封号。第 8 章将解构 `providers/base.py` 的 `ProviderProfile` 抽象与 `agent/credential_pool.py` 的凭证池——看清它如何用四种轮换策略、按错误码区分的冷却 TTL，在多把 key 之间智能调度，把"被限流/封号"从致命故障降级成自动切换。
