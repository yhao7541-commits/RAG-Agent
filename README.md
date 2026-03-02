# Modular RAG MCP Server

> 一个可插拔、可观测的模块化 RAG (检索增强生成) 服务框架，通过 MCP (Model Context Protocol) 协议对外暴露工具接口，支持 Copilot / Claude 等 AI 助手直接调用。

---

## 🏗️ 项目概览

- **Ingestion Pipeline**：PDF → Markdown → Chunk → Transform → Embedding → Upsert（支持多模态图片描述）
- **Hybrid Search**：Dense (向量) + Sparse (BM25) + RRF Fusion + 可选 Rerank
- **MCP Server**：通过标准 MCP 协议暴露 `query_knowledge_hub`、`list_collections`、`get_document_summary` 三个 Tools
- **Dashboard**：Streamlit 六页面管理平台（系统总览 / 数据浏览 / Ingestion 管理 / 追踪可视化 / 评估面板）
- **Evaluation**：Ragas + Custom 评估体系，支持 golden test set 回归测试

> 📖 详细架构设计和任务排期请参阅 [DEV_SPEC.md](DEV_SPEC.md)

---

## 📂 分支说明

| 分支 | 用途 | 说明 |
|------|------|------|
| **`main`** | 最新代码 | 始终只有 **1 个 commit**，包含项目的最新完整代码。 |
| **`dev`** | 开发过程记录 | 保留了完整的 commit 历史，记录了从零开始逐步构建的过程。 |
| **`clean-start`** | 干净起点 | 仅包含工程骨架（Skills + DEV_SPEC），任务进度全部清零。**适合想从零开始自己动手实现的用户**。 |

---

## 🚀 快速开始

### 1. 环境准备

```bash
# 克隆项目
git clone <repo-url>
cd Modular-RAG-MCP-Server

# 创建虚拟环境 (Python 3.10+)
python -m venv .venv

# 激活虚拟环境
# Windows:
.\.venv\Scripts\Activate.ps1
# macOS/Linux:
source .venv/bin/activate

# 安装依赖
pip install -e ".[dev]"
```

### 2. 配置 API Key

编辑 `config/settings.yaml`，填入你的 LLM 和 Embedding 服务配置：

```yaml
llm:
  provider: "azure"            # 可选: openai, azure, ollama, deepseek
  model: "gpt-4o"
  api_key: "your-api-key"      # 替换为你的 API Key
  azure_endpoint: "https://your-endpoint.openai.azure.com/"

embedding:
  provider: "azure"            # 可选: openai, azure, ollama
  model: "text-embedding-ada-002"
  api_key: "your-api-key"      # 替换为你的 API Key
  azure_endpoint: "https://your-endpoint.openai.azure.com/"
```

> **提示**：如果使用 Ollama（本地部署），无需 API Key，只需确保 Ollama 服务在运行。

### 3. 运行首次数据摄取

```bash
# 摄取示例文档
python scripts/ingest.py --path tests/fixtures/sample_documents/ --collection default

# 摄取单个 PDF 文件
python scripts/ingest.py --path /path/to/your/document.pdf --collection my_collection
```

### 4. 执行查询

```bash
# 基础查询
python scripts/query.py --query "你的查询问题"

# 带详细输出的查询
python scripts/query.py --query "Azure OpenAI 如何配置？" --verbose

# 指定 collection 查询
python scripts/query.py --query "测试查询" --collection my_collection
```

---

## ⚙️ 配置说明

所有配置集中在 `config/settings.yaml`，各字段含义如下：

| 配置块 | 字段 | 说明 | 默认值 |
|--------|------|------|--------|
| **llm** | `provider` | LLM 提供商 | `azure` |
| | `model` | 模型名称 | `gpt-4o` |
| | `temperature` | 创造性程度 (0-1) | `0.0` |
| | `max_tokens` | 最大输出 token 数 | `4096` |
| **embedding** | `provider` | Embedding 提供商 | `azure` |
| | `model` | 模型名称 | `text-embedding-ada-002` |
| | `dimensions` | 向量维度 | `1536` |
| **vector_store** | `provider` | 向量存储引擎 | `chroma` |
| | `persist_directory` | 持久化路径 | `./data/db/chroma` |
| | `collection_name` | 默认集合名 | `knowledge_hub` |
| **retrieval** | `dense_top_k` | 稠密检索返回数 | `20` |
| | `sparse_top_k` | 稀疏检索返回数 | `20` |
| | `fusion_top_k` | 融合后保留数 | `10` |
| | `rrf_k` | RRF 常数 | `60` |
| **rerank** | `enabled` | 是否启用重排 | `false` |
| | `provider` | 重排器类型 | `none` |
| **ingestion** | `chunk_size` | 分块大小 (字符) | `1000` |
| | `chunk_overlap` | 块间重叠 | `200` |
| | `splitter` | 分割策略 | `recursive` |
| **observability** | `log_level` | 日志级别 | `INFO` |
| | `trace_enabled` | 是否启用追踪 | `true` |
| | `trace_file` | 追踪日志路径 | `./logs/traces.jsonl` |

---

## 🔌 MCP 配置

### GitHub Copilot（VS Code）

在项目根目录创建 `.vscode/mcp.json`：

```json
{
  "servers": {
    "modular-rag": {
      "type": "stdio",
      "command": "python",
      "args": ["-m", "src.mcp_server.server"],
      "cwd": "${workspaceFolder}"
    }
  }
}
```

### Claude Desktop

编辑 `claude_desktop_config.json`（路径因系统而异）：

```json
{
  "mcpServers": {
    "modular-rag": {
      "command": "python",
      "args": ["-m", "src.mcp_server.server"],
      "cwd": "/path/to/Modular-RAG-MCP-Server"
    }
  }
}
```

> 配置文件位置：
> - **Windows**: `%APPDATA%\Claude\claude_desktop_config.json`
> - **macOS**: `~/Library/Application Support/Claude/claude_desktop_config.json`

### 可用工具 (Tools)

| Tool 名称 | 功能 | 参数 |
|-----------|------|------|
| `query_knowledge_hub` | 混合检索知识库 | `query` (必填), `top_k`, `collection` |
| `list_collections` | 列出所有集合 | `include_stats` |
| `get_document_summary` | 获取文档摘要 | `doc_id` (必填), `collection` |

---

## 📊 Dashboard 使用指南

### 启动 Dashboard

```bash
# 默认端口 8501
python scripts/start_dashboard.py

# 指定端口
python scripts/start_dashboard.py --port 8502
```

访问 `http://localhost:8501` 即可打开管理平台。

### 页面功能

| 页面 | 功能 | 说明 |
|------|------|------|
| **📊 Overview** | 系统总览 | 展示组件配置、集合统计 |
| **🔍 Data Browser** | 数据浏览 | 浏览文档列表、chunk 内容、元数据 |
| **📥 Ingestion Manager** | 摄取管理 | 上传文件、触发 Pipeline、实时进度条 |
| **🔬 Ingestion Traces** | 摄取追踪 | 查看摄取链路各阶段耗时、详细日志 |
| **🔎 Query Traces** | 查询追踪 | 查看检索链路各阶段、Dense/Sparse 对比 |
| **📏 Evaluation Panel** | 评估面板 | 运行评估、查看 hit_rate/MRR 等指标 |

---

## 🧪 运行测试

```bash
# 运行全部测试
pytest -q

# 仅运行单元测试（快速，无外部依赖）
pytest tests/unit/ -q

# 仅运行集成测试（可能需要外部服务）
pytest tests/integration/ -q -m integration

# 仅运行 E2E 测试
pytest tests/e2e/ -q -m e2e

# 跳过需要真实 LLM API 的测试
pytest -m "not llm" -q

# 带覆盖率报告
pytest --cov=src --cov-report=term-missing -q
```

### 测试分层

| 层级 | 目录 | 覆盖范围 | 运行速度 |
|------|------|---------|---------|
| 单元测试 | `tests/unit/` | 独立模块逻辑，Mock 外部依赖 | 快 (~10s) |
| 集成测试 | `tests/integration/` | 模块间交互，可选真实后端 | 中等 (~30s) |
| E2E 测试 | `tests/e2e/` | 完整链路（MCP Client / Dashboard） | 慢 (~30s) |

---

## 🔧 常见问题

### API Key 配置

**Q: 报错 `AuthenticationError` 或 `401`**

检查 `config/settings.yaml` 中 API Key 是否正确：
- Azure: 确认 `azure_endpoint`、`api_key`、`deployment_name` 三者匹配
- OpenAI: 确认 `api_key` 以 `sk-` 开头
- Ollama: 确认本地服务已启动 (`ollama serve`)

### 依赖安装

**Q: 安装 `chromadb` 失败**

```bash
# Windows 需要 Visual C++ Build Tools
pip install chromadb --no-binary :all:

# 或者使用预编译版本
pip install chromadb
```

**Q: 安装 `PyMuPDF` 失败**

```bash
pip install PyMuPDF
# 如果报 wheel 错误，尝试升级 pip
pip install --upgrade pip setuptools wheel
```

### 连接问题

**Q: MCP Server 无响应**

1. 确认虚拟环境已激活
2. 尝试直接运行：`python -m src.mcp_server.server`
3. 检查 stderr 输出（MCP 使用 stdout 传输 JSON-RPC，日志在 stderr）

**Q: Dashboard 无法启动**

```bash
# 确认 Streamlit 已安装
pip install streamlit

# 检查端口占用
python scripts/start_dashboard.py --port 8502
```

**Q: 查询返回空结果**

1. 确认已执行数据摄取：`python scripts/ingest.py --path <file>`
2. 检查 collection 名称是否匹配
3. 查看 `logs/traces.jsonl` 中的错误信息

---

## 📁 项目结构

```
├── config/
│   ├── settings.yaml          # 主配置文件
│   └── prompts/               # LLM prompt 模板
├── src/
│   ├── core/                  # 核心：类型、设置、查询引擎、响应构建
│   ├── ingestion/             # 摄取：Pipeline、Chunking、Transform、Storage
│   ├── libs/                  # 可插拔层：LLM/Embedding/Splitter/VectorStore/Reranker
│   ├── mcp_server/            # MCP Server：Protocol Handler + Tools
│   └── observability/         # 可观测性：Logger、Dashboard、Evaluation
├── scripts/                   # CLI 入口脚本
├── tests/                     # 测试：unit / integration / e2e / fixtures
├── data/                      # 数据存储（ChromaDB / BM25 / 图片）
└── logs/                      # 追踪日志
```

---

## 📄 License

MIT
