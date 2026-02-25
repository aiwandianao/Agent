# AI Agent 平台从零到一：架构理解文档

> **文档定位**：面向 AI 时代的开发者，从架构层面理解如何从零到一构建一个 AI Agent 应用。
> **核心关注**：系统设计、技术选型、实现路径、设计决策——而非具体代码实现细节。

---

## 一、全貌（Overview）

### 1.1 什么是 AI Agent 平台？

AI Agent 平台是一个**具备感知、规划、行动能力的智能系统**，它不是简单的对话机器人，而是能够：

- **理解需求**：通过 LLM 理解用户的自然语言输入
- **规划路径**：将复杂任务拆解为可执行的步骤
- **调用工具**：通过 MCP 协议调用外部服务（如天气查询、发文、搜索）
- **增强知识**：通过 RAG 技术检索私有知识库
- **记忆管理**：维护会话上下文，实现多轮对话
- **工作流编排**：支持多种执行策略（Loop/Step）完成复杂任务

### 1.2 系统全景图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         用户交互层                                    │
├─────────────────────────────────────────────────────────────────────┤
│  聊天界面  │  管理后台  │  可视化工作流  │  Dashboard 数据统计        │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓ HTTP/SSE
┌─────────────────────────────────────────────────────────────────────┐
│                         接口适配层                                    │
├─────────────────────────────────────────────────────────────────────┤
│  AiController (对话/工作流)  │  AdminController (配置管理)            │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│                         领域服务层                                    │
├─────────────────────────────────────────────────────────────────────┤
│  DispatchService (分发)    │  AugmentService (增强)                  │
│  ExecuteService (执行)     │  RagService (知识库)                     │
│  SessionService (会话)     │  AdminService (管理)                     │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│                         基础设施层                                    │
├─────────────────────────────────────────────────────────────────────┤
│  DAO (数据访问)  │  Redis (缓存/记忆)  │  PgVectorStore (向量存储)    │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│                         外部服务层                                    │
├─────────────────────────────────────────────────────────────────────┤
│  LLM API (豆包/通义/DeepSeek等)  │  MCP Server (工具服务)             │
│  MySQL (业务数据)  │  PostgreSQL (向量数据库)  │  Redis (缓存)        │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.3 核心技术栈

| 层次 | 技术选型 | 选型理由 |
|-----|---------|---------|
| **前端框架** | Vue 3 + Vite | 轻量高效，生态完善， Composition API 适合复杂状态管理 |
| **后端框架** | Spring Boot 3 | Java 企业级开发标准，AI 集成成熟（Spring AI） |
| **AI 框架** | Spring AI | 官方 AI 框架，屏蔽多模型差异，支持工具调用 |
| **数据库** | MySQL + PostgreSQL | MySQL 存业务数据，PostgreSQL + pgvector 存向量 |
| **缓存** | Redis | 会话记忆、配置缓存、实时推送 |
| **流程可视化** | vue-flow | 基于 React Flow 的 Vue 实现，适合工作流编排 |
| **工具协议** | MCP (Model Context Protocol) | Anthropic 标准，统一工具调用接口 |

### 1.4 DDD 分层架构

```
┌──────────────────────────────────────────────────────┐
│  ai-agent-types        (公共对象层：DTO、常量、Result)  │
├──────────────────────────────────────────────────────┤
│  ai-trigger (接口适配层：Controller、拦截器、参数校验) │
├──────────────────────────────────────────────────────┤
│  ai-agent-app       (应用层：配置类、启动类、AOP切面)  │
├──────────────────────────────────────────────────────┤
│  ai-agent-domain    (领域层：核心业务逻辑、服务接口)   │
├──────────────────────────────────────────────────────┤
│  ai-agent-infrastructure  (基础设施层：DAO、PO、外部服务) │
├──────────────────────────────────────────────────────┤
│  ai-agent-api        (API层：接口定义、请求响应契约)   │
└──────────────────────────────────────────────────────┘
```

**分层原则**：
- **依赖方向**：上层依赖下层，下层不依赖上层
- **职责分离**：每层只关注自己的职责
- **可替换性**：更换实现只需修改对应层

---

## 二、可以怎样实现（Alternatives）

### 2.1 整体架构方案对比

| 方案 | 优势 | 劣势 | 适用场景 |
|-----|------|------|---------|
| **单体架构** | 简单易部署，开发成本低 | 扩展性差，耦合度高 | 小型项目，快速验证 |
| **微服务架构** | 独立部署扩展，技术栈灵活 | 运维复杂，分布式事务难处理 | 大型项目，多团队协作 |
| **DDD 分层架构** | 职责清晰，易于维护和扩展 | 需要领域建模经验 | 中大型项目，复杂业务 |
| **事件驱动架构** | 松耦合，异步处理 | 调试复杂，一致性难保证 | 高并发场景，需要解耦 |

**本项目选择**：**DDD 分层架构** —— 适合 AI Agent 这种业务逻辑复杂的场景。

### 2.2 LLM 集成方案对比

| 方案 | 实现方式 | 优势 | 劣势 |
|-----|---------|------|------|
| **直接调用 API** | HTTP 调用各厂商 API | 灵活，可定制性强 | 需要处理各厂商差异 |
| **LangChain/LangGraph4j** | 使用编排框架 | 功能丰富，生态完善 | 学习曲线陡，性能开销 |
| **Spring AI** | 官方 AI 框架 | 与 Spring 生态无缝集成，屏蔽模型差异 | 功能相对简单 |

**本项目选择**：**Spring AI** —— 与 Spring Boot 完美集成，支持 ChatClient、工具调用、RAG、记忆管理。

### 2.3 工作流编排方案对比

| 方案 | 实现方式 | 优势 | 劣势 |
|-----|---------|------|------|
| **硬编码流程** | if-else 逻辑判断 | 简单直接 | 不灵活，难以扩展 |
| **状态机** | 定义状态和转移规则 | 可视化，易调试 | 复杂场景状态爆炸 |
| **责任链模式** | 节点串联处理 | 灵活，易扩展 | 链路长时难追踪 |
| **DSL + 解释器** | 定义领域特定语言 | 表达力强 | 开发成本高 |

**本项目选择**：**责任链模式 + 策略模式** —— 支持 Loop/Step 两种策略，每个节点独立实现。

### 2.4 向量数据库方案对比

| 方案 | 优势 | 劣势 | 成本 |
|-----|------|------|------|
| **专用向量数据库 (Milvus/Qdrant)** | 性能高，功能专业 | 运维成本高，需要独立部署 | 高 |
| **PostgreSQL + pgvector** | 一库多用，运维简单 | 性能略逊专用库 | 低 |
| **云服务 (Pinecone/Weaviate Cloud)** | 免运维，扩展性好 | 数据隐私，成本高 | 中高 |

**本项目选择**：**PostgreSQL + pgvector** —— 降低运维复杂度，满足中小规模需求。

### 2.5 会话记忆存储方案对比

| 方案 | 实现方式 | 优势 | 劣势 |
|-----|---------|------|------|
| **数据库存储** | MySQL 存历史消息 | 持久化可靠 | 查询性能差，不适合高频读取 |
| **Redis 存储** | Redis List/Hash | 高性能，支持过期 | 内存成本高 |
| **混合方案** | Redis 热 + MySQL 冷 | 兼顾性能和持久化 | 实现复杂 |

**本项目选择**：**Redis 存储** —— Spring AI 自带 RedisChatMemoryAdvisor，开箱即用。

### 2.6 MCP 通信方式对比

| 方案 | 实现方式 | 优势 | 劣势 |
|-----|---------|------|------|
| **SSE (Server-Sent Events)** | 长连接推送 | 实时性好，适合远程服务 | 需要保持连接 |
| **STDIO** | 标准输入输出通信 | 简单，适合本地进程 | 无法远程调用 |
| **WebSocket** | 双向通信 | 双向实时 | 协议开销大 |

**本项目选择**：**同时支持 SSE 和 STDIO** —— 灵活适应不同场景。

---

## 三、具体某种实现（Current Path）

### 3.1 对话系统实现

#### 3.1.1 请求处理流程

```
用户输入 → Controller → 参数校验 → 会话校验 → 获取 ChatClient
    ↓
增强服务（RAG 检索 + MCP 工具挂载）
    ↓
设置 ChatMemory (会话记忆)
    ↓
调用 LLM API (流式/非流式)
    ↓
返回结果 → 持久化消息 → 记录统计
```

#### 3.1.2 核心代码位置

| 功能 | 文件位置 | 行号 |
|-----|---------|------|
| 对话入口 | `AiController.java` | 136-208 |
| RAG 增强 | `AugmentService.java` | 57-88 |
| MCP 增强 | `AugmentService.java` | 91-152 |
| 会话管理 | `SessionService.java` | 全文 |

#### 3.1.3 关键实现要点

1. **ChatClient 动态获取**：根据前端传入的 `clientId` 从 Spring 容器获取对应的 Bean
2. **RAG 增强流程**：向量检索 → 构建系统提示 → 注入到对话上下文
3. **MCP 工具挂载**：动态创建 `McpSyncClient` → 封装为 `ToolCallbackProvider` → 挂载到 ChatClient
4. **会话记忆**：通过 `ChatMemoryAdvisor` 实现，使用 `sessionId` 作为 `conversationId`

### 3.2 Agent 工作流实现

#### 3.2.1 Loop 策略（循环模式）

```
Analyzer (分析任务)
    ↓
Performer (执行任务，可调用 MCP 工具)
    ↓
Supervisor (监督评估)
    ↓
通过？ → Summarizer (总结)
    ↓ 否，回到 Analyzer
完成
```

**适用场景**：需要多轮迭代、监督评估的复杂任务。

#### 3.2.2 Step 策略（步骤模式）

```
Inspector (检查任务)
    ↓
Planner (规划步骤)
    ↓
Runner (执行步骤，失败重试)
    ↓
Replier (回复用户)
```

**适用场景**：线性步骤、需要重试机制的任务。

#### 3.2.3 节点实现模板

每个节点继承 `AbstractExecuteNode`，实现 `doApply()` 方法：

```java
@Override
protected String doApply(ExecuteRequestEntity request, ExecuteContext context) {
    // 1. 从 Context 获取前置节点输出
    String analyzerResponse = context.getValue(ANALYZER.getContextKey());

    // 2. 获取对应的 ChatClient
    ChatClient client = getBean(clientBeanName);

    // 3. 构建提示词（支持模板变量）
    String prompt = flowPrompt.formatted(
        context.getUserMessage(),
        analyzerResponse
    );

    // 4. 调用 AI 模型（带 ChatMemory）
    String response = client.prompt(prompt)
        .advisors(a -> a
            .param(CHAT_MEMORY_CONVERSATION_ID_KEY, sessionId)
            .param(CHAT_MEMORY_RETRIEVE_SIZE_KEY, 50))
        .call()
        .content();

    // 5. SSE 推送结果
    sendSseMessage(context, response);

    // 6. 保存到 Context
    context.setValue(PERFORMER.getContextKey(), response);

    // 7. 返回下一个节点
    return router(request, context);
}
```

#### 3.2.4 执行上下文传递

```
ExecuteContext
├── userMessage (用户输入)
├── sessionId (会话 ID)
├── round (轮次计数)
└── nodeOutputs (各节点输出映射)
    ├── analyzer: "..."
    ├── performer: "..."
    ├── supervisor: "..."
    └── summarizer: "..."
```

### 3.3 RAG 知识库实现

#### 3.3.1 文档处理链路

```
文档上传 → TikaDocumentReader 读取
    ↓
TokenTextSplitter 切片（按 Token 数）
    ↓
写入 PgVectorStore（附带 knowledge=ragTag 元数据）
    ↓
完成
```

#### 3.3.2 检索增强流程

```java
// 1. 构建检索条件（按知识库标签过滤）
Filter.Expression expression = filterExpressionBuilder
    .eq("knowledge", ragTag)
    .build();

// 2. 执行向量检索
SearchRequest searchRequest = SearchRequest.builder()
    .query(userMessage)
    .filterExpression(expression)
    .topK(5)
    .build();

List<Document> documents = pgVectorStore.similaritySearch(searchRequest);

// 3. 注入系统提示
String documentString = documents.stream()
    .map(Document::getText)
    .collect(Collectors.joining("\n"));

return List.of(
    new SystemPromptTemplate(RAG_SYSTEM_PROMPT)
        .createMessage(Map.of("documents", documentString)),
    new UserMessage(userMessage)
);
```

### 3.4 MCP 服务实现

#### 3.4.1 MCP Server 开发模板

```java
@Component
public class CustomTool {

    @Tool(description = "工具描述")
    public String customMethod(String param) {
        // 实现逻辑
        return "result";
    }
}
```

#### 3.4.2 SSE 方式调用

```java
HttpClientSseClientTransport transport = HttpClientSseClientTransport
    .builder(baseUri)
    .sseEndpoint(sseEndPoint)
    .build();

McpSyncClient client = McpClient
    .sync(transport)
    .requestTimeout(Duration.ofMinutes(timeout))
    .build();

client.initialize();
```

#### 3.4.3 工具挂载到 ChatClient

```java
SyncMcpToolCallbackProvider toolCallbacks = augmentService.augmentMcpTool(mcpIdList);

chatClient.prompt()
    .toolCallbacks(toolCallbacks)
    .call()
    .content();
```

### 3.5 配置管理系统实现

#### 3.5.1 Armory 自动装配机制

```
启动时读取 armory 配置
    ↓
按责任链顺序装配：API → Model → MCP → Advisor → Client → Flow → Agent
    ↓
缓存已装配 ID（Redis Set 防重）
    ↓
完成
```

#### 3.5.2 AOP 缓存切面

```java
@Cacheable(key = "#clientId", expire = 3600)
public AiClientVO queryAiClientVOByClientId(String clientId) {
    // 查询数据库
}

@CacheEvict(key = "#clientId")
public void updateAiClient(AiClient client) {
    // 更新数据库，自动清除缓存
}
```

---

## 四、为什么这样实现（Rationale）

### 4.1 为什么选择 DDD 分层架构？

**设计动机**：
1. **业务复杂性**：AI Agent 涉及对话、工作流、知识库、工具调用等多个领域，需要清晰的边界
2. **团队协作**：分层后不同人员可专注不同层，减少冲突
3. **可测试性**：每层可独立测试，领域层不依赖基础设施
4. **可替换性**：更换 LLM 提供商、向量数据库只需修改基础设施层

**权衡取舍**：
- **代价**：增加了一些样板代码，需要学习 DDD 概念
- **收益**：长期维护成本降低，扩展更容易

### 4.2 为什么选择责任链 + 策略模式实现工作流？

**设计动机**：
1. **灵活性**：新增节点无需修改现有代码，符合开闭原则
2. **可读性**：每个节点职责单一，代码逻辑清晰
3. **可测试性**：每个节点可独立测试
4. **可视化**：基于 vue-flow 可直观展示工作流

**替代方案考量**：
- **硬编码**：不灵活，每次改流程都要改代码
- **状态机**：Loop 策略是循环状态，状态机会很复杂
- **DSL**：开发成本高，学习曲线陡

### 4.3 为什么选择 PostgreSQL + pgvector 而非专用向量数据库？

**设计动机**：
1. **运维简化**：少一个组件就少一份运维负担
2. **成本优化**：无需单独部署向量数据库
3. **数据一致性**：业务数据和向量数据在同一事务中
4. **性能足够**：pgvector 索引性能可满足中小规模需求

**何时需要升级到专用向量数据库**：
- 数据量超过千万级
- 需要更高并发
- 需要更专业的向量功能（如混合检索、重排序）

### 4.4 为什么支持 SSE 和 STDIO 两种 MCP 通信方式？

**设计动机**：
1. **场景适配**：
   - SSE：适合远程部署的 MCP 服务，如云端工具服务
   - STDIO：适合本地部署的 MCP 服务，如本地脚本工具
2. **灵活性**：用户可根据实际场景选择
3. **标准化**：遵循 MCP 协议规范

### 4.5 为什么使用 Redis 存储会话记忆？

**设计动机**：
1. **性能**：会话记忆需要高频读取，Redis 性能远高于数据库
2. **过期策略**：Redis 原生支持 TTL，可自动清理过期记忆
3. **框架支持**：Spring AI 自带 RedisChatMemoryAdvisor
4. **简单性**：无需实现复杂的冷热数据分离

**潜在问题**：
- Redis 容量有限，大量会话可能撑满内存
- **解决方案**：可设置合理的过期时间，或升级到冷热分离方案

### 4.6 为什么采用配置驱动而非代码硬编码？

**设计动机**：
1. **灵活性**：通过管理后台修改配置，无需重启服务
2. **可观测性**：配置可视化，便于理解和调试
3. **多租户支持**：不同用户/场景可使用不同配置
4. **A/B 测试**：可同时运行多个配置进行对比

**配置项**：
- API：LLM 提供商配置
- Model：模型选择和参数
- MCP：工具服务配置
- Advisor：记忆、重试等策略
- Client：模型 + API + Advisor 组合
- Prompt：提示词模板
- Flow：工作流配置
- Agent：Client + Flow 组合

---

## 五、扩展性（Extensibility）

### 5.1 如何新增 AI 模型提供商？

**步骤**：
1. 在管理后台添加 API 配置（如 DeepSeek API）
2. 添加 Model 配置并关联 API
3. 创建 Client 并关联 Model
4. 测试调用

**无需修改代码**：Spring AI 自动适配 OpenAI 兼容接口。

### 5.2 如何新增 MCP 服务？

**步骤**：
1. 创建新的 Spring Boot 项目（mcp-server-xxx）
2. 实现 Tool 接口并添加 @Tool 注解
3. 配置 SSE 端点
4. 部署并测试
5. 在管理后台添加 MCP 配置

**示例**：新增一个「飞书通知」MCP 服务。

### 5.3 如何新增工作流策略？

**步骤**：
1. 创建新的 ExecuteStrategy 实现类
2. 定义对应的节点（继承 AbstractExecuteNode）
3. 在 ExecuteStrategyFactory 中注册
4. 在管理后台创建 Agent 并选择新策略

**示例**：新增一个「并行执行」策略，多个节点同时执行。

### 5.4 如何新增知识库类型？

**步骤**：
1. 实现新的文档读取器（如支持 PDF、Word）
2. 实现新的切片策略（如按语义切片）
3. 扩展向量存储（如支持 Elasticsearch）

### 5.5 架构扩展点

```
┌─────────────────────────────────────────────────────┐
│  前端扩展点                                           │
│  - 新增可视化组件                                     │
│  - 新增管理界面                                       │
└─────────────────────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────┐
│  后端扩展点                                           │
│  - 新增 Controller (接口适配层)                      │
│  - 新增 Service (领域层)                             │
│  - 新增 ExecuteStrategy (工作流策略)                 │
│  - 新增 ExecuteNode (工作流节点)                     │
└─────────────────────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────┐
│  基础设施扩展点                                       │
│  - 新增 DAO (数据访问)                               │
│  - 新增 外部服务集成 (如新的 MCP 协议支持)            │
└─────────────────────────────────────────────────────┘
```

---

## 六、可用性（Availability）

### 6.1 容错机制

| 组件 | 容错策略 | 实现方式 |
|-----|---------|---------|
| **LLM API** | 超时重试 | Spring AI 内置 RetryAdvisor |
| **MCP 服务** | 失败隔离 | 单个 MCP 失败不影响其他，降级处理 |
| **Redis** | 缓存降级 | Redis 不可用时直接查数据库 |
| **数据库** | 连接池 | HikariCP 自动重连 |
| **工作流** | 节点重试 | Runner 节点支持 maxRetry 配置 |

### 6.2 监控与告警

- **统计记录**：StatService 记录每次对话/工作流的调用数据
- **Dashboard**：可视化展示调用趋势、成功率
- **日志**：关键节点记录日志，便于排查问题

### 6.3 性能优化

| 优化点 | 实现方式 | 效果 |
|-------|---------|------|
| **配置缓存** | Redis + AOP | 减少 90% 数据库查询 |
| **异步执行** | 线程池执行工作流 | 提升吞吐量 |
| **SSE 推送** | 实时推送执行进度 | 用户体验好 |
| **连接池** | HTTP 客户端连接池 | 减少连接建立开销 |

---

## 七、如何验证（Verification）

### 7.1 单元测试

```java
@SpringBootTest
class AugmentServiceTest {

    @Autowired
    private AugmentService augmentService;

    @Test
    void testMcpToolAugment() {
        // Given
        List<String> mcpIdList = List.of("mcp_amap");

        // When
        SyncMcpToolCallbackProvider result = augmentService.augmentMcpTool(mcpIdList);

        // Then
        assertNotNull(result);
        assertEquals(1, result.getToolCallbacks().size());
    }
}
```

### 7.2 集成测试

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class AiControllerIntegrationTest {

    @LocalServerPort
    private int port;

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void testChatComplete() {
        // Given
        AiChatRequest request = new AiChatRequest();
        request.setSessionId("test-session");
        request.setMessage("你好");

        // When
        ResponseEntity<Result> response = restTemplate.postForEntity(
            "/api/v1/ai/chat/complete",
            request,
            Result.class
        );

        // Then
        assertEquals(HttpStatus.OK, response.getStatusCode());
    }
}
```

### 7.3 工作流验证

**Loop 策略验证**：
1. 创建一个 Loop 类型的 Agent
2. 配置 Analyzer/Performer/Supervisor/Summarizer 节点
3. 执行工作流，观察节点输出
4. 验证 Supervisor 未通过时会回到 Analyzer

**Step 策略验证**：
1. 创建一个 Step 类型的 Agent
2. 配置 Inspector/Planner/Runner/Replier 节点
3. 执行工作流
4. 验证 Runner 失败时会重试

### 7.4 RAG 验证

1. 上传测试文档到知识库
2. 发送相关查询
3. 验证返回结果包含文档内容

### 7.5 MCP 验证

1. 确认 MCP 服务正常运行
2. 在管理后台配置 MCP
3. 发送需要调用工具的消息
4. 验证工具被正确调用

---

## 八、总结

### 8.1 核心要点回顾

1. **全貌**：AI Agent 平台 = LLM + 工具调用 + 知识检索 + 工作流编排 + 记忆管理
2. **实现方案**：DDD 分层 + Spring AI + MCP 协议 + PostgreSQL + Redis
3. **设计决策**：每个技术选型都有明确的动机和权衡
4. **扩展性**：通过配置和接口设计支持灵活扩展
5. **可用性**：容错、缓存、异步保证系统稳定
6. **验证方法**：单元测试、集成测试、端到端测试

### 8.2 学习路径建议

1. **第一步**：运行项目，体验完整功能
2. **第二步**：阅读技术设计方案，理解架构
3. **第三步**：阅读核心源码（DispatchService、ExecuteService）
4. **第四步**：尝试新增一个 MCP 服务
5. **第五步**：尝试创建一个新的 Agent 工作流
6. **第六步**：基于项目进行二次开发

### 8.3 关键文件索引

| 功能模块 | 关键文件 | 位置 |
|---------|---------|------|
| 对话入口 | AiController.java | backend/ai-agent-trigger/src/main/java/com/dasi/trigger/ |
| 增强服务 | AugmentService.java | backend/ai-agent-domain/src/main/java/com/dasi/domain/augment/service/ |
| 执行服务 | ExecuteService.java | backend/ai-agent-domain/src/main/java/com/dasi/domain/execute/service/ |
| Loop 策略 | ExecuteLoopStrategy.java | backend/ai-agent-domain/src/main/java/com/dasi/domain/execute/strategy/impl/ |
| Step 策略 | ExecuteStepStrategy.java | backend/ai-agent-domain/src/main/java/com/dasi/domain/execute/strategy/impl/ |
| RAG 服务 | RagService.java | backend/ai-agent-domain/src/main/java/com/dasi/domain/rag/service/ |
| 自动装配 | ArmoryConfig.java | backend/ai-agent-app/src/main/java/com/dasi/config/ |

---

**文档版本**：v1.0
**更新日期**：2026-02-24
**维护者**：AI Agent 平台团队
