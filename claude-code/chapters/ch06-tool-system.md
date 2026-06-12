# 第 6 章　统一工具计划：三层契约与 fail-closed 默认

> 模型说「我要调一个工具」，这句话要落地，得先回答一个更根本的问题：在 CCB 里，「工具」到底是什么形状？
> 这一章我们解构工具系统的骨架——一个工具从定义到被主循环接纳，要满足哪三层契约；以及那套「拿不准就往安全里靠」的默认值，是怎么把整个工具生态钉在 fail-closed 的地基上的。

## 6.1 三层契约：协议、宿主、工厂

第 2 章已经勾勒过工具系统的三层依赖链，这一章把它拆开看清楚。三层各司其职：

- **协议层** `packages/agent-tools/src/types.ts`：定义 `CoreTool` 接口——一个工具最少要长成什么样，与任何宿主无关。
- **宿主层** `src/Tool.ts`：在 `CoreTool` 之上扩出宿主才关心的 `Tool` 超集（React 渲染、更丰富的上下文类型）。
- **工厂层** `src/Tool.ts` 的 `buildTool()`：把「部分定义」补齐成「完整工具」，统一注入默认值。

这种分层不是为了好看。协议层注释写得直白：「This defines the protocol-level contract for any tool — independent of React rendering, specific context types, or host infrastructure.」**协议稳定，实现才能自由演化**——builtin 工具、MCP 工具、Skill、子 agent 都围着同一个 `CoreTool` 长出来，不必依赖 CLI 宿主的任何细节。这是依赖倒置原则在 Agent 工程里的标准落点。

## 6.2 CoreTool：一个工具的最小契约

`CoreTool<Input, Output, P, Context>` 是四个泛型参数的接口——输入 schema、输出类型、进度数据类型、执行上下文类型。它的字段按职责分了好几组，读懂这些组就读懂了「CCB 眼里工具的全部维度」。

**身份（Identity）**：`readonly name`、`aliases?`、`searchHint?`。`name` 是工具的唯一标识，`aliases` 是别名（比如第 14 章会看到 `Agent` 工具的别名是 `Task`）。

**Schema**：`readonly inputSchema`（Zod schema）、`inputJSONSchema?`（给 MCP 用的 JSON Schema）、`outputSchema?`。输入用 Zod 定义意味着每次工具调用前都能做结构化校验。

**执行（Execution）**：核心是 `call(args, context, canUseTool, parentMessage, onProgress?)`，返回 `Promise<ToolResult<Output>>`。注意第三个参数 `canUseTool`——工具在执行内部还能再发起权限检查（比如一个工具想调另一个工具时）。

**行为属性（Behavioral properties）**：这一组是工具系统的精华，每个都是影响调度与安全的开关：

```ts
isConcurrencySafe(input): boolean   // 能否与其他工具并发
isEnabled(): boolean                // 是否启用
isReadOnly(input): boolean          // 是否只读
isDestructive?(input): boolean      // 是否破坏性
isOpenWorld?(input): boolean        // 是否触及外部世界
interruptBehavior?(): 'cancel' | 'block'
requiresUserInteraction?(): boolean
```

特别注意这些方法**大多接受 `input` 参数**——也就是说「是否只读」「能否并发」不是工具的静态属性，而是**取决于具体调用参数的动态判断**。`Bash` 工具就是典型：`rm -rf` 不是只读，`ls` 是只读，同一个工具的 `isReadOnly(input)` 会因命令不同返回不同结果（第 9 章详解）。

**权限（Permissions）**：`validateInput?` 返回 `ValidationResult`，`checkPermissions` 返回 `PermissionResult`。后者是第 8 章权限管线的工具侧入口，它的返回类型穷举了三种可能：

```ts
export type PermissionResult =
  | { behavior: 'allow'; updatedInput: Record<string, unknown> }
  | { behavior: 'deny'; message: string }
  | { behavior: 'passthrough' }
```

`allow` 还能改写输入（`updatedInput`），`deny` 必须带可读的拒绝理由（`message`），`passthrough` 表示「我不表态，交给通用权限系统裁决」。这个三态设计是第 8 章的伏笔。

**输出（Output）**：`maxResultSizeChars`(必填) 限定单次工具结果的最大字符数——这是第 11 章上下文预算的源头之一。配套的 `mapToolResultToToolResultBlockParam`、`isResultTruncated?` 处理结果的序列化与截断。

**MCP/分类器标记**：`isMcp?`、`isLsp?`、`toAutoClassifierInput`（第 10 章 YOLO 分类器要用——把工具输入转成分类器能判断的形式）。

这个接口的字段密度本身就是一条信息：**一个「工具」远不止「一个能被调用的函数」**。它携带了并发性、只读性、破坏性、权限、输出预算、可观测描述等十几个维度的元信息，正是这些元信息让 Harness 能在调用前对工具做安全编排，而不是盲目执行。

## 6.3 宿主 Tool：协议的超集

`src/Tool.ts`（813 行）里的 `Tool` 类型是 `CoreTool` 的宿主版超集。注释解释了关系：宿主工具「structurally satisfy this interface because they implement all required fields」——它们结构上满足协议接口，但额外补充了宿主才关心的东西，比如 React/Ink 的 JSX 渲染（`SetToolJSXFn`）、更具体的 `ToolUseContext`（第 160 行，包含 abortController、getAppState、langfuseTrace 等运行时上下文）、`ToolPermissionContext`（第 125 行，一个 `DeepImmutable` 的权限上下文）。

`Tool.ts` 还提供了工具查找的工具函数：`toolMatchesName`（第 369 行，匹配 name 或 alias）和 `findToolByName`（第 379 行）。主循环拿到模型返回的工具名后，就是用它们在工具清单里定位具体工具的——别名机制（`Task` → `Agent`）就在这里生效。

## 6.4 buildTool 与 fail-closed 默认：地基中的地基

整个工具系统最关键的安全设计，藏在 `buildTool()` 工厂（第 778 行起）和它注入的 `TOOL_DEFAULTS`（第 778 行）里。`buildTool` 的作用很朴素——把一个可能缺字段的 `ToolDef` 补齐成完整 `Tool`：

```ts
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

注释明确要求「All tool exports should go through this」——所有工具都必须经过这个工厂，这样默认值只有一处来源，调用方永远不用写 `?.() ?? default`。而 `TOOL_DEFAULTS` 的每一个默认值，都是 fail-closed 原则的字面体现：

```ts
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?) => false,   // 假设不安全
  isReadOnly: (_input?) => false,          // 假设会写
  isDestructive: (_input?) => false,
  checkPermissions: (input, _ctx) =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: (_input?) => '',  // 跳过分类器
  userFacingName: (_input?) => '',
}
```

逐条读这些默认值背后的意图：

- **`isConcurrencySafe → false`**：默认假设一个工具**不能**与别的工具并发跑。要并发必须显式声明安全。代价是保守（少跑点并发），但绝不会因为忘了声明而引入并发竞争。
- **`isReadOnly → false`**：默认假设一个工具**会写**。这意味着它默认被当作「有副作用」对待，享受不到只读工具的快速通道（第 7 章只读工具可以无脑并发、第 9 章只读命令权限更宽松）。要想被当只读，必须显式证明自己只读。
- **`checkPermissions → allow + updatedInput`**：注释标明这是「defer to general permission system」——工具自己不表态放行，把决定权交给通用权限管线（第 8 章）。这里的 `allow` 不是「无条件放行」，而是「我这一层没意见，下一层去管」。
- **`toAutoClassifierInput → ''`**：默认返回空串，等于**跳过 YOLO 分类器**。注释加了一句警告：「security-relevant tools must override」——涉及安全的工具必须自己重写这个方法，否则就被分类器忽略了。

这套默认值的精妙在于：**「忘记声明」永远导向更保守的一侧**。一个工具作者如果什么都不写，他得到的是「不能并发、被当作会写、把权限交给通用系统裁决」的最保守配置。要放宽任何一条，都得显式地、有意识地去 override。安全不靠作者的自觉，而靠默认值的引力——这就是 fail-closed 的工程含义，也是第 1 章七面透镜里第一面的源头。

值得一提的是 `TOOL_DEFAULTS` 的类型设计：注释说 `ToolDefaults = typeof TOOL_DEFAULTS`，用默认对象的**实际形状**当类型，而非接口的严格签名（这样 0 参和全参调用点都能类型检查通过）。`BuiltTool<D>` 在类型层面镜像运行时的 `{ ...TOOL_DEFAULTS, ...def }`，并由「60+ 个工具零类型错误」来证明其正确性。这又是第 17 章「不变量固化进类型」的一例——默认值不只是运行时行为，连「补默认」这个动作都被类型系统盯着。

## 6.5 从契约到清单

有了 `CoreTool` 协议、`Tool` 超集、`buildTool` 工厂，一个工具的「出生」流程就完整了：`packages/builtin-tools` 里的每个工具用 `buildTool({...})` 声明自己（覆盖它需要覆盖的默认值），`src/tools.ts` 把它们 import 进来、按 feature flag 条件装配（第 2 章），最终汇成一份交给主循环的 `Tools` 清单（`readonly Tool[]`）。主循环拿到模型返回的工具名，用 `findToolByName` 定位，校验输入、查权限、再 `call`。

下一章我们就打开这份清单，看具体工具（Bash/Read/Edit/Grep/Glob…）长什么样，以及它们是怎么被 `partition` 成「能并发的」和「必须串行的」两组，由 `StreamingToolExecutor` 并发执行的。

## 本章小结

- 工具系统是三层契约：协议层 `packages/agent-tools`（`CoreTool`）→ 宿主层 `src/Tool.ts`（`Tool` 超集）→ 工厂层 `buildTool()`。协议稳定使 builtin/MCP/Skill/子 agent 能共用同一契约。
- `CoreTool<Input,Output,P,Context>` 按职责分组：身份（name/aliases）、Schema（Zod inputSchema + JSON Schema）、执行（`call`）、行为属性、权限、输出、MCP/分类器标记。
- 行为属性方法（`isReadOnly`/`isConcurrencySafe` 等）大多接受 `input` 参数——只读性、并发安全性是**依赖具体调用参数的动态判断**，不是静态属性（Bash 的 `rm` vs `ls`）。
- `PermissionResult` 三态：`allow`（可改写 `updatedInput`）/`deny`（必带 message）/`passthrough`（不表态，交通用权限系统）。
- `maxResultSizeChars` 是必填字段，限定单次工具结果字符数，是上下文预算的源头之一。
- 宿主 `Tool` 是 `CoreTool` 的结构化超集，补充 JSX 渲染、`ToolUseContext`、`DeepImmutable` 的 `ToolPermissionContext`；`toolMatchesName`/`findToolByName` 支持别名查找。
- `buildTool()` 是所有工具的统一出厂工厂，`TOOL_DEFAULTS` 把默认值集中一处。
- 默认值全部 fail-closed：`isConcurrencySafe → false`、`isReadOnly → false`、`isDestructive → false`、`checkPermissions → 交通用系统`、`toAutoClassifierInput → ''（须被安全工具覆盖）`。「忘记声明」永远导向最保守配置。
- 「补默认」这个动作本身被类型系统固化：`BuiltTool<D>` 在类型层镜像 `{...TOOL_DEFAULTS, ...def}`，由 60+ 工具零类型错误佐证。

## 动手实验

1. **实验一：清点 CoreTool 的维度** — 打开 `packages/agent-tools/src/types.ts` 的 `CoreTool` 接口，把所有字段按注释分组（身份/Schema/执行/行为/权限/输出/标记）抄一遍。数一数「行为属性」有几个，思考每个对调度或安全意味着什么。
2. **实验二：验证默认即 fail-closed** — 打开 `src/Tool.ts` 找到 `TOOL_DEFAULTS`，逐条记录 `isEnabled`/`isConcurrencySafe`/`isReadOnly`/`isDestructive`/`checkPermissions`/`toAutoClassifierInput` 的默认值。对每一个判断：这个默认是偏「安全保守」还是偏「放行」?为什么 `isReadOnly` 默认 `false` 比默认 `true` 更安全？
3. **实验三：找一个 override 默认的真实工具** — 在 `packages/builtin-tools/src/tools/` 里挑一个工具（如 `GlobTool` 或 `GrepTool`），看它的 `buildTool({...})` 定义里显式覆盖了哪些默认方法（比如把 `isReadOnly` 改成返回 `true`）。理解「要放宽必须显式声明」。
4. **实验四：追别名解析** — 在 `src/Tool.ts` 找到 `toolMatchesName` 和 `findToolByName`。再到 `packages/builtin-tools/src/tools/AgentTool/` 找到 `Agent` 工具的 `aliases`（应包含 `Task`）。推演：当模型返回 `tool_use` 名为 `Task` 时，主循环是如何定位到 `AgentTool` 的？

> **下一章预告**：契约讲清楚了，该看实物了。第 7 章我们打开内置工具清单——Bash/Read/Edit/Write/Grep/Glob 各自的设计取舍，以及 `partitionToolCalls` 如何把一批工具调用切成「可并发」与「须串行」两组，`StreamingToolExecutor` 又如何在模型还在输出时就把它们并发跑起来。
