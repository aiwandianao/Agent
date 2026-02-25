# 专题四：MCP 工具系统深度解析

> **学习目标**：掌握 MCP（Model Context Protocol）的原理与实现，能够开发和集成自定义工具。
> **前置知识**：了解 HTTP 协议、JSON-RPC、工具调用概念
> **预计用时**：55 分钟

---

## 一、真实业务场景

### 场景 1：企业自动化助手

**业务需求**：
- LLM 只能"说"不能"做"，需要调用实际的服务
- 查询天气、发送邮件、发布文章、查询订单
- 不同服务的调用方式各异，需要统一标准

**对应技术**：
- MCP 协议统一工具调用接口
- LLM 自动选择合适的工具
- 工具执行结果返回给 LLM

### 场景 2：企业内部系统集成

**业务需求**：
- 企业有多个内部系统（ERP、CRM、OA）
- 需要让 LLM 能够调用这些系统
- 不同系统的认证、协议不同

**对应技术**：
- 为每个系统开发 MCP Server
- 统一封装为 MCP 工具
- LLM 通过 MCP 调用各系统

### 场景 3：第三方服务集成

**业务需求**：
- 集成高德地图、企业微信、阿里云等服务
- 这些服务都有自己的 API
- 需要统一接入到 LLM

**对应技术**：
- 开发对应的 MCP Server
- 封装第三方 API 为工具
- 挂载到 ChatClient

---

## 二、MCP 协议详解

### 2.1 什么是 MCP？

**定义**：Model Context Protocol（模型上下文协议），由 Anthropic 提出的标准化协议

**核心价值**：
1. **统一接口**：不同工具用统一方式调用
2. **标准化**：工具定义、调用、返回格式统一
3. **可扩展**：轻松添加新工具
4. **解耦**：LLM 与工具实现解耦

### 2.2 MCP 架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                         LLM (如 Claude)                              │
│  决策：是否需要调用工具？调用哪个工具？传递什么参数？                 │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓ JSON-RPC
┌─────────────────────────────────────────────────────────────────────┐
│                         MCP Client                                   │
│  - 工具发现（列出可用工具）                                          │
│  - 工具调用（调用具体工具）                                          │
│  - 结果返回（返回工具执行结果）                                      │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓ SSE / STDIO
┌─────────────────────────────────────────────────────────────────────┐
│                         MCP Server                                   │
│  - 工具定义（@Tool 注解）                                            │
│  - 工具实现（具体业务逻辑）                                          │
│  - 结果封装（统一返回格式）                                          │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│                         外部服务                                     │
│  高德天气 API │ 企业微信 API │ 邮件服务 │ 数据库 │ ...                │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.3 MCP 通信方式

#### SSE（Server-Sent Events）

```
适用场景：远程部署的 MCP 服务

MCP Client                              MCP Server
    │                                      │
    ├─────── Initialize ──────────────────→│
    │←─────── Initialized (tools: [...]) ──┤
    │                                      │
    ├─────── Call Tool: checkWeather ────→│
    │←─────── Result: {temperature: 25} ───┤
    │                                      │
```

**优势**：
- 支持远程服务
- 实时性好
- 标准协议

**代码示例**：
```java
HttpClientSseClientTransport transport = HttpClientSseClientTransport
    .builder("http://localhost:9001")
    .sseEndpoint("/mcp/sse")
    .build();

McpSyncClient client = McpClient.sync(transport).build();
client.initialize();
```

#### STDIO（标准输入输出）

```
适用场景：本地进程的 MCP 服务

MCP Client                    MCP Server Process
    │                              │
    ├─ stdin: Initialize ─────────→│
    │←─ stdout: Initialized ───────┤
    │                              │
    ├─ stdin: Call Tool ──────────→│
    │←─ stdout: Result ────────────┤
    │                              │
```

**优势**：
- 简单，适合本地工具
- 无需网络
- 启动快

**代码示例**：
```java
ServerParameters params = ServerParameters.builder("python")
    .args("mcp_server.py")
    .build();

StdioClientTransport transport = new StdioClientTransport(params);
McpSyncClient client = McpClient.sync(transport).build();
```

---

## 三、MCP Server 开发

### 3.1 项目结构

```
mcp-server-xxx/
├── pom.xml                          # Maven 配置
├── src/main/java/
│   └── com/example/mcp/
│       ├── McpServerApplication.java    # 启动类
│       ├── config/
│       │   └── McpServerConfig.java     # MCP 配置
│       └── tool/
│           └── XxxTool.java             # 工具实现
└── src/main/resources/
    └── application.yml                  # 应用配置
```

### 3.2 最简示例

#### 步骤 1：创建 Spring Boot 项目

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-mcp-spring-boot-starter</artifactId>
    </dependency>
</dependencies>
```

#### 步骤 2：实现工具

```java
@Component
public class WeatherTool {

    @Tool(description = "查询指定城市的天气")
    public WeatherResponse checkWeather(
        @P("城市名称") String city
    ) {
        // 调用高德天气 API
        String url = "https://restapi.amap.com/v3/weather/weatherInfo?key=XXX&city=" + city;
        String result = restTemplate.getForObject(url, String.class);

        // 解析结果
        JSONObject json = JSON.parseObject(result);
        String temperature = json.getJSONObject("lives")
            .getJSONObject("0")
            .getString("temperature");

        return WeatherResponse.builder()
            .city(city)
            .temperature(temperature)
            .build();
    }
}
```

#### 步骤 3：配置 SSE 端点

```yaml
# application.yml
server:
  port: 9001

spring:
  application:
    name: mcp-server-weather

mcp:
  server:
    sse-endpoint: /mcp/sse
```

#### 步骤 4：启动服务

```java
@SpringBootApplication
public class McpServerWeatherApplication {
    public static void main(String[] args) {
        SpringApplication.run(McpServerWeatherApplication.class, args);
    }
}
```

### 3.3 工具定义规范

#### @Tool 注解

```java
@Tool(
    description = "工具描述，LLM 会根据描述选择是否调用此工具",
    name = "toolName"  // 可选，默认使用方法名
)
public String toolMethod(@P("参数描述") String param) {
    // 实现
}
```

#### 参数注解 @P

```java
public String search(
    @P("搜索关键词") String keyword,
    @P("结果数量，默认10") @DefaultValue("10") int count,
    @P("是否精确匹配") boolean exact
) {
    // 实现
}
```

#### 返回值类型

| 返回类型 | 说明 | 示例 |
|---------|------|------|
| **String** | 简单文本返回 | "操作成功" |
| **对象** | 结构化数据返回 | `{"temperature": "25°C"}` |
| **List** | 列表数据返回 | `[{"id": 1}, {"id": 2}]` |

### 3.4 完整示例：天气查询工具

```java
@Component
public class AmapTool {

    @Resource
    private RestTemplate restTemplate;

    @Value("${amap.api.key}")
    private String amapApiKey;

    @Tool(description = "根据城市名称查询当前天气")
    public WeatherResponse checkWeather(
        @P("城市名称，如：北京、上海") String city
    ) {
        try {
            // 1. 调用高德天气 API
            String url = String.format(
                "https://restapi.amap.com/v3/weather/weatherInfo?key=%s&city=%s&extensions=base",
                amapApiKey, city
            );

            String response = restTemplate.getForObject(url, String.class);

            // 2. 解析响应
            JSONObject json = JSON.parseObject(response);
            JSONObject lives = json.getJSONObject("lives");
            if (lives == null || lives.getJSONArray("0") == null) {
                return WeatherResponse.builder()
                    .city(city)
                    .error("未找到该城市信息")
                    .build();
            }

            JSONObject weather = lives.getJSONObject("0");

            // 3. 构建返回结果
            return WeatherResponse.builder()
                .city(city)
                .province(weather.getString("province"))
                .weather(weather.getString("weather"))
                .temperature(weather.getString("temperature"))
                .winddirection(weather.getString("winddirection"))
                .windpower(weather.getString("windpower"))
                .humidity(weather.getString("humidity"))
                .reporttime(weather.getString("reporttime"))
                .build();

        } catch (Exception e) {
            return WeatherResponse.builder()
                .city(city)
                .error("查询失败: " + e.getMessage())
                .build();
        }
    }
}

@Data
@Builder
class WeatherResponse {
    private String city;
    private String province;
    private String weather;
    private String temperature;
    private String winddirection;
    private String windpower;
    private String humidity;
    private String reporttime;
    private String error;
}
```

---

## 四、MCP Client 集成

### 4.1 动态挂载流程

```
用户发起对话 (mcpIdList = ["amap", "wecom"])
    ↓
1. 从数据库获取 MCP 配置
   amap: {baseUri: "http://localhost:9001", sseEndpoint: "/mcp/sse"}
   wecom: {baseUri: "http://localhost:9002", sseEndpoint: "/mcp/sse"}
    ↓
2. 创建 McpSyncClient
   amapClient = McpClient.sync(sseTransport(amapConfig)).build()
   wecomClient = McpClient.sync(sseTransport(wecomConfig)).build()
    ↓
3. 初始化（获取工具列表）
   amapClient.initialize() → tools: [checkWeather]
   wecomClient.initialize() → tools: [sendText, sendTextCard]
    ↓
4. 封装为 ToolCallbackProvider
   toolCallbacks = new SyncMcpToolCallbackProvider([amapClient, wecomClient])
    ↓
5. 挂载到 ChatClient
   chatClient.prompt()
       .toolCallbacks(toolCallbacks)
       .call()
       .content()
```

### 4.2 核心代码

```java
@Service
public class AugmentService {

    public SyncMcpToolCallbackProvider augmentMcpTool(List<String> mcpIdList) {
        List<McpSyncClient> mcpClients = new ArrayList<>();

        for (String mcpId : mcpIdList) {
            // 1. 获取 MCP 配置
            McpConfig mcpConfig = getMcpConfig(mcpId);

            // 2. 创建传输层
            if ("sse".equals(mcpConfig.getMcpType())) {
                HttpClientSseClientTransport transport = HttpClientSseClientTransport
                    .builder(mcpConfig.getBaseUri())
                    .sseEndpoint(mcpConfig.getSseEndpoint())
                    .build();

                // 3. 创建 MCP Client
                McpSyncClient client = McpClient
                    .sync(transport)
                    .requestTimeout(Duration.ofMinutes(mcpConfig.getTimeout()))
                    .build();

                // 4. 初始化
                client.initialize();
                mcpClients.add(client);
            }
        }

        // 5. 封装为 ToolCallbackProvider
        return new SyncMcpToolCallbackProvider(mcpClients);
    }
}
```

### 4.3 使用示例

```java
// Controller
@PostMapping("/chat/stream")
public SseEmitter chatStream(@RequestBody AiChatRequest request) {
    // 1. 获取 ChatClient
    ChatClient chatClient = getChatClient(request.getClientId());

    // 2. 挂载 MCP 工具
    SyncMcpToolCallbackProvider toolCallbacks = augmentService.augmentMcpTool(request.getMcpIdList());

    // 3. 调用 LLM
    String response = chatClient.prompt()
        .user(request.getMessage())
        .toolCallbacks(toolCallbacks)  // 挂载工具
        .call()
        .content();

    // 4. 返回结果
    return response;
}
```

---

## 五、工具调用过程

### 5.1 完整流程图

```
用户："帮我查一下北京今天的天气"
    ↓
┌─────────────────────────────────────────────────────────────────────┐
│  Step 1: LLM 理解意图                                               │
│  分析：用户需要查询天气，需要调用工具                                │
└─────────────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────────────┐
│  Step 2: LLM 选择工具                                              │
│  选择：checkWeather (描述：查询指定城市的天气)                       │
└─────────────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────────────┐
│  Step 3: LLM 生成工具调用                                           │
│  {                                                                  │
│    "tool": "checkWeather",                                          │
│    "arguments": {"city": "北京"}                                    │
│  }                                                                  │
└─────────────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────────────┐
│  Step 4: MCP Client 调用 MCP Server                                 │
│  Call Tool: checkWeather({"city": "北京"})                          │
└─────────────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────────────┐
│  Step 5: MCP Server 执行工具                                        │
│  1. 接收调用请求                                                    │
│  2. 解析参数：city = "北京"                                         │
│  3. 调用高德天气 API                                                │
│  4. 解析响应                                                        │
│  5. 构建返回结果                                                    │
└─────────────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────────────┐
│  Step 6: MCP Server 返回结果                                        │
│  {                                                                  │
│    "city": "北京",                                                  │
│    "weather": "晴",                                                 │
│    "temperature": "25°C",                                           │
│    "windpower": "3级"                                               │
│  }                                                                  │
└─────────────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────────────┐
│  Step 7: LLM 基于工具结果生成最终回复                                │
│  "北京今天天气晴朗，温度 25 摄氏度，风力 3 级。"                      │
└─────────────────────────────────────────────────────────────────────┘
    ↓
返回给用户
```

### 5.2 多工具调用

```
用户："查一下北京天气，然后通知团队今天下午3点开会"
    ↓
┌─────────────────────────────────────────────────────────────────────┐
│  LLM 决策：需要调用两个工具                                         │
│  1. checkWeather({"city": "北京"})                                 │
│  2. sendText({"content": "今天下午3点开会，北京天气晴，25度"})      │
└─────────────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────────────┐
│  工具执行（串行或并行）                                             │
│  方式1：串行执行                                                    │
│    checkWeather → 等待结果 → sendText                               │
│                                                                     │
│  方式2：并行执行（如果工具间无依赖）                                 │
│    checkWeather ─┐                                                  │
│                   ├→ 并行执行 → 等待所有结果 → LLM 生成回复           │
│    sendText ─────┘                                                  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 六、已实现的 MCP 服务

### 6.1 服务清单

| 服务名称 | 端口 | 工具 | 功能 |
|---------|------|------|------|
| mcp-server-amap | 9001 | checkWeather | 查询天气 |
| mcp-server-csdn | 9002 | saveArticle | 发布文章到 CSDN |
| mcp-server-wecom | 9003 | sendText, sendTextCard | 发送企业微信消息 |
| mcp-server-email | 9004 | sendEmail | 发送邮件 |
| mcp-server-bocha | 9005 | webSearch | 联网搜索 |

### 6.2 高德天气工具

```java
@Tool(description = "根据城市名称查询当前天气")
public WeatherResponse checkWeather(
    @P("城市名称") String city
) {
    // 调用高德天气 API
    // 返回天气信息
}
```

**使用示例**：
```
用户："帮我查一下北京今天的天气"
LLM：调用 checkWeather({"city": "北京"})
结果：{"city": "北京", "weather": "晴", "temperature": "25°C"}
回复："北京今天天气晴朗，温度 25 摄氏度。"
```

### 6.3 CSDN 发布工具

```java
@Tool(description = "发布文章到 CSDN")
public SaveArticleResponse saveArticle(
    @P("文章标题") String title,
    @P("文章内容") String content,
    @P("文章分类") String category
) {
    // 调用 CSDN API
    // 返回发布结果
}
```

**使用示例**：
```
用户："帮我写一篇关于 AI Agent 的文章并发布到 CSDN"
LLM：
  1. 生成文章内容
  2. 调用 saveArticle({"title": "...", "content": "...", "category": "人工智能"})
结果：{"articleId": "123456", "url": "https://blog.csdn.net/xxx/article/details/123456"}
回复："文章已成功发布到 CSDN：https://blog.csdn.net/xxx/article/details/123456"
```

### 6.4 企业微信工具

```java
@Tool(description = "发送企业微信文本消息")
public String sendText(
    @P("接收者，多个用|分隔") String toUser,
    @P("消息内容") String content
) {
    // 调用企业微信 API
    // 返回发送结果
}

@Tool(description = "发送企业微信卡片消息")
public String sendTextCard(
    @P("接收者") String toUser,
    @P("标题") String title,
    @P("描述") String description,
    @P("跳转链接") String url
) {
    // 调用企业微信 API
    // 返回发送结果
}
```

**使用示例**：
```
用户："通知团队成员今天下午3点开会"
LLM：调用 sendText({"toUser": "user1|user2|user3", "content": "今天下午3点开会"})
结果："消息已发送"
回复："已通知团队成员：user1, user2, user3"
```

---

## 七、业务场景实战

### 场景 1：智能客服

**需求**：
- 客户咨询产品问题
- 查询知识库（RAG）
- 查询订单状态（MCP）

**配置**：
```java
// 同时启用 RAG 和 MCP
SyncMcpToolCallbackProvider toolCallbacks = augmentService.augmentMcpTool(
    Arrays.asList("order_query", "inventory_check")
);

// 增强提示词
String prompt = augmentRagMessage(userMessage, "product_knowledge");

// 调用 LLM
String response = chatClient.prompt()
    .messages(prompt)
    .toolCallbacks(toolCallbacks)
    .call()
    .content();
```

**对话示例**：
```
用户："我的订单什么时候发货？"
LLM：[RAG 检索] [调用 MCP 查询订单]
     "您的订单 #12345 将在明天发货，预计后天送达。"
```

### 场景 2：个人助理

**需求**：
- 查询天气
- 发送提醒
- 安排日程

**配置**：
- mcp-server-amap（天气）
- mcp-server-wecom（消息）
- mcp-server-calendar（日程）

**对话示例**：
```
用户："帮我查一下明天的天气，如果下雨就提醒我带伞"
LLM：[调用 checkWeather] [调用 sendText]
     "明天有雨，已发送提醒：记得带伞出门"
```

### 场景 3：内容创作助手

**需求**：
- 搜索资料
- 撰写文章
- 发布平台

**配置**：
- mcp-server-bocha（搜索）
- mcp-server-csdn（发布）

**对话示例**：
```
用户："写一篇关于 AI Agent 的文章并发布到 CSDN"
LLM：[调用 webSearch 搜索资料]
     [生成文章内容]
     [调用 saveArticle 发布]
     "文章已发布：https://blog.csdn.net/xxx/article/details/123456"
```

---

## 八、最佳实践

### 8.1 工具设计原则

| 原则 | 说明 | 示例 |
|-----|------|------|
| **单一职责** | 每个工具只做一件事 | checkWeather 只查天气，不发通知 |
| **明确命名** | 工具名要清晰表达功能 | saveArticle 而非 execute |
| **详细描述** | description 要详细 | "查询指定城市的当前天气" 而非 "天气" |
| **参数校验** | 校验参数合法性 | city 不能为空 |
| **错误处理** | 返回明确的错误信息 | {"error": "未找到该城市"} |
| **幂等性** | 相同参数返回相同结果 | 查询操作天然幂等 |

### 8.2 工具描述优化

#### 差的描述

```java
@Tool(description = "天气")
public WeatherResponse checkWeather(String city) {
    // ...
}
```

**问题**：
- 描述太简单
- 没有说明参数格式
- LLM 可能误用

#### 好的描述

```java
@Tool(description = "根据城市名称查询当前天气，支持中国所有城市")
public WeatherResponse checkWeather(
    @P("城市名称，如：北京、上海、广州") String city
) {
    // ...
}
```

**优势**：
- 清楚说明功能
- 指定参数格式
- 给出示例

### 8.3 错误处理

```java
@Tool(description = "查询订单状态")
public OrderResponse queryOrder(
    @P("订单号，如：ORD123456") String orderId
) {
    try {
        // 1. 参数校验
        if (StringUtils.isEmpty(orderId)) {
            return OrderResponse.builder()
                .error("订单号不能为空")
                .build();
        }

        // 2. 业务逻辑
        Order order = orderService.getByOrderId(orderId);
        if (order == null) {
            return OrderResponse.builder()
                .error("订单不存在: " + orderId)
                .build();
        }

        // 3. 返回结果
        return OrderResponse.builder()
            .orderId(order.getOrderId())
            .status(order.getStatus())
            .build();

    } catch (Exception e) {
        // 4. 异常处理
        return OrderResponse.builder()
            .error("查询失败: " + e.getMessage())
            .build();
    }
}
```

### 8.4 性能优化

| 优化点 | 实现方式 | 效果 |
|-------|---------|------|
| **连接复用** | McpSyncClient 缓存 | 减少连接建立开销 |
| **超时控制** | requestTimeout | 防止长时间等待 |
| **并行调用** | 多线程并行执行 | 提升响应速度 |
| **结果缓存** | Redis 缓存 | 减少重复调用 |

---

## 九、自测清单

### 理解自测（共 10 题）

#### 基础概念

**Q1**: 什么是 MCP？它解决了什么问题？

<details>
<summary>点击查看答案</summary>

**答案**：
- **定义**：Model Context Protocol（模型上下文协议），由 Anthropic 提出
- **解决问题**：
  1. LLM 只能"说"不能"做"，MCP 赋予 LLM 调用工具的能力
  2. 不同工具的调用方式各异，MCP 统一了调用接口
  3. 工具集成困难，MCP 提供了标准化方案

**核心价值**：
- 统一接口：工具定义、调用、返回格式统一
- 可扩展：轻松添加新工具
- 解耦：LLM 与工具实现解耦

</details>

---

**Q2**: SSE 和 STDIO 两种通信方式有什么区别？

<details>
<summary>点击查看答案</summary>

**答案**：

| 维度 | SSE | STDIO |
|-----|-----|-------|
| **全称** | Server-Sent Events | Standard Input/Output |
| **适用场景** | 远程部署的 MCP 服务 | 本地进程的 MCP 服务 |
| **通信方式** | HTTP 长连接 | 标准输入输出 |
| **实时性** | 高（服务端可主动推送） | 中（需要轮询） |
| **复杂度** | 中（需要 HTTP 服务器） | 低（进程间通信） |
| **独立性** | 可独立部署 | 依赖主进程 |
| **示例** | Docker 容器中的服务 | Python 脚本工具 |

**选择建议**：
- 工具需要远程部署 → SSE
- 工具是本地脚本 → STDIO
- 需要服务端推送 → SSE
- 追求简单 → STDIO

</details>

---

#### 实现原理

**Q3**: 描述 MCP 工具调用的完整流程。

<details>
<summary>点击查看答案</summary>

**答案**：
```
1. 用户输入问题
2. LLM 理解意图，判断是否需要调用工具
3. LLM 选择合适的工具（根据工具描述）
4. LLM 生成工具调用请求（JSON 格式）
5. MCP Client 将请求发送给 MCP Server
6. MCP Server 执行工具逻辑
7. MCP Server 返回执行结果
8. LLM 基于工具结果生成最终回复
9. 返回给用户
```

**关键点**：
- LLM 自动决策是否调用工具
- LLM 自动选择合适的工具
- LLM 自动解析工具返回结果

</details>

---

**Q4**: @Tool 和 @P 注解的作用是什么？

<details>
<summary>点击查看答案</summary>

**答案**：
**@Tool 注解**：
- 作用：标注一个方法为 MCP 工具
- 参数：
  - `description`：工具描述，LLM 根据描述选择是否调用
  - `name`：工具名称，默认使用方法名
- 示例：
```java
@Tool(description = "查询指定城市的天气")
public WeatherResponse checkWeather(String city) {
    // ...
}
```

**@P 注解**：
- 作用：标注工具参数的描述
- 参数：参数描述，LLM 根据描述生成参数值
- 示例：
```java
public WeatherResponse checkWeather(
    @P("城市名称，如：北京、上海") String city
) {
    // ...
}
```

</details>

---

**Q5**: 如何将 MCP 工具挂载到 ChatClient？

<details>
<summary>点击查看答案</summary>

**答案**：
```java
// 1. 创建 McpSyncClient
HttpClientSseClientTransport transport = HttpClientSseClientTransport
    .builder("http://localhost:9001")
    .sseEndpoint("/mcp/sse")
    .build();

McpSyncClient mcpClient = McpClient.sync(transport).build();
mcpClient.initialize();  // 获取工具列表

// 2. 封装为 ToolCallbackProvider
SyncMcpToolCallbackProvider toolCallbacks =
    new SyncMcpToolCallbackProvider(Arrays.asList(mcpClient));

// 3. 挂载到 ChatClient
String response = chatClient.prompt()
    .user("帮我查一下北京天气")
    .toolCallbacks(toolCallbacks)  // 挂载工具
    .call()
    .content();
```

</details>

---

#### 架构设计

**Q6**: 为什么需要 MCP，直接调用 API 不行吗？

<details>
<summary>点击查看答案</summary>

**答案**：
**直接调用 API 的问题**：
1. **LLM 不知道如何调用**：
   - 不同 API 的认证方式、请求格式不同
   - LLM 难以生成正确的调用代码
2. **难以扩展**：
   - 每新增一个 API 都要修改代码
   - 无法动态添加工具
3. **错误处理复杂**：
   - 需要处理各种异常情况
   - 重试、超时等逻辑分散

**MCP 的优势**：
1. **标准化接口**：
   - 统一的工具定义格式
   - 统一的调用协议
2. **可扩展**：
   - 动态挂载工具
   - 配置化管理
3. **LLM 友好**：
   - 工具描述清晰
   - LLM 自动选择和调用
4. **解耦**：
   - LLM 与工具实现解耦
   - 工具可独立开发和部署

</details>

---

**Q7**: 如何实现工具的并行调用？

<details>
<summary>点击查看答案</summary>

**答案**：
**方案 1：LLM 自动并行**
- 某些 LLM（如 Claude 3.5）支持自动并行调用
- 当工具间无依赖时，LLM 会同时发起调用

**方案 2：手动并行**
```java
// 提交并行任务
CompletableFuture<WeatherResponse> weatherFuture =
    CompletableFuture.supplyAsync(() -> weatherTool.checkWeather("北京"));

CompletableFuture<OrderResponse> orderFuture =
    CompletableFuture.supplyAsync(() -> orderTool.queryOrder("ORD123"));

// 等待所有结果
CompletableFuture.allOf(weatherFuture, orderFuture).join();

// 获取结果
WeatherResponse weather = weatherFuture.get();
OrderResponse order = orderFuture.get();
```

**方案 3：使用 Reactive Streams**
```java
Flux.just(
    weatherTool.checkWeather("北京"),
    orderTool.queryOrder("ORD123")
).parallel().run(Schedulers.parallel())
 .subscribe();
```

</details>

---

**Q8**: MCP 工具调用失败如何处理？

<details>
<summary>点击查看答案</summary>

**答案**：
**失败场景**：
1. 工具不存在
2. 参数错误
3. 工具执行超时
4. 工具执行异常

**处理策略**：

| 场景 | 处理方式 |
|-----|---------|
| **工具不存在** | LLM 自动选择其他工具或直接回答 |
| **参数错误** | 返回错误信息，LLM 重新生成参数 |
| **执行超时** | 设置合理的超时时间，重试或降级 |
| **执行异常** | 工具返回错误信息，LLM 基于错误回答 |

**代码示例**：
```java
@Tool(description = "查询订单")
public OrderResponse queryOrder(@P("订单号") String orderId) {
    try {
        Order order = orderService.getByOrderId(orderId);
        if (order == null) {
            return OrderResponse.builder()
                .error("订单不存在: " + orderId)  // 明确错误
                .build();
        }
        return OrderResponse.fromEntity(order);
    } catch (Exception e) {
        return OrderResponse.builder()
            .error("查询失败: " + e.getMessage())  // 返回错误
            .build();
    }
}
```

**LLM 处理**：
```
LLM 收到：{"error": "订单不存在: ORD123"}
LLM 回复："抱歉，订单 ORD123 不存在，请检查订单号是否正确。"
```

</details>

---

**Q9**: 如何保护 MCP 工具的安全性？

<details>
<summary>点击查看答案</summary>

**答案**：
**安全措施**：

1. **认证授权**：
```java
// MCP Server 添加认证
@PostMapping("/mcp/sse")
public SseEmitter connect(@RequestHeader("Authorization") String token) {
    // 验证 token
    if (!authService.validate(token)) {
        throw new UnauthorizedException();
    }
    // ...
}
```

2. **参数校验**：
```java
@Tool(description = "删除订单")
public String deleteOrder(@P("订单号") String orderId) {
    // 校验权限
    if (!hasPermission("delete:order")) {
        return "无权操作";
    }
    // ...
}
```

3. **敏感操作确认**：
```java
@Tool(description = "删除订单（危险操作，需要用户确认）")
public String deleteOrder(@P("订单号") String orderId) {
    // 返回确认信息
    return "请确认是否删除订单 " + orderId;
}
```

4. **审计日志**：
```java
@Tool(description = "查询用户信息")
public UserResponse getUser(@P("用户ID") String userId) {
    // 记录审计日志
    auditService.log("getUser", userId);
    // ...
}
```

5. **限流**：
```java
@Tool(description = "发送短信")
public String sendSms(@P("手机号") String phone) {
    // 限流检查
    if (rateLimiter.exceedLimit(phone)) {
        return "发送过于频繁，请稍后再试";
    }
    // ...
}
```

</details>

---

**Q10**: 如何优化 MCP 工具的调用性能？

<details>
<summary>点击查看答案</summary>

**答案**：
**优化方向**：

1. **连接复用**：
```java
// 缓存 McpSyncClient
private final Map<String, McpSyncClient> clientCache = new ConcurrentHashMap<>();

public McpSyncClient getClient(String mcpId) {
    return clientCache.computeIfAbsent(mcpId, id -> createClient(id));
}
```

2. **超时控制**：
```java
McpSyncClient client = McpClient.sync(transport)
    .requestTimeout(Duration.ofSeconds(30))  // 设置超时
    .build();
```

3. **结果缓存**：
```java
@Cacheable(key = "#city", expire = 600)
public WeatherResponse checkWeather(String city) {
    // 缓存 10 分钟
}
```

4. **批量调用**：
```java
@Tool(description = "批量查询订单")
public List<OrderResponse> queryOrders(@P("订单号列表") List<String> orderIds) {
    // 批量查询，减少网络开销
}
```

5. **异步执行**：
```java
@Async
@Tool(description = "发送邮件")
public CompletableFuture<String> sendEmailAsync(@P("收件人") String to) {
    // 异步执行，不阻塞 LLM
    return CompletableFuture.completedFuture("邮件已发送");
}
```

</details>

---

### 实战自测（共 3 题）

**Q11**: 开发一个"查询快递物流"的 MCP 工具。

<details>
<summary>点击查看答案</summary>

**答案**：
```java
@Component
public class LogisticsTool {

    @Resource
    private RestTemplate restTemplate;

    @Value("{logistics.api.key}")
    private String apiKey;

    @Tool(description = "根据快递单号查询物流信息")
    public LogisticsResponse queryLogistics(
        @P("快递单号，如：SF1234567890") String trackingNumber
    ) {
        try {
            // 1. 参数校验
            if (StringUtils.isEmpty(trackingNumber)) {
                return LogisticsResponse.builder()
                    .error("快递单号不能为空")
                    .build();
            }

            // 2. 调用物流 API
            String url = String.format(
                "https://api.logistics.com/query?key=%s&number=%s",
                apiKey, trackingNumber
            );

            String response = restTemplate.getForObject(url, String.class);

            // 3. 解析响应
            JSONObject json = JSON.parseObject(response);
            if (!json.getBoolean("success")) {
                return LogisticsResponse.builder()
                    .error("查询失败: " + json.getString("message"))
                    .build();
            }

            // 4. 构建物流轨迹
            List<Trace> traces = json.getJSONArray("traces")
                .stream()
                .map(obj -> {
                    JSONObject trace = (JSONObject) obj;
                    return Trace.builder()
                        .time(trace.getString("time"))
                        .location(trace.getString("location"))
                        .status(trace.getString("status"))
                        .build();
                })
                .collect(Collectors.toList());

            // 5. 返回结果
            return LogisticsResponse.builder()
                .trackingNumber(trackingNumber)
                .currentStatus(json.getString("status"))
                .estimatedArrival(json.getString("estimatedArrival"))
                .traces(traces)
                .build();

        } catch (Exception e) {
            return LogisticsResponse.builder()
                .error("查询异常: " + e.getMessage())
                .build();
        }
    }

    @Data
    @Builder
    static class LogisticsResponse {
        private String trackingNumber;
        private String currentStatus;
        private String estimatedArrival;
        private List<Trace> traces;
        private String error;
    }

    @Data
    @Builder
    static class Trace {
        private String time;
        private String location;
        private String status;
    }
}
```

**使用示例**：
```
用户："我的快递 SF1234567890 到哪了？"
LLM：调用 queryLogistics({"trackingNumber": "SF1234567890"})
结果：{
  "trackingNumber": "SF1234567890",
  "currentStatus": "派送中",
  "traces": [
    {"time": "2024-01-15 09:00", "location": "北京市朝阳区", "status": "派送中"},
    {"time": "2024-01-15 06:00", "location": "北京市朝阳区营业点", "status": "到达营业点"},
    {"time": "2024-01-14 22:00", "location": "北京转运中心", "status": "到达转运中心"}
  ]
}
回复："您的快递 SF1234567890 正在派送中，预计今天送达。

最新轨迹：
- 09:00 北京市朝阳区 派送中
- 06:00 北京市朝阳区营业点 到达营业点
- 昨天 22:00 北京转运中心 到达转运中心"
```

</details>

---

**Q12**: 用户反馈 MCP 工具调用失败率高，如何排查？

<details>
<summary>点击查看答案</summary>

**答案**：
**排查步骤**：

1. **检查 MCP Server 是否正常运行**
```bash
curl http://localhost:9001/health
```

2. **检查网络连接**
```bash
# 测试 SSE 连接
curl -N http://localhost:9001/mcp/sse
```

3. **查看日志**
```java
// MCP Server 添加详细日志
@Slf4j
@Component
public class WeatherTool {
    @Tool(description = "查询天气")
    public WeatherResponse checkWeather(String city) {
        log.info("调用 checkWeather, city: {}", city);
        try {
            // ...
            log.info("查询成功: {}", result);
            return result;
        } catch (Exception e) {
            log.error("查询失败", e);
            return WeatherResponse.builder()
                .error("查询失败: " + e.getMessage())
                .build();
        }
    }
}
```

4. **检查超时配置**
```java
// 增加超时时间
McpSyncClient client = McpClient.sync(transport)
    .requestTimeout(Duration.ofMinutes(2))  // 2 分钟
    .build();
```

5. **监控和告警**
```java
// 记录调用统计
@Component
public class McpMetrics {
    private final AtomicLong successCount = new AtomicLong(0);
    private final AtomicLong failureCount = new AtomicLong(0);

    public void recordSuccess() {
        successCount.incrementAndGet();
    }

    public void recordFailure() {
        failureCount.incrementAndGet();
    }

    public double getFailureRate() {
        long total = successCount.get() + failureCount.get();
        return total == 0 ? 0 : (double) failureCount.get() / total;
    }
}
```

**常见问题**：

| 问题 | 原因 | 解决方案 |
|-----|------|---------|
| 连接超时 | 网络问题或服务未启动 | 检查网络和服务状态 |
| 调用失败 | 参数错误或 API 变化 | 检查参数和 API 文档 |
| 返回空 | 权限问题或数据不存在 | 检查权限和数据 |
| 频繁失败 | 限流或配额不足 | 联系 API 提供商 |

</details>

---

**Q13**: 如何实现 MCP 工具的版本管理和灰度发布？

<details>
<summary>点击查看答案</summary>

**答案**：
**方案 1：多版本共存**
```java
// v1 版本
@Component("weatherToolV1")
public class WeatherToolV1 {
    @Tool(description = "查询天气（v1，支持中国城市）")
    public WeatherResponse checkWeather(String city) {
        // v1 实现
    }
}

// v2 版本
@Component("weatherToolV2")
public class WeatherToolV2 {
    @Tool(description = "查询天气（v2，支持全球城市）")
    public WeatherResponse checkWeather(String city, String country) {
        // v2 实现
    }
}
```

**方案 2：灰度发布**
```java
@Service
public class McpRouter {

    @Value("${mcp.weather.version: v1}")
    private String weatherVersion;

    public McpSyncClient getWeatherClient() {
        if ("v2".equals(weatherVersion)) {
            return mcpClientV2;
        }
        return mcpClientV1;
    }
}
```

**方案 3：A/B 测试**
```java
@Service
public class McpABTest {

    public McpSyncClient getWeatherClient(String userId) {
        // 10% 用户使用 v2
        if (hash(userId) % 10 == 0) {
            return mcpClientV2;
        }
        return mcpClientV1;
    }
}
```

**方案 4：配置化管理**
```sql
-- ai_mcp 表添加版本字段
ALTER TABLE ai_mcp ADD COLUMN version VARCHAR(10) DEFAULT 'v1';
ALTER TABLE ai_mcp ADD COLUMN gray_percentage INT DEFAULT 0;

-- 配置灰度
UPDATE ai_mcp SET version = 'v2', gray_percentage = 10 WHERE mcp_id = 'amap';
```

```java
// 灰度路由
public McpSyncClient getMcpClient(String mcpId, String userId) {
    McpConfig config = getMcpConfig(mcpId);

    // 判断是否在灰度范围
    if (isGrayUser(userId, config.getGrayPercentage())) {
        return getMcpClient(mcpId + "_v2");
    }

    return getMcpClient(mcpId + "_v1");
}
```

</details>

---

## 十、关键代码索引

| 功能 | 文件 | 位置 |
|-----|------|------|
| MCP 工具挂载 | AugmentService.augmentMcpTool() | backend/ai-agent-domain/src/main/java/com/dasi/domain/augment/service/ |
| 高德天气工具 | AmapTool.java | mcp/mcp-server-amap/src/main/java/com/dasi/mcp/amap/tool/ |
| CSDN 发布工具 | CsdnTool.java | mcp/mcp-server-csdn/src/main/java/com/dasi/mcp/csdn/tool/ |
| 企业微信工具 | WecomTool.java | mcp/mcp-server-wecom/src/main/java/com/dasi/mcp/wecom/tool/ |
| 邮件工具 | EmailTool.java | mcp/mcp-server-email/src/main/java/com/dasi/mcp/email/tool/ |
| 搜索工具 | BochaTool.java | mcp/mcp-server-bocha/src/main/java/com/dasi/mcp/bocha/tool/ |

---

**文档版本**：v1.0
**更新日期**：2026-02-24
**下一篇**：专题五 - 配置管理系统深度解析
