# tech-qa 08 · AI 应用工作流（流式 / 异步 / 排障）高频问答

> 落点：`KnowledgeBaseController.queryKnowledgeBaseStream`、`RagChatController.sendMessageStream`、`frontend/src/api/stream.ts`、`voiceinterview/config/WebSocketConfig`、`handler/VoiceInterviewWebSocketHandler`、`common/async/*`。

---

### 做 AI 应用，工程上要额外关注什么？

**答：** 相比普通 CRUD，多了五类问题：

| 关注点 | 具体表现 |
|--------|----------|
| **延迟** | 单次调用秒级到分钟级，不能占着请求线程 |
| **不确定性** | 同样输入不同输出，解析可能失败，必须有重试与降级 |
| **成本** | 每次调用真花钱，要去重、裁上下文、限流 |
| **可见性** | 用户需要「进行中/失败/可重试」的状态，而不是转圈 |
| **安全** | 用户内容会进 Prompt（注入面），输出可能带敏感信息 |

这五条决定了架构长什么样：**流式给人看、队列扛长任务、状态机管可见性、结构化 + 重试保可靠**。

---

### SSE、WebSocket、轮询怎么选？

**答：**

| 方案 | 方向 | 适用 | 代价 |
|------|------|------|------|
| 轮询 | 客户端拉 | 状态查询（分析完了吗） | 有延迟、请求多 |
| **SSE** | 服务端单向推 | 文本流式生成 | 单向、需保持连接 |
| **WebSocket** | 全双工 | 实时语音、双向控制 | 协议自定、重连与状态管理复杂 |

判断依据是**方向和粒度**：只是「把生成的字推给你」用 SSE；边说边听、还要发控制指令，才需要 WebSocket。

**本仓库全用上了**：异步任务状态用轮询（`analyzeStatus`/`vectorStatus`）、RAG 回答用 SSE、语音对话用 WebSocket。

---

### 后端流式怎么实现？`Flux` 和 `SseEmitter` 有什么区别？

**答：** `SseEmitter` 是 Spring MVC 的 Servlet 异步方案，手工 `send`/`complete`；`Flux<ServerSentEvent<T>>` 是响应式方案，能直接对接上游模型的流，`doOnNext`/`doOnComplete`/`doOnError` 让「边推边落库」写得很自然。

**本仓库：** 两个流式端点都是 WebFlux `Flux`，**没有用 `SseEmitter`**：

- `POST /api/knowledgebase/query/stream` → `Flux<String>`（`text/event-stream`）
- `POST /api/rag-chat/sessions/{id}/messages/stream` → `Flux<ServerSentEvent<String>>`

---

### 前端为什么不用 `EventSource`？

**答：** `EventSource` 只能发 **GET**、不能自定义请求头/请求体。RAG 问答要 POST 一个 JSON 体（问题 + 知识库 id + 会话 id），所以只能用 `fetch` + `response.body.getReader()` + `TextDecoder` 自己解析流。代价是**浏览器自带的自动重连没了**，要自己实现。

**本仓库：** `frontend/src/api/stream.ts` 的 `streamSse` 支持两种解析模式——`line`（按 `\n` 取 `data:` 行，KB 查询用）与 `event`（按 `\n\n` 分块、能识别 `event: error`，RAG 会话用）。

---

### 流式中途断了怎么办？已经生成的内容怎么处理？

**答：** 原则是**不要留空记录**。断流分两种：

1. **上游/服务端出错**：把已累积内容落库，或落一条明确的错误文案；
2. **客户端断开**：服务端仍应完成落库，用户下次进来能看到已生成的部分。

**本仓库：** RAG 会话流用 `StringBuilder` 累积，`doOnComplete` 落完整内容，`doOnError` 落已有部分内容或 `【错误】回答生成失败：...`；SSE data 里换行被转义成 `\n`，前端反转义还原（否则 SSE 的 `\n\n` 分隔语义会被正文里的换行破坏）。

⚠️ 项目未覆盖：前端 SSE **没有重试/续传**，单次 `fetch` 失败即结束（学习计划 G7）。

---

### 首包延迟怎么压下来？

**答：** 关键是「不要等整段做完」。四个手段：

1. **流式生成**：token 一到就推；
2. **分段下游处理**：语音场景按**句**切给 TTS，合成好一句就播，而不是等整段文本；
3. **并行**：前一句合成时后一句已开始；
4. **预热**：开场白等固定内容提前生成缓存。

**本仓库：** 语音链路就是这么做的（句级并发 TTS + `audio_chunk` 分片下发）；开场白预热做成了默认关的开关，因为每次启动都真调云端 API 太费。

---

### 长耗时任务为什么用消息队列而不是裸 `@Async`？

**答：** `@Async` 的队列在 JVM 内存里：重启丢任务、没有重试、没有可见性、多实例无法负载均衡。队列（这里是 Redis Stream）提供持久化、消费者组、ACK/重投、失败可观测。

判断标准很简单：**这个任务丢了要不要人来补？** 要，就上队列。

**本仓库：** 简历分析、文字/语音评估、文档向量化四条链路全走 Stream（见 [tech-qa/03](03-redis.md)）。

---

### 长任务的状态怎么让用户看见？

**答：** 用**显式状态机 + 轮询**，并且状态要包含失败原因与重试入口：

```text
PENDING → PROCESSING → COMPLETED
                    ↘ FAILED（带 error 文案，可重试）
```

**本仓库：** `resumes.analyze_status`、`interview_sessions.evaluate_status`、`knowledge_bases.vector_status` 都是这套；失败原因存在 `*_error` 字段（截断到 500 字符），前端提供 `reanalyze` / `revectorize` 重试。

---

### 成本怎么控？

**答：** 优先级从高到低：**去重**（同一输入不重复调）> **裁上下文**（只塞必要片段）> **换小模型/规则**（简单任务不用大模型）> **限输出长度** > **接口限流防刷**。

**本仓库：** 文件哈希去重直接省掉整次分析；RAG 只塞 TopK；技能包参考资料按分类注入；所有 AI 入口都有 `@RateLimit`。⚠️ 项目未覆盖：token 用量统计。

---

### 一次请求跨 HTTP / Stream / WebSocket，怎么排障？

**答：** 需要三样：**贯穿全链路的关联 id**（traceId 从 HTTP 请求带进消息体，消费者日志里能查到）、**分段耗时指标**（哪一段慢一眼看出）、**结构化日志**（可按 id 聚合）。缺关联 id 时，排查异步失败只能靠时间戳 + 业务 id 手工拼。

**本仓库：** 语音链路的分段指标做得比较全（`app.voice.interview.*`）；⚠️ 项目未覆盖：MDC/traceId 与分布式追踪（学习计划 G3）。当前只能靠 `sessionId`/`resumeId` 这类业务 id 串日志。

---

### 安全上要注意什么？

**答：** 四条：用户内容进 Prompt 前过滤并用边界包裹（见 [tech-qa/05](05-prompt-engineering.md)）；密钥只在服务端、不回传前端；日志不打完整 Prompt/密钥/个人敏感信息；对外「连通性测试」这类能发起任意请求的接口要防内网探测（SSRF）。

**本仓库：** 有 sanitizer + 随机边界 + SafeGuardAdvisor 词表 + 密钥加密存储；Provider 测试接口的内网防护属于要加固的点。

---

### 一个 AI 功能的开发顺序应该是什么？

**答：** 我在这个项目里形成的顺序：

1. **先用强模型手工验证效果**（能不能做，Prompt 大致怎么写）；
2. **定输出结构**（DTO/schema 先定下来，前后端并行）；
3. **接结构化调用 + 重试兜底**（保证不 500）；
4. **接异步 + 状态机**（保证不卡请求线程）；
5. **加流式**（体验）；
6. **加限流与指标**（成本与可观测）；
7. **最后才是调参与评测**。

顺序错了最典型的表现是：先花几天调 Prompt，结果发现链路根本撑不住并发或失败不可见。

---

## 我的口述清单

1. 流式给人看、队列扛长任务、状态机管可见性——三件事分工明确。
2. POST + 流式 = 只能 `fetch` 手工解析，自动重连要自己实现。
3. 断流必须落库（完整或部分），不能留空记录。
4. 首包延迟靠「分段 + 并行 + 预热」，不靠换更快的模型。
5. 我这套的可观测缺口是 traceId 贯穿；语音的分段指标是可复用的正面样本。
