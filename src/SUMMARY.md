# 全书目录

## 前言

WinMatrix（翠花）是一个企业级 AI 数字员工协作平台。在这里，AI 不仅是工具，更是团队的一员 —— 数字员工拥有角色、技能、记忆和工具，与人类同事协同完成从项目启动到交付的全流程。本书从源码出发，系统剖析其后端架构设计、工程实践与设计哲学。

---

## 第一部分：基础架构

### [第 1 章：全景概览](part1-foundation/ch01-overview.md)
### [第 2 章：启动流程与进程模型](part1-foundation/ch02-bootstrap.md)
### [第 3 章：类型系统与代码组织](part1-foundation/ch03-type-system.md)

---

## 第二部分：基础设施层

### [第 4 章：数据库与持久化](part2-infrastructure/ch04-database.md)
### [第 5 章：缓存与消息队列](part2-infrastructure/ch05-cache-queue.md)
### [第 6 章：认证与授权](part2-infrastructure/ch06-auth.md)

---

## 第三部分：API 与实时通信

### [第 7 章：REST API 设计](part3-api/ch07-rest-api.md)
### [第 8 章：WebSocket 与流式通信](part3-api/ch08-websocket.md)

---

## 第四部分：Agent 系统

### [第 9 章：数字员工模型](part4-agent/ch09-digital-employee.md)
### [第 10 章：渐进式决策引擎](part4-agent/ch10-decision-engine.md)
### [第 11 章：Turn 执行引擎](part4-agent/ch11-turn-runner.md)
### [第 12 章：记忆系统](part4-agent/ch12-memory.md)

---

## 第五部分：技能与工具系统

### [第 13 章：技能架构](part5-skill-tool/ch13-skill-architecture.md)
### [第 14 章：工具执行系统](part5-skill-tool/ch14-tool-execution.md)
### [第 15 章：编码工作站](part5-skill-tool/ch15-coding-workstation.md)

---

## 第六部分：知识与 RAG

### [第 16 章：知识库系统](part6-knowledge/ch16-knowledge-base.md)
### [第 17 章：RAG Worker 与检索增强](part6-knowledge/ch17-rag.md)

---

## 第七部分：协作与编排

### [第 18 章：项目管理系统](part7-collaboration/ch18-project.md)
### [第 19 章：协作会话](part7-collaboration/ch19-collaboration.md)
### [第 20 章：流程编排](part7-collaboration/ch20-flow-orchestration.md)

---

## 第八部分：集成层

### [第 21 章：企业微信集成](part8-integration/ch21-wechat.md)
### [第 22 章：MCP 与外部 Agent](part8-integration/ch22-mcp.md)

---

## 第九部分：后台处理

### [第 23 章：Worker 系统](part9-background/ch23-workers.md)
### [第 24 章：定时任务系统](part9-background/ch24-scheduled-tasks.md)

---

## 第十部分：工程实践

### [第 25 章：可观测性](part10-engineering/ch25-observability.md)
### [第 26 章：构建与部署](part10-engineering/ch26-build-deploy.md)
### [第 27 章：测试策略](part10-engineering/ch27-testing.md)

---

## 第十一部分：设计哲学

### [第 28 章：设计模式提炼](part11-philosophy/ch28-design-patterns.md)
### [第 29 章：工程哲学](part11-philosophy/ch29-philosophy.md)

---

## 附录

### [附录 A：术语表](appendix/appendix-a-glossary.md)
### [附录 B：源码导航](appendix/appendix-b-source-map.md)

---

## 第十二部分：开发经验文集（70 篇）

### [文集首页：目录与阅读路线](articles/README.md)

### [把 LLM 包进一个 Turn：Agent 执行引擎的设计与踩坑](articles/01-turn-engine.md)
### [让 AI 自己决定做什么：渐进式决策引擎的三层边界](articles/02-decision-engine.md)
### [Agent 记忆系统的真实难题：不是 RAG，是遗忘](articles/03-memory-system.md)
### [流式输出这件事：从 LLM token 到前端逐字的完整链路](articles/04-streaming.md)
### [工具调用不止 function calling：企业级工具治理怎么做](articles/05-tool-governance.md)
### [技能（Skill）即数字员工的能力单元：从定义到治理全流程](articles/06-skill-system.md)
### [让多个 AI 员工协作：协作编排与"会干活"的团队](articles/07-multi-agent.md)
### [企业级 AI 的可观测性：ExecutionSpan 如何取代散落的日志](articles/08-observability.md)
### [我们踩过的坑：Prisma v7 时区、连接池风暴与 single-flight](articles/09-pitfalls-infra.md)
### [从"AI 助手"到"AI 员工"：我们做对和做错的几件事](articles/10-reflections.md)
### [把 LLM 关进 K8s Pod：编码工作站的三层镜像是怎么切出来的](articles/11-coding-workstation-three-layer.md)
### [一次编码任务的幂等与回调：迟到的 attempt 不会覆盖新状态](articles/12-coding-task-idempotency.md)
### [MCP 双向架构：一个平台怎么既当 Server 又当 Consumer](articles/13-mcp-bidirectional.md)
### [把 Claude Code / Hermes 变成虚拟数字员工：外部 Agent 怎么接入](articles/14-external-agent-bootstrap.md)
### [配置热更新：为什么我们用 PG LISTEN/NOTIFY 而不是 Redis Pub/Sub](articles/15-config-hot-reload.md)
### [定时任务系统：16 个系统任务的幂等与补偿](articles/16-scheduled-task-idempotency.md)
### [BullMQ 三档队列：按重量分流，而不是一个队列塞所有](articles/17-queue-weight-tiering.md)
### [构建与部署：dev / prod / k8s 四进程对齐](articles/18-build-deploy-four-process.md)
### [测试策略：为什么我们的 E2E 坚决不 mock 数据库](articles/19-e2e-no-mock.md)
### [知识库入库 pipeline：从 PDF 到向量分块的全链路](articles/20-rag-ingest-pipeline.md)
### [企业微信双轨接入：长连接 AiBot + Webhook](articles/21-wecom-dual-track.md)
### [PMDOC 项目容器：项目是协作的物理容器](articles/22-pmdoc-project-container.md)
### [幂等设计全景：同一个请求被执行两次，世界还能保持一致吗](articles/23-idempotency.md)
### [分布式锁分工：Redis SET NX + Lua 与 PG advisory lock 各管什么](articles/24-distributed-lock.md)
### [优雅降级：当 Redis、ES、BullMQ、LLM 分别挂掉时系统怎么活](articles/25-graceful-degradation.md)
### [并发控制三层模型：全局信号量、会话锁、任务租约各管一个粒度](articles/26-concurrency-control.md)
### [Context Engineering 实战：每轮该给 LLM 注入什么、注入多少](articles/27-context-engineering.md)
### [安全护栏多层防御：认证、授权、输入、执行、基础设施各一道墙](articles/28-security-guardrails.md)
### [LLM 成本治理：把每一分 token 归因到员工、技能、项目](articles/29-cost-governance.md)
### [热更新与零停机：改配置不重启、发技能不中断、灰度路由](articles/30-hot-reload.md)
### [WinMatrix vs Anthropic orchestrator-worker：角色优先 vs 大脑优先](articles/31-vs-orchestrator-worker.md)
### [编码工作站 vs K8s Agent Sandbox：自建沙箱 vs 社区标准](articles/32-vs-agent-sandbox.md)
### [渐进式决策 vs 纯 LLM 路由：为什么我们不学 AutoGPT](articles/33-vs-llm-routing.md)
### [ExecutionSpan vs Langfuse/LangSmith：自建可观测 vs 通用平台](articles/34-vs-observability-platforms.md)
### [企业级 AI 落地的五个层级：从模型到可观测](articles/35-five-layer-model.md)
### [悬挂的 LLM 调用：start 记了，end 永远不来](articles/36-llm-call-watchdog.md)
### [孤儿任务回收：进程崩了，running 状态谁来收敛](articles/37-orphan-task-reconcile.md)
### [Schema 漂移：代码发了但迁移没跑](articles/38-schema-drift.md)
### [WebSocket 半断开：连接没断但消息不再来](articles/39-ws-half-open.md)
### [全书源码之旅的总复盘：如果每个模块只记住一件事](articles/40-series-finale.md)
### [数字员工的"人格"是怎么来的：从 agent_config 到提示词](articles/41-agent-persona-config.md)
### [智能路由 route_rule：一条规则如何同时用正则、意图词、语义锚点](articles/42-route-rule-fusion-router.md)
### [语义分诊门 SemanticTriageGate：闲聊、终止、bound、unbound 怎么分](articles/43-semantic-triage-gate.md)
### [五种执行模式：interactive / coordinator / react / skill / workstation 怎么选](articles/44-execution-modes.md)
### [PromptOverride：同一个角色，不同项目/分身怎么定制提示词](articles/45-prompt-override.md)
### [会话执行调度器：一个会话里多条消息怎么排队](articles/46-conversation-execution-dispatcher.md)
### [数字员工的"专长画像"：SpecialistProfile 怎么驱动能力绑定](articles/47-specialist-profile.md)
### [协作催促 CollaborationFollowup：被同事卡住了，系统怎么催](articles/48-collaboration-followup.md)
### [Token 凭证体系：PAT / WMA / WMEC 三种令牌各保护什么](articles/49-token-broker.md)
### [ExecutionSpan 的事件双写：JSON 数组 + 子表，为什么要两份](articles/50-execution-span-events.md)
### [终态收敛：分布式系统里"未完成"比"失败"更危险](articles/51-terminal-convergence.md)
### [确定性优先：能用规则就别用 LLM](articles/52-determinism-first.md)
### [可重放性：一个 bug 能不能"按原样再跑一遍"](articles/53-replayability.md)
### [透明代理并非透明：PgBouncer/连接池/中间件的隐含状态](articles/54-transparent-proxy.md)
### [先建新后关旧：热替换的节奏](articles/55-create-before-destroy.md)
### [横切关注点别靠纪律：Proxy/中间件/Hook 收口](articles/56-cross-cutting-concerns.md)
### [重放 ≠ 永远安全：哪些操作能自动重试](articles/57-replay-safety.md)
### [配置即数据：把配置放数据库而不是代码/配置文件](articles/58-config-as-data.md)
### [多租户隔离：同一个平台，不同项目怎么真正隔开](articles/59-multi-tenancy-isolation.md)
### [凭证管理：密钥/Token 在系统里怎么流转才安全](articles/60-credential-management.md)
### [审计日志：谁在什么时候对配置做了什么](articles/61-audit-logging.md)
### [背压与限流：当 LLM 处理不过来时怎么办](articles/62-backpressure-rate-limiting.md)
### [版本化：技能/流程/镜像怎么管版本才能不串](articles/63-versioning.md)
### [优雅停机：进程收到 SIGTERM 后要做完哪些事才能退](articles/64-graceful-shutdown.md)
### [健康检查与就绪检查：liveness vs readiness 不是一回事](articles/65-health-readiness.md)
### [分层 import 门禁：check:agent-layers:strict 怎么守住六层](articles/66-layer-import-gate.md)
### [启动序列的坑：DI 注册顺序错了为什么直接崩](articles/67-startup-di-order.md)
### [cron 尖峰迁移：为什么 09:00 的任务要把它们"散开"](articles/68-cron-spread-migration.md)
### [PromptSection 漂移：提示词模板改了，旧会话怎么办](articles/69-prompt-section-drift.md)
### [全系列终章：做 AI 平台三年，我会告诉刚开始的自己什么](articles/70-three-years-of-ai-platform.md)
