# 源码核实报告：Agent 系统（ch09-12）

源码根：`E:/winning/code/AI/winmatrix/winmatrix-server/src/`

> CLAUDE.md 指向的 SSOT 全部成立：决策引擎 `agents/core/agent/decision/DecisionEngine.ts`，Turn 编排 `agents/core/agent/turn/TurnRunner.ts`。但八大 Role 目录是双层结构：Role **基类与注册表**在 `agents/core/worker/role/`，具体 Role **实现类**在 `agents/domain-harness/roles/`。

## 主题1：数字员工模型（第9章）

### 关键文件
- `agents/core/worker/role/BaseRole.ts`（1041 行）— Role 基类，MetaGPT 式 Observe-Think-Act 循环
- `agents/domain-harness/roles/*.ts` — 七大业务 Role 实现类
- `agents/core/worker/role/roleRegistry.ts` — RoleRegistry 单例：插件注册 + 工厂 + 实例缓存
- `agents/core/worker/digitalEmployee/DigitalEmployee.ts` — DigitalEmployee 执行编排层 + `createDigitalEmployee()` 工厂
- `startup/agents.ts` — 七大 Role 工厂注册（行 126-133）
- `types/config.ts` — `AgentConfig` 接口 + Zod schema
- `prisma/schema.prisma` — `agent_config` 模型（行 1159-1181）

### 核心签名
**BaseRole** — `agents/core/worker/role/BaseRole.ts`
- L66 `export abstract class BaseRole`；L67-72 四抽象身份字段 `name/profile/goal/constraints`
- L103-118 `constructor(config?: RoleConfig)`
- L251-280 `protected onInitialized()` — 注册 `configManager.onConfigChange` 监听（热重载）
- L293-310 `protected async observe(): Promise<Message[]>` — 从 `rc.getNews()` 取新消息，去重
- L340-392 `protected async think(): Promise<boolean>` — 消费 `todoQueue` → `getExecutionModeDecision` → `executeStep`
- L406-544 `protected async act()` + L415 `async runQueuedAction(options)` — 执行 queued action
- L643-717 `async executeWithRuntimeKernel(input)` — chat/single 执行路径
- L860-870 `async initialize(userPermissions?: string[])`
- L917-935 `async reloadFromConfig()` — 配置热重载

**八大员工（全部已核实）**
| Role ID | 类 | 文件 | 昵称 |
|---|---|---|---|
| orchestrator | OrchestratorRole | roles/OrchestratorRole.ts L28 | 大福 |
| process_manager | ProcessManagerRole | roles/ProcessManagerRole.ts L25 | 阿宁 |
| product_design_manager | ProductDesignRole | roles/ProductDesignRole.ts L27 | 小品 |
| tech_manager | TechManagerRole | roles/TechManagerRole.ts L28 | 阿码 |
| quality_manager | QualityManagerRole | roles/QualityManagerRole.ts L28 | 小质 |
| sre_manager | SreManagerRole | roles/SreManagerRole.ts L26 | 大维 |
| architect | ArchitectRole | roles/ArchitectRole.ts L26 | Architector |
| external-agent | ExternalAgentChatAdapter | 运行时动态创建（DigitalEmployee.ts L396-401） | — |

注意：第 8 类 external-agent 不是 `domain-harness/roles/` 下的类，而是 `createDigitalEmployee()` 对外联 Agent 生成虚拟 DigitalEmployeeRecord + ExternalAgentChatAdapter。

**RoleRegistry** — `agents/core/worker/role/roleRegistry.ts`
- L36 `class RoleRegistry`（单例，L264 `export const roleRegistry`）
- L154 `registerFactory(roleId, factory)`
- L178 `async createRole(roleId)` — **每次创建新实例，不缓存**（避免单例并发覆盖）
- L197 `async getRole(roleId)` — 带缓存懒加载

工厂注册 `startup/agents.ts` L126-133：
```ts
roleRegistry.registerFactory('architect', () => new ArchitectRole());
roleRegistry.registerFactory('orchestrator', () => new OrchestratorRole());
roleRegistry.registerFactory('process_manager', () => new ProcessManagerRole());
roleRegistry.registerFactory('product_design_manager', () => new ProductDesignRole());
roleRegistry.registerFactory('tech_manager', () => new TechManagerRole());
roleRegistry.registerFactory('sre_manager', () => new SreManagerRole());
roleRegistry.registerFactory('quality_manager', () => new QualityManagerRole());
Architector.initialize();
```

`agent_config` 模型（schema L1159-1181）：`id/name_cn/name_en/nickname/emoji/description/role/focus/projectId/workstation/workstation_config_version/version`。
`AgentConfig` 接口（types/config.ts L164-187）：`nickname/emoji/personality:string[]/principles:AgentPrinciple[]/workstation:AgentWorkstationConfig/memory:AgentMemoryRecallConfig`。

### 代码片段
Role 从 agent_config 加载身份（`TechManagerRole.ts:35-49`）：
```ts
constructor() {
  super({ reactMode: RoleReactMode.REACT, watch: [] });
  const agentConfig = this.loadConfigFromManager('tech_manager');
  this.name = 'tech_manager';
  this.profile = agentConfig?.role || '研发技术管理者';
  this.goal = agentConfig?.description || '制定工程化标准，守护代码质量...';
  this.constraints = agentConfig?.focus || '简单优先、成熟稳定...';
  this.nickname = agentConfig?.nickname || '阿码';
  this.onInitialized();
}
```

数字员工工厂（每会话独立 Role 实例）（`DigitalEmployee.ts:410-436`）：
```ts
const role = await roleRegistry.createRole(record.role_id);
if (!role) throw new Error(`[createDigitalEmployee] 未找到对应 Role: ${record.role_id}`);
const promptOverrideService = container.resolve<{ getPromptContext(id): Promise<{...}> }>('PromptOverrideService');
const promptContext = await promptOverrideService.getPromptContext(digitalEmployeeId);
if (promptContext) {
  record.role_supplement_note = promptContext.roleSupplementNote ?? null;
  record.prompt_override = promptContext.promptOverride ?? null;
}
return new DigitalEmployee(record, role);
```

### 智能路由 route_rule
`route_rule` 模型（schema L2689-2730）：`name(unique)/patterns:String[]/patternFlags:String[]/positiveIntents:String[]/negativeIntents:String[]/semanticAnchors:String[]/semanticThreshold:Float(默认0.85)/roleId/mode/skillName/declaredAction/producesDataKind/requiresDataKind/toolHint/status(active|shadow)/source(manual)/priority/hitCount/lastHitAt`。

FusionRouter（`agents/core/agent/decision/fusion-router.ts`）
- L116 `export class FusionRouter`
- L165 `route(input): RouteResult | null` — 遍历所有规则 computeScore，`score >= route.semanticThreshold` 且最高分获胜；shadow 规则只记录不路由
- L194-201 命中方法：regex→'regex'；positiveIntents→'fusion'；否则→'semantic'

```ts
route(input: string): RouteResult | null {
  let bestRoute: RouteEntry | null = null;
  let bestScore = 0;
  for (const route of this.routes) {
    const score = this.computeScore(route, input);
    if (score >= route.semanticThreshold) {
      if (route.status === 'shadow') { this.bumpMetrics(route.id); this.shadowHits.push({...}); continue; }
      if (score > bestScore) {
        bestScore = score; bestRoute = route;
        const regexHit = route.patterns.some(p => p.test(input));
        if (regexHit) bestMethod = 'regex';
        else if (route.positiveIntents.some(w => input.includes(w))) bestMethod = 'fusion';
        else bestMethod = 'semantic';
      }
    }
  }
  if (bestRoute) { this.bumpMetrics(bestRoute.id); return { route: bestRoute, confidence: bestScore, method: bestMethod, source: bestRoute.source }; }
  return null;
}
```

### 设计要点
1. 三层分离：DigitalEmployee（执行编排层，唯一外部 RoleContext 写入者）/ BaseRole（能力定义层）/ RoleContext（运行时数据容器）。
2. MetaGPT 式 Observe-Think-Act 循环，公式 `Role = LLM + Observation + Thinking + Action + Memory`。
3. 配置驱动 + 热重载：Role 身份从 agent_config 表加载，`configManager.onConfigChange` 监听 PG `pg_notify('config_change')`，命中 affectedRoleIds 时 reloadFromConfig。
4. 会话级独立实例：createRole 每次创建不缓存，规避单例并发覆盖。
5. 一 Role 对多 DigitalEmployee：agent_config 带 projectId 区分平台 Role（null）与项目空间 Role。
6. FusionRouter 双模式：active 竞速选最佳；shadow 只记录指标（A/B 验证新路由）。

## 主题2：渐进式决策引擎（第10章）

### 关键文件
- `agents/core/agent/decision/DecisionEngine.ts` — **决策引擎 SSOT**（57KB，5 阶段管线）
- `agents/core/agent/decision/architector/Architector.ts` — 决策入口编排器（单例）
- `agents/core/agent/decision/stages/ExactRouter.ts` — Stage 1 精确路由
- `agents/core/agent/decision/stages/SimpleChatGuard.ts` — Stage 1 闲聊守卫
- `agents/core/agent/decision/stages/FusionRouterStage.ts` — Stage 2
- `agents/core/agent/decision/stages/QuickAcceptGate.ts` — Stage 3 快速接受 / 语义 cache
- `agents/core/agent/decision/stages/DecisionPlanner.ts` — Stage 4 LLM Planner（40KB）
- `agents/core/agent/decision/stages/DecisionCommitmentDeriver.ts` — Stage 5 承诺派生（21KB）
- `agents/core/agent/decision/types.ts` — `DecisionRouteLayer = 'L0'|'L1'|'L2'|'L3'|'Chat'`
- `agents/core/agent/decision/SemanticTriageGate.ts` — 语义分诊门

### 核心签名
**DecisionEngine** — `DecisionEngine.ts`
- L113 `export class DecisionEngine`
- L115-127 字段：`exactRouter / fusionRouterStage / quickAcceptGate / decisionPlanner / commitmentDeriver / simpleChatGuard`，构造接 `hooks: PipelineHook[]`
- L368 `async decide(input: DecisionInput): Promise<DecisionResult>` — 公开入口
- L372 `private async decideInner(input)` — 五阶段管线主体（L372-676）
  - Stage 1（L392-423）：`ExactRouter + PlanExtraction` → exactRouter.routeAsync + simpleChatGuard.extract + extractExplicitActionChainJson
  - 显式流程编排短路（L425-439）：`tryExplicitFlowOrchestrationPlan`
  - Stage 2（L441-495）：`FusionRouter` → fusionRouterStage.execute 产候选
  - Stage 3（L503-616）：`QuickAcceptGate` → 含 cacheLookup（语义 planner cache pre-check）；命中则 finalizeAcceptedPlan + commitmentDeriver.derivePlan，直接 return
  - Stage 4（L646-664）：`DecisionPlanner`（LLM 规划）→ plannerStage
  - Stage 5（L666-675）：`DecisionCommitmentDeriver` → derivePlanStage

**Architector** — `architector/Architector.ts`
- L103 `export class Architector`（单例 L104 `_instance`，L128 `getInstance()`，L139 `initialize()`）
- L175 `async decide(context: UnifiedDecisionContext): Promise<DecisionOutput>` — 七步流程
- L240 `async decideRoute(context)` — 便利方法

**分层路由** — `types.ts`
- L77 `export type DecisionRouteLayer = 'L0' | 'L1' | 'L2' | 'L3' | 'Chat'`
- L84 `getDecisionRouteLayer(d)` — 从 metadata 读 routeLayer
- L196-200 `predecidedTargetId` — L0 预决策（worker/定时任务注入）
- L185-188 `candidateCapabilitySnapshot` — L1 候选能力快照（Turn 层注入，禁止重复 collect）

**SemanticTriageGate** — `SemanticTriageGate.ts`
- L77-81 `SemanticTriageClassification = 'casual_chat' | 'termination_intent' | 'bound' | 'unbound'`
- L93-98 `SemanticBoundIntent = 'continue_retry' | 'append_modify' | 'cancel' | 'unknown'`
- L108 `LOW_CONFIDENCE_THRESHOLD = 0.65`，L102 `TERMINATION_TIMEOUT_MS = 10_000`

### 代码片段
五阶段管线骨架（`DecisionEngine.ts:372-440`）：
```ts
private async decideInner(input: DecisionInput): Promise<DecisionResult> {
  let draft = this.emptyDraft(input, resolvedTurn);
  const extractedCandidates: ExtractedPlanCandidate[] = [];
  // Stage 1: ExactRouter + plan extraction
  await invokePipelineHooks(this.hooks, 'DecisionEngine', 'onStageStart', 'ExactRouter+PlanExtraction', draft);
  const exactMatch = await this.exactRouter.routeAsync(input, resolvedTurn);
  const continuationCandidate = ExactRouter.extractContinuationCandidate(input, exactMatch);
  if (continuationCandidate) extractedCandidates.push(continuationCandidate);
  const { candidate: chatCandidate, abstainReason: simpleChatAbstain } =
    await this.simpleChatGuard.extract(input, exactMatch, resolvedTurn);
  // ExplicitFlowOrchestration: 用户显式描述多步流程时直接编译，跳过后续
  const explicitFlowOutcome = this.tryExplicitFlowOrchestrationPlan(input, resolvedTurn, exactMatch, emptyCandidates, draft);
  if (explicitFlowOutcome?.terminal) {
    explicitFlowOutcome.plan.boundRoute = 'explicit_flow_orchestration';
    return this.finalize(explicitFlowOutcome);
  }
  // Stage 2: FusionRouter ...
```

分层路由 L0-L3（`types.ts:77-100`）：
```ts
export type DecisionRouteLayer = 'L0' | 'L1' | 'L2' | 'L3' | 'Chat';
function isDecisionRouteLayer(v: unknown): v is DecisionRouteLayer {
  return v === 'L0' || v === 'L1' || v === 'L2' || v === 'L3' || v === 'Chat';
}
export function getDecisionRouteLayer(d: { metadata?: unknown }): DecisionRouteLayer | undefined {
  // ...从 d.metadata.routeMetadata.routeLayer 读取并类型守卫
}
```

### 设计要点
1. **渐进式 5 阶段管线**（非一次性 LLM）：ExactRouter+PlanExtraction → FusionRouter → QuickAcceptGate → DecisionPlanner(LLM) → DecisionCommitmentDeriver。每阶段可 terminal 提前返回，越早越便宜。
2. **三层短路加速**：(a) SimpleChatGuard 命中规则直接 chat；(b) ExplicitFlowOrchestration 显式多步流程直接编译；(c) QuickAcceptGate cacheLookup（embedding 相似度 + inputFingerprint）命中跳过 LLM。
3. **分层路由 L0/L1/L2/L3/Chat**：写入 metadata.routeMetadata.routeLayer。L0=系统预决策 target；L1=候选能力快照；L2=异步 LayeredRouter；L3=readiness gate；Chat=闲聊。**注意：L0-L3 是路由层维度，与 5 阶段管线是两个不同维度**。
4. Architector 单例编排，decide() 是所有路由决策统一入口。
5. 候选 → 计划 → 承诺分离。
6. pipeline hooks 观测：每阶段 onStageStart/onStageEnd 记录 stageTrace、elapsedMs。

## 主题3：Turn 执行引擎（第11章）

### 关键文件
- `agents/core/agent/turn/TurnRunner.ts` — **Turn 编排 SSOT**（对象字面量，非 class）
- `agents/core/agent/turn/TurnAdmission.ts` — 准入门（runPreRouteGates / SemanticTriageGate A）
- `agents/core/agent/turn/TurnExecutionAssembly.ts` — prepareTurnContext + assembleTurnResult + executePreparedTurn
- `agents/core/agent/turn/turnShared.ts` — executeRoute / resolveInteractiveSessionForPipeline
- `agents/core/ai-execution/tool-loop/StreamingToolExecutor.ts` — **LLM Tool 调用循环**（流式 + 终止条件）
- schema：`agent_run`(L1864) / `agent_run_step`(L2022) / `agent_run_decision`(L1971) / `agent_run_state`(L1986)

### 核心签名
**TurnRunner** — `TurnRunner.ts`
- L67 `export const TurnRunner`（对象字面量）
- L68 `async run(options: TurnRequest): Promise<ChatPipelineResult>` — Turn 生命周期主体
- L102-108 三路并行：`Promise.all([TurnContextLoader.loadPipelineContext, resolveAgentContext, getHistoryAdapter().getRecentMessages(conversationId, 30)])`
- L130-138 `assembleLlmExecutionBinding` — 装配无密钥 LLM binding
- L146-151 `buildTurnCandidateCapabilitySnapshot` — L1 候选能力快照
- L153-179 `runPreRouteGates` — 准入门
- L217-246 路由决策：`options.predecidedRoute`（缓存）或 executeRoute
- L248-253 `materializeExecutionProjection`
- L281-301 `executePreparedTurn`
- L302-308 `finally` — 未移交 ticket 则 releaseLlmExecutionBinding

**StreamingToolExecutor** — `StreamingToolExecutor.ts`
- L877 `export class StreamingToolExecutor`
- L946 `async execute(messages: ChatMessage[]): Promise<ExecutorResult>`
- L1026 `while (iteration < maxIterations && totalLlmRounds < hardCapLlmRounds)` — 主循环
- L269 `resolveMetaToolLlmRoundHardCap(maxIterations)` — 硬上限 = maxIterations + 20（AGENT_META_TOOL_EXTRA_LLM_ROUNDS）
- L678 `resolveEffectiveMaxIterations(params)` — 按决策模式动态调整
- L1027-1046 `roundBudgetMs` 超预算提前结束
- L1085 `callLLMStreaming` — 每轮流式
- L1087-1108 空 args 修复：流式返回 tool_calls arguments 为空时用非流式补全

### 代码片段
TurnRunner.run（`TurnRunner.ts:67-100`）：
```ts
export const TurnRunner = {
  async run(options: TurnRequest): Promise<ChatPipelineResult> {
    const { input, conversationId: providedConversationId, userId, project: originalProject, target, logPrefix, metadata } = options;
    const pipelineInteractiveSession = resolveInteractiveSessionForPipeline(options);
    const conversationId = providedConversationId ?? conversationIdFactory.createChatConversationId();
    const executionSessionId = options.executionSessionId ?? generateSessionId();
    if (!projectContext && !options.metadata.scheduledTaskName) {
      throw new Error('TURN_RUNNER_REQUIRES_PROJECT_CONTEXT');
    }
    // 三路并行：loadPipelineContext + resolveAgentContext + getRecentMessages
    const [turnPipelineContext, context, historyResult] = await Promise.all([
      TurnContextLoader.loadPipelineContext(conversationId),
      resolveAgentContext(conversationId),
      useCausalHistory ? Promise.resolve({...}) : getHistoryAdapter().getRecentMessages(conversationId, 30),
    ]);
```

Tool 调用循环与终止条件（`StreamingToolExecutor.ts:1026-1052`）：
```ts
while (iteration < maxIterations && totalLlmRounds < hardCapLlmRounds) {
  if (roundBudgetMs !== undefined && Date.now() - loopStartAt > roundBudgetMs) {
    logger.warn(`[StreamingToolExecutor] 超出单轮预算 ${roundBudgetMs}ms，提前结束`);
    this.loopAbortController?.abort('round_budget_exceeded');
    return finalizeSuccess({
      output: output || `已达到执行预算（${roundBudgetMs}ms），先返回当前结果；如需继续请明确下一步。`,
      intermediateSteps, iterations: iteration,
      terminationReason: 'round_budget_exceeded',
    });
  }
  iteration++;
  totalLlmRounds++;
  logger.info(`[StreamingToolExecutor] 迭代 ${iteration}/${maxIterations}（LLM 轮次 ${totalLlmRounds}/${hardCapLlmRounds}）`);
  let assistantMessage: AssistantMessage = await this.callLLMStreaming(clientWithTools, messagesForLlm);
```

### agent_run 与 Turn 的关系
- `agent_run`（L1864-1919）：一次完整多步执行根聚合。`conversationId/intentSummary/decomposition:Json?/orchestrationPlan:Json?/reviewDecisions:Json?/finalOutput/status(running|completed|partial|failed)/executionPlanKind`。
- `agent_run_decision`（L1971-1984）：决策审计。`runId/stage/decisionKind/inputDigest:Json?/outputSnapshot:Json?/normalizedMode`。索引 `[runId, stage]`。
- `agent_run_step`（L2022-2051）：步骤级。`runId/mode/stepId/briefId/parentStepId/orderIndex/roleId/digitalEmployeeId/status/completionDecision:Json?/spanId`。唯一键 `[runId, stepId]`。
- `agent_run_state`（L1986-2000）：编排状态外置。`runId/stateType/status/compactState:Json?/checkpointRef`。唯一 `[runId, stateType]`。
- `agent_worker_execution`（L2053-2084）：worker 执行（含 codingTaskId/contractSnapshot/outputData）。唯一 `[runId, stepId, attemptNo]`（支持重试）。
- 关系：Turn 通过 `turnAgentRunId` 贯穿决策→执行→评审。`AgentExecutionTicket` 是内存桥梁，持久化落到 agent_run 聚合根 + step/decision/state 子表。

### 设计要点
1. 单轨编排（single-track）：load → admitTurn → resolveRoute → assembleExecution。文件头注释「Turn single-track orchestration SSOT」。
2. 三路并行 IO 合并：三者仅依赖 conversationId，Promise.all 消除一段 RTT。
3. 预决策路由缓存：predecidedRoute 准入通过后直接复用，跳过 executeRoute。
4. Tool 循环双终止：`iteration < maxIterations` AND `totalLlmRounds < hardCapLlmRounds`（= maxIterations + 20）。另有 roundBudgetMs 超预算终止。豁免工具轮不占配额（DEFAULT_ITERATION_BUDGET_EXEMPT_TOOL_NAMES L233）。
5. 流式 + 容错：每轮 callLLMStreaming；空 args 降级非流式。LLM binding 在 finally 按 bindingHandedOff 决定释放。
6. SemanticTriageGate Point A/B：Point A 路由前语义分诊，casual_chat 直接短路。

## 主题4：记忆系统（第12章）

### 关键文件
- `docs/MEMORY_SYSTEM.md` — 记忆系统设计文档（架构图、数据流、对比表）
- `agents/core/ai-kernel/context/adapters/MemoryContextBootstrap.ts` — **长期记忆注入 SSOT**（injectLongTermMemory / 三区检索）
- `infrastructure/memory/MemoryIndexManager.ts` — 记忆索引管理器
- `infrastructure/memory/HybridSearch.ts` — 混合检索（PG tsvector + ES cosine）
- `infrastructure/session/ConversationService.ts` — 会话历史管理（Redis + PG）
- `infrastructure/memory/conversationMemory.ts` — Redis 短期缓存（50 条）
- `infrastructure/memory/syncSessionTranscriptsToMemory.ts` — TranscriptSyncManager
- `business-tools/memory/MemorySearchTool.ts` — 显式检索工具 memory_search
- `agents/core/ai-kernel/memory/MemoryKernel.ts` — MemoryKernel
- schema：`memory_chunks`(L536) / `memory_files`(L560)

### 核心签名
**MemoryContextBootstrap** — `MemoryContextBootstrap.ts`
- L22 `DEFAULT_MEMORY_LIMIT = 5`（top-k）；L25 `DEFAULT_MIN_SCORE = 0.05`
- L29-34 三区检索常量：
  - `ZONE1_LIMIT = 3; ZONE1_MIN_SCORE = 0.25`（当前会话 session）
  - `ZONE2_LIMIT = 3; ZONE2_MIN_SCORE = 0.5`（项目 memory）
  - `ZONE3_LIMIT = 1; ZONE3_MIN_SCORE = 0.8`（跨会话 session，降级）
- L88 `formatMemoryResults(results)` — 格式化为 Markdown 注入文本（带 [decision/finding/preference/constraint/info] 类型标签）
- L101 `checkMemoryReference(injectedMemories, llmOutput)` — 检测 LLM 输出是否引用注入记忆
- L127 `async function searchLongTermMemoryResults(input)` — 三区检索（前两区不足 2 条才触发 Zone3）
- L370 `export async function injectLongTermMemory(input)` — **注入主入口**

**MemoryIndexManager** — `MemoryIndexManager.ts`
- L50 `export class MemoryIndexManager`
- L299 `async indexContent(virtualPath, content, source?, projectId?, agentId?)` — 虚拟路径索引
- L356 `async search(opts)` — 委托 hybridSearch
- L370 `private async indexContentCore(...)` — 核心：hash 检查 → 删旧 → 分块 → 双写 ES+PG
- L420-426 ES 写入：`defaultMemoryVectorStore.addDocuments(...)`
- L442-443 PG 写入：upsertMemoryFile + upsertMemoryChunks
- L464 `watchDirectories(dirs, source?, projectId?, agentId?)` — chokidar 目录监听

memory_chunks（L536-558）：`id/path/source(default 'memory')/start_line/end_line/hash/text/chroma_id(legacy)/project_id/agent_id/tsv(tsvector,Gin)`。
memory_files（L560-574）：`path(PK)/source/hash/mtime:BigInt/size/project_id/agent_id`。

### 代码片段
三区检索（`MemoryContextBootstrap.ts:167-224`）：
```ts
// Zone 1: 当前会话 session
if (conversationId && input.projectId) {
  const zone1Results = await memoryService.search({
    query: input.searchQuery, projectId: input.projectId, conversationId,
    sources: ['session'], limit: ZONE1_LIMIT, minScore: ZONE1_MIN_SCORE, hybrid: true,
  });
  results.push(...zone1Results.map(r => ({ ...r, isCrossConversation: false })));
}
// Zone 2: 项目 memory
if (input.projectId) {
  const zone2Results = await memoryService.search({
    query: input.searchQuery, projectId: input.projectId, agentId: input.agentId,
    sources: ['memory'], limit: ZONE2_LIMIT, minScore: ZONE2_MIN_SCORE, hybrid: true,
  });
  results.push(...zone2Results.map(r => ({ ...r, isCrossConversation: false })));
}
// Zone 3: 跨会话 session（仅当 Zone 1+2 不足 2 条时触发）
if (results.length < 2 && conversationId && input.projectId) {
  const zone3Results = await memoryService.search({
    query: input.searchQuery, projectId: input.projectId, excludeConversationId: conversationId,
    sources: ['session'], limit: ZONE3_LIMIT, minScore: ZONE3_MIN_SCORE, hybrid: true,
  });
  results.push(...zone3Results.map(r => ({ ...r, isCrossConversation: true })));
}
```

记忆索引双写 ES + PG（`MemoryIndexManager.ts:370-454`）：
```ts
private async indexContentCore(pathKey, content, source, projectId, agentId, fileMeta, logAs): Promise<number> {
  const contentHash = hashContent(content);
  const existing = await getMemoryFile(pathKey);
  if (existing && existing.hash === contentHash) {
    logger.debug(`[MemoryIndexManager] ${logAs === 'file' ? '文件' : '内容'}未变更，跳过: ${pathKey}`);
    return 0;   // hash 比较跳过未变更内容
  }
  if (existing) { /* 删旧 chunks + ES */ }
  const chunks = chunkText(content, pathKey, source, projectId, agentId);
  if (this.esAvailable) {
    try { await defaultMemoryVectorStore.addDocuments(chromaIds, documents, metadatas); }
    catch (esErr) { logger.warn(`[MemoryIndexManager] 写入 ES 失败，仅写入 PG: ${getErrorMsg(esErr)}`); }
  }
  await upsertMemoryFile(memoryFile);    // PG memory_files
  await upsertMemoryChunks(chunks);      // PG memory_chunks
}
```

### 设计要点
1. 三层记忆：(a) 会话层（ConversationService + Redis conversation:{id} 最多 50 条 + PG conversation_histories）；(b) 转录层（PG session_transcript 含 tool_call/thinking）；(c) 长期索引层（PG memory_chunks/memory_files + ES dense_vector）。
2. 分层检索（三区 Zone1/2/3）：Zone1=当前会话 session（3, 0.25）；Zone2=项目 memory（3, 0.5）；Zone3=跨会话 session（1, 0.8，仅 Zone1+2 不足 2 条时触发）。跨会话记忆标记 ⚠️。
3. 自动注入机制：injectLongTermMemory 每轮构造系统提示前自动检索（模型无需显式调 memory_search）。注入文本带类型标签 + 相关度分数 + 来源。
4. 会话→长期记忆自动转化：TranscriptSyncManager.markDirty → 防抖 10s → syncDirtyKeys → formatTranscriptAsMarkdown → indexContent(sessions)。三种触发：事件驱动（增量 10s 防抖）、定时兜底（30 分钟全量）、启动同步（全量）。
5. 混合检索 + 双写：hybridSearch = PG tsvector（BM25）+ ES dense_vector（cosine）加权。索引双写，失败降级仅 PG。hash 去重跳过未变更（0 开销）。
6. 员工维隔离 + 项目维过滤：memory_chunks.agent_id 按员工隔离。综合区（projectId='_general'）不传 projectId 检索全库。
7. 上下文可判定性：每轮注入有明确上限——记忆 top-k=5、对话历史 getRecentMemory(10)、工作上下文 getWorkingContext(10)。观测 llm_context_snapshot 事件。

## 路径修正提示
1. 八大 Role 目录是双层：基类/注册表 `agents/core/worker/role/`，七个业务 Role `agents/domain-harness/roles/`，第八类 external-agent 运行时由 DigitalEmployee.ts 动态生成。
2. 决策引擎：DecisionEngine.ts 是底层管线，对外入口是 architector/Architector.ts 单例。
3. Tool 调用循环不在 Turn 目录：实际在 `agents/core/ai-execution/tool-loop/StreamingToolExecutor.ts`。
4. L1/L2/L3 不是"决策阶段"而是"分层路由层"（DecisionRouteLayer）：L0=预决策、L1=候选能力快照、L2=异步 LayeredRouter、L3=readiness gate、Chat=闲聊。DecisionEngine 内部 5 阶段是另一维度。
