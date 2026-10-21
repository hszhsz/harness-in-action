# 第 10 章　pi 的「无权限」哲学：刻意不做什么

> 不内置审批不是忘了写，而是把「该不该拦」留给了最有资格判断的那一层。

读到这里，你已经见过 pi 的循环、钩子和工具流水线。现在我们来面对整套设计里最容易被误解的一点：

**pi 的核心没有权限系统。**

没有「这个命令危险吗」的规则表，没有「先问用户 Yes/No」的默认流程，没有把工作目录当成牢笼的路径检查。`bash` 工具拿到 `rm -rf` 就会真的执行，`write` 工具拿到 `~/.zshrc` 就会真的写。

但与此同时，你在仓库里又能看到 `project-trust.ts`、`security.md`、`permission-gate.ts`、`protected-paths.ts`、Gondolin、OpenShell……这些看起来都很像「权限」的东西。

这一章要做的，就是把这些概念摆回各自的位置：**核心为什么什么都不做、唯一的闸门长什么样、项目信任到底管什么、真正的安全边界又在哪里。**

---

## 10.1 核心的三条「不做」

先看 `pi-agent-core` 明确不做哪三件事。

### 不做一：不在循环里插入审批步骤

`runLoop` 的 turn 里没有「等待人类确认」这个状态。工具调用被解析、校验后，直接进入执行。整个 `packages/agent` 里没有 UI、没有 `confirm()`、没有「危险命令」字符串匹配。

审批如果需要存在，只能由调用方通过 `beforeToolCall` 注入。核心只定义这个钩子的形状：

```ts
export interface BeforeToolCallResult {
	block?: boolean;
	reason?: string;
	terminate?: boolean;
}
```

这是一份**结果契约**，不是一份策略。核心不知道什么该 block，它只负责「如果钩子说 block，我就把这次调用变成一条错误工具结果」。

### 不做二：不理解命令语义

pi 不会解析 `rm -rf /`，不会判断某个路径是否越界，不会识别 `curl | sh`。原因很实际：**命令语义是不可判定的**。`bash -c "$(cat script.sh)"`、别名、函数、编码后的 payload，任何基于字符串的规则都能被绕过。

既然做不到可靠，pi 干脆不做「假装可靠」的检查——这比给你一个能被绕过的安全错觉更诚实。

### 不做三：不把工作目录当沙箱

cwd 只决定资源发现的位置（`.pi/`、`.agents/skills`、`AGENTS.md`）和工具的默认根目录。它**不阻止** `bash` 访问 `../..`，不阻止 `read` 读取 `/etc/passwd`。真正的边界只能来自操作系统或虚拟机，见 10.4。

> 一句话总结：核心只提供「调用前可以被拦截」这个**机制**，不提供任何「什么该拦」的**策略**。

---

## 10.2 唯一的闸门：`beforeToolCall`

虽然是「无权限」，但 pi 并不是没有接缝。所有工具调用——包括模型直接发起的、以及工具内部嵌套调用的——都必须经过同一条流水线，闸门就在流水线的第三步。

`packages/agent/src/agent-loop.ts` 的 `prepareToolCall()` 按固定顺序做四件事：

```ts
const preparedToolCall = prepareToolCallArguments(tool, toolCall); // 1. 修复模型坏参数
const validatedArgs = validateToolArguments(tool, preparedToolCall); // 2. schema 校验
const beforeResult = await config.beforeToolCall({...});             // 3. 唯一闸门
// 4. 通过则执行；block 则生成错误结果
```

拦截的语义非常克制（[agent-loop.ts:745](file:///Users/heshaozhong/Code/pi/packages/agent/src/agent-loop.ts#L745)）：

```ts
if (beforeResult?.block) {
	const result = createErrorToolResult(beforeResult.reason || "Tool execution was blocked");
	if (beforeResult.terminate === true) {
		result.terminate = true;
	}
	return { kind: "immediate", result, isError: true };
}
```

注意三个细节：

1. **block 不是抛异常**。它变成一条 `isError: true` 的工具结果，照常回灌给模型。模型会看到「你被拦了，原因是 X」，然后自己决定换个做法还是停下来。这正是第 4 章讲的「错误即数据」。
2. **`terminate` 只是提示**。即使拦截了，也只有当整批工具结果都同意终止时循环才会停——单个 block 不会粗暴地掐断整个会话。
3. **钩子抛异常等于拦截**。扩展在 `tool_call` 里抛错，`AgentSession` 会把它转成「Extension failed, blocking execution」，调用不会被执行。fail-closed 发生在扩展这一层，而不是核心。

### 扩展如何接管这道闸门

`AgentSession` 把 `beforeToolCall` 接到了扩展总线上（[agent-session.ts:653](file:///Users/heshaozhong/Code/pi/packages/coding-agent/src/core/agent-session.ts#L653)）：

```ts
this.agent.beforeToolCall = (context) => this._beforeToolCall(context);
this.agent.afterToolCall = (context) => this._afterToolCall(context);
```

扩展的 `tool_call` 处理器按注册顺序执行，**第一个返回 `{ block: true }` 的处理器直接终止后续处理**（`runner.ts:1249`）。这让「多个策略叠加」有了确定的优先级：谁先注册谁先说话。

仓库里给了两个最小范例，可以直接拿来改成你自己的策略：

- `examples/extensions/permission-gate.ts`：匹配 `rm -rf`、`sudo`、`chmod 777`，有 UI 就弹窗，没 UI 就直接 block（非交互模式 fail-closed）。
- `examples/extensions/protected-paths.ts`：拦截对 `.env`、`.git/`、`node_modules/` 的 `write`/`edit`。

```ts
pi.on("tool_call", async (event, ctx) => {
	if (event.toolName !== "bash") return undefined;
	if (!isDangerous(event.input.command as string)) return undefined;
	if (!ctx.hasUI) return { block: true, reason: "Dangerous command blocked (no UI)" };
	const choice = await ctx.ui.select(`Allow?\n\n${event.input.command}`, ["Yes", "No"]);
	return choice === "Yes" ? undefined : { block: true, reason: "Blocked by user" };
});
```

这段代码就是 pi 全部的「内置权限系统」——它在 examples 里，不在 core 里。

### 嵌套调用也绕不过去

工具可以调用工具（codemode、MCP 资源工具、subagent 等）。pi 为此提供了 `runToolCall()`（[agent-loop.ts:811](file:///Users/heshaozhong/Code/pi/packages/agent/src/agent-loop.ts#L811)），它复用完全相同的 prepare → validate → before → execute → after 流程，只是不发事件、不落消息。`AgentSession` 还会给嵌套调用补上 `parentToolCallId`，让扩展知道「这一次调用是谁发起的」。

这意味着：**只要你的策略挂在 `tool_call` 上，它不会因为工具套了一层壳就失效。**

---

## 10.3 项目信任不是权限系统

很多人第一次看到 `trust.json` 和启动时的「Trust project folder?」提示，会以为那就是 pi 的权限系统。不是。它管的事情完全不同。

### 它解决的问题：启动时要不要加载这个目录里的代码

项目目录里可能藏着 `.pi/extensions/*.ts`——这些是**会直接在 pi 进程里执行的代码**。如果 clone 一个恶意仓库、进去执行 `pi`，扩展就被静默加载了，这比模型乱跑命令更危险，因为它发生在你看到任何界面之前。

所以 pi 在启动阶段加了一道闸：检测到项目里有「需要信任的资源」时，先问一句。

需要信任的资源清单（`trust-manager.ts:30`）：

```ts
const TRUST_REQUIRING_PROJECT_CONFIG_RESOURCES = [
	"settings.json", "mcp.json", "extensions", "skills",
	"prompts", "themes", "SYSTEM.md", "APPEND_SYSTEM.md",
];
```

外加当前目录或祖先目录下的 `.agents/skills`（用户级的 `~/.agents/skills` 除外）。

### 决策顺序

`resolveProjectTrusted()`（[project-trust.ts:47](file:///Users/heshaozhong/Code/pi/packages/coding-agent/src/core/project-trust.ts#L47)）按固定优先级决策：

```ts
if (options.trustOverride !== undefined) return options.trustOverride;   // 1. --approve / --no-approve
if (!hasTrustRequiringProjectResources(cwd)) return true;                // 2. 没有受保护资源，直接放行
// 3. 扩展的 project_trust 事件，第一个给出 yes/no 的说了算
// 4. ~/.pi/agent/trust.json 里最近的已保存决策（含父目录）
// 5. defaultProjectTrust：always / never / ask（默认 ask）
// 6. 有 UI 则弹窗；无 UI 则 false
```

几个容易忽略的点：

- **没有受保护资源就完全跳过询问**。一个空的 `.pi` 目录不触发信任流程。
- **保存的决策会向上查找父目录**。在 monorepo 根信任一次，子目录都继承；子目录也可以单独覆盖。
- **非交互模式（print / JSON / RPC）没有弹窗**。`defaultProjectTrust: "ask"` 在无 UI 时等同于拒绝。自动化里要显式传 `--approve` 或设为 `always`。
- **信任只影响「加载什么」，不影响「执行什么」**。信任之后，工具仍然以 pi 进程的操作系统权限运行。

### 它明确不承诺的事

官方文档把话说得很直（`docs/security.md`）：

> Project trust controls which project resources load at startup, but it does not make that content or the resulting actions safe. … Watching the transcript, using project trust, and reviewing changes do not create a security boundary.

还有两个刻意留下的缺口：

- **`sessionDir` 在信任决策之前就被读取**。因为要先知道会话存哪，才能选/建会话。拒绝信任也无法撤销这次读取。
- **`AGENTS.md` / `CLAUDE.md` / `AGENTS.override.md` 无论信任与否都会加载**。它们只是文本，但文本可以影响模型——这属于 prompt injection 的范畴，不是启动信任能解决的。

所以正确的理解是：**项目信任是一道「启动期代码加载闸」，不是「运行期行为闸」。** 前者防的是「还没开始对话就被执行」，后者才是权限系统要防的「对话过程中做坏事」。pi 把后者完全外包了。

---

## 10.4 真正的边界：操作系统、容器、虚拟机

既然核心不拦，那安全从哪里来？pi 的答案只有一句：**来自 pi 进程之外。**

`docs/security.md` 给了一张非常清晰的表：

| 运行方式 | 被保护的是什么 | 局限 |
|---|---|---|
| 直接用当前用户权限跑 | 该用户无权访问的一切 | 仍与同机其他进程共享文件与网络 |
| 整个 pi 跑在容器 / VM / 沙箱里 | 未挂载进环境的主机文件与进程 | 挂进去的凭证、网络仍然可达 |
| pi 在主机，只有内置工具进隔离环境 | 通过这些工具访问的主机资源 | pi 自身和未走隔离后端的扩展仍在边界外 |

`docs/containerization.md` 进一步把主流方案摊开：

| 方案 | pi 在哪跑 | 隔离了什么 | 凭证处理 |
|---|---|---|---|
| Plain Docker | 容器 | pi、内置工具、`!` 命令、扩展 | 凭证传入容器 |
| Docker Sandboxes | 托管沙箱 | 同上 | 真实 key 留在主机，由代理替换 |
| OpenShell | 本地/远程沙箱 | pi 全进程 | 策略化凭证与推理路由 |
| Gondolin 扩展 | 主机 | 只有内置工具与 `!` 命令 | pi 的凭证留在主机，但命令继承主机环境变量 |

这张表最重要的信息不是「用哪个」，而是**边界画在哪里**：

- 整个进程进容器，扩展也进容器，边界最完整；
- 只把内置工具路由进 Gondolin 微 VM，pi 进程和**其他扩展工具**仍在主机——如果某个扩展自己实现了 `bash`，它就绕过了 VM。文档专门警告了这一点。

换句话说：**沙箱的有效性取决于你是否把所有执行路径都关进去，而不是取决于你装了什么插件。**

---

## 10.5 代价、收益，以及你该怎么补

### 这笔交易换来了什么

| 收益 | 具体表现 |
|---|---|
| 核心极小 | 循环里没有策略分支，行为可预测、易测试 |
| 策略可组合 | 审批、路径保护、审计、限流都能以扩展形式叠加，互不绑架 |
| 不制造安全幻觉 | 不会让你因为「它拦了 rm -rf」就以为它能拦住所有危险 |
| 适配任何环境 | 同一内核可以裸跑、进容器、走远程、嵌进 CI |

### 代价是什么

| 代价 | 具体表现 |
|---|---|
| 默认不安全 | 首次运行没有任何确认，模型让干啥就干啥 |
| 责任转移 | 使用者必须自己想清楚「边界放在哪一层」 |
| 策略质量参差 | 字符串匹配的 permission-gate 很容易被绕过，而它看起来像安全措施 |
| 非交互场景易踩坑 | `defaultProjectTrust: "ask"` 在无 UI 时静默拒绝；扩展在无 UI 时若忘了 fail-closed，就等于放行 |

### 一个务实的加约束顺序

如果你要把 pi 用在真实项目里，建议按这个顺序加，而不是一上来写规则：

1. **先画进程边界**：能不能整个跑进容器/VM？能，就先做这个——它比任何字符串规则都可靠。
2. **再收紧凭证**：短生命周期、最小 scope、别把 `~/.pi/agent` 挂进容器。
3. **然后加行为策略**：用 `tool_call` 拦真正不可逆的操作（推远端、删数据、写生产配置），并且**无 UI 时默认 block**。
4. **最后加可观测**：把 `tool_execution_start/end`、`tool_result` 记进审计日志——事后能查，比事前猜更现实。

这也解释了为什么 pi 把 `permission-gate.ts` 放在 examples 而不是 core：**它示范的是「怎么写策略」，而不是「策略应该是什么」。** 你的项目该拦什么，只有你知道。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/agent/src/types.ts` | `BeforeToolCallResult` / `BeforeToolCallContext` 契约 |
| `packages/agent/src/agent-loop.ts` | `prepareToolCall()`：唯一闸门；`runToolCall()`：嵌套调用复用同一流水线 |
| `packages/coding-agent/src/core/agent-session.ts` | 把 `beforeToolCall` 接到扩展 `tool_call` 总线 |
| `packages/coding-agent/src/core/extensions/runner.ts` | `emitToolCall()`：首个 block 终止后续处理器 |
| `packages/coding-agent/src/core/project-trust.ts` | 启动期信任决策的完整优先级 |
| `packages/coding-agent/src/core/trust-manager.ts` | 受保护资源清单、`trust.json` 读写与父目录继承 |
| `packages/coding-agent/docs/security.md` | 官方安全模型：信任不是边界 |
| `packages/coding-agent/docs/containerization.md` | Docker / Sandboxes / OpenShell / Gondolin 的边界对比 |
| `packages/coding-agent/examples/extensions/permission-gate.ts` | 最小审批策略 |
| `packages/coding-agent/examples/extensions/protected-paths.ts` | 最小路径保护策略 |

## 动手实验

1. 写一个 `tool_call` 扩展，要求：`bash` 中出现 `git push` 时，有 UI 就确认，无 UI 就 block，并在 block 的 `reason` 里写清楚「为什么拦、该怎么改」。跑一次 `pi -p "..."`（无 UI），确认模型收到的是错误工具结果而不是会话中断。
2. 在一个含 `.pi/extensions/hello.ts` 的临时目录里分别执行 `pi --no-approve` 和 `pi --approve`，观察扩展是否加载；再删掉 `.pi` 下所有受保护资源，确认不再出现信任提示。
3. 用 Gondolin 示例把内置工具路由进微 VM，然后写一个自定义 `bash` 扩展（不复用 `BashOperations`），思考：它是否绕过了边界？这验证了 10.4 的哪一句话？
