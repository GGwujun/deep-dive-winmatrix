# 全书目录

## 前言

WinMatrix（翠花）是一个企业级 AI 数字员工协作平台。在这里，AI 不仅是工具，更是团队的一员 —— 数字员工拥有角色、技能、记忆和工具，与人类同事协同完成从项目启动到交付的全流程。本书从源码出发，系统剖析其后端架构设计、工程实践与设计哲学。

---

## 第一部分：基础架构

### [第 1 章：全景概览](part1-foundation/ch01-overview.md)
- 1.1 WinMatrix 是什么
- 1.2 技术栈选型：Fastify 5 + Prisma 7 + LangChain + BullMQ + ES + Neo4j
- 1.3 代码库全景：Monorepo 的组织哲学
- 1.4 六层架构：从 Interface 到 Infrastructure
- 1.5 核心数据流：一次对话的完整旅程

### [第 2 章：启动流程与进程模型](part1-foundation/ch02-bootstrap.md)
- 2.1 入口点：index.ts 与 api.ts 的双模启动
- 2.2 进程角色：WIN_PROCESS_ROLE 的拆分策略
- 2.3 Fastify 服务器初始化：插件注册顺序与依赖
- 2.4 中间件管线：12 层中间件的职责划分
- 2.5 启动性能：连接池预热与延迟初始化

### [第 3 章：类型系统与代码组织](part1-foundation/ch03-type-system.md)
- 3.1 TypeScript 严格模式 + ESM 模块
- 3.2 路径别名：tsc-alias 方案
- 3.3 分层依赖规则：跨层约束的设计
- 3.4 依赖倒置：Port-Adapter 模式
- 3.5 Zod Schema 在协议层的应用

---

## 第二部分：基础设施层

### [第 4 章：数据库与持久化](part2-infrastructure/ch04-database.md)
- 4.1 PostgreSQL + Prisma 7：121 个模型的组织哲学
- 4.2 核心领域模型：数字员工、项目、任务、对话
- 4.3 迁移管理：40+ 迁移文件的演进策略
- 4.4 连接池管理：pg driver + Prisma adapter
- 4.5 事务模式与并发控制

### [第 5 章：缓存与消息队列](part2-infrastructure/ch05-cache-queue.md)
- 5.1 Redis 连接管理：ioredis 配置与连接池
- 5.2 BullMQ 任务队列：8 种 Worker 的设计
- 5.3 队列拓扑与 Worker 生命周期
- 5.4 重试策略与死信处理
- 5.5 分布式锁与并发控制

### [第 6 章：认证与授权](part2-infrastructure/ch06-auth.md)
- 6.1 JWT 认证流程：JwtService + jwtAuth 中间件
- 6.2 多登录方式：密码 / 企业微信 OAuth / SSO / PAT
- 6.3 RBAC 权限模型：permission_definition + role_permission_binding
- 6.4 会话管理：@fastify/secure-session
- 6.5 权限检查管线：per-route 鉴权 + projectPermission

---

## 第三部分：API 与实时通信

### [第 7 章：REST API 设计](part3-api/ch07-rest-api.md)
- 7.1 路由组织：90+ 路由文件的领域划分
- 7.2 registerRoutes() 统一注册机制
- 7.3 请求验证与错误响应规范
- 7.4 API 审计日志：写操作自动记录到 ES
- 7.5 限流策略：@fastify/rate-limit

### [第 8 章：WebSocket 与流式通信](part3-api/ch08-websocket.md)
- 8.1 @fastify/websocket 集成
- 8.2 聊天流式协议：thinking / chunk / done / error
- 8.3 Dashboard 实时推送
- 8.4 外部 Agent 协作 WebSocket
- 8.5 连接管理：心跳、断线重连、状态机

---

## 第四部分：Agent 系统

### [第 9 章：数字员工模型](part4-agent/ch09-digital-employee.md)
- 9.1 DigitalEmployee 实体：角色、技能、工具、记忆
- 9.2 七大内置角色：大福 / 阿宁 / 小品 / 阿码 / 小质 / 大维 / Architector
- 9.3 Agent 配置系统：agent_config / prompt_template / tool / capability
- 9.4 Agent 运行时上下文

### [第 10 章：渐进式决策引擎](part4-agent/ch10-decision-engine.md)
- 10.1 五阶段决策管线概览
- 10.2 ExactRouter：精确路由匹配
- 10.3 FusionRouter：融合路由决策
- 10.4 QuickAcceptGate：快速接受门控
- 10.5 DecisionPlanner：LLM 辅助规划
- 10.6 DecisionCommitment：决策提交与执行
- 10.7 路由规则管理与自动发现

### [第 11 章：Turn 执行引擎](part4-agent/ch11-turn-runner.md)
- 11.1 TurnRunner：加载上下文 → 准入 → 路由 → 组装执行
- 11.2 Observe-Think-Act 循环
- 11.3 四种执行模式：interactive / coordinator / react / skill / workstation
- 11.4 Coordinator Driven Workflow (CDW)
- 11.5 Agent 执行记录与追踪
- 11.6 Human-in-the-Loop：人工介入机制

### [第 12 章：记忆系统](part4-agent/ch12-memory.md)
- 12.1 五层记忆架构概览
- 12.2 Working Memory：当前任务上下文
- 12.3 Session Memory：会话级记忆
- 12.4 Long-term Memory：持久化向量记忆
- 12.5 混合检索：向量 70% + 全文 30%
- 12.6 记忆同步：memorySyncWorker 异步提取
- 12.7 记忆治理：system-memory-tidy

---

## 第五部分：技能与工具系统

### [第 13 章：技能架构](part5-skill-tool/ch13-skill-architecture.md)
- 13.1 Skill 的本质：从文件到运行时
- 13.2 30+ 内置技能解析
- 13.3 技能前置元数据：provides / consumes 契约
- 13.4 技能就绪检查：L1 / L2 / L3 边界
- 13.5 技能治理：白名单、角色绑定、项目覆盖

### [第 14 章：工具执行系统](part5-skill-tool/ch14-tool-execution.md)
- 14.1 29 个业务工具域
- 14.2 工具上下文：ToolContext 的组成
- 14.3 工具执行管线：权限 → 验证 → 执行 → 结果
- 14.4 工具与 Agent 的绑定
- 14.5 MCP 工具：外部协议到内部工具

### [第 15 章：编码工作站](part5-skill-tool/ch15-coding-workstation.md)
- 15.1 CodingTask 模型与生命周期
- 15.2 Workstation 类型配置
- 15.3 Agent Engine 绑定
- 15.4 沙箱管理：Go sandbox-api + K8s
- 15.5 工件管理：CodingTaskArtifact

---

## 第六部分：知识与 RAG

### [第 16 章：知识库系统](part6-knowledge/ch16-knowledge-base.md)
- 16.1 知识库数据模型
- 16.2 文档摄入：PDF / Word / Excel / Markdown 解析
- 16.3 文本分块策略与 FAQ 提取
- 16.4 Embedding 服务
- 16.5 向量索引：Elasticsearch kNN + ChromaDB

### [第 17 章：RAG Worker 与检索增强](part6-knowledge/ch17-rag.md)
- 17.1 rag-worker 进程架构
- 17.2 混合检索策略
- 17.3 RAG 上下文注入
- 17.4 Embedding 缓存
- 17.5 TFS/Git 知识库同步

---

## 第七部分：协作与编排

### [第 18 章：项目管理系统](part7-collaboration/ch18-project.md)
- 18.1 项目领域模型
- 18.2 项目启动：kickoff 流程
- 18.3 工作项管理与 agent_run
- 18.4 项目级 LLM 配置覆盖
- 18.5 项目凭证管理

### [第 19 章：协作会话](part7-collaboration/ch19-collaboration.md)
- 19.1 多 Agent 协作：collaboration sessions
- 19.2 跨 Agent 触发：crossAgentTriggerWorker
- 19.3 会话生成：sub-session spawning
- 19.4 后续动作：follow-up actions
- 19.5 对话持久化

### [第 20 章：流程编排](part7-collaboration/ch20-flow-orchestration.md)
- 20.1 Flow 模板系统
- 20.2 Flow 运行引擎
- 20.3 编排指令
- 20.4 Flow 工件
- 20.5 DAG 可视化与执行

---

## 第八部分：集成层

### [第 21 章：企业微信集成](part8-integration/ch21-wechat.md)
- 21.1 企业微信 OAuth 认证
- 21.2 WeChat Work 消息通道
- 21.3 微信消息映射
- 21.4 外部发送

### [第 22 章：MCP 与外部 Agent](part8-integration/ch22-mcp.md)
- 22.1 MCP Server 实现
- 22.2 MCP Consumer
- 22.3 外部 Agent 注册
- 22.4 外部 Agent 通信协议
- 22.5 OpenClaw 集成

---

## 第九部分：后台处理

### [第 23 章：Worker 系统](part9-background/ch23-workers.md)
- 23.1 BullMQ Worker 架构：8 种 Worker 的职责
- 23.2 kickoffJobWorker / memorySyncWorker / roleInboxWorker
- 23.3 scheduledTaskWorker / crossAgentTriggerWorker
- 23.4 Worker 错误处理与重试

### [第 24 章：定时任务系统](part9-background/ch24-scheduled-tasks.md)
- 24.1 系统定时任务清单
- 24.2 每日任务：memory-tidy / log-cleanup / transcript-compact
- 24.3 高频任务：timeout-sweep / reminder-delivery / watchdog
- 24.4 route-rule-discovery：自动发现路由规则
- 24.5 用户自定义定时任务

---

## 第十部分：工程实践

### [第 25 章：可观测性](part10-engineering/ch25-observability.md)
- 25.1 执行 Span 系统
- 25.2 Agent 执行日志
- 25.3 统一观测规则
- 25.4 TraceId 中间件
- 25.5 API 审计日志

### [第 26 章：构建与部署](part10-engineering/ch26-build-deploy.md)
- 26.1 esbuild 构建管线
- 26.2 Docker 多阶段构建
- 26.3 Kubernetes 部署
- 26.4 Harbor + Tekton CI
- 26.5 进程拆分部署

### [第 27 章：测试策略](part10-engineering/ch27-testing.md)
- 27.1 Vitest 三级测试配置
- 27.2 单元测试
- 27.3 集成测试
- 27.4 E2E 测试
- 27.5 Knip 死代码检测

---

## 第十一部分：设计哲学

### [第 28 章：设计模式提炼](part11-philosophy/ch28-design-patterns.md)
- 28.1 分层架构与依赖倒置
- 28.2 进程角色拆分
- 28.3 Observe-Think-Act 循环
- 28.4 渐进式决策的工程权衡
- 28.5 五层记忆的认知架构
- 28.6 技能治理的全生命周期

### [第 29 章：工程哲学](part11-philosophy/ch29-philosophy.md)
- 29.1 数字员工隐喻
- 29.2 安全优先
- 29.3 可观测性文化
- 29.4 混合存储策略
- 29.5 生产级 AI 平台的工程法则

---

## 附录

### [附录 A：术语表](appendix/appendix-a-glossary.md)
### [附录 B：源码导航](appendix/appendix-b-source-map.md)
