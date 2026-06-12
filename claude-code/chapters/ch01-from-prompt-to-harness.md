# 第 1 章　从 Prompt 到 Harness：为什么要解构 Claude Code

> 同一个大模型，套上不同的工程外壳，会长成性格迥异的 Agent。
> 这层外壳，就是本书要解构的对象——Harness。

## 1.1 Agent = Model + Harness

如果你只调用过一次 LLM 的 chat 接口，你拿到的是一段文本。如果你把这段文本里的「我要读 `foo.ts`」真的变成一次文件读取、把读到的内容再喂回模型、让它接着说——你就跨过了从「Chat」到「Agent」的那条线。跨过这条线的全部工程，就是 Harness。

可以用一个粗暴但准确的等式概括：

```
Agent = Model + Harness
```

Model 提供「想做什么」的能力，Harness 决定「能做什么、看到什么、被拦在哪、错了怎么办」。模型是发动机，Harness 是底盘、变速箱、刹车和安全气囊。一台再强的发动机，没有刹车也不能上路。

本书要解构的 CCB（claude-code-best/claude-code），是对 Anthropic 官方 Claude Code CLI 的逆向还原。它的价值恰恰在于「还原」二字：官方的 Claude Code 以高度压缩的产物发布，它的工程决策——双层主循环、AST 级命令安全门、多级上下文压缩——都藏在 minify 后的代码里难以辨认。而 CCB 把这套骨架用可读的 TypeScript 重新写了一遍，让我们第一次能逐行去读「一个生产级编码 Agent 到底是怎么搭起来的」。

## 1.2 为什么同一个模型表现天差地别

你一定有过这种体验：同一个 Claude 模型，在某个产品里像个能干的工程师，在另一个产品里却像个只会复读的实习生。差距不在模型，在 Harness。

举一个本书后面会反复出现的例子。当用户让 Agent 执行 `rm -rf build/` 时，一个朴素的 Agent 会直接把这串字符丢给 shell。而 CCB 会做这样一连串事情（细节见第 9 章）：先用 tree-sitter 把命令解析成语法树，检查里面有没有命令替换、子 shell、危险展开（`DANGEROUS_TYPES` 列表）；再把每个子命令的 argv 拿出来做语义检查（`checkSemantics`）；判断它是不是「只读」命令（`checkReadOnlyConstraints`）；查 allow / deny 权限规则；决定要不要丢进沙箱（`shouldUseSandbox`）；甚至在 Auto 模式下，把这个动作交给一个专门会「拒绝」的分类器模型去裁决（第 10 章）。

同一个 LLM 输出的同一串命令，在朴素 Agent 里直接执行，在 CCB 里要穿过至少五道闸门。表现的差异，全在这些闸门里。

## 1.3 Harness 的几个核心职责

通读 CCB 的源码后，可以把 Harness 的职责归纳成几条主线，它们也正好对应本书的章节弧线：

| 职责 | 在 CCB 里的体现 | 本书章节 |
|---|---|---|
| **编排** | `src/query.ts` 的 `queryLoop` 双层 `while` 循环 | 第 4 章 |
| **接入** | `src/services/api/claude.ts` 的 provider 抽象、重试、Fallback | 第 5 章 |
| **能力** | `packages/agent-tools` + `src/Tool.ts` 的统一工具计划 | 第 6、7 章 |
| **约束** | `src/utils/permissions` 权限管线 + BashTool AST 安全门 | 第 8、9、10 章 |
| **上下文** | snip / microcompact / autocompact 多级压缩 | 第 11、12 章 |
| **记忆** | `CLAUDE.md` 层级加载 + `src/memdir` 自动记忆 | 第 13 章 |
| **扩展** | AgentTool / Skill / MCP / Hook / 插件 | 第 14、15、16 章 |
| **可信** | 配置级联、JSONL 持久化、`tengu_` 遥测、DI 测试 | 第 17 章 |

注意这张表里「约束」占了三章。这不是凑数——在 CCB 里，与「安全」相关的代码量远超「让 Agent 跑起来」的代码量。一个朴素 Agent 可能 200 行就能跑通主循环，但要让它**敢**在你的真实文件系统和 shell 上跑，需要的工程量是另一个数量级。这正是「生产级」和「Demo」的分水岭。

## 1.4 本书方法论：源码是唯一真理

Agent 领域充斥着「一般来说 Agent 会怎么做」的二手叙述。本书拒绝这种叙述。我们的每一个论断，都能在 CCB 的某个具体文件、某个具体函数、某个具体常量里找到对应。

举例来说，本书不会写「Claude Code 会在上下文快满时自动压缩」这种含糊的话，而会写：CCB 在 `src/services/compact/autoCompact.ts` 里定义了 `getAutoCompactThreshold(model)`，它等于 `getEffectiveContextWindowSize(model)` 减去 `getAutocompactBufferTokens`（默认 `AUTOCOMPACT_BUFFER_TOKENS = 13_000`，但当上下文窗口 ≥ 800,000 时变成 50,000）；触发判定用的是 `tokenCountWithEstimation(messages) - snipTokensFreed` 与这个阈值比较；压缩失败 3 次（`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`）会熔断。

这种「读得到的具体」，才是值得你花时间的内容。也因此，本书每章末尾都附有「动手实验」——它们不是练习题，而是邀请你亲手回到源码里，验证（甚至证伪）书中的每一个结论。

读源码时，有几个反复出现的「工程透镜」值得你主动去找，它们会在第 17 章被系统提炼：

1. **默认即关闭（fail-closed）**：不确定时选更安全的一侧。CCB 的 `buildTool` 默认 `isReadOnly: false` / `isConcurrencySafe: false`；命令安全门解析失败时 fail-closed 拦截。
2. **约束只能收紧**：多来源配置合并后只会更严。`policySettings` 永远赢，agent 工具子集是逐层做减法。
3. **错误即文档**：校验失败时给出可操作的修复建议，而不是抛栈。
4. **边界不信任外部输入**：路径、命令、模型输出都先净化再使用。
5. **留结构去内容**：脱敏 / 压缩时保留形状、抹掉敏感值。
6. **不变量固化进类型和测试**：关键契约用字面量类型和精确断言锁死。
7. **可信度即流水线**：DI + 隔离 + 断言 + 脱敏叠成一条链，而非单点开关。

带着这七面透镜读下去，你会发现 CCB 的每一处「啰嗦」都不是偶然。

## 本章小结

- **Agent = Model + Harness**：模型决定「想做什么」，Harness 决定「能做什么、看到什么、被拦在哪、错了怎么办」。
- 同一个模型在不同产品里表现迥异，差距不在模型权重，而在外层 Harness 的工程质量。
- CCB 是官方 Claude Code 的逆向还原，把藏在压缩产物里的工程决策重新摊成可读源码，是研究 Claude Code 内部机制的难得标本。
- Harness 的核心职责可归纳为：编排、接入、能力、约束、上下文、记忆、扩展、可信——本书按这条线自外向内展开。
- 在 CCB 里，「约束 / 安全」相关的代码量远超「让 Agent 跑起来」的代码量，这是「生产级」区别于「Demo」的根本标志。
- 本书唯一的方法论铁律：**以真实源码为唯一真理来源**，每个论断都落到具体文件、函数、常量、字面量。
- 七面工程透镜（fail-closed、约束收紧、错误即文档、边界不信任、留结构去内容、不变量入类型与测试、可信度即流水线）将贯穿全书。

## 动手实验

1. **实验一：丈量「约束」的体量** — clone CCB 仓库后，统计 `src/utils/permissions/` 和 `packages/builtin-tools/src/tools/BashTool/` 两个目录的代码行数（`find ... -name '*.ts' | xargs wc -l`），再统计 `src/query.ts` 单文件行数。对比「安全」与「主循环」的代码量比例，亲手验证 1.3 节的论断。
2. **实验二：找到那条「跨过 Chat 的线」** — 打开 `src/query.ts`，搜索 `needsFollowUp`。读懂这个布尔变量是如何决定「模型这一轮调了工具，所以还要再转一圈」的。这一行就是「Agent」区别于「Chat」的代码级分界。
3. **实验三：验证 fail-closed 默认** — 打开 `src/Tool.ts`，找到 `TOOL_DEFAULTS`。逐个记录 `isEnabled` / `isConcurrencySafe` / `isReadOnly` / `checkPermissions` 的默认值，判断每一个默认值是偏「安全」还是偏「放行」。
4. **实验四：建立你的「透镜」笔记** — 准备一个空文档，把 1.4 节的七面透镜抄下来。在后续每读一章时，记录你在源码里找到的对应实例（确切的文件名 + 函数名 + 字面量）。读到第 17 章时，回头对照本书的提炼。

> **下一章预告**：透镜备好了，标本还没摊开。第 2 章我们将俯瞰 CCB 的整体工程结构——monorepo 怎么切分、`src/` 与 `packages/` 各管什么、一次请求从入口到工具结果要穿过哪些区域，建立一张贯穿全书的地图。
