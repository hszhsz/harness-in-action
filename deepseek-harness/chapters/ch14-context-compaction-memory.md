# 第 14 章　上下文、压缩与记忆

> 模型窗口有限，Agent 的上下文必须被精心组装、压缩和遗忘。这一章拆开 dsh 的上下文体系：system-prompt assembly、`agent.inject()`、compaction seam、skills、goals/schedule/todo。

## 14.1　System-prompt assembly：prompt sections 的按序拼接

`ctx.systemPrompt` 拥有 prompt-section 与 tool-schema 组装。[[docs/subsystems/system-prompt.md]](https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/subsystems/system-prompt.md)

```ts
interface PromptSection {
  name: string          // 唯一名，重复注册抛错
  // sections 按升序拼接；约定：-100 是…
}
```

**Sections 按升序拼接**（"Sections are concatenated in ascending order"）。约定：`-100` 是 harness identity opener，`0` 是 deployment persona，`50` 是 plan policy，`-99` 是 harness:source（`addHarnessSourceSection` 注册，告诉 agent dsh 实现 checkout 在哪、警告不要从路径推断 cwd）。

`AssembleContext` 是 merge-extensible：`dsh-agent` 加可选 live `agent` 字段，`assembleContextFor(agent, signal)` 一起设置显式字段。`ToolProviderResult` 提供模型可见 schema 集 + 预限制名字宇宙（`knownNames`——区分"配置名字打错了"和"故意隐藏的已知工具"）。

**协作组装**（cooperative assembly）：sections 的贡献者通过 waterfall 协作，一个有效的 `complete` section 成为唯一 prompt section。每个 section 的文本可以静态，也可以从当前 assembly 上下文解析。

## 14.2　上下文插件：`agent.inject()` 与 per-request context

第 6.1 节讲过 `agent.inject()`：排队模型可见上下文给下一个 pre-step，**不唤醒**。`docs/architecture.md` 的扩展点地图：

> Add model-facing context → call `agent.inject()`; it lands in the next admitted request.

注入的上下文会成为 `user/message` 事件（`source` 区分：file-change notices、subdir AGENTS.md、skill content、cron notifications、goal continuation round——"All three project their `content` verbatim; `source` tells them apart"）。还有 `packages/context` 组的 request-context 插件。

## 14.3　Compaction seam：`compaction/*` 事件与 tool-result pruner

`ctx.compaction`（严格说 compaction 是 seam：`compaction/compaction` Definition + `compaction-basic` / `compaction-tool-result-pruner` Provider + 主循环 Consumer）。[[docs/subsystems/compaction.md]](https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/subsystems/compaction.md)

compaction 通过声明合并给 `SessionEventMap` 加 `compaction/start` / `compaction/summary` / `compaction/end`——**不是 `SurfaceEventType`**（无 `surfaceOp`）。`CompactionEngine` 接口 + `CompactionResult`。`command-compact` 是 `/compact` 命令消费者。

`tool-result-pruner` 剪枝工具结果（spill 化的大输出），`compaction-basic` 做基本摘要压缩。

## 14.4　Skills：注册表、目录注入与 `skill` 工具

skill 能力族（`packages/skill`）：Definition（`dsh-skill`，`ctx.skills`）+ Provider（`skill-filesystem` 本地、`skill-badge` 打包徽章）+ Consumer（`tool-skill`）。[[docs/subsystems/skills.md]](https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/subsystems/skills.md)

- **注册表是 host+per-scope 分层的**（沿用 tools registry 在 `dsh-scope` 上建立的形状）：host 行和 repo 插件进全局层，agent preset 挂载的插件进该 preset 的层——provider 名**每层唯一**而不是进程级唯一。读合并全局层 + 查看 scope 的链，最近层的条目在重名时直接胜出。
- **发现缓存按 scope 链 key**：re-parenting 一个 scope（blank-session recompose）对下次读可见。
- **Consumer 拥有初始与替换目录 + 模型面对的 `skill` 工具**；`user/message` replacement catalogs via `agent.inject()`。
- **Skills 是可选指令，不是 session 事件**——词汇在 skill 页而不是 core.md。

## 14.5　Goals / Schedule / Todo：同会话目标与定时记忆

- **Goals**（`goal/`）：持久化的同会话目标域。`GoalId` branded，`GoalRef`（id + revision）CAS 变更；`GoalPhase` = active / paused / blocked / complete。blocked 是唯一的"被问题卡住"持久状态，带稳定 lower-kebab-case `code`（机器路由）+ 人类可读 `message`。工具 `create_goal`/`get_goal`/`update_goal`：create/edit/pause/resume 要求直接人类 root authority；complete/blocked 也接受当前目标轮次；默认 blocked 下界是 3 个已承认轮次（对照我们自己的 goal 工具——dsh 把同款设计开源了）。
- **Schedule**（`schedule/`）：session-local 提醒。`schedule_create`/`schedule_delete`/`schedule_list`，v1 接受 after_seconds、显式绝对 at、有界固定速率 every_seconds，披露 session-local delivery；管理读与变更需要共享 Session persistence barrier。只在 opt-in Schedule 插件加载后创建的 live root Agent scopes 内注册。
- **Todo**（`todo/`）：`todo_write` 是 session-owned 状态；UI 渲染最新的 `todo/write` 事件为 checklist。`allowParallelInProgress` 无默认必须显式声明——"so the catalog states its choice: true, whose description invites several in_progress items"。

## 本章小结

dsh 的上下文与记忆体系：

1. **组装**：prompt sections 升序拼接，`-100` identity → `0` persona → `50` plan policy；
2. **注入**：`agent.inject()` 不唤醒地排队模型可见上下文，落进下一个被承认的请求；
3. **压缩**：compaction seam 加 log-only 事件，basic 摘要 + tool-result pruner 剪枝；
4. **Skills**：host+per-scope 分层注册表，目录可注入，可选指令不是事件；
5. **目标/定时/todo**：Goals 持久化 CAS 状态机、Schedule session-local 提醒、Todo log-only 状态。

## 动手实验

1. 读 `docs/subsystems/system-prompt.md` 的 section order 约定，在源码里找出所有注册的 section 及其 order。
2. 读 `packages/compaction/compaction-basic`，看基本压缩如何生成 `compaction/summary`。
3. 读 `packages/goal/goal/src/types.ts` 的 `GoalPhase` 与 `GoalBlockReason`，对照我们的 create_goal/update_goal 工具设计。
4. 读 `packages/schedule/schedule`，确认 v1 支持的时间形式与 delivery 语义。
