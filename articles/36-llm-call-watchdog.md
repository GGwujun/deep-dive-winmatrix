# 悬挂的 LLM 调用：start 记了，end 永远不来

> 这是《WinMatrix 开发经验文集》第 36 篇。讲一个在 Agent 平台里特别阴险的故障：一次 LLM 调用记了"开始"，却永远等不到"结束"。它不报错、不崩溃，只是悄悄留下一堆永远挂着的执行——直到把系统的状态机搅成一锅粥。这一篇讲我们怎么发现它、根因是什么、最后怎么用一只"看门狗"收口。

做 Agent 平台的人都知道，一次 LLM 调用是整个系统里最"贵"也最"脆"的一环。贵在 token 和延迟，脆在它依赖外部网络、依赖供应商稳定性、依赖你自己的进程别在这中间崩。我们对每一次 LLM 调用都做了完整的遥测——调用前记一个 `llm_call_start`，调用后记一个 `llm_call_end`，失败记 `llm_call_error`。听起来很完整。

但生产环境会教你做人。某天我们看可观测大盘，发现一个诡异的数字：**`llm_call_start` 的累计计数，比 `llm_call_end` + `llm_call_error` 加起来多了几千。** 这几千次调用，既没成功也没失败，就这么凭空"消失"了。

更严重的是，这些消失的调用背后，是一堆永远停在 `running` 状态的 `agent_run`、`scheduled_task_run`——它们在前端表现为"任务一直在转圈，永远不出结果，也永远停不下来"。

这一篇就讲这个坑：悬挂的 LLM 调用。

---

## 现象：几千个"只有开头没有结尾"的调用

事情是从定时任务的对账逻辑里暴露出来的。

我们有一个每 30 分钟跑一次的 `system-scheduled-run-reconcile` 任务，专门收敛那些"running 时间过长"的执行。某次它的告警突然变多，打开一看，几百条 `agent_run` 的 `status='running'`，但它们的 `started_at` 是三天前的——一个正常的 Agent run 顶多几分钟，跑三天还在 running，显然是假的。

顺着 `agent_run` 往上查，发现它们都关联到某个 LLM 调用 span。这个 span 的状态是 `pending`——也就是说，`llm_call_start` 记了，但 `llm_call_end` 从来没写过。

```
正常的 LLM 调用:
  llm_call_start ──→ llm_call_end      (span status: succeeded)
  llm_call_start ──→ llm_call_error    (span status: failed)

悬挂的 LLM 调用:
  llm_call_start ──→ ???                (span status: pending，永远)
```

这些"???"才是真正的坑。它不是报错（报错我们就有 `llm_call_error` 了），而是**调用发出去之后，进程在等响应的过程中被打断了，没有人在 finally 里补一个 end**。

---

## 根因：外部调用 + 进程中断 = 悬挂

排查下来，`llm_call_end` 丢失的场景集中在三类：

1. **LLM 供应商超时返回前，Pod 被 OOM kill 或滚动更新杀掉。** 进程没了，正在 await 的 LLM 调用直接中断，finally 块没机会执行，`llm_call_end` 没写。
2. **网络层长时间挂起（不是报错，是不返回）。** 某些供应商在限流时会让连接长时间挂着不响应，既不 close 也不 reset。我们的超时配置是分钟级的，但 `agent_run` 的超时更长，于是调用挂在那，run 也挂在那。
3. ** BullMQ worker 在处理一个 LLM 重任务时，被 BullMQ 的 active timeout 移出 active 列表、重投到 delayed。** 老的执行路径被中断，但它的 `llm_call_start` 已经写进 span 了。

这三类有一个共同点：**它们都不是"LLM 调用本身报错"，而是"承载这次调用的进程被外部力量打断了"。** 你的 try/catch / finally 再严密，也拦不住 SIGKILL 和网络黑洞。

这就是问题的本质：我们为一个**本质上不可靠的外部调用**建了完整的 start/end 遥测契约，却没有考虑"记了 start 的那个进程，可能根本活不到记 end 的时候"。

```
  传统心智模型:                    真实世界:
  start ──→ end                    start ──→ end     ✓
  start ──→ error                  start ──→ error   ✓
                                    start ──→ 💥 SIGKILL   ✗ (进程没了)
                                    start ──→ 🕳️ 网络黑洞   ✗ (永远等不到)
```

一旦意识到这一点，修复思路就清晰了：**不能指望"记 start 的同一个执行流来记 end"，必须有一个独立于业务执行流的"看门狗"周期性扫描悬挂调用，代为补写 end，并级联收敛所有依赖它的状态。**

---

## 修复：一只每 10 分钟巡视一次的看门狗

我们建了一个专门的系统任务 `system-llm-call-watchdog-sweeper`，它是一个**interval 类型**的定时任务，每 10 分钟扫一次所有"开始时间超过阈值却还没结束"的 LLM 调用，代它们补写 end、级联 finalize 上层的 run、并补发失败回执。

### 第一层：扫描阈值的选取

```typescript
// infrastructure/scheduled/llmCallWatchdogSweeper.ts（第 1-53 行）
export const LLM_CALL_WATCHDOG_SWEEPER_TASK_NAME = 'system-llm-call-watchdog-sweeper';

function resolveSweepThresholdMs(): number {
  const hardMs = getConfig().llmCallHardTimeoutMs;
  return hardMs > 0 ? hardMs * 2 : 360_000;  // 2x hard timeout 或 6 分钟
}
```

阈值是"硬超时的 2 倍"。为什么是 2 倍而不是 1 倍？因为正常情况下，LLM 硬超时触发时，业务代码会捕获超时异常并自己写一个 `llm_call_error`——这是正常路径，不该被看门狗当成悬挂。看门狗只接管那些**连异常捕获都没机会执行**的情况。留一倍冗余，确保正常超时的调用不会被误判为悬挂。

默认 6 分钟（当没配硬超时时），配合 10 分钟的扫描周期，意味着一个悬挂调用最多存活约 16 分钟就会被收敛。这个延迟在 Agent 场景下是可接受的——它远短于 `agent_run` 的超时窗口，不会让用户盯着一个永远转圈的任务。

### 第二层：补写 end + 级联 finalize

看门狗扫到悬挂调用后，不只是简单地把 span 标记成 failed 就完事——它要**级联收敛所有依赖这次 LLM 调用的上层状态**。

一次悬挂的 LLM 调用，它身上挂着的可不止是一个 span。还有：

- **`agent_run`**：这次 LLM 调用属于某个 Turn，Turn 属于某个 agent_run。run 还停在 `running`。
- **`scheduled_task_run`**：如果是定时任务触发的，`scheduled_task_run` 也停在 `running`，它的结果投递 outbox 也卡住没发。
- **对话前端**：用户的界面还在转圈，等一个永远不会来的 terminal 事件。

所以看门狗的收敛是一个**自底向上的级联**：

```
悬挂的 llm_call span（补写 end，status=failed，failure_reason_code=watchdog_swept）
    ↑ 属于
agent_run（finalize 为 failed/partial，释放会话锁）
    ↑ 属于
scheduled_task_run（标记失败，触发结果投递）
    ↓ 关联
对话前端（补发 terminal 事件，停止转圈）
```

每一层都要收敛到终态。少收敛一层，就会留下一个"僵尸状态"，迟早被下一个对账任务翻出来，或者让用户永远等下去。

### 第三层：补发失败回执（DanglingFailureReceiptPusher）

级联 finalize 解决了状态机的收敛，但还有一个遗留问题：**用户那边怎么办？**

正常失败的执行，业务代码在 catch 块里会通过 Emitter 发布一个 `buildFailedTerminal` 事件，前端收到后停止转圈、显示错误。但悬挂的调用是被看门狗**在另一个进程、另一个时间点**收敛的，那个会话的 Emitter 早就没了，原来的 WebSocket 连接也早断了。

所以我们专门做了一个 `DanglingFailureReceiptPusher`：在级联 finalize 之后，针对那些"用户还在等结果"的会话，**补发一条失败回执**。这条回执走的是 push registry（`webConversationPushRegistry`），即使用户的连接已经重连过、换了 socket，只要他还在这个会话里，就能收到"你之前那个任务失败了"的通知。

```
正常失败路径:
  业务 catch ──→ Emitter.emitRunError ──→ WebSocketChannel ──→ 前端

悬挂失败路径（看门狗代偿）:
  Watchdog 扫描 ──→ 补写 end ──→ 级联 finalize
                                   └─→ DanglingFailureReceiptPusher
                                          └─→ push registry ──→ 前端（可能已重连）
```

这里有个细节：补发回执必须走 push registry 而不是原来的 Emitter。因为原来的 Emitter 绑定的是一次已经死掉的执行流，它的订阅者早就清空了。push registry 是按 `conversationId + ownerId` 注册的，它跟踪的是"谁在关注这个会话"，而不是"谁在关注这次执行"。这就是我们在第 4 篇流式输出里讲的"会话级 push 能力"的价值——它让会话的异步通知摆脱了对某次具体执行生命周期的依赖。

---

## 教训

这个坑让我们重新认识了几件事。

**第一，任何"start/end 配对"的遥测契约，都必须假设 end 可能永远不会被业务代码写。** start 是业务代码写的，但业务代码可能在 end 之前就被杀掉。一个独立的、周期性的、代偿性的扫描器，不是可选的增强，而是契约成立的必要条件。没有它，你的 start/end 配对就是一个会漏水的桶。

**第二，外部调用是悬挂的重灾区。** 内部的计算要么完成要么抛错，很少会"挂着"。但外部调用——LLM、HTTP API、数据库——都可能因为网络黑洞、供应商故障、进程被杀而进入"永不返回"状态。**只要你的系统里有对外的 I/O 调用，就该有一个悬挂检测机制。** 阈值取正常超时的 2 倍，周期取业务可容忍的最大延迟。

**第三，状态收敛是分层的，少一层都不行。** 一个悬挂的 LLM 调用，它污染的不只是自己那个 span，还有 agent_run、scheduled_task_run、对话前端。修复时必须自底向上把每一层都收敛到终态。我们见过有人只修了 span（"遥测干净了"），结果 agent_run 还是 running，定时任务的对账照样告警。**收敛不是把一个标记改对，是把一条因果链上所有受影响的状态都改对。**

**第四，补发回执要解耦于具体执行的生命周期。** 悬挂调用被收敛时，产生它的那个执行流早就没了。想给用户通知，就必须走一个比"单次执行"更持久的通道——会话级的 push registry。这是流式系统设计时就该预留的能力：**通知通道的生命周期要长于任何一次具体执行。**

最后说一句关于"2 倍冗余"的判断。看门狗的扫描阈值是硬超时的 2 倍，这个数字是反复调出来的——太小会误杀正常超时的调用，太大会让用户等太久。**任何代偿机制都有一条"正常路径"和"代偿路径"的边界，这条边界必须留够冗余，让正常路径先有机会自己处理。** 代偿是兜底，不是抢跑。抢跑的代偿，会制造比它解决的更多的问题。

---

> **下一篇**：[《孤儿任务回收：进程崩了，running 状态谁来收敛》](./37-orphan-task-reconcile.md)——悬挂的 LLM 调用只是一类悬挂，更普遍的问题是"进程崩了留下一堆 running 任务"。启动时怎么把脏状态收敛干净。
