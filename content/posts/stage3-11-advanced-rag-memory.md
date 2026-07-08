---
title: "11 | 高级 RAG 与记忆系统——从能用到好用"
date: 2026-07-07
draft: false
weight: 303
tags: ["Agent", "RAG", "记忆系统"]
summary: "超越基础 RAG：Multi-hop 多步检索、Self-RAG 自适应检索、Graph RAG 知识图谱增强，以及 Agent 记忆系统的设计——短期对话记忆、长期知识记忆、记忆压缩策略。"
ShowToc: true
---

上一篇我们讲了 RAG 的基础 pipeline：文档切分 → Embedding → 向量检索 → 注入 Prompt。这套流程能让 Agent "查到知识库里的信息"，但实践中你会很快遇到几个让人头疼的问题：

- 用户问"A 公司的 CEO 在哪所大学读的博士？"——**答案分散在两份文档里**，一次检索根本找不全。
- Agent 明明已经回答过的问题，下次对话又从零开始——**它没有任何记忆**。
- 检索回来的文档"看着相关但其实不对"——**向量相似不等于语义正确**。

这篇文章就解决这些问题。前半部分讲**高级 RAG 技术**，后半部分讲**Agent 记忆系统**。两者的共同目标是：让你的 Agent 从"能用"变成"好用"。

![高级 RAG 与记忆系统](/images/stage3-11-advanced-rag-memory.svg)

---

## Part 1: 高级 RAG 技术

### 基础 RAG 的局限性

回顾一下基础 RAG 的流程：

```
用户问题 → Embedding → 向量数据库检索 Top-K → 注入 Prompt → LLM 生成回答
```

这个流程有三个核心局限：

**1. 单次检索不够用（Single-hop Limitation）**

有些问题的答案不是"一段文字能说清"的。比如：

```
问题："Transformer 论文的第一作者现在在哪家公司？"
```

你需要：先找到"Transformer 论文的第一作者是谁"（Ashish Vaswani），再去找"他现在在哪"。基础 RAG 一次检索只能完成其中一步。

**2. 检索质量不稳定（Relevance Gap）**

向量检索基于 embedding 相似度，但"向量距离近"不等于"对回答问题有用"。比如搜"Python 的 GIL 是什么"，可能检索到一篇讲 Python 安装的文章，因为里面提到了 GIL 这个词——但它显然不是你要的解释。

**3. 无法跨文档推理（Cross-document Reasoning）**

当答案散落在多个文档中，且需要综合、对比或推理时，基础 RAG 的"检索 → 拼接"模式力不从心。

针对这些问题，学术界和工业界提出了一系列高级 RAG 方案。

---

### Multi-hop RAG：多步检索，逐步逼近答案

Multi-hop RAG 的核心思想很直观：**一次检索不够，就检索多次，每次用上次的结果来指导下次。**

举个具体例子：

```
用户问题："李飞飞创立的 AI 实验室获得了哪家公司的投资？"

第 1 步：用原始问题检索
→ 找到文档：李飞飞是斯坦福 HAI 的联合创始人。
→ 提取关键实体：HAI（Stanford Institute for Human-Centered AI）

第 2 步：用"HAI 投资 融资"作为新查询检索
→ 找到文档：HAI 获得了 Google、Microsoft 等多家公司的资助。
→ 综合两步结果，生成最终回答。
```

实现思路：

```python
def multi_hop_rag(question, llm, retriever, max_hops=3):
    """多步检索：每一步基于上一步的结果生成新的查询"""

    context = []
    current_query = question

    for hop in range(max_hops):
        # 1. 检索
        docs = retriever.search(current_query, top_k=3)
        context.extend(docs)

        # 2. 让 LLM 判断：信息够了吗？还缺什么？
        reflection = llm.invoke(f"""
基于以下已检索的信息：
{format_docs(context)}

原始问题：{question}

请判断：
1. 已有信息是否足够回答这个问题？（是/否）
2. 如果不够，还需要检索什么信息？请生成一个新的搜索查询。
""")

        # 3. 如果够了，直接生成回答
        if "是" in reflection.content:
            break

        # 4. 不够的话，用新生成的查询继续检索
        current_query = extract_new_query(reflection.content)

    # 最终回答
    answer = llm.invoke(f"""
基于以下检索到的信息：
{format_docs(context)}

请回答：{question}
""")
    return answer
```

**关键点**：Multi-hop 的本质是把"检索"变成了一个**迭代过程**——每一步都评估"还缺什么"，然后针对性地补查。这比盲目增加 top_k 高效得多。

---

### Self-RAG：让 LLM 自己决定要不要检索

Self-RAG（Self-Reflective RAG）是一个更优雅的升级。它的核心创新是引入了一组 **Reflection Tokens**，让 LLM 在生成过程中自己决定：

1. **Retrieve**：现在需不需要检索？（有些问题模型自己就知道答案）
2. **IsRel**：检索回来的文档跟问题相关吗？
3. **IsSup**：生成的回答被检索到的文档支持吗？
4. **IsUse**：最终回答对用户有用吗？

流程如下：

```
用户问题
  │
  ├─ 模型判断：需要检索吗？
  │   ├─ 不需要 → 直接生成回答
  │   └─ 需要 ↓
  │
  ├─ 检索文档
  │
  ├─ 模型判断：文档相关吗？
  │   ├─ 不相关 → 换策略 / 告知用户
  │   └─ 相关 ↓
  │
  ├─ 基于文档生成回答
  │
  ├─ 模型判断：回答被文档支持吗？
  │   ├─ 不支持 → 重新生成
  │   └─ 支持 ↓
  │
  └─ 输出最终回答
```

这个设计的妙处在于：**不是所有问题都需要检索**。问"1+1等于几"不需要查知识库，问"公司 Q3 营收多少"才需要。Self-RAG 让模型自己判断，避免了不必要的检索开销和无关信息的干扰。

简化的实现思路：

```python
def self_rag(question, llm, retriever):
    """Self-RAG：LLM 自主决定是否需要检索"""

    # Step 1: 判断是否需要检索
    need_retrieval = llm.invoke(f"""
问题：{question}

判断这个问题是否需要外部知识来回答。
- 如果是常识/通用知识，回答"不需要"
- 如果是特定领域/时效性/私有知识，回答"需要"
""")

    if "不需要" in need_retrieval.content:
        # 直接回答
        return llm.invoke(f"请回答：{question}")

    # Step 2: 检索
    docs = retriever.search(question, top_k=5)

    # Step 3: 过滤不相关文档
    relevant_docs = []
    for doc in docs:
        relevance = llm.invoke(f"""
问题：{question}
文档：{doc.text}
这个文档与问题直接相关吗？回答"相关"或"不相关"。
""")
        if "相关" in relevance.content:
            relevant_docs.append(doc)

    if not relevant_docs:
        return "抱歉，我没有找到与您问题相关的信息。"

    # Step 4: 基于相关文档生成回答
    answer = llm.invoke(f"""
基于以下参考文档回答问题。如果文档中没有足够信息，请明确说明。

{format_docs(relevant_docs)}

问题：{question}
""")

    # Step 5: 验证回答是否被文档支持
    verification = llm.invoke(f"""
回答：{answer.content}
参考文档：{format_docs(relevant_docs)}
这个回答是否完全被参考文档支持？回答"支持"或"不支持"。
""")

    if "不支持" in verification.content:
        # 重新生成，更严格地约束
        answer = llm.invoke(f"""
严格基于以下文档回答问题，不要添加文档中没有的信息。
{format_docs(relevant_docs)}
问题：{question}
""")

    return answer
```

Self-RAG 的效果提升主要来自**减少了幻觉**：通过 IsSup（支持性验证），模型被"逼着"检查自己的回答是否有据可查。

---

### Graph RAG：用知识图谱增强检索

基础 RAG 的一个根本局限是：**它只能做"文本相似度匹配"，不能做"关系推理"。**

考虑这个问题："哪些公司的 CEO 是 MIT 毕业的？" 要回答这个问题，你需要知道：
- 各公司的 CEO 是谁（关系：CEO_of）
- 这些 CEO 的教育背景（关系：educated_at）
- 其中哪些是 MIT（过滤条件）

这是一种**图结构的关系查询**，向量检索天然不擅长。

**Microsoft 的 GraphRAG** 项目提供了一个优雅的解决方案：

```
文档 → 实体提取 → 构建知识图谱 → 图检索 + 文本检索 → LLM 回答
```

核心流程：

1. **实体和关系提取**：用 LLM 从文档中抽取实体（人、公司、地点等）和关系
2. **构建知识图谱**：把实体和关系存储为图结构
3. **社区检测**：对图做聚类，发现实体之间的社区结构
4. **双层检索**：结合图遍历和文本检索

```python
# 伪代码：GraphRAG 的实体和关系提取
def extract_entities_and_relations(text, llm):
    """用 LLM 从文本中提取知识图谱的节点和边"""

    result = llm.invoke(f"""
从以下文本中提取所有实体和它们之间的关系。

文本：{text}

请按以下格式输出：
实体：[名称, 类型]
关系：[实体A, 关系类型, 实体B]

示例：
实体：[Satya Nadella, Person]
实体：[Microsoft, Company]
关系：[Satya Nadella, CEO_of, Microsoft]
""")
    return parse_graph_elements(result.content)


# 查询时的图遍历
def graph_rag_query(question, knowledge_graph, text_retriever, llm):
    """Graph RAG：结合图检索和文本检索"""

    # 1. 从问题中提取关键实体
    entities = extract_entities(question, llm)

    # 2. 在知识图谱中找到相关实体和它们的关系
    subgraph = knowledge_graph.get_subgraph(
        entities,
        depth=2  # 探索 2 跳以内的关系
    )

    # 3. 同时做文本检索
    text_docs = text_retriever.search(question, top_k=3)

    # 4. 把图结构和文本一起给 LLM
    answer = llm.invoke(f"""
知识图谱信息：
{format_subgraph(subgraph)}

相关文档：
{format_docs(text_docs)}

请回答：{question}
""")
    return answer
```

**Graph RAG 特别适合这些场景**：
- 需要跨多个文档的关系推理
- 实体之间的关系是核心信息（如企业知识图谱、医疗知识）
- 需要回答"有哪些""分别是什么"类型的综述性问题

GraphRAG 的代价是**构建成本高**：需要额外的实体提取和图构建步骤。但对于企业级知识库来说，这个投入是值得的。

---

### Hybrid Search：向量检索 + 关键词检索的融合

向量检索擅长"语义相似"，关键词检索（BM25）擅长"精确匹配"。两者互补。

一个直觉例子：搜"Python 3.12 asyncio TaskGroup"——
- **向量检索**能找到讲异步编程的文章（语义相关），但可能漏掉提到 TaskGroup 具体 API 的文档
- **BM25 检索**能精确匹配到包含 "TaskGroup" 这个词的文档，但可能返回只是偶然提到这个词的无关文章

把两者融合，取长补短：

```python
from rank_bm25 import BM25Okapi
import numpy as np


class HybridRetriever:
    """混合检索：向量 + BM25"""

    def __init__(self, documents, embeddings, alpha=0.5):
        self.documents = documents
        self.embeddings = embeddings  # 预计算的文档 embedding
        self.bm25 = BM25Okapi([doc.split() for doc in documents])
        self.alpha = alpha  # 向量 vs BM25 的权重

    def search(self, query, query_embedding, top_k=5):
        # 1. 向量检索分数
        vector_scores = cosine_similarity(query_embedding, self.embeddings)

        # 2. BM25 关键词检索分数
        bm25_scores = self.bm25.get_scores(query.split())

        # 3. 归一化到 [0, 1]
        vector_scores = normalize(vector_scores)
        bm25_scores = normalize(bm25_scores)

        # 4. 加权融合
        combined_scores = (
            self.alpha * vector_scores +
            (1 - self.alpha) * bm25_scores
        )

        # 5. 取 Top-K
        top_indices = np.argsort(combined_scores)[-top_k:][::-1]
        return [
            {"text": self.documents[i], "score": combined_scores[i]}
            for i in top_indices
        ]


# 使用
retriever = HybridRetriever(documents, embeddings, alpha=0.6)
results = retriever.search(
    query="Python asyncio TaskGroup",
    query_embedding=embed_model("Python asyncio TaskGroup"),
    top_k=5
)
```

**alpha 参数的调节经验**：
- `alpha=0.7~0.8`：语义为主，适合概念性查询（"什么是注意力机制"）
- `alpha=0.3~0.5`：关键词为主，适合具体实体查询（函数名、产品名、错误代码）
- `alpha=0.5~0.6`：通用平衡，适合大多数混合场景

很多向量数据库（如 Weaviate、Qdrant）已经内置了 Hybrid Search 支持，实际使用时可以直接调用。

---

## Part 2: Agent 记忆系统

### 为什么 Agent 需要记忆？

想象你和一个助手合作：

```
第 1 次对话：
你："帮我写一个 Python 脚本处理 CSV 数据。"
助手："好的，我来用 pandas 写..."

第 2 次对话（第二天）：
你："帮我写一个脚本处理 JSON 数据。"
助手："好的，请问你想用什么语言？用什么库？"
你："……昨天不是刚说过用 Python + pandas 吗？"
```

没有记忆的 Agent 每次交互都从零开始。它不知道你的偏好、你的项目背景、之前做过什么决定。**记忆是让 Agent 从"工具"变成"助手"的关键。**

---

### 三种记忆类型

类比人类记忆系统，Agent 的记忆可以分为三类：

#### 1. 短期记忆（Working Memory）

当前对话的上下文。就是我们在 Context Engineering 那篇讲的内容——当前会话中的消息历史。

```python
# 短期记忆 = 当前对话的消息列表
working_memory = [
    {"role": "user", "content": "帮我优化这个 SQL 查询"},
    {"role": "assistant", "content": "好的，请给我看看查询语句..."},
    {"role": "user", "content": "SELECT * FROM orders WHERE ..."},
    # ...当前会话的所有消息
]
```

特点：**会话结束后清空，不需要持久化。**

#### 2. 长期记忆（Long-term Memory）

跨会话持久化的信息。用户偏好、项目背景、历史决策等。

```python
# 长期记忆 = 持久化存储的用户画像和项目知识
long_term_memory = {
    "user_preferences": {
        "language": "Python",
        "framework": "FastAPI",
        "database": "PostgreSQL",
        "style": "喜欢简洁的代码，不喜欢过度注释"
    },
    "project_context": {
        "name": "DataPilot",
        "description": "数据分析平台",
        "tech_stack": ["Python", "FastAPI", "React", "PostgreSQL"],
        "stage": "MVP 阶段"
    },
    "past_decisions": [
        {"date": "2026-06-15", "decision": "选择 PostgreSQL 而非 MongoDB，因为需要复杂 JOIN"},
        {"date": "2026-06-20", "decision": "用 Celery 做异步任务，而非自建队列"},
    ]
}
```

特点：**持久化存储，跨会话可用，需要检索机制。**

#### 3. 程序性记忆（Procedural Memory）

Agent 学到的操作模式。比如"处理 CSV 文件时应该先检查编码"、"部署到 AWS 时需要检查 IAM 权限"这种经验性知识。

```python
# 程序性记忆 = Agent 学到的操作模式和最佳实践
procedural_memory = [
    {
        "trigger": "用户要求处理 CSV 文件",
        "pattern": "先检测文件编码（chardet），再用 pandas 读取",
        "reason": "中文 CSV 经常是 GBK 编码，直接读会报错"
    },
    {
        "trigger": "用户要求部署到 AWS",
        "pattern": "先检查 IAM 权限，再检查 VPC 配置，最后部署",
        "reason": "权限问题是最常见的部署失败原因"
    },
    {
        "trigger": "用户要求写 SQL 查询",
        "pattern": "先确认表结构和索引情况，再写查询",
        "reason": "不知道索引情况就写查询可能导致性能问题"
    }
]
```

特点：**Agent 通过成功经验积累，可复用的操作模式。**

---

### 记忆存储方案：选哪个？

三种记忆类型对存储的需求不同，方案也不同：

| 记忆类型 | 存储方案 | 理由 |
|---------|---------|------|
| **短期记忆** | 内存（Python list/dict） | 会话内使用，无需持久化 |
| **长期记忆（结构化）** | SQLite / PostgreSQL | 偏好、设置等结构化数据，需要 CRUD 操作 |
| **长期记忆（语义化）** | 向量数据库（Chroma/Pinecone） | 历史对话、经验等需要语义检索的内容 |
| **程序性记忆** | JSON 文件 / SQLite | 模式匹配触发，结构简单 |

一个实用的组合方案：

```python
import sqlite3
import chromadb


class AgentMemory:
    """Agent 记忆系统：SQLite + 向量数据库"""

    def __init__(self, user_id):
        self.user_id = user_id

        # 结构化记忆用 SQLite
        self.db = sqlite3.connect(f"memory_{user_id}.db")
        self.db.execute("""
            CREATE TABLE IF NOT EXISTS user_profile (
                key TEXT PRIMARY KEY,
                value TEXT,
                updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        self.db.execute("""
            CREATE TABLE IF NOT EXISTS decisions (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                date TEXT,
                decision TEXT,
                context TEXT
            )
        """)

        # 语义记忆用向量数据库
        self.vector_db = chromadb.Client()
        self.conversation_store = self.vector_db.get_or_create_collection(
            name=f"conversations_{user_id}",
            metadata={"hnsw:space": "cosine"}
        )

    def save_preference(self, key, value):
        """保存用户偏好"""
        self.db.execute(
            "INSERT OR REPLACE INTO user_profile (key, value) VALUES (?, ?)",
            (key, value)
        )
        self.db.commit()

    def get_preferences(self):
        """获取所有用户偏好"""
        cursor = self.db.execute("SELECT key, value FROM user_profile")
        return dict(cursor.fetchall())

    def save_conversation(self, conversation_text, metadata=None):
        """保存对话到向量数据库（用于语义检索）"""
        self.conversation_store.add(
            documents=[conversation_text],
            ids=[f"conv_{len(self.conversation_store.get()['ids'])}"],
            metadatas=[metadata or {}]
        )

    def recall_conversation(self, query, top_k=3):
        """语义检索历史对话"""
        results = self.conversation_store.query(
            query_texts=[query],
            n_results=top_k
        )
        return results["documents"][0]
```

---

### 记忆检索：在需要时找到相关记忆

存储只是第一步，更重要的是**在正确的时机检索出正确的记忆**。

```python
class MemoryRetriever:
    """记忆检索器：根据当前对话自动检索相关记忆"""

    def __init__(self, memory: AgentMemory):
        self.memory = memory

    def retrieve_relevant_memories(self, current_query, llm):
        """综合检索：结构化偏好 + 语义历史"""

        memories = []

        # 1. 始终注入用户偏好（数据量小，直接全量注入）
        prefs = self.memory.get_preferences()
        if prefs:
            pref_text = ", ".join(f"{k}: {v}" for k, v in prefs.items())
            memories.append(f"[用户偏好] {pref_text}")

        # 2. 语义检索历史对话（数据量大，按相关度检索）
        relevant_convos = self.memory.recall_conversation(current_query, top_k=3)
        for i, convo in enumerate(relevant_convos):
            memories.append(f"[历史对话 {i+1}] {convo}")

        # 3. 检索相关决策记录
        decisions = self._search_decisions(current_query, llm)
        for d in decisions:
            memories.append(f"[历史决策] {d['date']}: {d['decision']}")

        return memories

    def _search_decisions(self, query, llm):
        """用 LLM 辅助判断哪些历史决策与当前问题相关"""
        cursor = self.memory.db.execute(
            "SELECT date, decision, context FROM decisions ORDER BY date DESC LIMIT 20"
        )
        all_decisions = [
            {"date": r[0], "decision": r[1], "context": r[2]}
            for r in cursor.fetchall()
        ]

        if not all_decisions:
            return []

        # 让 LLM 筛选相关的决策
        relevant = llm.invoke(f"""
当前问题：{query}

历史决策列表：
{format_list(all_decisions)}

请选出与当前问题相关的决策，返回编号（逗号分隔）。如果没有相关的，返回"无"。
""")

        if "无" in relevant.content:
            return []

        indices = parse_indices(relevant.content)
        return [all_decisions[i] for i in indices if i < len(all_decisions)]
```

---

### 记忆压缩和遗忘策略

不是所有记忆都值得永久保留。就像人类会遗忘不重要的细节，Agent 的记忆也需要**主动管理**：

#### 策略 1：时间衰减

越久远的记忆权重越低，超过阈值的自动归档：

```python
def decay_memories(conversation_store, days_threshold=30):
    """时间衰减：超过 N 天的记忆降低优先级"""
    all_docs = conversation_store.get(include=["metadatas"])

    for i, doc_id in enumerate(all_docs["ids"]):
        metadata = all_docs["metadatas"][i]
        days_old = (datetime.now() - parse_date(metadata["date"])).days

        if days_old > days_threshold:
            # 标记为"低优先级"而非删除
            metadata["priority"] = "low"
            conversation_store.update(
                ids=[doc_id],
                metadatas=[metadata]
            )
```

#### 策略 2：摘要压缩

当某个主题的对话太多时，压缩为一段摘要：

```python
def compress_topic_conversations(conversation_store, topic, llm):
    """把同一主题的多段对话压缩为一段摘要"""

    # 找到该主题的所有对话
    results = conversation_store.query(
        query_texts=[topic],
        n_results=20
    )

    if len(results["documents"][0]) < 3:
        return  # 太少，不需要压缩

    # 让 LLM 压缩
    summary = llm.invoke(f"""
将以下关于"{topic}"的多段对话压缩为一段简洁的摘要。
保留关键结论和决策，省略具体的来回讨论过程。

对话内容：
{chr(10).join(results['documents'][0])}
""")

    # 删除原始对话，存入压缩后的摘要
    old_ids = results["ids"][0]
    conversation_store.delete(ids=old_ids)
    conversation_store.add(
        documents=[summary.content],
        ids=[f"summary_{topic}_{datetime.now().isoformat()}"],
        metadatas=[{"type": "compressed_summary", "topic": topic}]
    )

    return summary.content
```

#### 策略 3：冲突消解

当新信息和旧记忆矛盾时，以新信息为准：

```python
def resolve_conflict(new_info, old_memory, llm):
    """检测并解决记忆冲突"""

    check = llm.invoke(f"""
新信息：{new_info}
旧记忆：{old_memory}

这两条信息是否矛盾？如果矛盾，请指出具体矛盾点。
回答格式：矛盾/不矛盾 + 原因
""")

    if "矛盾" in check.content and "不矛盾" not in check.content:
        # 以新信息为准，更新旧记忆
        return new_info
    return old_memory
```

---

### LangChain 和 LlamaIndex 的记忆模块

主流框架都提供了记忆模块，开箱即用：

#### LangChain Memory

```python
from langchain.memory import ConversationBufferMemory
from langchain.memory import ConversationSummaryMemory
from langchain.memory import VectorStoreRetrieverMemory

# 1. 简单缓冲记忆（保留最近 N 轮）
buffer_memory = ConversationBufferMemory(
    memory_key="chat_history",
    return_messages=True
)

# 2. 摘要记忆（自动压缩历史对话）
summary_memory = ConversationSummaryMemory(
    llm=llm,
    memory_key="chat_history",
    return_messages=True
)

# 3. 向量存储记忆（语义检索历史）
vector_memory = VectorStoreRetrieverMemory(
    retriever=vectorstore.as_retriever(search_kwargs=dict(k=3)),
    memory_key="chat_history"
)
```

#### LlamaIndex Memory

```python
from llama_index.core.memory import ChatMemoryBuffer
from llama_index.core.storage.chat_store import SimpleChatStore

# 持久化聊天存储
chat_store = SimpleChatStore()

memory = ChatMemoryBuffer.from_defaults(
    token_limit=3000,        # 限制记忆 token 数
    chat_store=chat_store,
    chat_store_key="user_1"  # 每个用户一个 key
)

# 保存到磁盘
chat_store.persist(persist_path="./chat_store.json")

# 加载
chat_store = SimpleChatStore.from_persist_path("./chat_store.json")
```

**选择建议**：
- **简单聊天机器人**：LangChain `ConversationBufferMemory` 足够
- **长期运行的 Agent**：用向量存储记忆 + SQLite 结构化数据
- **需要跨用户共享知识**：用向量数据库做共享知识库

---

## 总结

这篇文章讲了两个方向来让 Agent "更好用"：

**高级 RAG 技术**——解决"检索不到"和"检索不准"：
- **Multi-hop RAG**：多步迭代检索，每步用上一步的结果指导下一步
- **Self-RAG**：LLM 自主判断是否需要检索、结果是否相关、回答是否被支持
- **Graph RAG**：用知识图谱补充向量检索无法处理的关系推理
- **Hybrid Search**：向量语义检索 + BM25 关键词检索的融合

**Agent 记忆系统**——解决"每次从零开始"：
- **短期记忆**：当前对话上下文，会话结束后清空
- **长期记忆**：用户偏好、项目背景等跨会话信息
- **程序性记忆**：Agent 学到的可复用操作模式
- **存储方案**：SQLite（结构化）+ 向量数据库（语义化）的组合
- **记忆管理**：时间衰减、摘要压缩、冲突消解

---

## 下一步

到这里，我们已经覆盖了 Agent 的核心技术栈：LLM 基础 → Prompt Engineering → 推理模型 → Agent 循环 → 工具调用 → Context Engineering → RAG 与记忆。

**下一阶段是实战篇**——把这些技术组合起来，从零搭建一个完整的 AI Agent 项目。我们会选择一个实际场景（技术文档助手），走完从架构设计到部署上线的全流程。

---

## 推荐资源

- [Self-RAG: Learning to Retrieve, Generate, and Critique through Self-Reflection](https://arxiv.org/abs/2310.11511) — Self-RAG 原论文
- [Microsoft GraphRAG](https://github.com/microsoft/graphrag) — Graph RAG 的官方实现
- [IRCoT: Interleaving Retrieval with Chain-of-Thought](https://arxiv.org/abs/2212.10509) — Multi-hop RAG 的代表性工作
- [LangChain Memory 文档](https://python.langchain.com/docs/modules/memory/) — 记忆模块的完整 API 和示例
- [LlamaIndex Memory 文档](https://docs.llamaindex.ai/en/stable/module_guides/storing/chat_stores/) — 持久化聊天存储
- [BM25 + 向量混合检索实战](https://weaviate.io/blog/hybrid-search-explained) — Weaviate 的 Hybrid Search 详解
