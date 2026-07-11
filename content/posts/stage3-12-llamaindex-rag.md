---
title: "12 | 实战——用 LlamaIndex 构建知识库问答系统"
date: 2026-07-07
draft: false
weight: 304
tags: ["Agent", "RAG", "LlamaIndex", "MCP", "实战"]
summary: "用 LlamaIndex 从零构建一个知识库问答系统：索引技术文档、构建检索链、问答测试，最后接入 MCP Server 让 Agent 通过标准协议访问知识库。完整可运行代码。"
ShowToc: true
---
前面三篇我们学习了 Context Engineering、Memory 机制和 RAG 的理论。但理论终究要落地——这篇文章，我们**动手干**。

目标很明确：用 LlamaIndex 从零构建一个可以回答技术文档问题的知识库问答系统，最后把它封装成 MCP Server，让任何支持 MCP 协议的 Agent 都能直接调用。

![LlamaIndex RAG 实战路线](/images/stage3-12-llamaindex-rag.svg)

---

## 环境准备

### 安装依赖

```bash
pip install llama-index llama-index-embeddings-openai llama-index-llms-openai
pip install llama-index-readers-web  # 网页加载
pip install llama-index-vector-stores-chroma  # 本地向量数据库
pip install chromadb
pip install pypdf  # PDF 支持
```

### 配置 API Key

```python
import os
os.environ["OPENAI_API_KEY"] = "sk-..."  # 你的 OpenAI Key
```

如果你使用其他兼容 OpenAI API 的服务（比如 Azure、本地部署的 Ollama），后面会讲怎么切换。

### 准备测试文档

创建一个 `data/` 目录，放入一些测试文档：

```bash
mkdir data
```

`data/python_asyncio.md`：

```markdown
# Python asyncio 入门指南

asyncio 是 Python 的异步 I/O 库，用于编写并发代码。

## 核心概念

- **Coroutine**: 用 async def 定义的函数，可以被暂停和恢复
- **Event Loop**: 运行和管理协程的循环
- **Task**: 对 Coroutine 的封装，用于并发执行
- **await**: 暂停当前协程，等待另一个协程完成

## 基本用法

```python
import asyncio

async def fetch_data(url):
    print(f"开始请求 {url}")
    await asyncio.sleep(1)  # 模拟网络请求
    return f"{url} 的数据"

async def main():
    # 并发执行多个请求
    results = await asyncio.gather(
        fetch_data("https://api.example.com/1"),
        fetch_data("https://api.example.com/2"),
        fetch_data("https://api.example.com/3"),
    )
    print(results)

asyncio.run(main())
```

## 性能对比

同步方式请求 100 个 URL 约需 100 秒（每个 1 秒），
asyncio 并发方式仅需约 1 秒（所有请求同时进行）。

`data/fastapi_guide.md`：

```Markdown
# FastAPI 快速上手

FastAPI 是一个用于构建 API 的现代 Python Web 框架，基于类型提示自动生成文档。

## 安装

pip install fastapi uvicorn

## 最小示例

from fastapi import FastAPI
app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World"}

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    return {"user_id": user_id, "name": "Alice"}

## 特性

- 自动 API 文档（Swagger UI 和 ReDoc）
- 数据验证基于 Pydantic
- 原生支持 async/await
- 依赖注入系统
- 自动生成 OpenAPI schema

## 与 asyncio 的关系

FastAPI 原生支持异步路由。当你的路由函数用 async def 定义时，
FastAPI 会在事件循环中直接运行它，无需额外线程。
这使得 FastAPI 非常适合 I/O 密集型应用。
```

---

## Step 1: 文档加载

LlamaIndex 支持多种文档格式。我们把常用的几种都演示一遍。

### 加载 Markdown 和纯文本

最简单的方式——直接读目录：

```python
from llama_index.core import SimpleDirectoryReader

# 加载 data/ 目录下的所有文档
documents = SimpleDirectoryReader("data").load_data()

print(f"加载了 {len(documents)} 个文档")
for doc in documents:
    print(f"  - {doc.metadata.get('file_name', 'unknown')}: {len(doc.text)} 字符")
```

`SimpleDirectoryReader` 会自动识别 `.md`、`.txt`、`.pdf`、`.docx` 等格式。

### 加载 PDF

```python
# SimpleDirectoryReader 已经支持 PDF
# 也可以单独加载：
from llama_index.readers.file import PDFReader

pdf_reader = PDFReader()
pdf_docs = pdf_reader.load_data("path/to/your/document.pdf")
```

### 加载网页

```python
from llama_index.readers.web import SimpleWebPageReader

web_docs = SimpleWebPageReader().load_data([
    "https://docs.python.org/3/library/asyncio.html",
    "https://fastapi.tiangolo.com/",
])

print(f"从网页加载了 {len(web_docs)} 个文档")
```

### 手动创建 Document

有时你想从数据库、API 或其他来源导入内容：

```python
from llama_index.core import Document

custom_doc = Document(
    text="LlamaIndex 是一个用于构建 LLM 应用的框架，核心能力是连接外部数据与 LLM。",
    metadata={"source": "manual", "topic": "llamaindex"}
)
documents.append(custom_doc)
```

---

## Step 2: 文档切分与索引

原始文档通常很长，直接存入向量数据库效果不好。我们需要：

1. 把文档切成合适大小的 chunk
2. 把每个 chunk 转换为向量（Embedding）
3. 存入向量数据库

### 切分文档

```python
from llama_index.core.node_parser import SentenceSplitter

# 按句子切分，每个 chunk 约 512 tokens，相邻 chunk 重叠 50 tokens
splitter = SentenceSplitter(chunk_size=512, chunk_overlap=50)
nodes = splitter.get_nodes_from_documents(documents)

print(f"切分后得到 {len(nodes)} 个节点")
for i, node in enumerate(nodes[:3]):
    print(f"\n节点 {i+1} ({len(node.text)} 字符):")
    print(node.text[:100] + "...")
```

**chunk_size 怎么选？**

| 场景         | 建议 chunk_size | 原因                         |
| ------------ | --------------- | ---------------------------- |
| FAQ、定义类  | 256             | 答案通常很短，小块更精准     |
| 技术文档     | 512             | 平衡上下文和精准度           |
| 长篇文章分析 | 1024            | 需要更长上下文来理解段落关系 |

### 选择 Embedding 模型并存入向量数据库

```python
from llama_index.core import VectorStoreIndex, Settings
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.llms.openai import OpenAI
from llama_index.vector_stores.chroma import ChromaVectorStore
import chromadb

# 配置 LLM 和 Embedding
Settings.llm = OpenAI(model="gpt-4o-mini", temperature=0)
Settings.embed_model = OpenAIEmbedding(model="text-embedding-3-small")

# 使用 Chroma 作为本地向量数据库（持久化存储）
chroma_client = chromadb.PersistentClient(path="./chroma_db")
chroma_collection = chroma_client.get_or_create_collection("tech_docs")
vector_store = ChromaVectorStore(chroma_collection=chroma_collection)

# 构建索引
index = VectorStoreIndex(nodes, vector_store=vector_store)
print("索引构建完成！")
```

`Settings` 是 LlamaIndex 的全局配置对象，设置后所有后续操作都会使用这些模型。`text-embedding-3-small` 是性价比最高的 Embedding 模型，1536 维，速度快、成本低。

---

## Step 3: 构建查询引擎

这是最核心的一步——把索引变成可以问答的系统。

### 基本查询

```python
query_engine = index.as_query_engine()

response = query_engine.query("asyncio 的 Event Loop 是什么？")
print(response)
```

输出大致是：

```
Event Loop 是 asyncio 中用于运行和管理协程的循环。它负责调度协程的执行，
处理 I/O 事件，并在协程等待时切换到其他任务。通过 Event Loop，
多个协程可以在同一线程中并发执行。
```

### 查看检索到的上下文

想看看模型是基于哪些内容回答的？

```python
response = query_engine.query("FastAPI 为什么适合 I/O 密集型应用？")

# 查看模型看到了哪些文档片段
for i, node in enumerate(response.source_nodes):
    print(f"\n--- 参考文档 {i+1} (相关度: {node.score:.4f}) ---")
    print(node.text[:200])
```

这一步很重要——**RAG 的答案质量，直接取决于检索到的文档质量**。如果检索到的内容不相关，再强的模型也给不出好答案。

---

## Step 4: 高级检索配置

默认配置能用，但往往不是最优的。我们来调优。

### 调整 top_k

`top_k` 控制检索返回多少个文档片段给模型：

```python
# 默认 top_k=2，有时太少
query_engine_top5 = index.as_query_engine(similarity_top_k=5)

response = query_engine_top5.query("asyncio 和 FastAPI 有什么关系？")
print(f"参考了 {len(response.source_nodes)} 个文档片段")
```

跨文档的问题（比如 asyncio 和 FastAPI 的关系）通常需要更大的 `top_k`，因为相关信息分散在不同文档里。

### 添加 Reranker

向量检索是"语义近似"，有时会召回相关度不够高的结果。Reranker 用更精确的交叉编码器对初步检索结果做二次排序：

```python
from llama_index.core.postprocessor import SentenceTransformerRerank

# 用本地模型做 Rerank（不需要 API，速度快）
reranker = SentenceTransformerRerank(
    model="cross-encoder/ms-marco-MiniLM-L-6-v2",
    top_n=3  # Rerank 后只保留前 3 个
)

# 先检索 top-10，再用 Reranker 精选 top-3
query_engine_rerank = index.as_query_engine(
    similarity_top_k=10,
    node_postprocessors=[reranker]
)

response = query_engine_rerank.query("如何用 asyncio 并发请求多个 API？")
print(response)
print(f"\nRerank 后保留了 {len(response.source_nodes)} 个片段")
```

**Reranker 为什么有效？** 向量 Embedding 是"双编码器"——分别编码 query 和 document，再算余弦相似度。而 Cross-Encoder 会把 query 和 document 拼在一起输入模型，能捕捉更细粒度的语义关系，所以排序更准。代价是速度更慢，所以先用向量检索粗筛，再用 Reranker 精选。

### 对比效果

```python
question = "FastAPI 的路由函数用 async def 定义时，底层是怎么执行的？"

# 基础版
r1 = query_engine.query(question)
print(f"基础版: {r1}\n参考 {len(r1.source_nodes)} 个片段\n")

# Reranker 版
r2 = query_engine_rerank.query(question)
print(f"Reranker 版: {r2}\n参考 {len(r2.source_nodes)} 个片段")
```

你会发现 Reranker 版的答案通常更精准，尤其在知识库内容较多时效果更明显。

---

## Step 5: 对话模式

问答系统只能一问一答？太单调了。我们用 `ChatEngine` 实现多轮对话。

```python
from llama_index.core.memory import ChatMemoryBuffer

# 创建带记忆的 ChatEngine
memory = ChatMemoryBuffer.from_defaults(token_limit=3000)

chat_engine = index.as_chat_engine(
    chat_mode="condense_question",  # 把多轮对话浓缩为一个新的查询
    memory=memory,
    verbose=True,  # 打印中间过程
)
```

### 多轮对话测试

```python
# 第一轮
r1 = chat_engine.chat("asyncio 有哪些核心概念？")
print(f"Agent: {r1}\n")

# 第二轮——"它们"指代上一轮的内容，需要记忆才能理解
r2 = chat_engine.chat("它们之间是怎么配合工作的？")
print(f"Agent: {r2}\n")

# 第三轮——继续深入
r3 = chat_engine.chat("给一个实际例子，用 asyncio 并发爬取 3 个网页")
print(f"Agent: {r3}")
```

`condense_question` 模式的原理：把历史对话和当前问题交给 LLM，让它生成一个"自包含"的查询（不再依赖上下文才能理解），然后用这个查询去检索。这样即使对话进行了很多轮，检索依然精准。

### 重置对话

```python
chat_engine.reset()  # 清空记忆，开始新对话
```

---

## Step 6: 接入 MCP Server

这是最有价值的一步——把知识库封装为标准 MCP Server，让任何支持 MCP 协议的 Agent（Claude Desktop、Cursor、你自己的 Agent）都能直接调用。

```python
# mcp_server.py
from mcp.server.fastmcp import FastMCP
from llama_index.core import VectorStoreIndex, Settings, StorageContext
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.llms.openai import OpenAI
from llama_index.vector_stores.chroma import ChromaVectorStore
import chromadb
import os

os.environ["OPENAI_API_KEY"] = "sk-..."

# ---- 初始化索引（和前面一样） ----
Settings.llm = OpenAI(model="gpt-4o-mini", temperature=0)
Settings.embed_model = OpenAIEmbedding(model="text-embedding-3-small")

chroma_client = chromadb.PersistentClient(path="./chroma_db")
chroma_collection = chroma_client.get_or_create_collection("tech_docs")
vector_store = ChromaVectorStore(chroma_collection=chroma_collection)
storage_context = StorageContext.from_defaults(vector_store=vector_store)

# 从已有向量库加载索引（不需要重新构建）
index = VectorStoreIndex.from_vector_store(vector_store)
query_engine = index.as_query_engine(similarity_top_k=5)

# ---- 创建 MCP Server ----
mcp = FastMCP("TechDocsKB", instructions="技术文档知识库问答服务。可以回答关于 Python asyncio、FastAPI 等技术的问题。")

@mcp.tool()
def ask_knowledge_base(question: str) -> str:
    """向知识库提问，返回答案和参考来源。
  
    Args:
        question: 要问的技术问题
    """
    response = query_engine.query(question)
  
    sources = []
    for node in response.source_nodes:
        source = node.metadata.get("file_name", "unknown")
        sources.append(f"- {source} (相关度: {node.score:.2f})")
  
    return f"**答案：**\n{response}\n\n**参考来源：**\n" + "\n".join(sources)

@mcp.tool()
def search_documents(query: str, top_k: int = 5) -> str:
    """搜索知识库中的相关文档片段，不生成答案，只返回原文。
  
    Args:
        query: 搜索关键词
        top_k: 返回结果数量
    """
    retriever = index.as_retriever(similarity_top_k=top_k)
    nodes = retriever.retrieve(query)
  
    results = []
    for i, node in enumerate(nodes):
        source = node.metadata.get("file_name", "unknown")
        results.append(f"### 结果 {i+1} ({source}, 相关度: {node.score:.2f})\n{node.text}")
  
    return "\n\n---\n\n".join(results)

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

### 配置 MCP Client

在 Claude Desktop 的 `claude_desktop_config.json` 中添加：

```json
{
  "mcpServers": {
    "tech-docs-kb": {
      "command": "python",
      "args": ["/path/to/mcp_server.py"]
    }
  }
}
```

配好后，Agent 就可以通过 MCP 协议调用 `ask_knowledge_base` 和 `search_documents` 工具来访问你的知识库了。**知识库变成了一个"可被 Agent 调用的技能"。**

---

## 效果对比实验

我们用一个具体问题，对比三种配置的效果：

```python
question = "asyncio 并发请求和同步请求的性能差异有多大？具体怎么用？"

# 配置 1: 基础 RAG（top_k=2，无后处理）
engine_basic = index.as_query_engine(similarity_top_k=2)

# 配置 2: 加 Reranker（先检索 top-10，Rerank 保留 top-3）
engine_rerank = index.as_query_engine(
    similarity_top_k=10,
    node_postprocessors=[
        SentenceTransformerRerank(model="cross-encoder/ms-marco-MiniLM-L-6-v2", top_n=3)
    ]
)

# 配置 3: Hybrid Search（向量 + 关键词混合检索）
# 需要 BM25 支持
from llama_index.core.retrievers import BM25Retriever
from llama_index.core.query_engine import RetrieverQueryEngine
from llama_index.core import QueryBundle

bm25_retriever = BM25Retriever.from_defaults(
    docstore=index.docstore, similarity_top_k=5
)
vector_retriever = index.as_retriever(similarity_top_k=5)

# 合并两种检索结果
from llama_index.core.retrievers import RouterRetriever
from llama_index.core.query_engine import RetrieverQueryEngine

# 简单方式：分别检索再合并
def hybrid_query(query_str):
    vector_nodes = vector_retriever.retrieve(query_str)
    bm25_nodes = bm25_retriever.retrieve(query_str)
  
    # 去重合并
    seen_ids = set()
    all_nodes = []
    for node in vector_nodes + bm25_nodes:
        if node.node_id not in seen_ids:
            seen_ids.add(node.node_id)
            all_nodes.append(node)
  
    # 取相关度最高的 5 个
    all_nodes.sort(key=lambda n: n.score, reverse=True)
    return all_nodes[:5]

# 测试对比
r1 = engine_basic.query(question)
r2 = engine_rerank.query(question)

print("=== 基础 RAG ===")
print(f"答案: {r1}")
print(f"参考片段数: {len(r1.source_nodes)}")
print(f"参考来源: {[n.metadata.get('file_name') for n in r1.source_nodes]}")

print("\n=== Reranker RAG ===")
print(f"答案: {r2}")
print(f"参考片段数: {len(r2.source_nodes)}")
print(f"参考来源: {[n.metadata.get('file_name') for n in r2.source_nodes]}")
```

### 实验结论

| 配置               | 检索精准度 | 答案质量 | 延迟 | 适用场景                         |
| ------------------ | ---------- | -------- | ---- | -------------------------------- |
| 基础 RAG (top_k=2) | 中         | 基础可用 | ~1s  | 快速原型、简单问题               |
| + Reranker         | 高         | 明显更好 | ~2s  | 生产环境、精准问答               |
| + Hybrid Search    | 最高       | 最好     | ~3s  | 关键词敏感场景（错误码、函数名） |

**核心洞察**：向量检索擅长语义匹配（"怎么并发"能找到"asyncio.gather"），但不擅长精确关键词匹配（搜索 `aiohttp.ClientSession` 可能漏掉包含这个精确字符串的文档）。Hybrid Search 用 BM25 弥补了这个短板。

---

## 关键收获

1. **RAG 质量的关键瓶颈是检索，不是生成**。模型够用了，检索到对的文档才是胜负手。
2. **chunk_size 不是越大越好**。太大引入噪声，太小丢失上下文——512 是多数技术文档的甜区。
3. **Reranker 是性价比最高的提升手段**。加一行代码，答案质量提升明显，成本几乎为零（本地模型）。
4. **Hybrid Search 解决向量检索的盲区**。当你需要搜函数名、错误码等精确字符串时，BM25 比向量检索靠谱得多。
5. **MCP 让知识库变成 Agent 的标准能力**。不再需要每个 Agent 自己实现 RAG，调用 MCP Server 即可。

---

## 第三阶段总结（09-12）

回顾一下这个阶段我们学了什么：

| 篇章 | 主题                | 核心能力                       |
| ---- | ------------------- | ------------------------------ |
| 09   | Context Engineering | 给模型"看什么"比"怎么说"更重要 |
| 10   | Agent Memory        | 让 Agent 拥有短期和长期记忆    |
| 11   | RAG 原理            | 检索增强生成的理论基础和架构   |
| 12   | LlamaIndex 实战     | 从理论到可运行的知识库系统     |

这四篇构成了一个完整的"Agent 知识能力"体系：Context Engineering 告诉你**信息环境怎么构建**，Memory 告诉你**怎么跨对话保留知识**，RAG 告诉你**怎么从外部知识库检索信息**，最后 LlamaIndex 实战把这些全部落地。

---

## 下一阶段预告：多 Agent 与 Harness Engineering

一个 Agent 能做的事终究有限。下一阶段我们进入更激动人心的领域：

- **多 Agent 协作**：让多个专业 Agent 分工合作，像团队一样完成复杂任务
- **Agent 通信协议**：Agent 之间怎么传递信息、协调决策
- **Harness Engineering**：怎么测试、监控、调试 Agent 系统——这是生产化的关键
- **评估框架**：怎么量化衡量 Agent 的表现，而不只是"感觉还不错"

---

## 扩展练习

1. **替换 LLM**：把 `gpt-4o-mini` 换成 Ollama 本地模型（比如 Llama 3），体验全流程本地化运行
2. **增量更新**：往 `data/` 目录添加新文档，不重建整个索引，只追加新节点
3. **流式输出**：用 `query_engine.query` 的 streaming 模式，实现打字机效果
4. **评估 RAG 质量**：用 `llama-index` 内置的评估模块，量化测试你的检索和生成质量
5. **多知识库路由**：构建多个 Index（比如"Python 库"和"DevOps 工具"），根据问题类型自动选择检索哪个

---

## 推荐资源

- [LlamaIndex 官方文档](https://docs.llamaindex.ai/) — 最权威的参考
- [LlamaIndex Starter Pack](https://github.com/run-llama/llama_index/tree/main/docs/docs/examples) — 大量官方示例
- [Chroma 文档](https://docs.trychroma.com/) — 本文使用的向量数据库
- [Sentence Transformers](https://www.sbert.net/) — Reranker 模型的来源
- [MCP 协议规范](https://modelcontextprotocol.io/) — 理解 MCP Server 的设计哲学
- [RAG 评估框架 RAGAS](https://docs.ragas.io/) — 系统评估 RAG 系统质量

---

*下一站：多 Agent 系统——当一个 Agent 不够用时，怎么让一群 Agent 协作？*
