# RAG MCP Server

> 一个基于 Model Context Protocol (MCP) 的独立 RAG 服务，提供文档知识库和语义检索能力

## 简介

RAG MCP Server 是一个标准化的 MCP 服务器，将 RAG（检索增强生成）能力封装成可复用的工具。任何支持 MCP 协议的 Agent 应用都可以通过标准接口集成文档知识库功能。

## 核心特性

- **智能检索** - 语义向量搜索（pgvector + OpenAI embeddings）
- **文档管理** - 支持 PDF、TXT、Markdown、DOCX
- **MCP 协议** - 即插即用的标准化接口
- **用户隔离** - 多用户数据隔离和权限控制
- **高性能** - 批量处理、向量索引优化

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
# 编辑 .env 文件，配置数据库和 API 密钥
```

### 3. 初始化数据库

```bash
# 启动 PostgreSQL（如果还未启动）
docker-compose up -d

# 运行迁移
uv run python -m scripts.init_db
```

### 4. 启动服务

```bash
uv run python -m src.main
```

服务将在 `http://localhost:8000` 启动。

## MCP 工具列表

| 工具名称 | 描述 |
|---------|------|
| `upload_document` | 上传并处理文档（自动分块、向量化） |
| `search_knowledge_base` | 语义搜索知识库 |
| `list_documents` | 列出用户的所有文档 |
| `delete_document` | 删除文档及其向量数据 |
| `create_collection` | 创建文档集合 |
| `add_to_collection` | 将文档添加到集合 |

## 使用示例

### 在 Universal Agent Backend 中集成

在 UAF 的 MCP 配置中添加：

```python
MCP_SERVERS = {
    "rag": {
        "command": "uv",
        "args": ["--directory", "/path/to/RAG-MCP-Server", "run", "python", "-m", "src.main"],
        "env": {
            "DATABASE_URL": "postgresql+asyncpg://...",
            "OPENAI_API_KEY": "sk-..."
        }
    }
}
```

### Agent 调用示例

```python
# Agent 调用 RAG 工具
tool_call: search_knowledge_base(
    query="人工智能应用",
    top_k=5
)

# 返回带引用的答案
response: "根据你的文档[1][2]，人工智能的主要应用包括..."
```

## 技术栈

- **MCP 框架**: fastmcp
- **向量数据库**: PostgreSQL + pgvector
- **Embedding**: 智谱AI embedding-3
- **文档处理**: LangChain + Unstructured
- **异步框架**: asyncio + SQLAlchemy 2.0

## 项目结构

```
RAG-MCP-Server/
├── src/
│   ├── main.py              # MCP 服务器入口
│   ├── tools/               # MCP 工具实现
│   ├── services/            # 业务逻辑层
│   ├── models/              # 数据模型
│   └── core/                # 核心配置
├── migrations/              # 数据库迁移
├── tests/                   # 测试
├── pyproject.toml
├── .env.example
└── README.md
```

## 环境变量

```bash
# 数据库
DATABASE_URL="postgresql+asyncpg://postgres:postgres@localhost:5432/rag_mcp"

# 智谱AI
ZHIPUAI_API_KEY="your_zhipuai_api_key"

# 文档处理
CHUNK_SIZE=1000
CHUNK_OVERLAP=200
MAX_FILE_SIZE=52428800  # 50MB

# 对象存储
STORAGE_BACKEND="local"  # local | minio | s3
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

```bash
# Docker 部署
docker-compose up -d

# 查看日志
docker-compose logs -f rag-mcp
```

## 相关项目

- [Universal Agent Backend](https://github.com/ReikiC/Universal-Agent-Backend) - Agent 编排框架
- [nuomi-frontend](https://github.com/ReikiC/nuomi-frontend) - 前端界面
- [UAF-Data-Base](https://github.com/ReikiC/UAF-Data-Base) - 数据库配置

## 许可证

Apache License 2.0
