# 第 9 章　执行策略与审批流：放行、问人、还是拒绝

第 8 章的沙箱是"硬隔离"——把命令关进笼子。但隔离不是免费的：在沙箱里跑命令更慢、有些命令在沙箱里根本跑不了（比如要联网装依赖），而且不是所有命令都需要同等戒备。`ls` 显然安全，`rm -rf /` 显然危险，`pip install` 则处在灰色地带——可能没问题，也可能在 `setup.py` 里藏了恶意代码。

于是 Codex 在沙箱之上又叠了一层**决策系统**：对每条命令先做分级，决定它该被**放行、问用户、还是直接拒绝**。这一章拆这套执行策略与审批流：`AskForApproval` 的总策略、`execpolicy` 的规则引擎、`BANNED_PREFIX_SUGGESTIONS` 的危险前缀拦截、`ExecApprovalRequirement` 的三态裁决，以及 `safety.rs` 如何把这些综合成最终决定。代码主要在 `execpolicy/` crate、`core/src/exec_policy.rs`、`core/src/safety.rs`、`core/src/tools/sandboxing.rs`。

---

## 9.1 总策略：`AskForApproval` 的五档

一切的顶层旋钮是 `AskForApproval`（`protocol/src/protocol.rs`），它表达用户对"什么时候该问我"的总体偏好。五个变体，注释里讲得很细：

- **`UnlessTrusted`**（`untrusted`）：最谨慎。只有 `is_safe_command()` 认定的"已知安全且只读"的命令才自动放行，**其他一切都问用户**。
- **`OnFailure`**（已废弃）：所有命令自动放行，但期望它们跑在断网、写受限的沙箱里；命令失败时才升级给用户批准"不带沙箱重跑"。注释建议改用 `OnRequest`（交互）或 `Never`（非交互）。
- **`OnRequest`**（默认）：**由模型决定**何时问用户。这是默认值——把"要不要打扰用户"的判断权部分交给模型。
- **`Granular`**：细粒度。带一个 `GranularApprovalConfig`，对不同类别的审批分别开关——shell 审批（`sandbox_approval`）、execpolicy 规则触发的提示（`rules`）、skill 脚本执行（`skill_approval`）、`request_permissions` 工具（`request_permissions`）、MCP elicitation（`mcp_elicitations`）。注释强调：字段为 `false` 时，该类请求被**自动拒绝**而非展示给用户——这是 fail-closed 的又一处落地：关掉某类审批不等于"全部放行"，而是"全部拒绝"。
- **`Never`**：永不打扰。失败立即返回给模型，绝不升级给用户。这是无头/CI 场景的选择。

注意默认是 `OnRequest`——既不是最松（`Never`）也不是最紧（`UnlessTrusted`），而是"让模型在需要时主动求助"的折衷。这反映了 Codex 对交互式编码场景的判断：完全不问会失控，事事都问会烦死人。

---

## 9.2 规则引擎：`execpolicy` 与三态 `Decision`

`AskForApproval` 是粗粒度的总开关，真正对**具体命令**做判断的是独立的 `execpolicy` crate。它是一个小型的策略 DSL 引擎：用户/系统用 `.rules` 文件（默认 `default.rules`）写下规则，引擎解析（`PolicyParser`）成 `Policy`，对每条命令做 `Evaluation`，产出一个 `Decision`。

`Decision`（`execpolicy/src/decision.rs`）只有三个值，干净利落：

```rust
pub enum Decision {
    Allow,      // 无需进一步批准即可运行
    Prompt,     // 请求用户显式批准；在 approval_policy="never" 下直接拒绝
    Forbidden,  // 直接阻断，不再考虑
}
```

这三态是整个审批系统的原子词汇。注意 `Prompt` 的注释里那句"在 `approval_policy="never"` 下直接拒绝"——它揭示了两层策略如何交互：execpolicy 说"该问用户"，但如果总策略是 `Never`（无人可问），结果就退化成拒绝。`exec_policy.rs` 里的常量 `PROMPT_CONFLICT_REASON = "approval required by policy, but AskForApproval is set to Never"` 正是这种冲突的诊断信息——又一次"错误即文档"。

引擎的规则用 `Rule`/`PrefixRule`/`PrefixPattern`/`PatternToken` 等类型建模（`execpolicy/src/rule.rs`），既能按命令前缀匹配（`git status` 允许、`git push` 提示），也能匹配网络规则（`NetworkRuleProtocol`）。规则可以被**动态追加**：`blocking_append_allow_prefix_rule` 和 `blocking_append_network_rule` 让"用户批准一次后，把这个前缀加进允许规则"成为可能——这就是审批结果缓存的底层机制（见 9.5）。

---

## 9.3 危险前缀拦截：`BANNED_PREFIX_SUGGESTIONS`

execpolicy 有一个"建议追加规则"的能力：当用户批准了某条命令，系统可以提议"以后这类命令都自动放行"。但有些命令前缀**绝不能**被这样一键放行——因为它们是任意代码执行的入口。`core/src/exec_policy.rs` 里的 `BANNED_PREFIX_SUGGESTIONS` 列出了这份黑名单，扫一眼就明白它在防什么：

```rust
static BANNED_PREFIX_SUGGESTIONS: &[&[&str]] = &[
    &["python3"], &["python3", "-c"], &["python"], &["python", "-c"],
    &["bash"], &["bash", "-lc"], &["sh"], &["sh", "-c"], &["sh", "-lc"],
    &["zsh"], &["zsh", "-lc"], &["node"], &["node", "-e"],
    &["perl"], &["perl", "-e"], &["ruby"], &["ruby", "-e"],
    &["php"], &["php", "-r"], &["lua"], &["lua", "-e"],
    &["sudo"], &["env"], &["osascript"],
    &["powershell"], &["powershell", "-Command"], &["pwsh", "-c"],
    // ……（python 的各种别名 py/pyw/pypy、各种 shell 的绝对路径变体等）
];
```

为什么是这些？因为 `python3 -c "..."`、`bash -lc "..."`、`node -e "..."` 这类前缀后面可以跟**任意代码**——如果允许把 `bash` 加进"永久放行"列表，那等于把整个机器拱手让人。`sudo` 更不必说，`env` 能用来设环境变量绕过限制，`osascript` 能在 macOS 上驱动整个系统。这份名单覆盖了几乎所有解释器、shell 及其常见别名和绝对路径变体（`/bin/bash`、`powershell.exe` 等）——**穷举各种写法，不给绕过留缝隙**。

逻辑在 `exec_policy.rs` 约 865 行：当系统准备"建议把某前缀规则加进允许列表"时，先检查这个前缀是否**恰好等于**某个 banned 前缀（长度相等且逐词匹配），是则不生成这条建议。注意它检查的是"精确等于"——`exec_policy_tests.rs` 里专门有两个测试印证这个边界：`...returns_none_for_exact_banned_prefix_rule`（精确命中 banned 就不建议）和 `...allows_non_exact_banned_prefix_rule_match`（非精确匹配则放行）。这意味着 `bash` 这个裸前缀不能被一键放行，但 `bash my-known-script.sh` 这种更具体的规则是允许的——**禁的是"任意 bash"，不是"具体某个 bash 脚本"**。颗粒度拿捏得很准。

---

## 9.4 三态裁决：`ExecApprovalRequirement` 与 `SafetyCheck`

把总策略、规则引擎、沙箱能力综合起来，最终落到两个枚举上。

工具侧的结论是 `ExecApprovalRequirement`（`core/src/tools/sandboxing.rs`），告诉 orchestrator 该怎么处理这次调用：

- **`Skip { bypass_sandbox, proposed_execpolicy_amendment }`**：无需审批。`bypass_sandbox` 表示首次尝试就跳过沙箱（被策略明确放行时）。
- **`NeedsApproval { reason, proposed_execpolicy_amendment }`**：需要审批，带原因。
- **`Forbidden { reason }`**：禁止执行，带原因。

两个非禁止变体都带一个 `proposed_execpolicy_amendment`——"如果用户批准，可以提议把这条规则永久加上"，正是 9.3 里被 banned 名单守卫的那个东西。

而决定走哪个变体的核心函数 `default_exec_approval_requirement` 把 `AskForApproval` 映射成是否需要审批，注释给出了清晰的真值表：

> - `Never`、`OnFailure`：不问
> - `OnRequest`：除非文件系统访问不受限，否则问
> - `Granular`：同 OnRequest，但当 granular sandbox 审批被关时**自动拒绝**
> - `UnlessTrusted`：总是问

补丁侧（第 7 章）则用 `SafetyCheck`（`core/src/safety.rs`）表达同样的三态思想：`AutoApprove` / `AskUser` / `Reject`。`assess_patch_safety` 和命令侧的逻辑共享同一套"放行/问人/拒绝"哲学——这就是为什么第 7 章说"改代码的审批和跑命令的审批走同一套机制"。两条路、一种裁决语言。

冲突的兜底也写成了常量：`REJECT_SANDBOX_APPROVAL_REASON`、`REJECT_RULES_APPROVAL_REASON`——当策略要求审批、但对应的 granular 开关是关的，就用这些字符串拒绝并说明原因。每一次"自动拒绝"都有据可查。

---

## 9.5 审批缓存：批准一次，不再重复打扰

如果每次跑 `cargo test` 都要用户点一次"批准"，体验会崩。所以 Codex 支持**审批缓存**：用户批准某条命令后，相似的命令在本会话内可以不再询问。机制就是 9.2 提到的 `blocking_append_allow_prefix_rule`——把用户批准的前缀**动态追加**进 execpolicy 的允许规则，后续匹配到同前缀的命令直接 `Allow`。

这套缓存与 `proposed_execpolicy_amendment` 配合：审批弹窗给用户的不只是"批准这一次"，还可以是"以后这类都批准"（生成一条 amendment）。但正因为有了这个便利，9.3 的 banned 名单才如此重要——它确保用户不会被诱导着把 `bash`、`sudo` 这种"万能钥匙"前缀加进永久白名单。**便利与安全在这里精确制衡：能缓存批准以减少打扰，但绝不允许缓存那些等于交出整台机器的前缀。**

注意所有这些规则状态用 `arc_swap::ArcSwap` 持有（`exec_policy.rs` 顶部的 `use arc_swap::ArcSwap`）——这是一种支持"读多写少、无锁热替换"的并发原语。审批规则在会话中被频繁读取（每条命令都要查）、偶尔写入（用户批准时追加），`ArcSwap` 让读路径无需加锁，写入则原子地换上新的规则集。这是又一处"为高频路径选对数据结构"的工程考量。

---

## 9.6 两层防御如何协作

把第 8、9 两章合起来看，Codex 对"跑命令"这件事构筑了**两层正交的防御**：

1. **决策层（本章）**：命令该不该跑、要不要问用户——`AskForApproval` + `execpolicy` + `SafetyCheck`。
2. **隔离层（第 8 章）**：就算决定跑，也把它关进沙箱限制能干什么——`SandboxType` + `SandboxPolicy`。

两层是**叠加而非互斥**的：一条命令可能被决策层放行，但仍然在隔离层被关进 `WorkspaceWrite` 沙箱里跑；也可能被决策层判为"需审批"，用户批准后选择"不带沙箱跑"（`bypass_sandbox`）。`OnFailure` 策略的设计就利用了这种协作——先在沙箱里乐观地跑，失败了再升级问用户"要不要脱离沙箱重试"。

这种"决策 + 隔离"的纵深，正是第 1 章"让会动手的 Agent 值得信任"的完整答案：信任不来自"相信模型不会干坏事"，而来自"即使模型想干坏事，每条命令也先过决策闸门、再被隔离笼子约束"。**两道独立的防线，任何一道都不假设另一道存在。**

---

## 本章小结

- 总策略 `AskForApproval` 五档：`UnlessTrusted`（只放行已知安全只读）/`OnFailure`（废弃）/`OnRequest`（默认，模型决定何时问）/`Granular`（分类开关，关闭=自动拒绝)/`Never`（永不问）。
- 规则引擎 `execpolicy` 用 `.rules` DSL，对命令产出三态 `Decision`：`Allow`/`Prompt`/`Forbidden`；`Prompt` 在 `Never` 总策略下退化为拒绝（`PROMPT_CONFLICT_REASON`）。
- `BANNED_PREFIX_SUGGESTIONS` 是危险前缀黑名单：`python -c`/`bash -lc`/`node -e`/`sudo`/`env`/`osascript` 等任意代码执行入口，**穷举各种别名与绝对路径变体**；它们**不可被一键加进永久放行**（精确匹配才禁，更具体的脚本规则仍允许）。
- 工具侧裁决 `ExecApprovalRequirement`：`Skip`（含 `bypass_sandbox`）/`NeedsApproval`/`Forbidden`，携带 `proposed_execpolicy_amendment`；`default_exec_approval_requirement` 把 `AskForApproval` 映射成是否审批。
- 补丁侧 `SafetyCheck`（`AutoApprove`/`AskUser`/`Reject`）与命令侧共享同一"放行/问人/拒绝"哲学。
- 审批缓存靠 `blocking_append_allow_prefix_rule` 动态追加允许前缀，用 `ArcSwap` 无锁热替换；便利与安全由 banned 名单制衡。
- 决策层（本章）与隔离层（第 8 章）是**叠加的两层正交防御**——任一道都不假设另一道存在。

## 动手实验

1. **实验一：辨析五档总策略** — 阅读 `protocol/src/protocol.rs` 的 `AskForApproval` 与 `GranularApprovalConfig`。为四个真实场景（本地交互编码、CI 无人值守、只想跑只读命令、想精细控制 MCP 审批）各选一档最合适的策略并说明理由。
2. **实验二：读懂危险前缀名单** — 打开 `core/src/exec_policy.rs` 的 `BANNED_PREFIX_SUGGESTIONS`，挑出 5 个条目，分别解释"为什么把这个前缀加进永久放行等于交出机器"。再找出 `python` 在名单里出现的所有别名变体。
3. **实验三：跟踪三态裁决** — 在 `core/src/tools/sandboxing.rs` 找到 `ExecApprovalRequirement` 三个变体与 `default_exec_approval_requirement` 的真值表注释。对照 `execpolicy/src/decision.rs` 的 `Decision`，画出"总策略 + 规则裁决 → 最终要不要问用户"的决策表。
4. **实验四：验证 banned 名单的精确匹配** — 阅读 `core/src/exec_policy_tests.rs` 里 `derive_requested_execpolicy_amendment_returns_none_for_exact_banned_prefix_rule` 和 `...allows_non_exact_banned_prefix_rule_match` 两个测试。解释为什么裸 `bash` 不能被建议放行，而 `bash my-script.sh` 这样更具体的规则可以。

> **下一章预告**：审批策略、沙箱策略、模型选择、重试上限——这些"行为旋钮"从哪来？它们如何在命令行参数、环境变量、配置文件、托管策略之间合并，又如何保证"合并只会让约束更紧、不会让它更松"？下一章我们进入配置系统：多层加载与合并、`ConfigBuilder` 的装配、以及"约束只能收紧"这条贯穿全书的铁律在配置层的实现。
