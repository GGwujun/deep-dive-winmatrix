# 企业级 AI 的可观测性：ExecutionSpan 如何取代散落的日志

> 这是《WinMatrix 开发经验文集》第 8 篇。做 Agent 平台，最让人崩溃的不是写 Agent，而是调试一个"跑飞了"的 Agent——它到底走了哪条路、调了哪个工具、卡在哪一步？这篇讲我们怎么用统一的执行追踪（ExecutionSpan）终结散落日志的噩梦。

如果你做过 Agent 平台，一定经历过这种绝望：

生产环境一个用户反馈"AI 回答得不对"。你打开日志，看到的是几万行散落的 `console.log`：

```
[2026-08-13 10:23:11] [INFO] 收到用户消息
[2026-08-13 10:23:11] [INFO] 开始决策
[2026-08-13 10:23:12] [DEBUG] tool_call: rag_search
[2026-08-13 10:23:13] [INFO] LLM 调用完成
[2026-08-13 10:23:13] [ERROR] 某个错误
...
```

这些日志的问题是：**它们之间没有关联**。你不知道那个 ERROR 是哪个用户、哪次对话、哪个工具调用产生的。你想还原"这次执行到底发生了什么"，得靠时间戳和祈祷去拼——而且经常拼不上。

这篇文章讲我们怎么用**统一的执行追踪（ExecutionSpan）**终结这个噩梦。

---

## 第一步：把"日志"升级成"树"

传统日志是**线性的**——一条一条往下排。但一次 Agent 执行，结构上不是线性的，是**树形的**：

```
一次 agent_run（根）
├── 决策阶段
│   ├── Stage 1: 精确路由
│   └── Stage 2: 融合路由
├── Turn 执行
│   ├── LLM 调用 #1
│   ├── 工具调用：rag_search
│   │   └── 子 LLM 调用（改写查询）
│   ├── LLM 调用 #2
│   └── 工具调用：write_doc
└── 结果持久化
```

每个节点都可能成功、失败、耗时不同、产生不同的数据。线性日志根本表达不了这种结构。

我们的解法是 **ExecutionSpan**——每一个执行单元（agent_run、Turn、工具调用、LLM 调用）都是 span 树上的一个节点：

```prisma
// prisma/schema.prisma（第 2990-3038 行）
model ExecutionSpan {
  spanId              String    @id @map("span_id")
  traceId             String    @map("trace_id")
  parentSpanId        String?   @map("parent_span_id")   ← 父节点，构成树
  spanKind            String    @map("span_kind")          ← 节点类型
  status              String    @default("pending")
  outcome             String?
  tokenInput          Int?      @map("token_input")       ← LLM token 用量
  tokenOutput         Int?
  tokenThinking       Int?
  model               String?
  provider            String?
  failureReasonCode   String?   @map("failure_reason_code")
  agentRunId          String?   @map("agent_run_id")
  attributes          Json?                                 ← 自由属性
  events              Json?     @default("[]")             ← 事件（与子表双写）
  @@index([traceId])
  @@index([spanKind, status])
  @@index([attributes], type: Gin)   ← GIN 索引支持 JSON 查询
}
```

几个关键设计：

### traceId + parentSpanId 构成树

同一个 `traceId` 下，所有 span 通过 `parentSpanId` 串成一棵树。一次完整的 agent_run 是一棵树，你能从根走到任何叶子，看清每一步。

### spanKind 标识节点类型

`spanKind` 区分这是 agent_run、Turn、tool_call、llm_call 还是别的。按类型查询、统计都靠它。

### GIN 索引让 JSON 可查

`attributes` 是个 Json 字段，存的是各 span 类型特有的属性（比如 llm_call 存 model、token 数；tool_call 存工具名、参数摘要）。给它建 GIN 索引（`@@index([attributes], type: Gin)`），意味着你能高效地按 JSON 内字段查询——"找出所有用了 claude 模型且失败了的 LLM 调用"。

---

## 一个真实的查询：从"用户说不对"到"定位到根因"

有了 span 树，排查一次问题变得极其清晰。CLAUDE.md 里有一条专门给运维的 SQL：

```sql
-- 查看某会话执行轨迹（execution_span，retire-agent-execution-log 后 SSOT）
SELECT span_kind, started_at, executor_employee_name, status
FROM execution_span
WHERE conversation_id = '<conv_id>'
ORDER BY started_at ASC;
```

**一句 SQL，一次对话里"谁、什么时候、做了什么、成功没"全出来了。** 你看到的是一棵扁平化的执行树：决策 span、Turn span、几个 tool_call span、几个 llm_call span，按时间排好，每个带状态。

定位问题变成"看哪个 span 是 failed / 异常耗时"，而不是"在几万行日志里 grep"。

这就是为什么我们在注释里明确写着"retire-agent-execution-log 后 SSOT"——我们甚至把旧的 `agent_execution_log` 表**退役了**，统一用 span。单一事实来源（SSOT）的价值。

---

## 第二步：Span Events——把"日志"搬进 span 里

光有 span 节点还不够。一个 span 内部可能有很多事件——开始、进度、子事件、结束。这些事件怎么存？

我们的做法是**子表 + JSON 双写**：

```prisma
// prisma/schema.prisma（第 3041-3060 行）
model ExecutionSpanEvent {
  id        BigInt   @id @default(autoincrement())
  spanId    String   @map("span_id")
  eventType String   @map("event_type")
  phase     String?
  ts        DateTime @db.Timestamptz(6)
  attrs     Json     @default("{}")
  seq       Int                                   ← 序列号
  @@unique([spanId, seq], map: "uq_span_event_span_seq")   ← 防重
  @@index([spanId, ts])
  @@map("execution_span_event")
}
```

每个 span 在 `execution_span.events`（JSON 数组）和 `execution_span_event`（子表）里**双写**。子表是完整事件源，JSON 是为了快速读取。

`@@unique([spanId, seq])` 这个约束很重要——它保证同一 span 内事件序列号唯一，防止重复写入导致的事件重复。**任何会重复写入的分布式系统，都要有幂等约束兜底。**

---

## 第三步：三 Sink 路由——不是所有事件都要落库

一个 Agent 系统每秒可能产生成千上万的事件。如果每个都写进 span 表，数据库扛不住，也没必要。

我们的做法是用一张**规则表**，决定每个事件类型该怎么处理：

```prisma
// prisma/schema.prisma（第 3063-3094 行）
model UnifiedObservabilityRule {
  eventType        String   @map("event_type")        ← 事件类型
  runtimeActionKey String   @default("")
  sinkSpan         Boolean  @default(true)             ← 落 span 表？
  sinkLog          Boolean  @default(false)            ← 落日志？
  sinkChannels     String[] @default([])               ← 推到哪些渠道？
  opensSpan        Boolean  @default(true)
  opensInvocation  Boolean  @default(false)
  redactionPolicy  Json     @default("{}")             ← 脱敏策略
  @@id([eventType, runtimeActionKey])
}
```

每种事件类型（如 `llm_call_start`、`tool_call_end`、`thinking:delta`）都有自己的路由规则：

- 要不要落 span 表？（`sinkSpan`）
- 要不要落日志？（`sinkLog`）
- 要不要推给前端/企微/大屏？（`sinkChannels`）
- 要不要脱敏？（`redactionPolicy`）

`ObservabilityHub` 在 `record()` 时查这张规则表，把事件分流到对应的 Sink。**不是所有事件都值得持久化，"记什么"本身应该是可配置的。**

这个设计还带了**脱敏**——有些事件（比如包含用户敏感信息的 LLM 请求）需要按 `redactionPolicy` 脱敏后才能落库。这在企业场景下是合规刚需。

---

## LLM 调用遥测：token 和钱的归因

AI 平台的可观测性，有一个传统后端没有的特殊诉求——**LLM 调用的成本归因**。

每次 LLM 调用花多少钱（多少 token）、用的什么模型、为什么调用，必须可查。否则月底 API 账单来了，你不知道钱花哪了，更不知道哪个技能/哪个员工是"烧钱大户"。

我们有一份专门的 LLM Span 遥测契约（`openspec/contracts/llm-call-span-telemetry-contract.md`），规定了每次 LLM 调用必须记录的事件：

| 时机 | 事件 | 必填字段 |
|------|------|---------|
| 调用前 | `llm_call_start` | llmInvocationId、actionName、**request（含 messages）** |
| 调用后 | `llm_call_end` | **response**、inputTokens、outputTokens |
| 失败 | `llm_call_error` | errorMessage（保留已发出的 request） |

注意几个强制要求：

### request 和 response 必须完整记录

`llm_call_start` 必须含 `request.messages`，`llm_call_end` 必须含 `response`。**只有 thinking:delta 或空 events 不算完成遥测。** 为什么这么严格？因为调试 LLM 问题时，你首先要看的就是"当时发给了模型什么、模型回了什么"。如果没有完整的 request/response，调试无从谈起。

### 完整 request 在源点暂存，防 compact 剥离

这里有个巧妙的工程细节。可观测 Hub 在把事件落到日志 Sink 时，为了控制日志体积，会 `compactLlmEventDataForLogSink`——把 request 剥掉。但这会导致日志里没有完整 request。

我们的做法是：**在源点（emitLLMCallStart 那一刻），先把完整 request 暂存起来**：

```typescript
// infrastructure/observability/llmObservability.ts（第 89-128 行）
export async function emitLLMCallStart(...) {
  // 提案2 D1：在 hub compactLlmEventDataForLogSink 剥离 request 之前，于源点暂存完整 request
  if (obsFields.llmInvocationId) {
    import('./elasticsearchLlmLogger.js')
      .then(({ storeFullRequestForRun }) => storeFullRequestForRun(invocationId, request))
      .catch(() => {});
  }
  await recordViaHubOrLegacy({ ..., eventType: 'llm_call_start', request, ... });
}
```

**在数据被"加工"之前，先把原始版本存下来。** 这是个普适的模式——任何有数据降采样的管线，都要在降采样前的源点保留一份完整数据，否则降采样后的数据出了问题，你永远回不到原始状态。

---

## 悬挂的 LLM 调用：一个真实的生产坑

讲一个真实的、让我们专门做了补偿机制的场景。

LLM 调用是网络请求，可能因为各种原因（网络抖动、进程崩溃、超时）半途而废。这时 `llm_call_start` 已经记了，但 `llm_call_end` 永远不会来——这个调用成了**悬挂调用**。

悬挂调用如果不管，会导致：

- span 树里永远有 status=pending 的孤儿节点
- 关联的 agent_run / scheduled_task_run 也永远停在 running 状态
- 统计数据失真（看起来一直没完成的调用）

我们的解法是一个**补偿 sweeper**：

```typescript
// infrastructure/scheduled/llmCallWatchdogSweeper.ts（第 1-53 行）
export const LLM_CALL_WATCHDOG_TASK_NAME = 'system-llm-call-watchdog-sweeper';
function resolveSweepThresholdMs(): number {
  const hardMs = getConfig().llmCallHardTimeoutMs;
  return hardMs > 0 ? hardMs * 2 : 360_000;   // 2x hard timeout 或 6 分钟
}
```

这个 sweeper 每 10 分钟跑一次（`system-llm-call-watchdog-sweeper`，interval 600_000ms），找出超过阈值（2 倍硬超时，或 6 分钟）还没结束的 LLM 调用，**补写一个 `llm_call_end`**，并**级联 finalize** 关联的 agent_run 和 scheduled_task_run——把它们从 running 状态收敛到终态。

它还会通过 `DanglingFailureReceiptPusher` 向会话补发一个失败回执——让用户知道"这次调用其实失败了"，而不是让对话永远卡着。

**教训：任何涉及外部网络调用的可观测系统，都必须有"悬挂检测 + 补偿收敛"机制。** 否则你的状态机会被孤儿节点堵死，统计会失真，用户会一直等。

---

## 可观测性的三重价值

回过头看，这套 ExecutionSpan 体系给我们带来了三重价值：

**1. 调试价值**：一次问题排查，从"几万行日志里 grep"变成"一条 SQL 查 span 树"。这是最直接的收益。

**2. 成本价值**：LLM 调用的 token、模型、归属都记录在案，成本归因变得可行。"哪个技能最烧钱""哪个员工调用最频繁"这些问题有了数据答案。

**3. 可靠性价值**：悬挂检测 + 补偿收敛，让系统状态不会因为半途失败而卡死。可观测性不只是"看"，还反向提升了系统的自愈能力。

---

## 给后来者的几条总结

1. **把日志升级成 span 树**。一次执行是树形的，不是线性的。traceId + parentSpanId 构成树，spanKind 标识节点类型。
2. **SSOT 单一事实来源**。我们退役了 agent_execution_log，统一用 ExecutionSpan。多套并存的观测表是混乱之源。
3. **JSON 字段配 GIN 索引**。自由属性用 Json 存，GIN 索引让它可高效查询。
4. **事件双写 + 幂等约束**。span.events JSON + 子表双写，`@@unique([spanId, seq])` 防重复。
5. **三 Sink 路由，"记什么"要可配置**。不是所有事件都落库，规则表决定 span/log/channel 的分流和脱敏。
6. **LLM 调用强制记录完整 request/response**。调试 LLM 问题全靠它。在 compact 前的源点暂存完整数据。
7. **悬挂检测 + 补偿收敛**。外部调用会半途失败，必须有 sweeper 补写终态、级联 finalize、补发回执。
8. **在写第一个 Agent 之前就把 span 追踪搭好**。等事故来了再补，面对的是几百万行无法关联的日志。

可观测性不是 Agent 平台的"可选项"，而是"能不能在生产里活下去"的底线。一个黑盒 Agent 系统，出问题时你只能干瞪眼——而它一定会出问题。

---

> **下一篇**：[《我们踩过的坑：Prisma v7 时区、连接池风暴与 single-flight》](./09-pitfalls-infra.md)——从可观测性跳到基础设施，讲三个让我们深夜爬起来的真实事故。
