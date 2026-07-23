# 附录 A：术语表

本附录收录本书中出现的 55+ 个核心术语，按拼音排序。

## A

**AI Bot（AI 机器人）**
企业微信中的智能机器人，WinMatrix 通过 WeCom AI Bot 通道实现企微消息交互。

**Agent（智能体）**
能够自主感知、决策、行动的 AI 实体。WinMatrix 中指执行任务的数字员工运行时。

**AgentExecutionLog（Agent 执行日志）**
记录 Agent 执行过程的结构化日志，包含 trace ID、token 用量、耗时等。

**AgentRun（Agent 运行记录）**
一次完整的 Agent 执行记录，包含意图、分解、编排计划、状态等。

**ALS（AsyncLocalStorage）**
Node.js 的异步上下文传播机制，WinMatrix 用于 TraceId 和 StreamingScope 的跨异步边界传递。

**Architector（系统设计者）**
WinMatrix 的系统内部角色（architect），负责内核管理和系统设计，不参与用户任务调度。

## B

**BaseRole（角色基类）**
所有数字员工角色的抽象基类，定义人格（profile/goal/constraints）和动作注册接口。

**BullMQ**
基于 Redis 的 Node.js 任务队列库，WinMatrix 用于所有异步 Worker。

## C

**CAS（Compare-And-Swap）**
乐观锁机制，通过版本号字段实现无锁并发控制。

**CDW（Coordinator-Driven Workflow）**
协调器驱动的工作流，WinMatrix 的默认执行模式，支持多步编排。

**Claim 模式**
分布式任务认领模式，Worker 通过 claim_token 认领任务，支持租约过期重新认领。

**ContentBlock（内容块）**
LLM 消息的内容单元（TextBlock / ThinkingBlock / ToolCallBlock），支持交错式思考。

**CrossAgentTrigger（跨 Agent 触发）**
一个数字员工异步委派任务给另一个数字员工的机制。

## D

**DecisionEngine（决策引擎）**
六阶段渐进式决策管线，决定由谁、用什么方式、执行什么。

**DecisionPlanner（决策规划器）**
决策引擎第 4 阶段，使用 LLM 进行复杂决策（约 1258 行）。

**DIContainer（依赖注入容器）**
WinMatrix 自研的轻量 DI 容器（130 行），支持 Singleton/Transient 生命周期。

**DigitalEmployee（数字员工）**
WinMatrix 的核心抽象，拥有角色、技能、工具、记忆的虚拟团队成员。

**Domain Pack（领域包）**
项目类型的预置配置，包含角色、工作流、技能、提示词。

## E

**EntityCache（实体缓存）**
L1（进程内 Map）+ L3（Redis）双层缓存，支持跨节点 Pub/Sub 失效。

**ExactRouter（精确路由器）**
决策引擎第 2 阶段，处理 @mention、/slash、精确匹配。

**ExecutionSpan（执行跨度）**
可观测性的基本单元，记录一个操作的开始、结束、输入、输出。

## F

**Fail-Closed（失败关闭）**
安全原则，权限检查出错时拒绝访问（返回 false），而非放行。

**FusionRouter（融合路由器）**
决策引擎第 3 阶段，多信号加权融合（正则 0.9 + 意图 ±0.2/0.8 + 语义 0.6）。

**Flow（流程）**
可复用的多步骤工作流模板，支持版本管理和 DAG 执行。

## H

**Human-in-the-Loop（人机交互）**
需要人类介入的场景，通过 IHumanInteractionPort 实现，支持 5 种交互类型。

## I

**IHumanInteractionPort（人机交互端口）**
定义人机交互契约的接口，支持 clarification/approval/risk_confirmation/credential_request/acceptance。

**Interactive 模式**
人机交互执行模式，使用收件箱（Role Inbox）机制。

## J

**JWT（JSON Web Token）**
无状态认证令牌，WinMatrix 使用 HS256 算法 + Redis 黑名单实现撤销。

## K

**Kickoff（项目启动）**
项目初始化流程，包含 execute/apply 两阶段，使用分布式锁和补偿机制。

**Knip**
Node.js 死代码检测工具，扫描未使用的导出和依赖。

## L

**Lease/Claim（租约/认领）**
分布式任务调度模式，通过 claim_token 和过期时间实现至少一次执行。

**LLMInstance**
LLM 客户端实例，底层为 ILLMClient。

## M

**MCP（Model Context Protocol）**
连接 AI 与外部工具的开放协议。WinMatrix 双向支持（Bridge 对外 + Manager 消费）。

**MemorySource（记忆来源）**
三种记忆类型：memory（项目长期）/ session（会话转录）/ pmdoc（项目文档）。

**MMR（Maximal Marginal Relevance）**
检索结果多样性重排算法，平衡相关性和多样性。

## O

**Observe-Think-Act（观察-思考-行动）**
React 模式的核心循环，LLM 驱动的步骤迭代。

**OpenClaw**
一种外部 Agent，运行在专用工作站中，通过 MCP Bridge 访问 WinMatrix 工具。

## P

**PendingSpan（待处理跨度）**
API 审计日志的注册表条目，用于可靠捕获中断的请求。

**PipelineHook（管线钩子）**
决策管线的横切关注点，包含 Audit/StageTrace/Feedback/Progress/CapabilitySnapshot/DecisionEvent 六种。

**Port-Adapter（端口-适配器）**
依赖倒置模式的实现，Domain 层定义 Port，Infrastructure 层实现 Adapter。

**ProcessRole（进程角色）**
WIN_PROCESS_ROLE 环境变量，控制进程行为（api/scheduled/rag）。

## R

**RBAC（Role-Based Access Control）**
基于角色的访问控制，WinMatrix 的权限模型，项目级作用域。

**React 模式**
LLM 驱动的 worker step loop 执行模式，7 态状态机。

**ReAct（Reasoning + Acting）**
推理与行动交替的 AI 范式，WinMatrix React 模式的理论基础。

**Result\<T\>（结果类型）**
显式返回成功/失败的类型，替代 throw-catch 异常处理。

**RRF（Reciprocal Rank Fusion）**
倒数排名融合，基于排名（而非分数）合并多路检索结果。

## S

**SemanticPlannerCache（语义缓存）**
基于 embedding 最近邻的 LLM 输出缓存，相似度 ≥ 0.95 时复用。

**SimpleChatGuard（简单聊天守卫）**
决策引擎第 1 阶段，拦截简单问候，毫秒级响应。

**Single-Flight（单飞模式）**
并发控制模式，多个相同请求复用同一个 in-flight Promise。

**Skill（技能）**
数字员工执行具体任务的能力单元，提示词模板 + Agent 封装。

**StreamingScope（流式作用域）**
基于 AsyncLocalStorage 的流式事件传播上下文。

**StreamEvent（流式事件）**
全系统流式事件的统一信封，使用 namespace:action 命名。

## T

**Think-Act-Observe（思考-行动-观察）**
React 循环的三阶段，每步 LLM 决策 → 执行 → 观察结果。

**TDZ（Temporal Dead Zone）**
ESM 模块的暂时性死区，访问未初始化的导出会抛出 ReferenceError。

**ToolExecutionContext（工具执行上下文）**
工具执行时注入的完整上下文（员工、项目、权限、凭证、工作站、流式）。

**ToolRegistry（工具注册表）**
管理所有工具元数据和执行的注册表，支持工厂模式懒加载。

**TraceId（追踪 ID）**
全链路追踪标识，通过 ALS 跨异步边界传播。

**Turn（轮次）**
一次完整的用户交互周期，从消息接收到回复完成。

**TurnRunner（轮次执行器）**
Turn 执行的单一真实来源（SSOT），四阶段：load → admit → route → assemble。

## W

**WeCom（企业微信）**
企业级微信，WinMatrix 的主要外部集成渠道。

**WebSocket**
全双工通信协议，WinMatrix 用于流式输出、仪表盘、外部 Agent。

**WIN_PROCESS_ROLE**
进程角色环境变量，决定进程的启动行为和加载的模块。

## 其他

**55+ 条术语**覆盖全书核心概念。
