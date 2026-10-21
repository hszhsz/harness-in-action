# 第 15 章　MCP、Codemode 与工具搜索

> 工具越多，模型越蠢——除非你不再把工具一个个塞进上下文。

一个真实的工作环境里，工具数量会迅速失控：文件、shell、Git、数据库、工单、监控、文档搜索……每个 MCP server 动辄几十个工具。如果全部声明给模型，系统提示词会膨胀到几万 token，模型的选择准确率反而下降。

pi 的应对不是「少接工具」，而是三层递进：

1. **MCP** 负责「接进来」；
2. **exposure** 负责「要不要声明」；
3. **codemode / tool_search** 负责「用的时候再找、在沙箱里编排」。

这一章把这套「工具供给链」拆开。

---

## 15.1 MCP：把外部能力接进来

### 配置与连接

pi 支持两种传输：

- **stdio**：`command` + `args` + `env` + `cwd`；
- **streamable HTTP**：`url` + `headers` + `oauth`。

旧版 SSE 传输被明确拒绝——文档建议改用 streamable HTTP（通常在 `/mcp`）。配置分两级：

```text
~/.pi/agent/mcp.json     用户级
<project>/.pi/mcp.json   项目级（需项目信任）
```

同名时项目条目覆盖用户条目。一个有用的特例：项目条目如果只写 `enabled` / `exposure` / `toolExposure`，它会**只覆盖这几个字段**，保留用户条目里的 `command`、`env`、`headers`、`auth`。于是「在这个项目里禁用某个用户级 server」只需一行：

```json
{ "mcpServers": { "internal-tools": { "enabled": false } } }
```

命令行也能管理：

```bash
pi mcp add filesystem -- npx -y @modelcontextprotocol/server-filesystem .
pi mcp add docs --url https://example.com/mcp --bearer-token-env-var DOCS_TOKEN
pi mcp list
```

`--` 之后是 server 进程的参数，不是 shell 字符串；`command` 只接受一个可执行文件。

### 工具命名与冲突

MCP 工具统一命名为：

```text
mcp__<server>__<tool>
```

非字母、数字、下划线的字符全部替换为 `_`。如果替换后发生碰撞，这些工具会统一加 hash 后缀。server 名只允许字母、数字、`_`、`-`，且 `-` 与 `_` 被视为等价——`dev-radius` 和 `dev_radius` 是同一个 server，第二个会被拒绝。

这套命名不是装饰：它让工具在 codemode 里有稳定的命名空间，也让 `searchTools(query, { namespace: "mcp__dev-radius" })` 能精确定位。

### 生命周期：后台连接，按需等待

pi 在会话启动时**后台连接所有启用的 server**，不阻塞首屏。工具在连接完成后才出现。

等待策略很讲究：

- 第一次 prompt 最多等 **10 秒**，而且只为带 `direct` 工具的 server 等——因为它们必须出现在这次请求的声明里；
- `codemode` 脚本只等它点名的 server（`mcp__<server>`）；一旦调用 `searchTools()` 或读 `ALL_TOOLS`，就等全部；
- `tool_search` 与资源工具也会等全部；
- HTTP 网络错误与 408/429/5xx 重试两次；
- 断线标记为 disconnected，下次调用时重连；
- server 宣告工具列表变化时，新增工具生效，撤回的工具变为不可达。

停止 stdio server 时，pi 关闭 stdin → SIGTERM → 对进程组 SIGKILL。这一串顺序保证了 `npx` / `uvx` 这类 wrapper 启动的子进程也能被清理干净。

### OAuth

远端 server 可以把认证完全交给 pi：

```json
{ "mcpServers": { "sentry": { "url": "https://mcp.sentry.dev/mcp" } } }
```

未认证时 `/mcp` 会提示登录，`/mcp login sentry` 或 `pi mcp login sentry` 打开授权页。pi 动态注册自己、把 token 存到 `~/.pi/agent/mcp-auth.json`、过期或被拒时刷新；scope 增加时重新登录。

几个工程细节值得注意：

- OAuth 只用于**没有 `Authorization` header** 的 HTTP server；
- 不支持动态注册的 server 可以配置 `clientId` / `clientSecret` / `callbackPort` / `callbackUrl`；
- 只支持 Client ID Metadata Document 的 server 可设 `clientRegistration: "cimd"`；
- 授权服务器的元数据地址可能配错，可用 `authServerMetadataUrl` 覆盖；
- 凭证按「server 名 + URL」归属：同名同 URL 在不同 `mcp.json` 共享登录，不同名即使 URL 相同也各登各的。

---

## 15.2 Exposure：工具「存在」不等于「被声明」

这是 pi 工具系统里最重要的一个区分（第 8 章已出现，这里给出 MCP 侧的完整语义）：

| exposure | 是否声明给模型 | 是否能被调用 | 典型用途 |
|---|---|---|---|
| `direct` | 是 | 是 | 少量高频工具 |
| `deferred` | 初始否，`tool_search` 命中后声明 | 是 | 大工具集，先搜后用 |
| `codemode`（MCP 默认） | 否 | 是（仅脚本内） | 需要组合、过滤、批量的调用 |
| `hidden` | 否 | 否 | 临时下线 / 明确禁用 |

注意 `codemode` 的工具**连 codemode 的描述里都不列**。codemode 的说明只列出「已声明」的工具（共享 3000 token 预算），其余靠 `searchTools()`、`describeTool()`、`ALL_TOOLS` 发现。这样 MCP server 连接/断开不会导致工具声明频繁变化，prompt cache 更稳定。

`deferred` 则不同：`tool_search` 一旦命中，工具会进入 active 集合，**在下一次模型调用时被声明**。而且这次「加载」是通过 active tool set 完成的，会像普通工具变更一样写进转录——所以 `/tree`、resume、fork 到该分支时，加载状态仍然有效。

> 一句话：**exposure 决定「默认可见性」，active set 决定「这次请求声明什么」，callable set 决定「脚本能调什么」。** 三者不是一回事。

---

## 15.3 Codemode：把工具调用写成程序

### 它解决什么问题

假设你要在 50 个 issue 里找出提到某个错误码的条目，再对命中的 5 个拉详情。传统方式是 50 次工具调用 + 50 条结果进上下文；codemode 的方式是一段脚本，只有最终 5 条摘要回到模型。

**中间结果不进上下文**，这是 codemode 的核心价值。

### 沙箱

脚本是**裸 JavaScript 源码**（不是 JSON，也不是 Markdown 代码块），作为 async 函数体在 QuickJS（编译为 WASM）中执行，因此支持顶层 `await` 和 `return`。

沙箱里没有：

- Node API；
- 文件系统；
- 网络；
- 定时器。

能接触外界的只有 `tools` 和 `models`。资源上限：

- **堆内存 256 MiB**（超限在脚本内抛 `InternalError: out of memory`）；
- **最多 2 个并发 worker**（共享于所有调用与会话，留一个给正在退出的 worker）；
- 输出默认上限 10000 token，超出保留首尾并把全文写入临时文件；
- 文本 + base64 图片总量超过 16 MiB 字符，或 `text()`/`image()`/`console` 调用超过 100000 次，脚本失败。

前两条是「别把宿主拖死」，后两条是「别把模型上下文撑爆」。

### 脚本接口

```js
// @options: {"max_output_tokens": 2000, "timeout_ms": 60000}
const issues = await tools.mcp__github__list_issues({ repo: "pi", state: "open" });
const hits = [];
for (const issue of issues) {
  if (issue.body.includes("E_CONN_RESET")) hits.push(issue);
}
const details = await Promise.all(hits.map((i) => tools.mcp__github__get_issue({ number: i.number })));
return details.map((d) => ({ title: d.title, url: d.url }));
```

选项行必须是**第一行**，出现在别处会在执行前直接失败。

可用的全局对象：

| 全局 | 作用 |
|---|---|
| `tools.<name>(args)` | 调用任意 callable 工具 |
| `text(value)` / `image(value)` / `return` | 产出输出项 |
| `console.log(...)` | 追加到 `<console_output>` |
| `exit()` | 成功结束 |
| `store(key, value)` / `load(key)` | 跨脚本保存小 JSON 状态 |
| `ALL_TOOLS` | 所有 callable 工具的 `{name, description}` |
| `searchTools(query, opts)` | BM25 排序，默认 8 条 |
| `describeTool(name)` / `describeNamespace(name)` | 取声明或命名空间说明 |
| `models` | 跑分类器与图像模型 |

工具名会转成合法 JS 标识符：`mcp__dev-radius__search` → `tools.mcp__dev_radius__search`。

返回值按工具有无 `outputSchema` 分流：有 schema 的返回结构化值（`bash` 返回 `{ output, truncated, exit_code, ... }`，其中 `output` 可达 1 MiB，而模型平时只看到 2000 行/50KB）；MCP 工具返回完整 `CallToolResult`；`read` 对图片返回可直接交给 `image()` 的 block。

调用失败、被拦截、参数非法都会 **reject**。要保留部分成功，用 `Promise.allSettled()`。

### Store 的边界语义

`store()` 只在脚本成功时落盘：每次成功写入会在会话里追加一条 `codemode-store` 自定义 entry。于是恢复会话时状态还在，而不同分支只看到自己路径上写入的值——**沙箱里的状态也遵守会话树语义**。

限制：单值 ≤ 256 KiB JSON，总量 ≤ 1 MiB；不要存图片。

### models：脚本里的非对话模型

`models.classify()` 与 `models.generateImages()` 用会话凭证调用分类器/图像模型：

- 每个脚本最多 **4 个并发**模型调用，超出的排队；
- 不抛 provider 错误，而是返回 `stopReason` / `errorMessage`；
- usage 会并入 codemode 的工具结果，计入会话成本。

一个细节：`codemode.mode` 为 `on` 时，已声明工具保持声明，只是描述里多一句「如何从脚本调用」；为 `only` 时，工具从模型视野消失，全部改为脚本调用。

---

## 15.4 `tool_search`：延迟加载的另一条路

`tool_search` 是一个可选工具，专门搜索**未声明**的工具（`codemode` 与 `deferred` exposure），并把命中的工具加载进 active 集合。

### 排序

它用的不是向量检索，而是 **BM25**（Okapi，`k1=1.2`, `b=0.75`），文档由以下内容拼成：

- 工具名，以及把 `_` 换成空格的名字；
- 描述；
- 参数 schema 里的描述与属性名（递归）；
- 所属 namespace 的名字、描述与 instructions。

分词会处理 camelCase、停用词与简单词形还原（`issues` → `issue`）。默认返回 8 条，并列时保持原顺序。

### 为什么 BM25 够用

工具名和描述是**高度结构化**的文本：`create_issue`、`search_docs`、`list_pull_requests`。在这种短文本、强命中的场景里，BM25 的性价比远高于 embedding：无需索引、无需额外模型、结果可解释、延迟几乎为零。

代码注释也留了口子：`ToolRanker` 是一个接口，未来可以替换成 hybrid ranker。

### 加载即转录

命中后，工具被加入 active set，下一次模型调用就会看到它们。因为 active set 的变化走的是与普通工具切换相同的转录机制，所以：

- `/tree` 回到加载前的节点，工具就不再声明；
- resume / fork 到加载后的节点，工具仍然可见。

**工具可见性成了会话状态的一部分**，而不是进程内的临时变量。

---

## 15.5 嵌套工具调用：脚本 → 工具 → 再脚本

codemode 脚本调用的工具，走的是与模型发起调用**完全相同**的流水线：

```text
prepareArguments → validate → beforeToolCall → execute → afterToolCall
```

这就是 `runToolCall()` 的存在意义（第 10 章）。`AgentSession` 通过 `NestedToolCallRunner` 给这些调用补上 `parentToolCallId`，并发出带该字段的 `tool_execution_*` 事件。

于是：

- 扩展的 `tool_call` 审批对脚本里的调用同样生效；
- UI 能把「脚本内部的调用」折叠显示在父调用之下；
- 嵌套调用本身也可以再调用工具（记录会递归收集）。

### 记录与限额

`nestedCalls` 会挂在**模型发起的那次工具调用**的结果消息上，包括 id、name、status、duration、error 与参数。限额（`NESTED_CALL_LIMITS`）：

| 限制 | 值 | 超出后 |
|---|---|---|
| 调用数 | 256 | 丢弃、标记不完整 |
| 单次参数 | 8 KiB | 只记录字节数，不记录参数 |
| 参数总量 | 32 KiB | 同上 |
| 错误文本 | 500 字符 | 截断 |

usage 会累加（包括被丢弃记录的调用），保证账本不漏。

**模型看不到这些中间记录**——它只看到脚本的最终输出。但会话文件、事件流和 UI 能看到，这就是「上下文里精简、审计上完整」。

---

## 15.6 取舍与借鉴

| 决策 | 收益 | 代价 |
|---|---|---|
| MCP 工具默认 `codemode` | 不占声明预算，鼓励脚本编排 | 模型不能直接调用，必须先写脚本或搜索 |
| `direct` 仅用于少量工具 | 首轮请求稳定、可缓存 | 需要人工判断哪些值得常驻 |
| 后台连接 + 按需等待 | 启动快 | 首次调用的延迟不可预测 |
| QuickJS 而非 Node `vm` | 真隔离、可限制内存 | 无 Node API，生态受限 |
| BM25 而非向量 | 零依赖、低延迟、可解释 | 语义泛化弱，换个说法可能搜不到 |
| 嵌套调用完整走钩子 | 安全策略无死角 | 每次调用多一层开销与记录 |
| 中间结果不进上下文 | 极大节省 token | 调试要去看事件/记录，而不是对话 |

如果你要为自己的 Agent 设计工具层，最值得抄的是这个分层：

1. **传输层**（MCP）只负责连接与协议；
2. **暴露层**（exposure/active）决定模型看见什么；
3. **编排层**（codemode）决定中间结果去哪；
4. **检索层**（tool_search）决定如何从大集合里捞出小集合。

四层各自可替换，且都不要求核心知道「工具有多少」。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/coding-agent/src/extensions/mcp/index.ts` | MCP 扩展：连接、工具注册、exposure、资源工具 |
| `packages/coding-agent/docs/mcp.md` | 配置、命名、生命周期、OAuth、exposure 全量说明 |
| `packages/coding-agent/src/extensions/codemode/tool.ts` | codemode 工具定义、可调用工具集合、声明生成 |
| `packages/coding-agent/src/extensions/codemode/execute.ts` | QuickJS 沙箱、输出限制、store、models、worker 限制 |
| `packages/coding-agent/src/extensions/tool-search/tool.ts` | BM25 排序、文档构建、`tool_search` 工具 |
| `packages/coding-agent/src/core/nested-tool-calls.ts` | 嵌套调用记录、限额、usage 汇总 |
| `packages/coding-agent/src/core/agent-session.ts` | `executeTool`、`parentToolCallId`、事件桥接 |
| `packages/coding-agent/docs/codemode.md` | codemode 脚本接口与语义 |

## 动手实验

1. 接一个带大量工具的 MCP server，分别设 `exposure: "direct"` 与 `"codemode"`，对比第一次请求的声明 token 数与模型选工具的准确率。
2. 写一个 codemode 脚本：调用 `searchTools("issue")` 找到工具，再用 `describeTool()` 看声明，最后批量调用并只返回摘要。打开事件流，确认中间调用都带 `parentToolCallId`。
3. 把 `tool_search` 的默认 limit 改成 3 或 20，观察「搜不到」与「声明膨胀」两种失败模式；再故意用同义词查询，体会 BM25 的语义边界。
