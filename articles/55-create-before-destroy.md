# 先建新后关旧：热替换的节奏

> 这是《WinMatrix 开发经验文集》第 55 篇。上一篇讲透明代理，最后留了个钩子："换资源会把透明的假设彻底打破。"这一篇就讲"换资源"的正确节奏。任何做过热更新、滚动部署、连接池重建的人都知道一个反直觉的事实——**切换的瞬间是最危险的时刻，而降低风险的唯一办法是把切换从一个"动作"拆成两个有序的动作：先把新的建好能用，再关掉旧的。** 这听起来像废话，但它的反面（先关旧再建新，或同时进行）几乎是所有切换事故的根因。这一篇从 WinMatrix 的连接池重建、滚动部署、配置热更新里提炼这条节奏，看它怎么贯穿完全不同的场景。

先讲一个经典事故。

某次系统升级，运维同学为了"干净利落"，先把旧版本的 Pod 缩到 0，再启动新版本的 Pod。结果新 Pod 启动失败（一个配置项没配对），整个系统断服 20 分钟。事后复盘，根因不是配置错误（那个迟早会修），而是**切换顺序**——先关旧再建新，导致切换窗口期没有任何实例可用。

这个事故的解法是一句听起来很废话的话：**应该先建新，等新能用，再关旧。** 但这句话之所以不是废话，是因为它在工程上的实现远比字面复杂——它要求系统能容忍"新旧并存"、能检测"新是否真的能用"、能在"新确认能用后"才触发"关旧"。这三件事每一件都不简单。

WinMatrix 在多个完全不同的场景里实现了这条节奏：连接池重建、滚动部署、配置注册。这一篇把它们拎出来，看同一条哲学怎么落地成三套不同的代码。

---

## 场景一：Prisma 连接池重建——publish 先于 close

上一篇讲过，WinMatrix 的 Prisma Client 遇到可恢复错误会触发重建。这里要讲的是**重建内部的顺序**，它就是"先建新后关旧"最精确的代码体现（`src/infrastructure/persistence/prisma/client.ts:193-221`，核实报告 ch04-06）：

```ts
async function rebuildPrismaResources(reason: unknown): Promise<void> {
  if (globalForPrisma.prismaRebuildInFlight) {
    return globalForPrisma.prismaRebuildInFlight;   // single-flight：并发只重建一次
  }
  globalForPrisma.prismaRebuildInFlight = (async () => {
    const previousResources = getRuntimeResources();     // [1] 拿到旧资源引用
    const nextResources = createPrismaResources();        // [2] 建新资源（新 pg.Pool + 新 PrismaClient）
    runtimeResources = nextResources;                     // [3] 新资源上线（publish）
    publishPrismaResources(nextResources);                //     全局可见
    await closePrismaResources(previousResources);        // [4] 最后才关旧资源
  })();
  try { await globalForPrisma.prismaRebuildInFlight; }
  finally { globalForPrisma.prismaRebuildInFlight = undefined; }
}
```

注意四步的顺序：

```
[1] previousResources = getRuntimeResources()    // 拿旧引用（不关）
[2] nextResources = createPrismaResources()      // 建新（此时旧还在跑）
[3] runtimeResources = nextResources             // 新上线，后续请求走新
    publishPrismaResources(nextResources)        // 全局 publish
[4] closePrismaResources(previousResources)      // 最后才关旧
```

**关键点是 [3] 和 [4] 的顺序绝不能反。** 如果先 `close` 旧的再 `publish` 新的，那 close 到 publish 之间进来的请求会拿到一个已经关闭的 Pool，直接报错。而现在的顺序保证了：**任何时刻 `getRuntimeResources()` 拿到的资源都是可用的**——要么是旧的（重建中），要么是新的（重建后），绝不是一个"关了但新的还没上"的空窗。

这是"先建新后关旧"最纯粹的形态。它的本质是：**切换不是一个原子动作，而是一个有明确阶段的过程，过程中系统始终有可用资源。**

再注意 single-flight 的配合：`globalForPrisma.prismaRebuildInFlight` 保证并发请求同时遇到错误时，只会有一个重建在进行，其他请求等同一个 Promise。否则 10 个请求同时触发重建，10 个都"先建新后关旧"，系统里瞬间出现 10 个新 Pool，资源爆炸。**single-flight 是"先建新后关旧"能正确执行的前提之一**——没有它，这条节奏会被并发打乱。

---

## 场景二：滚动部署——新 Pod 先起，旧 Pod 后删

把同样的节奏放大到集群级别，就是 K8s 的滚动部署。WinMatrix 的部署配置（`k8s/deployment.yaml`，核实报告 ch23-29）遵循的就是这条节奏：

```
滚动部署（RollingUpdate）：
  1. 启动新 Pod（新版本）
  2. 新 Pod readiness 探针通过（/readyz，initialDelay 60s）
  3. 新 Pod 加入 Service 端点，开始接收流量
  4. 才开始终止旧 Pod（旧 Pod 先收到 SIGTERM，进入优雅停机）
  5. 旧 Pod 处理完在途请求后退出
```

这里"先建新后关旧"体现在两个机制上：

**第一，readiness 探针决定摘流，但不决定重启。** WinMatrix 用了两个不同的探针（核实报告 ch23-29）：

| 探针 | 路径 | 作用 | 失败后果 |
|------|------|------|---------|
| liveness | /health | 决定是否重启 Pod | 重启 |
| readiness | /readyz | 决定是否摘流（不路由流量给该 Pod） | 摘流，**不重启** |

readiness 失败只是"这个 Pod 暂时不接流量"，旧 Pod 还在跑。这保证了：**新 Pod 还没 ready 时，旧 Pod 继续服务；新 Pod ready 后，流量才逐步切过去，旧 Pod 才开始退。** 如果 readiness 失败就重启，那就退化成了"先关旧再建新"的危险模式。

**第二，fatal 靠应用自退出，而不是探针杀死。** 这是一个更深的细节。WinMatrix 的设计是：致命错误（比如连不上 DB、配置加载失败）由应用自己 `process.exit(1)` 退出，而不是靠 liveness 探针发现后杀死。原因是探针杀死有个延迟（探针间隔 × 失败次数），这段时间应用可能在错误状态下继续接流量；而应用自退出是立刻的。**但这违反了"先建新后关旧"吗？不违反——应用自退出后，K8s 会发现 Pod 挂了，启动新 Pod，而此时其他旧 Pod（如果是多副本）还在服务。** 它针对的是"这个 Pod 自己已经没救了"的场景，前提是有其他 Pod 兜底。

**这两个机制共同保证了部署期间始终有可用实例。** 这是"先建新后关旧"在集群级别的体现：节奏一样，但落点是 Pod 生命周期管理。

---

## 场景三：配置热更新——先注册监听，再读状态

第三个场景更微妙，它不是"换资源"，而是"初始化顺序"。WinMatrix 的配置热更新走 PG LISTEN/NOTIFY（`ConfigDbListener.ts`，核实报告 ch04-06）。它的启动顺序也遵循"先建新后关旧"的精神——只不过这里是"先建监听，再读当前状态"：

```
ConfigDbListener 启动顺序（简化）：
  [1] connect()：建立独立 PG 连接，LISTEN config_change
  [2] 加载当前配置到内存（首次全量读 DB）
  [3] scheduleDebounce()：准备好防抖机制
  [4] 进入正常监听循环
```

为什么 [1] 必须在 [2] 之前？想象反过来：先读当前配置，再注册 LISTEN。那读配置到注册 LISTEN 之间，如果有配置变更发生，这个变更的通知你收不到（还没 LISTEN 呢），你的内存配置就停留在旧值，**直到下次变更才被动刷新**。这就是"先读状态后建监听"留下的空窗。

**正确顺序是先 LISTEN 再读**：即使读的瞬间有变更，那个变更的通知你已经在监听了，会在防抖后触发刷新。这个模式有个通用名字叫 **read-after-subscribe**（先订阅再读），它是所有"事件驱动 + 状态缓存"系统的正确初始化姿势。

这和"先建新后关旧"是同一个思想的不同表达：**不要留下"动作之间的空窗"，所有可能导致状态不一致的空窗都要靠顺序消除。**

```
"先建新后关旧"      切换资源时，始终有可用资源
"先订阅再读"        初始化监听时，不漏掉任何变更
        ↓ 同一个思想 ↓
   有序的两个动作，消除动作之间的空窗
```

---

## 这条节奏为什么反直觉

"先建新后关旧"听起来像常识，为什么实际工程里总有人违反它？因为它有三个反直觉的地方。

**第一，它要求系统能容忍"新旧并存"。** 先建新后关旧，意味着有一段时间新旧同时存在。如果系统设计成"只能有一个实例"（比如单进程、独占锁、单例），你就被迫"先关旧再建新"。WinMatrix 的 Prisma 重建之所以能"先建新后关旧"，是因为 `runtimeResources` 是个引用变量，新旧可以同时存在于内存里（旧的等被 close，新的已经 publish）。**要落地这条节奏，系统架构本身必须支持多实例并存。**

**第二，它要求能判断"新是否真的能用"。** 光建新不够，还要等新能用才关旧。Prisma 重建里"能用"就是 `createPrismaResources()` 返回成功；K8s 部署里"能用"是 readiness 探针通过。**如果判断不准（比如 readiness 探针太宽松，新 Pod 还没真正 ready 就标记为 ready），就会退化成"关了旧但新其实不能用"的事故。** 这也是为什么 WinMatrix 把 liveness 和 readiness 严格分开——readiness 必须真正反映"能不能服务"，不能含糊。

**第三，它要求"关旧"本身是优雅的。** 旧资源不能直接 kill，要给它时间处理完在途请求。Prisma 重建里 `closePrismaResources` 会等连接池里的连接自然归还；K8s 部署里旧 Pod 收到 SIGTERM 后进入优雅停机（释放连接、归还租约、flush pending）。**如果"关旧"是暴力的，那即使顺序对了，在途请求也会丢。** （第 64 篇会专门讲优雅停机的细节。）

---

## 节奏错了会怎样：三种典型事故

把违反这条节奏的三种典型事故列出来，反向印证它的重要性：

**事故一：先关旧再建新（空窗期断服）。** 就是开头讲的那個运维事故。切换窗口期没有任何实例，用户直接 502。这是最直接、损失最大的一种。

**事故二：同时关旧和建新（竞争导致不一致）。** 某些"聪明"的实现为了快，并发执行"关旧"和"建新"。结果两者竞争资源（比如端口、文件锁），可能出现新的建不起来、旧的已经关了，或者新旧状态打架。WinMatrix 的 single-flight 重建就是防这个——**重建必须串行，不能并发。**

**事故三：先读状态后订阅（漏掉变更）。** 配置初始化时先读 DB 再 LISTEN，读和订阅之间的变更永久丢失。这种事故最难发现——系统看起来正常，只是某个配置"莫名其妙没生效"，排查很久才发现是初始化时漏了一条通知。

这三种事故的共性是：**都发生在"动作之间的空窗"里。** "先建新后关旧"的全部价值，就是消除这个空窗。

---

## 节奏贯穿完全不同的场景

把三个场景放一起，最值得品味的是：**同一条节奏（有序的两个动作，消除空窗）贯穿了完全不同的抽象层级。**

| 场景 | 抽象层 | "建新" | "关旧" | 保证 |
|------|-------|--------|--------|------|
| Prisma 重建 | 进程内对象 | createPrismaResources + publish | closePrismaResources | 任意时刻 getRuntimeResources 都可用 |
| K8s 滚动部署 | 集群 Pod | 启动新 Pod + readiness 通过 | SIGTERM 旧 Pod + 优雅停机 | 部署期间始终有 ready 实例 |
| 配置热更新初始化 | 事件订阅 | LISTEN config_change | 加载当前配置 | 不漏掉任何变更通知 |

这是工程哲学之所以为"哲学"的原因——它不绑定某个技术、某个框架、某个语言，它是对"怎么管理状态切换"的本质约束。无论你换的是对象、是 Pod、还是订阅状态，只要存在"切换"，就要遵循"先建新后关旧"。

---

## 给后来者的总结

1. **切换不是一个原子动作，而是有明确阶段的过程。** 阶段之间不留空窗，是"先建新后关旧"的全部精髓。
2. **顺序永远是：建新 → publish 新 → 关旧。** 绝不能反过来。Prisma 重建里 `publishPrismaResources` 必须在 `closePrismaResources` 之前。
3. **系统架构必须支持新旧并存。** 如果你的设计是"只能有一个实例"，你就被迫走危险的"先关旧再建新"。WinMatrix 用引用变量（runtimeResources）让新旧能同时存在于内存。
4. **并发切换要 single-flight。** 否则 N 个并发请求各建一份新资源，资源爆炸。Prisma 重建用 `globalForPrisma.prismaRebuildInFlight` 保证全局只重建一次。
5. **readiness 和 liveness 要分开。** readiness 决定摘流（不重启），liveness 决定重启。readiness 失败就重启会退化成"先关旧再建新"。
6. **fatal 靠应用自退出，不靠探针杀死。** 探针杀死有延迟，应用自退出是立刻的。但前提是有其他 Pod 兜底（多副本）。
7. **事件订阅的初始化用"先订阅再读"。** 先 LISTEN 再读当前状态，否则读和订阅之间的变更永久丢失。这和"先建新后关旧"是同一个思想的不同表达。
8. **"关旧"必须优雅。** 直接 kill 旧资源会丢在途请求。给旧资源时间处理完手头的事（归还连接、flush pending、释放租约）。

切换的瞬间是系统最脆弱的时刻。把切换拆成有序的两个动作、消除空窗，是把脆弱变成稳健的最朴素也最有效的方法。这条节奏没有技术含量，但它区分了"能上生产"和"上生产就出事"的团队。

---

> **上一篇**：[《透明代理并非透明：PgBouncer/连接池/中间件的隐含状态》](./54-transparent-proxy.md)
>
> **下一篇**：[《横切关注点别靠纪律：Proxy/中间件/Hook 收口》](./56-cross-cutting-concerns.md)——切换的节奏保证了不空窗，但有一类问题比切换更普遍——横切关注点。下一篇讲为什么鉴权、限流、连接恢复、观测这些"每个请求都要做"的事，不能靠程序员每次记得写，必须用 Proxy/Hook 对业务代码透明地收口。
