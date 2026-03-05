# RAG MCP Server

> 一个基于 Model Context Protocol (MCP) 的语义搜索服务，为多租户 RAG 系统提供向量检索能力

## 简介

RAG MCP Server 是一个专注于语义搜索的 MCP 服务器，为 Universal Agent Backend 提供高效的向量检索能力。它通过 MCP 协议将搜索能力暴露给 Agent，实现智能的文档知识库查询。

**架构说明：**
- **文档管理**（上传、列表、删除）由 Universal Agent Backend 的 REST API 提供
- **语义搜索**由 RAG-MCP-Server 的 MCP 工具提供
- 所有操作严格按 `user_id` 隔离，确保多租户数据安全

## 核心特性

- **语义搜索** - 基于向量的语义检索（Pinecone/Weaviate）
- **用户隔离** - 向量数据库层面的元数据过滤
- **MCP 协议** - 标准化工具接口
- **高性能** - 向量索引优化、批量查询支持

## 快速开始

### 1. 安装依赖

```bash
# 使用 uv（推荐）
uv sync

# 或使用 pip
pip install -r requirements.txt
```

### 2. 配置环境变量

```bash
cp .env.example .env
# 编辑 .env 文件，配置向量数据库和 API 密钥
```

### 3. 启动服务

```bash
uv run python -m src.main
```

服务将以 MCP stdio 模式启动，等待 Agent 连接调用。

## MCP 工具列表

| 工具名称 | 描述 |
|---------|------|
| `search_knowledge_base` | 在用户的知识库中进行语义搜索（带用户隔离） |

**工具参数：**
- `user_id` (必需) - 用户 ID，用于数据隔离
- `knowledge_base_id` (必需) - 知识库 ID
- `query` (必需) - 搜索查询
- `top_k` (可选) - 返回结果数量，默认 5

**注意：** 文档上传、列表、删除等操作请使用 Universal Agent Backend 的 REST API 端点。

## 使用示例

### 在 Universal Agent Backend 中集成

在 UAF 的 MCP 配置中添加：

```python
MCP_SERVERS = {
    "rag": {
        "command": "uv",
        "args": ["--directory", "/path/to/RAG-MCP-Server", "run", "python", "-m", "src.main"],
        "env": {
            "PINECONE_API_KEY": "your-pinecone-key",
            "ZHIPUAI_API_KEY": "your-zhipuai-key"
        }
    }
}
```

### 端到端工作流程

```bash
# 1. 通过 REST API 创建知识库
POST /api/v1/knowledge-bases
{
  "name": "工作文档",
  "description": "我的工作资料"
}

# 2. 通过 REST API 上传文档
POST /api/v1/documents/upload
FormData: {knowledge_base_id, file}

# 3. Agent 调用 MCP 工具进行搜索
# (自动注入 user_id 和 knowledge_base_id)
tool_call: search_knowledge_base(
    user_id="user123",
    knowledge_base_id="kb456",
    query="人工智能应用",
    top_k=5
)
```

### 文档管理 REST API

文档管理的 REST API 端点由 Universal Agent Backend 提供：

**知识库管理：**
- `POST /api/v1/knowledge-bases` - 创建知识库
- `GET /api/v1/knowledge-bases` - 列出用户的知识库
- `GET /api/v1/knowledge-bases/{id}` - 获取知识库详情
- `DELETE /api/v1/knowledge-bases/{id}` - 删除知识库

**文档管理：**
- `POST /api/v1/documents/upload` - 上传文档
- `GET /api/v1/documents` - 列出文档
- `GET /api/v1/documents/{id}/status` - 查询处理状态
- `DELETE /api/v1/documents/{id}` - 删除文档

## 技术栈

- **MCP 框架**: FastMCP
- **向量数据库**: Pinecone 或 Weaviate
- **Embedding**: 智谱AI embedding-3 (1024 维)
- **异步框架**: asyncio

## 项目结构

```
RAG-MCP-Server/
├── src/
│   ├── main.py              # MCP 服务器入口
│   ├── tools/
│   │   └── rag_tools.py     # 搜索工具实现
│   ├── services/
│   │   ├── vector_store.py  # 向量数据库客户端
│   │   └── search.py        # 语义搜索服务
│   ├── config.py            # 配置管理
│   └── embedding.py         # Embedding 服务
├── tests/                   # 测试
├── pyproject.toml
├── .env.example
└── README.md
```

## 环境变量

```bash
# 向量数据库 (Pinecone)
PINECONE_API_KEY="your-pinecone-api-key"
PINECONE_ENVIRONMENT="us-west1-gcp"
PINECONE_INDEX_NAME="rag-knowledge-bases"

# 向量数据库 (Weaviate - 可选替代方案)
# WEAVIATE_URL="https://your-cluster.weaviate.cloud"
# WEAVIATE_API_KEY="your-weaviate-key"

# 智谱AI Embedding
ZHIPUAI_API_KEY="your_zhipuai_api_key"

# 向量维度
EMBEDDING_DIMENSION=1024
```

## 开发

```bash
# 运行测试
uv run pytest

# 代码格式化
uv run black .
uv run ruff check .

# 类型检查
uv run mypy .
```

## 部署

RAG-MCP-Server 可以通过 MCP stdio 协议与 Universal Agent Backend 通信：

```bash
# 在 Universal Agent Backend 中配置 MCP 服务器
# 服务会自动启动并保持连接
```

**部署建议：**
- 向量数据库：使用 Pinecone 云服务或自建 Weaviate 集群
- Embedding 服务：确保智谱AI API 可用
- 网络配置：确保 Universal Agent Backend 可以访问 RAG-MCP-Server 的 stdio 进程

## 相关项目

- [Universal Agent Backend](https://github.com/ReikiC/Universal-Agent-Backend) - Agent 编排框架
- [nuomi-frontend](https://github.com/ReikiC/nuomi-frontend) - 前端界面
- [UAF-Data-Base](https://github.com/ReikiC/UAF-Data-Base) - 数据库配置

## 许可证

Apache License 2.0
