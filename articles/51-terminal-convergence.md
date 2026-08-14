# 终态收敛：分布式系统里"未完成"比"失败"更危险

> 这是《WinMatrix 开发经验文集》第 51 篇，也是"工程哲学"系列的第一篇。前面 50 篇散落着很多看起来各不相干的设计——reconcileStaleRuns、llmCallWatchdogSweeper、ScheduledTaskRun 的 outbox 字段、CodingTask 的 attemptNo 防覆盖、flow_instruction 的租约过期。这一篇要把它们背后同一个思想抽出来：**running 是个毒状态，所有长流程都必须有强制收敛到终态的兜底。** 在分布式系统里，"未完成"比"失败"危险得多——失败是已知，未完成是未知。

先讲一个让我们吃过苦头的认知。

新手做后端，注意力通常放在"怎么让操作成功"和"怎么处理失败"上。成功走 happy path，失败抛异常走 catch——这两种都是**终态**。系统在任何时刻，每个操作要么成功、要么失败，干净利落。

但真实的生产系统不是这样的。一个操作除了"成功"和"失败"，还有一个状态叫**"未完成"**——它开始了（status=running），但既没成功也没失败，就那么挂着。在单机同步系统里，"未完成"是个极短暂的中间态（几毫秒），可以忽略。但在分布式系统里，"未完成"可能是一个**长期持续**的状态，而且是最危险的状态。

为什么危险？因为系统的所有决策——资源分配、并发控制、用户反馈、统计监控——都建立在"我知道每个操作现在是什么状态"的假设上。一个长期 running 的操作，系统不知道它到底成没成、会不会成、什么时候成。于是系统会：

- 给用户显示"任务进行中"，永远转圈
- 占着会话锁，新的请求进不来
- 占着并发配额，新的任务被延迟
- 污染统计数字（"平均耗时"被无限拉长）
- 级联阻塞依赖它的下游操作

**失败不可怕，系统知道怎么应对失败。未完成才可怕，因为系统不知道怎么应对一个"永远不来结果的开始"。**

WinMatrix 在这条原则上摔过跤（第 36 篇的悬挂 LLM 调用、第 37 篇的孤儿任务），最终在各个模块都长出了"强制收敛到终态"的机制。这一篇就把这些机制背后的统一思想讲清楚。

---

## running 是个毒状态

先把核心命题说透：**running 是个毒状态。**

为什么叫"毒"？因为它的毒性会扩散。一个 running 状态不是孤立地坏，它会通过所有依赖它的环节传播：

```
一个 agent_run 卡在 running
    │
    ├── 占着会话锁（conversationRunLocks）
    │     → 该会话的新消息被告知"正忙"，进不去
    │
    ├── 占着并发配额（scheduled_agent 信号量）
    │     → 新的定时任务被 moveToDelayed，延迟重投
    │
    ├── 关联的 LLM span status=pending
    │     → 统计上看起来"一直有活跃调用"
    │
    ├── ScheduledTaskRun 的 deliveryStatus=pending
    │     → 结果投递 outbox 一直积压
    │
    └── 前端永远转圈
          → 用户体验崩坏
```

一个 running 牵动五条下游。这就是为什么它叫"毒"——它不是局部的状态错误，而是会沿着依赖图全局扩散的污染。

更阴险的是，**running 状态的"毒性"是静默的**。它不报错、不崩溃、不告警——一切看起来都在正常运行，只是某个操作"还没结束"。等你从监控数字或用户反馈里发现异常时，往往已经积压了一大堆。

所以分布式系统设计的一条铁律是：**任何写进持久存储的 running 状态，都必须有对应的"强制收敛到终态"的兜底机制。** 不能指望"开始这个操作的同一个执行流来写终态"——那个执行流可能随时被打断。

---

## 四个模块，同一个思想

WinMatrix 里至少有四个独立模块，各自长出了"强制收敛"的机制。表面看它们风马牛不相及，但内核是同一个思想。一个一个看。

### 机制一：llmCallWatchdogSweeper——悬挂 LLM 调用的收敛

第 36 篇详细讲过。LLM 调用是网络请求，可能半途而废——`llm_call_start` 记了，但进程崩了 / 网络黑洞了 / BullMQ 重投了，`llm_call_end` 永远不来。这个调用就成了悬挂的 pending span。

收敛机制（`infrastructure/scheduled/llmCallWatchdogSweeper.ts:1-53`）：

```ts
export const LLM_CALL_WATCHDOG_TASK_NAME = 'system-llm-call-watchdog-sweeper';
function resolveSweepThresholdMs(): number {
  const hardMs = getConfig().llmCallHardTimeoutMs;
  return hardMs > 0 ? hardMs * 2 : 360_000;  // 2x hard timeout 或 6 分钟
}
```

每 10 分钟扫一次（interval 600_000ms），找出超过阈值还没结束的 LLM 调用，**代为补写 `llm_call_end`**，并级联 finalize 关联的 agent_run 和 scheduled_task_run。

**核心思想：不能指望"记 start 的执行流来记 end"，必须有一个独立于业务流的看门狗周期性收敛。** 这是"强制收敛"最典型的形态——业务流可能死，看门狗不能死（它是个独立的定时任务）。

### 机制二：reconcileStaleRunsOnBootstrap——孤儿任务的收敛

第 37 篇详细讲过。进程崩了，内存状态蒸发，持久存储里的 running 残骸成了孤儿。重启后这些孤儿继续占锁、占配额、搅乱监控。

收敛机制（`ScheduledTaskService.ts:2244-2298`）：

```ts
const thresholdSeconds = Math.max(
  Math.floor(config.scheduled.agentJobTimeoutMs / 1000),
  30 * 60,  // 最少 30 分钟
);
const locked = await withReconcileLock(config.reconcileAdvisoryLockKey, async () => {
  const workflowRows = await convergeRunningScheduledWorkflowRuns(500);
  const scheduledRows = await reconcileStaleRunning(thresholdSeconds);
  const pipelineRows = await getAgentRunRepository().reconcileStaleRunning(thresholdSeconds);
  const candidates = await getAgentRunRepository().listPartialScheduledRunCandidates(500);
  // ...对 candidates 逐个收敛
});
```

启动时用 advisory lock 串行化三路对账，把所有"看起来 running 但实际没 owner"的任务强制收敛到终态（failed 或 completed，看语义）。

**核心思想：既然崩溃发生在运行期，那就在启动期把脏状态收回来。** 启动期是个强一致窗口（advisory lock 保证只有一个进程在对账），适合做运行期不敢做的全量清理。

注意这里还有个细节——除了 running，**partial 状态也要收敛**（`listPartialScheduledRunCandidates`）。partial 是"部分成功"，它也是个非终态，也需要被推到终态。这呼应了命题的后半句：**不只是 running，任何非终态都是毒。**

### 机制三：ScheduledTaskRun 的 outbox——结果投递的收敛

第 16 篇讲过 ScheduledTaskRun 的投递状态机。一次定时任务执行完（终态了），它的结果还要投递出去（比如推到企微）。投递可能失败，失败要重试。

```
ScheduledTaskRun 的 deliveryStatus 状态机:
  not_requested → pending → delivering → delivered
                                          ↘ failed（重试）
```

收敛机制是独立的 outbox sweeper——每 15 秒扫一次 `sweepPendingScheduledResultDeliveries(20)`，把 pending 的投递推出去，失败的按指数退避重试（`deliveryAttempts` / `deliveryNextAttemptAt`）。

**核心思想：执行终态和结果投递解耦，各自收敛。** 任务执行完了不等于结果送达了——这是两个独立的"终态"，各要有各的收敛机制。不能假设"任务成功了投递自然就成功"。

### 机制四：CodingTask 的 attemptNo——迟到回调的收敛

第 12 篇讲过 CodingTask 的幂等设计。这里只看它和"终态收敛"相关的部分。

编码任务的终态是由**工作站回调**写入的。但回调可能迟到——attempt #1 超时了，系统启动了 attempt #2，attempt #2 成功了（终态=completed），这时 attempt #1 的迟到回调才来，想把状态改回 running。

收敛机制是 attemptNo 版本门——回调带 attemptNo，只有当 attemptNo >= 当前记录的 attemptNo 时才接受。**迟到的小号不覆盖大号。**

```typescript
attempt #1 发出 → 超时 → attempt #2 启动 → completed
                                          ↑
                            attempt #1 迟到回调来
                            attemptNo < 当前 → 拒绝
```

**核心思想：终态一旦写入，就要被保护住，不被迟到的旧版本推翻。** 这不是"收敛 running"，而是"保护已收敛的终态"——方向相反，但目标一致：确保状态最终稳定在正确的终态。

---

## 四个机制的统一内核

把这四个机制放一起，看它们的共性。

| 机制 | 收敛什么 | 触发 | 核心约束 |
|------|---------|------|---------|
| llmCallWatchdogSweeper | 悬挂的 pending LLM span | 每 10 分钟定时 | 阈值 = 2x 硬超时 |
| reconcileStaleRunsOnBootstrap | 重启后的 running 残骸 | 启动时 + advisory lock | 阈值 = max(超时, 30 分钟) |
| ScheduledTaskRun outbox sweeper | 未投递的结果 | 每 15 秒定时 | 指数退避重试 |
| CodingTask attemptNo | 迟到回调的乱序 | 每次回调校验 | 版本号门 |

它们的统一内核是三句话：

**1. 任何写进持久存储的非终态，都要有独立的收敛机制。** 不能依赖"开始它的那个执行流"来收尾——那个执行流可能死。看门狗、启动对账、outbox sweeper 都是"独立于业务执行流"的收敛者。

**2. 收敛是按类型的，不是一刀切。** 四个机制收敛四种不同的非终态，各自的终态语义不同：LLM span 要补 end、agent_run 要级联子表、投递要重试、回调要版本门。图省事用 `UPDATE SET status='failed' WHERE status='running'` 会省下代码、付出十倍代价。

**3. 收敛的依据是"超时阈值"。** 每个机制都有一个阈值——超过这个阈值还没到终态，就认为它"不可能正常结束了"，强制收敛。阈值的选择是权衡：太短会误杀真正在跑的长任务，太长会让毒状态扩散太久。WinMatrix 的选择是"至少 30 分钟"或"2 倍硬超时"，给正常任务留够窗口。

---

## 为什么"终态收敛"是个哲学问题

读到这里，有人会说：这不就是"加个定时任务扫一下"吗，算什么哲学？

是技术手段，但它背后的认知是哲学级的。这个认知是：**在分布式系统里，你要把"未完成"当成一种必须被主动消灭的状态，而不是"暂时这样、早晚自己会好"的过渡态。**

新手的心智模型：

```
操作开始 → 成功 或 失败
            （两终态，覆盖一切）
```

成熟工程师的心智模型：

```
操作开始 → 成功 或 失败 或 未完成
                         ↓
                    未完成是毒，必须主动收敛
                    （不能等它自己好）
```

这两个心智模型的差异，决定了你设计的系统在不在生产里活下去。前一个模型设计的系统，会把"未完成"当成"暂时现象"忽略掉，结果就是第 36 篇的几千个悬挂调用、第 37 篇的几百个孤儿任务——都是"未完成被忽略"的产物。

后一个模型设计的系统，从一开始就承认"未完成会发生且必须被收敛"，于是每个非终态都有对应的兜底。这些兜底平时不显眼（看起来就是几个定时任务），但它们是系统的免疫系统——没有它们，系统每运行一次都在累积脏状态，脏到一定程度就崩。

**这个认知没法靠"加个定时任务"获得，它是一种世界观——你相不相信"未完成是危险的"。** 信了，你才会在每个写 running 的地方同时设计它的收敛机制；不信，你就会觉得"加个 sweeper 是多此一举"，直到生产事故教你做人。

---

## partial 也要收敛

前面命题里有一句"partial 也要收敛"，单独展开。

很多人意识到 running 要收敛，但忽略了 partial。partial 是"部分成功"——一次多步执行，有的 step 成功了，有的失败了，整体停在中间。它不是 running（没有活跃执行在推进它），也不是 completed/failed（不是干净终态）。

partial 为什么也是毒？因为它意味着**因果链断了**。一次执行应该是"完整地走完"，partial 是走到一半停了——下游依赖这次执行结果的，不知道该不该继续。比如一次 agent_run 有 5 个 step，前 3 个成了后 2 个失败了，整体 partial——那前 3 个 step 的产出算不算数？下游能不能用？

WinMatrix 对 partial 的处理（`listPartialScheduledRunCandidates`）是**逐个判断、推到终态**：

```
partial 状态的 run
    ↓ 逐个分析它的 step
    ├── 前 3 步成功、后 2 步失败，整体不可用 → 推到 failed
    ├── 关键步骤成功、非关键步骤失败，整体可用 → 推到 completed（带标注）
    └── 状态模糊，需要人工判断 → 标记待处理
```

这比 running 的收敛更难——因为 partial 到哪个终态，要理解业务语义。一刀切地 partial→failed 会丢失部分产出，一刀切地 partial→completed 会掩盖部分失败。**partial 的收敛必须带业务判断，不能纯靠规则。**

这条延伸了命题：**不只是 running，任何非干净终态（包括 partial、unknown、timeout 等）都是毒，都要被收敛。** 终态收敛的完整范围是"所有非终态"，不只是 running。

---

## 与其他几条工程哲学的关系

终态收敛不是孤立的，它和 WinMatrix 的其他几条工程哲学互相支撑。

**与"幂等"的关系（第 23 篇）。** 幂等保证"重复执行不影响结果"，终态收敛保证"未完成最终会被推到终态"。两者合起来覆盖了分布式系统的两大风险——重复和丢失。收敛机制本身也需要幂等（比如 sweeper 补写 end 不能重复写，靠 `[spanId, seq]` 唯一约束防重）。**幂等是收敛能安全进行的前提。**

**与"确定性优先"的关系（下一篇第 52 篇）。** 收敛机制本身必须是确定性的——sweeper 扫描、判断阈值、补写终态，每一步都不能依赖 LLM 或模糊判断，必须是规则驱动的。如果收敛逻辑自己不确定，它就成不了别人的兜底。**收敛是确定性优先原则在可靠性领域的延伸。**

**与"fail-closed"的关系（第 28 篇）。** 收敛时"收敛到哪个终态"是个 fail-open/fail-closed 的抉择。LLM span 补写 end 是 fail-closed（标记为失败，不是假装成功）；结果投递重试是 fail-open（暂时 pending，继续试）。**收敛的目标终态要符合安全/可用性的取舍——安全相关 fail-closed，可用性相关可暂时 pending。**

这三条关系说明，终态收敛不是一条孤立的工程建议，而是和幂等、确定性、fail-closed 共同构成分布式系统可靠性的四大支柱。

---

## 给后来者的几条总结

1. **running 是个毒状态。** 它的毒性是静默的、会沿着依赖图全局扩散。任何写进持久存储的 running，都要有对应的收敛机制。
2. **不能指望"开始操作的执行流"来收尾。** 那个执行流可能死。收敛必须由独立的、更可靠的机制承担（看门狗、启动对账、outbox sweeper）。
3. **"未完成"比"失败"更危险。** 失败是已知，系统知道怎么应对；未完成是未知，系统不知道怎么应对。设计要消灭的是未完成，不是失败。
4. **收敛是按类型的，一刀切会出事。** 四种非终态（悬挂 span、孤儿 run、未投递结果、乱序回调）四种收敛方式，各自语义不同。
5. **收敛的依据是超时阈值。** 太短误杀长任务，太长让毒扩散。WinMatrix 取 max(超时, 30 分钟) 或 2x 硬超时。
6. **partial 也要收敛。** 任何非干净终态都是毒。partial 的收敛带业务判断，不能纯规则——一刀切 partial→failed 会丢产出，partial→completed 会掩盖失败。
7. **终态一旦写入要被保护。** CodingTask 的 attemptNo 版本门防迟到回调推翻终态。收敛不只是"推向终态"，也是"保护已收敛的终态"。
8. **收敛机制本身必须确定性和幂等。** sweeper 不能用 LLM 判断、补写不能重复。不确定的收敛机制成不了别人的兜底。
9. **这是一条世界观，不是一条技巧。** 信不信"未完成是危险的"，决定了你设计的系统有没有免疫系统。它没法靠"加个定时任务"获得。

最后说一句：终态收敛的机制平时是隐形的。系统正常运行时，你看不到 sweeper 在补写 end、看不到启动对账在清孤儿——它们默默工作，让系统保持干净。但它们一旦缺失，系统就会像没有免疫系统的身体——每运行一次都在累积病灶，直到某天全面爆发。**这些机制不是"增强"，是"必须"。**

---

> **下一篇**：[《确定性优先：能用规则就别用 LLM》](./52-determinism-first.md)——终态收敛要求收敛机制本身是确定性的。下一篇把这条线拉满：WinMatrix 的决策引擎、FusionRouter、闲聊守卫、缓存，背后都是同一个原则——确定性优先，LLM 是兜底而不是默认。
