# 第 23 章 Worker 系统

> "异步 Worker 是高可用系统的隐形引擎。"

当一个用户在对话里点下"发送"，前端拿到的是一次同步的 HTTP 响应；但响应背后真正繁重的工作——记忆同步、跨 Agent 触发、技能轨迹采集、定时巡检——并不在请求线程里发生。这些工作被投递到 BullMQ 队列，由一组后台 Worker 消费。WinMatrix 把这些 Worker 收敛在一个独立进程里（`scheduled-worker.ts`），和 API 进程、RAG 进程物理隔离，互不拖累。

本章从"进程角色守卫"这一最容易被忽视的启动细节讲起，依次展开 Worker 的启动编排、消费者标准模式、生产与隔离运行时的分级启动策略、孤儿任务回收的强一致保障，最后看两个值得细品的工程细节：全局信号量复用和结果投递 Outbox 解耦。

## 23.1 进程角色守卫：动态 import 前的 fail-fast

WinMatrix 有三个生产入口——`api.js`、`scheduled-worker.js`、`rag-worker.js`。每个入口在加载任何业务模块之前，做的第一件事是**校验自己的进程角色**：

```ts
// src/scheduled-worker.ts（第 1-20 行，完整）
#!/usr/bin/env node
import './infrastructure/sandbox/config/undiciConfig.js';
import { assertProcessRole } from '@/startup/processRole.js';
try { assertProcessRole('scheduled'); }
catch (err) {
  const message = err instanceof Error ? err.message : String(err);
  process.stderr.write(`[ProcessRole] scheduled-worker entry startup aborted: ${message}\n`);
  process.exit(1);
}
await import('@/startup/scheduledEntry.js');
```

这段代码非常短，但每一行都有讲究：

1. **第一行先配 undici**：`undiciConfig.js` 在任何网络请求发出前，把全局 HTTP 客户端的默认行为（连接池、超时、代理）固定下来。这是一个典型的"启动期一次性配置"模式。
2. **`assertProcessRole` 是同步的，且在动态 import 之前**：`processRole.ts` 会读取 `WIN_PROCESS_ROLE` 环境变量，和传入的期望值比对。不匹配就抛错。关键在于这个校验发生在 `await import('@/startup/scheduledEntry.js')` **之前**——如果角色不对，那些重量级模块（Prisma、BullMQ、Agent Runtime）根本不会被加载。这是一个 fail-fast 设计：宁可启动时立即退出，也不要带着错误的配置跑起来，再在运行中暴露出各种诡异问题。
3. **错误处理是防御性的**：`try/catch` 捕获异常后，把消息写到 stderr 再 `process.exit(1)`。即使 `assertProcessRole` 内部抛了非 Error 类型，也不会让进程带着未捕获异常继续运行。

这个守卫模式在三个入口里完全一致。它是第 26 章讲的"四进程对齐"（dev 4 终端 ↔ prod docker-compose 4 服务 ↔ k8s 主应用+worker 分离）的代码级落点。**进程角色不是注释，而是代码强制的契约。**

## 23.2 Worker 启动编排：15 个 BullMQ Worker + 4 个补偿 Scanner

scheduled 进程的启动编排集中在 `startup/scheduled.ts`（281 行）。它不是一个简单的"挨个 new Worker"的列表，而是一个分级的、有条件的启动序列。

### 启动序列概览

```ts
// src/startup/scheduled.ts（第 64-96 行，节选）
async function initScheduledWorkers(): Promise<void> {
  assertWorkstationCallbackEndpointConfigured();
  await initAgentStack({ includeMcp: true });
  await ensureBullmqConnectionsReady();
  markBullmqReadyForHealth();
  if (shouldRunScheduledTasks()) {
    const { runWithScheduledSyncLeader } = await import(
      '@/infrastructure/scheduled/scheduledSyncLeader.js'
    );
    const ranSync = await runWithScheduledSyncLeader(
      () => registerDefaultScheduledTasks(),
    );
    startScheduledTaskWorker();
  }
  startMemorySyncWorker();
  startProjectRepositoryWorker();
  startConfluenceKnowledgeSyncWorker();
  // ... 共 15 个 startXxxWorker() 调用
}
```

几个设计决策值得展开：

- **先断言工作站回调端点已配置**：`assertWorkstationCallbackEndpointConfigured()` 在所有 Worker 启动前跑。如果工作站回调没配好，编码任务的回调链路就是断的，启动没有意义。这又是一个 fail-fast。
- **`ensureBullmqConnectionsReady` + `markBullmqReadyForHealth`**：BullMQ 连接就绪前，健康检查端点不能返回 healthy（否则 k8s 会把流量打进来，但 Worker 还没准备好消费）。这两步把"基础设施就绪"和"健康检查放行"解耦。
- **定时任务注册用 leader 锁包裹**：`runWithScheduledSyncLeader(() => registerDefaultScheduledTasks())` 确保多节点环境下只有一个节点真正注册 repeatable job（详见第 24 章）。注册完才启动消费 Worker。
- **`shouldRunScheduledTasks()` 门控**：不是所有进程角色都该跑定时任务。这个函数把"是否注册+消费定时任务"收敛成一个开关。

### 15 个 BullMQ Worker 清单

`startup/scheduled.ts`（第 64-206 行）里实际调用的 15 个 `startXxxWorker()`：

| # | Worker 函数 | 队列 | 职责 |
|---|------------|------|------|
| 1 | `startScheduledTaskWorker` | scheduled-agent / scheduled-system / scheduled-light | 定时任务执行（三队列） |
| 2 | `startMemorySyncWorker` | memory-sync | 会话转录 → 长期记忆同步 |
| 3 | `startProjectRepositoryWorker` | - | 项目仓库索引 |
| 4 | `startConfluenceKnowledgeSyncWorker` | - | Confluence 知识同步 |
| 5 | `startAgentRunProjectionOutboxWorker` | - | Agent 运行投影 Outbox（仅 productionRuntime） |
| 6 | `startCoordinatorGateTimeoutWorker` | - | 协调门超时（仅 productionRuntime） |
| 7 | `startWorkstationTokenUsageCollectorWorker` | - | 工作站 token 用量采集 |
| 8 | `startTraceExtractWorker` | - | 技能轨迹采集（见第 13 章） |
| 9 | `startDistillWorker` | distill | 知识蒸馏（见第 13 章） |
| 10 | `startCrossAgentTriggerWorker` | cross-agent-trigger | 跨 Agent 触发 |
| 11 | `startPersonalTaskWorker` | - | 个人任务 |
| 12 | `startPersonalScheduleWorker` | - | 个人日程 |
| 13 | `startPersonaWechatChannelWorker` | - | 分身企微通道 |
| 14 | `startRoleInboxWorker` | role-inbox | 角色收件箱（见第 11 章） |
| 15 | `startKickoffJobWorker` | kickoff | 项目启动（分布式锁） |

### 4 个补偿 Scanner（非 BullMQ）

除了 BullMQ Worker，还有 4 个**基于 `setInterval` 的补偿 Scanner**——它们不消费队列，而是定时扫描数据库，修复那些"本应该发生但没发生"的状态：

| Scanner 函数 | 扫描动作 | 职责 |
|-------------|---------|------|
| `startKickoffRecoveryScanner` | `scanKickoffOrphanedJobs` | 回收孤儿 Kickoff 任务 |
| `startRoleInboxRecoveryScanner` | `runBootstrapRoleInboxRecoveryScan` | 启动时角色收件箱补偿 |
| `startWorkstationTaskReconcileScanner` | `scanWorkstationTaskReconcileCandidates` | 工作站任务状态对账 |
| `startWorkstationTaskQueueDispatcher` | `dispatchQueuedWorkstationTasks` | 排队工作站任务派发 |

为什么这些用 Scanner 而不是 BullMQ？因为它们的工作模式是"主动巡检数据库里的异常状态"，而不是"消费一个事件"。BullMQ 适合事件驱动，Scanner 适合状态对账。**选择正确的异步原语，本身就是一种设计判断。**

## 23.2.1 Worker 与 API 进程的物理隔离

为什么要花这么大力气把 Worker 拆到独立进程？答案藏在两类工作负载的本质冲突里。

API 进程的职责是**低延迟响应**——每个请求应该在几百毫秒内返回。但记忆同步、知识蒸馏、TFS 导出这些任务动辄几秒到几十秒。如果它们跑在 API 进程里，会占用请求线程（或事件循环时间片），拖慢所有请求的响应。更危险的是，这些任务可能调 LLM、查大表、连外部服务——任何一个卡住，都会让 API 进程看起来"假死"。

物理隔离的好处是双向的：

| 方向 | 不隔离的后果 | 隔离后的保障 |
|------|------------|------------|
| Worker → API | 重型任务抢占请求线程，API 响应变慢 | API 进程只跑轻量请求逻辑 |
| API → Worker | API 进程崩溃（OOM、fatal）连累在途任务 | Worker 进程独立，API 崩溃不影响任务消费 |
| 资源竞争 | LLM 并发额度被请求和任务混用 | 信号量在 Worker 进程内独立治理 |

这种隔离不是"把代码挪个目录"那么简单——它要求共享的 DI 单例（Prisma Client、Redis 连接、BullMQ 连接）在每个进程里独立初始化，但行为一致。这就是为什么 `startup/common.ts` 的启动序列要在每个进程入口都跑一遍（第 26 章的四进程对齐）。

### 单例保护的必要性

```ts
// memorySyncWorker.ts
let worker: Worker<MemorySyncJobData> | null = null;

export function startMemorySyncWorker(): void {
  if (worker) return;   // 已存在则直接返回
  worker = new Worker(...);
}
```

`if (worker) return` 这行看起来朴素，但它解决的是一个真实的 bug 场景。在某些启动路径里（比如 all-in-one 模式 + 热重载），`startXxxWorker` 可能被调用多次。如果不做单例保护：

- 第一次调用创建 Worker A，消费队列。
- 第二次调用创建 Worker B，**也消费同一个队列**。
- 现在 A 和 B 竞争同一个 job——如果 A 拿到了 job 并开始处理，B 也可能拿到（在 BullMQ 的 ack 延迟窗口内），导致**同一个 job 被处理两次**。

单例保护让"启动多次"等价于"启动一次"。这是一个小代码解决大问题的典型例子。

## 23.3 消费者标准模式：以 memorySyncWorker 为例

所有 BullMQ Worker 遵循同一个结构模式。以最简洁的 `memorySyncWorker` 为例：

```ts
// src/interface/workers/memorySyncWorker.ts（第 9-36 行）
export function startMemorySyncWorker(): void {
  if (worker) return;   // 单例保护
  worker = new Worker<MemorySyncJobData>(
    MEMORY_SYNC_QUEUE_NAME,
    async (job) => {
      if (!job) return;
      const data = job.data;
      if (data.type === 'full') { await transcriptSyncManager.syncAll(); return; }
      await transcriptSyncManager.syncDirtyKeys();
    },
    { connection: bullmqWorkerConnection, concurrency: 3 },
  );
}
```

这个 30 行的模式浓缩了四个关键决策：

1. **模块级单例**：`if (worker) return` 防止重复启动。Worker 是进程级资源，启动两次会导致同一个队列被两个消费者竞争，行为不可预测。
2. **共享 Worker 连接**：`bullmqWorkerConnection` 是全局共享的 Redis 连接（见第 5 章），**刻意不带 `commandTimeout`**。原因是 BullMQ Worker 内部用 `BRPOP` 这类阻塞命令阻塞等待任务，如果加了 `commandTimeout`，阻塞命令会被超时打断，触发重连风暴。
3. **`concurrency: 3`**：单个 Worker 进程最多并发处理 3 个 job。这个数字是按 memory-sync 任务的实际负载调过的——记忆同步是 I/O 密集型，并发太高会压垮数据库。
4. **任务内部分支**：`data.type === 'full'` 走全量同步，否则走增量同步（`syncDirtyKeys`）。同一个队列承载两种工作模式，靠 job data 里的 `type` 字段分发。

### 与 API 进程的连接差异

值得对比的是：BullMQ 的 **Queue（生产端）连接**用的是另一组配置：

```ts
// Queue 连接：加 commandTimeout（普通写操作）
export const bullmqQueueConnection = new Redis(config.redisUrl, {
  commandTimeout: 30000,
});

// Worker 连接：不加 commandTimeout（阻塞命令）
export const bullmqWorkerConnection = new Redis(config.redisUrl, {
  // 明确不加 commandTimeout
});
```

同一份代码、同一个 Redis、两组连接配置——区别只在于"是否阻塞"。**这是分布式系统里一个典型的"按用途差异化配置"模式：不要用一把尺子量所有连接。**

### roleInboxWorker：Claim 模式与遥测白名单

`roleInboxWorker`（第 11 章详细讲了收件箱模型）展示了 BullMQ 消费者的另一个重要模式——Claim 认领 + 遥测转发白名单：

```ts
// src/interface/workers/roleInboxWorker.ts（第 1-68 行）
const ROLE_INBOX_WORKER_CONCURRENCY = 4;
const CLAIM_OWNER_PREFIX = 'role-inbox-worker';

/** Interactive 生命周期遥测转发白名单 */
const INTERACTIVE_FORWARD_WHITELIST: ReadonlySet<string> = new Set([
  'span_started', 'span_event', 'span_ended',
  'status', 'run:error',
  'thinking:start', 'thinking:end',
]);

let workerInstance: Worker<RoleInboxJobData, RoleInboxJobResult> | null = null;
```

几个设计点：

- **`CLAIM_OWNER_PREFIX = 'role-inbox-worker'`**：每个 Worker 实例认领任务时，claim_token 带这个前缀 + 实例 ID。这让"哪个 Worker 认领了哪个任务"可追溯——排查"任务卡在 claimed 状态"时，能定位到具体 Worker。
- **`concurrency: 4`**：收件箱任务并发 4。这比 memorySync 的 3 高——收件箱任务通常更轻（转发消息），可以更高并发。
- **遥测转发白名单**：收件箱在处理 interactive 生命周期的消息时，会产生大量遥测事件（span_started / span_event / thinking:start 等）。如果不加筛选全部转发到全局 forwarder，事件噪声会淹没真正重要的信号。白名单只转发关键事件，过滤掉噪声。

### 跨 Agent 触发的 ESM TDZ 重试

`asyncContinuationPrepareRetry` 处理一个微妙的 ESM 问题：

```ts
// src/interface/workers/asyncContinuationPrepareRetry.ts（完整 43 行）
const DEFAULT_RETRY_DELAY_MS = 100;
const ESM_TDZ_ERROR_PATTERN = /\bCannot access\b.*\bbefore initialization\b/i;

export function isRetryableAsyncContinuationPrepareError(error: unknown): boolean {
  return error instanceof ReferenceError
    && ESM_TDZ_ERROR_PATTERN.test(getErrorMsg(error));
}

export async function runAsyncContinuationPrepareWithRetry<T>(
  prepare: () => Promise<T>,
  options: AsyncContinuationPrepareRetryOptions = {},
): Promise<T> {
  const enabled = options.enabled ?? true;
  try {
    return await prepare();
  } catch (error) {
    if (!enabled || !isRetryableAsyncContinuationPrepareError(error)) {
      throw error;
    }
    logger.warn(
      { err: error, ...(options.logContext ?? {}) },
      '[CrossAgentWorker] asyncContinuation prepare hit transient ESM initialization error, retrying once',
    );
    const delayMs = options.delayMs ?? DEFAULT_RETRY_DELAY_MS;
    if (delayMs > 0) {
      await new Promise((resolve) => setTimeout(resolve, delayMs));
    }
    return prepare();   // 重试一次（不 await 的 re-throw 语义由调用方处理）
  }
}
```

**ESM 暂时性死区（Temporal Dead Zone, TDZ）问题**：ESM 模块在完全初始化前，访问其导出会抛 `ReferenceError: Cannot access 'X' before initialization`。这在循环依赖、动态 import、模块加载顺序不确定的场景下可能发生。

这个 Worker 的解法是：**检测到这个特定错误后，延迟 100ms 重试一次。** 100ms 足够让模块完成初始化。为什么只重试一次？因为如果重试一次还不行，说明不是 TDZ 问题，而是真正的模块损坏，继续重试无意义。

这是一个"用最小代价绕过平台特性"的典型工程技巧——ESM 的 TDZ 是 Node.js 的既定行为，无法改变，但可以通过检测 + 延迟重试来吸收它的影响。

## 23.4 生产 vs 隔离运行时：分级启动

不是所有 Worker 都该在所有环境启动。`startup/scheduled.ts` 用 `isolationId === 'prod'`（即 `productionRuntime`）来决定是否启动某些 Worker：

```
                        ┌─────────────────────────────────────┐
                        │   productionRuntime（isolationId    │
                        │              === 'prod'）           │
                        └──────────────┬──────────────────────┘
                                       │
              ┌────────────────────────┼────────────────────────┐
              │                        │                        │
              ▼                        ▼                        ▼
   startAgentRunProjection    startCoordinatorGate     startKickoffRecoveryScanner
       OutboxWorker           TimeoutWorker            startRoleInboxRecoveryScanner
                                                         startWorkstationTaskReconcileScanner
                                                         startWorkstationTaskQueueDispatcher
```

被 `productionRuntime` 门控的 Worker 和 Scanner 有几类：

- **`startAgentRunProjectionOutboxWorker`**：Agent 运行投影 Outbox。这是给生产看板/统计用的，开发环境跑了也没意义。
- **`startCoordinatorGateTimeoutWorker`**：协调门超时。只在多节点生产环境才有协调需求。
- **四个补偿 Scanner**：孤儿任务回收、收件箱补偿、工作站对账、工作站派发。这些 Scanner 的前提是"任务可能被中断"，而开发环境通常没有这种问题。

为什么不在所有环境都启动？两个原因：**资源成本**和**行为可预测性**。Scanner 在开发环境跑起来，可能和你在手动调试的状态打架——你以为某个任务是 `pending`，Scanner 一扫给你改成 `failed` 了。**分级启动让开发环境保持"可预测"，生产环境保持"自愈"。**

`configure_runtime_isolation`（`start.sh`，第 26 章）会在开发环境给所有 BullMQ 队列名加主机名后缀，生产环境强制 `prod`。这和这里的 `productionRuntime` 门控是同一个隔离思想的两面：队列名隔离 + Worker 启动分级。

### TFS 导出 Worker：长任务异步 + 结果回传

`tfsQueryExportWorker` 展示了"长任务异步执行 + 结果回传"的完整模式。TFS（Team Foundation Server）查询可能很慢（复杂 WIQL、大量工作项、长超时），不能在 Agent 请求线程里同步执行。

```ts
// src/interface/workers/tfsQueryExportWorker.ts（第 38-54 行）
async function notifyConversationOnComplete(
  payload: TfsQueryExportJobPayload,
  result: TfsQueryExportJobResult,
  items: WorkItemListEntry[] = [],
): Promise<void> {
  const conversationId = payload.conversationId?.trim();
  const roleId = payload.roleId?.trim();
  if (!conversationId || !roleId) return;

  const triggerMessage = result.success
    ? [
        '[TFS查询后台任务完成]',
        `查询：${payload.queryLabel}`,
        `共 ${result.total} 条工作项${result.truncated ? `（已达上限 ${payload.top}）` : ''}。`,
        ...(items.length > 0 ? ['', '查询结果：', formatWorkItemsSummary(items)] : []),
        '请向用户汇总需求号或工作项列表。',
      ].join('\n')
    : [
        '[TFS查询后台任务失败]',
        `查询：${payload.queryLabel}`,
        `原因：${result.error ?? '未知错误'}`,
      ].join('\n');

  // 通过跨 Agent 调用回传结果
  await crossAgentCallRegistry.register({
    targetConversationId: conversationId,
    context: { sourceConversationId: conversationId, sourceAgentId: roleId },
  });
}
```

这个 Worker 的完整链路：

1. Agent 在对话里需要查 TFS，但查询很慢 → 投递 job 到 `tfs-query-export` 队列。
2. Agent 立即返回"正在查询中"，不阻塞对话。
3. `tfsQueryExportWorker` 在后台执行 WIQL 查询。
4. 查询完成后，通过 `crossAgentCallRegistry` 把结果回传到原对话。
5. 可选：向项目成员发送企微完成通知。

**这个模式的核心是"异步 + 回传"**——把慢操作从请求路径里剥离，结果通过消息机制异步送达。它和第 23.7 节的 Result Delivery Outbox 是两种不同的"结果送达"策略：Outbox 用独立扫描器轮询，crossAgentCall 用事件注册 + 回调。前者适合定时任务（没有明确的"等待方"），后者适合对话场景（有明确的 targetConversationId）。

## 23.5 孤儿任务回收：启动时的强一致保障

Worker 进程崩溃、Pod 重启、节点宕机——这些都会留下"本应完成但停在 `running` 状态"的孤儿任务。如果不处理，它们会永远占用状态，阻塞后续逻辑。WinMatrix 在启动时做一次强一致的回收。

### 两路回收

```ts
// 启动时（startup/scheduled.ts 编排）
// 1. 编码任务超时回收
await failTimedOutRunningTasks();        // 标记超时的编码任务为 failed

// 2. pipeline / scheduled run 终态收敛
await reconcileStaleRunsOnBootstrap();    // 用 advisoryLock 串行化
```

`failTimedOutRunningTasks` 处理编码任务（`coding_task`）层面的超时——那些在编码工作站里跑了一半、超过阈值的任务。

`reconcileStaleRunsOnBootstrap` 处理更高层的 pipeline run 和 scheduled run。它的关键在于用 **advisoryLock 串行化**：

```ts
// src/business/application/services/ScheduledTaskService.ts（第 2244-2298 行）
export async function reconcileStaleRunsOnBootstrap(): Promise<void> {
  const thresholdSeconds = Math.max(
    Math.floor(config.scheduled.agentJobTimeoutMs / 1000),
    30 * 60,   // 至少 30 分钟
  );
  const locked = await withReconcileLock(config.reconcileAdvisoryLockKey, async () => {
    const workflowRows = await convergeRunningScheduledWorkflowRuns(500);
    const scheduledRows = await reconcileStaleRunning(thresholdSeconds);
    const pipelineRows = await getAgentRunRepository().reconcileStaleRunning(thresholdSeconds);
    const candidates = await getAgentRunRepository().listPartialScheduledRunCandidates(500);
    // ... 收敛这些半成品 run 到终态
  });
}
```

三路 reconcile 各管一段：

| reconcile 函数 | 处理对象 | 作用 |
|---------------|---------|------|
| `convergeRunningScheduledWorkflowRuns` | scheduled workflow run | 把卡住的 workflow run 收敛到终态 |
| `reconcileStaleRunning`（scheduled 层） | scheduled_task_run | 超过阈值的 scheduled run 标记失败 |
| `reconcileStaleRunning`（agent run 层） | agent_run | 超过阈值的 agent run 标记失败 |
| `listPartialScheduledRunCandidates` | 部分 scheduled run | 找出"半成品"run 做补偿 |

### 为什么必须串行化

`withReconcileLock` 用的是 PostgreSQL advisory lock（见第 4 章）。为什么这里必须串行化？

想象多节点同时启动的场景：节点 A 和节点 B 同时跑 `reconcileStaleRunsOnBootstrap`，它们会扫到同一批孤儿任务。如果不加锁，两个节点会**同时尝试把同一个 run 标记失败**——轻则重复写入，重则状态机错乱（一个标 failed，另一个标 succeeded）。advisory lock 保证：**同一时刻只有一个节点执行 reconcile，其他节点直接跳过。**

注意 `reconcileAdvisoryLockKey` 是一个固定的 bigint——这意味着锁是"全局唯一"的，不是按 runId 粒度。这是合理的：reconcile 是启动期的一次性动作，不需要细粒度锁。

> 关于分布式锁的选型：第 4 章讲过 PG advisory lock 和 Redis SET NX 的双轨设计。**事实清单明确：分布式锁走 Redis（SET NX EX + Lua），PG advisory lock 已废弃用于通用互斥**——但这里的 `reconcileAdvisoryLockKey` 是个例外，它属于启动期的"一次性、单数据库、强一致"场景，用 advisory lock 比 Redis 更合适（无需引入 Redis 的 TTL/续租复杂度）。这是"工具按场景选"的体现，不是规则违背。

## 23.6 全局信号量复用：crossAgentTriggerWorker 与 scheduled-agent 共享

跨 Agent 触发（cross-agent-trigger）和定时 Agent 任务（scheduled-agent）都会调 LLM，都会消耗昂贵的并发额度。但它们是两个不同的队列、两个不同的 Worker。如何防止它们的并发叠加把 LLM provider 打爆？

WinMatrix 的答案是：**让两个 Worker 共享同一个全局信号量。**

```ts
// crossAgentTriggerWorker 与 scheduled-agent 共享 getScheduledAgentSemFromEnv
// 超过并发额度的 job 不直接失败，而是 moveToDelayed + 抛 DelayedError 重投
```

`getScheduledAgentSemFromEnv` 从环境变量读取并发上限，返回一个进程级的信号量。两个 Worker 在 `acquire` 时竞争同一个信号量——无论哪个队列的 job 先拿到，另一个队列的 job 就得等。这是**跨队列的全局并发治理**。

拿到信号量失败时，Worker 不是让 BullMQ 把 job 标记失败，而是：

```ts
import { DelayedError } from 'bullmq';

// 信号量拿不到时
throw new DelayedError('semaphore full, retry later');
```

`DelayedError` 是 BullMQ 的特殊错误——**job 不会被计为失败，而是被重新放回队列，延迟执行。** 这和普通的重试（`attempts` + 指数退避）不同：DelayedError 不会消耗重试次数，job 会一直被重投直到信号量有空位。这是"背压"的经典实现——当下游（LLM provider）扛不住时，上游主动减速。

这个设计解决了一个真实的运维痛点：**多个异步入口共享同一个昂贵资源（LLM 并发额度）时，不能用"每个队列自己限流"的思路，必须有全局视图。**

## 23.7 结果投递 Outbox：执行终态与投递解耦

定时任务执行完，结果可能要投递到企业微信、企微、Webhook 等多个渠道。如果把"执行"和"投递"耦合在一起——执行成功就同步发企微——会有两个问题：投递失败会拖累执行的事务，投递渠道多了执行逻辑会越来越臃肿。

WinMatrix 用 **Result Delivery Outbox** 解耦：

```
定时任务执行 ──→ 写 ScheduledTaskRun（deliveryStatus=pending）
                        │
                        │  独立 setInterval（每 15s）
                        ▼
              sweepPendingScheduledResultDeliveries(20)
                        │
                        ▼
              按 deliveryTarget 投递（wechat / wecom）
                        │
                        ▼
              更新 deliveryStatus（delivered / failed）
```

`sweepPendingScheduledResultDeliveries(20)` 每 15 秒被独立定时器调用一次，每次扫 20 条 `pending` 状态的投递任务。投递成功标 `delivered`，失败走指数退避（`deliveryAttempts` + `deliveryNextAttemptAt`，见第 24 章的 `ScheduledTaskRun` 模型）。

这是经典的 **Outbox 模式**——把"要做的副作用"先写进一个可靠的存储（这里是 PG 的 `ScheduledTaskRun` 投递字段），再由独立的扫描器异步执行。好处是：

1. **执行事务和投递事务分离**：执行成功即可提交，不必等企微 API 返回。
2. **投递可重试**：企微临时不可用不会丢失投递，扫描器会重试。
3. **投递状态可观测**：`deliveryStatus` 是一个明确的状态机（`not_requested → pending → delivering → delivered | failed`），比"埋在日志里找"清晰得多。

## 23.8 agent_worker_execution：支持重试的执行记录

Worker 执行的细节记录在 `agent_worker_execution` 表里。它的 schema 设计直接支持"重试"语义：

```prisma
// schema.prisma（第 2053-2084 行）
model agent_worker_execution {
  id                  String          @id @default(uuid())
  runId               String          @map("run_id")
  stepId              String          @map("step_id")
  executionKind       String          @map("execution_kind")
  status              String          @default("pending")
  attemptNo           Int             @default(1) @map("attempt_no")
  codingTaskId        String?         @map("coding_task_id")
  spanId              String?         @map("span_id")
  @@unique([runId, stepId, attemptNo], map: "agent_worker_execution_attempt_key")
  @@index([runId, status], map: "idx_agent_worker_execution_run_status")
  @@index([spanId], map: "idx_agent_worker_execution_span")
  @@map("agent_worker_execution")
}
```

关键在 `@@unique([runId, stepId, attemptNo])`——同一个 run 的同一个 step，每次尝试（attemptNo）是一条独立的记录。这意味着：

- 第一次执行：`attemptNo=1`，status 从 `pending → running → succeeded/failed`。
- 如果失败重试：`attemptNo=2`，是一条新记录，不会覆盖第一次的痕迹。
- 历史可追溯：你能看到某个 step 失败了几次、每次失败时的 spanId（关联到第 25 章的 ExecutionSpan）。

`spanId` 字段把 worker 执行记录和可观测性系统打通——一个 worker execution 对应一个或多个 ExecutionSpan，构成完整的执行链路。`idx_agent_worker_execution_span` 索引让"从 span 反查 worker execution"这个查询高效。

## 23.9 Worker 错误处理与重试

### BullMQ 默认重试配置

```ts
// src/infrastructure/queue/queue.ts（第 17-28 行）
const defaultJobOptions = {
  attempts: 2,                                    // 重试次数（含首次）
  backoff: { type: 'exponential' as const, delay: 5000 },  // 指数退避，起步 5s
  removeOnComplete: {
    count: Number(process.env.BULLMQ_SCHEDULED_COMPLETE_COUNT) || 200,
    age: Number(process.env.BULLMQ_SCHEDULED_COMPLETE_AGE) || 7 * 24 * 3600,
  },
  removeOnFail: {
    count: Number(process.env.BULLMQ_SCHEDULED_FAIL_COUNT) || 100,
    age: Number(process.env.BULLMQ_SCHEDULED_FAIL_AGE) || 7 * 24 * 3600,
  },
};
```

几个值得注意的决策：

- **`attempts: 2`**：含首次共两次。不是"重试 2 次"而是"最多尝试 2 次"——这个语义差别在排查"为什么只重试了一次"时要记清楚。
- **`removeOnComplete`/`removeOnFail` 同时按 count 和 age 裁剪**：`{ count: 200, age: 7 天 }` 表示"保留最近 200 条且不超过 7 天"。两个条件是 AND 关系，先到先触发。这避免 Redis 因为堆积太多已完成 job 而 OOM。
- **`backoff.delay: 5000` + `exponential`**：第一次重试等 5s，第二次等 10s，第三次 20s……指数退避是分布式重试的标准做法，避免在下游短暂故障时打出一波"重试风暴"。

### DelayedError：背压重投

前面 23.6 已经讲了 `DelayedError` 的信号量场景。它的语义是"这个 job 现在不能处理，但不是失败，请稍后再试"。和普通重试的区别在于：**DelayedError 不消耗 `attempts` 配额**。一个 job 可以被 DelayedError 几十次，只要 `attempts` 没耗尽，它就还是"活的"。

这让"信号量满"这种暂时性背压，不会被错误地计为"任务失败"。

## 23.9.1 错误处理的分级策略

Worker 的错误处理不是"抓异常记日志"那么简单，而是一个分级策略：

| 错误类型 | 处理方式 | 理由 |
|---------|---------|------|
| **业务错误**（job.data 无效、业务逻辑失败） | 记日志，job 标记失败，走 BullMQ 重试 | 这是"正常的失败"，重试可能成功 |
| **可恢复基础设施错误**（Redis 断连、PG 连接池满） | Proxy 自动重建（Prisma）/ BullMQ 自动重连（Redis） | 瞬时故障，自动恢复 |
| **背压**（信号量满、LLM 限流） | `DelayedError` 重投，不消耗 attempts | 暂时性容量问题，延迟重试 |
| **Worker 级错误**（未捕获异常） | `worker.on('error')` 记日志，Worker 继续运行 | 不能让一个 job 的错误拖垮整个 Worker |
| **致命错误**（无法恢复） | 进程 fatal_exiting，k8s 重启 Pod | 最后手段 |

这个分级策略的关键洞察是：**不同错误有不同的"可恢复性"**。把所有错误都当"重试"处理是浪费（有些错误重试也不会成功）；把所有错误都当"失败"处理是危险（有些错误只是暂时性的）。分级处理让每种错误得到最合适的响应。

### Worker 的 failed 与 error 事件区分

```ts
worker.on('failed', (job, err) => {
  logger.warn(`[XxxWorker] job failed: ${job?.id}, error: ${err.message}`);
});
worker.on('error', (err) => {
  logger.warn(`[XxxWorker] worker error: ${err.message}`);
});
```

这两个事件容易混淆但语义不同：

- **`failed`**：某个 **job** 处理失败（业务函数抛错）。job 被标记失败，但 Worker 继续运行，消费下一个 job。
- **`error`**：**Worker 本身**遇到错误（连接问题、内部 bug）。Worker 可能需要重连或重建。

区分这两个事件的意义在于：一个 job 失败不应该影响其他 job 的消费。如果 job 失败就停 Worker，一个坏 job 就能让整个队列停摆。

## 23.10 Worker 的启动与关闭顺序

### 启动顺序

Worker 的启动不是随意的，`initScheduledWorkers` 里的顺序有依赖关系：

1. **基础设施就绪**：`initAgentStack` → `ensureBullmqConnectionsReady` → `markBullmqReadyForHealth`。
2. **定时任务注册（leader 锁）**：`runWithScheduledSyncLeader(() => registerDefaultScheduledTasks())`。
3. **定时任务消费**：`startScheduledTaskWorker`（注册完才消费，避免消费到还没注册的任务）。
4. **其余 14 个 Worker**：按依赖关系依次启动。
5. **4 个补偿 Scanner**：最后启动，它们依赖前面的 Worker 已经能正常处理回收出来的任务。

### 关闭顺序

关闭时**反序停止**，每个步骤都有超时保护（`safeStep`）：

```
关闭顺序（反序）：
1. stopKickoffRecoveryScanner          — Scanner 先停
2. stopWorkstationTaskReconcileScanner
3. closeKickoffJobWorker               — Worker 开始停
4. closeCrossAgentTriggerWorker
5. stopRoleInboxWorker
6. closeMemorySyncWorker
7. closeScheduledTaskWorker            — 30s 超时，最长
8. stopRagIngestWorker                 — 条件
9. closeTraceExtractWorker + closeDistillWorker
```

反序关闭的每一层都有讲究：

1. **先停 Scanner**：Scanner 不再扫出新的回收任务，避免"Scanner 扫出任务但 Worker 已经停了"的悬空状态。
2. **再停 Worker**：Worker 收到关闭信号后，会停止接新 job，但**等当前在途 job 处理完**才真正关闭。这是优雅关闭的核心——不粗暴中断正在处理的 job。
3. **最后停连接**：`closeScheduledTaskWorker` 超时 30s，是所有关闭步骤里最长的。定时任务可能正在执行一个复杂逻辑，给它足够时间收尾。如果 30s 还没结束，`safeStep` 会强制关闭——不能让一个卡死的 job 阻止整个进程退出。

反序关闭的意义：**让在途的任务有机会被处理完，而不是被粗暴中断。** 如果先关连接，在途的 job 会变成孤儿，下一次启动又得靠 reconcile 回收——多一次折腾。

### safeStep 超时保护

每个关闭步骤都包在 `safeStep` 里，它给每一步设了超时：

```ts
// 概念示意
async function safeStep(name: string, fn: () => Promise<void>, timeoutMs: number): Promise<void> {
  await Promise.race([
    fn(),
    new Promise((_, reject) => setTimeout(() => reject(new Error(`${name} timeout`)), timeoutMs)),
  ]).catch((err) => logger.warn(`[shutdown] ${name}: ${err.message}`));
}
```

`safeStep` 确保即使某一步卡死，关闭流程也能继续。这很重要——关闭流程卡死比启动失败更讨厌，因为进程既不能服务也不能退出，占着资源。

## 23.10.1 健康检查与 BullMQ 就绪门控

回顾启动序列里的这两行：

```ts
await ensureBullmqConnectionsReady();
markBullmqReadyForHealth();
```

这两行是健康检查的关键门控。`ensureBullmqConnectionsReady` 等待 BullMQ 的 Redis 连接真正建立（不是"开始连接"而是"连接已就绪"）。只有连接就绪后，才调 `markBullmqReadyForHealth` 放行健康检查。

为什么这很重要？因为 k8s 的 readiness 探针（第 26 章）会查询健康检查端点。如果 Worker 还没连上 Redis 就被标记 healthy，k8s 会把流量打进来——但此时 Worker 还不能消费任何 job，请求会超时或失败。`markBullmqReadyForHealth` 保证了"被标记 healthy"意味着"真能干活"。

这是"就绪语义"的一个细节——ready 不是"进程启动了"，而是"能服务了"。两者之间的差距，由 `markBullmqReadyForHealth` 这类门控函数填补。

## 本章小结

本章深入分析了 WinMatrix 的 Worker 系统：

1. **进程角色守卫**：三入口（api/scheduled/rag）在动态 import 前用 `assertProcessRole` fail-fast，角色不匹配立即退出，不加载重量级模块。
2. **15 个 BullMQ Worker + 4 个补偿 Scanner**：BullMQ Worker 消费事件驱动任务，Scanner 巡检状态异常——按工作模式选正确的异步原语。
3. **消费者标准模式**：模块级单例 + 共享 Worker 连接（无 commandTimeout）+ 显式 concurrency，以 `memorySyncWorker` 为范例。
4. **生产 vs 隔离运行时分级启动**：`isolationId === 'prod'` 门控 Outbox/CoordinatorGateTimeout/Recovery Scanner，开发环境保持可预测，生产环境保持自愈。
5. **孤儿任务回收的强一致保障**：启动时 `failTimedOutRunningTasks` + `reconcileStaleRunsOnBootstrap`（advisoryLock 串行化三路 reconcile），把崩溃遗留的 running 状态收敛到终态。
6. **全局信号量复用**：`crossAgentTriggerWorker` 与 scheduled-agent 共享 `getScheduledAgentSemFromEnv`，超并发用 `moveToDelayed + DelayedError` 重投，实现跨队列的 LLM 并发治理。
7. **结果投递 Outbox 解耦**：独立 `setInterval` 每 15s 扫 `sweepPendingScheduledResultDeliveries(20)`，执行终态与多渠道投递分离。
8. **`agent_worker_execution` 模型**：`@@unique([runId, stepId, attemptNo])` 支持重试留痕，`spanId` 打通可观测性链路。
9. **重试策略**：`attempts: 2` + 指数退避 + DelayedError 背压重投 + removeOnComplete/Fail 按 count+age 裁剪。

在下一章中，我们将深入定时任务系统——看那 16 个系统级定时任务如何被定义、调度、补偿，以及三队列按重量分流的工程权衡。
