# PromptOverride：同一个角色，不同项目/分身怎么定制提示词

> 这是《WinMatrix 开发经验文集》第 45 篇。[第 41 篇](./41-agent-persona-config.md) 讲数字员工人格时，我们提了一句"角色补注 role_supplement_note 解耦角色定义和实例微调"，但没展开。这一篇就把这个话题讲透——PromptOverrideService + agent_prompt_template 这套组合，是怎么让"同一个阿码，在不同项目里有不同表现"的。

先说一个会让所有做 Agent 平台的人头疼的需求。

你的平台有个角色叫"阿码"（tech_manager），负责代码评审。它定义得好好的，有 profile、goal、personality。现在：

- 项目 A 的负责人说："阿码在我们项目里评审时，要特别关注 React 19 的并发渲染坑。"
- 项目 B 的负责人说："阿码在我们项目里，所有评审意见要附上对应的 SonarQube 规则编号。"
- 张三（阿码的个人分身）说："我带的阿码，风格要更直接一点，别那么啰嗦。"

这三句话，**都不是改阿码的角色定义**——角色定义是平台级的，改了会影响所有项目。它们都是"在某个具体实例上做微调"。

如果你为每个微调 fork 一个角色，很快就有几十个"阿码-A""阿码-B""阿码-张三"，维护噩梦。如果你忽略这些诉求，用户觉得不够灵活。

PromptOverride 就是解这个死结的。

---

## 两张表：agent_prompt_template 与 PromptOverride

WinMatrix 里"提示词定制"涉及两套机制，先分清楚：

**第一套：`agent_prompt_template`——按角色管理提示词模板**。它面向"管理员维护角色的提示词"，是角色级的、版本化的。表结构：

```prisma
// prisma/schema.prisma（第 1184-1202 行）
/// Agent 提示词模板（按 scenario/profile/version 管理）
model agent_prompt_template {
  id           String       @id @default(cuid())
  agent_id     String       @map("agent_id")
  scenario     String       @map("scenario")
  profile      String       @default("standard") @map("profile")
  template     String       @map("template")
  token_budget Int          @default(2000) @map("token_budget")
  version      String       @default("v1.0.0") @map("version")
  content_hash String?      @map("content_hash")
  enabled      Boolean      @default(true) @map("enabled")
  updated_at   DateTime     @default(now()) @db.Timestamptz(6)
  updated_by   String?
  agent_config agent_config @relation(fields: [agent_id], references: [id], onDelete: Cascade)

  @@unique([agent_id, scenario, profile, version])
  @@index([scenario, profile, enabled])
}
```

几个设计要点：

**`scenario` + `profile` + `version` 三维定位**。`scenario` 是场景（比如 `code_review`、`tech_solution`），`profile` 是风格档位（默认 `standard`，可以有 `concise`、`detailed` 等），`version` 是版本。同一个角色同一个场景，可以有多个 profile 和多个版本。

**`token_budget`**。每个模板带 token 预算。注入提示词时要考虑预算上限——某个场景的模板太长会挤占其他部分的 token。把预算显式声明出来，注入器才能做取舍。

**`content_hash`**。模板内容的哈希。判断"模板变没变"靠它，不用逐字比较。缓存失效、增量刷新都依赖它。

**`onDelete: Cascade`**。角色删了，它的模板级联删除。避免悬挂引用。

**唯一约束 `@@unique([agent_id, scenario, profile, version])`**。同一角色同一场景同一 profile 下，版本号唯一。这让"同场景多版本共存"成为可能——新版本可以和旧版本并存，灰度切换。

### AgentPromptTemplateService：模板的 CRUD

这张表的服务层有几个细节值得注意：

```typescript
// business/domain/agentPromptTemplate/AgentPromptTemplateService.ts（第 23-34 行）
export const AgentPromptTemplateUpsertSchema = z.object({
  id: z.string().min(1).optional(),
  agentId: z.string().min(1),
  scenario: PromptScenarioSchema,
  profile: PromptProfileSchema.default('standard'),
  template: z.string().min(1),
  tokenBudget: z.number().int().positive().default(2000),
  version: z.string().min(1).default('v1.0.0'),
  enabled: z.boolean().default(true),
  updatedBy: z.string().min(1).optional(),
});
```

`scenario` 和 `profile` 不是任意字符串，而是受枚举 schema 约束（`PromptScenarioSchema` / `PromptProfileSchema`）。**场景和档位必须是预定义词表里的值**，不能随便填。这保证模板的寻址空间是封闭的、可枚举的，不会出现"每个管理员自己发明场景名"的混乱。

upsert 时算 contentHash：

```typescript
// AgentPromptTemplateService.ts（第 60-68 行，upsert 片段）
async upsert(input: AgentPromptTemplateUpsertInput): Promise<void> {
  const parsed = AgentPromptTemplateUpsertSchema.parse(input);
  const id = parsed.id ?? randomUUID();
  const contentHash = sha256Hex(parsed.template);
  await prisma.$executeRaw`
    INSERT INTO agent_prompt_template
    (id, agent_id, scenario, profile, template, token_budget, version, content_hash, enabled, updated_at, updated_by)
    VALUES (...)
    ON CONFLICT (id) DO UPDATE SET ...`;
}
```

用 `ON CONFLICT (id) DO UPDATE`——upsert 语义，有则更新无则插入。`contentHash` 在写入时算好，不依赖触发器或应用层后续计算。

**第二套：PromptOverrideService——实例级的提示词覆盖**。它面向"具体数字员工实例的差异化"，是实例级的、缓存的。这就是 [第 41 篇](./41-agent-persona-config.md) 提到的 `role_supplement_note` 和 `prompt_override`。

---

## PromptOverride：实例级差异化

PromptOverride 解的是更细粒度的问题：**不是改角色的模板，而是在某个具体实例上叠加一层覆盖**。

### 结构化的覆盖字段

PromptOverride 不是一段自由文本，而是结构化的字段：

```typescript
// business/domain/digitalEmployee/PromptOverrideService.ts（第 21-32 行）
export const DigitalEmployeePromptOverrideSchema = z
  .object({
    personality_tone: z.string().min(1).optional(),
    delivery_style: z.string().min(1).optional(),
    extra_responsibilities: z.array(z.string().min(1)).optional(),
    extra_constraints: z.array(z.string().min(1)).optional(),
    glossary: z.record(z.string()).optional(),
  })
  .strict();
```

五个字段，每个有明确语义：

| 字段 | 作用 | 例子 |
|------|------|------|
| `personality_tone` | 语气基调 | "更直接，少用客套" |
| `delivery_style` | 表达风格 | "结论先行，先给方案再给理由" |
| `extra_responsibilities` | 额外职责 | ["关注 React 19 并发渲染坑"] |
| `extra_constraints` | 额外约束 | ["评审意见附 SonarQube 规则编号"] |
| `glossary` | 术语表 | { "PR": "Pull Request" } |

为什么结构化？因为**覆盖是叠加在角色定义之上的，不是替换**。角色定义里已经有 personality、principles，覆盖只是微调其中一部分。如果用自由文本，注入时不知道该往哪叠，容易和角色定义冲突或重复。结构化字段让"覆盖什么"清晰——personality_tone 只影响语气，extra_constraints 只加约束，各管各的。

注意 `.strict()`——schema 严格模式，**多写字段直接报错**。这防止"管理员随手多填一个字段被静默忽略"的坑。严格 schema 是配置治理的底线。

### 覆盖怎么注入实例

数字员工工厂创建实例时，会查 PromptOverride 并注入：

```typescript
// agents/core/worker/digitalEmployee/DigitalEmployee.ts（第 410-436 行）
const role = await roleRegistry.createRole(record.role_id);
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

注入发生在**实例创建时**，不是每次 LLM 调用时。这意味着：

- **一次会话内人格稳定**。不会"聊到一半风格变了"——override 在会话开始时就定型。
- **改了 override 影响下一次会话**，不会影响正在进行中的会话。这是合理的——进行中的会话切换人格会让用户困惑。

---

## 60 秒 Redis 缓存：成本与时效的平衡

PromptOverrideService 最值得讲的工程细节是它的缓存策略：

```typescript
// PromptOverrideService.ts（第 43-73 行）
export class PromptOverrideService {
  private static readonly CACHE_TTL_SECONDS = 60;
  private static readonly CACHE_PREFIX = 'prompt_override:digital_employee:';

  async getPromptContext(digitalEmployeeId: string): Promise<DigitalEmployeePromptContext | null> {
    // 1. 先查缓存
    const cached = await this.getCachedPromptContext(digitalEmployeeId);
    if (cached) return cached;

    // 2. 缓存未命中，查 DB
    const record = await this.getRepo().getById(digitalEmployeeId);
    if (!record) return null;

    // 3. Zod 安全解析
    const parsed = this.safeParseOverride(record.prompt_override);
    const context = {
      digitalEmployeeId: record.id,
      roleSupplementNote: record.role_supplement_note ?? undefined,
      promptOverride: parsed ?? undefined,
    };

    // 4. 回填缓存
    await this.setCachedPromptContext(context);
    return context;
  }
}
```

**60 秒 TTL**。这是个有意思的选择。太短（比如 1 秒）等于没缓存，每个会话都要查库；太长（比如 1 小时）改了之后要等很久才生效。60 秒是个平衡点——一个典型会话（几分钟到几十分钟）内大概率命中缓存，而改了 override 后最多等 1 分钟生效。

**为什么不是"改了立即生效"？** 因为要做到"立即生效"，你得在每次改 override 时主动清缓存。这在技术上可行（而且这个服务确实做了），但**缓存失效要覆盖所有修改入口**——管理台改、API 改、数据库直接改——任何一个入口漏了清缓存，就会出现"改了但没生效"的诡异现象。60 秒 TTL 是兜底——即使主动失效漏了，最多 60 秒后也会自然过期。

### 主动失效 + TTL 兜底

实际上这个服务两者都做了：

```typescript
// PromptOverrideService.ts（第 82-92 行）
async updatePromptOverride(
  digitalEmployeeId: string,
  override: DigitalEmployeePromptOverride | null
): Promise<boolean> {
  if (override) {
    DigitalEmployeePromptOverrideSchema.parse(override);   // 写前强校验
  }
  const ok = await this.getRepo().update(digitalEmployeeId, { prompt_override: override });
  await this.invalidatePromptContextCache(digitalEmployeeId);  // 主动失效
  return ok;
}
```

走 service 的 `updatePromptOverride` 改 override 时，写完立即清缓存——**正常路径立即生效**。但如果有人绕过 service 直接改 DB（比如运维紧急修复），主动失效不会触发——这时靠 60 秒 TTL 兜底。

**主动失效 + TTL 兜底**，是缓存一致性的经典组合拳。主动失效覆盖正常路径（秒级生效），TTL 兜底覆盖异常路径（最多 1 分钟）。两者一起，既快又稳。

`invalidatePromptContextCache` 还支持全量清理（不传 ID 时 `KEYS prompt_override:digital_employee:*` 删所有）。这用于"批量重置"场景——比如全局配置变更后清掉所有实例缓存。

---

## Zod 安全解析：脏数据不能炸会话

override 存在 DB 的 JSONB 字段里。JSONB 的特点是灵活——什么 JSON 都能存进去。这意味着可能有脏数据：历史遗留的格式不对、手动改 DB 改错了、老版本 schema 的数据没迁移。

如果直接 `JSON.parse` 然后用，脏数据会让整个会话崩掉。PromptOverrideService 的做法是 **Zod 安全解析 + 降级**：

```typescript
// PromptOverrideService.ts（第 159-174 行）
/** Zod 安全解析：损坏的 JSONB 数据降级为 null 而不抛异常 */
private safeParseOverride(raw: unknown): DigitalEmployeePromptOverride | null {
  if (!raw) return null;
  try {
    const value = typeof raw === 'string' ? JSON.parse(raw) : raw;
    return DigitalEmployeePromptOverrideSchema.parse(value);
  } catch (error) {
    logger.warn(
      { err: getErrorMsg(error) },
      '[PromptOverrideService] invalid prompt_override payload; ignored'
    );
    return null;
  }
}
```

核心思想：**解析失败不抛异常，降级为 null**。

- DB 里没存 override → 返回 null（正常，用角色默认定义）。
- DB 里存了脏数据 → 解析失败，log warn，返回 null（当作没 override）。
- DB 里存了合法 override → 解析成功，返回结构化对象。

**脏数据最坏的影响是"这个实例的 override 不生效"，而不是"这个实例崩溃"**。这是面向用户体验的设计——一个实例的配置脏了，不应该影响它正常工作（用默认定义），更不应该影响其他实例。

warn 日志保证脏数据可发现——运维查日志能看到"哪些实例的 override 解析失败了"，再去修。静默吞掉错误（既不崩也不记日志）是最糟的，问题永远暴露不出来。

### 写入时的强校验

注意读取时是"安全解析"（失败降级），但写入时是"强校验"（失败抛异常）：

```typescript
if (override) {
  DigitalEmployeePromptOverrideSchema.parse(override);  // 写入前强校验，不合法直接抛
}
```

为什么不对称？因为**写入是受控的**（走 service 的管理操作），不合法的配置应该在写入时就被拦住，而不是写入后再降级读取。读路径要容忍历史脏数据，写路径要阻止新脏数据进入。这是"严进宽出"的治理思路。

---

## 两套机制的分工

把两套机制放一起看，分工很清晰：

```
┌──────────────────────────────────────────────────────────┐
│  角色级模板（agent_prompt_template）                      │
│                                                           │
│  面向：管理员维护角色的标准提示词                          │
│  粒度：角色 × 场景 × profile × version                    │
│  场景：阿码的"代码评审"场景有个 standard profile 模板     │
│  更新：版本化，带 content_hash，可灰度                    │
│  消费：注入器按场景+profile 选模板                         │
└──────────────────────────────────────────────────────────┘
                          │
                          │ 标准定义
                          ▼
┌──────────────────────────────────────────────────────────┐
│  实例级覆盖（PromptOverride）                             │
│                                                           │
│  面向：具体数字员工实例的差异化                            │
│  粒度：单个 digitalEmployeeId                              │
│  场景：项目 A 的阿码要额外关注 React 19 坑                │
│  更新：60s Redis 缓存 + 主动失效                          │
│  消费：实例创建时叠加在角色定义之上                        │
└──────────────────────────────────────────────────────────┘
```

简单说：**模板管"角色的标准提示词长什么样"，override 管"某个具体实例怎么微调"**。前者是基线，后者是增量。两者叠加，才是这个实例最终的人格。

---

## 和第 41 篇的关系：五层人格的落点

回到 [第 41 篇](./41-agent-persona-config.md) 的五层人格模型，现在能更清晰地看到 override 的位置：

```
人格五层：
  1. 身份（agent_config）         ← 平台级，所有项目共享
  2. 性格（personality）          ← AgentConfig 字段
  3. 原则（principles）           ← AgentConfig 字段
  4. 全局护栏（PromptSection）    ← 全局/角色级，DB 化
  5. 实例补注（role_supplement_note + prompt_override）  ← 本篇主角
```

前四层是"角色定义"，第五层是"实例覆盖"。PromptOverride 就是第五层的实现。它保证了：

- **角色定义不漂移**——阿码在所有项目里的 profile/goal/principles 是一致的。
- **实例差异化不 fork**——每个项目的阿码可以有自己的叮嘱，但底子还是同一个阿码。
- **分身继承不平行实现**——张三的阿码分身默认继承项目空间的阿码，要差异化用 override，不另写一套（这呼应 [第 41 篇](./41-agent-persona-config.md) 的"分身同源继承"原则）。

---

## 业界对比：override vs fork

横向看一下，"同一角色多场景差异化"大致两种做法：

**一种是 fork**——每个差异化场景复制一份完整的角色定义。简单粗暴，但维护成本高。改了原角色定义，所有 fork 都要同步改，最终必然漂移成几个独立角色。

**另一种是 override**——保持角色定义单一，差异化部分作为覆盖叠加。WinMatrix 走的是这条路。override 只描述"和默认不一样的地方"，不复制默认部分。角色定义改了，所有实例自动继承新定义，override 不受影响。

override 模式的代价是**注入逻辑更复杂**——要把角色定义和 override 叠加成最终的 prompt，不能简单拼接。但这个复杂度是一次性的（写在注入器里），换来的是长期的维护性（角色定义不漂移）。

**短期看 fork 简单，长期看 override 划算。** 角色越多、差异化场景越多，override 的优势越明显。这就是为什么 WinMatrix 从一开始就选了 override 模式——宁可前期把注入逻辑做扎实，也不要后期陷入"几十个 fork 各自漂移"的泥潭。

---

## 给后来者的总结

1. **提示词定制有两套机制，分工清晰**。agent_prompt_template 管角色级标准模板（scenario/profile/version 三维定位），PromptOverride 管实例级差异化（per digitalEmployeeId）。前者是基线，后者是增量。

2. **agent_prompt_template 的 scenario/profile 受枚举约束**。不是任意字符串，必须是预定义词表。保证寻址空间封闭可枚举，避免"每人发明场景名"的混乱。

3. **PromptOverride 是结构化字段，不是自由文本**。personality_tone / delivery_style / extra_responsibilities / extra_constraints / glossary 五个字段各管各的，叠加在角色定义之上不冲突不重复。`.strict()` 严格模式防多字段被静默忽略。

4. **60 秒 Redis 缓存 + 主动失效 = 成本与时效的平衡**。主动失效覆盖正常路径（秒级生效），60 秒 TTL 兜底覆盖异常路径（绕过 service 改 DB 的场景）。一个典型会话内大概率命中缓存。

5. **override 在实例创建时注入，不是每次 LLM 调用时**。保证一次会话内人格稳定，改 override 影响下一次会话。

6. **读路径安全解析（脏数据降级 null），写路径强校验（不合法抛异常）**。严进宽出——读容忍历史脏数据不崩会话，写阻止新脏数据进入。warn 日志保证脏数据可发现。

7. **override 而非 fork**。保持角色定义单一，差异化作为覆盖叠加。短期看 fork 简单，长期看 override 划算——角色定义不漂移，实例自动继承新定义。

8. **override 是"分身同源继承"原则的落地**。分身默认继承项目空间角色，要差异化用 override，禁止平行实现。从架构上防腐化。

PromptOverride 看起来只是个小服务（200 行不到），但它解决的是一个规模化才会暴露的痛点——**当你的平台有几十个角色、上百个项目、无数个分身时，怎么让"同一个角色"既有统一底线又支持灵活差异化**。override 模式是这个问题的标准答案。

---

> **下一篇**：[《会话执行调度器：一个会话里多条消息怎么排队》](./46-conversation-execution-dispatcher.md)——讲完了人格和提示词定制，最后一篇回到执行层。一个会话里，用户连发了三条消息，系统怎么处理？是并行还是排队？coordinator 模式下的内存队列是怎么工作的？conversationExecutionDispatcher + CoordinatorAdapter 拆解。
