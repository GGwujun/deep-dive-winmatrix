# 定时任务系统：16 个系统任务的幂等与补偿

> 这是《WinMatrix 开发经验文集》第 16 篇，也是这一批（11-16）的收尾。前几篇讲了编码工作站、外部 Agent、配置热更新这些"看得见"的能力，这一篇钻进后台，讲那些用户看不见、但一旦坏了整个平台就停摆的东西——**定时任务**。

每个后台系统都有一堆定时任务：清理过期数据、同步转录到记忆、巡检异常状态、回收孤儿任务……它们不直接面向用户，却是系统"自清洁、自修复"的命脉。

WinMatrix 有 16 个系统级定时任务。听起来不多，但每一个都要回答这些问题：

- **多实例部署时，同一个任务会不会被多个实例同时执行？**（多跑）
- **任务执行到一半进程崩了，重启后谁来收敛这个悬挂状态？**（漏跑）
- **任务结果要通知用户（比如企微推送），但推送这步本身也可能失败，怎么办？**（结果投递）
- **cron 配置改了，怎么保证改完只有一个生效的 schedule，不会跑两份？**（schedule 漂移）

这一篇就讲 WinMatrix 的定时任务系统怎么回答这些问题。核心是三个设计：**幂等迁移审计、启动期串行补偿、结果投递 outbox 解耦**。

---

## 先看全貌：16 个任务，三档队列，三触发模式

定时任务的预定义清单在 `infrastructure/scheduled/types.ts` 里，一共 16 个 `SYSTEM_SCHEDULED_TASKS`。节选几个有代表性的：

```typescript
// infrastructure/scheduled/types.ts（第 89-243 行，节选）
export const SYSTEM_SCHEDULED_TASKS: ScheduledTaskConfig[] = [
  { name: 'system-memory-tidy', pattern: '0 3 * * *',
    message: '会话转录 → 长期记忆全量同步', triggerType: 'direct' },
  { name: 'system-log-cleanup', pattern: '0 4 * * *', triggerType: 'direct', ... },
  { name: 'system-coding-task-timeout-sweep', pattern: '*/5 * * * *',
    triggerType: 'direct', ... },
  { name: 'system-reminder-delivery', pattern: '*/1 * * * *', ... },
  { name: 'system-llm-call-watchdog-sweeper',
    scheduleType: 'interval', intervalMs: 600_000, ... },
  { name: 'system-scheduled-run-reconcile', pattern: '*/30 * * * *', ... },
  { name: 'system-observability-cleanup', pattern: '30 4 * * *',
    message: '清理可观测数据...execution_span / pipeline_run / session_transcript', ... },
  // ... 共 16 个
];
```

### 三档队列：按重量分流

这 16 个任务不是塞进一个队列，而是按"重量"分到三档队列：

```typescript
// infrastructure/queue/queue.ts（第 30-53 行）
const scheduledAgentQueue = new Queue(resolveBullmqQueueName('scheduled-agent'), {...});
const scheduledSystemQueue = new Queue(resolveBullmqQueueName('scheduled-system'), {...});
const scheduledLightQueue = new Queue(resolveBullmqQueueName('scheduled-light'), {...});

export function getQueueForTask(taskName: string): Queue<ScheduledJobData> {
  if (SYSTEM_TASKS.has(taskName)) return scheduledSystemQueue;
  if (LIGHT_TASKS.has(taskName)) return scheduledLightQueue;
  return scheduledAgentQueue;   // 默认走最重的 agent 队列
}
```

三档的分工：

| 队列 | 承载 | 特点 |
|------|------|------|
| `scheduled-agent` | LLM 重的任务（要调模型的） | concurrency 受 `llmConcurrency` 限制，防 LLM 过载 |
| `scheduled-system` | DB/ES 维护任务（清理、同步） | 不可 drain（关停时必须跑完） |
| `scheduled-light` | 轻量扫描（每分钟的提醒检查） | 高频、低耗 |

**为什么不一个队列塞所有？** 因为"重任务挤占轻任务"。如果提醒投递（每分钟）和记忆同步（凌晨 3 点）在同一个队列，凌晨的记忆同步跑起来，提醒投递就被堵在后面——用户该收的提醒收不到。分流后，轻任务走自己的队列，永远不被重任务阻塞。

这是 [第 9 篇] 踩坑精神的延伸——**队列是资源，按负载特征隔离资源，才能避免互相挤占。**

### 三触发模式：direct / message / workflow

任务怎么触发？不止"到点跑"一种：

- **direct**：直接调一个方法。最简单，适合纯内部维护任务（清理、同步）。
- **message**：发一条消息触发智能体响应。适合"定时让某个数字员工干活"——比如每天早上让大福生成项目日报。
- **workflow**：绑定一个 flow_template，到点执行整个流程。适合"定时跑一个多步骤编排"。

三种模式覆盖了"定时触发"的完整谱系：从最轻的方法调用，到最重的流程编排。**不是所有定时任务都是"跑个函数"，有的要触发 AI，有的要跑流程。把它们统一进一个任务系统，而不是各搞一套。**

---

## 幂等之一：cron 迁移审计，专治"09:00 尖峰"

定时任务最容易出的一种故障是**配置漂移**：cron 表达式改了，但旧的 schedule 没清干净，新旧两个 schedule 并存，任务被执行两次。

一种特别凶险的漂移叫"09:00 尖峰"。很多任务一开始配的是 `0 9 * * *`（每天 9 点），大家都挤在 9 点，瞬间压力巨大。运维发现后想把它们"摊开"——改成 `0 9 * * *` 和 `5 9 * * *`、`10 9 * * *` 等分散开。但这个"摊开"操作本身要审计：哪些任务从什么 pattern 改成了什么 pattern？改完有没有生效？有没有残留旧 schedule？

WinMatrix 用一张专门的表记录这件事：

```prisma
// prisma/schema.prisma（第 79-89 行）
model ScheduledCronMigrationLog {
  taskName          String   @map("task_name")
  migrationVersion  String   @map("migration_version")
  // 记录 originalPattern → spreadPattern 的迁移
  @@id([taskName, migrationVersion])   // 复合主键保证幂等
  @@map("scheduled_cron_migration_log")
}
```

两个关键设计：

1. **`@@id([taskName, migrationVersion])` 复合主键**。同一个 (task, version) 只能有一条记录——这意味着迁移脚本跑多少遍，效果一样（幂等）。第一次跑写入记录并执行迁移；重复跑因为主键冲突写不进去，迁移不会重复执行。**用主键约束实现幂等，是最可靠的方式——不靠代码逻辑，靠数据库约束。**
2. **记录 originalPattern → spreadPattern**。事后能审计"这个任务的 cron 是怎么演变的"，出了问题能回溯。

> **repeatable job 自愈**：除了迁移审计，系统每次启动还会清理无效的 BullMQ repeatable job key、保留单一有效 key。BullMQ 的 repeatable job 是靠 key 标识的，cron 改了会生成新 key，老 key 如果不清就会一直触发。启动时强制清理 + 保留单一有效 key，是防止 schedule 漂移的兜底。

---

## 幂等之二：启动期串行补偿，收敛崩溃留下的悬挂状态

这是定时任务系统里最硬核的部分。

考虑这个场景：一个任务（比如 `system-scheduled-run-reconcile`）每 30 分钟跑一次。它在某次执行时，把一个 scheduled_run 标记成 running，正要往下执行，进程崩了。重启后，这条 run 永远卡在 running——没人继续推进它，也没人标记它失败。它成了一个"孤儿"。

如果不管它，越积越多，整个系统的状态机就乱了。

WinMatrix 的解法是**启动时强制补偿**。每次进程启动，在 `reconcileStaleRunsOnBootstrap` 里跑一轮"孤儿回收"：

```typescript
// business/application/services/ScheduledTaskService.ts（第 2244-2298 行）
const thresholdSeconds = Math.max(
  Math.floor(config.scheduled.agentJobTimeoutMs / 1000),
  30 * 60,  // 至少 30 分钟
);
const locked = await withReconcileLock(config.reconcileAdvisoryLockKey, async () => {
  // 三路 reconcile
  const workflowRows = await convergeRunningScheduledWorkflowRuns(500);
  const scheduledRows = await reconcileStaleRunning(thresholdSeconds);
  const pipelineRows = await getAgentRunRepository().reconcileStaleRunning(thresholdSeconds);
  const candidates = await getAgentRunRepository().listPartialScheduledRunCandidates(500);
  // ...
});
```

三个关键设计：

### 三路 reconcile：workflow / scheduled / pipeline

孤儿状态不只出现在一个地方——workflow runs、scheduled runs、pipeline runs 都可能有卡住的 running。所以要**三路并行收敛**：

```
启动时 reconcile：
  ├─ convergeRunningScheduledWorkflowRuns —— workflow 终态收敛
  ├─ reconcileStaleRunning              —— scheduled task run 终态收敛
  └─ agentRunRepository.reconcileStaleRunning —— pipeline run 终态收敛
```

每一路都把"超过阈值还卡在 running 的"强制推进到终态（failed 或 completed）。**每个有状态机的地方，都要有自己的 reconcile。** 不是"一个 reconcile 包打天下"——不同的状态机有不同的终态语义，分开处理才精确。

### thresholdSeconds：30 分钟硬下限

阈值是 `max(agentJobTimeoutMs/1000, 30*60)`——至少 30 分钟。意思是：一个 run 卡在 running 超过 30 分钟，才认定它是孤儿（而不是"正常跑着呢，只是有点慢"）。

30 分钟是硬下限，防止 `agentJobTimeoutMs` 配得太小（比如配成 1 分钟）导致正常执行中的任务被误判为孤儿。**补偿逻辑要有"宁可慢一点也别误杀"的保守倾向——误杀正常任务比漏掉孤儿严重得多。**

### advisoryLock 串行化：多实例启动不撞车

最后，整个 reconcile 套在 `withReconcileLock(config.reconcileAdvisoryLockKey, ...)` 里。

为什么？因为多实例部署时，多个实例可能几乎同时启动（比如发版后一起拉起来）。如果每个实例都跑一轮 reconcile，就会撞车——两个实例同时把同一条 run 标记成 failed，或者一个标记 failed 另一个还在处理。

advisoryLock（PG 的应用级锁）保证**同一时间只有一个实例能跑 reconcile**。其他实例要么等，要么跳过。

> 注：[第 4 章核实报告] 提到 PG advisory lock 在某些路径"已退化为 no-op"，但在**启动期**的 reconcile 这里，它仍然承担串行化职责。**运行期用 Redis 锁，启动期用 PG advisory lock**——这是两种锁的分工。运行期并发高、锁竞争频繁，Redis 更合适；启动期是一次性的、串行的，PG advisory lock 够用且无额外依赖。

**启动期串行补偿的核心思想：进程重启不是"重新开始"，而是"先收拾上次崩溃留下的烂摊子，再开始干活"。** 一个没有补偿逻辑的系统，重启后状态是"脏的"——带着一堆幽灵 running，迟早爆雷。

---

## 结果投递：outbox 模式解耦执行与送达

定时任务执行完了，往往要把结果通知用户（比如企微推送"日报已生成"）。但这步"通知"本身也可能失败——网络抖动、企微 API 限流、token 过期……

如果"执行任务"和"投递结果"耦合在一起（执行完立即推送），推送失败就会拖累整个任务——要么任务标失败（但它其实执行成功了），要么要复杂重试逻辑。

WinMatrix 用 **outbox 模式**解耦。每个任务执行记录自带"投递状态机"：

```prisma
// prisma/schema.prisma（ScheduledTaskRun，第 92-128 行）
model ScheduledTaskRun {
  // 每次执行一条记录
  deliveryStatus  String  // not_requested|pending|delivering|delivered|failed
  deliveryAttempts Int?
  deliveryNextAttemptAt DateTime?
  agentRunId      String?
  traceId         String?
  @@index([deliveryStatus, deliveryNextAttemptAt])
}
```

投递状态机有五态：`not_requested`（不需要投递）→ `pending`（待投递）→ `delivering`（投递中）→ `delivered`（已送达）/ `failed`（投递失败）。

**核心解耦**：任务执行完毕标记终态（completed/failed），和结果投递是**两件事**。执行完了不等于投递成功，投递失败不影响执行已完成的判定。

独立的 sweep timer 每 15 秒扫一次"待投递"的记录：

```
独立 sweep timer（每 15s）：
  扫 deliveryStatus IN (pending, failed需重试) AND deliveryNextAttemptAt <= now
  → 投递（企微/其他渠道）
  → 成功：delivered
  → 失败：deliveryAttempts++，deliveryNextAttemptAt = 指数退避后的时间
```

三个细节：

1. **指数退避重试**（`deliveryNextAttemptAt`）。第一次失败等一会儿再试，越失败等越久——避免疯狂重试压垮推送渠道。
2. **`@@index([deliveryStatus, deliveryNextAttemptAt])`**。这个复合索引让 sweep 扫描高效——直接定位"该投递但还没投递的"，不用全表扫。
3. **执行和投递各有各的状态**。执行状态（completed/failed）和投递状态（delivered/failed）独立，互不污染。

**outbox 模式的本质：把"做一件事"和"通知这件事做完了"分开。** 执行是 SSOT（以它为准），投递是 best-effort（尽力而为）。这样执行的成功不依赖投递渠道的可靠性，系统整体的健壮性大大提升。

---

## 分布式 Leader 锁：谁负责注册 schedule

最后一个问题：16 个任务的 schedule（cron 规则）是怎么注册到 BullMQ 的？

如果每个实例都注册一遍，同一个任务会被注册多次，跑 N 份。所以需要一个 **leader**——只有 leader 实例负责注册 schedule。

```typescript
// infrastructure/scheduled/scheduledSyncLeader.ts（第 10-115 行）
const LOCK_KEY = 'scheduled:sync:leader';
const LOCK_TTL_SECONDS = 120;
const RENEW_INTERVAL_MS = 40_000;

export async function runWithScheduledSyncLeader(syncFn: () => Promise<void>): Promise<boolean> {
  const acquired = await tryAcquireScheduledSyncLeader();
  if (!acquired) return false;   // 没抢到锁，不注册
  startScheduledSyncLeaderRenewal();   // 抢到锁，启动续租
  try { await syncFn(); return true; }
  finally { stopScheduledSyncLeaderRenewal(); await releaseScheduledSyncLeader(); }
}
```

这是 Redis 分布式锁的标准用法（`SET NX EX 120`）。几个细节：

- **TTL 120 秒，续租间隔 40 秒**。续租频率（40s）< TTL/2（60s），保证续租有时间余量——不会出现"续租前锁就过期了"的窗口。
- **续租用 Lua 脚本比对 fence**（hostname:pid:timestamp）。释放锁前先校验"锁还属于我"，防止"我的锁过期了、别人抢到了、我又误把它释放了"的经典漏洞。**Redis 锁的误释放是分布式锁最常见的坑，Lua 比对是标准解法。**
- **没抢到锁就返回 false，不报错**。非 leader 实例不注册 schedule 是正常行为，不是错误。leader 挂了，锁过期，其他实例会抢到。

这个 leader 锁和前面的启动期 advisoryLock 补偿，构成了定时任务系统**分布式协调的两条线**：

- **Leader 锁**（运行期、Redis）：保证同一时间只有一个实例注册 schedule。
- **Advisory lock**（启动期、PG）：保证同一时间只有一个实例跑孤儿补偿。

**运行期协调用 Redis，启动期协调用 PG——各用各的长处。** Redis 快但非持久，适合频繁的 leader 判断；PG advisory lock 轻量且和数据库事务同源，适合一次性的启动期串行化。

---

## defaultJobOptions：队列的默认重试策略

最后补一个 BullMQ 层面的细节——任务的默认重试策略：

```typescript
// infrastructure/queue/queue.ts（第 17-28 行）
const defaultJobOptions = {
  attempts: 2,                    // 最多重试 2 次
  backoff: { type: 'exponential', delay: 5000 },   // 指数退避，起步 5s
  removeOnComplete: { count: 200, age: 7 * 24 * 3600 },  // 完成保留 200 条或 7 天
  removeOnFail: { count: 100, age: 7 * 24 * 3600 },      // 失败保留 100 条或 7 天
};
```

- **attempts: 2**。失败自动重试一次（共 2 次尝试）。不多——因为很多任务失败是逻辑问题，重试没用；2 次主要是防"偶发网络抖动"。
- **指数退避起步 5s**。第一次失败 5s 后重试，不是立即——立即重试往往还是失败（抖动没过）。
- **完成/失败记录按 count + age 双维度保留**。200 条或 7 天，哪个先到就清理。保证近期可查，又不无限堆积。

这些默认值不是随便填的——是"偶发抖动有重试、逻辑错误不浪费、历史记录够排查"的平衡。

---

## 给后来者的总结

1. **队列按重量分三档**：agent（LLM 重）/ system（DB 维护，不可 drain）/ light（轻量扫描）。重任务不能挤占轻任务。
2. **三触发模式统一**：direct（方法）/ message（触发 AI）/ workflow（跑流程）。不是所有定时任务都是"跑函数"。
3. **cron 迁移用复合主键实现幂等**。ScheduledCronMigrationLog 的 `@@id([taskName, migrationVersion])` 让迁移脚本可重复执行，靠数据库约束而非代码逻辑。启动时清理无效 repeatable job key 防 schedule 漂移。
4. **启动期串行补偿三路 reconcile**。workflow / scheduled / pipeline 三个状态机各有孤儿，分别收敛到终态。thresholdSeconds 有 30 分钟硬下限防误杀。
5. **advisoryLock 串行化启动补偿**。多实例同时启动时，只一个跑 reconcile。运行期用 Redis 锁，启动期用 PG advisory lock——各用所长。
6. **结果投递用 outbox 模式解耦**。执行终态和投递状态独立，sweep timer 每 15s 扫待投递，指数退避重试。执行成功不依赖投递渠道可靠。
7. **Leader 锁防 schedule 重复注册**。Redis `SET NX EX` + Lua fence 比对防误释放。TTL 120s，续租 40s（< TTL/2 有余量）。
8. **默认重试 attempts:2 + 指数退避起步 5s**。防偶发抖动但不浪费在逻辑错误上。记录按 count + age 双维度保留。
9. **进程重启不是"重新开始"，是"先收拾烂摊子"**。补偿逻辑是系统自愈能力的核心，没有它的重启是脏的。

定时任务系统是典型的"看不见但致命"的基础设施——用户从不知道它们的存在，但它们一旦坏了，记忆不同步了、孤儿堆积了、提醒不送达了、日志撑爆磁盘了……整个平台慢慢卡死。把它做扎实，靠的不是什么高深技术，而是一层层的幂等和补偿：迁移有审计、崩溃有补偿、投递有 outbox、注册有 leader 锁。每一层都防一类失败，叠起来才是"稳"。

---

> **下一篇**：[《BullMQ 三档队列与后台 Worker 全景》](./17-bullmq-queues.md)（即将发布）——定时任务讲完了，下一篇我们把视角拉到整个后台 Worker 体系：18 个 BullMQ Worker 怎么组织、三档队列怎么配置、开发机的队列怎么防串。
