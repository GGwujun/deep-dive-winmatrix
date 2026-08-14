# 源码核实报告：基础设施层（ch04-06）

源码根：`E:/winning/code/AI/winmatrix/winmatrix-server/src/`，schema：`prisma/schema.prisma`

## 主题 1：数据库与持久化（第 4 章）

### 1.1 Prisma schema 事实
- **model 总数：157 个**（实测 grep -c）
- **schema 总行数：4065 行**
- generator（行 1-5）：`provider="prisma-client"`，`output=../src/infrastructure/persistence/prisma/generated`，`previewFeatures=["partialIndexes"]`
- datasource（行 7-9）：`provider="postgresql"`

### 1.2 Prisma Client 封装
**文件**：`src/infrastructure/persistence/prisma/client.ts`（487 行）
- `createPrismaPool()` 行 111-123 — pg.Pool（`PRISMA_POOL_MAX` 默认 25、keepAlive 10s）
- `buildConnectionString()` 行 71-90 — 强制 `options=-c TimeZone=UTC`（规避 Prisma #28629）
- `withPrismaPoolClient<T>()` 行 166-175 — 同一 pg 连接跑 session 语义（advisory lock）
- `rebuildPrismaResources()` 行 193-221 — **single-flight 重建**
- `isRecoverablePrismaPoolError()` 行 223-236 — 9 种可恢复错误模式
- `createPrismaProxy()` 行 334-366 — ES Proxy 包裹
- `withPrismaRecovery()` 行 305-332 — 命中可恢复错误→重建→对只读方法重放
- `shouldReplayAfterPrismaRebuild` 行 288-303 — 只读方法白名单（findMany/findFirst/aggregate/count/$queryRaw）
- `connectPrisma()` 行 414-448 — 启动重试（5 次，指数退避，封顶 30s）
- `getPrismaPoolMetrics()` 行 400-409 + `registerPoolMetrics()` 行 454-486 — prom-client Gauge

single-flight 重建（`client.ts:193-221`）：
```ts
async function rebuildPrismaResources(reason: unknown): Promise<void> {
  if (globalForPrisma.prismaRebuildInFlight) {
    return globalForPrisma.prismaRebuildInFlight;
  }
  globalForPrisma.prismaRebuildInFlight = (async () => {
    const previousResources = getRuntimeResources();
    const nextResources = createPrismaResources();
    runtimeResources = nextResources;
    publishPrismaResources(nextResources);
    logger.warn({ reason: getErrorMsg(reason) }, '[Prisma] 检测到可恢复连接池错误，已重建 Prisma Client 与 pg.Pool');
    await closePrismaResources(previousResources);
  })();
  try { await globalForPrisma.prismaRebuildInFlight; }
  finally { globalForPrisma.prismaRebuildInFlight = undefined; }
}
```

### 1.3 乐观锁 + 事务
**文件**：`src/interface/api/adminRoleWorkstationRoutes.ts` 行 190-247（`workstation_config_version` 字段）
```ts
const transactionResult = await prisma.$transaction(async (tx) => {
  const current = await tx.agent_config.findFirst({
    where: { id: roleId, projectId: null },
    select: { id: true, workstation: true, workstation_config_version: true, updated_at: true },
  });
  if (!current) throw new RoleWorkstationRouteError(404, NOT_FOUND_CODE, '...');
  if (current.workstation_config_version !== body.expectedVersion)
    throw new RoleWorkstationRouteError(409, CONFLICT_CODE, '配置已被其他管理员更新，请重新加载');
  const updated = await tx.agent_config.updateMany({
    where: { id: roleId, projectId: null, workstation_config_version: current.workstation_config_version },
    data: { workstation: merged, workstation_config_version: { increment: 1 }, updated_at: nextUpdatedAt },
  });
  if (updated.count !== 1)
    throw new RoleWorkstationRouteError(409, CONFLICT_CODE, '配置已被其他管理员更新，请重新加载');
  return { configVersion: current.workstation_config_version + 1 };
});
```
schema 中带 `version Int` 字段的 model 有 10 处。

### 1.4 分布式锁
- `src/infrastructure/persistence/distributedLock.ts`（51 行）— Redis 锁，`SET NX EX` + Lua（owner 校验防误释放）
```ts
export async function acquireKickoffLock(jobId: string, owner: string, ttlSeconds = 30): Promise<boolean> {
  const redis = await getRedisClient();
  const result = await redis.set(lockKey(jobId), owner, 'EX', ttlSeconds, 'NX');
  return result === 'OK';
}
```
- `src/infrastructure/persistence/advisoryLock.ts`（15 行）— **已退化为 no-op**，PG advisory lock 已废弃。

### 1.5 持久化状态机
`src/infrastructure/persistence/agentRunLedgerStatus.ts`（160 行）
- `AGENT_RUN_ACTIVE_STATUSES`（5 个）/ `AGENT_RUN_TERMINAL_STATUSES`（7 个）
- `resolveAgentRunLedgerStatusFromCloseContext()` 行 90-115 — SSOT，从 Coordinator/CloseStage 推导 agent_run.status

### 1.6 多层缓存
**(A) `MultiLevelCache`**（L1 内存 → L2 文件 → L3 Redis）`src/infrastructure/cache/multiLevelCache.ts`（337 行）；L1 = `Map<string, CacheEntry>`（`CONFIG_MEMORY_TTL` 默认 1h），L2 = `FileCache`，L3 = `RedisCache`。
**(B) `EntityCache`**（L1 内存 + Redis）`src/infrastructure/cache/EntityCache.ts`；5 个 scope（de/sa/srb/ac/wst，各 15-30min TTL）。

### 主题1 设计要点
1. 连接出口收敛：1 个 pg.Pool + 1 个 PrismaPg adapter + 1 个 PrismaClient，`globalForPrisma` 防 HMR 重复实例化。
2. PgBouncer transaction 模式纵深防御：keepAliveInitialDelayMillis=10s + Proxy 重建兜底。
3. single-flight 重建防整点回收风暴。
4. 只读重放策略：仅 9 种只读方法重放，写操作不重放。
5. 乐观锁 + 事务：version 字段 + updateMany where + count!==1 判定 → 409。
6. 分布式锁走 Redis，PG advisory lock 已废弃。

## 主题 2：缓存与消息队列（第 5 章）

### 2.1 Redis 连接矩阵（6 条）
`src/infrastructure/persistence/database/RedisConnectionManager.ts`（128 行）：`shared / bullmq-queue / bullmq-worker / wecom-lazy / wecom-pubsub / redis-cache`
- shared: `database/redis.ts` 行 23-35 — JWT 黑名单/分布式锁
- bullmq-queue: `database/bullmqConnections.ts` 行 15-23 — `commandTimeout` 30s
- bullmq-worker: 行 25-33 — **明确不加 commandTimeout**（blocking 命令不可 timeout）
- wecom-lazy: `database/lazyRedis.ts` 行 42-64 — 状态机 uninit→ready/degraded，60s±15s jitter 重连
- redis-cache: `cache/redisCache.ts` 行 49-61 — `retryStrategy: Math.min(times*200,5000)` 永不放弃

`RedisCache`：`src/infrastructure/cache/redisCache.ts`（399 行），key 前缀 `winmatrix:config:`，TTL `CONFIG_REDIS_TTL` 默认 86400s，`clear()` 用 SCAN 而非 KEYS。

### 2.2 BullMQ 队列
`src/infrastructure/queue/queue.ts`（85 行）
- 行 30-43 创建 3 队列：`scheduled-agent` / `scheduled-system` / `scheduled-light`
- `defaultJobOptions`：`attempts: 2`，指数退避 5s，`removeOnComplete`/`removeOnFail` 按 count+age 保留
- `getQueueForTask(taskName)` 行 49-53 — 按任务名路由

`src/infrastructure/queue/queueRegistry.ts`（71 行，纯数据零副作用）
- `QUEUE_REGISTRY` 行 22-43 — 10 个队列元数据
- `SYSTEM_QUEUE_TASK_NAMES` 行 51-60 — 8 个系统任务
- `LIGHT_QUEUE_TASK_NAMES` 行 62-70 — 7 个轻量任务

`src/infrastructure/queue/runtimeQueueIsolation.ts`（104 行）
- `resolveRuntimeIsolationId()` 行 28-42 — 生产强制 `prod`，非生产按 hostname
- `resolveBullmqQueueName()` 行 48-59 — 非生产加 `-host-{isolationId}` 后缀，防开发机订阅共享队列
- 行 35-39：生产 isolationId 非 prod 直接抛错

进程内队列：`src/infrastructure/queue/InputQueue.ts`（239 行）— `InputQueue` 会话输入合并队列（非 BullMQ）；`memorySyncQueue.ts`（21 行）— attempts:3。

### 2.3 配置热更新（PG LISTEN/NOTIFY）
发送端 `src/infrastructure/persistence/database/notifyConfigChange.ts`（14 行）：
```ts
export async function notifyConfigChange(payload: ConfigChangeNotifyPayload): Promise<void> {
  await prisma.$executeRaw`SELECT pg_notify('config_change', ${JSON.stringify(payload)})`;
}
```
接收端 `src/agents/core/kernel-management/config/listener/ConfigDbListener.ts`（414 行）
- 约束（行 90-112）：**必须走真实 PG 会话，不能经 PgBouncer transaction pooling**，优先取 `DATABASE_LISTEN_URL`
- `connect()` 行 209-236 — 独立 `pg.Client`（不用连接池），`LISTEN config_change`
- `handleNotification()` 行 241-271 — 去重放入 `pendingChanges` Map（key=`${type}:${id}`）
- `scheduleDebounce()` 行 336-345 — **500ms 防抖合并**
- `scheduleReconnect()` 行 296-331 — 指数退避（base 1s，封顶 30s）
- `setNotificationSuppressed()` 行 194-204 — bulk 写入期间暂停

去重 + 防抖（`ConfigDbListener.ts:241-271`）：
```ts
private handleNotification(msg: pg.Notification): void {
  if (msg.channel !== 'config_change' || !msg.payload) return;
  if (this.notificationSuppressed) return;
  try {
    const payload: ConfigChangePayload = JSON.parse(msg.payload);
    const key = `${payload.type}:${payload.id}`;
    this.pendingChanges.set(key, { configType: payload.type, configId: payload.id, action: payload.action, timestamp: payload.timestamp });
    this.scheduleDebounce();
  } catch (error) { logger.warn(`...`); }
}
```
启动：`src/startup/common.ts` 行 331-358（`STARTUP_SKIP_CONFIG_DB_LISTENER=true` 可跳过）。

### 主题2 设计要点
1. Redis 连接分类管理：6 条按用途隔离，统一 `RedisConnectionManager`。
2. BullMQ 队列按负载分三档。
3. 运行时队列隔离：开发环境加 hostname 后缀，生产强制 prod 并校验。
4. 配置热更新走 PG LISTEN/NOTIFY（非 Redis Pub/Sub）：独立 pg.Client + 500ms 防抖 + Map 去重；不能走 PgBouncer transaction pooling。
5. Worker 连接禁用 commandTimeout（blocking 命令）。
6. 多级缓存分级：配置走 L1+L2+L3，业务实体走 L1+Redis。

## 主题 3：认证与授权（第 6 章）

### 3.1 JWT
`src/infrastructure/auth/JwtService.ts`（273 行）
- `JwtPayload` 行 8-19（userId/username/isAdmin）；`ThirdPartyJwtPayload` 行 25-36（SSO，systemId/expTimestamp）
- 构造器行 57-64：**强制 secret ≥ 32 字符**
- `generateToken(userId, username, isAdmin)` 行 69-84 — HS256，`expiresIn='24h'`
- `verifyToken(token)` 行 90-107 — 校验签名 + 过期/无效分类抛错
- `verifyAndCheckBlacklist(token)` 行 112-125 — verify + Redis 黑名单
- `revokeToken(token)` 行 131-158 — 解码取 exp，`setex(jwt:blacklist:{token}, ttl, '1')`，TTL 与 token 剩余一致
- `isTokenRevoked(token)` 行 163-177 — Redis 出错时返回 false 放行（可用性优先）
- `verifyThirdPartyToken(token, secrets)` 行 216-272 — SSO 多密钥轮询

登录：`src/business/domain/auth/AuthService.ts` — `login` 行 88-127（bcrypt → generateToken），`logout` 行 132-141（revokeToken）。

### 3.2 Fastify 鉴权中间件
`src/interface/middleware/jwtAuth.ts`（148 行）
- `createJwtAuthMiddleware(jwtService)` 行 79-104 — preHandler，无 token/校验失败 401
- `getTokenFromRequest()` 行 112-125 — **三路 token**：Authorization Bearer → X-Auth-Token → query.token
- `attachUserFromToken()` 行 22-31 — verify 后查 DB 规范化用户身份，不一致告警
- `createOptionalJwtAuthMiddleware()` 行 132-147 — 可选鉴权

`src/interface/middleware/projectPermission.ts`（128 行）— `createProjectPermissionGuard(getMemberRole)` 行 45-91，流程：认证检查 → 系统管理员旁路 → 提取 projectCode → 查角色 → `hasProjectPermission`。
`src/interface/middleware/permission.ts`（177 行）— 5 个 preHandler 工厂：requirePermission/requireAnyPermission/requireAllPermissions/requireRole/requireAdmin。
Fastify 装配：`src/interface/core/app.ts` 行 43-282。

### 3.3 权限模型
schema 行 2733-2756：`permission_definition`（name 描述 category）/ `role_permission_binding`（role permission enabled，`@@unique([role, permission])`）。
消费方 `src/business/domain/rbac/PermissionService.ts`：`checkPermission` 行 23-49（Redis 缓存 300s），`getUserPermissions` 行 86-96。

**两套权限并存**：静态矩阵 `src/business/domain/project/projectPermission.ts`（PermissionLevel 0-5、15 个 ProjectRole、60+ PermissionKey M1-M14，编译期常量，前后端各维护一份）+ 动态 RBAC（DB 表 + PermissionService）。

### 3.4 token 模型
- `personal_access_tokens`（schema 2497-2513，"个人访问令牌（MCP Bridge 鉴权）"）：`tokenHash`（唯一，SHA-256）、`defaultProjectId`、`expiresAt`、`lastUsedAt`、`revokedAt`。消费方 `src/infrastructure/auth/PersonalAccessTokenStore.ts`（248 行）：`generate` 生成 `wm_pat_{random16hex}` 强制绑定默认项目，`verify` 按 hash 查库。
- `external_agent_computer_token`（schema 2478-2494）：外部 Agent Computer 托管模式令牌。

**MCP Token Broker** `src/interface/mcp/tokenBroker.ts`（370 行）— `resolve(token)` 行 353-367 按 token 前缀路由：
```ts
export async function resolve(token, extra?): Promise<TokenBrokerResolveResult> {
  if (token.startsWith(PAT_PREFIX)) return tryResolvePat(token, extra);   // wm_pat_
  if (token.startsWith(WMA_PREFIX)) return tryResolveWma(token);          // wma_
  if (token.startsWith(WMEC_PREFIX)) return tryResolveWmec(token);        // wmec_
  return { status: 'unknown' };
}
```
PAT 强制绑定默认项目 + membership 校验；WMA 按 registration.tools 限定；WMEC 外部接入方应用身份。鉴权后写 `toolProxySessionStore` 到 Redis。

### 主题3 设计要点
1. JWT + Redis 黑名单，secret 强制 ≥32 字符。
2. 三路 token 提取适配代理链路与 WebSocket 升级。
3. 双轨权限：静态矩阵（编译期）+ 动态 RBAC（DB + Redis 缓存 300s）。
4. 三类 preHandler：jwtAuth → createProjectPermissionGuard / createPermissionMiddleware，系统管理员旁路。
5. MCP 多 token 体系（PAT/WMA/WMEC），Token Broker 统一路由，PAT 强制绑定项目。
6. token 不存明文，只存 SHA-256 hash。
