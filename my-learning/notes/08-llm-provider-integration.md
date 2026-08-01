# 08 · Spring AI 多 Provider 接入与结构化输出

> 对应源码：`common/ai/LlmProviderRegistry.java`、`StructuredOutputInvoker.java`、`ApiPathResolver.java`、`modules/llmprovider/`  
> Provider 配置怎么落库（含密钥加密字段）见 [03 库表设计 §6](03-db-schema-design.md)。

## 设计目标

支持多个 LLM Provider（DashScope / Kimi / DeepSeek / GLM / LM Studio / 自定义 OpenAI 兼容），运行时动态切换，新增 Provider 仅需配置无需改代码。

## 核心类分析

### LlmProviderRegistry

```
Provider 配置(DB/yml) → ChatClient 缓存(Map) → 业务代码按 provider 名获取
```

- 维护 `ChatClient` 实例缓存，按 `providerId` 索引
- `getChatClientOrDefault(provider)` — 统一入口，未指定时回退到默认 Provider
- 支持运行时 `refresh()` 热加载，不重启应用
- 提供三种变体：plain / tools-enabled / voice-specific

### StructuredOutputInvoker

LLM 输出的 JSON 不可靠 → 封装重试 + 修复逻辑：

1. 调用 LLM 获取原始文本
2. 尝试 JSON 解析
3. 失败 → 将错误信息注入 Prompt，要求 LLM 修正
4. 重试 N 次（可配置）
5. 仍失败 → 返回降级结果

关键设计：错误信息注入 Prompt 而非正则修复，让 LLM 自己纠正更可靠。

### ApiPathResolver

OpenAI 兼容 API 的路径解析。不同 Provider 的 chat/embedding 端点路径不同，集中管理避免散落。

## 关键设计决策

| 决策 | 选择 | 理由 |
|------|------|------|
| ChatClient 缓存 | `ConcurrentHashMap` | 避免每次请求重建，线程安全 |
| API Key 存储 | DB 加密存储 + `~/.interview-guide/` 落盘 | 解耦源码，运行时可配置 |
| 默认 Provider | `app.ai.default-provider` | 不指定时自动回退 |
| 连通性测试 | 限时 + 防内网访问 | 安全 + 可用性保障 |

## 核心要点

- **策略模式**：不同 Provider = 不同策略，`LlmProviderRegistry` 是上下文，新增 Provider 只需注册无需改动调用方
- **Spring AI 2.0**：`ChatClient` fluent API vs 旧版 `ChatModel`，2.0 提供了更简洁的链式调用和更好的 OpenAI 兼容性
- **为什么不用 `@Primary` 多 Bean**：运行时动态切换，Provider 配置来自 DB 而非 Spring 容器，`@Primary` 无法满足热切换需求
