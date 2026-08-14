# 第二批 30 篇选题表（11-40）

> 续编自第一批 01-10。选题角度参考了业界公开讨论（Anthropic / LangChain / K8s Agent Sandbox / Datadog Langfuse 等），但**所有代码与工程细节均来自 WinMatrix 已核实源码**，不搬运任何外部文字。
>
> 四类重心：① 深挖未写透的源码模块 ② 跨模块横切主题 ③ 行业对比方法论 ④ 踩坑扩展。

## 一、深挖源码模块（11-22，每模块 1-2 篇）

| # | 标题 | 源码模块 | 核心论点 |
|---|------|---------|---------|
| 11 | 编码工作站：把 LLM 关进 K8s Pod 的三层镜像 | ch15 / coding_workstations + workstation_* | base→engine→component 三层镜像正交解耦，引擎与运行时分离；对比业界 K8s Agent Sandbox CRD |
| 12 | 一次编码任务的幂等与回调：迟到 attempt 不会覆盖新状态 | CodingTask / CodingTaskArtifact | (conversation,trigger,idempotencyKey) running 态去重 + attemptNo 防覆盖 + callbackTokenHash 鉴权 + partial unique index 复用 session |
| 13 | MCP 双向架构：既是 Server 又是 Consumer | ch22 / mcp_services + McpManager | 一个平台同时暴露内部工具和消费外部 MCP；Promise.allSettled 并行连接 + tool_whitelist/assigned_agents 多租户 |
| 14 | 外部 Agent 接入：把 Claude Code/Hermes 变成虚拟数字员工 | externalAgentBootstrap + ExternalAgentGateway | ext_ 前缀防冲突、统一 roleId、分布式 Owner 路由（本地直连 vs 跨实例 RPC）、不可达抛 503 |
| 15 | 配置热更新：为什么我们用 PG LISTEN/NOTIFY 而不是 Redis Pub/Sub | ConfigDbListener + notifyConfigChange | 独立 pg.Client + 500ms 防抖 + Map 去重；为什么必须直连 PG 不能走 PgBouncer transaction-pool |
| 16 | 定时任务系统：16 个系统任务的幂等与补偿 | ch24 / ScheduledTask* 4 模型 | ScheduledCronMigrationLog 09:00 尖峰迁移审计、reconcileStaleRunsOnBootstrap advisoryLock 串行化、Result Delivery outbox 解耦 |
| 17 | BullMQ 三档队列：按重量分流而不是一个队列塞所有 | queue.ts + queueRegistry + runtimeQueueIsolation | scheduled-agent/system/light 三档 + hostname 后缀防开发机串 + worker 禁 commandTimeout |
| 18 | 构建与部署：dev/prod/k8s 四进程对齐 | ch26 / Makefile + docker-compose + k8s | esbuild 主构建 + tsc 校类型；api/scheduled/rag/embedding 四进程；prod 警告"仅启动 API" |
| 19 | 测试策略：为什么我们的 E2E 坚决不 mock 数据库 | ch27 / vitest 4 project + tests/fixtures | unit 用占位串、E2E 加载 .env.test 不 mock 缺失即 exit(1)；真实事故回放 fixture |
| 20 | 知识库入库 pipeline：从 PDF 到向量分块的全链路 | ch16 / RagService + DocumentChunker | parse→chunk（parent-child 双模式）→双写 ES+PG；文档类型感知分隔符；ABSOLUTE_MAX_CHUNK_SIZE 防 API 超限 |
| 21 | 企业微信双轨接入：长连接 AiBot + Webhook | ch21 / WeComAiBotMessageBridge | WsFrame→WeChatMessage 复用 processMessage；discovered_wechat_chats 群发现；docid vs pad_id 区分 |
| 22 | PMDOC 项目容器：项目是协作的物理容器 | ch18 / projects + pmdocSharedStage | pmdocPath + teamtaskPath 双路径；00_共享/ 固定阶段目录；chokidar 仅监听子目录 |

## 二、跨模块横切主题（23-30，从多模块提炼通用经验）

| # | 标题 | 涉及模块 | 核心论点 |
|---|------|---------|---------|
| 23 | 幂等设计全景：从 role_inbox 到 CodingTask 到 flow_instruction | 多模块 | idempotency_key / attemptNo / claim_token / hash 去重——WinMatrix 里幂等的 5 种形态 |
| 24 | 分布式锁：Redis SET NX + Lua vs PG advisory lock 的分工 | distributedLock + advisoryLock | Redis 锁做运行时互斥、PG advisory lock 已退化为 no-op 仅启动期用；owner 校验防误释放 |
| 25 | 优雅降级：当 Redis/ES/BullMQ/LLM 分别挂掉时系统怎么活 | 多模块 | Redis→L1、ES→PG tsvector、BullMQ→自动重连、工作站→NoopSandbox、LLM→watchdog 补偿 |
| 26 | 并发控制：信号量、租约、会话锁的三层并发模型 | scheduledAgentSem + conversationRunLocks + claim lease | 全局信号量防 LLM 过载、会话锁防同会话并发、租约防任务重复处理 |
| 27 | Context Engineering 实战：每轮注入什么、注入多少 | ch12 记忆 + ProjectContextSource | 对比 LangChain write/select/compress/isolate；WinMatrix 的记忆 top-k=5 + 历史 10 + 工作上下文 10 + 可观测 snapshot |
| 28 | 安全护栏多层防御：认证/授权/输入/执行/基础设施 | ch06 + guardrails | 每层假设其他层失败；fail-closed vs fail-open；声明式操作授权防 prompt 注入 |
| 29 | LLM 成本治理：token 归因到员工/技能/项目 | ExecutionSpan + tokenInput/Output | per-agent token 归因（对比 Datadog/Zylos 模式）；mini 模型蒸馏省钱；语义缓存复用决策 |
| 30 | 热更新与零停机：配置、技能、路由规则都能热加载 | ConfigDbListener + SkillArtifactStore + route_rule | 改 DB 不重启；技能 artifact-store 热加载；route_rule shadow→active 灰度 |

## 三、行业对比与方法论（31-35）

| # | 标题 | 对比对象 | 核心论点 |
|---|------|---------|---------|
| 31 | WinMatrix vs Anthropic orchestrator-worker：角色优先 vs 大脑优先 | Anthropic 多 Agent research system | 数字员工角色边界 vs lead-subagent 编排；两种多 Agent 范式的取舍 |
| 32 | 编码工作站 vs K8s Agent Sandbox：自建沙箱 vs 社区标准 | kubernetes-sigs/agent-sandbox | Sandbox CRD / Kata / gVisor；WinMatrix 三层镜像 + 远程 sandbox-api 的取舍 |
| 33 | 渐进式决策 vs 纯 LLM 路由：为什么我们不学 AutoGPT | LangGraph / AutoGPT / Dify | 确定性优先于 LLM；分层短路 vs 全 LLM 图；成本与稳定性权衡 |
| 34 | ExecutionSpan vs Langfuse/LangSmith：自建可观测 vs 通用平台 | Langfuse / LangSmith / Datadog | 为什么退役 agent_execution_log 自建 SSOT；通用平台够不够用 |
| 35 | 企业级 AI 落地的五个层级：从模型到可观测 | Augment Code 五层模型 | 基础模型→编排→工具→治理→可观测，每一层的选型与自建边界 |

## 四、踩坑/复盘扩展（36-40）

| # | 标题 | 真实事故 | 核心论点 |
|---|------|---------|---------|
| 36 | 悬挂的 LLM 调用：start 记了 end 永不来 | llmCallWatchdogSweeper | 每 10min 扫超时调用补写 end + 级联 finalize + 补发失败回执；外部调用必有悬挂检测 |
| 37 | 孤儿任务回收：进程崩了，running 状态谁来收敛 | reconcileStaleRunsOnBootstrap + failTimedOutRunningTasks | 启动时 advisoryLock 串行化三路 reconcile；partial→终态收敛 |
| 38 | Schema 漂移：代码发了但迁移没跑 | skillArtifactSchemaDrift | 启动 information_schema 探测 + 运行期 Prisma P2021/P2022 识别 + HTTP 503 整体降级 |
| 39 | WebSocket 半断开：连接没断但消息不再来 | ws ping/pong + connection:sync | 协议层 ping/pong 检测半开；重连 sync 帧三步（鉴权→注册 push→读状态）防 terminal 竞态 |
| 40 | 全书源码之旅的总复盘：一句话记住每个模块 | 全书 | 29 章 + 附录的精华浓缩，每模块一句"如果只记住一件事"；系列总收尾 |

## 风格约束（与第一批一致）
- 每篇 2500-4500 字，独立成文，手机友好（不用 mermaid，用文字 + ASCII 图）
- 真实代码引用（文件路径 + 行号），来自 `.verification/` 已核实报告
- 结构：引言（点出痛点/对比）→ 主体（源码 + 设计权衡 + 必要的业界对比视角）→ 给后来者的总结 → 文末互链
- 中文，代码注释保留原文
- 业界对比只用于"角度和视角"，不引用他人文字
