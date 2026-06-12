# 第 7 章　内置工具与并发执行：从 partition 到 StreamingToolExecutor

> 契约是抽象的，工具是具体的。这一章我们打开 CCB 那份装着 50 多个工具的清单，看几把最常用的工具各自做了什么取舍，再深入它最讲究的一处工程——如何在「让 Agent 跑得快」和「别让并发把文件系统搞乱」之间走钢丝。

## 7.1 内置工具一览：每把工具都是一组属性的取舍

`packages/builtin-tools/src/tools/` 下有 50 多个工具目录。日常编码最核心的就那几把：`BashTool`（执行 shell）、`FileReadTool`（读文件）、`FileEditTool`（改文件）、`FileWriteTool`（写文件）、`GlobTool`（按 glob 找文件）、`GrepTool`（按内容搜）、`AgentTool`（派子 agent）、`WebFetchTool`/`WebSearchTool`（联网）、`TodoWriteTool`（任务清单）、`SkillTool`（调技能）。它们都经过第 6 章的 `buildTool()` 出厂，区别只在于各自 override 了哪些默认值。

把第 6 章的「行为属性」落到实物上，立刻能看出每把工具的性格。先看 `GlobTool`（`GlobTool.ts`）：

```ts
isConcurrencySafe() { return true },
isReadOnly() { return true },
maxResultSizeChars: 100_000,
```

它显式声明「我只读、我能并发」，因为找文件不改任何状态——多个 glob 同时跑毫无风险。这是它主动从第 6 章那个 fail-closed 默认（`isConcurrencySafe → false`、`isReadOnly → false`）里「申请放宽」的结果。

再看 `BashTool`（`BashTool.tsx`），它的属性是动态的：

```ts
maxResultSizeChars: 30_000,
isConcurrencySafe(input) {
  return this.isReadOnly?.(input) ?? false;
},
isReadOnly(input) {
  const result = checkReadOnlyConstraints(input, compoundCommandHasCd);
  ...
}
```

注意这里的精妙：Bash 的 `isConcurrencySafe` **直接等于** `isReadOnly`。也就是说一条 bash 命令只有在被判定为只读时才允许并发——`ls`、`cat`、`grep` 可以和别的只读命令并排跑，而 `rm`、`git commit`、`npm install` 这类有副作用的命令一律串行。而 `isReadOnly` 本身又不是常量，是把 `input`（具体命令字符串）丢给 `checkReadOnlyConstraints` 动态算出来的（第 9 章详解）。这就把第 6 章「行为属性取决于具体参数」的论断落到了最典型的实物上：**同一个 BashTool，跑 `ls` 时只读可并发，跑 `rm` 时有副作用须串行。**

各工具的 `maxResultSizeChars` 也透露了设计意图——它们是不同的：`FileReadTool`/`FileEditTool`/`FileWriteTool`/`GlobTool` 都是 `100_000`，`BashTool` 是 `30_000`，`GrepTool` 是 `20_000`。读文件给 10 万字符预算（一个文件可能很长），但 grep 只给 2 万（搜索结果应该是命中的若干行，太多说明 pattern 太宽该收窄），bash 给 3 万（命令输出失控会很大，需要更紧的闸）。这些数字不是拍脑袋，是对「这把工具的正常输出该多大」的判断（第 11 章会看到这些预算如何在主循环里被强制执行）。

工具里还藏着安全细节。`GlobTool` 的 `validateInput` 里有一段注释带 `SECURITY:` 标记——遇到 UNC 路径（`\\` 或 `//` 开头）直接跳过文件系统操作，注释解释是「prevent NTLM credential leaks」（防止 Windows 凭据泄露）。这是第 8 章「边界不信任外部输入」原则在单个工具里的微观体现。

## 7.2 并发的核心难题：保序与隔离

模型一轮可能同时要求调好几个工具（比如「读 A、读 B、读 C」）。全串行太慢，全并发会出事——如果「写文件 X」和「读文件 X」并发，结果就不确定了。CCB 的解法是一条朴素但严格的规则：**只读工具可以扎堆并发，非只读工具必须独占串行，且整体保持模型给出的顺序。**

这条规则的载体是 `src/services/tools/toolOrchestration.ts` 里的 `partitionToolCalls`（第 106 行）。它的注释把规则写得明明白白：

```
Partition tool calls into batches where each batch is either:
1. A single non-read-only tool, or
2. Multiple consecutive read-only tools
```

实现是一个 `reduce`，逐个扫描工具调用，把**连续的**并发安全工具合并进同一个 batch，遇到非并发安全的就单独开一个 batch：

```ts
if (isConcurrencySafe && acc[acc.length - 1]?.isConcurrencySafe) {
  acc[acc.length - 1]!.blocks.push(toolUse)   // 并入上一个并发批
} else {
  acc.push({ isConcurrencySafe, blocks: [toolUse] })  // 自己开一批
}
```

关键词是「**连续（consecutive）**」。假设模型要求 `[读A, 读B, 写C, 读D]`，分区结果是 `[{并发: 读A,读B}, {串行: 写C}, {串行: 读D}]`——读 D 虽然只读，但它在写 C 之后，不能跳到前面和 A、B 一起跑，否则就破坏了「写 C 完成后再读 D」可能隐含的依赖。**保序优先于并发**，这是正确性高于性能的体现。

`partitionToolCalls` 还有一处 fail-closed 细节：判定 `isConcurrencySafe` 时套了 `try/catch`：

```ts
try {
  return Boolean(tool?.isConcurrencySafe(parsedInput.data))
} catch {
  // 如果 isConcurrencySafe 抛错（如 shell-quote 解析失败），保守当作不安全
  return false
}
```

注释直说——一旦 `isConcurrencySafe` 抛异常（比如 Bash 命令里有解析不了的 shell 引号），就**保守地当作不可并发**。又是「拿不准就往安全靠」。

分好区后，`runTools`（第 20 行）遍历每个 batch：并发批走 `runToolsConcurrently`（第 169 行），串行批走 `runToolsSerially`（第 133 行）。并发批的并发上限由 `getMaxToolUseConcurrency()`（第 9 行）控制，默认 `10`，可被环境变量 `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` 覆盖。

## 7.3 StreamingToolExecutor：在模型还没说完时就开跑

`partitionToolCalls` 解决的是「拿到全部工具调用后怎么分批跑」。但第 4 章说过，CCB 更激进——它在模型**流式输出还没结束**时就开始执行工具。承载这个能力的是 `src/services/tools/StreamingToolExecutor.ts`。

它维护一个工具队列，每个工具有四种状态：`'queued' | 'executing' | 'completed' | 'yielded'`（`ToolStatus`）。主循环每从模型流里解析出一个 `tool_use` 块，就调 `addTool(block, assistantMessage)`（第 90 行）把它入队；入队时同样用 `inputSchema.safeParse` + `isConcurrencySafe` 算出它的并发安全性。

调度的核心是 `canExecuteTool`（第 156 行）和 `processQueue`（第 167 行）。`canExecuteTool` 的逻辑精炼地表达了和 `partition` 一致的并发约束：

```ts
private canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executingTools = this.tools.filter(t => t.status === 'executing')
  return (
    executingTools.length === 0 ||
    (isConcurrencySafe && executingTools.every(t => t.isConcurrencySafe))
  )
}
```

翻译过来：要么**当前没有任何工具在跑**（随便起），要么**我自己并发安全且正在跑的全都并发安全**（只读扎堆）。只要有一个非只读工具在跑，谁也别想插进来。

`processQueue` 则负责保序：

```ts
for (const tool of this.tools) {
  if (tool.status !== 'queued') continue
  if (this.canExecuteTool(tool.isConcurrencySafe)) {
    await this.executeTool(tool)
  } else {
    // 还不能跑，且为了维持非并发工具的顺序，到此为止
    if (!tool.isConcurrencySafe) break
  }
}
```

它顺着队列扫，能跑的就起；遇到一个不能跑的非并发工具就 `break`——绝不跳过它去跑后面的工具，从而严格维持顺序。这和 `partition` 的「连续只读才合批」是同一条规则的两种实现：一个面向「已知全量」的批处理，一个面向「流式增量」的实时调度。

`StreamingToolExecutor` 还处理了大量边界情况。`discard()`（第 73 行）在 fallback 发生时被调用——它把 `discarded` 置位、abort 正在跑的工具子进程（如 Bash spawn 出来的进程），让被丢弃的执行器和它缓冲的结果能被垃圾回收，这正是第 4、5 章 fallback 清理逻辑的执行侧。`createSyntheticErrorMessage`（第 180 行）为三种异常情况合成工具结果：`sibling_error`（同批某个工具出错）、`user_interrupted`（用户按 ESC 拒绝，UI 显示「User rejected edit」而非「Error」）、`streaming_fallback`。这些都是「让 Agent 在真实世界的中断与失败里不崩」的工程量。

## 7.4 两条路径，一致的安全契约

CCB 里其实存在两套工具执行入口——批处理式的 `runTools` 和流式的 `StreamingToolExecutor`。它们服务于不同时机（拿到完整响应 vs 边流边跑），但都严格遵守同一条并发契约：**只读可并发、非只读独占、顺序不可乱、判定出错就当不安全**。这种「同一规则、两处实现、互相印证」的设计，本身就是对正确性的双重保险。

把这一章和上一章合起来看：第 6 章定义了「工具是什么形状、默认有多保守」，第 7 章展示了「这些形状属性如何被并发调度真正用起来」。`isReadOnly`、`isConcurrencySafe` 这两个看似不起眼的布尔方法，正是连接「契约」与「执行」的枢纽——它们在第 6 章被声明，在第 7 章决定了一批工具是飞快并发还是老实排队。而 Bash 的 `isReadOnly` 到底怎么算出来,引出了下一部分的主题：安全。

## 本章小结

- `packages/builtin-tools/src/tools/` 有 50+ 工具，核心是 Bash/Read/Edit/Write/Glob/Grep/Agent/WebFetch/TodoWrite/Skill；都经 `buildTool()` 出厂，区别在各自 override 的默认值。
- `GlobTool` 显式声明 `isReadOnly → true`、`isConcurrencySafe → true`，主动从 fail-closed 默认申请放宽；`maxResultSizeChars` 因工具而异：Read/Edit/Write/Glob=100_000、Bash=30_000、Grep=20_000，反映对各工具「正常输出多大」的判断。
- `BashTool` 的 `isConcurrencySafe(input)` 直接等于 `isReadOnly(input)`，而 `isReadOnly` 由 `checkReadOnlyConstraints` 按命令动态计算——`ls` 只读可并发，`rm` 有副作用须串行。
- 工具内藏安全细节，如 `GlobTool.validateInput` 对 UNC 路径（`\\`/`//`）跳过文件系统操作以防 NTLM 凭据泄露。
- 并发规则：只读工具可扎堆并发，非只读必须独占串行，整体保持模型给出的顺序（保序优先于并发）。
- `partitionToolCalls`（reduce 实现）把**连续的**并发安全工具合进同一 batch，非并发的单独成批；`[读A,读B,写C,读D]` → `[{读A,读B}],[写C],[读D]`。
- 并发安全判定套 `try/catch`，`isConcurrencySafe` 抛错（如 shell 引号解析失败）即保守当作不可并发；并发上限 `getMaxToolUseConcurrency()` 默认 10。
- `StreamingToolExecutor` 在模型流式输出未结束时即开跑：工具状态机 `queued/executing/completed/yielded`，`canExecuteTool` 实现「无人在跑 或 自己只读且在跑的全只读」，`processQueue` 遇非并发工具即 `break` 以保序。
- `discard()` 在 fallback 时 abort 子进程并使缓冲结果可回收；`createSyntheticErrorMessage` 为 sibling_error/user_interrupted/streaming_fallback 合成工具结果。
- 批处理 `runTools` 与流式 `StreamingToolExecutor` 是同一并发契约的两处实现，互为正确性保险。

## 动手实验

1. **实验一：给工具的性格建档** — 在 `packages/builtin-tools/src/tools/` 挑 5 把工具（Glob/Grep/Read/Edit/Bash），分别记录它们的 `isReadOnly`、`isConcurrencySafe`、`maxResultSizeChars`。做一张表，标出哪些是静态返回、哪些依赖 `input` 动态计算。
2. **实验二：手动分区** — 假设模型一轮要求 `[Read a, Grep b, Write c, Read d, Glob e]`，依据 `partitionToolCalls` 的规则（连续只读合批、保序）手算分区结果。再读 `toolOrchestration.ts` 的 reduce 验证你的答案。
3. **实验三：理解 Bash 的并发=只读** — 打开 `BashTool.tsx`，确认 `isConcurrencySafe(input)` 返回的就是 `isReadOnly(input)`。思考：为什么 Bash 不能像 Glob 那样无条件 `isConcurrencySafe() { return true }`？举一个并发会出错的双命令例子。
4. **实验四：读流式调度器的两条不变量** — 在 `StreamingToolExecutor.ts` 读 `canExecuteTool` 和 `processQueue`。用自己的话写出它维护的两条不变量（并发隔离 + 顺序保持），并解释 `processQueue` 里那句 `if (!tool.isConcurrencySafe) break` 为什么不能改成 `continue`。

> **下一章预告**：工具要跑起来，得先过权限这一关。第 8 章我们进入安全与护栏部分，解构 `src/utils/permissions` 的审批管线——`allow`/`deny`/`ask`/`passthrough` 四种行为、六种权限模式、`Tool(content)` 规则格式，以及「约束只能收紧」的多来源配置合并，看 CCB 如何在 Agent 动手前踩下刹车。
