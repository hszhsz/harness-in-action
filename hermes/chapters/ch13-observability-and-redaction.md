# 第 13 章　可观测性与脱敏:让诊断不泄密

前面十二章里的 Hermes,能读你的文件、连你的账号、在你机器上跑命令、记住关于你的事。这样一个深度介入你数字生活的 Agent,一旦出了问题需要看日志、导出诊断、上报 issue,就面临一个尖锐的矛盾:**诊断需要信息,但信息里混着你的密钥、token、密码和手机号**。把日志原样发出去,等于把家门钥匙一起寄走。

这一章读 Hermes 的可观测性与脱敏子系统——`hermes_logging.py`(日志分级与文件管理)和 `agent/redact.py`(基于正则的密钥脱敏,实测 496 行)。核心问题只有一个:怎么让日志足够详细到能排障,又保证敏感信息**永远写不进磁盘**?

## 13.1 日志分级:四个文件,各司其职

`hermes_logging.py` 的模块文档把日志架构讲得清清楚楚。所有日志落在 `~/.hermes/logs/`(profile-aware,经 `get_hermes_home()`),分四个文件:

- `agent.log` —— INFO+,所有 agent/tool/session 活动,主日志(catch-all,什么都往这进)。
- `errors.log` —— WARNING+,只有错误和警告,用于快速分诊(quick triage)。
- `gateway.log` —— INFO+,仅 gateway 事件(mode="gateway" 时创建)。
- `gui.log` —— INFO+,dashboard/websocket/TUI-gateway 事件(mode="gui" 时创建)。

这种分级不是随意的。`errors.log` 只收 WARNING 以上,是为了让运维"一眼看到出没出问题"而不必在海量 INFO 里翻找。而 `gateway.log` 和 `gui.log` 做了**组件隔离**:gateway.log 只接收 `gateway.*` logger 的记录(平台适配器、会话管理、slash 命令、投递),gui.log 只接收 `hermes_cli.web_server`、`hermes_cli.pty_bridge`、`tui_gateway.*`、`uvicorn.*` 的记录。把不同子系统的日志分流到不同文件,排查网关问题时不用被 dashboard 的噪声淹没。

还有一个便于关联(correlation)的设计:`set_session_context(session_id)`(约第 112 行)在对话开始时设置、`clear_session_context()` 结束时清除,之后**该线程上的每一行日志都带上 `[session_id]`**,日志格式 `_LOG_FORMAT`(约第 52 行)里的 `%(session_tag)s` 就是干这个的。多会话并发时,靠这个 tag 就能把一次对话的所有日志线串起来。

日志文件都用 `RotatingFileHandler` 做轮转,默认 `max_bytes` 是 `(max_size_mb or cfg_max_size or 5) * 1024 * 1024`(约第 255 行,默认 5MB)、`backups` 默认 3(可由 config.yaml 的 `logging.backup_count` 覆盖),防止日志无限增长撑爆磁盘。但最关键的一行在模块文档里:`All files use RotatingFileHandler with RedactingFormatter so secrets are never written to disk.`——**所有 handler 都挂 `RedactingFormatter`**,这是脱敏生效的总开关。

## 13.2 脱敏即 Formatter:在写盘的最后一道关口拦截

脱敏的接入点设计得极简却极有效。`RedactingFormatter`(`agent/redact.py` 约第 488 行)继承 `logging.Formatter`,只重写了 `format`:

```python
def format(self, record: logging.LogRecord) -> str:
    original = super().format(record)
    return redact_sensitive_text(original)
```

它先让父类正常格式化出整行日志,然后整行过一遍 `redact_sensitive_text` 再返回。把脱敏放在 Formatter 这一层,意味着**无论代码哪里打的日志、用什么 logger、记了什么内容,只要经过这个 formatter 写盘,就一定被脱敏过**。开发者不需要在每个 `logger.info(...)` 调用处记得手动脱敏——脱敏是基础设施级别的、默认发生的,而不是要靠人记得的。这正是"默认即安全"在日志层的落地:你想泄密都得绕过它,而不是想保密才特地启用它。

## 13.3 默认开启,且无法在会话中被关掉

脱敏的开关 `_REDACT_ENABLED`(约第 67 行)是 `os.getenv("HERMES_REDACT_SECRETS", "true").lower() in {"1", "true", "yes", "on"}`——**默认 "true"**,这是 issue #17691 明确要求的安全默认。用户确实能关(在 config.yaml 里设 `security.redact_secrets: false`,或 `.env` 里 `HERMES_REDACT_SECRETS=false`),但关闭时 gateway 和 CLI 启动会打一条降级警告,让运维看到这个降级。

最精妙的是注释里的一句话:这个开关**在 import 时快照**——`Snapshot at import time so runtime env mutations (e.g. LLM-generated export HERMES_REDACT_SECRETS=false) cannot disable redaction mid-session.` 想想这个威胁模型:Agent 自己会跑命令,如果一个被注入的 prompt 诱导 Agent 执行 `export HERMES_REDACT_SECRETS=false`,而脱敏开关是实时读 env 的,那么脱敏就被 Agent 自己关掉了。Hermes 把开关在模块加载时就固化成常量,**会话中途任何对环境变量的修改都无法关闭脱敏**。这是把"Agent 可能被诱导攻击自己"纳入考量的防御性设计——脱敏不仅要防外部泄露,还要防自己被骗着自废武功。

## 13.4 脱敏认得什么:厂商前缀、结构化字段与 URL

`agent/redact.py` 识别敏感信息的覆盖面相当惊人,体现了大量真实世界的积累。

**厂商密钥前缀**(`_PREFIX_PATTERNS`,约第 70 行)是一长串正则,几乎把主流服务的 key 格式都列了:`sk-`(OpenAI/Anthropic)、`ghp_`/`github_pat_`/`gho_`/`ghu_`/`ghs_`/`ghr_`(GitHub 各类 token)、`xox[baprs]-`(Slack)、`AIza`(Google)、`AKIA`(AWS Access Key ID)、`sk_live_`/`sk_test_`/`rk_live_`(Stripe)、`hf_`(HuggingFace)、`xai-`(xAI/Grok)等几十种。每条都配了注释说明属于哪家。这意味着哪怕一个密钥在日志里裸奔,只要它有已知前缀,就会被认出来打码。

**结构化字段**也被覆盖:`_ENV_ASSIGN_RE` 匹配 `KEY=value` 形式里 KEY 含 `API_KEY`/`TOKEN`/`SECRET`/`PASSWORD`/`CREDENTIAL`/`AUTH` 的赋值;`_JSON_FIELD_RE` 匹配 JSON 里 `"apiKey": "..."`、`"token": "..."` 这类字段;`_AUTH_HEADER_RE` 匹配 `Authorization: Bearer <token>`;`_PRIVATE_KEY_RE` 匹配整块 `-----BEGIN ... PRIVATE KEY----- ... -----END ...-----`;`_DB_CONNSTR_RE` 匹配数据库连接串 `postgres://user:PASSWORD@host` 里的密码;`_JWT_RE` 认 `eyJ` 开头的 JWT;`_TELEGRAM_RE` 认 Telegram bot token;`_SIGNAL_PHONE_RE` 认 E.164 手机号。

**URL 里的秘密**更是细致到三种形态都覆盖:`_URL_WITH_QUERY_RE` 扫 query string 里的敏感参数、`_URL_USERINFO_RE` 扫 `scheme://user:password@host` 的 userinfo、`_HTTP_REQUEST_TARGET_QUERY_RE` 扫 HTTP access log 里相对路径 `"POST /webhook?password=..."` 的 query(因为它不含 `://`,普通 URL 正则看不到)。敏感参数名列在 `_SENSITIVE_QUERY_PARAMS` 和 `_SENSITIVE_BODY_KEYS`(约第 19、40 行)里,且注释强调是**精确匹配而非子串**——`token_count` 和 `session_id` 不能误伤。这些大多注明 `Ported from nearai/ironclaw#2529`,是从别的项目踩坑里移植来的经验。

## 13.5 保结构、去内容:`mask_secret` 的可调试性权衡

脱敏不是简单地把秘密换成一坨 `***` 就完事——那样虽然安全,但排障时你连"哪个 key 错了"都看不出来。Hermes 的 `mask_secret`(约第 196 行)在安全和可调试性之间做了精细权衡。

它的策略是**保留首尾、隐藏中间**:默认 `head=4, tail=4`,值短于 `floor=12` 的**全部打码**返回 `***`,够长的则返回 `value[:head]...value[-tail:]`。文档里的例子很直观:`mask_secret("sk-proj-abcdef1234567890")` → `'sk-p...7890'`,`mask_secret("short")` → `'***'`。短 token 全遮是因为留首尾会暴露太大比例;长 token 留首尾是为了"还能认出这是哪个 key"。日志专用的 `_mask_token`(约第 243 行)更保守:`head=6, tail=4, floor=18`——18 字符以下全遮,够长的留 6 前缀 4 后缀。

这正是工程透镜里"留结构去内容"的精确体现:脱敏后的 `sk-p...7890` 保留了数据的**形状**(看得出是个 sk- 开头的 OpenAI key、看得出尾号),抹掉了能被盗用的**内容**(中间的实际密钥)。排障的人能据此确认"哦是这个 key 配错了",但拿不到可用的凭证。

## 13.6 性能与正确性:廉价预筛 + "无漏报"保证

几十条正则,如果每行日志都全跑一遍,开销不小。Hermes 用了**廉价预筛(cheap pre-check)**优化,而且做得很有讲究。

`_has_known_prefix_substring`(约第 462 行)在跑昂贵的 `_PREFIX_RE` 大正则前,先用简单的 `in` 子串检查过滤——若整行里连一个已知前缀的字面子串都没有,大正则不可能匹配,直接跳过。关键在于这些预筛子串 `_PREFIX_SUBSTRINGS` 不是手写的,而是用 `_extract_literal_prefix`(约第 441 行)从 `_PREFIX_PATTERNS` **自动派生**——它取每条正则在第一个元字符(`[`、`(`、`\`、`.` 等)之前的字面前缀。注释把不变量讲得极清楚:`the bound is "no false negatives" and that holds because every pattern ... has at least one of these as a literal substring`——**允许误报(白跑一次正则没匹配上,无害),但保证无漏报**(不会因为预筛跳过而漏掉真该脱敏的内容)。而且自动派生意味着将来有人往 `_PREFIX_PATTERNS` 加新前缀,预筛会自动跟上,不会"加了新 key 格式却忘了更新预筛"而静默失效。`_has_http_method_substring`(约第 484 行)同理,扫 access-log 请求行前先廉价检查有没有 HTTP 方法字样。

把性能优化建立在"无漏报"的形式化保证之上、并让它随模式列表自动演进——这是把"安全不变量"固化进代码结构、而非靠人记得维护的典范。

把这一章合起来看,Hermes 的可观测性子系统贯彻了几个层层递进的原则:日志分级让诊断高效、组件隔离让排查聚焦;脱敏挂在 Formatter 这一最末关口,保证"经手必脱敏";开关默认开启且 import 时快照,连 Agent 自己也关不掉;识别覆盖厂商前缀/结构化字段/三种 URL 形态;`mask_secret` 留首尾去中间,在安全与可调试间取平衡;预筛优化建立在"无漏报"保证和自动派生之上。一句话:**让你能放心地把日志发给别人求助,因为里面早就没有你的秘密了。**

## 本章小结

- 日志分四个文件(`~/.hermes/logs/`):`agent.log`(INFO+ catch-all)、`errors.log`(WARNING+ 快速分诊)、`gateway.log`(仅 `gateway.*`)、`gui.log`(dashboard/TUI),组件隔离让排查聚焦。
- `set_session_context` 给线程上每行日志打 `[session_id]` tag(`%(session_tag)s`),便于多会话并发时关联;`RotatingFileHandler` 默认 5MB×3 轮转防撑爆磁盘。
- 脱敏接入点是 `RedactingFormatter.format`(约第 488 行):整行格式化后过 `redact_sensitive_text`——**经手必脱敏**,无需开发者在每处手动调用。
- 开关 `_REDACT_ENABLED` 默认 `true`(issue #17691),且**在 import 时快照**,会话中途改环境变量(含 Agent 被诱导执行 `export`)无法关闭脱敏——防自废武功。
- 识别覆盖三类:厂商密钥前缀(`sk-`/`ghp_`/`AKIA`/`xai-` 等几十种)、结构化字段(env 赋值/JSON 字段/Authorization header/私钥块/DB 连接串/JWT/Telegram token/E.164 手机号)、URL 三形态(query/userinfo/access-log 请求行)。
- 敏感参数名精确匹配而非子串(`token_count`/`session_id` 不误伤),多处移植自 `nearai/ironclaw#2529`。
- `mask_secret`(约第 196 行)留首尾去中间:短于 `floor` 全遮 `***`,够长返回 `head...tail`(如 `sk-p...7890`)——保留数据形状、抹掉可用内容,兼顾安全与可调试。
- 性能用廉价子串预筛(`_has_known_prefix_substring`),预筛子串由 `_extract_literal_prefix` 从模式列表**自动派生**,保证"无漏报、允许误报",且新增前缀时预筛自动跟上不会静默失效。

## 动手实验

1. **实验一:追踪一行日志的脱敏路径** —— `Read` `hermes_logging.py` 约第 255–305 行的 handler 装配与 `agent/redact.py` 的 `RedactingFormatter`(约第 488 行)。论证:一条 `logger.info("key=sk-proj-abcdef1234567890")` 从调用到落盘,在哪一步被脱敏?为什么把脱敏放在 Formatter 而不是每个调用点更可靠?

2. **实验二:验证"关不掉"的脱敏** —— `Read` `agent/redact.py` 约第 60–67 行 `_REDACT_ENABLED` 的定义与注释。设想会话中途 Agent 执行了 `export HERMES_REDACT_SECRETS=false`。推演:本次会话剩余的日志还会脱敏吗?为什么?这防住了什么攻击场景?

3. **实验三:测脱敏覆盖面** —— `Read` `_PREFIX_PATTERNS`(约第 70 行)与各结构化正则。判断下面哪些会被脱敏:`ghp_xxxxxxxxxxxx`、`Authorization: Bearer abc123...`、`postgres://u:p@host/db`、`+8613800138000`、`session_id=12345`。最后一个为什么不该被脱敏?

4. **实验四:理解"无漏报"预筛** —— `Read` `_extract_literal_prefix`(约第 441 行)和 `_has_known_prefix_substring`(约第 462 行)。解释:为什么预筛"允许误报但禁止漏报"?如果有人给 `_PREFIX_PATTERNS` 加了一条新前缀但忘了改预筛,会发生什么?自动派生如何避免这个坑?

> **下一章预告**:前面十三章揭示的所有安全属性——默认脱敏、约束收紧、配对的密码学、路径逃逸防御——光靠读代码相信它们"应该对"是不够的,必须有测试把它们钉死成回归保护网。第 14 章将解构 Hermes 的测试与可信度基建:依赖注入如何让时间和随机可被测试、临时目录与冻结时钟如何隔离测试、以及断言如何把"安全契约"变成跑得起来的代码。
