# cron 尖峰迁移：为什么 09:00 的任务要把它们"散开"

> 这是《WinMatrix 开发经验文集》第 68 篇。讲一个数据库层面的"拥挤事故"：所有人写定时任务都爱写整点，结果整点 DB 连接被打爆。以及我们怎么用一张迁移审计表，幂等地把这堆挤在一起的 cron 打散。代码来自 WinMatrix 后端真实实现，没有杜撰。

如果你做过任何带"定时任务"的系统，大概率见过这个现象：监控里每天上午 9 点准时出现一个 DB 连接数尖峰、CPU 尖峰、慢查询尖峰，持续两三分钟，然后恢复平静。频率固定，时间精准，像闹钟一样。

这不是巧合，是 cron 的"拥挤效应"。

这一篇讲 WinMatrix 踩的这个坑：当几十上百个项目各自配了"每天 9 点跑日报""每周一 9 点跑周报"的任务时，整点会发生什么，以及我们怎么用一张叫 `ScheduledCronMigrationLog` 的审计表，把挤在一起的 cron 幂等地打散到整点附近。

---

## 现象：9 点整的 DB 尖峰

WinMatrix 支持每个项目配置自己的定时任务（ScheduledTaskOverride）。任务的 cron 表达式存在数据库里，默认值很自然——日报就写 `0 9 * * *`（每天 9 点），周报写 `0 9 * * 1`（每周一 9 点），数据清理写 `0 3 * * *`（凌晨 3 点）。

问题出在规模上。当平台上跑着几十上百个项目时，大量任务的 cron 挤在同样的整点：

```
项目A 的日报任务   0 9 * * *    ← 9:00:00
项目B 的日报任务   0 9 * * *    ← 9:00:00
项目C 的周报任务   0 9 * * 1    ← 9:00:00（周一）
项目D 的同步任务   0 9 * * *    ← 9:00:00
... 30+ 个任务全在 9:00:00 触发
```

BullMQ 在 9:00:00 这一秒，把 30 多个 repeatable job 同时出队。这些 job 大多会触发 LLM 调用（生成日报）、DB 写入（写报告、更新任务状态）、ES 索引（记忆同步）。瞬间：

- DB 连接池被打满，其他正常请求报 `ECONNRESET` / 连接超时
- LLM 信号量（`scheduledAgentSem`）被占满，新进来的对话请求被限流
- 慢查询堆积，PG 的 `pg_stat_activity` 里全是 active 查询

用户那边的表现是：每天 9 点前后几分钟，平台"卡一下"。如果不是天天看监控，很难定位到是 cron 拥挤。

更隐蔽的是，这种尖峰还会和第 9 篇讲的"连接池整点回收"叠加——pgBouncer 的 `server_lifetime` 也在整点附近触发，cron 尖峰 + 连接回收，双重打击。

---

## 根因：cron 的表达力有盲区

为什么所有任务都挤在整点？不是大家没意识到，而是 **cron 表达式天然鼓励"整点思维"**。

一个 `0 9 * * *` 写起来直观、读起来清晰、改起来省事。让你把任务挪到 `17 9 * * *`（9:17），你会本能觉得"为什么不干脆 9 点"。人类对"整点"有认知偏好，而 cron 的 5 段式语法（minute hour day month weekday）又恰好让"整点"成为最容易写的值。

但系统不关心你的认知偏好。它只关心"同一时刻有多少 job 出队"。30 个 `0 9 * * *` 就是 30 个并发请求，和 30 个用户同时点按钮没区别。

更深层的问题是：**定时任务的调度是分布式的（每个项目独立配置），但 DB 资源是共享的。** 每个项目配置任务时，只想着自己——"我每天 9 点跑个日报很合理"。没有人会想到"还有 29 个项目也选了 9 点"。这是典型的"局部合理导致全局拥塞"。

而且一旦任务配好了，它就长期不变。配置时没冲突，半年后项目多了，冲突才慢慢积累。等你发现 9 点尖峰时，数据库里已经躺着几十个 `0 9 * * *`，手动改一个个很容易漏、很容易改错。

**所以我们需要的是：一个能在不停服的前提下，把这堆挤在一起的 cron 批量、幂等、可回滚地打散的工具。**

---

## 修复：spread-cron 算法 + 迁移审计表

修复分两部分：一个确定性的"打散算法"，一张保证幂等的"迁移审计表"。

### 打散算法：spreadCronPattern

核心算法在 `src/interface/channel/channels/scheduled/spreadCron.ts`，思路是**用 hash 把每个任务的触发时间在窗口内确定性偏移**：

```typescript
// src/interface/channel/channels/scheduled/spreadCron.ts:67
export function spreadCronPattern(
  pattern: string,
  taskName: string,
  projectId: string | undefined,
  options?: { windowMinutes?: number; skipSystemTasks?: boolean },
): string {
  const parts = parseCronParts(pattern);
  if (!parts) return pattern;

  const useSystemWindow = !(options?.skipSystemTasks === false) && SYSTEM_TASKS_FOR_SPREAD.has(taskName);
  const defaultWindow = useSystemWindow
    ? (Number(process.env.WIN_CRON_SYSTEM_SPREAD_WINDOW_MINUTES) || 15)
    : (Number(process.env.WIN_CRON_SPREAD_WINDOW_MINUTES) || 30);
  const windowMinutes = options?.windowMinutes ?? defaultWindow;
  if (windowMinutes <= 0) return pattern;

  if (!isFixedMinuteField(parts[0]!)) {
    return pattern;   // 非固定分钟（如 */5）不动
  }

  const minute = parseInt(parts[0]!, 10);
  const seed = hashCode(taskName + (projectId ?? ''));
  const offset = seed % windowMinutes;
  parts[0] = String((minute + offset) % 60);
  return parts.join(' ');
}
```

几个关键设计点：

**第一，偏移量是 hash 算出来的，不是随机的。** `hashCode(taskName + projectId) % windowMinutes` 是确定性的——同一个任务，无论迁移跑多少次，算出来的偏移量都一样。这保证了迁移的幂等性：跑一次和跑十次，结果相同。如果是随机偏移，每次重跑都会把 cron 改成不同的值，重复执行就成了灾难。

**第二，窗口大小可配（默认 30 分钟）。** 30 个任务打散到 30 分钟窗口里，平均每分钟 1 个，尖峰就削平了。系统任务（如 `system-memory-tidy`）用更小的 15 分钟窗口（因为它们数量少且都是维护任务）：

```typescript
// src/interface/channel/channels/scheduled/spreadCron.ts:5
const SYSTEM_TASKS_FOR_SPREAD = new Set([
  'system-memory-tidy',
  'system-log-cleanup',
  'system-execution-log-cleanup',
  'system-route-rule-discovery',
  'system-transcript-compact',
  'system-observability-cleanup',
]);
```

**第三，只动"固定分钟"的 cron。** `isFixedMinuteField` 判断 minute 字段是不是一个固定数字（如 `0`、`15`），而不是 `*`、`*/5`、`0,15,30`。`*/5`（每 5 分钟）这类本身就不集中在一个时间点，不需要打散；只有 `0 9 * * *` 这种"精确到某一分钟"的才会被处理。这避免了对已经天然分散的任务做无意义的改动。

**第四，偏移只改 minute，不改 hour。** `(minute + offset) % 60` 可能让 `0 9` 变成 `17 9`（9:17）或 `45 9`（9:45），但不会跨小时。这保证了任务还在"9 点附近"，不破坏用户对"这个任务早上跑"的预期。如果改成 `17 10`（10:17），用户会困惑"我的日报怎么变成 10 点了"。

打散效果可以用 dry-run 预览：

```
迁移前（minute 分布）：  minute=0 出现 30 次   ← 尖峰
迁移后（minute 分布）：  minute=2 出现 1 次
                       minute=5 出现 2 次
                       minute=11 出现 1 次
                       ... 均匀分散到 0-29 分钟
                       peakMinuteBefore=30 → peakMinuteAfter=2
```

### 幂等保证：ScheduledCronMigrationLog

光有算法不够。30 个任务批量改 cron，这是个写操作——如果跑到一半进程挂了，怎么知道哪些改了、哪些没改？如果重跑，会不会把已经打散的再打散一次？

答案是一张审计表。schema 定义（`prisma/schema.prisma:79`）：

```prisma
model ScheduledCronMigrationLog {
  taskName          String @map("task_name")
  migrationVersion  Int    @map("migration_version")
  originalPattern   String @map("original_pattern")
  spreadPattern     String @map("spread_pattern")
  projectId         String? @map("project_id")
  migratedAt        String @map("migrated_at")

  @@id([taskName, migrationVersion])
  @@map("scheduled_cron_migration_log")
}
```

几个设计要点：

**复合主键 `@@id([taskName, migrationVersion])`。** 同一个任务，同一个迁移版本，只能有一条记录。这是幂等的数据库层保障——`create` 会因为主键冲突失败（或者用 upsert），绝不会为同一个任务的同一个版本写两条日志。

**`migrationVersion` 字段。** 迁移是版本化的。v1 只处理"尖峰指纹"（`isPeakCronFingerprint`，即 `0 9 * * *` 这种精确的 9:00）；v2 扩展到"集群检测"（任意同一时:分槽位有 ≥2 个任务就打散）。版本号让两次迁移互不干扰：

```typescript
// src/business/application/services/MigrateScheduledCronService.ts:20
const MIGRATION_VERSION = Number(process.env.WIN_CRON_MIGRATION_VERSION) || 1;
```

想跑 v2，设 `WIN_CRON_MIGRATION_VERSION=2`，它会查 `migrationVersion: 2` 的日志，只处理 v2 范围内还没迁移的任务。

**`originalPattern` 和 `spreadPattern` 都存。** 这是为了可回滚。万一打散后某个任务出了问题（比如某个外部系统要求精确 9:00 触发），可以用 `rollbackScheduledCronMigration` 把它恢复：

```typescript
// MigrateScheduledCronService.ts:243
export async function rollbackScheduledCronMigration(version = MIGRATION_VERSION): Promise<number> {
  const logs = await prisma.scheduledCronMigrationLog.findMany({
    where: { migrationVersion: version },
  });
  let count = 0;
  for (const log of logs) {
    const row = (await getOverridesMap())[log.taskName];
    if (!row) continue;
    await upsertOverride(log.taskName, {
      // ... 把 pattern 改回 log.originalPattern
      pattern: log.originalPattern,
      patternIntent: log.originalPattern,
      // ...
    });
    await refreshRepeatable(log.taskName, log.originalPattern, row.tz);  // 重建 BullMQ repeatable job
    await prisma.scheduledCronMigrationLog.delete({
      where: { taskName_migrationVersion: { taskName: log.taskName, migrationVersion: log.migrationVersion } },
    });
    count += 1;
  }
  return count;
}
```

回滚 = 把 cron 改回 originalPattern + 删掉这条日志。删日志是为了让下次迁移可以重新处理这个任务（否则它会一直被当成"已迁移"跳过）。

### 应用迁移的完整流程

`applyScheduledCronMigrationBatch` 把算法、审计表、BullMQ repeatable job 重建串起来：

```typescript
// MigrateScheduledCronService.ts:199
export async function applyScheduledCronMigrationBatch(
  limit = BATCH_SIZE,  // 默认 50，分批避免一次性改太多
  version = MIGRATION_VERSION,
): Promise<{ applied: number; skipped: number }> {
  const all = await collectCandidatesForVersion(version);
  const pending = all.filter((c) => !c.skipped).slice(0, limit);
  let applied = 0;
  let skipped = all.filter((c) => c.skipped).length;

  for (const item of pending) {
    const row = (await getOverridesMap())[item.taskName];
    if (!row) continue;
    const patternIntent = row.patternIntent?.trim() || item.originalPattern;
    // 1. 改 DB 里的 override（持久化新 cron）
    await upsertOverride(item.taskName, {
      // ...
      pattern: item.spreadPattern,
      patternIntent,
      // ...
    });
    // 2. 重建 BullMQ repeatable job（让新 cron 立即生效）
    await refreshRepeatable(item.taskName, item.spreadPattern, row.tz);
    // 3. 写迁移日志（幂等保证：复合主键防重）
    await prisma.scheduledCronMigrationLog.create({
      data: {
        taskName: item.taskName,
        migrationVersion: version,
        originalPattern: item.originalPattern,
        spreadPattern: item.spreadPattern,
        projectId: item.projectId,
        migratedAt: dateToLocalTimeString(new Date()),
      },
    });
    applied += 1;
    logger.info(`[CronMigrate] applied v${version} taskName=${item.taskName} ${item.originalPattern} -> ${item.spreadPattern}`);
  }
  return { applied, skipped };
}
```

注意三步的顺序：**先改持久化的 override，再重建 BullMQ 的 repeatable job，最后写日志。** 这个顺序很重要——如果先写日志再改 cron，进程挂了就会留下"日志说已迁移但 cron 没改"的脏状态。先改生效、后记日志，即使中途挂了，重跑时 `collectCandidatesForVersion` 会发现"这个任务的实际 cron 还没散开（或者 hash 算出来还是旧的）"，重新处理一次，日志的主键冲突由数据库挡住。

`refreshRepeatable` 这一步也很关键：

```typescript
// MigrateScheduledCronService.ts:186
async function refreshRepeatable(taskName: string, pattern: string, tz: string): Promise<void> {
  const repeatable = await getAllRepeatableJobs();
  const job = repeatable.find((j) => j.name === taskName);
  const queue = getQueueForTask(taskName);
  if (job?.key) {
    await queue.removeRepeatableByKey(job.key);   // 删旧 repeatable
  }
  const jobData: ScheduledJobData = await getJobDataForTask(taskName);
  await queue.add(taskName, jobData, {
    repeat: { pattern, tz: tz || DEFAULT_TZ },    // 加新 repeatable
  });
}
```

光改数据库里的 cron 字段不够——BullMQ 的 repeatable job 是按 job key 注册在 Redis 里的，DB 改了 Redis 不知道。必须先 `removeRepeatableByKey` 删掉旧的 repeatable job，再 `queue.add` 用新 pattern 重新注册。否则会出现"DB 里写的是新 cron，但 BullMQ 还在用旧 cron 触发"的不一致。

### v2：集群检测，不只盯 9:00

v1 只处理 `0 9 * * *` 这种精确的尖峰指纹。但实际中，有些任务挤在 `0 3 * * *`（凌晨 3 点）、`0 10 * * *`（10 点），只要同一时:分槽位有多个任务，就会形成小尖峰。v2 引入了集群检测：

```typescript
// MigrateScheduledCronService.ts:61
async function collectClusterCandidates(version: number, minClusterSize = CLUSTER_MIN_SIZE) {
  const overrides = await getOverridesMap();
  const migrated = await loadMigratedTaskNames(version);
  const slotCounts = new Map<string, number>();

  for (const row of Object.values(overrides)) {
    const key = cronHourMinuteKey(row.pattern);   // "9:0"、"3:0" 这样的槽位 key
    if (!key) continue;
    slotCounts.set(key, (slotCounts.get(key) ?? 0) + 1);   // 统计每个槽位的任务数
  }

  for (const [taskName, row] of Object.entries(overrides)) {
    if (migrated.has(taskName)) { /* 已迁移，跳过 */ continue; }
    const slotKey = cronHourMinuteKey(row.pattern);
    if (!slotKey || (slotCounts.get(slotKey) ?? 0) < minClusterSize) {
      continue;   // 槽位任务数 < 2，不算集群，不处理
    }
    // ... 计算打散 pattern
  }
}
```

`cronHourMinuteKey` 把 `0 9 * * *` 提炼成 `"9:0"`，`CLUSTER_MIN_SIZE` 默认 2——只要同一时:分有 2 个以上任务，就视为集群，纳入打散范围。这比 v1 的"只认 9:00"通用得多，能发现任意整点的拥挤。

---

## 教训

**第一，定时任务的"整点偏好"是个系统级陷阱，要主动对抗。** 每个用户配置任务时都想用整点，这无可厚非。但平台作为资源的共有者，必须意识到"局部最优的全局叠加是灾难"。WinMatrix 的做法不是禁止用户写整点（那太不友好），而是在后台默默把挤在一起的整点打散——用户无感，系统健康。这是平台责任和用户体验的平衡。

**第二，批量改配置必须幂等，幂等必须靠审计表 + 复合主键。** 改 30 个任务的 cron 是个危险操作——中途崩溃、重复执行都可能留下不一致状态。`ScheduledCronMigrationLog` 用 `[taskName, migrationVersion]` 复合主键 + `loadMigratedTaskNames` 查询，把"哪些已迁移"变成了一个可查询、可审计、可回滚的事实。这不是过度设计，是任何批量配置变更的基本功。没有审计表的批量改配置，都是在赌进程不会崩。

**第三，确定性的 hash 打散优于随机打散。** 用 `hashCode(taskName + projectId) % window` 而不是 `Math.random()`，是为了让迁移可重放——跑 10 次结果一样，幂等才有意义。随机打散每次结果不同，重复执行就会反复改 cron（即使复合主键挡住了重复写日志，cron 本身会被反复改）。确定性 hash 是幂等的算法层保障，复合主键是数据库层保障，两者配合才完整。

**第四，改配置后要同步刷新调度引擎的状态。** 光改 DB 里的 cron 字段，BullMQ（或任何调度器）不会自动感知。必须显式地删旧 repeatable job、加新的。这个"DB 与调度引擎的双写一致性"是定时任务系统的经典坑——DB 是配置 SSOT，调度引擎的内存/Redis 状态是执行真源，两者必须同步。漏了刷新这一步，就会出现"配置改了但不生效"的诡异现象。

**第五，迁移要分批、可 dry-run、可回滚。** 看 `applyScheduledCronMigrationBatch` 的设计：`BATCH_SIZE=50` 分批（避免一次性改太多影响线上）、`dryRunScheduledCronMigration` 可预览（先看效果再决定要不要跑）、`rollbackScheduledCronMigration` 可回滚（出问题能恢复）。这三件套是任何"批量线上配置变更"的标配。没有 dry-run 的批量变更，等于盲改；没有回滚的批量变更，等于赌博。

**第六，尖峰检测要做成通用的"集群检测"，而不是只认特定时间点。** v1 只盯 9:00，治标不治本——用户很快会挤到 10:00、3:00。v2 的"任意时:分槽位 ≥2 个任务就打散"才是根治。这背后的思路是：**不要把通用问题特殊化。** "9 点拥挤"是"整点拥挤"的特例，"整点拥挤"是"同槽位聚集"的特例。每往上一层抽象，解法就更通用、更持久。

最后，这个故事其实是个隐喻：**做平台，就是在做"局部合理 vs 全局健康"的仲裁者。** 每个项目配置 9 点任务都很合理，但平台要为"所有人同时 9 点"的后果买单。这种仲裁不能靠"请用户错峰配置"的呼吁（用户不会听），只能靠平台层的技术手段（hash 打散 + 幂等迁移 + 审计表）。平台的价值，恰恰在于替用户承担他们看不见的全局协调成本。

---

> **上一篇**：[《启动序列的坑：DI 注册顺序错了为什么直接崩》](./67-startup-di-order.md)
>
> **下一篇**：[《PromptSection 漂移：提示词模板改了，旧会话怎么办》](./69-prompt-section-drift.md)——讲提示词模板改了之后，正在进行的旧会话用新还是旧 prompt、prompt override 的 60s 缓存怎么失效。
