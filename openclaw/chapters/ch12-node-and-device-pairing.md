# 第 12 章 节点与设备配对——如何让一个陌生人安全地加入总线

## 12.1 从"调用谁"到"谁能进来"

第 10 章我们沿着 `callGateway` 走了一遍：每一次调用都收敛到同一条网关总线上，被一张方法表逐条裁决。第 11 章我们看到这条总线如何在不停机的情况下安全地吞下新配置。但这两章都隐含了一个前提——**调用方已经在总线上了**。

本章要回答的是上一个问题：一个此前从未出现过的实体——一台新手机、一个远程节点(node)——想第一次接入这条总线时,会发生什么?它要如何证明自己可信?它拿到的令牌(token)能干什么、不能干什么?令牌如何轮换、如何吊销?当一台设备被踢出去时,它已经塞进 socket 缓冲区的那些请求帧又该怎么办?

还记得第 10 章方法表里那一整片 `operator.pairing` scope 的方法吗?`node.pair.approve`、`device.token.rotate`、`device.token.revoke`——它们全都活在这一章。配对(pairing)是整个信任体系的入口,也是最危险的地方:它是系统里唯一一个"把外人变成自己人"的操作。OpenClaw 在这里的设计哲学只有一句话——**默认关闭(fail-closed),每一道门都要主动证明才能打开**。

## 12.2 配对到底在配什么:令牌就是身份

先看最底层。一个配对令牌是什么?答案出奇地朴素[[pairing-token.ts]](https://github.com/openclaw/openclaw/blob/main/src/infra/pairing-token.ts):

```ts
export const PAIRING_TOKEN_BYTES = 32;

export function generatePairingToken(): string {
  return randomBytes(PAIRING_TOKEN_BYTES).toString("base64url");
}

export function verifyPairingToken(provided: string, expected: string): boolean {
  if (provided.trim().length === 0 || expected.trim().length === 0) {
    return false;
  }
  return safeEqualSecret(provided, expected);
}
```

令牌就是 32 字节的密码学随机数,编码成 URL-safe 的 base64url 字符串。它没有结构、没有签名、不携带任何 claim——它只是一个**持有即证明(bearer)**的秘密。谁握着它,谁就是那台设备。

正因为它是纯秘密,验证时有两个细节值得停下来看:

第一,**空令牌一律拒绝**。`provided` 或 `expected` 任意一方去掉空白后长度为 0,直接返回 `false`。这堵住了一个经典漏洞:如果某台设备的令牌字段意外为空,而验证逻辑又用 `===` 比较,那么传一个空字符串就能"匹配"。这里用前置判断把这条路彻底封死。

第二,**常量时间比较**。它不用 `===`,而是用 `safeEqualSecret`。普通字符串比较会在第一个不同的字符处提前返回,攻击者可以通过测量响应时间,一个字节一个字节地把令牌猜出来(时序侧信道攻击)。`safeEqualSecret` 保证无论令牌在哪一位不同,比较耗时都一样。

记住这个朴素的事实:**令牌即身份**。本章后面所有关于"轮换"和"吊销"的复杂性,本质上都是在管理这个秘密的生命周期——什么时候换一个新的、什么时候让旧的失效、新的能不能透露给请求者。

## 12.3 节点自动批准:一道层层设防的 fail-closed 闸门

设备配对默认需要人工批准(operator 点一下"approve")。但有一类场景必须自动化:在一个受信任的内网里,运维不想为每台新上线的节点都手动点一次批准。OpenClaw 为此开了一个口子——**可信 CIDR 自动批准**。

但这个口子开得极其克制。我们来读 `shouldAutoApproveNodePairingFromTrustedCidrs`,它是一个布尔函数,而它的函数体几乎全部由 `return false` 组成[[node-pairing-auto-approve.ts]](https://github.com/openclaw/openclaw/blob/main/src/gateway/node-pairing-auto-approve.ts):

```ts
export function shouldAutoApproveNodePairingFromTrustedCidrs(params: {...}): boolean {
  if (params.existingPairedDevice) return false;
  if (params.role !== "node") return false;
  if (params.reason !== "not-paired") return false;
  if (params.scopes.length > 0) return false;
  if (params.hasBrowserOriginHeader || params.isControlUi || params.isWebchat) return false;
  if (
    params.reportedClientIpSource === "none" ||
    params.reportedClientIpSource === "loopback-trusted-proxy"
  ) return false;
  if (!params.reportedClientIp) return false;

  const autoApproveCidrs = params.autoApproveCidrs
    ?.map((entry) => entry.trim())
    .filter((entry) => entry.length > 0);
  if (!autoApproveCidrs || autoApproveCidrs.length === 0) return false;

  return isTrustedProxyAddress(params.reportedClientIp, autoApproveCidrs);
}
```

这是一个标准的 **fail-closed 闸门**:函数从头到尾在寻找拒绝的理由,只有当**所有**拒绝条件都不成立时,才走到最后一行做唯一一次"放行"判断。让我们把这八道门逐一拆开,理解每一道门挡的是什么:

| 门 | 条件 | 挡住的攻击/误用 |
| --- | --- | --- |
| ① 已配对 | `existingPairedDevice` | 自动批准**只用于首次入网**,绝不用于已有设备 |
| ② 非 node 角色 | `role !== "node"` | 只有 node 能走这条路;operator/admin 永远要人工 |
| ③ 非首次 | `reason !== "not-paired"` | 角色升级、scope 升级、元数据变更一律拒绝 |
| ④ 带 scope | `scopes.length > 0` | 自动批准的节点只能拿"空 scope",不能自带权限 |
| ⑤ 浏览器路径 | `hasBrowserOriginHeader \|\| isControlUi \|\| isWebchat` | 任何来自浏览器/控制台/网页聊天的请求都不自动批准 |
| ⑥ IP 来源不可信 | source 为 `none` 或 `loopback-trusted-proxy` | 拿不到真实来源 IP,或只是环回代理,都不算数 |
| ⑦ 无来源 IP | `!reportedClientIp` | 连 IP 都没有,谈何 CIDR 匹配 |
| ⑧ 未配置白名单 | `autoApproveCidrs` 为空 | **没配置就等于关闭**——这是最关键的一道默认门 |

只有穿过这八道门,才会执行最后一行:`isTrustedProxyAddress(reportedClientIp, autoApproveCidrs)`,检查来源 IP 是否落在配置的可信 CIDR 内。

第 ③ 道门特别值得说。它检查的 `reason` 类型只有四个值[[node-pairing-auto-approve.ts]](https://github.com/openclaw/openclaw/blob/main/src/gateway/node-pairing-auto-approve.ts):

```ts
export type NodePairingAutoApproveReason =
  | "not-paired"       // 首次入网——唯一允许自动批准的
  | "role-upgrade"     // 想提升角色
  | "scope-upgrade"    // 想提升权限
  | "metadata-upgrade";// 想改元数据
```

文件头的注释把意图写得明明白白:"Allows first-time node pairing from configured CIDRs while rejecting upgrades/browser paths."(允许来自配置 CIDR 的首次节点配对,但拒绝任何升级路径与浏览器路径。)

这正是一个反复出现的设计准则:**入门可以方便,升级必须严格**。第一次进门,如果你来自可信内网、不带任何权限、明确是个 node,系统愿意替运维点一次"批准";但只要你想**多要一点点**——换个角色、加个 scope、改个标识——这条快速通道立刻关闭,把你打回人工审批。约束只能收紧,不能放松。

### IP 来源的分类:别把代理的脸当成访客的脸

第 ⑥ 道门依赖一个分类函数 `resolveNodePairingClientIpSource`,它回答的问题是:"我报告的这个 `clientIp`,到底是怎么来的、可不可信?"[[node-pairing-auto-approve.ts]](https://github.com/openclaw/openclaw/blob/main/src/gateway/node-pairing-auto-approve.ts)

```ts
export function resolveNodePairingClientIpSource(params: {...}): NodePairingAutoApproveClientIpSource {
  if (!params.reportedClientIp) return "none";
  if (!params.hasProxyHeaders || !params.remoteIsTrustedProxy) return "direct";
  return params.remoteIsLoopback ? "loopback-trusted-proxy" : "trusted-proxy";
}
```

四种来源:`direct`(客户端直连,IP 可信)、`trusted-proxy`(经过可信代理转发,代理填的头可信)、`loopback-trusted-proxy`(代理本身是环回的)、`none`(没有 IP)。自动批准只接受 `direct` 和 `trusted-proxy`;`none` 和 `loopback-trusted-proxy` 被第 ⑥ 道门拒绝。

为什么要把环回代理单独拎出来拒绝?因为如果代理跑在和网关同一台机器上(环回),那么"来源 IP"这个信息其实是代理自己填的、未经外部网络验证的——它无法证明"请求真的来自那个内网地址"。把这种情况归类为不可信,避免了"本地伪造来源 IP 绕过 CIDR 白名单"的攻击。

## 12.4 节点配对的请求-批准生命周期

自动批准是特例,常态是"请求 → 待审 → 批准/拒绝"。我们沿着 `nodeHandlers` 里的几个方法走一遍这个生命周期[[server-methods/nodes.ts]](https://github.com/openclaw/openclaw/blob/main/src/gateway/server-methods/nodes.ts)。

**`node.pair.request`**:一个节点带着自己的"画像"(nodeId、displayName、platform、version、caps、commands、permissions、remoteIp 等)来敲门。处理器把这些字段塞进 `requestNodePairing`,生成一条 pending 记录。这里有一个细节:如果同一个节点用新的画像再次请求,旧的请求会被**取代(superseded)**,系统会为每条被取代的旧请求广播一条 `node.pair.resolved`(decision 为 `rejected`):

```ts
for (const superseded of result.superseded ?? []) {
  context.broadcast("node.pair.resolved", {
    requestId: superseded.requestId, nodeId: superseded.nodeId,
    decision: "rejected", ts: resolvedAt,
  }, { dropIfSlow: true });
}
if (result.status === "pending" && result.created) {
  context.broadcast("node.pair.requested", result.request, { dropIfSlow: true });
}
```

为什么要取代而不是堆积?因为一个节点同时挂着多条互相矛盾的待审请求,会让 operator 无所适从——到底批哪一条?取代机制保证"一个节点最多一条有效待审请求",审批界面永远干净。注意广播都带 `{ dropIfSlow: true }`:这是第 10 章讲过的"通知尽力而为"——配对解析事件是状态同步,慢客户端宁可丢掉也不能拖垮总线。

**`node.pair.approve`**:这是 `operator.pairing` 家族的核心。处理器开头有一行注释道破了它的安全姿态[[server-methods/nodes.ts]](https://github.com/openclaw/openclaw/blob/main/src/gateway/server-methods/nodes.ts):

```ts
// Intentionally fail closed for RPC callers without an explicit scoped session.
const callerScopes = Array.isArray(client?.connect?.scopes) ? client.connect.scopes : [];
```

拿不到调用方的 scope 数组,就当成空数组——**故意 fail closed**。然后把 `callerScopes` 交给 `approveNodePairing` 去做真正的授权判断。如果返回 `status === "forbidden"`,处理器会回一个带 `missingScope` 的错误,告诉调用方"你缺了哪个 scope"。这又是"错误即文档"——拒绝里带着可操作的修复信息。

**审批所需的 scope 不是固定的,而是按节点想干什么动态升级的**。这是本章最精巧的一处设计,藏在 `resolveNodePairApprovalScopes` 里[[node-pairing-authz.ts]](https://github.com/openclaw/openclaw/blob/main/src/infra/node-pairing-authz.ts):

```ts
export function resolveNodePairApprovalScopes(commands: unknown): NodeApprovalScope[] {
  const normalized = Array.isArray(commands) ? commands.filter(...) : [];
  if (normalized.some((command) => NODE_SYSTEM_RUN_COMMANDS.some((allowed) => allowed === command))) {
    return [OPERATOR_PAIRING_SCOPE, OPERATOR_ADMIN_SCOPE];  // 想跑系统命令 → 要 admin
  }
  if (normalized.length > 0) {
    return [OPERATOR_PAIRING_SCOPE, OPERATOR_WRITE_SCOPE];  // 想跑任意命令 → 要 write
  }
  return [OPERATOR_PAIRING_SCOPE];                          // 纯配对 → 只要 pairing
}
```

三个台阶,清清楚楚:

- **纯配对**(不声明任何 command):只需 `operator.pairing`。
- **声明了 command**:还需要 `operator.write`——因为批准它就等于授予它执行命令的能力。
- **声明了系统级命令**(命中 `NODE_SYSTEM_RUN_COMMANDS`):还需要 `operator.admin`——能跑系统命令的节点,危险性等同于管理员,所以审批它的人也必须是管理员。

这是**最小权限**的镜像应用:批准一个实体所需的权限,等于这个实体将获得的能力。一个只想报告状态的传感器节点,任何有 pairing scope 的运维都能批;但一个想在主机上执行 shell 的节点,只有 admin 才有资格放它进来。权限的"重量"在审批环节就被精确称量。

批准成功后,处理器会用节点声明的能力面去更新注册表(`updateSurface`),并经过 `resolveNodeCommandAllowlist` + `normalizeDeclaredNodeCommands` 二次过滤——节点**声明**想要的命令,不等于它**最终拿到**的命令,后者还要被网关侧的 allowlist 收口。最后广播 `node.pair.resolved`(decision `approved`)。

**`node.pair.reject` / `node.pair.remove`**:拒绝一条待审请求,或移除一个已配对节点。`remove` 比 `reject` 多做几件清理:从 `pendingNodeActionsById` 删除挂起动作、把注册表里的能力面清空(`caps: [], commands: [], permissions: undefined`)、调用 `removeRemoteNodeInfo`,然后广播 decision `removed`。**移除即彻底失能**——能力面被清空意味着即使该节点的连接还活着,它也什么都做不了。

**`node.pair.verify`**:用 `verifyNodeToken(nodeId, token)` 验证一个节点令牌是否有效。这就是 12.2 那个 bearer 验证逻辑在节点侧的入口。

## 12.5 设备的自服务授权模型:你只能管你自己

节点的批准是"运维管节点";设备(device,通常指用户自己的手机/电脑)还多了一层——**自服务(self-service)**:设备可以管理自己,而不必每件事都求助 admin。但"管理自己"和"管理别人"之间必须有一道清晰的墙。这堵墙建在 `devices.ts` 的几个授权函数上[[server-methods/devices.ts]](https://github.com/openclaw/openclaw/blob/main/src/gateway/server-methods/devices.ts)。

第一块基石:**只有设备令牌认证才能证明所有权**。

```ts
function resolveDeviceSessionAuthz(client: GatewayClient | null): DeviceSessionAuthz {
  const callerScopes = Array.isArray(client?.connect?.scopes) ? client.connect.scopes : [];
  const rawCallerDeviceId = client?.connect?.device?.id;
  const callerDeviceId =
    // Plain shared-auth connections may report device metadata, but only
    // device-token auth proves ownership for self-service pairing actions.
    client?.isDeviceTokenAuth && typeof rawCallerDeviceId === "string" && rawCallerDeviceId.trim()
      ? rawCallerDeviceId.trim()
      : null;
  return { callerDeviceId, callerScopes, isAdminCaller: callerScopes.includes("operator.admin") };
}
```

注释点出了一个微妙但致命的区别:**一个普通的 shared-auth 连接也可能在握手里"报告"自己的 device id,但只有 device-token 认证才真正证明所有权**。所以 `callerDeviceId` 只在 `client.isDeviceTokenAuth` 为真时才被采纳,否则为 `null`。这堵住了"我随便填一个别人的 device id,就声称自己是那台设备"的伪造——光说没用,你得拿那台设备的令牌连进来。

第二块基石:**跨设备管理默认禁止**。

```ts
function deniesCrossDeviceManagement(authz: DeviceManagementAuthz): boolean {
  return Boolean(
    authz.callerDeviceId &&
    authz.callerDeviceId !== authz.normalizedTargetDeviceId &&
    !authz.isAdminCaller,
  );
}
```

一个用设备令牌连进来的调用方,如果目标设备不是它自己,而它又不是 admin——拒绝。翻译成大白话:**你可以管你自己,admin 可以管所有人,但你不能管别人**。

第三块基石:**非 admin 只能碰 operator 角色**。

```ts
function deniesDeviceTokenRoleManagement(authz, targetRole): boolean {
  const normalizedTargetRole = targetRole.trim();
  if (!normalizedTargetRole || authz.isAdminCaller) return false;
  return normalizedTargetRole !== "operator";
}
```

即使是管自己,一个普通设备也只能管理自己的 `operator` 角色令牌;想动 admin 之类的高权角色令牌,必须本身是 admin。这把"自服务"的范围牢牢框死在低权限角色里。

这三块基石组合起来,`device.pair.remove`、`device.token.rotate`、`device.token.revoke` 全都套用同一套判断:**先查跨设备(`deniesCrossDeviceManagement`),再查角色越权(`deniesDeviceTokenRoleManagement`),两关都过才放行**。而 `device.pair.approve`/`reject` 则额外校验"待审请求的 deviceId 是否等于 callerDeviceId"(`device-ownership-mismatch`),并对非 operator 角色的请求要求 admin(`role-management-requires-admin`)。同一套所有权语义,贯穿所有设备操作。

## 12.6 令牌轮换与吊销:只能收紧,不能透露给别人

设备令牌不是一次发放永久有效,它有完整的生命周期。我们看轮换 `rotateDeviceToken`,它揭示了两条关键不变量[[device-pairing.ts]](https://github.com/openclaw/openclaw/blob/main/src/infra/device-pairing.ts)。

**不变量一:轮换不能扩权。**

```ts
const approvedScopes = resolveApprovedDeviceScopeBaseline(device);
if (!approvedScopes) {
  return { ok: false, reason: "missing-approved-scope-baseline" };
}
if (!scopesWithinApprovedDeviceBaseline({ role, scopes: requestedScopes, approvedScopes })) {
  return { ok: false, reason: "scope-outside-approved-baseline" };
}
```

每台设备在被批准时确立了一个**已批准 scope 基线(approved scope baseline)**。轮换令牌时,新令牌请求的 scope 必须落在这条基线之内,否则返回 `scope-outside-approved-baseline`。这意味着:**轮换可以换密钥、可以缩小权限,但永远无法越过当初审批时给定的上限**。如果连基线都不存在(`missing-approved-scope-baseline`),直接拒绝——没有基线就没有授权依据,默认关闭。

这正是贯穿全书的那条铁律:**约束只能收紧,不能放松**。一台设备入网时被授予了什么,就是它一生权限的天花板;任何后续操作(轮换、重连)都不能突破这个天花板。要扩权?只能走重新审批,再过一遍 12.5 的所有权与角色检查。

随后还有一道调用方自身的 scope 校验:`resolveMissingRequestedScope` 检查调用方是否拥有它想授予的 scope,缺了就返回 `caller-missing-scope`——你不能把你自己都没有的权限发给一个令牌。

**不变量二:只有"自己轮换自己"才能拿到新令牌。**

这条逻辑在处理器侧[[server-methods/devices.ts]](https://github.com/openclaw/openclaw/blob/main/src/gateway/server-methods/devices.ts):

```ts
function shouldReturnRotatedDeviceToken(authz: DeviceManagementAuthz): boolean {
  // Admins can rotate any token, but only a device rotating itself receives
  // the new token in-band; other rotations are notification/invalidations.
  return Boolean(authz.callerDeviceId && authz.callerDeviceId === authz.normalizedTargetDeviceId);
}
```

```ts
respond(true, {
  deviceId, role: entry.role,
  ...(shouldReturnRotatedDeviceToken(authz) ? { token: entry.token } : {}),
  scopes: entry.scopes,
  rotatedAtMs: entry.rotatedAtMs ?? entry.createdAtMs,
}, undefined);
```

admin 能轮换任何设备的令牌,但**只有设备自己轮换自己时,响应里才带回新令牌(`token`)**。admin 替别人轮换,等于一次"通知 + 作废"——旧令牌失效了,但新令牌不会经由 admin 这条带外通道泄露出去。这避免了"管理员意外或恶意地截获用户设备令牌"的风险:设备的秘密,只回到设备自己手里。

吊销 `revokeDeviceToken` 更简单:校验调用方 scope 后,给目标令牌打上 `revokedAtMs` 时间戳。它不删除令牌记录,而是标记吊销时间——保留审计痕迹,让"这个令牌何时被吊销"成为可查询的事实。

## 12.7 关闭门时的竞态:在回包之前先让旧请求失效

轮换或吊销一个令牌,意味着持有旧令牌的连接必须被踢下线。但这里藏着一个非常隐蔽的竞态。三个设备令牌操作(remove/rotate/revoke)都用同一段注释和同一个手法解决它[[server-methods/devices.ts]](https://github.com/openclaw/openclaw/blob/main/src/gateway/server-methods/devices.ts):

```ts
// Mark affected clients invalid *before* responding so any RPCs already
// pipelined into their WS socket buffer are rejected at the per-request
// dispatch check, closing the race between queueMicrotask-scheduled
// disconnect and inflight frames.
context.invalidateClientsForDevice?.(deviceId.trim(), {
  role: entry.role, reason: "device-token-rotated",
});
respond(true, {...}, undefined);
queueMicrotask(() => {
  context.disconnectClientsForDevice?.(deviceId.trim(), { role: entry.role });
});
```

想象这个时序:网关决定吊销设备 X 的令牌,于是它要断开 X 的连接。但断开是异步的(`queueMicrotask`),不会立刻发生。而在断开真正执行之前的那个微小窗口里,X 早已把好几个 RPC 请求帧一股脑地塞进了 WebSocket 的发送缓冲区——这些帧**已经在路上了**。如果网关只是简单地"调度一个断开"然后就去处理这些已到达的帧,那么这些帧会带着一个**刚刚被吊销的令牌**被正常执行。令牌已经吊销了,请求却还在生效——这就是漏洞。

OpenClaw 的解法分两步,顺序至关重要:

1. **先标记失效(`invalidateClientsForDevice`),且在 `respond` 之前**。失效是同步生效的——它在每个请求的分发检查(per-request dispatch check)处立起一道闸:任何属于这个被失效设备的请求,在被真正处理前都会被拒绝。于是那些已经躺在 socket 缓冲区里的"管道化"帧,逐个走到分发检查时,全部被挡下。
2. **再调度断开(`queueMicrotask(disconnectClientsForDevice)`)**。物理断开可以慢慢来,因为逻辑上的"这个连接已经不可信"在第 1 步就已经确立了。

一句话:**先吊销资格,再清场;而不是先清场,任由清场前的余孽得逞**。把"标记失效"放在"回包"之前,是这段代码反复强调(`*before* responding`)的核心——它关闭了"已调度断开"与"在途帧"之间的竞态。这是一种把异步系统里的安全边界做对的典范:不要依赖"我马上就要断开它了"这种未来时,要让"它现在已经无效了"成为当下立即生效的事实。

## 12.8 不外泄与跨运行时一致:两个收尾的细节

**列表里永不外泄令牌material。** `device.pair.list` 返回配对列表时,每个已配对设备都过一遍 `redactPairedDevice`[[server-methods/devices.ts]](https://github.com/openclaw/openclaw/blob/main/src/gateway/server-methods/devices.ts):

```ts
function redactPairedDevice(device) {
  // Pairing lists are visible to operators; expose token lifecycle metadata
  // without returning raw token material or the internal approved-scope set.
  const { tokens, approvedScopes: _approvedScopes, ...rest } = device;
  return { ...rest, tokens: summarizeDeviceTokens(tokens) };
}
```

它剥掉两样东西:原始令牌 `tokens` 和内部的 `approvedScopes` 基线。取而代之的是 `summarizeDeviceTokens`——只暴露**生命周期元数据**:角色、scope、创建/轮换/吊销/最后使用的时间戳,但绝不暴露令牌本身的秘密值[[device-pairing.ts]](https://github.com/openclaw/openclaw/blob/main/src/infra/device-pairing.ts)。运维需要知道"这台设备有哪些令牌、各自什么状态"来做运维决策,但运维**不需要、也不应该**看到令牌的明文。能看见"何时轮换过",看不见"轮换成了什么"——可观测但不可窃取。

此外,非 admin 调用 `device.pair.list` 时,列表会被进一步过滤到只剩调用方自己的设备(`device.deviceId.trim() === authz.callerDeviceId`)。所有权语义连"看"这个动作都管住了。

**跨运行时的元数据归一化。** 配对要在不同平台(TypeScript 网关、Swift iOS、Kotlin Android)之间比对设备元数据,而字符串归一化在不同语言里极易产生分歧。OpenClaw 用了两个不同的归一化函数,各司其职[[device-metadata-normalization.ts]](https://github.com/openclaw/openclaw/blob/main/src/gateway/device-metadata-normalization.ts):

```ts
export function normalizeDeviceMetadataForAuth(value?: string | null): string {
  const trimmed = normalizeTrimmedMetadata(value);
  if (!trimmed) return "";
  // Keep cross-runtime normalization deterministic (TS/Swift/Kotlin) by only
  // lowercasing ASCII metadata fields used in auth payloads.
  return toLowerAscii(trimmed);
}

export function normalizeDeviceMetadataForPolicy(value?: string | null): string {
  const trimmed = normalizeTrimmedMetadata(value);
  if (!trimmed) return "";
  return normalizeLowercaseStringOrEmpty(trimmed.normalize("NFKD").replace(/\p{M}/gu, ""));
}
```

两者的差异是刻意的:

- **用于认证**(`...ForAuth`):只做 ASCII 小写化(`toLowerAscii`)。注释说明了原因——**为了让 TS/Swift/Kotlin 三个运行时的结果完全确定一致**。Unicode 的大小写折叠、NFKD 分解在不同语言的标准库里行为可能有细微差异;认证比对必须三端逐字节一致,否则同一台设备在不同平台算出不同的归一化值,认证就会莫名失败。所以认证路径上**只动 ASCII**,把不确定性彻底排除。
- **用于策略**(`...ForPolicy`):做 NFKD 分解 + 去除组合记号(`\p{M}`)+ 小写化。策略分类(判断平台/设备家族)需要把 Unicode 的"形近字(confusables)"折叠成稳定的 ASCII-ish token 再匹配,以免有人用形近字符绕过平台规则。

这是一个常被忽视的工程智慧:**同一个"归一化"需求,在不同语境下答案不同**。认证要的是"确定性"(宁可不折叠,也要三端一致),策略要的是"鲁棒性"(尽量折叠,挡住形近字攻击)。把它们拆成两个函数,而不是用一个"通用归一化"凑合,正是对"上下文决定正确性"的尊重。

## 本章小结

这一章我们走完了一个陌生实体接入网关总线的全过程,核心是一套层层设防的信任建立机制:

- **令牌即身份**:配对令牌是 32 字节随机 bearer 秘密,验证时拒绝空令牌、用常量时间比较防时序攻击。后续所有轮换/吊销,本质都是管理这个秘密的生命周期。
- **节点自动批准是 fail-closed 闸门**:`shouldAutoApproveNodePairingFromTrustedCidrs` 用八道 `return false` 把关,只放行"来自可信 CIDR 的、首次的、零 scope 的、非浏览器路径的 node 请求"。**没配置白名单就等于关闭**;任何升级路径(角色/scope/元数据)立刻打回人工。
- **审批权限按能力动态升级**:`resolveNodePairApprovalScopes` 让纯配对只需 `operator.pairing`、带命令需 `operator.write`、带系统命令需 `operator.admin`——批准一个实体所需的权限,等于它将获得的能力。
- **设备自服务的三块基石**:只有 device-token 认证才证明所有权;跨设备管理默认禁止(你管你自己,admin 管所有人);非 admin 只能碰 operator 角色。同一套所有权语义贯穿 remove/rotate/revoke/approve。
- **轮换只能收紧、不能透露给别人**:新 scope 必须落在已批准基线内(`scope-outside-approved-baseline`),没有基线直接拒;只有"自己轮换自己"才在响应里拿回新令牌,admin 替别人轮换只是"通知+作废"。
- **关门时先吊销资格再清场**:`invalidateClientsForDevice` 在 `respond` 之前同步立闸,让已管道化进 socket 缓冲区的在途帧在分发检查处被挡下,再 `queueMicrotask` 物理断开,关闭异步竞态。
- **可观测但不可窃取 + 上下文决定归一化**:列表只暴露令牌生命周期元数据、剥掉明文与基线;认证用确定性的 ASCII 归一化保证三端一致,策略用 NFKD+去记号的归一化挡住形近字攻击。

如果说前几章的安全是"管住已经在场的人能调用什么",这一章的安全是"管住谁能进场、进场后能拿到什么钥匙、钥匙何时换何时废"。两者合起来,网关才有一个完整闭环的信任边界:**入口默认关闭,权限只能收紧,秘密只回到主人手里,关门动作没有缝隙**。

## 动手实验

> 以下实验基于本章引用的源码文件,建议在你 clone 的 OpenClaw 仓库里对照阅读与验证。

**实验一:数清自动批准闸门挡住的每一种攻击。**
打开 `src/gateway/node-pairing-auto-approve.ts`,把 `shouldAutoApproveNodePairingFromTrustedCidrs` 的每一个 `return false` 单独抠出来,为每一条写一句"如果删掉这道门,攻击者能做什么"。重点想清楚第 ⑧ 道门(`autoApproveCidrs` 为空即拒绝)——为什么"未配置等于关闭"比"未配置等于放行"安全得多?再想第 ⑥ 道门:为什么 `loopback-trusted-proxy` 必须被排除,而 `trusted-proxy` 可以放行?

**实验二:推演审批权限的三个台阶。**
读 `src/infra/node-pairing-authz.ts` 的 `resolveNodePairApprovalScopes`。构造三个节点配对请求:(a) 不带任何 command;(b) 带一个普通 command;(c) 带一个命中 `NODE_SYSTEM_RUN_COMMANDS` 的命令。对每种情况,写出审批它需要的 scope 集合,并解释为什么一个"想跑系统命令的节点"必须由 admin 批准。这与第 10 章的最小权限模型如何呼应?

**实验三:验证"轮换不能扩权"不变量。**
读 `src/infra/device-pairing.ts` 的 `rotateDeviceToken`。假设一台设备入网时的已批准基线是 `["operator.read"]`,现在它发起一次轮换、请求 scope `["operator.read", "operator.write"]`。沿着代码走一遍,指出它会在哪一行、以什么 `reason` 被拒绝。然后改成请求 `["operator.read"]`,确认它能通过。最后回答:为什么"缩小到 `[]`"是允许的,而"扩大"是禁止的?

**实验四:重演关门竞态。**
读 `devices.ts` 里 `device.token.revoke` 处理器结尾的 `invalidateClientsForDevice` → `respond` → `queueMicrotask(disconnectClientsForDevice)` 三步。画一条时间线,标出"客户端把 3 个 RPC 帧塞进 socket 缓冲区"发生在哪个时刻。然后做一个反事实推演:如果把 `invalidateClientsForDevice` 挪到 `respond` 之后、甚至省掉只留 `queueMicrotask` 断开,那 3 个在途帧会发生什么?用这个推演解释注释里 `*before* responding` 这个强调为什么不能动。

## 下一章预告

我们已经把"谁能进总线、进来后能调用什么、配置如何热更、设备/节点如何配对"这条主干走完了。但有一类参与者一直站在聚光灯外却无处不在——**插件(plugin)与外部能力的运行时**。第 13 章我们将深入 OpenClaw 的**插件运行时与 MCP(Model Context Protocol)集成**:一个第三方插件如何被安装、被隔离、被赋予恰到好处的能力面?当插件想往核心里塞工具、塞节点命令时,网关如何坚持"插件能扩展、不能削弱核心"这条底线?配置热更(第 11 章)与配对授权(本章)又如何在插件的安装/卸载/重载里交汇?我们将看到,前面所有章节铺设的信任边界与可观测生命周期,如何在"对接外部世界"这个最开放的场景里继续守得住。
