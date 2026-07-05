---
title: "RAG Pipeline 实战：从文档切分到检索增强的完整链路"
date: 2026-07-03
draft: false
tags: ["RAG", "LLM", "Agent"]
summary: "搭建生产级 RAG 系统的全流程：文档切分策略、Embedding 模型选型、向量数据库对比、检索重排，以及 Prompt 注入的最佳实践。"
ShowToc: true
---

RAG（Retrieval-Augmented Generation）是让 LLM 基于私有知识库回答的核心技术。这篇文章记录搭建完整 RAG pipeline 的实战经验。

## 文档切分策略

切分质量直接决定检索质量。两种主流方案：

### 递归字符切分

按 `\n\n` → `\n` → sentence → word 逐层递归。简单高效，适合大多数场景。推荐配置：

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,       # 每块最大字符数
    chunk_overlap=64,     # 块间重叠字符数
    separators=["\n\n", "\n", "。", "，", " "]
)
chunks = splitter.split_text(document)
```

### 语义切分

用 embedding 相似度判断语义断点。效果更好但成本高 3-5 倍。适合法律、医学等精度要求高的领域。

> **实测数据**：在内部技术文档 QA 任务上，语义切分比递归切分的 Answer Accuracy 高 4.2%，但索引时间增加了 3.7 倍。

## Embedding 模型选型

当前（2026 年）MTEB 基准上表现最好的开源 embedding 模型：

| 模型 | 维度 | 多语言 | 推荐场景 |
|------|------|--------|---------|
| BGE-M3 | 1024 | 支持 | 中文场景首选，多语言+多粒度 |
| GTE-Qwen2 | 1024 | 支持 | 通用场景，性能优异 |
| Jina-Embeddings-v3 | 1024 | 支持 | 长文本友好（8192 tokens） |

中文场景推荐 **BGE-M3**，兼顾多语言和检索质量。

## 向量数据库对比

```
小规模（<100K docs）  → Chroma（零配置，嵌入式）
中等规模（100K-1M）   → Milvus（功能全，可分布式）
大规模（>1M）         → Pinecone / Weaviate（托管服务）
不想运维              → Pinecone / Supabase pgvector
```

## 检索与重排

推荐**两阶段检索**：

```python
# Stage 1: 粗召回 — hybrid search（向量 + BM25）
candidates = hybrid_search(query, top_k=20)

# Stage 2: 精排 — cross-encoder reranker
reranked = reranker.rerank(query, candidates, top_k=5)
```

实测 reranker 能将 Top-5 准确率提升 8-12%。推荐模型：BGE-Reranker-v2-M3。

## Prompt 注入

将检索到的 context 注入 prompt 时的最佳实践：

```python
def build_rag_prompt(query: str, contexts: list[str]) -> str:
    refs = "\n\n".join(
        f"[{i+1}] {ctx}" for i, ctx in enumerate(contexts)
    )
    return f"""基于以下参考资料回答问题。如果资料中没有相关信息，请明确说明。

## 参考资料
{refs}

## 问题
{query}

## 要求
- 引用来源编号（如 [1]、[2]）
- 如果多个来源有冲突信息，列出所有观点
- 不要编造资料中没有的内容"""
```

关键原则：按相关性排序 context、标注来源编号、设置 token 上限避免 overflow。
