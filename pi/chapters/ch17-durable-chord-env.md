# 第 17 章　Durable、Chord、Env：另一条前沿

> `coding-agent` 是「一个能用的 Agent CLI」，而 pi 仓库里还有三个不那么显眼的包：`durable`、`chord`、`env`。它们不服务终端里的对话，而在回答三个更底层的问题——**进程崩了，Agent 的活儿怎么接着干？多进程插件和状态怎么热更新？工具怎么跑在另一台机器上？** 这一章看 pi 如何把 Harness 从「一个进程里的循环」推进到「可恢复、可复制、可远程执行」的基础设施。

前面十六章的主线是 `coding-agent`：`AgentSession` 在内存里跑循环，会话树写在本地 JSONL，工具在本机执行。这条路线适合「人在终端前用 Agent」，但它默认了三件事：进程活着、状态只在一个地方、工具和凭据在同一台机器上。

`durable`、`chord`、`env` 把这三个默认逐个打破。它们都是 **Experimental**：API 会随版本变化，不是 `coding-agent` 的内部依赖，也不是「pi CLI 的隐藏功能」。但正因为它们把问题推到了边界，读它们能看清 pi 对「Agent 基础设施」的完整想象。三个包的分工是：

| 包 | 一句话 | 关键问题 |
|---|---|---|
| `@earendil-works/pi-durable` | 持久化 Agent Harness | 崩溃/重启后，对话、任务、工具调用如何从最后一个检查点继续 |
| `@earendil-works/chord` | 类型化多进程运行时 | 对象、契约、复制状态、RPC、热重载如何在 server/worker/browser 之间工作 |
| `@earendil-works/pi-env` | 远程执行环境 | 工具如何通过 SSH 在另一台机器上执行，而语义与本地一致 |

---

## 17.1 Durable：先提交，再展示

Durable 的 README 第一句话就是它的设计信条：

> Conversations, model turns, tool calls, and your own state are committed to storage before anything is shown. If the process dies mid-turn, reopening the storage picks the work up where it stopped.

这句话包含两个方向：**写路径先落盘再可见**，**读路径从已提交状态恢复**。它和 `coding-agent` 的 JSONL 会话树是同一个思路的彻底版：后者把「对话」做成 append-only 日志，Durable 把「对话 + 任务 + 文档」一起做成一个原子提交模型。

### 17.1.1 五个核心概念

```ts
const harness = await Harness.open(new MemoryStorage(), { models, registry }, context);
const root = await harness.root(context, { agent: { model: { provider: "openai", modelId: "gpt-6-sol" } } });

const submission = await root.submit({ type: "input", content: "What is the capital of France?" }, context);
const settled = await submission.wait(context);
if (settled.status === "done" && settled.type === "input") {
	const answer = await root.commit((tx) => tx.entry(AssistantEntry, settled.answer), context);
}
```

- **Harness**：一个打开的存储 + 在其上运行 Agent 的机器。所有修改走同一条原子提交线；没有「先展示、后落盘」的路径。
- **Conversation**：一份 transcript，由不可变 **Entry** 组成（`pi.user`、`pi.assistant`、`pi.tool-result`、`pi.system`、`pi.reset`，以及你自己的 kind）。模型只看到最近一次 reset 之后的条目。
- **Commit**：一次原子写。`conversation.commit((tx) => ...)` 可以在同一个事务里追加条目、修改文档、创建任务——要么全落盘，要么全不落。
- **Document**：和 transcript 放在一起的类型化 JSON 状态，随 commit 修改。内置文档包括 `pi.agent`（模型/扩展/工具/cwd）、`pi.provider`（provider 会话身份）、`pi.live`（正在跑的生成与工具）、`pi.inbox`（排队输入）、`pi.usage`（花费）。
- **Task**：一个持久化状态机，每一步都保存检查点；进程重启后从最后一个检查点继续。答案由内置任务 `pi.generation` 产生，它拥有每次工具调用的 `pi.tool` 任务，等它们结束后把控制权交给下一轮生成。

一次「输入 → 回答」在数据上的形状是：

```text
submit(input) → pi.user
  pi.generation → pi.system (仅当 prompt 或工具变化), pi.assistant (tool calls)
    pi.tool × n → pi.tool-result × n   (由 generation 拥有，generation 等它们)
  pi.generation → pi.assistant (answer) → submission done
```

注意这里没有「内存里的消息数组」。transcript 是存储里的不可变条目，生成任务每次从已提交状态重建模型上下文。**UI 看到的所有东西也都是已提交状态**：`viewState()` 返回一个只读的 Chord state，每次 commit 后更新；`watch()` 则给出每次 commit 的精确 Chord 操作，适合远程客户端增量应用。

### 17.1.2 任务：检查点、效果三明治与睡眠

Durable 的任务用「阶段（phase）」描述：

```ts
const Reminder = defineTask<{ text: string; at: number }, { phase: "wait" } | { phase: "deliver" }, null>({
	name: "app.reminder",
	version: 1,
	initial: () => ({ phase: "wait" }),
	phases: {
		wait: (task, runtime, context) => runtime.sleepUntil(task.input.at, { phase: "deliver" }, context),
		deliver: async (task, runtime, context) => {
			await notify(task.input.text);
			await runtime.commit(() => ({ status: "terminal", outcome: { status: "completed", result: null } }), context);
		},
	},
	abort: (_task, runtime, context) =>
		runtime.commit(() => ({ status: "terminal", outcome: { status: "aborted" } }), context),
});
```

任务状态是 `pending | running | waiting | sleeping | completing | terminal`。几个关键规则：

- **阶段会从头重跑**。进程在 `deliver` 中间崩溃，重启后从 `deliver` 的开头再执行一次。所以阶段里的外部效果必须幂等，跨阶段需要保留的东西必须放进 checkpoint。
- **睡眠是持久的**。`sleepUntil()` 把 `{ status: "sleeping", until, checkpoint }` 提交进存储，任务释放 invocation，不再占内存；时间到了由调度器恢复。重启不会丢闹钟。
- **效果三明治**。Durable 推荐的写法是「意图 → 效果 → 结果」：先前进一步 checkpoint 记下要做什么和幂等键，再执行外部效果，最后把结果和下一状态一起提交。外部效果永远不在 Session 的事务里跑（不变量 4），事务只负责记录。
- **崩溃循环会被停车**。连续 `restartLimit`（默认 5）次在「没有新状态提交」的情况下被崩溃/驱逐切断，任务会被 park，状态为 blocked、原因 `restart_limit`，`wake()` 不再把它算作活跃；需要 `harness.retryTask(id)` 手动放行。这个设计很务实：确定性崩溃的任务如果无限重启，会把整个进程拖进崩溃循环。
- **结构化并发**。任务可以拥有子任务，用 `waiting` 状态等它们，`failFast`/`allSettled` 决定失败传播；父任务先结束而子任务还在跑时进入 `completing`；中止是自底向上的。

### 17.1.3 恢复、幂等与「重放」

Durable 的恢复不是「把内存快照读回来」，而是「重新调度未完成的任务」。这带来两个必须正视的问题：

1. **提交重复**：客户端重试同一个输入怎么办？`submit({ requestId })` 用 requestId 去重，重启后同样的 requestId 会找回同一个 Submission，而不是再提交一次。
2. **效果重复**：任务重跑时，已经发生的工具调用/支付/发消息怎么办？工具可以声明 `replay` 策略；嵌套工具调用（`api.executeTool()`）有稳定的 key，重放时按 key 找回已完成的结果；Code Mode 把脚本拿到的每个工具结果先写 journal，重启后按顺序喂回，脚本因此走同一条路径，不会重复产生副作用。

这也解释了为什么 Durable 强调 **「tool calls are tasks」**：只有把一次工具调用提升为有 ID、有检查点、有重放策略的持久任务，崩溃恢复才有地方落脚。普通的函数调用在进程死掉的那一刻就什么都不剩了。

### 17.1.4 存储与宿主

Durable 的存储后端有四类：

| 后端 | 特点 |
|---|---|
| `MemoryStorage` | 不持久化，测试/演示 |
| SQLite（`/storage/sqlite/node`） | 单文件；WAL + `synchronous = NORMAL`，进程崩溃可恢复，断电可能丢最新一次提交 |
| JSONL（`/storage/jsonl/node`） | 追加文件，可 `fsync`；整段历史也在内存里，适合短会话 |
| Cloudflare Durable Object（`/storage/sqlite/cloudflare`） | 用 DO 的 SQLite storage，commit 就是它的存储事务 |

存储契约有共享的 conformance suite（`registerStorageConformance`），自定义后端可以自己跑。一个进程同一时间只能拥有一个 storage，没有跨进程锁——这是它和「多进程 Chord」之间的一条清晰边界。

宿主侧还有两个值得抄的细节：

- **`onWake`**：在 Cloudflare Durable Object 这类会被驱逐的宿主上，Harness 不自己挂长定时器，而是把 `{ active, at }` 告诉宿主；宿主用 alarm 在最早睡眠点唤醒，并调用 `resume()`。这让「Agent 休眠一个月再醒来」不需要一个常驻进程。
- **`viewState()` / `watch()`**：UI 不需要订阅内部事件，直接读已提交状态的快照和增量。慢消费者最多保留 100 个未交付帧，超过后用一份最新全量视图替换——这是第 16 章「差分渲染」在分布式侧的对应物。

> **给 Agent 开发者的启示**：Durable 把 Agent 的「记忆」从进程内存搬到了提交日志，于是「崩溃恢复」不再是特判，而是调度器的默认行为。代价是每个外部效果都要回答两个问题：**它幂等吗？它的结果被记录了吗？** 如果你正在做长时任务、后台 Agent、或会被平台驱逐的 Serverless Agent，这两问比任何框架选型都重要。

---

## 17.2 Chord：对象、契约与热重载

Durable 的文档状态、视图和任务图都建立在 Chord 上。Chord 自己的定位是「用对象搭应用」：

> Chord builds applications out of objects that live in several processes (a server, workers, browser tabs) and can be hot reloaded while they run.

它的五个关键词是：

- **object**：地址 + 代码 + 状态。按需创建，被引用时存活。
- **reference**：调用方拿到的是引用，永远不是对象本身；本地和跨进程的引用长得一样，且能跨热重载存活。
- **principal**：每个对象知道自己的主体；所有者和访客拿到不同的对象、不同的方法。
- **state**：可观察的 JSON。远程观察者先拿快照，再拿增量。
- **hot reload**：运行中换代码。在飞的工作用旧代码跑完，状态保留，不兼容的变化在安全点迁移。

### 17.2.1 契约是运行时的值

Chord 的契约不是 TypeScript 类型，而是运行时值——因为发布的 bundle 里没有类型，第三方插件也是独立构建的：

```ts
export const Counter = remote("demo.counter", {
	version: 1,
	input: Type.Object({ name: Type.String() }),
	host: hostNamed("server"),
	variants: {
		default: {
			state: { count: Type.Number() },
			methods: { increment: method(Type.Object({ by: Type.Number() })) },
		},
	},
});
```

几个设计点：

- **`remote` vs `local`**：remote 类型可被其他 host 寻址，输入/状态/方法参数都要 schema；local 类型只在本 host，成员可以是函数、原生对象，但契约必须列全（`method`/`opaque`/状态 schema），这样引用在还没 live 时也知道 `h.x` 是方法还是状态。
- **输入即身份**：对象身份是 `(type, canonicalInput, principalKey)`。输入先应用默认值、去掉未知字段、校验，再转成 canonical JSON；等于默认值的字段被省略。改默认值或规范化规则 = 改身份，必须换类型名。
- **placement**：remote 类型必须声明 `host: hostNamed(name)` 或 `hostFamily(prefix, key)`——数据在契约里，调用方永远不自己指定 host。
- **版本与兼容**：同版本时，provider 的描述必须覆盖 consumer 的；否则只能通过 provider 声明的 `accepts[M]` 适配器；再否则 `contract_mismatch`。加字段不用升版本，删/改/新必填要升版本。版本没升但描述不覆盖，会被拒绝——这是一条能抓「忘了 bump」的硬检查。

### 17.2.2 对象的一生

```ts
host.serve(StatusStore, ({ session }, _host, self) => {
	const status = self.state("status", StatusSchema, { running: false, tasks: 0 });
	const catalog = self.get(Catalog, {});
	const watch = self.resource("watch", {
		version: 1,
		open: async ({ signal }) => openWatch(session, signal),
		close: async (w) => w.close(),
	});
	self.on("changes", watch, (w) => w.events, (event, ctx) => status.change(ctx, (d) => { d.tasks = event.tasks; }));
	self.task(async (ctx) => { /* loop until ctx.signal aborts */ });
	return { status, report: (args, ctx) => { status.replace(ctx, args); } };
});
```

工厂是**同步**的，对象的激活是**异步**的：

1. 同步执行 factory，收集声明（state/resource/get/on/task/derive/register）；
2. await 声明的 resource `open()`；
3. await 声明的引用变 live（依赖先激活）；
4. 并行执行带 `init` 的 cell；
5. 挂上 derive、listener、task；
6. 对象 live，引用变为 `live`、`ready` resolve。

这个「同步声明 + 异步激活」的分离，让对象在 factory 里就能安全地建立依赖图，而不需要 `async factory` 那种「声明到一半 await」的复杂度。代价是 factory 里不能使用占位符：`resource.value`、`init` cell 的 `value`、引用调用都会抛 `not_ready`。

生命周期由 `lifetime` 决定：默认 `referenced`（第一次 `get` 创建，最后一个引用释放后 + `linger` 销毁）；`host`（facet 加载时创建，host 关闭时销毁，只允许空输入，只允许它 `register`）。`self.state()` 的 cell 跨热重载保留同一个值和同一份 identity；`resource` 在 `version`/`deps`/`exclusive` 不变时保留；`task` 在重载提交时用新代码重启；listener 在 source 和 select 不变时保留订阅、只换 handler。

### 17.2.3 根、代码版本与两版本规则

Chord 把一次短的调用树叫 **root**。`ctx` 携带 `{ signal, rootId, codeVersion, principal }`，显式传递，不依赖 AsyncLocalStorage。

热重载的核心是**两版本规则**：每个 host 最多保留 `current` 和 `previous` 两个代码版本。一个调用带着 code version E 进来，就用 E 的实现；host 没有 E（太旧，或 host 启动得晚）就返回 `superseded`（可重试，开一个新 root），**绝不**用其他版本顶替。

重载分为六步：

1. **构建/类型检查**（应用工具链，Chord 之外）；
2. **加载**：每个受影响 host 用 cache-busting 的方式 import 新 facet；
3. **准备**：对每个 live 对象用新 factory 跑一遍「录制模式」——已有 cell 可读、写入抛错；listener/task/registration 只记录不启动；新 resource 先打开为 candidate；任何失败就 abort；
4. **闸门**：只对「cell 版本升高」或「替换 exclusive resource」的对象，以及依赖它们的对象。先切断旧版本新进入的 root（`upgrade_conflict`），等在飞的 root 结束（默认 5 秒），替换 exclusive resource，执行 `migrate`；
5. **提交**：每个 host 在一个同步步骤里安装迁移后的 cell 值、安装新实现为 code version E'、换 registry 表、重启 task、重绑 listener、推送 shape/reset；
6. **退役**：等 `previous` 的 root 都结束，再丢弃旧实现；超时则 abort 这些 root 后强制退役。

兼容变更（不升版本、不换 exclusive resource）直接跳过闸门，不等待。状态兼容规则是：schema 和版本都没变 = 兼容；版本升高 = 闸门 + `migrate`（或保留旧值）并校验；schema 变了但版本没升 = 拒绝；版本降低 = 拒绝。「语义变了但 schema 没变」检测不出来，只能靠作者自觉——Chord 在文档里明确说了这一点。

### 17.2.4 链接与一致性

Chord 的 host 之间通过 **link** 通信：内存、Node IPC、WebSocket。线上是 JSON 帧：`hello`/`bye`、`hold`/`release`、`shape`、`call`/`result`/`cancel`、`sub`/`unsub`/`resync`/`state`、`gone`、`root_end`、`retire`、`ping`/`pong`、`ack`，以及重载控制帧。

几个值得注意的机制：

- **principal 由接受方盖章**：连接方不能自称身份；父链路可以转发 principal，其他来源的 principal 字段被忽略。
- **流控只拦状态**：两端在 `hello` 里声明 `acks`，按累计帧数确认；未确认的帧超过窗口（默认 256 KiB JSON）时，状态帧在发送端排队/合并。结果帧不被状态挡住，所以 `get` 不会被别的流拖慢。等待中的状态帧按成员流合并，超过每链路上限就退化成一次「当前值 reset」——慢消费者只会看到更新的快照，不会无限占内存。
- **协议版本不协商**：`hello` 带 `PROTOCOL_VERSION`（当前 4）；版本不一致就关闭链路并报告，旧的一方必须加载新代码（浏览器刷新页面）。
- **link 不恢复**：链路断开时 provider 立刻释放所有通过它持有的对象；重连拿到的是新对象。在飞调用 reject `unavailable`，是否执行过未知。这是 Chord 明确列出的限制，也是「远程调用必须幂等」的又一个理由。

> **给 Agent 开发者的启示**：Chord 把「插件热更新」当成一等公民，而不是开发期的便利。它付出的代价是契约版本、闸门、两版本、迁移、流控这一整套机制；换来的能力是「Agent 跑到一半，你可以换掉它的工具实现，而在飞的工作仍然用旧代码安全结束」。如果你要做「可在线升级的 Agent 平台」，这套模型比「重启进程」更接近正确答案。注意它仍然是 Experimental，且有几条明确限制（link 不恢复、commit 决策不持久化、Node loader 会泄漏旧模块）——读它的 `spec.md` 的 Known limitations，比读 API 更有价值。

---

## 17.3 Env：让工具跑在另一台机器上

`pi-env` 解决的是另一个维度：**执行位置**。Durable 的 `ExecutionEnv` 是一个接口，`NodeExecutionEnv` 在本机执行；`RemoteExecutionEnv` 则把同一套操作发给远端 daemon。

架构很小：

- `packages/env/daemon/`：一个 **Rust** 小程序，跑在远端机器上，通过 stdin/stdout 上的分帧协议接收文件操作和命令请求；
- `RemoteExecutionEnv`：客户端，把 `ExecutionEnv` 的方法翻译成协议帧；结果语义与「在远端跑 `NodeExecutionEnv`」一致，只有错误消息可能不同；
- `connectSsh()`：检测远端系统、按 SHA-256 部署/校验 daemon、通过 SSH 启动它；
- `LocalExecutionEnv`：在本机也跑同一个 daemon（`localDaemon()`），于是本地和远程的 `listFiles` 走同一份代码。

使用方式是先验证 host key，再连接：

```ts
const target = { host: "gpu-box", knownHostsFile: "/data/ssh/known_hosts", hostKeyAlias: "pi-env-gpu" };
const { lines, fingerprints } = await scanHostKey(target);
await acceptHostKey(target, lines);

const { connection } = await connectSsh(target);
const env = new RemoteExecutionEnv({ connection, id: "pi-env:gpu", cwd: "/home/me/project" });
```

安全相关的取舍写在 README 里：SSH 用 `BatchMode`、对应用自己的 known-hosts 做严格校验、固定 alias、不做任何端口/agent/X11 转发、不复用连接、不加载用户配置的命令、不转发 locale。host key 必须由人带外比对后 `acceptHostKey`。daemon 按内容哈希命名，每次启动前校验。

语义上最费功夫的是「和本地一致」：`docs/semantics.md` 逐条列出远端与 Node 标准库的差异和复刻方式——递归 `mkdir` 的 `EEXIST`/`ENOTDIR`、`rm` 的 `force`/`recursive`、`mkdtemp` 的模板、`tmpdir` 的优先级、`listDir` 的字节序、非 UTF-8 文件名、`/dev/null` 与 FIFO 的读取、命令的 shell 选择、进程组与超时 kill、输出 spill 到临时文件、文件监视的原生事件与轮询回退。甚至连「命令退出后还要收完两个管道，或 100ms 没有输出才结束」这种细节都规定了。

此外还有：

- `listFiles()`：像 `fd --hidden --exclude .git` 一样列树，不跟随符号链接，尊重 gitignore，有 `maxEntries`/`maxBytes`/`timeoutMs` 和 `truncated`；
- `openTerminal()`：在 daemon 的伪终端里跑程序（默认登录 shell），支持原始输出、输入、resize 和退出码，用于交互式终端；
- 平台覆盖：Linux、macOS、Android/Termux、Windows，x86-64 与 arm64；Windows 上字符串命令走 Git Bash，argv 命令直接执行。

> **给 Agent 开发者的启示**：远程执行不是「把 `child_process.exec` 换成 `ssh`」那么简单。一旦工具跑在另一台机器上，文件语义、路径解析、错误码、进程清理、输出截断、文件监视都会出现平台差异；pi-env 的做法是**先写一份语义规范，再写一致性测试**，让 remote 和 local 跑同一套 conformance suite。另一个容易被忽略的点是凭据：Durable worker、存储和 API key 留在本地，只有工具动作去远端——这比把 key 复制到每台执行机要安全得多。

---

## 17.4 三者如何拼在一起

把三个包放在一张图里，关系是清晰的：

```text
┌───────────────────────────── Durable ─────────────────────────────┐
│ Harness / Session / TaskScheduler                                 │
│   ├── 对话、任务、提交：Storage（Memory / SQLite / JSONL / DO）    │
│   ├── 文档、视图、任务图：Chord（replicated state + watch）        │
│   └── 工具执行：ExecutionEnv ── NodeExecutionEnv（本地）           │
│                                  └── RemoteExecutionEnv ── pi-env daemon（SSH）
└───────────────────────────────────────────────────────────────────┘
```

- Durable 用 Chord 表达文档和可观察状态（`viewState`、`watch`、`taskGraph`）；
- Durable 用 `ExecutionEnv` 抽象工具的执行位置，`pi-env` 提供远程实现；
- Chord 自己只依赖 `delta`、Node/browser 内建和 TypeBox，不依赖 Durable——依赖方向是单向的；
- `env` 依赖 `durable`（为了 `ExecutionEnv` 类型）和 `chord`（复用其基础设施），但 daemon 本身是独立的 Rust 程序。

三者共同体现了一条和 `coding-agent` 不同的工程路线：**`coding-agent` 把复杂度放在「一个进程内的循环与上下文管理」，这三个包把复杂度放在「跨进程、跨机器、跨时间的状态与执行」**。前者优化的是「一次对话的体验」，后者优化的是「Agent 作为长期服务时的可靠性」。

> **给 Agent 开发者的启示**：如果你只需要一个本地 CLI，`coding-agent` 的会话树 + 压缩 + 工具已经足够；如果你要做后台 Agent、多人协作、Serverless 部署或远程执行，Durable 的三条不变量值得直接抄：**提交先于展示、外部效果不入事务、任务检查点即恢复点**。Chord 的契约版本与两版本规则、Env 的「先规范后实现」也各自是一份可复用的设计清单。

## 17.5 为什么 coding-agent 不使用它们

读到这里很容易产生一个疑问：既然 Durable 更可靠、Chord 更解耦、Env 能远程执行，为什么 `coding-agent` 一个都不用？

先看依赖事实：`packages/coding-agent/package.json` 的运行时依赖只有 `pi-ai`、`pi-agent-core`、`pi-tui`、`pi-mcp`、`pi-codemode`，没有 `pi-durable`、`chord`、`pi-env`；`pi-agent-core` 更是只依赖 `pi-ai` 与 TypeBox。这不是遗漏，而是两个产品处在设计空间的不同位置：

| 维度 | `coding-agent` | `durable` |
|---|---|---|
| 进程模型 | 一个本地 CLI 进程，人在终端前 | 可被驱逐、重启、多副本的宿主 |
| 一致性目标 | 会话文件可读、可分支、崩溃后能恢复到最后一条 entry | 提交原子、只有已提交状态可观察 |
| 工具效果 | 就地执行，失败即错误结果，由模型决定重试 | 阶段可重放，要求幂等或用 checkpoint 去重 |
| 延迟取向 | 流式 token 立刻上屏，工具结果立刻回灌 | 先提交、再展示 |
| 扩展模型 | jiti 进程内加载，事件钩子可改写请求 | 定义注册表 + 持久任务，扩展要参与调度 |

具体到三个包：

- **不用 Durable**：`AgentSession` 的循环是「流式产出 → 写 entry → 继续」，追求的是一次对话的即时体验；Durable 的每次状态变化都要走 commit，工具调用是持久任务，还要处理 replay、park、wake。把 coding-agent 搬过去，等于给一个交互式 CLI 套上后台服务的调度与恢复成本，而它并不需要跨重启继续一个「正在跑的工具」。
- **不用 Chord**：coding-agent 是单进程，扩展、TUI、会话管理器共享同一个事件循环；Chord 解决的跨进程对象引用、principal、热重载两版本规则在这里没有对手方。进程内 `EventEmitter` + 显式接口已经够用。
- **不用 Env**：coding-agent 把执行后端压缩成一个更窄的接缝 `BashOperations`（`packages/coding-agent/src/core/tools/bash.ts:75`），远端执行只需实现这个接口；`pi-env` 则要承诺整套文件系统语义、PTY、文件监视和 daemon 部署。前者是「够用的最小抽象」，后者是「跨机器的完整 ExecutionEnv」。

反过来说，这三条边界也不是铁墙：`BashOperations` 可以适配 `RemoteExecutionEnv`，Durable 的示例里也能搭出带工具的编码 Agent。pi 的选择是**让每个产品只支付它需要的复杂度**——这正是全书反复出现的同一条原则：内核给接缝，复杂度按需装配。

> **给 Agent 开发者的启示**：不要因为一个架构「更可靠」就把它塞进所有产品。先问三个问题：状态需不需要跨进程/跨重启？外部效果能不能重放？谁在观察状态？三个答案都指向「需要」时，Durable/Chord/Env 才划算；否则它们带来的提交、调度和契约成本会直接吃掉交互体验。

---

## 本章小结

- `durable` 是一个持久化 Agent Harness：Harness = 存储 + 调度器 + registry；Session 把所有修改收进一条原子提交线；Conversation 是不可变 Entry 的 transcript；Document 是随 commit 修改的 Chord 状态；Task 是带检查点的持久状态机。核心不变量是「只有已提交状态可观察」。
- 任务用 phase + checkpoint 描述，状态为 `pending/running/waiting/sleeping/completing/terminal`；阶段崩溃后从头重跑，所以外部效果必须幂等，跨阶段数据放 checkpoint。`sleepUntil` 把睡眠持久化，重启后由调度器恢复；连续崩溃 `restartLimit` 次会被 park，需要 `retryTask()`。
- 恢复靠重新调度而非内存快照：`requestId` 去重提交，工具 `replay` 策略与嵌套调用 key 去重复效果，Code Mode 用 journal 让脚本重放走同一路径。`onWake` 让可驱逐宿主自己安排唤醒。
- `chord` 用运行时契约（remote/local、input 身份、variants、placement、版本兼容）描述对象；factory 同步声明、对象异步激活；引用跨进程与热重载存活；principal 决定对象与可见成员。
- 热重载靠两版本规则 + prepare/gate/commit/retire：在飞 root 用旧代码跑完，不兼容变更在闸门迁移；兼容变更不等待。限制包括 link 不恢复、commit 决策不持久化、loader 泄漏。
- `pi-env` 用 Rust daemon + stdin/stdout 协议 + SSH 把 `ExecutionEnv` 延伸到远端；严格 host key 校验、按哈希部署、语义与 `NodeExecutionEnv` 对齐，另有 `listFiles`、PTY、跨平台支持。Durable worker/存储/凭据留在本地，只有工具动作去远端。

下一章回到工程视角：pi 如何用 faux provider、类型纪律、测试分层与 dogfooding，把这套越来越大的系统做成「可信、可测、可持续维护」的工程。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/durable/src/harness/harness.ts` | `Harness.open()`、`resume()`/`wake()`、`inspect()`、`abortTask()`/`retryTask()`、`waitForTask()` |
| `packages/durable/src/harness/scheduler.ts` | `TaskScheduler`：reservation、reconciliation、restart limit/parking、任务图 |
| `packages/durable/src/harness/generation.ts` | 内置生成任务：模型调用、工具轮次、压缩衔接 |
| `packages/durable/src/harness/tool.ts` | 工具任务、嵌套调用、replay/checkpoint |
| `packages/durable/src/session/session.ts` / `transaction.ts` | Session 的提交线与事务 |
| `packages/durable/src/storage/*` | Memory / SQLite / JSONL 存储与契约测试 |
| `packages/durable/docs/spec.md` | 规范性文档：记录类型、任务语义、事务、内置任务、存储契约 |
| `packages/chord/src/contract.ts` | `remote`/`local`/`method`/`opaque`、placement、版本兼容 |
| `packages/chord/src/object.ts` / `host.ts` | factory、声明（state/resource/get/on/task/derive/register）、激活、lifetime |
| `packages/chord/src/reload.ts` | prepare / gate / commit / retire 与迁移 |
| `packages/chord/src/link.ts` / `wire.ts` | 内存/IPC/WebSocket 链路、帧协议、流控 |
| `packages/chord/docs/spec.md` | 契约、生命周期、重载、链接、限制的规范 |
| `packages/env/daemon/` | Rust 远端 daemon：文件、命令、监视、PTY |
| `packages/env/src/remote-env.ts` | `RemoteExecutionEnv`：把 `ExecutionEnv` 映射到协议 |
| `packages/env/src/ssh.ts` | host key 扫描/接受、SSH 连接与 daemon 部署 |
| `packages/env/docs/protocol.md` / `semantics.md` | 线协议与「与本地一致」的逐条语义 |

## 动手实验

1. 用 `MemoryStorage` 跑一遍 Durable 的 Quick Start，然后故意在 `deliver` 阶段里 `throw`。观察任务如何 fault、`onReport` 收到什么、`inspect()`/任务图如何显示；再用 SQLite storage 重跑，在外部效果前后各 `process.exit()` 一次，确认 `resume()` 从哪个 checkpoint 继续。把外部效果换成非幂等的（比如追加一行日志），看重复执行发生在哪里——这就是「效果三明治」要解决的问题。
2. 写一个两阶段的 `defineTask`：第一阶段只 `sleepUntil(Date.now() + 2000)`，第二阶段提交 terminal。用假的 `now`/`setTimer` 跑，确认睡眠期间任务不占 invocation；再在睡眠中重开 Harness，确认睡眠仍然有效。对照 `settings.restartLimit` 把限制调成 1，制造一次确定性崩溃，观察 park 与 `retryTask()`。
3. 读 `packages/chord/docs/spec.md` 的 §7 Reload，然后对照 `packages/chord/test/` 里 reload 相关测试，找出「兼容变更不等待」「版本升高走闸门 + migrate」「late old root 得到 `upgrade_conflict`」分别由哪个测试覆盖。再读 §12 的 Known limitations，挑一条（比如 link 不恢复）想清楚它对你的调用方意味着什么。
4. 读 `packages/env/docs/semantics.md`，挑三条与 Node 标准库不同的行为（例如递归 `mkdir`、`rm`、`listDir` 的字节序），在 `daemon/` 里找到对应实现；再用 `LocalExecutionEnv` 与 `NodeExecutionEnv` 跑同一段操作，验证 `test/differential.test.ts` 的思路——为什么「本地也跑 daemon」能让一致性更容易保证？
