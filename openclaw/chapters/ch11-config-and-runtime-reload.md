# 第 11 章 配置系统与运行时重载——让一个长跑进程安全地吞下新配置

> 上一章我们拆完了网关这条总线,并在结尾留了个伏笔:总线上跑的方法,有相当一部分(那些 `config.*`、那些打了 `controlPlaneWrite: true` 标记的方法)做的事情就是读写系统的"活配置"。当一个操作者通过 `config.set` 改了一行配置,或者直接手动编辑了配置文件,一个**已经跑了好几天、手里攥着几十个会话、连着十几条 channel** 的网关进程,该怎么办?
>
> 最粗暴的答案是"重启"。但重启意味着所有 channel 断线重连、所有 in-flight 的会话被打断、几秒到几十秒的不可用窗口。对一个 7×24 跑着的 agent 运行时来说,这代价太大了。另一个极端是"全都热替换",但有些配置(比如网关监听地址、插件加载方式)**根本没法在不重启的前提下安全切换**——硬热替换只会让系统进入一个半新半旧的、谁也说不清的状态。
>
> OpenClaw 的答案是**精确分类 + 最小动作**:diff 出到底哪几个配置路径变了,逐条判断每条变化"能热重载就热重载、必须重启才重启、无所谓就什么都不做",然后只执行刚好够用的那一组动作。这一章,我们就来拆这套机制——它是"快速失败 + 可恢复"这条全书主线在配置层面的一次完整演练。

本章路线:

1. 配置变化是怎么被**发现**的(文件 watcher + 进程内写通知,双通道);
2. 变化是怎么被**精确定位**到具体路径的(`diffConfigPaths`);
3. 每条路径变化对应**什么动作**(`config-reload-plan` 的规则表与三分类);
4. 计划是怎么被**调度执行**的(防抖、串行化、四种 reload mode);
5. 当配置/密钥**坏掉**时,系统如何**拒绝坏配置、停在 last-known-good 上**(可恢复)。

---

## 11.1 两个看似矛盾的诉求

先把问题摆清楚。一个长期运行的 agent 网关,对"配置变更"有两个互相拉扯的诉求:

- **要够灵敏**:改一个模型、调一个心跳间隔、加一条 cron,最好立刻生效,不要逼用户重启整个服务。
- **要够安全**:有些变更(改监听端口、换插件加载策略)在运行中热切换是危险的;而一份**语法错误或语义非法**的配置如果被照单全收,整个进程可能直接崩掉。

把这两个诉求拆开,你会发现它们其实指向同一套机制的两端:

- 灵敏的一端,需要一个**精确的分类器**——知道哪些变更可以热重载、热重载时具体要重启哪些子系统;
- 安全的一端,需要一个**守门人**——坏配置进不来,且即便进不来,系统也得**稳稳停在上一份已知良好的配置上**继续服务,而不是崩溃。

OpenClaw 把这两端分别落在 `config-reload-plan.ts`(分类器)和 `config-reload.ts` + `server-startup-config.ts`(守门人与执行器)里。我们一段段看。

---

## 11.2 发现变化:文件 watcher 与进程内写通知

配置可能从两个地方变:**有人在外面手动编辑了配置文件**,或者**进程内部通过 `config.set` / `config.apply` 这类网关方法写了配置**。`startGatewayConfigReloader`(`config-reload.ts`)同时盯着这两条通道。

第一条是文件系统监听,用的是 `chokidar`:

```typescript
const watcher = chokidar.watch(opts.watchPath, {
  ignoreInitial: true,
  awaitWriteFinish: { stabilityThreshold: 200, pollInterval: 50 },
  usePolling: Boolean(process.env.VITEST),
});
watcher.on("add", scheduleFromWatcher);
watcher.on("change", scheduleFromWatcher);
watcher.on("unlink", scheduleFromWatcher);
```

注意 `awaitWriteFinish`——它等文件写"稳定"了再触发,避免在一次保存的中途读到半截文件。

第二条是进程内写通知 `subscribeToWrites`。当网关自己写了配置,它会带着一份**已经在内存里的、解析好的配置**直接通知 reloader,而不是让 reloader 再去磁盘上重读一遍:

```typescript
const unsubscribeFromWrites = opts.subscribeToWrites?.((event) => {
  if (event.configPath !== opts.watchPath) return;
  pendingInProcessConfig = {
    config: event.runtimeConfig,
    compareConfig: event.sourceConfig,
    persistedHash: event.persistedHash,
    afterWrite: event.afterWrite,
  };
  lastAppliedWriteHash = event.persistedHash;
  scheduleAfter(0);  // 进程内写无需防抖,立即处理
});
```

这两条通道会"撞车":进程内写了配置,文件 watcher 也会因为文件变了而被触发。OpenClaw 用 `lastAppliedWriteHash` 去重——文件 watcher 读到的快照如果 hash 和刚刚进程内写过的一致,就直接跳过:

```typescript
const snapshot = await opts.readSnapshot();
if (lastAppliedWriteHash && typeof snapshot.hash === "string") {
  if (snapshot.hash === lastAppliedWriteHash) {
    return;  // 这次文件变化就是我们自己刚写的,已处理过
  }
  lastAppliedWriteHash = null;
}
```

> **设计观察:同一个事实可能从多个通道到达,去重要靠内容指纹而非来源。** 用 hash 而不是"谁触发的"来判断"是不是同一次变更",是因为来源不可靠(文件事件和写通知的到达顺序无法保证),但内容指纹是可靠的。这是分布式/并发系统里的一个通用招式——**幂等性建立在内容上,不建立在事件次序上**。

### 防抖:把一阵抖动收成一次重载

配置写入常常不是原子的一次文件变化,而是"写临时文件→改名→可能再补一次"的一连串事件。如果每个事件都触发一次重载,就会做无用功甚至引发竞态。`scheduleAfter` 用一个可清除的定时器把短时间内的一连串触发**合并(coalesce)**成一次重载:

```typescript
const scheduleAfter = (wait: number) => {
  if (stopped) return;
  if (debounceTimer) clearTimeout(debounceTimer);  // 有新触发就重置计时
  debounceTimer = setTimeout(() => { void runReload(); }, wait);
};
```

防抖时长默认 300ms(见 `config-reload-settings.ts` 的 `DEFAULT_RELOAD_SETTINGS`),可由 `gateway.reload.debounceMs` 配置。而进程内写调用的是 `scheduleAfter(0)`——它本就拿着解析好的内存配置,没有"文件写到一半"的风险,无需等。

### 串行化:重载过程中又来了变化怎么办

`runReload` 用三个布尔量保证**任意时刻只有一次重载在跑**,且不丢掉重载过程中到来的新变化:

```typescript
const runReload = async () => {
  if (stopped) return;
  if (running) { pending = true; return; }  // 已在跑,标记"还有活儿"
  running = true;
  try {
    /* ……执行重载…… */
  } finally {
    running = false;
    if (pending) { pending = false; schedule(); }  // 跑完发现期间有新变化,再来一轮
  }
};
```

这是一个经典的"运行中标记 + 收尾补跑"模式:重载不可重入,但也绝不会因为"正忙"而丢掉一次变更——忙完了一定会补上。

---

## 11.3 定位变化:`diffConfigPaths` 把两份配置压成一串路径

发现"配置变了"还不够,得知道**具体是哪几个键变了**——因为后面要按路径决定动作。`config-diff.ts` 里的 `diffConfigPaths` 干这件事:递归比较新旧两份配置,产出一串**点分路径**(如 `agents.defaults.model`、`cron.jobs.daily`)。

它的实现短得惊人,但每一行都有讲究:

```typescript
export function diffConfigPaths(prev: unknown, next: unknown, prefix = ""): string[] {
  if (prev === next) return [];  // 引用相同,整棵子树都没变
  if (isPlainObject(prev) && isPlainObject(next)) {
    const keys = new Set([...Object.keys(prev), ...Object.keys(next)]);  // 并集:能发现新增和删除
    const paths: string[] = [];
    for (const key of keys) {
      if (prev[key] === undefined && next[key] === undefined) continue;
      const childPrefix = prefix ? `${prefix}.${key}` : key;
      paths.push(...diffConfigPaths(prev[key], next[key], childPrefix));
    }
    return paths;
  }
  if (Array.isArray(prev) && Array.isArray(next)) {
    // 数组里可能装着对象(如 memory.qmd.paths);结构相等就不算变
    if (isDeepStrictEqual(prev, next)) return [];
  }
  return [prefix || "<root>"];
}
```

三个关键决策:

1. **`prev === next` 的提前返回。** 配置树里大量子树在两次快照间是**同一个引用**(没被改过),引用相等就能整棵跳过,不必深比。这让"只改一个键"的常见场景几乎零成本。
2. **键取并集。** 用 `new Set([...prev keys, ...next keys])` 而非只遍历其中一份,才能同时发现"新增的键"和"删除的键"。
3. **数组用结构相等比较。** 数组不能像对象那样逐键下钻(下标不稳定),所以一旦两个数组引用不同,就用 `isDeepStrictEqual` 做一次结构比较——内容相同就不报告为变化。注释专门点了 `memory.qmd.paths` 这种"数组里装对象"的真实场景:不这么比,重排或等值重写会被误判成变更,触发无谓的重载。

`diffConfigPaths` 的产物——一串变化路径——就是下一步规则匹配的输入。

---

## 11.4 分类变化:规则表与三分类

到了这一章的心脏:`config-reload-plan.ts`。它要回答的问题是——**给定一条变化路径(如 `cron.jobs.daily`),系统该做什么?** 答案被编码成一张**有序的前缀规则表**,每条规则把一个配置前缀映射到三种"分类"之一:

```typescript
type ReloadRule = {
  prefix: string;
  kind: "restart" | "hot" | "none";
  actions?: ReloadAction[];
};
```

- **`restart`**:这条变化无法安全热重载,**必须重启整个网关**。
- **`hot`**:可以热重载,且附带一组**具体动作**(`actions`)——重启哪个子系统、重载哪部分状态。
- **`none`**:无所谓,**什么都不用做**(运行时每次都从当前配置现读这个值)。

挑几条规则感受一下这套分类的判断标准:

```typescript
const BASE_RELOAD_RULES: ReloadRule[] = [
  { prefix: "gateway.remote", kind: "none" },
  { prefix: "gateway.channelHealthCheckMinutes", kind: "hot", actions: ["restart-health-monitor"] },
  { prefix: "hooks.gmail", kind: "hot", actions: ["restart-gmail-watcher"] },
  { prefix: "hooks", kind: "hot", actions: ["reload-hooks"] },
  { prefix: "agents.defaults.model", kind: "hot", actions: ["restart-heartbeat"] },
  { prefix: "models.pricing", kind: "restart" },
  { prefix: "models", kind: "hot", actions: ["restart-heartbeat"] },
  { prefix: "cron", kind: "hot", actions: ["restart-cron"] },
  { prefix: "mcp", kind: "hot", actions: ["dispose-mcp-runtimes"] },
  { prefix: "plugins.load", kind: "restart" },
  { prefix: "plugins.installs", kind: "restart" },
];
```

读这张表本身就是一堂"什么能热、什么不能热"的实战课:

- 改 `cron` → **热重载**,但要 `restart-cron`(重建调度器,让新的任务定义生效);
- 改 `agents.defaults.model` → 热重载 + `restart-heartbeat`(心跳要按新模型跑);
- 改 `mcp` → 热重载 + `dispose-mcp-runtimes`(销毁旧的 MCP 运行时,下次按需重建);
- 改 `models.pricing` → **必须重启**(定价表渗透得太深,没法干净热替换);
- 改 `plugins.load` / `plugins.installs` → **必须重启**(插件加载方式变了,只能从头来)。

注意 `hooks.gmail` 排在 `hooks` 前面——**顺序很重要**。规则是按表中顺序逐条匹配的(`matchRule` 遇到第一个前缀命中的规则就返回),所以更具体的前缀必须排在更宽泛的前缀之前,否则就会被宽泛规则"截胡"。

### 兜底:不认识的路径,一律重启

表的最尾巴(`BASE_RELOAD_RULES_TAIL`)是宽泛的兜底规则,最后两条尤其关键:

```typescript
const BASE_RELOAD_RULES_TAIL: ReloadRule[] = [
  // ……一批 kind: "none" 的宽泛前缀……
  { prefix: "plugins", kind: "hot", actions: ["reload-plugins", "dispose-mcp-runtimes"] },
  { prefix: "ui", kind: "none" },
  { prefix: "gateway", kind: "restart" },
  { prefix: "discovery", kind: "restart" },
];
```

而真正的安全网在 `matchRule` 之外——如果一条路径**任何规则都没匹配上**,`buildGatewayReloadPlan` 会怎么办?

```typescript
const rule = matchRule(path);
if (!rule) {
  plan.restartGateway = true;
  plan.restartReasons.push(path);
  continue;
}
```

`resolveConfigReloadMetadata` 也是同样的兜底:

```typescript
return { kind: matchRule(path)?.kind ?? "restart" };
```

**匹配不到 = 重启。** 这是"默认安全(fail-safe default)"的又一次落地——和第 10 章"未分类方法默认拒绝"是同一种思维:**面对未知,选择最保守的动作。** 一个我们没见过的配置路径,我们无法保证它能被安全热重载,那就重启——宁可慢一点、稳一点,也不要让系统进入一个不可知的半新半旧状态。

### 插件可以扩展规则,但有边界

这张表不是写死的。`listReloadRules` 会把**插件和 channel 声明的重载规则**拼接进来:channel 插件可以贡献 `hot`/`none` 前缀,普通插件可以贡献 `restart`/`hot`/`none` 前缀。但注意拼接顺序:

```typescript
const rules = [
  ...BASE_RELOAD_RULES,       // 核心规则在前
  ...pluginReloadRules,
  ...channelReloadRules,
  ...channelPluginStateRules,
  ...BASE_RELOAD_RULES_TAIL,  // 宽泛兜底在后
];
```

核心的具体规则在最前,宽泛兜底在最后,插件规则夹在中间。由于是"首个命中即返回",插件**无法覆盖**核心已经精确分类的前缀,只能在核心没管的命名空间里补充规则。这又是第 10 章那条"插件能扩展、不能削弱核心"原则的延续。

而且这张拼接出来的表是**带版本缓存**的:只要活跃插件注册表的版本号没变,规则表就复用缓存,让每一次 config diff 都很便宜:

```typescript
if (registry !== cachedRegistry ||
    activeRegistryVersion !== cachedActiveRegistryVersion ||
    channelRegistryVersion !== cachedChannelRegistryVersion) {
  cachedReloadRules = null;  // 注册表变了才重建
  /* ……更新缓存版本…… */
}
```

### 从分类到计划:`buildGatewayReloadPlan`

把一串路径喂给 `buildGatewayReloadPlan`,它逐条匹配规则、累积动作,产出一个 `GatewayReloadPlan`——一份"该重启什么、该热重载什么"的清单:

```typescript
export type GatewayReloadPlan = {
  changedPaths: string[];
  restartGateway: boolean;       // 是否需要整体重启
  restartReasons: string[];      // 为什么要重启(哪些路径触发的)
  hotReasons: string[];
  reloadHooks: boolean;
  restartCron: boolean;
  restartHeartbeat: boolean;
  restartHealthMonitor: boolean;
  reloadPlugins: boolean;
  restartChannels: Set<ChannelKind>;
  disposeMcpRuntimes: boolean;
  noopPaths: string[];
  // ……
};
```

注意 `restartReasons` 和 `hotReasons` 是**字符串数组**,记录的是"哪些路径触发了重启/热重载"。这又是"错误即文档"的影子——计划本身就携带了它为什么这么决策的人类可读理由,出问题时日志里能直接看到"是因为改了 `models.pricing` 才重启的"。最后还有一条小小的级联:

```typescript
if (plan.restartGmailWatcher) {
  plan.reloadHooks = true;  // 重启 gmail watcher 必然伴随重载 hooks
}
```

---

## 11.5 执行计划:四种 reload mode

有了计划,`applySnapshot`(`config-reload.ts`)负责执行。但执行前还有一道总开关:**reload mode**。`config-reload-settings.ts` 定义了四种模式,默认 `hybrid`:

```typescript
const DEFAULT_RELOAD_SETTINGS: GatewayReloadSettings = {
  mode: "hybrid",
  debounceMs: 300,
};
```

四种模式的语义,在 `applySnapshot` 的决策树里看得最清楚:

```typescript
if (settings.mode === "off") {
  opts.log.info("config reload disabled (gateway.reload.mode=off)");
  return;  // off:什么都不做,改了也不生效(直到手动重启)
}
// ……
if (settings.mode === "restart") {
  queueRestart(plan, nextConfig);  // restart:任何变化都整体重启
  return;
}
if (plan.restartGateway) {
  if (settings.mode === "hot") {
    // hot:只热重载;遇到本该重启的变化,拒绝执行并警告
    opts.log.warn(`config reload requires gateway restart; hot mode ignoring (...)`);
    return;
  }
  queueRestart(plan, nextConfig);  // hybrid:该重启就重启
  return;
}
await opts.onHotReload(plan, nextConfig);  // 走到这:执行热重载
```

- **`off`**:关掉自动重载。改了配置也不生效,等人工重启。
- **`restart`**:省事但粗暴——任何变化都整体重启。
- **`hot`**:只做热重载;遇到"本该重启"的变化,**拒绝并警告**,绝不偷偷重启。这给了用户一个"我就是不想让它自动重启"的强保证。
- **`hybrid`(默认)**:能热则热,该重启则重启——也就是把 11.4 那张分类表的判断**忠实执行**。

`hybrid` 是这套机制的精华:它让大多数日常微调(改模型、调心跳、加 cron)走热重载、毫秒级生效,只在确实必要时(改端口、换插件加载)才付出重启的代价。**默认就是"灵敏与安全的最优折中"**,而极端诉求(完全不重启 / 总是重启 / 干脆别管)都有对应模式可选。

### 写入方的意图也有一票

值得一提的是,配置的写入方还能附带"意图",由 `resolveConfigWriteFollowUp` 解析:

```typescript
const followUp = resolveConfigWriteFollowUp(afterWrite);
if (followUp.mode === "none") {
  opts.log.info(`config reload skipped by writer intent (${followUp.reason})`);
  return;  // 写入方明说"这次写入不要触发重载"
}
// ……
if (followUp.requiresRestart) {
  queueRestart({ ...plan, restartGateway: true,
    restartReasons: [...plan.restartReasons, followUp.reason] }, nextConfig);
  return;  // 写入方明说"这次必须重启"
}
```

也就是说,发起写入的那个网关方法,可以声明"我这次只是写个时间戳,别重载"或"我改的东西必须重启"。这让分类表(基于路径的静态判断)和写入方(基于语义的动态判断)能协同决策。

### `restart` 失败了不会拖垮 reloader

重启检查本身可能失败(比如配置里有解析不出来的 SecretRef)。`queueRestart` 把这种失败隔离掉,**保持 reloader 存活**,让后续的变更还有机会重试:

```typescript
const queueRestart = (plan, nextConfig) => {
  if (restartQueued) return;
  restartQueued = true;
  void (async () => {
    try {
      await opts.onRestart(plan, nextConfig);
    } catch (err) {
      restartQueued = false;  // 失败后解锁,允许未来的变更重试
      opts.log.error(`config restart failed: ${String(err)}`);
    }
  })();
};
```

注释说得很直白:重启检查可能因为未解析的 SecretRef 而失败,但不能让这个失败搞死整个重载器。**一个子操作的失败,不应该让上层的协调器失能**——这是健壮编排的基本素养。

---

## 11.6 守门人:拒绝坏配置,停在 last-known-good

现在轮到安全的那一端。如果新配置**语法错误或语义非法**怎么办?reloader 的回答是:**根本不让它进来。**

```typescript
const handleInvalidSnapshot = (snapshot: ConfigFileSnapshot): boolean => {
  if (snapshot.valid) return false;
  const issues = formatConfigIssueLines(snapshot.issues, "").join(", ");
  opts.log.warn(`config reload skipped (invalid config): ${issues}`);
  return true;  // 非法配置:跳过本次重载,运行时维持原样
};
```

注意它的行为不是"崩溃",也不是"勉强用一半",而是**跳过这次重载,把具体的校验问题打进日志**。运行时继续用着当前那份合法配置——也就是 **last-known-good(上一份已知良好)**。坏配置被挡在门外,服务不受影响,用户从日志里能看到到底哪里写错了。

文件找不到时同理,而且还带有限重试(应对"改名瞬间文件短暂不存在"这种竞态):

```typescript
const MISSING_CONFIG_MAX_RETRIES = 2;
const handleMissingSnapshot = (snapshot: ConfigFileSnapshot): boolean => {
  if (snapshot.exists) { missingConfigRetries = 0; return false; }
  if (missingConfigRetries < MISSING_CONFIG_MAX_RETRIES) {
    missingConfigRetries += 1;
    scheduleAfter(MISSING_CONFIG_RETRY_DELAY_MS);  // 短暂重试
    return true;
  }
  opts.log.warn("config reload skipped (config file not found)");
  return true;
};
```

### 只有"被接受"的快照才会被晋升为 last-known-good

那么 last-known-good 是怎么维护的?关键在于:**只有真正通过了校验、被运行时接受的快照,才会被"晋升(promote)"成新的 last-known-good。**

```typescript
const promoteAcceptedSnapshot = async (snapshot, reason) => {
  if (!opts.promoteSnapshot || !snapshot.exists || !snapshot.valid) return;
  try {
    await opts.promoteSnapshot(snapshot, reason);
  } catch (err) {
    opts.log.warn(`config reload last-known-good promotion failed: ${String(err)}`);
  }
};
```

在 `runReload` 里,晋升发生在 `applySnapshot` 成功之后:

```typescript
await applySnapshot(snapshot.config, snapshot.sourceConfig);
await promoteAcceptedSnapshot(snapshot, "valid-config");  // 接受后才晋升
```

这就形成了一个清晰的闭环:**坏配置 → 校验失败 → 跳过、不晋升 → last-known-good 保持不变 → 服务继续**。好配置 → 校验通过 → 应用 → 晋升为新的 last-known-good。系统永远站在最近一份"被证明能用"的配置上。

### 密钥解析失败:降级但不下线

last-known-good 的威力在**密钥(secrets)解析**这条路径上体现得最充分。`server-startup-config.ts` 处理了一个棘手场景:运行中,某个密钥提供方临时挂了,导致密钥解析失败。系统怎么办?

```typescript
const handleSecretsActivationError = (err, activationParams, eventConfig): never => {
  const details = String(err);
  if (!secretsDegraded) {
    params.logSecrets.error?.(`[SECRETS_RELOADER_DEGRADED] ${details}`);
    if (activationParams.reason !== "startup") {
      params.emitStateEvent("SECRETS_RELOADER_DEGRADED",
        `Secret resolution failed; runtime remains on last-known-good snapshot. ${details}`,
        eventConfig);
    }
  }
  secretsDegraded = true;
  if (activationParams.reason === "startup") {
    // 启动期密钥就拿不到,那是硬失败——没有 last-known-good 可退
    throw new Error(`Startup failed: required secrets are unavailable. ${details}`, { cause: err });
  }
  throw err;  // 运行期:抛错让上层维持旧快照,但进程不下线
};
```

这里有一个非常成熟的区分:

- **启动期(`reason === "startup"`)密钥失败 = 硬失败、直接报错退出。** 因为此刻还没有任何"已知良好"的运行态可退守——连第一份配置都立不起来,继续跑下去只会是错的。
- **运行期密钥失败 = 软降级(degraded)。** 系统进入 `secretsDegraded` 状态,发出 `SECRETS_RELOADER_DEGRADED` 事件,**但运行时继续用 last-known-good 的密钥快照服务**,不下线。

而当密钥提供方恢复后,系统会发出一个对称的恢复事件:

```typescript
if (secretsDegraded) {
  const recoveredMessage =
    "Secret resolution recovered; runtime remained on last-known-good during the outage.";
  params.logSecrets.info(`[SECRETS_RELOADER_RECOVERED] ${recoveredMessage}`);
  params.emitStateEvent("SECRETS_RELOADER_RECOVERED", recoveredMessage, prepared.config);
}
secretsDegraded = false;
```

`DEGRADED` 与 `RECOVERED` 成对出现,把"我降级了→我恢复了"这个生命周期变成**可观测的状态事件**。这正是全书反复强调的"可观测的生命周期":一次故障不是悄无声息地发生又消失,而是有明确的开始信号、维持期间的明确行为(停在 last-known-good)、和明确的结束信号。

> **设计观察:可恢复性 = 永远有一个"已知良好"的退守点 + 退守期间仍能服务。** 很多系统的"容错"止步于"出错时报个错",但 OpenClaw 的 last-known-good 机制做到了"出错时**继续用上一份能用的**"。区别在于:前者把故障传导给用户,后者把故障**吸收**在系统内部,只留下一条可观测的降级事件。对一个 7×24 的 agent 运行时来说,这是"能跑住"和"动不动就崩"的分水岭。

---

## 11.7 一条配置变更的完整旅程

把这一章串起来,跟着一次"用户改了 `agents.defaults.model`"走完全程:

1. **发现**:用户编辑配置文件保存。chokidar 的 `awaitWriteFinish` 等文件稳定后触发 `change` 事件,`schedule()` 启动 300ms 防抖定时器。
2. **去抖与读取**:300ms 内没有新变化,`runReload` 触发。读取快照,发现 hash 与 `lastAppliedWriteHash` 不同(不是进程内自己写的),继续。
3. **守门**:快照 `exists && valid` —— 通过守门。(若非法,这里就会 `skip` 并保持 last-known-good。)
4. **定位**:`diffConfigPaths(prev, next)` 递归比较,因为只改了一个键,绝大部分子树引用相等被跳过,最终产出 `["agents.defaults.model"]`。
5. **分类**:`buildGatewayReloadPlan` 用规则表匹配,命中 `{ prefix: "agents.defaults.model", kind: "hot", actions: ["restart-heartbeat"] }`,生成计划 `{ restartHeartbeat: true, hotReasons: ["agents.defaults.model"], restartGateway: false }`。
6. **执行**:reload mode 是默认的 `hybrid`,计划不需要整体重启 → 走 `onHotReload`,只重启心跳子系统。其余几十个会话、十几条 channel **毫发无伤**。
7. **晋升**:热重载成功,`promoteAcceptedSnapshot(snapshot, "valid-config")` 把这份新配置晋升为新的 last-known-good。

整个过程:毫秒级生效、零会话中断、新配置即刻成为下一次故障的退守点。而如果用户改的是 `models.pricing`(规则表里是 `restart`),第 5 步分类会得出 `restartGateway: true`,第 6 步就走 `queueRestart`——付出重启代价,但这是为了正确性必须付的。**分类表替用户做出了"这次值不值得重启"的判断。**

---

## 本章小结

这一章我们拆解了 OpenClaw 如何让一个长跑进程安全地吞下配置变更。几条主线:

1. **双通道发现 + 内容指纹去重。** 文件 watcher(带 `awaitWriteFinish`)和进程内写通知并行盯着配置,用 `lastAppliedWriteHash` 按内容(而非事件次序)去重,防抖把一阵抖动收成一次重载,串行化保证不可重入又不丢变更。
2. **`diffConfigPaths` 精确定位。** 引用相等提前剪枝、键取并集发现增删、数组用结构相等比较——把两份配置压成一串精确的点分路径,作为后续决策的输入。
3. **三分类规则表是核心。** `config-reload-plan.ts` 把每个配置前缀映射到 `restart` / `hot` / `none`;`hot` 携带具体动作(重启心跳/cron/health-monitor、销毁 MCP 运行时等)。规则有序匹配,具体前缀在前;**匹配不到一律重启**(fail-safe default)。插件能在核心未占用的命名空间补充规则,但无法覆盖核心分类。
4. **四种 reload mode。** `off`/`restart`/`hot`/`hybrid`,默认 `hybrid` 忠实执行分类表,在灵敏与安全间取最优折中;写入方意图(skip/必须重启)可与静态分类协同;重启失败被隔离,不拖垮 reloader。
5. **last-known-good 是可恢复性的支点。** 坏配置/缺文件被守门人挡下、只跳过不崩溃;只有被接受的快照才晋升为 last-known-good。密钥运行期失败软降级(停在旧快照继续服务)、启动期失败硬退出,`DEGRADED`/`RECOVERED` 成对的可观测状态事件让降级有始有终。

如果说第 10 章的网关讲的是"请求如何安全地收敛",这一章讲的就是"配置如何安全地流动":同样的 fail-safe default(未知→最保守动作)、同样的"插件能扩展不能削弱核心"、同样的"错误即文档"(计划里的 `restartReasons`、降级事件里的 details),只是这次它们服务于一个更刁钻的目标——**让系统在改变自己的同时,不把自己改坏。**

---

## 动手实验

> 以下实验基于本章引用的源码文件。建议在 OpenClaw 工作副本(`git clone https://github.com/openclaw/openclaw.git`)里进行。

### 实验一:给规则表做一次"动作普查"

打开 `src/gateway/config-reload-plan.ts`,通读 `BASE_RELOAD_RULES` 与 `BASE_RELOAD_RULES_TAIL`。

- 列一张表:把所有 `kind: "restart"` 的前缀挑出来(如 `models.pricing`、`plugins.load`、`plugins.installs`、`gateway`、`discovery`),思考它们的共性——为什么这些"必须重启"?
- 再把所有 `kind: "hot"` 且带 `restart-heartbeat` 动作的前缀挑出来(`models`、`agents.defaults.model`、`agents.list`、`agent.heartbeat` 等),思考:为什么改"模型/智能体相关"配置都要重启心跳?
- 进阶:找出 `hooks.gmail` 和 `hooks` 这一对,验证"具体前缀必须排在宽泛前缀之前"。如果把它们调换顺序,改 `hooks.gmail` 时会发生什么?(提示:看 `matchRule` 的"首个命中即返回"逻辑。)

### 实验二:推演 `diffConfigPaths` 的剪枝

阅读 `src/gateway/config-diff.ts` 的 `diffConfigPaths`。

- 纸上构造两份配置:`prev = { a: { b: 1, c: 2 }, d: [1,2,3] }`,`next = { a: { b: 1, c: 99 }, d: [1,2,3] }`。手动推演:函数会下钻哪些子树、跳过哪些、最终返回什么?(答案应是 `["a.c"]`。)
- 关键验证:`d` 数组虽然 `prev.d !== next.d`(引用不同),但 `isDeepStrictEqual` 判定结构相等,所以**不**报告为变化。把这一行删掉会怎样?
- 再构造一个"删除键"的场景:`prev = { x: 1 }`,`next = {}`。验证"键取并集"为什么能发现 `x` 被删了。

### 实验三:对照四种 reload mode 的决策树

阅读 `src/gateway/config-reload.ts` 的 `applySnapshot` 决策树和 `config-reload-settings.ts`。

- 假设变化路径是 `["models.pricing"]`(分类为 `restart`)。分别推演四种 mode 下系统的行为:`off`(?)、`restart`(?)、`hot`(?)、`hybrid`(?)。
- 重点体会 `hot` 模式:遇到本该重启的变化,它**拒绝并警告**而不是偷偷重启。为什么这个"宁可不生效也不偷偷重启"的保证对某些用户很重要?
- 再假设变化路径是 `["cron.jobs.daily"]`(分类为 `hot`),四种 mode 又各自如何?

### 实验四:验证 last-known-good 的"只晋升被接受的快照"

阅读 `runReload` 里 `applySnapshot` 与 `promoteAcceptedSnapshot` 的调用顺序,以及 `handleInvalidSnapshot`。

- 画一条时间线:t0 应用了合法配置 A(晋升 A 为 last-known-good)→ t1 用户写入**非法**配置 B → t2 用户修好写入合法配置 C。逐步标注:每个时刻 last-known-good 是什么?运行时实际在用哪份配置?
- 确认 t1 时刻:`handleInvalidSnapshot` 返回 `true` 导致 `runReload` 提前 `return`,**既没 apply 也没 promote**,运行时维持在 A。
- 进阶:对照 `server-startup-config.ts` 的 `handleSecretsActivationError`,区分"启动期密钥失败(硬退出)"与"运行期密钥失败(软降级停在 last-known-good)"。思考:为什么这两个时机要用截然不同的失败策略?

---

> 下一章,我们将把视线从"单个进程内的配置流动"扩展到"跨进程、跨设备的信任建立"——OpenClaw 的**节点(node)与设备配对(pairing)**。还记得第 10 章方法表里那一整片 `operator.pairing` scope 的方法(`node.pair.approve`、`device.token.rotate`、`device.token.revoke`)吗?当一台新设备或一个远程节点想接入这条网关总线时,它要如何证明自己可信、令牌如何轮换与吊销、配对如何在"方便"与"安全"之间取舍——又一次围绕信任边界的精心设计。
