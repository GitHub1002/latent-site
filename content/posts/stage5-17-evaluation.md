---
title: "17 | 评估方法论——怎么知道 Agent 做得好不好？"
date: 2026-07-09
draft: false
weight: 501
tags: ["Agent", "Evaluation", "Ragas"]
summary: "Agent 开发不是写完 Prompt 就结束了——怎么评估 Agent 的质量才是生产化的关键。这篇讲解评估维度（任务完成率、工具调用准确率、幻觉率、成本效率）、评估方法（人工 vs 自动 vs LLM-as-Judge）、Ragas 框架的使用，以及如何建立持续评估的闭环。"
ShowToc: true
---

第四阶段我们聊了多 Agent 架构和 Harness Engineering，到这儿你已经能搭一个能跑的多 Agent 系统了。但"能跑"和"跑得好"完全是两码事。

就像你写了个排序算法，单元测试都过了——但你不知道它在生产环境里平均耗时是多少、在极端输入下会不会崩、跟竞品比性能排第几。Agent 也一样，demo 跑通只是起点，真正上线之前你得回答一个核心问题：**这个 Agent 到底有多靠谱？**

这篇文章把 Agent 评估这件事拆成三块来讲：怎么定义"好"（评估维度）、怎么测量"好"（评估方法）、怎么持续追踪"好不好"（评估闭环）。

![Agent 评估方法论](/images/stage5-17-evaluation.svg)

---

## Agent 评估的 4 个维度

评估一个 Agent 不像评估一个分类模型——有明确的 accuracy、precision、recall 可以算。Agent 的行为是多步骤的、有状态的、有时还是开放式的，所以评估本身就得从多个角度来看。我把它分成四个维度。

### 1. 任务完成率（Task Completion Rate）

这是最直觉的指标——Agent 到底有没有把用户交代的事办成。

具体怎么操作呢？你需要一组标准测试用例。比如你的 Agent 是帮用户查天气和订餐厅的，那就准备 50 个典型请求：

- "帮我查北京明天的天气"
- "找一家海淀区评分最高的日料店，两个人，周六晚上 7 点"
- "如果明天北京下雨，帮我推荐室内活动"

让 Agent 把这 50 个用例跑一遍，人工判断每个是否完成。假设 42 个完成了，那任务完成率就是 **42/50 = 84%**。

难点在哪？在"什么叫完成"。查天气很简单，回答对了就是对了。但"推荐室内活动"这种开放式任务，怎么算完成？这就涉及到评估标准的设计——你可能需要定义一个 rubric，比如：推荐了至少 2 个活动、每个活动有名称和理由、活动确实在北京、确实是室内的。满足 3 条以上算完成。

对于开放式任务，业界常用的做法是把"完成"拆成多个子维度打分，然后取加权平均。这跟我们后面讲的 LLM-as-Judge 方法直接相关。

### 2. 工具调用准确率（Tool Accuracy）

Agent 跟普通 LLM 最大的区别就是它会调用工具。工具调用的正确性直接决定了任务能不能完成。

这个维度要关注三件事：

- **调对了工具没有？** 用户说"查天气"，Agent 调的是 `get_weather` 还是 `search_web`？
- **参数传对了没有？** 查北京天气，location 参数传的是"北京"还是空字符串？
- **调用顺序对不对？** 如果是"先查天气，如果下雨就推荐室内活动"，Agent 有没有先调天气 API 再决定下一步？

假设你的测试集一共产生了 30 次工具调用，其中 27 次工具选择正确、参数正确、顺序正确，那工具调用准确率就是 **27/30 = 90%**。

常见的错误模式有几种：参数缺失（忘了传日期）、重复调用（同样的 API 调了三次）、幻觉调用（调用了一个根本不存在的工具方法）。这些在评估时都要单独记录和分类，方便后续针对性修复。

### 3. 幻觉率（Hallucination Rate）

幻觉是 LLM 系统在生产环境里最大的信任杀手。

什么是幻觉？Agent 信誓旦旦地说"故宫博物院周一闭馆，门票 60 元"，但实际上门票早就改成免费预约了。回答看起来很自信、很完整，但信息是错的。

在 RAG（检索增强生成）系统里，幻觉通常表现为：回答中的某个声明在检索到的上下文里找不到支撑。这就是 Faithfulness（忠实度）评估——我们会在后文用 Ragas 框架来量化它。

实际操作中，幻觉率的计算方式取决于你的评估粒度。如果你评估的是"回答级别"——100 个回答中有 8 个包含至少一处不真实信息，幻觉率就是 **8%**。如果你评估的是"声明级别"（claim-level）——把每个回答拆解成多个事实声明，逐个验证，那 100 个回答可能包含 350 个声明，其中 12 个是幻觉，声明级幻觉率就是 **12/350 = 3.4%**。

声明级的评估更精细，但成本也更高。一般来说，生产环境里先用回答级评估做粗筛，对高风险场景（医疗、法律、金融）再做声明级评估。

### 4. 成本效率（Cost Efficiency）

很多团队在开发阶段不太关注成本，觉得"先跑通再说"。但上了生产环境之后，每天几千次调用，成本问题会迅速浮出水面。

成本效率有两个维度：**金钱成本**和**时间成本**。

金钱成本主要是 API 调用费用。一个典型的 Agent 任务可能需要 3-5 轮 LLM 调用（规划、工具调用、结果总结），加上工具调用本身的费用。假设平均每次任务消耗 12,000 tokens（输入 8K + 输出 4K），用 GPT-4o 的话大约是 $0.04 一次。看起来不多，但如果每天处理 5,000 个请求，月成本就是 **$0.04 × 5000 × 30 = $6,000**。

时间成本就是用户等待的延迟。一个多步骤 Agent 任务如果串行执行，每次工具调用 1 秒、LLM 调用 2 秒，三步下来就 9 秒了。用户能不能等 9 秒？这取决于场景——聊天机器人最好控制在 3-5 秒内，后台批处理任务可能无所谓。

成本优化的方向很明确：用小模型处理简单任务（路由判断、格式转换），大模型只用在关键环节；缓存高频查询的结果；避免 Agent 陷入不必要的循环（比如反复调用同一个工具验证已经拿到的数据）。

---

## 三种评估方法

知道了评什么，接下来是怎么评。主流的评估方法有三种，各有适用场景。

### 人工评估：最准但最贵

让人类专家逐个看 Agent 的输出，按评分标准打分。这是最"靠谱"的方式——毕竟 Agent 最终是给人用的，人来评价最合理。

问题是成本和可扩展性。假设你有 200 个测试用例，每个用例人工评审需要 3 分钟（看输入、看输出、对照标准打分），那就是 10 小时的工作量。每次改了 Prompt 都得重跑一遍，一个迭代周期里可能跑五六次评估——人力成本直接爆炸。

人工评估适合两种场景：一是建立黄金标准集（golden set），用来校准自动评估方法；二是关键路径的验收，比如上线前的最终检查。日常迭代还是得靠自动化方法。

### 规则评估：最便宜但最粗糙

用程序化的规则来检查输出——正则匹配、关键词搜索、格式校验、字段完整性检查。

举个例子，如果你的 Agent 输出一个 JSON 格式的餐厅推荐，规则评估可以检查：

```python
import json
from typing import Dict, List, Tuple


def rule_based_evaluate(agent_output: str, expected: Dict) -> Dict[str, bool]:
    """基于规则的 Agent 输出评估。

    Args:
        agent_output: Agent 返回的原始文本
        expected: 期望结果的字典

    Returns:
        各规则检查项的通过情况
    """
    results = {}

    # 规则 1: 输出是否为合法 JSON
    try:
        parsed = json.loads(agent_output)
        results["valid_json"] = True
    except json.JSONDecodeError:
        results["valid_json"] = False
        results["has_name"] = False
        results["has_rating"] = False
        results["has_address"] = False
        results["rating_in_range"] = False
        return results

    # 规则 2: 必填字段是否齐全
    required_fields = ["name", "rating", "address"]
    for field in required_fields:
        results[f"has_{field}"] = field in parsed

    # 规则 3: 评分是否在合理范围 (1.0 - 5.0)
    if "rating" in parsed:
        try:
            rating = float(parsed["rating"])
            results["rating_in_range"] = 1.0 <= rating <= 5.0
        except (ValueError, TypeError):
            results["rating_in_range"] = False

    # 规则 4: 餐厅名是否包含预期关键词
    if "name" in parsed and "expected_keywords" in expected:
        name = str(parsed["name"]).lower()
        results["name_matches"] = any(
            kw.lower() in name for kw in expected["expected_keywords"]
        )

    return results


# 使用示例
output = '{"name": "海的寿司", "rating": 4.5, "address": "海淀区中关村大街1号"}'
expected = {"expected_keywords": ["寿司", "日料", "海鲜"]}

scores = rule_based_evaluate(output, expected)
for rule, passed in scores.items():
    status = "PASS" if passed else "FAIL"
    print(f"  {rule}: {status}")
# 输出:
#   valid_json: PASS
#   has_name: PASS
#   has_rating: PASS
#   has_address: PASS
#   rating_in_range: PASS
#   name_matches: PASS
```

规则评估的好处是速度极快、成本为零、结果完全确定。坏处是它只能评估结构化的、可预知的方面——你没法用正则判断一个回答"好不好"。

### LLM-as-Judge：性价比最高

这是目前业界用得最多的方法——用一个 LLM（通常是比被测 Agent 更强的模型）来评估 Agent 的输出质量。

原理很简单：你给 Judge 模型一组评分维度和标准，让它对 Agent 的回答打分。因为 LLM 理解自然语言的能力很强，它可以评估那些规则评估搞不定的维度——比如回答是否自然、逻辑是否通顺、信息是否完整。

下面是一个完整可运行的 LLM-as-Judge 实现：

```python
"""LLM-as-Judge 评估实现。

使用前请确保已安装 openai 库：pip install openai
并设置环境变量 OPENAI_API_KEY。
"""

import json
from openai import OpenAI


# 评分维度的 Prompt 模板
JUDGE_PROMPT = """你是一个专业的 AI 输出质量评估员。请对以下 Agent 回答进行评分。

## 用户问题
{question}

## Agent 的回答
{answer}

## 参考答案（如有）
{reference}

## 评分维度
请从以下 4 个维度评分（每个维度 1-5 分）：

1. **准确性 (accuracy)**: 回答中的事实信息是否正确？
   - 5分: 所有信息完全正确
   - 3分: 大部分正确，有小错误
   - 1分: 包含严重事实错误

2. **完整性 (completeness)**: 回答是否覆盖了用户问题的所有方面？
   - 5分: 全面覆盖，没有遗漏
   - 3分: 覆盖了主要方面，有少量遗漏
   - 1分: 大量遗漏，回答不完整

3. **相关性 (relevance)**: 回答是否切题？有没有答非所问？
   - 5分: 完全切题，每句话都跟问题相关
   - 3分: 基本切题，有少量跑题内容
   - 1分: 大量不相关内容

4. **自然度 (naturalness)**: 回答的语言是否自然流畅？
   - 5分: 像人类专家写的，流畅自然
   - 3分: 能看懂，但有些地方生硬
   - 1分: 语法混乱或机器味很重

## 输出格式
请严格以 JSON 格式输出：
{{
    "accuracy": <1-5>,
    "completeness": <1-5>,
    "relevance": <1-5>,
    "naturalness": <1-5>,
    "overall": <1-5>,
    "reasoning": "<用一两句话解释你的评分理由>"
}}
"""


def llm_judge_evaluate(
    question: str,
    answer: str,
    reference: str = "无",
    model: str = "gpt-4o",
) -> dict:
    """用 LLM-as-Judge 方法评估 Agent 回答质量。

    Args:
        question: 用户的原始问题
        answer: Agent 生成的回答
        reference: 参考答案（可选）
        model: 用作 Judge 的模型名称

    Returns:
        包含各维度评分的字典
    """
    client = OpenAI()

    prompt = JUDGE_PROMPT.format(
        question=question,
        answer=answer,
        reference=reference,
    )

    response = client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": prompt}],
        temperature=0.1,  # 低温度保证评分稳定
        response_format={"type": "json_object"},
    )

    result = json.loads(response.choices[0].message.content)
    return result


def batch_evaluate(test_cases: list[dict], model: str = "gpt-4o") -> dict:
    """批量评估多个测试用例。

    Args:
        test_cases: 测试用例列表，每个包含 question、answer、reference
        model: Judge 模型

    Returns:
        汇总评估结果
    """
    all_scores = []

    for i, case in enumerate(test_cases):
        score = llm_judge_evaluate(
            question=case["question"],
            answer=case["answer"],
            reference=case.get("reference", "无"),
            model=model,
        )
        all_scores.append(score)
        print(f"[{i+1}/{len(test_cases)}] overall={score['overall']}/5 - {score['reasoning'][:50]}")

    # 汇总统计
    n = len(all_scores)
    summary = {
        "total_cases": n,
        "avg_accuracy": sum(s["accuracy"] for s in all_scores) / n,
        "avg_completeness": sum(s["completeness"] for s in all_scores) / n,
        "avg_relevance": sum(s["relevance"] for s in all_scores) / n,
        "avg_naturalness": sum(s["naturalness"] for s in all_scores) / n,
        "avg_overall": sum(s["overall"] for s in all_scores) / n,
    }
    return summary


# 使用示例
if __name__ == "__main__":
    test_cases = [
        {
            "question": "Python 的 GIL 是什么？它对多线程有什么影响？",
            "answer": (
                "GIL（全局解释器锁）是 CPython 中的一把互斥锁，"
                "它确保同一时刻只有一个线程执行 Python 字节码。"
                "这意味着在 CPU 密集型任务中，多线程并不能真正并行执行。"
                "但对于 I/O 密集型任务（如网络请求），GIL 会在等待 I/O 时释放，"
                "所以多线程仍然有效。如果需要真正的并行，可以用 multiprocessing 模块。"
            ),
            "reference": "GIL 是 CPython 的全局解释器锁，限制同一时刻只有一个线程执行字节码。",
        },
        {
            "question": "帮我推荐 3 个适合初学者的 Python 项目练手",
            "answer": (
                "1. 待办事项 CLI 工具：用 argparse 实现命令行增删改查，"
                "学习文件操作和命令行参数解析。\n"
                "2. 天气查询脚本：调用 OpenWeather API，学习 HTTP 请求和 JSON 解析。\n"
                "3. Markdown 转 HTML 转换器：用正则表达式实现基本的 Markdown 语法转换，"
                "学习字符串处理和正则。"
            ),
            "reference": "无",
        },
    ]

    summary = batch_evaluate(test_cases)
    print("\n--- 评估汇总 ---")
    for key, value in summary.items():
        if isinstance(value, float):
            print(f"  {key}: {value:.2f}")
        else:
            print(f"  {key}: {value}")
```

LLM-as-Judge 有几个要注意的点：

- **Judge 偏差**：LLM 打分有一些已知偏差，比如位置偏差（倾向于给排在前面的答案更高分）、长度偏差（倾向于给更长的回答更高分）、自我偏好（倾向于给跟自己生成风格相似的回答更高分）。论文 *"Judging LLM-as-a-judge with MT-Bench and Chatbot Arena"*（2023）对此有详细分析。
- **温度设置**：Judge 模型的温度要设低（0.0-0.2），保证同一个输入多次评分结果一致。
- **交叉验证**：最好用 2-3 个不同的 Judge 模型评分取平均，减少单一模型的偏差影响。

三种方法不是非此即彼的关系。最佳实践是组合使用：规则评估做第一道筛（过滤明显错误），LLM-as-Judge 做批量评估（日常迭代），人工评估做最终验收（上线前检查）。

---

## Ragas 框架：标准化的 RAG 评估

如果你的 Agent 涉及 RAG（检索增强生成），那 Ragas 几乎是绑定使用的评估框架。它在 GitHub 上有 13,000+ stars，专门为 RAG pipeline 设计了一套标准化的评估指标。

### 四个核心指标

Ragas 的四个核心指标各自衡量 RAG 流程中不同环节的质量：

**Faithfulness（忠实度）** 衡量的是"回答是否被检索到的文档支持"。如果 Agent 说"公司年假是 15 天"，但检索到的公司手册里写的是 10 天，那这个回答的 Faithfulness 就低。这个指标直接对应我们前面说的幻觉率——Faithfulness 越低，幻觉越多。

**Answer Relevance（回答相关性）** 衡量的是"回答是否切题"。用户问"怎么去机场"，Agent 回答"机场的建成历史"——信息可能没错，但不切题。

**Context Precision（上下文精确度）** 衡量的是"检索到的文档中有多少是真正相关的"。检索了 10 个文档片段，只有 3 个跟问题相关，那 Precision 就是 0.3。这反映的是检索模块的质量。

**Context Recall（上下文召回率）** 衡量的是"所有相关文档中有多少被检索到了"。如果回答问题需要 5 个关键信息点，检索结果里只覆盖了 3 个，那 Recall 就是 0.6。

每个指标的输出都是 0 到 1 之间的浮点数。理想情况下四个指标都在 0.8 以上，实际项目中 Faithfulness > 0.85、Answer Relevance > 0.8 算合格。

### 实战：用 Ragas 评估一个 RAG 系统

下面是一个完整的示例，演示如何用 Ragas 对 RAG 系统进行端到端评估：

```python
"""使用 Ragas 框架评估 RAG 系统。

安装依赖：pip install ragas datasets
"""

from datasets import Dataset
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevance,
    context_precision,
    context_recall,
)


def build_eval_dataset() -> Dataset:
    """构建评估数据集。

    实际项目中，这些数据来自你的 RAG 系统运行日志。
    """
    data = {
        "question": [
            "公司的远程办公政策是什么？",
            "如何申请出差报销？",
            "新产品 X 的核心功能有哪些？",
        ],
        "answer": [
            # Agent 的实际回答
            "公司允许每周远程办公 2 天，需要直属领导审批。",
            "出差报销需要填写报销单，附上发票，提交给财务部，一般 5 个工作日到账。",
            "产品 X 支持实时协作、自动备份、以及 AI 辅助写作功能。",
        ],
        "contexts": [
            # RAG 检索到的上下文
            [
                "公司远程办公政策（2025 版）：员工每周可申请远程办公 2 天，"
                "需提前一天向直属领导提交申请并获得批准。",
                "公司考勤制度：迟到 30 分钟以上记为旷工半天。",  # 不相关的上下文
            ],
            [
                "差旅报销流程：员工出差结束后 7 个工作日内提交报销申请，"
                "需填写《差旅报销单》并附上所有原始发票。"
                "财务部审核通过后，款项将在 5 个工作日内打入工资卡。",
            ],
            [
                "产品 X 功能列表：1. 实时多人协作编辑；2. 自动云备份（每 5 分钟）；"
                "3. AI 辅助写作（支持摘要生成、续写、翻译）。",
                "产品 X 定价：基础版免费，专业版 99 元/月。",  # 部分相关
            ],
        ],
        "ground_truth": [
            # 标准答案（人工标注）
            "每周可远程办公 2 天，需直属领导提前审批。",
            "出差结束后 7 个工作日内提交报销单和发票，财务 5 个工作日处理。",
            "核心功能包括实时协作、自动备份、AI 辅助写作。",
        ],
    }
    return Dataset.from_dict(data)


def run_ragas_evaluation():
    """运行 Ragas 评估并输出结果。"""
    dataset = build_eval_dataset()

    # 执行评估
    results = evaluate(
        dataset=dataset,
        metrics=[
            faithfulness,
            answer_relevance,
            context_precision,
            context_recall,
        ],
    )

    # 打印总体得分
    print("=== Ragas 评估结果 ===\n")
    print(f"  Faithfulness:        {results['faithfulness']:.3f}")
    print(f"  Answer Relevance:    {results['answer_relevance']:.3f}")
    print(f"  Context Precision:   {results['context_precision']:.3f}")
    print(f"  Context Recall:      {results['context_recall']:.3f}")

    # 逐条查看详细结果
    print("\n=== 逐条明细 ===\n")
    df = results.to_pandas()
    for i, row in df.iterrows():
        print(f"Q{i+1}: {row['question'][:30]}...")
        print(f"  Faithfulness:      {row['faithfulness']:.2f}")
        print(f"  Answer Relevance:  {row['answer_relevance']:.2f}")
        print(f"  Context Precision: {row['context_precision']:.2f}")
        print(f"  Context Recall:    {row['context_recall']:.2f}")
        print()

    return results


if __name__ == "__main__":
    run_ragas_evaluation()
```

跑出来的结果可能像这样：

| 指标 | 得分 | 含义 |
|------|------|------|
| Faithfulness | 0.92 | 回答中 92% 的声明被上下文支持，幻觉风险低 |
| Answer Relevance | 0.85 | 回答整体切题，少量冗余信息 |
| Context Precision | 0.75 | 检索结果中 75% 是相关的，有优化空间 |
| Context Recall | 0.88 | 关键信息覆盖较全面 |

拿到这些分数后，你就能精准定位问题：如果 Faithfulness 低，说明 Agent 在"编造"信息，要收紧 Prompt 或加强 grounding；如果 Context Precision 低，说明检索模块在召回噪音，要优化 embedding 模型或调整 top-k 参数。

---

## 建立评估闭环

评估不是一次性的事情。你不可能在项目开始时跑一次评估，然后就再也不看了——Agent 的行为会随着 Prompt 修改、工具变更、模型升级而变化。你需要的是一个持续运转的评估闭环。

这个闭环有四个步骤，跟软件工程的 CI/CD 非常像：

**第一步：构建测试集。** 初始阶段准备 50-100 个测试用例，覆盖主要场景和边界情况。用例要分三类：基础用例（happy path，占 60%）、边界用例（模糊指令、多步任务，占 30%）、对抗用例（故意误导、注入攻击，占 10%）。

**第二步：自动化跑评估。** 每次改了 Prompt、换了工具、升级了模型，都自动跑一遍评估。可以集成到 CI pipeline 里：

```python
"""评估流水线集成示例——在 CI 中自动运行评估。

这个脚本可以在 GitHub Actions / GitLab CI 中运行，
将评估结果与基线对比，决定是否通过。
"""

import json
import sys
from pathlib import Path


def load_baseline() -> dict:
    """加载基线评估分数。"""
    baseline_path = Path("eval_baseline.json")
    if not baseline_path.exists():
        print("WARNING: 未找到基线文件，跳过对比")
        return {}
    return json.loads(baseline_path.read_text())


def run_evaluation_pipeline(
    test_cases_path: str = "eval_test_cases.json",
    baseline_path: str = "eval_baseline.json",
    threshold: float = 0.05,
) -> bool:
    """运行评估流水线并与基线对比。

    Args:
        test_cases_path: 测试用例文件路径
        baseline_path: 基线分数文件路径
        threshold: 允许的最大分数下降幅度（如 0.05 表示 5%）

    Returns:
        True 表示评估通过，False 表示未通过
    """
    # 这里调用你实际的评估函数
    # current_scores = run_your_agent_evaluation(test_cases_path)
    # 示例分数：
    current_scores = {
        "task_completion": 0.84,
        "tool_accuracy": 0.90,
        "faithfulness": 0.92,
        "avg_overall_llm_judge": 4.2,
    }

    baseline = load_baseline()

    if not baseline:
        # 首次运行，保存为基线
        Path(baseline_path).write_text(json.dumps(current_scores, indent=2))
        print("首次运行，已保存基线分数")
        return True

    # 对比基线
    passed = True
    print("\n=== 评估对比 ===")
    for metric, current in current_scores.items():
        base = baseline.get(metric, 0)
        diff = current - base
        diff_pct = (diff / base * 100) if base else 0
        status = "PASS" if diff >= -threshold else "FAIL"

        if status == "FAIL":
            passed = False

        print(
            f"  {metric}: {current:.3f} (baseline: {base:.3f}, "
            f"diff: {diff_pct:+.1f}%) [{status}]"
        )

    # 更新基线（如果通过）
    if passed:
        Path(baseline_path).write_text(json.dumps(current_scores, indent=2))
        print("\n评估通过，已更新基线分数")
    else:
        print(f"\n评估未通过！分数下降超过 {threshold*100:.0f}%，请检查改动")

    return passed


if __name__ == "__main__":
    success = run_evaluation_pipeline()
    sys.exit(0 if success else 1)
```

**第三步：对比前后分数。** 每次评估完，跟上次（或基线）的分数做对比。如果某个指标下降了超过 5%，就应该触发告警，阻止这次改动合入主分支。这跟代码的单元测试是一个道理——测试不过就不让 merge。

**第四步：定期补充新用例。** 真实用户的使用方式总会超出你的想象。定期从线上日志里采样一些真实的用户请求，标注后加入测试集。我一般每两周补充 10-20 个新用例，保持测试集跟实际使用场景同步演化。

这四步形成一个闭环：构建 → 评估 → 对比 → 补充 → 再构建。每一轮迭代，你对 Agent 质量的信心就多一分。

---

## 小结

Agent 评估的核心思路可以浓缩成三句话：

**多维度衡量**——不要只看一个指标。任务完成率告诉你"能不能用"，工具准确率告诉你"稳不稳"，幻觉率告诉你"可不可信"，成本效率告诉你"值不值"。

**多方法交叉**——规则评估做快速筛查，LLM-as-Judge 做批量评估，人工评估做最终验收。单一方法都有盲区，组合起来才全面。

**持续闭环**——评估不是一次性任务，而是一个跟 CI/CD 类似的持续流程。每次改动都跑评估，用数据驱动决策，而不是凭感觉。

---

## 下一步

评估告诉我们 Agent 现在"好不好"——但知道现状还不够，我们还得防止 Agent "变坏"。一个评估分数 90% 的 Agent，如果被恶意用户通过 Prompt 注入攻击绕过安全限制，那 90% 就毫无意义。

下一篇我们进入 Agent 安全防护——怎么让你的 Agent 在面对恶意输入、越权操作、数据泄露等风险时依然稳如老狗。

---

## 推荐资源

- [Ragas 文档](https://docs.ragas.io/) — RAG 评估的标准框架，API 清晰，上手快
- [DeepEval](https://github.com/confident-ai/deepeval) — 另一个 LLM 评估框架，支持更多指标类型
- [LLM Judge 论文](https://arxiv.org/abs/2306.05685) — "Judging LLM-as-a-judge with MT-Bench and Chatbot Arena"，理解 LLM 评分偏差的必读文献
