# 源码核实报告：协作/编排/集成（ch18-22）

源码根：`E:/winning/code/AI/winmatrix/winmatrix-server/src/`

> **重要**：本仓库 Prisma 模型已按表拆分为 `infrastructure/persistence/prisma/generated/models/<model>.ts`，每个文件独立。schema.prisma 仍是 SSOT 但 generated 目录是逐表文件。

## 主题1：项目管理（第18章）

### 关键文件
- projects 模型：`infrastructure/persistence/prisma/generated/models/projects.ts`
- tasks/teams/members 模型：同目录
- PMDOC 阶段常量：`business/domain/project/pmdocSharedStage.ts`
- PMDOC 路径工具：`business/domain/document/utils/pmdocPathUtils.ts`
- PMDOC RAG 索引：`infrastructure/rag/PmdocRagIndexer.ts`
- 项目上下文注入：`agents/core/ai-kernel/context/adapters/sources/ProjectContextSource.ts`
- 项目仓储：`infrastructure/persistence/repositories/ProjectRepositoryImpl.ts`

### 真实字段
- **projects**：`id, name, code, description, pmdocPath, teamtaskPath, createdAt, updatedAt, projectBackground, backgroundSource, templateId, integrationsOverrides, tags[], email, emailPassword`；关系：documents/flowTemplates/flowRuns/flowSchedules/flowResources/flowInstructionBatches/flowInstructions/domainInstance/knowledge_bases/tasks/teams/personalAccessTokens
- **tasks**：`id, projectId, taskId, taskName, ownerId, startDate, endDate, duration, status, completion, deliverable, description, taskPath, workDocPath, source, externalId, externalUrl, lastSyncedAt`；复合唯一 `projectId_taskId`
- **teams**：主键 `projectId`（与 projects 一对一），`members: string`
- **members**：`id, name, roles, status, permissions, projectId, wechatUserId, nickname, mentionName, isAgent, email, userId, digitalEmployeeId, wechatOpenUserId`；复合唯一 `name_projectId`

### 代码片段
PMDOC 共享阶段（`pmdocSharedStage.ts:1-22`）：
```ts
export const PMDOC_SHARED_STAGE_DIR = '00_共享';
export const PMDOC_MEMORY_DIR_RELATIVE = `${PMDOC_SHARED_STAGE_DIR}/memory`;
export const PMDOC_GUIDELINES_RELATIVE = `${PMDOC_SHARED_STAGE_DIR}/agentrule.md`;
export const PMDOC_FALLBACK_OTHER_RELATIVE = `${PMDOC_SHARED_STAGE_DIR}/其他`;
export const PMDOC_MEMBER_MEMORY_PREFIX = '成员记忆_';
export const PMDOC_PROJECT_MEMORY_FILENAME = '项目记忆.md';
```

项目上下文注入（`ProjectContextSource.ts:33-64`）：
```ts
export interface ProjectInfo {
  name?: string; phase?: string; members?: string[]; description?: string;
  body?: string;  // 预渲染的「项目上下文正文」
  completeness?: ProjectContextCompleteness; fetchFailureReason?: string;
}
export class ProjectContextSource implements ContextSource {
  readonly id = 'project_context';
  readonly priority = 4;
}
```

### 设计要点
1. 项目 = 双路径容器：`pmdocPath`（PMDOC 文档树）+ `teamtaskPath`（任务/团队数据根），强制非空。
2. PMDOC 固定阶段目录约定：`00_共享/` 为长期记忆专目录，下设 `memory/`、`agentrule.md`；chokidar 仅监听该子目录。
3. teams 与 projects 一对一、members 软归属（projectId 可空，支持全局数字员工）。
4. tasks 既有业务字段也有外部同步字段（source/externalId/externalUrl/lastSyncedAt），是外部 PM 系统镜像。
5. 项目上下文分层注入：结构化字段（budget 决策）vs 预渲染 body（拼入 prompt）。
6. templateId 关联项目模板，项目可由模板实例化。

## 主题2：协作会话（第19章）

### 关键文件
- conversation_histories / conversation_meta / session_transcript / agent_collaboration / CollaborationFollowup / role_inbox 模型
- Role 收件箱入队：`business/domain/agentExecution/RoleInboxEnqueueService.ts`
- 会话历史仓储：`infrastructure/persistence/repositories/ConversationHistoryRepositoryImpl.ts`
- Role 收件箱 worker：`interface/workers/roleInboxWorker.ts`

### 真实字段
- **conversation_histories**（读模型："LLM 上下文真源为 session_transcript"）：`id, conversationId, userId, projectId, role, content, metadata, roleId, digitalEmployeeName, parentConversationId, digitalEmployeeId, agent_id, agent_name, attachments, llmPurpose, llmPhase`
- **conversation_meta**（"会话元数据/持久上下文权威字段"）：`conversationId, userId, title, isPinned, isArchived, projectId, projectName, roleId, sessionMode, interactiveDigitalEmployeeId`
- **session_transcript**（"Canonical message store，统一真源"）：`id, session_key, entry_type, timestamp, role, content, tool_name, tool_call_id, tool_result, tool_success, thinking, conversation_id, message_id, llm_purpose, llm_phase`
- **agent_collaboration**：`id, agent_id, collaborator_agent_id, scenarios: string[]`
- **CollaborationFollowup**（"协作阻塞检测后创建的延时跟进任务"）：`conversationId, blockedByRoleId, blockedByEmployeeId, followUpMessage, reason, delayMinutes, bullJobId, status, projectId, scheduledAt, triggeredAt, completedAt`
- **role_inbox**（"Interactive role durable inbox"）：`event_id, role_id, digital_employee_id, conversation_id, source, event_type, payload, idempotency_key, status, claim_owner, claim_expires_at, retry_count, max_retries, turn_id, message_id`

### 代码片段
Role 收件箱入队（`RoleInboxEnqueueService.ts:59-90`）：
```ts
async enqueue(input: EnqueueRoleEventInput): Promise<EnqueueRoleEventResult> {
  const turnId = input.turnId?.trim() || `turn_${randomUUID()}`;
  const inputWithTurnId = { ...input, turnId };
  const routing = validateInteractiveRoleEventRouting(toRoutingProbeEvent(inputWithTurnId));
  if (!routing.ok) throw new Error(`interactive_runtime_routing_error:${routing.reason ?? 'pipeline_assignment'}`);
  const wasAgentIdle = input.conversationId ? await probeAgentIdle(input.roleId, input.conversationId) : false;
  try {
    const payload = normalizeRoleInboxEventPayloadForInteractiveEnvironment({...});
    record = await roleInboxRepository.insertQueuedEvent({ ...input, payload, turnId });
  } catch (error) {
    if (error instanceof RoleInboxDuplicateEventError) { record = error.existing; deduplicated = true; }
```

### 设计要点
1. 会话三层分工：conversation_histories（读模型）+ session_transcript（LLM 上下文真源，含 tool/thinking trace）+ conversation_meta（权威元数据）。
2. role_inbox 持久收件箱：租约抢占（claim_owner/claim_expires_at）+ 重试（retry_count/max_retries）+ 幂等（idempotency_key）+ 轮次关联（turn_id）。入队策略 PG 先写 → BullMQ 投递（OpenSpec §3.1）。
3. 入队前置三重校验：路由合法性 + Agent 空闲探测 + 负载规范化；命中幂等键返回 deduplicated=true。
4. 协作催促是延时任务（delayMinutes/bullJobId + 全生命周期时间戳）。
5. agent_collaboration 用 scenarios[] 描述协作场景。

## 主题3：流程编排（第20章）

### 关键文件
- flow_template / flow_template_version / flow_run / flow_step_run / flow_orchestration_instruction / flow_orchestration_instruction_batch / flow_artifact / flow_resource 模型
- 编排启动：`business/domain/flowOrchestration/FlowOrchestrationStartService.ts`
- 指令派发：`business/domain/flowOrchestration/FlowInstructionDispatchCoordinator.ts`
- 契约编译：`business/domain/flowOrchestration/FlowTemplateContractCompiler.ts`
- Skill 契约校验：`business/domain/flowOrchestration/FlowSkillContractValidationService.ts`
- 资源路径：`business/domain/flowOrchestration/FlowResourcePathService.ts`

> flowOrchestration/ 目录有 40+ 个 .ts 文件。

### 真实字段
- **flow_template**（草稿/root）：`projectId, name, description, category, status, ownerId, currentVersionId, draftDefinitionJson`
- **flow_template_version**（不可变发布版，带 checksum）：`templateId, version, definitionJson, inputSchemaJson, outputSchemaJson, publishedBy, publishedAt, checksum`
- **flow_run**（实例，关联 agent_run 审计）：`projectId, templateId, templateVersionId, agentRunId, triggerType, triggerPayload, status, startedBy, duplicateKey, metadata`
- **flow_orchestration_instruction**（"Each instruction owns an independent flow_run"）：`batchId, projectId, flowRunId, agentRunId, conversationId, demandId, workItemId, sequenceNo, status, instructionJson, instructionSchemaVersion, checksum, idempotencyKey, claimToken, claimedAt, claimExpiresAt, claimedBy, attempt, skipReason, failureCode, failureMessage`
- **flow_artifact**（"Secret-bearing content must stay outside metadata"）：`flowRunId, stepRunId, type, name, storageRef, url, metadata`
- **flow_resource**（"File bytes stay in project document/object storage"）：`projectId, templateId, templateVersionId, flowSlug, kind, displayName, flowRelativePath, storageRef, sourceDocumentRef, versionRef, checksum, size, deletedAt`

### 代码片段
指令派发租约抢占循环（`FlowInstructionDispatchCoordinator.ts:71-107`）：
```ts
async dispatchNext(params: FlowInstructionDispatchParams) {
  const batch = await this.instructionRepository.getBatch(params.batchId);
  const flowRunIds: string[] = [];
  let failedCount = 0;
  while (flowRunIds.length < Math.max(1, params.concurrencyLimit)) {
    const active = await this.instructionRepository.countActiveInstructionsByBatchId(params.batchId);
    if (active.data >= Math.max(1, params.concurrencyLimit)) break;
    const instruction = await this.instructionRepository.claimInstruction({
      batchId: params.batchId, claimedBy: params.userId,
      leaseUntil: new Date(Date.now() + (this.options.leaseMs ?? 30 * 60 * 1000)),
    });
    if (!instruction.data) break;
    const dispatched = await this.dispatchInstruction(instruction.data, {...});
    if (dispatched.success === false) {
      failedCount++;
      await this.instructionRepository.updateInstructionStatus(instruction.data.id, 'failed', {
        failureCode: dispatched.error.code, failureMessage: dispatched.error.message,
      });
      continue;
    }
    flowRunIds.push(dispatched.data);
  }
```

provides/consumes 契约快照（`FlowTemplateContractCompiler.ts:26-51`）：
```ts
export interface FlowTemplateContractStepSnapshot {
  stepId: string; name: string; required: boolean; enabled: boolean;
  dependsOn: string[];
  executionMode: FlowStepDefinition['execution']['mode'];
  targetRef?: string;
  consumes: Array<{ field; as; type; required; sourceScope; sourceStepId?; sourcePath }>;
  provides: Array<{ key; type; artifact; required }>;
  requiredEvidence: string[];
  fallbackStrategy?: string;
}
```

### 设计要点
1. 三层模型：template（草稿）→ template_version（不可变发布版，checksum）→ run（实例）。
2. 指令驱动按需编排：每条 instruction 拥有独立 flow_run，batchId 聚合、sequenceNo 定序；状态机 pending/claimed/running/completed/failed/skipped/cancelled/timed_out。
3. 并发受租约 + claim_token 控制：claimInstruction 设 leaseUntil（默认 30 分钟），countActiveInstructionsByBatchId 卡并发。
4. provides/consumes 是显式契约：FlowSkillContractValidationService 强校验，缺失报 missing_skill_provides/missing_skill_consumes。
5. Flow 资源与文档存储解耦：flow_resource 仅存元数据，字节留文档/对象存储；FlowResourcePathService 禁止 `..` 越狱。
6. 触发源可插拔：5 类（AgentTool/Api/Manual/ScheduledTask/Webhook），统一经 FlowStartEnvelopeSchema 校验。

## 主题4：企业微信（第21章）

### 关键文件
- wechat_chat_mappings / discovered_wechat_chats / WecomConversationWedocDoc 模型
- UserWecomAibotBinding（非独立模型，定义于 `infrastructure/persistence/repositories/UserChannelRegistrationRepository.ts:36-49`）
- AiBot 长连接桥接：`interface/channel/channels/wecom-aibot/aibot/WeComAiBotMessageBridge.ts`
- 企微 API 封装：`integration/connectors/wechat/`（Token/AppMessage/Contact/File/Media/Schedule/Wedoc 7 个服务）

### 真实字段
- **wechat_chat_mappings**：主键 chatId，与 projects 一对一；`chatId, projectId, webhookUrl, projectWebhookUrl`
- **discovered_wechat_chats**（"已发现但未配置的群聊"）：`chatId, projectId, lastSeenAt, firstMessage`
- **WecomConversationWedocDoc**（"create_doc 返回的 API docid；非浏览器 pad_id"）：`conversationId, docid, url, docName, docType, projectId, digitalEmployeeId, agentId`
- **UserWecomAibotBindingRecord**：`ownerUserId, botId, secretCiphertext, pairedUserid, status(configured/testing/paired/error/disabled), pairingCodeHash, pairingExpiresAt`

### 代码片段
AiBot 长连接桥接（`WeComAiBotMessageBridge.ts:33-74`）：
```ts
export type AiBotProcessedMessageHandler = (params: {
  digitalEmployeeId: string; botId: string; msgid: string; content: string;
  fromUserid: string; chatid?: string; chattype: 'single' | 'group'; reqId: string;
}) => Promise<void>;

export class WeComAiBotMessageBridge {
  async handleMessage(frame: WsFrame<BaseMessage>): Promise<void> {
    const body = frame.body;
    if (!body) return;
    const content = this.extractTextContent(body);
    await this.options.onProcessMessage({
      digitalEmployeeId: this.client.digitalEmployeeId,
      botId: body.aibotid, msgid: body.msgid, content,
      fromUserid: body.from.userid, chatid: body.chatid, chattype: body.chattype,
      reqId: frame.headers.req_id,
    });
  }
}
```

### 设计要点
1. 双轨接入：长连接 AiBot + Webhook。AiBot WsFrame 转统一 WeChatMessage 复用 processMessage 管线。
2. 群聊自动发现：discovered_wechat_chats 保存待配置群，UI 补全 chatId→projectId 映射。
3. 微文档登记区分 API docid 与浏览器 pad_id（保证可程序化访问）。
4. 绑定状态机五态 + 配对码流程 + secretCiphertext 加密。
5. 企微 API 分服务封装（7 个独立服务）。

## 主题5：MCP 与外部 Agent（第22章）

### 关键文件
- mcp_services / external_agent_registration / external_agent_computer / external_agent_activity_event / external_agent_pause / external_agent_reminder 模型
- MCP 管理器：`infrastructure/mcp/McpManager.ts`
- 外部 Agent 引导：`agents/core/agent/external-agent/externalAgentBootstrap.ts`（**该目录只有这一个文件**）
- 外部 Agent 连接器：`integration/connectors/external-agent/`（13 个文件，含 connection/distributed/health/security 子目录）
- 分布式网关：`integration/connectors/external-agent/distributed/ExternalAgentGateway.ts`

> 重要更正：`src/agents/core/agent/external-agent/` 目录只有一个文件 `externalAgentBootstrap.ts`。外部 Agent 连接器/RPC/网关逻辑在 `integration/connectors/external-agent/`。

### 真实字段
- **mcp_services**（"MCP 服务注册/外部工具扩展"）：`project_id, name, description, transport_type, url, api_key, headers, is_enabled, is_builtin, tool_whitelist, assigned_agents`
- **external_agent_registration**（"外部智能体注册（Hermes/OpenClaw 等接入）"）：`userId, agentType, name, capabilities, apiKeyHash, apiKeyEncrypted, isConnected, lastHeartbeatAt, status, computerId, lastSessionId, capabilityProfile, tools, endpoint, endpointToken, hermesEndpoint`
- **external_agent_computer**（"外部 Agent 运行所在的物理计算机"）：`userId, installationId, hostname, os, arch, daemonVersion, detectedRuntimes, supportedAgentTypes, isConnected`
- **external_agent_pause**（"项目级外部 Agent 熔断开关"）：`projectId, conversationId, externalAgentId, paused, reason, pausedBy`
- **external_agent_reminder**（"Agent 上报的未来提醒"）：`externalAgentId, userId, remindAt, title, body, delivered, deliveredAt`

### 代码片段
外部 Agent 转虚拟数字员工（`externalAgentBootstrap.ts:73-101`）：
```ts
const EXTERNAL_AGENT_ROLE_ID = 'external-agent';
export async function getExternalAgentsForUser(userId: string): Promise<DispatchableDigitalEmployee[]> {
  const registrations = await prisma.external_agent_registration.findMany({
    where: { userId, status: 'active', isConnected: true },
    orderBy: { createdAt: 'desc' },
  });
  for (const reg of registrations) {
    const reachable = connectionPool.isAgentReachable(reg.id) || reg.isConnected;
    result.push({
      id: `ext_${reg.id}`,
      roleId: EXTERNAL_AGENT_ROLE_ID,
      name: reg.name?.trim() || reg.agentType,
      isExternal: true,
      externalAgentId: reg.id,
      externalAgentStatus: reachable ? 'connected' : 'disconnected',
    });
  }
}
```

分布式 Owner 路由（`ExternalAgentGateway.ts:32-63`）：
```ts
async listSessions(agentId: string, opts = {}) {
  const owner = await this.ownerRegistry.getAgentOwner(agentId);
  if (!owner) {
    if (this.connectionPool.getConnection(agentId)?.isConnected)
      return this.sessionQuery.listSessionsLocal(agentId, opts);
    throw unavailable(`owner_not_found: agentId=${agentId}`);
  }
  if (owner.instanceId === this.instanceId)
    return this.sessionQuery.listSessionsLocal(agentId, opts);
  const response = await this.rpcBus.call(owner.instanceId, {
    op: 'agent.session.list', agentId, payload: opts,
  });
  return responseToResult<{ sessions; total? }>(response);
}
```

MCP 初始化（`McpManager.ts:50-96`）：
```ts
async initialize(): Promise<void> {
  if (this.initialized) return;
  const services = await prisma.mcp_services.findMany({ where: { is_enabled: true } });
  await Promise.allSettled(services.map(svc => this.connectService(svc)));
  this.initialized = true;
  logger.info(`[${LOG_TAG}] 已初始化 ${this.clients.size}/${services.length} 个 MCP 服务`);
}
```

### 设计要点
1. 外部 Agent = 虚拟数字员工：转 DispatchableDigitalEmployee，id 加 `ext_` 前缀防冲突，roleId='external-agent'，进调度决策层。
2. 多 Agent 类型接入（Hermes/OpenClaw），agentType 区分，endpoint/hermesEndpoint/endpointToken 支持不同协议。
3. 物理计算机抽象 + 项目级熔断（external_agent_pause）。
4. MCP 服务支持热加载与多租户（project_id/tool_whitelist/assigned_agents/credential_binding），McpManager 并行连接（Promise.allSettled）。
5. 分布式 Owner 路由：本地直连 vs 跨实例 RPC，对调用方透明；不可达抛 503 unavailable。
6. 外部 Agent 生命周期全观测（activity_event 时间线 + reminder + health/audit）。
