# 第 16 章 测试策略与可信度工程——凭什么敢让它在无人值守时自己跑

前面十五章,我们一路拆解 OpenClaw 是怎么"安全地跑、安全地被观察、安全地交出现场"的。但每一章其实都悬着一个没正面回答的问题:**这些安全承诺,凭什么相信它们今天成立、明天改了代码之后还成立?**

"fail-closed""脱敏不漏""路径不逃逸""约束只能收紧"——这些原则写在代码里、写在注释里,看起来都对。可"看起来对"和"持续被验证对"之间,隔着一整个工程鸿沟。一个 agent 系统最特殊的地方在于:它会在**无人值守**下自动干活。半夜没人盯着的时候,它在调工具、连远程节点、跑 shell。这种系统的信任不能靠"我读过代码觉得没问题",必须靠一套能**反复、自动、廉价地重新证明这些不变量**的机制。这一章的主角,就是这套机制。

我们会看到,OpenClaw 的测试不是事后补的覆盖率作业,而是把前面每一章的安全主张**翻译成可执行断言**的工程实践:它有 3628 个单元测试文件、近 80 个分片化的测试项目;它专门为"测试可信度"造了一批基础设施——隔离的临时环境、可冻结的时钟、可替换的 reader;它把"密钥必须脱敏""npm 安装必须 `--ignore-scripts`"这类安全规则,直接写成了断言辅助函数。可信度不是一种感觉,而是一条流水线的产出。

## 16.1 为可测试性而设计:依赖注入不是为了优雅

回看第 15 章那个支持导出的函数签名,有个细节当时一带而过:

```ts
export type DiagnosticSupportExportOptions = {
  // ...
  readLogTail?: typeof readConfiguredLogTail;
  readStatusSnapshot?: SupportSnapshotReader;
  readHealthSnapshot?: SupportSnapshotReader;
};
```

`readLogTail`、`readStatusSnapshot`、`readHealthSnapshot`——这三个"读取器"都是可选参数,默认用真实实现,但**允许调用方传入替身**。这不是为了代码好看,而是一个非常务实的可测试性设计:测试时,你可以传一个返回固定数据的假 reader,从而在**不碰真实文件系统、不依赖真实网关**的前提下,验证"导出逻辑对各种输入的处理是否正确"。

这就是依赖注入(DI)在这套代码里的真正用途——**把"难以控制的外部世界"(磁盘、网络、时钟)从核心逻辑里抽出来,变成可以替换的参数。** 一个把 `fs.readFileSync` 直接写死在内部的函数,几乎无法测试它在"文件损坏""权限不足""返回超大内容"时的行为;而一个接受 reader 参数的函数,测试可以轻松喂给它任何边界情形。

第 15 章我们还见过同样的模式:`buildDiagnosticSupportExport` 接受 `env`、`stateDir`、`now` 作为可选参数。`now?: Date` 尤其典型——把"当前时间"做成参数,测试就能传一个固定时间,让导出文件名、时间戳变得**可预测、可断言**。可测试性不是测试阶段才考虑的事,它从函数签名设计的第一刻就被编织进去了。这也呼应了第 14 章末尾那句:可观测性和安全一样,从第一行代码就被设计进骨架——可测试性同样如此。

## 16.2 测试的规模与分片:近 80 个项目,各管一摊

打开仓库根目录的 `vitest.config.ts`,它只是个转发器,真正的配置在 `test/vitest/` 下,核心是一份 `rootVitestProjects` 列表——**近 80 个独立的测试项目**,每个管一个领域:

```ts
export const rootVitestProjects = [
  "test/vitest/vitest.unit.config.ts",
  "test/vitest/vitest.boundary.config.ts",
  "test/vitest/vitest.gateway-core.config.ts",
  "test/vitest/vitest.gateway-server.config.ts",
  "test/vitest/vitest.secrets.config.ts",
  "test/vitest/vitest.runtime-config.config.ts",
  "test/vitest/vitest.plugins.config.ts",
  "test/vitest/vitest.logging.config.ts",
  // ...还有 contracts-channel-*、extension-* 几十个
] as const;
```

为什么要拆成这么多片,而不是一个大 `vitest` 跑全部?几个原因彼此交织:

- **并行与速度**:近 80 个项目可以并行调度。配置里专门导出了 `resolveDefaultVitestPool`、`resolveLocalVitestMaxWorkers`、`resolveLocalVitestScheduling` 这几个函数,说明测试的**池(进程/线程)、worker 数、调度策略都是被显式管理的**——测试套件本身被当成一个需要性能优化的工程对象。
- **隔离边界对应代码边界**:你能看到 `vitest.boundary.config.ts`(架构边界)、`vitest.contracts-channel-*.config.ts`(渠道契约)、`vitest.secrets.config.ts`(密钥)、`vitest.extension-*.config.ts`(每个外部渠道一个)。测试的切分**镜像了系统的模块切分**——网关有网关的测试项目,插件有插件的,每个 extension(Slack/Telegram/Matrix/Feishu…)各有一个独立项目。
- **失败定位**:当 CI 红了,你立刻知道是"gateway-server 这片崩了"还是"extension-telegram 这片崩了",而不是在一个几千测试的大泥球里大海捞针。

这种分片本身就是一种工程信号:**当一个系统认真对待测试时,测试套件会长得和系统架构一样有结构。** 它不是一堆散落的 `*.test.ts`,而是一个有项目、有领域、有调度策略的子系统。

## 16.3 测试隔离:每个测试都活在自己的小宇宙里

测试要可信,首先得**互不污染**。如果测试 A 改了某个全局状态、写了某个文件,而测试 B 恰好读到了,那测试结果就成了"看运行顺序的玄学"——这种 flaky 测试比没有测试更糟,因为它会慢慢侵蚀团队对整个测试套件的信任。OpenClaw 为隔离造了一整套 `src/test-utils/` 基础设施。

最典型的是 `temp-home.ts`,它为"重度依赖配置/状态目录"的测试创建一个**完全隔离的临时家目录**:

```ts
const HOME_ENV_KEYS = ["HOME", "USERPROFILE", "HOMEDRIVE", "HOMEPATH", "OPENCLAW_STATE_DIR"] as const;

export async function createTempHomeEnv(prefix: string): Promise<TempHomeEnv> {
  // ...
  const snapshot = captureEnv([...HOME_ENV_KEYS]);
  process.env.HOME = home;
  process.env.USERPROFILE = home;
  process.env.OPENCLAW_STATE_DIR = path.join(home, ".openclaw");
  // ...返回 { home, restore }
}
```

它把 `HOME`、`USERPROFILE`、`OPENCLAW_STATE_DIR` 等环境变量整体**快照、替换、最后还原**。这样一来,任何读写"用户配置目录"的代码,在测试里都会落到一个临时目录,既不会污染开发者真实的 `~/.openclaw`,也不会让两个测试互相看见对方的状态。返回的 `restore()` 保证测试结束后环境被精确复原——**借了就要还**。

配套的 `tracked-temp-dirs.ts` 更进一步,它的注释开门见山:"Tracks temporary directories created by tests so leaks can be detected."(跟踪测试创建的临时目录,以便检测泄漏。)它不只是创建临时目录,还**记录**所有创建过的目录,从而能在套件结束时检查"有没有谁忘了清理"。**连"测试自己是否干净"都被测试了**——这是把"卫生"这件事工程化的典型体现。

时间也是一种需要隔离的全局状态。`frozen-time.ts` 提供了冻结时钟的能力:

```ts
export function useFrozenTime(at: string | number | Date): void {
  vi.useFakeTimers();
  vi.setSystemTime(at);
}
```

任何断言时间戳、超时、定时器的测试,都可以把时间钉死在一个确定的点上。配合 16.1 节那个 `now?: Date` 参数,系统在两个层面都做到了"时间可控":既能从参数注入,也能从全局冻结。一个依赖真实墙上时钟的测试是不可复现的;把时间变成可控量,测试才有资格谈"确定性"。

## 16.4 把安全规则写成断言:可信度的硬核所在

前面讲的都是"如何让测试可信",但这一章真正的高潮是:**OpenClaw 把前几章那些安全主张,直接固化成了断言辅助函数。** 安全不再是注释里的承诺,而是测试里会失败的硬约束。

看 `auth-token-assertions.ts`,它把第 12 章"token 即身份"的规则写死了:

```ts
export function expectGeneratedTokenPersistedToGatewayAuth(params: {
  generatedToken?: string;
  authToken?: string;
  persistedConfig?: OpenClawConfig;
}) {
  expect(params.generatedToken).toMatch(/^[0-9a-f]{48}$/);
  expect(params.authToken).toBe(params.generatedToken);
  expect(params.persistedConfig?.gateway?.auth?.mode).toBe("token");
  expect(params.persistedConfig?.gateway?.auth?.token).toBe(params.generatedToken);
}
```

四条断言把一个安全不变量钉死:生成的网关 token **必须**是 48 位十六进制(对应 24 字节随机)、**必须**原样既返回又持久化、auth 模式**必须**是 `token`。哪天有人不小心改短了 token 长度、或者生成的和存的不一致,这个断言立刻红。第 12 章那句"token 即身份"于是从一句设计哲学,变成了一条 CI 会拦住的红线。

更精彩的是 `exec-assertions.ts` 里这条,它守护的是第 8 章/第 13 章供应链安全的核心规则——**安装第三方包绝不能执行其安装脚本**:

```ts
export function expectSingleNpmInstallIgnoreScriptsCall(params: {
  calls: Array<[unknown, { cwd?: string } | undefined]>;
  expectedTargetDir: string;
}) {
  const npmCalls = params.calls.filter((call) => Array.isArray(call[0]) && call[0][0] === "npm");
  expect(npmCalls.length).toBe(1);
  // ...
  expect(argv).toEqual(["npm", "install", "--omit=dev", "--loglevel=error", "--ignore-scripts"]);
  // ...
  expect(path.basename(cwd)).toMatch(/^\.openclaw-install-stage-/);
}
```

这条断言逐字检查了安装命令:必须**恰好调用一次** npm、参数必须**精确等于** `["npm", "install", "--omit=dev", "--loglevel=error", "--ignore-scripts"]`、安装目录必须是隔离的 `.openclaw-install-stage-` 暂存目录。`--ignore-scripts` 是这里的灵魂——它阻止恶意 npm 包在安装阶段执行任意代码(供应链攻击最常见的入口)。把它写进 `toEqual` 的精确匹配里,意味着**任何人想偷偷去掉这个标志,测试都会立刻失败**。安全规则不再依赖 reviewer 的警觉,而是变成了机器每次都会检查的契约。

这就是"可信度工程"的精髓:**你最在乎的那些不变量,要么能被一条会失败的断言守护,要么它就只是一句口号。** OpenClaw 的做法是给关键安全规则都配上专门的断言辅助函数,让它们在整个测试套件里被反复复用、反复验证。这里还有一个跨平台的细节——`exec-assertions.ts` 专门处理了 macOS 把 `/tmp` 暴露成 `/private/var/` 的怪癖(`normalizeDarwinTmpPath`),并用 `fs.realpathSync.native` 规范化符号链接后再比较。**连"断言本身在不同操作系统上是否成立"都被照顾到了**,否则一个在 Linux 上通过、在 macOS 上偶发失败的断言,同样会侵蚀信任。

## 16.5 集成测试:让边界情形在真实链路上跑一遍

单元测试用替身换掉外部世界,跑得快、定位准,但它有个固有盲区:**替身可能和真实实现行为不一致。** 所以 OpenClaw 还有一层集成测试(仓库里有 20 个 `*.integration.test.ts`),让关键链路在尽量真实的环境里端到端跑一遍。

第 14 章我们提到过 `diagnostic-stuck-session-recovery.integration.test.ts` 这类文件——"卡死会话恢复"这种涉及定时器、状态机、多个协调者的复杂行为,光靠单元 mock 很难验证它们**协同**起来是否正确,必须放到接近真实的运行时里跑。这两层测试是互补的:

- **单元测试**回答"每个零件单独看对不对":喂边界输入、断言精确输出,数量巨大(3628 个)、速度快。
- **集成测试**回答"零件装到一起还转不转":数量少(20 个)、更慢、更贵,但能抓住单元测试因为用了替身而漏掉的"接缝处的 bug"。

这个 3628 : 20 的比例本身就是一个清醒的工程判断——**绝大多数信心来自快而廉价的单元测试,少量昂贵的集成测试只用在"接缝最容易出事"的地方。** 这正是经典的"测试金字塔":底座是海量快速单元测试,顶端是少量端到端测试,而不是反过来堆一堆又慢又脆的端到端用例。

## 16.6 测试即文档:边界情形被写成了规格

最后一个常被忽视、却极其重要的作用:**测试是这套系统最诚实、最不会过期的文档。** 我们在前面每一章拆解的那些边界行为——第 12 章"未配置等于不批准"、第 14 章"拒绝 NaN/Infinity/负数"、第 15 章"路径逃逸要被挡住"——它们的**确切契约**,往往不在散文式的文档里,而在对应的 `*.test.ts` 里。

为什么测试比文档更可信?因为**文档会和代码脱节,而测试不会**:一旦行为变了,测试要么跟着改、要么就红了;一份描述旧行为的过期文档却可以静静地误导你好几年。当你想确认"OpenClaw 在某个边界情形下到底怎么处理"时,最权威的答案不是 README,而是那条断言。这也是为什么前面每一章的"动手实验"我都反复建议你去读对应的源码和测试——**测试就是被写成可执行形式的规格说明**。

从 `secret-ref-test-vectors.ts`(密钥引用的测试向量)、`secret-file-fixture.ts`(密钥文件夹具)这些文件名就能看出:连"密钥应该长什么样、不该泄露成什么样"都被沉淀成了可复用的测试向量。安全规则一旦被写成测试向量,就变成了团队共享的、不会随人员流动而丢失的知识资产。一个人离职,他脑子里的"这里为什么要这么写"会消失;但他写下的断言不会。

## 本章小结

这一章我们回答了贯穿全书的那个隐含问题——**凭什么敢信任一个无人值守自动运行的 agent**:

1. **为可测试性而设计**:`readLogTail`/`readStatusSnapshot`/`now` 等可注入参数,把磁盘、网络、时钟从核心逻辑里抽出来,让边界情形可被廉价地喂给函数。可测试性从函数签名设计的第一刻就开始。
2. **规模与分片**:3628 个单元测试文件、近 80 个 vitest 项目,切分镜像系统架构(boundary/secrets/gateway/plugins/extension-*),池与调度被显式管理,失败可被精确定位。
3. **测试隔离**:`temp-home` 快照-替换-还原环境变量、`tracked-temp-dirs` 检测临时目录泄漏、`frozen-time` 冻结时钟——把磁盘、状态、时间这些全局量都变成可控、可复原的,杜绝 flaky。
4. **安全规则写成断言**:`auth-token-assertions` 钉死 token 形态与持久化、`exec-assertions` 精确匹配 `--ignore-scripts` 安装命令——把第 8/12/13 章的安全承诺变成会失败的红线;连 macOS 路径怪癖都被规范化处理。
5. **集成测试补盲区**:20 个 `*.integration.test.ts` 在真实链路上验证单元 mock 漏掉的"接缝 bug",3628:20 的比例体现测试金字塔的清醒取舍。
6. **测试即文档**:边界行为的确切契约活在测试里而非散文文档里;测试不会和代码脱节,密钥测试向量等把安全知识沉淀成不随人员流动丢失的资产。

贯穿全章、也贯穿全书的那条主线在这里收束成一句话:**OpenClaw 的可信度不是来自"代码看起来对",而是来自"一套快速、隔离、把关键不变量写成断言的流水线持续证明它对"。** 一个敢在深夜自动跑的 agent,背后必然站着一套敢于在每次提交时重新质问自己的测试。

## 动手实验

> 以下实验基于本章引用的源码文件,建议在你 clone 的 OpenClaw 仓库里对照阅读与验证。

**实验一:体会依赖注入如何解锁边界测试。**
读 `src/logging/diagnostic-support-export.ts` 里 `DiagnosticSupportExportOptions` 的 `readLogTail`/`readStatusSnapshot`/`now` 三个可选参数。设想你要测"当日志读取抛异常时,导出会不会优雅降级"(对应第 15 章的 `failedLogTail`)。回答:如果这三个都是函数内部写死的真实实现,你要怎样才能制造一次"日志读取失败"?有了可注入的 `readLogTail` 之后又变得多简单?再设想测"导出文件名里的时间戳",`now?: Date` 参数让这件事从"不可断言"变成了什么?

**实验二:读懂测试分片的结构。**
打开 `test/vitest/vitest.config.ts` 的 `rootVitestProjects` 列表。挑出 5 个你在前面章节认得出对应子系统的项目(例如 `vitest.secrets.config.ts`、`vitest.boundary.config.ts`、`vitest.gateway-server.config.ts`),分别说出它大致守护的是哪一章讲过的能力。然后回答:为什么把测试拆成近 80 个项目而不是一个大套件?从"并行速度""失败定位""隔离边界"三个角度各说一条理由。

**实验三:验证一条安全断言的威力。**
读 `src/test-utils/exec-assertions.ts` 的 `expectSingleNpmInstallIgnoreScriptsCall`。假设有个开发者为了"修一个安装失败的 bug",把安装参数从 `["npm","install","--omit=dev","--loglevel=error","--ignore-scripts"]` 改成了去掉 `--ignore-scripts`。推演:这个断言会发生什么?它在 CI 里扮演了什么角色?把这条断言和第 13 章供应链审计、第 8 章沙箱执行联系起来,说说"把安全规则写进精确匹配的断言"为什么比"在 code review 时记得检查"更可靠。

**实验四:用测试当文档。**
任选前面章节的一个安全不变量(例如第 14 章"`firstFiniteTalkEventNumber` 拒绝 NaN/Infinity/负数",或第 15 章"路径逃逸被挡住")。在仓库里找到对应的 `*.test.ts`,读它的断言。回答:测试里描述的确切行为,和你从正文里读到的理解一致吗?有没有哪个边界情形是测试覆盖了、但你之前没想到的?最后总结:为什么说"想确认 OpenClaw 在某个边界到底怎么处理,读测试比读文档更权威"?

## 下一章预告

到这一章,我们已经把 OpenClaw 这套 agent 系统从里到外拆了个透:启动与运行时、agent loop、上下文引擎、工具系统、沙箱执行、子 agent、网关路由、配置热重载、节点与设备配对、插件与 MCP、可观测性与遥测、诊断导出,以及刚刚讲完的测试与可信度。我们见过的每一个设计决定背后,几乎都站着同一组反复出现的原则:fail-closed、错误即文档、约束只能收紧、能扩展不能削弱核心、把不变量固化进类型与测试。第 17 章——也是本书的收束章——我们将跳出单个模块,把这些散落在各章的设计哲学**收拢成一套可迁移的"构建可信 agent 的工程心法"**:当你要从零搭建自己的 agent 时,从 OpenClaw 这十六章里到底能带走哪些不分语言、不分框架的通用经验?我们将一起回答这本书最初的那个问题——一个值得信任的 agent,到底是被什么样的工程纪律塑造出来的。
