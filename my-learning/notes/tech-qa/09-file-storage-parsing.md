# tech-qa 09 · 对象存储与文档解析 高频问答

> 落点：`infrastructure/file/`（`FileStorageService`、`FileValidationService`、`FileHashService`、`ContentTypeDetectionService`、`DocumentParseService`、`TextCleaningService`）、`common/config/S3Config`、`StorageConfigProperties`、`infrastructure/export/PdfExportService`。  
> 本地 RustFS 起法见 [01 本地启动](../01-env-setup.md)。

---

### 一次文件上传要做哪几件事？

**答：** 六件，缺一件都会在后面还债：

```text
校验大小 → 嗅探真实类型并过白名单 → 算哈希查重 → 抽正文 → 存对象存储 → 落库元数据（+ 投异步任务）
```

**本仓库：** 简历与知识库两条线共用这套基础设施，差异只在上限、白名单、前缀和后续任务：

| | 简历 | 知识库 |
|--|------|--------|
| 业务上限 | 10MB | 50MB |
| 白名单 | PDF/DOC/DOCX/TXT | MIME 列表 **或** `.md`/`.markdown` 扩展名兜底 |
| 前缀 | `resumes/` | `knowledgebases/` |
| 后续任务 | 分析评分 | 切块向量化 |

---

### 接 MinIO / RustFS 这类 S3 兼容存储，第一个坑是什么？

**答：** **必须开 path-style 访问**。SDK 默认用 virtual-host 风格把桶名拼成子域名（`bucket.endpoint`），本地根本没有这个 DNS 记录，直接解析失败。

| 风格 | URL | 适用 |
|------|-----|------|
| Virtual-host（默认） | `https://<bucket>.s3.amazonaws.com/<key>` | 真 AWS |
| **Path-style** | `http://localhost:9000/<bucket>/<key>` | RustFS / MinIO |

**本仓库：** `S3Config` 里 `.endpointOverride(...)` + `.forcePathStyle(true)`，另设 `apiCallTimeout` / `apiCallAttemptTimeout`，避免存储卡住时请求线程被一直挂着。还有个环境侧的坑：系统 HTTP 代理会把 `localhost:9000` 也代理走，需要 `NO_PROXY=localhost,127.0.0.1,::1`。

---

### 桶不存在怎么处理？

**答：** 启动时 `HeadBucket` 探测，不存在则 `CreateBucket`；并发场景下 `CreateBucket` 可能撞 **409**，这时应当视为「别人已经建好了」而不是失败。是否自动建桶要能开关（生产往往由运维预先建好）。

**本仓库：** `FileStorageService` 的 `@PostConstruct` + `app.storage.auto-create-bucket`（默认 `true`）。

---

### 对象 Key 怎么设计？

**答：**

```text
{prefix}/{yyyy/MM/dd}/{8位UUID}_{安全文件名}
resumes/2026/07/31/a1b2c3d4_ZhangSanJianLi.pdf
```

三个考虑：**日期分层**避免单前缀下堆积、也便于按天排查；**UUID 前缀**防同名覆盖；**文件名净化**——汉字转拼音（pinyin4j），其余非 `[A-Za-z0-9._-]` 一律替换。中文/空格/特殊字符直接进 Key，会在不同 S3 实现和 URL 编码上踩坑。

---

### 私有文件怎么给前端？

**答：** 三种：桶公开可读（简单但不安全）、**预签名 URL**（临时授权、不经业务服务器传输，推荐）、**后端代理下载**（权限最灵活，但流量过应用）。

**本仓库：** `getFileUrl()` 只是把 `endpoint/bucket/key` 拼成字符串，**不是签名 URL**，桶非公开时不能直接给浏览器；实际走 `downloadFile` 后端中转。要做真正的私有直链需要补预签名。

---

### 文件类型校验为什么不能信 `Content-Type`？

**答：** 请求头里的 `Content-Type` 由客户端提供，可任意伪造；把 `.exe` 改名成 `.pdf` 并声明 `application/pdf` 是最基础的攻击。必须按**文件内容**嗅探（magic bytes），再对嗅探结果过白名单。

**本仓库：** `ContentTypeDetectionService` 用 Tika 嗅探，校验用嗅探值。但要注意现状：上传到 S3 时 `contentType(file.getContentType())` 用的仍是请求头值——**校验用嗅探、存储元数据用客户端值**，两者可能不一致。

---

### 哈希去重值不值得？

**答：** 在「后面要花钱调 AI」的场景里非常值：命中即复用，省掉存储和一次完整 AI 调用。代价与边界要说清：

- 需要**多读一遍文件字节**（本项目查重与落库各算一次 SHA-256，属于可优化点）；
- 只能识别**字节完全一致**，改一个字就是新文件；
- 哈希计算异常若被吞成「没查到」，会静默退化为不去重。

**本仓库：** `FileHashService` 算 SHA-256，`resumes.file_hash` / `knowledge_bases.file_hash` 唯一索引。

---

### 用 Tika 抽正文要显式关掉什么？

**答：** 关掉嵌入资源解析，否则文本里混进图片占位与 `file:/...` 临时路径，喂给 LLM 全是噪声。

| 配置 | 作用 |
|------|------|
| `AutoDetectParser` | 按内容选解析器，不靠扩展名 |
| `BodyContentHandler(5MB)` | 只取正文并设上限 |
| `NoOpEmbeddedDocumentExtractor` | 不解析嵌入图片/附件 |
| `PDFParserConfig.setExtractInlineImages(false)` | 不抽内嵌图片 |
| `PDFParserConfig.setSortByPosition(true)` | 按坐标排序，改善多栏 PDF 阅读顺序 |

抽完还要清洗：`TextCleaningService` 去图片标记、`file:` URL、控制字符、压缩空白。**清洗质量直接决定 Embedding 和上下文质量**（见 [tech-qa/06](06-rag.md)）。

---

### 上传大小限制在哪几层？

**答：** 至少三层，改一层不改另一层等于白改：

1. **反向代理**（Nginx `client_max_body_size`，本项目本地未涉及）；
2. **Servlet**：`spring.servlet.multipart.max-file-size`（本项目 50MB），超限在进 Controller 前就被拦，由全局异常处理器转成统一 `Result`；
3. **业务层**：简历 10MB / 知识库 50MB。

另外还有一条容易忽略的线：文件 50MB **不等于** 正文 5MB —— Tika 抽取上限是 5MB，超长文档正文会被截断。

---

### 先存对象还是先落库？失败了怎么办？

**答：** 两个顺序各有风险：

| 顺序 | 风险 |
|------|------|
| 先存 S3 后落库 | 落库失败 → **孤儿对象**（存储有文件，库里没记录） |
| 先落库后存 S3 | 存储失败 → 记录指向不存在的对象，读取时才报错 |

严格一致要靠补偿：落库失败时删掉刚上传的对象，或加巡检任务清理孤儿。删除方向同理——对象删除失败时是「保 DB 一致」还是「阻止删除」，要明确选一个。

**本仓库：** 先 S3 后落库，**没有补偿删除**（已知缺口）；删除时 S3 失败只记日志、DB 照删（有意为之：以 DB 为准，代价是需要巡检）。

---

### 导出 PDF 中文乱码是什么原因？

**答：** 用了不含中文字形的默认字体，或编码不是 `IDENTITY_H`。解决：**内嵌中文字体文件 + IDENTITY_H 编码**，同时过滤字体不支持的字符（emoji 等）避免出现空白方块。

**本仓库：** iText 8，字体 `fonts/ZhuqueFangsong-Regular.ttf`；`PdfExportService.exportResumeAnalysis` / `exportInterviewReport` 是纯读库 + 内存生成字节流，与上传链路无关；`sanitizeText` 负责过滤不支持字符。

---

### 大文件处理要注意内存吗？

**答：** 要。`MultipartFile.getBytes()` 把整个文件读进堆内存，几个并发大文件就能把内存吃掉。更稳的做法是走流式：`getInputStream()` 传给 SDK、用分片上传、必要时落临时文件。哈希与解析也应该复用同一次流读取而不是各读一遍。

**本仓库：** 当前是「读字节 + 多次使用」的简单实现，10/50MB 上限下可接受，但这是**并发上量后第一个要改的地方**。

---

## 配置速查

| 配置键 | 环境变量 | 默认 |
|--------|----------|------|
| `app.storage.endpoint` | `APP_STORAGE_ENDPOINT` | `http://localhost:9000` |
| `app.storage.access-key` / `secret-key` | `APP_STORAGE_ACCESS_KEY` / `_SECRET_KEY` | 必填 |
| `app.storage.bucket` | `APP_STORAGE_BUCKET` | `interview-guide` |
| `app.storage.region` | `APP_STORAGE_REGION` | `us-east-1` |
| `app.storage.auto-create-bucket` | `APP_STORAGE_AUTO_CREATE_BUCKET` | `true` |
| `app.storage.api-call-timeout` / `api-call-attempt-timeout` | — | `60s` / `20s` |
| `spring.servlet.multipart.max-file-size` | — | `50MB` |
| `app.resume.allowed-types` | — | PDF / DOC / DOCX / TXT |

---

## 我的口述清单

1. 接 S3 兼容存储先开 path-style，本地代理要 `NO_PROXY`。
2. 类型校验靠内容嗅探，不靠请求头；也要知道存进库的是哪个值。
3. 哈希去重省的是「后面那次 AI 调用」，值。
4. Tika 必须显式关嵌入资源解析，再清洗一遍。
5. 大小限制分层；文件大小 ≠ 正文上限。
6. 先存后落库会产生孤儿对象，要么补偿删除，要么定期巡检。
