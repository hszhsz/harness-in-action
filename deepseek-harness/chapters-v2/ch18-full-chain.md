# 第 18 章 完整调用链：从终端到任务完成

这是最后一章。让我们把前 17 章的内容串联成一条完整的调用链——从用户按下回车到任务完成，每一步都精确到函数名。

## 完整调用链

### 阶段一：CLI 解析

```
用户输入: dsh --profile headless "帮我写一个 hello.py"
    │
    ▼
bin.ts: parseDshArgs(process.argv.slice(2), readVersion())
    │  args.ts: Commander 解析 --profile, --patch
    │  返回: { mode: 'profile', profile: 'headless', patches: [], args: ['帮我写一个 hello.py'] }
    │
    ▼
bin.ts: switch(invocation.mode) → case 'profile'
    │  动态导入 profile-boot.ts
    │  调用 loadLayeredEnv('dsh') 冻结环境快照
    │
    ▼
profile-boot.ts: runProfile({ environment, profile: 'headless', patchFiles: [], args })
```

### 阶段二：Profile 组装

```
profile-boot.ts: composeProfile('headless', [])
    │
    ├─ prepareProfile('headless')
    │   ├─ healProfilesModuleFallback(INSTALL_ANCHOR)
    │   │   创建 $DSH_HOME/profiles/node_modules 符号链接
    │   │
    │   └─ loadProfile('dsh', 'headless', INSTALL_ANCHOR)
    │       ├─ 读取 profiles/headless/package.json
    │       ├─ 解析 dsh.profile.bundles: ['@deepseek-ai/dsh-base', '@deepseek-ai/dsh-headless']
    │       ├─ resolveBundleDir() → 加载每个 bundle 的 cordis.patch.yml
    │       └─ 返回 Profile { layers, patches }
    │
    ├─ loadOptionalPatches() → $DSH_HOME/cordis.patch.yml
    ├─ loadOverlayPatches() → --patch 文件
    ├─ resolveTelemetryPatch() → 遥测开关
    │
    └─ 返回 ComposedProfile { profile, bundlePatches, homePatches, overlays, rows }
```

### 阶段三：Cordis 启动

```
profile-boot.ts: boot('dsh', rootConfig, allPatches, prepare)
    │
    ├─ app-boot/index.ts: new Context()
    │   ├─ context.ts: 创建 Proxy 包装的根 Context
    │   ├─ 创建根 Fiber (runtime=null, state=ACTIVE)
    │   └─ 安装 ReflectService, RegistryService, EventsService, LoggerService
    │
    ├─ ctx.plugin(Loader)
    │   安装 Cordis Loader 服务
    │
    ├─ prepare(hostCtx)
    │   ├─ hostCtx.provide(DSH_LAUNCH_ENVIRONMENT_KEY, environment)
    │   └─ provideCmdline(hostCtx, { args, exit })
    │
    ├─ mountRootInclude(ctx, rootConfig, patches)
    │   ├─ 注册 Include 和 Group 为 builtin
    │   ├─ 创建 cordis:include 条目
    │   └─ Loader.create(rootInclude)
    │       └─ Include 解析 YAML → 产生子条目 → Loader 并发挂载
    │
    ├─ loader.await() → 等待所有条目稳定
    ├─ assertEntriesActivated() → 验证 Fiber 状态
    │
    └─ installFailLoud() → 进程级安全网
```

### 阶段四：Session 创建

```
应用插件（headless）读取 cmdlineArgs
    │
    ├─ ctx.sessions.create(sessionId, { meta: { cwd } })
    │   ├─ SessionStore.prepare() → Session.create()
    │   ├─ SessionStore.enter() → 安装发布钩子
    │   └─ SessionStore.announce() → 发射 'session/created'
    │
    ├─ session.append('turn/start', { turn: 1 })
    │   ├─ snapshotJsonValue() → 验证 JSON 可序列化
    │   ├─ deepFreeze() → 冻结事件
    │   └─ 通知 session/event 观察者
    │
    └─ session.append('user/message', userMessage, { surfaceOp: 'append' })
        ├─ SurfaceManager.validateNext()
        └─ 持久化插件接收事件并缓冲
```

### 阶段五：Agent Loop 启动

```
AgentLoop.create(sessionId, options)
    │
    ├─ new ReactLoopAgent(loopCtx, sessionId, options, session)
    │   ├─ 创建 Inbox
    │   ├─ phase = { kind: 'idle', lastTurn: 0 }
    │   ├─ createScope() → Agent 级别的 Cordis 作用域
    │   └─ new RuntimeContextProjection()
    │
    ├─ wakeDriver()
    │   └─ setPhase({ kind: 'running', ... })
    │
    └─ kick()
        └─ while(await this.turn()) {}
```

### 阶段六：Turn 和 Step

```
turn()
    │
    ├─ session.append('turn/start', { turn: 1 })
    │
    └─ preStep('next-turn', { turn: 1, step: 1 })
        ├─ inbox.claim() → 获取待处理消息
        ├─ systemPrompt.assemble() → 组装系统提示词
        ├─ renderContextSections() → 渲染上下文
        ├─ runtimeContext.project() → 注入运行时上下文
        └─ dispatch.waterfall('agent/pre-step') → 插件拦截点
            │
            ▼ 如果决定进入
        session.append('step/start', { turn: 1, step: 1 })
            │
            ▼
        buildRequest(1, 1, tools, system, messages, signal)
            ├─ session.requestHeader() → 获取历史配置
            ├─ dispatch.waterfall('agent/request') → 插件修改配置
            ├─ llm.prepareCall() → 解析 LLM 适配器
            ├─ session.append('request/header', { header, reason })
            └─ 返回 GenerateOptions
```

### 阶段七：LLM 调用

```
llm.generate(request)
    │
    ├─ BlockAssembler.accumulate(chunk) → 逐 token 组装
    │
    ├─ session.append('assistant/chunk', { turn, step, chunk })
    │   每个 token 一个事件
    │
    ├─ createAssistantMessage(assembler) → 组装完整消息
    │
    └─ session.append('assistant/message', { turn, step, message, usage })
        │
        ▼ 如果 message 包含 tool_calls
        │
        ├─ session.append('tool/call', { turn, step, callId, name: 'write_file', arguments })
        │
        └─ executeToolCalls(ctx, session, turn, step, toolCalls, signal)
```

### 阶段八：工具执行

```
executeToolCalls()
    │
    ├─ 段1: validateSchema(tool.parameters, args)
    │   验证参数符合 JSON Schema
    │
    ├─ 段2: 参数规范化
    │   JSON 字符串 → 对象
    │
    ├─ 段3: 能力检查 (Capability Seam)
    │   ctx.get('fsTarget') → FsTarget 接口
    │
    ├─ 段4: 沙箱检查
    │   sandbox.wrapCommand() → bwrap/Landlock/Seatbelt 包装
    │
    ├─ 段5: 审批
    │   requestApproval({ tool: 'write_file', risk: 'medium' })
    │   fail-closed: 无处理器 → 拒绝
    │
    ├─ 段6: 执行
    │   fs.writeFile(resolvedPath, content)
    │   │
    │   └─ 如果启用了沙箱
    │       ├─ bwrap: 命名空间隔离
    │       ├─ Landlock: 文件系统访问控制
    │       └─ Seatbelt: macOS 沙箱 profile
    │
    └─ session.append('tool/result', { turn, step, message, meta: { diff } })
```

### 阶段九：模型继续

```
step 循环继续（target = 'next-step'）
    │
    ├─ preStep('next-step', { turn: 1, step: 2 })
    │
    ├─ llm.generate() → 模型返回 shell 调用
    │
    ├─ session.append('tool/call', { name: 'shell', arguments: 'python hello.py' })
    │
    └─ executeToolCalls()
        ├─ spawn('bash', ['-c', 'python hello.py'])
        │   如果有沙箱 → wrapCommand()
        ├─ 捕获 stdout: "Hello, World!"
        ├─ 捕获 exitCode: 0
        └─ session.append('tool/result', { message, meta: { exitCode, stdout } })
```

### 阶段十：Turn 结束

```
模型返回纯文本: "已创建 hello.py 并成功执行"
    │
    ├─ session.append('assistant/message', { message: finalMessage })
    │   没有 tool_calls → turn 结束
    │
    ├─ session.append('turn/end', { turn: 1, reason: { kind: 'completed' } })
    │
    ├─ 持久化 flush
    │   session/flush → 并行写入 JSONL/SQLite
    │
    └─ phase = { kind: 'idle', lastTurn: 1 }
```

## 关键设计模式总结

### 1. 事件溯源
所有状态变化记录为不可变事件。对话状态从事件日志派生。

### 2. 能力接缝
接口在消费者侧定义，实现在提供者侧，通过 DI 连接。

### 3. 补丁层叠
配置通过 YAML 补丁列表组合，后面层覆盖前面层。

### 4. fail-closed
安全系统在不确定时拒绝而非允许。

### 5. 深度防御
多层安全检查：Schema 验证 → 能力检查 → 沙箱 → 审批。

### 6. Proxy 依赖注入
Context 是 Proxy，属性读取走反射层，DI 对使用者透明。

## 本章小结

整个请求旅程经过了 10 个主要阶段：
1. CLI 解析 → `parseDshArgs`
2. Profile 组装 → `composeProfile`
3. Cordis 启动 → `boot` + `mountRootInclude`
4. Session 创建 → `SessionStore.create`
5. Agent Loop 启动 → `ReactLoopAgent` + `wakeDriver`
6. Turn/Step → `turn` + `preStep` + `buildRequest`
7. LLM 调用 → `llm.generate` + `BlockAssembler`
8. 工具执行 → 六段守卫管线
9. 模型继续 → step 循环
10. Turn 结束 → 持久化 flush

每个阶段都有明确的入口函数、处理逻辑、和出口函数。请求从一个模块流向下一个模块，每个模块做且只做自己的事情。

## 阅读思考题

1. 在整个调用链中，哪些步骤是同步的，哪些是异步的？这对错误处理有什么影响？
2. 如果在阶段八（工具执行）中沙箱拒绝了操作，整个调用链会如何回退？
3. 这条调用链中最可能出错的是哪个阶段？为什么？

---

**全书完**

感谢你跟着这个请求走完了 DeepSeek Harness 的完整旅程。希望这本书帮助你理解了 dsh 的架构设计和实现细节。如果你想深入某个模块，可以从对应的章节开始，沿着调用链继续探索。
