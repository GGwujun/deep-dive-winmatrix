# 启动序列的坑：DI 注册顺序错了为什么直接崩

> 这是《WinMatrix 开发经验文集》第 67 篇。讲一个我们踩过好几次的坑：依赖注入（DI）的注册顺序、Port 注入的时序，以及为什么我们要在启动序列里塞一堆"看起来多余"的断言。代码来自 WinMatrix 后端真实实现，没有杜撰。

做后端的人对"启动慢"通常不陌生，但对"启动直接崩"往往准备不足。一个 Node 进程，启动时 `npm start` 一跑，屏幕上一闪就退出了，留下一个 `Cannot read properties of undefined (reading 'xxx')` 或者 `adapter 注册数期望 4，实际 0`。

这类崩溃的特点是：**它不在运行时发生，而在启动序列里发生；bug 不在业务逻辑里，而在"初始化的顺序"里。** 而且特别难调——因为进程已经退了，你没法 attach 调试器，只能靠日志倒推。

这一篇讲 WinMatrix 启动序列里的三个坑：DI 注册顺序、Port 注入时序、以及为什么我们对"adapter 必须 4 个"这种数字做硬断言。核心论点是：**启动期是整个系统最脆弱的瞬间，必须用 fail-fast 断言把所有"顺序依赖"显式化。**

---

## 现象：三种"启动即崩"的姿势

WinMatrix 的启动序列在 `src/startup/common.ts`，分阶段执行：阶段 1 注册服务、连数据库，阶段 2 初始化配置和缓存，阶段 3 初始化记忆、图谱，等等。这套序列在 dev 和 prod 都跑。

我们踩过的"启动即崩"大致分三类。

**第一类：DI 容器 resolve 不到服务。** 某天有人在 `registerServices()` 里加了一个新 service，但忘了注册；或者注册了，但注册顺序写在了某个依赖它的初始化步骤之后。进程启动到一半，某个 repository 调 `container.resolve('IFooRepository')`，拿到 undefined，下一行 `.findById()` 直接 `TypeError`。

**第二类：Port 注入了 undefined。** Agent 层不直接 import Business（这是分层铁律，见上一篇 66），而是通过 17+ 个业务 Port（`setConversationServicePort`、`setSkillManifestPort` 等）在启动时注入。如果 Port 的实现依赖某个还没初始化好的单例（比如 configManager 还没 initialize），注入的就是个"半成品"对象，运行时调用某个方法才发现它内部状态是空的。

**第三类：side-effect adapter 没注册。** 这是最隐蔽的一类。我们有 4 个 side-effect adapter（发通知、发邮件、发 markdown、发图文消息），靠 `import '@/infrastructure/integration/sideEffect/registerAdapters.js'` 这一句 side-effect import 注册。如果有人重构时不小心把这一行挪到了"不该挪的地方"，或者 tree-shaking 把它优化掉了，启动时 adapter 注册数为 0——但进程不会立刻报错，而是等运行时第一次调用 `assertAdapterInput` 才发现 registry 是空的。

这三类的共同点是：**根因都是"顺序"，但表现都像"运行时空指针"。** 如果你不在启动期主动检测，它们会潜伏到业务高峰期才爆发。

---

## 根因：启动序列里藏着大量隐式顺序依赖

为什么启动序列这么脆弱？因为 DI + Port 注入这套模式，本质上是**用代码顺序来声明依赖图**。

看 `startup/common.ts` 的阶段 1（`initInfrastructure`，行 242-285）：

```typescript
// src/startup/common.ts:242
export async function initInfrastructure(options?: RoleInitOptions): Promise<void> {
  const role = resolveInitRole(options?.role);
  logStep(1, '注册服务到依赖注入容器...');
  const t0 = Date.now();
  registerServices();              // ① 注册 DI
  await registerAgentOwnedSingletons();  // ② 注册 Agent 拥有的单例（含 wireAgentBusinessPorts）
  registerWorkstationServices();   // ③ 注册工作站服务
  assertCoreDiRegistrations();     // ④ 断言核心 DI 已注册
  // ⑤ side-effect adapter 数量断言
  const adapterCount = listAdapters().length;
  if (adapterCount !== 4) {
    throw new Error(
      `[Startup] side-effect adapter 注册数期望 4，实际 ${adapterCount}。` +
        '请检查 registerAdapters.ts 是否被 import（应在 startup/common.ts 顶部 side-effect import）。',
    );
  }
  // ⑥ LLM Provider Registry 初始化（rag 跳过）
  if (shouldInitLlmProviderRegistry(role)) {
    const llmFactory = await import('@/infrastructure/llm/llmFactory.js');
    llmFactory.initProviderRegistry();
  }
  // ⑦ 连 Prisma
  await connectPrisma();
  // ⑧ 时区一致性校验
  await warnIfTimezoneInconsistent();
  // ⑨ 确保系统用户
  await ensureSystemUsers();
}
```

这九步里，每一步都可能依赖前面某一步的产物。比如 `ensureSystemUsers()` 要读数据库，所以必须在 `connectPrisma()` 之后；`connectPrisma()` 又得在 DI 注册之后（因为连接对象要被 DI 容器管理）。这些顺序依赖是"隐式的"——它们不在类型系统里，不在编译期检查，全靠开发者脑补。

更危险的是 `registerAgentOwnedSingletons()`（行 159-189），它内部调用了 `wireAgentBusinessPorts()`：

```typescript
// src/startup/common.ts:159
export async function registerAgentOwnedSingletons(): Promise<void> {
  registerBuiltInPolicies();
  const toolConfigAdapter = defaultToolConfigPortAdapter;
  injectToolConfigPort(toolConfigAdapter);
  // ... 一堆 tool kernel/visibility 的 Port 设置 ...
  wireKernelPortsFromExecution();
  wireAgentBusinessPorts();        // ← 这里注入 17+ 业务 Port
  assertAgentBusinessPortsWired(); // ← 这里断言所有 Port 都已注入
  wireInfraAgentPorts();
  container.register('ConfigManager', () => configManager);
  // ... 其他单例注册 ...
}
```

`wireAgentBusinessPorts()` 会把 17+ 个业务 Port 一个个 set 进去（见 `startup/wireAgentBusinessPorts.ts:123`）：

```typescript
// src/startup/wireAgentBusinessPorts.ts（节选）
export function wireAgentBusinessPorts(): void {
  setConversationServicePort(getConversationService());
  setConversationMessageAppendPort(getConversationMessageAppendService());
  setRoleInboxPort(roleInboxEnqueueService);
  setIdentityPort({...});
  setDigitalEmployeeBootstrapPort({...});
  setProjectContextPort({...});
  setSkillManifestPort({...});
  setSkillCredentialResolutionPort({...});
  setCodingTaskPort({...});
  setWorkstationPort({...});
  setAgentPromptTemplatePort({...});
  setExternalAgentActivityPort({...});
  setFlowOrchestrationPort({...});
  setKickoffExecutionPort({...});
  setPromptOverridePort({...});
  // ...
}
```

注意这里有个隐藏的顺序依赖：**很多 Port 的实现内部又会去 `container.resolve` 别的 service**。比如 `getConversationService()` 可能 resolve `IMemoryService`，而后者必须已经在 `registerServices()` 里注册过。如果 `registerServices()` 漏注册了 `IMemoryService`，`wireAgentBusinessPorts()` 这一行就会拿到 undefined，注入的 Port 就是坏的。

但坏 Port 不会立刻报错。它会潜伏到运行时——某个 Turn 执行时调 `conversationServicePort.append()`，才发现这个方法不存在。这时距启动已经过了好几分钟，日志里根本看不出是注册顺序的锅。

**这就是启动期最大的陷阱：顺序错误不会立刻炸，而是变成一颗埋在运行时的地雷。**

---

## 修复：把所有顺序依赖用 fail-fast 断言显式化

我们的修复策略是一句话：**启动期能查出来的问题，绝不让它活到运行时。** 具体靠三类断言。

### 断言一：assertCoreDiRegistrations——核心 service 必须都在

在 `registerServices()` → `registerAgentOwnedSingletons()` → `registerWorkstationServices()` 三步都跑完后，立刻调 `assertCoreDiRegistrations()`：

```typescript
// src/startup/common.ts:250
registerServices();
await registerAgentOwnedSingletons();
registerWorkstationServices();
assertCoreDiRegistrations();   // ← 核心服务注册完整性断言
```

这个函数（在 `infrastructure/di/serviceRegistrations.ts`）会逐个检查"启动必须存在的"核心 service 有没有被注册。任何一个缺失，立刻抛错，进程退出。错误信息会明确指出是哪个 service 没注册，开发者能直接定位是 `registerServices()` 里漏了一行，还是顺序写错了。

它的价值不是"检测 bug"——单测也能检测。它的价值是**检测得足够早**：在所有业务代码加载之前，在进程开始监听端口之前。这样你看到错误时，进程还没对外提供服务，不会留下半初始化的脏状态。

### 断言二：assertAgentBusinessPortsWired——17+ Port 全部注入

紧接着 `wireAgentBusinessPorts()` 之后，有一行 `assertAgentBusinessPortsWired()`（行 171）：

```typescript
// src/startup/common.ts:170
wireAgentBusinessPorts();
assertAgentBusinessPortsWired();   // ← Port 注入完整性断言
```

这个函数在 `agents/core/agent/shared/ports/agentBusinessPortRegistry.ts` 里，它会遍历所有已知的 Port setter，检查每一个 Port 的值是否非空。如果有任何一个 Port 还是 undefined（说明 `wireAgentBusinessPorts` 漏调了某个 setter，或者 setter 内部 resolve 失败），立刻抛错。

这个断言解决的是"Port 注入了 undefined"那类坑。它把"Port 是否完整"从一个"运行时某个方法调用时才发现"的问题，变成了"启动期立刻暴露"的问题。17+ 个 Port，少一个都活不过启动。

### 断言三：adapter 数量必须等于 4

最值钱的一条断言，是 side-effect adapter 的数量断言（行 252-259）：

```typescript
// src/startup/common.ts:252
// 启动断言：4 个 side-effect adapter 必须已注册（覆盖 send_notification / send_email / markdown / mpnews）
// 失败即抛错快速失败，避免运行时首次调用 assertAdapterInput 才发现 registry 为空（D9 / R1）
const adapterCount = listAdapters().length;
if (adapterCount !== 4) {
  throw new Error(
    `[Startup] side-effect adapter 注册数期望 4，实际 ${adapterCount}。` +
      '请检查 registerAdapters.ts 是否被 import（应在 startup/common.ts 顶部 side-effect import）。',
  );
}
logger.info(`[Startup] side effect adapters registered: ${adapterCount}`);
```

注意它的设计细节，这些细节才是精髓：

**为什么是严格等于 4，而不是 ≥1？** 因为这 4 个 adapter（通知、邮件、markdown、图文）是功能完整性的硬要求。少一个，某类消息就发不出去。如果用 `≥1`，你可能会在"通知能发但邮件发不了"的状态下启动成功，问题潜伏到用户投诉"收不到邮件"才暴露。`!== 4` 把"少注册一个"也当成致命错误。

**为什么检查数量而不是逐个检查名字？** 数量检查更宽松——它不关心你注册的是哪 4 个，只关心"是不是正好 4 个"。这给了 adapter 实现重构的空间（你可以换一个 adapter 的实现，只要总数对）。但如果有人删了一个 adapter 又加了一个不同类型的，数量检查会发现不了——所以我们用数量 + 注释里列明 4 个名字的组合，兼顾宽松和可读。

**为什么在顶部 side-effect import？** 看文件顶部（行 4-5）：

```typescript
// side-effect import: 注册 4 个 side-effect adapter（H 阶段 bootstrap，覆盖 api/scheduled/rag/all-in-one 全进程）
import '@/infrastructure/integration/sideEffect/registerAdapters.js';
import { listAdapters } from '@/infrastructure/integration/sideEffect/adapterRegistry.js';
```

side-effect import（没有 `from` 绑定，只为了触发模块副作用）是 ES Module 里注册全局 adapter 的标准手法。但它的风险是：tree-shaking 或某些 bundler 配置可能把它优化掉。把它放在 `common.ts` 顶部、紧挨着 `listAdapters` 的 import，加上数量断言，就形成了一个闭环——如果 import 被优化掉了，`listAdapters()` 返回 0，断言立刻炸。

**错误信息里直接给出排查路径。** 注意错误信息里的"请检查 registerAdapters.ts 是否被 import（应在 startup/common.ts 顶部 side-effect import）"。这条信息本身就是一次"故障定位指南"——开发者看到这个错误，不需要去翻代码找原因，直接被告知去检查哪一行。这是 fail-fast 错误信息的写法：**不只是报错，还要告诉排查者第一步该看哪里。**

### 断言四：时区一致性（warn，不阻断）

不是所有断言都要"炸"。时区一致性校验（行 224-239）就只告警不阻断：

```typescript
// src/startup/common.ts:224
async function warnIfTimezoneInconsistent(): Promise<void> {
  try {
    const rows = await prisma.$queryRaw`SELECT EXTRACT(timezone FROM now())::int AS off_secs`;
    const pgOffsetSeconds = rows[0]?.off_secs;
    if (pgOffsetSeconds !== 0) {
      logger.warn(
        `[Startup] PG session 时区非 UTC（偏移 ${pgOffsetSeconds / 3600}h）。` +
          '因 Prisma #28629，session 非 UTC 会导致 Date 写入 timestamptz 时偏移；' +
          '请检查 DATABASE_URL 的 options=-c TimeZone=UTC 是否生效（pgbouncer transaction 模式可能丢弃 options）。',
      );
    }
  } catch (err) {
    logger.debug?.(`[Startup] 时区一致性校验跳过：${getErrorMsg(err)}`);
  }
}
```

时区偏移 8 小时是"数据会错但不立刻崩"的问题（详见第 9 篇），所以用 warn 而不是 throw。注释里明确写了"仅告警，绝不阻断启动"——因为时区问题可能发生在生产环境，如果阻断启动，服务直接挂了，比"时间偏移"严重得多。这里的设计取舍是：**致命错误 fail-fast，数据正确性问题告警，两者不能用同一把尺子。**

这套断言组合起来，把启动序列从"靠开发者记住顺序"变成了"顺序错了立刻死"。三类"启动即崩"的坑，都被堵在了进程监听端口之前。

---

## 教训

**第一，启动期是系统最脆弱的瞬间，要在这里投入远超直觉的防御。** 很多人对运行时代码写满 try/catch，但对启动序列却很随意——"反正就初始化一下"。但启动序列出错，往往意味着整个进程不可用，影响面比运行时单点 bug 大得多。我们在启动序列里塞的断言（assertCoreDiRegistrations、assertAgentBusinessPortsWired、adapter 数量断言、时区校验、PromptRegistry 就绪检查），比任何一段等长运行时代码都多。

**第二，顺序依赖必须显式化，能断言就断言、能注释就注释。** DI + Port 注入的顺序依赖是隐式的，编译器查不出来。对付隐式依赖，只有两个办法：要么用类型系统/框架把它变成显式的（比如用 IoC 容器的依赖声明），要么用断言在启动时把它查出来。WinMatrix 选了后者——我们没用重型 IoC 框架，而是靠"固定顺序 + fail-fast 断言"。代价是顺序错了会炸，收益是启动极快、依赖图清晰可读。这是个有意识的取舍。

**第三，fail-fast 的错误信息要自带"排查指南"。** 看 adapter 那条断言的错误信息：它不只是说"adapter 数量不对"，还告诉你"请检查 registerAdapters.ts 是否被 import（应在 startup/common.ts 顶部 side-effect import）"。这不是啰嗦，这是把"排查经验"固化进错误信息。一个半夜被叫起来看启动失败的同学，看到这条信息能省 20 分钟。所有 fail-fast 断言的错误信息都应该这样写：**报什么错 + 为什么会错 + 第一步看哪里**。

**第四，断言要区分"致命"和"告警"，但标准要清晰。** adapter 数量不对是致命（功能缺失），throw；时区偏移是告警（数据会偏但不崩），warn。判断标准是"不阻断启动是否比启动失败更糟"。如果启动失败会让服务完全不可用，而问题本身能容忍（如时区偏移），就告警；如果问题会让系统"带病运行"（如缺 adapter、缺 Port），就 throw，宁可启动失败也不要带病上线。

**第五，side-effect import 是脆弱的，要给它配"哨兵"。** ES Module 的 side-effect import（`import './foo.js'`）是注册全局副作用的常用手法，但它不被类型系统追踪，容易被 tree-shaking、重构、bundler 配置悄悄破坏。对付它的办法是配一个"哨兵断言"——import 之后立刻检查副作用是否生效（这里就是 `listAdapters().length`）。哨兵在，副作用丢了立刻发现；哨兵不在，你可能要等运行时某次消息发不出去才知道。

**第六，启动序列的每一步都要有超时。** 看 configManager 的初始化（行 311-318）：

```typescript
const CM_TIMEOUT_MS = readEnvNumber('CONFIG_MANAGER_INIT_TIMEOUT_MS', 60_000);
await Promise.race([
  configManager.initialize(),
  new Promise<never>((_, reject) => {
    const t = setTimeout(() => reject(new Error(`ConfigManager.initialize 超时 (${CM_TIMEOUT_MS}ms)`)), CM_TIMEOUT_MS);
    t.unref();
  }),
]);
```

启动卡死比启动崩溃更难查——进程不退、不报错、不服务，健康检查一直不通过。给每一步加超时（ConfigDbListener 15s、ConfigManager 60s），让"卡死"也变成"fail-fast 报错"。`t.unref()` 保证超时定时器本身不会阻止进程退出。这是 Node 启动序列的细节功夫。

最后回到那个宏观的判断：**启动期断言不是"多余的防御代码"，它是把隐式顺序依赖显式化的唯一手段。** 没有这些断言，你的 DI 系统就是一个"运行时才验证正确性"的黑盒——每次重启都像在赌运气。有了它们，启动成功就意味着"所有核心服务就位、所有 Port 注入完整、所有 side-effect 生效"，你可以在运行时放心地假设基础设施是好的。这份确定性，值回所有写断言的代码量。

---

> **上一篇**：[《分层 import 门禁：check:agent-layers:strict 怎么守住六层》](./66-layer-import-gate.md)
>
> **下一篇**：[《cron 尖峰迁移：为什么 09:00 的任务要把它们"散开"》](./68-cron-spread-migration.md)——讲所有人 cron 都设在整点导致 DB 尖峰，怎么用 `ScheduledCronMigrationLog` 幂等地把 cron 打散。
