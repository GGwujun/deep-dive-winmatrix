# 横切关注点别靠纪律：Proxy/中间件/Hook 收口

> 这是《WinMatrix 开发经验文集》第 56 篇。前几篇讲了可重放、透明代理、切换节奏，都是"系统层面"的哲学。这一篇往下挖一层，讲一个每天都在折磨工程师的问题：**横切关注点（cross-cutting concerns）怎么管。** 鉴权、限流、连接恢复、审计、观测……这些事每个请求都要做，但它们和业务逻辑无关。最直觉的做法是"在业务代码里记得写"，但这条路走三个月就崩——总有人忘，总有人写错，总有人为了赶需求绕过去。这一篇讲 WinMatrix 在三个层面（数据库、API、决策管线）怎么用 Proxy / 中间件 / Hook 把横切逻辑对业务代码彻底透明地收口。

先讲一个每个团队都经历过的场景。

系统上线初期，你定了个规矩："每个 API 都要先校验权限。" 大家都遵守了。三个月后，新来了个同学，写了 5 个新 API，其中 2 个忘了加权限校验。安全审计一扫，这两个 API 是越权漏洞——任何匿名用户都能调。你找那个同学，他说"我赶需求，忘了"。你没法怪他，因为**纪律本来就是个会被违反的约定**。

这不是那个同学的问题，是架构的问题。**任何依赖"程序员每次记得写"的横切逻辑，最终一定会被违反。** 时间、压力、人员流动会让任何纪律失效。唯一靠谱的办法是：让横切逻辑在架构上对业务代码**透明**——业务代码根本不需要知道横切逻辑的存在，它自然就被处理了。

WinMatrix 在三个层面实现了这种"透明收口"：Prisma Proxy（数据库层）、Fastify hook（API 层）、pipeline hooks（决策管线层）。这一篇看这三层怎么各自用不同的机制，达成同一个目标。

---

## 第一层：Prisma Proxy——业务代码假装连接池永远健康

上一篇讲过，Prisma Client 遇到可恢复错误会触发 single-flight 重建 + 只读重放。现在要问的是：**这套自愈逻辑放在哪里？**

最直觉的答案是"放在业务代码里"——每次查 DB 都包一层 try/catch，遇到连接池错误就重建重放。这条路是灾难：

```ts
// ❌ 错误做法：横切逻辑散落在每个业务方法里
async function getUser(id: string) {
  try {
    return await prisma.user.findUnique({ where: { id } });
  } catch (err) {
    if (isRecoverablePrismaPoolError(err)) {
      await rebuildPrismaResources(err);
      return await prisma.user.findUnique({ where: { id } });   // 重放
    }
    throw err;
  }
}
```

每个业务方法都这么写，代码量翻倍，而且保证有人会写错（漏判断、重放了写操作、忘了 single-flight）。**这是横切逻辑靠纪律的典型失败。**

WinMatrix 的做法是 **Prisma Proxy**（`createPrismaProxy`，`client.ts:334-366`，核实报告 ch04-06）。它把 PrismaClient 包成一个 ES Proxy：

```
业务代码调 prisma.user.findMany(...)
        │
        ▼
   Prisma Proxy（ES Proxy 拦截）
        │
        ├── 正常情况：透传给真实 PrismaClient
        └── 遇到可恢复错误：
              ├── isRecoverablePrismaPoolError(err)？
              ├── single-flight rebuildPrismaResources()
              ├── shouldReplayAfterPrismaRebuild(method)？
              │     ├── 只读方法（findMany 等 9 种）→ 重放一次
              │     └── 写方法 → 抛错（不重放）
              └── 重建失败 → 抛错
```

业务代码看到的永远是 `prisma.user.findMany(...)`，它**完全不知道**背后有连接池重建、重放、single-flight 这些事。Proxy 在它和真实 PrismaClient 之间，把所有自愈逻辑透明地处理掉了。

```ts
// ✅ 正确做法：业务代码干净，横切逻辑在 Proxy 里
async function getUser(id: string) {
  return await prisma.user.findUnique({ where: { id } });
  // 连接池错误？重建？重放？——业务代码完全不用知道
}
```

**这是横切收口的第一种形态：拦截器（Proxy）。** 它的要点是——横切逻辑集中在一个地方，业务代码调用方式完全不变。新增 100 个业务方法，自愈逻辑自动生效，不需要任何人记得写。

---

## 第二层：Fastify hook——业务路由假装鉴权/限流/审计已经做了

第二个层面是 API。每个 HTTP 请求都要做一堆"和具体业务无关"的事：JWT 校验、权限检查、限流、审计日志、请求 ID 注入……这些事放在路由处理函数里就是灾难。

WinMatrix 用 Fastify 的 **preHandler hook** 体系来收口（核实报告 ch04-06）。Fastify 的 hook 机制允许你在请求生命周期的不同阶段插入逻辑，业务路由完全不感知：

```
HTTP 请求进来
  │
  ▼
[Fastify 框架]
  │
  ├── onRequest hook：提取三路 token（Authorization Bearer / X-Auth-Token / query.token）
  │                  → 注入请求上下文
  │
  ├── preHandler hook：jwtAuth（校验 JWT + Redis 黑名单）
  │                   → createProjectPermissionGuard（项目权限检查）
  │                   → requirePermission / requireRole（细粒度权限）
  │                   → 限流检查
  │                   → 审计日志记录
  │
  ▼
[业务路由处理函数]
  async function handler(req, reply) {
    // 这里能放心假设：进来的请求已经鉴权过、有权限、被限流保护、被审计
    return await doBusinessLogic(req.body);
  }
```

业务路由的处理函数看到的是一个**已经被清洗过**的请求——用户身份已确认（`req.user`）、权限已校验、限流已应用。它只管写业务逻辑，不需要在任何地方 `if (!authenticated) return 401`。

看一个权限守卫的真实签名（核实报告 ch04-06，`projectPermission.ts:45-91`）：

```ts
createProjectPermissionGuard(getMemberRole):
  流程：认证检查 → 系统管理员旁路 → 提取 projectCode → 查角色 → hasProjectPermission
```

这个守卫是作为 Fastify preHandler 注册的，业务路由这样用：

```ts
// 业务路由声明它需要什么权限，权限检查由 hook 自动做
fastify.post('/api/projects/:code/tasks', {
  preHandler: [requirePermission('task:create')],   // ← 横切逻辑收口在 hook
}, async (req, reply) => {
  // 处理函数只写业务
  return await createTask(req.body);
});
```

**新增 100 个路由，只要在路由声明里写 `preHandler: [...]`，所有鉴权/限流/审计自动生效。** 不需要在每个 handler 里手写。这就是横切收口的第二种形态：**声明式 hook（中间件）**。它比 Proxy 拦截更进一步——业务代码不只是"不用写逻辑"，连"调用 hook"都不用，只要在路由配置里声明需求。

WinMatrix 在这一层收口的横切逻辑包括（核实报告 ch04-06）：

| 横切关注点 | 实现 | 位置 |
|-----------|------|------|
| JWT 校验 | createJwtAuthMiddleware（preHandler） | middleware/jwtAuth.ts |
| 三路 token 提取 | getTokenFromRequest | middleware/jwtAuth.ts:112-125 |
| 项目权限守卫 | createProjectPermissionGuard | middleware/projectPermission.ts:45-91 |
| 细粒度权限 | 5 个 preHandler 工厂（requirePermission 等） | middleware/permission.ts |
| 动态 RBAC | PermissionService（Redis 缓存 300s） | rbac/PermissionService.ts |
| MCP token 路由 | TokenBroker 按 token 前缀路由（PAT/WMA/WMEC） | mcp/tokenBroker.ts:353-367 |

注意一个细节：**JWT 黑名单查询在 Redis 出错时 fail-open（返回 false 放行）**（`JwtService.ts:163-177`）。这个"降级策略"也是收口在 hook 内部的，业务代码既不知道有 Redis 黑名单，也不知道它出错时会 fail-open。**横切逻辑的横切逻辑，还是收口在横切层。** 这是"透明收口"的极致——连降级行为都对业务不可见。

---

## 第三层：pipeline hooks——业务决策假装自己被观测了

第三个层面更贴近 Agent 平台。决策引擎是一个 5 阶段管线（ExactRouter → FusionRouter → QuickAcceptGate → DecisionPlanner → DecisionCommitmentDeriver），每个阶段都要记录耗时、输入输出摘要、stage trace 用于观测和调试。

如果每个阶段自己在代码里写"记录开始时间、记录结束时间、发观测事件"，那就是横切逻辑散落——5 个阶段 × 几十行观测代码 = 决策引擎里一半是观测代码。

WinMatrix 的做法是 **pipeline hooks**（`DecisionEngine.ts`，核实报告 ch09-12）：

```ts
// DecisionEngine 构造时接收 hooks: PipelineHook[]
export class DecisionEngine {
  constructor({ exactRouter, fusionRouterStage, ..., hooks }: { hooks: PipelineHook[] }) {
    this.hooks = hooks;
  }

  private async decideInner(input): Promise<DecisionResult> {
    // Stage 1
    await invokePipelineHooks(this.hooks, 'DecisionEngine', 'onStageStart', 'ExactRouter+PlanExtraction', draft);
    const exactMatch = await this.exactRouter.routeAsync(input, resolvedTurn);
    // ...
    // 每个 stage 的 onStageStart / onStageEnd 都通过 invokePipelineHooks 调用
  }
}
```

决策引擎的业务逻辑（路由、规划、承诺派生）和观测逻辑（记录 stage trace、耗时、输入输出）**完全解耦**：

```
决策管线（业务逻辑）                Pipeline Hooks（观测逻辑）
  ExactRouter stage        →→→→→   onStageStart('ExactRouter', draft)
                                    onStageEnd('ExactRouter', result, elapsedMs)
  FusionRouter stage       →→→→→   onStageStart('FusionRouter', draft)
                                    ...
  QuickAcceptGate stage    →→→→→   ...
  DecisionPlanner stage    →→→→→   ...
  CommitmentDeriver stage  →→→→→   ...
```

决策引擎只负责"在 stage 边界调用 hook"，不负责"hook 具体干什么"。观测逻辑（写 span、发指标、记 stage trace）作为 `PipelineHook[]` 注入，可以随时增减而不动决策引擎一行代码。

**这是横切收口的第三种形态：扩展点（hook point）。** 它和 Fastify hook 的区别是——Fastify hook 是框架级的（生命周期固定），pipeline hook 是业务级的（业务自己定义扩展点）。但精神一样：**横切逻辑通过扩展点注入，业务流程只声明"这里有个扩展点"，不关心谁来扩展、扩展什么。**

---

## 三层放一起：同一种精神的三种实现

把三层横切收口放一起对比，能看到同一种精神的三种实现：

| 层 | 形态 | 业务代码看到什么 | 横切逻辑在哪 |
|----|------|----------------|-------------|
| 数据库（Prisma） | ES Proxy 拦截 | `prisma.user.findMany()` | Proxy 内部（重建/重放/single-flight） |
| API（Fastify） | 声明式 preHandler | 一个干净的 handler，只写业务 | hook 链（鉴权/权限/限流/审计） |
| 决策管线（DecisionEngine） | Pipeline Hook 扩展点 | stage 业务逻辑 | hook 实现（观测/trace/指标） |

三层的共性是：**业务代码假装横切逻辑不存在，而横切逻辑在架构上保证它必然被执行。** 这是"透明收口"的核心。

三种形态的区别在于"横切逻辑介入的深度"：

```
Proxy 拦截       最透明（业务代码完全无感，连声明都不用）
      ↓
声明式 hook      较透明（业务代码声明需求，但不写实现）
      ↓
扩展点 hook      较显式（业务代码主动声明扩展点，但不关心 hook 实现）
```

选择哪种形态，取决于横切逻辑的**通用程度**。数据库连接恢复对每个调用都一样 → Proxy 最透明。API 鉴权每个路由需求不同（有的要 admin、有的要 task:create）→ 声明式 hook。决策观测每个 stage 边界不同 → 扩展点 hook。

---

## 为什么纪律守不住：三个真实原因

讲完正确做法，再回头说为什么"靠纪律"这条路注定失败。不是因为程序员不负责任，而是有三个结构性的原因：

**第一，纪律的执行成本是 O(n)。** 每新增一个业务方法/API/路由，都要记得写横切逻辑。系统越大，遗漏的概率越高。而 Proxy/hook 是 O(1)——写一次，所有现有和未来的代码自动受益。**当横切逻辑靠纪律时，系统的安全性和"最近一次代码审查严不严"成正比，这是不可持续的。**

**第二，横切逻辑本身会变。** 今天鉴权用 JWT，明天要加 SSO，后天要加 IP 白名单。如果鉴权散落在 100 个 handler 里，改一次要改 100 处。如果收口在一个 hook 里，改一处。**纪律不仅守不住"记得写"，更守不住"记得改"。**

**第三，横切逻辑的失败模式很危险。** 鉴权忘了写 → 越权漏洞。限流忘了写 → 雪崩。连接恢复忘了写 → 用户看到 500。这些失败都是**静默的**——代码看起来跑得好好的，直到出事。Proxy/hook 把这些危险逻辑收口到单一可信来源，审计时只审那一处，而不是审 100 个 handler。

---

## 反面：什么时候不要收口

公平起见，横切收口也有它不适用的场景。

**当横切逻辑和业务强耦合时，不要强行收口。** 比如"这个 API 需要检查用户是不是项目的 owner"——这不是通用权限（不是"有没有 task:create 权限"），而是业务规则（owner 关系是业务概念）。强行做成 hook 会让 hook 里塞满业务逻辑，反而更乱。**判断标准是：这个逻辑换一个业务还会用吗？会 → 收口；不会 → 留在业务里。**

WinMatrix 的实践是：通用的（鉴权、限流、审计、连接恢复、观测）收口到 Proxy/hook；业务的（项目所有权、任务状态机、流程契约校验）留在业务层。**FlowSkillContractValidationService 这种业务校验，就老老实实是业务服务，不做成 hook。** 这条线划清楚，才能避免"hook 里什么都有"的上帝类。

---

## 给后来者的总结

1. **任何依赖"程序员每次记得写"的横切逻辑，最终一定会被违反。** 纪律是会被时间、压力、人员流动冲垮的约定。横切逻辑必须靠架构收口，不能靠纪律。
2. **数据库层用 Proxy 拦截。** Prisma Proxy 把连接池重建、single-flight、只读重放对业务代码完全透明。业务代码只看到 `prisma.xxx.findMany()`。
3. **API 层用声明式 hook（Fastify preHandler）。** 鉴权、权限、限流、审计收口在 hook 链，业务路由声明需求（`preHandler: [...]`），不写实现。
4. **业务管线用扩展点 hook。** 决策引擎的 pipeline hooks 让观测逻辑（stage trace、耗时、指标）和业务逻辑（路由、规划）解耦。
5. **三种形态的透明度递减：Proxy 最透明，声明式 hook 次之，扩展点 hook 最显式。** 按横切逻辑的通用程度选择——越通用越透明。
6. **横切逻辑的降级行为也要收口在横切层。** JWT 黑名单 Redis 出错 fail-open、ES 写失败降级 PG，这些降级决策对业务代码不可见。业务代码不该知道横切逻辑什么时候降级。
7. **横切和业务的分界线：换一个业务还会用吗？** 会 → 收口到 Proxy/hook；不会 → 留在业务层。强行把业务逻辑做成 hook 会得到上帝类。
8. **审计时只审横切层那一处，而不是审 N 个业务方法。** 这是收口带来的安全收益——可信来源唯一，审计成本从 O(n) 降到 O(1)。

横切关注点的管理，是区分"能跑的系统"和"能维护的系统"的分水岭。前者靠每个程序员自觉，后者靠架构强制。三个月内两者看不出差别，三年后前者变成没人敢动的屎山，后者依然清晰。**当你在纠结"这个逻辑该不该做成中间件"时，默认答案是"该"。** 每一次收口，都是在为系统的长期健康买保险。

---

> **上一篇**：[《先建新后关旧：热替换的节奏》](./55-create-before-destroy.md)
>
> **下一篇**：[《重放 ≠ 永远安全：哪些操作能自动重试》](./57-replay-safety.md)——横切收口让业务代码不用关心自愈，但有一种横切逻辑必须谨慎——自动重试。下一篇讲为什么 Prisma 重建后只重放只读方法、为什么 DelayedError 要看幂等性，以及"重放安全"的判断标准。
