# 第 13 章 插件运行时与 MCP——把外部世界请进来,又不让它拆了承重墙

## 13.1 最开放的场景,最难守的边界

前面十二章,我们守的都是"内部"的边界:谁能调用网关、配置如何热更、设备如何配对、令牌如何轮换。这些场景里,参与者要么是 OpenClaw 自己的代码,要么是经过配对授信的设备。但真正的考验从来不在内部——而在**对接外部世界**的那一刻。

第三方插件(plugin)就是这个外部世界。它是别人写的代码,跑在你的进程里;它想往核心里塞工具、塞 channel、塞 model provider、塞 CLI 命令;它可能来自 npm 仓库,带着整条供应链的不确定性。一个插件系统的设计水平,不取决于它"能扩展多少",而取决于它在扩展的同时**守住了多少**。

本章的主线只有一句话——这也是贯穿全书的那条底线在最开放场景下的回响:**插件能扩展核心,但不能削弱核心**。我们会看到 OpenClaw 如何用四道防线把这句话落到实处:运行时加载边界(bundled 插件必须是编译产物)、能力贡献的所有权追踪、把插件工具通过 MCP 暴露时强制套上与 agent 同样的执行前钩子、以及一整套面向供应链的信任审计(allowlist、版本锁定、完整性校验、漂移检测)。

## 13.2 插件 scope:`undefined` 与空数组的语义鸿沟

先从最小的一块基石看起——插件的 scope(作用域)如何归一化。`plugin-scope.ts` 只有 38 行,却埋了一个极易被忽视、又极其关键的语义区分[[plugin-scope.ts]](https://github.com/openclaw/openclaw/blob/main/src/plugins/plugin-scope.ts):

```ts
export type PluginIdScope = readonly string[] | undefined;

/** True when plugin scope was explicitly provided, including an empty scope. */
export function hasExplicitPluginIdScope(ids?: readonly string[]): boolean {
  return ids !== undefined;
}

/** True when plugin scope was explicitly provided with at least one id. */
export function hasNonEmptyPluginIdScope(ids?: readonly string[]): boolean {
  return ids !== undefined && ids.length > 0;
}

export function createPluginIdScopeSet(ids?: readonly string[]): ReadonlySet<string> | null {
  if (ids === undefined) {
    return null;
  }
  return new Set(ids);
}
```

注意 `undefined` 和 `[]`(空数组)被刻意区分成两种完全不同的含义:

- `undefined` —— **未限定(unscoped)**。`createPluginIdScopeSet` 对它返回 `null`,代表"没有 scope 过滤这回事",所有插件都在范围内。
- `[]` —— **显式限定为空**。`hasExplicitPluginIdScope([])` 返回 `true`(因为它"明确提供了 scope"),但 `hasNonEmptyPluginIdScope([])` 返回 `false`。`createPluginIdScopeSet([])` 返回一个空 Set,代表"明确地,一个插件都不在范围内"。

为什么要这么较真?因为这两者代表的安全意图截然相反:`undefined` 是"我没设限",`[]` 是"我设了限,而且限死为零"。如果代码用 `ids?.length` 这种写法去判断,两者会被混为一谈——空数组的 `.length` 是 0,`undefined?.length` 也是 `undefined`(falsy),于是"明确禁止全部"会被误解成"未设限"。`serializePluginIdScope` 同样把 `undefined` 序列化成专门的 `"__unscoped__"` 字符串,与 `JSON.stringify([])` 的 `"[]"` 区分开,保证缓存键不会把两种语义撞在一起。

这是一个微小但反复出现的工程纪律:**"没说"和"说了不要"是两件事**。在权限系统里把它们区分清楚,是避免"默认放行"漏洞的第一步。

## 13.3 运行时加载边界:内置插件必须是编译产物

插件代码要被加载进进程才能跑。这一步看似只是 `import()`,实则是又一道安全闸。`runtime-plugin-boundary.ts` 的 `loadPluginBoundaryModule` 揭示了一条针对**来源(origin)**的硬规则[[runtime-plugin-boundary.ts]](https://github.com/openclaw/openclaw/blob/main/src/plugins/runtime/runtime-plugin-boundary.ts):

```ts
export function loadPluginBoundaryModule<TModule>(
  modulePath: string,
  loaders: PluginModuleLoaderCache,
  options: { origin?: PluginOrigin } = {},
): TModule {
  if (isJavaScriptModulePath(modulePath)) {
    const native = tryNativeRequireJavaScriptModule(modulePath, {
      allowWindows: true,
      fallbackOnNativeError: options.origin !== "bundled",
    });
    if (native.ok) {
      return native.moduleExport as TModule;
    }
    if (options.origin === "bundled") {
      throw new Error(`bundled plugin runtime module must load natively: ${modulePath}`);
    }
  } else if (options.origin === "bundled") {
    throw new Error(`bundled plugin runtime module must be built JavaScript: ${modulePath}`);
  }

  return getPluginBoundarySourceLoader(modulePath, loaders)(modulePath) as TModule;
}
```

把 `origin === "bundled"`(随 OpenClaw 一起发布的内置插件)单独拎出来,施加两条它独有的约束:

1. **bundled 插件必须是编译好的 JavaScript**。如果给它一个非 `.js` 路径(比如 `.ts` 源码),直接 `throw`——"bundled plugin runtime module must be built JavaScript"。
2. **bundled 插件必须能原生加载(native require)**。如果原生加载失败,它**不允许**回退到源码加载器(`fallbackOnNativeError: origin !== "bundled"`),而是直接 `throw`——"bundled plugin runtime module must load natively"。

为什么对自己人反而更严?因为 bundled 插件是 OpenClaw 信任链的一部分,它在发布前就应该被编译、被验证。允许它在运行时回退到"现编译源码"这条路,等于在一个本该确定的信任锚点上引入不确定性——万一源码被篡改、或者编译环境有差异,内置插件的行为就可能偏离发布时的承诺。而第三方(非 bundled)插件本来就不在这条强信任链里,反而允许更灵活的源码加载回退(`fallbackOnNativeError: true`)。

这又是一处"约束只能收紧"的体现,只不过方向反直觉:**越是核心、越被信任的组件,加载约束越严**。信任不是松绑的理由,而是更高标准的来源。

边界解析还有一个细节值得一提:`resolvePluginRuntimeRecordByEntryBaseNames` 在按入口名反查插件时,如果有多个插件都匹配,会直接抛出 "plugin runtime boundary is ambiguous"[[runtime-plugin-boundary.ts]](https://github.com/openclaw/openclaw/blob/main/src/plugins/runtime/runtime-plugin-boundary.ts)——**歧义即拒绝**,绝不"随便挑一个"。在边界解析里,模棱两可比直接失败更危险,因为它会悄悄把请求路由到错误的插件。

## 13.4 能力贡献与所有权:每一项扩展都记得是谁加的

插件最核心的价值是"贡献能力"。OpenClaw 允许插件向核心贡献八类东西,定义在 `PluginRegistryContributionKey` 里[[plugin-registry-contributions.ts]](https://github.com/openclaw/openclaw/blob/main/src/plugins/plugin-registry-contributions.ts):

```ts
export type PluginRegistryContributionKey =
  | "providers"             // model provider
  | "channels"              // 聊天/消息渠道
  | "channelConfigs"        // 渠道配置
  | "setupProviders"        // 安装向导步骤
  | "cliBackends"           // CLI 后端
  | "modelCatalogProviders" // 模型目录
  | "commandAliases"        // 命令别名
  | "contracts";            // 契约
```

但贡献不是"塞进去就完事"。整个 contributions 模块的设计核心,是一套**所有权(ownership)追踪**:每一项被贡献的能力,系统都记得它来自哪个插件。`resolveProviderOwners`、`resolveChannelOwners`、`resolveCliBackendOwners`、`resolveSetupProviderOwners`……这一组函数回答的都是同一个问题:"这个 provider/channel/CLI 后端,是哪些插件贡献的?"[[plugin-registry-contributions.ts]](https://github.com/openclaw/openclaw/blob/main/src/plugins/plugin-registry-contributions.ts)

为什么所有权如此重要?设想一个场景:某个 model provider 出了安全问题,你需要立刻知道"是哪个插件引入了它,要不要连带禁用"。没有所有权追踪,贡献进核心的能力就成了无主的孤儿——你能看见它,却不知道该找谁负责、该从哪里拔掉。有了所有权,核心始终掌握"每一项扩展的来源",这正是"扩展可被精确撤销"的前提。

第二个关键设计:**禁用的插件,其贡献会被过滤掉**。`resolveContributionPluginIds` 在不含 `includeDisabled` 时,只返回 `isInstalledPluginEnabled` 为真的插件[[plugin-registry-contributions.ts]](https://github.com/openclaw/openclaw/blob/main/src/plugins/plugin-registry-contributions.ts):

```ts
function resolveContributionPluginIds(params: {...}): readonly string[] {
  if (params.includeDisabled) {
    return params.index.plugins.map((plugin) => plugin.pluginId);
  }
  return params.index.plugins
    .filter((plugin) => isInstalledPluginEnabled(params.index, plugin.pluginId, params.config))
    .map((plugin) => plugin.pluginId);
}
```

而且 `filterContributionOwnerIds` 还会**二次过滤所有者**:即使某个能力的 owner 列表里记着插件 X,但如果 X 当前被禁用,它就不会出现在返回的 owner 列表里。这保证了"禁用一个插件"是真正彻底的——它贡献的所有能力,连同它作为 owner 的身份,都从活跃视图里消失。禁用不是"打个标记还在跑",而是"从能力图谱里整个摘除"。

这呼应了第 11 章配置热更的精神:`skills.*` 配置一变就让会话重建快照,避免继续广告已不存在的工具。插件这里是同一个原则的延伸——**核心永远只暴露当前真实启用的能力,绝不让禁用的扩展留下幽灵**。

## 13.5 MCP 桥:插件工具走出去,也得过同一道执行前的门

OpenClaw 通过 MCP(Model Context Protocol)把插件工具暴露给外部 agent 客户端。这是一条"出口"通道:别的 MCP 客户端可以列出、调用 OpenClaw 这边的插件工具。问题来了——通过这条新通道调用插件工具,会不会绕过 agent 内部那一整套执行前的安全检查?

答案藏在 `createPluginToolsMcpHandlers` 开头几行,这是本章最关键的一段代码[[plugin-tools-handlers.ts]](https://github.com/openclaw/openclaw/blob/main/src/mcp/plugin-tools-handlers.ts):

```ts
export function createPluginToolsMcpHandlers(tools: AnyAgentTool[]) {
  const wrappedTools = tools.map((tool) => {
    if (isToolWrappedWithBeforeToolCallHook(tool)) {
      return rewrapToolWithBeforeToolCallHook(tool, undefined, { approvalMode: "report" });
    }
    // The ACPX MCP bridge should enforce the same pre-execution hook boundary
    // as the agent and HTTP tool execution paths.
    return wrapToolWithBeforeToolCallHook(tool, undefined, { approvalMode: "report" });
  });
  ...
}
```

注释把意图说得明明白白:"The ACPX MCP bridge should enforce the **same** pre-execution hook boundary as the agent and HTTP tool execution paths."(MCP 桥必须强制执行与 agent、HTTP 工具执行路径**相同**的执行前钩子边界。)

每一个要通过 MCP 暴露出去的插件工具,都被 `wrapToolWithBeforeToolCallHook` 重新包了一层 `beforeToolCall` 钩子。如果它已经被包过了(`isToolWrappedWithBeforeToolCallHook`),就 `rewrap` 一次确保配置一致。也就是说:无论工具是被 agent 内部调用、被 HTTP 接口调用,还是被 MCP 客户端调用,**它都必须穿过同一道执行前的门**。MCP 不是一条绕过安全检查的后门,而是又一个被纳入同一套边界的前门。

这正是"插件不能削弱核心"的精髓所在。一个天真的实现会把"暴露工具给 MCP"做成简单的转发——拿到工具、直接 `execute`。那样的话,agent 路径上所有的执行前审批/校验逻辑,在 MCP 路径上就全部失效了,插件工具就有了一条不受管的旁路。OpenClaw 拒绝这种实现:**新增一条通道,必须复用已有的边界,而不是在边界上开一个口子**。

`callTool` 的实现也很克制:工具不存在返回 `isError`,执行抛错被 `formatErrorMessage` 捕获并包成错误内容而非让异常逃逸,结果统一规整成 `content` 数组[[plugin-tools-handlers.ts]](https://github.com/openclaw/openclaw/blob/main/src/mcp/plugin-tools-handlers.ts)。这条出口通道对外表现得稳定、可预测——把混乱挡在边界之内。

## 13.6 供应链信任审计:从 allowlist 到完整性哈希的四层防线

前面几节守的是"插件运行时怎么被加载和调用"。但还有一个更早、更根本的问题:**这些插件本身,你凭什么信它?** 一个来自 npm 的插件,可能在某次更新里被投毒(供应链攻击)。`audit-plugins-trust.ts`(578 行)就是 OpenClaw 给运维的一套体检工具,它把信任风险翻译成一条条可操作的 `SecurityAuditFinding`。我们看其中最有代表性的四层。

**第一层:没有 allowlist 就是高风险。** 当扩展目录下有插件、但 `plugins.allow` 没设置时,审计会报一条 finding[[audit-plugins-trust.ts]](https://github.com/openclaw/openclaw/blob/main/src/security/audit-plugins-trust.ts):

```ts
findings.push({
  checkId: "plugins.extensions_no_allowlist",
  severity: skillCommandsLikelyExposed ? "critical" : "warn",
  title: "Extensions exist but plugins.allow is not set",
  detail:
    `Found ${pluginDirs.length} extension(s) under ${extensionsDir}. Without plugins.allow, ` +
    `any discovered plugin id may load ...`,
  remediation: "Set plugins.allow to an explicit list of plugin ids you trust.",
});
```

注意 severity 是**动态的**:如果检测到至少一个已配置的聊天界面开启了原生 skill 命令(`skillCommandsLikelyExposed`),严重级别从 `warn` 直接升到 `critical`。逻辑很直白——没有 allowlist 时任何被发现的插件都可能加载,而如果这些插件的工具还能通过聊天界面被触达,风险就从"潜在"变成了"现实可达"。审计不是机械地报警,而是结合实际暴露面去评估真实风险。

**第二层:权限策略太宽,插件工具可能可达。** 审计会用一个**合成探针**去测每个 agent 上下文的工具策略[[audit-plugins-trust.ts]](https://github.com/openclaw/openclaw/blob/main/src/security/audit-plugins-trust.ts):

```ts
// Probe with a synthetic plugin tool id: broad allow policies will allow
// it, while restrictive profiles or explicit allowlists should not.
const broadPolicy = deps.isToolAllowedByPolicies("__openclaw_plugin_probe__", policies);
```

它构造一个根本不存在的工具 id `__openclaw_plugin_probe__`,然后问当前策略"这个工具允许吗?"。一个收紧的 profile(`minimal`/`coding`)或显式 allowlist 应该拒绝这个陌生 id;但一个过于宽松的"allow-all"策略会连这个虚构的 id 都放行——于是探针就暴露了"这个上下文对插件工具门户大开"。检出后报 `plugins.tools_reachable_permissive_policy`,并建议对处理不可信输入的 agent 用收紧 profile 或排除插件工具的显式 allowlist。

用一个不存在的 id 探测策略宽松度,是一个非常漂亮的技巧:**它不依赖任何具体插件,直接测量"边界本身有多松"**。

**第三、四层:npm 包的锁定与完整性。** 对来源为 npm 的插件,审计连查三项供应链证据[[audit-plugins-trust.ts]](https://github.com/openclaw/openclaw/blob/main/src/security/audit-plugins-trust.ts):

- **未锁定版本(unpinned spec)**:install 记录里的 spec 不是精确版本(如用了 `^`/`~`/范围),报 `plugins.installs_unpinned_npm_specs`,建议钉死到 `@scope/pkg@1.2.3`。范围版本意味着下次安装可能悄悄拉到一个不同的、未经审查的版本。
- **缺失完整性元数据(missing integrity)**:install 记录没有 `integrity` 哈希,报 `plugins.installs_missing_integrity`。没有完整性哈希,就无法验证下载到的包字节是否被篡改。
- **版本漂移(version drift)**:记录里的版本与本地实际安装的 `package.json` 版本不一致,报 `plugins.installs_records_drift`。注释点出处理原则——"Installed package.json is the local truth"[[audit-plugins-trust.ts]](https://github.com/openclaw/openclaw/blob/main/src/security/audit-plugins-trust.ts):本地安装的才是事实,记录漂移说明供应链证据该刷新了。同样的三连检查也施加于 hook installs。

这三层合起来,构成了对供应链的纵深防御:**锁定版本**(知道你装的是哪一版)、**完整性哈希**(确认字节没被换)、**漂移检测**(发现记录与现实脱节)。它们都不阻止你安装插件——它们只是把每一处不确定性变成一条带修复建议的、可见的 finding,让信任成为一个可审计、可决策的事情,而不是一句"应该没问题"的侥幸。

最后,审计扫描目录时会先用 `shouldIgnoreInstalledPluginDirName` 过滤掉 `node_modules`、隐藏目录、`.bak`/`.backup-`/`.disabled` 这类安装残骸[[installed-plugin-dirs.ts]](https://github.com/openclaw/openclaw/blob/main/src/security/installed-plugin-dirs.ts)。注释说明了缘由:失败安装和回滚副本里可能有陈旧的插件代码,只审计活跃的根目录、忽略这些生成的备份,能让 finding 保持"可操作(actionable)"。一个会对一堆死代码报警的审计工具,很快就会被运维当成噪音忽略——**让告警保持可信,本身就是安全设计的一部分**。

## 本章小结

这一章我们在最开放的场景——对接第三方插件与 MCP——里,看 OpenClaw 如何把"扩展核心而不削弱核心"这条底线落到代码:

- **`undefined` ≠ `[]`**:插件 scope 严格区分"未限定"(unscoped,`null`)与"显式限定为零"([],空 Set)。"没说"和"说了不要"是两回事,混淆它们就是默认放行漏洞的温床。
- **越核心越严**:`loadPluginBoundaryModule` 对 bundled 内置插件施加更严约束——必须是编译产物、必须能原生加载、不许回退到源码加载;第三方插件反而允许更灵活的回退。信任是更高标准的来源,不是松绑的理由。歧义的运行时边界一律拒绝而非猜测。
- **每一项贡献都有主**:插件向核心贡献八类能力(providers/channels/cliBackends/...),系统用一套 owner 追踪记住每项能力来自哪个插件;禁用插件时,其贡献与 owner 身份一并从活跃视图摘除。核心永不暴露禁用扩展的幽灵能力。
- **MCP 是前门不是后门**:`createPluginToolsMcpHandlers` 给每个对外暴露的插件工具重新套上与 agent、HTTP 路径相同的 `beforeToolCall` 钩子。新增通道必须复用已有边界,绝不在边界上开口子。
- **供应链四层防线**:无 allowlist(动态升级到 critical)、宽松策略探针(用合成 id 测边界松紧)、npm 版本锁定、完整性哈希与版本漂移。审计不阻止安装,只把每处不确定性变成可操作的 finding;并主动过滤安装残骸,让告警保持可信。

把这一章放回全书的脉络:从第 10 章的网关总线、第 11 章的配置热更、第 12 章的配对授信,到本章的插件信任,OpenClaw 反复演练的是同一种思维——**任何新增的能力、通道、参与者,都必须先被纳入既有的边界与可观测体系,然后才被允许发挥作用**。扩展永远是"在边界之内长出来的",而不是"在边界上凿出来的"。

## 动手实验

> 以下实验基于本章引用的源码文件,建议在你 clone 的 OpenClaw 仓库里对照阅读与验证。

**实验一:制造 `undefined` 与 `[]` 的语义事故。**
读 `src/plugins/plugin-scope.ts`。假设有人把 `createPluginIdScopeSet` 的判断从 `if (ids === undefined)` 改成 `if (!ids || ids.length === 0)`,会发生什么?具体推演:当调用方"明确传入空 scope([])表示禁止所有插件"时,这个改动会让结果变成什么?用 `hasExplicitPluginIdScope` 与 `hasNonEmptyPluginIdScope` 的返回值差异,解释为什么这两个函数必须分开存在。

**实验二:理解"越核心越严"。**
读 `src/plugins/runtime/runtime-plugin-boundary.ts` 的 `loadPluginBoundaryModule`。分别推演三种输入:(a) origin=bundled + `.ts` 路径;(b) origin=bundled + `.js` 但原生加载失败;(c) origin=非 bundled + `.js` 但原生加载失败。各自的结果是什么?然后回答:为什么对"自己人"(bundled)的加载约束反而比第三方更严?这与第 12 章"约束只能收紧"如何呼应、又在方向上有何不同?

**实验三:追踪一项能力的所有权与禁用传播。**
读 `src/plugins/plugin-registry-contributions.ts` 的 `resolveContributionPluginIds` 和 `filterContributionOwnerIds`。假设插件 A 贡献了一个 channel `foo`,现在把 A 在配置里禁用。沿着代码走一遍:`resolveChannelOwners("foo")` 还会返回 A 吗?为什么"禁用"必须同时从"贡献列表"和"owner 列表"两处摘除,只摘一处会留下什么隐患?

**实验四:复现合成探针。**
读 `src/security/audit-plugins-trust.ts` 中用 `__openclaw_plugin_probe__` 做策略探测的那段。解释:为什么用一个**不存在**的工具 id 反而能测出策略的宽松度?如果改用一个真实存在的插件工具 id 去探测,结论会有什么偏差?再把这条 finding(`plugins.tools_reachable_permissive_policy`)的 remediation 与第一层 finding(`plugins.extensions_no_allowlist`)的动态 severity 联系起来,说说这套审计是如何"结合真实暴露面"而非机械报警的。

## 下一章预告

我们已经把"如何安全地把外部能力请进来"讲透了。但插件、节点、工具一旦跑起来,就会产生海量的运行时事件——调用了什么、耗时多久、报了什么错、谁在背后触发。第 14 章我们将转向 OpenClaw 的**可观测性与遥测(observability & telemetry)**:这些在前十三章里反复出现的"广播事件""日志""降级/恢复信号""审计 finding",在底层是如何被统一采集、结构化、并在不泄露敏感信息的前提下变得可查询的?当一个 agent 在深夜自动跑崩时,运维要如何从这条遥测链路里还原出"到底发生了什么"?我们将看到,可观测性不是事后补的日志,而是和安全边界一样,从第一行代码就被设计进系统的骨架里。
