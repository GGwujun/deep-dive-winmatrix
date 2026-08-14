# 审计日志：谁在什么时候对配置做了什么

> 这是《WinMatrix 开发经验文集》第 61 篇。前 40 篇里"审计"这个词零散出现过——第 28 篇安全护栏提了"操作审计"，第 58 篇配置即数据提了"配置入库 = 审计"。但没把审计日志作为一根线系统讲过。这篇就补这个缺口：一个企业级 AI 平台，审计日志要记什么、记在哪、怎么保证不漏记、记下来怎么用。WinMatrix 的做法是三层审计——配置变更审计、API 写操作审计、内核治理审计——加上配置变更触发热更新通知，构成一条"谁改了什么、改完生效了没"的完整可追溯链。

先说审计日志最容易出问题的三个场景。

**场景 1：配置改了但查不到谁改的。** 系统出了故障，怀疑是某个 agent_config 被改了。查 DB 看到当前值，但不知道是谁改的、什么时候改的、改之前是什么。于是只能挨个问"你动过没有"。这种"配置无溯源"的团队，故障复盘永远卡在"第一步"。

**场景 2：写操作出了问题但日志不全。** 某个 API 接口被刷了，产生了异常数据。查日志发现：成功请求记了，失败请求没记；或者请求记了，响应没记；或者客户端中途断开的请求压根没留痕。审计日志缺了关键场景，追溯就断了。

**场景 3：配置改了但没生效，没人知道。** 管理员在 DB 改了条工具策略，以为会自动生效，结果系统还在用旧缓存——因为没人触发配置变更通知。配置改了和配置生效之间，差了一次"notify"。

WinMatrix 对这三个场景的应对是三层审计 + 变更通知：**ConfigAuditLog（配置变更留痕）+ apiAuditLog（写操作全生命周期）+ KernelManagementAuditLog（内核治理动作）+ notifyConfigChange（变更触发热更新）**。核心原则：**配置变更必须有 before/after diff，写操作必须覆盖全生命周期（包括中断），变更必须触发通知让缓存失效**。这篇逐层拆。

---

## 第一层：ConfigAuditLog——配置变更留 before/after

最基础的审计是配置变更留痕。WinMatrix 把 agent_config、skill_artifact、tool_config、route_rule 这些核心配置都放 DB（第 58 篇"配置即数据"），配置改了必须记一笔。

载体是 `ConfigAuditLog` 表（schema.prisma:1844-1859）：

```prisma
model ConfigAuditLog {
  id           String   @id @default(cuid())
  configType   String   @map("config_type")      // 哪类配置
  configId     String   @map("config_id")        // 哪条配置
  action       String                              // create/update/delete
  oldValue     Json?    @map("old_value")        // 改之前
  newValue     Json?    @map("new_value")        // 改之后
  operatorId   String?  @map("operator_id")      // 谁改的
  operatorName String?  @map("operator_name")    // 名字
  reason       String?                             // 改的原因
  createdAt    DateTime @default(now()) @map("created_at") @db.Timestamptz(6)

  @@index([configType, configId])
  @@index([createdAt])
  @@map("config_audit_log")
}
```

几个设计要点：

**before/after 都记**。`oldValue` 和 `newValue` 都是 Json，存的是配置的完整快照。这意味着审计不只是"改了"，而是"从什么样改成什么样"——精确 diff。故障复盘时能直接还原配置演变过程。

**操作人留 id + name 双份**。为什么不只留 id？因为用户可能改名、可能注销。审计日志的 operatorName 是当时的快照，即使后来用户改名了或离开了，历史审计记录依然可读。

**reason 可选但有**。`reason` 字段鼓励操作者留下变更理由。合规要求严的场景（比如金融），reason 可以设成必填。

**按 configType + configId 索引**。查"某条 agent_config 的变更历史"，走 `WHERE configType='agent_config' AND configId='xxx'`，索引命中，快。按 createdAt 索引支持"最近 N 天的所有变更"。

这一层的核心价值：**配置的每一次修改都有完整的 before/after + 操作人 + 时间 + 理由**。故障复盘第一步（"谁改了什么"）永远能答出来。

---

## 第二层：apiAuditLog——写操作的全生命周期

配置审计记的是"管理面操作"，但用户面的 API 写操作（POST/PUT/PATCH/DELETE）也要审计。这层的实现是 `apiAuditLog` 中间件（`interface/middleware/apiAuditLog.ts`，663 行，核实报告 ch07-08 主题 1）。

这一层比配置审计复杂得多，因为 API 请求有各种"非正常结束"：客户端断开、超时、服务重启。审计必须全覆盖。文件头注释（行 1-13）把设计意图说得清楚：

```ts
/**
 * API 请求审计日志中间件
 * Fastify onRequest + onSend + onResponse + 事件监听，记录 POST/PUT/PATCH/DELETE 写操作到 ES。
 * 跳过 GET/HEAD/OPTIONS、AI 路径、健康检查、观测查询接口。
 *
 * 中断请求补录机制：
 * - Span Registry：请求进入时登记 PendingSpan
 * - 事件优先级：onResponse > close/error > sweep
 * - close + aborted 检测客户端取消（替代废弃的 aborted 事件）
 * - Sweep 定期清理超时/未完成 span
 * - Shutdown 时 flush 所有 pending spans
 */
```

核心机制是 **PendingSpan Registry + 多触发点 finalize**。每个请求进来时先登记一个 PendingSpan，然后多个时机都可能把它"定稿"落 ES：

```
请求生命周期审计
   │
   ├─ onRequest：登记 PendingSpan（startTime/method/path/...）
   │     监听 request.raw 的 close / error 事件
   │
   ├─ onResponse（正常完成）→ finalizeNormal（statusCode + responseBody）
   │     GET 成功 / 查询类 POST 成功 → completeWithoutLog（删 span 不记）
   │
   ├─ close + aborted（客户端断开）→ finalizeInterrupted('client_aborted', 499)
   │
   ├─ error（socket 错误）→ finalizeInterrupted('socket_error', 502)
   │
   ├─ Sweep 定时器（每 5s）→ 超时(60s) finalizeInterrupted('stale_timeout')
   │                          TTL 硬上限(150s) finalizeInterrupted('ttl_expired')
   │
   └─ Shutdown（进程停机）→ flushPendingSpans('server_shutdown')
```

这个设计解决的是"审计日志不漏记"。传统审计只记正常完成的请求，中途断的、超时的、服务重启时的全漏了。WinMatrix 用 PendingSpan Registry 兜住所有非正常结束：

**Registry 有硬上限**（行 27-39）：

```ts
const MAX_REGISTRY_SIZE = 10_000;        // 最多 1 万个 pending span
const SWEEP_INTERVAL_MS = 5_000;          // 每 5 秒扫一次
const TIMEOUT_THRESHOLD_MS = 60_000;      // 超时阈值 60 秒
const TTL_HARD_LIMIT_MS = 150_000;        // TTL 硬上限 150 秒
const MAX_SWEEP_PER_ROUND = 1_000;        // 每轮最多处理 1000 条
```

满了会淘汰最旧的未完成 span（`finalizeInterrupted('registry_evicted')`，行 460-467），还不行就拒绝登记（`droppedSpanCount++`，行 470-477）。这是个"防 Registry 内存爆炸"的保护——恶意刷请求也不能把审计 Registry 撑爆。

**中断请求补录**（行 597-618）：

```ts
// close 事件：检测客户端取消（替代废弃的 aborted）
request.raw.on('close', () => {
  setImmediate(() => {
    if (request.raw.aborted && !span.completed) {
      finalizeInterrupted(span, 'client_aborted', 499);
    }
  });
});
// error 事件：socket 错误
request.raw.on('error', () => {
  setImmediate(() => {
    if (!span.completed) finalizeInterrupted(span, 'socket_error', 502);
  });
});
```

注意用 `setImmediate` 延迟一拍——因为 close 事件触发时，onResponse 可能还没来得及跑。延迟到下一个 tick，让 onResponse 优先。事件优先级（注释行 9）："onResponse > close/error > sweep"——正常完成永远优先于中断判定。

**写操作才记，查询不记**（行 99-121）：

```ts
const SKIP_PATHS = ['/health', '/readyz', '/api/v1/logs', '/api/v1/traces',
                    '/api/v1/agents/chat/stream', '/api/v1/external-agents/connect'];
function isQueryPost(path: string): boolean {
  if (path === '/api/v1/tasks/status') return true;        // 查状态
  if (path === '/api/v1/agents/route') return true;        // 路由探测
  if (/\/tool-visibility\/preview/.test(path)) return true; // 预览
  return false;
}
```

GET/HEAD/OPTIONS 成功不记（行 649-655），某些 POST 本质是查询（`isQueryPost`）成功也不记——但**失败照记**。健康检查、日志自查询、WS 长连接这些高频但无审计价值的直接跳过。这样审计日志信号噪比高，不被探针请求淹没。

这一层的核心：**写操作全生命周期覆盖（正常/中断/超时/停机），查询不记噪音，Registry 有保护防撑爆**。

---

## 第三层：KernelManagementAuditLog——内核治理动作

第三层是 AI 平台特有的：**L3 内核治理动作的审计**。管理员对 AI 内核做的治理操作（策略变更、能力开关、角色配置覆盖），影响范围大，必须单独审计。

载体是 `KernelManagementAuditLog`（schema.prisma:3133-3152）：

```prisma
model KernelManagementAuditLog {
  auditId           String   @id @map("audit_id")
  kernelId          String   @map("kernel_id")
  actionId          String?  @map("action_id")
  policyKey         String?  @map("policy_key")
  actorId           String   @map("actor_id")
  actorRole         String?  @map("actor_role")
  reason            String?
  targetScope       String   @map("target_scope")
  targetId          String?  @map("target_id")
  changeDiffSummary String?  @map("change_diff_summary")
  result            String                              // success/failed/...
  traceId           String?  @map("trace_id")
  rollbackHint      String?  @map("rollback_hint")
  createdAt         DateTime @default(now()) @map("created_at") @db.Timestamptz(6)

  @@index([kernelId, createdAt(sort: Desc)])
  @@index([actorId, createdAt(sort: Desc)])
}
```

跟 ConfigAuditLog 比，这层多了几个治理专属字段：

**targetScope + targetId**：治理动作的"作用域"。内核治理可能影响某个角色、某个项目、某个技能——targetScope 标明作用域类型，targetId 标明具体对象。这比 ConfigAuditLog 的 configType/configId 更面向"治理语义"。

**result**：动作的结果（success/failed）。治理动作可能失败（比如校验不过、权限不足），失败也要记，因为"有人尝试改但被拒了"本身就是审计价值信息（可能是攻击尝试）。

**rollbackHint**：回滚提示。治理动作的变更 diff 太复杂时，可以留一条"怎么回滚"的提示。这是面向"故障快速恢复"的设计——审计不只是追溯，还要能快速 undo。

**traceId**：关联全链路追踪。一次治理动作可能触发一系列后续事件（配置重载、缓存失效、通知广播），traceId 把它们串起来。

**changeDiffSummary**：注意是"summary"不是完整 diff。这跟 ConfigAuditLog 的 oldValue/newValue 不同——内核治理的变更可能很大（比如整个策略树重写），全量存 diff 成本高且难读，所以存摘要。需要完整 diff 时，通过 traceId 去翻 ExecutionSpan 的事件流（第 8 篇）。

这一层的核心：**AI 内核治理是高风险操作，审计要带作用域、结果、回滚提示、全链路关联**。

---

## 变更触发通知：审计与生效的闭环

前面三层都是"记下来"。但审计还有个隐含功能：**变更触发通知，让配置真正生效**。光记下"改了"不够，还要让系统知道"该重载了"。

这是 `notifyConfigChange` 的职责（`infrastructure/persistence/database/notifyConfigChange.ts`，14 行，核实报告 ch04-06）：

```ts
export async function notifyConfigChange(payload: ConfigChangeNotifyPayload): Promise<void> {
  await prisma.$executeRaw`SELECT pg_notify('config_change', ${JSON.stringify(payload)})`;
}
```

配置变更流程通常这样：管理员改 DB → 写 ConfigAuditLog → `notifyConfigChange` 发 PG NOTIFY → `ConfigDbListener` 收到 → 500ms 防抖合并 → 触发对应缓存失效/重载。

ConfigDbListener 的接收逻辑（`ConfigDbListener.ts:241-271`，核实报告 ch04-06）：

```ts
private handleNotification(msg: pg.Notification): void {
  if (msg.channel !== 'config_change' || !msg.payload) return;
  if (this.notificationSuppressed) return;
  try {
    const payload: ConfigChangePayload = JSON.parse(msg.payload);
    const key = `${payload.type}:${payload.id}`;      // 去重 key
    this.pendingChanges.set(key, { ... });            // Map 去重
    this.scheduleDebounce();                          // 500ms 防抖
  } catch (error) { logger.warn(...); }
}
```

**去重 + 防抖**是关键。短时间内同一配置被改多次（比如批量更新），Map 按 `type:id` 去重，500ms 防抖合并成一次重载——避免缓存反复失效。

这一层把审计和生效闭环：**改配置 → 记 ConfigAuditLog → notify → listener 重载 → 新配置生效**。审计不只是"事后追溯"，还是"实时生效"的触发器。如果 notify 这环断了（比如错走 PgBouncer transaction-pool 导致 LISTEN 丢失，第 54 篇讲过），配置改了但不生效——审计日志显示"改了"，系统还在用旧的，这种静默不一致比完全没审计还危险。

---

## 脱敏：审计日志不能成泄露源

审计记下了请求体和响应体，但这俩里头可能有凭证（第 60 篇讲过）。审计日志自己不能变成泄露源——所以脱敏是审计的硬约束。

apiAuditLog 的脱敏（行 178-201，第 60 篇详讲过）：

```ts
const SENSITIVE_BODY_KEYS = new Set([
  'value', 'secretCiphertext', 'secret', 'token', 'pat',
  'personalAccessToken', 'apiKey', 'api_key', 'password', 'passwd',
]);
function redactSensitiveBodyFields(value: unknown): unknown {
  // 命中敏感 key → [REDACTED]，递归处理嵌套
}
```

还有请求头脱敏（行 147-152）：Authorization 统一变 `Bearer [REDACTED]`。

这套脱敏逻辑跟第 60 篇的凭证管理是配套的。凭证存的时候加密，记审计的时候脱敏——两个环节都不留明文。**审计日志的访问权限通常比 DB 宽（运维、合规都要看），所以它的脱敏标准必须更严。**

---

## 三层审计放一起

把三层 + 通知串起来看：

```
【管理面】配置变更
   ├─ ConfigAuditLog（before/after diff + 操作人 + reason）
   ├─ KernelManagementAuditLog（治理动作 + 作用域 + rollbackHint）
   └─ notifyConfigChange → ConfigDbListener → 缓存失效/重载
         （审计同时是生效的触发器）

【用户面】API 写操作
   └─ apiAuditLog（PendingSpan Registry + 全生命周期 finalize）
         onRequest 登记 → onResponse/close/error/sweep 定稿 → ES
         中断/超时/停机全覆盖，Registry 有上限防撑爆

【贯穿】脱敏
   SENSITIVE_BODY_KEYS 黑名单 + 递归 [REDACTED]
   Authorization → Bearer [REDACTED]
```

三层的分工：

| 层 | 记什么 | 载体 | 核心机制 |
|----|------|------|------|
| ConfigAuditLog | 配置变更 | PG 表 | before/after Json + 操作人 + 索引 |
| apiAuditLog | API 写操作 | ES | PendingSpan + 多触发点 finalize |
| KernelManagementAuditLog | 内核治理 | PG 表 | 作用域 + result + rollbackHint |

**三者正交，覆盖不同场景**：管理员改配置走第一层；用户调 API 走第二层；管理员做内核治理走第三层。一次完整的管理操作可能同时进两层（比如改 agent_config 既进 ConfigAuditLog，如果是内核治理又进 KernelManagementAuditLog），这是有意的冗余——不同视角的审计各有用途。

---

## 为什么审计要分三层

新手会问：为什么不统一成一张"AuditLog"表？答案还是"机制匹配场景"。

**第一，查询模式不同。** 配置审计要按"某条配置的变更历史"查（configType + configId 索引）；API 审计要按"某个用户/IP/时间段的操作"查（ES 全文检索）；内核审计要按"治理作用域"查（targetScope）。一张表的索引满足不了这三种查询模式。所以配置/内核审计进 PG（结构化查询），API 审计进 ES（全文检索）。

**第二，生命周期不同。** 配置审计可能要长期保留（合规要求，几年）；API 审计量大，通常按时间分片定期清理（ES 滚动删除）；内核审计介于两者之间。混在一张表，清理策略没法定。

**第三，写入路径不同。** 配置审计是业务代码主动写（改配置时同步写）；API 审计是中间件统一拦截（Fastify hook）；内核审计是治理流程写。三种写入路径硬塞进一个表，要么侵入业务代码要么拦截不到。

**所以分层的本质是"对不同查询模式、生命周期、写入路径的审计分别建表"**。跟前面几篇讲幂等、并发、多租户"别统一成一个服务"是一个道理——横切的是审计意识，不是统一实现。

---

## 一个细节：Shutdown 时 flush pending spans

apiAuditLog 这层有个细节值得拎出来——进程停机时，PendingSpan Registry 里可能还有未定稿的 span（请求处理到一半进程要退了）。直接退的话，这批 span 就丢了，审计断了。

WinMatrix 的处理是 `flushPendingSpans`（行 520-527）：

```ts
export function flushPendingSpans(reason: InterruptionReason = 'server_shutdown'): void {
  for (const span of registry.values()) {
    if (!span.completed) {
      finalizeInterrupted(span, reason, 502);   // 全标记为中断
    }
  }
  registry.clear();
}
```

停机时把所有未完成的 span 统一标记成 `'server_shutdown'` 中断，落 ES。这样审计日志里能看出"这些请求是被停机打断的"，而不是"莫名其妙就没了"。

这个 flush 在哪调用？在优雅停机序列里（第 64 篇会详讲 shutdown 序列）。停机不是直接 kill，而是按序释放资源，flushPendingSpans 是其中一环。**审计的完整性依赖优雅停机——暴力 kill 的进程，pending span 是会丢的。**

这个细节体现的是审计与运维的耦合：审计不只是应用层的事，还依赖停机、探针、清理策略这些运维机制配合。所以审计是个跨界主题——它横跨 DB（配置审计）、中间件（API 审计）、ES（日志存储）、运维（停机 flush）。

---

## 给后来者的几条总结

1. **审计是三层不是一层**。ConfigAuditLog（配置变更）+ apiAuditLog（API 写操作）+ KernelManagementAuditLog（内核治理），各记各的场景。
2. **配置审计记 before/after diff**。oldValue + newValue 完整快照 + 操作人 id/name + reason，故障复盘第一步永远能答。
3. **API 审计用 PendingSpan Registry 覆盖全生命周期**。正常/中断/超时/停机都不漏记，查询类成功不记减噪音。
4. **中断请求补录**。close/error 事件 + sweep 定时器 + Registry 上限，防漏记也防撑爆。
5. **内核治理审计带作用域和 rollbackHint**。targetScope/result/rollbackHint/traceId，面向快速回滚。
6. **审计同时是配置生效的触发器**。改配置 → ConfigAuditLog → notifyConfigChange → ConfigDbListener → 缓存失效。notify 这环断了，配置改了不生效，比没审计还危险。
7. **脱敏是审计的硬约束**。SENSITIVE_BODY_KEYS 黑名单 + 递归 [REDACTED]，Authorization 脱敏；审计日志权限比 DB 宽，脱敏要更严。
8. **停机时 flush pending spans**。优雅停机序列里调 flushPendingSpans，未完成 span 标记 'server_shutdown' 落 ES；暴力 kill 会丢。
9. **分三层是因为查询模式/生命周期/写入路径都不同**。配置进 PG（结构化、长期），API 进 ES（全文、滚动清理），内核进 PG（治理语义）。横切的是审计意识，不是统一表。

审计日志是合规底线，也是故障复盘的基础。没有审计，出了问题永远是"谁改的？不知道。改了啥？不知道。啥时候改的？不知道"——三不知道的团队，系统只会越跑越烂。把三层审计都做扎实，平台才有"敢让多人改配置"的底气。

---

> **下一篇**：[《背压与限流：当 LLM 处理不过来时怎么办》](./62-backpressure-rate-limiting.md)——审计保证了"改了什么能查"，但系统跑起来还有个更现实的问题：高峰期请求打爆了怎么办。讲 WinMatrix 的三层限流 + 全局信号量 + 超载延迟重投。
