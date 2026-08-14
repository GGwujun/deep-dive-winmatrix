# PromptSection 漂移：提示词模板改了，旧会话怎么办

> 这是《WinMatrix 开发经验文集》第 69 篇。讲一个所有做 Agent 平台的人迟早会撞上的问题：提示词模板（prompt template）改了，但正在跑的会话用的是旧版本，缓存里还躺着更旧的覆盖——怎么让这些"漂移"不变成 bug。代码来自 WinMatrix 后端真实实现，没有杜撰。

做 Agent 平台的人，注意力大多在"prompt 怎么写效果好"上。但跑过几个月之后你会发现，**比"写 prompt"难十倍的是"改 prompt"**。

因为 prompt 不是代码。代码改了，重新编译部署，所有调用方立刻用新版。但 prompt 改了之后，会冒出一堆尴尬的问题：

- 正在进行的会话，下一轮用新 prompt 还是旧 prompt？
- 缓存里还存着旧的 prompt override，什么时候失效？
- 改了一个角色的 prompt，所有用到这个角色的项目，下次会话会不会行为突变？
- 改错了想回滚，但旧的 prompt 版本还在吗？

这些问题 collectively 我称之为"PromptSection 漂移"。这一篇讲 WinMatrix 怎么用版本化 + 缓存 + 热更新 + 单 enabled 不变式，把漂移从"不可控的行为突变"变成"可预期、可回滚、可审计"的受控变更。

---

## 现象：三种"漂移"导致的行为不一致

我们踩过的 prompt 相关 bug，大致分三类。

**第一类：改了 prompt 但没生效。** 管理员在后台把某个角色的系统提示词改了，刷新页面也看到了新内容。但用户那边反馈"这个员工的行为没变化"。排查发现：prompt 是从 Redis 缓存里读的，缓存还没失效，用户拿到的还是旧的。手动清了缓存才生效。

**第二类：旧会话行为突变。** 一个会话已经聊了 20 轮，第 21 轮时管理员改了 prompt。用户突然发现"这个员工的语气变了"——因为第 21 轮的 prompt 是新版的。用户没改任何东西，体验却断裂了。更糟的是，如果新 prompt 删了某些约束（比如"必须用中文回答"），旧会话突然开始用英文，用户会困惑"我没让它说英文啊"。

**第三类：分身（persona）用了错误的 override。** WinMatrix 支持数字分身——同一个角色在不同项目/不同员工身上可以有 prompt override（个性化补充）。某个分身的 override 缓存里存了一份旧的、已经不合法的 override（比如引用了一个已删除的字段），导致这个分身的 prompt 拼装失败，降级成默认 prompt，行为和别的分身不一致。

这三类的共同点是：**它们都不是"prompt 写错了"，而是"prompt 的版本管理出了问题"。** prompt 本身是对的，但该用新版的用了旧版、该一致的却不一致。

---

## 根因：prompt 的三个"时态"没有统一管理

为什么 prompt 漂移这么难管？因为在 Agent 系统里，prompt 同时存在于三个"时态"：

```
┌──────────────────────────────────────────────────────────────┐
│  数据库（过去真源）                                            │
│  - prompt_section 表（全局/角色级 prompt 段）                  │
│  - agent_prompt_template 表（角色-场景-档位三元组）            │
│  - 同一个槽位可能有多条版本，但只有一条 enabled=true           │
│                                                                │
│  缓存（现在生效）                                              │
│  - PromptSectionService.sectionsCache（内存 Map）              │
│  - PromptOverrideService 的 Redis 60s 缓存                     │
│  - AgentPromptTemplateService.fragmentsCache（内存数组）       │
│                                                                │
│  会话上下文（瞬时快照）                                        │
│  - 每一轮 LLM 调用时，把 prompt 拼装成 messages[]             │
│  - 这个 messages[] 是不可变的——一旦发出，就是历史              │
└──────────────────────────────────────────────────────────────┘
```

问题就出在三个时态的同步上。数据库改了，缓存不知道；缓存刷新了，正在进行的长会话可能拿到了新旧混合的 prompt。

先看数据库层。WinMatrix 有两张表管 prompt。

**`prompt_section` 表（全局/角色级 prompt 段）**，schema 在 `prisma/schema.prisma:1825`：

```prisma
model PromptSection {
  id          String   @id @default(cuid())
  sectionType String   @map("section_type")       // 段类型，如 "system_preamble"
  scope       String   @default("global")
  roleId      String?  @map("role_id")            // 可空：空=全局，非空=角色级
  content     String
  version     String   @default("v1.0.0")
  contentHash String   @map("content_hash")
  enabled     Boolean  @default(true)
  builtin     Boolean  @default(true)
  updatedAt   DateTime @default(now()) @map("updated_at") @db.Timestamptz(6)
  updatedBy   String?  @map("updated_by")

  @@unique([sectionType, scope, version], map: "uq_prompt_section_type_scope_version")
  @@index([scope, enabled], map: "idx_prompt_section_scope_enabled")
  @@map("prompt_section")
}
```

**`agent_prompt_template` 表（角色-场景-档位三元组）**，schema 在 `prisma/schema.prisma:1184`：

```prisma
model agent_prompt_template {
  id           String       @id @default(cuid())
  agent_id     String       @map("agent_id")
  scenario     String       @map("scenario")      // 场景：chat / kickoff / followup
  profile      String       @default("standard") @map("profile")   // 档位
  template     String
  token_budget Int          @default(2000) @map("token_budget")
  version      String       @default("v1.0.0")
  content_hash String?      @map("content_hash")
  enabled      Boolean      @default(true) @map("enabled")
  // ...
  @@unique([agent_id, scenario, profile, version])
  @@map("agent_prompt_template")
}
```

注意一个关键设计：**两张表都允许同一个"槽位"存在多条版本记录，但通过 `enabled` 字段保证每个槽位只有一条生效。** prompt_section 的槽位是 `[sectionType, scope, roleId]`；agent_prompt_template 的槽位是 `[agent_id, scenario, profile]`。

这个设计的好处是：**旧版本不会被删除，只是被禁用。** 回滚就是"禁用新的、启用旧的"。审计就是"查一个槽位的所有版本历史"。

但代价是：**"每槽位单 enabled"是一个不变式，必须靠代码强制维护。** 如果有人在 DB 里直接 INSERT 了一条 enabled=true 的新版本，又没禁用旧的，这个槽位就有两条 enabled=true，读的时候取哪条就成了未定义行为。这就是漂移的温床。

---

## 修复一：createVersion 用事务守住"单 enabled"不变式

PromptSectionService 的 `createVersion` 方法（`infrastructure/persistence/promptSection/PromptSectionService.ts:161`）用事务守住这个不变式：

```typescript
// src/infrastructure/persistence/promptSection/PromptSectionService.ts:161
async createVersion(input: PromptSectionCreateVersionInput): Promise<{ id: string; contentHash: string }> {
  const scope = input.scope ?? 'global';
  const roleId = input.roleId ?? null;
  const contentHash = sha256Hex(input.content);
  const id = randomUUID();
  await prisma.$transaction(async (tx) => {
    // 1. 先禁用同槽位旧的 enabled 行
    await tx.$executeRaw`
      UPDATE prompt_section SET enabled = false, updated_at = NOW()
      WHERE section_type = ${input.sectionType}
        AND scope = ${scope}
        AND COALESCE(role_id, '') = COALESCE(${roleId}, '')
        AND enabled = true
    `;
    // 2. 再插入新行（enabled=true）
    await tx.$executeRaw`
      INSERT INTO prompt_section (id, section_type, scope, role_id, content, version, content_hash, enabled, builtin, updated_at, updated_by)
      VALUES (${id}, ${input.sectionType}, ${scope}, ${roleId}, ${input.content}, ${input.version}, ${contentHash}, true, ${builtin}, NOW(), ${input.updatedBy ?? null})
    `;
  });
  return { id, contentHash };
}
```

两步在一个事务里：先 UPDATE 禁用旧的，再 INSERT 新的。事务保证了"中间状态不可见"——不会有任何一个时刻，这个槽位有两条 enabled=true，或者零条 enabled=true。

`COALESCE(role_id, '') = COALESCE(${roleId}, '')` 这个写法值得注意。`role_id` 可空，而 SQL 里 `NULL = NULL` 的结果是 `NULL`（falsy），直接 `WHERE role_id = ${roleId}` 在 roleId 为空时匹配不到任何行。用 COALESCE 把 NULL 转成空串再比较，才能正确匹配"全局段"（role_id 为空的那些）。这是 SQL 处理可空字段的经典技巧，处理不好就会导致"全局 prompt 不变量失效"。

`setEnabled` 方法（行 186）用了同样的思路：启用一条时，先禁用同槽位其他 enabled 行：

```typescript
// 启用时先禁用同槽位其他 enabled 行（保持单 enabled 不变式）
async setEnabled(id: string, enabled: boolean, updatedBy?: string | null): Promise<void> {
  await prisma.$transaction(async (tx) => {
    if (enabled) {
      const rows = await tx.$queryRaw<...>`
        SELECT section_type, scope, role_id FROM prompt_section WHERE id = ${id} LIMIT 1
      `;
      const row = rows[0];
      if (row) {
        await tx.$executeRaw`
          UPDATE prompt_section SET enabled = false, updated_at = NOW()
          WHERE section_type = ${row.section_type}
            AND scope = ${row.scope}
            AND COALESCE(role_id, '') = COALESCE(${row.role_id}, '')
            AND enabled = true
        `;
      }
    }
    await tx.$executeRaw`
      UPDATE prompt_section SET enabled = ${enabled}, updated_at = NOW(), updated_by = ${updatedBy ?? null}
      WHERE id = ${id}
    `;
  });
}
```

这套不变式守护把"单 enabled"从口头约定变成了数据库事务保证。无论怎么改版本，槽位始终有且仅有一条 enabled=true 的记录。

---

## 修复二：三层缓存 + 增量热更新

不变式守住了数据库，但读路径走的是缓存。PromptSectionService 有一层内存缓存：

```typescript
// src/infrastructure/persistence/promptSection/PromptSectionService.ts:72
export class PromptSectionService {
  private readonly sectionsCache: Map<string, PromptSectionEntry> = new Map();

  async refreshCache(): Promise<number> {
    const rows = await prisma.$queryRaw<PromptSectionRow[]>`
      SELECT id, section_type, scope, role_id, content, version, content_hash, enabled, builtin
      FROM prompt_section
      WHERE enabled = true
    `;
    this.sectionsCache.clear();
    for (const row of rows) {
      const entry = this.rowToEntry(row);
      if (entry) {
        this.sectionsCache.set(cacheKey(entry.sectionType, entry.roleId), entry);
      }
    }
    return this.sectionsCache.size;
  }
}
```

缓存 key 是 `sectionType:roleId`（或 `sectionType` 表示全局）。读取时 per-role 优先，缺省回退全局：

```typescript
// per-role 优先；缺省回退全局（roleId=null）
getCachedSection(sectionType: string, roleId?: string | null): PromptSectionEntry | null {
  if (roleId) {
    const roleScoped = this.sectionsCache.get(cacheKey(sectionType, roleId));
    if (roleScoped) return roleScoped;
  }
  return this.sectionsCache.get(cacheKey(sectionType, null)) ?? null;
}
```

这个回退逻辑很重要：**角色级 prompt 段优先，没有就用全局段。** 这样你可以写一个"通用的系统前言"（全局），再为特定角色写一个覆盖版（角色级）。读的时候自动选最具体的那个。

但缓存引入了新问题：**DB 改了，缓存怎么知道？** 答案是增量热更新。`refreshById` 方法只刷新一条记录：

```typescript
// src/infrastructure/persistence/promptSection/PromptSectionService.ts:109
async refreshById(id: string): Promise<boolean> {
  const rows = await prisma.$queryRaw<PromptSectionRow[]>`
    SELECT id, section_type, scope, role_id, content, version, content_hash, enabled, builtin
    FROM prompt_section WHERE id = ${id} LIMIT 1
  `;
  if (rows.length === 0) return false;
  const row = rows[0]!;
  const key = cacheKey(row.section_type, row.role_id);
  if (!row.enabled) {
    this.sectionsCache.delete(key);   // 被禁用，从缓存删掉
    return true;
  }
  const entry = this.rowToEntry(row);
  if (entry) {
    this.sectionsCache.set(key, entry);   // 更新缓存
  }
  return true;
}
```

谁触发 `refreshById`？是 ConfigDbListener 监听 PG 的 `pg_notify('config_change')`（详见第 15 篇）。DB 有 trigger，prompt_section 表变更时发 NOTIFY，监听器收到后调 `refreshById(configId)`。这套链路在 `registerDefaultConfigChangeSideEffects.ts` 里：

```typescript
// src/agents/core/kernel-management/config/registerDefaultConfigChangeSideEffects.ts:109
if (configType === 'prompt_section') {
  const refreshed = await promptSectionService.refreshById(configId);
  if (!refreshed) {
    await safeRefreshPromptSectionCache();   // refreshById 失败则全量刷新（兜底）
  }
  logger.info({ configType, configId, action, refreshedById: refreshed },
    '[PromptSectionService] prompt_section 热更新');
  return;
}
```

注意这个双保险：**优先增量刷新（`refreshById`），增量失败则全量刷新（`safeRefreshPromptSectionCache`）。** 增量刷新只查一条记录，快；但如果那条记录已经被删了（`refreshById` 返回 false），说明 DB 状态和预期不符，退回全量刷新保证一致性。

agent_prompt_template 的热更新链路完全对称：

```typescript
// registerDefaultConfigChangeSideEffects.ts:65
if (configType === 'agent_prompt_template') {
  promptRegistry.invalidateByConfigType(configType);   // 先失效 registry 里的旧片段
  const refreshed = await agentPromptTemplateService.refreshById(configId);   // 增量刷新 service 缓存
  if (refreshed) {
    promptRegistry.replaceBySource(
      'db-agent-prompt-template',
      agentPromptTemplateService.getCachedFragments(),
    );   // 用新缓存重建 registry
  } else {
    await safeRefreshAgentPromptTemplateCache();   // 全量兜底
    promptRegistry.replaceBySource('db-agent-prompt-template',
      agentPromptTemplateService.getCachedFragments());
  }
  return;
}
```

这里多了一层：`promptRegistry`（运行时拼装 prompt 的注册表）也要同步刷新。因为 prompt 拼装时是从 registry 读的，不直接读 service 缓存。`invalidateByConfigType` + `replaceBySource` 的组合，保证了 registry 里永远是最新版本。

---

## 修复三：PromptOverride 的 60s Redis 缓存 + 主动失效

上面讲的是"平台级"的 prompt 段和模板。还有一层是"分身级"的个性化覆盖——PromptOverride。它给每个数字员工单独存一份 `roleSupplementNote`（角色补充说明）和 `promptOverride`（结构化覆盖，如语气、风格、额外职责）。

PromptOverrideService 用 Redis 缓存（`business/domain/digitalEmployee/PromptOverrideService.ts`）：

```typescript
// src/business/domain/digitalEmployee/PromptOverrideService.ts:43
export class PromptOverrideService {
  private static readonly CACHE_TTL_SECONDS = 60;
  private static readonly CACHE_PREFIX = 'prompt_override:digital_employee:';

  async getPromptContext(digitalEmployeeId: string): Promise<DigitalEmployeePromptContext | null> {
    const cached = await this.getCachedPromptContext(digitalEmployeeId);
    if (cached) return cached;   // 缓存命中直接返回

    const record = await this.getRepo().getById(digitalEmployeeId);   // 缓存未命中读 DB
    if (!record) return null;

    const parsed = this.safeParseOverride(record.prompt_override);   // Zod 安全解析
    const context = {
      digitalEmployeeId: record.id,
      roleSupplementNote: record.role_supplement_note ?? undefined,
      promptOverride: parsed ?? undefined,
    };
    await this.setCachedPromptContext(context);   // 回填缓存
    return context;
  }
}
```

60 秒 TTL 是个有意为之的权衡：**够长，避免每轮对话都查 DB；够短，改了 override 后最多 60 秒自动生效。** 这是"最终一致性"窗口——改了 override，不保证立刻生效，但保证 60 秒内一定生效。

但 60 秒对于"管理员明确改了想立刻看效果"的场景还是太长。所以有主动失效：

```typescript
// PromptOverrideService.ts:82
async updatePromptOverride(
  digitalEmployeeId: string,
  override: DigitalEmployeePromptOverride | null,
): Promise<boolean> {
  if (override) {
    DigitalEmployeePromptOverrideSchema.parse(override);   // 写入前 Zod 强制校验
  }
  const ok = await this.getRepo().update(digitalEmployeeId, { prompt_override: override });
  await this.invalidatePromptContextCache(digitalEmployeeId);   // 写后立刻失效缓存
  return ok;
}
```

写 override 后，立刻 `invalidatePromptContextCache` 删掉 Redis 里那条缓存。这样下次读会 miss、回源 DB、拿到新值。**写时主动失效 + 读时 TTL 兜底**，这是缓存一致性的经典双保险：

- 正常路径：管理员改 override → updatePromptOverride → 失效缓存 → 下次读拿新值
- 异常路径（失效失败）：TTL 60 秒后自动过期 → 下次读拿新值

`invalidatePromptContextCache` 支持按 ID 失效和全量失效两种：

```typescript
// PromptOverrideService.ts:101
async invalidatePromptContextCache(digitalEmployeeId?: string): Promise<void> {
  try {
    const redis = await getRedisClient();
    if (digitalEmployeeId) {
      await redis.del(this.getCacheKey(digitalEmployeeId));   // 失效单个
      return;
    }
    const keys = await redis.keys(`${PromptOverrideService.CACHE_PREFIX}*`);
    if (keys.length > 0) {
      await redis.del(...keys);   // 失效全部
    }
  } catch (error) {
    logger.warn({ err: getErrorMsg(error), digitalEmployeeId },
      '[PromptOverrideService] failed to invalidate prompt override cache');
  }
}
```

全量失效用 `KEYS` 扫前缀——这在 prompt override 这种"key 总数可控"（等于员工数）的场景下可以接受。如果是海量 key，应该用 SCAN 而不是 KEYS。

还有一个细节：**读取时的 Zod 安全解析**。

```typescript
// PromptOverrideService.ts:160
private safeParseOverride(raw: unknown): DigitalEmployeePromptOverride | null {
  if (!raw) return null;
  try {
    const value = typeof raw === 'string' ? JSON.parse(raw) : raw;
    return DigitalEmployeePromptOverrideSchema.parse(value);
  } catch (error) {
    logger.warn({ err: getErrorMsg(error) },
      '[PromptOverrideService] invalid prompt_override payload; ignored');
    return null;   // 损坏数据降级为 null，不抛异常
  }
}
```

如果 DB 里存的 override payload 损坏了（比如历史脏数据、字段改名后的遗留），`safeParseOverride` 不会抛异常，而是降级为 null（忽略这个 override）。这避免了"一个员工的坏 override 让整个会话崩溃"的连锁反应。**读取时永远要假设数据可能是脏的，用 schema 校验 + 降级。**

---

## 修复四：旧会话用新 prompt 的语义

回到开头那个问题：**正在进行的会话，prompt 改了，下一轮用新还是旧？**

WinMatrix 的答案是：**用新的。每一轮都重新拼装 prompt，从最新的 registry 读取。**

这不是 bug，是有意的设计。原因有三：

1. **prompt 是"系统指令"而非"对话历史"。** 对话历史（用户说了什么、AI 回了什么）是不可变的事实。但 prompt（系统前言、角色设定、约束）是系统指令，应该始终反映最新的配置。就像你更新了 app 的设置，下一次操作就应该用新设置，而不是沿用旧的。
2. **旧会话的"行为突变"是可接受的代价。** 相比"改了 prompt 永远不生效，必须开新会话"，"正在进行的会话下一轮用新 prompt"对用户的冲击小得多。大多数用户感知不到一轮之间的 prompt 微调；如果改动很大，管理员应该选择低峰期改。
3. **prompt 版本化让任何变化可追溯。** 因为每条 prompt 都有 version 和 contentHash，即使行为变了，你也能查到"这个会话的第 21 轮用的是 prompt 的哪个版本"，可以回放和审计。`AgentPromptTemplateService` 的 `rowToFragment` 把 version 和 contentHash 都带进了 fragment 的 meta：

```typescript
// src/business/domain/agentPromptTemplate/AgentPromptTemplateService.ts:201
return {
  meta: {
    promptId: `agent.${row.agent_id}.${scenario}.${profile}`,
    version: row.version,           // ← 版本号
    contentHash: row.content_hash ?? sha256Hex(row.template),   // ← 内容指纹
    // ...
    source: 'db-agent-prompt-template',
  },
  template: row.template,
};
```

如果某天你要查"这个会话的 prompt 是哪个版本"，span 追踪里记了 promptId + version + contentHash，一目了然。

---

## 教训

**第一，prompt 是配置，不是代码——要按配置治理的规矩来管它。** 配置治理的规矩是什么？版本化、可回滚、可审计、热更新。prompt_section 和 agent_prompt_template 都带 version、contentHash、enabled、updatedBy，就是为了这套。如果你把 prompt 写死在代码里，改一次就要重新构建部署，那不是配置，是灾难。

**第二，"每槽位单 enabled"是版本管理的命门，必须用事务守。** 一个槽位（sectionType+scope+roleId 或 agent+scenario+profile）同时有多条 enabled=true，读取就是未定义行为。我们用 `createVersion` 和 `setEnabled` 里的 `BEGIN → 禁旧 → 启新 → COMMIT` 事务守住这条不变式。处理可空字段（role_id）时用 COALESCE 而不是直接 `=`，否则全局段匹配不上。这些是 SQL 层的细节，但细节错了不变式就形同虚设。

**第三，缓存一致性靠"写时主动失效 + 读时 TTL 兜底"的双保险。** PromptOverride 的 60 秒 Redis 缓存就是这套：正常路径写后主动 del，异常路径（del 失败、进程崩溃）靠 60 秒 TTL 自动收敛。不要追求"强一致"（那要分布式锁，代价太大），也不要放任"永久不一致"——60 秒的最终一致性窗口，对 prompt 这种非关键路径的配置来说刚刚好。

**第四，增量刷新优先、全量刷新兜底。** `refreshById` 只查一条记录，快；但查不到（记录被删）说明状态异常，退回全量 `refreshCache`。这个"增量 → 兜底全量"的模式适用于所有"事件驱动的缓存刷新"——增量是为了性能，全量是为了正确性，两者不可偏废。

**第五，读取时永远假设数据可能是脏的。** `safeParseOverride` 用 Zod 校验 + try/catch + 降级为 null，是读取外部存储的标准姿势。DB 里可能有历史脏数据、字段改名后的遗留、人为误操作写入的非法值。如果你的代码读到这些就抛异常，一个坏数据会让整个会话崩。读取时 schema 校验 + 降级，是把"数据正确性"和"系统可用性"解耦的关键。

**第六，prompt 变更的可观测性要落到 span 里。** 每一轮 LLM 调用的 span 应该记录用的 promptId + version + contentHash。这样当用户反馈"这个员工行为变了"，你能查到是哪一轮、哪个 prompt 版本造成的。没有这个，prompt 变更就是黑盒——改了什么、影响了谁、什么时候影响的，全凭猜。把 prompt 版本写进 span，prompt 治理就从"玄学"变成了"工程"。

最后，prompt 漂移这件事的本质是：**prompt 既是"代码"（它决定行为），又是"数据"（它存在 DB 里、会被缓存、会被热更新）。** 它兼具两者的复杂性，却往往得不到两者的待遇——写代码的人不管 prompt 的版本，管 prompt 的人不关心缓存一致性。真正能管好 prompt 的团队，是把 prompt 当成"带版本的、可热更新的、可观测的运行时配置"来对待的。这一整套 PromptSection + agent_prompt_template + 三层缓存 + 热更新 + span 版本追踪，就是这套对待方式的工程化落地。

---

> **上一篇**：[《cron 尖峰迁移：为什么 09:00 的任务要把它们"散开"》](./68-cron-spread-migration.md)
>
> **下一篇**：[《全系列终章：做 AI 平台三年，我会告诉刚开始的自己什么》](./70-three-years-of-ai-platform.md)——01-69 全系列精华浓缩，给"刚开始做 AI 平台"的人的忠告，正式完结整个文集。
