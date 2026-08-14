# 第 27 章 测试策略

> "测试不是成本，而是信心的来源。"

一个生产级 AI 平台的测试策略，远比"写几个 unit test"复杂。WinMatrix 要测的不仅是业务逻辑，还有决策引擎的非确定性输出、LLM 调用的边界、多进程协作的时序、数据库迁移的正确性。它的测试体系分成四个 project（unit/integration/migration-exec/e2e），每个 project 有自己的超时、隔离策略、mock 边界。更难得的是，它把**真实生产事故的回放数据**做成了 fixture——让 bug 成为回归测试，确保同类问题不再重现。

本章从 vitest 的四 project 配置讲起，依次展开 E2E 与单元测试的 mock 边界、共享单例 app 模式、自动建库机制、专项回归开关，最后看那些真实事故回放 fixture。

## 27.1 四个 vitest project：分层与分级

`vitest.config.ts` 定义了四个 project，每个 project 对应一类测试，有独立的超时、隔离、并发配置。

```ts
// winmatrix-server/vitest.config.ts（节选）
export default defineConfig({
  resolve: {
    alias: [
      { find: /^@\/(.*)$/, replacement: path.resolve(__dirname, './src/$1') },
      { find: /^@tests\/(.*)$/, replacement: path.resolve(__dirname, './tests/$1') },
    ],
  },
  optimizeDeps: { exclude: ['pg'] },    // 排除 pg，避免预构建问题
  test: {
    environment: 'node',
    globals: true,
    projects: [
      // unit / integration / migration-exec / e2e
    ],
  },
});
```

### unit project

```ts
// vitest.config.ts（第 44-60 行）
{
  test: {
    name: 'unit',
    include: ['tests/unit/**/*.test.ts', 'tests/characterization/**/*.test.ts'],
    setupFiles: ['./tests/unit/setup.ts'],
    env: {
      DATABASE_URL: 'postgresql://user:pass@localhost:5432/winmatrix_unit_test',
      SANDBOX_API_AUTH_TOKEN: 'unit-test-sandbox-control-token',
    },
    isolate: true,        // 测试隔离
    threads: true,        // 多线程并行
    testTimeout: 10000,   // 10s
    hookTimeout: 10000,
  },
}
```

unit project 的关键设计：

- **`DATABASE_URL` 是占位串**：`postgresql://user:pass@localhost:5432/winmatrix_unit_test`。这个 URL 指向一个根本不存在的数据库。**unit 测试不连真数据库**——所有 DB 访问都被 mock 掉。这个占位串只是为了让代码里的 `process.env.DATABASE_URL` 不为空，避免启动校验报错。
- **`SANDBOX_API_AUTH_TOKEN` 也是占位**：同理，让代码读到一个值但不实际调用 sandbox。
- **`isolate: true` + `threads: true`**：每个测试文件在独立线程、独立 isolate 里跑，完全并行，互不干扰。
- **10s 超时**：单元测试必须快。超过 10s 说明要么是测试写得有问题（等了不该等的 I/O），要么是被测代码太慢。

### integration project

```ts
{
  test: {
    name: 'integration',
    include: ['tests/integration/**/*.test.ts'],
    setupFiles: ['./tests/shared/setup.ts', './tests/integration/setup.ts'],
    threads: false,       // 串行
    testTimeout: 30000,   // 30s
    hookTimeout: 60000,   // 60s
  },
}
```

integration 测试**串行执行**（`threads: false`），因为它们共享真实数据库，并行会数据竞争。30s 超时 + 60s hook 超时——集成测试要连真实 PG/ES，启动和清理都慢。

### migration-exec project

这是第四个 project（unit/integration/migration-exec/e2e），专门验证**数据库迁移的可执行性**。每个迁移文件都会在这里被执行一次，确保迁移脚本不会因为语法错误、依赖缺失而在生产部署时炸掉。

### e2e project

```ts
// vitest.config.ts（第 89-100 行）
{
  test: {
    name: 'e2e',
    include: ['tests/e2e/**/*.{test,spec}.ts'],
    setupFiles: ['./tests/shared/setup.ts', './tests/e2e/setup.ts'],
    threads: false,       // 串行
    testTimeout: 60000,   // 60s
    hookTimeout: 300000,  // 5 分钟
    retry: 1,             // 重试 1 次
  },
}
```

e2e 是最重的一级：

- **`threads: false` 串行**：E2E 测试启动完整应用，占用端口，绝对不能并行。
- **60s 测试超时 + 300s（5 分钟）hook 超时**：E2E 的 `beforeAll` 要启动整个应用（连 DB、连 Redis、初始化所有服务），这个 hook 可能要几分钟。300s 给了足够余量。
- **`retry: 1`**：E2E 受环境影响大（网络抖动、服务未就绪），允许重试 1 次。但注意——retry 只重试失败的 test，不重试 hook。如果 `beforeAll` 失败，整个 describe block 直接挂。

### 四 project 对比

| project | 线程 | 测试超时 | hook 超时 | retry | DB | 场景 |
|---------|------|---------|----------|-------|-----|------|
| **unit** | 并行 | 10s | 10s | 0 | mock（占位串） | 纯逻辑 |
| **integration** | 串行 | 30s | 60s | 0 | 真实 PG | 跨模块链路 |
| **migration-exec** | 串行 | - | - | 0 | 真实 PG | 迁移可执行性 |
| **e2e** | 串行 | 60s | 300s | 1 | 真实全栈 | API 端到端 |

**分级超时是这套配置的精髓**——不是所有测试都该用同一个超时。快测试用短超时（快速发现慢测试），慢测试用长超时（避免误杀）。

## 27.1.1 migration-exec project：迁移可执行性验证

第四个 project `migration-exec` 容易被忽视，但它解决一个真实且严重的问题：**数据库迁移脚本本身可能有问题。**

Prisma 迁移是 SQL 文件（或 prisma db push 的自动生成 SQL）。这些 SQL 在开发时跑过，但生产环境的数据库状态可能和开发环境不同——开发环境可能是空库，生产环境有大量历史数据。一个迁移在空库上能跑，在有数据的库上可能失败（比如加 NOT NULL 列但没设 DEFAULT，已有行无法填充）。

migration-exec project 做的事是：**对每个迁移文件，在一个干净的测试库上执行，验证它能成功跑完。** 这不是验证迁移的"业务正确性"（那是 integration/e2e 的事），而是验证"可执行性"——SQL 语法对不对、依赖的列/表存不存在、约束冲突不冲突。

这种测试的价值在 CI 里——每次 PR 如果改了 schema.prisma 或迁移文件，migration-exec 会验证新迁移能跑通。**一个不能执行的迁移在生产部署时才暴露，是运维灾难。** migration-exec 把这个风险提前到 CI。

### coverage 排除策略

```ts
coverage: {
  provider: 'v8',
  reporter: ['text', 'json', 'html', 'lcov'],
  exclude: [
    'node_modules/', 'dist/', 'coverage/', 'tests/',
    '**/*.test.ts', '**/*.spec.ts', '**/*.d.ts',
    'scripts/', 'src/agents/config/',
  ],
}
```

覆盖率统计的 exclude 列表揭示了"哪些代码不需要被测试覆盖"：

- **`tests/`**：测试代码本身不需要被测。
- **`**/*.test.ts` / `**/*.spec.ts`**：测试文件。
- **`**/*.d.ts`**：类型声明文件，无运行时逻辑。
- **`scripts/`**：构建脚本、运维脚本，不是业务代码。
- **`src/agents/config/`**：Agent 配置（可能是声明式配置，逻辑少）。

**排除这些目录是为了让覆盖率数字反映"业务代码的测试覆盖"，而不是被非业务代码稀释。** 如果不排除 `scripts/`，覆盖率会被那些从不被 import 的构建脚本拉低，失去参考价值。

## 27.2 E2E vs 单元测试的 mock 边界

这是测试策略里最重要的设计决策：**什么该 mock，什么不该 mock。**

### 单元测试：全 mock

```ts
// unit project
env: {
  DATABASE_URL: 'postgresql://user:pass@localhost:5432/winmatrix_unit_test',
  SANDBOX_API_AUTH_TOKEN: 'unit-test-sandbox-control-token',
}
```

单元测试**不加载 `.env.test`**。所有外部依赖（DB、Redis、LLM、sandbox）都被 mock。`DATABASE_URL` 只是一个占位串，让代码里的环境变量校验通过。

这带来的好处是：**单元测试可以在任何环境（CI、开发机、离线）快速运行**，不需要真实的基础设施。

### E2E 测试：全不 mock

```ts
// tests/e2e/setup.ts（第 6-35 行）
process.env.NODE_ENV = 'test';
const required = ['DATABASE_URL', 'REDIS_URL'] as const;
const missing = required.filter(
  (key) => !process.env[key] || process.env[key]!.trim() === '',
);
if (missing.length > 0) {
  console.error('[E2E] 缺少必填配置:', missing.join(', '));
  process.exit(1);
}
// 还校验 LLM Provider 配置
```

E2E 测试**加载 `.env.test`**，并且**不 mock DB/Redis/LLM**。它要求所有必填配置都存在——`DATABASE_URL`、`REDIS_URL` 缺失直接 `process.exit(1)`，还会校验 LLM Provider 配置。

为什么 E2E 不 mock？因为 E2E 的目的是**验证真实链路**——从 HTTP 请求到 DB 写入到 LLM 调用到响应返回的完整流程。如果 mock 掉任何一环，就失去了 E2E 的意义。

**缺失必填项直接 `process.exit(1)` 而不是跳过测试**，这是一个重要决策。它保证了 E2E 测试要么真正跑起来（配置齐全），要么明确失败（配置缺失），不会出现"静默跳过一半测试"这种假绿。

### 边界总结

```
        mock 一切                  不 mock 任何东西
        （快、隔离）               （慢、真实）
            │                          │
            ▼                          ▼
       unit 测试                    E2E 测试
   DATABASE_URL=占位串        DATABASE_URL=真实 PG
   不连任何服务                连真实 PG/Redis/LLM
   10s 超时                   60s 超时 + retry
```

**清晰的边界比"中间地带"更有价值。** 最糟糕的做法是"半 mock"——mock 了 DB 但没 mock Redis，结果测试既不能快速跑，又不能验证真实链路，两头不讨好。

## 27.2.1 SANDBOX_API_AUTH_TOKEN 占位的意义

回顾 unit project 的 env 配置：

```ts
env: {
  DATABASE_URL: 'postgresql://user:pass@localhost:5432/winmatrix_unit_test',
  SANDBOX_API_AUTH_TOKEN: 'unit-test-sandbox-control-token',
}
```

除了 `DATABASE_URL`，还有 `SANDBOX_API_AUTH_TOKEN`。为什么要占位这个？

因为代码里有一处启动校验——如果 `SANDBOX_API_AUTH_TOKEN` 为空，sandbox 相关功能会直接禁用或报错。单元测试里，某些被测代码路径会间接读这个环境变量。如果不占位，测试可能因为"token 为空"而走了和正常情况不同的分支，测试结果失去意义。

**占位串的哲学是"让环境看起来正常，但不实际连接"。** 单元测试需要的是"代码能跑到被测逻辑"，而不是"代码能连到真实服务"。占位串消除了"环境变量缺失导致的异常分支"，让测试聚焦于业务逻辑。

### unit setup 的职责

```ts
setupFiles: ['./tests/unit/setup.ts'],
```

`tests/unit/setup.ts` 做的事很轻——可能只是设置一些全局 mock（如 mock 掉 logger、mock 掉某些 always-on 的初始化）。它不做重活（不连 DB、不启动服务），因为 unit 测试要快。

对比 `tests/shared/setup.ts`（integration/e2e 共用），它做的事更重——可能包含真实 DB 连接初始化、Redis 连接等。setup 文件的"轻重"直接对应了 project 的"重量"。

## 27.3 E2E 共享单例 app

E2E 测试启动一个完整的应用实例，这个实例**在所有测试之间共享**：

```ts
// tests/e2e/api/p0-smoke.test.ts（第 11-58 行）
describe('P0 API smoke', () => {
  beforeAll(async () => {
    const app = await getTestApp();    // 共享单例
    baseUrl = app.baseUrl;
    apiClient = new ApiClient(baseUrl);
    userCtx = await registerAndLogin(baseUrl);
    adminCtx = await registerAndLogin(baseUrl, { isAdmin: true });
  });
  afterAll(async () => { await closeTestApp(); });
  it('有效 input 返回 200', async () => {
    const res = await apiClient.anonymous().post('/api/v1/agents/route',
      { body: { input: '你好', userId: userCtx.userId } });
    expect([200, 404, 503].includes(res.status)).toBe(true);
  });
});
```

`getTestApp()` 返回一个全局单例——第一次调用时启动应用，后续调用复用。`closeTestApp()` 在所有测试结束后关闭。

**为什么共享单例？** 因为启动一个完整应用（包括 `prisma db push` 建表、初始化所有服务、连接 Redis/ES）要几十秒甚至几分钟。如果每个测试文件都启动一次，E2E 套件会慢到无法接受。共享单例让启动成本分摊到所有测试上。

### 断言的灵活性

注意这个断言：

```ts
expect([200, 404, 503].includes(res.status)).toBe(true);
```

它接受 200（成功）、404（路由未配置）、503（服务降级）三种状态。这是 E2E 测试的务实——在某些测试环境里，特定功能可能没配置（404）或被降级（503），这不算测试失败。**E2E 测试验证"系统没崩溃"，而不是"每个功能都完美工作"。**

## 27.4 自动建库

E2E 测试需要一个数据库，但手动建库很麻烦。`testApp.ts` 实现了自动建库：

```ts
// tests/e2e/testApp.ts（第 33-60 行）
async function ensureTestDatabase(): Promise<void> {
  const parsed = new URL(process.env.DATABASE_URL);
  const dbName = (parsed.pathname || '/').slice(1).replace(/\/$/, '') || 'winmatrix_test';
  parsed.pathname = '/postgres';    // 连 postgres 系统库
  const client = new Client({ connectionString: parsed.toString() });
  await client.connect();
  const res = await client.query(
    'SELECT 1 FROM pg_database WHERE datname = $1', [dbName],
  );
  if (res.rows.length === 0) {
    await client.query(`CREATE DATABASE "${dbName.replace(/"/g, '""')}"`);
  }
  await client.end();
}
```

建库逻辑：

1. 从 `DATABASE_URL` 解析出数据库名。
2. 连接到 `postgres` 系统库（这个库一定存在）。
3. 查 `pg_database` 看目标库存不存在。
4. 不存在就 `CREATE DATABASE`。
5. 之后 `prisma db push` 建表。

注意 `"${dbName.replace(/"/g, '""')}"`——双引号转义。这是防止数据库名里有特殊字符导致 SQL 注入。虽然测试环境的数据库名通常是固定的，但这个转义是防御性编程的好习惯。

**自动建库让 E2E 测试"开箱即用"**——只要有一个可访问的 PG 实例和正确的 `DATABASE_URL`，测试就能自己准备好数据库，无需手动操作。

## 27.5 测试目录结构

```
tests/
├── unit/                 # 单元测试，按被测源码层归类
│   ├── agents/           # 按 agents 层结构组织
│   │   ├── action/
│   │   ├── application/
│   │   ├── architecture/
│   │   └── context/      # 最大（60+ 文件）
│   └── setup.ts
├── integration/          # 跨模块链路
│   ├── es-degradation/   # ES 降级
│   ├── es-knn-search/    # ES kNN 搜索
│   ├── es-pg-sync/       # ES-PG 同步
│   ├── observability/    # 可观测性
│   ├── startup/          # 启动流程
│   └── ...
├── e2e/
│   ├── api/              # API E2E（24 个测试文件）
│   ├── agent-decision-pipeline/
│   │   └── fixtures/expected/WF-*.json   # 40+ 决策管线快照
│   └── ...
├── shared/               # 共享 setup / fixtures / helpers / factories
├── fixtures/             # 测试数据与回放样例
│   ├── incident-2026-05-26-job-84711/    # 真实事故回放
│   ├── incident-2026-05-26-job-84712/    # 真实事故回放
│   └── decision-planner-91695-replay/    # 决策规划器回放
├── manual/               # 人工验证、专项诊断脚本
└── characterization/     # 特征化测试（记录实际行为）
```

几个值得注意的组织方式：

- **unit 按源码层归类**：`tests/unit/agents/` 对应 `src/agents/`。这让"某个源码文件的测试在哪"一目了然。
- **shared 提供共享设施**：`factories`（测试数据工厂）、`fixtures`（共享数据）、`helpers`（测试辅助函数）。所有 project 都能用。
- **`e2e/api/` 有 24 个测试文件**：API E2E 覆盖了主要业务接口。
- **fixtures 里藏真实事故**：`incident-2026-05-26-job-*` 是真实生产事故的回放数据（见 27.7）。

## 27.5.1 测试目录按源码层归类的设计

为什么 unit 测试目录要按源码层归类（`tests/unit/agents/` 对应 `src/agents/`），而不是按"功能"或"场景"归类？

因为单元测试的**定位是"测某个源码文件"**。当你改了 `src/agents/core/agent/turn/TurnRunner.ts`，你想知道的是"TurnRunner 的测试在哪"。如果按功能归类，你可能要在 `tests/turn-behavior/`、`tests/agent-execution/` 等目录里翻找。按源码层归类，直接去 `tests/unit/agents/core/agent/turn/` 就行——目录结构和源码一一对应。

这种"镜像结构"还有一个好处：**可以机械地检查"每个源码文件有没有对应的测试"**。写一个脚本扫 `src/` 和 `tests/unit/`，比对文件名，就能发现缺少测试的源码文件。

### shared 目录的 factories

```
tests/shared/
├── setup.ts          # 共享 setup
├── fixtures/         # 共享测试数据
├── helpers/          # 测试辅助函数
└── factories/        # 测试数据工厂
```

`factories` 是测试数据工厂——用工厂模式生成测试数据，而不是在每个测试里手写。比如：

```ts
// 概念示意
const conversation = ConversationFactory.build({
  userId: 'test-user',
  projectId: 'test-project',
});
```

工厂模式的好处是：

1. **默认值集中管理**：所有必填字段有合理默认值，测试只需指定"我关心的字段"。
2. **变体方便**：`ConversationFactory.build({ status: 'failed' })` 快速生成失败状态的会话。
3. **复用**：多个测试用同一个工厂，避免重复定义测试数据。

**factories 是大型测试体系的标配。** 没有它，测试数据散落在各处，改一个字段定义要改几十个测试文件。

## 27.6 决策管线回归快照

决策引擎（第 10 章）的输出是**非确定性**的——LLM 的响应每次可能不同。如何对这种系统做回归测试？

WinMatrix 用"黄金快照"方案：

```
tests/e2e/agent-decision-pipeline/fixtures/expected/
├── WF-001.json
├── WF-002.json
├── ... (40+ 快照)
```

每个 `WF-*.json` 记录了一个特定输入下决策引擎的完整输出——路由结果、规划步骤、commitment 等。测试时重新跑这个输入，把输出和快照对比。

但这种对比不是"完全相等"——因为 LLM 输出有非确定性。快照对比通常是**结构化对比**：只比决策结构（路由到哪个 agent、规划了几步），不比 LLM 生成的自由文本。

40+ 个快照覆盖了主要的工作流场景。每次修改决策引擎后跑这些快照，确保不引入回归——如果某个 WF 快照从"路由到 A"变成了"路由到 B"，就是明确的回归信号。

## 27.7 真实事故回放 fixture

这是 WinMatrix 测试体系最独特的部分。`tests/fixtures/` 下有几个以日期和 job ID 命名的目录：

```
tests/fixtures/
├── incident-2026-05-26-job-84711/     # 2026-05-26 的事故，job 84711
├── incident-2026-05-26-job-84712/     # 同日另一 job
└── decision-planner-91695-replay/     # 决策规划器回放
```

这些不是普通的测试数据，而是**真实生产事故的现场回放**。当生产环境出了一个 bug（比如某个 job 卡死、决策规划器给出错误结果），团队会把当时的输入、上下文、预期输出都固化成 fixture，写一个回归测试。

这种做法的价值是：

1. **bug 不再重现**：一旦事故被回放成测试，同类问题以后会在 CI 里被拦住，不会再次进入生产。
2. **事故知识沉淀**：fixture 名字里的日期和 job ID 是"事故档案"——后来者能通过这些 fixture 了解"这个系统曾经出过什么问题"。
3. **真实数据覆盖**：相比人工编造的测试数据，真实事故数据更能暴露边角场景。

**把生产事故变成回归测试，是测试工程成熟度的标志。** 它把"被动的救火"转化成了"主动的防护"。

## 27.7.1 事故回放 fixture 的构建过程

一个真实事故回放 fixture 是怎么诞生的？以 `incident-2026-05-26-job-84711` 为例：

1. **生产告警**：2026-05-26，监控系统告警——某个 job（ID 84711）卡在 running 状态超过预期。
2. **排查**：工程师介入，找到根因（可能是某个边界条件没处理、某个外部服务超时、某个状态机转换遗漏）。
3. **修复**：修复代码。
4. **固化 fixture**：把当时的输入数据、上下文、预期行为提取成 fixture。
5. **写回归测试**：用这个 fixture 写一个测试，验证修复后的代码能正确处理这个场景。

这个过程的第 4-5 步是关键——**修复 bug 不是终点，把 bug 变成回归测试才是终点。** 没有这一步，同样的 bug 可能在未来的重构中复现。

### fixture 命名的信息量

```
incident-2026-05-26-job-84711/
incident-2026-05-26-job-84712/
decision-planner-91695-replay/
```

这些命名包含了丰富的信息：

- **`incident-`** 前缀：这是事故回放，不是普通测试数据。
- **日期**：事故发生时间，可以作为"时间线"的锚点。
- **`job-XXXXX`**：生产 job ID，可以在生产日志里追溯当时的完整上下文。
- **`replay`**：回放数据，用于复现特定行为。

**好的 fixture 命名是"事故档案"**——后来者看到名字就知道"这是什么事故、什么时候发生、涉及哪个 job"，不用翻 commit history。

### decision-planner-91695-replay 的特殊性

这个 fixture 不是"事故"，而是"决策规划器的回放"。它的场景可能是：某个特定的用户输入（ID 91695）在决策规划器里产生了异常的规划结果。不是 crash 级别的事故，但行为不符合预期。

把这种"行为不符合预期"的场景也固化成 fixture，体现了测试的另一个价值——**锁定预期行为**。一旦 fixture 说"这个输入应该路由到 A"，未来任何把它改成"路由到 B"的修改都会触发测试失败，强制修改者解释"为什么要改这个路由"。

## 27.8 特征化测试

`tests/characterization/` 记录系统的**实际行为**而非期望行为：

```
turn-admission-paths.test.ts   # Turn 准入路径的实际行为
turn-admission-map.md          # 准入路径地图（文档）
```

特征化测试（Characterization Test）是 Michael Feathers 在《修改代码的艺术》里提出的概念。面对遗留代码（或复杂逻辑），你不知道它的"正确行为"是什么，但你可以记录它的"当前行为"——这样在重构时，如果行为变了，测试会告诉你。

WinMatrix 的 Turn 准入逻辑（第 8 章）非常复杂，有多条准入路径。特征化测试把这些路径的实际行为固化下来，重构时确保不改变。

## 27.9 专项回归开关

某些测试默认不开，需要显式开关：

```ts
// kickoff 测试默认 describe.skip
describe.skip('Kickoff 流程', () => {
  // 需要 E2E_KICKOFF=1 才跑
});
```

Kickoff 流程（第 18 章）涉及完整的项目初始化，耗时长、依赖多。默认 `describe.skip` 跳过，只在显式设 `E2E_KICKOFF=1` 时才跑。这让常规 E2E 不会被 kickoff 测试拖慢，而需要验证 kickoff 时又能方便地开启。

这种"专项开关"模式适合**重流程、低频验证**的场景——不是每次 CI 都要跑，但不能没有。

## 27.9.1 为什么 kickoff 要单独开关

Kickoff（第 18 章）是项目初始化流程——它涉及创建项目、初始化所有基础设施、启动多个 Agent。这是 WinMatrix 里最重的操作之一。为什么它要单独开关？

**因为 kickoff 测试的副作用太大。** 一个 kickoff 测试会：

- 创建新项目（污染 DB）。
- 初始化知识库（写 ES）。
- 可能触发真实的 LLM 调用（花钱）。
- 耗时几分钟。

如果每次 CI 都跑 kickoff 测试，CI 会非常慢，而且 DB 会被大量测试项目污染。默认 `describe.skip` 让常规 CI 跳过它们，只在需要时（`E2E_KICKOFF=1`）手动触发。

**"默认关闭 + 显式开启"是重流程测试的通用策略。** 它保证了测试存在（需要时能跑），但不拖累常规流程。

### describe.skip 与 conditional describe

```ts
// 方式一：describe.skip（默认跳过）
describe.skip('Kickoff 流程', () => { ... });

// 方式二：conditional describe（按环境变量决定）
const describeKickoff = process.env.E2E_KICKOFF === '1' ? describe : describe.skip;
describeKickoff('Kickoff 流程', () => { ... });
```

两种方式都常见。方式一更简洁，方式二更显式。无论哪种，核心都是"让重测试默认不跑"。

## 27.10 组合测试命令

```json
// package.json scripts（测试相关）
"test:quick": "...",           // 快速测试（unit 子集）
"test:verify": "...",          // 验证（build:tsc + unit + quick）
"test:all": "...",             // 全量
"test:unit": "...",            // 仅单元
"test:e2e:fast": "...",        // 快速 E2E
"test:e2e:nollm": "...",       // 无 LLM 的 E2E
```

几个值得展开的命令：

- **`test:quick`**：快速反馈，只跑最核心的单元测试。开发时频繁用。
- **`test:verify`**：提 PR 前的验证——类型检查 + 单元测试 + 快速测试。组合命令，一次跑完。
- **`test:all`**：全量测试，CI 用。
- **`test:e2e:nollm`**：E2E 但不调真实 LLM（用桩）。既验证了端到端流程，又节省了 LLM 成本。这是一个很实用的命令——E2E 的主要价值是验证链路完整性，而不是验证 LLM 输出质量，所以 mock 掉 LLM 是合理的。

**组合命令的本质是"为不同场景预设不同的测试子集"**——开发时追求快（test:quick），提 PR 时追求覆盖（test:verify），CI 时追求全（test:all），省成本时用 nollm。

## 27.11 手动基准测试

`tests/manual/` 里的脚本不是 Vitest 测试，而是用 `tsx` 运行的分析工具：

```
tests/manual/
├── benchmark-simple-task-runtime.ts      # 简单任务运行时基准
├── benchmark-prompt-visibility-perf.ts   # 提示词可见性性能
├── simulate-complexity-tier-traffic.ts   # 复杂度分层流量模拟
├── simulate-layered-routing.ts           # 分层路由模拟
└── verify-decision-shadow-metadata.ts    # 决策影子元数据验证
```

这些脚本是开发者的"分析工具"——性能基准、流量模拟、影子验证。它们不进 CI，但在性能调优、容量规划时非常有用。**把"手动分析脚本"和"自动化测试"放在一起，让它们共享 fixtures 和 helpers，是一个务实的组织方式。**

## 27.11.1 基准测试与生产容量规划

`tests/manual/` 里的基准脚本不只是"跑跑看性能数字"，它们直接服务于**生产容量规划**。

```
tests/manual/
├── benchmark-simple-task-runtime.ts      # 简单任务运行时基准
├── simulate-complexity-tier-traffic.ts   # 复杂度分层流量模拟
├── simulate-layered-routing.ts           # 分层路由模拟
```

- **`benchmark-simple-task-runtime`**：测一个简单任务（如闲聊）的端到端延迟和吞吐。这个数字直接回答"一台 Pod 能支撑多少 QPS"。
- **`simulate-complexity-tier-traffic`**：模拟不同复杂度层级的流量混合。生产环境不是全简单任务或全复杂任务，而是混合——简单任务占 80%、复杂任务占 20%。模拟这种分布才能测出真实的资源需求。
- **`simulate-layered-routing`**：模拟分层路由（L0/L1/L2/L3/Chat）的命中率。这回答"多少比例的请求能被低层短路、多少要走完整管线"——直接影响 LLM 成本估算。

这些脚本的输出（延迟、吞吐、命中率、LLM 调用次数）是容量规划的输入。"要不要加 Pod"、"要不要升级 LLM plan"、"配额够不够"——这些运维决策都依赖基准数据。**没有基准，容量规划就是猜。**

### 为什么用 tsx 而不是 vitest

这些脚本用 `tsx` 运行，不走 vitest。原因是它们不是"断言式测试"——没有 `expect(x).toBe(y)`。它们是"测量式分析"——跑一个场景，输出数据，由人来解读。

vitest 适合"给定输入，断言输出"的确定性测试。但"模拟 1000 个并发的延迟分布"不是断言——它是测量。强行塞进 vitest 的 describe/it 框架反而别扭。用 tsx 独立运行，输出到 stdout 或文件，更自然。

**"选对工具"包括"知道什么时候不用测试框架"。** 测量和断言是两种不同的活动，混在一起反而降低两者的清晰度。

## 27.12 架构守卫

除了功能测试，CI 还跑架构守卫：

```json
"check:layers": "node scripts/check-layer-imports.cjs",
"check:agent-layers:strict": "node scripts/check-agent-layer-imports.cjs --strict && npm run check:tool-kernel-consumption",
"check:cdw-boundaries": "node scripts/check-cdw-boundaries.mjs",
"check:observability-rules": "...",
"check:time-semantics": "...",
```

这些检查（第 28 章详述）确保代码不违反架构约束：分层依赖规则、Agent 内部子层隔离、时间语义约定（禁 toISOString、禁硬编码时区）、可观测性规则。**架构守卫把"架构原则"从文档变成了可执行的检查**——文档会被忽视，但 CI 里失败的检查不会被忽视。

## 本章小结

本章深入分析了 WinMatrix 的测试策略：

1. **四个 vitest project**：unit（10s，并行，全 mock）/ integration（30s，串行，真实 PG）/ migration-exec（迁移可执行性）/ e2e（60s+300s hook，串行，真实全栈，retry 1），分级超时。
2. **E2E vs 单元 mock 边界**：unit 不加载 `.env.test`，DATABASE_URL 用占位串、全 mock；E2E 加载 `.env.test`、不 mock DB/Redis/LLM、缺失必填项 `process.exit(1)`——清晰边界优于"半 mock"。
3. **E2E 共享单例 app**：`getTestApp()` beforeAll 启动、`closeTestApp()` afterAll 关闭，启动成本分摊到所有测试。
4. **自动建库**：`ensureTestDatabase` 连 postgres 系统库查 `pg_database`，不存在则 `CREATE DATABASE`，再 `prisma db push` 建表，开箱即用。
5. **目录结构**：unit 按源码层归类、shared 共享 facilities、e2e/api 24 文件、characterization 特征化、manual 基准脚本。
6. **决策管线回归快照**：40+ WF-*.json 黄金快照，结构化对比决策输出，防 LLM 非确定性回归。
7. **真实事故回放 fixture**：`incident-2026-05-26-job-84711/84712`、`decision-planner-91695-replay` 把生产 bug 变成回归测试。
8. **专项回归开关**：kickoff 默认 `describe.skip`，需 `E2E_KICKOFF=1` 开启；组合命令 test:quick/verify/all/e2e:nollm。
9. **架构守卫**：check:layers + check:agent-layers:strict + check:time-semantics + check:observability-rules，架构原则变成 CI 可执行检查。

在下一章中，我们将从这些工程实践里提炼出可复用的设计模式——看分层架构、Port 注入、DI 容器、DomainResult 等模式如何贯穿全书。
