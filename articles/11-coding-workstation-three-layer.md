# 把 LLM 关进 K8s Pod：编码工作站的三层镜像是怎么切出来的

> 这是《WinMatrix 开发经验文集》第 11 篇，也是第二批的开篇。上一篇 [《从"AI 助手"到"AI 员工"》](./10-reflections.md) 聊的是产品取舍，从这一篇起，我们重新钻进源码，讲那些"无聊但决定生死"的工程细节。

让 LLM 能真正"写代码"，是所有 AI 编码产品的核心难题之一。难不在模型，在**运行环境**。

模型给一行"修改 `src/index.ts` 的第 42 行"很简单。但这一行要落到一个真实的文件系统上、在一个有依赖的工程里跑通、产物要能取回来——这件事，纯聊天框做不到。你需要一个隔离的、可控的、可观测的执行环境。业内管这个叫 sandbox 或 coding workstation。

业界的开源方案大致两种：一种是 gVisor / Kata Containers 那种内核级隔离，另一种是社区最近推的 K8s Agent Sandbox CRD（自定义资源）思路——用一份声明式的 `Sandbox` 资源描述"我要什么环境"，Operator 给你拉起来。思路都对，但对一个已经有 K8s 集群、还要管十几种引擎和运行时的产品来说，CRD 这套偏重平台工程，不够灵活。

WinMatrix 的做法是把镜像拆成**正交的三层**，让"换引擎"和"换运行时"变成两条互不干扰的轴。这一篇就拆给你看：base、engine、component 这三层是怎么分的，为什么这么分。

---

## 先看问题：为什么不能"一个镜像塞到底"

最直觉的做法是给每种工作站打一个镜像：claude-code 工作站一个镜像、codex 工作站一个镜像、openclaw 工作站一个镜像……每个镜像里从基础系统、到引擎、到组件全装好。

听起来简单，跑起来灾难：

- **组合爆炸**。4 种引擎 × N 种运行时 × M 个组件，你要维护 4×N×M 个镜像，任何一个小改都要重打一大批。
- **升级困难**。引擎要升级（claude-code 发新版本了），但因为引擎和运行时绑在一个镜像里，你得连运行时一起重打。
- **复用不了**。同一份"Python 3.11 + Node 20"运行时，在 claude-code 工作站和 codex 工作站里各打了一份，内容漂移是迟早的事。

根因是**把两个本该正交的维度耦合在了一起**：引擎（LLM 怎么干活）和运行时（代码在什么环境里跑）。这两件事的变化频率完全不同——引擎可能一两个月一更，运行时几年不变。

正交解耦，就是把它们拆成独立的层。

---

## 三层镜像：base → engine → component

WinMatrix 的工作站镜像分三层，每层有自己的表、自己的版本、自己的唯一约束：

```
┌─────────────────────────────────────────────┐
│  component（安装脚本 + 兼容性声明）           │  最上层，变化最频繁
│  workstation_component                       │
│  installScript 全文 + checksum              │
├─────────────────────────────────────────────┤
│  engine（claude_code/codex/hermes/openclaw）│  中间层，引擎维度
│  workstation_engine_image                    │
│  @@unique([engineType, engineName, engineVersion]) │
├─────────────────────────────────────────────┤
│  base（OCI 基础镜像，digest 是真源）         │  最下层，最稳定
│  workstation_runtime_base_image              │
│  @@unique([repository, tag])                 │
└─────────────────────────────────────────────┘
```

三层各自独立版本化、独立演进。下面一层一层看。

### base：最稳定的运行时底座

base 层就是普通的 OCI 基础镜像——一个装好操作系统、系统依赖、常用运行时（Python、Node）的镜像。它的"身份"不是 tag，而是 **digest**：

```prisma
// prisma/schema.prisma（第 2151-2178 行）
model workstation_runtime_base_image {
  repository    String
  tag           String
  digest        String?   @map("digest")   // 真源，可为空（pending/failed 时）
  platforms     String?
  status        String    @default("pending")  // succeeded|failed|pending
  @@unique([repository, tag])
  @@map("workstation_runtime_base_image")
}
```

注意三个细节：

1. **`digest` 才是真源**。tag 是可变的（`python:3.11` 今天和下个月可能指向不同内容），digest 是不可变的（`sha256:abc...` 永远指向同一份）。要追溯"这个工作站到底跑的是哪个 base"，只能靠 digest。
2. **`status` 字段**。镜像不是"打了就能用"——base 镜像有个 succeeded/failed/pending 的构建状态。拉起工作站前要确认 base 是 succeeded，否则拉起来也是个坏环境。
3. **`@@unique([repository, tag])`**。repository + tag 唯一，意味着同一个 `python:3.11` 在表里只有一条记录，digest 变了就更新这条记录，而不是插一条新的。

base 层是最少动的——Python、Node 的版本几个月才动一次。把它独立出来，是为了让上面两层的变动**不必牵连重打几百兆的基础镜像**。

### engine：把"用什么引擎干活"独立出来

engine 层是中间层，管的是"这个工作站用什么 LLM 引擎干活"。WinMatrix 支持 4 种引擎：

```prisma
// prisma/schema.prisma（第 2182-2216 行）
model workstation_engine_image {
  engineType         String   @map("engine_type")
  // engineType = claude_code | codex | hermes | openclaw
  engineName         String   @map("engine_name")
  engineVersion      String   @map("engine_version")
  imageRepository    String   @map("image_repository")
  imageTag           String   @map("image_tag")
  imageDigest        String?  @map("image_digest")
  imageBuildStatus   String   @map("image_build_status")
  runtimeBaseImageId String?  @map("runtime_base_image_id")
  @@unique([engineType, engineName, engineVersion])
  @@map("workstation_engine_image")
}
```

关键是**唯一约束是 `[engineType, engineName, engineVersion]`**——也就是说，engine 是按"类型 + 名字 + 版本"独立管理的，**不绑定具体的 type_config**。

这听起来是句废话，但它的含义很重要：同一个 claude_code 1.2.3 引擎镜像，可以挂在"普通编码工作站"上，也可以挂在"SRE 工作站"上。引擎是独立可复用的资产，不是某个工作站的私有配置。

对比"一个镜像塞到底"的做法：现在 claude-code 发新版本，你只需要打一个新的 engine 镜像（改 `engineVersion`），所有用这个引擎的工作站配置都能切过去，**运行时 base 一动不动**。这就是正交解耦的收益。

### component：最上层的"装什么"脚本

component 是变化最频繁的一层。它管的是"这个工作站里要装哪些额外的东西"——某个项目要装特定的 SDK，某个团队要装内部的 CLI。component 的做法不是打镜像，而是**存安装脚本**：

```prisma
// prisma/schema.prisma（第 2287 行，workstation_component）
// 存 installScript 全文 + checksum
// 用 compatibleEngines[] 声明它兼容哪些引擎
```

两个设计点：

1. **脚本而非镜像**。组件级别的变更（加个 pip 包、装个小工具）不值得重打镜像。存脚本，启动工作站时按脚本装，既灵活又省存储。
2. **`compatibleEngines[]` 声明兼容性**。一个组件脚本可能只在 claude_code 上跑得通（比如用了 claude 特有的环境变量），用 `compatibleEngines` 数组显式声明，避免在 codex 工作站上误装导致崩溃。

### 三层怎么串起来：type→engine→skill 配置树

三层镜像之上，还有一棵配置树把它们组合成具体的工作站类型：

```
workstation_type_config（工作站类型，如 coding / sre）
   └── workstation_agent_engine（这个类型用哪些引擎）
        └── workstation_agent_skill（这个引擎配哪些技能）
             + workstation_skill_assignment（技能分配）
```

也就是说：**type 决定形态、engine 决定引擎、skill 决定能力**，三者组合成一个完整的工作站规格。这棵树的每一层都可以独立增删，不会牵一发动全身。

---

## 工作站本体：一个 K8s Pod，不是一个 docker 容器

三层镜像解决的是"怎么组装环境"，工作站本体解决的是"环境在哪里跑"。

先纠正一个常见的误解：WinMatrix 的工作站**不是本地 docker 容器**。`BaseWorkstation` 的注释开宗明义：

```typescript
// business/domain/workstation/runtime/BaseWorkstation.ts（第 8 行）
// 明示非本地 docker，所有工作站走远程 sandbox-api（K8s Pod）
```

每个工作站对应一个 K8s Pod，通过远程的 sandbox-api 管理。这从数据模型上也能看出来——`coding_workstations` 表里全是 K8s 概念：

```prisma
// prisma/schema.prisma（第 212-252 行）
model coding_workstations {
  id                String   @id @default(uuid())
  container_name    String   @unique @map("container_name")
  agent_id          String
  owner_kind        String   // employee | project
  image             String
  status            String   // created|running|scaled_down|stopped|error
  backend           String   @default("local")  // 实际走远程 sandbox-api
  type              String   @default("coding")
  llm_binding_fingerprint String?  // Pod 创建期的 LLM binding 指纹（无密钥）
  serviceName       String?  @map("service_name")
  servicePort       Int?     @map("service_port")
  // ...
}
```

几个值得注意的字段：

- **`llm_binding_fingerprint`（无密钥）**。Pod 创建时要把 LLM 的 API 密钥传进去，但数据库里不能存明文密钥。这里存的是一个 fingerprint（可以理解成密钥的哈希/指纹），用于校验"Pod 拿到的密钥对不对"，但不暴露密钥本身。
- **`serviceName` / `servicePort`**。工作站作为一个 K8s Service 暴露出来，其他组件通过 service name + port 访问。这是标准的 K8s 服务发现。
- **`status` 五态**：created → running → scaled_down/stopped → error。scaled_down 是个很重要的中间态——工作站不干活时可以缩容（省 Pod 资源），但配置还在，需要时再拉起来。

### Runner：容器内命令的统一抽象

工作站环境有了，代码在里面怎么跑？答案是一个叫 `WorkstationRunner` 的抽象，它实现了 `IWorkspaceCommandRunner` 接口，把"在容器里执行命令"这件事统一成几个原语：

```typescript
// business-tools/workstation/runner/WorkstationRunner.ts（第 34-54 行）
export class WorkstationRunner implements IWorkspaceCommandRunner {
  readonly kind = 'workstation' as const;
  async runRead(absoluteFilePath, options, ctx): Promise<string> {
    return this.exec(this.buildReadCommand(absoluteFilePath, options), undefined, ctx);
  }
  async runWrite(absoluteFilePath, content, options, ctx): Promise<void> {
    await this.exec(this.buildWriteCommand(absoluteFilePath, content, options), undefined, ctx);
  }
  async runExec(command, options, ctx): Promise<string> {
    return this.exec(command, options, ctx);
  }
}
```

三个原语：**read（读文件）、write（写文件）、exec（执行命令）**。所有的工具（WorkspaceReadTool、WriteTool、GlobTool、ExecTool）都基于这三个原语构建。

为什么强调这个抽象？因为**本地的工具实现和工作站的工具实现，共用同一套接口**。本地开发时，Runner 可以指向本地文件系统；生产环境，Runner 指向远程 K8s Pod。上层工具代码完全一样，底层切换实现即可。这是经典的依赖倒置——用接口隔离了"本地"和"远程"的差异。

### 双路径：host 和 container 的路径转换

工作站还有一个容易踩坑的地方：**宿主机路径和容器内路径不是一回事**。

Agent 可能用宿主机路径（比如用户在本地看到的 `/project/src/index.ts`），但工作站里执行命令要用容器内路径（比如 `/workspace/src/index.ts`）。WinMatrix 用一个 pathRegistry 做**双向转换**：

```
宿主机路径 /project/...  ──→  容器内路径 /workspace/...  （执行命令时）
容器内路径 /workspace/... ──→  宿主机路径 /project/...   （返回结果时）
```

workspace discovery 同时维护这两条路径，确保 Agent 看到的是它熟悉的宿主机视图，而容器里执行的是正确的容器路径。**这种"对调用方隐藏底层差异"的设计，是分布式系统避免认知负担的关键。**

---

## 个人分身工作站：独立的模型

最后说一个特殊形态。除了项目级和员工级的工作站，还有**个人分身工作站**（`UserPersonalWorkstation`，schema 第 3624 行）。它和 `coding_workstations` 是两个独立模型，因为生命周期策略不同：

- 项目/员工工作站是"按需创建、用完停"，受项目或员工资源配额管理。
- 个人分身工作站是"跟着员工走"，有自己的可用性策略（`availabilityPolicy`）：`adaptive`（自适应，闲时缩容）、`always_on`（常驻）、`on_demand`（按需拉起），还有 `podGeneration` 控制换代。

**为什么独立模型？** 因为把两种生命周期的东西塞进一张表，必然出现一堆 `if personal then ... else ...` 的分支，代码很快就会腐化。宁可多一张表，把差异显式化。

---

## 业界对比：CRD vs 三层镜像

回到开头那个对比。K8s Agent Sandbox 用 CRD 的思路，是把"我要一个 sandbox"声明化、标准化，让平台 Operator 来管。它的好处是**标准化、社区生态、跨工具复用**；代价是**抽象层级固定**，你想精细控制镜像组合方式就得绕 Operator。

WinMatrix 的三层镜像走的是另一条路：**不依赖 Operator 抽象，自己用数据库表管镜像组合**。好处是**组合方式完全自定义**（component 可以随时加脚本，engine 可以独立升级），坏处是**这套组合逻辑得自己实现**（没有社区 Operator 帮你）。

这是个典型的"标准 vs 灵活"权衡。对我们来说，因为要支持 4 种引擎、多种运行时、还要和内部的 LLM binding、回调机制深度耦合，标准 CRD 的抽象层级反而是负担。但对一个只想要"快速起个隔离环境跑 LLM"的团队，CRD 显然更省事。**没有银弹，只有适合当前阶段的权衡。**

---

## 给后来者的总结

1. **引擎和运行时是两个正交维度，必须拆开管理**。base（运行时）+ engine（引擎）+ component（装什么）三层正交解耦，让"换引擎"不动运行时、"加组件"不动引擎。
2. **base 镜像以 digest 为真源，tag 只是人读的**。要追溯"到底跑的是哪个镜像"，永远看 digest。
3. **engine 按唯一约束独立管理，不绑 type**。一个 engine 镜像可以复用到多个工作站类型，避免组合爆炸。
4. **component 存安装脚本而不是打镜像**。组件级变更不值得重打镜像；用 `compatibleEngines` 声明兼容性防误装。
5. **工作站是 K8s Pod，不是本地 docker**。LLM 密钥用 fingerprint 传递不落库，serviceName/servicePort 走标准服务发现。
6. **Runner 用接口隔离本地和远程**。read/write/exec 三个原语 + IWorkspaceCommandRunner 接口，上层工具无感切换。
7. **宿主机路径和容器路径要双向转换**。对 Agent 暴露宿主机视图，底层转换成容器路径执行。
8. **不同生命周期的实体用不同模型**。个人分身工作站独立成表，别在一个模型里堆 if 分支。
9. **标准（CRD）和灵活（自建组合）是权衡**。深度耦合、组合需求多时自建；只起隔离环境时标准更快。

把 LLM 关进 Pod，看起来是个部署问题，本质是个**组合管理**问题。三层镜像把"引擎"和"运行时"这两条变化频率完全不同的轴拆开，让系统的演进从"牵一发动全身"变成"各改各的"。这不性感，但每一台跑在生产上的编码工作站，都靠这套不起眼的分层活下来。

---

> **下一篇**：[《一次编码任务的幂等与回调：迟到的 attempt 不会覆盖新状态》](./12-coding-task-idempotency.md)——环境搭好了，但一次编码任务执行到一半进程崩了怎么办？重试时怎么保证不覆盖更新的结果？回调怎么防伪造？这是下一篇的主题。
