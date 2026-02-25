# AI Agent 平台零起点复现任务清单

---

## 🧑‍🏫 导师人设 (Mentor Persona)
**角色**：资深架构师 / 你的复现导师
**风格**：AI 时代思维，侧重系统设计、架构原理与验证方法。
**职责**：
1. **全貌视角**：始终先从全局俯瞰，明确系统在整个生态中的位置。
2. **多维解构**：针对每个模块，提供“7维度分析法”：
   - **全貌 (Overview)**：模块的功能定位。
   - **实现方案 (Alternatives)**：多种可选的技术路径。
   - **具体实现 (Current Path)**：本项目选择的实现方案。
   - **设计动机 (Rationale)**：为什么选择该方案及其背后的考量。
   - **扩展性 (Extensibility)**：未来如何应对变化。
   - **可用性 (Availability)**：如何保证系统的稳定运行。
   - **验证方法 (Verification)**：如何通过自动化手段验证正确性。
3. **进度监督**：实时跟进复现进度。
4. **源码指引**：提供精准的代码参考。

---

## 📅 总体进度
- [ ] **第一阶段：基础设施与多数据源搭建** (进行中)
- [ ] **第二阶段：Agent 核心大脑：规则树与模型层** (待开始)
- [ ] **第三阶段：MCP 协议扩展：赋予 Agent “双手”** (待开始)
- [ ] **第四阶段：RAG 增强与任务调度** (待开始)
- [ ] **第五阶段：前端交互与管理后台** (待开始)

---

## 🛠 详细任务分解

### 第一阶段：基础设施与多数据源搭建 (Environment & Infra)
**核心目标**：构建项目的“地基”，支持业务数据、缓存和向量存储。
- [ ] **1.1 搭建 Maven 多模块工程**
    - 创建父工程 `ai-agent`
    - 创建子模块：`ai-agent-api`, `ai-agent-app`, `ai-agent-domain`, `ai-agent-infrastructure`, `ai-agent-trigger`, `ai-agent-types`
- [ ] **1.2 依赖管理配置**
    - 在父 `pom.xml` 中配置 Spring Boot 3.4.3 及核心依赖版本
- [ ] **1.3 数据库环境准备 (Docker)**
    - 启动 MySQL 8.0, Redis, PostgreSQL (含 Vector 扩展)
- [ ] **1.4 基础配置类实现**
    - 实现多数据源配置 [DataSourceConfig.java](file:///e:/opencodetest/0224%20agent%20skill%20mcp%20java/Agent/backend/ai-agent-app/src/main/java/com/dasi/config/DataSourceConfig.java)
    - 实现 Redis 序列化配置 [RedisConfig.java](file:///e:/opencodetest/0224%20agent%20skill%20mcp%20java/Agent/backend/ai-agent-app/src/main/java/com/dasi/config/RedisConfig.java)
- [ ] **1.5 缓存切面 AOP**
    - 实现 `@Cacheable` 和 `@CacheEvict` 自定义注解及切面逻辑

### 第二阶段：Agent 核心大脑：规则树与模型层 (Core Logic)
**核心目标**：实现 Agent 的思考逻辑，让它能根据 Prompt 调用不同的模型。
- [ ] **2.1 领域模型设计 (Domain Model)**
    - 理解“装配链路 (Armory)”如何把配置装配成可用的运行时对象
- [ ] **2.2 核心节点实现**
    - 装配入口：[ArmoryRootNode.java](file:///e:/opencodetest/0224%20agent%20skill%20mcp%20java/Agent/backend/ai-agent-domain/src/main/java/com/dasi/domain/ai/service/armory/node/ArmoryRootNode.java)
    - API 装配：[ArmoryApiNode.java](file:///e:/opencodetest/0224%20agent%20skill%20mcp%20java/Agent/backend/ai-agent-domain/src/main/java/com/dasi/domain/ai/service/armory/node/ArmoryApiNode.java)
    - Model 装配：[ArmoryModelNode.java](file:///e:/opencodetest/0224%20agent%20skill%20mcp%20java/Agent/backend/ai-agent-domain/src/main/java/com/dasi/domain/ai/service/armory/node/ArmoryModelNode.java)
    - 动态 Bean 注册机制：[AbstractArmoryNode.java](file:///e:/opencodetest/0224%20agent%20skill%20mcp%20java/Agent/backend/ai-agent-domain/src/main/java/com/dasi/domain/ai/service/armory/node/AbstractArmoryNode.java)
- [ ] **2.3 多模型适配器**
    - 封装 OpenAI、DeepSeek、Ollama 的统一调用接口

### 第三阶段：MCP 协议扩展：赋予 Agent “双手” (Tool Use)
**核心目标**：通过 MCP 协议让 Agent 具备调用外部工具的能力。
- [ ] **3.1 MCP 服务端开发**
    - 实现基于高德 API 的天气查询服务 [mcp-server-amap](file:///e:/opencodetest/0224%20agent%20skill%20mcp%20java/Agent/mcp/mcp-server-amap/)
- [ ] **3.2 Agent 工具链对接**
    - MCP 客户端装配（SSE/STDIO）：[ArmoryMcpNode.java](file:///e:/opencodetest/0224%20agent%20skill%20mcp%20java/Agent/backend/ai-agent-domain/src/main/java/com/dasi/domain/ai/service/armory/node/ArmoryMcpNode.java)
    - 将 MCP 工具挂到 ChatClient（Tool Call）：[ArmoryClientNode.java](file:///e:/opencodetest/0224%20agent%20skill%20mcp%20java/Agent/backend/ai-agent-domain/src/main/java/com/dasi/domain/ai/service/armory/node/ArmoryClientNode.java)
    - MCP 配置从数据库加载与解析：[AiRepository.java](file:///e:/opencodetest/0224%20agent%20skill%20mcp%20java/Agent/backend/ai-agent-infrastructure/src/main/java/com/dasi/infrastructure/repository/AiRepository.java#L227-L284)
- [ ] **3.3 任务规划器 (Planner)**
    - 实现 `PlannerNode` 拆解复杂任务
    - 
**关键链路：主应用 <-> MCP（你需要掌握的“关系”）**
1. MCP Server 是独立 Spring Boot 应用（在 `mcp/` 下），通过 `spring.ai.mcp.server.*` 暴露工具能力，例如 [mcp-server-amap application.yml](file:///e:/opencodetest/0224%20agent%20skill%20mcp%20java/Agent/mcp/mcp-server-amap/src/main/resources/application.yml)。
2. 主应用侧并不是“Spring 自动发现 MCP”，而是由业务配置驱动：MCP 定义存于 `ai_mcp`（字段 `mcpType` + `mcpConfig` JSON）并通过 `ai_config` 绑定到 client。
3. 运行时装配时，主应用创建 `McpSyncClient`（SSE/STDIO）并动态注册为 Spring Bean，然后由 `ChatClient` 的 ToolCallbacks 触发实际工具调用。

### 第四阶段：RAG 增强与任务调度 (RAG & Jobs)
**核心目标**：支持知识库检索和异步任务处理。
- [ ] **4.1 向量检索实现**
    - 实现 PgVector 存储与检索逻辑 [PgVectorStoreConfig.java](file:///e:/opencodetest/0224%20agent%20skill%20mcp%20java/Agent/backend/ai-agent-app/src/main/java/com/dasi/config/PgVectorStoreConfig.java)
- [ ] **4.2 自动化任务调度**
    - 实现基于 Job 的 Agent 执行逻辑 [AgentTaskJob.java](file:///e:/opencodetest/0224%20agent%20skill%20mcp%20java/Agent/backend/ai-agent-trigger/src/main/java/com/dasi/trigger/job/AgentTaskJob.java)

### 第五阶段：前端交互与管理后台 (Frontend & Admin)
**核心目标**：提供美观、实用的用户交互界面。
- [ ] **5.1 聊天交互页面**
    - 基于 Vue3 + Tailwind 实现 [Chat.vue](file:///e:/opencodetest/0224%20agent%20skill%20mcp%20java/Agent/frontend/src/components/Chat.vue)
- [ ] **5.2 管理后台 (Admin)**
    - 实现可视化流程配置与状态监控

---

## 📝 导师指导记录
- **2026-02-24**：初始化复现计划，确立五个主要阶段。
