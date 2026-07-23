# 第 27 章 测试策略

> "测试不是成本，而是信心的来源。"

WinMatrix 拥有庞大的测试体系——1224+ 个单元测试文件、三级测试配置（unit/integration/e2e）、决策管线回归快照和死代码检测。本章将分析这套测试体系的组织结构和策略。

## 27.1 Vitest 三级测试配置

```typescript
// vitest.config.ts（完整 86 行）
import { defineConfig } from 'vitest/config';
import path from 'path';

const sharedAlias = [
  { find: /^@\/(.*)$/, replacement: path.resolve(__dirname, './src/$1') },
  { find: /^@tests\/(.*)$/, replacement: path.resolve(__dirname, './tests/$1') },
];

export default defineConfig({
  esbuild: { target: 'es2022' },
  resolve: {
    extensions: ['.ts', '.js'],
    alias: sharedAlias,
  },
  optimizeDeps: {
    exclude: ['pg'],    // 排除 pg，避免预构建问题
  },
  test: {
    environment: 'node',
    globals: true,
    exclude: ['node_modules/', 'dist/', 'coverage/'],
    reporters: process.env.VITEST_JSON_REPORT ? ['json', 'default'] : ['default', 'html'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html', 'lcov'],
      exclude: [
        'node_modules/', 'dist/', 'coverage/', 'tests/',
        '**/*.test.ts', '**/*.spec.ts', '**/*.d.ts',
        'scripts/', 'src/agents/config/',
      ],
    },
    projects: [
      // unit / integration / e2e
    ],
  },
});
```

### 三级测试项目

```typescript
projects: [
  {
    test: {
      name: 'unit',
      include: ['tests/unit/**/*.test.ts', 'tests/characterization/**/*.test.ts'],
      setupFiles: ['./tests/unit/setup.ts'],
      env: {
        DATABASE_URL: 'postgresql://user:pass@localhost:5432/winmatrix_unit_test',
      },
      isolate: true,        // 测试隔离
      threads: true,        // 多线程
      testTimeout: 10000,   // 10s 超时
      hookTimeout: 10000,
    },
  },
  {
    test: {
      name: 'integration',
      include: ['tests/integration/**/*.test.ts'],
      setupFiles: ['./tests/shared/setup.ts', './tests/integration/setup.ts'],
      threads: false,       // 串行
      testTimeout: 30000,   // 30s
      hookTimeout: 60000,   // 60s
    },
  },
  {
    test: {
      name: 'e2e',
      include: ['tests/e2e/**/*.{test,spec}.ts'],
      setupFiles: ['./tests/shared/setup.ts', './tests/e2e/setup.ts'],
      threads: false,
      testTimeout: 60000,   // 60s
      hookTimeout: 300000,  // 5 分钟
      retry: 1,             // 重试 1 次
    },
  },
],
```

### 三级测试对比

| 级别 | 线程 | 超时 | Hook 超时 | 重试 | 数据库 |
|------|------|------|----------|------|--------|
| **unit** | 多线程 | 10s | 10s | 0 | 内存桩 |
| **integration** | 串行 | 30s | 60s | 0 | 真实 PG |
| **e2e** | 串行 | 60s | 300s | 1 | 真实全栈 |

### 单元测试的内存桩

```typescript
// unit project 配置
env: {
  DATABASE_URL: 'postgresql://user:pass@localhost:5432/winmatrix_unit_test',
},
```

单元测试使用内存中的 DATABASE_URL 桩，不需要真实数据库——这让它可以在任何环境（包括 CI）快速运行。

## 27.2 测试目录结构

```
tests/
├── unit/                 # 1224+ 单元测试
│   ├── agents/
│   │   ├── action/
│   │   ├── application/
│   │   │   ├── coordination/
│   │   │   ├── orchestration/
│   │   │   ├── services/
│   │   │   └── turn/
│   │   ├── architecture/
│   │   ├── command/
│   │   └── context/       # 最大（60+ 文件）
│   └── setup.ts
├── integration/          # 集成测试
│   ├── es-degradation    # ES 降级
│   ├── es-knn-search     # ES kNN 搜索
│   ├── es-pg-sync        # ES-PG 同步
│   ├── es-search-quality # 搜索质量
│   ├── rag-ingest-async  # RAG 异步摄入
│   ├── agents/
│   ├── observability/
│   ├── prompt/
│   ├── trace/
│   ├── recipe/
│   └── startup/
├── e2e/                  # 端到端测试
│   ├── api.all.test.ts
│   ├── agent-resilience
│   ├── coding-workstation
│   ├── employees
│   ├── memory-system
│   ├── openclaw-fusion-phase2
│   ├── session-management
│   ├── skill-discovery
│   ├── tool-policy
│   ├── web-tools
│   ├── testApp.ts         # knip 入口
│   └── agent-decision-pipeline/
│       └── fixtures/expected/WF-*.json  # 40+ 决策管线快照
├── characterization/     # 特征化测试
│   ├── turn-admission-paths.test.ts
│   └── turn-admission-map.md
├── fixtures/             # 测试夹具
│   ├── sample-conversations/
│   ├── role-runtime-kernel/
│   ├── shared/
│   ├── decision-planner-91695-replay/
│   └── decision-replay/
├── manual/               # 手动基准测试（tsx 运行）
│   ├── benchmark-simple-task-runtime.ts
│   ├── benchmark-prompt-visibility-perf.ts
│   ├── complexity-tier-shadow-queries.ts
│   ├── simulate-complexity-tier-traffic.ts
│   ├── simulate-layered-routing.ts
│   └── verify-decision-shadow-metadata.ts
└── shared/
    └── setup.ts          # 共享 setup
```

## 27.3 单元测试示例

```typescript
// tests/unit/elasticsearch-config.test.ts
import { describe, expect, it } from 'vitest';
import {
  getElasticsearchEndpointConfig,
  isElasticsearchDisabled,
  parseElasticsearchNodes,
} from '@/infrastructure/config/elasticsearch.js';

describe('Elasticsearch endpoint config', () => {
  it('parses a single node into node client options', () => {
    const result = getElasticsearchEndpointConfig('http://127.0.0.1:9200');

    expect(result.normalizedNodes).toEqual(['http://127.0.0.1:9200']);
    expect(result.clientOptions).toEqual({
      node: 'http://127.0.0.1:9200',
      requestTimeout: 30_000,
      maxRetries: 3,
      sniffOnStart: false,
      name: 'winmatrix',
    });
    expect(result.displayUrl).toBe('http://127.0.0.1:9200');
  });
});
```

### 单元测试特点

- **纯函数测试**：不依赖外部服务
- **明确断言**：`toEqual` 精确匹配
- **快速执行**：10s 超时，多线程并行
- **`@/` 别名**：与生产代码相同的导入路径

## 27.4 集成测试

集成测试验证组件间的协作：

| 测试域 | 验证内容 |
|--------|---------|
| `es-degradation` | ES 不可用时的降级行为 |
| `es-knn-search` | 向量搜索准确性 |
| `es-pg-sync` | ES 与 PG 的数据同步 |
| `es-search-quality` | 搜索质量评估 |
| `rag-ingest-async` | RAG 异步摄入管线 |
| `observability` | 可观测性数据记录 |
| `startup` | 启动流程 |

集成测试使用真实数据库（30s 超时），串行执行避免数据竞争。

## 27.5 E2E 测试与决策管线快照

E2E 测试覆盖完整的业务流程：

```typescript
// tests/e2e/
// - api.all.test.ts           - API 全量测试（kickoff 入口）
// - agent-resilience          - Agent 韧性（崩溃恢复）
// - coding-workstation        - 编码工作站
// - memory-system             - 记忆系统
// - skill-discovery           - 技能发现
// - tool-policy               - 工具策略
```

### 决策管线回归快照

```
tests/e2e/agent-decision-pipeline/fixtures/expected/
├── WF-001.json
├── WF-002.json
├── ... (40+ 快照)
```

40+ 个决策管线的"黄金快照"——记录了特定输入下决策引擎的完整输出。每次修改决策引擎后，运行这些快照确保不引入回归。

### E2E 特点

- **5 分钟 Hook 超时**：完整业务流程可能很长
- **重试 1 次**：E2E 可能受环境影响，允许重试
- **串行执行**：避免端口/资源冲突

## 27.6 特征化测试

`tests/characterization/` 记录系统的实际行为（而非期望行为）：

```
turn-admission-paths.test.ts   # Turn 准入路径
turn-admission-map.md          # 准入路径地图（文档）
```

特征化测试（Characterization Test）是 Michael Feathers 提出的概念——对于遗留代码，先记录它的实际行为，再进行重构。这确保重构不会意外改变行为。

## 27.7 手动基准测试

`tests/manual/` 包含非 Vitest 的手动运行脚本：

```typescript
// benchmark-simple-task-runtime.ts    - 简单任务运行时基准
// benchmark-prompt-visibility-perf.ts - 提示词可见性性能
// simulate-complexity-tier-traffic.ts - 复杂度分层流量模拟
// simulate-layered-routing.ts         - 分层路由模拟
// verify-decision-shadow-metadata.ts  - 决策影子元数据验证
```

这些脚本通过 `tsx` 运行，用于性能基准测试和流量模拟——它们不是自动化测试，而是开发者的分析工具。

## 27.8 测试命令

```json
// package.json scripts（测试相关）
"test": "...",                    // 默认（e2e）
"test:unit": "...",               // 单元测试
"test:integration": "...",        // 集成测试
"test:routine": "...",            // 常规测试
"test:quick": "...",              // 快速测试
"test:signoff:obs-scenarios": "...",  // 可观测性场景验收
"test:e2e:fast": "...",           // 快速 E2E
"test:e2e:nollm": "...",          // 无 LLM 的 E2E（不调用真实 LLM）
"test:verify": "...",             // 验证（build:tsc + unit + quick）
```

`test:e2e:nollm` 是一个有意思的命令——E2E 测试但不调用真实 LLM（使用桩），既验证了流程又节省了成本。

## 27.9 Knip 死代码检测

```json
// knip.json
{
  "$schema": "https://unpkg.com/knip@latest/schema.json",
  "entry": [
    "src/index.ts",                  // 生产入口
    "scripts/gemini-text-to-image.ts", // 脚本
    "tests/e2e/testApp.ts"           // E2E 测试入口
  ],
  "project": ["src/**/*.ts"],
  "ignore": ["dist/**", "tests/**", "**/*.test.ts", "**/*.spec.ts"],
  "ignoreDependencies": [
    "@swc/cli", "@swc/core",
    "@vitest/coverage-v8", "@vitest/ui", "vitest", "tsx"
  ]
}
```

### Knip 配置要点

1. **三个入口**：生产入口 + 脚本 + 测试入口
2. **项目范围**：`src/**/*.ts`
3. **忽略**：dist、tests、测试文件
4. **依赖白名单**：SWC、Vitest、tsx 等 CLI 工具（不通过 import 使用）

```bash
npm run check:unused          # 全量检测
npm run check:unused:production  # 仅生产代码
```

## 27.10 架构守卫测试

除了功能测试，WinMatrix 还有架构守卫：

```json
"check:layers": "node scripts/check-layer-imports.cjs",
"check:agent-layers:strict": "node scripts/check-agent-layer-imports.cjs --strict && npm run check:tool-kernel-consumption",
"check:cdw-boundaries": "node scripts/check-cdw-boundaries.mjs",
"check:observability-rules": "...",
"check:tool-kernel-consumption": "...",
"check:time-semantics": "...",
```

这些检查在 CI 中执行，确保代码不违反架构约束（见第 3 章）。

## 本章小结

本章深入分析了 WinMatrix 的测试策略：

1. **三级测试**：unit（10s，多线程，内存桩）/ integration（30s，串行，真实 PG）/ e2e（60s，重试，全栈）
2. **1224+ 单元测试**：按业务域组织，context 最大（60+ 文件）
3. **集成测试**：ES 降级、kNN 搜索、PG 同步、RAG 摄入
4. **决策管线快照**：40+ 黄金快照，防止回归
5. **特征化测试**：记录实际行为，支持安全重构
6. **手动基准测试**：性能基准 + 流量模拟
7. **test:e2e:nollm**：不调用真实 LLM 的 E2E
8. **Knip 死代码检测**：三个入口 + 项目范围 + 依赖白名单
9. **架构守卫**：分层检查 + CDW 边界 + 可观测性规则 + 时间语义

---

至此，本书的 29 章正文全部完成。附录 A（术语表）和附录 B（源码导航）提供了快速参考。
