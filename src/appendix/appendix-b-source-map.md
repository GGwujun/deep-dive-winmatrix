# 附录 B：源码导航

本附录提供 WinMatrix 后端关键文件的索引，按六层架构组织。所有路径相对于 `winmatrix-server/`。

## 入口与启动

| 文件 | 行数 | 职责 |
|------|------|------|
| `src/index.ts` | 248 | dev all-in-one 入口 |
| `src/api.ts` | 19 | 生产 API 入口（WIN_PROCESS_ROLE=api） |
| `src/scheduled-worker.ts` | 20 | 定时任务进程入口 |
| `src/rag-worker.ts` | 20 | RAG 进程入口 |
| `src/embedding-server.ts` | - | Embedding 服务（端口 8401） |
| `src/startup/common.ts` | 630 | init 函数 + gracefulExit + 信号处理 |
| `src/startup/api.ts` | 567 | Fastify 插件注册 + startListening |
| `src/startup/apiEntry.ts` | 21 | API 入口逻辑 |
| `src/startup/processRole.ts` | 19 | assertProcessRole 守卫 |
| `src/startup/agents.ts` | 166 | initAgentStack + Role 工厂注册 |
| `src/startup/shutdown/apiPrefix.ts` | - | 关闭步骤 1-3 |
| `src/startup/shutdown/scheduledWorkers.ts` | - | 关闭步骤 4-12 |
| `src/startup/shutdown/sharedTeardown.ts` | - | 关闭步骤 13-19 |

## L6 Interface 接口层

### 核心

| 文件 | 职责 |
|------|------|
| `src/interface/core/app.ts` | Fastify 应用工厂（273 行，12 层中间件） |
| `src/interface/api/index.ts` | registerRoutes 路由注册（298 行） |

### 中间件

| 文件 | 职责 |
|------|------|
| `src/interface/middleware/traceId.ts` | TraceId 中间件（ALS 注入） |
| `src/interface/middleware/apiAuditLog.ts` | API 审计日志（599 行，中断捕获） |
| `src/interface/middleware/jwtAuth.ts` | JWT 认证（148 行） |
| `src/interface/middleware/permission.ts` | 权限检查（178 行，5 种守卫） |
| `src/interface/middleware/projectPermission.ts` | 项目权限 |
| `src/interface/middleware/errorHandler.ts` | 错误处理（311 行） |

### API 路由（90+ 文件，部分代表）

| 文件 | 职责 |
|------|------|
| `src/interface/api/auth.ts` | 认证路由（登录/登出/企微） |
| `src/interface/api/projects.ts` | 项目路由（含 5 个子路由） |
| `src/interface/api/agents/agentWebSocket.ts` | Agent 聊天 WebSocket |
| `src/interface/api/dashboardWebSocket.ts` | 仪表盘 WebSocket |
| `src/interface/api/externalAgentWebSocket.ts` | 外部 Agent WebSocket |

### Workers

| 文件 | 职责 |
|------|------|
| `src/interface/workers/scheduledTaskWorker.ts` | 定时任务 Worker |
| `src/interface/workers/memorySyncWorker.ts` | 记忆同步 Worker（45 行） |
| `src/interface/workers/kickoffJobWorker.ts` | Kickoff Worker（分布式锁） |
| `src/interface/workers/crossAgentTriggerWorker.ts` | 跨 Agent 触发 Worker |
| `src/interface/workers/roleInboxWorker.ts` | 角色收件箱 Worker |
| `src/interface/workers/tfsQueryExportWorker.ts` | TFS 导出 Worker |

### MCP

| 文件 | 职责 |
|------|------|
| `src/interface/mcp/mcpBridge.ts` | MCP Bridge Server |
| `src/interface/mcp/mcpBridgeAuth.ts` | Bearer Token 认证 |

## L2-4 Agents 层

### 核心（六层内核）

| 目录 | 职责 |
|------|------|
| `src/agents/core/ai-kernel/` | AI 内核（LLM、Prompt、Tool） |
| `src/agents/core/ai-execution/` | AI 执行层（67 文件） |
| `src/agents/core/agent/` | Agent（decision / turn / modes） |
| `src/agents/core/worker/` | Role / Action / runtime |
| `src/agents/core/kernel-management/` | 内核管理（config 治理） |

### 决策引擎

| 文件 | 职责 |
|------|------|
| `src/agents/core/agent/decision/DecisionEngine.ts` | 六阶段决策管线 |
| `src/agents/core/agent/decision/createDecisionPipeline.ts` | 管线工厂 |
| `src/agents/core/agent/decision/fusion-router.ts` | 融合路由算法 |
| `src/agents/core/agent/decision/route-registry.ts` | 路由规则注册表 |
| `src/agents/core/agent/decision/SemanticPlannerCache.ts` | 语义缓存 |
| `src/agents/core/agent/decision/stages/ExactRouter.ts` | Stage 2 |
| `src/agents/core/agent/decision/stages/DecisionPlanner.ts` | Stage 4（1090 行） |

### Turn 执行

| 文件 | 职责 |
|------|------|
| `src/agents/core/agent/turn/TurnRunner.ts` | Turn 编排 SSOT |
| `src/agents/core/agent/turn/TurnContext.ts` | 上下文加载 |
| `src/agents/core/agent/turn/turnRouting.ts` | 路由决策 |

### 执行模式

| 目录 | 职责 |
|------|------|
| `src/agents/core/agent/modes/cdw/` | Coordinator-Driven Workflow |
| `src/agents/core/agent/modes/interactive/` | 人机交互模式 |
| `src/agents/core/agent/modes/react/` | LLM 驱动循环（ReactRuntime） |

### 角色定义

| 文件 | 职责 |
|------|------|
| `src/agents/domain-harness/roles/OrchestratorRole.ts` | 大福（项目总指挥） |
| `src/agents/domain-harness/roles/ProcessManagerRole.ts` | 阿宁（过程管理） |
| `src/agents/domain-harness/roles/ProductDesignRole.ts` | 小品（产品设计） |
| `src/agents/domain-harness/roles/TechManagerRole.ts` | 阿码（技术架构） |
| `src/agents/domain-harness/roles/SreManagerRole.ts` | 大维（运维） |
| `src/agents/domain-harness/roles/QualityManagerRole.ts` | 小质（质量） |

## L4.5 Business-Tools 工具层（29 域）

| 文件 | 职责 |
|------|------|
| `src/business-tools/autoRegister.ts` | 自动注册（25 模块懒加载） |
| `src/business-tools/base/BaseTool.ts` | 工具基类 |
| `src/business-tools/base/interfaces.ts` | IToolRegistry 接口 |
| `src/business-tools/wecom-contact/index.ts` | 企微联系人（3 工具） |
| `src/business-tools/wecom-document/index.ts` | 企微文档（13 工具） |
| `src/business-tools/wecom-schedule/index.ts` | 企微日程（6 工具） |
| `src/business-tools/tfs/tfsQueryExportCore.ts` | TFS 查询导出 |

## L5 Business 业务层（30 子域）

### 核心域

| 文件 | 职责 |
|------|------|
| `src/business/domain/digitalEmployee/EmployeeService.ts` | 数字员工服务 |
| `src/business/domain/digitalEmployee/IDigitalEmployeeRepository.ts` | Port 接口 |
| `src/business/domain/project/ProjectServiceImpl.ts` | 项目服务 |
| `src/business/domain/rbac/PermissionService.ts` | RBAC 权限 |
| `src/business/domain/auth/AuthService.ts` | 认证服务 |

### 其他域

| 目录 | 职责 |
|------|------|
| `src/business/domain/conversation/` | 对话 |
| `src/business/domain/agentExecution/` | Agent 执行 |
| `src/business/domain/knowledgeBase/` | 知识库 |
| `src/business/domain/codingTask/` | 编码任务 |
| `src/business/domain/flowOrchestration/` | 流程编排（47 文件） |
| `src/business/domain/workstation/runtime/` | 工作站运行时 |
| `src/business/domain/externalAgent/` | 外部 Agent |
| `src/business/application/services/ScheduledTaskService.ts` | 定时任务服务 |

## L1 Infrastructure 基础设施层

### 持久化

| 文件 | 职责 |
|------|------|
| `src/infrastructure/persistence/prisma/client.ts` | Prisma Client（474 行，自动恢复） |
| `src/infrastructure/persistence/database/bullmqConnections.ts` | BullMQ 双连接 |
| `src/infrastructure/persistence/database/RedisConnectionManager.ts` | Redis 连接管理 |
| `src/infrastructure/persistence/advisoryLock.ts` | PG Advisory Lock |
| `src/infrastructure/persistence/distributedLock.ts` | Redis Kickoff Lock |
| `src/infrastructure/persistence/ExecutionPendingContextStore.ts` | CAS 乐观锁 |
| `src/infrastructure/persistence/repositories/` | 46 个仓储实现 |

### 缓存

| 文件 | 职责 |
|------|------|
| `src/infrastructure/cache/EntityCache.ts` | L1+Redis 双层缓存 |
| `src/infrastructure/cache/cacheInvalidationBus.ts` | 跨节点失效总线 |

### 认证

| 文件 | 职责 |
|------|------|
| `src/infrastructure/auth/JwtService.ts` | JWT 服务 |
| `src/infrastructure/auth/WeChatOAuthService.ts` | 企微 OAuth |

### 记忆与 RAG

| 文件 | 职责 |
|------|------|
| `src/infrastructure/memory/HybridSearch.ts` | 混合检索 |
| `src/infrastructure/memory/MemoryService.ts` | 记忆服务 |
| `src/infrastructure/memory/MemoryConsolidationService.ts` | 夜间整理 |
| `src/infrastructure/memory/syncSessionTranscriptsToMemory.ts` | 转录同步 |
| `src/infrastructure/rag/RagHybridSearch.ts` | RAG 混合检索 |
| `src/infrastructure/rag/DocumentChunker.ts` | 文档分块 |
| `src/infrastructure/rag/parsers/PdfParser.ts` | PDF 解析 |

### LLM 与工具

| 文件 | 职责 |
|------|------|
| `src/infrastructure/llm/core/types.ts` | ContentBlock 模型 |
| `src/infrastructure/mcp/McpManager.ts` | MCP 服务管理 |
| `src/infrastructure/mcp/McpToolAdapter.ts` | MCP 工具适配 |

### 可观测性

| 文件 | 职责 |
|------|------|
| `src/infrastructure/observability/traceContext.ts` | TraceId + ALS |
| `src/infrastructure/streaming/types.ts` | 流式事件类型 |
| `src/infrastructure/streaming/StreamingScope.ts` | 流式作用域 |

### DI

| 文件 | 职责 |
|------|------|
| `src/infrastructure/di/Container.ts` | DI 容器（211 行） |

### 沙箱

| 文件 | 职责 |
|------|------|
| `src/infrastructure/sandbox/index.ts` | 沙箱工厂 |
| `src/infrastructure/sandbox/config/workstationDefaults.ts` | 工作站默认配置 |
| `src/infrastructure/sandbox/command/remoteSandbox.ts` | 远程沙箱 |

## Integration 集成层

| 目录 | 职责 |
|------|------|
| `src/integration/connectors/external-agent/` | 外部 Agent（12 文件） |
| `src/integration/connectors/wechat/` | 微信连接器 |
| `src/integration/connectors/openclaw/` | OpenClaw |

## 跨层共享

| 文件 | 职责 |
|------|------|
| `src/types/errors.ts` | 错误类层次（360 行，20+ 子类） |
| `src/types/context.ts` | ProjectRef/UserRef/TargetRef 值对象 |
| `src/types/conversation.ts` | 对话消息类型 |
| `src/types/config.ts` | AgentConfig + Zod Schema |
| `src/types/services.ts` | Result\<T\> + ServiceRegistry |
| `src/types/wechat.ts` | 企微类型子集（接口隔离） |
| `src/shared/types/agent.ts` | IAgentConfigAdapter |
| `src/infrastructure/config/digitalEmployeeTeam.ts` | 七大角色配置 |

## 脚本与配置

| 文件 | 职责 |
|------|------|
| `scripts/check-layer-imports.cjs` | 分层依赖检查（183 行） |
| `scripts/check-agent-layer-imports.cjs` | Agent 六层检查（610 行） |
| `scripts/build-esbuild.cjs` | esbuild 构建 |
| `scripts/tsc-alias-build.cjs` | 路径别名替换 |
| `tsconfig.json` | TypeScript 配置（strict） |
| `knip.json` | 死代码检测 |
| `prisma/schema.prisma` | Prisma Schema（3057 行，121 模型） |

## Prisma 关键模型

| 模型 | 职责 |
|------|------|
| `DigitalEmployee` | 数字员工 |
| `projects` | 项目 |
| `agent_run` | Agent 运行 |
| `AgentExecutionLog` | 执行日志 |
| `conversation_histories` | 对话历史 |
| `knowledge_bases` | 知识库 |
| `memory_chunks` | 记忆分块 |
| `skill_artifact` | 技能产物 |
| `route_rule` | 路由规则 |
| `flow_template` / `flow_run` | 流程模板/运行 |
| `mcp_services` | MCP 服务 |
| `workstation_type_config` | 工作站配置 |

---

本导航覆盖 100+ 关键文件，按六层架构组织，便于读者快速定位源码。
