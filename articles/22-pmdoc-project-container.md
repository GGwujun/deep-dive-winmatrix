# PMDOC 项目容器：项目是协作的物理容器

> 这是《WinMatrix 开发经验文集》第 22 篇。讲一个贯穿 WinMatrix 全系统的设计决策：为什么我们把"项目"设计成一个文件系统容器？PMDOC 这个目录约定是怎么来的、解决了什么问题？这篇讲透"项目即容器"的设计哲学。代码来自 WinMatrix 后端真实实现。

如果你用过项目管理软件，"项目"这个词大概意味着：一个名字、一段描述、一个任务列表、几个成员。它是个**数据库实体**——一行记录，外加一些关联表。

WinMatrix 早期也是这么做的。但跑着跑着发现一个问题：AI 员工要在这个项目里"干活"——写文档、记会议纪要、存决策记录、维护成员画像、积累项目经验——这些东西放哪儿？放数据库？每类一个表？还是放文件系统？如果放文件系统，怎么组织、怎么让 AI 找到、怎么和数据库里的项目实体关联？

我们最终的选择是：**把"项目"设计成一个文件系统容器**。一个项目不仅有数据库里的元数据（名字、成员、配置），还有一片**物理的文件空间**——PMDOC（Project Management DOCument）文档树。AI 写的所有东西都落在这棵树上，就像人类员工在公司的共享文件夹里干活一样。

这篇讲这个设计是怎么落地的，以及它为什么比"全放数据库"或"全放对象存储"更合适。

---

## 项目 = 双路径容器

先看数据模型。WinMatrix 的 `projects` 表里有两个关键字段：

```
projects 表（关键字段）：
  id, name, code, description
  pmdocPath        ← PMDOC 文档树根路径
  teamtaskPath     ← 任务/团队数据根路径
  templateId, tags[], ...
```

一个项目挂**两条物理路径**：

- **`pmdocPath`**：PMDOC 文档树。AI 和人类产出的所有文档——PRD、技术方案、会议纪要、记忆文件、项目规则——都存这里。
- **`teamtaskPath`**：任务和团队数据根。任务清单（tasks 表镜像）、团队成员（members）的结构化数据存这里。

为什么要分两条路径，而不是塞进一个目录？因为它们的**性质完全不同**：

- PMDOC 是**文档空间**——非结构化、文件为主、人类可读、AI 可读写、频繁被 RAG 索引。
- teamtask 是**数据空间**——结构化、数据库表为主、机器读写为主、人类通过 UI 看。

把它们放在不同的路径（和不同的存储子系统），各自用最适合的方式管理。文档用文件系统 + RAG，任务数据用数据库 + 同步。强行合并只会让两边都不舒服。

**教训："项目"不只是数据库里的一行记录，它是"元数据 + 物理存储"的组合体。** 纯数据库方案擅长结构化数据，但不擅长文档；纯文件系统方案擅长文档，但不擅长查询和关联。双路径容器让两种存储各司其职，项目实体把它们绑在一起。

---

## PMDOC 的固定阶段目录约定

PMDOC 文档树不是随便长的，它有一套**固定的阶段目录约定**。最核心的是 `00_共享/` 这个目录：

```typescript
// src/business/domain/project/pmdocSharedStage.ts（全文，第 1-22 行）
/**
 * PMDOC「共享」阶段目录名（磁盘路径段）。
 *
 * 长期记忆 chokidar 仅监听 pmdoc 下该目录；
 * memory_write、项目准则、模板首阶段 path 与此对齐。
 */
export const PMDOC_SHARED_STAGE_DIR = '00_共享';

/** 相对 pmdoc 根：每日记忆目录 */
export const PMDOC_MEMORY_DIR_RELATIVE = `${PMDOC_SHARED_STAGE_DIR}/memory`;

/** 相对 pmdoc 根：项目准则文件 */
export const PMDOC_GUIDELINES_RELATIVE = `${PMDOC_SHARED_STAGE_DIR}/agentrule.md`;

/** TemplatePathResolver 等使用的兜底子路径 */
export const PMDOC_FALLBACK_OTHER_RELATIVE = `${PMDOC_SHARED_STAGE_DIR}/其他`;

/** 成员记忆文件名前缀（拼接成员名后加 .md） */
export const PMDOC_MEMBER_MEMORY_PREFIX = '成员记忆_';

/** 项目记忆文件名 */
export const PMDOC_PROJECT_MEMORY_FILENAME = '项目记忆.md';
```

这套约定定义了一个项目的 PMDOC 树大致长这样：

```
<项目>/
├── 00_共享/                      ← 共享阶段，长期记忆 + 项目规则
│   ├── memory/
│   │   ├── 项目记忆.md            ← 项目级的长期记忆
│   │   ├── 成员记忆_张三.md        ← 张三的个人画像记忆
│   │   └── 成员记忆_小品.md        ← 小品的个人画像记忆
│   ├── agentrule.md              ← 项目准则（AI 必须遵守的规则）
│   └── 其他/                     ← 兜底目录
├── 01_需求/                       ← 需求阶段文档
├── ...
└── 99_进度管理/                   ← 进度管理文档
```

几个设计要点：

### 为什么用"数字前缀"命名阶段？

`00_共享`、`01_需求`、`99_进度管理`——数字前缀是为了**强制排序**。文件系统默认按字母排序，纯中文名排序是乱的。加数字前缀后，阶段目录在文件管理器、在 `readdir` 结果里都按项目流程顺序排列。`00` 永远在最前面（共享的东西最先看到），`99` 永远在最后（进度管理是收尾）。

### `00_共享` 为什么是特殊目录？

注释里那句"长期记忆 chokidar 仅监听 pmdoc 下该目录"是关键。PMDOC 整棵树可能很大（几十上百个文档），但**需要实时监听变更的只有 `00_共享/`**——因为只有这里的文件（记忆、规则）会影响 AI 的行为，需要变更时及时感知。其他阶段目录的文档是"按需读取"的（AI 用工具主动 read），不需要实时监听。

这种"全量文档树 + 局部实时监听"的分层，平衡了"实时性"和"性能"——监听一整个大树的文件变更是很贵的（inotify 句柄、事件风暴），只监听关键子目录，既保证记忆/规则的实时更新，又不被无关文档的频繁编辑拖累。

**教训：文件系统作为存储时，要用"目录约定"建立结构，而不是让文件随便堆。** 约定要包含排序（数字前缀）、分类（阶段目录）、特殊语义（共享目录）。同时，监听/索引这类昂贵操作要按"子目录"分级——只对真正需要实时性的部分做监听，其余按需读取。

---

## 记忆文件：成员记忆和项目记忆的分离

`00_共享/memory/` 目录下，记忆文件分两种：

```
00_共享/memory/
├── 项目记忆.md            ← 项目级长期记忆（全局）
├── 成员记忆_张三.md        ← 张三的个人画像
└── 成员记忆_小品.md        ← 小品的个人画像
```

- **项目记忆**（`项目记忆.md`）：整个项目共享的长期记忆。比如"这个客户对交付时间很敏感"、"上次方案被否是因为预算"。
- **成员记忆**（`成员记忆_张三.md`）：每个成员的个人画像。比如"张三是技术负责人，擅长后端"、"张三喜欢简洁的汇报风格"。

为什么要分？因为它们的**作用域和更新频率不同**。项目记忆是全局的、更新慢（项目级的事实不会天天变）；成员记忆是按人的、更新相对频繁（每次和某成员交互后可能要更新他的画像）。

这种分离让 AI 在注入上下文时可以按需选择——讨论项目方向时主要注入项目记忆，和某个成员协作时主要注入该成员的记忆。如果全塞一个文件，既臃肿又难以选择性注入。

记忆文件的扫描逻辑在 `MemoryFileScanner`：

```typescript
// src/agents/core/ai-kernel/context/adapters/MemoryFileScanner.ts（第 4-8 行，文件头）
/**
 * 记忆文件扫描器
 *
 * 扫描项目 pmdoc 下 00_共享/memory/ 目录中的长期记忆文件（成员记忆、项目记忆），
 * 生成可注入系统提示词的文件列表段落，告知 LLM 可用 read_file 读取完整内容。
 */
```

注意它的策略：**扫描生成"文件列表"注入提示词，而不是把整个文件内容塞进去**。记忆文件可能很长（项目跑半年，项目记忆积累几十KB），全塞进 prompt 会爆 token。所以只注入一个"清单"——告诉 LLM"有这些记忆文件，你需要的话用 read_file 读"。LLM 按需读取，这才是 token 经济的做法。

扫描结果还带缓存（`scanCache`，TTL 可配），避免每次注入上下文都重新扫描文件系统。

**教训：记忆不是"越大越好"地往 prompt 里塞，而是"分层按需"。** 先给 LLM 一个目录（清单），让它自己决定读哪个。这比"一次性塞所有记忆"既省 token 又精准——LLM 只读和当前任务相关的记忆，不会被无关记忆干扰。

---

## agentrule.md：项目级 AI 准则

`00_共享/agentrule.md` 是另一个特殊文件——**项目级 AI 准则**。它存的是"在这个项目里，AI 必须遵守的规则"。比如：

- "本项目的所有代码必须经过安全审查才能提交"
- "给客户发邮件前必须经过项目经理确认"
- "技术方案文档必须用公司模板"

这个文件在 AI 决策的**很早阶段**就被加载。看决策引擎里的引用：

```typescript
// src/agents/core/agent/decision/types.ts（第 403 行附近）
/**
 * 预读的 00_共享/agentrule.md 全文（由 UnifiedDecisionMaker 在策略执行前从磁盘加载），
 */
```

决策提示词里也反复提到它：

```typescript
// src/agents/core/agent/decision/prompts/decisionPlannerPrompt.ts（第 24-25 行）
// - 项目级规则、路径、技能目录、非敏感集成说明等统一以系统提示词中的
//   `## 项目上下文` / `00_共享/agentrule.md` 为准。
// - 不要把这类信息拆成"读取项目记忆"或"写入项目记忆"的步骤；
//   若用户要调整项目规则，应描述为更新 `00_共享/agentrule.md`，
//   而不是写 `00_共享/memory/项目记忆.md`。
```

注意最后两句——它明确告诉 LLM **"项目规则"和"项目记忆"是两个不同的东西，不要搞混**：

- 调整项目规则 → 更新 `agentrule.md`
- 积累项目经验/事实 → 写 `memory/项目记忆.md`

这个区分非常关键。规则是"必须遵守的约束"（normative），记忆是"发生过的事实"（descriptive）。如果把两者混在一个文件里，LLM 会搞不清哪些是"必须遵守的"哪些是"参考信息"，行为就会漂移。

**教训：AI 的行为约束（规则）和 AI 的知识（记忆）要分文件存。** 规则文件是强制的、在决策早期加载、影响每个动作；记忆文件是参考的、按需读取、辅助理解。混在一起会让 LLM 分不清"必须遵守"和"仅供参考"的边界。

---

## 项目上下文的分层注入

讲完存储，讲讲项目信息怎么进到 AI 的上下文里。WinMatrix 用的是一个**上下文源（ContextSource）** 体系，项目上下文是其中一个源：

```typescript
// src/agents/core/ai-kernel/context/adapters/sources/ProjectContextSource.ts（第 33-47 行）
export interface ProjectInfo {
  name?: string;
  phase?: string;             // 项目当前阶段
  members?: string[];         // 成员列表
  description?: string;       // 一行摘要
  body?: string;              // 预渲染的「项目上下文正文」
  completeness?: ProjectContextCompleteness;
  fetchFailureReason?: string;
}

export class ProjectContextSource implements ContextSource {
  readonly id = 'project_context';
  readonly priority = 4;
}
```

注意 `ProjectInfo` 里既有**结构化字段**（name、phase、members、description），又有**预渲染正文**（body）。这两者是分层注入的：

- **结构化字段**：用于决策阶段。比如路由决策要看"这个项目有哪些成员"来决定该找谁，这种判断用结构化数据高效。
- **预渲染正文**（body）：用于 prompt 组装。一段精心组织的文本，包含项目背景、当前阶段、关键约束，直接拼进系统提示词。注释说它"与 `promptBuilder.buildProjectContextBody` 字节级一致"——意思是这个 body 是预先渲染好的、确定性的，不是每次拼随机的。

`completeness` 字段也值得一提——它记录"项目上下文的完整度"（哪些类别有、哪些必需类别缺失）。这让 AI 知道"我对这个项目的了解够不够"，缺失关键信息时可以主动询问，而不是在信息不足的情况下硬猜。

`priority = 4` 表示项目上下文在所有上下文源里的优先级。当 token 预算紧张、需要裁剪上下文时，优先级高的源先保留。项目上下文 priority=4，比一般记忆（优先级可能更低）重要——项目的基本信息是每次对话都需要的基础上下文。

**教训：上下文注入不是"一股脑全塞"，而是分层、分优先级、按场景。** 结构化数据给决策用，预渲染文本给 prompt 用；不同源有不同的优先级，token 紧张时按优先级裁剪。这种精细化管理的本质是"token 经济"——在有限的上下文窗口里，塞最有价值的信息。

---

## PMDOC 路径的规范化

PMDOC 是文件系统，就会遇到文件系统经典的问题——**路径不规范**。Windows 用 `\`，Linux 用 `/`；有的路径带前缀，有的不带；API 返回的相对路径和前端传的可能格式不一样。

WinMatrix 有专门的路径工具处理这些：

```typescript
// src/business/domain/document/utils/pmdocPathUtils.ts（第 4-17 行）
/** 规范化后的 pmdoc API 相对路径；无法推导时返回 null */
export function effectivePmdocApiPath(
  relativePath: string | null | undefined,
  apiRelativePath: string | null | undefined,
): string | null {
  const api = apiRelativePath?.trim();
  if (api) {
    return api.replace(/\\/g, '/');    // Windows \ → /
  }
  const pathStr = (relativePath ?? '').replace(/\\/g, '/');
  const segments = pathStr.split('/').filter(Boolean);
  const under = segments.length <= 1 ? segments : segments.slice(1);  // 去掉首段
  // ...
}
```

两个关键操作：

1. **斜杠统一**（`replace(/\\/g, '/')`）：把 Windows 的反斜杠统一成正斜杠。WinMatrix 跑在 Windows 和 Linux 两个环境上，但 PMDOC 路径在系统内部必须统一格式，否则同一个文档在不同环境下的路径对不上。
2. **首段处理**（`segments.slice(1)`）：PMDOC 路径有时带项目根目录名作为前缀，有时不带。这个函数把前缀去掉，只保留 PMDOC 内部的相对路径。

还有一个 `canonicalPmdocTreePath` 函数，处理更复杂的场景——同一个文档可能有多种路径表达（有的带阶段目录、有的不带），它要推导出一个"规范路径"用于树聚合和去重。这类路径规范化的代码看着琐碎，但它是"文件系统作为存储"必须付出的代价——**一旦路径不统一，同一个文档会被当成两个，索引重复、检索错乱**。

**教训：用文件系统做存储，路径规范化是第一要务。** 跨平台（`\` vs `/`）、前缀差异、相对 vs 绝对——这些差异必须在系统边界（入口处）统一掉，让内部代码只面对一种规范路径。否则路径混乱会像幽灵一样在各种角落出没。

---

## tasks 表：外部 PM 系统的镜像

最后讲讲 `teamtaskPath` 对应的任务数据。WinMatrix 的 `tasks` 表不只是内部任务，它还**镜像外部 PM 系统**（比如 Microsoft Project、Jira）：

```
tasks 表（关键字段）：
  projectId, taskId, taskName, ownerId
  startDate, endDate, duration, status, completion
  source           ← 任务来源（internal / external）
  externalId       ← 外部系统的任务 ID
  externalUrl      ← 外部系统的链接
  lastSyncedAt     ← 最后同步时间
```

`source`、`externalId`、`externalUrl`、`lastSyncedAt` 这几个字段说明：tasks 表是**外部 PM 系统的本地镜像**。外部系统的任务通过同步写进来，AI 可以读这些任务（理解项目进度），但不直接改——改了也要同步回外部系统，复杂且容易冲突。

这种"镜像"设计让 AI 能利用已有的项目管理数据（不用让用户在 WinMatrix 里重新建一遍任务），同时不试图"替代"专业 PM 工具。WinMatrix 聚焦在"AI 协作"，任务管理让专业工具做，自己做镜像就够了。

`teams` 表和 `projects` 是一对一（主键就是 projectId），`members` 是软归属（projectId 可空，支持全局数字员工不属于特定项目）。这种关系设计让"项目容器"既能装人类成员，也能裂数字员工，还能装外部同步来的任务。

**教训：企业系统集成要做"镜像"而不是"替代"。** 专业的 PM 工具、通讯工具、文档工具已经存在且好用，AI 平台要做的是"接入并利用它们的数据"，而不是"重新发明一遍"。镜像表 + 同步机制，是集成的最低成本方案。

---

## 一个项目的完整物理形态

把上面所有点串起来，一个 WinMatrix 项目在物理上长这样：

```
                        projects 表（元数据）
                        ├── id, name, code
                        ├── pmdocPath ──────────┐
                        ├── teamtaskPath ───────┤
                        ├── templateId          │
                        └── tags[], ...         │
                                                │
                    ┌───────────────────────────┘
                    ▼
    ┌───────────────────────────────────────────────┐
    │            文件系统 / 对象存储                  │
    │                                               │
    │  pmdocPath/                  teamtaskPath/    │
    │  ├── 00_共享/                （任务/团队数据   │
    │  │   ├── memory/              镜像，DB 同步）  │
    │  │   │   ├── 项目记忆.md                      │
    │  │   │   └── 成员记忆_*.md                    │
    │  │   ├── agentrule.md        ← chokidar 监听  │
    │  │   └── 其他/               （仅 00_共享/）  │
    │  ├── 01_需求/                                 │
    │  ├── ...                                      │
    │  └── 99_进度管理/                              │
    │                                               │
    │  RAG 索引覆盖全树          AI 按需 read_file   │
    └───────────────────────────────────────────────┘
```

项目实体是"胶水"——把元数据、文档树、任务数据粘在一起。AI 在这个容器里干活：读 agentrule.md 知道规则、扫 memory/ 知道记忆、按需 read_file 读阶段文档、写产出到对应阶段目录。人类也在这个容器里干活：用文件管理器看文档、用 UI 看任务、编辑 agentrule.md 调整 AI 行为。

**项目不只是数据，项目是 AI 和人类共同的工作空间。**

---

## 给后来者的几条总结

1. **把"项目"设计成物理容器，不只是数据库实体。** 元数据放 DB，文档放文件系统，任务放镜像表，一个项目实体把它们绑在一起。
2. **双路径分离文档空间和数据空间。** PMDOC（非结构化文档）和 teamtask（结构化数据）性质不同，强行合并只会互相拖累。
3. **用"数字前缀 + 阶段目录"建立文件结构约定。** 强制排序、清晰分类，让文件系统也有结构。
4. **chokidar 只监听关键子目录（00_共享/），不全树监听。** 平衡实时性和性能。
5. **记忆文件按"项目级"和"成员级"分离。** 作用域不同、更新频率不同，分文件存。
6. **记忆注入给"清单"而非"全文"**。LLM 按需 read_file，省 token 又精准。
7. **规则（agentrule.md）和记忆（memory/）分文件存。** 规则是约束，记忆是事实，混了会让 AI 行为漂移。
8. **上下文注入分层分优先级**。结构化字段给决策，预渲染正文给 prompt，token 紧张按优先级裁剪。
9. **路径规范化在系统边界统一掉**。跨平台斜杠、前缀差异，进了系统就只能有一种规范路径。
10. **外部系统集成用镜像而非替代。** 接入已有工具的数据，别重新发明。

"项目即容器"是 WinMatrix 最核心的设计决策之一。它让 AI 不是浮在数据库上的抽象实体，而是真正"住进"了项目里——有自己的办公桌（PMDOC 文件）、有同事（成员画像）、有规章制度（agentrule.md）、有项目记忆（memory/）。这种"具身性"让 AI 协作从概念变成了可操作的工程实践。

---

> **下一篇**：[《幂等设计全景：从 role_inbox 到 CodingTask 到 flow_instruction》](./23-idempotency-panorama.md)——WinMatrix 里幂等不是一处设计，而是五种形态。从收件箱到编码任务到流程指令，讲透幂等的全景。
