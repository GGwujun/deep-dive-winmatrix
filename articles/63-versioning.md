# 版本化：技能/流程/镜像怎么管版本才能不串

> 这是《WinMatrix 开发经验文集》第 63 篇。前 40 篇里"版本"这个词反复出现——第 38 篇 schema 漂移、第 06 篇技能系统、第 20 篇流程编排、第 11 篇工作站镜像——但都是各讲各的。这篇把"版本化"作为一根线系统讲：一个 AI 平台上，技能、流程、镜像、配置这些"会变的东西"怎么管版本，才能做到可复现、可回滚、可审计。核心思路一句话：**不可变发布版 + 内容指纹（checksum/digest/sha256），让每个版本有唯一身份，改一个字节指纹就变**。版本不是个数字，是可复现性的基础。

先讲没有版本化会怎样。

**场景 1：技能升级后线上炸了，回不去。** 技能 v1 跑得好好的，有人改了技能正文发了 v2，没保留 v1。v2 上线后发现有个边界条件没处理，线上大面积失败。想回滚到 v1——发现 v1 已经被覆盖了，只能紧急修 v2 发 v3。这中间几小时线上瘫着。

**场景 2：同样的配置跑出不同结果。** "上周这个流程跑得好好的，今天同样的输入结果不一样了。"排查发现：流程定义在 DB 里被直接改了，没有版本记录，改了什么、什么时候改的、谁改的一概不知。连"到底是不是被改过"都说不清。

**场景 3：镜像被悄悄换了。** 编码工作站用的 engine image，本来是 v1.2 的镜像，有人重新构建了一次 v1.2 tag（同名），镜像内容变了但 tag 没变。所有用 v1.2 的工作站行为都变了，但看 tag 还以为是同一个镜像。

这三个场景的共同根因：**"会变的东西"被当成"不可变"在用**。技能正文、流程定义、镜像——这些都可能被改、被重建，但系统假装它们是固定的，只用名字/tag 引用。一旦内容变了但名字没变，引用方就跟着错。

WinMatrix 的应对是系统的版本化：**每个会变的东西都建"不可变发布版"模型 + 内容指纹**。技能用 version + sha256、流程用 template_version + checksum、镜像用 digest。指纹变了就是新版本，不能偷偷改。这篇逐类拆。

---

## 技能版本化：skill_artifact 的 version + sha256

技能是数字员工的能力单元（第 06 篇）。一个技能的正文（markdown 指令）、manifest（凭证需求等）、trust_level、persona_eligible 这些都可能变。怎么管版本？

看 `skill_artifact` 模型（schema.prisma:1273-1305，核实报告 ch13-17）：

```prisma
model skill_artifact {
  id                    String   @id @default(cuid())
  name                  String                     // 技能名
  version               String                     // 版本号
  artifact_path         String   @map("artifact_path")
  sha256                String                     // 正文内容指纹
  origin                String
  scope                 String                     // 范围（global/project/personal）
  project_name          String?  @map("project_name")
  trust_level           String   @map("trust_level")
  manifest              Json                       // 声明的凭证需求等
  enabled               Boolean  @default(true)
  package_storage_key   String?  @map("package_storage_key")
  package_sha256        String?  @map("package_sha256")  // 打包内容指纹
  persona_eligible      Boolean  @default(true) @map("persona_eligible")
  @@unique([name, version, scope, project_name])
  @@map("skill_artifact")
}
```

几个关键的版本化设计：

**唯一键含 version**。`@@unique([name, version, scope, project_name])`——同一个技能名在同一 scope/project 下，version 必须唯一。这意味着 **v1 和 v2 可以共存**，不是 v2 覆盖 v1。升级到 v2 后，v1 还在 DB 里，随时能回滚。

**两个 sha256 指纹**。`sha256` 是正文内容指纹，`package_sha256` 是打包内容指纹。这两个是"内容真源"——改一个字节，指纹就变。版本号是给人看的（v1.0、v1.1），指纹是给机器校验的（不一样就是不一样）。

**为什么需要指纹而不仅是版本号？** 因为版本号是人填的，可能填错、可能重复构建。指纹是算出来的，客观唯一。凭证绑定时锁的是 `artifactDigest`（第 60 篇讲过）——指纹，不是 version 字符串。这样即使有人手滑把 v1 的正文改了又重新发成 v1（理论上唯一键会冲突，但如果绕过校验直接改库），指纹对不上，凭证绑定自动失效，技能跑不起来——fail-closed。

**漂移检测三层**（核实报告 ch13-17 主题 1，第 38 篇详讲）。启动期查 information_schema 看 skill_artifact 表有没有缺列（`skillArtifactSchemaDrift.ts:56-83`），运行期识别 Prisma P2021/P2022 错误，HTTP 503 兜底。这防的是"代码发了但迁移没跑"的 schema 漂移——版本化不只是数据版本，还包括 schema 版本。

**persona_eligible 是分身可用性的版本级控制**。`persona_eligible Boolean @default(true)`，默认对分身生效（第 06 篇讲过分身继承）。这个字段是 per-artifact（每个技能版本一条），意味着可以"v2 对分身不可用、v1 可用"——版本级的精细化控制。

---

## 流程版本化：template → template_version → run 三层

流程编排（第 20 篇讲过）的版本化更复杂，因为流程有"草稿"和"发布"两种状态。WinMatrix 用三层模型（核实报告 ch18-22 主题 3）：

```
flow_template（草稿/root）
   │  draftDefinitionJson，可随意改
   │
   ▼ 发布（不可逆）
flow_template_version（不可变发布版）
   │  definitionJson + checksum，发布后不能改
   │
   ▼ 实例化
flow_run（实例）
   关联 templateVersionId，一次运行一条
```

看 `flow_template_version`（schema.prisma:746-766）：

```prisma
model flow_template_version {
  id               String          @id @default(uuid())
  templateId       String          @map("template_id")
  version          Int                              // 版本号（递增）
  definitionJson   Json            @map("definition_json")  // 发布时的定义快照
  inputSchemaJson  Json            @default("{}")
  outputSchemaJson Json            @default("{}")
  publishedBy      String?         @map("published_by")
  publishedAt      DateTime        @default(now()) @map("published_at")
  checksum         String                          // 内容校验和
  template         flow_template   @relation(...)
  runs             flow_run[]
  @@unique([templateId, version])
  @@index([checksum])
}
```

几个设计要点：

**草稿和发布版分离**。`flow_template` 存草稿（draftDefinitionJson），可以随便改、试错。`flow_template_version` 是发布版，一旦发布内容就冻结。这样"改草稿不影响线上流程"——线上跑的是某个已发布的 version，草稿改了不生效，直到重新发布成新 version。

**definitionJson 是完整快照**。发布时把草稿的 definitionJson 复制到 template_version。之后 flow_template 的草稿怎么改，已发布的 version 不动。run 关联的是 templateVersionId，跑的就是那个版本的定义。

**checksum 是内容指纹**。跟 skill_artifact 的 sha256 一个思路——版本号是人给的（1, 2, 3），checksum 是算出来的。`@@index([checksum])` 支持按 checksum 查"哪个版本是这个内容"。两个 version 不可能有相同 checksum（除非定义完全一样，那也没必要发两版）。

**publishedBy + publishedAt 留发布审计**。谁发布的、什么时候发布的，内置在版本里。不需要额外查审计日志就能知道发布历史。

**flow_run 强制关联版本**（schema.prisma:787）：

```prisma
templateVersion   flow_template_version @relation(fields: [templateVersionId], references: [id], onDelete: Restrict)
```

`onDelete: Restrict`——有 run 引用的 version 不能删。这保证"曾经跑过的版本永远能追溯"，不会被误删导致 run 变成孤儿。

**指令层也带 checksum**（flow_orchestration_instruction，schema.prisma:848）。每条指令自带 checksum 字段，保证指令内容也不被篡改。

这层的核心：**草稿可改、发布不可变、run 绑定版本**。流程定义的演进有完整时间线，任何一次发布都能追溯，任何一次运行都能还原"跑的是哪个版本的定义"。

---

## 镜像版本化：OCI digest 是真源，tag 只是展示

编码工作站的镜像（第 11 篇）是最需要版本化的东西之一——因为镜像重建后内容会变，但 tag 可以一样。WinMatrix 的做法是严格遵循 OCI 规范：**digest 是真源，tag 只用于展示**。

看 `workstation_engine_image`（schema.prisma:2182-2216，核实报告 ch13-17 主题 3）：

```prisma
model workstation_engine_image {
  id                 String   @id @default(cuid())
  engineType         String   @map("engine_type")    // claude_code|codex|hermes|openclaw
  engineName         String   @map("engine_name")
  engineVersion      String?  @map("engine_version")
  runtimeBaseImageId String?  @map("runtime_base_image_id")
  imageRepository    String?  @map("image_repository")
  imageTag           String?  @map("image_tag")      // 展示用 tag
  imageDigest        String?  @map("image_digest")   // OCI digest，真源
  imageBuildStatus   String   @map("image_build_status")  // pending|building|succeeded|failed
  @@unique([engineType, engineName, engineVersion])
  @@index([imageDigest])
}
```

base image 层也一样（schema.prisma:2150-2178）：

```prisma
model workstation_runtime_base_image {
  repository    String
  tag           String                          // 展示用
  /// OCI digest（如 sha256:abc123...）；构建成功后写入，是 Engine Image 关联的事实真源
  digest        String?
  @@unique([repository, tag])
  @@index([digest])
}
```

schema 注释（行 2158-2161）写得直白：

> 展示用 tag（如 1.0.4、latest）；生产关联以 digest 为真源
> OCI digest（如 sha256:abc123...）；构建成功后写入，是 Engine Image 关联的事实真源

**为什么 digest 是真源而 tag 不是？** 因为 tag 是可变的——同一个 `claude-code:1.2` tag，今天构建和明天构建，镜像内容可能不同（基础包更新了、依赖版本飘了）。但 digest 是镜像内容的 sha256，内容变 digest 就变。用 digest 关联，保证"我引用的这个镜像"和"实际跑的这个镜像"是同一个东西。

target image 同理（schema.prisma:2384-2385）：

```prisma
/// OCI digest（构建成功后写入）
targetDigest    String?  @map("target_digest")
```

三层镜像（base → engine → target）每层都存 digest，构建成功才写入。`imageBuildStatus` 字段标 `pending|building|succeeded|failed`——只有 succeeded 才有 digest，没构建完的镜像 digest 为空，不会被引用。

这层的核心：**镜像用 OCI digest 做 SSOT，tag 只用于人类识别**。这跟 OCI/容器生态的best practice 一致（镜像 digest pinning），避免"tag 不变内容变"的隐蔽漂移。

---

## 配置版本化：乐观锁 + 审计

配置（agent_config、tool_config 等）的版本化相对简单，但也要做。WinMatrix 用**乐观锁字段 + 审计表**（核实报告 ch04-06 主题 1）。

乐观锁的字段是 `version` 或 `workstation_config_version`（schema 里带 version 字段的 model 有 10 处）。更新时走"读当前 version → updateMany where version=? → 检查 count===1"（`adminRoleWorkstationRoutes.ts:190-247`）：

```ts
const transactionResult = await prisma.$transaction(async (tx) => {
  const current = await tx.agent_config.findFirst({
    where: { id: roleId, projectId: null },
    select: { id: true, workstation: true, workstation_config_version: true, updated_at: true },
  });
  if (current.workstation_config_version !== body.expectedVersion)
    throw new RoleWorkstationRouteError(409, CONFLICT_CODE, '配置已被其他管理员更新，请重新加载');
  const updated = await tx.agent_config.updateMany({
    where: { id: roleId, projectId: null, workstation_config_version: current.workstation_config_version },
    data: { workstation: merged, workstation_config_version: { increment: 1 }, updated_at: nextUpdatedAt },
  });
  if (updated.count !== 1)
    throw new RoleWorkstationRouteError(409, CONFLICT_CODE, '配置已被其他管理员更新，请重新加载');
  return { configVersion: current.workstation_config_version + 1 };
});
```

这是经典的乐观锁：**前端提交时带 expectedVersion，后端 updateMany where version=expectedVersion，如果别人已经改过（version 不匹配），count===0，返回 409**。两个管理员同时改同一条配置，后改的那个会被挡住，不会覆盖前者的修改。

配置变更同时写 ConfigAuditLog（第 61 篇讲过），留 before/after diff。这样配置的每次修改都有版本号递增 + 完整 diff 记录——虽然配置不是"不可变发布版"那种版本化，但通过乐观锁 + 审计，能实现"并发改不覆盖"+"改了什么能追溯"。

---

## 四类版本化放一起

把技能、流程、镜像、配置的版本化并列看：

```
版本化四类
   │
   ├─【技能】skill_artifact
   │     version + sha256 + package_sha256
   │     唯一键含 version（v1/v2 共存）
   │     凭证绑定锁 artifactDigest
   │
   ├─【流程】template → template_version → run
   │     草稿可改，发布版不可变（definitionJson 快照）
   │     checksum 内容指纹，publishedBy/At 审计
   │     run onDelete: Restrict 防版本被删
   │
   ├─【镜像】OCI digest 为真源
   │     tag 只展示，digest 才关联
   │     三层（base/engine/target）各存 digest
   │     构建成功才写 digest，buildStatus 标记
   │
   └─【配置】乐观锁 + 审计
         version/workstation_config_version 字段
         updateMany where version + count===1
         ConfigAuditLog 留 before/after diff
```

四类的共同模式：

| 类 | 版本载体 | 内容指纹 | 不可变性 |
|----|------|------|------|
| 技能 | version 字段 | sha256 | 唯一键保证 v1/v2 共存 |
| 流程 | template_version | checksum | 发布后 definitionJson 冻结 |
| 镜像 | (digest 即版本) | OCI digest | digest 内容绑死 |
| 配置 | version 字段 | （靠审计 diff） | 可改，靠乐观锁防覆盖 |

**核心是"内容指纹"这个概念**：技能有 sha256、流程有 checksum、镜像有 digest。内容指纹让"版本"不只是个人填的数字，而是客观的内容身份。改一个字节，指纹变，就是新版本，不能偷偷改了假装还是旧版。

---

## 为什么版本化是可复现性的基础

第 53 篇讲过"可重放性"——一个 bug 能不能按原样再跑一遍。版本化是可重放性的前提。

**没有版本化，"同样的输入"是个伪命题。** 你说"上次这个流程跑得好好的"，但"上次"跑的是哪个版本的流程定义？如果没有 template_version，流程定义被改过 N 次，你根本不知道"上次"是哪一版。输入再一样，流程定义不一样，结果就不一样。

**有了版本化，每次 run 都绑定明确的版本。** flow_run.templateVersionId 指向具体的 template_version，那个 version 的 definitionJson 是冻结的。要重放某次 run，取它的 templateVersionId，还原那个版本的流程定义，跑出来就是同样的结果（在 LLM 确定性的范围内，第 52 篇讲过确定性优先）。

**版本化也是回滚的基础。** v2 有 bug，回滚到 v1——因为 v1 还在 DB（唯一键允许共存），改个 enabled 标记或者引用指回 v1 就行。如果 v2 覆盖了 v1，回滚就只能"凭记忆重建 v1"，那不叫回滚叫重写。

**版本化还是审计的基础。** 第 61 篇讲的 ConfigAuditLog 留 before/after diff，本质就是"配置的版本化记录"。KernelManagementAuditLog 带 changeDiffSummary，也是治理动作的版本化。没有版本化的审计只能记"改了"，有版本化的审计能记"从什么改成什么"。

所以版本化不是"多存几个字段"的工程便利，而是**可复现、可回滚、可审计**这三件大事的共同基础。一个没做好版本化的系统，永远在"上周还好好的"和"不知道改了啥"之间挣扎。

---

## 一个细节：凭证绑定的 digest 锁与 stale 行

版本化里有个细节值得展开——技能升级后，凭证绑定怎么办？

第 60 篇讲过 ProjectSkillCredentialBinding 的唯一键含 `artifactDigest`。技能 v1 的 sha256 是 abc123，v2 是 def456。v1 时绑的凭证，绑定记录是 `(projectId, skill, abc123, API_KEY) → credentialId`。技能升到 v2，digest 变了，这个绑定对 v2 **不生效**——v2 要重新绑 `(projectId, skill, def456, API_KEY) → credentialId`。

schema 注释（行 3562-3565）说：

> digest 变化时旧行保留用于显示 stale 和审计，但 readiness 只接受当前 digest 的精确绑定。

为什么要保留 stale 旧行？两个原因：

**审计**：曾经的绑定关系是审计证据，不能删。哪天出了"凭证误用"事故，要查"v1 时绑的是哪个 credential"，stale 行就是证据。

**UX**：用户重新绑 v2 时，看到"v1 绑的是这个 credential，v2 还没绑"，可以一键沿用。如果 stale 行删了，用户得从头查"上次绑的啥"。

但 **readiness 硬约束只认当前 digest**（第 60 篇的 SkillCredentialResolutionService）。stale 行不影响运行时——L3 解析时传的是当前准入 artifact 的 digest，只匹配那个 digest 的绑定，stale 的自动被过滤。

这个细节体现的是版本化设计的精细：**历史版本的数据保留（审计/UX），但运行时只认当前版本（安全/正确）**。旧版本不是"删掉"，而是"标记为 stale 但不再生效"。

---

## 给后来者的几条总结

1. **版本化的核心是不可变发布版 + 内容指纹**。技能 sha256、流程 checksum、镜像 digest——指纹让版本有客观身份，不靠人填的数字。
2. **技能 version + sha256，唯一键含 version**。v1/v2 共存可回滚，凭证绑定锁 digest 防版本错位。
3. **流程三层：草稿 → 发布版 → run**。草稿可改、发布版冻结（definitionJson 快照）、run 绑版本且 onDelete: Restrict。
4. **镜像 OCI digest 是真源，tag 只展示**。三层镜像各存 digest，构建成功才写入。避免"tag 不变内容变"的隐蔽漂移。
5. **配置用乐观锁 + 审计**。version 字段 + updateMany where version + count===1，防并发覆盖；ConfigAuditLog 留 diff。
6. **版本化是可复现/可回滚/可审计的共同基础**。没版本化，"同样输入"是伪命题，回滚靠记忆，审计只能说"改了"。
7. **stale 旧行保留但不生效**。历史版本数据用于审计和 UX，运行时只认当前 digest。删是物理删但绑定关系留痕。
8. **漂移检测是版本化的护栏**。启动期查 schema 缺列、运行期识别 Prisma 错误、HTTP 503 兜底——防"代码发了迁移没跑"的 schema 版本错位。
9. **版本化横跨数据/schema/镜像/配置四类**。每类的机制不同（唯一键/三层模型/digest/乐观锁），但原则一致：不可变 + 内容指纹 + 可追溯。

版本化是不性感的基础设施。demo 阶段一个版本跑到底，什么版本化问题都没有；一上生产，技能升级、流程改版、镜像重建、配置并发改，没版本化就是"上周还好好的"地狱。把四类的版本化都做扎实，平台才有"敢变、能回滚、可追溯"的底气。

---

> **下一篇**：[《优雅停机：进程收到 SIGTERM 后要做完哪些事才能退》](./64-graceful-shutdown.md)——版本化保证了"变的时候不乱"，但进程停的时候呢？收到停机信号直接 exit，正在处理的请求、未落盘的数据、占着的租约怎么办。讲 WinMatrix 的分阶段停机序列。
