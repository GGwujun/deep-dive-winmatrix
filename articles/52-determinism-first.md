# 确定性优先：能用规则就别用 LLM

> 这是《WinMatrix 开发经验文集》第 52 篇，"工程哲学"系列的第二篇。第 10 篇讲过"渐进式决策"，第 33 篇对比过"纯 LLM 路由"，第 28 篇讲过"声明式授权防 prompt 注入"。这一篇要把这些散落在决策引擎、FusionRouter、闲聊守卫、缓存、收敛机制里的设计，归纳成同一条原则：**确定性优先，LLM 是兜底而不是默认。** 这条原则不是"LLM 不好用"，而是"什么时候该用 LLM"有一个清晰的判断标准。

先讲一个让很多人意外的数字。

WinMatrix 的决策引擎（第 02 篇、第 33 篇详细讲过）是一个五阶段管线：ExactRouter+PlanExtraction → FusionRouter → QuickAcceptGate → DecisionPlanner(LLM) → DecisionCommitmentDeriver。其中只有第四阶段（DecisionPlanner）是 LLM 调用，前三个阶段都是确定性的。

生产数据显示，**80% 以上的请求在前三个确定性阶段就被解决了**，根本到不了 LLM 规划。换句话说，每五条用户消息里，只有一条需要 LLM 来"决策"。

这个数字背后是一条贯穿 WinMatrix 全部设计的原则：**能用规则解决的，绝不用 LLM。** 这条原则不是出于省钱（虽然确实省钱），而是出于三个更深的工程考量——可测试、可复现、零成本。这一篇就把这三个考量和它们在各模块的落地讲清楚。

---

## 确定性的三大优势

为什么不优先用 LLM？不是因为 LLM 不强，而是因为确定性手段有三个 LLM 永远比不了的优势。

### 优势一：可测试

确定性代码是可测试的——同样的输入永远产生同样的输出，写单元测试直接断言就行。

```
确定性路由测试:
  输入 "帮我生成日会纪要"
  断言: route = { roleId: 'process_manager', skill: 'daily_standup' }
  → 永远通过
```

LLM 路由的测试呢？同样的输入，LLM 这次可能路由到 process_manager，下次可能路由到 orchestrator（因为它"觉得"项目总指挥该管日会）。你没法断言"必须路由到 X"，只能断言"路由到 X 或 Y 都算对"——这种测试的回归保护力大打折扣。

更糟的是，LLM 的输出会随模型版本变化——今天用的模型路由对了，明天供应商更新模型，同一输入路由错了，你的测试一个没挂，但生产坏了。**确定性代码的测试是护栏，LLM 的测试是橡皮图章。**

### 优势二：可复现

确定性代码的执行结果可以 100% 复现——同一个输入，今天跑、明天跑、在测试环境跑、在生产环境跑，结果一模一样。这意味着 bug 可以"按原样再跑一遍"来调试。

LLM 的输出有随机性（即使 temperature=0，不同批次、不同负载下也可能有微小差异）。一个"LLM 路由错了"的 bug，你拿同样的输入再跑一遍，它可能就路由对了——bug 无法复现，就无法定位根因。

可复现性是调试的前提（第 53 篇会专门讲）。没有可复现性，调试 LLM 问题靠的是"猜+祈祷"——改改 prompt、调调参数，"好像好了"就发版，下次什么时候复现不知道。

### 优势三：零成本

确定性代码的执行成本是 CPU 和内存，对现代服务器来说几乎免费。LLM 的执行成本是 token 费 + 网络延迟——每次调用都花钱、都耗时。

```
一条用户消息的决策成本:
  确定性路由（前 3 阶段）:  ~0 元，<10ms
  LLM 规划（第 4 阶段）:     ~0.01-0.1 元，1-5s
```

80% 的请求走确定性，意味着 80% 的决策成本是 0。如果每条消息都走 LLM，成本会高 5 倍以上，延迟也会高一个数量级。这不是小事——对一个企业级平台，成本和延迟直接决定能不能活下去。

**可测试、可复现、零成本，这三条是确定性优先原则的全部理性基础。** 它们不是"教条"，是可以量化的工程优势。

---

## 什么时候该用 LLM：三条判断标准

确定性优先不等于"不用 LLM"。WinMatrix 的第五阶段 DecisionPlanner 就是 LLM，闲聊回复也用 LLM，工具调用结果的总结也用 LLM。关键问题是：**什么时候该用 LLM？**

这里有三个判断标准。

### 标准一：输入是不是结构化的？

如果输入是结构化的（明确的命令、匹配的关键词、确定的意图），用规则。如果输入是开放的自然语言（模糊的描述、需要理解的隐喻），用 LLM。

```
结构化输入（规则）:
  "生成日会纪要"           → 关键词命中 → 确定性路由
  "@小品 写 PRD"            → @提及 + 明确动作 → 确定性路由

开放输入（LLM）:
  "我们这个项目感觉节奏有点乱，能不能帮忙理一理"
  → 意图模糊，需要 LLM 理解 + 规划
```

### 标准二：输出是不是可枚举的？

如果输出是有限的、可枚举的集合（比如路由到 N 个角色之一、匹配 M 条规则之一），用规则 + 融合打分。如果输出是开放的生成（比如写一段话、总结一份文档），用 LLM。

```
可枚举输出（规则）:
  路由决策 → 8 个角色之一 → FusionRouter 打分选最佳
  意图分类 → casual_chat / termination / bound / unbound → SemanticTriageGate

开放输出（LLM）:
  生成 PRD → 无限可能的文本 → LLM
  总结工具结果 → 无限可能的摘要 → LLM
```

### 标准三：错误代价有多高？

如果错误代价高（涉及安全、资金、不可恢复操作），用确定性规则（可审计、可追责）。如果错误代价低（可修正、可重试），可以容忍 LLM 的不确定性。

```
高代价（规则）:
  工具授权 → declaredOperations 硬约束（第 28 篇）
  凭证解析 → L3 fail-closed（不推断）
  终态收敛 → sweeper 确定性逻辑（第 51 篇）

低代价（LLM）:
  闲聊回复 → 错了就错了，用户再说一遍
  内容生成 → 草稿用户会改
```

**这三条标准合起来，就是"什么时候该用 LLM"的判断框架。** 满足任何一条"该用规则"的，优先规则；三条都指向"该用 LLM"的，才用 LLM。

---

## 确定性优先在各模块的落地

讲完原则，看 WinMatrix 各模块怎么落地。这些模块表面不相干，但都遵循同一个原则。

### 落地一：决策引擎的渐进式管线

最典型的落地。决策引擎五阶段（`DecisionEngine.ts:372-440`）：

```ts
private async decideInner(input: DecisionInput): Promise<DecisionResult> {
  // Stage 1: ExactRouter + PlanExtraction（确定性：关键词/正则）
  const exactMatch = await this.exactRouter.routeAsync(input, resolvedTurn);
  const { candidate: chatCandidate } = await this.simpleChatGuard.extract(input, exactMatch, resolvedTurn);
  // 显式流程编排短路（确定性：用户显式描述多步流程）
  const explicitFlowOutcome = this.tryExplicitFlowOrchestrationPlan(input, ...);
  if (explicitFlowOutcome?.terminal) return this.finalize(explicitFlowOutcome);
  // Stage 2: FusionRouter（确定性：多信号融合打分）
  // Stage 3: QuickAcceptGate（确定性：语义 cache 命中）
  // ...命中则 finalizeAcceptedPlan 直接 return
  // Stage 4: DecisionPlanner（LLM）——只有前面都没解决才到这
  // Stage 5: DecisionCommitmentDeriver（确定性：承诺推导）
}
```

每一阶段都比下一阶段便宜，且都是 terminal 的（命中就返回，不往下走）。LLM 在第四阶段，是兜底。**80% 的请求在前三阶段解决，只有 20% 才到 LLM。**

这里还有个三层短路加速（核实报告 ch09-12 主题 2 设计要点 2）：

- **SimpleChatGuard**：闲聊直接走 chat 回复，不进后续决策
- **ExplicitFlowOrchestration**：用户显式描述多步流程，直接编译成流程编排
- **QuickAcceptGate cacheLookup**：语义 cache（embedding 相似度 + inputFingerprint）命中，跳过 LLM

三层短路都是确定性的，各自拦掉一类不需要 LLM 的请求。合起来把 LLM 的调用面压到最小。

### 落地二：FusionRouter 的多信号融合

FusionRouter（`fusion-router.ts:165-201`）是确定性路由的精髓。一条用户消息，怎么确定性地判断它该路由到哪个角色？

```ts
route(input: string): RouteResult | null {
  let bestRoute: RouteEntry | null = null;
  let bestScore = 0;
  for (const route of this.routes) {
    const score = this.computeScore(route, input);
    if (score >= route.semanticThreshold) {
      if (route.status === 'shadow') {        // shadow 规则只记录不路由
        this.bumpMetrics(route.id);
        this.shadowHits.push({...});
        continue;
      }
      if (score > bestScore) {
        bestScore = score; bestRoute = route;
        const regexHit = route.patterns.some(p => p.test(input));
        if (regexHit) bestMethod = 'regex';
        else if (route.positiveIntents.some(w => input.includes(w))) bestMethod = 'fusion';
        else bestMethod = 'semantic';
      }
    }
  }
  // ...
}
```

一条 route_rule（`schema.prisma:2689-2730`）同时用四种信号：

```
route_rule 的四信号融合
├── patterns:String[]         正则模式（最强：精确匹配）
├── positiveIntents:String[]  正向意图词（强：关键词命中）
├── negativeIntents:String[]  负向意图词（排除：命中则降分）
└── semanticAnchors:String[]  语义锚点（弱：embedding 相似度）
    + semanticThreshold:Float  阈值（默认 0.85）
```

四种信号融合打分，超过阈值且最高分的规则获胜。命中方式还有分级——regex 最强、fusion 次之、semantic 最弱。

**这是"确定性优先"的极致体现：连"语义匹配"这种看起来必须用 LLM 的活，都尽量用 embedding 阈值 + 规则融合来做，不调 LLM。** embedding 是向量化（一次性、可缓存），不是生成式 LLM 调用——成本和延迟都低一个数量级。

shadow 状态的设计也是确定性的体现（第 23 篇讲过）：新规则先 shadow（只记录命中、不实际路由），观察一段时间确认效果再切 active。这是"规则生效"这件事本身的确定性灰度——不靠 LLM 判断"这条规则该不该上线"，靠实际命中数据。

### 落地三：闲聊守卫的短路

SemanticTriageGate（`SemanticTriageGate.ts`）在 Turn 的准入门就拦截掉一类请求：

```ts
// 分类（核实报告 ch09-12 主题 2）:
SemanticTriageClassification = 'casual_chat' | 'termination_intent' | 'bound' | 'unbound'
LOW_CONFIDENCE_THRESHOLD = 0.65
```

casual_chat（闲聊）直接短路走 chat 回复，不进决策引擎、不进工具调用、不进 LLM 规划。"你好""谢谢""在吗"这类消息，用规则识别出来，直接回复，零 LLM 成本。

注意 `LOW_CONFIDENCE_THRESHOLD = 0.65`——这是个"宁可放过"的阈值。如果分类置信度低于 0.65，不当闲聊处理，让它进后续管线（可能用 LLM）。**确定性判断不确定时，让步给 LLM 兜底，而不是强行用规则决定。** 这才是"确定性优先，LLM 兜底"的完整姿态——规则优先，但规则不自信时让位。

### 落地四：缓存的确定性命中

QuickAcceptGate 的 cacheLookup 是"用确定性手段跳过 LLM"的另一种形态。语义 cache 的逻辑：

```
新请求进来
    ↓ 算 inputFingerprint + embedding
    ↓ 查 cache: 有没有相似度够高的历史决策？
    ↓ 命中 → 直接复用历史决策（跳过 LLM）
    ↓ 未命中 → 走 LLM 规划，结果写进 cache
```

cache 命中是确定性的（embedding 相似度 + fingerprint 比对），但它跳过的是 LLM 调用。**这是"用确定性查询替代不确定的 LLM 调用"——相似请求复用历史决策，既省成本又保证一致性（相似输入给相似输出）。**

### 落地五：收敛机制的确定性（呼应第 51 篇）

第 51 篇讲的终态收敛机制——llmCallWatchdogSweeper、reconcileStaleRunsOnBootstrap、outbox sweeper——全部是确定性规则驱动的。阈值判断、状态扫描、终态补写，每一步都是 if-else，不调 LLM。

这是必须的。收敛机制是系统的免疫系统，**它自己不能不确定**——如果 sweeper 用 LLM 判断"这个 span 该不该收敛"，LLM 说"再等等"，那就永远收敛不了。收敛必须靠硬规则（超时阈值），不能靠软判断。

**这条呼应了第 51 篇的命题：收敛机制本身必须确定性优先。** 这是两条哲学的交汇点——终态收敛要求确定性，确定性优先保障收敛可靠。

---

## 一个反例：什么时候确定性退场，LLM 登场

讲这么多确定性的好，公平起见也说说它什么时候要让位。

确定性规则的致命弱点是**覆盖不了所有情况**。规则是枚举的，但用户输入是无限的。不管你写多少条 route_rule，总有匹配不上的输入——长尾。

```
请求分布:
  ┌──────────────────────────────────┐
  │ 80% 常见请求（规则覆盖）           │  ← 确定性搞定
  ├──────────────────────────────────┤
  │ 15% 少见请求（规则勉强覆盖）       │  ← 规则 + LLM 兜底
  ├──────────────────────────────────┤
  │ 5%  长尾请求（规则覆盖不了）       │  ← 纯 LLM
  └──────────────────────────────────┘
```

那 5% 的长尾，就是 LLM 的主场。它们是规则设计时没想到的、用户表达千奇百怪的请求。对这些请求，确定性规则会返回"无法路由"，这时 LLM 登场，用它的一般化理解能力去规划。

WinMatrix 的 DecisionPlanner（第四阶段）就是这个角色——它是兜底，专门处理前三阶段搞不定的 20%。它的存在不是否定确定性优先，恰恰是**确定性优先的补充**：规则负责大头，LLM 负责长尾；规则保证 80% 的便宜和稳定，LLM 保证 20% 的覆盖。

这种分工的心智模型是：

```
不要:  LLM 包打天下（贵、不稳、不可测）
不要:  规则包打天下（覆盖不了长尾）
要:    规则优先 + LLM 兜底（便宜稳定 + 长尾覆盖）
```

**确定性优先不是"消灭 LLM"，而是"把 LLM 用在该用的地方"。** 让 LLM 干它擅长的（开放域理解、生成、长尾覆盖），别让它干它不擅长的（结构化判断、可枚举决策、高代价操作）。

---

## 这条原则的普遍性

确定性优先不是 WinMatrix 独有的发明，它是工程世界的普适原则。看几个跨领域的呼应：

**编译器 vs 解释器。** 编译器尽量在编译期（确定性）做优化和类型检查，运行期（动态）才处理编译期搞不定的事。能在编译期确定的，绝不留到运行期。

**数据库的查询优化器。** 优化器用基于规则的启发式（确定性）先做大部分优化，只有复杂查询才用基于代价的动态规划。规则优先，代价兜底。

**缓存。** 缓存是"用确定性查询替代不确定的计算"——能从缓存拿的，就不重新算。和 QuickAcceptGate 的语义 cache 一个思路。

**类型系统。** 静态类型（确定性）能在编译期挡住一大类错误，动态类型（运行时）兜底剩下的。能在类型层防住的，绝不留到运行时。

这些呼应说明：**确定性优先是"把已知留给规则、把未知留给灵活"的普适工程智慧。** WinMatrix 把它落在 AI 平台上，表现为"决策引擎渐进式管线、FusionRouter 多信号融合、闲聊守卫短路、语义 cache、收敛机制硬规则"——形态各异，内核相同。

---

## 给后来者的几条总结

1. **确定性优先不是省钱，是可测试 + 可复现 + 零成本。** 这三条是量化的工程优势，不是教条。
2. **能用规则解决的，绝不用 LLM。** WinMatrix 80% 的请求走确定性路由，只有 20% 才到 LLM。
3. **判断"该不该用 LLM"的三标准：输入结构化吗、输出可枚举吗、错误代价高吗。** 三条都指向规则就别用 LLM，三条都指向 LLM 才用 LLM。
4. **决策引擎的渐进式管线是确定性优先的典范。** 五阶段里只有一个是 LLM，前三个确定性阶段解决 80%。每阶段 terminal，越早越便宜。
5. **连"语义匹配"都尽量不用 LLM。** FusionRouter 用 embedding 阈值 + 四信号融合，比 LLM 调用便宜一个数量级。
6. **确定性不自信时让位给 LLM。** LOW_CONFIDENCE_THRESHOLD=0.65——规则置信度不够就让 LLM 兜底，不强行决定。"确定性优先，LLM 兜底"是完整姿态。
7. **收敛机制必须确定性。** 系统的免疫系统不能不确定。sweeper 用硬规则，不用 LLM 判断。
8. **LLM 的角色是长尾覆盖，不是默认路径。** 规则负责大头（便宜稳定），LLM 负责长尾（开放覆盖）。两者分工，不是替代。
9. **这条原则是普适的工程智慧。** 编译器、数据库优化器、缓存、类型系统都遵循"已知留给规则、未知留给灵活"。AI 平台只是它最新的落地场景。

最后说一句：做 AI 平台的人，容易被 LLM 的强大迷惑，什么都想用 LLM 解决。这种诱惑要克制。LLM 是工具箱里最灵活但也最不可控的那把工具——用它有代价（成本、延迟、不可测）。**真正成熟的 AI 工程师，不是"能用 LLM 就用 LLM"，而是"能不用 LLM 就不用 LLM"。** 把 LLM 留给真正需要它的地方，是确定性优先原则的最终奥义。

---

> **下一篇**：[《可重放性：一个 bug 能不能"按原样再跑一遍"》](./53-replayability.md)——确定性优先的一个直接收益就是"可复现"。下一篇把这条线展开：span 树 + transcript + 事件 seq 怎么让任意一次执行可回放，以及为什么可重放是调试的前提。
