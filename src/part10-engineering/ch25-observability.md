# 第 25 章 可观测性

> "看不见的系统是不可控的。"

可观测性是生产级系统的生命线。WinMatrix 构建了一套完整的可观测性体系——从全链路 TraceId 到执行 Span、从 API 审计到数据脱敏。本章将深入这些实现。

## 25.1 可观测性模块概览

`src/infrastructure/observability/` 包含 60+ 文件，提供多层可观测能力：

```typescript
// src/infrastructure/observability/index.ts（第 1-31 行）
/**
 * 简化的可观测性模块
 * 直接使用 Pino 日志系统记录关键事件
 */
import { v4 as uuidv4 } from 'uuid';
import { getHybridObservabilityLogger } from './HybridLogger.js';

// 工具函数
export { calculateDuration, getLogLevel, getSamplingConfig, shouldSample } from './utils.js';

/** 时间戳与事件记录 */
export { formatTimestamp, logEvent, recordObservabilityEvent } from './eventCore.js';

/** ObservabilityHub（事件中枢） */
export {
  getObservabilityHub,
  registerObservabilityHubShutdown,
  ObservabilityHub,
  ChannelSink,
  LogSink,
  SpanSink,
} from './hub/index.js';

/** 链路追踪上下文 */
export {
  generateTraceId,
  runWithTraceId,
  setTraceId,
  getTraceId,
  getParentTraceId,
} from './traceContext.js';
```

### 子模块组织

| 子目录 | 职责 |
|--------|------|
| `hub/` | ObservabilityHub（事件中枢）、SpanSink、LogSink、ChannelSink |
| `spans/` | Span 上下文、类型、发射器 |
| `redaction/` | DataSafetyKernel、脱敏引擎、敏感字段规则 |
| （根） | HybridLogger、DatabaseLogger、ElasticsearchLogService、性能监控 |

## 25.2 TraceContext：全链路追踪

`traceContext.ts` 基于 AsyncLocalStorage 实现 traceId 的隐式传播：

```typescript
// src/infrastructure/observability/traceContext.ts（完整 54 行）
import { AsyncLocalStorage } from 'node:async_hooks';
import { randomUUID } from 'node:crypto';

interface TraceContext {
  traceId: string;            // 链路追踪 ID
  parentTraceId?: string;     // 父链路 ID（跨 Agent 调用）
}

const storage = new AsyncLocalStorage<TraceContext>();

/** 生成新的 traceId（32 位小写 hex，无连字符） */
export function generateTraceId(): string {
  return randomUUID().replace(/-/g, '');
}

/** 在给定 traceId 的上下文中执行 fn */
export function runWithTraceId<T>(
  traceId: string | undefined,
  fn: () => T,
  parentTraceId?: string,
): T {
  const id = traceId || generateTraceId();
  return storage.run({ traceId: id, parentTraceId }, fn);
}

/** 设置当前异步执行上下文的 traceId（HTTP 请求场景） */
export function setTraceId(traceId: string, parentTraceId?: string): void {
  storage.enterWith({ traceId, parentTraceId });
}

/** 获取当前链路 traceId */
export function getTraceId(): string | undefined {
  return storage.getStore()?.traceId;
}

/** 获取父链路 traceId */
export function getParentTraceId(): string | undefined {
  return storage.getStore()?.parentTraceId;
}
```

### ALS 传播的价值

AsyncLocalStorage 让 traceId **跨异步边界自动传播**——无需在函数参数中显式传递。在一个涉及多层异步调用（HTTP → Service → Repository → LLM）的请求中，任何代码都能通过 `getTraceId()` 获取当前 traceId，实现全链路关联。

### traceId 格式

```typescript
return randomUUID().replace(/-/g, '');  // 32 位小写 hex
```

去掉连字符的格式兼容性更好（某些日志系统对连字符处理有问题）。

## 25.3 执行 Span 系统

ExecutionSpan 是可观测性的基本单元，记录一个操作的完整生命周期：

### Span 数据模型

```typescript
// src/infrastructure/persistence/repositories/ExecutionSpanRepository.ts（第 35-64 行）
export interface InsertExecutionSpanInput {
  spanId: string;
  traceId: string;
  parentSpanId?: string;       // 父 Span（支持嵌套）
  spanKind: string;             // llm_call / agent_invocation / tool_call / decision 等
  spanName: string;
  status: string;
  startedAt: Date;
  endedAt?: Date;
  durationMs?: number;
  // 角色标识
  role?: string;
  digitalEmployeeId?: string;
  // 上下文
  conversationId?: string;
  projectId?: string;
  agentRunId?: string;
  // Token 计数
  tokenInput?: number;
  tokenOutput?: number;
  tokenThinking?: number;
  model?: string;
  provider?: string;
  // 事件
  events: SpanEvent[];
  attributes?: Record<string, unknown>;
}
```

### SpanKind 分类

Agent 执行日志的 eventType 映射到 spanKind：

```typescript
// src/infrastructure/persistence/repositories/AgentExecutionRepositoryImpl.ts（第 35-45 行）
export function executionEventTypeToSpanKind(eventType: string): string | undefined {
  if (eventType.startsWith('llm_call')) return 'llm_call';
  if (eventType.startsWith('agent_call') || eventType.startsWith('agent_invoke')) return 'agent_invocation';
  if (eventType.startsWith('tool_call') || eventType.startsWith('extractor')) return 'tool_call';
  if (eventType.startsWith('decision_call')) return 'decision';
  if (eventType.startsWith('workflow_call')) return 'pipeline';
  if (eventType.startsWith('retrieval_call')) return 'retrieval';
  if (eventType.startsWith('mode_preflight')) return 'preflight';
  if (eventType === 'agent_run_start' || eventType === 'agent_run_end') return 'pipeline';
  return undefined;
}
```

## 25.4 Span 事件双写

Span 事件采用**子表 + JSON 双写**策略：

```typescript
// src/infrastructure/persistence/repositories/ExecutionSpanEventRepository.ts（第 1-70 行）
/**
 * 子表为完整事件源，JSON 为有界 UI 兼容副本。
 * 子表 INSERT 失败为 warn-only。
 */

export interface AppendSpanEventInput {
  spanId: string;
  traceId: string;
  eventType: string;
  phase?: string;
  role?: string;
  ts: string | Date;
  attrs?: Record<string, unknown>;
  seq: number;                 // span 内单调序号
}

export class ExecutionSpanEventRepository {
  /**
   * 幂等 append：@@unique([span_id, seq]) 冲突（P2002）视为重复写，静默忽略。
   */
  async appendEvent(input: AppendSpanEventInput): Promise<boolean> {
    try {
      await prisma.executionSpanEvent.create({
        data: {
          spanId: input.spanId,
          traceId: input.traceId,
          eventType: input.eventType,
          // ...
        },
      });
      return true;
    } catch (error) {
      if (error instanceof Prisma.PrismaClientKnownRequestError && error.code === 'P2002') {
        return false;   // 幂等：重复写静默忽略
      }
      throw error;
    }
  }
}
```

### 双写设计

- **子表（execution_span_event）**：完整事件源，支持精确查询
- **JSON（execution_span.events）**：有界 UI 兼容副本，便于前端展示

### 幂等 append

`@@unique([span_id, seq])` 约束 + P2002 冲突静默忽略——确保回填/重放安全，不会产生重复事件。

## 25.5 API 审计日志

API 审计日志的 `finalizeNormal` 函数构建完整的事件记录（见第 7 章）：

```typescript
// src/interface/middleware/apiAuditLog.ts（第 226-365 行）
function finalizeNormal(span: PendingSpan, statusCode: number, request, reply): void {
  if (span.completed) return;
  span.completed = true;

  const finalTraceId = span.traceId || `audit-${span.registryKey}`;
  const duration = Date.now() - span.startTime;

  // JWT 解码（如果路由跳过了 JWT 鉴权）
  // 敏感字段脱敏（redactSensitiveBodyFields）
  // body 截断（truncateBody）
  // 错误上下文合并（__auditError + fallback）

  const event: ApiRequestEvent = {
    timestamp: nowIso(),
    sessionId: finalTraceId,
    eventType: 'api_request',
    traceId: finalTraceId,
    completion: 'normal',           // 正常完成
    userId, userName,
    duration,
    method: span.method,
    path: decodeURIComponent(span.path),
    statusCode,
    requestBody, responseBody,
    error: isError ? (auditError?.message || fallbackError) : undefined,
    clientIp: span.clientIp,
    userAgent: span.userAgent,
    routePattern,
    requestSize, responseSize,
    projectId, conversationId,
    authMethod: detectAuthMethod(request),
    slowRequest: duration > SLOW_REQUEST_THRESHOLD_MS,   // 慢请求标记
  };

  void recordObservabilityEvent(event).catch((err) => {
    logger.warn({ error: getErrorMsg(err), traceId: finalTraceId }, '[apiAuditLog] ES 写入失败');
  });

  registry.delete(span.registryKey);
  // 清理监听器，防止 socket 复用时内存泄漏
  request.raw.removeAllListeners('close');
  request.raw.removeAllListeners('error');
}
```

### 错误分类

```typescript
// src/interface/middleware/apiAuditLog.ts（第 200-207 行）
function classifyHttpError(statusCode: number): string {
  if (statusCode >= 500) return 'ServerError';
  if (statusCode === 429) return 'RateLimited';
  if (statusCode === 401) return 'Unauthorized';
  if (statusCode === 403) return 'Forbidden';
  if (statusCode === 404) return 'NotFound';
  if (statusCode >= 400) return 'ClientError';
  return 'Unknown';
}
```

### 认证方式检测

```typescript
function detectAuthMethod(request: FastifyRequest): string {
  const authHeader = request.headers['authorization'];
  if (authHeader && authHeader.startsWith('Bearer ')) return 'bearer_token';
  if (request.session?.userId) return 'cookie_session';
  return 'none';
}
```

## 25.6 数据脱敏

可观测性数据的脱敏是安全的关键一环：

```typescript
// src/infrastructure/observability/redaction/
// DataSafetyKernel.ts      - 数据安全内核
// redactionEngine.ts        - 脱敏引擎
// sensitiveFieldRules.ts    - 敏感字段规则
```

### Sink 级别脱敏

```typescript
// ExecutionSpanEventRepository.ts
function toJsonAttrs(value: Record<string, unknown> | undefined): Prisma.InputJsonValue {
  if (value === undefined) return {};
  const redacted = redactForSink(value, 'database', { policy: 'strict' });
  return JSON.parse(JSON.stringify(redacted)) as Prisma.InputJsonValue;
}
```

不同 sink（database / elasticsearch / log）有不同的脱敏策略——数据库使用 strict 策略，确保敏感数据不入库。

## 25.7 ObservabilityHub

ObservabilityHub 是事件中枢，支持多种 Sink：

```typescript
// src/infrastructure/observability/hub/
// ObservabilityHub.ts - 事件中枢
// ChannelSink.ts      - 通道 Sink（WebSocket 推送）
// LogSink.ts          - 日志 Sink
// SpanSink.ts         - Span Sink
```

Hub 将事件分发到多个 Sink，支持实时推送（Dashboard）和持久化（ES/DB）。

## 25.8 性能监控

```typescript
// src/infrastructure/observability/performanceMonitor.ts
// 性能指标采集
// - 事件循环滞后（startEventLoopLagTracking）
// - 进程状态（updateProcessStateMetric）
// - Redis 连接状态（startMetricsLogging）
// - 内存使用
```

### 进程状态指标

```typescript
// src/infrastructure/process/processState.ts
// 状态机：starting → running → shutting_down → fatal_exiting
// 通过 Prometheus 暴露
updateProcessStateMetric('running');
```

## 本章小结

本章深入分析了 WinMatrix 的可观测性系统：

1. **60+ 文件的可观测性模块**：hub / spans / redaction 三层组织
2. **TraceContext**：基于 ALS 的全链路追踪，跨异步边界隐式传播
3. **ExecutionSpan**：完整生命周期记录，含 Token 计数和模型信息
4. **SpanKind 分类**：llm_call / agent_invocation / tool_call / decision 等
5. **事件双写**：子表（完整源）+ JSON（UI 兼容），幂等 append
6. **API 审计**：中断请求捕获 + 错误分类 + 认证方式检测 + 慢请求标记
7. **数据脱敏**：Sink 级别策略，数据库使用 strict 策略
8. **ObservabilityHub**：多 Sink 分发（实时推送 + 持久化）
9. **性能监控**：事件循环滞后 + 进程状态 + 连接状态

在下一章中，我们将深入构建与部署。
