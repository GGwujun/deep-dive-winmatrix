# 数字员工的"人格"是怎么来的：从 agent_config 到提示词

> 这是《WinMatrix 开发经验文集》第 41 篇，第三批"更细的源码剖析"的第一篇。前 40 篇里我们多次提到"八大角色""配置驱动人格"，但一直没把"一个数字员工的身份到底由哪些字段拼出来的、改一个字怎么热生效"这件事单独讲透。这一篇就补上。

做过 Agent 的人多半写过这种代码：在某个常量文件里写死一段 prompt，叫"系统提示词"，所有请求都带上它。一开始够用，但很快会撞墙——同一个"阿码"，在 A 项目里要按 B 规范审代码，在 B 项目里要兼顾前端框架；同事说"我想让大福更啰嗦一点"；安全说"全局加一条'不许执行删库'的护栏"。这些诉求用一段写死的 prompt 全解决不了。

WinMatrix 的解法是把"人格"拆成几个正交的维度，全部入库，配置驱动，热重载生效。这一篇拆开看这件事是怎么做的。

> 如果你还没读过 [第 6 篇（技能系统）](./06-skill-system.md) 和 [第 9 篇（基础设施踩坑）](./09-pitfalls-infra.md)，建议先看一眼——这篇会引用其中的概念。

---

## 人格不是一段 prompt，是五个正交维度

很多人把"人格"等同于 system prompt。在 WinMatrix 里，人格是几个**正交维度**的叠加：

```
一个数字员工的"人格" =
   身份（profile / goal / constraints）     ← 我是谁、我干什么、我不干什么
 + 性格（personality）                      ← 我说话什么风格
 + 原则（principles）                       ← 我做决策守什么底线
 + 全局护栏（PromptSection）                ← 所有角色都适用的硬约束
 + 角色补注（role_supplement_note）         ← 这个具体分身/项目的额外叮嘱
```

这五层不是平铺在一个字符串里，而是各自有独立的来源、独立的更新通道、独立的缓存语义。把它们拆开，才能做到"改一处不牵动全身"。

我们一个个看。

---

## 第一层：身份从 `agent_config` 表加载

最底层的是"身份"——这个员工叫什么、干什么、关注什么。这部分存在 `agent_config` 表里：

```prisma
// prisma/schema.prisma（第 1159-1181 行）
model agent_config {
  id                     String                  @id
  name_cn                String                  @map("name_cn")
  name_en                String                  @map("name_en")
  nickname               String                  @map("nickname")
  emoji                  String?
  description            String
  role                   String                  @map("role")
  focus                  String                  @map("focus")
  projectId              String?                 @map("project_id")
  workstation            Json?
  workstation_config_version Int                 @default(0)
  version                String?
  updated_at             DateTime                @db.Timestamptz(6)
}
```

几个关键字段的语义：

| 字段 | 作用 | 例子（阿码） |
|------|------|-------------|
| `name_cn` / `name_en` | 正式名字 | 研发技术管理者 |
| `nickname` | 昵称 | 阿码 |
| `role` | 职位描述，作为 profile | "研发技术管理者" |
| `description` | 详细职责，作为 goal | "制定工程化标准，守护代码质量…" |
| `focus` | 工作焦点，作为 constraints | "简单优先、成熟稳定…" |
| `projectId` | 项目归属 | null = 平台 Role；非空 = 项目空间 Role |

注意 `projectId` 的设计：**一个 Role 定义（roleId）可以对应多条 agent_config 行**——平台有一行（projectId=null），每个项目可以再各有一行。这意味着"阿码"在平台层面是同一个角色，但在不同项目里可以有完全不同的 description、focus，甚至工作站配置。

Role 实现类在构造时从这张表加载身份：

```typescript
// agents/domain-harness/roles/TechManagerRole.ts（第 35-49 行）
constructor() {
  super({ reactMode: RoleReactMode.REACT, watch: [] });
  const agentConfig = this.loadConfigFromManager('tech_manager');
  this.name = 'tech_manager';
  this.profile = agentConfig?.role || '研发技术管理者';
  this.goal = agentConfig?.description || '制定工程化标准，守护代码质量...';
  this.constraints = agentConfig?.focus || '简单优先、成熟稳定...';
  this.nickname = agentConfig?.nickname || '阿码';
  this.onInitialized();
}
```

注意每个字段都有 `||` 兜底——如果配置没加载到，用硬编码默认值。这让系统在"配置表还没初始化""DB 抖动读不到"时也能跑起来。**生产代码里，外部配置永远要有兜底默认值，而且默认值要写进版本控制、可审计。**

---

## 第二层：AgentConfig 把身份升格为"能力 + 性格 + 原则"

`agent_config` 是数据库行，是纯数据。代码里消费它的是 `AgentConfig` 接口，这个接口比表结构更丰富：

```typescript
// types/config.ts（第 164-187 行）
export interface AgentConfig {
  id: string;
  name_cn: string;
  nickname: string;
  description: string;
  role: string;
  focus: string;
  /** 为空表示平台 Role；非空表示项目空间内的 Role 定义 */
  projectId?: string | null;
  tools: AgentToolConfig[];
  capabilities: AgentCapability[];
  skills: AgentSkillConfig[];
  collaboration?: AgentCollaboration[];
  /** 性格特征列表（如 "🎯 目标导向：始终盯着最终目标"） */
  personality?: string[];
  /** 决策原则列表 */
  principles?: AgentPrinciple[];
  workstation?: AgentWorkstationConfig;
  memory?: AgentMemoryRecallConfig;
}

export interface AgentPrinciple {
  principle: string;
  description: string;
}
```

这里有两个表里没有的字段值得注意：

**`personality: string[]`**——性格特征列表。不是一段散文，而是结构化的、带 emoji 的短句数组（比如"🎯 目标导向""🔇 降噪优先"）。为什么是数组？因为这样每条都能独立增删、独立审计，比一段抹不开的散文好维护。注入提示词时也是逐条渲染，顺序稳定。

**`principles: AgentPrinciple[]`**——决策原则。每条带 `principle`（名字）和 `description`（详述）。这比 personality 更硬，是"做决策时必须遵守的底线"。比如"不绕过测试""改动前先看影响面"。

把性格和原则拆成两个维度，是因为它们**更新频率和审批级别不同**。性格可以随便调（"今天大福话多点"），原则通常要评审（"不许跳过安全检查"）。混在一起改，一次"调风格"的改动可能误删一条原则。分开存，改动面才清晰。

---

## 第三层：三层分离——DigitalEmployee / BaseRole / RoleContext

光有数据还不够，得搞清楚"谁来读这些配置、读到之后写到哪"。WinMatrix 的答案是三层分离：

```
┌─────────────────────────────────────────────────────┐
│  DigitalEmployee（执行编排层）                       │
│  唯一外部 RoleContext 写入者                         │
│  负责把 agent_config + prompt override 注入上下文    │
└──────────────┬──────────────────────────────────────┘
               │ 持有
┌──────────────▼──────────────────────────────────────┐
│  BaseRole（能力定义层）                              │
│  Observe-Think-Act 循环、技能执行                    │
│  从 configManager 加载身份字段                       │
└──────────────┬──────────────────────────────────────┘
               │ 读写
┌──────────────▼──────────────────────────────────────┐
│  RoleContext（运行时数据容器）                       │
│  当前会话的状态、消息队列、临时变量                  │
└─────────────────────────────────────────────────────┘
```

为什么 DigitalEmployee 是"唯一外部 RoleContext 写入者"？因为 RoleContext 是会话级运行时状态，多个地方乱写就会并发覆盖。把写入权收口到一个组件，并发问题就从"处处小心"变成"一处兜住"。

数字员工工厂创建实例时，会把 PromptOverride 的结果一并注入：

```typescript
// agents/core/worker/digitalEmployee/DigitalEmployee.ts（第 410-436 行）
const role = await roleRegistry.createRole(record.role_id);
if (!role) throw new Error(`[createDigitalEmployee] 未找到对应 Role: ${record.role_id}`);
const promptOverrideService = container.resolve<{
  getPromptContext(id): Promise<{...}>
}>('PromptOverrideService');
const promptContext = await promptOverrideService.getPromptContext(digitalEmployeeId);
if (promptContext) {
  record.role_supplement_note = promptContext.roleSupplementNote ?? null;
  record.prompt_override = promptContext.promptOverride ?? null;
}
return new DigitalEmployee(record, role);
```

这里有两个细节：**每次会话都创建独立的 Role 实例**（`createRole` 不缓存），避免单例被并发请求互相覆盖；**PromptOverride 在工厂里注入**，而不是在每次 LLM 调用时再查——这样一次会话内人格稳定，不会"聊到一半风格变了"。

---

## 第四层：PromptSection——全局护栏的 DB 化

前三层讲的是"角色的人格"。还有一类内容不属于任何角色，但所有角色都要遵守——**全局护栏**。比如"不许执行删库""工具调用前必须确认"。

这些护栏早年写死在代码常量里。问题是改一条要发版。WinMatrix 的做法是把它也入库，用 `PromptSection` 表承载：

```prisma
// prisma/schema.prisma（第 1825-1841 行）
/// 全局/角色级 prompt section（轴 A：ContextSource 路径）
/// 承载全局 guardrail（security_guardrails/tool_call_instructions 等）的 DB 化 + 版本控制。
model PromptSection {
  id          String   @id @default(cuid())
  sectionType String   @map("section_type")
  scope       String   @default("global")
  roleId      String?  @map("role_id")
  content     String
  version     String   @default("v1.0.0")
  contentHash String   @map("content_hash")
  enabled     Boolean  @default(true)
  builtin     Boolean  @default(true)
  @@unique([sectionType, scope, version])
}
```

几个设计要点：

**`sectionType` + `scope` + `roleId` 的三维寻址**。`sectionType` 区分内容类型（security_guardrails / tool_call_instructions 等），`scope` 是作用范围，`roleId` 可空——空表示全局，非空表示某角色专属。这样同一种护栏可以有"全局默认版"和"某角色定制版"。

**带 `version` 和 `contentHash`**。护栏内容也是版本化的——改了能追溯，能回滚。`contentHash` 用来判断"这条内容到底变没变"，避免无意义的缓存失效。

**`enabled` + `builtin` 双字段**。`enabled` 控制是否生效，`builtin` 标记是否系统内置——内置条目可以被覆盖但不能被删，保护护栏不被误清空。

加载时有内存缓存，且支持 per-role 优先、缺省回退全局：

```typescript
// infrastructure/persistence/promptSection/PromptSectionService.ts（第 96-102 行）
/** per-role 优先；缺省回退全局（roleId=null）。 */
getCachedSection(sectionType: string, roleId?: string | null): PromptSectionEntry | null {
  if (roleId) {
    const roleScoped = this.sectionsCache.get(cacheKey(sectionType, roleId));
    if (roleScoped) return roleScoped;
  }
  return this.sectionsCache.get(cacheKey(sectionType, null)) ?? null;
}
```

**先查角色级，查不到回退全局**——这是"特例优先、默认兜底"的经典模式。大多数角色用全局护栏就够了，个别角色（比如运维要执行危险操作）可以有专属护栏覆盖。

---

## 第五层：角色补注 role_supplement_note

前面四层解决的是"角色定义"。还有个现实需求：**同一个角色在不同项目、不同分身里要有微调**。比如阿码在 A 项目要额外遵守"所有 PR 必须链接到 Jira 单"，在 B 项目不用。

如果每种微调都新建一个 Role，角色会爆炸。WinMatrix 的做法是 `role_supplement_note`——一段附加在具体数字员工实例上的"补注"。它不改正文角色定义，只在实例层叠加。

补注从哪来？从 `PromptOverrideService` 读：

```typescript
// business/domain/digitalEmployee/PromptOverrideService.ts（第 43-73 行）
export class PromptOverrideService {
  private static readonly CACHE_TTL_SECONDS = 60;
  private static readonly CACHE_PREFIX = 'prompt_override:digital_employee:';

  async getPromptContext(digitalEmployeeId: string): Promise<DigitalEmployeePromptContext | null> {
    const cached = await this.getCachedPromptContext(digitalEmployeeId);
    if (cached) return cached;
    const record = await this.getRepo().getById(digitalEmployeeId);
    if (!record) return null;
    const parsed = this.safeParseOverride(record.prompt_override);
    const context = {
      digitalEmployeeId: record.id,
      roleSupplementNote: record.role_supplement_note ?? undefined,
      promptOverride: parsed ?? undefined,
    };
    await this.setCachedPromptContext(context);
    return context;
  }
}
```

几个工程要点：

**60 秒 Redis 缓存**。补注不是每次 LLM 调用都查库——查一次缓存 60 秒，一个会话内（通常几十秒到几分钟）都走缓存。但 60 秒又足够短，改了之后最多一分钟生效，不用手动清缓存。

**Zod 校验 + 安全降级**。`safeParseOverride` 把损坏的 JSONB 数据降级为 null，而不是抛异常——存进 DB 的脏数据不应该让整个会话崩掉。

**更新时自动失效缓存**。`updatePromptOverride` 写完后立即 `invalidatePromptContextCache`，保证"改了立刻生效，不用等 60 秒"。这是"缓存 + 主动失效"的标准组合。

这一层我们在 [下一篇第 45 篇](./45-prompt-override.md) 还会更详细地展开，这里先记住它存在的意义：**把"角色定义"和"实例微调"解耦，避免为了一个项目的小改动去 fork 一个角色。**

---

## 热重载：改一个字怎么不重启就生效

把这些都入库之后，下一个问题是"改了怎么生效"。重启进程是最笨的办法，但配置改动频繁时不可接受。

WinMatrix 的热重载链路是这样的：

```
管理员在管理台改 agent_config / PromptSection
        │
        ▼
DB 写入 + pg_notify('config_change', payload)
        │
        ▼
ConfigDbListener（独立 pg.Client，LISTEN config_change）
   500ms 防抖 + Map 去重
        │
        ▼
按 type 分发：
  agent_config  → affectedRoleIds → role.reloadFromConfig()
  prompt_section → PromptSectionService.refreshById()
        │
        ▼
下一次会话创建实例时拿到新配置
```

几个关键设计：

**用 PG LISTEN/NOTIFY 而不是 Redis Pub/Sub**。这部分 [第 15 篇](./15-config-hot-reload.md) 详细讲过，这里只说结论：PG 原生的 LISTEN/NOTIFY 能保证"写库的事务提交"和"通知发出"是原子的——绝不会出现"配置写了但通知丢了"或反过来。Redis Pub/Sub 做不到这个保证。

**独立 pg.Client，不走连接池**。LISTEN 是长连接语义，而 PgBouncer 的 transaction-pool 模式会在事务结束时归还连接，LISTEN 就丢了。所以 ConfigDbListener 必须用独立的、不池化的 pg.Client。这是个踩过坑才记得住的约束。

**500ms 防抖 + Map 去重**。管理员批量改配置时，可能一秒内触发几十个 notify。防抖把它们合并成一次刷新；Map 按 `${type}:${id}` 去重，同一个配置连改三次只处理最后一次。这把"批量改配置"从一次小型风暴降维成一次刷新。

**Role 实例感知热重载**。`BaseRole` 在 `onInitialized` 时注册了 `configManager.onConfigChange` 监听，命中自己 roleId 时调 `reloadFromConfig`。这意味着**正在运行的 Role 实例能感知到配置变更**，不用等下次创建新实例。但要注意：运行中会话的行为已经定型，热重载影响的是**下一次**会话。

---

## 五层叠加的完整图景

把五层和热重载串起来：

```
┌──────────────────────────────────────────────────────────┐
│  会话创建实例                                             │
│  ┌────────────────────────────────────────────────────┐  │
│  │ DigitalEmployee (执行编排)                         │  │
│  │  └─ 从 agent_config 加载 identity (profile/goal)   │  │
│  │  └─ 注入 AgentConfig.personality / principles      │  │
│  │  └─ 注入 PromptOverride (role_supplement_note)     │  │
│  │  └─ 持有 BaseRole 实例                             │  │
│  │     └─ Observe-Think-Act 循环                      │  │
│  │        └─ 每轮 LLM 调用前组装 system prompt：      │  │
│  │           identity + personality + principles      │  │
│  │           + PromptSection 全局护栏                 │  │
│  │           + role_supplement_note 实例补注          │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
        ▲
        │ 热重载影响下一次实例创建
        │
┌──────────────────────────────────────────────────────────┐
│  配置层（全部入库，热更新）                                │
│  agent_config (身份)                                      │
│  PromptSection (全局护栏，DB 化 + 版本化)                 │
│  PromptOverride (实例补注，60s Redis 缓存)                │
│                                                           │
│  ConfigDbListener: pg_notify → 防抖去重 → 分发刷新        │
└──────────────────────────────────────────────────────────┘
```

关键洞察：**身份是数据，性格是数据，护栏是数据，补注也是数据**。人格不再是代码里的常量，而是 DB 里的行——这意味着它可以被管理台编辑、被审计、被版本化、被热更新。这才支撑得起"几十个数字员工 × 多个项目 × 频繁微调"的规模。

---

## 给后来者的总结

1. **人格是多个正交维度的叠加，不是一段 prompt**。identity（身份）/ personality（性格）/ principles（原则）/ 全局护栏 / 实例补注——五层各自有来源、更新通道、缓存语义。拆开才能"改一处不牵动全身"。

2. **身份从 `agent_config` 表加载，每字段都要有兜底默认值**。外部配置永远可能读不到，硬编码默认值保底，且默认值进版本控制。

3. **`projectId` 让一个 Role 支持多项目差异化定义**。平台一行 + 每个项目一行，避免为每个项目 fork 角色。

4. **personality 用数组、principles 用对象数组**。结构化字段比散文好维护、好审计、好渲染。两者更新频率和审批级别不同，必须分开存。

5. **三层分离：DigitalEmployee 是唯一 RoleContext 写入者**。把并发收口到一个组件，而不是处处小心。

6. **全局护栏也要 DB 化（PromptSection 表）**。带 version/contentHash/enabled/builtin，支持 per-role 优先、缺省回退全局。改护栏不发版。

7. **role_supplement_note 解耦"角色定义"和"实例微调"**。60 秒 Redis 缓存 + 主动失效，既不频繁查库又能快速生效。Zod 安全解析避免脏数据炸会话。

8. **热重载走 PG LISTEN/NOTIFY + 500ms 防抖 + Map 去重**。必须用独立 pg.Client，不能走 PgBouncer transaction-pool。运行中 Role 实例能感知变更，但影响下一次会话。

数字员工的"人格"工程，本质上是把一段写死的 prompt 拆解成一组可治理的配置。当你的平台要支持几十个角色、上百个项目、频繁的风格微调时，这套分层是必须的——否则你会被"改一个 emoji 都要发版"的泥潭淹没。

---

> **下一篇**：[《智能路由 route_rule：一条规则如何同时用正则、意图词、语义锚点》](./42-route-rule-fusion-router.md)——讲完了人格，下一步看消息进来后怎么路由到对的员工。route_rule 这张表的一条规则里，patterns、positiveIntents、semanticAnchors 是怎么融合打分的，shadow 模式又是干嘛的。
