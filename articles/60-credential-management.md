# 凭证管理：密钥/Token 在系统里怎么流转才安全

> 这是《WinMatrix 开发经验文集》第 60 篇。第 28 篇讲安全护栏时提过一句"token 不存明文，只存 SHA-256 hash""secret 用 aes-256-gcm 加密"，但没展开。这篇就专门、系统地讲凭证管理——API key、Token、密码这些敏感凭据，从用户录入、到落库、到绑定技能、到运行时注入工具，整条流转链怎么设计才不会泄露。这是个横跨数据库、加密、技能治理、工具执行的跨界主题，也是企业级 AI 平台绕不开的合规底线。

先讲凭证管理最容易踩的三个坑。

**坑 1：明文落库。** 为了"方便"，把第三方 API key 直接明文存数据库。一旦 DB 被拖库，所有外部系统的凭证一把端走。这在传统后端就不可接受，在 AI 平台更危险——因为 Agent 运行时会把这些凭证当环境变量塞进工具，泄露途径更多。

**坑 2：审计日志里留了明文。** 凭证存得挺安全（加密了），但用户更新凭证时，审计日志把请求体原样记下来了——明文凭证就这么进了 ES，谁有日志查询权限谁就能看到。这种"加密了存储却明文记日志"的漏洞极常见。

**坑 3：凭证被串用。** A 技能声明需要 API_KEY，系统去解析时"聪明"地做了 fallback——找不到 A 技能专属的凭证，就拿项目里同名的顶上。结果 A 技能拿到的可能是给 B 技能准备的凭证，权限范围不对，要么权限过大要么用错 key。

WinMatrix 对这三个坑的应对是一套完整的凭证管理链：**加密存储（aes-256-gcm）+ hash 索引（SHA-256）+ 显式声明绑定 + 审计脱敏 + L3 硬约束解析**。核心原则一句话：**需还原用的加密，需验证用的 hash；凭证绑定到技能版本；不确定必须 fail-closed**。这篇逐段拆。

---

## 存储层：加密 vs hash，各管一种用途

凭证落库的第一道设计抉择：**什么时候加密、什么时候 hash**。这两个不是一回事，搞混了要么泄露要么用不了。

WinMatrix 的原则清晰：

- **需要还原明文使用的凭证 → 加密**（AES-256-GCM），因为运行时要把明文塞进环境变量给工具用。
- **只需要验证"对不对"的凭证 → hash**（SHA-256），因为验证时比 hash 就行，永远不需要还原。

看几类典型凭证的存储方式（核实报告 ch04-06 主题 3、ch13-17、ch18-22）：

```
凭证类型              存储方式           原因
─────────────────────────────────────────────────────
项目技能凭证           aes-256-gcm 加密   运行时要还原成明文塞 env
(ProjectCredentialSecret.secretCiphertext)

Confluence PAT        aes-256-gcm-v1     调 Confluence API 要还原
(project_confluence_credentials)

JWT token             不存（只签发）      签发即用，无需存

PAT 个人访问令牌       SHA-256 hash       验证时 hash 比对即可
(personal_access_tokens.tokenHash)

WMA 注册 apiKey       apiKeyHash + apiKeyEncrypted
(external_agent_registration)   hash 用于查找，encrypted 用于调用

企微 AiBot secret     secretCiphertext   长连接鉴权要还原
(UserWecomAibotBinding)
```

注意 `external_agent_registration` 那行——它同时有 `apiKeyHash` 和 `apiKeyEncrypted` 两个字段。这不是冗余，是**两种用途并存**：查表时用 hash（"这个 key 在不在库"），实际调用时用 encrypted（还原明文发请求）。一个字段干不了这两件事。

项目级技能凭证的存储模型（schema.prisma:3532-3557）：

```prisma
model ProjectCredentialSecret {
  id               String   @id @default(cuid())
  projectId        String   @map("project_id")
  name             String                     // 管理员可识别名称，同项目唯一
  secretCiphertext String   @map("secret_ciphertext")   // AES-256-GCM 密文
  cipherVersion    String   @default("aes-256-gcm-v1") @map("cipher_version")
  ...
  @@unique([projectId, name])
}
```

schema 注释（行 3527-3531）写得很明确：

> 只保存 AES-256-GCM ciphertext 与 cipher version，不保存 plaintext、masked secret 或通用 JSON secret。删除采用物理删除并由独立审计记录留痕；只要存在任何 binding（包括 stale digest），就拒绝删除。展示只使用固定 `[SET]`。

几个要点：

- **密文格式固定**：`hex:iv:authTag:ciphertext`（注释行 3538）。GCM 模式带认证标签，防篡改。
- **不存明文、不存掩码**：连 `sk-****1234` 这种掩码都不存，因为掩码本身也是信息泄露。
- **展示固定 `[SET]`**：UI 上只显示"已设置"，不显示任何内容。这跟 PAT 的"只存 hash"逻辑一致——能不展示就别展示。
- **删除是物理删除**：不留软删除的"已删除凭证"行（会被拖库），但留独立审计记录。
- **有 binding 拒绝删除**：只要还有技能在用这个凭证（哪怕 stale digest），就不能删，防误删导致技能跑不起来。

这一层的核心思想：**DB 里永远不存明文，展示永远不给内容，删除永远留审计**。

---

## 绑定层：凭证绑定到技能版本（digest）

凭证存好了，还要绑给技能用。绑定关系就是上一章讲过的 `ProjectSkillCredentialBinding`（schema.prisma:3566-3592）。这里从凭证管理角度再串一遍。

```
ProjectSkillCredentialBinding
├── projectId          项目
├── skillTargetId      技能名
├── artifactDigest     技能版本 sha256
├── requirementName    凭证需求名（manifest 声明）
└── credentialId       指向 ProjectCredentialSecret（复合外键）
```

几个关键设计（核实报告 ch13-17 主题 1，第 59 篇详讲过）：

**复合外键防跨项目**：`credential` 关系是 `fields: [credentialId, projectId] references: [id, projectId]`，DB 层阻止 A 项目的绑定指向 B 项目的密文。

**digest 锁版本**：绑定唯一键含 `artifactDigest`。技能升级（digest 变）后旧绑定失效，必须重新绑。schema 注释原文（行 3562-3565）：

> 绑定唯一键固定为 (projectId, skillTargetId, artifactDigest, requirementName)。使用 (credentialId, projectId) 复合外键指向 secret 的 (id, projectId)，在数据库层阻止跨项目绑定。digest 变化时旧行保留用于显示 stale 和审计，但 readiness 只接受当前 digest 的精确绑定。

**为什么这么严？** 因为技能的 manifest 声明凭证需求，是跟版本绑定的。v1 声明需要 API_KEY，v2 可能改成 TOKEN；如果绑定不锁版本，v2 代码可能拿到 v1 的 API_KEY，类型和用途都不对。digest 锁强制每次升级重新确认——安全优于便利。

**绑定多对一**：一个凭证密文（ProjectCredentialSecret）可以被同项目多个技能需求绑定（`bindings ProjectSkillCredentialBinding[]`），但一个绑定只指一个密文。这样"一份 Confluence PAT 给项目里三个技能共用"是允许的，但跨项目共用是被 DB 外键挡死的。

---

## 解析层：L3 SkillReadinessGate 的硬约束

凭证绑好了，运行时怎么取出来给工具用？这是凭证管理最敏感的一环——明文要从 DB 取出、解密、塞进环境变量、传给工具执行上下文。任何一个环节"聪明地"做 fallback，都可能让技能拿到不该拿的凭证。

WinMatrix 把这一层放在 L3 readiness gate（`SkillReadinessGate.ts`，核实报告 ch13-17），核心约束是 `SkillCredentialResolutionService`（`SkillCredentialResolutionService.ts:40-152`）。

函数签名（行 40-54）要求三个 canonical 字段强制必填：

```ts
export async function resolveSkillCredentialRequirements(input: {
  /** canonical 项目 ID（必填，与 Port 接口一致） */
  scopeProjectId: string;
  /** 当前 Skill target ID（必填） */
  skillTargetId: string;
  /** 当前已准入 artifact 的 sha256 digest（必填） */
  artifactDigest: string;
  ownerUserId?: string;
  requirements: SkillCredentialRequirementInput[];
  deps?: SkillCredentialResolverDeps;
}): Promise<SkillCredentialResolutionResult> {
```

注意三个字段都标了"必填"，而且注释强调"canonical 项目 ID（不使用路由别名/项目名）"。这意味着：**绝不用 URL 里的项目代号、绝不用技能别名、绝不靠任何间接信息推projectId**，只认从准入流程传下来的 canonical id。

文件头部的注释（行 36-39）把硬约束说得很死：

> projectCredential source 存在时，canonical 字段（scopeProjectId、skillTargetId、artifactDigest）必须齐全；缺失直接标记 missing/unsupported_source，不使用 projectId fallback、不使用路由参数或别名回退。

**不做 fallback 是核心**。看实现（行 63-98）：

```ts
if (projectCredentialReqs.length > 0) {
  const resolver = input.deps?.projectCredentialResolver;
  if (!resolver) {
    // 未注入 resolver：标记 unsupported_source（feature flag 关闭时）
    for (const req of projectCredentialReqs) {
      summaries.push({ ..., status: 'unsupported_source' });
      if (req.required) ok = false;   // 必填凭证未解析 → 整体失败
    }
  } else {
    // canonical 字段由 Port 接口强制必填，直接使用
    const outcome = await resolver.resolve({
      scopeProjectId: input.scopeProjectId,
      skillTargetId: input.skillTargetId,
      artifactDigest: input.artifactDigest,
      requirements: projectCredentialReqs,
    });
    ...
    if (!outcome.allRequiredResolved) ok = false;
  }
}
```

resolver 没注入？标记 `unsupported_source`，必填的话整体 fail。解析出来缺了某个必填？`allRequiredResolved=false`，整体 fail。**从不"找个替代品凑合用"**。

返回结构（行 151）：

```ts
return { ok, summaries, transientEnv: ok ? transientEnv : {} };
```

**注意 `transientEnv: ok ? transientEnv : {}`**——只要有一个必填凭证没解析成功（ok=false），整个明文环境变量表就清空返回。这是个"要么全有要么全无"的设计：宁可让技能因为缺凭证跑不起来（报错给用户看），也不能让它拿着半套凭证瞎跑（可能误用部分凭证）。

`transientEnv` 本身是**纯内存**（transient），注释行 8-9：

> 同一次 step 对声明的凭证需求只解析一次，输出可诊断 summary（不含明文）与仅内存 transientEnv（明文，只允许进入当前进程/当前 Action 的执行上下文）。

明文只在内存里，只活在当前这一个 Action 的执行周期内，Action 结束就 GC。不落日志、不落 DB、不进 span。

这一层的核心：**canonical 字段强制必填 + 不做任何 fallback + 全有或全无 + 明文纯内存**。

---

## 注入层：只有声明了的工具才能拿凭证

凭证解析出来成了 `transientEnv`，怎么给工具？这又是一道隔离——不是所有工具都能拿凭证，必须显式声明。

BaseTool 的元数据里有 `acceptsSkillCredentialEnv` 字段（`interfaces.ts:65-87`，核实报告 ch13-17 主题 2）：

```ts
interface ToolMetadata {
  isReadOnly: boolean;
  isDestructive: boolean;
  isConcurrencySafe: boolean;
  defaultTimeoutMs?: number;
  declaredOperations?: string[];
  acceptsSkillCredentialEnv?: boolean;   // 只有 true 的工具才接凭证 env
  ...
}
```

**默认 false**。只有工具作者在代码里显式声明 `acceptsSkillCredentialEnv: true`，这个工具才有资格接收技能凭证环境变量。这是"敏感资源访问必须基于显式声明"的原则（第 28 篇讲过）——跟 `declaredOperations` 防注入是一个思路：把"可被 prompt 注入绕过的软判断"变成"编译期就确定的硬约束"。

为什么要这层隔离？因为 LLM 是会越权的。如果任何工具都能拿凭证，prompt 注入让 LLM 调一个"看起来无害"的工具（比如读文件、发请求），凭证就可能被这个工具带出去。限制了"只有声明接受凭证的工具"才能拿到，攻击面就大幅收窄——LLM 没法让一个本来不该碰凭证的工具去泄露凭证。

配合的还有声明式操作授权（`declaredOperations`，BaseTool.ts:105-121）：工具能做什么操作是编译期写死的，LLM 改不了。两个机制叠加，凭证泄露路径被卡得很死。

---

## 审计层：脱敏是硬约束，不是可选

凭证管理还有一层容易被忽视的：审计日志。前面说过坑 2 是"审计日志留了明文"。WinMatrix 的 `apiAuditLog` 中间件对此有明确的脱敏逻辑（`apiAuditLog.ts:178-201`）：

```ts
/**
 * OpenSpec: manage-skill-required-credentials §4.5
 * 脱敏请求/响应体中的敏感字段（value、secretCiphertext、token、apiKey 等）。
 * 替换为 [REDACTED]，不保留原值或长度。
 */
const SENSITIVE_BODY_KEYS = new Set([
  'value', 'secretCiphertext', 'secret', 'token', 'pat',
  'personalAccessToken', 'apiKey', 'api_key', 'password', 'passwd',
]);

function redactSensitiveBodyFields(value: unknown): unknown {
  if (Array.isArray(value)) return value.map(redactSensitiveBodyFields);
  if (!value || typeof value !== 'object') return value;
  const redacted: Record<string, unknown> = {};
  for (const [key, child] of Object.entries(value)) {
    redacted[key] = SENSITIVE_BODY_KEYS.has(key)
      ? '[REDACTED]'                       // 命中敏感 key → 替换
      : redactSensitiveBodyFields(child);  // 递归
  }
  return redacted;
}
```

几个设计要点：

**字段名黑名单 + 递归**：不管请求体嵌套多深，只要 key 命中黑名单（value / secretCiphertext / token / apiKey / password 等），值就替换成 `[REDACTED]`。递归处理，防"藏在嵌套对象里"。

**替换不保留长度**：注释行 175 "替换为 [REDACTED]，不保留原值或长度"。这意味着不能从 `[REDACTED]` 的长度推断原值长度——所有敏感字段统一替换成同一个标记，零信息泄露。

**注释引用了 OpenSpec 契约**：脱敏规则不是临时想出来的，是 `manage-skill-required-credentials §4.5` 契约规定的。契约驱动意味着这条脱敏逻辑有明确的 SSOT，改的时候要改契约，不能偷偷放宽。

**请求头也脱敏**：`sanitizeHeaders` 对 Authorization 统一替换成 `Bearer [REDACTED]`（行 147-152）。Token 不会因为藏在 header 里就漏掉。

这一层的核心：**凭证管理不能只管 DB，还要管日志、管 header、管所有可能留痕的地方**。脱敏是硬约束，所有审计中间件强制走，不是"记得调一下"。

---

## 凭证全生命周期：一张图

把存储、绑定、解析、注入、审计串成一条链：

```
用户录入凭证
   │
   ▼
【存储层】AES-256-GCM 加密 → ProjectCredentialSecret.secretCiphertext
   │     格式 hex:iv:authTag:ciphertext，展示固定 [SET]
   │     hash 索引用于查找（external_agent_registration.apiKeyHash）
   │
   ▼
【绑定层】ProjectSkillCredentialBinding
   │     唯一键 (projectId, skillTargetId, artifactDigest, requirementName)
   │     复合外键 (credentialId, projectId) 防跨项目
   │     digest 锁版本，升级失效
   │
   ▼
【解析层】L3 SkillCredentialResolutionService
   │     canonical 字段强制必填，不做 fallback
   │     全有或全无：ok=false 时 transientEnv 返回空
   │     明文纯内存（transientEnv），不落库不落日志
   │
   ▼
【注入层】只给 acceptsSkillCredentialEnv=true 的工具
   │     配合 declaredOperations 编译期硬约束
   │     Action 结束 transientEnv 随 GC
   │
   ▼
【审计层】apiAuditLog 脱敏
         SENSITIVE_BODY_KEYS 黑名单 + 递归 [REDACTED]
         请求头 Authorization → Bearer [REDACTED]
```

这条链的每一步都假设其他步可能出错：存储假设 DB 被拖（所以加密）；绑定假设有人想跨项目（所以复合外键）；解析假设字段会缺（所以 fail-closed）；注入假设 LLM 想越权（所以显式声明）；审计假设有人会看日志（所以脱敏）。**纵深防御，每层都假设其他层会失守。**

---

## 为什么 L3 解析必须 fail-closed

把 L3 这一层单独拎出来强调，因为它是凭证管理最容易"好心办坏事"的地方。

设想一个"聪明"的实现：技能声明需要 API_KEY，系统去找绑定时发现没有，于是"贴心地"去项目里找名为 API_KEY 的通用凭证顶上，或者用项目根的某个默认 token。听起来很方便，实际上两个致命问题：

**第一，权限范围不对。** A 技能声明的 API_KEY 可能是只读权限，你 fallback 拿来的项目根 token 可能是读写权限——技能本该只读的操作，现在能写了。权限放大。

**第二，用途不对。** A 技能声明的 TOKEN 是给 Confluence 用的，你 fallback 拿来的可能是给 Jira 的。技能拿着 Confluence token 去调 Jira，要么报错要么更糟——误调成功造成副作用。

WinMatrix 的设计是**宁可让技能跑不起来**。`ok=false` 时 `transientEnv` 清空，技能直接失败，用户看到"凭证未绑定"的错误，去后台补绑定。这比"拿错凭证瞎跑"安全得多。

这是凭证管理的核心抉择：**便利 vs 安全，安全永远赢**。跟第 28 篇讲的"安全护栏必须 fail-closed"完全一致——不确定时拒绝，不凑合。

---

## 给后来者的几条总结

1. **加密和 hash 不是一回事**。需还原用加密（AES-256-GCM），需验证用 hash（SHA-256）；查表用 hash、调用用 encrypted 可以并存（external_agent_registration 的双字段）。
2. **密文格式固定、展示固定**。AES-256-GCM 用 `hex:iv:authTag:ciphertext`，UI 展示固定 `[SET]`，不存明文不存掩码。
3. **凭证绑定到技能版本（digest）**。复合外键防跨项目，digest 锁防版本错位，升级自动失效要重新绑。
4. **L3 解析必须 fail-closed**。canonical 字段强制必填，不做任何 fallback，全有或全无——宁可跑不起来也不拿错凭证。
5. **明文纯内存（transientEnv）**。不落库不落日志不进 span，Action 结束随 GC。
6. **只有声明了的工具才能拿凭证**。`acceptsSkillCredentialEnv: true` 是编译期硬约束，配合 `declaredOperations` 防 LLM 越权。
7. **审计脱敏是硬约束不是可选**。字段名黑名单 + 递归 [REDACTED]，不保留长度；header 的 Authorization 也要脱敏。
8. **纵深防御，每层假设其他层失守**。存储防拖库、绑定防跨项目、解析防缺字段、注入防越权、审计防看日志——五层叠加。
9. **便利 vs 安全永远选安全**。凭证管理没有"贴心 fallback"，不确定就拒绝。

凭证管理是个不性感的活，但它是企业级 AI 平台的合规底线。一次泄露——不管是 DB 拖库、日志外泄、还是技能拿错 key——都是灾难级事故。把这条链的五层都做扎实，平台才有"敢存敏感凭证"的资格。

---

> **下一篇**：[《审计日志：谁在什么时候对配置做了什么》](./61-audit-logging.md)——凭证管理讲完了，更广的问题是：平台上所有配置变更、所有写操作，都要能追溯"谁、什么时候、改了什么"。讲 WinMatrix 的三层审计日志体系。
