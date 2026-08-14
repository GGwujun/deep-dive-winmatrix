# 让多个 AI 员工协作：协作编排与"会干活"的团队

> 这是《WinMatrix 开发经验文集》第 7 篇。单 Agent 不难，难的是让多个 Agent 像一个团队一样协作——谁主导、谁配合、消息怎么派发、结果怎么汇总、卡住了怎么办。这篇讲 WinMatrix 的多 Agent 协作与流程编排。

如果你只做到上一篇——有了技能、有了角色、有了工具——你已经能做出一个挺能干的单 Agent 了。用户说"写 PRD"，小品就去写。

但真实的企业工作不是这样的。真实的工作是：

> 大福（项目总指挥）启动项目 → 小品（产品）写 PRD → 阿码（架构）评审 PRD 并出技术方案 → 小质（质量）做测试计划 → 阿宁（事项管理）跟踪进度。

这是一条**多人（多 Agent）接力**的工作流。每一步由不同角色负责，上一步的产出是下一步的输入，中间可能卡住、可能要催、可能要返工。

让多个 Agent 把这种工作流跑起来，比单 Agent 难一个数量级。这篇文章讲我们怎么做。

---

## 第一个问题：消息该派给谁——会话的三层存储

多 Agent 协作的前提，是有一个可靠的"会话"作为承载。一个会话里可能有大福、小品、阿码多个角色的发言，有工具调用记录、有思考过程。这些怎么存？

我们的会话是**三层存储分工**：

```
conversation_histories (读模型:列表/详情/owner校验)  --读--> UI
session_transcript    (LLM 上下文真源,含 tool/thinking trace) --喂给--> LLM
conversation_meta     (权威元数据:置顶/归档/sessionMode)
```

- **conversation_histories**：读模型。负责列表、详情、owner 校验。UI 展示主要读它。
- **session_transcript**：LLM 上下文的真源。包含完整的 `tool_name`/`tool_result`/`tool_success`/`thinking`。**给 LLM 喂上下文，从这里取。**
- **conversation_meta**：权威元数据。置顶、归档、`sessionMode`、`interactiveDigitalEmployeeId`。

**为什么要分三层？** 因为读、写、喂 LLM 是三种不同的访问模式，用一张表兼顾三者，要么读慢、要么写冲突、要么 LLM 上下文不完整。

特别值得注意的是：**LLM 上下文的真源是 session_transcript，不是 conversation_histories。** 后者只是给人看的读模型。这个区分避免了"给人看的"和"给模型看的"互相污染——你可以为展示做裁剪、聚合，但不影响喂给 LLM 的完整上下文。

---

## 第二个问题：员工之间怎么传递工作——role_inbox 持久收件箱

大福要把一个任务交给小品，怎么"交"？最直觉的做法：大福直接调小品的某个方法。

错。这是**同步耦合**——大福得等小品处理完才能继续，小品如果忙、或者挂了，大福就被拖死。

企业里的工作派发不是这样的。企业用的是**收件箱**（inbox）——大福把任务投到小品的收件箱，小品有空就处理，处理完通知大福。两者解耦。

WinMatrix 用的是 **role_inbox——持久化的角色收件箱**：

```prisma
// prisma/schema.prisma（role_inbox 模型）
model role_inbox {
  event_id
  role_id                    ← 投给哪个角色
  digital_employee_id
  conversation_id
  event_type
  payload
  idempotency_key            ← 幂等键
  status
  claim_owner                ← 租约持有者
  claim_expires_at           ← 租约过期时间
  retry_count                ← 重试次数
  max_retries
  turn_id                    ← 关联的轮次
  message_id                 ← 关联的消息
}
```

这个收件箱的字段，每一个都对应一个分布式系统的硬需求：

### 幂等（idempotency_key）

同一个任务可能被投递多次（网络重试、worker 重启）。`idempotency_key` 保证同一个任务只被处理一次——重复投递会命中已有记录，返回 `deduplicated=true` 而不是再处理一遍。

### 租约抢占（claim_owner / claim_expires_at）

多个 worker 实例可能同时拉收件箱。怎么防止同一个任务被两个 worker 同时处理？**租约**——worker 抢到任务时写入自己的 `claim_owner` 和一个过期时间。在租约期内，别人不能抢。租约过期了（worker 挂了），才能被别人重新抢。

### 重试（retry_count / max_retries）

任务可能失败。`retry_count` 记录已重试次数，`max_retries` 是上限。超过上限的任务进死信，不再无限重试。

### 关联（turn_id / message_id）

每个任务关联到具体的轮次和消息。事后能追溯"这个任务是在哪个对话的哪一轮产生的"。

### 入队策略：PG 先写，BullMQ 投递

```typescript
// business/domain/agentExecution/RoleInboxEnqueueService.ts
/**
 * Enqueue durable role inbox events: PG first, then BullMQ (OpenSpec §3.1).
 */
```

入队的策略是**先写 PG（持久化），再投 BullMQ（通知）**。为什么这个顺序？因为如果先投 BullMQ 再写 PG，万一写 PG 失败了，BullMQ 里的任务就找不到对应的持久化记录——成了"幽灵任务"。**先持久化，再通知，保证通知有据可查。**

**这是分布式消息系统的基本纪律：持久化优先于通知。** 通知可以丢（重投就行），持久化不能丢。

---

## 第三个问题：流程怎么编排——template / version / run 三层

员工接力干活，背后往往是一个**预定义的流程**：第一步谁做、第二步谁做、每步的输入输出是什么。

WinMatrix 的流程是三层模型：

```
flow_template (草稿/root) --发布--> flow_template_version (不可变发布版,带 checksum)
flow_template_version --实例化--> flow_run (一次具体执行)
```

- **flow_template**：流程的草稿，可以编辑。
- **flow_template_version**：发布后的不可变版本，带 `checksum`。**一旦发布就不能改**——要改就发新版本。这保证了"同一个流程版本的行为是确定的、可复现的"。
- **flow_run**：一次具体的执行实例，关联到某个 template_version。

为什么要有不可变的 version？因为流程是企业核心逻辑，不能"跑到一半被改了"。带 checksum 的不可变版本，让你能精确复现"三个月前那次流程跑的是哪一版定义"。

### 指令驱动的按需编排

流程怎么跑？最直觉的做法：一个中心调度器，按顺序一步步调。

我们没那么做。我们用的是**指令驱动**——每个流程步骤生成一条"编排指令"，每条指令**拥有一个独立的 flow_run**：

```prisma
// prisma/schema.prisma（flow_orchestration_instruction）
model flow_orchestration_instruction {
  batchId         String    ← 属于哪个批次
  flowRunId       String    ← 独立的 flow_run
  agentRunId      String
  sequenceNo      Int       ← 序号
  status          String    ← pending/claimed/running/completed/failed/skipped/cancelled/timed_out
  idempotencyKey
  claimToken      String    ← 抢占令牌
  claimedAt
  claimExpiresAt
  attempt         Int       ← 尝试次数
}
```

每条指令是独立的、可抢占的、可重试的执行单元。状态机有 8 种状态，覆盖了正常、异常、跳过、超时的所有情况。

### 并发受租约 + claimToken 控制

多条指令可以并发执行，但要有控制：

```typescript
// business/domain/flowOrchestration/FlowInstructionDispatchCoordinator.ts（第 71-107 行）
async dispatchNext(params) {
  while (flowRunIds.length < concurrencyLimit) {
    const active = await this.instructionRepository.countActiveInstructionsByBatchId(batchId);
    if (active >= concurrencyLimit) break;
    const instruction = await this.instructionRepository.claimInstruction({
      batchId, claimedBy,
      leaseUntil: new Date(Date.now() + (this.options.leaseMs ?? 30 * 60 * 1000)),  // 默认 30 分钟租约
    });
    if (!instruction.data) break;
    const dispatched = await this.dispatchInstruction(instruction.data, {...});
    if (dispatched.success === false) {
      failedCount++;
      await this.instructionRepository.updateInstructionStatus(instruction.data.id, 'failed', {
        failureCode: dispatched.error.code, failureMessage: dispatched.error.message,
      });
      continue;   // 失败立即标记，继续下一条，不阻塞整个批次
    }
    flowRunIds.push(dispatched.data);
  }
}
```

几个设计要点：

- **`countActiveInstructionsByBatchId` 卡并发上限**：不能无限并发。
- **`claimInstruction` 设 30 分钟租约**：抢到的指令有 30 分钟处理权，过期可被重新抢。
- **失败立即标记 + continue**：某条指令失败，不阻塞整个批次，标记失败继续下一条。**局部失败不影响整体推进。**

### provides/consumes 在编排里串联

还记得上一篇讲的技能契约吗？在流程编排里，它发挥了关键作用——每一步的 consumes/consumes 被编译成显式的数据流：

```typescript
// FlowTemplateContractCompiler.ts（第 26-51 行）
export interface FlowTemplateContractStepSnapshot {
  stepId: string;
  consumes: Array<{ field; as; type; required; sourceStepId?; sourcePath }>;  ← 从哪步的哪条路径来
  provides: Array<{ key; type; artifact; required }>;
}
```

每步的 consumes 不只声明"我要什么"，还声明"它从哪个步骤（sourceStepId）的哪条路径（sourcePath）来"。这让数据流是**显式的、可追溯的**，而不是运行时隐式匹配。

而且 `FlowSkillContractValidationService` 强校验——编排里某步 consumes 了一个 key，但没有任何前置步骤 provides 它，直接报 `missing_skill_provides`。**编译期发现的数据流断裂，绝不留到运行期。**

---

## 第四个问题：卡住了怎么办——协作催促

多 Agent 协作有个常见痛点：**阻塞**。大福把任务交给小品，但小品一直没处理（可能忙、可能忘了、可能挂了），整个流程卡在大福→小品这一步。

怎么处理？人肉去催吗？我们做的是**自动的协作催促**——`CollaborationFollowup`：

```prisma
// prisma/schema.prisma（CollaborationFollowup）
model CollaborationFollowup {
  conversationId
  blockedByRoleId        ← 被谁阻塞
  blockedByEmployeeId
  followUpMessage        ← 催促消息
  reason
  delayMinutes           ← 延迟多久后催
  bullJobId              ← BullMQ 任务
  status
  scheduledAt            ← 计划催促时间
  triggeredAt            ← 实际触发时间
  completedAt
}
```

这是一条**延时任务**。当系统检测到某步被某角色阻塞，就创建一条催促任务，带 `delayMinutes`（比如延迟 30 分钟）。30 分钟后如果还没处理，自动给那个角色发一条催促消息。

它带完整的生命周期时间戳（scheduledAt / triggeredAt / completedAt），让你能追踪"这个催促计划了吗、触发了吗、解决了吗"。

**这是多 Agent 协作系统必备的自愈机制——不能指望每条消息都被及时处理，要有自动的阻塞检测和催促。** 否则一个被遗忘的任务能让整个流程永久卡死。

---

## 第五个问题：流程的产出怎么存——资源与文档解耦

流程会产生很多产出——文档、图表、报告。这些怎么存？

最朴素的做法：直接塞进 flow_run 的某个字段。错——产出可能很大、可能很多、可能要单独访问。

我们的做法是**资源元数据与文档字节解耦**：

```prisma
// prisma/schema.prisma（flow_resource）
// Flow resource inventory metadata. File bytes stay in project document/object storage
model flow_resource {
  flowRelativePath     ← 流程内的相对路径
  storageRef           ← 实际存储引用
  checksum
  size
}
```

`flow_resource` 只存**元数据**（路径、引用、校验和、大小），文件字节留在项目文档/对象存储里。**元数据和内容分开，是处理大产出的标准做法**——元数据便于检索和列表，内容按需读取。

而且路径有安全校验：

```typescript
// FlowResourcePathService.ts
// 以 flow-resources/<flowSlug>/<relativePath> 规范化路径
// 并禁止 .. 越狱
```

`..` 路径穿越是文件操作的经典漏洞。流程资源路径必须规范化、禁止 `..`，防止 Agent（或恶意输入）读到流程目录之外的文件。**安全是治理的一部分。**

---

## 第六个问题：流程怎么触发——五种触发源

流程不是只能手动启动。WinMatrix 支持五种触发源：

```typescript
// FlowOrchestrationStartService.ts
import { AgentToolTriggerProvider } from './trigger/AgentToolTriggerProvider.js';
import { ApiTriggerProvider } from './trigger/ApiTriggerProvider.js';
import { ManualTriggerProvider } from './trigger/ManualTriggerProvider.js';
import { ScheduledTaskTriggerProvider } from './trigger/ScheduledTaskTriggerProvider.js';
import { WebhookTriggerProvider } from './trigger/WebhookTriggerProvider.js';
```

- **AgentTool**：某个 Agent 在对话中调用工具触发的流程
- **Api**：外部 API 调用触发
- **Manual**：用户手动启动
- **ScheduledTask**：定时任务触发（比如每周一自动跑周报流程）
- **Webhook**：外部系统 webhook 触发

这五种触发源是**可插拔**的（用 `triggerProviders: Map` 注册），统一经 `FlowStartEnvelopeSchema` 校验后进入批次创建。

**为什么要支持这么多触发源？** 因为企业流程的启动场景是多样的——有的流程是人点的、有的是定时跑的、有的是别的系统调过来的。一个只会手动启动的流程系统，覆盖不了真实企业需求。

---

## 多 Agent 协作的全景

把上面六点串起来，一次完整的多 Agent 协作长这样：

```
用户 --触发"新产品立项"流程--> 流程编排
流程编排: 按模板创建 flow_run + 指令批次
流程编排 --指令1:大福启动项目(claim抢占)--> role_inbox
role_inbox --派发--> 大福
大福 --产出 prd_request,投递给小品--> role_inbox
role_inbox --派发(PG先写+BullMQ)--> 小品

  [小品忙,未及时处理]
  流程编排: 检测阻塞,创建 CollaborationFollowup
  流程编排 --延时催促--> 小品

小品 --产出 prd_document(provides)--> role_inbox
role_inbox --派发:阿码评审(consumes prd_document)--> 阿码
阿码 --产出 tech_solution--> role_inbox
流程编排: 所有指令 completed, flow_run 收敛终态
流程编排 --流程完成通知--> 用户
```

---

## 给后来者的几条总结

1. **会话要三层存储分工**。读模型、LLM 上下文真源、权威元数据分开。给 LLM 喂的上下文（session_transcript）要完整，不能和给人看的读模型混用。
2. **员工间派发用持久收件箱**，不要同步调用。role_inbox 带幂等、租约、重试、关联四件套。
3. **入队策略：持久化优先于通知**。先写 PG 再投 BullMQ，通知可丢，持久化不能丢。
4. **流程是 template→version→run 三层**。version 不可变带 checksum，保证流程行为可复现。
5. **指令驱动的按需编排**。每条指令是独立、可抢占、可重试的单元；局部失败不阻塞整体。
6. **并发靠租约 + claimToken 控制**。租约防重复处理，countActive 卡并发，失败立即标记继续。
7. **provides/consumes 在编排里是显式数据流**。sourceStepId/sourcePath 让数据可追溯，FlowSkillContractValidationService 编译期强校验。
8. **阻塞要自动催促**。CollaborationFollowup 延时任务，检测到阻塞自动催，不能让流程永久卡死。
9. **资源元数据与文档字节解耦**。flow_resource 只存元数据，字节留对象存储，路径禁止 `..` 越狱。
10. **支持多种触发源**。AgentTool/Api/Manual/ScheduledTask/Webhook 可插拔，覆盖真实企业的流程启动场景。

让单个 AI 会干活，是工具和技能的事；让多个 AI 像团队一样干活，是协作和编排的事。后者的复杂度远高于前者，但也是"AI 员工"区别于"AI 助手"的关键——**一个会协作的 AI 团队，能完成单 Agent 永远完成不了的多步骤、跨角色、长周期工作。**

如果你只做单 Agent，恭喜你省了一大半复杂度。但只要你打算做"AI 团队"，上面这套机制——会话分层、持久收件箱、不可变流程、指令驱动编排、自动催促、资源解耦、多触发源——一条都省不了。

---

> **下一篇**：[《企业级 AI 的可观测性：ExecutionSpan 如何取代散落的日志》](./08-observability.md)——协作跑起来了，怎么知道它跑对了？可观测性是 Agent 团队的"指挥仪表盘"。
