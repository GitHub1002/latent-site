---
title: "09 | Context Engineering——比 Prompt 更重要的「给模型看什么」"
date: 2026-07-07
draft: false
weight: 301
tags: ["Agent", "Context Engineering"]
summary: "Prompt Engineering 关注'怎么说'，Context Engineering 关注'给模型看什么'。这篇文章系统讲解上下文工程的核心策略：窗口管理、信息优先级、动态构建、System Prompt 进阶设计。"
ShowToc: true
---
第二阶段的 4 篇文章，我们学会了让 Agent "自主循环做事"。但在实践中你会发现一个问题：**同一个 Agent，面对不同的上下文信息，表现天差地别。**

给它 5 条精确的相关文档，它能给出高质量回答。给它 50 条杂乱的搜索结果，它可能"迷失"在信息海洋里，给出混乱甚至矛盾的答案。

这就是 **Context Engineering** 要解决的问题：不是"怎么写好指令"（那是 Prompt Engineering），而是 **"怎么构建模型看到的整个信息环境"**。

![Context Engineering 核心策略](/images/stage3-09-context-engineering.svg)

---

## 为什么 Context 比 Prompt 更重要？

先看一个对比实验。同一个问题，三种上下文：

```
问题: "Python 的 asyncio 怎么用在 Web 爬虫中？"
```

**上下文 A**（无额外信息）：

```
System Prompt: 你是一个 Python 专家。
用户问题: Python 的 asyncio 怎么用在 Web 爬虫中？
```

→ 模型给出通用回答，可能包含过时的库推荐。

**上下文 B**（塞入大量无关信息）：

```
System Prompt: 你是一个 Python 专家。
[附带 50 篇随机 Python 文章的摘要]
用户问题: Python 的 asyncio 怎么用在 Web 爬虫中？
```

→ 模型被无关信息干扰，回答不够聚焦，甚至可能引用不相关的内容。

**上下文 C**（精心筛选的上下文）：

```
System Prompt: 你是一个 Python 专家，专注异步编程方向。

[参考文档 1] aiohttp 官方文档: 异步 HTTP 客户端示例...
[参考文档 2] Python 3.12 asyncio 新特性: TaskGroup 用法...
[参考文档 3] 实际案例: 用 asyncio + aiohttp 爬取 1000 个页面的性能对比...

用户问题: Python 的 asyncio 怎么用在 Web 爬虫中？
```

→ 模型给出精准、有深度、引用具体文档的高质量回答。

**结论：给模型"看什么"比"怎么说"更重要。** 上下文是模型生成回答的"原料"——原料不好，厨师（模型）再厉害也做不出好菜。

---

## 上下文窗口的容量是有限资源

第一篇我们讲过，Context Window 是输入和输出共用的。以 128K 窗口为例：

```
128K tokens 总量
├── System Prompt:      ~500 tokens    (0.4%)
├── 历史对话:           ~2,000 tokens  (1.6%)
├── 检索到的文档:       ~120,000 tokens (93.8%)  ← 大头在这里
├── 用户当前问题:       ~200 tokens    (0.2%)
└── 留给模型输出:       ~5,300 tokens  (4.1%)
```

问题很明显：**如果你把 93.8% 的窗口都塞给了检索文档，模型只剩 4% 的空间来思考和回答。** 这对 Agent 尤其严重——Agent 每轮循环的工具调用结果、历史 Thought/Action/Observation 都在消耗这个有限的窗口。

所以 Context Engineering 的第一原则是：**窗口是稀缺资源，每一段信息都要有明确的价值。**

---

## 信息优先级：什么该放，什么不该放？

不是所有信息都应该塞进上下文。你需要一个**优先级框架**：

### 优先级从高到低

| 优先级              | 信息类型            | 示例                   | 占比建议 |
| ------------------- | ------------------- | ---------------------- | -------- |
| **P0 必放**   | 任务定义和角色      | System Prompt          | 1-3%     |
| **P1 高优先** | 直接相关的文档/数据 | RAG 检索的 top-3 结果  | 20-40%   |
| **P2 中优先** | 历史对话上下文      | 最近 3-5 轮对话        | 10-20%   |
| **P3 低优先** | 补充背景信息        | 相关但不直接需要的文档 | 按需     |
| **P4 不放**   | 无关信息            | 检索结果中相关度低的   | 0%       |

### 实际操作：用相关度分数做筛选

```python
# RAG 检索返回了 20 个文档片段，每个都有相关度分数
results = [
    {"text": "...", "score": 0.95},
    {"text": "...", "score": 0.87},
    {"text": "...", "score": 0.82},
    {"text": "...", "score": 0.45},  # 相关度低
    {"text": "...", "score": 0.38},  # 相关度低
    ...
]

# 策略：只取 top-3 且分数 > 0.7 的
selected = [r for r in results[:5] if r["score"] > 0.7]

# 构建上下文
context = "\n\n".join([
    f"[参考文档 {i+1}] (相关度: {r['score']:.2f})\n{r['text']}"
    for i, r in enumerate(selected)
])
```

注意我在每段文档前标注了**相关度分数**——这让模型知道哪些信息更可靠，在回答时可以优先引用高分文档。

---

## 动态上下文构建：不要给所有任务看同样的信息

不同的任务需要不同的上下文。一个好的 Agent 系统应该**动态调整上下文**：

### 场景 1：简单问答

```
System Prompt + 用户问题 → 不需要检索，直接回答
窗口占用：极少
```

### 场景 2：知识查询

```
System Prompt + RAG 检索结果 + 用户问题 → 基于文档回答
窗口占用：中等
```

### 场景 3：复杂推理

```
System Prompt + RAG 检索 + 历史推理链 + CoT 提示 + 用户问题
窗口占用：大量
```

### 动态路由实现

```python
def build_context(user_query, conversation_history):
    """根据任务复杂度动态构建上下文"""

    # 1. 基础层：始终包含
    context = [SYSTEM_PROMPT]

    # 2. 判断是否需要 RAG
    if needs_external_knowledge(user_query):
        docs = rag_search(user_query, top_k=3)
        context.append(format_documents(docs))

    # 3. 判断是否需要历史对话
    if len(conversation_history) > 0:
        # 不是全部历史，只取最近 3 轮
        recent = conversation_history[-6:]  # 3 轮 = 6 条消息
        context.append(format_history(recent))

    # 4. 判断是否需要 CoT 引导
    if is_complex_reasoning(user_query):
        context.append("请按步骤分析这个问题...")

    return "\n\n".join(context)
```

核心思想：**不要一刀切地给所有请求塞同样的上下文。** 简单问题给轻量上下文，复杂问题给丰富上下文。

---

## 历史对话管理：不是越多越好

多轮对话中，历史消息会越来越多，最终撑爆窗口。你需要策略性地管理历史：

### 策略 1：滑动窗口

只保留最近 N 轮对话：

```python
def trim_history_sliding(history, max_turns=5):
    """保留最近 N 轮对话（每轮 = user + assistant 共 2 条）"""
    return history[-(max_turns * 2):]
```

简单粗暴，但会丢失早期重要信息。

### 策略 2：摘要压缩

当历史太长时，让 LLM 把早期对话压缩成摘要：

```python
def compress_history(history, max_tokens=2000):
    """超长历史时压缩早期对话为摘要"""
    if estimate_tokens(history) <= max_tokens:
        return history

    # 分割：早期 + 近期
    split_point = len(history) // 2
    early = history[:split_point]
    recent = history[split_point:]

    # 用 LLM 压缩早期对话
    summary = llm.invoke(
        f"将以下对话历史压缩为一段简洁的摘要（保留关键信息和结论）：\n"
        f"{format_messages(early)}"
    )

    # 返回：摘要 + 近期对话
    return [
        {"role": "system", "content": f"[历史对话摘要] {summary.content}"},
        *recent
    ]
```

### 策略 3：结构化记忆

把历史中的关键信息提取为结构化数据，而非保留原始对话：

```python
# 不是保留原始对话
history_raw = [
    {"role": "user", "content": "我叫张三，在北京做 AI 开发"},
    {"role": "assistant", "content": "你好张三..."},
    {"role": "user", "content": "我们公司在做 RAG 相关产品"},
    ...
]

# 而是提取为结构化记忆
memory_structured = {
    "user_name": "张三",
    "location": "北京",
    "profession": "AI 开发",
    "company_focus": "RAG 产品"
}

# 注入上下文时只需要一小段
context = f"用户信息：{memory_structured['user_name']}，{memory_structured['location']}，{memory_structured['profession']}"
```

三种策略可以组合使用：结构化记忆记关键信息 + 摘要压缩记中间历史 + 滑动窗口留近期对话。

---

## System Prompt 进阶设计

第一阶段的 System Prompt 讲的是基本角色设定。在 Agent 场景下，System Prompt 需要更精细的设计：

### 一个好的 Agent System Prompt 结构

```python
AGENT_SYSTEM_PROMPT = """
## 角色
你是一个专业的技术文档助手，专注于 AI/LLM 领域。

## 能力
你可以搜索互联网、查询知识库、执行 Python 代码进行数据分析。

## 行为规范
1. 回答必须基于事实，引用具体来源
2. 如果不确定，明确说"我不确定"而不是编造
3. 代码示例用 Python，除非用户指定其他语言
4. 使用中文回答，技术术语保留英文

## 工具使用规则
- 遇到实时性问题（价格、新闻、天气），必须先搜索
- 遇到计算问题，使用 calculator 而不是心算
- 不要在不确定时猜测，用工具验证

## 输出格式
- 简洁直接，不要"很高兴为您解答"等废话
- 技术内容用 Markdown 格式化
- 关键结论加粗
"""
```

注意**行为规范**和**工具使用规则**——这些直接影响 Agent 的决策质量。写得越具体，Agent 的行为越可控。

---

## Agent 场景下的上下文工程

Agent 比单轮对话的上下文管理复杂得多，因为每轮循环都在往上下文里添加新信息：

```
初始上下文: System Prompt + 用户任务
第 1 轮: + Thought₁ + Action₁ + Observation₁
第 2 轮: + Thought₂ + Action₂ + Observation₂
第 3 轮: + Thought₃ + Action₃ + Observation₃
...
第 N 轮: 上下文已经很长了
```

### 问题：Agent 循环越多，上下文越臃肿

10 轮循环后，上下文可能已经用了 50K+ tokens，其中大部分是早期的 Thought/Observation——这些信息对当前决策可能已经没用了。

### 解决方案：Agent 上下文压缩

```python
def compress_agent_context(history, current_loop):
    """压缩 Agent 的历史循环信息"""

    if current_loop <= 5:
        # 前 5 轮：保留完整信息
        return history

    # 超过 5 轮：压缩早期循环
    early = history[:5]   # 前 5 轮
    recent = history[5:]  # 最近几轮

    # 把早期循环压缩为摘要
    early_summary = summarize_agent_loops(early)

    return [
        {"role": "system", "content": f"[已完成的操作摘要] {early_summary}"},
        *recent  # 最近几轮保持完整
    ]
```

这和我们之前讲的历史对话压缩策略是一样的思路——只是应用在了 Agent 循环的层面。

---

## 下一步

这篇文章讲了 Context Engineering 的核心策略——从窗口管理、信息优先级、动态构建到历史压缩。你可能注意到了，其中大量场景都依赖一个能力：**从外部知识库中检索出最相关的信息**。

这就是 RAG（Retrieval-Augmented Generation）——下一篇的主题。RAG 是 Context Engineering 最重要的实践手段，也是当前 LLM 应用中最广泛使用的技术之一。

---

## 推荐资源

- [Anthropic: Context Engineering 最佳实践](https://docs.anthropic.com/en/docs/prompt-engineering) — Claude 官方的上下文设计指南
- [OpenAI: Context Window 管理](https://cookbook.openai.com/) — 实用技巧和代码示例
- [LangChain: Memory 模块文档](https://python.langchain.com/docs/modules/memory/) — 历史管理的多种策略实现
