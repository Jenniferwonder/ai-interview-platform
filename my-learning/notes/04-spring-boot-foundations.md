# 04 · Spring Boot 三层架构地基

> 对应学习项：LEARNING_PLAN **L1**  
> 对照通用笔记：本地 `TechSkills/.../spring-framework/SpringBoot.md`（概念 + 高频问答）  
> 本文：把同一套概念钉到 **本仓库真实代码**上——读懂一条请求怎么走完、事务边界在哪、统一响应/异常怎么接。这也是「4 起技术要点」系列的第一篇，后续的 Redis、存储、AI 各篇都建立在这套分层之上。

## 我要建立的能力

学完本篇后，我应能：

1. 沿着 `Controller → Service → Repository` 讲清一次业务请求（如会话列表）的分层职责  
2. 解释 **IoC / DI** 与项目里的 **构造器注入**（`@RequiredArgsConstructor`）  
3. 说清 **`@Transactional` 边界**：短 DB 临界区可以包；**LLM / HTTP / Redis Stream 发送不要包进同一事务**  
4. 用 `Result<T>` + `BusinessException` + `GlobalExceptionHandler` 讲统一响应与全局异常  
5. 指出配置应走 `@ConfigurationProperties`，而不是在 Service 里散落 `@Value`  
6. 认出 Spring Data JPA **派生查询**，并知道列表场景别滥用 `findAll()`（详见 [05](05-jpa-persistence.md)）

---

## 1. 从一条列表请求看三层

入口：`GET /api/interview/sessions`

```51:56:app/src/main/java/interview/guide/modules/interview/InterviewController.java
@GetMapping("/api/interview/sessions")
public Result<List<SessionListItemDTO>> listSessions() {
    List<SessionListItemDTO> items = persistenceService.findAll().stream()
        .map(SessionListItemDTO::from)
        .toList();
    return Result.success(items);
}
```

| 层 | 本项目角色 | 本例落点 |
|----|-----------|----------|
| **Controller** | 路由、参数接收、包装 `Result`，**不做业务编排** | `InterviewController` |
| **Service** | 业务编排、事务边界、领域规则 | `InterviewPersistenceService.findAll()` |
| **Repository** | 持久化访问（JPA） | `InterviewSessionRepository.findAllByOrderByCreatedAtDesc()` |

DTO（如 `SessionListItemDTO`）是对外形状；**Entity 不直接返回前端**（`AGENTS.md` / 项目约定）。

创建会话的编排更典型：`InterviewSessionService.createSession` 里先读历史题、调 LLM 出题、写 Redis，再调 `persistenceService.saveSession` 落库——**出题（LLM）与落库是分开的步骤**，这是后面事务边界的活证据。

---

## 2. IoC / DI：谁 `new`，谁注入

| 概念 | 一句话 |
|------|--------|
| **IoC** | 对象生命周期交给容器，业务代码不再自己 `new` 依赖 |
| **DI** | 容器把依赖「塞进」目标对象；本项目优先 **构造器注入** |

项目惯例：

```java
@Slf4j
@Service
@RequiredArgsConstructor  // Lombok 生成 final 字段的构造器 → Spring 构造器注入
public class InterviewPersistenceService {
    private final InterviewSessionRepository sessionRepository;
    private final InterviewAnswerRepository answerRepository;
    // ...
}
```

对照通用笔记：字段 / setter 注入能「看起来方便」，但构造器注入让依赖不可变、易测、循环依赖更早暴露。Boot 2.6+ 对循环依赖更严格——实践上就是 **构造器注入 + 避免环**。

主启动类：

```17:30:app/src/main/java/interview/guide/App.java
@EnableScheduling
@SpringBootApplication(exclude = {
    OpenAiAudioSpeechAutoConfiguration.class,
    // ... 排除 Spring AI 默认 OpenAI 自动配置
})
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}
```

`@SpringBootApplication` ≈ `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan`。这里 **exclude** 掉一批 OpenAI 自动配置，是因为项目用自建的 `LlmProviderRegistry` 管多 Provider，而不是吃默认单 Provider 自动装配（L2 再深挖）。

---

## 3. `@Transactional`：包什么，不包什么

### 3.1 本项目怎么用

持久化写操作集中在 Service，并带 `rollbackFor = Exception.class`（默认只回滚 Runtime；这里把受检异常也纳入回滚）：

```46:47:app/src/main/java/interview/guide/modules/interview/service/InterviewPersistenceService.java
@Transactional(rollbackFor = Exception.class)
public InterviewSessionEntity saveSession(...) {
```

规则（与 `AGENTS.md` 一致）：

- `@Transactional` **只放 Service**，Controller 不写事务  
- 事务范围尽量小：只覆盖「读改本地库」的短临界区  
- **同类内部 `this.xxx()` 调用带 `@Transactional` 的方法会失效**（走的是代理）；需要时用独立 Bean，例如：

```9:28:app/src/main/java/interview/guide/common/transaction/TransactionalExecutor.java
/**
 * 小范围事务执行器，用于避免同类内部调用导致事务注解失效。
 */
@Service
public class TransactionalExecutor {
    @Transactional(rollbackFor = Exception.class)
    public void run(Runnable action) { action.run(); }

    @Transactional(rollbackFor = Exception.class, propagation = Propagation.REQUIRES_NEW)
    public void runRequiresNew(Runnable action) { action.run(); }
}
```

`EvaluateStreamProducer` / `AnalyzeStreamProducer` 在 **发送 Stream 失败** 时，用 `runRequiresNew` 单独开事务把状态打成 `FAILED`——DB 写入与 Redis 发送解耦。

### 3.2 为什么事务不要包住 LLM / HTTP

`@Transactional` 会占用连接池里的连接直到方法结束。LLM 常 10～60s：

| 风险 | 后果 |
|------|------|
| 连接长期占用 | 池耗尽，其它请求拿不到连接 |
| 外部调用不可回滚 | 超时/失败时 DB 回滚语义混乱 |
| 与异步入队混用 | 未提交数据就被消费者读到（脏读时机） |

**正确拆分（本项目模式）**：

```text
请求线程：
  ① 短事务：写 PENDING / 落本地状态
  ② 事务外：发 Redis Stream / 调 LLM
  ③ 失败时：REQUIRES_NEW 短事务写 FAILED
```

`InterviewSessionService.enqueueEvaluationTask`：先 `updateEvaluateStatus(...PENDING)`，再 `evaluateStreamProducer.sendEvaluateTask`——评估真正跑 LLM 在 **Consumer** 里，不在请求事务中。

> 概念补充见 SpringBoot.md「为什么说事务不要包住 LLM / HTTP」；本文给的是仓库证据链。

---

## 4. 统一响应与全局异常

### 4.1 `Result<T>`

对外统一信封：`code` / `message` / `data`。成功走 `Result.success(...)`，失败走 `Result.error(...)`。

### 4.2 `BusinessException` + `ErrorCode`

业务失败 **不要** `throw new RuntimeException(...)`，而用：

```java
throw new BusinessException(ErrorCode.INTERVIEW_SESSION_NOT_FOUND);
// 或带定制文案
throw new BusinessException(ErrorCode.INTERVIEW_QUESTION_NOT_FOUND, "无效的问题索引: " + index);
```

`ErrorCode` 按模块分段（简历 2xxx、会话 3xxx、AI 7xxx、限流 8xxx…），方便前端与日志对齐。

### 4.3 `GlobalExceptionHandler`

`@RestControllerAdvice` 兜底：

| 异常 | 处理 |
|------|------|
| `BusinessException` | `Result.error(code, message)` |
| `MethodArgumentNotValidException` / `BindException` | 校验失败文案 → `BAD_REQUEST` |
| `ResourceAccessException` / `RestClientException` | AI 网络/鉴权/限流 → 7xxx |
| 其它 `Exception` | `INTERNAL_ERROR`，日志打全量 |

注意：这里多用 `@ResponseStatus(HttpStatus.OK)`，即 **HTTP 200 + 业务码**。前后端约定看 `Result.code`，而不是只看 HTTP 状态。这是本项目的选择；其它团队也可能用 4xx/5xx——讲清楚「我们选了业务码模型」即可。

---

## 5. 配置：`@ConfigurationProperties` 优先

散落 `@Value` 难维护、难校验。项目把一组相关配置绑到 Properties 类，例如：

- `AppConfigProperties`（`app.resume`）  
- `VoiceInterviewProperties`（`app.voice-interview`，含 TTS 预热开关）  
- `ResumeAnalysisProperties`、`LlmProviderProperties` 等  

```yaml
app:
  voice-interview:
    warmup-opening-audio-enabled: ${APP_VOICE_INTERVIEW_WARMUP_OPENING_AUDIO_ENABLED:false}
```

环境变量覆盖默认值：开发关预热、生产可打开——这是「配置外置」的实操例子（见 [11](11-voice-realtime.md)）。

---

## 6. JPA Repository：派生查询我在读什么

`InterviewSessionRepository` 继承 `JpaRepository`，大量方法靠 **方法名派生**，无需手写 SQL：

| 方法名片段 | 含义 |
|-----------|------|
| `findBySessionId` | WHERE session_id = ? |
| `findByResumeIdOrderByCreatedAtDesc` | 按简历 + 创建时间倒序 |
| `findFirstByResumeIdAndStatusInOrderByCreatedAtDesc` | 未完成会话（状态 IN） |
| `findTop10BySkillIdOrderByCreatedAtDesc` | 限 10 条历史 |

需要 JOIN FETCH / 投影时再用 `@Query`（JPQL）。列表性能与投影分析、`ddl-auto` 陷阱都在 [05 JPA 实操](05-jpa-persistence.md)。

---

## 7. 我的小验证

### 7.1 口述验收：「事务不含 LLM」

对照链路自检：

1. `createSession`：LLM 出题在 `questionService.generateQuestionsBySkill`，**该方法本身不是**「大事务包住 LLM + 多表写入」的反例写法；落库是随后的 `saveSession` 短事务。  
2. 交卷评估：`enqueueEvaluationTask` 只写 PENDING + 发 Stream；真正评估在 Consumer。  
3. Stream 发送失败：`TransactionalExecutor.runRequiresNew` 单独更新 FAILED。

### 7.2 代码验证：派生查询

在 `InterviewSessionRepository` 增加派生方法：

```java
boolean existsBySessionId(String sessionId);
```

Spring Data 会按方法名生成 `exists … by sessionId` 查询。本轮用 `:app:compileJava` 确认接口可被解析；完整 `@DataJpaTest` 切片在 Spring Boot 4 需单独核对 test starter，留待后续补（不阻塞 L1 概念验收）。

已有可对照的派生查询样例：`findBySessionId`、`findTop10BySkillIdOrderByCreatedAtDesc`、`findFirstByResumeIdAndStatusInOrderByCreatedAtDesc`。

---

## 8. 和后续 L / G 的接口

| 后续项 | 从 L1 接出去的点 |
|--------|------------------|
| L7 Redis Stream | 为何不用裸 `@Async`；Producer/Consumer 与短事务如何配合 |
| L8 限流 AOP | 横切与 `@Transactional` 一样靠代理；`@RateLimit` + Aspect |
| G1 Security | 统一异常里已有 401/403 码位，鉴权过滤器尚未落地 |
| G2 Flyway | 从 `ddl-auto=update` 迁到 `validate` + 版本化 SQL |

---

## 核心要点（可对外复述）

1. **三层**：Controller 薄、Service 编排、Repository 只碰数据；对外 `Result` + DTO，不暴露 Entity。  
2. **DI**：构造器注入 + `@RequiredArgsConstructor`；主类可 exclude 自动配置以接管 Bean 装配。  
3. **事务**：只包短 DB；LLM / HTTP / 发 Stream 在事务外；同类自调用用 `TransactionalExecutor`。  
4. **异常**：`BusinessException(ErrorCode)` → `GlobalExceptionHandler` → HTTP 200 + 业务码。  
5. **配置**：`@ConfigurationProperties` + yml/环境变量；敏感信息不进仓库。

→ 通用概念与高频题：本地 `SpringBoot.md`（与本文双轨：概念复述 ↔ 仓库证据）  
→ 环境 / ddl-auto / 列表投影：[01](01-env-setup.md) · [05](05-jpa-persistence.md)

---

## 9. 实读案例：日程模块（Repository + Service）

> 源码：`modules/interviewschedule/repository/InterviewScheduleRepository.java`、`service/InterviewScheduleService.java`  
> 补记：对照提问整理（把「看不懂的接口/服务」钉成可复述知识点）

### 9.1 Repository：只声明，不写实现

```java
@Repository
public interface InterviewScheduleRepository extends JpaRepository<InterviewScheduleEntity, Long> {
  List<InterviewScheduleEntity> findByStatusAndInterviewTimeBefore(...);
  List<InterviewScheduleEntity> findByStatus(...);
  List<InterviewScheduleEntity> findByInterviewTimeBetween(...);

  @Modifying
  @Query("UPDATE InterviewScheduleEntity e SET e.status = :newStatus WHERE ...")
  int updateStatusByStatusAndInterviewTimeBefore(...);
}
```

| 点 | 含义 |
|----|------|
| `JpaRepository<实体, 主键类型>` | 白送 `save` / `findById` / `findAll` / `deleteById` |
| 前三个方法 | **派生查询**：方法名 → WHERE（`And` / `Before` / `Between`） |
| `@Query` + `@Modifying` | 手写 **JPQL 批量 UPDATE**；返回 `int` = 影响行数 |
| 无实现类 | 运行时 Spring 生成代理（类似「只写 API client 接口」） |

派生命名对照：

| 方法名片段 | 语义 |
|-----------|------|
| `findByStatus` | `WHERE status = ?` |
| `...AndInterviewTimeBefore` | 且 `interviewTime < ?` |
| `...Between` | 时间落在 `[start, end]` |

批量更新典型用途：定时任务把「已过点仍是 PENDING」改成过期态（见 `ScheduleStatusUpdater`）。

### 9.2 Service：业务编排 + Entity↔DTO

`InterviewScheduleService` 注入唯一依赖 `InterviewScheduleRepository`，对外返回 **DTO**，不返回 Entity。

| 方法 | 做什么 |
|------|--------|
| `create` | request → Entity，`status=PENDING`，`save`，`toDTO` |
| `update` | 先 `getByIdOrThrow`；`copyProperties` 时 **忽略** `id`/`status` |
| `delete` | `deleteById`（当前未先校验存在） |
| `updateStatus` | 只改状态，与内容更新拆开 |
| `getAll` | 过滤优先级：有起止时间 → `Between`；否则有 status → `findByStatus`；否则 `findAll` |
| `getById` / `getByIdOrThrow` | `Optional.orElseThrow(BusinessException)` |
| `toDTO` | `BeanUtils.copyProperties(entity, dto)` |

链路：

```text
Controller → InterviewScheduleService → InterviewScheduleRepository → 表 interview_schedule
```

### 9.3 易混点（读代码时）

1. **`BeanUtils.copyProperties`** ≈ 浅拷贝同名字段；第三参是忽略字段列表。  
2. **`getAll` 同时传时间范围和 status 时只按时间查**，status 不生效。  
3. 类里的 `COPYABLE_FIELDS` 常量当前未被使用（可读作残留）。  
4. 与 §1 会话模块同一套分层：Controller 薄、Service 定规则、Repository 碰库。

### 9.4 `getAll` 与 `stream().map().toList()`（小白版）

```java
public List<InterviewScheduleDTO> getAll(String status, LocalDateTime start, LocalDateTime end) {
    List<InterviewScheduleEntity> entities;

    if (start != null && end != null) {
        entities = repository.findByInterviewTimeBetween(start, end);
    } else if (status != null) {
        entities = repository.findByStatus(InterviewStatus.valueOf(status));
    } else {
        entities = repository.findAll();
    }

    return entities.stream()
        .map(this::toDTO)
        .toList();
}
```

**上半段：先决定「查哪些行」**

| 条件 | 调用 | 得到什么 |
|------|------|----------|
| 有起止时间 | `findByInterviewTimeBetween` | 落在时间窗内的 Entity 列表 |
| 否则有 status | `findByStatus(valueOf(status))` | 该状态的 Entity 列表；`valueOf` 把字符串转成枚举 |
| 都没有 | `findAll()` | 全表 Entity 列表 |

变量 `entities` 类型是 `List<InterviewScheduleEntity>`——这是**数据库形状**，还不能直接当 API 返回值。

**下半段：为什么要 `.stream()`？**

目标：把「每一条 Entity」变成「一条 DTO」，得到 `List<InterviewScheduleDTO>`。

| 写法 | 在干什么 |
|------|----------|
| `entities.stream()` | 把列表变成 **Stream（流水线）**，方便对「每一个元素」依次做操作 |
| `.map(this::toDTO)` | 对每个 Entity 调用 `toDTO`；`this::toDTO` = 方法引用，等价于 `e -> this.toDTO(e)` |
| `.toList()` | 把流水线结果重新收成不可变 `List`（Java 16+） |

类比前端：

```js
entities.map(e => toDTO(e))  // JS
entities.stream().map(this::toDTO).toList()  // Java
```

**为什么不直接返回 Entity？**

- Entity 绑 JPA/表结构，可能带懒加载、内部字段；DTO 才是对外合同。  
- Service 层统一做 Entity → DTO，Controller 只拿 DTO 装进 `Result`。

**不用 stream 的等价写法（帮助理解）：**

```java
List<InterviewScheduleDTO> result = new ArrayList<>();
for (InterviewScheduleEntity e : entities) {
    result.add(toDTO(e));
}
return result;
```

`stream` 不是「必须」，而是把「遍历 + 转换 + 收集」写成一条管道，意图更直白；列表不大时性能差异可忽略，这里主要是**可读性与习惯写法**。

### 9.5 `List` / `ArrayList` / `Stream` 是什么关系？

| 概念 | 是什么 | 类比 |
|------|--------|------|
| **`List`** | **接口**（契约）：有序、可重复的集合，定义了 `add`/`get`/`size` 等，**本身不是具体实现** | TS 的 `Array<T>` 类型约定，或「数组能做什么」的说明书 |
| **`ArrayList`** | `List` 的一种**实现**：底层大致是可变数组，随机访问快 | 真正的 `[]` / 可变数组对象 |
| **`Stream`** | **不是新容器**，而是对数据源（List/数组等）的**一次性处理流水线**；用完就结束，一般不回头改原列表 | 更像「对数组做 `.map/.filter` 的计算过程」，不是另一种 List |

关系简图：

```text
List（接口）
  ↑ 实现
ArrayList / LinkedList / ...（具体类）

List.stream()  →  Stream（流水线）
                    ├─ map / filter / ...
                    └─ toList() / collect(...)  →  又得到一个 List
```

常见声明：

```java
List<String> list = new ArrayList<>();  // 变量类型用接口，右边用实现（面向接口编程）
```

方法返回 `List<...>` 时，运行时往往是 `ArrayList` 或 `toList()` 产生的不可变 List，调用方通常只依赖 `List` 接口即可。

**能不能直接在 `List` 上 `.map()`？**

- **标准 `List` 接口没有 `map` 方法**（到常用的 JDK 版本都如此）。
- 所以不能写 `entities.map(this::toDTO)`；要嘛 `for` 循环，要嘛 `entities.stream().map(...).toList()`。
- 这和 JS 不同：JS 的 `Array` 自带 `map`；Java 把「集合存储」和「函数式批量转换」拆开了——存储用 `List`，管道操作用 `Stream`。

补充：若已引入第三方库（如 Eclipse Collections）或自己写工具方法，可以封装出「List 上的 map」，但**语言自带 API 的路径就是 Stream**。项目里 `getAll` 用的是这种标准写法。

### 9.6 为什么返回 `List<...>` 而不是 `ArrayList<...>`？

`getAll` 签名是：

```java
public List<InterviewScheduleDTO> getAll(...)
```

而不是 `ArrayList<InterviewScheduleDTO>`。

| 返回 `List` | 返回 `ArrayList` |
|-------------|------------------|
| 只承诺「有序、可按索引访问的集合」 | 把实现细节钉死为数组列表 |
| 内部可换成 `toList()` 不可变列表、`LinkedList` 等，调用方不用改 | 换实现就要改所有调用方的类型 |
| 符合「依赖抽象、不依赖具体类」 | 过度暴露实现 |

调用方真正需要的是：**拿到一串 DTO，能遍历 / 取 size**——这些都在 `List` 接口上。  
至于运行时到底是 `ArrayList` 还是 `stream().toList()` 产生的不可变 List，Service 内部的事，调用方（Controller）不该关心。

类比 TypeScript：

```ts
function getAll(): InterviewScheduleDTO[]  // 承诺「数组形状」
// 而不是承诺一定是某个特殊 MutableArray 子类
```

经验法则：**变量/参数/返回值尽量写接口（`List`/`Map`）；`new` 的时候再写具体类（`new ArrayList<>()`）。**

### 9.7 为什么「只有 `new` 时才写具体类」？

```java
List<String> names = new ArrayList<>();  // ✅ 左边接口，右边实现
// ArrayList<String> names = new ArrayList<>();  // 能跑，但把实现暴露到变量类型上
```

| 好处 | 说明 |
|------|------|
| **换实现不动调用代码** | 后面改成 `new LinkedList<>()` 或工厂返回别的 `List`，只要左边仍是 `List`，用 `add`/`get` 的代码通常不用改 |
| **降低耦合** | 方法参数写成 `void print(List<String> list)`，调用方传 `ArrayList`/`toList()` 结果都行；若参数写成 `ArrayList`，别的 `List` 实现就传不进去 |
| **表达意图** | 「我需要列表能力」≠「我必须用数组列表」；具体结构是局部实现细节 |
| **与框架一致** | JPA/`stream().toList()` 返回的往往也是「某种 `List`」，你的代码用接口接最省事 |

什么时候变量也可以写具体类？

- 你**明确要用**只有该实现才有的方法（例如某些 `ArrayList` 特有 API，实际很少需要）
- 或极热路径里为避免接口虚调用做微观优化（业务代码几乎不必纠结）

默认仍推荐：**对外（参数/返回值）用接口；对内 `new` 选一个具体实现。**

### 9.8 为什么 `new` 时「只能」写具体类？

你已经理解：变量写成 `List` 是为了依赖抽象。那右边为什么必须是 `new ArrayList<>()`？

因为 **`new` 的含义是：在堆上真正造出一个对象**。造对象必须知道：

- 字段怎么布局（`ArrayList` 内部有数组、size 等）
- 每个方法具体怎么执行（`add` 往数组里放）

而 **`List` 只是接口**：只规定「要有 `add`/`get`…」，**没有方法体，也没有自己的字段**。它像一张说明书，不是一台机器。

```java
List<String> names = new List<>();       // ❌ 编译错误：List 是 abstract / interface，不能实例化
List<String> names = new ArrayList<>();  // ✅ 造的是 ArrayList 这台「真机器」，用 List 类型的「遥控器」拿着
```

可以记成：

| 位置 | 写什么 | 为什么 |
|------|--------|--------|
| 左边（类型） | 接口 `List` | 我只要求「列表能力」 |
| 右边（`new`） | 具体类 `ArrayList` | 我必须选一个**能落地的实现**来真正创建对象 |

所以不是「风格上建议 `new` 写具体类」，而是 **语言规则：接口/抽象类不能 `new`（除非匿名实现，那是另一回事）**；具体类才有完整实现，才能 `new`。

选 `ArrayList` 而不是 `LinkedList`，只是「这次用数组列表这种实现」的选择；左边仍写 `List`，换实现时只改右边的 `new`。

### 9.9 Java 接口 vs 类：区别与联系

| | **接口（interface）** | **类（class）** |
|--|----------------------|----------------|
| 本质 | **能力约定**（能做什么） | **具体实现**（怎么做、数据存在哪） |
| 方法 | 默认只有方法签名；Java 8+ 可有 `default`/`static` 方法 | 通常有完整方法体 |
| 字段 | 一般是 `public static final` 常量 | 可有普通实例字段（对象自己的状态） |
| `new` | **不能**直接 `new` 接口 | 非抽象类可以 `new` |
| 继承关系 | 一个类可 `implements` **多个**接口 | 类只可 `extends` **一个**父类（单继承） |
| 典型例子 | `List`、`Repository` 接口 | `ArrayList`、`InterviewScheduleService` |

**联系（最重要）：**

```text
接口  ←—— implements ——  类
 List                      ArrayList
（约定）                   （实现约定的全部方法）
```

1. **类实现接口**：`class ArrayList implements List` → 必须实现接口要求的方法，于是「是一个 List」。  
2. **变量可用接口类型指向类实例**：`List x = new ArrayList()` → 多态：同一接口，不同实现可替换。  
3. **接口不能代替类去存状态造对象**；类负责落地。接口负责解耦与统一口径。

和 TypeScript 的粗略对照（帮助记忆，不完全等同）：

| Java | 有点像 TS |
|------|-----------|
| `interface List` | `interface` / 类型约定 |
| `class ArrayList implements List` | `class Foo implements Bar` |
| 不能 `new List()` | 不能 `new` 纯 interface（除非有实现类） |

Spring 里你会反复看到同一种模式：**接口定契约，类（或框架生成的代理）做实现**——例如你写 `InterviewScheduleRepository extends JpaRepository`，运行时由 Spring 提供实现类。

### 9.10 实读：`InterviewScheduleEntity` 上的注解

> 源码：`modules/interviewschedule/model/InterviewScheduleEntity.java`  
> 文件目的：用一个 Java 类描述数据库表 `interview_schedule` 的一行，供 JPA 做 ORM（对象 ↔ 表行）。

#### `@Entity` vs `@Data`（先分清两套体系）

| | `@Entity` | `@Data` |
|--|-----------|---------|
| 来自哪 | **JPA / Jakarta Persistence**（`jakarta.persistence`） | **Lombok**（编译期代码生成） |
| 管什么 | 「这个类是一张表的映射」——持久化 / ORM | 「帮我生成 getter/setter 等样板代码」——少写手写方法 |
| 和数据库 | **有关** | **无关**（纯 Java 语法糖） |
| 能不能互相替代 | 不能 | 不能 |

没有 `@Entity`，JPA 不会把它当表映射。  
没有 `@Data`，你得手写大量 `getXxx`/`setXxx`；有没有 `@Data` 都不影响「是否映射表」。

#### 这个文件整体在干什么

```text
表 interview_schedule 的一行  ←JPA→  InterviewScheduleEntity 对象
Repository.save(entity) / find...  读写的就是这类对象
Service 再转成 DTO 给前端
```

#### 每个注解做什么

**类上**

| 注解 | 功能 |
|------|------|
| `@Entity` | 声明这是 JPA 实体，默认对应一张表 |
| `@Table(name = "interview_schedule")` | 指定表名（否则常按类名推默认名） |
| `@Data` | Lombok：生成 getter/setter、`toString`、`equals`/`hashCode`、必要构造相关方法，避免手写样板 |

**字段上**

| 注解 | 功能 |
|------|------|
| `@Id` | 主键 |
| `@GeneratedValue(strategy = IDENTITY)` | 主键由数据库自增生成（如 PostgreSQL serial/identity） |
| `@Column(name = "...")` | 字段 ↔ 列名；Java 用驼峰，库表常用下划线 |
| `@Column(nullable = false)` | 列非空（与建表/校验相关） |
| `@Column(columnDefinition = "TEXT")` | 提示列类型为 TEXT（长文本） |
| `@Column(updatable = false)` | 更新实体时**不改**该列（如 `created_at` 只在插入时写） |
| `@Enumerated(EnumType.STRING)` | 枚举存库用**字符串名**（`PENDING`），而不是序号 0/1/2（更稳、可读） |

未加 `@Column` 的字段（如 `interviewer`、`position`）：仍会映射，列名由命名策略推导（常为 `interviewer`、`position`）。

**生命周期回调（方法上）**

| 注解 | 功能 |
|------|------|
| `@PrePersist` | **第一次插入前**调用 → 这里给 `createdAt`/`updatedAt` 赋 `now()` |
| `@PreUpdate` | **每次更新前**调用 → 刷新 `updatedAt` |

#### 和前后端的对应感觉

- `@Entity` + `@Table` + `@Column` ≈ 「这个 model 绑哪张表、哪一列」  
- `@Data` ≈ 「别手写一堆 getter/setter」  
- `@PrePersist` / `@PreUpdate` ≈ 「存盘前后自动填时间戳」

### 9.11 Entity 能自动生成吗？和 MyBatis / MyBatis-Plus 有何不同？

本项目用的是 **Spring Data JPA**（`InterviewScheduleEntity` + `JpaRepository`），不是 MyBatis。

#### 这类 Entity 要手写吗？

**两种方向都可以，不是只能手写：**

| 方向 | 做法 | 常见工具 |
|------|------|----------|
| **库表 → 代码**（反向工程） | 已有表，生成 Entity | IntelliJ：**JPA Buddy**；Database 工具里 Generate Persistence Mapping；旧版 Hibernate reverse engineering |
| **代码 → 库表**（正向） | 先写 Entity，再让库对齐 | 开发期 `ddl-auto=update`；或 Flyway 手写/生成 SQL（生产更常见） |
| **手写** | 表简单或字段要精细控制时 | 本仓库大量 Entity 即此风格 |

IntelliJ 没有「装一个插件就替代理解」的银弹：生成的是样板，**字段含义、枚举、`updatable`、关联关系仍要人改对**。小表手写往往更快。

#### 和 MyBatis、MyBatis-Plus 的差别（怎么访问数据库）

| | **JPA / Spring Data JPA**（本项目） | **MyBatis** | **MyBatis-Plus** |
|--|--------------------------------------|-------------|------------------|
| 核心思路 | **对象优先**：Entity ↔ 表；CRUD 少写 SQL | **SQL 优先**：Mapper 接口 + XML/注解里写 SQL | 在 MyBatis 上加强：通用 CRUD、条件构造器、代码生成 |
| 典型文件 | `@Entity` 类 + `XxxRepository` 接口 | `XxxMapper.java` + `XxxMapper.xml`（或注解 SQL）+ 普通 POJO | Mapper 继承 `BaseMapper<T>`，常配代码生成器出 Entity/Mapper/Service |
| 简单查询 | 派生方法名 / JPQL | 自己写 `SELECT ...` | `lambdaQuery().eq(...)` 等，少写 XML |
| 复杂 SQL | 能写，但不如 MyBatis 直观 | **强项**：动态 SQL、精细优化 | 同样强，复杂处仍可回退 XML |
| 生成代码 | JPA Buddy / 反向工程 Entity | **MyBatis Generator** | **官方 Generator**（很常用） |

一句话：

- **JPA**：表结构围着对象转，日常 CRUD 省事；本项目的 `Entity` + `Repository` 就是这条路。  
- **MyBatis**：你掌握每一句 SQL；POJO 可以像 Entity 一样手写或生成，但**没有** JPA 那种 `@Entity` 托管。  
- **MyBatis-Plus**：MyBatis 的「加速包」，生成器和通用 CRUD 很省事，国内业务项目很常见。

选型不是「谁绝对更好」，而是：**更想少写 SQL、用对象模型 → JPA；更想 SQL 完全可控 → MyBatis(Plus)。** 同一项目一般选定一种主方案，避免两套混用加重心智负担。

### 9.12 `orElseThrow` 是谁的方法？为什么能写在 `repository.findById` 后面？

```java
return repository.findById(id)
    .orElseThrow(() -> new BusinessException(...));
```

容易误读成「在 repository 上调用 `orElseThrow`」。实际是 **链式调用**，分两步：

```text
repository.findById(id)     →  返回 Optional<InterviewScheduleEntity>
       .orElseThrow(...)    →  在这个 Optional 上调用，不是 Repository 的方法
```

| 点 | 说明 |
|----|------|
| `orElseThrow` 定义在哪 | JDK 的 **`java.util.Optional`**（Java 8+），不是 Spring、也不是 Repository |
| 为什么能接着写 | `JpaRepository.findById` 的返回类型就是 `Optional<T>`，所以可以点出 Optional 的方法 |
| 做什么 | Optional **有值** → 取出 Entity 返回；**空** → 执行 lambda，抛出你给的异常 |

等价理解：

```java
Optional<InterviewScheduleEntity> opt = repository.findById(id);
if (opt.isPresent()) {
    return opt.get();
} else {
    throw new BusinessException(...);
}
```

`orElseThrow` 把「找不到就抛业务异常」写成一行，避免到处 `null` 判断。  
相关：`Optional.of` / `empty` / `map` / `orElse(默认值)` 都是同一套 API。

### 9.13 什么时候用 `Optional`？`save()` 又是谁的方法？

#### `Optional` 用在什么场景？

`Optional<T>` = 「这个结果**可能有、也可能没有**」的盒子，用来替代含糊的 `null`。

| 典型场景 | 例子（本项目） |
|----------|----------------|
| **按 id 查询可能查不到** | `repository.findById(id)` → `Optional`，再用 `orElseThrow` / `orElse` |
| **方法约定：找不到不返回 null** | 调用方必须处理「空」的情况，少漏判 NPE |
| **链式转换** | `optional.map(...).orElse(...)` |

**不太适合**：当作字段到处存、或所有返回值都包一层（会啰嗦）。更常见是：**查找类 API 的返回类型**。

对比：

```java
// 旧习惯：可能返回 null，调用方容易忘判
InterviewScheduleEntity e = ...; // null?

// Optional：返回类型就表明「可能空」
Optional<InterviewScheduleEntity> e = repository.findById(id);
```

#### `save()` 在哪定义？做什么？

```java
repository.save(entity);  // InterviewScheduleService 的 create / update / updateStatus
```

| 点 | 说明 |
|----|------|
| 定义位置 | Spring Data 的 **`CrudRepository` / `JpaRepository`** 接口（本仓库 `InterviewScheduleRepository extends JpaRepository<..., Long>`，因此继承了 `save`） |
| 谁实现 | 运行时由 Spring Data JPA **生成代理实现**，不是你手写的类 |
| 功能 | **插入或更新**实体：一般无主键 / 视为新 → INSERT；已有主键且已存在 → UPDATE（以 JPA 持久化上下文判断为准） |
| 返回值 | 保存后的实体（可能带上数据库生成的 `id`、时间戳等） |

和 `Optional` 的分工：

```text
findById  → Optional（读：可能没有）
save      → Entity（写：保存后的对象，通常总有返回值）
```

所以：`orElseThrow` 配「查」；`save` 配「增/改」。二者都来自「Repository 继承的 Spring Data API」，不是业务 Service 自己发明的方法。

### 9.14 Hibernate 是什么？

**Hibernate** = 一个流行的 **ORM 框架**（Object-Relational Mapping）：在 Java 对象和关系型数据库表之间自动转换。

本项目里你接触到的链路大致是：

```text
你写的代码
  InterviewScheduleEntity（@Entity）
  InterviewScheduleRepository（Spring Data JPA）
        ↓
  Spring Data JPA（Repository 派生查询、save/findById…）
        ↓
  JPA 标准 API（jakarta.persistence：@Entity、@Column…）
        ↓
  Hibernate（默认实现：生成 SQL、管理实体状态、ddl-auto 等）
        ↓
  PostgreSQL
```

| 名字 | 角色 |
|------|------|
| **JPA** | **规范/接口**（怎么标注实体、EntityManager 等约定） |
| **Hibernate** | **实现**（真正拼 SQL、读写库；Spring Boot 默认带上的就是它） |
| **Spring Data JPA** | 在 JPA 之上再封装：让你用 `JpaRepository` + 方法名，少写样板 |

类比：JPA 像 USB 规范，Hibernate 像某品牌 U 盘实现；Spring Data JPA 像更顺手的文件管理器。

和你已见概念的关系：

- `@Entity` / `@Column` → JPA 注解，由 **Hibernate** 解读并映射到表  
- `ddl-auto`（见 [05](05-jpa-persistence.md)）→ 多是 **Hibernate** 在启动时根据实体改 schema  
- `repository.save` / `findById` → Spring Data 调用，底层仍落到 Hibernate 发 SQL  

一句话：**你写 Entity + Repository；Hibernate 在底下负责「对象 ↔ SQL ↔ 表」。** 日常可以说「我们用 JPA」，技术栈里实现通常就是 Hibernate。

### 9.15 Servlet 是什么？（对照 `application.yml`）

**Servlet** = Java Web 里处理 **HTTP 请求/响应** 的经典组件模型（规范在 Jakarta Servlet）。一个 Servlet 大致做：收到请求 → 处理 → 写出响应。

本项目你几乎**不直接写** `HttpServlet`，而是写 `@RestController`。关系是：

```text
浏览器 / 前端
    → HTTP
内嵌 Tomcat（Servlet 容器）
    → 把请求交给 DispatcherServlet（Spring MVC 的核心 Servlet）
    → 路由到你的 @RestController 方法
```

| 概念 | 角色 |
|------|------|
| **Servlet 规范** | 约定「容器如何调你的 Web 组件」 |
| **Servlet 容器** | 实现规范的服务器，Boot 默认 **Tomcat**（打在 jar 里） |
| **`DispatcherServlet`** | Spring MVC 的前门 Servlet，再分发给 Controller |
| **你写的 Controller** | 业务接口；底下仍走 Servlet 那一套请求响应 |

所以：`server.port: 8082` 是 Tomcat 监听端口；配置里的 `servlet:` 是在调 **Servlet 容器 / Spring 对 Servlet 的封装选项**。

本仓库 `application.yml` 里两处：

```yaml
server:
  port: 8082
  servlet:
    encoding:          # HTTP 请求/响应字符编码（UTF-8）
      charset: UTF-8
      enabled: true
      force: true

spring:
  servlet:
    multipart:         # 基于 Servlet 的文件上传（multipart/form-data）
      enabled: true
      max-file-size: 50MB
      max-request-size: 50MB
```

| 配置 | 含义 |
|------|------|
| `server.servlet.encoding` | 避免中文乱码：请求/响应强制 UTF-8 |
| `spring.servlet.multipart` | 允许上传文件，并限制单文件/整请求大小（简历、知识库上传会用到） |

和前端的感觉：Servlet 容器 ≈ Node 里听端口的 HTTP server；`multipart` ≈ 限制 body 上传大小；你写的 Controller ≈ 路由处理器，只是 Java 生态里传统入口叫 Servlet。

### 9.16 `DispatcherServlet` 做什么？原理是什么？

**`DispatcherServlet`** = Spring MVC 的**前端控制器（Front Controller）**：几乎所有进到 Spring Web 的 HTTP 请求，先经过它，再由它**分发**到具体的 `@RestController` / `@Controller` 方法。

本项目你写的是：

```java
@RestController
@RequestMapping("/api/interview-schedule")
public class InterviewScheduleController { ... }
```

请求实际路径更像：

```text
HTTP 请求
  → Tomcat（Servlet 容器）
  → DispatcherServlet（唯一大门）
  → HandlerMapping：根据 URL/方法找到 Controller 方法
  → HandlerAdapter：调用该方法（绑定 @PathVariable、@RequestBody 等）
  → 方法返回 Result / 对象
  → HttpMessageConverter：写成 JSON 响应
  → 回到客户端
```

#### 它具体干什么（职责）

| 步骤 | 做什么 |
|------|--------|
| 接收请求 | 作为 Servlet 被容器调用（`service`/`doDispatch`） |
| 找处理器 | `HandlerMapping`：哪一个 Controller 的哪个方法匹配 |
| 调处理器 | `HandlerAdapter`：真正 `invoke` 你的方法 |
| 处理异常 | 可走到 `@RestControllerAdvice`（如本项目的 `GlobalExceptionHandler`） |
| 写回响应 | 视图解析（传统 MVC）或 **消息转换**（REST JSON） |

你**不用手写** `DispatcherServlet`：Spring Boot 自动注册；加了 `spring-boot-starter-webmvc` 就会有。

#### 原理（简化版）

1. **单一入口**：避免每个 URL 对应一个原生 Servlet；统一横切（编码、异常、拦截器）。  
2. **策略可插拔**：映射、适配、转换都是接口，Boot 提供默认实现。  
3. **和拦截器的关系**：`HandlerInterceptor` 在「进 Controller 前后」由 DispatcherServlet 串起来调用（类似前置/后置钩子）。  
4. **和 Filter 的关系**：`Filter` 在 Servlet 更外层（容器级）；请求先 Filter，再进 DispatcherServlet，再到 Controller。

```text
Filter（Servlet 容器）→ DispatcherServlet → Interceptor → Controller 方法
```

类比前端：DispatcherServlet ≈ 总路由（如框架的 router），Controller 方法 ≈ 各个 route handler；它负责「匹配到谁、怎么调、怎么把返回值变成 HTTP 响应」，而不是写业务本身。
