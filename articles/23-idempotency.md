# 幂等设计全景：同一个请求被执行两次，世界还能保持一致吗

> 这是《WinMatrix 开发经验文集》第 23 篇，也是"跨模块横切主题"的第一篇。分布式系统里有一道永恒的难题：网络会重传、用户会双击、进程会崩溃后重启、消息队列会 at-least-once 投递——同一个逻辑请求几乎一定会被执行多次。这篇不写某个模块，而是把 WinMatrix 里散落在 5 个模块的幂等设计拎出来，看清楚它们各自在防什么、用了什么形态、为什么不能统一成一套。

先讲一个每个做过分布式系统的人都遇过的场景。

用户在企微里点了一下"@小品 帮我写个 PRD"。这个点击在网络抖动下可能被前端重发一次；消息进入系统后被 BullMQ 投递，队列是 at-least-once 语义，崩溃重连时会重投；这条消息最终交给小品的角色收件箱，收件箱 worker 抢占处理，抢到一半进程被杀，lease 过期后另一个实例重新抢到。**同一个"@小品 写 PRD"的语义请求，在系统里可能变成 3 到 5 次物理执行。**

如果系统不做幂等，用户会收到 3 份 PRD，而且它们互相覆盖——因为每次执行都覆盖同一个会话状态。这就是为什么幂等不是"锦上添花"，而是"系统能不能活的底线"。

幂等的核心定义只有一句话：**执行一次和执行多次，对外可见的效果完全一致。** 但怎么实现这句话，在不同的场景下长出了完全不同的形态。WinMatrix 里就有 5 种。

---

## 形态一：业务幂等键（role_inbox 的 idempotency_key）

最直观的幂等是"业务幂等键"——业务方生成一个唯一 ID，重复请求带相同的 ID，系统识别后只执行一次。

看 role_inbox 这张表，它是角色消息的持久收件箱：

```
role_inbox（Interactive role durable inbox）
├── event_id            事件 ID
├── role_id             归属角色
├── conversation_id     会话
├── event_type / payload
├── idempotency_key     ← 幂等键
├── status              pending / claimed / running / done / failed
├── claim_owner         ← 抢占者
├── claim_expires_at    ← 租约过期
├── retry_count / max_retries
└── turn_id             ← 轮次关联
```
*源码：`prisma/schema.prisma`（role_inbox 模型，核实报告 ch18-22）*

入队时走的是这个逻辑（`business/domain/agentExecution/RoleInboxEnqueueService.ts:59-90`）：

```ts
async enqueue(input: EnqueueRoleEventInput): Promise<EnqueueRoleEventResult> {
  const turnId = input.turnId?.trim() || `turn_${randomUUID()}`;
  // ... 路由校验、空闲探测、负载规范化 ...
  try {
    record = await roleInboxRepository.insertQueuedEvent({ ...input, payload, turnId });
  } catch (error) {
    if (error instanceof RoleInboxDuplicateEventError) {
      record = error.existing;       // 命中唯一约束 → 复用已存在记录
      deduplicated = true;           // 标记为"去重命中"
    }
  }
}
```

**这里用的是数据库唯一约束做兜底**：`idempotency_key` 上建唯一索引，第二次插入会抛 `RoleInboxDuplicateEventError`，入队服务 catch 之后不报错，而是返回 `deduplicated=true` 并复用已存在的那条记录。调用方拿到的是一个"看起来成功"的结果，但实际上没产生第二条记录。

这种形态的要点：

- **幂等键由业务方生成**（这里是 turnId + 业务上下文算出来的），它有业务含义，不是随机 UUID。
- **去重靠 DB 唯一约束**，不靠应用层先 select 再 insert（那个写法在并发下有竞态）。
- **重复请求被识别后要有合理的"复用"语义**，而不是简单报错——调用方不需要知道这是重复。

业务幂等键适合"消息/事件入队"这种场景：幂等粒度是"一条逻辑消息"，防的是消息重复投递。

---

## 形态二：复合键 + 状态门（CodingTask 的三元组去重）

幂等更复杂的版本是"带状态的复合键"。看编码任务 CodingTask：

```
CodingTask
├── workstationId / triggerTool   (coding_task | sre_skill_task | execute_task)
├── idempotencyKey
├── attemptNo                     ← 第几次尝试
├── callbackTokenHash             ← 回调鉴权
├── taskFingerprint               ← 任务指纹
├── claudeSessionId
└── partial unique index:
    [sessionBindingKey, taskFingerprint, status, updatedAt]
    WHERE claude_session_id IS NOT NULL AND status='completed'
```
*源码：`prisma/schema.prisma:310-386`（核实报告 ch13-17）*

CodingTask 的幂等键不是单字段，而是三元组 `(conversation_id, trigger_tool, idempotency_key)`。它的设计要同时防三种问题：

**问题 1：同一个任务被触发两次。** 三元组复合唯一，第二次插入要么被唯一约束拦下，要么命中已有记录。

**问题 2：迟到的旧 attempt 覆盖新状态。** 这才是真正棘手的。想象这个时序：

```
attempt #1 发出 → 网络超时 → 系统认为失败，启动 attempt #2
attempt #2 完成 → 状态更新为 completed
attempt #1 的迟到的回调终于来了 → 想把状态改回 running
```

如果没有防护，attempt #1 会把已经完成的任务"打回"running。`attemptNo` 字段就是防这个的——回调带 attemptNo，只有当 attemptNo >= 当前记录的 attemptNo 时才接受。**迟到的小号不覆盖大号。**

**问题 3：回调被伪造。** 编码任务在工作站 Pod 里执行，执行完回调系统。回调请求带 `callbackTokenHash`，服务端用 hash 校验是不是当初派发任务时签发的那个 token。**没有正确 token 的回调直接拒绝——防止有人猜一个任务 ID 就来伪造结果。**

注意还有个 partial unique index：

```prisma
// WHERE claude_session_id IS NOT NULL AND status='completed'
```

这个"部分唯一索引"的意思是：在"已完成且绑定了 session"的记录里，`(sessionBindingKey, taskFingerprint)` 唯一。它的作用是**让同一个 session 里完成的同类任务可以被"复用"**——重复请求不是被拒绝，而是找到已存在的那条 completed 记录复用它的结果。这是幂等键的"复用语义"在编码任务场景下的具体应用。

这种形态的要点：**幂等不只是"防重复"，还要防"乱序覆盖"和"伪造"。** 当一个操作跨越多次异步往返（任务派发→Pod 执行→回调），光有幂等键不够，还要带版本号（attemptNo）和鉴权（callbackTokenHash）。

---

## 形态三：claim_token + 租约（flow_instruction 的抢占式幂等）

第三种形态出现在流程编排里。flow_orchestration_instruction 是一条流程指令：

```
flow_orchestration_instruction
├── batchId / sequenceNo          批次与序号
├── flowRunId                     独立 flow_run
├── status                        pending→claimed→running→completed/failed
├── idempotencyKey
├── claimToken                    ← 抢占令牌
├── claimedAt / claimExpiresAt    ← 抢占时间 + 租约
├── claimedBy
└── attempt
```
*源码：`prisma/schema.prisma`（核实报告 ch18-22）*

派发协调器抢指令的逻辑（`FlowInstructionDispatchCoordinator.ts:71-107`）：

```ts
async dispatchNext(params) {
  const batch = await this.instructionRepository.getBatch(params.batchId);
  while (flowRunIds.length < Math.max(1, params.concurrencyLimit)) {
    const active = await this.instructionRepository
      .countActiveInstructionsByBatchId(params.batchId);
    if (active.data >= Math.max(1, params.concurrencyLimit)) break;
    const instruction = await this.instructionRepository.claimInstruction({
      batchId: params.batchId,
      claimedBy: params.userId,
      leaseUntil: new Date(Date.now() + (this.options.leaseMs ?? 30 * 60 * 1000)),
    });
    if (!instruction.data) break;
    // ... 派发执行 ...
  }
}
```

这里的幂等不是"防同一请求重复执行"，而是**"防同一个任务被多个 worker 同时抢走"**。它是并发控制维度的幂等。机制是：

1. **claimInstruction 是原子的**：一条指令只能被一个 worker 抢到（靠 DB 行锁 + 条件更新实现）。
2. **抢到后拿 claimToken**：后续所有对这个指令的状态变更都要带这个 token，证明"我才是当初抢到它的那个"。
3. **租约 leaseUntil 默认 30 分钟**：worker 没在租约内完成，租约过期，别的 worker 可以重新抢。

**claimToken 和 CodingTask 的 callbackTokenHash 是亲戚，但场景不同。** CodingTask 是"派发方签发 token 给执行方，执行方拿 token 回调"；flow_instruction 是"派发方也参与执行，token 是抢占凭证"。前者防外部伪造，后者防内部并发抢占。

这种形态适合"有多个 worker 消费同一批任务"的场景，典型如流程编排、任务队列消费。

---

## 形态四：内容指纹去重（记忆与知识的 hash）

幂等还有第四种形态——不是防"同一条请求重复"，而是防"同一份内容被重复索引"。这出现在记忆和知识库的入库逻辑里。

记忆索引的核心逻辑（`infrastructure/memory/MemoryIndexManager.ts:370-454`）：

```ts
private async indexContentCore(pathKey, content, source, projectId, agentId, ...): Promise<number> {
  const contentHash = hashContent(content);
  const existing = await getMemoryFile(pathKey);
  if (existing && existing.hash === contentHash) {
    logger.debug(`[MemoryIndexManager] 内容未变更，跳过: ${pathKey}`);
    return 0;   // hash 比较跳过未变更内容
  }
  if (existing) { /* 删旧 chunks + ES 文档 */ }
  const chunks = chunkText(content, pathKey, source, projectId, agentId);
  if (this.esAvailable) {
    try { await defaultMemoryVectorStore.addDocuments(chromaIds, documents, metadatas); }
    catch (esErr) { logger.warn(`写入 ES 失败，仅写入 PG: ${getErrorMsg(esErr)}`); }
  }
  await upsertMemoryFile(memoryFile);    // PG memory_files
  await upsertMemoryChunks(chunks);      // PG memory_chunks
}
```

memory_files 表里有个 `hash` 字段。每次要索引一个文件/一段内容，先算 hash，如果 hash 和已存在的一致，直接 return 0，**0 开销跳过**。只有内容真的变了，才走"删旧 → 重新分块 → 双写 ES+PG"的完整链路。

这种形态和前三种本质不同：

| 维度 | 业务幂等键 | 内容指纹 |
|------|-----------|---------|
| 防什么 | 同一逻辑请求重复执行 | 同一内容重复索引 |
| 锚点 | 业务 ID（turnId 等） | 内容 hash |
| 触发去重时机 | 写入时（唯一约束） | 写入前（先查 hash） |
| 典型场景 | 消息入队、任务派发 | 文件同步、记忆转索引 |

知识库里也有同样的设计：`knowledges` 表有 `file_hash` 字段做去重，`flow_template_version` 带 `checksum`，`flow_resource` 带 `checksum`，`workstation_component` 存 installScript 的 checksum。**只要涉及"内容可能被重复同步"的地方，都有 hash 指纹。**

这种形态的要点：**幂等不一定要等请求到了才识别，可以在内容层提前算指纹，把"重复"消灭在写入之前。** 对高频同步场景（比如记忆每 10 秒增量同步一次），hash 跳过省下的开销是巨大的。

---

## 形态五：灰度幂等（route_rule 的 shadow）

最后一种形态比较隐蔽，叫它"灰度幂等"——一条规则可能被求值很多次，但只在某些状态下产生实际效果。

FusionRouter 路由时（`agents/core/agent/decision/fusion-router.ts:165-201`）：

```ts
route(input: string): RouteResult | null {
  for (const route of this.routes) {
    const score = this.computeScore(route, input);
    if (score >= route.semanticThreshold) {
      if (route.status === 'shadow') {           // shadow 规则只记录，不路由
        this.bumpMetrics(route.id);
        this.shadowHits.push({...});
        continue;
      }
      // ... active 规则才实际命中 ...
    }
  }
}
```

route_rule 表的 `status` 字段有 `active` 和 `shadow` 两个值。**shadow 状态的规则即使分数够高也不实际路由**，只是记录"如果启用会命中多少次"。

这跟幂等有什么关系？关系在于：**shadow 规则保证"一条规则从 shadow 切到 active 的那一刻，系统行为的变化是幂等的"**——因为你已经观察过它在这批输入上的命中情况，切过去不会产生意料之外的爆炸性影响。

这是一种"灰度"的幂等：不是防重复执行，而是防"规则生效"这个动作产生不可预测的放大效应。它是企业级系统才有的设计——demo 系统永远直接 active。

---

## 五种形态放在一起看

把五种形态横向对比，幂等的"形状"就清楚了：

| 形态 | 代表模块 | 防什么 | 锚点 | 实现机制 |
|------|---------|--------|------|---------|
| 业务幂等键 | role_inbox | 重复投递 | idempotency_key | DB 唯一约束 + 复用已存在 |
| 复合键+状态门 | CodingTask | 重复+乱序+伪造 | 三元组+attemptNo+tokenHash | partial unique index + 版本号门 + 回调鉴权 |
| claim_token+租约 | flow_instruction | 并发抢占 | claimToken+leaseUntil | 原子 claim + 租约过期 |
| 内容指纹 | memory/knowledge | 内容重复索引 | hash | 写前查 hash 跳过 |
| 灰度幂等 | route_rule | 规则生效放大 | status=shadow | 只记录不执行 |

这五种的共性是什么？**它们都不依赖"请求只来一次"这个假设。** 它们都承认网络、队列、并发、重启会让同一个逻辑动作重复发生，然后用各自的机制把"重复"收敛成"等效于一次"。

---

## 为什么不能统一成一套

新手看了上面五种，第一反应往往是："为什么不抽一个 IdempotencyService 统一管？" 答案是：**幂等的粒度和语义是业务决定的，不是技术决定的。**

- role_inbox 的幂等粒度是"一条消息"，重复了复用同一条记录即可。
- CodingTask 的幂等粒度是"一次任务派发的完整往返"，包括回调，所以必须有 attemptNo 防乱序。
- flow_instruction 的幂等粒度是"一个 worker 对一条指令的独占处理"，核心是并发互斥。
- memory 的幂等粒度是"一段内容被索引"，跟请求次数无关，只跟内容有没有变有关。

它们的"等"定义都不一样。"执行一次和执行多次效果一致"这句话在不同业务里被翻译成不同的契约。强行抽象成一个统一服务，要么丢失语义（比如把 CodingTask 的 attemptNo 门砍掉），要么变成一个什么都管的上帝类（参数爆炸）。**横切的是"幂等意识"，不是"幂等实现"。**

WinMatrix 的做法是：**每个模块自己实现自己的幂等，但都遵循同一个心智模型——识别锚点、用 DB 约束兜底、重复时给合理的复用语义。** 这比一个统一服务更灵活，也更好维护。

---

## 给后来者的几条总结

1. **幂等的"等"是业务定义的。** 别一上来就问"怎么实现幂等"，先问清楚"在这个场景里，什么叫等价于一次"。
2. **业务幂等键靠 DB 唯一约束兜底**，不要靠应用层 select-then-insert（并发下必败）。重复时复用已存在记录，而不是报错。
3. **跨多次异步往返的操作要带版本号**（如 attemptNo）。迟到的小号不能覆盖大号，这是乱序幂等的核心。
4. **回调场景必须鉴权**（callbackTokenHash）。光认任务 ID 不够，还要认当初签发的 token，防伪造回调。
5. **并发抢占场景用 claim_token + 租约**。原子 claim + lease 过期重抢，是分布式任务队列的通用解。
6. **内容索引场景用 hash 提前跳过**。把"重复"消灭在写入之前，对高频同步场景开销节省巨大。
7. **新规则、新配置上线用 shadow 灰度**。先观察命中再实际生效，是防"生效放大"的企业级纪律。
8. **别追求一个统一的 IdempotencyService。** 横切的是意识，不是实现。每个模块的幂等语义不同，强行抽象会丢失语义或参数爆炸。

分布式系统里，承认"同一个请求一定会被执行多次"，并为每一次重复准备好收敛机制，是把 demo 变成生产系统的第一步。WinMatrix 这五种形态不是全部，但覆盖了最常见的几类场景——下次你设计一个会跨网络、跨进程、跨队列的操作时，先想清楚它属于哪一类。

---

> **下一篇**：[《分布式锁分工：Redis SET NX + Lua 与 PG advisory lock 各管什么》](./24-distributed-lock.md)——幂等解决了"重复执行"，但有些互斥场景要靠锁。WinMatrix 里两套锁各司其职，一套在运行时干活，一套几乎退成了 no-op，为什么？
