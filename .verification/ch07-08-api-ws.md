# 源码核实报告：API 层与实时通信（ch07-08）

源码根：`E:/winning/code/AI/winmatrix/winmatrix-server/src/`（所有路径相对此根）。所有行号、签名、代码片段均经直接 Read 核实。

## 主题 1：REST API 设计（第 7 章）

### 关键文件路径

| 文件 | 职责 |
|---|---|
| `startup/api.ts` | Fastify 应用搭建、插件注册、路由挂载总入口（生产 + dev） |
| `startup/apiEntry.ts` | 生产 API 进程入口（`WIN_PROCESS_ROLE=api` 守卫） |
| `index.ts` | dev all-in-one 入口（API + worker） |
| `api.ts`（根目录） | 生产 API 入口薄壳，动态 import `startup/apiEntry.js` |
| `interface/api/index.ts` | `registerRoutes()` — 全部 REST 路由集中注册表 |
| `interface/middleware/errorHandler.ts` | 全局错误处理 + 统一错误响应格式 |
| `interface/middleware/traceId.ts` | TraceId 注入（ALS 全链路） |
| `interface/middleware/apiAuditLog.ts` | 写操作审计日志（ES + PendingSpan Registry） |
| `interface/middleware/jwtAuth.ts` | JWT 认证（Header / X-Auth-Token / query.token） |
| `interface/api/retiredLegacyRestRoutes.ts` | 退役旧 REST 路由 410 守卫 |
| `interface/api/agents/agentRestChatV2.ts` | V2 稳定同步 Chat 端点 |
| `interface/api/agentChatLimiter.ts` | Agent 对话并发限制器 |
| `types/errors.ts` | WinMatrixError 错误类型体系 |

### 核心行号区间与关键导出

**`startup/api.ts`**（共 611 行）
- L74-78: `export const apiServer = Fastify({ logger: true, bodyLimit: 50*1024*1024, pluginTimeout: 60_000 })`
- L93-99: `isStartupComplete()`, `isServerListenSucceeded()` 状态查询
- L102-466: `export async function initRagAndPlugins(): Promise<void>` — 阶段4：插件与路由注册
- L469-548: `export async function startListening(startupStartTime: number): Promise<void>` — 阶段5：监听端口
- L550-586: `export async function startApi(): Promise<void>` — 生产 API 启动序列
- L589-610: `export async function shutdownApi(): Promise<void>` — 优雅停机

**`interface/api/index.ts`**（共 399 行）
- L130-132: `export interface ApiRouteDependencies { projectSkillCredentials: ProjectSkillCredentialRouteOptions }`
- L134-399: `export async function registerRoutes(server, dependencies)` — ~80 个路由插件按前缀挂载

**`interface/middleware/errorHandler.ts`**（共 316 行）
- L90-101: `interface ErrorResponse { success: false; error: { name; message; code; statusCode; timestamp; context?; stack? } }`
- L111-207: `export async function errorHandler(error, request, reply): Promise<void>`
- L228-247: `createErrorResponse(error: WinMatrixError): ErrorResponse`
- L252-266: `export async function notFoundHandler(request, reply): Promise<void>`
- L272-303: `export function asyncHandler<T>(fn): T`

**`interface/api/agents/agentRestChatV2.ts`**（共 822 行）
- L78-88: `export const V2ErrorCodes = { UNAUTHORIZED, INVALID_REQUEST, INVALID_ATTACHMENT_REFS, NO_DISPATCHABLE_EMPLOYEES, PROJECT_NOT_FOUND, FORBIDDEN, WORKSTATION_BUSY, SRE_EMPLOYEE_NOT_FOUND, AGENT_CHAT_FAILED }`
- L137-156: `v2Success(reply, data, traceId?)`, `v2Error(reply, statusCode, code, message, traceId?)`
- L463-822: `export async function agentRestChatV2Routes(server)` — 注册 `POST /chat` 与 `POST /sre/workstation-tasks`

**`types/errors.ts`**
- L11-52: `export class WinMatrixError extends Error` — 基类，含 `code / statusCode / isOperational / timestamp / context`，`toJSON()` 序列化
- 派生类：`ConfigError`(500), `ConfigNotFoundError`(404), `ConfigValidationError`(400), `AgentError`(500), `AgentNotFoundError`(404), `AgentExecutionError`, `LLMError`, `ValidationError`, `PermissionError`, `AuthenticationError`, `ConflictError`, `NotFoundError`, `ServiceUnavailableError`

### 真实代码片段

**片段 1：Fastify 实例与插件注册序列** — `startup/api.ts:74-78, 106-167`
```ts
export const apiServer = Fastify({
  logger: true,
  bodyLimit: 50 * 1024 * 1024,
  pluginTimeout: 60_000,
});
await apiServer.register(metricsPlugin, { endpoint: '/metrics' });
await apiServer.register(cors, {
  origin: true,
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Auth-Token', 'Accept'],
  credentials: true,
});
registerTraceIdMiddleware(apiServer);
registerApiAuditLogMiddleware(apiServer);
await apiServer.register(formbody);
await apiServer.register(multipart, { limits: { fileSize: MULTIPART_MAX_FILE_BYTES } });
await apiServer.register(websocket);
await apiServer.register(rateLimit, {
  max: Number(process.env.RATE_LIMIT_MAX ?? 1000),
  timeWindow: process.env.RATE_LIMIT_WINDOW ?? '1 minute',
  keyGenerator: (request) => { /* X-Forwarded-For 首段或 request.ip */ },
  allowList: (request, _key) => { /* /assets/ /health /readyz 静态资源白名单 */ },
});
apiServer.setErrorHandler(errorHandler);
```

**片段 2：API 版本策略 — V1 退役守卫 + V2 稳定端点** — `interface/api/retiredLegacyRestRoutes.ts:3-33`
```ts
const RETIRED_LEGACY_REST_ROUTES = new Set([
  'POST /api/v1/agents/chat',
  'POST /api/v1/agents/chat/file',
  'POST /api/v1/agents/chat/stop',
  // ... 共 16 条
]);
export function registerRetiredLegacyRestRouteGuard(server): void {
  server.addHook('onRequest', async (request, reply) => {
    const pathname = request.url.split('?', 1)[0];
    const routeKey = `${request.method.toUpperCase()} ${pathname}`;
    if (!RETIRED_LEGACY_REST_ROUTES.has(routeKey)) return;
    return reply.code(410).send({
      code: 'LEGACY_REST_API_RETIRED',
      message: '该旧版 REST API 已退役，请使用 winmatrix-ui 对应接口或 WebSocket 会话入口。',
      websocket: '/api/v1/agents/chat/stream',
    });
  });
}
```
版本挂载点（`interface/api/index.ts:166-169`）：
```ts
await server.register(agentsRoutes, { prefix: '/api/v1/agents' });           // V1（chat 已 410）
await server.register(agentRestChatV2Routes, { prefix: '/api/v2/agents' });  // V2 稳定 REST Chat
```

**片段 3：统一错误响应 + 分类处理** — `interface/middleware/errorHandler.ts:111-133, 159-207`
```ts
export async function errorHandler(error, request, reply): Promise<void> {
  if (reply.sent) return;
  const normalizedStatusCode = isInfrastructureConnectionError(error) ? 503 : (error.statusCode || 500);
  // Stash 错误详情到 request，供审计中间件 onResponse 读取
  (request as any).__auditError = { name, message, code, statusCode, stack, classification, cause, context };
  // WinMatrix 自定义错误 → 直接序列化
  if (error instanceof WinMatrixError) {
    reply.status(error.statusCode).send(createErrorResponse(error)); return;
  }
  // 基础设施连接错误 → 503
  if (isInfrastructureConnectionError(error)) { /* ... */ }
  // Fastify 验证错误 → 400 ParamValidationError
  if (error.validation) { /* ... */ }
  // 透传 4xx（如 rate-limit 429），避免被包成 500
  if (upstreamStatusCode && upstreamStatusCode >= 400 && upstreamStatusCode < 500) { /* ... */ }
  // 未知错误 → 500 INTERNAL_ERROR（生产环境隐藏 message）
  reply.status(500).send(createErrorResponse(internalError));
}
```

### 真实设计要点

1. **分层入口**：生产用 `api.ts`→`startup/apiEntry.ts`（`assertProcessRole('api')` 守卫），dev 用 `index.ts` all-in-one；两者最终都走 `startup/api.ts` 的 `initRagAndPlugins()` + `startListening()`。
2. **插件注册集中在根实例**：CORS、formbody、multipart、cookie、secureSession、websocket、rateLimit 全部在根 `apiServer` 注册一次；固定 50MB 上限。
3. **可配置 basePath**：`process.env.BASE_PATH`（如 `/WxpCopilot1/winmatrix`）将路由与静态文件挂载到前缀，兼容 nginx 去前缀部署。
4. **API 版本策略是"V1 前缀 + 410 守卫 + V2 并行"**：16 条旧 chat/file/stop 等 REST 路由由守卫在 onRequest 统一返回 410 + `LEGACY_REST_API_RETIRED` + 指引 WS 入口；同步 chat 走 V2。
5. **三层限流**：全局 `@fastify/rate-limit`（默认 1000/min，X-Forwarded-For 取首段做 key，静态资源白名单）+ Agent 对话级 `agentChatLimiter`（`AGENT_CHAT_CONCURRENCY`，V2 与 WS 共用）+ 会话级 `conversationRunLocks`（同一会话同时只允许一次 run）。
6. **统一错误信封 + 错误码体系**：`WinMatrixError` 基类携带 `code/statusCode/isOperational/timestamp/context`；errorHandler 分支处理"自定义错误 / 基础设施连接错误(→503) / Fastify 验证错误(→400) / 4xx 透传 / 未知(→500)"，生产环境隐藏内部 message；V2 端点另用独立 `V2ErrorCodes`。
7. **可观测性贯穿请求生命周期**：`traceId` 中间件（onRequest 注入 ALS，onSend 回写响应头）；`apiAuditLog` 中间件用 PendingSpan Registry + sweep 定时器记录写操作到 ES。

## 主题 2：WebSocket 与流式通信（第 8 章）

### 关键文件路径

| 文件 | 职责 |
|---|---|
| `interface/api/agents/agentWebSocket.ts` | WS `/chat/stream` 入口路由、连接生命周期、心跳、stop 处理 |
| `interface/api/agents/ws/agentWebSocketTypes.ts` | `ChatRequestContext` 单参上下文类型 |
| `interface/api/agents/wsChatMessage.ts` | `WsChatMessage` 消息体校验 |
| `interface/api/agents/agentWebSocketConstants.ts` | 心跳/空闲检测常量 |
| `interface/api/agents/ws/agentWebSocketState.ts` | 会话运行锁 + AbortController 注册表 |
| `interface/api/agents/ws/webSocketChannelSetup.ts` | WS 通道创建 + push registry 注册 |
| `interface/api/agents/ws/chatTurnOrchestrator.ts` | 单轮聊天线性编排（核心） |
| `interface/api/agents/ws/WsTurnAdapter.ts` | WS 特有 executionResume 短路 |
| `interface/api/agents/ws/conversationExecutionDispatcher.ts` | 会话模式分发（coordinator/interactive/react） |
| `interface/api/agents/ws/connectionSyncHandler.ts` | `connection:sync` 重连控制帧处理 |
| `infrastructure/channel/WebSocketChannel.ts` | WS 输出通道（emitter→socket.send 转发） |
| `infrastructure/streaming/types.ts` | 流式事件类型"单一真相源" |
| `infrastructure/streaming/index.ts` | 流式模块统一导出 |
| `infrastructure/observability/spans/Emitter.ts` | `Emitter` 类（Hub 兼容门面） |
| `infrastructure/observability/spans/emitterFactory.ts` | `createStreamingEmitter` 工厂 |

### 核心行号区间与关键导出

**`interface/api/agents/agentWebSocket.ts`**（共 508 行）
- L67-162: `async function buildChatTurnContext(data, authenticatedUserId, socketId?): Promise<ChatRequestContext>` — 入口归一化
- L168-369: `async function handleWebSocketMessage(message, socket, authenticatedUserId?, ...)` — 消息处理核心
- L375-508: `export async function agentWebSocketRoutes(server)` — L379 `server.get('/chat/stream', { websocket: true, preHandler: [jwtAuth] }, ...)`；含 L391-438 原生 ping/pong 半断开检测

**`infrastructure/streaming/types.ts`**（共 402 行）
- L16-101: 事件类型定义（`namespace:action` 命名）：
  - ThinkingEventType: `thinking:start|delta|end`
  - ContentEventType: `content:start|delta|replace|end`
  - ToolEventType: `tool:start|result:start|result:delta|result:end|tool:end`
  - LifecycleEventType: `agent:start|end`, `run:start|end|error`
  - DecisionEventType: `decision:start|end`
  - PlanEventType: `plan:init|step|end`
  - HubSpanStreamEventType: `span_started|span_event|span_ended`
  - L64-90: `LegacyStreamEventType`（遗留 WS 协议，含 `conversation:*`, `sub_agent:*`, `react_*`, `async:*`, `cross_agent:*`）
- L112-135: `interface StreamEvent<T>` — 统一事件结构（type/runId/seq/conversationId/parentTaskId/targetChannels/ts/spanId/traceId/parentSpanId/payload）
- L137-142: `streamEventTargetsChannel(event, channel)` — 通道过滤谓词
- L148-343: 各 PayloadMap
- L396-401: `DEFAULT_TOOL_RESULT_STREAM_CONFIG = { chunkSize: 5, chunkDelay: 0, minLength: 50, wrapInCodeBlock: false }`

**`infrastructure/observability/spans/Emitter.ts`**（共 339 行）
- L122-330: `export class Emitter implements IEmitter`
  - L131-151: `constructor(runId, deps: EmitterDeps, conversationId?)`
  - L178-180: `subscribe(listener): Unsubscribe`
  - L190-192: `emitStream(type: EmittableStreamEventType, payload): void`
  - L215-251: `thinking = {start, delta, end}`, `content = {start, delta, end}`, `tool.result = {start, delta, end}`
  - L253-259: `emitPhase(phase)`, `emitRunError(code, message)`

**`infrastructure/channel/WebSocketChannel.ts`**（共 257 行）
- L28-54: `export class WebSocketChannel implements PushableChannel` — `capabilities = { streaming: true, push: true }`
- L196-229: `private subscribeToEmitter(emitter)` — 核心：`emitter.subscribe(event => { if(!streamEventTargetsChannel(event,'websocket')) return; socket.send(JSON.stringify(event)) })`
- L234-255: `updateSocket(socket)`（重连）、`updateEmitter(emitter)`（新轮次）

**`interface/api/agents/ws/chatTurnOrchestrator.ts`**（共 627 行）
- L72-106: `class ContentTracker` — 数组收集 + 一次性拼接，监听 `content:delta/replace/end`
- L137-144: `export async function executeChatTurn(ctx, socket): Promise<void>`
- L146-626: `executeChatTurnBody` — 线性流程：streaming context → 准入校验 → 用户消息落库 → TurnRunner.run → role.run（带 streaming timeout）→ 内容投影 → 持久化 → 发布 terminal

**`interface/api/agents/ws/conversationExecutionDispatcher.ts`**（共 254 行）
- L23-47: `export class ConversationExecutionDispatcher` — 按 `sessionMode` 选 adapter
- L57-225: `export class CoordinatorAdapter implements ConversationExecutionAdapter` — 内存队列 + 队列位置通知 + 缓冲消息逐条执行
- L245-253: `export function createDefaultConversationExecutionDispatcher()` — 注册 coordinator/interactive/react 三个 adapter

**`interface/api/agents/ws/agentWebSocketState.ts`**（共 52 行）
- L9: `export const conversationRunLocks = new Map<string, boolean>()`
- L12: `export const conversationAbortControllers = new Map<string, AbortController>()`
- L20-51: `findAbortControllerByWeComContext(wecomUserId, chatId?)`

**`interface/api/agents/agentWebSocketConstants.ts`**（共 13 行）
- L6: `IDLE_CHECK_MS = 5_000`; L9: `HEARTBEAT_INTERVAL_MS = 25_000`; L12: `HEARTBEAT_THRESHOLD_MS = 25_000`

### 真实代码片段

**片段 1：WS 路由 + JWT preHandler + 原生 ping/pong 半断开检测** — `interface/api/agents/agentWebSocket.ts:375-438, 487-506`
```ts
export async function agentWebSocketRoutes(server: FastifyInstance) {
  const jwtService = await container.resolve<JwtService>('JwtService');
  const jwtAuth = createJwtAuthMiddleware(jwtService);
  server.get('/chat/stream', { websocket: true, preHandler: [jwtAuth] }, (connection, req) => {
    const socket = getWsSocket(connection);
    const socketId = `ws_${Date.now()}_${Math.random().toString(36).slice(2, 10)}`;
    incWebSocketConnection('agent');
    // 服务端原生 ping/pong 半断开检测（浏览器自动响应 ping 帧）
    const PING_INTERVAL_MS = 30_000;
    const PONG_DEADLINE_MS = 10_000;
    let pongReceived = true;
    if (canPing) {
      rawSocket.on?.('pong', () => { pongReceived = true; clearTimeout(pongTimer); });
      pingTimer = setInterval(() => {
        if (!pongReceived) { rawSocket.terminate(); return; }  // 上一轮未 pong → terminate
        pongReceived = false;
        rawSocket.ping();
        pongTimer = setTimeout(() => { if (!pongReceived) rawSocket.terminate(); }, PONG_DEADLINE_MS);
      }, PING_INTERVAL_MS);
    }
    socket.on('close', () => {
      decWebSocketConnection('agent');
      clearInterval(pingTimer); clearTimeout(pongTimer);
      for (const setup of conversationChannelSetups.values()) setup.close();
      for (const convId of registeredConversations) conversationPresenceService.unregister(convId, socketId);
    });
  });
}
```

**片段 2：LLM 流式 token → 客户端的核心转发链** — `infrastructure/channel/WebSocketChannel.ts:196-216`
```ts
private subscribeToEmitter(emitter: Emitter): void {
  this.unsubscribe = emitter.subscribe((event: StreamEvent) => {
    if (!this.isAlive()) return;
    if (!streamEventTargetsChannel(event, 'websocket')) return;  // 通道过滤
    try {
      const json = JSON.stringify(event);
      this.socket.send(json);                                   // 直接转发为 WS 文本帧
      this._lastOutputTime = Date.now();
    } catch (error) {
      logger.warn(`[WebSocketChannel] 事件转发失败: ${errMsg}`);
    }
  });
  this.projectionUnsubscribe = emitter.subscribeChatProjection((event) => {
    if (!this.isAlive()) return;
    this.socket.send(JSON.stringify(event));
    this._lastOutputTime = Date.now();
  });
}
```
配合 Emitter 领域便捷方法（`Emitter.ts:227-237`）：
```ts
content = {
  start: (_span?) => { this.hub.stream.content.start(_span); },
  delta: (text, _span?) => { this.hub.stream.content.delta(text, _span); },  // 每个 token delta
  end: (content, _span?) => { this.hub.stream.content.end(content, _span); },
};
```

**片段 3：stop 控制帧 + AbortController 中断活跃会话** — `interface/api/agents/agentWebSocket.ts:180-254`
```ts
// 0. 优先检查 stop 消息（不经 AI 管道，直接中断活跃会话）
if (rawParsed.success && rawParsed.data?.type === 'stop') {
  const convId = String(rawParsed.data.conversationId ?? '');
  const abortCtrl = conversationAbortControllers.get(convId);
  if (abortCtrl) {
    abortCtrl.abort();                                  // 触发 AbortError
    const stopEmitter = createStreamingEmitter(`stop_${Date.now()}`, convId);
    const stopUnsub = stopEmitter.subscribe((event) => { if (socket.readyState === 1) socket.send(JSON.stringify(event)); });
    try { stopEmitter.emitRunError('stopped_by_user', '用户已手动停止本次会话'); }
    finally { stopUnsub(); stopEmitter.clearListeners(); }
  } else {
    nopEmitter.emitPhase('no_active_run');
    nopEmitter.emitRunError('noop', '当前无进行中的任务');
  }
  return;
}
```

### 真实设计要点

1. **单一 WS 主入口 `/api/v1/agents/chat/stream`**：`agentWebSocketRoutes` 注册到 `/api/v1/agents` 前缀，`preHandler: [jwtAuth]` 强制 JWT 认证，token 走 Header / X-Auth-Token / query.token 三路解析。外部 Agent 另有独立 WS `/api/v1/external-agents/connect`。
2. **流式转发是"Emitter subscribe → JSON.stringify → socket.send"**：协议是 **WebSocket 文本帧**（非 SSE）。`WebSocketChannel` 在构造时自动 `emitter.subscribe`，对每个 `StreamEvent` 做 `streamEventTargetsChannel(event,'websocket')` 过滤后 `socket.send(JSON.stringify(event))`；LLM 侧用 `emitter.content.delta(text, span)` 逐 token 推送。
3. **统一事件类型体系（单一真相源）**：`infrastructure/streaming/types.ts` 定义全部流式事件，命名规范 `namespace:action`，含 8 大类 + 一组 `LegacyStreamEventType`；每个 `StreamEvent` 带 `runId/seq/conversationId/spanId/traceId/parentSpanId/targetChannels`；`seq` per-conversation 递增防串流。
4. **会话状态机由 ConversationExecutionDispatcher 驱动**：入口归一化后按 `sessionMode`（coordinator/interactive/react）选 adapter；CoordinatorAdapter 用内存队列实现"会话内串行执行 + 排队位置通知 + 缓冲消息逐条回放"。`conversationRunLocks` Map 保证同一会话同时只允许一次 run。
5. **断线/重连机制**：(a) 半断开检测用服务端原生 ws ping/pong（30s ping + 10s pong deadline，超时 `terminate()`），不依赖应用层 heartbeat；(b) 重连 sync 通过 `connection:sync` 控制帧（绕过 agentChatLimiter）：批量鉴权 → 为授权会话重新注册 push registry → 批量读 canonical 状态 → 回 `connection:sync_result`，鉴权失败会话绝不注册 push。sync 帧有 schema version。
6. **stop / 中断 / terminal 三态收口**：前端发 `{type:'stop', conversationId}` → 查 `conversationAbortControllers` → `abortCtrl.abort()` 触发 AbortError → orchestrator catch 块识别 → 发布 `buildStoppedTerminal` / `buildFailedTerminal` / `buildCompletedTerminal` / `buildWaitingUserInputTerminal`。terminal 是根回合唯一收口点，持久化成功后才发布。
7. **异步推送与多端路由**：`WebSocketChannel` 同时实现 `streaming` 与 `push` 能力；`webConversationPushRegistry` 按 conversationId + ownerId（socketId）注册回调，支持会话迁移。事件 `targetChannels` 字段支持 websocket/wecom/dashboard 三通道路由。

## CLAUDE.md 声明核实
- "V1 POST /api/v1/agents/chat 已 410，V2 才是同步 chat" — **属实**。
- WS 入口路径 `/api/v1/agents/chat/stream` — **属实**。
