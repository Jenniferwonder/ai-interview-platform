# 方向二 · 按核心技术梳理高频问答

> 每篇的写法固定：**问题 → 简答（能口述） → 本仓库落点（能指到文件）**。  
> 概念的系统展开我放在本地通用笔记（`SpringBoot.md`、`redis.md`、`ref-LLM.md`、`ref-prompt-engineering.md`、`ref-RAG.md`、`ai-app-dev-workflow.md`）；这里只保留**能用本项目代码印证**的部分，避免变成第二份八股。

| 笔记 | 主题 | 主要印证模块 |
|------|------|--------------|
| [01 Spring Boot](01-spring-boot.md) | IoC/DI、自动配置、三层、统一响应/异常、事务边界、配置绑定、AOP、Servlet（含日程实读） | 全部 |
| [02 JPA 与事务](02-jpa-transaction.md) | Hibernate 关系、`ddl-auto`、派生查询、事务边界、投影与 N+1 | resume / interview / schedule |
| [03 Redis](03-redis.md) | Redisson、缓存、Stream 消费者组、ACK/重试/死信、幂等 | 全部异步链路 |
| [04 LLM 接入](04-llm-integration.md) | Spring AI `ChatClient`、多 Provider、流式、密钥、成本 | llmprovider |
| [05 Prompt 工程](05-prompt-engineering.md) | 模板化、结构化约束、注入防护三层 | interview / resume / knowledgebase |
| [06 RAG](06-rag.md) | 分块、Embedding、pgvector、Query Rewrite、TopK、评测 | knowledgebase |
| [07 Tool-Calling / Agent](07-tool-calling-agent.md) | 工具调用流程、SkillsTool、ReAct、何时不给工具 | interview / voiceinterview |
| [08 AI 应用工作流](08-ai-app-dev-workflow.md) | SSE / WebSocket / 异步、首包延迟、断流处理、排障 | knowledgebase / voiceinterview |
| [09 存储与文档解析](09-file-storage-parsing.md) | S3 path-style、Key 设计、哈希去重、Tika、PDF 导出 | resume / knowledgebase |
| [10 可观测与限流](10-observability-rate-limit.md) | AOP + Lua 滑动窗口、多维度 key、Micrometer、Actuator | 全部 |
| [11 AI 质量与评估](11-ai-quality-evaluation.md) | 结构化输出可靠性、重试与降级、分批评估、eval 缺口 | resume / interview / voiceinterview |

**怎么配合方向一用**

| 我想 | 看哪边 |
|------|--------|
| 搞懂某个功能是怎么串起来的 | [modules/](../modules/README.md) |
| 把某项技术讲清楚、答得出 | 本目录 |
| 找表和字段 | [03 库表设计](../03-db-schema-design.md) |

标注约定：**⚠️ 项目未覆盖** = 这条我只有概念答案，仓库里还没有实现可指，属于我要补的缺口。
