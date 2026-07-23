# 第 16 章 知识库系统

> "知识是 RAG 的燃料，分块是燃烧的效率。"

知识库让数字员工能够基于企业文档回答问题。WinMatrix 的知识库系统支持多种文档格式（PDF / Word / Excel / Markdown），使用智能分块策略和可选 OCR，将非结构化文档转化为可检索的向量。本章将深入文档摄入、分块策略和索引管理。

## 16.1 知识库数据模型

知识库的持久化结构：

```prisma
// prisma/schema.prisma（第 1355-1466 行）
model knowledge_bases {
  id                         String   @id @default(uuid())
  project_id                 String
  name                       String
  description                String?
  type                       String   @default("document")  // document | faq | tfs_git
  chunking_config            Json     @default("{}")        // 分块配置
  embedding_model            String?
  extract_config             Json?
  is_pinned                  Boolean  @default(false)
  faq_config                 Json?
  question_generation_config Json?
  tfs_git_config             Json?
  @@map("knowledge_bases")
}

model knowledge_chunks {
  id           String   @id
  knowledge_id String
  content      String
  metadata     Json
  embedding    Unsupported("vector")?
  tsv          Unsupported("tsvector")?
  @@map("knowledge_chunks")
}
```

三种知识库类型：

- **document**：文档型（PDF/Word/Excel/Markdown）
- **faq**：问答型（Q&A 对）
- **tfs_git**：TFS/Git 代码仓库

### KnowledgeBaseService

```typescript
// src/business/domain/knowledgeBase/KnowledgeBaseServiceImpl.ts（第 20-30 行）
export class KnowledgeBaseServiceImpl implements IKnowledgeBaseService {
  constructor(private readonly repo: IKnowledgeBaseRepository) {}

  async create(params: CreateKnowledgeBaseParams): Promise<DomainResult<KnowledgeBase>> {
    const existing = await this.repo.exists(params.projectId, params.name);
    if (existing) {
      return {
        success: false,
        error: { code: 'DUPLICATE_NAME', message: `知识库名称已存在: ${params.name}` },
      };
    }
    // ... 创建逻辑
  }
}
```

创建时的唯一性检查避免重名知识库。

## 16.2 RAG 基础设施

`src/infrastructure/rag/` 目录包含完整的 RAG 处理链：

```
src/infrastructure/rag/
├── DocumentChunker.ts          # 分块策略
├── RagService.ts               # 主服务
├── RagRepository.ts            # ES 持久化
├── RagHybridSearch.ts          # 混合检索
├── RagQuestionGenerator.ts     # 问题生成
├── RagAssetManager.ts          # 资产管理
├── RagAssetS3Store.ts          # S3 存储
├── RagDirectoryWatcher.ts      # 目录监听
├── PmdocRagIndexer.ts          # PMDoc 索引器
├── parserDefaults.ts           # 解析器默认配置
├── ragToolFormat.ts            # 工具格式化
├── types.ts
├── parsers/                    # 文档解析器
│   ├── BaseParser.ts
│   ├── PdfParser.ts
│   ├── DocxParser.ts
│   ├── ExcelParser.ts
│   ├── CsvParser.ts
│   ├── HtmlDocParser.ts
│   ├── ImageDocParser.ts
│   ├── MarkdownDocParser.ts
│   ├── ParserRegistry.ts
│   └── TextParser.ts
├── ingest/                     # 摄入管线
│   ├── ragIngestWorker.ts
│   ├── ragIngestQueue.ts
│   ├── knowledgeSourceReader.ts
│   └── ragIngestPreflight.ts
└── ocr/                        # OCR 后端
    ├── OcrBackend.ts
    ├── VlmOcrBackend.ts        # VLM 视觉模型 OCR
    └── DummyOcrBackend.ts
```

## 16.3 文档解析

### PDF 解析器

```typescript
// src/infrastructure/rag/parsers/PdfParser.ts（第 1-60 行）
/**
 * PDF 文档解析器
 * 使用 pdf-parse v2（PDFParse 类 + pdfjs）提取 PDF 文本。
 * v1 的默认函数 pdfParse(buffer) 在 v2 已移除，须用 new PDFParse({ data }).getText()。
 * 对于扫描件/图片型 PDF（文本为空），可选调用 OCR 后端。
 */
export class PdfParser extends BaseDocParser {
  async parseToMarkdown(buffer: Buffer): Promise<ParsedDocument> {
    const { PDFParse } = await import('pdf-parse');
    const parser = new PDFParse({ data: buffer });
    const textResult = await parser.getText();

    let content = textResult.text;

    // 扫描件 fallback：文本为空时调用 OCR
    if (!content && isOcrEnabled()) {
      content = await this.ocrFallback(buffer);
    }
    // ... 元数据提取（Title/Author）
  }
}
```

PDF 解析的两个关键点：

1. **pdf-parse v2 迁移**：v2 移除了默认函数，必须用 `new PDFParse({ data }).getText()`
2. **OCR Fallback**：扫描件（图片型 PDF）文本提取为空时，调用 VLM OCR 后端

### ParserRegistry

```typescript
// src/infrastructure/rag/parsers/ParserRegistry.ts
// 根据 MIME 类型/扩展名路由到对应解析器
// PDF → PdfParser
// Word → DocxParser
// Excel → ExcelParser
// Markdown → MarkdownDocParser
// HTML → HtmlDocParser
// Image → ImageDocParser（OCR）
```

### 多格式支持

| 格式 | 解析器 | 依赖 |
|------|--------|------|
| PDF | PdfParser | pdf-parse + 可选 OCR |
| Word | DocxParser | mammoth |
| Excel | ExcelParser | exceljs |
| CSV | CsvParser | csv-parse |
| Markdown | MarkdownDocParser | 内置 |
| HTML | HtmlDocParser | cheerio |
| Image | ImageDocParser | VLM OCR |

## 16.4 文本分块策略

`DocumentChunker.ts` 实现了智能分块，算法移植自 WeKnora 的 `splitter.go`：

```typescript
// src/infrastructure/rag/DocumentChunker.ts（第 1-40 行）
/**
 * 核心算法移植自 WeKnora splitter.go，增加：
 * - 保护区间：LaTeX / 代码块 / 表格 / Markdown 图片与链接不会被截断
 * - 可配置分隔符：按优先级拆分，保留分隔符
 * - 绝对最大块上限（7500 字符）：防止 embedding API 超限
 * - 改进的 overlap 计算：按 unit 粒度回溯，跳过纯分隔符
 * - Parent-child 分块复用同一核心算法
 */
const ABSOLUTE_MAX_CHUNK_SIZE = 7500;

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
```

### 分块选项

```typescript
interface ChunkerOptions {
  maxChunkSize: number;     // 默认 800
  overlapSize: number;      // 默认 200
  separators: string[];     // 分隔符优先级
  minChunkSize: number;     // 默认 20
}
```

### 保护区间

分块算法的关键是**不破坏语义完整性**：

- **LaTeX 公式**：不会被截断
- **代码块**：完整的 ` ``` ` 块保持在同一 chunk
- **表格**：Markdown 表格不被拆分
- **图片和链接**：保持完整

### 文档类型感知

不同文档类型使用不同的分隔符优先级：

```typescript
// DocumentChunker.ts（第 60-66 行）
// markdown / word / pdf / excel 各有不同的分隔符配置
```

### 绝对上限保护

```typescript
const ABSOLUTE_MAX_CHUNK_SIZE = 7500;
```

这个 7500 字符的硬上限防止单个 chunk 超出 embedding API 的输入限制（如 OpenAI 的 8192 token 限制）。

## 16.5 RAG 摄入 Worker

文档摄入是异步的，通过 BullMQ Worker 执行：

```typescript
// src/infrastructure/rag/ingest/ragIngestWorker.ts（第 1-80 行）
let workerInstance: Worker<RagIngestJobPayload, RagIngestJobResult> | null = null;

async function loadBytes(payload: RagIngestJobPayload): Promise<Buffer> {
  if (payload.sourceKind === 'local-path') {
    return await fsp.readFile(payload.localFilePath);   // 本地路径
  }
  // 从知识库读取
  return readKnowledgeSourceBuffer({ /* ... */ });
}

async function verifyContentHash(job, buffer, token?): Promise<'ok' | 'drop' | 'retry'> {
  const actualHash = md5HashBuffer(buffer);
  // 内容 hash 验证 + mtime 宽限期
  // 如果文件正在被写入，延迟重试
  throw new DelayedError('content hash mismatch, recent mtime');
}
```

### 内容 Hash 验证

摄入过程中的 hash 验证确保数据完整性：

1. 计算文件 buffer 的 MD5 hash
2. 与预期 hash 比较
3. 如果不匹配（文件可能正在写入），延迟重试（`DelayedError`）

这种设计避免了"读到半个文件"的问题——大文件上传过程中可能被部分读取。

## 16.6 OCR 后端

对于扫描件和图片，系统使用 VLM（视觉语言模型）OCR：

```typescript
// src/infrastructure/rag/ocr/
// OcrBackend.ts      - OCR 接口
// VlmOcrBackend.ts   - VLM 视觉模型 OCR（如 GPT-4V）
// DummyOcrBackend.ts - 占位实现（未配置时）
```

VLM OCR 比传统 OCR（Tesseract）更准确，特别是对于复杂排版、表格和手写内容。

## 16.7 向量索引

知识分块向量化后存储在 Elasticsearch：

```typescript
// src/infrastructure/rag/RagRepository.ts
// searchRag()        - kNN 向量搜索
// fullTextSearchRag() - BM25 全文搜索
// fetchChunkTextsByIds() - 按 ID 获取分块文本
```

ES 索引 `winmatrix_rag` 同时支持：

- **kNN 向量搜索**（cosine 相似度）
- **BM25 全文搜索**（document 字段）

## 本章小结

本章深入分析了 WinMatrix 的知识库系统：

1. **三种知识库类型**：document / faq / tfs_git
2. **多格式解析**：PDF / Word / Excel / CSV / Markdown / HTML / Image，通过 ParserRegistry 路由
3. **OCR Fallback**：扫描件文本为空时调用 VLM OCR
4. **智能分块**：移植自 WeKnora，保护区间（LaTeX/代码/表格/链接）+ 文档类型感知分隔符
5. **绝对上限**：7500 字符硬上限，防止 embedding API 超限
6. **异步摄入**：BullMQ Worker + 内容 hash 验证 + mtime 宽限期 + 延迟重试
7. **向量索引**：Elasticsearch kNN + BM25 双索引

在下一章中，我们将深入 RAG Worker 与检索增强。
