# 会话执行调度器：一个会话里多条消息怎么排队

> 这是《WinMatrix 开发经验文集》第 46 篇，第三批"更细的源码剖析"的收尾篇。前面讲的都是"单条消息怎么处理"——路由、分诊、执行模式、人格。这一篇讲一个更外层的问题：**一个会话里，用户连发多条消息，系统怎么调度**。这看起来是个小问题，但做错了会让会话乱套。

先说一个每个人都会遇到的场景。

用户在群里发了一句"帮我做个技术方案"。系统开始处理——这是个 coordinator 模式的多步任务，要规划、调研、写方案，跑个几十秒很正常。

用户发完这句，等了 5 秒，觉得没说清楚，又补了一句"重点关注并发安全"。再过 3 秒，又补一句"用我们项目的命名规范"。

现在系统收到了三条消息。问题来了：

- 这三条消息，是各自独立处理（并行跑三个任务）？显然不行——它们是同一个任务的不同补充。
- 是只处理第一条、忽略后两条？也不行——用户补充的信息很重要。
- 是把后两条排队，等第一条处理完再逐条处理？方向对了，但细节很多——排队时要不要告诉用户"你的消息在排队"？第一条处理完，后两条怎么合进去？

这就是 **conversationExecutionDispatcher + CoordinatorAdapter** 要解决的问题。

---

## 先看会话模式：三种 sessionMode

在讲调度器之前，先看会话有哪些模式。WinMatrix 的会话模式（sessionMode）定义得很简洁：

```typescript
// agents/core/runtime/session/conversationMessageStoreContracts.ts（第 30 行）
export type ConversationSessionMode = 'coordinator' | 'interactive' | 'react';
```

三种模式，[第 44 篇](./44-execution-modes.md) 讲执行模式时提过，这里从会话调度角度再区分：

| sessionMode | 调度特征 | 多消息处理 |
|-------------|---------|-----------|
| `coordinator` | 串行队列 | 排队，逐条执行 |
| `interactive` | 多角色轮转 | 按角色议程推进 |
| `react` | 单 Agent 自循环 | 即时执行 |

注意 `coordinator` 和 `react` 的关键区别：coordinator 要排队（多步任务不能并行），react 不需要排队（单 Agent 即时响应）。这个差异决定了调度器对不同模式的行为不同。

`normalizeConversationSessionMode` 把无效值统一回退到 coordinator：

```typescript
// 无效值统一返回 coordinator
export function normalizeConversationSessionMode(mode: string | null | undefined): ConversationSessionMode {
  if (mode === 'interactive' || mode === 'react') return mode;
  return 'coordinator';  // 默认 / 非法值
}
```

**默认回退到最保守的 coordinator**——这是个安全选择。会话模式存 DB 里，可能读到脏数据或老版本数据。遇到不认识的值，按最严格的串行队列处理，不会出错；如果默认 react（即时执行），遇到本该排队的 coordinator 场景，多条消息并行跑就乱了。

---

## ConversationExecutionDispatcher：按 sessionMode 选 adapter

调度器的核心是个 dispatcher，它按 sessionMode 选择对应的 adapter：

```typescript
// interface/api/agents/ws/conversationExecutionDispatcher.ts（第 23-47 行）
export class ConversationExecutionDispatcher {
  private readonly adapters: Map<ConversationSessionMode, ConversationExecutionAdapter>;

  constructor(adapters: readonly ConversationExecutionAdapter[]) {
    this.adapters = new Map(adapters.map((adapter) => [adapter.sessionMode, adapter]));
  }

  async dispatch(ctx: ChatRequestContext, socket: WsSocket): Promise<ConversationExecutionResult> {
    const sessionMode = ctx.sessionMode;
    const adapter = this.adapters.get(sessionMode);

    if (!adapter) {
      throw new Error(`CONVERSATION_EXECUTION_ADAPTER_NOT_FOUND:${sessionMode}`);
    }

    await adapter.execute(ctx, socket);

    return {
      requestedSessionMode: ctx.sessionModeDegradation?.requestedMode ?? sessionMode,
      effectiveSessionMode: sessionMode,
      adapterSessionMode: adapter.sessionMode,
      ...(ctx.sessionModeDegradation ? { degraded: true } : {}),
    };
  }
}
```

这个设计是经典的**策略模式**——dispatcher 不关心具体怎么执行，只负责按 sessionMode 找到对应的 adapter，把活派下去。

几个设计要点：

**adapter 是个接口**（`ConversationExecutionAdapter`），每种模式一个实现。dispatcher 只认接口，不认具体类。加新模式只要加个 adapter，dispatcher 不用改。

**找不到 adapter 直接抛错**。不静默降级——sessionMode 是个有限集合（三种），出现第四种值说明数据有问题，应该让错误暴露而不是假装没事。

**返回 `ConversationExecutionResult` 带 `degraded` 标记**。注意 `sessionModeDegradation`——这是"请求的模式不可用、降级到了别的模式"的情况。比如请求 interactive 但当前会话没有绑定 interactive 角色，可能降级到 coordinator。`degraded: true` 让上层知道发生了降级，可以提示用户。

### 默认注册三个 adapter

```typescript
// conversationExecutionDispatcher.ts（第 245-253 行）
export function createDefaultConversationExecutionDispatcher(): ConversationExecutionDispatcher {
  return new ConversationExecutionDispatcher(
    [
      new CoordinatorAdapter(),              // ← 含队列逻辑
      createInteractiveEnvironmentAdapter(),
      createReactAdapter(),
    ],
  );
}
```

三个 adapter，但复杂度天差地别：

```typescript
// interactive adapter（第 227-234 行）—— 薄封装
export function createInteractiveEnvironmentAdapter(): ConversationExecutionAdapter {
  return {
    sessionMode: 'interactive',
    execute: async (ctx, socket) => {
      await handleInteractiveMode(ctx, socket);   // 直接调 handler
    },
  };
}

// react adapter（第 236-243 行）—— 薄封装
export function createReactAdapter(): ConversationExecutionAdapter {
  return {
    sessionMode: 'react',
    execute: async (ctx, socket) => {
      await handleNormalMode(ctx, socket);        // 直接调 handler
    },
  };
}
```

interactive 和 react 的 adapter 都是薄封装——拿到 ctx 直接调对应的 handler，没有队列、没有缓冲。这是因为它们的调度语义不需要排队。

**只有 CoordinatorAdapter 是个真正的类**，带状态、带队列逻辑。因为只有 coordinator 模式需要"会话内串行执行"。

---

## CoordinatorAdapter：内存队列的串行执行

CoordinatorAdapter 是这篇的主角。它的职责是：**保证一个 coordinator 会话里，同一时间只有一个 Turn 在执行，多余的消息排队，执行完逐条回放**。

```typescript
// conversationExecutionDispatcher.ts（第 57-121 行）
export class CoordinatorAdapter implements ConversationExecutionAdapter {
  readonly sessionMode = 'coordinator';

  private readonly registry = getSessionExecutionContextRegistry();

  async execute(ctx: ChatRequestContext, socket: WsSocket): Promise<void> {
    // 1. 获取或创建会话执行上下文
    const sessionCtx = this.registry.getOrCreate({
      conversationId: ctx.conversationId,
    });

    // 2. 入队检查
    const processResult = sessionCtx.processInput(ctx.input, {
      userId: ctx.userId,
      userName: ctx.userName,
      projectId: ctx.project.id,
      // ...
    });

    // 3. 如果已入队，发送通知并返回
    if (processResult.queued) {
      await this.notifyQueued(ctx, socket, processResult);
      return;
    }

    // 4. 如果不应处理（队列已满或上下文已销毁）
    if (!processResult.shouldProcess) {
      return;
    }

    // 5. 执行单轮聊天
    try {
      await handleNormalMode(ctx, socket);
    } finally {
      // 6. 标记为空闲
      sessionCtx.markIdle();
    }

    // 7. 处理缓冲消息（逐条执行）
    await this.processBufferedMessages(ctx, socket, sessionCtx);
  }
}
```

整个流程用图表示：

```
消息进来，CoordinatorAdapter.execute：
   │
   ▼
getOrCreate sessionCtx（会话执行上下文）
   │
   ▼
sessionCtx.processInput(input)
   │
   ├─ 当前有 Turn 在跑？
   │    ├─ 是 → 入队，返回 { queued: true, queuePosition: N }
   │    │        │
   │    │        ▼
   │    │      notifyQueued（告诉用户"你排第 N 位"）
   │    │      return（不执行，等当前 Turn 完了再说）
   │    │
   │    └─ 否 → 直接执行
   │              │
   │              ▼
   │           handleNormalMode（执行这一轮）
   │              │
   │              ▼
   │           finally: markIdle（标记空闲）
   │              │
   │              ▼
   │           processBufferedMessages（把排队的逐条跑）
```

每一步都有讲究。

### SessionExecutionContext：会话级状态机

`sessionCtx` 是会话级的执行上下文，存在内存注册表里（`getSessionExecutionContextRegistry`），按 conversationId 查找或创建。它维护这个会话的执行状态——当前有没有 Turn 在跑、队列里有几条消息。

`processInput` 的返回结果有三种情况：

**直接执行**（`shouldProcess: true, queued: false`）——当前没 Turn 在跑，这条消息直接执行。这是最常见的路径。

**入队**（`queued: true`）——当前有 Turn 在跑，这条消息进队列等。返回队列位置。

**拒绝**（`shouldProcess: false`）——队列满了或上下文已销毁。静默丢弃（记日志）。

### notifyQueued：告诉用户在排队

消息被入队时，CoordinatorAdapter 会立即通知用户：

```typescript
// conversationExecutionDispatcher.ts（第 123-148 行）
private async notifyQueued(
  ctx: ChatRequestContext,
  socket: WsSocket,
  processResult: ProcessInputResult,
): Promise<void> {
  const emitter = createStreamingEmitter(`early_${Date.now()}`);
  const unsubscribe = emitter.subscribe((event) => {
    if (socket.readyState === 1) {
      socket.send(JSON.stringify(event));
    }
  });

  try {
    emitter.emitPhase('queued');
    emitter.emitStream('async:start', {
      taskId: `queue_${ctx.conversationId}`,
      position: processResult.queuePosition,
      rejected: false,
    });
  } finally {
    unsubscribe();
  }
}
```

这是个很重要的体验细节。用户发了消息，如果系统默默排队不反馈，用户会以为"消息丢了"或者"系统卡了"，然后反复重发——重发的消息又进队列，雪上加霜。

`emitPhase('queued')` + `emitStream('async:start', { position })` 立即推一条"你排第 N 位"的反馈。用户看到反馈，知道消息被接受了、在排队，就不会重发。

**排队要透明**——用户不需要知道"队列"这个概念，但他需要知道"我的消息被收到了、要等一下"。这一条反馈把焦虑消除了。

### processBufferedMessages：逐条回放

当前 Turn 执行完（`markIdle` 之后），CoordinatorAdapter 会检查队列里有没有积压消息，逐条处理：

```typescript
// conversationExecutionDispatcher.ts（第 156-224 行）
private async processBufferedMessages(
  ctx: ChatRequestContext,
  socket: WsSocket,
  sessionCtx: SessionExecutionContext,
): Promise<void> {
  let bufferedTurnCount = 0;
  let nextMsg = sessionCtx.takeNext();   // 从队列取下一条

  while (nextMsg) {
    bufferedTurnCount += 1;

    // 构造单轮上下文（使用缓冲消息的元数据）
    const metaProject: ProjectRef = {
      id: nextMsg.meta?.projectId ?? ctx.project.id,
      code: nextMsg.meta?.projectCode ?? ctx.project.code,
      name: nextMsg.meta?.projectName ?? ctx.project.name,
    };
    const metaTarget: TargetRef = {
      roleId: nextMsg.meta?.roleId ?? ctx.target.roleId,
      digitalEmployeeId: nextMsg.meta?.digitalEmployeeId ?? ctx.target.digitalEmployeeId,
    };

    const singleCtx: ChatRequestContext = {
      ...ctx,
      input: nextMsg.mergedInput,    // 合并后的输入
      userId: nextMsg.meta?.userId ?? ctx.userId,
      // ...
      sessionMode: 'coordinator',
    };

    // 尝试开始运行
    if (!sessionCtx.tryStartRun()) {
      break;   // 无法启动（比如又来新消息抢占了），退出循环
    }

    try {
      await handleNormalMode(singleCtx, socket);   // 执行这一条
    } catch (error) {
      logger.error({ /* ... */ });
    } finally {
      sessionCtx.markIdle();              // 每轮执行后标记空闲
      nextMsg = sessionCtx.takeNext();    // 取下一条
    }
  }
}
```

几个关键设计：

**`mergedInput`——缓冲消息会被合并**。注意取出的消息叫 `nextMsg.mergedInput`，不是原始 input。这意味着排队期间，如果多条消息都是"补充说明"，它们可能被合并成一条，而不是逐条独立执行。这回到我们开头的场景——用户连发"做技术方案""关注并发安全""用项目命名规范"，这三条很可能合并成一条更完整的输入来执行，而不是跑三个任务。**合并 vs 逐条是设计选择，合并通常更适合"用户补充"场景。**

**用缓冲消息自己的 meta 构造上下文**。每条缓冲消息带自己的 meta（projectId、roleId、userId）。构造 singleCtx 时优先用缓冲消息的 meta，而不是用最初那条消息的。因为用户可能在不同上下文下发消息（虽然少见），用消息自带的 meta 更准确。

**`tryStartRun` 防并发抢占**。每条缓冲消息执行前要 `tryStartRun`——尝试把状态从 idle 切回 running。如果切不成功（比如期间又有新消息进来抢占了），直接 break，剩下的留给下一轮。这保证"同一时间只有一个 Turn 在跑"的不变量。

**每轮 finally markIdle + takeNext**。一条执行完，标记空闲，取下一条。循环直到队列空。这是个同步循环——缓冲消息是串行执行的，不会并行。

### 异常不中断循环

注意 try-catch 的位置：

```typescript
try {
  await handleNormalMode(singleCtx, socket);
} catch (error) {
  logger.error({ /* 记录错误 */ });
  // 不 break，继续处理下一条
} finally {
  sessionCtx.markIdle();
  nextMsg = sessionCtx.takeNext();
}
```

某条缓冲消息执行失败了，**只记日志，不中断循环**——下一条继续处理。这是合理的：用户补充的三条消息，如果第二条执行失败了，第三条不该因此被丢弃。每条消息的执行是独立的，失败隔离。

---

## 会话运行锁：保证同时只有一个 run

CoordinatorAdapter 的内存队列保证了"adapter 层面的串行"。但还有一个更底层的锁——`conversationRunLocks`：

```typescript
// interface/api/agents/ws/agentWebSocketState.ts（第 9 行）
export const conversationRunLocks = new Map<string, boolean>();
```

这是一个**会话级运行锁**，保证同一个 conversationId 同时只允许一个 run。它和 CoordinatorAdapter 的队列是两层保障：

```
消息进来
   │
   ▼
conversationRunLocks 检查（底层锁）
   │  同一会话同时只允许一个 run
   │
   ▼
CoordinatorAdapter.execute（上层队列）
   │  coordinator 模式下，消息排队
   │
   ▼
handleNormalMode（实际执行）
```

为什么要有两层？因为 **conversationRunLocks 是跨模式的底层保障**——不管是 coordinator、interactive 还是 react，都不能出现"同一会话两个 run 并行"。而 CoordinatorAdapter 的队列是 coordinator 模式特有的，它处理的是"多步任务期间的补充消息怎么排队"。

react 模式不需要队列（即时响应），但它依然受 conversationRunLocks 约束——如果 react 的一个 Turn 没跑完，同一会话再来一条消息，也得等。只是 react 的 Turn 通常很短（单 Agent 推理），用户感知不到排队。

**底层锁管"能不能跑"，上层队列管"怎么排队"**。分层各司其职。

---

## 和 abort 的配合：用户点停止怎么办

讲完了排队，还有个必须处理的场景：**用户点了停止**。

在 [第 1 篇](./01-turn-engine.md) 我们讲过，stop 控制帧绕过 AI 管道，直接查 `conversationAbortControllers` 并 abort。这个 abort 信号会贯穿调用链，正在执行的 Turn 会中断。

中断之后，CoordinatorAdapter 的 `finally` 块会执行 `markIdle`——状态切回空闲。然后 `processBufferedMessages` 会继续处理队列里的消息。

这意味着：**用户停止的是"当前这一条"，不是"整个队列"**。停止当前任务后，排队的消息会继续执行。这通常是对的——用户想停的是那个跑飞了的任务，不是放弃所有补充。

但如果用户是想"全停"呢？那他需要在每次新消息开始前再点停止。这是个体验权衡——"停当前"比"停全部"更常见，系统按更常见的场景设计。

（设计意图：如果要做"一键全停"，可以在 stop 控制帧处理时清空 sessionCtx 的队列。但当前源码里 CoordinatorAdapter 没有暴露清空队列的接口，这是基于"停当前"的假设。）

---

## 完整的调度流程

把所有部分串起来，一个 coordinator 会话处理多条消息的完整流程：

```
用户发消息 1："做技术方案"
   │
   ▼
WebSocket 入口 → conversationRunLocks 加锁
   │
   ▼
CoordinatorAdapter.execute
   │
   ▼
sessionCtx.processInput → 当前空闲，直接执行
   │
   ▼
handleNormalMode（开始跑技术方案任务）
   │
   │  任务跑着（规划、调研...）
   │
   │  === 期间用户连发两条 ===
   │
   │  用户发消息 2："关注并发安全"
   │     │
   │     ▼
   │  CoordinatorAdapter.execute
   │     │
   │     ▼
   │  sessionCtx.processInput → 当前有 run，入队
   │     │
   │     ▼
   │  notifyQueued → 推"你排第 1 位"
   │
   │  用户发消息 3："用项目命名规范"
   │     │
   │     ▼
   │  同上，入队，推"你排第 2 位"
   │
   ▼
消息 1 执行完
   │
   ▼
finally: markIdle
   │
   ▼
processBufferedMessages：
   取消息 2（可能已和消息 3 合并）→ tryStartRun → 执行
   │
   ▼
   finally: markIdle → takeNext（队列空了）
   │
   ▼
conversationRunLocks 解锁
   │
   ▼
完成
```

---

## 给后来者的总结

1. **ConversationExecutionDispatcher 是策略模式**——按 sessionMode 选 adapter，三种模式三个 adapter。dispatcher 只认接口，加模式不改 dispatcher。找不到 adapter 直接抛错，不静默降级。

2. **sessionMode 无效值回退 coordinator**。最保守的串行队列。回退到即时执行（react）会让该排队的场景并行，乱套。

3. **只有 coordinator 模式需要队列**。interactive 和 react 的 adapter 是薄封装，直接调 handler。coordinator 要保证"会话内串行执行"，所以 CoordinatorAdapter 是个带状态的类。

4. **CoordinatorAdapter 内存队列三态**：直接执行（当前空闲）/ 入队（当前有 run）/ 拒绝（队列满）。`sessionCtx.processInput` 决定走哪条路。

5. **排队要透明**。notifyQueued 立即推"你排第 N 位"。不反馈会让用户以为消息丢了，反复重发加剧队列。用户不需要知道"队列"概念，但要知道"消息被收到了、要等"。

6. **缓冲消息合并 + 逐条回放**。processBufferedMessages 从队列取消息（可能是合并后的 mergedInput），逐条串行执行。每条 tryStartRun 防并发抢占，finally markIdle + takeNext 循环到空。

7. **缓冲消息执行失败不中断循环**。一条失败只记日志，下一条继续。失败隔离——用户补充的三条消息，第二条失败不该连累第三条。

8. **两层串行保障**：conversationRunLocks（跨模式底层锁，同会话同时只一个 run）+ CoordinatorAdapter 队列（coordinator 模式特有的排队逻辑）。底层锁管"能不能跑"，上层队列管"怎么排队"。

9. **停止的是"当前这一条"，不是整个队列**。abort 后 markIdle，缓冲消息继续执行。按"更常见场景"设计——用户通常想停跑飞的任务，不是放弃所有补充。

会话执行调度器看似是个小模块（就 254 行代码），但它解决的是一个"用户体验能不能立住"的核心问题。用户连发消息是日常场景，处理得好（排队、反馈、合并、串行），用户觉得系统稳；处理得不好（并行跑飞、消息丢失、无反馈），用户觉得系统不可靠。**工程价值不总是看代码量，有时候看的是"把常见的复杂场景处理得让用户感觉不到复杂"。**

---

> **下一篇**：[《数字员工的"专长画像"：specialist profile 怎么驱动能力绑定》](./47-specialist-profile.md)——第三批前六篇讲完了。下一篇进入第 47 篇，讲数字员工的专长画像——orchestration/techManagement/sre/qualityAssurance 等岗位画像是怎么定义的，画像如何驱动能力绑定。
