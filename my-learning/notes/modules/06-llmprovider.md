# 模块 06 · 模型与语音服务配置（`llmprovider`）

> 源码：`modules/llmprovider/`（`controller/LlmProviderController`、`service/LlmProviderConfigService`、`LlmProviderBootstrapService`、`ApiKeyEncryptionService`、`model/*`、`dto/*`）+ `common/ai/LlmProviderRegistry`、`common/config/LlmProviderProperties`、`LlmEmbeddingConfig`  
> 表：`llm_provider_config`、`llm_global_setting`（见 [库表设计 §6](../03-db-schema-design.md)）  
> 前端：`/settings`；API `/api/llm-provider`

这个模块决定「其它模块调 AI 时拿到的是谁」。它把 Provider 配置从代码里搬到**数据库 + 设置页**，并在运行时热切换，不用改代码、不用重启。

---

## 1. 三层结构

```text
设置页 /settings
   ↓ REST
LlmProviderConfigService     ← CRUD、校验、密钥加解密、设默认、写 YAML（ASR/TTS）
   ↓
llm_provider_config / llm_global_setting     ← 落库
   ↓ 启动或 reload 时读取
LlmProviderRegistry          ← 构造并缓存 ChatClient / EmbeddingModel
   ↓
业务模块（简历、出题、评估、RAG、语音）
```

`LlmProviderBootstrapService` 在启动时做「库里没有就从 YAML 播种」，所以第一次跑不用手工在页面里录一遍。

---

## 2. Registry：三种 ChatClient 变体

业务侧统一走 `getChatClientOrDefault(provider)`，但 Registry 内部按用途缓存了三种客户端：

| 方法 | 缓存 key | 工具 | Advisor |
|------|----------|------|---------|
| `getChatClient` / `getDefaultChatClient` | `{providerId}` | 挂 `interviewSkillsToolCallback` | 完整默认栈 |
| `getPlainChatClient` | `{providerId}:plain` | **不挂** | 只 `SafeGuardAdvisor` |
| `getVoiceChatClient` | `{providerId}:voice` | 挂 skills 工具 | `ToolCallingAdvisor`（开对话历史）+ SafeGuard |

默认 Advisor 栈由 `app.ai.advisors.*` 逐项开关：`tool-call-enabled`（默认开）、`message-chat-memory-enabled`（默认关）、`simple-logger-enabled`（默认关）、`safeguard-enabled`（默认开）。

为什么要拆变体：**出题要稳定 JSON**（用 plain，避免模型跑去调工具）、**语音要能续上下文**（用 voice 变体）、**RAG/JD 解析用默认栈**。这是「同一个 Provider，不同调用姿势」的典型处理。

主启动类还 `exclude` 掉了 Spring AI 的一批 OpenAI 自动配置——因为 Bean 装配由这个 Registry 接管，而不是吃默认单 Provider 自动装配。

---

## 3. 密钥怎么存

`llm_provider_config` 里 API Key 分两列：`api_key_ciphertext`（密文）+ `api_key_nonce`（随机数），由 `ApiKeyEncryptionService` 用 `APP_AI_CONFIG_ENCRYPTION_KEY` 加解密；明文既不进仓库也不落库。相关开关：

| 配置 | 作用 |
|------|------|
| `app.ai.security.api-key-encryption-key` | 加密密钥（来自 `.env`） |
| `app.ai.security.require-encryption-key` | 默认 `true`，缺密钥时拒绝启动/拒绝写入 |

响应 DTO 不回传明文 Key，只给可展示信息——**「配置可改但不可读」**是这里的设计意图。

---

## 4. ASR / TTS 配置：不在库里

语音相关配置走的是另一条路：`app.voice-interview.qwen.asr` / `.tts` 绑到 `VoiceInterviewProperties`，设置页的更新接口会改**运行时属性 + 可写 YAML 文件**（必要时连带写 `.env` 里的 Key），**没有对应 Entity**。

所以排查「设置页改了没生效」时要分清两套存储：Provider 在库里，ASR/TTS 在 YAML。

---

## 5. 涉及的核心技术要点

| 技术 | 在这个模块的体现 | 深入 |
|------|------------------|------|
| 多 Provider 抽象 | Registry + 缓存 + 默认回退，新增 Provider 只配置不改调用方 | [tech-qa/04](../tech-qa/04-llm-integration.md) |
| Spring AI `ChatClient` | 三种变体、Advisor 装配、`ObservationRegistry` 接入 | 同上 |
| Embedding 模型 | `supports_embedding`、`embedding_model`、维度 1024 与 pgvector 对齐 | [tech-qa/06](../tech-qa/06-rag.md) |
| 工具装配 | `AgentUtilsConfiguration` 提供 `SkillsTool`，Registry 决定谁挂 | [tech-qa/07](../tech-qa/07-tool-calling-agent.md) |
| 配置外置与绑定 | `@ConfigurationProperties`（`LlmProviderProperties`、`StructuredOutputProperties`…） | [tech-qa/01](../tech-qa/01-spring-boot.md) |
| 敏感信息处理 | 密文 + nonce 分列、密钥来自环境变量、响应不回明文 | [tech-qa/10](../tech-qa/10-observability-rate-limit.md) |
| 缓存与热重载 | `refresh()` 清 `ConcurrentHashMap` 缓存，下次调用重建客户端 | [tech-qa/04](../tech-qa/04-llm-integration.md) |
| 限流 | 读接口 GLOBAL 30、写接口 GLOBAL 5、测试接口 GLOBAL 10（每秒） | [tech-qa/10](../tech-qa/10-observability-rate-limit.md) |

---

## 6. 数据落点要点

- `llm_global_setting` 是**单例行**（`id=1`），存默认 Chat / Embedding Provider 的**字符串 id**，没有 DB 外键，由 Service 在写入时校验 Provider 存在且启用。
- `builtin` 标记内置 Provider，避免把预置项误删。
- Provider 维度写死 1024 是为了和 `spring.ai.vectorstore.pgvector.dimensions` 对齐——改一边不改另一边会在检索时炸维度。

---

## 7. 我注意到的坑与取舍

| 点 | 说明 |
|----|------|
| 双模式回退 | 仓库不可用时会退回 YAML/env「legacy」模式，行为差异要意识到 |
| 软引用默认 Provider | 删/禁用 Provider 时要顺带检查 `llm_global_setting` 是否还指着它 |
| 连通性测试的安全 | 测试接口需要限时并防内网地址探测（SSRF 面），这类接口天生敏感 |
| Embedding 校验 | 配置里把 chat 模型名填到 embedding 上会被拦；这类「像是配对了其实没有」的错误值得早报错 |
| 缓存清理粒度 | `refresh()` 是整体清空，简单但会让所有 Provider 下次调用重建 |
| 观测缺口 | Registry 只把 `ObservationRegistry` 传给模型，没有自建「按 Provider 的成功率/延迟」指标（学习计划交付物 1 想补的正是这块） |

---

## 8. 想动手改的话

1. **补 Provider 维度指标**：调用成功率、延迟、token 消耗按 `provider` 打标签。
2. **加一个 OpenAI 兼容 Provider**：验证运行时切换与默认回退（学习计划 L4 的小验证）。
3. **收紧连通性测试**：显式禁止内网/回环地址，超时更短。
