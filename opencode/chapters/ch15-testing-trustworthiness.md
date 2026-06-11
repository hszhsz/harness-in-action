# 第 15 章 可测试性与可信度:把每一条不变式钉进断言

一个 AI agent 的可怕之处在于它的**不确定性**:同样的输入,模型可能给出不同的输出;工具在不同环境里行为各异;并发的 fiber 让时序难以复现。在这样的系统里,怎么知道一次重构没有悄悄破坏权限边界?怎么确认压缩逻辑在边界条件下不丢消息?答案不是"小心点",而是**测试**——把每一条你不想被破坏的不变式,写成一个会在回归时立刻失败的断言。

OpenCode 的 `packages/core` 里有 128 个测试文件。这一章我们不逐个看,而是抓住它的测试**基础设施**:一个把 Effect、内存数据库、虚拟时钟缝合在一起的测试调和层,以及它如何让前面十四章那些设计(类型化错误、Layer 注入、纯函数边界)同时也成为"可测的"。你会发现,这个系统好测,不是因为后期补了测试,而是因为它从架构上就是为可测性设计的。

## 15.1 测试调和层:让 Effect 测试既确定又简洁

测试的地基在 `packages/core/test/lib/effect.ts`。它把 Effect 程序包装成 Bun 的测试用例,核心是一个 `run` 函数:

```ts
const run = <A, E, R, E2>(value: Body<A, E, R | Scope.Scope>, layer: Layer.Layer<R, E2>) =>
  Effect.gen(function* () {
    const exit = yield* body(value).pipe(Effect.scoped, Effect.provide(layer), Effect.exit)
    if (Exit.isFailure(exit)) {
      for (const err of Cause.prettyErrors(exit.cause)) {
        yield* Effect.logError(err)
      }
    }
    return yield* exit
  }).pipe(Effect.runPromise)
```

短短几行,做对了四件关键的事:

1. **`Effect.scoped`**:每个测试都跑在自己的作用域里,测试结束时所有申请的资源(数据库连接、订阅、临时文件)被自动回收。测试之间不会因为资源泄漏而互相污染。
2. **`Effect.provide(layer)`**:把测试需要的依赖一次性注入。这正是依赖注入在测试里的兑现——被测代码声明它要什么服务,测试提供"测试版"的那些服务。
3. **`Effect.exit` + `Cause.prettyErrors`**:不让失败直接抛成不可读的异常,而是捕获完整的 `Exit`,把因果链(cause)漂亮地打印出来。一个测试失败时,你看到的是结构化的错误树,而不是一行干瘪的堆栈。
4. **`Effect.runPromise`**:最后才把 Effect 跑成 Bun 能理解的 Promise。

最巧妙的是 `make` 函数同时产出两套运行器——`effect` 和 `live`:

```ts
const testEnv = Layer.mergeAll(TestConsole.layer, TestClock.layer())
const liveEnv = TestConsole.layer
export const it = make(testEnv, liveEnv)
export const testEffect = <R, E>(layer: Layer.Layer<R, E>) =>
  make(Layer.provideMerge(layer, testEnv), Layer.provideMerge(layer, liveEnv))
```

`it.effect(...)` 跑在 `testEnv` 里,它带了 **`TestClock`**——一个虚拟时钟。这意味着涉及超时、延迟、调度的逻辑可以被**确定性**地测试:你不用真的 `sleep` 五秒,而是手动把虚拟时钟拨快五秒,瞬间得到结果。`it.live(...)` 则用真实时钟,留给那些必须和真实时间打交道的场景。两套环境共享同一份测试代码,只换底层时钟——这就是为什么 OpenCode 的测试既快又不 flaky:大部分时序被虚拟化了。

`testEffect(layer)` 是给具体模块用的工厂:把模块自己的依赖 Layer 和测试环境 `provideMerge` 起来,产出一个绑定好依赖的 `it`。我们马上在权限测试里看到它怎么用。

## 15.2 内存数据库:确定性测试的另一半

时钟之外,确定性的另一大敌是**数据库状态**。如果测试跑在真实的磁盘库上,前一个测试的残留会污染后一个,顺序一变结果就变。OpenCode 的解法干净利落——用 SQLite 的内存模式。

在 `packages/core/test/permission.test.ts` 的开头就能看到:

```ts
const database = Database.layerFromPath(":memory:")
```

还记得第 12 章 `database.ts` 里的 `layerFromPath(filename)` 吗?它接受任意文件名,而 `:memory:` 是 SQLite 的特殊路径,表示"建一个纯内存的库,进程结束即灰飞烟灭"。每个测试文件拿到一个全新的空库,跑完即弃。第 12 章那套迁移机制照常运行——内存库也会建表、应用迁移——所以测试跑的是和生产**完全相同**的 schema,只是介质从磁盘换成了内存。

这就是分层架构的红利:因为数据库是一个被注入的 Layer,把它从磁盘换成内存,被测代码一个字都不用改。生产用 `path()` 解析出的磁盘路径,测试用 `:memory:`,同一套 `Database.Service` 接口。

## 15.3 用 Layer 搭一个真实但隔离的世界

权限测试的依赖组装,是观察 OpenCode 测试哲学的最佳样本。它没有 mock 掉权限引擎的依赖,而是用**真实的实现**、只把最外层的输入输出换成可控的测试版:

```ts
const events = EventV2.layer.pipe(Layer.provide(database))
const store = SessionStore.layer.pipe(Layer.provide(database))
const sessions = SessionV2.layer.pipe(
  Layer.provide(events), Layer.provide(database), Layer.provide(store),
  Layer.provide(Project.defaultLayer),
  Layer.provide(SessionExecution.noopLayer),
)
const saved = PermissionSaved.layer.pipe(Layer.provide(database))
const layer = PermissionV2.locationLayer.pipe(
  Layer.provideMerge(database), Layer.provideMerge(store),
  Layer.provideMerge(events), Layer.provideMerge(current),
  Layer.provideMerge(sessions), Layer.provideMerge(SessionExecution.noopLayer),
  Layer.provideMerge(saved),
)
const it = testEffect(layer)
```

注意这里用的是**真实的** `EventV2.layer`、`SessionStore.layer`、`SessionV2.layer`、`PermissionSaved.layer`——它们全都接到同一个内存 `database` 上。测试里的事件真的会落进内存库的 event 表,权限真的会经过第 10 章那套 `evaluate` + `findLast` 引擎,saved 规则真的会写进 `PermissionTable`。

只有两处被换成了测试替身:`SessionExecution.noopLayer`(执行后端换成空操作,因为权限测试不关心真的跑工具)和 `Location.Service`(用一个固定 `/project` 目录的 fixture)。这是一种克制而高明的测试策略:**测真实的协作,只隔离不相关的副作用**。它测的不是"权限引擎的单个函数返回什么",而是"权限引擎接上真实的事件系统、会话存储、saved 仓库之后,整体行为对不对"——这才是回归最容易出问题的地方。

## 15.4 断言即不变式:权限测试读出的规格

把第 10 章的权限规则口头描述一遍,和把它写成会失败的断言,是两回事。后者才是真正的规格。我们看权限测试如何把第 10 章那些抽象规则钉成具体断言。

**三效果直接求值,ask 不入队**:

```ts
yield* setup([{ action: "read", resource: "*", effect: "allow" }])
expect(yield* service.ask(assertion())).toEqual({ id: ..., effect: "allow" })
expect(yield* service.list()).toEqual([])  // allow 不产生待决请求
yield* setRules([{ action: "read", resource: "*", effect: "deny" }])
expect(yield* service.ask(assertion())).toEqual({ id: ..., effect: "deny" })
yield* setRules([])
expect(yield* service.ask(assertion())).toEqual({ id: ..., effect: "ask" })
expect(yield* service.get(...)).toBeDefined()  // 只有 ask 留下待决记录
```

这一个测试用例就把"allow/deny 立即返回且不入队、ask 才创建待决请求"这条规则锁死了。注意它还覆盖了第 10 章的默认值——空规则集 `setRules([])` 下结果是 `ask`,正是 `evaluate` 那个 `?? { effect: "ask" }` 兜底的体现。

**fail-closed 的默认**(对应第 10 章 `missingAgentPermissions`):

```ts
it.effect("denies omitted-agent permissions when no primary default agent exists", () =>
  ... // 删掉 test 和 build 两个 agent,会话 agent 也置 null
  expect(yield* service.ask(assertion())).toEqual({ id: ..., effect: "deny" })
)
```

当没有任何可用 agent 时,结果是 `deny` 而不是 `allow`。这正是第 10 章"找不到 agent 就当全 deny"的 fail-closed 设计——它被写成了一条会在有人不小心改成 fail-open 时立刻变红的测试。

**saved 规则只能升级、不能压过 deny**(对应第 10 章的精妙规则):

```ts
yield* saved.add({ projectID: Project.ID.global, action: "bash", resources: ["pwd"] })
expect(yield* service.ask(assertion({ action: "bash", resources: ["pwd"] }))).toEqual({ effect: "allow" })
yield* setRules([{ action: "bash", resource: "*", effect: "deny" }])
expect(yield* service.ask(assertion({ action: "bash", resources: ["pwd"] }))).toEqual({ effect: "deny" })
```

先验证 saved 的 `bash pwd` 批准能把 ask 升级成 allow;再加一条 agent 级的 deny,验证 saved 批准**压不过** agent 的 deny。第 10 章那句"saved 规则只能把 ask 升级成 allow,永远不能覆盖 agent 的 deny",在这里有了不可辩驳的证据。

**异步 ask→reply 解决一次**:

```ts
function waitForRequest() {
  return Effect.gen(function* () {
    ...
    const unsubscribe = yield* events.listen((event) =>
      event.type === PermissionV2.Event.Asked.type
        ? Deferred.succeed(asked, event.data as ...).pipe(Effect.asVoid) : Effect.void)
    yield* Effect.addFinalizer(() => unsubscribe)
    const fiber = yield* service.assert(assertion()).pipe(Effect.forkScoped)
    const request = yield* Deferred.await(asked)
    return { service, fiber, request }
  })
}
```

这段测试辅助函数本身就是一份"如何测异步等待"的范本:它 `forkScoped` 把 `assert` 派到后台 fiber(因为 ask 会阻塞等回复),用 `events.listen` 监听 `Asked` 事件、用 `Deferred` 接住那个请求,再 `addFinalizer` 保证订阅在测试结束时取消。然后主测试 `reply({ reply: "once" })`、`Fiber.join(fiber)`,断言请求被消费、`list()` 清空。第 10 章那个"ask 创建 Deferred、reply 唤醒它"的机制,在这里被完整地、确定性地复现了——没有任何真实的等待,全靠 Deferred 和 fiber 编排。

## 15.5 工具测试的统一入口

工具是 agent 最容易出 bug 的地方(它们和真实文件系统、真实命令打交道)。OpenCode 给工具测试也配了统一的辅助层 `test/lib/tool.ts`:

```ts
export const settleTool = (registry, input) =>
  registry.materialize().pipe(Effect.flatMap((materialized) => materialized.settle(input)))
export const executeTool = (registry, input) =>
  settleTool(registry, input).pipe(Effect.map((settlement) => settlement.result))
```

它把"物化工具注册表 → 执行一个工具调用 → 取结果"这条链路封装成一个函数。`toolDefinitions` 还能传入 `permissions` 参数——这让工具测试可以在不同权限配置下验证工具行为,把第 8 章的工具系统和第 10 章的权限系统在测试里接到一起。统一的入口意味着每个工具测试不必重复样板,只专注于自己的断言。

## 15.6 临时目录:文件系统测试的纪律

凡是碰文件系统的测试,都要面对一个老问题:测试留下的脏文件怎么清?`test/fixture/tmpdir.ts` 给了一个优雅的答案:

```ts
export const tmpdir = async () => {
  const dir = await fs.realpath(await fs.mkdtemp(path.join(osTmpdir(), "opencode-core-test-")))
  return { path: dir, async [Symbol.asyncDispose]() { await remove(dir) } }
}
```

它返回一个实现了 `Symbol.asyncDispose` 的对象——配合 `await using` 语法,临时目录会在作用域结束时**自动删除**,无论测试成功还是失败。`realpath` 解软链(macOS 的 `/tmp` 是软链,不解会导致路径比较失败),前缀 `opencode-core-test-` 让残留目录一眼可辨。

`remove` 函数还藏着一个真实世界的细节——对 `EBUSY` 错误重试:

```ts
async function remove(dir: string, retries = 30): Promise<void> {
  try { await fs.rm(dir, { recursive: true, force: true }) }
  catch (error) {
    if (retries === 0 || ... error.code !== "EBUSY") throw error
    Bun.gc(true); await Bun.sleep(100); return remove(dir, retries - 1)
  }
}
```

在 Windows 或文件被句柄占用时,`rm` 会报 `EBUSY`。这里的处理是:强制 GC(释放可能还攥着文件句柄的对象)、睡 100 毫秒、重试,最多 30 次。这不是过度工程,而是从 CI 上真实的 flaky 失败里学来的教训——测试基础设施本身也要可信,否则它产生的"失败"会变成噪音,侵蚀团队对测试的信任。

## 15.7 可信度是一条流水线

把这一章的观察连起来,会看到一个比"写测试"更大的图景:OpenCode 的可信度不是某一处的努力,而是一条贯穿架构的流水线。

- **架构让代码可测**:因为依赖是 Layer 注入的,测试才能把磁盘库换成内存库、把执行后端换成 noop;因为错误是类型化的值,测试才能用 `Effect.flip` 精确断言"这里应该失败成 DeniedError";因为副作用被收进 Effect,测试才能用 TestClock 把时序虚拟化。这些第 1 到 14 章反复出现的设计,在这里兑现成了"可测性"。
- **测试隔离但不失真**:用真实的协作实现(真事件、真存储、真权限引擎)接到内存库上,只隔离无关副作用。测的是集成行为,而非孤立的函数。
- **断言即规格**:权限的每一条规则——三效果、fail-closed 默认、saved 只升级不覆盖、ask 解决一次——都被钉成了会在回归时变红的断言。文档会过时,断言不会。
- **基础设施本身要可信**:虚拟时钟消除 flaky 时序,自动清理的临时目录加 EBUSY 重试消除 flaky 文件系统。一个会随机失败的测试套件,比没有测试更糟,因为它教会团队忽略红色。

对一个本质上不确定的 AI 系统,这条流水线就是把"我觉得它没坏"变成"我能证明它没坏"的全部依凭。它也呼应了这本书反复出现的那条主线:**不要把正确性寄托在运行时的自觉,把它钉进类型、约束,和测试里**。

## 本章小结

- `test/lib/effect.ts` 的 `run` 用 `Effect.scoped`(资源自动回收)+ `Effect.provide`(依赖注入)+ `Cause.prettyErrors`(结构化错误)+ `Effect.runPromise` 把 Effect 程序包成 Bun 测试;`make` 同时产出带 `TestClock` 的 `it.effect` 和带真实时钟的 `it.live`,让时序可被确定性测试。
- 数据库测试用 `Database.layerFromPath(":memory:")`——复用第 12 章的同一套迁移和 schema,只把介质从磁盘换成内存,得益于 Database 是可注入的 Layer,被测代码无需改动。
- 权限测试用**真实**的 `EventV2`/`SessionStore`/`SessionV2`/`PermissionSaved` 实现接到内存库,只把 `SessionExecution` 换 noop、`Location` 换 fixture——测真实协作、只隔离无关副作用。
- 权限测试把第 10 章的规则钉成断言:allow/deny 立即返回且不入队、空规则集兜底为 ask、无可用 agent 时 fail-closed 为 deny、saved 只能把 ask 升级为 allow 却压不过 agent deny、ask 经 `forkScoped` + `Deferred` + `events.listen` 异步解决一次。
- 工具测试用 `test/lib/tool.ts` 的 `settleTool`/`executeTool` 统一"物化注册表→执行→取结果",并能注入权限配置,把第 8 章工具系统与第 10 章权限系统在测试里接通。
- 文件系统测试用 `tmpdir.ts`:`Symbol.asyncDispose` + `await using` 自动清理、`realpath` 解软链、`EBUSY` 强制 GC 后重试最多 30 次——保证测试基础设施本身不 flaky。
- 可信度是一条流水线:架构(Layer 注入、类型化错误、Effect 收束副作用)让代码可测,测试隔离但不失真,断言即规格,基础设施本身也要可信。

## 动手实验

1. **跑一次权限测试并读它的断言**:在 `packages/core` 下运行 `bun test test/permission.test.ts`,确认全绿。然后挑 `"denies omitted-agent permissions when no primary default agent exists"` 这个用例,把第 10 章的 `missingAgentPermissions` 临时改成 `allow`,重跑,观察它如何立刻变红。体会"断言守护 fail-closed"的实感。改完记得改回去。

2. **验证内存库的隔离性**:阅读 `permission.test.ts` 的 `setup` 函数,它每次都往 `ProjectTable`/`SessionTable` 插入数据。思考:为什么多个测试用例反复插入同一个 `ses_test` 不会冲突报错?(提示:看 `:memory:` 的生命周期 + `onConflictDoNothing`。)然后试着把 `:memory:` 换成一个磁盘文件路径,连跑两次测试,观察脏数据如何破坏第二次运行。

3. **用 TestClock 测一个超时**:找一个涉及延迟/超时的 core 模块测试(grep `TestClock` 看哪些测试用了它),阅读它如何用 `TestClock.adjust` 拨动虚拟时钟来触发超时,而不是真的等待。对照 `it.effect` 和 `it.live` 的区别,理解为什么大部分测试该用前者。

4. **给临时目录清理制造 EBUSY**:阅读 `tmpdir.ts` 的 `remove`,设计一个场景:在 `await using` 的临时目录里打开一个文件句柄但不关闭,然后让作用域结束。观察(或推理)`remove` 的 EBUSY 重试逻辑如何介入。思考:如果去掉 `Bun.gc(true)` 这一行,重试还能成功吗?为什么强制 GC 是这里的关键?

---

最后一章不再引入新代码。第 16 章将把前面十五章散落的观察,提炼成一组**与语言、框架无关、可迁移**的工程原则——每一条原则都回指到前面具体的章节作为证据。我们会看到,无论你用什么语言写自己的 agent,OpenCode 在 fail-closed 默认、约束只收紧、错误即文档、边界不信任、保结构去内容、不变式入类型与测试、可信度即流水线这七个透镜下展现的判断,都是可以直接借走的。
