# 多租户 RAG 知识库系统实现计划

## 上下文

当前项目需要实现一个多租户的 RAG (Retrieval-Augmented Generation) 系统，使每个用户拥有自己独立的知识库。

**当前状态：**
- ✅ Universal Agent Backend 已实现 JWT 认证、会话管理、MCP 集成
- ✅ 数据库已有 users、sessions、messages 表
- ❌ 缺少知识库相关表结构
- ❌ RAG-MCP-Server 仅有文档，尚未实现

**目标：**
- 每个用户可以上传、管理、搜索自己的文档
- 所有数据严格按 user_id 隔离
- 通过 MCP 协议与现有 Agent 系统集成

---

## 架构设计

### 1. 数据库架构

在 `Universal-Agent-Backend/app/models/database.py` 中新增三个模型：

#### 1.1 KnowledgeBase (知识库表)
```python
class KnowledgeBase(Base):
    __tablename__ = "knowledge_bases"

    id: str                    # UUID 主键
    user_id: str               # 所属用户 (外键 → users.id, ON DELETE CASCADE)
    name: str                  # 知识库名称
    description: str | None    # 描述
    embedding_model: str       # 使用的嵌入模型
    chunk_size: int            # 文档分块大小
    chunk_overlap: int         # 分块重叠
    is_active: bool            # 是否激活
    created_at: datetime
    updated_at: datetime

    # 关系
    user: User
    documents: list[Document]
```

#### 1.2 Document (文档表)
```python
class Document(Base):
    __tablename__ = "documents"

    id: str                      # UUID 主键
    knowledge_base_id: str       # 所属知识库
    user_id: str                 # 所属用户 (冗余字段，用于快速查询)
    title: str                   # 文档标题
    file_name: str               # 原始文件名
    file_type: str               # 文件类型 (pdf/txt/md/docx)
    file_size: int               # 文件大小 (字节)
    file_path: str               # 存储路径
    chunk_count: int             # 分块数量
    status: str                  # 处理状态 (processing/completed/failed)
    error_message: str | None    # 错误信息
    metadata: dict | None        # 元数据
    created_at: datetime
    updated_at: datetime

    # 关系
    knowledge_base: KnowledgeBase
    chunks: list[DocumentChunk]
```

#### 1.3 DocumentChunk (文档分块表)
```python
class DocumentChunk(Base):
    __tablename__ = "document_chunks"

    id: str                      # UUID 主键
    document_id: str             # 所属文档
    user_id: str                 # 所属用户 (关键字段，用于用户隔离)
    knowledge_base_id: str       # 所属知识库
    chunk_index: int             # 在文档中的位置
    content: str                 # 文本内容
    vector_id: str               # 向量数据库中的 ID (Pinecone/Weaviate)
    metadata: dict | None        # 元数据 (页码、章节等)
    created_at: datetime

    # 关系
    document: Document

    # 索引
    # Index: idx_chunks_user_vector (user_id, vector_id)
```

**注意：** 向量嵌入存储在独立的向量数据库中，PostgreSQL 只保留元数据和 vector_id 引用。

**关键点：**
- 所有表都有 `user_id` 字段，确保数据隔离
- 使用 `ON DELETE CASCADE` 级联删除
- `DocumentChunk.user_id` 是用户隔离的核心字段
- **向量嵌入存储在独立向量数据库**（Pinecone/Weaviate），PostgreSQL 保留元数据
- 每个用户可以有**多个知识库**，按主题组织文档
- 原始文件存储在**本地文件系统**

---

### 2. 用户隔离策略

#### 2.1 三层安全架构

**第一层：数据库级隔离**
- 所有查询必须包含 `WHERE user_id = ?`
- 外键约束防止跨用户数据访问
- 向量索引包含 `user_id` 过滤

**第二层：应用级授权**
- 创建 `app/core/crud/knowledge_base.py`
- 所有 CRUD 操作验证 `user_id` 权限
- 遵循 `SessionCRUD.get_by_id_and_user()` 模式

**第三层：MCP 工具上下文传递**
- 将 `user_id` 从 JWT 传递到 MCP 工具参数
- RAG-MCP-Server 在每次操作时验证用户上下文

#### 2.2 CRUD 操作模式

所有操作遵循 `get_by_id_and_user(id, user_id)` 模式：

```python
async def get_by_id_and_user(
    db: AsyncSession,
    kb_id: str,
    user_id: str,
) -> KnowledgeBase | None:
    """仅当知识库属于用户时才返回"""
    result = await db.execute(
        select(KnowledgeBase).where(
            and_(
                KnowledgeBase.id == kb_id,
                KnowledgeBase.user_id == user_id
            )
        )
    )
    return result.scalar_one_or_none()
```

---

### 3. 架构职责划分

#### 3.1 职责分离原则

**重要架构决策：将 CRUD 操作和搜索能力分离到不同的服务层**

```
┌─────────────────────────────────────────────────────────┐
│                      前端 (nuomi-frontend)               │
│  - 知识库管理界面                                         │
│  - 文档上传组件                                           │
│  - 文档列表显示                                           │
└────────────┬────────────────────────────┬───────────────┘
             │ REST API                  │ SSE/HTTP
             │ (CRUD 操作)               │ (聊天流)
             ▼                           ▼
┌─────────────────────────────────────────────────────────────────┐
│              Universal Agent Backend                            │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ REST API 端点 (文档管理)                                    │ │
│  │  - POST   /api/v1/knowledge-bases          创建知识库        │ │
│  │  - GET    /api/v1/knowledge-bases          列出知识库        │ │
│  │  - GET    /api/v1/knowledge-bases/{id}     获取知识库        │ │
│  │  - DELETE /api/v1/knowledge-bases/{id}     删除知识库        │ │
│  │  - POST   /api/v1/documents/upload         上传文档          │ │
│  │  - GET    /api/v1/documents                列出文档          │ │
│  │  - GET    /api/v1/documents/{id}/status    文档状态          │ │
│  │  - DELETE /api/v1/documents/{id}           删除文档          │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Agent Graph (聊天编排)                                      │ │
│  │  - 接收用户查询                                             │ │
│  │  - 决定是否需要搜索知识库                                   │ │
│  │  - 调用 MCP 工具进行搜索                                    │ │
│  │  - 生成最终回答                                             │ │
│  └────────────────────────────────────────────────────────────┘ │
└──────────────────────┬──────────────────────────────────────────┘
                       │ MCP Protocol
                       │ (仅用于搜索)
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                    RAG-MCP-Server                               │
│  MCP Tool:                                                       │
│    search_knowledge_base(user_id, kb_id, query, top_k)          │
│                                                                 │
│  职责：                                                          │
│    - 向量搜索                                                   │
│    - 语义匹配                                                   │
│    - 返回相关文档片段                                            │
└─────────────────────────────────────────────────────────────────┘
```

**职责划分理由：**

1. **前端 → Universal Agent Backend (REST API)**
   - 直接的 CRUD 操作
   - 更快的响应时间
   - 简单的权限控制
   - 文件上传、列表展示等基础功能

2. **Agent → RAG-MCP-Server (MCP)**
   - 智能搜索能力
   - Agent 可以决定何时搜索
   - 上下文感知的查询
   - 语义理解和匹配

#### 3.2 Universal Agent Backend REST API

在 `Universal-Agent-Backend/app/api/v1/` 中实现：

**知识库管理** (`knowledge_bases.py`):
```python
@router.post("/api/v1/knowledge-bases")
async def create_knowledge_base(
    name: str,
    description: Optional[str] = None,
    current_user: dict = Depends(get_current_user),
):
    """创建新知识库"""
    user_id = current_user["user_id"]
    kb = await KnowledgeBaseCRUD.create_for_user(db, user_id=user_id, name=name)
    return {"id": kb.id, "name": kb.name}

@router.get("/api/v1/knowledge-bases")
async def list_knowledge_bases(current_user: dict = Depends(get_current_user)):
    """列出用户的所有知识库"""
    user_id = current_user["user_id"]
    kbs = await KnowledgeBaseCRUD.list_for_user(db, user_id)
    return [{"id": kb.id, "name": kb.name, "document_count": len(kb.documents)} for kb in kbs]

@router.delete("/api/v1/knowledge-bases/{kb_id}")
async def delete_knowledge_base(kb_id: str, current_user: dict = Depends(get_current_user)):
    """删除知识库"""
    user_id = current_user["user_id"]
    kb = await KnowledgeBaseCRUD.get_by_id_and_user(db, kb_id, user_id)
    if not kb:
        raise HTTPException(404, "知识库不存在")
    # 删除向量数据库中的所有向量
    for doc in kb.documents:
        await vector_store.delete_by_document(doc.id)
    await db.delete(kb)
    return {"message": "已删除"}
```

**文档管理** (`documents.py`):
```python
@router.post("/api/v1/documents/upload")
async def upload_document(
    knowledge_base_id: str,
    file: UploadFile = File(...),
    current_user: dict = Depends(get_current_user),
):
    """上传文档到知识库"""
    user_id = current_user["user_id"]

    # 验证知识库所有权
    kb = await KnowledgeBaseCRUD.get_by_id_and_user(db, knowledge_base_id, user_id)
    if not kb:
        raise HTTPException(403, "无权访问")

    # 保存文件
    file_path = f"{UPLOAD_DIR}/{user_id}/{uuid.uuid4()}_{file.filename}"

    # 创建文档记录
    document = Document(
        knowledge_base_id=knowledge_base_id,
        user_id=user_id,
        title=file.filename,
        file_path=file_path,
        status="processing",
    )
    db.add(document)
    await db.commit()

    # 异步处理文档
    asyncio.create_task(
        document_processor.process_document_async(
            document_id=document.id,
            user_id=user_id,
            knowledge_base_id=knowledge_base_id,
            file_path=file_path,
        )
    )

    return {"document_id": document.id, "status": "processing"}

@router.get("/api/v1/documents")
async def list_documents(
    knowledge_base_id: str,
    current_user: dict = Depends(get_current_user),
):
    """列出知识库中的所有文档"""
    user_id = current_user["user_id"]
    docs = await DocumentCRUD.list_by_kb_and_user(db, knowledge_base_id, user_id)
    return [
        {
            "id": d.id,
            "title": d.title,
            "status": d.status,
            "chunk_count": d.chunk_count,
        }
        for d in docs
    ]

@router.get("/api/v1/documents/{document_id}/status")
async def get_document_status(
    document_id: str,
    current_user: dict = Depends(get_current_user),
):
    """查询文档处理状态"""
    user_id = current_user["user_id"]
    doc = await DocumentCRUD.get_by_id_and_user(db, document_id, user_id)
    if not doc:
        raise HTTPException(404, "文档不存在")
    return {
        "document_id": doc.id,
        "status": doc.status,
        "chunk_count": doc.chunk_count,
        "error_message": doc.error_message,
    }
```

#### 3.3 RAG-MCP-Server MCP 工具（仅搜索）

在 `RAG-MCP-Server/src/tools/rag_tools.py` 中**只保留搜索工具**：

```python
from fastmcp import FastMCP

mcp = FastMCP("RAG Knowledge Base Search")

@mcp.tool()
async def search_knowledge_base(
    user_id: str,              # 必需：用户隔离
    knowledge_base_id: str,
    query: str,
    top_k: int = 5,
) -> list[dict]:
    """在用户的知识库中进行语义搜索

    这是唯一的 MCP 工具，专门用于 Agent 搜索知识库

    Args:
        user_id: 用户 ID（从 Agent 上下文传入）
        knowledge_base_id: 知识库 ID
        query: 搜索查询
        top_k: 返回结果数量

    Returns:
        相关文档片段列表
    """
    # 向量搜索 + user_id 过滤
    results = await search_service.semantic_search(
        knowledge_base_id=knowledge_base_id,
        user_id=user_id,  # 关键：用户隔离
        query=query,
        top_k=top_k
    )

    return [
        {
            "content": r.content,
            "document_title": r.document.title,
            "document_id": r.document_id,
            "score": r.score,
            "metadata": r.metadata
        }
        for r in results
    ]
```

#### 3.4 用户上下文传递流程（仅搜索）

```
┌─────────────────────────────────────────────────────────────┐
│  用户聊天流程                                                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. 前端发送聊天请求                                          │
│     POST /api/v1/chat/agent/stream                           │
│     Headers: Authorization: Bearer <jwt_token>               │
│     Body: { "message": "我的文档中有什么内容？" }             │
│                                                              │
│  2. Backend 提取 user_id                                      │
│     user_id = get_current_user()["user_id"]                  │
│                                                              │
│  3. Agent Graph 执行                                          │
│     - 分析用户意图                                            │
│     - 决定是否需要搜索知识库                                  │
│     - 如果需要，调用 search_knowledge_base                    │
│                                                              │
│  4. 注入 user_id 到搜索工具                                   │
│     tool_args["user_id"] = user_id                           │
│     tool_args["knowledge_base_id"] = selected_kb_id          │
│     tool_args["query"] = "我的文档中有什么内容？"              │
│                                                              │
│  5. RAG-MCP-Server 搜索                                       │
│     - 验证 user_id                                            │
│     - 向量搜索（带 user_id 过滤）                              │
│     - 返回相关片段                                            │
│                                                              │
│  6. Agent 生成回答                                            │
│     基于搜索结果生成回答                                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

在 `app/graphs/agent.py` 中修改 `call_tool_node`：

```python
async def call_tool_node(state: AgentState, config: dict):
    """执行工具调用，注入用户上下文"""
    messages = state["messages"]
    last_message = messages[-1]

    if not isinstance(last_message, AIMessage) or not last_message.tool_calls:
        return {"messages": []}

    # 从 config 中获取 user_id（在调用 agent 时传入）
    user_id = config.get("configurable", {}).get("user_id")

    tool_messages = []
    for tool_call in last_message.tool_calls:
        tool_name = tool_call["name"]
        tool_args = tool_call["args"]

        # 将 user_id 注入到 RAG 搜索工具
        if tool_name == "search_knowledge_base":
            tool_args["user_id"] = user_id
            # 如果用户没有指定知识库，使用默认知识库
            if "knowledge_base_id" not in tool_args:
                tool_args["knowledge_base_id"] = await get_default_kb(user_id)

        # 执行工具
        tool = next((t for t in tools if t.name == tool_name), None)
        if tool:
            result = await tool.ainvoke(tool_args)
            tool_messages.append(
                ToolMessage(
                    content=str(result),
                    tool_call_id=tool_call["id"],
                    name=tool_name,
                )
            )

    return {"messages": tool_messages}
```

**关键点：**
- 只有 `search_knowledge_base` 需要 user_id 注入
- 文档上传、列表等操作通过 REST API 直接调用，不需要经过 Agent
- Agent 只在需要智能搜索时才调用 MCP 工具

---

### 4. 向量搜索实现

使用独立向量数据库（Pinecone/Weaviate）实现用户范围的语义搜索：

#### 4.1 向量数据库客户端

创建 `RAG-MCP-Server/src/services/vector_store.py`：

```python
from pinecone import Pinecone, ServerlessSpec

class VectorStoreService:
    """向量数据库服务，支持用户隔离"""

    def __init__(self, api_key: str, index_name: str):
        self.pc = Pinecone(api_key=api_key)
        self.index = self.pc.Index(index_name)

    async def upsert_chunks(
        self,
        chunks: list[dict],
    ) -> list[str]:
        """批量插入向量，包含用户元数据用于过滤

        Args:
            chunks: [
                {
                    "id": "chunk_uuid",
                    "vector": [0.1, 0.2, ...],
                    "metadata": {
                        "user_id": "user123",        # 关键：用于过滤
                        "knowledge_base_id": "kb456",
                        "document_id": "doc789",
                        "content": "chunk text...",
                    }
                },
                ...
            ]
        """
        return self.index.upsert(vectors=chunks)

    async def search(
        self,
        query_vector: list[float],
        user_id: str,
        knowledge_base_id: str,
        top_k: int = 5,
    ) -> list[dict]:
        """用户隔离的向量搜索

        关键：在向量数据库层面通过元数据过滤实现用户隔离
        """
        results = self.index.query(
            vector=query_vector,
            filter={
                "user_id": {"$eq": user_id},              # 关键：用户隔离
                "knowledge_base_id": {"$eq": knowledge_base_id}
            },
            top_k=top_k,
            include_metadata=True,
        )

        return results["matches"]
```

#### 4.2 搜索服务

创建 `RAG-MCP-Server/src/services/search.py`：

```python
async def semantic_search(
    knowledge_base_id: str,
    user_id: str,
    query: str,
    top_k: int = 5,
) -> list[dict]:
    """用户隔离的语义搜索"""

    # 1. 生成查询向量
    query_embedding = await embedding_service.embed(query)

    # 2. 向量数据库搜索 (带用户过滤)
    vector_results = await vector_store.search(
        query_vector=query_embedding,
        user_id=user_id,              # 关键：用户隔离
        knowledge_base_id=knowledge_base_id,
        top_k=top_k
    )

    # 3. 从 PostgreSQL 获取完整元数据
    chunk_ids = [r["id"] for r in vector_results]
    chunks = await db.execute(
        select(DocumentChunk).where(
            and_(
                DocumentChunk.id.in_(chunk_ids),
                DocumentChunk.user_id == user_id  # 二次验证
            )
        )
    )

    return list(chunks.scalars().all())
```

**关键点：**
- 在向量数据库层面通过 `filter` 元数据实现用户隔离
- PostgreSQL 中保留 `user_id` 作为二次验证
- 向量数据库的命名空间或元数据过滤确保用户不能搜索其他用户的向量

---

### 5. 数据库迁移

创建 `Universal-Agent-Backend/migrations/add_knowledge_base_tables.sql`：

```sql
-- 创建 knowledge_bases 表
CREATE TABLE knowledge_bases (
    id VARCHAR(36) PRIMARY KEY,
    user_id VARCHAR(36) NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    embedding_model VARCHAR(50) DEFAULT 'zhipu-embedding-3',
    chunk_size INTEGER DEFAULT 1000,
    chunk_overlap INTEGER DEFAULT 200,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_knowledge_bases_user ON knowledge_bases(user_id);

-- 创建 documents 表
CREATE TABLE documents (
    id VARCHAR(36) PRIMARY KEY,
    knowledge_base_id VARCHAR(36) NOT NULL REFERENCES knowledge_bases(id) ON DELETE CASCADE,
    user_id VARCHAR(36) NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(500) NOT NULL,
    file_name VARCHAR(255) NOT NULL,
    file_type VARCHAR(50) NOT NULL,
    file_size INTEGER,
    file_path VARCHAR(500) NOT NULL,
    chunk_count INTEGER DEFAULT 0,
    status VARCHAR(20) DEFAULT 'processing',
    error_message TEXT,
    metadata JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_documents_user_status ON documents(user_id, status);
CREATE INDEX idx_documents_kb_status ON documents(knowledge_base_id, status);

-- 创建 document_chunks 表 (元数据表，向量存储在独立数据库)
CREATE TABLE document_chunks (
    id VARCHAR(36) PRIMARY KEY,
    document_id VARCHAR(36) NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    user_id VARCHAR(36) NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    knowledge_base_id VARCHAR(36) NOT NULL REFERENCES knowledge_bases(id) ON DELETE CASCADE,
    chunk_index INTEGER NOT NULL,
    content TEXT NOT NULL,
    vector_id VARCHAR(255) NOT NULL,  -- 向量数据库中的 ID
    metadata JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 创建索引
CREATE INDEX idx_chunks_user_vector ON document_chunks(user_id, vector_id);
CREATE INDEX idx_chunks_kb_user ON document_chunks(knowledge_base_id, user_id);
CREATE INDEX idx_chunks_document ON document_chunks(document_id, chunk_index);
```

**注意：** 不再需要 pgvector 扩展，因为向量数据存储在独立向量数据库中。

---

## 实现步骤

### 阶段 1：数据库基础 (1-2 周)

**优先级 1：数据库设置**
- [ ] 在 `Universal-Agent-Backend/app/models/database.py` 中创建三个新模型
- [ ] 编写数据库迁移脚本 `migrations/add_knowledge_base_tables.sql`
- [ ] 测试数据库约束和级联删除
- [ ] **注意：不再需要 pgvector 扩展**

**优先级 2：CRUD 操作**
- [ ] 创建 `app/core/crud/knowledge_base.py`
- [ ] 实现 KnowledgeBaseCRUD、DocumentCRUD 类
- [ ] 所有方法都遵循 `get_by_id_and_user(id, user_id)` 模式
- [ ] 编写单元测试验证所有权检查

**优先级 3：REST API 端点（文档管理）**
- [ ] 创建 `app/api/v1/knowledge_bases.py`
  - [ ] POST /api/v1/knowledge-bases (创建知识库)
  - [ ] GET /api/v1/knowledge-bases (列出用户的知识库)
  - [ ] GET /api/v1/knowledge-bases/{id} (获取知识库详情)
  - [ ] DELETE /api/v1/knowledge-bases/{id} (删除知识库)

- [ ] 创建 `app/api/v1/documents.py`
  - [ ] POST /api/v1/documents/upload (上传文档)
  - [ ] GET /api/v1/documents (列出文档)
  - [ ] GET /api/v1/documents/{id}/status (查询处理状态)
  - [ ] DELETE /api/v1/documents/{id} (删除文档)

---

### 阶段 2：文档处理 (2-3 周)

**优先级 4：文档上传管道**
- [ ] 实现文件上传端点，带验证 (大小 50MB、类型 pdf/txt/md/docx)
- [ ] 创建 `app/services/document_processor.py`
  - [ ] 异步文档处理服务
  - [ ] 文本分块（LangChain RecursiveCharacterTextSplitter）
  - [ ] 集成智谱 AI embedding API
  - [ ] 支持多种文件格式（PDF、TXT、MD、DOCX）

- [ ] 文件存储
  - [ ] 本地文件系统存储
  - [ ] UUID 文件名避免冲突
  - [ ] 按用户 ID 分目录

**优先级 5：向量搜索**
- [ ] 创建向量数据库服务
  - [ ] Pinecone 或 Weaviate 客户端封装
  - [ ] upsert_chunks、delete_by_document、search 方法

- [ ] 创建搜索服务 `RAG-MCP-Server/src/services/search.py`
  - [ ] semantic_search 函数
  - [ ] 生成查询向量
  - [ ] 向量搜索（带 user_id 过滤）
  - [ ] 从 PostgreSQL 获取完整元数据

- [ ] 测试搜索准确性
  - [ ] 使用示例文档测试
  - [ ] 验证用户隔离

---

### 阶段 3：MCP 集成 (1-2 周)

**优先级 6：RAG-MCP-Server 实现**
- [ ] 初始化 RAG-MCP-Server 项目结构
- [ ] 使用 FastMCP 创建 MCP 服务器
- [ ] **只实现一个 MCP 工具**: `search_knowledge_base`
  - [ ] 必需参数：user_id, knowledge_base_id, query, top_k
  - [ ] 用户隔离验证
  - [ ] 返回格式化的搜索结果

- [ ] 配置环境变量
  - [ ] Pinecone 或 Weaviate 连接信息
  - [ ] 智谱 AI API key

- [ ] 测试与 Universal Agent Backend 的集成
  - [ ] MCP 工具注册
  - [ ] 端到端搜索测试

**优先级 7：Agent Graph 增强**
- [ ] 修改 `app/graphs/agent.py` 中的 `call_tool_node`
  - [ ] 从 config 中提取 user_id
  - [ ] 将 user_id 注入到 search_knowledge_base 调用
  - [ ] 处理默认知识库选择

- [ ] 测试端到端流程
  - [ ] 用户上传文档 → 处理完成 → Agent 搜索 → 返回结果
  - [ ] 验证用户隔离（用户A不能搜索用户B的文档）

---

### 阶段 4：前端集成 (2 周)

**优先级 8：前端 UI**
- [ ] 知识库管理页面
  - [ ] 创建/删除知识库
  - [ ] 列出所有知识库

- [ ] 文档管理组件
  - [ ] 文档上传（支持拖拽）
  - [ ] 文档列表（显示处理状态）
  - [ ] 删除文档
  - [ ] 进度跟踪（处理中/完成/失败）

- [ ] 聊天界面集成
  - [ ] Agent 自动调用搜索
  - [ ] 显示搜索结果引用

**优先级 9：测试与优化**
- [ ] 跨所有层的集成测试
- [ ] 大文档的性能测试（10MB+ PDF）
- [ ] 隔离绕过的安全审计
- [ ] 并发用户的负载测试
- [ ] 错误处理和重试机制

---

## 关键文件清单

需要创建或修改的关键文件：

### Universal Agent Backend（REST API + Agent 编排）

1. **`app/models/database.py`** (修改)
   - 添加 KnowledgeBase、Document、DocumentChunk 模型
   - 在 User 模型中添加 relationship

2. **`app/core/crud/knowledge_base.py`** (新建)
   - 实现 KnowledgeBaseCRUD、DocumentCRUD 类
   - 所有方法都带 user_id 验证

3. **`app/api/v1/knowledge_bases.py`** (新建)
   - POST /api/v1/knowledge-bases - 创建知识库
   - GET /api/v1/knowledge-bases - 列出知识库
   - GET /api/v1/knowledge-bases/{id} - 获取知识库
   - DELETE /api/v1/knowledge-bases/{id} - 删除知识库

4. **`app/api/v1/documents.py`** (新建)
   - POST /api/v1/documents/upload - 上传文档
   - GET /api/v1/documents - 列出文档
   - GET /api/v1/documents/{id}/status - 查询状态
   - DELETE /api/v1/documents/{id} - 删除文档

5. **`app/graphs/agent.py`** (修改)
   - 修改 `call_tool_node` 注入 user_id 到 search_knowledge_base

6. **`app/services/document_processor.py`** (新建)
   - 异步文档处理服务
   - 分块、嵌入、向量化

### RAG-MCP-Server（仅搜索能力）

7. **`src/tools/rag_tools.py`** (新建)
   - **只有一个 MCP 工具**: search_knowledge_base
   - 必需参数：user_id, knowledge_base_id, query

8. **`src/services/vector_store.py`** (新建)
   - Pinecone 或 Weaviate 客户端
   - upsert_chunks、delete_by_document、search

9. **`src/services/search.py`** (新建)
   - semantic_search 函数
   - 生成查询向量 + 向量搜索 + 获取元数据

### 数据库

10. **`UAF-Data-Base/docker/init-db.sql`** (修改)
    - 创建新表和索引
    - 注意：不再需要 pgvector 扩展

11. **`Universal-Agent-Backend/migrations/add_knowledge_base_tables.sql`** (新建)
    - 数据库迁移脚本

---

## 验证计划

### 测试用户隔离

```python
# 测试：用户只能搜索自己的知识库
async def test_user_isolation():
    # user1 创建知识库并上传文档
    kb1 = await KnowledgeBaseCRUD.create_for_user(db, user_id="user1", name="KB1")
    doc1 = await upload_document_via_api(
        knowledge_base_id=kb1.id,
        user_id="user1",
        file="secret.pdf"
    )

    # 等待文档处理完成
    await wait_for_document_processing(doc1.id)

    # user2 尝试搜索 user1 的知识库（通过 MCP 工具）
    results = await search_knowledge_base(
        knowledge_base_id=kb1.id,
        user_id="user2",  # 不同的用户
        query="secret"
    )

    # 验证：user2 不应该看到任何结果
    assert len(results) == 0

    # user1 搜索自己的知识库
    results = await search_knowledge_base(
        knowledge_base_id=kb1.id,
        user_id="user1",
        query="secret"
    )

    # 验证：user1 应该看到结果
    assert len(results) > 0
```

### API 测试

```python
# 测试 REST API 权限
async def test_api_authorization():
    # user1 创建知识库
    response = await client.post(
        "/api/v1/knowledge-bases",
        json={"name": "My KB"},
        headers={"Authorization": f"Bearer {user1_token}"}
    )
    kb_id = response.json()["id"]

    # user2 尝试删除 user1 的知识库
    response = await client.delete(
        f"/api/v1/knowledge-bases/{kb_id}",
        headers={"Authorization": f"Bearer {user2_token}"}
    )

    # 应该返回 403 或 404
    assert response.status_code in [403, 404]
```

### 端到端测试流程

#### 场景 1：完整的文档管理和搜索流程

```
1. 用户登录
   POST /api/v1/auth/login
   → 获得 JWT token

2. 创建知识库
   POST /api/v1/knowledge-bases
   Headers: Authorization: Bearer <token>
   Body: {"name": "工作文档", "description": "我的工作资料"}
   → 返回知识库 ID

3. 上传文档
   POST /api/v1/documents/upload
   Headers: Authorization: Bearer <token>
   FormData: {knowledge_base_id, file}
   → 返回 document_id, status: "processing"

4. 查询文档状态
   GET /api/v1/documents/{document_id}/status
   Headers: Authorization: Bearer <token>
   → 轮询直到 status: "completed"

5. Agent 聊天（自动触发搜索）
   POST /api/v1/chat/agent/stream
   Headers: Authorization: Bearer <token>
   Body: {"message": "我的文档中关于 XXX 的内容"}
   → Agent 自动调用 search_knowledge_base
   → 返回基于搜索结果的回答
```

#### 场景 2：用户隔离验证

```
1. User A 上传文档到知识库 A
2. User B 尝试通过 API 访问知识库 A
   → 应该失败（403/404）

3. User B 通过 Agent 搜索知识库 A
   → 应该返回空结果

4. User A 通过 Agent 搜索知识库 A
   → 应该返回相关结果
```

### 性能测试

```python
# 测试：大文档处理
async def test_large_document():
    # 上传 50MB PDF
    doc = await upload_document(
        file="large_50mb.pdf",
        knowledge_base_id=kb.id
    )

    # 验证异步处理
    assert doc.status == "processing"

    # 等待处理完成
    await wait_for_document_processing(doc.id, timeout=300)

    # 验证分块数量
    doc = await get_document(doc.id)
    assert doc.chunk_count > 100
    assert doc.status == "completed"

# 测试：并发搜索
async def test_concurrent_searches():
    # 10 个用户同时搜索
    tasks = [
        search_knowledge_base(
            user_id=f"user{i}",
            knowledge_base_id=f"kb{i}",
            query="test"
        )
        for i in range(10)
    ]

    results = await asyncio.gather(*tasks)

    # 验证所有搜索都成功
    assert all(len(r) > 0 for r in results)
```

---

## 安全考虑

1. **所有查询都必须包含 user_id 过滤**
2. **MCP 工具必须验证 user_id 参数**
3. **文件上传验证**：限制大小 (50MB)、文件类型 (pdf/txt/md/docx)
4. **路径遍历防护**：使用 UUID 文件名，保留原始文件名在数据库
5. **速率限制**：上传和搜索端点添加速率限制

---

## 技术栈

- **关系数据库**：PostgreSQL 16 (存储元数据)
- **向量数据库**：Pinecone 或 Weaviate (存储向量嵌入)
- **文件存储**：本地文件系统
- **嵌入模型**：智谱 AI embedding-3 (1024 维)
- **MCP 框架**：FastMCP
- **异步框架**：asyncio + SQLAlchemy 2.0
- **任务队列**：Celery 或 asyncio 任务 (异步文档处理)
- **文档处理**：LangChain + Unstructured
- **前端**：nuomi-frontend (React)

**架构特点：**
- 多知识库支持：每个用户可创建多个知识库，按主题组织
- 用户隔离：在向量数据库层面通过元数据过滤实现
- 异步处理：文档上传后立即返回，后台处理并更新状态

---

### 6. 向量数据库集成

#### 6.1 Pinecone 配置

在 `RAG-MCP-Server/.env` 中添加：

```bash
# Pinecone 配置
PINECONE_API_KEY="your-pinecone-api-key"
PINECONE_ENVIRONMENT="us-west1-gcp"
PINECONE_INDEX_NAME="rag-knowledge-bases"

# 向量维度 (根据嵌入模型)
EMBEDDING_DIMENSION=1024
```

#### 6.2 向量数据库初始化

创建 `RAG-MCP-Server/src/services/vector_store.py`：

```python
import os
from pinecone import Pinecone, ServerlessSpec

class VectorStoreService:
    def __init__(self):
        self.pc = Pinecone(api_key=os.getenv("PINECONE_API_KEY"))
        self.index_name = os.getenv("PINECONE_INDEX_NAME")

        # 确保索引存在
        self._ensure_index_exists()

    def _ensure_index_exists(self):
        """创建索引（如果不存在）"""
        existing_indexes = [idx.name for idx in self.pc.list_indexes()]

        if self.index_name not in existing_indexes:
            self.pc.create_index(
                name=self.index_name,
                dimension=1024,  # 智谱 embedding-3 维度
                metric="cosine",
                spec=ServerlessSpec(
                    cloud="aws",
                    region="us-east-1"
                )
            )

        self.index = self.pc.Index(self.index_name)

    async def upsert_chunks(self, chunks: list[dict]) -> list[str]:
        """批量插入向量"""
        return self.index.upsert(vectors=chunks)

    async def delete_by_document(self, document_id: str):
        """删除文档的所有向量"""
        self.index.delete(filter={"document_id": {"$eq": document_id}})

    async def search(
        self,
        query_vector: list[float],
        user_id: str,
        knowledge_base_id: str,
        top_k: int = 5,
    ) -> list[dict]:
        """用户隔离的向量搜索"""
        results = self.index.query(
            vector=query_vector,
            filter={
                "user_id": {"$eq": user_id},
                "knowledge_base_id": {"$eq": knowledge_base_id}
            },
            top_k=top_k,
            include_metadata=True,
        )
        return results["matches"]
```

**关键设计决策：**
- 向量 ID 格式：`{chunk_uuid}` (使用 PostgreSQL 中的 chunk ID)
- 元数据包含：`user_id`, `knowledge_base_id`, `document_id`, `content`
- 通过元数据过滤实现用户隔离（而不是命名空间）
- 支持按文档删除所有相关向量

#### 6.3 Weaviate 替代方案

如果选择 Weaviate，配置类似：

```python
import weaviate

class VectorStoreService:
    def __init__(self):
        self.client = weaviate.Client(
            url=os.getenv("WEAVIATE_URL"),
            auth_client_secret=weaviate.AuthApiKey(os.getenv("WEAVIATE_API_KEY"))
        )

    async def upsert_chunks(self, chunks: list[dict]):
        """批量插入向量"""
        with self.client.batch(batch_size=100) as batch:
            for chunk in chunks:
                batch.add_data_object(
                    data_object=chunk["metadata"],
                    class_name="DocumentChunk",
                    vector=chunk["vector"]
                )

    async def search(self, query_vector, user_id, knowledge_base_id, top_k=5):
        """用户隔离的搜索"""
        results = self.client.query.get(
            "DocumentChunk",
            ["content", "document_id"]
        ).with_near_vector({
            "vector": query_vector
        }).with_where({
            "operator": "And",
            "operands": [
                {
                    "path": ["user_id"],
                    "operator": "Equal",
                    "valueText": user_id
                },
                {
                    "path": ["knowledge_base_id"],
                    "operator": "Equal",
                    "valueText": knowledge_base_id
                }
            ]
        }).with_limit(top_k).do()

        return results["data"]["Get"]["DocumentChunk"]
```

---

### 7. 异步文档处理

#### 7.1 任务队列架构

使用 asyncio 实现后台任务处理：

创建 `RAG-MCP-Server/src/services/document_processor.py`：

```python
import asyncio
from typing import Optional
from langchain.text_splitter import RecursiveCharacterTextSplitter

class DocumentProcessor:
    """异步文档处理服务"""

    def __init__(self, embedding_service, vector_store, db):
        self.embedding_service = embedding_service
        self.vector_store = vector_store
        self.db = db
        self.tasks: dict[str, asyncio.Task] = {}
        self.text_splitter = RecursiveCharacterTextSplitter(
            chunk_size=1000,
            chunk_overlap=200,
            length_function=len,
        )

    async def process_document_async(
        self,
        document_id: str,
        user_id: str,
        knowledge_base_id: str,
        file_path: str,
    ):
        """异步处理文档：分块 → 嵌入 → 存储"""

        try:
            # 1. 读取文件内容
            content = await self._read_file(file_path)

            # 2. 文本分块
            chunks = self.text_splitter.split_text(content)

            # 3. 生成嵌入向量
            embeddings = await self.embedding_service.embed_batch(chunks)

            # 4. 准备向量数据
            vector_data = []
            for i, (chunk, embedding) in enumerate(zip(chunks, embeddings)):
                chunk_id = str(uuid.uuid4())

                # 保存到 PostgreSQL
                db_chunk = DocumentChunk(
                    id=chunk_id,
                    document_id=document_id,
                    user_id=user_id,
                    knowledge_base_id=knowledge_base_id,
                    chunk_index=i,
                    content=chunk,
                    vector_id=chunk_id,  # 向量数据库中的 ID
                )
                self.db.add(db_chunk)

                # 准备向量数据
                vector_data.append({
                    "id": chunk_id,
                    "vector": embedding,
                    "metadata": {
                        "user_id": user_id,
                        "knowledge_base_id": knowledge_base_id,
                        "document_id": document_id,
                        "content": chunk,
                    }
                })

            await self.db.commit()

            # 5. 批量上传到向量数据库
            await self.vector_store.upsert_chunks(vector_data)

            # 6. 更新文档状态
            document = await self.db.get(Document, document_id)
            document.status = "completed"
            document.chunk_count = len(chunks)
            await self.db.commit()

        except Exception as e:
            # 错误处理
            document = await self.db.get(Document, document_id)
            document.status = "failed"
            document.error_message = str(e)
            await self.db.commit()
            raise

    async def _read_file(self, file_path: str) -> str:
        """根据文件类型读取内容"""
        if file_path.endswith('.pdf'):
            # PDF 处理逻辑
            pass
        elif file_path.endswith('.txt') or file_path.endswith('.md'):
            async with aiofiles.open(file_path, 'r') as f:
                return await f.read()
        # ... 其他文件类型
```

#### 7.2 API 端点实现

创建 `Universal-Agent-Backend/app/api/v1/documents.py`：

```python
from fastapi import APIRouter, Depends, UploadFile, File
from sqlalchemy.ext.asyncio import AsyncSession

router = APIRouter()

@router.post("/upload")
async def upload_document(
    knowledge_base_id: str,
    file: UploadFile = File(...),
    db: AsyncSession = Depends(get_db),
    current_user: dict = Depends(get_current_user),
):
    """上传文档，立即返回，后台异步处理"""

    user_id = current_user["user_id"]

    # 1. 验证知识库所有权
    kb = await KnowledgeBaseCRUD.get_by_id_and_user(db, knowledge_base_id, user_id)
    if not kb:
        raise HTTPException(status_code=403, detail="无权访问该知识库")

    # 2. 验证文件
    if file.size > 50 * 1024 * 1024:  # 50MB
        raise HTTPException(status_code=400, detail="文件过大")

    if file.filename.split('.')[-1] not in ['pdf', 'txt', 'md', 'docx']:
        raise HTTPException(status_code=400, detail="不支持的文件类型")

    # 3. 保存文件到本地
    file_path = f"{UPLOAD_DIR}/{user_id}/{uuid.uuid4()}_{file.filename}"
    os.makedirs(os.path.dirname(file_path), exist_ok=True)

    with open(file_path, 'wb') as f:
        f.write(await file.read())

    # 4. 创建文档记录 (状态: processing)
    document = Document(
        knowledge_base_id=knowledge_base_id,
        user_id=user_id,
        title=file.filename,
        file_name=file.filename,
        file_type=file.filename.split('.')[-1],
        file_size=os.path.getsize(file_path),
        file_path=file_path,
        status="processing",
    )
    db.add(document)
    await db.commit()
    await db.refresh(document)

    # 5. 启动后台处理任务
    asyncio.create_task(
        document_processor.process_document_async(
            document_id=document.id,
            user_id=user_id,
            knowledge_base_id=knowledge_base_id,
            file_path=file_path,
        )
    )

    # 6. 立即返回
    return {
        "document_id": document.id,
        "status": "processing",
        "message": "文档已上传，正在后台处理中"
    }

@router.get("/{document_id}/status")
async def get_document_status(
    document_id: str,
    db: AsyncSession = Depends(get_db),
    current_user: dict = Depends(get_current_user),
):
    """获取文档处理状态"""

    user_id = current_user["user_id"]

    # 获取文档（带用户验证）
    result = await db.execute(
        select(Document).where(
            and_(
                Document.id == document_id,
                Document.user_id == user_id
            )
        )
    )
    document = result.scalar_one_or_none()

    if not document:
        raise HTTPException(status_code=404, detail="文档不存在")

    return {
        "document_id": document.id,
        "status": document.status,
        "chunk_count": document.chunk_count,
        "error_message": document.error_message,
    }
```

#### 7.3 多知识库管理 API

创建 `Universal-Agent-Backend/app/api/v1/knowledge_bases.py`：

```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession

router = APIRouter()

@router.post("/api/v1/knowledge-bases")
async def create_knowledge_base(
    name: str,
    description: Optional[str] = None,
    db: AsyncSession = Depends(get_db),
    current_user: dict = Depends(get_current_user),
):
    """创建新知识库"""
    user_id = current_user["user_id"]

    kb = await KnowledgeBaseCRUD.create_for_user(
        db,
        user_id=user_id,
        name=name,
        description=description,
    )

    return {
        "id": kb.id,
        "name": kb.name,
        "description": kb.description,
        "created_at": kb.created_at,
    }

@router.get("/api/v1/knowledge-bases")
async def list_knowledge_bases(
    db: AsyncSession = Depends(get_db),
    current_user: dict = Depends(get_current_user),
):
    """列出用户的所有知识库"""
    user_id = current_user["user_id"]

    kbs = await KnowledgeBaseCRUD.list_for_user(db, user_id)

    return [
        {
            "id": kb.id,
            "name": kb.name,
            "description": kb.description,
            "document_count": len(kb.documents),
            "created_at": kb.created_at,
        }
        for kb in kbs
    ]

@router.get("/api/v1/knowledge-bases/{kb_id}")
async def get_knowledge_base(
    kb_id: str,
    db: AsyncSession = Depends(get_db),
    current_user: dict = Depends(get_current_user),
):
    """获取知识库详情（带用户验证）"""
    user_id = current_user["user_id"]

    kb = await KnowledgeBaseCRUD.get_by_id_and_user(db, kb_id, user_id)
    if not kb:
        raise HTTPException(status_code=404, detail="知识库不存在或无权访问")

    return {
        "id": kb.id,
        "name": kb.name,
        "description": kb.description,
        "embedding_model": kb.embedding_model,
        "document_count": len(kb.documents),
        "created_at": kb.created_at,
        "updated_at": kb.updated_at,
    }

@router.delete("/api/v1/knowledge-bases/{kb_id}")
async def delete_knowledge_base(
    kb_id: str,
    db: AsyncSession = Depends(get_db),
    current_user: dict = Depends(get_current_user),
):
    """删除知识库（级联删除所有文档和向量）"""
    user_id = current_user["user_id"]

    kb = await KnowledgeBaseCRUD.get_by_id_and_user(db, kb_id, user_id)
    if not kb:
        raise HTTPException(status_code=404, detail="知识库不存在或无权访问")

    # 删除向量数据库中的所有向量
    for document in kb.documents:
        await vector_store.delete_by_document(document.id)

    # 删除数据库记录（级联删除）
    await db.delete(kb)
    await db.commit()

    return {"message": "知识库已删除"}
```

---

## 后续优化

- 文档集合/标签
- 混合搜索 (语义 + 关键词)
- 文档版本控制
- 用户间共享知识库
- 高级分块策略
