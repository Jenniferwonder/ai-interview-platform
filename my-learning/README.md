# 学习产出索引

> **学习方向**：从前端扩展到 AI 全栈
> **学习周期**：2026.07.10 起
> **原项目**：[Snailclimb/interview-guide](https://github.com/Snailclimb/interview-guide)

---

## 交付物路线

完整计划见 [LEARNING_PLAN.md](LEARNING_PLAN.md)。  
笔记按「跑起来 → 看懂功能 → 看懂数据 → 逐个技术要点」排序，建议从 [01](notes/01-env-setup.md) 顺读。

| # | 交付物 | 核心问题 | 产出 |
|---|--------|----------|:--:|
| 0 | 本地环境一键启动 | 怎么把全套基础设施跑起来？ | ✅ [→](notes/01-env-setup.md) |
| 1 | LLM Provider 指标补齐 | Provider 维度延迟/成功率/Token 去哪看？（结构化指标已有） | ⬜ |
| 2 | 简历分析全链路集成测试 | 异步任务链路怎么保证每个环节正确？ | ⬜ |
| 3 | RAG 反馈采集（MVP） | 检索有没有帮助？先留下可查询反馈 | ⬜ |
| 4 | Prompt A/B 实验（可选） | 两版 Prompt 怎么量化对比？（离线工具） | ⬜ |
| 5 | 语音延迟诊断报告 | 瓶颈在哪一段？（埋点大多已有，重分析） | ⬜ |
| 6 | SSE 可靠性（重试+不丢内容） | 闪断后如何重试且保留已渲染内容？（基于 stream.ts） | ⬜ |

---

## 笔记索引

**① 先跑起来、看懂全貌**

| 笔记 | 讲什么 | 状态 |
|------|--------|:--:|
| [01 项目前后端本地启动](notes/01-env-setup.md) | Docker 基础设施 + 后端/前端启动 + 8 个启动踩坑 | ✅ |
| [02 项目各模块全功能概览](notes/02-project-features-overview.md) | 六大模块能力、页面路由、流程图 | ✅ |
| [03 数据库库表设计](notes/03-db-schema-design.md) | 15 张表字段、关系、状态机 | ✅ |

**② 各功能模块的技术要点与踩坑**

| 笔记 | 讲什么 | 状态 |
|------|--------|:--:|
| [04 Spring Boot 三层地基](notes/04-spring-boot-foundations.md) | 分层、DI、事务边界、统一响应/异常、JPA 入门 | ✅ |
| [05 JPA 持久化实操](notes/05-jpa-persistence.md) | `ddl-auto` 丢数据、列表查询 DTO 投影 | ✅ |
| [06 Redis 与 Redis Stream](notes/06-redis-stream-async.md) | Redis/Redisson、消费者组、ACK、重试与死信 | ✅ |
| [07 对象存储与文档解析](notes/07-file-storage-parsing.md) | RustFS/S3 path-style、哈希去重、Tika、PDF 导出 | ✅ |
| [08 Spring AI 多 Provider](notes/08-llm-provider-integration.md) | Registry 缓存、结构化输出重试、密钥加密 | 🟡 待增补 |
| [09 RAG 检索增强全链路](notes/09-rag-pipeline.md) | 分块、Embedding、Query Rewrite、TopK、SSE | 🟡 待增补 |
| [10 统一评估引擎](notes/10-evaluation-engine.md) | 分批评估、二次汇总、降级兜底 | 🟡 待增补 |
| [11 实时语音链路](notes/11-voice-realtime.md) | WebSocket、ASR/LLM/TTS 级联、VAD、首包延迟 | 🟡 待增补 |

**③ 计划中（编号续排）**

`12` Prompt 工程与注入防护 · `13` Agent 工具调用 · `14` 限流与 AOP · `15` 认证鉴权 · `16` Flyway 迁移 · `17` 追踪与可观测 · `18` RAG 质量评估 · `19` 容器化与 CI

---

## 代码改动

按交付物组织，见 [code-changes/](code-changes/)。

| 目录 | 对应交付物 | 状态 |
|------|-----------|:--:|
| `code-changes/00-env-setup/` | 环境搭建 + 踩坑记录；会话清单 [session-2026-07-15.md](code-changes/00-env-setup/session-2026-07-15.md) | ✅ |
| `code-changes/01-llm-observability/` | LLM Provider 指标补齐 | ⬜ |
| `code-changes/02-resume-integration-test/` | 简历分析全链路集成测试 | ⬜ |
| `code-changes/03-rag-feedback-loop/` | RAG 反馈采集（MVP） | ⬜ |
| `code-changes/04-prompt-ab-test/` | Prompt A/B 实验（可选） | ⬜ |
| `code-changes/05-voice-latency-diagnosis/` | 语音延迟诊断报告 | ⬜ |
| `code-changes/06-sse-reconnect/` | SSE 可靠性（stream.ts 重试） | ⬜ |
