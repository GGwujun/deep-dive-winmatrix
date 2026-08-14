# 我们踩过的坑：Prisma v7 时区、连接池风暴与 single-flight

> 这是《WinMatrix 开发经验文集》第 9 篇。讲三个真实的、让我们深夜爬起来查日志的基础设施事故，以及我们最终怎么修的。代码来自 WinMatrix 后端真实实现，没有杜撰。

做 AI 平台的人，注意力大多在模型、Agent、prompt 上。但真正把你按在凌晨三点的椅子上动弹不得的，往往是那些最"无聊"的基础设施问题——数据库时区、连接池、并发重建。

这一篇就讲三个。每个都是我们在生产环境真实踩过的，每个都有明确的根因和修复方案。读完你会知道：当一个 Node 进程面对 pgBouncer、Prisma 和"整点连接回收"这三者同时存在时，事情会变得多微妙。

---

## 坑一：Prisma v7 的时区，让所有时间悄悄偏移了 8 小时

### 现象

某天 QA 反馈：页面上显示的"任务创建时间"比实际创建时间晚了 8 小时。不是偶发，是**所有时间字段都偏**。`startedAt`、`finishedAt`、`createdAt`，无一幸免。

更诡异的是：数据库里存的时间是对的，本地开发环境也是对的，只有**连了生产 pgBouncer 的环境**出问题。

### 根因

我们用的是 Prisma 7 + `@prisma/adapter-pg`（v7 的 Query Compiler 路径）。问题出在它序列化 `Date` 的方式上：

> `@prisma/adapter-pg` 把 JS 的 `Date` 序列化成一个**不带时区标记的 UTC 墙钟字符串**，然后发给 PostgreSQL。PostgreSQL 收到一个"没有时区的字符串"后，会按**当前 session 的时区**去解释它，再存成 `timestamptz`。

这套机制在本地（session 时区恰好是 UTC）没问题。但生产环境通过 pgBouncer 连接，pgBouncer 给每个 session 设置的默认时区是服务器的本地时区（我们生产是 `Asia/Shanghai`，UTC+8）。于是：

- 一个本该是 `14:00 UTC` 的时间，被当成 `14:00 Asia/Shanghai` 解释，存进去就成了 `06:00 UTC`。
- 读出来再格式化，又偏一次。

结果就是所有时间系统性偏移。这是 Prisma 的已知 issue（#28629）。

### 修复

不在应用层逐个 `Date` 去补时区——那是打地鼠。根治办法是**强制让每个 PG session 的时区为 UTC**，从源头消灭歧义：

```typescript
// src/infrastructure/persistence/prisma/client.ts
function buildConnectionString(): string {
  const base = process.env.DATABASE_URL ?? '';
  // PG session TimeZone 必须设为 UTC —— 规避 Prisma #28629。
  // adapter-pg 把 JS Date 序列化为「无时区标记的 UTC 墙钟串」，
  // PG 收到无时区串后按当前 session 时区解释成 timestamptz。
  // 强制 session 时区为 UTC，从根上消除歧义。
  const tzOption = 'options=-c%20TimeZone%3DUTC';
  return base.includes('?') ? `${base}&${tzOption}` : `${base}?${tzOption}`;
}
```

通过在连接串里追加 `options=-c TimeZone=UTC`，每个 PG session 建立时都被强制设成 UTC 时区。这样一来，无论 adapter-pg 发来的"无时区字符串"被怎么解释，结果都一致。

### 教训

- **不要假设连接的 session 时区**。尤其是经过 pgBouncer、连接池、跨容器部署时，session 时区是"环境的隐变量"。
- **遇到系统性偏移，先查序列化和解释两端**，而不是查业务代码。8 小时这个数字本身就是时区的强烈信号。
- **能从协议层根治的，不要在应用层打补丁**。一个连接串参数，胜过散落在几十个 repository 里的 `toISOString()`。

---

## 坑二：连接池"整点回收"引发的瞬时连接风暴

### 现象

监控里每隔一段时间（基本是整点附近）就出现一波 PostgreSQL 连接数尖峰，紧接着是一批请求报 `ECONNRESET` / `ETIMEDOUT`。频率不高，但每次都让告警群炸一下。

排查后发现：pgBouncer 配置了连接生命周期管理，会在某些时刻批量回收 server 端连接（比如 `server_lifetime`、`server_idle_timeout` 到点）。回收的瞬间，正好压在这批连接上的几十个在途请求，**同时**收到连接错误。

### 根因

第一版修复的思路是"遇到可恢复的连接错误就重建连接池"。听起来合理，但踩了第二个坑：

> 整点回收时，几十个并发请求**几乎同一时刻**都遇到可恢复错误。如果每个请求都独立判断"我该重建连接池了"然后各自调用 `createPrismaResources()`，就会在错误恢复的瞬间又创建一批新连接——形成**二次连接风暴**。旧风暴刚停，新风暴又起。

这是典型的"惊群"（thundering herd）。错误恢复本身放大了错误。

### 修复：single-flight 重建

核心思路：**重建是全局唯一的，并发触发只执行一次，其余等待同一个结果。**

```typescript
// src/infrastructure/persistence/prisma/client.ts
async function rebuildPrismaResources(reason: unknown): Promise<void> {
  // single-flight：整点回收时数十个并发请求会同时命中可恢复错误，
  // 若各自 createPrismaResources() 会形成瞬时连接风暴。
  // 第一个进入者执行真正重建，后续并发调用复用同一个 in-flight Promise，
  // 全部完成后各自继续重试。
  if (globalForPrisma.prismaRebuildInFlight) {
    return globalForPrisma.prismaRebuildInFlight;
  }
  globalForPrisma.prismaRebuildInFlight = (async () => {
    const previousResources = getRuntimeResources();
    const nextResources = createPrismaResources();
    runtimeResources = nextResources;
    publishPrismaResources(nextResources);
    await closePrismaResources(previousResources);  // 先建新的，再关旧的，避免重建期无可用连接
  })();
  try {
    await globalForPrisma.prismaRebuildInFlight;
  } finally {
    globalForPrisma.prismaRebuildInFlight = undefined;
  }
}
```

几个关键点：

1. **`prismaRebuildInFlight` 是一个挂在全局对象上的 Promise 引用**。第一个进来的请求创建它并执行真正的重建；后续并发请求发现它已存在，直接 `return` 那个 Promise，**不会触发第二次重建**。
2. **先建新连接，后关旧连接**（`publishPrismaResources` 先于 `closePrismaResources`）。重建期间系统始终有可用连接，避免重建窗口期的请求雪崩。
3. **重建完成后清空 in-flight 标记**（`finally` 块），确保下次错误仍能触发重建。

### 教训

- **错误恢复代码本身是并发热点**。任何"出错就重建"的逻辑都要问一句：如果 N 个请求同时出错，会不会触发 N 次重建？
- **single-flight 是处理惊群的标准武器**，不只用于连接池——缓存重建、token 刷新、配置热加载都适用。
- **先建新后关旧**是做热替换的基本节奏。很多人写重建是"关旧的 → 建新的"，中间的空窗期就是新一轮故障。

---

## 坑三：让"自动恢复"对业务代码彻底透明

### 现象

前两个坑修完，新问题来了：单点重建逻辑虽然对，但**调用方怎么用**？

如果让每个 repository 自己 `try / catch`、判断是不是可恢复错误、调用 `rebuildPrismaResources`、再重试——那这套逻辑会散落到 45 个 repository 文件里，且每个写法都可能不一样。这是代码腐化的开始。

### 修复：Proxy 包一层

我们最终选择把"可恢复错误的检测 + single-flight 重建 + 重放"整个封装成一个 Proxy，对外暴露的依然是一个看起来普通的 `prisma` 对象：

```typescript
// src/infrastructure/persistence/prisma/client.ts
function withPrismaRecovery(path: readonly PropertyKey[], args: readonly unknown[]): unknown {
  try {
    const result = invokePrismaPath(path, args);
    if (!isPromiseLike(result)) return result;
    return Promise.resolve(result).catch(async (err: unknown) => {
      if (!isRecoverablePrismaPoolError(err)) throw err;  // 不可恢复，直接抛
      await rebuildPrismaResources(err);                   // single-flight 重建
      if (!shouldReplayAfterPrismaRebuild(path)) throw err;// 有的操作不该重放（如非幂等的写入）
      return invokePrismaPath(path, args);                 // 重建后自动重放
    });
  } catch (err) {  // 同步路径同样处理
    if (!isRecoverablePrismaPoolError(err)) throw err;
    return rebuildPrismaResources(err).then(() => {
      if (!shouldReplayAfterPrismaRebuild(path)) throw err;
      return invokePrismaPath(path, args);
    });
  }
}

export const prisma = (globalForPrisma.prismaProxy ?? createPrismaProxy()) as WinMatrixPrismaClient;
```

业务代码完全无感：

```typescript
// repository 里就是普通的调用，背后是带自动恢复的 Proxy
const user = await prisma.users.findUnique({ where: { id } });
```

两个容易忽略的细节：

- **不是所有操作都该重放**（`shouldReplayAfterPrismaRebuild`）。比如某些非幂等的写入、side effect 类操作，重建后盲目重放会重复执行。重放策略要按操作类型区分。
- **同步路径也要处理**（外层 `catch`）。虽然 Prisma 大多返回 Promise，但 Proxy 拦截在属性访问链上，某些路径可能同步抛错。

### 教训

- **横切关注点（cross-cutting concern）不要靠"大家记得写 try/catch"**。靠纪律守不住，靠 Proxy / 中间件 / 装饰器收口才能持久。
- **封装的标准是"调用方能否假装它不存在"**。如果业务代码里出现了"重建连接池"的字样，封装就失败了。
- **重放 ≠ 永远安全**。自动重试必须配一个"哪些操作可以重放"的白名单，否则你只是把瞬时错误换成了数据错误——后者更难查。

---

## 三个坑的共同主线

回头看，这三个坑其实是同一条主线的三个阶段：

1. **时区坑**：连接的隐含状态（session 时区）和序列化协议产生了不一致 → 在**协议层**根治。
2. **风暴坑**：错误恢复逻辑在并发下放大了错误 → 用 **single-flight** 收口。
3. **透明坑**：横切逻辑的散落风险 → 用 **Proxy** 对业务代码彻底隐藏。

它们的共性是：**真正难的从来不是单点逻辑，而是把单点逻辑放进一个有并发、有故障、有几十个调用方的真实系统里时，怎么保证它不退化。**

做 Agent 平台的人常说"prompt 是新的代码"。但只要你的 Agent 还要读数据库，老代码里的那些坑——时区、连接池、惊群——一个都不会少。把它们修扎实，你的 Agent 才有"站得稳"的地基。

---

> **下一篇**：[《从"AI 助手"到"AI 员工"：我们做对和做错的几件事》](./10-reflections.md)——跳出代码，聊聊产品定位、架构取舍和团队协作上的得失。
