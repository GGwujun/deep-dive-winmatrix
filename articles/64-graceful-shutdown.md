# 优雅停机：进程收到 SIGTERM 后要做完哪些事才能退

> 这是《WinMatrix 开发经验文集》第 64 篇。第 18 篇讲构建部署时提过"liveness 失败会重启 pod"，第 61 篇讲审计时提过"Shutdown 时 flush 所有 pending spans"。但停机这件事从来没作为主线讲过。这篇就补这个缺口：一个跑着 HTTP 服务、BullMQ worker、长连接、定时任务的进程，收到停机信号（SIGTERM/SIGINT）后，不能直接 exit——还有一堆资源要释放、状态要收敛、pending 要 flush。这个过程叫优雅停机（graceful shutdown）。它决定了"停机时数据丢不丢、重启后能不能干净恢复"。WinMatrix 的做法是分阶段停机，每阶段带超时兜底，最终优雅退出。这是个横跨进程管理、连接释放、队列清理、审计完整性的跨界主题。

先讲不优雅停机会怎样。

**场景 1：正在处理的请求被截断。** 进程收到 SIGTERM 直接 exit，正在处理的 50 个 HTTP 请求全部断掉。用户看到"连接被重置"，数据可能写了一半——比如配置更新写了 newValue 但没写 ConfigAuditLog，审计断了。

**场景 2：BullMQ job 被晾在 active 状态。** worker 正在处理一个定时 Agent 任务，跑到一半调 LLM 呢，进程退了。job 留在 active 状态，没人消费也没释放。要么等租约超时（可能很久），要么靠启动时的 reconcile 补（第 37 篇讲过孤儿任务回收）。这期间这个 job 像消失了。

**场景 3：占着的租约没释放。** scheduledSyncLeader 抢了个 Redis 分布式锁当 leader，进程退了但没释放锁。别的实例要等锁 TTL（120 秒）过期才能接手——这 120 秒里定时任务没人调度。

**场景 4：审计日志的 pending span 丢了。** 第 61 篇讲过 apiAuditLog 用 PendingSpan Registry 记录请求。进程暴力退出，Registry 里没定稿的 span 全丢——这批请求的审计永远空了。

这四个场景的共同点：**进程退得太快，留下一堆"半截子"状态**。优雅停机要做的就是"退之前把这些半截子收敛完"。WinMatrix 的停机序列分阶段、带超时、有强制兜底，是个值得细讲的工程范式。

---

## 停机的触发：SIGTERM 与两次信号机制

先说停机怎么触发。生产环境通常是 k8s/docker 发 SIGTERM（pod 缩容、滚动更新、手动 delete）。开发环境是 Ctrl-C 发 SIGINT。两个信号都应该触发优雅停机。

WinMatrix 的入口是 `shutdownApi()`（`startup/api.ts:588-610`）：

```ts
/** API shutdown：序 1–3 → 跳过 4–12 → 13–19 → G1–G3 */
export async function shutdownApi(): Promise<void> {
  if (isExiting()) return;                    // 已经在退，幂等
  processState.set('shutting_down');           // 标记状态（readyz 会据此摘流）
  updateProcessStateMetric('shutting_down');
  armForceExitOnSecondSignal();                // 武装"第二次信号强制退"

  try {
    await shutdownApiPrefix(apiServer);        // 阶段 1-3：HTTP + 审计 flush
    await sharedTeardown({                     // 阶段 13-19：资源释放
      configDbListener,
      onConfigDbListenerStopped: clearConfigDbListener,
      includeMcp: !config.startup.skipMcp,
      includeNeo4j: true,
      includeExternalAgent: true,
    });
    logger.info('✅ API 主组件已停止，开始 graceful exit');
    await gracefulExit(0, { skipObservabilityStop: true });
  } catch (err) {
    logger.error({ err: getErrorMsg(err) }, '❌ API shutdown 阶段出错，转 graceful exit(1)');
    await gracefulExit(1, { skipObservabilityStop: true });   // 出错也优雅退（exit 1）
  }
}
```

几个设计要点：

**幂等**。`isExiting()` 检查——如果已经在退，直接返回。防止 SIGTERM 被多次触发时重复执行停机序列。

**状态标记**。`processState.set('shutting_down')`——这个状态会被 readiness 探针读到（第 65 篇会讲），探针返回 503，k8s 立刻把 pod 从 Service 摘掉，新流量不再进来。这是停机的第一步：**先拒绝新请求**。

**两次信号机制**。`armForceExitOnSecondSignal()`（`common.ts:411-420`）：

```ts
export function armForceExitOnSecondSignal(): void {
  if (forceExitArmed) return;
  forceExitArmed = true;
  const handler = (sig: NodeJS.Signals): void => {
    logger.warn(`[Exit] 第二次收到 ${sig}，立即强制退出（pid=${process.pid}）`);
    process.exit(1);
  };
  process.once('SIGINT', () => handler('SIGINT'));
  process.once('SIGTERM', () => handler('SIGTERM'));
}
```

第一次 SIGTERM 触发优雅停机（慢慢收敛）。如果运维着急，再发一次 SIGTERM（或 SIGINT），**立即 process.exit(1) 强制退出**。这是个"逃生舱"——优雅停机卡住时（比如某个 close 挂死），第二次信号能强制脱身，不用等 k8s 的 terminationGracePeriodSeconds 超时 kill。

**出错也优雅退**。catch 块里调 `gracefulExit(1)`，而不是直接 `process.exit(1)`。即使停机过程出错，也走统一的退出流程（gracefulExit 会做最后的清理）。exit code 1 表示异常退出，k8s 会认为这是"非正常退出"。

注释里的"序 1–3 → 跳过 4–12 → 13–19"是什么意思？这是 API 进程和 scheduled/rag 进程停机序列的差异——

```
全量停机序（all-in-one / scheduled / rag）
  1-3   HTTP prefix + 审计 flush（API 才有）
  4-12  worker 清理（18 个 BullMQ worker + scanner，scheduled/rag 才有）
  13-19 共享清理（Redis/config/cache/MCP/Neo4j/externalAgent）
  G1-G3 gracefulExit（DB pool 关闭、observability stop、process.exit）

API 进程停机：序 1-3 → 跳过 4-12 → 13-19 → G1-G3
  （API 不跑 BullMQ worker，跳过 worker 清理）
```

API 进程不跑 worker（第 18 篇讲过四进程分离），所以跳过序 4-12。scheduled/rag 进程反过来——不对外服务 HTTP，跳过序 1-3，但要清理 worker。all-in-one 模式全跑。**不同进程角色的停机序列不同，但共享同一套阶段定义**。

---

## 序 1-3：先关 HTTP，再 flush 审计

API 进程停机的第一阶段是 `shutdownApiPrefix`（`shutdown/apiPrefix.ts`，13 行）：

```ts
/** Shutdown steps 1–3: HTTP server + API audit / observability prefix. */
export async function shutdownApiPrefix(server: FastifyInstance): Promise<void> {
  await safeStep('server.close', () => server.close(), 5_000);
  {
    const { flushPendingSpans } = await import('@/interface/middleware/apiAuditLog.js');
    flushPendingSpans('server_shutdown');
  }
  await safeStep('mainObservabilityLogger.stop', () => mainObservabilityLogger.stop(), 3_000);
}
```

三步顺序不能乱：

**第 1 步：server.close()（超时 5 秒）**。Fastify 的 `server.close()` 会**停止接受新连接**，但**等已接收的请求处理完**。这很关键——新请求不进了（配合 readiness 摘流），但正在处理的请求给时间跑完。5 秒超时兜底，防止某个请求 hang 住卡死整个停机。

**第 2 步：flushPendingSpans（无超时，内存操作很快）**。server.close 等请求处理完后，Registry 里可能还有没定稿的 pending span（比如响应刚发完但 onResponse 还没跑）。这时 `flushPendingSpans('server_shutdown')` 把它们全部标记成"server_shutdown 中断"落 ES。这保证审计日志的完整性——停机时所有未完成请求都有"被停机打断"的记录（第 61 篇详讲过）。

**为什么必须在 server.close 之后？** 因为 server.close 之前还有新请求进来，flush 了还会有新的。server.close 之后才保证不再有新 span 产生，flush 才是完整的。

**第 3 步：observabilityLogger.stop（超时 3 秒）**。观测日志的 buffer flush 和 writer 关闭。保证已产生的日志都落盘了。

这一阶段的核心：**先拒绝新请求、等已有请求完、flush 审计、关日志**。这是停机最关键的一步——大部分"停机丢数据"的事故都是这步没做好。

---

## 序 4-12：worker 清理（scheduled/rag 进程）

scheduled 和 rag 进程跑 BullMQ worker，停机时要按"依赖顺序"清理。这部分由 `shutdownScheduledWorkers`（`shutdown/scheduledWorkers.ts`，82 行）负责。

注意这个函数里每个 `closeXxxWorker` 都包在 `safeStep` 里带超时：

```ts
export async function shutdownScheduledWorkers(options?): Promise<void> {
  await safeStep('stopKickoffRecoveryScanner', stopKickoffRecoveryScanner, 2_000);
  await safeStep('stopWorkstationTaskReconcileScanner', stopWorkstationTaskReconcileScanner, 2_000);
  await safeStep('stopWorkstationTaskQueueDispatcher', stopWorkstationTaskQueueDispatcher, 2_000);
  await safeStep('closeKickoffJobWorker', closeKickoffJobWorker, 3_000);
  await safeStep('closeCrossAgentTriggerWorker', closeCrossAgentTriggerWorker, 3_000);
  // ... 共 20+ 个 safeStep
  await safeStep('closeScheduledTaskWorker', closeScheduledTaskWorker, 30_000);   // 主队列给 30 秒
  // ...
  {
    stopScheduledSyncLeaderRenewal();
    await releaseScheduledSyncLeader();           // 释放 leader 锁
  }
}
```

几个设计要点：

**顺序有讲究**。先停 scanner（扫描器，主动 poll 的）→ 再停 worker（被动消费的）→ 最后释放 leader 锁。因为 scanner 可能给 worker 投活，先停 scanner 避免"停了又来新活"。leader 锁最后释放——只要 leader 还在，定时任务的调度就还能收敛，等 worker 都停干净了再让出 leader 身份。

**超时逐个不同**。scanner 给 2 秒（轻量，扫一下就退），普通 worker 给 3-5 秒，scheduledTaskWorker 给 30 秒（主队列，可能在跑长任务）。**超时长的说明那个步骤值得等，超时短的是"差不多就该退了"**。

**safeStep 兜底**。`safeStep`（`common.ts:439-445`）的逻辑是"带超时执行，超了或错了就记日志跳过"：

```ts
export async function safeStep(label: string, fn: () => Promise<unknown>, ms: number): Promise<void> {
  try {
    await withTimeout(label, fn() as Promise<unknown>, ms);
  } catch (err) {
    logger.error({ err: getErrorMsg(err) }, `[Exit] 步骤失败：${label}`);
  }
}
```

**任何一步卡住或报错，不阻塞后续步骤**。这是停机序列的韧性——不能因为一个 worker 的 close 卡死，整个进程都退不出去。超时跳过，继续下一步，最坏情况是那个 worker 的 job 留 active 状态等启动时 reconcile（第 37 篇的孤儿任务回收兜底）。

**释放 leader 锁**（行 63-68）：

```ts
const { stopScheduledSyncLeaderRenewal, releaseScheduledSyncLeader } = await import(
  '@/infrastructure/scheduled/scheduledSyncLeader.js'
);
stopScheduledSyncLeaderRenewal();      // 停续租
await releaseScheduledSyncLeader();    // 主动释放锁
```

这两步很关键：先停续租（不再 renew fence），再主动 release。如果不主动释放，别的实例要等锁 TTL（120 秒）过期才能接手 leader。主动 release 让别的实例立刻能抢，停机对调度的影响从 120 秒降到接近 0。

---

## 序 13-19：共享资源清理

worker 清理完，剩下的是跨进程共享的资源——Redis、config listener、缓存、MCP、Neo4j、externalAgent。这部分由 `sharedTeardown`（`shutdown/sharedTeardown.ts`，53 行）负责，API/scheduled/rag 都会走：

```ts
export async function sharedTeardown(options: SharedTeardownOptions): Promise<void> {
  await safeStep('shutdownOnDemandCache', async () => {
    const { shutdownOnDemandCache } = await import('@/business/application/services/ScheduledTaskService.js');
    await shutdownOnDemandCache();
  }, 3_000);

  await safeStep('shutdownRedisConnections', shutdownRedisConnections, 5_000);

  if (options.configDbListener) {
    await safeStep('configDbListener.stop', () => options.configDbListener!.stop(), 2_000);
    options.onConfigDbListenerStopped?.();
  }

  try { configManager.close(); } catch (err) { logger.warn(...); }

  await safeStep('entityCache.close', () => entityCache.close(), 2_000);
  await safeStep('cacheInvalidationBus.close', () => cacheInvalidationBus.close(), 2_000);

  if (options.includeMcp) {
    await safeStep('mcpManager.shutdown', () => getMcpManager().shutdown(), 3_000);
  }
  if (options.includeNeo4j) {
    await safeStep('closeNeo4j', closeNeo4j, 3_000);
  }
  if (options.includeExternalAgent) {
    await safeStep('externalAgentModule.shutdown', () => shutdownExternalAgentModule(), 3_000);
  }
}
```

顺序的意义：

**先关 on-demand cache**（定时任务用的）→ **再关 Redis**（底层连接）。因为 on-demand cache 可能依赖 Redis，先关上层再关下层。

**ConfigDbListener.stop**：停止监听 PG LISTEN/NOTIFY。这很重要——停了之后即使 DB 有配置变更也不再触发重载（进程都要退了，重载没意义）。`onConfigDbListenerStopped` 回调清掉全局引用，防 HMR 复用旧实例。

**configManager / entityCache / cacheInvalidationBus**：内存里的缓存和事件总线关闭，flush 掉该落盘的。

**MCP / Neo4j / externalAgent**：外部连接按角色条件关闭。API 进程有 MCP（对外服务），scheduled 可能有 Neo4j（图谱抽取），externalAgent 连接器。`includeXxx` 选项按进程角色开关。

这一阶段的核心：**按依赖顺序关共享资源，上层先关下层后关**。

---

## G1-G3：gracefulExit 最终退出

所有资源清理完，最后是 `gracefulExit`（`common.ts:461+`）。这一步做最终的"底线清理"：

```
gracefulExit(code, options)
   │
   ├─ G1：标记 exiting=true（幂等），武装第二次信号
   ├─ G2：stopBackgroundMemoryJobsBeforeDb（等 transcriptSync idle）
   ├─ G3：prisma pool 关闭（poolClosed=true，readyz 据此返 503）
   │      observability stop（除非 skipObservabilityStop）
   └─ process.exit(code)
```

几个要点：

**stopBackgroundMemoryJobsBeforeDb**：等 transcriptSyncManager 跑完当前的同步任务（有个 `waitUntilIdle` + 超时）。因为 transcriptSync 往 DB 写转录，关 DB pool 前要等它写完，否则转录丢了。

**prisma pool 关闭**：DB 连接池关掉。关之前设 `poolClosed=true`——readyz 探针会读到这个标记返回 503（第 65 篇讲），保证 pool 关闭过程中不会有新请求试图用 DB。

**observability stop**：最终 flush 观测数据。API 进程在 shutdownApiPrefix 已经 stop 过了，所以这里 `skipObservabilityStop: true` 跳过；scheduled/rag 进程没走过那步，这里会 stop。

**process.exit(code)**：最终退出。code=0 正常，code=1 异常。k8s 据此判断 pod 是正常结束还是崩溃。

---

## safeStep：停机序列的韧性核心

整个停机序列，每个步骤都包在 `safeStep(label, fn, ms)` 里。这个函数是停机韧性的核心，值得单独看。

`safeStep` = `withTimeout` + try/catch（`common.ts:422-445`）：

```ts
export async function withTimeout<T>(label: string, p: Promise<T>, ms: number): Promise<T | undefined> {
  let timer: NodeJS.Timeout | null = null;
  try {
    return await Promise.race<Promise<T | undefined>>([
      p,
      new Promise<undefined>((resolve) => {
        timer = setTimeout(() => {
          logger.warn(`[Exit] 步骤超时 ${ms}ms 跳过：${label}`);   // 超时记日志
          resolve(undefined);                                      // 返回 undefined 不报错
        }, ms);
      }),
    ]);
  } finally {
    if (timer) clearTimeout(timer);
  }
}
```

设计要点：

**Promise.race 实现超时**。实际操作和定时器赛跑，谁先完成算谁。操作先完成→ clearTimeout，正常返回。定时器先响→ 操作被抛弃（注意：没法真的取消 Promise，只是不等它了），返回 undefined。

**超时不抛错，返回 undefined**。这样 safeStep 的 try/catch 不会抓到超时，只抓真正的异常。超时算"正常跳过"，异常算"异常跳过"，都继续下一步。

**每个步骤独立超时**。不同步骤超时不同（scanner 2s、worker 3-5s、主队列 30s），匹配各自的合理停机时间。不能一刀切——全给 30 秒的话，20 个步骤最坏要等 10 分钟；全给 2 秒的话，长任务被强行截断。

**为什么这么在意"不卡住"？** 因为停机是有总时限的。k8s 的 `terminationGracePeriodSeconds` 默认 30 秒，超了直接 SIGKILL。如果某个步骤卡住把整个停机序列卡在 30 秒以外，k8s 强杀，前面所有优雅停机的努力白费。safeStep 保证"每个步骤都有上限，一个卡住不影响其他"，让整个序列在 grace period 内跑完。

---

## 停机窗口期的任务怎么处理

讲完整序列，单独说说停机窗口期正在处理的任务怎么办。这是"优雅停机"最难的部分——既要让任务有机会跑完，又不能无限等。

**HTTP 请求**：server.close 等已有请求完，但最多等 5 秒。5 秒没跑完的，强行断（response 发不出去），审计里记 'server_shutdown' 中断。用户侧看到的是"请求失败，请重试"。

**BullMQ job**：worker.close 会让正在处理的 job 停下。BullMQ 的机制是——worker 停了，active 的 job 没被 acknowledge，会留在 active 状态。等下次有 worker 启动（或者别的实例的 worker），job 的 visibility timeout 到了会被重新投递。WinMatrix 还有启动时的 reconcile（reconcileStaleRunsOnBootstrap，第 37 篇）兜底，把滞留的 running 状态收敛。所以 BullMQ job 不会因为停机丢失，只是"延迟处理"。

**LLM 调用**：如果停机时正在调 LLM，AbortController 会被触发（随进程退出），LLM 请求中断。这个调用对应的 span 会留 pending，由 llmCallWatchdogSweeper（第 36 篇）在下次启动后补写 llm_call_end。

**租约/锁**：scheduledSyncLeader 主动释放（前面讲过），role_inbox/flow_instruction 的 claim lease 靠 TTL 过期自动释放（第 26 篇讲过租约机制）。停机时不主动释放也能靠 TTL 兜底，但主动释放更快。

**核心原则：停机不丢任务，靠的是"延迟重投 + 启动 reconcile"双保险**。停机时没跑完的，重启后或者别的实例接手。这跟第 23 篇幂等、第 37 篇孤儿回收是一套体系——终态收敛，running 永远有兜底。

---

## 停机全景：一张图

把整个停机序列画成图：

```
收到 SIGTERM/SIGINT
   │
   ├─ 幂等检查（isExiting）
   ├─ processState = 'shutting_down' → readiness 摘流
   ├─ armForceExitOnSecondSignal（第二次信号强制退）
   │
   ▼
【序 1-3】shutdownApiPrefix（API/all-in-one 才有）
   ├─ server.close（拒新连接，等已有请求，5s 超时）
   ├─ flushPendingSpans('server_shutdown')（审计补全）
   └─ observabilityLogger.stop（日志 flush）
   │
   ▼
【序 4-12】shutdownScheduledWorkers（scheduled/rag 才有）
   ├─ 停 scanner（2s 各）
   ├─ 停 worker（3-5s 各，主队列 30s）
   ├─ 停 outbox/coordinator/collector（2s 各）
   ├─ 释放 scheduledSyncLeader 锁（主动释放，不等 TTL）
   └─ 停 distillWorker/traceExtractWorker
   │
   ▼
【序 13-19】sharedTeardown（所有进程）
   ├─ shutdownOnDemandCache（3s）
   ├─ shutdownRedisConnections（5s）
   ├─ configDbListener.stop（2s）
   ├─ configManager.close + entityCache.close + cacheInvalidationBus.close
   └─ MCP / Neo4j / externalAgent 条件关闭（3s 各）
   │
   ▼
【G1-G3】gracefulExit
   ├─ 等 transcriptSync idle（写完转录）
   ├─ 关 prisma pool（poolClosed=true）
   ├─ observability stop（scheduled/rag）
   └─ process.exit(code)
```

每一步都包在 `safeStep(label, fn, ms)` 里，超时跳过、异常跳过、继续下一步。整个序列的目标：**在 k8s grace period（默认 30s，可配）内跑完，跑不完的被 SIGKILL 但尽量减少损失**。

---

## 给后来者的几条总结

1. **优雅停机是分阶段序列，不是直接 exit**。先摘流拒新请求、等已有请求完、flush 审计、按依赖顺序关资源、最终 exit。每阶段带超时兜底。
2. **第一步是标 shutting_down 让 readiness 摘流**。processState 改成 shutting_down，探针返 503，新流量不再进来。
3. **两次信号机制是逃生舱**。第一次触发优雅停机，第二次强制 process.exit(1)。防优雅停机卡死。
4. **HTTP 先 close 再 flush 审计**。server.close 等已有请求完（5s 超时），之后 flushPendingSpans 补全审计中断记录。顺序不能反。
5. **worker 按依赖顺序清理**。先停 scanner（不再投活）→ 再停 worker → 最后释放 leader 锁（让别人立刻接手）。
6. **主动释放锁，别等 TTL**。scheduledSyncLeader 主动 release，影响从 120s 降到 0；role_inbox 靠 TTL 兜底。
7. **safeStep 是韧性核心**。每步带超时、超时跳过、异常跳过、继续下一步。一个卡住不阻塞全序列。超时逐个配（scanner 2s、worker 3-5s、主队列 30s）。
8. **停机不丢任务，靠延迟重投 + 启动 reconcile 双保险**。BullMQ active job 等 visibility timeout 重新投递；启动时 reconcileStaleRunsOnBootstrap 收敛滞留 running 状态。
9. **不同进程角色停机序列不同**。API 跳过 worker 清理（序 4-12），scheduled/rag 跳过 HTTP（序 1-3），但共享序 13-19 和 G1-G3。
10. **出错也走 gracefulExit**。停机过程出错不 process.exit(1) 了事，走统一的 gracefulExit(code=1) 做最后清理。

优雅停机不性感，但它决定了"滚动更新时用户感知不到"、"重启后没有僵尸任务"、"审计日志不缺段"。把停机序列做扎实、每个步骤都带超时兜底，平台才有"敢随时重启"的底气。而"敢随时重启"，是 k8s 环境下高可用最基本的资格。

---

> **下一篇**：[《健康检查与就绪检查：liveness vs readiness 不是一回事》](./65-health-readiness.md)——停机讲完了，紧接着的问题是：k8s 怎么知道一个 pod 该不该接流量、该不该重启？靠两个探针。讲 liveness 和 readiness 的分工，以及"fatal 靠应用自退出而非探针"的设计。
