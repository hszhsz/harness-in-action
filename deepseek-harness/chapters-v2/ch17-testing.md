# 第 17 章 测试策略：覆盖完整请求路径

上一章我们看了远程通信和可观测性。现在我们已经追踪了请求从 CLI 输入到工具执行的完整旅程。但如何确保这条路径的每一步都是正确的？这一章我们看 dsh 的测试策略——如何用测试覆盖这条完整路径。

## 请求到达这里时发生了什么

dsh 使用 Vitest 作为测试框架，测试分布在各个包中。测试策略分为几层：

1. **单元测试**：测试单个函数/类
2. **集成测试**：测试模块间的交互
3. **端到端测试**：测试完整的请求路径

## 关键代码路径

### 测试基础设施

每个包的 `tests/` 目录包含该包的测试文件。根目录的 `vitest.config.ts` 配置了全局测试选项。

测试中大量使用 mock 和 fake：

```typescript
// 模拟 LLM 适配器
const fakeLlm = {
  prepareCall: vi.fn().mockResolvedValue({ config: { provider: 'test', model: 'test' } }),
  generate: vi.fn().mockImplementation(async function* () {
    yield { type: 'text', content: 'Hello' }
  }),
}

// 模拟文件系统
const fakeFs: FsTarget = {
  readFile: vi.fn().mockResolvedValue('file content'),
  writeFile: vi.fn().mockResolvedValue(undefined),
  editFile: vi.fn().mockResolvedValue(undefined),
}
```

### Session 测试

Session 的测试覆盖了：
- 事件追加和验证
- Surface 操作（append、replace）
- 事件溯源和重放
- 头部验证和冻结
- fork 和 seed

```typescript
test('append validates JSON serializability', () => {
  const session = Session.create(SessionId('test'))
  expect(() => session.append('user/message', {
    id: 'msg-1',
    role: 'user',
    content: [{ type: 'text', text: 'hello' }],
    source: { kind: 'human' },
  } as UserMessage)).not.toThrow()
})

test('append rejects non-serializable data', () => {
  const session = Session.create(SessionId('test'))
  expect(() => session.append('user/message', {
    id: 'msg-1',
    role: 'user',
    content: [{ type: 'text', text: 'hello', extra: BigInt(42) }],
    source: { kind: 'human' },
  } as any)).toThrow()
})
```

### Agent Loop 测试

Agent Loop 的测试模拟了完整的 turn 循环：

```typescript
test('full turn with tool call', async () => {
  const session = createTestSession()
  const agent = new ReactLoopAgent(ctx, sessionId, options, session)
  
  // 注入用户消息
  agent.send(createUserMessage('write hello.py'), 'next-turn', true)
  
  // 等待 turn 完成
  await agent.whenIdle()
  
  // 验证事件序列
  const events = session.events
  expect(events[0].type).toBe('turn/start')
  expect(events[1].type).toBe('user/message')
  expect(events[2].type).toBe('step/start')
  // ...
})
```

### 端到端测试

端到端测试模拟从 CLI 到工具执行的完整路径：

```typescript
test('dsh --profile headless creates and runs hello.py', async () => {
  // 设置 mock LLM 返回 write_file 调用
  mockLlm.generate = mockGenerate([
    { type: 'tool-call', name: 'write_file', arguments: { path: 'hello.py', content: 'print("hello")' } },
    { type: 'text', content: 'Done!' },
  ])
  
  // 启动 profile
  const { ctx } = await runProfile({
    environment: createTestEnvironment(),
    profile: 'headless',
    patchFiles: [],
    args: ['write a hello.py'],
  })
  
  // 等待完成
  await waitForIdle(ctx)
  
  // 验证文件被写入
  expect(mockFs.writeFile).toHaveBeenCalledWith(
    expect.stringContaining('hello.py'),
    'print("hello")',
  )
  
  // 验证 Session 事件
  const session = ctx.sessions.list()[0]
  const events = session.events
  expect(events.some(e => e.type === 'tool/call')).toBe(true)
  expect(events.some(e => e.type === 'tool/result')).toBe(true)
})
```

### Snapshot 测试

对于配置 dump（`dsh --dump-config`），dsh 使用 snapshot 测试来验证输出格式：

```typescript
test('dump-config matches snapshot', async () => {
  const output = await runDumpConfig('headless', true, [])
  expect(output).toMatchFileSnapshot('./fixtures/headless-default.yml')
})
```

### Fail-loud 测试

`installFailLoud` 的测试验证了未捕获异常的处理：

```typescript
test('fail-loud exits on unhandled rejection', async () => {
  const fakeProcess = createFakeProcess()
  installFailLoud('dsh', fakeProcess)
  
  fakeProcess.emit('unhandledRejection', new Error('test error'))
  
  expect(fakeProcess.stderr.write).toHaveBeenCalledWith(
    expect.stringContaining('fatal load failure'),
  )
  expect(fakeProcess.exit).toHaveBeenCalledWith(1)
})
```

## 调用链：从这里到下一步

测试覆盖了完整的请求路径。现在让我们回顾整个旅程，把所有章节串联起来。

**下一步：最后一章——完整调用链回顾。我们将把 18 章串联成一条完整的调用链，从终端输入到任务完成。**

## 关键设计决策

1. **分层测试**：单元、集成、端到端三层测试，每层覆盖不同的关注点。

2. **Mock 而非真实 API**：LLM 和文件系统用 mock，测试不依赖外部服务。

3. **事件序列验证**：通过检查 Session 事件序列来验证 Agent 行为，而非检查内部状态。

4. **Snapshot 测试**：对于配置输出，用 snapshot 保证格式稳定性。

## 本章小结

- 测试分三层：单元、集成、端到端
- LLM 和文件系统用 mock，不依赖外部服务
- Agent Loop 测试通过事件序列验证行为
- 端到端测试模拟从 CLI 到工具执行的完整路径
- Snapshot 测试保证配置输出格式稳定

## 阅读思考题

1. 为什么 Agent Loop 测试通过事件序列而非内部状态来验证行为？
2. 端到端测试中，如何模拟 LLM 的流式响应？
3. 如果新增一个工具类型，需要在哪些测试层添加测试？

> **下一章预告**：终于到了最后一章！我们将把 18 章的内容串联成一条完整的调用链——从用户按下回车到任务完成，每一步都精确到函数名。
