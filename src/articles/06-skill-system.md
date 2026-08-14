# 技能（Skill）即数字员工的能力单元：从定义到治理全流程

> 这是《WinMatrix 开发经验文集》第 6 篇。工具是原子的，技能是"提示词 + Agent 封装 + 契约"的组合体，是数字员工"会干活"的基本单元。一个技能从被定义、加载、检查、执行、到事后学习改进，全流程怎么做？这篇拆给你看。

如果你只看上一篇，会觉得"工具调用"已经够用了——LLM 能调工具，能拿结果，看起来就能干活。

但真正做过企业级 Agent 的人知道，**光有工具不够**。一次"代码评审"，不是一个 `code_review` 工具调用就完事的——它是一套完整的流程：先读代码、再对照规范、再给出评审意见、再生成报告。这套流程里要用到多个工具，有特定的顺序，有特定的提示词，有特定的输入输出。

**技能（Skill）就是这套流程的封装。** 它是数字员工"会干活"的基本单元。这篇文章讲 WinMatrix 的技能系统：一个技能从定义到治理的全流程。

---

## 技能是什么：不是 prompt，不是函数，是产物

很多人把"技能"理解成一段 prompt。错。也有些人理解成一个函数。也不全对。

在 WinMatrix 里，一个技能是一个**完整的产物（Artifact）**：

```prisma
// prisma/schema.prisma（第 1273-1305 行）
model skill_artifact {
  name                  String              // 技能名
  version               String              // 版本
  artifact_path         String              // 工件路径
  sha256                String              // 内容哈希
  scope                 String              // 作用域（system/project）
  trust_level           String              // 信任级别
  manifest              Json                // 清单（步骤、参数、契约）
  persona_eligible      Boolean  @default(true)  // 分身技能可用性
  package_storage_key   String?             // MinIO/S3 存储 key
  package_sha256        String?             // 包哈希
  enabled               Boolean  @default(true)
  @@unique([name, version, scope, project_name])
}
```

一个技能包含：

- **提示词模板**（在 manifest 里）——告诉 LLM 怎么干这件事
- **Agent 封装**（用哪个模型、什么工具、什么记忆）——决定执行环境
- **契约声明**（provides 产出什么、consumes 依赖什么）——决定它怎么和别的技能串联
- **版本和哈希**——决定它怎么演进、怎么分发

它是**带版本、带哈希、可分发、可治理的产物**。不是一段散落的 prompt。

### 为什么必须是产物

把技能做成产物，带来三个关键能力：

**可演进**。技能会变——今天的"代码评审"和三个月后的"代码评审"流程可能不同。带版本（`version`）和哈希（`sha256`），你能追踪"这个技能改过什么、什么时候改的、谁改的"。

**可分发**。技能可以通过市场（Marketplace）安装——从远程仓库拉取，校验 `package_sha256`，写入 `skill_artifact` 表。哈希校验是供应链安全的底线。

**可治理**。技能是独立的治理单元——可以加白名单、要审批、做项目级覆盖。这对企业是刚需。

我们早期踩过一个坑：一开始把技能做成"bundled"的硬编码包，随代码编译进镜像。很快走不通——技能要热更新、要市场安装、要项目复用，bundled 模式满足不了。最终重构为 artifact-store 模式。

```typescript
// scripts/seed-bundled-skills.ts（第 4 行头部注释）
// 显式将 repo bundled skills 写入 artifact-store 与 DB bindings。
// 不进 API 启动链；空库/发版运维使用。
```

**教训：需要动态加载、版本管理、热更新的能力，从一开始就用"产物 + 存储 + 哈希"的思路，别走硬编码 bundled。** 后期迁移成本远大于前期设计成本。

---

## 契约：技能之间怎么串联

技能不是孤岛。一次完整的项目工作，往往要多个技能接力——PRD 写完给技术方案，技术方案完了给代码评审。技能之间怎么传递数据？

答案是**契约**。每个技能声明 `provides`（我产出什么）和 `consumes`（我依赖什么）：

```typescript
// agents/domain-harness/schema/flowContractSchema.ts（第 24-54 行）
export const FlowOutputContractBaseSchema = z.object({
  provides: z.array(z.object({
    key: z.string().min(1),               // 输出键名
    type: z.string().min(1).default('string'),
    required: z.boolean().default(false),
    artifact: z.boolean().default(false), // 是否为制品型输出
  })).default([]),
});

export const FlowInputBindingSchema = z.object({
  key: z.string().min(1),                 // 依赖的输入键
  required: z.boolean().default(true),
  onMissing: z.enum(['block', 'manual', 'skip', 'use_default']).default('block'),
});
```

比如：

- `prd_writing` 技能 `provides: [{ key: 'prd_document', type: 'markdown', artifact: true }]`
- `tech_solution` 技能 `consumes: [{ key: 'prd_document', required: true, onMissing: 'block' }]`

把这两个技能放进一个流程编排，系统就知道：技术方案这一步要等 PRD 产出，PRD 的产出自动注入技术方案的输入。**契约让多技能串联从"写胶水代码"变成"声明数据流"。**

### onMissing 四态：精细的缺失处理

注意 `onMissing` 的四个选项——当 consumes 的输入缺失时，不是一刀切"报错"，而是分四档：

| 值 | 行为 |
|----|------|
| `block` | 阻塞，停下来报错 |
| `manual` | 转人工处理 |
| `skip` | 跳过这一步 |
| `use_default` | 用默认值继续 |

这让编排面对不完整输入时有了精细的策略。有些步骤缺了输入就必须停（block），有些可以跳过（skip），有些用默认值也能跑（use_default）。**现实世界的数据很少是完整的，契约必须能处理"缺这少那"。**

### 契约可被项目覆盖，但不改正文

项目可以微调技能契约，但**覆盖是 merge，不是 replace**：

```prisma
// prisma/schema.prisma（第 1328-1344 行）
model skill_contract_override {
  provides_override  Json?
  consumes_override  Json?
  mode               String  @default("merge")   // 合并模式
}
```

`mode=merge` 意味着项目级覆盖和技能默认契约合并，不是完全替换。这保护了技能的原始定义——你可以调一项，不会意外破坏核心行为。

而且流程编排层有 `FlowSkillContractValidationService` **强校验**：如果编排里某步 consumes 了一个 key，但没有任何前置步骤 provides 它，直接报 `missing_skill_provides` 错误。**编译期就能发现的错误，绝不留到运行期。**

---

## L1/L2/L3：技能能不能用，分三层查

技能在执行前，要经过多层就绪检查。WinMatrix 分成 L1、L2、L3 三层，**每一层都比下一层便宜，越早过滤掉不可用的技能，整体成本越低**。

先澄清一个容易混的点：这里的 L1/L2/L3 是**技能就绪边界**，和决策引擎里的分层路由 L0/L1/L2/L3 是两个完全不同的维度，别混。

### L1：决策阶段的能力清单（最便宜）

L1 发生在决策阶段，只回答一个问题：**这个员工/项目当前有哪些技能可用？**

```typescript
// ProjectSkillScopeService.ts（第 89-101 行）
// L1：从 skill_artifact 读取 enabled ∩ persona_eligible=true 的技能名
async getPersonaEligibleSkillNames(/* ... */): Promise<string[]> {
  // 注释明令"禁止复用 skill_artifact.scope"
  // 范围判定以 persona_eligible + enabled 为准
}
```

L1 只加载技能的正文描述，不做资源绑定、不做预检——那些是更贵的操作。**L1 的目标是让决策引擎在几十上百个技能里快速筛出候选。**

### L2：规划阶段的快照软校验

L2 发生在规划阶段。决策引擎选定要执行某技能后，L2 做一次轻量的 snapshot 校验：

> L2 snapshot-only：传入时跳过 listAvailable I/O。

关键词是 **snapshot-only** 和 **软校验**。它复用 L1 已经加载的候选快照，**跳过重复的 listAvailable I/O**，只做契约快照检查。而且 L2 **禁止**调用 `SkillRegistry.resolve` 或 `SkillReadinessGate.check`——那些是 L3 的职责。

**L1 产生的候选快照会被注入决策上下文，L2 必须复用它，禁止重复 collect。** 避免"决策查一遍、规划又查一遍"的重复 I/O。

### L3：运行时的硬性闸门（最贵，最严格）

L3 发生在技能真正执行前，是**硬拦**——不通过直接拒绝，不 fallback、不推断。

```typescript
// SkillReadinessGate.ts（第 92-343 行）
export class SkillReadinessGate {
  async check(input: SkillReadinessCheckInput): Promise<SkillReadinessResult> {
    // ...
  }
}
```

L3 检查运行时资源是否真的就绪——凭证是否解析得出、工作站是否可用。最典型的是**凭证解析**：

```typescript
// SkillReadinessGate.ts（第 250-325 行）
// manifest 声明 requiresCredentials 时：
// canonical 字段（scopeProjectId / skillTargetId / artifactDigest）任一缺失
// 即 skill_preflight_credential_canonical_missing 失败
// 绝不用原始 projectId fallback
```

**任何一个 canonical 字段缺失，直接失败，绝不退而求其次用原始 projectId 兜底。** "宁可不跑，也不跑错"——一个带着错误凭证跑起来的技能，比跑不起来的技能危险得多。

### 三层的核心：尽早失败

```
[决策阶段] --L1: 能力清单(仅加载正文)--> [规划阶段]
[规划阶段] --L2: snapshot 软校验(复用 L1 快照)--> [执行前]
[执行前]  --L3: 硬性闸门(凭证/资源/工作站)--> [执行]
```

如果某技能注定会因为缺凭证而失败，最好在 L1 就不让它进候选，最差也要在 L3 拦住——**绝不能让它跑到一半才崩**。每一层都比下一层便宜，这是分层的全部意义。

---

## 治理：让 AI 在企业里"可控地做事"

技能治理是企业落地的核心。一个能调部署工具、能改数据库、能发企微消息的技能，没有治理就是悬在头上的风险。

### persona_eligible：分身技能的默认继承

`skill_artifact` 有个 `persona_eligible` 字段，它是分身（员工个人副本）技能治理的关键。背后是一条强制设计原则：

> 项目空间新增或变更能力时，默认同时对分身生效。不适用分身的必须显式加入排除名单。禁止为分身新建平行实现。

意思是：**给"阿码"加新技能，阿码的数字分身默认也获得，除非显式设 `persona_eligible=false`。** 不允许为分身单独维护平行技能栈。

为什么这么严格？因为"平行实现"是大型系统腐化的头号来源。一旦为分身单独写一套技能，两套就会逐渐漂移，最终变成两个产品。**强制同源继承 + 显式排除名单，从架构上杜绝腐化。**

### 白名单 + 角色默认绑定 + 契约覆盖

- **角色默认绑定**：阿码默认会代码评审，这在数据层面成立，不靠 prompt 写死。
- **项目白名单**：特定项目可以收紧可用技能——对外项目禁用内部运维技能。
- **契约覆盖**：项目可以 merge 微调技能契约，不必 fork 技能本身。

### 市场 + 哈希校验

技能是带哈希的产物，支持市场安装。安装链路上 `package_sha256` 校验是供应链安全底线——**被篡改的技能包，哈希对不上，安装直接失败。**

---

## 学习闭环：技能越用越好

技能执行的过程被完整记录，用于持续优化。这不是简单日志，而是数据飞轮。

三张表构成技能进化的数据基础：

- **SkillTrace**：原始轨迹。每次执行的工具调用、迭代轮数、成功与否、耗时。带 `skillContentHash` 和 `toolSetHash`——区分"同技能不同版本"的表现。
- **SkillExecGuide**：蒸馏出的最佳实践。核心工具、参数规则、常见坑，还带 p50/p95 耗时和置信度。
- **SkillEscapeEvent**：逃逸事件。技能偏离预期、要求扩展工具的情况——识别"需要改进的技能"。

飞轮的转动靠两个异步 Worker：

```typescript
// agents/harness/learning/distillation/consumers/distillConsumer.ts（第 22-70 行）
// BullMQ Worker，DISTILL_QUEUE，concurrency=1

const traceRows = await prisma.skillTrace.findMany({
  where: { roleId, skillName, skillSource, success: true },  // 只看成功轨迹
  orderBy: { createdAt: 'desc' },
  take: 10,                                                  // 取最近 10 条
});

const guide = await distiller.distill({
  skillContent, traces, visibleToolNames, ...
});  // 调 mini 模型蒸馏，消毒后写 SkillExecGuideCache
```

蒸馏逻辑：

1. **只看成功轨迹**（`success: true`）——失败的不参与最佳实践提炼。
2. **取最近 10 条**——避免历史过时数据淹没近期改进。
3. **调 LLM 蒸馏**（mini 模型，`purpose='skill_distill'`）——把 10 条轨迹 + 技能正文提炼成指南。
4. **消毒后落库**——`SkillGuideSanitizer` 移除敏感工具、过滤不当内容（记录 `removedTools`/`wasFiltered`）。

**技能用得越多，指南越准，执行越好。** 这是个正向飞轮。

---

## Schema 漂移：一个容易被忽视的工程问题

最后讲一个隐蔽但致命的问题。

技能定义会随版本演进。`skill_artifact.manifest` 的结构会变。但数据库里的旧技能工件，和代码里的新 schema，可能对不上——比如你发了新代码，但数据库迁移没跑成功。

如果没有检测，这表现为各种诡异错误——某技能能跑、某不能、错误信息看不出根因。

我们的做法是**启动期探测 + 运行期识别 + 整体降级**：

```typescript
// business/domain/skillManagement/skillArtifactSchemaDrift.ts（第 56-96 行）
export async function warnSkillArtifactSchemaDriftOnStartup(): Promise<void> {
  const rows = await prisma.$queryRaw`
    SELECT column_name FROM information_schema.columns
    WHERE table_schema = 'public' AND table_name = 'skill_artifact'
      AND column_name IN ('project_name', 'trust_level', 'manifest', 'enabled', ...)`;
  const missing = SKILL_ARTIFACT_REQUIRED_COLUMNS.filter((c) => !present.has(c));
  if (missing.length === 0) return;
  logger.warn({ event: 'skill_artifact_schema_drift', missingColumns: missing },
    `[Startup] skill_artifact 缺列; skill 相关 API 将返回 503，请执行 npx prisma migrate deploy`);
}
```

它直接查 `information_schema.columns`，而不是依赖 Prisma 类型——因为要检测的是**数据库实际状态**和代码定义是否一致。一旦检测到漂移，技能 API **整体返回 503**，而不是让每个请求各自崩溃。

**把"随机崩溃"变成"启动告警 + 安全降级"，是工程成熟度的体现。**

---

## 给后来者的几条总结

1. **技能是带版本/哈希/契约的产物**，不是 prompt 不是函数。需要动态加载/版本/热更新，从一开始就用产物模式。
2. **provides/consumes 契约让技能串联变成声明数据流**。onMissing 四态给精细的缺失处理；契约可 merge 覆盖但不改正文。
3. **L1/L2/L3 就绪检查尽早失败**。L1 决策阶段出能力清单（便宜）、L2 规划阶段 snapshot 软校验（复用快照）、L3 执行前硬闸门（贵但严格）。
4. **分身技能同源继承**。persona_eligible 默认继承，禁止平行实现——从架构上防腐化。
5. **治理靠组合拳**。白名单 + 角色默认绑定 + 契约覆盖 + 市场哈希校验。
6. **技能越用越好的飞轮**。SkillTrace 原始轨迹 → SkillExecGuide 蒸馏指南 → SkillEscapeEvent 逃逸信号。只看成功、只取近期、消毒后落库。
7. **Schema 漂移三层防御**。启动期探测 + 运行期识别 + 整体 503 降级。把随机崩溃变启动告警。
8. **流程编排层强校验契约**。missing_skill_provides/consumes 编译期就报错，别留到运行期。

技能系统是 WinMatrix 让 AI"能做事"且"可控地做事"的核心机制。它把零散的工具调用，组织成了有版本、有契约、有治理、能学习的能力单元。**这才是"数字员工"和"聊天机器人"的本质区别。**

---

> **下一篇**：[《让多个 AI 员工协作：协作编排与"会干活"的团队》](./07-multi-agent.md)——单个技能会了，多个员工怎么配合？协作编排怎么做。
