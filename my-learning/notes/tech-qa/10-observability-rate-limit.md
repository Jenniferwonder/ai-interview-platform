# tech-qa 10 · 可观测性与限流 高频问答

> 落点：`common/annotation/RateLimit`、`common/aspect/RateLimitAspect`、`resources/scripts/rate_limit_single.lua`、`common/exception/RateLimitExceededException`、`common/ai/StructuredOutputInvoker`（指标）、`voiceinterview/handler/VoiceInterviewWebSocketHandler`（指标）、`application.yml` 的 `management.*`。

---

### 为什么限流要放在 AOP + Redis Lua，而不是 Filter 或本地计数？

**答：** 两个原因：

1. **AOP 而非 Filter**：限流规则是「方法级 + 可配额度」的业务语义，需要读方法上的注解元数据；Filter 只看 URL，表达不了「同一个方法两条不同维度的规则」。
2. **Redis Lua 而非本地计数**：本地计数在多实例部署下各算各的，额度会被放大 N 倍；Lua 在 Redis 里**原子执行**「读取-判断-扣减」，避免竞态。

**本仓库：** `RateLimitAspect` 在 `@PostConstruct` 用 `scriptLoad` 预加载脚本，之后每次 `evalSha` 调用（`RScript.Mode.READ_WRITE`，`StringCodec`）。

---

### 固定窗口、滑动窗口、令牌桶有什么区别？项目用的是哪种？

**答：**

| 算法 | 特点 | 问题 |
|------|------|------|
| 固定窗口 | 计数器按窗口重置，最简单 | 窗口边界可瞬时放行 2 倍流量 |
| 滑动窗口 | 记录每次请求时间，按时间轴回收额度 | 需要存时间戳，内存/结构更重 |
| 令牌桶 | 按速率匀速补充令牌，允许突发 | 需要维护补充逻辑 |
| 漏桶 | 匀速流出，平滑输出 | 不允许突发 |

**本仓库是滑动窗口**：每个维度两个 key——`{baseKey}:value` 存当前可用额度，`{baseKey}:permits` 是 ZSET，成员为请求 id、score 为时间戳。每次请求先 `zrangebyscore` 找出窗口外的旧许可、`zremrangebyscore` 清掉并把额度加回（上限为 `max_tokens`），再判断是否够扣。返回 `1` 放行、`0` 拒绝。

注意它**不是**固定窗口（没有整点重置），也不是经典令牌桶（额度来自旧许可过期而非匀速补充）。

---

### 多维度限流的 key 怎么设计？

**答：** key 要包含**规则身份**（哪个方法的哪条规则）+ **限流主体**（全局/IP/用户）：

```text
ratelimit:{ClassName:MethodName}:global
ratelimit:{ClassName:MethodName}:ip:<clientIp>
ratelimit:{ClassName:MethodName}:user:<userId>
```

`{...}` 是 **Redis Cluster 的 hash tag**：保证同一方法的相关 key 落在同一槽位，Lua 才能跨 key 原子操作。IP 的取值顺序是 `X-Forwarded-For`（取第一个）→ `X-Real-IP` → `Proxy-Client-IP` → `WL-Proxy-Client-IP` → `remoteAddr`。

**本仓库：** 维度枚举是 `GLOBAL` / `IP` / `USER`；`USER` 已实现但**尚无接口使用**（项目还没有登录体系）。

---

### `@RateLimit` 可重复是什么意思？多条规则怎么判定？

**答：** 注解带 `@Repeatable`，一个方法可以叠加多条规则，**全部通过才放行**（顺序 AND）。这样能表达「整体每秒 5 次，且单 IP 每秒 5 次」这类组合。

**本仓库典型配置：**

| 接口 | 限流 |
|------|------|
| `POST /api/resumes/upload` | GLOBAL 5 + IP 5 |
| `POST /api/resumes/{id}/reanalyze` | GLOBAL 2 + IP 2 |
| `POST /api/interview/sessions` | GLOBAL 5 + IP 5 |
| `POST /api/interview/sessions/{id}/answers` | GLOBAL 10 |
| `POST /api/knowledgebase/query` | GLOBAL 10 + IP 10 |
| `POST /api/knowledgebase/query/stream` | GLOBAL 5 + IP 5 |
| `POST /api/interview/skills/parse-jd` | IP 5 |
| `/api/llm-provider` 读 / 写 / 测试 | GLOBAL 30 / 5 / 10 |

默认值：`dimension=GLOBAL`、`interval=1`、`timeUnit=SECONDS`；`count` 必填。所以 `@RateLimit(count = 5)` 就是「每秒 5 次」。

---

### 被限流之后返回什么？

**答：** 两条路径：注解配了 `fallback` 就调同类的降级方法；否则抛 `RateLimitExceededException`，由全局异常处理器转成 **HTTP 200 + `Result.error(8001, "请求过于频繁，请稍后再试")`**。

这里有个值得说的取舍：按 HTTP 语义应该返回 **429**，但项目统一了「200 + 业务码」的响应模型（见 [tech-qa/01](01-spring-boot.md)）。代价是网关/监控无法直接从状态码识别限流。

另外要区分两种「限流」：**应用自己的限流是 8001**；**上游模型返回 429** 被映射成 `AI_RATE_LIMIT_EXCEEDED(7005)`，两者根因完全不同。

---

### Redis 重启后 Lua 脚本找不到怎么办？

**答：** `evalSha` 依赖 Redis 端脚本缓存，Redis 重启后缓存清空，会报 `NOSCRIPT`。正确做法是**捕获这个错误、重新 `scriptLoad` 再重试**，而不是每次都用 `eval` 传全文（浪费带宽）。

**本仓库：** `RateLimitAspect` 里做了 NOSCRIPT 的重载处理。

---

### 限流失效或误伤，通常是什么原因？

**答：** 六个常见坑：

1. **同类自调用**：内部方法调用不走代理，注解无效（见 [tech-qa/01](01-spring-boot.md)）；
2. **反代未传真实 IP**：所有请求都算到同一个 IP 上，IP 维度形同虚设或全被误伤；
3. **多实例 + 本地计数**：额度被放大；
4. **窗口太粗**：1 秒窗口对突发友好但对持续压制无效，长任务应该配更长窗口；
5. **只限入口不限成本**：一次请求可能触发多次 LLM 调用，按请求数限流不等于按成本限流；
6. **配置没生效**：项目里 `@RateLimit` 的额度**只在注解上**，`application.yml` 没有全局开关——想集中调额度需要改代码。

**本仓库另一个易误解点**：`app.voice-interview.rate-limit.*`（每会话/每 IP/最大并发）属性存在，但主流程**没有调用**，看到配置不代表已生效。

---

### 项目里有哪些自定义指标？

**答：** 两处，都用 Micrometer。

**① 结构化 LLM 调用**（`StructuredOutputInvoker`）：

| 指标 | 类型 | 标签 |
|------|------|------|
| `app.ai.structured_output.invocations` | Counter | `context`、`status` |
| `app.ai.structured_output.attempts` | Counter | `context`、`status`（每次尝试都记） |
| `app.ai.structured_output.latency` | Timer | `context`、`status` |

开关 `app.ai.structured-metrics-enabled`（默认 true）。

**② 语音链路**（`VoiceInterviewWebSocketHandler`，`ObjectProvider<MeterRegistry>` 可缺省）：

| 指标 | 含义 |
|------|------|
| `app.voice.interview.asr.final_segments` | ASR 最终分段数 |
| `app.voice.interview.asr.merge_wait` | 触发 LLM 前的合并等待 |
| `app.voice.interview.llm.first_token_latency` | **首 token 延迟**（流式体感核心） |
| `app.voice.interview.llm.duration` / `.calls` | LLM 耗时与调用数（带 `streaming` 标签） |
| `app.voice.interview.tts.duration` / `.empty_audio` | TTS 耗时与空音频 |
| `app.voice.interview.turn.duration` / `.completed` | 整轮对话耗时与成败 |
| `app.voice.interview.errors` | 错误计数（带 `stage`） |

**这套「每段单独计时」是项目里最值得复用的做法**——只有分段，才能回答「慢在哪一段」。

---

### 指标标签怎么设计？项目踩了什么坑？

**答：** 原则：标签用于**切分维度**（场景、状态、Provider），必须是**低基数**枚举值；绝不能放 sessionId、用户 id、原始文本这类高基数值，否则时间序列爆炸。

**本仓库的坑：** `StructuredOutputInvoker` 的 `context` 标签由调用方传入，而调用方传的是中文（「简历分析」「方向题」「批次评估」「JD 解析」），规整逻辑只保留 `[a-z0-9_]`，**纯中文会整体塌成 `unknown`**——指标存在但按场景切分不出来。修法是传英文 key。

⚠️ 还缺的维度：没有按 **Provider / 模型** 打标签的成功率与延迟指标（想补的第一项）。

---

### Actuator 暴露了什么？

**答：**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      probes:
        enabled: true
```

即 `/actuator/health`（含 K8s 探针）、`/info`、`/metrics`、`/prometheus`（依赖 `micrometer-registry-prometheus`）。生产上这些端点要与业务端口隔离或加鉴权，不能裸奔。

Spring AI 侧还把 `ObservationRegistry` 传给了模型（`LlmProviderRegistry`），因此框架级的 chat/embedding observation 也能被采集——它和上面 `app.*` 自定义指标是两套东西。

---

### 日志方面做了什么、缺什么？

**答：** 做了：控制台/文件 UTF-8（Windows 下中文日志乱码的常见根因）、`logback-spring.xml` 设好 charset 后 include Boot 的 `base.xml`、SLF4J 占位符 + 异常作为最后一个参数。

⚠️ 缺的：**没有 MDC / traceId**，没有分布式追踪依赖。所以一次请求横跨 HTTP → Stream → 消费者时，只能用业务 id（`sessionId`/`resumeId`）手工串联（学习计划 G3）。

安全上要守住：日志不打完整 Prompt、不打 API Key、不打用户简历原文。

---

### 想给这个项目补可观测性，先做哪一步？

**答：** 我的排序：

1. **修 `context` 标签**（改几行就让现有指标可用）；
2. **加 Provider/模型维度的成功率与延迟**；
3. **traceId 贯穿 HTTP → Stream → 消费者**（异步排障刚需）；
4. **RAG 链路埋点**（检索耗时、命中数、生成耗时——目前只有日志）；
5. 最后才是仪表盘与告警。

原则是**先让已有数据可用，再加新数据**。

---

## 我的口述清单

1. 限流放 AOP（要注解元数据）+ Redis Lua（要跨实例原子）。
2. 本项目是滑动窗口：ZSET 存许可时间戳，过期即回收额度。
3. key 里的 `{}` 是 Cluster hash tag，保证多 key Lua 可原子执行。
4. `evalSha` 要处理 `NOSCRIPT`（Redis 重启）。
5. 指标标签必须低基数；中文标签会塌成 `unknown`，这是我踩到的真坑。
6. 分段计时（ASR/LLM/TTS/整轮）是回答「慢在哪」的唯一办法。
