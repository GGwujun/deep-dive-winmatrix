# 把 Claude Code / Hermes 变成虚拟数字员工：外部 Agent 怎么接入

> 这是《WinMatrix 开发经验文集》第 14 篇。上一篇讲了 MCP 让工具互通，这一篇往上走一层：当整个"Agent"是外部的时候——比如一台跑着 Claude Code 或 Hermes 的远程机器——怎么把它接入到 WinMatrix 的统一调度里，让它和内部的数字员工平起平坐。

先说一个听起来简单但做起来很绕的问题：**你的平台已经有了一批数字员工（大福、阿码、小质……），现在用户说"我还有个自己装的 Claude Code，跑在我自己的开发机上，能不能也让它进 WinMatrix 的群里一起干活？"**

这个诉求很合理。用户可能已经有了一套自己熟悉的 Agent 工具链（Claude Code、Hermes、OpenClaw），不想为了用 WinMatrix 就把它们扔掉。但如果每个外部 Agent 都单独接入一次、走完全不同的调用路径，平台的调度层很快就会被各种"特例"搞乱。

WinMatrix 的解法是一个很优雅的抽象：**把外部 Agent 包装成"虚拟数字员工"**。从调度层看，外部 Agent 和内部员工长得一模一样——有 id、有 roleId、有名字、能派活、能汇报。差别只在底层：外部 Agent 的执行不在本地，而在一台远程计算机上。

这一篇就讲这个"包装"是怎么做的，以及它背后那个更难的工程问题——**当 Agent 分散在不同物理机器上、甚至不同实例上时，怎么路由调用。**

---

## 第一步：把外部 Agent 变成"看起来一样"的虚拟员工

核心转换发生在 `externalAgentBootstrap.ts` 里。这个文件不大（整个 `external-agent/` 目录就这一个文件），但它做了一件关键的事：

```typescript
// agents/core/agent/external-agent/externalAgentBootstrap.ts（第 73-101 行）
const EXTERNAL_AGENT_ROLE_ID = 'external-agent';

export async function getExternalAgentsForUser(userId: string): Promise<DispatchableDigitalEmployee[]> {
  const registrations = await prisma.external_agent_registration.findMany({
    where: { userId, status: 'active', isConnected: true },
    orderBy: { createdAt: 'desc' },
  });
  for (const reg of registrations) {
    const reachable = connectionPool.isAgentReachable(reg.id) || reg.isConnected;
    result.push({
      id: `ext_${reg.id}`,                          // 关键：ext_ 前缀防冲突
      roleId: EXTERNAL_AGENT_ROLE_ID,                // 统一 roleId
      name: reg.name?.trim() || reg.agentType,
      isExternal: true,                              // 标记外部
      externalAgentId: reg.id,
      externalAgentStatus: reachable ? 'connected' : 'disconnected',
    });
  }
}
```

这段代码把一个 `external_agent_registration`（外部 Agent 注册记录）转换成了一个 `DispatchableDigitalEmployee`（可派活的数字员工）。三个设计点：

### ext_ 前缀：ID 防冲突

`id: \`ext_${reg.id}\``——所有外部 Agent 的员工 id 都加 `ext_` 前缀。为什么？因为内部员工的 id 是 UUID（比如 `a1b2c3...`），外部 Agent 的注册 id 也是 UUID。两者格式一样，**如果不加前缀，id 空间就重叠了**——理论上可能出现一个内部员工和一个外部 Agent 恰好 id 相同的情况（虽然概率极低）。加了 `ext_` 前缀，外部 Agent 的 id 永远以 `ext_` 开头，和内部员工的 UUID 物理隔离，零冲突可能。

这是个很小但很重要的技巧。**当两套不同来源的 ID 要进同一个命名空间时，加前缀是最省事的隔离办法。** 它比"相信 UUID 碰撞概率"靠谱（碰撞概率是低，但不是零，生产系统不能赌这个），也比"加一张映射表"简单（映射表意味着多一次 join，复杂度上升）。

### 统一 roleId：一个角色管所有外部 Agent

`roleId: EXTERNAL_AGENT_ROLE_ID`（常量 `'external-agent'`）。注意这是 WinMatrix 的**第八个角色**——前七个（大福、阿宁、阿码等）是固定的业务角色，第八个 `external-agent` 是运行时动态生成的，专门给外部 Agent 用。

为什么所有外部 Agent 共用一个 roleId？因为外部 Agent 的"能力"不是平台定义的——它跑什么完全取决于那台远程机器上装了什么。平台没法预先给"Claude Code"和"Hermes"分别定义角色和技能。**用一个统一的 `external-agent` 角色，把它们都归到"外部能力"这个桶里，具体能力交给运行时探测。**

这也是 WinMatrix 八大角色的设计弹性所在——七个固定业务角色管内部，第八个动态角色管外部，既保持了"角色优先"的架构，又兼容了"外部能力无法预定义"的现实。

### isExternal + 可达性：状态透明

转换出来的员工对象有两个额外字段：`isExternal: true`（标记外部）和 `externalAgentStatus`（`connected` / `disconnected`）。这让调度层能区分"这是外部 Agent"并做相应处理——比如外部 Agent 断线时不派活给它、或者派活了要有超时兜底。

可达性判断 `connectionPool.isAgentReachable(reg.id) || reg.isConnected` 用了"或"——连接池说可达**或**注册记录说已连接，就算可达。这是**乐观判断**：只要有一方说"在线"，就先认为它在线，让它进调度池。真正的失败在派活时由超时和重试兜底。对外部不可控的资源，乐观比悲观更合适——悲观会漏掉实际可用的 Agent。

---

## 注册模型：一个外部 Agent 长什么样

转换的输入是 `external_agent_registration`，它的字段决定了平台对外部 Agent "知道什么"：

```prisma
// prisma/schema.prisma（external_agent_registration 模型）
model external_agent_registration {
  userId            String
  agentType         String    // Claude Code / Hermes / OpenClaw
  name              String
  capabilities      String[]
  apiKeyHash        String    // hash 存储
  apiKeyEncrypted   String    // 加密存储
  isConnected       Boolean
  lastHeartbeatAt   DateTime?
  status            String    // active | inactive
  computerId        String?   // 跑在哪台计算机上
  lastSessionId     String?
  capabilityProfile Json?
  tools             String[]
  endpoint          String?
  endpointToken     String?
  hermesEndpoint    String?
}
```

几个有意思的设计：

- **`agentType` 区分种类**。Claude Code、Hermes、OpenClaw 各是一种，它们用的协议、调用的方式可能不同。
- **三套 endpoint 字段**（`endpoint` / `hermesEndpoint` / `endpointToken`）。为什么不是统一的 `endpoint`？因为不同 Agent 类型的连接协议不一样——Hermes 有自己的 endpoint 约定，其他用通用 endpoint。**协议差异显式化，比强行统一字段再 if-else 判断清晰。**
- **心跳 + 连接状态双字段**。`isConnected` 是当前状态，`lastHeartbeatAt` 是最近一次心跳时间。两个配合判断真实可达性——光看 `isConnected` 可能误判（Agent 崩了但状态还没更新），加上 `lastHeartbeatAt` 能发现"虽然状态是 connected 但心跳停了 5 分钟"这种半死状态。
- **密钥双存储**（`apiKeyHash` + `apiKeyEncrypted`）。hash 用于校验"这个 key 对不对"，加密版用于实际调用时解密使用。一个查、一个用，各司其职。

### 物理计算机抽象：computer 模型

注册记录里有 `computerId`，指向 `external_agent_computer`——它表示"这台 Agent 跑在哪台物理机器上"：

```prisma
// external_agent_computer 模型
userId            String
installationId    String
hostname          String
os                String      // 操作系统
arch              String      // 架构
daemonVersion     String
detectedRuntimes  String[]    // 检测到的运行时
supportedAgentTypes String[]  // 支持哪些 Agent 类型
isConnected       Boolean
```

为什么要单独抽象"计算机"？因为**一台机器上可能跑多个 Agent**（Claude Code 和 Hermes 共存），也可能**同一个 Agent 类型在多台机器上都有**。计算机和 Agent 是多对多关系，必须分开建模。

`detectedRuntimes` 和 `supportedAgentTypes` 让平台知道"这台机器支持什么"——派活时可以据此选择合适的机器。比如一个需要 Node 20 的任务，只能派给装了 Node 20 的机器。

---

## 真正的难题：分布式 Owner 路由

前面的"包装成虚拟员工"还算直观。真正难的是：**当外部 Agent 不在当前实例上时，怎么找到它？**

考虑这个场景：WinMatrix 部署了 3 个实例（多副本），用户 A 的 Claude Code 注册到了实例 1（它的 websocket 连到了实例 1）。现在用户 A 通过实例 3 发消息"让我的 Claude Code 干个活"——实例 3 上没有这个 Agent 的连接，它怎么把活派过去？

这就是**分布式 Owner 路由**问题。WinMatrix 的解法在 `ExternalAgentGateway.ts` 里：

```typescript
// integration/connectors/external-agent/distributed/ExternalAgentGateway.ts（第 32-63 行）
async listSessions(agentId: string, opts = {}) {
  // 1. 查这个 Agent 的 Owner 在哪个实例
  const owner = await this.ownerRegistry.getAgentOwner(agentId);
  if (!owner) {
    // 2a. 没有 Owner 记录，但本地恰好有连接 → 本地直连
    if (this.connectionPool.getConnection(agentId)?.isConnected)
      return this.sessionQuery.listSessionsLocal(agentId, opts);
    // 2b. 既没 Owner，本地也没连接 → 真不可达
    throw unavailable(`owner_not_found: agentId=${agentId}`);
  }
  // 3. Owner 是自己 → 本地直连
  if (owner.instanceId === this.instanceId)
    return this.sessionQuery.listSessionsLocal(agentId, opts);
  // 4. Owner 是别的实例 → 跨实例 RPC
  const response = await this.rpcBus.call(owner.instanceId, {
    op: 'agent.session.list', agentId, payload: opts,
  });
  return responseToResult<{ sessions; total? }>(response);
}
```

这段逻辑是分布式路由的教科书式实现。四条路径，按优先级：

```
收到对 agentId 的调用：
  │
  ├─ 查 ownerRegistry → 有 Owner？
  │    │
  │    ├─ 有 Owner，且是自己 → 本地直连（最快）
  │    │
  │    └─ 有 Owner，是别的实例 → 跨实例 RPC 转发
  │
  └─ 没 Owner
       │
       ├─ 本地恰好有连接 → 本地直连（兜底）
       │
       └─ 本地也没连接 → 抛 503 unavailable（诚实失败）
```

几个精妙之处：

### Owner Registry：谁拥有这个 Agent

`ownerRegistry.getAgentOwner(agentId)` 返回的是"这个 Agent 的 websocket 当前连在哪个实例上"。Owner registry 是所有实例共享的（通常存在 Redis 里），任何实例都能查。Agent 连接时在 registry 里登记"我在实例 X"，断开时清除。

这是**服务发现**的标准模式——用一个共享注册表记录"谁在哪"。比"广播问所有实例'你有这个 Agent 吗'"高效得多。

### 本地兜底：Owner 没记录但本地有连接

注意第 2a 条路径：Owner registry 没记录，但本地恰好有这个 Agent 的连接——这种情况会发生在"Agent 刚连上来、registry 还没同步"的窗口期。这时不急着抛错，先看看本地有没有，有就先用。**这是对"最终一致性的窗口期"的容错。**

### 跨实例 RPC：对调用方透明

第 4 条路径：Owner 是别的实例，通过 RPC bus 转发。关键在于——**调用方完全感知不到这是跨实例的**。它调的就是 `gateway.listSessions(agentId)`，gateway 内部决定本地执行还是 RPC 转发，调用方拿到的结果格式一模一样。

这是分布式系统里"位置透明性"（location transparency）的体现。调用方只认 agentId，不关心 Agent 在哪台机器、哪个实例上。**让复杂性沉到 gateway 层，让上层保持简单。**

### 诚实失败：不可达就抛 503

第 2b 条路径：既没 Owner，本地也没连接——这是真不可达。**这时 gateway 选择诚实地抛 503 `unavailable`，而不是假装成功返回空结果。**

为什么强调"诚实"？因为有些系统遇到这种情况会返回空数组（"没有 session"），这在语义上是**撒谎**——不是没有 session，是你联系不上 Agent。撒谎会让上层做出错误判断（以为 Agent 真的没 session，于是重新初始化）。抛 503 是告诉上层"我联系不上，你自己决定怎么办"——上层可以重试、可以降级、可以提示用户。**把决策权还给上层，而不是用假数据掩盖问题。**

---

## 项目级熔断：外部 Agent 也会被"拉闸"

外部 Agent 接入后，还有一个安全机制：**项目级熔断**。

```prisma
// external_agent_pause 模型（"项目级外部 Agent 熔断开关"）
projectId         String
conversationId    String?
externalAgentId   String
paused            Boolean
reason            String?
pausedBy          String?
```

当一个外部 Agent 在某个项目里表现异常（比如疯狂消耗 token、或者产出可疑内容），管理员可以把它在这个项目里"暂停"（`paused=true`），不影响它在其他项目里的使用。这是**熔断的粒度控制**——不是一刀切禁用整个 Agent，而是精准到"某项目 × 某 Agent"。

**熔断要支持细粒度。** 一个 Agent 可能在 10 个项目里正常、在 1 个项目里出问题。一刀切禁用会误伤那 10 个正常的；按 (project, agent) 粒度熔断，只影响出问题的那个组合。

---

## 生命周期观测：activity_event + reminder

外部 Agent 一旦接入，它的所有行为都要能观测。两个模型支撑这件事：

- **`external_agent_activity_event`**：活动事件时间线。Agent 的每次重要行为（连上、断开、执行任务、产出结果）都记一条，形成完整的时间线。事后排查"这个 Agent 昨天干了什么"，拉这条时间线就行。
- **`external_agent_reminder`**：Agent 上报的未来提醒。Agent 自己可以设置"明天 10 点提醒我"，平台负责到点投递。

这两个模型让外部 Agent 不只是"能派活"，还是"可观测、可交互"的——它的历史可追溯、它的未来计划有记录。**把外部 Agent 当一等公民对待，而不是"调一下就扔"的外设。**

---

## 业界对比：外接 Agent 的两种范式

横向看一下，外部 Agent 接入大致两种范式：

**一种是"工具代理"**——平台把外部 Agent 当成一个工具，需要时调用一下。外部 Agent 是被动的，平台主动调。这种范式简单，但外部 Agent 只是个"函数调用"，不参与平台的统一调度、不能被其他角色协作、没有持续的身份。

**另一种是"身份注入"**——也就是 WinMatrix 这种，把外部 Agent 包装成虚拟数字员工，给它一个 id 和角色，让它进入平台的统一调度。外部 Agent 是主动的参与者，可以被派活、可以协作、有持续的身份。

工具代理适合"偶尔用一下外部能力"的场景；身份注入适合"让外部 Agent 长期融入团队"的场景。后者复杂度高得多（要处理分布式路由、状态同步、熔断、观测），但换来的是**外部和内部的一致性**——从调度层看，没有内外之分，都是员工。

这种一致性带来的最大好处是**协作**。"让阿码（内部）评审 Claude Code（外部）写的代码"——这样一句话能成立，正是因为两者都是"员工"，都能被编排进同一个协作流程。如果 Claude Code 只是个工具，这句话就说不出来。

---

## 给后来者的总结

1. **把外部 Agent 包装成虚拟数字员工**。加 `ext_` 前缀防 ID 冲突，统一 roleId 进调度层。从调度层看，内外一致，没有特例。
2. **两套来源的 ID 进同一命名空间，加前缀最省事**。比"相信 UUID 不碰"靠谱，比"加映射表"简单。
3. **第八个角色 `external-agent` 是动态角色**。外部能力无法预定义，用一个统一角色归桶，具体能力运行时探测。架构弹性体现于此。
4. **分布式 Owner 路由四条路径**：有 Owner 且是自己→本地直连；有 Owner 是别人→跨实例 RPC；没 Owner 但本地有连接→兜底直连；都没有→诚实抛 503。
5. **位置透明性**。调用方只认 agentId，gateway 内部决定本地还是 RPC。让复杂性沉到网关层，上层保持简单。
6. **诚实失败比假数据好**。不可达就抛 503，别返回空结果假装成功。把决策权还给上层。
7. **熔断要细粒度**。按 (project, agent) 粒度，不是一刀切禁用整个 Agent。
8. **外部 Agent 是一等公民**。activity_event 时间线 + reminder，让它可观测、可交互，而不是"调一下就扔"的外设。
9. **身份注入 vs 工具代理是范式选择**。长期融入团队用身份注入（复杂但内外一致），偶尔用一下用工具代理（简单但被动）。

外部 Agent 接入的核心洞察是：**内外一致性比接入便利性更值钱。** 多花点功夫做包装和路由，换来的是"外部 Agent 能参与协作"这个质变。一个能和内部员工并肩干活的外部 Agent，价值远高于一个只能被调用的外部工具。

---

> **下一篇**：[《配置热更新：为什么我们用 PG LISTEN/NOTIFY 而不是 Redis Pub/Sub》](./15-config-hot-reload.md)——Agent 和外部资源都接进来了，但有个基础设施层面的问题还没讲：配置改了怎么不重启就生效？为什么选 PG 的 LISTEN/NOTIFY 而不是看起来更主流的 Redis Pub/Sub。
