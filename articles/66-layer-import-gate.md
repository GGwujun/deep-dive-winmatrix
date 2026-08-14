# 分层 import 门禁：check:agent-layers:strict 怎么守住六层

> 这是《WinMatrix 开发经验文集》第 66 篇。讲一个真实的工程痛点：一个三十万行的 TypeScript 项目，团队十几个人同时改，分层架构怎么"不靠纪律守住"。代码来自 WinMatrix 后端真实实现，没有杜撰。

做平台的人大概都有过这样的经历：项目刚开始时，架构图画得清清楚楚，"这个层不能调那个层"写在 wiki 里。半年后你随手 grep 一下，发现 infrastructure 反向 import 了 agents，business 直接 new 了一个 Role 类，channel 和 connectors 互相耦合成了死结。

不是谁故意破坏架构。是凌晨两点改 bug 时，"就 import 这一下能省两小时"，纪律就这么一点一点被蚕食。

这一篇讲我们怎么用两个门禁脚本把"分层"从口头纪律变成 CI 硬约束。核心是两个看似反直觉的设计：**存量给白名单、增量零容忍**，以及**横向隔离比纵向分层更难守**。

---

## 现象：分层腐化是怎么发生的

WinMatrix 是一个严格分层的项目，仓库大层是五层：

```
L6 Interface（API / Channel / Middleware）   最外层
L2 Agents（Agent Runtime，内部还有六子层）
L3 Business-Tools（agents 唯一可调的业务门面）
L4 Business（领域服务）
L1 Infrastructure（基础设施）                最内层
```

依赖方向应该是单向的：外层可以依赖内层，内层不能反向依赖外层。比如 `agents` 可以调 `business-tools`，但 `business` 不能调 `agents`，`infrastructure` 更不能调 `agents`。

听起来简单。但实际跑起来，腐化会在三个地方悄悄发生：

**第一，跨层反向 import。** 一个 infrastructure 下的工具函数，为了拿个配置，顺手 import 了 `agents/core/kernel-management/config/configManager`。这一下就把"最内层依赖最外层"的禁令打破了。

**第二，Agent 内部子层越界。** Agents 内部其实还有六个子层（ai-kernel、ai-execution、worker、agent、kernel-management、runtime）。`ai-kernel` 是最底层的"原语层"，不该依赖 `worker`（具体角色运行时）；但一个 context adapter 为了读角色信息，顺手 import 了 `worker/role/BaseRole`。这一下就把 ai-kernel 的纯净性毁了。

**第三，横向耦合。** 这个最隐蔽。`agents/harness/`（L3 通用框架）和 `agents/domain-harness/`（L4 业务定制）是同层但不同包，按理不该互相 import；`interface/channel/`（接入层）和 `integration/connectors/`（连接器）也是。但为了复用一段代码，开发者很容易在这两对之间架一条"捷径"。时间一长，这两个本该可独立演进的包就焊死在一起了。

腐化的共同特征是：**每一次违规都很小、很合理，但累积起来，架构图就成了摆设。**

---

## 根因：为什么 code review 守不住

很多人会问：分层违规，code review 卡掉不就行了？

实践中我们发现，code review 对分层腐化的防御力很弱，原因有三个：

1. **违规的"合理性"太强。** 改 bug 的人 import 一个本不该 import 的符号，往往是因为那是当下最快的解法。reviewer 看到这个 PR，如果没有明确的规则手册，很难说"不行"——毕竟业务压力摆在那。
2. **规则太复杂，人脑记不住。** 我们的分层规则有 R1 到 R7 七条，每条对应不同的包对。reviewer 不可能每次都对着规则表逐条核对。
3. **存量违规是新违规的借口。** "你看那边已经有一个 infrastructure import agents 了，我再加一个怎么了？" 一旦有存量，纪律就滑坡。

根本问题是：**分层是一种结构性约束，靠人盯人一定会漏。** 它必须是 CI 的硬约束——就像编译错误一样，不通过就不让合。

但这里又有个现实矛盾：项目已经三十万行了，里面已经有一些存量违规。如果你今天上一个"零容忍"的门禁，CI 会立刻挂掉几百个文件，谁也没精力去清。门禁上不去，分层就继续腐化。

这是个鸡生蛋的问题。

---

## 修复：双门禁 + 存量白名单 + 增量零容忍

我们的解法是两个脚本配合，形成"两道闸"：

```
┌──────────────────────────────────────────────────────────┐
│  check:layers             （大层门禁，存量白名单）        │
│  规则：infrastructure ↛ agents；agents ↛ business         │
│  策略：存量违规进 STATIC_IMPORT_WHITELIST，只要不新增就过 │
│                                                            │
│  check:agent-layers:strict （Agent 子层门禁，零容忍）     │
│  规则：R1–R7（见下表）                                     │
│  策略：strict 模式要求 ALLOWLIST 必须清零，否则 exit(1)   │
└──────────────────────────────────────────────────────────┘
```

### 第一道闸：check:layers（大层门禁，存量容忍）

这道闸管的是最粗粒度的两条铁律：infrastructure 不能 import agents，agents 不能 import business。脚本在 `scripts/check-layer-imports.cjs`。

它的核心设计是"白名单 + 不新增即可"。脚本顶部有：

```js
// scripts/check-layer-imports.cjs
const DYNAMIC_IMPORT_WHITELIST = [];
const STATIC_IMPORT_WHITELIST = [];
```

扫描时，如果某个违规 import 命中白名单，直接跳过。`isStaticWhitelisted` 命中就 `return []`（返回空违规列表）。

这条规则的潜台词是：**我们承认存量违规的存在，不要求一夜清零，但绝不允许新增。** 今天有 29 处违规，明天不能变成 30 处。每个想新增违规的人，都得先想办法消掉一处旧的——把"还债"变成了"加新违规的前置条件"。

大层门禁用 type-only 豁免来缓解"只想用个类型"的合理诉求：

```js
const TYPE_ONLY_IMPORT_REGEX = /import\s+type\s+(?:[\w*\s,{}]+)\s+from\s+['"]([^'"]+)['"]/g;
// ...
while ((typeMatch = TYPE_ONLY_IMPORT_REGEX.exec(content)) !== null) {
  typeOnlyPaths.add(typeMatch[1]);  // type-only import 不算违规
}
```

`import type { Foo } from '@/business/...'` 只引入类型、运行时不产生依赖，所以放行。这处理掉了相当一部分"只是想用接口定义"的合理跨层引用。

### 第二道闸：check:agent-layers:strict（子层门禁，零容忍）

这道闸才是真正的硬骨头。它管的是 Agent Runtime 内部六子层 + 几条横向隔离规则，对应 R1 到 R7。脚本在 `scripts/check-agent-layer-imports.cjs`，由 `npm run check:agent-layers:strict` 触发：

```json
// package.json:72
"check:agent-layers:strict": "node scripts/check-agent-layer-imports.cjs --strict && npm run check:tool-kernel-consumption"
```

七条规则的真实代码如下（节选自 `check-agent-layer-imports.cjs`）：

| 规则 | 约束 | 代码函数 |
|------|------|----------|
| R1 | infrastructure/ 下不得有产品目录（dashboard/identity/session 等） | `checkInfrastructurePurity` |
| R2 | agents/core/ 不得出现具体角色类名（如 `OrchestratorRole`） | `checkL2DomainPurity` |
| R3 | agents/harness/ 不得 import 具体角色类、不得 `instanceof XxxRole` | `checkHarnessIsolation` |
| R4 | agents/harness/ ↔ agents/domain-harness/ 横向隔离 | `checkL3L4PackageIsolation` |
| R5 | business/domain/flowOrchestration/ 不得 export 定义类型 | `checkL4L5DefinitionBoundary` |
| R6 | interface/channel/ ↔ integration/connectors/ 横向隔离 | `checkChannelIntegrationBoundary` |
| R7 | business-tools/ 不得 import DB 前缀、不得有 recipe/ 目录 | `checkBusinessToolsClassification` |

我重点讲其中最值钱的两条——横向隔离。

### R4：L3 ↔ L4 横向隔离

L3 `agents/harness/` 是通用框架层（能力治理、技能编排、学习蒸馏），L4 `agents/domain-harness/` 是业务定制层（七大业务 Role 实现）。它们是同层（都在 agents 下），但语义上是"框架"和"定制"的关系。

规则的定义只有两行：

```js
// scripts/check-agent-layer-imports.cjs:72
const L3_L4_ISOLATION_PREFIXES = [
  { from: 'agents/harness/', forbidden: '@/agents/domain-harness/' },
  { from: 'agents/domain-harness/', forbidden: '@/agents/harness/' },
];
```

检查逻辑：

```js
// R4 — L3/L4 横向隔离
function checkL3L4PackageIsolation(relPath, imports) {
  for (const { from, forbidden } of L3_L4_ISOLATION_PREFIXES) {
    if (!relPath.startsWith(from)) continue;
    for (const imp of imports) {
      if (imp === forbidden.replace(/\/$/, '') || imp.startsWith(forbidden)) {
        return [{ rule: 'r4:l3-l4-package-isolation', importPath: imp }];
      }
    }
  }
  return [];
}
```

为什么这条这么重要？因为 **L3 是可以被多个 L4 复用的稳定框架，L4 是随业务频繁变化的定制层**。一旦 L3 反向 import 了某个具体 L4（比如某个 domain Role），L3 就被这个业务绑死了——别的业务想复用 L3，得把这个业务也拖进来。反过来 L4 import L3 是允许的（定制依赖框架），但 L3 import L4 绝对不行。

这条规则的深层含义是：**分层不只是"上下"，还有"左右"。** 纵向分层（不能反向依赖）大家容易想到，但横向隔离（同层不同包之间不互相依赖）往往被忽略，而它恰恰是"可独立演进"的关键。

### R6：channel ↔ connectors 横向隔离

另一条横向隔离规则：

```js
// scripts/check-agent-layer-imports.cjs:77
const CHANNEL_INTEGRATION_ISOLATION = [
  { from: 'interface/channel/', forbidden: '@/integration/connectors/' },
  { from: 'integration/connectors/', forbidden: '@/interface/channel/' },
];
```

`interface/channel/` 是消息接入层（企微 AiBot、Webhook、定时任务触发的 channel），`integration/connectors/` 是外部系统集成层（企微 API、外部 Agent RPC、wedoc）。两者都涉及"企微"，很容易互相耦合——channel 想直接调 connector 的发消息 API，connector 想直接把事件推给 channel 处理。

但它们必须隔离，原因是 **职责分离**：channel 负责"消息怎么进系统"，connector 负责"系统怎么调外部"。让它们互相独立，意味着可以换 channel 不换 connector（比如加个钉钉 channel，企微 connector 不动），反之亦然。焊死之后，任何一边的改动都会牵动另一边。

### strict 模式：白名单必须清零

第二道闸最狠的地方是 `--strict` 标志。普通模式下，命中 `ALLOWLIST` 的违规会被静默跳过；但 strict 模式要求 `ALLOWLIST` 本身也必须清零：

```js
// scripts/check-agent-layer-imports.cjs:571
if (strict && infraLlmAllowlist.length > 0) {
  console.log(`❌ strict 模式：infra-llm allowlist 仍有 ${infraLlmAllowlist.length} 文件组未清零\n`);
  process.exit(1);
}
if (strict && deferredAllowlist.length > 0) {
  console.log(`❌ strict 模式：deferred allowlist 仍有 ${deferredAllowlist.length} 文件组未清零\n`);
  for (const entry of deferredAllowlist) {
    console.log(`   - [${entry.rule}] ${entry.file}`);
  }
  console.log('');
  process.exit(1);
}
if (strict) {
  console.log('  strict 通过：allowlist 已清零（含 R1–R7）\n');
}
```

也就是说，`check:agent-layers:strict` 是一个**会随时间收紧的约束**：今天 ALLOWLIST 是空的，明天谁想加一条，CI 立刻拒绝。这把门禁从"防止新增"升级成了"强制还债"。

这套设计的关键是**两道闸的松紧搭配**。第一道闸（check:layers）宽松，容忍存量，让项目能继续往前走；第二道闸（check:agent-layers:strict）严苛，对最核心的 Agent 子层零容忍，保证最关键的部分不腐化。如果两道都严，团队被存量压垮上不了 CI；如果两道都松，形同虚设。一松一紧，才是能落地的工程方案。

---

## 教训

回头看，这套门禁给我们带来的教训有几条。

**第一，架构防腐要靠脚本，不要靠纪律。** 纪律是人，人是会疲惫、会妥协、会忘规则的。而 CI 不会。一条 `check:agent-layers:strict` 在 pipeline 里挂着，它会在每一个 PR 上无情地执行，不管提 PR 的人是实习生还是架构师。把结构性约束从"code review checklist"挪到"CI hard fail"，是架构能长期保鲜的唯一办法。

**第二，存量用白名单、增量零容忍，是上线的唯一可行姿势。** 如果要求"一步到位清零所有存量"，门禁永远上不了线。允许存量进白名单，但禁止新增，给了团队一个渐进还债的窗口。而 strict 模式对子层的"白名单清零"要求，则是把"还债"本身也变成了硬约束。这套节奏（先增量冻结，再存量攻坚）适用于任何想给老项目加架构约束的场景。

**第三，横向隔离比纵向分层更难、更值钱。** 纵向分层（不能反向依赖）大家容易想到，因为依赖图直观。但横向隔离（L3↔L4、channel↔connectors）往往被忽略，而它恰恰是"能不能独立演进、独立复用"的关键。我们在 R4 和 R6 上花的力气，比纵向分层多得多——因为纵向违规一眼就能看出来，横向违规要不是脚本扫，根本发现不了。

**第四，type-only 豁免是分层的必要减压阀。** 很多跨层 import 其实只是想用一个类型定义（接口、类型别名），运行时不产生依赖。`import type` 的豁免处理掉了这部分合理诉求，让门禁聚焦在"真正的运行时依赖"上，否则门禁会被大量误报淹没，最终被开发者绕过（比如加 `// eslint-disable`）。

**第五，门禁脚本本身要有版本化设计。** 注意 `ScheduledCronMigrationLog` 那个 `migrationVersion` 字段——我们给"迁移"加了版本号。分层门禁也是同理：R1-R7 是会演进的，`ALLOWLIST` 是会收缩的。每加一条规则、每清一条白名单，都应该是一个有记录的动作，而不是静默修改脚本。架构约束本身也需要可审计。

最后一个更宏观的体会：**分层不是画图，是执行。** 一张漂亮的架构图谁都会画，但它能不能在三十万行代码、十几个人、一年迭代之后还成立，取决于你有没有给它配上"自动守卫"。WinMatrix 的分层能守住，不是因为我们在 wiki 里写得清楚，而是因为 `npm run check:agent-layers:strict` 在每一个 PR 上跑着。模型是引擎，分层是底盘，门禁是底盘上的那道焊缝——焊缝在，底盘才散不了。

---

> **下一篇**：[《启动序列的坑：DI 注册顺序错了为什么直接崩》](./67-startup-di-order.md)——讲 Port 注入的顺序依赖、`assertCoreDiRegistrations` 的 fail-fast 断言，以及"adapter 必须 4 个否则启动直接抛错"背后的设计意图。
