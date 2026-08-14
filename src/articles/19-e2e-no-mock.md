# 测试策略：为什么我们的 E2E 坚决不 mock 数据库

> 这是《WinMatrix 开发经验文集》第 19 篇。讲一个在 AI 平台里争议很大的问题：E2E 测试到底该不该 mock 数据库、Redis、甚至 LLM？很多人为了"快"把 E2E 也 mock 了，我们选择了相反的路——坚决不 mock。这篇讲为什么，以及怎么把不 mock 的 E2E 跑得稳。代码来自 WinMatrix 后端真实实现。

如果你做过一段时间后端测试，大概率见过这样的 E2E 测试：

```typescript
// 一个"看起来是 E2E，其实全是 mock"的测试
beforeEach(() => {
  vi.mock('@/infrastructure/persistence/prisma/client');
  vi.mock('@/infrastructure/cache/redisCache');
  vi.mock('@/infrastructure/llm/...');
});
```

每个测试自己 mock 一堆依赖，跑起来飞快，CI 绿油油的。大家都很开心——直到上线那天，一个"测试都过了"的功能在生产环境炸了。

为什么会这样？因为**你 mock 掉的那些东西，恰恰是生产环境里最容易出问题的部分**。Prisma 的连接池行为、Redis 的序列化、PG 的时区、约束、索引——这些都不是你 mock 一个返回值就能模拟的。你的 E2E 测的是"我的业务逻辑在理想世界里对不对"，而不是"我的系统在真实环境里能不能跑"。

这篇文章讲 WinMatrix 的测试策略，核心一句话：**E2E 坚决不 mock 基础设施，缺必填配置直接退出，宁可跑不起来也不跑假的**。

---

## 四个测试 project，各有各的 mock 边界

先看整体结构。WinMatrix 的测试分四个 vitest project，每个 project 的 mock 策略和 timeout 都不同：

```typescript
// vitest.config.ts（第 43-101 行，四个 project）
projects: [
  {
    test: {
      name: 'unit',
      include: ['tests/unit/**/*.test.ts', 'tests/characterization/**/*.test.ts'],
      setupFiles: ['./tests/unit/setup.ts'],
      env: {
        DATABASE_URL: 'postgresql://user:pass@localhost:5432/winmatrix_unit_test',  // 占位串
        SANDBOX_API_AUTH_TOKEN: 'unit-test-sandbox-control-token',
      },
      isolate: true, threads: true,
      testTimeout: 10000, hookTimeout: 10000,
    },
  },
  {
    test: {
      name: 'integration',
      setupFiles: ['./tests/shared/setup.ts', './tests/integration/setup.ts'],
      threads: false, testTimeout: 30000, hookTimeout: 60000,
    },
  },
  {
    test: {
      name: 'migration-exec',  // 专用库执行级回归
      threads: false, testTimeout: 30000, hookTimeout: 60000,
    },
  },
  {
    test: {
      name: 'e2e',
      include: ['tests/e2e/**/*.{test,spec}.ts'],
      setupFiles: ['./tests/shared/setup.ts', './tests/e2e/setup.ts'],
      threads: false,    // 串行
      testTimeout: 60000, hookTimeout: 300000, retry: 1,
    },
  },
]
```

这张表里藏着测试策略的精髓，我们逐行拆。

### unit：占位串 + 秒级 timeout

unit project 的 `DATABASE_URL` 是 `postgresql://user:pass@localhost:5432/winmatrix_unit_test`——一个**根本连不上的占位串**。为什么？因为单元测试**本来就不该碰数据库**。如果你在单元测试里真的连了数据库，那它就不是单元测试，是集成测试，应该挪到 integration project。

占位串的意义是：让需要这个环境变量的代码能 import（不至于 undefined 报错），但任何真的试图连数据库的操作都会立刻失败——这是一种"防御性占位"。单元测试测的是纯逻辑（函数、类的行为），依赖应该全用 mock/stub，但 mock 的是**具体的 collaborator**（某个 service、某个 repository 的返回值），而不是基础设施本身。

timeout 10 秒也说明问题——单元测试应该秒级完成，超过 10 秒基本就是在做 I/O，该挪窝了。

### integration：30s timeout，加载 .env.test

integration project 的 `setupFiles` 加载 `tests/shared/setup.ts`，这个文件会读 `.env.test`，注入真实的 `DATABASE_URL`。也就是说，**集成测试连的是真实数据库**（测试专用库），不 mock。timeout 30 秒、hook 60 秒，给数据库操作留足时间。

### e2e：60s timeout、串行、retry 1

e2e 是最重的。timeout 60 秒，hook（beforeAll/afterAll）300 秒——因为 e2e 要启动完整应用、建库建表、跑真实的 HTTP 请求和 WebSocket。`threads: false` 意味着**串行执行**，不并行——多个 e2e 同时跑会互相踩到（共享数据库、端口冲突）。`retry: 1` 给一次重试机会，因为 e2e 涉及网络和数据库，偶发的 flaky 难免。

**教训：测试分层不是"按目录分"，而是"按 mock 边界和 I/O 成本分"。** 每一层有明确的契约——unit 不碰 I/O、integration 连真实 DB 但不启动 HTTP、e2e 全真。混淆边界（比如 unit 里连数据库、e2e 里全 mock）是测试腐化的开始。

---

## E2E 不 mock 的第一道闸门：缺配置直接退出

上面说了 e2e 不 mock 基础设施。但这带来一个问题：**不 mock 就意味着必须有真实的基础设施可用**。如果开发者的机器上没起 PostgreSQL、没起 Redis、没配 LLM key，e2e 怎么办？

很多人的解法是"找不到就 mock 一个"——这恰恰是我们最反对的。WinMatrix 的解法是：**缺必填配置，直接 `process.exit(1)`，绝不偷偷 mock**：

```typescript
// tests/e2e/setup.ts（第 6-18 行）
process.env.NODE_ENV = 'test';

const required = ['DATABASE_URL', 'REDIS_URL'] as const;
const missing = required.filter((key) => !process.env[key] || process.env[key]!.trim() === '');
if (missing.length > 0) {
  console.error('[E2E] 缺少必填配置:', missing.join(', '));
  console.error('[E2E] 请在 .env 或 .env.test 中配置以上变量。');
  process.exit(1);
}
```

这段代码的意思是：e2e 启动前先检查 `DATABASE_URL` 和 `REDIS_URL` 是不是真的配了。没配？直接退出，报错告诉你"去配"。**不会**fallback 到一个 mock 的内存数据库，不会悄悄跳过那些需要基础设施的测试。

为什么这么"轴"？因为**偷偷 mock 是测试里最危险的谎言**。假设你的 CI 环境某天 Redis 挂了，e2e setup 发现 REDIS_URL 连不上，于是"好心"地 fallback 到一个 mock——CI 绿了，大家以为没事，结果生产环境的 Redis 行为和你的 mock 不一样，上线炸了。这种"绿色谎言"比"红色失败"可怕一百倍。

补一句：对 LLM provider 也是同样处理。`WIN_LLM_PROVIDER=openai` 时检查 `WIN_OPENAI_API_KEY` 和 `WIN_OPENAI_MODEL_NAME`，缺了直接退出。不 mock LLM，不假装调用了。

**教训：测试基础设施缺失时，fail 比假装通过好。** 宁可测试跑不起来（你会去修），也别让测试在"假环境"里绿着（你会以为没事）。`process.exit(1)` 是诚实，mock fallback 是自欺。

---

## E2E 自动建库：不是 mock，是真的建一个测试库

不 mock 数据库，那测试库怎么来？手动建？太容易忘。WinMatrix 的做法是**测试启动时自动建库建表**：

```typescript
// tests/e2e/testApp.ts（ensureTestDatabase）
async function ensureTestDatabase(): Promise<void> {
  const parsed = new URL(process.env.DATABASE_URL);
  const dbName = (parsed.pathname || '/').slice(1).replace(/\/$/, '') || 'winmatrix_test';
  parsed.pathname = '/postgres';   // 连到默认 postgres 库
  const client = new Client({ connectionString: parsed.toString() });
  await client.connect();
  const res = await client.query('SELECT 1 FROM pg_database WHERE datname = $1', [dbName]);
  if (res.rows.length === 0) {
    await client.query(`CREATE DATABASE "${dbName.replace(/"/g, '""')}"`);
  }
  await client.end();
}
```

逻辑很直白：连到 PostgreSQL 的默认 `postgres` 库，查 `DATABASE_URL` 指向的那个库存不存在，不存在就 `CREATE DATABASE`。然后跑 `prisma db push` 把表结构建好。

几个细节：

1. **连 `postgres` 库而不是目标库**。因为你要 `CREATE DATABASE`，必须连一个已存在的库，`postgres` 是 PostgreSQL 安装时就有的默认库，永远在。
2. **库名做了引号转义**（`replace(/"/g, '""')`）。防止库名里有特殊字符导致 SQL 注入——虽然测试库名通常是固定的，但这种防御习惯值得保持。
3. **建完库之后 `prisma db push`**，把 schema.prisma 里的表结构同步到这个空库。这样每个测试套件跑完都是一张干净的新表。

这套机制保证了：**只要你的 PostgreSQL 是通的，e2e 就能自己准备好测试库，不需要手动干预**。这是"不 mock"能落地的前提——如果每次跑 e2e 都要手动建库清表，没人会愿意跑。

---

## E2E 共享单例 app：启动一次，跑完所有用例

启动一个完整应用是很重的（加载配置、连数据库、注册路由、起 worker），不能每个测试都起一遍。WinMatrix 的做法是**共享单例 app**：

```typescript
// tests/e2e/api/p0-smoke.test.ts（典型用例）
describe('P0 API smoke', () => {
  beforeAll(async () => {
    const app = await getTestApp();      // 共享单例：第一次调用启动，后续复用
    baseUrl = app.baseUrl;
    apiClient = new ApiClient(baseUrl);
    userCtx = await registerAndLogin(baseUrl);
    adminCtx = await registerAndLogin(baseUrl, { isAdmin: true });
  });
  afterAll(async () => { await closeTestApp(); });   // 全部跑完才关
  it('有效 input 返回 200 与 unifiedDecision.executionPlan schema', async () => {
    const res = await apiClient.anonymous().post('/api/v1/agents/route', {
      body: { input: '你好', userId: userCtx.userId },
    });
    expect([200, 404, 503].includes(res.status)).toBe(true);
  });
});
```

`getTestApp()` 是个共享单例——第一个测试文件调用时启动应用，后续所有测试文件复用同一个实例。全部跑完，`closeTestApp()` 统一关闭。

注意这个测试的断言：`expect([200, 404, 503].includes(res.status)).toBe(true)`。它接受 404 和 503——为什么？因为这是真实的端到端请求，不是 mock。某些情况下（比如 LLM provider 没配、某项服务降级），接口会返回 503，这也是合法的"系统正常响应"。E2E 测的是"请求能走完整条链路并返回一个合理的响应"，不是"必须返回 200"。这种对真实行为的容忍，正是不 mock 的体现——mock 会让你只断言你"期望"的路径，忽略系统真实会产生的所有合法路径。

**教训：E2E 测试要测"系统的真实行为"，包括它的降级和错误响应，而不只是"happy path"。** 断言写得太严，会掩盖系统真实的复杂度；写得太松，又没意义。合理的度是"断言这条链路产生了一个它确实会产生的结果"。

---

## 真实事故回放：测试 fixture 的最高级用法

WinMatrix 的 `tests/fixtures/` 里有一批特殊的目录，不是普通的测试数据，而是**真实生产事故的回放样例**：

```
tests/fixtures/
├── incident-2026-05-26-job-84711/       # 真实事故 1
├── incident-2026-05-26-job-84712/       # 真实事故 2
├── decision-planner-91695-replay/       # 决策引擎事故回放
├── decision-replay/
├── sample-conversations/
```

这些 fixture 是怎么来的？生产环境出了事故（比如某个任务跑飞了、某个决策路径走错了），我们把当时的输入、上下文、中间状态完整保存下来，变成一个可回放的 fixture。然后写一个测试，加载这个 fixture，重放当时的场景，断言"现在不会再出这个问题了"。

这是测试的最高级用法——**用真实事故驱动测试**。它比任何"我自己编的测试用例"都更有说服力，因为它来自真实用户的真实失败。而且它有双重价值：

1. **防止回归**。这个事故修了，但以后改代码会不会又引入同样的问题？有 fixture 在，CI 会一直帮你盯着。
2. **事故知识沉淀**。半年后新人想知道"为什么这段代码要这么写"，对应的 fixture 就是一份活文档——它告诉你"曾经出过这种事"。

**教训：把生产事故变成测试 fixture，是测试投资里 ROI 最高的一件事。** 你已经为这个事故付过学费了（排查时间、用户影响、加班），不把它的"复现条件"沉淀下来，就等于学费白交。下次同样的问题换个马甲再来，你还得重新排查一遍。

---

## 专项回归开关：贵的测试默认跳过

有些测试特别贵（比如项目启动的完整流程，要起多个 worker、跑很长的编排），不适合每次 CI 都跑。WinMatrix 用**环境变量开关**控制：

- kickoff 相关测试默认 `describe.skip`，只有设了 `E2E_KICKOFF=1` 才跑
- 组合命令分级：`test:quick`（快）、`test:verify`（类型 + unit + quick）、`test:all`（全量）

```json
// package.json（节选）
"test:verify": "npm run build:tsc && npm run test:unit && npm run test:quick",
```

这是个务实的权衡：**不是所有测试都要每次跑**。日常开发跑 quick，PR 合并跑 verify，发版前跑 all。贵的测试留给"重要节点"，平时的反馈环用快测试保住。

但注意——**跳过 ≠ mock**。kickoff 测试跳过了，但它真跑的时候是不 mock 的，连真实库、起真实 worker。跳过只是"暂时不跑"，不是"假装跑了"。

---

## migration-exec project：一个反污染的细节

最后讲一个很细但很重要的设计。看 vitest.config.ts 里的 `migration-exec` project：

```typescript
{
  test: {
    name: 'migration-exec',
    include: ['tests/integration/prisma/publish-persona-domain-pack-seed-migration.integration.test.ts'],
    // 不加载 shared setup（避免 DATABASE_URL / .env 回退污染专用守卫）
    threads: false, testTimeout: 30000, hookTimeout: 60000,
  },
},
```

注释里那句"不加载 shared setup（避免 DATABASE_URL / .env 回退污染专用守卫）"是关键。这个 project 测的是"数据库迁移脚本执行"，它需要**自己控制连哪个库**，不能被 shared setup 注入的 `DATABASE_URL` 污染。所以它故意不加载 shared setup，自己管自己的环境。

这是一个"测试隔离"的细节——**不同性质的测试，对环境隔离的要求不同**。大多数测试共享一套环境配置是 ok 的，但某些测试（比如迁移、schema 变更）必须用自己的独立环境，否则会互相污染。把它们拆成独立 project、独立 setup，是保证"互不干扰"的最干净做法。

**教训：测试隔离不只是"并发不互相踩"，还包括"环境配置不互相污染"。** 涉及 schema 变更、库级别的操作，要有独立的测试 project 和独立的环境，别和常规测试混在一个池子里。

---

## 给后来者的几条总结

1. **E2E 坚决不 mock 基础设施**。mock 掉的恰恰是生产里最容易出问题的部分。不 mock 才是 E2E 的价值。
2. **缺必填配置直接 `process.exit(1)`**，绝不偷偷 mock fallback。绿色谎言比红色失败可怕一百倍。
3. **测试分层按 mock 边界和 I/O 成本分**，不是按目录分。unit 占位串不碰 I/O，e2e 全真，边界清晰。
4. **E2E 自动建库建表**，让"不 mock"能落地。手动建库没人愿意跑。
5. **共享单例 app**，启动一次跑完所有用例。E2E 启动重，不能每测一遍。
6. **断言容忍真实的多路径**（200/404/503 都可能是合法响应）。E2E 测的是真实行为，不只是 happy path。
7. **把生产事故变成 fixture**。这是 ROI 最高的测试投资，防回归 + 知识沉淀双重价值。
8. **贵的测试用环境变量开关跳过**，但跳过 ≠ mock。真跑的时候还是不 mock。
9. **schema/迁移类测试要独立 project 独立环境**，别被常规 setup 污染。

测试这件事，本质是"你愿意为信心付多少成本"。mock 一切是廉价的信心，不 mock 是昂贵的信心——但只有后者在关键时刻靠得住。AI 平台的复杂度高、链路长、失败模式多，省测试成本省出来的钱，最后都会连本带利还给生产事故。

---

> **下一篇**：[《知识库入库 pipeline：从 PDF 到向量分块的全链路》](./20-rag-ingest-pipeline.md)——一份 PDF 传进知识库，要经历解析、分块、向量化、双写，才能被检索到。这条 pipeline 每一步都有坑，这篇讲怎么趟平。
