# 专题二：Agent 工作流深度解析

> **学习目标**：掌握 AI Agent 工作流的设计原理与实现，能够设计复杂的多步骤任务执行流程。
> **前置知识**：了解责任链模式、策略模式、异步编程
> **预计用时**：60 分钟

---

## 一、真实业务场景

### 场景 1：内容生产流水线

**业务需求**：
一个自媒体平台需要自动化生产文章：
1. 分析用户需求，确定文章主题
2. 搜索相关资料
3. 撰写文章内容
4. 审核文章质量
5. 发布到各大平台

**对应工作流**：
```
Analyzer（分析需求）
    ↓
Performer（执行任务：搜索、撰写）
    ↓
Supervisor（监督评估：检查质量）
    ↓ 未通过 → 回到 Analyzer
    ↓ 通过
Summarizer（总结并发布）
```

### 场景 2：故障排查助手

**业务需求**：
运维系统遇到故障时需要：
1. 检查故障现象
2. 规划排查步骤
3. 执行排查操作（支持重试）
4. 生成故障报告

**对应工作流**：
```
Inspector（检查故障）
    ↓
Planner（规划排查步骤）
    ↓
Runner（执行步骤，失败重试）
    ↓
Replier（生成报告）
```

### 场景 3：数据分析 Agent

**业务需求**：
1. 理解用户的数据分析需求
2. 生成 SQL 查询语句
3. 执行查询并获取结果
4. 生成可视化图表
5. 输出分析结论

**对应工作流**：Loop 策略（需要多轮迭代优化）

---

## 二、工作流系统架构

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         用户层                                        │
├─────────────────────────────────────────────────────────────────────┤
│  前端界面 │ API 调用 │ 其他系统                                      │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓ HTTP POST /api/v1/ai/work/execute
┌─────────────────────────────────────────────────────────────────────┐
│                         接口层                                        │
├─────────────────────────────────────────────────────────────────────┤
│  AiController.execute()                                             │
│  - 接收执行请求 (sessionId, agentId, userInput)                     │
│  - 创建 SseEmitter 用于实时推送                                     │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│                         分发层                                        │
├─────────────────────────────────────────────────────────────────────┤
│  DispatchService.dispatchExecuteStrategy()                          │
│  - 根据 agentId 获取 Agent 配置                                     │
│  - 选择对应的执行策略 (Loop/Step)                                   │
│  - 提交到线程池异步执行                                             │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│                         执行策略层                                    │
├─────────────────────────────────────────────────────────────────────┤
│  ExecuteStrategy (策略接口)                                         │
│  ├─ ExecuteLoopStrategy  (循环策略)                                │
│  │   └─ Analyzer → Performer → Supervisor → Summarizer             │
│  └─ ExecuteStepStrategy  (步骤策略)                                │
│      └─ Inspector → Planner → Runner → Replier                     │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│                         节点层                                        │
├─────────────────────────────────────────────────────────────────────┤
│  AbstractExecuteNode (节点基类)                                     │
│  ├─ ExecuteAnalyzerNode    (分析节点)                              │
│  ├─ ExecutePerformerNode   (执行节点)                              │
│  ├─ ExecuteSupervisorNode  (监督节点)                              │
│  ├─ ExecuteSummarizerNode  (总结节点)                              │
│  ├─ ExecuteInspectorNode   (检查节点)                              │
│  ├─ ExecutePlannerNode     (规划节点)                              │
│  ├─ ExecuteRunnerNode      (运行节点)                              │
│  └─ ExecuteReplierNode     (回复节点)                              │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│                         推送层                                        │
├─────────────────────────────────────────────────────────────────────┤
│  SseEmitter 实时推送执行进度                                         │
│  - performer: Performer 节点输出                                    │
│  - supervisor: Supervisor 节点输出                                  │
│  - complete: 执行完成                                               │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 核心概念

| 概念 | 定义 | 举例 |
|-----|------|------|
| **Agent** | 完整的工作流代理，对应一个完整的任务执行流程 | 文章生产 Agent、故障排查 Agent |
| **Flow** | Agent 的执行流程配置，包含节点顺序和提示词 | Analyzer → Performer → Supervisor |
| **Client** | 每个节点使用的 LLM 配置 | analyzer_client、performer_client |
| **Node** | 工作流中的单个执行步骤 | Analyzer 节点负责分析需求 |
| **Context** | 节点间传递的上下文数据 | userInput、analyzerResponse、performerResponse |
| **Round** | Loop 策略的执行轮次 | 第 1 轮、第 2 轮... |

---

## 三、Loop 策略详解

### 3.1 策略设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Loop 策略流程图                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│    ┌──────────────┐                                                 │
│    │  用户输入     │                                                 │
│    └──────┬───────┘                                                 │
│           ↓                                                         │
│    ┌──────────────┐                                                 │
│    │   Analyzer   │  ← 分析需求，制定计划                           │
│    │  (分析节点)   │                                                 │
│    └──────┬───────┘                                                 │
│           ↓                                                         │
│    ┌──────────────┐                                                 │
│    │  Performer   │  ← 执行任务，可调用 MCP 工具                     │
│    │  (执行节点)   │                                                 │
│    └──────┬───────┘                                                 │
│           ↓                                                         │
│    ┌──────────────┐                                                 │
│    │  Supervisor  │  ← 评估执行结果                                 │
│    │  (监督节点)   │                                                 │
│    └──────┬───────┘                                                 │
│           │                                                         │
│           ├── 通过 ──→ ┌──────────────┐                            │
│           │             │  Summarizer  │  ← 总结结果                │
│           │             │  (总结节点)   │                            │
│           │             └──────┬───────┘                            │
│           │                    ↓                                    │
│           │              ┌──────────┐                               │
│           │              │  完成    │                               │
│           │              └──────────┘                               │
│           │                                                        │
│           └── 未通过 ──→ 回到 Analyzer（下一轮）                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 节点职责

| 节点 | 职责 | 输入 | 输出 | MCP 支持 |
|-----|------|------|------|----------|
| **Analyzer** | 分析用户需求，制定执行计划 | 用户输入 | 分析结果（JSON 格式） | 否 |
| **Performer** | 执行具体任务，调用工具 | 分析结果 | 执行结果 | 是 |
| **Supervisor** | 评估执行结果，决定是否继续 | 执行结果 | 评估结论（通过/不通过） | 否 |
| **Summarizer** | 总结最终结果 | 所有节点输出 | 最终回复 | 否 |

### 3.3 Prompt 设计示例

#### Analyzer Prompt

```
你是一名专业的任务分析专家。

【用户需求】
{0}

【你的任务】
1. 理解用户的核心需求
2. 将需求拆解为可执行的步骤
3. 确定需要调用的工具（如有）

【输出格式】（JSON）
{
  "understanding": "对需求的理解",
  "steps": ["步骤1", "步骤2", "步骤3"],
  "tools": ["tool_name1", "tool_name2"],
  "expected_output": "期望的输出"
}
```

#### Performer Prompt

```
你是一名专业的任务执行专家。

【用户需求】
{0}

【分析结果】
{1}

【你的任务】
根据分析结果执行具体任务。
- 如果需要搜索信息，使用 webSearch 工具
- 如果需要查询数据，使用相应的查询工具
- 将执行结果详细记录

【可用工具】
- webSearch: 联网搜索
- checkWeather: 查询天气
- ...

【输出格式】（JSON）
{
  "status": "success/failure",
  "result": "执行结果详情",
  "data": {...}
}
```

#### Supervisor Prompt

```
你是一名专业的质量监督专家。

【用户需求】
{0}

【分析结果】
{1}

【执行结果】
{2}

【你的任务】
评估执行结果是否满足用户需求。

【评估标准】
1. 结果是否完整
2. 结果是否准确
3. 结果是否满足用户期望

【输出格式】（JSON）
{
  "passed": true/false,
  "reason": "通过/不通过的原因",
  "suggestions": "改进建议（如不通过）"
}
```

#### Summarizer Prompt

```
你是一名专业的结果总结专家。

【用户需求】
{0}

【完整执行过程】
- 分析结果: {1}
- 执行结果: {2}
- 评估结果: {3}

【你的任务】
将以上执行过程总结为清晰、友好的回复给用户。

【输出格式】
自然的语言回复，不需要 JSON 格式。
```

### 3.4 执行上下文传递

```java
// ExecuteContext 数据结构
public class ExecuteContext {
    private String sessionId;        // 会话 ID
    private String userInput;        // 用户输入
    private int round;               // 当前轮次
    private Map<String, Object> nodeOutputs;  // 各节点输出

    // 节点输出示例
    nodeOutputs.put("analyzer", analyzerResponse);
    nodeOutputs.put("performer", performerResponse);
    nodeOutputs.put("supervisor", supervisorResponse);
}

// 节点间数据传递
String analyzerResponse = context.getValue("analyzer");
String performerPrompt = String.format(performerTemplate,
    context.getUserMessage(),
    analyzerResponse
);
```

### 3.5 SSE 实时推送

```java
// 推送 Performer 节点输出
ExecuteResponseEntity response = ExecuteResponseEntity.createPerformerResponse(
    context.getRound(),
    performerResponse
);

sseEmitter.send(SseEmitter.event()
    .name("performer")  // 事件名称
    .data(response));   // 事件数据

// 前端监听
eventSource.addEventListener('performer', (event) => {
    const data = JSON.parse(event.data);
    console.log('Performer 输出:', data.content);
});
```

---

## 四、Step 策略详解

### 4.1 策略设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Step 策略流程图                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│    ┌──────────────┐                                                 │
│    │  用户输入     │                                                 │
│    └──────┬───────┘                                                 │
│           ↓                                                         │
│    ┌──────────────┐                                                 │
│    │  Inspector   │  ← 检查任务可行性                               │
│    │  (检查节点)   │                                                 │
│    └──────┬───────┘                                                 │
│           ↓                                                         │
│    ┌──────────────┐                                                 │
│    │   Planner    │  ← 规划执行步骤                                 │
│    │  (规划节点)   │                                                 │
│    └──────┬───────┘                                                 │
│           ↓                                                         │
│    ┌──────────────┐                                                 │
│    │   Runner     │  ← 执行步骤（失败重试）                         │
│    │  (运行节点)   │  └─ maxRetry: 3                                │
│    └──────┬───────┘                                                 │
│           ↓                                                         │
│    ┌──────────────┐                                                 │
│    │   Replier    │  ← 整理结果并回复                               │
│    │  (回复节点)   │                                                 │
│    └──────┬───────┘                                                 │
│           ↓                                                         │
│      ┌──────────┐                                                   │
│      │  完成    │                                                   │
│      └──────────┘                                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 节点职责

| 节点 | 职责 | 特殊功能 |
|-----|------|---------|
| **Inspector** | 检查任务是否可执行 | 可提前终止不可行任务 |
| **Planner** | 规划详细的执行步骤 | 生成步骤列表 |
| **Runner** | 执行步骤 | **支持重试**（maxRetry） |
| **Replier** | 整理结果并回复用户 | 格式化输出 |

### 4.3 Runner 重试机制

```java
@Override
protected String doApply(ExecuteRequestEntity request, ExecuteContext context) {
    int maxRetry = 3;
    int retryCount = 0;
    String result = null;

    while (retryCount < maxRetry) {
        try {
            // 执行步骤
            result = executeStep(context);
            break;  // 成功则退出
        } catch (Exception e) {
            retryCount++;
            if (retryCount >= maxRetry) {
                throw new RuntimeException("执行失败，已重试 " + maxRetry + " 次");
            }
            // 记录重试日志
            log.warn("执行失败，第 {} 次重试", retryCount);
        }
    }

    return "replier";
}
```

### 4.4 Prompt 设计示例

#### Inspector Prompt

```
你是一名专业的任务检查专家。

【用户需求】
{0}

【你的任务】
检查该任务是否可以执行。

【检查项】
1. 需求是否清晰明确
2. 是否具备执行条件
3. 是否存在明显的障碍

【输出格式】（JSON）
{
  "executable": true/false,
  "reason": "可执行/不可执行的原因",
  "missing_info": ["缺失的信息1", "缺失的信息2"]
}
```

#### Planner Prompt

```
你是一名专业的任务规划专家。

【用户需求】
{0}

【检查结果】
{1}

【你的任务】
将任务拆解为详细的执行步骤。

【输出格式】（JSON）
{
  "steps": [
    {"order": 1, "action": "步骤1", "tool": "tool_name"},
    {"order": 2, "action": "步骤2", "tool": null},
    {"order": 3, "action": "步骤3", "tool": "tool_name"}
  ],
  "estimated_time": "预计耗时"
}
```

#### Runner Prompt

```
你是一名专业的任务执行专家。

【规划步骤】
{0}

【你的任务】
按顺序执行每个步骤。

【注意】
- 严格按照步骤顺序执行
- 如果步骤指定了工具，必须使用该工具
- 如果执行失败，详细记录错误信息

【输出格式】（JSON）
{
  "status": "success/failure",
  "completed_steps": [1, 2],
  "failed_step": 3,
  "error": "错误信息",
  "result": "执行结果"
}
```

---

## 五、策略对比与选择

### 5.1 对比表

| 维度 | Loop 策略 | Step 策略 |
|-----|----------|----------|
| **执行模式** | 循环迭代 | 线性步骤 |
| **适用场景** | 需要多轮优化、监督评估 | 需要严格按步骤执行 |
| **灵活性** | 高（可回到之前节点） | 中（固定流程） |
| **可控性** | 中（轮次不确定） | 高（步骤确定） |
| **重试支持** | 无 | 有（Runner 节点） |
| **典型用例** | 内容创作、数据分析 | 数据导入、批量处理 |

### 5.2 选择决策树

```
任务是否需要多轮迭代优化？
    ├── 是 → Loop 策略
    └── 否
        ↓
    任务是否需要严格按步骤执行？
        ├── 是 → Step 策略
        └── 否 → 都可以，看复杂度
            ├── 简单 → Step 策略（更可控）
            └── 复杂 → Loop 策略（更灵活）
```

---

## 六、节点实现模板

### 6.1 节点基类

```java
public abstract class AbstractExecuteNode {

    /**
     * 节点执行入口
     */
    public final String apply(ExecuteRequestEntity request, ExecuteContext context) {
        // 1. 前置处理
        preProcess(request, context);

        // 2. 核心逻辑（子类实现）
        String nextNode = doApply(request, context);

        // 3. 后置处理
        postProcess(request, context);

        return nextNode;
    }

    /**
     * 子类实现核心逻辑
     */
    protected abstract String doApply(ExecuteRequestEntity request, ExecuteContext context);

    /**
     * 获取节点的 ContextKey
     */
    public abstract String getContextKey();
}
```

### 6.2 具体节点实现

```java
@Component
public class ExecutePerformerNode extends AbstractExecuteNode {

    @Override
    protected String doApply(ExecuteRequestEntity request, ExecuteContext context) {
        // 1. 获取前置节点输出
        String analyzerResponse = context.getValue(ANALYZER.getContextKey());

        // 2. 获取对应的 ChatClient
        String clientBeanName = getClientBeanName(context.getAgentId(), PERFORMER);
        ChatClient performerClient = getBean(clientBeanName);

        // 3. 构建提示词
        String flowPrompt = getFlowPrompt(context.getFlowId(), PERFORMER);
        String performerPrompt = flowPrompt.formatted(
            context.getUserMessage(),
            analyzerResponse
        );

        // 4. 调用 AI 模型
        String performerResponse = performerClient.prompt(performerPrompt)
            .advisors(a -> a
                .param(CHAT_MEMORY_CONVERSATION_ID_KEY, context.getSessionId())
                .param(CHAT_MEMORY_RETRIEVE_SIZE_KEY, 50))
            .call()
            .content();

        // 5. SSE 推送
        ExecuteResponseEntity responseEntity = ExecuteResponseEntity.createPerformerResponse(
            context.getRound(),
            performerResponse
        );
        sendSseMessage(context, responseEntity);

        // 6. 保存到 Context
        context.setValue(PERFORMER.getContextKey(), performerResponse);

        // 7. 返回下一个节点
        return SUPERVISOR.getContextKey();
    }

    @Override
    public String getContextKey() {
        return "performer";
    }
}
```

---

## 七、工作流配置与管理

### 7.1 数据库设计

#### ai_agent（Agent 表）

| 字段 | 类型 | 说明 |
|-----|------|------|
| agent_id | VARCHAR | Agent ID |
| agent_name | VARCHAR | Agent 名称 |
| agent_type | VARCHAR | Agent 类型（loop/step） |
| agent_desc | VARCHAR | Agent 描述 |
| status | INT | 状态（0-禁用，1-启用） |

#### ai_flow（Flow 表）

| 字段 | 类型 | 说明 |
|-----|------|------|
| flow_id | VARCHAR | Flow ID |
| agent_id | VARCHAR | 关联的 Agent ID |
| client_id | VARCHAR | 关联的 Client ID |
| flow_order | INT | 执行顺序 |
| flow_prompt | TEXT | Flow 提示词 |

### 7.2 配置示例

```sql
-- 创建一个 Loop 类型的 Agent
INSERT INTO ai_agent (agent_id, agent_name, agent_type, agent_desc)
VALUES ('agent_article_writer', '文章生产 Agent', 'loop', '自动生产并发布文章');

-- 配置 Analyzer 节点
INSERT INTO ai_flow (flow_id, agent_id, client_id, flow_order, flow_prompt)
VALUES ('flow_analyzer', 'agent_article_writer', 'client_analyzer', 1, '分析文章主题...');

-- 配置 Performer 节点
INSERT INTO ai_flow (flow_id, agent_id, client_id, flow_order, flow_prompt)
VALUES ('flow_performer', 'agent_article_writer', 'client_performer', 2, '执行文章撰写...');

-- 配置 Supervisor 节点
INSERT INTO ai_flow (flow_id, agent_id, client_id, flow_order, flow_prompt)
VALUES ('flow_supervisor', 'agent_article_writer', 'client_supervisor', 3, '评估文章质量...');

-- 配置 Summarizer 节点
INSERT INTO ai_flow (flow_id, agent_id, client_id, flow_order, flow_prompt)
VALUES ('flow_summarizer', 'agent_article_writer', 'client_summarizer', 4, '总结并发布...');
```

### 7.3 可视化配置

基于 vue-flow 的流程画布：
- 拖拽节点添加
- 连线定义顺序
- 在线编辑 Prompt
- 实时预览流程

---

## 八、业务场景实战

### 场景 1：文章生产流水线

**需求**：根据用户输入自动生产文章并发布

**配置**：
- Agent 类型：Loop
- 节点配置：
  - Analyzer：分析文章主题和风格
  - Performer：搜索资料、撰写文章（支持 MCP 工具）
  - Supervisor：检查文章质量（字数、结构、原创度）
  - Summarizer：发布到平台

**执行示例**：
```
用户: "写一篇关于 AI Agent 的技术文章，1000 字"

Round 1:
  Analyzer: {"topic": "AI Agent", "style": "技术", "keywords": [...]}
  Performer: [调用 webSearch] {"article": "AI Agent 是..."}
  Supervisor: {"passed": false, "reason": "字数不足，只有 800 字"}

Round 2:
  Analyzer: {"需要补充": "技术细节"}
  Performer: {"article": "AI Agent 是...（补充后 1200 字）"}
  Supervisor: {"passed": true}
  Summarizer: "文章已生成并发布到 CSDN"
```

### 场景 2：数据导入任务

**需求**：从 API 获取数据并导入数据库

**配置**：
- Agent 类型：Step
- 节点配置：
  - Inspector：检查 API 可用性
  - Planner：规划导入步骤（分页、去重、验证）
  - Runner：执行导入（失败重试 3 次）
  - Replier：生成导入报告

**执行示例**：
```
用户: "从用户中心 API 导入昨天的新增用户"

Inspector: {"executable": true, "api_available": true}
Planner: {"steps": ["获取用户列表", "去重", "验证", "入库"]}
Runner: {"status": "success", "imported": 1523, "failed": 0}
Replier: "成功导入 1523 条用户记录"
```

---

## 九、自测清单

### 理解自测（共 10 题）

#### 基础概念

**Q1**: Loop 策略和 Step 策略的核心区别是什么？

<details>
<summary>点击查看答案</summary>

**答案**：
| 维度 | Loop | Step |
|-----|------|------|
| 执行模式 | 循环迭代，可回到之前节点 | 线性执行，不回头 |
| 控制节点 | Supervisor 决定是否继续 | 无控制节点 |
| 重试支持 | 无 | Runner 节点支持重试 |
| 适用场景 | 需要多轮优化、监督评估 | 需要严格按步骤执行 |

</details>

---

**Q2**: 工作流中的 Client 和节点是什么关系？

<details>
<summary>点击查看答案</summary>

**答案**：
- **一对一关系**：每个节点对应一个 Client
- **Client**：包含模型配置（API、Model、Advisor）
- **节点**：使用 Client 调用 LLM，执行具体任务
- **示例**：
  - Analyzer 节点 → analyzer_client（使用快速模型）
  - Performer 节点 → performer_client（使用强大模型）
  - Supervisor 节点 → supervisor_client（使用评估优化模型）

</details>

---

#### 实现原理

**Q3**: ExecuteContext 在工作流中起什么作用？

<details>
<summary>点击查看答案</summary>

**答案**：
- **作用**：在整个工作流中传递数据
- **包含内容**：
  - sessionId：会话 ID
  - userInput：用户输入
  - round：当前轮次（Loop 策略）
  - nodeOutputs：各节点输出映射
- **使用方式**：
  - 设置：`context.setValue("analyzer", result)`
  - 获取：`String analyzerOutput = context.getValue("analyzer")`

</details>

---

**Q4**: SSE 推送是如何实现的？前端如何监听？

<details>
<summary>点击查看答案</summary>

**答案**：
**后端推送**：
```java
SseEmitter sseEmitter = new SseEmitter(0L);
sseEmitter.send(SseEmitter.event()
    .name("performer")
    .data(responseData));
```

**前端监听**：
```javascript
const eventSource = new EventSource('/api/v1/ai/work/execute');
eventSource.addEventListener('performer', (event) => {
    const data = JSON.parse(event.data);
    console.log('Performer 输出:', data.content);
});
eventSource.addEventListener('complete', () => {
    eventSource.close();
});
```

</details>

---

**Q5**: Loop 策略如何实现"回到之前节点"？

<details>
<summary>点击查看答案</summary>

**答案**：
通过 Supervisor 节点的评估结果决定：

```java
// Supervisor 节点返回
JSONObject supervisorObject = parseJsonObject(supervisorResponse);
boolean passed = supervisorObject.getBoolean("passed");

if (passed) {
    // 通过 → 进入 Summarizer
    return SUMMARIZER.getContextKey();
} else {
    // 不通过 → 回到 Analyzer，开始新一轮
    context.incrementRound();
    return ANALYZER.getContextKey();
}
```

</details>

---

#### 架构设计

**Q6**: 为什么每个节点需要独立的 Client？

<details>
<summary>点击查看答案</summary>

**答案**：
1. **模型需求不同**：
   - Analyzer：需要快速响应 → 使用小模型
   - Performer：需要高质量输出 → 使用大模型
   - Supervisor：需要评估能力 → 使用专门优化模型
2. **参数配置不同**：
   - Temperature：Analyzer 低，Performer 中，Summarizer 高
   - MaxTokens：各节点需求不同
3. **工具挂载不同**：
   - 只有 Performer 需要 MCP 工具
   - 其他节点不需要工具调用
4. **成本优化**：
   - 不是所有节点都需要昂贵的大模型

</details>

---

**Q7**: 工作流执行失败如何处理？

<details>
<summary>点击查看答案</summary>

**答案**：
**Loop 策略**：
- Supervisor 评估不通过 → 自动重试（回到 Analyzer）
- 设置最大轮次限制，防止无限循环
- 超过限制后返回失败报告

**Step 策略**：
- Runner 节点内置重试机制（maxRetry）
- 重试失败后返回错误信息
- Replier 节点生成错误报告

**通用处理**：
```java
try {
    executeStrategy(request);
} catch (Exception e) {
    sseEmitter.send(SseEmitter.event()
        .name("error")
        .data("执行失败: " + e.getMessage()));
    sseEmitter.complete();
}
```

</details>

---

**Q8**: 如何实现工作流的并发执行？

<details>
<summary>点击查看答案</summary>

**答案**：
当前设计不支持节点并发，但可以通过以下方式实现：

**方案 1：新增 Parallel 策略**
```java
public class ExecuteParallelStrategy implements ExecuteStrategy {
    @Override
    public void execute(ExecuteRequestEntity request, ExecuteContext context) {
        // 并发执行多个节点
        List<CompletableFuture<String>> futures = nodes.stream()
            .map(node -> CompletableFuture.supplyAsync(() -> node.apply(request, context)))
            .collect(Collectors.toList());

        // 等待所有节点完成
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
    }
}
```

**方案 2：节点内并发**
- 在 Performer 节点内并发调用多个 MCP 工具
- 使用 CompletableFuture 或 Reactive Streams

</details>

---

**Q9**: 工作流如何支持人工干预？

<details>
<summary>点击查看答案</summary>

**答案**：
**方案 1：添加人工确认节点**
```java
@Component
public class ExecuteHumanNode extends AbstractExecuteNode {
    @Override
    protected String doApply(ExecuteRequestEntity request, ExecuteContext context) {
        // 发送确认请求
        String confirmId = sendConfirmRequest(context);

        // 等待用户确认（通过 WebSocket 轮询）
        while (true) {
            Boolean confirmed = checkConfirmStatus(confirmId);
            if (confirmed != null) {
                return confirmed ? "next_node" : "terminate";
            }
            Thread.sleep(1000);
        }
    }
}
```

**方案 2：Supervisor 评估时请求人工确认**
- Supervisor 输出：`{"passed": null, "require_human": true}`
- 前端收到后显示确认按钮
- 用户确认后继续执行

</details>

---

**Q10**: 如何监控工作流的执行性能？

<details>
<summary>点击查看答案</summary>

**答案**：
**1. 节点耗时统计**：
```java
long startTime = System.currentTimeMillis();
String result = doApply(request, context);
long duration = System.currentTimeMillis() - startTime;
statService.recordNodeDuration(nodeName, duration);
```

**2. 整体耗时统计**：
```java
workflow.setStartTime(System.currentTimeMillis());
// ... 执行工作流
workflow.setEndTime(System.currentTimeMillis());
workflow.setDuration(workflow.getEndTime() - workflow.getStartTime());
```

**3. Dashboard 展示**：
- 平均执行时间
- 最慢节点分析
- 成功率统计
- 轮次分布（Loop 策略）

</details>

---

### 实战自测（共 3 题）

**Q11**: 设计一个"竞品分析" Agent 工作流。

<details>
<summary>点击查看答案</summary>

**答案**：
**策略选择**：Loop（需要多轮优化）

**节点设计**：
1. **Analyzer**：
   - 分析竞品是谁
   - 确定分析维度（功能、价格、用户体验）
   - 制定分析计划

2. **Performer**：
   - 搜索竞品信息（webSearch 工具）
   - 收集产品数据
   - 整理对比表格

3. **Supervisor**：
   - 检查信息完整性
   - 评估分析深度
   - 决定是否需要补充

4. **Summarizer**：
   - 生成分析报告
   - 提供优化建议

**Prompt 示例**：
```
Analyzer: 你是竞品分析专家，分析用户提出的竞品分析需求...
Performer: 使用 webSearch 工具搜索竞品信息...
Supervisor: 评估分析报告的完整性和深度...
Summarizer: 生成最终的竞品分析报告...
```

</details>

---

**Q12**: 用户反馈工作流执行太慢，如何优化？

<details>
<summary>点击查看答案</summary>

**答案**：
**排查方向**：
1. 查看 Dashboard，确定慢在哪个节点
2. 检查 LLM API 响应时间
3. 检查 MCP 工具调用耗时

**优化方案**：
- **模型层面**：
  - Analyzer 使用更快的模型
  - 减少 MaxTokens
  - 调整 Temperature（降低可加速）
- **架构层面**：
  - 并行执行独立节点
  - 缓存重复查询结果
  - MCP 工具响应优化
- **流程层面**：
  - 减少不必要的轮次
  - 优化 Prompt，减少 LLM 思考时间
  - 设置合理的超时时间

</details>

---

**Q13**: 如何实现工作流的版本管理？

<details>
<summary>点击查看答案</summary>

**答案**：
**方案 1：数据库版本字段**
```sql
ALTER TABLE ai_flow ADD COLUMN version INT DEFAULT 1;
ALTER TABLE ai_agent ADD COLUMN current_version INT DEFAULT 1;
```

**方案 2：配置快照**
```java
public class FlowSnapshot {
    private String snapshotId;
    private String agentId;
    private Integer version;
    private String flowConfigJson;  // 序列化的流程配置
    private Date createdAt;
}
```

**使用场景**：
- 修改工作流前创建快照
- 出问题时快速回滚
- A/B 测试不同版本

**操作界面**：
```
工作流编辑器
├── 保存为新版本
├── 查看历史版本
├── 回滚到指定版本
└── 版本对比
```

</details>

---

## 十、关键代码索引

| 功能 | 文件 | 位置 |
|-----|------|------|
| 工作流入口 | AiController.execute() | backend/ai-agent-trigger/src/main/java/com/dasi/trigger/ |
| 策略分发 | DispatchService.dispatchExecuteStrategy() | backend/ai-agent-domain/src/main/java/com/dasi/domain/dispatch/service/ |
| Loop 策略 | ExecuteLoopStrategy.java | backend/ai-agent-domain/src/main/java/com/dasi/domain/execute/strategy/impl/ |
| Step 策略 | ExecuteStepStrategy.java | backend/ai-agent-domain/src/main/java/com/dasi/domain/execute/strategy/impl/ |
| 节点基类 | AbstractExecuteNode.java | backend/ai-agent-domain/src/main/java/com/dasi/domain/execute/node/ |
| Analyzer 节点 | ExecuteAnalyzerNode.java | backend/ai-agent-domain/src/main/java/com/dasi/domain/execute/node/impl/ |
| Performer 节点 | ExecutePerformerNode.java | backend/ai-agent-domain/src/main/java/com/dasi/domain/execute/node/impl/ |
| Supervisor 节点 | ExecuteSupervisorNode.java | backend/ai-agent-domain/src/main/java/com/dasi/domain/execute/node/impl/ |
| Summarizer 节点 | ExecuteSummarizerNode.java | backend/ai-agent-domain/src/main/java/com/dasi/domain/execute/node/impl/ |

---

**文档版本**：v1.0
**更新日期**：2026-02-24
**下一篇**：专题三 - RAG 知识库深度解析
