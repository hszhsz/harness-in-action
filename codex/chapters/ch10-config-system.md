# 第 10 章　配置系统：多层加载、合并与"约束只能收紧"

前面九章里出现了大量"行为旋钮"：审批策略 `AskForApproval`、沙箱策略 `SandboxPolicy`、模型选择、重试上限、网络开关……这些值从哪来？答案是配置系统。但 Codex 的配置远不止"读一个 TOML 文件"那么简单——它要同时满足两类彼此冲突的诉求：**用户想自由定制**，而**企业管理员想强制约束**。当一个公司把 Codex 部署给上千名工程师，管理员需要确保"任何人都不能把审批策略改成 `Never`"——哪怕用户在自己的配置文件里写了。

这一章拆 Codex 如何用"分层 + 合并 + 约束"三件套优雅地解决这个矛盾。代码集中在独立的 `config/` crate 和 `core/src/config/`。核心要记住一句话：**配置可以分层覆盖，但约束只能逐层收紧，绝不放松。**

---

## 10.1 配置的形态：`ConfigToml` 与分层

用户写的配置是 `ConfigToml`（`config/src/config_toml.rs`）——它就是 `$CODEX_HOME/config.toml` 反序列化后的结构体，字段对应你能在配置文件里写的每一项。第 2 章提过 `AGENTS.md` 的一条铁律："改了 `ConfigToml` 就要跑 `just write-config-schema` 更新 `config.schema.json`"——配置即契约，结构体一变，对外的 JSON Schema 就得同步。

但 Codex 的配置不是单一文件，而是**多个来源的层叠**。每一层用 `ConfigLayerSource`（`app-server-protocol/src/protocol/v2/config.rs`）标识来源，并且——这是关键——每个来源有一个明确的**优先级数字** `precedence()`：

| 来源 | precedence | 含义 |
|---|---|---|
| `Mdm` | 0 | macOS 的 MDM 托管偏好（企业下发） |
| `System` | 10 | 系统级 `managed_config.toml` |
| `EnterpriseManaged` | 15 | 云端配置包下发的企业层 |
| `User` | 20 区间 | 用户 `$CODEX_HOME/config.toml`（+ profile 叠加） |
| `Project` | 25 | 工作区内的 `.codex/` 文件夹 |
| `SessionFlags` | 30 | 命令行 `-c`/`--config` 临时覆盖 |
| `LegacyManagedConfigTomlFromFile` | 40 | 遗留的 managed_config（淘汰中） |
| `LegacyManagedConfigTomlFromMdm` | 50 | 遗留的 MDM managed_config |

`ConfigLayerStack`（`config/src/state.rs`）把这些层装进一个 `Vec<ConfigLayerEntry>`，注释写明："**从最低优先级（base）到最高优先级（top）排列，后面的条目覆盖前面的。**" 它还记着一个 `user_layer_index`——指向"可写的那一层用户配置"，因为只有用户层是用户能改的、profile-aware 编辑要写回的目标。

这个分层设计回答了"自由 vs 约束"的一半：**普通配置项越靠近用户、越靠后、优先级越高**——你的命令行 `-c` 能覆盖你的项目配置，项目配置能覆盖你的用户配置。日常定制非常灵活。

---

## 10.2 合并：`merge_toml_values` 的递归深合并

多层怎么合成一份最终配置？靠 `config/src/merge.rs` 的 `merge_toml_values(base, overlay)`——"把 `overlay` 合进 `base`，`overlay` 优先"。它的核心是 `merge_toml_values_at_path` 的**递归深合并**：

```rust
fn merge_toml_values_at_path(base, overlay, path) {
    if 两者都是 Table {
        for (key, value) in overlay_table {
            if base 里已有这个 key {
                递归合并(existing, value)   // 深入子表
            } else {
                base.insert(key, value)     // 新键直接加
            }
        }
    } else {
        *base = overlay  // 非表（标量/数组）→ overlay 整个替换 base
    }
}
```

这里的设计选择很讲究：**表（table）是深度合并的，标量和数组是整体替换的**。也就是说，如果高优先级层只设了 `[model] name = "gpt-5"`，它不会清空 base 里 `[model]` 表下的其他字段，只覆盖 `name`；但如果它设了一个数组（比如 `writable_roots = [...]`），那就是整个数组替换，而非追加。这种"表合并、值替换"的语义符合大多数人对配置覆盖的直觉。

合并过程中还顺手做了**键归一化**：`normalize_key_aliases` 处理配置项的别名（让 `model_provider` 和 `model-provider` 这类写法等价），`is_permission_network_domains_path` + `normalize_network_domain_keys` 专门把网络域名 key 用 `normalize_host` 归一化——确保 `Example.COM` 和 `example.com` 被当作同一个域。配置解析这种"边界处"对用户输入的各种写法宽容，正是第 1 章"边界不信任、但要包容"的体现。

---

## 10.3 约束：`Constrained<T>` 与"只能收紧"

合并解决了"覆盖",但还没解决核心矛盾：**如果用户层（precedence 20）优先级高于企业管理层（precedence 15），用户岂不是能用自己的配置覆盖掉管理员的强制约束？** 如果合并是唯一机制，那企业管理就形同虚设。

Codex 的答案是把"配置值"和"配置约束"**分成两条独立的轨道**。配置值走 10.2 的合并；而约束走另一套机制——`ConfigRequirements`（`config/src/config_requirements.rs`），它不参与合并，而是作为 `ConfigLayerStack` 的一个独立字段 `requirements`，在"从所有层推导出最终 Config"时**强制裁剪**最终值。

约束的载体是泛型 `Constrained<T>`（`config/src/constraint.rs`），它把一个值和一个**校验器**绑在一起：

```rust
pub struct Constrained<T> {
    value: T,
    validator: Arc<dyn Fn(&T) -> ConstraintResult<()> + Send + Sync>,
    normalizer: Option<Arc<dyn Fn(T) -> T + Send + Sync>>,
}
```

构造时 `validator(&initial_value)?` 立即校验——**一个 `Constrained<T>` 一旦存在，就保证它的值满足约束**。这是"把不变量编码进类型"的范本（第 1 章原则）：你拿到一个 `Constrained<AskForApproval>`，就不必再怀疑它是不是合法的——类型系统替你保证了。`allow_any` 构造一个"接受任意值"的版本（无约束默认），`normalized` 则用一个归一化函数把值规整到合法域。

`ConfigRequirements` 就是一组这样的受约束字段，每个还带来源（`ConstrainedWithSource`，记录这条约束是哪一层设的）：

```rust
pub struct ConfigRequirements {
    pub approval_policy: ConstrainedWithSource<AskForApproval>,
    pub permission_profile: ConstrainedWithSource<PermissionProfile>,
    pub windows_sandbox_mode: ConstrainedWithSource<...>,
    pub web_search_mode: ConstrainedWithSource<WebSearchMode>,
    pub network: Option<Sourced<NetworkConstraints>>,
    pub filesystem: Option<Sourced<FilesystemConstraints>>,
    // ……
}
```

默认值很说明问题：`permission_profile` 默认 `allow_any(PermissionProfile::read_only())`——无约束时默认只读；`web_search_mode` 默认 `Cached`。这些约束来自企业层（`requirements.toml`、MDM、云配置包），它们**不会被用户层的高优先级覆盖**，因为它们根本不在合并的轨道上。当最终从层栈推导 `Config` 时，用户合并出来的值要**先过一遍 requirements 的 validator**——如果企业约束 `approval_policy` 只允许 `{OnRequest, UnlessTrusted}`，而用户写了 `Never`，校验失败，被拒绝。

错误信息再次体现"错误即文档"：`ConstraintError::InvalidValue` 的格式是 `"invalid value for {field}: {candidate} is not in the allowed set {allowed} (set by {requirement_source})"`——不仅告诉你哪个值不合法、合法集是什么，还告诉你**这条约束是谁设的**（哪一层企业策略）。管理员排查"为什么用户的配置没生效"时，这条信息直接指向源头。

---

## 10.4 "约束只能收紧"的完整图景

现在可以把矛盾的完整解法拼出来了。Codex 用**两条正交的轨道**同时满足"用户自由"和"企业管控"：

1. **值轨道（合并，越靠用户越优先）**：日常配置项靠分层合并，用户的命令行能覆盖项目、项目能覆盖用户配置。灵活、就近、符合直觉。
2. **约束轨道（requirements，企业设定、不可被合并覆盖）**：企业层下发 `ConfigRequirements`，规定每个敏感字段的**合法取值集**。无论用户在值轨道上怎么折腾，最终值都必须落在约束允许的集合内。

这两条轨道合起来，就是贯穿全书的那句铁律的精确实现：**配置可以分层覆盖（值轨道让它更灵活），但约束只能收紧（约束轨道让安全边界单调不放松）。** 一个企业管理员可以放心地下发"审批策略必须至少是 OnRequest""权限 profile 不得超过 workspace-write""不许联网"这类约束，确信没有任何一层用户配置能把它松开。

还有几个相关开关印证这套思路：`ignore_user_and_project_exec_policy_rules`（企业可以让用户/项目的 `.rules` 文件失效，防止用户用自己的 execpolicy 规则给自己放行）、`enforce_residency`（数据驻留约束）、`network`/`filesystem` 约束。每一个都是"企业能收紧、用户不能放松"的具体旋钮。

---

## 10.5 装配：`ConfigBuilder`

最后，把分层、合并、约束、认证、模型管理拧成一个可用的 `Config`，是 `core/src/config/mod.rs` 的 `ConfigBuilder` 的职责。第 3 章我们见过 `cli/src/main.rs` 用 `ConfigBuilder` 加载配置——它是 builder 模式：接收 CLI 覆盖、`$CODEX_HOME`、cwd 等输入，依次加载各层、做合并、套用 requirements 约束、最终产出一个**已被验证、不可变**的 `Config` 供整个会话使用。

这呼应了第 4 章 `ThreadConfigSnapshot` 的设计：配置一旦被 `ConfigBuilder` 定稿，就快照进线程，一轮对话期间行为确定、不会因外部配置变动而中途漂移。**配置的生命周期是"启动时一次性装配定稿，运行时只读"**——这让 Agent 的行为可预测、可审计。

---

## 本章小结

- 用户配置是 `ConfigToml`（`config/src/config_toml.rs`），改它必须同步 `config.schema.json`（配置即契约）。
- 配置**多来源分层**，每层有 `ConfigLayerSource` 与明确的 `precedence()`：Mdm(0) < System(10) < EnterpriseManaged(15) < User(20) < Project(25) < SessionFlags(30) < Legacy(40/50)；`ConfigLayerStack` 按"低优先级在前、后者覆盖前者"排列。
- 合并 `merge_toml_values` 是**递归深合并**：表深度合并、标量/数组整体替换；过程中做键别名归一化与网络域名 host 归一化。
- 约束走**独立轨道** `ConfigRequirements`，不参与合并；载体 `Constrained<T>` 把值与 validator 绑定，构造即校验——**把不变量编码进类型**。
- 约束错误 `ConstraintError::InvalidValue` 报出非法值、合法集、**及设定该约束的来源层**（错误即文档）。
- 核心铁律：**值可分层覆盖（越靠用户越优先），约束只能收紧（企业设定、用户不可放松）**——两条正交轨道同时满足"用户自由"与"企业管控"。
- `ConfigBuilder`（`core/src/config/mod.rs`）一次性装配定稿出不可变 `Config`，快照进线程，运行时只读、行为确定。

## 动手实验

1. **实验一：画出优先级阶梯** — 阅读 `app-server-protocol/src/protocol/v2/config.rs` 的 `ConfigLayerSource::precedence`，按数字从小到大列出所有来源。设想用户在 `$CODEX_HOME/config.toml` 和命令行 `-c` 同时设了 `model`，最终生效哪个？为什么？
2. **实验二：理解表合并 vs 值替换** — 阅读 `config/src/merge.rs` 的 `merge_toml_values_at_path`，找到"两者都是 Table 则递归、否则整体替换"的分支。举例说明：高优先级层只设 `[model] name` 时，base 里 `[model]` 下的其他字段会不会被清掉？
3. **实验三：读懂 `Constrained<T>`** — 在 `config/src/constraint.rs` 阅读 `Constrained::new` 如何在构造时 `validator(&initial_value)?`。解释为什么"拿到一个 `Constrained<AskForApproval>` 就不必再校验它合法"是一种把不变量编码进类型的安全设计。
4. **实验四：验证"约束只能收紧"** — 在 `config/src/config_requirements.rs` 找到 `ConfigRequirements` 的字段（如 `approval_policy`、`permission_profile`）及其默认值。设想企业约束 `approval_policy` 只允许 `OnRequest`，而用户配置写了 `Never`——结合 `ConstraintError::InvalidValue` 的格式，说明用户会看到怎样的报错、谁被标记为约束来源。

> **下一章预告**：配置定稿了，但有一项配置特殊到要单独成章——认证。Codex 怎么知道"你是谁、用哪个账户的额度"？下一章我们进入认证与登录：ChatGPT 的设备码登录流程、令牌如何安全存储（keyring）、access token 怎么自动刷新，以及 401 失效时第 5 章那个 `PendingUnauthorizedRetry` 如何把请求救回来。
