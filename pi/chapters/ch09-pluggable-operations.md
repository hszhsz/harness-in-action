# 第 9 章　可插拔执行后端：BashOperations 与文件变更队列

> 工具描述的是「语义」，Operations 描述的是「在哪儿、用什么方式落地」。

第 8 章讲了工具的接口。这一章往下看一层：同一个 `bash` 工具，如何在本地 shell、容器、远端 VM 上执行？同一个 `write`，如何避免并发写坏文件？答案不在工具实现里，而在 **Operations** 这一层抽象。

---

## 9.1 每个工具都有自己的一组 Operations

pi 不是只给 bash 做了抽象。八个内置工具各自定义了最小的后端契约：

| 工具 | Operations 接口 | 关键方法 |
|---|---|---|
| `read` | `ReadOperations` | `readFile` / `access` / `detectImageMimeType` |
| `edit` | `EditOperations` | `readFile` / `writeFile` / `access` |
| `write` | `WriteOperations` | `writeFile` / `mkdir` |
| `grep` | `GrepOperations` | `isDirectory` / `readFile` |
| `find` | `FindOperations` | `exists` / `glob` |
| `ls` | `LsOperations` | `exists` / `stat` / `readdir` |
| `bash` | `BashOperations` | `exec` |
| `powershell` | `PowerShellOperations` | 等价于 `BashOperations` |

这些接口的共同特点：

- **只暴露语义所需的最小方法**，不泄漏`fs` 的完整能力；
- 方法是可替换的（构造工具时传 `options.operations`）；
- 默认实现直接用 `node:fs` 或 `child_process`；
- 远端实现（如 Gondolin、SSH）只需实现这些方法，就能复用整个工具逻辑。

以 `read` 为例：

```ts
export interface ReadOperations {
	readFile: (absolutePath: string) => Promise<Buffer>;
	access: (absolutePath: string) => Promise<void>;
	detectImageMimeType?: (absolutePath: string) => Promise<string | null | undefined>;
}
```

一个微 VM 后端只要实现「读文件 + 检查可读 + 探测图片类型」，就自动获得 `read` 工具的分页、截断、图片处理、错误信息等全部行为。**这就是把「策略」和「传输」分开的收益。**

## 9.2 `BashOperations`：一个 `exec` 撑起的后端

```ts
export interface BashOperations {
	exec: (
		command: string,
		cwd: string,
		options: {
			onData: (data: Buffer) => void;
			signal?: AbortSignal;
			timeout?: number;
			env?: NodeJS.ProcessEnv;
		},
	) => Promise<{ exitCode: number | null }>;
}
```

契约里有几个重要约定：

1. **输出必须流式回调 `onData`**。工具层用 `OutputAccumulator` 实时累积，UI 才能边跑边显示。
2. **`exitCode` 为 `null` 视为失败**。被信号终止时应报告 `128 + signalNumber`（shell 惯例）。
3. **`timeout` 的单位是秒**，由工具 schema 传入。本地实现对上限有硬约束：`MAX_TIMEOUT_MS = 2_147_483_647`（Node 定时器上限），超过直接报错。
4. **必须尊重 `signal`**。中止时要杀掉整个进程树，而不是只杀父进程。

这个接口小到可以在 20 行内实现，却足以覆盖本地 shell、容器 `exec`、SSH 通道、微 VM。

## 9.3 本地实现：进程组、超时与清理

`createLocalShellOperations(shellName, resolveShellConfig)` 是 bash 与 powershell 共用的执行核心。关键细节：

```ts
const child = spawn(shellConfig.shell, commandFromStdin ? shellConfig.args : [...shellConfig.args, command], {
	cwd,
	detached: process.platform !== "win32",
	env: env ?? getShellEnv(),
	stdio: [commandFromStdin ? "pipe" : "ignore", "pipe", "pipe"],
	windowsHide: true,
});
```

- **`detached: true`（非 Windows）**：让子进程成为进程组组长。这样 `killProcessTree(pid)` 能一次杀掉 shell 及其所有后代——否则 `bash -c "npm test"` 里的子进程会成为孤儿。
- **命令传输方式可配置**：有的 shell 通过 argv 传命令，有的通过 stdin（避免超长 argv 与转义问题）。
- **超时与中止都走 `killProcessTree`**：不是 `child.kill()`，而是整棵树。
- **进程退出后清理**：解除定时器、移除 abort 监听、关闭流，避免 Node 进程被句柄拖住。

还有一个容易忽略的产品化细节：bash 工具会向子进程注入 `PI_*` 环境变量：

```
PI_SESSION_ID / PI_SESSION_FILE / PI_PROVIDER / PI_MODEL / PI_REASONING_LEVEL
```

并在每次执行前删除旧值，避免从父环境泄漏。这样用户在脚本里就能知道「当前是哪个会话、哪个模型在跑我」，实现可观测的自动化。

## 9.4 输出硬化：既不撑爆上下文，也不丢证据

命令输出是不可信的：`npm install` 可能刷出几十万行，`cat` 一个二进制文件可能直接让终端崩掉。pi 用 `OutputAccumulator` 做三层处理：

1. **保留尾部窗口**：默认最多 2000 行或 50KB（`DEFAULT_MAX_LINES` / `DEFAULT_MAX_BYTES`），以先到者为准；
2. **落盘完整输出**：超出窗口的内容写入临时文件（`pi-bash-*`），结果里给出路径；
3. **结构化结果限流**：`structuredContent.output` 最多 1MB（`STRUCTURED_OUTPUT_MAX_BYTES`），超出则标记 `truncated` 并给出完整文件路径。

模型看到的是「尾部 + 截断说明」，而人类/程序仍能通过临时文件拿到完整输出。这比「直接截断、不告诉任何人」可靠得多。

`truncate.ts` 还提供了四种截断策略：

| 函数 | 适用 |
|---|---|
| `truncateHead` | 保留开头（如文件头部） |
| `truncateTail` | 保留结尾（如命令输出） |
| `truncateMiddle` | 保留首尾（如长文件预览） |
| `truncateLine` | 单行超长（grep 匹配行最多 500 字符） |

## 9.5 文件写入：按真实路径串行化

并行工具执行带来一个现实问题：两个 `edit` 同时改同一个文件，后写者会覆盖先写者。pi 的解法是 `withFileMutationQueue(path, fn)`：

```ts
export async function withFileMutationQueue<T>(filePath: string, fn: () => Promise<T>): Promise<T> {
	const registration = registrationQueue.then(async () => {
		const key = await getMutationQueueKey(filePath);
		const currentQueue = fileMutationQueues.get(key) ?? Promise.resolve();
		// 把本次操作挂到该 key 的 promise 链尾部
		// ...
	});
	registrationQueue = registration.then(() => undefined, () => undefined);

	const { key, currentQueue, chainedQueue, releaseNext } = await registration;
	await currentQueue;
	try {
		return await fn();
	} finally {
		releaseNext();
		if (fileMutationQueues.get(key) === chainedQueue) fileMutationQueues.delete(key);
	}
}
```

三个设计点：

- **key 用 `realpath`**：符号链接指向同一文件时，仍然共享同一把锁。
- **不存在的文件用 `resolve` 后的路径**：新建文件还没有 realpath，但同一路径的创建也应串行。
- **`registrationQueue` 串行化「建链」过程**：否则两个并发调用可能同时读到同一个 `currentQueue`，形成竞态。

不同文件之间仍然并行——这是「细粒度锁」而不是「全局写锁」。

## 9.6 把后端换掉：容器、微 VM、远端

因为 Operations 只是普通对象，替换后端不需要改工具代码：

```ts
const bash = createBashTool(cwd, {
	operations: {
		exec: async (command, cwd, { onData, signal, timeout }) => {
			// 转发到容器 / SSH / 微 VM
			return await myRemoteExec(command, { cwd, onData, signal, timeout });
		},
	},
});
```

pi 官方文档 `docs/containerization.md` 给出四种隔离路径：

| 方式 | 边界 | 特点 |
|---|---|---|
| Plain Docker | 整个 Pi 进程 | 最简单，凭据需传入容器 |
| Docker Sandboxes | 托管沙箱 | 代理持有真实凭据，容器内只看到替代值 |
| OpenShell | 本地/远程沙箱 | 文件、进程、网络、凭据策略 |
| Gondolin 扩展 | 微 VM | Pi 留在宿主机，内置工具与 `!` 命令进 VM |

它们对应不同的信任模型：**你是在隔离「模型生成的命令」，还是在隔离「整个 Agent 进程」？** 这个问题的答案决定了凭据放哪、扩展在哪运行、文件如何共享。第 10 章会把这条思路扩展到「权限该放在哪一层」。

## 9.7 Operations 的边界：不该放什么

Operations 只负责「怎么执行」，不应该承担：

- **权限判断**：那是 `beforeToolCall` / 扩展的职责；
- **参数校验**：那是 schema 与 `prepareArguments` 的职责；
- **结果格式化**：那是工具实现与 `content` / `structuredContent` 的职责；
- **重试策略**：那是循环或 provider 层的职责。

把这些混进 Operations，会导致「换后端就换了一套安全策略」——这是最危险的一种耦合。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/coding-agent/src/core/tools/bash.ts` | `BashOperations`、本地 shell 实现、超时与进程树 |
| `packages/coding-agent/src/core/tools/powershell.ts` | Windows 后端（复用 `BashOperations` 契约） |
| `packages/coding-agent/src/core/tools/output-accumulator.ts` | 尾部窗口、临时文件、完整输出 |
| `packages/coding-agent/src/core/tools/truncate.ts` | 四种截断策略与默认上限 |
| `packages/coding-agent/src/core/tools/file-mutation-queue.ts` | 按 realpath 串行化写入 |
| `packages/coding-agent/src/core/tools/{read,edit,write,grep,find,ls}.ts` | 各自的 Operations 契约 |
| `packages/coding-agent/docs/containerization.md` | Docker / OpenShell / Gondolin 接入 |
| `packages/coding-agent/examples/extensions/gondolin/` | 微 VM 后端的真实样例 |

## 动手实验

1. 实现一个「内存文件系统」的 `WriteOperations` 与 `ReadOperations`，用它们构造 `write` / `read` 工具，跑一轮「写文件再读回来」的 Agent 循环。全程不碰磁盘。
2. 用 `withFileMutationQueue` 并发写同一个文件 50 次，每次写入递增数字，验证最终内容没有丢失更新。再用不同路径（含符号链接）测试 key 的归一化。
3. 跑一个输出 10 万行的命令，检查模型看到的内容、`details.truncation`、`fullOutputPath` 三者是否一致。
4. 阅读 `docs/containerization.md` 的四种方式，为你的场景选择一种，并写出「凭据放在哪、扩展在哪运行、文件如何共享」三行答案。
