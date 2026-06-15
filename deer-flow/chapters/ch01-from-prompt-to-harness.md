# 第 1 章　从 Prompt 到 Harness：为什么要解构 DeerFlow

如果你写过一个能调工具的 LLM 应用，大概率经历过这样的演化：最早是一段 system prompt 加几个 `if` 分支；后来工具多了，你手写了一个 `while` 循环，让模型在「思考—调工具—看结果」之间转圈；再后来，你开始往这个循环里塞各种补丁——超时重试、上下文太长就截断、检测模型是不是卡在同一个工具上反复调、给危险命令加道闸。等你回过神，会发现真正决定这个 Agent 好不好用的，已经不是那段 prompt，而是包在模型外面、由无数补丁堆出来的那层工程。这层工程，就是本书要讲的 **Harness（驾驭层 / 执行骨架）**。

Harness 决定模型能调什么工具、看到什么上下文、在什么边界上被拦下、出错后怎么恢复、副作用能不能落地。同一个大模型，套上不同的 harness，就长成性格迥异的 Agent。本书选 **DeerFlow 2.0** 作为标本，原因有三：它是一次彻底的重写（官方在 README 里明确写着「DeerFlow 2.0 is a ground-up rewrite. It shares no code with v1」），没有历史包袱；它把自己定位成一个「super agent harness」而非某个垂直场景的应用；最关键的，它选择了一条和大多数解构对象都不同的路线——**不自己造主循环**。

## 一个不自己造心脏的 Agent

读过本合集其他几本书的读者会有个直觉：解构一个 Agent，第一步往往是找到那个 `while` 主循环，看它怎么驱动「模型 → 工具 → 模型」的节拍。OpenClaw 有 `queryLoop` 的双层 while，Codex 有自己的状态机。但当你打开 DeerFlow 的 `backend/langgraph.json`，会看到这样一行：

```json
"graphs": {
    "lead_agent": "deerflow.agents:make_lead_agent"
}
```

整个系统对外暴露的「图」只有一个，叫 `lead_agent`，它的构造函数是 `make_lead_agent`。顺着这个函数读进去（`backend/packages/harness/deerflow/agents/lead_agent/agent.py`），你不会找到任何手写的 agent loop。它最终调用的是 LangChain 的 `langchain.agents.create_agent`——主循环是别人（LangGraph）造的。DeerFlow 把心脏外包了。

这不是偷懒，而是一个明确的工程选择。当你不需要维护主循环本身，你就可以把全部工程精力投到「如何在这颗别人的心脏外面，包一层属于自己的约束、安全和可观测性」。DeerFlow 的答案是 **中间件（middleware）**。

## 工程意志，全在那条 14 段的中间件链里

打开 `agents/factory.py`，`_assemble_from_features` 函数的文档字符串里，赫然列着一条 14 段的装配顺序：

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

这就是 DeerFlow 的「性格」所在。每一段中间件都是一个工程决策的物化：沙箱基建负责把文件操作约束进虚拟路径；`DanglingToolCallMiddleware` 在模型看到历史之前修补悬挂的工具调用；`SummarizationMiddleware` 在上下文撑爆之前把它压缩；`LoopDetectionMiddleware` 检测并打断模型的重复劳动；`ClarificationMiddleware` 永远排在最后，拦截模型的澄清请求。注释里有两个细节透露了设计的克制：`DanglingToolCallMiddleware` 和 `ToolErrorHandlingMiddleware` 后面都跟着 `(always)`——它们不可配置、永远在；而 `ClarificationMiddleware` 后面是 `(always last)`，并且代码里专门用了一段逻辑保证它即使被 `@Next` 装饰器推动也会被重新拉回队尾。

本书后面会用整整一章（第 5 章）来解剖这条链，因为读懂它的装配顺序，就读懂了 DeerFlow 为什么是现在这个样子。

## 本书要回答的问题

我们不打算给你一份 DeerFlow 的使用手册——那是官方文档的事。本书要回答的是一连串「为什么」：

- 一个建立在 LangGraph 之上的 Agent，如何在不碰主循环的前提下，把「沙箱、护栏、摘要、记忆、子 agent」这些能力一段段焊进去？（第 4、5 章）
- 当你的部署用的是「不安全的本地沙箱」时，DeerFlow 凭什么默认禁掉 host bash，又留了哪个开关？（第 8 章）
- 上下文要爆了，它压缩谁、保留谁？为什么 skill 的内容有专门的保留预算？（第 9 章）
- 模型陷入「调同一个工具 20 次」的死循环，它怎么发现、怎么打断？（第 10 章）
- 一个 run 跑到一半进程崩了，它靠什么恢复？（第 3、16 章）

每一个问题的答案，都不来自「Agent 一般怎么做」的常识，而来自 DeerFlow 仓库里那一行行真实的 Python。这是本书唯一的铁律：**以源码为唯一真理来源**。写到某个机制前，我们先把那个文件读了，引用确切的函数名、常量名、字面量值、默认值；读不到的，不写。

## 怎么读这本书

本书顺着「一次真实请求穿过整个系统」的链路自外向内展开。如果你时间有限，可以这样取舍：

- 只想搞懂 DeerFlow 的整体骨架 → 读第 2、4、5 章。
- 关心安全与沙箱 → 读第 8、10、15 章。
- 关心上下文工程与成本 → 读第 9、10、11 章。
- 关心扩展机制（自己加 skill / 接 MCP）→ 读第 12、13、14 章。
- 想提炼可迁移的工程原则用到自己的项目 → 直接读第 17 章，再回头按图索骥。

读完之后，你手里会有的不只是「DeerFlow 是怎么实现的」这一份知识，更是一套「如何评判和构建一个可信 Agent harness」的判断力——这套判断力，和你用 Python 还是 TypeScript、用不用 LangGraph，都没关系。

## 本章小结

- DeerFlow 2.0 是一次「从零重写、不与 1.x 共享代码」的工程实践，定位是通用的「super agent harness」。
- 它最显著的工程选择是 **不自己造主循环**：`backend/langgraph.json` 只暴露一个 `lead_agent` 图，构造函数 `make_lead_agent` 最终调用 LangChain 的 `langchain.agents.create_agent`。
- DeerFlow 的工程意志几乎全部物化在一条 **14 段中间件链** 里，其装配顺序记录在 `agents/factory.py` 的 `_assemble_from_features` 文档字符串中。
- 链里有两类中间件：标注 `(always)` 的不可关闭（如 `DanglingToolCallMiddleware`、`ToolErrorHandlingMiddleware`），以及标注 `(always last)` 的 `ClarificationMiddleware`，框架专门保证它永远在队尾。
- 本书的方法论铁律是「以源码为唯一真理来源」，所有论断都引用真实的文件名、函数名、常量与默认值。

## 动手实验

1. **实验一：找到唯一的图入口** — 克隆 deer-flow 仓库，打开 `backend/langgraph.json`，确认 `graphs` 字段下只有 `lead_agent` 一个键，并记录它指向的 `deerflow.agents:make_lead_agent`。思考：为什么一个「能做几乎任何事」的 super agent，对外只暴露一张图？
2. **实验二：数一数中间件** — 打开 `backend/packages/harness/deerflow/agents/factory.py`，找到 `_assemble_from_features` 的文档字符串，把 14 段中间件按编号抄下来。对每一段，猜一猜它解决的是「安全、上下文、还是体验」哪一类问题，后面章节再来验证你的猜测。
3. **实验三：定位「不自己造主循环」的证据** — 在 `agents/lead_agent/agent.py` 里搜索 `create_agent`，确认 `_make_lead_agent` 最终 `return create_agent(...)`。再搜索整个 `deerflow` 包里有没有手写的 `while True` 主循环（用 `grep -rn "while True" backend/packages/harness/deerflow`），对比结果，体会 DeerFlow 与「自己撸 loop」的 Agent 的根本差异。
4. **实验四：建立你自己的阅读地图** — 对照本章「怎么读这本书」给出的取舍路径，结合你当前最关心的问题（安全？成本？扩展？），在纸上画出你打算先读哪几章、为什么。读完全书后再回来看这张图，校准你的判断。

> **下一章预告**：知道了 DeerFlow「借了 LangGraph 的壳」，下一章我们拉远镜头，俯瞰整个 2.0 仓库——backend 的 monorepo 结构、`deerflow` harness 包的内部分区、以及一次请求从 HTTP 网关到 `lead_agent` 图的完整路径，建立起后续逐层深入所需的全景坐标系。
