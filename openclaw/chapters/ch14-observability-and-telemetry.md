# 第 14 章 可观测性与遥测——把"到底发生了什么"安全地说清楚

前十三章里,我们反复遇到一类东西:第 4 章 agent loop 里广播的事件、第 8 章沙箱里的执行日志、第 10 章网关的路由记录、第 11 章配置热重载时打出的 `Unrecognized key` 诊断、第 12 章配对失败时的拒绝原因、第 13 章供应链审计吐出的一条条 finding。它们有一个共同的归宿:被采集、被写进日志、被导出成支持包,最终成为运维在深夜排障时唯一能依赖的东西。

可观测性听上去是个事后话题——"等出了问题再看日志"。但 OpenClaw 把它当成和安全边界同等级的设计约束放进了骨架里。原因很简单:**一个能自动跑、能调外部工具、能连远程节点的 agent,它的日志里天然会流过 token、密码、私钥、信用卡号。** 如果可观测性是事后补的,那它补进来的第一天就是数据泄露的第一天。所以这一章的主角不是"如何打日志",而是一个更尖锐的矛盾:**怎样在把事情说清楚的同时,不把不该说的东西也说出去。**

我们会沿着一条数据流走:一段原始文本(可能含密钥)→ 脱敏引擎把秘密抹掉但保留可读性 → 对于需要关联但不能明文的标识符,用哈希换出一个稳定的代号 → 遥测开关决定要不要把聚合数据发出去 → 指标采集时把被污染的数字挡在门外 → 日志分级控制噪声。每一环都在回答同一个问题:**这条信息,导出去安全吗?**

## 14.1 脱敏不是开关,而是默认态

先看一个容易被忽略的设计决定:脱敏的默认模式不是"关",而是"tools"。

```ts
const DEFAULT_REDACT_MODE: RedactSensitiveMode = "tools";
const DEFAULT_REDACT_MIN_LENGTH = 18;
const DEFAULT_REDACT_KEEP_START = 6;
const DEFAULT_REDACT_KEEP_END = 4;
export type RedactSensitiveMode = "off" | "tools";
```

`RedactSensitiveMode` 只有两个值:`"off"` 和 `"tools"`。注意 `normalizeMode` 的写法——它不是"识别已知值,否则报错",而是**只有显式传 `"off"` 才关,其余一切(包括 `undefined`、拼错的值、空字符串)都回落到默认的 tools 模式**:

```ts
function normalizeMode(value?: string): RedactSensitiveMode {
  return value === "off" ? "off" : DEFAULT_REDACT_MODE;
}
```

这又是一次我们在前面章节反复见到的 fail-closed:**配置缺失或写错,系统选择"更安全"的那一边。** 在第 11 章里,配置错误会让功能默认关闭;在第 12 章里,未配置等于不批准配对;这里则是——你没法因为漏配一个开关就意外把密钥打进日志。要关闭脱敏,你必须明明白白地写下 `"off"`,这个动作本身就是一份审计记录:有人主动放弃了保护。

## 14.2 一份"秘密长什么样"的目录

脱敏引擎的核心是一张很长的模式表 `DEFAULT_REDACT_PATTERNS`。它不是泛泛地"找像密钥的字符串",而是把秘密会出现的**语法位置**逐一列举出来。粗略归类:

- **环境变量赋值**:`API_KEY=xxx`、`DB_PASSWORD: yyy`,以及被 shell 转义过的版本。
- **URL 查询参数**:`?access_token=...`、`&signature=...`。
- **JSON 字段**:`"apiKey": "..."`、`"refreshToken": "..."`。
- **HTTP 客户端诊断对象**:很多 HTTP 库报错时会把请求配置 `util.inspect` 出来,字段长这样 `clientSecret: '...'`。
- **请求头**:`authorization`、`cookie`、`x-api-key`、`x-auth-token`。
- **CLI 参数**:`--token xxx`、`--api-key yyy`。
- **Authorization 头**:`Bearer <jwt>`、`Basic <base64>`。
- **PEM 私钥块**:`-----BEGIN ... PRIVATE KEY-----` 到 `-----END ...`。
- **一长串带前缀的 token**:`sk-`、`ghp_`、`github_pat_`、`xoxb-`、`xapp-`、`gsk_`、`AIza`、`ya29.`、JWT 三段式 `eyJ...`、`pplx-`、`npm_`、`AKID`、`LTAI`、`hf_`、`r8_`,乃至 Telegram bot token 的 `/bot<digits>:<rest>` 形态。

值得停下来看一条注释,它解释了为什么环境变量这条模式**故意保持大小写敏感**:

```ts
// ENV-style assignments. Keep this case-sensitive so diagnostics like
// `Unrecognized key: "llm"` do not lose the actual config key.
```

这里藏着一个微妙的平衡。如果脱敏太激进,把所有 `xxx=yyy` 都抹掉,那第 11 章里那句救命的诊断 `Unrecognized key: "llm"` 就会变成 `Unrecognized key: "***"`——错误信息被脱敏吃掉了,运维反而看不懂哪里配错了。所以脱敏的目标从来不是"抹得越多越好",而是**精确命中秘密、放过诊断**。把"错误即文档"的原则(我们从第 1 章一路讲到现在)和数据安全放在同一架天平上,这条注释就是那个支点。

为了让这张大表不至于拖慢每一行普通日志,引擎前面还挡了一道**预过滤**:

```ts
function couldMatchDefaultRedactPatterns(text: string): boolean {
  return DEFAULT_REDACT_PREFILTER_RE.test(text);
}
```

`DEFAULT_REDACT_PREFILTER_RE` 是一个粗粒度的"可能含秘密吗"的快速判断(命中 `KEY|TOKEN|SECRET|PASSWORD|...` 等关键词或已知前缀)。一段普通文本如果连这道粗筛都过不了,就直接原样返回,完全不去跑那几十条精细正则。这是典型的"快路径优先":绝大多数日志行不含秘密,它们应该几乎零成本地通过。

## 14.3 maskToken:抹掉,但留个手把手

脱敏命中之后并不是简单替换成 `***`,而是经过 `maskToken`:

```ts
function maskToken(token: string): string {
  if (token.length < DEFAULT_REDACT_MIN_LENGTH) {
    return "***";
  }
  const start = token.slice(0, DEFAULT_REDACT_KEEP_START);
  const end = token.slice(-DEFAULT_REDACT_KEEP_END);
  return `${start}…${end}`;
}
```

短于 18 字符的,整个抹成 `***`;足够长的,保留前 6 后 4,中间用省略号。为什么不一律抹光?因为运维排障时常常需要**区分"是哪一个 token"**:同一个服务可能配了好几把密钥,日志里如果全是 `***`,你根本不知道报错的是哪一把;但如果看到 `ghp_Fk…ltf`,你就能和你手上的密钥前后缀对一下,确认是不是过期的那一把。保留前后缀既泄露不了秘密(中间几十位才是熵的来源),又给了排障一个可对照的指纹。这是"可读性"与"安全"之间一个非常实用的折中。

PEM 私钥块的处理也遵循同样的哲学——保留头尾,中间抹掉:

```ts
function redactPemBlock(block: string): string {
  const lines = block.split(/\r?\n/).filter(Boolean);
  if (lines.length < 2) {
    return "***";
  }
  return `${lines[0]}\n…redacted…\n${lines[lines.length - 1]}`;
}
```

你还能看出"这是一个 RSA/EC 私钥块"(`-----BEGIN RSA PRIVATE KEY-----`),但拿不到密钥本身。结构信息保留,机密内容抹除。

## 14.4 两个不能误伤的特例:shell 引用与配置键

脱敏最难的部分不是"抹掉秘密",而是"不要把不是秘密的东西也抹掉"。`redact.ts` 里专门处理了两个高频误伤场景。

**第一个是 shell 变量引用。** 考虑这样一行命令:`API_KEY=$API_KEY`。等号右边的 `$API_KEY` 不是密钥,它只是一个变量引用——把它抹成 `API_KEY=***` 反而会误导读者以为这里有个明文密钥。引擎用 `isShellReferenceToKey` 识别这种情况:

```ts
function isShellReferenceToKey(key: string, value: string): boolean {
  if (!/^[A-Z_][A-Z0-9_]*$/.test(key)) {
    return false;
  }
  const bare = value.match(/^\$([A-Z_][A-Z0-9_]*)$/);
  if (bare) {
    return bare[1] === key;
  }
  const braced = value.match(/^\$\{([A-Z_][A-Z0-9_]*)(?::[-=?+])?\}$/);
  return braced?.[1] === key;
}
```

只有当右值恰好是 `$KEY` 或 `${KEY}`(且引用的变量名和左边的键一致)时,才判定为引用而非秘密,原样保留。这个保留逻辑只对几个特定模式生效——它们被登记在 `SHELL_REFERENCE_PRESERVING_PATTERN_SOURCES` 里,通过一个 `WeakSet` 标记,确保只有 env 赋值类的模式才会启用这条豁免。

**第二个是配置键诊断**,也就是上一节那条大小写敏感的注释要保护的东西。两个特例合起来传递的是同一个工程价值观:**脱敏要足够聪明,聪明到能分清"这是秘密"和"这只是看起来像秘密"。** 一个只会无脑抹的脱敏器,最终会让所有日志都变成无法排障的 `***` 噪声,逼得运维干脆把脱敏关掉——那才是真正的安全灾难。

## 14.5 结构化对象的递归脱敏

日志里不只有字符串,还有大量被序列化的对象:工具调用的参数、HTTP 请求体、配置快照。对这些,引擎提供了递归脱敏 `redactStructuredSecretValue`,它沿着对象/数组一路下钻,对每个键值对判断敏感性:

```ts
if (Array.isArray(value)) {
  if (seen.has(value)) {
    return "[Circular]";
  }
  seen.add(value);
  const out = value.map((entry) => redactStructuredSecretValue(key, entry, seen, options));
  seen.delete(value);
  return out;
}
```

几个细节很能说明"防御性"这三个字:

- **循环引用保护**:用一个 `seen` WeakSet 跟踪正在处理的对象,一旦发现回环就返回 `"[Circular]"`,绝不让脱敏过程自己陷入死循环。一个会因为日志对象自引用就把整个进程卡死的脱敏器,是不可接受的。
- **只脱敏"纯对象"**:`isPlainRedactableObject` 检查 `prototype === Object.prototype || prototype === null`。带原型链的类实例(比如某个内部对象、Error 子类)被原样跳过——递归进一个不受控的类实例去改写它的字段,可能触发 getter 副作用或破坏不变量,代价大于收益。
- **键名也是信号**:`isSensitiveFieldKey` 用两套正则(普通字段名 + 全大写环境变量名)判断这个键本身是否敏感。即使值看起来"不像"秘密,只要键名叫 `password`、`apiKey`、`clientSecret`,值也会被 mask。

## 14.6 app-specific password:在歧义里做判断

有一类秘密特别难处理:Apple 的 app-specific password 长成 `xxxx-xxxx-xxxx-xxxx` 这样四段小写字母。问题是,**很多正常单词组合也长这样**——`open-claw-demo-test`、`main-path-name-file` 都符合这个形状。如果一律抹掉,会误伤大量正常文本。

引擎的解法是维护一张"良性词表" `BENIGN_APP_PASSWORD_WORDS`(`case`、`claw`、`demo`、`file`、`main`、`name`、`open`、`path`、`slug`、`test`),然后:

```ts
function looksLikeAppSpecificPassword(candidate: string): boolean {
  return candidate.split("-").every((part) => !BENIGN_APP_PASSWORD_WORDS.has(part.toLowerCase()));
}
```

只有当四段**全都不是**常见英文词时,才判定为真正的密码并抹除。这是一个坦诚的工程权衡:形状歧义无法 100% 消解,与其追求完美,不如用一张小词表把最常见的误伤挡掉,接受"极小概率漏判/误判"的残余风险。可观测性的脱敏从来不是数学证明,而是不断收窄误差的实践。

## 14.7 ReDoS 防护:别让一行日志卡死事件循环

上面那几十条正则,绝大多数在普通长度的文本上飞快。但日志里偶尔会出现超长字符串——一个被完整 dump 的 HTTP 响应、一份几兆字节的支持导出。某些正则在超长输入上可能退化(灾难性回溯,ReDoS),把单线程的 Node 事件循环卡死。`redact-bounded.ts` 专门防这个:

```ts
const REDACT_REGEX_CHUNK_THRESHOLD = 32_768;
const REDACT_REGEX_CHUNK_SIZE = 16_384;
```

`replacePatternBounded` 的逻辑是:文本短于阈值(32KB)就正常一次性替换;一旦超过,就切成 16KB 的块逐块替换,避免在一整段巨大字符串上跑正则:

```ts
let output = "";
// Chunking may miss matches spanning chunk boundaries; use only for token-like redaction patterns.
for (let index = 0; index < text.length; index += chunkSize) {
  output += text.slice(index, index + chunkSize).replace(pattern, replacer);
}
return output;
```

注意那条诚实的注释:**分块可能漏掉跨块边界的匹配,所以这招只用于"token 状"的短模式。** 一个 token 长度有限,被切到两块边界上的概率极低;而像 PEM 块那种可能横跨很长的多行模式,则不会走分块路径(它们在阈值以内时由正常路径处理)。这是一个清醒的取舍:用"极小概率漏一个 token"换"绝不卡死事件循环"。对一个长期运行、夜里也要自动干活的 agent 来说,**可用性本身就是一种安全属性**——一个被日志卡死的进程,等于失去了全部可观测性。

## 14.8 关联但不泄露:哈希出一个稳定代号

排障经常需要"把同一个用户/会话/设备的多条日志串起来"。但用户 ID、邮箱、设备指纹这些标识符本身往往是敏感的,不能明文落盘。`redact-identifier.ts` 给出的答案是:**用 sha256 前缀换出一个稳定但不可逆的代号。**

```ts
export function sha256HexPrefix(value: string, len = 12): string {
  const safeLen = Number.isFinite(len) ? Math.max(1, Math.floor(len)) : 12;
  return crypto.createHash("sha256").update(value).digest("hex").slice(0, safeLen);
}

export function redactIdentifier(value: string | undefined, opts?: { len?: number }): string {
  const trimmed = normalizeOptionalString(value);
  if (!trimmed) {
    return "-";
  }
  return `sha256:${sha256HexPrefix(trimmed, opts?.len ?? 12)}`;
}
```

注释一句话点透:"Returns a stable sha256 hex prefix for non-secret identifier correlation."(为非秘密标识符的关联返回稳定的 sha256 前缀。)关键词是 **stable(稳定)**:同一个用户 ID 每次都哈希出同一个 `sha256:a1b2c3...`,于是运维能在日志里把它们串成一条线;但因为是单向哈希,谁也无法从 `sha256:a1b2c3` 反推回真实 ID。空值统一返回 `"-"`,保证日志格式整齐、不会出现 `undefined`。

这其实和第 12 章里"token 即身份"的思路是一脉相承的:**让一个不透明的代号承担"标识"职责,而把"内容"本身保护起来。** 第 12 章用它来做鉴权,这一章用它来做关联——同一个密码学原语,在系统不同位置反复复用。

## 14.9 遥测开关:环境变量盖过持久化设置

不是所有遥测都会无条件上报。安装遥测有一个明确的开关 `isInstallTelemetryEnabled`,它的优先级设计值得一看:

```ts
export function isInstallTelemetryEnabled(
  settingsManager: SettingsManager,
  telemetryEnv: string | undefined = process.env.OPENCLAW_TELEMETRY,
): boolean {
  return telemetryEnv !== undefined
    ? isTruthyEnvFlag(telemetryEnv)
    : settingsManager.getEnableInstallTelemetry();
}
```

文件头注释解释了为什么:"Environment overrides win over persisted settings for CI and packaged launcher control."(环境变量盖过持久化设置,以便 CI 和打包启动器控制。)逻辑是:**只要设了 `OPENCLAW_TELEMETRY` 环境变量,它就说了算(无论开还是关);没设,才回落到用户持久化的设置。** 这让运维和打包方有一个不需要改用户配置文件就能强制开关遥测的旋钮——CI 里可以一键关掉避免噪声,打包发行版可以按合规要求统一控制。`isTruthyEnvFlag` 也很克制,只认 `"1"`、`"true"`、`"yes"`(大小写不敏感),其余一律视为假,避免"设了个奇怪的值反而误开"。

## 14.10 指标卫生:别让上游的脏数据污染聚合

可观测性链路的最后一环是指标。这里有个容易被忽视的攻击面:**很多指标来自 provider 自己的 JSON 载荷,而这些载荷是不可信的。** 如果上游因为 bug 报回一个 `-1`、`NaN` 或 `Infinity`,而你不加检查地把它累加进聚合指标,整张监控面板就被一个上游 bug 毁了。`event-metrics.ts` 在入口就把这种脏数据挡掉:

```ts
export function firstFiniteTalkEventNumber(
  record: Record<string, unknown> | undefined,
  keys: readonly string[],
): number | undefined {
  if (!record) {
    return undefined;
  }
  for (const key of keys) {
    const value = record[key];
    if (typeof value === "number" && Number.isFinite(value) && value >= 0) {
      // Reject negative, NaN, and Infinity values before diagnostics/logging so
      // provider bugs cannot poison aggregate Talk metrics.
      return value;
    }
  }
  return undefined;
}
```

三重门:必须是 `number` 类型、必须是有限数(挡掉 `NaN`/`Infinity`)、必须非负。文件头那句话把意图说得很白:"Talk event payloads are provider-owned JSON blobs, so callers must coerce records and read only bounded numeric counters that are safe to export."(Talk 事件载荷是 provider 自己的 JSON,调用方必须强制转换、只读取可安全导出的有界数值计数器。)这又是我们熟悉的边界思维——**把外部数据当成不可信输入,在它进入你的聚合之前就验证好。** 第 7 章工具系统校验工具入参、第 13 章把 provider 的 JSON 当不可信对待,到这里指标采集也是同一条原则的延续。

## 14.11 日志分级:控制噪声的最后一道闸

最后是最基础也最实用的一环——日志级别。`levels.ts` 定义了标准的级别阶梯:

```ts
export const ALLOWED_LOG_LEVELS = [
  "silent", "fatal", "error", "warn", "info", "debug", "trace",
] as const;
```

`tryParseLogLevel` 只接受白名单内的值(`trim` 后比对),拼错的级别名一律返回 `undefined`;`normalizeLogLevel` 在解析失败时回落到 `"info"`。`levelToMinLevel` 把级别映射成 tslog 的数值阈值:

```ts
const map: Record<LogLevel, number> = {
  trace: 1, debug: 2, info: 3, warn: 4, error: 5, fatal: 6,
  silent: Number.POSITIVE_INFINITY,
};
```

注意 `silent` 被映射成 `+Infinity`——因为 tslog 的过滤规则是"logLevelId 小于 minLevel 就丢弃",把阈值设成正无穷,等于丢弃一切日志。用一个数学上的极值来表达"全部静音",干净利落,不需要任何特判分支。这是一个小而优雅的设计:**用类型系统和数值语义本身去表达意图,而不是堆叠 if-else。**

日志分级看似平淡,但它是可观测性链路的"音量旋钮":生产环境跑 `info`,排障时调到 `debug`/`trace`,CI 里可能要 `silent`。它和前面所有脱敏、哈希、指标卫生的机制配合起来,才构成一条完整的链路:**该说的说清楚(分级 + 诊断保留),不该说的抹干净(脱敏),需要关联的换代号(哈希),不可信的挡门外(指标卫生),要不要发出去由开关定(遥测)。**

## 本章小结

这一章我们沿着一条数据流,看完了 OpenClaw 把"到底发生了什么"安全地说清楚的全过程:

1. **脱敏是默认态而非开关**:`DEFAULT_REDACT_MODE` 是 `"tools"`,`normalizeMode` 只有显式 `"off"` 才关——又一次 fail-closed,漏配不会泄密。
2. **秘密目录按语法位置枚举**:`DEFAULT_REDACT_PATTERNS` 覆盖 env/URL/JSON/header/CLI/PEM/各类 token 前缀,前置 `DEFAULT_REDACT_PREFILTER_RE` 做快路径跳过;env 模式故意大小写敏感,以免吃掉 `Unrecognized key` 这类诊断。
3. **maskToken 留指纹不留秘密**:保留前 6 后 4(≥18 字符时),短的抹成 `***`,PEM 保留头尾——可读性与安全的实用折中。
4. **两个不能误伤的特例**:shell 变量引用(`$KEY`/`${KEY}`)和配置键诊断都被特意保护,脱敏要分得清"是秘密"和"看起来像秘密"。
5. **结构化递归脱敏带防御**:循环引用返回 `[Circular]`、只处理纯对象、键名本身也作为敏感信号。
6. **歧义场景靠词表收窄**:app-specific password 用 `BENIGN_APP_PASSWORD_WORDS` 避免误伤正常单词组合。
7. **ReDoS 防护**:`replacePatternBounded` 在 32KB 以上分块替换,用"极小概率漏 token"换"绝不卡死事件循环"——可用性即安全。
8. **关联不泄露**:`redactIdentifier` 用稳定的 sha256 前缀(`sha256:<12hex>`)换出可串联但不可逆的代号。
9. **遥测开关**:`OPENCLAW_TELEMETRY` 环境变量盖过持久化设置,给 CI 和打包方一个不动用户配置的旋钮。
10. **指标卫生**:`firstFiniteTalkEventNumber` 三重门挡掉负数/NaN/Infinity,不让上游 bug 污染聚合。
11. **日志分级**:白名单解析 + 回落 `info`,`silent` 映射成 `+Infinity` 优雅表达"全静音"。

贯穿全章的,是同一个我们从第 1 章追到现在的工程哲学:**边界处对一切外部输入保持不信任,默认选更安全的那一边,但绝不为了安全而牺牲掉系统本该提供的可读性与可用性。** 可观测性不是事后补的日志,而是一开始就和安全边界长在一起的骨架。

## 动手实验

> 以下实验基于本章引用的源码文件,建议在你 clone 的 OpenClaw 仓库里对照阅读与验证。

**实验一:验证"漏配不泄密"。**
读 `src/logging/redact.ts` 的 `normalizeMode` 与 `resolveRedactOptions`。假设有人把脱敏配置项写成 `redactSensitive: "Off"`(大写 O)或 `redactSensitive: "disable"`,脱敏会被关掉吗?沿着 `normalizeMode` 走一遍给出结论。然后反过来:要真正关闭脱敏,配置必须写成什么?解释为什么这个"必须精确写对才能关"的设计本身就是一种安全属性,并把它和第 11 章配置错误默认关闭功能、第 12 章未配置等于不批准做横向对比。

**实验二:制造一次脱敏误伤,再修好它。**
读 `maskToken`、`DEFAULT_REDACT_PATTERNS` 里的 env 赋值模式,以及那条"保持大小写敏感"的注释。构造一行诊断文本 `Unrecognized key: "OPENAI_API_KEY_PATH"`,推演:如果 env 模式改成大小写不敏感且更激进地匹配,这行会被脱敏成什么?诊断还能用吗?再构造一行 `export GITHUB_TOKEN=ghp_xxxxxxxxxxxxxxxxxxxx`,确认它会被正确 mask 成什么形状(前几位 + 省略号 + 后几位)。用这两个例子说明"精确命中秘密、放过诊断"是如何同时满足的。

**实验三:跑通一次结构化递归脱敏。**
读 `redactStructuredSecretValue` 和 `isPlainRedactableObject`。给定输入对象 `{ apiKey: "sk-abcdefghijklmnop", note: "hello", nested: { password: "p", arr: [1, 2] } }`,逐字段推演输出。然后做两个变体:(a) 把对象改成一个 `Error` 实例(带原型链)再传进去,会发生什么?为什么 `isPlainRedactableObject` 要把它跳过?(b) 构造一个自引用对象 `const o = {}; o.self = o;`,说明 `seen` WeakSet 如何阻止无限递归、最终输出里 `self` 变成什么。

**实验四:理解分块脱敏的边界取舍。**
读 `src/logging/redact-bounded.ts` 的 `replacePatternBounded` 和那条"chunking may miss matches spanning chunk boundaries"的注释。构造一个长度刚好跨过 32768 阈值的字符串,并把一个 `ghp_` token 故意放在第 16384 字节的边界上(让它被切成两半)。推演:这个 token 会被脱敏吗?为什么作者认为这个风险可以接受?再回答:如果有人想"修复"这个漏匹配,改成不分块、一次性替换,会引入什么更严重的问题?把你的结论和"可用性本身就是一种安全属性"这句话联系起来。

## 下一章预告

我们已经看清了 OpenClaw 如何把运行时发生的一切**安全地记录下来**。但记录只是起点——这些脱敏过的日志、哈希过的标识符、清洗过的指标,最终要被打包成可以交给别人(同事、支持工程师、issue 区)的东西,而"交出去"这个动作本身又是一道新的安全闸门:导出包里会不会夹带没被脱敏干净的快照?会不会把整个配置原样塞进去?第 15 章我们将走进 OpenClaw 的**诊断与支持导出(diagnostics & support export)**:当用户运行一条命令把"现场"打包成一个 bundle 时,系统是如何在收集足够多信息(让问题可复现)和不泄露任何敏感数据(让 bundle 可以安全分享)之间走钢丝的?我们会看到本章的脱敏引擎在这里被推到极限——它不再只是过滤一行日志,而要为一整份可能落到陌生人手里的"系统快照"负责。
