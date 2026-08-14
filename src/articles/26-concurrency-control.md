# 并发控制三层模型：全局信号量、会话锁、任务租约各管一个粒度

> 这是《WinMatrix 开发经验文集》第 26 篇。前两篇讲了幂等和降级，这篇讲并发。并发问题在 AI 平台上有个特殊性——它不像传统后端只是"数据库连接池打满"，而是会从三个完全不同的层面爆炸：LLM 全局过载、同会话并发互相覆盖、任务被重复处理。这三个层面用一把锁解决不了，得用三层模型。讲 WinMatrix 怎么分层管并发。

先说三个真实的事故场景。

**场景 A：LLM 被打爆。** 系统高峰期，10 个用户同时对话，每个用户的 Agent 又并发调 3 个工具，每个工具触发一轮 LLM 调用。瞬间 30+ 个并发 LLM 请求打向 API。LLM provider 返回 429（限流），甚至暂时封号。整个系统所有用户都受影响。

**场景 B：同会话并发覆盖。** 用户在企微里连发了两条消息"@小品 写 PRD"和"@小品 改一下刚才的 PRD"。系统并发处理这两条，第二条执行时第一条还没写完，结果第二条读到的是旧状态，把第一条的工作覆盖了。用户看到的是"PRD 没改成"，实际是被并发写覆盖了。

**场景 C：任务重复处理。** 一个定时任务到期触发，BullMQ 把 job 投递给 worker。worker 处理到一半 Pod 被重启，job 被 redelivered 给另一个 worker。如果没有防护，同一个 job 被处理两次，副作用翻倍。

这三个场景对应三个完全不同的并发粒度：

- **全局粒度**：LLM 并发总数（场景 A）。
- **会话粒度**：同一会话串行执行（场景 B）。
- **任务粒度**：同一 job 不重复处理（场景 C）。

WinMatrix 用三层并发控制分别解决，各管一个粒度。这篇逐层拆。

---

## 第一层：全局信号量 scheduledAgentSem——防 LLM 过载

第一层是全局的，管"整个系统同一时刻最多有多少个 LLM 密集型任务在跑"。

入口是 BullMQ 的 scheduled-agent 队列（核实报告 ch23-29）。所有 LLM 密集型的定时任务都进这个队列，worker 消费时受一个全局信号量约束：

```ts
// startup/scheduled.ts（核实报告 ch23-29，主题1 设计要点4）
// crossAgentTriggerWorker 与 scheduled-agent 共享 getScheduledAgentSemFromEnv
// 超并发 → moveToDelayed + DelayedError 重投
```

信号量的容量来自配置（`config.scheduled.llmConcurrency`）。当并发任务数超过这个值时，新来的 job 不会被直接执行，而是被 `moveToDelayed` 加上 `DelayedError` 抛出——BullMQ 会把这种 job 当成"暂时失败"延迟重投。

```
BullMQ worker 取 job
   ├── 当前并发 < llmConcurrency？
   │     ├── 是 → semaphore.acquire() → 执行 → release()
   │     └── 否 → moveToDelayed（延迟重投）→ 抛 DelayedError
   └── job 重新进队列，等下次轮到
```

**关键设计：信号量是跨 worker、跨队列共享的。** `crossAgentTriggerWorker`（跨 Agent 触发器）和 `scheduled-agent`（定时 Agent 任务）用的是同一个 `getScheduledAgentSemFromEnv()`。这意味着即使你起多个 worker 进程，它们共享一个 Redis 信号量，总数受控。

为什么要全局管？因为 **LLM 的瓶颈是 provider 侧的，不是单机瓶颈**。你单个 Pod 自己并发 5 个没问题，但 10 个 Pod 各并发 5 个就是 50 个，会把 provider 打爆。必须有个跨实例的全局闸门。

这一层的几个要点：

- **粒度是"LLM 密集型任务"**，不是所有任务。轻量任务（DB 维护、扫描）走 `scheduled-system` 或 `scheduled-light` 队列，不受这个信号量约束。
- **超量不拒绝，而是延迟重投**。BullMQ 的 `moveToDelayed + DelayedError` 让 job 自动延后，不丢任务。
- **容量可配置**（`llmConcurrency`），不同部署规模可调。

---

## 第二层：会话锁 conversationRunLocks——防同会话并发覆盖

第二层是会话粒度的，管"同一会话同一时刻只允许一个 run 在执行"。

这个锁的实现在 WebSocket 会话状态里（`interface/api/agents/ws/agentWebSocketState.ts`，52 行）：

```ts
// 会话运行锁：同一会话同时只允许一次 run
export const conversationRunLocks = new Map<string, boolean>();
// 会话 AbortController 注册表
export const conversationAbortControllers = new Map<string, AbortController>();
```

`conversationRunLocks` 是个内存 Map，key 是 conversationId，value 是 boolean（占用与否）。当一条消息进来要执行时：

```
消息进来
  ├── conversationRunLocks.get(convId)？
  │     ├── true（已锁定）→ 拒绝/排队，不能并发执行
  │     └── false → set true → 执行 → finally set false/delete
  └── 异常也保证释放（finally）
```

**为什么要锁会话？** 因为 Agent 的执行是有状态的——它会读会话历史、写中间状态、产生 tool_call 副作用。如果同一会话并发执行两个 run，它们的读写会互相干扰：

- run A 读历史 → run B 写历史 → run A 基于旧历史做决策 → 错。
- run A 和 run B 都调 write_doc 工具写同一个文档 → 互相覆盖。

会话锁强制把同一会话的执行串行化。用户连发两条消息时，第二条会等第一条执行完（或者被 ConversationExecutionDispatcher 排进内存队列，参考第 8 章 CoordinatorAdapter）。

注意这个锁是**内存 Map，不是 Redis 锁**。这有个隐含约束：**它只在单实例内有效**。如果同一会话的消息被路由到不同实例（比如负载均衡随机分发），这个锁就失效了。

WinMatrix 怎么保证同一会话路由到同一实例？靠会话亲和性（session affinity）——WebSocket 长连接本身就有这个属性，一旦连接建立，这个会话的所有消息都走这个连接所在的实例。所以内存锁在 WS 场景下是够的。

但要注意：企微 webhook 这种"无连接"的入口，理论上可能把同一会话的不同消息打到不同实例。这种场景下，会话锁要升级成 Redis 锁。WinMatrix 在企微入口也用了会话路由策略来 mitigate 这个问题。

这一层的要点：

- **粒度是"单个会话"**，不影响其他会话的并发。
- **内存锁在 WS 场景够用**（会话亲和性），但跨实例入口要升级成 Redis 锁。
- **锁释放必须用 finally**，否则异常会留下"永久锁定"的会话。
- **配合 AbortController**：用户点 stop 时，从 `conversationAbortControllers` 取出 controller 调 `abort()`，中断正在执行的 run。

---

## 第三层：任务租约 claim lease——防任务重复处理

第三层是任务粒度的，管"同一个任务不会被多个 worker 同时抢走处理"。这是上一篇分布式锁讲过的 claim_token + lease 模式，这里从并发控制角度再串一遍。

这层在两个地方出现：role_inbox 和 flow_instruction。

**role_inbox 的租约抢占**（核实报告 ch18-22）：

```
role_inbox
├── claim_owner          ← 抢占者 ID
├── claim_expires_at     ← 租约过期时间
├── retry_count / max_retries
└── status               pending → claimed → running → done
```

worker 抢消息时，原子地把 status 从 pending 改成 claimed，写上 claim_owner 和 claim_expires_at。抢到才有资格处理；没抢到的 worker 直接 skip 这条。

租约过期后（claim_expires_at 到了），别的 worker 可以重新抢——这是为了处理"抢到消息的 worker 进程崩了"的情况。如果不设租约，那条消息会永远停在 claimed 状态没人处理。

**flow_instruction 的 claim_token + lease**（`FlowInstructionDispatchCoordinator.ts:71-107`）：

```ts
const instruction = await this.instructionRepository.claimInstruction({
  batchId: params.batchId,
  claimedBy: params.userId,
  leaseUntil: new Date(Date.now() + (this.options.leaseMs ?? 30 * 60 * 1000)),  // 默认 30 分钟
});
```

`claimInstruction` 是原子的（DB 行锁 + 条件更新），同一条指令只能被一个 worker 抢到。租约默认 30 分钟，期间别的 worker 不能抢；30 分钟没完成，租约过期，别人可以接手。

这层还配合"并发上限"控制：

```ts
const active = await this.instructionRepository.countActiveInstructionsByBatchId(params.batchId);
if (active.data >= Math.max(1, params.concurrencyLimit)) break;
```

`countActiveInstructionsByBatchId` 数当前批次里 claimed/running 状态的指令数，超过 `concurrencyLimit` 就不再抢新的。**这是租约 + 计数的组合：租约防单条指令重复，计数防整个批次过载。**

这一层的要点：

- **粒度是"单条任务"**。
- **claim 必须原子**（DB 行锁或 Redis SET NX），不能 select-then-update。
- **租约过期机制必须有**，否则崩溃的 worker 会留下"僵尸任务"。
- **配合计数器控并发上限**，租约只防重复，总量控制要靠计数。

---

## 三层放一起：各管一个粒度

把三层放一起对比，并发控制的"形状"就清楚了：

```
请求到来
   │
   ├─【第一层】全局信号量 scheduledAgentSem
   │     粒度：整个系统 LLM 密集型任务总数
   │     防什么：LLM provider 过载
   │     机制：Redis 信号量 + 超量 moveToDelayed
   │
   ├─【第二层】会话锁 conversationRunLocks
   │     粒度：单个会话
   │     防什么：同会话并发执行互相覆盖
   │     机制：内存 Map + finally 释放 + AbortController
   │
   └─【第三层】任务租约 claim lease
         粒度：单条任务/消息
         防什么：任务被多 worker 重复处理
         机制：原子 claim + 租约过期 + 计数控上限
```

这三层是**正交的**，解决完全不同的问题：

| 层 | 解决的并发问题 | 如果缺失会怎样 |
|----|--------------|--------------|
| 全局信号量 | LLM provider 被打爆 | 高峰期所有用户被 429，可能被封号 |
| 会话锁 | 同会话读写互相覆盖 | 用户看到"我的修改丢了"，实际被并发覆盖 |
| 任务租约 | 任务被重复处理 | 副作用翻倍（消息发两遍、文档写两次） |

**这三层缺一不可。** 只用全局信号量解决不了会话覆盖；只用会话锁解决不了 LLM 过载；只用任务租约控不住总量。它们各管一个维度，组合起来才能覆盖"AI 平台并发"这个多面问题。

---

## 为什么是三层而不是一层

新手会问：为什么不能用一把全局锁解决所有并发问题？答案分几方面。

**第一，粒度差太远。** 全局信号量管的是"几十到几百个任务"，会话锁管的是"一个会话"，任务租约管的是"一条消息"。一把锁要么粒度太粗（全局锁所有，吞吐极低），要么粒度太细（每条消息一把锁，根本防不了 LLM 过载）。不同粒度的问题必须用不同粒度的锁。

**第二，机制不同。** 全局信号量必须是跨实例的（Redis 信号量），因为 LLM 瓶颈在 provider 侧；会话锁可以是内存的（WS 会话亲和性），因为是单实例内的串行化；任务租约必须持久化（DB 行锁），因为要防 worker 崩溃后重复处理。三种机制各有最适合的实现，强行统一会别扭。

**第三，失败语义不同。** 全局信号量超量是"延迟重投"（不丢任务）；会话锁冲突是"排队或拒绝"（用户体验问题）；任务租约冲突是"别人已经在处理"（直接 skip）。三种失败的处理方式不同。

**所以分层的本质是"对不同粒度、不同机制、不同失败语义的并发问题分别求解"。** 这和第 23 篇讲幂等"别统一成一个服务"是一个道理——横切的是意识，不是实现。

---

## 一个细节：AbortController 与会话锁的配合

会话锁这层还有个值得展开的细节：用户点 stop 怎么办？

光有会话锁不够——锁住后 run 还在执行，用户点 stop 应该能立即中断。WinMatrix 用 AbortController 实现（核实报告 ch07-08，`agentWebSocket.ts:180-254`）：

```ts
// stop 消息处理（简化）
if (rawParsed.data?.type === 'stop') {
  const convId = rawParsed.data.conversationId;
  const abortCtrl = conversationAbortControllers.get(convId);
  if (abortCtrl) {
    abortCtrl.abort();   // 触发 AbortError
    // 发送 stopped_by_user 错误回执
  }
}
```

每个会话的 run 启动时，创建一个 AbortController 存进 `conversationAbortControllers`。run 的执行链路（LLM 调用、工具调用、流式输出）都监听这个 abort 信号。用户点 stop 时，调 `abortCtrl.abort()`，整条链路收到 AbortError 立即停止。

**这是"锁 + 中断"的组合：锁防并发启动，中断让运行中的 run 能被叫停。** 如果只有锁没有中断，用户点 stop 后要等 run 自然结束，体验极差。

注意 abort 之后会话锁也要释放——这通过执行链路的 finally 块保证。即使 abort 抛了 AbortError，finally 还是会清理 `conversationRunLocks` 和 `conversationAbortControllers`，不会留僵尸状态。

---

## 给后来者的几条总结

1. **AI 平台的并发问题分三层，不是一个问题。** 全局 LLM 过载、会话并发覆盖、任务重复处理，各管一个粒度，各用一套机制。
2. **全局信号量防 provider 过载**。跨实例共享的 Redis 信号量，超量 `moveToDelayed` 重投，容量可配置。
3. **会话锁防同会话覆盖**。内存 Map + finally 释放，WS 场景靠会话亲和性，跨实例入口要升级 Redis 锁。
4. **任务租约防重复处理**。原子 claim + 租约过期 + 计数控上限，worker 崩溃后租约过期别人可接手。
5. **三层正交，缺一不可**。粒度、机制、失败语义都不同，强行统一成一把锁行不通。
6. **会话锁要配合 AbortController**。锁防并发启动，中断让运行中的 run 能被叫停，用户点 stop 能立即响应。
7. **锁释放必须用 finally**。任何异常路径都要清理锁状态，否则会留"永久锁定"的僵尸会话。
8. **横切的是并发意识，不是统一实现。** 每一层的并发问题单独求解，比抽象一个 ConcurrencyService 更灵活。

并发控制是 AI 平台最容易被忽视的工程难点。demo 阶段单用户单任务，什么并发问题都没有；一上生产，多用户、多会话、多 worker、多 Pod 同时跑，三层问题同时爆。提前把这三层想清楚、各做各的机制，比事后补丁强得多。

---

> **下一篇**：[《Context Engineering 实战：每轮注入什么、注入多少》](./27-context-engineering.md)——并发解决了"能跑多少"，但 Agent 跑得好不好还取决于"喂给 LLM 什么"。每轮该注入多少记忆、多少历史、多少工作上下文？WinMatrix 用明确的 top-k 上限和可观测 snapshot 回答这个问题。
