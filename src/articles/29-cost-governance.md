# LLM 成本治理：把每一分 token 归因到员工、技能、项目

> 这是《WinMatrix 开发经验文集》第 29 篇，也是横切主题的第二篇。做 Agent 平台，技术话题里有一个最容易被工程师忽略、却最容易被老板盯上的东西——成本。这篇讲我们怎么把"花了多少 token"从一个模糊的总数，变成可以按员工、按技能、按项目拆账的明细。

如果你的 Agent 平台已经上了生产，你一定收到过这样的问题：

> "这个月 LLM 账单三万八，比上个月翻了一倍——钱到底花在哪了？"

你想回答，却发现自己答不上来。你有一个月度的 API 用量汇总（provider 给你的），但那只是一个数字。哪个数字员工烧的？哪个技能烧的？哪个项目烧的？——全都查不出来。

这就是 LLM 成本治理的核心痛点：**API 账单是平的，但你的成本结构是立体的。** 不把它拆开，你既无法做预算、也无法做优化、更无法向业务方解释"为什么这个项目要用这么多额度"。

这篇讲 WinMatrix 怎么用 ExecutionSpan 做 token 归因，怎么用 mini 模型蒸馏省钱，怎么用语义缓存复用决策——三条独立的成本治理路径。

---

## 第一条路径：把 token 归因到"谁花的"

成本治理的第一步不是省钱，是**看清楚钱花在哪**。这一步做不到，后面所有优化都是瞎拍。

我们做这件事的基础设施，是上一篇（第 8 篇）讲过的 ExecutionSpan。这里只看其中和成本最相关的字段：

```prisma
// prisma/schema.prisma（第 2990-3038 行）
model ExecutionSpan {
  spanId              String    @id @map("span_id")
  traceId             String    @map("trace_id")
  parentSpanId        String?   @map("parent_span_id")   ← 父节点，构成树
  spanKind            String    @map("span_kind")          ← 节点类型
  status              String    @default("pending")
  tokenInput          Int?      @map("token_input")       ← 本次调用输入 token
  tokenOutput         Int?                                 ← 本次调用输出 token
  tokenThinking       Int?                                 ← 思考 token（推理模型）
  model               String?                              ← 用的哪个模型
  provider            String?                              ← 哪家 provider
  agentRunId          String?   @map("agent_run_id")      ← 属于哪次执行
  attributes          Json?                                ← 自由属性
  @@index([traceId])
  @@index([spanKind, status])
  @@index([attributes], type: Gin)   ← GIN 索引支持 JSON 查询
}
```

注意三个字段：`tokenInput` / `tokenOutput` / `tokenThinking`。每一次 LLM 调用，只要走过我们的遥测契约（`emitLLMCallStart` / `emitLLMCallEnd`），这三个值就会被记到对应的 `llm_call` span 上。

但光记到 span 上还不够。一个 span 只知道"这次调用花了 1200 input / 380 output token，用的是 claude-sonnet"，它不知道这次调用是**谁触发**的。要回答"谁花的"，得把 span 树往上回溯。

### 归因链路：从 llm_call span 回溯到员工

一棵执行树长这样：

```
agent_run（根，executor_employee_name = 小品）
├── decision_span（决策阶段，无 LLM token）
├── turn_span
│   ├── llm_call #1（tokenInput=1200, tokenOutput=380, model=claude-sonnet）
│   ├── tool_call: rag_search
│   │   └── llm_call #2（查询改写，tokenInput=180, tokenOutput=60, model=claude-haiku）
│   ├── llm_call #3（tokenInput=2100, tokenOutput=520, model=claude-sonnet）
│   └── tool_call: write_doc
└── result_persist_span
```

关键点：`llm_call` 是叶子，但通过 `parentSpanId` 一路向上，能找到它所属的 `agent_run`，而 `agent_run` 上带着 `executor_employee_name`（执行它的数字员工）、`conversationId`（所属会话）、会话再关联到 `projectId`（所属项目）。

这就构成了完整的归因链：

```
llm_call tokenInput/Output
    ↑ parentSpanId
turn_span
    ↑ parentSpanId
agent_run（executor_employee_name, conversationId, projectId）
```

所以，一句 SQL 就能回答"这个月哪个员工烧钱最多"：

```sql
-- 按员工聚合 token（伪 SQL，表达归因逻辑）
SELECT ar.executor_employee_name,
       SUM(s.token_input)  AS total_in,
       SUM(s.token_output) AS total_out,
       SUM(s.token_input + s.token_output) AS total_tokens
FROM execution_span s
JOIN agent_run ar ON ar.id = s.agent_run_id
WHERE s.span_kind = 'llm_call'
  AND s.started_at >= '2026-08-01'
GROUP BY ar.executor_employee_name
ORDER BY total_tokens DESC;
```

同样地，把 `GROUP BY` 换成项目（`ar.project_id`）、技能（从 `attributes` 里取 `skillName`）、模型（`s.model`），就能得到四个维度的成本视图：**员工维度、项目维度、技能维度、模型维度**。

### 为什么这件事必须用 span 做，不能用 API 账单做

很多人会想：provider 不也给我用量明细吗？为什么要自己搞？

provider 的账单有三个硬伤：

1. **它只知道总账，不知道细账**。"今天调了 8000 次 claude-sonnet，花了 X 元"——但你不知道这 8000 次里，有多少是小品写 PRD 的、有多少是阿码做代码评审的、有多少是 RAG 查询改写的。
2. **它有延迟**。月底才出账，等你发现某个项目异常烧钱时，可能已经烧了一个月。
3. **它不知道业务语义**。provider 不知道什么是"员工"、什么是"项目"、什么是"技能"——这些是你的业务概念，只有你的系统能赋予 token 以业务含义。

**token 归因的本质，是把"基础设施维度的成本"翻译成"业务维度的成本"。** 这个翻译只有系统自己做得到，provider 帮不了你。

---

## 第二条路径：mini 模型蒸馏，把"贵"的活儿挪给"便宜"的

看清了成本结构，下一步就是优化。一个很自然的发现是：**不是所有 LLM 调用都需要用最贵的模型。**

WinMatrix 里有一类典型的"不需要贵模型"的活儿——技能蒸馏。

### 什么是技能蒸馏

一个技能（比如"写 PRD"）被数字员工执行了很多次。每一次执行都会留下轨迹（SkillTrace）：用了哪些工具、调了几轮 LLM、每轮干了什么、最终成功没。这些轨迹是金矿——它记录了"这个技能实际是怎么被干成的"。

但轨迹本身是一堆原始日志，不能直接复用。我们的做法是用一个 **mini 模型**（便宜的小模型）把多条轨迹压缩成一份"执行指南"（SkillExecGuide）：

```typescript
// agents/harness/learning/distillation/SkillKnowledgeDistiller.ts（第 46-139 行）
// distill()，用 mini 模型 purpose='skill_distill'
const guide = await distiller.distill({
  skillContent,        // 技能正文
  traces,              // 最近 10 条成功轨迹
  visibleToolNames,    // 这个技能可见的工具集
  roleId: data.roleId,
  skillName: data.skillName,
  skillSource: data.skillSource,
  skillContentHash, toolSetHash: traceRows[0]?.toolSetHash ?? undefined,
  traceDurationsMs,
});
```

关键细节在 `purpose='skill_distill'` 这个标记。我们的 LLM 工厂（`llmFactory`）会按 purpose 路由到不同的模型——`skill_distill` 这种内部任务路由到便宜的 mini 模型，而面向用户的对话走更强的模型。

为什么这件事能省钱？因为蒸馏是个**离线、批量、低风险**的任务：

- **离线**：它不在用户请求路径上，慢一点没关系。
- **批量**：一次处理 10 条轨迹，用 mini 模型跑一遍，产出的执行指南可以被成千上万次后续执行复用。
- **低风险**：蒸馏产出的是"指南"不是"命令"，指南质量差点，运行时还有 L1/L2/L3 readiness gate（见第 6 篇）兜底，不会出大错。

所以它天然适合用便宜模型。**用 sonnet 干蒸馏，是用牛刀杀鸡；用 mini 模型干蒸馏，才是成本和质量的平衡点。**

### 蒸馏产出的 SkillExecGuide 怎么反向省钱

蒸馏产出的 `SkillExecGuide`（schema 第 2612-2651 行）不只是给人看的文档，它会被注入回技能执行时的上下文，让后续执行更高效：

```
SkillExecGuide 字段（节选）：
  coreTools        ← 这个技能核心会用到的工具（优先准备）
  optionalTools    ← 可能用到但不一定用的
  executionGuide   ← 执行指南正文（@db.Text）
  paramRules       ← 参数填写规则
  dataFlow         ← 数据怎么流转
  pitfalls         ← 常见坑
  confidence       ← 置信度
  avgDurationMs    ← 平均耗时
```

有了这份指南，一个技能下次执行时，LLM 不用从零开始"想"该怎么干——指南直接告诉它"先调这个工具、再调那个工具、参数这么填、注意这个坑"。**这意味着同样的任务，用更少的 LLM 轮次、更短的 prompt 就能完成。**

而"更少轮次、更短 prompt"直接等于"更少 token"。这是蒸馏的第二层省钱：不只是蒸馏本身用便宜模型，**蒸馏的产出让所有后续执行都变便宜了**。

蒸馏还有一道"消毒"工序（`SkillGuideSanitizer`），会审计产出的指南，剔除不该出现的工具、修正不当建议（`sanitizedAt` / `removedTools` / `wasFiltered` 字段记录消毒痕迹）。这是必要的——mini 模型的产出不能盲信，必须有一道独立的校验。

---

## 第三条路径：语义缓存，让相似的请求不重复花钱

第三条省钱路径，是复用历史决策。

用户问"帮我写个 PRD"和"我要写一份产品需求文档"，对决策引擎来说是**同一个决策**——都应该路由到小品、走 prd_writing 技能。如果第一次已经用 LLM 规划过了，第二次为什么还要再花一次 LLM 的钱？

这就是语义缓存（Semantic Planner Cache）。它埋在决策引擎的 Stage 3（QuickAcceptGate）：

```typescript
// agents/core/agent/decision/DecisionEngine.ts（第 503-563 行）
// Stage 3: QuickAcceptGate (fast-accept / cache pre-check)
const coverageEval = await this.quickAcceptGate.evaluate(
  { candidates: extractedCandidates, userInput: input.userInput, ... },
  { ... },
  {
    validatePatch: ...,
    skillReferenceCatalog,
    cacheLookup: (params) => this.lookupSemanticPlannerCache(
      params.userInput, input, exactMatch, skillReferenceCatalog
    )
  },
);
```

`cacheLookup` 做的事：**这个用户输入，以前是不是决策过？** 用 embedding 相似度 + inputFingerprint（输入指纹）比对，如果发现"半年前有个几乎一样的问题，当时决策成了 X"，直接复用那个决策，**跳过 Stage 4 的 LLM 规划**。

语义缓存命中意味着什么？意味着一次 LLM 规划调用（通常是最贵的单次调用，因为 prompt 最长、要输出结构化计划）被完全省掉了。

### 语义缓存的成本账

我们算笔账。假设一个企业每天有 1000 条用户消息：

- 80% 在 Stage 1（精确路由/闲聊守卫）被解决，零 LLM 成本——这部分和缓存无关。
- 剩下 20%（200 条）进入 Stage 2/3/4。其中假设 30%（60 条）能被语义缓存命中。
- 每次完整 LLM 规划假设花 0.3 元（input 2000 + output 800 token，sonnet 价格）。
- 一天省 60 × 0.3 = 18 元，一个月省 540 元。

听起来不多？但这只是"规划"这一步。语义缓存的思路可以推广到工具调用结果、RAG 检索结果——凡是"相同输入应该得到相同输出"的 LLM 调用，都能缓存。**积少成多，缓存往往是 LLM 成本优化里 ROI 最高的手段**，因为它不改变任何业务逻辑，纯粹是"不重复劳动"。

### 语义缓存的边界

但语义缓存不是万能的，它有三个边界：

1. **时效性边界**。缓存的决策可能过期——技能配置变了、路由规则变了、业务上下文变了。所以缓存必须带 TTL 和失效机制，不能永久复用。
2. **语义相似 ≠ 意图相同**。"帮我写 PRD"和"帮我改 PRD"语义很像，但一个是新建一个是修改。所以缓存命中要有阈值（embedding 相似度 + inputFingerprint 双重校验），宁缺毋滥。
3. **上下文相关性**。同一个用户输入，在不同项目、不同会话上下文里，正确的决策可能不同。缓存 key 不能只看输入文本，还得带上上下文维度。

这三条边界决定了语义缓存只能是"锦上添花"，不能是"唯一依赖"。它是渐进式决策管线里的一道短路（见第 2 篇），命中了省钱，没命中照常走 LLM。

---

## 三条路径的关系：归因是基础，优化是上层

把三条路径放一起看，它们是一个递进的结构：

```
                    ┌─────────────────────────┐
                    │  第三层：语义缓存复用    │  ← 不重复花钱
                    │  （跳过 LLM 规划）       │
                    └────────────┬────────────┘
                                 │ 命中率不够时
                    ┌────────────▼────────────┐
                    │  第二层：mini 模型蒸馏   │  ← 让每次更便宜
                    │  （产出执行指南复用）     │
                    └────────────┬────────────┘
                                 │ 所有调用
                    ┌────────────▼────────────┐
                    │  第一层：token 归因       │  ← 先看钱花哪了
                    │  （ExecutionSpan）        │
                    └─────────────────────────┘
```

**归因是基础**——你得先知道钱花在哪，才能判断该从哪里下手。归因告诉你"70% 的成本在 sonnet、其中阿码的代码评审占了一半"，你才知道该优化代码评审这个技能。

**蒸馏是结构性优化**——它把"贵的离线活儿"挪给便宜模型，并产出能反向降低在线调用成本的指南。

**缓存是即时性优化**——它不改变任何结构，只是让重复的劳动不再发生。

三条路径要配套用：只做归因不优化，等于光记账不节流；只做缓存不归因，等于瞎省一通不知道省了多少、也不知道该继续省哪。**成本治理是一个闭环——归因发现热点，优化降低热点，归因再验证效果。**

---

## 一个常被忽略的点：归因也要防"悬挂"

第 8 篇讲过一个生产坑——悬挂的 LLM 调用。`llm_call_start` 记了，但 `llm_call_end` 永远不来（网络抖动、进程崩溃、超时）。这个坑在成本治理里特别致命：

**一个悬挂的 llm_call span，它的 `tokenInput` / `tokenOutput` 可能是 null 或 0（因为 end 没来，token 数还没回填）。** 如果你直接 `SUM(token_input)`，这些悬挂调用要么被漏算（null），要么被低估（0），导致成本统计失真。

我们的解法是 `llmCallWatchdogSweeper`（每 10 分钟跑一次）：

```typescript
// infrastructure/scheduled/llmCallWatchdogSweeper.ts（第 1-53 行）
export const LLM_CALL_WATCHDOG_TASK_NAME = 'system-llm-call-watchdog-sweeper';
function resolveSweepThresholdMs(): number {
  const hardMs = getConfig().llmCallHardTimeoutMs;
  return hardMs > 0 ? hardMs * 2 : 360_000;   // 2x hard timeout 或 6 分钟
}
```

它扫出超时未结束的调用，**补写 `llm_call_end`**，并级联 finalize 关联的 agent_run / scheduled_task_run。对于成本治理来说，它的意义是：**保证归因数据的完整性**——悬挂调用要么被估算 token（按超时阈值估算最大可能的 token），要么被标记为失败（不计入正常成本），但不会以"null token"的形式污染统计。

**任何成本归因系统，都必须有悬挂检测兜底，否则你的账永远对不齐。**

---

## 给后来者的几条总结

1. **token 归因是成本治理的第一步**。provider 账单是平的，你的成本是立体的。不拆开看，优化无从谈起。
2. **归因靠 span 树回溯**。llm_call 是叶子，通过 parentSpanId 回溯到 agent_run，拿到员工/项目/技能的业务语义。ExecutionSpan 是 SSOT。
3. **四个维度的成本视图**：员工、项目、技能、模型。一句 GROUP BY 切换维度。
4. **mini 模型蒸馏是结构性省钱**。离线、批量、低风险的任务用便宜模型，产出还能反向降低在线调用成本。
5. **语义缓存是即时性省钱**。相似请求复用历史决策，跳过 LLM 规划。ROI 最高，但有阈值和失效边界。
6. **三层递进**：归因（看钱花哪）→ 蒸馏（让每次更便宜）→ 缓存（不重复花钱）。配套用才闭环。
7. **归因数据要防悬挂**。llmCallWatchdogSweeper 补写悬挂调用的终态，保证 token 统计完整。
8. **成本治理是闭环**。归因发现热点 → 优化降低热点 → 归因再验证效果。只做一层都不够。

做企业级 AI 产品，成本不是"等账单来了再算"，而是从第一天就要有归因能力。ExecutionSpan 那套可观测基础设施，既是调试的底座，也是成本治理的底座——一份投入，两份回报。

---

> **下一篇**：[《热更新与零停机：配置、技能、路由规则都能热加载》](./30-hot-reload.md)——成本讲完了，接着讲另一个横切主题：怎么做到改配置不重启、发技能不中断、灰度路由规则不重启进程。
