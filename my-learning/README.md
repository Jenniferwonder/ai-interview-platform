# 学习产出索引

> **学习方向**：从前端扩展到 AI 全栈
> **学习周期**：2026.07.10 起
> **原项目**：[Snailclimb/interview-guide](https://github.com/Snailclimb/interview-guide)

---

## 交付物路线

完整计划见 [LEARNING_PLAN.md](LEARNING_PLAN.md)。  
笔记分三层：**01–03 项目全貌**（跑起来 → 看懂功能 → 看懂数据），然后按两个方向展开——[按业务模块](notes/modules/README.md) 和 [按核心技术高频问答](notes/tech-qa/README.md)。原「三层地基」已并入 [tech-qa/01 Spring Boot](notes/tech-qa/01-spring-boot.md)。建议从 [01](notes/01-env-setup.md) 顺读项目全貌部分。


| #   | 交付物               | 核心问题                                  | 产出                           |
| --- | ----------------- | ------------------------------------- | ---------------------------- |
| 0   | 本地环境一键启动          | 怎么把全套基础设施跑起来？                         | ✅ [→](notes/01-env-setup.md) |
| 1   | LLM Provider 指标补齐 | Provider 维度延迟/成功率/Token 去哪看？（结构化指标已有） | ⬜                            |
| 2   | 简历分析全链路集成测试       | 异步任务链路怎么保证每个环节正确？                     | ⬜                            |
| 3   | RAG 反馈采集（MVP）     | 检索有没有帮助？先留下可查询反馈                      | ⬜                            |
| 4   | Prompt A/B 实验（可选） | 两版 Prompt 怎么量化对比？（离线工具）               | ⬜                            |
| 5   | 语音延迟诊断报告          | 瓶颈在哪一段？（埋点大多已有，重分析）                   | ⬜                            |
| 6   | SSE 可靠性（重试+不丢内容）  | 闪断后如何重试且保留已渲染内容？（基于 stream.ts）        | ⬜                            |


---



## 笔记索引

**① 项目全貌：先跑起来、看懂全貌**（顺读）


| 笔记                                                     | 讲什么                             | 状态  |
| ------------------------------------------------------ | ------------------------------- | --- |
| [01 项目前后端本地启动](notes/01-env-setup.md)                  | Docker 基础设施 + 后端/前端启动 + 8 个启动踩坑 | ✅   |
| [02 项目各模块全功能概览](notes/02-project-features-overview.md) | 六大模块能力、页面路由、流程图                 | ✅   |
| [03 数据库库表设计](notes/03-db-schema-design.md)             | 15 张表字段、关系、状态机                  | ✅   |


**② 方向一：按业务模块梳理**（[notes/modules/](notes/modules/README.md)）

一个模块一篇，回答「做什么 → 一条请求怎么走完 → 用到哪些技术 → 数据落在哪 → 我踩到什么」。


| 笔记                                               | 模块                  |
| ------------------------------------------------ | ------------------- |
| [01 简历分析](notes/modules/01-resume.md)            | `resume`            |
| [02 文字模拟问答](notes/modules/02-interview.md)       | `interview`         |
| [03 实时语音问答](notes/modules/03-voiceinterview.md)  | `voiceinterview`    |
| [04 知识库与 RAG](notes/modules/04-knowledgebase.md) | `knowledgebase`     |
| [05 日程管理](notes/modules/05-interviewschedule.md) | `interviewschedule` |
| [06 模型与语音配置](notes/modules/06-llmprovider.md)    | `llmprovider`       |


**③ 方向二：按核心技术梳理高频问答**（[notes/tech-qa/](notes/tech-qa/README.md)）

每条「问题 → 能口述的简答 → 能指到文件的本仓库落点」；标 ⚠️ 的是我承认还没实现的缺口。


| 笔记                                                                | 主题                                               |
| ----------------------------------------------------------------- | ------------------------------------------------ |
| [01 Spring Boot](notes/tech-qa/01-spring-boot.md)                 | IoC/DI、自动配置、三层、统一响应/异常、事务边界、AOP、Servlet（含日程实读附录） |
| [02 JPA 与事务](notes/tech-qa/02-jpa-transaction.md)                 | `ddl-auto`、派生查询、事务边界、投影、N+1                      |
| [03 Redis](notes/tech-qa/03-redis.md)                             | Redisson、缓存、Stream 消费者组、ACK/重试/幂等                |
| [04 LLM 接入](notes/tech-qa/04-llm-integration.md)                  | `ChatClient`、多 Provider、流式、密钥、成本                 |
| [05 Prompt 工程](notes/tech-qa/05-prompt-engineering.md)            | 模板化、结构化约束、注入防护三层                                 |
| [06 RAG](notes/tech-qa/06-rag.md)                                 | 分块、Embedding、pgvector、改写、TopK、评测                 |
| [07 Tool-Calling / Agent](notes/tech-qa/07-tool-calling-agent.md) | 工具调用流程、SkillsTool、ReAct、护栏                       |
| [08 AI 应用工作流](notes/tech-qa/08-ai-app-dev-workflow.md)            | SSE / WebSocket / 异步、首包延迟、断流、排障                  |
| [09 存储与文档解析](notes/tech-qa/09-file-storage-parsing.md)            | S3 path-style、Key 设计、去重、Tika、PDF                 |
| [10 可观测与限流](notes/tech-qa/10-observability-rate-limit.md)         | AOP + Lua 滑动窗口、Micrometer、Actuator               |
| [11 AI 质量与评估](notes/tech-qa/11-ai-quality-evaluation.md)          | 结构化输出重试、降级、分批评估、eval 缺口                          |


---



## 代码改动

按交付物组织，见 [code-changes/](code-changes/)。


| 目录                                         | 对应交付物                                                                                     | 状态  |
| ------------------------------------------ | ----------------------------------------------------------------------------------------- | --- |
| `code-changes/00-env-setup/`               | 环境搭建 + 踩坑记录；会话清单 [session-2026-07-15.md](code-changes/00-env-setup/session-2026-07-15.md) | ✅   |
| `code-changes/01-llm-observability/`       | LLM Provider 指标补齐                                                                         | ⬜   |
| `code-changes/02-resume-integration-test/` | 简历分析全链路集成测试                                                                               | ⬜   |
| `code-changes/03-rag-feedback-loop/`       | RAG 反馈采集（MVP）                                                                             | ⬜   |
| `code-changes/04-prompt-ab-test/`          | Prompt A/B 实验（可选）                                                                         | ⬜   |
| `code-changes/05-voice-latency-diagnosis/` | 语音延迟诊断报告                                                                                  | ⬜   |
| `code-changes/06-sse-reconnect/`           | SSE 可靠性（stream.ts 重试）                                                                     | ⬜   |


