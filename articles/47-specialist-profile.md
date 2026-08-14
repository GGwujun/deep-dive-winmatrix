# 数字员工的"专长画像"：SpecialistProfile 怎么驱动能力绑定

> 这是《WinMatrix 开发经验文集》第 47 篇。前面几篇讲了数字员工的"人格"（agent_config 驱动的身份/原则/提示词）和"执行模式"（五种模式怎么选）。这一篇讲一个更底层的问题：一个数字员工"会什么"，到底由什么决定？为什么同样是技术岗，阿码能写代码而大维（SRE）不能？这背后不是 prompt 写得好不好，而是一张叫 SpecialistProfile 的"专长画像"。讲这张画像怎么来、怎么驱动能力绑定、又为什么必须独立于人格单独建模。

先抛一个看似简单的问题：WinMatrix 里有八个角色——大福（项目总指挥）、阿宁（事项管理）、小品（产品）、阿码（技术）、小质（质量）、大维（SRE）、Architector（架构师）、还有运行时动态生成的外联 Agent。**凭什么阿码能调工作站写代码、跑代码评审技能，而小品就不能？**

新手的第一反应是：因为阿码的 prompt 里写了"你是技术管理者"。但这解释不通——prompt 只是话术，LLM 真要去调一个它"不该有"的工具，prompt 拦不住。真正的拦截发生在工程层：**一个角色的能力边界，是被一张"专长画像"显式声明的，而不是被 prompt 暗示的。** 这张画像就是 SpecialistProfile。

这一篇我们就挖这张画像：它从哪来，怎么和技能/工具/工作站绑定，又为什么必须和"人格"分开建。

---

## 先看一个角色是怎么被装配出来的

要看懂画像，得先看一个角色实例从配置到运行的装配链。这条链的起点在启动注册，终点是会话级独立实例。

七大业务角色在启动时被注册进 RoleRegistry（`startup/agents.ts:126-133`）：

```ts
roleRegistry.registerFactory('orchestrator', () => new OrchestratorRole());
roleRegistry.registerFactory('process_manager', () => new ProcessManagerRole());
roleRegistry.registerFactory('product_design_manager', () => new ProductDesignRole());
roleRegistry.registerFactory('tech_manager', () => new TechManagerRole());
roleRegistry.registerFactory('sre_manager', () => new SreManagerRole());
roleRegistry.registerFactory('quality_manager', () => new QualityManagerRole());
roleRegistry.registerFactory('architect', () => new ArchitectRole());
```

每个 Role 实现类的构造器里，从 `agent_config` 表加载自己的身份四件套——`name/profile/goal/constraints`（`agents/domain-harness/roles/TechManagerRole.ts:35-49`）：

```ts
constructor() {
  super({ reactMode: RoleReactMode.REACT, watch: [] });
  const agentConfig = this.loadConfigFromManager('tech_manager');
  this.name = 'tech_manager';
  this.profile = agentConfig?.role || '研发技术管理者';
  this.goal = agentConfig?.description || '制定工程化标准，守护代码质量...';
  this.constraints = agentConfig?.focus || '简单优先、成熟稳定...';
  this.nickname = agentConfig?.nickname || '阿码';
  this.onInitialized();
}
```

注意这里加载的是**身份**——你是谁、你的目标、你的约束。这是"人格"层。但身份里并没有写"你会写代码"。那么"会写代码"这个能力，是从哪来的？

答案在装配链的更深一层：每个角色除了身份，还绑定了**一组岗位画像**（SpecialistProfile），画像声明了这个角色覆盖哪些"专长域"，而专长域再向下绑定技能、工具、工作站配置。身份是"你是谁"，画像是"你会什么"。这两层是分开的。

```
角色装配链
├── agent_config              身份（name/profile/goal/constraints/nickname/emoji）
│   └── 人格、原则、提示词
├── SpecialistProfile         专长画像（这个角色覆盖哪些岗位域）
│   ├── orchestration          项目总指挥域
│   ├── techManagement         技术管理/编码域
│   ├── sre                    SRE 域
│   ├── qualityAssurance       质量域
│   └── ...                    （每个域定义一组能力默认值）
└── 能力绑定（由画像驱动）
    ├── 技能集（L1 ProjectSkillScopeService 过滤）
    ├── 工具策略（project_tool_policy role_id 维度）
    └── 工作站配置（workstation type→engine→skill 树）
```

身份和画像分离，是 WinMatrix 角色模型的核心抽象之一。

---

## 画像为什么必须独立于人格

为什么不把"会什么"直接写进 prompt？这是这一篇最值得想清楚的设计权衡。

**第一，prompt 是建议，不是约束。** LLM 完全可能在 prompt 注入或自身幻觉下，去尝试调用一个它"名义上不该有"的工具。如果能力边界只靠 prompt 暗示，那这边界是软的、可被绕过的。WinMatrix 在工程层把能力边界做成了硬约束——L3 readiness gate（`SkillReadinessGate.check`）是 SSOT，"在 L3 直接拒绝，不 fallback、不按名称/关键词推断"（`types.ts:85-88`）。这种硬约束的前提，是有一个结构化的"该角色有哪些能力"的清单——那就是画像。

**第二，身份和能力是两个独立的演进维度。** 一个角色的"人格"（怎么说话、什么风格）变化频率低，且改动往往只是文案。但"能力"（能调哪些工具、绑定哪些技能）会随业务频繁变化——新技能上线、工具权限调整、工作站引擎升级。如果能力写死在 prompt 里，每次能力调整都要改提示词模板、重新回归测试所有对话；而如果能力是独立的画像绑定，调整只影响能力层，人格层纹丝不动。**演进隔离，是分层最朴素也最实在的收益。**

**第三，画像要能驱动"默认值"，而不是每次显式配置。** 想象一个新项目初始化，要给"技术管理者"这个角色配上它能用的技能和工具。如果没有画像，管理员得手动一项项勾选；有了画像，系统知道"技术管理者覆盖 techManagement 域"，这个域的默认技能集（代码评审、技术方案、工作站编码）和默认工具策略就自动生效。画像让"新建一个技术管理者"这件事变成零配置。

这三个理由合起来就是一句话：**能力边界必须是结构化的、可演进隔离的、能驱动默认值的——这些都要求它独立于 prompt 存在。**

---

## 画像驱动的三层能力绑定

画像声明了"覆盖哪些域"，但光有声明不够，还要让这个声明在运行时真正生效。WinMatrix 让画像驱动三个地方。

### 第一层：技能过滤（L1 scope）

技能在 L1 决策阶段（最便宜的过滤）要被"这个角色能用哪些技能"收窄。这一步的 SSOT 是 `ProjectSkillScopeService.getPersonaEligibleSkillNames()`（`ProjectSkillScopeService.ts:89-101`），它从 `skill_artifact` 表读 `enabled ∩ persona_eligible=true`。注释里有一句很关键的话——**"禁止复用 skill_artifact.scope"**。

```
skill_artifact
├── scope              技能作用范围（global/project/...）—— 不能直接当角色过滤用
└── persona_eligible   分身/角色可用性真源（默认 true）
    ├── true  = 默认对所有相关角色生效
    └── false = 不进角色的 L1 列表
```

`persona_eligible` 这个字段（`schema.prisma:1296-1304`）注释写得很直白——"分身技能可用性真源（design D3.1/D3.2）：默认 true=默认继承；false=不进 personal L1"。

它的设计意图是：**一个技能要不要对某类角色生效，是技能自己的属性，不是角色配置的属性。** 技能声明"我适合哪些专长域"，画像声明"这个角色覆盖哪些域"，两者在 L1 求交。这比"每个角色维护一个技能白名单"灵活得多——新增技能时只要声明它属于哪个域，所有覆盖该域的角色自动获得它。

配合数字分身同源继承原则（AGENTS.md:104-110："项目空间新增或变更能力时，默认同时对分身生效……禁止为分身新建平行实现"），这套机制让"一个新技能上线，立刻对所有技术线角色和它们的分身可用"成为可能——零配置扩散。

### 第二层：工具策略（project_tool_policy）

工具层的策略表 `project_tool_policy`（`schema.prisma:1414-1434`）颗粒度极细：

```
project_tool_policy
├── project_id            项目
├── role_id?              角色（画像驱动的默认策略落在这）
├── digital_employee_id?  具体员工（可覆盖角色默认）
├── tool_name             工具
├── effect                allow | deny
├── access_mode           member | visitor
├── source                manual | role_default | system
└── status                active | disabled
```

注意 `source=role_default`——这就是画像驱动的默认工具策略。当一个技术管理者角色被新建，系统会按它的画像预置一批 `role_default` 策略：工作站执行工具 allow、代码读写工具 allow、而"删除项目"这种 destructive 工具 deny。管理员后续可以用 `source=manual` 的策略覆盖这些默认值。

这里有个值得说的分层：`role_id`（角色级默认）和 `digital_employee_id`（员工级覆盖）可以同时存在，**员工级覆盖角色级**。这意味着"阿码"这个角色有一套默认工具权限，而具体某个技术管理者员工可以被单独收紧或放宽。画像驱动默认，个体允许差异化——这是企业级 RBAC 的标准形态。

### 第三层：工作站配置（workstation 树）

工作站是更重的能力——它不只是个工具调用，而是一个 K8s Pod 里的完整编码环境。工作站配置走"type→engine→skill"树（`schema.prisma:2219`起）：

```
workstation_type_config   工作站类型（coding/sre/openclaw）
    ↓ 选引擎
workstation_agent_engine  类型绑哪个引擎（claude_code/codex/hermes/openclaw）
    ↓ 绑技能
workstation_agent_skill   引擎跑哪些技能
    ↓ 装组件
workstation_component     installScript + checksum（每个组件的可执行脚本）
```

技术管理者的画像绑 `coding` 类型工作站（claude_code 引擎），SRE 角色绑 `sre` 类型——不同画像拿到的是完全不同的执行环境。`agent_config.workstation` 字段（`schema.prisma:1159-1181`）存的就是这个工作站配置，而且带了 `workstation_config_version` 做乐观锁——改工作站配置要走 version 校验（参考第 9 篇的乐观锁机制），防止两个管理员并发改乱。

**画像在这一层的作用是：决定一个角色"配不配拥有工作站"以及"配哪种工作站"。** 产品经理角色根本不会进工作站配置树——它的画像里没有 techManagement/sre 域。

---

## 画像与 L1/L2/L3 的配合

前面三层讲的是"画像驱动默认值"。但能力最终能不能在运行时执行，还要过 L1/L2/L3 三道关（参考第 13 章）。画像和这三道关的关系值得单独理清。

| 层 | 语义 | 画像的作用 |
|----|------|-----------|
| L1 | 技能存不存在、对角色可不可见（最便宜） | **画像直接决定**——`persona_eligible ∩ 画像域` 过滤 |
| L2 | 依赖的输入满不满足（snapshot 软校验） | 画像不参与，纯契约检查 |
| L3 | 运行时资源就绪（硬拦，SSOT） | 画像不参与，只看凭证/环境是否就绪 |

关键观察：**画像只在 L1 起作用。** L2、L3 是跨角色通用的契约/资源检查，和"这个角色是什么画像"无关。这个分工很干净——画像解决"该不该有这个能力"，L2/L3 解决"这个能力现在能不能跑"。

这也解释了为什么 L1 的实现（`ProjectSkillScopeService`）注释里反复强调"轻量、仅 listAvailable、不做 binding/preflight"——因为 L1 是画像过滤的执行点，它会被决策阶段高频调用（每条消息都要列出可用技能），必须极快。画像过滤是 O(技能数) 的内存操作，不碰 DB、不做 I/O，正好满足这个性能要求。

---

## 第八类角色：画像的反面教材

讲一个有趣的对照。前面七大角色都有静态画像（写死在实现类和 agent_config 里）。但第八类 external-agent（外联 Agent）没有静态画像——它是运行时动态生成的（`DigitalEmployee.ts:396-401`）。

外联 Agent 的装配（`externalAgentBootstrap.ts:73-101`）：

```ts
const EXTERNAL_AGENT_ROLE_ID = 'external-agent';
const registrations = await prisma.external_agent_registration.findMany({
  where: { userId, status: 'active', isConnected: true },
});
for (const reg of registrations) {
  result.push({
    id: `ext_${reg.id}`,
    roleId: EXTERNAL_AGENT_ROLE_ID,
    name: reg.name?.trim() || reg.agentType,
    isExternal: true,
    externalAgentId: reg.id,
  });
}
```

外联 Agent 的"能力"不是来自画像，而是来自注册时声明的 `tools` 字段和 `capabilityProfile`（`external_agent_registration` 表）。它是个**运行时自描述的能力集合**，没有预置的岗位域归属。

这是个刻意的取舍：外联 Agent 接入的可能是 Hermes、OpenClaw 之类完全异构的系统，你没法预先给它们定义"属于哪个专长域"。所以它们的画像退化为"自带工具列表"，能力边界由注册数据决定，而不是由系统画像匹配。

**这个对照反过来证明了画像的价值。** 七大内置角色因为有画像，能力可以零配置扩散、可以做域级治理、可以驱动工作站树；外联 Agent 没有画像，每个能力都要显式声明、无法享受域级默认值。**画像的本质是"把零散的能力收敛成可治理的域"——你放弃这个收敛，就得回到逐项配置的原始状态。**

---

## 画像与数字分身：同源继承的实现基础

最后讲一个画像的延伸价值——它是"数字分身同源继承"能成立的前提。

AGENTS.md 的强制条文（104-110）：

> 项目空间新增或变更能力时，默认同时对分身生效。不适用分身的必须显式加入排除名单。禁止为分身新建平行实现；需要差异化时用参数或排除名单，不复制一套 service/组件。

这条原则的落地，靠的就是 `persona_eligible` 字段。分身（personal employee）运行在隐藏的个人项目（`projects.kind='personal'`）里，它的能力栈和项目空间角色同源。当一个新技能上线，只要 `persona_eligible=true`，它就同时进入：

- 所有覆盖该域的**项目空间角色**的 L1 列表
- 这些角色的**所有分身**的 L1 列表

如果某个技能明确不适用于分身（比如它依赖项目级凭证，分身没有），就把 `persona_eligible` 设成 false——这是"显式排除名单"。整个过程不需要给分身单独配一遍，也不需要写平行实现。

**画像 + persona_eligible，是把"分身继承"从一句口号变成可执行机制的两块拼图。** 没有画像，"继承"就无从谈起——你不知道一个能力属于哪个域，就不知道它该不该对分身生效。

---

## 给后来者的几条总结

1. **能力边界要独立于人格建模。** prompt 是建议不是约束；把"会什么"写进 prompt，既拦不住越权，也经不起演进。用一张结构化画像显式声明能力域。
2. **画像的本质是把零散能力收敛成可治理的域。** 放弃这个收敛（如外联 Agent），就回到逐项配置的原始状态。七大角色有画像、能域级治理；外联 Agent 没画像、只能逐项声明——这就是对照。
3. **画像驱动三层默认值**：技能过滤（L1 的 persona_eligible）、工具策略（role_default）、工作站配置（type→engine→skill 树）。让"新建一个角色"变成零配置。
4. **画像只在 L1 起作用，L2/L3 与画像无关。** L1 解决"该不该有"，L2/L3 解决"现在能不能跑"。分工要干净，别让画像逻辑渗透到运行时校验里。
5. **技能声明自己属于哪个域，角色声明自己覆盖哪些域，两者求交。** 这比"每角色维护白名单"灵活——新技能自动扩散到所有相关角色。
6. **persona_eligible 是分身继承的执行字段。** 默认 true=继承，false=显式排除。同源继承靠它落地，不靠口号。
7. **演进隔离是分层最实在的收益。** 人格层（prompt）和能力层（画像绑定）变化频率差一个数量级，分开建才不会互相拖累。
8. **L3 是硬约束的 SSOT，画像只是它的输入。** "直接拒绝，不 fallback、不按名称/关键词推断"——硬约束的前提是有结构化的能力清单，而不是 prompt 暗示。

数字员工"会什么"，从来不是一个 prompt 问题，而是一个工程问题。把能力边界做成结构化画像，是让数字员工从"看起来会"变成"真的只会"的关键一步。

---

> **下一篇**：[《协作催促 CollaborationFollowup：被同事卡住了，系统怎么催》](./48-collaboration-followup.md)——讲完了单个员工的能力画像，下一篇转到协作：当一个员工干活时被另一个角色卡住，系统怎么自动检测、延时催促、并把整个催促生命周期变成可观测事件。
