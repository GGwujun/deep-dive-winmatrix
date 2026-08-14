# WinMatrix 开发经验文集

> 配套《深入理解 WinMatrix 源码》的开发经验总结，面向微信公众号阅读。每篇独立成文，不依赖 Mermaid 图与复杂排版（全部用 ASCII 图/表格），手机端友好。

这 70 篇文章从 WinMatrix（翠花）这个企业级 AI 数字员工协作平台的真实研发中提炼。**所有代码与工程细节均来自 WinMatrix 已核实源码**；行业对比类文章（31-35）只借用业界公开的"做法方向"作为对比视角，不引用、不搬运任何外部文字。

覆盖四个方向：

- **Agent / LLM 平台工程实战**——Turn 编排、决策引擎、工具调用、记忆、流式、编码工作站
- **企业级 AI 落地方法论**——数字员工、技能治理、协作编排、可观测性、成本治理、安全护栏
- **跨模块横切主题**——幂等、分布式锁、降级、并发、Context Engineering、热更新、终态收敛、确定性、可重放、配置即数据
- **行业对比 / 踩坑复盘**——对比业界方案 + 真实工程事故的根因与修复

---

## 第一批：核心十篇（01-10）

| # | 标题 | 方向 |
|---|------|------|
| 01 | [把 LLM 包进一个 Turn：Agent 执行引擎的设计与踩坑](./01-turn-engine.md) | 平台工程 |
| 02 | [让 AI 自己决定做什么：渐进式决策引擎的三层边界](./02-decision-engine.md) | 平台工程 |
| 03 | [Agent 记忆系统的真实难题：不是 RAG，是遗忘](./03-memory-system.md) | 平台工程 |
| 04 | [流式输出这件事：从 LLM token 到前端逐字的完整链路](./04-streaming.md) | 平台工程 |
| 05 | [工具调用不止 function calling：企业级工具治理怎么做](./05-tool-governance.md) | 平台工程 |
| 06 | [技能（Skill）即数字员工的能力单元](./06-skill-system.md) | 企业落地 |
| 07 | [让多个 AI 员工协作：协作编排与"会干活"的团队](./07-multi-agent.md) | 企业落地 |
| 08 | [企业级 AI 的可观测性：ExecutionSpan 如何取代散落的日志](./08-observability.md) | 企业落地 |
| 09 | [我们踩过的坑：Prisma v7 时区、连接池风暴与 single-flight](./09-pitfalls-infra.md) | 踩坑复盘 |
| 10 | [从"AI 助手"到"AI 员工"：做对和做错的几件事](./10-reflections.md) | 踩坑复盘 |

## 第二批：源码模块深挖（11-22）

| # | 标题 | 源码模块 |
|---|------|---------|
| 11 | [编码工作站：把 LLM 关进 K8s Pod 的三层镜像](./11-coding-workstation-three-layer.md) | ch15 |
| 12 | [一次编码任务的幂等与回调：迟到 attempt 不会覆盖新状态](./12-coding-task-idempotency.md) | CodingTask |
| 13 | [MCP 双向架构：既是 Server 又是 Consumer](./13-mcp-bidirectional.md) | ch22 |
| 14 | [外部 Agent 接入：把 Claude Code/Hermes 变成虚拟数字员工](./14-external-agent-bootstrap.md) | externalAgent |
| 15 | [配置热更新：为什么用 PG LISTEN/NOTIFY 而不是 Redis Pub/Sub](./15-config-hot-reload.md) | ConfigDbListener |
| 16 | [定时任务系统：16 个系统任务的幂等与补偿](./16-scheduled-task-idempotency.md) | ch24 |
| 17 | [BullMQ 三档队列：按重量分流而不是一个队列塞所有](./17-queue-weight-tiering.md) | queue |
| 18 | [构建与部署：dev/prod/k8s 四进程对齐](./18-build-deploy-four-process.md) | ch26 |
| 19 | [测试策略：为什么我们的 E2E 坚决不 mock 数据库](./19-e2e-no-mock.md) | ch27 |
| 20 | [知识库入库 pipeline：从 PDF 到向量分块的全链路](./20-rag-ingest-pipeline.md) | ch16 |
| 21 | [企业微信双轨接入：长连接 AiBot + Webhook](./21-wecom-dual-track.md) | ch21 |
| 22 | [PMDOC 项目容器：项目是协作的物理容器](./22-pmdoc-project-container.md) | ch18 |

## 第三批：跨模块横切主题（23-30）

| # | 标题 | 横切主题 |
|---|------|---------|
| 23 | [幂等设计全景：同一个请求执行两次，世界还能保持一致吗](./23-idempotency.md) | 幂等 5 形态 |
| 24 | [分布式锁：Redis SET NX+Lua vs PG advisory lock 的分工](./24-distributed-lock.md) | 分布式锁 |
| 25 | [优雅降级：当 Redis/ES/BullMQ/LLM 分别挂掉时系统怎么活](./25-graceful-degradation.md) | 降级 |
| 26 | [并发控制：信号量、租约、会话锁的三层并发模型](./26-concurrency-control.md) | 并发 |
| 27 | [Context Engineering 实战：每轮注入什么、注入多少](./27-context-engineering.md) | 上下文工程 |
| 28 | [安全护栏多层防御：认证/授权/输入/执行/基础设施](./28-security-guardrails.md) | 安全 |
| 29 | [LLM 成本治理：token 归因到员工/技能/项目](./29-cost-governance.md) | 成本 |
| 30 | [热更新与零停机：配置、技能、路由规则都能热加载](./30-hot-reload.md) | 热更新 |

## 第四批：行业对比与方法论（31-35）

| # | 标题 | 对比对象 |
|---|------|---------|
| 31 | [WinMatrix vs Anthropic orchestrator-worker：角色优先 vs 大脑优先](./31-vs-orchestrator-worker.md) | Anthropic 多 Agent |
| 32 | [编码工作站 vs K8s Agent Sandbox：自建沙箱 vs 社区标准](./32-vs-agent-sandbox.md) | K8s Agent Sandbox |
| 33 | [渐进式决策 vs 纯 LLM 路由：为什么我们不学 AutoGPT](./33-vs-llm-routing.md) | LangGraph/AutoGPT |
| 34 | [ExecutionSpan vs Langfuse/LangSmith：自建可观测 vs 通用平台](./34-vs-observability-platforms.md) | Langfuse/LangSmith |
| 35 | [企业级 AI 落地的五个层级：从模型到可观测](./35-five-layer-model.md) | 五层模型 |

## 第五批：踩坑复盘扩展（36-40）

| # | 标题 | 真实事故 |
|---|------|---------|
| 36 | [悬挂的 LLM 调用：start 记了 end 永不来](./36-llm-call-watchdog.md) | llmCallWatchdogSweeper |
| 37 | [孤儿任务回收：进程崩了，running 状态谁来收敛](./37-orphan-task-reconcile.md) | reconcileStaleRuns |
| 38 | [Schema 漂移：代码发了但迁移没跑](./38-schema-drift.md) | skillArtifactSchemaDrift |
| 39 | [WebSocket 半断开：连接没断但消息不再来](./39-ws-half-open.md) | ping/pong + connection:sync |
| 40 | [全书源码之旅的总复盘：一句话记住每个模块](./40-series-finale.md) | 系列总收尾 |

## 第六批：更细的源码剖析（41-50）

| # | 标题 | 子模块 |
|---|------|--------|
| 41 | [数字员工的人格是怎么来的：从 agent_config 到 prompt](./41-agent-persona-config.md) | agent_config/人格 |
| 42 | [智能路由 route_rule：正则、意图词、语义锚点怎么融合](./42-route-rule-fusion-router.md) | FusionRouter |
| 43 | [语义分诊门 SemanticTriageGate：闲聊/终止/bound 怎么分](./43-semantic-triage-gate.md) | SemanticTriageGate |
| 44 | [五种执行模式：interactive/coordinator/react/skill/workstation](./44-execution-modes.md) | modes/ |
| 45 | [PromptOverride：同角色在不同项目/分身怎么定制提示词](./45-prompt-override.md) | PromptOverrideService |
| 46 | [会话执行调度器：一个会话里多条消息怎么排队](./46-conversation-execution-dispatcher.md) | CoordinatorAdapter |
| 47 | [specialist profile：数字员工的专长画像](./47-specialist-profile.md) | SpecialistProfile |
| 48 | [协作催促 CollaborationFollowup：被同事卡住了系统怎么催](./48-collaboration-followup.md) | CollaborationFollowup |
| 49 | [Token 凭证体系：PAT/WMA/WMEC 三种令牌各保护什么](./49-token-broker.md) | tokenBroker |
| 50 | [ExecutionSpan 的事件双写：JSON 数组 + 子表，为什么要两份](./50-execution-span-events.md) | ExecutionSpanEvent |

## 第七批：更深的工程哲学（51-58）

| # | 标题 | 哲学 |
|---|------|------|
| 51 | [终态收敛：分布式系统里"未完成"比"失败"更危险](./51-terminal-convergence.md) | 终态收敛 |
| 52 | [确定性优先：能用规则就别用 LLM](./52-determinism-first.md) | 确定性 |
| 53 | [可重放性：一个 bug 能不能"按原样再跑一遍"](<./53-replayability.md>) | 可重放 |
| 54 | [透明代理并非透明：中间件的隐含状态](./54-transparent-proxy.md) | 伪透明 |
| 55 | [先建新后关旧：热替换的节奏](./55-create-before-destroy.md) | 热替换 |
| 56 | [横切关注点别靠纪律：Proxy/中间件/Hook 收口](./56-cross-cutting-concerns.md) | 横切收口 |
| 57 | [重放 ≠ 永远安全：哪些操作能自动重试](./57-replay-safety.md) | 重放安全 |
| 58 | [配置即数据：把配置放数据库而不是代码](./58-config-as-data.md) | 配置即数据 |

## 第八批：跨界主题（59-65）

| # | 标题 | 主题 |
|---|------|------|
| 59 | [多租户：同一个平台不同项目/团队怎么隔离](./59-multi-tenancy-isolation.md) | 多租户五维 |
| 60 | [凭证管理：密钥/Token 在系统里怎么流转才安全](./60-credential-management.md) | 凭证 |
| 61 | [审计日志：谁在什么时候对配置做了什么](./61-audit-logging.md) | 审计 |
| 62 | [背压与限流：当 LLM 处理不过来时怎么办](./62-backpressure-rate-limiting.md) | 背压 |
| 63 | [版本化：技能/流程/镜像/配置怎么管版本](./63-versioning.md) | 版本化 |
| 64 | [优雅停机：进程收到 SIGTERM 后要做完哪些事才能退](./64-graceful-shutdown.md) | 停机 |
| 65 | [健康检查与就绪检查：liveness vs readiness 不是一回事](./65-health-readiness.md) | 探针 |

## 第九批：延伸踩坑 + 全系列终章（66-70）

| # | 标题 | 主题 |
|---|------|------|
| 66 | [分层 import 门禁：check:agent-layers:strict 怎么守住六层](./66-layer-import-gate.md) | 架构门禁 |
| 67 | [启动序列的坑：DI 注册顺序错了为什么直接崩](./67-startup-di-order.md) | 启动断言 |
| 68 | [cron 尖峰迁移：为什么 09:00 的任务要把它们散开](./68-cron-spread-migration.md) | cron 打散 |
| 69 | [PromptSection 漂移：提示词模板改了旧会话怎么办](./69-prompt-section-drift.md) | prompt 版本 |
| 70 | [全系列终章：做 AI 平台三年，我会告诉刚开始的自己什么](./70-three-years-of-ai-platform.md) | 全系列完结 |

---

## 阅读建议

- **想做 Agent 平台的工程师**：01-05 → 11-12（编码工作站）→ 23-27（横切）→ 41-46（决策/执行细节）→ 08/29（可观测+成本）
- **想做企业 AI 产品的决策者**：10（反思）→ 06-07（技能+协作）→ 31-35（行业对比）→ 59-65（多租户/凭证/审计）→ 70（终章）
- **喜欢看踩坑的所有人**：09 → 36-39（踩坑系列）→ 66-69（延伸踩坑）→ 10
- **想看工程哲学**：51-58（终态收敛/确定性/可重放/透明代理…）→ 70
- **想快速建立全局认知**：70（全系列终章）→ 40（全书总复盘）→ 01-02（核心引擎）→ 35（五层模型）

每篇 2500-5000 字，配真实代码片段与 ASCII 图示，力求"读完能直接用"。文章间文末互链，可按编号顺序读，也可按兴趣跳读。全系列 70 篇，第 70 篇为全系列终章。

## License

CC BY-NC-SA 4.0
