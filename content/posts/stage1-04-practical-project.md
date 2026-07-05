---
title: "实战——用 Prompt Engineering Guide 搭建你的第一个 LLM 应用"
date: 2026-07-05
draft: false
tags: ["LLM", "Prompt Engineering", "实战"]
summary: "AI Agent 学习路线第一阶段第 4 篇（实战篇）。跟着 Prompt Engineering Guide 动手练习，从 API 调用到完整的小项目，把前三篇的理论知识串起来。"
ShowToc: true
---

前三篇文章我们学了大模型的工作原理、Prompt Engineering 的 6 大技巧、以及推理模型的概念。这篇文章是**动手篇**——我们要写代码，把这些理论变成可以运行的程序。

我们选用的学习资源是 [Prompt Engineering Guide](https://github.com/dair-ai/Prompt-Engineering-Guide)，它是目前最全面的 Prompt 工程教程（GitHub 48k+ stars）。但光看不练没用，所以这篇文章会带你做一个完整的小项目：**一个基于 LLM 的技术文章摘要生成器**。

![实战项目架构](/images/stage1-04-practical-project.svg)

---

## 准备工作

### 环境要求

- Python 3.9+
- 一个 LLM API key（OpenAI、DeepSeek、或其他兼容 OpenAI 接口的服务都可以）

### 安装依赖

```bash
pip install openai python-dotenv
```

### 配置 API Key

在项目根目录创建 `.env` 文件：

```bash
OPENAI_API_KEY=sk-your-api-key-here
# 如果用 DeepSeek，改为：
# OPENAI_BASE_URL=https://api.deepseek.com
# OPENAI_API_KEY=sk-your-deepseek-key
```

---

## 第一步：最简 API 调用

先跑通最基本的 LLM 调用，确保环境没问题：

```python
import os
from dotenv import load_dotenv
from openai import OpenAI

load_dotenv()

client = OpenAI(
    api_key=os.getenv("OPENAI_API_KEY"),
    base_url=os.getenv("OPENAI_BASE_URL", "https://api.openai.com/v1")
)

response = client.chat.completions.create(
    model="gpt-4o-mini",  # 或 "deepseek-chat"
    messages=[
        {"role": "user", "content": "用一句话解释什么是 LLM"}
    ]
)

print(response.choices[0].message.content)
```

跑通后你应该能看到模型的回复。这个最简单的例子就是一个 **Zero-shot Prompt**——直接问，没给示例。

---

## 第二步：用 System Prompt 定义角色

现在给模型加一个 System Prompt，让它扮演一个"技术文章编辑"：

```python
system_prompt = """你是一位资深技术编辑，擅长将复杂的技术文章浓缩为简洁的摘要。

你的摘要应该：
1. 第一句话概括文章核心观点
2. 列出 2-3 个关键要点
3. 最后一句话说明适用人群

不要使用"本文介绍了..."这种开头，直接说重点。"""

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": "请为以下文章写摘要：..."}
    ]
)
```

对比一下加了 System Prompt 前后的输出质量——你会发现角色设定对输出的专业度和格式有显著提升。

---

## 第三步：加入 Few-shot 示例

给模型看几个"优秀摘要"的例子，帮它理解你想要什么风格：

```python
messages = [
    {"role": "system", "content": system_prompt},

    # Few-shot 示例
    {"role": "user", "content": "文章：《Attention Is All You Need》论文解读"},
    {"role": "assistant", "content": """Transformer 架构用自注意力机制完全取代了 RNN 的循环结构，在机器翻译任务上取得 SOTA。
关键要点：
- Self-Attention 让每个 token 能关注序列中任意位置
- 多头注意力捕捉不同层次的语言关系
- 并行计算大幅提升训练效率
适合关注 NLP 架构演进的研究者和工程师。"""},

    # 实际请求
    {"role": "user", "content": f"文章：{article_text}"}
]
```

---

## 第四步：用 CoT 提升摘要质量

对于特别长的文章，让模型先分析再总结：

```python
cot_prompt = f"""请分析以下技术文章并生成摘要。

先思考以下问题（不需要输出思考过程）：
1. 文章的核心问题是什么？
2. 提出了什么解决方案？
3. 关键技术细节有哪些？
4. 目标读者是谁？

然后按以下 JSON 格式输出：
{{
    "one_liner": "一句话核心观点",
    "key_points": ["要点1", "要点2", "要点3"],
    "audience": "适用人群",
    "difficulty": "入门/中级/高级"
}}

文章内容：
{article_text}"""
```

这里同时用了 **CoT**（引导思考）和 **Structured Output**（JSON 格式），两个技巧组合使用。

---

## 第五步：完整项目——批量摘要生成器

把前面的步骤组合成一个完整的工具：

```python
import json
from pathlib import Path

class ArticleSummarizer:
    def __init__(self, model="gpt-4o-mini"):
        self.client = OpenAI(
            api_key=os.getenv("OPENAI_API_KEY"),
            base_url=os.getenv("OPENAI_BASE_URL", "https://api.openai.com/v1")
        )
        self.model = model
        self.system_prompt = """你是一位资深技术编辑..."""  # 同上文

    def summarize(self, article_text: str) -> dict:
        """生成单篇文章的结构化摘要"""
        response = self.client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": self.system_prompt},
                {"role": "user", "content": self._build_prompt(article_text)}
            ],
            temperature=0.3,  # 摘要任务要确定性高
            response_format={"type": "json_object"}  # 强制 JSON 输出
        )
        return json.loads(response.choices[0].message.content)

    def batch_summarize(self, articles: list[str]) -> list[dict]:
        """批量生成摘要"""
        results = []
        for i, article in enumerate(articles):
            print(f"处理第 {i+1}/{len(articles)} 篇...")
            result = self.summarize(article)
            result["token_usage"] = self._get_last_usage()
            results.append(result)
        return results

    def _build_prompt(self, article_text: str) -> str:
        return f"""请分析以下技术文章并生成摘要...

文章内容：
{article_text}"""

    def _get_last_usage(self) -> dict:
        # 获取本次调用的 token 使用量
        pass  # 实际实现中从 response.usage 获取

# 使用
summarizer = ArticleSummarizer()

# 读取文章
articles = [
    Path("articles/article1.md").read_text(),
    Path("articles/article2.md").read_text(),
]

# 批量生成
results = summarizer.batch_summarize(articles)

# 保存结果
for article, result in zip(articles, results):
    print(json.dumps(result, ensure_ascii=False, indent=2))
```

---

## 第六步：对比实验——不同技巧的效果

这是最有学习价值的部分。对同一篇文章，分别用不同技巧生成摘要，对比质量差异：

```python
article = "这里放一篇 500 字以上的技术文章..."

# 实验 1: Zero-shot（最简）
print("=== Zero-shot ===")
print(call_llm(f"总结这篇文章：{article}"))

# 实验 2: + System Prompt
print("\n=== + System Prompt ===")
print(call_llm(f"总结这篇文章：{article}", system=system_prompt))

# 实验 3: + Few-shot
print("\n=== + Few-shot ===")
print(call_llm(few_shot_messages + [user_msg(article)]))

# 实验 4: + CoT
print("\n=== + CoT ===")
print(call_llm(cot_prompt.format(article_text=article)))

# 实验 5: 用推理模型 (o1-mini / deepseek-reasoner)
print("\n=== 推理模型 ===")
print(call_llm(f"总结这篇文章：{article}", model="o1-mini"))
```

你会发现：从实验 1 到实验 4，输出质量逐步提升。而推理模型（实验 5）在不需要复杂 Prompt 的情况下就能给出高质量结果——验证了上一篇讲的内容。

---

## 关键收获

通过这个实战项目，你应该体会到以下几点：

**Prompt Engineering 技巧是叠加使用的**。实际项目中不是只用一个技巧，而是 System Prompt + Few-shot + CoT + Structured Output 组合起来，每个技巧解决一个层面的问题。

**Temperature 不是越高越好**。摘要、分类这类任务需要确定性输出，Temperature 设低（0.2～0.5）。创意写作可以设高（0.8～1.2）。

**结构化输出让 LLM 可编程**。当输出是 JSON 格式时，你可以用代码解析、存储、传递——这是后面做 Agent 时工具调用的基础。

**不同模型适合不同任务**。快速摘要用普通模型，复杂推理用推理模型。选对模型比调 Prompt 更重要。

---

## 第一阶段总结

恭喜你完成了第一阶段的全部学习！回顾一下你学到的东西：

| 篇目 | 核心知识 |
|------|---------|
| 第 1 篇 | LLM 是逐 token 预测的概率机器，Transformer 架构、Tokenization、Context Window |
| 第 2 篇 | 6 大 Prompt Engineering 技巧：Zero-shot、Few-shot、CoT、Self-Consistency、System Prompt、Structured Output |
| 第 3 篇 | 推理模型通过 test-time compute 自主推理，和 CoT Prompt 有本质区别 |
| 第 4 篇 | 动手搭建 LLM 应用，组合使用各种技巧，对比不同方案的效果 |

这些知识构成了你理解 Agent 的基础。接下来第二阶段，我们会进入 **Loop Engineering**——让 LLM 不只是回答一个问题，而是**自主循环执行任务**。你会学到 ReAct 范式、Function Calling、以及从 Tool 到 Skill 的封装思维。

准备好了就进入第二阶段吧！

---

## 扩展练习

想继续提升？试试这些练习：

1. **加入 Self-Consistency**：对同一篇文章生成 5 个摘要，投票选出最佳版本
2. **接入 RAG**：先检索相关文章片段，再生成摘要（预习第三阶段内容）
3. **做成 Web 应用**：用 Gradio 或 Streamlit 给摘要器加一个界面
4. **对比不同模型**：用同一组文章测试 GPT-4o-mini、DeepSeek-V3、Qwen 的表现差异
