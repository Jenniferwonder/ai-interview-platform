# tech-qa 04 · LLM 接入 高频问答

> 落点：`common/ai/LlmProviderRegistry`、`common/config/LlmProviderProperties`、`LlmEmbeddingConfig`、`modules/llmprovider/`、`common/ai/StructuredOutputInvoker`。  
> 模块视角见 [modules/06 模型与语音配置](../modules/06-llmprovider.md)。

---

### Java 应用怎么接入大模型？

**答：** 三种路径：

1. **云端 API（主流）**：HTTP 调 OpenAI 兼容接口，或用 SDK / Spring AI 封装；
2. **自建推理服务**：vLLM / Ollama / LM Studio 起本地模型，仍以 OpenAI 兼容协议暴露；
3. **进程内推理**：Java 生态很少这么干。

只要对方兼容 OpenAI 协议，客户端代码基本不用改，切换成本主要在 `base-url` + `api-key` + 模型名。

**本仓库：** Spring AI 的 `OpenAiChatModel` + `ChatClient`，Provider 的 base-url / model / api-key 全部可配可落库。

---

### `ChatModel` 和 `ChatClient` 有什么区别？

**答：** `ChatModel` 是底层模型抽象（一次请求一次响应）；`ChatClient` 是上层流式 API，负责 prompt 组装、Advisor 链（工具调用、记忆、日志、内容防护）、结构化输出转换。业务代码应该面向 `ChatClient`。

**本仓库：** `LlmProviderRegistry` 用 `OpenAiChatModel.builder()` 造模型，再包成 `ChatClient` 缓存起来；业务只调 `getChatClientOrDefault(provider)`。

---

### 多 Provider 怎么抽象？为什么不用多个 Bean + `@Primary`？

**答：** 多 Bean 的问题是**运行时不可变**：Provider 列表、默认模型、密钥都写死在启动配置里，加一个模型要改代码重启。注册表模式（Registry）把「Provider 定义」变成数据，运行时按 id 取客户端、按需重建。

**本仓库：**

```text
设置页 → llm_provider_config（密钥加密存库）
       → LlmProviderRegistry（ConcurrentHashMap 缓存 ChatClient）
       → refresh() 清缓存，下次调用重建
```

启动类还 `exclude` 了 Spring AI 的 OpenAI 自动配置，避免默认单 Provider 装配和 Registry 抢事。

---

### 为什么同一个 Provider 要拆成三种客户端？

**答：** 不同场景对「模型自由度」的要求不同。

| 变体 | 工具 | Advisor | 用在哪 |
|------|------|---------|--------|
| 默认 | 挂 skills 工具 | 完整栈 | RAG 问答、JD 解析 |
| plain | 不挂 | 只 SafeGuard | 出题（要稳定 JSON，不希望模型跑去调工具） |
| voice | 挂 skills 工具 | ToolCalling（开对话历史）+ SafeGuard | 语音对话（要连续上下文） |

Advisor 逐项可关：`app.ai.advisors.tool-call-enabled`（默认开）、`message-chat-memory-enabled`（默认关）、`simple-logger-enabled`（默认关）、`safeguard-enabled`（默认开）。

---

### 模型怎么选？

**答：** 先分清任务类型（对话/推理、Embedding、多模态语音），再在四个维度上折中：**效果、延迟、成本、上下文长度**。实践顺序通常是：用强模型验证「这个功能到底能不能做」，跑通后再往下换便宜/快的模型，用固定用例对比效果。

**本仓库：** 结构化产出（评分、出题、评估）对 JSON 稳定性敏感，宁可贵一点；语音对话对**首 token 延迟**敏感，优先快；Embedding 只要维度与 pgvector 的 1024 对齐。

---

### 为什么相同输入会得到不同输出？要怎么控？

**答：** 生成是按概率采样的，`temperature`/`top_p` 越高越随机；服务端批处理、模型版本变更也会造成波动。降低随机性可以调低温度、固定 seed（若支持）、把格式约束写进 Prompt 并用结构化解析兜底——但**不要假设完全可复现**。

**本仓库：** 结构化场景不指望「一次成功」，而是 `StructuredOutputInvoker` 解析失败就带错误信息重试（见 [tech-qa/11](11-ai-quality-evaluation.md)）。

---

### 流式和非流式怎么选？

**答：** 面向人的长文本一定要流式——首字出现时间决定体感；面向程序的结构化输出用非流式，因为 JSON 要完整才能解析。

**本仓库：** RAG 问答与语音对话是流式（`Flux` / `.stream().content()`），评分/出题/评估是非流式结构化调用。

---

### API Key 怎么管？

**答：** 不进代码、不进仓库、不回传给前端。最低要求是环境变量；配置要落库时应加密存储，并且响应里只返回「是否已配置」而不是明文。

**本仓库：** `.env` 提供 `APP_AI_CONFIG_ENCRYPTION_KEY`，`ApiKeyEncryptionService` 把 Key 加密成 `api_key_ciphertext` + `api_key_nonce` 两列；`require-encryption-key` 默认 `true`，缺密钥直接拒绝。响应 DTO 不含明文。

---

### 超时、重试、降级怎么设计？

**答：** 分三层：

1. **超时**：连接/读超时必须显式设置，尤其流式要区分「首包超时」和「整体超时」；
2. **重试**：只重试可恢复错误（网络、5xx、429），并且要退避；对已产生副作用的调用不能盲目重试；
3. **降级**：明确「失败了给用户什么」——排队重试、返回兜底内容，或者显式失败但保留可重试入口。

**本仓库：** 上游 429 映射成 `AI_RATE_LIMIT_EXCEEDED(7005)`；结构化解析失败重试 2 次（可配）；出题彻底失败落到 `generateFallbackQuestions` 兜底题；异步任务失败标 `FAILED` 并支持重新触发（`reanalyze`、`revectorize`）。

---

### 成本怎么控？

**答：** 控输入比控输出更有效：裁剪上下文（RAG 只塞 TopK 片段而不是整篇文档）、缓存可复用结果、把简单任务下沉到小模型/规则、限制单次输出长度、给用户侧加限流防刷。

**本仓库：** 简历哈希去重直接省掉重复分析；参考资料按分类**按需注入**而非全量拼接；接口层 `@RateLimit` 挡刷（见 [tech-qa/10](10-observability-rate-limit.md)）。⚠️ 项目未覆盖：token 用量统计与按 Provider 的成本看板。

---

### Embedding 模型和向量库怎么对齐？

**答：** 三处必须一致：模型输出维度、向量库列维度、检索时的距离度量。换 Embedding 模型意味着**历史向量全部失效**，必须重新向量化，不能新旧混用。

**本仓库：** 维度写死 1024、距离 COSINE（`spring.ai.vectorstore.pgvector.*`），Provider 表里也有 `supports_embedding` / `embedding_model` 字段；配置校验会拦「把 chat 模型名填到 embedding 上」。

---

### 本地模型能不能接进来？

**答：** 能，只要暴露 OpenAI 兼容端点（LM Studio、Ollama、vLLM），当成一个 base-url 指向 `localhost` 的 Provider 即可。好处是零 API 成本、数据不出本机；代价是效果与吞吐受本机限制，结构化输出的稳定性通常明显下降。

**本仓库：** Provider 是数据而不是代码，所以接本地模型不用改任何业务代码——这正是 Registry 抽象换来的收益。

---

## 我的口述清单

1. 面向 `ChatClient` 编程；Provider 是**数据**，可增删改、可热切换。
2. 同一 Provider 按用途拆客户端变体：要 JSON 就别给工具，要连续对话才开历史。
3. 密钥加密存储、不回传明文、缺密钥拒绝启动。
4. 流式给人看，结构化给程序用；两者的失败处理完全不同。
5. 换 Embedding 模型 = 重建向量库，不存在平滑混用。
