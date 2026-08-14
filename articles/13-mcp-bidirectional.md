# MCP 双向架构：一个平台怎么既当 Server 又当 Consumer

> 这是《WinMatrix 开发经验文集》第 13 篇。MCP（Model Context Protocol）是 Anthropic 推的"LLM 调工具的标准协议"，业内讨论很多。但大多数实现都只聊"怎么消费 MCP"——也就是让你的 Agent 去调外部的 MCP server。这一篇讲一个少有人聊的角度：**一个平台同时做 Server 和 Consumer，怎么设计。**

先说个常见的误解：很多人以为 MCP 是"工具市场"——你装一堆 MCP server，你的 Agent 就有了各种能力。这只是 MCP 的半边。

MCP 的另一半是**暴露**：如果你的平台本身有一堆强大的内部工具（项目管理、知识库检索、流程编排……），你不应该让外部 Agent 每次都重新集成一遍，而是应该把这些工具**也变成 MCP server**，让任何支持 MCP 的客户端都能用。

这就是 WinMatrix 的做法——它**同时是 MCP Consumer（消费外部 MCP 工具）和 MCP Server（把内部能力暴露出去）**。这种双向架构带来一个很有意思的工程问题：两边的关注点完全不同，却要共存于一个平台里。这一篇拆给你看。

---

## Consumer 侧：怎么把外部 MCP 工具接进来

先看消费侧。外部 MCP 服务在数据库里是这样登记的：

```prisma
// prisma/schema.prisma（mcp_services 模型）
model mcp_services {
  project_id      String?
  name            String
  description     String?
  transport_type  String   @map("transport_type")  // sse | http
  url             String?
  api_key         String?
  headers         Json?
  is_enabled      Boolean  @default(true)
  is_builtin      Boolean  @default(false)
  tool_whitelist  String[] @map("tool_whitelist")
  assigned_agents String[] @map("assigned_agents")
  // ...
}
```

几个字段值得注意，它们决定了"外部工具怎么被消费"：

- **`transport_type`**。MCP 支持两种传输：sse（服务端推送）、http。transport_type 决定了怎么连这个 MCP server。
- **`tool_whitelist`（工具白名单）**。一个 MCP server 可能暴露几十个工具，但你未必都想暴露给 Agent。白名单过滤——只把白名单里的工具纳入 Agent 的工具集。
- **`assigned_agents`（分配给哪些 Agent）**。多租户的关键：同一个 MCP server，可以分配给项目 A 的 Agent，但不给项目 B。这是"工具的可见性是按租户控制的"。
- **`project_id`**。MCP 服务可以挂在项目级别（项目私有），也可以全局可用（project_id 为空）。这是另一个维度的可见性控制。

这几个字段加起来，构成了一个**多租户的 MCP 消费模型**：同一个外部 MCP server，不同的项目、不同的 Agent，看到的是不同的工具子集。

### 连接管理：Promise.allSettled 并行连接，坏的不拖好的

MCP 管理器初始化时，要把所有 `is_enabled=true` 的 MCP 服务都连上。怎么连？一个一个串行连？不行——十个 MCP 服务，每个握手要 1 秒，串行就是 10 秒启动延迟。

WinMatrix 的做法是**并行连接 + allSettled 容错**：

```typescript
// infrastructure/mcp/McpManager.ts（第 50-96 行）
async initialize(): Promise<void> {
  if (this.initialized) return;
  const services = await prisma.mcp_services.findMany({
    where: { is_enabled: true },
  });
  await Promise.allSettled(services.map(svc => this.connectService(svc)));
  this.initialized = true;
  logger.info(`[${LOG_TAG}] 已初始化 ${this.clients.size}/${services.length} 个 MCP 服务`);
}
```

关键是 `Promise.allSettled` 而不是 `Promise.all`。区别在于：

- `Promise.all`：任何一个 reject，整个 Promise 立即 reject。一个 MCP 服务挂了，其他正常的也白连了。
- `Promise.allSettled`：等所有 Promise 都 settle（不管 resolve 还是 reject），坏的不影响好的。

最后那行日志 `已初始化 ${clients.size}/${services.length}` 也很关键——它告诉你"10 个里连上了 7 个"，而不是非黑即白地报"成功/失败"。**部分失败是分布式系统的常态，你的日志要能反映"部分"，而不是强行二值化。**

---

## Server 侧：把内部能力暴露成 MCP

消费侧讲完了，现在看暴露侧——WinMatrix 自己作为 MCP Server 的部分。

为什么要做 Server？因为 WinMatrix 内部有 27 个工具域模块（项目管理、任务、文档、知识库、RAG 检索……），这些都是经过治理的能力（有权限、有契约、有审计）。如果外部 Agent（比如一个第三方的 Claude Code 客户端）想用这些能力，你不应该让它重新实现一遍集成，而是应该**用 MCP 协议把这些能力标准化地暴露出去**。

Server 侧的核心问题不是"怎么实现协议"（MCP 协议本身不复杂），而是**鉴权和多 token 体系**——谁来调？调什么？

### 多 token 体系：三类身份，Token Broker 统一路由

Server 侧的鉴权比消费侧复杂得多。消费侧你只是个客户端，拿个 api_key 就行。Server 侧你是服务端，要区分"谁在调我"——不同身份有不同的权限边界。

WinMatrix 设计了三类 token，用一个 Token Broker 统一路由：

```typescript
// src/interface/mcp/tokenBroker.ts（第 353-367 行）
export async function resolve(token, extra?): Promise<TokenBrokerResolveResult> {
  if (token.startsWith(PAT_PREFIX)) return tryResolvePat(token, extra);   // wm_pat_
  if (token.startsWith(WMA_PREFIX)) return tryResolveWma(token);          // wma_
  if (token.startsWith(WMEC_PREFIX)) return tryResolveWmec(token);        // wmec_
  return { status: 'unknown' };
}
```

三类 token 各自的身份和边界：

| token 前缀 | 名字 | 身份 | 权限边界 |
|-----------|------|------|---------|
| `wm_pat_` | PAT（Personal Access Token） | 人 + 项目 | 强制绑定默认项目 + membership 校验 |
| `wma_` | WMA | 外部 Agent 注册身份 | 按 registration.tools 限定 |
| `wmec_` | WMEC | 外部接入方应用身份 | 应用级授权 |

为什么用前缀路由？因为三类 token 的语义完全不同——PAT 是"某个人在某项目里的身份"，WMA 是"某个外部 Agent 的身份"，WMEC 是"某个第三方应用的身份"。**用前缀显式区分，比把它们塞进一个字段再"猜"要清晰得多。** 前缀是类型，一眼能看出身份。

几个细节：

- **PAT 强制绑定默认项目**。一个 PAT 必须绑定一个默认项目，而且鉴权时要校验"这个 PAT 的属主是不是这个项目的成员"。防止"拿着 A 项目的 PAT 调 B 项目的接口"。
- **WMA 按 `registration.tools` 限定**。外部 Agent 注册时声明要用哪些工具，WMA token 只能调声明的工具——最小权限。
- **鉴权后写 session**。鉴权通过后，会把 session 信息写到 Redis（`toolProxySessionStore`），后续请求不用每次都重新鉴权，查 session 即可。

### PAT 的存储：hash 不存明文

和上一篇讲的 callbackToken 一样，PAT 也只存 SHA-256 hash：

```prisma
// prisma/schema.prisma（personal_access_tokens，第 2497-2513 行）
model personal_access_tokens {
  // "个人访问令牌（MCP Bridge 鉴权）"
  tokenHash        String   @unique   // 唯一，SHA-256，不存明文
  defaultProjectId String
  expiresAt        DateTime?
  lastUsedAt       DateTime?
  revokedAt        DateTime?
}
```

`tokenHash` 唯一 + SHA-256，意味着：

- 生成时返回明文给用户一次（`wm_pat_xxxxxxxx`），之后再也查不到明文。
- 校验时算 hash 比对，对得上就通过。
- 数据库被拖库，攻击者拿到的只是一堆 hash，没法直接当 token 用。

**token 不存明文是底线安全实践。** 无论是回调 token 还是访问令牌，一律 hash 存储。这条原则没有例外。

---

## 双向架构的真正难点：两边的关注点完全不同

讲完两边，现在说重点——**为什么"同时做 Server 和 Consumer"是个值得单独拎出来讲的架构问题**。

因为这两边的关注点是完全相反的：

| 维度 | Consumer 侧（消费外部 MCP） | Server 侧（暴露内部能力） |
|------|---------------------------|-------------------------|
| 主要风险 | 外部服务不可靠（挂、慢、返回垃圾） | 调用方不可信（伪造、越权） |
| 关键能力 | 容错（allSettled）、超时、降级 | 鉴权、限流、审计 |
| 状态管理 | 多个外部连接的生命周期 | 多类 token 的 session |
| 失败处理 | 坏的不拖好的（部分成功） | fail-closed（宁可拒绝不误放） |

一个平台同时做两边，意味着你要**在同一套代码里同时处理"对外部不可靠"和"对内不可信"**。这两件事的思维模式是冲突的：

- Consumer 侧你要"乐观"——尽量连、连不上就算了、坏的不拖好的。
- Server 侧你要"悲观"——默认不信任、严格校验、宁可错杀。

很多系统只做一边（要么纯消费、要么纯暴露），所以只需要一种思维。但双向平台必须**同时维持两种思维**，并且不能让它们串台——Consumer 侧的"乐观"不能渗透到 Server 侧的鉴权里（那叫"鉴权容错过度"，安全隐患），Server 侧的"悲观"不能渗透到 Consumer 侧的连接里（那叫"一个外部服务挂了全部连不上"，可用性灾难）。

WinMatrix 的做法是**两边物理隔离**：MCP Consumer 的代码在 `infrastructure/mcp/`，MCP Server 的代码在 `interface/mcp/`，两套互不干扰。共用的只有"工具的元数据定义"（因为同一份工具可能既内部用也外部暴露）。**让两种相反的关注点住在不同的目录里，是防止思维串台的最直接办法。**

---

## 多租户：同一个 MCP server，不同的可见性

最后讲一个双向架构都绕不开的问题：多租户。

无论是消费侧（外部 MCP 工具给谁用）还是暴露侧（内部能力给谁调），都有一个"可见性控制"的问题。WinMatrix 用了三层控制：

1. **项目级**。`mcp_services.project_id` 控制外部 MCP 是项目私有还是全局；`personal_access_tokens.defaultProjectId` 控制 PAT 绑定哪个项目。
2. **Agent 级**。`assigned_agents` 数组控制外部 MCP 分配给哪些 Agent；WMA 的 `registration.tools` 控制 Agent 能调哪些工具。
3. **工具级**。`tool_whitelist` 控制外部 MCP 里哪些工具暴露出来；Server 侧的工具权限走 RBAC。

三层叠加，能表达很精细的策略："项目 A 的 Agent X 可以用 MCP 服务 M 里的工具 t1、t2，但不能用 t3。" 这种精细控制在企业场景里是刚需——不同部门、不同角色，能看到的能力本来就该不一样。

**多租户不要指望一个字段搞定。** 它天然是多个维度的叠加——租户、角色、资源、操作。每个维度单独看都简单，叠加起来才复杂。好的设计是把每个维度都做成独立的控制点，而不是硬塞进一个"权限"字段。

---

## 给后来者的总结

1. **MCP 不只是消费，还有暴露**。双向平台同时是 Consumer 和 Server。两边的关注点相反（Consumer 乐观容错、Server 悲观鉴权），必须物理隔离防止思维串台。
2. **Consumer 侧用 allSettled 并行连接**。部分失败是常态，用 allSettled 让坏的不拖好的；日志要反映"部分成功"而不是二值化。
3. **Server 侧用多 token 体系区分身份**。PAT（人+项目）/ WMA（外部 Agent）/ WMEC（应用）三类，Token Broker 按前缀统一路由。前缀即类型，显式比猜好。
4. **token 一律 hash 存储**。无论回调 token 还是访问令牌，SHA-256 hash + 唯一约束，明文只在生成时返回一次。
5. **多租户是多维叠加，不是单字段**。项目级 + Agent 级 + 工具级三层控制，每层独立设计，叠加表达精细策略。
6. **PAT 强制绑定项目 + membership 校验**。防"A 项目的 token 调 B 项目的接口"，这是跨项目越权最常见的漏洞。
7. **WMA 按注册声明限定工具**。外部 Agent 注册时声明用哪些工具，token 只能调声明的——最小权限原则。
8. **双向架构的价值是"能力复用"**。内部能力 MCP 化后，任何支持 MCP 的客户端都能用，不必每个客户端重新集成。

MCP 双向架构的核心洞察是：**消费和暴露是同一枚硬币的两面，但它们的安全姿态必须相反。** 让它们住在不同的代码区域、用不同的思维对待，是双向平台不被"一边的隐患传染到另一边"的关键。

---

> **下一篇**：[《把 Claude Code / Hermes 变成虚拟数字员工：外部 Agent 怎么接入》](./14-external-agent-bootstrap.md)——MCP 讲的是工具层面的互通，这一篇讲的是"整个 Agent"层面的接入：怎么把一个跑在外部计算机上的 Claude Code 或 Hermes，变成 WinMatrix 里和内部员工并列的"虚拟数字员工"。
