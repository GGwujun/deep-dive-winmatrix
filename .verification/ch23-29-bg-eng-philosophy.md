# 源码核实报告：后台/工程实践/设计哲学（ch23-29）

源码根：`E:/winning/code/AI/winmatrix/winmatrix-server/`，仓库根：`E:/winning/code/AI/winmatrix/`

## 主题1：Worker 系统（第23章）

### 关键文件
- `src/scheduled-worker.ts`（生产入口，20 行）
- `src/rag-worker.ts`（rag 入口）
- `src/interface/workers/`（18 个 Worker 文件，消费者模式）
- `src/startup/scheduled.ts`（scheduled-worker 全量启动编排，281 行）
- `src/startup/scheduledEntry.ts`
- schema `agent_worker_execution`（L2053-2084）

### 代码片段
scheduled-worker.ts:1-20 — 进程角色 fail-fast 守卫：
```ts
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

startup/scheduled.ts:64-96 — 启动 18 个 Worker：
```ts
async function initScheduledWorkers(): Promise<void> {
  assertWorkstationCallbackEndpointConfigured();
  await initAgentStack({ includeMcp: true });
  await ensureBullmqConnectionsReady();
  markBullmqReadyForHealth();
  if (shouldRunScheduledTasks()) {
    const { runWithScheduledSyncLeader } = await import('@/infrastructure/scheduled/scheduledSyncLeader.js');
    const ranSync = await runWithScheduledSyncLeader(() => registerDefaultScheduledTasks());
    startScheduledTaskWorker();
  }
  startMemorySyncWorker();
  startProjectRepositoryWorker();
  startConfluenceKnowledgeSyncWorker();
  // ... 共 18 个 startXxxWorker() 调用
}
```

memorySyncWorker.ts:9-36 — BullMQ Worker 消费者标准模式（3 并发）：
```ts
export function startMemorySyncWorker(): void {
  if (worker) return;
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

agent_worker_execution（schema.prisma:2053-2084）：
```prisma
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

### 后台任务清单（startup/scheduled.ts:64-206）
必启 BullMQ Worker：
1. startScheduledTaskWorker（3 队列：scheduled-agent / scheduled-system / scheduled-light）
2. startMemorySyncWorker
3. startProjectRepositoryWorker
4. startConfluenceKnowledgeSyncWorker
5. startAgentRunProjectionOutboxWorker（仅 productionRuntime）
6. startCoordinatorGateTimeoutWorker（仅 productionRuntime）
7. startWorkstationTokenUsageCollectorWorker
8. startTraceExtractWorker（技能轨迹采集）
9. startDistillWorker（知识蒸馏）
10. startCrossAgentTriggerWorker
11. startPersonalTaskWorker
12. startPersonalScheduleWorker
13. startPersonaWechatChannelWorker
14. startRoleInboxWorker
15. startKickoffJobWorker

4 个补偿 Scanner（非 BullMQ，定时扫描）：
- startKickoffRecoveryScanner + scanKickoffOrphanedJobs
- startRoleInboxRecoveryScanner + runBootstrapRoleInboxRecoveryScan
- startWorkstationTaskReconcileScanner + scanWorkstationTaskReconcileCandidates
- startWorkstationTaskQueueDispatcher + dispatchQueuedWorkstationTasks

### 设计要点
1. 三进程分离 + 进程角色守卫：api / scheduled / rag，每个入口先调 assertProcessRole 再动态 import。
2. 生产 vs 隔离运行时分级启动：isolationId==='prod' 决定是否启动 Outbox/CoordinatorGateTimeout/Recovery Scanner。
3. 孤儿任务回收是启动时强一致保障：启动时跑 failTimedOutRunningTasks（编码任务）、reconcileStaleRunsOnBootstrap（pipeline/scheduled run 终态收敛），用 advisoryLock 串行化。
4. BullMQ Worker 复用全局信号量：crossAgentTriggerWorker 与 scheduled-agent 共享 getScheduledAgentSemFromEnv，超并发 moveToDelayed + DelayedError 重投。
5. Result Delivery Outbox 解耦：执行终态与结果投递解耦，独立 setInterval 每 15s 扫 sweepPendingScheduledResultDeliveries(20)。

## 主题2：定时任务系统（第24章）

### 关键文件
- schema.prisma:33-128（4 模型）
- `src/infrastructure/scheduled/types.ts`（244 行，全量任务定义 + cron 清单）
- `src/infrastructure/queue/queue.ts`（3 队列定义）
- `src/infrastructure/queue/queueRegistry.ts`（任务名→队列路由表）
- `src/infrastructure/scheduled/scheduledSyncLeader.ts`（Redis leader lock，116 行）
- `src/business/application/services/ScheduledTaskService.ts`（3760 行，注册/调度/补偿 SSOT）
- `src/interface/workers/scheduledTaskWorker.ts`（1645 行，BullMQ 消费者）

### 模型（schema.prisma:33-128）
- **ScheduledTaskOverride**（33-56）：taskName @id、pattern 默认 `0 0 * * *`、scheduleType 默认 cron、status(active|stopped)、resultDeliveryTarget(none|wechat|wecom)
- **ScheduledTaskWorkflowBinding**（59-76）：任务绑定 workflow（templateId + 可选 templateVersionId），级联删除
- **ScheduledCronMigrationLog**（79-89）：09:00 尖峰 cron 存量迁移审计，复合主键 `@@id([taskName, migrationVersion])`
- **ScheduledTaskRun**（92-128）：每次执行一条，含 deliveryStatus(not_requested|pending|delivering|delivered|failed) 投递状态机、deliveryAttempts/deliveryNextAttemptAt（指数退避）、agentRunId、traceId；索引 `[deliveryStatus, deliveryNextAttemptAt]`

### 代码片段
系统维护任务预定义清单（`infrastructure/scheduled/types.ts:89-243`，节选）：
```ts
export const SYSTEM_SCHEDULED_TASKS: ScheduledTaskConfig[] = [
  { name: 'system-memory-tidy', pattern: '0 3 * * *', tz: DEFAULT_TZ, message: '会话转录 → 长期记忆全量同步', triggerType: 'direct' },
  { name: 'system-log-cleanup', pattern: '0 4 * * *', triggerType: 'direct', ... },
  { name: 'system-coding-task-timeout-sweep', pattern: '*/5 * * * *', triggerType: 'direct', ... },
  { name: 'system-reminder-delivery', pattern: '*/1 * * * *', ... },
  { name: 'system-execution-log-cleanup', pattern: '30 3 * * *', message: '...agent_execution_log 已退役（retire-agent-execution-log）', ... },
  { name: 'system-llm-call-watchdog-sweeper', scheduleType: 'interval', intervalMs: 600_000, ... },
  { name: 'system-scheduled-run-reconcile', pattern: '*/30 * * * *', ... },
  { name: 'system-observability-cleanup', pattern: '30 4 * * *', message: '清理可观测数据...execution_span(级联 execution_span_event) / pipeline_run / session_transcript', ... },
  // ... 共 16 个 SYSTEM_SCHEDULED_TASKS
];
```

3 队列 + 任务名路由（`queue.ts:30-53`）：
```ts
const scheduledAgentQueue = new Queue<ScheduledJobData>(resolveBullmqQueueName('scheduled-agent'), {...});
const scheduledSystemQueue = new Queue<ScheduledJobData>(resolveBullmqQueueName('scheduled-system'), {...});
const scheduledLightQueue = new Queue<ScheduledJobData>(resolveBullmqQueueName('scheduled-light'), {...});
export function getQueueForTask(taskName: string): Queue<ScheduledJobData> {
  if (SYSTEM_TASKS.has(taskName)) return scheduledSystemQueue;
  if (LIGHT_TASKS.has(taskName)) return scheduledLightQueue;
  return scheduledAgentQueue;
}
```

defaultJobOptions（queue.ts:17-28）：
```ts
const defaultJobOptions = {
  attempts: 2,
  backoff: { type: 'exponential' as const, delay: 5000 },
  removeOnComplete: { count: 200, age: 7 * 24 * 3600 },
  removeOnFail: { count: 100, age: 7 * 24 * 3600 },
};
```

### 幂等与补偿
scheduledSyncLeader.ts:10-115 — Redis 分布式锁：
```ts
const LOCK_KEY = 'scheduled:sync:leader';
const LOCK_TTL_SECONDS = 120;
const RENEW_INTERVAL_MS = 40_000;
// SET NX EX 120 抢锁；renew 用 Lua 比对 fence（hostname:pid:timestamp）续租
export async function runWithScheduledSyncLeader(syncFn: () => Promise<void>): Promise<boolean> {
  const acquired = await tryAcquireScheduledSyncLeader();
  if (!acquired) return false;
  startScheduledSyncLeaderRenewal();
  try { await syncFn(); return true; }
  finally { stopScheduledSyncLeaderRenewal(); await releaseScheduledSyncLeader(); }
}
```

reconcileStaleRunsOnBootstrap（ScheduledTaskService.ts:2244-2298）— advisoryLock 串行化三路 reconcile：
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

### 设计要点
1. 三队列按重量分流：scheduled-agent（LLM 重，concurrency=config.scheduled.llmConcurrency）、scheduled-system（DB/ES 维护，不可 drain）、scheduled-light（轻量扫描，*/5 * * * *）。
2. 三触发模式：direct（直接调方法）、message（发消息触发智能体）、workflow（绑定 flow_template）。
3. 重复 repeatable job 自愈：每次启动清理无效 key、保留单一有效 key。
4. 09:00 尖峰 cron 迁移审计：ScheduledCronMigrationLog 记录 originalPattern→spreadPattern，复合主键保证幂等。
5. Result Delivery 与执行终态解耦：ScheduledTaskRun 自带 outbox 字段，独立 sweep timer 每 15s 扫一次。

## 主题3：可观测性（第25章）

### 关键文件
- schema.prisma:2990-3094（ExecutionSpan / ExecutionSpanEvent / UnifiedObservabilityRule）
- `src/infrastructure/observability/`（41 个文件）
- `src/infrastructure/observability/hub/`（14 个文件，ObservabilityHub 58505 字节）
- `src/infrastructure/observability/llmObservability.ts`（LLM 调用遥测入口）
- `src/infrastructure/observability/performanceMonitor.ts`（PerformanceMonitor）
- `openspec/contracts/llm-call-span-telemetry-contract.md`（稳定契约）
- `src/infrastructure/scheduled/llmCallWatchdogSweeper.ts`（悬挂 LLM 调用补偿）

### 模型（schema.prisma:2990-3094）
ExecutionSpan（2990-3038）— retire-agent-execution-log 后的 SSOT：
```prisma
model ExecutionSpan {
  spanId              String    @id @map("span_id")
  traceId             String    @map("trace_id")
  parentSpanId        String?   @map("parent_span_id")
  spanKind            String    @map("span_kind")
  status              String    @default("pending")
  outcome             String?
  tokenInput          Int?      @map("token_input")
  tokenOutput         Int?
  tokenThinking       Int?
  model               String?
  provider            String?
  failureReasonCode   String?   @map("failure_reason_code")
  agentRunId          String?   @map("agent_run_id")
  attributes          Json?
  events              Json?     @default("[]")
  @@index([traceId], map: "idx_span_trace")
  @@index([spanKind, status], map: "idx_span_kind_status")
  @@index([attributes], map: "idx_span_attrs", type: Gin)
  @@map("execution_span")
}
```

ExecutionSpanEvent（3041-3060）— span 内原子事件规范化子表（与 execution_span.events JSON 双写，子表为完整事件源）：
```prisma
model ExecutionSpanEvent {
  id        BigInt   @id @default(autoincrement())
  spanId    String   @map("span_id")
  eventType String   @map("event_type")
  phase     String?
  ts        DateTime @db.Timestamptz(6)
  attrs     Json     @default("{}")
  seq       Int
  @@unique([spanId, seq], map: "uq_span_event_span_seq")
  @@index([spanId, ts], map: "idx_span_event_span_ts")
  @@index([traceId, eventType], map: "idx_span_event_trace_type")
  @@map("execution_span_event")
}
```

UnifiedObservabilityRule（3063-3094）— 统一遥测路由规则：
```prisma
model UnifiedObservabilityRule {
  eventType        String   @map("event_type")
  runtimeActionKey String   @default("") @map("runtime_action_key")
  sinkSpan         Boolean  @default(true) @map("sink_span")
  sinkLog          Boolean  @default(false) @map("sink_log")
  sinkChannels     String[] @default([]) @map("sink_channels")
  opensSpan        Boolean  @default(true) @map("opens_span")
  opensInvocation  Boolean  @default(false) @map("opens_invocation")
  redactionPolicy  Json     @default("{}") @map("redaction_policy")
  @@id([eventType, runtimeActionKey])
  @@map("unified_observability_rule")
}
```

### LLM Span 遥测契约（contract.md:13-35）
| 时机 | eventType | 必填 attrs |
|------|-----------|------------|
| 调用前 | llm_call_start | llmInvocationId、actionName、request（messages[]、可选 tools） |
| 调用后 | llm_call_end | response（模型文本或结构化 JSON）、inputTokens/outputTokens |
| 失败 | llm_call_error/llm_call_failed | errorMessage，保留已发出的 request |

### 代码片段
emitLLMCallStart（llmObservability.ts:89-128）— 完整 request 暂存（防 Hub compact 剥离）：
```ts
export async function emitLLMCallStart(
  sessionContext, runId, client, messages, actionName?, requestTools?, runtimeInvocation?,
): Promise<void> {
  // 提案2 D1：在 hub compactLlmEventDataForLogSink 剥离 request 之前，于源点暂存完整 request
  if (obsFields.llmInvocationId) {
    import('./elasticsearchLlmLogger.js')
      .then(({ storeFullRequestForRun }) => storeFullRequestForRun(invocationId, request))
      .catch(() => {});
  }
  await recordViaHubOrLegacy({ ..., eventType: 'llm_call_start', request, ... });
}
```

ObservabilityHub 三 Sink（hub/ObservabilityHub.ts:340-412）：
```ts
export class ObservabilityHub {
  readonly channelSink: ChannelSink;
  readonly spanSink: SpanSink;
  readonly logSink: LogSink;
  private readonly ruleStore: UnifiedObservabilityRuleStore;
  private readonly activeSpanRegistry = new HubActiveSpanRegistry();
  constructor(options) {
    this.channelSink = new ChannelSink({...});
    this.logSink = options.logSink ?? new LogSink();
    this.spanSink = new SpanSink({..., channelSink: this.channelSink});
  }
}
```

llmCallWatchdogSweeper.ts:1-53 — 悬挂 llm_call_start 补偿：
```ts
export const LLM_CALL_WATCHDOG_SWEEPER_TASK_NAME = 'system-llm-call-watchdog-sweeper';
function resolveSweepThresholdMs(): number {
  const hardMs = getConfig().llmCallHardTimeoutMs;
  return hardMs > 0 ? hardMs * 2 : 360_000;  // 2x hard timeout 或 6 分钟
}
```

### 设计要点
1. ExecutionSpan 是 retire-agent-execution-log 后的 SSOT：agent_execution_log 已无 model 定义。
2. Span Events 双写：execution_span.events JSON 与 execution_span_event 子表双写，子表有 [spanId, seq] 唯一约束防重。
3. 三 Sink 路由：UnifiedObservabilityRule 决定每个 eventType 落 span/log/channel 组合，ObservabilityHub 在 record() 时查规则分流。
4. LLM 调用强制 I/O 可读：契约要求 llm_call_start 必含 request.messages，llm_call_end 必含 response。
5. 悬挂 LLM 调用级联补偿：llmCallWatchdogSweeper（每 10 分钟）补写 llm_call_end 并级联 finalize agent_run/scheduled_task_run。
6. GIN 索引支持 JSON 属性查询：execution_span.attributes 上 `@@index(..., type: Gin)`。

## 主题4：构建与部署（第26章）

### 关键文件
- `Makefile`（29781 字节，44 个 target）
- `winmatrix-server/package.json`（14057 字节，scripts 在 7-117 行）
- `docker/Dockerfile`（10753 字节，三阶段）
- `docker/docker-compose.yml`（15269 字节，7 服务）
- `k8s/deployment.yaml`、`configmap.yaml`、`secret.yaml.example`、`ops.sh`
- `scripts/deploy/start.sh`（43698 字节，1240 行）

### Makefile 主要 target（44 个，节选）
1. 镜像构建推送：build-push-winmatrix、build-push-sandbox-api、build-push-base-build、build-push-base-runtime、build-push-runtime-base
2. 工作站镜像：build-push-coding-workstation、build-push-sre-workstation、build-push-openclaw-workstation
3. Engine 镜像（4 种）：build-engine-claude-code、build-engine-codex、build-engine-hermes、build-engine-openclaw、build-all-engines、build-push-all-engines
4. 冒烟测试：smoke-test、smoke-test-base-build/runtime、smoke-test-sandbox、test-runtime-base、test-engine-*
5. 离线包准备：prepare-coding-workstation-offline、prepare-openclaw-workstation-offline
6. login、push-dev-latest、build-protocol、help

### package.json scripts
构建链（4 种）：
- `build`(9)：prisma generate + check-no-js-in-src + build-esbuild + tsc-alias-build + fix-remaining-alias-in-dist + verify-no-alias-in-dist
- `build:tsc`(14)：类型检查（--noEmit）
- `build:prod`(13)：生产构建

生产启动四进程（package.json:26-31）：
```json
"start:prod": "WIN_PROCESS_ROLE=api node dist/api.js",
"start:prod:monolith": "node dist/index.js",
"start:prod:api": "WIN_PROCESS_ROLE=api node dist/api.js",
"start:prod:scheduled": "WIN_PROCESS_ROLE=scheduled node dist/scheduled-worker.js",
"start:prod:rag": "WIN_PROCESS_ROLE=rag node dist/rag-worker.js",
```

### docker-compose.yml 服务结构（7 服务）
1. winmatrix（主应用，WIN_PROCESS_ROLE=api，端口 3000→8080）
2. winmatrix-embedding（端口 8401）
3. winmatrix-scheduled-worker（WIN_PROCESS_ROLE=scheduled，WORKER_HEALTH_PORT=8402）
4. winmatrix-rag-worker（WIN_PROCESS_ROLE=rag）
5. pgbouncer（transaction-pooling，5000 人规模避免 too many clients）
6. shared-net、volumes

scheduled-worker 与 rag-worker 都 depends_on: winmatrix-embedding: condition: service_healthy。

### Dockerfile 三阶段
1. webui-builder（前端构建 winmatrix-ui，支持 NPM_USE_MIRROR 切 npmmirror）
2. 后端构建（winmatrix-node-build-base 基础镜像）
3. 生产运行（winmatrix-node-runtime-base，含 kdocs-cli）
构建参数：BUILD_BASE_IMAGE / RUNTIME_BASE_IMAGE（公司 Harbor）、NPM_REGISTRY_URL（默认内网 Nexus）。

### k8s/deployment.yaml
- replicas: 1，imagePullSecrets: winex-wxp-copilot
- liveness /health（initialDelay 90s），readiness /readyz（initialDelay 60s）—— readiness 摘流不重启，fatal 靠应用自退出
- resources: requests 512Mi/250m，limits 2Gi/2000m
- 显式覆盖 NODE_ENV=production + DATABASE_URL + REDIS_URL（避免 Secret 中 NODE_ENV=test 污染）

### start.sh（1240 行）
main() 命令路由：start / start:watch / start:prod / stop / restart / status / logs
- configure_runtime_isolation：以主机名 WIN_RUNTIME_ISOLATION_ID 隔离所有 BullMQ 队列
- free_port + cleanup_winmatrix_orphans：跨平台（pgrep/Windows taskkill/PowerShell）端口与孤儿进程清理
- wait_for_http：健康检查超时重试（后端 90 次，UI 120 次）
- prod 模式警告："仅启动 API；scheduled/rag/embedding 请使用 docker compose"

### 设计要点
1. 四进程对齐：dev（4 终端）↔ prod（docker-compose 4 服务）↔ k8s（主应用 + scheduled worker 分离），每个入口内联 WIN_PROCESS_ROLE 守卫。
2. 构建链以 esbuild 为主、tsc 校类型。
3. Makefile 覆盖全镜像矩阵：主应用 + sandbox-api + 3 workstation + 4 engine + 2 base。
4. start.sh 跨平台兼容：同时支持 Linux（pgrep/kill 树）与 Windows（taskkill //T //F、PowerShell）。
5. dev all-in-one 默认开启可观测性 JSONL：START_OBSERVABILITY_LOG=true → JSONL 落盘。

## 主题5：测试策略（第27章）

### 关键文件
- `winmatrix-server/vitest.config.ts`（4 project：unit/integration/migration-exec/e2e）
- `winmatrix-server/tests/README.md`（107 行，E2E 覆盖矩阵）
- `tests/e2e/setup.ts`（必填项校验）
- `tests/e2e/testApp.ts`（建库建表 + 完整应用启动）
- `.env.test.example`

### 目录结构
```
tests/
├── unit/            # 单元测试，按被测源码层归类
├── integration/     # 跨模块链路
├── e2e/             # API E2E（tests/e2e/api/ 24 个测试文件）
├── shared/          # 共享 setup / fixtures / helpers / factories
├── fixtures/        # 测试数据与回放样例（含 decision-replay / incident-2026-05-26-job-* 真实事故回放）
├── manual/          # 人工验证、专项诊断脚本
└── characterization/# 特征测试
```

### vitest.config.ts 四 project
unit project（44-60）：
```ts
{ test: {
  name: 'unit',
  include: ['tests/unit/**/*.test.ts', 'tests/characterization/**/*.test.ts'],
  setupFiles: ['./tests/unit/setup.ts'],
  env: { DATABASE_URL: 'postgresql://user:pass@localhost:5432/winmatrix_unit_test', SANDBOX_API_AUTH_TOKEN: 'unit-test-sandbox-control-token' },
  isolate: true, threads: true, testTimeout: 10000, hookTimeout: 10000,
}}
```
e2e project（89-100）：
```ts
{ test: {
  name: 'e2e',
  include: ['tests/e2e/**/*.{test,spec}.ts'],
  setupFiles: ['./tests/shared/setup.ts', './tests/e2e/setup.ts'],
  threads: false, testTimeout: 60000, hookTimeout: 300000, retry: 1,
}}
```

### E2E 不 mock 校验（tests/e2e/setup.ts:6-35）
```ts
process.env.NODE_ENV = 'test';
const required = ['DATABASE_URL', 'REDIS_URL'] as const;
const missing = required.filter((key) => !process.env[key] || process.env[key]!.trim() === '');
if (missing.length > 0) { console.error('[E2E] 缺少必填配置:', missing.join(', ')); process.exit(1); }
// 还校验 LLM Provider
```

testApp.ts 建库（33-60）：
```ts
async function ensureTestDatabase(): Promise<void> {
  const parsed = new URL(process.env.DATABASE_URL);
  const dbName = (parsed.pathname || '/').slice(1).replace(/\/$/, '') || 'winmatrix_test';
  parsed.pathname = '/postgres';
  const client = new Client({ connectionString: parsed.toString() });
  await client.connect();
  const res = await client.query('SELECT 1 FROM pg_database WHERE datname = $1', [dbName]);
  if (res.rows.length === 0) {
    await client.query(`CREATE DATABASE "${dbName.replace(/"/g, '""')}"`);
  }
  await client.end();
}
```

### E2E 测试模式（tests/e2e/api/p0-smoke.test.ts:11-58）
```ts
describe('P0 API smoke', () => {
  beforeAll(async () => {
    const app = await getTestApp();  // 共享单例 app
    baseUrl = app.baseUrl;
    apiClient = new ApiClient(baseUrl);
    userCtx = await registerAndLogin(baseUrl);
    adminCtx = await registerAndLogin(baseUrl, { isAdmin: true });
  });
  afterAll(async () => { await closeTestApp(); });
  it('有效 input 返回 200 与 unifiedDecision.executionPlan schema', async () => {
    const res = await apiClient.anonymous().post('/api/v1/agents/route', { body: { input: '你好', userId: userCtx.userId } });
    expect([200, 404, 503].includes(res.status)).toBe(true);
  });
});
```

### 设计要点
1. E2E vs 单元测试的 mock 边界明确：单元测试不加载 .env.test，DATABASE_URL 用占位串；E2E 加载 .env.test，不 mock DB/Redis/LLM，缺失必填项 process.exit(1)。
2. E2E 共享单例 app：getTestApp() 在 beforeAll 启动一次、closeTestApp() 在 afterAll 关闭。
3. E2E 自动建库：ensureTestDatabase 连 postgres 库查 pg_database，不存在则 CREATE DATABASE，再 prisma db push 建表。
4. 四 project 分级 timeout：unit 10s / integration 30s（hook 60s）/ e2e 60s（hook 300s，retry 1）。e2e threads:false 串行。
5. 专项回归开关：kickoff 默认 describe.skip，需 E2E_KICKOFF=1。组合命令 test:quick / test:verify / test:all。
6. 真实事故回放 fixture：tests/fixtures/incident-2026-05-26-job-84711 / 84712 / decision-planner-91695-replay 是生产事故回放样例。

## 主题6：设计哲学/模式（第28-29章）

### 关键文件
- `AGENTS.md`（31007 字节，根目录，架构 SSOT）
- `CLAUDE.md`（6620 字节）
- `winmatrix-server/CLAUDE.md`（4100 字节）
- `winmatrix-server/scripts/check-agent-layer-imports.cjs`（六层 import 门禁）
- `winmatrix-server/scripts/check-layer-imports.cjs`（大层门禁）
- `src/startup/common.ts`（27660 字节，启动序列）
- `src/startup/wireAgentBusinessPorts.ts`（Port 注入，347 行）

### 六层分层核实
**仓库大层（严格 5 层）**：
```
src/
├── interface/        # API/Channel/Middleware（最外层）
├── agents/           # Agent Runtime（六子层）
├── business-tools/   # 业务工具（agents 唯一可调的业务门面）
├── business/         # 领域服务
├── infrastructure/   # 基础设施（最内层）
├── integration/      # 横切连接器
├── startup/          # 启动引导（横切，DI 注册 + Port 注入）
├── shared/ types/ utils/ config/ cli/  # 横切共享
```

**Agent Runtime 六子层**（src/agents/core/）：
| 层 | 路径 | 职责 |
|----|------|------|
| AI Kernel | agents/core/ai-kernel/ | 六域内核原语 + Port 契约 |
| AI Execution | agents/core/ai-execution/ | cutover 编排、Port 适配器 |
| Worker | agents/core/worker/ | Role / Action / 数字员工运行时 |
| Agent | agents/core/agent/ | 决策引擎 + Turn 编排 + 执行模式 |
| Kernel Management | agents/core/kernel-management/ | 配置治理 + DB 热更新 |
| Runtime | agents/core/runtime/ | 会话转录 + 执行挂起 |

### 依赖规则详表（AGENTS.md:255-268）
| 允许 | 禁止 |
|------|------|
| ✅ agents → business-tools | ❌ agents → business |
| ✅ agents → infrastructure | ❌ business → agents |
| ✅ business-tools → business | ❌ business-tools → agents |
| ✅ business → infrastructure | ❌ infrastructure → agents |
| ✅ interface → 任意下层 | ❌ L3 harness → L4 domain-harness |
| ✅ ai-execution → kernel + agents | ❌ ai-kernel → agents/core/worker |
| ✅ ai-kernel → infrastructure | ❌ ai-kernel → business/ |

### import 边界门禁（check-agent-layer-imports.cjs:15-80）
```js
const L3_L4_ISOLATION_PREFIXES = [
  { from: 'agents/harness/', forbidden: '@/agents/domain-harness/' },
  { from: 'agents/domain-harness/', forbidden: '@/agents/harness/' },
];
const CHANNEL_INTEGRATION_ISOLATION = [
  { from: 'interface/channel/', forbidden: '@/integration/connectors/' },
  { from: 'integration/connectors/', forbidden: '@/interface/channel/' },
];
const INFRA_FORBIDDEN_PRODUCT_DIRS = [
  'infrastructure/dashboard', 'infrastructure/identity', 'infrastructure/session',
  'infrastructure/codingTask', 'infrastructure/kickoff', 'infrastructure/sandbox/workstation',
];
```
`npm run check:agent-layers:strict`（package.json:72）= check-agent-layer-imports.cjs --strict && npm run check:tool-kernel-consumption。

### startup/common.ts Port 注入序列（242-285）
阶段 1（initInfrastructure）：
1. registerServices()（DI 容器注册）
2. registerAgentOwnedSingletons()
3. registerWorkstationServices()
4. assertCoreDiRegistrations()（断言核心 DI 完成）
5. side-effect import：'@/infrastructure/integration/sideEffect/registerAdapters.js'（4 个 adapter，adapterCount !== 4 抛错）
6. llmFactory.initProviderRegistry()（仅 api/scheduled/all-in-one，rag 跳过）
7. connectPrisma()
8. warnIfTimezoneInconsistent()（Node TZ vs PG session 时区校验）
9. ensureSystemUsers()

阶段 2（initConfigAndCache）：
- entityCache.initialize() + cacheInvalidationBus.initialize()
- configManager.initialize()（带 60s 超时）
- bootstrapPromptRegistry()（rag 跳过）
- ConfigDbListener（监听 pg_notify('config_change')，rag/STARTUP_SKIP_CONFIG_DB_LISTENER 跳过）

Port 注入（wireAgentBusinessPorts.ts:123-214）— 17+ 业务 Port setter：
```ts
export function wireAgentBusinessPorts(): void {
  setConversationServicePort(getConversationService());
  setConversationMessageAppendPort(getConversationMessageAppendService());
  setRoleInboxPort(roleInboxEnqueueService);
  setIdentityPort({...});
  setDigitalEmployeeBootstrapPort({...});
  setProjectContextPort({...});
  setSkillManifestPort({...});
  setSkillCredentialResolutionPort({...});
  setCodingTaskPort({...});
  setWorkstationPort({...});
  setAgentPromptTemplatePort({...});
  setExternalAgentActivityPort({...});
  setFlowOrchestrationPort({...});
  setKickoffExecutionPort({...});
  setPromptOverridePort({...});
  // ...
}
```

### AGENTS.md 关键设计哲学条文（真实引用）
Backend Architecture（250-253）：
> Layering is strict: `Interface -> Agents -> Business-Tools -> Business -> Infrastructure`.
> Agents must not import Business directly; use Business-Tools.
> Infrastructure/Business must not import Agents.
> DomainResult: only access `result.error` after `!result.success` or use `getDomainError(result)`.

时间语义约定（298-307，强制）：
> 时间来源单一真源 = 运行环境（Pod TZ），代码零时区假设。
> - 禁用 toISOString() 流出：禁用于 API 返回、日志、DB 写入
> - 禁硬编码时区：禁止 Asia/Shanghai、+08:00 出现在应用代码中
> - DB 时间字段强制带时区：Prisma DateTime 必须 @db.Timestamptz(6)

Span 遥测 I/O（271，强制）：
> 凡新/改 LLM 调用须经 emitLLMCallStart/emitLLMCallEnd，events 含 request/response

数字分身能力默认继承（104-110，强制）：
> 项目空间新增或变更能力时，默认同时对分身生效。不适用分身的必须显式加入排除名单。禁止为分身新建平行实现；需要差异化时用参数或排除名单，不复制一套 service/组件。

Skill Readiness Boundaries L1/L2/L3（91-96）：L1 决策（一次 listAvailable）、L2 规划（仅 snapshot 软校验，禁止 SkillRegistry.resolve/Gate.check）、L3 运行时（SkillReadinessGate.check 为 SSOT）。

### 设计要点
1. 严格分层 + 门禁脚本：5 大层 + Agent 内部 6 子层，check:layers（存量违规 allowlist）+ check:agent-layers:strict（增量零容忍）双门禁。L3 harness 与 L4 domain-harness 横向隔离，channel 与 connectors 横向隔离。
2. Port 注入解耦：Agent 不直 import Business，通过 wireAgentBusinessPorts() 在 startup 注入 17+ 业务 Port，Business 端实现接口、Agent 端只看 Port。
3. DI 容器 + 启动断言：registerServices() → assertCoreDiRegistrations() → side-effect adapter 数量断言（必须 4 个），任何注册缺失 fail-fast。
4. 时间统一真源 = Pod TZ：禁 toISOString() 流出、禁硬编码时区、Prisma DateTime 强制 @db.Timestamptz(6)，启动跑 warnIfTimezoneInconsistent()。
5. 配置 DB 热更新：ConfigDbListener 监听 PG LISTEN/NOTIFY（pg_notify('config_change')），修改 DB 配置无需重新构建。但 LISTEN/NOTIFY 必须直连 PG，不能走 PgBouncer transaction-pool。
6. 数字分身同源继承原则：分身运行在隐藏个人项目（projects.kind='personal'），能力栈与项目空间同源；新增能力默认对分身生效，排除须显式名单；禁止平行实现。

## 补充核实
- retire-agent-execution-log：system-execution-log-cleanup 任务描述明确"agent_execution_log 已退役"。schema 中 agent_execution_log 已无 model 定义，仅 ExecutionSpan + ExecutionSpanEvent 作为 SSOT。
- npm run check:agent-layers:strict 在 package.json:72 真实存在。
- agent_worker_execution 模型第 2053 行——精确核实。
- scheduled-worker.ts 在 src 根下（20 行，含 assertProcessRole('scheduled') 守卫）。
