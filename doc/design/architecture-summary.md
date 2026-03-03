# 多租户 RAG 系统架构设计总结

## 项目背景

为 Universal Agent Backend 系统设计一个多租户的 RAG (Retrieval-Augmented Generation) 系统，使每个用户拥有自己独立的知识库。

## 关键架构决策

### 1. 向量存储方案
**选择：独立向量数据库（Pinecone 或 Weaviate）**

**理由：**
- 高性能和可扩展性
- 专业的向量搜索能力
- 通过元数据过滤实现用户隔离
- 适合大规模和高并发场景

**实现要点：**
- 向量嵌入存储在向量数据库中
- PostgreSQL 只保留元数据和 vector_id 引用
- 元数据包含：`user_id`, `knowledge_base_id`, `document_id`, `content`
- 通过元数据过滤实现用户隔离

### 2. 知识库组织方式
**选择：多个知识库**

**理由：**
- 用户可以按主题组织文档（工作、学习、个人等）
- 更灵活的数据管理
- 更好的搜索相关性

**实现要点：**
- 每个用户可以创建多个知识库
- 提供完整的 CRUD API
- 支持知识库的创建、查询、删除

### 3. 文档存储位置
**选择：本地文件系统**

**理由：**
- 实现简单快速
- 适合开发和小规模部署
- 成本低

**实现要点：**
- 原始文档存储在服务器本地磁盘
- 使用 UUID 文件名避免冲突
- 按用户 ID 分目录存储

### 4. 文档处理方式
**选择：异步处理**

**理由：**
- 更好的用户体验（立即返回）
- 可以处理大文件
- 支持进度跟踪

**实现要点：**
- 上传后立即返回
- 后台异步处理（分块、嵌入、向量化）
- 提供状态查询端点
- 使用 asyncio 或 Celery 实现任务队列

## 核心数据模型

### KnowledgeBase（知识库）
```python
- id: UUID
- user_id: 用户ID（外键）
- name: 知识库名称
- description: 描述
- embedding_model: 嵌入模型
- chunk_size: 分块大小
- chunk_overlap: 分块重叠
- is_active: 是否激活
- created_at, updated_at: 时间戳
```

### Document（文档）
```python
- id: UUID
- knowledge_base_id: 所属知识库
- user_id: 所属用户（冗余字段）
- title: 文档标题
- file_name: 原始文件名
- file_type: 文件类型
- file_size: 文件大小
- file_path: 存储路径
- chunk_count: 分块数量
- status: 处理状态（processing/completed/failed）
- error_message: 错误信息
- metadata: 元数据
- created_at, updated_at: 时间戳
```

### DocumentChunk（文档分块）
```python
- id: UUID
- document_id: 所属文档
- user_id: 所属用户（关键隔离字段）
- knowledge_base_id: 所属知识库
- chunk_index: 在文档中的位置
- content: 文本内容
- vector_id: 向量数据库中的ID
- metadata: 元数据（页码、章节等）
- created_at: 时间戳
```

## 用户隔离策略

### 三层安全架构

**第一层：数据库级隔离**
- 所有表都有 `user_id` 字段
- 外键约束防止跨用户数据访问
- `ON DELETE CASCADE` 确保数据清理

**第二层：应用级授权**
- 所有 CRUD 操作验证 `user_id` 权限
- 遵循 `get_by_id_and_user(id, user_id)` 模式

**第三层：MCP 工具上下文传递**
- 将 `user_id` 从 JWT 传递到 MCP 工具参数
- RAG-MCP-Server 在每次操作时验证用户上下文

## 向量搜索实现

### 搜索流程
1. 生成查询向量（使用智谱 AI embedding-3）
2. 在向量数据库中搜索（带 user_id 过滤）
3. 从 PostgreSQL 获取完整元数据
4. 返回结果

### Pinecone 搜索示例
```python
results = index.query(
    vector=query_vector,
    filter={
        "user_id": {"$eq": user_id},
        "knowledge_base_id": {"$eq": knowledge_base_id}
    },
    top_k=5,
    include_metadata=True,
)
```

## 架构职责划分

### Universal Agent Backend (REST API)

**负责：文档和知识库的 CRUD 操作**

- POST /api/v1/knowledge-bases - 创建知识库
- GET /api/v1/knowledge-bases - 列出知识库
- GET /api/v1/knowledge-bases/{id} - 获取知识库详情
- DELETE /api/v1/knowledge-bases/{id} - 删除知识库
- POST /api/v1/documents/upload - 上传文档
- GET /api/v1/documents - 列出文档
- GET /api/v1/documents/{id}/status - 查询处理状态
- DELETE /api/v1/documents/{id} - 删除文档

**特点：**
- 前端直接调用，无需经过 Agent
- 简单的权限控制（JWT + user_id）
- 快速响应（除了文档上传是异步处理）

### RAG-MCP-Server (MCP 工具)

**负责：智能语义搜索**

**只有一个 MCP 工具：**
- `search_knowledge_base(user_id, knowledge_base_id, query, top_k)`

**特点：**
- Agent 在需要时调用
- 自动注入 user_id 确保隔离
- 返回语义相关的文档片段

### 用户上下文传递流程

```
┌─────────────────────────────────────────────────────────────┐
│  场景 1：文档管理（前端 → REST API）                         │
├─────────────────────────────────────────────────────────────┤
│  前端 → POST /api/v1/documents/upload                       │
│         Headers: Authorization: Bearer <jwt>                │
│         FormData: {file, knowledge_base_id}                 │
│                                                              │
│  Backend → 提取 user_id 从 JWT                               │
│         → 验证知识库所有权                                     │
│         → 保存文件                                           │
│         → 创建文档记录                                        │
│         → 启动异步处理                                        │
│         → 立即返回 {document_id, status}                     │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  场景 2：智能搜索（用户 → Agent → MCP）                      │
├─────────────────────────────────────────────────────────────┤
│  用户 → "我的文档中关于 XXX 的内容"                           │
│                                                              │
│  前端 → POST /api/v1/chat/agent/stream                      │
│         Headers: Authorization: Bearer <jwt>                │
│         Body: {message}                                      │
│                                                              │
│  Backend → 提取 user_id                                       │
│         → Agent 分析意图                                      │
│         → 决定需要搜索知识库                                  │
│         → 调用 search_knowledge_base(user_id, kb_id, query)  │
│                                                              │
│  RAG-MCP-Server → 验证 user_id                                │
│                  → 向量搜索（带 user_id 过滤）                 │
│                  → 返回相关片段                               │
│                                                              │
│  Agent → 基于搜索结果生成回答                                 │
│        → 流式返回给用户                                       │
└─────────────────────────────────────────────────────────────┘
```

## API 端点设计

### 知识库管理
- POST /api/v1/knowledge-bases - 创建知识库
- GET /api/v1/knowledge-bases - 列出用户的知识库
- GET /api/v1/knowledge-bases/{id} - 获取知识库详情
- DELETE /api/v1/knowledge-bases/{id} - 删除知识库

### 文档管理
- POST /documents/upload - 上传文档
- GET /documents/{id}/status - 获取处理状态

## 技术栈

- **关系数据库**: PostgreSQL 16 (存储元数据)
- **向量数据库**: Pinecone 或 Weaviate (存储向量嵌入)
- **文件存储**: 本地文件系统
- **嵌入模型**: 智谱 AI embedding-3 (1024 维)
- **MCP 框架**: FastMCP
- **异步框架**: asyncio + SQLAlchemy 2.0
- **任务队列**: asyncio 任务或 Celery
- **文档处理**: LangChain + Unstructured
- **前端**: nuomi-frontend (React)

## 实现步骤

### 阶段 1：数据库基础 (1-2 周)
1. 创建数据库模型
2. 编写迁移脚本
3. 实现 CRUD 操作
4. 创建基础 API 端点

### 阶段 2：文档处理 (2-3 周)
1. 实现文件上传端点
2. 创建文档处理服务
3. 集成智谱 AI 嵌入 API
4. 实现异步任务队列
5. 实现向量搜索

### 阶段 3：MCP 集成 (2-3 周)
1. 实现 RAG-MCP-Server
2. 创建 MCP 工具
3. 增强 Agent Graph
4. 测试端到端流程

### 阶段 4：前端集成 (2 周)
1. 创建知识库管理页面
2. 添加文档上传组件
3. 显示搜索结果
4. 添加进度跟踪

## 安全考虑

1. **所有查询都必须包含 user_id 过滤**
2. **MCP 工具必须验证 user_id 参数**
3. **文件上传验证**：限制大小 (50MB)、文件类型
4. **路径遍历防护**：使用 UUID 文件名
5. **速率限制**：上传和搜索端点

## 后续优化

- 文档集合/标签
- 混合搜索（语义 + 关键词）
- 文档版本控制
- 用户间共享知识库
- 高级分块策略
