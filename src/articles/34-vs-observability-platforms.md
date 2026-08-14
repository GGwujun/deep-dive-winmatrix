# ExecutionSpan vs Langfuse/LangSmith：自建可观测 vs 通用平台

> 这是《WinMatrix 开发经验文集》第 34 篇，"行业对比与方法论"系列第四篇。Agent 系统的可观测性，业界有一批成熟的通用平台（Langfuse、LangSmith、以及各家通用 APM 的 LLM 观测模块）。WinMatrix 却选择自建了一套 ExecutionSpan 体系，甚至把旧的 agent_execution_log 表退役了。这篇聊"通用平台够不够用"和"什么时候必须自建"。

做 Agent 平台，可观测性不是一个可选项，而是"能不能在生产里活下去"的底线。一个黑盒 Agent 系统，出问题时你只能干瞪眼——而它一定会出问题。

可观测这件事，业界有两条路。

一条是**用通用平台**：业界有一批专门做 LLM/Agent 可观测的产品，它们的核心能力大同小异——把 LLM 调用和 Agent 执行抽象成 trace/span，提供 waterfall 视图、按 agent 归因 token、prompt 版本管理、评估打分等。接入通常是 SDK 几行代码，平台托管存储和可视化，开箱即用。

另一条是**自建**：WinMatrix 选的这条路。我们做了一套 ExecutionSpan 体系，每一个执行单元（agent_run、Turn、tool_call、llm_call）都是 span 树上的节点，存在自己的 PG 里，退役了旧的 agent_execution_log，统一用 span 作为 SSOT。

这篇就聊"通用平台够不够用"，以及"什么时候自建是值得的"。结论不是"自建更好"——通用平台在大多数场景下够用且省事。但 WinMatrix 的某些诉求，通用平台确实满足不了。

---

## 先说通用平台擅长什么

公平起见，先说清楚通用平台（Langfuse、LangSmith 这一类）擅长什么。它们不是没价值，恰恰相反，对很多团队来说它们是最优解。

**通用平台的核心能力**：

```
通用 LLM 可观测平台的典型能力栈：
  ┌─────────────────────────────────────┐
  │  可视化层：trace waterfall / dashboard │
  ├─────────────────────────────────────┤
  │  归因层：按 session/agent/user 聚合    │
  │         token、成本、延迟              │
  ├─────────────────────────────────────┤
  │  采集层：SDK 自动埋点 LLM 调用 +        │
  │         自定义 span                    │
  ├─────────────────────────────────────┤
  │  存储层：平台托管（通常 ClickHouse 或   │
  │         类似列存）                      │
  └─────────────────────────────────────┘
```

它们擅长的事：

1. **快速接入**。SDK 几行代码，LLM 调用自动埋点，不用自己设计 schema、不用自己建存储。
2. **现成的可视化**。trace 的 waterfall 图、token 成本 dashboard、延迟分布——这些不用自己做前端。
3. **LLM 调用的通用归因**。按 session/user/agent 维度聚合 token 和成本，是这类平台的基础能力。
4. **prompt 版本管理和评估**。有些平台还带 prompt 版本控制、A/B 测试、自动评估打分这些工作流。
5. **托管运维**。存储、查询性能、可用性都由平台负责，你的团队不用背这个运维负担。

**对于一个"刚起步、想快速看到 Agent 执行全貌"的团队，通用平台是非常好的选择。** 几天就能接入，立刻有了 trace 可视化和成本归因，不用自己造轮子。如果你的 Agent 系统规模不大、可观测诉求不特殊，通用平台完全够用。

---

## 通用平台在哪里不够用：三个 WinMatrix 的真实诉求

那么 WinMatrix 为什么还要自建？不是因为"造轮子好玩"，而是有三个真实诉求，通用平台满足不了。

### 诉求一：可观测数据要和业务数据在同一库，能 JOIN

通用平台的可观测数据存在它们自己的后端（通常是独立的列存数据库），和你的业务库（PG）是物理隔离的。这带来一个硬伤——**可观测数据没法和业务数据 JOIN**。

举个例子。我想查一个问题："这个月，阿码（tech_manager）在做代码评审时，有多少次 LLM 调用失败、失败的 agent_run 关联了哪些编码任务、这些任务属于哪些项目？"

这个查询要 JOIN 四张表：

```
execution_span（LLM 调用，status=failed）
  ↓ agent_run_id
agent_run（执行，executor_employee_name='阿码'）
  ↓ 关联
coding_task（编码任务）
  ↓ project_id
projects（项目）
```

在 WinMatrix 里，这四张表都在同一个 PG 库，一句 SQL 全出来。但如果可观测数据在通用平台的独立后端，你就得：先从平台导出失败 LLM 调用的列表 → 手动和业务库 JOIN → 还可能因为 ID 体系对不上而 JOIN 不了。**跨数据源的关联查询，在生产排查时是极其痛苦的。**

第 8 篇讲过的那个排查场景——"用户说不对，一句 SQL 查 span 树定位根因"：

```sql
SELECT span_kind, started_at, executor_employee_name, status
FROM execution_span
WHERE conversation_id = '<conv_id>'
ORDER BY started_at ASC;
```

这一句 SQL 能用，前提是 `execution_span` 和 `conversation_id`、`executor_employee_name` 这些业务字段在同一个库。如果 span 在平台、conversation 在 PG，这句 SQL 根本写不出来。

**自建的第一个理由：可观测数据和业务数据同库，能 JOIN，能一句 SQL 串联排查。** 这是通用平台的结构性限制，不是它们做得不够好。

### 诉求二：span 要能承载"业务语义"，不只是"技术指标"

通用平台的 span 模型，核心是技术指标——duration、token、status、model。这些足够回答"这次调用花了多久、花了多少 token、成没成功"。但企业场景里，你还要回答"这次调用是**哪个业务实体**触发的"。

WinMatrix 的 ExecutionSpan 带着丰富的业务语义：

```prisma
// prisma/schema.prisma（第 2990-3038 行）
model ExecutionSpan {
  spanId              String    @id
  traceId             String                        ← trace 树
  parentSpanId        String?                       ← 父节点
  spanKind            String                        ← 节点类型（agent_run/turn/tool_call/llm_call）
  status              String    @default("pending")
  outcome             String?
  tokenInput          Int?                          ← token 归因
  tokenOutput         Int?
  tokenThinking       Int?
  model               String?
  provider            String?
  failureReasonCode   String?                       ← 失败原因码（业务语义）
  agentRunId          String?                       ← 关联 agent_run
  attributes          Json?                         ← 自由属性（业务语义）
  events              Json?     @default("[]")
  @@index([attributes], type: Gin)                  ← GIN 索引支持 JSON 查询
}
```

注意两个设计：

**`failureReasonCode`**——失败原因码。这不是通用的"error"或"timeout"，而是带业务语义的（比如"skill_not_ready""credential_missing""tool_policy_denied"）。这让"为什么失败"能按业务原因聚合，而不只是按技术错误类型聚合。

**`attributes` Json + GIN 索引**——自由属性。不同 spanKind 的 span，attributes 里装不同的业务字段：llm_call 的 attributes 装 actionName/llmInvocationId；tool_call 的装 toolName/参数摘要；turn 的装 conversationId/roleId。给它建 GIN 索引（`@@index([attributes], type: Gin)`），意味着你能高效地按 JSON 内字段查询——"找出所有用了 claude 模型且 failureReasonCode=credential_missing 的 llm_call"。

```
通用平台 span 模型：          WinMatrix ExecutionSpan：
  duration                     duration
  token                        token
  status                       status
  model                        model
  （到此为止）                  + failureReasonCode（业务语义）
                               + attributes Json（自由业务字段）
                               + agentRunId（业务关联）
                               + GIN 索引（业务字段可查）
```

**自建的第二个理由：span 要能承载业务语义，而不只是技术指标。** 通用平台的 span 模型是面向"通用 LLM 调用"设计的，它的字段集是技术指标为主；企业的业务语义（员工、技能、项目、失败原因码）塞不进它的固定 schema，只能硬塞 metadata，而 metadata 通常没有索引、查不动。

### 诉求三：三 Sink 路由 + 脱敏，是平台做不到的精细化控制

通用平台的采集是"全量上报到平台"——SDK 自动埋点，所有事件都发过去。这在简单场景下没问题，但在企业场景里有两个硬伤：

1. **不是所有事件都值得持久化**。一个 Agent 系统每秒可能产生成千上万的事件（每个 token 流式输出都是一个事件）。全量上报到平台，一是成本高（平台按量收费），二是噪声大（排查时被无关事件淹没）。
2. **敏感数据脱敏要可控**。LLM 调用的 prompt 里可能包含用户隐私、商业机密。全量上报到第三方平台，合规风险高。

WinMatrix 的做法是**三 Sink 路由**——用一张规则表决定每个事件类型该怎么处理：

```prisma
// prisma/schema.prisma（第 3063-3094 行）
model UnifiedObservabilityRule {
  eventType        String   @map("event_type")        ← 事件类型
  runtimeActionKey String   @default("")
  sinkSpan         Boolean  @default(true)             ← 落 span 表？
  sinkLog          Boolean  @default(false)            ← 落日志？
  sinkChannels     String[] @default([])               ← 推到哪些渠道（前端/企微/大屏）？
  opensSpan        Boolean  @default(true)
  opensInvocation  Boolean  @default(false)
  redactionPolicy  Json     @default("{}")             ← 脱敏策略
  @@id([eventType, runtimeActionKey])
}
```

每种事件类型（`llm_call_start`、`tool_call_end`、`thinking:delta`……）都有自己的路由规则：

- 要不要落 span 表？（`sinkSpan`）——关键事件落，噪声事件不落。
- 要不要落日志？（`sinkLog`）——部分事件落日志做长期归档。
- 要不要推给前端/企微/大屏？（`sinkChannels`）——用户可见的进度事件推，内部事件不推。
- 要不要脱敏？（`redactionPolicy`）——含敏感信息的事件按策略脱敏后再落。

```
事件流的三 Sink 分流（ObservabilityHub）：
                    事件进来
                       ↓
              ┌─────────────────┐
              │ UnifiedObserv-  │
              │ abilityRule 查表 │
              └────┬─────┬──────┘
                   ↓     ↓     ↓
              ┌────┐ ┌────┐ ┌────────┐
              │Span│ │Log │ │Channel │
              │Sink│ │Sink│ │Sink    │
              └────┘ └────┘ └────────┘
              （落PG）（落日志）（推前端/企微）
```

`ObservabilityHub` 在 `record()` 时查这张规则表，把事件分流到对应的 Sink。**"记什么、不记什么、记了怎么脱敏"本身就是可配置的**，而且配置在业务库里，可以热更新（第 30 篇）。

**自建的第三个理由：事件路由和脱敏要精细化、可配置、可热更新。** 通用平台是"全量上报"模型，你很难对"这个事件类型只落日志不落 span""那个事件类型要脱敏后才能持久化"做精细控制。而企业合规和数据治理，恰恰要求这种精细控制。

---

## 退役 agent_execution_log：SSOT 的价值

讲完了三个诉求，再讲一个自建带来的衍生价值——**SSOT（Single Source of Truth，单一事实来源）**。

在自建 ExecutionSpan 之前，WinMatrix 的可观测数据散在好几处：agent_execution_log（旧的执行日志表）、各种 console.log、ES 里的 LLM 调用日志……排查问题时要在多个数据源之间跳来跳去，还经常对不上。

自建 ExecutionSpan 后，我们做了一个干脆的决定——**退役 agent_execution_log**。系统维护任务里明确写着"agent_execution_log 已退役（retire-agent-execution-log）"，schema 里 agent_execution_log 已经没有 model 定义了。所有执行追踪统一用 ExecutionSpan + ExecutionSpanEvent。

```
退役前（多套并存）：              退役后（SSOT）：
  agent_execution_log              ExecutionSpan（+ Event 子表）
  + console.log                    ↑
  + ES LLM 日志                     唯一真源
  + 各种散落日志
  （排查要跨源跳，对不上）          （一句 SQL 全出来）
```

SSOT 的价值不是"少一张表"，而是**消除"哪个数据源是真的"的歧义**。多套并存时，你永远要问"这个执行的 status 到底以哪张表为准"——因为不同表可能因为更新时序不同而暂时不一致。统一成一个真源后，没有这个问题。

这是自建的第四个理由（虽然是衍生的）：**自建让你能做到 SSOT，而接入通用平台本质上是引入了"第二套数据源"**——平台一套、你的业务库一套，两者之间的对账和一致性又成了新的负担。

---

## 自建的代价：诚实交代

讲了这么多自建的理由，也必须诚实地说自建的代价。这些代价不小，不是所有团队都该背：

1. **工程投入大**。设计 schema、建索引、写 ObservabilityHub、做可视化前端——这一整套从零做起来，至少几个工程师月的投入。通用平台几天接入，自建几个月起步。
2. **可视化要自己做**。通用平台的 trace waterfall、dashboard 是现成的；自建的话，要么自己写前端，要么接受"只能写 SQL 查"的朴素体验。WinMatrix 目前偏后者——运维排查主要靠 SQL，没有花哨的可视化。
3. **存储和查询性能要自己背**。span 数据量大（尤其开了 events 双写），GIN 索引、分区、清理策略都得自己设计。我们有个 `system-observability-cleanup` 定时任务（每天 4:30）清理过期的 execution_span / pipeline_run / session_transcript。
4. **持续维护负担**。LLM 调用遥测契约（`llm-call-span-telemetry-contract.md`）要维护、悬挂调用补偿（llmCallWatchdogSweeper）要维护、新 spanKind 要加规则——这些都是自建带来的长期负担。
5. **没有现成的"行业基准"对比**。通用平台能看到"你的 Agent 延迟对比行业基准如何"；自建只有自己的数据，没有横向对比。

**这些代价意味着：自建可观测是"重投入、慢回报"的选择。** 团队规模小、Agent 系统简单、可观测诉求不特殊时，通用平台是更好的起点。只有当你的系统到了"通用平台的结构性限制开始拖后腿"的规模，自建才值得。

---

## 一个混合方案：通用平台 + 自建核心

其实"通用平台"和"自建"不是非此即彼。一个务实的混合方案是：

- **核心执行追踪自建**（ExecutionSpan，和业务同库，能 JOIN，带业务语义）——这是排查和归因的真源。
- **LLM 调用的明细上报一份给通用平台**——用平台的可视化做日常监控和 dashboard，省掉自己做前端的活。

这样既保留了自建"可观测 + 业务 JOIN + SSOT"的核心价值，又借用了通用平台"现成可视化"的便利。代价是同一份 LLM 调用数据存了两处（自建 span + 平台），但对账压力比"全量业务可观测都外挂平台"小得多——因为核心排查靠自建 span，平台只是辅助视图。

WinMatrix 目前是纯自建，但如果重来一次，我可能会考虑这个混合方案——把"造可视化前端"的活甩给通用平台，自己只专注"核心 span 数据 + 业务 JOIN"这一层。这是 hindsight，给后来者参考。

---

## 什么时候该选什么

总结一下选型建议。

**选通用平台**：

- 团队小、起步阶段，想快速看到 Agent 执行全貌。
- Agent 系统规模不大，可观测诉求以"LLM 调用监控 + 成本归因"为主。
- 不需要可观测数据和业务数据深度 JOIN。
- 没有严格的敏感数据合规要求（或者平台能满足）。
- 不想背可观测基础设施的运维负担。

**选自建**：

- Agent 系统到了生产规模，排查需要可观测数据和业务数据频繁 JOIN。
- span 要承载丰富的业务语义（员工、技能、项目、业务失败原因码）。
- 事件路由和脱敏要精细化、可配置、可热更新。
- 追求 SSOT，不想维护"业务库 + 平台"两套数据源的对账。
- 团队有持续的工程投入能力，能背 schema/索引/清理/可视化的长期维护。

**选混合方案**：

- 核心排查自建（SSOT + 业务 JOIN），日常监控借通用平台可视化。
- 适合"已经自建了核心、但不想自己做前端 dashboard"的团队。

**判断标准很简单：你的可观测诉求是否需要和业务数据深度关联？** 需要就自建（或混合）；不需要就通用平台。大多数团队起步时不需要，所以通用平台是好的默认选择。但企业级、生产规模、深度排查导向的系统，迟早会撞上通用平台的结构性限制——那时再考虑自建或混合。

---

## 给后来者的几条总结

1. **通用平台擅长快速接入、现成可视化、托管运维**。起步阶段和简单场景，它们是最优解。别为了"造轮子"而自建。
2. **自建的核心理由是"可观测 + 业务同库可 JOIN"**。跨数据源关联查询在生产排查时极其痛苦。这是通用平台的结构性限制。
3. **span 要承载业务语义，不只是技术指标**。failureReasonCode、attributes Json + GIN 索引，让 span 能按业务维度聚合查询。
4. **三 Sink 路由 + 脱敏是精细化控制**。不是所有事件都持久化，含敏感信息的要脱敏。"记什么"本身应可配置、可热更新。
5. **SSOT 的价值是消除"哪个数据源是真的"的歧义**。我们退役 agent_execution_log，统一用 ExecutionSpan。多套并存是混乱之源。
6. **自建代价不小：工程投入、可视化、存储性能、长期维护**。重投入慢回报，团队要有准备。
7. **混合方案是务实选择**：核心 span 自建（SSOT + 业务 JOIN），日常监控借通用平台可视化。
8. **选型标准：看你是否需要可观测和业务深度关联**。需要就自建/混合；不需要就通用平台。规模和诉求决定选型，不是"谁更先进"。

可观测性是 Agent 系统的底座。底座选错了，上层做得再好也是黑盒。但底座也不是越自建越好——通用平台能解决的别自建，自建要做就做透（SSOT + 业务 JOIN + 精细路由），别做半吊子。

---

> **下一篇**：[《企业级 AI 落地的五个层级：从模型到可观测》](./35-five-layer-model.md)——可观测讲完了，最后一篇用"五层模型"把整个系列串起来，讲每一层的选型和 build vs buy。
