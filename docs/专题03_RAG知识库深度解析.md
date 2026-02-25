# 专题三：RAG 知识库深度解析

> **学习目标**：掌握 RAG（检索增强生成）的完整实现原理，能够设计并实现企业级知识库系统。
> **前置知识**：了解向量数据库、Embedding 概型、相似度检索
> **预计用时**：50 分钟

---

## 一、真实业务场景

### 场景 1：企业知识库问答

**业务需求**：
- 企业有大量内部文档（产品手册、技术文档、规章制度）
- 员工提问时，系统需要基于这些文档回答
- 不同部门的知识需要隔离（销售、技术、HR）

**对应技术**：
- 上传文档到知识库
- 按 ragTag 隔离不同知识库
- 向量检索 + LLM 生成答案

### 场景 2：客服知识库

**业务需求**：
- 客服需要快速回答客户问题
- 基于产品文档、FAQ、历史工单
- 需要引用原文来源

**对应技术**：
- RAG 检索增强
- 返回引用的文档片段
- 支持多文档检索

### 场景 3：代码助手

**业务需求**：
- 开发者询问代码问题
- 需要检索代码仓库
- 基于项目代码回答

**对应技术**：
- Git 仓库导入
- 代码切片与向量化
- 上下文检索

---

## 二、RAG 系统架构

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         数据摄入层                                    │
├─────────────────────────────────────────────────────────────────────┤
│  文件上传 │ Git 仓库 │ 网页爬取 │ 数据库导入                         │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│                         文档处理层                                    │
├─────────────────────────────────────────────────────────────────────┤
│  TikaDocumentReader (读取文档)                                      │
│  TokenTextSplitter (切片)                                           │
│  EmbeddingModel (向量化)                                            │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│                         向量存储层                                    │
├─────────────────────────────────────────────────────────────────────┤
│  PgVectorStore (PostgreSQL + pgvector)                             │
│  ├─ id: 文档切片 ID                                                │
│  ├─ embedding: 向量 (1536 维)                                       │
│  ├─ content: 文档内容                                              │
│  └─ metadata: 元数据 (knowledge=ragTag, source, page)               │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│                         检索层                                        │
├─────────────────────────────────────────────────────────────────────┤
│  用户输入 → 向量化 → 相似度检索 → TopK 结果                         │
│  Filter.Expression (按 knowledge 标签过滤)                          │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│                         生成层                                        │
├─────────────────────────────────────────────────────────────────────┤
│  构建增强提示词：                                                   │
│  "基于以下文档回答问题：{检索到的文档}                               │
│   用户问题：{用户输入}"                                             │
│  ↓                                                                   │
│  LLM 生成答案                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 核心概念

| 概念 | 定义 | 举例 |
|-----|------|------|
| **Embedding** | 将文本转换为向量表示，捕捉语义信息 | "你好" → [0.1, -0.2, 0.5, ...] |
| **向量数据库** | 存储向量并支持相似度检索的数据库 | PostgreSQL + pgvector |
| **切片 (Chunk)** | 将长文档切分成小段，便于检索 | 每 500 tokens 一段 |
| **TopK** | 检索最相似的 K 个片段 | topK=5 表示检索前 5 个最相关片段 |
| **元数据过滤** | 检索时按元数据过滤结果 | 只检索特定知识库的文档 |
| **相似度** | 两个向量之间的语义相似程度 | 0-1 之间，越接近 1 越相似 |

---

## 三、文档处理流程

### 3.1 完整流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        文档上传                                      │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│                   TikaDocumentReader 读取                            │
│  支持格式：PDF, DOC, DOCX, PPT, PPTX, TXT, HTML, MD                 │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│                      文档预处理                                      │
│  ├─ 清理特殊字符                                                    │
│  ├─ 移除空白页                                                      │
│  └─ 提取文本内容                                                    │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│                    TokenTextSplitter 切片                           │
│  切片策略：按 Token 数量切分（如 500 tokens/chunk）                  │
│  重叠策略：相邻切片重叠 50 tokens（保持上下文）                      │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│                   EmbeddingModel 向量化                              │
│  调用 OpenAI Embedding API (text-embedding-3-small)                 │
│  生成 1536 维向量                                                   │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│                     写入 PgVectorStore                              │
│  每个切片作为一条记录：                                              │
│  ├─ embedding: 向量                                                 │
│  ├─ content: 文本内容                                              │
│  └─ metadata: {knowledge: "product_docs", source: "manual.pdf"}     │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 切片策略对比

| 策略 | 实现方式 | 优点 | 缺点 | 适用场景 |
|-----|---------|------|------|---------|
| **固定 Token 数** | 每 N tokens 切一片 | 切片大小均匀 | 可能断开句子 | 通用场景 |
| **固定字符数** | 每 N 字符切一片 | 实现简单 | 可能截断单词 | 英文文档 |
| **按段落切分** | 以段落为单位 | 保持语义完整 | 切片大小不均 | 结构化文档 |
| **语义切分** | 按语义边界切分 | 最符合语义 | 实现复杂 | 高质量要求 |

**本项目选择**：固定 Token 数（500 tokens/chunk，重叠 50 tokens）

### 3.3 重叠策略

```
原文：A B C D E F G H I J（每个字母代表 100 tokens）

无重叠切片：
Chunk 1: A B C D E
Chunk 2: F G H I J  ← E 和 F 之间的上下文丢失

重叠切片（重叠 100 tokens）：
Chunk 1: A B C D E
Chunk 2: E F G H I  ← 保留了 E 的上下文
Chunk 3: I J        ← 保留了 I 的上下文
```

**重叠的作用**：
- 保持上下文连贯性
- 避免关键信息在切片边界丢失
- 提高检索召回率

---

## 四、向量检索原理

### 4.1 相似度计算

#### 余弦相似度（最常用）

```
similarity = cos(θ) = (A · B) / (|A| × |B|)

其中：
- A 和 B 是两个向量
- A · B 是向量点积
- |A| 和 |B| 是向量模长
```

**直观理解**：
- 相似度 = 1：完全相同
- 相似度 = 0：完全无关
- 相似度 = 0.8：比较相似

#### 实际计算示例

```
查询向量：Q = [0.1, 0.2, 0.3]
文档向量：D1 = [0.1, 0.2, 0.4]  → 相似度 0.98
文档向量：D2 = [0.5, 0.1, 0.1]  → 相似度 0.71
文档向量：D3 = [-0.1, -0.2, -0.3] → 相似度 -1.0（完全相反）

检索结果：D1 > D2 > D3
```

### 4.2 检索流程

```
用户输入："公司的年假政策是什么？"
    ↓
1. 向量化用户输入
   QueryVector = EmbeddingModel.embed("公司的年假政策是什么？")
   ↓
2. 计算与所有文档切片的相似度
   Score1 = cosine(QueryVector, Doc1Vector) = 0.92
   Score2 = cosine(QueryVector, Doc2Vector) = 0.85
   Score3 = cosine(QueryVector, Doc3Vector) = 0.78
   ...
   ↓
3. 按相似度排序，取 TopK
   Top5 = [Doc1, Doc2, Doc3, Doc4, Doc5]
   ↓
4. 返回检索结果
   [
     {content: "年假政策...", score: 0.92, metadata: {...}},
     {content: "请假规定...", score: 0.85, metadata: {...}},
     ...
   ]
```

### 4.3 元数据过滤

**场景**：企业有多个部门的知识库，检索时只检索特定部门

**实现**：
```java
// 构建过滤条件
FilterExpressionBuilder builder = new FilterExpressionBuilder();
Filter.Expression expression = builder
    .eq("knowledge", "hr_docs")  // 只检索 HR 知识库
    .build();

// 执行过滤检索
SearchRequest request = SearchRequest.builder()
    .query(userMessage)
    .filterExpression(expression)  // 应用过滤
    .topK(5)
    .build();

List<Document> results = pgVectorStore.similaritySearch(request);
```

**支持的过滤操作**：
| 操作 | 说明 | 示例 |
|-----|------|------|
| eq | 等于 | knowledge = "hr_docs" |
| ne | 不等于 | source != "temp.pdf" |
| in | 在列表中 | category in ["policy", "faq"] |
| and | 并且 | knowledge = "hr" AND year = 2024 |
| or | 或者 | type = "pdf" OR type = "docx" |

---

## 五、RAG 提示词工程

### 5.1 提示词模板

#### 基础模板

```
你是一个专业的知识库助手。

【参考文档】
{documents}

【用户问题】
{question}

【回答要求】
1. 答案必须基于参考文档
2. 如果文档中没有相关信息，明确告知用户
3. 引用文档来源，如"根据《员工手册》第3章..."
4. 保持专业、友好的语气

请回答：
```

#### 增强模板（带引用）

```
你是一个专业的知识库助手。

【参考文档】
文档1（来源：{source1}，相关度：{score1}）：
{content1}

文档2（来源：{source2}，相关度：{score2}）：
{content2}

文档3（来源：{source3}，相关度：{score3}）：
{content3}

【用户问题】
{question}

【回答要求】
1. 优先使用相关度高的文档
2. 在答案中标注引用来源
3. 如果文档内容矛盾，说明不同来源的观点
4. 如果文档信息不足，告知用户需要补充的信息

请回答：
```

### 5.2 提示词优化技巧

| 技巧 | 说明 | 示例 |
|-----|------|------|
| **明确来源** | 要求 LLM 标注信息来源 | "根据《产品手册》v2.0 第5页..." |
| **设置阈值** | 只使用相关度高于阈值的结果 | "只使用相关度 > 0.7 的文档" |
| **冲突处理** | 告知 LLM 如何处理矛盾信息 | "如果文档内容矛盾，分别说明不同观点" |
| **拒绝回答** | 明确何时拒绝回答 | "如果文档中没有相关信息，直接说'文档中没有找到相关信息'" |
| **结构化输出** | 要求 LLM 按结构输出 | "请按以下格式回答：结论、依据、来源" |

### 5.3 提示词示例对比

#### 差的提示词

```
基于以下文档回答问题：
{documents}

问题：{question}
```

**问题**：
- 没有明确要求基于文档
- LLM 可能产生幻觉
- 没有处理文档中没有信息的情况

#### 好的提示词

```
你是一个专业的知识库助手。请严格按照以下规则回答：

【参考文档】
{documents}

【用户问题】
{question}

【重要规则】
1. 你只能使用参考文档中的信息回答问题
2. 如果参考文档中没有相关信息，必须说："抱歉，文档中没有找到关于{question}的信息"
3. 回答时请引用具体的文档来源
4. 不要添加文档中没有的内容

请回答：
```

---

## 六、知识库管理

### 6.1 知识库隔离

**实现方式**：通过 ragTag（元数据中的 knowledge 字段）隔离

```java
// 上传时指定 ragTag
ragService.uploadTextFile("product_docs", fileList);
// 所有文档的 metadata.knowledge = "product_docs"

// 检索时指定 ragTag
Filter.Expression expression = builder
    .eq("knowledge", "product_docs")
    .build();
```

**应用场景**：
| 知识库 | ragTag | 内容 | 访问权限 |
|-------|--------|------|---------|
| 产品文档 | product_docs | 产品手册、API 文档 | 所有用户 |
| 内部规章 | hr_policy | 员工手册、规章制度 | 仅员工 |
| 技术文档 | tech_docs | 架构设计、代码规范 | 技术团队 |

### 6.2 文档更新策略

#### 全量更新

```java
// 删除旧文档
pgVectorStore.delete(Filter.Expression.builder()
    .eq("knowledge", ragTag)
    .build());

// 上传新文档
ragService.uploadTextFile(ragTag, fileList);
```

**优点**：简单，不会有残留
**缺点**：更新期间知识库不可用

#### 增量更新

```java
// 每个文档添加版本号
metadata.put("version", "v2.0");
metadata.put("upload_time", System.currentTimeMillis());

// 检索时使用最新版本
Filter.Expression expression = builder
    .eq("knowledge", ragTag)
    .eq("version", "v2.0")
    .build();
```

**优点**：更新期间不影响使用
**缺点**：需要清理旧版本数据

### 6.3 文档去重

```java
// 使用文档内容的 Hash 作为唯一标识
String docId = DigestUtils.md5Hex(content);

// 检查是否已存在
if (pgVectorStore.exists(docId)) {
    // 更新而不是新增
    pgVectorStore.update(docId, newDocument);
} else {
    // 新增
    pgVectorStore.add(newDocument);
}
```

---

## 七、业务场景实战

### 场景 1：客服知识库

**需求**：
- 基于 FAQ、产品手册、历史工单回答客户问题
- 显示答案来源
- 支持多语言

**实现**：

1. **上传文档**
```java
// FAQ 文档
ragService.uploadTextFile("faq", faqFiles);

// 产品手册
ragService.uploadTextFile("manual", manualFiles);

// 历史工单
ragService.uploadTextFile("tickets", ticketFiles);
```

2. **检索配置**
```java
// 跨多个知识库检索
Filter.Expression expression = builder
    .in("knowledge", Arrays.asList("faq", "manual", "tickets"))
    .build();

SearchRequest request = SearchRequest.builder()
    .query(userQuestion)
    .filterExpression(expression)
    .topK(10)  // 检索更多结果
    .build();
```

3. **提示词优化**
```
你是一个专业的客服助手。

【参考文档】
{documents_with_sources}

【用户问题】
{question}

【回答要求】
1. 如果有 FAQ 直接匹配，优先使用 FAQ
2. 说明答案来源（如"根据 FAQ 第 15 条"）
3. 如果需要更多操作，引导用户联系人工客服
```

### 场景 2：法律文档检索

**需求**：
- 检索法律条文
- 答案需要精确引用
- 不能产生幻觉

**实现**：

1. **切片策略调整**
```java
// 按条款切分，每条一款作为一个切片
TokenTextSplitter splitter = new TokenTextSplitter(null, 200, 50, 5000, true, true);
```

2. **提示词严格约束**
```
你是一个法律文档检索助手。

【参考文档】
{documents}

【用户问题】
{question}

【严格规则】
1. 只能使用参考文档中的原文
2. 必须标注出处（如"根据《民法典》第XXX条"）
3. 如果文档中没有答案，必须说"文档中没有找到相关法律条文"
4. 不要添加任何解释或推测
5. 输出格式：出处 + 原文

请输出：
```

### 场景 3：代码助手

**需求**：
- 检索代码仓库
- 基于项目上下文回答问题

**实现**：

1. **Git 仓库导入**
```java
ragService.uploadGitRepo(AiUploadRequest.builder()
    .ragTag("project_code")
    .gitUrl("https://github.com/xxx/project.git")
    .branch("main")
    .build());
```

2. **代码切片优化**
```java
// 按函数/类切分，保持代码完整性
CodeSplitter splitter = new CodeSplitter();
splitter.setChunkStrategy("function");  // 按函数切分
splitter.setOverlapLines(5);  // 重叠 5 行
```

---

## 八、性能优化

### 8.1 向量索引

```sql
-- 创建 IVFFlat 索引（适合中等规模）
CREATE INDEX ON vector_store
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

-- 创建 HNSW 索引（适合大规模，性能更好）
CREATE INDEX ON vector_store
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

**索引对比**：
| 索引类型 | 构建速度 | 查询速度 | 内存占用 | 适用规模 |
|---------|---------|---------|---------|---------|
| IVFFlat | 快 | 中等 | 低 | < 100 万 |
| HNSW | 慢 | 快 | 高 | > 100 万 |

### 8.2 缓存策略

```java
// 缓存常见问题的检索结果
@Cacheable(key = "#question.hashCode()", expire = 3600)
public List<Document> searchDocuments(String question, String ragTag) {
    // 检索逻辑
}

// 缓存向量化结果
@Cacheable(key = "#text.hashCode()", expire = 86400)
public float[] embedText(String text) {
    return embeddingModel.embed(text);
}
```

### 8.3 批量处理

```java
// 批量向量化
List<float[]> embeddings = embeddingModel.embedAll(texts);

// 批量写入
pgVectorStore.addAll(documents);
```

---

## 九、自测清单

### 理解自测（共 10 题）

#### 基础概念

**Q1**: 什么是 RAG？它解决了什么问题？

<details>
<summary>点击查看答案</summary>

**答案**：
- **全称**：Retrieval-Augmented Generation（检索增强生成）
- **解决问题**：
  1. LLM 知识有截止日期，不知道最新信息
  2. LLM 不知道私有数据（如公司内部文档）
  3. 减少 LLM 幻觉（基于真实文档回答）
  4. 提供可验证的信息来源

**工作原理**：检索相关文档 → 将文档注入提示词 → LLM 基于文档生成答案

</details>

---

**Q2**: Embedding 是什么？为什么能用于语义检索？

<details>
<summary>点击查看答案</summary>

**答案**：
- **定义**：将文本转换为高维向量表示，捕捉语义信息
- **为什么能检索**：
  - 语义相近的文本，向量距离也相近
  - 例如："狗"和"犬"的向量距离很近，虽然字面不同
  - 通过计算向量相似度，可以找到语义相关的内容

**举例**：
```
"苹果公司" → [0.1, 0.2, 0.3, ...]
"苹果水果" → [0.5, 0.1, 0.2, ...]
"微软公司" → [0.12, 0.18, 0.31, ...]

相似度：
"苹果公司" vs "微软公司" = 0.98 (高，都是科技公司)
"苹果公司" vs "苹果水果" = 0.45 (低，语义不同)
```

</details>

---

#### 实现原理

**Q3**: 描述 RAG 的完整工作流程。

<details>
<summary>点击查看答案</summary>

**答案**：
```
1. 文档摄入阶段（离线）：
   - 读取文档（TikaDocumentReader）
   - 切片（TokenTextSplitter）
   - 向量化（EmbeddingModel）
   - 写入向量数据库（PgVectorStore）

2. 检索阶段（实时）：
   - 用户输入问题
   - 向量化问题
   - 计算与文档切片的相似度
   - 按相似度排序，取 TopK
   - 应用元数据过滤（如按知识库过滤）

3. 生成阶段（实时）：
   - 将检索结果注入提示词
   - LLM 基于增强的提示词生成答案
   - 返回答案给用户
```

</details>

---

**Q4**: 为什么需要对文档进行切片？切片过大或过小有什么问题？

<details>
<summary>点击查看答案</summary>

**答案**：
**为什么切片**：
- LLM 有输入长度限制（如 4K/8K tokens）
- 向量检索对长文本效果不好
- 切片后可以更精准地检索相关内容

**切片过大的问题**：
- 检索结果包含大量无关信息
- 降低检索精度
- 超出 LLM 输入限制

**切片过小的问题**：
- 丢失上下文信息
- 语义不完整
- 需要检索更多切片

**推荐**：500 tokens/chunk，重叠 50 tokens

</details>

---

**Q5**: 什么是 TopK？如何设置合适的 TopK 值？

<details>
<summary>点击查看答案</summary>

**答案**：
- **定义**：检索最相似的 K 个文档切片
- **如何选择**：

| TopK | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| 3-5 | 简单问题 | 节省 token，快速 | 可能遗漏相关信息 |
| 5-10 | 一般问题（推荐） | 平衡精度和成本 | token 消耗适中 |
| 10-20 | 复杂问题 | 信息全面 | token 消耗大，可能引入噪音 |

**动态调整策略**：
```java
// 根据问题长度调整 TopK
int topK = question.length() > 50 ? 10 : 5;
```

</details>

---

#### 架构设计

**Q6**: PostgreSQL + pgvector 和专用向量数据库（如 Milvus）如何选择？

<details>
<summary>点击查看答案</summary>

**答案**：

| 维度 | PostgreSQL + pgvector | 专用向量数据库 |
|-----|----------------------|---------------|
| **运维成本** | 低（复用现有数据库） | 高（独立部署） |
| **性能** | 中等（适合中小规模） | 高（适合大规模） |
| **功能** | 关系型 + 向量 | 专注向量检索 |
| **数据一致性** | 强（支持事务） | 弱（最终一致性） |
| **成本** | 低 | 中高 |
| **学习曲线** | 低（SQL 熟悉） | 中（需要学习新系统） |

**选择建议**：
- < 100 万文档 → PostgreSQL + pgvector
- > 100 万文档 → 专用向量数据库
- 需要强一致性 → PostgreSQL + pgvector
- 追求极致性能 → 专用向量数据库

</details>

---

**Q7**: 如何实现多个知识库的联合检索？

<details>
<summary>点击查看答案</summary>

**答案**：
**方案 1：元数据过滤**
```java
Filter.Expression expression = builder
    .in("knowledge", Arrays.asList("faq", "manual", "policy"))
    .build();
```

**方案 2：分别检索后合并**
```java
// 分别检索
List<Document> faqResults = search("faq", question, 3);
List<Document> manualResults = search("manual", question, 3);
List<Document> policyResults = search("policy", question, 3);

// 合并并重新排序
List<Document> allResults = Stream.of(faqResults, manualResults, policyResults)
    .flatMap(List::stream)
    .sorted(Comparator.comparing(Document::getScore).reversed())
    .collect(Collectors.toList());
```

**方案 3：加权检索**
```java
// FAQ 权重更高
SearchRequest request = SearchRequest.builder()
    .query(question)
    .filterExpression(expression)
    .topK(5)
    .weight("knowledge", Map.of("faq", 2.0, "manual", 1.0, "policy", 1.0))
    .build();
```

</details>

---

**Q8**: 如何评估 RAG 系统的效果？

<details>
<summary>点击查看答案</summary>

**答案**：
**评估指标**：

| 指标 | 说明 | 计算方式 |
|-----|------|---------|
| **召回率** | 检索到的相关文档占比 | 相关文档被检索到数 / 总相关文档数 |
| **精确率** | 检索结果中相关文档占比 | 检索到的相关文档数 / 总检索文档数 |
| **MRR** | 第一个相关文档的排名倒数 | 1 / 第一个相关文档的位置 |
| **NDCG** | 考虑位置的相关性得分 | 综合考虑排序和相关性 |

**答案质量评估**：
```java
// 检查答案是否包含检索结果中的关键词
boolean answerRelevant = checkAnswerContainsKeywords(answer, retrievedDocs);

// 检查答案是否准确（对比标准答案）
double accuracy = calculateSimilarity(answer, standardAnswer);
```

**人工评估**：
- 随机抽取 100 个问题
- 人工评估答案质量
- 计算满意度

</details>

---

**Q9**: RAG 系统如何处理文档更新？

<details>
<summary>点击查看答案</summary>

**答案**：
**方案 1：全量更新**
```java
// 1. 删除旧文档
pgVectorStore.delete(Filter.Expression.builder()
    .eq("knowledge", ragTag)
    .eq("version", "v1.0")
    .build());

// 2. 上传新文档
ragService.uploadTextFile(ragTag, newFiles);
```

**方案 2：版本管理**
```java
// 新文档带版本号
metadata.put("version", "v2.0");
metadata.put("upload_time", System.currentTimeMillis());

// 检索时使用最新版本
Filter.Expression expression = builder
    .eq("knowledge", ragTag)
    .eq("version", "v2.0")
    .build();

// 定期清理旧版本
scheduleCleanupOldVersions();
```

**方案 3：增量更新**
```java
// 只更新变化的文档
for (File file : changedFiles) {
    String docId = calculateDocId(file);
    if (pgVectorStore.exists(docId)) {
        pgVectorStore.update(docId, newDocument);
    } else {
        pgVectorStore.add(newDocument);
    }
}
```

</details>

---

**Q10**: 如何提高 RAG 系统的检索准确率？

<details>
<summary>点击查看答案</summary>

**答案**：
**优化方向**：

1. **切片优化**
   - 调整切片大小（500-1000 tokens）
   - 增加重叠（50-100 tokens）
   - 按语义边界切分

2. **检索优化**
   - 使用更好的 Embedding 模型
   - 调整 TopK 值
   - 添加重排序（Rerank）

3. **查询优化**
   - 查询改写（Query Rewriting）
   - 查询扩展（Query Expansion）
   - 多轮检索（Recursive Retrieval）

4. **提示词优化**
   - 明确要求基于文档
   - 设置引用规则
   - 处理冲突信息

5. **反馈学习**
   - 收集用户反馈
   - 优化切片策略
   - 调整检索参数

</details>

---

### 实战自测（共 3 题）

**Q11**: 设计一个法律文档检索系统，要求答案必须精确引用条文。

<details>
<summary>点击查看答案</summary>

**答案**：
**切片策略**：
```java
// 按条款切分，每条一款作为一个切片
TokenTextSplitter splitter = new TokenTextSplitter();
splitter.setChunkSize(200);  // 较小切片，确保精准
splitter.setChunkOverlap(0);  // 无重叠，避免混淆
```

**元数据设计**：
```java
metadata.put("law_name", "民法典");
metadata.put("article", "第123条");
metadata.put("paragraph", "第1款");
```

**提示词设计**：
```
你是一个法律文档检索助手。

【参考文档】
{documents}

【用户问题】
{question}

【严格规则】
1. 只能使用参考文档中的原文
2. 必须标注完整出处（法律名称、条款、款）
3. 格式：根据《{law_name}》第{article}条{paragraph}款："{content}"
4. 如果文档中没有答案，必须说"未找到相关法律条文"
5. 不要添加任何解释或推测

请输出：
```

**检索配置**：
```java
SearchRequest request = SearchRequest.builder()
    .query(question)
    .topK(3)  // 少而精
    .similarityThreshold(0.85)  // 高相似度阈值
    .build();
```

</details>

---

**Q12**: 用户反馈 RAG 系统回答不够准确，如何排查和优化？

<details>
<summary>点击查看答案</summary>

**答案**：
**排查步骤**：

1. **检查检索质量**
   ```java
   // 查看检索结果的相关性
   List<Document> results = search(question, ragTag);
   for (Document doc : results) {
       log.info("Score: {}, Content: {}", doc.getScore(), doc.getContent());
   }
   ```

2. **分析问题类型**
   - 检索不到 → 检查切片、Embedding 质量
   - 检索到了但没用 → 检查提示词
   - 答案不准确 → 检查 LLM 配置

3. **优化方案**

   | 问题 | 优化方案 |
   |-----|---------|
   | 检索不到相关文档 | 调整切片大小、增加 TopK、优化元数据过滤 |
   | 检索结果不相关 | 使用更好的 Embedding 模型、查询改写 |
   | LLM 没用检索结果 | 优化提示词、降低 Temperature |
   | 答案不准确 | 添加重排序、增加示例 |

4. **A/B 测试**
   - 部署新版本
   - 收集用户反馈
   - 对比效果

</details>

---

**Q13**: 如何实现多语言 RAG 系统？

<details>
<summary>点击查看答案</summary>

**答案**：
**方案 1：多语言 Embedding**
```java
// 使用支持多语言的 Embedding 模型
EmbeddingModel embeddingModel = new OpenAiEmbeddingModel(
    OpenAiEmbeddingOptions.builder()
        .withModel("text-embedding-3-large")  // 支持 100+ 语言
        .build()
);
```

**方案 2：翻译后检索**
```java
// 1. 检测语言
String lang = detectLanguage(question);

// 2. 翻译成英文（如果有英文文档）
if (!lang.equals("en") && hasEnglishDocs) {
    question = translateToEnglish(question);
}

// 3. 检索
List<Document> results = search(question, ragTag);

// 4. 翻译结果回原语言
if (!lang.equals("en")) {
    for (Document doc : results) {
        doc.setContent(translateFromEnglish(doc.getContent()));
    }
}
```

**方案 3：语言隔离**
```java
// 不同语言的文档分别存储
ragService.uploadTextFile("docs_zh", chineseFiles);
ragService.uploadTextFile("docs_en", englishFiles);

// 检索时选择对应语言
Filter.Expression expression = builder
    .eq("knowledge", "docs_" + detectLanguage(question))
    .build();
```

**推荐**：方案 1（多语言 Embedding）最简单有效

</details>

---

## 十、关键代码索引

| 功能 | 文件 | 位置 |
|-----|------|------|
| RAG 服务入口 | RagService.java | backend/ai-agent-domain/src/main/java/com/dasi/domain/rag/service/ |
| 文件上传 | RagService.uploadTextFile() | backend/ai-agent-domain/src/main/java/com/dasi/domain/rag/service/ |
| Git 仓库导入 | RagService.uploadGitRepo() | backend/ai-agent-domain/src/main/java/com/dasi/domain/rag/service/ |
| RAG 增强 | AugmentService.augmentRagMessage() | backend/ai-agent-domain/src/main/java/com/dasi/domain/augment/service/ |
| 向量存储配置 | PgVectorStoreConfig.java | backend/ai-agent-app/src/main/java/com/dasi/config/ |

---

**文档版本**：v1.0
**更新日期**：2026-02-24
**下一篇**：专题四 - MCP 工具系统深度解析
