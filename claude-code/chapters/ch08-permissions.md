# 第 8 章　权限与审批流：在动手前踩刹车

> 模型说要跑 `rm -rf node_modules`，Harness 不能直接照办，也不能每次都弹窗烦死用户。它需要一套既安全又不碍事的规则系统，来决定哪些动作放行、哪些拒绝、哪些得问一句。
> 这一章我们解构 `src/utils/permissions`——这是整本书「约束」三章的第一章，也是 CCB 代码量最密集的区域之一。

## 8.1 三种行为、五种模式：权限系统的两个坐标轴

权限系统的最底层是两个正交的概念。第一个是**行为（behavior）**，定义在 `src/types/permissions.ts`：

```ts
export type PermissionBehavior = 'allow' | 'deny' | 'ask'
```

只有三种结果：放行、拒绝、问用户。注意这和第 6 章工具自身的 `PermissionResult`（`allow`/`deny`/`passthrough`）不同——工具层的 `passthrough` 意为「我不表态」,到了权限系统这一层最终必须收敛成 `allow`/`deny`/`ask` 之一。

第二个坐标轴是**权限模式（mode）**，它决定整个会话默认有多宽松：

```ts
export const EXTERNAL_PERMISSION_MODES = [
  'acceptEdits', 'bypassPermissions', 'default', 'dontAsk', 'plan',
] as const
```

这五种是「用户可寻址」的外部模式：`default`（标准，该问就问）、`plan`（计划模式，只读不动手）、`acceptEdits`（自动接受文件编辑）、`dontAsk`（尽量不问）、`bypassPermissions`（绕过权限——危险，有专门的 `bypassPermissionsKillswitch.ts` 兜底禁用）。它们可以通过 `settings.json` 的 `defaultMode`、`--permission-mode` CLI flag、或会话恢复来设置。

内部还多两种：

```ts
export type InternalPermissionMode = ExternalPermissionMode | 'auto' | 'bubble'
```

`auto` 是第 10 章主角——把放行权交给一个分类器模型。注释特别说明 `auto` 永远可用，但当 `TRANSCRIPT_CLASSIFIER` 关闭时分类器不可用，auto 会回落到「弹窗询问」。这又是 fail-closed：能力不可用时退回更保守的行为。

整套模式用 `as const satisfies readonly PermissionMode[]` 锁死——这是第 17 章「不变量固化进类型」的又一例，新增模式必须在类型里登记，写错一个字符串编译就报错。

## 8.2 规则的形状：`Tool(content)` 与它的来源

具体的放行/拒绝规则用 `PermissionRule` 表达，它由三部分组成：

```ts
export type PermissionRule = {
  source: PermissionRuleSource
  ruleBehavior: PermissionBehavior
  ruleValue: { toolName: string; ruleContent?: string }
}
```

`ruleValue` 就是大家在配置里见到的 `Tool(content)` 格式——比如 `Bash(npm run test:*)` 表示「允许跑以 `npm run test` 开头的 bash 命令」，`Read(/etc/**)` 表示某种读规则，`WebFetch` 不带括号表示整个工具。解析这个格式的是 `permissionRuleParser.ts`。

而 `source`（规则来源）是理解「约束只能收紧」原则的关键，它有明确的枚举：

```ts
export type PermissionRuleSource =
  | 'userSettings' | 'projectSettings' | 'localSettings'
  | 'flagSettings' | 'policySettings'
  | 'cliArg' | 'command' | 'session'
```

这些来源不是平等的。`policySettings`（企业策略，通常由 MDM/组织下发）拥有最高优先级——它定义的规则任何下层都无法推翻。`deletePermissionRule` 里能看到佐证：当 `rule.source === 'policySettings'` 时，删除会被特殊对待（第 1356 行附近）。这就是「约束只能收紧」：多来源配置合并后，结果只会比任一来源更严，组织管理员设的红线，用户的本地 settings 改不动。

## 8.3 审批的优先级流水线：deny 永远先行

权限判定的核心入口是 `hasPermissionsToUseTool`（第 473 行，类型是 `CanUseToolFn`——还记得第 6 章 `call` 方法的第三个参数吗？就是它）。它内部委托给一系列检查，其中规则判定的主体是 `checkRuleBasedPermissions`（第 1092 行）。这个函数的代码顺序，**就是权限决策的优先级**，读懂这个顺序就读懂了整套审批流。它严格按以下次序短路返回：

**1a. 工具被整体 deny。** 第一件事就是查 deny 规则：

```ts
const denyRule = getDenyRuleForTool(appState.toolPermissionContext, tool)
if (denyRule) {
  return { behavior: 'deny', decisionReason: { type: 'rule', rule: denyRule },
    message: `Permission to use ${tool.name} has been denied.` }
}
```

**deny 在所有判断之前。** 这是安全系统的铁律——拒绝优先于一切，一旦命中拒绝规则，后面的 allow 规则、ask 规则、工具自身判断全都不再有机会翻盘。这保证了「禁止」是不可被绕过的。

**1b. 工具有 ask 规则。** 接着查 ask 规则。这里有一个有意思的例外：如果工具是 Bash 且满足「沙箱已启用 + 开启了沙箱内自动放行 + 这条命令会被沙箱化」（`canSandboxAutoAllow`），则**跳过 ask、继续往下**——因为沙箱本身提供了隔离保护，沙箱里的命令可以不必每次问（第 9 章详解沙箱）。否则就 `return { behavior: 'ask', ... }`。

**1c. 工具自身的权限检查。** 走到这里才调 `tool.checkPermissions(parsedInput, context)`——就是第 6 章那个工具自定义的权限方法。Bash 在这里判断子命令级别的规则（`Bash(git push:*)` 这种）。默认返回 `passthrough`（不表态）。

**1d. 工具实现说 deny。** 如果工具自己返回了 deny（比如 Bash 子命令被拒），直接 `return`。

**1f. 内容级 ask 规则。** 如果工具返回的是带 `ruleBehavior: 'ask'` 的 ask（如 `Bash(npm publish:*)`），返回它。

**1g. 安全检查（bypass-immune）。** 最关键的兜底：某些路径的编辑（`.git/`、`.claude/`、`.vscode/`、shell 配置文件）触发的 ask 是「免疫绕过」的——注释写得很重：「they must prompt even when a PreToolUse hook returned allow」。也就是说，即使一个 PreToolUse hook（第 16 章）已经放行了，碰到这些敏感路径**仍然强制弹窗**。这是给「自动放行链路」留的一道不可逾越的安全闸——再怎么自动化，动你的 git 目录和配置文件还是得当面问你。

**最后**，如果以上规则都没意见，返回 `null`——表示「规则层面无异议」，把决定权交给上层（模式判断、分类器等）。

把这个顺序抽象出来，CCB 的权限优先级是：**整体 deny > 整体 ask >（沙箱例外）> 工具自定义 deny > 工具自定义 ask > 敏感路径强制 ask > 放行**。拒绝永远在最前，敏感路径的询问永远不可绕过，这两条是整套系统的安全脊梁。

## 8.4 ToolPermissionContext：一份不可变的权限快照

所有这些规则被装进 `ToolPermissionContext`——第 6 章提过它是个 `DeepImmutable`（深度只读）的类型。为什么要深度不可变？因为权限上下文在一次工具调用的判定过程中会被多个函数读取，如果中途能被修改，就可能出现「检查时是 A 规则、执行时变成 B 规则」的 TOCTOU（检查与使用之间的时间差）漏洞。把它冻成只读，从类型层面杜绝了这类篡改。

权限规则从磁盘加载、合并到这个上下文的过程由 `permissionsLoader.ts` 和 `applyPermissionRulesToPermissionContext`（第 1429 行）、`syncPermissionRulesFromDisk`（第 1440 行）负责。合并时按 `allow`/`deny`/`ask` 三类分别处理（第 1455、1480 行能看到 `behaviors: PermissionBehavior[] = ['allow', 'deny', 'ask']` 的遍历），各来源的规则按优先级叠加——再次强调，叠加的语义是 AND（收紧），不是 OR（放宽）。

## 8.5 拒绝也要被记账

权限系统不只是「当场判一下」，它还跟踪拒绝历史。`denialTracking.ts` 与 `persistDenialState`（第 984 行）、`handleDenialLimitExceeded`（第 1005 行）一起，记录某个动作被拒绝了多少次。这有两个用途：一是避免模型在被拒后无限重试同一个被禁动作（撞墙式死循环），二是当拒绝次数超限时可以采取进一步措施。这是「错误即文档」原则的延伸——被拒绝不是一个静默的失败，而是一个被记录、被反馈、能影响后续决策的事件。

权限管线还为 headless agent 预留了 hook 介入点：`runPermissionRequestHooksForHeadlessAgent`（第 400 行）让 PreToolUse/PermissionRequest 钩子在弹不出窗的无人值守场景里也能参与放行/拒绝决策（第 16 章详解）。

读到这里，权限系统的全貌清晰了：它是一台**优先级明确、拒绝优先、约束只收紧、敏感操作不可绕过、且对自身决策可观测**的审批机器。但它还有一个没展开的角落——当动作是一条 bash 命令时，「这条命令到底安不安全」本身就是个大难题。下一章，我们钻进 BashTool 的 AST 安全门。

## 本章小结

- 权限系统有两个正交坐标轴：行为 `PermissionBehavior = 'allow' | 'deny' | 'ask'`，与模式 `EXTERNAL_PERMISSION_MODES`（`default`/`plan`/`acceptEdits`/`dontAsk`/`bypassPermissions`）。
- 内部模式多出 `auto`（第 10 章分类器）和 `bubble`；`auto` 在分类器不可用时回落到弹窗询问（fail-closed）；模式集用 `as const satisfies` 固化进类型。
- 规则形状是 `Tool(content)`（如 `Bash(npm run test:*)`），由 `permissionRuleParser.ts` 解析；`PermissionRule` 含 source/behavior/value。
- 规则来源有优先级，`policySettings`（企业策略）最高且不可被下层推翻——「约束只能收紧」，合并语义是 AND 不是 OR。
- 审批核心 `checkRuleBasedPermissions` 的代码顺序即决策优先级：整体 deny → 整体 ask →（沙箱自动放行例外）→ 工具自定义 deny → 工具自定义 ask → 敏感路径强制 ask → 放行(返回 null)。
- **拒绝永远先行**：命中 deny 规则后所有 allow/ask 判断都无机会翻盘。
- 敏感路径（`.git/`、`.claude/`、`.vscode/`、shell 配置）的 ask 是 bypass-immune 的——即使 PreToolUse hook 已放行仍强制弹窗。
- Bash + 沙箱化命令可跳过 ask（`canSandboxAutoAllow`），因为沙箱本身提供隔离保护。
- `ToolPermissionContext` 是 `DeepImmutable` 深度只读，从类型层面杜绝判定过程中的 TOCTOU 篡改。
- `denialTracking` 记录拒绝历史，防止模型撞墙式无限重试被禁动作；`hasPermissionsToUseTool`(`CanUseToolFn`) 即第 6 章 `call` 的第三个参数。

## 动手实验

1. **实验一：列出五种模式的宽松度排序** — 打开 `src/types/permissions.ts`，找到 `EXTERNAL_PERMISSION_MODES`。结合命名和注释，把这五种模式按「从最保守到最宽松」排序，并说明每种模式下「编辑文件」会发生什么（问/不问/拒绝）。
2. **实验二：复刻决策优先级** — 精读 `checkRuleBasedPermissions`（`src/utils/permissions/permissions.ts` 第 1092 行）的每个 `return`。画一张流程图，标出 1a→1g 每一步的条件和结果。重点理解为什么 1a（deny）必须在最前、1g（敏感路径）必须不可绕过。
3. **实验三：验证 policy 不可推翻** — 在 `permissions.ts` 搜索 `policySettings`，找到 `deletePermissionRule` 里对它的特殊处理（第 1356 行附近）。再看 `PermissionRuleSource` 的枚举顺序。论证：用户在 `localSettings` 里写一条 allow，能否覆盖 `policySettings` 里的一条 deny？
4. **实验四：触碰 bypass-immune 的边界** — 找到处理敏感路径的 `checkPathSafetyForAutoEdit`（返回 `{type:'safetyCheck'}`）相关逻辑（permissions.ts 1g 分支）。列出哪些路径属于「即使 hook 放行也强制询问」。思考这道闸为什么对一个会自动改代码的 Agent 至关重要。

> **下一章预告**：权限系统判定「要不要让 Bash 跑」之前，得先回答「这条命令到底是什么、危不危险」。第 9 章我们钻进 BashTool 的安全核心——用 tree-sitter 把命令解析成 AST、`DANGEROUS_TYPES` 黑名单、`checkReadOnlyConstraints` 只读判定、以及 `shouldUseSandbox` 的沙箱决策，看 CCB 如何在 shell 这个最危险的入口建起 AST 级的 fail-closed 安全门。
