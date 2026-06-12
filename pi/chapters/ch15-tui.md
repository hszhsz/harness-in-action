# 第 15 章　自研 TUI：不用 React 也能做差分渲染

> 一个 Agent CLI 最终要被人「看见」。pi 没有用 ink、没有用 React，而是从零写了一个 `pi-tui`——一个把组件渲染成 `string[]`、靠逐行比对做差分刷新的极简终端 UI 库。这一章看它为什么这么选,以及一个「不发明虚拟 DOM」的渲染器能做到什么程度。

前面十四章,我们一路从 `streamSimple` 的模型契约,走到双层 `runLoop`、`AgentHarness` 编排、JSONL 会话树、压缩、工具系统、扩展与 Skill。所有这些都是「内核」——它们不关心屏幕上长什么样。但 pi 作为一个交互式 CLI,终究要把流式输出、工具调用、可折叠的 skill 块画到终端上,还要接收键盘输入、支持中文输入法、在窗口缩放时不闪屏。这一层就是 `packages/tui`(`@earendil-works/pi-tui`)。

这一章的主角是一个反直觉的工程决策:**pi 放着成熟的 ink/React 终端框架不用,自研了一套渲染器。** 我们会看清它的核心——把一切组件抽象成「渲染成字符串数组」的纯函数式接口,然后用「逐行 diff + 同步输出」把变化最小化地写到终端。

---

## 15.1 为什么不用 ink/React：把组件压成 `string[]`

ink 是 React 在终端的实现:你写 JSX,它维护一棵虚拟 DOM,用 reconciler 做 diff,再翻译成终端转义序列。强大,但也带来一整套 React 运行时、调和算法和生命周期的复杂度——对一个 Agent CLI 来说,这是「为了画几行文字,扛进来一整个 UI 框架」。

pi 的选择极端朴素。看 `tui.ts` 里的 `Component` 接口:

```ts
export interface Component {
	/**
	 * Render the component to lines for the given viewport width
	 * @param width - Current viewport width
	 * @returns Array of strings, each representing a line
	 */
	render(width: number): string[];

	handleInput?(data: string): void;
	wantsKeyRelease?: boolean;
	invalidate(): void;
}
```

一个组件的本质,就是 **「给我一个宽度,我吐给你一个 `string[]`,每个元素是一行(可含 ANSI 转义)」**。没有虚拟 DOM、没有 fiber、没有 reconciler。组件树就是普通对象树;渲染就是从根 `render(width)` 递归向下,把子组件的行拼起来。`Box` 组件的 `render` 就是最朴实的写照:

```ts
render(width: number): string[] {
	const contentWidth = Math.max(1, width - this.paddingX * 2);
	const leftPad = " ".repeat(this.paddingX);
	const childLines: string[] = [];
	for (const child of this.children) {
		const lines = child.render(contentWidth);
		for (const line of lines) {
			childLines.push(leftPad + line);
		}
	}
	// ……再套上背景色与上下 padding
}
```

> **给 Agent 开发者的启示**:终端是个「按行、按列」的二维字符网格,它的「DOM」天然就是 `string[]`。React 的虚拟 DOM 是为浏览器那种「任意嵌套、任意属性」的树设计的——搬到终端上是杀鸡用牛刀。pi 的洞察是:**终端 UI 的最小充分抽象就是 `(width) => string[]`**。一旦认清这一点,整个 UI 库的复杂度就坍缩成「怎么把新旧两个 `string[]` 的差异高效写到终端」这一个问题。少一层抽象,就少一整类 bug 和一整个依赖。

每个组件自带一个 `invalidate()`——当主题变了、或者需要从头重渲时调用,用来清掉内部缓存(`Box` 就缓存了上次的 `childLines` 和渲染结果,宽度/背景/子行都没变时直接返回缓存)。这是手写的、显式的缓存失效,不是 React 那种自动追踪依赖——但对这个规模的 UI 足够了,而且行为完全可预测。

## 15.2 差分渲染:逐行比对 + CSI 2026 同步输出

有了「整棵树 render 成 `newLines: string[]`」,刷新屏幕最笨的办法是清屏重画。但那样每次模型吐一个 token 都会让整个历史闪一下。pi 的 `render` 主循环(`tui.ts`)做的是**差分**:把这次的 `newLines` 和上次缓存的 `this.previousLines` 逐行比,只重写变化的那几行。

核心逻辑是找出「第一处变化」和「最后一处变化」:

```ts
let firstChanged = -1;
let lastChanged = -1;
const maxLines = Math.max(newLines.length, this.previousLines.length);
for (let i = 0; i < maxLines; i++) {
	const oldLine = i < this.previousLines.length ? this.previousLines[i] : "";
	const newLine = i < newLines.length ? newLines[i] : "";
	if (oldLine !== newLine) {
		if (firstChanged === -1) firstChanged = i;
		lastChanged = i;
	}
}
```

算出 `[firstChanged, lastChanged]` 这个变化区间后,光标只移动到 `firstChanged` 行,重写到 `lastChanged` 行,其余行原封不动。流式追加(最常见的场景——模型一行行往下吐)被特判成 `appendedLines`:大多数情况下 `firstChanged` 就是旧内容末尾,只往下追加新行,完全不动已有内容。当全文无变化时(`firstChanged === -1`),连一个字节都不写,只在必要时挪一下硬件光标。

但「只写变化的行」还不够——如果浏览器或终端在你写到一半时刷新,用户会看到撕裂。pi 用 **CSI 2026 同步输出**把一次刷新包成原子操作:

```ts
let buffer = "\x1b[?2026h"; // Begin synchronized output
// ……拼接所有要写的行……
buffer += "\x1b[?2026l"; // End synchronized output
this.terminal.write(buffer);
```

`\x1b[?2026h` / `\x1b[?2026l` 是终端的「开始/结束同步更新」私有序列:支持的终端会把这两者之间的所有写入缓冲起来,一次性原子地呈现——杜绝闪烁和撕裂。整个缓冲区一次 `write` 写出,而不是一行一个系统调用。

有几种情况差分救不了,必须**全量重画**(`fullRender(true)`,会先 `\x1b[2J\x1b[H\x1b[3J` 清屏 + 清 scrollback):

- **宽度变了**(`widthChanged`):换行位置全变,旧的行布局作废,只能重排。
- **高度变了**(`heightChanged`)——但有个精妙的例外:Termux(安卓终端)在软键盘弹出/收起时会改高度,若每次都全量重画,整段历史会在键盘开合时反复重放。所以代码特判 `!isTermuxSession()`,在 Termux 上不因高度变化而全量重画。
- **内容缩短**(`clearOnShrink`):新内容比历史最高点矮,需要清掉下方残留的空行。

> **给 Agent 开发者的启示**:差分渲染的精髓不是「算法多聪明」,而是**把「最常见的那条路径」做到极致便宜**。Agent CLI 里 99% 的刷新是「流式追加几行」,pi 为这条路径设计了 `appendedLines` 特判——几乎零成本。剩下的全量重画只在宽高变化这种罕见时刻发生。而 Termux 的高度特例则提醒你:**真实终端的行为千奇百怪,差分渲染器的复杂度往往不在算法,而在这些环境适配的边界**。`PI_DEBUG_REDRAW=1` 还能把每次全量重画的原因写进日志——这种「让不可见的渲染决策变得可观测」的细节,是成熟工程的标志。

## 15.3 硬件光标与中文输入法:`CURSOR_MARKER` 与 `Focusable`

终端有一个「硬件光标」——那个闪烁的方块。普通输出不需要管它,但有两件事必须把它放对位置:一是编辑器里你正在输入的插入点,二是**中文/日文输入法的候选词窗口**——它会贴着硬件光标弹出。如果光标位置不对,中文输入时候选框会飘到屏幕角落。

pi 用一个巧妙的「零宽标记」解决。`Focusable` 接口和 `CURSOR_MARKER` 常量:

```ts
export interface Focusable {
	/** Set by TUI when focus changes. Component should emit CURSOR_MARKER when true. */
	focused: boolean;
}

/**
 * Cursor position marker - APC (Application Program Command) sequence.
 * This is a zero-width escape sequence that terminals ignore.
 */
export const CURSOR_MARKER = "\x1b_pi:c\x07";
```

机制是这样的:获得焦点的组件(比如 `Input`、`Editor`)在 `render` 时,在「光标应该在的那个字符位置」插入 `CURSOR_MARKER`——一个 APC 转义序列,终端会直接忽略它、不占任何显示宽度。`Input` 组件的 `render` 末尾:

```ts
const marker = this.focused ? CURSOR_MARKER : "";
```

然后 TUI 在真正写屏前,用 `extractCursorPosition` 扫描所有行,**找到这个标记的行列坐标,把它从输出里剥掉,再用普通的 ANSI 光标定位序列把硬件光标移到那个坐标**:

```ts
const markerIndex = line.indexOf(CURSOR_MARKER);
// ……记录 row/col……
lines[row] = line.slice(0, markerIndex) + line.slice(markerIndex + CURSOR_MARKER.length);
```

这样,组件**不需要知道自己最终被渲染在屏幕的第几行**——它只要在自己的相对坐标里埋一个标记,TUI 在合成完整画面后自然算出绝对坐标。这是一种漂亮的关注点分离:组件管「光标在我内部的哪儿」,TUI 管「我内部最终落在屏幕的哪儿」。中文输入法的候选窗口因此能精准贴在插入点旁边(还有 `PI_HARDWARE_CURSOR` 环境变量可调行为)。

## 15.4 键盘:Kitty 协议与可折叠的工具/Skill 可视化

终端键盘输入是一团历史包袱:Ctrl+I 和 Tab 是同一个字节,Esc 和方向键前缀混在一起,组合键无法区分。pi 在 `keys.ts` 里支持 **Kitty 键盘协议**——一个现代终端约定,把按键编码成结构化、可区分修饰键(ctrl/shift/alt/super)、还能上报「按下/释放/重复」事件的序列:

```ts
export {
	isKittyProtocolActive,
	setKittyProtocolActive,
	matchesKey, parseKey,
	type KeyId, type KeyEventType,
} from "./keys.ts";
```

`Component.wantsKeyRelease` 这个可选字段就是为它服务的:默认 TUI 会把「按键释放」事件过滤掉(大多数组件只关心按下),但需要的组件可以声明 `wantsKeyRelease = true` 来接收它们。键位绑定本身则由 `keybindings.ts`(`KeybindingsManager`、`TUI_KEYBINDINGS`)集中管理,还能检测冲突(`KeybindingConflict`)。

有了渲染和输入两条腿,pi 的交互组件就能把内核的事件画成丰富的 UI。回顾第 14 章的 `SkillInvocationMessageComponent`——它就是个 `Box` 子类,默认折叠成一行 `[skill] 名字 (按键展开)`,展开后用 `Markdown` 组件渲染全文。同样的「可折叠」模式贯穿了所有重组件:`ToolExecutionComponent`(`tool-execution.ts`)把一次工具调用渲染成可展开的块——折叠时只显示工具名和摘要,展开看完整参数和输出;它还能内联渲染图片(`Image` 组件 + Kitty 图片协议),把工具产出的图片直接画在终端里。

这正是第 11 章「树里存全集、喂模型子集」思维在 UI 层的第三次回响:**会话树里存的是全量内容,喂给模型的是裁剪过的子集,而呈现给用户的——默认是折叠的一行,需要时才展开全文**。内核、模型、用户三方各取所需的视图,互不干扰。

> **给 Agent 开发者的启示**:Agent 的输出天然是「层次化、可折叠」的——一次工具调用有「调了什么」和「返回了什么」,一个 skill 有「名字」和「正文」。如果全部平铺,终端会被刷屏淹没。pi 的 UI 组件统一采用「默认折叠成一行、按键展开全文」的范式,把**屏幕空间这个稀缺资源**留给用户最关心的主线对话。设计 Agent UI 时,先问「哪些是主线、哪些是可按需展开的细节」,比纠结配色重要得多。

## 15.5 一个内核,多个入口:interactive / print / rpc

最后值得点出:`pi-tui` 只是 pi 的**一个**前端。同一个 Agent 内核(`AgentSessionRuntime`)被接到三种 `modes/`:

| 模式 | 文件 | 用途 |
|---|---|---|
| **interactive** | `modes/interactive/interactive-mode.ts` | 默认的全屏 TUI——差分渲染、键盘、可折叠组件,本章主角 |
| **print** | `modes/print-mode.ts` | 单发模式:`pi -p "prompt"` 输出文本;`--mode json` 输出 JSON 事件流,然后退出 |
| **rpc** | `modes/rpc/rpc-mode.ts` | 把 pi 当作子进程,通过 JSONL 协议(`rpc-types.ts`)被别的程序驱动 |

print 模式根本不加载 TUI——它只订阅内核事件,把最终回复(或 JSON 事件流)直接写 stdout 再退出,这正是脚本化、CI、管道场景需要的。rpc 模式则把 pi 变成一个可被远程调用的引擎。三种模式共享完全相同的内核循环,只是「输出去哪儿、输入从哪来」不同。

这印证了全书的主线:**Harness 内核与呈现层是彻底解耦的**。`runLoop`、`AgentHarness`、会话树、扩展——它们从不知道屏幕上长什么样;`pi-tui` 也从不碰模型契约。正因为这条接缝切得干净,pi 才能用同一个内核同时服务「人在终端前敲字」「脚本一次性调用」「程序远程驱动」三种截然不同的形态。

至此,pi 的「对外的脸」就讲完了。下一章是全书最后一章,我们退回工程化的视角,看 pi 如何用 `faux` provider、TypeBox strip-only 类型、以及「用自己开发自己」的 dogfooding,把这套 Harness 做成**可信、可测、可持续维护**的系统。

---

## 本章小结

- pi 不用 ink/React,自研 `pi-tui`。核心抽象是 `Component.render(width) => string[]`——组件就是「给定宽度、吐出每行字符串(可含 ANSI)」的函数,组件树是普通对象树,没有虚拟 DOM/reconciler。`invalidate()` 是显式手动的缓存失效。
- 差分渲染:把本次 `newLines` 与缓存的 `previousLines` 逐行比对,算出 `[firstChanged, lastChanged]` 区间,只重写变化行;流式追加走 `appendedLines` 特判几乎零成本;无变化时一个字节都不写。
- 一次刷新用 CSI 2026 同步输出(`\x1b[?2026h`/`\x1b[?2026l`)包成原子操作,杜绝闪烁撕裂,整个缓冲区一次 `write`。宽度变化、(非 Termux 的)高度变化、内容缩短(clearOnShrink)触发全量重画;`PI_DEBUG_REDRAW=1` 记录重画原因。
- 硬件光标用零宽 APC 标记 `CURSOR_MARKER` 解决:焦点组件在 render 时埋标记,TUI 扫描出行列坐标、剥掉标记、再定位硬件光标——组件无需知道自己的绝对屏幕位置。这让中文/日文输入法候选窗口能精准贴在插入点。
- 键盘支持 Kitty 协议(结构化修饰键 + 按下/释放/重复事件),`wantsKeyRelease` 控制是否接收释放事件,`KeybindingsManager` 集中管理键位与冲突。
- 工具调用、skill 块统一采用「默认折叠一行、按键展开全文」的可视化范式(`ToolExecutionComponent`/`SkillInvocationMessageComponent`),是「树存全集、模型取子集、UI 默认折叠」三方分层的体现。
- 同一个内核接三种前端:interactive(TUI)、print(单发文本/JSON 后退出,不加载 TUI)、rpc(JSONL 协议被远程驱动)——印证内核与呈现层彻底解耦。

下一章,看 pi 如何用 faux provider、TypeBox 与 dogfooding 把这套 Harness 做成可信、可测的系统。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/tui/src/tui.ts` | `Component`/`Focusable` 接口、`CURSOR_MARKER`、差分渲染主循环(firstChanged/lastChanged、appendedLines、fullRender、CSI 2026 同步输出、extractCursorPosition) |
| `packages/tui/src/index.ts` | `pi-tui` 公共导出:组件、键盘、键位、自动补全、模糊匹配 |
| `packages/tui/src/components/box.ts` | `Box` 容器:padding/背景、子组件拼接、渲染缓存 |
| `packages/tui/src/components/input.ts` | `Input` 组件:`focused` + `CURSOR_MARKER` 的典型用法 |
| `packages/tui/src/keys.ts` | Kitty 键盘协议:`setKittyProtocolActive`/`matchesKey`/`parseKey`/`KeyId` |
| `packages/tui/src/keybindings.ts` | `KeybindingsManager`、`TUI_KEYBINDINGS`、冲突检测 |
| `packages/coding-agent/src/modes/interactive/components/tool-execution.ts` | `ToolExecutionComponent`:可折叠的工具调用可视化、内联图片 |
| `packages/coding-agent/src/modes/print-mode.ts` | 单发模式:文本 / JSON 事件流输出后退出 |
| `packages/coding-agent/src/modes/rpc/rpc-mode.ts` | RPC 模式:JSONL 协议驱动内核 |

## 动手实验

1. 设 `PI_DEBUG_REDRAW=1` 启动 pi,做几件事:正常对话(流式追加)、把终端拉宽再拉窄、按 Ctrl+L 之类触发重画。然后看 `~/.pi/agent/pi-debug.log`,对照 `tui.ts` 里 `logRedraw` 的各个分支,搞清每次**全量重画的原因**(first render / width changed / height changed / clearOnShrink)。哪些操作走了差分、哪些被迫全量重画?
2. 读 `components/box.ts` 的 `render` 与缓存逻辑(`matchCache`/`invalidateCache`)。手动构造:同一个 `Box` 连续两次 `render(80)`、子组件不变——第二次是否命中缓存返回?再调一次 `setBgFn` 换背景色,观察它**不**直接失效缓存而是靠 `bgSample` 采样检测的设计,想想为什么这样更稳。
3. 在支持 Kitty 协议的终端(如 Kitty/Ghostty)和不支持的终端(如老版 Terminal.app)里分别启动 pi,用 `isKittyProtocolActive()` 的返回值对照行为差异。再在 `Input` 里输入中文,观察输入法候选窗口是否精准贴在光标处——然后到 `tui.ts` 的 `extractCursorPosition` 看 `CURSOR_MARKER` 是如何被扫描、剥离并转成硬件光标定位的。
