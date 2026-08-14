# 智能路由 route_rule：一条规则如何同时用正则、意图词、语义锚点

> 这是《WinMatrix 开发经验文集》第 42 篇。在 [第 2 篇](./02-decision-engine.md) 和 [第 33 篇](./33-vs-llm-routing.md) 里，我们都提到过决策引擎的 Stage 2 有个 FusionRouter，用一张 `route_rule` 表做多信号融合。但那两篇都没展开讲——一条路由规则里到底有哪些字段，正则、意图词、语义锚点是怎么融合的，shadow 模式到底怎么用。这一篇就把这张表拆开看。

先说一个真实的工程场景。

你在做一个 AI 协作平台，有七八个数字员工。用户在群里发一句"每日早会"，你得知道这是要触发"早会主持"这条流程，而不是把它当成普通聊天。怎么判断？

最直觉的做法是写正则——`/每日早会/`。但用户可能说"开个早会""今天早会走起""来个 morning meeting"，正则覆盖不全。

那上 LLM 路由？让 LLM 判断这句话该走哪。能做，但贵——每条消息都调一次 LLM，而且慢、不可复现。

WinMatrix 的解法介于两者之间：**一条规则里同时放正则、意图词、语义锚点，多信号融合打分，过阈值才算命中**。确定性优先，LLM 兜底。这一篇就拆开这个设计。

---

## route_rule 表：一条规则长什么样

先看这张表的结构。它是路由的配置真源：

```prisma
// prisma/schema.prisma（第 2689-2730 行）
model route_rule {
  id                String   @id @default(cuid())
  name              String   @unique @db.VarChar(64)
  description       String?  @db.VarChar(255)

  patterns          String[]
  patternFlags      String[]
  positiveIntents   String[]
  negativeIntents   String[]
  semanticAnchors   String[]
  semanticThreshold Float     @default(0.85)

  roleId            String    @db.VarChar(64)
  mode              String    @db.VarChar(16)
  skillName         String?   @db.VarChar(64)

  declaredAction    String?   @db.VarChar(64)
  producesDataKind  String?   @db.VarChar(64)
  requiresDataKind  String?   @db.VarChar(64)
  toolHint          String?   @db.VarChar(128)

  status            String    @default("active") @db.VarChar(32)
  source            String    @default("manual") @db.VarChar(32)
  priority          Int       @default(100)

  hitCount          Int       @default(0)
  lastHitAt         DateTime? @db.Timestamptz(6)
}
```

一条规则分四组字段。我们一组组看。

### 第一组：匹配信号（patterns / positiveIntents / negativeIntents / semanticAnchors）

这是这条规则"拿什么去匹配用户输入"的部分。注意有**四种**信号，不是一个：

| 字段 | 类型 | 作用 |
|------|------|------|
| `patterns` | String[] | 正则表达式列表 |
| `patternFlags` | String[] | 正则 flags（如 `i` 忽略大小写） |
| `positiveIntents` | String[] | 正向意图词，命中加分 |
| `negativeIntents` | String[] | 负向意图词，命中扣分 |
| `semanticAnchors` | String[] | 语义锚点文本，用于 embedding 相似度 |
| `semanticThreshold` | Float | 过阈才算命中（默认 0.85） |

为什么需要四种？因为每种信号各有盲区：

- **正则**最精准，但只能覆盖"写法接近"的输入。用户换种说法就漏。
- **意图词**覆盖"提到某个概念"，但单独用太宽——用户说"早会"可能只是"我刚开完早会"，不是要触发早会。
- **语义锚点**能处理同义表达（"morning meeting" 也能命中"早会"），但要算 embedding，有成本。
- **负向意图词**最容易被忽略。它的作用是"踩刹车"——如果用户说"取消早会"，虽然命中了正则和正向意图，但"取消"这个词命中负向，要大幅扣分，避免误触发。

四种信号融合，互相补盲，互相校准。这是"多信号融合"比"单信号匹配"强的地方。

### 第二组：路由目标（roleId / mode / skillName）

匹配上了，往哪路由？

- `roleId`：派给哪个角色。
- `mode`：执行模式，`direct`（直接对话）或 `skill`（调技能）。
- `skillName`：mode=skill 时，调哪个技能。

这一组决定了"命中之后干什么"。`direct` 是轻的——让那个角色正常对话；`skill` 是重的——直接跳到某个技能执行（比如"每日早会"直接跳到 `morning_meeting` 技能）。

### 第三组：语义元数据（declaredAction / producesDataKind / requiresDataKind / toolHint）

这一组是给下游决策用的"声明"。命中这条规则后：

- `declaredAction`：这条规则产出什么语义动作（比如 `daily_standup`）。
- `producesDataKind`：产出什么类型的数据（比如 `meeting_minutes`）。
- `requiresDataKind`：依赖什么上游数据。
- `toolHint`：建议用什么工具。

这部分字段不带匹配逻辑，纯粹是"我命中了，我会产出这些"，让下游编排（比如流程编排里的 provides/consumes 契约）能据此串联。我们在 [第 6 篇](./06-skill-system.md) 讲过技能契约的 provides/consumes，这里是同类思路在路由层的体现。

### 第四组：治理（status / source / priority / hitCount）

- `status`：`active`（正常生效）或 `shadow`（只记录不路由）。这个稍后单独讲。
- `source`：规则怎么来的——`manual`（手写）或 `llm_discovery`（LLM 发现并建议的）。
- `priority`：优先级，同分时谁先。
- `hitCount` / `lastHitAt`：命中计数和最近命中时间，用于评估规则健康度。

`source` 字段容易被忽略但很重要。它让"哪些规则是人写的、哪些是机器建议的"可追溯——机器建议的规则要经过人工 review（`reviewedBy` / `reviewedAt` 字段）才能信任。这是把 LLM 用在"配置生成"而非"运行时决策"上的典型模式。

---

## FusionRouter 怎么融合打分

配置看完了，接下来看运行时怎么算。核心在 `FusionRouter` 类。

它是一个**纯计算组件**，不接 DB、不调 LLM。文件头注释把算法说得很清楚：

```typescript
// agents/core/agent/decision/fusion-router.ts（第 1-20 行）
/**
 * 多信号融合路由器
 *
 * 信号融合算法：
 *   - 正则匹配：+0.9 权重
 *   - 正向意图关键词：+0.2
 *   - 负向意图关键词：-0.8
 *   - 语义相似度（可选，需要注入 embedding 函数）：maxSim * 0.6
 *
 * 路由决策：
 *   - score >= route.semanticThreshold：确定性路由（返回 RouteResult）
 *   - score < route.semanticThreshold：返回 null（交给下游决策管道）
 */
```

### computeScore：四种信号加权

同步路径的打分逻辑非常简洁：

```typescript
// fusion-router.ts（第 333-349 行）
private computeScore(route: RouteEntry, input: string): number {
  let score = 0;

  // 信号 1：正则匹配（权重 0.9）
  const regexHit = route.patterns.some(p => p.test(input));
  if (regexHit) score += 0.9;

  // 信号 2：正向意图关键词（权重 0.2）
  const hasPositive = route.positiveIntents.some(w => input.includes(w));
  if (hasPositive) score += 0.2;

  // 信号 3：负向意图关键词（权重 -0.8）
  const hasNegative = route.negativeIntents.some(w => input.includes(w));
  if (hasNegative) score -= 0.8;

  return score;
}
```

权重的设计很有讲究：

**正则 0.9 是绝对主力**。正则命中几乎就能过阈值（默认 0.85）。这反映了设计取向——正则是人工精心写的，可信度高。

**正向意图 0.2 是辅助加分**。单独一个意图词不够过阈值（0.2 < 0.85），但配合正则或语义锚点能拉高分数。

**负向意图 -0.8 几乎能否定一条规则**。即使正则命中了（+0.9），只要命中负向意图词（-0.8），总分 0.1 远低于 0.85 阈值，这条规则就不会命中。这是"踩刹车"的设计——一句"取消早会"绝不该触发早会流程。

```
信号权重图（默认阈值 0.85）：

正则命中       ████████████████████  +0.9   (主力，单独几乎可过)
正向意图       ████                  +0.2   (辅助加分)
负向意图       █████████████████     -0.8   (强力刹车)
语义锚点       ████████████          ×0.6   (乘以最高相似度)
               ─────────────────────────
阈值           █████████████████    0.85    过线才算命中
```

### 语义锚点：可选的 embedding 加权

`computeScore` 是同步的、零成本的。但还有些场景需要语义相似度——比如用户说"morning meeting"，中文正则和意图词都匹配不上，但语义上和"每日早会"很近。

这种情况下要走 `routeAsync`，它会注入 embedding 函数，算用户输入和 `semanticAnchors` 的余弦相似度：

```typescript
// fusion-router.ts（第 355-369 行）
private computeSemanticScore(route: RouteEntry, inputEmbedding: number[]): number {
  if (!route.precomputedEmbeddings || route.precomputedEmbeddings.length === 0) {
    return 0;
  }
  let maxSim = 0;
  for (const anchorEmbedding of route.precomputedEmbeddings) {
    const sim = cosineSimilarity(inputEmbedding, anchorEmbedding);
    if (sim > maxSim) maxSim = sim;
  }
  return maxSim * 0.6;
}
```

几个细节：

**锚点的 embedding 是预计算好的**（`precomputedEmbeddings`）。不会每次路由都把锚点文本重新 embedding——加载规则时算一次，之后复用。运行时只 embedding 用户输入一次。

**取最高相似度 `maxSim`**，不是平均。一条规则可能配多个语义锚点，只要其中一个和输入足够像就行。

**权重是 `×0.6`**。语义信号不如正则可信（语义模型会漂），所以权重压低。即使相似度 1.0，也只加 0.6，单独不够过阈值（0.6 < 0.85）——必须配合其他信号。

**注入式可选**。`embedFn` 没注入时，`routeAsync` 退化为 `route`，只走同步路径。这让 FusionRouter 在"只有正则规则"的场景下零成本运行，需要语义时才付 embedding 的代价。

### 命中方法标记：regex / fusion / semantic

命中之后，FusionRouter 会标记这条命中是靠什么方法过的：

```typescript
// fusion-router.ts（第 194-201 行，route() 内部）
if (regexHit) bestMethod = 'regex';
else if (route.positiveIntents.some(w => input.includes(w))) bestMethod = 'fusion';
else bestMethod = 'semantic';
```

`method` 字段看似不起眼，但事后排查时极有价值——"这条路由是正则硬命中，还是语义近似命中？"如果是语义命中，可能需要调阈值或锚点；如果是正则命中却路由错了，说明正则写错了。**把"为什么命中"留在决策记录里，是可调试性的基础。**

---

## active 竞速 vs shadow 观察

route_rule 的 `status` 字段有两个值：`active` 和 `shadow`。`route()` 方法对它们的处理完全不同：

```typescript
// fusion-router.ts（第 165-211 行，route() 主体）
route(input: string): RouteResult | null {
  let bestRoute: RouteEntry | null = null;
  let bestScore = 0;

  for (const route of this.routes) {
    const score = this.computeScore(route, input);

    if (score >= route.semanticThreshold) {
      // 影子规则：只记录指标和日志，不实际路由
      if (route.status === 'shadow') {
        logger.info({ routeId: route.id, score, input: input.slice(0, 100) },
          '[FusionRouter] shadow hit');
        this.bumpMetrics(route.id);
        this.shadowHits.push({
          routeId: route.id, score,
          input: input.slice(0, 200), timestamp: new Date(),
        });
        continue;   // ← 不参与竞速，继续看下一条
      }

      // active 规则：参与正常竞速
      if (score > bestScore) {
        bestScore = score;
        bestRoute = route;
        // ...标记 method
      }
    }
  }

  if (bestRoute) {
    this.bumpMetrics(bestRoute.id);
    return { route: bestRoute, confidence: bestScore, method: bestMethod, source: bestRoute.source };
  }
  return null;
}
```

行为差异：

```
一条输入进来，遍历所有 route_rule：

  对每条规则算 score
       │
       ├─ score < 阈值 → 不命中，跳过
       │
       └─ score >= 阈值
            │
            ├─ status === 'shadow'
            │     → 记日志 + bumpMetrics + push 到 shadowHits
            │     → continue，不参与竞速
            │
            └─ status === 'active'
                  → 与当前 bestScore 比，更高则更新 bestRoute

  最后：
    bestRoute 存在 → 返回 RouteResult（命中）
    bestRoute 为空 → 返回 null（交给下游 LLM 决策）
```

### shadow 模式解决什么问题

shadow 是**路由规则的灰度发布**。想象这个场景：你写了一条新规则"识别用户想生成周报"，但不确定它准不准——可能误命中别的场景。

如果直接设成 active，一旦误判，用户消息就被错误路由了，影响线上。

设成 shadow，它只记录"如果启用会命中哪些输入"，不影响实际路由。跑几天看 shadowHits 日志：

- 如果命中的都是预期的"周报"输入 → 信心足够，转 active。
- 如果命中了一堆乱七八糟的输入 → 调锚点、调阈值，再观察。

这是企业级系统才有的成熟度——**新规则先灰度观察，验证后再生效**。类比一下，这是配置版的"蓝绿部署"。

### shadowHits 的生命周期

shadow 命中记录不是无限累积的。FusionRouter 提供了 `drainShadowHits`：

```typescript
// fusion-router.ts（第 255-259 行）
drainShadowHits(): ShadowHit[] {
  const hits = this.shadowHits;
  this.shadowHits = [];
  return hits;
}
```

`drain` 是"取走并清空"。DecisionEngine 在每次决策完成后调它，把本轮 shadow 命中回写到决策记录的 `actualDecisionPath` 里。这样事后能查到"这条规则在这条消息上 shadow 命中了，分数多少"。

**把 shadow 命中绑定到单次决策、而不是全局累积**，既方便观测，又避免内存无限增长。

---

## 多规则竞速：最高分获胜

实际系统里不可能只有一条规则。一个输入进来，可能有多条规则都过了阈值。谁赢？

答案很简单：**最高分获胜**。

```typescript
if (score > bestScore) {
  bestScore = score;
  bestRoute = route;
}
```

这是"竞速"模式——所有 active 规则平等参与，谁分高谁赢。`priority` 字段在同分时起作用（用于排序遍历），但正常情况下分数差异足够区分。

竞速模式的好处是**可解释**——某条规则赢了，是因为它分数最高，能查到。坏处是**分数可比性**要求高——不同规则的正则、意图词权重是一样的（都是固定权重），否则分数不可比。这也是为什么权重写死在代码里（0.9 / 0.2 / -0.8 / ×0.6），而不是每条规则自己配——**只有权重统一，跨规则的分数才有意义**。

---

## FusionRouter 在决策引擎里的位置

最后看一下 FusionRouter 在整条决策管线里的位置。它不是独立使用的，而是被 `FusionRouterStage` 包裹，作为 Stage 2 接入五阶段管线：

```
用户输入
   │
   ▼
Stage 1: ExactRouter + PlanExtraction
   │  （精确匹配 + 简单闲聊守卫）
   ├─ 命中 → 直接 return
   │
   ▼
Stage 2: FusionRouter（本篇主角）
   │  route_rule 多信号融合打分
   ├─ 命中 active 规则 → 产候选，继续
   ├─ shadow 规则 → 只记录
   │
   ▼
Stage 3: QuickAcceptGate
   │  （语义 cache 命中 → 跳过 LLM）
   ├─ 命中 → 直接 return
   │
   ▼
Stage 4: DecisionPlanner（LLM 规划）
   │  （贵，但能处理全新场景）
   ▼
Stage 5: DecisionCommitmentDeriver
```

FusionRouter 处于"确定性路由"和"LLM 兜底"之间。它比 Stage 1 的精确匹配强（多信号融合），比 Stage 4 的 LLM 便宜（零 LLM 调用）。命中就能省一次 LLM 调用；没命中，交给下游。

**这就是"确定性优先、LLM 兜底"在路由层的落地。** 能用配置规则解决的，绝不调 LLM；配置解决不了的，再让 LLM 兜底。我们在 [第 2 篇](./02-decision-engine.md) 和 [第 33 篇](./33-vs-llm-routing.md) 从不同角度讲过这个哲学，这一篇从 FusionRouter 的字段和算法层面把它落到了代码。

---

## 给后来者的总结

1. **一条 route_rule 同时配四种信号**：patterns（正则，+0.9）/ positiveIntents（+0.2）/ negativeIntents（-0.8）/ semanticAnchors（×0.6 相似度）。互相补盲，互相校准。

2. **权重写死在代码里、全局统一**。只有这样跨规则的分数才可比。如果让每条规则自己配权重，竞速就失去意义。

3. **负向意图词是必须的"刹车"**。-0.8 几乎能否定一条规则。"取消早会"不该触发早会流程——没有负向意图，这类误触发无解。

4. **语义锚点可选、预计算 embedding、取最高相似度**。embedFn 不注入时退化为同步零成本；注入时只 embedding 用户输入一次，锚点复用预计算结果。

5. **命中方法标记（regex/fusion/semantic）要留痕**。事后排查"为什么路由到这"全靠它。可调试性的基础。

6. **shadow 模式是路由规则的灰度发布**。新规则先 shadow 观察命中率，验证后转 active。shadow 命中绑定到单次决策（drainShadowHits），不无限累积。

7. **active 规则竞速，最高分获胜**。简单可解释。priority 字段只在同分时起排序作用。

8. **第三组语义元数据（declaredAction/producesDataKind 等）让路由结果能驱动下游编排**。路由不只是"派给谁"，还要声明"会产出什么"，让契约串联成为可能。

9. **source 字段区分人写还是机器建议**。LLM 可以用来生成规则候选，但必须经人工 review 才信任——把 LLM 用在"配置生成"而非"运行时决策"上。

FusionRouter 的精髓不在算法多复杂——它就是加权求和——而在于把"什么时候用确定性、什么时候让 LLM 兜底"这条线划得清楚。一条规则四种信号、过阈值才算数、shadow 灰度、竞速选最优，这一套组合拳让"给系统加新路由场景"变成可控的工程，而不是玄学。

---

> **下一篇**：[《语义分诊门 SemanticTriageGate：闲聊、终止、bound、unbound 怎么分》](./43-semantic-triage-gate.md)——FusionRouter 解决的是"路由到哪个员工"，但在路由之前还有个更前置的问题：这条消息要不要进决策管线？是闲聊直接短路，还是想终止进行中的任务，还是想续接之前的执行？SemanticTriageGate 就是这道分诊门。
