# 第 17 章　测试、可信度与工程原则综合

十六章走下来,我们把 Codex 从入口拆到了前端:第 1 章看 monorepo 与运行时入口,第 2-4 章拆主循环与 turn 生命周期,第 5-6 章拆工具系统与命令执行,第 7-8 章拆子 agent 与沙箱,第 9 章拆 guardian 与审批,第 10 章拆模型网关,第 11 章拆鉴权,第 12 章拆上下文工程,第 13 章拆 rollout 与 SQLite 持久化,第 14 章拆扩展体系,第 15 章拆可观测性,第 16 章拆 TUI 与 exec 两副前端。

这最后一章不再引入新的功能模块,而是做两件事:先补上贯穿全书却一直没正面拆过的一层——**测试与可信度**,看 Codex 如何用工程手段保证"它真的按我们读到的那样工作";再把全书的观察提炼成几条**与语言、框架无关、可迁移**的工程原则,每一条都回指前面具体章节作为证据。

这是为这次"解构 Codex"收束的一章。

---

## 17.1 测试不是附属品:一个共享的测试支撑层

先看一个数字。在 `codex-rs/` 里,引用 `TempDir`/`tempfile`/`tempdir` 的 Rust 文件超过 **425 个**——几乎每个会碰文件系统的模块都在用临时目录跑测试。这本身就说明:**测试隔离不是少数核心模块的特权,而是整个代码库的默认动作**。

支撑这一切的是 `core/tests/common/lib.rs` 这个共享测试支撑层。它的存在,把"如何写一个隔离、确定、可复现的测试"这件事固化成了团队公共资产。其中最能说明设计意图的,是 `load_default_config_for_test` 这个函数的文档注释:

> Returns a default `Config` whose on-disk state is confined to the provided temporary directory. Using a per-test directory keeps tests hermetic and avoids clobbering a developer's real `~/.codex`.

一句话点出了两个目标:**hermetic(密封)** 和 **不污染开发者真实的 `~/.codex`**。每个测试拿到的 config,其所有落盘状态都被关进一个 `TempDir` 里,测试之间互不干扰,也绝不会动到你本机的真实配置。这正是第 13 章持久化、第 11 章鉴权那些"会写文件"的模块能被安全测试的前提。

它还做了一件容易被忽略但极重要的事——`without_managed_config_for_tests()`:测试时显式关掉"受管配置"的加载,让测试不受外部企业策略影响,行为完全由测试自己掌控。**测试要可复现,就必须把所有外部不确定性掐断在入口。**

---

## 17.2 把"不确定"变成"确定":determinism 的几处硬开关

测试最大的敌人是不确定性——时间、随机数、进程 ID、并发顺序。Codex 在测试支撑层用 `#[ctor]`(进程启动时、测试线程开跑前执行)装了几个确定性开关:

```rust
#[ctor]
fn enable_deterministic_unified_exec_process_ids_for_tests() {
    codex_core::test_support::set_thread_manager_test_mode(true);
    codex_core::test_support::set_deterministic_process_ids(true);
}
```

`set_deterministic_process_ids(true)`——**让进程 ID 在测试里变成确定的**。第 5、6 章的命令执行会产出进程 ID,如果这些 ID 是真实的、随机的,涉及它们的快照测试就永远在变。把它们设成确定的,断言才有意义。

还有一个 `#[ctor]` 处理快照测试的工作区根:`configure_insta_workspace_root_for_snapshot_tests` 把 `INSTA_WORKSPACE_ROOT` 指到仓库的 `codex-rs` 目录——这样 insta(快照测试框架)生成的快照路径在任何机器上都一致。**确定性不是测试时临时凑的,而是在进程启动的最早时刻就被钉死。**

这种"把不确定源头变成可注入/可固定的开关"的做法,正是第 15 章可观测性、第 13 章持久化里反复出现的依赖注入思想在测试层的延伸:能注入时钟、能注入 reader、能固定 ID,测试才能脱离真实世界独立运行。

---

## 17.3 等待异步事件:断言要对准"语义"而非"时序"

agent 是高度异步的:提交一个 op,事件会陆陆续续从流里冒出来。测试怎么对一个异步事件流做断言?Codex 给出的答案是一组 `wait_for_event` 系列工具(`core/tests/common/lib.rs`):

- `wait_for_event(codex, predicate)`——循环 `next_event()`,直到某个事件满足谓词;
- `wait_for_event_match`——不只等,还把匹配到的事件**取值**出来供后续断言;
- `wait_for_event_with_timeout`——所有等待都带超时(注释说明会给 async 启动工作留够时间,`wait_time.max(Duration::from_secs(10))`),**超时直接 panic 报 "timeout waiting for event"**。

这套设计的精妙在于:测试不写"sleep 500ms 然后检查"(那是时序耦合,既慢又脆),而是写"一直等,直到出现满足语义条件的那个事件"。`submit_thread_settings` 是个范例——它提交一个 op,然后循环等待,只接受 `ThreadSettingsApplied`,碰到 `Error` 立刻 panic 报出失败原因,碰到别的事件也 panic("unexpected ... event")。**断言对准的是"发生了什么"(语义),不是"什么时候发生"(时序)。**

而且这些 helper 在失败时给的信息都是可操作的:不是干巴巴的 `assertion failed`,而是 "stream ended unexpectedly while waiting for MCP startup"、"timeout waiting for thread settings update"。这呼应了我们后面会提炼的"错误即文档"——连测试 helper 的 panic 信息,都在教你哪里出了问题。

---

## 17.4 用假服务器测真链路:wiremock 与可断言的 ResponseMock

agent 的核心是跟模型 API 对话,但测试不能真去打 OpenAI 的接口——又慢、又花钱、又不确定。Codex 用 `wiremock`(`core/tests/common/responses.rs`)起一个本地假 HTTP 服务器,扮演模型后端,让整条"core → 网关 → API"的链路在测试里跑起来,但对端是可控的。

`ResponseMock` 这个结构尤其值得看:它不只是"返回假响应",还**捕获并记录所有发出去的请求**(`requests: Arc<Mutex<Vec<ResponsesRequest>>>`),并提供一组断言接口——`single_request()`(断言恰好一个请求)、`saw_function_call(call_id)`(断言某次工具调用确实发出去了)、`function_call_output_text(call_id)`(取出某次工具调用的返回文本)。

这意味着测试能断言的不只是"core 收到了什么",还有"core 到底往模型发了什么"。第 12 章上下文工程里"我们到底把哪些东西塞进了 prompt"、第 6 章工具调用里"工具结果有没有正确回灌给模型"——这些关键契约,都能靠 `ResponseMock` 捕获的真实请求来精确验证。**用假的对端,测真的链路,并把链路上的每一次交互都变成可断言的对象。**

---

## 17.5 跳过而非假过:测试对环境差异的诚实

最后看一组很能体现工程品味的宏:`skip_if_sandbox!`、`skip_if_no_network!`、`skip_if_remote!`、`codex_linux_sandbox_exe_or_skip!`、`skip_if_windows!`。

它们解决的是一个现实问题:有些测试在某些环境下根本没法跑——沙箱里跑不了需要真实 seatbelt 的用例,断网环境跑不了需要网络的用例,Windows 上跑不了 Unix 专属的用例,拿不到 `codex-linux-sandbox` 二进制就跑不了 Linux 沙箱用例。

面对这种情况,有两种态度。坏的态度是:在这些环境里让断言放水、假装通过(测试变绿但毫无意义)。Codex 选的是好态度:**显式跳过,并打印为什么跳过**——比如 `skip_if_sandbox!` 检测到 `CODEX_SANDBOX=seatbelt` 就 `eprintln!` 一句 "... skipping test" 然后 `return`。测试结果如实反映"这条没跑",而不是"这条过了"。

这是一种诚实:**宁可承认"我在这个环境下没验证",也不伪造一个"通过"**。测试的价值在于它说真话;一个会在不该过的时候假装过的测试,比没有测试更危险。

---

至此,贯穿全书的"内核"已经拆完,"如何相信内核"也补齐了。下面把十七章的观察收成几条可迁移的工程原则。每条原则都不是凭空提炼,而是从我们实际读过的代码里长出来的。

---

## 17.6 原则一:默认即关闭(Fail-closed defaults)

**不确定时,选更安全、更受限的那一侧。**

证据贯穿全书:第 9 章 guardian 与审批,缺省走"问"或"拒"而非放行;第 16 章 exec 的 `handle_server_request` 对所有不支持的交互请求默认报错 "not supported in exec mode",而非默默放行;第 15 章遥测在 `cfg!(debug_assertions)`(debug 构建)下**强制把 Statsig 导出器关成 `None`**,本地开发默认不发任何遥测流量。

识别这条原则的方法:看常量里的 `DEFAULT_*`、看 switch/if 的 fallback 分支走的是拒绝还是放行。在 Codex 里,fallback 几乎总是走更保守的一侧。**安全的系统,默认值本身就是一道防线。**

---

## 17.7 原则二:边界不信任外部输入(Boundary distrust)

**任何跨信任边界进来的数据,先校验、再净化,然后才用。**

证据:第 16 章 exec 的 `decode_prompt_bytes`——管道进来的字节先剥 BOM、拒绝 UTF-32、解码 UTF-16,失败给结构化 `PromptDecodeError` 而非 panic;第 15 章遥测标签的两道防线——`validate_tag_value` 字符白名单校验 + `sanitize_metric_tag_value` 主动清洗 + `bounded_originator_tag_value` 把开放维度强行收敛到 `KNOWN_ORIGINATOR_TAG_VALUES` 有限集合;第 8 章沙箱对命令的隔离;第 16 章 exec 进主循环前的 git 信任检查。

识别方法:在 parse 层、IO 边界、反序列化处找白名单/正则/重解析校验。Codex 的一致姿态是:**外部输入在跨过边界的那一刻就被当成不可信,边界内的代码才能安心。**

---

## 17.8 原则三:错误即文档(Errors as documentation)

**错误信息本身教人正确用法,而不只是抛一个栈。**

证据:第 16 章 exec 拒绝交互时说的是 "… is not supported in exec mode …"(告诉你这个模式做不了什么),git 检查失败说的是 "Not inside a trusted directory and --skip-git-repo-check was not specified."(连绕过它的 flag 名都给了);`decode_prompt_bytes` 对 UTF-32 报错时附带 iconv 转码提示;本章 17.3 的测试 helper,panic 信息也是 "timeout waiting for thread settings update" 这种可定位的句子。

识别方法:看校验失败分支、自定义 Error 类型、断言信息里有没有可操作的修复建议、字段名、期望格式。在 Codex 里,**报错是一次沟通,不是一次甩锅。**

---

## 17.9 原则四:策略与机制分离(Policy/mechanism separation)

**"做什么"与"怎么呈现/怎么执行"解耦,让同一个机制服务于不同策略。**

最纯粹的证据是第 16 章:exec 用一个 `EventProcessor` trait 抽象"消费事件流"这个机制,挂上人类版(`EventProcessorWithHumanOutput`,给人看的彩色 stderr)和机器版(`EventProcessorWithJsonOutput`,给程序读的 JSONL)两套策略,主循环代码一行不改。再往大看,整个第 16 章的论点——**同一个 core,两副前端(TUI 与 exec)**——就是这条原则的最大化体现:内核只认 app-server 协议这个机制,人在回路还是全自动批处理这两种策略,都建在同一条协议边界上。

识别方法:找 trait/interface 背后的多实现,找"只认协议、不认前端"的窄边界。**当机制与策略分离,长出第三种、第四种策略(IDE 插件、Web、语音)就不必再动机制。**

---

## 17.10 原则五:可观测性覆盖失败路径(Observe the failure paths)

**给降级和失败路径埋够观测,而不是只给成功路径埋点。**

证据:第 13 章持久化的 `record_fallback(操作, 原因)`,把每一次回退按原因(`db_unavailable`/`db_error`/`missing_row`/`backfill_incomplete`…)量化;第 15 章把"降级是一等观测对象"作为压轴小节;第 16 章 exec 的 `error_seen` → `exit(1)` 把"出错了"翻译成自动化能读懂的非零退出码,以及 `maybe_backfill_turn_completed_items` 专门为"背压丢了通知"这条失败路径做补偿并可被观测。

识别方法:看失败分支、降级分支有没有对应的指标/日志/退出码。一个系统的成熟度,**不在于它成功时记了多少,而在于它失败时能不能说清楚自己怎么失败的。**

---

## 17.11 原则六:可信度即流水线(Trustworthiness as a pipeline)

**"可信"不是一个开关,而是依赖注入 + 测试隔离 + 语义断言 + 确定性组成的一条流水线。**

证据集中在本章:17.1 的 hermetic config(每测试一个 `TempDir`、`without_managed_config_for_tests`)、17.2 的确定性开关(`set_deterministic_process_ids`、固定 `INSTA_WORKSPACE_ROOT`)、17.3 的语义化等待(`wait_for_event` 系列对准语义而非时序)、17.4 的可断言假服务器(`ResponseMock` 捕获并断言真实请求)、17.5 的诚实跳过(`skip_if_*` 宁可承认没测也不假装通过)。这几样叠加起来,才让"我们读到的代码真的按它声称的那样工作"这件事变得可信。

识别方法:看可注入的参数(时钟、reader、ID 开关)、测试隔离基建(temp HOME、固定根、确定性)、多层校验链。**可信度是被一层层工程手段累积出来的,没有银弹。**

---

## 17.12 收束:一条贯穿始终的脊梁

把六条原则再压缩一次,会发现它们其实指向同一根脊梁:**Codex 在每一个边界上都选择诚实与保守**。

- 对外部输入,它不信任(原则二);
- 对不确定,它选最安全的默认(原则一);
- 对失败,它如实记录、如实退出(原则五);
- 对用户和开发者,它用错误信息说清楚(原则三);
- 对自己的前端形态,它用一条窄协议把变化挡在内核之外(原则四);
- 对"它是否真的可信"这个元问题,它用一整条测试流水线来回答(原则六)。

这六条没有一条依赖 Rust、依赖 ratatui、依赖 OpenAI 的 API——它们是任何一个想被信任的 agent 都该具备的工程品格。语言会变,框架会变,模型会变,但"在边界上保持诚实与保守"这件事不会过时。

这,就是解构 Codex 这十七章,最想留下的东西。

---

## 本章小结

- **测试隔离是默认动作**:`codex-rs/` 里 425+ 文件用 `TempDir`/`tempfile`;`core/tests/common/lib.rs` 的 `load_default_config_for_test` 把每个测试的落盘状态关进 `TempDir`,保证 hermetic 且不污染真实 `~/.codex`;`without_managed_config_for_tests()` 掐断外部策略影响。
- **确定性在进程启动时钉死**:`#[ctor]` 装 `set_deterministic_process_ids(true)`、`set_thread_manager_test_mode(true)`,并把 `INSTA_WORKSPACE_ROOT` 指向 `codex-rs`,让快照路径跨机一致。
- **断言对准语义而非时序**:`wait_for_event`/`wait_for_event_match`/`wait_for_event_with_timeout`(超时即 panic)循环等待满足谓词的事件;`submit_thread_settings` 只接受 `ThreadSettingsApplied`,碰 `Error`/意外事件即 panic 并报因;不写 sleep。
- **用假服务器测真链路**:`core/tests/common/responses.rs` 用 `wiremock` 起本地假模型后端;`ResponseMock` 捕获所有出站请求并提供 `single_request`/`saw_function_call`/`function_call_output_text` 等断言,验证 core 到底往模型发了什么。
- **诚实跳过而非假过**:`skip_if_sandbox!`/`skip_if_no_network!`/`skip_if_remote!`/`codex_linux_sandbox_exe_or_skip!`/`skip_if_windows!` 在无法运行的环境下显式跳过并打印原因,而非让断言放水。
- **六条可迁移工程原则**(每条回指章节证据):① 默认即关闭(ch9/ch15/ch16);② 边界不信任外部输入(ch8/ch15/ch16);③ 错误即文档(ch16 + 测试 helper);④ 策略与机制分离(ch16 一 trait 两实现、一 core 两前端);⑤ 可观测性覆盖失败路径(ch13 `record_fallback`、ch15、ch16 `exit(1)`/背压补偿);⑥ 可信度即流水线(本章 DI + 隔离 + 语义断言 + 确定性)。
- **一根脊梁**:Codex 在每个边界上都选择诚实与保守——这套品格与语言/框架无关,是任何想被信任的 agent 的共同基础。

## 动手实验

1. **实验一:验证 hermetic 测试** — 阅读 `core/tests/common/lib.rs` 的 `load_default_config_for_test` 及其文档注释。解释"per-test `TempDir`"如何同时实现 hermetic 和"不污染 `~/.codex`"两个目标。再说明 `without_managed_config_for_tests()` 去掉了什么、为什么去掉它对测试可复现性是必要的。
2. **实验二:追踪一个确定性开关** — 找到 `#[ctor]` 修饰的 `enable_deterministic_unified_exec_process_ids_for_tests`,解释 `set_deterministic_process_ids(true)` 解决了什么测试难题。假设关掉它,第 5/6 章涉及进程 ID 的快照测试会出现什么现象?
3. **实验三:语义等待 vs 时序等待** — 阅读 `wait_for_event_with_timeout` 与 `submit_thread_settings`。把它改写成"sleep 固定时长后检查"的伪代码,列出这种写法的至少两个缺陷(慢、脆),并说明 `wait_for_event` 为什么更健壮。再指出 helper 的 panic 信息如何体现"错误即文档"。
4. **实验四:用六条原则反向审视一章** — 任选前面某一章(如 ch8 沙箱或 ch13 持久化),用 17.6–17.11 的六条原则当 checklist,逐条在那一章的代码里找对应证据(找到就记下确切的文件/函数/常量)。若某条原则在该章没有体现,说明为什么——这正是"原则必须从代码里长出来,而非套模板"的练习。

> **全书完**:从第 1 章的运行时入口,到这一章的工程原则综合,我们用十七章把 OpenAI Codex 这个 agent 从外到内、从启动到收束完整解构了一遍。每一个技术论断都来自实际读过的源码,每一条原则都回指具体证据。愿这次解构留下的,不只是对 Codex 的理解,更是一套可以带去解构、乃至构建下一个可信 agent 的透镜。
