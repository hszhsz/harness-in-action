# 第 11 章　认证与登录：设备码、keyring 与 401 自愈

第 10 章配置定稿了，但有一项"配置"特殊到必须单独成章——**认证**。Codex 怎么知道"你是谁、用哪个账户的额度跑模型"？一个会自己跑命令、改代码、连模型的 Agent，凭证就是它的命门：令牌存哪、怎么刷新、过期了怎么办、被吊销了怎么办，每一个环节出错都意味着"要么用不了、要么泄露了"。

这一章拆 Codex 的认证体系：多种认证模式如何统一抽象成 `CodexAuth`、ChatGPT 的**设备码登录**流程、PKCE 如何防中间人、令牌如何用 **keyring** 安全存储、access token 怎么**主动+被动**双轨刷新，以及当请求撞上 401 时第 5 章那个 `PendingUnauthorizedRetry` 如何配合一台**状态机**把请求救回来。代码主要在独立的 `login/` crate、`keyring-store/` crate，以及 `core/src/client.rs`。

---

## 11.1 一种凭证，六种形态：`CodexAuth`

Codex 支持的认证方式不止一种，但它把它们收敛进一个枚举 `CodexAuth`（`login/src/auth/manager.rs`），六个变体：

```rust
pub enum CodexAuth {
    ApiKey(ApiKeyAuth),
    Chatgpt(ChatgptAuth),
    ChatgptAuthTokens(ChatgptAuthTokens),
    AgentIdentity(AgentIdentityAuth),
    PersonalAccessToken(PersonalAccessTokenAuth),
    BedrockApiKey(BedrockApiKeyAuth),
}
```

这六种各有适用场景：`ApiKey` 是经典的 `OPENAI_API_KEY`/`CODEX_API_KEY`；`Chatgpt` 是用 ChatGPT 订阅登录、走 Codex 后端的 OAuth 令牌；`ChatgptAuthTokens` 是由外部父应用注入的、活在内存里的 ChatGPT 令牌；`AgentIdentity` 是 agent 身份 JWT；`PersonalAccessToken` 是个人访问令牌；`BedrockApiKey` 给 AWS Bedrock 用。无论哪种，上层只通过 `get_token()`（取 bearer 令牌）、`auth_mode()`、`get_account_id()` 这些统一方法打交道——**调用方不必关心凭证到底是 API key 还是 OAuth access token**。这正是第 6 章工具系统"统一契约、多态实现"思路在认证层的复刻。

注意 `CodexAuth` 的 `PartialEq` 实现很讲究：大多数模式只比较 `api_auth_mode()`，但 `PersonalAccessToken` 和 `BedrockApiKey` 会真正比较内容。这是为后面"判断 auth 有没有变"埋的伏笔。

---

## 11.2 设备码登录：没有浏览器回调也能登

最有意思的是 ChatGPT 的**设备码登录**（`login/src/device_code_auth.rs`）。传统 OAuth 要在本地起一个 server 接收浏览器回调，但很多场景（SSH 到远端、容器、无图形界面）根本没法弹浏览器、也没法监听端口。设备码流程绕开了这个限制：

1. **请求用户码**：`request_user_code` 向 `{auth_base_url}/deviceauth/usercode` POST 一个 `client_id`，拿回 `device_auth_id`、`user_code` 和轮询 `interval`。
2. **提示用户**：`print_device_code_prompt` 在终端打印一段引导——让用户在**任意设备**的浏览器里打开 `{base_url}/codex/device`，输入那个一次性 `user_code`。注释里那句尤其值得记下：`"Device codes are a common phishing target. Never share this code."`（设备码是常见的钓鱼目标，绝不要分享）——把安全提醒直接写进给用户的提示里。
3. **轮询令牌**：`poll_for_token` 死循环地向 `{auth_base_url}/deviceauth/token` POST，直到拿到授权码。这里的状态处理很干净：HTTP 成功就返回；遇到 `FORBIDDEN`(403) 或 `NOT_FOUND`(404) 表示"用户还没授权完"，于是 sleep 一个 `interval` 再轮询；其他状态码直接报错。
4. **超时兜底**：`max_wait = Duration::from_secs(15 * 60)`——**整个授权窗口 15 分钟**，超时则报 `"device auth timed out after 15 minutes"`。这与提示里"expires in 15 minutes"严格一致——文档和代码不是两套数字。

而且代码很诚实地处理了"服务端不支持设备码"的情况：`request_user_code` 收到 404 时不是抛一个含糊的错误，而是明确说 `"device code login is not enabled for this Codex server. Use the browser login or verify the server URL."`——又一次"错误即文档"，告诉用户该改用浏览器登录或检查 URL。

---

## 11.3 PKCE：即便授权码被截获也无用

设备码流程拿到的是一个**授权码（authorization code）**，还要拿它去换真正的令牌。这一步如果被中间人截走授权码，攻击者就能冒充你换令牌。Codex 用 **PKCE**（Proof Key for Code Exchange）堵死这条路，实现就在小巧的 `login/src/pkce.rs`：

```rust
pub fn generate_pkce() -> PkceCodes {
    let mut bytes = [0u8; 64];
    rand::rng().fill_bytes(&mut bytes);
    // Verifier: URL-safe base64 without padding (43..128 chars)
    let code_verifier = base64::engine::general_purpose::URL_SAFE_NO_PAD.encode(bytes);
    // Challenge (S256): BASE64URL-ENCODE(SHA256(verifier)) without padding
    let digest = Sha256::digest(code_verifier.as_bytes());
    let code_challenge = base64::engine::general_purpose::URL_SAFE_NO_PAD.encode(digest);
    PkceCodes { code_verifier, code_challenge }
}
```

原理：客户端先生成一个高熵随机 `code_verifier`（64 字节随机数做 URL-safe base64），再算出它的 SHA-256 哈希作为 `code_challenge`。授权请求里只发 `code_challenge`（公开的哈希），换令牌时才发 `code_verifier`（私密的原文）。服务端验证 `SHA256(verifier) == challenge`。这样即使授权码在传输中被截获，攻击者没有那个**只存在于本机内存的 `code_verifier`**，就换不出令牌。注意这里用的是 PKCE 的 **S256** 方法（SHA-256），而非弱的 `plain`——选了最稳的那档。设备码流程在 `complete_device_code_login` 里把服务端返回的 `code_verifier`/`code_challenge` 组装成 `PkceCodes`，再调 `exchange_code_for_tokens` 完成兑换。

---

## 11.4 令牌存哪：从 `auth.json` 到 keyring 的四档存储

换来的令牌（id_token / access_token / refresh_token）要落地保存，否则每次启动都得重新登录。Codex 把"存哪"抽象成 `AuthStorageBackend` trait（`login/src/auth/storage.rs`），有四种实现，对应配置项 `AuthCredentialsStoreMode` 的四档：

- **`File`** → `FileAuthStorage`：写明文 JSON 到 `$CODEX_HOME/auth.json`。但有一处关键防御——在 Unix 上用 `options.mode(0o600)` 创建文件，**只有属主能读写**。令牌是敏感数据，绝不能让同机其他用户读到。
- **`Keyring`** → `KeyringAuthStorage`：写进操作系统的密钥环（macOS Keychain、Windows Credential Manager、Linux Secret Service），服务名固定 `KEYRING_SERVICE = "Codex Auth"`。这是比明文文件更安全的存储。
- **`Auto`** → `AutoAuthStorage`：**优先 keyring，失败回退文件**。`load`/`save` 都先试 keyring，出错就 `warn!` 并降级到 file storage——既要安全，又不能因为某些环境没有可用 keyring 就完全登不上。
- **`Ephemeral`** → `EphemeralAuthStorage`：存进一个进程内的全局 `HashMap`（`EPHEMERAL_AUTH_STORE`），不落盘。外部注入的 ChatGPT 令牌（`ChatgptAuthTokens`）就走这条——它本就不该持久化。

keyring 存储里有个细节很见功力：键名不是直接用路径，而是 `compute_store_key` 把 `codex_home` 路径 canonical 化后做 SHA-256、取前 16 个 hex 字符，拼成 `cli|{truncated}`。为什么？因为不同的 `CODEX_HOME` 应该对应不同的凭证条目，但路径本身可能很长、含特殊字符、或泄露目录结构——哈希成短而稳定的 key 既规避了这些问题，又保证同一个 home 始终映射到同一条目。而且 `KeyringAuthStorage::save` 成功写 keyring 后，会顺手 `delete_file_if_exists` 删掉可能残留的明文 `auth.json`——**升级到 keyring 后不留明文尾巴**。

keyring 操作本身封装在独立的 `keyring-store/` crate：`KeyringStore` trait 三个方法 `load`/`save`/`delete`，`DefaultKeyringStore` 是真实实现，而 crate 里还内置了一个 `MockKeyringStore`（用 `Arc<Mutex<HashMap>>` + `MockCredential`）供测试注入——这又是第 12 章会反复看到的"用依赖注入让安全组件可测"的范式。

---

## 11.5 双轨刷新：主动续期与被动救援

access token 是会过期的。Codex 的刷新分**两条轨道**，都收敛在 `AuthManager`（`login/src/auth/manager.rs`）这个"认证单一真相源"里。

### 主动刷新（proactive）

每次调 `AuthManager::auth()` 取凭证时，会先问 `should_refresh_proactively`：如果是 ChatGPT 模式，且 access token 的 JWT 过期时间 `expires_at <= now + 5 分钟`（`CHATGPT_ACCESS_TOKEN_REFRESH_WINDOW_MINUTES = 5`），就提前刷新——**不等它真过期，留 5 分钟缓冲**。如果 JWT 里读不到过期时间，则退而看 `last_refresh`：超过 `TOKEN_REFRESH_INTERVAL = 8` 天没刷就刷。这种"快过期就续期"避免了请求打到一半因令牌过期而失败。

### 刷新的并发与一致性保护

刷新是有副作用的网络操作，多个并发请求同时刷新会浪费、甚至互相覆盖。`AuthManager` 用 `refresh_lock: Semaphore::new(1)` 把刷新串行化——**同一时刻只有一个刷新在飞**。更妙的是 `refresh_token` 的"守卫式重载"逻辑：拿到锁后先 `reload_if_account_id_matches`，

- 如果磁盘上的令牌已经**变了**（`ReloadedChanged`）——说明别的进程/实例已经刷过了，直接用新的，**跳过这次刷新**；
- 如果磁盘令牌**没变**（`ReloadedNoChange`）——才真的去找令牌授权方刷新；
- 如果账户 ID **不匹配**（`Skipped`）——说明用户已登出或换了账户，报 `REFRESH_TOKEN_ACCOUNT_MISMATCH_MESSAGE`。

这套设计避免了"多个 Codex 实例共用一个 `auth.json` 时反复互相刷新令牌"的惊群问题。刷新成功后 `persist_tokens` 把新令牌写回存储、更新 `last_refresh`，再 `reload()` 让内存缓存同步。

### 永久失败的快速失败缓存

刷新可能**永久性**失败——refresh token 过期、被复用、被吊销。`request_chatgpt_token_refresh` 把后端返回的错误码分类成 `RefreshTokenFailedReason`：`refresh_token_expired` → `Expired`、`refresh_token_reused` → `Exhausted`、`refresh_token_invalidated` → `Revoked`，每种都有专门的、给人看的提示（如 `REFRESH_TOKEN_EXPIRED_MESSAGE = "Your access token could not be refreshed because your refresh token has expired. Please log out and sign in again."`）。一旦判定永久失败，`record_permanent_refresh_failure_if_unchanged` 会把这个失败**缓存到当前 auth 快照**上，后续对同一凭证的刷新尝试直接 `fail fast`、不再打网络——别让一个注定失败的刷新反复浪费往返。

---

## 11.6 401 自愈：`UnauthorizedRecovery` 状态机

主动刷新管"快过期"，但总有令牌在请求飞行途中失效、服务端回 **401** 的时候。第 5 章我们见过 `client.rs` 里那个 `PendingUnauthorizedRetry`——现在揭晓它背后的引擎：`UnauthorizedRecovery` 状态机（`login/src/auth/manager.rs`）。

它的设计注释把哲学讲得很清楚：客户端每遇到一次 401 就调一次 `next()`，**每次重试只走一步**。对 API key 认证，它什么都不做、让错误冒泡给用户（API key 错了刷新也没用）。对 ChatGPT 认证，它分两步走：

1. **`Reload`**：先尝试从磁盘重载 auth——但**仅当账户 ID 匹配**当前进程运行的账户时才重载（防止串号）。
2. **`RefreshToken`**：用 OAuth 刷新流程刷新令牌。

如果两步走完服务端仍回 401，就让错误冒泡。对外部 auth（`ChatgptAuthTokens` 或自定义 provider 的 bearer），则走 `ExternalRefresh` 一步——向父应用要新令牌。状态机用 `UnauthorizedRecoveryStep` 枚举（`Reload`/`RefreshToken`/`ExternalRefresh`/`Done`）建模，`has_next()` 判断还有没有可走的步、`unavailable_reason()` 给出"为什么救不了"的诊断字符串（`"not_chatgpt_auth"`、`"recovery_exhausted"`、`"no_external_auth"` 等）——排查认证问题时，这些字符串直接指向卡在哪。

在 `core/src/client.rs` 侧，每次发请求前用 `AuthManager::unauthorized_recovery()` 拿一个状态机，`handle_unauthorized` 在撞 401 时驱动它 `next()`，成功就用 `PendingUnauthorizedRetry::from_recovery` 标记"这次重试是 401 恢复触发的"，并把 `recovery_mode`/`recovery_phase` 记进遥测（第 15 章）。这就把第 5 章那条重试链补完整了：**网络层的重试负责"连接抖动"，而 401 自愈负责"凭证失效"——两种失败，两套恢复，但共用同一条重试循环。**

注意 `AuthManager` 的另一条设计铁律（doc 注释明说）：**外部对 `auth.json` 的修改不会被自动观察到，除非显式 `reload()`。** 这是刻意为之——避免程序不同部分在一轮运行中看到不一致的认证数据。这与第 4 章 `ThreadConfigSnapshot`、第 10 章 `ConfigBuilder` 定稿快照是同一种"运行期状态要稳定、不中途漂移"的工程直觉。

---

## 11.7 登录限制：企业能强制登录方式与工作区

第 10 章说"约束只能收紧"，认证层也有它的体现。`enforce_login_restrictions`（`login/src/auth/manager.rs`）让企业能强制两类约束：

- **`forced_login_method`**：强制必须用 API key 或必须用 ChatGPT。如果当前凭证违反（比如要求 API key 却在用 ChatGPT），直接登出并报明确原因。
- **`forced_chatgpt_workspace_id`**：把登录限制到特定 workspace（账户）。如果当前 ChatGPT 凭证的 `chatgpt_account_id` 不在允许列表里，登出并报 `"Login is restricted to workspace(s) ...，but current credentials belong to ..."`。

而且登出时 `logout_all_stores` 会**同时清理 ephemeral 和持久化两个 store**——确保强制登出真正抹掉所有活跃凭证，不留死角。这与第 8 章"fail-closed、不留绕过缝隙"一脉相承：企业约束一旦违反，宁可登出也不放行。

---

## 本章小结

- 多种认证统一抽象为 `CodexAuth` 六变体（`ApiKey`/`Chatgpt`/`ChatgptAuthTokens`/`AgentIdentity`/`PersonalAccessToken`/`BedrockApiKey`），上层只用 `get_token()`/`auth_mode()` 等统一方法——契约统一、多态实现。
- ChatGPT **设备码登录**（`device_code_auth.rs`）无需浏览器回调：请求 `user_code` → 提示用户在任意设备打开链接输码 → 轮询 token endpoint（403/404 表示未完成则 sleep `interval` 重试）→ **15 分钟超时**；提示里直接写防钓鱼警告；服务端不支持时给明确改用浏览器的提示。
- **PKCE**（`pkce.rs`）用 64 字节随机 `code_verifier` + 其 SHA-256 `code_challenge`（S256 方法），即便授权码被截获、无 verifier 也换不出令牌。
- 令牌存储 `AuthStorageBackend` 四档（`AuthCredentialsStoreMode`）：`File`（Unix `0o600` 私有权限）/`Keyring`（服务名 `"Codex Auth"`，键 `cli|<sha256前16位>`，写成功后删明文残留）/`Auto`（keyring 优先、失败回退文件）/`Ephemeral`（进程内 HashMap，不落盘）；keyring 操作封装在 `keyring-store` crate，内置 `MockKeyringStore` 供测试注入。
- **双轨刷新**：主动刷新（access token 过期前 5 分钟 `CHATGPT_ACCESS_TOKEN_REFRESH_WINDOW_MINUTES`，或 `last_refresh` 超 8 天 `TOKEN_REFRESH_INTERVAL`）；`refresh_lock: Semaphore(1)` 串行化刷新，守卫式重载避免多实例惊群；永久失败（expired/reused/invalidated）被缓存以快速失败。
- **401 自愈** `UnauthorizedRecovery` 状态机：每次 401 走一步（`Reload` 仅账户匹配时 → `RefreshToken`；外部 auth 走 `ExternalRefresh`）；与 `core/src/client.rs` 的 `PendingUnauthorizedRetry` 配合，补全第 5 章"网络重试 + 凭证自愈"两套恢复共用一条重试循环。
- `AuthManager` 是认证单一真相源，外部改 `auth.json` 不自动生效、需显式 `reload()`——运行期状态稳定不漂移（呼应 ch04/ch10 快照思想）。
- `enforce_login_restrictions` 让企业强制登录方式与 workspace，违反即登出（清理所有 store）——约束只能收紧在认证层的落地。

## 动手实验

1. **实验一：跟踪设备码登录** — 阅读 `login/src/device_code_auth.rs` 的 `request_user_code`、`poll_for_token`、`run_device_code_login`。说明用户在终端看到的两步提示分别对应哪个 HTTP 请求，并解释为什么 `poll_for_token` 把 403/404 当作"还没授权完"而非错误。找到那个 15 分钟超时常量。
2. **实验二：理解 PKCE 防截获** — 阅读 `login/src/pkce.rs` 的 `generate_pkce`。解释 `code_verifier` 与 `code_challenge` 的关系，以及为什么授权请求只发 challenge、换令牌才发 verifier 能挡住中间人。指出它用的是 S256 而非 plain。
3. **实验三：辨析四档存储** — 在 `login/src/auth/storage.rs` 找到 `AuthStorageBackend` 的四个实现与 `create_auth_storage` 的 match。说明 `Auto` 模式的回退顺序、`File` 模式的 `0o600` 意义、以及 `compute_store_key` 为什么要对路径做哈希而非直接当 key。
4. **实验四：画出 401 自愈状态机** — 阅读 `login/src/auth/manager.rs` 的 `UnauthorizedRecovery`，列出 `UnauthorizedRecoveryStep` 的状态转移（managed 模式 vs external 模式各走哪些步）。再到 `core/src/client.rs` 找 `PendingUnauthorizedRetry` 与 `handle_unauthorized`，说明一次 401 是如何被一步步救回、且每步都记进遥测的。

> **下一章预告**：认证解决了"你是谁"，但模型要写好你的代码，还得知道"你的项目长什么样、有什么规矩"。下一章我们进入上下文工程：Codex 如何组装发给模型的指令（base instructions）、`AGENTS.md` 如何被发现与注入、用户记忆与项目约定怎样进入对话，以及当上下文逼近窗口上限时，第 4 章那个 compaction 如何决定"丢什么、留什么"。
