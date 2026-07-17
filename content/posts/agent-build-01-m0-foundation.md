---
title: "阶段一·搭地基：让 Agent 跑通第一个问答（M0）"
date: 2026-07-16
draft: false
weight: 210
tags: ["Agent", "RAG", "FastAPI", "Qwen", "实战", "系列二"]
summary: "系列二第一篇实战：从空目录搭出 M0 最小闭环——FastAPI + 单轮问答 + 身份注入骨架，并接通外网 Qwen。记录三个真实踩坑：venv 隔离、pip 中断重跑、PowerShell/GitBash 下 curl 传中文 JSON 的引号陷阱。"
ShowToc: true
---
[上篇开篇路线图](/posts/agent-build-00-roadmap/) 里我们定好了整个项目的骨架和需求。这篇开始动真格：**从空目录，把"能问答的 Agent"最小闭环跑起来**。

我把它叫做 **M0（Milestone 0）**。为什么叫"最小闭环"而不是"完整 Agent"？因为这一步只想验证一件事：**整个技术链路到底通不通**——外网能不能调到大模型、HTTP 接口能不能收发文案、用户身份能不能从请求头一路带到底层。RAG、数据库、权限这些"重头戏"一个都还没碰，是故意留白的。

---

## 1. 本篇目标 / 当前进度

M0 要证明三件事：

1. **外网能真正调通大模型**（不是本地 mock）；
2. **能通过一个 HTTP 接口做单轮问答**；
3. **用户身份能从请求头注入到响应**（权限骨架就位）。

做完后，这个 Agent 能干什么？——收到一句话，调 Qwen 生成一句回答，并原样把 `user_id` 带回来。仅此而已。

它**刻意不**做：不记历史（无状态）、不接 RAG、不查数据库。这些在 M1+ 逐步加。**先把最不确定的"链路通不通"验证掉，再谈能力**，是我今晚最大的体会。

---

## 2. 设计决策

动手前，有几个选型是先在脑子里定好的（详细辩论见开篇路线图，这里只给结论和"为什么是它"）：

- **Python + LangChain，而不是 Java + Spring AI**
  AI 生态 Python 优先（新模型/新工具都是 Python 先出），MVP 速度最快，而且我系列一就是用 LangChain 写的 ReAct Agent，肌肉记忆在。权限虽是 Java/Spring Security 的强项，但 M0 先用"依赖注入"把它做成硬约束也够用。
- **LangChain 而不是 CrewAI / AgentScope**
  这是个**单 Agent + 一组工具**的形状（一个大脑决定调哪个工具），CrewAI 那套"多 Agent 角色扮演"用不上；AgentScope 得多 Agent 协作才发挥价值。两者都留作后期备选。
- **`python -m venv`，而不是 `conda create`**
  纯 Python 包项目，venv 轻量、和 `requirements.txt` 工作流无缝。conda 适合要管 CUDA/系统库的训练项目，这里用不上。
- **Qwen via DashScope 的 OpenAI 兼容端点**
  不锁厂商。环境变量用**供应商中立的 `LLM_*`**（不是 `OPENAI_*`），以后换 OpenAI / Anthropic 只改 `.env`，代码一行不动。

---

## 3. 关键实现

M0 一共四个文件，职责分明：

| 文件              | 干什么                                                                                    |
| ----------------- | ----------------------------------------------------------------------------------------- |
| `app/config.py` | 从`.env` 读 `LLM_API_KEY` / `LLM_MODEL` / `LLM_BASE_URL`，进程内单例 `settings` |
| `app/llm.py`    | `get_llm()` 工厂，返回 langchain 的 `ChatOpenAI`，调用前先检查密钥                    |
| `app/auth.py`   | `get_current_user` 从 `X-User-Id` 请求头取身份——**权限红线的第一块基石**      |
| `app/main.py`   | FastAPI 入口：`/health` 健康检查 + `/chat` 异步单轮问答                               |

整个 `/chat` 端点其实就这几行核心逻辑：

```python
class ChatRequest(BaseModel):
    message: str

class ChatResponse(BaseModel):
    reply: str
    user_id: str  # 原样回传，证明身份注入端到端打通

@app.post("/chat", response_model=ChatResponse)
async def chat(
    req: ChatRequest,
    user_id: str = Depends(get_current_user),  # ← 服务端注入，绝不从模型输入取
):
    llm = get_llm()
    response = await llm.ainvoke(req.message)
    return ChatResponse(reply=response.content, user_id=user_id)
```

注意 `user_id: str = Depends(get_current_user)` 这一行：**它从 HTTP 头拿身份，参数里根本没有"让模型决定查谁"的余地**。M0 只是个桩（靠请求头传 `X-User-Id`），但契约从第一天就钉死了——这正是开篇路线图里那条"权限红线"的最小落地。

`llm.py` 的工厂长这样，把"换模型"的成本压到最低：

```python
def get_llm() -> ChatOpenAI:
    if not settings.api_key:
        raise RuntimeError("LLM_API_KEY 未设置。请把 .env.example 复制为 .env 并填入密钥。")
    return ChatOpenAI(
        model=settings.model,        # qwen-plus
        api_key=settings.api_key,
        base_url=settings.base_url,  # DashScope 兼容端点
        temperature=0.3,
    )
```

---

## 4. 踩坑与调试

这部分是今晚**真实踩过**的坑，不是教科书。每一个都卡过我，记下来希望你别再踩。

### 坑 1：venv 创建与激活

PowerShell 下激活命令是 `.\venv\Scripts\activate`，激活后前缀出现 `(venv)` 就对了。但我的终端一开始显示 `(venv) (base)`——venv 和 Anaconda 的 base 环境共存了。虽然 venv 在前、Python 实际走的是 venv，但为了不把包装进 conda base，建议先 `conda deactivate` 收尾。

用 `pip show langchain` 验证了一下，`Location` 指向 `...\enterprise-knowledge-agent\venv\Lib\site-packages`——**证明依赖真装进了隔离环境，没污染全局**。这一步很关键，不然哪天另一个项目要升个包，版本就打架了。

### 坑 2：pip install 中途取消要重跑

装依赖时我手滑按了取消，报 `ERROR: Operation cancelled by user`。**别慌，`pip install` 是幂等的**——直接重跑同一条 `pip install -r requirements.txt`，它会自动跳过已装的、补齐没装完的，不会装重也不会冲突。

### 坑 3（最阴）：PowerShell / Git Bash 下 curl 传中文 JSON 必翻车

这是今晚最磨人的一个。我用 `-d '{"message": "用一句话解释什么是 RAG"}'` 直接传参，FastAPI 返回：

```
{"detail":[{"type":"json_invalid",...}]}   # PowerShell 下
{"detail":"There was an error parsing the body"}  # Git Bash 下
```

看起来像服务端崩了，其实**不是**——FastAPI 返回的是结构化 422 错误，说明请求到了服务器、只是 JSON 坏了。根因是：**在 PowerShell / MINGW64 里，直接把带中文 + 双引号的 JSON 当命令行参数传给 curl，引号被吃掉了**，发给 FastAPI 的根本不是合法 JSON。

**最稳的解法：把 JSON 写成文件，让 curl 读文件**：

```bash
# 用 heredoc 写文件（单引号防展开，UTF-8 存盘）
cat > body.json <<'EOF'
{"message": "用一句话解释什么是 RAG"}
EOF

# curl 读文件发请求（-d @文件）
curl -X POST http://127.0.0.1:8000/chat \
  -H "Content-Type: application/json" \
  -H "X-User-Id: zhangsan" \
  -d @body.json
```

文件法彻底绕开了 shell 引号 / 中文编码的所有坑。PowerShell 下也可以用 `Invoke-RestMethod` 原生命令，但文件法在两种终端都稳。

> 这个坑建议你记住：凡是在命令行里传带中文/引号的 JSON，**先存文件再 `-d @文件`**，能省下大把对着报错发呆的时间。

---

## 5. 验证效果

服务跑起来后（`uvicorn app.main:app --reload`），先确认活着：

```bash
curl http://127.0.0.1:8000/health
# {"status":"ok"}
```

再用文件法发问：

```json
{
  "reply": "RAG（Retrieval-Augmented Generation，检索增强生成）是一种将外部知识检索与大语言模型生成能力相结合的技术：它先从可信知识库中实时检索相关文档片段，再将这些信息作为上下文输入给大语言模型，从而生成更准确、可溯源、且时效性更强的回答。",
  "user_id": "zhangsan"
}
```

**三件事全部验证通过**：外网 Qwen 真实生成了回答（不是 mock）、HTTP 单轮问答通、而且 `user_id: "zhangsan"` 原样回传——身份注入链路端到端打通。M0 最小闭环，完成。

---

## 6. 架构图更新

![M0 最小闭环架构](/images/agent-build-01-m0-architecture.svg)

当前已实现的部分是**上半截**：用户带身份进来 → 鉴权桩注入 `user_id` → `/chat` 调 `ChatOpenAI` 工厂 → 打到 DashScope Qwen → 回复原样带回 `user_id`。

下半截那圈**灰色虚框**就是接下来要填的：M1 让 Agent 读懂私有文件（RAG）、M2 查数据库、M3 把鉴权桩换成真正的 JWT 并守权限。它们现在都还是"待实现"，但骨架已经给它们留好了位置。

---

## 7. 下一篇预告

下一篇是 **阶段二：让 Agent 读懂私有文件（M1）**。

这才是 RAG 真正开始的地方——我们要把公司散落的 PDF / Word 制度文档，经过**解析 → 切分 → Embedding → 向量库（ChromaDB）→ 检索 → 封装成工具**这一整条链路，接到现在的 `/chat` 里。到时 `/chat` 不再是"盲答"，而是"先去知识库里翻一翻再答"。

届时你会看到：LangChain 的 RAG 原语（`DocumentLoader` / `TextSplitter` / `Embeddings` / `VectorStore`）是怎么一条龙拼起来的，以及——**检索到的内容怎么塞进 prompt 给模型**。

如果这篇的踩坑帮到了你，或者你在本机跑 M0 时遇到别的坑，欢迎在评论区告诉我。我们下篇见。
