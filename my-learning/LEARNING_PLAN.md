# 学习行动计划 — 从前端到 Java + AI 全栈

> **我是谁**：在职前端（React + TS 熟练），有一定 Java 基础，正把能力边界从前端向 Java 后端 + AI 应用全栈扩展。
> **学习载体**：本仓库（Spring Boot 4.1 + Java 21 + Spring AI 2.0 + PostgreSQL/pgvector + Redis + React）。
> **我的方式（learning in public）**：公开记录「读懂真实实现 → 补齐背后概念 → 动手改一点 → 沉淀成可复用笔记」的全过程，把踩过的坑和验证结论都开源出来，让走同一条路（前端转 Java + AI）的人少走弯路。
> **修订（2026-07-16）**：重构为「先吃透本项目的 Java AI 全栈实现，再补齐它没覆盖的生产化能力」。

---

## 一、我要建立的 AI 全栈能力地图

下面是我梳理的「一个 Java AI 全栈应用需要哪些能力」，以及本项目的覆盖情况——这也是我判断「先学什么、还缺什么」的依据。

| 能力域 | 具体内容 | 本项目 |
|--------|----------|:------:|
| Java 后端地基 | Spring Boot 分层、DI、JPA、事务、统一响应/异常、配置管理 | ✅ 完整 |
| LLM 接入 | 多 Provider、ChatClient、流式、结构化输出、重试 | ✅ 完整 |
| Prompt 工程 | 模板管理、注入防护、结构化约束 | ✅ 有 |
| RAG | 向量库、embedding、检索、Query Rewrite、TopK/阈值 | ✅ 完整 |
| Agent / 工具调用 | tool-calling、技能编排 | ✅ 有（agent-utils） |
| 异步 / 消息 | 队列解耦、可靠消费、重试/死信 | ✅ Redis Stream |
| 实时通信 | WebSocket / SSE 流式 | ✅ 完整 |
| 可观测性 | 指标、健康检查（追踪待补） | 🟡 半 |
| 工程质量 | 限流、异常体系、测试 | 🟡 半 |
| 生产化 | 认证鉴权、DB 迁移、部署、CI/CD | ❌ 缺 |
| AI 质量保障 | 评测/eval、幻觉与检索质量度量 | 🟡 有评分无 eval |
| 前端对接 | SSE/WebSocket、类型安全、流式 UI | ✅ 完整 |

本项目已经覆盖了其中大部分核心能力，我先把这些真实实现吃透；剩下的「生产化 + AI 质量保障」是项目里没有的，我自己动手补上并公开记录——这两块既是我最想搞懂的，也是笔记里对他人最有参考价值的干货。

---

## 二、本项目里我要系统深挖的技术亮点（L 系列）

> 每个亮点 = 读源码 → 建立概念 → 做一个小验证 → 沉淀笔记。按 # 顺序推进；「笔记」列对应 [第五节](#五笔记索引按-l--g-分类) 的文件。

| # | 主题 | 读什么（核心源码） | 要建立的概念 | 我的小验证 | 笔记 |
|---|------|-------------------|-------------|-----------|------|
| L0 | 环境与工程基建 ✅ | `docker-compose.dev.yml`、启动日志 | Compose 编排、端口/依赖排查、ddl-auto 陷阱 | 已完成 | [01](notes/01-env-setup.md) · [tech-qa/02](notes/tech-qa/02-jpa-transaction.md) |
| L1 | Spring Boot 三层地基 | `interview` 模块 Controller/Service/Repository；`common/result`、`common/exception`、`common/config/*Properties` | DI 与构造器注入、`@Transactional` 边界、派生查询、`Result<T>`、全局异常体系 | 已完成：笔记 + `existsBySessionId` 派生查询 | [tech-qa/01](notes/tech-qa/01-spring-boot.md) ✅ |
| L2 | Spring AI 多 Provider | `common/ai/LlmProviderRegistry`、`LlmProviderProperties`、`ApiPathResolver`、`modules/llmprovider/*` | `ChatClient`/`ChatModel`、OpenAI 兼容协议、Advisor、多 Provider 抽象、密钥加密 | 加一个 OpenAI 兼容 Provider，验证运行时切换/回退 | [modules/06](notes/modules/06-llmprovider.md) · [tech-qa/04](notes/tech-qa/04-llm-integration.md) |
| L3 | 结构化输出与可靠性 | `common/ai/StructuredOutputInvoker`、`StructuredOutputProperties`、`ResumeGradingService` | LLM JSON 不可靠、`BeanOutputConverter`、重试/降级、判别边界 | 制造坏 JSON，观察重试与指标变化 | [tech-qa/11](notes/tech-qa/11-ai-quality-evaluation.md) |
| L4 | Prompt 工程与注入防护 | `common/ai/PromptSanitizer`、`PromptSecurityConstants`、`resources/prompts/*.st` | 模板化管理、注入攻击与防护、system/user 分离 | 写恶意输入用例验证 sanitizer | [tech-qa/05](notes/tech-qa/05-prompt-engineering.md) |
| L5 | RAG 检索增强全链路 | `knowledgebase/service/KnowledgeBaseVectorService`、`KnowledgeBaseQueryService`、`listener/VectorizeStream*` | embedding 维度/COSINE、HNSW、分块、Query Rewrite、TopK/阈值、召回 vs 精度 | 调 chunk/TopK，人工对比检索差异（为 G5 打基础） | [modules/04](notes/modules/04-knowledgebase.md) · [tech-qa/06](notes/tech-qa/06-rag.md) |
| L6 | Agent / 工具调用 | `common/ai/AgentUtilsConfiguration`、Registry 的 tools/voice 变体、`resources/skills/` | tool-calling 原理、工具注册与编排、Agent vs 纯 RAG | 画一次「LLM 决定调用工具 → 执行 → 回填」时序 | [tech-qa/07](notes/tech-qa/07-tool-calling-agent.md) |
| L7 | Redis Stream 异步 | `common/async/AbstractStreamProducer`/`Consumer`、`infrastructure/redis/RedisService`、各模块 `listener/` | 消费者组、ACK、Pending 回收、死信、幂等、为何不用 `@Async` | 给一个新任务类型走一遍模板 | [tech-qa/03](notes/tech-qa/03-redis.md) |
| L8 | 限流与横切（AOP） | `common/aspect/RateLimitAspect`（Lua + Redisson）、`common/annotation/RateLimit` | AOP 切面、注解驱动、Lua 原子限流、多维度（GLOBAL/IP/USER） | 给接口加 `@RateLimit` 压测触发 | [tech-qa/10](notes/tech-qa/10-observability-rate-limit.md) |
| L9 | 实时语音 WebSocket | `voiceinterview/handler/VoiceInterviewWebSocketHandler`、`QwenAsrService`/`QwenTtsService`/`DashscopeLlmService`、`WebSocketConfig` | WebSocket/SSE/WebRTC、ASR→LLM→TTS 级联、边生成边合成、首包延迟 | 标注一轮对话各段 Micrometer 指标 | [modules/03](notes/modules/03-voiceinterview.md) · [tech-qa/08](notes/tech-qa/08-ai-app-dev-workflow.md) |
| L10 | 统一评估 + 文件/导出 | `common/evaluation/UnifiedEvaluationService`、`infrastructure/file/*`、`export/`、`mapper/` | 文字/语音共用评估、S3 兼容存储、Tika 解析、MapStruct | 读懂评估装配与文件解析链路 | [tech-qa/11](notes/tech-qa/11-ai-quality-evaluation.md) · [tech-qa/09](notes/tech-qa/09-file-storage-parsing.md) |

> 备注：列表查询的 JPQL 投影分析（`findAll()` 整行加载 → 只查 DTO 列）属常规查询优化，并入 [tech-qa/02](notes/tech-qa/02-jpa-transaction.md) 留档，不单列为学习任务。

---

## 三、本项目没覆盖、我要补齐的工程化能力（G 系列）

> 这些是本项目**没有或很弱**、但一个能真正上线的 AI 应用绕不开的能力。

| # | 主题 | 现状 → 我要加什么 | 我的验收 | 笔记 |
|---|------|------------------|----------|:--:|
| G1 | 认证与鉴权 | 接口裸奔 → `spring-boot-starter-security` + JWT 登录 + `SecurityFilterChain` + `@PreAuthorize`，按用户隔离 | 未登录 401、越权 403、`/api/**` 需 token | ⬜ 待新增 |
| G2 | 数据库迁移 | 靠 ddl-auto（[tech-qa/02](notes/tech-qa/02-jpa-transaction.md) 踩过丢数据）→ Flyway `V1__init.sql`，`ddl-auto` 改 `validate` | 重启不依赖自动建表、可版本化回滚 | [tech-qa/02 缺口](notes/tech-qa/02-jpa-transaction.md) · ⬜ 待新增 Flyway 专篇 |
| G3 | 分布式追踪与日志 | 只有指标 → `micrometer-tracing` + OTel/Zipkin，日志加 MDC，覆盖一次 RAG/简历链路 | 一次请求能看到跨 Service/Stream 的 span | [tech-qa/10 缺口](notes/tech-qa/10-observability-rate-limit.md) · ⬜ 待新增 |
| G4 | 测试体系 | Redis 测试多 `@Disabled` → Testcontainers 起真 Redis/PG，Mock S3/LLM 跑全链路 | `./gradlew :app:test` 默认跑通链路（PENDING→COMPLETED/FAILED） | 增补 [tech-qa/03](notes/tech-qa/03-redis.md) · [modules/01](notes/modules/01-resume.md) |
| G5 | AI 质量评估 | 有评分无 eval → 建 20~30 条评测集，Java 内 LLM-as-judge 度量 faithfulness/命中率 | 「参数改动 → 指标变化」对比表 + 结论 | [tech-qa/11 最小方案](notes/tech-qa/11-ai-quality-evaluation.md) · ⬜ 待落地 |
| G6 | 容器化与 CI | 有 compose 无流水线 → 多阶段 `Dockerfile` + 全栈 compose + GitHub Actions | 一条命令起全栈、PR 触发 CI（build + test） | ⬜ 待新增 |
| G7 | SSE 流式可靠性 | 断网丢已渲染内容 → `frontend/src/api/stream.ts` 指数退避重试 + 内容保留/按 messageId 补齐 | 断网可恢复且不丢已渲染内容 | 增补 [tech-qa/08](notes/tech-qa/08-ai-app-dev-workflow.md) · [modules/04](notes/modules/04-knowledgebase.md) |



---

## 四、分阶段学习路线

```mermaid
flowchart TD
    P0["阶段0 环境 已完成"] --> P1
    P1["阶段1 后端地基 L1,L8,L10"] --> P2
    P2["阶段2 AI 核心 L2,L3,L4,L5,L6"] --> P3
    P3["阶段3 工程化 L7,L9 + G3,G4"] --> P4
    P4["阶段4 生产化+AI质量 G1,G2,G5,G6,G7"]
```

| 阶段 | 目标 | 学习项 | 主要产出 |
|------|------|--------|----------|
| 1 后端地基 | 能读懂/改任一模块 | L1、L8、L10 | [tech-qa/01](notes/tech-qa/01-spring-boot.md)、[tech-qa/10](notes/tech-qa/10-observability-rate-limit.md)、[tech-qa/09](notes/tech-qa/09-file-storage-parsing.md)、[tech-qa/11](notes/tech-qa/11-ai-quality-evaluation.md) |
| 2 AI 核心 | 掌握 LLM/RAG/Agent | L2、L3、L4、L5、L6 | [modules/06](notes/modules/06-llmprovider.md)、[tech-qa/04](notes/tech-qa/04-llm-integration.md)、[tech-qa/05](notes/tech-qa/05-prompt-engineering.md)、[modules/04](notes/modules/04-knowledgebase.md)、[tech-qa/06](notes/tech-qa/06-rag.md)、[tech-qa/07](notes/tech-qa/07-tool-calling-agent.md) |
| 3 工程化 | 异步/实时/可观测/可测 | L7、L9、G3、G4 | [tech-qa/03](notes/tech-qa/03-redis.md)、[modules/03](notes/modules/03-voiceinterview.md)、[tech-qa/08](notes/tech-qa/08-ai-app-dev-workflow.md) + 集成测试 |
| 4 生产化 | 认证/迁移/评估/部署 | G1、G2、G5、G6、G7 | 待新增专篇；缺口说明已挂在对应 tech-qa / modules |

**我的执行原则**：
1. 每个学习项先「读代码 + 画一张图」再动手，不让自己停在「看过但没懂」
2. 小改动优先（加一个方法/一条测试/一个配置），先跑通再深入
3. 每项沉淀公开笔记：能顺着链路讲清、能指到文件回答技术问题、把踩坑与取舍写明白——这是我践行 learning in public 的方式
4. 生产化缺口（G 系列）是我重点投入的部分，也是我最想帮到同路人的干货

---

## 五、笔记索引（按 L / G 分类）

完整目录索引也可从 [my-learning/README.md](README.md) 进入。

**笔记组织约定**（2026-08-01 重排）

不再用单一序列编号，改成「项目全貌 + 两个方向」：

- [`notes/01–03`](#51-项目全貌)：项目全貌——本地启动、功能全景、库表设计。
- [`notes/modules/`](notes/modules/README.md)：**方向一**，按业务模块各一篇，讲清链路与该模块涉及的技术面。
- [`notes/tech-qa/`](notes/tech-qa/README.md)：**方向二**，按核心技术各一篇高频问答（问题 → 简答 → 本仓库落点）；Spring Boot 三层地基归入 [tech-qa/01](notes/tech-qa/01-spring-boot.md)。
- 一个 L/G 项通常横跨两个方向：模块篇给「在哪用」，tech-qa 篇给「怎么讲清楚」。

### 5.1 项目全貌

| 笔记 | 讲什么 |
|------|--------|
| [01 本地启动](notes/01-env-setup.md) | Docker 基础设施 + 后端/前端启动 + 踩坑 |
| [02 功能全景](notes/02-project-features-overview.md) | 六大模块能力、页面路由、流程图 |
| [03 库表设计](notes/03-db-schema-design.md) | 表字段、关系、状态机 |

### 5.2 方向一 · 按业务模块

| 笔记 | 模块 |
|------|------|
| [01 简历分析](notes/modules/01-resume.md) | `resume` |
| [02 文字模拟问答](notes/modules/02-interview.md) | `interview` |
| [03 实时语音问答](notes/modules/03-voiceinterview.md) | `voiceinterview` |
| [04 知识库与 RAG](notes/modules/04-knowledgebase.md) | `knowledgebase` |
| [05 日程管理](notes/modules/05-interviewschedule.md) | `interviewschedule` |
| [06 模型与语音配置](notes/modules/06-llmprovider.md) | `llmprovider` |

索引：[modules/README.md](notes/modules/README.md)

### 5.3 方向二 · 按核心技术问答

| 笔记 | 主题 |
|------|------|
| [01 Spring Boot](notes/tech-qa/01-spring-boot.md) | IoC/DI、三层、统一响应/异常、事务边界、AOP |
| [02 JPA 与事务](notes/tech-qa/02-jpa-transaction.md) | `ddl-auto`、派生查询、投影、N+1 |
| [03 Redis](notes/tech-qa/03-redis.md) | Redisson、Stream、ACK/重试/幂等 |
| [04 LLM 接入](notes/tech-qa/04-llm-integration.md) | `ChatClient`、多 Provider、流式、密钥 |
| [05 Prompt 工程](notes/tech-qa/05-prompt-engineering.md) | 模板化、注入防护 |
| [06 RAG](notes/tech-qa/06-rag.md) | 分块、Embedding、pgvector、TopK |
| [07 Tool-Calling / Agent](notes/tech-qa/07-tool-calling-agent.md) | SkillsTool、ReAct、护栏 |
| [08 AI 应用工作流](notes/tech-qa/08-ai-app-dev-workflow.md) | SSE / WebSocket / 异步、断流 |
| [09 存储与文档解析](notes/tech-qa/09-file-storage-parsing.md) | S3、去重、Tika、PDF |
| [10 可观测与限流](notes/tech-qa/10-observability-rate-limit.md) | Lua 滑动窗口、Micrometer |
| [11 AI 质量与评估](notes/tech-qa/11-ai-quality-evaluation.md) | 结构化输出、降级、eval 缺口 |

索引：[tech-qa/README.md](notes/tech-qa/README.md)

### 5.4 L / G → 笔记对照

**L 系列 · 深挖项目已有亮点**

| 项 | 笔记 | 状态 |
|----|------|:--:|
| L0 | [01-env-setup](notes/01-env-setup.md) · [tech-qa/02](notes/tech-qa/02-jpa-transaction.md) | ✅ 完成 |
| L1 | [tech-qa/01](notes/tech-qa/01-spring-boot.md) | ✅ 完成 |
| L2 · L3 | [modules/06](notes/modules/06-llmprovider.md) · [tech-qa/04](notes/tech-qa/04-llm-integration.md) · [tech-qa/11](notes/tech-qa/11-ai-quality-evaluation.md) | 📝 已成篇，待补 Provider 指标实操 |
| L4 | [tech-qa/05](notes/tech-qa/05-prompt-engineering.md) | 📝 已成篇，待补 A/B 记录 |
| L5 | [modules/04](notes/modules/04-knowledgebase.md) · [tech-qa/06](notes/tech-qa/06-rag.md) | 📝 已成篇，待补调参实验 |
| L6 | [tech-qa/07](notes/tech-qa/07-tool-calling-agent.md) | 📝 已成篇，待补工具埋点 |
| L7 | [tech-qa/03](notes/tech-qa/03-redis.md) | 📝 已成篇，待补集成测试（见 G4） |
| L8 | [tech-qa/10](notes/tech-qa/10-observability-rate-limit.md) | 📝 已成篇 |
| L9 | [modules/03](notes/modules/03-voiceinterview.md) · [tech-qa/08](notes/tech-qa/08-ai-app-dev-workflow.md) | 📝 已成篇，待补延迟诊断报告 |
| L10 | [tech-qa/11](notes/tech-qa/11-ai-quality-evaluation.md) · [tech-qa/09](notes/tech-qa/09-file-storage-parsing.md) | 📝 已成篇 |

**G 系列 · 补齐工程化能力**

| 项 | 笔记 | 状态 |
|----|------|:--:|
| G1 | 认证鉴权（Spring Security + JWT） | ⬜ 待新增 |
| G2 | Flyway 迁移（缺口见 [tech-qa/02](notes/tech-qa/02-jpa-transaction.md)） | ⬜ 待新增 |
| G3 | traceId 贯穿与追踪（缺口见 [tech-qa/10](notes/tech-qa/10-observability-rate-limit.md)） | ⬜ 待新增 |
| G4 | 异步链路集成测试（增补 [tech-qa/03](notes/tech-qa/03-redis.md) · [modules/01](notes/modules/01-resume.md)） | ⬜ 待增补 |
| G5 | RAG 评测集（最小方案见 [tech-qa/11](notes/tech-qa/11-ai-quality-evaluation.md)） | ⬜ 待新增 |
| G6 | 容器化与 CI | ⬜ 待新增 |
| G7 | SSE 可靠性（增补 [tech-qa/08](notes/tech-qa/08-ai-app-dev-workflow.md) · [modules/04](notes/modules/04-knowledgebase.md)） | ⬜ 待增补 |

代码改动按 [`code-changes/`](code-changes/) 目录组织；G 系列每完成一项，新增对应目录与 diff。

---

## 六、成果主线：走完这条路我能讲清楚、也能教别人的能力

这条路径走完，我希望能把下面每一块都讲清楚原理、说明白取舍，并通过公开笔记帮到同样从前端补齐 Java 后端 + AI 的人：

- **Java AI 全栈**：基于 Spring Boot 4 + Spring AI 2，读懂并动手扩展多 Provider LLM 接入、结构化输出容错、RAG（pgvector）、tool-calling 与实时语音（WebSocket）全链路
- **工程化**：Redis Stream 异步任务（ACK/重试/死信/XAUTOCLAIM 回收）、AOP + Lua 分布式限流、Micrometer 指标与分布式追踪
- **生产化**：Spring Security + JWT 鉴权、Flyway 迁移、Testcontainers 集成测试、Docker + CI
- **AI 质量**：为 RAG 建评测集并用数据驱动调参（faithfulness/命中率），把「改得好不好」量化下来
- **前端衔接**：React + TS 的 SSE/WebSocket 流式 UI 与断连恢复
