# 解构 Agent：OpenClaw

> 一本面向 Agent 开发者的开源技术书籍 —— 通过逐层拆解 [OpenClaw](https://github.com/openclaw/openclaw) 的真实代码，理解一个生产级 Agent 系统是如何被工程化的。

## 关于本书

市面上关于 Agent 的内容大多停留在「Prompt 技巧」和「Demo 拼装」层面。但当你真正要把一个 Agent 做成**可约束、可观测、可复现、可迭代**的生产系统时，会发现真正的难点在于模型之外的那一整套工程系统——也就是 **Harness（驾驭层）**。

本书以 OpenClaw 这个成熟的开源 Agent CLI（TypeScript pnpm monorepo，138 个扩展、约 60 个 Skill）为标本，**自底向上、顺着一次真实请求的执行链路**，把 Agent 的每一个子系统逐一拆开：从进程入口、Agent 主循环、模型流式抽象、统一工具计划，到权限审批、上下文压缩、SQLite 状态持久化、插件/Skill/MCP 扩展体系，最后到 TUI/Web/Gateway 多端接入。

**读完本书你将掌握**：一个生产级 Agent 的完整骨架，以及每一个工程决策背后的「为什么」。

## 读者对象

- 正在或准备开发 Agent / Agent 框架的工程师
- 想理解 Claude Code / Codex 这类 Agent CLI 内部原理的开发者
- 需要为团队搭建可落地 Harness 的技术负责人

**前置知识**：熟悉 TypeScript / Node.js，了解 LLM API 基本概念（消息、工具调用、流式输出）。

## 阅读主线

本书遵循一条「跟随一次请求穿过整个系统」的主线：

```
进程入口 (openclaw.mjs → entry.ts)
  → 运行时选择 (src/agents/runtime)
    → Agent 主循环 (packages/agent-core/agent-loop.ts)
      → 模型流式抽象 (llm-runtime / src/llm/providers)
        → 统一工具计划 (src/tools + agent-tools)
          → 权限审批与安全 (plugin-sdk/approval-*, src/security)
            → 上下文引擎与压缩 (context-engine, harness/compaction)
              → 状态持久化 (SQLite, src/state)
                → 扩展体系 (plugin-sdk / skills / MCP)
                  → 多端接入 (TUI / Web / Gateway / Channels)
```

---

## 目录

### 第一部分　导论：为什么要解构 Agent

- **第 1 章　从 Prompt 到 Harness：Agent 工程范式的演进**
  - 1.1 `Agent = Model + Harness`：能力边界由谁决定
  - 1.2 为什么同一个模型在不同产品里表现天差地别
  - 1.3 Harness 的六个核心职责：上下文、工具、编排、状态、观测、约束
  - 1.4 本书方法论：以真实代码为唯一真理来源

- **第 2 章　OpenClaw 全景：一个生产级 Agent 长什么样**
  - 2.1 项目定位与设计哲学（`VISION.md`：core stays lean / capability via plugins）
  - 2.2 Monorepo 全景：`src/` `packages/` `apps/` `extensions/` `skills/` `ui/`
  - 2.3 四条工程红线：核心精简、能力插件化、配置唯一、状态只用 SQLite
  - 2.4 如何高效阅读这套代码（`AGENTS.md` 与 `scripts/check-*` 守卫）

### 第二部分　骨架：Agent 的执行内核

- **第 3 章　从进程到运行时：一次请求的起点**
  - 3.1 启动器 `openclaw.mjs`：Node 版本守卫、源码/产物分流、编译缓存
  - 3.2 入口 `src/entry.ts` 与运行时门面 `src/agents/runtime`
  - 3.3 嵌入式 Agent Runner：`src/agents/embedded-agent-runner` 的职责
  - 3.4 消息驱动路径：`src/auto-reply/reply/agent-runner.ts`

- **第 4 章　Agent 主循环：心脏是如何跳动的**
  - 4.1 `packages/agent-core` 为什么要独立成可复用包
  - 4.2 拆解 `agent-loop.ts`：`while(true)` 的一轮 turn 究竟做了什么
  - 4.3 `streamAssistantResponse → executeToolCalls → ToolResultMessage` 回灌
  - 4.4 Steering 消息、Abort 信号与用量（Usage）记账
  - 4.5 `Agent` 类与 `agent.ts`：队列模式与前后置 Hook
  - 4.6 依赖注入：`runtime-deps.ts` 如何解耦 stream/complete

- **第 5 章　Harness 编排层：把单轮循环串成完整任务**
  - 5.1 `src/harness/agent-harness.ts` 的全局视角
  - 5.2 Session、阶段（phases）与 Skill 调用编排
  - 5.3 何时继续、何时停止、何时压缩

### 第三部分　与模型对话：LLM 抽象层

- **第 6 章　多 Provider 抽象：让模型可插拔**
  - 6.1 类型契约 `packages/llm-core`：`Model` / `Context` / `Message` / `Usage`
  - 6.2 运行时分发 `packages/llm-runtime`：`api-registry` 与 `stream()`
  - 6.3 具体实现 `src/llm/providers`：Anthropic / OpenAI / Google / Mistral 的差异
  - 6.4 OpenAI Completions vs Responses vs ChatGPT-Responses 三套形态

- **第 7 章　流式、重试与成本：与模型对话的工程细节**
  - 7.1 流式包装 `stream-wrappers/` 与事件流 `EventStream`
  - 7.2 重试与退避：`provider-runtime/operation-retry.ts`
  - 7.3 Token 与成本核算如何穿过整个循环
  - 7.4 工具调用修复：`packages/tool-call-repair` 处理模型坏输出

### 第四部分　让 Agent 动手：工具系统

- **第 8 章　统一工具计划：core / plugin / channel / MCP 一视同仁**
  - 8.1 工具描述符契约 `src/tools/types.ts`：`ToolDescriptor` 与 `ToolOwnerRef`
  - 8.2 计划构建 `planner.ts` 与可用性表达式 `availability.ts`
  - 8.3 描述符 → 线上 Schema：`protocol.ts` 与 `execution.ts`

- **第 9 章　内置编码工具：read / edit / bash 的实现**
  - 9.1 `create-openclaw-coding-tools` 与 Claude 风格别名
  - 9.2 文件读写工具的边界硬化（Tool Filesystem Hardening）
  - 9.3 Bash 执行：`src/process/exec.ts` 与 shell 沙箱
  - 9.4 工具注册与缓存：`src/plugins/tools.ts`、`tool-descriptor-cache.ts`

### 第五部分　安全与可控：Agent 的护栏

- **第 10 章　权限与审批流：在动手前踩刹车**
  - 10.1 审批运行时 `plugin-sdk/approval-*`：handler / gateway / delivery / native
  - 10.2 执行审批 `exec-approvals-runtime.ts` 与系统命令 allowlist
  - 10.3 危险工具识别 `src/security/dangerous-tools.ts`

- **第 11 章　信任模型与安全审计**
  - 11.1 `SECURITY.md` 的信任边界：One-User / Operator / Trusted Plugins
  - 11.2 Workspace Memory 信任边界与 Sub-Agent 委派硬化
  - 11.3 静态分析：`security/opengrep` 规则包与 CI 中的 secret 扫描
  - 11.4 深度代码安全审计 `audit-deep-code-safety.ts`

### 第六部分　记住一切：上下文与状态

- **第 12 章　上下文引擎：决定模型看到什么**
  - 12.1 可插拔的 `src/context-engine`：`registry` / `delegate` / `AssembleResult`
  - 12.2 System Prompt 构建与 Skill 广告：`harness/system-prompt.ts`
  - 12.3 `AGENTS.md` 风格文件加载与 frontmatter 解析
  - 12.4 per-turn 与 thread-bootstrap 的上下文投影

- **第 13 章　压缩与剪枝：在有限窗口里走得更远**
  - 13.1 压缩流程 `harness/compaction`：分支摘要 branch-summarization
  - 13.2 运行时侧的压缩安全网 `agent-hooks/compaction-safeguard`
  - 13.3 上下文剪枝 `context-pruning/` 策略

- **第 14 章　状态持久化：为什么只用 SQLite**
  - 14.1 「SQLite-only」工程决策的代价与收益
  - 14.2 全局库与 per-agent 库：`src/state/openclaw-state-db` 与 `openclaw-agent-db`
  - 14.3 Schema、Kysely 类型与「`doctor --fix` 迁移、运行时不兜底」原则
  - 14.4 Session 生命周期、恢复与轨迹导出 `src/trajectory`

### 第七部分　可扩展性：让 Agent 长出能力

- **第 15 章　插件 SDK：核心精简、能力外挂**
  - 15.1 两种插件形态：代码插件 vs Bundle 插件
  - 15.2 `src/plugin-sdk` 与 `packages/plugin-sdk` 的公共契约
  - 15.3 激活与注册：`activation-planner`、`active-runtime-registry`
  - 15.4 在 `package.json` 中声明 `openclaw.{extensions,skills,prompts,themes}`

- **第 16 章　Skill 系统：用 Markdown 教会 Agent 做事**
  - 16.1 SKILL.md 单元与 frontmatter 契约
  - 16.2 加载与按需读取：`src/skills/loading` 与 `runtime/tool-dispatch`
  - 16.3 Skill 安全扫描 `skills/security/scanner.ts`

- **第 17 章　MCP 集成：把外部世界接进工具计划**
  - 17.1 MCP Server 侧：`tools-stdio-server` / `channel-server`
  - 17.2 MCP Client/Bridge 与 `kind:"mcp"` 的统一接入
  - 17.3 MCP 工具如何被同样的审批与安全约束覆盖

- **第 18 章　子 Agent 与多 Agent 委派**
  - 18.1 ACP（Agent Client Protocol）：`acp-spawn` 与 `packages/acp-core`
  - 18.2 子 Agent 咨询工具 `talk/agent-consult-tool.ts`
  - 18.3 为什么 OpenClaw 刻意不做「manager-of-managers」默认层级

### 第八部分　对外的脸：多端接入

- **第 19 章　终端 TUI：Agent 的主战场**
  - 19.1 `src/tui` 架构：launch / backend / event-handlers / formatters
  - 19.2 工具执行的可视化 `components/tool-execution`
  - 19.3 会话恢复 `tui-last-session`

- **第 20 章　Gateway 与 Web/Channel 接入**
  - 20.1 `packages/gateway-protocol` 与 `gateway-client`
  - 20.2 Web UI（`ui/`）如何通过 Gateway 连到运行时
  - 20.3 Channels（Discord / Telegram 等）作为工具 Owner 的统一抽象

### 第九部分　工程化收尾：测试、评估与发布

- **第 21 章　测试与评估：怎么知道改好了**
  - 21.1 分层测试：vitest 单测、e2e、docker、live
  - 21.2 架构不变量守卫：`test/architecture-smells.test.ts` 与 `scripts/check-*`
  - 21.3 评估场景 `qa/scenarios` 与 frontier-harness 计划

- **第 22 章　构建与发布：从源码到可执行**
  - 22.1 打包器 tsdown：`tsdown.config.ts` 的多入口设计
  - 22.2 `scripts/build-all.mjs` 编排与 plugin-sdk 类型导出
  - 22.3 多渠道发布：Docker、Fly、Render、macOS appcast 自动更新

### 附录

- **附录 A　OpenClaw 目录速查表**（顶层 13 个区域逐一索引）
- **附录 B　一次请求的完整时序图**（从 TUI 输入到工具结果回灌）
- **附录 C　术语表**（Harness / Tool Descriptor / Context Engine / ACP / MCP …）
- **附录 D　从 OpenClaw 借鉴：搭建你自己 Agent 的最小 Harness 清单**

---

## 配套说明

- 本书引用的代码路径均来自 OpenClaw 公开仓库，建议边读边对照源码。
- 每章末尾计划提供「关键文件清单」与「动手实验」。
- 欢迎通过 Issue / PR 参与勘误与共建。

## License

本书内容采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可；涉及的 OpenClaw 代码版权归其原作者所有。
