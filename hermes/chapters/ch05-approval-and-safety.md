# 第 5 章　审批与命令安全

给 Agent 一个终端，等于给它一把双刃剑。它能帮你装依赖、跑测试、整理文件——也能（在模型幻觉、被恶意提示注入、或单纯犯错时）执行 `rm -rf /`、`mkfs`、把 curl 下来的脚本直接管道给 sh。一个负责任的生产级 Agent，必须在"让它能干活"和"别让它把机器干废"之间划出清晰、可靠、不可被轻易绕过的边界。

这一章我们解构 Hermes 的安全核心：`tools/approval.py` 的两级命令阻断体系，以及 `agent/file_safety.py` 的文件写入防线。你会看到 Hermes 如何把"什么命令绝对不许跑"和"什么命令需要先问一句"区分对待，又如何把最关键的几个控制面文件保护得密不透风。

## 5.1 两个层级的根本区别：硬线 vs 危险

Hermes 把危险命令分成两个截然不同的层级，理解这个划分是理解整套安全体系的钥匙。

第一层是 **`HARDLINE_PATTERNS`**（约第 255 行，共 12 条）——**无条件阻断，连 `--yolo` 模式都绕不过去**。这是绝对红线，无论用户多么想"放飞自我"，这些命令都不会被执行。

第二层是 **`DANGEROUS_PATTERNS`**（约第 373 行，约 47 条）——**需要审批，但 YOLO 模式可以放行**。这些命令有风险，正常情况下要先问过用户，但如果用户明确进入了"我自己负责"的 YOLO 模式，它们可以通过。

这个二分法的工程价值在于：它承认"危险"是有等级的。`rm -rf` 一个项目目录是危险但可能合理（用户确实想清理），而 `rm -rf /` 或 `mkfs` 几乎不可能是合理意图——后者无论如何都该被挡死。把"绝对不行"和"需要确认"分开，既不会因为一刀切的严格而让 Agent 寸步难行，也不会因为统一的宽松而留下致命缺口。

两级名单都用 `_RE_FLAGS = re.IGNORECASE | re.DOTALL` 预编译成正则（分别是 `HARDLINE_PATTERNS_COMPILED` 与 `DANGEROUS_PATTERNS_COMPILED`），大小写不敏感、`.` 能匹配换行——因为攻击者可能用大小写混写或换行来规避朴素的字符串匹配。

## 5.2 硬线名单：12 条绝对红线

`HARDLINE_PATTERNS` 的 12 条规则，每一条都对应一类"几乎不可能是善意、一旦执行就是灾难"的操作。按源码，它们覆盖：

- 对根文件系统的递归删除（`rm -rf /`）、对系统目录或 `~` / `$HOME` 的递归删除；
- `mkfs`（格式化文件系统）；
- `dd of=/dev/sd…`（向裸块设备写入，会直接破坏磁盘）；
- 向裸块设备重定向输出；
- fork 炸弹（经典的 `:(){ :|:& };:`）；
- `kill -1`（向所有进程发信号）；
- 系统级关机/重启操作：`shutdown` / `reboot` / `halt` / `poweroff`、`init 0/6`、`systemctl poweroff/reboot`、`telinit 0/6`。

这些模式通过一个叫 `_CMDPOS` 的正则片段来**锚定在命令起始位置**——这是个重要细节。如果不做位置锚定，像 `echo "don't run rm -rf /"` 这种把危险字符串放在无害上下文里的命令也会被误杀。锚定到命令位置，意味着只有当这些危险词真的处在"要被执行的命令"位置时才触发，减少误报。检测由 `detect_hardline_command()`（约第 326 行）执行，命中后走 `_hardline_block_result`（约第 339 行）返回阻断结果。源码注释里提到这份硬线名单参考了 "Mercury Agent's permission-hardened blocklist"——也就是说，这套红线是站在前人经验的肩膀上沉淀出来的，而不是凭空臆想。

## 5.3 危险名单：47 条需审批的操作

`DANGEROUS_PATTERNS` 的约 47 条规则覆盖面更广，是日常操作中"有风险但可能合理"的灰色地带。按源码，典型的有：

- 递归删除（`rm -r`，非根目录）；
- 权限放纵：`chmod 777` / `chmod 666`、`chown -R root`；
- 危险的 SQL：`DROP`、不带 `WHERE` 的 `DELETE`（正则 `\bDELETE\s+FROM\b(?![^\n]*\bWHERE\b)` 精确地表达了"DELETE FROM 后面如果没有 WHERE 就危险"这个语义）、`TRUNCATE`；
- 服务操控：`systemctl stop/restart/disable/mask`；
- 进程强杀：`kill -9 -1`、`pkill -9`、`killall -KILL` / `killall -r`；
- 远程脚本直执：`curl … | sh`、`wget … | sh`（先下载再无审查地执行，是最常见的攻击载体之一）；
- 向敏感文件写入：用 `tee` 或重定向写 `.env` / `config.yaml`；
- 间接删除：`xargs … rm`、`find -exec rm`；
- shell `-c` 调用等。

那条不带 `WHERE` 的 `DELETE` 正则特别值得欣赏。它没有粗暴地把所有 `DELETE` 都标危险（那样会让正常的带条件删除也要审批，烦死用户），而是用一个否定前瞻 `(?![^\n]*\bWHERE\b)` 精准地只盯住"会删光整张表"的那种 DELETE。**安全规则的精度，决定了它会不会因为误报太多而被用户关掉**——一个总在误报的安全机制，最终会被绕过或禁用，等于没有。检测由 `detect_dangerous_command(command)`（约第 594 行）执行，返回 `(is_dangerous, pattern_key, description)` 三元组，其中 `description` 会告诉用户"这条命令为什么危险"。

## 5.4 一道独立的闸：sudo 密码爆破防护

除了两级名单，`approval.py` 还有一道针对特定攻击向量的专门防线。`_SUDO_STDIN_RE` 配合 `_check_sudo_stdin_guard()`（约第 307 行）**无条件阻断** `sudo -S` 形式的命令——除非环境里设置了 `SUDO_PASSWORD`。

为什么单独防这个？`sudo -S` 表示"从标准输入读取 sudo 密码"。如果允许 Agent 自由使用这个形式，它就可能被诱导去尝试各种密码（密码爆破），或者把环境里偷来的密码喂进去提权。Hermes 的策略是：除非用户已经通过 `SUDO_PASSWORD` 明确提供了密码（表示"我授权你用 sudo"），否则 `sudo -S` 一律拦死。这是一个很精准的针对性防护——它不阻止所有 sudo（那太严了），只阻止"从 stdin 喂密码"这一种高危用法。

## 5.5 YOLO 模式：在导入时就冻结的开关

"YOLO 模式"是 Hermes 给高级用户的逃生舱：在受信任的环境里，用户可以接受"不再逐条审批危险命令"。但这个开关的实现方式透露了 Hermes 的谨慎。

全局 YOLO 在**模块导入时就被冻结**：`_YOLO_MODE_FROZEN = is_truthy_value(os.getenv("HERMES_YOLO_MODE", ""))`（约第 29 行）。注意"冻结"二字——它在导入那一刻读取环境变量并定下来，之后运行期间再改环境变量也不会影响它。这防止了运行中途环境被篡改而悄悄打开 YOLO 的风险。

除了全局冻结开关，还有更细粒度的**会话级 YOLO**：用一个 `_session_yolo: set[str]` 集合管理，配套 `enable_session_yolo` / `disable_session_yolo` / `is_session_yolo_enabled` / `is_current_session_yolo_enabled`。这让"放飞"可以被限定在单个会话内，而不是全局生效——某个你信任的会话可以开 YOLO，其他会话依然受保护。

但请始终记住前面 5.1 的铁律：**YOLO 只能放行 `DANGEROUS_PATTERNS`，对 `HARDLINE_PATTERNS` 完全无效**。哪怕你开了全局 YOLO、又开了会话 YOLO，`rm -rf /` 和 `mkfs` 依然会被挡死。逃生舱再大，也大不过绝对红线。

## 5.6 审批状态与审批模式

当一条命令落入"需要审批"的区间，Hermes 需要记住"用户批没批过"。`is_approved(session_key, pattern_key)`（约第 758 行）检查两层：`_permanent_approved`（永久批准）和 `_session_approved`（会话内批准），还带一个 `_approval_key_aliases` 的遗留键回退（兼容老版本的批准记录格式）。配套有 `approve_permanent`、`approve_session`，以及 `load_permanent` / `load_permanent_allowlist`（从配置里读 `command_allowlist`）、`clear_session_approvals`。这套设计让用户可以"批准一次，本会话不再问"，也可以把常用命令写进永久白名单"永远不问"。

审批的**模式**由 `_get_approval_mode()`（约第 963 行）返回 `manual` / `smart` / `off`，默认 `"manual"`（最安全的逐条人工审批）。这里有一个有趣的兼容性处理：`_normalize_approval_mode()` 专门处理 YAML 1.1 的一个著名坑——`off` 这个词会被 YAML 解析成布尔 `False`。所以当配置里写 `approval_mode: off`，解析出来是 `False`，归一化函数要把这个 `False` 翻译回 `"off"`。这种"为已知的配置格式陷阱写补偿代码"的细节，是成熟项目和玩具项目的分水岭。

审批超时由 `_get_approval_timeout()` 控制，默认 **60 秒**——用户在 60 秒内不响应，审批请求按超时处理（而不是无限期挂起）。针对 cron 定时任务还有独立的 `_get_cron_approval_mode()`，默认 `"deny"`：**无人值守的定时任务遇到危险命令，默认拒绝而非批准**。这又是一处 fail-closed——定时任务在后台跑、没人能即时确认，那就保守地拒绝，绝不冒险放行。

在网关（多平台）场景下，审批还要跨进程/跨平台流转，由 `prompt_dangerous_approval`、`submit_pending`、`resolve_gateway_approval`、`_await_gateway_decision`、`has_blocking_approval` 等一组函数处理，并通过 `_fire_approval_hook(hook_name, **kwargs)` 触发生命周期钩子，让插件能介入审批流程。

## 5.7 文件写入防线：`agent/file_safety.py`

命令安全之外，文件写入是另一条必须守住的边界。`agent/file_safety.py` 的 `is_write_denied(path)`（约第 96 行）是核心：它先把路径 realpath 解析（解开符号链接，防止用软链绕过），再对照 `build_write_denied_paths(home)` / `build_write_denied_prefixes(home)` 做检查。

它显式保护的"控制面文件"是 `("auth.json", "config.yaml", "webhook_subscriptions.json")`——这三个文件分别掌管鉴权凭证、全局配置、webhook 订阅。**它们在"当前激活 profile 的 home"和"root"两个位置都被保护**。为什么这三个文件如此关键？因为如果 Agent 能改 `auth.json`，它就能改写自己的凭证；能改 `config.yaml`，就能改写自己的安全设置（比如把审批模式关掉）；能改 `webhook_subscriptions.json`，就能给自己加后门入口。换句话说，**这三个文件是"能修改安全机制本身的文件"，必须比普通数据保护得更严**——否则前面所有的命令审批都可能被一次配置改写架空。此外，`mcp-tokens/` 和 `pairing/` 两个目录也被整体保护（分别存 MCP 令牌和设备配对信息）。

`is_write_denied` 还支持一个强约束：`HERMES_WRITE_SAFE_ROOT`（通过 `get_safe_write_root()` 读取）。一旦设置，**任何在这个根目录之外的写入都被拒绝**——把 Agent 的写权限整个圈进一个沙盒目录，是部署到不可信环境时的终极保险。

读取侧也有防线：`get_read_block_error(path)`（约第 165 行）拦截对一组项目级环境文件的读取，`_BLOCKED_PROJECT_ENV_BASENAMES = {".env", ".env.local", ".env.development", ".env.production", ".env.test", ".env.staging", ".envrc"}`（约第 154 行）。这些文件里通常塞满 API key 和密码，阻止 Agent 读它们能防止凭证经由对话泄露出去。

最后，一组"跨区域"分类器处理路径逃逸的灰色场景。`PROFILE_SCOPED_AREAS = ("skills", "plugins", "cron", "memories")` 定义了 profile 私有区域，`classify_cross_profile_target` / `get_cross_profile_warning`、`classify_sandbox_mirror_target`、`classify_container_mirror_target` 等函数都通过 `Path(...).expanduser().resolve()` 解析后用 `.relative_to(...)` 来判断目标是否逃逸出预期区域。**所有路径判断都先 resolve 再比较**，这是防住"用 `../../` 或符号链接逃逸"的标准姿势——直接比字符串会被相对路径骗过，resolve 成绝对真实路径再比就骗不了了。

## 本章小结

- 危险命令分两级：`HARDLINE_PATTERNS`（12 条，无条件阻断，YOLO 也绕不过）与 `DANGEROUS_PATTERNS`（约 47 条，需审批但 YOLO 可放行）；二分法承认"危险有等级"，兼顾可用与安全。
- 两级名单均以 `re.IGNORECASE | re.DOTALL` 预编译，硬线模式用 `_CMDPOS` 锚定命令起始位置以减少误报；硬线名单参考了 Mercury Agent 的加固经验。
- 危险名单的精度是关键：不带 `WHERE` 的 `DELETE` 用否定前瞻 `(?![^\n]*\bWHERE\b)` 精准命中"会删全表"的语句，避免误报导致安全机制被用户关掉。
- `_check_sudo_stdin_guard` 无条件阻断 `sudo -S`（除非已设 `SUDO_PASSWORD`），专门防密码爆破/提权这一高危用法。
- YOLO 全局开关 `_YOLO_MODE_FROZEN` 在模块导入时冻结，防运行中途篡改环境偷开；会话级 YOLO 用 `_session_yolo` 集合精细控制；但二者都只能放行 DANGEROUS，对 HARDLINE 无效。
- 审批模式默认 `manual`，超时默认 60 秒，cron 审批默认 `deny`（无人值守 fail-closed）；`_normalize_approval_mode` 专门修补 YAML 1.1 把 `off` 解析成 `False` 的坑。
- `agent/file_safety.py` 把 `auth.json` / `config.yaml` / `webhook_subscriptions.json` 三个"能修改安全机制本身"的控制面文件在 home 与 root 两处都设为写禁止，并保护 `mcp-tokens/`、`pairing/` 目录；`HERMES_WRITE_SAFE_ROOT` 可把写权限整个圈进沙盒。
- 读取侧阻断 `.env` 系列文件防凭证泄露；所有路径判断都先 `resolve()` 再 `relative_to` 比较，防 `../` 与符号链接逃逸。

## 动手实验

1. **实验一：给两级名单分类** —— `Read` `tools/approval.py` 的 `HARDLINE_PATTERNS`（约第 255 行）与 `DANGEROUS_PATTERNS`（约第 373 行）。挑五条命令（`rm -rf /`、`rm -rf ./build`、`chmod 777 .`、`mkfs.ext4 /dev/sdb`、`DROP TABLE users`），逐一判断它落在硬线还是危险层，以及在 YOLO 开启时分别会怎样。

2. **实验二：拆解 DELETE 正则** —— 把不带 `WHERE` 的 DELETE 正则 `\bDELETE\s+FROM\b(?![^\n]*\bWHERE\b)` 拆成几段解释，尤其是否定前瞻 `(?!...)` 的作用。构造一条会命中的 SQL 和一条不会命中的 SQL，验证你的理解，并思考"高精度规则"对避免误报的意义。

3. **实验三：验证 YOLO 的边界** —— 用 Grep 定位 `_YOLO_MODE_FROZEN`（约第 29 行）与硬线检测 `detect_hardline_command`（约第 326 行）。论证：为什么 YOLO 模式无法绕过 `HARDLINE_PATTERNS`？再解释"导入时冻结"相比"运行时读环境变量"防住了什么攻击。

4. **实验四：理解控制面文件保护** —— `Read` `agent/file_safety.py:is_write_denied`（约第 96 行）。说明为什么 `auth.json`、`config.yaml`、`webhook_subscriptions.json` 要比普通文件保护得更严（提示：如果 Agent 能改 `config.yaml` 里的审批模式会怎样）。再找到 `HERMES_WRITE_SAFE_ROOT` 的逻辑，描述它如何把整个写权限收进一个沙盒目录。

> **下一章预告**：命令通过了审批，接下来就要真正执行——但在哪里执行？第 6 章将解构 Hermes 的沙箱抽象 `tools/environments/`，看清 `BaseEnvironment` 抽象基类如何统一 local、Docker、SSH、Singularity、Modal、Daytona 六种终端后端，以及 `_wrap_command` 如何把一条命令包裹成可隔离、可捕获退出码、可跨调用保持环境与工作目录的 bash 脚本。
