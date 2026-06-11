# 第 14 章　测试与可信度:把安全契约钉成回归网

前面十三章揭示了 Hermes 一长串安全属性:配对码用恒定时间比较、脱敏默认开启且会话中关不掉、会话 ID 哈希化、命令执行有白名单、约束只能收紧。但读代码"看起来对"和"它真的对、而且永远对"之间,隔着一整套测试基建。如果没有测试把这些属性钉死成回归保护网,那么任何一次重构、任何一个新贡献者的 PR,都可能在无人察觉中悄悄掀翻某条安全防线。

这一章读 Hermes 的测试与可信度基建——主要是 `tests/conftest.py`(共享 fixture 与"封闭测试"不变量)和 `scripts/run_tests_parallel.py`(测试运行器)。Hermes 的测试规模本身就说明问题:`tests/` 下实测有 **1489 个 Python 测试文件**,注释里提到完整套件约 17000 个测试。但数量不是重点,重点是这套基建怎么保证每个测试都跑在一个**干净、确定、与生产一致**的环境里。

## 14.1 封闭测试:不变量写进 conftest 文档

打开 `tests/conftest.py`,模块文档开头就列了一组"封闭测试不变量(Hermetic-test invariants)",这是整个测试哲学的纲领:

1. **无凭证环境变量**——所有形如 `_API_KEY`、`_TOKEN`、`_SECRET`、`_PASSWORD`、`_CREDENTIALS` 的 env var 在每个测试前都被清空,本地开发者的真实密钥不可能泄进测试。
2. **隔离的 HERMES_HOME**——`HERMES_HOME` 指向每个测试独有的临时目录,读 `~/.hermes/*` 的代码碰不到真实的那个。
3. **确定性运行时**——`TZ=UTC`、`LANG=C.UTF-8`、`PYTHONHASHSEED=0`。
4. **不继承 HERMES_SESSION_***——Agent 当前的网关会话不能漏进测试。

这四条的共同目标写在文档里:`These invariants make the local test run match CI closely.`——让本地跑测试和 CI 跑测试结果一致。多少项目栽在"我本地是过的啊"这种话上,根源就是本地环境带着开发者的密钥、时区、locale,而 CI 是干净的。Hermes 把"封闭性"做成强制的、自动的不变量,从根上消灭这类幽灵 bug。

## 14.2 凭证防漏:一张细到发指的清单

不变量 1(无凭证 env var)的实现 `_looks_like_credential`(约第 162 行)值得细看。它两路判断:一是后缀匹配 `_CREDENTIAL_SUFFIXES`(`_API_KEY`、`_TOKEN`、`_SECRET`、`_ACCESS_KEY`、`_PRIVATE_KEY`、`_OAUTH_TOKEN`、`_WEBHOOK_SECRET`、`_AES_KEY` 等十几种后缀);二是精确名单 `_CREDENTIAL_NAMES`,这是一个长得惊人的 frozenset——`OPENAI_API_KEY`、`AWS_SECRET_ACCESS_KEY`、`TELEGRAM_BOT_TOKEN`、`FEISHU_APP_SECRET`、`WECOM_CALLBACK_ENCODING_AES_KEY`、`SUDO_PASSWORD`……几乎把全书出现过的每个平台、每个 provider 的凭证名都列了进去。

为什么要这么细?注释点明了攻击场景:有些测试断言"当某 key 存在时自动检测某 provider"。如果开发者本地恰好 `export` 了 `OPENAI_API_KEY`,这类测试就会因为捡到真实 key 而表现得和 CI 不同。把所有凭证形状的变量无条件清空,这类"自动检测"测试才有确定的前提。`_hermetic_environment`(autouse fixture,约第 327 行)遍历 `os.environ` 把每个匹配的 key `monkeypatch.delenv` 掉,并清空 `_HERMES_BEHAVIORAL_VARS` 里那一长串会改变行为的 `HERMES_*` 变量(`HERMES_YOLO_MODE`、`HERMES_REDACT_SECRETS`、各平台的 `*_ALLOWED_USERS`/`*_ALLOW_ALL_USERS` 等)——后者尤其关键,平台允许名单若从开发者 shell 漏进来,会让网关授权测试随机 flake。

## 14.3 隔离即临时目录:per-test HERMES_HOME

不变量 2 的实现也在 `_hermetic_environment` 里(约第 354 行起):它用 pytest 的 `tmp_path` 造一个 `fake_hermes_home = tmp_path / "hermes_test"`,提前建好 `sessions/`、`cron/`、`memories/`、`skills/` 子目录,然后 `monkeypatch.setenv("HERMES_HOME", str(fake_hermes_home))`。于是任何通过 `get_hermes_home()` 读 `~/.hermes/*` 的代码——会话存储、cron 任务、记忆文件、技能——全都落到这个一次性临时目录里,测试结束随 `tmp_path` 自动清理,绝不会污染真实的 `~/.hermes`。

这里有个诚实到值得学习的注释:他们**故意不重定向 `HOME`**,只重定向 `HERMES_HOME`。原因是曾经同时改 `HOME` 导致 CI 挂了——某些测试的子进程继承 `HOME` 并期望它稳定。所以注释立了条规矩:任何用 `Path.home() / ".hermes"` 而非规范的 `get_hermes_home()` 读路径的代码,**是 callsite 的 bug,要在那里修**。这是一种很成熟的态度——不为了测试方便去 hack 全局,而是把"必须走规范入口"反过来变成对生产代码的约束。

不变量 3(确定性)同样在这个 fixture 里钉死:`TZ=UTC`、`LANG=C.UTF-8`、`LC_ALL=C.UTF-8`、`PYTHONHASHSEED=0`。还顺手禁了 AWS IMDS 查询(`AWS_EC2_METADATA_DISABLED=true`),注释解释不禁的话每个碰到 provider 自动检测的测试都要白等约 2 秒去 timeout 那个 `169.254.169.254` 元数据服务——测试又不在 EC2 上跑。这些细节合起来,就是"确定性运行时"四个字的全部分量。

## 14.4 进程级隔离:每文件一个新解释器

测试之间最阴险的耦合是**跨文件状态泄漏**:模块级的 dict、ContextVar、缓存,被前一个测试文件改了,后一个文件读到脏值。Hermes 解决这个的方式很果断——`scripts/run_tests_parallel.py`:**每个测试文件 spawn 一个独立的 `python -m pytest <file>` 子进程**。

`conftest.py` 的注释把这个设计讲得很透:每个文件拿到一个全新的 Python 解释器,跨文件状态泄漏在物理上不可能。文件内的顺序耦合(同文件里测试 A 改了状态、测试 B 读到)则是"测试作者的责任"——因为那种 bug 直接 `pytest tests/foo.py` 也会暴露。运行器的注释还算了笔账解释为什么是"每文件"而非"每测试":每测试 spawn 的开销 `~250ms × 17k = 70 分钟`太贵,每文件 `~250ms × ~850 文件 = ~3.5 分钟`才划算,而隔离边界(跨文件模块级状态)恰好就靠"每文件一个解释器"守住。他们还特意**抛弃了 xdist**,因为 xdist 的持久 worker 会跨文件累积状态——那正是要消灭的泄漏源。用一个信号量门控的 `subprocess.Popen` 池约 60 行就够,反而更干净。这是"把隔离做成基建、而非靠测试作者自觉"的典范。

## 14.5 自动防护:测试不准误伤真实系统

最能体现"防御性测试基建"的是 `_live_system_guard`(autouse fixture,约第 538 行)。Agent 的代码会真去 `os.kill`、跑 `systemctl restart hermes-gateway`、扫描 gateway 进程——这些在生产里是正常操作,但在测试里万一执行了,就可能**真的把开发者机器上跑着的 Hermes 网关杀掉**。这个 fixture 把所有能向外部进程发信号或终止它的原语全 monkeypatch 拦截:`os.kill`/`os.killpg`、`subprocess.run`/`Popen`/`call`/`check_call`/`check_output`/`getoutput`、`os.system`/`os.popen`、`pty.spawn`、`asyncio.create_subprocess_exec`/`create_subprocess_shell`。

拦截还做得很细:它检查**整条命令字符串**而非只看 `tokens[0]`,所以 `bash -c "systemctl restart ..."`、`sudo systemctl ...`、`env systemctl ...`、`setsid systemctl ...` 都逃不掉,`pkill`/`killall`/`taskkill` 针对 hermes/python 的调用也被挡。真正需要发信号的测试(比如给自己子进程发 SIGINT 的 PTY 测试)可以用 `@pytest.mark.live_system_guard_bypass` 显式 opt-out。这是一道"默认即安全"的护栏:测试默认连真实系统的能力都没有,要用得主动声明。

## 14.6 断言即安全网:把属性写成可执行的契约

有了干净环境,安全属性才能被断言钉死。看几个真实例子。

`tests/agent/test_redact.py` 把第 13 章每一类密钥的脱敏都变成测试:`test_openai_sk_key` 断言 `redact_sensitive_text("Using key sk-proj-abc123...")` 的结果里 `"sk-pro" in result`(前缀保留)、`"abc123def456" not in result`(中间内容抹掉)、`"..." in result`(有省略标记)。`test_github_pat_classic`、`test_slack_token`、`test_google_api_key` 各自针对一种厂商前缀做同样的"原文不得出现在结果里"断言。这些测试把"脱敏必须遮住密钥"这条抽象契约,变成了几十条只要正则退化就立刻变红的硬约束。

`tests/gateway/test_pii_redaction.py` 则给第 9 章的会话哈希化上锁:`test_hash_id_deterministic` 断言 `_hash_id("12345") == _hash_id("12345")`(同输入同输出);`test_hash_id_12_hex_chars` 断言哈希正好 12 个十六进制字符;`test_hash_sender_id_prefix` 断言 `_hash_sender_id("12345").startswith("user_")` 且总长 17(`"user_"` + 12);`test_hash_chat_id_preserves_prefix` 断言哈希后 `result.startswith("telegram:")` 但 `"12345" not in result`——平台前缀保留、原始 ID 不泄露。这正是把"保留结构、移除内容"的脱敏哲学,精确到字符长度地固化成了测试。

把这些连起来看,Hermes 的可信度不是一句口号,而是一条流水线:封闭环境(凭证清零 + HERMES_HOME 隔离 + 确定性运行时)保证测试前提干净、本地与 CI 一致;每文件子进程隔离保证测试互不污染;live system guard 保证测试不误伤真实系统;断言把每条安全属性钉成回归红线。读代码让你相信 Hermes"应该是安全的",而这套测试基建让你相信它"会一直保持安全"——因为任何破坏安全属性的改动,都会先在这 1489 个文件里撞红。

## 本章小结

- `tests/` 实测 **1489 个测试文件**(套件约 17k 测试);`conftest.py` 文档把"封闭测试不变量"列为纲领:无凭证 env、隔离 HERMES_HOME、确定性运行时、不继承 session。
- 核心目标是"本地跑与 CI 跑一致",从根上消灭"我本地是过的"这类因环境差异(密钥/时区/locale)产生的幽灵 bug。
- `_looks_like_credential`(约第 162 行)两路判断(后缀 `_API_KEY`/`_TOKEN`/`_SECRET` 等 + 精确名单),`_hermetic_environment`(autouse)清空所有凭证形状变量与会改行为的 `HERMES_*`(含各平台 `*_ALLOWED_USERS`),防本地 key 干扰自动检测类测试。
- 隔离用 `tmp_path` 造 per-test `HERMES_HOME`(预建 sessions/cron/memories/skills),`get_hermes_home()` 读到的全是临时目录;**故意不改 HOME**(曾致 CI 挂),并把"用 `Path.home()` 而非 `get_hermes_home()`"定为 callsite bug。
- 确定性:`TZ=UTC`/`LANG=C.UTF-8`/`PYTHONHASHSEED=0`,并禁 AWS IMDS 查询省掉每测试 ~2s 的 timeout。
- `run_tests_parallel.py` **每文件一个子进程**(全新解释器),物理上杜绝跨文件状态泄漏;选每文件而非每测试是成本权衡(~3.5min vs ~70min);**弃用 xdist**因其持久 worker 会累积状态。
- `_live_system_guard`(autouse,约第 538 行)拦截 `os.kill`/`subprocess.*`/`os.system`/`pty.spawn`/`asyncio.create_subprocess_*`,检查整条命令串(`bash -c`/`sudo`/`env`/`setsid` 全覆盖),防测试误杀真实网关;需要时 `@pytest.mark.live_system_guard_bypass` opt-out。
- 断言把安全属性钉成回归网:`test_redact.py` 断言密钥原文不出现在脱敏结果;`test_pii_redaction.py` 断言 `_hash_id` 确定性、12 位十六进制、`user_` 前缀且原 ID 不泄露——"保留结构去内容"精确到字符长度。

## 动手实验

1. **实验一:审计封闭不变量** —— `Read` `tests/conftest.py` 的 `_hermetic_environment`(约第 327 行)。列出它清空/设置的所有 env 类别。论证:如果删掉"清空凭证 env"这一步,一个测试"当 `OPENAI_API_KEY` 存在时应自动选 OpenAI provider"会怎样在本地和 CI 表现不一致?

2. **实验二:理解每文件隔离** —— `Read` `scripts/run_tests_parallel.py` 头部注释。回答:为什么"每文件一个子进程"能消灭跨文件状态泄漏,而文件内的顺序耦合却要测试作者自己负责?为什么不做"每测试一个子进程"?为什么弃用 xdist?

3. **实验三:触发 live system guard** —— `Read` `_live_system_guard`(约第 538 行)。设想一个测试间接调用了执行 `sudo systemctl restart hermes-gateway` 的代码。推演:这条命令会真的执行吗?guard 检查 `tokens[0]` 还是整条命令串?如果某测试确实需要给自己的子进程发信号,该怎么 opt-out?

4. **实验四:把契约读成断言** —— `Read` `tests/agent/test_redact.py` 与 `tests/gateway/test_pii_redaction.py`。挑 `test_openai_sk_key` 和 `test_hash_sender_id_prefix` 两个,各写出它们守护的是第几章的哪条安全属性。设想有人重构脱敏正则导致 `sk-` key 不再被遮,哪条断言会先变红?

> **下一章预告**:十四章逐个子系统读下来,我们在不同模块里反复撞见同一批设计直觉——默认即关闭、约束只能收紧、错误即文档、边界不信任、留结构去内容、不变量进类型与测试、可信即流水线。第 15 章作为全书收束,将把这些散落各处的证据归纳成可迁移的工程原则,每条都回指前面章节里读到过的确切代码,让你带走的不只是"Hermes 怎么写的",而是"一个可信的 Agent 该怎么造"。
