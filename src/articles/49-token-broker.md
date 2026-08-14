# Token 凭证体系：PAT / WMA / WMEC 三种令牌各保护什么

> 这是《WinMatrix 开发经验文集》第 49 篇。前面第 06、14、22、28 篇都从不同角度提到过 token——技能凭证、外部 Agent 接入、安全护栏。但有一个问题始终没单独展开：WinMatrix 里到底有几种 token？它们各自保护什么、谁签发、谁校验？这一篇把三类令牌（PAT / WMA / WMEC）摆在一起，讲清楚它们的分工，以及那个按前缀路由的 Token Broker。

先说为什么要三种 token，而不是一种。

很多系统的做法是"一个 JWT 走天下"——用户登录拿 JWT，所有请求都带它。WinMatrix 也有 JWT（第 28 篇讲过，HS256 + Redis 黑名单）。但 JWT 解决的是"已登录的真人用户访问系统"。一旦把视角放大到"外部程序也要访问 WinMatrix"，JWT 就不够用了：

- 一个真人用户想在自己的脚本里调 WinMatrix 的 MCP 工具，他不想每次都走登录流程拿 JWT——他要的是一把"长期有效的钥匙"（PAT）。
- 一个外部编码 Agent（Hermes/OpenClaw）要注册进系统变成虚拟数字员工，它的身份不是"某个用户"，而是"某个 Agent"——它需要一种专门的注册凭证（WMA）。
- 一个外部接入方应用（比如某个集成平台）要以"应用身份"代表自己访问 WinMatrix，而不是代表某个具体用户——它需要第三种凭证（WMEC）。

这三种场景的**身份语义完全不同**：PAT 是"我是某用户、在某项目里"，WMA 是"我是某 Agent、会这些工具"，WMEC 是"我是某应用"。如果把它们硬塞进 JWT，要么丢失语义（把 Agent 当用户），要么 JWT payload 膨胀成什么都装的杂货铺。WinMatrix 的选择是——**三种独立令牌，各有前缀，各有存储，Token Broker 按前缀统一路由。**

---

## 三种令牌的分工

先把三种令牌的字段和语义摆出来，再讲路由。

### PAT：Personal Access Token（人-项目）

PAT 是最接近传统 API key 的令牌——一个真人用户为自己的某个项目生成长期访问凭证。看它的存储（`personal_access_tokens`，schema L2497-2513）：

```
personal_access_tokens（"个人访问令牌（MCP Bridge 鉴权）"）
├── tokenHash              唯一，SHA-256（不存明文）
├── userId                 归属用户
├── defaultProjectId       ← 强制绑定的默认项目
├── expiresAt              过期时间
├── lastUsedAt             最后使用时间
└── revokedAt              撤销时间
```

几个关键设计（核实报告 ch04-06 主题 3.4）：

**前缀 `wm_pat_`。** 生成时格式是 `wm_pat_{random16hex}`（`PersonalAccessTokenStore.ts`）。前缀不是装饰——它是 Token Broker 路由的依据。

**强制绑定 `defaultProjectId`。** 这是 PAT 最重要的语义：**一把 PAT 只能代表"某用户在某项目"**。没有"全局 PAT"。为什么这么严格？因为 PAT 是长期凭证，一旦泄露，攻击面等同于"该用户在该项目的全部权限"。强制绑定项目，把爆炸半径限制在单个项目内——泄露了也只影响一个项目。

**只存 `tokenHash`（SHA-256）。** 数据库里没有原始 token，只有 hash。验证时也是 hash 比对。这是第 28 篇讲过的"hash 不存明文"原则——即使 DB 被拖库，攻击者也拿不到可用的 token。

校验时除了 hash 匹配，还要做 **membership 校验**：确认这个 PAT 的 userId 确实是 defaultProjectId 的成员。光有合法的 PAT 还不够，还要确认"这个用户还在这个项目里"——如果用户被移出项目，PAT 立刻失效。

### WMA：WinMatrix Agent（外部 Agent 注册）

WMA 是给外部编码 Agent（Hermes/OpenClaw 等）的注册凭证。它的身份语义不是"某用户"，而是"某 Agent"。看注册表 `external_agent_registration`：

```
external_agent_registration（"外部智能体注册（Hermes/OpenClaw 等接入）"）
├── userId                 归属用户（注册者）
├── agentType              Agent 类型
├── name                   Agent 名字
├── capabilities           能力声明
├── apiKeyHash             API key 的 hash（查找用）
├── apiKeyEncrypted        API key 密文（实际调用用）
├── isConnected            连接状态
├── status                 active | ...
├── computerId             所在物理机
├── tools                  ← 这个 Agent 会哪些工具
├── capabilityProfile      能力画像
├── endpoint               端点
├── endpointToken          端点凭证
└── hermesEndpoint         Hermes 专用端点
```

WMA 的前缀是 `wma_`。它的语义是"我是某个已注册的外部 Agent，我能用我注册时声明的那些 tools"。**WMA 的权限边界由注册时的 `tools` 字段限定**——Token Broker 解析 WMA 后，会按 `registration.tools` 决定这个 Agent 能调哪些工具。

注意这里同时有 `apiKeyHash` 和 `apiKeyEncrypted`——这是第 60 篇会详细讲的"需还原用加密、需验证用 hash"的双字段模式：hash 用于快速查找（O(1) 索引），encrypted 用于实际调用时还原出原始 key。WMA 自己则是一种独立的、按 Agent 身份签发的令牌。

### WMEC：WinMatrix External Computer（外部接入方应用身份）

WMEC 的前缀是 `wmec_`，语义是"外部接入方应用身份"。它代表的不是某个用户或某个 Agent，而是一个**外部应用/平台**——比如某个集成系统要以自己的身份（而非代表某个用户）访问 WinMatrix。

这种"应用身份"在传统系统里通常用 OAuth2 的 client_credentials 模式实现。WinMatrix 没有走完整 OAuth2，而是用 WMEC 这种轻量令牌——够用、可控。

### 三者横向对比

| 令牌 | 前缀 | 身份语义 | 权限边界 | 存储 |
|------|------|---------|---------|------|
| PAT | `wm_pat_` | 某用户 + 某项目 | 用户在该项目的 membership | tokenHash + defaultProjectId |
| WMA | `wma_` | 某外部 Agent | registration.tools 声明 | apiKeyHash + apiKeyEncrypted |
| WMEC | `wmec_` | 某外部应用 | 应用级授权 | （接入方凭证） |

这张表回答了开头的"为什么要三种"——**身份语义不同，权限边界不同，存储模型不同。** 硬塞进一种 token 会丢失语义。

---

## Token Broker：按前缀统一路由

三种令牌各有存储，但调用方不应该关心"这是哪种 token、要去哪张表查"。WinMatrix 用一个 Token Broker 做统一入口（`interface/mcp/tokenBroker.ts`，370 行）。

核心是 `resolve` 函数（`:353-367`）：

```ts
export async function resolve(token, extra?): Promise<TokenBrokerResolveResult> {
  if (token.startsWith(PAT_PREFIX)) return tryResolvePat(token, extra);   // wm_pat_
  if (token.startsWith(WMA_PREFIX)) return tryResolveWma(token);          // wma_
  if (token.startsWith(WMEC_PREFIX)) return tryResolveWmec(token);        // wmec_
  return { status: 'unknown' };
}
```

这就是"按前缀路由"。一个 token 进来，先看前缀，分流到对应的 resolver。每个 resolver 知道怎么查自己的表、怎么做校验。

这个设计有几个值得说的点。

**第一，前缀是强制的，不是可选的。** 每种令牌生成时就必须带自己的前缀。这不是命名规范，是路由契约——没有前缀的 token，Broker 直接返回 `unknown`。这逼着所有令牌类型在设计时就声明自己的身份，不能含糊。

**第二，统一入口，分散实现。** Broker 只做"分流"，真正的校验逻辑在各 resolver 里。`tryResolvePat` 知道 PAT 要查 personal_access_tokens 表 + 做 membership 校验；`tryResolveWma` 知道 WMA 要查 external_agent_registration + 读 tools 字段；`tryResolveWmec` 知道 WMEC 的应用身份逻辑。**新增一种令牌类型，只需要加一个前缀常量 + 一个 resolver 函数，Broker 的入口代码不动。** 这是开闭原则在凭证体系里的落地。

**第三，resolve 之后写 session。** 校验通过后，Broker 会把解析出的身份信息写进 `toolProxySessionStore`（Redis）。后续的工具调用不用每次都重新 resolve token，而是直接读 session。这是性能优化——token 校验涉及 DB 查询（查 hash、查 membership），每次工具调用都跑一遍太贵。session 有 TTL，过期后重新 resolve。

---

## PAT 的强制绑定：最小爆炸半径原则

三种令牌里，PAT 最值得单独深挖——因为它是真人用的、长期有效的，风险最高。

回看 PAT 的强制绑定 `defaultProjectId`。这个设计体现的是**最小爆炸半径原则**：

```
方案 A（全局 PAT）:  一把 PAT = 用户全部权限
    泄露 → 攻击者拿到该用户在所有项目的权限
    爆炸半径 = 用户的所有项目

方案 B（项目绑定 PAT, WinMatrix 的选择）:  一把 PAT = 用户在某一个项目的权限
    泄露 → 攻击者只拿到该用户在该项目的权限
    爆炸半径 = 单个项目
```

WinMatrix 选了 B。代价是用户要为每个项目分别生成 PAT，稍微麻烦。但收益是泄露时的影响被严格限制在单项目。

再加上 membership 校验——即使用户本来是项目成员，后来被移出了，PAT 自动失效。这是一种"权限撤销联动"：权限的变更（移出项目）会自动传导到凭证层（PAT 失效），不需要管理员记得去单独 revoke。

PAT 还有 `expiresAt`（过期）和 `revokedAt`（主动撤销）双重兜底。过期是自动的，撤销是手动的。`lastUsedAt` 字段则支持审计——能看出某把 PAT 最近有没有被用过，长期不用的应该 revoke。

**这几层兜底合起来，是长期凭证的标准防护套件：强制限定范围 + 权限联动 + 过期 + 可撤销 + 可审计。** 缺任何一层，都是可被利用的漏洞。

---

## 为什么不直接用 JWT

讲完三种 token，有人会问：为什么不把外部程序访问也做成 JWT？给每个外部程序签发一个 JWT 不就行了？

技术上当然可以。但 JWT 有几个特性让它在"外部程序长期访问"场景下不太合适：

**JWT 是有状态的语义载体。** JWT 的 payload 里装着 userId、isAdmin 这些字段，这些字段是"签发时的快照"。如果用户权限变了（比如从管理员降成普通用户），已签发的 JWT 里还是旧值——除非配合 Redis 黑名单（第 28 篇）。黑名单是兜底，但它把"无状态 JWT"变成了"事实上有状态"。

**JWT 不擅长"绑定上下文"。** PAT 强制绑定 defaultProjectId、WMA 绑定 tools——这些是"凭证与上下文强耦合"的语义。JWT 要表达这个，得把 projectId/tools 塞进 payload，但 payload 一旦签发就不可变，上下文变了（用户换了项目）JWT 就失效了。而 PAT 是可查库的——每次 resolve 都现查当前状态，上下文变了立刻反映。

**JWT 的撤销是重量级的。** 撤销 JWT 要么等过期，要么进 Redis 黑名单。而 PAT/WMA/WMEC 的撤销是轻量级的——`revokedAt` 设个时间戳、或 `status` 改一下，下次 resolve 自然就拒绝了。

所以 JWT 适合"真人登录会话"这种**短期、用户在场、上下文固定**的场景；PAT/WMA/WMEC 适合"外部程序长期访问"这种**长期、用户不在场、上下文可能变**的场景。两者分工，不是替代关系。

---

## Token 体系与安全护栏的配合

最后把这三种 token 放回第 28 篇讲的多层防御里，看它们站在哪道墙。

```
请求进来（带 token）
   │
   ├─【第一道】认证（你是谁）
   │     JWT → JwtService.verifyToken
   │     PAT/WMA/WMEC → TokenBroker.resolve（按前缀路由）
   │     ↓ 解析出身份（用户/Agent/应用 + 项目/工具范围）
   │
   ├─【第二道】授权（你能干什么）
   │     PAT → membership 校验（用户还在这个项目吗）
   │     WMA → registration.tools 限定（只能调这些工具）
   │     ↓ project_tool_policy 多维策略（项目×角色×员工×工具）
   │
   ├─【第三道】输入（声明式操作授权）
   │     ... 与 token 类型无关，统一走 declaredOperations
   │
   └─ ...
```

**Token 解析在第一道（认证），token 携带的范围限定在第二道（授权）。** PAT 的项目绑定、WMA 的 tools 限定，都是"认证时确定身份、授权时收窄范围"的两步配合。

这套配合的关键是：**token 解析出的不是"能不能访问"的最终答案，而是"身份 + 范围"的输入**。最终能不能做某件事，还要过 project_tool_policy 这道多维策略。token 给的是身份，策略给的是裁决。两者独立，各自演进——token 体系可以不变，而策略可以随时调整。

---

## 给后来者的几条总结

1. **不同身份语义要用不同 token，别一种 JWT 走天下。** 真人用户、外部 Agent、外部应用，三种身份的权限边界和存储模型都不同。硬塞进一种 token 会丢失语义。
2. **前缀路由是凭证体系的开闭原则。** 每种令牌强制带前缀，Token Broker 按前缀分流。新增令牌类型只加 resolver，不改入口。
3. **PAT 强制绑定项目，最小爆炸半径。** 长期凭证泄露的影响必须被限制住。一把 PAT = 某用户在某项目，不是全局。
4. **需还原用加密、需验证用 hash。** external_agent_registration 同时有 apiKeyHash（查找）和 apiKeyEncrypted（调用）；PAT 只存 tokenHash（只验证不还原）。模式要看场景。
5. **membership 校验是权限撤销联动。** 用户被移出项目，PAT 自动失效。权限变更要传导到凭证层，不能靠管理员记得 revoke。
6. **resolve 之后写 session，别每次都查库。** token 校验涉及 DB 查询，高频调用要配 Redis session 缓存 + TTL。
7. **JWT 适合短期会话，长期外部访问用专门令牌。** JWT 的无状态在"上下文可能变"的长期场景下反而是负担。PAT/WMA/WMEC 是可查库的，上下文变了立刻反映。
8. **token 给身份，策略给裁决。** token 解析只输出"身份 + 范围"，最终授权要再过策略层。两者独立演进。

凭证体系是 AI 平台对外的门面之一——所有外部程序访问都从这里进门。把这道门设计成"前缀路由 + 多令牌分工 + 强制绑定 + 多层校验"，是让系统既开放又可控的基础。

---

> **下一篇**：[《ExecutionSpan 的事件双写：JSON 数组 + 子表，为什么要两份》](./50-execution-span-events.md)——第 08 篇讲过 ExecutionSpan 的整体设计，这一篇专挖一个细节：span 的事件为什么同时存在 JSON 数组字段和独立子表里，这种"双写"的一致性边界在哪。
