# 第 8 章　工具系统：一个工具从声明到结算的一生

> 第 5 章主循环里那句 `toolMaterialization.settle(...)`，第 7 章 `LLMEvent` 里那组 `tool-call`/`tool-result`，都在这一章落地。
> 模型吐出一个工具调用，到副作用真正发生、再到结果被"边收边结算"喂回模型——这条链路上，OpenCode 用一套不到一千行的代码，把"工具"建模成一个**可注册、可物化、可结算、输出有上限**的一等公民。

第 1 章我们说，Harness 的六大职责里有一条是"工具"——让模型能伸手触碰真实世界。但"让模型调工具"这件事远不止"定义一个函数"那么简单：工具要能按 agent 权限被裁剪、要能被进程级和位置级两种作用域注册、要能在并发结算时不串味、输出还不能撑爆上下文。本章读 `packages/core/src/tool/` 这个目录（`tool.ts`、`registry.ts`、`tools.ts` 加几个 builtin），以及 `tool-output-store.ts`，看 v2 怎么把这些约束一条条钉进代码。读完你会理解，为什么这个目录的 `AGENTS.md` 开头第一句就强调它只拥有"**one local tool representation**"——一种工具表示，没有第二种。

## 8.1 `Tool.make`：一个不透明的规范工具值

整个工具系统的原子，是 `tool.ts` 里的 `Tool.make`。它的 `AGENTS.md` 把意图讲得很硬气：

> `tool.ts` defines the opaque canonical `Tool.make({ description, input, output, execute, toModelOutput })` value. Application tools and shipped built-ins use the same type.

一个工具就是五个字段的配置：

```ts
type Config<Input, Output> = {
  readonly description: string
  readonly input: Input        // Schema 编解码器
  readonly output: Output      // Schema 编解码器
  readonly execute: (input, context) => Effect.Effect<Output, ToolFailure>
  readonly toModelOutput?: (input) => ReadonlyArray<Content>
}
```

注意 `Tool.make` 的返回值有多"不透明"：

```ts
const tool = Object.freeze({}) as Definition<Input, Output>
// ……
runtimes.set(tool, { definition, settle, permission })
return tool
```

它返回的是一个**被冻结的空对象** `Object.freeze({})`，真正的运行时能力（怎么生成定义、怎么结算、要哪个权限）被存在一张**模块级的 `WeakMap`**（`runtimes`）里，用这个空对象当 key。这是一种刻意的封装：对外，一个工具值就是个不可变的、什么都看不出来的句柄；它的 codec、executor、定义派生、权限声明全是"private runtime details"（`AGENTS.md` 原话）。想用它，只能通过 `definition(name, tool)`、`settle(tool, call, context)`、`permission(tool, name)` 这几个模块函数，它们内部都先 `runtimeOf(tool)` 从 WeakMap 里把运行时捞出来——捞不到就直接 `throw new TypeError("Invalid Core Tool value")`。

**`settle` 函数是一个工具执行的完整管道**，它把"不可信的模型输入"和"必须合规的输出"两头都用 schema 卡死：

```ts
settle: (call, context) =>
  Schema.decodeUnknownEffect(config.input)(call.input).pipe(
    Effect.mapError((error) => new ToolFailure({ message: `Invalid tool input: ${error.message}` })),
    Effect.flatMap((input) =>
      config.execute(input, context).pipe(
        Effect.flatMap((output) =>
          Schema.encodeEffect(config.output)(output).pipe(
            Effect.mapError((error) =>
              new ToolFailure({ message: `Tool returned an invalid value for its output schema: ${error.message}` })),
          ),
        ),
        // ……
```

三步一线：**先把模型填进来的 `call.input` 按 input schema 解码**（解不出就报 `Invalid tool input`），**再 execute**，**最后把结果按 output schema 编码**（编不出就报 `Tool returned an invalid value for its output schema`）。模型给的参数是边界数据，工具产出也要保证结构合法——这正是第 1 章"边界不可信"原则在工具入口的双向落地：进来的要校验，出去的也要校验。

工具名也有规矩。`validateName` 用正则 `/^[A-Za-z][A-Za-z0-9_-]{0,63}$/` 限制名字：字母开头、只含字母数字和 `-_`、最长 64 字符。不合规直接 `RegistrationError`——又一个"不变量固化进校验"的微观样本。

## 8.2 两种作用域：application 与 Location

工具不是"全局一张表"。`AGENTS.md` 的"Registration"一节划出了两条注册通道：

> Built-ins register through `Tools.Service.register({ [name]: tool })`. Application tools register through `ApplicationTools.Service.register(...)`, exposed publicly as `opencode.tools.register(...)`.

- **Application 工具**是**进程作用域**的，被所有 Location 共享——比如插件通过 `opencode.tools.register(...)` 注册的工具。
- **内置工具**是 **Location 作用域**的，绑定在某个具体的运行位置上——这呼应第 4 章"Runner、模型解析、工具注册表、权限、文件系统全是 Location-scoped"那条规则。

`registry.ts` 里的 `materialize` 把两者**叠起来**，Location 注册覆盖 application 注册：

```ts
materialize: (permissions = []) => {
  const registrations = new Map(applications.entries())   // 先铺 application 层
  for (const [name, entries] of local) {                  // 再用 Location 层覆盖
    const registration = entries.at(-1)?.registration
    if (registration) registrations.set(name, registration)
  }
  // ……
}
```

`AGENTS.md` 用四条铁律描述这套作用域语义，每条都对应注册表里的具体代码：

> - The latest active same-placement registration wins.
> - Closing any registration removes only that registration and reveals the next active one.
> - Location registrations take precedence over application registrations.
> - An invocation captures the effective tool once settlement starts.

第一、二条体现在 `register` 用的是**栈式注册**——同名工具的注册被 push 进一个数组（`local.set(name, [...prev, { token, registration }])`），`at(-1)` 取最新的那个；而注册时挂的 `Effect.addFinalizer` 在作用域关闭时，只 `filter` 掉自己 token 对应的那条，露出下一个。**注册是带生命周期的，关掉一个不会误伤别人。** 整个 register 还包在 `Effect.uninterruptible` 里，保证"加进表 + 挂 finalizer"这串状态变更是原子的。

## 8.3 按权限物化：看不见 ≠ 不执行

第 5 章我们看到主循环调 `tools.materialize(agent.info?.permissions)`——工具是**按当前 agent 的权限被物化的**。这里看物化时那道权限闸门：

```ts
for (const [name, registration] of registrations)
  if (whollyDisabled(permission(registration.tool, name), permissions)) registrations.delete(name)
```

`whollyDisabled` 判断一个工具是不是被权限规则**整体禁掉**了：

```ts
function whollyDisabled(action: string, rules: PermissionV2.Ruleset) {
  const rule = rules.findLast((rule) => Wildcard.match(action, rule.action))
  return rule?.resource === "*" && rule.effect === "deny"
}
```

它倒着找最后一条匹配该 action 的规则——只有当那条规则是"**对所有资源（`resource === "*"`）一律拒绝（`effect === "deny"`）**"时，才把工具从定义列表里删掉。换句话说，只有"彻底没救"的工具才从模型的工具清单里消失。

这里藏着一个极其重要、极易被误解的区分，`AGENTS.md` 专门用一节强调：

> Definition filtering is catalog visibility, not execution authorization. A call still executes the captured leaf policy if it reaches settlement.

**物化阶段的权限过滤，过滤的是"模型能不能看见这个工具"（catalog visibility），不是"这次调用能不能执行"（execution authorization）。** 一个工具即便通过了物化、出现在模型的工具清单里，它真正执行时还会再走一遍自己内部的权限断言（下一节的 `permission.assert`，也是第 10 章的主角）。注册表本身**根本不依赖 `PermissionV2.Service`**、不做任何执行授权——`AGENTS.md` 说得很白："The registry has no `PermissionV2.Service` dependency and performs no execution authorization." 这是一道清晰的职责分离：**注册表管"给模型看什么菜单"，工具叶子自己管"这一口能不能真吃下去"。**

`AGENTS.md` 还交代了一个权限命名细节：大多数工具的权限 action 默认就是它的注册名，但 `edit`、`write`、`apply_patch` 三个共享同一个 `edit` action——三种改文件的方式，用一把权限闸门统一管。这正是 `withPermission(tool, "edit")` 这个装饰器干的事。

## 8.4 防"过期工具调用"：用 identity 对身份

并发与流式带来一个微妙问题：模型在某一轮看到的工具菜单，到工具真正结算时，菜单可能已经被并发地换掉了（比如插件重新注册了同名工具）。如果不管，模型就可能"调用一个它以为存在、其实已经被替换"的工具。`settleWith` 用一个 `identity` 对象解决它：

```ts
const registration =
  local.get(input.call.name)?.at(-1)?.registration ?? applications.entries().get(input.call.name)
if (!registration)
  return { result: { type: "error", value: advertised ? `Stale tool call: ${input.call.name}` : `Unknown tool: ${input.call.name}` } }
if (advertised && registration.identity !== advertised)
  return { result: { type: "error", value: `Stale tool call: ${input.call.name}` } }
```

物化时，`settle` 闭包捕获了当时那个 registration 的 `identity` 引用（`settleWith(input, registration.identity)`）。结算时再查一次当前同名工具的 identity，**两个 identity 引用不相等，就判定为 `Stale tool call`**——这次调用瞄准的工具版本已经过期了。`AGENTS.md` 那条"An invocation captures the effective tool once settlement starts"就是这个机制：**一次调用在结算开始时就锁定了它要执行的那个工具实例，中途被换不算数。** 注意它还区分了两种错法：完全找不到工具是 `Unknown tool`，找到了但 identity 对不上是 `Stale tool call`——又一次"错误即文档"，两种失败有各自的人话。

## 8.5 输出上限：50KB / 2000 行，超了就落盘

工具能返回任意大的输出——一个 `grep` 可能刷出几万行。如果原样喂回模型，上下文瞬间被撑爆、成本爆炸。`tool-output-store.ts` 是这条线的总闸，它定义了三个硬限额：

```ts
export const MAX_LINES = 2_000
export const MAX_BYTES = 50 * 1024
export const RETENTION = Duration.days(7)
```

**每个工具输出最多 2000 行、50KB**，超限的完整内容落盘保留 7 天。`AGENTS.md` 点明这是唯一的结算与裁剪边界："`ToolRegistry.Materialization.settle` is the only execution and generic model-output bounding boundary and owns managed retention paths." `registry.ts` 的 `settleWith` 在工具执行之后、返回之前，强制过一道 `resources.bound(...)`：

```ts
const bounded = yield* resources.bound({ sessionID, toolCallID, output })
const result = ToolOutput.toResultValue(bounded.output)
```

`bound` 的逻辑很务实：先算输出的行数和字节数，**两者都在限额内就原样返回**（`outputPaths: []`）；超了就把完整内容写进一个 `tool_<id>` 文件，给模型看的只是一段**头尾采样的预览**加一行醒目的标记：

```ts
const outputPath = yield* write(contextual)
const marker = `... output truncated; full content saved to ${outputPath} ...`
```

`preview`/`boundedPreview` 这组函数把内容切成"头一半 + 尾一半"（`headLines = Math.ceil(maxLines/2)`、`tailLines = Math.floor(maxLines/2)`），中间塞进 marker。**为什么是头尾都留、而不是只留头？** 因为很多工具输出（编译错误、测试结果）的关键信息在末尾——只截头会丢掉最重要的部分。这是第 1 章"保结构、去内容"原则的又一次体现：宁可在中间挖掉一段、用 marker 标明"完整内容在某文件里"，也要让模型同时看到开头和结尾。

限额还能被配置覆盖——`limits()` 会读 config 里的 `tool_output.max_lines`/`max_bytes`，没配才用默认值。落盘的文件由一个每小时跑一次的 `cleanup` 按 7 天 `RETENTION` 清理，且 `cleanupLayer` 的注释强调它"Runs retention scanning once globally rather than once per active Location"——清理是全局跑一次，不是每个 Location 各跑一遍。

还有一个分层细节值得点出：`AGENTS.md` 说生产者自己的捕获上限是**另一回事**——"Bash keeps `AppProcess.maxOutputBytes` and accurately reports stdout/stderr capture loss, but it does not run model-output truncation"。也就是说 Bash 工具在**捕获**进程输出时有自己的字节上限（第 9 章细说），那是"别让子进程刷爆内存"；而这里的 50KB/2000 行是"别让结果刷爆模型上下文"——两道上限管的是不同的风险，各司其职。

## 8.6 一个真实工具：`read` 的路径逃逸防御

抽象讲完，看一个真 builtin 怎么把这些拼起来。`read.ts` 的 `execute` 是一段教科书级的边界防御：

```ts
const absolute = path.resolve(location.directory, input.path)
const selected = path.isAbsolute(input.path) ? path.dirname(absolute) : location.directory
if (!path.isAbsolute(input.path) && !FSUtil.contains(location.directory, absolute))
  return yield* Effect.die(new Error("Path escapes the allowed read root"))
const real = yield* fs.realPath(absolute).pipe(Effect.orDie)
const root = yield* fs.realPath(selected).pipe(Effect.orDie)
if (!FSUtil.contains(root, real))
  return yield* Effect.die(new Error("Path escapes the allowed read root"))
```

它**两次**校验路径不逃逸：先对原始路径查一遍 `FSUtil.contains`，再对 `realPath`（解析符号链接后的真实路径）查一遍。为什么要查两遍？因为符号链接可以指向根目录外——只查字符串路径会被 `ln -s /etc/passwd ./link` 这种把戏绕过，必须把符号链接解析成真实路径再查一次。这正是第 1 章"边界不可信"在文件系统攻击面上的防御。

校验通过后，它才调 `permission.assert(...)` 做执行授权（注意 `source` 严格按 `AGENTS.md` 规定的格式从 `context` 构造），再真正读文件。`read` 还顺手做了图片处理：支持的图片 MIME 走 `image.normalize`（缩放），二进制文件直接 `BinaryFileError` 拒绝。`AGENTS.md` 给叶子工具立的规矩是"Leaves own resolution, permission, and side-effect ordering. Translate only expected typed errors into `ToolFailure`; do not use `catchCause`, because interruption and defects must survive"——**只把预期内的错翻译成 `ToolFailure`，绝不用 `catchCause` 把中断和 defect 一起吞掉**。看它的 `mapError`：只对几种已知错误（`BinaryFileError`、`MediaIngestLimitError`、`Image.DecodeError/SizeError`）保留原文，其余统一成 `Unable to read ...`——既给用户有用信息，又不让内部细节泄露。

## 本章小结

- `Tool.make` 产出一个**不透明的、被 `Object.freeze` 冻结的句柄**，真正的运行时能力存进模块级 `WeakMap`，对外只通过 `definition`/`settle`/`permission` 三个函数访问——一种工具表示，没有第二种。
- `settle` 管道两头用 schema 卡死：模型输入按 input schema 解码（失败报 `Invalid tool input`）、工具产出按 output schema 编码（失败报输出非法）——边界不可信的双向落地；工具名受正则 `/^[A-Za-z][A-Za-z0-9_-]{0,63}$/` 约束。
- 两种作用域：application 工具进程级共享、内置工具 Location 级绑定；`materialize` 把两层叠起来、Location 覆盖 application；注册是栈式带 finalizer 的，最新者胜、关一个只露下一个、整个注册原子不可中断。
- 物化时按 agent 权限过滤，但 `whollyDisabled` 只删"对所有资源一律 deny"的工具；**这是 catalog 可见性、不是执行授权**——注册表不依赖权限服务，执行授权由工具叶子内部的 `permission.assert` 负责。
- `settleWith` 用 `identity` 引用比对防"过期工具调用"：结算时锁定的工具实例若被并发替换，报 `Stale tool call`，与 `Unknown tool` 区分。
- `tool-output-store.ts` 强制 `MAX_LINES=2000`/`MAX_BYTES=50KB` 上限，超限内容落盘 7 天、给模型头尾采样预览加 marker——保结构去内容；它是唯一的结算与裁剪边界，与 Bash 自身的捕获上限分属不同风险。
- `read.ts` 示范叶子工具：两次路径逃逸校验（含 realPath 解符号链接）、`permission.assert` 执行授权、只把预期错误译成 `ToolFailure` 而不吞中断与 defect。

## 动手实验

1. **实验一：确认"一种工具表示"** — 打开 `tool/tool.ts`，找到 `runtimes` 这个 `WeakMap` 和 `Object.freeze({})`。用一句话解释：为什么把运行时能力藏进 WeakMap、而不是直接挂在返回对象的字段上？这对"工具值不可变"有什么好处？
2. **实验二：手推一次权限过滤** — 在 `registry.ts` 读 `whollyDisabled`。假设权限规则是 `[{action: "bash", resource: "*", effect: "deny"}]`，再假设 `[{action: "bash", resource: "ls *", effect: "allow"}, {action: "bash", resource: "*", effect: "deny"}]`。分别推演 `bash` 工具会不会被 `materialize` 从定义列表里删掉？为什么"删掉"不等于"禁止执行"？
3. **实验三：算一次输出裁剪** — 在 `tool-output-store.ts` 读 `MAX_LINES`/`MAX_BYTES` 和 `preview`。假设一个工具返回 5000 行、每行 20 字节的文本，手推：`bound` 会落盘吗？给模型看到的预览大致由哪两段拼成、中间是什么？为什么不只截前 2000 行？
4. **实验四：找全 `read` 的边界防御** — 在 `tool/read.ts` 的 `execute` 里，把所有"防御性检查"列出来（路径逃逸两道、permission.assert、二进制拒绝）。挑符号链接那道（第二次 `FSUtil.contains(root, real)`），说说如果删掉它、只留第一道字符串检查，攻击者能用什么手法读到根目录外的文件。

> **下一章预告**：本章我们看清了工具"怎么被定义、注册、物化、结算、限流"。但工具里最危险的一个——`bash`——是怎么真正把命令跑起来、又怎么把"别让子进程刷爆内存/别让它挂死"这些风险关进笼子的？第 9 章进入命令执行与沙箱，读 `tool/bash.ts` 与进程管理，看 OpenCode 如何执行一条 shell 命令而不被它反噬。
