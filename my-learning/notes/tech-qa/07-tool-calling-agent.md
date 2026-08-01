# tech-qa 07 · Tool-Calling 与 Agent 高频问答

> 落点：`common/ai/AgentUtilsConfiguration`（`interviewSkillsToolCallback`）、`common/ai/LlmProviderRegistry`（三种客户端变体）、`app/src/main/resources/skills/`、`modules/interview/skill/InterviewSkillService`。

---

### 什么是 Function Calling / Tool Calling？一次调用怎么走完？

**答：** 让模型不直接回答，而是「请求调用某个函数」，由应用执行后把结果回喂，模型再据此作答。流程：

```text
① 应用把工具清单（名称/描述/参数 schema）随请求发给模型
② 模型返回 tool_call：要调哪个工具、参数是什么
③ 应用本地执行（查库、读文件、调 API）
④ 结果作为一条 tool 消息回传
⑤ 模型基于结果生成最终回答（可能继续请求下一次调用）
```

关键认知：**模型只负责「决定调什么」，执行与权限完全在应用侧**。

**本仓库：** 由 Spring AI 的 `ToolCallingAdvisor` 负责这套循环，业务代码只需在构造 `ChatClient` 时挂上工具。

---

### 这个项目有哪些工具？

**答：** 目前只有一个：`interviewSkillsToolCallback`（来自 spring-ai-agent-utils 的 `SkillsTool`），作用是**让模型按需加载技能包的 `SKILL.md` 正文**（persona 与指令），而不是一次性把所有技能文档塞进上下文。

```java
@Bean("interviewSkillsToolCallback")
public ToolCallback interviewSkillsToolCallback() {
    Resource skillsRootResource = resourceLoader.getResource(normalizedSkillsRoot);
    return SkillsTool.builder().addSkillsResource(skillsRootResource).build();
}
```

配置键：`app.ai.agent-utils.skills-root`（默认 `classpath:skills`）。仓库里**没有** `@Tool` 注解的自定义方法。

---

### 什么时候**不该**给模型工具？

**答：** 当任务是「一次性产出固定结构」时。挂了工具会带来三个副作用：模型可能绕道去调工具、输出多一层不确定性、token 与延迟上升。结构化产出应该走最短路径。

**本仓库：** 这条有直接证据——出题走 `getPlainChatClient()`（不挂工具、只留内容防护），并且 `interview-question-skill-system.st` 里明确写了「不要调用工具」。而语音对话走 `getVoiceChatClient()`（挂工具 + 开对话历史），因为它需要按阶段/话题按需取资料。

---

### 技能包（skills）目录是怎么组织的？

**答：**

```text
resources/skills/
  java-backend/
    SKILL.md          ← front matter(name, description) + 正文 persona
    skill.meta.yml    ← displayName + categories[{key,label,priority,ref,shared}]
    references/*.md   ← 该技能独有的参考资料
  frontend/ ...
  system-design/ ...
  _shared/references/*.md   ← 跨技能共享（java.md、mysql.md、redis.md…）
```

加载：`InterviewSkillService.loadPresetSkills()` 在 `@PostConstruct` 扫 `classpath:skills/*/SKILL.md`（跳过 `_shared`），解析 front matter 与 meta，建 `presetRegistry` 与 `categoryRefIndex`。参考文件解析顺序是：共享目录 → `skills/{id}/references/{file}` → `skills/{id}/{file}`。

---

### 「按需 tool 加载」和「批量注入资料」怎么权衡？

**答：** 两种上下文策略：

| 策略 | 优点 | 代价 |
|------|------|------|
| 工具按需加载 | 上下文省、可扩展到大量资料 | 多轮往返、延迟高、模型可能不去调 |
| 批量预注入 | 一次成型、结果稳定 | 上下文占用大、资料多了放不下 |

**本仓库同时存在两条路**：语音对话用工具按需取（`SkillsTool`）；出题用 `buildReferenceSection` 把命中分类的参考 markdown **预先拼进 Prompt**。选择依据就是上一条——要稳定 JSON 就预注入。

---

### 什么是 ReAct？Agentic Loop 长什么样？

**答：** ReAct = Reasoning + Acting，让模型交替「思考 → 行动（调工具）→ 观察结果」直到能回答。运行时循环大致是：

```text
while 未完成 且 未超预算:
    模型基于当前上下文决定：直接回答 / 调用工具
    若调用工具 → 执行 → 把结果追加进上下文
终止条件：模型给出最终答案 / 达到最大步数 / 超时 / 超 token 预算
```

**本仓库：** 只有「单层工具调用」这种最轻量的 Agent 形态，⚠️ 没有多步规划、没有自主任务分解。说清楚边界比夸大更重要。

---

### 工具怎么设计才好用？

**答：** 五条：

1. **描述写给模型看**：说清「什么时候用」，而不只是「这个函数做什么」；
2. **参数尽量少且强类型**，枚举优于自由文本；
3. **幂等 + 无副作用优先**；有副作用的工具（发消息、写库）要显式确认或加白名单；
4. **返回结构化且精简**：把无关字段裁掉，返回内容会占上下文；
5. **失败要可读**：把错误变成模型能理解的提示（「参数缺失，请提供 xxx」），而不是抛栈。

---

### 怎么防止 Agent 死循环或跑飞？

**答：** 三类护栏：**步数/时间/token 上限**、**重复检测**（同一工具同一参数连续调用就中断）、**权限最小化**（工具只能访问必要资源，写操作单独审批）。再加上可观测：把每步的工具名、参数、耗时记下来，否则出问题无从复盘。

**本仓库：** 靠单层工具调用天然规避死循环；⚠️ 项目未覆盖：工具调用的埋点与步数护栏。

---

### Agent、RAG、纯对话怎么选？

**答：**

| 形态 | 适用 | 代价 |
|------|------|------|
| 纯对话 | 通用知识、写作 | 无外部事实保障 |
| RAG | 有明确知识库、要引用 | 索引与检索工程 |
| Agent（工具） | 需要**动作**（查库、算数、调接口）或多步流程 | 延迟、不确定性、护栏成本 |

判断问题：**「答案在文档里」还是「需要去做点什么」**？前者 RAG，后者 Agent。很多需求两者结合：先检索，再用工具核对。

---

### 工具调用也会「幻觉」吗？

**答：** 会，表现为：调用不存在的工具、编造参数、无视返回结果继续编。缓解：工具 schema 严格校验（参数不合法直接打回让模型重试）、返回结果显式要求引用、必要字段缺失时不允许「猜」、以及把执行结果作为唯一事实来源写进 Prompt 规则。

---

### MCP 在这里扮演什么角色？

**答：** MCP（Model Context Protocol）把「工具/资源/提示」标准化成协议，客户端与服务端解耦——同一个工具服务可以被不同 AI 客户端复用，不必为每个框架各写一遍适配。适合工具需要跨应用共享或由第三方提供的场景。

**本仓库：** ⚠️ 项目未覆盖，工具是进程内 Bean。如果以后要把技能包/知识库能力开放给外部客户端使用，MCP 是自然的下一步。

---

## 我的口述清单

1. 工具调用是「模型决定、应用执行」，权限与安全全在应用侧。
2. 要稳定 JSON 就别挂工具——这条我在出题链路上做了实际取舍。
3. 上下文策略二选一：工具按需加载省 token，批量预注入更稳定。
4. Agent 护栏三件套：步数/时间/token 上限、重复检测、最小权限。
5. 我这套只到「单层工具调用」，多步规划与工具埋点是明确缺口。
