# MCP、Agent、记忆模块、Prompt 开发实现指南

## 文档说明

本指南基于 Dasi Agent 项目，提供在工作中开发 MCP 服务、Agent 工作流、记忆模块和 Prompt 的实战指导。每个模块都会提供详细的实现步骤、代码示例和最佳实践。

---

## 一、MCP 服务开发指南

### 1.1 MCP 简介

**MCP（Model Context Protocol）** 是 AI 模型与外部工具交互的标准协议。通过 MCP，可以让 AI Agent 调用外部服务（如发送邮件、查询天气、发布文章等）。

### 1.2 创建 MCP 服务的完整步骤

#### 步骤 1：初始化项目

创建一个新的 Spring Boot 项目：

```bash
# 项目结构
mcp-server-custom/
├── src/main/java/com/dasi/mcp/
│   ├── McpServerCustomApplication.java
│   ├── tool/
│   │   └── CustomTool.java
│   ├── dto/
│   │   ├── CustomToolRequest.java
│   │   └── CustomToolResponse.java
│   ├── port/
│   │   └── ICustomPort.java
│   └── properties/
│       └── CustomProperties.java
├── src/main/resources/
│   └── application.yml
└── pom.xml
```

**pom.xml 依赖**：
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>com.alibaba.fastjson2</groupId>
        <artifactId>fastjson2</artifactId>
        <version>2.0.43</version>
    </dependency>
</dependencies>
```

#### 步骤 2：定义 DTO（数据传输对象）

**请求 DTO**：
```java
package com.dasi.mcp.dto;

import lombok.Data;

@Data
public class CustomToolRequest {
    /**
     * 工具名称
     */
    private String name;

    /**
     * 工具参数（根据实际需求定义）
     */
    private String param1;

    private Integer param2;
}
```

**响应 DTO**：
```java
package com.dasi.mcp.dto;

import lombok.Data;

@Data
public class CustomToolResponse {
    /**
     * 是否成功
     */
    private Boolean success;

    /**
     * 返回消息
     */
    private String message;

    /**
     * 返回数据
     */
    private Object data;

    public static CustomToolResponse success(String message, Object data) {
        CustomToolResponse response = new CustomToolResponse();
        response.setSuccess(true);
        response.setMessage(message);
        response.setData(data);
        return response;
    }

    public static CustomToolResponse error(String message) {
        CustomToolResponse response = new CustomToolResponse();
        response.setSuccess(false);
        response.setMessage(message);
        return response;
    }
}
```

#### 步骤 3：实现 Tool（工具类）

```java
package com.dasi.mcp.tool;

import com.dasi.mcp.dto.CustomToolRequest;
import com.dasi.mcp.dto.CustomToolResponse;
import com.dasi.mcp.port.ICustomPort;
import jakarta.annotation.Resource;
import lombok.extern.slf4j.Slf4j;
import org.springframework.ai.tool.annotation.Tool;
import org.springframework.stereotype.Component;

/**
 * 自定义 MCP 工具
 *
 * @Tool 注解说明：
 * - description: 工具描述，AI 会根据此描述决定是否调用该工具
 */
@Slf4j
@Component
public class CustomTool {

    @Resource
    private ICustomPort customPort;

    /**
     * 自定义工具方法
     *
     * @param request 工具请求参数
     * @return 工具执行结果
     */
    @Tool(description = "执行自定义操作的工具，用于...")
    public CustomToolResponse executeCustom(CustomToolRequest request) {
        log.info("调用 MCP 工具：name={}, param1={}, param2={}",
                 request.getName(), request.getParam1(), request.getParam2());

        try {
            // 调用端口层执行实际业务逻辑
            return customPort.doSomething(request);
        } catch (Exception e) {
            log.error("MCP 工具执行失败", e);
            return CustomToolResponse.error("执行失败：" + e.getMessage());
        }
    }
}
```

#### 步骤 4：实现 Port（端口层）

```java
package com.dasi.mcp.port;

import com.dasi.mcp.dto.CustomToolRequest;
import com.dasi.mcp.dto.CustomToolResponse;

public interface ICustomPort {
    /**
     * 执行具体业务逻辑
     */
    CustomToolResponse doSomething(CustomToolRequest request);
}
```

```java
package com.dasi.mcp.port;

import com.dasi.mcp.dto.CustomToolRequest;
import com.dasi.mcp.dto.CustomToolResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Slf4j
@Service
public class CustomPort implements ICustomPort {

    @Override
    public CustomToolResponse doSomething(CustomToolRequest request) {
        // 示例：调用外部 API
        // RestTemplate restTemplate = new RestTemplate();
        // String result = restTemplate.postForObject(url, request, String.class);

        // 示例：执行业务逻辑
        String result = "处理结果：" + request.getParam1();

        return CustomToolResponse.success("执行成功", result);
    }
}
```

#### 步骤 5：配置应用

**application.yml**：
```yaml
server:
  port: 9006  # 避免与其他 MCP 服务冲突

spring:
  application:
    name: mcp-server-custom

# 自定义配置
custom:
  config:
    api-url: https://api.example.com
    timeout: 5000
```

**配置类**：
```java
package com.dasi.mcp.properties;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Data
@Component
@ConfigurationProperties(prefix = "custom.config")
public class CustomProperties {
    private String apiUrl;
    private Integer timeout;
}
```

#### 步骤 6：启动类

```java
package com.dasi;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class McpServerCustomApplication {
    public static void main(String[] args) {
        SpringApplication.run(McpServerCustomApplication.class, args);
    }
}
```

#### 步骤 7：打包部署

**Dockerfile**：
```dockerfile
FROM openjdk:17-slim
WORKDIR /app
COPY target/mcp-server-custom.jar /app/app.jar
EXPOSE 9006
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

**构建脚本**：
```bash
#!/bin/bash
cd mcp-server-custom
mvn clean package -DskipTests
docker build -t mcp-server-custom:latest .
docker run -d -p 9006:9006 --name mcp-custom mcp-server-custom:latest
```

### 1.3 在主系统中注册 MCP 服务

#### 方式 1：通过管理后台注册

1. 登录管理后台
2. 进入"服务管理" → "MCP 管理"
3. 点击"新增"
4. 填写配置：
   - **MCP ID**：mcp_custom
   - **MCP 名称**：自定义服务
   - **MCP 类型**：sse
   - **SSE 配置**：
     - Base URI：http://localhost:9006
     - SSE EndPoint：/mcp/sse
   - **超时时间**：5

#### 方式 2：直接插入数据库

```sql
INSERT INTO ai_mcp (
    mcp_id,
    mcp_name,
    mcp_type,
    sse_config,
    mcp_timeout,
    status
) VALUES (
    'mcp_custom',
    '自定义服务',
    'sse',
    '{"baseUri": "http://localhost:9006", "sseEndPoint": "/mcp/sse"}',
    5,
    1
);
```

### 1.4 测试 MCP 服务

#### 单元测试

```java
@SpringBootTest
class CustomToolTest {

    @Resource
    private CustomTool customTool;

    @Test
    void testExecute() {
        CustomToolRequest request = new CustomToolRequest();
        request.setName("测试");
        request.setParam1("参数1");
        request.setParam2(100);

        CustomToolResponse response = customTool.executeCustom(request);
        assertTrue(response.getSuccess());
        System.out.println(response.getMessage());
    }
}
```

#### 集成测试（通过 AI 对话）

1. 在前端对话界面
2. 选择 Client（如 doubao）
3. 选择 MCP（勾选刚添加的 custom）
4. 输入问题："请调用自定义工具帮我处理..."
5. 观察 AI 是否正确调用了 MCP 工具

---

## 二、Agent 工作流开发指南

### 2.1 Agent 架构理解

**Agent** 是一个能够自主执行任务的智能体，由多个 **Client**（角色）组成。项目支持两种 Agent 类型：

#### Loop 类型（循环模式）

适用于需要多轮迭代、监督评估的复杂任务：

```
Analyzer（分析任务）
    ↓
Performer（执行任务）
    ↓
Supervisor（监督评估）
    ↓ 通过？
Summarizer（总结）
    ↓
完成
```

**适用场景**：
- 联网搜索并总结
- 数据分析并生成报告
- 需要多轮优化的任务

#### Step 类型（步骤模式）

适用于线性步骤、需要重试的任务：

```
Inspector（检查任务）
    ↓
Planner（规划步骤）
    ↓
Runner（执行步骤，支持重试）
    ↓
Replier（回复用户）
```

**适用场景**：
- 文章生成与发布
- 天气查询与播报
- 步骤明确的任务

### 2.2 创建 Agent 的步骤

#### 步骤 1：规划 Agent 功能

明确以下问题：
1. Agent 要解决什么问题？
2. 需要哪些角色（Client）？
3. 使用 Loop 还是 Step 策略？
4. 需要调用哪些 MCP 工具？

**示例：天气播报 Agent**
- 问题：查询天气并播报
- 角色：Inspector（检查）、Planner（规划）、Runner（执行）、Replier（回复）
- 策略：Step
- MCP：高德天气 API

#### 步骤 2：创建 Client（角色）

每个 Client 代表 Agent 中的一个角色，需要配置：
- API：使用的 AI 模型
- Model：具体模型
- MCP：需要调用的工具
- Advisor：工作内存
- Prompt：提示词

**通过管理后台创建 Client**：

1. 进入"服务管理" → "CLIENT 管理"
2. 点击"新增"
3. 填写配置：
   - **Client ID**：client_runner_weather
   - **Client 名称**：天气执行者
   - **Client 角色**：runner
   - **Client 类型**：work
   - **API**：选择已有 API（如 doubao）
   - **Model**：选择已有 Model（如 Doubao-Seed-1.8）
   - **MCP**：选择 amap（高德天气）
   - **Advisor**：选择 advisor_work_memory

4. 重复创建其他角色 Client：
   - client_inspector_weather（检查者）
   - client_planner_weather（规划者）
   - client_replier_weather（回复者）

#### 步骤 3：编写 Prompt

为每个角色编写对应的 Prompt。

**client_runner_weather.txt**（执行者）：
```txt
角色：天气查询执行专家

职责：根据规划者的输出，调用高德天气 API 查询指定地址的天气。

可用工具：
- checkWeather：查询天气，参数 address（必填）

执行规则：
1. 从规划者输出中解析出要查询的地址
2. 调用 checkWeather 工具查询天气
3. 解析返回的天气数据
4. 返回格式化的天气信息

输出格式（必须严格遵守 JSON）：
{
    "runner_target": "查询广州的今日天气",
    "runner_process": "调用 checkWeather 工具，参数 address='广州'",
    "runner_result": "广州今日天气：多云，气温 15-26℃，北风 1-3级"
}
```

**client_replier_weather.txt**（回复者）：
```txt
角色：天气播报回复专家

职责：根据执行者的结果，生成自然、友好的天气播报。

输入：
- 用户原始问题
- 执行者的查询结果

回复规则：
1. 使用自然、口语化的语言
2. 突出关键信息（天气状况、温度、风力）
3. 提供出行建议
4. 语气友好、简洁

输出格式：
直接输出最终的播报文本，无需 JSON 格式。
```

**存储位置**：
- 系统提示词：`backend/ai-agent-app/src/main/resources/prompt/system-prompt/`
- 用户提示词：`backend/ai-agent-app/src/main/resources/prompt/user-prompt/`

#### 步骤 4：创建 Flow（工作流）

1. 进入"工作流管理" → "FLOW 管理"
2. 点击"新增 Agent"
3. 填写配置：
   - **Agent ID**：agent_weather
   - **Agent 名称**：天气播报智能体
   - **Agent 类型**：step
   - **Agent 描述**：查询指定地址的天气并进行播报
   - **状态**：启用

4. 配置 Flow（为每个角色分配 Client）：
   点击"查看配置图"，按顺序添加：
   - inspector → client_inspector_weather
   - planner → client_planner_weather
   - runner → client_runner_weather
   - replier → client_replier_weather

#### 步骤 5：测试 Agent

1. 在前端"Work 会话"中新建会话
2. 选择 Agent：天气播报智能体
3. 输入问题："帮我查询广州今天的天气"
4. 观察执行流程：
   - Inspector 检查任务
   - Planner 规划步骤
   - Runner 调用高德 API 查询天气
   - Replier 生成播报文本
5. 查看最终结果

### 2.3 Agent 开发最佳实践

#### 实践 1：角色职责明确

每个 Client 只负责一件事：
- **Analyzer**：只分析需求，不做判断
- **Performer**：只执行任务，不做总结
- **Supervisor**：只评估结果，不做执行

#### 实践 2：Prompt 模板化

使用变量占位符，使 Prompt 可复用：

```txt
用户需求：{0}
分析结果：{1}
执行结果：{2}

请根据以上信息...
```

代码中使用：
```java
String prompt = flowPrompt.formatted(userMessage, analyzerResponse, performerResponse);
```

#### 实践 3：错误处理

在 Prompt 中明确错误处理规则：

```txt
如果出现错误：
1. 不允许编造数据
2. 在 result 字段中说明具体错误
3. 继续执行下一个节点
```

代码中捕获异常：
```java
try {
    String response = performerClient.prompt(performerPrompt).call().content();
    JSONObject result = parseJsonObject(response);
} catch (Exception e) {
    result = buildExceptionObject("执行失败", e.getMessage());
}
```

#### 实践 4：JSON 格式严格

强制 AI 输出严格的 JSON 格式：

```txt
输出格式约束：
- 必须且只能输出一个合法 JSON 对象
- JSON 前后不得出现任何字符、解释、Markdown、代码块
- 必须可被一次性 parse 成功
- 禁止尾逗号、单引号、注释、NaN/Infinity
```

---

## 三、记忆模块开发指南

### 3.1 ChatMemory 原理

**ChatMemory** 是 Spring AI 提供的会话记忆机制，用于在多轮对话中保持上下文。

**核心概念**：
- **conversationId**：会话唯一标识
- **retrieveSize**：保留的历史消息数量
- **存储介质**：Redis（分布式）或内存（单机）

### 3.2 使用 ChatMemory

#### 基本用法

```java
@Autowired
private ChatClient.Builder chatClientBuilder;

public String chat(String sessionId, String userMessage) {
    ChatClient chatClient = chatClientBuilder.build();

    return chatClient.prompt()
        .advisors(a -> a
            .param(CHAT_MEMORY_CONVERSATION_ID_KEY, sessionId)
            .param(CHAT_MEMORY_RETRIEVE_SIZE_KEY, 10))
        .messages(userMessage)
        .call()
        .content();
}
```

#### 参数说明

- **CHAT_MEMORY_CONVERSATION_ID_KEY**：
  - 值：`"chat_memory_conversation_id"`
  - 用途：标识会话，不同 sessionId 的记忆互不干扰

- **CHAT_MEMORY_RETRIEVE_SIZE_KEY**：
  - 值：`"chat_memory_retrieve_size"`
  - 用途：指定保留的历史消息数量
  - Chat 对话：建议 10 条
  - Work 对话：建议 50 条（因为对话更长）

### 3.3 ChatMemory 配置

#### Redis 配置

**application.yml**：
```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password: your_password
```

**RedisConfig.java**：
```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
}
```

#### ChatMemory Advisor 配置

Spring AI 会自动使用 Redis 存储 ChatMemory，无需额外配置。

### 3.4 自定义记忆管理

如果需要更灵活的记忆管理，可以实现自定义的记忆服务：

```java
@Service
public class CustomMemoryService {

    @Resource
    private RedisTemplate<String, Object> redisTemplate;

    private static final String MEMORY_KEY_PREFIX = "chat_memory:";
    private static final int MAX_MEMORY_SIZE = 50;

    /**
     * 添加消息到记忆
     */
    public void addMessage(String sessionId, Message message) {
        String key = MEMORY_KEY_PREFIX + sessionId;

        // 获取当前记忆
        List<Message> memory = getMemory(sessionId);

        // 添加新消息
        memory.add(message);

        // 限制记忆大小
        if (memory.size() > MAX_MEMORY_SIZE) {
            memory = memory.subList(memory.size() - MAX_MEMORY_SIZE, memory.size());
        }

        // 保存到 Redis
        redisTemplate.opsForValue().set(key, memory, Duration.ofHours(24));
    }

    /**
     * 获取记忆
     */
    public List<Message> getMemory(String sessionId) {
        String key = MEMORY_KEY_PREFIX + sessionId;
        Object memory = redisTemplate.opsForValue().get(key);
        return memory != null ? (List<Message>) memory : new ArrayList<>();
    }

    /**
     * 清除记忆
     */
    public void clearMemory(String sessionId) {
        String key = MEMORY_KEY_PREFIX + sessionId;
        redisTemplate.delete(key);
    }
}
```

### 3.5 记忆持久化到数据库

如果需要永久保存会话历史：

```java
@Service
public class MessageService {

    @Resource
    private IMessageDao messageDao;

    /**
     * 保存用户消息
     */
    public void saveUserMessage(String sessionId, String content) {
        Message message = new Message();
        message.setSessionId(sessionId);
        message.setRole("user");
        message.setContent(content);
        message.setCreateTime(new Date());
        messageDao.insert(message);
    }

    /**
     * 保存助手消息
     */
    public void saveAssistantMessage(String sessionId, String content) {
        Message message = new Message();
        message.setSessionId(sessionId);
        message.setRole("assistant");
        message.setContent(content);
        message.setCreateTime(new Date());
        messageDao.insert(message);
    }

    /**
     * 获取会话历史
     */
    public List<Message> getSessionHistory(String sessionId) {
        return messageDao.selectBySessionId(sessionId);
    }
}
```

### 3.6 最佳实践

#### 实践 1：区分会话类型

不同会话类型使用不同的记忆大小：

```java
public static final int CHAT_MEMORY_SIZE = 10;
public static final int WORK_MEMORY_SIZE = 50;

public int getRetrieveSize(String sessionType) {
    return "chat".equals(sessionType) ? CHAT_MEMORY_SIZE : WORK_MEMORY_SIZE;
}
```

#### 实践 2：会话过期策略

设置合理的过期时间：

```java
// Chat 会话：24 小时过期
redisTemplate.opsForValue().set(key, memory, Duration.ofHours(24));

// Work 会话：7 天过期
redisTemplate.opsForValue().set(key, memory, Duration.ofDays(7));
```

#### 实践 3：记忆压缩

对于长会话，可以实现记忆压缩：

```java
/**
 * 压缩记忆（保留关键信息）
 */
public List<Message> compressMemory(List<Message> memory) {
    // 保留最近 10 条消息
    List<Message> recent = memory.subList(memory.size() - 10, memory.size());

    // 总结前面的消息
    List<Message> old = memory.subList(0, memory.size() - 10);
    String summary = summarizeOldMessages(old);

    // 创建总结消息
    Message summaryMsg = new Message();
    summaryMsg.setRole("system");
    summaryMsg.setContent("[历史对话摘要] " + summary);

    // 返回压缩后的记忆
    List<Message> compressed = new ArrayList<>();
    compressed.add(summaryMsg);
    compressed.addAll(recent);
    return compressed;
}
```

---

## 四、Prompt 编写指南

### 4.1 Prompt 结构

一个完整的 Prompt 包含以下部分：

```txt
# 角色
你是...

# 职责
你的主要职责是...

# 可用工具/资源
- 工具1：...
- 工具2：...

# 规则
1. 规则1
2. 规则2

# 输出格式
必须按照以下格式输出：
...

# 示例
用户输入：...
输出：...
```

### 4.2 Prompt 编写原则

#### 原则 1：角色定位清晰

**不好**：
```txt
你是一个 AI 助手。
```

**好**：
```txt
角色：一名专业的 Performer 联网搜索任务的执行专家，具备调用 MCP 工具进行联网搜索的能力。
```

#### 原则 2：职责明确

**不好**：
```txt
你需要帮助用户完成任务。
```

**好**：
```txt
职责：基于提供的信息，根据用户需求和任务分析专家的输出，调用联网搜索工具，实际执行具体的任务。

输出字段解释：
- performer_target：本轮执行要达成的具体目标。
- performer_process：实际执行了哪些关键步骤/调用了哪些工具与参数。
- performer_result：执行得到的关键结果/数据/结论。
```

#### 原则 3：规则具体

**不好**：
```txt
输出 JSON 格式。
```

**好**：
```txt
输出格式约束：
- 禁止把 MCP 的方法参数序列化成字符串，应该传递 JSON
- 必须且只能输出一个合法 JSON 对象：以 { 开头、以 } 结尾，JSON 前后不得出现任何字符/解释/Markdown/代码块
- JSON 必须可被一次性 parse 成功：禁止尾逗号、单引号、注释、NaN/Infinity、未闭合引号/括号
- 必须严格按下述 schema 返回：字段名、层级、数量完全一致；不得新增字段、不得删除字段、不得改字段名
```

#### 原则 4：提供示例

**不好**：
```txt
查询天气并返回结果。
```

**好**：
```txt
示例：

用户输入：帮我查询广州今天的天气
执行步骤：
1. 调用 checkWeather 工具，参数 address="广州"
2. 解析返回结果
3. 格式化输出

输出：
{
    "runner_target": "查询广州的今日天气",
    "runner_process": "调用 checkWeather 工具，参数 address='广州'",
    "runner_result": "广州今日天气：多云，气温 15-26℃，北风 1-3级"
}
```

### 4.3 常用 Prompt 模板

#### Analyzer（分析者）模板

```txt
角色：任务分析专家

职责：分析用户需求，制定执行计划

分析要点：
1. 用户想要达成什么目标
2. 需要调用哪些工具
3. 执行的先后顺序
4. 可能遇到的问题

输出字段：
- analyzer_target：用户目标
- analyzer_tools：需要的工具列表
- analyzer_plan：执行计划

输出格式（严格遵守 JSON）：
{
    "analyzer_target": "",
    "analyzer_tools": [],
    "analyzer_plan": ""
}
```

#### Performer（执行者）模板

```txt
角色：任务执行专家

职责：根据分析结果，调用工具执行具体任务

可用工具：
- 工具1：描述，参数：param1, param2
- 工具2：描述，参数：param3

执行规则：
1. 必须实际调用工具，不能编造结果
2. 如果工具调用失败，在 result 中说明错误
3. 详细记录执行过程

输出字段：
- performer_target：本轮执行目标
- performer_process：执行步骤
- performer_result：执行结果

输出格式（严格遵守 JSON）：
{
    "performer_target": "",
    "performer_process": "",
    "performer_result": ""
}
```

#### Supervisor（监督者）模板

```txt
角色：任务监督评估专家

职责：评估执行结果，决定是否继续

评估标准：
1. 结果是否满足用户需求
2. 数据是否准确完整
3. 是否需要进一步优化

输出字段：
- supervisor_evaluation：评估结果（pass/continue）
- supervisor_reasoning：评估理由
- supervisor_suggestion：改进建议

输出格式（严格遵守 JSON）：
{
    "supervisor_evaluation": "pass",
    "supervisor_reasoning": "",
    "supervisor_suggestion": ""
}
```

#### Summarizer（总结者）模板

```txt
角色：任务总结专家

职责：汇总整个执行过程，生成最终报告

总结要点：
1. 用户原始需求
2. 执行过程概述
3. 最终结果
4. 关键发现

输出格式：
使用自然、友好的语言，直接输出最终总结文本，无需 JSON 格式。
```

### 4.4 Prompt 调优技巧

#### 技巧 1：Few-Shot Learning

提供多个示例：

```txt
示例1：
输入：查询北京天气
输出：{"target": "查询北京天气", "process": "...", "result": "北京今日：晴天，15-25℃"}

示例2：
输入：查询上海天气
输出：{"target": "查询上海天气", "process": "...", "result": "上海今日：多云，18-28℃"}

现在请处理：{0}
```

#### 技巧 2：思维链（Chain of Thought）

引导 AI 逐步思考：

```txt
请按以下步骤思考：
步骤1：理解用户需求...
步骤2：选择合适的工具...
步骤3：准备工具参数...
步骤4：执行工具调用...
步骤5：解析返回结果...

输出格式：
{
    "thought_process": "步骤1-5的思考过程",
    "final_output": "最终结果"
}
```

#### 技巧 3：错误引导

告诉 AI 如何处理错误：

```txt
如果出现以下错误，请按对应方式处理：

错误1：工具调用失败
处理方式：在 result 字段说明"工具调用失败：{错误信息}"，然后继续下一个任务

错误2：参数缺失
处理方式：在 result 字段说明"缺少必要参数：{参数名}"，并提醒用户提供

错误3：返回数据为空
处理方式：在 result 字段说明"未查询到数据，请稍后重试"
```

### 4.5 Prompt 版本管理

#### 方式 1：文件系统

按版本存储 Prompt：

```
prompt/
├── v1.0/
│   ├── client_performer_web.txt
│   └── client_analyzer_web.txt
├── v1.1/
│   ├── client_performer_web.txt
│   └── client_analyzer_web.txt
└── current -> v1.1/
```

#### 方式 2：数据库存储

在 ai_prompt 表中增加版本字段：

```sql
ALTER TABLE ai_prompt ADD COLUMN version VARCHAR(10);
ALTER TABLE ai_prompt ADD COLUMN is_current TINYINT;
```

查询当前版本：
```sql
SELECT * FROM ai_prompt
WHERE prompt_id = 'prompt_performer_web'
  AND is_current = 1;
```

#### 方式 3：Git 管理

将 Prompt 放入 Git 仓库：

```
prompt/
├── system-prompt/
│   └── client_performer_web.txt
└── user-prompt/
    └── client_performer_web.txt
```

每次修改提交 Git：
```bash
git add prompt/
git commit -m "优化 Performer Prompt，增加错误处理规则"
git tag v1.1.0
```

### 4.6 Prompt 测试

#### 单元测试

```java
@SpringBootTest
class PromptTest {

    @Resource
    private ChatClient chatClient;

    @Test
    void testPerformerPrompt() {
        String prompt = loadPrompt("client_performer_web.txt");
        String userInput = "帮我搜索最新的 AI 技术动态";

        String response = chatClient.prompt()
            .user(prompt.formatted(userInput, ""))
            .call()
            .content();

        // 验证是否是合法 JSON
        JSONObject json = JSONObject.parseObject(response);
        assertNotNull(json.getString("performer_target"));
        assertNotNull(json.getString("performer_process"));
        assertNotNull(json.getString("performer_result"));
    }
}
```

#### A/B 测试

创建不同版本的 Prompt，对比效果：

```java
@Test
void testPromptComparison() {
    String promptV1 = loadPrompt("v1.0/client_performer_web.txt");
    String promptV2 = loadPrompt("v1.1/client_performer_web.txt");

    String responseV1 = testPrompt(promptV1);
    String responseV2 = testPrompt(promptV2);

    // 对比质量、准确性、格式规范性
    comparePrompts(responseV1, responseV2);
}
```

---

## 五、实战案例

### 5.1 案例：文章生成与发布 Agent

**需求**：用户输入主题，AI 生成文章并发布到 CSDN，然后通知企业微信。

**Agent 配置**：

1. **Agent 类型**：Step

2. **Client 配置**：
   - client_inspector_article：检查主题合法性
   - client_planner_article：规划文章结构
   - client_runner_article：生成文章并发布
   - client_replier_article：回复用户发布结果

3. **MCP 工具**：
   - csdn：saveArticle（发布文章）
   - wecom：sendTextCard（发送通知）

4. **Prompt 示例**（client_runner_article.txt）：

```txt
角色：文章生成与发布执行专家

职责：根据规划者的输出，生成文章并发布到 CSDN

可用工具：
- saveArticle：发布文章到 CSDN
  参数：title（标题）、markdownContent（Markdown 内容）
- wecom：sendTextCard（发送企业微信通知）
  参数：title、description、url

执行步骤：
1. 根据文章规划生成完整内容
2. 调用 saveArticle 发布文章
3. 解析返回的文章 ID 和 URL
4. 调用 wecom 发送通知

输出格式（严格遵守 JSON）：
{
    "runner_target": "生成并发布关于 {主题} 的文章",
    "runner_process": "生成文章内容 -> 调用 saveArticle -> 文章ID: {id} -> 调用 wecom",
    "runner_result": "文章已成功发布！链接：{url}"
}
```

**测试**：
```
用户输入：写一篇关于 Spring AI 的文章

执行流程：
1. Inspector 检查主题合法
2. Planner 规划文章结构（简介、核心功能、实战案例、总结）
3. Runner 生成文章并发布到 CSDN，发送企业微信通知
4. Replier 回复：文章已发布！链接：https://blog.csdn.net/xxx/123
```

### 5.2 案例：联网搜索智能体

**需求**：用户提问，AI 联网搜索并总结答案。

**Agent 配置**：

1. **Agent 类型**：Loop

2. **Client 配置**：
   - client_analyzer_web：分析问题
   - client_performer_web：搜索并提取信息
   - client_supervisor_web：评估答案质量
   - client_summarizer_web：总结最终答案

3. **MCP 工具**：
   - bocha：webSearch（联网搜索）

4. **Prompt 示例**（client_supervisor_web.txt）：

```txt
角色：答案质量监督评估专家

职责：评估 Performer 的搜索结果是否满足用户需求

评估标准：
1. 是否准确回答了用户问题
2. 信息是否完整
3. 来源是否可靠

评估字段：
- supervisor_evaluation：pass（通过）/ continue（继续）
- supervisor_reasoning：评估理由
- supervisor_suggestion：改进建议（如果 evaluation=continue）

输出格式（严格遵守 JSON）：
{
    "supervisor_evaluation": "pass",
    "supervisor_reasoning": "搜索结果准确回答了用户问题，信息完整",
    "supervisor_suggestion": ""
}
```

**测试**：
```
用户输入：2025 年最新的 AI 技术趋势有哪些？

执行流程：
Round 1:
1. Analyzer：分析问题，需要搜索 2025 AI 技术趋势
2. Performer：调用 webSearch 搜索，得到结果
3. Supervisor：评估结果不够详细，建议继续
4. 返回 Analyzer

Round 2:
1. Analyzer：根据建议，搜索更具体的内容
2. Performer：搜索"2025 AI 大模型趋势"
3. Supervisor：评估结果详细，通过
4. Summarizer：总结最终答案

最终输出：
2025 年 AI 技术的主要趋势包括：
1. 多模态大模型...
2. Agent 应用爆发...
3. 边缘 AI 部署...
...
```

---

## 六、常见问题与解决方案

### 6.1 MCP 工具不被调用

**问题**：配置了 MCP 工具，但 AI 不调用。

**解决方案**：
1. 检查 @Tool 注解的 description 是否清晰
2. 在 Prompt 中明确告诉 AI 可以使用该工具
3. 提供工具调用示例

**优化 Prompt**：
```txt
可用工具：
- webSearch：联网搜索工具（必须使用此工具获取最新信息）

执行规则：
- 必须调用 webSearch 工具，不能编造信息
- 参数 query：搜索关键词（必填）
- 参数 freshness：时间范围（必填），可选值：oneDay, oneWeek, oneMonth
```

### 6.2 Agent 输出格式错误

**问题**：AI 不按要求的 JSON 格式输出。

**解决方案**：
1. 在 Prompt 中多次强调格式要求
2. 使用负面约束（"禁止..."）
3. 在代码中做格式验证和重试

**代码示例**：
```java
int retryCount = 0;
String response;
while (retryCount < 3) {
    response = chatClient.call().content();
    try {
        JSONObject json = JSONObject.parseObject(response);
        break;
    } catch (Exception e) {
        retryCount++;
        if (retryCount >= 3) {
            throw new RuntimeException("AI 输出格式错误，已重试 3 次");
        }
    }
}
```

### 6.3 记忆混乱

**问题**：不同会话的记忆互相干扰。

**解决方案**：
1. 确保 sessionId 唯一
2. 不要混用 Chat 和 Work 的 sessionId
3. 定期清理过期记忆

**代码示例**：
```java
// 生成唯一的 sessionId
String sessionId = UUID.randomUUID().toString();

// 标记会话类型
String sessionType = "work";
String key = sessionType + ":" + sessionId;

// 设置过期时间
redisTemplate.opsForValue().set(key, memory, Duration.ofHours(24));
```

### 6.4 Prompt 效果不好

**问题**：Prompt 写了，但 AI 执行效果不理想。

**解决方案**：
1. 使用 A/B 测试对比不同版本
2. 添加更多示例
3. 分步引导（思维链）
4. 收集错误案例，针对性优化

**优化流程**：
```
1. 收集问题案例
2. 分析失败原因
3. 修改 Prompt
4. 测试验证
5. 对比效果
6. 迭代优化
```

---

## 七、总结

通过本指南，你应该能够：

1. **开发 MCP 服务**：从零创建一个自定义 MCP 工具
2. **配置 Agent 工作流**：设计并实现复杂的 Agent 任务流程
3. **管理记忆模块**：使用 ChatMemory 保存会话上下文
4. **编写高质量 Prompt**：遵循最佳实践编写有效的 Prompt

**关键要点**：
- MCP：工具是 AI Agent 的手脚，负责执行具体操作
- Agent：Agent 是大脑，协调多个角色完成复杂任务
- Memory：记忆是上下文，保持多轮对话的连贯性
- Prompt：Prompt 是指令，告诉 AI 如何思考和行动

**持续学习**：
- 关注 Spring AI 官方文档更新
- 学习 Prompt Engineering 最佳实践
- 参考开源 Agent 项目（如 AutoGPT、LangChain）
- 实践中不断总结和优化

祝你在 AI Agent 开发之路上一帆风顺！
