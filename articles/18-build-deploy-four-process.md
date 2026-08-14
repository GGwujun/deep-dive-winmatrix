# 构建与部署：dev / prod / k8s 四进程对齐

> 这是《WinMatrix 开发经验文集》第 18 篇。讲一个工程味很浓、却没多少人愿意讲透的问题：当你的一份代码要跑成 api、scheduled、rag、embedding 四个进程，开发本地、docker-compose、k8s 三套环境怎么保持对齐？构建链又该怎么搭才不出事？代码来自 WinMatrix 后端真实实现。

很多人对"部署"的理解是：写完代码，`npm run build`，把 dist 扔到服务器上 `node dist/index.js` 起来，完事。

一旦你的系统稍微复杂一点，这套就崩了。WinMatrix 跑起来其实是**四个进程**：

```
┌─────────────────────┐  ┌─────────────────────┐
│  api（HTTP/WebSocket）│  │  scheduled（后台任务）│
│  对外服务             │  │  18 个 BullMQ worker │
└─────────────────────┘  └─────────────────────┘
┌─────────────────────┐  ┌─────────────────────┐
│  rag（向量检索/重排）  │  │  embedding（向量化） │
│  独立进程，ES 重排    │  │  Fastify 微服务      │
└─────────────────────┘  └─────────────────────┘
```

为什么拆四个？因为它们对资源的需求完全不同——api 要低延迟、scheduled 要长跑稳、rag 要吃内存做向量计算、embedding 要护 GPU/模型容量。塞在一个进程里，任何一个 OOM 或卡死，全军覆没。

这篇文章讲我们把"一份代码、四个进程、三套环境"做对齐的经验。核心思路：**用进程角色守卫把"谁该跑什么"焊死在入口文件里，让三套环境共享同一份对齐契约**。

---

## 四个进程，靠一个环境变量焊死

先看最关键的一个设计——**进程角色守卫**。每个进程入口文件的第一行不是 import 业务代码，而是先断言自己的角色：

```typescript
// src/startup/processRole.ts（全文，18 行）
export type ProcessRole = 'api' | 'scheduled' | 'rag';

/**
 * Fail-fast when WIN_PROCESS_ROLE does not match the dedicated entry file.
 */
export function assertProcessRole(expected: ProcessRole): void {
  const actual = process.env.WIN_PROCESS_ROLE?.trim();
  if (actual !== expected) {
    const hint =
      actual === undefined || actual === ''
        ? 'WIN_PROCESS_ROLE is unset'
        : `WIN_PROCESS_ROLE=${actual}`;
    throw new Error(
      `[ProcessRole] Entry requires WIN_PROCESS_ROLE=${expected}, but ${hint}. ` +
        'Fix manifest command/env mismatch before starting.',
    );
  }
}
```

然后每个进程入口这样用：

```typescript
// src/scheduled-worker.ts（第 1-20 行）
#!/usr/bin/env node
import './infrastructure/sandbox/config/undiciConfig.js';
import { assertProcessRole } from '@/startup/processRole.js';
try { assertProcessRole('scheduled'); }
catch (err) {
  const message = err instanceof Error ? err.message : String(err);
  process.stderr.write(`[ProcessRole] scheduled-worker entry startup aborted: ${message}\n`);
  process.exit(1);
}
await import('@/startup/scheduledEntry.js');
```

`rag-worker.ts` 入口几乎一样，只是 `assertProcessRole('rag')`。

这个守卫的意思是：**scheduled-worker 这个入口，只有当 `WIN_PROCESS_ROLE=scheduled` 时才允许启动，否则直接 exit(1)**。

为什么要焊死？因为真实发生过这种事故：有人改了部署脚本，把启动命令从 `WIN_PROCESS_ROLE=scheduled node dist/scheduled-worker.js` 误改成了 `node dist/scheduled-worker.js`（漏了环境变量）。没有守卫的话，scheduled-worker 会以"未声明角色"的身份跑起来——它可能部分能工作（启动一些 worker），但配置加载、路由注册都是错乱的，故障极其隐蔽。有了守卫，漏配环境变量直接启动失败，立刻暴露问题。

注意守卫之后是 `await import('@/startup/scheduledEntry.js')`——**动态 import**。这意味着业务代码在守卫通过后才加载。如果守卫失败，连业务代码都不会 import，不会有任何副作用进程跑起来。

**教训：进程入口的第一件事是"确认我没被错配"。** 任何"不该跑却跑了"的进程，都比"跑不起来"危险得多——前者是隐形故障，后者是显性故障。fail-fast 永远优于 fail-silent。

---

## package.json：四进程的启动契约对齐

四个进程的启动方式，在 `package.json` 的 scripts 里定义得清清楚楚：

```json
// winmatrix-server/package.json（第 26-31 行）
"start:prod": "WIN_PROCESS_ROLE=api node dist/api.js",
"start:prod:monolith": "node dist/index.js",
"start:prod:api": "WIN_PROCESS_ROLE=api node dist/api.js",
"start:prod:scheduled": "WIN_PROCESS_ROLE=scheduled node dist/scheduled-worker.js",
"start:prod:rag": "WIN_PROCESS_ROLE=rag node dist/rag-worker.js",
```

注意每条命令都**内联了 `WIN_PROCESS_ROLE` 环境变量**。这不是冗余，这是**把"角色守卫要的输入"焊进启动命令本身**——不依赖外部环境是否正确设了这个变量，命令自带。

对开发者本地，对应的 dev 命令也有四条：

```json
// package.json（第 19-23 行）
"dev:api": "node scripts/dev-singleton.cjs --role api src/api.ts",
"dev:scheduled": "node scripts/dev-singleton.cjs --role scheduled src/scheduled-worker.ts",
"dev:rag": "node scripts/dev-singleton.cjs --role rag src/rag-worker.ts",
"dev:embedding": "node scripts/dev-singleton.cjs src/embedding-server.ts",
```

开发时开四个终端跑这四条，模拟线上四进程。这背后的契约是：**dev 和 prod 的进程拓扑必须一一对应**。你在 dev 时是一个进程跑所有东西（`dev` 单体），还是分进程跑？我们选择了"分进程跑"的 dev 模式，因为只有这样，dev 才能提前暴露"某个 worker 在 api 进程里漏注册了"这类部署期才会发现的问题。

> 还有一条 `start:prod:monolith`——单体模式，不设 `WIN_PROCESS_ROLE`。这是给"小规模部署、一台机器跑不下四个进程分离"的场景留的兜底。但注意它的入口是 `dist/index.js`，而 `index.ts` 内部会按"all-in-one"模式加载所有东西。这是有意保留的灵活性，但生产主力部署还是走四进程分离。

---

## 构建链：esbuild 打包 + tsc 校类型

讲完运行，讲构建。WinMatrix 的构建链有个很多人会困惑的设计：**主构建用 esbuild，类型检查用单独的 tsc 命令**，两者分开。

```json
// package.json（第 9 行，主构建）
"build": "npx prisma generate && node scripts/check-no-js-in-src.cjs && node scripts/build-esbuild.cjs && node scripts/tsc-alias-build.cjs && node scripts/fix-remaining-alias-in-dist.cjs && node scripts/verify-no-alias-in-dist.cjs",

// package.json（第 14 行，类型检查）
"build:tsc": "npx prisma generate && tsc --project tsconfig.typecheck.json --noEmit",
```

为什么不直接用 `tsc` 一把梭出 dist？因为**esbuild 快一到两个数量级**。我们代码量大，纯 tsc 构建要一两分钟，esbuild 只要几秒。开发反馈环对团队士气的影响远大于"多写一行脚本"的成本。

但 esbuild 有个硬伤：**它不做类型检查，只做转译**。也就是说，类型错误能在 esbuild 构建里"漏网"——构建成功了，运行时却可能因为类型不匹配炸掉。

解法是**把类型检查拆成独立步骤**（`build:tsc` 用 `--noEmit`，只检查不产出），在 CI 或 pre-commit 跑。这样构建快、类型也查了，互不耽误：

```
开发：build（esbuild，秒级）→ 立刻能跑
CI：  build:tsc（tsc --noEmit）→ 类型闸门
```

构建链里还有几个细节值得注意：

- `check-no-js-in-src.cjs`——禁止 src 下混入 .js 文件（强制全 TypeScript）
- `fix-remaining-alias-in-dist.cjs`——esbuild 产物里残留的路径别名（`@/`）做一次文本级修正
- `verify-no-alias-in-dist.cjs`——最后再校验一遍 dist 里没有任何 `@/` 别名残留

这三步是在解决 esbuild + Node ESM + 路径别名（`@/` → `src/`）的一个经典痛点。esbuild 打包时能解析别名，但打包成多 chunk 时，有些边界情况会漏掉别名替换，导致运行时 `ERR_MODULE_NOT_FOUND`。所以构建完要做两轮校验，确保 dist 里全是相对路径。

**教训：构建工具的选型不是"哪个更全"，而是"哪个更适合你的反馈环"。** esbuild 快但不管类型，tsc 全但慢——把它们拆成两个步骤，各管一摊，比纠结"用哪个"更实际。代价是多写几行脚本，收益是开发体验。

---

## docker-compose：四服务镜像，依赖健康检查

生产部署的主力是 docker-compose。看它的服务结构：

```yaml
# docker/docker-compose.yml（节选）
services:
  winmatrix:                    # api 进程
    container_name: winmatrix-server
    environment:
      - WIN_PROCESS_ROLE=api
    ports: [...]
  
  winmatrix-embedding:          # embedding 微服务
    container_name: winmatrix-embedding
    ports: [8401]
  
  winmatrix-scheduled-worker:   # scheduled 进程
    container_name: winmatrix-scheduled-worker
    environment:
      - WIN_PROCESS_ROLE=scheduled
      - WORKER_HEALTH_PORT=8402
    depends_on:
      winmatrix-embedding:
        condition: service_healthy
  
  winmatrix-rag-worker:         # rag 进程
    container_name: winmatrix-rag-worker
    environment:
      - WIN_PROCESS_ROLE=rag
      - WORKER_HEALTH_PORT=8402
    depends_on:
      winmatrix-embedding:
        condition: service_healthy
```

几个对齐细节：

1. **每个服务的 environment 里都显式写 `WIN_PROCESS_ROLE`**。这和 package.json 的 `start:prod:*` 命令内联角色变量是对齐的——同一个守卫契约，dev 用脚本变量、docker-compose 用环境变量，殊途同归。

2. **scheduled 和 rag 都 `depends_on: winmatrix-embedding: condition: service_healthy`**。意思是 embedding 服务必须先通过健康检查，scheduled 和 rag 才启动。为什么？因为 scheduled 处理的知识任务、rag 的检索任务，都可能要调 embedding 服务向量化。如果 embedding 还没起来就启动它们，第一批任务会因为 embedding 不可达而失败。`condition: service_healthy` 比单纯的 `depends_on`（只等容器启动）可靠得多——容器启动 ≠ 服务就绪。

3. **scheduled 和 rag 都带 `WORKER_HEALTH_PORT=8402`**。这两个进程不对外暴露 HTTP API，但它们需要一个健康检查端口，让编排系统知道"我还活着"。这个端口跑的是一个极简的 health endpoint，不和 api 的 3000 端口冲突。

加上 pgbouncer，docker-compose 一共 7 个服务（4 应用 + pgbouncer + 共享网络 + volumes）。这套 compose 文件就是"单机生产"的标准部署形态。

---

## start.sh：跨平台 + 运行时隔离 + prod 警告

除了 docker-compose，我们还有个 `scripts/deploy/start.sh`（1240 行），是开发者和运维都会用的"万能启停脚本"。它做了三件值得讲的事。

### 第一件：跨平台进程清理

start.sh 要同时支持 Linux（pgrep/kill 树）和 Windows（taskkill //T //F、PowerShell）。`free_port` 和 `cleanup_winmatrix_orphans` 函数会跨平台地清理占用端口和孤儿进程。这听起来不起眼，但一个团队里总有人用 Windows、有人用 Mac、CI 跑在 Linux——同一个脚本三端都能跑，省下无数"我这能跑你那不能跑"的扯皮。

### 第二件：运行时队列隔离

```bash
# scripts/deploy/start.sh（第 287-295 行）
configure_runtime_isolation() {
    if [ -z "${WIN_RUNTIME_ISOLATION_ID:-}" ]; then
        # 没显式设 → 用 hostname 自动生成
        WIN_RUNTIME_ISOLATION_ID="$(printf '%s' "$raw_hostname" | tr ... | cut -c1-48)"
        WIN_RUNTIME_ISOLATION_ID="${WIN_RUNTIME_ISOLATION_ID:-unknown-host}"
        export WIN_RUNTIME_ISOLATION_ID
    fi
    print_info "BullMQ runtime isolation: $WIN_RUNTIME_ISOLATION_ID"
}
```

这一段和上一篇讲的 BullMQ 队列隔离是配套的——start.sh 启动前先设好 `WIN_RUNTIME_ISOLATION_ID`，让本机所有 BullMQ 队列都带 hostname 后缀，不串到共享 Redis 上的生产队列。脚本层和代码层一起把这个隔离做实。

### 第三件：prod 模式的明确警告

```bash
# scripts/deploy/start.sh（第 864 行）
print_warning "生产模式仅启动 API；scheduled/rag/embedding 请使用 docker compose"
```

`start:prod` 这条命令**只启动 api 进程**。如果你在生产环境用它来"一键启动全部"，你会得到一个只有 api、没有后台任务、没有检索、没有向量化的半残系统。所以脚本明确警告：prod 模式只起 API，其余三个进程请用 docker-compose 起全栈。

为什么 `start:prod` 不干脆把四个进程都起起来？因为在一台机器上用脚本起四个进程（nohup & 后台跑）远不如 docker-compose 的健康检查、重启策略、日志聚合来得可靠。脚本启动是"临时/调试"场景，docker-compose 才是"正经生产"场景。把两者的边界讲清楚，比假装脚本能搞定一切要诚实。

---

## k8s：显式覆盖环境变量，防 Secret 污染

最后看 k8s 部署。最值得讲的一个细节是**显式覆盖关键环境变量**：

```yaml
# k8s/deployment.yaml（第 30-36 行）
# 显式覆盖 DB/Redis 与 NODE_ENV，避免 Secret 中 NODE_ENV=test
# 或其它来源覆盖导致仍连测试环境
env:
  - name: NODE_ENV
    value: production
  - name: DATABASE_URL
    valueFrom: { secretKeyRef: ... }
  - name: REDIS_URL
    valueFrom: { secretKeyRef: ... }
```

这段注释点出了一个真实的坑：**Secret 对象里的环境变量会覆盖 deployment 里直接写的 `value`**，但顺序和优先级在不同 k8s 版本里有过微妙变化。如果 Secret 里不小心带了一个 `NODE_ENV=test`（从测试环境复制过来没改），deployment 启动的 pod 就会以 test 模式跑——连测试库、开 debug 日志、跳过某些生产守卫。

解法是**在 deployment 里显式写 `NODE_ENV: production`**，并把它放在从 Secret 取的变量之前。同时只从 Secret 取真正敏感的 `DATABASE_URL` / `REDIS_URL`，不让 Secret 承载"运行模式"这种本该由部署清单决定的配置。

### liveness vs readiness 的分工

```yaml
# k8s/deployment.yaml（第 62-78 行）
livenessProbe:
  httpGet: { path: /health }
  initialDelaySeconds: 90
readinessProbe:
  httpGet: { path: /readyz }
  initialDelaySeconds: 60
```

这两个探针分工明确：

- **liveness `/health`**（初始延迟 90s）：基本存活检查。失败了 **重启 pod**。用于"进程卡死、死锁、OOM"这类需要重启才能恢复的故障。
- **readiness `/readyz`**（初始延迟 60s）：摘流专用，带进程状态、连接池、DB 超时检查。失败了 **不重启，只是从 Service 摘掉流量**。用于"我还活着，但暂时没法接请求"（比如正在重连数据库、正在预热缓存）。

关键区别：**readiness 失败不重启**。这是一个容易被忽略的设计——很多系统把 readiness 和 liveness 配成一样的，结果临时性故障（DB 重连 30 秒）被当成"进程死了"触发重启，重启完还是要重连，陷入重启循环。把两者分开，临时故障靠 readiness 摘流自愈，真死了才靠 liveness 重启。

对于"致命但无法靠重启恢复"的故障（比如配置错到启动都过不了），靠的是**应用自己 `process.exit(1)` 主动退出**，让 pod 重启而非无限重试。这是把"知道自己没救了就别硬撑"做成应用层责任。

---

## 三套环境对齐的总图

把上面所有点串起来，dev / docker-compose / k8s 三套环境的对齐关系：

```
                  进程角色守卫（assertProcessRole）
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
     dev（4 终端）    docker-compose    k8s
          │               │               │
  dev:api → WIN_     环境变量         env 显式覆盖
  PROCESS_ROLE=api   WIN_PROCESS_     NODE_ENV=production
  dev:scheduled      ROLE=api/sched/  （防 Secret 污染）
  dev:rag            rag
  dev:embedding
          │               │               │
          └───────────────┼───────────────┘
                          ▼
              同一份 dist 产物
              （esbuild 构建 + tsc 校类型）
```

三套环境用的是**同一份构建产物、同一套角色守卫、同一种进程拓扑**。差异只在"怎么设环境变量、怎么做健康检查、怎么编排"。这种对齐的核心收益是：**dev 能暴露的问题，prod 也能暴露；prod 出的问题，能在 dev 复现**。如果三套环境各搞各的，dev 跑得好好的上了 prod 炸了，那种调试痛苦谁经历谁知道。

---

## 给后来者的几条总结

1. **用进程角色守卫把"谁该跑什么"焊死在入口**。守卫要放在业务代码 import 之前，错了直接 exit(1)。
2. **构建链拆成"快速转译 + 独立类型检查"两步**。esbuild 管快，tsc --noEmit 管全，别纠结用哪个。
3. **dev 和 prod 的进程拓扑必须一一对应**。dev 分进程跑，能提前暴露部署期才暴露的问题。
4. **docker-compose 的依赖要用 `condition: service_healthy`**，不是裸 `depends_on`。容器启动 ≠ 服务就绪。
5. **prod 模式的启动脚本要明确警告"我只起了一部分"**。别假装脚本能一键搞定全栈，诚实标注边界。
6. **k8s 的关键环境变量要在 deployment 显式覆盖**，别全塞 Secret，防 Secret 污染运行模式。
7. **liveness 和 readiness 分工**：临时故障靠 readiness 摘流自愈，真死了才靠 liveness 重启，别混用。
8. **构建产物要校验路径别名残留**。esbuild + ESM + 别名是个经典坑，构建完做两轮校验不丢人。

部署这件事，不性感，但它决定了"你的代码能不能真的跑起来给人用"。把四进程对齐、构建链、健康检查、环境变量隔离做扎实，你的平台才有"上得去、稳得住"的基础。

---

> **下一篇**：[《测试策略：为什么我们的 E2E 坚决不 mock 数据库》](./19-e2e-no-mock.md)——E2E 测试该不该 mock 数据库、Redis、LLM？我们选择了"坚决不 mock"，这篇讲为什么，以及怎么把不 mock 的 E2E 跑得又稳又快。
