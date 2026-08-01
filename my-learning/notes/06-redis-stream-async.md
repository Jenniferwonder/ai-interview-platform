# 06 · Redis 与 Redis Stream 异步任务

> 对应源码：`common/async/AbstractStreamProducer.java`、`AbstractStreamConsumer.java`、`AsyncTaskStreamConstants.java`、各模块 `listener/` 包  
> 前置：[04 Spring Boot 三层地基](04-spring-boot-foundations.md)（事务边界为什么不能包住外部调用）

## 设计目标

将耗时操作（简历分析、问答评估、文档向量化）从请求线程中剥离，通过 Redis Stream 解耦生产与消费。

## 核心模板类

### AbstractStreamProducer\<T\>

```java
// 使用方式：继承并调用 send(t)
public abstract class AbstractStreamProducer<T> {
    // 1. 创建 Stream（如不存在）
    // 2. 序列化消息为 JSON
    // 3. XADD 发送到 Stream
    // 4. 可选：设置消息 TTL
}
```

### AbstractStreamConsumer\<T\>

```java
public abstract class AbstractStreamConsumer<T> {
    // 1. 创建消费者组
    // 2. XREADGROUP 拉取 Pending 消息
    // 3. 反序列化 → 调用 processMessage()
    // 4. 成功 → XACK
    // 5. 失败 → 重试（最多 3 次）→ 标记 FAILED
}
```

## 任务生命周期

```
PENDING → 消费者拉取 → PROCESSING
                         ├── 成功 → XACK → 删除
                         └── 失败
                              ├── retry < 3 → 重新 PENDING
                              └── retry >= 3 → FAILED
```

消费前先校验实体是否存在：
- 实体已删除 → ACK 丢弃（避免死循环）
- 实体存在 → 正常处理

## Stream 定义

所有 Stream Key、消费者组、任务状态常量集中在 `AsyncTaskStreamConstants`：

```
resume:analyze      → 简历分析
interview:evaluate  → 面试评估
voice:evaluate      → 语音面试评估
knowledge:vectorize → 文档向量化
```

## 关键设计决策

| 决策 | 选择 | 理由 |
|------|------|------|
| 消息队列 | Redis Stream | 已有 Redis，不想引入 Kafka/RabbitMQ |
| 消费者组 | 按业务模块分组 | 同组内负载均衡，不同组独立消费 |
| 重试次数 | 3 次 | 超过即标记 FAILED，人工介入 |
| 序列化 | JSON | 简单可读，调试方便 |

## Pending 回收与 XAUTOCLAIM

源码：`infrastructure/redis/RedisService.reclaimPendingMessages()` → Redisson `RStream.autoClaim(...)`。

### 作用

消费者组里若有消息被取出后长时间未 ACK（进程崩溃、超时），会进入 Pending 列表。`XAUTOCLAIM` 按空闲时间把这些消息认领到当前消费者，避免任务永久卡在 Pending。

### 版本要求

- `XAUTOCLAIM` 自 **Redis 6.2** 引入。
- 项目 Docker 镜像为 `redis:7`，本身支持。
- 若启动报 `ERR unknown command 'XAUTOCLAIM'`，通常不是 Redisson 4.0 的问题，而是 **应用连到了更旧的 Redis 实例**（例如本机 Windows 老 Redis 占用了 `localhost:6379`）。

### 验证（宿主机，勿只信 docker exec）

```powershell
redis-cli -h 127.0.0.1 -p 6379 info server | findstr redis_version
redis-cli -h 127.0.0.1 -p 6379 command info xautoclaim
```

`docker exec interview-redis redis-cli INFO` 只能证明容器内版本；bootRun 在宿主机时连的是 `localhost:6379` 上**实际响应**的那个进程。

环境侧诊断与处理见 [01 本地启动 §问题 3](01-env-setup.md)。

## 核心要点

- **Redis Stream vs Kafka**：Stream 部署零成本，适合中小规模；Kafka 持久化更强、吞吐更高，适合大规模事件流
- **消费者组机制**：同组内消息只被一个消费者处理，天然负载均衡；不同组独立消费同一 Stream，互不干扰
- **消息可靠性**：ACK 确认 + Pending 消息恢复（`XAUTOCLAIM`）+ 最多 3 次重试 + FAILED 死信状态，形成完整兜底链路
- **为什么不是 `@Async`**：`@Async` 依赖 JVM 内存队列，重启丢消息、无持久化；Stream 消息落盘 Redis，重启可恢复
- **连对 Redis**：Stream 高级命令对版本敏感；本地多 Redis 并存时，以宿主机 `redis-cli` 验证为准

---

## 小白补充：Redis 是什么？Stream 怎么用？

### Redis 是什么？

**Redis** = 内存里的高速键值数据库（也常当缓存、分布式锁、消息队列用）。

和 PostgreSQL 的粗对比：

| | PostgreSQL | Redis |
|--|------------|-------|
| 主要存在哪 | 磁盘（持久库） | 内存（极快；也可持久化） |
| 典型用途 | 业务正式数据（会话、简历） | 缓存、限流、异步消息 |
| 数据结构 | 表/行 | String、Hash、List、Set、**Stream** 等 |

本项目 Docker 里有 `interview-redis`，业务用 **Redisson** 客户端连它。

### Redis 与 Redisson：关系和区别

| | **Redis** | **Redisson** |
|--|-----------|--------------|
| 是什么 | **服务器端程序**：内存数据库 / 缓存 / Stream 等 | **Java 客户端库**：用来在 Java 里连接并操作 Redis |
| 跑在哪 | 独立进程（本项目 Docker：`interview-redis`） | 跟着你的 Spring Boot 应用，在 JVM 里 |
| 类比 | MySQL 服务器 | JDBC / 某个更高级的 MySQL 驱动封装 |
| 谁规定命令 | Redis 协议（`GET`/`SET`/`XADD`…） | 把命令封成 Java API（`RBucket`、`RStream`、`RLock`…） |

**关系：**

```text
你的 Java 代码
  → Redisson（客户端 SDK）
  → 网络
  → Redis 服务器（存数据、执行命令）
```

没有 Redis 服务器，Redisson 连不上；没有客户端，Java 也很难直接用 Redis。

**区别（别混成同一个东西）：**

- Redis = **服务**（也可以换成别的客户端，如 Lettuce、Jedis）
- Redisson = **本项目选用的 Java 客户端**（`redisson-spring-boot-starter`），额外提供分布式锁、易用的 Stream API、与 Spring 集成等

本仓库相关落点：

| 能力 | 大致走 Redisson 的哪类 API |
|------|---------------------------|
| Stream 入队/消费 | `RStream`（经 `RedisService` 封装） |
| 限流 Lua | 与 Redis 脚本执行相关 |
| 连接配置 | `application.yml` 里 `spring.redis.redisson.config` |

一句话：**Redis 是仓库；Redisson 是你手里的进货手推车（Java 版）。**

常见用法（本仓库相关）：

| 用法 | 场景 |
|------|------|
| 缓存 | 文字面试会话状态（`InterviewSessionCache`） |
| 限流 | Lua + Redis 计数（`RateLimitAspect`） |
| **Stream 消息队列** | 评估/分析/向量化等耗时任务 |

---

### Redis Stream 是什么？

**Stream** = Redis 自带的「日志型消息队列」：

- 生产者往某条 Stream **追加**一条消息（命令类似 `XADD`）
- 消费者按**消费者组**拉取（`XREADGROUP`），处理完 **ACK**
- 消息可持久留在 Redis；崩溃未 ACK 的会进 Pending，可用 `XAUTOCLAIM` 回收

类比：

```text
Stream  ≈ 一条传送带（key = interview:evaluate:stream）
消息    ≈ 传送带上的包裹（字段：sessionId、retryCount）
Producer ≈ 往传送带放包裹的人
Consumer ≈ 从传送带取包裹干活的人（同组内一人取走，避免重复干）
```

为什么评估要用 Stream，而不是请求里直接调 LLM？

- LLM 评估可能很慢（十几秒～更久）
- HTTP 请求线程不该一直占用连接等 AI
- 服务重启时，若消息已在 Redis，任务还能被消费者接着做（比纯 `@Async` 内存队列可靠）

---

## 实读：`EvaluateStreamProducer.java`

> 路径：`modules/interview/listener/EvaluateStreamProducer.java`  
> 角色：**文字模拟会话「评估任务」的生产者**——交卷后把「请评估这个 session」丢进 Redis Stream。

### 它在整条链路里的位置

```text
用户交卷
  → InterviewSessionService 把评估状态写成 PENDING
  → EvaluateStreamProducer.sendEvaluateTask(sessionId)   ← 本文件
  → Redis Stream: interview:evaluate:stream
  → EvaluateStreamConsumer 拉取
  → 调 LLM 做评估，结果写回 DB
```

### 类声明逐段看

```java
@Component
public class EvaluateStreamProducer extends AbstractStreamProducer<String> {
```

| 写法 | 含义 |
|------|------|
| `@Component` | 交给 Spring 管理，可被 Service 注入 |
| `extends AbstractStreamProducer<String>` | 复用公共「发 Stream」模板；泛型 `String` = 载荷是 `sessionId` 字符串 |
| `@Slf4j` | Lombok 生成 `log` |

构造器注入三个依赖：

| 依赖 | 干什么 |
|------|--------|
| `RedisService` | 真正执行 `streamAdd`（父类用） |
| `InterviewSessionRepository` | 发送失败时更新库里的评估状态 |
| `TransactionalExecutor` | 失败写库用独立短事务（`REQUIRES_NEW`） |

对外只暴露：

```java
public void sendEvaluateTask(String sessionId) {
    sendTask(sessionId);  // 父类模板
}
```

业务方（如 `InterviewSessionService`）只调这一句，不用关心 Redis 命令细节。

### 必须实现的抽象方法（填空题）

父类 `AbstractStreamProducer` 规定骨架；子类只填「发到哪、消息长什么样、失败怎么办」：

| 方法 | 本类怎么填 | 作用 |
|------|------------|------|
| `taskDisplayName()` | `"评估"` | 日志里好认 |
| `streamKey()` | `interview:evaluate:stream` | 发到哪条传送带 |
| `buildMessage(sessionId)` | `sessionId` + `retryCount=0` | 消息体（Map → Redis 字段） |
| `payloadIdentifier(...)` | `"sessionId=xxx"` | 日志定位用 |
| `onSendFailed(...)` | 单独开事务，把评估状态打成 `FAILED` | 入队失败也要让前端/库知道失败 |

父类 `sendTask` 核心逻辑（简化）：

```text
try:
  redisService.streamAdd(streamKey, buildMessage(...), maxLen)
catch:
  onSendFailed(...)   // 本类写 FAILED
```

### `onSendFailed` 为什么还要动数据库？

若 Redis 挂了或入队异常，消费者永远收不到任务。若不改 DB，会话会一直停在「评估中」。  
所以失败时：

```java
transactionalExecutor.runRequiresNew(() -> updateEvaluateStatus(..., FAILED, error));
```

- `runRequiresNew`：新开短事务写库（和外面可能的事务隔开）  
- `findBySessionId(...).ifPresent(...)`：会话还在才更新（`Optional`）

---

## 和 Consumer 的分工（一张表）

| | Producer（本文件） | Consumer（`EvaluateStreamConsumer`） |
|--|-------------------|--------------------------------------|
| 时机 | 交卷后立刻 | 后台一直轮询/阻塞读 Stream |
| 做什么 | `XADD` 放消息 | `XREADGROUP` 取消息 → 调 LLM → ACK |
| 失败 | 入队失败 → DB FAILED | 处理失败 → 重试 / 最终 FAILED |

读完 Producer，下一步可读同包的 `EvaluateStreamConsumer` 与 `AbstractStreamConsumer`，对照「取消息 + ACK + 重试」。