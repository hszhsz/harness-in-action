# 第 15 章　可观测性、遥测与诊断

第 14 章把扩展体系拆透了——MCP、插件、技能让 Codex 的能力边界不断外扩。但能力越多、链路越长,"线上到底发生了什么"就越难看清:一次 turn 慢在哪?compaction 到底省了多少 token?SQLite 索引多久退一次文件系统?用户报障时,我们能不能拿到足够的现场、又不至于把他的代码或密钥一并捞走?

这一章拆 Codex 的**可观测性三件套**:**指标**(OTEL metrics,量化关键路径)、**遥测/分析**(analytics,理解用户行为且**默认脱敏**)、**诊断**(feedback diagnostics,报障时收集环境但守住隐私)。贯穿始终的一条线是:**观测是为了改进,但观测本身绝不能成为新的泄漏面**。代码主要在 `otel/`、`analytics/`、`feedback/` 三个 crate,以及 `utils/string/` 里的脱敏工具。

---

## 15.1 指标骨架:OTEL + 默认 Statsig 导出器

Codex 的指标走 OpenTelemetry。`otel/src/lib.rs` 是这层的门面,对外导出 `SessionTelemetry`(会话级遥测句柄)、`OtelProvider`、`Timer` 以及一众指标名常量。导出器由 `OtelExporter` 枚举决定,默认是 `Statsig`——`config.rs` 的 `resolve_exporter` 把它解析成一个发往 `STATSIG_OTLP_HTTP_ENDPOINT = "https://ab.chatgpt.com/otlp/v1/metrics"` 的 OTLP/HTTP(JSON)导出器。

这里有一处对开发者友好的克制:`resolve_exporter` 在 `cfg!(debug_assertions)`(debug 构建)下**强制把 Statsig 关成 `OtelExporter::None`**,注释说明意图——"Keep the built-in Statsig default off in debug builds so incremental local development and test runs do not emit best-effort OTEL traffic"。**本地开发和跑测试时,默认不往生产遥测端点发任何流量**,除非显式配置一个导出器。这是"观测不打扰开发"的体现。

`OtelSettings` 还把 trace、metrics 拆成独立的 exporter 字段(`trace_exporter`/`metrics_exporter`),允许"只发指标不发 trace"这类细粒度配置;`StatsigMetricsSettings` 则是一个只携带 `environment` 的精简结构,注释点明它的用途是"让另一个进程能重建内置指标导出器配置,而**不必在进程间传递通用导出器凭据**"——连内部组件之间传配置,都遵循最小权限。

---

## 15.2 指标命名:`codex.<域>.<动作>[.duration_ms]` 的一致表

`otel/src/metrics/names.rs` 把所有指标名集中成常量,命名高度规整——一律 `codex.` 前缀,按域分组,计时类统一以 `.duration_ms` 结尾。随手摘几组就能看出 Codex 在意什么:

- **工具与执行**:`codex.tool.call`、`codex.tool.call.duration_ms`、`codex.process.start`。
- **API 链路**:`codex.api_request`、`codex.api_request.duration_ms`、`codex.sse_event`,乃至把推理时延拆得极细的 `codex.responses_api_engine_iapi_ttft.duration_ms`(首 token 时间)、`..._tbt.duration_ms`(token 间隔)——**模型响应的快慢被拆到 TTFT/TBT 粒度量化**。
- **turn 级**:`codex.turn.e2e_duration_ms`(端到端)、`codex.turn.ttft.duration_ms`、`codex.turn.token_usage`、`codex.turn.memory`。
- **第 9 章 guardian**:`codex.guardian.review`、`codex.guardian.review.duration_ms`、`codex.guardian.review.token_usage`。
- **第 14 章扩展**:`codex.plugins.install_suggestion`、`codex.plugins.startup_sync`、`codex.hooks.run.duration_ms`。
- **第 14 章技能**(在 `render.rs` 里被打点):`codex.thread.skills.enabled_total`、`codex.thread.skills.kept_total`、`codex.thread.skills.truncated`、`codex.thread.skills.description_truncated_chars`——这正是第 14 章"技能预算截断"被量化的地方,团队据此能看到线上有多少会话的技能描述被压缩了。

把指标名收成一张常量表,而不是散落在调用点用字面量字符串,本身就是一种"可观测性卫生":改名、查引用、防拼写错,全在一处。`SessionTelemetry` 暴露的打点接口也很统一——`counter`、`histogram`、`record_duration`、`start_timer`,各业务模块用同一套 API 投递,标签语义一致。

---

## 15.3 标签的两道防线:字符白名单校验 + 基数收敛

指标的"维度"靠标签(tag)携带,但标签是个危险品:**值若来自用户输入,就可能注入非法字符、甚至把高基数的敏感信息(路径、ID)灌进监控系统**。Codex 给标签设了两道防线。

**第一道:字符白名单校验**(`otel/src/metrics/validation.rs`)。`validate_tag_value` 要求标签值非空、且每个字符都满足 `is_tag_char`——只允许 ASCII 字母数字加 `.`、`_`、`-`、`/`。指标名更严(`is_metric_char` 连 `/` 都不许)。不合规直接返回 `MetricsError`,打点失败而非把脏数据塞进去。

**第二道:脱敏与基数收敛**(`utils/string/src/lib.rs` 的 `sanitize_metric_tag_value`)。它比校验更进一步——**主动清洗**:把非白名单字符一律替换成 `_`,首尾的 `_` 去掉,长度截到 `MAX_LEN = 256`;若清洗后空了或没有任何字母数字,直接返回 `"unspecified"`。**任何外部字符串进监控前,都先被这把"安全剪刀"修一遍**。

但只清洗字符还不够防"基数爆炸"。`otel/src/metrics/tags.rs` 的 `bounded_originator_tag_value` 给出了第二招——**枚举白名单兜底**:originator 标签先 `sanitize`,再去 `KNOWN_ORIGINATOR_TAG_VALUES`(`codex_cli_rs`、`codex-tui`、`codex_vscode`、`codex_exec`……一张有限清单)里找,**找不到就归为 `"other"`**。为什么?因为如果 originator 能取任意值,监控系统里就会冒出成千上万个不同的标签值(高基数),既烧存储又毫无分析价值。**把开放维度强行收敛到一个已知的小集合**,是时序监控里非常重要的一条工程纪律。

`SessionMetricTagValues` 把会话级的标准标签(`auth_mode`、`session_source`、`originator`、`service_name`、`model`、`app.version`)按固定顺序组装,每个都过 `validate_tag_key`/`validate_tag_value`。**标签不是随手加的 key-value,而是一套受校验、受基数约束的结构。**

---

## 15.4 分析事件的隐私底线:把代码哈希掉,再把哈希也别上传

指标量化"快慢多少",**分析**(analytics)则理解"用户怎么用"。其中最敏感的一类是"用户接受了 AI 写的多少行代码"——这天然要碰用户的真实代码。`analytics/src/accepted_lines.rs` 处理这件事的方式,堪称隐私设计的范本。

`accepted_line_fingerprints_from_unified_diff` 解析一份 unified diff,统计接受的增/删行数。对每一行新增代码,它**不存原文**,而是算一个指纹:

```rust
pub fn fingerprint_hash(domain: &str, value: &str) -> String {
    let mut hasher = sha1::Sha1::new();
    hasher.update(b"file-line-v1\0");
    hasher.update(domain.as_bytes());
    hasher.update(b"\0");
    hasher.update(value.as_bytes());
    format!("{:x}", hasher.finalize())
}
```

文件路径、代码行都先被 SHA-1 成 `path_hash`/`line_hash`——**原始代码永远不离开本地**。`domain` 参数(`"path"`/`"line"`/`"repo"`)做域分隔,避免不同种类的值碰撞;`b"file-line-v1\0"` 前缀给指纹算法上了版本号,日后能演进。仓库标识同理:`accepted_line_repo_hash_for_cwd` 取 git 远端 URL,`canonicalize` 后 `fingerprint_hash("repo", ...)`——**连"这是哪个仓库"都只上传哈希,不上传 URL 本身**。

更见克制的是最后一步。`accepted_line_fingerprint_event_requests` 在组装真正上传的事件时,把已经算好的 `line_fingerprints` **清空成 `Vec::new()`**,注释解释得明明白白:"Keep computing local fingerprints for parsing tests and future attribution, but **do not upload path/line hashes in the analytics event payload**"。也就是说,即便已经是单向哈希,**上传的 payload 里连哈希都不放**,只留聚合后的增/删行计数和仓库哈希。这是"数据最小化"原则的极致——**能不传的,哪怕脱敏过,也不传**。

---

## 15.5 诊断收集:报障时收集环境,却为隐私留了余地

用户报障时,光有日志不够,常常要知道"他的网络环境是不是有代理在捣乱"。`feedback/src/feedback_diagnostics.rs` 的 `FeedbackDiagnostics` 就干这件事——它扫描一组代理相关环境变量:

```rust
const PROXY_ENV_VARS: &[&str] = &[
    "HTTP_PROXY", "http_proxy", "HTTPS_PROXY", "https_proxy", "ALL_PROXY", "all_proxy",
];
```

`collect_from_env` 把设置了的代理变量收进一条 "Proxy environment variables are set and may affect connectivity." 的诊断,最终 `attachment_text` 渲染成一份名为 `codex-connectivity-diagnostics.txt`(`FEEDBACK_DIAGNOSTICS_ATTACHMENT_FILENAME`)的附件随反馈一起上报。**有针对性地收集"最可能导致连接问题"的那几个变量**,而不是把整个环境一股脑打包——这本身就是一种收敛。

这里值得诚实指出一个**设计张力**:测试 `collect_from_pairs_reports_raw_values_and_attachment` 显示,代理 URL 是**原值**上报的(`HTTPS_PROXY = https://user:password@...` 连内嵌凭据都在内)。这说明诊断附件的定位是"用户主动发起、用于排障的本地文本",信任边界落在"用户自己决定要不要发这份附件"上——附件名固定、内容可读、范围限定在代理变量,让用户在点"发送"前能审阅自己到底交出了什么。**把脱敏的最后一道闸交给知情的用户**,是诊断类功能区别于后台静默遥测的关键:后者必须自动脱敏(见 15.4),前者则靠"可审阅 + 范围受限 + 用户知情同意"。

---

## 15.6 降级也是一等观测对象:每次回退都打带原因的点

回顾第 13 章那个 SQLite 状态库——它每一条降级路径都调用 `record_fallback(操作, 原因)`,原因字符串如 `db_unavailable`、`db_error`、`metadata_filter`、`missing_row`、`backfill_incomplete`。把这件事放到本章的视角下看,它的意义更清楚了:**降级不是"出错了悄悄兜底"就完事,它是一个需要被量化的一等观测对象**。

为什么重要?因为 SQLite 那层(第 13 章)是"派生缓存、可重建、绝非单点故障"——但一个缓存到底有多可靠,不能靠拍脑袋,得靠数据。每次回退都打一个带原因的遥测点,团队就能回答:线上 SQLite 多久退一次?退的主因是库不可用、还是回填没跟上、还是行陈旧?**只有把"降级率"和"降级原因分布"量化出来,才能判断这层缓存是真在加速,还是其实形同虚设、该重新设计**。

这与本章前几节是同一种思维的延伸:compaction 进遥测(第 12 章 `CompactionAnalyticsAttempt`)、技能截断进遥测(15.2 的 `codex.thread.skills.*`)、降级进遥测——**所有"有损的、兜底的、可能悄悄劣化体验的"路径,都被刻意地、带原因地打了点**。可观测性的成熟度,不在于你给"成功路径"埋了多少指标,而在于你有没有给"失败和降级路径"埋够指标。

---

## 15.7 把可观测性串成一条线:从一次 turn 看全景

把前几节拼起来,一次普通 turn 的可观测性足迹大致是这样:turn 开始,`SessionTelemetry` 用 `start_timer` 起 `codex.turn.e2e_duration_ms` 计时;发起 API 请求,记 `codex.api_request` 计数与时延,SSE 流式响应按 TTFT/TBT 拆点;模型要调工具,记 `codex.tool.call` 与时长,若触发 guardian 审批(第 9 章)再记 `codex.guardian.review`;若上下文逼近上限触发 compaction(第 12 章),`CompactionAnalyticsAttempt` 记下压缩前后 token;若这一轮装配了技能(第 14 章),`codex.thread.skills.*` 记下启用/保留/截断数;turn 结束,`codex.turn.token_usage` 落账。所有这些点的标签都过了 15.3 的校验与基数收敛;任何碰到用户代码的分析(15.4)都先哈希再最小化;一旦哪层走了降级(15.6),还会多一个带原因的回退点。

这条线的设计哲学,可以收成一句话:**让系统在不暴露用户隐私的前提下,对自己的每一个关键决策与每一次劣化保持诚实。**

---

## 本章小结

- **指标骨架**(`otel/`):OTEL metrics,默认导出器 `Statsig`(`resolve_exporter` → `STATSIG_OTLP_HTTP_ENDPOINT`),**debug 构建强制关成 `None`**,本地开发/测试不发遥测;trace/metrics 可独立配置;`StatsigMetricsSettings` 仅传 `environment`,不跨进程传通用凭据。
- **指标命名**(`metrics/names.rs`):统一 `codex.<域>.<动作>[.duration_ms]` 常量表;覆盖 tool/api/turn/guardian/plugins/hooks/skills;API 时延细到 TTFT/TBT;技能截断由 `codex.thread.skills.*` 量化(呼应第 14 章预算)。
- **标签两道防线**:`validation.rs` 字符白名单校验(`is_tag_char` 只许字母数字 + `.`/`_`/`-`/`/`);`sanitize_metric_tag_value` 主动清洗(非法字符→`_`、截 `MAX_LEN = 256`、空则 `"unspecified"`);`bounded_originator_tag_value` 用 `KNOWN_ORIGINATOR_TAG_VALUES` 白名单把开放维度收敛到有限集合(否则 `"other"`),防基数爆炸。
- **分析的隐私底线**(`analytics/accepted_lines.rs`):接受行统计**只算 SHA-1 指纹**(`fingerprint_hash`,带 `domain` 域分隔 + `file-line-v1` 版本前缀),原始代码/路径/仓库 URL 不出本地;**且上传 payload 把 `line_fingerprints` 清空**——脱敏过的哈希也不传,只留聚合计数 + 仓库哈希,数据最小化到极致。
- **诊断收集**(`feedback/feedback_diagnostics.rs`):报障时只扫 `PROXY_ENV_VARS` 六个代理变量,渲染成固定名 `codex-connectivity-diagnostics.txt` 附件;代理 URL 原值上报,靠"范围受限 + 内容可读 + 用户知情发送"守边界——后台遥测自动脱敏,用户发起的诊断靠知情同意。
- **降级是一等观测对象**:第 13 章 `record_fallback(操作, 原因)` 把每次回退量化(`db_unavailable`/`db_error`/`missing_row`/`backfill_incomplete`...);可观测性的成熟度在于给失败与降级路径埋够指标,而非只埋成功路径。

## 动手实验

1. **实验一:验证脱敏函数** — 阅读 `utils/string/src/lib.rs` 的 `sanitize_metric_tag_value`。对输入 `"feature/AB 测试@v2"`、`"___"`、`"a".repeat(300)` 分别给出输出,并说明它在什么情况下返回 `"unspecified"`。再读 `metrics/validation.rs`,解释 `sanitize`(清洗)与 `validate`(校验)的分工——为什么两者都需要?
2. **实验二:画出指标地图** — 阅读 `otel/src/metrics/names.rs`,把指标按"工具/API/turn/guardian/plugins/skills"分组各列两个。再对照第 14 章 `render.rs`,指出 `codex.thread.skills.truncated` 在什么时机被打点、它量化的是什么现象。
3. **实验三:跟踪一行代码的隐私路径** — 阅读 `analytics/src/accepted_lines.rs`。说明 `fingerprint_hash` 里 `domain` 参数和 `b"file-line-v1\0"` 前缀各自的作用。再解释 `accepted_line_fingerprint_event_requests` 为什么把 `line_fingerprints` 设成 `Vec::new()`——既然已经哈希过,为何连哈希都不传?
4. **实验四:辨析两种脱敏边界** — 对比 15.4 的分析事件(自动哈希 + 不上传)与 15.5 的诊断附件(代理 URL 原值上报)。结合 `FeedbackDiagnostics::collect_from_env` 与 `attachment_text`,解释为什么"后台静默遥测"必须自动脱敏,而"用户发起的诊断"可以靠知情同意——两者的信任边界差在哪?

> **下一章预告**:我们已经看遍了 Codex 的"内核"——循环、工具、沙箱、上下文、持久化、扩展、观测。但用户真正触摸到的,是它的**前端形态**。下一章进入 **TUI 与 exec**:同一套 core 如何同时驱动一个交互式终端界面(ratatui)和一个非交互、可脚本化的 `exec` 子命令,事件如何从 core 流向屏幕,以及"人在回路"与"全自动批处理"两种模式如何共享同一个引擎。
