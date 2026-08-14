# 协作催促 CollaborationFollowup：被同事卡住了，系统怎么催

> 这是《WinMatrix 开发经验文集》第 48 篇。上一篇讲单个数字员工的"专长画像"。这一篇转到协作场景：当大福（项目总指挥）让小品（产品）写 PRD，但小品正在忙别的事，大福就卡住了。真人团队里你会去催一下同事；在数字员工团队里，谁来催、怎么催、催了还不回怎么办？这一篇讲 WinMatrix 的 CollaborationFollowup——一个把"催促"做成完整可观测生命周期的设计。

先还原一个真实的协作场景。

大福在推进一个项目，它派了一条消息给小品："帮我写个 PRD，需求是 X"。这条消息进入小品的 role_inbox（角色持久收件箱），等小品处理。但小品此刻正在处理另一条消息——它的会话正忙。大福的这一条就排在小品的队列里，没人理。

如果是真人团队，大福会隔一会儿走到小品工位说"那个 PRD 弄得怎么样了？"。但在数字员工团队里，**大福自己也是个 Agent，它的执行流已经走完了（派完任务就 return 了），不会主动去催**。于是这条消息可能在小品的收件箱里躺很久，整个项目就卡在这。

这就是"协作阻塞"。WinMatrix 的解法是 CollaborationFollowup——一个独立的延时催促机制：检测到阻塞 → 创建延时任务 → 到点触发 → 记录全生命周期。**催促不是附属于业务流程的副作用，而是一种独立的一等事件。**

---

## 第一步：阻塞怎么被检测出来

要催，先得知道"被卡住了"。阻塞检测的前提是 role_inbox 的状态可见。

role_inbox 这张表（核实报告 ch18-22）是角色消息的持久收件箱：

```
role_inbox（Interactive role durable inbox）
├── event_id            事件 ID
├── role_id             归属角色
├── digital_employee_id 具体员工
├── conversation_id     会话
├── event_type / payload
├── idempotency_key     ← 幂等键（防重复入队）
├── status              pending / claimed / running / done / failed
├── claim_owner         ← 抢占者
├── claim_expires_at    ← 租约过期时间
├── retry_count / max_retries
└── turn_id             ← 关联的轮次
```

入队时（`RoleInboxEnqueueService.ts:59-90`）有个关键前置检查——Agent 空闲探测：

```ts
async enqueue(input: EnqueueRoleEventInput): Promise<EnqueueRoleEventResult> {
  const turnId = input.turnId?.trim() || `turn_${randomUUID()}`;
  const routing = validateInteractiveRoleEventRouting(toRoutingProbeEvent(inputWithTurnId));
  if (!routing.ok) throw new Error(`interactive_runtime_routing_error:${routing.reason ?? 'pipeline_assignment'}`);
  const wasAgentIdle = input.conversationId
    ? await probeAgentIdle(input.roleId, input.conversationId)
    : false;
  try {
    record = await roleInboxRepository.insertQueuedEvent({ ...input, payload, turnId });
  } catch (error) {
    if (error instanceof RoleInboxDuplicateEventError) {
      record = error.existing;
      deduplicated = true;
    }
  }
}
```

注意 `probeAgentIdle` 这一步——入队时就探测了目标角色忙不忙。如果忙，这条消息一进队列就已经是"潜在阻塞"了。`wasAgentIdle` 这个标志位会被记录下来，作为后续是否要创建催促的依据之一。

阻塞的判定逻辑大致是：**一条消息进了某个角色的 role_inbox，但目标角色当前正忙（claim_owner 被别人占着、或者有更早的 pending/running 消息），且这种"忙"持续超过预期。** 这就是触发催促的信号。

这里有个设计取舍：阻塞检测不是"实时打断"，而是"延时观察"。因为短时间的排队是正常的——小品处理完手头的事，自然会轮到大福的请求。催促只在"等待时间超出合理预期"时才触发。这个"合理预期"就是 `delayMinutes` 字段。

---

## 第二步：CollaborationFollowup 的生命周期

检测到阻塞后，系统不是立刻催，而是创建一个**延时任务**。看 CollaborationFollowup 这张表（核实报告 ch18-22）：

```
CollaborationFollowup（协作阻塞检测后创建的延时跟进任务）
├── conversationId          会话
├── blockedByRoleId         被谁阻塞（目标角色）
├── blockedByEmployeeId     具体哪个员工
├── followUpMessage         催促时要发的话
├── reason                  阻塞原因
├── delayMinutes            延迟多少分钟才催
├── bullJobId               ← 对应的 BullMQ job ID
├── status                  pending → triggered → completed
├── projectId
├── scheduledAt             计划触发时间
├── triggeredAt             实际触发时间
└── completedAt             完成时间
```

几个字段值得细看。

**`delayMinutes` + `scheduledAt`：不是立刻催，是延时催。** 这模拟的是真人协作的节奏——你不会同事一忙就立刻催，你会等一会儿。delayMinutes 就是这个"等一会儿"的预期。不同的协作场景延迟不同：紧急任务可能 5 分钟就催，日常协作可能 30 分钟。这个值由创建时的上下文决定。

**`bullJobId`：催促本身是个 BullMQ 任务。** 这是最关键的设计——催促不是一个定时器、不是一个线程，而是一个**进队列的、可重试的、可观测的 BullMQ job**。看这个字段的设计意图：

```
创建 Followup
    ↓
BullMQ.add(jobId, { followupId }, { delay: delayMinutes * 60_000 })
    ↓
记录 bullJobId 回 Followup 表
    ↓
status = 'pending'，等 BullMQ 到点投递
```

把催促做成 BullMQ job 有三个好处：

1. **可靠性。** 进程崩了，BullMQ 的 delayed job 还在 Redis 里，重启后继续等。如果是 `setTimeout`，进程一死催促就丢了。
2. **可取消。** 如果阻塞在催促触发前就解除了（小品处理完大福的请求了），可以 `BullMQ.remove(bullJobId)` 把 job 删掉，催促就不会发。`status` 会从 pending 直接跳到 completed（cancelled）。
3. **统一观测。** 催促和所有其他后台任务走同一套 BullMQ 监控，大盘上能看到"当前有多少 pending 的催促、多少 triggered、多少 completed"。

**`status` 三态：pending → triggered → completed。**

```
pending    延时 job 还在等，没到点
   │
   │ BullMQ 到点投递，worker 拿到 job
   ↓
triggered  催促消息已发出给目标角色
   │
   │ 目标角色响应（处理了原消息 / 明确回复）
   ↓
completed  催促闭环
```

这个状态机的每个跃迁都有时间戳（`scheduledAt` / `triggeredAt` / `completedAt`）。**这意味着一次催促从"计划"到"触发"到"闭环"的全过程，都是可追溯的。**

---

## 第三步：催促也是一种可观测事件

 CollaborationFollowup 最值得称道的设计，是它把"催促"当成了一个一等公民来观测。

为什么要这么郑重？因为在一个全是 Agent 的协作系统里，**"催促"本身就是一个关键的协作信号**。一次催促被触发，意味着：

- 某个协作链路出现了非预期延迟
- 某个角色可能过载（总是被催的那个）
- 某些任务的优先级设置可能不合理（重要任务排了很久才被催）

这些信号如果埋在业务日志里，永远没人看。但如果 Followup 是个独立的、带生命周期的实体，你就可以：

- **统计哪个角色最常被催** → 它可能过载了，考虑扩容或调并发
- **统计平均阻塞时长** → 协作效率指标
- **统计催促后的响应时长** → 角色响应能力的指标
- **找出"催了也不回"的异常 case** → 可能是死锁或 bug

这些都建立在 Followup 是结构化实体、status 可查、时间戳齐全的基础上。**催促不是 fire-and-forget 的副作用，而是可观测、可统计、可归因的一等事件。**

---

## 与 role_inbox 的配合：两套机制各管什么

讲到这里要理清一个容易混淆的点：role_inbox 和 CollaborationFollowup 是两套独立的机制，它们各管各的。

| 机制 | 管什么 | 谁推进 |
|------|--------|--------|
| role_inbox | 消息的接收/排队/抢占/重试 | roleInboxWorker（租约抢占 + 重试） |
| CollaborationFollowup | 阻塞检测后的延时催促 | BullMQ delayed job + 催促 worker |

```
大福 → 派消息给小品
        ↓
   role_inbox 入队（status=pending）
        ↓ probeAgentIdle 发现小品忙
        ↓ 超过预期没被 claim
   CollaborationFollowup 创建（delayMinutes 后催）
        ↓ delayMinutes 后 BullMQ 投递
   催促消息发出（status=triggered）
        ↓ 小品响应
   role_inbox 消息被 claim → done
   CollaborationFollowup status=completed
```

role_inbox 管"消息怎么被可靠地递交给目标角色并处理"，Followup 管"消息迟迟没被处理时怎么催"。前者是消息投递语义，后者是协作节奏语义。**分开建，是因为它们的关注点不同——投递关注可靠性（不丢、不重、可重试），催促关注协作健康度（别让协作卡死）。**

注意 role_inbox 自己也有重试（`retry_count` / `max_retries`），但那个重试是"处理失败后重试"，不是"没人理时催促"。两者的触发条件完全不同：重试是 `status=failed` 时触发，催促是 `status=pending 且超时` 时触发。不能混为一谈。

---

## 延时催促的取舍：为什么不用优先级队列

读到这里，有人会问：为什么不直接给 role_inbox 设优先级？大福的请求优先级调高，小品的队列里它先被处理，不就不用催了吗？

这是个好问题，答案是：**优先级队列解决的是"谁先被处理"，催促解决的是"目标角色压根没空处理任何东西"。**

小品正忙，不是因为队列里有更低优先级的消息挡着大福的请求，而是因为小品这个角色当前的执行槽被占满了（它正在执行一个 Turn）。这种"忙"不是队列调度能解决的——你把大福的请求插到队首，小品当前这个 Turn 不跑完，队列首部也启动不了。

所以催促的本质，是**给正在忙的角色一个"你有别的事在等着，处理完手头的来看看"的提醒**。它不改变处理顺序（顺序还是 role_inbox 自己的调度决定），它只是确保"被卡住的协作不会无限期沉默"。

delayMinutes 的设计也呼应这点：它不是"X 分钟后插队"，而是"X 分钟后发一条提醒消息"。提醒到了，目标角色在下一个空闲窗口自然会看到并处理。

---

## 给后来者的几条总结

1. **多 Agent 协作里，"催促"是刚需，不是可选。** Agent 自己不会主动催（它的执行流已经 return 了）。没有一个独立的催促机制，协作链路会无限期卡在"等同事"。
2. **催促要做成延时任务，不是立刻打断。** 短时排队是正常的，只有超出预期才催。delayMinutes 模拟真人协作的节奏。
3. **催促用 BullMQ delayed job 实现，别用 setTimeout。** 进程会崩，setTimeout 会丢，BullMQ 在 Redis 里持久化，重启后继续。
4. **催促要可取消。** 阻塞在催促触发前解除，要能删 job、转 status。别让已经不需要的催促还发出去。
5. **催促是独立的一等事件，不是业务流程的副作用。** 给它独立的实体（Followup 表）、独立的状态机（pending→triggered→completed）、独立的时间戳。
6. **催促数据是协作健康度的金矿。** 谁最常被催、平均阻塞多久、催了多久才回——这些指标只能从结构化的 Followup 实体里统计出来，埋在日志里等于没有。
7. **催促和 role_inbox 重试是两回事。** 重试是"处理失败后重跑"，催促是"没被理睬时提醒"。触发条件不同、机制不同，别混建。
8. **优先级队列替代不了催促。** 优先级解决"谁先处理"，催促解决"目标角色没空处理任何东西"。忙的角色不是被低优先级消息挡住，是执行槽被占满——队列调度管不到这层。

催促这件事看起来很小，但它暴露的是多 Agent 系统的一个本质难题：**协作的节奏感，不能靠每个 Agent 的自觉，必须靠系统级的机制来托底。** 把催促做成一等公民，是让 Agent 团队"真的像团队"而不是"一群各干各的"的关键之一。

---

> **下一篇**：[《Token 凭证体系：PAT / WMA / WMEC 三种令牌各保护什么》](./49-token-broker.md)——从协作回到安全：当外部 Agent、外部接入方、真人用户都要访问 WinMatrix 时，三类令牌怎么分工，Token Broker 又怎么按前缀路由。
