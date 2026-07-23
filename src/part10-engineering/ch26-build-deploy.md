# 第 26 章 构建与部署

> "从源码到生产，每一步都应该是可重复的。"

WinMatrix 的构建部署体系涵盖了从 TypeScript 编译到 Kubernetes 部署的完整链路。本章将分析 esbuild 构建管线、Docker 多阶段构建、K8s 部署清单和进程拆分策略。

## 26.1 构建管线

构建脚本体现了完整的六步管线：

```json
// winmatrix-server/package.json（第 8-14 行）
{
  "scripts": {
    "build": "npx prisma generate && \
              node scripts/check-no-js-in-src.cjs && \
              node scripts/build-esbuild.cjs && \
              node scripts/tsc-alias-build.cjs && \
              node scripts/fix-remaining-alias-in-dist.cjs && \
              node scripts/verify-no-alias-in-dist.cjs",

    "build:swc": "node scripts/check-no-js-in-src.cjs && swc src -d dist --strip-leading-paths",
    "build:tsc-emit": "node scripts/check-no-js-in-src.cjs && node --max-old-space-size=12288 ./node_modules/typescript/bin/tsc -p tsconfig.build.json --incremental && node scripts/tsc-alias-build.cjs",
    "build:prod": "node scripts/check-no-js-in-src.cjs && node scripts/build-esbuild.cjs && node scripts/tsc-alias-build.cjs && node scripts/fix-remaining-alias-in-dist.cjs && node scripts/verify-no-alias-in-dist.cjs",
    "build:tsc": "npx prisma generate && tsc --project tsconfig.typecheck.json --noEmit"
  }
}
```

### 六步构建详解

| 步骤 | 脚本 | 职责 |
|------|------|------|
| 1 | `prisma generate` | 生成 Prisma Client |
| 2 | `check-no-js-in-src` | 确保 src/ 无 .js 文件混入 |
| 3 | `build-esbuild` | esbuild 编译 TypeScript → JavaScript |
| 4 | `tsc-alias-build` | 将 `@/` 别名替换为相对路径 |
| 5 | `fix-remaining-alias` | 修补残余别名 |
| 6 | `verify-no-alias-in-dist` | 校验 dist/ 无残留别名 |

### 多种构建工具

WinMatrix 支持三种构建工具：

- **esbuild**（主构建器）：最快，`bundle: false` 逐文件转译
- **SWC**（备选）：`build:swc`，`--strip-leading-paths`
- **tsc**（类型检查）：`build:tsc`，`--noEmit` 仅检查不产出

## 26.2 esbuild 构建脚本

```javascript
// scripts/build-esbuild.cjs（第 1-68 行）
/**
 * esbuild 多文件转译：将 src 下 .ts 转为 dist 下 .js，保持目录结构。
 * 构建产物必须只输出到 dist/，禁止输出到 src/。
 * 路径别名 @/ 在构建时解析；产出中的 @/ 由 tsc-alias 替换。
 */
const esbuild = require('esbuild');
const path = require('path');
const fs = require('fs');

const rootDir = path.resolve(__dirname, '..');
const srcDir = path.join(rootDir, 'src');

/** 将 @/ 标为 external，产出保留 @/，由 tsc-alias 替换 */
const aliasExternalPlugin = {
  name: 'alias-external',
  setup(build) {
    build.onResolve({ filter: /^@\// }, () => ({ path: 'external', external: true }));
    build.onResolve({ filter: /^@$/ }, () => ({ path: 'external', external: true }));
  },
};

/** 排除目录 */
const EXCLUDE_DIRS = ['typeorm', 'persistence/prisma/generated'];

function collectTsFiles(dir, base = '') {
  // 递归收集 .ts 文件，排除 EXCLUDE_DIRS 和 .test.ts/.spec.ts
}

async function main() {
  const entryPoints = collectTsFiles(srcDir);

  await esbuild.build({
    entryPoints,
    outdir: path.join(rootDir, 'dist'),
    outbase: srcDir,
    bundle: false,              // 逐文件转译，不打包
    format: 'esm',              // ESM 格式
    platform: 'node',
    target: 'es2022',
    sourcemap: false,
    plugins: [aliasExternalPlugin],
    outExtension: { '.js': '.js' },
  });

  // 校验每个 entry 都有对应 .js 产物
  // 避免 Docker/Linux 下缺文件导致 ERR_MODULE_NOT_FOUND
  const missing = [];
  for (const entry of entryPoints) {
    const outPath = path.join(distDir, rel.replace(/\.ts$/, '.js'));
    if (!fs.existsSync(outPath)) missing.push(/* ... */);
  }
}
```

关键设计：

1. **`bundle: false`**：逐文件转译，保持目录结构，不打包
2. **`aliasExternalPlugin`**：将 `@/` 标为 external，保留到产出中由 tsc-alias 处理
3. **排除目录**：`typeorm`（已迁移到 Prisma）和 `prisma/generated`（自动生成）
4. **产物校验**：构建后校验每个 entry 都有对应 .js，避免 Docker 环境缺文件

## 26.3 tsc-alias 构建

```javascript
// scripts/tsc-alias-build.cjs（完整 13 行）
/**
 * 在项目根目录下执行 tsc-alias，避免 Docker 等环境下 cwd 非项目根导致 "Invalid file path"。
 */
const path = require('path');
const { execSync } = require('child_process');

const rootDir = path.resolve(__dirname, '..');

execSync('npx tsc-alias -p tsconfig.build.json', {
  stdio: 'inherit',
  cwd: rootDir,    // 显式指定项目根目录
});
```

这个脚本的存在解决了一个真实的 Docker 环境问题——`tsc-alias` 在 cwd 不是项目根时会报 "Invalid file path"。通过显式指定 `cwd: rootDir` 解决。

## 26.4 Docker 多阶段构建

```dockerfile
# docker/Dockerfile（第 1-80 行，三阶段构建）
# ============================================
# WinMatrix 多阶段构建 Dockerfile
# 阶段1: 前端构建 (winmatrix-ui)
# 阶段2: 后端构建
# 阶段3: 生产运行
#
# 基于公司 Harbor 双基础镜像：
#   构建阶段使用 winmatrix-node-build-base
#   运行阶段使用 winmatrix-node-runtime-base
# ============================================

# 基础镜像参数（可通过 --build-arg 覆盖）
ARG BUILD_BASE_IMAGE=registry.winning.com.cn/devops/winmatrix-node-build-base:25-v1
ARG RUNTIME_BASE_IMAGE=registry.winning.com.cn/devops/winmatrix-node-runtime-base:25-v2

# ============================================
# 阶段1: 前端构建
# ============================================
FROM ${BUILD_BASE_IMAGE} AS webui-builder
WORKDIR /app/webui
COPY winmatrix-ui/package*.json ./

# 内网 Nexus registry，可切换镜像
ARG NPM_REGISTRY_URL=http://172.16.9.57:8081/repository/npm-group
ARG NPM_USE_MIRROR=0
RUN if [ "$NPM_USE_MIRROR" = "1" ]; then npm config set registry https://registry.npmmirror.com; \
    else npm config set registry "$NPM_REGISTRY_URL"; fi

# BuildKit 缓存挂载
RUN --mount=type=cache,id=winmatrix-ui-npm-cache,target=/root/.npm \
    npm ci --no-audit --no-fund --fetch-retries=5

COPY winmatrix-ui/ ./
ARG VITE_WEBUI_BASE_PATH=/winmatrix-ui
ARG VITE_API_BASE_URL=/winmatrix
RUN npm run build

# ============================================
# 阶段2a: 后端依赖（与源码解耦）
# ============================================
FROM ${BUILD_BASE_IMAGE} AS backend-deps
WORKDIR /app/backend

# @winmatrix/protocol：预编译 dist（不在镜像内下载）
COPY clients/protocol/package.json /app/clients/protocol/
COPY clients/protocol/dist/ /app/clients/protocol/dist/
RUN test -f /app/clients/protocol/dist/index.js || \
  (echo "Error: clients/protocol/dist/index.js missing" && exit 1)

COPY winmatrix-server/package.json ./
COPY winmatrix-server/package-lock.json ./
```

### Docker 构建亮点

1. **双基础镜像**：构建阶段（build-base）和运行阶段（runtime-base）分离
2. **BuildKit 缓存挂载**：`--mount=type=cache` 加速 npm 安装
3. **registry 切换**：内网 Nexus 默认，可切换 npmmirror
4. **依赖解耦**：`backend-deps` 阶段仅依赖 package.json，源码变更不触发重装
5. **protocol 预编译**：`@winmatrix/protocol` 使用宿主机预编译的 dist

### 其他 Dockerfile

```
docker/
├── Dockerfile                          # 主应用
├── Dockerfile.node-base               # Node 基础镜像
├── Dockerfile.openclaw-workstation    # OpenClaw 工作站
├── Dockerfile.sre-workstation         # SRE 工作站
└── base/
    ├── Dockerfile.coding-workstation-base
    ├── Dockerfile.node-build          # 构建基础
    └── Dockerfile.node-runtime        # 运行基础
```

## 26.5 Kubernetes 部署

```
k8s/
├── namespace.yaml           # 命名空间
├── deployment.yaml          # 主应用部署
├── service.yaml             # 服务
├── ingress.yaml             # 入口
├── configmap.yaml           # 配置
├── secret.yaml.example      # 密钥示例
├── pvc.yaml                 # 持久卷
├── elasticsearch/           # ES 集群
├── minio/                   # MinIO 对象存储
├── neo4j/                   # Neo4j 图数据库
└── sandbox-api/             # 沙箱 API（含 RBAC）
```

### 主应用 Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: winmatrix
  namespace: winmatrix
spec:
  replicas: 1
  template:
    spec:
      imagePullSecrets:
        - name: winex-wxp-copilot    # Harbor 镜像拉取密钥
      containers:
        - name: winmatrix-server
          image: ...:latest
          ports:
            - containerPort: 3000    # HTTP 端口
          env:
            - NODE_ENV=production
          envFrom:
            - configMapRef: { name: winmatrix-config }
            - secretRef: { name: winmatrix-secret }
          volumeMounts:
            - { name: data, mountPath: ... }
            - { name: logs, mountPath: ... }
            - { name: workspace, mountPath: /winmatrix-workspace }
            - { name: claude-code-config, mountPath: /opt/winmatrix/config/claude-code, readOnly: true }
```

### 基础设施服务

K8s 部署包含完整的基础设施栈：

| 服务 | 用途 | 资源 |
|------|------|------|
| **Elasticsearch** | 向量/全文搜索 | Deployment + Kibana + PVC |
| **MinIO** | S3 兼容对象存储 | Deployment + Service + Job(init bucket) |
| **Neo4j** | 知识图谱 | Deployment + Service + PVC |
| **sandbox-api** | K8s 沙箱管理 | Deployment + RBAC（ServiceAccount/Role/RoleBinding） |

## 26.6 进程拆分部署

生产环境支持三种进程独立部署：

```json
// package.json
"start:prod:api": "WIN_PROCESS_ROLE=api node dist/api.js",
"start:prod:scheduled": "WIN_PROCESS_ROLE=scheduled node dist/scheduled-worker.js",
"start:prod:rag": "WIN_PROCESS_ROLE=rag node dist/rag-worker.js",
"start:prod:monolith": "node dist/index.js"
```

在 K8s 中，可以通过修改 Deployment 的 command 和环境变量实现进程拆分：

```yaml
# 进程拆分示例（概念性）
containers:
  - name: api
    command: ["node", "dist/api.js"]
    env:
      - { name: WIN_PROCESS_ROLE, value: api }
```

### 开发模式

```json
"dev": "node scripts/dev-singleton.cjs src/index.ts",
"dev:api": "node scripts/dev-singleton.cjs --role api src/api.ts",
"dev:scheduled": "node scripts/dev-singleton.cjs --role scheduled src/scheduled-worker.ts",
"dev:rag": "node scripts/dev-singleton.cjs --role rag src/rag-worker.ts"
```

开发模式使用 `dev-singleton.cjs` 确保单例（DI 容器、Prisma Client 等）在热重载时不分裂。

## 26.7 CI/CD

```
.tekton-ci.yaml    # Tekton CI 管线
.github/workflows/ # GitHub Actions
```

CI 管线执行：

1. 代码检出
2. 依赖安装
3. 类型检查（`build:tsc`）
4. 分层检查（`check:layers`、`check:agent-layers:strict`）
5. 单元测试（`test:unit`）
6. 构建（`build:prod`）
7. Docker 镜像构建并推送到 Harbor
8. K8s 部署

## 本章小结

本章深入分析了 WinMatrix 的构建与部署：

1. **六步构建管线**：prisma generate → check → esbuild → tsc-alias → fix → verify
2. **三种构建工具**：esbuild（主）/ SWC（备）/ tsc（类型检查）
3. **esbuild 配置**：bundle:false 逐文件转译 + `@/` external + 产物校验
4. **Docker 三阶段**：前端构建 → 后端依赖 → 生产运行
5. **双基础镜像**：构建阶段和运行阶段分离
6. **BuildKit 缓存**：npm 缓存挂载加速构建
7. **K8s 完整栈**：主应用 + ES + MinIO + Neo4j + sandbox-api
8. **进程拆分**：api / scheduled / rag 独立部署
9. **CI/CD**：Tekton + GitHub Actions，分层检查 + 类型检查 + 测试

在下一章中，我们将深入测试策略。
