# 第 9 章　可插拔的 Operations：让「动手那一刻」可替换、可远程、可加固

> 一个工具最危险的，是它「真正动手」的那一行——`spawn` 一个进程、写一块磁盘。pi 把这一行单独抽成一个接口，于是它能被换成 SSH、被换成容器、被换成一个假实现。

第 8 章我们把工具拆到了 `execute`，但 `execute` 内部「真正动手」的那一刻——`bash` 怎么 spawn 进程、`write` 怎么落盘——还是黑盒。这一章我们打开最后一层：`Operations` 抽象。它是 pi「约束外包」哲学在工具层最锋利的体现——**工具的「做什么」与「在哪做、怎么做」被彻底分开，后者变成一个可注入的接口。**

---

## 9.1 `BashOperations`：把「执行命令」抽成一个接口

看 `bash` 工具的核心抽象（`core/tools/bash.ts`）。pi 没有把 `spawn` 直接写死在 `execute` 里，而是先定义了一个接口：

```ts
/**
 * Pluggable operations for the bash tool.
 * Override these to delegate command execution to remote systems (for example SSH).
 */
export interface BashOperations {
	exec: (
		command: string,
		cwd: string,
		options: {
			onData: (data: Buffer) => void;   // 流式输出回调
			signal?: AbortSignal;             // 中止
			timeout?: number;                 // 超时
			env?: NodeJS.ProcessEnv;          // 环境变量
		},
	) => Promise<{ exitCode: number | null }>;
}
```

注释一句话点破了它存在的全部理由：**「Override these to delegate command execution to remote systems (for example SSH)」**——重写它，就能把命令执行委托给远程系统。`bash` 工具本身只依赖这个接口，不关心命令究竟是在本地 shell、SSH 到的远程机器、还是一个容器里跑的。

这就是「机制与策略分离」在工具最底层的落地：`bash` 工具的逻辑（解析参数、流式汇报、截断输出、渲染结果）是**机制**，而「命令到底在哪个执行后端跑」是**策略**，被 `BashOperations` 这个注入点接管。

## 9.2 `createLocalBashOperations`：默认的本地实现里藏着多少细节

pi 自带一个本地实现 `createLocalBashOperations`，它是 `BashOperations` 的默认值。别看接口只有一个 `exec`，真要把「在本地跑一条命令」做对、做安全，里面的细节多得惊人：

```ts
export function createLocalBashOperations(options?): BashOperations {
	return {
		exec: async (command, cwd, { onData, signal, timeout, env }) => {
			const { shell, args } = getShellConfig(options?.shellPath);
			// 1. 先确认 cwd 存在，否则给一条清晰的错误
			try { await fsAccess(cwd, constants.F_OK); }
			catch { throw new Error(`Working directory does not exist: ${cwd}...`); }
			// 2. 已经被 abort 就别启动了
			if (signal?.aborted) throw new Error("aborted");

			const child = spawn(shell, [...args, command], {
				cwd,
				detached: process.platform !== "win32",   // 关键：开新进程组
				env: env ?? getShellEnv(),
				stdio: ["ignore", "pipe", "pipe"],
				windowsHide: true,
			});
			if (child.pid) trackDetachedChildPid(child.pid);
			// ...
		},
	};
}
```

逐个看这些「魔鬼细节」：

- **`detached: true` + `killProcessTree`**：命令在非 Windows 上以 `detached` 方式 spawn，开一个独立的**进程组**。为什么？因为一条 `bash` 命令可能 fork 出一堆子进程（`npm test` 拉起的 worker、`&` 后台进程……）。如果只 kill 父进程，子进程会变成「孤儿」继续跑、继续占资源。pi 在超时和 abort 时调用的是 `killProcessTree(child.pid)`——**杀整棵进程树**，斩草除根：

  ```ts
  const onAbort = () => { if (child.pid) killProcessTree(child.pid); };
  if (signal) {
      if (signal.aborted) onAbort();
      else signal.addEventListener("abort", onAbort, { once: true });
  }
  ```

- **超时即杀树**：`timeout` 到了，标记 `timedOut = true` 并同样 `killProcessTree`，最后抛 `timeout:${timeout}` 让上层知道是超时而非正常退出：

  ```ts
  timeoutHandle = setTimeout(() => {
      timedOut = true;
      if (child.pid) killProcessTree(child.pid);
  }, timeout * 1000);
  ```

- **`trackDetachedChildPid` / `untrackDetachedChildPid`**：pi 全局登记所有 detached 子进程的 pid，进程退出时注销。这样即便主程序要退出，也能找到并清理掉所有还在跑的后台命令，不留僵尸。
- **`waitForChildProcess`**：注释解释了一个微妙的坑——detached 的后代进程可能继续持有继承来的 stdio 句柄，导致「进程已退出但流不关闭、`await` 一直挂着」。pi 用专门的 `waitForChildProcess` 来正确等待退出，避免 hang。
- **`stdio: ["ignore", "pipe", "pipe"]` + `onData`**：stdin 忽略（Agent 跑的命令不该等交互输入），stdout/stderr 都接到 `onData` 流式回调——这正是第 8 章 `execute` 的 `onUpdate` 能实时回显命令输出的源头。

> **给 Agent 开发者的启示**：「让 Agent 能跑 shell 命令」听起来是一行 `spawn`，但要做到**能中止、能超时、能杀干净子孙进程、不 hang、不留僵尸**，是一整套工程。pi 把这套复杂度封进 `createLocalBashOperations`，对外只露一个干净的 `exec` 接口——这才是「可插拔」有价值的前提：默认实现得足够好，替换实现才有意义。

## 9.3 为什么「可替换的执行后端」是 pi 的命脉

回到第 1、2 章反复强调的 pi 立场：**核心不内置权限系统，把约束外包给容器和可替换的执行后端。** `BashOperations` 就是这句话里「可替换的执行后端」的技术原型。

把 `exec` 抽成接口，意味着同一个 `bash` 工具可以接不同的「后端」：

| 后端实现 | 效果 | 用途 |
|---|---|---|
| `createLocalBashOperations`（默认） | 在本地 shell 跑 | 开发者本机直接用 |
| 一个 SSH 实现 | 命令在远程机器跑 | 远程开发、跑在算力更强的机器上 |
| 一个容器实现 | 命令在 Docker/沙箱里跑 | **安全隔离**——这是 pi「约束外包」的核心姿态 |
| 一个 faux/mock 实现 | 返回预设输出，不真跑 | 测试整个工具链不产生副作用 |

注意第三行：pi 不在核心里写「禁止 `rm -rf`」这类规则，而是说——你担心安全？把 `BashOperations` 换成一个「在容器里跑」的实现，命令就被关进沙箱了，能不能 `rm -rf` 由容器的挂载和权限说了算，跟 pi 核心无关。**这就是「约束外包」：不在代码里做策略判断，而是把执行环境本身变成可替换的边界。** 第 10 章我们会专门展开这套哲学的代价与收益。

`createLocalBashOperations` 的注释还点了一个巧妙用法：扩展可以拦截 `user_bash` 事件（用户用 `!` 前缀直接跑的命令），在**重写或包装命令之后**，仍然复用 pi 标准的本地执行行为——因为本地后端是个公开的、可被组合的函数，而不是藏在工具内部的私货。

## 9.4 `withFileMutationQueue`：同文件写操作的串行化

`bash` 解决的是「进程执行」的可插拔；而 `write`/`edit` 这类**改文件**的工具，面临的是另一个经典并发问题：**两个工具调用同时改同一个文件，会互相覆盖、产生竞态。**

第 3 章讲过一道防线——工具可以声明 `executionMode: "sequential"` 让整批串行。但这有点「一刀切」：只要有一个写工具，整批（哪怕其余是无关的读操作）都得排队，太浪费并行度。pi 还有一道更精细的防线：`withFileMutationQueue`（`core/tools/file-mutation-queue.ts`），**按文件粒度**串行化写操作——同一个文件的修改排队，不同文件的修改照样并行：

```ts
/**
 * Serialize file mutation operations targeting the same file.
 * Operations for different files still run in parallel.
 */
export async function withFileMutationQueue<T>(filePath: string, fn: () => Promise<T>): Promise<T> {
	// ...按 key 维护一条 Promise 链，同 key 的 fn 依次执行
}
```

它的精巧在两处：

**1. 用「真实路径」做队列 key，而非字符串路径。** 它先 `realpath` 解析路径再当 key：

```ts
async function getMutationQueueKey(filePath: string): Promise<string> {
	const resolvedPath = resolve(filePath);
	try {
		return await realpath(resolvedPath);   // 解析符号链接、规范化
	} catch (error) {
		if (isMissingPathError(error)) return resolvedPath;  // 文件还不存在（如新建）也要能排队
		throw error;
	}
}
```

为什么非要 `realpath`？因为 `./a.txt`、`a.txt`、一个指向同一文件的符号链接，是同一个物理文件的三种写法。只有解析成真实路径，才能保证「逻辑上的同一文件」落到同一个队列 key，真正避免竞态。而文件还不存在时（比如 `write` 新建），`realpath` 会失败，此时退回用规范化后的绝对路径当 key——新建也得排队，否则两个并发的「创建同一文件」也会打架。

**2. 用 Promise 链实现无锁队列。** 每个 key 在 `fileMutationQueues` 这张 Map 里维护一条 Promise 链：新任务 `.then` 到当前链尾，执行完 `releaseNext()` 释放下一个。同 key 的任务严格依次执行，不同 key 的链彼此独立、天然并行。还有一个 `registrationQueue` 串行化「注册」这一步本身，避免两个任务同时读改 Map 造成 race。完成后若链尾还是自己就从 Map 删除 key，避免内存泄漏。

> 这一节和第 3 章的 `executionMode`、第 5 章 `AgentHarness` 的 phase 机串起来看，能看到 pi 处理「互斥」的一以贯之的审美：**不用锁、不用信号量，全用 Promise 链 / 状态字段 / 队列**这些 JS 原生、无死锁风险的手段，在不同粒度上（会话级、工具批级、单文件级）各自串行化。

至此，工具的三层我们全拆完了：第 8 章的「工具是什么」（AgentTool/ToolDefinition）、本章的「工具在哪做、怎么做安全」（Operations + 文件队列）。一个有意思的事实是——到这里为止，pi 核心**从未出现过「这个命令危险、拒绝执行」之类的判断**。它凭什么敢这样?下一章，我们正面回答 pi 最有争议、也最值得玩味的设计:「无权限」哲学。

---

## 本章小结

- `BashOperations` 把「执行命令」抽成一个只有 `exec` 的接口，工具只依赖接口，不关心命令在本地 / SSH / 容器 / mock 哪个后端跑——「机制与策略分离」在工具最底层的落地。
- 默认实现 `createLocalBashOperations` 看似简单，实则封装了一整套工程：`detached` 进程组 + `killProcessTree` 杀整棵树、超时杀树、`track/untrackDetachedChildPid` 防僵尸、`waitForChildProcess` 防 hang、stdio 流式回调。
- 可替换的执行后端是 pi「约束外包」的技术原型：要安全就把后端换成容器实现，让沙箱而非核心代码来约束命令——核心不做策略判断。
- `withFileMutationQueue` 按「真实路径（`realpath`）」为 key、用 Promise 链无锁串行化同文件写操作，不同文件仍并行；比 `executionMode: "sequential"` 的「整批串行」精细得多。
- pi 处理互斥的统一审美：不用锁，全用 Promise 链 / 状态字段 / 队列，在会话级、工具批级、单文件级各自串行化。

下一章，正面解读 pi 最有争议的设计——「无权限」哲学：核心为什么不内置准入控制，这样做的代价与收益各是什么。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/coding-agent/src/core/tools/bash.ts` | `BashOperations` 接口、`createLocalBashOperations` 默认实现 |
| `packages/coding-agent/src/utils/shell.ts` | `getShellConfig`/`getShellEnv`/`killProcessTree`/`track*DetachedChildPid` |
| `packages/coding-agent/src/utils/child-process.ts` | `waitForChildProcess`（正确等待 detached 进程退出） |
| `packages/coding-agent/src/core/tools/file-mutation-queue.ts` | `withFileMutationQueue`：按真实路径串行化同文件写操作 |

## 动手实验

1. 读 `createLocalBashOperations`，找出「超时」和「用户 abort」两条路径最终都汇聚到哪个函数（提示：`killProcessTree`）。再想：如果把 `detached` 去掉、只 kill `child.pid`，跑一个 `sleep 100 &` 会发生什么？
2. 设计一个最小的「远程 `BashOperations`」接口实现（伪代码即可）：`exec` 通过 SSH 把 `command` 发到远程、把远程 stdout 通过 `onData` 流回来。对照 9.3 的表格，说明为什么 `bash` 工具一行都不用改就能用上它。
3. 用 `withFileMutationQueue` 同时发起「对 `a.txt` 的两次写」和「对 `b.txt` 的一次写」，加日志观察执行顺序：验证 a 的两次严格串行、而 b 与 a 并行。再把其中一个路径换成指向 `a.txt` 的符号链接，验证 `realpath` 让它仍然和 a 排进同一队列。
