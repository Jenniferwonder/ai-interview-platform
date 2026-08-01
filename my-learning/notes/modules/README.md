# 方向一 · 按业务模块梳理技术要点

> 源码根：`app/src/main/java/interview/guide/modules/`  
> 每篇回答同一组问题：**这个模块做什么 → 一条请求怎么走完 → 用到了哪些技术 → 数据落在哪 → 我踩到什么**。

| 笔记 | 模块包 | 一句话 | 主要技术面 |
|------|--------|--------|------------|
| [01 简历分析](01-resume.md) | `resume` | 上传文件 → 去重 → 解析 → 异步 AI 评分 → 报告/导出 | 文件上传、Tika、S3、哈希去重、Stream 异步、结构化输出、PDF |
| [02 文字模拟问答](02-interview.md) | `interview` | 技能包出题 → 逐题作答 → 交卷 → 异步评估 | 技能包加载、结构化输出+降级、双状态机、Redis 缓存、事务边界 |
| [03 实时语音问答](03-voiceinterview.md) | `voiceinterview` | WebSocket 全双工：ASR → LLM → TTS | WebSocket、流式级联、VAD、句级并发 TTS、Micrometer 埋点 |
| [04 知识库与 RAG](04-knowledgebase.md) | `knowledgebase` | 文档向量化 + 单次检索 + 多轮流式问答 | pgvector、切块/Embedding、Query Rewrite、SSE、向量任务提升 |
| [05 日程管理](05-interviewschedule.md) | `interviewschedule` | 文本解析成结构化日程 + 定时推进状态 | 派生查询、`@Modifying` 批量更新、`@Scheduled`、DTO 映射 |
| [06 模型与语音配置](06-llmprovider.md) | `llmprovider` | Provider/密钥/默认模型/ASR·TTS 配置 | 配置落库+加密、运行时热重载、客户端缓存、YAML 双模式 |

**横向对照**

| 想看什么 | 去哪 |
|----------|------|
| 表结构与字段 | [03 库表设计](../03-db-schema-design.md) |
| 分层/事务/异常的通用规则 | [tech-qa/01 Spring Boot](../tech-qa/01-spring-boot.md) · [tech-qa/02 JPA](../tech-qa/02-jpa-transaction.md) |
| 某项技术的原理与高频问答 | [方向二 tech-qa](../tech-qa/README.md) |

阅读建议：先读 01（链路最完整、技术面最广），再按兴趣跳。
