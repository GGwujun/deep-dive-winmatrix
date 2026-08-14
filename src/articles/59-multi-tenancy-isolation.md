# 多租户隔离：同一个平台，不同项目怎么真正隔开

> 这是《WinMatrix 开发经验文集》第 59 篇，进入第三批的"跨界主题"段。前 40 篇里"项目（project）"这个词反复出现过——凭证绑定到项目、工具策略按项目配、记忆按项目分区检索。但从来没把"多租户隔离"作为一根线单独拎出来讲透。这篇就补这个缺口：一个企业级 AI 平台上，多个项目、多个团队共用一套引擎，怎么做到数据不串、工具不串、记忆不串、队列不串。多租户不是"加个 projectId 字段"那么简单，它是五个维度同时隔离的系统工程。

先讲一个真实会发生的故障。

某天，A 项目的管理员给"删除文档"工具配了一条 deny 策略，本意是只管自己项目。结果配置上线后，B 项目的数字员工也调不动删除工具了。排查发现：工具策略的查询语句 where 条件里漏了 `projectId`，一条 deny 把全平台都禁了。

或者更隐蔽的：A 项目用户跟数字员工聊天，聊的是 A 项目的需求；结果 Agent 检索记忆时，命中了 B 项目的某份 PRD 片段，把它拼进了 prompt。用户看到的回复里夹着别家的内部信息。这种"记忆串味"在生产里特别难查——输出看起来都对，就是偶尔"幻觉"出一些不该知道的东西。

这两个场景对应多租户隔离最核心的两条红线：**工具策略串租户**和**记忆串租户**。一旦突破，轻则功能错乱，重则数据泄露。WinMatrix 作为一个多项目共用引擎的平台，对这条线的处理不是"加 projectId"一刀切，而是按**数据 / 工具 / 记忆 / MCP / 队列**五个维度分别隔离，每个维度用最适合的机制。这篇逐维拆。

---

## 第一维：数据隔离——projectId 作为一切的主键

最底层是数据隔离。WinMatrix 的几乎所有业务表都带 `projectId`，它不是个普通字段，而是**数据归属的唯一标识**。

看几个核心模型的字段（核实报告 ch18-22）：

```
projects            id, name, code, pmdocPath, teamtaskPath, ...
tasks               id, projectId, taskId, taskName, ownerId, ...
knowledge_bases     id, projectId, name, type, ...
flow_template       id, projectId, name, status, ...
flow_run            id, projectId, templateId, ...
agent_run           id, conversationId, projectId, intentSummary, ...
```

注意 `agent_run` 也带 `projectId`——一次 Agent 执行的产物（span、token 消耗、工具调用）都能追溯到项目。这不是为了"查询方便"，是为了**成本归因和审计**（参考第 29 篇成本治理）：A 项目跑的 LLM 花的钱，不能算到 B 项目头上。

数据隔离的硬约束在**项目凭证绑定**这一层做得最彻底（schema.prisma:3566-3592，核实报告 ch13-17）：

```prisma
model ProjectSkillCredentialBinding {
  projectId       String   @map("project_id")
  skillTargetId   String   @map("skill_target_id")
  artifactDigest  String   @map("artifact_digest")
  requirementName String   @map("requirement_name")
  credentialId    String   @map("credential_id")
  credential      ProjectCredentialSecret @relation(fields: [credentialId, projectId], references: [id, projectId], ...)
  @@unique([projectId, skillTargetId, artifactDigest, requirementName])
}
```

注意 `credential` 这行外键：它是 `(credentialId, projectId)` **复合外键**，指向 `ProjectCredentialSecret` 的 `(id, projectId)`。这意味着在数据库层面，一个项目的凭证绑定**根本无法指向另一个项目的密文**——PG 的外键约束会直接拒绝。这不是应用层"记得加 where projectId"的纪律，是 schema 层焊死的硬隔离。

数据这一维的要点：

- **projectId 是数据归属的唯一真源**，几乎所有业务表都带。
- **敏感关系用复合外键**（credentialId + projectId），DB 层阻止跨项目引用。
- **查询路径都要过 projectId 过滤**，不能裸查。

---

## 第二维：工具隔离——project_tool_policy 的多维策略

数据隔开了，还要隔工具。同一个数字员工（比如小品）在 A 项目和 B 项目能调的工具可能不一样——A 项目允许她发邮件，B 项目因为合规原因禁止。

工具策略的载体是 `project_tool_policy` 表（schema.prisma:1414-1434，核实报告 ch13-17）：

```
project_tool_policy
├── project_id          项目（隔离主键）
├── role_id?            角色（可选，细化）
├── digital_employee_id? 员工（可选，再细化）
├── tool_name           工具
├── effect              allow | deny
├── access_mode         member | visitor
├── source              manual | role_default | system
└── status              active | disabled
```

这张表的颗粒度是**项目 × 角色 × 员工 × 工具 × allow/deny**。`project_id` 是第一维，所有策略查询都从这里开始。可以精确到"小品这个员工，在 A 项目里，不能用 delete 工具"，而 B 项目里同一个小品不受影响。

策略查询的层级（第 28 篇详讲）：

```
工具调用授权链
   ├── project_tool_policy WHERE project_id=? AND tool_name=?
   │     （项目级 allow/deny，deny 优先）
   ├── tool_config 的 scope（role_default | project_scoped | agent_only）
   └── 静态权限矩阵 + 动态 RBAC
```

关键约束：**deny 优先于 allow**。一个项目里只要有一条 deny 命中，不管有多少 allow，都拒绝。这是安全优先的设计——多租户场景下，宁可误杀不可放行。

这一维的要点：

- **工具策略以 project_id 为根**，跨项目天然不互扰。
- **颗粒度可细到员工级**，role/employee 都能单独配。
- **deny 优先**是多租户安全的底线。

---

## 第三维：记忆隔离——agent_id 划定记忆边界

这一维是 AI 平台特有的，也是最容易出"串味"问题的地方。数字员工的记忆（会话历史、长期记忆、项目知识库）必须严格按归属隔离，否则 prompt 里就会混入别家信息。

WinMatrix 的记忆分三层（核实报告 ch09-12，第 03 篇），每一层都有隔离边界：

```
记忆三层
├── 会话层（Redis conversation:{id} + PG conversation_histories）
│     隔离键：conversationId → projectId（conversation_meta.projectId）
├── 转录层（PG session_transcript）
│     隔离键：session_key（含 conversationId / agent 维度）
└── 长期记忆层（PG memory_chunks/memory_files + ES dense_vector）
      隔离键：projectId + agent_id
```

长期记忆的隔离尤其关键。`memory_chunks` 这类表带 `agent_id` 字段——**数字员工在 A 项目积累的记忆，物理上就跟 B 项目分开存**。检索时 where 条件必须带 projectId + agent_id，不能裸搜。

检索的三区分区（第 03 篇）也强化了这一点：

```
三区检索
├── Zone1：当前会话（session，3 条，权重 0.25）
├── Zone2：项目记忆（memory，3 条，权重 0.5）  ← 项目隔离在这
└── Zone3：跨会话（session，1 条，权重 0.8，仅前两区不足 2 条时触发）
```

Zone2 的"项目记忆"查询天然带 projectId 过滤，不会捞到别的项目的 chunk。Zone1/Zone3 是会话级，而会话本身就归属项目（conversation_meta.projectId），所以也是间接隔离。

知识库（knowledge_bases）同样以 projectId 隔离：A 项目的 Confluence 同步、文档分块、向量索引，都存在 A 项目名下。RAG 检索时必须指定项目上下文，跨项目搜索是要显式授权的（参考第 17 篇）。

这一维的要点：

- **记忆以 agent_id + projectId 物理隔离**，不是逻辑隔离。
- **检索 where 条件必须带项目维度**，裸搜是 bug。
- **会话通过 conversation_meta.projectId 间接归属项目**。

---

## 第四维：MCP 隔离——project_id + assigned_agents + tool_whitelist

MCP（Model Context Protocol）让外部工具服务接入平台，是多租户隔离的新战场。一个 MCP 服务（比如某个内部 API 桥接）可能是项目级的，也可能是全平台的，必须分清楚。

看 `mcp_services` 模型（schema.prisma，核实报告 ch18-22 主题 5）：

```
mcp_services
├── project_id           所属项目（隔离主键）
├── name / description   服务信息
├── transport_type       传输协议
├── url / api_key        接入地址与密钥
├── is_enabled           启用状态
├── is_builtin           是否内置（全平台）
├── tool_whitelist       工具白名单（JSON）
├── assigned_agents      分配给哪些 Agent（JSON）
└── credential_binding   凭证绑定（JSON）
```

四个隔离旋钮：`project_id`（这个服务属于哪个项目）、`is_builtin`（是不是全平台内置）、`tool_whitelist`（只暴露哪些工具）、`assigned_agents`（只让哪些 Agent 用）。

McpManager 初始化时并行连接所有启用的服务（`McpManager.ts:50-96`）：

```ts
async initialize(): Promise<void> {
  if (this.initialized) return;
  const services = await prisma.mcp_services.findMany({ where: { is_enabled: true } });
  await Promise.allSettled(services.map(svc => this.connectService(svc)));
  this.initialized = true;
  logger.info(`[${LOG_TAG}] 已初始化 ${this.clients.size}/${services.length} 个 MCP 服务`);
}
```

连接是全量的，但**调用时按 Agent 上下文过滤**——一个 Agent 要用某个 MCP 工具时，系统会查这个服务是不是分配给了它（assigned_agents）、工具在不在白名单里（tool_whitelist）、所属项目对不对。三个条件全过才放行。

外部 Agent（ext_ 前缀的虚拟数字员工）的熔断也是项目级的（`external_agent_pause` 模型，核实报告 ch18-22）：

```
external_agent_pause
├── projectId           项目
├── conversationId      会话
├── externalAgentId     外部 Agent
├── paused              是否熔断
├── reason              原因
└── pausedBy            操作人
```

A 项目把某个外部 Agent 熔断了，不影响 B 项目继续用它。熔断是项目级开关，不是全局开关。

这一维的要点：

- **MCP 服务以 project_id 为根**，配合 is_builtin 区分全平台服务。
- **tool_whitelist + assigned_agents 双重过滤**，调用时校验。
- **熔断是项目级**，一个项目挂了不影响别的项目。

---

## 第五维：队列隔离——运行时 isolationId 防 Redis 串

最后这一维不是业务隔离，是**运行时隔离**——防止开发机的任务串到生产 Redis，或者多实例之间的队列互相干扰。

BullMQ 队列跑在共享 Redis 上，如果两套环境用同一个队列名，开发机的 job 就会被生产 worker 消费，灾难性后果。WinMatrix 的解法是 `resolveBullmqQueueName`（`runtimeQueueIsolation.ts:48-59`，核实报告 ch04-06）：

```ts
// 生产强制 isolationId='prod'，非生产按 hostname
export function resolveBullmqQueueName(baseName: string): string {
  const isolationId = resolveRuntimeIsolationId();
  if (isolationId === 'prod') return baseName;                    // 生产用裸名
  return `${baseName}-host-${isolationId}`;                       // 非生产加 hostname 后缀
}
```

`resolveRuntimeIsolationId` 的逻辑（行 28-42）：生产环境 isolationId 必须是 `'prod'`，不是就直接抛错（行 35-39）；非生产环境用 hostname 自动生成 isolationId。结果就是：

```
环境          队列名
─────────────────────────────────────
生产          scheduled-agent
开发机A       scheduled-agent-host-macbook-air-1234
开发机B       scheduled-agent-host-lenovo-5678
```

每台开发机的队列名都带自己的 hostname 后缀，互不干扰。生产用裸名，所有生产实例共享一个队列（因为它们就是要分担负载）。

这一维虽然不涉及"项目"这个业务概念，但它是多实例共享 Redis 时的硬隔离——没有它，多租户的数据隔得再干净，队列层一串还是全完蛋。

这一维的要点：

- **生产 isolationId 强制 'prod'**，错了直接抛错 fail-fast。
- **非生产按 hostname 隔离**，每台机器队列名不同。
- **防的是"共享 Redis 串流"**，不是业务隔离。

---

## 五维放一起：综合才是多租户

把五个维度并列看：

```
多租户隔离五维
   │
   ├─【数据】projectId 主键 + 复合外键
   │     防数据串查、防凭证跨项目引用
   │     机制：DB schema 层硬约束
   │
   ├─【工具】project_tool_policy 多维策略
   │     防工具越权、防策略串租户
   │     机制：projectId × role × employee × tool，deny 优先
   │
   ├─【记忆】agent_id + projectId 物理隔离
   │     防 prompt 串味、防记忆泄露
   │     机制：检索 where 必带项目维度
   │
   ├─【MCP】project_id + assigned_agents + tool_whitelist
   │     防外部工具越界、防熔断蔓延
   │     机制：三重过滤 + 项目级熔断
   │
   └─【队列】runtime isolationId（hostname/prod）
         防 Redis 串流、防开发机任务进生产
         机制：队列名后缀 + 生产强制 prod
```

这五维是**正交的**，各防一类串流：

| 维度 | 防什么 | 如果缺失 |
|----|------|------|
| 数据 | 数据串查、凭证跨引用 | A 项目查到 B 项目数据，凭证被跨项目盗用 |
| 工具 | 工具越权调用 | A 项目的 deny 策略误伤 B 项目 |
| 记忆 | prompt 混入他项目信息 | 输出"幻觉"出别家内部信息 |
| MCP | 外部工具越界 | A 项目熔断的外部 Agent 影响 B 项目 |
| 队列 | Redis 串流 | 开发机任务被生产 worker 执行 |

**这五维缺一不可。** 只做数据隔离，记忆会串味；只做记忆隔离，工具策略会串；五维全做，多租户才真正成立。

---

## 为什么是五维而不是"加个 projectId"

新手会问：为什么不能所有表加个 projectId 字段就完事？答案在于**隔离的机制要匹配隔离的威胁**。

**第一，不同维度的"串"途径不同。** 数据串靠"where 漏了 projectId"；记忆串靠"检索没带项目过滤"；队列串靠"共享 Redis 同名队列"。一个 projectId 字段解决不了检索过滤和队列命名问题。

**第二，软约束守不住。** "记得所有查询带 projectId"是纪律，纪律守不住——总有新来的同事忘加，总有重构时漏掉。必须用 DB 外键（复合外键）、检索模板（必带 where）、队列名生成器（强制后缀）这种**机制层硬约束**兜底。

**第三，颗粒度差异。** 数据隔离是"项目级"；工具隔离要细到"项目 × 角色 × 员工"；MCP 隔离要叠加"工具白名单 + 分配 Agent"。一个 projectId 字段表达不了这种多维颗粒度，所以才需要 project_tool_policy 这种专门的策略表。

**所以"五维"的本质是"对不同途径的串流分别设防，每道防都用最合适的机制"。** 这和第 26 篇讲并发"别用一把锁解决所有问题"、第 23 篇讲幂等"别统一成一个服务"是一个道理——横切的是隔离意识，不是统一实现。

---

## 一个细节：凭证绑定的 digest 锁

凭证绑定这一层有个值得展开的设计——`artifactDigest` 字段。看绑定唯一键（schema.prisma:3590）：

```prisma
@@unique([projectId, skillTargetId, artifactDigest, requirementName])
```

绑定不是"项目 × 技能 × 凭证需求"，而是"项目 × 技能 × **技能版本 digest** × 凭证需求"。这意味着：技能升级后（digest 变了），旧的凭证绑定**自动失效**，必须重新绑定。

为什么这么设计？因为技能的 manifest 声明哪些凭证需求，是跟版本绑定的——v1 可能需要 API_KEY，v2 可能改成需要 TOKEN。如果绑定不锁 digest，技能从 v1 升到 v2 后，v2 的代码拿到的可能是 v1 声明的 API_KEY，类型不对、用途不对，轻则报错重则误用。

digest 锁强制每次技能升级都要重新确认凭证绑定——这是个"安全优于便利"的抉择。digest 变化时旧行保留（用于显示 stale 和审计），但 readiness 只接受当前 digest 的精确绑定（schema 注释原文）。**宁可让技能暂时跑不起来等重新绑凭证，也不能拿错凭证瞎跑。**

这个细节体现的是多租户隔离的深层逻辑：**隔离不只是"隔开不同租户"，还要"隔开同一租户的不同版本"**。时间维度上的隔离和空间维度上的隔离同样重要。

---

## 给后来者的几条总结

1. **多租户隔离是五维系统工程，不是加个字段。** 数据、工具、记忆、MCP、队列五个维度分别隔离，各用最合适的机制。
2. **数据隔离靠 projectId + 复合外键**。敏感关系（凭证绑定）用 (credentialId, projectId) 复合外键，DB 层阻止跨项目引用，不靠应用层纪律。
3. **工具隔离靠 project_tool_policy 多维策略**。颗粒度细到项目 × 角色 × 员工 × 工具，deny 优先于 allow。
4. **记忆隔离靠 agent_id + projectId 物理隔离**。检索 where 必带项目维度，裸搜是 bug；记忆串味是最难查的多租户事故。
5. **MCP 隔离靠 project_id + assigned_agents + tool_whitelist 三重过滤**。熔断是项目级，一个项目挂了不影响别的。
6. **队列隔离靠 runtime isolationId**。生产强制 'prod'，非生产按 hostname，防共享 Redis 串流。
7. **五维正交，缺一不可**。只做一两维，其他维度的串流照样炸。
8. **凭证绑定锁 digest**：技能升级自动失效旧绑定，宁可暂时跑不起来也不拿错凭证。时间维度的隔离和空间维度一样重要。
9. **横切的是隔离意识，不是统一实现**。每个维度单独设防，比抽象一个 TenantIsolationService 更灵活可靠。

多租户这件事，demo 阶段单项目跑什么都对；一上生产多项目共存，五维问题同时爆。提前把这五个维度想清楚、每维都用机制层硬约束兜住，比事后补"又串了"的窟窿强得多。

---

> **下一篇**：[《凭证管理：密钥/Token 在系统里怎么流转才安全》](./60-credential-management.md)——多租户隔离讲完了，紧接着的一个问题是：项目里的密钥、Token、API key 这些凭证，存在哪、怎么加密、怎么绑定到技能、怎么传给工具才不会被泄露。讲 WinMatrix 的凭证管理体系。
