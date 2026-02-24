# Dasi Agent 项目文档

## 项目简介

Dasi Agent 是一个集成了 AI 对话、多角色 Agent 工作流、RAG 知识库和 MCP 工具调用的全栈 AI Agent 平台。本项目采用前后端分离架构，基于 Spring Boot 3 + Vue 3 构建。

## 文档导航

### 📖 [技术设计方案.md](./技术设计方案.md)
详细的项目技术设计文档，包含：
- 系统架构设计（DDD 分层架构）
- 核心功能设计（AI 对话、Agent 工作流、RAG、MCP、会话记忆）
- 数据库设计
- 技术亮点分析
- 部署方案
- 扩展指南

**适合人群**：架构师、技术负责人、想深入了解项目整体设计的开发者

---

### 🛠️ [MCP-Agent-Prompt开发实现指南.md](./MCP-Agent-Prompt开发实现指南.md)
实战开发指南，重点讲解如何在工作中实现：
- **MCP 服务开发**：从零创建自定义 MCP 工具
- **Agent 工作流开发**：设计并实现复杂 Agent 任务流程
- **记忆模块开发**：使用 ChatMemory 管理会话上下文
- **Prompt 编写**：遵循最佳实践编写高质量 Prompt

**适合人群**：开发者、AI 工程师、需要在工作中落地 AI Agent 技术的人员

---

## 项目截图

### 聊天界面
- **01-聊天界面-MCP工具调用示例.png**：展示通过 MCP 工具查询天气的对话界面
- **05-聊天界面-RAG知识库对话.png**：展示 RAG 增强对话，基于知识库回答问题

### 管理后台
- **02-FLOW管理界面-Agent列表.png**：展示 Agent 列表和管理界面
- **03-CONFIG-CANVAS-可视化工作流配置.png**：展示基于 vue-flow 的可视化工作流配置
- **04-DASHBOARD仪表盘-数据统计.png**：展示数据统计和趋势图表
- **07-MODEL管理-模型配置界面.png**：展示模型管理界面
- **08-FLOW配置-Agent工作流详细配置.png**：展示 Agent 工作流详细配置

### 工作流执行
- **06-WORK工作流-AI文章生成与发布.png**：展示文章生成与发布工作流的执行过程

### 代码结构
- **09-前端代码结构.png**：展示前端项目的代码组织结构

---

## 快速开始

### 环境要求
- JDK 17+
- Node.js 18+
- MySQL 8.0+
- PostgreSQL 14+ (带 pgvector 扩展)
- Redis 6.0+

### 后端启动
```bash
cd Agent/backend
mvn clean install
cd ai-agent-app
mvn spring-boot:run
```

### 前端启动
```bash
cd Agent/frontend
npm install
npm run dev
```

### MCP 服务启动
```bash
cd Agent/mcp
./docker-build.sh
docker-compose up -d
```

---

## 核心功能

### 1. AI 对话
- ✅ 普通对话（支持流式/非流式）
- ✅ RAG 增强对话（知识库问答）
- ✅ MCP 工具调用对话（外部服务集成）
- ✅ 会话记忆管理

### 2. Agent 工作流
- ✅ Loop 策略（循环模式：Analyzer → Performer → Supervisor → Summarizer）
- ✅ Step 策略（步骤模式：Inspector → Planner → Runner → Replier）
- ✅ 可视化工作流配置
- ✅ SSE 实时推送执行进度

### 3. RAG 知识库
- ✅ 文件上传（支持多种文档格式）
- ✅ Git 仓库导入
- ✅ 向量检索（基于 PostgreSQL + pgvector）
- ✅ 知识库标签隔离

### 4. MCP 服务
- ✅ 支持 SSE 方式（远程服务）
- ✅ 支持 STDIO 方式（本地进程）
- ✅ 动态工具挂载
- ✅ 内置 5 个 MCP 服务（CSDN、企业微信、高德天气、邮件、博查搜索）

### 5. 管理后台
- ✅ Dashboard 数据统计
- ✅ Flow/Agent/Client 管理
- ✅ Model/API/MCP/Prompt 管理
- ✅ Session 会话管理
- ✅ User 用户管理

---

## 技术栈

### 后端
- Spring Boot 3
- Java 17
- Spring AI
- MyBatis
- MySQL + PostgreSQL + Redis

### 前端
- Vue 3
- Vite
- TailwindCSS
- Pinia
- vue-flow

---

## 项目结构

```
Agent/
├── backend/                     # 后端工程
│   ├── ai-agent-api            # API 层
│   ├── ai-agent-app            # 应用启动与配置层
│   ├── ai-agent-domain         # 领域层
│   ├── ai-agent-infrastructure # 基础设施层
│   ├── ai-agent-trigger        # 接口适配层
│   └── ai-agent-types          # 公共对象层
│
├── frontend/                    # 前端工程
│   ├── src/                    # 源码目录
│   └── package.json            # 依赖配置
│
└── mcp/                         # MCP 服务集合
    ├── mcp-server-csdn         # CSDN 文章发布
    ├── mcp-server-wecom        # 企业微信通知
    ├── mcp-server-amap         # 高德天气查询
    ├── mcp-server-email        # 邮件发送
    └── mcp-server-bocha        # 博查联网搜索
```

---

## 学习路径

### 1️⃣ 了解项目整体设计
阅读 [技术设计方案.md](./技术设计方案.md)，理解：
- 系统架构和分层设计
- 核心功能的实现原理
- 技术亮点和最佳实践

### 2️⃣ 学习实战开发技能
阅读 [MCP-Agent-Prompt开发实现指南.md](./MCP-Agent-Prompt开发实现指南.md)，掌握：
- 如何开发自定义 MCP 服务
- 如何配置 Agent 工作流
- 如何使用 ChatMemory 管理会话
- 如何编写高质量 Prompt

### 3️⃣ 查看截图理解功能
查看项目截图，直观了解：
- 前端界面设计
- 工作流配置方式
- Agent 执行过程

### 4️⃣ 阅读源码深入理解
查看关键源码文件：
- `AiController.java`：AI 对话和工作流执行入口
- `AugmentService.java`：RAG 和 MCP 增强服务
- `DispatchService.java`：策略分发服务
- `ExecuteLoopStrategy.java`：Loop 策略实现
- `ExecutePerformerNode.java`：Performer 节点实现

---

## 常见问题

### Q1: 如何新增一个 MCP 服务？
参考 [MCP-Agent-Prompt开发实现指南.md](./MCP-Agent-Prompt开发实现指南.md) 第一章，包含完整的创建步骤。

### Q2: 如何创建一个新的 Agent？
参考 [MCP-Agent-Prompt开发实现指南.md](./MCP-Agent-Prompt开发实现指南.md) 第二章，包含详细的配置流程。

### Q3: ChatMemory 如何使用？
参考 [MCP-Agent-Prompt开发实现指南.md](./MCP-Agent-Prompt开发实现指南.md) 第三章，包含基本用法和配置。

### Q4: Prompt 如何优化？
参考 [MCP-Agent-Prompt开发实现指南.md](./MCP-Agent-Prompt开发实现指南.md) 第四章，包含编写原则和调优技巧。

---

## 贡献指南

欢迎提交 Issue 和 Pull Request！

---

## 许可证

MIT License

---

## 联系方式

如有问题，请提交 Issue 或联系项目维护者。
