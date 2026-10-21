# 第 18 章　可信度工程：测试、evals、供应链与决策清单

> 一个 Agent 的内核越精简，越要有信心相信它「真的对」。pi 不靠堆集成测试来买信心，而是从五处下手：一个不联网、可编排的 `faux` provider，让整条循环可被确定性地测；一套 `test.sh` + `npm run check` 的本地门禁；一组把「文档有没有用」变成可比较数字的 evals；一条从锁文件到 CI 的供应链；以及「用自己开发自己」的 dogfooding。这是全书最后一章，也是把前十七章的工程决策收口成「可信系统」的一章。

我们已经把 pi 的每个子系统拆开看过了：模型契约、双层循环、AgentSession 编排、会话树、压缩、工具、扩展、MCP、TUI，以及 Durable/Chord/Env 这条基础设施前沿。但还有一个贯穿始终的问题没正面回答：**你怎么知道这一切是对的？**

Agent 系统最难测——它依赖一个不确定、要花钱、会限流的外部大模型；它的核心价值恰恰是「多轮工具调用」这种难以复现的长链路。这一章看 pi 怎么把「可信度」做成工程，而不是靠运气。

---

## 18.1 `faux` provider：不联网、不烧 token 地测整条循环

第 6、7 章讲过，pi 的模型层现在是一个显式的 `Models` 运行时：`Provider` 注册模型，`Models` 负责路由，`streamSimple` 是统一契约。这条接缝最大的红利，在测试时才完全兑现：**你可以注册一个假的 Provider，让整条 Agent 循环跑在它上面，完全不碰网络。**

`packages/ai/src/providers/faux.ts` 提供两种用法：

```ts
// 方式一：注册到全局 API registry（coding-agent 的测试骨架用这个）
import { registerFauxProvider, fauxAssistantMessage, fauxToolCall } from "@earendil-works/pi-ai/compat";

const faux = registerFauxProvider({ provider: "faux", api: "faux" });
faux.setResponses([
	fauxAssistantMessage([fauxToolCall("read", { path: "a.txt" })]),
	fauxAssistantMessage("读到了，内容是……"),
]);

// 方式二：拿到一个显式的 Provider handle，挂到自己的 Models 集合
import { fauxProvider } from "@earendil-works/pi-ai/providers/faux";

const handle = fauxProvider();
const models = createModels();
models.setProvider(handle.provider);
handle.setResponses([fauxAssistantMessage("hi")]);
```

faux 不是 mock 库里那种「打桩单个函数」，而是一个**完整实现了 `stream` / `streamSimple` 的合成 Provider**——对 `runLoop`、`AgentSession` 来说，它和真的 Anthropic provider 长得一模一样。关键在于它的响应是你**预先编排**好的脚本：

```ts
export type FauxResponseFactory = (
	context: TranscriptContext,
	options: SimpleStreamOptions | undefined,
	state: FauxProviderState,
	model: Model<string>,
) => AssistantMessage | Promise<AssistantMessage>;

export type FauxResponseStep = AssistantMessage | FauxResponseFactory;
```

配上一组工厂函数，你能用几行代码「写出」模型本回合该说什么：

```ts
fauxText("好的，我来读这个文件");
fauxThinking("先看看目录结构");
fauxToolCall("read", { path: "a.txt" });          // 让模型「决定」调 read 工具
fauxAssistantMessage([fauxText("内容是……")], { stopReason: "stop" });
```

`setResponses([...])` 把一串响应排进队列，Agent 每调一次模型就消费一步；`state.callCount` 记录到了第几步。于是一次「模型先调 read 工具 → 拿到结果 → 再总结」的完整多轮链路，可以被**逐字编排成一个确定性测试**：第一步返回一个 `fauxToolCall("read", ...)`，循环真的去执行 read 工具（真的读文件、真的把结果回灌），第二步返回 `fauxText(...)` 收尾。整个 `runLoop` 的工具抽取、执行、回灌、停止判定全程被真实跑过，唯独模型是假的。

faux 还支持几个让测试更真实的开关：

- `deferred`：模拟 provider 的延迟响应，`state.deferredFetchCount` 和 `cancelledDeferred` 让你断言轮询与取消；
- `tokensPerSecond` / `tokenSize`：控制流式输出的节奏与分块，用来测增量渲染、abort、超时；
- `FauxModelDefinition`：自定义模型 id、上下文窗口、图片输入、成本，用来测路由、压缩阈值与图片处理；
- `fauxAssistantMessage` 可以带 `errorMessage` 或 `stopReason: "error"/"aborted"`，把错误路径也变成脚本。

> **给 Agent 开发者的启示**：Agent 测试的死穴是「模型不确定 + 要花钱 + 慢」。很多团队的反应是「那就少测、靠手点」。pi 的解法是把不确定性**锁在一个清晰的接缝后面**——因为 Provider 早就是可插拔的（第 6 章），换上 faux 就把「不确定的模型」变成「我说它返回什么它就返回什么」的脚本，而被测的循环逻辑是**真实代码**。这就是「依赖倒置」在 Agent 工程里的最高回报：**把唯一不可控的依赖做成可注入的契约，剩下的一切都能确定性地测。**

`AGENTS.md` 把这条写成硬规矩：

> For `packages/coding-agent/test/suite/`, use `test/suite/harness.ts` + the faux provider. No real provider APIs, keys, or paid tokens.

回归测试还要在测试旁注明 GitHub issue 编号，方便从代码回溯到问题。faux 本身也是公开 API（`packages/ai/src/index.ts` 与 `/providers/faux` 导出），不是测试专用的私货。

## 18.2 测试分层：从 `node:test` 到 SSH 一致性套件

pi 有 11 个 workspace，测试策略不是「一套 vitest 跑所有」，而是按包的形态选最轻的工具，并给每类测试划清边界。

| 层级 | 工具/命令 | 说明 |
|---|---|---|
| 单元/集成（多数包） | `vitest --run` | `ai`、`agent`、`coding-agent`、`codemode`、`mcp`、`durable`、`chord`、`env`、`telemetry` |
| TUI 纯渲染 | `node --test test/*.test.ts` | `packages/tui`，连 vitest 都不需要 |
| durable 专项 | `vitest --run` ×2 + `test:workers` | 第二遍用 `PI_DURABLE_DOCUMENT_RETENTION_MS=0`，再跑 workers 配置 |
| 远端环境 | `conformance.test.ts` + `differential.test.ts` | 同一套语义分别跑 `RemoteExecutionEnv` 与 `NodeExecutionEnv`，并做随机操作序列对比 |
| MCP | `npm run test:mcp-conformance` | CI 单独一个 job |
| evals | `vitest run --config vitest.test.config.ts` / `eval:host` / `eval:docs` | 见 18.4 |
| 全量本地 | `./test.sh` | 隔离 HOME、清空 API key、禁用本地 LLM，再 `npm test` |

这里最容易被忽略的是**命令的边界**。`AGENTS.md` 明确规定：

- 改完代码（不是文档）跑 `npm run check`，并且要修掉所有 error/warning/info；
- **不要**直接跑全量 vitest 套件——它包含 e2e，只要环境里有 endpoint/auth 就会激活；
- 要跑全部非 e2e 测试，用根目录的 `./test.sh`；
- 单独跑某个测试，用包内路径，例如 `node "$(git rev-parse --show-toplevel)/node_modules/vitest/dist/cli.js" --run test/specific.test.ts`。

`test.sh` 本身也是一份「测试环境即代码」的样本：它用 `mktemp -d` 建一个带所有权标记的临时目录，把 `HOME`、`TMPDIR`、`XDG_*`、`NPM_CONFIG_*` 全部指进去，清掉 API key，禁用 AWS metadata 和本地 LLM，然后才 `npm test`。这样测试不会读到开发者本机的 `~/.pi` 配置、缓存或凭据，也不会因为环境里恰好有 key 就去打真实 provider。

CI 的分层也对应得很清楚（`.github/workflows/ci.yml`）：

```text
npm ci --ignore-scripts
npm run build
npm run check
npm run build:daemon --workspace packages/env
npm test
```

外加一个独立的 MCP client conformance job。`env.yml` 则在 Linux/macOS/Windows 的多架构矩阵上构建 daemon，跑 SSH 连接、conformance、differential 和外部 SSH 测试；`env-daemons.yml` 负责发布 daemon 二进制。**平台相关的代码，在对应平台上测**——这句话说起来简单，做起来就是一条 CI 矩阵。

> **给 Agent 开发者的启示**：测试分层不是为了「显得工程化」，而是为了让每条命令都有明确的适用边界。全量测试如果会偷偷打真实 API，开发者就会不敢跑；TUI 如果硬套 vitest，就多一层不必要的依赖；远端语义如果只在本机 mock，SSH/路径/进程组的差异永远测不出来。先划清「哪一层测什么、用什么跑、需要什么环境」，再谈覆盖率。

## 18.3 类型纪律：`erasableSyntaxOnly` + TypeBox

第 8 章讲工具 schema 时提过 TypeBox。这里要点出它背后一条更深的工程取舍：pi 刻意约束自己**只用「可擦除的 TypeScript 语法」**。

这条约束不是靠自觉，而是写进 `tsconfig.base.json`：

```json
{
	"compilerOptions": {
		"erasableSyntaxOnly": true,
		"rewriteRelativeImportExtensions": true
	}
}
```

`erasableSyntaxOnly` 会拒绝任何「需要生成额外 JS 代码」的 TS 特性——`enum`（要生成对象）、`namespace`（要生成 IIFE）、参数属性（`constructor(private x)` 要生成赋值）、`import =`/`export =` 等等。`rewriteRelativeImportExtensions` 则让源码里的 `./foo.ts` 在编译输出里被改写成 `./foo.js`。

这带来的直接好处是：**源码可以被 Node 直接执行**。pi 的开发与测试大量使用 `node --experimental-strip-types` / Node 22.19+ 的类型擦除能力，源码即运行物；`npm run build` 仍然存在，用来产出发布到 npm 的 `dist/`，但它不再是「跑起来」的前置条件。你 debug 的就是你写的代码，没有 sourcemap 对不上、dev 和 prod 行为不一致这一整类问题。

那运行时的类型校验怎么办？——这正是 TypeBox 补位的地方。`enum` 这类「编译期类型 + 运行时值」的特性被禁了，但工具参数、配置、协议消息又确实需要**运行时**校验。TypeBox 让你用普通的、可擦除的代码构造 schema（`Type.Object({...})`），既产出静态 TS 类型，又产出可在运行时跑的校验器。

版本纪律也很硬：`ai`、`agent`、`coding-agent`、`durable`、`chord` 都钉死 `typebox@1.3.27`（`AGENTS.md` 规定外部直接依赖一律精确版本锁定，`.npmrc` 里还有 `save-exact=true`）。这不是「保守」，而是把依赖升级当成一次需要 review 的代码变更。

> **给 Agent 开发者的启示**：「少一层构建」听起来是洁癖，实则是**降低整个系统的认知与故障面**。有构建步骤，就有 sourcemap、缓存、产物与源码不一致的一整类问题。pi 用「strip-only + TypeBox」把这一层压到最薄：开发时直接跑源码，发布时再编译；运行时校验用同一份 schema，不另写一套。这条纪律也解释了第 13 章为什么扩展用 jiti 运行时加载 `.ts`——整个 pi 从内核到扩展，都建立在「TS 源码可直接运行」这个统一前提上。一个一致的底层选择，会在很多看似无关的地方同时省事。

## 18.4 evals：把「文档有没有用」变成可比较的数字

传统测试回答「代码是否按预期工作」；evals 回答「Agent 在真实任务上是否表现更好」。pi 的 `packages/evals` 是这一层的独立 workspace（private，`@earendil-works/pi-evals`），基于 `vitest-evals`。

它有两类 eval：

- `*.docs.eval.ts`：**文档增益评估**。同一个任务分别在 `without_docs` 和 `with_docs` 两个隔离容器里跑，比较通过率。`without_docs` 会移除 coding-agent 的 `README.md`、`CHANGELOG.md`、`docs/`、`examples/`，并从默认系统提示词里删掉文档路由段；`with_docs` 保留。两者安装同一批本地 workspace tarball，且内部依赖包的文档与源码被对称移除，防止它们变成「另一份说明书」。
- 其他 `*.eval.ts`：host evals，在本机用 Vitest 跑，是普通的 vitest-evals 套件，不做配对比较（smoke、documentation audit 等）。

文档评估的执行链路很值得看：

1. 把仓库临时挂载进 Docker 构建，用仓库自己的 consumer-install 机制打包当前 workspace；
2. 构建 `without_docs` / `with_docs` 两个镜像；
3. 在两个镜像里发现 case，并要求 cohort 完全一致；
4. 先规划所有 `(case, variant, model, runNumber)` arm，再逐个在**全新容器**里跑；
5. 读取 Vitest 原生 JSON，配对精确的 arm，计算 lift；
6. 任何一对被阻塞（缺失、重复、跳过、pending、未评分、报错），就不发布 headline lift，进程以非零退出。

隔离措施也很具体：每个 arm 使用新的 home、agent 目录、workspace、session 目录和容器文件系统；eval 定义、评估器、fixture 与 Vitest 配置归 root 所有，harness 永久降到非特权 UID 后不可读；文档 eval 默认只允许 `read`、`write`、`edit`、`grep`、`find`、`ls`，不暴露 shell 和 web search。

这套东西的定位很清醒：**eval 分数是数据，不是断言**。`judgeThreshold: null` 用于比较评分；低分是观察结果，不是基础设施故障；一次重复不能证明稳定性，要显式增加 `--runs-per-variant`。报告还会标记「没有提升」「负增益」「对照组/实验组饱和」「观察到的抖动」。

> **给 Agent 开发者的启示**：当你改的是提示词、文档、工具描述或模型路由时，单元测试帮不上忙——它们只能证明「代码没坏」，不能证明「Agent 变得更会做事」。evals 补的就是这一层：用成对、隔离、可重复的实验，把「感觉变好了」变成「在 N 个 case 上通过率提升了多少，且没有把别的 case 弄坏」。代价是要花钱、要跑容器、要接受统计噪声；所以它适合放在发布前或定期任务里，而不是每次提交都跑。

## 18.5 供应链：把「依赖变更」当成代码变更

Agent 项目通常依赖大量 npm 包，供应链是绕不开的可信度问题。pi 的做法可以概括成一句话：**把依赖、锁文件、安装脚本都当成需要 review 的代码。**

`.npmrc` 只有两行，但很关键：

```ini
save-exact=true
min-release-age=2
```

- `save-exact=true`：新装的直接依赖写精确版本，不写 `^`；
- `min-release-age=2`：只安装发布超过 2 天的版本，降低「刚发布的恶意包/坏版本」风险。

`AGENTS.md` 的 Dependency and Install Security 一节进一步规定：

- 直接依赖（包括内部 workspace 依赖）精确锁版；
- 更新 `undici` 必须先读目标版本的 changelog/release notes，评估是否影响功能；
- 本地安装用 `npm install --ignore-scripts`，CI 用 `npm ci --ignore-scripts`；不跑 lifecycle scripts，除非用户明确要求；
- 依赖元数据变化后，用 `npm install --package-lock-only --ignore-scripts` 刷新 lockfile；
- 新增带 lifecycle script 的依赖，必须显式 review 并加入 `scripts/generate-coding-agent-install-lock.mjs` 的 allowlist，不能静默添加。

pre-commit 还会跑 `scripts/check-lockfile-commit.mjs`：只要 staged 的 `package-lock.json` 里有非 workspace 的包变化，就阻止提交，除非设置 `PI_ALLOW_LOCKFILE_CHANGE=1`。这不是为了刁难，而是因为「锁文件悄悄变了」是供应链事故最常见的入口之一。

`npm run check` 本身也是一条完整的门禁链：

```text
biome-check --write
→ check:pinned-deps        （直接依赖必须精确锁版）
→ check:runtime-deps       （运行时依赖不能误放到 devDependencies）
→ check:ts-imports         （相对导入必须带扩展名）
→ check:entry-graphs       （包的 entry 图不能断）
→ check:install-lock       （coding-agent 安装锁与 package.json 一致）
→ tsc --noEmit             （全仓类型检查）
→ check:browser-smoke      （浏览器侧 smoke）
```

注意 `tsc --noEmit` 只是其中一环，不是全部。很多项目把「类型过了」当成「检查过了」，pi 则把格式、依赖、导入、entry 图、安装锁、浏览器 smoke 都放进了同一条命令。CI 再在多个 Node 版本/平台上跑这条命令，并额外构建 daemon、跑 MCP conformance 与 evals。

> **给 Agent 开发者的启示**：供应链安全不是「加一个 npm audit」就结束了。真正有效的做法是把风险前移到**提交时**：精确锁版、限制安装脚本、锁文件变更需要显式确认、依赖升级要求读 changelog。再配合 CI 的 `npm ci --ignore-scripts` 和多平台测试，才能把「某天早上依赖被投毒」变成一件需要多道门同时失守才会发生的事。

## 18.6 可观测性与 dogfooding：让系统自己说话

### 18.6.1 telemetry：显式上下文，不绑架后端

`packages/telemetry`（`@earendil-works/pi-telemetry`）是一条容易被忽略、但对「可信系统」很重要的线。它提供的不是 exporter，而是一组**厂商中立的契约**：

- `TelemetryContext` / `TelemetrySpan` 用回调式 API 管理 span 的生命周期：`startSpan(options, fn)`，没有公开的 `end()`，span 在回调返回/ reject 时自动结束；
- 没有 `AsyncLocalStorage`，也不依赖任何运行时特有的「当前 span」；父上下文通过参数显式传递；
- `NOOP_TELEMETRY_CONTEXT` 让 telemetry 完全可选；
- `InMemoryTelemetryContext` 是参考实现，适配器可以实现 OpenTelemetry、Sentry、日志等后端；
- 支持 propagation carrier、span links、counter/up-down counter/histogram，以及**类型化 schema**：`defineTelemetrySchema()` + `createTypedMetrics()` 在编译期检查指标名、类型、属性；
- 属性值只允许原始标量与数组，明确要求低基数；不要记录 prompt、补全、工具参数/输出、文件内容、provider payload、header、凭据或自由格式错误详情。

pi 的 AI 与 Durable 各自导出自己的 schema（`AI_TELEMETRY_SCHEMA`、`DURABLE_TELEMETRY_SCHEMA`），用 `pi.ai.*` / `pi.durable.*` 命名；适配器可以翻译成后端约定，但不改 pi 的词汇。Durable 的 README 还特别说明：属性里只有 ID 和控制元数据，绝不包含消息、参数或结果——ID 指向 transcript，需要细节时去读已提交的记录。

> **给 Agent 开发者的启示**：可观测性最容易犯的错，是把「业务状态」和「诊断数据」混在一起。telemetry 不是 durable state：不要把 `TelemetryContext`、span 或后端 trace 对象持久化进消息、快照或 deferred handle；需要跨进程关联时，只传一个不透明的 carrier。另一个好习惯是**显式传递上下文**——它比隐式的「当前 span」啰嗦，但在多 Agent、多工具、并发子任务里不会串线。

### 18.6.2 dogfooding：pi 用自己开发自己

pi 最强的回归测试不是任何一个 `.test.ts`——是 `AGENTS.md` 这个文件本身的存在。它是一份给 **Agent（也就是 pi 自己）** 看的开发守则：怎么提交、怎么跑测试、怎么改依赖、怎么写 commit message（`{feat,fix,docs}[(ai,tui,agent,coding-agent)]: ...`）。换句话说，**pi 的维护者用 pi 来开发 pi**。

这件事的工程意义远超「省事」。一个 Agent 每天被它的开发者用来读自己的代码、改自己的 bug、跑自己的测试——这是一种**最严苛、最高频的真实负载测试**。如果 read 工具有边界 bug、如果上下文压缩在长会话里丢信息、如果扩展钩子时序不对，开发者会第一个在自己的日常工作流里撞上，而不是等用户报 issue。

`AGENTS.md` 里那些极其具体的条款，正是 dogfooding 撞出来的：

- 「Multiple pi sessions may be running in this cwd at the same time... Only commit files YOU changed in THIS session.」——因为多个 pi session 并行开发同一个仓库，是他们的日常；
- 「Never run `git reset --hard`, `git checkout .`, `git clean -fd`, `git stash`, `git add -A`...」——因为这些操作会毁掉另一个 session 的工作；
- 「For testing pi's interactive mode, load and follow `.pi/skills/interactive-testing.md`」——把「怎么用 tmux 驱动 TUI 做真实交互测试」写成 Agent 可执行的 Skill；
- 「Never run `npm run build` or `npm test` unless requested」——因为全量构建/测试会打断正在并行工作的其他 Agent；
- release、GitHub 工作流、添加 provider 也各自有 `.pi/skills/*.md`。

> **给 Agent 开发者的启示**：dogfooding 对 Agent 项目不是「锦上添花」，而是**唯一能覆盖「真实长链路任务」的测试手段**。再多的 faux 编排也只能测你想到的场景；只有真的拿它干活，才会暴露你没想到的那些。如果你在做 Agent，最该做的第一件集成测试就是：**用它来开发它自己**。它跑不顺的地方，就是你的用户会跑不顺的地方。但也要清醒：dogfooding 产生的是「症状」，不是「确定性证据」——把撞出来的问题固化成 faux 测试或 eval case，才算真正闭环。

## 18.7 收口：一份「最小 Harness 清单」

读到这里，我们可以把全书拆解过的所有子系统，反向收口成一份**搭建你自己 Agent 时的最小 Harness 检查清单**——每一项都对应前面某一章 pi 给出的答案：

| 能力 | 最小要求 | pi 的答案（章） |
|---|---|---|
| 模型抽象 | 一个与 Provider 无关的 `stream` 契约 | `Provider` / `Models` / `streamSimple`（6、7） |
| 主循环 | 生成 → 抽取工具 → 执行 → 回灌，能停 | 双层 `runLoop` / `stopReason`（3） |
| 可编程 | 在动手前后插钩子，机制与策略分离 | `AgentLoopConfig` 十个钩子（4、5） |
| 工具系统 | 统一定义 + schema 校验 + 坏输入修复 | `ToolDefinition` / `prepareArguments`（8） |
| 执行后端 | 把 exec 抽成可替换接口 | `BashOperations` / `ExecutionEnv`（9、17） |
| 准入控制 | 一个能拦截/改写工具调用的闸门 | `beforeToolCall` / `tool_call`（4、10、13） |
| 状态 | append-only、可分支、可导航 | JSONL 会话树 v3（11） |
| 长程 | 超窗自动压缩 + 分支摘要 | compaction（12） |
| 扩展 | 不改核心就能装能力 | Extension / Skill / Pi Packages（13、14） |
| 工具聚合 | 大工具集按需暴露、组合调用 | MCP / codemode / tool search（15） |
| 呈现 | 内核与 UI 解耦，多端复用 | `pi-tui` / print / JSON / RPC / SDK（16） |
| 持久与远程 | 崩溃可恢复、工具可异地执行 | Durable / Chord / Env（17） |
| 可信 | 确定性测试 + 行为评估 + 门禁 + dogfooding | faux / evals / `npm run check` / `AGENTS.md`（18） |

而 pi 真正想教给你的，其实是这份清单**之外**的一条：**「不做什么」和「做什么」同样是工程决策。**

- 刻意不内置权限系统（第 10 章），把约束外包给扩展与容器；
- 刻意不用 React，把 UI 压成 `string[]`（第 16 章）；
- 刻意不要全局 API registry，把模型集合变成显式的 `Models` 运行时（第 6 章）；
- 刻意不用 `AsyncLocalStorage`，把 telemetry 上下文显式传递（第 18 章）；
- 刻意不在事务里执行外部效果，把「先提交、再展示」做成 Durable 的不变量（第 17 章）；
- 刻意不把 dogfooding 当成测试的全部，而是把撞出来的问题固化成 faux 测试与 eval case。

每一个「不做」，都是为了让那个精简的核心**更稳、更可测、更容易被信任**。这也是「把约束外包」这条工程性格的终点：一个把策略、UI、构建、权限、模型路由都推到边缘的内核，小到你可以完整地读懂它、确定性地测试它、放心地在它之上扩展——这就是 pi 留给每一个 Agent 开发者的范本。

全书到此结束。

---

## 本章小结

- `faux` provider 是一个完整实现 `stream`/`streamSimple` 的合成 Provider，不联网、不烧 token。用 `fauxText`/`fauxThinking`/`fauxToolCall`/`fauxAssistantMessage` + `setResponses`/`appendResponses` 把模型每回合的输出**预先编排成脚本**，`state.callCount` 记录进度；支持 deferred、流式节奏、错误/abort 与自定义模型定义。`coding-agent/test/suite/` 强制使用 `harness.ts` + faux，禁止真实 API/key/付费 token。
- 测试按包分层：多数包用 vitest，`tui` 用 `node --test`，`durable` 跑双遍 + workers，`env` 有 conformance/differential/SSH 测试，MCP 有独立 conformance job。本地全量用 `./test.sh`（隔离 HOME、清 key），**不要**直接跑全量 vitest；改代码后跑 `npm run check`。
- `erasableSyntaxOnly` + `rewriteRelativeImportExtensions` 写进 `tsconfig.base.json`，禁用需要 JS emit 的 TS 特性，让源码可被 Node 直接执行；TypeBox 1.3.27 精确锁版，用可擦除代码同时产出静态类型与运行时校验器。发布仍有 `tsc` 构建，但开发/测试不依赖它。
- `packages/evals` 用成对、隔离、可重复的容器实验衡量「文档增益」与行为表现；任何一对被阻塞就不发布 headline lift；eval 分数是数据，不是断言，一次重复不能证明稳定性。
- 供应链：`save-exact=true`、`min-release-age=2`、`npm ci --ignore-scripts`、生命周期脚本 allowlist、锁文件提交门禁（`PI_ALLOW_LOCKFILE_CHANGE=1` 才能放行）、`undici` 升级读 changelog；`npm run check` 串起 Biome、pinned deps、runtime deps、TS imports、entry graphs、install lock、`tsc --noEmit`、browser smoke。
- `telemetry` 是厂商中立的显式上下文契约：无 exporter、无 `AsyncLocalStorage`、`NOOP` 可选、类型化 schema、低基数属性、绝不记录 prompt/工具内容/凭据；AI 与 Durable 各自导出领域 schema。
- dogfooding：`AGENTS.md` 是给 pi 自己看的开发守则，`.pi/skills/` 把交互测试、发布、GitHub 工作流写成 Agent 可执行的 Skill。它是覆盖真实长链路的唯一手段，但撞出的问题要固化成 faux 测试或 eval case 才算闭环。
- 收口：一份「最小 Harness 清单」把全书章节反向索引；pi 的元启示是「不做什么也是工程决策」——不内置权限、不用 React、不要全局 registry、不用隐式 telemetry 上下文、不在事务里做外部效果。每个「不做」都让核心更稳、更可测、更可信。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/ai/src/providers/faux.ts` | faux 合成 Provider：`fauxText`/`fauxThinking`/`fauxToolCall`/`fauxAssistantMessage`、`FauxResponseFactory`、`setResponses`/`appendResponses`、deferred、`fauxProvider()` |
| `packages/ai/src/compat.ts` | `registerFauxProvider()` 与 `streamSimple` 等兼容入口 |
| `packages/coding-agent/test/suite/harness.ts` | 用 faux 驱动整条 `AgentSession` 的测试骨架 |
| `packages/coding-agent/test/suite/*.test.ts` | 会话、压缩、工具、MCP、retry 等确定性回归测试 |
| `packages/evals/README.md` / `src/cli.ts` / `src/docker.ts` / `src/plan.ts` / `src/report.ts` | 文档增益评估：双镜像、arm 规划、配对、隔离与报告 |
| `packages/telemetry/README.md` / `src/index.ts` | 显式 telemetry 契约、NOOP、内存参考实现、类型化 schema、propagation |
| `tsconfig.base.json` | `erasableSyntaxOnly`、`rewriteRelativeImportExtensions` |
| `AGENTS.md` | 给 Agent 看的开发守则：检查/测试命令、strip-only、依赖安全、Git 规则 |
| `.npmrc` | `save-exact=true`、`min-release-age=2` |
| `test.sh` | 隔离环境下的全量非 e2e 测试入口 |
| `scripts/check-lockfile-commit.mjs` | 锁文件变更提交门禁 |
| `scripts/generate-coding-agent-install-lock.mjs` | coding-agent 安装锁；生命周期脚本 allowlist |
| `.github/workflows/ci.yml` / `env.yml` / `env-daemons.yml` | 构建、检查、测试、daemon 多平台矩阵与 MCP conformance |
| `packages/*/package.json` | 各包测试运行器、TypeBox 精确版本、脚本定义 |

## 动手实验

1. 读 `packages/coding-agent/test/suite/harness.ts` 和任意一个 `test/suite/*.test.ts`，找出它如何用 `registerFauxProvider()` + `setResponses([...])` 编排一次「模型调工具 → 拿结果 → 收尾」的多轮对话。然后**故意**把第二步的 `fauxAssistantMessage` 改成另一个 `fauxToolCall`，跑测试，观察循环是否真的又执行了一次工具——以此确认「被测的是真实循环逻辑，只有模型被替换」。
2. 在 `packages/coding-agent/src` 里**故意**写一个 strip-only 禁用的语法（比如一个 `enum` 或 `constructor(private x: number)`），跑 `npm run check`，看 `tsc --noEmit` 报什么错。再对照 `AGENTS.md` 的 Code Quality 一节，把它改写成「显式字段 + 构造函数赋值」的等价形式。体会「源码可直接运行」对你能写什么代码的硬约束。
3. 读 `packages/evals/README.md` 的「The runner」五步，然后构造一个最小的 docs eval：同一个任务，分别在 `without_docs` 与 `with_docs` 下跑一次。观察 `report.json` 里 pair 是怎么配对的、什么情况下 headline lift 会被 withhold。再想一想：为什么它用「成对比较」而不是「两组平均值」？
4. 通读 `AGENTS.md` 全文，挑出三条**只可能从 dogfooding 中得出**的规则（提示：看 Git 一节里关于「多 session 并行、别 `git add -A`」的条款）。再读 `.pi/skills/interactive-testing.md`，看它如何把「用 tmux 驱动 TUI」写成 Agent 可执行的步骤。把这个问题套到你自己的 Agent 项目上：哪些规则是你「用了才知道」的？
5. 用 `NOOP_TELEMETRY_CONTEXT` 与 `InMemoryTelemetryContext` 各跑一段带嵌套 span 的代码，比较两者的开销与可观察结果；再给一个 span 加上 `setAttributes`，确认属性里**没有** prompt、工具参数或凭据。最后读一遍 `packages/telemetry/README.md` 的 Security and Portability，列出三条你会写进自己项目规范的红线。
