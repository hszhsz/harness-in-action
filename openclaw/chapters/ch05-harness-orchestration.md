# 第 5 章　Harness 编排层：从"一次循环"到"一场任务"

> 第 4 章我们解剖了那颗心脏——一个 `while(true)` 的单轮循环。
> 但心脏不能裸奔。它需要一具身体：决定用哪颗心脏、心脏跳动前要做哪些准备、上下文撑爆了怎么办、一次尝试失败了如何归类与恢复。
> 这具身体，就是本章的主角——**Harness（运行时编排层）**。

第 4 章结束时，我们停在 `agent-core` 的 `runLoop` 上。它优雅、纯粹、与 provider 无关——但也正因为纯粹，它**什么都不管**：不管模型怎么选、不管 token 撑没撑爆、不管这次跑失败了该不该换个模型重来。这些"脏活累活"全被 `agent-core` 故意推给了宿主。

本章就来看宿主是怎么接住这些活的。我们会精读 OpenClaw 的 Harness 层——它**不在**你以为的 `src/harness/`（那个目录根本不存在），而是分布在 `src/agents/harness/` 与 `src/agents/embedded-agent-runner/` 两处。读完你会理解一件事：**OpenClaw 真正可怕的工程量，不在那 851 行的循环里，而在这层把"单轮循环"包装成"一场完整任务"的编排代码里——光是一个 `attempt.ts` 就有 5400 多行。**

---

## 5.1 一个反直觉的发现：Harness 是一个"可被插件替换"的抽象

打开 `src/agents/harness/types.ts`，第一个让人意外的事实就跳出来了——**OpenClaw 的"运行时"不是写死的一份代码，而是一个叫 `AgentHarness` 的接口契约**：

```ts
export type AgentHarness = AgentHarnessRunCapability &
  AgentHarnessSideQuestionCapability &
  AgentHarnessClassificationCapability &
  AgentHarnessCompactionCapability &
  AgentHarnessSessionLifecycleCapability;
```

它用 TypeScript 的交叉类型，把一个 Harness 拆成了**五种能力（capability）的组合**：

| 能力 | 方法 | 职责 |
|---|---|---|
| `RunCapability` | `supports()` / `runAttempt()` | 声明"我支持哪些 provider/model"，并真正跑一次尝试 |
| `SideQuestionCapability` | `runSideQuestion?()` | 在主任务之外回答一个临时小问题（可选） |
| `ClassificationCapability` | `classify?()` | 把一次运行结果归类（ok / 空回复 / 仅推理…，可选） |
| `CompactionCapability` | `compact?()` | 自己拥有上下文压缩逻辑（可选） |
| `SessionLifecycleCapability` | `reset?()` / `dispose?()` | 会话重置、资源释放（可选） |

注意只有 `RunCapability` 的两个方法是必填的，其余全是可选钩子。这个设计的意图极其清晰：**"能跑一轮"是底线，其他都是增量能力。**

### 为什么要把运行时做成可替换的？

答案藏在 `builtin-openclaw.ts` 这 20 行里——OpenClaw 自己的内置运行时，也只是这个接口的**一个实现**而已：

```ts
export function createOpenClawAgentHarness(): AgentHarness {
  return {
    id: "openclaw",
    label: "OpenClaw embedded agent",
    contextEngineHostCapabilities: OPENCLAW_EMBEDDED_CONTEXT_ENGINE_HOST.capabilities,
    supports: () => ({ supported: true, priority: 0 }),
    runAttempt: runEmbeddedAttempt,   // ← 指向那个 5400 行的庞然大物
  };
}
```

`supports: () => ({ supported: true, priority: 0 })`——内置 Harness **无条件支持一切**，但优先级为 0（最低）。这意味着：任何插件注册的 Harness 只要声明支持当前 provider，就能**抢在内置实现之前**接管这次运行。

> **给 Agent 开发者的启示**：OpenClaw 把"我用什么 Agent 引擎跑这一轮"也变成了一个可插拔点。比如它内置支持把某些模型路由给 Codex 协议的外部 Harness。**你的 Agent 内核如果只硬编码一种执行后端，就永远没法在不改核心代码的前提下接入第二种引擎。** 把"执行后端"抽象成一个 `supports + run` 的小接口，是面向未来的关键一步。

---

## 5.2 注册表与选择器：谁来跑这一轮？

既然 Harness 可以有多个，就必然需要两个配套机制：**一个登记处**和**一个裁判**。

### 注册表（registry.ts）：进程级的全局 Harness 表

`registry.ts` 用了一个值得学习的小技巧——**用全局 Symbol 锚定状态**：

```ts
const AGENT_HARNESS_REGISTRY_STATE = Symbol.for("openclaw.agentHarnessRegistryState");

function getAgentHarnessRegistryState(): AgentHarnessRegistryState {
  const globalState = globalThis as ...;
  globalState[AGENT_HARNESS_REGISTRY_STATE] ??= { harnesses: new Map(...) };
  return globalState[AGENT_HARNESS_REGISTRY_STATE];
}
```

注释解释了为什么这么做：`Symbol.for` 在整个进程里返回同一个 symbol，因此**无论模块被重复 import 多少次、插件如何懒加载、测试如何重置模块**，大家共享的都是同一张 Harness 表。这是在 Node 模块系统里实现"真单例"的稳健写法——比 `let registry = new Map()` 这种模块级变量更抗"模块被多份实例化"的坑。

注册表还暴露了 `resetRegisteredAgentHarnessSessions` 与 `disposeRegisteredAgentHarnesses`，它们用 `Promise.all` 对所有 Harness 扇出（fan-out）生命周期钩子，并且**单个 Harness 的失败不会中断其他 Harness 的清理**（每个 `await` 都包在 try/catch 里只 `log.warn`）。这是"清理逻辑必须尽力而为、不能一个崩全崩"的标准写法。

### 选择器（selection.ts）：一棵决策树

`selectAgentHarnessDecision` 是这层的裁判。它的逻辑可以画成一棵决策树，输出一个带"为什么选它"理由的决策对象。先看它显式建模的几种选择理由：

```ts
selectedReason:
  | "forced_openclaw"                       // 配置强制用内置
  | "forced_plugin"                         // 配置强制用某插件 Harness
  | "implicit_plugin_unavailable_openclaw"  // 隐式想用 Codex 但没注册，退回内置
  | "cli_runtime_passthrough_openclaw"      // CLI 运行时别名，无对应插件，走内置
  | "auto_plugin"                           // auto 模式选中了支持的插件 Harness
  | "auto_openclaw";                        // auto 模式没有插件支持，落回内置
```

**把"为什么做这个选择"显式建模成一个枚举，而不是埋在 if/else 里隐式发生**——这是这段代码最值得抄的工程习惯。当线上出现"为什么这次用了 X 引擎"的疑问时，`selectedReason` 直接给出答案，而 `logAgentHarnessSelection` 会把它连同所有候选项一起打进 debug 日志。

裁判的核心逻辑在 `auto` 模式：

```ts
const candidates = pluginHarnesses.map((harness) => ({
  harness,
  support: harness.supports({ provider, modelId, requestedRuntime: runtime }),
}));
const supported = candidates
  .filter((entry) => entry.support.supported)
  .toSorted(compareHarnessSupport);   // 按 priority 降序，平局按 id 字典序
const selected = supported[0]?.harness;
return selected
  ? /* auto_plugin */
  : /* 没人支持 → auto_openclaw，落回内置 */;
```

排序规则 `compareHarnessSupport` 是"先比 priority（高者胜），priority 相同再按 id 字典序"——**一个完全确定性（deterministic）的排序**，保证同样的输入永远选出同一个 Harness，不会因 Map 遍历顺序而抖动。

### 一道隐蔽但重要的安全闸：插件 Harness 的"拒绝所有工具"传播

选择器里有一段容易被忽略、却体现了"约束与恢复"职责的代码。当选中的是**插件 Harness**（非内置）时，会先经过 `applyPluginHarnessDenyAllToolPolicy`：

```ts
const attemptParams =
  harness.id === "openclaw" ? params : applyPluginHarnessDenyAllToolPolicy(params);
```

它会逐层检查 sender / group / runtime / sandbox / subagent 各级工具策略，一旦发现某层"拒绝所有工具"（`deny: ["*"]`），就**清空 `toolsAllow` 并往系统提示里追加一句话**，明确告诉模型"本会话工具被策略禁用，别假装重试有用"：

```ts
return {
  ...params,
  toolsAllow: [],
  extraSystemPrompt: appendPluginHarnessToolPolicyPrompt(params.extraSystemPrompt, prompt),
};
```

为什么只对插件 Harness 做这件事？因为内置 Harness 在它那 5400 行里自己会完整执行工具策略；而外部插件 Harness 是"黑盒"，OpenClaw 不能假设它会尊重工具策略，于是在**进入黑盒之前**就把"禁用工具"这件事翻译成模型能理解的自然语言指令，从源头堵住越权。**这是"信任边界"的一次精彩实践：对自己人放行，对外部实现做最坏假设。**

---

## 5.3 lifecycle.ts：给每一次尝试套上"可观测的外壳"

裁判选出 Harness 后，运行并不是直接 `harness.runAttempt(params)`，而是经过 `runAgentHarnessLifecycleAttempt` 包一层。这层不改变"跑什么"，只负责"怎么观测地跑"——它是**可观测性（observability）职责**在编排层的落点。

它做了四件事：

**① 上下文引擎宿主能力校验（fail-fast）。** 跑之前先确认"这个 Harness 能不能满足当前上下文引擎的宿主要求"：

```ts
function assertAgentHarnessContextEngineSupport(harness, params) {
  if (!params.contextEngine || params.contextEngine.info.id === "legacy") return;
  assertContextEngineHostSupport({ contextEngine, operation: "agent-run", host: ... });
}
```

不支持就**立刻抛错**，而不是跑到一半才发现能力缺失。

**② 阶段（phase）追踪。** 它用一个 `phase` 变量记录当前走到哪了：

```ts
let phase: AgentHarnessLifecyclePhase = "prepare";
// ...
phase = "send";    const rawResult = await harness.runAttempt(params);
phase = "resolve"; return applyAgentHarnessResultClassification(harness, rawResult, params);
```

一旦中途抛错，`emitAgentHarnessRunError` 会把 `phase` 一起 emit 出去——于是你能区分"是 send 阶段（调模型）挂了，还是 resolve 阶段（解析结果）挂了"。

**③ 诊断事件三连。** 整个过程发出 `harness.run.started` → `harness.run.completed`/`harness.run.error` 三类可信诊断事件，带上 runId、sessionId、provider、model、耗时、结果归类。**注意 `shouldEmitAgentRunDiagnostics(harness)` 对内置 OpenClaw Harness 特意不发额外的 `run.started`/`run.completed`**——因为内置路径自己内部已经有更细的埋点，避免重复。

**④ 结果归类。** `applyAgentHarnessResultClassification` 把原始结果交给 Harness 自己的 `classify?()` 钩子（如果有）做最终归类。这就接上了我们在 `agent-harness-runtime.ts` 里见过的 `classifyAgentHarnessTerminalOutcome`——它把"跑完了但没有可见输出"的回合细分为 `planning-only`（只有计划）、`reasoning-only`（只有推理）、`empty`（纯空）。**为什么要分这么细？因为下游的"模型降级/重试"决策依赖它**：一个"只做了规划没说话"的回合，和一个"彻底空白"的回合，该不该触发 fallback 是不同的。

> **启示**：把"运行"和"观测运行"分成两层（`selection` 调 `lifecycle`，`lifecycle` 再调 `harness.runAttempt`），意味着你可以给任意 Harness——包括第三方插件的——免费套上统一的诊断、追踪、归类外壳，而 Harness 自己完全不需要知道这些埋点的存在。这是"机制/策略分离"在编排层的又一次体现。

---

## 5.4 上下文压缩：Agent 长跑的"续命术"

到这里我们触及了 Harness 层最体现"长期任务"含金量的能力——**上下文压缩（compaction）**。

一个真实的 Agent 任务可能要跑几十上百轮，每轮都往上下文里塞工具结果。token 是有限的，迟早会撑爆。压缩，就是在快撑爆时把历史对话**摘要化**，腾出空间让 Agent 能继续跑下去。这是"状态/记忆"职责里最硬核的一块。

OpenClaw 把压缩拆成了**两个时机**，分别对应两套代码。

### 时机一：抢先压缩（preemptive，发请求之前）

`run/preemptive-compaction.ts` 干的是"**未雨绸缪**"。在把上下文真正发给模型**之前**，先估算这一轮渲染出来的 token 压力，判断会不会越界：

```ts
export function shouldPreemptivelyCompactBeforePrompt(params): PreemptiveCompactionDecision
export function estimateRenderedLlmBoundaryTokenPressure(params): LlmBoundaryTokenPressure
export const PREEMPTIVE_OVERFLOW_ERROR_TEXT = ...   // 预判会溢出时给出的明确错误文案
```

如果预判会溢出，就在发请求前先压一轮——**用一次便宜的本地估算，避免一次昂贵的、注定失败的越界请求**。这正是第 4 章 4.4 节提到的 `transformContext`/`shouldStopAfterTurn` 这类钩子在宿主侧的真实用途之一。

### 时机二：被动压缩与"谁拥有压缩"

`harness/compaction.ts` 的 `maybeCompactAgentHarnessSession` 处理的是另一个问题：**压缩这件事，到底该谁来做？**

```ts
if (!harness.compact) {
  if (harness.id !== "openclaw") {
    return { ok: false, compacted: false,
      reason: `Agent harness "${harness.id}" does not support compaction.`,
      failure: { reason: "unsupported_harness_compaction" } };
  }
  return undefined;   // 内置 Harness：返回 undefined，交还给 embedded runner 的原生压缩路径
}
// 插件 Harness 声明了 compact 钩子 → 由它自己压缩
return harness.compact(resolvedApiKey ? { ...compactParams, resolvedApiKey } : compactParams);
```

这里的语义分层很讲究：

- **内置 OpenClaw**：`compact` 钩子是缺省的，所以这里返回 `undefined`，意思是"我不在 Harness 这层压缩，请走 embedded runner 内部的原生压缩逻辑"。
- **插件 Harness 且声明了 `compact`**：说明它**自己拥有压缩**（`ownsCompaction`），那就尊重它，调它的钩子。
- **插件 Harness 但没声明 `compact`**：直接返回一个明确的失败结果 `unsupported_harness_compaction`——绝不假装压缩成功。

`agent-harness-runtime.ts` 还专门导出了一套 `compactWithSafetyTimeout` / `resolveCompactionTimeoutMs`，注释写得很直白：暴露这套工具，是为了让像 Codex 这样**自己拥有压缩**的插件 Harness，能用**和内置运行器完全一样的、有限的、宿主统一解析的超时**来给自己的 `compact()` 兜底——"一份共享实现，绝不复制粘贴看门狗（watchdog）"。

> **给 Agent 开发者的启示**：压缩不是"快满了就摘要一下"这么简单。它至少要回答三个问题——**何时压（抢先 vs 被动）、谁来压（内置 vs 插件自有）、压超时了怎么办（统一的安全超时）**。OpenClaw 把这三个问题分别落在 `preemptive-compaction.ts`、`compaction.ts`、`compaction-safety-timeout.ts` 三处，边界清清楚楚。如果你的 Agent 想跑长任务，这三个问题一个都躲不掉。

---

## 5.5 embedded-agent-runner：内置 Harness 那 5400 行里到底装了什么

`builtin-openclaw.ts` 把 `runAttempt` 指向了 `runEmbeddedAttempt`——这是 `src/agents/embedded-agent-runner/run/attempt.ts`，**5406 行**，是全仓库最重的单文件之一。我们不可能逐行读完，但有一个数据结构能让你一眼看清它在干什么：`execution-phase.ts` 里的**执行阶段清单**。

### 一次内置尝试的 15 个里程碑

`EMBEDDED_AGENT_EXECUTION_PHASES` 把"跑一轮"细分成了 15 个有序里程碑：

```ts
export const EMBEDDED_AGENT_EXECUTION_PHASES = [
  "runner_entered",            // 1. 进入运行器
  "workspace",                 // 2. 准备工作区
  "runtime_plugins",           // 3. 加载运行时插件
  "before_agent_reply",        // 4. 触发"回复前"钩子
  "model_resolution",          // 5. 解析模型
  "auth",                      // 6. 鉴权 / 取 API Key
  "context_engine",            // 7. 装配上下文引擎
  "attempt_dispatch",          // 8. 分发尝试
  "context_assembled",         // 9. 上下文组装完成
  "turn_accepted",             // 10. 回合被接受
  "process_spawned",           // 11. 进程已派生
  "tool_execution_started",    // 12. 工具执行开始
  "assistant_output_started",  // 13. 助手输出开始
  "model_call_started",        // 14. 模型调用开始
] as const;
```

（数组里 14 项，类型上还有 `model_call_started` 收尾——OpenClaw 用 `as const satisfies Record<...>` 保证标签表与阶段表一一对应，不会漏标。）

读懂这张清单，你就读懂了第 4 章的循环和本章的编排是怎么咬合的：**第 1～11 步全是"开跑之前的准备"，真正进入第 4 章那个 `runLoop` 循环，是从第 12 步 `tool_execution_started` 之后才开始的。** 换句话说，第 4 章那 851 行只是这 5400 行的第 12 步以后的一小段。前面 11 步——工作区、插件、模型解析、鉴权、上下文装配——才是把"裸循环"喂饱、让它能真正跑起来的工程主体。

注释里那句话点明了这张清单的用途：**"Keep labels stable: external status surfaces and diagnostics consume the formatted values."**（保持标签稳定，外部状态展示与诊断会消费这些格式化值。）——这就是你在 TUI 里看到"正在解析模型…""正在装配上下文…"这类实时状态的数据来源。

### 它如何接回第 4 章的循环

第 4 章我们说，`agent-core` 的循环只认一个注入进来的 `streamSimple`。在 `embedded-agent-runner` 里，这个注入是怎么完成的？答案在 `stream-resolution.ts`：

```ts
import { streamSimple } from "../../llm/stream.js";
import type { StreamFn } from "../runtime/index.js";
```

它从 `llm/stream.js` 拿到真正会调 provider 的 `streamSimple`，并把它包装成各种"边界感知/Vertex 适配"的 `StreamFn`，最终通过第 3 章见过的 `src/agents/runtime/index.ts`——那个 `extends CoreAgent`、`openClawAgentCoreRuntime = { streamSimple, completeSimple }` 的门面——注入给 `agent-core`。

至此，我们把第 3、4、5 章串成了一条完整的因果链：

```
src/agents/runtime/index.ts          (第3章：DI 门面，注入 streamSimple)
        ↓ extends / 注入
packages/agent-core/runLoop          (第4章：与 provider 无关的纯循环)
        ↑ 被调用
embedded-agent-runner/run/attempt.ts (第5章：15 阶段编排，第12步起进入循环)
        ↑ runAttempt
harness/builtin-openclaw.ts          (第5章：把内置运行器包成 AgentHarness)
        ↑ 被选中
harness/selection.ts → lifecycle.ts  (第5章：裁判 + 可观测外壳)
```

**循环是心脏，runtime 是供血的血管，而 Harness 编排层是决定"这颗心脏要不要跳、怎么跳、跳之前做哪些准备、跳坏了如何抢救"的整套神经与免疫系统。**

---

## 5.6 把六根支柱对回来

第 1 章我们立了 Harness 工程的六根支柱。读完本章，正好可以把它们和这层代码一一对上——这也是检验"我们真的理解了 Harness"的最好方式：

| 支柱 | 本章对应实现 |
|---|---|
| **上下文管理** | `preemptive-compaction.ts` 抢先压缩 + `compaction.ts` 被动压缩 + `compaction-safety-timeout.ts` 安全超时 |
| **工具系统** | `selection.ts` 的"拒绝所有工具"策略传播；内置 runner 第 12 阶段 `tool_execution_started` |
| **执行编排** | `execution-phase.ts` 的 15 个有序里程碑；`selection.ts` 的确定性 Harness 选择 |
| **状态/记忆** | `registry.ts` 进程级会话表 + `reset`/`dispose` 生命周期钩子 |
| **评估/可观测** | `lifecycle.ts` 的 started/completed/error 诊断三连 + phase 追踪 + 结果归类 |
| **约束/恢复** | 上下文引擎宿主能力 fail-fast 校验；插件 Harness 的最坏假设 deny-all 兜底 |

六根支柱，没有一根是悬空的。它们全都沉淀成了 `src/agents/harness/` 这二十几个文件里可运行、可测试、可观测的真实代码。

---

## 本章小结

- OpenClaw 的"运行时"不是写死的，而是一个 `AgentHarness` 接口——由 run/sideQuestion/classify/compact/lifecycle **五种能力**交叉组合而成，只有 `supports + runAttempt` 是必填。
- 内置 OpenClaw 运行时（`builtin-openclaw.ts`）也只是该接口的一个实现，**无条件支持、优先级 0**，任何插件 Harness 都能凭更高优先级抢先接管。
- `registry.ts` 用全局 `Symbol.for` 实现进程级单例 Harness 表；`selection.ts` 是一棵把"为什么选它"显式建模成 `selectedReason` 枚举的确定性决策树。
- 选择器对**插件 Harness**做"拒绝所有工具"的最坏假设兜底，把工具策略翻译成系统提示——对自己人放行、对外部黑盒设防的信任边界实践。
- `lifecycle.ts` 给每次尝试套上"可观测外壳"：上下文引擎能力 fail-fast、phase 追踪、started/completed/error 诊断三连、结果归类（planning-only/reasoning-only/empty）。
- 上下文压缩分**抢先**（`preemptive-compaction.ts`，发请求前估算并预压）与**被动**（`compaction.ts`，按"谁拥有压缩"路由到内置或插件），并由统一的安全超时兜底。
- 内置尝试 `attempt.ts` 多达 5406 行，用 `execution-phase.ts` 的 **15 个有序里程碑**编排；第 4 章的 `runLoop` 只是其中第 12 步 `tool_execution_started` 之后的一小段——**准备工作才是工程主体**。
- 第 3/4/5 章串成一条因果链：runtime 门面注入 `streamSimple` → agent-core 纯循环 → embedded runner 15 阶段编排 → 包装成 Harness → 选择器+生命周期外壳。

下一章，我们钻进这 15 个阶段里最神秘的两个——`context_engine` 与 `context_assembled`，看 OpenClaw 的**上下文引擎**到底是如何把"散落的会话历史、文件、技能、记忆"装配成一份发给模型的最终上下文的。

---

> **动手实验（建议在读第 6 章前完成）**
> 1. 打开 `src/agents/harness/selection.test.ts`（33KB，是全 harness 目录测试量最大的文件）。找出测试用例是如何覆盖那六种 `selectedReason` 的——尤其是 `auto_plugin` 与 `auto_openclaw` 的分叉。这会让你彻底明白"裁判"在边界条件下的行为。
> 2. 在 `execution-phase.ts` 的 15 个阶段里，用一句话向同事解释："为什么 `auth`（取 API Key）被放在第 6 步、而不是第 1 步？"（提示：回想第 4 章 4.4 节"每轮重新解析 API Key 防 token 过期"——和这里的阶段顺序有什么呼应？）
