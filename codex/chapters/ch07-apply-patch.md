# 第 7 章　`apply_patch`：让模型安全地改你的代码

在所有工具里，`apply_patch` 是编码 Agent 的灵魂。第 6 章我们看到它只是 `handlers/` 目录下众多工具之一，走的是和 shell、MCP 工具完全相同的调度链。但它承载的责任独一无二：**它是模型真正"动手改你代码"的那把手术刀。** 一个 chat 模型能告诉你"把这行改成那样"，而一个编码 Agent 能把改动直接落到磁盘——这之间的全部工程，就藏在 `apply_patch` 里。

这一章我们专门拆它：补丁格式长什么样、如何被解析、如何用"模糊匹配"把模型生成的不那么精确的补丁应用到真实文件、改动如何被记进 `TurnDiffTracker`、以及 Codex 用哪些手段保证"改代码"这件高危操作的安全。代码主要分布在独立的 `apply-patch` crate（`apply-patch/src/`）和 `core` 侧的 `core/src/apply_patch.rs`、`core/src/turn_diff_tracker.rs`、`core/src/tools/handlers/apply_patch.rs`。

---

## 7.1 补丁格式：一种为模型设计的 diff

Codex 没有让模型直接产出 `git diff` 那种带 `@@ -12,7 +12,8 @@` 行号的 unified diff——因为模型数行号很容易数错。它定义了一套**自有的、对模型更友好的补丁格式**，其官方 Lark 语法就写在 `apply-patch/src/parser.rs` 的文件头注释里：

```
start: begin_patch environment_id? hunk+ end_patch
begin_patch: "*** Begin Patch" LF
end_patch: "*** End Patch" LF?

hunk: add_hunk | delete_hunk | update_hunk
add_hunk: "*** Add File: " filename LF add_line+
delete_hunk: "*** Delete File: " filename LF
update_hunk: "*** Update File: " filename LF change_move? change?

change: (change_context | change_line)+ eof_line?
change_context: ("@@" | "@@ " /(.+)/) LF
change_line: ("+" | "-" | " ") /(.+)/ LF
eof_line: "*** End of File" LF
```

读懂这套格式的关键在于它**刻意回避了行号**。一个 update hunk 不说"在第 42 行改"，而是给出一段**上下文**（前缀空格的行）夹着要删的行（`-` 前缀）和要加的行（`+` 前缀），靠上下文去文件里"找位置"。`@@` 是可选的上下文锚点（比如函数签名行），帮助定位到文件的大致区域。这种设计把"定位"的负担从"精确数行"转成了"匹配文本"——后者正是语言模型擅长的。

这些标记字面量在 `parser.rs` 里都是常量：`BEGIN_PATCH_MARKER = "*** Begin Patch"`、`UPDATE_FILE_MARKER = "*** Update File: "`、`MOVE_TO_MARKER = "*** Move to: "`、`EOF_MARKER = "*** End of File"` 等。解析产物是一个 `Hunk` 枚举，三个变体对应三种操作：`AddFile { path, contents }`、`DeleteFile { path }`、`UpdateFile { path, move_path, chunks }`——注意 update 带一个可选的 `move_path`，这意味着"改内容"和"重命名/移动"可以在一个 hunk 里一起完成。

格式是怎么交付给模型的？在 `tools/handlers/apply_patch_spec.rs` 里，`create_apply_patch_freeform_tool` 把上面那份 `.lark` 语法（`include_str!("apply_patch.lark")`）打包成一个 **Freeform 工具**——`ToolSpec::Freeform`，格式类型是 `"grammar"`、语法是 `"lark"`。工具描述里有一句很关键的叮嘱："This is a FREEFORM tool, so do not wrap the patch in JSON." 也就是说，模型不是用 JSON 参数调它，而是直接吐出符合 Lark 语法的纯文本补丁，服务端用这套语法约束模型的输出。这是第 5 章"结构化输出严格校验"思想在工具层的延伸——**用语法本身约束模型，让它生成的补丁天然合法。**

---

## 7.2 宽松解析：向 gpt-4.1 的妥协

源码里有一个诚实得可爱的常量：`parser.rs` 的 `PARSE_IN_STRICT_MODE: bool = false`。它的注释解释了为什么：

> "目前唯一明确需要宽松解析的 OpenAI 模型是 gpt-4.1。虽然我们可以要求所有调用方传入一个 strictness 参数，但把它穿过所有调用点很麻烦，所以我们退而允许对所有模型都用宽松解析。"

这是工程现实的一面镜子：理论上应该严格，但因为某个具体模型的输出习惯，加上"把参数穿过所有调用点"的成本，团队选择了务实的折衷。注释甚至直接点名是 `ParseMode::Lenient` 在为 gpt-4.1 开后门。这种"把妥协写进注释、连原因带责任人一起记下"的做法，正是第 2 章 `AGENTS.md` 工程文化的微观体现——**妥协可以有，但必须可追溯。**

---

## 7.3 模糊匹配：`seek_sequence` 的四级降级

补丁格式靠"匹配上下文"定位，那问题来了：如果模型生成的上下文行和文件里的真实内容有细微出入（多一个空格、用了花引号、Unicode 破折号），还能不能找到位置？这正是 `apply-patch/src/seek_sequence.rs` 要解决的核心难题，也是整个 apply_patch 最精妙的一段代码。

`seek_sequence(lines, pattern, start, eof)` 的任务是：在 `lines` 里从 `start` 之后找到 `pattern` 这段连续行。它的注释开宗明义——**"以递减的严格度依次尝试匹配"**。具体是四级降级：

1. **精确匹配**：`lines[i..i+len] == *pattern`，一字不差。
2. **忽略行尾空白**（rstrip）：逐行比较 `trim_end()` 后是否相等——容忍模型在行尾多/少了空格。
3. **忽略首尾空白**（trim）：逐行比较 `trim()` 后是否相等——容忍缩进层面的细微差异。
4. **Unicode 归一化匹配**：这是最宽松的一关。`normalise` 函数把各种"花哨"字符映射回 ASCII：六七种 Unicode 破折号（`\u{2010}`～`\u{2015}`、`\u{2212}`）全部归一成 `-`，花式单/双引号归一成 `'`/`"`，各种宽度的空格（不间断空格、全角空格 `\u{3000}` 等）归一成普通空格。注释说这"模仿了 `git apply` 的模糊行为"。

为什么要做到这种程度？因为模型在生成代码时，可能会把源码里的 typographic 标点"美化"，或者粘贴时引入了不可见的特殊空格。如果 apply_patch 因为一个肉眼看不见的破折号差异就拒绝打补丁，用户体验会非常糟。这四级降级让补丁"尽可能能打上"，同时又是**从严到松、逐级放宽**的——精确匹配优先，只有在更严的层级都失败时才放宽。

还有两处防御式编程值得记下，注释里都标了来历：

- **空 pattern → `Some(start)`**：空模式是 no-op 匹配，直接返回起点。
- **`pattern.len() > lines.len()` → `None`**：模式比文件还长，不可能匹配，提前返回——注释特别注明这是为了"避免 2025-04-12 之前出现过的越界 panic"。一个真实的 bug 修复，被永久地刻进了注释。

`eof` 参数也有讲究：当补丁意在匹配文件结尾时，先从 `lines.len() - pattern.len()` 处（即文件末尾）开始找，找不到再回退到从 `start` 搜索——让"加在文件末尾"这类意图能精确命中结尾。

---

## 7.4 安全闸门：`assess_patch_safety` 的三种裁决

补丁能解析、能定位，不代表能直接落盘。**改文件是高危操作，必须先过安全闸门。** 这一步在 `core/src/apply_patch.rs` 的 `apply_patch` 函数里，它调用 `safety::assess_patch_safety`，传入 action、审批策略（`approval_policy`）、权限 profile、文件系统沙箱策略、cwd、Windows 沙箱级别——综合判定后返回一个 `SafetyCheck`，三种裁决对应三条路：

- **`AutoApprove`**：根据用户的沙箱策略，这个补丁看起来本就该被允许（或用户此前已显式批准）。返回 `DelegateToRuntime`，`auto_approved` 取决于是否是用户显式批的，`exec_approval_requirement` 设为 `Skip`——**放行，交给运行时落盘**。
- **`AskUser`**：拿不准，需要问用户。同样 `DelegateToRuntime`，但 `exec_approval_requirement` 设为 `NeedsApproval`。注释说明，审批提示（包括缓存的审批结果）被委托给工具运行时处理，"与 shell/unified_exec 的审批一样由 orchestrator 驱动"——**又一次"一视同仁"：改代码的审批和跑命令的审批走同一套机制**（详见第 9 章）。
- **`Reject`**：直接拒绝。返回 `Output(Err(...))`，错误信息是 `"patch rejected: {reason}"`，回灌给模型——模型会收到"你的补丁被拒了，原因是……"的明确反馈，而不是石沉大海。

这三条路精确对应第 1 章说的"fail-closed"姿态：默认不信任，能自动放行的才放行，拿不准就问人，越界就拒绝。注意中间的 `InternalApplyPatchInvocation` 枚举把"已被处理（Output）"和"委托给运行时执行（DelegateToRuntime）"两种结果分得清清楚楚——前者用于补丁内嵌在 shell 调用里、被程序化处理且用户已显式批准的情况，后者才是通过环境文件系统真正落盘。

落盘后，`convert_apply_patch_to_protocol` 把 `ApplyPatchFileChange`（内部表示）翻译成协议层的 `FileChange`（`Add`/`Delete`/`Update{unified_diff, move_path}`），供 UI 渲染和事件广播。

---

## 7.5 记账：`TurnDiffTracker` 如何攒出本轮总 diff

第 4 章我们见过 `run_turn` 里那个 `Arc<Mutex<TurnDiffTracker>>`，并记下一句关键注释："从 codex.rs 的视角看它的生命周期是一个包含多个 turn 的 Task，但从用户视角看它是一个单独的 turn。" 现在我们打开 `core/src/turn_diff_tracker.rs` 看它到底攒了什么。

`TurnDiffTracker` 的职责，doc 注释一句话讲清：**"追踪当前 turn 由已提交的 apply_patch 改动累积出的净文本 diff，而无需重新读取工作区文件系统。"** 关键词是"无需重读文件系统"——它在内存里维护两份快照：

- `baseline_by_path`：每个文件在本轮**开始前**的内容（基线）。
- `current_by_path`：每个文件**当前**的内容。

每个被追踪的文件用 `TrackedPath { environment_id, path }` 标识——注意带了 `environment_id`，因为 Codex 支持多环境（多个工作区），同名路径在不同环境里是不同文件。内容用 `TrackedContent { content, revision }` 表示，`revision` 是单调递增的版本号，配合 `DiffCacheKey` 缓存"某两个版本之间渲染出的 diff"，避免重复计算。

为什么要这样设计？因为用户想看的是"这一轮 Agent 总共把我的代码改成了什么样"——是**净改动**，而不是中间每一步的流水账。如果模型先把 A 改成 B、又把 B 改回 A，净 diff 应该是空的。靠 baseline 和 current 两份快照做差，就能自然得出净效果。而"不重读文件系统"则保证了即便文件在外部被并发改动，tracker 报告的也是"Codex 自己这一轮干了什么"，归因清晰。

还有一个性能细节：`DIFF_TIMEOUT = Duration::from_millis(100)`。注释说"正常编辑都能在 100ms 内完成；遇到病态输入则回退到一个粗粒度、内容精确的 diff，而不拖住工具完成"。这是典型的"给慢路径设上限、保证主流程不卡死"——和第 5 章流空闲超时、第 6 章保守默认是同一种工程直觉。

---

## 7.6 流式 diff：边生成边渲染

还记得第 6 章的 `create_diff_consumer` 和第 5 章的 `ToolCallInputDelta` 吗？`apply_patch` 正是它们最典型的消费者。在 `tools/handlers/apply_patch.rs` 里，`ApplyPatchArgumentDiffConsumer` 实现了 `ToolArgumentDiffConsumer` trait：当模型还在**流式吐出**补丁文本时，它用一个 `StreamingPatchParser` 增量解析已到达的片段，每解析出新的 hunk 就转成 `PatchApplyUpdatedEvent` 事件发给 UI——于是用户能看到补丁"边写边显示"，而不必等整个补丁生成完。

这里有一个节流细节：`APPLY_PATCH_ARGUMENT_DIFF_BUFFER_INTERVAL = Duration::from_millis(500)`。`push_delta` 里判断，如果距上次发送不足 500ms，就把事件暂存到 `pending` 而不立即发——避免高频增量把 UI 刷爆。等下一次满足间隔、或 `finish` 收尾时再把 pending 发出去。同时这个消费者受一个特性开关 `Feature::ApplyPatchStreamingEvents` 控制，没开就直接返回 `None`——流式渲染是可灰度的增量能力。

把这条线串起来：模型流式生成补丁 → `ToolCallInputDelta` 增量 → `ApplyPatchArgumentDiffConsumer` 解析并节流 → `PatchApplyUpdatedEvent` → UI 实时渲染 diff。这就是为什么你在 Codex 里能看到代码改动"像打字一样"逐步浮现。

---

## 本章小结

- `apply_patch` 用**自有补丁格式**（Lark 语法，定义在 `parser.rs` 头注释）而非 unified diff，刻意**回避行号**，靠上下文文本定位——把负担从"数行"转成"匹配文本"，契合模型能力。
- 格式经 `create_apply_patch_freeform_tool` 打包成 `ToolSpec::Freeform`（grammar/lark），描述明确"不要包 JSON"——用语法约束模型输出，天然合法。
- 解析默认**宽松**（`PARSE_IN_STRICT_MODE = false`），注释坦白是为 gpt-4.1 妥协——妥协可追溯。
- `seek_sequence` 是核心：**四级降级匹配**（精确 → 忽略行尾空白 → 忽略首尾空白 → Unicode 归一化），从严到松；防御式处理空 pattern 与超长 pattern（后者注释记着一次真实的越界 panic 修复）。
- 安全闸门 `assess_patch_safety` 给出三种裁决：`AutoApprove`（放行）/`AskUser`（问人，与 shell 审批同走 orchestrator）/`Reject`（拒绝并回灌 `"patch rejected: ..."`）——fail-closed。
- `TurnDiffTracker` 用 `baseline`/`current` 两份内存快照算**本轮净 diff**，不重读文件系统；带 `environment_id` 支持多环境，`revision` + `DiffCacheKey` 缓存渲染；`DIFF_TIMEOUT = 100ms` 给慢路径兜底。
- 流式 diff：`ApplyPatchArgumentDiffConsumer` 消费 `ToolCallInputDelta`，用 `StreamingPatchParser` 增量解析、500ms 节流、发 `PatchApplyUpdatedEvent`，受 `Feature::ApplyPatchStreamingEvents` 控制——代码改动边生成边渲染。

## 动手实验

1. **实验一：读懂补丁语法** — 打开 `apply-patch/src/parser.rs` 顶部的 Lark 语法注释，手写一个把某文件第二行 `foo` 改成 `bar` 的补丁（用 `*** Begin Patch` / `*** Update File:` / `@@` / `-`/`+` 标记）。对照 `Hunk` 枚举的三个变体，说明你的补丁会被解析成哪个。
2. **实验二：跟踪四级降级** — 阅读 `apply-patch/src/seek_sequence.rs` 的 `seek_sequence`，列出它依次尝试的四种匹配策略。再看 `normalise` 函数，写出三个"模型可能生成、但与源码字节不同却应匹配成功"的字符例子（如花引号、Unicode 破折号、全角空格）。
3. **实验三：辨析三种安全裁决** — 在 `core/src/apply_patch.rs` 找到 `assess_patch_safety` 的三个分支 `AutoApprove`/`AskUser`/`Reject`，分别说明它们返回的 `InternalApplyPatchInvocation` 是什么、`exec_approval_requirement` 取什么值，以及被 Reject 时模型会收到怎样的反馈字符串。
4. **实验四：理解净 diff** — 阅读 `core/src/turn_diff_tracker.rs` 的 `TurnDiffTracker` 字段 `baseline_by_path` 与 `current_by_path`。设想模型在一轮里先把 A 改成 B、又把 B 改回 A，解释为什么最终渲染出的净 diff 是空的，以及为什么 tracker 强调"无需重读文件系统"。

> **下一章预告**：`apply_patch` 改的是文件，而 shell 工具要跑的是**任意命令**——这是比改文件更危险的能力。下一章我们进入命令执行与多平台沙箱：Codex 如何在 Linux 用 Landlock/seccomp、在 macOS 用 Seatbelt、在 Windows 用受限令牌，把模型跑的每一条命令关进一个"能干活但跑不出去"的笼子。
