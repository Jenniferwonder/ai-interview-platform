# 🎯 AI 面试平台工程实践 —— 从前端到 Java + AI 全栈的公开学习记录

> Forked from [Snailclimb/interview-guide](https://github.com/Snailclimb/interview-guide)
> —— 一个集成了简历分析、模拟面试、RAG 知识库和实时语音的智能面试平台。
>
> 不止是跑通：以**业务链路为线索**，用一系列可追溯的**工程级代码改动**，
> 逐处读懂并验证 Spring AI + RAG + 异步任务 + 实时语音的技术设计，并公开分享全过程。

[![Java](https://img.shields.io/badge/Java-21-orange?logo=openjdk)](https://openjdk.org/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.1-green?logo=springboot)](https://spring.io/projects/spring-boot)
[![Spring AI](https://img.shields.io/badge/Spring%20AI-2.0-blue)](https://spring.io/projects/spring-ai)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-pgvector-336791?logo=postgresql)](https://www.postgresql.org/)
[![Redis](https://img.shields.io/badge/Redis-Stream-DC382D?logo=redis)](https://redis.io/)
[![React](https://img.shields.io/badge/React-18.3-61DAFB?logo=react)](https://react.dev/)

---

## 📖 我在做什么

这是我对 [Snailclimb/interview-guide](https://github.com/Snailclimb/interview-guide) 的**学习型 fork**，也是我践行 **learning in public** 的地方——我把「读懂源码 → 动手改 → 验证」的全过程公开记录下来，踩过的坑、走过的弯路一并留档，希望对同样想从前端补齐 Java 后端 + AI 的人有用。

我将项目拆解为若干**工程级交付物**，每个都超越"加个配置就能跑"的层面，而是**发现真实问题 → 加指标/日志 → 定位根因 → 写代码改 → 验证效果 → 输出数据**的完整工程闭环。

输出形式：
- 🛠️ **代码改动**：可追溯的 git diff，每个交付物有完整的实现 + 测试 + 效果数据（见 `my-learning/code-changes/`）
- 📝 **支撑笔记**：按「项目全貌 → 业务模块 → 技术问答」三层组织（见下），锚定真实源码路径——能顺着链路讲清模块怎么走、能指到文件回答技术问题、并把踩过的坑和取舍写明白
- 🐛 **踩坑记录**：环境搭建中的实际问题诊断与修复过程

> 完整学习路线（L 深挖已有亮点 + G 补齐生产化缺口）见 [LEARNING_PLAN.md](my-learning/LEARNING_PLAN.md)。笔记索引见 [my-learning/README.md](my-learning/README.md)。

**支撑笔记怎么读**（`my-learning/notes/`；总索引见 [my-learning/README.md](my-learning/README.md)）：

| 层 | 目录 | 干什么 |
|----|------|--------|
| 项目全貌 | `notes/01–03` | 本地启动 → 功能全景 → 库表设计，先把项目跑起来、看懂全貌 |
| 方向一 | [`notes/modules/`](my-learning/notes/modules/README.md) | 按业务模块讲清一条链路用到了哪些技术 |
| 方向二 | [`notes/tech-qa/`](my-learning/notes/tech-qa/README.md) | 按核心技术整理高频问答：问题 → 简答 → 本仓库落点 |

**方向一 · 业务模块笔记索引**（[modules/README](my-learning/notes/modules/README.md)）

| 笔记 | 模块 |
|------|------|
| [01 简历分析](my-learning/notes/modules/01-resume.md) | `resume` |
| [02 文字模拟问答](my-learning/notes/modules/02-interview.md) | `interview` |
| [03 实时语音问答](my-learning/notes/modules/03-voiceinterview.md) | `voiceinterview` |
| [04 知识库与 RAG](my-learning/notes/modules/04-knowledgebase.md) | `knowledgebase` |
| [05 日程管理](my-learning/notes/modules/05-interviewschedule.md) | `interviewschedule` |
| [06 模型与语音配置](my-learning/notes/modules/06-llmprovider.md) | `llmprovider` |

**方向二 · 核心技术问答索引**（[tech-qa/README](my-learning/notes/tech-qa/README.md)）

| 笔记 | 主题 |
|------|------|
| [01 Spring Boot](my-learning/notes/tech-qa/01-spring-boot.md) | IoC/DI、三层、统一响应/异常、事务边界、AOP |
| [02 JPA 与事务](my-learning/notes/tech-qa/02-jpa-transaction.md) | `ddl-auto`、派生查询、投影、N+1 |
| [03 Redis](my-learning/notes/tech-qa/03-redis.md) | Redisson、Stream、ACK/重试/幂等 |
| [04 LLM 接入](my-learning/notes/tech-qa/04-llm-integration.md) | `ChatClient`、多 Provider、流式、密钥 |
| [05 Prompt 工程](my-learning/notes/tech-qa/05-prompt-engineering.md) | 模板化、注入防护 |
| [06 RAG](my-learning/notes/tech-qa/06-rag.md) | 分块、Embedding、pgvector、TopK |
| [07 Tool-Calling / Agent](my-learning/notes/tech-qa/07-tool-calling-agent.md) | SkillsTool、ReAct、护栏 |
| [08 AI 应用工作流](my-learning/notes/tech-qa/08-ai-app-dev-workflow.md) | SSE / WebSocket / 异步、断流 |
| [09 存储与文档解析](my-learning/notes/tech-qa/09-file-storage-parsing.md) | S3、去重、Tika、PDF |
| [10 可观测与限流](my-learning/notes/tech-qa/10-observability-rate-limit.md) | Lua 滑动窗口、Micrometer |
| [11 AI 质量与评估](my-learning/notes/tech-qa/11-ai-quality-evaluation.md) | 结构化输出、降级、eval 缺口 |

---

## 🗺️ 学习路线：L 深挖 + G 补齐

> 分两条线：**L 系列**是吃透项目里已有的技术亮点（读懂真实实现 → 改一点 → 验证），**G 系列**是补齐项目没覆盖、但真实上线绕不开的生产化能力（自己动手加）。完整版见 [LEARNING_PLAN.md](my-learning/LEARNING_PLAN.md)。

### L 系列 · 深挖项目已有实现

> 状态：✅ 完成 · 📝 笔记已成篇、动手项待做 · ⬜ 未开始

| 阶段 | # | 主题 | 我要读懂/验证的核心 | 笔记 | 状态 |
|------|---|------|----------------------|------|:--:|
| 0 | L0 | 环境与工程基建 | Docker Compose 编排、端口排查、ddl-auto 陷阱 | [01](my-learning/notes/01-env-setup.md) · [tech-qa/02](my-learning/notes/tech-qa/02-jpa-transaction.md) | ✅ |
| 1 | L1 | Spring Boot 三层地基 | DI / 事务边界 / 派生查询 / 异常体系 | [tech-qa/01](my-learning/notes/tech-qa/01-spring-boot.md) | ✅ |
| 1 | L2 | 限流与 AOP | Lua 滑动窗口、注解驱动、多维度 | [tech-qa/10](my-learning/notes/tech-qa/10-observability-rate-limit.md) | 📝 |
| 1 | L3 | 统一评估 + 文件/导出 | 共用评估、S3、Tika 解析、iText | [tech-qa/11](my-learning/notes/tech-qa/11-ai-quality-evaluation.md) · [tech-qa/09](my-learning/notes/tech-qa/09-file-storage-parsing.md) | 📝 |
| 2 | L4 | Spring AI 多 Provider | ChatClient 缓存、回退、Advisor、密钥加密 | [modules/06](my-learning/notes/modules/06-llmprovider.md) · [tech-qa/04](my-learning/notes/tech-qa/04-llm-integration.md) | 📝 |
| 2 | L5 | 结构化输出与可靠性 | 重试循环、schema 校验、`BeanOutputConverter` | [tech-qa/11](my-learning/notes/tech-qa/11-ai-quality-evaluation.md) | 📝 |
| 2 | L6 | Prompt 工程与注入防护 | 模板化、sanitizer、system/user 分离 | [tech-qa/05](my-learning/notes/tech-qa/05-prompt-engineering.md) | 📝 |
| 2 | L7 | RAG 检索增强全链路 | 向量化、Query Rewrite、TopK/阈值 | [modules/04](my-learning/notes/modules/04-knowledgebase.md) · [tech-qa/06](my-learning/notes/tech-qa/06-rag.md) | 📝 |
| 2 | L8 | Agent / 工具调用 | tool-calling 原理、技能编排 | [tech-qa/07](my-learning/notes/tech-qa/07-tool-calling-agent.md) | 📝 |
| 3 | L9 | Redis Stream 异步 | 消费者组、ACK、Pending 回收、死信 | [tech-qa/03](my-learning/notes/tech-qa/03-redis.md) | 📝 |
| 3 | L10 | 实时语音 WebSocket | VAD 断句、级联管线、首包延迟 | [modules/03](my-learning/notes/modules/03-voiceinterview.md) · [tech-qa/08](my-learning/notes/tech-qa/08-ai-app-dev-workflow.md) | 📝 |

### G 系列 · 补齐项目没覆盖的生产化能力

| # | 主题 | 项目现状 → 我要加什么 | 笔记 | 状态 |
|---|------|------------------------|------|:--:|
| G1 | 认证与鉴权 | 接口裸奔 → Spring Security + JWT + `@PreAuthorize` | 待新增 | ⬜ |
| G2 | 数据库迁移 | 靠 ddl-auto → Flyway 版本化迁移 + `validate` | [tech-qa/02 缺口](my-learning/notes/tech-qa/02-jpa-transaction.md) | ⬜ |
| G3 | 分布式追踪 | 只有指标 → micrometer-tracing + traceId 串链路 | [tech-qa/10 缺口](my-learning/notes/tech-qa/10-observability-rate-limit.md) | ⬜ |
| G4 | 测试体系 | 多 @Disabled → Testcontainers 起真 Redis/PG 跑全链路 | [modules/01 增补](my-learning/notes/modules/01-resume.md) | ⬜ |
| G5 | AI 质量评估 | 有评分无 eval → 建评测集，量化 faithfulness/命中率 | [tech-qa/11 最小方案](my-learning/notes/tech-qa/11-ai-quality-evaluation.md) | ⬜ |
| G6 | 容器化与 CI | 有 compose 无流水线 → 多阶段 Dockerfile + GitHub Actions | 待新增 | ⬜ |
| G7 | SSE 流式可靠性 | 断网丢内容 → 指数退避重试 + 内容不丢 | [tech-qa/08 缺口](my-learning/notes/tech-qa/08-ai-app-dev-workflow.md) | ⬜ |

---

## 🧠 各主题涉及的源码范围

> 每一项都锚定到真实文件，方便对照阅读。

### L 深挖 · 项目已有实现

- **L1 三层地基**：`modules/interview/InterviewController.java` → `service/InterviewPersistenceService.java` → `repository/InterviewSessionRepository.java`；`common/result/Result.java`、`common/exception/*`、`common/config/*Properties.java`
- **L2 限流 AOP**：`common/aspect/RateLimitAspect.java`（Lua + Redisson）、`common/annotation/RateLimit.java`
- **L3 评估与文件基建**：`common/evaluation/UnifiedEvaluationService.java`、`infrastructure/file/*`（S3 / Tika / 去重）、`infrastructure/export/`（iText）、`infrastructure/mapper/`（MapStruct）
- **L4 多 Provider**：`common/ai/LlmProviderRegistry.java`（ChatClient 缓存 / 回退 / 三变体 / Advisor 装配）、`common/config/LlmProviderProperties.java`、`common/ai/ApiPathResolver.java`、`modules/llmprovider/*`（配置落库 + 密钥加密）
- **L5 结构化输出**：`common/ai/StructuredOutputInvoker.java`（重试 + schema 校验 + `app.ai.structured_output.*` 指标）、`resume/service/ResumeGradingService.java`
- **L6 Prompt 安全**：`common/ai/PromptSanitizer.java`、`PromptSecurityConstants.java`、`resources/prompts/*.st`
- **L7 RAG 全链路**：`modules/knowledgebase/service/KnowledgeBaseVectorService.java`、`KnowledgeBaseQueryService.java`（Query Rewrite / 动态 TopK）、`listener/VectorizeStream*`（异步向量化）
- **L8 工具调用**：`common/ai/AgentUtilsConfiguration.java`、`LlmProviderRegistry` 的 tools/voice 变体、`resources/skills/`
- **L9 异步任务**：`common/async/AbstractStreamProducer.java` / `AbstractStreamConsumer.java`、`infrastructure/redis/RedisService.java`（`XAUTOCLAIM` 回收）、各模块 `listener/`
- **L10 实时语音**：`modules/voiceinterview/handler/VoiceInterviewWebSocketHandler.java`、`service/QwenAsrService.java` / `QwenTtsService.java` / `DashscopeLlmService.java`、`config/WebSocketConfig.java`

### G 补齐 · 我要新增的改动范围

- **G1 认证鉴权**：新增 Security 配置 + JWT 过滤器；给 `modules/**/Controller` 加权限，接口按用户隔离
- **G2 数据库迁移**：新增 `db/migration/V1__init.sql`；`application.yml` 的 `ddl-auto` 改 `validate`
- **G3 分布式追踪**：引入 `micrometer-tracing` + OTel/Zipkin，日志加 MDC，覆盖一次 RAG/简历分析链路
- **G4 测试体系**：`app/src/test/` 用 Testcontainers 起 Redis/PG，Mock S3/LLM 跑「简历分析全链路」（`PENDING → COMPLETED / FAILED`）
- **G5 AI 评估**：新增评测集（Q + 期望要点）与度量工具（Java 内 LLM-as-judge），产出「参数 → 指标」对比表
- **G6 容器化与 CI**：完善 `app/Dockerfile`、`docker-compose.yml` 全栈编排；加 GitHub Actions（build + test）
- **G7 SSE 可靠性**：`frontend/src/api/stream.ts` 指数退避重试 + 保留已渲染内容（现状 `fetch`+`ReadableStream`，非 `EventSource`）——把我熟悉的前端流式体验做稳

---

## 📂 目录说明

```
├── README.md              👈 学习产出首页
├── my-learning/
│   ├── LEARNING_PLAN.md   完整学习路线（L 深挖亮点 + G 补齐缺口）
│   ├── README.md          笔记与交付物总索引
│   ├── notes/
│   │   ├── 01–03          项目全貌：启动 / 功能全景 / 库表
│   │   ├── modules/       方向一：按业务模块的技术要点
│   │   └── tech-qa/       方向二：按核心技术的高频问答
│   ├── code-changes/      交付物对应的代码改动与会话清单
│   └── demos/             独立可运行示例（可选）
│
├── app/                   原项目后端源码（Spring Boot）
├── frontend/              原项目前端源码（React）
└── ...                    其余为原项目文件
```

> 💡 **本分支（`my-learning`）**专用于学习产出展示。
> 原项目代码保留在 `master` 分支，定期从上游同步。

---

## 🔄 同步策略

```bash
git checkout master && git pull upstream master          # 从原仓库拉取最新代码
git checkout my-learning && git merge master             # 将上游更新合并到学习分支
```

---

## 🙋 关于我

在职前端开发，日常工作以 React + TypeScript 为主，正把能力边界从前端向 Java 后端 + AI 应用全栈扩展。我以这个 Spring AI + RAG 全栈项目为载体，通过有深度的工程改动（Provider 指标、异步链路测试、RAG 反馈、延迟诊断、SSE 可靠性）来打通从前端到后端、从业务代码到基础设施的完整认知。

我相信最好的学习是公开地学、分享着学：这里的每一篇笔记、每一次改动都尽量写清楚「为什么这么做、怎么验证的」，如果能帮到走同一条路的人，就是它最大的价值。
