# 第 16 章　自研 TUI：两个渲染器与 `string[]` 差分

> 一个 Agent CLI 最终要被人「看见」。pi 没有用 ink、没有用 React，而是从零写了 `pi-tui`：组件只负责把自身渲染成 `string[]`，刷新交给两个渲染器——保留终端回滚区的 `TuiMainScreen`，和应用自己拥有视口的 `TuiAltScreen`。这一章看它为什么这么选，以及「不发明虚拟 DOM」的渲染器如何做到不闪屏、不撕裂、还能贴住中文输入法。

前面十五章，我们从 `streamSimple` 的模型契约一路走到会话树、压缩、工具、扩展与 MCP。它们都不关心屏幕上长什么样。但 pi 作为交互式 CLI，终究要把流式输出、工具调用、可折叠的 Skill 块画到终端上，还要接收键盘和鼠标、支持 IME、在窗口缩放时保持稳定。这一层就是 `packages/tui`（`@earendil-works/pi-tui`）。

这一章的主角是一个反直觉的工程决策：**pi 放着成熟的 ink/React 终端框架不用，自研了一套渲染器。** 而且最新版本不是一个渲染器，而是两个：默认的 `fullscreen` 用备用屏（alternate screen）自己管理滚动，`regular` 则直接写入主屏、把回滚交给终端。两者共享同一个组件模型，却在「谁拥有屏幕」上做出完全不同的选择。

---

## 16.1 两个渲染器，一条组件模型

先看组件契约。`tui.ts` 里的 `Component` 只有一个必须实现的方法：

```ts
export interface Component {
	render(width: number): string[];

	handleInput?(data: string): void;
	handleMouse?(event: TuiMouseEvent): TuiMouseEventResult | undefined;
	wantsKeyRelease?: boolean;
	invalidate(): void;
}
```

一个组件的本质，就是 **「给我一个宽度，我吐给你一个 `string[]`，每个元素是一行（可含 ANSI 转义）」**。没有虚拟 DOM、没有 fiber、没有 reconciler。组件树就是普通对象树；渲染就是从根 `render(width)` 递归向下，把子组件的行拼起来。`Container` 是最朴素的写照：遍历 `children`，逐个 `render(width)`，再把行首尾相接。

`invalidate()` 是显式的手动缓存失效。主题变了、状态变了，组件清掉自己的缓存；TUI 不追踪依赖，也不猜测哪里脏了。对这个规模的 UI，这足够好，而且行为完全可预测。

在此之上，`TuiBase` 统一处理与渲染器无关的公共设施：

- **焦点**：`setFocus()` 维护当前聚焦组件，并把 `focused` 写到实现 `Focusable` 的组件上；
- **输入**：把键盘数据转发给聚焦组件，并管理全局 `addInputListener`；
- **覆盖层**：`showOverlay()`/`hideOverlay()` 维护一个带焦点恢复的 overlay 栈；
- **渲染调度**：`requestRender()` 合并同一帧内的多次请求，两次真实渲染之间至少间隔 16ms；
- **终端能力**：查询主题色（OSC 10/11/4，再以 DA1 回复作为结束标记）、显示/隐藏硬件光标、终端配色变化通知。

真正写屏的是两个子类：

| | `TuiMainScreen`（`regular`） | `TuiAltScreen`（`fullscreen`，默认） |
|---|---|---|
| 屏幕 | 终端主屏，历史留在 scrollback | 备用屏 `\x1b[?1049h`，固定高度视口 |
| 滚动 | 终端负责，历史可无限回溯 | 应用负责，`ScrollView` + 布局树 |
| 差分对象 | `previousLines` | `previousScreen`（按行等长比较） |
| 鼠标 | 交给终端（终端拥有选择与滚轮） | 归一化事件路由给组件、选择、链接 |
| 搜索 | 由终端/应用各自处理 | 内置全文搜索、高亮、跳转 |
| 退出 | 内容留在主屏 | 退出备用屏，并把最终文档重新打印到主屏 |

`createInteractiveTui()` 根据 `tuiMode` 二选一：`"fullscreen"` 构造 `TuiAltScreen`，`"regular"` 构造 `TuiMainScreen`。默认值是 `fullscreen`，所以「打开 pi 后历史不会污染终端回滚区、滚轮由 pi 自己处理」是当前版本的默认体验；`regular` 则更像传统 CLI，退出后整段对话仍在终端历史里。

> **给 Agent 开发者的启示**：终端 UI 的最小充分抽象就是 `(width) => string[]`。React 的虚拟 DOM 是为浏览器那种「任意嵌套、任意属性」的树设计的；终端是「按行、按列」的二维字符网格，它的 DOM 天然就是字符串数组。一旦认清这一点，UI 库的复杂度就坍缩成两个问题：**怎么把组件树变成行**，以及**怎么把新旧行的差异高效写到终端**。剩下的取舍——谁拥有滚动、谁处理鼠标——可以按场景各选一套，而不必让整个框架背着一个 reconciler。

## 16.2 Regular 模式：在主屏上做行级差分

`TuiMainScreen` 的刷新逻辑，是「整棵树 render 成 `newLines`，再和上次的 `previousLines` 逐行比」。最笨的办法是清屏重画，但那样模型每吐一个 token，整段历史都会闪一下。它做的是：

1. 找出第一处和最后一处变化的行（`firstChanged` / `lastChanged`）；
2. 流式追加被特判成 `appendedLines`：大多数时候 `firstChanged` 就是旧内容末尾，只往下追加；
3. 光标只移动到 `firstChanged`，只重写到 `lastChanged`（且不超过视口底部），其余行原封不动；
4. 全文无变化时（`firstChanged === -1`），一个字节都不写，只在必要时挪一下硬件光标。

```ts
for (let i = 0; i < maxLines; i++) {
	const oldLine = i < this.previousLines.length ? this.previousLines[i] : "";
	const newLine = i < newLines.length ? newLines[i] : "";
	if (oldLine !== newLine) {
		if (firstChanged === -1) firstChanged = i;
		lastChanged = i;
	}
}
```

「只写变化的行」还不够——终端如果在写到一半时刷新，用户会看到撕裂。pi 用 **CSI 2026 同步输出**把一次刷新包成原子操作：

```ts
output.append("\x1b[?2026h"); // Begin synchronized output
// ……拼接所有要写的行……
output.append("\x1b[?2026l"); // End synchronized output
output.flush();
```

支持的终端会把这两者之间的写入缓冲起来，一次性呈现。pi 还用 `BoundedTerminalWriter` 把输出切成有界块再 `write`，避免一次写出巨大的字符串。

有几种情况差分救不了，必须 **全量重画**：

- **宽度变了**：换行位置全变，旧的行布局作废；
- **高度变了**：需要重新对齐视口——但 Termux（安卓终端）在软键盘弹出/收起时会改高度，若每次都全量重画，整段历史会在键盘开合时反复重放，所以代码特判 `isTermuxSession()`，在 Termux 上忽略高度变化；
- **内容缩短**且开启了 `clearOnShrink`：新内容比历史最高点矮，需要清掉下方残留的空行；
- **第一处变化在视口上方**：差分只能碰「曾经真正可见」的行，视口外的历史已经交给终端的 scrollback，改不了，只能重画。

最后一条尤其重要，源码里的注释写得很直白：**Differential rendering can only touch what was actually visible.** 主屏模式下，滚动出去的内容归终端所有，pi 不会假装自己还能修改它。这也是为什么主屏模式必须把「视口顶部」`viewportTop` 当作一等状态来跟踪。

可观测性也做在这儿：`PI_TUI_DEBUG_REDRAW=1` 会把每次全量重画的**原因**追加到 `pi-tui-debug.log`（first render / width changed / height changed / clearOnShrink / firstChanged < viewportTop…）；`PI_TUI_DEBUG=1` 还会把每一帧的 `newLines`、`previousLines` 和光标状态写到 `/tmp/tui/render-*.log`。

> **给 Agent 开发者的启示**：差分渲染的精髓不是「算法多聪明」，而是**把最常见的那条路径做到极致便宜**。Agent CLI 里 99% 的刷新是「流式追加几行」，pi 为这条路径设计了 `appendedLines` 特判，几乎零成本；剩下的全量重画只留给宽高变化、视口上方变化这类罕见时刻。Termux 特例则提醒你：真实终端千奇百怪，差分渲染器的复杂度往往不在算法，而在环境适配的边界。

## 16.3 Fullscreen 模式：应用拥有视口

备用屏是另一种世界观。进入时写 `\x1b[?1049h`，终端切换到一个没有历史、固定高度的空白屏；退出时写 `\x1b[?1049l`，原来的主屏内容原样回来。因为屏幕完全归应用所有，`TuiAltScreen` 可以做主屏做不到的事：

- **应用自己的滚动**：内容先渲染进布局树，`ScrollView` 决定哪一段落在视口里；
- **全文搜索**：`alt-screen-search.ts` 维护匹配列表、当前项、高亮与跳转；
- **鼠标选择与复制**：按下、拖拽、双击、自动滚动、剪贴板，全部由 pi 处理；
- **OSC 8 链接**：链接优先于外层点击区域；
- **覆盖层、闪烁提示、滚动到底部的指示条**。

它的差分更直接：整屏固定为 `height` 行，逐行比较 `screen[row] !== previousScreen[row]`，只重写变化的行。发生宽高变化、或第一次渲染时全量重画；每一行用 `\x1b[row;1H\x1b[2K` 定位并清行。它同样用 CSI 2026 包住整帧。

图片是这里最麻烦的部分。Kitty 图片协议把图片作为「放在某个单元格上的 placement」传输，终端会在后续文本写入碰到被覆盖的行时擦掉图片。于是 `TuiAltScreen` 要：

- 全量重画时先删除所有 placement；
- 只重画变化行时，如果变化行与图片覆盖的行相交，就触发图片重绘；
- 在 WezTerm 上，因为「写入相交行会擦掉 Kitty 图片」，把图片 placement 放到所有文本之后最后绘制；
- 用预算缓存屏幕外的图片，超出上限才删除，避免滚动时反复上传。

退出备用屏时，`afterTerminalStop()` 会把最终组件树重新 `render()` 一次，剥掉光标标记、按宽度截断，再逐行打印到主屏。所以 `fullscreen` 退出后你仍能看到完整的最终对话，而不是一片空白——只是这份「最终文档」是退出时重画的一份快照，不再参与后续差分。

> **给 Agent 开发者的启示**：主屏和备用屏不是「新旧两版」，而是两种所有权模型。主屏把滚动和历史交给终端，换来「退出后内容仍在」和极低的内存占用；备用屏把视口拿回来，换来搜索、鼠标、固定布局和可预测的重绘，但要自己处理滚动、图片和退出时的交接。选哪一种，取决于你的产品更在意「像传统 CLI 一样融入终端」，还是「像一个应用一样掌控界面」。pi 把两者都做成同一套 `Component` 上的渲染器，而不是让组件去适配两套 API——这是接缝切得好的价值。

## 16.4 硬件光标与中文输入法：`CURSOR_MARKER`

终端有一个「硬件光标」——那个闪烁的方块。普通输出不需要管它，但有两件事必须把它放对位置：编辑器里的插入点，以及**中文/日文输入法的候选词窗口**。候选窗贴着硬件光标弹出，光标位置错了，候选窗就会飘到屏幕角落。

pi 用一个零宽标记解决。`Focusable` 接口和 `CURSOR_MARKER` 常量：

```ts
export interface Focusable {
	/** Set by TUI when focus changes. Component should emit CURSOR_MARKER when true. */
	focused: boolean;
}

export const CURSOR_MARKER = "\x1b_pi:c\x07";
```

机制是：获得焦点的组件（`Input`、`Editor`）在 `render` 时，在「光标应该在的字符位置」插入这个 APC 序列。终端会忽略它、不占显示宽度。TUI 在写屏前调用 `extractCursorPosition()`：

```ts
const markerIndex = line.indexOf(CURSOR_MARKER);
const beforeMarker = line.slice(0, markerIndex);
const col = visibleWidth(beforeMarker); // 按可见列宽，而不是字符数
lines[row] = line.slice(0, markerIndex) + line.slice(markerIndex + CURSOR_MARKER.length);
return { row, col };
```

它只扫描底部 `height` 行（可见视口），找到标记后算出**可见列宽**（中文、emoji、组合字符都算对），剥掉标记，再用 `\x1b[row;colH` 把硬件光标移过去。主屏和备用屏各自在渲染末尾调用同一套逻辑。

组件因此**不需要知道自己最终被渲染在屏幕的第几行**：它只在自己的相对坐标里埋一个标记，TUI 在合成完整画面后自然算出绝对坐标。这是漂亮的关注点分离——组件管「光标在我内部的哪儿」，TUI 管「我内部最终落在屏幕的哪儿」。

还有一层「假光标」：当硬件光标被隐藏时，`renderFakeCursor()` 用 `FAKE_CURSOR_START/END` 包住光标字符，TUI 把它解析成反显（`\x1b[7m`）；当硬件光标显示时，聚焦组件里紧跟在 `CURSOR_MARKER` 后的假光标会被丢弃，避免「画了一个光标、终端又画一个」。这解决了不支持硬件光标、或用户关掉光标时的可见性问题。

> **给 Agent 开发者的启示**：IME 支持不是「加个输入框」就自动有的。任何包住 `Input`/`Editor` 的容器，都必须把 `focused` 状态**逐层传下去**，否则标记根本不会出现在正确的位置——这是 pi 文档里专门写的一条坑。跨平台终端 UI 的许多 bug，最后都落在「你以为你在画字符，其实你在画单元格」这句话上：宽度要按 `visibleWidth()` 算，截断要按列切，光标要按可见列定位。

## 16.5 键盘与鼠标：把终端的历史包袱收进一层

终端键盘输入是一团历史包袱：Ctrl+I 和 Tab 是同一个字节，Esc 和方向键前缀混在一起，组合键无法区分。pi 在 `keys.ts` 里支持 **Kitty 键盘协议**——把按键编码成结构化、可区分修饰键、还能上报按下/释放/重复的序列。不支持的终端则回退到传统字节解析。`matchesKey()` 和 `Key` 让组件按语义判断按键，`KeybindingsManager` 把「动作 → 键位」集中管理，还带冲突检测。

`Component.wantsKeyRelease` 是为释放事件服务的：默认 TUI 会过滤「按键释放」，大多数组件只关心按下；需要的组件声明 `true` 才会收到。

鼠标在 `fullscreen` 下由 `TuiAltScreen` 接管：它把终端上报的坐标归一化成 `TuiMouseEvent`，按布局盒（layout box）路由给组件；未被处理的滚轮事件滚动最近的 `ScrollView`，未被处理的主键拖拽留给对话选择，OSC 8 链接优先于外层点击区域。组件可以实现 `handleMouse()` 返回「是否消费」，从而把鼠标能力做成和键盘一样的可选契约。

`regular` 模式则刻意**不**接管鼠标——终端拥有 scrollback，也拥有选择与滚轮。pi 文档里因此有一条设计约束：即使 fullscreen 支持鼠标，每个交互也必须有一条纯键盘路径。

## 16.6 组件库与「默认折叠」的呈现哲学

`pi-tui` 自带一组不依赖具体应用的组件：

| 类别 | 组件 |
|---|---|
| 内容 | `Text`、`Markdown`、`Image`、`TruncatedText` |
| 布局 | `Container`、`VStack`、`HStack`、`Box`、`Spacer` |
| 输入 | `Input`、`Editor` |
| 选择 | `SelectList`、`SettingsList` |
| 视口 | `ScrollView` |
| 状态 | `Loader`、`CancellableLoader` |
| 交互 | `MouseRegion` |

`coding-agent` 在此之上组合出自己的界面：`ToolExecutionComponent`（`tool-execution.ts`）把一次工具调用渲染成可展开的块——折叠时只显示工具名和摘要，展开看完整参数与输出，还能内联渲染图片；`SkillInvocationMessageComponent`（`skill-invocation-message.ts`）把一段 Skill 展开默认折叠成一行。它们都遵循同一个范式：**默认折叠成一行，按键展开全文**。

这是第 11 章「树里存全集、喂模型子集」思维在 UI 层的回响：会话树存的是全量内容，喂给模型的是裁剪过的子集，呈现给用户的默认是折叠的一行。内核、模型、用户三方各取所需的视图，互不干扰。

> **给 Agent 开发者的启示**：Agent 的输出天然是层次化、可折叠的——一次工具调用有「调了什么」和「返回了什么」，一个 Skill 有「名字」和「正文」。如果全部平铺，终端会被刷屏淹没。设计 Agent UI 时，先问「哪些是主线、哪些是按需展开的细节」，比纠结配色重要得多。另外，主题色一律通过 `theme` 注入，不要在组件里硬编码 ANSI 颜色；`invalidate()` 要能清掉所有依赖主题的缓存，否则切换主题后会留下旧颜色。

## 16.7 一个内核，多个入口

`pi-tui` 只是 pi 的一个前端。同一个 Agent 内核（`AgentSessionRuntime`）被接到四种 `AppMode`：

| 模式 | 入口 | 用途 |
|---|---|---|
| **interactive** | `modes/interactive/interactive-mode.ts` | 默认 TUI：差分渲染、键盘、鼠标、可折叠组件 |
| **print** | `modes/print-mode.ts` | 单发：`pi -p "prompt"` 输出文本后退出 |
| **json** | `modes/json-event.ts` | 单发但输出 JSON 事件流，供程序消费 |
| **rpc** | `modes/rpc/rpc-mode.ts` | 把 pi 当子进程，用 JSONL 协议远程驱动 |

模式选择在 `main.ts` 的 `resolveAppMode()` 里：`--mode rpc` / `--mode json` 直接锁定；`--print`、stdin 不是 TTY、或 stdout 不是 TTY，都会落到 `print`；否则才进入 `interactive`。这也解释了为什么管道里运行 pi 不会卡在等键盘——它根本不会加载 TUI。

print 模式只订阅内核事件，把最终回复（或 JSON 事件）写 stdout 再退出，正是脚本、CI、管道需要的。rpc 模式把 pi 变成一个可被远程调用的引擎，`rpc-client.ts` 则是对应的客户端。四种模式共享完全相同的会话内核和工具循环，只是「输入从哪来、输出去哪儿」不同。

如果宿主是另一个 Node.js / Bun 程序而不是终端，还有第五种入口——**SDK**：`createAgentSession()` 直接在进程内返回 `AgentSession`；需要切换/分叉会话时用 `AgentSessionRuntime`（`newSession()`、`switchSession()`、`fork()`、`importFromJsonl()`）。RPC 适合跨语言或需要进程隔离的场景，SDK 适合同进程深度集成；两者与上面四种模式共用同一套会话与工具实现。

这印证了全书的主线：**Harness 内核与呈现层是彻底解耦的**。`runLoop`、`AgentSession`、会话树、扩展从不知道屏幕上长什么样；`pi-tui` 也从不碰模型契约。正因为这条接缝切得干净，pi 才能用同一个内核同时服务「人在终端前敲字」「脚本一次性调用」「程序远程驱动」三种截然不同的形态。

---

## 本章小结

- 组件契约是 `Component.render(width) => string[]`，没有虚拟 DOM/reconciler；`invalidate()` 是显式缓存失效。`TuiBase` 统一焦点、输入、覆盖层、渲染调度（16ms 合并）和终端能力查询。
- 两个渲染器共享组件模型：`TuiMainScreen`（`regular`）写主屏、把滚动交给终端；`TuiAltScreen`（`fullscreen`，默认）用备用屏自持视口，自己做滚动、搜索、鼠标和图片。
- 主屏差分：逐行找 `firstChanged`/`lastChanged`，流式追加走 `appendedLines`，无变化零写入；宽度变化、非 Termux 的高度变化、`clearOnShrink`、以及「首处变化在视口上方」触发全量重画；CSI 2026 同步输出 + 有界写入防撕裂。`PI_TUI_DEBUG_REDRAW=1` 记录重画原因。
- 备用屏差分：固定高度逐行比较，宽高变化全量重画；Kitty 图片按覆盖行触发重绘，WezTerm 上最后绘制图片；退出时重渲染最终文档打印回主屏。
- 硬件光标用零宽 APC 标记 `CURSOR_MARKER`：焦点组件在 render 时埋标记，TUI 只扫描可见视口、按 `visibleWidth` 算列、剥掉标记并定位硬件光标；`renderFakeCursor()` 处理硬件光标隐藏时的可见光标。容器必须把 `focused` 传给子 `Input`/`Editor`，否则 IME 候选窗会错位。
- 键盘支持 Kitty 协议与传统回退，`wantsKeyRelease` 控制释放事件，`KeybindingsManager` 管理键位与冲突；fullscreen 路由归一化鼠标事件，regular 把鼠标留给终端。
- 组件库覆盖文本、布局、输入、选择、视口、加载与鼠标；coding-agent 用「默认折叠一行、按键展开」呈现工具调用与 Skill。同一内核接 interactive/print/json/rpc 四种模式，`resolveAppMode()` 根据 `--mode` 与 TTY 状态选择；同进程集成则走 `createAgentSession()` / `AgentSessionRuntime` 的 SDK 入口。

下一章换一条前沿：`durable`、`chord`、`env` 三个包，分别回答「进程崩了怎么续」「多进程插件与状态怎么热更新」「工具怎么跑在另一台机器上」。

---

## 关键文件清单

| 文件 | 作用 |
|---|---|
| `packages/tui/src/tui.ts` | `Component`/`Focusable`/`TUI` 接口、`CURSOR_MARKER`、`FAKE_CURSOR_*`、`TuiBase`（焦点、覆盖层、渲染调度、`extractCursorPosition`、`resolveFakeCursors`） |
| `packages/tui/src/tui-main-screen.ts` | `TuiMainScreen`：主屏行级差分、`appendedLines`、Termux 高度特例、`clearOnShrink`、全量重画原因日志 |
| `packages/tui/src/tui-alt-screen.ts` | `TuiAltScreen`：备用屏、固定视口逐行差分、搜索/选择/滚动条、Kitty 图片缓存与 WezTerm 特例、退出时打印最终文档 |
| `packages/tui/src/layout.ts` / `layout-node.ts` | 布局树与 `renderLayoutFrame`：把组件树排成带矩形盒的帧，供覆盖层、搜索、鼠标路由使用 |
| `packages/tui/src/keys.ts` | Kitty 键盘协议、`matchesKey`/`parseKey`/`Key`、传统回退 |
| `packages/tui/src/keybindings.ts` | `KeybindingsManager`、`TUI_KEYBINDINGS`、冲突检测 |
| `packages/tui/src/terminal.ts` / `terminal-image.ts` | 终端能力探测、颜色查询、Kitty/iTerm2 图片协议 |
| `packages/tui/src/components/*` | `Box`/`VStack`/`HStack`/`ScrollView`/`Editor`/`Markdown`/`Image`/`SelectList` 等组件 |
| `packages/coding-agent/src/modes/interactive/tui-renderer.ts` | `createInteractiveTui()`：`fullscreen` → `TuiAltScreen`，`regular` → `TuiMainScreen` |
| `packages/coding-agent/src/modes/interactive/interactive-mode.ts` | 交互模式主编排：把会话事件接成组件树 |
| `packages/coding-agent/src/modes/print-mode.ts` / `json-event.ts` / `rpc/rpc-mode.ts` | 另外三种入口 |
| `packages/coding-agent/src/main.ts` | `resolveAppMode()`：`--mode`、`--print` 与 TTY 判定 |

## 动手实验

1. 分别用 `--tui-mode regular` 和默认的 `fullscreen` 启动 pi，做同一段对话：流式追加、拉宽/拉窄窗口、用滚轮翻历史、退出。观察 regular 退出后内容留在终端历史里，而 fullscreen 退出时备用屏恢复、再打印一份最终文档。对照 `beforeTerminalStart()`/`afterTerminalStop()` 看 `\x1b[?1049h`/`\x1b[?1049l` 的时机。
2. 设 `PI_TUI_DEBUG_REDRAW=1`，在 regular 模式里拉宽、拉窄、触发 `clearOnShrink`，看日志里每次全量重画的 reason；再设 `PI_TUI_DEBUG=1`，打开 `/tmp/tui/render-*.log` 对照某一帧的 `newLines`、`previousLines`、`cursorPos`，确认「只重写变化行」到底写了哪几行。
3. 在支持 Kitty 协议的终端（Kitty/Ghostty）和不支持的终端里分别启动 pi，用 `isKittyProtocolActive()` 对照按键解析差异；在 `Input` 里输入中文，观察候选窗是否贴住插入点。然后读 `extractCursorPosition()` 和 `positionHardwareCursor()`，确认标记是如何被扫描、剥离、换算成 `row/col` 的。
4. 读 `examples/extensions/qna.ts` 和 `doom-overlay/`，分别体验「一次性自定义屏幕」和「持续渲染的覆盖层」；再读 `packages/coding-agent/src/modes/interactive/components/tool-execution.ts`，找出它如何用 `Box` + `Markdown` 实现折叠/展开。
