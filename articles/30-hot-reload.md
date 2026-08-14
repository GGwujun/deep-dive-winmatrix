# 热更新与零停机：改配置不重启、发技能不中断、灰度路由

> 这是《WinMatrix 开发经验文集》第 30 篇，横切主题的最后一篇。企业系统有一个绕不开的诉求——不能停机。但 AI 平台的配置（角色提示词、技能定义、路由规则）又特别频繁地变。这篇讲我们怎么用三套独立的热更新机制，把"改东西"和"重启进程"解耦。

传统后端系统改配置，流程是：改配置文件 → 提交 → CI/CD 构建 → 部署 → 进程重启 → 生效。这个流程在低频变更的场景下没问题——一周改一次配置，重启一次无所谓。

但 AI 平台不是这个节奏。WinMatrix 在生产里，一天可能有几十次配置变更：

- 产品经理发现小品的提示词要微调一句——改 agent_config。
- 运维要给阿码加一个新工具的权限——改 project_tool_policy。
- 要新增一条路由规则"凡是提到'周报'的走阿宁"——改 route_rule。
- 发了一个新版本的 prd_writing 技能——更新 skill_artifact。

如果每一次变更都要重启进程，系统一天要重启几十次。每次重启意味着正在进行的对话被打断、正在跑的编码任务被丢弃、正在排队的工作被延迟。这在企业场景里是不可接受的。

我们的解法是三套独立的热更新机制，分别管配置、技能、路由规则。它们的设计哲学一样——**改 DB，不重启；但都有一套从 DB 到内存的同步管道**。

---

## 第一套：配置热更新——PG LISTEN/NOTIFY

最基础、最频繁的是配置热更新。数字员工的身份、提示词、工具白名单、技能绑定，都存在 DB 的多张表里。改了 DB，怎么让运行中的进程立刻感知？

### 为什么选 PG LISTEN/NOTIFY 而不是 Redis Pub/Sub

这是项目里一个被反复问到的设计决策。我们没有用 Redis Pub/Sub 做配置变更广播，而是直接用了 PostgreSQL 原生的 LISTEN/NOTIFY 机制。

发送端非常简单，一句 SQL：

```typescript
// infrastructure/persistence/database/notifyConfigChange.ts（第 1-14 行）
export async function notifyConfigChange(payload: ConfigChangeNotifyPayload): Promise<void> {
  await prisma.$executeRaw`SELECT pg_notify('config_change', ${JSON.stringify(payload)})`;
}
```

任何一次配置写入（比如更新 agent_config），事务提交后顺手 `pg_notify('config_change', ...)`，把变更类型和 ID 推出去。

接收端是 ConfigDbListener，核心约束在它的注释里——**必须走真实 PG 会话，不能经 PgBouncer transaction pooling**：

```typescript
// agents/core/kernel-management/config/listener/ConfigDbListener.ts（第 90-112 行）
// 约束：必须走真实 PG 会话，不能经 PgBouncer transaction pooling
// 优先取 DATABASE_LISTEN_URL
```

为什么这个约束这么硬？因为 PgBouncer 在 transaction pooling 模式下，每个事务结束就归还连接，LISTEN 是会话级的——你在一条连接上 LISTEN 了，连接被归还后再被别人拿走，NOTIFY 就找不到你了。**LISTEN/NOTIFY 和连接池 transaction 模式天然不兼容。** 我们的做法是给 ConfigDbListener 单独开一条 `DATABASE_LISTEN_URL`，绕过 PgBouncer 直连 PG。

这恰恰是"为什么不用 Redis Pub/Sub"的答案——不是 Redis 不能做，而是**配置的真源本来就在 PG 里**。用 PG 原生的 LISTEN/NOTIFY，省掉了一次"PG → Redis → 进程"的中转，少一个组件、少一次一致性风险。配置写入和通知发送在同一个事务里，要么都成功要么都失败，不会出现"DB 改了但通知没发"的割裂。

### 独立 pg.Client + 500ms 防抖 + Map 去重

ConfigDbListener 的连接、去重、防抖三个设计值得一说。

**独立 pg.Client**。它不用连接池，而是一个独立的 `pg.Client`（见 `connect()` 第 209-236 行）。原因是 LISTEN 是长连接语义——这条连接必须一直活着，才能持续收到 NOTIFY。连接池会回收连接，不适合。

**500ms 防抖**。一次配置变更可能触发多条 NOTIFY（比如批量更新多个角色）。如果每条都立刻处理，会造成"抖动"——刚处理完 A，又来一条 A，又处理一遍。我们的做法是收到 NOTIFY 后放进 `pendingChanges` Map，等 500ms 没有新通知了再批量处理：

```typescript
// ConfigDbListener.ts（第 336-345 行）
private scheduleDebounce(): void {
  // 500ms 防抖合并
  if (this.debounceTimer) return;
  this.debounceTimer = setTimeout(() => {
    this.flushPendingChanges();
    this.debounceTimer = null;
  }, 500);
}
```

**Map 去重**。同一个配置项在防抖窗口内被改多次，只处理最后一次。`pendingChanges` 是个 Map，key 是 `${type}:${id}`，后到的覆盖先到的：

```typescript
// ConfigDbListener.ts（第 241-271 行）
private handleNotification(msg: pg.Notification): void {
  if (msg.channel !== 'config_change' || !msg.payload) return;
  if (this.notificationSuppressed) return;
  const payload: ConfigChangePayload = JSON.parse(msg.payload);
  const key = `${payload.type}:${payload.id}`;
  this.pendingChanges.set(key, { configType: payload.type, configId: payload.id, action: payload.action, timestamp: payload.timestamp });
  this.scheduleDebounce();
}
```

防抖 + 去重的效果是：哪怕管理员一口气改了 30 个角色配置，系统也只在最后一次变更后 500ms 统一刷新一次，而不是刷新 30 次。

### 热更新生效：Role 的 reloadFromConfig

配置刷新到内存后，怎么让已经在跑的数字员工实例生效？BaseRole 基类在初始化时注册了对 `configManager.onConfigChange` 的监听：

```typescript
// agents/core/worker/role/BaseRole.ts（第 251-280 行）
protected onInitialized() {
  // 注册 configManager.onConfigChange 监听（热重载）
  // 命中 affectedRoleIds 时调 reloadFromConfig()
}
```

`reloadFromConfig()` 会重新从 agent_config 表加载角色的 `name / profile / goal / constraints / nickname`，替换内存中的实例字段。正在运行的会话下次用到这个角色时，拿到的就是新配置。

这就是为什么"改小品的提示词"不需要重启——改的是 DB，DB 通过 LISTEN/NOTIFY 通知到进程，进程刷新内存里的 Role 实例，下一次对话自动用新提示词。**整个链路不涉及进程重启，也不涉及正在进行的对话被中断**（正在进行的 Turn 会用旧配置跑完，下一个 Turn 才用新配置——这是合理的"已开始的不打断"语义）。

---

## 第二套：技能热更新——Artifact Store + 哈希校验

配置热更新管的是"角色的身份和权限"，技能热更新管的是"技能本身的定义和产物"。

技能在 WinMatrix 里不是一个函数，而是一个完整的产物包（artifact）——提示词模板、Agent 封装、契约（provides/consumes）、凭证要求、可见工具集。这些打包成一个 artifact，存在 SkillArtifactStore 里：

```prisma
// prisma/schema.prisma（第 1273-1305 行）
model skill_artifact {
  name                  String
  version               String
  scope                 String
  project_name          String?
  trust_level           String?
  manifest              Json          ← 技能清单（提示词、契约、工具集）
  persona_eligible      Boolean       @default(true)   ← 分身是否默认继承
  package_storage_key   String?                            ← 产物在对象存储的 key
  package_sha256        String?                            ← 产物哈希
  @@unique([name, version, scope, project_name])
}
```

两个关键字段：`package_storage_key`（产物存在哪）和 `package_sha256`（产物的哈希）。

### 产物模式 vs 硬编码 bundled

技能热更新的前提，是技能本身是"可替换的产物"而不是"硬编码在代码里的函数"。这是个有教训的决策——早期 WinMatrix 的技能是 bundled 的（硬编码打包在代码里），后来发现这条路走不通：

> 技能要演进、要热更新、要从市场安装，bundled 路径走不通，最终重构为 artifact-store（技能产物存储）模式，bundled 被标为 deprecated。

bundled 模式的问题是：改一个技能要改代码、走 CI/CD、重启进程——这不叫"热更新"，叫"发版"。而 artifact 模式把技能变成一份带版本的产物，发新版本只是往 skill_artifact 表插一行新记录（version + 1），运行时通过 SkillRegistry.resolve() 动态解析最新版本。

### 哈希校验防篡改

`package_sha256` 不是装饰。技能产物从对象存储加载回来后，系统会算一遍哈希，和 DB 里记的对不上就拒绝加载。这是防"产物在传输或存储中被篡改"——技能的提示词里可能包含凭证要求、工具授权这些敏感信息，被篡改了后果严重。

哈希校验的另一层意义是**版本一致性**。`skillContentHash` 和 `toolSetHash` 还会被记录在 SkillExecGuide 里（见第 29 篇）——当技能内容变了（hash 变了），旧的执行指南自动失效，会触发重新蒸馏。这就把"技能热更新"和"蒸馏产物"串成了一条一致的链：技能变了 → hash 变 → 旧指南失效 → 重新蒸馏 → 新指南生效。**所有环节都靠 hash 保证版本对齐，不会出现"用了新技能但配着旧指南"的错配。**

### Schema 漂移：热更新的反面

热更新再好，也有一个绕不开的坑——Schema 漂移。技能定义会演进（加字段、改类型），但如果代码发了新版本、DB 迁移没跟上（migration 没跑），skill_artifact 表可能缺列。这时技能加载会出错。

我们的做法是三层漂移检测：

```typescript
// business/domain/skillManagement/skillArtifactSchemaDrift.ts（第 56-83 行）
export async function warnSkillArtifactSchemaDriftOnStartup(): Promise<void> {
  const rows = await prisma.$queryRaw`
    SELECT column_name FROM information_schema.columns
    WHERE table_schema = 'public' AND table_name = 'skill_artifact'
      AND column_name IN ('project_name','trust_level','manifest','enabled','installed_at','created_at','updated_at')`;
  const present = new Set(rows.map((row) => row.column_name));
  const missing = SKILL_ARTIFACT_REQUIRED_COLUMNS.filter((column) => !present.has(column));
  if (missing.length === 0) return;
  logger.warn({ event: 'skill_artifact_schema_drift', missingColumns: missing },
    `[Startup] skill_artifact 缺列 (${missing.join(', ')}); skill 相关 API 将返回 503，请执行 npx prisma migrate deploy`);
}
```

- **启动期**：查 `information_schema.columns` 看表结构对不对，缺列就告警。
- **运行期**：识别 Prisma 的 P2021/P2022 错误（表不存在/列不存在），动态判定为漂移。
- **HTTP 降级**：漂移时技能相关 API 返回 503，提示运维跑 migration，而不是让请求带着半残的技能瞎跑。

热更新和 Schema 漂移是一对矛盾——**你越想做热更新，越要防 Schema 漂移，因为热更新意味着代码和 DB 的版本可能错配。** 漂移检测是热更新体系的"安全网"。

---

## 第三套：路由规则热更新——shadow → active 灰度

第三套热更新管的是决策引擎的路由规则（route_rule）。改一条路由规则——比如"凡是提到'周报'的消息走阿宁"——不能重启进程，这没问题（route_rule 在 DB 里，ConfigDbListener 能感知）。但路由规则有一个特殊的诉求：**灰度验证**。

你想加一条新规则，但不确定它准不准。如果直接上线，万一规则错了，一大堆消息会被路由错——用户感受是"突然 AI 变傻了"。怎么降低这个风险？

### shadow 规则：只观察，不生效

我们的 route_rule 表有个 `status` 字段，取值 `active` 或 `shadow`。FusionRouter 对两种状态的处理完全不同：

```typescript
// agents/core/agent/decision/fusion-router.ts（第 165-212 行）
route(input: string): RouteResult | null {
  let bestRoute: RouteEntry | null = null;
  let bestScore = 0;
  for (const route of this.routes) {
    const score = this.computeScore(route, input);
    if (score >= route.semanticThreshold) {
      if (route.status === 'shadow') {       // 影子规则只记录，不路由
        this.bumpMetrics(route.id);
        this.shadowHits.push({...});
        continue;                             // ← 关键：跳过，不参与实际路由
      }
      if (score > bestScore) {
        bestScore = score; bestRoute = route;
      }
    }
  }
  // ...
}
```

`shadow` 状态的规则，FusionRouter 算了它的分（`computeScore`），如果本来会命中，**只记一笔"如果启用会命中"的指标（`shadowHits`），但不实际路由**。实际路由还是走 active 规则。

这就是灰度验证——你把新规则设成 shadow 上线，跑一段时间（比如一周），看两个数据：

1. **命中率**：这条 shadow 规则在真实流量下命中了多少次？（`hitCount` 字段）
2. **命中质量**：命中的那些消息，如果真的按这条规则路由，对不对？（人工抽查 shadowHits 记录的输入）

确认命中率和准确率都达标后，再把 status 从 `shadow` 改成 `active`——一次 DB 更新，ConfigDbListener 感知，FusionRouter 下次加载就让它真正生效。**整个过程零重启、零中断、可控可回滚**（发现问题再改回 shadow 即可）。

### 灰度的本质：可观测先于生效

shadow → active 这套机制的本质，是把"上线"拆成了两步：**先让它可见（shadow，能观测但不生效），再让它生效（active）**。

```
新路由规则的生命周期：
  DB 插入 status=shadow
      ↓ ConfigDbListener 热加载（零重启）
  FusionRouter 灰度观察期（记 shadowHits，不影响真实路由）
      ↓ 人工验证命中率和准确率
  DB 更新 status=active
      ↓ ConfigDbListener 再次热加载（零重启）
  规则正式生效，参与真实路由
      ↓ 发现问题？
  DB 改回 status=shadow（瞬时回滚，零重启）
```

每一步都是改 DB，不重启。这就是"热更新"和"灰度"的结合——**热更新让变更成本低，灰度让变更风险可控**。两者配套，才能在企业里安全地高频迭代路由规则。

---

## 三套机制的共同模式

把三套热更新放一起，能看到一个共同的模式：

| 机制 | 真源 | 同步管道 | 生效方式 |
|------|------|---------|---------|
| 配置热更新 | PG agent_config 等 | ConfigDbListener（LISTEN/NOTIFY + 防抖 + 去重） | Role.reloadFromConfig() |
| 技能热更新 | PG skill_artifact + 对象存储 | SkillArtifactStore + 哈希校验 | SkillRegistry.resolve() 动态解析最新版 |
| 路由热更新 | PG route_rule | ConfigDbListener（同配置） | FusionRouter 下次 route() 读最新规则 |

三个共同点：

**1. 真源都在 DB**。不在配置文件、不在环境变量、不在代码里。DB 是唯一真源，改 DB 就是改真源。

**2. 都有一根从 DB 到内存的同步管道**。配置和路由走 ConfigDbListener，技能走 SkillArtifactStore。管道的作用是把"DB 的事实"同步到"进程的内存"。

**3. 生效都是"下次读取时拿新版"**。不是"立刻把正在跑的任务换成新版"，而是"正在跑的跑完，下次开始的新任务用新版"。这是热更新的安全边界——**不打断进行中的事，只让新开始的事用新配置**。

---

## 热更新的边界：什么不能热更新

讲完了三套能热更新的，也要说清楚什么不能热更新——否则会给人"什么都能热更新"的错误印象。

**不能热更新的，是涉及执行代码逻辑的变更**。比如：

- StreamingToolExecutor 的循环逻辑改了——要发版重启。
- TurnRunner 的编排流程改了——要发版重启。
- 任何 TypeScript 代码层面的修改——要发版重启。

热更新只管"数据"（配置、技能产物、路由规则），不管"代码"。代码层的变更，老老实实走 CI/CD + 滚动部署。我们的四进程架构（api / scheduled / rag / embedding，见第 26 章）配合 K8s 滚动更新，能做到代码发版时的零停机——但那是"多副本滚动"的功劳，不是"热更新"的功劳。

**区分清楚"数据热更新"和"代码滚动部署"**，是做零停机系统的基本认知。前者靠 ConfigDbListener 这类机制，后者靠多副本 + 健康检查。两者解决的问题不同，不能混为一谈。

---

## 给后来者的几条总结

1. **热更新的前提是真源在 DB**。配置、技能、路由规则都存 DB，改 DB 就是改真源。配置文件和环境变量不适合高频变更。
2. **PG LISTEN/NOTIFY 适合"真源就在 PG"的场景**。省掉一次中转，配置写入和通知在同一个事务里。但必须直连 PG，不能走 PgBouncer transaction pooling。
3. **独立连接 + 防抖 + 去重是热更新管道的三件套**。独立 pg.Client 防回收、500ms 防抖防抖动、Map 去重防重复处理。
4. **技能热更新用产物模式，不要硬编码 bundled**。artifact + version + hash 是动态能力的标配。bundled 是条死路，后期迁移代价远大于前期设计。
5. **哈希校验保证版本一致性**。技能变了 hash 变，旧蒸馏指南自动失效。所有环节靠 hash 对齐。
6. **Schema 漂移是热更新的反面**。越想做热更新，越要防代码和 DB 版本错配。三层漂移检测（启动期/运行期/HTTP 503）是安全网。
7. **shadow → active 是路由规则的灰度范式**。先让规则可见但不生效，验证后再生效。每一步改 DB，零重启、可回滚。
8. **区分数据热更新和代码滚动部署**。前者靠同步管道，后者靠多副本 + 健康检查。别指望热更新解决代码发版。

零停机不是一项技术，而是一组设计——真源在 DB、同步管道可靠、生效不打断、变更可灰度可回滚。这套设计做扎实了，企业里的高频迭代才有可能。

---

> **下一篇**：[《WinMatrix vs Anthropic orchestrator-worker：角色优先 vs 大脑优先》](./31-vs-orchestrator-worker.md)——横切主题讲完了，接下来进入"行业对比与方法论"系列。第一篇聊聊两种多 Agent 范式的取舍。
