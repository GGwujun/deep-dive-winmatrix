# 可重放性：一个 bug 能不能"按原样再跑一遍"

> 这是《WinMatrix 开发经验文集》第 53 篇，进入第三批的"更深的工程哲学"段。前面 40 篇里，ExecutionSpan、session_transcript、幂等键、漂移检测这些概念反复出现过，但都是"为了解决某个具体问题"。这一篇把它们拎到一个统一的视角下：**可重放性（replayability）**——一个生产事故发生后，你能不能把那一次执行"按原样再跑一遍"，看到同样的决策路径、同样的工具调用、同样的失败点。可重放不是某个模块的功能，它是贯穿数据库、可观测性、测试 fixture 的一条暗线。这条线决定了你的系统是"能跑"还是"能查、能改、能进化"。

先讲一个每个做 Agent 平台的人都做过的噩梦。

某天早上，一个用户反馈："大福昨天下午帮我拆项目，拆到一半卡住了，什么都没返回。" 你打开日志面板，看到的是几万行散落的 log，里面有 `turn_start`、有 `llm_call_start`、有 `tool_call`、有 `route_match`……但它们分布在十几个文件、跨越三个进程（api / scheduled / rag），时间戳还因为时区问题对不齐。你 grep 了半小时，拼不出"那一次大福到底走了哪条决策路径、在哪一步崩的"。

这就是**不可重放系统**的典型困境：生产事故来了，你只能看一个模糊的"大概在哪崩了"，没法精确复现。修了 bug 没法回归，因为压根没有"原样输入"可以重跑。

可重放性不是"再加一个功能"，而是从一开始就在数据模型、可观测性、测试基础设施里埋下的几条契约。WinMatrix 在这条线上做了四件事，凑在一起才让"按原样再跑一遍"成为可能。

---

## 第一件事：执行树，而不是执行日志

传统系统的可观测性是"日志"——一条一条平铺的事件流。Agent 系统用日志会死得很惨，因为一次执行是**一棵树**，不是一条线：一次 agent_run 下面挂多个 step，每个 step 下挂多个 LLM 调用，每个 LLM 调用下挂多个工具调用，工具调用还可能触发子 agent_run。平铺日志里这些事件的父子关系全丢了，你看到的是一锅粥。

WinMatrix 的解法是 **ExecutionSpan 树**（`prisma/schema.prisma:2990-3038`，核实报告 ch23-29）：

```
model ExecutionSpan {
  spanId              String    @id
  traceId             String             // 一次执行一棵树，traceId 是树根
  parentSpanId        String?            // 父节点，构成树
  spanKind            String             // agent_run / turn / llm_call / tool_call ...
  status              String             // pending / ok / error
  outcome             String?
  tokenInput / tokenOutput / tokenThinking   Int?
  model / provider    String?
  failureReasonCode   String?
  agentRunId          String?
  attributes          Json?              // 结构化属性（GIN 索引可查）
  events              Json?              // 事件数组（快读）
}
```

关键点：

- **`traceId` 把一次执行的所有 span 串成一棵树**，`parentSpanId` 维护父子关系。查一次执行就是 `WHERE traceId = ? ORDER BY started_at`，树结构自然浮现。
- **`spanKind` 区分节点类型**（agent_run / turn / llm_call / tool_call），不是靠日志里的字符串猜。
- **`attributes` 带 GIN 索引**（`@@index([attributes], type: Gin)`），可以直接按 JSON 属性查"哪个 span 用了某个模型""哪个 span 失败码是 X"。

这是 retire-agent-execution-log 后的 SSOT（agent_execution_log 已无 model 定义，核实报告 ch23-29）。**一句 SQL 查一棵树，比 grep 十几个日志文件高一个维度。** 这是可重放性的第一个前提：你得先把"那一次执行"的结构完整、有序地捞出来。

---

## 第二件事：事件有全局序号，双写不丢不重

光有 span 树还不够。一个 span 内部可能发生很多事件（LLM 调用开始、流式 token、工具调用、结束），这些事件的**先后顺序**对复现至关重要——"是先收到 tool_call 再超时，还是先超时再收到 tool_call"是完全不同的 bug。

WinMatrix 用 **ExecutionSpanEvent 子表 + `[spanId, seq]` 唯一约束**保证事件有序且不重不丢（`schema.prisma:3041-3060`）：

```
model ExecutionSpanEvent {
  id        BigInt   @id @default(autoincrement())
  spanId    String
  eventType String             // llm_call_start / llm_call_end / tool_call_start ...
  phase     String?
  ts        DateTime @db.Timestamptz(6)
  attrs     Json
  seq       Int                // 全局递增序号
  @@unique([spanId, seq], map: "uq_span_event_span_seq")   // 防重
  @@index([spanId, ts])
  @@index([traceId, eventType])
}
```

这里有个容易被忽视的设计：**事件是双写的**——`execution_span.events`（JSON 数组，快读）和 `execution_span_event` 子表（完整事件源）。为什么要两份？

- **JSON 数组快读**：查一个 span 的事件不用 join，一次读出来就是有序数组，调试时直接看。
- **子表是完整事件源**：JSON 数组可能因为各种原因（compact、截断）不全，子表才是"真源"。`[spanId, seq]` 唯一约束保证同一个事件不会被写两次——重放时不会因为重复事件把顺序搞乱。

**事件的全局序号 + 双写**是可重放性的第二个前提：你捞出来的不是"一堆带时间戳的事件"（时间戳精度、时钟漂移会害死你），而是"一个严格有序、不重不丢的事件序列"。重放就是按 `seq` 升序回放。

---

## 第三件事：会话转录是"LLM 视角"的完整真源

Span 树是"系统视角"的执行记录（谁调用了谁、花了多久）。但复现一个 Agent bug，你还需要"LLM 视角"的记录——**LLM 在每一轮看到的是什么上下文、输出了什么、调了什么工具、想了什么（thinking）**。这是另一条线。

WinMatrix 的会话层有三张表，分工严格（核实报告 ch18-22）：

| 表 | 角色 | 内容 |
|----|------|------|
| conversation_histories | 读模型（展示用） | role / content / metadata，给前端列表 |
| **session_transcript** | **LLM 上下文真源** | entry_type / role / content / tool_name / tool_call_id / tool_result / tool_success / **thinking** / llm_purpose / llm_phase |
| conversation_meta | 权威元数据 | title / projectId / sessionMode / interactiveDigitalEmployeeId |

`session_transcript` 是这里的重中之重。schema 注释明写它是 **"Canonical message store，统一真源"**。它不只是"聊天记录"，它把**每一次 LLM 调用的完整输入输出链**都记下来了：

- `entry_type` 区分是用户消息、assistant 回复、tool_call、tool_result 还是 thinking 片段。
- `tool_name / tool_call_id / tool_result / tool_success` 把工具调用的入参出参完整留住。
- `thinking` 字段留住 LLM 的推理过程（这对调试"LLM 为什么决定调这个工具"是金矿）。
- `llm_purpose / llm_phase` 标注这次调用是干什么用的（决策规划 / 技能执行 / 摘要……），避免把不同用途的调用混在一起。

**为什么 transcript 和 span 树要分开存？** 因为它们服务不同的复现场景：

- Span 树回答"系统层面发生了什么"（哪个进程、哪个 span、多长时间）。
- Transcript 回答"LLM 层面看到了什么、做了什么"（prompt、completion、tool I/O）。

复现一个 Agent bug，你通常需要两者交叉看：从 span 树定位到"崩在哪个 LLM 调用"，再从 transcript 看那次调用的完整 I/O。这两个维度分开存、各自是 SSOT、通过 `conversation_id / run_id` 关联——这就是可重放性的第三个前提。

---

## 第四件事：真实事故变成可重放的 fixture

前三件事保证了"生产里发生的，数据库里都有"。但"都有"还不够，你还得能**把它喂回测试系统重跑**。这就是 decision-replay fixture 的作用。

WinMatrix 的测试目录里有一批**真实生产事故的回放样例**（`tests/fixtures/`，核实报告 ch23-29）：

```
tests/fixtures/
├── incident-2026-05-26-job-84711/      # 真实事故：某次 job 84711 的完整回放数据
├── incident-2026-05-26-job-84712/      # 同一天另一个 job
└── decision-planner-91695-replay/      # 决策规划器的某次回放
```

这些 fixture 不是人造的测试数据，而是**从生产库里导出的某次真实执行的 transcript + span + 输入快照**。它们的存在意味着：

1. **生产事故可以被"冻结"**：某次 bug 触发后，把相关的 transcript / span / 决策快照导出成 fixture，这个 bug 就被"钉"住了，不会因为生产数据被清理而消失。
2. **修复可回归**：修了 bug 后，把这个 fixture 喂给决策引擎重跑，看输出是不是符合预期。下次有人改了相关代码，CI 自动跑这些 fixture，保证不回归。
3. **决策路径可对比**：`decision-planner-91695-replay` 这种命名说明它专门复现"某次决策规划器跑了 91695 号请求"的路径，改了决策引擎后能精确对比"同输入下新旧版本决策是否一致"。

**这是可重放性的闭环**：生产事故 → 导出 fixture → 测试重放 → 修复验证 → CI 守护。没有前两步（span 树 + transcript），你导不出 fixture；没有这一步，导出的数据只能看不能用。

---

## 四件事凑在一起：一次完整的复现长什么样

把四件事串起来，一次"按原样再跑一遍"的复现流程是这样的：

```
用户报告："大福昨天拆项目卡住了"
        │
        ▼
[1] 查 ExecutionSpan 树
    SELECT spanId, spanKind, status, failureReasonCode, parentSpanId
    FROM execution_span WHERE traceId = ? ORDER BY started_at;
        │  定位到：崩在某个 llm_call span，failureReasonCode='round_budget_exceeded'
        ▼
[2] 查 ExecutionSpanEvent 子表（按 seq 升序）
    看这个 span 内事件的精确顺序：
    llm_call_start → tool_call_start → tool_call_end → llm_call_start → 超预算
        │  发现：第二轮 LLM 调用因为工具返回太多内容，token 超了预算
        ▼
[3] 查 session_transcript（按 conversation_id + run_id）
    看那次 LLM 调用的完整 I/O：
    - prompt 里注入了哪些记忆（发现一条无关的长记忆挤占了上下文）
    - LLM 的 thinking（"我决定再调一次工具细化"）
    - tool_result 的全文
        │  找到根因：记忆注入没做长度裁剪
        ▼
[4] 导出这次执行为 decision-replay fixture
    写入 tests/fixtures/incident-xxx/
    修复后 CI 自动重放，保证不再发生
```

四步走完，这个 bug 不只是"修了"，而是被**永久钉死在 fixture 库里**，成为系统行为契约的一部分。下次任何人改记忆注入、决策引擎、工具循环，CI 都会用这个 fixture 验证"那次事故不会重演"。

---

## 可重放性的代价：存储和保留策略

可重放不是免费的。span 树 + 事件子表 + transcript，一次复杂的多步执行可能产生几百条记录、几十 KB 的 JSON。生产环境下，这些数据指数级增长。WinMatrix 的处理方式是**分层保留 + 定时清理**（核实报告 ch23-29）：

```
system-observability-cleanup 任务（pattern: '30 4 * * *'）
  清理：execution_span（级联 execution_span_event）/ pipeline_run / session_transcript
```

每天凌晨 4:30 跑一次，按保留策略清理过期的可观测数据。注意它清理的是**三个表一起**——span 树、事件子表、transcript 是配套的，清的时候要级联，否则会留下"孤儿 transcript"（有 LLM 记录但没有 span 结构）。

**可重放性的一个隐含权衡是"保留多久"。** 保留太短，事故隔两天才发现就捞不到了；保留太长，存储成本扛不住。WinMatrix 的选择是"足够复现近期事故的窗口 + 真实事故导出为 fixture 永久保留"——常规数据按窗口清理，**真正有价值的事故通过 fixture 固化下来**，跳出保留窗口的限制。这是一个很漂亮的分工：数据库负责"近期待查"，fixture 库负责"长期回归"。

---

## 一个反直觉的点：可重放性 > 实时性

很多团队在设计可观测性时，第一反应是"我要实时 dashboard、实时告警"。这些当然重要，但**可重放性比实时性更底层、更值钱**。

原因是：实时性解决的是"现在出问题了赶紧知道"，可重放性解决的是"出问题后能查清楚、能修复、能保证不再发生"。一个系统可以没有实时告警（靠用户反馈也能知道出问题），但不能没有可重放性——没有它，你连"问题到底是什么"都搞不清楚，更别提修复。

而且实时告警的噪声问题几乎无解（告警太多 = 没有告警），但可重放性的价值是确定的：每多一个 fixture，系统的行为契约就多一条硬约束。**把投入往可重放性上倾斜，长期回报远高于堆实时 dashboard。**

---

## 给后来者的总结

1. **执行用树（ExecutionSpan），不要用平铺日志。** `traceId + parentSpanId` 构成树，`spanKind` 区分节点类型，`attributes` 带 GIN 索引可查。一次执行一棵树，查起来比 grep 高一个维度。
2. **事件要有全局序号（seq），双写到 JSON 数组 + 子表。** JSON 数组快读，子表是真源，`[spanId, seq]` 唯一约束防重。重放就是按 seq 升序回放，不要靠时间戳排序（时钟漂移会害死你）。
3. **会话转录（session_transcript）是 LLM 视角的真源，要和 span 树分开存。** Span 回答"系统层面发生了什么"，transcript 回答"LLM 看到了什么、做了什么"。两者通过 conversation_id / run_id 关联，各自是 SSOT。
4. **真实事故要能变成 fixture。** 生产事故 → 导出 transcript + span 快照 → 写入 tests/fixtures/ → 修复后 CI 重放。这是把"一次性事故"变成"永久行为契约"的唯一办法。
5. **保留策略要分层：常规数据按窗口清理，有价值的事故导出为 fixture 永久保留。** 清理 span / event / transcript 要级联，别留孤儿数据。
6. **可重放性优先于实时性。** 实时告警的噪声问题几乎无解，可重放性的价值是确定的。投入往可重放上倾斜。
7. **可重放是横切多条线的契约，不是某个模块的功能。** 它要求数据库模型（span 树 + transcript）、可观测性（事件 seq）、测试基础设施（fixture）从第一天就配合。事后补，成本是十倍。

一个系统成熟的标志，不是它能跑得多快，而是它出问题后能多快被"精确复现、定位、修复、保证不再发生"。这条线上每多投入一分，你深夜被叫醒的次数就少一次。这是工程哲学里最朴素的复利。

---

> **上一篇**：[《终态收敛：分布式系统里"未完成"比"失败"更危险》](./52-terminal-convergence.md)
>
> **下一篇**：[《透明代理并非透明：PgBouncer/连接池/中间件的隐含状态》](./54-transparent-proxy.md)——可重放解决了"执行能复现"，但有一种东西会悄悄破坏复现：你以为透明的代理，其实改了你的会话语义。下一篇讲怎么识别这些隐含状态。
