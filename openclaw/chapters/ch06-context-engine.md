# 第 6 章　上下文引擎：决定模型"看见什么"的那只手

> 模型本身没有记忆。它每一轮看到的，只有你这一轮塞给它的那段文本。
> 所以 Agent 的"智能"上限，从来不取决于模型多聪明，而取决于**你能不能在有限的窗口里，把最该让它看见的东西，按最该有的顺序，喂到它眼前**。
> 这件事，OpenClaw 交给了一个叫"上下文引擎"的子系统。

第 5 章的 15 个执行阶段里，藏着两个我们当时一笔带过的里程碑：`context_engine`（第 7 步，装配上下文引擎）和 `context_assembled`（第 9 步，上下文组装完成）。这两步之间发生的事，就是本章的全部内容。

读完本章你会理解一个常被忽视的真相：**对一个 Agent 来说，"上下文工程"和"模型选择"同等重要，甚至更重要。** 而 OpenClaw 把这件事做成了一个**可插拔、能降级、带能力校验**的独立子系统——`src/context-engine/`。

---

## 6.1 一个接口定义的"上下文生命周期"

打开 `src/context-engine/types.ts`，核心是一个 387 行文件里的 `ContextEngine` 接口。它没有把"上下文管理"当成一个函数，而是当成一套**有生命周期的契约**：

```ts
export interface ContextEngine {
  readonly info: ContextEngineInfo;

  bootstrap?(...): Promise<BootstrapResult>;        // 会话初始化（可选）
  ingest(...): Promise<IngestResult>;               // 摄入单条消息（必填）
  ingestBatch?(...): Promise<IngestBatchResult>;    // 摄入一整轮（可选）
  assemble(...): Promise<AssembleResult>;           // 组装出发给模型的上下文（必填）
  afterTurn?(...): Promise<void>;                   // 回合结束后的善后（可选）
  maintain?(...): Promise<...>;                     // 转录维护/重写（可选）
  compact(...): Promise<CompactResult>;             // 压缩（必填）
  prepareSubagentSpawn?(...): Promise<...>;         // 子 Agent 派生前准备（可选）
  onSubagentEnded?(...): Promise<void>;             // 子 Agent 结束通知（可选）
  dispose?(): Promise<void>;                        // 资源释放（可选）
}
```

只有三个方法是必填的——`ingest`（怎么把消息存进来）、`assemble`（怎么把上下文拼出去）、`compact`（怎么在撑爆时压缩）。其余全是可选钩子。这个"必填三件套"恰好对应了一个上下文引擎不可回避的三个动作：**进、出、瘦身**。

### `assemble` 的返回值才是真正的灵魂

整个接口里最关键的方法是 `assemble`——它就是第 5 章那个 `context_assembled` 阶段的执行体。看它的返回类型 `AssembleResult`：

```ts
export type AssembleResult = {
  messages: AgentMessage[];                 // 有序的、要发给模型的最终消息
  estimatedTokens: number;                  // 估算总 token
  promptAuthority?: "assembled" | "preassembly_may_overflow";
  systemPromptAddition?: string;            // 可选：要追加到系统提示的引擎指令
  contextProjection?: ContextEngineProjection;  // 可选：投影生命周期
};
```

这里有两个设计细节，体现了"上下文工程"的真实难度：

**① `promptAuthority`——估算谁说了算。** 注释说得很清楚：返回的 `messages` 永远是发给模型的内容，但**用哪个 token 估算去做"抢先压缩"的越界预判**是可调的。默认 `"assembled"`（只看组装后的估算）；但有些引擎的"组装后视图"可能**掩盖了底层转录里其实存在的溢出**，于是它们可以选 `"preassembly_may_overflow"`，让预检取"组装后估算"和"未开窗的完整历史估算"中的较大值。这正是第 5 章 5.4 节抢先压缩的数据来源——**压缩决策的准确性，源头就在这个字段**。

**② `contextProjection`——投影模式。** 对于像 Codex 那种"后端持有持久线程"的宿主，每轮重新注入整段上下文是浪费。引擎可以返回 `mode: "thread_bootstrap"` 配一个 `epoch`，告诉宿主"这段上下文只在这个 epoch 注入一次，之后复用后端线程，直到 epoch 变化再轮换"。这是为"有状态后端"做的性能优化——而默认的 `per_turn` 则是无状态的"每轮全量投影"。

> **给 Agent 开发者的启示**：很多人把"组装上下文"理解成"把历史消息拼成一个数组"。OpenClaw 告诉你，一个生产级的 `assemble` 至少还要回答：**我组装后的估算可信吗（promptAuthority）？我能不能复用后端线程省钱（projection）？我要不要往系统提示里塞点引擎专属指令（systemPromptAddition）？** 上下文工程的复杂度，全藏在这些"附加返回值"里。

---

## 6.2 默认实现的反转：LegacyContextEngine 几乎什么都不做

读到这里你大概以为，OpenClaw 的默认上下文引擎一定是个庞然大物。打开 `legacy.ts`——只有 88 行，而且核心方法**全是空操作或透传**：

```ts
export class LegacyContextEngine implements ContextEngine {
  readonly info = { id: "legacy", name: "Legacy Context Engine", version: "1.0.0" };

  async ingest(): Promise<IngestResult> {
    return { ingested: false };   // 空操作：SessionManager 自己负责持久化
  }

  async assemble(params): Promise<AssembleResult> {
    // 透传：真正的 sanitize → validate → limit → repair 流水线在 attempt.ts 里
    return { messages: params.messages, estimatedTokens: 0 };
  }

  async afterTurn(): Promise<void> { /* 空操作 */ }

  async compact(params): Promise<CompactResult> {
    return await delegateCompactionToRuntime(params);   // 委托回运行器原生压缩
  }
}
```

这个"反转"恰恰是理解 OpenClaw 上下文架构的钥匙。注释把真相点破了：

> *"assemble: pass-through (existing sanitize/validate/limit pipeline in attempt.ts handles this)"*

也就是说——**默认情况下，真正的上下文组装逻辑根本不在 `context-engine/` 这个包里，而在第 5 章那个 5400 行的 `attempt.ts` 里。** `context-engine` 包提供的是一个**抽象层和插槽**，而 `LegacyContextEngine` 是一个"什么都不拦，让老逻辑继续跑"的兼容垫片，目的写在注释里：**"preserving 100% backward compatibility"**。

为什么要这样设计？因为这是一次**没有大爆炸的架构演进**：OpenClaw 想引入"可插拔上下文引擎"这个新抽象，但又不想把已经稳定运行的老组装逻辑推倒重写。于是它先定义接口、提供一个透传的 legacy 实现兜底，让**新插件引擎可以渐进式接管**，而老路径一行都不用动。

> **这是"核心精简"红线（第 2 章）的又一次体现**：新增能力时，优先用"接口 + 透传默认实现"的方式接缝化，而不是把新逻辑硬塞进老代码。

---

## 6.3 真正的"上下文配方"：legacy 路径到底拼了些什么

既然 legacy 的 `assemble` 是透传，那真正发给模型的上下文是怎么拼出来的？答案分散在两处：**系统提示的构建**与**会话历史的清洗**。

### 配方一：系统提示——`buildEmbeddedSystemPrompt` 的几十种配料

`system-prompt.ts` 里的 `buildEmbeddedSystemPrompt`，是把"模型该知道的一切背景"压成一段系统提示的地方。它的入参清单本身就是一张"上下文配料表"：

| 配料类别 | 字段（节选） | 作用 |
|---|---|---|
| **运行环境** | `runtimeInfo`（host/os/arch/node/model/provider/channel） | 让模型知道自己跑在什么机器、什么渠道、什么模型上 |
| **技能与文档** | `skillsPrompt` / `docsPath` / `sourcePath` | 注入可用 Skill 列表与文档路径 |
| **记忆** | `includeMemorySection` / `memoryCitationsMode` | 是否注入长期记忆段及其引用模式 |
| **工作区文件** | `contextFiles` / `workspaceNotes` | 把 bootstrap 文件内容带进上下文（见配方二） |
| **沙箱** | `sandboxInfo` | 告诉模型当前沙箱的能力边界 |
| **工具** | `tools`（取 `tool.name`） | 让系统提示与运行时实际可用工具对齐 |
| **时间** | `userTimezone` / `userTime` / `userTimeFormat` | 注入用户时区与当前时间 |
| **行为模式** | `promptMode` / `silentReplyPromptMode` / `subagentDelegationMode` | 控制硬编码段落的取舍 |

注意 `tools: params.tools.map((tool) => tool.name)` 这一行——**系统提示里描述的可用工具，和运行时真正注册的工具，是同一份来源**。这避免了一类隐蔽 bug：模型以为某工具可用、实际却没注册，于是反复幻觉调用。OpenClaw 在 `assemble` 的入参里专门带了 `availableTools?: Set<string>`，注释写明用途是"让引擎把提示指引与运行时工具访问对齐"。

### 配方二：工作区引导文件——`bootstrap-files`

`bootstrap-files.ts` 负责把工作区里的引导文件（比如项目根的 `AGENTS.md`/`CLAUDE.md` 这类约定文件）读进来，变成上下文。它的关键在于**有界注入**：

```ts
export function buildBootstrapContextForFiles(bootstrapFiles, params): EmbeddedContextFile[] {
  return buildBootstrapContextFiles(bootstrapFiles, {
    maxChars: resolveBootstrapMaxChars(params.config, params.agentId),       // 单文件上限
    totalMaxChars: resolveBootstrapTotalMaxChars(params.config, params.agentId), // 总量上限
    warn: params.warn,
  });
}
```

**每个文件有单独的 `maxChars` 上限、所有文件合起来还有 `totalMaxChars` 总上限。** 超了就截断并 `warn`。这是上下文工程里最朴素也最重要的纪律：**任何注入源都必须有界，否则一个超大的工作区文件就能把整个上下文窗口吃光。**

### 配方三：历史清洗——sanitize / repair

发给模型之前，会话历史还要过一道清洗。`attempt.ts` 里引用了多个修复器：

```ts
import { repairSessionFileIfNeeded } from "../../session-file-repair.js";
import { sanitizeToolUseResultPairing } from "../../session-transcript-repair.js";
import { sanitizeSessionHistory } from "...";
```

它们解决的是一类很现实的脏数据问题：**工具调用（toolCall）和工具结果（toolResult）必须成对出现**——如果上一轮跑到一半崩了，留下一个"有调用没结果"的孤儿，下一轮直接发给模型，provider 会因消息配对不合法而拒绝请求。`sanitizeToolUseResultPairing` / `repairAttemptToolUseResultPairing` 就是来修补这种断裂的配对的。

> **启示**：legacy 路径的"配方"告诉我们，一份发给模型的上下文 = **系统提示（环境+技能+记忆+工具+时间）+ 有界注入的工作区文件 + 清洗修复过的会话历史**。这三者缺一不可，而且每一项都需要"有界"和"自愈"。把这张配方表记下来，你就有了设计任何 Agent 上下文层的检查清单。

---

## 6.4 注册表：可插拔，但"失败就隔离"

`registry.ts`（1011 行）是上下文引擎的登记处与解析器。它和第 5 章的 Harness 注册表共享同一套"全局单例"哲学（这里用的是 `resolveGlobalSingleton`），但多了一个极其重要的安全机制：**隔离（quarantine）**。

### 解析顺序与降级

`resolveContextEngine` 的解析逻辑很直接：

```ts
const slotValue = config?.plugins?.slots?.contextEngine;
const engineId = (slotValue?.trim()) || defaultSlotIdForKey("contextEngine");  // 默认 "legacy"
```

先看配置里 `plugins.slots.contextEngine` 这个插槽指定了谁，没指定就用默认的 `legacy`。但接下来的处理才是精华——**对非默认引擎，任何一步失败都会被"隔离"并静默降级回默认引擎**：

```ts
// 1. 已被隔离的自定义引擎，直接降级
const quarantine = !isDefaultEngine ? getContextEngineQuarantine(engineId) : undefined;
if (quarantine) return resolveDefaultContextEngine(defaultEngineId, factoryCtx);

// 2. 没注册 → 隔离 + 降级
if (!entry) {
  if (isDefaultEngine) throw new Error(`Context engine "${engineId}" is not registered. ...`);
  recordContextEngineQuarantine({ engineId, operation: "resolve", error: "not registered", ... });
  return resolveDefaultContextEngine(defaultEngineId, factoryCtx);
}

// 3. 工厂抛错 → 隔离 + 降级
try { engine = await entry.factory(factoryCtx); }
catch (factoryError) {
  if (isDefaultEngine) throw factoryError;
  recordContextEngineQuarantine({ engineId, operation: "factory", ... });
  return resolveDefaultContextEngine(defaultEngineId, factoryCtx);
}

// 4. 契约校验不过 → 隔离 + 降级
const contractError = describeResolvedContextEngineContractError(engineId, engine);
if (contractError) { /* 同上：隔离 + 降级 */ }
```

注意贯穿始终的那个判断：**`if (isDefaultEngine) throw`**。这是整个降级策略的安全底线——

- **非默认（插件）引擎失败**：隔离它，静默降级回 legacy。用户的 Agent 不会因为一个第三方上下文引擎插件写崩了而彻底罢工。
- **默认引擎本身失败**：**没有下一层兜底了，必须抛错**。如果连 legacy 都解析不了，那是真出大事了，不能假装无事发生。

"隔离"的语义是：被隔离的引擎会一直保持降级状态，直到显式清除隔离或进程重启。这避免了"每一轮都重试一个注定失败的插件引擎"的无谓开销和日志刷屏。

> **给 Agent 开发者的启示**：可插拔 ≠ 脆弱。OpenClaw 把"可插拔上下文引擎"和"插件失败必须优雅降级"做成了一体两面——**给扩展点配一个永远兜底的默认实现 + 一个把坏插件隔离掉的熔断器**，是开放扩展性而不牺牲稳定性的标准做法。这套 `quarantine → fallback to default → 只有默认失败才 throw` 的三段式，值得你在任何插件系统里复刻。

---

## 6.5 能力校验：不是所有宿主都配得上所有引擎

最后一块拼图，是 `host-compat.ts`。它回答一个微妙的问题：**一个高级上下文引擎，凭什么相信当前宿主能正确地运行它？**

上下文引擎可能依赖一些宿主才能提供的能力——比如"能在发请求前组装"、"能跑回合后维护"、"能提供 LLM 推理给引擎自己用"。这些被建模成一组 `ContextEngineHostCapability`：

```ts
export type ContextEngineHostCapability =
  | "bootstrap"
  | "assemble-before-prompt"
  | "after-turn"
  | "maintain"
  | "compact"
  | "runtime-llm-complete"
  | "thread-bootstrap-projection";
```

不同宿主声明自己拥有哪些能力。比如内置嵌入式宿主 `OPENCLAW_EMBEDDED_CONTEXT_ENGINE_HOST` 和 Codex 应用服务器宿主 `CODEX_APP_SERVER_CONTEXT_ENGINE_HOST` 各有一份能力清单。而引擎可以在 `info.hostRequirements` 里声明"我执行 `agent-run` 操作时，宿主必须具备哪些能力"。

校验发生在第 5 章 5.3 节我们见过的 `lifecycle.ts` 里——它调用 `assertContextEngineHostSupport`，一旦发现能力缺失就**抛出一条信息极其详尽的错误**：

```ts
`Missing host capabilities: ${missing.join(", ")}. ` +
`Required capabilities: ${required}. ` +
`Host capabilities: ${actual}.${guidance}`
```

它不只说"不支持"，而是把"缺了哪些、要求哪些、宿主有哪些"三者都列出来——这是为排障而设计的错误信息。

为什么需要这道校验？注释里有一句话点破了：宿主能力不匹配时，强行运行"会悄无声息地降级或损坏引擎的行为"（*silently degrade or corrupt the engine's behavior*）。**比起让一个高级引擎在不兼容的宿主上"看起来在跑、实际在产出垃圾上下文"，OpenClaw 选择在入口处 fail-fast。** 这是"约束与恢复"职责里"宁可明确报错，不可静默错误"原则的又一次贯彻。

---

## 6.6 把生命周期串起来：一次回合中的上下文引擎

现在把本章的零件，按一次真实回合的时间线串成一条链。`src/agents/harness/context-engine-lifecycle.ts` 提供的几个 helper，正好标出了这条链上的关键节点：

```
回合开始
  │
  ├─ resolveContextEngine()              解析引擎（含隔离/降级）        ← registry.ts
  ├─ assertContextEngineHostSupport()    宿主能力校验（fail-fast）      ← host-compat.ts
  ├─ bootstrapHarnessContextEngine()     首轮初始化引擎状态            ← bootstrap?()
  │
  │   ┌──────── 第 5 章 15 阶段的第 7 步 context_engine 到此为止 ────────┐
  │
  ├─ assembleHarnessContextEngine()      组装上下文（legacy 透传 →       ← assemble()
  │                                       attempt.ts 真正拼：系统提示
  │                                       + bootstrap 文件 + 清洗历史）
  │
  │   └──────── 第 9 步 context_assembled 完成，上下文就绪 ────────────┘
  │
  ├─ （进入第 4 章的 runLoop，模型开始生成、调工具）
  │
  ├─ finalizeHarnessContextEngineTurn()  回合善后                       ← afterTurn()
  ├─ runHarnessContextEngineMaintenance() 转录维护/安全重写             ← maintain()
  │
回合结束（必要时触发 compact()，见第 5 章 5.4）
```

这条链让第 4、5、6 三章彻底闭环：

- **第 4 章**给了心脏（纯循环）；
- **第 5 章**给了把心脏喂饱的 15 阶段编排，其中第 7、9 两步是"装配/组装上下文"；
- **第 6 章**告诉你那两步背后，是一个**可插拔、能隔离降级、带宿主能力校验**的上下文引擎子系统，而它的默认实现刻意保持透传，把真正的组装配方留在 `attempt.ts` 里以保证零破坏演进。

**模型决定"能想多深"，上下文引擎决定"能看见什么"。** 一个 Agent 的实际表现，是这两者的乘积——而后者，往往是工程上更可控、也更被低估的那一个因子。

---

## 本章小结

- 上下文引擎是 `ContextEngine` 接口定义的一套**有生命周期的契约**：必填 `ingest`（进）/`assemble`（出）/`compact`（瘦身），其余皆为可选钩子。
- `assemble` 的返回值 `AssembleResult` 才是灵魂——`promptAuthority` 决定抢先压缩的估算口径，`contextProjection` 决定能否复用后端线程省钱，`systemPromptAddition` 允许引擎注入专属指令。
- 默认的 `LegacyContextEngine` 仅 88 行且几乎全是透传/空操作——真正的组装逻辑仍在 `attempt.ts`。这是一次"接口 + 透传默认实现"的零破坏架构演进，保留 100% 向后兼容。
- legacy 路径的"上下文配方" = **系统提示**（`buildEmbeddedSystemPrompt`：环境+技能+记忆+工具+时间）+ **有界注入的工作区文件**（`bootstrap-files`，单文件与总量双重 maxChars）+ **清洗修复过的会话历史**（sanitize/repair 工具调用配对）。
- `registry.ts` 用全局单例登记引擎，并以 **quarantine（隔离）→ fallback to default → 只有默认引擎失败才 throw** 的三段式，实现"可插拔但坏插件必优雅降级"。
- `host-compat.ts` 用 7 种 `ContextEngineHostCapability` 做宿主能力校验，缺能力即 fail-fast 并给出"缺哪些/要哪些/有哪些"的详尽错误——宁可明确报错，不可静默产出垃圾上下文。
- 第 4/5/6 章闭环：纯循环（心脏）← 15 阶段编排（喂饱心脏）← 上下文引擎（决定心脏看见什么）。

下一章，我们转向另一根支柱——**工具系统**。看 OpenClaw 如何定义工具、如何把上百个工具按策略筛选与分组、以及那道我们在第 4 章 4.5 节惊鸿一瞥的 `beforeToolCall` 审批闸门，在工具侧到底长什么样。

---

> **动手实验（建议在读第 7 章前完成）**
> 1. 打开 `src/context-engine/legacy.ts`，把它和 `types.ts` 里的 `ContextEngine` 接口对照看——数一数 legacy 实现了几个方法、空操作了几个。然后用一句话回答："如果我要写一个真正做检索增强（RAG）的上下文引擎，最少需要改写哪几个方法？"
> 2. 在 `registry.ts` 的 `resolveContextEngine` 里找到那四处 `if (isDefaultEngine) throw`。向同事解释："为什么插件引擎失败要静默降级，而默认引擎失败必须抛错？把这两种行为对调会发生什么灾难？"
