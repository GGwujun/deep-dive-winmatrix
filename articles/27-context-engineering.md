# Context Engineering 实战：每轮该给 LLM 注入什么、注入多少

> 这是《WinMatrix 开发经验文集》第 27 篇。"Context Engineering"是最近被反复讨论的概念——比起 prompt engineering（怎么写一句好提示），context engineering 更关心"每一轮喂给 LLM 的上下文里到底该有什么、有多少"。这件事在 Agent 系统里极其关键：注入太少 LLM 失忆，注入太多爆 token 又贵又慢，注入错了 relevant 的东西 LLM 被带偏。这篇讲 WinMatrix 怎么用明确的数字回答"注入什么、注入多少"。

先抛一个让做过 Agent 的人都纠结过的问题。

用户发了句"接着改刚才的 PRD"。这句"刚才的 PRD"指的是什么？系统要给 LLM 喂什么上下文，它才能正确理解？

候选答案很多：

- 最近的对话历史？多少条？
- 项目的长期记忆？挑哪些？
- 当前工作区里相关的文档？怎么选？
- 这个用户的偏好？"改"这个动作的约定？

每一个都是"context engineering"要回答的问题。**demo 阶段可以把所有历史一股脑塞给 LLM，反正上下文够长；生产阶段这么做必崩——token 账单爆炸、LLM 注意力被无关信息稀释、关键信息被淹没。**

WinMatrix 的做法是用**明确的数字上限 + 分区检索 + 可观测 snapshot**，把"注入什么、注入多少"变成可工程化、可调参的问题。

---

## 业界视角：四种上下文策略

先借业界的框架帮我们想清楚这个问题。业界讨论 context engineering 时，常提到四种策略（这四种是我们借来的"思考角度"，不引文字）：

- **Write**：什么内容该写进上下文（记忆、历史、工作产物）。
- **Select**：每轮从可写的内容里选哪些注入（检索、过滤、排序）。
- **Compress**：上下文太长时怎么压缩（摘要、裁剪、遗忘）。
- **Isolate**：不同上下文之间怎么隔离（会话间、员工间、项目间）。

这四个角度提供了一个清单：设计 context 时挨个问自己"我写了吗？选了吗？压了吗？隔了吗？"

WinMatrix 没有显式说"我在做这四件事"，但它的实现刚好覆盖了全部四个角度。这篇用这个框架串 WinMatrix 的源码。

---

## Write：什么内容会进入上下文池

先看"写"——哪些东西会成为潜在的可注入内容。WinMatrix 的记忆是三层结构（核实报告 ch09-12，第 12 章）：

```
【会话层】ConversationService + Redis conversation:{id}（最多 50 条）+ PG conversation_histories
   ↓ 转录同步（防抖 10s）
【转录层】PG session_transcript（LLM 上下文真源，含 tool_call/thinking trace）
   ↓ TranscriptSyncManager.markDirty → 防抖 10s → indexContent
【长期索引层】PG memory_chunks/memory_files + ES dense_vector
```

这三层就是"写"的三个池子：

- **会话层**：实时对话消息，写 Redis（快）+ PG（持久）。最多缓存 50 条。
- **转录层**：完整记录每次对话的 tool_call、thinking trace，是"LLM 上下文真源"。注意它比会话层更详细——会话层只存消息，转录层连 LLM 的思考过程都存。
- **长期索引层**：会话结束后，转录被分块、向量化，双写进 PG（memory_chunks，带 tsvector）和 ES（dense_vector）。这就是"会话→长期记忆"的自动转化。

**"写"的关键是分块和向量化**（`MemoryIndexManager.ts:370-454`）：

```ts
private async indexContentCore(pathKey, content, ...): Promise<number> {
  const contentHash = hashContent(content);
  const existing = await getMemoryFile(pathKey);
  if (existing && existing.hash === contentHash) {
    return 0;   // hash 比较跳过未变更内容（0 开销）
  }
  // 删旧 → 重新分块 → 双写 ES + PG
  const chunks = chunkText(content, pathKey, source, projectId, agentId);
  // ... ES + PG 写入 ...
}
```

注意 hash 去重——内容没变就跳过，0 开销。这让"每 10 秒同步一次"这种高频写入变得可行。

记忆还被打了类型标签：`[decision/finding/preference/constraint/info]`（`MemoryContextBootstrap.ts:88`）。这些标签后续会被注入文本，让 LLM 知道"这是用户的偏好""这是项目约束"，而不是一堆无差别的文本。

---

## Select：每轮注入多少——明确的 top-k 上限

"选"是 context engineering 的核心。WinMatrix 的回答是**每个来源都有明确的 top-k 上限**。

每轮注入的总量是这几个数字加起来（核实报告 ch09-12，第 12 章设计要点 7）：

| 来源 | 上限 | 备注 |
|------|------|------|
| 长期记忆（三区合计） | top-k=5 | `DEFAULT_MEMORY_LIMIT = 5` |
| 对话历史 | 10 条 | `getRecentMessages(conversationId, 10)` |
| 工作上下文 | 10 条 | `getWorkingContext(10)` |
| 项目上下文 | 1 条 body | ProjectContextSource 注入 |
| 当前用户消息 | 1 条 | 本轮输入 |

这几个数字不是拍脑袋定的，每个都有理由。

### 长期记忆的三区检索

长期记忆不是简单 top-k，而是分三区检索（`MemoryContextBootstrap.ts:29-34, 167-224`）：

```ts
// 三区检索常量
const ZONE1_LIMIT = 3; const ZONE1_MIN_SCORE = 0.25;   // 当前会话 session
const ZONE2_LIMIT = 3; const ZONE2_MIN_SCORE = 0.5;    // 项目 memory
const ZONE3_LIMIT = 1; const ZONE3_MIN_SCORE = 0.8;    // 跨会话 session（降级）

// Zone 1：当前会话
const zone1Results = await memoryService.search({
  query, projectId, conversationId,
  sources: ['session'], limit: ZONE1_LIMIT, minScore: ZONE1_MIN_SCORE, hybrid: true,
});
// Zone 2：项目 memory
const zone2Results = await memoryService.search({
  query, projectId, agentId,
  sources: ['memory'], limit: ZONE2_LIMIT, minScore: ZONE2_MIN_SCORE, hybrid: true,
});
// Zone 3：跨会话（仅当 Zone1+2 不足 2 条时触发）
if (results.length < 2 && conversationId && projectId) {
  const zone3Results = await memoryService.search({
    query, projectId, excludeConversationId: conversationId,
    sources: ['session'], limit: ZONE3_LIMIT, minScore: ZONE3_MIN_SCORE, hybrid: true,
  });
}
```

三区的逻辑是**"近的优先、远的严格"**：

- **Zone 1（当前会话，3 条，分数 0.25）**：当前会话里讨论过的东西最相关，门槛最低，给 3 个名额。
- **Zone 2（项目记忆，3 条，分数 0.5）**：项目沉淀的长期记忆，门槛中等，也给 3 个名额。
- **Zone 3（跨会话，1 条，分数 0.8）**：别人在其他会话讨论的相关内容，门槛最高（怕串台），只给 1 个名额，**而且只有 Zone1+2 不足 2 条时才触发**——平时不查跨会话记忆，避免引入噪音。

**为什么门槛越来越高？** 因为相关性越来越弱。当前会话的东西大概率相关，给低门槛；跨会话的东西可能完全不相关，必须高门槛严控。这是 select 策略里"按相关性梯度调阈值"的典型实现。

**为什么总数控制在 5 条？** 这是 token 预算的考量。每条记忆被格式化成 Markdown（带类型标签、相关度分数、来源），平均一条几百 token，5 条就是 1-2k token——加上历史和上下文，总注入量控制在可接受范围。无限加记忆会让 LLM 注意力稀释，反而表现更差。

### 对话历史和工作上下文

历史 10 条 + 工作上下文 10 条，是另外两个明确的数字。注意 TurnRunner 启动时的三路并行（核实报告 ch09-12，TurnRunner.ts:102-108）：

```ts
const [turnPipelineContext, context, historyResult] = await Promise.all([
  TurnContextLoader.loadPipelineContext(conversationId),
  resolveAgentContext(conversationId),
  useCausalHistory ? Promise.resolve({...}) : getHistoryAdapter().getRecentMessages(conversationId, 30),
]);
```

这里 history 取了 30 条（给决策引擎用），但**最终注入 LLM 的历史是 10 条**——多取的 20 条是给决策引擎做上下文理解用的，不是全塞给 LLM。这是个常见模式：**内部处理可以多用一些上下文，真正注入 LLM 的要严格裁剪**。

### 选完之后还要格式化

选出的记忆不是裸塞，而是格式化（`formatMemoryResults`）：

```ts
// 格式化为 Markdown 注入文本
// 带 [decision/finding/preference/constraint/info] 类型标签
// 带相关度分数 + 来源
```

注入文本大概长这样：

```
## 长期记忆（相关度 > 阈值）
- [preference] 用户偏好简洁的 PRD（来源：项目记忆，相关度 0.72）
- [constraint] 本项目 PRD 必须包含验收标准（来源：项目记忆，相关度 0.68）
- [decision] 上次决定用 React 而非 Vue（来源：当前会话，相关度 0.45）
...
```

**类型标签让 LLM 知道每条记忆的性质**——preference 是偏好（可协商），constraint 是约束（必须遵守），decision 是已决定的事（不要重新讨论）。这比一堆无差别文本有用得多。

---

## Compress：上下文太长怎么办

"压缩"在 WinMatrix 里有两层。

**第一层：检索即压缩。** 三区检索本身就是一种压缩——从成千上万的 memory_chunks 里只选 top-k=5。这种压缩发生在 select 阶段，用户无感。

**第二层：降采样的源点暂存。** 这是个更精妙的模式，出现在可观测性里（参考第 8 篇）。LLM 调用的 request 会被 observability hub "compact"（剥掉 request.messages 来控制日志体积），但如果 compact 后再想看完整 request 就没了。WinMatrix 的做法是：

```ts
// llmObservability.ts:89-128（核实报告 ch23-29）
// 在 hub compactLlmEventDataForLogSink 剥离 request 之前，于源点暂存完整 request
if (obsFields.llmInvocationId) {
  import('./elasticsearchLlmLogger.js')
    .then(({ storeFullRequestForRun }) => storeFullRequestForRun(invocationId, request))
    .catch(() => {});
}
```

**在数据被降采样之前，先把原始版本存下来。** 这是个普适的压缩原则：任何压缩都是不可逆的信息损失，压缩前必须有"原始版本可回溯"的机制，否则出问题时无从 debug。

**会话→长期记忆的转化也是一种压缩。** 一次会话可能几百轮对话，全部塞给 LLM 不现实。会话结束后，TranscriptSyncManager 把它转成长期记忆的分块索引（防抖 10s 增量同步 + 30 分钟全量兜底 + 启动全量）。下次相关会话时，通过检索只取 top-k=5 条最相关的回来。**这是"把长对话压缩成可检索的短记忆"的典型路径。**

---

## Isolate：不同上下文之间怎么隔离

"隔离"是常被忽略的一环。WinMatrix 有多重隔离：

**会话隔离**：Zone 3 检索跨会话记忆时，门槛极高（0.8）而且默认不触发（只有 Zone1+2 不足 2 条才查）。这是为了防止"串台"——A 会话讨论的内容跑到 B 会话的上下文里。跨会话记忆还会被标记 `isCrossConversation: true`，注入时带 ⚠️ 提示 LLM"这是别的会话的内容，谨慎引用"。

**员工隔离**：`memory_chunks.agent_id` 按员工隔离（核实报告 ch09-12 设计要点 6）。小品（产品）的记忆不会污染阿码（研发）的上下文。每个数字员工有自己的记忆空间。

**项目隔离**：检索默认带 `projectId` 过滤，只在项目范围内查。综合区（`projectId='_general'`）不传 projectId 检索全库，但这是显式选择，不是默认行为。

**这三重隔离对应 context engineering 里"isolate"的不同维度**：横向（会话间）、纵向（员工间）、范围（项目间）。没有隔离，记忆会互相污染，LLM 会被无关上下文带偏。

---

## 可观测 snapshot：注入了什么，事后能查

context engineering 最容易被忽视的一环是可观测性。你调了 top-k、调了阈值、调了三区比例——怎么知道效果好坏了？靠猜不行，必须有数据。

WinMatrix 的做法是**每轮注入都产生一个 snapshot 事件**（核实报告 ch09-12 设计要点 7）：

> 上下文可判定性：每轮注入有明确上限——记忆 top-k=5、对话历史 getRecentMemory(10)、工作上下文 getWorkingContext(10)。**观测 `llm_context_snapshot` 事件**。

这个 snapshot 记录的是"这次 LLM 调用实际注入了什么"——哪些记忆被选进来、各自的相关度分数、来自哪个区、历史取了几条、工作上下文是什么。事后排查"LLM 为什么这么回答"时，这个 snapshot 是关键证据。

**没有 snapshot 的 context engineering 是黑盒调参。** 你改了 top-k 从 5 到 8，效果好了还是坏了？如果不知道每次实际注入了什么，你只能靠用户反馈猜——而用户反馈是延迟的、模糊的、样本极小的。有了 snapshot，你能精确对比"这次回答差，是因为记忆没检索到，还是检索到了但相关度低，还是注入了但 LLM 没用上"。

这一层特别重要，值得单独强调：**context engineering 不是"一次性设计"，而是"持续调优"。** 没有可观测性，调优无从谈起。

---

## 注入链路全景

把上面四步串起来，一次 LLM 调用的上下文注入全景：

```
用户消息进来
   │
   ├─【Write 池子已就绪】
   │     会话层（Redis 50 条）→ 转录层（PG session_transcript）→ 长期索引（ES+PG）
   │
   ├─【Select 注入】TurnRunner 三路并行
   │     ├─ 长期记忆：三区检索（Zone1: 3 条 / Zone2: 3 条 / Zone3: 1 条）→ top-k=5
   │     ├─ 对话历史：getRecentMessages(10)
   │     ├─ 工作上下文：getWorkingContext(10)
   │     └─ 项目上下文：ProjectContextSource body
   │
   ├─【Compress 已隐式完成】
   │     检索即压缩（千条 → 5 条）+ 降采样的源点暂存
   │
   ├─【Isolate 已生效】
   │     会话/员工/项目三重隔离 + 跨会话标记 ⚠️
   │
   ├─ 格式化：类型标签 + 相关度 + 来源
   │
   ├─ 注入 LLM
   │
   └─【可观测】发 llm_context_snapshot 事件
         记录"这次实际注入了什么"供事后排查
```

每个环节都有明确的数字和机制，不是"塞给 LLM 看看效果"的黑盒。

---

## 用业界四角度对照总结

回到开头那个框架，WinMatrix 的实现对照：

| 策略 | WinMatrix 实现 | 关键数字/机制 |
|------|---------------|-------------|
| Write | 三层记忆池 + hash 去重双写 | 会话 50 条 / 转录含 tool trace / 长期 ES+PG |
| Select | 三区检索 + 明确 top-k | 记忆 top-k=5 / 历史 10 / 工作上下文 10 |
| Compress | 检索即压缩 + 降采样前暂存 | 三区门槛 0.25/0.5/0.8 / 完整 request 源点暂存 |
| Isolate | 会话/员工/项目三重 | agent_id 隔离 / projectId 过滤 / 跨会话 ⚠️ |

这四个角度都覆盖了，而且每个都有明确的数字和机制。**这不是"我们做了 context engineering"的口号，而是可工程化、可调参、可观测的实现。**

---

## 给后来者的几条总结

1. **context engineering 比 prompt engineering 更重要。** prompt 是一句话，context 是每轮所有信息的总和。Agent 表现差，往往不是 prompt 不好，而是 context 没选对。
2. **每个注入来源都要有明确 top-k 上限**。记忆 top-k=5、历史 10、工作上下文 10——这些数字不是拍脑袋，是 token 预算和注意力稀释的权衡。
3. **检索要分区，按相关性梯度调阈值**。WinMatrix 三区（当前会话/项目记忆/跨会话）门槛从 0.25 升到 0.8，近的宽松、远的严格，防串台。
4. **跨来源的内容要标记隔离**。跨会话记忆带 ⚠️，员工记忆按 agent_id 隔离，项目记忆按 projectId 过滤。
5. **注入要带类型标签**。[decision/finding/preference/constraint] 让 LLM 知道每条记忆的性质，比无差别文本有用得多。
6. **压缩前必须有源点暂存**。任何降采样/compact 都是不可逆损失，压缩前存原始版本，否则出问题无从 debug。
7. **必须有 llm_context_snapshot 可观测**。没有 snapshot 的 context 调优是黑盒，改了 top-k 不知道好坏。
8. **context engineering 是持续调优，不是一次性设计**。数字会随业务变化，关键是有一套可观测、可调参的机制，而不是死守一组"最佳参数"。

每轮注入什么、注入多少，是 Agent 系统最核心的工程问题之一。WinMatrix 的答案不是"一个魔法 prompt"，而是"分区检索 + 明确上限 + 类型标签 + 可观测 snapshot"这套组合拳。这套思路比任何 prompt 技巧都更能决定 Agent 的实际表现。

---

> **下一篇**：[《安全护栏多层防御：认证、授权、输入、执行、基础设施》](./28-security-guardrails.md)——context engineering 解决了"喂什么"，但还有一个更基础的问题——喂进去的东西安不安全？prompt 注入、越权调用、敏感数据泄露，每一类威胁都要有专门的护栏。
