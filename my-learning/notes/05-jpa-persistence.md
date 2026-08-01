# 05 · JPA 持久化实操：ddl-auto 与查询投影

> 对应配置：`app/src/main/resources/application.yml` → `spring.jpa.hibernate.ddl-auto`  
> 对应源码：`InterviewSessionRepository`、`InterviewPersistenceService`、`InterviewController`、`SessionListItemDTO`  
> 关系定位：[04 Spring Boot 三层地基](04-spring-boot-foundations.md) 讲分层与派生查询，本篇专讲**落到数据库时最容易踩的两个坑**——schema 被重建导致丢数据、列表查询把大字段全读出来。

本篇两个主题：

| 主题 | 现象 | 结论 |
|------|------|------|
| §1 `ddl-auto` | 重启后业务数据消失 | `create` 每次删表重建；开发用 `update`，生产用迁移工具 |
| §2 列表查询投影 | 列表接口偏慢 | `findAll()` 会读 TEXT 大字段；列表应按 DTO 字段投影 |

---

## 1. `ddl-auto` 与数据丢失

### 1.1 现象

本地 `bootRun` 重启后，前端记录页为空，`GET /api/interview/sessions` 返回空列表。PostgreSQL 容器与数据卷仍在，并非 Docker `down -v` 清库。

### 1.2 根因

`spring.jpa.hibernate.ddl-auto` 曾设为 **`create`**：

- 每次应用启动时，Hibernate **先删表再建表**；
- `interview_sessions` 及关联答案等业务数据随之清空；
- Docker 卷还在，但表内容已被 schema 重建擦掉。

配置文件中的注释也写明了意图：

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: update #首次启动用 create，表创建成功后，改回 update
```

首次建表可用 `create`，建好后必须改回 `update`（或更严格的模式），否则每次重启都会丢数据。

### 1.3 `ddl-auto` 取值对照

| 值 | 行为 | 适用场景 |
|----|------|----------|
| `create` | 启动时删表再建 | 仅一次性初始化 / 可丢数据的实验 |
| `create-drop` | 启动建表，**关闭应用时再删表** | 集成测试 |
| `update` | 按实体增量改表结构，**尽量保留数据** | 本地开发（当前推荐） |
| `validate` | 只校验实体与库表是否一致，不改库 | 接近生产的环境 |
| `none` | 完全不碰 schema | 生产（配合 Flyway/Liquibase） |

注意：`update` 不会做复杂迁移（如删列、改类型），大改表结构仍需手工 SQL 或迁移工具。

真正执行「按实体改表」的是 **Hibernate**（JPA 的默认实现），不是 Spring Data 本身——这条链路见 [04 §9.14](04-spring-boot-foundations.md)。

### 1.4 修复与现状

1. 将 `ddl-auto` 设为 `update`（本仓库当前已是 `update`）。
2. 生产环境：使用 Flyway / Liquibase 管理 schema，并将 `ddl-auto` 设为 `validate` 或 `none`（见 `AGENTS.md`：生产不能依赖自动建表）。

表结构现状见 [03 库表设计](03-db-schema-design.md)。

### 1.5 验证

```powershell
# 1. 确认配置
Select-String -Path app\src\main\resources\application.yml -Pattern 'ddl-auto'

# 2. 新建一条会话记录后重启后端
# 3. 再请求列表，记录应仍在
curl http://localhost:8082/api/interview/sessions
```

在 PostgreSQL 中也可核对：

```powershell
docker exec -it interview-postgres psql -U postgres -d interview_guide -c "SELECT session_id, created_at FROM interview_sessions ORDER BY created_at DESC LIMIT 5;"
```

---

## 2. 列表查询投影（JPQL → DTO）

> 状态：🟡 **已分析根因与改法，代码尚未落地**——当前仍是 `findAll()` 加载完整 Entity，再 `SessionListItemDTO.from(entity)`。

### 2.1 名词：JPQL 与 DTO

| 缩写 | 全称 | 一句话 |
|------|------|--------|
| **JPQL** | **J**ava **P**ersistence **Q**uery **L**anguage | JPA 的查询语言，写在实体/属性层面（如 `FROM InterviewSessionEntity s`），由 Hibernate 翻译成 SQL |
| **DTO** | **D**ata **T**ransfer **O**bject | 只承载传输所需字段的数据对象；列表用轻量 DTO，避免把整张表/Entity 原样返回前端 |

**投影（projection）**：查询时只选出需要的列，并直接构造成 DTO（如 `SELECT new ...SessionListItemDTO(...)`），而不是先查出完整 Entity 再在内存里裁剪。

### 2.2 现象与根因

记录页加载慢：列表 API 响应时间偏长，数据量不大时也不理想。

列表页 DTO（`SessionListItemDTO`）只用轻量字段：`sessionId`、`skillId`、`status`、`overallScore`、`createdAt` 等。

旧路径却是：

```text
Controller → persistenceService.findAll()
          → sessionRepository.findAllByOrderByCreatedAtDesc()  // 整行 Entity
          → SessionListItemDTO.from(entity)
```

`InterviewSessionEntity` 含多个 **TEXT** 大字段（`questionsJson`、`overallFeedback`、`strengthsJson`、`improvementsJson`、`referenceAnswersJson` 等）。`findAll()` 会把这些列全部读入内存，再丢弃不用——I/O 与反序列化开销浪费在列表场景上。

### 2.3 优化做法

**① JPQL 构造器投影**——在 `InterviewSessionRepository` 中只 SELECT 列表需要的列，直接构造 DTO：

```java
@Query("""
    SELECT new interview.guide.modules.interview.model.SessionListItemDTO(
        s.sessionId, s.skillId, s.difficulty, s.resumeId,
        COALESCE(s.totalQuestions, 0), s.status, s.evaluateStatus,
        s.evaluateError, s.overallScore, s.createdAt, s.completedAt
    )
    FROM InterviewSessionEntity s
    ORDER BY s.createdAt DESC
    """)
List<SessionListItemDTO> findAllListItems();
```

**② Service / Controller 接线**

- `InterviewPersistenceService.findAllListItems()`：`@Transactional(readOnly = true)`，委托投影查询。
- `InterviewController.listSessions()`：改为 `return Result.success(persistenceService.findAllListItems())`，不再 `findAll().stream().map(...)`。

**③ 排序索引**

实体表索引包含 `created_at`（及 resume / skill 组合索引），支撑 `ORDER BY created_at DESC`，避免大表全表排序拖累。

### 2.4 验证

```powershell
Measure-Command { Invoke-RestMethod http://localhost:8082/api/interview/sessions }
```

对比优化前后耗时；必要时在 PostgreSQL 用 `EXPLAIN ANALYZE` 确认未扫描 TEXT 大列。

---

## 核心要点

- **重启丢业务数据，优先查 `ddl-auto`，不要先怪 Docker 卷。**
- `create` / `create-drop` 适合「可丢库」场景；日常开发用 `update`；生产用迁移工具 + `validate`/`none`。
- **列表 API 的 SELECT 应由 DTO 字段驱动**，不要先加载完整 Entity 再裁剪。
- JPQL `SELECT new ...DTO(...)` 是 Spring Data JPA 常用的轻量投影方式。
- 大字段（TEXT/JSON）适合详情接口按需加载；列表与详情查询路径应分开。

→ 相关踩坑摘要见 [code-changes/00-env-setup](../code-changes/00-env-setup/README.md)
