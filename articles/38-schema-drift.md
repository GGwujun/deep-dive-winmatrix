# Schema 漂移：代码发了但迁移没跑

> 这是《WinMatrix 开发经验文集》第 38 篇。前两篇讲的都是"运行时崩溃留下的脏状态"，这一篇讲另一个经典坑：代码和数据库对不上。代码引用了新字段，但数据库迁移还没跑——线上直接炸 `Unknown column`。讲我们怎么用三层防御（启动期探测 + 运行期识别 + HTTP 503 整体降级）把这个故障控制住。

这个坑，每一个用 ORM 的人都踩过，或者终将踩过。

你发了一版新代码，里面引用了一个新的数据库字段——比如 `skill_artifact` 表加了一列 `manifest`，用来存技能的 manifest JSON。代码在本地跑通了（本地库当然是最新的），CI 也过了（测试库也是最新的），发到生产——炸了。

```
PrismaClientValidationError: Unknown field `manifest` for select statement
// 或者
PrismaClientKnownRequestError: The column `manifest` does not exist in the database
```

原因很简单：**代码发了，但 `prisma migrate deploy` 没跑。** 可能是部署脚本漏了一步，可能是 DBA 还在审批，可能是 K8s 滚动更新时新代码已经接流量了但 DBA 那边的迁移窗口还没到。

这不是一个"会不会发生"的问题，是一个"什么时候发生"的问题。任何依赖 DB 迁移的发布流程，只要人和流程在，就一定会出现代码和 schema 错位的那一刻。区别只在于：你是被它打个措手不及，还是提前准备好了兜底。

这一篇讲 WinMatrix 的三层防御。

---

## 现象：发布后，所有技能相关 API 报错，但只有技能模块炸

这个坑第一次暴露，是一次技能模块的迭代发布后。

新代码引用了 `skill_artifact` 表的几个新字段（`project_name`、`trust_level`、`manifest` 等）。发版后，所有调到技能相关接口的请求开始报 500。日志里是 Prisma 的字段校验错误——代码在 select 一个数据库里不存在的列。

但诡异的是，**系统的其他部分还在正常跑**。对话、定时任务、知识库检索，都没受影响。只有技能模块炸了。

这是因为 DB 迁移是"按表"的——`skill_artifact` 的迁移没跑，只影响读写这张表的代码。其他表没事。所以故障是**模块级的**，而不是全局的。

这带来一个判断：**对于 schema 漂移，全局崩溃是不必要的，正确的做法是"漂移的模块降级，其他模块照常服务"。** 但前提是，你得先知道漂移发生在哪、能不能探测到。

---

## 根因：代码的 schema 假设 vs 数据库的真实 schema

Prisma 的工作方式是：你在 `schema.prisma` 里声明模型（包括字段），`prisma generate` 生成 Client 代码（带类型），`prisma migrate deploy` 把 schema 实际应用到数据库。

这三步的时序，在生产部署里通常是：

```
正常流程:
  代码构建 (含 prisma generate) ──→ 迁移 (prisma migrate deploy) ──→ 启动新代码

出问题的流程:
  代码构建 ──→ 启动新代码 ──✕ 迁移还没跑（或失败了）
                    ↑
              代码引用新字段，数据库还没有 ──→ 报错
```

问题出在"启动新代码"和"迁移"之间的时序错位。代码以为 schema 是最新的（因为它是按最新 schema 生成的 Client），但数据库还停在旧版本。

更阴险的是，**这种错位不一定是"迁移完全没跑"，也可能是"迁移跑了一半"**。比如一个迁移包含 5 张表的变更，跑到第 3 张表时失败了——前 3 张是新 schema，后 2 张是旧 schema。代码对前 3 张表的引用没问题，对后 2 张的引用就炸。这种"部分漂移"最难排查，因为你看到的报错是局部的、不规律的。

对技能模块而言，漂移的具体表现是：代码里的 `SkillArtifactStore`（技能产物存储）用 Prisma 查 `skill_artifact` 表时，select 了 `manifest`、`trust_level`、`project_name` 等字段，但这些列在数据库里还不存在。Prisma v7 的 Client 会在运行期校验字段存在性，于是抛 `PrismaClientKnownRequestError`，错误码是 **P2021**（表存在但字段不存在）或 **P2022**（列在 select 里但数据库没有）。

---

## 修复：三层防御

我们为 schema 漂移建了三层防御，从"尽早发现"到"优雅降级"。

### 第一层：启动期 information_schema 探测

最理想的情况是：**代码刚启动，还没接流量，就知道 schema 漂移了**，直接告警甚至拒绝启动。

```typescript
// business/domain/skillManagement/skillArtifactSchemaDrift.ts（第 56-83 行）
export async function warnSkillArtifactSchemaDriftOnStartup(): Promise<void> {
  const rows = await prisma.$queryRaw<Array<{ column_name: string }>>`
    SELECT column_name FROM information_schema.columns
    WHERE table_schema = 'public' AND table_name = 'skill_artifact'
      AND column_name IN ('project_name','trust_level','manifest','enabled','installed_at','created_at','updated_at')`;
  const present = new Set(rows.map((row) => row.column_name));
  const missing = SKILL_ARTIFACT_REQUIRED_COLUMNS.filter((column) => !present.has(column));
  if (missing.length === 0) return;
  logger.warn(
    { event: 'skill_artifact_schema_drift', missingColumns: missing },
    `[Startup] skill_artifact 缺列 (${missing.join(', ')}); skill 相关 API 将返回 503，请执行 npx prisma migrate deploy`,
  );
  // ... 设置全局漂移标记
}
```

这里用的是**直接查 `information_schema.columns`**——绕过 Prisma，直接问 PG"你这张表到底有哪些列"。这是最可靠的探测方式：不依赖代码生成的 Client（它可能已经假设了新字段），只依赖数据库的真实元数据。

几个关键点：

1. **查的是 `information_schema`，不是 Prisma。** Prisma Client 是按"代码里声明的 schema"生成的，它自己就带着漂移假设，不能用它来检测漂移。`information_schema` 是 PG 的元数据表，记录的是数据库的**真实结构**，用它做探测才是 ground truth。

2. **探测的是一张特定的关键表（`skill_artifact`）。** 我们没有对全部 157 张表都做漂移探测——那样太慢也太吵。只对"迭代频繁、漂移概率高、漂移后影响大"的关键表做。技能 artifact 表就是其中之一，因为技能模块迭代快，且它是技能体系的存储真源。

3. **发现漂移后，只告警 + 设标记，不拒绝启动。** 这是个重要的取舍。拒绝启动（fail-fast）听起来很安全，但实际操作中，你可能需要先启动服务才能跑迁移（有些迁移需要应用配合）。所以我们选择"告警 + 降级"而不是"拒绝启动"。全局漂移标记会被后面的第二、三层防御读取。

4. **日志里直接告诉运维"该跑什么命令"**：`请执行 npx prisma migrate deploy`。故障恢复的指导要写在第一时间出现的日志里，别让人去翻文档。

### 第二层：运行期识别 P2021/P2022

启动期探测覆盖了"已知关键表"，但万一漂移发生在没探测的表上呢？或者启动时没漂移、运行中某个时刻漂移了（比如 DBA 手动改了表）？

第二层防御是**在运行期识别 Prisma 的漂移错误码**，把本该是 500 的错误，转成"可控的降级"。

Prisma 对 schema 漂移有两个明确的错误码：

| 错误码 | 含义 |
|---|---|
| **P2021** | 表在数据库里不存在（代码引用了一张数据库没有的表） |
| **P2022** | 列在数据库里不存在（代码 select/write 了一列，但数据库表里没有） |

我们的基础设施层在捕获到这两个错误码时，会把它标记为"schema 漂移错误"，并触发对应的降级逻辑（而不是当成普通的数据库错误往上抛 500）。

这一层和第一层是互补的：第一层是"启动时主动探测已知风险表"，第二层是"运行时被动识别任何表的漂移错误"。两层一起，覆盖了"已知的已知"和"未知的已知"。

### 第三层：HTTP 503 整体降级

前两层都是"发现"，第三层是"处置"——发现漂移之后，受影响的模块怎么响应。

我们的策略是：**漂移模块的 API 统一返回 HTTP 503（Service Unavailable），而不是 500（Internal Server Error）。**

这个区别很重要：

- **500 意味着"我出 bug 了"**——前端会显示"系统错误"，用户以为整个系统挂了，运维可能被误叫起来查一个"全局故障"。
- **503 意味着"我这个模块暂时不可用，但我知道为什么"**——前端可以针对 503 显示"技能服务正在升级中"，其他模块照常工作，运维看一眼告警就知道是 schema 漂移，跑个迁移就好。

具体的降级响应是一个专门的函数 `sendSkillArtifactSchemaDriftReply`——当技能模块的 API 收到请求，且全局漂移标记为 true（第一层设的）或本次请求触发了 P2021/P2022（第二层识别的），就直接返回 503，body 里带一个清晰的错误码和"请等待迁移完成"的提示。

```
请求进来 ──→ 漂移标记为 true? ──是──→ 503 "skill_service_unavailable: schema drift detected"
                │ 否
                ↓
            正常处理 ──→ 触发 P2021/P2022? ──是──→ 503（同上）
                            │ 否
                            ↓
                        正常响应
```

这样，即使 schema 漂移发生在生产，用户体验也是"技能功能暂时不可用，其他都正常"——而不是"整个系统一片 500"。

---

## 三层防御的协作

把三层串起来看，它们形成了一个从"预防"到"兜底"的完整链条：

```
第一层（启动期 information_schema 探测）
   ├── 探测已知关键表的漂移
   ├── 告警 + 设全局漂移标记
   └── 触发第三层的 503 降级

第二层（运行期 P2021/P2022 识别）
   ├── 兜底第一层没覆盖的表
   ├── 兜底运行中动态发生的漂移
   └── 触发第三层的 503 降级

第三层（HTTP 503 模块降级）
   └── 受影响模块返回 503，其他模块照常服务
```

第一层负责"快"——启动就发现，省掉一段"线上炸了才发现"的窗口。第二层负责"全"——覆盖第一层没预见的漂移。第三层负责"稳"——即使漂移了，系统也不崩，只是降级。

这个设计的核心信念是：**schema 漂移是必然事件，你不能阻止它，只能控制它的影响范围。** 试图通过流程（"发布前必须跑迁移"）来消灭漂移，在现实中一定会失败——人会犯错、流程会断、时序会错位。把必然事件的危害降到可控，才是工程该做的。

---

## 教训

**第一，用数据库的元数据表（information_schema）探测真实结构，不要用 ORM Client。** ORM Client 是按你声明的 schema 生成的，它本身就是"漂移假设"的载体——你怎么能用一个带着漂移假设的工具去检测漂移？`information_schema` 是数据库自我描述的元数据，它是 ground truth。这个思路推广开去：**检测"现实 vs 假设"的偏差，必须以"现实"为探测源，不能用"假设"自证。**

**第二，关键表重点探测，不要全表探测。** 157 张表全做漂移探测，启动会变慢、告警会变吵、维护成本高。挑出那些"迭代频繁 + 影响面大 + 有明确降级路径"的关键表做探测。我们选了 `skill_artifact`，因为它最常变、且是技能体系的存储真源。**防御要有重点，全覆盖的防御往往等于没防御——因为维护它自己的成本会让团队把它关掉。**

**第三，500 和 503 是两件事，故障处置时必须分清。** 500 是"我不知道怎么了"，503 是"我知道我暂时不行"。对于 schema 漂移这种"已知的、可控的、局部的"故障，503 是正确的语义——它让前端、运维、监控都能正确理解"这是预期的降级，不是意外的崩溃"。**错误码不是给机器看的格式，是给整个协作链（前端、运维、监控、oncall）的语义信号。选错错误码，会触发错误的应急流程。**

**第四，"代码发了迁移没跑"是必然事件，为它准备降级而不是试图消灭它。** 流程上的保证（CI/CD 里加迁移检查、发布审批里加 DBA 签字）能降低频率，但不能消灭。工程上，把 schema 漂移当成一个"已知故障模式"来设计防御，比指望流程永不犯错可靠得多。**可靠性的极致不是"不出错"，而是"出错时系统行为可控"。**

**第五，日志里要写"该干什么"，不只是"出了什么"。** 我们的启动告警里直接写了 `请执行 npx prisma migrate deploy`。故障发生的第一时间，运维看到的是明确的恢复指令，而不是一个需要解读的错误码。**好的告警 = 现象 + 原因 + 恢复动作，缺一不可。** 只报现象的告警，只是制造焦虑；带恢复动作的告警，才是赋能。

---

> **下一篇**：[《WebSocket 半断开：连接没断但消息不再来》](./39-ws-half-open.md)——从"代码和数据库对不上"回到网络层：连接看起来是好的，但消息早就不通了。半断开是长连接系统最隐蔽的故障。
