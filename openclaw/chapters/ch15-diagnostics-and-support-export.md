# 第 15 章 诊断与支持导出——把"现场"打包成一个能安全交出去的盒子

第 14 章我们看完了 OpenClaw 如何把运行时发生的一切**安全地记录下来**:脱敏过的日志、哈希过的标识符、清洗过的指标。但记录只是中间态。真正的考验在最后一步——当一个用户在 issue 区贴出"我的 agent 半夜跑崩了",维护者需要的是一份**完整的现场**:配置长什么样、日志里有什么、网关状态如何、稳定性快照里记了什么事件。用户得有办法把这些一键打包,交给一个素未谋面的陌生人。

这个动作本身就是一道全新的、也是最危险的安全闸门。前面所有的脱敏都是"过滤一行日志";到了这里,脱敏要为**一整份可能落到任何人手里的系统快照**负责。漏过一个 token、把整份配置原样塞进去、让日志里的聊天正文跟着流出去——任何一个疏漏,都是把用户的密钥、隐私、对话内容直接公开。

所以这一章的核心矛盾,比上一章更尖锐:**怎样在收集到足够多信息(让问题可复现)和不泄露任何敏感数据(让 bundle 可以安全分享)之间走钢丝。** 我们会看到 OpenClaw 的答案是一套分层防御:从"默认不收原始载荷"的总方针,到字段级的白名单/黑名单,到路径前缀替换,到 zip 写入时的路径逃逸防护,再到一份写给维护者也写给用户的隐私清单。每一层都在回答同一个问题:**这份盒子,交出去之后,我还能不能睡得着觉。**

## 15.1 总方针:payload-free,而且写在脸上

先看导出产物的 manifest 里有什么。`buildDiagnosticSupportExport` 最终拼出的清单里,有一个 `privacy` 字段:

```ts
privacy: {
  payloadFree: true,
  rawLogsIncluded: false,
  notes: [
    "Stability bundles are payload-free diagnostic snapshots.",
    "Logs keep operational summaries and safe metadata fields; payload-like fields are omitted.",
    "Status and health snapshots redact secrets, payload-like fields, and account/message identifiers.",
    "Config output includes useful settings with credentials, private identifiers, and prompt text redacted.",
  ],
},
```

注意这两个布尔字段的类型——在 manifest 的类型定义里,它们不是 `boolean`,而是**字面量类型** `payloadFree: true` 和 `rawLogsIncluded: false`:

```ts
privacy: {
  payloadFree: true;
  rawLogsIncluded: false;
  notes: string[];
};
```

这是一个很硬的设计承诺。`payloadFree: true` 不是一个可能为假的开关,而是**类型系统层面写死的不变量**:这份导出永远不含原始载荷。如果哪天有人不小心往里塞了原始日志,TypeScript 会在 manifest 这里就报错——把"不泄露载荷"从一句口头约定,变成了编译器强制的契约。

更值得玩味的是,这份隐私声明不仅在 manifest 里,还原样写进了给人读的 `summary.md`:

```
## Privacy

- raw chat text, webhook bodies, tool outputs, tokens, cookies, and secrets are not included intentionally
- log records keep operational summaries and safe metadata fields
- status and health snapshots redact secret fields, payload-like fields, and account/message identifiers
- config output keeps useful settings but redacts secrets, private identifiers, and prompt text
```

而 summary 的开头一句,直接点出了这份 bundle 的设计目标:"Attach this zip to the bug report. It is designed for maintainers to inspect without asking for raw logs first."(把这个 zip 附到 bug 报告里。它的设计目标是让维护者无需先索要原始日志就能排查。)这句话道出了整个子系统存在的理由:**好的诊断导出能终结"能不能把完整日志发我看看"这种危险的来回索要。** 用户不必再手动复制粘贴可能含密钥的日志,维护者也不必背负"我手里有别人的密钥"的责任。一次安全的、自包含的导出,对双方都是解脱。

## 15.2 两种脱敏哲学:配置 vs 快照

第 14 章的脱敏引擎是"按模式抹秘密"。但支持导出面对的数据形态更复杂:有结构化的配置、有任意形状的状态快照、有逐行的日志。OpenClaw 为此在 `diagnostic-support-redaction.ts` 里写了一套**字段感知**的脱敏,而且对"配置"和"普通快照"用了两种严格程度不同的策略。

普通快照用 `sanitizeSupportSnapshotValue`,它判断字段名是否私密的标准是三类正则的并集:

```ts
function isPrivateSupportField(key: string): boolean {
  return (
    SECRET_SUPPORT_FIELD_RE.test(key) ||   // authorization/cookie/credential/key/password/secret/token
    PAYLOAD_SUPPORT_FIELD_RE.test(key) ||  // body/chat/content/message/payload/prompt/result/text/tool/transcript...
    IDENTIFIER_SUPPORT_FIELD_RE.test(key)  // account_id/chat_id/email/phone/user_id/username...
  );
}
```

这里有个第 14 章没有的维度:**不只抹秘密,还抹"载荷类"字段(`PAYLOAD_SUPPORT_FIELD_RE`)和"身份类"字段(`IDENTIFIER_SUPPORT_FIELD_RE`)。** 因为支持包里,聊天正文(`chat`/`content`/`message`/`transcript`)和用户身份(`email`/`phone`/`user_id`)和密钥一样,都是绝对不能外泄的。注意一个细节——连**数字类型**的私密字段也会被抹:

```ts
if (typeof value === "number") {
  return isPrivateSupportField(key) ? "<redacted>" : value;
}
```

一个叫 `phone` 的字段哪怕值是纯数字,也会变成 `<redacted>`。脱敏不只看值像不像秘密,更看**这个字段在语义上该不该出现**。

配置用更严的 `sanitizeSupportConfigValue`,它的私密判断在上面三类之外又加了一类 `CONFIG_PRIVATE_FIELD_RE`(`allow_from`/`deny_to`/`owner_id`/`sender_id`/`blocked_users`…)。原因很实在:配置里的访问控制列表——谁被允许、谁被拉黑——本身就是敏感的社交图谱。而且对配置里的私密字段,处理得更彻底:

```ts
if (isPrivateConfigField(key)) {
  return isSecretRefShape(record) ? sanitizeSecretRefForSupport(record) : "<redacted>";
}
```

如果一个私密字段恰好是"密钥引用"形态(指向某个 secret 而非内联密钥),它会被 `sanitizeSecretRefForSupport` 处理成只保留 `source`/`provider`、把 `id` 换成 `<redacted>`——**保留"这里引用了一个来自 X 的密钥"这条排障有用的结构信息,但抹掉密钥本体**。这又是我们熟悉的"留结构、去内容"哲学,和第 14 章 PEM 块保留头尾是同一个思路。

## 15.3 私密映射:连键名都不能留

有些数据,光抹掉值还不够,连键名本身都是隐私。比如一个 `users` 映射,它的键可能就是用户 ID 或用户名。对这类字段,`PRIVATE_MAP_SUPPORT_FIELD_RE`(`accounts`/`chats`/`conversations`/`messages`/`threads`/`users`)有专门处理。

在普通快照里,直接退化成只报数量:

```ts
if (PRIVATE_MAP_SUPPORT_FIELD_RE.test(key)) {
  return { count: countOwnObjectEntries(record) };
}
```

一个 `users: { "alice@x.com": {...}, "bob@y.com": {...} }` 变成 `{ count: 2 }`——维护者知道"有两个用户",但拿不到是谁。

在配置里则更精细一点,保留条目数的同时,把键名替换成带序号的占位:

```ts
if (redactEntryKeys) {
  privateEntryIndex += 1;
  outputKey = `<redacted-${privateEntryLabel}-${privateEntryIndex}>`;
}
```

于是 `users` 映射变成 `{ "<redacted-user-1>": {...}, "<redacted-user-2>": {...} }`——维护者能看到"第几个用户的配置长什么样"(结构对排障有用),但键名里的真实身份被序号代号取代。这是在"可排障"与"不泄露"之间又一次精心的折中。

## 15.4 文本级脱敏:一条流水线把各种秘密形态逐个抹掉

字段名能挡住"叫 password 的字段",但秘密也会藏在普通文本里——一段错误消息里夹着的 JWT、一个 URL 里的 userinfo、日志正文里出现的邮箱。`redactTextForSupport` 是一条把这些都过一遍的流水线:

```ts
export function redactTextForSupport(value: string): string {
  let redacted = redactCommonCredentialTextForSupport(value);  // Basic auth / Cookie / AWS key / JWT
  redacted = redactSensitiveTextForSupport(redacted);           // 复用第14章的 redactSensitiveText
  redacted = redactUrlSecretsForSupport(redacted);              // URL userinfo + 敏感 query 参数
  redacted = redactServiceIdentifiersForSupport(redacted);      // Matrix user/room/event id
  redacted = redactContactIdentifiersForSupport(redacted);      // email / @handle
  return redactLongIdentifiersForSupport(redacted);             // 9 位以上纯数字 id
}
```

几个值得注意的点:

- **它复用了第 14 章的引擎**:`redactSensitiveTextForSupport` 内部就是 `redactSensitiveText(value, { mode: "tools" })`。支持导出不是另起炉灶,而是站在上一章那套通用脱敏之上,再叠加面向"分享场景"的额外规则。
- **URL userinfo 单独处理**:`scheme://user:pass@host` 这种把凭据嵌在 URL 里的写法,会被替换成 `scheme://<redacted>:<redacted>@`,既抹掉账号密码,又保留"这是个带认证的 URL"这一结构。
- **它连服务特定标识符都认识**:Matrix 的用户 ID(`@x:y`)、房间 ID(`!x:y`)、事件 ID(`$xxxx`)都有专门的正则。这说明脱敏是**结合 OpenClaw 实际接入的渠道**来设计的,而不是泛泛地抹"看起来像 ID 的东西"。
- **兜底抹长数字**:`LONG_DECIMAL_ID_RE` 把 9 位以上的纯数字串一律换成 `<redacted-id>`,因为电话号码、雪花 ID、Telegram chat id 往往就长这样。这是一道"宁可错杀"的兜底,接受偶尔误伤一个无害的大数字,换取不漏掉身份类长数字。

每个被抹的地方都用了**语义化的占位符**:`<redacted-jwt>`、`<redacted-email>`、`<redacted-aws-key>`、`<redacted-matrix-user>`。维护者看到 `<redacted-jwt>` 就知道"这里原本有个 JWT",比一律 `***` 更有助于理解上下文——又一次"抹内容、留语义"。

## 15.5 路径脱敏:把家目录换成 `~`

诊断里满是文件路径——日志文件路径、配置路径、状态目录。而绝对路径里几乎总含用户名:`/home/alice/.openclaw/...`、`C:\Users\Bob\...`。`redactPathForSupport` 专门把这些已知前缀替换成符号化的标签:

```ts
function pathRedactionPrefixes(options: SupportRedactionContext): PathRedactionPrefix[] {
  const prefixes = new Map<string, PathRedactionPrefix>();
  addPathPrefixVariants(prefixes, options.stateDir, "$OPENCLAW_STATE_DIR");
  addPathPrefixVariants(prefixes, options.env.HOME, "~");
  addPathPrefixVariants(prefixes, options.env.USERPROFILE, "~");
  return [...prefixes.values()].toSorted((a, b) => b.prefix.length - a.prefix.length);
}
```

`/home/alice/.openclaw/logs/x.log` 变成 `$OPENCLAW_STATE_DIR/logs/x.log` 或 `~/...`——既抹掉了用户名,又**保留了路径的相对结构**,维护者照样能看出"这是状态目录下 logs 里的文件"。几个工程细节:

- **按长度从长到短排序**:`toSorted((a, b) => b.prefix.length - a.prefix.length)`,确保更具体的前缀(状态目录)优先于更宽泛的前缀(家目录)被匹配,不会把 `$OPENCLAW_STATE_DIR` 误并进 `~`。
- **跨平台**:`isWindowsAbsolutePath` 识别 `C:\` 和 UNC 路径,Windows 路径做大小写不敏感匹配,还额外登记一个把 `\` 换成 `/` 的变体——因为日志里同一个路径可能以两种斜杠形态出现。
- **已经是符号路径就放过**:`if (file.startsWith("$")) return file;`——已经脱敏成 `$OPENCLAW_STATE_DIR/...` 的路径不会被二次处理。
- **匹配不上就回落到文本脱敏**:认不出的绝对路径,交给 `redactSensitiveTextForSupport` 再过一遍,避免漏网。

## 15.6 边界硬约束:深度、长度、条数都封顶

脱敏解决"内容安全",但还有一类风险是"体积/形态安全"——一个恶意构造的或纯粹巨大的数据结构,可能让导出过程吃光内存或生成一个谁也打不开的巨型文件。`diagnostic-support-redaction.ts` 顶部一排常量就是干这个的:

```ts
const MAX_SUPPORT_STRING_LENGTH = 2000;
const MAX_SUPPORT_SNAPSHOT_DEPTH = 10;
const MAX_SUPPORT_ARRAY_ITEMS = 1000;
const MAX_SUPPORT_OBJECT_ENTRIES = 1000;
```

递归到第 10 层就返回 `"<truncated>"`;数组超过 1000 项、对象超过 1000 个键就截断,并且**把截断这件事本身记录进结果**:

```ts
function supportArrayResult(items: unknown[], count: number): unknown[] | Record<string, unknown> {
  if (count <= MAX_SUPPORT_ARRAY_ITEMS) {
    return items;
  }
  return { items, truncated: true, count, limit: MAX_SUPPORT_ARRAY_ITEMS };
}
```

注意 `truncated: true, count, limit` 这三个字段——它不是悄悄丢掉多余数据,而是**诚实地告诉维护者"这里原本有 count 个,我只保留了 limit 个"**。这一点很重要:被截断的诊断如果不声明截断,会误导排障(以为只有这么多);声明了截断,维护者就知道"如果不够,得换个方式拿全量"。这是"错误即文档"原则在数据截断上的延伸。

还有一个安全细节:遍历对象时会跳过**原型污染**相关的危险键:

```ts
if (isBlockedObjectKey(key) || entries.length >= MAX_SUPPORT_OBJECT_ENTRIES) {
  continue;
}
```

并且构造输出用的是 `Object.create(null)`(`createSupportRecord`)——一个没有原型链的纯净对象,从源头上杜绝 `__proto__`/`constructor` 这类键在重建对象时引发原型污染。处理不可信数据时,连"用什么对象来装结果"都被考虑进了威胁模型。

## 15.7 日志脱敏:白名单 + 黑名单的双保险

逐行日志是最危险的——它什么都可能含。`diagnostic-support-log-redaction.ts` 对每一条日志记录采用了**白名单为主、黑名单兜底**的双重策略。

先看黑名单 `OMITTED_LOG_FIELD_RE`,凡是字段名命中它的,**整个字段直接丢弃**:

```ts
const OMITTED_LOG_FIELD_RE =
  /(?:authorization|body|chat|content|cookie|credential|detail|error|header|instruction|message|password|payload|prompt|result|secret|session[-_]?id|...|text|token|tool|transcript|url)/iu;
```

再看白名单 `LOG_STRING_FIELD_RE` 和 `LOG_SCALAR_FIELD_RE`,**只有字段名在白名单里的才会被保留**(字符串字段限白名单字符串名,数字/布尔字段可来自任一白名单):

```ts
function isSafeLogField(key: string, value: unknown): boolean {
  if (typeof value === "string") {
    return LOG_STRING_FIELD_RE.test(key);
  }
  return LOG_STRING_FIELD_RE.test(key) || LOG_SCALAR_FIELD_RE.test(key);
}
```

白名单里是 `action`/`level`/`module`/`phase`/`requestId`/`runId`/`status`/`traceId` 这类**纯操作性元数据**,以及 `bytes`/`count`/`durationMs`/`pid`/`statusCode` 这类**安全的标量指标**。两份名单合起来构成一个非常严格的过滤器:**未知字段默认不导出。** 这是教科书式的 fail-closed——黑名单负责挡掉"明确危险的",白名单负责"只放行明确安全的",一个新出现的、谁也没预料到的字段,会因为不在白名单里而被默默丢弃,而不是因为不在黑名单里就被放出去。

即便一个字段叫 `msg`(在白名单里),它的**内容**还要再过一道 `UNSAFE_LOG_MESSAGE_RE`:

```ts
if (key === "msg" && (!message || UNSAFE_LOG_MESSAGE_RE.test(message))) {
  addOmittedLogMessageMetadata(sanitized, value);
  return;
}
```

`UNSAFE_LOG_MESSAGE_RE` 专门识别那些可能携带对话内容的消息模式(`ai response`/`assistant said`/`prompt`/`raw webhook body`/`tool output`/`transcript`/`user said`…)。一旦命中,消息正文不进 bundle,取而代之的是一条元数据:

```ts
function addOmittedLogMessageMetadata(sanitized: Record<string, unknown>, value: string): void {
  sanitized.omitted = "log-message";
  sanitized.omittedLogMessageBytes = numericLogMetadata(sanitized.omittedLogMessageBytes) + byteLength(value);
  sanitized.omittedLogMessageCount = numericLogMetadata(sanitized.omittedLogMessageCount) + 1;
}
```

注意它记录了 `omittedLogMessageBytes` 和 `omittedLogMessageCount`——**省略了多少条、多少字节,都被诚实地统计出来**。维护者看到 `omitted: "log-message", omittedLogMessageCount: 12` 就知道"这里有 12 条消息出于隐私被省略了",而不是误以为这段时间没有日志。又一次"被省略的事实本身也是诊断信息"。

如果一条日志解析后没有任何安全字段可留,它也不会消失,而是退化成一条占位记录:`{ omitted: "no-safe-fields", bytes: ... }`。整条记录被抹掉了,但"这里曾有一条 N 字节的日志"这个事实保留下来。

## 15.8 写盘那一刻:别让路径逃出盒子

数据都脱敏干净了,最后还有"写到磁盘"这一步。这里的风险是**路径逃逸**:如果某个 bundle 文件的相对路径被构造成 `../../etc/passwd`,写入时就可能跳出输出目录、覆盖系统文件。`diagnostic-support-bundle.ts` 用两道关卡防这个。

第一道,在构造每个 bundle 文件时就校验相对路径:

```ts
function assertSafeBundleRelativePath(pathName: string): string {
  const normalized = pathName.replaceAll("\\", "/");
  if (
    !normalized ||
    normalized.startsWith("/") ||
    normalized.split("/").some((part) => part === "" || part === "." || part === "..")
  ) {
    throw new Error(`Invalid bundle file path: ${pathName}`);
  }
  return normalized;
}
```

空路径、绝对路径、含 `.`/`..`/空段的路径,一律抛错。第二道,在真正解析出绝对路径后**再查一次**:

```ts
function resolveSupportBundleFilePath(outputDir: string, pathName: string): string {
  const safePath = assertSafeBundleRelativePath(pathName);
  const resolvedBase = path.resolve(outputDir);
  const resolvedFile = path.resolve(resolvedBase, safePath);
  // Re-check after path.resolve so crafted relative paths cannot escape the output directory.
  if (resolvedFile === resolvedBase || !isPathInside(resolvedBase, resolvedFile)) {
    throw new Error(`Bundle file path escaped output directory: ${pathName}`);
  }
  return resolvedFile;
}
```

那条注释说得很清楚:**在 `path.resolve` 之后重新检查,因为精心构造的相对路径可能在解析后才逃出目录。** 字符串层面的检查和路径解析后的检查不能互相替代——这是"纵深防御"的标准做法:不依赖单一检查点,而是在数据流的多个位置重复验证。

写文件时还有一组很克制的权限和标志:

```ts
await fsp.writeFile(filePath, file.content, {
  encoding: "utf8",
  flag: "wx",   // 文件已存在则失败,绝不覆盖
  mode: 0o600,  // 仅属主可读写
});
```

目录用 `0o700`(仅属主可访问),文件用 `0o600`(仅属主可读写),`flag: "wx"` 保证**绝不覆盖已有文件**(防止符号链接攻击或意外覆盖)。一份可能含敏感运维信息的诊断包,从它落地的那一刻起,就不该被同机器的其他用户读到。

## 15.9 把整个流程串起来

回到顶层 `buildDiagnosticSupportExport`,现在可以看清它编排的这条完整流水线:

1. 解析状态目录、配置路径,准备好脱敏上下文 `{ env, stateDir }`。
2. 读日志尾部(默认 5000 行 / 1MB 上限),逐行过 `sanitizeSupportLogRecord`。
3. 读配置,产出两份:`shape`(只含计数/键名/网关模式等结构概览)和 `sanitized`(脱敏后的配置详情)。
4. 并行采集网关 status / health 快照,各自过 `sanitizeSupportSnapshotValue`。
5. 读最新的稳定性 bundle(本身就是 payload-free 的)。
6. 从脱敏后的日志里**衍生**出 Bonjour/mDNS 摘要——注意它是从已脱敏的日志里统计的,不碰原始数据。
7. 拼出 `diagnostics.json`、`config/shape.json`、`config/sanitized.json`、`logs/openclaw-sanitized.jsonl`、各快照、`stability/latest.json`、`summary.md`,最后加上 `manifest.json`。
8. 全部打进一个 `0o600` 权限的 zip。

整条链路里,**没有任何一处把原始数据直接写进文件**:日志先脱敏再写、配置先脱敏再写、快照先脱敏再写、路径先符号化再写。`payloadFree: true` 这个类型字面量,是这整条流水线共同兑现的承诺。而那份 `summary.md` 里的 "Maintainer Quick Read",则把每个文件的用途一条条列给维护者看——**导出不只是把数据塞进盒子,还附上了一张让陌生人快速上手的说明书**。

## 本章小结

这一章我们看完了 OpenClaw 如何把一次崩溃的"现场"打包成一个能安全交给任何人的盒子:

1. **payload-free 是类型级承诺**:manifest 里 `payloadFree: true`/`rawLogsIncluded: false` 是字面量类型,编译器强制这份导出永不含原始载荷;隐私清单同时写进 manifest 和给人读的 summary.md。
2. **两种脱敏策略**:普通快照(`sanitizeSupportSnapshotValue`)抹秘密 + 载荷 + 身份三类字段(连私密数字也抹);配置(`sanitizeSupportConfigValue`)更严,额外抹访问控制字段,密钥引用保留 source/provider、抹 id。
3. **私密映射连键名都换掉**:`users`/`chats` 等映射在快照里退化成 `{count}`,在配置里键名换成 `<redacted-user-N>`,留结构去身份。
4. **文本级脱敏流水线**:在第 14 章引擎之上叠加 Basic/Cookie/AWS key/JWT、URL userinfo、Matrix 标识符、邮箱/handle、长数字 id,每处用语义化占位符。
5. **路径符号化**:家目录/状态目录前缀换成 `~`/`$OPENCLAW_STATE_DIR`,按前缀长度排序、跨平台、已符号化则放过。
6. **体积/形态硬约束**:深度 10、数组 1000、对象 1000、字符串 2000 封顶,且把截断诚实记入结果;用 `Object.create(null)` + `isBlockedObjectKey` 防原型污染。
7. **日志白名单 + 黑名单双保险**:黑名单丢危险字段、白名单只放行操作性元数据,未知字段默认不导出;`msg` 内容再过 `UNSAFE_LOG_MESSAGE_RE`,省略时记录条数与字节数。
8. **写盘防逃逸**:相对路径两道校验(字符串层 + `path.resolve` 后重查)、`flag:"wx"` 不覆盖、目录 `0o700`/文件 `0o600`。

贯穿全章的,依然是那条从第 1 章追到现在的主线:**在边界处对一切保持不信任,默认选更安全的一边,但绝不为安全牺牲掉排障所需的可读性。** 支持导出把这条原则推到了极致——因为这一次,接收方是一个素未谋面的陌生人,容错率为零。

## 动手实验

> 以下实验基于本章引用的源码文件,建议在你 clone 的 OpenClaw 仓库里对照阅读与验证。

**实验一:验证 payload-free 是编译期契约。**
读 `src/logging/diagnostic-support-export.ts` 里 `DiagnosticSupportExportManifest` 的类型定义,特别是 `privacy: { payloadFree: true; rawLogsIncluded: false; ... }`。回答:如果有人在 `buildDiagnosticSupportExport` 里把 `payloadFree` 赋成一个运行时计算出的 `boolean` 变量(而不是字面量 `true`),TypeScript 会发生什么?为什么把它写成字面量类型,比写一句"记得别加原始载荷"的注释更可靠?把这个设计和第 11 章配置 schema 校验、第 14 章脱敏默认 tools 模式放在一起,说说"用类型/默认值把安全不变量固化下来"这条主线。

**实验二:推演两种脱敏策略的差异。**
给定一个对象 `{ owner_id: "u_123456789", gateway: { port: 8080 }, users: { "a@x.com": {}, "b@y.com": {} }, note: "hello" }`。分别用 `sanitizeSupportSnapshotValue` 和 `sanitizeSupportConfigValue` 逐字段推演输出。重点解释:为什么 `owner_id` 在 config 策略下被抹、在普通快照策略下却不一定(对照 `CONFIG_PRIVATE_FIELD_RE` 与 `isPrivateSupportField`)?`users` 映射在两种策略下分别变成什么形状?

**实验三:跑通一条日志的双重过滤。**
读 `sanitizeSupportLogRecord` 和 `addSafeLogField`。给定一行日志 JSON:`{"level":"warn","msg":"assistant said: hello user","token":"sk-abc...","durationMs":42,"customField":"x"}`。逐字段推演最终进入 bundle 的记录长什么样:`level`/`durationMs` 为什么留?`token` 为什么被丢?`customField` 为什么被丢(它既不在黑名单也不在白名单)?`msg` 命中 `UNSAFE_LOG_MESSAGE_RE` 后,输出里会多出哪两个统计字段?用这个例子说明"黑名单兜底 + 白名单放行"为什么比只用其中一个更安全。

**实验四:构造一次路径逃逸并验证它被挡住。**
读 `assertSafeBundleRelativePath` 和 `resolveSupportBundleFilePath`。分别推演这几个 `pathName` 会在哪一道关卡被拦:(a) `"../secret.txt"`;(b) `"logs/../../etc/passwd"`;(c) `"/etc/passwd"`;(d) 一个看起来正常、但 `path.resolve` 后会跳出 `outputDir` 的精心构造路径。然后回答:既然 `assertSafeBundleRelativePath` 已经禁掉了 `..`,为什么 `resolveSupportBundleFilePath` 还要在 `path.resolve` 之后再用 `isPathInside` 查一遍?这两道检查是冗余还是纵深防御?

## 下一章预告

到这里,我们已经把 OpenClaw 从启动、agent loop、工具、沙箱、子 agent、网关、配置、节点配对、插件、可观测性,一路讲到了诊断导出——一个 agent 系统"安全地跑起来、安全地被观察、安全地交出现场"的完整闭环。但有一个贯穿始终、却一直没被单独拆开看的主角:**这套系统的测试与可信度是怎么建立的?** 我们在前面每一章都瞥见过 `*.test.ts` 文件,看到过"为了可测试而做的依赖注入"(`readLogTail`/`readStatusSnapshot` 这些可替换的 reader),看到过那些把边界情形写成断言的测试。第 16 章我们将专门走进 OpenClaw 的**测试策略与可信度工程**:它如何用测试把"fail-closed""脱敏不漏""路径不逃逸"这些安全承诺从"代码看起来对"变成"持续被验证对"?当一个 agent 要在无人值守下自动干活时,究竟是什么让我们敢信任它?
