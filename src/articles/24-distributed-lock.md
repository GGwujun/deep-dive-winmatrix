# 分布式锁分工：Redis SET NX + Lua 与 PG advisory lock 各管什么

> 这是《WinMatrix 开发经验文集》第 24 篇。上一篇讲了幂等——防"同一逻辑请求被重复执行"。但有些场景光幂等不够，还需要"同一时刻只有一个执行者"。这就用到锁。WinMatrix 里有两套锁机制：一套是 Redis 的 `SET NX EX` + Lua 脚本，另一套是 PG advisory lock。有意思的是，后者几乎退成了 no-op。这篇讲清楚它们各管什么、为什么这么分工。

先说一个让很多人意外的事实：**WinMatrix 里 PG advisory lock 基本是个 no-op。**

```ts
// src/infrastructure/persistence/advisoryLock.ts（15 行）
// PG advisory lock 已退化为 no-op，仅启动期占位
```

这 15 行的文件，函数体基本是直接 return。但同一个系统里，Redis 锁却是实打实在干活的：JWT 黑名单、定时任务 leader 选举、kickoff 任务互斥，全靠它。为什么会有这种"一套在干活、一套退化了"的分裂？

要从两种锁的本质差异说起。

---

## 两种锁，两种哲学

分布式锁有两种典型实现路径：**基于 Redis 的内存锁**和**基于关系数据库的锁**。它们的设计哲学完全不同。

### Redis 锁：快、轻、但不是绝对可靠

Redis 锁的实现（`src/infrastructure/persistence/distributedLock.ts`，51 行）：

```ts
export async function acquireKickoffLock(jobId: string, owner: string, ttlSeconds = 30): Promise<boolean> {
  const redis = await getRedisClient();
  const result = await redis.set(lockKey(jobId), owner, 'EX', ttlSeconds, 'NX');
  return result === 'OK';
}
```

经典的 `SET key value NX EX`：key 不存在才设置，同时给 TTL。这个原子操作完成"加锁 + 设过期"两件事。返回 `OK` 表示抢到了，`nil` 表示别人占着。

释放锁走 Lua 脚本（伪代码）：

```lua
-- 只有 owner 匹配才删，防止误释放别人的锁
if redis.call('get', KEYS[1]) == ARGV[1] then
  return redis.call('del', KEYS[1])
else
  return 0
end
```

**这里有一个极易踩的坑：释放锁必须校验 owner。** 想象这个时序：

```
worker A 加锁（TTL 30s）→ 业务执行慢，超过 30s → 锁自动过期
worker B 加锁成功 → 此时 worker A 终于来释放锁
如果不校验 owner：worker A 把 worker B 的锁删了 → 互斥失效
```

所以释放时一定要带 owner 比对，用 Lua 保证"比对+删除"是原子的。**这是 Redis 锁的铁律。**

Redis 锁的优点是快（毫秒级）、实现简单、不依赖数据库。缺点是**不是绝对可靠**：Redis 主从切换时可能丢锁，TTL 到期时业务还没跑完也会"假释放"。所以 Redis 锁适合"对偶发失效有一定容忍度"的场景。

### PG advisory lock：事务级强一致，但要占用 DB 连接

PG 原生提供 advisory lock——应用层的、不绑定具体数据的锁。它的特点是**和事务/会话绑定**：session 级的锁只要连接不断就一直在，事务级的锁 commit/rollback 自动释放。

这种锁强一致、可靠，但代价是：**持锁期间必须占着一条 DB 连接。** 而且它要求"加锁和业务操作走同一个 session"——这就约束了它的使用方式（参考第 4 章的 `withPrismaPoolClient`，在同一 pg 连接里跑 session 语义）。

两套锁对比一目了然：

| 维度 | Redis 锁 | PG advisory lock |
|------|---------|------------------|
| 速度 | 毫秒级 | 较慢（要走 DB） |
| 可靠性 | 不是绝对可靠（TTL/主从） | 强一致 |
| 占用资源 | 不占 DB 连接 | 持锁期间占一条 DB 连接 |
| 失效方式 | TTL 自动过期 | 事务结束/连接断开 |
| 适合场景 | 运行时高频互斥 | 启动期/批处理强一致串行 |

理解了这两套的取舍，就能看懂 WinMatrix 为什么这么分工。

---

## 运行时用 Redis 锁：三个典型场景

运行时（系统在正常服务期间）所有需要锁的地方，几乎都走 Redis。看三个典型场景。

### 场景 1：定时任务 leader 选举

定时任务系统里，多个 scheduled-worker 实例可能同时启动，但**注册默认定时任务这件事必须只有一个实例做**，否则会重复注册、重复触发。

`scheduledSyncLeader.ts`（116 行）：

```ts
const LOCK_KEY = 'scheduled:sync:leader';
const LOCK_TTL_SECONDS = 120;
const RENEW_INTERVAL_MS = 40_000;

export async function runWithScheduledSyncLeader(syncFn: () => Promise<void>): Promise<boolean> {
  const acquired = await tryAcquireScheduledSyncLeader();   // SET NX EX 120
  if (!acquired) return false;
  startScheduledSyncLeaderRenewal();                        // 后台 40s 续租
  try { await syncFn(); return true; }
  finally {
    stopScheduledSyncLeaderRenewal();
    await releaseScheduledSyncLeader();                     // Lua 比对 fence 释放
  }
}
```

注释写得很清楚：抢锁用 `SET NX EX 120`，续租用 Lua 比对 fence（`hostname:pid:timestamp`）续。**fence 就是上面说的 owner——同一个概念，叫法不同。** 续租每 40 秒一次，远小于 TTL 120 秒，留足缓冲。

为什么这里用 Redis 而不是 PG？因为这个锁在系统启动时抢、抢到后要持有比较久（要把所有默认任务注册完）。如果占 PG 连接，会挤占宝贵的连接池资源。Redis 锁轻量，不占 DB 连接，是更合适的选择。

### 场景 2：JWT 黑名单（隐式锁）

JWT 黑名单严格说不是"互斥锁"，但用了同样的机制——`setex(jwt:blacklist:{token}, ttl, '1')`（核实报告 ch04-06 JwtService.ts:131-158）。注销时把 token 写进黑名单，TTL 跟 token 剩余有效期一致；校验时查这个 key 在不在。

这其实是一种"标记锁"——锁的不是并发，而是"这个 token 已失效"。用 Redis 是因为读频率极高（每次请求都要校验），PG 扛不住这个查询量。

### 场景 3：kickoff 任务互斥

`acquireKickoffLock(jobId, owner, ttlSeconds)` 就是上面那段源码。kickoff 是任务启动的意思，同一个 jobId 不能被两个 worker 同时 kickoff。这种"高频、短时"的运行时互斥，Redis 锁是标配。

**这三个场景的共性是：都是运行时的高频操作，对延迟敏感，且对偶发失效（比如 Redis 主从切换瞬间丢锁）有一定容忍——最坏情况是某个任务被重复触发一次，幂等机制（上一篇讲的）会兜底。**

---

## 启动期本来用 PG advisory lock：但已经退化

PG advisory lock 在 WinMatrix 里曾经用在启动期的状态收敛上。看 reconcile 的代码（`ScheduledTaskService.ts:2244-2298`）：

```ts
const thresholdSeconds = Math.max(Math.floor(config.scheduled.agentJobTimeoutMs / 1000), 30 * 60);
const locked = await withReconcileLock(config.reconcileAdvisoryLockKey, async () => {
  const workflowRows = await convergeRunningScheduledWorkflowRuns(500);
  const scheduledRows = await reconcileStaleRunning(thresholdSeconds);
  const pipelineRows = await getAgentRunRepository().reconcileStaleRunning(thresholdSeconds);
  const candidates = await getAgentRunRepository().listPartialScheduledRunCandidates(500);
  // ...
});
```

启动时要做的"孤儿任务回收"——把上次崩溃时停在 running 状态的 workflow/scheduled/pipeline run 收敛到终态。这件事**多个实例不能同时做**，否则会互相干扰。所以用 `withReconcileLock` 串行化。

但这里有个微妙的事实：**`advisoryLock.ts` 那个文件已经退化为 no-op 了。** 这意味着 `withReconcileLock` 内部调用 advisory lock 的那一步，实际上是空操作。

这怎么可能？串行化不就失效了？答案是：**串行化的职责被移到了别处。** 启动期本来就有更强的保障：

1. **进程角色守卫 + 启动顺序**：api/scheduled/rag 三进程分离，每个入口有 `assertProcessRole` 守卫。scheduled-worker 在启动序列里跑 reconcile，而生产环境通常只有一个 scheduled-worker 实例（参考 docker-compose：`winmatrix-scheduled-worker` 是单实例）。
2. **leader 选举兜底**：注册默认定时任务前已经用 Redis 锁选过 leader 了（`runWithScheduledSyncLeader`），reconcile 在 leader 上下文里跑。
3. **幂等兜底**：即使 reconcile 真的并发了，每一路 reconcile（`convergeRunningScheduledWorkflowRuns` / `reconcileStaleRunning`）内部的状态变更都是幂等的——把 running 改成 failed 是幂等的，改两次和改一次效果一样。

所以 advisory lock 退化成 no-op，是**系统演进的结果**：当更强的保障（单实例 + leader + 幂等）到位后，PG advisory lock 这一层就成了多余的保险，干脆简化掉。这不是设计缺陷，而是"够用就好"的工程判断。

---

## 为什么不全用 Redis

那为什么运行时和启动期不都用 Redis 锁？反过来想：什么场景 PG advisory lock 反而更合适？

PG advisory lock 真正的杀手锏是**和事务绑定的强一致**。考虑这个场景：

```
事务 A：BEGIN → advisory_lock(key) → update balance → commit
事务 B：BEGIN → advisory_lock(key) → 阻塞，直到 A commit → 拿到锁 → update balance → commit
```

PG advisory lock 在事务 commit 时自动释放，**锁的生命周期和事务严格绑定，不可能出现"锁释放了但事务还没 commit"或反过来**。这种"锁就是事务"的语义，Redis 锁做不到——Redis 锁是应用层的，和 DB 事务是两回事，强行拼在一起会有"锁释放了但事务还没提交"的窗口。

所以如果你的互斥需求是**"保护一段 DB 事务的完整性"**，PG advisory lock 比 Redis 锁更正确。WinMatrix 里 `withPrismaPoolClient`（同一 pg 连接跑 session 语义，第 4 章）就是为这种用法准备的——它保证 advisory lock 和后续的业务 SQL 走同一条连接。

但 WinMatrix 大部分互斥场景不要求"锁事务一致"——定时任务 leader、JWT 黑名单、kickoff 互斥，这些都不涉及"锁必须和某个事务绑定"。所以 Redis 锁是更轻、更合适的选择。**advisory lock 退化，正是因为系统里"强一致锁事务"的场景不多。**

---

## 释放锁的细节：owner 校验为什么是铁律

回过头强调一下 Redis 锁释放时的 owner 校验，这是分布式锁最容易踩的坑。

很多人写 Redis 锁的释放是这样：

```ts
// 错误写法
async function releaseLock(key: string) {
  await redis.del(key);
}
```

这个写法在 99% 的时候能工作，但 1% 的时候会出大事故。事故时序：

```
T0: worker A acquireLock(key, ownerA, ttl=30s)
T0~T30s: worker A 业务执行慢
T30s: 锁自动过期
T31s: worker B acquireLock(key, ownerB, ttl=30s) 成功
T32s: worker A 终于执行完，调用 releaseLock(key)
      → 错误写法直接 del → 把 worker B 的锁删了！
T33s: worker C acquireLock 成功 → 此时 B 和 C 都以为自己持有锁 → 互斥失效
```

正确的写法必须带 owner 比对，而且"比对+删除"必须是原子的（用 Lua）：

```lua
if redis.call('get', KEYS[1]) == ARGV[1] then   -- owner 匹配？
  return redis.call('del', KEYS[1])
else
  return 0                                       -- 不是我的锁，不删
end
```

WinMatrix 的 `releaseScheduledSyncLeader` 用的就是这个模式，fence 字段就是 owner。**任何用 Redis 锁的系统，释放时都要这样校验，没有例外。**

---

## 锁和幂等的分工

把上一篇和这篇放一起看，会发现锁和幂等是配合使用的：

```
请求来了
  ├── 第一道防线：幂等键（去重）
  │     → 重复的请求直接复用已存在结果，根本不进入执行
  │
  └── 第二道防线：锁（互斥）
        → 真正进入执行的请求，如果访问共享资源，加锁防并发
              └── 锁失效了？幂等再来兜底
```

**幂等是"软"防线，锁是"硬"防线。** 幂等让重复请求无害化，锁让并发请求串行化。两者结合，系统才能在面对"网络重传 + 并发抢占"的双重压力时保持一致。

WinMatrix 里典型的组合是：

- role_inbox：幂等键（去重）+ claim 租约（互斥）。
- flow_instruction：idempotencyKey（去重）+ claimToken+lease（互斥）。
- CodingTask：复合键（去重）+ attemptNo（乱序防护）+ callbackTokenHash（鉴权）。

**每一个"会重复、会并发"的操作，都是幂等和锁的组合拳。** 单用锁不够（锁失效就崩），单用幂等也不够（并发执行会互相覆盖中间状态）。

---

## 给后来者的几条总结

1. **两套锁不是冗余，是分工。** Redis 锁管运行时高频互斥（轻、快、容忍偶发失效），PG advisory lock 管"锁必须和事务绑定"的强一致场景。WinMatrix 里后者场景不多，所以退化为 no-op。
2. **Redis 锁用 `SET NX EX` 原子加锁 + TTL**，别用 `SETNX` + `EXPIRE` 两步（非原子，崩了不释放）。
3. **释放锁必须 owner 校验，用 Lua 保证原子。** 直接 `del` 是定时炸弹，TTL 过期后会误删别人的锁。
4. **leader 选举用 Redis 锁 + 续租**。TTL 120s + 续租 40s 是个合理的缓冲比例（小于 TTL 的 1/2）。
5. **PG advisory lock 的优势是事务级强一致**，持锁期间占 DB 连接。适合"锁必须和某段 DB 操作绑定"的场景。
6. **advisory lock 退化不是设计缺陷，是"够用就好"的工程判断**。当单实例 + leader + 幂等三层兜底到位后，多余的保险可以简化。
7. **锁和幂等配合使用，不是二选一。** 幂等防重复，锁防并发，两者一起才能扛住"重传 + 并发"的双重压力。
8. **锁失效时幂等要能兜底。** 任何锁都不是绝对可靠的（Redis 主从切换、TTL 过期），设计时假设"锁偶尔会失效"，让幂等在失效时收敛。

分布式锁没有银弹。理解每种锁的哲学——Redis 的"快但不绝对可靠" vs PG 的"强一致但占资源"——然后根据场景选，比死记"用什么锁"重要得多。

---

> **下一篇**：[《优雅降级：当 Redis、ES、BullMQ、LLM 分别挂掉时系统怎么活》](./25-graceful-degradation.md)——锁失效会带来短时不一致，那基础设施整个挂了呢？Redis 挂了走 L1、ES 挂了走 PG、BullMQ 挂了自动重连、LLM 卡住有 watchdog 补偿。讲 WinMatrix 的多层降级策略。
