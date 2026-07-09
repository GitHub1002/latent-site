---
title: '15 | Harness Engineering——"Agents 不难，Harness 难"'
date: 2026-07-08
draft: false
weight: 403
tags: ["Agent", "Harness Engineering", "Multi-Agent"]
summary: "Harness Engineering 是 Agent 工程中最关键也最容易被忽视的层次——设计整个运行环境的约束、反馈循环、工具链编排和质量保障。这篇从概念到实践，讲解如何为 Agent 系统搭建'脚手架'。"
ShowToc: true
---

在之前的文章里，我们已经学了 Prompt Engineering（指令级）、Loop Engineering（循环级）、Context Engineering（信息级）。这些都是"怎么让单个 Agent 或一组 Agent 工作得更好"。但有一个更高层次的问题一直没被讨论：

**谁在管理 Agent 的运行环境？**

打个比方：你招了一群很优秀的员工（Agent），但如果没有好的办公室、流程、考核制度和反馈机制，他们再优秀也发挥不出来。一个自由职业者在家办公可能每天产出 4 小时有效工作，但如果给他配上一个成熟的团队——有明确的需求文档、代码审查流程、CI/CD 管道、性能监控面板——同样这个人可能产出 8 小时的高质量工作。人没变，变的是环境。

**Harness Engineering 就是设计这个"办公环境"的工程。**

"Agents 不难，Harness 难"——这是 2026 年 Agent 工程师圈子里最流行的一句话。说这话的人大多踩过同一个坑：在 demo 阶段 Agent 表现惊艳，一上生产就各种翻车——超时、烧钱、幻觉、死循环。问题不在 Agent 本身，而在包裹 Agent 的那层"脚手架"没搭好。

---

## 什么是 Harness？

Harness 这个词直译是"马具"或"缰绳"——不是马本身，而是套在马上、让马能按你意愿奔跑的那套装备。一匹好马没有缰绳，可能跑到悬崖边上去；一个能力一般的 Agent 配上好的 Harness，反而能稳定地完成任务。

在软件工程里，harness 常指"测试框架"或"运行框架"——围绕核心逻辑的外围支撑系统。比如 test harness 就是包裹着被测代码的断言、mock、fixture 那一整套东西。

把这个概念迁移到 Agent 领域：

**Agent Harness = Agent 运行所需的一切基础设施和约束机制。**

具体拆开来看，它包含五个维度：

- **工具链编排**：哪些工具按什么顺序调用，失败了走哪条分支
- **约束与边界**：超时阈值、预算上限、token 配额、重试策略、权限沙箱
- **反馈循环**：执行结果好不好？评分多少？要不要重来？
- **质量保障**：输出验证、幻觉检测、格式校验、人工审核节点
- **可观测性**：日志、trace、指标面板、告警规则

一个 Agent 系统能不能上生产，90% 取决于 Harness 做得好不好。Agent 本身的"智能"只是引擎，Harness 才是底盘、变速箱、刹车和仪表盘。

![Harness Engineering 四层架构](/images/stage4-15-harness-engineering.svg)

---

## Harness 的 4 层架构

理论说够了，来看一个具体的分层模型。我把 Harness 拆成 4 层：编排层、约束层、反馈层、可观测层。每一层解决不同的问题，层层递进。

### 第 1 层：编排层（Orchestration）

编排层回答的核心问题是：**Agent 的执行流程是什么？遇到分支怎么走？**

最简单的编排是线性的：A → B → C。但现实中大多数任务需要条件分支、并行执行和结果聚合。

举个具体场景：用户问了一个问题，系统需要先去知识库搜索。如果搜索结果足够好（相关性得分 > 0.8），直接生成回答；如果不够好（得分 < 0.5），切换到 Web 搜索；如果在中间，两者结合。

```python
from dataclasses import dataclass
from enum import Enum
from typing import Any

class RouteTarget(Enum):
    KNOWLEDGE_BASE = "knowledge_base"
    WEB_SEARCH = "web_search"
    HYBRID = "hybrid"
    ESCALATE = "escalate_to_human"

@dataclass
class OrchestrationResult:
    route: RouteTarget
    confidence: float
    data: Any

async def orchestrate_query(user_query: str, kb_search_fn, web_search_fn, llm_fn):
    """
    编排层核心逻辑：根据搜索结果质量动态路由。
    """
    # Step 1: 先走知识库搜索
    kb_results = await kb_search_fn(user_query)
    kb_score = max((r.relevance for r in kb_results), default=0.0)

    # Step 2: 条件路由
    if kb_score >= 0.8:
        # 知识库结果足够好，直接用
        answer = await llm_fn(
            prompt=f"根据以下资料回答用户问题：\n"
                   f"资料：{kb_results[:3]}\n"
                   f"问题：{user_query}"
        )
        return OrchestrationResult(
            route=RouteTarget.KNOWLEDGE_BASE,
            confidence=kb_score,
            data=answer
        )

    elif kb_score < 0.5:
        # 知识库不够用，切换到 Web 搜索
        web_results = await web_search_fn(user_query, max_results=5)
        answer = await llm_fn(
            prompt=f"根据以下网络搜索结果回答用户问题：\n"
                   f"搜索结果：{web_results}\n"
                   f"问题：{user_query}"
        )
        return OrchestrationResult(
            route=RouteTarget.WEB_SEARCH,
            confidence=max(r.score for r in web_results),
            data=answer
        )

    else:
        # 两者结合
        web_results = await web_search_fn(user_query, max_results=3)
        combined = kb_results[:2] + web_results
        answer = await llm_fn(
            prompt=f"综合以下内部资料和外部搜索结果回答问题：\n"
                   f"资料：{combined}\n"
                   f"问题：{user_query}"
        )
        return OrchestrationResult(
            route=RouteTarget.HYBRID,
            confidence=kb_score,
            data=answer
        )
```

这段代码展示了一个典型的条件路由：根据知识库搜索的相关性得分（0.8 / 0.5 两个阈值），动态决定走哪条路径。在生产环境中，这些阈值通常不是拍脑袋定的，而是通过分析历史数据得出的——比如分析了 2000 条真实问答后发现，相关性得分在 0.8 以上时用户满意率达到 92%，而 0.5 以下时只有 34%。

编排层还有一类重要逻辑是**并行执行+结果聚合**。比如一个调研任务需要同时搜索市场规模、竞争格局、技术趋势三个方向，然后汇总：

```python
import asyncio

async def parallel_research(topic: str, search_fn, llm_fn):
    """并行执行多个搜索任务，然后聚合结果。"""

    # 定义三个并行子任务
    async def search_aspect(aspect: str) -> dict:
        results = await search_fn(f"{topic} {aspect}")
        summary = await llm_fn(f"总结以下搜索结果的关键发现：\n{results}")
        return {"aspect": aspect, "summary": summary, "sources": len(results)}

    # 并行启动三个搜索（不互相依赖）
    market_task = search_aspect("市场规模 增长趋势")
    competitor_task = search_aspect("主要玩家 竞争格局")
    tech_task = search_aspect("技术趋势 创新方向")

    # 等待全部完成
    market, competitor, tech = await asyncio.gather(
        market_task, competitor_task, tech_task
    )

    # 聚合结果
    final_report = await llm_fn(
        prompt=f"综合以下三个维度的调研结果，撰写一份结构化的行业分析摘要：\n"
               f"1. 市场：{market['summary']}\n"
               f"2. 竞争：{competitor['summary']}\n"
               f"3. 技术：{tech['summary']}"
    )
    return final_report
```

并行执行的好处很明显：如果每个搜索任务耗时 2 秒，串行需要 6 秒，并行只需要 2 秒出头（加上聚合的开销大约 2.5 秒）。对于用户等待体验来说，这是质的提升。

### 第 2 层：约束层（Constraints）

约束层回答的核心问题是：**Agent 能做什么、不能做什么、做多久、花多少钱？**

这一层是最容易被忽略的，但也是上生产后最先出问题的。没有约束的 Agent 就像一个没有预算管控的实习生——可能一个下午就把你整个月的 API 额度用完了。

来看一个真实的"翻车"案例：某团队部署了一个代码审查 Agent，没有设置 token 上限。某天有人提交了一个 5000 行的 PR，Agent 逐行审查，上下文滚到了 180K tokens，单次调用花了 $3.20。一天之内触发了 40 多次这样的审查，当天 API 账单 $128——是他们月度预算的 4 倍。

约束层需要覆盖的维度：

```python
from dataclasses import dataclass, field
from typing import Optional
import time
import asyncio

@dataclass
class AgentConstraints:
    """Agent 运行约束配置。"""

    # 超时控制
    tool_call_timeout_sec: float = 30.0        # 单次工具调用最多 30 秒
    total_task_timeout_sec: float = 300.0      # 整个任务最多 5 分钟

    # 预算控制
    max_cost_usd: float = 0.50                 # 单次任务最多花 $0.50
    max_tokens: int = 50_000                   # 单次任务最多消耗 50K tokens

    # 重试策略
    max_retries: int = 3                       # 最多重试 3 次
    retry_backoff_base_sec: float = 1.0        # 退避基数 1 秒
    # 实际退避时间: 1s → 2s → 4s → 8s（指数增长）

    # 循环控制
    max_loop_iterations: int = 15              # Agent 循环最多跑 15 轮
    max_consecutive_failures: int = 3          # 连续失败 3 次则终止

    # 权限边界
    allowed_tools: list = field(default_factory=list)    # 白名单工具列表
    blocked_commands: list = field(default_factory=lambda: [
        "rm -rf", "DROP TABLE", "sudo", "chmod 777"
    ])
    file_access_scope: str = "read_only"       # read_only | read_write | none


class ConstraintViolation(Exception):
    """约束违反异常。"""
    def __init__(self, constraint_type: str, message: str):
        self.constraint_type = constraint_type
        super().__init__(f"[{constraint_type}] {message}")


class ConstraintGuard:
    """约束守卫——在 Agent 运行的每个关键节点检查约束。"""

    def __init__(self, constraints: AgentConstraints):
        self.constraints = constraints
        self.total_cost = 0.0
        self.total_tokens = 0
        self.loop_count = 0
        self.consecutive_failures = 0
        self.start_time = time.time()

    def check_timeout(self):
        elapsed = time.time() - self.start_time
        if elapsed > self.constraints.total_task_timeout_sec:
            raise ConstraintViolation(
                "TIMEOUT",
                f"任务已运行 {elapsed:.1f}s，超过上限 "
                f"{self.constraints.total_task_timeout_sec}s"
            )

    def check_budget(self, additional_cost: float):
        projected = self.total_cost + additional_cost
        if projected > self.constraints.max_cost_usd:
            raise ConstraintViolation(
                "BUDGET",
                f"预计花费 ${projected:.4f} 将超过预算 "
                f"${self.constraints.max_cost_usd}"
            )
        self.total_cost = projected

    def check_tokens(self, tokens_used: int):
        self.total_tokens += tokens_used
        if self.total_tokens > self.constraints.max_tokens:
            raise ConstraintViolation(
                "TOKEN_LIMIT",
                f"已消耗 {self.total_tokens} tokens，超过上限 "
                f"{self.constraints.max_tokens}"
            )

    def check_loop(self):
        self.loop_count += 1
        if self.loop_count > self.constraints.max_loop_iterations:
            raise ConstraintViolation(
                "LOOP_LIMIT",
                f"循环已达 {self.loop_count} 轮，超过上限 "
                f"{self.constraints.max_loop_iterations}"
            )

    def record_failure(self):
        self.consecutive_failures += 1
        if self.consecutive_failures >= self.constraints.max_consecutive_failures:
            raise ConstraintViolation(
                "CONSECUTIVE_FAILURES",
                f"连续失败 {self.consecutive_failures} 次，终止执行"
            )

    def record_success(self):
        self.consecutive_failures = 0  # 成功则重置连续失败计数

    async def retry_with_backoff(self, fn, *args, **kwargs):
        """带指数退避的重试执行。"""
        last_error = None
        for attempt in range(self.constraints.max_retries):
            try:
                result = await asyncio.wait_for(
                    fn(*args, **kwargs),
                    timeout=self.constraints.tool_call_timeout_sec
                )
                self.record_success()
                return result
            except asyncio.TimeoutError:
                last_error = "工具调用超时 "
                f"({self.constraints.tool_call_timeout_sec}s)"
            except Exception as e:
                last_error = str(e)

            self.record_failure()
            backoff = self.constraints.retry_backoff_base_sec * (2 ** attempt)
            print(f"  重试 {attempt + 1}/{self.constraints.max_retries}，"
                  f"等待 {backoff}s（错误: {last_error}）")
            await asyncio.sleep(backoff)

        raise ConstraintViolation(
            "RETRY_EXHAUSTED",
            f"重试 {self.constraints.max_retries} 次后仍失败: {last_error}"
        )
```

这套约束系统的核心思想是**在每个关键节点都进行检查**——每次循环开始前检查超时和循环次数，每次 LLM 调用后检查预算和 token 消耗，每次工具调用时检查超时和权限。任何一个维度触发了红线，立刻终止并抛出明确的异常信息。

你可能觉得这些约束"太保守了"。没关系，参数是可以调的。关键不是数字大小，而是**你必须设一个数字**。没有约束的 Agent 上线，就像没有油表的汽车上路——你不知道它什么时候会停下来，只知道停下来的时候一定很贵。

### 第 3 层：反馈层（Feedback）

反馈层回答的核心问题是：**Agent 的输出质量好不好？要不要重来？**

这是 Harness 中最有价值但也最难做好的部分。前面两层（编排和约束）是"控制流"，反馈层是"质量控制"。没有反馈的 Agent 系统，就像一条没有质检的生产线——产品出来了，但没人知道合不合格。

反馈机制有三种主要形式：

**机制 A：LLM-as-Judge——让另一个模型当评委**

用另一个 LLM（通常是同级别或更高级别的模型）来评估 Agent 的输出质量。这个方法的好处是全自动、可扩展，缺点是评估本身也有成本（多一次 LLM 调用）。

```python
import json

JUDGE_PROMPT = """你是一个严格的输出质量评估器。请从以下维度评分（1-10）：

1. 准确性（Accuracy）：回答是否事实正确，有无幻觉？
2. 完整性（Completeness）：是否完整回答了用户的问题？
3. 相关性（Relevance）：是否紧扣问题，有无跑题？
4. 格式（Format）：输出格式是否清晰、易读？

请严格以 JSON 格式输出评估结果：
{
    "accuracy": <1-10>,
    "completeness": <1-10>,
    "relevance": <1-10>,
    "format": <1-10>,
    "overall": <加权平均，准确性权重 0.4，完整性 0.3，相关性 0.2，格式 0.1>,
    "issues": ["列出发现的具体问题"],
    "suggestion": "给出生成者的改进建议"
}

---
用户原始问题：{user_query}
Agent 的回答：{agent_response}
"""


async def llm_as_judge(
    user_query: str,
    agent_response: str,
    judge_llm_fn,
    passing_threshold: float = 7.0
) -> dict:
    """
    LLM-as-Judge 评分函数。
    返回评分结果和是否通过判定。
    """
    prompt = JUDGE_PROMPT.format(
        user_query=user_query,
        agent_response=agent_response
    )

    raw_output = await judge_llm_fn(prompt)

    # 解析 JSON 评分
    try:
        # 尝试从输出中提取 JSON（模型有时会包裹在 markdown 代码块里）
        json_str = raw_output
        if "```json" in raw_output:
            json_str = raw_output.split("```json")[1].split("```")[0]
        elif "```" in raw_output:
            json_str = raw_output.split("```")[1].split("```")[0]

        scores = json.loads(json_str.strip())
    except (json.JSONDecodeError, IndexError):
        return {
            "passed": False,
            "scores": None,
            "error": f"评分模型输出格式错误: {raw_output[:200]}"
        }

    overall = scores.get("overall", 0)
    passed = overall >= passing_threshold

    return {
        "passed": passed,
        "scores": scores,
        "overall": overall,
        "threshold": passing_threshold,
        "issues": scores.get("issues", []),
        "suggestion": scores.get("suggestion", "")
    }


async def run_with_feedback(
    user_query: str,
    agent_fn,
    judge_fn,
    max_attempts: int = 3,
    passing_threshold: float = 7.0
):
    """
    带反馈循环的执行：不达标就重试，把评估反馈给 Agent 让它改进。
    """
    for attempt in range(max_attempts):
        # Agent 生成回答
        response = await agent_fn(user_query)

        # Judge 评估
        evaluation = await judge_fn(
            user_query, response, passing_threshold
        )

        if evaluation["passed"]:
            return {
                "response": response,
                "evaluation": evaluation,
                "attempts": attempt + 1,
                "status": "passed"
            }

        # 没通过——把评估反馈注入下一轮的 prompt
        feedback_msg = (
            f"你上一次的回答评分为 {evaluation['overall']}/10，"
            f"未达到 {passing_threshold} 分的及格线。\n"
            f"发现的问题：{evaluation['issues']}\n"
            f"改进建议：{evaluation['suggestion']}\n"
            f"请根据以上反馈重新生成回答。"
        )
        # 将反馈追加到 agent 的上下文
        agent_fn = partial_with_feedback(agent_fn, feedback_msg)

    # 多次尝试仍未通过
    return {
        "response": response,  # 返回最后一次的结果
        "evaluation": evaluation,
        "attempts": max_attempts,
        "status": "failed_after_retries"
    }
```

实际跑一下这个反馈循环，你会发现一个有趣的现象：Agent 在收到"你的回答准确性只有 5 分，遗漏了 XX 信息"这样的反馈后，第二次生成的质量通常能提升 2-3 分。这就是 Reflexion 论文（Shinn et al., 2023）的核心发现——**让 Agent 看到对自己输出的评价，然后据此改进，效果显著。**

代价也很明显：每多一轮反馈循环，就多消耗一次 LLM 调用的 token 和延迟。一次 Judge 评估大约消耗 800-1200 tokens，延迟 500-800ms。如果你的 Agent 需要跑 3 轮才达标，总的额外开销大约是 3000 tokens + 2 秒延迟。对于大多数应用场景来说，这是一个值得的投入。

**机制 B：Human-in-the-Loop（人工审核节点）**

对于高风险操作（比如 Agent 要发邮件、修改数据库、执行金融交易），在关键节点插入人工确认是必要的。这个实现相对简单：

```python
from enum import Enum

class ApprovalStatus(Enum):
    PENDING = "pending"
    APPROVED = "approved"
    REJECTED = "rejected"

class HumanApprovalGate:
    """人工审核节点。"""

    def __init__(self, notify_fn, timeout_sec: int = 300):
        self.notify_fn = notify_fn       # 通知审核者的函数
        self.timeout_sec = timeout_sec   # 等待审核的超时时间（5 分钟）

    async def request_approval(
        self, action_description: str, risk_level: str, context: dict
    ) -> ApprovalStatus:
        """
        发送审核请求并等待结果。
        risk_level: "low" | "medium" | "high" | "critical"
        """
        # 只有 medium 及以上风险才需要人工审核
        if risk_level == "low":
            return ApprovalStatus.APPROVED

        review_request = {
            "action": action_description,
            "risk_level": risk_level,
            "context": context,
            "timestamp": time.time()
        }

        # 通知审核者（比如发送到 Slack/飞书/钉钉）
        approval_future = await self.notify_fn(review_request)

        # 等待审核结果（带超时）
        try:
            result = await asyncio.wait_for(
                approval_future, timeout=self.timeout_sec
            )
            return ApprovalStatus.APPROVED if result else ApprovalStatus.REJECTED
        except asyncio.TimeoutError:
            # 超时未审核 → 默认拒绝（安全优先）
            return ApprovalStatus.REJECTED
```

**机制 C：规则验证——用硬规则守住底线**

LLM-as-Judge 是"软评估"，有主观性。对于一些硬性要求（比如输出必须是合法 JSON、不能包含 PII 信息、长度不能超过限制），用规则验证更靠谱：

```python
import re
import json

class OutputValidator:
    """输出验证器——用规则检查 Agent 输出。"""

    @staticmethod
    def validate_json(output: str) -> tuple[bool, str]:
        try:
            json.loads(output)
            return True, ""
        except json.JSONDecodeError as e:
            return False, f"非法 JSON: {e}"

    @staticmethod
    def validate_no_pii(output: str) -> tuple[bool, str]:
        """检测是否包含个人敏感信息（简化版）。"""
        # 手机号正则（中国大陆）
        phone_pattern = r'1[3-9]\d{9}'
        # 身份证号
        id_pattern = r'\d{17}[\dXx]'
        # 邮箱
        email_pattern = r'[\w.+-]+@[\w-]+\.[\w.]+'

        for pattern, name in [
            (phone_pattern, "手机号"),
            (id_pattern, "身份证号"),
            (email_pattern, "邮箱地址")
        ]:
            if re.search(pattern, output):
                return False, f"输出包含 {name}，违反 PII 保护规则"

        return True, ""

    @staticmethod
    def validate_length(output: str, max_chars: int = 2000) -> tuple[bool, str]:
        if len(output) > max_chars:
            return False, f"输出长度 {len(output)} 超过上限 {max_chars}"
        return True, ""

    @classmethod
    def full_validate(
        cls, output: str, require_json: bool = True, max_chars: int = 2000
    ) -> dict:
        """执行全部验证，返回综合结果。"""
        results = []

        if require_json:
            ok, msg = cls.validate_json(output)
            results.append(("json_format", ok, msg))

        ok, msg = cls.validate_no_pii(output)
        results.append(("pii_check", ok, msg))

        ok, msg = cls.validate_length(output, max_chars)
        results.append(("length_check", ok, msg))

        all_passed = all(ok for _, ok, _ in results)
        failed = [(name, msg) for name, ok, msg in results if not ok]

        return {
            "passed": all_passed,
            "failed_checks": failed,
            "total_checks": len(results)
        }
```

### 第 4 层：可观测层（Observability）

可观测层回答的核心问题是：**Agent 系统在运行什么、运行得怎么样、出了问题怎么排查？**

如果你用过 LangFuse 或 LangSmith，你对 Agent 可观测性应该不陌生。核心理念是把 Agent 的每一步"思考"和"行动"都记录下来，形成一条完整的 trace（追踪链路）。

一条典型的 Agent trace 长这样：

```
Trace ID: abc-123-def
├── [0.0s]  用户输入: "帮我查一下订单 #8842 的物流状态"
├── [0.3s]  Agent Thought: 需要先查询订单系统获取物流单号
├── [0.5s]  Tool Call: order_api.get_order(order_id="#8842")
│           ├── 输入 tokens: 245
│           ├── 输出 tokens: 180
│           ├── 耗时: 320ms
│           ├── 成本: $0.0012
│           └── 结果: {tracking_number: "SF1234567890", status: "shipped"}
├── [1.0s]  Agent Thought: 拿到物流单号了，调用物流查询接口
├── [1.2s]  Tool Call: logistics_api.track(tracking_id="SF1234567890")
│           ├── 输入 tokens: 310
│           ├── 输出 tokens: 220
│           ├── 耗时: 450ms
│           ├── 成本: $0.0015
│           └── 结果: {location: "上海转运中心", eta: "明天 14:00"}
├── [1.9s]  Agent 生成回答: "您的订单 #8842 目前已到达上海转运中心..."
│           ├── 输入 tokens: 890
│           ├── 输出 tokens: 156
│           ├── 耗时: 680ms
│           └── 成本: $0.0045
└── [2.6s]  完成
    ├── 总耗时: 2.6s
    ├── 总 tokens: 2,001
    ├── 总成本: $0.0072
    ├── 工具调用次数: 2
    └── LLM 调用次数: 3
```

来看一个轻量级的 trace 收集器实现：

```python
import time
import uuid
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class TraceSpan:
    """追踪链路中的一个节点。"""
    name: str
    span_type: str           # "llm_call" | "tool_call" | "agent_thought" | "user_input"
    start_time: float
    end_time: Optional[float] = None
    input_tokens: int = 0
    output_tokens: int = 0
    cost_usd: float = 0.0
    input_data: str = ""
    output_data: str = ""
    error: Optional[str] = None
    metadata: dict = field(default_factory=dict)

    @property
    def duration_ms(self) -> float:
        if self.end_time is None:
            return 0.0
        return (self.end_time - self.start_time) * 1000


@dataclass
class Trace:
    """一条完整的 Agent 执行追踪。"""
    trace_id: str = field(default_factory=lambda: str(uuid.uuid4())[:12])
    spans: list = field(default_factory=list)
    start_time: float = field(default_factory=time.time)
    user_input: str = ""
    final_output: str = ""

    def add_span(self, span: TraceSpan):
        self.spans.append(span)

    @property
    def total_duration_ms(self) -> float:
        if not self.spans:
            return 0.0
        last_end = max(s.end_time or s.start_time for s in self.spans)
        return (last_end - self.start_time) * 1000

    @property
    def total_tokens(self) -> int:
        return sum(s.input_tokens + s.output_tokens for s in self.spans)

    @property
    def total_cost(self) -> float:
        return sum(s.cost_usd for s in self.spans)

    @property
    def llm_call_count(self) -> int:
        return sum(1 for s in self.spans if s.span_type == "llm_call")

    @property
    def tool_call_count(self) -> int:
        return sum(1 for s in self.spans if s.span_type == "tool_call")

    def summary(self) -> dict:
        return {
            "trace_id": self.trace_id,
            "total_duration_ms": round(self.total_duration_ms, 1),
            "total_tokens": self.total_tokens,
            "total_cost_usd": round(self.total_cost, 6),
            "llm_calls": self.llm_call_count,
            "tool_calls": self.tool_call_count,
            "span_count": len(self.spans),
            "errors": [s.error for s in self.spans if s.error]
        }


class TraceCollector:
    """追踪收集器——包装 Agent 的每一步操作。"""

    def __init__(self, alert_callback=None, cost_threshold_usd=0.10):
        self.traces: list[Trace] = []
        self.current_trace: Optional[Trace] = None
        self.alert_callback = alert_callback
        self.cost_threshold = cost_threshold_usd

    def start_trace(self, user_input: str) -> Trace:
        self.current_trace = Trace(user_input=user_input)
        return self.current_trace

    def end_trace(self, final_output: str) -> dict:
        if self.current_trace is None:
            return {}

        self.current_trace.final_output = final_output
        summary = self.current_trace.summary()
        self.traces.append(self.current_trace)
        self.current_trace = None

        # 检查是否需要告警
        if summary["total_cost_usd"] > self.cost_threshold and self.alert_callback:
            self.alert_callback({
                "type": "cost_alert",
                "trace_id": summary["trace_id"],
                "cost": summary["total_cost_usd"],
                "threshold": self.cost_threshold
            })

        return summary

    def record_llm_call(self, prompt: str, response: str,
                        input_tokens: int, output_tokens: int, cost: float):
        """记录一次 LLM 调用。"""
        span = TraceSpan(
            name="llm_call",
            span_type="llm_call",
            start_time=time.time() - 0.5,  # 简化处理
            end_time=time.time(),
            input_tokens=input_tokens,
            output_tokens=output_tokens,
            cost_usd=cost,
            input_data=prompt[:500],
            output_data=response[:500]
        )
        if self.current_trace:
            self.current_trace.add_span(span)

    def record_tool_call(self, tool_name: str, input_data: str,
                         output_data: str, duration_ms: float, error=None):
        """记录一次工具调用。"""
        span = TraceSpan(
            name=tool_name,
            span_type="tool_call",
            start_time=time.time() - duration_ms / 1000,
            end_time=time.time(),
            input_data=input_data[:500],
            output_data=output_data[:500],
            error=error,
            metadata={"duration_ms": duration_ms}
        )
        if self.current_trace:
            self.current_trace.add_span(span)

    def daily_report(self) -> dict:
        """生成每日汇总报告。"""
        if not self.traces:
            return {"message": "今日无执行记录"}

        costs = [t.total_cost for t in self.traces]
        durations = [t.total_duration_ms for t in self.traces]
        errors = [s.error for t in self.traces for s in t.spans if s.error]

        return {
            "date": time.strftime("%Y-%m-%d"),
            "total_traces": len(self.traces),
            "total_cost_usd": round(sum(costs), 4),
            "avg_cost_usd": round(sum(costs) / len(costs), 6),
            "max_cost_usd": round(max(costs), 6),
            "avg_duration_ms": round(sum(durations) / len(durations), 1),
            "p95_duration_ms": round(sorted(durations)[int(len(durations)*0.95)], 1)
                if len(durations) > 1 else durations[0],
            "total_errors": len(errors),
            "error_rate": f"{len(errors)/len(self.traces)*100:.1f}%"
        }
```

可观测层在生产环境中的价值主要体现在两个方面：

一是**故障排查**。当用户报告"Agent 回答错了"，你可以直接通过 trace ID 找到那条完整的执行链路，看到 Agent 在哪一步出了问题——是工具返回了错误数据？还是 LLM 在某个环节产生了幻觉？没有 trace，你只能对着一个最终输出猜测。

二是**成本优化**。通过分析每日报告，你可能发现：某个特定类型的查询平均消耗 12K tokens 和 $0.08，但用户满意度只有 60%——这意味着你需要优化这类查询的编排逻辑，或者调整 Agent 的 prompt 来减少无效推理。

---

## 一个完整的 Harness 设计案例：客服 Agent 系统

理论讲完了，来看一个真实的案例。假设你要为一个电商平台搭建客服 Agent 系统，每天处理大约 5000 条用户咨询。下面是完整的 Harness 设计方案。

### 编排设计

```
用户提问
  ↓
意图识别 Agent（分类: FAQ / 订单查询 / 退换货 / 投诉 / 其他）
  ↓ 条件路由
  ├── FAQ → FAQ Agent（知识库检索 + 回答生成）
  ├── 订单查询 → Order Agent（调用订单 API + 物流 API）
  ├── 退换货 → Return Agent（查退换货政策 + 创建工单）
  ├── 投诉 → [Human-in-the-Loop] → 转人工
  └── 其他 → 通用 Agent（尝试回答，信心低则转人工）
```

### 约束配置

```python
customer_service_constraints = AgentConstraints(
    # 超时：客服场景对响应速度要求高
    tool_call_timeout_sec=10.0,          # API 调用 10s 超时
    total_task_timeout_sec=300.0,        # 整个会话 5 分钟超时

    # 预算：每通会话控制在 $0.10 以内
    max_cost_usd=0.10,
    max_tokens=15_000,                   # 15K tokens 足够客服场景

    # 重试：客服场景快速失败比长时间等待好
    max_retries=2,
    retry_backoff_base_sec=0.5,          # 退避: 0.5s → 1s

    # 循环控制
    max_loop_iterations=10,              # 最多 10 轮对话
    max_consecutive_failures=2,          # 连续失败 2 次 → 转人工

    # 权限
    allowed_tools=["order_api", "logistics_api", "knowledge_base",
                    "return_policy_db", "ticket_system"],
    blocked_commands=["DELETE", "UPDATE", "DROP"],  # 只读为主
    file_access_scope="none"
)
```

每天 5000 通会话，每通预算 $0.10，日均 API 成本约 $500。如果优化得好（大部分 FAQ 查询在 2 轮内解决），实际成本可能在 $300 左右。对比人工客服每人每天处理 100 通、月薪 6000 元（约 $830），Agent 系统的成本优势在 40% 以上——前提是你把 Harness 做好了。

### 反馈设计

```python
# 每通会话结束后的自动评估
async def post_session_evaluation(session_trace, user_satisfaction=None):
    """会话结束后的质量评估。"""
    evaluation = {
        "resolved": False,          # 是否解决了用户问题
        "escalated": False,         # 是否转了人工
        "turns": 0,                 # 对话轮次
        "cost": 0.0,                # 实际成本
        "satisfaction": None        # 用户满意度 (1-5)
    }

    # 从 trace 中提取基础指标
    evaluation["turns"] = session_trace.llm_call_count
    evaluation["cost"] = session_trace.total_cost

    # 判断是否转人工（最后一个 span 是 "escalate" 类型）
    if session_trace.spans and session_trace.spans[-1].name == "escalate":
        evaluation["escalated"] = True
        evaluation["resolved"] = False
    else:
        evaluation["resolved"] = True

    # 如果用户给了满意度评分
    if user_satisfaction is not None:
        evaluation["satisfaction"] = user_satisfaction
        if user_satisfaction <= 2:
            # 低满意度 → 自动升级，让人工复盘
            evaluation["escalated"] = True

    return evaluation
```

### 可观测设计

每日运营报告的核心指标：

```
══════ 客服 Agent 日报 - 2026-07-07 ══════
总会话数:       5,231
解决率:         78.3% (4,096 / 5,231)
转人工率:       21.7% (1,135 / 5,231)
平均对话轮次:   3.2 轮
平均响应延迟:   2,340ms (P95: 5,800ms)
日均成本:       $387.50 (平均 $0.074/通)
用户满意度:     4.1 / 5.0
错误率:         1.2% (63 次工具调用失败)
═══════════════════════════════════════════
```

当某个指标异常时（比如解决率突然从 78% 掉到 60%），告警系统自动触发通知。运维人员通过 LangFuse 查看 trace，发现是知识库索引出了问题——最近新增的 200 条 FAQ 没有被正确索引，导致搜索召回率下降。定位问题后，重建索引，指标恢复。整个排查过程 15 分钟——如果没有可观测层，你可能需要一整天。

---

## Harness vs 其他 Engineering 的关系

到这里你应该能感受到，Harness Engineering 和前面学的几种"Engineering"不在同一个层次上。来做一个对比：

| 工程类型 | 关注点 | 层次 | 典型例子 |
|---------|--------|------|---------|
| Prompt Engineering | 单次指令怎么写 | 指令级 | "请用 JSON 格式输出，包含 name 和 value 字段" |
| Context Engineering | 给模型看什么信息 | 信息级 | RAG 检索 Top-5 结果 + 对话历史最近 10 轮 |
| Loop Engineering | 单 Agent 的思考-行动循环 | 循环级 | ReAct 的 Thought → Action → Observation 循环 |
| Harness Engineering | 整体运行环境的设计 | 系统级 | 编排路由 + 预算约束 + 反馈循环 + 追踪告警 |

一个类比来帮你记住这四层关系：

- **Prompt** = 你跟员工说的一句话（"请把这个报告写好"）
- **Context** = 你给员工看的参考资料（行业数据、往期报告）
- **Loop** = 员工自己的工作节奏（先调研、再构思、再写、再改）
- **Harness** = 整个公司的运作体系（部门分工、审批流程、绩效考核、监控面板）

越往上层，杠杆越大。优化一个 Prompt 可能提升 10% 的输出质量；优化 Harness 可能让整个系统的可用率从 70% 提升到 95%。

---

## 下一步

到这篇为止，第四阶段的理论部分已经全部讲完了。回顾一下这个阶段的内容：

- 第 13 篇：多 Agent 架构（层级式 / 辩论式 / 流水线式）
- 第 14 篇：A2A 协议（Agent 之间怎么通信）
- 第 15 篇：Harness Engineering（整个运行环境怎么搭）

这三篇分别解决了"怎么拆""怎么联""怎么管"三个核心问题。理论到这里就够了——接下来该动手了。

下一篇是实战：我们用 **CrewAI** 框架把前面学的东西全部落地。CrewAI 是目前最主流的多 Agent 编排框架之一，它的设计理念和 Harness Engineering 的思维高度吻合——定义 Agent 的角色（编排）、设定任务约束（约束）、配置回调和监控（可观测）。你会发现，有了理论基础之后，上手一个框架的速度会快很多。

---

## 推荐资源

- [LangFuse](https://github.com/langfuse/langfuse) — 开源 LLM 可观测性平台，可以自部署，支持 trace、评估、成本管理。社区活跃，GitHub 30K+ stars。适合中小团队快速搭建 Agent 监控体系。
- [LangSmith](https://smith.langchain.com/) — LangChain 官方的 Agent 追踪平台。和 LangChain/LangGraph 生态集成最好，如果你已经在用 LangChain，这是最自然的选择。
- [Guardrails AI](https://github.com/guardrails-ai/guardrails) — LLM 输出验证框架。定义验证规则（JSON Schema、PII 检测、毒性过滤等），自动校验和修复 Agent 输出。非常适合做 Harness 反馈层的"规则验证"组件。
- [Braintrust](https://www.braintrust.dev/) — 专注于 LLM 评估和可观测性的平台，提供 Evals（评估框架）+ Logging（日志）+ Playground（调试）的一体化方案。
