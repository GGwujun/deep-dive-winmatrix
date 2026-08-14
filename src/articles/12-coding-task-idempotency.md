# 一次编码任务的幂等与回调：迟到的 attempt 不会覆盖新状态

> 这是《WinMatrix 开发经验文集》第 12 篇。上一篇讲了编码工作站的镜像分层，这一篇往下钻一层：当 LLM 在工作站里真正开始"跑一个任务"时，幂等、重试、回调这三件事是怎么做扎实的。

分布式系统里有个经典难题：**一个操作执行到一半，进程挂了，重启后怎么办？**

对普通业务来说，答案通常是"重试 + 幂等"。但对编码任务来说，这个问题格外凶险——因为编码任务的"执行"是发生在**远程 K8s Pod 里的 LLM 引擎**上，它的生命周期比一次数据库写入复杂得多：

```
WinMatrix 后端  ──派发任务──→  K8s Pod（claude_code 引擎）
     ↑                              │
     │                              │ 跑了很久（可能几分钟、十几分钟）
     └────回调通知──────────────────┘
```

中间任何一环都可能出事：

- 后端派发任务后，Pod 还在跑，后端自己崩了重启——重启后要不要重派？重派会不会和还在跑的那次撞车？
- Pod 跑完了回调通知后端，网络抖动回调丢了，后端以为没完成又派了一次——结果两次的产物会不会冲突？
- Pod 跑了一半超时，后端判定失败发起了重试（attempt 2），但这时 attempt 1 的迟到回调姗姗来迟——attempt 1 的结果会不会把 attempt 2 的状态覆盖掉？

这些坑每一个都能让你在凌晨三点爬起来。这一篇就讲 WinMatrix 的 `CodingTask` 模型怎么把它们一个个堵住。

---

## CodingTask 模型：先看字段长什么样

核心模型在 schema 里，字段不少，但每一个都有用：

```prisma
// prisma/schema.prisma（第 310-386 行）
model CodingTask {
  id                  String   @id @default(uuid())
  workstationId       String   @map("workstation_id")
  triggerTool         String   @map("trigger_tool")
  // triggerTool = coding_task | sre_skill_task | execute_task
  idempotencyKey      String   @map("idempotency_key")
  attemptNo           Int      @default(1) @map("attempt_no")
  callbackTokenHash   String?  @map("callback_token_hash")
  taskFingerprint     String   @map("task_fingerprint")
  claudeSessionId     String?  @map("claude_session_id")
  hostProjectPath     String?  @map("host_project_path")
  workstationProjectPath String? @map("workstation_project_path")
  artifactRunRoot     String?  @map("artifact_run_root")
  lifecycleMetadata   Json?    @map("lifecycle_metadata")
  // ... status 字段（running/completed/failed/...）
}
```

光看字段就有几个关键的：`idempotencyKey`、`attemptNo`、`callbackTokenHash`、`taskFingerprint`。下面逐个拆，它们各自防的是哪类故障。

---

## 第一道防线：running 态去重，防"同任务被派两次"

最基础的幂等是：**同一个逻辑任务，不管被触发多少次，只能有一个在跑。**

WinMatrix 的做法是用 `(workstationId, triggerTool, idempotencyKey)` 这个组合在 **running 态**去重。也就是说：如果已经有一个 running 状态的任务拿着这个组合在跑，新来的同组合任务不会重复派发。

为什么是"running 态去重"而不是"全局唯一"？因为编码任务是有生命周期的——一个任务跑完了（completed），用户可能想用同样的参数再跑一次。如果全局唯一，第二次就被拦死了。**去重只在 running 态生效**，终态的任务可以重新发起，这才符合用户预期。

这背后有一个容易忽视的点：**幂等键的含义要和业务语义对齐**。`idempotencyKey` 在这里代表的是"这是同一次意图"，而不是"这是同一个参数"。同样是"修 bug #123"这个意图，第一次失败后重试，是同一次意图（应该去重）；但第一次成功后用户主动说"再修一次"，是新一次意图（不该去重）。状态字段的存在让这个区分成为可能。

---

## 第二道防线：attemptNo 防迟到 attempt 覆盖新状态

这是所有坑里最阴险的一个。场景还原：

```
时间线：
t1: 后端派发 attempt 1 → Pod 开始跑
t2: 后端等了很久没收到回调，判定 attempt 1 超时失败
t3: 后端派发 attempt 2 → Pod 开始跑
t4: attempt 2 跑完，回调，状态写成 completed（attempt 2 的结果）
t5: attempt 1 的 Pod 终于跑完了，迟到的回调姗姗来迟
    ── 如果不加防护，attempt 1 的结果会覆盖 attempt 2 的状态！
```

t5 这一幕是灾难——最新的、正确的 attempt 2 结果，被一个过时的 attempt 1 结果盖掉了。用户看到的是 attempt 1 的旧产物，而它本该被丢弃。

防护靠的是 `attemptNo`。回调写入状态时，**必须校验回调里的 attemptNo 是否等于当前任务的 attemptNo**：

```
回调到达时：
  if (回调的 attemptNo < 当前任务 attemptNo)
    → 这是一个过时的回调，直接丢弃，不写状态
  if (回调的 attemptNo === 当前任务 attemptNo)
    → 正常写入状态
```

这个简单的数值比较，挡住了"迟到者覆盖新状态"。**核心思想：状态只能向前走，不能被更老的版本回滚。**

业界这种模式叫**单调性保护**（monotonic guard）——用一个单调递增的版本号，拒绝任何比当前版本小的写入。它比"加锁"优雅得多：不需要全局锁，不需要分布式协调，一个数字比较就够。代价是每次"换代"要记得 bump 一下 attemptNo。

> 补充：`agent_worker_execution` 表（schema 第 2053-2084 行）也有同样的 attemptNo 字段和唯一约束 `@@unique([runId, stepId, attemptNo])`，保证 worker 执行层的重试也不会撞车。这是同一套思路在两个层面的复用。

---

## 第三道防线：callbackTokenHash 鉴权，防伪造回调

回调是后端"被动接收"的——Pod 跑完了，主动 HTTP 回调后端。这里有个安全问题：**谁能回调？**

如果任何人都能往 `/api/coding-task/callback` 发请求伪造回调，那整个状态机就形同虚设——攻击者可以伪造一个"completed"回调，让任务提前结束。

WinMatrix 的做法是**一次性回调令牌 + hash 存储**：

- 派发任务时，后端生成一个随机 token，发给 Pod（Pod 拿着它作为回调凭证）。
- 后端把这个 token 的 **hash** 存进 `callbackTokenHash`（不存明文，防数据库泄露）。
- 回调到达时，后端算回调 token 的 hash，和库里存的比，对上才接受。

为什么存 hash 不存明文？和密码不存明文一个道理——数据库被拖库了，攻击者拿到的只是一堆 hash，没法直接用来伪造回调。这是"防御深度"（defense in depth）的体现：**你不能假设任何一层是绝对安全的，每一层都要假设别的层可能失守。**

token 是一次性的：任务一旦终态（completed/failed），这个 token 就作废了，哪怕泄露也没用。这是把"令牌的有效期"和"任务的生命周期"绑定，最小化暴露面。

---

## 第四道防线：partial index 锁定可复用的成功 session

编码任务有一个很值钱的资源：**Claude session**。claude_code 引擎在一个 session 里跑，session 里会积累上下文（读过的文件、之前的对话）。如果每次重试都开新 session，上下文全丢，引擎要重新读一遍代码——又慢又贵。

理想情况是：**重试时尽量复用上次的 session**。但"复用"有风险——你不能无脑复用，否则一个跑崩了的 session 被反复重用，永远跑不出来。

WinMatrix 在 `CodingTask` 上建了一个 **partial index**（注意是普通部分索引，不是唯一索引），把"哪些 session 值得被复用"这个集合高效地圈出来：

```prisma
// prisma/schema.prisma（第 377 行）
// partial index（非唯一）：
@@index(
  [sessionBindingKey, taskFingerprint, status, updatedAt],
  where: raw("(claude_session_id IS NOT NULL AND status = 'completed')")
)
```

这个 partial index 的含义是：**只对"成功完成且带 session"的任务建索引**。系统想找一个可复用的 session 时，查这个部分索引就能快速命中，而不用全表扫、也不用过滤掉失败任务。

换句话说：

- 一个成功完成的任务（status='completed' 且有 claude_session_id），它的 (sessionBindingKey, taskFingerprint) 进了这个索引——下次用同样的指纹来发起任务，系统能快速查到这个成功的 session 并复用。
- 失败的任务、没 session 的任务，根本不进这个索引，既不占索引体积，也不会被误当作"可复用"候选。

注意这里是 **partial index 而非 partial unique index**——它**不强制唯一性**，作用是"圈定一个可高效查询的候选集合"，真正的"复不复用、复用哪个"的判断由应用层在查到候选后决定。这是个容易看错的细节：partial unique index 会"禁止重复"，而这里要的是"快速找到候选"，两者目的相反。**partial index + 应用层判断，比一个 unique 约束更贴合"只复用成功 session"的语义**——如果用 unique，反而会禁止两个成功任务共享同一指纹的合理场景。

---

## 五道防线串起来：一次完整的重试时序

把上面四道防线 + 状态校验串起来，一次"派发 → 超时 → 重试 → 迟到回调"的完整时序是这样的：

```
后端                              K8s Pod
  │
  │── t1: 派发 attempt 1 ────────→  开始跑（持 token T1）
  │    记录 idempotencyKey=K, attemptNo=1, callbackTokenHash=hash(T1)
  │
  │    （等待回调，超时窗口到）
  │── t2: 判定 attempt 1 超时，bump attemptNo=2
  │
  │── t3: 派发 attempt 2 ────────→  开始跑（持 token T2）
  │    更新 callbackTokenHash=hash(T2)
  │
  │                              │ t4: attempt 2 跑完
  │←──── 回调（token T2）─────────│
  │  校验 hash(T2)==库存 ✓
  │  校验 attemptNo 2==当前 2 ✓
  │  写入 completed（attempt 2 的结果）
  │
  │                              │ t5: attempt 1 迟迟跑完
  │←──── 回调（token T1）─────────│
  │  校验 attemptNo 1 < 当前 2 ✗
  │  丢弃！不覆盖状态
  │
  ▼
  最终状态 = attempt 2 的结果（正确）
```

注意几个细节：

1. **token 随 attempt 换**。每次 bump attemptNo 会换发新 token，旧 token 的 hash 被覆盖。所以即使 attempt 1 的迟到回调带着 token T1，hash 校验也会失败（库存已经是 T2 的 hash）。这给了"迟到丢弃"**双重保险**——attemptNo 校验和 token 校验，任意一个失败都会拒绝。
2. **状态机只前进不后退**。completed 是终态，不接受任何后续写入（包括合法的 attempt 2 重复回调——幂等丢弃）。
3. **idempotencyKey 全程不变**。无论 attempt 几次，idempotencyKey 都是同一个 K，代表"这是同一次意图"。状态机和 attemptNo 只是把这次意图的多次执行管起来。

---

## taskFingerprint 和 lifecycleMetadata：可观测性的补充

除了四道防线，模型里还有两个字段值得提：

- **`taskFingerprint`（任务指纹）**。对任务的输入（指令、参数、目标）算一个 hash。它和 partial index 一起，决定了"什么算同一个任务"（用于快速查可复用的成功 session）。也用于事后排查——两个看似不同的指令如果 fingerprint 一样，说明它们逻辑上等价。
- **`lifecycleMetadata`（Json）**。生命周期里那些"杂七杂八但很重要"的元数据：派发时的时间戳、每次 attempt 的开始/结束、回调到达的原始 payload……因为格式不固定，用 Json 存。这是把"结构化字段 + 半结构化元数据"分层的常见做法——核心字段建索引查询，细节塞 Json 留痕。

**经验：关键实体一定要有"留痕"字段。** 你不知道未来会出什么诡异问题，但出问题的时候，手里有没有元数据决定了你是一个小时查清还是三天查清。

---

## 业界对比：幂等的几种形态

编码任务这套幂等设计，本质是组合了分布式系统里几种经典的幂等模式。横向对比一下：

| 模式 | 作用 | 在 CodingTask 里的体现 |
|------|------|----------------------|
| 幂等键（idempotency key） | 防重复提交 | `(workstationId, triggerTool, idempotencyKey)` running 态去重 |
| 版本号/attempt 号 | 防过时覆盖 | `attemptNo` 单调递增，拒绝更小的写入 |
| 一次性令牌（one-time token） | 防伪造回调 | `callbackTokenHash`，hash 存储，随 attempt 换发 |
| partial index | 条件索引（非唯一） | 只对 completed+has_session 建索引，高效圈定可复用 session 候选 |
| 单调状态机 | 防终态被回滚 | completed/failed 终态不接受写入 |

这五种模式不是 WinMatrix 发明的，都是分布式系统的老套路。**值得说的是"把它们组合用"这件事本身**——很多系统只做了一两种（比如只加了个 idempotency key 就觉得幂等搞定了），结果上线后被迟到回调、被伪造回调、被过时覆盖各坑一次。**幂等不是一个开关，是一个体系。任何一种模式缺席，都会在某个边界场景爆雷。**

---

## 给后来者的总结

1. **幂等键只在 running 态去重，不是全局唯一**。终态的任务允许用相同参数重新发起，符合用户语义。幂等键的语义要和"这是同一次意图还是同一份参数"对齐。
2. **attemptNo 单调递增，拒绝过时回调**。迟到的旧 attempt 回调不能覆盖新状态——这是分布式重试里最阴险的坑，用一个数字比较就能挡住。
3. **回调令牌 hash 存储，随 attempt 换发**。双重保险：attemptNo 校验 + token 校验，任意一个失败都拒绝。防御深度，别假设任何一层绝对安全。
4. **partial index 圈定可复用候选**。一个 partial index（非唯一）只对"成功完成且带 session"的任务建索引，让系统快速查到可复用的 session 候选；真正复不复用由应用层判断。注意它不是 unique 约束——目的是"快速找候选"而非"禁止重复"。
5. **核心字段建索引，细节塞 Json 留痕**。taskFingerprint + lifecycleMetadata 这对组合，一个定身份一个存细节，是可观测性的标配。
6. **幂等是体系不是开关**。idempotency key / attemptNo / callbackToken / partial index / 单调状态机，五种模式缺一不可。只做一两种的"幂等"，迟早会被边界场景打穿。
7. **状态机只前进不后退**。终态不接受写入，连合法的重复回调都幂等丢弃。状态机越严格，系统越不容易被"特殊情况"搞乱。

编码任务的幂等和回调，看起来是个"数据库问题"，本质是个**并发与时序**问题——多个 attempt、多次回调、多次重试，它们在时间上交错到达，而你必须保证最终状态是"最新的、正确的"。五道防线，每一道都防一类交错，缺一道就是一个未来的事故。

---

> **下一篇**：[《MCP 双向架构：一个平台怎么既当 Server 又当 Consumer》](./13-mcp-bidirectional.md)——编码任务讲完了，我们把视角拉回平台本身。MCP（Model Context Protocol）让 LLM 能调用外部工具，但 WinMatrix 不只是消费 MCP——它同时把内部能力暴露成 MCP Server。这种双向架构怎么设计。
