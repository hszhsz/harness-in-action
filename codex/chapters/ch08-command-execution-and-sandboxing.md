# 第 8 章　命令执行与多平台沙箱：把每条命令关进笼子

`apply_patch` 改的是文件，而 shell 工具要跑的是**任意命令**——`cargo test`、`ls`、`rm -rf`、甚至 `curl | sh`。这是比改文件危险得多的能力：一条命令可以读走你的密钥、删光你的磁盘、把数据偷传出去。一个"会自己跑命令"的 Agent 如果不被约束，就是一颗定时炸弹。

Codex 的回答是**沙箱**：让模型跑的每一条命令都被关进一个"能在工作区里干活、却跑不出去"的笼子。这一章我们拆这套多平台沙箱：Linux 用 Landlock/seccomp（经 bubblewrap），macOS 用 Seatbelt，Windows 用受限令牌；以及统领它们的抽象 `SandboxType`、`SandboxPolicy` 与命令执行流。代码主要在 `sandboxing/` crate 和 `core/src/exec.rs`。

---

## 8.1 沙箱的抽象：`SandboxType` 与 `get_platform_sandbox`

不同操作系统的隔离机制天差地别，但 Codex 把它们收敛成一个统一枚举 `SandboxType`（`sandboxing/src/manager.rs`），四个变体：

```rust
pub enum SandboxType {
    None,
    MacosSeatbelt,
    LinuxSeccomp,
    WindowsRestrictedToken,
}
```

每个变体都有一个 `as_metric_tag()`，分别返回 `"none"`、`"seatbelt"`、`"seccomp"`、`"windows_sandbox"`——这意味着"用了哪种沙箱"会进遥测（第 15 章），团队能统计线上到底有多少命令跑在哪种隔离下。

选哪种沙箱由 `get_platform_sandbox(windows_sandbox_enabled)` 决定，逻辑朴素得近乎一目了然：macOS → `MacosSeatbelt`，Linux → `LinuxSeccomp`，Windows → 只有显式开启 `windows_sandbox_enabled` 才返回 `WindowsRestrictedToken`，否则 `None`。这个"Windows 默认无沙箱"的保守姿态本身就是一种诚实：Codex 不假装自己在所有平台都有同等强度的隔离。

还有一个 `SandboxablePreference` 枚举（`Auto`/`Require`/`Forbid`）表达"这条命令对沙箱的偏好"——是自动判断、必须沙箱、还是禁止沙箱。这给了上层（审批流，第 9 章）一个旋钮：某些命令可以要求"必须在沙箱里跑，否则不跑"。

---

## 8.2 策略的抽象：`SandboxPolicy` 的四档

`SandboxType` 回答"用哪种隔离机制"，`SandboxPolicy`（`protocol/src/protocol.rs`）则回答"允许干到什么程度"。它是个序列化友好的枚举（`#[serde(tag = "type", rename_all = "kebab-case")]`），四档权限**从松到紧**：

- **`DangerFullAccess`**（`danger-full-access`）：毫无限制。注释直言"Use with caution"——名字里带 `Danger` 就是要让你在配置里写下它时心里发怵。
- **`ReadOnly`**：只读，默认连网都不给（`network_access: false`）。
- **`ExternalSandbox`**：进程已经在一个外部沙箱里了，于是允许全盘访问，但尊重外部沙箱给定的网络设置。这是给"Codex 跑在别人的容器里"这种场景留的口子。
- **`WorkspaceWrite`**（`workspace-write`）：在只读基础上，额外允许写**当前工作区**。它有一组精细的旋钮：`writable_roots`（除 cwd 外还能写哪些目录）、`network_access`（默认 `false`）、`exclude_tmpdir_env_var` 和 `exclude_slash_tmp`（是否把 `$TMPDIR` 和 `/tmp` 排除出默认可写集）。

这四档构成了第 1 章"约束只能收紧"原则的具体载体：默认就在偏紧的一档，越往松走名字越吓人。注意**网络默认全部关闭**——即便是 `WorkspaceWrite`，`network_access` 也默认 `false`。对一个会跑命令的 Agent，"默认断网"是防数据外泄的第一道防线。

### `WritableRoot`：可写目录里的只读飞地

最值得玩味的是 `WritableRoot` 结构（同文件）。当某个目录被标记为可写时，它还带一个 `read_only_subpaths` 列表——**"可写根目录下，这些子路径仍保持只读"**。注释解释了为什么：

> "这主要用于确保可写根目录下那些'一旦被改就能提升 agent 权限'的文件夹（如 `.codex`、`.git`，尤其是 `.git/hooks`）不被 agent 修改。"

这是一处教科书级的纵深防御。想象 Agent 有权写你的项目目录（`WorkspaceWrite`），如果它能往 `.git/hooks/pre-commit` 里塞一段脚本，那么下次你一 `git commit`，这段脚本就会以**你的身份、在沙箱外**执行——沙箱就被绕过了。Codex 预见了这条提权路径，于是在"可写根"里挖出 `.git/hooks` 这块**只读飞地**。同理 `.codex` 目录（Codex 自己的配置）也被保护，防止 Agent 改写配置来给自己松绑。**安全不是"给个大权限"，而是"给权限的同时堵死权限被滥用的具体路径"。**

---

## 8.3 Linux：自我重执行成 `codex-linux-sandbox`

还记得第 3 章的 arg0 trick 吗？Linux 沙箱正是它最重要的用武之地。Codex 不去找一个外部沙箱程序，而是**用 `codex-linux-sandbox` 这个名字重新 exec 自己**——常量 `CODEX_LINUX_SANDBOX_ARG0 = "codex-linux-sandbox"`（`sandboxing/src/landlock.rs`）就是那个触发分发的文件名。

要在沙箱里跑一条命令，`create_linux_sandbox_command_args_for_permission_profile` 把权限 profile 序列化成 JSON，拼成一串传给 helper 的 CLI 参数：`--sandbox-policy-cwd`、`--command-cwd`、`--permission-profile <json>`，可选的 `--use-legacy-landlock` 或 `--allow-network-for-proxy`，最后用 `--` 隔开真正要跑的命令。注释说得明白：**"helper 在解析这些参数后执行真正的沙箱化（默认 bubblewrap + seccomp）。"**

这里有两层隔离叠加：

- **bubblewrap（bwrap）**：用 Linux 的 namespace 机制构造一个隔离的文件系统/网络视图，让命令只能看到被允许的目录。`sandboxing/src/bwrap.rs` 里还有 `is_wsl1` 检测和 `WSL1_BWRAP_WARNING`——因为 WSL1 不支持 bubblewrap，Codex 会明确报错而非默默降级。
- **seccomp**：用系统调用过滤进一步限制命令能调哪些 syscall。

注释特别提到，"proxy-only 网络需要 bubblewrap 的隔离网络命名空间"——也就是说，"既要断网、又要允许走代理"这种细粒度需求，必须靠 bwrap 的网络 namespace 才能实现。`--use-legacy-landlock` 则是给旧路径留的兼容开关。

为什么要绕一圈"自我重执行"而不在主进程里直接 sandbox？因为沙箱一旦施加就**不可逆**——你不能给主进程套上 Landlock 后还指望它继续做别的。fork 出一个干净的子进程、对它施加最严的限制、让它去跑那条危险命令，主进程则毫发无损，这是隔离的正确姿势。

---

## 8.4 macOS：Seatbelt 的 `(deny default)`

macOS 上 Codex 用系统自带的 `sandbox-exec`（Seatbelt）。`sandboxing/src/seatbelt.rs` 里有一处防御细节：常量 `MACOS_PATH_TO_SEATBELT_EXECUTABLE = "/usr/bin/sandbox-exec"`——**只认 `/usr/bin` 下的那个**。注释解释："防止攻击者在 PATH 上注入恶意版本的 sandbox-exec。如果 `/usr/bin/sandbox-exec` 都被篡改了，那攻击者已经有 root 了。" 这是"边界不信任、连自己依赖的系统工具都走绝对路径"的体现。

沙箱策略本身用 Apple 的 SBPL（Sandbox Profile Language）写成，编译进二进制：`seatbelt_base_policy.sbpl`（基础）、`seatbelt_network_policy.sbpl`（网络）、`restricted_read_only_platform_defaults.sbpl`（受限只读默认）。打开基础策略文件，第一条非注释规则就是灵魂：

```
; start with closed-by-default
(deny default)
```

**默认拒绝一切**，然后再逐条 `allow` 放开必需的能力——这正是第 1 章 fail-closed 的标准范式。策略里随后才小心翼翼地放开 `process-exec`、`process-fork`、对 `/dev/null` 的写、以及一长串只读的 `sysctl`（`hw.activecpu`、`hw.cputype`、`hw.memsize` 等硬件信息）。注释还坦白这套策略"灵感来自 Chrome 的 sandbox 策略"并附上了 Chromium 源码链接——站在巨人肩膀上，且诚实标注出处。

网络策略单独成文件，配合 `WorkspaceWrite { network_access }` 这类开关动态启停。loopback 代理的端口探测（`proxy_loopback_ports_from_env`）则保证"允许走本地代理"时只放开代理实际监听的那个端口，而非整个 localhost。

---

## 8.5 Windows：受限令牌

Windows 没有 Landlock 也没有 Seatbelt，Codex 走的是**受限令牌（restricted token）**路线，对应 `SandboxType::WindowsRestrictedToken`，实现在体量颇大的 `windows-sandbox-rs/` crate 里。扫一眼它的模块就能感受到 Windows 隔离的复杂度：`token.rs`（令牌操作）、`acl.rs`/`workspace_acl.rs`（访问控制列表）、`wfp.rs`（Windows Filtering Platform，做网络过滤）、`desktop.rs`（隔离桌面）、`deny_read_acl.rs`（拒绝读取的 ACL）、`hide_users.rs` 等。

核心思路是：用一个被剥夺了大部分特权的受限令牌来启动子进程，再配合 ACL 精确控制它能读写哪些路径、用 WFP 控制它能不能联网。这比 Unix 的 namespace 方案琐碎得多——光是"哪些路径该授予读权限"就单独有 `core/src/windows_sandbox_read_grants.rs` 在管。也正因为这套机制更脆弱、更难审计，`get_platform_sandbox` 才让它**默认关闭、需显式开启**。

三个平台、三套机制，但它们在上层被 `SandboxType` 和 `SandboxPolicy` 统一抽象——调用方只需说"我要 `WorkspaceWrite`、不许联网"，由 `SandboxManager`（`manager.rs`）翻译成各平台的具体咒语。这与第 6 章工具系统的"统一契约、多态实现"是同一种架构智慧。

---

## 8.6 执行流：`exec.rs` 的超时与清理

命令真正被跑起来在 `core/src/exec.rs`。这里有几个关键常量定义了"跑命令"的安全边界：

- `DEFAULT_EXEC_COMMAND_TIMEOUT_MS = 10_000`——默认命令超时 10 秒。模型跑的命令不能无限期挂着。
- `EXEC_TIMEOUT_EXIT_CODE = 124`——超时退出码，沿用 GNU `timeout` 的惯例（注释写着 "conventional timeout exit code"）。
- `SIGKILL_CODE = 9`、`TIMEOUT_CODE = 64`——信号与超时编码。
- `IO_DRAIN_TIMEOUT_MS = 2_000`——超时杀掉子进程后，给 IO 管道 2 秒把残留输出排空。注释解释这是因为"kill 直接子进程后，那些管道可能还开着"，需要兜底排干。

超时机制通过 `ExecExpiration` 枚举建模，它能是 `DefaultTimeout`、固定 `Timeout(Duration)`、或 `TimeoutOrCancellation`（超时**或**被取消，二者皆可终止）。`wait_with_outcome` 用 `tokio::select!` 同时等命令完成、等超时、等取消令牌——任一先到都干净收场。注意 `kill_child_process_group`（来自 `codex-utils-pty`）杀的是**整个进程组**而非单个进程：模型跑的命令可能 fork 出一堆子孙进程，只杀父进程会留下孤儿，杀进程组才能斩草除根。

这就接回了第 4 章那条贯穿全局的"取消"红线：从 `run_turn` 的 `CancellationToken` 一路传到这里，最终化作对一整个进程组的 SIGKILL。**"会动手的 Agent 必须随时能被干净地叫停"**，在命令执行这一层兑现为：有超时、能取消、杀进程组、排空 IO。

---

## 本章小结

- 沙箱机制统一抽象为 `SandboxType`：`None`/`MacosSeatbelt`/`LinuxSeccomp`/`WindowsRestrictedToken`，各带 `as_metric_tag()` 进遥测；`get_platform_sandbox` 按平台选择，**Windows 默认无沙箱、需显式开启**。
- 权限程度抽象为 `SandboxPolicy` 四档（从松到紧）：`DangerFullAccess`（名字带 Danger）/`ReadOnly`/`ExternalSandbox`/`WorkspaceWrite`；**网络一律默认关闭**。
- `WritableRoot.read_only_subpaths` 在可写根里挖**只读飞地**保护 `.git/hooks`、`.codex` 等提权路径——纵深防御。
- Linux：经 arg0 trick 自我重执行成 `codex-linux-sandbox`（`CODEX_LINUX_SANDBOX_ARG0`），helper 用 **bubblewrap + seccomp** 双层隔离；WSL1 不支持会明确报错；proxy-only 网络靠 bwrap 网络 namespace。
- macOS：Seatbelt，只认 `/usr/bin/sandbox-exec`（防 PATH 注入）；SBPL 策略以 `(deny default)` 开场（fail-closed），灵感来自 Chrome 并标注出处。
- Windows：受限令牌 + ACL + WFP（`windows-sandbox-rs`），机制琐碎故默认关闭。
- 执行流 `exec.rs`：默认超时 `10_000ms`、超时码 `124`、IO 排空 `2_000ms`；`ExecExpiration` 用 `tokio::select!` 同时等完成/超时/取消；`kill_child_process_group` 杀**整个进程组**斩草除根——兑现"随时能干净叫停"。

## 动手实验

1. **实验一：枚举四档策略** — 阅读 `protocol/src/protocol.rs` 的 `pub enum SandboxPolicy`，把四个变体按"权限从松到紧"排序，并指出每一档的 `network_access` 默认值。思考为什么默认值清一色偏紧。
2. **实验二：理解只读飞地** — 在同一文件找到 `WritableRoot` 与其 `read_only_subpaths` 字段的注释。解释一个攻击者若能写 `.git/hooks/pre-commit`，会如何绕过沙箱；以及 Codex 如何用只读飞地堵死这条路。
3. **实验三：读懂 `(deny default)`** — 打开 `sandboxing/src/seatbelt_base_policy.sbpl`，找到 `(deny default)` 那一行，再往下数出它随后 `allow` 了哪三类能力。用自己的话说明"默认拒绝 + 逐条放开"为什么比"默认允许 + 逐条禁止"安全。
4. **实验四：跟踪超时与清理** — 在 `core/src/exec.rs` 找到 `DEFAULT_EXEC_COMMAND_TIMEOUT_MS`、`EXEC_TIMEOUT_EXIT_CODE`、`IO_DRAIN_TIMEOUT_MS` 三个常量，再看 `ExecExpiration::wait_with_outcome` 的 `tokio::select!`。解释为什么超时后要 `kill_child_process_group`（杀进程组）而非只杀直接子进程。

> **下一章预告**：沙箱是"硬隔离"，但不是所有命令都该一刀切地放进或挡在沙箱外。有些命令（如 `ls`）显然安全，有些（如 `rm -rf /`）显然危险，还有大量命令处于灰色地带需要问用户。下一章我们进入执行策略与审批流：`exec_policy` 如何给命令分级、`BANNED_PREFIX_SUGGESTIONS` 如何拦截危险前缀、`ExecApprovalRequirement` 如何决定"放行 / 问人 / 拒绝"，以及审批结果如何被缓存复用。
