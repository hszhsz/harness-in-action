# 解构 Agent：pi

> 一本面向 Agent 开发者的开源技术书籍 —— 通过逐层拆解 [pi](https://github.com/earendil-works/pi) 的真实代码，理解一个「把约束外包出去、核心只留精简骨架」的 Agent 系统是如何被工程化的。

## 关于本书

市面上关于 Agent 的内容大多停留在「Prompt 技巧」和「Demo 拼装」层面。但当你真正要把一个 Agent 做成**可约束、可观测、可复现、可扩展**的工程系统时，会发现真正的难点都在模型之外——也就是 **Harness（执行骨架层）**：工具怎么挂载、上下文如何组装与压缩、循环在哪里被打断、出错后怎么恢复、副作用能不能落地、能力如何被第三方安全地扩展。

本书以 **pi** 这个由 [earendil-works](https://github.com/earendil-works/pi) 维护的 TypeScript Agent 为标本。pi 是一个 pnpm monorepo，由四个职责清晰的包组成：`@earendil-works/pi-ai`（多 Provider LLM 抽象）、`pi-agent-core`（Agent 运行时与 Harness 子系统）、`pi-coding-agent`（编码 Agent CLI 与内置工具）、`pi-tui`（自研终端 UI）。

pi 最鲜明的工程性格是 **「把约束外包」**：它的核心**刻意不内置权限系统**，而是把「该不该执行这条命令」这类策略决策下放给扩展与容器（README 里反复强调 OpenShell / Docker / 沙箱方案）；核心循环只负责一件事——**忠实地把「生成 → 调工具 → 把结果喂回去」这个机制跑稳**。这是一种「机制与策略彻底分离」的设计哲学，和那些 fail-closed 偏执型 Agent 形成鲜明对照。

本书**自底向上、顺着一次真实请求的执行链路**，把 pi 的每一个子系统逐一拆开：从 `streamSimple` 这条最底层的模型调用契约，到 `runLoop` 的双层循环，到 `AgentHarness` 的阶段编排，再到 JSONL 会话树、压缩与分支摘要、可插拔 Operations 的工具系统、自扩展的 Extension/Skill 体系，最后到差分渲染的 TUI 与多端模式。

**读完本书你将掌握**：一个生产级 Agent 的完整骨架，以及 pi 在每一个工程决策背后的「为什么」——尤其是「不做什么」背后的取舍。

## 读者对象

- 正在或准备开发 Agent / Agent 框架的工程师
- 想理解 Claude Code / Codex 这类 Agent CLI 内部原理的开发者
- 需要为团队搭建可落地 Harness、并思考「权限到底该放在哪一层」的技术负责人

**前置知识**：熟悉 TypeScript / Node.js，了解 LLM API 基本概念（消息、工具调用、流式输出）。

## 阅读主线

本书遵循一条「跟随一次请求穿过整个系统」的主线：

```
进程入口与运行时 (coding-agent: main.ts / modes)
  → Harness 编排 (packages/agent/src/harness/agent-harness.ts)
    → Agent 主循环 (packages/agent/src/agent-loop.ts → runLoop)
      → 模型流式抽象 (packages/ai: stream.ts / api-registry.ts)
        → Provider 实现 (packages/ai/src/providers + transform-messages)
          → 统一工具系统 (coding-agent: tools + ToolDefinition 包装)
            → 准入与执行分离 (beforeToolCall 钩子 / 无内置权限)
              → 上下文与会话树 (harness/session: JSONL 树 + leaf 导航)
                → 压缩与分支摘要 (harness/compaction)
                  → 扩展体系 (extensions / skills / slash commands)
                    → 多端接入 (TUI 差分渲染 / print / rpc)
```

---

## 目录

### 第一部分　导论：为什么要解构 Agent

- **第 1 章　从 Prompt 到 Harness：Agent 工程范式的演进**
  - 1.1 `Agent = Model + Harness`：能力边界由谁决定
  - 1.2 为什么同一个模型在不同产品里性格迥异
  - 1.3 Harness 的六个核心职责：上下文、工具、编排、状态、观测、约束
  - 1.4 pi 的独特立场：把「约束」外包给扩展与容器
  - 1.5 本书方法论：以真实代码为唯一真理来源

- **第 2 章　pi 全景：四个包如何各司其职**
  - 2.1 monorepo 结构：`packages/{ai,agent,coding-agent,tui}`
  - 2.2 一条最关键的接缝：`streamSimple` 注入与 provider 无关
  - 2.3 设计红线：核心精简、约束外包、状态用 JSONL 树
  - 2.4 如何高效阅读这套代码（`AGENTS.md` 与 pi 用自己开发自己）

### 第二部分　骨架：Agent 的执行内核

- **第 3 章　Agent 主循环：心脏是如何跳动的**
  - 3.1 `packages/agent` 为什么要独立成可复用包
  - 3.2 三层 API：`agentLoop` / `runAgentLoop` / 私有 `runLoop`
  - 3.3 双层循环：工具驱动的内层 vs 消息驱动的外层
  - 3.4 一轮 turn 究竟做了什么：流式生成 → 抽取工具 → 执行 → 回灌
  - 3.5 串行还是并行：由工具自己声明 `executionMode`
  - 3.6 `terminate` 与 `stopReason`：循环如何停下来

- **第 4 章　可编程的循环：钩子、队列与三种消息**
  - 4.1 `AgentLoopConfig` 的九个钩子：机制与策略的分界线
  - 4.2 三种消息队列：steering / follow-up / next-turn 的语义差异
  - 4.3 `beforeToolCall` / `afterToolCall`：在动手前后插一脚
  - 4.4 `Agent` 类与 `prompt` 队列模式：把循环包成对象
  - 4.5 错误即数据：失败为什么不抛异常而编码进流

- **第 5 章　Harness 编排层：把单轮循环串成完整任务**
  - 5.1 `AgentHarness` 的全局视角：1064 行里有什么
  - 5.2 阶段机（phases）：idle / turn / compaction / branch_summary / retry
  - 5.3 中间件式事件钩子与 `pendingSessionWrites`
  - 5.4 `prompt` / `skill` / `steer` / `followUp` / `nextTurn` / `compact` / `navigateTree` / `abort`
  - 5.5 何时继续、何时压缩、何时停止

### 第三部分　与模型对话：LLM 抽象层

- **第 6 章　多 Provider 抽象：让模型可插拔**
  - 6.1 类型契约 `packages/ai/src/types.ts`：`Model` / `Context` / `Api`
  - 6.2 注册表 `api-registry.ts`：`registerApiProvider` 与 `sourceId` 卸载
  - 6.3 门面 `stream.ts`：`stream` / `complete` / `streamSimple` / `completeSimple`
  - 6.4 环境变量 API Key 的隐式注入（`withEnvApiKey`）

- **第 7 章　流式、转换与认证：与模型对话的工程细节**
  - 7.1 `AssistantMessageEventStream` 与「错误编码进流」契约
  - 7.2 `transform-messages.ts`：跨 Provider 的消息归一化
  - 7.3 Anthropic / OpenAI-completions / OpenAI-responses / Codex-responses 的形态差异
  - 7.4 `faux` provider：不联网、不烧 token 地测整个循环
  - 7.5 OAuth 与会过期的短期 token：`getApiKey` 每轮重解析

### 第四部分　让 Agent 动手：工具系统

- **第 8 章　统一工具模型：ToolDefinition 与 AgentTool**
  - 8.1 七个内置工具：read / bash / edit / write / grep / find / ls
  - 8.2 `ToolDefinition` → `AgentTool` 的包装层 `wrapToolDefinition`
  - 8.3 工具集合工厂：coding / read-only / all 三套预设
  - 8.4 `prepareArguments`：在 schema 校验前修复模型坏输出

- **第 9 章　可插拔 Operations：把执行后端抽象出去**
  - 9.1 `BashOperations` 契约：为什么把 `exec` 单独抽成接口
  - 9.2 `createLocalBashOperations`：本地 shell 的默认实现
  - 9.3 文件变更队列 `file-mutation-queue.ts`：串行化写操作
  - 9.4 输出硬化：截断、字节上限与 `OutputAccumulator`

### 第五部分　安全与可控：约束放在哪一层

- **第 10 章　pi 的「无权限」哲学：刻意不做什么**
  - 10.1 为什么核心**不内置**权限系统
  - 10.2 准入与执行分离：`beforeToolCall` 作为唯一闸门
  - 10.3 把约束外包给容器：OpenShell / Docker / 沙箱模式
  - 10.4 这种设计的代价与收益：灵活性 vs 默认安全

### 第六部分　记住一切：上下文与状态

- **第 11 章　会话即一棵树：JSONL 持久化与 leaf 导航**
  - 11.1 append-only JSONL：`jsonl-repo.ts` 与 `jsonl-storage.ts`
  - 11.2 `SessionTreeEntry` 联合类型：message / compaction / branch_summary / leaf
  - 11.3 基于 leaf 的导航：`navigateTree` 与「当前路径」重建
  - 11.4 `memory-repo` 与可测试性：把存储后端抽象掉

- **第 12 章　压缩与分支摘要：在有限窗口里走得更远**
  - 12.1 何时压缩：`shouldCompact` 与 token 估算
  - 12.2 压缩流程 `compaction.ts`：`findCutPoint` / `keepRecentTokens`
  - 12.3 分支摘要 `branch-summarization.ts`：把旧分支折叠成摘要节点
  - 12.4 压缩感知的上下文重建：摘要节点如何参与 `convertToLlm`

### 第七部分　可扩展性：让 Agent 长出能力

- **第 13 章　自扩展的 Extension 系统**
  - 13.1 `jiti` 加载 TypeScript 扩展：`loader.ts` 与运行时桩
  - 13.2 `ExtensionContext` / `ToolDefinition`：扩展能注册什么
  - 13.3 阻塞式 `tool_call` 钩子：扩展如何拦截与改写工具调用
  - 13.4 真实样例：`tps` 扩展用 `agent_start` / `agent_end` 算 token/s

- **第 14 章　Skill 与 Slash Command：用 Markdown 教会 Agent**
  - 14.1 `SKILL.md` + frontmatter 契约与渐进式披露
  - 14.2 加载与按需读取：`core/skills.ts` 与系统提示词广告位
  - 14.3 `system-prompt.ts`：`<available_skills>` 是怎么拼出来的
  - 14.4 Slash Command：把可复用动作挂到命令面板

### 第八部分　对外的脸：多端接入

- **第 15 章　自研 TUI：不用 React 也能做差分渲染**
  - 15.1 `pi-tui` 为什么放弃 ink/React，用 `string[]` 自己渲染
  - 15.2 差分渲染器与 Kitty 键盘协议
  - 15.3 工具执行的可视化与会话恢复
  - 15.4 print 模式与 rpc 模式：把同一个内核接到不同入口

### 第九部分　工程化收尾：测试、供应链与发布

- **第 16 章　可信度工程：测试、faux provider 与供应链**
  - 16.1 用 `faux` provider 驱动确定性测试
  - 16.2 TypeBox schema 与 Node strip-only 模式：少一层构建
  - 16.3 pi 用自己开发自己（dogfooding）作为最强回归测试
  - 16.4 从 pi 借鉴：搭建你自己 Agent 的最小 Harness 清单

### 附录

- **附录 A　pi 目录速查表**（四个包逐一索引）
- **附录 B　一次请求的完整时序图**（从 TUI 输入到工具结果回灌）
- **附录 C　术语表**（Harness / EventStream / SessionTree / Operations / Extension …）
- **附录 D　pi vs 偏执型 Agent：约束放置位置的工程对照**

---

## 配套说明

- 本书引用的代码路径均来自 pi 公开仓库（解读时 HEAD 为 `1da90398`），建议边读边对照源码。
- 每章末尾提供「关键文件清单」与「动手实验」。
- 欢迎通过 Issue / PR 参与勘误与共建。

## License

本书内容采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可；涉及的 pi 代码版权归其原作者所有。
