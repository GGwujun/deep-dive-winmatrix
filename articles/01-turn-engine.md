# 把 LLM 包进一个 Turn：Agent 执行引擎的设计与踩坑

> 这是《WinMatrix 开发经验文集》第 1 篇。讲一个最基础也最容易做歪的问题：当用户发来一条消息，到 LLM 给出回答之间，到底发生了什么。代码来自 WinMatrix 后端真实实现。

很多人第一次做 Agent，写出来的核心循环大概长这样：

```python
while not done:
    response = llm.chat(messages, tools=tools)
    if response.tool_calls:
        for call in response.tool_calls:
            result = execute(call)
            messages.append(result)
    else:
        done = True
return response.content
```

跑起来也能用。但你很快会撞上一连串问题：用户中途想停下来怎么办？工具调用卡死了整个 Turn 怎么办？跑了 50 轡还没收敛怎么办？流式输出和工具调用冲突怎么办？一个 Turn 执行到一半进程崩了，重启后怎么恢复？

这篇文章，就讲我们在 WinMatrix 里是怎么把"一个 Turn"做扎实的。

---

## 什么是 Turn：先把这个概念立住

在 WinMatrix 里，**Turn 是一次完整的"用户消息 → 系统响应"闭环**。它不是一次 LLM 调用——一次 Turn 内部可能发生多轮 LLM 调用、多次工具调用、多次流式输出。

Turn 是我们对 Agent 执行的最小可观测、可恢复、可中断的单元。把这个边界立住，后面所有的工程（并发控制、中断、可观测性、持久化）才有依附。

一个 Turn 的生命周期，在我们的 `TurnRunner` 里被组织成一条**单轨编排**（single-track orchestration）：

```
load（加载上下文）
  → admitTurn（准入门：语义分诊、前置校验）
    → resolveRoute（路由决策：选员工、选技能、选模式）
      → assembleExecution（装配并执行：工具循环、流式输出）
        → 持久化、发布 terminal
```

这条链路是严格顺序的，每一步都有明确的输入输出和终止条件。为什么是单轨而不是异步图？因为**可读性和可调试性**——当一个 Turn 跑飞了，你需要能顺着这条轨道一步步还原"它走到哪、为什么停"。异步图省下来的那点并行时间，远抵不上调试时的痛苦。

---

## 第一个工程问题：把上下文加载的串行等待合并掉

一个 Turn 开始，要先加载三样东西：会话上下文、Agent 上下文、最近的历史消息。最直觉的写法是串行：

```typescript
const turnCtx = await loadPipelineContext(conversationId);
const agentCtx = await resolveAgentContext(conversationId);
const history = await getRecentMessages(conversationId, 30);
```

但这三件事**只依赖 `conversationId`，彼此没有依赖关系**。串行意味着三次 RTT 的等待被叠加。我们的做法是用 `Promise.all` 把它们合并：

```typescript
// agents/core/agent/turn/TurnRunner.ts（第 102-108 行）
const [turnPipelineContext, context, historyResult] = await Promise.all([
  TurnContextLoader.loadPipelineContext(conversationId),
  resolveAgentContext(conversationId),
  useCausalHistory
    ? Promise.resolve({...})
    : getHistoryAdapter().getRecentMessages(conversationId, 30),
]);
```

一次 RTT 的时间，干完三件事。这是个很小的优化，但它在每一条消息上都会发生——积少成多。

**教训：任何"看起来必须按顺序"的初始化，先问一句"它们真的有依赖吗？"** 很多时候顺序只是写代码的人顺手写的，不是逻辑必需的。能并行的并行掉，是最低成本的延迟优化。

---

## 第二个工程问题：工具调用循环什么时候停

这是 Agent 引擎最核心、也最容易出事的地方。先看我们的真实循环：

```typescript
// agents/core/ai-execution/tool-loop/StreamingToolExecutor.ts（第 1026-1052 行）
while (iteration < maxIterations && totalLlmRounds < hardCapLlmRounds) {
  if (roundBudgetMs !== undefined && Date.now() - loopStartAt > roundBudgetMs) {
    logger.warn(`[StreamingToolExecutor] 超出单轮预算 ${roundBudgetMs}ms，提前结束`);
    this.loopAbortController?.abort('round_budget_exceeded');
    return finalizeSuccess({
      output: output || `已达到执行预算（${roundBudgetMs}ms），先返回当前结果；如需继续请明确下一步。`,
      intermediateSteps, iterations: iteration,
      terminationReason: 'round_budget_exceeded',
    });
  }
  iteration++;
  totalLlmRounds++;
  // 每轮均流式调用（思考+文本实时推送）
  let assistantMessage = await this.callLLMStreaming(clientWithTools, messagesForLlm);
  // ...
}
```

这里有**三道终止闸门**，每一道都在防一种"跑飞"：

### 闸门一：迭代次数（maxIterations）

`iteration < maxIterations` 是最基本的——工具调用循环不能无限跑下去。但光看迭代次数不够，因为有 loopholes。

### 闸门二：LLM 轮次硬上限（hardCapLlmRounds）

注意循环条件是 `iteration < maxIterations AND totalLlmRounds < hardCapLlmRounds`，两个都要满足。为什么要有两个计数器？

因为存在一类"豁免工具"（meta tool），它们的轮次不计入 `iteration` 配额，但会消耗 LLM 调用。`hardCapLlmRounds = maxIterations + 20`（默认加 20）是一个**硬性兜底**——无论豁免逻辑怎么走，LLM 调用的总次数不能超过这个绝对上限。

```typescript
// StreamingToolExecutor.ts（第 269 行）
function resolveMetaToolLlmRoundHardCap(maxIterations: number): number {
  return maxIterations + AGENT_META_TOOL_EXTRA_LLM_ROUNDS;  // 默认 +20
}
```

**教训：当你给某个机制开了"豁免/特例"，一定要为这个豁免本身设一个兜底。** 否则豁免就会被滥用（或被 LLM 钻空子），最终绕过你原来的保护。"信任但要核实"在 Agent 系统里尤其重要。

### 闸门三：单轮预算（roundBudgetMs）

前两个是计数闸门，这个是**时间闸门**。即使迭代次数没到，只要这一轮的总耗时超过预算，立即停止，并返回一个带 `terminationReason: 'round_budget_exceeded'` 的"部分结果"。

注意停止时的输出：

```typescript
output: output || `已达到执行预算（${roundBudgetMs}ms），先返回当前结果；如需继续请明确下一步。`
```

不是抛错，而是**优雅地返回已经完成的部分，并告诉用户怎么继续**。这一点很重要：Agent 跑超时不该是一个"崩溃"，而该是一个"暂停"。用户看到的是"我先做到这儿了，你要继续就告诉我"，而不是一个红色错误。

### 三道闸门的设计哲学

```
开始循环
   │
   ▼
iteration < max? ──否──► 按迭代数终止
   │是
   ▼
LLM 轮次 < hardCap? ──否──► 硬上限兜底（防豁免滥用死循环）
   │是
   ▼
超时间预算? ──是──► 优雅返回部分结果(round_budget_exceeded)
   │否
   ▼
执行一轮 ──► (回到 iteration < max?)
```

三道闸门分别防三种失败：**无限循环、豁免滥用、单次卡死**。一个生产级 Agent 引擎，这三个缺一不可。只防第一个的循环，上线必炸。

---

## 第三个工程问题：流式输出，和工具调用天生打架

LLM 的流式输出（streaming）和工具调用（tool calling）有一个内在矛盾：

- **流式**要求 LLM 边生成边吐 token，越早吐越好，体验好。
- **工具调用**要求 LLM 把 `tool_calls.arguments` 生成完整，否则没法执行。

而流式模式下，有些模型的 `tool_calls.arguments` 会分多次 delta 返回，甚至偶尔返回**空的 arguments**。如果你直接拿空 arguments 去执行工具，必然出错。

我们的处理是：**流式返回空 args 时，降级用非流式补全**：

```typescript
// StreamingToolExecutor.ts（第 1087-1108 行）
// 流式返回 tool_calls arguments 为空时用非流式补全
```

背后的 LLM 客户端为此准备了**双轨接口**：

```typescript
// infrastructure/llm/core/types.ts（第 344-399 行）
export interface ILLMClient {
  // 流式：用于对话，思考+文本实时推送
  stream(messages, options?): AsyncIterable<LLMStreamEvent>;
  // 非流式：用于工具调用场景，确保 tool_calls.arguments 完整
  invokeNonStreaming();
  // 真正 stream:false，用于工具调用场景，确保 tool_calls.arguments 完整
  withTools(tools): ILLMClient;
}
```

`stream()` 服务于对话体验，`invokeNonStreaming()` 服务于工具调用的正确性。**这是两套不同的诉求，不该硬塞进一个接口。**

**教训：流式和工具调用是两种语义不同的 LLM 交互，不要试图用一个"万能 stream 接口"同时搞定。** 该流式时流式，该阻塞拿完整结果时阻塞。承认这个矛盾，比假装它不存在要好。

---

## 第四个工程问题：用户想停下来，得真能停下来

用户在 Agent 跑得正欢时点了"停止"，你能不能真的停下来？

很多人实现的"停止"是软停止——设个 flag，等当前工具调用完才停。这不行。一次工具调用可能要几十秒（比如调一个外部 API），用户点停止后还要等几十秒，体验极差。

我们的做法是**AbortController + 全局注册表**：

```typescript
// interface/api/agents/ws/agentWebSocketState.ts（第 12 行）
export const conversationAbortControllers = new Map<string, AbortController>();
```

每个正在进行的 Turn，启动时往这个 Map 里注册一个 `AbortController`，key 是 `conversationId`。用户点停止时：

```typescript
// interface/api/agents/agentWebSocket.ts（第 180-254 行，stop 控制帧）
if (rawParsed.data?.type === 'stop') {
  const convId = String(rawParsed.data.conversationId ?? '');
  const abortCtrl = conversationAbortControllers.get(convId);
  if (abortCtrl) {
    abortCtrl.abort();   // 触发 AbortError，贯穿整个调用链
    // ...发送停止回执
  }
}
```

`abort()` 会触发一个 `AbortError`，这个信号能贯穿整个异步调用链——正在进行的 LLM 流式请求、正在等待的工具调用，全都会被中断。这是**硬停止**，几乎瞬时生效。

而且 stop 是一个**绕过 AI 管道的控制帧**——它不经过决策引擎、不进 Turn 队列，直接查注册表、abort、回执。停止必须比执行有更高的优先级。

**教训：长任务的"取消"必须是一个一等公民，而不是事后补的 flag。** 用 AbortController 把取消信号做成贯穿调用链的基础设施，而不是每个环节各自 poll 一个 flag。前者瞬时，后者拖泥带水。

---

## 第五个工程问题：Turn 执行完了，得能回放和审计

一个 Turn 跑完了，里面发生了几轮 LLM 调用、调了哪些工具、每步耗时多少、最后为什么停——这些信息事后必须能查到。否则生产事故来了你两眼一抹黑。

我们在 Turn 和持久化之间放了一个**内存桥梁**叫 `AgentExecutionTicket`，它聚合了决策结果、上下文、运行时状态，持久化时落到一棵表树上：

```
agent_run（一次完整多步执行根聚合）
  ├── agent_run_decision（决策阶段审计，按 stage）
  ├── agent_run_step（步骤级执行记录）
  ├── agent_run_state（编排状态外置）
  └── agent_worker_execution（worker 执行，含 attemptNo 支持重试）
```

一个 `turnAgentRunId` 贯穿决策 → 执行 → 评审三个阶段，把整棵表树串起来。事后排查，一条 SQL 就能把一次对话的完整执行轨迹拉出来。

这是 Agent 可观测性的基础。我们在后面专门有一篇（第 8 篇）讲 ExecutionSpan，这里先记住一点：**Turn 是执行单元，agent_run 是它的持久化投影，两者通过一个 ID 贯穿。**

---

## 第六个工程问题：连接、密钥这些资源的生命周期

最后一个容易被忽略的点。一个 Turn 执行时，会"借"一些昂贵资源——LLM binding（带密钥的客户端）、数据库连接、工作站租约。这些资源不能无限占用，也不能用完就扔（下一个 Turn 还要用）。

我们的做法是在 Turn 的 `finally` 块里，按**所有权是否移交**来决定是否释放：

```typescript
// TurnRunner.ts（第 302-308 行）
finally {
  // 未移交 ticket 则 releaseLlmExecutionBinding
  // 成功产出 ticket 时所有权移交给 role/CDW 生命周期，不在这里释放
}
```

关键判断是 `bindingHandedOff`：如果 Turn 成功产出了 ticket，LLM binding 的所有权就移交给了下游（role 或 coding workstation），这里不释放；如果失败、没产出 ticket，立即释放，避免泄漏。

**资源所有权的移交，比"谁申请谁释放"更清晰。** 一个资源在生命周期里可能有多个持有者，明确"现在归谁管"，比让申请者负责到底要可靠。

---

## 一个 Turn 的完整时序

把上面六点串起来，一个 Turn 的完整时序是这样的：

```
用户 --发送消息--> WebSocket入口
WebSocket入口 --run(options)--> TurnRunner
TurnRunner: Promise.all 加载上下文(三路并行) → admitTurn 准入门 → resolveRoute 路由决策
TurnRunner --executePreparedTurn--> StreamingToolExecutor

  [工具循环，受三道闸门约束]
  StreamingToolExecutor --callLLMStreaming--> LLM
  LLM --流式 token + 可能的 tool_calls--> StreamingToolExecutor
    有 tool_calls: --executeToolCall--> 工具 --> 结果 --> 继续循环
    无 tool_calls/触发终止: --finalize--> 跳出循环

TurnRunner: 持久化 agent_run 树 → 发布 terminal
WebSocket入口 --流式输出 + 结束--> 用户

随时可中断：
  用户 --stop 控制帧(绕过 AI 管道)--> WebSocket入口
  WebSocket入口: abortController.abort()
  → AbortError 贯穿调用链，瞬时停止
```

---

## 给后来者的几条总结

1. **把 Turn 这个边界立住。** 它是你做并发、中断、恢复、可观测性的基础。没有清晰的 Turn 边界，这些全都做不到位。
2. **工具循环要设三道闸门**：迭代次数、LLM 硬上限、单轮时间预算。只设一道，上线必炸。记得给"豁免"本身设兜底。
3. **流式和工具调用是两种语义，用两个接口。** 别用一个万能 stream 接口硬扛。
4. **停止必须是一等公民。** 用 AbortController 做贯穿调用链的硬停止，别用 flag 软停止。
5. **初始化里能并行的并行掉。** 那些"看起来必须按顺序"的加载，多半只是顺手写的。
6. **Turn 和持久化之间用一个 ID 贯穿。** agent_run 是 Turn 的投影，事后能回放是生产底线。
7. **资源用所有权移交管理生命周期。** 比"谁申请谁释放"清晰得多。

一个扎实的 Turn 引擎，是 Agent 平台的脊梁骨。它不性感，但每一个跑在生产上的 Agent 系统，最终都会被这些"无聊"的工程细节决定生死。

---

> **下一篇**：[《让 AI 自己决定做什么：渐进式决策引擎的三层边界》](./02-decision-engine.md)——消息进来了，该让哪个员工、用哪个技能来响应？决策引擎怎么做这件事，又怎么不烧钱。
