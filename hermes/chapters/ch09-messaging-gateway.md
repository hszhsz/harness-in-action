# 第 9 章　消息网关与多平台接入

到这里，我们解构的 Hermes 一直活在终端里。但它真正的野心写在 README 的第一句：让你"从 Telegram 跟它对话，而它在云端 VM 上干活"。Hermes 能从 Telegram、Discord、Slack、WhatsApp、Signal、飞书、Matrix、钉钉、企业微信、邮件、短信等十余个平台接收消息，全部由**一个网关进程**统一承载。

把十几个 IM 平台塞进一个进程，工程上的难点不在于"能不能连上"，而在于：怎么用一套抽象统一这些 API 各异、消息格式各异、长度限制各异的平台？怎么安全地把一个陌生的聊天账号"配对"到你的 Agent？怎么把 Agent 那动辄几千字的回复，投递到一个单条消息只允许几千字符的平台？这一章我们就读 `gateway/` 这个目录。

## 9.1 统一异构平台：`PlatformEntry`

第 4 章的工具、第 8 章的 provider，都用"一份数据描述 + 注册表"的模式统一异构对象。网关平台也不例外——`gateway/platform_registry.py` 约第 39 行的 `@dataclass class PlatformEntry` 就是描述一个平台的数据结构。

它的字段把"接入一个 IM 平台需要知道什么"都列了出来：`name`、`label`、`adapter_factory`（创建该平台适配器的工厂）、`check_fn`（可用性检查）、`validate_config`、`is_connected`、`required_env`（需要的环境变量，比如 bot token）、`install_hint`（缺依赖时的安装提示）、`setup_fn`、`source="plugin"`、`plugin_name`、`allowed_users_env`、`allow_all_env`、`max_message_length=0`、`pii_safe=False`、`emoji="🔌"`、`allow_update_command=True`。

两个字段特别值得停下来看。`max_message_length`（默认 0，0 表示不限）记录了**该平台单条消息的字符上限**——Telegram、Discord、Slack 各不相同，把它做成平台的属性，投递层就能据此切块（9.4 节详述）。`pii_safe`（默认 False）标记**这个平台是否适合传输个人隐私信息**——这是一个安全属性，意味着 Hermes 在设计上就把"某些平台不该发敏感内容"这件事编码进了平台描述里。注册与创建由 `register(...)` 和 `create_adapter(name, config)` 完成，模式与前几章的注册表一脉相承。

## 9.2 平台适配器基类：`BasePlatformAdapter`

每个平台的具体通信逻辑由一个适配器承担，它们共同继承 `gateway/platforms/base.py` 约第 1786 行的 `class BasePlatformAdapter(ABC)`。这个抽象基类规定了所有平台必须实现的三个抽象方法：`connect()`、`disconnect()`、`send(...)`——连接、断开、发送，这是任何 IM 集成的最小公共面。

围绕这个基类，base.py 还定义了一整套消息领域模型：`MessageEvent`（约第 1413 行，一条进来的消息）、`SendResult`（约第 1542 行，一次发送的结果）、`MessageType(Enum)`、`ProcessingOutcome(Enum)`、`EphemeralReply(str)`（短暂回复，发完即逝）、`CachedMedia`、`TextDebounceState`（文本去抖状态）。这套模型把"不同平台五花八门的消息事件"归一化成统一的内部表示——无论消息来自 Telegram 还是飞书，进到网关内部后都是一个 `MessageEvent`，上层逻辑不必关心它的出身。

`gateway/platforms/` 目录下是各平台的具体实现：`telegram.py`、`slack.py`、`whatsapp.py`、`signal.py`、`feishu.py`、`matrix.py`、`dingtalk.py`、`wecom.py`、`weixin.py`、`email.py`、`sms.py`、`webhook.py`、`api_server.py`、`bluebubbles.py`，以及 `qqbot/` 子目录。其中 `__init__.py` 用 PEP 562 的 `__getattr__` 做**懒加载**（如 `QQAdapter`、`YuanbaoAdapter`）——只有真正用到某个平台时才导入它的依赖，避免为了支持十几个平台就在启动时拉起所有平台的 SDK。这又是第 4 章"按需导入控制冷启动"思路在网关层的复用。

发送侧的可靠性由 `_send_with_retry`（约第 3329 行）保障——平台 API 抖动、临时失败时自动重试，确保消息尽量送达。

## 9.3 设备配对：`PairingStore` 的密码学细节

让一个陌生的聊天账号能指挥你的 Agent，是个高危操作——必须有严格的配对（pairing）机制，否则任何人给你的 bot 发消息都能控制它。`gateway/pairing.py` 的 `class PairingStore`（约第 81 行）把这件事做得相当严谨，值得逐一品读它的密码学选择。

配对码用 `CODE_LENGTH = 8` 位，有效期 `CODE_TTL_SECONDS = 3600`（源码注释直言 "Codes expire after 1 hour"）。**短码 + 短有效期**是抗暴力破解的基本组合——即便有人想猜码，1 小时的窗口也极大限制了尝试空间。

生成配对码用的是 `secrets.choice(ALPHABET)` 循环 8 次（`generate_code()`），条目 id 用 `secrets.token_hex(8)`。注意这里用的是 `secrets` 而非 `random`——`secrets` 是 Python 专为密码学场景设计的随机源，而 `random` 是可预测的伪随机，绝不能用于安全令牌。源码文件开头的注释也专门点出 "Cryptographic randomness via secrets.choice()"。

验证环节更见功力：`secrets.compare_digest(candidate_hash, entry["hash"])`。用 `compare_digest` 而非普通的 `==` 来比较哈希，是为了防**时序攻击（timing attack）**——普通字符串比较会在第一个不匹配的字符处提前返回，攻击者能通过测量比较耗时逐字符猜出正确值；`compare_digest` 则保证比较时间恒定，不泄露任何信息。而且，配对码是**以哈希形式存储**的（比较的是 `candidate_hash` 和 `entry["hash"]`），即使配对文件泄露，攻击者也拿不到明文码。

存储布局上，每个平台有 `{platform}-pending.json`（待批准）、`{platform}-approved.json`（已批准）和 `_rate_limits.json`（限流），过期清理由 `CODE_TTL_SECONDS` 把关。把这些细节连起来——密码学随机、哈希存储、恒定时间比较、短有效期、限流——`PairingStore` 几乎把"安全配对"的每一个已知坑都堵上了。**配对是网关的信任入口，而 Hermes 在这个入口上没有任何偷懒**。

## 9.4 会话管理：哈希化的身份与会话键

网关要同时服务很多用户、很多平台的很多对话，会话管理必须既能区分又能保护隐私。`gateway/session.py` 的 `class SessionStore`（约第 685 行）、`@dataclass SessionEntry`（约第 426 行）、`class SessionContext`、`class SessionSource` 共同承担这件事。

一个关键的隐私设计：发送者和聊天的标识符都被**哈希化**存储——`_hash_sender_id`（约第 39 行）、`_hash_chat_id`。也就是说，Hermes 在自己的会话存储里不直接保存"某某人的 Telegram 用户 ID"明文，而是存它的哈希。这样即便存储被翻看，也无法直接反推出真实的平台身份。会话 id 的格式是 `f"{now:%Y%m%d_%H%M%S}_{uuid.uuid4().hex[:8]}"`——时间戳前缀便于排序和调试，随机后缀保证唯一。

会话有 `suspended`（挂起）、`resume_pending`（待恢复）等状态，配套 `build_session_key()`（约第 601 行）生成会话键、`is_shared_multi_user_session()` 判断是否多人共享会话。多人共享会话是个微妙的场景——一个群里多人和同一个 Agent 对话，会话状态怎么隔离、隐私怎么保障，都需要专门处理。

## 9.5 投递：把长回复切成平台能咽下的块

最后一个实际问题：Agent 的回复经常很长（一段分析、一份报告几千字很正常），但平台对单条消息有字符上限。`gateway/delivery.py` 解决投递的"最后一公里"。

核心常量是 `MAX_PLATFORM_OUTPUT = 4000`（约第 23 行）——这是一个保守的通用上限。`class DeliveryRouter`（约第 175 行）的 `async deliver(...)`（约第 195 行）在投递时检查：如果 `len(content) > MAX_PLATFORM_OUTPUT`（约第 320 行），就把内容**切块**后分多条发送。配套的 `class DeliveryTarget`（约第 95 行）描述投递目标。

切块看似简单，实则要处理不少细节：不能从单词或代码块中间硬切（会破坏可读性和 Markdown 渲染），要尽量在自然边界（段落、换行）断开。把这个上限设成一个统一常量、而不是散落在各平台代码里，意味着投递策略可以集中调整——再结合 9.1 节 `PlatformEntry.max_message_length`，网关既有一个通用的保守上限，又能让每个平台用自己更精确的限制。

把这一章串起来看，`gateway/` 是一个把"异构平台统一、安全配对、隐私会话、可靠投递"四件事各自封装好的子系统。它让 Hermes 从一个终端工具，变成了一个"活在你日常用的每个聊天软件里"的存在——而支撑这种无处不在的，是 `PlatformEntry` 的数据化抽象、`PairingStore` 一丝不苟的密码学、`SessionStore` 的哈希化隐私保护，和 `DeliveryRouter` 的切块投递。

## 本章小结

- 十余个 IM 平台由单个网关进程承载；异构平台用 `@dataclass PlatformEntry`（`gateway/platform_registry.py` 约第 39 行）+ 注册表统一，模式同工具/provider。
- `PlatformEntry` 的 `max_message_length` 把单条消息上限做成平台属性供投递层切块；`pii_safe` 把"该平台是否适合传隐私"编码进平台描述，是设计层面的安全属性。
- 所有平台适配器继承 `BasePlatformAdapter(ABC)`，抽象方法仅 `connect` / `disconnect` / `send`；base.py 定义 `MessageEvent` / `SendResult` 等领域模型把异构消息归一化；`platforms/__init__.py` 用 PEP 562 `__getattr__` 懒加载平台 SDK 控冷启动。
- 配对 `PairingStore` 的密码学严谨：`CODE_LENGTH = 8` + `CODE_TTL_SECONDS = 3600` 短码短有效期抗暴破；用 `secrets`（非 `random`）生成；`secrets.compare_digest` 防时序攻击；配对码以哈希存储防泄露。
- 会话 `SessionStore` 把发送者/聊天 ID 哈希化存储保护隐私（`_hash_sender_id`）；会话 id 格式 `时间戳_uuid8`；支持 `suspended` / `resume_pending` 状态与多人共享会话判断。
- 投递 `DeliveryRouter` 用 `MAX_PLATFORM_OUTPUT = 4000` 统一上限，超长内容切块多发；上限集中为常量便于统一调整，并与 `PlatformEntry.max_message_length` 互补。

## 动手实验

1. **实验一：读懂平台描述** —— `Read` `gateway/platform_registry.py` 的 `PlatformEntry`（约第 39 行）。逐个解释 `required_env`、`max_message_length`、`pii_safe`、`allowed_users_env` 的作用。设想接入一个新平台，你需要填哪些字段才能让网关认识它？

2. **实验二：审计配对的密码学** —— `Read` `gateway/pairing.py` 的 `generate_code()` 与验证逻辑。回答三个问题：为什么用 `secrets.choice` 而不是 `random.choice`？为什么用 `secrets.compare_digest` 而不是 `==`？为什么配对码要哈希存储而不是明文？每个问题对应一种被防住的攻击。

3. **实验三：跟踪一次切块投递** —— `Read` `gateway/delivery.py` 约第 320 行 `len(content) > MAX_PLATFORM_OUTPUT` 的分支。构造一段 9000 字符的回复，推演它会被切成几块、如何分别投递。思考：如果切块时不考虑代码块边界会出什么问题？

4. **实验四：验证会话身份的隐私保护** —— 用 Grep 找到 `_hash_sender_id`（约第 39 行）的全部调用点。论证：把发送者 ID 哈希化存储，在"会话存储文件被泄露"的场景下保护了什么？再看会话 id 格式 `时间戳_uuid8`，解释时间戳前缀对调试的价值。

> **下一章预告**：网关收发的每一条消息、Agent 跑过的每一个回合，都需要被持久化下来——否则重启即失忆、更谈不上"跨会话回忆"。第 10 章将深入近 20 万字节的 `hermes_state.py`，解构它如何用 SQLite 存储会话与消息、如何用 FTS5（含 trigram 分词支持 CJK）做全文检索、以及面对 NFS/SMB 这类"WAL 不友好"的文件系统时如何优雅降级。
