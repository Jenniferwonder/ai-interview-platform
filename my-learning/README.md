# 学习产出索引

> **学习方向**：从前端扩展到 AI 全栈
> **学习周期**：2026.07.10 起
> **原项目**：[Snailclimb/interview-guide](https://github.com/Snailclimb/interview-guide)

---

## 交付物路线

完整计划见 [LEARNING_PLAN.md](LEARNING_PLAN.md)。  
产品功能总览见 [ai-interview-platform-features.md](ai-interview-platform-features.md)。  
库表设计与模块逻辑见 [database-schema.md](database-schema.md)。

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

## 支撑笔记

| 笔记 | 对应交付物 |
|------|-----------|
| [01 环境搭建](notes/01-env-setup.md) | 交付物 0 |
| [02 Spring AI 多 Provider 管理](notes/02-spring-ai-provider.md) | 交付物 1 |
| [03 统一评估引擎](notes/03-unified-evaluation.md) | 交付物 3、4 |
| [04 RAG 检索增强全链路](notes/04-rag-pipeline.md) | 交付物 3、6 |
| [05 Redis Stream 异步任务](notes/05-redis-stream-async.md) | 交付物 2 |
| [06 实时语音通信](notes/06-voice-interview.md) | 交付物 5 |
| [07 JPA ddl-auto 与数据丢失](notes/07-jpa-ddl-auto.md) | L0（持久化踩坑） |
| [08 面试列表投影优化](notes/08-interview-list-projection.md) | 留档（常规查询优化，非任务） |
| [10 Spring Boot 三层地基](notes/10-spring-backend-foundations.md) | L1 ✅ |

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
