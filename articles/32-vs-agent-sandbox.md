# 编码工作站 vs K8s Agent Sandbox：自建沙箱 vs 社区标准

> 这是《WinMatrix 开发经验文集》第 32 篇，"行业对比与方法论"系列第二篇。让 AI 写代码，就得给它一个能跑代码的环境——这个环境怎么隔离、怎么构建、怎么管理，是个实打实的工程问题。这篇聊我们自建的编码工作站，和业界用 K8s 做 Agent 沙箱的通用思路，各自意味着什么。

AI 写代码这件事，难点不在"生成代码"，而在"让生成的代码能跑起来验证"。一个能写代码的 AI，需要一个真实的环境：能装依赖、能跑测试、能执行命令、能读改文件。这个环境就是"沙箱"。

但"沙箱"两个字背后，藏着一大堆工程问题：

- **隔离**：AI 跑的代码可能出错、可能删库、可能发起恶意网络请求。怎么保证它不污染宿主？
- **构建**：每个任务的环境可能不一样（不同语言、不同依赖、不同工具链）。怎么高效构建？
- **生命周期**：任务是短时的，但环境启动有成本。环境什么时候建、什么时候销毁？
- **管理**：成百上千个并发任务，每个一个环境，怎么调度、怎么回收？

这些问题，业界有两种典型思路。一种是**用社区标准的 K8s 沙箱方案**（比如以自定义资源 + 容器运行时隔离为核心的通用 Agent sandbox 模式），把 AI 任务当成 K8s 上的 ephemeral workload 来管。另一种是 WinMatrix 选的——**自建一套以"三层镜像 + 远程 sandbox-api"为核心的编码工作站体系**。

这篇就把两种思路摆开，讲清楚各自的取舍。

---

## 先把两种思路画清楚

为了便于对比，先勾勒两种思路的典型形态。这里只描述业界已知的、公开的做法方向，不涉及任何具体实现细节。

**社区 K8s Agent Sandbox 思路**：

```
   ┌─────────────────────────────────┐
   │  K8s 集群                        │
   │  ┌───────────────────────────┐  │
   │  │  Sandbox CRD（自定义资源） │  │  ← 声明式描述"我要一个沙箱"
   │  └─────────────┬─────────────┘  │
   │                ↓ 调谐            │
   │  ┌───────────────────────────┐  │
   │  │  单容器 Pod                │  │
   │  │  隔离：Kata / gVisor       │  │  ← 容器运行时级隔离
   │  │  生命周期：ephemeral       │  │  ← 用完即销
   │  └───────────────────────────┘  │
   └─────────────────────────────────┘
```

特征：把"沙箱"抽象成一种 K8s 自定义资源（Custom Resource），用声明式 API 描述"我要一个什么样的沙箱"；K8s 控制器把它调谐成一个 Pod；Pod 用 Kata Containers 或 gVisor 这类强隔离容器运行时；任务完成 Pod 销毁，是 ephemeral workload。这套思路的好处是**和 K8s 生态原生契合**——调度、资源管理、网络策略都能复用 K8s 的能力。

**WinMatrix 自建工作站思路**：

```
   ┌──────────────────────────────────────────┐
   │  WinMatrix 主应用                         │
   │  ┌────────────────────────────────────┐  │
   │  │  coding_workstations（DB 模型）     │  │
   │  │  type=coding / engine=claude_code  │  │
   │  └──────────────┬─────────────────────┘  │
   │                 │ 远程调用                  │
   │  ┌──────────────▼─────────────────────┐  │
   │  │  sandbox-api（K8s Pod 远程服务）    │  │
   │  │  ┌─────────┬─────────┬───────────┐ │  │
   │  │  │base     │engine   │component  │ │  │  ← 三层镜像正交
   │  │  │image    │image    │           │ │  │
   │  │  │(OCI base)│(claude/ │(安装脚本) │ │  │
   │  │  │         │ codex/  │           │ │  │
   │  │  │         │ hermes) │           │ │  │
   │  │  └─────────┴─────────┴───────────┘ │  │
   │  └────────────────────────────────────┘  │
   └──────────────────────────────────────────┘
```

特征：在 DB 里用 `coding_workstations` 表描述每个工作站（type、engine、关联的 agent）；构建用三层镜像模型（base → engine → component），引擎和运行时正交解耦；所有工作站走远程 sandbox-api（K8s Pod），不是本地 docker。

---

## 第一个对比点：隔离的强度和方式

两种思路最显眼的差异，是隔离怎么做。

社区 K8s Agent Sandbox 思路倾向于**容器运行时级强隔离**——用 Kata Containers（基于轻量虚拟机）或 gVisor（用户态内核）这类技术，让每个沙箱在内核层面就和其他沙箱、和宿主隔离。这是"宁可慢一点，也要绝对隔离"的思路，适合"沙箱里跑的代码完全不可信"的场景（比如运行用户提交的任意代码）。

WinMatrix 的隔离思路不太一样。我们的编码工作站主要不是跑"用户提交的任意代码"，而是**给数字员工（AI）提供写代码的工作环境**。这个环境里的代码是 AI 在既定工作流里生成和执行的，可信度比"完全任意的用户代码"高一个层级——它依然要防出错、防误删，但不需要防到"假设这是恶意攻击者构造的 payload"的程度。

所以 WinMatrix 的隔离主要靠**容器隔离 + 远程 Pod**：每个工作站是一个独立的 K8s Pod（`coding_workstations` 表里的 `container_name` 唯一），跑在远程 sandbox-api 上，和主应用进程物理分离。这层隔离保证了：工作站里的代码出错（比如误删文件、装错依赖），不会影响主应用和其他工作站。

```
coding_workstations 的运行形态（schema 第 212-252 行）：
  id / container_name(@unique) / agent_id
  owner_kind = employee | project
  image / status = created|running|scaled_down|stopped|error
  backend = local / type = coding
  llm_binding_fingerprint / serviceName / servicePort   ← K8s Service 暴露
```

注意 `status` 有个 `scaled_down` 状态——这是 WinMatrix 特有的"缩容不销毁"语义（后面讲）。而社区 Sandbox 思路通常是 ephemeral（用完即销），没有这层中间态。

**隔离强度的取舍，本质是信任模型的取舍。** 如果你假设沙箱里跑的是完全不可信的代码，上 Kata/gVisor 强隔离是对的；如果沙箱是给你自己的 AI 用的、代码来自既定工作流，容器隔离 + 远程 Pod 这一档就够了，强隔离反而带来不必要的性能开销（Kata 的启动比普通容器慢一个数量级）。

---

## 第二个对比点：镜像怎么构建

构建是两种思路差异最大的地方，也是 WinMatrix 选择自建的核心原因。

社区 K8s Sandbox 思路里，镜像构建相对简单——通常是一个基础镜像 + 任务时动态注入的代码/依赖。沙箱是 ephemeral 的，每次任务可能都用同一个基础镜像，把任务代码挂进去跑完就扔。构建成本低，但**环境不可定制**——你想让这个沙箱预装一套特殊的工具链（比如某个内部框架的 CLI、某个定制 LSP），就很难。

WinMatrix 的三层镜像模型，解决的就是"环境要深度可定制"的问题：

```
三层镜像（schema 第 2151-2287 行，正交解耦）：

  第 1 层：workstation_runtime_base_image（OCI base）
    repository / tag / digest / platforms / status
    @@unique([repository, tag])
    ↑ 最底层，操作系统的基座，digest 是真源

  第 2 层：workstation_engine_image（引擎层）
    engineType = claude_code | codex | hermes | openclaw
    engineName / engineVersion
    imageRepository / imageTag / imageDigest
    @@unique([engineType, engineName, engineVersion])
    ↑ 编码引擎，独立管理，不绑 type

  第 3 层：workstation_component（组件层）
    installScript（全文）+ checksum
    compatibleEngines[]   ← 控兼容性
    ↑ 按需安装的组件，声明它兼容哪些引擎
```

三层是**正交解耦**的：base 层管操作系统，engine 层管编码引擎（claude_code / codex / hermes / openclaw 四种），component 层管具体组件（某个语言运行时、某个工具、某个内部 CLI）。换 base 不用动 engine，换 engine 不用动 component，component 还能声明它兼容哪些 engine（`compatibleEngines[]`）。

这种正交解耦带来的好处是**组合性**：

```
type → engine → skill 配置树：
  type_config（工作站类型）
    └── agent_engine（选哪个引擎）
          └── agent_skill + workstation_skill_assignment（装哪些组件/技能）
```

一个"研发编码工作站"可以用 claude_code 引擎 + 一套研发工具组件；一个"SRE 运维工作站"可以用 hermes 引擎 + 一套运维工具组件。引擎和组件自由组合，不互相绑架。

**这是社区 Sandbox 思路很难做到的。** 社区思路假设沙箱是同质的（每个沙箱都长一样），而 WinMatrix 假设沙箱是异质的（不同角色、不同任务需要不同环境）。后者的构建复杂度高得多，但换来的是"给每个数字员工量身定做工作环境"的能力。

### 代价：构建矩阵爆炸

正交解耦的代价是构建矩阵爆炸。我们有：

- base 镜像：若干（不同 OS / 不同平台）
- engine 镜像：4 种（claude_code / codex / hermes / openclaw）
- workstation 镜像：3 种（coding / sre / openclaw）
- 组件：若干

Makefile 里光是镜像相关的 target 就有十几个（`build-push-coding-workstation` / `build-push-sre-workstation` / `build-engine-claude-code` / `build-engine-codex` / `build-engine-hermes` / `build-engine-openclaw` / `build-all-engines`……）。这是一套自建的构建矩阵，维护成本不低。

**这是"自建 vs 社区标准"最直观的取舍**：社区标准帮你省掉了构建矩阵的维护成本，但代价是你只能用它预设的、同质的沙箱形态；自建让你能做异质、可组合的深度定制，但构建矩阵得自己背。

---

## 第三个对比点：生命周期管理

两种思路对沙箱生命周期的假设也不一样。

社区 K8s Sandbox 思路倾向于 ephemeral——任务来了建 Pod，任务完了销毁 Pod。这种假设适合"短时、无状态"的任务。好处是资源利用率高（不用的沙箱不占资源），坏处是冷启动成本高（每次任务都要重新拉镜像、装依赖、初始化）。

WinMatrix 的生命周期更复杂，因为编码任务往往不是"短时无状态"的。一个编码任务可能要跑很久（几十分钟到几小时），中途要反复读改文件、跑测试、看结果。如果每次都冷启动，成本受不了。所以 `coding_workstations.status` 里有 `scaled_down` 这个中间态：

```
status 状态机：
  created → running → scaled_down → running → ... → stopped/error
                 ↑                    ↑
                 任务来了              下个任务来了
                 拉起                  重新拉起（快，不用重建）
```

`scaled_down` 是"缩容但保留"——Pod 可能被缩到最小资源占用，但环境（已装的依赖、已 clone 的代码、已初始化的会话）不销毁。下个任务来了，从 `scaled_down` 拉回 `running` 比从 `created` 冷启动快得多。

这层"缩容不销毁"的语义，是 WinMatrix 为编码场景专门做的——它假设工作站的启动成本高（镜像大、依赖多、会话要恢复），值得用资源换启动速度。社区 ephemeral 思路没有这层，因为它的假设是"每个任务都从头来，启动成本可控"。

### 任务幂等与回调安全

编码任务的生命周期管理里，还有一组社区思路通常不太关注的问题——任务的幂等和回调安全。WinMatrix 的 CodingTask 模型（schema 第 310-386 行）为这做了专门设计：

```
CodingTask 关键字段：
  workstationId / triggerTool = coding_task | sre_skill_task | execute_task
  idempotencyKey              ← 幂等键
  attemptNo                   ← 防迟到旧 attempt 覆盖新状态
  callbackTokenHash           ← 回调鉴权
  taskFingerprint             ← 任务指纹
  claudeSessionId             ← 会话复用
  partial unique index:
    WHERE claude_session_id IS NOT NULL AND status='completed'
    ← 复用已完成任务的 session
```

这些字段解决的是分布式环境下的各种边界情况：

- **同一个任务被重复触发**怎么办？`(conversation_id, trigger_tool, idempotency_key)` 在 running 态去重。
- **迟到的旧 attempt 回来了，会不会覆盖新状态**？`attemptNo` 防覆盖。
- **回调怎么确认是合法的**？`callbackTokenHash` 鉴权。
- **任务完成了，下次相似任务能不能复用 session**？partial unique index 让已完成任务的 session 可被复用。

这些问题在"长生命周期编码任务 + 分布式环境"里是刚需。社区 ephemeral 思路因为假设任务短时无状态，通常不做这么重的幂等和会话复用——但那是因为它不做编码场景。

---

## 第四个对比点：和业务的耦合度

最后一个对比点更软性，但很关键——沙箱和业务的耦合度。

社区 K8s Agent Sandbox 思路追求的是**通用**——它不想知道你的业务是什么，它只提供"一个隔离的、可执行的环境"，业务逻辑你自己往里塞。这种通用性让它能服务各种场景（不只编码，还能跑数据分析、跑测试、跑批处理），但也意味着它不理解你的业务——你的"数字员工"概念、"技能"概念、"项目"概念，对它来说都是黑盒。

WinMatrix 的工作站是**深度业务耦合**的。`coding_workstations` 表里直接带着 `agent_id`（关联到数字员工）、`owner_kind = employee | project`（属于员工还是项目）、`type = coding`（工作站类型）。工作站的创建、调度、资源分配，都和数字员工、技能、项目这些业务概念绑死。

```
coding_workstations 的业务关联字段：
  agent_id          ← 哪个数字员工的
  owner_kind        ← 属于员工（个人分身）还是项目
  project_id?       ← 属于哪个项目
  type = coding     ← 工作站类型
  llm_binding_fingerprint ← Pod 创建期的 LLM 绑定（无密钥）
```

这种深度耦合的好处是**业务语义自然渗透到资源管理层**——"这个工作站的 LLM 用量算哪个员工的""这个工作站缩容了要不要通知对应的数字员工""个人分身的工作站和项目工作站的资源怎么隔离"，这些问题在数据模型层面就有答案。坏处是**这套工作站体系很难独立拿出来给别的项目用**——它和 WinMatrix 的数字员工、项目、技能体系长在一起。

---

## 各自适合什么场景

总结一下两种思路的适用场景。

**社区 K8s Agent Sandbox 思路适合**：

- 需要运行不可信代码、需要强隔离的场景（比如在线代码执行平台、用户提交代码评测）。
- 沙箱同质、任务短时无状态的场景（每个沙箱长得一样，用完即销）。
- 想复用 K8s 生态能力、不想自己造沙箱轮子的团队。
- 业务逻辑和沙箱解耦、沙箱作为通用底座的场景。
- 团队有 K8s 运维能力，能驾驭 CRD + 控制器的开发模式。

**WinMatrix 自建工作站思路适合**：

- 沙箱是给可信的 AI（自己的数字员工）用的，信任模型不需要到"防恶意攻击"级别。
- 沙箱需要深度可定制（不同引擎、不同工具链、不同组件组合）。
- 任务长时、有状态、需要会话复用和缩容保活。
- 沙箱要和业务概念（员工、技能、项目）深度耦合，业务语义要渗透到资源管理。
- 团队愿意为这套构建矩阵和维护体系持续投入。

**判断标准**：如果你的沙箱是"给陌生用户跑任意代码"，用社区强隔离方案；如果你的沙箱是"给自己的 AI 提供量身定做的工作环境"，且环境需要高度可组合、可复用、和业务深度绑定，自建更合适。前者是平台思维（提供通用底座），后者是产品思维（为特定场景深度优化）。

---

## 自建的代价与诚实交代

WinMatrix 选自建，不是因为它"更好"，而是因为我们的场景（给数字员工做编码环境）需要深度定制，社区通用方案满足不了。但这条路代价不小，诚实地说：

1. **构建矩阵的维护负担**。三层镜像 × 多引擎 × 多组件 × 多平台，组合起来构建任务不少。Makefile 44 个 target 里一大半是镜像相关的。这是持续的工程投入。
2. **K8s 运维负担**。远程 sandbox-api 这套，等于在 K8s 上又自建了一层调度逻辑（coding_workstations 表 + WorkstationRunner + 远程调用）。它不和 K8s 原生的 Deployment/Job 复用调度能力，得自己管。
3. **隔离强度的取舍风险**。我们没上 Kata/gVisor，隔离靠容器 + 远程 Pod。如果未来场景变成"要跑完全不可信的代码"，这套隔离不够，得重构。
4. **迁移成本**。这套工作站体系和 WinMatrix 的业务深度耦合，如果将来想换底座（比如改用社区 Sandbox 方案），迁移代价极大。

这些代价我们认了，因为场景值得。但如果你的场景不是"给 AI 做深度定制的编码环境"，这些代价可能不值得付——社区方案能帮你省掉一大半工作量。

---

## 给后来者的几条总结

1. **沙箱不是越强隔离越好，要看信任模型**。完全不可信代码上 Kata/gVisor；可信 AI 的编码环境，容器 + 远程 Pod 这档够用。过度隔离有性能代价。
2. **社区标准省工作量，自建换定制力**。K8s Agent Sandbox 思路帮你省构建矩阵和调度逻辑，但只能做同质沙箱；自建能做异质、可组合，但构建矩阵得自己背。
3. **三层镜像正交解耦是"可组合环境"的关键**。base / engine / component 三层独立演进，component 声明 compatibleEngines 控兼容性。组合性 > 单体定制。
4. **生命周期要匹配任务特征**。短时无状态用 ephemeral；长时有状态（编码任务）需要 scaled_down 中间态保活、会话复用。冷启动成本高的场景值得用资源换启动速度。
5. **幂等和回调安全是长生命周期任务的刚需**。idempotencyKey / attemptNo / callbackTokenHash / partial unique index，这些在分布式 + 长任务的组合下不可省。
6. **业务耦合是双刃剑**。深度耦合让业务语义渗透资源管理，但也让这套体系难以独立复用。想清楚你的沙箱是"通用底座"还是"特定场景的定制件"。
7. **自建前先评估维护能力**。构建矩阵 + K8s 调度逻辑 + 业务耦合，三者都是持续的工程投入。团队没有长期投入的准备，别自建。
8. **不存在"正确答案"，只有"匹配场景的答案"**。社区标准和自建都有成功的案例，也都有翻车的案例。关键是想清楚自己的信任模型、定制需求、维护能力。

沙箱是 AI 编码场景的"底盘"。底盘选错了，上层 AI 能力再强也跑不稳。但底盘也不是越豪华越好——够用、可维护、匹配场景，就是好底盘。

---

> **下一篇**：[《渐进式决策 vs 纯 LLM 路由：为什么我们不学 AutoGPT》](./33-vs-llm-routing.md)——沙箱讲完了，接着聊决策路由的两种哲学：让 LLM 全权决定，还是用确定性规则优先？
