# rag-mcp-server

把一个 RAG 知识库通过 [Model Context Protocol](https://modelcontextprotocol.io) 提供给 Claude Desktop、MCP Inspector 等客户端：可以检索文档、按 ID 读取原文，并提供文档清单与预置提示。

本仓库是从 RAG 知识库项目中拆出的 MCP 层，**不能单独运行**：它直接使用主项目的 `app` 包（检索工具、向量索引、PostgreSQL 访问），主项目目前为私有仓库。这里公开代码，用于展示设计与实现；要运行，需要在主项目的目录结构中使用（见下文）。

## 提供的能力

| 类型 | 名称 | 说明 |
|---|---|---|
| Tool | `search_documents` | 语义检索，返回带来源标注的文本片段。`top_k` 默认 5，上限 20 |
| Tool | `get_document_by_id` | 按文档 ID 分页读取正文，大文档会提示如何取下一页 |
| Resource | `docs://list` | 知识库文档清单（文档 ID、文件名、分块数）。来自索引元数据，不查数据库 |
| Resource | `docs://stats` | 知识库统计：文档数、分块数、切分策略、可用工具 |
| Prompt | `rag_qa` | 先检索再作答，并标注来源 |
| Prompt | `summarize_document` | 通读指定文档并生成结构化摘要 |

## 目录结构

```
app/mcp_server/
  __main__.py    入口：python -m app.mcp_server（stdio 传输）
  server.py      MCPServer 定义、lifespan（启动时装配服务）、模型指令
  bootstrap.py   服务装配：向量索引、数据库，每一步独立降级
  tools.py       两个 Tool，复用主项目的 ToolRegistry 实现
  resources.py   docs://list、docs://stats
  prompts.py     两个 Prompt 模板
tests/test_mcp_server.py       内存 Client 的单元测试
scripts/smoke_mcp_stdio.py     真实子进程的端到端冒烟测试
```

## 运行

前提：在主项目中已经构建过向量索引（`python scripts/build_index.py`），并且 PostgreSQL 可用（不可用时只有检索工具可以使用）。

```bash
# 在主项目根目录下
python -m app.mcp_server
```

### 接入 Claude Desktop

编辑 `claude_desktop_config.json`，在 `mcpServers` 中加入（与已有配置合并，不要整个替换）。路径换成你自己的：

```json
{
  "mcpServers": {
    "rag-knowledge-base": {
      "command": "/path/to/RAG_project/RAG_venv/bin/python",
      "args": ["-m", "app.mcp_server"],
      "env": {
        "PYTHONPATH": "/path/to/RAG_project",
        "PYTHONIOENCODING": "utf-8"
      }
    }
  }
}
```

Windows 下虚拟环境的 Python 位于 `RAG_venv\Scripts\python.exe`。修改后需要完全退出并重新启动 Claude Desktop。

### 用 MCP Inspector 调试

```bash
npx -y @modelcontextprotocol/inspector "/path/to/RAG_project/RAG_venv/bin/python" -m app.mcp_server
```

## 测试

```bash
# 单元测试（在主项目中运行，依赖主项目的 app 包）
pytest tests/test_mcp_server.py

# 端到端冒烟测试：启动真实子进程，检查协议协商、三类原语、工具调用与资源读取
python scripts/smoke_mcp_stdio.py
```

## 设计要点

- **stdout 是协议通道。** 日志全部输出到 stderr；包级别不做任何 import 或打印。任何写入 stdout 的内容都会让客户端解析失败，而且表现为"服务器无响应"，很难排查。
- **工具错误用 `raise ToolError`，不是返回错误字符串。** 返回字符串时 `is_error` 为 false，模型会把错误信息当成答案。
- **服务在 lifespan 中装配，不在模块导入时装配。** 模块导入发生在 stdout 保护之外。
- **静态 Resource 不能注入 `Context`**（MCP SDK 的限制），所以 `docs://list` 通过模块级引用读取服务。
- **任一依赖失败只降级，不阻止启动。** 索引加载失败时检索不可用，数据库连接失败时文档读取不可用，`docs://list` 仍然可用。

## 已知限制

- **不做用户隔离。** 它是运行在本机的 stdio 进程，拥有运行者的数据库访问权限。多用户的数据隔离只在 Web 接口中实现。
- **不能脱离主项目单独运行。** 见上文。

## 许可证

MIT，见 [LICENSE](LICENSE)。
