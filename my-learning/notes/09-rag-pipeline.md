# 09 · RAG 检索增强全链路

> 对应源码：`modules/knowledgebase/`、`prompts/knowledgebase-query-*.st`、pgvector 配置  
> 上游：文档怎么进来见 [07 对象存储与文档解析](07-file-storage-parsing.md)；向量表结构见 [03 库表设计 §4.4](03-db-schema-design.md)。

## 全链路流程

```
文档上传 → Tika 解析 → 分块(Chunking) → Embedding 向量化 → 存入 pgvector
                                                                    ↓
用户提问 → Query Rewrite → Embedding → 向量检索(COSINE) → TopK + 阈值过滤
                                                                    ↓
                              Context 拼接 → LLM 生成 → SSE 流式返回
```

## 各环节详解

### 1. 文档解析（Tika）

- 支持 PDF / DOCX / Markdown / TXT 等格式
- `DocumentParseService` 封装 Tika，自动检测文件类型
- `ContentTypeDetectionService` 防止 MIME 类型伪造

### 2. 分块策略

- 按段落/章节切分，保持语义完整性
- Chunk overlap 避免关键信息被切断
- 分块大小与 Embedding 模型的 context window 匹配

### 3. 向量化与存储

```
pgvector 配置：
- 向量维度：1024（由 Embedding 模型决定）
- 距离类型：COSINE
- 索引：IVFFlat（兼顾速度与精度）
```

### 4. Query Rewrite（查询改写）

- 用户口语化提问 → LLM 改写为检索友好的查询
- Prompt：`knowledgebase-query-rewrite.st`
- 为什么需要：用户问题往往不直接匹配知识库文档的语言风格

### 5. 检索与过滤

- COSINE 相似度计算
- TopK + 相似度阈值双重过滤
- 避免无关文档污染 Context

### 6. SSE 流式生成

- `SseEmitter` 实现打字机效果
- Context 拼接在 System Prompt 中注入
- Prompt：`knowledgebase-query-system.st` + `knowledgebase-query-user.st`

## 关键设计决策

| 决策 | 选择 | 理由 |
|------|------|------|
| 向量数据库 | pgvector | PG 原生支持，精简架构 |
| 距离算法 | COSINE | 语义相似度场景最常用 |
| 向量索引 | IVFFlat | 平衡检索速度与精度 |
| 异步向量化 | Redis Stream | 文档量大时不阻塞上传接口 |

## 核心要点

- **RAG vs 微调**：RAG 知识可动态增删，无需重新训练；微调适合固化模型行为，两者互补而非互斥
- **Chunk 大小的影响**：太大检索精度下降、Context 噪声增多；太小丢失语义上下文、Embedding 质量下降
- **Query Rewrite 的价值**：用户口语化提问与文档书面语之间存在表达差异，改写后检索命中率显著提升
- **为什么不是 Milvus/Pinecone**：当前规模下 pgvector 完全够用，减少运维组件数量，后续可平滑迁移
