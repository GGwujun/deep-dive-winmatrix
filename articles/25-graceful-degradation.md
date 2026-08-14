# 优雅降级：当 Redis、ES、BullMQ、LLM 分别挂掉时系统怎么活

> 这是《WinMatrix 开发经验文集》第 25 篇。生产系统最大的考验不是"一切正常时跑得多快"，而是"某个部件挂了之后还能不能活"。AI 平台尤其脆弱——它依赖的基础设施比传统后端多得多：Redis 做缓存和锁、ES 做向量检索、BullMQ 做异步、LLM API 做推理、K8s Pod 做沙箱。任何一个挂了，系统该怎么响应？这篇从 WinMatrix 多个模块里提炼降级策略，看清楚每一种挂法的应对套路。

先看一个真实的运维噩梦清单。WinMatrix 一次完整对话链路涉及的基础设施：

```
用户请求
  ├── Fastify API → 鉴权（JWT 黑名单查 Redis）
  ├── 决策引擎 → 语义缓存（Redis）→ 路由规则
  ├── Turn 执行 → LLM 调用（外部 API）
  ├── 工具调用 → RAG 检索（ES）→ 记忆检索（ES + PG）
  ├── 异步任务 → BullMQ（Redis）
  └── 持久化 → PG
```

链路上任何一个环节都可能挂。Redis 挂了，JWT 校验怎么办？ES 挂了，RAG 检索怎么办？LLM API 超时，对话怎么办？BullMQ 连接断了，异步任务怎么办？

如果每个挂法都让整个系统崩，这个系统根本没法上生产。**优雅降级（graceful degradation）的本质是：承认每个部件都会挂，提前为每一种挂法准备好"次优但可用"的 fallback。** 这篇讲 WinMatrix 里 5 类降级策略。

---

## 策略一：Redis 挂了走 L1——多层缓存兜底

Redis 是 WinMatrix 里最重的基础设施之一：配置缓存、JWT 黑名单、语义缓存、分布式锁全靠它。Redis 挂了是不是整个系统就瘫了？不是，因为有 L1 内存缓存兜底。

WinMatrix 的配置缓存是三级结构（`src/infrastructure/cache/multiLevelCache.ts`，337 行）：

```
L1: 内存 Map（Map<string, CacheEntry>，TTL CONFIG_MEMORY_TTL 默认 1h）
   ↓ miss
L2: 文件缓存（FileCache）
   ↓ miss
L3: Redis（RedisCache，TTL CONFIG_REDIS_TTL 默认 86400s）
   ↓ miss
回源 DB
```

读取顺序是 L1 → L2 → L3 → DB。**关键设计：Redis 是 L3，不是唯一数据源。** 当 Redis 挂了，L1 内存还在，系统继续用内存里的配置跑；只有 L1 miss 且需要回源时，才会感受到 Redis 不在——那时直接走 DB。

这就把"Redis 挂了"的影响从"系统瘫痪"降级成"缓存命中率下降、DB 压力上升"。配置这种读多写少的数据，L1 内存命中率本来就很高，Redis 挂掉的影响微乎其微。

业务实体缓存（`EntityCache.ts`）是两级：L1 内存 + Redis，5 个 scope（de/sa/srb/ac/wst，各 15-30min TTL）。同样的思路——Redis 是加速层，不是必需层。

**这个模式叫"缓存分层 + 内存兜底"。** 要点：

- **缓存不是"Redis = 缓存"这种一一对应**，而是多级的，Redis 只是其中一层。
- **L1 内存是 Redis 挂掉时的生命线**，必须常驻、有合理 TTL。
- **Redis 挂掉时降级路径要清晰**：L1 命中 → 业务正常；L1 miss → 回源 DB（慢但可用）。

---

## 策略二：ES 挂了走 PG tsvector——双写降级

ES 在 WinMatrix 里负责向量检索（RAG 检索、记忆检索）。但 ES 是个相对脆弱的组件——内存吃得多、索引容易出问题、重启慢。WinMatrix 的做法是**双写 ES + PG，ES 挂了降级到 PG 的 tsvector 全文检索**。

记忆索引的写入就是双写（`MemoryIndexManager.ts:370-454`）：

```ts
private async indexContentCore(pathKey, content, ...): Promise<number> {
  const contentHash = hashContent(content);
  // ... hash 去重 ...
  const chunks = chunkText(content, pathKey, source, projectId, agentId);
  if (this.esAvailable) {                                    // ES 可用？
    try {
      await defaultMemoryVectorStore.addDocuments(chromaIds, documents, metadatas);
    } catch (esErr) {
      // ES 写失败不抛错，继续写 PG
      logger.warn(`[MemoryIndexManager] 写入 ES 失败，仅写入 PG: ${getErrorMsg(esErr)}`);
    }
  }
  await upsertMemoryFile(memoryFile);    // PG memory_files（必写）
  await upsertMemoryChunks(chunks);      // PG memory_chunks（必写，含 tsvector）
}
```

注意三个细节：

1. **`this.esAvailable` 是个标志位**，ES 连不上时置 false，后续直接跳过 ES 写入，省去每次都试错的成本。
2. **ES 写失败是 `try/catch` 吞掉的**，只 warn 日志，不抛错——业务继续，PG 写入照常完成。
3. **PG 是"必写"的**，memory_chunks 表带 `tsv`（tsvector）字段和 GIN 索引。ES 挂了，检索自动降级到 PG 的 tsvector BM25 检索。

`HybridSearch.ts` 的混合检索就是 PG tsvector（BM25）+ ES dense_vector（cosine）加权融合。ES 挂了，只剩 PG tsvector 这一路，**从"语义+关键词混合"降级成"纯关键词"**——召回质量下降，但检索功能还在。

**这个模式叫"双写 + 优雅降级"。** 要点：

- **关键数据双写**：一份给"最好的检索引擎"（ES），一份给"可靠的兜底"（PG）。
- **写失败要吞**：不能因为 ES 挂了导致整个写入链路失败，PG 写入是 SLA。
- **读时降级**：检索引擎可用时走它，不可用时走兜底，业务不中断。

---

## 策略三：BullMQ 断了自动重连——永不放弃的连接策略

BullMQ 是 Redis 上的异步任务队列。Redis 网络抖动会让 BullMQ 连接断开。WinMatrix 的策略是**连接分类 + 差异化重连**，每条连接的重连策略都不同。

Redis 连接矩阵有 6 条（`RedisConnectionManager.ts`，128 行）：

| 连接 | 用途 | 重连/超时策略 |
|------|------|--------------|
| shared | JWT 黑名单/分布式锁 | 标准 |
| bullmq-queue | 投递任务 | commandTimeout 30s |
| bullmq-worker | 消费任务 | **禁用 commandTimeout** |
| wecom-lazy | 企微长连接 | 状态机 uninit→ready/degraded，60s±15s jitter 重连 |
| wecom-pubsub | 企微 pub/sub | 标准 |
| redis-cache | 缓存 | `retryStrategy: Math.min(times*200,5000)` 永不放弃 |

每一条连接的重连策略都是精心设计的，看几个关键的：

**redis-cache 永不放弃**：`retryStrategy: Math.min(times*200, 5000)`——第 N 次重连等 `N*200ms`，封顶 5 秒。这个策略的意思是"无论断多少次，都持续重试，间隔逐步加到 5 秒封顶"。**缓存这种"有则加速、无则降级"的组件，断了就持续试，总有一次会连上。**

**bullmq-worker 禁用 commandTimeout**：这是个反直觉的设计。BullMQ worker 用 `BRPOP` 这种 blocking 命令长时间阻塞等待任务，如果设了 commandTimeout，正常的 blocking 会被误判成超时断开。所以 worker 连接明确不加 commandTimeout（核实报告 ch04-06）。**这是个细节坑：不是所有 Redis 连接都该设超时，blocking 命令的连接设了反而出问题。**

**wecom-lazy 带抖动的状态机**：企微长连接的重连是状态机驱动的，`uninit→ready/degraded`，重连间隔 `60s±15s jitter`。jitter（随机抖动）很重要——如果一个服务挂了，所有客户端都在固定 60 秒后同时重连，会把刚恢复的服务再次打挂（thundering herd）。加 ±15s 抖动让重连分散开。

**这个模式叫"差异化重连"。** 要点：

- **不同用途的连接重连策略不同**：缓存"永不放弃"，blocking 命令"禁超时"，外部服务"带 jitter"。
- **重连间隔要有 jitter**，防 thundering herd。
- **状态机驱动重连**比简单循环更可控（ready/degraded 状态可见）。

---

## 策略四：LLM 卡住有 watchdog 补偿——悬挂调用收敛

LLM 是 WinMatrix 最不可控的基础设施——它是外部 API，可能超时、半途失败、甚至"假装在跑实际没响应"。LLM 调用一旦悬挂，会拖垮整条对话链路。

WinMatrix 的做法是**悬挂检测 + 补偿收敛**（`infrastructure/scheduled/llmCallWatchdogSweeper.ts`，53 行）：

```ts
export const LLM_CALL_WATCHDOG_SWEEPER_TASK_NAME = 'system-llm-call-watchdog-sweeper';

function resolveSweepThresholdMs(): number {
  const hardMs = getConfig().llmCallHardTimeoutMs;
  return hardMs > 0 ? hardMs * 2 : 360_000;   // 2x 硬超时，或 6 分钟
}
```

这个 sweeper 是个定时任务（每 10 分钟跑一次，`scheduleType: 'interval', intervalMs: 600_000`），专门扫"悬挂的 LLM 调用"——`llm_call_start` 记了但 `llm_call_end` 永远没来的调用。

它做的事（参考第 8 篇可观测性）：

1. **扫超过阈值（2 倍硬超时或 6 分钟）还没结束的 LLM 调用**。
2. **补写一个 `llm_call_end`**，让 span 树收敛。
3. **级联 finalize**：把这个 LLM 调用关联的 `agent_run` 和 `scheduled_task_run` 从 running 状态收敛到终态（failed）。
4. **补发失败回执**：通过 `DanglingFailureReceiptPusher` 向用户补发一个"这次调用失败了"的消息，而不是让对话永远卡着。

这是"外部调用必有悬挂检测"的典型实现。**LLM 这种不可控的外部依赖，光靠调用方的 try/catch 不够——调用方可能整个进程都崩了，根本来不及 catch。必须有一个独立的、定时跑的 sweeper 来兜底。**

这个模式叫"悬挂检测 + 补偿收敛"。要点：

- **外部调用的 start 和 end 必须可观测**（参考 LLM Span 遥测契约），否则 sweeper 无从扫起。
- **阈值要合理**：2 倍硬超时是经验值——小于硬超时会误杀正常慢调用，太大又会让用户等太久。
- **级联 finalize 比单点修复重要**：一个悬挂的 LLM 调用会污染整个 agent_run，必须级联收敛。
- **要补发用户回执**：技术上收敛了不够，用户体验上要告诉用户"失败了"，不能让对话永远转圈。

---

## 策略五：Prisma 连接池错误自愈——重建 + 重放

数据库连接池是另一个容易出问题的环节。PgBouncer、PG 重启、网络抖动都可能让 Prisma 的连接失效。WinMatrix 的 Prisma Client 封装了完整的自愈机制（`src/infrastructure/persistence/prisma/client.ts`，487 行）。

**可恢复错误识别**（`isRecoverablePrismaPoolError`，行 223-236）：识别 9 种可恢复错误模式。遇到这些错误不是直接抛给用户，而是触发重建。

**single-flight 重建**（`rebuildPrismaResources`，行 193-221）：

```ts
async function rebuildPrismaResources(reason: unknown): Promise<void> {
  if (globalForPrisma.prismaRebuildInFlight) {
    return globalForPrisma.prismaRebuildInFlight;   // 已有重建在进行，复用
  }
  globalForPrisma.prismaRebuildInFlight = (async () => {
    const previousResources = getRuntimeResources();
    const nextResources = createPrismaResources();   // 重建 pg.Pool + PrismaClient
    runtimeResources = nextResources;
    publishPrismaResources(nextResources);
    await closePrismaResources(previousResources);   // 关旧资源
  })();
  try { await globalForPrisma.prismaRebuildInFlight; }
  finally { globalForPrisma.prismaRebuildInFlight = undefined; }
}
```

**single-flight 是关键**：并发请求可能同时遇到连接池错误，如果每个都触发重建，会重建 N 次。single-flight 保证同一时刻只有一个重建在进行，其他并发请求等同一个 Promise。

**只读重放**（`withPrismaRecovery`，行 305-332）：重建后，对**只读方法**自动重放一次（`findMany/findFirst/aggregate/count/$queryRaw` 这 9 种）。写操作不重放——因为不知道前面那次是成功还是失败，重放可能导致重复写入。

```
请求 → Prisma 调用 → 连接池错误
                      ├── 是只读方法？
                      │     ├── 是 → 重建 → 重放一次 → 成功则返回
                      │     └── 否 → 抛错（让业务层处理）
                      └── 不可恢复错误 → 直接抛错
```

**这个模式叫"识别可恢复错误 + single-flight 重建 + 只读重放"。** 要点：

- **不是所有错误都重建**，要先识别"可恢复"模式（连接超时、连接断开等），业务错误（唯一约束冲突等）不该重建。
- **重建必须 single-flight**，防并发风暴。
- **只读可重放，写操作不可重放**——这是幂等性约束（参考上一篇）。

---

## 降级的本质：分层假设其他层会失败

把上面 5 类降级放一起看，会发现一个共同的心智模型：**每一层都假设其他层会失败，提前准备好 fallback。**

```
请求进来
  ├── Redis 挂了？→ L1 内存兜底（配置/实体缓存）
  ├── ES 挂了？→ PG tsvector 兜底（RAG/记忆检索）
  ├── BullMQ 断了？→ 差异化重连（永不放弃/jitter 分散）
  ├── LLM 卡了？→ watchdog 补偿（悬挂检测 + 级联 finalize）
  └── PG 连接池错了？→ 重建 + 只读重放（single-flight）
```

每一类挂法都有明确的降级路径，而且降级后的系统行为是可预期的：

| 部件挂了 | 降级到 | 用户感知 |
|---------|-------|---------|
| Redis | L1 内存 + DB 回源 | 变慢，但功能正常 |
| ES | PG tsvector | 检索质量下降（无语义），但有结果 |
| BullMQ | 自动重连 | 短暂延迟，任务不丢 |
| LLM | watchdog 补收敛 | 失败回执，不无限转圈 |
| PG 连接池 | 重建 + 重放 | 用户无感（只读自动重放） |

**优雅降级的目标不是"永不失败"，而是"失败时可预期、可恢复、不雪崩"。** 用户能接受的远比我们想象的多——慢一点可以接受，质量下降可以接受，甚至明确的失败回执也能接受。不能接受的是"永远转圈""雪崩""数据不一致"。

---

## 一个容易忽视的点：fail-closed vs fail-open

降级策略里有个隐含的选择：**当不确定时，是放行（fail-open）还是拒绝（fail-closed）？**

WinMatrix 里有两种选择的例子：

**fail-open 的例子**：JWT 黑名单查询，Redis 出错时返回 false（放行）：

```ts
// JwtService.ts（核实报告 ch04-06）
// isTokenRevoked：Redis 出错时返回 false 放行（可用性优先）
```

理由：Redis 挂掉是基础设施问题，不该让所有用户登不出去。放行的风险是有极少量已注销 token 短暂有效，但这个风险可接受。

**fail-closed 的例子**：技能凭证解析，缺失就失败（第 13 章 L3 readiness）：

```ts
// SkillReadinessGate（核实报告 ch13-17）
// manifest 声明 requiresCredentials 时，canonical 字段任一缺失即失败
// 绝不用原始 projectId fallback，不按名称/关键词推断
```

理由：技能凭证涉及敏感资源访问，不确定时必须拒绝，宁可不用这个技能也不能用错凭证。

**选择 fail-open 还是 fail-closed 的标准是"失败的代价"**：

- 失败代价小（用户体验、可恢复）→ fail-open，保可用性。
- 失败代价大（安全、数据不一致、不可恢复）→ fail-closed，保正确性。

这个选择没有统一答案，每个降级点都要单独想清楚。但有一条铁律：**安全相关的降级必须 fail-closed。** 第 28 篇讲安全护栏时会专门展开。

---

## 给后来者的几条总结

1. **缓存分层而不是单一 Redis**。L1 内存 + L2 文件 + L3 Redis + DB 回源，Redis 挂了走 L1，L1 miss 走 DB。配置和实体缓存都该这么设计。
2. **关键数据双写，写失败要吞**。ES + PG 双写，ES 挂了降级到 PG tsvector。检索引擎是加速层不是必需层。
3. **连接重连差异化**。缓存"永不放弃"，blocking 命令"禁超时"，外部服务"带 jitter 防 thundering herd"。
4. **外部调用必有悬挂检测**。LLM 这种不可控依赖，光靠调用方 try/catch 不够，要有独立 sweeper 定时扫悬挂 + 级联 finalize + 补发回执。
5. **连接池错误要识别 + 重建 + 重放**。识别可恢复错误模式，single-flight 重建防风暴，只读重放不写重放。
6. **每一层都假设其他层会失败**。降级不是某个模块的事，是整个系统的心智模型。
7. **fail-open 还是 fail-closed 看"失败代价"**。可用性相关可 fail-open，安全/正确性相关必须 fail-closed。
8. **降级路径要可预期、可观测**。每个降级点都要有日志、指标，运维能看见"现在系统在降级模式跑"，而不是黑盒。

生产系统不是"跑通 demo"，而是"在所有部件都出过问题之后还能活"。WinMatrix 这 5 类降级不是设计阶段一次性想清楚的，是踩了无数次坑之后逐步长出来的。每次事故补一个降级点，几年下来系统才有了"抗故障"的肌肉。

---

> **下一篇**：[《并发控制三层模型：全局信号量、会话锁、任务租约》](./26-concurrency-control.md)——降级解决了"部件挂了怎么办"，但还有一种"软故障"——并发太多把 LLM 撑爆、同一会话并发执行互相覆盖、同一任务被重复处理。WinMatrix 用三层并发控制解决，各管一个粒度。
