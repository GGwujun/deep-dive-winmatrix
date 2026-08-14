# 背压与限流：当 LLM 处理不过来时怎么办

> 这是《WinMatrix 开发经验文集》第 62 篇。第 26 篇讲并发控制时提过"全局信号量 scheduledAgentSem 防 LLM 过载"，但那是从"并发模型"角度讲的。这一篇换个视角——从"背压（backpressure）"角度看同一个问题：当请求速率超过系统处理能力时，怎么办。这是分布式系统和 AI 平台都要面对的跨界主题。传统后线的限流套路（QPS 限流）在 AI 平台不够用，因为瓶颈不在 CPU/内存，而在 LLM provider 的配额和并发。WinMatrix 的做法是三层限流 + 全局信号量 + 超载延迟重投，把"过载"从"丢弃/报错"变成"自动延迟、保任务不丢"。

先讲三种过载场景，感受 AI 平台过载的特殊性。

**场景 1：API 入口被刷爆。** 有人写个脚本狂调 `/api/v2/agents/chat`，每秒上百请求。Fastify worker 池打满，正常用户的请求排队进不去。这是传统后端也有的入口限流问题。

**场景 2：对话并发把内存打爆。** 同时 50 个用户在跟数字员工聊天，每个对话都启动一个 Turn 编排，内存里 50 个 Role 实例 + 50 套 prompt 上下文 + 50 个流式 emitter。内存爆了 OOM。

**场景 3：LLM provider 被打爆。** scheduled 队列里堆了 100 个定时 Agent 任务，worker 拼命消费，每个任务又触发多轮 LLM 调用。瞬间几百个并发请求打向 LLM API。provider 返回 429，严重的话暂时封号，**所有项目的所有用户**都受影响。

这三个场景对应三个完全不同的过载层面：**入口 QPS / 单实例对话并发 / 跨实例 LLM 全局并发**。WinMatrix 用三层限流 + 全局信号量分别应对，而且过载时不丢弃任务，而是延迟重投。这篇逐层拆。

---

## 第一层：入口 QPS 限流——@fastify/rate-limit

最外层是 HTTP 入口的 QPS 限流，防"脚本刷爆 API"。用 `@fastify/rate-limit` 插件（`startup/api.ts:147-167`，核实报告 ch07-08）：

```ts
await apiServer.register(rateLimit, {
  max: Number(process.env.RATE_LIMIT_MAX ?? 1000),
  timeWindow: process.env.RATE_LIMIT_WINDOW ?? '1 minute',
  keyGenerator: (request) => {
    // X-Forwarded-For 首段或 request.ip
    return realIp || request.ip;
  },
  allowList: (request, _key) => {
    const path = request.url.split('?')[0];
    return (
      path.startsWith('/assets/') ||
      path === '/health' ||
      path === '/readyz' ||
      path.startsWith('/winmatrix-ui/') ||
      /\.(js|css|ico|svg|woff2?|png|jpg|jpeg|gif|webp)(\?|$)/i.test(path)
    );
  },
});
```

几个设计要点：

**默认 1000/min，按 IP 限流**。key 是客户端 IP——从 `X-Forwarded-For` 首段取（过反向代理的真实 IP），取不到用 `request.ip`。这限的是"单个 IP 的请求速率"，防脚本刷。

**静态资源白名单**。`/assets/`、`/health`、`/readyz`、UI 静态文件这些高频但不消耗算力的请求，不走限流。否则前端加载一个页面（几十个静态资源）就把 IP 的配额耗光了。

**参数可配**。`RATE_LIMIT_MAX` 和 `RATE_LIMIT_WINDOW` 走环境变量，不同部署规模可调。小团队 1000/min 够，大平台可以调高。

**触发后 429**。超限的请求被 rateLimit 插件直接返回 429，不进业务逻辑。errorHandler 会透传 4xx 不包成 500（核实报告 ch07-08 主题 1 设计要点 6）。

这一层的特点：**粗粒度、入口拦截、429 拒绝**。它防的是"恶意刷接口"，不关心请求内容。被拒绝的请求直接失败，客户端要自己重试。

---

## 第二层：对话并发限流——agentChatLimiter

过了 QPS 限流的请求，进入对话处理。但 QPS 限流管不住"并发"——1000 个请求分散在一分钟里没问题，但如果同时涌进来 50 个对话，每个都要启动完整的 Agent 编排，单实例内存就扛不住。

第二层是 `agentChatLimiter`（`interface/api/agentChatLimiter.ts`，58 行，核实报告 ch07-08）：

```ts
/**
 * Agent 对话并发限制
 * 限制同时执行的 /chat 与 WebSocket 流式对话数
 * 通过环境变量 AGENT_CHAT_CONCURRENCY 配置，≤0 表示不限制
 */
const maxConcurrency = parseInt(process.env.AGENT_CHAT_CONCURRENCY || '0', 10);

class Limiter {
  private running = 0;
  private waitQueue: Array<() => void> = [];

  constructor(private readonly max: number) {
    if (this.max < 1) throw new Error('max must be >= 1');
  }

  async acquire(): Promise<void> {
    if (this.running < this.max) {
      this.running++;
      return;
    }
    return new Promise<void>((resolve) => {
      this.waitQueue.push(() => {   // 排队等
        this.running++;
        resolve();
      });
    });
  }

  release(): void {
    this.running--;
    if (this.waitQueue.length > 0 && this.running < this.max) {
      const next = this.waitQueue.shift();
      if (next) next();
    }
  }
}

const limiterInstance = maxConcurrency > 0 ? new Limiter(maxConcurrency) : null;

export const agentChatLimiter = {
  async acquire(): Promise<void> {
    if (limiterInstance) await limiterInstance.acquire();
    else await acquireNoLimit();
  },
  release(): void { /* ... */ },
};
```

这一层和 QPS 限流完全不同：

**限的是并发数，不是 QPS**。`AGENT_CHAT_CONCURRENCY` 控制的是"同时在跑的对话数"，不是"每分钟允许多少个"。50 个对话慢慢来，只要不同时超过并发上限，都不拦。

**超限不拒绝，而是排队**。`acquire()` 发现并发满了，返回一个 Promise，把请求挂到 `waitQueue` 里等。别人 `release()` 后才唤醒。这是个**背压**机制——过载时不报错不丢弃，而是让请求"排队等"，调用方阻塞但不失败。

**单实例内存限流**。这是个内存里的 `Limiter` 实例，不跨进程。V2 REST chat 和 WebSocket 流式对话共用（设计意图：同一实例的对话并发统一管）。跨实例的并发限制交给下一层（全局信号量）。

**≤0 不限制**。开发环境默认不配 `AGENT_CHAT_CONCURRENCY`，limiterInstance 为 null，直接通过。

这一层的特点：**中等粒度、单实例、排队背压**。它防的是"单实例内存/线程打爆"，过载时让请求等而不是失败。

---

## 第三层：会话串行——conversationRunLocks

再细一层是会话级锁，第 26 篇详讲过（`agentWebSocketState.ts`，52 行）：

```ts
// 会话运行锁：同一会话同时只允许一次 run
export const conversationRunLocks = new Map<string, boolean>();
export const conversationAbortControllers = new Map<string, AbortController>();
```

这一层不是"限流"，是"串行化"——同一会话的多个消息必须排队执行，防并发覆盖。它跟限流的关系是：限流管"总量"，会话锁管"同一会话的顺序"。第 26 篇讲过，这里不重复。

把前三层放一起看，是从粗到细的三级漏斗：

```
请求到来
   │
   ├─【第一层】@fastify/rate-limit（全局 QPS，按 IP）
   │     防脚本刷爆，超限 429 拒绝
   │     粒度：单 IP 每分钟请求数
   │
   ├─【第二层】agentChatLimiter（单实例对话并发）
   │     防内存/线程打爆，超限排队背压
   │     粒度：同时在跑的对话数
   │
   └─【第三层】conversationRunLocks（会话串行）
         防同会话并发覆盖，冲突排队
         粒度：单个会话
```

这三层都在 API 实例内。但 AI 平台最危险的过载不在 API 层，而在后台 LLM 任务——那是跨实例的全局瓶颈。

---

## 第四层：全局信号量——scheduledAgentSem

这一层是 AI 平台过载控制的核心。所有 LLM 密集型的后台任务（定时 Agent 任务、跨 Agent 触发器）都进 `scheduled-agent` 队列，worker 消费时受一个**跨实例全局信号量**约束（`ScheduledAgentConcurrencySemaphore.ts`，104 行，核实报告 ch23-29 主题 1）。

为什么必须全局？因为 **LLM 的瓶颈在 provider 侧，不是单机瓶颈**。你单个 Pod 自己并发 5 个没问题，但 10 个 Pod 各并发 5 个就是 50 个并发 LLM 调用，会把 provider 打爆。必须有个跨实例的全局闸门。

信号量用 Redis ZSET 实现（行 49-71）：

```ts
async tryAcquire(jobId: string): Promise<boolean> {
  const redis = await getRedisClient();
  const now = Date.now();
  const expiresAt = now + this.cfg.leaseTtlSeconds * 1000;
  const acquired = await redis.eval(
    `
    redis.call('ZREMRANGEBYSCORE', KEYS[1], '-inf', ARGV[1])   -- 先清过期租约
    local count = redis.call('ZCARD', KEYS[1])                  -- 看当前持有数
    if count >= tonumber(ARGV[2]) then
      return 0                                                  -- 满了，拒绝
    end
    redis.call('ZADD', KEYS[1], ARGV[3], ARGV[4])              -- 没满，占坑
    return 1
    `,
    1,
    LEASES_ZSET_KEY,
    now,
    this.cfg.globalMax,
    expiresAt,
    jobId,
  );
  return Number(acquired) === 1;
}
```

这是个**Lua 原子操作**，几件事一次做完：

1. `ZREMRANGEBYSCORE` 清掉 score（=过期时间）小于 now 的租约——这些是持有者崩溃没释放的僵尸租约。
2. `ZCARD` 看当前 ZSET 里有多少个有效租约。
3. 如果 < globalMax，`ZADD` 把自己的 jobId 加进去（score 是过期时间），返回 1。
4. 如果 >= globalMax，返回 0。

**为什么用 ZSET 而不是简单的 INCR/DECR？** 因为租约要带过期——持有者可能崩溃不释放。ZSET 的 score 存过期时间，每次 acquire 前先清过期的，自然实现"僵尸租约自动回收"。简单计数器做不到这点（崩溃后计数永远减不回去）。

**容量来自配置**（行 96-103）：

```ts
export function getScheduledAgentSemFromEnv(): ScheduledAgentConcurrencySemaphore {
  if (_instance) return _instance;
  const globalMax = config.scheduled.globalMaxParallel;
  const timeoutMs = config.scheduled.agentJobTimeoutMs;
  const leaseTtlSeconds = Math.ceil(timeoutMs / 1000) + 60;   // 超时 + 60s buffer
  _instance = new ScheduledAgentConcurrencySemaphore({ globalMax, leaseTtlSeconds });
  return _instance;
}
```

租约 TTL = 任务超时时间 + 60s buffer。一个任务跑超时了还没结束，租约自然过期，别的任务能顶上。这跟第 26 篇讲的"租约过期防僵尸任务"一脉相承。

**跨 worker 共享**。`crossAgentTriggerWorker`（跨 Agent 触发器）和 `scheduled-agent`（定时 Agent 任务）用的是同一个 `getScheduledAgentSemFromEnv()` 实例。两个不同入口的 LLM 任务加起来受全局上限约束。

这一层的特点：**跨实例、Redis ZSET + Lua 原子、租约自回收**。它防的是"LLM provider 被打爆"，这是 AI 平台最危险的过载。

---

## 超载不拒绝，而是 moveToDelayed 重投

信号量满了怎么办？这是 AI 平台过载控制跟传统限流最大的区别——**不拒绝、不丢弃，而是延迟重投**。

worker 消费 job 时的逻辑（`crossAgentTriggerWorker.ts:1261-1279`，核实报告 ch23-29）：

```ts
const acquired = await agentSem.tryAcquire(String(job.id));
if (!acquired) {
  // 信号量满 → 延迟重投
  await job.moveToDelayed(Date.now() + DEFAULT_SEM_DELAY_MS, token);
  // ...抛 DelayedError
}
try {
  // 执行任务...
} finally {
  await agentSem.release(String(job.id));
}
```

**`moveToDelayed + DelayedError`** 是 BullMQ 的机制：把 job 从 active 状态挪回 delayed 状态，指定一个延迟时间，到点后自动重新进队列等待消费。对调用方而言，job 没失败，只是"延后了"。

为什么要这样？因为 AI 平台的任务**通常不能丢**。一个定时 Agent 任务是"给客户发周报"，丢了这个任务，客户就收不到周报。429 拒绝（让调用方重试）也不合适——后台任务是系统自己调度的，没有"调用方"可以重试。延迟重投是唯一能"保任务不丢 + 自适应降速"的方案。

这个机制也用在 RAG 入库（`ragIngestWorker.ts:82-104`）：

```ts
// 内容 hash 不匹配（文件刚改过），延迟重投等稳定
await job.moveToDelayed(Date.now() + ragIngestConfig.hashRetryMs, token);
throw new DelayedError('content hash mismatch, recent mtime');

// 信号量满，延迟重投
await job.moveToDelayed(Date.now() + ragIngestConfig.semRetryMs, token);
throw new DelayedError('rag ingest sem cap');
```

RAG 入库也用同样的模式：hash 冲突延迟重投（等文件写完）、信号量满延迟重投。**DelayeError 是 BullMQ 的特殊错误类型，Job 实例不会因此进 failed 状态，而是自动重试。**

这一层的精髓：**过载时把"立即失败/丢弃"换成"延迟重投"**。系统自动降速（信号量满就延后），任务不丢（重投直到能跑），自适应调节（负载降下来后延后的任务自然消化）。

---

## 全景：从入口到 LLM 的四级背压链

把四层 + 重投放一起，是从入口到 LLM provider 的完整背压链：

```
请求/任务到来
   │
   ├─【入口】@fastify/rate-limit
   │     粗粒度防刷爆，超限 429 拒绝（调用方自重试）
   │
   ├─【对话】agentChatLimiter
   │     单实例并发上限，超限排队（背压不失败）
   │
   ├─【会话】conversationRunLocks
   │     会话串行化，冲突排队（防覆盖）
   │
   └─【LLM】scheduledAgentSem（跨实例全局信号量）
         Redis ZSET + Lua，防 provider 打爆
         超限 moveToDelayed + DelayedError（延迟重投不丢任务）
```

四层的特点对比：

| 层 | 粒度 | 机制 | 过载时 |
|----|------|------|------|
| rate-limit | 单 IP QPS | Fastify 插件 | 429 拒绝 |
| agentChatLimiter | 单实例对话并发 | 内存 Limiter | 排队背压 |
| conversationRunLocks | 单会话 | 内存 Map | 排队串行 |
| scheduledAgentSem | 全局 LLM 并发 | Redis ZSET + Lua | moveToDelayed 重投 |

**越往外越粗（IP/实例级）、失败语义越硬（拒绝）；越往里越细（会话/全局）、失败语义越软（排队/重投）。** 这是有意的分层——入口的恶意流量直接拒（省资源），内部的正常任务排队或重投（保任务）。

---

## 为什么 AI 平台过载控制跟传统后端不同

这个话题值得单独拎出来讲。传统后端的过载控制套路是"QPS 限流 + 熔断 + 降级"，瓶颈通常是 CPU/内存/DB 连接。AI 平台多了几个特殊性：

**第一，瓶颈在 provider 侧，不在自己机器。** 你自己的 Pod CPU 不满，但 LLM provider 已经 429 了。所以限流不能只看本机指标，要有跨实例的全局协调（scheduledAgentSem）。

**第二，任务不能丢。** 传统后端的请求"丢了就丢了"，用户刷新一下重来。AI 平台的后台任务（定时周报、知识库同步、技能蒸馏）是系统调度的，没有"用户刷新"。丢了就是真丢。所以过载要延迟重投，不能拒绝。

**第三，背压要跨层级传导。** LLM 过载 → 信号量满 → job 延迟重投 → 队列堆积 → 上游感知到延迟。这个背压链要让上游"知道下游慢了"，而不是上游继续猛灌。BullMQ 的 delayed 状态天然提供这个信号——延迟 job 多了，队列 dashboard 能看到。

**第四，LLM 调用又贵又慢。** 传统请求几百毫秒一个，过载了快速清。LLM 调用一个可能几十秒，过载时积压的任务消化极慢。所以限流要更保守（globalMaxParallel 通常配得比想象中小），宁可让任务排队也别把 provider 打爆。

这些特殊性决定了 AI 平台的过载控制要以**全局协调 + 延迟重投**为主，QPS 限流为辅。传统后端那套"熔断 + 降级 + 拒绝"在这里不够用。

---

## 一个细节：信号量租约的自回收

全局信号量有个容易忽略的细节——租约自回收。前面 tryAcquire 的 Lua 脚本第一步是 `ZREMRANGEBYSCORE` 清过期租约。但这只在每次 acquire 时触发。如果一段时间没新 job 来，过期的僵尸租约会赖着不走，占着额度。

WinMatrix 加了个定期 reconcile（行 20-44）：

```ts
async reconcileLeases(): Promise<number> {
  const redis = await getRedisClient();
  const removed = await redis.zremrangebyscore(LEASES_ZSET_KEY, '-inf', Date.now());
  if (removed > 0) {
    logger.warn({ removed }, '[ScheduledTask] sem expired leases reconciled');
  }
  return removed;
}

startReconcile(intervalMs = 60_000): void {
  if (this.reconcileTimer) return;
  this.reconcileTimer = setInterval(() => {
    void this.reconcileLeases();
  }, intervalMs);
  this.reconcileTimer.unref?.();   // 不阻止进程退出
}
```

每 60 秒扫一次，清掉过期租约。注意 `this.reconcileTimer.unref?.()`——这个 timer 不阻止进程退出，优雅停机时不会被它卡住（第 64 篇会讲 unref 对停机的意义）。

这个细节的价值：**即使 acquire 很少触发（队列空闲），僵尸租约也能被定期清理，信号量额度不会泄漏**。这是个"防缓慢泄漏"的保护——没有它，长时间运行的系统会因为零星崩溃累积出越来越多不释放的租约，最终 globalMax 被虚耗光。

---

## 给后来者的几条总结

1. **AI 平台过载分四层，不是一个问题**。入口 QPS（rate-limit）/ 单实例对话并发（agentChatLimiter）/ 会话串行（conversationRunLocks）/ 全局 LLM 并发（scheduledAgentSem），各管一个粒度。
2. **入口限流用 429 拒绝**。粗粒度防恶意刷爆，静态资源白名单，参数可配。
3. **对话并发用排队背压**。agentChatLimiter 是内存 Limiter，超限不失败而是排队等。
4. **会话级用串行锁**。conversationRunLocks 防同会话并发覆盖，是第 26 篇并发控制的第三层。
5. **全局 LLM 信号量是核心**。Redis ZSET + Lua 原子 acquire，跨实例共享，租约带 TTL 自回收。防的是 provider 侧过载。
6. **超载不拒绝，而是 moveToDelayed 重投**。DelayeError 让 job 自动延后，保任务不丢，系统自适应降速。
7. **租约定期 reconcile 防泄漏**。60 秒扫一次清过期租约，timer 用 unref 不卡停机。
8. **越往外越粗越硬（拒绝），越往里越细越软（排队/重投）**。入口恶意流量直接拒省资源，内部正常任务排队保任务。
9. **AI 平台过载控制跟传统后端不同**：瓶颈在 provider 侧、任务不能丢、背压要跨层传导、LLM 又贵又慢。以全局协调 + 延迟重投为主，QPS 限流为辅。

过载控制是 AI 平台上生产后的第一个真问题。demo 阶段单用户慢悠悠调，什么过载都没有；一上生产，脚本刷、并发涌、LLM 配额见底，四层同时爆。提前把这四层都想清楚、尤其把"延迟重投"这个 AI 特有的背压机制做扎实，系统才有"扛得住高峰"的底气。

---

> **下一篇**：[《版本化：技能/流程/镜像怎么管版本》](./63-versioning.md)——过载控制保证了"跑得动"，但跑的是什么版本？技能、流程、镜像如果没版本管理，"上次还能跑今天就不能了"就是常态。讲 WinMatrix 的不可变发布版 + checksum + digest 体系。
