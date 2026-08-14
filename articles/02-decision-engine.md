# 让 AI 自己决定做什么：渐进式决策引擎的三层边界

> 这是《WinMatrix 开发经验文集》第 2 篇。一条用户消息进来，该让哪个员工响应、用哪个技能、走什么流程？这个"决定"看似简单，做不好要么烧钱、要么不稳。讲讲我们在 WinMatrix 里怎么做"渐进式决策"。

一个用户在 WinMatrix 里发了句"帮我写个 PRD"。系统要做的第一个判断，不是"怎么写 PRD"，而是——**这条消息，该交给谁处理？**

候选答案有很多：

- 交给"小品"（产品管理人员）？她最擅长 PRD。
- 还是走一个通用的"PRD 编写"技能？
- 还是用户其实在闲聊，根本不需要走完整流程？

这个判断，我们叫它"决策"。它是 Agent 系统的入口，也是最容易被做歪的地方。

---

## 最直觉的方案：一锤子 LLM 调用

很多人会想：这有什么难的？把所有员工、所有技能的描述喂给 LLM，让它决定不就行了？

```python
decision = llm.chat(
    "你有这些员工：大福、阿宁、小品、阿码……\n"
    "这些技能：prd_writing、code_review、daily_report……\n"
    "用户说了：'帮我写个 PRD'。请决定由谁、用什么技能处理。"
)
```

听起来很美。我们一开始也这么想过。但很快撞上三堵墙：

**第一堵墙：贵。** 每条消息都过一遍完整的 LLM 决策，token 成本和延迟都顶不住。一个高频使用的系统，光"决定谁处理"这一步，LLM 账单就能吓死人。

**第二堵墙：不稳。** 同样的输入，LLM 今天让小品处理，明天可能让阿码处理。调试的时候你抓狂——"昨天还好好的，今天怎么路由错了？" 回归测试更是噩梦，因为决策结果根本不可重复。

**第三堵墙：没必要。** 绝大多数请求其实是模式化的。"生成日会"就该走日会技能，"写 PRD"就该找小品——这些根本不需要 LLM 来"决定"，规则匹配就够了。

---

## 真正的方案：渐进式管线

解决方案是**一条分阶段的管线，从最便宜、最确定的阶段开始，逐渐升级到昂贵的 LLM**。每一阶段都能"短路"——一旦它能确定答案，就直接返回，不往下走。

```
Stage 1: 精确路由（零 LLM 成本）
   ↓ 没命中
Stage 2: 融合路由（轻量，规则+语义）
   ↓ 不确定
Stage 3: 快速接受（语义缓存命中？）
   ↓ 没命中缓存
Stage 4: LLM 规划（完整决策，最贵）
   ↓
Stage 5: 承诺派生（校验、约束、落地）
```

这就是 WinMatrix 决策引擎的真实结构。让我一层层拆。

### Stage 1：精确路由 + 闲聊守卫

最便宜的一层。它的逻辑是纯确定性的：

- **精确路由（ExactRouter）**：用户说"生成今日日会"，关键词直接命中"日会技能"，零成本确定。不需要任何 LLM。
- **闲聊守卫（SimpleChatGuard）**：用户说"你好"、"谢谢"、"在吗"——这些是闲聊，直接走轻量 chat，不进任何技能流程。同样零 LLM 成本。

这一层还有一个巧妙的短路——**显式流程编排**。如果用户明确说"我要做 A 然后 B 再 C"这种多步流程，系统直接把它编译成一个流程（见第 20 章），跳过后续所有路由。

```typescript
// agents/core/agent/decision/DecisionEngine.ts（第 425-439 行）
// ExplicitFlowOrchestration: 用户显式描述多步流程时直接编译，跳过后续路由与 LLM 规划
const explicitFlowOutcome = this.tryExplicitFlowOrchestrationPlan(
  input, resolvedTurn, exactMatch, emptyCandidates, draft
);
if (explicitFlowOutcome?.terminal) {
  explicitFlowOutcome.plan.boundRoute = 'explicit_flow_orchestration';
  return this.finalize(explicitFlowOutcome);
}
```

**这一层的核心思想：能用确定性规则解决的，绝不上 LLM。** 80% 以上的日常请求在这里就被解决了，根本到不了昂贵的下游。

### Stage 2：融合路由（FusionRouter）

精确路由没命中？进融合路由。这一层开始用一点"智能"，但还不是完整 LLM。

我们有一张 `route_rule` 表，每条规则描述"什么样的输入该走哪条路由"。一条规则大概长这样：

- `patterns`：正则模式（如 `.*写.*PRD.*`）
- `positiveIntents` / `negativeIntents`：正向/负向意图词
- `semanticAnchors`：语义锚点（向量化后的参考句）
- `semanticThreshold`：语义相似度阈值（默认 0.85）
- `roleId` / `skillName`：命中后路由到哪个角色/技能

FusionRouter 对输入做**多信号融合打分**：

```typescript
// agents/core/agent/decision/fusion-router.ts（第 165-212 行）
route(input: string): RouteResult | null {
  let bestRoute: RouteEntry | null = null;
  let bestScore = 0;
  for (const route of this.routes) {
    const score = this.computeScore(route, input);
    if (score >= route.semanticThreshold) {
      if (route.status === 'shadow') {       // 影子规则只记录，不路由
        this.bumpMetrics(route.id); this.shadowHits.push({...}); continue;
      }
      if (score > bestScore) {
        bestScore = score; bestRoute = route;
        // 命中方法：regex / fusion（意图词）/ semantic（语义）
      }
    }
  }
  // ...
}
```

三个细节值得说：

**多信号打分**。一条规则不是只看正则、或只看语义，而是正则、意图词、语义锚点三者综合打分。regex 命中是最强的（精确），其次是意图词（fusion），最弱是纯语义匹配（semantic）。

**shadow 规则**。`route_rule.status` 有个 `shadow` 状态——这种规则只记录"如果启用会命中多少次"，但不实际路由。这是干什么的？**A/B 验证新路由规则**。你想加一条新规则，但不确定它准不准，先设成 shadow 跑一段时间，看它的命中情况，确认没问题再切到 active。这是企业级系统才有的"灰度"思维。

**阈值过滤**。即使是最高的语义分，也必须超过 `semanticThreshold` 才算命中。低置信度的匹配宁可放过，不可路由错。

### Stage 3：快速接受 + 语义缓存

到了这一层，已经有了候选技能。但还要不要真的去执行？这一层有个很关键的优化——**语义缓存**。

```typescript
// DecisionEngine.ts（第 503-563 行）
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

`cacheLookup` 做的事是：**这个用户输入，以前是不是决策过？** 用 embedding 相似度 + inputFingerprint 比对，如果发现"半年前有个几乎一样的问题，当时决策成了 X"，直接复用那个决策，跳过 Stage 4 的 LLM 规划。

语义缓存命中，意味着又省掉一次昂贵的 LLM 调用。而且这个复用是语义级的——"帮我写 PRD"和"我要写一份产品需求文档"会被识别为同一个决策。

### Stage 4：LLM 规划（DecisionPlanner）

到这里才动用完整的 LLM 决策。这一层是最贵的，但也是兜底的——真正模糊的、需要推理的、从来没见过的请求，在这里被处理。

注意它叫"Planner"而不是"Decider"——LLM 产出的不是一个"决定"，而是一个**计划 patch**（plan patch）。这个 patch 还不能直接执行，要经过下一层的校验。

### Stage 5：承诺派生（DecisionCommitmentDeriver）

最后一层。LLM 产出的计划是"软"的，这一层把它变成"硬"的可执行承诺：

- 校验 stepBindings（步骤绑定是否合法）
- 检查依赖是否满足
- 生成最终的可执行 plan

这一层的名字"Commitment"很传神——**LLM 可以"建议"，但系统只"承诺"它校验通过的**。LLM 的输出在这里被当作一个需要审核的提案，而不是直接执行的命令。

---

## 三层短路，三种省钱方式

整条管线有三个"提前返回"的点，对应三种不同的省钱策略：

```
用户输入
  │
  ▼
[Stage 1: 精确路由 + 闲聊守卫]
  ├──命中规则/闲聊──────► 短路1: 零 LLM 成本 ✓
  ├──显式多步流程────────► 短路2: 直接编译流程 ✓
  └──都没命中──► [Stage 2: FusionRouter 候选检索]
                    │
                    ▼
                 [Stage 3: QuickAcceptGate]
                    ├──语义缓存命中──► 短路3: 复用历史决策 ✓
                    └──未命中──► [Stage 4: LLM 规划]
                                    │
                                    ▼
                                 [Stage 5: 承诺派生校验]
                                    │
                                    ▼
                                 可执行 plan
```
（三个 ✓ 是省钱短路：规则/闲聊、显式流程、语义缓存）

1. **闲聊/规则短路**：零成本。80%+ 请求死在这里。
2. **显式流程短路**：用户自己说清楚了要干嘛，不用猜。
3. **语义缓存短路**：历史上有过几乎一样的决策，复用它。

只有这三道短路都没拦住的请求，才会走到 LLM。**实测下来，真正需要 LLM 规划的请求是个长尾，可能不到 20%。**

---

## 一个容易混淆的概念：L0/L1/L2/L3 不是决策阶段

讲到这里必须澄清一个我们文档里经常让人懵的点：**决策引擎内部有 5 个阶段（Stage 1-5），但系统里还有一个叫 L0/L1/L2/L3/Chat 的"分层路由"，这俩是完全不同的两个维度，别混。**

L0/L1/L2/L3/Chat 叫 `DecisionRouteLayer`，它是从**另一个角度**给决策打标签的：

| 层 | 含义 |
|----|------|
| **L0** | 系统预决策。worker、定时任务注入的 target，已经决定了让谁做，不需要再决策 |
| **L1** | Turn 层注入的候选能力快照。决策必须复用它，禁止重复 collect |
| **L2** | 异步段 LayeredRouter。读 agent_run.metadata、coding_task.lifecycle_state |
| **L3** | readiness gate（运行前最后一道能力校验） |
| **Chat** | 闲聊，不走任何技能 |

这个标签被写到 `decision.metadata.routeMetadata.routeLayer` 里，供消息持久化回填和后续的门禁判断用。

**5 个 Stage 是"决策怎么一步步算出来"的过程维度；L0-L3 是"这个决策是在哪一层、什么背景下做的"的上下文维度。** 两者正交。这个区分看起来啰嗦，但在调试和审计时极其重要——你得能说清"这个决策是 Stage 几算的、属于哪个 Route Layer"。

---

## 候选、计划、承诺：三个分离

整个决策管线把"决策"拆成了三个清晰的阶段产物：

1. **候选（Candidate）**：Stage 2 产出的"可能适用的技能/员工"列表。带证据（FusionScore、检索证据）。
2. **计划（Plan）**：Stage 4 LLM 产出的"打算怎么执行"的 patch。软的，待校验。
3. **承诺（Commitment）**：Stage 5 校验通过的、真正可执行的 plan。硬的。

这个分离的价值在于**可观测、可干预**：

- 候选错了？调 FusionRouter 的规则和阈值。
- 计划错了？调 LLM 的 prompt 和上下文。
- 承诺错了？调校验规则和 stepBindings。

每一层的问题都能定位到独立的环节去修，而不是"决策不对"这个混沌的整体。**把一个模糊的"决策"拆成候选→计划→承诺三段，是让 Agent 决策可工程化的关键。**

---

## pipeline hooks：让决策全过程可观测

最后说一个工程细节。整条管线的每个阶段，都埋了 hook 钩子：

```typescript
// DecisionEngine.ts
await invokePipelineHooks(this.hooks, 'DecisionEngine', 'onStageStart', 'ExactRouter+PlanExtraction', draft);
// ... 阶段逻辑 ...
await invokePipelineHooks(this.hooks, 'DecisionEngine', 'onStageEnd', '...', draft);
```

每个阶段的开始、结束都有 hook，记录 `stageTrace`、`elapsedMs`。这意味着事后你能看到：

- 这次决策走了几个阶段？
- 在哪个阶段短路返回的？
- 每个阶段花了多少毫秒？
- LLM 规划这一阶段实际跑了没有？

这就是为什么前面我能说"80% 请求死在 Stage 1"——因为有这些 hook 数据撑腰。**没有可观测性的决策引擎，就是黑盒；黑盒里的"省钱"和"准确"都是自我感觉。**

---

## 给后来者的几条总结

1. **决策不要一锤子 LLM**。它会贵、不稳、且没必要。分阶段，从最便宜的确定性手段开始。
2. **确定性优先于 LLM**。能用规则、关键词、闲聊守卫解决的，别上 LLM。LLM 是兜底，不是默认。
3. **多信号融合 + 阈值过滤 + shadow 灰度**。FusionRouter 的三个关键设计：综合打分、宁缺毋滥、新规则先 shadow 验证。
4. **语义缓存是省钱利器**。重复的、相似的请求复用历史决策，跳过 LLM。
5. **候选→计划→承诺三段分离**。让决策可观测、可干预、可独立优化。
6. **LLM 的输出是提案不是命令**。承诺派生层要校验 LLM 的计划，只放行合法的部分。
7. **给决策管线埋 hook**。没有 stageTrace 和 elapsedMs，"省钱"和"准确"都是黑盒里的自我感觉。
8. **分清"决策阶段"和"路由层"两个维度**。它们正交，调试时都要能说清。

渐进式决策的本质，是**对 LLM 的不信任**——能用更便宜、更确定的手段替代，就替代；LLM 只在最必要的时候登场，而且登场了也要校验它的输出。这种"把 LLM 当作最后的、需要监督的手段"的态度，是 Agent 系统能在企业里活下来的前提。

---

> **下一篇**：[《Agent 记忆系统的真实难题：不是 RAG，是遗忘》](./03-memory-system.md)——决策完了，执行的时候，Agent 该"记住"什么、又该"忘记"什么？
