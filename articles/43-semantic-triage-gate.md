# 语义分诊门 SemanticTriageGate：闲聊、终止、bound、unbound 怎么分

> 这是《WinMatrix 开发经验文集》第 43 篇。在 [第 2 篇](./02-decision-engine.md) 讲决策引擎时，我们提过"准入门里有语义分诊"。但 SemanticTriageGate 到底分了哪几类、每一类的判断逻辑是什么、为什么有个 0.65 的低置信度阈值——这些细节一直没展开。这一篇就专门讲这道门。

想象一个场景。

用户在一个会话里刚跑完一个"代码评审"的多步编排（coordinator 模式），系统正在等他对评审结果做下一步。这时候他发了一句"算了不搞了"。

这句"算了不搞了"该怎么处理？

- 当成普通聊天？不行，它其实是要终止刚才的流程。
- 当成新的任务？也不对，它没有新需求。
- 续接刚才的评审？更不对，用户明明是想结束。

如果直接把这句话扔给决策引擎去做路由规划，不仅贵（要调 LLM），还会规划出错误的结果。在消息进入决策管线**之前**，得先有一道门，把它分类：是闲聊、是想终止、是想续接之前的执行、还是全新的需求。

SemanticTriageGate 就是这道门。

---

## 四分类：casual_chat / termination_intent / bound / unbound

先看它把消息分成哪几类。源码里定义得很清楚：

```typescript
// agents/core/agent/decision/SemanticTriageGate.ts（第 77-81 行）
export type SemanticTriageClassification =
  | 'casual_chat'
  | 'termination_intent'
  | 'bound'
  | 'unbound';
```

这四类的语义：

| 分类 | 含义 | 处理方式 |
|------|------|---------|
| `casual_chat` | 闲聊（"你好""谢谢"） | 短路，直接闲聊响应，不进决策管线 |
| `termination_intent` | 终止意图（"算了""停一下"） | 识别并终止进行中的编排 |
| `bound` | 绑定到已有执行（续接/追加/取消） | 续接之前的 agent_run，不新建 |
| `unbound` | 全新需求，不绑定任何已有执行 | 进正常决策管线 |

这四个分类不是随便分的，它们精确覆盖了"一条消息和已有执行的关系"的所有可能性：

```
一条新消息进来，SemanticTriageGate 分诊：

  ┌─ 和任何已有执行无关？
  │    ├─ 是闲聊（你好/谢谢）         → casual_chat（短路）
  │    └─ 是新需求（帮我写个 PRD）     → unbound（进决策管线）
  │
  └─ 和某个已有执行有关？
       ├─ 想终止它（算了不搞了）       → termination_intent
       └─ 想续接/修改/取消它           → bound
            └─ boundIntent 细分：
                 ├─ continue_retry（继续/重试）
                 ├─ append_modify（追加/修改）
                 ├─ cancel（取消）
                 └─ unknown（说不清）
```

注意 `bound` 还会带一个 `boundIntent` 子分类：

```typescript
// SemanticTriageGate.ts（第 93-98 行）
const BOUND_INTENT_VALUES = [
  'continue_retry',
  'append_modify',
  'cancel',
  'unknown',
] as const;
```

这个子分类决定了续接的具体动作——用户说"继续"和"把刚才的第三步改一下"，虽然都是 bound，但续接逻辑完全不同。

---

## casual_chat 短路：最便宜的优化

四类里最值得讲透的是 `casual_chat`。

用户在群里发个"你好""收到""谢谢"，这种消息如果进决策引擎，要走完五阶段管线（ExactRouter → FusionRouter → QuickAcceptGate → DecisionPlanner → CommitmentDeriver），即使每阶段都能短路，也是几十毫秒到几百毫秒的开销。而且 LLM 可能把它误判成某种意图，给出奇怪的响应。

SemanticTriageGate 在**准入门阶段**（Point A）就把它识别出来，直接短路——不走路由、不进决策、不创建 agent_run。响应几乎是即时的。

这套思路和 [第 2 篇](./02-decision-engine.md) 讲的 SimpleChatGuard 类似，但位置不同——SimpleChatGuard 在决策引擎 Stage 1，SemanticTriageGate 在准入门更前置。**越早识别闲聊，省下的成本越多。**

为什么不只用 SimpleChatGuard？因为 SemanticTriageGate 还要同时识别 bound/termination，这两件事 SimpleChatGuard 不管。把四分类收口在一个地方，比散落在多个组件清晰。

---

## LOW_CONFIDENCE_THRESHOLD = 0.65：宁可放过

SemanticTriageGate 用 LLM 做分类判断（structured output），LLM 会返回一个置信度 `confidence`。这里有个关键阈值：

```typescript
// SemanticTriageGate.ts（第 108 行）
const LOW_CONFIDENCE_THRESHOLD = 0.65;
```

还有几个相关的硬约束：

```typescript
// SemanticTriageGate.ts（第 102-109 行）
const TERMINATION_TIMEOUT_MS = 10_000;     // 终止判断 10s 超时
const MAX_USER_MESSAGE_CHARS = 500;        // 用户消息截断到 500 字
const MAX_SUMMARY_CHARS = 300;
const MAX_CANDIDATES = 5;                  // 最多 5 个候选绑定对象
```

0.65 这个阈值的设计哲学是**宁可放过，不可误判**。置信度低于 0.65 时，SemanticTriageGate 不强行给出分类，而是降级为 fallback——把消息当成普通需求进决策管线，让更强的 LLM 规划来兜底。

为什么是"宁可放过"？因为误判的代价不对称：

- **误把闲聊当成需求**：最多多花点算力走决策管线，最后给个正常响应，用户感觉不到大问题。
- **误把需求当成闲聊**：用户说"帮我重构这个模块"被当成闲聊短路了，系统回个"好的没问题"——用户的需求被彻底忽略，这是体验灾难。
- **误把新需求当成 bound**：用户明明是要开始新任务，却被续接到一个不相关的旧执行上，行为完全错乱。

所以当置信度不够时，**选择最保守的"放过"**——当成 unbound 进正常管线。即使管线贵一点，至少不会错。这是个典型的"不对称损失下选保守策略"的工程决策。

10 秒超时（`TERMINATION_TIMEOUT_MS`）也是同理——LLM 分类卡住时，不能让用户一直等，降级放过比卡死强。

---

## bound 判定：怎么知道用户想续接哪个执行

bound 是四分类里最复杂的一类。它要回答两个问题：

1. 用户这句话是不是在说某个**已有的执行**？
2. 如果是，是哪个执行？

这就需要"候选卡"（Candidate Card）。SemanticTriageGate 会为当前会话构建一组候选，每个候选代表一个可能被续接的执行：

```typescript
// SemanticTriageGate.ts（第 165-179 行）
export interface SemanticTriageCandidateCard {
  candidateId: string;
  agentRunId: string;
  objectType: SemanticBindingObjectType;  // agent_run | worker_result | artifact | message | error
  objectId: string;
  status: string;
  recencyRank: number;                     // 近因排名（越近越靠前）
  source: SemanticTriageCandidateSource[]; // agent_run | checkpoint | message_metadata | memory
  originalGoal?: string;
  finalOutputSummary?: string;
  errorSummary?: string;
  artifactSummaries?: Array<{ id: string; type: string; summary: string }>;
  workerResultSummaries?: Array<{ briefId: string; workerId?: string; summary: string }>;
  semanticText: string;                    // 拼出来的语义文本，供 LLM 判断
}
```

每个候选聚合了这个执行的**关键信息**：原目标是什么、产出什么、有没有报错、有哪些 artifact。这些信息拼成一段 `semanticText`，连同用户消息一起喂给 LLM，让它判断"用户这句话是在说哪个候选"。

几个设计要点：

**候选来源有多种**（`SemanticTriageCandidateSource`）。一个候选可能从 agent_run 表来，也可能从 checkpoint（编排检查点）、message_metadata（消息元数据）、memory（长期记忆）来。**多来源采集候选，避免漏掉可能的续接对象。**

**`recencyRank` 近因排名**。用户更可能续接最近的任务而不是很久以前的。近因排名影响候选的呈现顺序和权重。

**最多 5 个候选**（`MAX_CANDIDATES = 5`）。候选太多会让 LLM 判断变慢、变贵、变不准。截断到 5 个，优先保留最近的、最相关的。

**`semanticText` 是精炼摘要，不是全文**。artifact 摘要、worker result 摘要都有长度限制（`MAX_CANDIDATE_SEMANTIC_TEXT_CHARS = 900`、`MAX_CANDIDATE_FIELD_CHARS = 300`）。把执行的全量信息塞给 LLM 既贵又干扰判断，精炼摘要才是正解。

### 绑定结果：boundIntent 细分

LLM 判断"用户在说候选 X"之后，还要判断"用户想对 X 做什么"。这就是 `boundIntent`：

```
用户消息："把刚才评审的第三步重做一下"
   │
   ▼
SemanticTriageGate 构建 5 个候选（最近的 5 个 agent_run）
   │
   ▼
LLM 判断：绑定到候选 #2（刚才的代码评审），boundIntent = append_modify
   │
   ▼
返回 SemanticTriageDecision:
  kind: 'bound'
  targetAgentRunId: 'run_xxx'
  boundIntent: 'append_modify'
  confidence: 0.88
```

下游拿到这个结果，知道该续接到 `run_xxx`，并且是"修改第三步"而不是"从头重来"。这比单纯说"这条消息和某个 run 有关"信息量大得多。

---

## 完整的 SemanticTriageDecision 结构

分诊结果的结构，把所有可能性都覆盖了：

```typescript
// SemanticTriageGate.ts（第 199-217 行）
export type SemanticTriageDecision =
  | { kind: 'casual_chat'; confidence: number; reason: string }
  | { kind: 'termination_intent'; confidence: number; reason: string }
  | {
      kind: 'bound';
      targetAgentRunId: string;
      targetObject?: SemanticBindingTarget;
      contextDigest: SemanticBindingContextDigest;
      confidence: number;
      reason: string;
      boundIntent?: SemanticBoundIntent;
    }
  | {
      kind: 'unbound';
      confidence: number;
      reason: string;
      fallbackReason?: string;
      schemaViolation?: string;
    };
```

注意 `unbound` 分支带了 `fallbackReason` 和 `schemaViolation`。这两个字段记录"为什么当成 unbound"——是 LLM 真的判断它是新需求，还是置信度太低降级了，还是 LLM 输出格式不合 schema 被丢了。

**把降级原因留在决策里**，事后排查"为什么这条消息没被识别为 bound"时才知道是阈值问题还是 LLM 输出问题。这是可观测性的细节功夫。

---

## SemanticTriageGate 在管线里的两个点位

SemanticTriageGate 不是只调一次。它在 Turn 生命周期里有两个触发点：

```
用户消息进来
   │
   ▼
TurnRunner.run
   │
   ▼
runPreRouteGates（准入门）
   │
   ├─ SemanticTriageGate Point A（路由前）
   │    casual_chat → 短路，直接闲聊响应
   │    termination_intent → 终止进行中的编排
   │    bound → 续接对应 agent_run
   │    unbound → 继续
   │
   ▼
resolveRoute（路由决策）
   │
   ▼
assembleExecution（执行）
   │
   ├─ SemanticTriageGate Point B（执行中，特定条件下触发）
   │    比如 coordinator 编排完成后的续接判断
```

**Point A** 在路由前，是最主要的分诊点。闲聊在这里短路，省下整条决策管线。

**Point B** 在执行中，处理"编排跑到一半，用户回来续接"的场景。比如一个多步编排跑到第 3 步挂起等待，用户回来发消息，Point B 判断他是在说这个挂起的编排，还是要开始别的。

这两个点位不重复——Point A 管的是"新消息怎么进门"，Point B 管的是"执行中怎么处理续接"。各自有触发条件，互不干扰。

---

## 输入归一化：截断与摘要

SemanticTriageGate 的输入有几个限制，值得单独说一下：

```typescript
// SemanticTriageGate.ts（第 103-109 行）
const MAX_USER_MESSAGE_CHARS = 500;
const MAX_SUMMARY_CHARS = 300;
const MAX_SESSION_SUMMARY_CHARS = 2000;
```

用户消息超 500 字会被截断（`truncateUserMessage`）。为什么要截断？

- **分类不需要全文**。"算了不搞了"和"算了不搞了，我刚才想了想这个方案还是有问题，主要是因为..."，对分类判断来说，前半句就够了。
- **长消息干扰判断**。用户洋洋洒洒写 2000 字，LLM 反而可能在细节里迷失，抓不住主要意图。
- **省钱**。分类用的 LLM 调用按 token 计费，截断直接省钱。

会话摘要（`MAX_SESSION_SUMMARY_CHARS = 2000`）是给 LLM 看的"这个会话之前发生了什么"的上下文。2000 字足够覆盖最近几轮交互，又不至于让 prompt 膨胀。

**给分类 LLM 的输入要精简，不是越多越好。** 这是用 LLM 做分类的经验——你给它全文它反而分类不准，给它提炼过的关键信息它判断更稳。

---

## 几个工程取舍

最后总结 SemanticTriageGate 里几个不那么显眼但重要的设计取舍。

### 为什么用 structured output 而不是自由文本

SemanticTriageGate 调 LLM 用的是 `completeChatStructured`（structured output），返回严格符合 schema 的 JSON。为什么不用自由文本再解析？

因为**分类需要下游程序消费**。返回 `{kind: 'bound', targetAgentRunId: 'run_xxx', boundIntent: 'append_modify'}`，下游代码能直接用。如果返回自然语言再解析，解析会失败、会出错、会引入额外复杂度。structured output 把"LLM 的语义判断"和"程序的确定消费"连起来，是最可靠的方式。

### 为什么候选要 digest

`SemanticBindingContextDigest` 是候选的摘要（带 hash、字符数、来源列表）。它不是给 LLM 看的（LLM 看的是 semanticText），而是给**观测和审计**看的。

分诊决策记录里带上 contextDigest，事后能知道"当时 LLM 是基于哪些候选、多少上下文做的判断"。如果分错了，能查到是候选采漏了还是上下文太短。**可观测性不能只记结果，要记"做决策时看到了什么"。**

### 为什么 termination 单独一类

"终止"看起来是 bound 的一种（绑定到某个执行并 cancel），为什么要单独成一类？

因为终止的**处理路径不同**。bound 要续接执行，终止要**中止**执行——这两个动作相反。终止可能还要触发清理（释放工作站租约、中止 LLM 调用、回写 agent_run 状态）。把它单独分出来，下游处理更清晰，不用在 bound 的 cancel 分支里塞一堆"其实是终止"的特殊逻辑。

---

## 给后来者的总结

1. **SemanticTriageGate 是决策管线前的分诊门，四分类覆盖消息与已有执行的所有关系**。casual_chat（闲聊短路）/ termination_intent（终止）/ bound（续接已有）/ unbound（全新需求）。每类的处理路径不同。

2. **bound 带 boundIntent 子分类**（continue_retry / append_modify / cancel / unknown）。续接不是铁板一块，"继续"和"修改第三步"是不同动作。

3. **LOW_CONFIDENCE_THRESHOLD = 0.65 是"宁可放过"的阈值**。置信度不够时降级为 unbound 进正常管线，而不是强行分类。误判损失不对称时，选保守策略。

4. **候选卡（Candidate Card）多来源采集、最多 5 个、带近因排名**。agent_run / checkpoint / message_metadata / memory 多来源不漏；截断到 5 个避免 LLM 判断失准。

5. **给分类 LLM 的输入要精简**。用户消息截断 500 字、候选 semanticText 900 字、会话摘要 2000 字。全文反而让分类不准。

6. **用 structured output 而非自由文本**。分类结果要被程序消费，严格 schema 比文本解析可靠。

7. **termination 单独一类而非 bound 的 cancel 分支**。终止和续接的处理路径相反（中止 vs 继续），分开更清晰，避免在 bound 里塞特殊逻辑。

8. **降级原因要留痕**（fallbackReason / schemaViolation）。事后排查"为什么没识别为 bound"全靠它。可观测性要记"做决策时看到了什么"，不只是结果。

9. **两个触发点位**。Point A 在路由前（最主要的分诊），Point B 在执行中（续接挂起的编排）。互不重复。

SemanticTriageGate 解决的是"消息进来第一瞬间的分流"问题。它看似只是个前置优化，但实际上决定了"这条消息要不要走重管线、走哪条管线"。把这道门做扎实，决策引擎的负载能降一大截，用户体验也更顺——闲聊秒回、续接准确、终止干净。

---

> **下一篇**：[《五种执行模式：interactive / coordinator / react / skill / workstation 怎么选》](./44-execution-modes.md)——分诊完了，消息进决策管线，决策结果会决定"用哪种模式执行"。同样是"让数字员工干活"，为什么要有五种模式？它们各自的状态机和终止条件有什么不同？
