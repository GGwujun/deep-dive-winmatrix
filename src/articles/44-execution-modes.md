# 五种执行模式：interactive / coordinator / react / skill / workstation 怎么选

> 这是《WinMatrix 开发经验文集》第 44 篇。前面几篇讲的都是"消息进来怎么路由、怎么分诊"。这一篇往下走一层：路由完了，决定让某个员工干活了，**这活怎么干**。WinMatrix 有五种执行模式，不是花哨，是因为"不同任务需要不同的执行语义"。

先说一个容易混淆的事。

很多人第一次看到"执行模式"会以为是"LLM 调用模式"——比如流式 vs 非流式。不是。**执行模式是"这个任务怎么编排、跑几轮、什么时候停"的语义**。它比单次 LLM 调用高一整层。

举几个例子你就明白了：

- 用户问"你好"——一次 LLM 调用就完事，不用编排。
- 用户说"帮我做技术方案"——要分多步（读 PRD → 调研 → 写方案 → 自审），每步可能换不同员工，需要编排。
- 用户说"跑一下这个编码任务"——要开一个 K8s 沙箱，让 Claude Code 在里面干活，等它出结果。
- 用户说"按这个技能的流程走"——直接跳到一个预定义的技能执行。

这四种诉求，执行语义完全不同。用一种模式硬扛，要么过度编排（闲聊也走多步），要么编排不足（复杂任务一步完事）。WinMatrix 的五种模式就是针对这类差异设计的。

---

## 五种模式速览

先把五种模式摆出来：

| 模式 | 目录位置 | 一句话语义 | 典型场景 |
|------|---------|-----------|---------|
| `interactive` | modes/interactive/ | 多员工轮流协作，有议程 | 会议、评审、讨论 |
| `coordinator` | modes/cdw/ | 多步编排，有规划-执行-收尾 | 复杂项目任务 |
| `react` | modes/react/ | 单 Agent 的思考-行动循环 | 单员工推理 |
| `skill` | （非独立目录） | 直接执行某个技能 | 明确的技能触发 |
| `workstation` | （远程 sandbox） | 远程沙箱里跑编码引擎 | 编码/SRE 任务 |

先澄清一个容易混的点：**这里的"执行模式"和工具循环里的 `decisionMode` 是两个维度**。工具循环里的 `decisionMode`（如 `chat_only` / `skill`）影响的是单次 Turn 内的迭代次数上限；这里说的五种模式影响的是**整个任务的编排结构**。别混。

我们一个个看。代码都在 `agents/core/agent/modes/` 下（`cdw` / `interactive` / `react` 三个子目录，skill 和 workstation 不在 modes 目录里，它们是特殊路径）。

---

## react：单 Agent 的思考-行动循环

最基础的模式是 react。它就是经典的 ReAct（Reason + Act）循环——一个 Agent 反复"思考 → 调工具 → 看结果 → 再思考"，直到完成任务。

```
react 模式（单 Agent 循环）：

  开始
   │
   ▼
  Think（LLM 推理：下一步该干什么）
   │
   ▼
  Act（调工具）
   │
   ▼
  Observe（看工具结果）
   │
   ├─ 任务完成？ ──是──► 输出
   │
   ▼
  回到 Think
```

目录结构能看出它的关注点：

```
modes/react/
  ├── ReactRuntime.ts          # 运行时
  ├── ReactBriefBuilder.ts     # 构建任务简报
  ├── ReactFinalComposer.ts    # 最终输出合成
  ├── ReactLoopState.ts        # 循环状态
  ├── ReactStepHistorySummary.ts  # 步骤历史摘要
  ├── ReactThinkDecider.ts     # 是否需要 think
  └── ReactThinkOutputNormalizer.ts  # think 输出归一化
```

react 的核心特征：**单 Agent、自循环、迭代到完成**。它不需要外部编排——Agent 自己决定下一步干什么，循环到任务结束。终止条件要么是 Agent 自己说"完成了"，要么撞上工具循环的三道闸门（迭代次数 / LLM 硬上限 / 时间预算，见 [第 1 篇](./01-turn-engine.md)）。

几个值得注意的设计：

**`ReactThinkDecider`**——不是每轮都要 think。有些轮次可以直接 act（比如工具结果很明确，下一步显而易见）。这能省 LLM 调用。**"什么时候该想、什么时候直接做"本身是个决策**，单独抽象出来。

**`ReactStepHistorySummary`**——循环到很多轮时，历史会变长。这个组件把历史步骤压缩成摘要，避免 context 无限膨胀。这是 ReAct 在长任务下的必备能力。

**`ReactFinalComposer`**——循环结束后，把多步的中间结果合成一份最终输出。不是简单拼接，而是重新组织成用户能看懂的答复。

react 模式适合"一个员工能独立完成"的任务——用户问个问题、查个资料、改段代码。一旦需要多员工配合，就该上 interactive 或 coordinator 了。

---

## interactive：多员工轮流协作

interactive 是"会议"模式。多个数字员工在一个会话里轮流发言，按议程推进。

```
interactive 模式（多员工轮流）：

  议程（Agenda）
   │
   ▼
  选参与者（Participant Selection）
   │
   ▼
  ┌─── 轮次循环 ──────────────────────┐
  │                                    │
  │  当前发言者 Act（发言/调工具）     │
  │      │                             │
  │      ▼                             │
  │  下一个发言者是谁？（路由）        │
  │      │                             │
  │      └─ 议程推进 ── 完成？ ──► 收尾 │
  │                                    │
  └────────────────────────────────────┘
```

从目录能看出它的复杂度：

```
modes/interactive/
  ├── InteractiveRoleRuntime.ts         # 运行时
  ├── interactiveAgendaSeed.ts          # 议程初始化
  ├── interactiveParticipantSelection.ts # 参与者选择
  ├── interactiveEnvironmentRoutingService.ts # 路由
  ├── interactiveRoleActExecutor.ts     # 单角色发言执行
  ├── interactiveDecisionPlanNormalization.ts # 计划归一化
  ├── interactiveEnvironmentSnapshotService.ts # 快照
  └── agentStateSnapshotBridge.ts       # 状态桥接
```

interactive 的核心是**议程 + 参与者 + 轮转**：

**议程（Agenda）**。会议要有主题和流程，不能漫无边际地聊。`interactiveAgendaSeed` 负责初始化议程——这次会议讨论什么、分几个环节。

**参与者选择（Participant Selection）**。不是所有员工都要参加每次会议。根据议程选择合适的参与者——讨论代码就拉阿码和小质，讨论运维就拉大维。

**轮转（Routing）**。谁先说、谁接话、什么时候换人——这是 interactive 最精细的部分。`interactiveEnvironmentRoutingService` 决定"当前发言者说完后，下一个该谁"。不是简单轮询，而是基于议程进度和发言内容判断。

**`interactiveRoleActExecutor`**。单个角色发言的执行——它要拿到前面的发言上下文，生成自己的发言，可能调工具（比如查资料），然后把发言发出去。

interactive 适合"需要多角色讨论才能推进"的场景——评审、方案讨论、早会。它的状态机比 react 复杂得多，因为有"多角色协调"这层。

---

## coordinator：多步编排的"项目经理"

coordinator 是五种模式里最重的。它像一个项目经理——接到任务后先**规划**（拆成几步、每步谁干），再**执行**（按计划派活），最后**收尾**（汇总结果、判断是否完成）。

```
coordinator 模式（规划 → 执行 → 收尾）：

  用户需求
   │
   ▼
  Planning（规划阶段）
   │  ConversationDeltaPlanner 把需求拆成多步计划
   │  每步指派角色和技能
   ▼
  ┌─── 执行循环 ──────────────────────────────┐
  │                                             │
  │  Dispatch（派活）                           │
  │      │                                      │
  │      ▼                                      │
  │  Subagent 执行（单个员工干一步）            │
  │      │                                      │
  │      ▼                                      │
  │  Progress（进度跟踪）                       │
  │      │                                      │
  │      ├─ 计划完成？                          │
  │      │     └─ 否 → 派下一步                 │
  │      │     └─ 出错？ → Repair（修复计划）   │
  │      │                                      │
  └──────┴──────────────────────────────────────┘
   │
   ▼
  Close（收尾阶段）
   │  汇总结果、判断终态、生成最终输出
   ▼
  完成
```

coordinator 的代码在 `modes/cdw/` 下，目录结构能看出它的体量：

```
modes/cdw/
  ├── planning/        # 规划（ConversationDeltaPlanner、PlanMutator）
  ├── coordinator/     # 协调器（CoordinatorRuntime、检查点）
  ├── dispatch/        # 派活
  ├── subagent/        # 子 Agent 执行
  ├── pipeline/        # 管线
  ├── progress/        # 进度跟踪
  ├── repair/          # 计划修复
  ├── continuation/    # 续接（挂起后恢复）
  ├── close/           # 收尾（CloseStage）
  └── contracts/       # 类型契约
```

coordinator 的几个独有能力：

**规划（Planning）**。`ConversationDeltaPlanner` 把用户需求拆解成多步计划。不是一次定死，而是"增量规划"——根据执行进度可以调整后续计划。

**派活（Dispatch）**。计划的每一步派给具体的 subagent 执行。派活要考虑负载、依赖关系（前一步的产出是后一步的输入）。

**修复（Repair）**。`PlanMutator` 处理"执行出错了怎么办"——某步失败了，是重试、跳过、还是改计划？这是 coordinator 比 interactive 多出来的能力：它能处理执行中的失败和计划调整。

**续接（Continuation）**。coordinator 任务可能跑很久，中间挂起（等用户输入、等异步结果）。`continuation/` 处理挂起后的恢复——这正是 [上一篇 SemanticTriageGate](./43-semantic-triage-gate.md) 里 bound 分类要对接的场景。

**收尾（Close）**。`CloseStage` 是 coordinator 的终点——汇总所有步骤的产出，判断任务是否真正完成，生成最终输出。收尾不是"最后一步执行完就自动结束"，而是有专门的阶段判断"是不是真的做完了、要不要再补一步"。

coordinator 的终止条件最复杂：计划完成、撞上时间预算、用户终止、不可恢复的错误，都可能触发收尾。

---

## skill：直接跳到技能执行

skill 模式是最直接的——路由结果明确指向某个技能（比如"每日早会"），直接跳到那个技能的流程执行，不做规划、不拆步骤。

它没有独立目录，因为它是决策引擎的一个"短路"路径。当 FusionRouter（[第 42 篇](./42-route-rule-fusion-router.md)）命中一条 `mode='skill'` 的 route_rule，或者 SemanticTriageGate 明确识别为某技能触发时，直接进入技能执行。

skill 模式的执行语义：**技能定义里已经规定了流程（第几步干什么、用什么工具），执行器照着走就行**。不需要 coordinator 那种动态规划。

skill 模式的终止条件：技能流程走完，或者某步失败（这时走技能契约的 onMissing 逻辑——block/manual/skip/use_default，见 [第 6 篇](./06-skill-system.md)）。

为什么 skill 和 coordinator 要分开？因为**有些任务的流程是固定的，不需要动态规划**。早会就是固定的几步（回顾昨日 → 汇报今日 → 提出风险），用 coordinator 去规划是杀鸡用牛刀，还可能规划出非标准流程。skill 模式保证固定流程按定义执行，不跑偏。

---

## workstation：远程沙箱执行

workstation 模式最特殊——执行不在本进程，而在一个远程 K8s Pod 里的编码引擎（Claude Code / Codex / Hermes / OpenClaw）。

```
workstation 模式（远程沙箱）：

  WinMatrix 主进程
   │
   ├── 创建/复用 workstation（K8s Pod）
   │
   ├── 把任务发到 sandbox-api
   │      │
   │      ▼
   │   Pod 里的编码引擎干活
   │   （Claude Code / Codex / ...）
   │      │
   │      ├── 读代码、改代码、跑测试
   │      │
   │      ▼
   │   产出结果、artifact
   │
   ├── 回调（callbackTokenHash 鉴权）
   │
   ▼
  收结果、落库、通知用户
```

workstation 模式的关键特征：

**执行环境隔离**。所有工作站走远程 sandbox-api（K8s Pod），不是本地 docker。这保证安全性——编码引擎在沙箱里执行任意命令，不影响主进程。

**异步回调**。任务发出去后，主进程不阻塞等，而是通过回调拿结果。回调用 `callbackTokenHash` 鉴权（[第 12 篇](./12-coding-task-idempotency.md) 讲过）。这意味着 workstation 模式的"终止"不是主进程决定的，而是沙箱完成后回调通知。

**三层镜像**。workstation 的环境由 `base_image → engine_image → component` 三层镜像组成（[第 11 篇](./11-coding-workstation-three-layer.md)）。不同任务用不同引擎组合。

workstation 模式的终止条件：沙箱回调完成、超时（`system-coding-task-timeout-sweep` 定时扫描）、用户手动停止。

---

## decisionMode 影响工具循环上限

讲完五种模式，补一个容易混的点。除了"模式"，决策结果还会带一个 `decisionMode`，它影响的是**单次 Turn 内工具循环的迭代上限**：

```typescript
// agents/core/ai-execution/tool-loop/StreamingToolExecutor.ts（第 678-694 行）
export function resolveEffectiveMaxIterations(params: {
  maxIterations: number;
  skillMaxIterations?: number;
  decisionMode?: StreamingToolExecutorConfig['decisionMode'];
}): number {
  if (params.decisionMode) {
    switch (params.decisionMode) {
      case 'chat_only':
        return Math.min(params.maxIterations, 3);   // 闲聊最多 3 轮
      case 'skill':
        return Math.min(params.maxIterations, params.skillMaxIterations ?? params.maxIterations);
      default:
        break;
    }
  }
  return params.maxIterations;
}
```

`chat_only` 把迭代上限压到 3——闲聊不需要多轮工具调用，3 轮足够。这是工具循环层的优化，和五种"执行模式"是正交的两个维度：

```
执行模式（interactive/coordinator/react/skill/workstation）
   决定"任务怎么编排"
       ×
decisionMode（chat_only/skill/...）
   决定"单次 Turn 内工具循环跑几轮"
```

别混。一个 coordinator 模式的任务，其中某一步可能是 chat_only（只是问个问题）；一个 react 模式的任务，整体 decisionMode 可能是默认的（不限死轮数）。

---

## 怎么选模式

实际系统里，模式不是用户选的，是**决策引擎根据任务特征决定的**。大致的决策逻辑：

```
任务进来
   │
   ├─ 是明确技能触发？ ──是──► skill 模式
   │
   ├─ 是编码/SRE 类任务（要在沙箱执行）？ ──是──► workstation 模式
   │
   ├─ 需要多角色讨论？ ──是──► interactive 模式
   │
   ├─ 是复杂多步任务（需要规划）？ ──是──► coordinator 模式
   │
   └─ 单员工能独立完成？ ──是──► react 模式
```

这个选择不是硬编码的 if-else，而是融合在决策引擎的规划阶段（DecisionPlanner）。决策引擎根据任务复杂度、参与者、是否需要沙箱等因素，输出一个带 `executionMode` 的计划。

**关键洞察：模式不是花哨，是"不同任务用不同执行语义"的工程化**。用 coordinator 去跑闲聊是浪费（规划开销），用 react 去跑复杂项目任务是不足（没有多步规划）。匹配任务特征的模式，才能在成本、正确性、体验之间找平衡。

---

## 给后来者的总结

1. **执行模式是"任务怎么编排"的语义，不是 LLM 调用模式**。它比单次 Turn 高一整层。和工具循环的 decisionMode（chat_only/skill）是正交维度，别混。

2. **react 是单 Agent 自循环**。Think-Act-Observe 到完成。有 ThinkDecider（不是每轮都 think）、StepHistorySummary（长任务压缩历史）、FinalComposer（合成输出）。适合单员工独立完成的任务。

3. **interactive 是多员工会议模式**。议程 + 参与者选择 + 轮转路由。适合评审、讨论等需要多角色协作的场景。轮转不是简单轮询，基于议程和内容判断。

4. **coordinator 是项目经理模式**。规划 → 派活 → 进度跟踪 → 修复 → 收尾。最重，但有动态规划、失败修复、挂起续接、专门收尾等独有能力。适合复杂多步项目任务。

5. **skill 是固定流程的直接执行**。不规划，照技能定义走。保证标准流程不跑偏。终止靠技能契约的 onMissing 逻辑。

6. **workstation 是远程沙箱执行**。K8s Pod 里跑编码引擎，异步回调拿结果。环境隔离 + 三层镜像 + callbackToken 鉴权。终止靠回调或超时扫描。

7. **coordinator 的 CloseStage 是专门的收尾阶段**。不是"最后一步完就结束"，而是判断"是不是真的做完了、要不要补一步"。复杂任务的终止判断本身是个决策。

8. **chat_only 模式把工具循环压到 3 轮**。闲聊不需要多轮工具调用。这是 Turn 内的优化，和五种执行模式正交。

9. **模式由决策引擎根据任务特征决定，不是用户选的**。任务复杂度、参与者、是否需沙箱，共同决定模式。匹配任务特征的模式才能平衡成本和正确性。

五种模式背后是同一个工程思想：**不要用一种编排方式硬扛所有任务**。闲聊用重的多步编排是浪费，复杂项目任务用轻的单步执行是不足。把执行语义分层，让每种任务找到合适的模式，系统才能在规模化的同时保持清晰。

---

> **下一篇**：[《PromptOverride：同一个角色，不同项目/分身怎么定制提示词》](./45-prompt-override.md)——讲完了执行模式，下一篇回到"人格"主题。同一个阿码，在不同项目里怎么有不同的叮嘱？同一个角色的不同分身怎么差异化？PromptOverrideService + agent_prompt_template 的组合是怎么解决这个问题的。
