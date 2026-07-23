# 第 23 章 Worker 系统

> "异步 Worker 是高可用系统的隐形引擎。"

WinMatrix 的 8 种 BullMQ Worker 处理所有异步任务——从记忆同步到跨 Agent 触发，从定时任务到 TFS 导出。本章将详细分析这些 Worker 的职责、并发控制和错误处理。

## 23.1 Worker 总览

`src/interface/workers/` 包含 8 个 Worker 文件：

| Worker | 队列 | 并发 | 职责 |
|--------|------|------|------|
| `scheduledTaskWorker` | scheduled-agent/system/light | 可配 | 定时任务执行 |
| `memorySyncWorker` | memory-sync | 3 | 记忆同步 |
| `kickoffJobWorker` | kickoff | 可配 | 项目启动（分布式锁） |
| `crossAgentTriggerWorker` | cross-agent-trigger | 可配 | 跨 Agent 触发 |
| `roleInboxWorker` | role-inbox | 4 | 角色收件箱 |
| `tfsQueryExportWorker` | tfs-query-export | 可配 | TFS 查询导出 |
| `asyncContinuationPrepareRetry` | - | - | 异步续跑重试 |
| `traceExtractWorker` / `distillWorker` | - | - | 技能轨迹蒸馏 |

## 23.2 通用 Worker 模式

所有 Worker 遵循相同的结构模式：

```typescript
// 通用模式
let workerInstance: Worker<JobData, JobResult> | null = null;

export function startXxxWorker(): void {
  if (workerInstance) return;   // 单例保护
  workerInstance = new Worker<JobData, JobResult>(
    QUEUE_NAME,
    async (job) => {
      // 任务处理逻辑
    },
    {
      connection: bullmqWorkerConnection,   // 共享连接（无 commandTimeout）
      concurrency: N,
    },
  );

  workerInstance.on('failed', (job, err) => {
    logger.warn(`[XxxWorker] job failed: ${job?.id}, error: ${err.message}`);
  });

  workerInstance.on('error', (err) => {
    logger.warn(`[XxxWorker] worker error: ${err.message}`);
  });
}

export async function closeXxxWorker(): Promise<void> {
  if (workerInstance) {
    await workerInstance.close();
    workerInstance = null;
  }
}
```

关键设计：

1. **模块级单例**：`workerInstance` 变量 + `if (workerInstance) return`
2. **共享连接**：使用 `bullmqWorkerConnection`（见第 5 章，无 commandTimeout）
3. **错误处理**：`failed` 和 `error` 事件分别处理
4. **优雅关闭**：`closeXxxWorker` 供关闭管线调用

## 23.3 roleInboxWorker：收件箱处理

```typescript
// src/interface/workers/roleInboxWorker.ts（第 1-68 行）
/**
 * Durable role inbox BullMQ worker (OpenSpec §3.2–3.3).
 */
import { Worker } from 'bullmq';
import { agentStateSnapshotRepository } from '@/infrastructure/persistence/repositories/AgentStateSnapshotRepository.js';
import { roleInboxRepository } from '@/infrastructure/persistence/repositories/RoleInboxRepository.js';
import { getRoleInboxWorkerConnection } from '@/interface/channel/channels/scheduled/roleInboxQueue.js';

const ROLE_INBOX_WORKER_CONCURRENCY = 4;
const CLAIM_OWNER_PREFIX = 'role-inbox-worker';

/** Interactive 生命周期遥测转发白名单 */
const INTERACTIVE_FORWARD_WHITELIST: ReadonlySet<string> = new Set([
  'span_started', 'span_event', 'span_ended',
  'status', 'run:error',
  'thinking:start', 'thinking:end',
]);

let workerInstance: Worker<RoleInboxJobData, RoleInboxJobResult> | null = null;

async function findLastPublishedChildEventId(
  conversationId: string,
  parentEventId: string,
): Promise<string | undefined> {
  const events = await roleInboxRepository.findByConversation(conversationId);
  return [...events]
    .reverse()
    .find((event) => getRoleInboxInteractiveMetadata(event.payload).parentEventId === parentEventId)
    ?.eventId;
}
```

### Claim 模式

```typescript
const CLAIM_OWNER_PREFIX = 'role-inbox-worker';
```

收件箱任务使用 Claim 模式——多个 Worker 实例竞争认领，每个任务只被一个 Worker 处理。

### 遥测转发白名单

```typescript
const INTERACTIVE_FORWARD_WHITELIST: ReadonlySet<string> = new Set([...]);
```

只转发关键事件到全局 forwarder，避免事件噪声。

## 23.4 tfsQueryExportWorker：TFS 导出

```typescript
// src/interface/workers/tfsQueryExportWorker.ts（第 1-60 行）
/**
 * TFS 查询后台 Worker（复杂 WIQL / 长超时场景）
 */
import { Worker, type Job } from 'bullmq';
import { bullmqWorkerConnection } from '@/infrastructure/persistence/database/bullmqConnections.js';
import { crossAgentTriggerService } from '@/interface/channel/channels/scheduled/CrossAgentTriggerService.js';
import { crossAgentCallRegistry } from '@/agents/core/runtime/session/CrossAgentCallRegistry.js';
import {
  TFS_QUERY_EXPORT_QUEUE_NAME,
  resolveTfsQueryExportJobTimeoutMs,
  type TfsQueryExportJobPayload,
  type TfsQueryExportJobResult,
} from '@/business-tools/tfs/tfsQueryExportTypes.js';
import { formatWorkItemsSummary, runTfsQueryFetchBackground } from '@/business-tools/tfs/tfsQueryExportCore.js';
import { sendTfsQueryExportSuccessWecomNotify } from '@/business-tools/tfs/tfsQueryExportWecomNotify.js';
```

### 完成后通知

```typescript
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
        ...(wecomTargets.length > 0 && result.total > 0
          ? [`已向项目成员（${wecomTargets.join('、')}）发送企微完成通知。`]
          : []),
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

这个 Worker 展示了"长任务异步执行 + 结果回传"的完整模式：

1. TFS 查询在后台执行（避免阻塞 Agent）
2. 完成后通过 `crossAgentCallRegistry` 回传结果
3. 可选发送企微通知给项目成员

## 23.5 asyncContinuationPrepareRetry：ESM TDZ 重试

这个 Worker 处理一个微妙的 ESM 问题：

```typescript
// src/interface/workers/asyncContinuationPrepareRetry.ts（完整 43 行）
const DEFAULT_RETRY_DELAY_MS = 100;
const ESM_TDZ_ERROR_PATTERN = /\bCannot access\b.*\bbefore initialization\b/i;

export function isRetryableAsyncContinuationPrepareError(error: unknown): boolean {
  return error instanceof ReferenceError && ESM_TDZ_ERROR_PATTERN.test(getErrorMsg(error));
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
    return prepare();   // 重试一次
  }
}
```

### ESM TDZ 问题

ESM 模块的**暂时性死区（Temporal Dead Zone, TDZ）**问题：在模块完全初始化前访问其导出会抛出 `ReferenceError: Cannot access 'X' before initialization`。这在模块热重载或循环依赖场景下可能发生。

解决方案：检测到这个特定错误后，延迟 100ms 重试一次。

## 23.6 Worker 错误处理与重试

### BullMQ 重试配置

```typescript
// src/infrastructure/queue/queue.ts（第 16-27 行）
const defaultJobOptions = {
  attempts: 2,                                    // 重试次数
  backoff: { type: 'exponential', delay: 5000 }, // 指数退避
  removeOnComplete: {
    count: Number(process.env.BULLMQ_SCHEDULED_COMPLETE_COUNT) || 200,  // 保留 200 条完成
    age: Number(process.env.BULLMQ_SCHEDULED_COMPLETE_AGE) || 7 * 24 * 3600,  // 7 天
  },
  removeOnFail: {
    count: Number(process.env.BULLMQ_SCHEDULED_FAIL_COUNT) || 100,      // 保留 100 条失败
    age: Number(process.env.BULLMQ_SCHEDULED_FAIL_AGE) || 7 * 24 * 3600,
  },
};
```

### DelayedError 延迟重试

```typescript
import { DelayedError } from 'bullmq';

// 当条件未满足时，抛出 DelayedError
// BullMQ 会将任务重新入队，延迟执行
throw new DelayedError('content hash mismatch, recent mtime');
```

`DelayedError` 是 BullMQ 的特殊错误——任务不会被标记为失败，而是延迟重试。

## 23.7 Worker 启动与关闭

### 启动顺序

Worker 在 `initAgents()` 阶段启动（见第 2 章）：

```typescript
// src/index.ts
// 1. scheduledTaskWorker（条件启动）
// 2. memorySyncWorker
// 3. traceExtractWorker + distillWorker（技能蒸馏）
// 4. crossAgentTriggerWorker
// 5. roleInboxWorker（条件启动）
// 6. kickoffJobWorker
// 7. workstationTaskReconcileScanner
// 8. ragIngestWorker（条件启动）
// 9. tfsQueryExportWorker
```

### 关闭顺序

关闭时反序停止（见第 2 章）：

```typescript
// src/startup/shutdown/scheduledWorkers.ts
// 1. stopKickoffRecoveryScanner
// 2. stopWorkstationTaskReconcileScanner
// 3. closeKickoffJobWorker
// 4. closeCrossAgentTriggerWorker
// 5. stopRoleInboxWorker
// 6. closeMemorySyncWorker
// 7. closeScheduledTaskWorker（30s 超时，最长）
// 8. stopRagIngestWorker（条件）
// 9. closeTraceExtractWorker + closeDistillWorker
```

每个关闭步骤都有 `safeStep` 超时保护，避免卡死。

## 本章小结

本章深入分析了 WinMatrix 的 Worker 系统：

1. **8 种 Worker**：定时任务、记忆同步、Kickoff、跨 Agent 触发、收件箱、TFS 导出、续跑重试、技能蒸馏
2. **通用模式**：模块级单例 + 共享连接 + 错误处理 + 优雅关闭
3. **roleInboxWorker**：Claim 模式 + 并发 4 + 遥测白名单
4. **tfsQueryExportWorker**：长任务异步 + 跨 Agent 回传 + 企微通知
5. **ESM TDZ 重试**：检测特定 ReferenceError + 延迟 100ms 重试
6. **BullMQ 重试**：attempts=2 + 指数退避 + DelayedError
7. **启动/关闭顺序**：反序关闭，每个步骤有超时保护

在下一章中，我们将深入定时任务系统。
