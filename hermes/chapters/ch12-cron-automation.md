# 第 12 章　定时自动化：cron 调度与无人值守的回合

到目前为止,Hermes 的每一次行动都由人触发——你发条消息,它干件事。但 README 里那个"早上醒来发现它已经把昨夜的事办好了"的承诺,需要 Agent 能**自己醒来**:每天早上汇总新闻、每小时检查一次监控、半小时后提醒你回到某件事。承载这种"无人值守自动化"的是 `cron/` 子系统——`cron/jobs.py`(任务存储)和 `cron/scheduler.py`(调度执行,实测近 10 万字节)。

无人值守是个特别考验工程严谨度的场景:没人盯着,出了错没人立刻发现;多个进程可能同时醒来抢着跑同一个任务;一个手改坏的任务配置可能让调度器每一拍都崩。这一章我们看 Hermes 怎么把"定时跑任务"这件听起来简单的事,做得在无人值守下也不出乱子。

## 12.1 调度模型:gateway 每 60 秒 tick 一次

先看整体节奏。`cron/__init__.py` 的模块文档讲清了运行方式:cron 任务由 gateway 守护进程自动执行(`hermes gateway install` 装成用户服务,或 `--system` 装成开机系统服务),而 **gateway 每 60 秒"tick"一次调度器**。每一拍,调度器检查有哪些任务到期、把它们跑掉。

模块文档还点出三类能力:按计划跑自动任务(cron 表达式 / 间隔 / 一次性)、自我调度提醒和后续任务、以及**在隔离会话里执行任务(无先前上下文)**。最后这点很重要——定时任务跑在一个干净的会话里,不带你之前聊天的历史,避免一个早上的新闻汇总任务莫名其妙地"记得"你昨天的私人对话。

## 12.2 三种调度的统一解析:`parse_schedule`

用户怎么表达"什么时候跑"?`cron/jobs.py` 的 `parse_schedule(schedule)`(约第 209 行)把多种人类写法统一解析成结构化的三类:`kind` 为 `"once"`(一次性)、`"interval"`(周期间隔)、或 `"cron"`(cron 表达式)。

它的解析顺序本身就是一份易用性说明书。先看 `every X` 前缀——`"every 30m"`、`"every 2h"` 解析成 `{"kind": "interval", "minutes": ...}`,这是"每隔多久重复一次"。再看是否像 cron 表达式:用 `parts = schedule.split()` 切分,若有 5 个及以上字段且前 5 个都匹配 `^[\d\*\-,/]+$`(cron 字段允许的字符),就当 cron 处理。这里有个诚实的依赖检查——cron 表达式需要 `croniter` 包,缺了就抛 `Cron expressions require 'croniter' package. Install with: pip install croniter`(约第 248 行);装了则用 `croniter(schedule)` 实际验证一遍,无效就报 `Invalid cron expression '...'`。这又是"错误即文档":错误信息直接告诉你缺什么包、怎么装。

接着判断 ISO 时间戳:含 `T` 或形如 `\d{4}-\d{2}-\d{2}` 的,用 `datetime.fromisoformat` 解析成一次性任务。这里有个时区上的细节值得称道——如果解析出的时间是 naive(无时区),代码立刻 `dt.astimezone()` 把它变成带本地时区的(约第 267 行)。注释解释了原因:`Make naive timestamps timezone-aware at parse time so the stored value doesn't depend on the system timezone matching at check time.`——**在解析时就固定时区,免得存的时候和检查的时候系统时区不一致导致任务跑错点**。无人值守任务一旦因时区漂移跑错时间,几乎没人会立刻发现,所以这个早绑定很关键。

最后兜底:像 `30m`、`2h`、`1d` 这样的纯时长,用 `parse_duration` 算出分钟数,`run_at = _hermes_now() + timedelta(minutes=minutes)`,变成"从现在起多久后跑一次"的一次性任务。全都不匹配时,抛出一条把四种格式连例子一起列全的错误(约第 286 行起)——再次,错误信息就是用法文档。

## 12.3 防重复执行:文件锁 + at-most-once

无人值守最怕的事之一:gateway 内部的 ticker、一个独立守护进程、一次手动 tick,可能同时醒来,把同一个到期任务跑两遍。Hermes 的第一道防线是**文件锁**。

`tick()`(约第 2055 行)一进来就抢锁:打开 `~/.hermes/cron/.tick.lock`,用 `fcntl.flock(lock_fd, fcntl.LOCK_EX | fcntl.LOCK_NB)`(Unix)或 `msvcrt.locking(..., msvcrt.LK_NBLCK, 1)`(Windows)加**非阻塞排他锁**。注意 `LOCK_NB`(non-blocking)——抢不到锁不等待,直接 `except (OSError, IOError)` 走"另一个实例正持锁"分支,记一条 debug 日志后 `return 0`(约第 2082 行)。函数文档明说返回值是"执行的任务数(若另一个 tick 正在运行则为 0)"。非阻塞是对的选择:与其让两拍排队叠在一起,不如直接跳过这一拍,反正 60 秒后还有下一拍。

第二道防线是 **at-most-once(至多一次)语义**。抢到锁后,代码做了个关键动作:对所有到期任务**先**调 `advance_next_run(job["id"])` 把 `next_run_at` 推进到下一次,**然后**才开始执行(约第 2098 行)。注释写得很清楚:`Advance next_run_at for all recurring jobs FIRST, under the file lock, before any execution begins. This preserves at-most-once semantics.` 先推进再执行,意味着即便执行过程中崩了,任务的下次时间也已经往前走了,不会在下一拍被当成"还没跑"而重复触发。对已经在并行跑的任务,`advance_next_run` 持续把 `next_run_at` 往前顶,让宽限窗口不过期;任务完成时 `mark_job_run()` 再覆盖 `next_run_at`。

## 12.4 并行与串行:两个线程池的隔离

到期任务可能不止一个,怎么并发跑又不互相干扰?Hermes 用了**两个线程池**,分得很有讲究。

`_get_parallel_pool(max_workers)`(约第 176 行)是并行池,跑互不干扰的任务,worker 数由 `HERMES_CRON_MAX_PARALLEL` 环境变量 > config.yaml > 无上限决定(约第 2105 行起)。而 `_get_sequential_pool()`(约第 191 行)是个**只有一个 worker** 的串行池。为什么需要单 worker 池?注释讲透了:`A single worker guarantees env/context-mutating jobs never overlap, even across ticks.`——有些任务会改 `os.environ`、改 profile 状态,这类任务必须串行,否则一个任务改了环境变量、另一个任务读到半改的状态就乱套了。单 worker 保证它们永不重叠,甚至跨 tick 也按序——新一拍排队的任务会等上一拍的串行任务跑完。两个池都用 `atexit.register(_shutdown_parallel_pool)` 在进程退出时优雅关闭。

最典型的"会改环境"的任务是**带 profile 的任务**。`_job_profile_context`(约第 240 行)让单个任务能临时切到另一个 Hermes profile 跑:它 `set_hermes_home_override` 改 home、快照 `os.environ`,跑完在 `finally` 里恢复。恢复用的是**增量恢复(delta-based restore)**——只删新增的 key、只还原被改的 key(约第 295 行起),注释解释这是为了"避免有个短暂窗口让其他线程看到空的 env"。正因为这种任务会临时改全局环境,`tick()` 才把 profile 任务放进串行池跑,保证这份临时改动对其他任务隔离。

## 12.5 `wakeAgent` 门控:不是每次到期都要惊动模型

并非每个定时任务到期都值得真去跑一轮(昂贵的)Agent 推理。比如一个"检查监控"的任务,大多数时候监控正常、根本没必要叫醒模型说话。Hermes 为此设计了 `wakeAgent` 门控。

逻辑在约第 1089 行:一个任务可以先跑个脚本(gate script),脚本若返回形如 `{"wakeAgent": false}` 的 JSON,**就完全跳过 Agent**——不构造 SessionDB、不导入 run_agent、不花一次模型调用。判断函数 `gate.get("wakeAgent", True) is not False`(约第 1107 行)——缺省、缺失、或 `wakeAgent: true` 都表示正常唤醒,只有显式的 `false` 才短路。代码注释强调这个检查发生在 `BEFORE importing run_agent / constructing SessionDB`(约第 1407 行),也就是说短路省下的不只是模型调用,连重对象的构造都省了。对一个每小时甚至每分钟醒一次的任务来说,这个"没事别叫醒我"的门控,直接决定了无人值守自动化是否经济可行。

## 12.6 防御手改坏的配置:把畸形数据当常态

无人值守还有个隐蔽的坑:`jobs.json` 可能被迁移脚本或手工编辑搞出畸形数据,而这种错误如果让调度器每拍都崩,就成了"无声的死循环"。Hermes 对此有专门防御。

`_resolve_origin`(约第 305 行)就是个活教材。任务的 `origin` 字段本应是个 dict(带 platform/chat_id 路由信息),但注释记录了真实事故 #18722:有个任务的 origin 被写成了自由文本字符串 `"combined-digest-replaces-x-and-y"`,于是每次触发都因 `'str' object has no attribute 'get'` 崩溃——`mark_job_run` 记下失败,但下一拍重新加载同样的"中毒 origin",崩得一模一样,直到有人手动改字段才好。修复就是 `if not isinstance(origin, dict): return None`——**把非 dict 的 origin 当成"缺失"而非崩溃理由**。类似地,输出路径用 job_id 拼接前,`_job_output_dir`(约第 55 行)会校验 job_id 合法性,拒绝非法 id 以防输出写逃出 cron 沙箱(路径逃逸防御)。这些都体现同一个态度:无人值守的代码必须假设"喂进来的数据可能是坏的",并保证坏数据只影响那一个任务,不拖垮整个调度循环。

把这一章串起来:cron 子系统让 Hermes 从"被动应答"变成"主动行动",而支撑无人值守可靠性的,是 `parse_schedule` 对多种调度的统一解析与时区早绑定、`tick()` 的非阻塞文件锁 + 先推进后执行的 at-most-once 语义、并行/串行双池对"会改环境的任务"的隔离、`wakeAgent` 门控对昂贵推理的按需短路,以及把畸形配置当常态处理的边界防御。没人盯着的时候,这些细节就是 Agent 不出乱子的全部底气。

## 本章小结

- cron 任务由 gateway 守护进程每 **60 秒 tick 一次**执行;任务跑在**隔离会话**(无先前上下文)里,避免定时任务沾染私人对话历史。
- `parse_schedule`(约第 209 行)把多种写法统一为三类 `kind`:`once`/`interval`/`cron`;cron 需 `croniter` 包,缺失时报带安装命令的错误。
- 时区早绑定:naive 时间戳在解析时即 `dt.astimezone()` 固定本地时区,避免存与查时系统时区不一致导致跑错点。
- 防重复第一道:`tick()`(约第 2055 行)用 `~/.hermes/cron/.tick.lock` 的**非阻塞排他锁**(`LOCK_EX|LOCK_NB`),抢不到锁直接 `return 0` 跳过本拍。
- 防重复第二道:抢锁后**先** `advance_next_run` 推进所有到期任务的 `next_run_at`、**再**执行,保证 at-most-once;完成时 `mark_job_run` 覆盖。
- 双线程池隔离:并行池(worker 数由 `HERMES_CRON_MAX_PARALLEL` 等控制)跑互不干扰任务;**单 worker 串行池**保证会改 `os.environ`/profile 的任务永不重叠(含跨 tick)。
- 带 profile 的任务用 `_job_profile_context` 临时切 home/env,`finally` 里**增量恢复**(只删新增、只还原改动),避免其他线程看到空 env;此类任务走串行池。
- `wakeAgent` 门控:gate 脚本返 `{"wakeAgent": false}` 则在导入 run_agent / 构造 SessionDB **之前**短路,省下模型调用与重对象构造,让高频任务经济可行。
- 边界防御:`_resolve_origin` 把非 dict 的 origin 当缺失(事故 #18722),`_job_output_dir` 校验 job_id 防输出路径逃逸——畸形配置只影响单个任务,不拖垮调度循环。

## 动手实验

1. **实验一:解析各种调度** —— `Read` `cron/jobs.py` 的 `parse_schedule`(约第 209 行)。对 `"30m"`、`"every 30m"`、`"0 9 * * *"`、`"2026-07-01T08:00"` 四个输入,分别说出解析出的 `kind` 和关键字段。思考:为什么 `30m` 是 `once` 而 `every 30m` 是 `interval`?

2. **实验二:推演锁竞争** —— `Read` `tick()`(约第 2055 行)开头的加锁逻辑。设想 gateway 的 ticker 和一次手动 `hermes cron tick` 在同一秒触发。推演:两者都能跑到期任务吗?抢不到锁的那个返回什么?为什么这里用 `LOCK_NB` 非阻塞而不是阻塞等待?

3. **实验三:验证 at-most-once** —— `Read` 约第 2098 行"先 advance_next_run 再执行"的逻辑与注释。设想一个 `every 5m` 的任务在执行中途进程崩溃。推演:下一拍这个任务会不会被当成"还没跑"而立即重跑?`advance_next_run` 先行如何防止这件事?

4. **实验四:理解 wakeAgent 短路** —— `Read` 约第 1089–1107 行的 `wakeAgent` 门控与约第 1407 行的"BEFORE importing run_agent"注释。构造一个"每分钟检查磁盘空间、正常则不打扰"的任务,说明 gate 脚本该返回什么 JSON 才能跳过模型;并解释短路发生在构造 SessionDB 之前省下了哪些开销。

> **下一章预告**:一个能读你文件、连你账号、跑在你机器上的 Agent,一旦需要导出日志做诊断或上报问题,怎么保证不把你的隐私一起泄出去?第 13 章将解构 Hermes 的可观测性与脱敏子系统——日志如何分级、敏感信息如何在导出时被掩码、以及"保留结构、抹掉内容"的脱敏哲学如何贯穿诊断导出的每一环。
