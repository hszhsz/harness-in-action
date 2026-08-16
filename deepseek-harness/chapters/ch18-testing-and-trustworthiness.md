# 第 18 章　测试与可信度：怎么知道改好了

> dsh 在"可信度"上的投入可能是所有已解读 Agent 里最重的：keyless snapshot 测试、运行时不变量注册表、每文件 100% 覆盖率门禁、防御模式文档、几十个 verify-* 门禁脚本。这一章讲它的工程可信度体系。

## 18.1　分层测试：vitest 单测、e2e、keyless snapshot

dsh 的测试体系（`package.json` scripts）：

- **`test`**：vitest 单测；
- **`test:coverage`**：CI 覆盖率门禁——**`packages/*/*/src` 每文件 100% 覆盖率**（"CI coverage gate: per-file 100% on packages/*/*/src"）。这是极端严苛的门槛，迫使每个分支都被测试覆盖；
- **`test:e2e`**：真实 API 测试，无 `DEEPSEEK_API_KEY` 自动跳过；
- **`test:snapshot`**：**keyless** ACP/headless 重放对比期望输出——不需要 API key 的模型行为测试。`test:snapshot:record` 重新录制（需要 key），`test:snapshot:refresh` 刷新；
- **`test:web` / `test:web:perf` / `test:web:stress` / `test:gui`**：Web/性能/压力/GUI 分层；
- **`check:all` / `check:ci`**：`scripts/run-gates.ts` 的完整门禁编排。

**关键测试哲学**（`AGENTS.md`）：

> **Tests describe behavior, not correctness.** Change obsolete behavior with its tests; explain why in the PR.

> Every non-trivial model- or product-user-visible behavior change adds or updates a keyless snapshot through a real runnable example in the same PR; package tests, e2e-only assertions, and mock-only fixtures do not substitute for the assembled application transcript. Fixtures must replay on macOS/Linux; fix fixtures, not normalizers.

**keyless snapshot 是硬要求**：任何非平凡的模型/产品用户可见行为变更，必须带一个真实可运行的 example + keyless snapshot——mock-only fixtures 不算数。"Fix fixtures, not normalizers"——修 fixture，不修 normalizer（防止掩盖真实差异）。

## 18.2　运行时不变量注册表（invariants）

`docs/subsystems/invariants.md`：运行时 invariant 注册表——选择 `Config`、`InvariantInstaller`/`InvariantFailure`、empty-companion 契约。

`AGENTS.md` 的核心规则：

> **Runtime invariants assert owned relationships.** Check authoritative event streams or mutable data, not service or method presence, plugin metadata or effects, or fixed pure examples. Without a plausible relationship, an explained empty companion is correct.

**不变量断言"拥有的关系"**——检查权威事件流或可变数据，而不是检查"服务/方法存在"、"插件元数据/effect"或固定纯例子。没有合理关系时，解释过的空同伴是正确的（"an explained empty companion is correct"）。

最典型的运行时不变量就是 "Model-visible ⟺ logged"（第 5.2 节）——任何到达模型请求的东西必须能从 session 日志重建。这类不变量被写进代码、在运行时断言，而不是只停留在文档里。

## 18.3　防御模式（defensive patterns）与 fail loud 哲学

`docs/defensive-patterns.md` 是生命周期、并发、subprocess、teardown 工作的必读。`AGENTS.md` 的防御规则精选：

- **Misconfiguration fails loud**：装载期自包含的错立刻报；否则在最早可解析点报。**绝不静默跳过缺失的引用**（"never silently skip a missing referent"）。
- **Opaque cross-boundary ids are branded**：跨边界的 id 用 `Branded<B>`，绝不用裸 `string`。
- **Trust TypeScript at typed same-process boundaries**：不在静态接口要求处加运行时校验/fallback/敌对输入测试；在 parser/config、queued、model/tool JSON、durable/file、worker、process、wire 边界校验。
- **An empty `catch` names what it swallows**：空 catch 必须说明吞掉了什么、为什么没有别的东西能到达它；`try` 只包一个语句。
- **Fail loud 的启动**：`installFailLoud`（第 4.3 节）——启动失败 = 一行带标签 stderr + exit(1)，先恢复终端再退出。
- **Approval fail-closed**（第 12.1 节）：应答者出任何问题都 `unavailable`，门是关的。
- **Sandbox 诚实报告**（第 11.3 节）：`partial` 就是 `partial`，不假装更安全。

## 18.4　从 DeepSeek Harness 借鉴：插件化 Harness 的最小清单

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

## 本章小结

dsh 的可信度体系是三件套：

1. **测试**：分层 vitest + keyless snapshot（真实 example 重放）+ 每文件 100% 覆盖率门禁；
2. **不变量**：运行时注册表断言拥有的关系，Model-visible ⟺ logged 是核心；
3. **防御模式**：fail loud、fail closed、branded ids、边界校验、诚实报告。

**一句话总结 dsh 的工程性格**：激进插件化 + 偏执的可信度。它敢把整个产品做成可替换的插件树，是因为每一层都有运行时断言、门禁测试和防御模式兜底。

## 动手实验

1. 跑 `pnpm run test:snapshot`（无 key 也能跑），看看 keyless 重放长什么样。
2. 读 `scripts/run-gates.ts`，列出 `check-all` 包含的所有 gate。
3. 在 `packages/core/session` 里找一个运行时 invariant，确认它断言的是"关系"而不是"存在"。
4. 读 `docs/defensive-patterns.md`，挑三条规则，在源码里各找一个应用实例。
