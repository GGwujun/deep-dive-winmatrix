# 源码核实报告：技能/工具/知识/RAG（ch13-17）

源码根：`E:/winning/code/AI/winmatrix/winmatrix-server/`（src/ 前缀相对 winmatrix-server/）

## 主题1：技能架构（第13章）

### 关键文件
- skill_artifact 模型：`prisma/schema.prisma:1273`
- skill 契约 SSOT：`agents/domain-harness/schema/flowContractSchema.ts`
- 漂移检测：`business/domain/skillManagement/skillArtifactSchemaDrift.ts`
- Artifact Store（真源）：`business/domain/skillManagement/SkillArtifactStore.ts`
- bundled seed（已 deprecate）：`scripts/seed-bundled-skills.ts` + `business/domain/skillManagement/BundledSkillArtifactSeeder.ts`
- L3 readiness gate：`agents/harness/capability/skillGovernance/SkillReadinessGate.ts`
- L3 预检编排：`agents/harness/capability/skillGovernance/SkillPreflightOrchestrator.ts`
- L1/L2 scope：`agents/harness/capability/skillGovernance/ProjectSkillScopeService.ts`
- 轨迹提取：`agents/harness/learning/distillation/SkillTraceExtractor.ts`
- 蒸馏 worker：`agents/harness/learning/distillation/consumers/distillConsumer.ts`
- 蒸馏器：`agents/harness/learning/distillation/SkillKnowledgeDistiller.ts`

### 核心导出
- `prisma/schema.prisma:1273-1305` `model skill_artifact` — `name/version/scope/project_name/trust_level/manifest/persona_eligible/package_storage_key/package_sha256`；唯一约束 `@@unique([name, version, scope, project_name])`（第 1299 行）
- `skillArtifactSchemaDrift.ts:56-96` `warnSkillArtifactSchemaDriftOnStartup()`
- `flowContractSchema.ts:24-41` `FlowOutputContractBaseSchema`（provides[]）；`:43-54` `FlowInputBindingSchema`（onMissing: 'block'|'manual'|'skip'|'use_default'）
- `SkillReadinessGate.ts:92-343` `class SkillReadinessGate`，`:102` `async check(input)`
- `SkillPreflightOrchestrator.ts:120-160` — run() 按 effectiveExecutionMode 分发到 SkillExecutionPreflight（server）或 WorkstationSkillPreflight（workstation）
- `distillConsumer.ts:83-110` `startDistillWorker()` — BullMQ，DISTILL_QUEUE，concurrency=1
- `SkillKnowledgeDistiller.ts:46-139` — distill()，用 mini 模型 purpose='skill_distill'

### 代码片段
skill_artifact 唯一约束 + persona_eligible（`schema.prisma:1296-1304`）：
```prisma
  /// 分身技能可用性真源（design D3.1/D3.2）：默认 true=默认继承；false=不进 personal L1
  persona_eligible      Boolean  @default(true) @map("persona_eligible")
  @@unique([name, version, scope, project_name])
  @@index([scope, project_name])
  @@index([name])
  @@index([package_storage_key])
  @@index([persona_eligible])
  @@map("skill_artifact")
```

启动期漂移检测（`skillArtifactSchemaDrift.ts:56-83`）：
```ts
export async function warnSkillArtifactSchemaDriftOnStartup(): Promise<void> {
  const rows = await prisma.$queryRaw<Array<{ column_name: string }>>`
    SELECT column_name FROM information_schema.columns
    WHERE table_schema = 'public' AND table_name = 'skill_artifact'
      AND column_name IN ('project_name','trust_level','manifest','enabled','installed_at','created_at','updated_at')`;
  const present = new Set(rows.map((row) => row.column_name));
  const missing = SKILL_ARTIFACT_REQUIRED_COLUMNS.filter((column) => !present.has(column));
  if (missing.length === 0) return;
  logger.warn({ event: 'skill_artifact_schema_drift', missingColumns: missing },
    `[Startup] skill_artifact 缺列 (${missing.join(', ')}); skill 相关 API 将返回 503，请执行 npx prisma migrate deploy`);
```

蒸馏 worker（`distillConsumer.ts:22-70`）：
```ts
const traceRows = await prisma.skillTrace.findMany({
  where: { roleId: data.roleId, skillName: data.skillName, skillSource: data.skillSource, success: true },
  orderBy: { createdAt: 'desc' }, take: 10,
});
const registry = skillRegistry;
const resolved = await registry.resolve(data.roleId, data.skillName);
if (resolved?.content?.content) { skillContent = resolved.content.content; skillContentHash = computeSkillContentHash(skillContent); }
const guide = await distiller.distill({
  skillContent, traces, visibleToolNames,
  roleId: data.roleId, skillName: data.skillName, skillSource: data.skillSource,
  skillContentHash, toolSetHash: traceRows[0]?.toolSetHash ?? undefined, traceDurationsMs,
});
```

### SkillTrace/SkillExecGuide/SkillEscapeEvent
- **SkillTrace**（L2588-2609）：`roleId/skillName/skillSource/sessionId/success/iterations/toolCalls(Json)/totalDuration/skillContentHash/toolSetHash`；索引 `[roleId, skillName, skillSource]`
- **SkillExecGuide**（L2612-2651）：`coreTools/optionalTools/executionGuide(@db.Text)/paramRules/dataFlow/pitfalls` + 置信度与时长统计（confidence/hitCount/totalExecutions/avgDurationMs/p50DurationMs/p95DurationMs）+ 消毒审计（sanitizedAt/sanitizationDetail/removedTools/wasFiltered）+ 内容指纹（skillContentHash/toolSetHash）；唯一 `[roleId, skillName, skillSource]`
- **SkillEscapeEvent**（L2654-2671）：触发于 request_tool_expansion；`reason/toolCountBefore/toolCountAfter/llmContext`

### L1/L2/L3 真实位置与语义
- **L1（available skill 列表，轻量）**：`SkillRegistry.ts:256`「L1 决策摘要专用：仅加载 SKILL 正文，不做 binding/preflight/runtime intent」；`ProjectSkillScopeService.ts:89-101` `getPersonaEligibleSkillNames()` 从 skill_artifact 读 `enabled ∩ persona_eligible=true`（注释明令"禁止复用 skill_artifact.scope"）
- **L2（snapshot，跳过 listAvailable I/O）**：`skillTargetPlanningAudit.ts:60`「L2 snapshot-only」
- **L3（运行时 readiness，硬拦）**：`SkillReadinessGate.ts` 全文 + `SkillPreflightOrchestrator.ts:118`；`types.ts:85-88`「向 Skill Action 注入的秘密，由 L3 readiness 解析；在 L3 直接拒绝，不 fallback、不按名称/关键词推断」

### 设计要点
1. SSOT 分层：`types/domain-harness/schema/flowContractSchema.ts` 仅 re-export `agents/domain-harness/schema/flowContractSchema.ts`；跨层依赖单向。
2. provides/consumes 契约可被覆盖但不改正文：`skill_contract_override`（schema:1328-1344，provides_override/consumes_override/mode=merge）。
3. **bundled skill 已 deprecate，但 seed 脚本仍走 artifact-store**：`scripts/seed-bundled-skills.ts:4`「显式将 repo bundled skills 写入 artifact-store 与 DB bindings。不进 API 启动链；空库/发版运维使用」。**注意：CLAUDE.md 中"bundled skill 已 deprecated"原文在当前 CLAUDE.md 未找到，应引用此脚本头部注释。**
4. 漂移检测三层：启动期（information_schema 查列）、运行期（识别 Prisma P2021/P2022）、HTTP 503（sendSkillArtifactSchemaDriftReply）。
5. 轨迹→蒸馏→执行指南闭环：SkillTraceExtractor 从 session_transcript（entry_type='tool_call'+tool_result，按 runId 边界）重建 SkillTraceData，去重写 skillTrace 表；distillConsumer 取最近 10 条成功轨迹调 distiller，产 SkillExecGuide 经 SkillGuideSanitizer 消毒后写 cache。
6. L3 凭证解析硬约束：manifest 声明 requiresCredentials 时，canonical 字段任一缺失即失败，绝不用原始 projectId fallback。

## 主题2：工具执行系统（第14章）

### 关键文件
- BaseTool 基类：`business-tools/base/BaseTool.ts`
- ITool/ToolResult/上下文接口：`business-tools/base/interfaces.ts`
- 工具结果封装：`business-tools/base/toolResultEnvelope.ts`
- 自动注册：`business-tools/autoRegister.ts`
- LLM↔工具桥接：`agents/core/ai-execution/tool-loop/StreamingToolExecutor.ts`
- LLM 客户端接口：`infrastructure/llm/core/types.ts:344`（ILLMClient）
- tool_config：`prisma/schema.prisma:1437`
- project_tool_policy：`prisma/schema.prisma:1414`
- ToolResultArtifact：`prisma/schema.prisma:3597`

### 核心导出
- `BaseTool.ts:36-199` abstract class，`:71` `abstract execute(args, context): Promise<ToolResult>`，`:105` `toToolDefinition()`（Zod→JSON Schema，剥离 $schema/additionalProperties 适配 OpenAI function calling，实例级缓存）
- `interfaces.ts:15-21` `ToolResult { success; data?; error?; spanAttributes? }`；`:30` `ToolScope = 'role_default'|'project_scoped'|'agent_only'`；`:35-58` `ToolCategory`（22 类）；`:65-87` `ToolMetadata`（isReadOnly/isDestructive/isConcurrencySafe/defaultTimeoutMs/declaredOperations/acceptsSkillCredentialEnv）；`:110-133` `ToolExecutionContext`
- `autoRegister.ts:18-46` `TOOL_MODULES` 懒加载 **27 个**域模块（project/task/document/command/member/notification/tool-result/email/workflow/memory/web/session/workstation/scheduled/sql/tfs/wecom-*/rag/mcp/image/meta/interaction/orchestration/kdocs/code-intelligence）
- `types.ts:344-399` ILLMClient：`:362` `stream(messages, options?)`、`:390` `invokeNonStreaming()`（工具调用专用，保证 tool_calls.arguments 完整）、`:398` `withTools(tools)`

### 代码片段
BaseTool 转 ToolDefinition（`BaseTool.ts:105-121`）：
```ts
toToolDefinition(): ToolDefinition {
  if (!this._toolDefCache) {
    const rawSchema = zodToJsonSchema(this.getSchema() as any);
    const { $schema: _, additionalProperties: __, ...parameters } = rawSchema as Record<string, unknown>;
    this._toolDefCache = {
      name: this.name,
      description: this.getDescription(),
      parameters,
      declaredOperations: this.metadata.declaredOperations ?? getStaticToolDeclaredOperations(this.name),
    };
  }
  return { ...this._toolDefCache };
}
```

executeToolCall：权限检查 + 参数注入 + before hook + 观测（`StreamingToolExecutor.ts:3001-3047`）：
```ts
if (injectToolArgs) { toolArgs = injectToolArgs(toolName, toolArgs); }
let hookCtx;
if (this.config.hookRegistry) {
  hookCtx = await this.config.hookRegistry.runBeforeHooks({
    toolName, args: toolArgs, context: this.config.toolExecutionContext ?? {} as ...,
  });
  if (hookCtx.args) toolArgs = hookCtx.args;
}
this.recordToolLoopObservabilityEvent({ eventType: 'tool_call_start', toolName, toolCallId, toolInput: toolArgs });
const toolFn = tools.get(toolName);
```

### tool_config/project_tool_policy/ToolResultArtifact
- **tool_config**（L1437-1453）：`name(@id)/description/category/scope/version/parameters(Json)/permissions(String[])/dependencies(Json)/mcp_bridge_visible/visitor_enabled` — 注释"工具配置（分配/展示用，实现仍在代码）"
- **project_tool_policy**（L1414-1434）：`project_id/role_id?/digital_employee_id?/tool_name/effect=allow|deny/access_mode=member|visitor/source=manual|role_default|system/status=active|disabled`；索引 `[project_id,status]`、`[project_id,role_id]`、`[project_id,digital_employee_id]`、`[tool_name]`
- **ToolResultArtifact**（L3597-3621）：`projectId/conversationId?/agentRunId?/toolCallId?/toolName?/skillTargetId?/artifactKind/contentType/byteSize/sha256/storageBackend=postgres/storageKey?/content(Bytes)/partIndex?/partCount?/expiresAt` — 大块工具产物落库 + TTL 过期

### 设计要点
1. 统一上下文替代参数注入：BaseTool 注释"统一 ToolExecutionContext 传递上下文，替代 __agentId 参数注入"；getAgentId 仍兼容 args.__agentId 回退。
2. LLM 工具调用桥接双轨：stream() 流式用于对话；invokeNonStreaming() 工具调用场景确保 tool_calls.arguments 完整。
3. 工具结果两态封装：有 spanAttributes 时返回 envelope，否则裸 data；result.success=false 时走 ErrorEnricher.enrich()。
4. 工具发现用声明式操作授权：declaredOperations「未声明时不得从描述/关键词推断」；acceptsSkillCredentialEnv 强约束技能凭证。
5. 策略分层：tool_config（展示/分配元数据，实现仍硬编码）+ project_tool_policy（项目级 allow/deny，可细分 role/employee/visitor）+ ToolScope 三态共同决定可见性。
6. executeToolCall 的 span 关联：resolveOrCreateToolSpan 注释「Anthropic 早期 id 不稳定，不能直接做 span-source 关联」。

## 主题3：编码工作站（第15章）

### 关键文件
- coding_workstations 模型：`prisma/schema.prisma:212`
- CodingTask / CodingTaskArtifact：`:310` / `:389`
- 三层镜像模型：`:2151`(base)/`:2182`(engine)/`:2219`(type_config)/`:2241`(agent_engine)/`:2264`(agent_skill)/`:2287`(component)
- 工作站抽象基类：`business/domain/workstation/runtime/BaseWorkstation.ts`
- Runner（容器内命令）：`business-tools/workstation/runner/WorkstationRunner.ts`
- skill/workstation context probe：`business-tools/workstation/workspaceDiscovery/buildWorkspaceDiscoveryContext.ts`
- 工作站工具：`business-tools/workstation/WorkstationExecuteTool.ts`、`WorkspaceReadTool/WriteTool/GlobTool/ExecTool`
- 工作站解析/保障：`business/domain/workstation/resolveAndEnsureWorkstation.ts`

### 真实字段
- **coding_workstations**（L212-252）：`id/container_name(@unique)/agent_id/owner_kind=employee|project/project_id?/image/status=created|running|scaled_down|stopped|error/backend=local/type=coding/llm_binding_fingerprint/serviceName/servicePort`
- **CodingTask**（L310-386）：`workstationId/triggerTool=coding_task|sre_skill_task|execute_task/idempotencyKey/attemptNo/callbackTokenHash/taskFingerprint/claudeSessionId/hostProjectPath/workstationProjectPath/artifactRunRoot/lifecycleMetadata(Json)`；partial unique index `[sessionBindingKey,taskFingerprint,status,updatedAt] WHERE claude_session_id IS NOT NULL AND status='completed'`（:377）
- **workstation_runtime_base_image**（L2151-2178）：`repository/tag/digest(@nullable)/platforms/status=succeeded|failed|pending`，`@@unique([repository, tag])`
- **workstation_engine_image**（L2182-2216）：`engineType=claude_code|codex|hermes|openclaw/engineName/engineVersion/imageRepository/imageTag/imageDigest/imageBuildStatus/runtimeBaseImageId`，`@@unique([engineType, engineName, engineVersion])`
- **workstation_type_config**（L2219）→ **workstation_agent_engine**（L2241）→ **workstation_agent_skill**（L2264）→ **workstation_component**（L2287，存 installScript 全文 + checksum）

### 代码片段
coding_workstations K8s 运行形态（`schema.prisma:230-242`）：
```prisma
  /// Pod 创建期 LLM bindingFingerprint（无密钥）
  llm_binding_fingerprint String? @map("llm_binding_fingerprint")
  ingressPath    String?   @map("ingress_path")
  serviceName    String?   @map("service_name")
  servicePort    Int?      @map("service_port")
```

WorkstationRunner 委托远程容器（`WorkstationRunner.ts:34-54`）：
```ts
export class WorkstationRunner implements IWorkspaceCommandRunner {
  readonly kind = 'workstation' as const;
  async runRead(absoluteFilePath, options, ctx): Promise<string> {
    return this.exec(this.buildReadCommand(absoluteFilePath, options), undefined, ctx);
  }
  async runWrite(absoluteFilePath, content, options, ctx): Promise<void> {
    await this.exec(this.buildWriteCommand(absoluteFilePath, content, options), undefined, ctx);
  }
  async runExec(command, options, ctx): Promise<string> { return this.exec(command, options, ctx); }
```

### 设计要点
1. 三层镜像模型：base_image（OCI base，digest 真源）→ engine_image（claude_code/codex/hermes/openclaw 引擎，独立管理不绑 type）→ component（target image 安装脚本，compatibleEngines[] 控兼容性）——引擎与运行时正交解耦。
2. type→engine→skill 配置树：type_config → agent_engine → agent_skill + workstation_skill_assignment。
3. 任务幂等与 callback 安全：CodingTask 用 (conversation_id, trigger_tool, idempotency_key) 在 running 态去重 + attemptNo 防迟到旧 attempt 覆盖新状态 + callbackTokenHash 鉴权回调。
4. 所有工作站走远程 sandbox-api（K8s Pod）：BaseWorkstation.ts:8 明示非本地 docker。
5. workspace discovery 双路径：同时维护 hostPath 与 containerPath（pathRegistry 双向转换）。
6. 个人分身工作站独立模型：UserPersonalWorkstation（schema:3624）与 coding_workstations.agent_id 员工语义隔离，availabilityPolicy=adaptive|always_on|on_demand + podGeneration。

## 主题4：知识库系统（第16章）

### 关键文件
- knowledge_bases/knowledges/chunks/tags：`prisma/schema.prisma:1456`/`:1491`/`:1660`/`:1685`
- Confluence 集成：`:1540`(sources)/`:1577`(pages)/`:1600`(sync_runs)/`:1629`(assets)
- Confluence PAT 密文：`:1530`(project_confluence_credentials)
- embedding_cache：`:495`
- 入库 pipeline：`infrastructure/rag/RagService.ts`（ingestBufferDetailed）
- BullMQ worker：`infrastructure/rag/ingest/ragIngestWorker.ts`
- 分块器：`infrastructure/rag/DocumentChunker.ts`
- Confluence 同步：`business-tools/knowledgeBase/ConfluenceKnowledgeSyncService.ts`
- 独立 embedding 服务：`embedding-server.ts`（仓库根）

### 真实字段
- **knowledge_bases**（L1456-1488）：`project_id/name/type=document|faq|tfs_git/chunking_config(Json，chunk_size/chunk_overlap/split_markers)/embedding_model?/extract_config(Json，图谱抽取)/faq_config?/question_generation_config?/tfs_git_config?`
- **knowledges**（L1491-1527）：`type=file|url|manual|tfs_git_file|confluence_page/parse_status=pending|processing|completed|failed/enable_status/file_hash(去重)/tag_id?/metadata(Json)`
- **knowledge_chunks**（L1660-1682）：`content/chunk_index/chunk_type=text|image|table/parent_chunk_id?(parent-child 分块)/metadata(Json)`
- **knowledge_base_confluence_sources**（L1540-1574）：`base_url/root_page_id/root_page_url/recursive/interval_minutes/next_sync_at/status=idle|.../lease_token/lease_until/asset_*`；partial unique index（WHERE deleted_at IS NULL）
- **knowledge_confluence_pages**（L1577-1597）：`source_id/page_id/knowledge_id/version_number/content_hash/document_id/asset_manifest_hash/connection_state`，`@@unique([source_id, page_id])`
- **embedding_cache**（L495-507）：复合主键 `@@id([provider, model, hash])` + `embedding_dims/chroma_id(@deprecated)`

RagService.ts:161 `ingestBufferDetailed(buffer, fileName, options)`；`:203` parserRegistry.parse → `:233/246` chunkDocumentParentChildAsync/chunkDocumentAsync → `:313` upsertRagChunks → `:320` syncChunksToDb → `:324` 图谱抽取 → `:330` 问题生成。
ragIngestWorker.ts:159-180 `startRagIngestWorker()` — BullMQ，RAG_INGEST_QUEUE_NAME，concurrency 来自 ragIngestConfig.workerConcurrency。

### 代码片段
ingestBufferDetailed pipeline（`RagService.ts:203-313`）：
```ts
const parseResult = await parserRegistry.parse(buffer, fileName);
let separators = kbChunkingCfg?.splitMarkers;
if (!separators) { separators = getSeparatorsByType(parseResult.documentType); }
const chunkingMode = options.chunkingMode ?? (kbChunkingCfg?.parentChildEnabled ? 'parent_child' : 'flat');
const buildChunks = async () =>
  chunkingMode === 'parent_child'
    ? (await chunkDocumentParentChildAsync(parseResult.sections, ...)).flatMap((pair) => [pair.parent, ...pair.children])
    : await chunkDocumentAsync(parseResult.sections, ..., { maxChunkSize: chunkSize, overlapSize: chunkOverlap, separators });
const synced = await upsertRagChunks(chunks);
await this.syncChunksToDb(options.knowledgeId, chunks);
if (isNeo4jAvailable() && options.knowledgeBaseId) {
  void this.extractGraphForChunks(chunks, options.knowledgeBaseId).catch(...);
}
```

DocumentChunker 文档类型感知分隔符（`DocumentChunker.ts:25-50`）：
```ts
const DEFAULT_SEPARATORS = ['\n\n', '\n', '。', '！', '？', ';', '；', '.'];
const MARKDOWN_SEPARATORS = ['\n\n\n', '\n\n', '## ', '### ', '#### ', '\n', '。', '！', '？', ';', '；', '.'];
const PDF_SEPARATORS = ['\n\n', '\n', '。', '!', '?', '.'];
const EXCEL_SEPARATORS = ['\n', '|', '\t', ','];
```

### 设计要点
1. 知识库三型一表：type 单字段承载 document|faq|tfs_git，配置全 Json——避免模型爆炸。
2. Confluence 同步完整审计链：sources（lease 防并发）→ sync_runs（计数）→ pages（稳定映射）→ assets（正文↔MinIO）。PAT 用 aes-256-gcm-v1 密文。
3. 分块双模式同算法：移植自 WeKnora splitter.go，flat 与 parent_child 复用同一核心（保护 LaTeX/代码块/表格/图片不被截断，ABSOLUTE_MAX_CHUNK_SIZE=7500）。
4. embedding_cache 复合主键去重；chroma_id @deprecated 说明曾从 ChromaDB 迁出。
5. 独立 embedding 微服务：embedding-server.ts 是 standalone Fastify（/v1/embed 等），含 withInflight 容量保护（429）+ serviceToken 鉴权；业务进程只通过 EmbeddingClient 调用。
6. 入库即资产化：分块前后两次调 buildChunks（资产替换前/后），图片转 URL 落盘，原件 saveOriginalDocument 保证 chunks=0 也能 re-ingest。

## 主题5：RAG（第17章）

### 关键文件
- rag-worker 入口：`rag-worker.ts`（仓库根，src 下）→ `startup/ragEntry.ts` → `startup/rag.ts`
- 混合检索引擎：`infrastructure/rag/RagHybridSearch.ts`
- 检索工具（5 个）：`business-tools/rag/RagSearchTool.ts`/`RagKnowledgeSearchTool.ts`/`RagGraphSearchTool.ts`/`RagListDocumentsTool.ts`/`RagReadChunksTool.ts`/`UnifiedKnowledgeRetrieveTool.ts`
- 检索融合/重排/去重：`infrastructure/search/fusion.ts`/`mmr.ts`/`reranker.ts`
- 查询分类/改写/动态权重：`infrastructure/search/QueryTypeClassifier.ts`/`QueryRewriter.ts`/`DynamicWeightCalculator.ts`/`DynamicThresholdCalculator.ts`

### 核心导出
- `rag-worker.ts:1-19`：`assertProcessRole('rag')` 守卫 → 动态 import('@/startup/ragEntry.js')。Role guard 早于动态 import。
- `RagHybridSearch.ts:149` `ragHybridSearch(opts)`：管线「Phase0 查询改写 → Phase1 查询分类+动态权重 → Phase2 并行向量+BM25 → 融合 → rerank → MMR → minScore 过滤 → limit 截断」
- `RagHybridSearch.ts:67-108` `deduplicateByDocument()`（按 documentPath 去重）
- `RagSearchTool.ts:34-124` `class RagSearchTool extends BaseTool`（name='rag_search'，scope='project_scoped'，category='document'），`:85-95` 调 ragService.hybridSearch({ hybrid: true, fusionStrategy: 'rrf' })

### 代码片段
rag-worker 生产入口（`rag-worker.ts:8-19`）：
```ts
import './infrastructure/sandbox/config/undicioConfig.js';
import { assertProcessRole } from '@/startup/processRole.js';
try { assertProcessRole('rag'); }
catch (err) {
  const message = err instanceof Error ? err.message : String(err);
  process.stderr.write(`[ProcessRole] rag-worker entry startup aborted: ${message}\n`);
  process.exit(1);
}
await import('@/startup/ragEntry.js');
```

混合检索管线（`RagHybridSearch.ts:147-229`）：
```ts
export async function ragHybridSearch(opts): Promise<RagHybridSearchResult[]> {
  const limit = opts.limit ?? 10;
  const minScore = opts.minScore ?? 0.3;
  const strategy = opts.fusionStrategy ?? 'rrf';
  // Phase 0: 查询改写
  let effectiveQuery = opts.query;
  const shouldRewrite = opts.enableQueryRewriting ?? isQueryRewritingEnabled();
  if (shouldRewrite) { rewriteResult = await rewriteQuery(opts.query, { enableLLM: false }); ... }
  // Phase 1: 动态权重
  if (isDynamicWeightsEnabled() && opts.vectorWeight === undefined && opts.textWeight === undefined) {
    const classification = classifyQuery(effectiveQuery);
    const weights = calculateWeights(classification.type);
    vectorWeight = weights.vectorWeight; textWeight = weights.textWeight;
  }
  // Phase 2: 并行搜索（向量 + BM25）
  const overRetrieval = useRerank || useMMR ? 5 : 2;
  const [vectorResults, ftsResults] = await Promise.all([
    searchVector(searchOpts, limit * overRetrieval),
    useHybrid ? searchFullText(searchOpts, limit * overRetrieval) : Promise.resolve([]),
  ]);
```

### 设计要点
1. 独立进程而非 worker thread：rag-worker.ts 用 WIN_PROCESS_ROLE=rag 进程级隔离，ES/重排与 API server 解耦。
2. 全 ES 混合检索（无独立向量库）：ES kNN 向量搜索（winmatrix_rag 索引，cosine）+ ES BM25——不再依赖 ChromaDB。
3. RRF 为默认融合（rrfK=60），weighted 为备选。
4. 可编排的检索增强管线：rerank/mmr/enableQueryRewriting/enableDocumentDedup 全开关；过召回倍数 overRetrieval = (rerank||mmr) ? 5 : 2。
5. 查询分类驱动的动态权重：classifyQuery 输出 QueryType，calculateWeights 算权重；DynamicThresholdCalculator 动态调 minScore。
6. 多工具 Agentic RAG：rag_search + rag_list_documents + rag_read_chunks（>500 字截断提示）+ rag_knowledge_search/rag_graph_search + UnifiedKnowledgeRetrieveTool。

## 任务描述偏差核实
1. 源码根：真实仓库在 `E:/winning/code/AI/winmatrix/winmatrix-server/`。
2. rag-worker.ts 在 src/ 下（winmatrix-server/src/rag-worker.ts）。
3. skill_artifact 第 1273 行——正确。
4. **CLAUDE.md"bundled skill 已 deprecated"原文在当前 CLAUDE.md 未找到**，应引用 `scripts/seed-bundled-skills.ts:4` 头部注释。
5. L1/L2/L3 不是独立文件，散落在 ProjectSkillScopeService（L1）、skillTargetPlanningAudit（L2）、SkillReadinessGate+SkillPreflightOrchestrator（L3）的注释与方法语义中。
