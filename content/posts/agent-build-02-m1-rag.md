---
title: "系列二 · 02 | 阶段二·让 Agent 读懂私有文件（M1）：RAG 全链路打通"
date: 2026-07-17
draft: false
weight: 620
tags: ["Agent", "RAG", "ChromaDB", "Qwen", "FastAPI", "LangChain", "实战", "系列二"]
summary: "系列二第二篇实战：把公司散落的 PDF/Word 制度文档，经 解析→切分→Embedding(text-embedding-v3)→ChromaDB→检索→封装成工具 全链路接到 /chat。Agent 会自己决定"调不调工具"。记录一个真实踩坑：OpenAIEmbeddings 默认去下载 tiktoken 编码表导致受限网络下 SSL 报错。"
ShowToc: true
---
[上篇 M0·搭地基](/posts/agent-build-01-m0-foundation/) 里，我们让 Agent 能收一句话、调 Qwen 回一句话。但那是个"盲答"的 Agent——它**只看你问了什么，不认识你公司的任何文档**。

这一篇要做的是项目真正的灵魂能力：**让 Agent 读懂你的私有文件**。也就是 RAG（Retrieval-Augmented Generation，检索增强生成）。做完这步，当你问"我们年假怎么规定"，它不再靠训练时的通用记忆瞎编，而是**先去你的知识库里翻出对应制度，再基于那份原文作答、可溯源**。

我把这步叫 **M1（Milestone 1）**。

---

## 1. 本篇目标 / 当前进度

M1 要打通一条 **RAG 全链路**：

```
私有文件(PDF/Word/...) → 解析 → 切分 → Embedding → 向量库 → 检索 → 封装成工具 → 接入 /chat
```

做完后，Agent 能干什么？

- 问**文档里有的内容**（如"年假几天"）：自动检索知识库，基于原文作答；
- 问**通用问题**（如"什么是 RAG"）：不触发检索，直接用模型自身知识回答；
- 问**知识库没有的**：返回"没找到相关内容"，不编造。

它**刻意不**做：不查数据库（M2）、不做权限隔离（M3）、不出流程图（M4）。这些后面加。先把"私有文件 → 能问答"跑通。

---

## 2. 设计决策

动手前先定几件事，理由讲清楚：

- **向量库用 ChromaDB，不上 Qdrant/Milvus**
  MVP 阶段要的是"零运维、本地文件就能跑"。ChromaDB 一个 `persist_directory` 搞定持久化，后面真要上规模再换 Qdrant 也平滑。**先跑通，别在选数据库上内耗。**
- **Embedding 用 DashScope 的 `text-embedding-v3`**
  你选了直连外网 Qwen，那 Embedding 也走同一家最省事（不用本地起 embedding 服务、不占算力）。它和对话模型共用 `LLM_BASE_URL` / `LLM_API_KEY`，换厂商只改 `.env`。
- **解析按扩展名分派，不引重型框架**
  PDF 用 `PyMuPDF`、Word 用 `python-docx`、TXT/MD 直接读。没必要为一两个文件上 `unstructured` 这种大依赖。**够用、可控、好讲原理**——这对写博客尤其重要。
- **工具用 LangChain 的 `@tool`，Agent 循环自己手搓**
  我**没用** `create_react_agent` 之类封装，而是手写了一个"思考→行动→观察"循环（见第 3 节）。原因：RAG 检索是 Function Calling 最典型的落地，手搓一遍你才真正看懂模型是怎么"决定调不调工具"的——这正是系列一阶段 2 讲的东西。
- **文档类问题和通用问题，让模型自己判断**
  不在代码里硬编码"含某关键词才检索"，而是把 `search_knowledge_base` 这个工具交给模型，由它决定调不调。这才是 Agent，不是 if-else。

---

## 3. 关键实现

M1 在 M0 基础上新增/改动如下：

| 文件 | 干什么 |
| --- | --- |
| `app/config.py` | 新增 `embedding_model` / `chroma_persist_dir` / `docs_dir` 三个配置 |
| `app/llm.py` | 新增 `get_embeddings()`，返回 `OpenAIEmbeddings`（和对话共用 base_url/key） |
| `app/rag/ingest.py` | `load_file` 解析、`split_docs` 切分、`ingest_directory` 入库 |
| `app/rag/retrieve.py` | `search()` 封装 Chroma 的 `similarity_search` |
| `app/agent/tools/rag_tool.py` | `search_knowledge_base` 工具（带说明，模型据此决定何时调） |
| `app/agent/orchestrator.py` | **Agent 主循环**：绑工具 → 调模型 → 有 tool_call 就执行 → 回灌 → 再调，直到不再调工具 |
| `app/main.py` | `/chat` 从 `llm.ainvoke` 改为 `run_agent(message, user_id)` |
| `scripts/ingest.py` | 一行命令把 `data/docs/` 下的文件索引进库 |

### 3.1 入库：解析 → 切分 → 向量化

核心是 `ingest.py` 这段：

```python
def load_file(path: str) -> list[Document]:
    ext = os.path.splitext(path)[1].lower()
    if ext == ".pdf":
        import fitz  # PyMuPDF
        doc = fitz.open(path)
        text = "\n".join(page.get_text() for page in doc)
    elif ext in (".docx", ".doc"):
        from docx import Document as DocxDocument
        d = DocxDocument(path)
        text = "\n".join(p.text for p in d.paragraphs if p.text)
    elif ext in (".txt", ".md"):
        with open(path, "r", encoding="utf-8") as f:
            text = f.read()
    return [Document(page_content=text, metadata={"source": os.path.basename(path)})]

def split_docs(docs, chunk_size=500, chunk_overlap=80):
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=chunk_size, chunk_overlap=chunk_overlap,
        separators=["\n\n", "\n", "。", "！", "？", "；", "，", " "],
    )
    return splitter.split_documents(docs)
```

几点说明：

- **切分是 RAG 的命门**。块太大 → 召回时噪声多、占上下文；块太小 → 语义被切碎。我先用 `chunk_size=500`（约 500 中文字）、`chunk_overlap=80`（重叠防截断），这是起步经验值，**不是最优**，后面 M4 讲重排时会再调。
- 中文分隔符特意把句号、逗号加进 `separators`，比默认按英文空格切更贴合中文。
- 每块都带上 `source` 元数据，回答时能溯源到文件名——这正是 RAG 相对"裸 LLM"的最大价值。

### 3.2 工具：让模型知道"有这么个检索能力"

```python
@tool
def search_knowledge_base(query: str) -> str:
    """在公司的私有知识库（制度文档、手册、规范等）中检索与问题相关的片段。
    当用户询问公司制度、流程、产品、内部规定等可能写在文档里的内容时使用。
    Args:
        query: 用于检索的关键词或问题（中文）。
    """
    docs = rag_search(query, k=4)
    if not docs:
        return "知识库中没有找到相关内容。"
    parts = [f"[片段{i} | 来源:{d.metadata.get('source')}]\n{d.page_content}"
             for i, d in enumerate(docs, 1)]
    return "\n\n".join(parts)
```

`@tool` 装饰器会把函数名、参数、docstring 编成一份"工具说明书"喂给模型。**模型能不能用对工具，一半取决于这段 docstring 写得好不好**——这是 Function Calling 最容易被忽视的细节。

### 3.3 编排器：Agent 的核心循环

`orchestrator.py` 就是系列一阶段 2 讲的 ReAct 落地：

```python
async def run_agent(user_message: str, user_id: str) -> str:
    tools = [search_knowledge_base]
    tool_map = {t.name: t for t in tools}
    llm_with_tools = get_llm().bind_tools(tools)
    messages = [SystemMessage(content=SYSTEM_PROMPT), HumanMessage(content=user_message)]

    for _ in range(MAX_ITERATIONS):
        ai_msg = await llm_with_tools.ainvoke(messages)
        messages.append(ai_msg)
        if not ai_msg.tool_calls:          # ← 不调工具了，这就是最终答案
            return ai_msg.content or ""
        for tc in ai_msg.tool_calls:       # ← 调了工具，执行并把结果塞回去
            fn = tool_map[tc["name"]]
            result = fn.invoke(tc["args"])
            messages.append(ToolMessage(content=str(result), tool_call_id=tc["id"]))
    return "抱歉，我在限定步数内没能给出确定答案……"
```

关键点：**`MAX_ITERATIONS` 是防死循环的保险丝**。模型可能反复调工具停不下来，循环到上限就强制收尾、走友好降级。这个"防呆"在 Agent 工程里是必备的。

---

## 4. 踩坑与调试

### 坑 1（重点）：OpenAIEmbeddings 默认去下载 tiktoken 编码表 → SSL 报错

这是 M1 最阴的一个坑，卡了我一下。

一开始 `get_embeddings()` 就这么写：

```python
return OpenAIEmbeddings(model=settings.embedding_model, api_key=..., base_url=...)
```

入库时直接炸：

```
requests.exceptions.SSLError: HTTPSConnectionPool(host='openaipublic.blob.core.windows.net', ...)
  tiktoken.get_encoding("cl100k_base")
```

**根因**：`OpenAIEmbeddings` 默认 `check_embedding_ctx_length=True`，它会用 **tiktoken** 把长文本按 token 切块，而 tiktoken 第一次要下载 OpenAI 的 `cl100k_base` 编码表（存在微软的 blob 上）。你的网络若访问不了那个 host，就 SSL 报错。**而且用 Qwen 的 embedding 却去下载 OpenAI 的 tokenizer，本来就不合理。**

**解法**：关掉这个检查。我们的文本已经过切分（约 500 字），远在 `text-embedding-v3` 的上下文上限内，不需要它再切：

```python
return OpenAIEmbeddings(
    model=settings.embedding_model, api_key=..., base_url=...,
    check_embedding_ctx_length=False,   # ← 关键：别去下载 OpenAI 的 BPE 编码表
)
```

> 这个坑在"非 OpenAI 模型 + 受限网络"下几乎必踩，记牢：`OpenAIEmbeddings` 接 Qwen/DeepSeek 等兼容端点时，记得 `check_embedding_ctx_length=False`。

### 坑 2（小）：Chroma 重复入库会重复片段

`store.add_documents(chunks)` **不带 id 时，重复跑 `ingest.py` 会往库里再塞一份同样的片段**。检索时同一段内容会返回两次。MVP 不影响正确性，但：

- 想重来 → 先删掉 `data/chroma/` 目录再 ingest；
- 要正式去重 → 给每个 chunk 生成稳定 id（如 `hash(源文件+起止位置)`），用 `upsert` 代替 `add`。M4 优化检索时再处理。

### 坑 3（小）：示例文档太短，只切出 1 块

我放的示例 `公司请假制度.md` 全文才 ~700 字，按 500 字切分只得到 **1 个片段**。这不影响验证，但真实的多页 PDF 会切出几十上百块——你会发现 `similarity_search` 的 `k` 值（召回几条）开始显著影响答案质量。这点留到 M4 调。

---

## 5. 验证效果

我在本地（已装好 M1 依赖、用真实 Qwen key）跑了一遍端到端，结果如下。

**① 入库 + 检索自检**

```
[ingest] 完成：1 个文件 -> 1 个片段，存于 ./data/chroma
RETRIEVE_TOP1_SOURCE = 公司请假制度.md
RETRIEVE_TOP1_SNIPPET = # 公司请假与年假制度（试行版） ... 工作满 1 年不满 10 年的，享有每年 5 天
```

**② 文档类问题 → 触发检索，基于原文作答**

> 问：我们公司的年假是怎么规定的？能休几天？
>
> 答：根据公司《请假与年假制度（试行版）》，年假规定如下：
> ✅ 年假天数标准（按累计工作年限）：满 1 年不满 10 年 **5 天/年**；满 10 年不满 20 年 **10 天/年**；满 20 年及以上 **15 天/年**……
> 年假应在当年 12 月 31 日前休完；未休完可结转至次年 3 月 31 日……

答案里的天数、结转规则，逐条对应示例文档原文——**证明它真的"读了文件"，不是凭记忆编的**。

**③ 通用问题 → 不触发工具，直接回答**

> 问：用一句话解释什么是 RAG
>
> 答：RAG（Retrieval-Augmented Generation，检索增强生成）是一种将信息检索与大语言模型生成能力结合的技术……（并自我标注"基于通用技术定义"）

这一条最关键：模型**自己判断出"解释 RAG"不需要查公司文档**，没去调 `search_knowledge_base`，直接用自身知识答了。这正是"Agent 自己决定调不调工具"的体现，而不是我写死的 if-else。

**④ FastAPI 端点**

`curl` 打 `/chat` 返回 `200`，`reply` 同上、`user_id` 原样回传。M0 的接口契约完全兼容，只是背后多了 RAG。

---

## 6. 架构图更新

![M1 RAG 架构](/images/agent-build-02-m1-rag.svg)

上半部分是**离线入库**（你改文档、加文件时跑一次）：文件流经 解析 → 切分 → Embedding → 落进 Chroma。下半部分是**在线问答**：用户提问带 `X-User-Id` 进来，Orchestrator 循环里模型决定要不要调 `search_knowledge_base`，要的话从 Chroma 取 top-k 片段拼进上下文，最后由 Qwen 产出答案。

注意图上那条**橙色虚线**标出的决策点：**调不调工具，是模型自己定的**。这是我们和 M0"盲答"最大的区别，也是 Agent 区别于普通 RAG 管道的地方。

---

## 7. 下一篇预告

下一篇是 **阶段三（上）：让 Agent 查数据库（M2）+ 权限红线（M3）**。

接下来我们要把"问年假"这件事从"查文档"升级成"查真实数据库"——`get_leave_balance(user_id)`，而且 **user_id 由服务端从登录态注入，工具根本不接受"查谁"的参数**。到时候你会看到：为什么"张三不能查李四年假"不能靠 prompt 提醒，而要靠"查询函数不接受外部 user 参数"这条硬约束。

如果你在本机跑 M1 时遇到别的坑，或者想看我对某个模块讲得更细，评论区告诉我。我们下篇见。
