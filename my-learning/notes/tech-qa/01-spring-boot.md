# tech-qa 01 · Spring Boot 高频问答

> 对应学习项：LEARNING_PLAN **L1**  
> 落点：`common/`（`result`、`exception`、`config`、`aspect`、`transaction`）与各模块 Controller/Service。  
> 对照本地通用笔记：`TechSkills/.../spring-framework/SpringBoot.md`（概念系统展开）；本文只钉**本仓库证据**。  
> 模块样本：[modules/05 日程](../modules/05-interviewschedule.md)（最小三层闭环）；事务/JPA 细节见 [tech-qa/02](02-jpa-transaction.md)。

---

### Spring Boot 相比 Spring 解决了什么？

**答：** 三件事——起步依赖（starter 统一版本与传递依赖）、自动配置（按 classpath 和条件注解装配默认 Bean）、内嵌容器（打成 jar 直接跑，不用外部 Tomcat）。本质是「约定优于配置」。

**本仓库：** `gradle/libs.versions.toml` 集中版本，`app/build.gradle` 引 starter；`App.java` 一个 `main` 起内嵌 Tomcat（`server.port` 默认 8080）。

---

### IoC 和 DI 有什么区别？为什么项目里一律构造器注入？

**答：** IoC 是思想——对象生命周期交给容器，不自己 `new`；DI 是实现方式——容器把依赖塞进来。构造器注入的好处：依赖可声明为 `final`、缺依赖时启动即失败、单元测试可直接 `new`、循环依赖更早暴露。

**本仓库：** 统一 `@RequiredArgsConstructor` + `private final`，例如：

```java
@Service
@RequiredArgsConstructor
public class InterviewPersistenceService {
    private final InterviewSessionRepository sessionRepository;
    private final InterviewAnswerRepository answerRepository;
}
```

字段注入（`@Autowired` 打在字段上）在业务代码里没有使用。

---

### `@SpringBootApplication` 包含什么？为什么本项目要 `exclude` 一批自动配置？

**答：** 约等于 `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan`。`exclude` 的场景是「我要自己接管这类 Bean」——自动配置给的默认实现不符合需求时，留着反而会冲突或产生多余 Bean。

**本仓库：** `App.java` 排除了 Spring AI 的若干 OpenAI 自动配置，因为多 Provider 由 `LlmProviderRegistry` 自建并缓存客户端，不吃「单 Provider 自动装配」那套。同时开了 `@EnableScheduling`。

---

### 三层各自的边界在哪？Controller 里能不能写业务？

**答：** Controller 只做路由、参数接收/校验、包装响应；Service 做业务编排和事务边界；Repository 只碰持久化。Controller 写业务的直接代价是：无法复用、事务位置错乱、测试必须走 HTTP。

**本仓库：** `AGENTS.md` 写成硬规则。列表请求一眼看清分层：

```java
// InterviewController
@GetMapping("/api/interview/sessions")
public Result<List<SessionListItemDTO>> listSessions() {
    List<SessionListItemDTO> items = persistenceService.findAll().stream()
        .map(SessionListItemDTO::from)
        .toList();
    return Result.success(items);
}
```

| 层 | 本例落点 |
|----|----------|
| Controller | `InterviewController`：包装 `Result`，不做编排 |
| Service | `InterviewPersistenceService.findAll()` |
| Repository | `InterviewSessionRepository.findAllByOrderByCreatedAtDesc()` |

对外只返回 DTO，**不返回 Entity**。更完整的编排见 `InterviewSessionService.createSession`：先调 LLM 出题、写 Redis，再 `saveSession` 落库——出题与落库是分开的步骤。

日程模块是更小的闭环样本：`InterviewScheduleController` → `InterviewScheduleService` → `InterviewScheduleRepository`（见下方实读附录，或 [modules/05](../modules/05-interviewschedule.md)）。

---

### 为什么统一响应用 HTTP 200 + 业务码？有什么代价？

**答：** 好处是前端只解析一套信封（`code`/`message`/`data`）；代价是丢掉了 HTTP 语义（网关、监控、重试策略都看不出成功失败），也不符合 REST 习惯。团队内一致最重要。

**本仓库：** `Result<T>` + `GlobalExceptionHandler` 上 `@ResponseStatus(HttpStatus.OK)`；`ErrorCode` 按模块分段（简历 2xxx、会话 3xxx、AI 7xxx、限流 8001…）。

---

### 全局异常处理怎么落地？为什么不用 `RuntimeException`？

**答：** `@RestControllerAdvice` 集中兜底，按异常类型映射错误码与文案。业务失败要用带错误码的自定义异常，否则前端只能拿到一句无结构的 message。

**本仓库：**

```java
throw new BusinessException(ErrorCode.INTERVIEW_SESSION_NOT_FOUND);
throw new BusinessException(ErrorCode.INTERVIEW_QUESTION_NOT_FOUND, "无效的问题索引: " + index);
```

处理器覆盖：`BusinessException`、校验异常、AI 网络/鉴权/上游 429（7xxx）、上传超限、限流（8001）与兜底 `Exception`。

---

### `@ConfigurationProperties` 和 `@Value` 怎么选？

**答：** 一组相关配置用 `@ConfigurationProperties` 绑到类型安全对象；`@Value` 适合极少数一次性单值，散落多处后很难知道「这个功能到底受哪些配置影响」。

**本仓库：** `StorageConfigProperties`、`VoiceInterviewProperties`、`LlmProviderProperties`、`StructuredOutputProperties`、`AppConfigProperties`。规则是「Service 里不出现 `@Value`」。例：`app.voice-interview.opening-audio-warmup-enabled` 用环境变量覆盖，开发关预热、生产可开（见 [modules/03](../modules/03-voiceinterview.md)）。

---

### `@PostConstruct` 在项目里承担了什么？有什么风险？

**答：** Bean 初始化后做一次性准备。风险是拖慢启动、外部依赖不可用时启动即报错、和自动配置顺序耦合。

**本仓库：** `FileStorageService.init()` 检查/创建 S3 桶；`InterviewSkillService.loadPresetSkills()` 扫技能包；语音开场白 TTS 预热（默认关：`opening-audio-warmup-enabled`）。

---

### 一个请求从 Tomcat 到 Controller 经过什么？Filter / Interceptor / Aspect 有何区别？

**答：**

```text
Filter（Servlet 容器层）→ DispatcherServlet → HandlerInterceptor → AOP 切面 → Controller 方法
```

- **Filter**：Servlet 规范级，能改 request/response 本身（编码、CORS、压缩），不认识 Controller。
- **Interceptor**：Spring MVC 级，知道即将调用哪个 handler，适合鉴权、日志。
- **Aspect**：Spring Bean 方法级，最贴近业务语义（限流、事务、缓存）。

**本仓库：** `DispatcherServlet` 由 Boot 自动注册；`server.servlet.encoding` 强制 UTF-8；限流是 Aspect（`RateLimitAspect`）而不是 Filter，因为它需要方法上的注解元数据。

---

### Servlet / DispatcherServlet 是什么？和 `@RestController` 什么关系？

**答：** Servlet 是 Java Web 处理 HTTP 的经典组件模型；Servlet 容器（Boot 默认 Tomcat）负责监听端口、把请求交给 Servlet。`DispatcherServlet` 是 Spring MVC 的前门控制器：几乎所有请求先经它，再由 `HandlerMapping` 找到 Controller 方法、`HandlerAdapter` 调用、`HttpMessageConverter` 写成 JSON。

```text
浏览器 → Tomcat → DispatcherServlet → @RestController 方法 → Result JSON
```

你几乎不直接写 `HttpServlet`，加了 `spring-boot-starter-webmvc` 就会自动注册 `DispatcherServlet`。

**本仓库配置：**

| 配置 | 含义 |
|------|------|
| `server.servlet.encoding` | 请求/响应强制 UTF-8，避免中文乱码 |
| `spring.servlet.multipart` | 允许上传，限制单文件/整请求 50MB（简历、知识库） |

---

### AOP 的实现原理是什么？为什么「同类内部调用」注解会失效？

**答：** Spring AOP 基于代理。调用方拿到的是代理对象；同类内部 `this.method()` 走原始对象，不经代理，于是 `@Transactional`、`@RateLimit`、`@Cacheable` 全部失效。

**本仓库：** `TransactionalExecutor` 把需要独立事务的片段交给另一个 Bean：

```java
@Service
public class TransactionalExecutor {
    @Transactional(rollbackFor = Exception.class)
    public void run(Runnable action) { action.run(); }

    @Transactional(rollbackFor = Exception.class, propagation = Propagation.REQUIRES_NEW)
    public void runRequiresNew(Runnable action) { action.run(); }
}
```

Stream 生产者失败写 FAILED 就靠 `runRequiresNew`。事务边界细则见 [tech-qa/02](02-jpa-transaction.md)。

---

### 事务为什么不能包住 LLM / HTTP / 发 Stream？

**答：** `@Transactional` 会占用连接池连接直到方法结束。LLM 常 10～60s：池耗尽、外部调用不可回滚、未提交数据就被消费者读到。

**本仓库正确拆法：**

```text
① 短事务：写 PENDING / 落本地状态
② 事务外：发 Redis Stream / 调 LLM
③ 失败时：REQUIRES_NEW 短事务写 FAILED
```

证据：`InterviewSessionService.enqueueEvaluationTask` 先写 PENDING 再 `sendEvaluateTask`；真正评估在 Consumer。

---

### 定时任务要注意什么？

**答：** 默认调度线程池只有一个线程；任务必须幂等；多实例部署会并发跑同一逻辑，需要分布式锁或选主。

**本仓库：** `@EnableScheduling` 在 `App.java`；`ScheduleStatusUpdater` 小时级把过期 `PENDING` 日程批量置 `CANCELLED`，用一条 `@Modifying` JPQL 批量 UPDATE。⚠️ 项目未覆盖：多实例下的调度去重。

---

### 请求体为什么优先用 `record`？为什么不直接返回 Entity？

**答：** `record` 不可变、字段即契约，配合 Bean Validation 就够表达入参。Entity 绑 JPA/表结构，可能带懒加载与内部字段；DTO 才是对外合同。Service 统一做 Entity → DTO，Controller 只装 `Result`。

**本仓库：** `CreateProviderRequest`、`QueryRequest` 等大量是 `record`；日程模块用 `BeanUtils.copyProperties(entity, dto)`（浅拷贝同名字段；字段改名后静默不生效，字段多时不如 MapStruct）。

---

### `List` / `ArrayList` / `Stream` 是什么关系？为什么不能 `list.map(...)`？

**答：**

| 概念 | 是什么 |
|------|--------|
| `List` | **接口**：有序集合契约，不能 `new List<>()` |
| `ArrayList` | `List` 的一种**实现**，才能 `new` |
| `Stream` | **不是容器**，是一次性处理流水线；`map`/`filter` 后要 `toList()` 收成新 List |

标准 `List` **没有** `map` 方法（和 JS 不同），所以要写 `entities.stream().map(this::toDTO).toList()`。经验法则：变量/参数/返回值写接口（`List`），`new` 时写具体类（`new ArrayList<>()`）。

**本仓库：** `InterviewScheduleService.getAll`、`InterviewController.listSessions` 都是这种写法。

---

### `Optional` 和 `orElseThrow` / `save` 分别干什么？

**答：** `Optional<T>` = 「可能有、也可能没有」的盒子，用来替代含糊的 `null`。`findById` 返回 `Optional`，所以能链式写：

```java
return repository.findById(id)
    .orElseThrow(() -> new BusinessException(...));
```

`orElseThrow` 是 **`java.util.Optional`** 的方法，不是 Repository 的。`save` 定义在 `JpaRepository` / `CrudRepository`，由 Spring Data 生成代理实现，做插入或更新，返回保存后的实体。

---

### Entity 上的注解各自干什么？JPA / Hibernate / Spring Data JPA 什么关系？

**答：** 类上 `@Entity` + `@Table` 声明表映射；`@Data` 是 Lombok（生成 getter/setter），和数据库无关、不能互相替代。字段上常见：`@Id`、`@GeneratedValue(IDENTITY)`、`@Column`、`@Enumerated(STRING)`、`@PrePersist` / `@PreUpdate`。

关系链：

```text
你写 Entity + Repository
  → Spring Data JPA（派生查询、save/findById）
  → JPA 规范（jakarta.persistence）
  → Hibernate（默认实现：拼 SQL、ddl-auto）
  → PostgreSQL
```

**本仓库：** 日程实体 `InterviewScheduleEntity` 是最小样本；`ddl-auto`、派生查询、投影见 [tech-qa/02](02-jpa-transaction.md)。

---

### Java 21 的虚拟线程在这个项目里有用吗？

**答：** 适合阻塞型 IO 密集任务；不会让 CPU 密集变快，也不改变连接池上限。

**本仓库：** `application.yml` 启用了虚拟线程。LLM/S3 即使跑在虚拟线程上，**也不能放进数据库事务**——限制来自连接池，不是线程模型。

---

## 我的口述清单

1. 三层 + `Result<T>` + `BusinessException` + `GlobalExceptionHandler` 是一套完整对外契约。
2. 构造器注入 + `@ConfigurationProperties` 是「可测、可配」的地基。
3. 代理是横切能力的共同原理，也是同类自调用失效的共同原因；事务只包短 DB。
4. `DispatcherServlet` 是 Spring MVC 大门；Filter → DispatcherServlet → Interceptor → Aspect → Controller。
5. `@PostConstruct` 里塞外部调用要给开关，否则本地开发被云端服务拖累。

---

## 实读附录：日程模块（三层最小闭环）

> 源码：`modules/interviewschedule/`。模块视角见 [modules/05](../modules/05-interviewschedule.md)；这里只钉「读代码时怎么对应到上面问答」。

### Repository：只声明，不写实现

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
| `JpaRepository<实体, 主键>` | 白送 `save` / `findById` / `findAll` / `deleteById` |
| 前三个方法 | **派生查询**：方法名 → WHERE（`And` / `Before` / `Between`） |
| `@Query` + `@Modifying` | JPQL 批量 UPDATE；返回 `int` = 影响行数 |
| 无实现类 | 运行时 Spring 生成代理 |

### Service：编排 + Entity↔DTO

| 方法 | 做什么 |
|------|--------|
| `create` | request → Entity，`status=PENDING`，`save`，`toDTO` |
| `update` | 先 `getByIdOrThrow`；`copyProperties` 时忽略 `id`/`status` |
| `updateStatus` | 只改状态，与内容更新拆开 |
| `getAll` | 有起止时间 → `Between`；否则有 status → `findByStatus`；否则 `findAll`，再 `stream().map(toDTO).toList()` |
| `getByIdOrThrow` | `Optional.orElseThrow(BusinessException)` |

```text
Controller → InterviewScheduleService → InterviewScheduleRepository → 表 interview_schedule
```

### 读代码时的易混点

1. **`getAll` 同时传时间范围和 status 时只按时间查**，status 被忽略。
2. **`delete` 不校验存在**，直接 `deleteById`。
3. 类里的 `COPYABLE_FIELDS` 常量当前未被使用（残留）。
4. 与会话模块同一套分层：Controller 薄、Service 定规则、Repository 碰库。

### 小验证（口述）

1. `createSession`：LLM 出题不在「大事务包住 LLM + 多表写入」里；落库是随后的短事务。
2. 交卷评估：`enqueueEvaluationTask` 只写 PENDING + 发 Stream；真正评估在 Consumer。
3. Stream 发送失败：`TransactionalExecutor.runRequiresNew` 单独更新 FAILED。
