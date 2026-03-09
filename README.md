# UAF-MCP-RAG-Server 

> 一个基于 Model Context Protocol (MCP) 的独立 RAG 服务，提供文档知识库和语义检索能力

## 简介

UAF 的配套 MCP 业务（RAG）后端。 需配合 UAF 的核心项目 使用，通过 MCP 为 UAF 提供 智能的知识库 查询。

## 核心特性

- **智能检索** - 语义向量搜索（pgvector + OpenAI embeddings）
- **文档管理** - 支持 PDF、TXT、Markdown、DOCX
- **MCP 协议** - 即插即用的标准化接口
- **用户隔离** - 多用户数据隔离和权限控制
- **高性能** - 批量处理、向量索引优化

## 相关项目

**UAF 生态系统** - 请配合使用：

### 核心组件

| 项目 | 说明 | 地址 |
|------|------|------|
| **UAF-Orchestrator** | Agent 编排框架（LangChain + LangGraph） | [https://github.com/ReikiC/UAF-Orchestrator](https://github.com/ReikiC/UAF-Orchestrator) |
| **UAF-Frontend-nuomi** | 前端界面（React + TypeScript + Vite） | [https://github.com/ReikiC/UAF-Frontend-nuomi](https://github.com/ReikiC/UAF-Frontend-nuomi) |
| **UAF-Data-Base** | PostgreSQL 数据库配置 | [https://github.com/ReikiC/UAF-Data-Base](https://github.com/ReikiC/UAF-Data-Base) |

### RAG 业务模块

| 项目 | 说明 | 地址 |
|------|------|------|
| **UAF-MCP-RAG-Frontend** | 知识库前端界面 | [https://github.com/ReikiC/UAF-MCP-RAG-Frontend](https://github.com/ReikiC/UAF-MCP-RAG-Frontend) |
| **UAF-MCP-RAG-Server** | RAG 后端服务（MCP 协议） | [https://github.com/ReikiC/UAF-MCP-RAG-Server](https://github.com/ReikiC/UAF-MCP-RAG-Server) |
| **UAF-MCP-RAG-DB** | RAG 向量数据库支持 | [https://github.com/ReikiC/UAF-MCP-RAG-DB](https://github.com/ReikiC/UAF-MCP-RAG-DB) |

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

## 许可证

Apache License 2.0
