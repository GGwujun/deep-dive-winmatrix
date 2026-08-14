# 孤儿任务回收：进程崩了，running 状态谁来收敛

> 这是《WinMatrix 开发经验文集》第 37 篇。上一篇讲"悬挂的 LLM 调用"——start 记了 end 没来。这一篇讲一个更普遍的问题：进程直接崩了，连 start 都来不及收尾，留下一堆停在 `running` 状态的任务没人管。这些"孤儿"会在重启后继续占着会话锁、卡住定时任务、污染监控数字。讲我们怎么在启动时用一把 advisory lock 串行化三路对账，把脏状态收敛干净。

上一篇的坑，根因是"记了 start 的执行流活不到记 end"。但还有一种更暴力的情况：**进程直接没了**。

Pod 被 OOM kill、节点宕机、滚动更新时旧 Pod 被直接 SIGTERM 后超时 SIGKILL、甚至就是某个 worker 进程 panic 退出——这些情况下，进程没机会做任何清理。它手里正在处理的那些任务，状态都停在 `running`。进程一死，这些 `running` 任务就成了"孤儿"：没有 owner 在推进它们，但数据库里它们活得好好的，看起来还在跑。

重启之后，这些孤儿就开始捣乱了。

---

## 现象：重启后，系统里凭空多出几百个"正在运行"的任务

这个问题是在一次滚动更新后被发现的。

发版后，监控显示系统的并发 LLM 调用数异常高——明明刚重启、还没什么用户流量，怎么会有这么多活跃调用？查 `agent_run` 表，几百条 `status='running'`，`started_at` 都是发版前的。它们不是真在跑，是上一轮进程死的时候留下的残骸。

更麻烦的是，这些孤儿还在占着资源：

- **会话锁（conversationRunLocks）**：某些孤儿 run 对应的会话，被标记为"有活跃执行"。新进来的用户消息被告知"会话正忙"，进不去。
- **BullMQ active job**：BullMQ 自己的 active 列表里也有残留（虽然 BullMQ 有 stalled job 检测，但它的超时窗口比较长，且部分 job 的状态影响是写在业务库里的，BullMQ 管不到）。
- **定时任务并发配额**：`scheduled_task_run` 的 running 计数偏高，新的定时任务因为"并发已满"被 moveToDelayed 延迟。

这些都是**假象**——没有任何进程在真正推进这些任务，但系统以为它们在跑。如果不清理，它们会一直占着锁、占着配额、搅乱所有基于"running 计数"的判断。

---

## 根因：内存状态随进程消失，持久状态留下了

理解这个坑，要分清两类状态。

```
            进程死掉后...
内存状态:    conversationRunLocks (Map)    ──→ 没了（进程内存随进程消失）
            AbortController (Map)         ──→ 没了
            AgentExecutionTicket          ──→ 没了

持久状态:    agent_run.status='running'   ──→ 还在！（在 PG 里）
            scheduled_task_run.running    ──→ 还在！
            coding_task.status='running'  ──→ 还在！
```

问题出在两类状态的**生命周期不一致**。内存状态（锁、ticket、abort controller）是进程级的，进程一死就没了。持久状态（数据库里的 running 标记）是跨进程的，进程死了它还在。

正常运行时，这两类状态是同步流转的：任务开始时写持久状态 + 设内存状态，结束时反过来。但进程崩溃时，只完成了"开始"那一半——持久状态写了"running"，内存状态也设了——然后进程死了，内存状态蒸发，持久状态永远停在 running。

更麻烦的是，这不是单一一张表的问题。WinMatrix 里"running 态"散落在至少三个地方：

| 持久化位置 | 含义 | 谁在写 running | 正常谁来写终态 |
|---|---|---|---|
| `agent_run` (pipeline/scheduled run) | 一次完整多步执行 | TurnRunner.run 开始时 | Turn 结束发 terminal 时 |
| `scheduled_task_run` | 一次定时任务执行 | scheduledTaskWorker 接到 job 时 | 执行完投递结果时 |
| CodingTask（编码任务） | 一次沙箱编码任务 | 创建任务进 running 时 | 工作站回调 callbackToken 鉴权后 |

三张表，三种 owner，三种终态写入路径。进程崩了，三条路径都没跑完，三个地方都留下 running 残骸。

---

## 修复：启动时 advisory lock 串行化三路 reconcile

修复思路是：**既然崩溃发生在运行期，那就在启动期把脏状态收回来。** 进程每次启动，都做一次"全量对账"，把所有"看起来在 running 但实际没有 owner"的任务，强制收敛到终态。

但这里有两个难点：第一，要对账三张不同语义的表；第二，多个进程同时启动（比如集群滚动更新）时，不能让它们并发对账、重复收敛。我们用一把 **PG advisory lock + 三路 reconcile 函数** 解决了这两个难点。

### reconcileStaleRunsOnBootstrap：三路对账的总入口

```typescript
// business/application/services/ScheduledTaskService.ts（第 2244-2298 行）
const thresholdSeconds = Math.max(
  Math.floor(config.scheduled.agentJobTimeoutMs / 1000),
  30 * 60,  // 最少 30 分钟
);
const locked = await withReconcileLock(config.reconcileAdvisoryLockKey, async () => {
  // 第一路：收敛 workflow run（显式流程编排的运行实例）
  const workflowRows = await convergeRunningScheduledWorkflowRuns(500);
  // 第二路：收敛 scheduled_task_run（定时任务执行）
  const scheduledRows = await reconcileStaleRunning(thresholdSeconds);
  // 第三路：收敛 agent_run（pipeline/scheduled run 根聚合）
  const pipelineRows = await getAgentRunRepository().reconcileStaleRunning(thresholdSeconds);
  // 第四路：处理 partial 状态的 scheduled run
  const candidates = await getAgentRunRepository().listPartialScheduledRunCandidates(500);
  // ...对 candidates 逐个收敛
});
```

逐行看这里的几个关键设计。

**阈值 `thresholdSeconds` = max(agentJobTimeoutMs, 30 分钟)。** 这是判断"一个 running 任务是否已经死了"的依据。如果一个 running 任务的 `started_at` 距今超过这个阈值，就认为它"不可能还在正常跑"，强制收敛。取 max 是因为：即使你配的 agentJobTimeout 很短，也要至少等 30 分钟——给那些真正在跑的长任务留够窗口，避免误杀。

**`withReconcileLock` —— advisory lock 串行化。** 这是解决"多进程同时启动"的关键。

### advisory lock：为什么启动期对账必须串行

```typescript
// withReconcileLock 内部
const locked = await withReconcileLock(config.reconcileAdvisoryLockKey, async () => {
  // ... 三路 reconcile
});
```

`withReconcileLock` 包的是一把 **PG advisory lock**（基于 `pg_try_advisory_lock`）。advisory lock 是 PG 级别的、跨会话的互斥锁——同一把 key，在整个 PG 实例范围内，同一时刻只有一个会话能持有。

为什么启动对账必须串行？考虑集群滚动更新的场景：Pod A 和 Pod B 在 30 秒内先后重启。如果两个 Pod 的启动对账并发跑：

```
Pod A 启动: 扫到 100 条 running，开始收敛
Pod B 启动: 也扫到同样 100 条（A 还没改完），也开始收敛
            ↓
            重复收敛、状态冲突、甚至互相覆盖
```

advisory lock 保证了：**不管多少个 Pod 同时启动，同一时刻只有一个在执行 reconcile。** 后到的 Pod 拿不到锁，要么等、要么跳过（因为前一个 Pod 已经把脏状态清干净了，它启动后看到的已经是干净状态）。

> 注意一个容易混淆的点：WinMatrix 的运行时分布式锁走的是 Redis（`SET NX EX` + Lua），PG advisory lock 已经在日常运行期废弃了。**唯独启动期的 reconcile 还在用 advisory lock**——因为这里需要的是"跨进程、与 Redis 状态无关"的强一致互斥，而 Redis 锁在重启边界本身可能不稳定（Redis 里的锁可能还在，但持有它的进程已经死了）。PG advisory lock 随 PG 会话生灭，进程退出即释放，天然适合"启动期一次性"的串行化。

### 三路 reconcile，各管各的语义

对账不是简单地把所有 running 改成 failed——不同类型的 run，终态语义不同，收敛方式也不同。

**第一路 `convergeRunningScheduledWorkflowRuns`：流程编排 run。** 显式流程编排（flow_run）的运行实例，如果卡在 running，要根据它的 step 状态推导出"最合理的终态"。比如某个 step 已经 failed，那整个 run 应该是 failed；如果所有 step 都 completed 但 run 没收尾，那应该是 completed。这一路做的是**语义推导**，不是一刀切。

**第二路 `reconcileStaleRunning`（scheduled_task_run）：定时任务 run。** 超过阈值的 running 直接标记为 failed，并触发它的结果投递（ScheduledTaskRun 自带 outbox 字段，收敛后要把"失败结果"投递出去）。

**第三路 `getAgentRunRepository().reconcileStaleRunning`（agent_run）：Agent 执行根聚合。** 同样按阈值收敛，但 agent_run 的收敛要级联它的子表（agent_run_step、agent_run_decision、agent_run_state），把整棵聚合树都收到终态。

**第四路 `listPartialScheduledRunCandidates`：partial 状态的精修。** 有些 run 不是 running 而是 partial（执行到一半，部分成功部分失败）。这类状态在某些场景下需要"继续推进"而不是"标记失败"，所以单独拎出来逐个判断。

### failTimedOutRunningTasks：编码任务的独立回收

上面三路是 `reconcileStaleRunsOnBootstrap` 管的，针对的是 pipeline/scheduled run。**编码任务（CodingTask）有它自己独立的回收路径**——`failTimedOutRunningTasks`，由 `system-coding-task-timeout-sweep` 这个定时任务（每 5 分钟）驱动，也在启动时跑一次。

为什么编码任务要单独一套？因为它的终态写入路径特殊：正常情况下，编码任务的终态是由**工作站回调**写入的（`callbackTokenHash` 鉴权）。进程崩了，回调还在来的路上，如果你贸然把任务标记成 failed，等回调真来了就会和你的终态打架。所以编码任务的回收要额外处理"回调竞态"——我们用 `attemptNo` 防止迟到旧 attempt 覆盖新状态（第 12 篇讲过），回收时也要遵守同样的 attempt 顺序。

---

## 教训

**第一，"持久状态"和"内存状态"的生命周期必须被显式管理。** 最隐蔽的 bug 来自"以为它们是同步的"。进程崩溃会把这种不同步暴露到极致——内存状态蒸发了，持久状态还在。**任何把 running 写进持久存储的系统，都必须有一个"启动时收敛悬挂 running"的机制。这不是增强，是必须。** 没有 it 的系统，每重启一次就脏一点，脏到一定程度就不可挽回。

**第二，启动期是一个"强一致窗口"，要善用。** 运行期是分布式的、并发的、最终一致的。但启动期不一样——尤其是 advisory lock 保护的这段对账，它是一个**强一致窗口**：此刻只有一个进程在改这些状态，没有并发竞争。适合做那些运行期不敢做（怕竞态）的全量清理。**好的系统设计，会把"需要强一致的操作"攒到启动期批量执行，而不是在运行期硬扛并发。**

**第三，对账是分类型的，一刀切会出事。** 我们的三路 reconcile（workflow run / scheduled run / agent run）加上编码任务的独立回收，每一路的终态语义都不同。workflow run 要做语义推导，scheduled run 要触发结果投递，agent run 要级联子表，编码任务要处理回调竞态。**如果你图省事用一个统一的 `UPDATE ... SET status='failed' WHERE status='running'`，省下的代码量会让你在别的地方付出十倍的代价。** 对账的复杂度，反映的是你业务状态机的复杂度——绕不过去。

**第四，advisory lock 适合"启动期一次性串行"，不适合运行期长期持有。** 我们运行期用 Redis 锁，唯独启动对账用 PG advisory lock，是有道理的。advisory lock 绑定 PG 会话，会话断开自动释放——这正好是"进程退出即释放"的语义，和"启动期"的边界天然匹配。运行期长期持有 advisory lock 反而危险（进程崩了锁还在，得等会话超时）。**锁的选择，要看你想要的释放语义，而不是看"哪个更新潮"。**

**第五，"收敛"不是把一个字段改对，是把因果链上所有关联状态都改对。** 这一点上一篇也强调过，这里再重复——因为太重要了。一个孤儿 agent_run 收敛，牵涉到会话锁释放、定时任务配额归还、scheduled_task_run 的 outbox 投递、前端 terminal 通知。漏掉任何一个，就是一个新 bug。**写 reconcile 代码时，先画一遍"一个 running 任务关联了哪些资源"，确保每一样都归还了。**

---

> **下一篇**：[《Schema 漂移：代码发了但迁移没跑》](./38-schema-drift.md)——代码和数据库 schema 不一致，是另一个经典的"重启后才发现"的坑。怎么在启动时就探测到、并在运行期优雅降级。
