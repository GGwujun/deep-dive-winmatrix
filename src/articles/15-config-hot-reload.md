# 配置热更新：为什么我们用 PG LISTEN/NOTIFY 而不是 Redis Pub/Sub

> 这是《WinMatrix 开发经验文集》第 15 篇。这一篇讲一个很基础设施、但每个平台都要面对的问题：**配置改了，怎么不重启就生效？** 也就是"热更新"。

做过服务端的人都知道这个尴尬：你在管理后台改了个配置（比如某个技能的开关、某个角色的权限），点了保存，然后……什么都没发生。因为代码里读的是启动时加载的配置，改了数据库不重启进程，内存里还是老值。

最粗暴的解法是"改完重启服务"。但生产环境重启一次有成本——连接断开、请求中断、冷启动延迟。而且有些配置改动很频繁（限流阈值、灰度开关），总不能每次都重启。

于是你需要**热更新**：数据库改了，运行中的进程能感知到，刷新内存里的缓存，不用重启。

热更新的技术选型不少：轮询（定时查库）、Redis Pub/Sub、消息队列、PG LISTEN/NOTIFY……WinMatrix 选的是 **PG LISTEN/NOTIFY**。这一篇就讲为什么这么选，以及实现里那些"不踩不知道"的坑。

---

## 先看选项：为什么不是轮询、不是 Redis Pub/Sub

把几个选项摆上台面，逐个排除。

### 轮询（polling）

最简单的：每隔 N 秒查一次数据库，看配置变没变。

问题：

- **延迟高**。改了配置到生效，最坏要等一个轮询周期（比如 30 秒）。对灰度开关这种场景，30 秒延迟很难受。
- **无效查询多**。配置大部分时间不变，99% 的轮询是"查了发现没变"——纯浪费数据库资源。
- **实时性和成本互相矛盾**。想低延迟就缩短周期（比如 1 秒），但 1 秒查一次数据库，压力上去了。

轮询适合"低频 + 不在乎延迟"的场景。配置热更新不太合适。

### Redis Pub/Sub

这是很多人会想到的方案——WinMatrix 本来就用了 Redis（BullMQ、缓存都依赖它），顺手用 Redis Pub/Sub 推配置变更通知，看起来很自然。

但 Redis Pub/Sub 有个致命特性：**消息不持久化，发出去没人收就丢了。**

场景还原：你改了配置，系统往 Redis 发了个通知。但此时你的应用进程正好在重启（比如刚发了个版本），没订阅上这个频道——通知发了就没了。进程重启完后，永远不知道刚才有过变更，内存里还是老的脏配置。

这就是"at-most-once"投递——最多送一次，丢了就没了。对配置这种"绝对不能漏"的场景，at-most-once 是不可接受的。

当然可以加补偿（重启后全量刷一次配置），但那就把"增量推送"的优点抵消了，逻辑也复杂。

### PG LISTEN/NOTIFY

PostgreSQL 原生的发布订阅机制。它有几个特性正好匹配配置热更新的需求：

- **推送式**（不轮询）：数据库一变，立即通知，延迟低。
- **和写操作同源**：配置写到 PG，通知也从 PG 发，天然一致——不会出现"数据库写成功但通知没发"的情况（因为通知就在同一个事务里）。
- **不依赖额外中间件**：WinMatrix 的配置本来就存在 PG，LISTEN/NOTIFY 是 PG 自带功能，零额外依赖。

缺点是它也有"断连期间消息丢失"的问题（和 Redis 一样是无持久化的 pub/sub）。但这一点可以通过"重连后全量刷新"来补偿——而这个补偿逻辑天然就有（进程启动时本来就要全量加载一次配置）。后面会讲。

权衡下来，PG LISTEN/NOTIFY 胜出——**因为配置的 SSOT（single source of truth）就是 PG，让通知也从 PG 发，是把"写"和"通知"绑在同一个事务边界内，一致性最强。**

---

## 发送端：一行 pg_notify

发送端的实现极简：

```typescript
// infrastructure/persistence/database/notifyConfigChange.ts（第 1-14 行）
export async function notifyConfigChange(payload: ConfigChangeNotifyPayload): Promise<void> {
  await prisma.$executeRaw`SELECT pg_notify('config_change', ${JSON.stringify(payload)})`;
}
```

就是调 PG 的 `pg_notify` 函数，往 `config_change` 频道发一条消息，内容是 payload 的 JSON。

这段代码通常写在"配置更新"的业务逻辑里——你改了配置，更新数据库，然后调 `notifyConfigChange` 通知。**写在同一个事务里**，保证"配置真写进去了"和"通知真发出去了"原子一致。这是 PG LISTEN/NOTIFY 相比 Redis 的核心优势——Redis 的通知和 PG 的写入是两个系统，没法做事务一致。

---

## 接收端：ConfigDbListener 的四个工程细节

接收端才是难点。`ConfigDbListener`（414 行）的实现里，有四个"不踩不知道"的工程细节。

### 细节一：必须用独立的 pg.Client，不能走 PgBouncer

这是最容易踩坑、也是最重要的一点。

WinMatrix 的生产环境用了 PgBouncer 做连接池（transaction-pooling 模式，5000 人规模下避免 too many clients）。但 **LISTEN/NOTIFY 和 PgBouncer transaction-pool 天然不兼容**。

为什么？因为 PgBouncer 的 transaction-pool 模式下，每个事务可能跑在不同的后端连接上——你在这个连接上 `LISTEN`，下一个事务被路由到另一个连接，那个连接根本没 `LISTEN` 过，通知收不到。

LISTEN 是**会话级**的状态，它绑定在具体的后端连接上。你必须在**同一个连接**上 LISTEN，才能收到那个连接上的通知。transaction-pool 打破了"同一个连接"这个前提。

所以 ConfigDbListener 的约束写得很明确：

```typescript
// ConfigDbListener.ts（第 90-112 行，约束）
// 必须走真实 PG 会话，不能经 PgBouncer transaction pooling
// 优先取 DATABASE_LISTEN_URL（直连 PG 的连接串）
```

它用**独立的 `pg.Client`**（不是连接池，不是 Prisma），专门建一条到 PG 的直连，在这条连接上 `LISTEN config_change`。这条连接长期独占，只干一件事：收通知。

配置上还支持单独的 `DATABASE_LISTEN_URL`——因为生产环境的应用默认连 PgBouncer（DATABASE_URL 指向 PgBouncer），你需要一个单独的、直连 PG 的连接串给 LISTEN 用。**这是生产部署的强制要求，不是可选项。**

**教训：凡是依赖"会话状态"的 PG 特性（LISTEN/NOTIFY、advisory lock、临时表、SET 命令），都不能走 transaction-pool 的 PgBouncer，必须直连。** 这个坑踩过一次就忘不了。

### 细节二：500ms 防抖，合并短时间内的密集通知

配置变更有个特点：**往往是成批的**。管理员在后台一次改 5 个配置项，或者某个批量操作触发了一连串变更，瞬间会涌来多条 LISTEN 通知。

如果每条通知都触发一次缓存刷新，就是"通知风暴"——5 条通知刷 5 次缓存，而实际上刷一次就够了（最后一次刷新涵盖所有变更）。

解法是**防抖（debounce）**：

```typescript
// ConfigDbListener.ts（第 241-271 行）
private handleNotification(msg: pg.Notification): void {
  if (msg.channel !== 'config_change' || !msg.payload) return;
  if (this.notificationSuppressed) return;
  try {
    const payload: ConfigChangePayload = JSON.parse(msg.payload);
    const key = `${payload.type}:${payload.id}`;
    this.pendingChanges.set(key, {
      configType: payload.type, configId: payload.id,
      action: payload.action, timestamp: payload.timestamp,
    });
    this.scheduleDebounce();
  } catch (error) { logger.warn(`...`); }
}
```

```typescript
// ConfigDbListener.ts（第 336-345 行）
private scheduleDebounce(): void {
  // 500ms 防抖：500ms 内的通知合并成一次刷新
}
```

两步走：

1. **收到通知先放进 `pendingChanges` Map**（不立即处理），key 是 `${type}:${id}`。同一配置的多次变更，后到的覆盖先到的（Map 自动去重）。
2. **调 `scheduleDebounce()`**：设一个 500ms 的定时器。500ms 内再来通知，重置定时器。定时器到期了（500ms 没有新通知），才真正批量刷新一次。

效果：5 条通知在 100ms 内涌来，只会触发**一次**刷新（在最后一条通知后的 500ms）。通知风暴被压成一次刷新。

**防抖是处理"密集事件流"的标准武器。** 不只配置热更新，前端输入框、滚动事件、窗口 resize——任何"短时间密集触发但只需处理最终态"的场景，都用防抖。

### 细节三：Map 去重，同配置只保留最新

注意上面 `pendingChanges.set(key, ...)`——key 是 `${type}:${id}`。这意味着：同一个配置（比如"角色阿码的技能开关"）如果在防抖窗口内变了 3 次，Map 里只保留最后一次。刷新时只处理这一个最新值。

这是**去重 + 取最新**的语义。比"处理每次变更"高效得多——因为中间值没意义（你只关心最终的开关状态是开还是关，不关心它中间抖了几次）。

### 细节四：重连的指数退避

LISTEN 连接是长连接，它会断（网络抖动、PG 重启、PgBouncer 维护）。断了必须重连。重连的姿势很讲究：

```typescript
// ConfigDbListener.ts（第 296-331 行）
private scheduleReconnect(): void {
  // 指数退避：base 1s，封顶 30s
}
```

指数退避（1s → 2s → 4s → 8s → 16s → 30s 封顶）——第一次重连等 1 秒，失败再等 2 秒，再失败 4 秒……封顶 30 秒。这比"固定间隔重试"好，因为：

- 如果 PG 真的挂了，固定间隔（比如每秒）会疯狂重试，浪费资源。
- 指数退避在持续失败时自动"退让"，减轻 PG 的压力。
- 封顶避免退避到无穷大（30 秒重试一次是底线，不能等太久）。

重连成功后有个**关键补偿**：全量刷新一次配置。因为断连期间漏掉的通知没法找回（LISTEN/NOTIFY 不持久化历史），只能假设"断连期间一切都可能变了"，全量刷一遍才能保证内存配置是最新的。

这个补偿逻辑和"进程启动时全量加载"是同一个动作——都是"不确定当前状态，全量重建"。**把"启动"和"重连"都当成"全量刷新"的触发点，逻辑统一。**

### 补充：bulk 写入期间暂停通知

还有个小细节（`setNotificationSuppressed`，第 194-204 行）：当系统自己在做批量配置写入（比如批量导入配置）时，会临时**暂停**处理通知。因为这些写入产生的通知，本来就是要被防抖合并掉的，但批量写入时通知量极大，干脆直接暂停，写完再全量刷一次。这是对防抖的补充——防抖处理"中等密度"，suppressed 处理"极高密度"。

---

## 启动约束：不是每个进程都启 Listener

最后说一个部署层面的细节。ConfigDbListener 的启动是有条件的：

```typescript
// startup/common.ts（第 331-358 行）
// STARTUP_SKIP_CONFIG_DB_LISTENER=true 可跳过
// rag 进程也跳过
```

- **`STARTUP_SKIP_CONFIG_DB_LISTENER=true` 可跳过**。有些场景（比如一次性脚本、特殊运维）不需要热更新，跳过省一条 PG 连接。
- **rag 进程跳过**。WinMatrix 有三个进程（api / scheduled / rag），rag 进程只做向量检索，不消费业务配置，不需要热更新。

**不是每个进程都需要所有功能。** 按进程角色裁剪启动逻辑，省资源、降复杂度。这是 [第 4 章] 讲的"进程角色守卫"思想在配置热更新上的延伸。

---

## 业界对比：配置中心 vs PG LISTEN/NOTIFY

横向对比一下，业界的配置热更新还有一条路：**专门的配置中心**（Apollo、Nacos、Consul KV、etcd）。它们提供更完整的配置管理能力——版本历史、灰度发布、多环境管理、专门的 dashboard。

WinMatrix 为什么没用配置中心？

- **配置的 SSOT 已经是 PG**。所有业务配置（技能、角色、工具、项目）都在 PG 的表里，有完整的关系模型。再引入一个配置中心，等于"两套配置源"，同步是噩梦。
- **LISTEN/NOTIFY 够用**。WinMatrix 的热更新需求是"配置改了能生效"，不需要灰度发布、版本回滚那些高级功能（那些是配置中心的强项）。
- **少一个中间件**。配置中心是额外的运维负担——部署、监控、备份。PG LISTEN/NOTIFY 是 PG 自带的，零额外成本。

**选型的核心是"够用就好"，不是"越强越好"。** 配置中心很强，但它的强项（灰度、版本、多环境）对 WinMatrix 当前阶段不是刚需；而它的代价（多一套 SSOT、多一个中间件）却是实打实的负担。PG LISTEN/NOTIFY 在"配置已在 PG"这个前提下，是性价比最高的选择。

当然，如果配置规模大到需要灰度发布、需要独立的配置治理团队，那配置中心就值得引入了。**技术选型没有绝对的对错，只有"当前阶段的匹配度"。**

---

## 给后来者的总结

1. **配置热更新选 PG LISTEN/NOTIFY 的核心理由**：配置的 SSOT 就是 PG，让通知和写入在同一个事务边界内，一致性最强。Redis Pub/Sub 是 at-most-once，断了就丢。
2. **LISTEN/NOTIFY 必须直连 PG，不能走 PgBouncer transaction-pool**。LISTEN 是会话级状态，transaction-pool 会把后续事务路由到别的连接，收不到通知。生产环境必须配 `DATABASE_LISTEN_URL`。
3. **用独立的 pg.Client，不用连接池**。LISTEN 连接长期独占，只干收通知一件事。
4. **500ms 防抖合并密集通知**。配置变更往往成批，防抖把通知风暴压成一次刷新。
5. **Map 去重取最新**。同一配置的多次变更只保留最后一次，中间值无意义。
6. **重连用指数退避（base 1s，封顶 30s），重连后全量刷新**。断连期间漏掉的通知找不回，只能全量重建。把"启动"和"重连"统一成"全量刷新"触发点。
7. **bulk 写入期间暂停通知**。极高密度下防抖不够，直接暂停 + 写完全量刷。
8. **按进程角色裁剪启动**。不是每个进程都启 Listener，rag 进程和特殊运维场景跳过。
9. **配置中心 vs LISTEN/NOTIFY 是匹配度问题**。配置已在 PG、不需要灰度版本时，LISTEN/NOTIFY 性价比最高；需要灰度/多环境/独立治理时，配置中心值得引入。

配置热更新看起来是个小功能，它考验的是"你对基础设施的理解深度"——PgBouncer 的 pooling 模式、LISTEN 的会话绑定、防抖的去重时机、断连的补偿策略……每一个点都是踩过坑才知道的。把这些坑都填上，热更新才能真正"热"起来。

---

> **下一篇**：[《定时任务系统：16 个系统任务的幂等与补偿》](./16-scheduled-task-idempotency.md)——配置能热更了，但后台还有一堆定时任务（清理、同步、巡检……）要跑。这些任务怎么不撞车、崩了怎么自愈、结果怎么投递——是最后一篇的主题。
