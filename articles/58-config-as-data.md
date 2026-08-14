# 配置即数据：把配置放数据库而不是代码/配置文件

> 这是《WinMatrix 开发经验文集》第 58 篇，也是"更深的工程哲学"段的最后一篇。前面 57 篇里，配置热更新、PgBouncer 绕过、幂等、横切收口都零散出现过。这一篇把它们统一到一个视角下：**配置即数据（config-as-data）。** 当 agent_config、skill_artifact、tool_config、route_rule 这些东西全部入库，配置就不再是代码的附属品，而是和业务数据平等的一等公民。这个决定带来巨大的收益——热更新、审计、版本、治理——但它也带来一个独特的代价：配置漂移。这一篇讲清楚为什么"配置入库"是对的，以及怎么用三层防御兜住它的代价。

先讲一个决定整个系统走向的选择。

做任何后端系统，你都会面对一类东西：**它既不是业务数据（用户、订单），也不是代码逻辑（算法、流程），而是"系统怎么跑"的描述**——数字员工的角色设定、技能的定义、工具的权限、路由的规则。这些东西传统上叫"配置"。

配置放哪里？有三个选择：

```
[1] 放代码里（硬编码或常量文件）
    优点：版本控制天然有，code review 把关
    缺点：改一次要重新构建部署，无法热更新

[2] 放配置文件（yaml/json/env）
    优点：改了重启就行，不用构建
    缺点：多实例要同步，改了要重启，审计弱

[3] 放数据库
    优点：热更新、审计、版本、治理全有
    缺点：配置漂移风险，DB 挂了系统启动不了
```

传统系统大多选 [1] 或 [2]。但 WinMatrix 是一个**企业级 AI 数字员工平台**，它的配置有三个特殊需求：

- **角色/技能/路由要频繁调**：数字员工的性格、技能绑定、路由阈值，是要根据业务持续优化的，不是上线就不动的。
- **改了要立刻生效**：调一个路由阈值，不能等重新构建部署，要改完秒级生效。
- **要审计**：谁在什么时候改了什么配置，出了问题要能追溯。

这三个需求加起来，只有 [3] 能满足。所以 WinMatrix 做了一个根本性的决定：**把配置当数据，存进 PG，和其他业务表平等。** 这个决定贯穿了整个系统的设计。

---

## 四大配置表：配置入库的全景

WinMatrix 入库的配置不是一两个开关，而是**四大类配置，每类一张主表**，覆盖了系统运行的方方面面：

| 配置表 | 管什么 | 关键字段 | 源码 |
|--------|--------|---------|------|
| **agent_config** | 数字员工的身份与人格 | name_cn/nickname/emoji/personality/principles/role/focus/workstation | schema.prisma:1159-1181 |
| **skill_artifact** | 技能定义（提示词+工具+契约） | name/version/scope/trust_level/manifest/persona_eligible/package_sha256 | schema.prisma:1273-1305 |
| **tool_config** | 工具元数据与展示 | name/category/scope/permissions/dependencies/mcp_bridge_visible | schema.prisma:1437-1453 |
| **route_rule** | 智能路由规则 | patterns/positiveIntents/semanticAnchors/semanticThreshold/roleId/status(active\|shadow) | schema.prisma:2689-2730 |

这四张表凑在一起，把"系统怎么跑"几乎全部描述了：

- **agent_config** 描述"谁在跑"——八大数字员工（大福/阿宁/小品/阿码/小质/大维/Architector/external-agent）的身份、性格、原则、工作站配置。Role 加载身份就是从这张表读（`TechManagerRole.ts:35-49`，核实报告 ch09-12）。
- **skill_artifact** 描述"会什么"——每个技能的提示词模板、契约（provides/consumes）、信任级别、是否对分身可用。带 `package_sha256` 做完整性校验，`@@unique([name, version, scope, project_name])` 防重复。
- **tool_config** 描述"能用什么工具"——工具的元数据（实现仍在代码，但展示和分配走这张表）。
- **route_rule** 描述"怎么决定走哪条路"——正则/意图词/语义锚点/阈值/目标角色，还有 `status=shadow` 灰度。

**这四张表的存在，意味着"改配置"变成了"改数据"。** 不用构建，不用重启，不用改代码。改一条 route_rule 的阈值，决策引擎立刻用新值。改 agent_config 的性格描述，数字员工下次回复就是新人格。这是"配置即数据"最直观的收益。

---

## 配置即数据的四大收益

把配置当数据，不是图新鲜，是为了买回四个传统配置方式给不了的能力：

### 收益一：热更新（ConfigDbListener）

最大的收益。配置改了，系统**不用重启**就生效。WinMatrix 的实现是 PG LISTEN/NOTIFY（`ConfigDbListener.ts`，核实报告 ch04-06）：

```
管理员改配置（UPDATE agent_config SET ...）
        │
        ▼
触发器/应用代码调 pg_notify('config_change', { type, id, action })
        │
        ▼
ConfigDbListener 收到通知（独立 pg.Client，LISTEN config_change）
        │
        ├── handleNotification：去重放入 pendingChanges Map
        │   （key = `${type}:${id}`，同 id 多次变更只保留最新）
        │
        └── scheduleDebounce：500ms 防抖合并
                │
                ▼ （防抖结束后批量处理）
            重新加载受影响的配置到内存
            BaseRole.reloadFromConfig() / 缓存失效 / 重新注册
```

注意三个细节：

- **去重 + 防抖**（`handleNotification` 行 241-271，`scheduleDebounce` 行 336-345）：同一配置 500ms 内改多次，只触发一次重载。防止"管理员来回试值"导致系统疯狂重载。
- **独立连接绕开 PgBouncer**（行 90-112）：LISTEN/NOTIFY 不能走 transaction pooling（参考第 54 篇），所以 ConfigDbListener 用独立的 `DATABASE_LISTEN_URL` 直连 PG。
- **bulk 写入期间暂停通知**（`setNotificationSuppressed` 行 194-204）：批量导入配置时暂时屏蔽通知，避免每个 INSERT 都触发一次重载。

**热更新的价值是：把"改配置"的反馈循环从"小时级"（构建部署）压缩到"秒级"。** 这对数字员工的持续优化至关重要——调人格、调路由、调技能，改完立刻能看效果。

### 收益二：审计（ConfigAuditLog）

配置入库后，"谁在什么时候改了什么"天然可查——因为配置变更就是 DB 的 UPDATE，有 `updated_at`，配上审计表就是完整变更链。这是配置文件和代码常量都做不到的。第 61 篇会专门讲审计日志，这里只点出：**配置入库是审计的前提。**

### 收益三：版本（checksum + version 字段）

skill_artifact 带 `package_sha256`（完整性校验），flow_template_version 带 `checksum`（不可变发布版，核实报告 fact-sheet），agent_config 带 `workstation_config_version`（乐观锁，核实报告 ch04-06）。**配置有版本，就能回滚——新版配置出问题，切回旧版。** 这是代码常量做不到的（代码回滚要重新构建）。

### 收益四：治理（权限 + 灰度）

配置入库后，"谁能改配置"就是 DB 权限问题，可以用 RBAC 精细控制。route_rule 的 `status=active|shadow` 让新规则先灰度观察（只记录命中、不实际路由，参考第 23 篇灰度幂等）。**这些都是配置文件/代码常量给不了的治理能力。**

---

## 代价：配置漂移

讲完收益，必须讲代价。"配置即数据"最大的代价是**配置漂移（config drift）**——代码假设的配置结构和 DB 里实际的配置结构不一致。

怎么会发生？三种典型场景：

**场景一：代码发了但迁移没跑。** 新版代码引用了 agent_config 的一个新字段，但生产 DB 的 migration 还没跑（或者跑了但漏了）。代码读这个字段读到 undefined，行为异常。这是最常见的漂移。

**场景二：手动改了 DB 没走代码。** 运维为了应急，直接 UPDATE 了 DB 里的某条配置，没走代码审查。这个改动没有记录、没有版本，下次部署可能被覆盖。

**场景三：多环境不一致。** 开发/测试/生产的配置表内容不同（开发有 mock 角色，生产是真实角色），某次改动只改了一个环境，其他环境漂移。

漂移的危险在于它**静默**——系统看起来在跑，但用的是错的配置，而且很难发现。直到某个功能莫名失效，排查半天才发现是某个字段不一致。

---

## 三层防御：把漂移兜住

WinMatrix 用三层防御把漂移风险兜住。这三层各自针对不同阶段的漂移：

### 防御一：启动期 Schema 漂移检测（fail-fast）

第一层在启动时检查"配置表结构和代码期望的是否一致"。最典型的实现是 skill_artifact 的漂移检测（`skillArtifactSchemaDrift.ts:56-96`，核实报告 ch13-17）：

```ts
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

这段代码在**进程启动时**直接查 `information_schema.columns`，检查 skill_artifact 表有没有代码期望的那些列。缺了就告警，并且 skill 相关 API 返回 503（直接拒绝服务，而不是用错误配置瞎跑）。

**这是 fail-fast 策略**——与其在错误配置下运行出诡异 bug，不如启动时就报错，逼着运维跑 migration。三层防御里这是最硬的一道。

### 防御二：运行期 Prisma 错误识别（P2021/P2022）

第二层在运行时识别漂移导致的错误。即使启动期检查过了，运行中也可能因为各种原因遇到字段缺失（比如 DBA 手动改了表）。WinMatrix 在运行期识别 Prisma 的 P2021（表不存在）和 P2022（列不存在）错误，这些错误几乎都是配置漂移的信号（核实报告 ch13-17）。

这一层比启动期检查更细——启动期检查的是"期望的列在不在"，运行期识别的是"实际查询时碰到了不存在的列/表"。两者互补：启动期挡住大部分，运行期兜住漏网的。

### 防御三：HTTP 503 显式拒绝（sendSkillArtifactSchemaDriftReply）

第三层在 API 响应层。当检测到漂移时，skill 相关 API 不返回错误数据，而是返回 503 + 明确提示"配置漂移，请跑 migration"（核实报告 ch13-17）。这比返回 500 或空数据友好——调用方能立刻知道是配置问题，不是代码 bug。

三层防御凑在一起：

```
配置漂移
  │
  ├── 启动期：warnSkillArtifactSchemaDriftOnStartup（查 information_schema）
  │           → 缺列则告警 + skill API 503
  │
  ├── 运行期：识别 Prisma P2021/P2022（表/列不存在）
  │           → 当作漂移信号处理
  │
  └── 响应层：sendSkillArtifactSchemaDriftReply
              → 503 + 明确提示"请执行 prisma migrate deploy"
```

**这三层的共同精神是：绝不带着漂移的配置"假装正常跑"。** 宁可拒绝服务（503），也不用错误配置产出错误结果。这是 fail-closed 在配置领域的体现——配置不确定时，停下来比乱跑安全。

---

## 一个更深的选择：哪些该入库，哪些不该

"配置即数据"不是要把所有配置都塞进 DB。WinMatrix 也做了明确的区分：

| 配置类型 | 放哪里 | 理由 |
|---------|--------|------|
| 数字员工人格/技能/路由/工具元数据 | **DB** | 频繁调、要热更新、要审计 |
| DB 连接串/Redis 地址/端口号 | **环境变量** | 启动时确定，不和业务挂钩 |
| 权限静态矩阵（ProjectRole/PermissionKey） | **代码常量** | 编译期确定，前后端共享，变更要严格 review |
| 分层规则（import 门禁） | **代码 + 脚本** | 架构约束，不该被运行时改 |

判断标准是：**这个配置会不会被业务人员/运维频繁改？改了要不要立刻生效？要不要审计？** 三个都"是" → 入库。否则用环境变量或代码常量。

**这个区分很重要——把不该入库的塞进 DB，会增加漂移风险；把该入库的留在代码，会失去热更新能力。** WinMatrix 把"业务可调的配置"（角色/技能/路由/工具）入库，把"基础设施配置"（连接串/地址）留环境变量，把"架构约束"（权限矩阵/分层规则）留代码。三者各司其职。

---

## 配置即数据 + 热更新：一个完整的闭环

把"配置入库"和"热更新"（第 30 篇）放一起，WinMatrix 的配置治理是一个完整闭环：

```
管理员改配置（DB UPDATE）
        │
        ├── [收益] pg_notify → ConfigDbListener → 500ms 防抖 → 重载
        │         配置秒级生效，无需重启
        │
        ├── [收益] ConfigAuditLog 记录"谁改了什么"
        │
        ├── [收益] version/checksum 字段支持回滚
        │
        └── [代价] 配置漂移风险
                  ├── 防御1：启动期 Schema 检测（fail-fast）
                  ├── 防御2：运行期 P2021/P2022 识别
                  └── 防御3：HTTP 503 显式拒绝
```

**这个闭环的精髓是：用"三层防御"的代价，换"热更新+审计+版本+治理"四大收益。** 对企业级 AI 平台来说，这个交换是划算的——数字员工的持续优化需要热更新，合规需要审计，安全需要版本和回滚，这些加起来的价值远大于漂移防御的开发成本。

---

## 给后来者的总结

1. **配置即数据：把"系统怎么跑"的描述（角色/技能/工具/路由）存进 DB，和业务数据平等。** 这是企业级 AI 平台的选择，因为它需要热更新、审计、版本、治理。
2. **四大配置表是 WinMatrix 的骨架**：agent_config（人格）/ skill_artifact（技能）/ tool_config（工具）/ route_rule（路由）。改配置就是改数据，不用构建部署。
3. **热更新用 PG LISTEN/NOTIFY + ConfigDbListener。** 去重 + 500ms 防抖 + 独立连接绕开 PgBouncer + bulk 写入期间暂停通知。配置秒级生效。
4. **配置入库的四大收益：热更新、审计、版本、治理。** 这四个能力是代码常量和配置文件都给不了的。
5. **代价是配置漂移，而且漂移是静默的。** 代码发了迁移没跑、手动改 DB、多环境不一致——三种典型场景，都很难发现。
6. **三层防御兜住漂移：启动期 Schema 检测（fail-fast）+ 运行期 P2021/P2022 识别 + HTTP 503 显式拒绝。** 精神是"绝不带着漂移假装正常跑"，宁可拒绝服务。
7. **不是所有配置都该入库。** 业务可调的入库，基础设施配置留环境变量，架构约束留代码。把不该入库的塞进 DB 增加漂移风险，把该入库的留代码失去热更新。
8. **配置漂移检测要 fail-closed。** 检测到漂移时返回 503，而不是用错误配置瞎跑。配置不确定时停下来比乱跑安全。

"配置即数据"不是技术潮流，是工程权衡。它用"漂移防御的复杂度"换"热更新+审计+版本+治理"的自由度。对 demo 系统或小工具，这个交换不划算（代码常量就够了）；对要长期运营的企业级平台，这个交换几乎是必须的——没有热更新，每次调参数都要构建部署，迭代速度会被拖死；没有审计，出了配置事故无法追溯；没有版本，回滚靠祈祷。**当你判断一个系统"该不该把配置入库"时，问自己：这个系统的配置会持续演进吗？需要合规审计吗？出了事需要回滚吗？三个都"是"，就入库，然后把三层漂移防御搭好。** 这是工程哲学段最后一条教训，也是贯穿整个系统设计的一条主线：**自由度和纪律永远成对出现，想吃下自由度的收益，就必须把配套的纪律（这里是漂移防御）一起建起来。**

---

> **上一篇**：[《重放 ≠ 永远安全：哪些操作能自动重试》](./57-replay-safety.md)
>
> **下一篇**：[《多租户：同一个平台，不同项目/团队怎么隔离》](./59-multi-tenancy.md)——哲学段（51-58）到此结束，接下来进入跨界主题段。下一篇从"配置即数据"延伸到"数据隔离"——同一个 WinMatrix 平台，不同项目/团队的配置、记忆、工具、队列怎么彻底隔离，是多租户的核心难题。
