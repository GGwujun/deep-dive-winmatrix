# 知识库入库 pipeline：从 PDF 到向量分块的全链路

> 这是《WinMatrix 开发经验文集》第 20 篇。讲一个看起来简单实则全是坑的问题：用户往知识库上传一份 PDF，到这份 PDF 能被 AI 检索到，中间发生了什么？解析、分块、向量化、双写，每一步都有坑。代码来自 WinMatrix 后端真实实现。

很多人做 RAG（检索增强生成），核心循环写得很快：用户提问 → 向量检索 → 把检索到的片段塞进 prompt → LLM 回答。跑通了，效果还行，挺开心。

但你上传一份新文档进去，问它"这份文档里说了什么"，它答不上来。你开始排查——是向量化的模型不对？是检索的阈值太严？是 chunk 切得不好？查了一圈发现：**文档根本没被正确分块，或者分块了但没写进向量库，或者写进去了但 PG 里的元数据对不上**。

检索那一步大家都很重视，因为它直接决定回答质量。但入库 pipeline 这一步常被忽视——它决定了"你的知识库里到底有没有正确的东西可检索"。一份文档从上传到可检索，要经历一条完整的 pipeline，任何一步出问题，后面的检索都是空中楼阁。

这篇文章拆解 WinMatrix 的知识库入库 pipeline，看每一步是怎么做的。核心思路：**分块要文档类型感知、双写保证一致、原件一定要留兜底**。

---

## 整条 pipeline 的骨架

先看全景。WinMatrix 的入库入口是 `RagService.ingestBufferDetailed`，一份文档进来要走这几步：

```
用户上传 buffer（PDF/Word/Markdown/Excel）
       │
       ▼
1. 大小校验 + 内容哈希去重（md5，未变更则 skip）
       │
       ▼
2. parserRegistry.parse → 按文件类型解析成 sections
       │
       ▼
3. 选分隔符：用户自定义 > 文档类型默认
       │
       ▼
4. 分块：parent_child 双模式 or flat，文档感知分隔符 + 保护区间
       │
       ▼
5. 图片处理：独立图块 + 内联 data URI 都转成 URL 落盘
       │
       ▼
6. 分块（资产替换后再跑一次）→ ABSOLUTE_MAX_CHUNK_SIZE 上限校验
       │
       ▼
7. 预清理旧数据 + 重存原件（re-ingest 兜底）
       │
       ▼
8. 双写：upsertRagChunks（ES 向量）+ syncChunksToDb（PG 元数据）
       │
       ▼
9. 异步增强：图谱抽取（Neo4j）+ 问题生成（FAQ）
```

每一步都有设计依据，我们逐段拆。

---

## 第一步：内容哈希去重，别重复处理同一份文档

pipeline 的入口先做两件事：大小校验和哈希去重。

```typescript
// src/infrastructure/rag/RagService.ts（ingestBufferDetailed，第 178-189 行）
if (buffer.length > ragIngestConfig.maxBytes) {
  throw new Error(
    `文件大小 (${(buffer.length / 1024 / 1024).toFixed(1)} MB) 超过限制 (${(ragIngestConfig.maxBytes / 1024 / 1024).toFixed(0)} MB)`,
  );
}

const contentHash = crypto.createHash('md5').update(buffer).digest('hex');
const cachedHash = this.documentHashes.get(documentPath);
if (cachedHash === contentHash) {
  logger.debug(`[RagService] 文档未变更，跳过: ${documentPath}`);
  return { status: 'skipped', chunkCount: 0 };
}
```

内容哈希用 md5 算 buffer 的指纹，缓存在内存 Map 里。如果同一份文档（内容完全一样）再次进来，直接 skip，不重复走 pipeline。这看起来是个小优化，但在实际场景里很重要——知识库同步（比如 Confluence 定时同步）会反复扫描同一批文档，没有去重的话，每次都重新解析、分块、向量化，白白烧 embedding API 的配额。

**教训：任何"会被重复触发"的 pipeline，入口都要做幂等判断。** 哈希去重是最便宜的一种——算个 md5 比走完整 pipeline 便宜几个数量级。

---

## 第二步：文档类型感知的分隔符

分块是整条 pipeline 最核心、也最容易做歪的一步。WinMatrix 的分块器有个关键设计——**不同文档类型用不同的分隔符**：

```typescript
// src/infrastructure/rag/DocumentChunker.ts（第 25-50 行）
const DEFAULT_SEPARATORS = ['\n\n', '\n', '。', '！', '？', ';', '；', '.'];

const MARKDOWN_SEPARATORS = [
  '\n\n\n',   // 章节分隔（优先级最高）
  '\n\n',     // 段落分隔
  '## ',      // 二级标题（保留标题在块中）
  '### ',     // 三级标题
  '#### ',    // 四级标题
  '\n',       // 行分隔
  '。', '！', '？', ';', '；', '.',  // 句子分隔
];

const PDF_SEPARATORS = ['\n\n', '\n', '。', '!', '?', '.'];

const EXCEL_SEPARATORS = ['\n', '|', '\t', ','];
```

为什么要分文档类型？因为不同格式的文档，"自然的语义边界"完全不同：

- **Markdown** 有明确的标题层级（`##`/`###`/`####`），分块应该优先在标题处切，保证一个块不会跨章节。所以 MARKDOWN_SEPARATORS 里把标题分隔符放在前面（高优先级）。
- **PDF** 提取出来的文本通常很破碎（换行多、段落边界模糊），分隔符要简单一点，避免过度切碎。
- **Excel** 本质是表格，分隔符用 `|`、`\t`、`,` 这种表格天然的边界。

注意分隔符是**有优先级顺序的**——排在前面的先尝试。分块器会先尝试用最高优先级的分隔符（比如 Markdown 的 `\n\n\n`）切，如果切出来的块还是太大，再用次优先级的分隔符继续切，直到块大小符合要求。这就是"递归字符分块"（recursive character splitting）的核心。

而在 pipeline 里，分隔符的选择逻辑是：**用户自定义 > 文档类型默认**：

```typescript
// RagService.ts（第 209-213 行）
let separators = kbChunkingCfg?.splitMarkers;        // 用户在知识库配置里自定义
if (!separators) {
  separators = getSeparatorsByType(parseResult.documentType);  // 按文档类型默认
  logger.debug(`[RagService] 使用文档类型分隔符: ${parseResult.documentType} → ${separators.length} 个分隔符`);
}
```

这样既给了用户"我这份文档很特殊，要按我自己的规则切"的灵活性，又给了"大部分文档按类型默认就行"的省心。

**教训：分块不是"按固定长度切字符串"，而是"按语义边界切语义单元"。** 一刀切 800 字符最容易，但会把一个完整的段落从中间切断，检索到一半的内容毫无意义。文档类型感知的分隔符，是让"块"尽量承载完整语义的第一道保障。

---

## 第三步：保护区间——代码块、表格、LaTeX 不能被切断

光有分隔符还不够。有些内容天生不该被任何分隔符切断——代码块、表格、LaTeX 公式。如果一段代码被从中间切断，检索到半截代码，对回答毫无帮助甚至有害。

DocumentChunker 的文件头注释明确点出了这个设计：

```typescript
// src/infrastructure/rag/DocumentChunker.ts（第 1-13 行，文件头注释）
/**
 * 文档感知分块器（增强版）
 *
 * 核心算法移植自 WeKnora splitter.go，增加：
 * - 保护区间：LaTeX / 代码块 / 表格 / Markdown 图片与链接不会被截断
 * - 可配置分隔符：按优先级拆分，保留分隔符
 * - 绝对最大块上限（7500 字符）：防止 embedding API 超限
 * - 改进的 overlap 计算：按 unit 粒度回溯，跳过纯分隔符
 * - Parent-child 分块复用同一核心算法
 */
```

分块器在遍历文本时，会先识别出这些"保护区间"（代码块用三反引号包裹、LaTeX 用 $ 包裹、表格用 | 对齐），把它们当作不可分割的原子单元。即使一个代码块有 600 字符、超过了目标 chunk size，也会整体保留在一个块里，不会从中间切开。

这一点对代码相关的知识库尤其重要。想象你的知识库里有几十段代码示例，如果分块器把每段代码切成两半，检索时只能拿到前半截或后半截，LLM 拿到的上下文就是残缺的，回答必然出错。

### ABSOLUTE_MAX_CHUNK_SIZE：防止 embedding API 超限

```typescript
// DocumentChunker.ts（第 24 行）
const ABSOLUTE_MAX_CHUNK_SIZE = 7500;
```

除了用户配置的目标 chunk size（比如 800 字符），还有一个**绝对上限 7500 字符**。这是给保护区间兜底的——如果一段不可分割的内容（比如一个超长表格）本身就超过 7500 字符，强制截断它，否则 embedding API 会因为输入超限报错。

这是一个"两个约束打架"时的仲裁：语义完整性（不切断保护区间）vs API 限制（不能超过 token 上限）。选前者选到一定程度（7500），就必须向后者妥协，否则整个 pipeline 直接崩。

**教训：分块器要有"保护区间"的概念。** 代码、表格、公式这些结构化内容，切开了就废了。宁可让某个块超过目标 size，也别从中间切断一个语义单元。同时设一个绝对上限，防止保护逻辑被极端长内容利用导致 API 超限。

---

## 第四步：parent-child 双模式分块

WinMatrix 的分块有两种模式，由知识库配置决定：

```typescript
// RagService.ts（第 215-216 行）
const chunkingMode = options.chunkingMode
  ?? (kbChunkingCfg?.parentChildEnabled ? 'parent_child' : 'flat');

// RagService.ts（第 231-257 行）
const buildChunks = async () =>
  chunkingMode === 'parent_child'
    ? (await chunkDocumentParentChildAsync(
        parseResult.sections, documentPath, parseResult.documentType,
        options.projectId, metadata,
        {
          parentChunkSize: options.parentChunkSize ?? kbChunkingCfg?.parentChunkSize,
          childChunkSize: options.childChunkSize ?? kbChunkingCfg?.childChunkSize,
          childOverlap: options.childOverlap ?? kbChunkingCfg?.childOverlap,
          separators,
        },
      )).flatMap((pair) => [pair.parent, ...pair.children])   // parent + children 摊平
    : await chunkDocumentAsync(
        parseResult.sections, documentPath, parseResult.documentType,
        options.projectId, metadata,
        { maxChunkSize: chunkSize, overlapSize: chunkOverlap, separators },
      );
```

两种模式的区别：

- **flat 模式**：每个块大小差不多（比如 800 字符），块之间有 overlap（比如 100 字符）。检索时直接检索这些块。
- **parent-child 模式**：把文档切成"父块"（大，比如 2000 字符）和"子块"（小，比如 400 字符）。子块用于向量检索（小而精准，匹配度高），父块用于喂给 LLM（大而完整，上下文充分）。

parent-child 的价值在于**解耦"检索精度"和"上下文完整性"**。向量检索喜欢短文本——短文本的语义向量更聚焦，匹配更准。但 LLM 需要长上下文——只有 400 字符的子块可能信息不够。parent-child 让"检索用子块、生成用父块"，两头都顾。

注意 `flatMap((pair) => [pair.parent, ...pair.children])`——parent 和 children 都写进索引，检索时可以根据需要选择命中子块后回溯到父块。数据库里的 `knowledge_chunks` 表专门有 `chunk_type` 和 `parent_chunk_id` 字段承载这个关系：

```typescript
// RagService.ts（syncChunksToDb，第 578-579 行）
chunk_type: (c.metadata.chunk_role as string) || 'text',   // text / parent / child
parent_chunk_id: c.metadata.parent_chunk_id || null,
```

**教训：检索粒度和生成粒度不必相同。** 一份长文档，检索时希望精准（小块），生成时希望完整（大块）。parent-child 分块是解耦这两个诉求的标准做法，比"一块到底"或"纯 flat"都更灵活。

---

## 第五步：双写——ES 向量 + PG 元数据

分完块，接下来是写入。WinMatrix 的写入策略是**双写**：向量写 ES，元数据写 PG。

```typescript
// RagService.ts（第 313-320 行）
const synced = await upsertRagChunks(chunks);        // 写 ES（向量 + 全文）
logger.info(`[RagService] 向量写入完成: ${fileName} (synced=${synced}/${chunks.length})`);

if (synced === 0) {
  throw new Error('ES 写入失败：无块成功写入，中止知识化');
}

await this.syncChunksToDb(options.knowledgeId, chunks);   // 写 PG（元数据）
```

为什么双写？因为 ES 和 PG 各有所长：

- **ES**：擅长向量检索（kNN）和全文检索（BM25），是检索引擎。但 ES 不擅长事务、不擅长复杂关系查询。
- **PG**：擅长事务、关系、管理界面。知识库管理后台要展示"这个知识库有哪些文档、每个文档分了多少块、块的内容是什么"，这些查询走 PG 最自然。

双写意味着要维护一致性。WinMatrix 的策略是：**先写 ES，ES 成功后再写 PG；如果 ES 一块都没写进去（synced === 0），直接抛错中止**。为什么 ES 失败要中止？因为 ES 是检索的根基——如果块没进 ES，这份文档就等于没入库，PG 里写了元数据也检索不到，反而制造了"文档存在但搜不到"的幻觉。

PG 的写入有个细节——分批：

```typescript
// RagService.ts（syncChunksToDb，第 588-600 行）
const batchSize = 100;
for (let i = 0; i < data.length; i += batchSize) {
  const batch = data.slice(i, i + batchSize);
  await prisma.knowledge_chunks.createMany({
    data: batch.map((d) => ({...})),
  });
}
```

一份大文档可能切出几百上千个块，一次性 `createMany` 几百条数据可能超 PG 的参数限制或超时。分批 100 条一批，稳妥。

**教训：向量库和关系库各有所长，双写是务实选择。** 但双写要明确"谁是真相源、谁是副本"——这里 ES 是检索真相源（ES 没写进去就中止），PG 是管理真相源。一致性策略要围绕"哪个失败该中止、哪个失败该容忍"来设计，别追求强一致（成本太高），也别完全不管（数据会漂移）。

---

## 第六步：异步增强——图谱抽取和问题生成

主 pipeline 跑完后，还有两个**异步增强**步骤，用 `void ... .catch()` 的"fire-and-forget"模式跑：

```typescript
// RagService.ts（第 324-333 行）
if (isNeo4jAvailable() && options.knowledgeBaseId) {
  void this.extractGraphForChunks(chunks, options.knowledgeBaseId).catch((err) =>
    logger.warn(`[RagService] 图谱抽取失败（不影响摄入）: ${getErrorMsg(err)}`),
  );
}

if (options.knowledgeBaseId) {
  void this.maybeGenerateQuestions(chunks, options.knowledgeBaseId, options.knowledgeId).catch((err) =>
    logger.warn(`[RagService] 问题生成失败（不影响摄入）: ${getErrorMsg(err)}`),
  );
}
```

两个增强：

- **图谱抽取**（Neo4j）：从文本里抽取实体和关系，构建知识图谱。图谱检索是向量检索的补充——向量擅长语义相似，图谱擅长关系推理。
- **问题生成**（FAQ）：用 LLM 为每个块生成"可能被问到的问题"，这些问题本身也被向量化，用于 FAQ 类知识库的精准匹配。

注意这两个都是**异步且失败不阻断**的。注释明确写"不影响摄入"——主 pipeline 已经成功（向量 + 元数据都写好了），文档已经可检索了。图谱和问题只是锦上添花，它们失败不该让整个入库回滚。这是"核心链路 vs 增强链路"的清晰划分：

```
核心链路（必须成功）：解析 → 分块 → 双写 → 文档可检索
增强链路（尽力而为）：图谱抽取 / 问题生成 / 知识蒸馏
```

核心链路失败要抛错、要重试、要让用户知道。增强链路失败只记 warn，下次再说。

**教训：pipeline 要分"核心"和"增强"两层。** 核心层保证"文档能被检索到"这个最小可用承诺，增强层逐步提升检索质量。两层用不同的失败策略——核心层 fail-hard，增强层 fail-soft。别把所有步骤都绑成一个事务，否则一个可选步骤的失败会拖垮整个 pipeline。

---

## 兜底设计：原件一定要留

最后讲一个贯穿整条 pipeline 的兜底设计——**原件一定要留，即使分块失败也能重新来过**。

在分块之前，pipeline 先存原件：

```typescript
// RagService.ts（第 286-291 行）
if (gateChunks.length === 0) {
  logger.warn(`[RagService] 文档分块后无有效块: ${fileName}`);
  await saveOriginalDocument(buffer, fileName, options.projectId, documentPath);
  return { status: 'no_chunks', chunkCount: 0 };
}
```

如果分块结果是 0（文档可能是纯图片 PDF、空文件、不支持的格式），**不抛错，而是把原件存起来，返回 `no_chunks` 状态**。为什么不抛错？因为"分不出块"不是故障，是一种合法的中间状态——也许以后分块算法升级了，或者用户补了 OCR，这份原件可以重新 ingest。

更精妙的是预清理后的重存：

```typescript
// RagService.ts（第 305-309 行）
// 预清理会删除刚存的原件，需重新保存（删旧的、存新的 的语义）
await deleteByDocumentPath(documentPath, options.projectId);   // 清旧的向量
await deleteDocumentAssets(options.projectId, documentPath);   // 清旧的图片
await deleteOriginalDocument(options.projectId, documentPath); // 清旧的原件
await saveOriginalDocument(buffer, fileName, options.projectId, documentPath);  // 重存新原件
```

"预清理"是为了处理"同一份文档重新上传"的场景——先把旧的所有痕迹（向量、图片、原件）全删掉，再写新的。但删完之后要**重新存一份原件**，因为后续如果出任何问题，还能从原件恢复。注释那句"删旧的、存新的 的语义"点出了设计意图：清理是为了去旧，但去旧之后必须保证"任何时候都有一份原件可用"。

**教训：任何不可逆的转换 pipeline（解析、分块、向量化都是"有损"的），原件必须留底。** 因为转换后的产物（chunks、向量）无法还原回原件，一旦原件丢了，这份文档就永久损失了。留底原件 = 保留"重新来过"的能力。

---

## 一个完整的入库时序

把所有步骤串起来，一份 PDF 入库的完整时序：

```
用户上传 report.pdf
       │
       ▼
[1] 大小校验 OK（< maxBytes）
[2] md5 哈希 → 与缓存比对 → 未变更则 skip
[3] parserRegistry.parse → 解析成 15 个 sections（含 3 张图片）
[4] 文档类型=pdf → PDF_SEPARATORS（\n\n, \n, 。）
[5] chunkingMode=parent_child → 切出 8 个父块 + 32 个子块
[6] ABSOLUTE_MAX_CHUNK_SIZE 校验 → 最大块 1800 字符 < 7500，通过
[7] 图片处理 → 3 张图转 URL 落盘
[8] 资产替换后重新分块 → 40 个块
[9] 预清理旧的（向量+图片+原件）→ 重存新原件
[10] upsertRagChunks → ES 写入 40/40 成功
[11] syncChunksToDb → PG 分批写 40 条元数据（parent_chunk_id 关联）
[12] 异步：Neo4j 图谱抽取（失败不阻断）
[13] 异步：FAQ 问题生成（失败不阻断）
       │
       ▼
文档可检索（向量检索 + 全文检索 + 图谱检索）
```

---

## 给后来者的几条总结

1. **入库 pipeline 比 retrieval 更值得投入**。检索算法再好，库里没有正确的东西也是白搭。
2. **内容哈希去重是入口标配**。md5 比走完整 pipeline 便宜几个数量级，重复扫描场景必备。
3. **分块要文档类型感知**。Markdown 按标题切、PDF 简单切、Excel 按表格切，一刀切必出问题。
4. **保护区间不能被切断**。代码块、表格、LaTeX 是原子单元，切开了就废了。
5. **设绝对上限防 API 超限**。保护区间是语义约束，ABSOLUTE_MAX_CHUNK_SIZE 是工程约束，两者冲突时后者兜底。
6. **parent-child 解耦检索粒度和生成粒度**。子块检索精准，父块上下文完整，两头都顾。
7. **双写要明确谁是真相源**。ES 是检索真相源（失败即中止），PG 是管理真相源。
8. **核心链路 fail-hard，增强链路 fail-soft**。别把可选的图谱/问题生成绑进必须成功的事务。
9. **原件永远留底**。不可逆转换的产物无法还原，原件是"重新来过"的唯一凭证。

知识库入库 pipeline 是 RAG 系统的地基。地基做扎实了，上层的检索、重排、生成才有发挥的空间。把分块、双写、兜底做扎实，你的 RAG 才不是建在沙子上。

---

> **下一篇**：[《企业微信双轨接入：长连接 AiBot + Webhook》](./21-wecom-dual-track.md)——企业微信怎么接入 AI 平台？长连接和 Webhook 各有什么坑？一篇讲透企微双轨接入的设计与权衡。
