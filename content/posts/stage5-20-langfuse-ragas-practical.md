---
title: "20 | 实战——用 LangFuse + Ragas 构建 Agent 评估与监控体系"
date: 2026-07-09
draft: false
weight: 504
tags: ["Agent", "LangFuse", "Ragas", "实战", "评估"]
summary: "系列收官之作。用 LangFuse 实现 Agent 的全链路追踪和可观测性，用 Ragas 构建自动化评估 pipeline，把前 19 篇学到的评估、安全、可观测性知识全部落地。最后回顾整个学习路线，给出后续进阶方向。"
ShowToc: true
---
从第 1 篇 Transformer 基础到第 19 篇数据飞轮，我们已经把 LLM/Agent 从原理到架构走了一遍。这篇是整个系列的收官实战——把评估（第 17 篇）、安全（第 18 篇）和可观测性（第 19 篇）这三块拼图拼到一起，搭一个真正能跑的生产级监控和评估系统。

工具选择上，我用的是 LangFuse 和 Ragas。

LangFuse 是一个开源的 LLM 可观测性平台，GitHub 上 8k+ stars，支持自部署，专门为 LLM 应用设计——不是把通用的 APM 工具硬套在 LLM 上，而是从 Trace 结构到指标定义都贴合 LLM 的工作方式。选它的核心原因是免费、数据在自己手里、API 设计干净。

Ragas 是目前最被广泛引用的 RAG 评估框架，13k+ stars，提出了一套被学术界和工业界都认可的指标体系。第 17 篇讲的 Faithfulness、Context Precision 那些指标，Ragas 都帮你实现好了，用 `ascore()` 逐条打分就能出分。

## 项目目标

给之前第 10 篇和第 12 篇做的 RAG Agent 加上完整的监控和评估层：

1. **LangFuse 全链路追踪** —— 每次用户提问，从 Query 理解、向量检索、LLM 生成到最终回答，每一步的耗时、token 消耗、输入输出全部可追溯。
2. **Ragas 自动评估** —— 每次回答自动计算 Faithfulness、Answer Relevance、Context Precision、Context Recall 四个分数。
3. **评估数据集 + 回归测试** —— 建一个标准化的测试集，每次改 prompt、换模型、调检索策略后跑一遍，防止"改了 A 好了 B 坏了"。
4. **安全和成本告警** ——  Faithfulness 掉到阈值以下、单次调用成本超标、延迟超标，都触发告警。
5. **数据飞轮闭环** —— 从线上 Trace 里筛出高质量样本，转成 SFT 训练数据，反哺模型。

![LangFuse + Ragas 实战架构](/images/stage5-20-langfuse-ragas-practical.svg)

## Step 1: 部署 LangFuse

两条路：本地 Docker 或 LangFuse Cloud。学习阶段推荐 Docker，数据在自己机器上，随便折腾。

### 本地 Docker 部署

新建一个 `docker-compose.yml`：

```yaml
version: "3.8"
services:
  langfuse:
    image: langfuse/langfuse:2
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/langfuse
      - NEXTAUTH_SECRET=mysecret
      - SALT=mysalt
      - NEXTAUTH_URL=http://localhost:3000
      - TELEMETRY_ENABLED=false
    depends_on:
      - db

  db:
    image: postgres:16
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=langfuse
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

启动：

```bash
docker compose up -d
```

等 30 秒左右，打开 `http://localhost:3000`，注册一个账号。进入项目设置页面，创建两个 API Key：一个 Public Key，一个 Secret Key。记下来，后面代码里要用。

验证一下部署是否成功：

```bash
curl http://localhost:3000/api/public/health
# 返回 {"status": "OK"} 就说明跑起来了
```

> **关于版本**：这里使用的是 Langfuse Server v2（Docker 镜像 `langfuse/langfuse:2`），只需 PostgreSQL 一个依赖，适合学习。Server v3 新增了 ClickHouse、Redis、S3 等组件，更适合生产环境，但部署复杂度也更高。本文的 Python SDK 代码基于 v3（`pip install langfuse>=3`），与 v2 服务端完全兼容，无需担心。

如果你不想管 Docker，LangFuse Cloud 的免费额度也够学习用——每月 50k 次 observation，个人项目绑绑有余。注册地址 `cloud.langfuse.com`，注册后同样拿到 API Key。

## Step 2: 接入 LangFuse 追踪

LangFuse 提供了两种接入方式，适用场景不同。

### 方式 1：LangChain Callback 集成

如果你用的是 LangChain（第 8 篇、第 12 篇的 Agent 都是基于 LangChain），最快的方式是 CallbackHandler，三行代码搞定：

```python
import os
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langfuse import Langfuse, get_client
from langfuse.langchain import CallbackHandler

# 设置 LangFuse 环境变量（也可以在 .env 中配置）
os.environ["LANGFUSE_PUBLIC_KEY"] = "pk-lf-xxxx"
os.environ["LANGFUSE_SECRET_KEY"] = "sk-lf-xxxx"
os.environ["LANGFUSE_HOST"] = "http://localhost:3000"

# 初始化 Langfuse 客户端（全局单例）
langfuse = Langfuse()
client = get_client()

# 创建 LangFuse callback handler
langfuse_handler = CallbackHandler()

# 构建一个简单的 RAG chain
prompt = ChatPromptTemplate.from_template("""
你是一个行业分析助手。根据以下参考资料回答用户问题。
如果参考资料中没有相关信息，请明确说明。

参考资料：
{context}

用户问题：{question}
""")

llm = ChatOpenAI(model="gpt-4o", temperature=0)
chain = prompt | llm | StrOutputParser()

# 调用时传入 callback，session/user 通过 metadata 传递
result = chain.invoke(
    {"context": "量子计算利用量子比特进行并行计算...", "question": "量子计算有哪些应用场景？"},
    config={
        "callbacks": [langfuse_handler],
        "metadata": {
            "langfuse_session_id": "rag-session-001",
            "langfuse_user_id": "user-42",
        }
    }
)
print(result)

# 确保数据上报到 LangFuse
client.flush()
```

跑完之后打开 LangFuse Web UI (`http://localhost:3000`)，进入 Traces 页面，你会看到刚才那次请求的完整记录：

- **Trace 概览**：总耗时（比如 1.83s）、总 token 数（prompt 1247 + completion 356 = 1603）、估算成本（$0.012）
- **Span 列表**：每个 LangChain 组件（PromptTemplate、ChatOpenAI、StrOutputParser）各是一个 Span，点进去能看到具体的输入输出
- **Metadata**：session_id、user_id、tags 都挂在这里

### 方式 2：@observe 装饰器

对于不基于 LangChain 的自定义代码，用 `@observe` 装饰器更灵活：

```python
from langfuse import Langfuse, observe, get_client

# 初始化 Langfuse 客户端（全局单例）
Langfuse(
    public_key="pk-lf-xxxx",
    secret_key="sk-lf-xxxx",
    host="http://localhost:3000"
)
client = get_client()

@observe()
def retrieve_documents(query: str, top_k: int = 3) -> list:
    """模拟向量检索"""
    # 实际项目中这里会调用向量数据库
    documents = [
        {"text": "量子计算利用量子比特的叠加和纠缠特性...", "score": 0.92},
        {"text": "量子退火算法在组合优化问题上...", "score": 0.87},
        {"text": "IBM 在 2024 年发布了 1000+ 量子比特的处理器...", "score": 0.81},
    ]
    return documents[:top_k]

@observe()
def generate_answer(query: str, context: str) -> str:
    """调用 LLM 生成回答"""
    from openai import OpenAI
    client = OpenAI()
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": "根据参考资料回答问题。"},
            {"role": "user", "content": f"资料：{context}\n问题：{query}"}
        ],
        temperature=0
    )
    # 手动记录 token 使用量
    client.update_current_observation(
        usage_details={
            "input_tokens": response.usage.prompt_tokens,
            "output_tokens": response.usage.completion_tokens,
        }
    )
    return response.choices[0].message.content

@observe(name="rag-pipeline", as_type="span")
def rag_pipeline(query: str) -> dict:
    """完整的 RAG 流程"""
    docs = retrieve_documents(query)
    context = "\n".join([d["text"] for d in docs])
    answer = generate_answer(query, context)

    return {
        "query": query,
        "answer": answer,
        "contexts": [d["text"] for d in docs],
        "retrieval_scores": [d["score"] for d in docs],
    }

# 运行
result = rag_pipeline("量子计算在密码学领域有什么影响？")
client.flush()  # 确保数据上报
print(result["answer"][:200])
```

`@observe` 会自动捕获函数的输入输出、执行时间、异常信息。三层嵌套调用（rag_pipeline → retrieve_documents + generate_answer）在 LangFuse UI 里呈现为一棵树，每层的耗时和输入输出一目了然。

两种方式可以混用：LangChain 部分用 Callback，自定义函数用装饰器，最终都汇聚到同一个 Trace 里。

> **版本提示**：Langfuse Python SDK v3 相比 v2 有几个重要变化：`CallbackHandler` 的导入路径从 `langfuse.callback` 改为 `langfuse.langchain`；`@observe` 装饰器和 `langfuse_context` 不再从 `langfuse.decorators` 导入，改为直接从 `langfuse` 包导入 `observe` 和 `get_client`；配置方式从 `langfuse_context.configure()` 改为先创建 `Langfuse()` 单例再通过 `get_client()` 获取客户端；`session_id`/`user_id` 等上下文信息改为通过 LangChain callback 的 `metadata` 参数传入（键名加 `langfuse_` 前缀）。

## Step 3: 用 Ragas 构建评估 Pipeline

有了追踪数据，下一步是自动评估回答质量。

### 构建评估数据集

评估数据集是整个评估体系的基石。你需要准备一组 `(question, ground_truth)` 对，作为"标准答案"来衡量系统表现。

```python
from datasets import Dataset

eval_data = {
    "question": [
        "量子计算的基本原理是什么？",
        "量子计算在密码学领域有什么影响？",
        "量子纠错码为什么重要？",
        "量子计算和经典计算的本质区别是什么？",
        "目前量子计算的主要挑战有哪些？",
        "量子退火算法适用于什么类型的问题？",
        "IBM 和 Google在量子计算上的进展如何？",
        "量子计算对药物研发有什么潜在影响？",
        "量子互联网的概念是什么？",
        "量子计算什么时候能实现商业化？",
    ],
    "ground_truth": [
        "量子计算利用量子力学的叠加和纠缠特性，使量子比特可以同时处于多个状态，实现指数级的并行计算能力。",
        "量子计算的Shor算法可以在多项式时间内分解大整数，对当前基于RSA和ECC的公钥密码体系构成根本性威胁。后量子密码学正在开发抗量子攻击的替代方案。",
        "量子纠错码用于保护脆弱的量子信息免受退相干和噪声的影响。由于量子不可克隆定理，量子纠错比经典纠错更复杂，需要多个物理量子比特编码一个逻辑量子比特。",
        "经典计算使用确定性的0或1比特，量子计算使用可叠加的量子比特。量子计算在特定问题上（如整数分解、搜索、模拟量子系统）具有指数级或多项式级加速。",
        "主要挑战包括：量子退相干时间短、量子门错误率高、需要极低温运行环境、量子比特扩展困难、缺乏有效的量子纠错方案、编程工具不成熟。",
        "量子退火适用于组合优化问题，如旅行商问题、投资组合优化、蛋白质折叠等。它通过量子隧穿效应来寻找全局最优解，在特定问题上比经典模拟退火更高效。",
        "IBM在2024年发布了超过1000量子比特的Condor处理器，并推进模块化量子计算架构。Google在2023年实现了70量子比特的Sycamore处理器，并在量子纠错方面取得重要突破。",
        "量子计算可以精确模拟分子和化学反应的量子行为，加速药物分子的设计和优化。传统计算机模拟大分子时计算量指数增长，量子计算可以避免这一瓶颈。",
        "量子互联网利用量子纠缠和量子隐形传态实现安全的量子信息传输网络。它可以提供理论上不可破解的通信安全，并连接分布式量子计算资源。",
        "业界预计2027-2030年会出现具有实际商业价值的量子计算应用，但通用容错量子计算机可能需要到2035年之后。NISQ（含噪声中等规模量子）设备目前已在特定领域开始试用。",
    ],
}

dataset = Dataset.from_dict(eval_data)
print(f"评估数据集：{len(dataset)} 个测试用例")
```

10 个用例是起步。实际项目中建议 50-100 个，覆盖不同类型的查询（事实查询、对比查询、推理查询、边界情况）。

### 运行 Ragas 评估

现在让我们的 RAG 系统跑一遍这些测试题，然后用 Ragas 打分：

```python
import asyncio
from openai import AsyncOpenAI
from ragas.llms import llm_factory
from ragas.metrics.collections import (
    Faithfulness,
    AnswerRelevancy,
    ContextPrecision,
    ContextRecall,
)

# 用 llm_factory 创建评估用 LLM（替代旧版 LangchainLLMWrapper）
eval_llm = llm_factory("gpt-4o", client=AsyncOpenAI())

# 初始化四个评估指标
faithfulness_metric = Faithfulness(llm=eval_llm)
answer_relevancy_metric = AnswerRelevancy(llm=eval_llm)
context_precision_metric = ContextPrecision(llm=eval_llm)
context_recall_metric = ContextRecall(llm=eval_llm)

# 让 RAG 系统回答所有测试问题
results = []
for item in eval_data["question"]:
    result = rag_pipeline(item)
    results.append(result)

# 逐条评估，收集分数
all_scores = {"faithfulness": [], "answer_relevancy": [], "context_precision": [], "context_recall": []}

async def evaluate_single(question, answer, contexts, reference):
    """评估单条回答的四个指标"""
    f_score = await faithfulness_metric.ascore(response=answer, retrieved_contexts=contexts)
    ar_score = await answer_relevancy_metric.ascore(user_input=question, response=answer)
    cp_score = await context_precision_metric.ascore(user_input=question, retrieved_contexts=contexts, reference=reference)
    cr_score = await context_recall_metric.ascore(retrieved_contexts=contexts, reference=reference)
    return f_score.value, ar_score.value, cp_score.value, cr_score.value

async def evaluate_all():
    for i, question in enumerate(eval_data["question"]):
        f, ar, cp, cr = await evaluate_single(
            question=question,
            answer=results[i]["answer"],
            contexts=results[i]["contexts"],
            reference=eval_data["ground_truth"][i],  # Ragas v0.4 参数名改为 reference
        )
        all_scores["faithfulness"].append(f)
        all_scores["answer_relevancy"].append(ar)
        all_scores["context_precision"].append(cp)
        all_scores["context_recall"].append(cr)

asyncio.run(evaluate_all())

# 计算平均分
avg_scores = {k: sum(v) / len(v) for k, v in all_scores.items()}

# 打印结果
print(f"整体评分（{len(eval_data['question'])} 条平均）：")
print(f"  Faithfulness:       {avg_scores['faithfulness']:.3f}")
print(f"  Answer Relevancy:   {avg_scores['answer_relevancy']:.3f}")
print(f"  Context Precision:  {avg_scores['context_precision']:.3f}")
print(f"  Context Recall:     {avg_scores['context_recall']:.3f}")
```

> **版本提示**：如果你之前用的是 Ragas v0.2/v0.3，注意 v0.4 有几个重大变化：`LangchainLLMWrapper` 已废弃，改用 `llm_factory()` 直接创建 LLM 适配器；指标从 `ragas.metrics` 下的实例变量（如 `faithfulness`）改为从 `ragas.metrics.collections` 导入的类（如 `Faithfulness`），需要实例化后使用；`ground_truth` 参数统一改为 `reference`；`ascore()` 返回的是 `ScoreResult` 对象而不是浮点数，需要通过 `.value` 取值。

跑完会输出类似这样的结果：

```
整体评分：
  Faithfulness:       0.847
  Answer Relevancy:   0.912
  Context Precision:  0.783
  Context Recall:     0.825
```

四个指标的含义（第 17 篇详细讲过，这里快速回顾）：

- **Faithfulness 0.847** —— 84.7% 的回答内容有检索上下文支撑，没有编造信息。这个值低于 0.7 就要警惕幻觉问题。
- **Answer Relevancy 0.912** —— 回答和问题的相关度很高，没有跑题。
- **Context Precision 0.783** —— 检索回来的文档中，78.3% 是真正相关的。说明检索策略还有优化空间。
- **Context Recall 0.825** —— ground_truth 中 82.5% 的关键信息能在检索结果中找到。

### 设定评估基线

把首次评估的分数记录下来作为 baseline：

```python
import json
from datetime import datetime

baseline = {
    "version": "v1.0-baseline",
    "timestamp": datetime.now().isoformat(),
    "config": {
        "model": "gpt-4o",
        "temperature": 0,
        "top_k": 3,
        "chunk_size": 500,
    },
    "scores": {
        "faithfulness": avg_scores["faithfulness"],
        "answer_relevancy": avg_scores["answer_relevancy"],
        "context_precision": avg_scores["context_precision"],
        "context_recall": avg_scores["context_recall"],
    }
}

with open("eval_baselines.jsonl", "a") as f:
    f.write(json.dumps(baseline, ensure_ascii=False) + "\n")

print(f"Baseline 已保存: {baseline['version']}")
```

以后每次改动（换模型、改 prompt、调检索参数），都跑一遍评估，新分数和 baseline 对比。Context Precision 从 0.783 降到 0.65？说明这次改动伤害了检索质量，需要回滚。

## Step 4: 自动化评估 + 告警

手动跑评估适合开发调试。生产环境需要的是：每次请求都自动评估，异常时自动告警。

### 把 Ragas 评估接入 LangFuse

核心思路：每次 RAG 请求结束后，异步计算 Faithfulness 分数，作为 Score 上报到 LangFuse。这样你在 LangFuse UI 里就能看到每次请求的质量分数，还能按分数过滤 Trace。

```python
import asyncio
from openai import AsyncOpenAI
from langfuse import Langfuse, observe, get_client
from ragas.llms import llm_factory
from ragas.metrics.collections import Faithfulness

# 初始化 Langfuse 客户端
Langfuse(
    public_key="pk-lf-xxxx",
    secret_key="sk-lf-xxxx",
    host="http://localhost:3000"
)
client = get_client()

# 用轻量模型做评估，控制成本
eval_llm = llm_factory("gpt-4o-mini", client=AsyncOpenAI())
faithfulness_metric = Faithfulness(llm=eval_llm)

async def compute_faithfulness(answer: str, contexts: list) -> float:
    """异步计算 Faithfulness 分数"""
    result = await faithfulness_metric.ascore(
        response=answer,
        retrieved_contexts=contexts,
    )
    return result.value  # v0.4 返回 ScoreResult 对象，通过 .value 取值

@observe(name="monitored-rag")
async def monitored_rag(query: str) -> dict:
    """带自动评估的 RAG pipeline"""
    # 正常的 RAG 流程
    docs = retrieve_documents(query)
    context = "\n".join([d["text"] for d in docs])
    answer = generate_answer(query, context)

    # 异步计算 Faithfulness
    faithfulness_score = await compute_faithfulness(answer, [d["text"] for d in docs])

    # 上报分数到 LangFuse
    client.score(
        trace_id=client.get_current_trace_id(),
        name="faithfulness",
        value=faithfulness_score,
    )

    return {
        "answer": answer,
        "faithfulness": faithfulness_score,
    }
```

用 `gpt-4o-mini` 做评估而不是 `gpt-4o`，是为了控制成本——Faithfulness 评估本身也要调 LLM，用便宜的小模型足够打分，单次评估成本从 $0.01 降到 $0.001。

### 设置告警规则

LangFuse 本身没有内置告警功能，但我们可以在应用层实现。思路很简单：每次请求后检查指标，超过阈值就触发通知。

```python
import time
from dataclasses import dataclass
from typing import Optional

@dataclass
class AlertRule:
    metric: str
    threshold: float
    direction: str  # "above" 或 "below"
    message: str

ALERT_RULES = [
    AlertRule("faithfulness", 0.7, "below", "Faithfulness 低于 0.7，可能存在幻觉"),
    AlertRule("cost_usd", 0.10, "above", "单次请求成本超过 $0.10"),
    AlertRule("latency_ms", 10000, "above", "请求延迟超过 10 秒"),
]

class AlertManager:
    def __init__(self):
        self.alerts = []

    def check(self, metrics: dict) -> list:
        triggered = []
        for rule in ALERT_RULES:
            value = metrics.get(rule.metric)
            if value is None:
                continue
            if rule.direction == "below" and value < rule.threshold:
                triggered.append(rule)
            elif rule.direction == "above" and value > rule.threshold:
                triggered.append(rule)

        for rule in triggered:
            alert = {
                "timestamp": time.time(),
                "metric": rule.metric,
                "value": metrics[rule.metric],
                "threshold": rule.threshold,
                "message": rule.message,
            }
            self.alerts.append(alert)
            self._send_notification(alert)

        return triggered

    def _send_notification(self, alert: dict):
        """发送告警通知（接入钉钉/飞书/邮件等）"""
        msg = (
            f"[Agent 告警] {alert['message']}\n"
            f"  指标: {alert['metric']} = {alert['value']:.3f}\n"
            f"  阈值: {alert['threshold']}\n"
            f"  时间: {time.strftime('%Y-%m-%d %H:%M:%S')}"
        )
        print(f"⚠ {msg}")
        # 实际项目中接入 webhook:
        # requests.post(WEBHOOK_URL, json={"text": msg})

alert_manager = AlertManager()

# 在请求结束后调用
def after_request(trace_data: dict):
    metrics = {
        "faithfulness": trace_data.get("faithfulness_score", 1.0),
        "cost_usd": trace_data.get("total_cost", 0),
        "latency_ms": trace_data.get("latency", 0),
    }
    triggered = alert_manager.check(metrics)
    if triggered:
        print(f"本次请求触发了 {len(triggered)} 条告警")
```

跑一段时间之后，你会在 LangFuse UI 里积累大量的 Trace 数据。用 Dashboard 功能可以看到 Faithfulness 分数的分布和趋势、平均延迟和成本的变化曲线、按 tag 过滤不同版本的对比数据。

## Step 5: 从评估数据到微调（数据飞轮落地）

这是第 19 篇讲的数据飞轮的最后一步：把线上积累的高质量 Trace 转成 SFT 训练数据。

### 从 LangFuse 导出高分 Trace

```python
from langfuse import Langfuse

langfuse = Langfuse(
    public_key="pk-lf-xxxx",
    secret_key="sk-lf-xxxx",
    host="http://localhost:3000"
)

def export_high_quality_traces(
    min_faithfulness: float = 0.85,
    limit: int = 100
) -> list:
    """从 LangFuse 导出高 Faithfulness 分数的 Trace"""
    traces = langfuse.fetch_traces(
        limit=limit,
        order_by="timestamp",
    )

    high_quality = []
    for trace in traces.data:
        # 获取该 trace 的 faithfulness 分数
        scores = [s for s in trace.scores if s.name == "faithfulness"]
        if not scores:
            continue
        if scores[0].value < min_faithfulness:
            continue

        # 提取输入输出
        observations = langfuse.fetch_observations(trace_id=trace.id)
        # 找到 generate_answer 这一步的输出
        gen_obs = [o for o in observations.data if o.name == "generate_answer"]
        if not gen_obs:
            continue

        high_quality.append({
            "trace_id": trace.id,
            "input": trace.input,
            "output": gen_obs[0].output,
            "faithfulness": scores[0].value,
        })

    print(f"导出 {len(high_quality)} 条高质量 Trace (共 {len(traces.data)} 条)")
    return high_quality

traces = export_high_quality_traces(min_faithfulness=0.85, limit=200)
```

### 转成 SFT 训练格式

```python
import json

def convert_to_sft_format(traces: list) -> list:
    """将 Trace 数据转成 SFT 训练格式（OpenAI 格式）"""
    sft_data = []
    for trace in traces:
        sft_data.append({
            "messages": [
                {
                    "role": "system",
                    "content": "你是一个专业的行业分析助手。根据提供的参考资料，准确、全面地回答用户问题。"
                },
                {
                    "role": "user",
                    "content": trace["input"]
                },
                {
                    "role": "assistant",
                    "content": trace["output"]
                }
            ]
        })
    return sft_data

sft_dataset = convert_to_sft_format(traces)

# 保存为 JSONL
with open("sft_dataset.jsonl", "w", encoding="utf-8") as f:
    for item in sft_dataset:
        f.write(json.dumps(item, ensure_ascii=False) + "\n")

print(f"SFT 数据集: {len(sft_dataset)} 条")
print(f"示例:\n{json.dumps(sft_dataset[0], ensure_ascii=False, indent=2)[:500]}")
```

拿到这个 `sft_dataset.jsonl`，就可以用 Hugging Face TRL 库去微调一个小模型（比如 Qwen2.5-7B），让它在你的特定领域上表现更好。微调不是这篇的重点，简单提一下流程：用 `SFTTrainer` 加载基座模型 + 数据集，训练 2-3 个 epoch，评估后部署替换原来的大模型。成本可以从 GPT-4o 的 $0.01/次降到自部署 7B 模型的 $0.0005/次，20 倍的成本差距。

这就是数据飞轮的完整闭环：线上服务 → 收集 Trace → 评估筛选 → 微调模型 → 部署上线 → 收集更好的 Trace。每转一圈，模型更准、成本更低、数据更多。

## 运行效果展示

用"量子计算行业分析"作为测试输入，跑一遍完整的 monitored RAG pipeline，看看最终效果。

LangFuse 里看到的 Trace 结构：

```
Trace: monitored-rag (总耗时 2.34s, 成本 $0.014)
├── Span: retrieve_documents (耗时 0.18s)
│   ├── Input: "量子计算行业目前的竞争格局如何？"
│   └── Output: 3 篇文档 (relevance scores: 0.94, 0.88, 0.82)
├── Span: generate_answer (耗时 1.96s, tokens: 1380+420)
│   ├── Input: system prompt + context + question
│   └── Output: 420 字的分析回答
└── Score: faithfulness = 0.91
```

Ragas 评估分数（100 次测试平均）：

| 指标              | 分数  | 说明                 |
| ----------------- | ----- | -------------------- |
| Faithfulness      | 0.862 | 86.2% 回答有据可查   |
| Answer Relevancy  | 0.924 | 回答高度切题         |
| Context Precision | 0.791 | 检索精度还有提升空间 |
| Context Recall    | 0.838 | 关键信息覆盖率 83.8% |

跑 100 次测试后的统计报告：

- **平均延迟**：2.1s（其中检索 0.2s，生成 1.7s，评估 0.2s）
- **平均成本**：$0.013/次（其中主 LLM $0.011，评估 LLM $0.002）
- **告警触发率**：4%（4 次 Faithfulness < 0.7，0 次成本超标，0 次延迟超标）
- **Faithfulness 分布**：中位数 0.88，最低 0.45（一条边界 case），最高 1.0

那 4 次低 Faithfulness 的 Trace，在 LangFuse UI 里点开一看，发现都是"量子计算 + 某冷门细分领域"的查询——检索回来的文档不够精准，导致 LLM 开始"自由发挥"。这就给了明确的优化方向：补充这些细分领域的文档，或者对低分查询自动触发二次检索。

## 系列回顾

20 篇文章，5 个阶段，从最底层的 Transformer 注意力机制一路走到生产级的 Agent 监控体系。回顾一下这条路线：

```
Stage 1（文章 01-04）: LLM 基础 + Prompt Engineering
  理解了 Transformer 架构、Prompt 设计原则、推理模型的区别、动手做了第一个项目

Stage 2（文章 05-08）: 单 Agent + Loop Engineering
  掌握了 Agent Loop 核心循环、Function Calling 机制、工具集成（MCP）、用 LangChain 构建完整 Agent

Stage 3（文章 09-12）: Context Engineering + RAG
  学会了上下文工程策略、RAG 基础与进阶、记忆系统、用 LlamaIndex 构建生产级 RAG

Stage 4（文章 13-16）: 多 Agent + Harness Engineering
  探索了多 Agent 协作架构、A2A 协议、Harness 编排、用 CrewAI 落地多 Agent 项目

Stage 5（文章 17-20）: 评估 + 安全 + 数据飞轮
  建立了系统化的评估体系、安全防护策略、可观测性基础设施，并用 LangFuse + Ragas 全部落地
```

如果要用一句话概括这 20 篇的核心收获：**构建 LLM/Agent 系统，最难的不是让它"能跑"，而是让它"可控、可评估、可迭代"。** Prompt 让系统能用，评估让你知道它好不好，安全让它不出事，可观测性让你知道哪里出了问题，数据飞轮让它越来越好。这五层叠加起来，才是一个完整的生产级系统。

回头看这趟旅程，最重要的不是记住了哪个 API 怎么调，而是建立了"动手验证"的习惯。LLM 领域变化极快，今天学的框架半年后可能就过时了，但"搭系统 → 跑评估 → 看数据 → 迭代优化"这个方法论不会过时。每一篇的实战代码，如果你都亲手跑过、调过、踩过坑，这些经验是看再多文章也替代不了的。

## 后续进阶方向

系列虽然收官，但 LLM/Agent 的探索远没有结束。以下是四个值得深入的垂直方向：

**Code Agent** —— 让 AI 自主完成编程任务。关注 SWE-bench 排行榜和 OpenHands 项目。SWE-bench 是目前最权威的代码 Agent 评测集，OpenHands（前 OpenDevin）是最活跃的开源 Code Agent 框架。这个方向的核心挑战是长上下文管理和多步规划。

**Web Agent** —— 让 AI 自主操作浏览器完成复杂任务。关注 Browser Use 和 WebArena。和 Code Agent 不同，Web Agent 需要处理视觉理解（看网页截图）+ 结构化操作（点击、输入、滚动）的组合，是多模态能力的天然应用场景。

**Multimodal Agent** —— 融合视觉、语言、甚至音频的统一 Agent。关注 GPT-4o 的原生多模态能力和开源的 LLaVA、Qwen-VL 系列。当 Agent 能"看懂"图片和视频，应用边界会大幅扩展——从自动 UI 测试到医疗影像分析。

**Reasoning Agent** —— 超越 Chain-of-Thought 的深度推理。关注 MCTS（蒙特卡洛树搜索）在推理中的应用和 Tree of Thoughts 框架。o1/o3 系列模型证明了推理时计算的威力，这个方向可能会重新定义"智能"的上限。

每个方向都可以展开成一个完整的系列，这里只点出起点。选择哪个方向，取决于你的实际场景和个人兴趣。

## 推荐资源

- [LangFuse 文档](https://langfuse.com/docs) — 开源 LLM 可观测性平台，部署和 API 文档写得很好
- [Ragas 文档](https://docs.ragas.io/) — RAG 评估的标准框架，论文和代码都有
- [Hugging Face TRL](https://github.com/huggingface/trl) — 微调工具库，SFT 和 RLHF 都支持
- [OpenHands](https://github.com/All-Hands-AI/OpenHands) — 开源 Code Agent 框架
- [Browser Use](https://github.com/browser-use/browser-use) — 浏览器自动化 Agent
