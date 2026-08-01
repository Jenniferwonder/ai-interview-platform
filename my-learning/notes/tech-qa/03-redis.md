# tech-qa 03 · Redis 高频问答

> 落点：`common/async/AbstractStreamProducer`/`AbstractStreamConsumer`、`infrastructure/redis/RedisService`、各模块 `listener/`、`common/aspect/RateLimitAspect`、`InterviewSessionCache`。

---

### Redis 是什么？为什么快？

**答：** 内存型键值数据库，也常当缓存、分布式锁、消息队列。快的原因：数据在内存、单线程处理命令避免锁竞争与上下文切换、IO 多路复用、数据结构本身为操作优化。注意「单线程」指命令执行主线程，持久化/删除等有后台线程。

**本仓库：** Docker 里的 `interview-redis`（`redis:7`），承担缓存、限流、异步队列三件事。

---

### Redis 和 Redisson 是一回事吗？

**答：** 不是。Redis 是**服务器**（可换客户端：Lettuce、Jedis、Redisson）；Redisson 是**Java 客户端库**，把命令封成 `RBucket`/`RStream`/`RLock` 这类 Java API，还提供分布式锁等高级封装。

```text
Java 代码 → Redisson（SDK）→ 网络 → Redis 服务器
```

**本仓库：** 用 `redisson-spring-boot-starter`，连接配置在 `application.yml` 的 `spring.redis.redisson.config`；`RedisService` 是项目自己的一层封装。

---

### 这个项目把 Redis 用在哪几处？

**答：** 三处，覆盖了 Redis 最常见的三种角色：

| 用途 | 落点 |
|------|------|
| 缓存 | `InterviewSessionCache` 缓存活跃会话，miss 回源 DB |
| 限流 | `RateLimitAspect` + Lua 脚本原子计数（见 [tech-qa/10](10-observability-rate-limit.md)） |
| 消息队列 | Redis Stream 跑简历分析、文字/语音评估、文档向量化 |

---

### Redis Stream 是什么？和 List/Pub-Sub 有什么区别？

**答：** Stream 是「日志型」结构：`XADD` 追加消息、`XREADGROUP` 按消费者组消费、`XACK` 确认。

| 对比 | 特点 |
|------|------|
| Pub/Sub | 发出去就不管，订阅者不在线就丢，无 ACK |
| List（`LPUSH`/`BRPOP`） | 能当队列，但没有消费者组、没有 ACK/重投 |
| **Stream** | 消息持久保留、消费者组负载均衡、Pending 列表 + ACK + 可重认领 |

**本仓库：** Stream key 与消费者组常量集中在 `AsyncTaskStreamConstants`：`resume:analyze`、`interview:evaluate`、`voice:evaluate`、`knowledge:vectorize`。

---

### 消费者组机制解决什么问题？

**答：** 同一组内，一条消息只会被一个消费者取到（水平扩容天然负载均衡）；不同组各自独立消费同一条 Stream（一份数据多种用途）。取走未 ACK 的消息进入该组的 Pending 列表，从而支持「崩溃后重投」。

---

### 消费者崩溃了、消息没 ACK 怎么办？`XAUTOCLAIM` 是干什么的？

**答：** 未 ACK 的消息停留在 Pending 列表，需要有人把它「认领」回来重试。`XAUTOCLAIM`（Redis **6.2+**）按空闲时间批量把超时消息转到当前消费者，避免任务永久卡住。

**本仓库：** `RedisService.reclaimPendingMessages()` → Redisson `RStream.autoClaim(...)`。踩坑记录：启动报 `ERR unknown command 'XAUTOCLAIM'` 时，容器里明明是 Redis 7——真实原因是**应用连到了宿主机上另一个旧 Redis**。诊断要在宿主机执行：

```powershell
redis-cli -h 127.0.0.1 -p 6379 info server | findstr redis_version
redis-cli -h 127.0.0.1 -p 6379 command info xautoclaim
```

`docker exec` 只能证明容器内版本，证明不了应用连的是谁。

---

### 为什么不用 `@Async`，也不用 Kafka / RabbitMQ？

**答：**

| 方案 | 问题 / 适用 |
|------|-------------|
| `@Async` | 队列在 JVM 内存里，重启即丢，没有持久化、重试、可见性 |
| Kafka | 吞吐与持久化更强，适合大规模事件流；代价是多一套集群要运维 |
| **Redis Stream** | 项目已经有 Redis，零新增组件即获得持久化 + 消费者组 + ACK，中小规模够用 |

选型逻辑是「用够用的最小组件」，不是「Stream 比 Kafka 好」。

---

### 消息可靠性是怎么兜住的？

**答：** 四道：

1. **ACK**：处理成功才确认；
2. **Pending 回收**：`XAUTOCLAIM` 捞回超时未确认的；
3. **有限重试**：最多 3 次，`retryCount` 随消息传递；
4. **死信状态**：超限后把业务实体状态打成 `FAILED` 并记错误信息，前端能看到失败而不是永远转圈。

另外还有一条边界处理：**消费前先校验实体是否存在**，实体已删就直接 ACK 丢弃，避免消息在队列里死循环。

---

### 怎么保证不重复处理（幂等）？

**答：** 消息可能被重投，所以消费端要幂等。常见做法：消费前查业务状态，已完成则直接 ACK；或用唯一约束让重复写入变成更新。

**本仓库：** 消费者会检查 `evaluateStatus == COMPLETED` / `vectorStatus == COMPLETED` 就直接丢弃；写入侧靠唯一约束（`(session_id, question_index)`、`file_hash`、`session_id` 唯一）。

---

### 入队本身失败了怎么办？

**答：** 这是很容易漏的分支——Redis 挂了或 `XADD` 抛异常，消费者永远收不到任务，如果不动数据库，业务就永远停在「处理中」。

**本仓库：** `AbstractStreamProducer` 留了 `onSendFailed` 钩子，子类用 `TransactionalExecutor.runRequiresNew` 开独立短事务把状态写成 `FAILED`。

---

### 缓存和数据库的一致性怎么处理？

**答：** 通用答案：更新时删缓存（Cache Aside）、给缓存设过期兜底、接受短暂不一致；强一致要付出很大代价，业务上通常不值得。

**本仓库：** 会话缓存以 **DB 为准**、缓存可随时重建——这类「可重算数据」是最省心的缓存场景，不需要复杂双写策略。

---

### 缓存穿透、击穿、雪崩分别是什么？项目里有吗？

**答：**

| 问题 | 含义 | 常用解法 |
|------|------|----------|
| 穿透 | 查不存在的 key，每次都打到 DB | 空值缓存、布隆过滤器、参数校验 |
| 击穿 | 单个热点 key 过期瞬间大量请求打穿 | 互斥重建（分布式锁）、逻辑过期 |
| 雪崩 | 大量 key 同时过期或 Redis 宕机 | 过期时间加随机、多级缓存、限流降级 |

**本仓库：** 缓存只用于会话这类低基数数据，没有面向海量读的热点缓存，所以这三类问题当前**没有真实场景**。这条我只有概念答案 ⚠️。

---

### Redisson 的分布式锁在项目里用了吗？

**答：** Redisson 提供 `RLock`（可重入、watchdog 自动续期、支持 RedLock）。本项目的并发控制主要靠 **Lua 原子限流** 和数据库唯一约束，⚠️ 没有大规模使用分布式锁。要注意的经典问题：锁超时与业务执行时间不匹配、非原子的「判断+删除」误删他人锁（应带唯一 requestId）。

---

## 我的口述清单

1. Redis 是服务、Redisson 是 Java 客户端，别混为一谈。
2. Stream 相对 List/Pub-Sub 的核心增量是**消费者组 + ACK + Pending 重认领**。
3. 可靠性四件套：ACK、Pending 回收、有限重试、死信状态；再加「实体已删则丢弃」。
4. 入队失败也要落状态，否则任务永远「处理中」。
5. 版本敏感命令（`XAUTOCLAIM` 需 6.2+）要确认**应用真正连的是哪个实例**。
