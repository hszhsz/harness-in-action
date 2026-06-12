# 第 16 章　可信度工程：测试、faux provider 与供应链

> 一个 Agent 的内核越精简,越要有信心相信它「真的对」。pi 不靠堆集成测试来买信心,而是从三处下手:一个不联网、可编排的 `faux` provider,让整条循环可被确定性地测;一套「Node strip-only」类型纪律,少一层构建;以及「用自己开发自己」的 dogfooding。这是全书最后一章,也是把前十五章的工程决策收口成「可信系统」的一章。

我们已经把 pi 的每个子系统拆开看过了:模型契约、双层循环、Harness 编排、会话树、压缩、工具、扩展、TUI。但还有一个贯穿始终的问题没正面回答:**你怎么知道这一切是对的?** Agent 系统最难测——它依赖一个不确定、要花钱、会限流的外部大模型;它的核心价值恰恰是「多轮工具调用」这种难以复现的长链路。这一章看 pi 怎么把「可信度」做成工程,而不是靠运气。

---

## 16.1 `faux` provider:不联网、不烧 token 地测整条循环

第 6、7 章讲过,pi 的所有 Provider 都注册到同一张 `api-registry`,对上暴露统一的 `stream` 契约。这条接缝最大的红利,在测试时才完全兑现:**你可以注册一个假的 Provider,让整条 Agent 循环跑在它上面,完全不碰网络。** 这就是 `packages/ai/src/providers/faux.ts`。

faux 不是 mock 库里那种「打桩单个函数」,而是一个**完整实现了 `StreamFunction` 契约的合成 Provider**——对 `runLoop`、`AgentHarness` 来说,它和真的 Anthropic provider 长得一模一样。关键在于它的响应是你**预先编排**好的脚本:

```ts
export type FauxResponseFactory = (
	context: Context,
	options: StreamOptions | undefined,
	state: { callCount: number },
	model: Model<string>,
) => AssistantMessage | Promise<AssistantMessage>;

export type FauxResponseStep = AssistantMessage | FauxResponseFactory;

export interface FauxProviderRegistration {
	// ……
	state: { callCount: number };
	setResponses: (responses: FauxResponseStep[]) => void;
	appendResponses: (responses: FauxResponseStep[]) => void;
	getPendingResponseCount: () => number;
	unregister: () => void;
}
```

配上一组工厂函数,你能用几行代码「写出」模型本回合该说什么:

```ts
fauxText("好的,我来读这个文件");
fauxToolCall("read", { path: "a.txt" });          // 让模型「决定」调 read 工具
fauxAssistantMessage([fauxText("内容是……")], { stopReason: "stop" });
```

`setResponses([...])` 把一串响应排进队列,Agent 每调一次模型就消费一步——`state.callCount` 记录到了第几步。于是一次「模型先调 read 工具 → 拿到结果 → 再总结」的完整多轮链路,可以被**逐字编排成一个确定性测试**:第一步返回一个 `fauxToolCall("read",...)`,循环真的去执行 read 工具(真的读文件、真的把结果回灌),第二步返回 `fauxText(...)` 收尾。整个 `runLoop` 的工具抽取、执行、回灌、停止判定全程被真实跑过,唯独模型是假的。

> **给 Agent 开发者的启示**:Agent 测试的死穴是「模型不确定 + 要花钱 + 慢」。很多团队的反应是「那就少测、靠手点」。pi 的解法是把不确定性**锁在一个清晰的接缝后面**——因为 Provider 早就是可插拔的(第 6 章),换上 faux 就把「不确定的模型」变成「我说它返回什么它就返回什么」的脚本,而被测的循环逻辑是**真实代码**。这就是「依赖倒置」在 Agent 工程里的最高回报:**把唯一不可控的依赖做成可注入的契约,剩下的一切都能确定性地测**。`callCount` 状态还让你能写「第一次失败、第二次成功」的重试测试——连错误恢复路径都能精确复现。

`faux` 被 `packages/ai/src/index.ts` 正式导出,不是测试专用的私货,而是 pi 公开 API 的一部分。`AGENTS.md` 里更把它写成硬规矩:`packages/coding-agent/test/suite/` 必须用 `test/suite/harness.ts` + faux provider,**「No real provider APIs, keys, or paid tokens」**。issue 回归测试统一放 `test/suite/regressions/`,命名 `<issue-number>-<short-slug>.test.ts`——每个 bug 都有一个不花钱、可永久复现的守门测试。

## 16.2 TypeBox + Node strip-only:少一层构建的类型纪律

第 8 章讲工具 schema 时提过 TypeBox。这里要点出它背后一条更深的工程取舍:pi 刻意约束自己**只用「可擦除的 TypeScript 语法」(Node strip-only mode)**。`AGENTS.md` 第 20 条写得明明白白:

> Use only erasable TypeScript syntax (Node strip-only mode) ... no parameter properties, `enum`, `namespace`/`module`, `import =`, `export =`, or other constructs needing JS emit. Use explicit fields with constructor assignments.

「strip-only」是 Node 较新版本的能力:它能直接运行 `.ts` 文件,办法是**把类型标注当注释一样剥掉**,而不做任何类型到 JS 的转译。代价是:任何「需要生成额外 JS 代码」的 TS 特性都不能用——`enum`(要生成对象)、`namespace`(要生成 IIFE)、参数属性(`constructor(private x)` 要生成赋值)统统禁用。pi 自愿戴上这副镣铐,换来的是**整个项目可以没有(或几乎没有)编译步骤**:源码即运行物。

那运行时的类型校验怎么办?——这正是 TypeBox 补位的地方。`enum` 这类「编译期类型 + 运行时值」的特性被禁了,但工具参数、配置、协议消息又确实需要**运行时**校验。TypeBox 让你用普通的、可擦除的代码构造 schema(`Type.Object({...})`),既产出静态 TS 类型,又产出可在运行时跑的校验器——把「类型」和「校验」用一个不需要 JS emit 的库统一了。每个包都钉死 `typebox@1.1.38`(`AGENTS.md` 还规定外部直接依赖一律精确版本锁定)。

> **给 Agent 开发者的启示**:「少一层构建」听起来是洁癖,实则是**降低整个系统的认知与故障面**。有构建步骤,就有 sourcemap 对不上、dev 和 prod 行为不一致、构建缓存出诡异 bug 这一整类问题。pi 用「strip-only + TypeBox」把这一层直接抹掉:你 debug 的就是你写的代码,没有中间产物。这条纪律也解释了第 13 章为什么扩展用 `jiti` 运行时加载 `.ts`——整个 pi 从内核到扩展,都建立在「TS 源码可直接运行」这个统一前提上。一个一致的底层选择,会在很多看似无关的地方同时省事。

值得注意的是 pi 的四个包用了**两套测试运行器**:`ai`/`agent`/`coding-agent` 用 vitest,而 `tui` 用原生 `node --test test/*.test.ts`——TUI 这种纯渲染逻辑,连 vitest 都不需要。这种「按需选最轻的工具」的克制,和全书一以贯之的「核心精简」性格同源。

## 16.3 dogfooding:pi 用自己开发自己

pi 最强的回归测试不是任何一个 `.test.ts`——是 `AGENTS.md` 这个文件本身的存在。它是一份给 **Agent(也就是 pi 自己)** 看的开发守则:怎么提交、怎么跑测试(`./test.sh`,不能直接跑全量 vitest 因为含 e2e)、怎么改依赖、怎么写 commit message(`{feat,fix,docs}[(scope)]: ...`)。换句话说,**pi 的维护者用 pi 来开发 pi**。

这件事的工程意义远超「省事」。一个 Agent 每天被它的开发者用来读自己的代码、改自己的 bug、跑自己的测试——这是一种**最严苛、最高频的真实负载测试**。如果 read 工具有边界 bug、如果上下文压缩在长会话里丢信息、如果扩展钩子时序不对,开发者会第一个在自己的日常工作流里撞上,而不是等用户报 issue。`AGENTS.md` 里那些极其具体的条款(「commit 只 stage 你自己改的文件,别 `git add -A`,因为可能有多个 pi session 同时在这个目录里跑」)正是 dogfooding 撞出真实痛点后沉淀下来的规则。

> **给 Agent 开发者的启示**:dogfooding 对 Agent 项目不是「锦上添花」,而是**唯一能覆盖「真实长链路任务」的测试手段**。再多的 faux 编排也只能测你想到的场景;只有真的拿它干活,才会暴露你没想到的那些。如果你在做 Agent,最该做的第一件集成测试就是:**用它来开发它自己**。它跑不顺的地方,就是你的用户会跑不顺的地方。

## 16.4 收口:从 pi 借鉴的「最小 Harness 清单」

读到这里,我们可以把全书拆解过的所有子系统,反向收口成一份**搭建你自己 Agent 时的最小 Harness 检查清单**——每一项都对应前面某一章 pi 给出的答案:

| 能力 | 最小要求 | pi 的答案(章) |
|---|---|---|
| 模型抽象 | 一个与 Provider 无关的 `stream` 契约 | `streamSimple` / api-registry(6、7) |
| 主循环 | 生成 → 抽取工具 → 执行 → 回灌,能停 | 双层 `runLoop` / `stopReason`(3) |
| 可编程 | 在动手前后插钩子,机制与策略分离 | `beforeToolCall` / 九个钩子(4、5) |
| 工具系统 | 统一定义 + schema 校验 + 坏输入修复 | `ToolDefinition` / `prepareArguments`(8) |
| 执行后端 | 把 exec 抽成可替换接口 | `BashOperations`(9) |
| 准入控制 | 一个能拦截/改写工具调用的闸门 | `beforeToolCall` / `tool_call` 钩子(4、10、13) |
| 状态 | append-only、可分支、可导航 | JSONL 会话树(11) |
| 长程 | 超窗自动压缩 + 分支摘要 | compaction(12) |
| 扩展 | 不改核心就能装能力 | Extension / Skill(13、14) |
| 呈现 | 内核与 UI 解耦,多端复用 | pi-tui / print / rpc(15) |
| 可信 | 不联网的确定性测试 + dogfooding | faux provider / AGENTS.md(16) |

而 pi 真正想教给你的,其实是这份清单**之外**的一条:**「不做什么」和「做什么」同样是工程决策。** pi 刻意不内置权限系统(第 10 章),把约束外包给扩展与容器;刻意不用 React,把 UI 压成 `string[]`(第 15 章);刻意不要构建步骤,用 strip-only + TypeBox(本章)。每一个「不做」,都是为了让那个精简的核心**更稳、更可测、更容易被信任**。

这正是「把约束外包」这条工程性格的终点:一个把策略、UI、构建、甚至权限都推到边缘的内核,小到你可以完整地读懂它、确定性地测试它、放心地在它之上扩展——这就是 pi 留给每一个 Agent 开发者的范本。全书到此结束。

---

## 本章小结

- `faux` provider(`packages/ai/src/providers/faux.ts`)是一个完整实现 `StreamFunction` 契约的合成 Provider,不联网、不烧 token。用 `fauxText`/`fauxToolCall`/`fauxAssistantMessage` + `setResponses` 把模型每回合的输出**预先编排成脚本**,`state.callCount` 记录进度——让「调工具→回灌→收尾」的完整多轮链路被确定性地测,被测循环是真实代码,只有模型是假的。
- faux 是公开 API(`ai/src/index.ts` 导出),`AGENTS.md` 规定 `test/suite/` 必须用 faux,「无真实 API/key/付费 token」;回归测试放 `test/suite/regressions/`,以 issue 号命名。
- pi 自我约束只用「可擦除 TS 语法」(Node strip-only):禁 `enum`/`namespace`/参数属性等需要 JS emit 的特性,换来源码即运行物、无(几乎无)构建步骤。运行时校验由 TypeBox 补位(`typebox@1.1.38` 精确锁版),用可擦除代码同时产出静态类型与运行时校验器。这与第 13 章扩展用 jiti 运行时加载 `.ts` 同源。
- 四包两套测试运行器:ai/agent/coding-agent 用 vitest,tui 用原生 `node --test`——按需选最轻的工具。
- dogfooding:`AGENTS.md` 是给 pi 自己看的开发守则,维护者用 pi 开发 pi。这是覆盖「真实长链路任务」的唯一手段;那些极具体的规则(只 stage 自己改的文件,因为多 session 并行)正是 dogfooding 撞出的痛点。
- 收口:一份「最小 Harness 清单」把全书章节反向索引;pi 的元启示是「不做什么也是工程决策」——不内置权限、不用 React、不要构建步骤,每个「不做」都让精简核心更稳、更可测、更可信。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/ai/src/providers/faux.ts` | `faux` 合成 Provider:`fauxText`/`fauxToolCall`/`fauxAssistantMessage`、`FauxResponseFactory`、`setResponses`/`appendResponses`、`state.callCount` |
| `packages/ai/src/index.ts` | 把 faux 作为公开 API 导出 |
| `packages/ai/test/faux-provider.test.ts` | faux provider 自身的测试 |
| `packages/coding-agent/test/suite/harness.ts` | 用 faux 驱动整条 Agent 循环的测试骨架 |
| `AGENTS.md` | 给 Agent 看的开发守则:测试规约、strip-only 语法纪律、commit/git 规则、依赖安全 |
| `packages/*/package.json` | 测试运行器(vitest vs node --test)、`typebox@1.1.38` 精确锁版、strip-only 下的 `"type":"module"` |

## 动手实验

1. 读 `packages/coding-agent/test/suite/harness.ts` 和任意一个 `test/suite/*.test.ts`,找出它如何 `setResponses([...])` 编排一次「模型调工具 → 拿结果 → 收尾」的多轮对话。然后**故意**把第二步的 `fauxText` 改成另一个 `fauxToolCall`,跑测试,观察循环是否真的又执行了一次工具——以此确认「被测的是真实循环逻辑,只有模型被替换」。
2. 在 `packages/coding-agent/src` 里**故意**写一个 strip-only 禁用的语法(比如一个 `enum` 或 `constructor(private x: number)`),跑 `npm run check`,看它报什么错。再对照 `AGENTS.md` 第 20 条,把它改写成「显式字段 + 构造函数赋值」的等价形式。体会「无构建步骤」对你能写什么代码的硬约束。
3. 通读 `AGENTS.md` 全文,挑出三条**只可能从 dogfooding 中得出**的规则(提示:看 Git 一节里关于「多 session 并行、别 `git add -A`」的条款)。想一想:如果 pi 的维护者不用 pi 开发 pi,这些规则会被写出来吗?把这个问题套到你自己的 Agent 项目上。
