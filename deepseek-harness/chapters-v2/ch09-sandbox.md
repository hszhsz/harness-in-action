# 第 9 章 沙箱：操作系统级的 Agent 限制

上一章 `write_file` 通过 `FsTarget` 接口到达了文件系统实现层。但在沙箱模式下，操作系统的安全机制会在最后一道关卡拦截不被允许的操作。这一章我们看 `packages/sandbox/sandbox/src/` 如何利用 bwrap、Landlock、Seatbelt 三种后端在 OS 层面限制 Agent。

## 请求到达这里时发生了什么

沙箱是 dsh 的最后一道安全防线——即使前面的所有守卫都通过了，操作系统层面的限制仍然可以阻止危险操作。沙箱的设计哲学是**深度防御**：不信任任何上层检查，用 OS 原语做最终裁决。

`SandboxMode` 定义了沙箱的几种模式：

```typescript
type SandboxMode = 'off' | 'bwrap' | 'landlock' | 'seatbelt'
```

每种模式对应不同的操作系统安全机制：
- **bwrap**（Bubblewrap）：Linux 的轻量级命名空间沙箱
- **Landlock**：Linux 的文件系统访问控制 LSM
- **Seatbelt**：macOS 的沙箱框架

## 关键代码路径

### 三种后端的选择

沙箱后端的选择取决于运行平台：

- **Linux**：优先使用 Landlock（内核 5.13+），回退到 bwrap
- **macOS**：使用 Seatbelt
- **其他/测试**：沙箱关闭（`off`）

选择逻辑在沙箱服务启动时确定：

```typescript
function selectSandboxBackend(): SandboxMode {
  if (process.platform === 'darwin') return 'seatbelt'
  if (process.platform === 'linux') {
    if (supportsLandlock()) return 'landlock'
    if (supportsBwrap()) return 'bwrap'
  }
  return 'off'
}
```

### Bubblewrap（bwrap）后端

bwrap 使用 Linux 命名空间创建隔离环境：

- **Mount namespace**：只挂载允许的目录
- **PID namespace**：隔离进程树
- **Network namespace**：默认断网

当 Agent 调用 shell 命令时，bwrap 后端会用 `bwrap` 命令包装实际执行：

```
bwrap --ro-bind /usr /usr --ro-bind /lib /lib --bind /workspace /workspace -- command
```

这样 `command` 只能看到 `/workspace` 是可写的，其他目录都是只读的。

### Landlock 后端

Landlock 是 Linux 5.13 引入的安全模块，允许进程自行限制自己的文件系统访问：

```typescript
const ruleset = landlock.createRuleset({
  handledAccessFs: LANDLOCK_ACCESS_FS_ALL,
})
landlock.addRule(ruleset, {
  allowedAccess: LANDLOCK_ACCESS_FS_READ_FILE | LANDLOCK_ACCESS_FS_WRITE_FILE,
  path: workspaceDir,
})
landlock.restrictSelf(ruleset)
```

关键特性：**Landlock 是单向收紧的**——一旦应用了规则，当前进程及其子进程都不能获得比规则更多的权限。这比 bwrap 更轻量，因为它不需要创建新的命名空间。

### Seatbelt 后端

macOS 的 Seatbelt 使用 `.sb` 配置文件定义沙箱策略：

```typescript
const profile = `
(version 1)
(allow default)
(deny file-write*
  (require-not
    (subpath "${workspaceDir}")))
`
sandbox.initWithProfile(profile)
```

Seatbelt 在进程级别应用，不需要命名空间。

### 沙箱与子进程

当 Agent 调用 `shell` 工具执行命令时（`packages/shell/`），沙箱会包装子进程的创建：

```typescript
async function spawnInSandbox(command: string, options: ShellOptions): Promise<ChildProcess> {
  if (sandbox.mode === 'off') return spawn(command, options)
  
  const wrapped = sandbox.wrapCommand(command)
  return spawn(wrapped.executable, wrapped.args, options)
}
```

`wrapCommand` 根据当前沙箱后端包装命令——bwrap 加上命名空间参数，Landlock 在 fork 后应用规则，Seatbelt 在 fork 后应用 profile。

## 调用链：从这里到下一步

沙箱检查通过后（或者沙箱关闭时），文件写入操作完成。但在此之前，某些操作需要用户审批。

**下一步：请求将流向审批系统——`packages/interaction/user-approval/src/`。审批如何工作？fail-closed 策略如何实现？**

## 关键设计决策

1. **深度防御**：沙箱在操作系统层面做最终裁决，不依赖上层的任何检查。即使工具系统的守卫管线有 bug，沙箱仍然能阻止危险操作。

2. **平台自适应**：三种后端覆盖了主流桌面操作系统。选择逻辑是自动的，用户不需要手动配置。

3. **Landlock 优先于 bwrap**：Landlock 更轻量（不需要命名空间），且是进程自我限制（不需要 root 权限）。bwrap 是回退方案。

4. **沙箱是透明的**：对上层工具系统来说，沙箱的存在是透明的——工具代码不需要知道它是否在沙箱中运行。

## 本章小结

- 沙箱是操作系统级的安全防线，采用深度防御策略
- 三种后端：bwrap（Linux 命名空间）、Landlock（Linux LSM）、Seatbelt（macOS）
- Landlock 优先于 bwrap，因为它更轻量且不需要 root
- 沙箱包装子进程创建，对上层工具透明
- 选择逻辑基于平台自动确定

## 阅读思考题

1. 如果 bwrap 和 Landlock 都不可用，沙箱模式变为 `off`。这在生产环境中安全吗？有什么缓解措施？
2. Landlock 的"单向收紧"特性意味着什么？为什么这对安全很重要？
3. 沙箱如何处理符号链接攻击——如果 workspace 内有一个指向 `/etc/passwd` 的符号链接？

> **下一章预告**：沙箱在 OS 层面做了最后一道检查。但在写入文件之前，还有一个环节——用户审批。对于高风险操作，dsh 会请求用户确认。审批系统如何工作？fail-closed 策略如何实现？
