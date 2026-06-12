# 第 2 章　CCB 全景：一个逆向还原的 Claude Code 长什么样

> 透镜已经备好，标本还没摊开。在拆解任何一个零件之前，先要知道整台机器有几个舱、每个舱装什么、一次请求要穿过哪几道门。
> 本章画的就是这张地图——它会贯穿全书，后面每一章都是在这张地图上放大某一块。

## 2.1 先认门牌：这是一个 Bun + TypeScript 的 monorepo

clone 下来第一眼，CCB 的仓库根目录就把它的技术身份写在了脸上：`bunfig.toml`、`bun.lock`、`biome.json`、`vite.config.ts`、`tsconfig.base.json`。没有 `package-lock.json`，没有 `jest.config.js`——这是一个用 **Bun** 当运行时和测试器、用 **Vite** 打包、用 **Biome** 做 lint/format 的纯 TypeScript 工程。`package.json` 里的 `name` 是 `claude-code-best`，`version` 是 `2.6.13`，对齐的是 Anthropic 官方 Claude Code 的版本号。

规模上它不是玩具：`src/` 加上 `.tsx`，源码文件超过 2200 个。这个体量本身就是一条信息——一个「能在你真实文件系统和 shell 上跑」的生产级 Agent，绝大部分代码不是在「让模型说话」，而是在「让模型动手之前先想清楚边界」。第 1 章那张职责表里「约束」占三章，正是这个比例在目录结构上的投影。

仓库被切成两大块：根目录的 `src/`（宿主应用，CLI 本体），和 `packages/`（可独立复用的子包）。理解这两块的分工，是读懂整个工程的第一把钥匙。

## 2.2 packages/：协议与能力，被刻意抽离的「可复用核」

`packages/` 目录下并排躺着十几个子包，每个都有自己的 `package.json` 和 scope 前缀 `@claude-code-best/`。它们大致分三类。

**第一类是工具系统的协议与实现，这是全书的主干。** 三个包构成一条清晰的依赖链：

- `@claude-code-best/agent-tools`：**协议层**。它的 `src/types.ts` 第一行注释写得很直白——「Protocol-level types, independent of any host framework」。这里定义了 `CoreTool<Input, Output, P, Context>` 这个核心接口，以及 `ToolInputJSONSchema`、`ToolProgress`、`PermissionResult` 等纯类型。它不依赖任何宿主框架，是「一个工具最少要长成什么样」的契约。
- `@claude-code-best/builtin-tools`：**实现层**。`src/tools/` 下是一个个具体工具的目录——`BashTool`、`FileReadTool`、`FileEditTool`、`GlobTool`、`GrepTool`、`AgentTool`、`WebFetchTool`、`SkillTool`……目录数超过 50 个。每个目录是一个工具的完整实现。
- `@claude-code-best/mcp-client`：**MCP 协议客户端**。`src/` 下有 `connection.ts`、`discovery.ts`、`execution.ts`、`manager.ts`、`sanitization.ts`，是一套与具体传输无关的纯协议实现（第 15 章会看到，真实传输由 `src/services/mcp/` 注入进来）。

为什么要把工具协议单独抽成一个包？因为协议一旦稳定，实现可以独立演化，子 agent、MCP server、Skill 都能围着同一个 `CoreTool` 契约长出来，而不必依赖 CLI 宿主的任何细节。这是「依赖倒置」在 Agent 工程里的标准用法，第 6 章会专门解构这条三层契约。

**第二类是带 `-napi` 后缀的原生加速包**：`color-diff-napi`、`image-processor-napi`、`modifiers-napi`、`url-handler-napi`、`audio-capture-napi`。这些是用 native 代码（N-API）实现的性能敏感模块——彩色 diff、图像处理、键盘修饰键、URL 解析、音频采集。它们的存在说明 CCB 不只是一个文本 Agent，还认真处理了终端 UI 的渲染性能和多模态输入。

**第三类是周边能力包**：`acp-link`（Agent Client Protocol 对接）、`remote-control-server`（远程控制）、`@ant/`（一组以 `claude-for-chrome-mcp`、`computer-use-mcp`、`model-provider`、`ink` 为代表的内部包，`ink` 是终端 UI 框架的定制分支）。这些是把 Agent 接到浏览器、桌面、模型 provider 上的桥梁。

## 2.3 src/：宿主应用，按「一次请求的生命周期」铺开

如果说 `packages/` 是零件库，`src/` 就是把零件组装成整车的车间。它的子目录非常多，但只要按「一次请求从哪进、到哪出」的顺序去看，立刻就有了秩序。下面这张表把 `src/` 的关键区域和它在请求链路上的位置对应起来：

| 区域 | 关键文件/目录 | 在链路上的职责 | 本书章节 |
|---|---|---|---|
| 进程入口 | `src/entrypoints/cli.tsx`、`src/main.tsx` | 解析参数、动态 import、分流交互/headless | 第 3 章 |
| 双驱动 | `src/screens/REPL.tsx`、`src/cli/print.ts`、`src/QueryEngine.ts` | 交互 REPL vs headless 引擎 | 第 3 章 |
| 主循环 | `src/query.ts`、`src/query/` | `queryLoop` 双层 while、turn 生命周期 | 第 4 章 |
| 模型网关 | `src/services/api/` | provider 抽象、流式、重试、Fallback | 第 5 章 |
| 工具计划 | `src/Tool.ts`、`src/tools.ts` | 宿主 Tool 契约、工具注册与装配 | 第 6、7 章 |
| 权限与安全 | `src/utils/permissions/`、`src/utils/bash/` | 审批管线、AST 命令安全门 | 第 8、9、10 章 |
| 上下文压缩 | `src/services/compact/`、`src/services/contextCollapse/` | snip / microcompact / autocompact | 第 11、12 章 |
| 记忆 | `src/utils/claudemd.ts`、`src/memdir/` | CLAUDE.md 层级、自动记忆 | 第 13 章 |
| 扩展 | `src/skills/`、`src/services/mcp/`、`src/plugins/`、`src/hooks/` | Skill / MCP / 插件 / Hook | 第 14、15、16 章 |
| 配置与可观测 | `src/services/analytics/`、`src/utils/sessionStorage.ts` | settings 级联、JSONL、`tengu_` 遥测 | 第 17 章 |

`src/Tool.ts`（813 行）是 `packages/agent-tools` 协议在宿主侧的「超集」：它在协议 `CoreTool` 之上补齐了宿主才关心的字段，并提供 `buildTool()` 工厂（第 6 章会细看它的 fail-closed 默认）。而 `src/tools.ts`（419 行）是「装配车间」——它把 `packages/builtin-tools` 里的一个个工具 import 进来，组装成最终交给主循环的工具清单。

值得注意的是 `src/services/` 这个目录极其庞大，光子目录就有几十个：`api`（模型网关）、`compact`（压缩）、`mcp`（MCP 真实传输）、`analytics`（遥测）、`auth`/`oauth`（鉴权）、`extractMemories`（记忆抽取）、`skillLearning`/`skillSearch`（技能学习与检索）……这印证了一件事：Agent 的「智能」很薄，「服务」很厚。模型调用只是其中一个 service，它周围环绕着鉴权、限流、压缩、记忆、遥测一大圈支撑系统。

## 2.4 条件装配：同一份源码，能编译出好几种 Agent

读 `src/tools.ts` 的开头，会看到一个非常关键的工程手法——**工具不是静态全装的，而是按编译期/运行期标志条件装配的**。比如：

```ts
const SleepTool =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('@claude-code-best/builtin-tools/tools/SleepTool/SleepTool.js')
        .SleepTool
    : null
const MonitorTool = feature('MONITOR_TOOL')
  ? require('@claude-code-best/builtin-tools/tools/MonitorTool/MonitorTool.js')
      .MonitorTool
  : null
```

这里的 `feature('X')` 不是普通的运行时判断，而是**编译期的 dead-code elimination 标记**（第 17 章会拆解 `scripts/vite-plugin-feature-flags.ts` 如何把它替换成字面量并消除死代码）。再加上 `process.env.USER_TYPE === 'ant'` 这类运行时分支——`REPLTool`、`SuggestBackgroundPRTool` 只有内部用户（`ant`）才会装上。

这意味着：**同一份 CCB 源码，按不同的 feature 组合和用户类型，能编译/运行出形态不同的 Agent**。一个面向公网用户的精简版、一个面向内部的全功能版、一个专做 proactive 任务的变体，共享同一套主循环和工具协议，只在「装哪些工具、开哪些能力」上分叉。这正是第 1 章「同一个模型，套上不同外壳长成不同 Agent」那句话在代码层面的字面体现——连「外壳」本身都是可配置装配的。

## 2.5 一次请求的全景流向

把前面几节拼起来，一次真实请求穿过 CCB 的路线是这样的（这也是本书自外向内的阅读主线）：

```
进程入口 cli.tsx → 动态 import main.tsx
  → 双驱动分流（交互 REPL.tsx / headless QueryEngine.ts）
    → 主循环 query.ts: queryLoop 双层 while
      → 上下文预算与压缩（applyToolResultBudget → snip → microcompact → autocompact）
        → 模型流式调用 services/api/claude.ts（provider 抽象 + 重试 + Fallback）
          → 解析模型输出里的 tool_use
            → 权限审批 utils/permissions + 命令安全门 utils/bash/ast.ts
              → 工具执行（builtin-tools，按并发安全性 partition 后并发跑）
                → 工具结果回填消息历史 → 判定继续还是终止
                  → （贯穿）记忆 / 配置 / JSONL 持久化 / tengu_ 遥测
```

这张图里的每一个箭头，后面都有一整章去放大。本章你只需要记住三件事：**协议在 `packages/`、装配在 `src/`、一切围着 `query.ts` 的主循环转**。带着这张地图，我们就可以从第一道门——进程入口——开始往里走了。

## 本章小结

- CCB 是一个 **Bun + TypeScript + Vite + Biome** 的 monorepo，`package.json` name 为 `claude-code-best`、version `2.6.13`，源码文件 2200+，对标官方 Claude Code。
- 仓库分两大块：`packages/`（可复用的协议与能力子包）和 `src/`（CLI 宿主应用）。
- 工具系统是一条三层依赖链：`agent-tools`（协议层，`CoreTool` 接口）→ `builtin-tools`（50+ 个具体工具实现）→ 宿主侧 `src/Tool.ts` 的超集契约与 `src/tools.ts` 的装配。
- `packages/` 还包含一组 `-napi` 原生加速包（彩色 diff、图像、音频等）和周边桥接包（`acp-link`、`@ant/computer-use-mcp` 等），说明 CCB 是多模态、跨端的。
- `src/services/` 极其庞大（几十个子目录），印证「Agent 智能很薄、支撑服务很厚」：模型调用只是众多 service 之一，周围环绕鉴权、限流、压缩、记忆、遥测。
- `src/tools.ts` 用 `feature('X')`（编译期 DCE）和 `process.env.USER_TYPE`（运行期）**条件装配**工具——同一份源码能产出形态不同的 Agent 变体。
- 一次请求的全景流向：入口 → 双驱动 → 主循环 → 上下文压缩 → 模型网关 → 工具审批与安全 → 工具并发执行 → 结果回填与继续判定，全程贯穿记忆/配置/持久化/遥测。

## 动手实验

1. **实验一：丈量两大块的体量** — clone CCB 后，分别统计 `packages/` 和 `src/` 下 `.ts`/`.tsx` 文件数与总行数（`find packages -name '*.ts' | xargs wc -l | tail -1` 与 `find src -name '*.ts' -o -name '*.tsx' | xargs wc -l | tail -1`）。再单独统计 `src/services/` 的占比，亲手验证 2.3 节「服务很厚」的论断。
2. **实验二：数清三层工具链** — 打开 `packages/agent-tools/src/types.ts` 找到 `CoreTool` 的字段；再 `ls packages/builtin-tools/src/tools/` 数出具体工具目录数；最后打开 `src/tools.ts` 看它 import 了哪些工具、又装配成了什么。画出这三层的依赖箭头。
3. **实验三：找出所有「条件装配」的工具** — 在 `src/tools.ts` 里搜索 `feature(` 和 `USER_TYPE`，列出每一个被条件装配的工具（如 `SleepTool`、`MonitorTool`、`REPLTool`），记录它依赖哪个标志。思考：把这些标志全开，Agent 会多出哪些能力？
4. **实验四：验证 monorepo 的 scope 边界** — 打开 `packages/agent-tools/package.json`、`packages/builtin-tools/package.json`、`packages/mcp-client/package.json`，确认它们的 `name` 都以 `@claude-code-best/` 开头；再在 `src/tools.ts` 里找到对应的 import 路径，验证宿主是通过包名而非相对路径引用这些子包的。

> **下一章预告**：地图画好了，现在从第一道门走进去。第 3 章我们将解构 CCB 的进程入口——`cli.tsx` 如何解析参数、为什么用动态 import、`--version` 为何能走快速路径，以及交互 REPL 与 headless QueryEngine 这「双驱动」是怎么从同一个入口分流出去的。
