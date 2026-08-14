# 第三批 30 篇选题表（41-70）

> 续编自 01-40。前 40 篇已较密地覆盖了核心模块与横切主题，这一批往**更细的源码剖析 + 更深的工程哲学 + 更多的跨界主题 + 延伸踩坑**方向挖，刻意避开与前 40 篇的重叠。
>
> 代码与工程细节均来自 WinMatrix 已核实源码（`.verification/`）；行业对比只借业界"做法方向"作视角，不引外部文字。

## 一、更细的源码剖析（41-50，深挖前 40 篇没单独展开的子模块）

| # | 标题 | 源码 | 核心论点 |
|---|------|------|---------|
| 41 | 数字员工的"人格"是怎么来的：从 agent_config 到 prompt | agent_config / AgentConfig / PromptSection | 角色=身份(profile/goal/constraints)+人格(personality)+原则(principles)+prompt override；配置驱动人格，热重载 |
| 42 | 智能路由 route_rule：一条规则如何同时用正则、意图词、语义锚点 | route_rule 模型 + FusionRouter | patterns/positiveIntents/negativeIntents/semanticAnchors/semanticThreshold 多信号融合；active vs shadow 灰度 |
| 43 | 语义分诊门 SemanticTriageGate：闲聊、终止、bound、unbound 怎么分 | SemanticTriageGate | casual_chat 短路、termination_intent 识别、bound/unbound + 四种 boundIntent；LOW_CONFIDENCE_THRESHOLD=0.65 宁可放过 |
| 44 | 五种执行模式：interactive / coordinator / react / skill / workstation 怎么选 | agents/core/agent/modes/ | 模式不是花哨，是"不同任务用不同执行语义"；各自的状态机和终止条件 |
| 45 | PromptOverride：同一个角色，不同项目/分身怎么定制提示词 | PromptOverrideService + agent_prompt_template | 60s Redis 缓存、role_supplement_note、分身差异化不靠平行实现靠 override |
| 46 | 会话执行调度器：一个会话里多条消息怎么排队 | conversationExecutionDispatcher + CoordinatorAdapter | sessionMode 选 adapter、内存队列串行、排队位置通知、缓冲消息回放 |
| 47 | specialist profile：数字员工的"专长画像" | SpecialistProfile + profiles/ | orchestration/techManagement/sre/qualityAssurance 等岗位画像；画像如何驱动能力绑定 |
| 48 | 协作催促 CollaborationFollowup：被同事卡住了，系统怎么催 | CollaborationFollowup + role_inbox | 阻塞检测→延时任务→bullJobId→触发→完成全生命周期；催促也是一种可观测事件 |
| 49 | Token 使用凭证体系：PAT / WMA / WMEC 三种令牌各保护什么 | tokenBroker + personal_access_tokens | 人-项目(PAT)/外部Agent注册(WMA)/外部接入方(WMEC)；前缀路由、hash 不存明文、强制绑定项目 |
| 50 | ExecutionSpan 的事件双写：JSON 数组 + 子表，为什么要两份 | ExecutionSpan + ExecutionSpanEvent | events JSON 快读、子表是完整事件源、[spanId,seq] 幂等；双写的一致性边界 |

## 二、更深的工程哲学（51-58，从源码提炼"道"）

| # | 标题 | 涉及 | 核心论点 |
|---|------|------|---------|
| 51 | 终态收敛：分布式系统里"未完成"比"失败"更危险 | reconcile + watchdog + outbox | running 是个毒状态；所有长流程都要有"强制收敛到终态"的兜底；partial 也要收敛 |
| 52 | 确定性优先：能用规则就别用 LLM | 决策引擎 + FusionRouter + 闲聊守卫 | 确定性=可测试/可复现/零成本；LLM 是兜底；"什么时候该用 LLM"的判断标准 |
| 53 | 可重放性：一个 bug 能不能"按原样再跑一遍" | decision-replay fixture + ExecutionSpan + session_transcript | 可重放是调试的前提；span 树+transcript+事件 seq 让任意一次执行可回放 |
| 54 | 透明代理并非透明：PgBouncer/连接池/中间件的隐含状态 | Prisma client + ConfigDbListener + pgbouncer | LISTEN/NOTIFY 不能走 transaction-pool；时区/会话语义被中间件悄悄改；识别隐含状态 |
| 55 | 先建新后关旧：热替换的节奏 | rebuildPrismaResources + 滚动部署 | 重建期始终有可用实例；关旧要在建新之后；切换窗口期的无服务容忍 |
| 56 | 横切关注点别靠纪律：Proxy/中间件/Hook 收口 | Prisma Proxy + Fastify hook + pipeline hooks | 纪律守不住；横切逻辑必须对业务代码透明（调用方假装它不存在） |
| 57 | 重放 ≠ 永远安全：哪些操作能自动重试 | shouldReplayAfterPrismaRebuild + DelayedError | 只读可重放、写操作要看幂等性；重试白名单是防"把瞬时错换成数据错" |
| 58 | 配置即数据：把配置放数据库而不是代码/配置文件 | agent_config/skill_artifact/tool_config/route_rule + ConfigDbListener | 配置入库=热更新/审计/版本/治理；代价是配置漂移，要用三层防御兜住 |

## 三、跨界主题（59-65，从 AI 平台延伸到更广的工程话题）

| # | 标题 | 涉及 | 核心论点 |
|---|------|------|---------|
| 59 | 多租户：同一个平台，不同项目/团队怎么隔离 | project_id 维度 + mcp_services + tool_policy + memory agent_id | 数据隔离/工具隔离/记忆隔离/队列隔离；综合才是多租户 |
| 60 | 凭证管理：密钥/Token 在系统里怎么流转才安全 | credentialBinding + secretCiphertext + aes-256-gcm + hash | 需还原用加密、需验证用 hash；凭证绑定到技能/项目；L3 凭证解析硬约束 |
| 61 | 审计日志：谁在什么时候对配置做了什么 | ConfigAuditLog + apiAuditLog + ConfigDbListener | 写操作审计、配置变更审计、配置变更触发 notify；审计是合规底线 |
| 62 | 背压与限流：当 LLM 处理不过来时怎么办 | rateLimit + agentChatLimiter + scheduledAgentSem + moveToDelayed | 三层限流（全局/对话/会话）+ 全局信号量；超载时延迟重投而非丢弃 |
| 63 | 版本化：技能/流程/镜像/配置怎么管版本 | skill_artifact version + flow_template_version checksum + engine_image | 不可变发布版、checksum、digest；版本是可复现/可回滚/可审计的基础 |
| 64 | 优雅停机：进程收到 SIGTERM 后要做完哪些事才能退 | shutdown 序列 + binding 释放 + sweep flush | 释放连接/归还租约/flush pending/拒绝新请求；停机窗口期的任务怎么处理 |
| 65 | 健康检查与就绪检查：liveness vs readiness 不是一回事 | k8s liveness/readiness + /health /readyz + markBullmqReady | liveness 决定重启、readiness 决定摘流；fatal 靠应用自退出而非探针 |

## 四、延伸踩坑 + 工具链（66-70）

| # | 标题 | 涉及 | 核心论点 |
|---|------|------|---------|
| 66 | 分层 import 门禁：check:agent-layers:strict 怎么守住六层 | check-agent-layer-imports.cjs + check:layers | 存量 allowlist + 增量零容忍；L3↔L4 横向隔离、channel↔connectors 横向隔离；门禁是架构防腐的硬约束 |
| 67 | 启动序列的坑：DI 注册顺序错了为什么直接崩 | startup/common.ts + assertCoreDiRegistrations + adapter 数量断言 | Port 注入有顺序依赖、adapter 必须 4 个否则抛错、fail-fast 启动；启动期断言的价值 |
| 68 | cron 尖峰迁移：为什么 09:00 的任务要把它们"散开" | ScheduledCronMigrationLog + spread-cron | 所有人都在整点跑任务→DB 尖峰；迁移日志幂等；把 cron 时间打散的工程 |
| 69 | PromptSection 漂移：提示词模板改了，旧会话怎么办 | agent_prompt_template + PromptSection + 热更新 | 提示词版本化、旧会话用新提示词的语义、prompt override 缓存失效 |
| 70 | 全系列终章：做 AI 平台三年，我会告诉刚开始的自己什么 | 全系列总结 | 跳出 WinMatrix，讲 40+30 篇里最普适的几条教训；给"刚开始做 AI 平台"的人的忠告；正式完结 01-70 |

## 风格约束（与前 40 篇完全一致）
- 每篇 2500-4500 字，独立成文，手机友好（无 mermaid，用 ASCII 图/表格）
- 真实源码引用（文件路径+行号），来自 `.verification/` 已核实报告
- 结构：引言 → 主体（源码+设计权衡+ASCII图/表格）→ "给后来者的总结"（或踩坑文的"教训"）→ 文末"下一篇"链接
- 中文，代码注释保留原文
- 行业对比只用角度，不引外部文字
- 文末互链：41→42→...→70；70 作为整个 01-70 文集的正式完结
