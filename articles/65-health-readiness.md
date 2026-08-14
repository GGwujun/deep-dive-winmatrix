# 健康检查与就绪检查：liveness vs readiness 不是一回事

> 这是《WinMatrix 开发经验文集》第 65 篇，也是"跨界主题"段的最后一篇。第 18 篇讲部署时提过一句"liveness /health 决定重启，readiness /readyz 决定摘流"，第 64 篇讲停机时提到 readiness 返 503 触发摘流。但这两个探针从来没作为主线讲透。这篇就补这个缺口：k8s 环境下，liveness 和 readiness 是完全不同的两件事，搞混了要么"没事乱重启"要么"挂了还接流量"。WinMatrix 的做法是两个探针分开实现、检查不同东西、失败语义不同，再配合 worker 进程的 markBullmqReady 和"fatal 靠应用自退出"的策略，构成一套完整的"进程健康"体系。这是个横跨 k8s 编排、HTTP 接口、进程状态的跨界主题。

先讲两个探针搞混的典型事故。

**事故 1：把 liveness 配成检查 DB 连通性。** 有人觉得"DB 连不上就是不健康"，于是 liveness /health 里去 `SELECT 1`。结果 DB 重启的 30 秒里，所有 pod 的 liveness 失败，k8s 把所有 pod 重启了一遍。重启完还是要连 DB，还是连不上，又重启——重启循环。本来 30 秒能自愈的 DB 抖动，变成了全平台雪崩。

**事故 2：把 readiness 配得跟 liveness 一样宽松。** readiness 只检查"进程活着"，不检查"能不能接请求"。结果启动还没完（路由没注册、缓存没加载）就被标记 ready 接流量，请求进来 500。或者连接池满了，readiness 不检查这个，还继续接流量，请求排队超时。

**事故 3：worker 进程没有探针。** scheduled/rag 进程不对外服务 HTTP，有人忘了配探针。进程卡死了（死锁、OOM 边缘）k8s 不知道，一直挂着不重启，定时任务不跑了、RAG 不索引了。

WinMatrix 对这三个问题的应对是：**liveness 只查"进程活着"、readiness 查"能接请求"、worker 进程独立健康端口 + markBullmqReady、fatal 故障靠应用自退出**。核心原则：**临时故障靠 readiness 摘流自愈，真死了才靠 liveness 重启，两者绝不能混用**。这篇逐个拆。

---

## liveness /health：只查"进程活着"

先看 liveness 探针对应的 `/health` 端点（`startup/api.ts:289-298`）：

```ts
apiServer.get('/health', async (_request, reply) => {
  if (!startupComplete) {
    return reply.status(503).send({
      status: 'starting',
      message: 'Agent startup in progress',
      timestamp: nowIso(),
    });
  }
  return reply.send({ status: 'ok', timestamp: nowIso() });
});
```

注意这个端点检查什么：**只看 `startupComplete` 这一个布尔值**。启动完了就返回 200，没完就 503。

**它不检查 DB、不检查 Redis、不检查 LLM provider、不检查连接池。** 这是刻意的。liveness 的语义是"这个进程是不是活着、有没有死锁/卡死"。只要 HTTP server 能响应请求，就说明事件循环还转着、没死锁——活着。DB 连不上不代表进程死了，可能只是临时网络抖动。

`startupComplete` 在启动序列完成后置 true（`startup/api.ts:93` 的 `isStartupComplete()`）。k8s 的 livenessProbe 有个 `initialDelaySeconds: 90`（第 18 篇讲过），即 pod 启动后 90 秒内 liveness 失败不算（不算就不会重启），给启动留时间。90 秒后如果 startupComplete 还没 true，liveness 失败，k8s 重启 pod——这是"启动卡住了就重来"的兜底。

**liveness 失败的后果：重启 pod**。所以 liveness 必须保守——宁可漏报"不健康"（进程其实有点问题但还能响应），也不能误报"不健康"（进程其实没问题却被重启）。把 DB/Redis 检查塞进 liveness，就是把"外部依赖抖动"误判成"进程死了"，导致不必要的重启。

有个 `/health/detailed` 端点（行 300-334）查得更细（postgres、prismaPool、cache、config stats），但它**不用于探针**，是给人看的运维面板。k8s 探针只用 `/health` 这个极简版。

这一层的核心：**liveness 只回答"进程活着吗"，不回答"系统健康吗"**。后者是 readiness 的活。

---

## readiness /readyz：查"能不能接请求"

readiness 探针对应的 `/readyz` 端点（`startup/api.ts:336-383`）就复杂多了：

```ts
apiServer.get('/readyz', async (_request, reply) => {
  const readyzStart = Date.now();
  const observe = (status: string) => { observeReadyz(status, (Date.now() - readyzStart) / 1000); };

  // 1. 启动没完 → 不 ready
  if (!startupComplete) {
    observe('starting');
    return reply.status(503).send({ status: 'starting', timestamp: nowIso() });
  }

  // 2. 正在停机 → 不 ready（摘流）
  const currentState = processState.get();
  if (currentState !== 'running') {
    observe(currentState);
    return reply.status(503).send({ status: currentState, timestamp: nowIso() });
  }

  // 3. DB pool 关了 → 不 ready
  if (poolClosed) {
    observe('pool_closed');
    return reply.status(503).send({ status: 'pool_closed', timestamp: nowIso() });
  }

  // 4. 连接池排队太多 → 过载摘流
  const metrics = getPrismaPoolMetrics();
  const waitingLimit = Number(process.env.PRISMA_POOL_WAITING_LIMIT ?? 5);
  if (metrics.waiting > waitingLimit) {
    observe('overloaded');
    return reply.status(503).send({ status: 'overloaded', prismaPool: metrics, timestamp: nowIso() });
  }

  // 5. DB ping → 连不上摘流
  const ping = await readyzDbPingGate.check(() => prisma.$queryRaw`SELECT 1`);
  if (!ping.ok) {
    observe(ping.status);
    return reply.status(503).send({ status: ping.status, prismaPool: metrics, ...(ping.error && { pgError: ping.error }), timestamp: nowIso() });
  }

  // 6. 路由注册 degraded → 标记但放行
  const routeRegistryDegraded = routeRegistry.isDegraded();
  observe(readyzObserveStatus(routeRegistryDegraded));
  return reply.send(buildReadyzSuccessPayload(metrics, routeRegistryDegraded, nowIso()));
});
```

六个检查，每个失败都返回 503：

| 检查 | 失败状态 | 含义 |
|------|------|------|
| startupComplete | starting | 启动还没完，路由/缓存没就绪 |
| processState | shutting_down 等 | 正在停机，别再发流量 |
| poolClosed | pool_closed | DB pool 已关（停机中） |
| prismaPool.waiting > limit | overloaded | 连接池排队过多，过载 |
| DB SELECT 1 | ping 失败 | DB 连不上 |
| routeRegistry degraded | degraded（但放行） | 路由部分降级 |

**关键区别：readiness 失败不重启 pod，只是从 Service 摘掉流量**。这是 readiness 和 liveness 最大的不同。DB 连不上，readiness 返 503，k8s 把这个 pod 从 Service 的 endpoints 里摘掉——外部请求不再发给它，但 pod 本身不重启。等 DB 恢复了，readiness 重新返 200，k8s 自动把它加回 endpoints。整个过程 pod 不重启，对进程内部状态零影响。

这就解决了事故 1 的问题——DB 抖动靠 readiness 摘流自愈，不会触发 liveness 重启循环。

**`overloaded` 状态特别值得说**。prismaPool.waiting > 5 时 readiness 返 503。这是个"主动背压"——连接池排队说明这个 pod 处理不过来了，先别给它新流量，让别的 pod 分担。等它消化完排队，waiting 降下来，自动恢复。这跟第 62 篇讲的背压是一个思路：过载时摘流而不是硬扛。

**`readyzDbPingGate`** 是个带熔断的 DB ping 闸门。DB 连不上时不每次都 ping（避免雪上加霜），有退避机制。ping 成功后重置。这是个"保护 DB"的设计——DB 已经抖了，探针不能再狂打 `SELECT 1` 给它添乱。

**routeRegistry degraded 是"软降级"**——路由注册部分失败（比如某个插件加载失败），但不影响核心功能，readiness 返 200 但 payload 里标 degraded。运维能看到"这个 pod 有点问题但还能用"。这比"任何小问题都摘流"务实。

---

## liveness vs readiness：核心区别

把两个探针对比看：

```
                liveness /health          readiness /readyz
─────────────────────────────────────────────────────────────
检查什么        进程活着（startupComplete）  能接请求（多维度）
失败后果        重启 pod                    摘流（从 Service 摘掉）
initialDelay    90s                         60s
检查频率        低（避免开销）              较高（及时摘流）
DB 抖动时       不受影响（不查 DB）         返 503 摘流，自愈
进程死锁时      超时失败 → 重启             也会失败 → 摘流
适用场景        "没救了才重启"              "暂时不能接流量"
```

**initialDelay 不同是有讲究的**：readiness 60s 比 liveness 90s 短。意思是 pod 启动 60 秒后就开始问"能接请求吗"，能接就早点接；但 90 秒内即使 liveness 失败也不重启（给启动留余地）。这样"能接流量就尽快接"和"启动期不误杀"都照顾到。

**两个探针的分工本质**：

- **liveness 是"核武器"**——重启是不可逆的、有副作用的（内存状态丢失、连接断开、正在处理的请求中断）。所以 liveness 必须保守，只在"进程真的死了/卡死了"时才触发。
- **readiness 是"调节阀"**——摘流是可逆的、无副作用的（流量只是暂时不给这个 pod）。所以 readiness 可以激进，只要"暂时不适合接流量"就摘。

**把 readiness 的检查塞进 liveness，等于把"调节阀"变成了"核武器"**——临时故障（DB 重连、连接池满）被当成"进程死了"触发重启，重启完还是要面对同样的临时故障，陷入循环。这就是事故 1 的根因。

反过来，**把 liveness 配得太宽松（什么都不检查）也不行**——进程死锁了还认为"健康"，不重启，pod 一直挂着占资源不干活。所以 liveness 要检查"进程真的能响应"，readiness 要检查"响应的质量好不好"。两个维度，不能合并。

---

## worker 进程的健康端口

前面讲的都是 API 进程（对外 HTTP）。scheduled 和 rag 进程不对外服务 HTTP，没有 /health /readyz 端点。但它们也需要健康检查——worker 卡死了 k8s 要知道。

WinMatrix 的做法是给 worker 进程起一个独立的极简 HTTP 健康服务（`startup/workerHealth.ts`，86 行，核实报告 ch23-29）：

```ts
export type WorkerHealthRole = Extract<ProcessRole, 'scheduled' | 'rag'>;

export async function startWorkerHealthServer(role: WorkerHealthRole): Promise<void> {
  const port = getWorkerHealthPort();   // WORKER_HEALTH_PORT，默认 8402

  healthServer = http.createServer(async (req, res) => {
    if (req.method !== 'GET' || req.url !== '/healthz') {
      res.statusCode = 404;
      res.end();
      return;
    }

    const check = await checkDependencies();
    const body = JSON.stringify(check.ok ? { ok: true, role } : { ok: false, role, error: check.error });
    res.setHeader('Content-Type', 'application/json');
    res.statusCode = check.ok ? 200 : 503;
    res.end(body);
  });

  await new Promise<void>((resolve, reject) => {
    healthServer!.listen(port, '0.0.0.0', () => resolve());
    healthServer!.once('error', reject);
  });

  logger.info(`[WorkerHealth] ${role} health server listening on :${port}/healthz`);
}
```

这个服务跑在 8402 端口（docker-compose 里 `WORKER_HEALTH_PORT=8402`，第 18 篇讲过），只响应 `/healthz` 这一个路径。k8s 的探针指到这个端口。

`checkDependencies`（行 26-51）检查四样东西：

```ts
async function checkDependencies(): Promise<{ ok: boolean; error?: string }> {
  try {
    await prisma.$queryRaw`SELECT 1`;                    // 1. Postgres
  } catch (err) {
    return { ok: false, error: `postgres: ${getErrorMsg(err)}` };
  }

  try {
    const redis = await getRedisClient();
    const pong = await redis.ping();                      // 2. Redis
    if (pong !== 'PONG') return { ok: false, error: 'redis: unexpected ping response' };
  } catch (err) {
    return { ok: false, error: `redis: ${getErrorMsg(err)}` };
  }

  if (!bullmqReady) {                                     // 3. BullMQ ready 标记
    return { ok: false, error: 'bullmq: not ready' };
  }
  if (bullmqQueueConnection.status !== 'ready' ||         // 4. BullMQ 连接状态
      bullmqWorkerConnection.status !== 'ready') {
    return { ok: false, error: 'bullmq: connections not ready' };
  }

  return { ok: true };
}
```

**注意 worker 的 /healthz 检查 DB 和 Redis，但 API 的 /health 不检查。** 为什么不一致？因为 worker 进程的"活着"定义跟 API 不同——API 进程即使 DB 断了，HTTP 还能响应（返回错误），算"活着"；worker 进程 DB/Redis 断了，BullMQ 根本消费不了 job，进程活着也没意义。所以 worker 的健康检查更严，DB/Redis 不通就认为不健康。

**worker 进程通常 liveness 和 readiness 用同一个 /healthz**（因为它不对外服务，"能否接流量"没意义，只有"该不该重启"有意义）。这是个合理的简化——worker 没有"摘流"的概念，不健康就该重启。

---

## markBullmqReadyForHealth：启动完成标记

worker 健康检查里有个 `bullmqReady` 布尔值。这个值默认 false，什么时候变 true？靠 `markBullmqReadyForHealth()`（行 22-24）：

```ts
export function markBullmqReadyForHealth(): void {
  bullmqReady = true;
}
```

这个函数在 worker 启动序列里调用（`startup/scheduled.ts:71`、`startup/rag.ts:27`，核实报告 ch23-29）：

```ts
async function initScheduledWorkers(): Promise<void> {
  assertWorkstationCallbackEndpointConfigured();
  await initAgentStack({ includeMcp: true });
  await ensureBullmqConnectionsReady();
  markBullmqReadyForHealth();        // ← BullMQ 连接 ready 后才标记
  if (shouldRunScheduledTasks()) {
    // ... 开始启动 worker
  }
}
```

调用时机很讲究——`ensureBullmqConnectionsReady()` 之后。意思是 BullMQ 的 queue 和 worker 连接都 ready 了，才告诉健康检查"可以认为我 ready 了"。

**为什么需要这个标记？** 因为 BullMQ 连接的 ready 是异步的。进程启动后 HTTP 健康服务立刻就能响应（端口 listen 了），但 BullMQ 的 Redis 连接可能还在握手中。如果健康检查只看 HTTP 存活，会误认为"worker 已 ready"，但实际 BullMQ 还没连上，job 消费不了。`markBullmqReadyForHealth` 提供了一个显式的"业务 ready"信号，跟 HTTP 存活分开。

这是个"启动完成"的多层判定：

```
worker 启动 ready 判定
   ├─ HTTP /healthz 端口 listen（能响应）
   ├─ bullmqReady = true（markBullmqReadyForHealth 调过）
   ├─ BullMQ queue connection.status === 'ready'
   └─ BullMQ worker connection.status === 'ready'
```

四样全过，/healthz 才返 200。少一样，503。这保证"k8s 认为 worker ready 时，它是真的能消费 job 了"。

---

## fatal 故障：靠应用自退出，不靠探针

最后一类情况：进程遇到"致命但无法靠重启恢复"的故障。比如配置错到启动都过不了、DB schema 严重不匹配、启动断言失败。这种情况 liveness 探针发现不了——进程可能还在响应 /health（返回 200），但业务完全跑不了。

WinMatrix 的策略是：**应用自己知道没救了，主动 `process.exit(1)` 退出**，让 k8s 重启 pod，而不是假装还活着。

几个例子：

**进程角色守卫**（第 18 篇讲过，`processRole.ts`）：启动时如果 `WIN_PROCESS_ROLE` 不匹配，直接 `process.exit(1)`。配置错配这种故障，重启也没用（除非改配置），但至少 exit 让 k8s 知道"这个 pod 起不来"，可以告警/回滚，而不是挂着假装活着。

**启动断言**（核实报告 ch23-29 主题 6）：`assertCoreDiRegistrations()` 断言核心 DI 完成，adapter 数量必须 4 个（`adapterCount !== 4 抛错`）。少了说明注册出问题，fail-fast 直接崩。

**优雅停机的 exit**（第 64 篇讲过）：`gracefulExit(code)` 最终 `process.exit(code)`。停机序列出错时 exit(1)，让 k8s 知道是异常退出。

**为什么不用探针来做这个？** 因为探针是"事后发现"，而 fatal 故障最好是"主动退出"：

- 探针发现"进程不健康"需要等一个探测周期（可能几十秒），这段时间 pod 还挂着占资源。
- 主动 exit 是"立刻"——故障发生的瞬间就退，k8s 立刻重启或告警。
- 有些 fatal 故障，进程其实还能响应 /health（HTTP 还活着，只是业务死了），探针根本发现不了。

所以"fatal 靠应用自退出"是更主动的策略：**应用知道自己没救了就别硬撑**，让编排系统来决定下一步（重启、回滚、告警）。探针是给"不自知的故障"（死锁、OOM 边缘）兜底的，不是给"自知的 fatal"用的。

---

## 全景：四类健康判定

把四类健康判定放一起：

```
健康判定体系
   │
   ├─【API liveness /health】进程活着吗
   │     查：startupComplete
   │     失败：重启 pod（核武器，保守）
   │
   ├─【API readiness /readyz】能接请求吗
   │     查：startup / processState / poolClosed / 过载 / DB ping / 路由
   │     失败：摘流（调节阀，可激进）
   │
   ├─【Worker /healthz】worker 还能用吗
   │     查：Postgres + Redis + bullmqReady + 连接状态
   │     失败：重启 pod（worker 无摘流概念）
   │
   └─【应用自退出】fatal 且无法靠重启恢复
         触发：启动断言 / 角色守卫 / 停机序列错
         动作：process.exit(1) 主动退，不等探针
```

四类的分工：

| 类 | 触发方 | 检查内容 | 失败动作 |
|----|------|------|------|
| API liveness | k8s 探针 | 进程活着 | 重启 |
| API readiness | k8s 探针 | 能接请求 | 摘流 |
| Worker healthz | k8s 探针 | 依赖 + BullMQ | 重启 |
| 应用自退出 | 应用自己 | fatal 故障 | exit(1) |

**四者正交，覆盖不同健康维度**：探针管"不自知的故障"（k8s 从外部探测），自退出管"自知的 fatal"（应用从内部判断）。liveness 管"进程死了"，readiness 管"暂时不能接流量"。API 和 worker 的检查内容不同，因为它们的"健康"定义不同。

---

## 为什么 readiness 检查 DB 但 liveness 不检查

这是最容易困惑的一点，单独拎出来讲。

**readiness 检查 DB**（/readyz 里有 `SELECT 1`）：因为"能不能接请求"取决于"请求处理需要 DB，DB 连不上的话请求会失败"。所以 DB 不通 → 不该接流量 → readiness 返 503。这是正确的。

**liveness 不检查 DB**（/health 只看 startupComplete）：因为"进程活着吗"跟"DB 通不通"是两回事。进程活着（事件循环转着、能响应 HTTP），只是 DB 临时抖了。这时重启进程没用——重启完还是要连 DB，还是连不上。重启反而有害（丢失内存状态、中断正在处理的请求）。

**DB 抖动的正确恢复路径**：DB 重启 → readiness 检测到 `SELECT 1` 失败 → 返 503 → k8s 摘流 → 请求不再来 → DB 恢复 → readiness 检测到 `SELECT 1` 成功 → 返 200 → k8s 加回流量。整个过程 pod 不重启，对进程零影响。

**如果 liveness 也检查 DB 会怎样**：DB 重启 → liveness 检测到失败 → k8s 重启所有 pod → 重启中 pod 不接流量 → DB 恢复了但 pod 都在重启 → 全平台不可用 → pod 重启完 → 终于能接流量。本来 30 秒的 DB 抖动，被放大成几分钟的全平台宕机。

所以"readiness 查 DB、liveness 不查"不是不一致，是**基于"重启 vs 摘流"语义差异的精确设计**。readiness 检查的是"接流量需要的条件"（DB 通），liveness 检查的是"进程本身的状态"（活着）。两者检查的东西本来就该不同。

---

## 给后来者的几条总结

1. **liveness 和 readiness 是两回事，绝不能混用**。liveness 失败重启 pod（核武器），readiness 失败摘流（调节阀）。
2. **liveness 只查"进程活着"**。/health 只看 startupComplete，不查 DB/Redis。保守，宁可漏报不误报。
3. **readiness 查"能接请求"**。/readyz 查 startup/processState/poolClosed/过载/DB ping/路由。激进，暂时不适合接流量就摘。
4. **readiness 检查 DB，liveness 不检查**。DB 抖动靠 readiness 摘流自愈，不触发 liveness 重启循环。这是精确的语义设计。
5. **readiness 的 overloaded 状态是主动背压**。连接池排队过多时摘流，让别的 pod 分担，过载缓解后自动恢复。
6. **worker 进程用独立的 /healthz 端口**。不对外服务 HTTP 的进程也要有探针，检查 DB + Redis + BullMQ。
7. **markBullmqReadyForHealth 提供"业务 ready"信号**。HTTP 存活不等于 BullMQ ready，要显式标记。
8. **fatal 故障靠应用自退出，不靠探针**。启动断言失败、角色守卫不过、停机序列出错——主动 process.exit(1)，不等探针发现。探针管不自知的故障，自退出管自知的 fatal。
9. **liveness 和 readiness 的 initialDelay 要不同**。readiness 短（60s，尽快接流量），liveness 长（90s，启动期不误杀）。
10. **两个探针 + worker healthz + 自退出，四者正交**。覆盖"进程死了/暂时不能接流量/worker 失联/自知的 fatal"四个维度，缺一不可。

健康检查不复杂，但它是 k8s 环境下"高可用"的入门门槛。把 liveness 和 readiness 分清楚、worker 探针配齐、fatal 自退出做好，平台才有"探针绿的就是真的能用、探针红的就是真的该处理"的可信度。否则探针就是摆设——绿的其实不能接流量，红的其实不需要重启，最后大家都不信探针，手动 kubectl 查——那就失去自动化的意义了。

---

> **下一篇**：[《分层 import 门禁：check:agent-layers:strict 怎么守住六层》](./66-layer-import-gates.md)——健康检查讲完了，第三批"跨界主题"段（59-65）正式收官。下一篇进入第四批"延伸踩坑 + 工具链"，讲怎么用门禁脚本守住分层架构。
