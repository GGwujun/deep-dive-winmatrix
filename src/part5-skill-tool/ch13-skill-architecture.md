# 第 13 章 技能架构

> "技能让数字员工从'能聊天'变为'能做事'。"

技能（Skill）是数字员工执行具体任务的能力单元——编写 PRD、做代码评审、生成日报，每个都是一项技能。WinMatrix 的技能系统涵盖了从文件定义到运行时加载、从契约约束到治理审批的完整生命周期。本章将深入这些实现。

## 13.1 技能的本质：从文件到运行时

WinMatrix 的技能系统借鉴了 Claude Code 的 Skill 设计——技能本质上是"提示词模板 + Agent 封装"。一个技能包含：

- **定义文件**：技能的描述、参数、步骤
- **提示词模板**：指导 LLM 如何执行
- **契约声明**：`provides`/`consumes` 跨步骤输出契约
- **工件包（Artifact）**：打包后的可分发单元

技能存储在 `skill_artifact` 表中：

```prisma
// prisma/schema.prisma（第 1195-1224 行）
model skill_artifact {
  id                    String   @id @default(cuid())
  name                  String              // 技能名
  version               String              // 版本
  artifact_path         String   @map("artifact_path")  // 工件路径
  sha256                String              // 内容哈希
  origin                String              // 来源
  scope                 String              // 作用域（system/project）
  project_name          String?  @map("project_name")
  trust_level           String   @map("trust_level")    // 信任级别
  manifest              Json                // 清单（含步骤、参数）
  enabled               Boolean  @default(true)
  installed_by          String?  @map("installed_by")
  tags                  String[] @default([])
  package_storage_key   String?  @map("package_storage_key")  // MinIO/S3 存储 key
  @@unique([name, version, scope, project_name])
  @@map("skill_artifact")
}
```

唯一约束 `[name, version, scope, project_name]` 确保同一技能在不同作用域（系统级/项目级）可以有不同版本。

## 13.2 30+ 内置技能

WinMatrix 预置了 30+ 内置技能，覆盖项目全流程：

| 类别 | 技能示例 |
|------|---------|
| 项目启动 | `project_kickoff`、`wbs_generation` |
| 需求设计 | `prd_writing`、`requirement_analysis` |
| 技术架构 | `tech_solution`、`architecture_design` |
| 质量管理 | `code_review`、`test_plan` |
| 过程管理 | `daily_report`、`weekly_report`、`daily_plan` |
| 运维 | `deployment_plan`、`monitoring_setup` |
| 文档 | `document_generation`、`meeting_summary` |

内置技能通过种子脚本加载：

```bash
# scripts/seed-bundled-skills.ts
# 将内置技能定义写入 skill_artifact 表
```

## 13.3 技能契约：provides / consumes

技能之间可以通过 `provides`/`consumes` 建立数据契约——一个技能的输出可以作为另一个技能的输入：

```typescript
// 概念性（基于 src/types/domain-harness/schema/flowContractSchema.ts）
interface SkillContract {
  provides: Array<{
    key: string;           // 输出键名
    type: string;          // 数据类型
    usage: 'context' | 'selector' | 'artifact' | 'audit';
    description: string;
  }>;
  consumes: Array<{
    key: string;           // 依赖的输入键
    required: boolean;
    onMissing: 'block' | 'manual' | 'skip' | 'use_default';
  }>;
}
```

例如：

- `prd_writing` 技能 `provides: [{ key: 'prd_document', type: 'markdown' }]`
- `tech_solution` 技能 `consumes: [{ key: 'prd_document', required: true }]`

这种契约机制使得多步骤工作流可以自动串联——前序技能的输出自动注入后续技能。

### Flow I/O 契约 Schema

```typescript
// src/types/domain-harness/schema/flowContractSchema.ts（第 3-17 行）
export const FlowOutputContractSchema = z.object({
  keys: z.array(z.string().min(1)).default([]),
  artifactTypes: z.array(z.string().min(1)).default([]),
  provides: z.array(z.object({
    key: z.string().min(1),
    type: z.string().min(1).default('string'),
    usage: z.enum(['context', 'selector', 'artifact', 'audit']).default('context'),
    description: z.string().default(''),
  })).default([]),
  artifacts: z.array(z.object({
    type: z.string().min(1),
    required: z.boolean().default(false),
    description: z.string().default(''),
  })).default([]),
});
```

## 13.4 技能就绪检查：L1 / L2 / L3 边界

技能在执行前需要通过多层就绪检查：

| 层级 | 检查内容 |
|------|---------|
| **L1** | 技能是否存在、是否启用 |
| **L2** | 依赖的输入（consumes）是否满足 |
| **L3** | 运行时资源是否就绪（如工作站、凭证） |

这种分层检查避免了"执行到一半才发现缺条件"的尴尬——L1 在决策阶段就过滤掉不可用技能，L2 在组装阶段检查输入，L3 在执行前最后确认。

## 13.5 技能治理

技能治理是企业的核心需求——不能让任意技能随意执行。WinMatrix 的治理体系包括：

### 白名单

```prisma
// prisma/schema.prisma
model project_skill_whitelist {
  // 项目级技能白名单
  // 只有白名单中的技能可在该项目使用
}
```

### 角色默认绑定

```prisma
model skill_role_default_bindings {
  // 角色 → 技能的默认绑定
  // 如 tech_manager 默认绑定 code_review
}
```

### 项目覆盖

```prisma
model skill_contract_override {
  // 项目级契约覆盖
  // 项目可以自定义技能的输入/输出契约
}
```

### 技能市场

技能可以通过市场（Marketplace）安装：

```typescript
// 概念性
// 从远程 Git 仓库安装技能
// 安装时验证 sha256 哈希
// 安装后写入 skill_artifact 表
```

## 13.6 技能执行轨迹

技能执行过程被完整记录，用于学习和优化：

```prisma
// prisma/schema.prisma
model SkillTrace {
  // 技能执行轨迹
  // 记录每次技能执行的输入、输出、耗时、成功/失败
}

model SkillExecGuide {
  // 技能执行指南
  // 从历史轨迹中提炼的最佳实践
}

model SkillEscapeEvent {
  // 技能逃逸事件
  // 记录技能执行偏离预期的情况
}
```

这些数据驱动技能的持续优化——通过分析轨迹发现常见失败模式，通过逃逸事件识别需要改进的技能。

### 技能蒸馏 Worker

技能轨迹的提炼由两个 Worker 异步处理：

```typescript
// src/agents/harness/learning/distillation/consumers/
// traceExtractConsumer.ts  - 轨迹提取
// distillConsumer.ts       - 蒸馏（提炼最佳实践）
```

## 13.7 技能 Schema 漂移检测

技能定义可能随版本演进，导致旧工件与新 Schema 不一致：

```typescript
// src/startup/agents.ts（initAgentsForApi）
export async function initAgentsForApi(): Promise<void> {
  await initAgentStack({ includeMcp: !config.startup.skipMcp });
  const { warnSkillArtifactSchemaDriftOnStartup } = await import(
    '@/business/domain/skillManagement/skillArtifactSchemaDrift.js'
  );
  await warnSkillArtifactSchemaDriftOnStartup();  // 启动时检测漂移
  // ...
}
```

启动时检测 Schema 漂移并发出警告，确保管理员 aware 潜在的兼容性问题。

## 本章小结

本章深入分析了 WinMatrix 的技能架构：

1. **技能本质**：提示词模板 + Agent 封装，存储在 skill_artifact 表
2. **30+ 内置技能**：覆盖项目启动到交付全流程
3. **provides/consumes 契约**：技能间数据依赖，自动串联工作流
4. **L1/L2/L3 就绪检查**：分层过滤，避免执行到一半失败
5. **技能治理**：白名单 + 角色绑定 + 项目覆盖 + 市场
6. **执行轨迹**：SkillTrace + ExecGuide + EscapeEvent，数据驱动优化
7. **蒸馏 Worker**：traceExtract + distill 异步提炼最佳实践
8. **Schema 漂移检测**：启动时警告兼容性问题

在下一章中，我们将深入工具执行系统。
