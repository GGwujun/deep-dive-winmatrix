# 透明代理并非透明：PgBouncer/连接池/中间件的隐含状态

> 这是《WinMatrix 开发经验文集》第 54 篇。上一篇讲可重放性，最后留了一个钩子："你以为透明的代理，其实改了你的会话语义。"这一篇就专门拆这个坑。所有用过 PgBouncer、用过连接池、用过任何"中间件"的人都踩过这个坑——文档上说"对应用透明"，生产上它悄悄改了你的时区、你的会话、你的连接生命周期，然后某个深夜你的 LISTEN/NOTIFY 突然不工作了，而你完全不知道为什么。"透明"是这个行业最大的谎言之一，这一篇讲怎么识破它。

先讲一个真实的故事。

WinMatrix 早期，配置热更新用的是 PG 的 LISTEN/NOTIFY：进程连上 PG，`LISTEN config_change`，配置一改，DB 发 `pg_notify('config_change', ...)`，进程收到通知后刷新缓存。这套机制在开发环境跑得好好的，上了生产突然不工作——配置改了，进程收不到通知，缓存一直不刷新。

排查了好久，最后发现：**生产环境用了 PgBouncer 做 transaction pooling，而 LISTEN/NOTIFY 在 transaction pooling 模式下根本不工作。** PgBouncer 把每个事务分配到不同的后端连接上，事务结束连接就归还，LISTEN 注册的会话状态根本保留不住——你以为你在一条连接上 LISTEN，实际上你的每个 NOTIFY 走的是不同的后端连接，根本到不了你的 LISTEN 耳朵里。

这就是"透明代理并非透明"的典型惨案。PgBouncer 的文档当然写了"transaction pooling 不支持会话级特性"，但这句话埋在几十页文档的某个角落，而"对应用透明"写在第一页。**代理悄悄改了你的会话语义，而你的代码完全不知情。**

这一篇拆三类最常见的"伪透明"中间件，以及 WinMatrix 是怎么在源码层面绕过它们的。

---

## 第一类：PgBouncer——会话状态的隐形杀手

PgBouncer 是 PG 的连接池，5000 人规模的部署几乎必备（避免 too many clients）。它有三种 pooling 模式：

```
session pooling      每个客户端独占一条后端连接，连接生命周期 = 会话生命周期
transaction pooling  每个事务借一条连接，事务结束归还（默认，最高效）
statement pooling    每条语句借一条连接（很少用）
```

文档说"transaction pooling 对应用透明"。**这是真的吗？** 不是。它只对"无状态"的应用透明。一旦你的应用依赖任何**会话级状态**，transaction pooling 就会悄悄把你搞挂：

| 会话级特性 | session pooling | transaction pooling |
|-----------|----------------|---------------------|
| LISTEN/NOTIFY | 正常 | **挂**（监听注册留不住） |
| 会话变量（SET） | 正常 | **可能丢**（每个事务重置） |
| 临时表 | 正常 | **挂**（事务结束即销毁） |
| advisory lock（会话级） | 正常 | **挂**（锁随连接归还释放） |
| prepared statement（会话级） | 正常 | **可能挂** |

WinMatrix 的配置热更新恰好就是 LISTEN/NOTIFY。它的解法很直接——**LISTEN/NOTIFY 必须走真实 PG 会话，绕开 PgBouncer**（`ConfigDbListener.ts:90-112`，核实报告 ch04-06）：

```ts
// ConfigDbListener 约束（行 90-112）：
// 必须走真实 PG 会话，不能经 PgBouncer transaction pooling
// 优先取 DATABASE_LISTEN_URL（直连 PG，绕开 PgBouncer）
```

具体做法是给 ConfigDbListener 配一个**独立的 DATABASE_LISTEN_URL**，指向直连 PG 的地址（不经 PgBouncer）。这个连接用独立的 `pg.Client`（不用连接池），`LISTEN config_change`，整条连接的生命周期和进程一样长（核实报告 ch04-06，`connect()` 行 209-236）。

```
应用进程
  ├── 普通 DB 操作 → PgBouncer（transaction pool）→ PG（高效）
  └── LISTEN/NOTIFY → DATABASE_LISTEN_URL 直连 PG（绕开 PgBouncer）
```

**这是识破"伪透明"的第一招：识别哪些操作依赖会话状态，给它们开一条"绕过代理"的专用通道。** 不要指望代理会兼容你的会话语义——代理的设计目标就是抹掉会话语义来换吞吐，这是它的本职工作，不是 bug。

---

## 第二类：连接池——TCP 生命周期被悄悄改写

PgBouncer 改的是"会话语义"，更底层的连接池（pg.Pool）改的是"TCP 生命周期"。这一层更隐蔽，因为它的"伪透明"不是功能性的，而是**故障语义**上的。

Prisma 在 WinMatrix 里通过一个 pg.Pool 连 PG（`src/infrastructure/persistence/prisma/client.ts:111-123`，核实报告 ch04-06）：

```ts
createPrismaPool() — pg.Pool（PRISMA_POOL_MAX 默认 25、keepAlive 10s）
```

这个池子会复用连接。问题来了：**当 PG 重启、网络瞬断、PgBouncer 回收后端连接时，池子里那条"看起来还活着"的连接，其实已经死了。** 你的下一个查询发出去，半天没响应，最后抛个 `connection terminated` 或 `ECONNRESET`。

如果你用"裸 Prisma"，这个错误会直接抛给业务层，用户看到 500 错误。**连接池在这里"不透明"的点在于：它把"PG 挂了一下"这个瞬时事件，放大成了"用户请求失败"。** 直连 PG 时，PG 挂了你的连接也断了，但你能立刻知道；经过连接池后，连接池会假装连接还活着，直到你真正用时才炸。

WinMatrix 的 Prisma Client 封装了一套完整的自愈机制来对付这个"伪透明"（`client.ts:223-236`，`isRecoverablePrismaPoolError` 识别 9 种可恢复错误模式）：

```
请求 → Prisma 调用 → 连接池错误
                      ├── 是可恢复错误？（9 种模式）
                      │     ├── 是 → single-flight 重建 → 只读方法重放
                      │     └── 否 → 抛错（让业务层处理）
                      └── 不可恢复错误 → 直接抛错
```

这里的几个设计都是针对"连接池的伪透明"的：

- **`keepAliveInitialDelayMillis=10s`**：让池子主动探测连接死活，而不是被动等用户请求触发才发现。
- **single-flight 重建**（`rebuildPrismaResources`，行 193-221）：并发请求同时遇到连接池错误时，只重建一次，其他请求等同一个 Promise。否则连接池错误会触发 N 个请求各重建一次，雪崩。
- **只读重放**（`withPrismaRecovery`，行 305-332）：重建后只对只读方法（findMany/findFirst/aggregate/count/$queryRaw 等 9 种）自动重放一次，写操作不重放（下一篇专门讲为什么）。

**识破"伪透明"的第二招：连接池会改写故障语义，你必须有一层"自愈封装"在它之上。** 这层封装要做三件事——识别可恢复错误、single-flight 重建、按方法语义决定是否重放。没有这层，连接池的"透明"就是把你暴露在所有瞬时故障下。

---

## 第三类：时区——被连接字符串偷偷改掉的全局状态

第三类"伪透明"最阴险，因为它改的是一个**全局状态**：时区。

WinMatrix 强制所有 DB 时间用 UTC（`buildConnectionString` 行 71-90，核实报告 ch04-06）：

```ts
buildConnectionString() — 强制 options=-c TimeZone=UTC（规避 Prisma #28629）
```

为什么要强制？因为 Prisma 有个老 bug（#28629）：如果连接里不带时区设置，Prisma 在某些场景会把时间字段的时区算错，导致存进去的和读出来的差 8 小时。这个 bug 在开发环境（直连 PG，时区一致）不会暴露，但上了生产（经过 PgBouncer、经过容器、经过 K8s Pod 的 TZ 设置）就炸。

**这里"伪透明"的点是：你以为连接字符串只是个地址，其实它携带的 `options=-c TimeZone=UTC` 是一个会话级的全局状态。** 这个状态决定了所有 `TIMESTAMP` 字段怎么解释。一旦中间件（PgBouncer、连接池、甚至 ORM 本身）改了这个状态，你的时间数据全错位，而且错得非常隐蔽——数据看着是有的，就是差几个小时。

WinMatrix 对此的防御是多层叠加的（核实报告 ch23-29，时间语义约定）：

| 层 | 约束 | 作用 |
|----|------|------|
| 连接串 | `options=-c TimeZone=UTC` | 会话级强制 UTC |
| Schema | Prisma DateTime 必须 `@db.Timestamptz(6)` | 字段级强制带时区 |
| 应用代码 | 禁 `toISOString()` 流出、禁硬编码 `Asia/Shanghai` | 防止应用层再转换 |
| 启动检查 | `warnIfTimezoneInconsistent()` | Node TZ vs PG 会话时区不一致告警 |

**识破"伪透明"的第三招：全局状态（时区、字符集、locale）会被任何中间件改写，必须从连接串、schema、应用代码、启动检查四层一起锁死。** 少任何一层，某天某个中间件一升级，你的时间数据就开始错位，而且很难发现。

---

## 怎么识别"伪透明"：三问

这三类坑凑在一起，能提炼出一个识别"伪透明中间件"的通用方法。每次引入一个新中间件（代理、连接池、ORM、网关），问自己三个问题：

**第一问：这个中间件有没有"会话"的概念？**
如果有（LISTEN/NOTIFY、临时表、会话变量、advisory lock），那它几乎一定会在 transaction pooling / 连接复用下出问题。**给依赖会话语义的操作开专用通道，绕开中间件。**

**第二问：这个中间件会不会"假装连接还活着"？**
连接池和代理都会。**在它之上加一层自愈封装**：识别可恢复错误、single-flight 重建、按方法语义决定是否重放。别让中间件的故障语义穿透到用户。

**第三问：这个中间件改没改全局状态（时区、字符集、locale）？**
几乎都会改，而且不会告诉你。**从连接串、schema、应用代码、启动检查四层一起锁死。**

这三问的本质是：**"透明"是一个相对概念——中间件对"无状态、无特殊语义"的调用透明，对"有状态、依赖会话语义"的调用完全不透明。** 你的代码里只要有任何一处依赖会话/全局状态，就要假设中间件会破坏它。

---

## WinMatrix 里的"伪透明"清单

把 WinMatrix 里所有踩过这个坑的地方列出来，你会发现它是一个**完整的风险地图**：

| 中间件 | "伪透明"点 | WinMatrix 的解法 | 源码 |
|--------|-----------|-----------------|------|
| PgBouncer（transaction pool） | LISTEN/NOTIFY 不工作 | 独立 DATABASE_LISTEN_URL 直连 PG | ConfigDbListener.ts:90-112 |
| pg.Pool（Prisma） | 连接死了假装活着 | 9 种可恢复错误识别 + single-flight 重建 + 只读重放 | client.ts:223-236 / 193-221 / 305-332 |
| Prisma ORM | 时区 bug #28629 | 强制 `options=-c TimeZone=UTC` | client.ts:71-90 |
| Prisma ORM | DateTime 时区解释 | `@db.Timestamptz(6)` + 禁 toISOString | schema 全表 + AGENTS.md 约定 |
| K8s Pod | TZ 环境变量污染 | 显式覆盖 NODE_ENV + 启动 warnIfTimezoneInconsistent | k8s/deployment.yaml + startup/common.ts |

这张表不是一次设计出来的，是**每次事故补一行**长出来的。每个坑都对应一次生产故障。这也是为什么"透明代理并非透明"是一条工程哲学而不是一个技术点——**你永远不知道下一个中间件会在哪里背叛你，你只能建立一套"识别 + 绕过 + 自愈 + 锁死"的通用肌肉记忆。**

---

## 一个更深的观察：中间件越多，隐含状态越多

这一篇表面在讲 PgBouncer 和连接池，背后其实是一条更普适的规律：**系统里每多一层中间件，就多一层隐含状态，可重放性就降一级。**

为什么？因为可重放性要求"同样的输入产生同样的输出"（参考上一篇）。但中间件引入的隐含状态（会话、时区、连接生命周期）会让"同样的输入"在不同的中间件状态下产生不同的输出。你在开发环境（直连 PG、UTC、单一连接）复现的 bug，到生产（PgBouncer、容器时区、连接池复用）就复现不出来，因为隐含状态不一样。

这也是为什么 WinMatrix 的 E2E 测试**坚决不 mock 数据库**（核实报告 ch23-29，tests/e2e/setup.ts:6-35）——mock 掉的数据库是没有 PgBouncer、没有连接池、没有时区问题的"理想 PG"，在它上面跑通的测试到了生产毫无意义。**真实环境才有真实的隐含状态**，你要么在真实环境测，要么就被中间件的"伪透明"坑到生产。

---

## 给后来者的总结

1. **"透明"是相对的。** 中间件只对"无状态、无特殊语义"的调用透明。任何依赖会话状态、全局状态、特定连接生命周期的操作，都要假设中间件会破坏它。
2. **LISTEN/NOTIFY、临时表、会话级 advisory lock 不能走 PgBouncer transaction pooling。** 给这些操作开一条绕过代理的专用通道（如独立的直连 URL）。
3. **连接池会假装死连接还活着。** 在它之上加一层自愈封装：识别可恢复错误（WinMatrix 识别 9 种模式）、single-flight 重建防雪崩、按方法语义决定是否重放。
4. **时区是会被中间件悄悄改写的全局状态。** 从连接串（`options=-c TimeZone=UTC`）、schema（`@db.Timestamptz(6)`）、应用代码（禁 toISOString）、启动检查（warnIfTimezoneInconsistent）四层一起锁死。
5. **引入新中间件时问三问：有没有会话语义？会不会假装连接活着？改没改全局状态？** 三个问题里有任何一个"是"，就要为它准备专门的防御。
6. **中间件越多，隐含状态越多，可重放性越差。** 每一层中间件都是可重放性的敌人。要么减少中间件，要么在测试里用真实环境（含中间件）验证。
7. **E2E 测试不要 mock 数据库。** mock 掉的数据库是没有隐含状态的理想环境，在它上面跑通的测试到生产会被"伪透明"坑死。

"透明代理"是这个行业最成功的营销话术之一。它让你以为可以不理解中间件内部就享受它的好处。现实是：每一个"透明"的中间件都在偷偷改你的语义，区别只是你什么时候发现。早发现的写进了 fixture（上一篇），晚发现的写进了事故复盘。**工程师对中间件的态度应该是：默认它不透明，除非你证明了它在你这个场景下透明。** 反过来相信，就是把自己交付给运气。

---

> **上一篇**：[《可重放性：一个 bug 能不能"按原样再跑一遍"》](./53-replayability.md)
>
> **下一篇**：[《先建新后关旧：热替换的节奏》](./55-create-before-destroy.md)——透明代理的坑讲完了，但有一种操作会把"透明"的假设彻底打破，那就是"换资源"。下一篇讲连接池重建和滚动部署共同遵循的一条节奏：先建新，后关旧。
