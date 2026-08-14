# ExecutionSpan 的事件双写：JSON 数组 + 子表，为什么要两份

> 这是《WinMatrix 开发经验文集》第 50 篇。第 08 篇和第 34 篇讲过 ExecutionSpan 的整体设计——它怎么取代散落日志、怎么和外部可观测平台对比。这一篇只挖一个细节：一个 span 内部的事件，为什么同时存在于 `execution_span.events`（JSON 数组）和 `execution_span_event`（独立子表）两个地方？这种"双写"不是冗余事故，而是有意为之的读写分离设计。讲它解决什么、一致性边界在哪、又为什么不能合并成一份。

先看一个 span 长什么样。

ExecutionSpan 是 WinMatrix 里 retire-agent-execution-log 后的 SSOT（单一事实来源）。一个 span 是执行树上的一个节点——可能是一次 agent_run、一个 Turn、一次工具调用、或一次 LLM 调用。它的核心字段（`schema.prisma:2990-3038`）：

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
  model               String?
  attributes          Json?
  events              Json?     @default("[]")    ← JSON 数组
  @@index([traceId])
  @@index([spanKind, status])
  @@index([attributes], type: Gin)
  @@map("execution_span")
}
```

注意 `events Json? @default("[]")` 这一行——span 自己带一个 JSON 数组存事件。但同时，还有一张独立的子表 `execution_span_event`（`:3041-3060`）：

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

同一份事件数据，存在两个地方。第一反应是：这不是数据冗余吗？为什么不选一种存法？这一篇就回答这个问题。

---

## 先看两种存法各自的特性

要理解为什么双写，得先看清两种存法各自的优劣。

### JSON 数组（`execution_span.events`）

事件作为 JSON 数组存在 span 行里。读一个 span 时，一次查询就连 span 带事件全拿回来了。

```
读一个 span 的事件：
SELECT span_kind, status, events FROM execution_span WHERE span_id = '...';
→ 一次查询，span 本体和事件数组一起返回
```

优点：**读快。** 单次查询拿到一切，不用 join。调试时"看一个 span 发生了什么"是最高频的操作，这种存法让它变成 O(1) 次查询。

缺点：**写有上限、查询能力弱。** JSON 数组在 PG 里虽然可以查询，但要按事件类型、时间范围筛选，性能和表达力都远不如结构化表。而且 JSON 数组理论上没有硬上限，但实际上 span 行不能无限膨胀——一个长 span 可能有几百上千个事件，全塞进一个 JSON 会让行变得很大，影响整行的读写性能。

### 独立子表（`execution_span_event`）

每个事件是一行，独立存。

```
按事件类型查询：
SELECT * FROM execution_span_event
WHERE trace_id = '...' AND event_type = 'llm_call_end'
ORDER BY ts;

按时间范围查询：
SELECT * FROM execution_span_event
WHERE span_id = '...' AND ts BETWEEN ... AND ...;
```

优点：**查询能力强。** 有 `[spanId, ts]` 和 `[traceId, eventType]` 两个索引，可以高效按 span、按 trace、按事件类型、按时间范围查询。事件可以无限增长而不影响 span 行本身。

缺点：**读一个 span 的全部事件要专门查一次。** 不能和 span 本体一次拿回来。

### 两者的本质差异

```
JSON 数组:   读快（一次拿全），写/查询弱
独立子表:    查询强（索引丰富），读要 join
```

这就是经典的"读优化 vs 写/查询优化"的权衡。**而可观测性场景同时要这两种特性**——调试时要快读一个 span 的全貌（要 JSON），统计分析时要按事件类型/时间范围查（要子表）。

---

## 双写：用冗余换读写分离

WinMatrix 的选择是：**两个都要，双写。**

> 核实报告 ch23-29 的原话："span.events JSON 与 execution_span_event 子表双写，子表有 [spanId, seq] 唯一约束防重。"

双写的语义是：

- **子表是完整事件源（source of truth）。** 所有事件都写进子表，它是权威的、完整的、可查询的。如果 JSON 数组和子表不一致，以子表为准。
- **JSON 数组是读缓存（read projection）。** 它是子表数据的一个"快照视图"，目的是让"读单个 span"这种高频操作变快。

用一张图表达这个分工：

```
事件产生
    │
    ├──→ 写 execution_span_event 子表（SSOT，完整事件源）
    │      可按 spanId/traceId/eventType/ts 查询
    │
    └──→ 更新 execution_span.events JSON 数组（读投影）
           读单 span 时一次拿回
```

这种模式在数据库设计里有个名字叫 **CQRS-like 的读写投影**——写一份权威数据（子表），同时维护一个为读优化的投影（JSON 数组）。读的时候，根据访问模式选合适的源：查单个 span 走 JSON（快），统计分析走子表（强）。

CQRS 这个词听起来高大上，但它的本质很朴素：**承认"读"和"写"有不同的最优结构，与其强行用一种结构兼顾两头（结果两头都不优），不如写两份、各优化各的。** 代价是冗余和一致性维护，但在很多场景下这个代价值得。

---

## 一致性边界：[spanId, seq] 幂等约束

双写最大的风险是"两份不一致"。WinMatrix 用一个关键约束兜住这个风险——子表的 `@@unique([spanId, seq])`：

```prisma
@@unique([spanId, seq], map: "uq_span_event_span_seq")
```

每个事件在一个 span 内有个序列号 `seq`，`(spanId, seq)` 唯一。这意味着：

**同一 span 内，同一 seq 的事件只能存在一条。** 如果因为某种原因（重试、重复投递、崩溃重放）同一个事件被写了两次，第二次会被唯一约束拒绝。这是分布式系统里防重复写入的通用手段（参考第 23 篇讲的幂等设计）。

这个约束的重要性怎么强调都不过分。没有它，双写会因为重复写入导致 JSON 数组里有重复事件、子表里也有重复行，调试时会看到"同一步发生了两次"的假象。有了它，至少子表这一侧是干净的——而子表是 SSOT，它干净，整体就可靠。

那 JSON 数组那边呢？它的重复风险靠"更新时基于子表重建"来防——理论上如果更新逻辑正确，JSON 数组就是子表的忠实投影。但**当 JSON 数组和子表出现不一致时，永远以子表为准**。这是双写体系里必须明确的"一致性方向"——不是两边对等，而是子表权威、JSON 投影。

### 双写的一致性边界在哪

这里要诚实地说出双写的局限：**它不是强一致的，而是"最终一致 + SSOT 兜底"。**

```
写事件:
  1. 写 execution_span_event（成功）
  2. 更新 execution_span.events JSON（可能失败）
     → 如果第 2 步失败，JSON 少了这个事件，但子表有

读事件:
  - 读单 span → 读 JSON（可能少最新几个还没投影的）
  - 统计分析 → 读子表（权威）
  - 发现不一致 → 以子表为准，重建 JSON
```

**这个边界是刻意接受的。** 为什么？因为可观测数据的特性是"读多写多、但容忍短暂不一致"。调试时看 JSON 少了最后一个事件，影响不大（再去子表查就行）；但如果为了强一致把双写包进一个分布式事务，写入性能会暴跌，反而拖垮整个可观测管线。

**双写体系的一致性原则是：子表强一致（靠唯一约束），JSON 最终一致（靠投影），冲突时子表为准。** 这是一个务实的取舍——用"子表权威 + JSON 尽力而为"换写入性能，同时保证可恢复（JSON 可以从子表重建）。

---

## 什么时候该双写，什么时候不该

双写不是免费的——它多了存储、多了写入开销、多了一致性维护的复杂度。所以要问：什么时候值得，什么时候不值得？

**值得双写的条件：**

1. **读和写的访问模式差异显著。** 读要快（单 span 全貌），写/分析要强查询（按事件类型/时间）。如果只有一种访问模式，单存一份就够。
2. **数据是"可重建"的。** JSON 投影可以从子表重建，所以即使 JSON 出错也能恢复。如果投影不可重建（比如 JSON 里有子表没有的独有数据），双写就危险了。
3. **容忍短暂不一致。** 如果业务要求"写入立刻可见且精确"，双写的不一致窗口是隐患。

WinMatrix 的 span events 完美满足这三条——读（调试）和写/分析（统计）模式差异大、JSON 可从子表重建、可观测数据容忍短暂不一致。

**不值得双写的反例：**

- 事务性数据（比如订单状态）不该双写——它要求强一致，双写的不一致窗口会导致业务错误。
- 只有一种访问模式的数据不值得双写——比如只用来批量扫描的日志，单存一张表（或 ES）就够，双写徒增复杂度。

这个判断标准很重要——**双写是读写分离的工具，不是"什么都双写"的银弹。** 用对地方它是性能利器，用错地方它是 bug 温床。

---

## 这个模式在 WinMatrix 其他地方的呼应

有意思的是，"双写/读写分离"在 WinMatrix 里不是 span events 独有。回忆几个相似的设计：

**记忆索引的双写（ES + PG）。** 第 03 篇讲过，记忆索引同时写 ES（dense_vector，检索强）和 PG（memory_chunks，事务性强）。ES 挂了降级仅写 PG，ES 恢复后可从 PG 重建。这和 span events 的"子表权威 + 投影尽力而为"如出一辙——两边访问模式不同（ES 擅长向量检索，PG 擅长事务），双写各取所长。

**会话的三层分工（conversation_histories / session_transcript / conversation_meta）。** 第 19 篇讲过：conversation_histories 是读模型，session_transcript 是 LLM 上下文真源（SSOT），conversation_meta 是权威元数据。这本质上也是一种"多视图"——同一份会话数据，按不同访问模式（读展示 / LLM 上下文 / 元数据查询）存成不同结构。

**ScheduledTaskRun 的 outbox 字段。** 第 16 篇和第 37 篇提到，ScheduledTaskRun 自带 outbox 字段（deliveryStatus、deliveryAttempts），执行终态和结果投递解耦。这也是"终态写一份、投递状态写一份"的双写——为的是让执行和投递可以独立演进、独立重试。

这些设计合起来，说明 WinMatrix 有一个一以贯之的工程取向：**承认不同的访问模式需要不同的数据结构，宁可写两份各自优化，也不强行用一种结构兼顾两头。** 这是 CQRS 思想在各处的具体落地。

---

## 一个真实的读路径：调试 LLM 问题时用哪个源

讲这么多原理，最后落到一个真实场景：调试一次 LLM 调用失败，到底读哪个源？

LLM 调用按遥测契约（代码里以 `llm-call-span-telemetry-contract` 概念名被引用，实现在 `llmObservability.ts`）会产生三类事件：`llm_call_start`（含完整 request）、`llm_call_end`（含 response 和 token 数）、`llm_call_error`（含 errorMessage）。

```
调试场景 1: 看"这次 LLM 调用整体怎么样"
  → 读 execution_span（单行）
  → status、outcome、tokenInput/Output、model 都在 span 本体
  → events JSON 里有 start/end/error 的摘要
  → 一次查询搞定

调试场景 2: 找"这个 trace 里所有失败的 LLM 调用"
  → 读 execution_span_event 子表
  → WHERE trace_id='...' AND event_type='llm_call_error'
  → 用 [traceId, eventType] 索引，快

调试场景 3: 找"昨天 10 点到 11 点所有 llm_call_end 事件"
  → 读子表
  → 按 ts 范围扫，JSON 数组做不了这个
```

看到了吗——**不同的调试问题，自然地路由到不同的源。** 这正是双写的价值：你不用为了"找某 trace 的失败调用"而去扫所有 span 的 JSON 数组（那会慢到不可用），子表的索引让这类查询极快；你也不用为了"看一个 span 的全貌"而去 join 子表（那要两次查询），JSON 数组让单 span 读取 O(1)。

两种源，两种读路径，各司其职。这就是双写存在的全部理由。

---

## 给后来者的几条总结

1. **双写是读写分离的工具，不是银弹。** 用在"读和写访问模式差异显著"的场景是利器，用在"只有一种访问模式"或"要求强一致"的场景是负担。
2. **子表是 SSOT，JSON 是读投影。** 双写必须明确一致性方向——一边权威、一边尽力而为。两边对等的双写会陷入"谁是真源"的混乱。
3. **`@@unique([spanId, seq])` 是双写的防重复底线。** 任何会重复写入的分布式系统，都要有幂等唯一约束兜底。没有它，双写会因为重复写入产生假象数据。
4. **冲突时以子表为准，JSON 可从子表重建。** 可重建是双写安全的根本——投影坏了能从权威源恢复，反过来就不行。
5. **可观测数据容忍短暂不一致。** 不要为可观测数据的双写上强一致事务——写入性能暴跌的代价远大于"JSON 少了最后几个事件"的影响。
6. **判断双写是否值得的三条件：** 读写模式差异显著、数据可重建、容忍短暂不一致。三条都满足才值得，否则单存一份。
7. **CQRS 不是高大上的词，是朴素的工程取向。** WinMatrix 在记忆（ES+PG）、会话（三层）、span events（JSON+子表）多处用了同一思想——承认不同访问模式需要不同结构，各优化各的。
8. **不同的调试问题自然路由到不同的源。** 设计双写时，想清楚"高频读"走哪边、"分析查询"走哪边，让访问路径清晰。

span events 的双写是个小细节，但它折射出的是一个大原则：**数据的形状应该服从访问的模式，而不是让访问去迁就数据的形状。** 把这条原则记住，你在很多地方都会发现双写/多视图的用武之地。

---

> **下一篇**：[《终态收敛：分布式系统里"未完成"比"失败"更危险》](./51-terminal-convergence.md)——从这一篇开始进入工程哲学。前面散落在 reconcile、watchdog、outbox、CodingTask 回收里的设计，背后是同一个思想：所有长流程都必须有强制收敛到终态的兜底。把它抽出来讲清楚。
