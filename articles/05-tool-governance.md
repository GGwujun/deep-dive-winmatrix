# 工具调用不止 function calling：企业级工具治理怎么做

> 这是《WinMatrix 开发经验文集》第 5 篇。LLM 想调一个工具，凭什么让它调？function calling 解决的是"怎么调"，企业场景要解决的是"凭什么调、调了之后呢、出事了谁负责"。这篇讲 WinMatrix 的工具治理体系。

大多数人理解 LLM 的工具调用（function calling / tool calling），停留在这样一个层面：

1. 定义一批工具的 schema（名字、参数、描述）
2. 喂给 LLM
3. LLM 决定调哪个、传什么参数
4. 执行，把结果塞回 LLM

跑通这个循环，demo 就成了。但你把它搬到企业里，立刻会撞上一堆问题：

- LLM 想调一个"删除数据库"的工具，你让不让？
- 同一个工具，产品经理能调，质量管理员能不能调？
- 工具执行产生了 50MB 的产物（比如生成的报告），怎么存、怎么传给 LLM？
- 调用出错了，错误信息怎么不泄露敏感信息又能帮 LLM 自我修正？
- 一次工具调用的全过程（谁调的、传了什么、返回了什么、耗时多久），事后怎么查？

这些问题，function calling 本身一个都不解决。它只管"怎么调"，不管"凭什么调、调了之后呢、出事了谁负责"。**这些才是企业级工具治理的核心。**

这篇文章讲 WinMatrix 怎么做。

---

## 第一步：工具不是裸函数，是有元数据的"受管资产"

我们先看一个工具在 WinMatrix 里长什么样。每个工具继承 `BaseTool`：

```typescript
// business-tools/base/BaseTool.ts（第 36-199 行）
export abstract class BaseTool implements ITool {
  abstract execute(args, context): Promise<ToolResult>;
  // ...
}
```

但一个工具远不止 `execute`。它还带一组**元数据**，描述这个工具"是什么、能干什么、有多危险"：

```typescript
// business-tools/base/interfaces.ts（第 65-87 行）
export interface ToolMetadata {
  isReadOnly: boolean;            // 只读吗？（不改变状态）
  isDestructive: boolean;         // 破坏性吗？（删除、覆盖）
  isConcurrencySafe: boolean;     // 并发安全吗？
  defaultTimeoutMs?: number;      // 默认超时
  declaredOperations?: string[];  // 显式声明的操作
  acceptsSkillCredentialEnv?: boolean;  // 接受技能凭证？
}
```

这几个字段不是装饰，它们直接决定了工具的治理方式：

- `isReadOnly=true` 的工具，可以更宽松地允许——它不改变状态，调错了也没事。
- `isDestructive=true` 的工具，要更严格的权限和确认——它可能造成不可逆的损害。
- `isConcurrencySafe=false` 的工具，不能并发执行，要排队。

**给工具打元数据标签，是工具治理的第一步。** 没有标签，所有工具都是"一样的裸函数"，你没法分级治理。

### scope：工具的可见范围

```typescript
// interfaces.ts（第 30 行）
export type ToolScope = 'role_default' | 'project_scoped' | 'agent_only';
```

工具分三种可见范围：

- **role_default**：某个角色的默认工具（如阿码默认有 code_review）
- **project_scoped**：特定项目才可见的工具
- **agent_only**：只有特定 Agent 能用

这决定了"谁能看到这个工具"。**不是所有工具都该对所有 Agent 可见——可见性本身就是一道权限。**

---

## 第二步：LLM 调工具的桥接，是两套接口

讲一个容易被忽略的工程细节。

LLM 客户端通常有两套调用方式：流式（stream）和非流式。很多人图省事，工具调用也走流式接口。

**这是错的。** 流式接口在工具调用场景下有一个致命问题——`tool_calls.arguments` 可能分多个 delta 返回，甚至偶尔返回空。

我们的 LLM 客户端为此准备了**双轨接口**：

```typescript
// infrastructure/llm/core/types.ts（第 344-399 行）
export interface ILLMClient {
  // 流式：用于对话，思考+文本实时推送
  stream(messages, options?): AsyncIterable<LLMStreamEvent>;
  // 非流式：用于工具调用场景，确保 tool_calls.arguments 完整
  invokeNonStreaming();
  // 真正 stream:false，用于工具调用场景，确保 tool_calls.arguments 完整
  withTools(tools): ILLMClient;
}
```

注释里直接写明了："真正 stream:false，用于工具调用场景，确保 `tool_calls.arguments` 完整"。

对话用 stream（体验好），工具调用用 invokeNonStreaming（正确性强）。**这是两种不同的诉求，不该硬塞进一个接口。** 想用一个万能 stream 接口同时搞定，必然在工具调用时踩空 args 的坑。

而且即使走了非流式，我们还有一道兜底——流式返回空 args 时，降级用非流式补全（见第 1 篇 Turn 引擎）。**承认模型的不稳定，用工程兜底。**

---

## 第三步：声明式操作授权——别让 LLM 钻空子

这是一个很微妙但极重要的安全设计。

很多系统的工具授权是基于"描述"或"关键词"的——LLM 想调 `delete_database`，系统扫一眼工具描述里有"删除"，就……让不让？这种基于自然语言推断的授权，是**可被绕过**的。LLM（或注入了恶意 prompt 的攻击者）可以用各种话术让推断逻辑误判。

WinMatrix 的做法是**声明式操作授权**：

```typescript
// BaseTool.ts（第 105-121 行）
toToolDefinition(): ToolDefinition {
  // ...
  this._toolDefCache = {
    name: this.name,
    description: this.getDescription(),
    parameters,
    declaredOperations: this.metadata.declaredOperations
      ?? getStaticToolDeclaredOperations(this.name),
  };
}
```

每个工具显式声明 `declaredOperations`（它能做的操作类型）。授权时**只认这个显式声明，不认描述、不认关键词推断**。

> 未声明时不得从描述/关键词推断。

这条规则的意义：**工具能做什么，由代码里的显式声明决定，不由 LLM 的描述决定。** 这把一个"可被 prompt 注入绕过"的安全判断，变成了"编译期就确定"的硬约束。

类似的还有 `acceptsSkillCredentialEnv`——它强约束"只有显式声明接受技能凭证的工具，才能拿到技能凭证环境变量"。**敏感资源的访问，必须基于显式声明，不能基于推断。**

---

## 第四步：工具策略——谁能在哪个项目用哪个工具

光有工具元数据和声明式授权还不够。企业里最常见的需求是：

> 这个工具，A 项目能用，B 项目不能用；产品经理能用，质量管理员不能用。

这是**项目级工具策略**：

```prisma
// prisma/schema.prisma（第 1414-1434 行）
model project_tool_policy {
  project_id          String
  role_id             String?             ← 可选：限定到角色
  digital_employee_id String?             ← 可选：限定到具体员工
  tool_name           String
  effect              String              ← allow | deny
  access_mode         String              ← member | visitor
  source              String              ← manual | role_default | system
  status              String              ← active | disabled
  @@index([project_id, status])
  @@index([project_id, role_id])
  @@index([project_id, digital_employee_id])
}
```

这张表的颗粒度非常细：

- **维度组合**：项目 × 角色 × 员工 × 工具。可以精确到"张三这个员工，在 A 项目里，不能用 delete 工具"。
- **allow/deny 双向**：既能显式允许，也能显式禁止。deny 优先于 allow（安全优先）。
- **member/visitor 区分**：项目成员和访客的工具权限不同。

**工具可见性的最终决定，是 `tool_config`（工具元数据）+ `project_tool_policy`（项目策略）+ `ToolScope`（范围）三者共同算出来的。** 不是单一来源。

为什么这么复杂？因为企业权限本来就是"项目 × 角色 × 人 × 资源 × 动作"的多维矩阵。简化它，要么牺牲安全，要么牺牲灵活。**与其假装简单，不如把这套矩阵做扎实。**

---

## 第五步：工具结果的两态封装——别把内部错误漏给 LLM

工具执行完，结果怎么返回给 LLM？最朴素的写法：

```typescript
return result;  // 直接返回
```

这有几个问题：

- 如果工具失败了呢？是抛异常、还是返回错误字符串？
- 如果结果带了不该给 LLM 看的内部信息（路径、密钥、堆栈）呢？
- 如果结果要写观测 span 呢？

我们的做法是**两态封装**：

```typescript
// business-tools/base/interfaces.ts（第 15-21 行）
export interface ToolResult {
  success: boolean;
  data?: unknown;
  error?: string;
  /** 写入 tool_call span attributes（经 pickToolSpanPersistedAttributes 白名单过滤） */
  spanAttributes?: Record<string, unknown>;
}
```

`success: true` 带 data，`success: false` 带 error。**成功的归成功，失败的归失败，结构清晰。**

而且失败时不是把原始错误丢给 LLM，而是走 `ErrorEnricher.enrich()`——把内部错误（比如数据库连接失败的堆栈）**加工成对 LLM 友好的、不泄露敏感信息的错误描述**。比如把"ECONNREFUSED to 10.0.0.5:5432"加工成"暂时无法访问数据存储，请稍后重试"。

**LLM 看到的错误，应该是它能理解、能据此自我修正的；不该是暴露内部架构的原始堆栈。** 这既是安全考虑（不泄露基础设施信息），也是效果考虑（LLM 看到"重试"比看到"TCP 连接被拒"更知道该怎么做）。

### spanAttributes：工具调用也要可观测

注意 `spanAttributes` 字段。工具执行的关键属性（耗时、影响的资源、关键参数摘要）通过它写入 tool_call 的 span。但写入前要经过 `pickToolSpanPersistedAttributes` **白名单过滤**——不是所有属性都该持久化，敏感的、过大的要过滤掉。

这和第 8 篇讲的 ExecutionSpan 是一套体系。**每一次工具调用，都是 span 树上的一个节点，可追溯、可审计。**

---

## 第六步：大块产物——别把 50MB 塞进对话

工具调用有时会产生大块产物——生成的一份报告、一张图、一段代码。这些怎么处理？

最糟糕的做法：把整个产物作为字符串塞进 ToolResult.data，让它进 LLM 上下文。50MB 的产物瞬间撑爆 token 上限。

我们的做法是**产物外置存储**：

```prisma
// prisma/schema.prisma（第 3597-3621 行）
model ToolResultArtifact {
  projectId        String
  conversationId   String?
  agentRunId       String?
  toolCallId       String?
  toolName         String?
  artifactKind     String
  contentType      String
  byteSize         Int
  sha256           String
  storageBackend   String   ← postgres
  storageKey       String?
  content          Bytes    ← 大块内容
  expiresAt        DateTime?   ← TTL 过期
}
```

大块产物存到 `ToolResultArtifact` 表（或对象存储），工具返回给 LLM 的只是一个**引用**（路径、摘要、关键片段），而不是全文。LLM 需要时，可以再调专门的"读制品"工具按需读取。

**这是 Agentic RAG 的思路在工具产物上的应用：不要一次把所有东西塞给 LLM，给它引用，让它按需深挖。** 既能处理大产物，又不爆 token。

而且产物带 `expiresAt` TTL——临时的中间产物会过期清理，避免无限堆积。带 sha256 保证完整性。

---

## 工具执行的全过程：executeToolCall 里发生了什么

把上面几点串起来，看一次 `executeToolCall` 的真实流程：

```typescript
// StreamingToolExecutor.ts（第 3001-3047 行）
// 1. 注入项目上下文
if (injectToolArgs) { toolArgs = injectToolArgs(toolName, toolArgs); }

// 2. Hook: beforeExecute（权限检查、参数校验的扩展点）
if (this.config.hookRegistry) {
  hookCtx = await this.config.hookRegistry.runBeforeHooks({
    toolName, args: toolArgs, context: this.config.toolExecutionContext,
  });
  if (hookCtx.args) toolArgs = hookCtx.args;
}

// 3. 观测：工具调用开始（写 span）
this.recordToolLoopObservabilityEvent({
  eventType: 'tool_call_start', toolName, toolCallId, toolInput: toolArgs,
});

// 4. 查找工具（不在可见集合里 → 拒绝）
const toolFn = tools.get(toolName);
if (!toolFn) { /* 拒绝 */ }

// 5. 执行 + 两态封装结果
// 6. 观测：工具调用结束（写 span）
```

注意几个治理点都在这里汇聚：

- **项目上下文注入**：工具拿到的 args 会被 `injectToolArgs` 补上项目上下文（projectId 等），工具不用自己从各处拼。
- **beforeExecute hook**：权限检查、参数校验、审计的前置钩子。这是个扩展点，企业可以插自己的治理逻辑。
- **可观测贯穿**：`tool_call_start` / `tool_call_end` 事件，每次调用都进 span。
- **可见集合校验**：`tools.get(toolName)` 找不到直接拒绝——工具必须先进入可见集合（经过 scope + policy 计算）才能被调用。

### 一个细节：为什么 span 关联要用 resolveOrCreateToolSpan

```typescript
// StreamingToolExecutor.ts（第 2989 行）
// Anthropic 早期 id 不稳定，不能直接做 span-source 关联
resolveOrCreateToolSpan(toolCallId, toolName)
```

注释点出一个真实的坑：Anthropic 早期返回的 `tool_call_id` 不稳定，不能直接拿来做 span 和源的关联。所以我们用 `resolveOrCreateToolSpan` 来"解析或创建"，把流式的 `tool_call_start` 事件的 draft span 和后续 `executeToolCall` 的 stable id 关联起来。

**可观测性要对抗的不只是"记什么"，还有"模型返回的东西不稳定"。** 这种细节不处理，span 树就会断。

---

## 给后来者的几条总结

1. **工具是带元数据的受管资产**，不是裸函数。isReadOnly/isDestructive/isConcurrencySafe 等标签决定治理方式。
2. **scope 决定可见性**。role_default / project_scoped / agent_only，可见性本身就是权限。
3. **LLM 调工具用非流式接口**，对话用流式。别用一个万能 stream 接口硬扛，空 args 的坑会咬你。
4. **声明式操作授权**。授权只认显式声明的 declaredOperations，不从描述/关键词推断——防 prompt 注入绕过。
5. **项目级工具策略是多维矩阵**。项目 × 角色 × 员工 × 工具 × allow/deny，别假装简单。
6. **工具结果两态封装**。success/error 分开，失败走 ErrorEnricher 加工，不把内部堆栈漏给 LLM。
7. **大块产物外置存储**。返回引用而非全文，让 LLM 按需深挖，避免爆 token。
8. **每次工具调用都进 span**。可观测 + 可审计，是对抗"模型不稳定"和"事后追责"的底线。

function calling 只是工具调用的起点。从一个 demo 级的 tool call，到一个企业级、可治理、可观测、可审计的工具执行，中间隔着元数据、声明式授权、多维策略、结果封装、产物外置、全链路 span 这一整套工程。**这些工程做得越扎实，你越敢把真正有力的工具交给 AI 用。**

---

> **下一篇**：[《技能（Skill）即数字员工的能力单元：从定义到治理全流程》](./06-skill-system.md)——工具是原子的，技能是工具 + 提示词 + 契约的封装。技能怎么管？
