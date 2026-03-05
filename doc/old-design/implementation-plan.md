# RAG 知识库功能 - Universal Agent Backend 项目结构实施

## Context

需要为 Universal Agent Backend 添加多租户 RAG 知识库功能。设计文档已确定使用方案2（轻量级改进）的文件组织结构，符合项目现有架构模式。

然而，这是把 RAG 嵌合进入 UAF-AO 的做法。这样子做并不优雅，UAF-AO 及其数据库 应该只负责 AI工作流、用户、对话记录

## 架构设计

### 完全遵循现有架构模式

**现有项目架构**：
```
app/core/
├── auth/           # 认证业务逻辑
├── sessions/       # 会话业务逻辑
├── tasks/          # 任务业务逻辑
└── crud/           # 统一的数据访问层
    ├── user.py
    ├── session.py
    └── message.py
```

**知识库架构（完全一致）**：
```
app/core/
├── auth/           # 现有
├── sessions/       # 现有
├── tasks/          # 现有
├── knowledge/      # ✅ 新增：知识库业务逻辑（与上面平级）
│   ├── processor.py
│   ├── vector_store.py
│   └── rag.py
└── crud/           # 统一的数据访问层
    ├── user.py
    ├── session.py
    ├── message.py
    ├── knowledge_base.py    # ✅ 新增
    ├── document.py          # ✅ 新增
    └── document_chunk.py    # ✅ 新增
```

### 关键设计原则

1. ✅ **不引入新的层**：没有 `services/` 目录，完全遵循现有模式
2. ✅ **CRUD 保持平级**：不在 `crud/` 下创建子目录
3. ✅ **业务逻辑独立**：`knowledge/` 与 `auth/`、`sessions/`、`tasks/` 平级
4. ✅ **职责清晰**：领域模块处理业务逻辑，CRUD 层处理数据访问

### 职责划分

**1. CRUD 层（`app/core/crud/`）**
- 纯数据库操作（增删改查）
- 不包含业务逻辑
- 可复用的数据访问层

**2. 领域模块层（`app/core/knowledge/`、`app/core/auth/` 等）**
- 业务逻辑编排
- 外部服务集成（向量存储、文件处理等）
- 跨多个 CRUD 操作的复杂流程


## 目标结构（按照设计文档）

### API 层 - `app/api/v1/knowledge.py`（新建端点文件）

**RESTful 设计原则**：
- 使用嵌套资源结构：Documents 是 Knowledge Bases 的子资源
- 统一的 URL 命名：使用 kebab-case（`knowledge-bases` 而非 `knowledge/bases`）
- 简化状态管理：`is_active` 通过 PATCH 更新，而非专用端点
- 使用查询参数过滤：`?is_active=true` 替代专用端点

**API 端点**：

```python
# Knowledge Base 端点（顶层资源）
POST   /api/v1/knowledge-bases                      # 创建知识库
GET    /api/v1/knowledge-bases                      # 列出用户的知识库（支持分页、搜索、排序）
GET    /api/v1/knowledge-bases?is_active=true       # 列出所有激活的知识库（Agent 使用）
GET    /api/v1/knowledge-bases/{kb_id}              # 获取知识库详情
PATCH  /api/v1/knowledge-bases/{kb_id}              # 更新知识库（包括 is_active 字段）
DELETE /api/v1/knowledge-bases/{kb_id}              # 删除知识库（级联删除所有文档和向量）

# Document 端点（嵌套资源，属于特定知识库）
POST   /api/v1/knowledge-bases/{kb_id}/documents                    # 上传文档
GET    /api/v1/knowledge-bases/{kb_id}/documents                    # 列出知识库的文档（支持分页）
GET    /api/v1/knowledge-bases/{kb_id}/documents/{doc_id}           # 获取文档详情
DELETE /api/v1/knowledge-bases/{kb_id}/documents/{doc_id}           # 删除文档（同步删除文件和向量）
GET    /api/v1/knowledge-bases/{kb_id}/documents/{doc_id}/status    # 查询处理状态

# 批量操作
DELETE /api/v1/knowledge-bases/{kb_id}/documents                    # 批量删除文档
PATCH  /api/v1/knowledge-bases/batch-active                         # 批量更新激活状态
```

**设计优势**：
1. ✅ **清晰的层级关系**：Documents 明确属于某个 Knowledge Base
2. ✅ **符合 REST 规范**：使用嵌套资源而非平行的端点
3. ✅ **简化的状态管理**：`is_active` 作为普通字段处理
4. ✅ **语义化 URL**：`knowledge-bases` 比 `knowledge/bases` 更清晰
5. ✅ **遵循项目模式**：单文件端点（如 `auth.py`, `sessions.py`）

### CRUD 层 - 新增知识库相关 CRUD（遵循现有模式）

**设计原则**：
- 遵循项目现有模式，与 `user.py`、`session.py`、`message.py` 平级
- 不创建子目录，直接在 `crud/` 下添加文件

```bash
app/core/crud/
├── __init__.py
├── user.py                  # 现有
├── session.py               # 现有
├── message.py               # 现有
├── knowledge_base.py        # ✅ 新增：KnowledgeBaseCRUD 类
├── document.py              # ✅ 新增：DocumentCRUD 类
└── document_chunk.py        # ✅ 新增：DocumentChunkCRUD 类
```

**CRUD 类**：
- `KnowledgeBaseCRUD` - 知识库 CRUD 操作
- `DocumentCRUD` - 文档 CRUD 操作
- `DocumentChunkCRUD` - 文档分块 CRUD 操作
- 所有方法遵循 `get_by_id_and_user(id, user_id)` 模式确保用户隔离

### 知识库业务逻辑层 - `app/core/knowledge/`（新建模块）

**设计原则**：
- 完全遵循现有架构模式，与 `auth/`、`sessions/`、`tasks/` 平级
- 包含知识库相关的业务逻辑和服务

```bash
app/core/
├── auth/                    # 现有：认证相关服务
├── sessions/                # 现有：会话相关服务
├── tasks/                   # 现有：任务相关服务
└── knowledge/               # ✅ 新增：知识库相关服务
    ├── __init__.py
    ├── processor.py          # 文档处理服务（分块、嵌入）
    ├── vector_store.py      # 本地向量存储（FAISS）
    └── rag.py               # RAG 搜索逻辑（可选，用于复杂查询）
```

**服务职责**：
- **processor.py**：文档处理（文本提取、分块、向量嵌入）
- **vector_store.py**：向量存储封装（upsert、delete、search）
- **rag.py**（可选）：RAG 查询逻辑（重排序、上下文组装等）

### 需要修改的现有文件
1. `app/models/database.py` - 添加 KnowledgeBase、Document、DocumentChunk ORM 模型
2. `app/graphs/agent.py` - 修改 `call_tool_node` 注入 user_id 到 MCP 工具调用
3. `app/api/v1/router.py` - 注册 knowledge 路由

## 新增文件清单（共 8 个新建，4 个修改）

### API 层（1 个文件）
1. `app/api/v1/knowledge.py` - 知识库和文档的 RESTful 端点实现

**设计原则**：
- 单文件包含所有相关端点，符合项目现有模式（如 `auth.py`, `sessions.py`）
- 使用 FastAPI 的 `APIRouter`，端点路径不带 `/api/v1` 前缀（在 router.py 中统一注册时添加 prefix）
- 支持 URL 路径参数的嵌套路由（`{kb_id}/documents`）

### CRUD 层（3 个文件）
2. `app/core/crud/knowledge_base.py` - KnowledgeBaseCRUD 类
3. `app/core/crud/document.py` - DocumentCRUD 类
4. `app/core/crud/document_chunk.py` - DocumentChunkCRUD 类

### 知识库业务逻辑层（3 个文件）
5. `app/core/knowledge/__init__.py` - 导出知识库服务
6. `app/core/knowledge/processor.py` - 文档处理服务（分块、嵌入）
7. `app/core/knowledge/vector_store.py` - 本地向量存储（FAISS）

### 需要修改的现有文件（4 个）
9. `app/models/database.py` - 添加 ORM 模型
10. `app/models/schemas.py` - 添加 Pydantic 请求/响应模型
11. `app/api/v1/router.py` - 注册 knowledge 路由
12. `app/graphs/agent.py` - 注入 user_id 到 MCP 工具调用

**设计原则**：
- 遵循项目现有模式，所有 Pydantic 模型集中在 `app/models/schemas.py`
- 不创建单独的 `schemas/` 目录

## 实施步骤

### Step 1: 创建目录结构
```bash
# 创建知识库业务逻辑目录（与 auth、sessions、tasks 平级）
mkdir -p app/core/knowledge

# 创建向量存储数据目录
mkdir -p data/vectors
```

**注意**：不需要创建 `crud/` 子目录，知识库的 CRUD 文件直接放在 `app/core/crud/` 下，与 `user.py`、`session.py` 平级。

### Step 2: 添加数据库模型
修改 `app/models/database.py`，添加三个 ORM 模型：
- `KnowledgeBase` - 知识库表（包含 `is_active` 字段用于用户勾选）
- `Document` - 文档表
- `DocumentChunk` - 文档分块表（元数据，向量存储在外部）

所有模型包含 `user_id` 字段用于多租户隔离。

**多知识库 Active 模式**：
- 用户可以创建多个知识库（工作、学习、项目等）
- 用户通过前端知识库管理页面勾选哪几个知识库是 active 的
- KnowledgeBase 模型包含 `is_active: bool` 字段（默认 True）
- Agent 搜索时在用户所有 `is_active=True` 的知识库中搜索
- 如果用户没有 active 知识库，提示用户先激活知识库

### Step 3: 实现 Pydantic 模型
修改 `app/models/schemas.py`，添加：

**分页响应模型**：
```python
class PaginatedResponse(BaseModel, Generic[T]):
    """通用分页响应模型"""
    items: list[T]
    total: int
    page: int
    page_size: int
    total_pages: int
```

**知识库相关模型**：
- `KnowledgeBaseCreateRequest` - 知识库创建请求
- `KnowledgeBaseResponse` - 知识库响应（包含 `is_active` 字段、文档数量）
- `KnowledgeBaseUpdateRequest` - 知识库更新请求（可选字段，包括 `is_active`）

**文档相关模型**：
- `DocumentResponse` - 文档响应（包含文件名、大小、状态、创建时间）
- `DocumentStatusResponse` - 文档状态响应（状态、分块数量、错误信息）

**批量操作模型**：
- `BatchDeleteRequest` - 批量删除请求（`doc_ids: list[str]`）
- `BatchUpdateActiveRequest` - 批量更新激活状态请求（`kb_ids: list[str]`, `is_active: bool`）
- `BatchOperationResponse` - 批量操作响应（`success: list[str]`, `failed: list[dict]`）

**设计要点**：
- 遵循项目现有模式，所有 Pydantic 模型集中在 `app/models/schemas.py`
- 使用 `pydantic.BaseModel` 作为基类
- 参考 `LoginRequest`, `TokenResponse`, `UserResponse` 等现有模型的风格
- 所有响应模型设置 `orm_mode = True`（或 Pydantic v2 的 `model_config = {"from_attributes": True}`）

### Step 4: 实现 CRUD 层
创建 CRUD 类，参考 `app/core/crud/user.py` 的模式：

**`app/core/crud/knowledge_base.py`** - KnowledgeBaseCRUD 类：
- `create_for_user(db, user_id, name, description)` - 创建知识库
- `get_by_id_and_user(db, kb_id, user_id)` - 获取知识库（带用户隔离）
- `list_by_user(db, user_id, search=None, is_active=None, sort_by="created_at", sort_order="desc", offset=0, limit=20)` - 列出用户的知识库（支持分页、搜索、排序）
- `count_by_user(db, user_id, search=None, is_active=None)` - 统计用户的知识库数量（用于分页）
- `update_by_id_and_user(db, kb_id, user_id, **kwargs)` - 更新知识库（包括 `is_active` 字段）
- `delete_by_id_and_user(db, kb_id, user_id)` - 删除知识库（级联删除所有文档）

**设计要点**：
- 使用标准的 `update` 方法处理 `is_active` 字段，而非专用的 `toggle_active` 方法
- `list_by_user` 支持可选的 `is_active` 过滤参数，避免重复方法

**`app/core/crud/document.py`** - DocumentCRUD 类：
- `create_for_user(db, kb_id, user_id, filename, file_path)` - 创建文档记录
- `get_by_id_and_user(db, doc_id, user_id)` - 获取文档（带用户隔离）
- `list_by_kb_and_user(db, kb_id, user_id, status=None, sort_by="created_at", sort_order="desc", offset=0, limit=20)` - 列出知识库的文档（支持分页、状态过滤）
- `count_by_kb_and_user(db, kb_id, user_id, status=None)` - 统计知识库的文档数量（用于分页）
- `update_status(db, doc_id, status, error_message=None)` - 更新处理状态
- `update_chunk_count(db, doc_id, chunk_count)` - 更新分块数量
- `delete_by_id_and_user(db, doc_id, user_id)` - 删除文档（级联删除所有分块）

**`app/core/crud/document_chunk.py`** - DocumentChunkCRUD 类：
- `create_for_document(db, doc_id, chunk_index, content, metadata)` - 创建分块记录
- `get_by_document(db, doc_id)` - 获取文档的所有分块
- `delete_by_document(db, doc_id)` - 删除文档的所有分块

### Step 5: 实现文档处理和向量存储服务

**`app/core/knowledge/processor.py`** - 文档处理服务：
- 文本提取：支持 PDF、TXT、MD、DOCX 格式
- 文本分块：使用 LangChain `RecursiveCharacterTextSplitter`
- 向量嵌入：集成智谱 AI embedding API（或本地模型）
- 异步处理流程，更新文档状态

**`app/core/knowledge/vector_store.py`** - 本地向量存储（FAISS）：

```python
import faiss
import numpy as np
import pickle
from pathlib import Path

class LocalVectorStore:
    """基于 FAISS 的本地向量存储"""

    def __init__(self, dimension: int = 1024):
        self.dimension = dimension
        self.index_path = Path("data/vectors/faiss.index")
        self.metadata_path = Path("data/vectors/metadata.pkl")

        # 加载或初始化索引
        if self.index_path.exists():
            self.index = faiss.read_index(str(self.index_path))
            with open(self.metadata_path, "rb") as f:
                self.metadata = pickle.load(f)
        else:
            self.index = faiss.IndexFlatIP(dimension)  # 内积相似度
            self.metadata = {}  # {doc_id: metadata}

    def upsert_chunks(self, doc_id: str, chunks: list[dict]):
        """插入或更新文档的分块向量"""
        vectors = np.array([c["embedding"] for c in chunks]).astype("float32")

        # 添加向量到索引
        start_idx = self.index.ntotal
        self.index.add(vectors)

        # 保存元数据
        self.metadata[doc_id] = {
            "start_idx": start_idx,
            "count": len(chunks),
            "texts": [c["text"] for c in chunks]
        }

        self._save()

    def delete_by_document(self, doc_id: str):
        """删除文档的所有向量"""
        if doc_id not in self.metadata:
            return

        # FAISS 不支持删除，需要重建索引
        self._rebuild_index(exclude_doc=doc_id)
        self._save()

    def search(self, query_vector: list[float], top_k: int = 5) -> list[dict]:
        """向量搜索"""
        query_vector = np.array([query_vector]).astype("float32")
        scores, indices = self.index.search(query_vector, top_k)

        results = []
        for score, idx in zip(scores[0], indices[0]):
            if idx == -1:  # FAISS 返回 -1 表示无效结果
                continue
            # 查找对应的文档和分块
            doc_id, chunk_idx = self._find_document_by_index(idx)
            results.append({
                "doc_id": doc_id,
                "chunk_id": f"{doc_id}_{chunk_idx}",
                "score": float(score),
                "text": self.metadata[doc_id]["texts"][chunk_idx]
            })

        return results

    def _save(self):
        """保存索引和元数据到磁盘"""
        faiss.write_index(self.index, str(self.index_path))
        with open(self.metadata_path, "wb") as f:
            pickle.dump(self.metadata, f)

    def _rebuild_index(self, exclude_doc: str = None):
        """重建索引（用于删除操作）"""
        # 收集所有保留的向量
        all_vectors = []
        all_metadata = {}

        for doc_id, meta in self.metadata.items():
            if doc_id == exclude_doc:
                continue

            # 重新提取向量（需要从索引中获取）
            start_idx = meta["start_idx"]
            count = meta["count"]
            vectors = self.index.reconstruct_n(start_idx, count)
            all_vectors.append(vectors)

            # 更新起始索引
            all_metadata[doc_id] = {
                "start_idx": len(all_vectors) - 1 if all_vectors else 0,
                "count": count,
                "texts": meta["texts"]
            }

        # 重建索引
        if all_vectors:
            self.index = faiss.IndexFlatIP(self.dimension)
            for vectors in all_vectors:
                self.index.add(vectors)
        else:
            self.index = faiss.IndexFlatIP(self.dimension)

        self.metadata = all_metadata

    def _find_document_by_index(self, idx: int) -> tuple[str, int]:
        """根据索引位置查找对应的文档和分块"""
        for doc_id, meta in self.metadata.items():
            start_idx = meta["start_idx"]
            count = meta["count"]
            if start_idx <= idx < start_idx + count:
                chunk_idx = idx - start_idx
                return doc_id, chunk_idx
        return None, None
```

**选择 FAISS 的原因**：
- ✅ 性能优异，支持大规模数据（百万级向量）
- ✅ 成熟稳定，Facebook/Meta 开源
- ✅ 内存占用小，搜索速度快
- ✅ 支持多种索引类型和相似度算法

### Step 6: 实现 API 端点
创建 `app/api/v1/knowledge.py`，参考 `app/api/v1/auth.py` 和 `app/api/v1/sessions.py` 的实现风格。

**通用设计原则**：
- 所有列表端点支持分页（默认 20 条/页，最大 100 条/页）
- 使用统一的错误响应格式
- 创建操作返回 201 状态码
- 删除操作级联删除关联数据和文件

**Knowledge Base 端点**：
```python
from fastapi import APIRouter, Depends, HTTPException, Query, UploadFile, status
from sqlalchemy.ext.asyncio import AsyncSession
import asyncio
import os
from datetime import datetime
from typing import Literal
from app.core.knowledge.vector_store import LocalVectorStore

router = APIRouter()
vector_store = LocalVectorStore()  # 本地向量存储实例

@router.post(
    "/knowledge-bases",
    response_model=KnowledgeBaseResponse,
    status_code=status.HTTP_201_CREATED,
    summary="创建知识库",
    description="为当前用户创建新的知识库"
)
async def create_knowledge_base(
    request: KnowledgeBaseCreateRequest,
    db: AsyncSession = Depends(get_db),
    current_user: dict = Depends(get_current_user),
) -> KnowledgeBaseResponse:
    """创建知识库"""
    user_id = current_user["user_id"]
    kb = await KnowledgeBaseCRUD.create_for_user(
        db, user_id, request.name, request.description
    )
    return kb

@router.get(
    "/knowledge-bases",
    response_model=dict,  # PaginatedResponse
    summary="列出知识库",
    description="支持分页、搜索、排序和过滤"
)
async def list_knowledge_bases(
    page: int = Query(1, ge=1, description="页码"),
    page_size: int = Query(20, ge=1, le=100, description="每页数量"),
    search: str | None = Query(None, description="搜索名称和描述"),
    sort_by: Literal["created_at", "updated_at", "name", "document_count"] = Query("created_at", description="排序字段"),
    sort_order: Literal["asc", "desc"] = Query("desc", description="排序方向"),
    is_active: bool | None = Query(None, description="过滤激活状态"),
    db: AsyncSession = Depends(get_db),
    current_user: dict = Depends(get_current_user),
) -> dict:
    """列出用户的知识库，支持分页、搜索、排序和过滤"""
    user_id = current_user["user_id"]

    # 获取总数和分页数据
    total = await KnowledgeBaseCRUD.count_by_user(
        db, user_id, search=search, is_active=is_active
    )
    kbs = await KnowledgeBaseCRUD.list_by_user(
        db, user_id,
        search=search,
        is_active=is_active,
        sort_by=sort_by,
        sort_order=sort_order,
        offset=(page - 1) * page_size,
        limit=page_size
    )

    return {
        "items": kbs,
        "total": total,
        "page": page,
        "page_size": page_size,
        "total_pages": (total + page_size - 1) // page_size
    }

@router.get("/knowledge-bases/{kb_id}", response_model=KnowledgeBaseResponse)
async def get_knowledge_base(
    kb_id: str,
    db: AsyncSession = Depends(get_db),
    current_user: dict = Depends(get_current_user),
):
    """获取知识库详情"""
    user_id = current_user["user_id"]
    kb = await KnowledgeBaseCRUD.get_by_id_and_user(db, kb_id, user_id)
    if not kb:
        raise HTTPException(status_code=404, detail="知识库不存在")
    return kb

@router.patch("/knowledge-bases/{kb_id}", response_model=KnowledgeBaseResponse)
async def update_knowledge_base(
    kb_id: str,
    request: KnowledgeBaseUpdateRequest,
    db: AsyncSession = Depends(get_db),
    current_user: dict = Depends(get_current_user),
):
    """更新知识库（包括 is_active 字段）"""
    user_id = current_user["user_id"]
    kb = await KnowledgeBaseCRUD.update_by_id_and_user(
        db, kb_id, user_id,
        name=request.name,
        description=request.description,
        is_active=request.is_active,
    )
    if not kb:
        raise HTTPException(404, "知识库不存在")
    return kb

@router.delete(
    "/knowledge-bases/{kb_id}",
    status_code=status.HTTP_204_NO_CONTENT,
    summary="删除知识库",
    description="删除知识库及其所有文档、文件和向量数据"
)
async def delete_knowledge_base(
    kb_id: str,
    db: AsyncSession = Depends(get_db),
    current_user: dict = Depends(get_current_user),
) -> None:
    """删除知识库（级联删除所有文档和向量数据）"""
    user_id = current_user["user_id"]

    # 获取知识库的所有文档
    docs = await DocumentCRUD.list_by_kb_and_user(db, kb_id, user_id)

    # 删除所有文档的文件和向量数据
    for doc in docs:
        # 删除文件
        if doc.file_path and os.path.exists(doc.file_path):
            try:
                os.remove(doc.file_path)
            except Exception as e:
                # 记录错误但继续删除
                print(f"删除文件失败: {e}")

        # 删除向量数据
        vector_store.delete_by_document(doc.id)

    # 删除数据库记录（级联删除文档和分块）
    success = await KnowledgeBaseCRUD.delete_by_id_and_user(db, kb_id, user_id)

    if not success:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail={"code": "KNOWLEDGE_BASE_NOT_FOUND", "message": "知识库不存在"}
        )
```

**Document 端点（嵌套路由）**：
```python
@router.post(
    "/knowledge-bases/{kb_id}/documents",
    response_model=DocumentResponse,
    status_code=status.HTTP_202_ACCEPTED,
    summary="上传文档",
    description="支持 PDF、TXT、MD、DOCX 格式，最大 50MB。文档将在后台异步处理。"
)
async def upload_document(
    kb_id: str,
    file: UploadFile,
    db: AsyncSession = Depends(get_db),
    current_user: dict = Depends(get_current_user),
) -> DocumentResponse:
    """上传文档到指定知识库（异步处理）"""
    user_id = current_user["user_id"]

    # 验证知识库存在且属于用户
    kb = await KnowledgeBaseCRUD.get_by_id_and_user(db, kb_id, user_id)
    if not kb:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail={"code": "KNOWLEDGE_BASE_NOT_FOUND", "message": "知识库不存在"}
        )

    # 验证文件大小和类型
    import mimetypes

    file_content = await file.read()
    if len(file_content) > 50 * 1024 * 1024:  # 50MB
        raise HTTPException(
            status_code=status.HTTP_413_REQUEST_ENTITY_TOO_LARGE,
            detail={"code": "FILE_TOO_LARGE", "message": "文件大小超过 50MB 限制"}
        )

    # 使用 MIME 类型验证文件类型
    file_ext = file.filename.split(".")[-1].lower() if "." in file.filename else ""
    if file_ext not in ["pdf", "txt", "md", "docx"]:
        raise HTTPException(
            status_code=status.HTTP_415_UNSUPPORTED_MEDIA_TYPE,
            detail={
                "code": "UNSUPPORTED_FILE_TYPE",
                "message": f"不支持的文件类型: .{file_ext}",
                "supported_types": ["pdf", "txt", "md", "docx"]
            }
        )

    # 验证 MIME 类型（防止文件扩展名伪造）
    mime_type, _ = mimetypes.guess_type(file.filename)
    allowed_mime_types = {
        "pdf": "application/pdf",
        "txt": "text/plain",
        "md": "text/markdown",
        "docx": "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
    }
    if mime_type and mime_type != allowed_mime_types.get(file_ext):
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail={"code": "INVALID_MIME_TYPE", "message": "文件内容与扩展名不匹配"}
        )

    # 生成文件路径并保存文件
    upload_dir = f"uploads/{user_id}/{kb_id}"
    os.makedirs(upload_dir, exist_ok=True)
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    file_path = f"{upload_dir}/{timestamp}_{file.filename}"

    with open(file_path, "wb") as f:
        f.write(file_content)

    # 创建文档记录并触发异步处理
    doc = await DocumentCRUD.create_for_user(db, kb_id, user_id, file.filename, file_path)

    # 异步处理文档（带错误处理）
    async def process_with_error_handling(doc_id: str):
        try:
            await process_document_async(doc_id)
        except Exception as e:
            # 记录错误并更新文档状态
            await DocumentCRUD.update_status(doc_id, "failed")
            # TODO: 添加日志记录
            print(f"文档处理失败: {e}")

    asyncio.create_task(process_with_error_handling(doc.id))

    return doc

@router.get(
    "/knowledge-bases/{kb_id}/documents",
    response_model=dict,  # PaginatedResponse
    summary="列出文档",
    description="列出知识库的所有文档，支持分页和排序"
)
async def list_documents(
    kb_id: str,
    page: int = Query(1, ge=1, description="页码"),
    page_size: int = Query(20, ge=1, le=100, description="每页数量"),
    sort_by: Literal["created_at", "filename", "file_size"] = Query("created_at", description="排序字段"),
    sort_order: Literal["asc", "desc"] = Query("desc", description="排序方向"),
    status: Literal["processing", "completed", "failed"] | None = Query(None, description="过滤处理状态"),
    db: AsyncSession = Depends(get_db),
    current_user: dict = Depends(get_current_user),
) -> dict:
    """列出知识库的所有文档（支持分页）"""
    user_id = current_user["user_id"]

    # 验证知识库存在
    kb = await KnowledgeBaseCRUD.get_by_id_and_user(db, kb_id, user_id)
    if not kb:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail={"code": "KNOWLEDGE_BASE_NOT_FOUND", "message": "知识库不存在"}
        )

    # 获取总数和分页数据
    total = await DocumentCRUD.count_by_kb_and_user(db, kb_id, user_id, status=status)
    docs = await DocumentCRUD.list_by_kb_and_user(
        db, kb_id, user_id,
        status=status,
        sort_by=sort_by,
        sort_order=sort_order,
        offset=(page - 1) * page_size,
        limit=page_size
    )

    return {
        "items": docs,
        "total": total,
        "page": page,
        "page_size": page_size,
        "total_pages": (total + page_size - 1) // page_size
    }

@router.get(
    "/knowledge-bases/{kb_id}/documents/{doc_id}",
    response_model=DocumentResponse,
    summary="获取文档详情",
    description="获取指定文档的详细信息"
)
async def get_document(
    kb_id: str,
    doc_id: str,
    db: AsyncSession = Depends(get_db),
    current_user: dict = Depends(get_current_user),
) -> DocumentResponse:
    """获取文档详情"""
    user_id = current_user["user_id"]

    doc = await DocumentCRUD.get_by_id_and_user(db, doc_id, user_id)
    if not doc or doc.knowledge_base_id != kb_id:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail={"code": "DOCUMENT_NOT_FOUND", "message": "文档不存在"}
        )

    return doc

@router.get(
    "/knowledge-bases/{kb_id}/documents/{doc_id}/status",
    response_model=DocumentStatusResponse,
    summary="查询文档处理状态",
    description="查询文档的异步处理状态（处理中/完成/失败）"
)
async def get_document_status(
    kb_id: str,
    doc_id: str,
    db: AsyncSession = Depends(get_db),
    current_user: dict = Depends(get_current_user),
) -> DocumentStatusResponse:
    """查询文档处理状态"""
    user_id = current_user["user_id"]

    doc = await DocumentCRUD.get_by_id_and_user(db, doc_id, user_id)
    if not doc or doc.knowledge_base_id != kb_id:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail={"code": "DOCUMENT_NOT_FOUND", "message": "文档不存在"}
        )

    return DocumentStatusResponse(
        doc_id=doc.id,
        status=doc.status,
        chunk_count=doc.chunk_count,
        error_message=doc.error_message
    )

@router.delete(
    "/knowledge-bases/{kb_id}/documents/{doc_id}",
    status_code=status.HTTP_204_NO_CONTENT,
    summary="删除文档",
    description="删除文档及其文件和向量数据"
)
async def delete_document(
    kb_id: str,
    doc_id: str,
    db: AsyncSession = Depends(get_db),
    current_user: dict = Depends(get_current_user),
) -> None:
    """删除文档（同步删除文件和向量数据）"""
    user_id = current_user["user_id"]

    # 获取文档
    doc = await DocumentCRUD.get_by_id_and_user(db, doc_id, user_id)
    if not doc or doc.knowledge_base_id != kb_id:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail={"code": "DOCUMENT_NOT_FOUND", "message": "文档不存在"}
        )

    # 删除文件
    if doc.file_path and os.path.exists(doc.file_path):
        try:
            os.remove(doc.file_path)
        except Exception as e:
            print(f"删除文件失败: {e}")

    # 删除向量数据
    vector_store.delete_by_document(doc_id)

    # 删除数据库记录
    await DocumentCRUD.delete_by_id_and_user(db, doc_id, user_id)

# 批量操作端点
class BatchDeleteRequest(BaseModel):
    doc_ids: list[str]

@router.delete(
    "/knowledge-bases/{kb_id}/documents",
    status_code=status.HTTP_207_MULTI_STATUS,
    summary="批量删除文档",
    description="批量删除多个文档及其文件和向量数据"
)
async def batch_delete_documents(
    kb_id: str,
    request: BatchDeleteRequest,
    db: AsyncSession = Depends(get_db),
    current_user: dict = Depends(get_current_user),
) -> dict:
    """批量删除文档"""
    user_id = current_user["user_id"]

    # 验证知识库存在
    kb = await KnowledgeBaseCRUD.get_by_id_and_user(db, kb_id, user_id)
    if not kb:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail={"code": "KNOWLEDGE_BASE_NOT_FOUND", "message": "知识库不存在"}
        )

    results = {"success": [], "failed": []}

    for doc_id in request.doc_ids:
        try:
            doc = await DocumentCRUD.get_by_id_and_user(db, doc_id, user_id)
            if not doc or doc.knowledge_base_id != kb_id:
                results["failed"].append({"doc_id": doc_id, "error": "文档不存在"})
                continue

            # 删除文件和向量
            if doc.file_path and os.path.exists(doc.file_path):
                os.remove(doc.file_path)
            vector_store.delete_by_document(doc_id)
            await DocumentCRUD.delete_by_id_and_user(db, doc_id, user_id)

            results["success"].append(doc_id)
        except Exception as e:
            results["failed"].append({"doc_id": doc_id, "error": str(e)})

    return results

class BatchUpdateActiveRequest(BaseModel):
    kb_ids: list[str]
    is_active: bool

@router.patch(
    "/knowledge-bases/batch-active",
    summary="批量更新激活状态",
    description="批量更新多个知识库的激活状态"
)
async def batch_update_active(
    request: BatchUpdateActiveRequest,
    db: AsyncSession = Depends(get_db),
    current_user: dict = Depends(get_current_user),
) -> dict:
    """批量更新知识库激活状态"""
    user_id = current_user["user_id"]

    results = {"success": [], "failed": []}

    for kb_id in request.kb_ids:
        try:
            kb = await KnowledgeBaseCRUD.update_by_id_and_user(
                db, kb_id, user_id, is_active=request.is_active
            )
            if kb:
                results["success"].append(kb_id)
            else:
                results["failed"].append({"kb_id": kb_id, "error": "知识库不存在"})
        except Exception as e:
            results["failed"].append({"kb_id": kb_id, "error": str(e)})

    return results
```

**设计要点**：
- 所有端点都需要 JWT 认证
- 从 token 中提取 `user_id` 实现用户隔离
- 使用 FastAPI 的路径参数实现嵌套资源
- 文件上传使用 `UploadFile` 类型
- 异步文档处理在后台执行
- **P0 改进**：
  - 所有列表端点支持分页（防止性能问题）
  - 删除操作级联删除文件和向量数据（防止磁盘泄漏）
  - 统一错误响应格式（结构化错误码和消息）
  - 正确的 HTTP 状态码（201 创建、202 异步处理、204 删除、413 文件过大、415 不支持的类型）
- **P1 改进**：
  - 支持搜索和排序（提升用户体验）
  - 批量操作端点（提升前端效率）
  - 详细的 OpenAPI 文档（自动生成清晰的 API 文档）

### Step 7: 集成到现有系统

**修改 `app/api/v1/router.py`** - 注册 knowledge 路由：
```python
from app.api.v1 import knowledge

router.include_router(knowledge.router, prefix="/api/v1")
```

**修改 `app/graphs/agent.py`** - 实现多知识库搜索：
```python
async def call_tool_node(state: AgentState, config: dict):
    """执行工具调用，支持多知识库 active 模式"""
    user_id = config.get("configurable", {}).get("user_id")

    # 获取用户所有激活的知识库
    active_kbs = await KnowledgeBaseCRUD.list_by_user(db, user_id, is_active=True)

    if not active_kbs:
        # 没有激活的知识库，返回提示
        return {
            "messages": [
                ToolMessage(
                    content="请先在知识库管理页面激活至少一个知识库",
                    tool_call_id=tool_call["id"],
                )
            ]
        }

    # 在所有激活的知识库中并行搜索
    search_tasks = [
        search_knowledge_base(
            user_id=user_id,
            knowledge_base_id=kb.id,
            query=query,
            top_k=3,  # 每个知识库取 3 个结果
        )
        for kb in active_kbs
    ]

    results = await asyncio.gather(*search_tasks)

    # 合并结果，按相关性排序
    all_results = []
    for kb_results in results:
        all_results.extend(kb_results)

    # 去重并排序
    all_results = deduplicate_and_sort(all_results)

    return {"messages": [tool_message]}
```

### 前端界面需求

**知识库管理页面**：
```
┌─────────────────────────────────────────┐
│  我的知识库                              │
├─────────────────────────────────────────┤
│  ☑ 工作文档 (12 篇)    [编辑] [删除]    │
│  ☑ 学习笔记 (8 篇)     [编辑] [删除]    │
│  ☐ 项目资料 (5 篇)     [编辑] [删除]    │
│  ☐ 个人日记 (3 篇)     [编辑] [删除]    │
│                                          │
│  [+ 创建新知识库]                         │
└─────────────────────────────────────────┘
```

- 用户可以勾选/取消勾选复选框
- 实时保存 `is_active` 状态
- 显示每个知识库的文档数量
