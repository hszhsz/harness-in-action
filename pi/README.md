# 解构 Agent：pi

> 一本面向 Agent 开发者的开源技术书 —— 通过拆解 [pi](https://github.com/earendil-works/pi) 的真实代码，理解一个「核心极简、约束外包、一切皆可替换」的 Agent 系统是如何被工程化的。

## 版本基线

本书解读的是 **pi `v1.3.0-46-g1d01b3b1f`**：

| 项目 | 值 |
|---|---|
| Git commit | `1d01b3b1f8de2e67a3a1c1694b14c984e4edd156` |
| `git describe` | `v1.3.0-46-g1d01b3b1f` |
| 各包版本 | 统一 `1.3.0`（`pi-ai`、`pi-agent-core`、`pi-coding-agent`、`pi-tui`、`pi-codemode`、`pi-mcp`、`pi-durable`、`pi-env`、`chord`、`pi-telemetry`、`pi-evals`，其中 `pi-evals` 为私有包） |
| 解读日期 | 2026-10-20 |

阅读时请先在自己的 pi 检出里执行 `git describe --tags --always` 与 `git log -1 --format=%H` 核对。若你读到的版本与上表不同，请以源码为准；本书描述的是上述 commit 的行为。

> 上一版电子书基于 `v0.79.1-15-g1da903983`（2026-06-11），与本版之间相隔约 2500 个提交。架构层面的主要变化已在正文中标注为「1.3 的变化」，便于老读者快速定位。

## 关于本书

市面上关于 Agent 的内容大多停留在 Prompt 技巧与 Demo 拼装。但要把 Agent 做成**可约束、可观测、可恢复、可扩展**的工程系统，真正的难点都在模型之外，也就是 **Harness（执行骨架层）**：上下文如何组装、工具如何挂载、循环在哪里停车、失败如何变成数据、状态如何持久化、第三方能力如何安全接入。

pi 的工程性格非常鲜明：

1. **内核只做机制，不做策略。** `pi-agent-core` 不知道什么是危险命令，不内置任何工具，甚至不知道会话存在哪里。
2. **约束外包。** 核心刻意不内置权限系统；准入通过 `beforeToolCall` / 扩展的 `tool_call` 事件完成，隔离交给容器与沙箱。
3. **转录即状态。** 从 1.3 起，系统提示词与工具声明不再旁路传递，而是作为 `SystemMessage` 写入对话转录；每一轮请求都由转录自身的状态推导出来。
4. **一切可替换。** 模型层可换 provider、换路由；工具层可换执行后端；编排层可换存储；同一个内核既能开 TUI，也能退化成 print / JSON / RPC，或被 SDK 嵌进别的程序。

本书跟随一次真实请求穿过整个系统，既讲「pi 怎么做」，也讲「为什么这样做，以及代价是什么」。

## 读者对象

- 正在开发 Agent / Agent 框架的工程师
- 想理解 Claude Code、Codex 这类 Agent CLI 内部原理的开发者
- 需要为团队搭建可落地 Harness，并思考「权限到底该放在哪一层」的技术负责人

**前置知识**：熟悉 TypeScript / Node.js，了解 LLM API 的基本概念（消息、工具调用、流式输出）。

## 精华速览（时间有限的读法）

只想用两小时抓住 pi 的精华，按这个顺序读：

1. **第 3 章**：看 `runLoop` 如何用「内层工具循环 + 外层 follow-up 循环」跑完一个任务。
2. **第 4 章**：看十个钩子如何把策略从机制里剥出去。
3. **第 5 章**：看 `AgentSession` 如何把单轮循环升级为可压缩、可重试、可路由的会话。
4. **第 11 章**：看「追加式 JSONL 树」如何同时解决持久化、分支与回溯。
5. **第 10 章**：看 pi 为什么不内置权限，以及这个选择的真实代价。
6. **第 13、14、15 章**：看扩展、Skill、MCP 与 codemode 如何让核心保持小而能力持续长大。

## 阅读主线

```
进程入口 (coding-agent: main.ts → modes)
  → 会话编排 (core/agent-session.ts → AgentSession)
    → 请求投影 (SessionManager.buildSessionProjection)
      → Agent 主循环 (agent/agent-loop.ts → runLoop)
        → 转录声明 (SystemMessage: prompt + toolsAdded/toolsRemoved)
          → 模型流 (ai/models.ts → Provider → stream)
            → 工具流水线 (prepare → validate → before → execute → after)
              → 结果回灌 (ToolResultMessage → 下一轮)
                → 持久化 (JSONL 树 + compaction + branch_summary)
                  → 扩展面 (extensions / skills / MCP / codemode / packages)
                    → 多端 (TUI / print / JSON / RPC / SDK)
```

---

## 目录

### 第一部分　导论

- **第 1 章　从 Prompt 到 Harness：Agent 工程范式的演进**
  - Agent = Model + Harness
  - 同一个模型为什么在不同产品里性格迥异
  - Harness 的六个核心职责
  - pi 的立场：把约束外包

- **第 2 章　pi 全景：十一个 workspace，三层接缝**
  - `packages/` 的真实构成（含 `chord`、`durable`、`env`、`mcp`、`codemode`、`telemetry`、`evals`）
  - 三条接缝：`StreamFn`、`AgentTool.execute`、`SessionManager`
  - 设计红线与阅读路线

### 第二部分　执行内核

- **第 3 章　Agent 主循环：心脏如何跳动**
  - 三层 API：`agentLoop` / `runAgentLoop` / `runLoop`
  - 转录原生：系统提示词和工具声明为什么进入 `SystemMessage`
  - 双层循环与一次 turn 的完整步骤
  - 并行/串行工具执行、截断响应、`terminate` 与 `stopReason`

- **第 4 章　可编程的循环：钩子、队列与错误即数据**
  - `AgentLoopConfig` 的十个钩子
  - `finishTurn` / `prepareRequest` / `prepareNextTurn` 的分工
  - steering / follow-up 队列与消费时机
  - `Agent` 类、生命周期与错误编码进流

- **第 5 章　AgentSession：把循环变成产品级会话**
  - 从 `AgentHarness` 到 `AgentSession` 的架构迁移
  - 请求投影、虚拟模型路由、自动压缩
  - 边界事件、工具钩子、系统提示词增量刷新
  - `AgentSessionRuntime`：换会话、派生、导入与重载

### 第三部分　模型层

- **第 6 章　Models 运行时：Provider 不再是一张全局表**
  - `createModels()` / `Models` / `Provider` 三件套
  - 模型目录、动态刷新、认证归属
  - 请求变换、重试、成本计算

- **第 7 章　流式协议、消息转换与认证**
  - `AssistantMessageEvent` 的完整事件序列
  - `transformMessages`：跨 Provider 归一化
  - faux provider 与确定性测试
  - OAuth、`getApiKey` 与短期 token

### 第四部分　工具系统

- **第 8 章　统一工具模型：ToolDefinition、负载与暴露**
  - 八个内置工具与三套预设
  - `ToolDefinition` → `AgentTool` 包装
  - `exposure` / `namespace` / `defaultActive` / `prepareLoadout`
  - `prepareArguments` 与参数校验

- **第 9 章　可插拔执行后端：BashOperations 与文件变更队列**
  - 一个 `exec` 撑起的后端抽象
  - 本地 shell、超时、进程组与输出累积
  - 文件写入串行化、截断与结构化结果
  - 容器 / SSH / Gondolin 的接入点

### 第五部分　安全哲学

- **第 10 章　pi 的「无权限」哲学**
  - 为什么核心不内置权限系统
  - 准入与执行分离：唯一闸门 `beforeToolCall`
  - 把约束外包给容器、沙箱与项目信任
  - 这个选择的收益与代价

### 第六部分　状态与上下文

- **第 11 章　会话即一棵树：JSONL v3 与投影**
  - 追加式 JSONL、11 种 entry、运行期 leaf 指针
  - `buildContextEntries` / `buildSessionProjection`
  - `context_edit`：不改历史也能改上下文
  - 分支、标签与 `createBranchedSession`

- **第 12 章　压缩、溢出恢复与分支摘要**
  - 阈值压缩与溢出压缩的三种路径
  - `findCutPoint`、split-turn 与结构化摘要
  - 压缩后的系统提示词快照
  - 分支摘要与「压缩感知」的上下文重建

### 第七部分　可扩展性

- **第 13 章　扩展系统与 Pi Packages**
  - jiti 加载、虚拟模块、注册 API 与动作 API
  - 事件墙：观察、改写、拦截、边界提交
  - 工具定义超集与自定义渲染
  - Pi Packages：用 npm / git / 本地目录分发能力

- **第 14 章　Skill、Slash Command 与提示词模板**
  - `SKILL.md` 契约与渐进式披露
  - 发现顺序、冲突规则与 `/skill:name`
  - 命令面板与 bash 风格参数模板
  - 系统提示词的分区与增量更新

- **第 15 章　MCP、Codemode 与工具搜索**
  - MCP 的四种暴露模式与 `mcp__server__tool` 命名
  - QuickJS WASM 沙箱中的 codemode
  - BM25 工具搜索与延迟加载
  - 嵌套工具调用：不进入模型上下文的工具链

### 第八部分　对外的脸

- **第 16 章　自研 TUI：两个渲染器与 `string[]` 差分**
  - 主屏渲染器与备用屏渲染器
  - `CURSOR_MARKER`、IME、Kitty 键盘协议
  - 布局、滚动、搜索与图片
  - interactive / print / JSON / RPC 四种模式与 SDK 入口

- **第 17 章　Durable、Chord、Env：另一条前沿**
  - 「先提交、再展示」的 durable harness
  - Chord：可热重载的对象与复制状态
  - Env：把工具执行搬到 SSH 远端
  - 为什么 coding-agent 不使用它们

### 第九部分　工程化收尾

- **第 18 章　可信度工程：测试、evals、供应链与决策清单**
  - faux provider、evals 与确定性回归
  - TypeBox + Node strip-only：少一层构建
  - 供应链与版本锁定
  - 搭建你自己的最小 Harness：一张决策清单

---

## 阅读约定

- 代码路径均相对于 pi 仓库根目录。示例代码为便于讲解做过删减，省略号不代表原文。
- 提到「1.3 的变化」时，指的是相对上一版电子书基线 `1da90398` 的差异。
- 每章末尾有「关键文件清单」和「动手实验」，实验都可以在本地 pi 检出中完成，不需要 API Key 的章节会特别说明。
- pi 的代码注释本身就是高质量设计文档，建议先读类型与注释，再读实现。

## License

本书内容采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可；涉及的 pi 代码版权归其原作者所有。
