---
title: "10 | RAG 入门——让 LLM 用你的私有知识回答问题"
date: 2026-07-07
draft: false
weight: 302
tags: ["Agent", "RAG", "Context Engineering"]
summary: "RAG 全流程详解：文档切分策略、Embedding 模型选择、向量检索原理、重排优化、Prompt 注入。每个环节配具体数值和代码示例。"
ShowToc: true
---

上一篇我们讲了 Context Engineering——"给模型看什么"比"怎么说"更重要。但有一个关键问题没展开：**那些"该给模型看"的文档，从哪里来？**

现实中有两个令人头疼的场景：

**场景 1：知识截止。** 你问 GPT-4 "2025 年 Q1 英伟达的财报数据"，它说不知道——因为训练数据截止到某个时间点，之后的世界对它来说是盲区。

**场景 2：私有知识。** 你问"我们公司的报销流程是什么"，任何通用 LLM 都答不上来——这些知识从未出现在它的训练数据里。

**RAG（Retrieval-Augmented Generation）** 就是为了解决这两个问题而生的：**先从外部知识库检索出相关文档，再把这些文档塞进 Prompt 让 LLM 生成回答。** 不训练模型，不微调权重，只需要一个好的检索系统。

![RAG 全流程概览](/images/stage3-10-rag-basics.svg)

---

## RAG 全流程：三步走

RAG 的流程可以拆成三个阶段：

```
Indexing（离线）          Retrieval（在线）        Generation（在线）
文档 → 切分 → 向量化      用户提问 → 向量化         检索结果 + 问题
         → 存入向量库          → 在向量库中检索          → 交给 LLM 生成回答
                              → （可选）重排
```

**Indexing** 是离线预处理：把文档切成小块、转成向量、存进数据库。只做一次（除非文档更新）。

**Retrieval** 是在线检索：用户提问时，把问题也转成向量，在数据库里找最相似的文档块。

**Generation** 是在线生成：把检索到的文档块和问题一起交给 LLM，生成最终回答。

下面逐个环节拆解。

---

## 文档切分（Chunking）

### 为什么要切分？

一份 PDF 可能有 5 万字。你不可能把整本书塞进 Prompt——不仅超过窗口限制，大量无关内容还会干扰模型。**切分的目标是把长文档拆成语义完整的小块，每块足够小以精准检索，又足够大以保留上下文。**

### 两种主要策略

**1. 固定大小切分（Fixed-size Chunking）**

按字符数或 token 数切分，最简单也最常用：

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,       # 每块最多 512 个字符
    chunk_overlap=64,     # 相邻两块重叠 64 个字符
    separators=["\n\n", "\n", "。", ".", " "]  # 按优先级尝试分割
)

chunks = splitter.split_text(document_text)
# 一篇 10,000 字的文档 → 大约 22 个 chunk（考虑 overlap）
```

关键参数怎么选？

| 参数 | 推荐值 | 理由 |
|------|--------|------|
| `chunk_size` | 256-1024 字符 | 太小（<128）丢语义，太大（>2048）检索不精准 |
| `chunk_overlap` | chunk_size 的 10%-20% | 防止句子被截断在边界，保留上下文连续性 |

经验法则：**如果你不确定，从 chunk_size=512、overlap=64 开始，然后根据效果调整。** 这个配置适合大多数技术文档和知识库场景。

**2. 语义切分（Semantic Chunking）**

不按固定大小，而是按语义边界（段落、标题、章节）切分：

```python
# 按 Markdown 标题切分
from langchain.text_splitter import MarkdownHeaderTextSplitter

md_splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=[
        ("#", "h1"),
        ("##", "h2"),
        ("###", "h3"),
    ]
)

# 一篇技术文档会被按章节自然切分
# 每个 chunk 自带标题元数据，检索时更精准
```

语义切分通常比固定大小切分效果好，但需要对文档格式有一定了解。**实践中常见的组合是：先用 Markdown 标题做粗切分，如果某个章节太长，再用固定大小做二次切分。**

---

## Embedding 模型：把文本变成向量

### 直觉理解

Embedding 就是把一段文本映射成一个高维空间中的点（向量）。**语义相似的文本，在空间中的距离更近。**

```
"Python 异步编程"  →  [0.12, -0.34, 0.56, ..., 0.78]   (1536 维)
"asyncio 教程"     →  [0.11, -0.31, 0.54, ..., 0.76]   ← 距离很近
"红烧肉的做法"     →  [-0.89, 0.45, -0.12, ..., 0.03]  ← 距离很远
```

这样，"检索相似文档"就变成了"在高维空间中找最近的点"——一个纯数学问题。

### 常见模型对比

| 模型 | 维度 | 最大 tokens | 特点 |
|------|------|-------------|------|
| OpenAI text-embedding-3-small | 1536 | 8191 | 性价比高，适合英文和中文 |
| OpenAI text-embedding-3-large | 3072 | 8191 | 精度更高，适合需要细粒度区分的场景 |
| BGE-M3 (BAAI) | 1024 | 8192 | 开源，多语言表现优秀，中文尤其好 |
| Cohere embed-v3 | 1024 | 512 | 支持搜索专用模型（search_document vs search_query） |

### 维度对效果的影响

直觉上，维度越高，能编码的信息越多，区分细微语义差异的能力越强。但维度越高也意味着：

- 存储成本翻倍（3072 维 vs 1024 维 = 3 倍存储）
- 检索速度更慢
- 边际收益递减——从 1024 到 3072 的提升远没有从 256 到 1024 大

**实用建议：1024 维对大多数应用已经足够。如果你的场景需要区分非常相似的技术概念（比如不同版本的 API 文档），可以考虑 3072 维。**

```python
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(
    model="text-embedding-3-small",  # 1536 维
    # 或 "text-embedding-3-large"  # 3072 维
)

# 将文本块批量转为向量
vectors = embeddings.embed_documents([chunk.text for chunk in chunks])
# 100 个 chunk → 100 个 1536 维向量
```

---

## 向量数据库：存储和检索

### 它在做什么？

向量数据库做两件事：**存储向量** + **快速找到最相似的向量**。

"找最相似"在技术上叫 Approximate Nearest Neighbor（ANN）搜索。你不需要理解算法细节（HNSW、IVF 之类的），只需要知道：**一个好的向量数据库能在百万级向量中，毫秒级返回最相似的 top_k 个结果。**

### 常见选型

| 数据库 | 定位 | 适用场景 |
|--------|------|----------|
| **ChromaDB** | 轻量嵌入式 | 本地开发、原型验证、<100 万向量 |
| **FAISS** | Meta 开源的库 | 高性能检索研究、大批量离线处理 |
| **Pinecone** | 云托管服务 | 生产环境、需要自动扩缩容 |
| **Milvus** | 分布式数据库 | 大规模生产、需要高可用和水平扩展 |
| **Qdrant** | Rust 写的向量库 | 注重性能，支持丰富的过滤条件 |

**新手建议：用 ChromaDB 入门。** 零配置，pip install 就能用，API 简洁。等到需要上生产再考虑 Pinecone 或 Milvus。

```python
import chromadb

# 零配置，数据存在内存中（也可持久化到磁盘）
client = chromadb.Client()
collection = client.create_collection(
    name="my_docs",
    metadata={"hnsw:space": "cosine"}  # 使用余弦相似度
)

# 存入文档块及其向量
collection.add(
    documents=[chunk.text for chunk in chunks],
    embeddings=vectors,
    ids=[f"chunk_{i}" for i in range(len(chunks))],
    metadatas=[{"source": "api_doc.pdf", "page": 3} for _ in chunks]
)
```

---

## 检索策略

### 相似度搜索（Similarity Search）

最基本的检索方式：计算查询向量和所有文档向量的相似度，返回 top_k 个最相似的。

```python
results = collection.query(
    query_embeddings=[query_vector],
    n_results=5  # 返回最相似的 5 个
)
# results = [{"text": "...", "distance": 0.12}, ...]
# distance 越小越相似（cosine distance）
```

### MMR（Maximal Marginal Relevance）：多样性检索

相似度搜索有个问题：返回的 top_k 可能高度重复。比如你搜"Python 异步编程"，top-5 可能全是同一篇文章的不同段落。

**MMR 在"相似性"和"多样性"之间取平衡：** 不仅找和问题相似的文档，还确保返回的文档之间彼此不太重复。

```python
from langchain_community.vectorstores import Chroma

vectorstore = Chroma.from_documents(chunks, embeddings)

# 普通相似度搜索
docs_sim = vectorstore.similarity_search("Python 异步编程", k=5)
# 可能返回 5 个高度重复的 chunk

# MMR 搜索（lambda_mult 控制多样性，0=最多样，1=最相似）
docs_mmr = vectorstore.max_marginal_relevance_search(
    "Python 异步编程",
    k=5,
    lambda_mult=0.7  # 0.7 偏向相似，0.3 偏向多样
)
# 返回 5 个覆盖面更广的 chunk
```

### top_k 怎么选？

| top_k | 适用场景 | 代价 |
|-------|----------|------|
| 3-5 | 简单问答，答案在少数文档中 | token 消耗低 |
| 5-10 | 需要综合多个来源的复杂问题 | 中等 |
| 10-20 | 探索性查询，需要广泛信息 | token 消耗高，可能引入噪声 |

**实用建议：从 top_k=5 开始。** 如果发现回答不够全面就加大，如果发现回答被无关信息干扰就减小。

---

## 重排（Reranking）：精排提升质量

### 为什么需要重排？

向量检索（也叫"初检"或"召回"）用的是 Embedding 模型——它把查询和文档各自独立编码成向量，然后算距离。这种"双编码器"架构速度快，但精度有上限：**它无法捕捉查询和文档之间的细粒度交互。**

比如查询"如何配置 Nginx 反向代理"，两个候选文档：
- 文档 A：讲 Nginx 安装和基础配置（向量距离近，但没讲反向代理）
- 文档 B：讲用 Caddy 配置反向代理（向量距离稍远，但主题更匹配）

Embedding 模型可能给 A 更高的分数，因为"Nginx"这个词更匹配。但实际上 B 的主题（反向代理配置）更切题。

### Cross-encoder 重排

**Cross-encoder 把查询和文档拼在一起输入模型，做一次完整的注意力计算，输出一个精确的相关度分数。** 它比 Embedding 模型慢得多（不能预计算），所以只用来对初检的 top-N 结果做精排。

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain_cohere import CohereRerank

# 第一步：向量检索召回 top-20（宽松召回）
base_retriever = vectorstore.as_retriever(search_kwargs={"k": 20})

# 第二步：用 Cross-encoder 重排，保留 top-5
reranker = CohereRerank(model="rerank-multilingual-v3.0", top_n=5)

# 组合：先粗召回，再精排
compression_retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=base_retriever
)

# 使用
docs = compression_retriever.invoke("如何配置 Nginx 反向代理")
# 返回经过重排的 top-5，质量显著优于纯向量检索
```

重排的效果提升通常是显著的——在 MTEB 等评测中，**加入重排后检索准确率（nDCG@5）可以提升 5-15 个百分点。** 代价是延迟增加 200-500ms（取决于重排模型和候选数量），对大多数应用可以接受。

---

## Prompt 注入：把检索结果交给 LLM

检索到了相关文档，最后一步是把它们组装成 Prompt。这一步看似简单，但细节决定回答质量。

### 基础模板

```python
RAG_PROMPT_TEMPLATE = """你是一个专业的技术文档助手。请基于以下参考文档回答用户的问题。

## 参考文档
{context}

## 回答要求
1. 只基于参考文档中的信息回答，不要编造
2. 如果参考文档中没有足够信息，明确说"根据现有文档无法确定"
3. 引用具体的文档来源（如 [文档1]）
4. 用中文回答，技术术语保留英文

## 用户问题
{question}
"""

def format_docs(docs):
    """把检索结果格式化为上下文"""
    formatted = []
    for i, doc in enumerate(docs):
        source = doc.metadata.get("source", "未知")
        formatted.append(f"[文档{i+1}] (来源: {source})\n{doc.page_content}")
    return "\n\n".join(formatted)

# 组装最终 Prompt
final_prompt = RAG_PROMPT_TEMPLATE.format(
    context=format_docs(docs),
    question=user_question
)
```

### 几个容易犯的错误

1. **不设"不知道"兜底。** 如果 Prompt 不说"信息不足时请明说"，LLM 会用训练数据中的知识胡编——这叫 Hallucination，RAG 最大的敌人之一。

2. **文档没有标注来源。** 不标注来源，LLM 无法在回答中引用，用户也无法验证。

3. **上下文太长。** 塞入 20 个文档块（每个 512 字符）= 10,240 字符 ≈ 3,000+ tokens。如果模型窗口只有 8K，就挤占了思考空间。回顾上一篇的原则：**P1 级文档占 20-40% 窗口为宜。**

---

## 端到端代码示例

下面是一个完整的 RAG 系统，用 LangChain + ChromaDB + BGE Embedding（本地） + DeepSeek 实现：

```python
# pip install langchain langchain-openai langchain-chroma langchain-huggingface chromadb python-dotenv

import os
from dotenv import load_dotenv
load_dotenv()

from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough
from langchain_openai import ChatOpenAI
from langchain_huggingface import HuggingFaceEmbeddings
from langchain_chroma import Chroma

# ============ 1. 准备文档 ============
documents = [
    "LangChain 是一个用于构建 LLM 应用的框架。它提供了 Chain、Agent、Memory 等核心抽象。",
    "Chain 是 LangChain 中的核心概念，表示一系列按顺序执行的组件。可以用 LCEL 构建。",
    "Agent 使用 LLM 作为推理引擎，自主决定调用哪些工具。ReAct 是最常用的 Agent 架构。",
    "RAG（检索增强生成）通过检索外部知识库来增强 LLM 的回答能力，减少幻觉。",
    "向量数据库用于存储和检索文本的向量表示。常见选择包括 Chroma、FAISS、Pinecone。"
]
# 这里文档已经是短文本，不需要切分
# 实际项目中，你会用 RecursiveCharacterTextSplitter 切分长文档

# ============ 2. 创建本地 Embedding ============
# 首次运行会从 HuggingFace 下载 BAAI/bge-large-zh-v1.5，约 1.2GB
embeddings = HuggingFaceEmbeddings(
    model_name="BAAI/bge-large-zh-v1.5",
    model_kwargs={"device": "cpu"},  # 没有 GPU 就用 cpu
    encode_kwargs={"normalize_embeddings": True}
)

# ============ 3. 创建向量存储 ============
vectorstore = Chroma.from_texts(
    texts=documents,
    embedding=embeddings
)

retriever = vectorstore.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 2}
)

# ============ 4. 构建 Prompt ============
RAG_PROMPT = ChatPromptTemplate.from_template("""
你是一个专业的 AI 技术助手。请基于以下参考文档回答问题。

参考文档：
{context}

要求：
- 只基于参考文档回答，不确定时明确说明
- 引用文档编号 [1] [2] 等
- 用中文回答，技术术语保留英文

问题：{question}
""")

# ============ 5. 构建 RAG Chain ============
llm = ChatOpenAI(model="deepseek-v4-flash", temperature=0, max_tokens=1000)

def format_docs(docs):
    return "\n\n".join(
        f"[{i+1}] {doc.page_content}" for i, doc in enumerate(docs)
    )

rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | RAG_PROMPT
    | llm
    | StrOutputParser()
)

# ============ 6. 使用 ============
question = "LangChain 中的 Agent 和 Chain 有什么区别？"
answer = rag_chain.invoke(question)
print(f"问题：{question}\n")
print(f"回答：{answer}")
# 输出会引用 [1] [2] 等文档来源，基于知识库回答
```

这个流程就是完整的 RAG 系统了。代码中用 `dotenv` 加载 `.env` 文件里的 API Key（比如 `OPENAI_API_KEY` 或自定义的 `BASE_URL`），避免硬编码在代码里。Embedding 用的是本地的 BGE 模型（`BAAI/bge-large-zh-v1.5`），首次运行会从 HuggingFace 下载约 1.2GB，之后就不需要联网了——相比 OpenAI 的 Embedding API，本地方案零调用成本，数据也不出本机。

在真实项目中，你还需要考虑：文档加载（PDF、网页、数据库）、增量更新（新文档加入时不重建全部索引）、评估（检索质量和生成质量的量化指标）。

---

## 各环节的数值速查表

| 环节 | 参数 | 推荐值 | 备注 |
|------|------|--------|------|
| Chunking | chunk_size | 512 字符 | 技术文档的甜点值 |
| Chunking | chunk_overlap | 64 字符（~12%） | 防止语义截断 |
| Embedding | 维度 | 1024-1536 | 1024 够用，1536 更精细 |
| Embedding | batch_size | 100-500 | 太大可能 OOM |
| 向量检索 | top_k | 5 | 简单问答的起点 |
| 向量检索 | MMR lambda_mult | 0.7 | 偏向相似但保留多样性 |
| 重排 | 召回 top-N | 20 | 宽松召回，交给重排精筛 |
| 重排 | 精排 top-k | 5 | 最终进入 Prompt 的文档数 |
| Prompt | 总 token 占比 | 20-40% | 不要挤占模型思考空间 |

---

## 总结

RAG 的核心思路非常简单：**不改模型，改模型看到的上下文。** 通过检索把最相关的文档块喂给 LLM，就能让它回答训练数据之外的问题。

整个流程的每个环节都有优化空间：

- **切分** 决定了文档块的质量和粒度
- **Embedding** 决定了语义表示的精度
- **向量检索** 决定了能不能找到对的信息
- **重排** 决定了最终进入 Prompt 的信息质量
- **Prompt 注入** 决定了 LLM 怎么用这些信息

任何一个环节做得不好，都会影响最终效果。这也是为什么 RAG 看似简单，做好却不容易。

---

## 下一步

这篇文章覆盖了 RAG 的基础流程。但在实践中你会遇到很多进阶问题：

- 多轮对话中怎么做 RAG？（查询改写、上下文继承）
- 文档类型多样（PDF + 表格 + 图片）怎么处理？
- 怎么做 Hybrid Search（关键词检索 + 向量检索混合）？
- 怎么评估 RAG 系统的质量？（Faithfulness、Relevance、RAGAS 框架）

这些是下一篇 **高级 RAG** 的内容。

---

## 推荐资源

- [LangChain RAG 教程](https://python.langchain.com/docs/tutorials/rag/) — 官方入门教程，跟着做一遍就能跑通
- [ChromaDB 文档](https://docs.trychroma.com/) — 轻量向量数据库，适合本地开发和学习
- [Anthropic: RAG 最佳实践](https://docs.anthropic.com/en/docs/prompt-engineering) — Claude 官方的检索增强指南
- [RAG From Scratch（LangChain YouTube 系列）](https://www.youtube.com/playlist?list=PLfaIDFEXeq2LMXKqE34BgGR5hXvJE3xJL) — 14 集视频，每集聚焦 RAG 的一个环节
- [MTEB Leaderboard](https://huggingface.co/spaces/mteb/leaderboard) — Embedding 模型的权威评测榜单，选模型时参考
