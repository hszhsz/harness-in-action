# 第 16 章　前端形态：TUI 与 exec

第 15 章把可观测性拆透了——指标、分析、诊断,让我们能从外部看清 Codex 内核每一次决策与每一次降级。但到这一章为止,我们一直在谈"内核":循环、工具、沙箱、上下文、持久化、扩展、观测。用户真正用手指触摸到的,却是它的**前端形态**。

Codex 有两个截然不同的前端:一个是 `tui`——基于 ratatui 的交互式终端界面,人坐在屏幕前敲键、看流式输出、按 y/n 审批;另一个是 `exec`——非交互、可脚本化的子命令,跑在 CI、cron、管道里,没有人盯着,所有输出要么是干净的最终答案,要么是机器可解析的 JSONL。

这一章的核心论点只有一句:**两个前端共享同一个引擎**。它们都通过 app-server 协议驱动同一套 core,差别不在"能力",而在"交互假设"——一个假设有人在,一个假设没人在。把这条线拆开,你会看到一个优秀的工程团队如何用同一个内核,长出两副完全不同却又各自自洽的皮肤。

代码主要在两个 crate:`exec/`(非交互)和 `tui/`(交互),它们各自的入口逻辑在 `exec/src/lib.rs` 和 `tui/src/lib.rs`。

---

## 16.1 同一个内核,两个客户端

先看两个 crate 怎么连到 core。`exec/src/lib.rs` 用的是 `InProcessAppServerClient::start(in_process_start_args)`——它在**同一个进程内**起一个 app-server,然后像普通客户端一样通过协议跟它对话。启动参数里写明了身份:`session_source: SessionSource::Exec`、`client_name: "codex_exec"`。

`tui/src/lib.rs` 更进一步——它同时 `use` 了两种客户端:`InProcessAppServerClient`(进程内)和 `RemoteAppServerClient`(连一个远端 app-server,配套 `RemoteAppServerConnectArgs`、`RemoteAppServerEndpoint`)。也就是说,TUI 既能像 exec 那样进程内驱动 core,也能连到一个独立运行的 app-server 进程上去。

这就是整本书反复出现的那条设计线的终点:**core 不知道前端是谁**。它只认 app-server 协议——收 `AppCommand`(submit op),发 `ServerNotification`(thread item / turn 状态 / 错误)。exec 和 tui 不过是这套协议的两个消费者。同一个 `ThreadItem::AgentMessage` 流过来,exec 把它打到 stdout,tui 把它渲染进滚动历史——内核完全无感。

一个直接的证据是 `exec/src/lib.rs` 里对 originator 的设置:`set_default_originator("codex_exec")`。而第 15 章我们见过 `KNOWN_ORIGINATOR_TAG_VALUES` 这张白名单里同时有 `codex-tui` 和 `codex_exec`——**同一套遥测系统,靠 originator 标签区分是哪个前端在说话**。前端是可替换的标签值,内核是不变的主体。

---

## 16.2 exec 的第一纪律:stdout 是契约

`exec` 跑在没有人的地方,它的输出会被另一个程序读。所以 `exec/src/lib.rs` 开篇就立了一条编译期红线:

```rust
#![deny(clippy::print_stdout)]
```

**整个 crate 默认禁止 `println!`**。文件头注释把契约说得清清楚楚:默认模式下,只有最终消息会进 stdout;`--json` 模式下,stdout 是合法的 JSONL——每行一个事件;**其余一切都进 stderr**。

为什么这么狠?因为对一个被管道消费的程序来说,stdout 不是"打印的地方",而是**与下游的数据契约**。如果进度信息、警告、调试日志混进 stdout,下游 `jq` 一解析就炸。把红线钉在 clippy 层面,意味着任何想往 stdout 写东西的代码都得显式 `#[allow(clippy::print_stdout)]` 自证清白——我们在 16.3 会看到这个豁免出现在哪里,而它出现的地方少得可数,正说明这条纪律被严格守住了。

人类输出处理器 `EventProcessorWithHumanOutput` 把这条纪律落到了实处。看它的 `render_item_started` / `render_item_completed`:**所有过程性输出——`exec`/`mcp:`/`web search:`/`patch:`/工具退出码/推理摘要/token 用量——一律用 `eprintln!` 走 stderr**。整个文件里只有一处例外,就在 `print_final_output`:它用 `#[allow(clippy::print_stdout)]` 把**最终答案**用 `println!` 送进 stdout。过程归 stderr,结果归 stdout——这就是 exec 对外的全部契约。

---

## 16.3 两种输出处理器:人看的 vs 机器读的

exec 内部用一个 trait 把"如何呈现事件"抽象出来。`exec/src/event_processor.rs` 定义:

```rust
pub(crate) trait EventProcessor {
    fn print_config_summary(&mut self, config: &Config, prompt: &str, session_configured: &SessionConfiguredEvent);
    fn process_server_notification(&mut self, notification: ServerNotification) -> CodexStatus;
    fn process_warning(&mut self, message: String) -> CodexStatus;
    fn print_final_output(&mut self) {}
}
```

`process_*` 方法返回 `CodexStatus`(`Running` 或 `InitiateShutdown`),由它告诉主循环"这一轮该继续还是该收摊"。同一个事件流,接两种实现:

**人类版 `EventProcessorWithHumanOutput`**(`event_processor_with_human_output.rs`)。它持有一组 `owo_colors` 的 `Style`(bold/cyan/dimmed/green/italic/magenta/red/yellow),通过 `create_with_ansi(with_ansi, ...)` 决定上色还是输出纯文本——**输出到管道(非 TTY)时自动退化为无色**,避免 ANSI 转义码污染下游。它把每个 `ThreadItem` 渲染成给人看的行:命令执行打 `exec ... in <cwd>`,完成时按状态打 ` succeeded in 123ms:`(绿)/ ` exited 1:`(红)/ ` declined:`(黄);MCP 工具打 `mcp: server/tool (completed)`;计划更新用 `✓`/`→`/`•` 三个符号画进度。

**机器版 `EventProcessorWithJsonOutput`**(`event_processor_with_jsonl_output.rs`)。它维护一个 `next_item_id: AtomicU64`(产出 `item_0`、`item_1`……)和 `raw_to_exec_item_id: HashMap`,把内部事件映射成稳定的 `ThreadEvent`,再用 `emit()` 一行一个 JSON 打出去。`emit()` 正是那处罕见的 `#[allow(clippy::print_stdout)]`——因为在 `--json` 模式下,**JSONL 本身就是契约定义的 stdout 内容**,这里 `println!` 是合法的。

一个 trait,两套呈现,主循环代码一行不改。这是策略模式在前端层最朴素也最有效的应用:**"做什么"(消费事件流)与"怎么呈现"(给人 vs 给机器)彻底解耦**。

---

## 16.4 exec 拒绝交互:没人时,默认不问

exec 最关键的设计约束是:**没有人能回答问题**。所以它必须在两个层面把"需要人"的路径全部堵死。

**第一层:把审批策略默认设成"从不"**。`exec/src/lib.rs` 在装配 config 时,headless 默认 `approval_policy: Some(AskForApproval::Never)`;沙箱上 `removed_full_auto` 落到 `WorkspaceWrite`,`dangerously_bypass` 落到 `DangerFullAccess`。也就是说,exec 默认不会走到"该不该批准"的岔路口。

**第二层:即便 core 真的发来一个交互请求,也程序化地拒绝或取消**。这体现在 `handle_server_request` 里。对一整类请求——`ExecCommandApproval`、`FileChangeApproval`、`ApplyPatchApproval`、`PermissionsRequestApproval`、`ToolRequestUserInput`、`DynamicToolCall`、`ChatgptAuthTokensRefresh`、`AttestationGenerate`——exec 直接报错回去:"… is not supported in exec mode …"。**它不假装能批准,而是诚实地说"这个模式做不了"**。

唯一被特殊对待的是 MCP 的交互式表单 `McpServerElicitationRequest`:exec 不报错,而是**自动取消**——调 `canceled_mcp_server_elicitation_response()`,带 `McpServerElicitationAction::Cancel`。为什么对它网开一面?因为 elicitation 是一个可以"取消"的请求(取消是一个合法答案),而审批不行——你不能"取消"一个 y/n。**对每类交互,exec 都选了语义上最正确的非交互应答:能拒绝的拒绝,能取消的取消。**

把这两层放在一起,exec 的"无人值守"就不是靠运气,而是被两道闸门焊死的:策略上默认不问,实现上即便被问也有确定性的非交互答案。

---

## 16.5 输入的鲁棒性:从 stdin 解码一段可能很脏的字节

非交互前端的输入往往来自管道,而管道里的字节可能带各种编码"杂质"。`exec/src/lib.rs` 的 `decode_prompt_bytes` 处理得相当较真:

- 剥掉 UTF-8 BOM;
- 显式**拒绝** UTF-32LE/BE,报错并提示用 iconv 转码;
- 能识别并解码 UTF-16LE/BE;
- 解码失败时返回结构化错误 `PromptDecodeError`(`InvalidUtf8 { valid_up_to }` / `InvalidUtf16 { encoding }` / `UnsupportedBom { encoding }`),而不是糊一个 panic。

prompt 从哪来,也有一套优先级。`resolve_prompt` / `resolve_root_prompt` / `prompt_with_stdin_context` 协同:`-` 这个 sentinel 强制从 stdin 读;若 stdin 是管道(piped),其内容会被包成 `<stdin>...\n</stdin>` 追加到 prompt 上下文里。`StdinPromptBehavior` 这个枚举把三种姿态固化下来——`RequiredIfPiped`(管道时必读)、`Forced`(强制读)、`OptionalAppend`(可选追加)。

**一个给机器用的入口,必须假设输入是脏的、来源是多样的,并为每种情况给出确定的行为或明确的报错。** 这正是 exec 在输入端的鲁棒性纪律,和它在输出端的 stdout 契约一脉相承——边界两端都不含糊。

---

## 16.6 退出码与背压补偿:让自动化可靠

exec 是给自动化用的,自动化靠**退出码**判断成败。它的主循环用 `tokio::select!` 同时盯两件事:ctrl_c 中断,和 `client.next_event()`。一旦事件流里出现错误,`error_seen` 被置位,收尾时 `std::process::exit(1)`——**非交互前端必须把"出错了"翻译成非零退出码**,否则 CI 永远是绿的,失败被悄悄吞掉。

进入主循环前还有一道闸门:**git 信任检查**。如果不在受信目录、又没带 `--skip-git-repo-check`,exec 直接退出并提示 "Not inside a trusted directory and --skip-git-repo-check was not specified.",除非用 `--yolo` 显式越过。自动化环境里"在哪跑"是个安全问题,这道检查不让你在不信任的目录里稀里糊涂地放开手脚。

还有一处很能体现"为可靠性兜底"的细节:`maybe_backfill_turn_completed_items` / `should_backfill_turn_completed_items`。in-process 客户端在背压(backpressure)下有可能丢掉 item 通知——事件来得太快,通道来不及消化。exec 不赌"通知一定都到齐了",而是在满足条件时(`!thread_ephemeral && payload.turn.items.is_empty()`)主动发一次 `thread/read`,**把这一 turn 的最终内容补回来**。对交互式 TUI 来说,丢几帧中间状态无伤大雅(下一帧重绘就好);但对 exec 来说,**丢掉最终消息 = 自动化拿到空结果**,这是不能接受的,所以它专门加了这层补偿。

最后,exec 还支持 resume:`resolve_resume_thread_id` 通过 `thread/list` + `thread/resume` 找回会话(代码注释强调 "exec no longer reaches into rollout storage directly"——它不再直接伸手进第 13 章的 rollout 存储,而是走协议)。UUID 直接命中,否则去 state_db 用 `find_thread_by_exact_title` 找,再不行就搜索。**连"恢复历史会话"这种本可以抄近路读文件的操作,exec 也坚持走 app-server 协议**——这又是 16.1 那条"前端不越过协议碰内核内部"的纪律。

---

## 16.7 TUI 的心脏:AppEvent 消息总线

切到交互式前端。TUI 的世界里有人,所以它要处理的是另一类问题:键盘、粘贴、窗口缩放、流式重绘、滚动历史。

它的组织核心是一条**内部消息总线**。`tui/src/app_event.rs` 的模块注释直接点明:`AppEvent` 是 "the internal message bus between UI components and the top-level App loop"——UI 组件与顶层 App 循环之间的内部消息总线。各个组件不直接互相调用,而是往这条总线投 `AppEvent`,由顶层 `App` 统一消费。退出也是一条消息:`AppEvent::Exit(ExitMode)`。

投递事件的门面是 `tui/src/app_event_sender.rs` 的 `AppEventSender`,它包着一个 `UnboundedSender<AppEvent>`。它的 `send()` 做了两件事:一是**为高保真会话回放记录入站事件**——调 `session_log::log_inbound_app_event`(但跳过 `CodexOp`,因为那些在提交点已另行记录,避免重复);二是发送失败时吞掉错误并 `tracing::error!` 记一笔,不让一次发送失败炸掉整个 UI。

`AppEventSender` 还提供了一组**类型化的便捷方法**,把常见出站命令包成 `AppEvent`:`interrupt`、`compact`、`set_thread_name`、`review`、`list_skills`,以及一组审批/应答方法——`exec_approval`、`patch_approval`、`request_permissions_response`、`resolve_elicitation`。注意这里:**16.4 里 exec 一律拒绝的那些审批请求,在 TUI 里全都有对应的应答方法**。同一个 core 发来的同一个审批请求,exec 说"不支持",TUI 说"我问问用户"——这就是"同一内核、两种交互假设"最具体的对照。这些审批类方法包成的不是 `CodexOp`,而是 `AppEvent::SubmitThreadOp { thread_id, op }`,因为审批是针对某个具体 thread 的应答。

---

## 16.8 TUI 的事件循环与终端事件

`App` 是顶层状态机,`tui/src/app.rs` 里它的 `run` 是主循环。它消费两类东西:从 `AppEventSender` 来的内部 `AppEvent`,和从终端来的 `TuiEvent`。后者定义在 `tui/src/tui.rs`:

```rust
enum TuiEvent {
    Key(KeyEvent),
    Paste(String),
    Resize,
    Draw,
}
```

值得注意的是 `Resize` 和 `Draw` 是分开的两个变体,不是合并成一个"该重绘了"。注释解释:把 `Resize` 单独拆出来,是为了给某些 feature-gated 的"预渲染"逻辑留一个钩子——缩放和普通重绘的时机需要分别处理。

`handle_tui_event` 按变体分派。一个很说明问题的细节是 `Paste` 的处理:它把粘贴文本里的 `\r` 规整成 `\n`,注释标注这是为了对付 iTerm2 的怪癖——不同终端粘贴时的换行符不一致,TUI 在边界处统一掉,后面的逻辑就不必到处判断。`Draw`/`Resize` 则触发 backtrack 重建、通知、`pre_draw_tick`,最后走 `render_chat_widget_frame`:先算 `desired_height = chat_widget.desired_height(width)`,再用 inline viewport(`draw_with_resize_reflow`)绘制。

退出时,`App` 收束出 `AppExitInfo { token_usage, thread_id, thread_name, update_action, exit_reason }`——把这一程的统计、会话标识、是否要触发更新、退出原因一并交还给上层。**交互式前端的"收尾"不是简单退出,而是把状态完整地交接出去。**

---

## 16.9 把历史写进滚动区:终端即历史

TUI 最别致的设计在于它对"历史"的处理。`tui/src/insert_history.rs` 的模块注释开宗明义:**Codex 直接把终端自身的滚动回看区(scrollback)当作已定稿的聊天历史**。所以"插入一行历史"不是一次普通的 ratatui 渲染,而是一次**转义序列操作**——把行写到 viewport 上方的滚动区里。

为什么这么设计?因为这样一来,定稿的历史就交给终端自己管理:用户可以用终端原生的滚动、搜索、复制去操作它,Codex 只需在屏幕底部维持一个活跃的 viewport(composer + 当前 turn 的实时输出)。**历史归终端,现场归 viewport**,职责清爽。

但"写进终端"有不少脏活。`insert_history_lines` 默认走 `HistoryLineWrapPolicy::PreWrap`,对换行做了三路处理:URL-only 的行**保持不折断**(`line_contains_url_like`),好让终端把它识别成可点击链接;URL 与散文混合的行用 `adaptive_wrap_line` 自适应折行(URL token 不拆、散文正常折);纯文本行同样走自适应折行。**为了让用户能点链接、能完整复制源文本,折行策略被拆到 token 粒度**——这是只有"假设有人在看"的前端才会去操心的细节。

`tui/src/tui.rs` 里的 `Tui` 结构承载了这套机制的运行时状态:`frame_requester`(请求重绘)、`draw_tx`(一个容量为 1 的 broadcast 通道,合并重绘请求)、`pending_history_lines`(缓冲待写的历史行)、`alt_screen_active`/`terminal_focused`(AtomicBool)、`is_zellij`(检测 Zellij 多路复用器),以及一个 `_stderr_guard`——把不受管理的 stderr 输出挡在 inline viewport 之外,免得乱码窜进界面。`insert_history_lines` 会把同策略的批次**合并**后再调度一帧,`flush_pending_history_lines` 真正把它们写到 viewport 上方(Zellij 下走特殊的 `InsertHistoryMode::ZellijRaw` 路径,因为 Zellij 不把软折行的续行约束在 Codex 的滚动区里,需要单独处理)。

最后是健壮性兜底:`set_panic_hook` 安装一个 panic 钩子,**进程崩溃时先把终端恢复原状**(退出 alt screen、恢复光标),再让 panic 信息正常打出来。一个会接管整个终端的程序,如果崩了不还原,用户的终端就成了一团乱麻——这个钩子是交互式前端的基本礼貌。

---

## 16.10 两副皮肤,一条脊梁

把这一章拼起来,exec 与 tui 的对照清晰可见:

| 维度 | exec(非交互) | tui(交互) |
|---|---|---|
| 假设 | 没有人 | 有人 |
| 连接 core | `InProcessAppServerClient` | 进程内 **或** `RemoteAppServerClient` |
| 输出契约 | stdout 是数据契约(`#![deny(clippy::print_stdout)]`) | 终端 scrollback 即历史 + 底部 viewport |
| 输出形态 | 人类版 / JSONL 版(一个 trait 两实现) | ratatui 渲染 + 流式重绘 |
| 审批 | 默认 `Never`,被问则拒绝/取消 | 包成 `SubmitThreadOp`,问用户 |
| 成败信号 | 退出码(`exit(1)`)+ 背压补偿 | `AppExitInfo` 交接 + panic 恢复终端 |
| 共同点 | **都只通过 app-server 协议驱动同一个 core** | |

最后一行才是这一章真正的脊梁。前端可以差异巨大——一个为机器,一个为人;一个守 stdout,一个画屏幕;一个拒绝交互,一个拥抱交互。但它们都**不越过协议去碰内核内部**:exec 恢复会话也走 `thread/list`/`thread/resume` 而非直读 rollout 文件,TUI 投递命令也走 `AppCommand` 而非直调 core 函数。

**正是这条被严格守住的协议边界,让"换一副皮肤"成为可能——甚至让 TUI 能选择连本地进程或远端 server 都不影响 core。** 当内核与前端之间只剩一条窄而清晰的协议时,你想长出第三副、第四副前端(IDE 插件、Web、语音),都不必动内核分毫。这是整本书我们反复见到的那条原则,在最贴近用户的一层,给出了最终的注脚。

---

## 本章小结

- **同一个内核,两个客户端**:exec 用 `InProcessAppServerClient::start`(`session_source: SessionSource::Exec`、`client_name: "codex_exec"`);tui 同时支持 `InProcessAppServerClient` 与 `RemoteAppServerClient`(本地进程或远端 server)。core 只认 app-server 协议,不知道前端是谁;靠 originator 标签(`codex_exec` vs `codex-tui`)区分。
- **exec 的 stdout 契约**:`#![deny(clippy::print_stdout)]` 钉死全 crate;默认模式只把最终答案 `println!` 进 stdout(唯一豁免 `#[allow]` 在 `print_final_output`),`--json` 模式 stdout 是 JSONL,其余一切 `eprintln!` 进 stderr。
- **两种输出处理器**:一个 `EventProcessor` trait(`process_*` 返回 `CodexStatus::{Running, InitiateShutdown}`),两套实现——人类版 `EventProcessorWithHumanOutput`(owo_colors 上色、非 TTY 退化无色)、机器版 `EventProcessorWithJsonOutput`(`AtomicU64` 产 `item_N`、`emit()` 打 JSONL)。策略模式解耦"消费事件"与"如何呈现"。
- **exec 拒绝交互**:默认 `approval_policy: Never`;`handle_server_request` 对 `ExecCommandApproval`/`FileChangeApproval`/`ApplyPatchApproval`/`PermissionsRequestApproval`/`ToolRequestUserInput` 等报 "not supported in exec mode";仅 `McpServerElicitationRequest` 自动取消(`McpServerElicitationAction::Cancel`)——能拒的拒、能取消的取消。
- **输入鲁棒性**:`decode_prompt_bytes` 剥 UTF-8 BOM、拒 UTF-32、解 UTF-16,失败给结构化 `PromptDecodeError`;`StdinPromptBehavior`(`RequiredIfPiped`/`Forced`/`OptionalAppend`)+ `-` sentinel + 管道内容包成 `<stdin>...</stdin>`。
- **退出码与背压补偿**:主循环 `tokio::select!`(ctrl_c vs `next_event`),`error_seen` → `exit(1)`;git 信任检查(`--skip-git-repo-check`/`--yolo`);`maybe_backfill_turn_completed_items` 在背压丢通知时发 `thread/read` 把最终消息补回;resume 走 `thread/list`+`thread/resume` 而非直读 rollout。
- **TUI 的 AppEvent 总线**:`AppEvent` 是 UI 组件与顶层 `App` 的内部消息总线;`AppEventSender.send()` 记录入站事件供会话回放(跳过 `CodexOp`)、吞发送错误;审批类方法包成 `SubmitThreadOp`——与 exec 的拒绝形成对照。
- **TUI 事件循环与历史**:`TuiEvent { Key, Paste, Resize, Draw }`(Resize/Draw 分开为预渲染留钩子);Paste 把 `\r`→`\n`(iTerm2 怪癖);`insert_history.rs` 把终端 scrollback 当历史(转义序列写入 viewport 上方),折行按 URL/混合/纯文本三路 + token 粒度保链接可点;`Tui` 用 `frame_requester`/容量 1 的 `draw_tx`/`pending_history_lines`/`is_zellij`/`_stderr_guard`;`set_panic_hook` 崩溃时先还原终端。

## 动手实验

1. **实验一:追踪一条 stdout 契约** — 阅读 `exec/src/lib.rs` 文件头注释与 `#![deny(clippy::print_stdout)]`,再在 `event_processor_with_human_output.rs` 里找出**唯一**带 `#[allow(clippy::print_stdout)]` 的位置(`print_final_output`)。说明 `should_print_final_message_to_stdout` 与 `should_print_final_message_to_tty` 这两个判定函数为什么要同时检查 stdout 和 stderr 是否为 TTY——管道、终端、混合三种场景各会走哪条分支?
2. **实验二:对比两种处理器** — 同时阅读 `EventProcessorWithHumanOutput` 与 `EventProcessorWithJsonOutput`。对同一个 `ThreadItem::CommandExecution`(完成、exit_code=0),分别写出人类版的 stderr 输出和机器版的一行 JSONL 大致长什么样。再解释机器版为什么需要 `next_item_id: AtomicU64` 和 `raw_to_exec_item_id: HashMap`,而人类版不需要。
3. **实验三:对照"问"与"不问"** — 把 `exec/src/lib.rs` 的 `handle_server_request`(拒绝/取消各类交互请求)和 `tui/src/app_event_sender.rs` 的审批方法(`exec_approval`/`patch_approval`/`request_permissions_response`/`resolve_elicitation`)并排读。列出一个被 exec 拒绝、却被 TUI 包成 `SubmitThreadOp` 应答的请求,并解释为什么 elicitation 在 exec 里是"取消"而非"报错"。
4. **实验四:理解"终端即历史"** — 阅读 `tui/src/insert_history.rs` 的模块注释与 `insert_history_hyperlink_lines_with_mode_and_wrap_policy` 的折行三路逻辑。解释为什么 URL-only 的行要"保持不折断",以及 `HistoryLineWrapPolicy::PreWrap` 与 `Terminal` 的区别。再读 `tui/src/tui.rs` 的 `set_panic_hook`,说明若去掉这个钩子、TUI 在 alt screen 下 panic,用户的终端会发生什么。

> **下一章预告**:十六章走下来,我们把 Codex 从入口到前端、从循环到沙箱、从持久化到观测全部拆开了。最后一章不再引入新代码,而是回头做一次**综合**:测试与可信度如何贯穿这套架构(依赖注入、隔离、断言即安全网),以及把全书的观察提炼成若干条**与语言、框架无关、可迁移**的工程原则——每一条都回指到前面具体的章节作为证据。这是为这次"解构 Codex"收束的一章。
