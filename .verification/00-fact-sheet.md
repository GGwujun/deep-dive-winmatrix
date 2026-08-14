# 全书统一事实清单（源码核实后）

> 补强各章时，所有数字与事实以此为准。早期版本的多处口径不一致（如"123 模型""七大角色""五层记忆"）需统一修正。

## 数据库
- **Prisma model 总数：157 个**（grep -c "^model " 实测），非 123/121
- schema.prisma 总行数：4065 行
- generator：`prisma-client`（非 prisma-client-js），`previewFeatures=["partialIndexes"]`，output 到 `src/infrastructure/persistence/prisma/generated`
- datasource：`provider="postgresql"`
- **Prisma 模型已按表拆分为 `infrastructure/persistence/prisma/generated/models/<model>.ts`**，每表独立文件（schema.prisma 仍是 SSOT）

## 角色与员工
- **八大角色**：orchestrator(大福)、process_manager(阿宁)、product_design_manager(小品)、tech_manager(阿码)、quality_manager(小质)、sre_manager(大维)、architect(Architector)、external-agent(运行时动态创建)
- Role 基类与注册表：`agents/core/worker/role/`；七个业务 Role 实现类：`agents/domain-harness/roles/`；第八类 external-agent 运行时由 DigitalEmployee.ts 动态生成
- DigitalEmployee 是执行编排层，BaseRole 是能力定义层，RoleContext 是运行时数据容器（三层分离）

## 执行模式
- 五种执行模式：interactive（轮流协作）、coordinator（多步编排）、react（单 Agent 推理）、skill（直接技能执行）、workstation（沙箱编码）

## 决策引擎
- 5 阶段管线：ExactRouter+PlanExtraction → FusionRouter → QuickAcceptGate → DecisionPlanner(LLM) → DecisionCommitmentDeriver（在 DecisionEngine.ts，SSOT）
- 对外入口：architector/Architector.ts 单例（decide/decideRoute）
- 三层短路：SimpleChatGuard / ExplicitFlowOrchestration / QuickAcceptGate cacheLookup
- **L0/L1/L2/L3/Chat 是分层路由层（DecisionRouteLayer），与 5 阶段管线是两个不同维度**
  - L0=系统预决策 target；L1=候选能力快照；L2=异步 LayeredRouter；L3=readiness gate；Chat=闲聊

## Turn 与 Tool
- Turn 编排 SSOT：`agents/core/agent/turn/TurnRunner.ts`（对象字面量，非 class），单轨编排
- **LLM Tool 调用循环实际在 `agents/core/ai-execution/tool-loop/StreamingToolExecutor.ts`**（不在 Turn 目录）
- Tool 循环双终止：`iteration < maxIterations` AND `totalLlmRounds < hardCapLlmRounds`（= maxIterations + 20）

## 记忆
- 三层记忆：会话层（Redis conversation:{id} 最多 50 条 + PG conversation_histories）、转录层（PG session_transcript）、长期索引层（PG memory_chunks/memory_files + ES dense_vector）
- 三区检索 Zone1（当前会话 session，3 条，0.25）/ Zone2（项目 memory，3 条，0.5）/ Zone3（跨会话 session，1 条，0.8，仅前两区不足 2 条时触发）
- 会话→长期记忆：TranscriptSyncManager.markDirty → 防抖 10s → indexContent

## 知识/RAG
- **embedding_cache 整表已废弃**（generated model 标 `@deprecated 未再被代码层读取/写入`），`chroma_id` 字段也是 deprecated（历史 ChromaDB 迁出痕迹）。曾经用 `@@id([provider, model, hash])` 做去重缓存，现不再读写。向量化由 ES 或独立 embedding 微服务（embedding-server.ts 端口 8401）处理。
- 知识库三型一表：knowledge_bases.type=document|faq|tfs_git，配置全 Json
- 全 ES 混合检索（kNN 向量 + BM25），RRF 默认融合，查询改写默认 enableLLM:false

## 技能
- 技能契约 SSOT：`agents/domain-harness/schema/flowContractSchema.ts`（types/ 下的是 re-export）
- L1/L2/L3 readiness：L1=ProjectSkillScopeService（persona 范围，listAvailable 轻量）；L2=snapshot 软校验；L3=SkillReadinessGate.check（SSOT，硬拦）
- bundled skill 已 deprecate：引用 `scripts/seed-bundled-skills.ts:4` 头部注释（**非 CLAUDE.md，该原文在当前 CLAUDE.md 未找到**）
- SkillTrace / SkillExecGuide / SkillEscapeEvent：轨迹→蒸馏→执行指南闭环

## 工具
- **27 个工具域模块**（autoRegister.ts TOOL_MODULES）
- BaseTool：`business-tools/base/BaseTool.ts`；统一 ToolExecutionContext 替代参数注入
- LLM 工具调用双轨：stream()（对话）+ invokeNonStreaming()（工具调用，保证 tool_calls.arguments 完整）

## 工作站
- 三层镜像：base_image → engine_image(claude_code/codex/hermes/openclaw) → component
- 所有工作站走远程 sandbox-api（K8s Pod），非本地 docker
- type→engine→skill 配置树

## 协作与编排
- 会话三层分工：conversation_histories（读模型）/ session_transcript（LLM 上下文真源，含 tool/thinking trace）/ conversation_meta（权威元数据）
- role_inbox 持久收件箱：租约抢占 + 重试 + 幂等 + turn_id 关联；入队 PG 先写→BullMQ 投递
- 流程编排三层：template（草稿）→ template_version（不可变发布版，checksum）→ run（实例）
- 指令驱动按需编排：每条 instruction 拥有独立 flow_run，租约 + claim_token 控并发
- provides/consumes 显式契约：FlowSkillContractValidationService 强校验

## 集成
- 企微双轨接入：长连接 AiBot + Webhook
- 外部 Agent = 虚拟数字员工（id 加 ext_ 前缀，roleId='external-agent'）
- MCP 多 token 体系：PAT(wm_pat_，人-项目) / WMA(wma_，外部 Agent 注册) / WMEC(wmec_，外部接入方应用身份)，Token Broker 统一路由

## 后台
- 三进程分离：api / scheduled / rag，每个入口 assertProcessRole 守卫
- BullMQ 三队列：scheduled-agent（重）/ scheduled-system（DB/ES 维护）/ scheduled-light（轻量扫描）
- 运行时队列隔离：开发环境加 hostname 后缀，生产强制 prod
- 配置热更新走 PG LISTEN/NOTIFY（pg_notify('config_change')），独立 pg.Client + 500ms 防抖 + Map 去重，**不能走 PgBouncer transaction-pool**

## 可观测性
- **ExecutionSpan 是 retire-agent-execution-log 后的 SSOT**（agent_execution_log 已无 model 定义）
- Span Events 双写：execution_span.events JSON + execution_span_event 子表
- 三 Sink 路由：UnifiedObservabilityRule 决定每个 eventType 落 span/log/channel 组合

## 工程实践
- JWT：HS256，secret 强制 ≥32 字符，Redis 黑名单
- 三路 token 提取：Authorization Bearer / X-Auth-Token / query.token
- 双轨权限：静态矩阵（编译期）+ 动态 RBAC（DB + Redis 缓存 300s）
- Prisma Client：single-flight 重建 + 只读重放白名单 + 强制 TimeZone=UTC（规避 #28629）
- 分布式锁走 Redis（SET NX EX + Lua），PG advisory lock 已废弃

## 哲学
- 六层分层：Interface → Agents → Business-Tools → Business → Infrastructure；Agent 内部 6 子层
- import 门禁：check:layers（存量 allowlist）+ check:agent-layers:strict（增量零容忍）
- Port 注入：wireAgentBusinessPorts() 注入 17+ 业务 Port
- 时间统一真源 = Pod TZ：禁 toISOString() 流出、禁硬编码时区、Prisma DateTime 强制 @db.Timestamptz(6)
- 数字分身同源继承：新增能力默认对分身生效，排除须显式名单，禁止平行实现
