# BullMQ 三档队列：按重量分流，而不是一个队列塞所有

> 这是《WinMatrix 开发经验文集》第 17 篇。讲一个看起来"低级"、却决定了后台系统生死的问题：当你有成堆的后台任务——有的要调一次 LLM 跑两分钟，有的只是扫一张表清一下日志，有的每分钟跑一次——该用几个队列、怎么分流？代码来自 WinMatrix 后端真实实现。

很多人做后台任务系统，第一版都是这样的：开一个 BullMQ 队列，所有任务都往里塞，worker 拿到就跑。

```
┌─────────────────────────────────┐
│        一个大队列                │
│  system-log-cleanup              │
│  agent-日会生成（调 LLM 2 分钟）  │
│  system-reminder-delivery        │
│  agent-周报生成（调 LLM 5 分钟）  │
│  system-pending-events-cleanup   │
└─────────────────────────────────┘
```

跑起来你会发现：一个要跑 5 分钟的"周报生成"卡在 worker 上，后面排队的"每分钟跑一次的提醒投递"全被堵住。等周报跑完，可能已经积压了几十条提醒。更糟的是你想给"重任务"限并发（怕打爆 LLM），但队列是同一个，限了重的就连轻的也一起限了。

这篇文章，就讲我们在 WinMatrix 里怎么把后台任务队列做扎实的——核心思路就一句话：**按重量分流，而不是一个队列塞所有**。

---

## 三档队列：按"重量"切分

WinMatrix 的后台任务被切成三个队列，不是按"业务模块"切，而是按**单次执行的成本和耗时**切：

```
┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐
│   scheduled-agent     │  │   scheduled-system    │  │   scheduled-light     │
│   （重：调 LLM）       │  │   （中：DB/ES 维护）   │  │   （轻：高频扫描）     │
│                       │  │                       │  │                       │
│  日会/周报生成         │  │  日志清理             │  │  提醒投递（*/1 min）   │
│  触发智能体响应消息    │  │  记忆整理             │  │  编码任务超时扫描     │
│  agent_run 投影       │  │  可观测数据清理        │  │  悬挂 LLM 调用补偿    │
│                       │  │  工作站清理           │  │  scheduled_run 收敛   │
│  并发 = llmConcurrency│  │  不可 drain          │  │  */5 * * * * 高频     │
└──────────────────────┘  └──────────────────────┘  └──────────────────────┘
```

这个划分的依据是什么？看 `infrastructure/queue/queue.ts` 的队列定义和 `queueRegistry.ts` 的路由表：

```typescript
// src/infrastructure/queue/queue.ts（第 30-43 行）
const scheduledAgentQueue = new Queue<ScheduledJobData>(resolveBullmqQueueName('scheduled-agent'), {
  connection: bullmqQueueConnection,
  defaultJobOptions,
});

const scheduledSystemQueue = new Queue<ScheduledJobData>(resolveBullmqQueueName('scheduled-system'), {
  connection: bullmqQueueConnection,
  defaultJobOptions,
});

const scheduledLightQueue = new Queue<ScheduledJobData>(resolveBullmqQueueName('scheduled-light'), {
  connection: bullmqQueueConnection,
  defaultJobOptions,
});
```

三个队列共用同一条 Redis 连接（`bullmqQueueConnection`），共享同一套 `defaultJobOptions`，但**消费它们的是不同的 worker，配不同的并发**。一个任务该进哪个队列，由 `getQueueForTask` 这个路由函数决定：

```typescript
// src/infrastructure/queue/queue.ts（第 49-53 行）
export function getQueueForTask(taskName: string): Queue<ScheduledJobData> {
  if (SYSTEM_TASKS.has(taskName)) return scheduledSystemQueue;
  if (LIGHT_TASKS.has(taskName)) return scheduledLightQueue;
  return scheduledAgentQueue;   // 默认走 agent 队列（最重）
}
```

注意这个路由的**默认值是 `scheduledAgentQueue`**——最重的那个。为什么默认走重的？因为"不确定重量"的任务，宁可被当作重任务限流，也别让它冲垮轻量队列。**安全侧 default：分不清就当重的处理。**

**教训：队列分流的关键不是队列多，而是"按什么维度分"。** 按业务模块分会越分越碎，按成本/耗时分会自然收敛到三档：重、中、轻。这就是经典调度理论里的"多级反馈队列"思想——不同性质的作业进不同队列，避免相互饿死。

---

## 路由表是纯数据，和队列实例解耦

上面那个 `getQueueForTask` 依赖两个集合：`SYSTEM_QUEUE_TASK_NAMES` 和 `LIGHT_QUEUE_TASK_NAMES`。这两个集合定义在 `queueRegistry.ts` 里：

```typescript
// src/infrastructure/queue/queueRegistry.ts（第 51-70 行）
export const SYSTEM_QUEUE_TASK_NAMES: ReadonlySet<string> = new Set([
  'system-memory-tidy',
  'system-log-cleanup',
  'system-execution-log-cleanup',
  'system-route-rule-discovery',
  'system-transcript-compact',
  'system-observability-cleanup',
  'system-workstation-ephemeral-cleanup',
  'system-side-effect-terminal-cleanup',
]);

export const LIGHT_QUEUE_TASK_NAMES: ReadonlySet<string> = new Set([
  'system-reminder-delivery',
  'system-coding-task-timeout-sweep',
  'system-detached-worker-sweep',
  'system-pending-events-cleanup',
  'system-acceptance-event-reap',
  'system-llm-call-watchdog-sweeper',
  'system-scheduled-run-reconcile',
]);
```

注意文件头部的注释——这很重要：

> **队列元数据注册表（纯数据，零副作用）。从 queue.ts 拆出，避免 import 时触发 BullMQ Queue 实例化。**

为什么要拆？因为 `queue.ts` 里有 `new Queue(...)`，一 import 这个文件就会真的去连 Redis、实例化队列对象。但单元测试常常只需要"这个任务名该进哪个队列"这个纯数据判断，不需要真的连 Redis。把路由表拆成零副作用的纯模块，单测就能放心 import：

```typescript
// 单测里这样 import，不会触发 Redis 连接
import { SYSTEM_QUEUE_TASK_NAMES } from './queueRegistry.js';
```

**教训：把"纯数据/纯规则"和"有副作用的实例化"拆开。** 这是个不起眼但极有用的工程习惯。很多项目的单测又慢又脆，就是因为 import 链路里藏着 `new Queue()` / `new Redis()` 这种副作用。拆开后，单测快、实例化集中、路由规则可静态审计。

---

## 三档任务的并发配置不一样

分流只是第一步。真正让"重量分流"发挥价值的是：**三档队列的 worker 配不同的并发**。

`system` 队列和 `light` 队列的并发是常量（DB/ES 操作，并发可以开高一点）。但 `agent` 队列的并发是动态的，因为它要调 LLM——并发太高会把 LLM provider 打爆，并发太低又吞吐不够：

```typescript
// src/infrastructure/queue/queue.ts（第 14-15 行，re-export 自 concurrencyConstants.js）
export { SYSTEM_QUEUE_CONCURRENCY, LIGHT_QUEUE_CONCURRENCY, CROSS_AGENT_TRIGGER_CONCURRENCY }
  from '../scheduled/concurrencyConstants.js';
```

agent 队列的并发走 `config.scheduled.llmConcurrency`，由环境变量调。而且 agent 队列还共享一个**全局信号量**（scheduled-agent semaphore）——不光队列 worker 有并发上限，真正调 LLM 时还要再过一道信号量，防止"队列并发 10，但每个任务内部又 fan-out 出 5 个子调用"把 LLM 顶死。跨 agent 触发的任务（cross-agent-trigger 队列）也复用同一个信号量。

这就是分流的真正收益：

```
不分流：所有任务挤一个队列，要么全限流（轻任务被拖累），要么全放开（重任务打爆 LLM）
分流后：重任务单独限并发（护 LLM）、轻任务高频跑（护 SLA）、系统维护任务独占一条道（不会 drain）
```

`queueRegistry.ts` 里还有个细节值得注意——`scheduled-system` 队列的能力标记里 `drain: false`：

```typescript
{ name: 'scheduled-system', displayName: '系统维护队列', group: 'scheduled',
  capabilities: { pause: true, retryFailed: true, clean: true, drain: false } },
```

`drain: false` 意思是这个队列**不允许清空**。为什么？因为系统维护任务（日志清理、可观测数据清理、记忆整理）一旦被 drain 掉，可能就再也不补跑了，数据会越积越多。这种"绝不能丢"的任务，要在能力声明层面就禁止 drain 操作，而不是靠运维"记得别点这个按钮"。

**教训：并发配置要按队列的性质分别设，而不是全局一个值。** 重任务护资源（限并发 + 信号量），轻任务护 SLA（高并发 + 高频），关键任务护不丢（drain: false）。一份配置打天下，最后一定某一边出事。

---

## 运行时队列隔离：开发机别串到生产队列

三档分流解决的是"任务之间别互相堵"，但还有一个更阴险的坑：**开发机和线上共用一套 Redis 时，开发机的 worker 会消费到线上的任务**。

想象这个场景：你在本机调试，跑了个 worker。它连的是测试环境的 Redis（和生产同一套）。线上投递了一个"周报生成"任务到 `scheduled-agent` 队列，你的本机 worker 抢到了——然后它用你本地的 LLM key、你本地的配置去执行，结果发到一个莫名其妙的地方，或者直接因为缺配置崩掉。任务状态变成 failed，线上的周报就此失踪。

这不是假想，这是真实发生过的混乱。WinMatrix 的解法是**给队列名加 hostname 后缀**：

```typescript
// src/infrastructure/queue/runtimeQueueIsolation.ts（第 48-59 行）
export function resolveBullmqQueueName(
  baseName: string,
  env: RuntimeIsolationEnvironment = process.env,
  host: string = hostname(),
): string {
  const base = baseName.trim();
  if (!base) throw new Error('BullMQ queue name must not be empty');
  const isolationId = resolveRuntimeIsolationId(env, host);
  if (isolationId === 'prod') return base;              // 生产用原名
  const suffix = `-host-${isolationId}`;                 // 非生产加后缀
  return base.endsWith(suffix) ? base : `${base}${suffix}`;
}
```

效果是：

```
生产环境：   scheduled-agent           （所有 pod 共享）
开发机 A：  scheduled-agent-host-mbp-wj   （只有 A 自己消费）
开发机 B：  scheduled-agent-host-desktop   （只有 B 自己消费）
```

开发机投递的任务只进自己后缀的队列，本机 worker 只消费自己后缀的队列，**和生产队列物理隔离**。你本机跑崩了，不会影响线上一个任务。

### 生产环境的"防呆"：强制 prod

更妙的是 `resolveRuntimeIsolationId` 里的一道守卫：

```typescript
// src/infrastructure/queue/runtimeQueueIsolation.ts（第 34-41 行）
const productionRuntime = env.NODE_ENV === 'production' || nonEmpty(env.KUBERNETES_SERVICE_HOST);
if (productionRuntime && explicitId && explicitId !== 'prod') {
  throw new Error(
    'Production runtime has a non-prod WIN_RUNTIME_ISOLATION_ID; refusing to split shared BullMQ queues.',
  );
}
if (productionRuntime) return 'prod';
```

如果在生产环境（`NODE_ENV=production` 或在 K8s 里）显式设了一个非 `prod` 的 isolation id，**直接抛错拒绝启动**。为什么这么狠？因为生产环境一旦加了 hostname 后缀，多个 pod 就会各跑各的队列，定时任务会被每个 pod 各执行一次——这是灾难。这种"一旦配错就静默出错"的场景，必须 fail-fast，宁可启动失败，也不能带着错误配置跑起来。

**教训：多实例共享队列的系统，开发和生产必须物理隔离队列名。** 而且生产环境的隔离逻辑要做成"防呆"——配错了直接报错，不留任何"也许能跑"的余地。我们后面会有一篇专门讲配置热更新的文章（第 15 篇讲过 PG LISTEN/NOTIFY），那里也会反复强调："危险配置要 fail-fast，不要 fail-silent"。

---

## worker 连接的一个反直觉细节：禁止 commandTimeout

最后一个容易被忽略的点。BullMQ 的 Queue 和 Worker 用的是两条不同的 Redis 连接：

```typescript
// src/infrastructure/persistence/database/bullmqConnections.ts（第 15-33 行）
export const bullmqQueueConnection = new Redis(config.redisUrl, {
  maxRetriesPerRequest: null,   // BullMQ 要求
  commandTimeout: Number(process.env.REDIS_BULLMQ_COMMAND_TIMEOUT) || 30000,
  // ...
});

export const bullmqWorkerConnection = new Redis(config.redisUrl, {
  maxRetriesPerRequest: null,   // BullMQ 要求
  // 明确不加 commandTimeout —— blocking 命令不可 timeout
  retryStrategy(times) { return Math.min(times * 200, 5000); },
});
```

注意 `bullmqQueueConnection` 有 30 秒的 `commandTimeout`，而 `bullmqWorkerConnection` **明确没有**。注释解释了原因：

> Worker 使用 `BRPOLLPUSH` / `XREADGROUP BLOCK` 等 blocking 命令，加 commandTimeout 会导致每 N 秒被 ioredis 强制 reject，触发 reconnect 风暴。

这是 BullMQ 一个很隐蔽的坑。Worker 消费任务时，会对 Redis 发一个 blocking 命令——"如果没有任务就阻塞等待，最多等 X 毫秒"。这个命令**本身就是要长时间挂着等**的。如果你给连接设了 commandTimeout=30s，那每次 blocking 等到 30 秒就会被 ioredis 当成超时强制 reject，触发重连。重连完又 blocking，又 30 秒 reject……这就是一个无谓的"reconnect 风暴"，既浪费连接又让日志全是噪音。

而 Queue 端（投递任务用）发的都是普通命令（`ZADD`、`HSET`），该设超时就设超时，防止某次投递卡死整个调用链。

**教训：blocking 操作和普通操作要用不同的连接配置。** 这是一个"框架行为和库配置冲突"的典型例子——ioredis 的 commandTimeout 是个通用机制，但 BullMQ 的 worker 连接天然是 blocking 的，两者放一起就出事。用任何带 blocking 语义的库（Redis blocking 命令、长轮询、stream 消费），都要先想清楚"这个连接能不能设超时"。

---

## 一个完整的任务流转

把上面几点串起来，一个"周报生成"任务从投递到执行的完整链路：

```
1. ScheduledTaskService 注册周报任务（cron: 0 9 * * 1）
       │
       ▼
2. cron 触发 → getQueueForTask('agent-周报生成')
       │  不在 SYSTEM 集合，不在 LIGHT 集合
       │  → 返回 scheduledAgentQueue
       ▼
3. scheduledAgentQueue.add(jobData)
       │  队列名 = resolveBullmqQueueName('scheduled-agent')
       │  生产环境 → 'scheduled-agent'
       │  开发机   → 'scheduled-agent-host-mbp-wj'
       ▼
4. scheduled-worker 进程的 Worker 消费（concurrency = llmConcurrency）
       │  连接 = bullmqWorkerConnection（无 commandTimeout，blocking 安全）
       ▼
5. 执行前先获取 scheduled-agent 信号量（全局 LLM 并发闸门）
       │  拿到 → 调 LLM 跑周报
       │  拿不到 → moveToDelayed + DelayedError 延后重投
       ▼
6. 完成 → removeOnComplete 保留 200 条/7 天
   失败 → attempts:2 指数退避重试，removeOnFail 保留 100 条/7 天
```

每一步都有明确的设计依据，不是随便堆出来的。

---

## 给后来者的几条总结

1. **按"重量"分队列，不要按业务模块分。** 重/中/轻三档是自然收敛的结果，对应不同的并发和 SLA 要求。
2. **路由表用纯数据，和队列实例解耦。** 单测能秒级跑，不用连 Redis。
3. **三档队列配不同的并发和重试策略。** 重任务护资源（限并发 + 信号量），轻任务护 SLA（高频 + 高并发），关键任务护不丢（drain: false）。
4. **开发机和生产必须物理隔离队列名。** 用 hostname 后缀，让本机 worker 只消费自己的队列。
5. **生产环境的隔离配置要 fail-fast。** 配错了直接拒绝启动，别留"也许能跑"的灰色地带。
6. **blocking 操作的连接禁止设 commandTimeout。** Queue 连接可以设，Worker 连接绝不能设，否则 reconnect 风暴。
7. **分不清重量的任务，默认走最重的队列。** 安全侧 default：宁可被限流，也别冲垮轻量队列。

后台任务系统是 AI 平台的"地下管线"——用户看不见，但它一旦堵了，上面所有的 Agent、对话、流程全得停摆。把队列分流、隔离、并发配置做扎实，你的平台才有个稳的底座。

---

> **下一篇**：[《构建与部署：dev/prod/k8s 四进程对齐》](./18-build-deploy-four-process.md)——一个代码库要跑成 api / scheduled / rag / embedding 四个进程，开发、docker-compose、k8s 三套环境怎么对齐？构建链又该怎么搭？
