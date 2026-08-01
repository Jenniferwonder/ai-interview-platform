# 10 · 统一评估引擎（文字 / 语音共用）

> 对应源码：`common/evaluation/UnifiedEvaluationService.java`、`EvaluationReport.java`、`QaRecord.java`、`prompts/interview-evaluation-*.st`  
> 触发方式（异步入队）见 [06 Redis Stream](06-redis-stream-async.md)；结果落库字段见 [03 库表设计](03-db-schema-design.md)。

## 设计目标

文字问答和语音问答共用同一套评估引擎，评估结果可对比。

## 评估流程

```
批量 Q&A 记录
    ↓
分批（每批 N 条，避免 Token 超限）
    ↓
每批独立评估（结构化输出）
    ├── 知识点覆盖度
    ├── 回答准确性
    ├── 表达清晰度
    └── 深度与广度
    ↓
二次汇总（合并各批次结果）
    ↓
最终评估报告
    ↓
降级兜底（LLM 调用失败时）
```

## 核心类分析

### UnifiedEvaluationService

- 输入：`List<QaRecord>`（文字/语音通用模型）
- 分批策略：按 Token 估算自动分组
- 调用 `StructuredOutputInvoker` 获取结构化评估
- 汇总：`interview-evaluation-summary-system.st` Prompt

### EvaluationReport

评估报告结构体，包含维度评分和综合建议。文字和语音面试输出格式一致，可直接对比。

### QaRecord

问答记录模型，统一文字面试和语音面试的数据格式。

## Prompt 设计

```
interview-evaluation-system.st   → 单批评估 System Prompt
interview-evaluation-user.st     → 单批评估 User Prompt
interview-evaluation-summary-system.st → 二次汇总 System Prompt
interview-evaluation-summary-user.st   → 二次汇总 User Prompt
```

## 评估维度

| 维度 | 说明 |
|------|------|
| 知识点覆盖度 | 是否覆盖该方向的核心概念 |
| 回答准确性 | 技术细节是否正确 |
| 表达清晰度 | 逻辑是否通顺、表达是否简洁 |
| 深度与广度 | 是否有深度见解、是否旁征博引 |

## 关键设计决策

| 决策 | 选择 | 理由 |
|------|------|------|
| 分批评估 | 每次 N 条 | 避免 LLM Token 限制 |
| 结构化输出 | `StructuredOutputInvoker` + JSON Schema | 保证评估结果可解析 |
| 二次汇总 | 再次调用 LLM | 合并多批次时保持评估一致性 |
| 降级兜底 | 规则打分 | LLM 调用失败时给出基本评价 |

## 核心要点

- **为什么分批**：LLM context window 有上限，QA 数量多时需分组提交，避免 Token 截断导致评估不完整
- **为什么要二次汇总**：多批独立评估可能评分尺度不一致（前批宽松、后批严格），汇总阶段统一校准
- **降级策略**：LLM 不可用时不能直接报错，切换到规则打分保证基本可用性
- **文字 vs 语音场景差异**：核心引擎相同，但语音场景多一步 ASR 转写，转写质量会影响最终评估准确性
