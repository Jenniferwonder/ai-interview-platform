# tech-qa 02 · JPA 与事务 高频问答

> 落点：各模块 `model/*Entity`、`repository/*`、`*PersistenceService`、`common/transaction/TransactionalExecutor`、`application.yml` 的 `spring.jpa.*`。

---

### JPA、Hibernate、Spring Data JPA 是什么关系？

**答：** JPA 是**规范**（`jakarta.persistence` 的注解与 API）；Hibernate 是最常用的**实现**（真正拼 SQL、管理实体状态、执行 `ddl-auto`）；Spring Data JPA 在其上再封一层，用 `JpaRepository` + 方法名派生省掉样板代码。类比：JPA 是 USB 规范，Hibernate 是某品牌 U 盘，Spring Data JPA 是更顺手的文件管理器。

**本仓库：** 写的是 `@Entity` + `XxxRepository extends JpaRepository`，底层是 Hibernate 发 SQL 到 PostgreSQL。

---

### `ddl-auto` 各取值什么行为？为什么它能删掉业务数据？

**答：**

| 值 | 行为 | 场景 |
|----|------|------|
| `create` | 启动时**删表重建** | 一次性初始化 |
| `create-drop` | 启动建、停机删 | 集成测试 |
| `update` | 增量改结构，尽量保留数据 | 本地开发 |
| `validate` | 只校验不改库 | 准生产 |
| `none` | 完全不碰 schema | 生产 + 迁移工具 |

`create` 每次启动都重建表，Docker 卷还在但表内容已被清空——所以「重启后数据没了」应该先查 `ddl-auto`，不要先怪存储。

**本仓库：** 我真踩过这个坑（当时配的是 `create`，重启后表被清空），现在开发环境是 `update`；环境侧的排查过程记在 [01 本地启动](../01-env-setup.md)，生产应改 `validate` + Flyway（学习计划 G2）。

---

### 派生查询的命名规则是什么？什么时候该换成 `@Query`？

**答：** 方法名按 `find/exists/count + By + 属性 + 条件关键字(And/Or/Between/Before/In/OrderBy/Top/First)` 解析成查询。适合条件固定且不多的场景；一旦出现动态条件、JOIN FETCH、只取部分列、批量写，就该用 `@Query`（JPQL）或 Specification。

**本仓库：** `findBySessionId`、`findFirstByResumeIdAndStatusInOrderByCreatedAtDesc`、`findTop10BySkillIdOrderByCreatedAtDesc`、`existsBySessionId`；批量更新用 `@Modifying @Query`（日程模块）。

---

### `@Transactional` 的生效原理是什么？哪些情况会失效？

**答：** 基于代理：调用方经过代理才会开事务。常见失效场景：

1. **同类内部调用** `this.save()`（没走代理）；
2. 方法不是 `public`（代理拦不到）；
3. 抛的是**受检异常**且没配 `rollbackFor`（默认只回滚 `RuntimeException`）；
4. 异常被自己 `catch` 掉了（事务不知道失败）；
5. 类不是 Spring Bean。

**本仓库：** 写操作统一带 `rollbackFor = Exception.class`；同类自调用问题用 `TransactionalExecutor.run` / `runRequiresNew` 绕开。

---

### 为什么事务里绝对不能调 LLM / S3 / 外部 HTTP？

**答：** 事务期间数据库连接被独占。LLM 动辄 10–60 秒，会造成：连接池被长时间占满 → 其它请求拿不到连接；外部调用**不可回滚**，超时后 DB 回滚语义混乱；如果事务里先入队再提交，消费者可能读到还没提交的数据。

**正确拆法（本项目模式）：**

```text
① 短事务：写 PENDING 状态
② 事务外：发 Redis Stream / 调 LLM
③ 失败时：REQUIRES_NEW 短事务写 FAILED
```

**本仓库：** `InterviewSessionService.enqueueEvaluationTask`、`EvaluateStreamProducer.onSendFailed` 就是这三步的实现。

---

### `REQUIRES_NEW` 用在什么地方？

**答：** 需要「外层成功与否都要落下这条记录」时——典型是失败标记、审计日志。它会挂起外层事务、开新连接执行，用多了会翻倍占用连接，所以只在短小写操作上使用。

**本仓库：** Stream 入队失败后把任务状态打成 `FAILED`，必须独立于外层事务提交，否则外层回滚会把「失败标记」一起回滚掉，任务永远停在「处理中」。

---

### 列表接口为什么不该用 `findAll()`？有哪些投影方式？

**答：** `findAll()` 会 SELECT 全部列，包含列表根本不用的 TEXT/JSON 大字段，浪费 IO 与反序列化。投影方式：

| 方式 | 写法 | 适用 |
|------|------|------|
| 构造器投影 | `SELECT new com.x.SessionListItemDTO(...)` | 列表页最常用 |
| 接口投影 | 定义只含 getter 的接口 | 简单只读视图 |
| `@EntityGraph` | 控制关联加载 | 需要 Entity 但要避免 N+1 |

**本仓库：** `SessionListItemDTO` 只需 `sessionId`/`status`/`overallScore`/`createdAt` 等轻量字段，但 `InterviewSessionEntity` 有 5 个 TEXT 大字段；`listSessions` 目前仍是 `findAll().map(...)`，投影 `findAllListItems()` 属于**已分析未落地**的改动。

---

### N+1 是什么？项目里怎么规避？

**答：** 先查 N 条主记录，再为每条触发一次关联查询。规避：`JOIN FETCH`、`@EntityGraph`、批量查询后在内存拼装，或者干脆只查需要的列（投影）。反面写法是「循环里 `findById`」。

**本仓库：** `InterviewSessionEntity` 除了 `@ManyToOne` 的 `resume`，还额外映射了一个**只读** `resume_id` 列（`insertable=false, updatable=false`），用于只需要 id 时避免触发 LAZY 加载。`AGENTS.md` 也明确禁止循环调 DB。

---

### 幂等写入怎么保证？

**答：** 靠数据库唯一约束 + upsert 语义，而不是「先查再插」（并发下仍会重复）。

**本仓库：** `interview_answers` 上 `(session_id, question_index)` 唯一约束，同一题重复提交只会更新那一行；`resumes.file_hash`、`knowledge_bases.file_hash` 唯一索引承担文件级去重；`voice_interview_evaluations.session_id` 唯一保证一场会话一份报告。

---

### 枚举字段存字符串还是序号？

**答：** 存字符串（`@Enumerated(EnumType.STRING)`）。序号（`ORDINAL`）在枚举顺序调整或中间插入新值后，历史数据语义会整体错位，且库里看不出含义。代价是占用略多、改名要迁移。

**本仓库：** 所有状态枚举都是 `STRING`：`AsyncTaskStatus`、`VectorStatus`、`InterviewStatus`、`SessionStatus`、`InterviewPhase` 等。

---

### 为什么大量结构化数据存成 JSON TEXT，而不是拆子表？

**答：** 取舍。JSON TEXT 的好处是模型演进自由、不用为每个数组建表、读写一次搞定；代价是**数据库层无法查询/统计/校验**，全靠应用序列化，字段改名后老数据要兼容。适合「整体读写、不需要按内部字段检索」的数据。

**本仓库：** `questionsJson`、`strengthsJson`、`suggestionsJson`、`referenceAnswersJson`、`questionEvaluationsJson` 都是这种；反过来，`interview_answers` 需要按题号查询和更新，就老老实实建了表。

---

### 批量更新怎么写？`@Modifying` 有什么注意点？

**答：** 用一条 JPQL/SQL 更新多行，而不是查出来循环 `save`。注意：`@Modifying` 会绕过持久化上下文，可能让内存中的实体变成脏数据（必要时 `clearAutomatically`），且方法必须在事务内。

**本仓库：** `InterviewScheduleRepository.updateStatusByStatusAndInterviewTimeBefore(...)` 返回 `int` 影响行数，由小时级定时任务调用。

---

### 生产环境该怎么管 schema？

**答：** 版本化迁移工具（Flyway/Liquibase）+ `ddl-auto: validate` 或 `none`。让「表结构变更」和「代码变更」一样进版本控制、可审查、可回滚；`update` 做不了删列、改类型、数据回填这类复杂迁移。

**本仓库：** ⚠️ 项目未覆盖——当前开发用 `update`，Flyway 是我要补的 G2。表结构现状见 [03 库表设计](../03-db-schema-design.md)。

---

### 有没有绕过 JPA 直接写 SQL 的地方？

**答：** 有，而且是合理的：向量表由 Spring AI 管理，不是业务 Entity。

**本仓库：** `knowledgebase/repository/VectorRepository` 用 `JdbcTemplate` 操作 `vector_store`。里面有个值得记的细节——PostgreSQL 的 JSON 存在性操作符 `?` 会和 JDBC 占位符冲突，所以改用 `metadata->>'kb_id' IS NOT NULL` 这种写法，另用 `jsonb_set` 改写元数据。

---

## 我的口述清单

1. `ddl-auto` 是开发期便利、生产期风险；丢数据先查它。
2. 事务只包短 DB 操作；外部调用一律在事务外，失败用 `REQUIRES_NEW` 补记。
3. 列表查询由 DTO 字段驱动 SELECT，不要先加载完整 Entity 再裁剪。
4. 幂等交给唯一约束，别靠「先查再写」。
