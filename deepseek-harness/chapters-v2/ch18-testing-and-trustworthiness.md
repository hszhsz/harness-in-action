# 第 18 章　测试与可信度：怎么知道改好了

dsh 在"可信度"上的投入可能是所有已解读 Agent 里最重的：keyless snapshot 测试、运行时不变量注册表、每文件 100% 覆盖率门禁、防御模式文档、几十个 verify-* 门禁脚本。这一章讲它的工程可信度体系。读完后，你应该能理解为什么"行为变更必须带可重放证据"是 dsh 测试哲学的核心，以及从 dsh 可以借鉴的最小清单。

## 分层测试：vitest 单测、e2e、keyless snapshot

dsh 的测试体系（`package.json` scripts）：

- **`test`**：vitest 单测。
- **`test:coverage`**：CI 覆盖率门禁——**`packages/*/*/src` 每文件 100% 覆盖率**。这是极端严苛的门槛，迫使每个分支都被测试覆盖。
- **`test:e2e`**：真实 API 测试，无 `DEEPSEEK_API_KEY` 自动跳过。
- **`test:snapshot`**：**keyless** ACP/headless 重放对比期望输出——不需要 API key 的模型行为测试。`test:snapshot:record` 重新录制（需要 key），`test:snapshot:refresh` 刷新。
- **`test:web` / `test:web:perf` / `test:web:stress` / `test:gui`**：Web/性能/压力/GUI 分层。
- **`check:all` / `check:ci`**：`scripts/run-gates.ts` 的完整门禁编排。

**关键测试哲学**（`AGENTS.md`）：

> **Tests describe behavior, not correctness.** Change obsolete behavior with its tests; explain why in the PR.

> Every non-trivial model- or product-user-visible behavior change adds or updates a keyless snapshot through a real runnable example in the same PR; package tests, e2e-only assertions, and mock-only fixtures do not substitute for the assembled application transcript. Fixtures must replay on macOS/Linux; fix fixtures, not normalizers.

**keyless snapshot 是硬要求**：任何非平凡的模型/产品用户可见行为变更，必须带一个真实可运行的 example + keyless snapshot——mock-only fixtures 不算数。"Fix fixtures, not normalizers"——修 fixture，不修 normalizer（防止掩盖真实差异）。

## 运行时不变量注册表（invariants）

`docs/subsystems/invariants.md`：运行时 invariant 注册表——选择 `Config`、`InvariantInstaller`/`InvariantFailure`、empty-companion 契约。

`AGENTS.md` 的核心规则：

> **Runtime invariants assert owned relationships.** Check authoritative event streams or mutable data, not service or method presence, plugin metadata or effects, or fixed pure examples. Without a plausible relationship, an explained empty companion is correct.

**不变量断言"拥有的关系"**——检查权威事件流或可变数据，而不是检查"服务/方法存在"、"插件元数据/effect"或固定纯例子。没有合理关系时，解释过的空同伴是正确的。

最典型的运行时不变量就是 "Model-visible ⟺ logged"——任何到达模型请求的东西必须能从 session 日志重建。这类不变量被写进代码、在运行时断言，而不是只停留在文档里。

## 防御模式（defensive patterns）与 fail loud 哲学

`docs/defensive-patterns.md` 是生命周期、并发、subprocess、teardown 工作的必读。`AGENTS.md` 的防御规则精选：

- **Misconfiguration fails loud**：装载期自包含的错立刻报；否则在最早可解析点报。**绝不静默跳过缺失的引用**。
- **Opaque cross-boundary ids are branded**：跨边界的 id 用 `Branded<B>`，绝不用裸 `string`。`packages/core/session/src/types.ts` 里的 `SessionId`、`packages/subagent/subagent/src/types.ts` 里的 `SubagentRunId`、`packages/workflow/workflow/src/types.ts` 里的 `WorkflowRunId` 全部是 branded。
- **Trust TypeScript at typed same-process boundaries**：不在静态接口要求处加运行时校验/fallback/敌对输入测试；在 parser/config、queued、model/tool JSON、durable/file、worker、process、wire 边界校验。
- **An empty `catch` names what it swallows**：空 catch 必须说明吞掉了什么、为什么没有别的东西能到达它；`try` 只包一个语句。
- **Fail loud 的启动**：`installFailLoud`——启动失败 = 一行带标签 stderr + exit(1)，先恢复终端再退出。
- **Approval fail-closed**：应答者出任何问题都 `unavailable`，门是关的。
- **Sandbox 诚实报告**：`partial` 就是 `partial`，不假装更安全。

## 从 DeepSeek Harness 借鉴：插件化 Harness 的最小清单

把 dsh 的方法论压缩成一份可抄的清单：

1. **没有特权核心**：主循环、工具、日志、适配器全部是插件；扩展 = 在旁边挂插件，不是打补丁。
2. **注册即 effect**：每个贡献返回 disposer；卸载自动回卷——热更新与失败清理共享一套逻辑。
3. **Model-visible ⟺ logged**：模型看到的一切必须能从事件日志重建；新输入先加新事件。
4. **Capability Seam 三件套**：Definition / Provider / Consumer 一起设计，换 Provider 换产品。
5. **工具只返回规范值**：呈现由管线负责；policy/hooks/sandbox 通过事件瀑布插入，不改工具、不改循环。
6. **显式默认**：`resolve(request): Spec` 把默认值显式落定，部署差异 = 配置差异。
7. **fail loud + fail closed**：配置错大声报，审批/沙箱缺应答者时关门。
8. **诚实报告**：沙箱 enforcement `full`/`partial`、快照可重放、UI 呈现纯函数。
9. **keyless snapshot + 每文件 100% 覆盖**：行为变更必须带可重放证据。
10. **文档与源码同源**：生成的 catalog + type-equiv 漂移检查，文档永不过期。

## 架构速览

```
可信度三件套
┌─────────────────────────────────────────────┐
│  测试                                        │
│  vitest 单测 + 每文件 100% 覆盖率             │
│  keyless snapshot (真实 example 重放)         │
│  e2e (需要 API key) + web/perf/stress/gui    │
├─────────────────────────────────────────────┤
│  不变量                                      │
│  运行时注册表, 断言拥有的关系                   │
│  Model-visible ⟺ logged (核心)               │
│  检查权威事件流, 不检查存在性                   │
├─────────────────────────────────────────────┤
│  防御模式                                     │
│  fail loud (配置错大声报)                      │
│  fail closed (审批/沙箱缺应答者关门)            │
│  branded ids (跨边界 id 品牌化)                │
│  边界校验 (parser/config/wire/...)            │
│  诚实报告 (沙箱 enforcement)                   │
└─────────────────────────────────────────────┘
```

## 关键设计决策

1. **每文件 100% 覆盖率门禁**：这是极端严苛的门槛。它迫使每个分支都被测试覆盖，包括错误路径和边界条件。代价是开发速度变慢，收益是代码的可靠性极高。

2. **keyless snapshot 是硬要求**：mock-only fixtures 不算数。行为变更必须带真实可运行的 example + 重放对比。这让"改了什么"有可验证的证据，而不是"我相信它没坏"。

3. **不变量断言关系，不检查存在性**：检查"服务是否注册"不如检查"事件流是否一致"有意义。这让不变量能发现真正的问题（数据不一致），而不是误报（服务暂时不可用）。

4. **branded ids 是编译期安全**：`Branded<'SessionId'>` 在运行时就是 `string`，但在 TypeScript 的类型系统里是不同的类型。这让"把 SessionId 传给需要 SubagentRunId 的函数"成为编译错误，而不是运行时 bug。

## 本章小结

- dsh 的可信度体系是三件套：测试 + 不变量 + 防御模式。
- 每文件 100% 覆盖率门禁 + keyless snapshot（真实 example 重放）。
- 运行时不变量断言拥有的关系，"Model-visible ⟺ logged"是核心。
- fail loud、fail closed、branded ids、边界校验、诚实报告。
- 一句话总结 dsh 的工程性格：**激进插件化 + 偏执的可信度**。

## 阅读思考题

1. 每文件 100% 覆盖率门禁在什么场景下可能过于严格？比如一个只有一行的 re-export 文件，也需要 100% 覆盖吗？dsh 用 `v8 ignore` 注释处理这种情况——这种做法是"务实"还是"作弊"？
2. keyless snapshot 要求"Fixtures must replay on macOS/Linux; fix fixtures, not normalizers"。如果一个测试在 macOS 上通过但在 Linux 上失败（比如路径分隔符不同），dsh 的做法是"修 fixture"而不是"加 normalizer"。为什么？
3. dsh 的可信度清单有 10 条。如果只能选 3 条应用到你的项目，你会选哪 3 条？为什么？

> **全书完**。这 18 章从"什么是 Harness"开始，一路穿过 Cordis 的五块基石、启动组装线、事件日志、主循环、LLM 抽象、工具管线、能力接缝、沙箱、审批、持久化、上下文、子 Agent、Workflow、远程调用，最后收束于可信度体系。dsh 的工程哲学可以浓缩为一句话：**一切皆插件，注册即 effect，模型可见即日志**。
