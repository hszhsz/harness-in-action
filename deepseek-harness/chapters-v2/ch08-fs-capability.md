# 第 8 章 文件系统能力：FsTarget 与 Capability Seam

上一章工具守卫管线通过了 `write_file` 的调用。现在请求到达了文件系统层——`packages/fs/fs/src/` 和 `packages/fs/tool-fs/src/`。在 dsh 中，文件系统不是一个直接的 I/O 操作，而是一个通过"能力接缝"（Capability Seam）注入的抽象接口。

## 请求到达这里时发生了什么

dsh 的文件系统设计遵循一个核心原则：**工具不直接操作文件系统，而是通过一个能力接口**。这个接口就是 `FsTarget`。

`FsTarget` 定义了文件系统操作的抽象：

```typescript
interface FsTarget {
  readFile(path: string): Promise<string>
  writeFile(path: string, content: string): Promise<void>
  editFile(path: string, oldText: string, newText: string): Promise<void>
  listDir(path: string): Promise<DirEntry[]>
  // ...
}
```

当 `write_file` 工具被调用时，它不直接调用 `fs.writeFile`，而是通过 `ctx` 获取注入的 `FsTarget` 实例来操作。

## 关键代码路径

### Capability Seam 模式

"能力接缝"是 dsh 中反复出现的设计模式。它的核心思想：

1. **接口定义在消费者侧**：`FsTarget` 定义在 `packages/fs/fs/src/`，这是消费者的包
2. **实现在提供者侧**：实际的文件系统操作在 `packages/fs/tool-fs/src/` 中实现
3. **通过 Cordis 依赖注入连接**：提供者注册一个 Service，消费者注入它

这意味着：
- 在本地运行时，`FsTarget` 可以直接操作真实的文件系统
- 在沙箱环境中，`FsTarget` 可以代理到受限的文件系统
- 在测试中，`FsTarget` 可以用内存文件系统替代

### tool-fs：FS 工具的实现

`packages/fs/tool-fs/src/` 注册了文件系统相关的工具（`read_file`、`write_file`、`edit_file` 等）。每个工具：

1. 从 `ctx` 获取 `FsTarget` 实例
2. 验证路径（防止路径穿越攻击）
3. 调用 `FsTarget` 的对应方法
4. 返回带有 `meta`（如 contextual diff）的 `ToolResult`

对于 `write_file`，典型的执行流程：

```typescript
async execute(params: { path: string; content: string }, context) {
  const fs = context.ctx.get('fsTarget')
  const resolvedPath = resolvePath(context.cwd, params.path)
  validatePath(resolvedPath, context.cwd)  // 防止路径穿越
  await fs.writeFile(resolvedPath, params.content)
  return {
    content: [{ type: 'text', text: `Wrote ${resolvedPath}` }],
    meta: { diff: computeDiff(original, params.content) }
  }
}
```

### 路径安全

`validatePath` 是一个关键的安全检查。它确保：
- 解析后的路径在允许的目录范围内（通常是 `cwd` 或 agent 的工作区）
- 不能通过 `../` 跳出允许的目录
- 不能写入敏感路径（如 `~/.ssh/`、`/etc/`）

### edit_file 的精确性

`edit_file` 工具接受 `oldText` 和 `newText`，执行精确的文本替换而非覆盖整个文件。这要求：
- `oldText` 必须在文件中恰好出现一次
- 如果出现多次或不出现，操作失败

这种设计让模型可以精确修改文件的特定部分，而不必重写整个文件。

## 调用链：从这里到下一步

当 `write_file` 通过 `FsTarget.writeFile` 执行时，如果当前运行在沙箱环境中，请求会经过沙箱层。

**下一步：请求将流向 Sandbox 模块——`packages/sandbox/sandbox/src/`。沙箱如何限制文件写入？bwrap、Landlock、Seatbelt 三种后端有什么区别？**

## 关键设计决策

1. **能力接缝而非直接 I/O**：通过接口抽象文件系统，使得同一个工具可以在不同环境下（本地、沙箱、测试）使用不同的实现。

2. **路径验证在工具侧**：路径安全检查在 `tool-fs` 中而非在 `FsTarget` 实现中，因为安全策略可能因工具而异。

3. **edit_file 的精确替换**：要求 `oldText` 恰好出现一次，避免了模糊匹配可能引入的错误。

4. **meta 携带 diff**：`write_file` 和 `edit_file` 在 `meta` 中携带上下文差异，让 UI 可以展示"什么变了"，而核心系统不需要理解 diff 的结构。

## 本章小结

- 文件系统通过 `FsTarget` 接口抽象，不直接操作 I/O
- Capability Seam 模式：接口在消费者侧，实现在提供者侧，通过 DI 连接
- `tool-fs` 注册 FS 工具，验证路径安全后调用 `FsTarget`
- `edit_file` 使用精确文本替换而非覆盖
- `meta` 字段携带 contextual diff 供 UI 展示

## 阅读思考题

1. 如果把 `FsTarget` 的定义移到提供者侧（`tool-fs` 包），会引入什么循环依赖问题？
2. `validatePath` 为什么不在 `FsTarget` 实现中做，而是在工具侧做？
3. `edit_file` 要求 `oldText` 恰好出现一次，如果文件内容恰好有两处相同的文本，模型应该怎么处理？

> **下一章预告**：文件系统操作已经到达了 `FsTarget` 的实现层。但在沙箱模式下，写入操作会被 bwrap、Landlock 或 Seatbelt 拦截。下一章我们看沙箱如何在操作系统层面限制 Agent 的能力。
