---
title: "19 | 可观测性与数据飞轮——从'能跑'到'越跑越好'"
date: 2026-07-09
draft: false
weight: 503
tags: ["Agent", "Observability", "数据飞轮", "Fine-tuning"]
summary: "Agent 系统上线后，你需要持续追踪每一步推理和工具调用。这篇讲解可观测性的三个层次（日志/指标/追踪），LangFuse 的使用方式，以及如何从运行日志中构建数据飞轮——筛选高质量轨迹、生成训练数据、微调专用模型，实现'越用越好'的正循环。"
ShowToc: true
---

前几篇讲了评估和安全。评估是"定期体检"——每隔一段时间跑一遍测试集，看看系统有没有退化；安全是"装防盗门"——拦住恶意输入和危险操作。但这两样东西都有一个共同的盲区：**它们不关心 Agent 日常运行的状态**。

你需要一个"24 小时监控摄像头"，让 Agent 每一次推理、每一次工具调用都有据可查。出了问题能秒级定位，没出问题的时候能持续积累数据。

这就是**可观测性（Observability）**。

而且可观测性还有一个更深层的价值——Agent 的运行日志本身就是最宝贵的训练数据。把这些数据用好，就能构建**数据飞轮**：让 Agent 越用越好，越跑越便宜。

![可观测性与数据飞轮全景图](/images/stage5-19-observability-data-flywheel.svg)

## 可观测性的 3 个层次

可观测性不是"加几行 print"那么简单。业界把它分成三个递进的层次：日志、指标、追踪。每往上一层，你能看到的信息更结构化、更全局。

### 第一层：日志（Logging）

日志是最基础的——把 Agent 的每一步操作记录下来。注意，这里说的日志不是 `print("debug: xxx")`，而是**结构化的、带时间戳的、可搜索的记录**。

一个 Agent 的日志至少要包含这些内容：

- 用户的原始输入
- 发给 LLM 的完整 prompt 和返回的 response
- 每次工具调用的参数和返回结果
- 最终给用户的输出
- 每一步的耗时和 token 消耗

下面是一个实用的结构化日志系统，直接可以用：

```python
import json
import time
from datetime import datetime
from pathlib import Path
from dataclasses import dataclass, field, asdict
from typing import Any

@dataclass
class LogEntry:
    """一条结构化日志"""
    timestamp: str
    session_id: str
    step_type: str          # "llm_call" | "tool_call" | "user_input" | "final_output"
    content: dict[str, Any]
    duration_ms: float = 0
    token_usage: dict = field(default_factory=dict)

class AgentLogger:
    """Agent 结构化日志器"""

    def __init__(self, log_dir: str = "./agent_logs"):
        self.log_dir = Path(log_dir)
        self.log_dir.mkdir(exist_ok=True)
        self.entries: list[LogEntry] = []

    def log(self, session_id: str, step_type: str,
            content: dict, duration_ms: float = 0,
            token_usage: dict | None = None):
        entry = LogEntry(
            timestamp=datetime.utcnow().isoformat() + "Z",
            session_id=session_id,
            step_type=step_type,
            content=content,
            duration_ms=duration_ms,
            token_usage=token_usage or {},
        )
        self.entries.append(entry)

        # 按天写入文件，每行一条 JSON（方便后续用 grep/jq 搜索）
        log_file = self.log_dir / f"{datetime.utcnow():%Y-%m-%d}.jsonl"
        with open(log_file, "a", encoding="utf-8") as f:
            f.write(json.dumps(asdict(entry), ensure_ascii=False) + "\n")

    def query(self, session_id: str | None = None,
              step_type: str | None = None) -> list[LogEntry]:
        """按 session_id 或 step_type 过滤日志"""
        results = self.entries
        if session_id:
            results = [e for e in results if e.session_id == session_id]
        if step_type:
            results = [e for e in results if e.step_type == step_type]
        return results


# ── 使用示例 ──────────────────────────────────────
logger = AgentLogger()

# 记录用户输入
logger.log("sess-001", "user_input", {"query": "帮我查一下北京的天气"})

# 记录 LLM 调用
t0 = time.time()
# ... 实际调用 LLM ...
elapsed = (time.time() - t0) * 1000
logger.log("sess-001", "llm_call", {
    "model": "gpt-4o",
    "prompt_summary": "天气查询决策",
    "response": "需要调用 weather_api 工具",
}, duration_ms=elapsed, token_usage={"prompt": 180, "completion": 45})

# 记录工具调用
logger.log("sess-001", "tool_call", {
    "tool": "weather_api",
    "params": {"city": "Beijing"},
    "result": {"temp": "28C", "condition": "晴"},
})
```

每天一个 `.jsonl` 文件，每行一条 JSON。这种格式的好处是：用 `grep` 就能快速搜索，用 `pandas` 就能批量分析，不需要额外的数据库。

### 第二层：指标（Metrics）

日志是原始数据，指标是从日志中聚合出来的**可量化数字**。你可以把它理解成 Agent 系统的"仪表盘"。

日常运营中，你最需要关注的几个指标：

| 指标 | 含义 | 示例值 |
|------|------|--------|
| P50/P95 延迟 | 端到端响应时间 | P50=2.1s, P95=4.2s |
| 单次请求成本 | token 消耗换算成 API 费用 | 平均 $0.038 |
| 任务成功率 | 用户意图被正确完成的比例 | 91% |
| 幻觉率 | 自动检测出的事实错误比例 | 3.7% |
| 平均工具调用次数 | 每次任务平均调用多少次工具 | 2.3 次 |

举一个真实的场景：日均 1000 次请求的客服 Agent，P95 延迟 4.2 秒，成功率 91%，日均 API 成本 $38.50。这意味着每天有大约 90 次请求没有成功完成任务——可能是工具调用失败，可能是 LLM 给出了不合理的回答。如果你不看这个数字，根本不知道有这么多用户在"默默失败"。

下面是一个从日志中自动计算指标的脚本：

```python
import json
from collections import defaultdict
from pathlib import Path

def compute_daily_metrics(log_dir: str, date: str) -> dict:
    """从 JSONL 日志中计算当日核心指标"""
    log_file = Path(log_dir) / f"{date}.jsonl"
    entries = []
    with open(log_file, encoding="utf-8") as f:
        for line in f:
            entries.append(json.loads(line))

    # 按 session 分组
    sessions = defaultdict(list)
    for e in entries:
        sessions[e["session_id"]].append(e)

    total_requests = len(sessions)
    total_cost = 0.0
    durations = []
    success_count = 0

    for sid, events in sessions.items():
        # 计算单次请求的延迟（所有步骤耗时之和）
        session_duration = sum(e.get("duration_ms", 0) for e in events)
        durations.append(session_duration)

        # 计算 token 成本（以 GPT-4o 为例：input $2.5/1M, output $10/1M）
        for e in events:
            if e["step_type"] == "llm_call" and e.get("token_usage"):
                prompt_tokens = e["token_usage"].get("prompt", 0)
                completion_tokens = e["token_usage"].get("completion", 0)
                cost = (prompt_tokens * 2.5 + completion_tokens * 10) / 1_000_000
                total_cost += cost

        # 判断是否成功（有 final_output 且有明确的成功标记）
        final_events = [e for e in events if e["step_type"] == "final_output"]
        if final_events and final_events[0]["content"].get("success", False):
            success_count += 1

    durations.sort()
    p50 = durations[len(durations) // 2] if durations else 0
    p95 = durations[int(len(durations) * 0.95)] if durations else 0

    return {
        "date": date,
        "total_requests": total_requests,
        "avg_cost_per_request": round(total_cost / max(total_requests, 1), 4),
        "total_cost_usd": round(total_cost, 2),
        "p50_latency_ms": round(p50, 1),
        "p95_latency_ms": round(p95, 1),
        "success_rate": round(success_count / max(total_requests, 1) * 100, 1),
    }


# ── 使用示例 ──────────────────────────────────────
metrics = compute_daily_metrics("./agent_logs", "2026-07-09")
print(json.dumps(metrics, indent=2, ensure_ascii=False))
# 输出示例:
# {
#   "date": "2026-07-09",
#   "total_requests": 1024,
#   "avg_cost_per_request": 0.0385,
#   "total_cost_usd": 39.42,
#   "p50_latency_ms": 2100.0,
#   "p95_latency_ms": 4200.0,
#   "success_rate": 91.2
# }
```

有了这个脚本，你可以设一个每日定时任务，把指标推送到 Slack 或飞书群里。一旦成功率从 91% 掉到 85%，或者 P95 延迟从 4.2s 飙升到 8s，你第一时间就能发现。

### 第三层：追踪（Tracing）

日志是平铺的记录，指标是聚合的数字，而追踪（Tracing）是**一次请求的完整链路**。

打个比方：日志像是每个路口的监控截图，指标像是全城的平均车速，而追踪是一辆车从出发到目的地的**完整行驶轨迹**。

追踪系统的核心概念是 **Span（跨度）**，这个概念来自微服务领域的分布式追踪（如 Jaeger、Zipkin）。一个 Span 代表一个操作单元，Span 之间可以嵌套形成树形结构。

一次 Agent 请求的 Trace 长这样：

```
[Root Span: 用户请求]  (总耗时 3200ms)
  ├── [Span: LLM 推理 #1]  (1800ms, 决定调用搜索工具)
  │     └── prompt_tokens: 320, completion_tokens: 85
  ├── [Span: 工具调用 search_api]  (600ms)
  │     └── query: "2026年AI发展趋势", results: 5条
  ├── [Span: LLM 推理 #2]  (700ms, 基于搜索结果生成回答)
  │     └── prompt_tokens: 580, completion_tokens: 210
  └── [Span: 最终输出]  (100ms)
```

下面是一个简洁但完整的 Trace 系统实现：

```python
import time
import uuid
from dataclasses import dataclass, field
from typing import Any
from contextlib import contextmanager

@dataclass
class Span:
    """一个操作跨度"""
    name: str
    span_id: str = field(default_factory=lambda: uuid.uuid4().hex[:8])
    parent_id: str | None = None
    start_time: float = 0
    end_time: float = 0
    attributes: dict[str, Any] = field(default_factory=dict)
    children: list["Span"] = field(default_factory=list)

    @property
    def duration_ms(self) -> float:
        return (self.end_time - self.start_time) * 1000

    def to_dict(self) -> dict:
        """序列化为字典，方便存储和传输"""
        d = {
            "span_id": self.span_id,
            "name": self.name,
            "parent_id": self.parent_id,
            "duration_ms": round(self.duration_ms, 1),
            "attributes": self.attributes,
        }
        if self.children:
            d["children"] = [c.to_dict() for c in self.children]
        return d


class Tracer:
    """Agent 追踪器"""

    def __init__(self):
        self.traces: list[Span] = []

    @contextmanager
    def trace(self, name: str, **attributes):
        """创建一个 Root Span 上下文"""
        root = Span(name=name, attributes=attributes)
        root.start_time = time.time()
        self.traces.append(root)
        self._current = root
        try:
            yield root
        finally:
            root.end_time = time.time()

    @contextmanager
    def span(self, name: str, **attributes):
        """在 Root Span 下创建子 Span"""
        child = Span(
            name=name,
            parent_id=self._current.span_id,
            attributes=attributes,
        )
        child.start_time = time.time()
        self._current.children.append(child)

        parent = self._current
        self._current = child
        try:
            yield child
        finally:
            child.end_time = time.time()
            self._current = parent

    def export_traces(self) -> list[dict]:
        """导出所有 Trace"""
        return [t.to_dict() for t in self.traces]


# ── 使用示例 ──────────────────────────────────────
tracer = Tracer()

with tracer.trace("user_request", query="帮我总结今天的新闻") as root:

    with tracer.span("llm_call", model="gpt-4o", step="planning") as s:
        # 模拟 LLM 调用
        time.sleep(0.8)
        s.attributes["response"] = "需要搜索新闻并总结"
        s.attributes["tokens"] = {"prompt": 150, "completion": 40}

    with tracer.span("tool_call", tool="news_search") as s:
        time.sleep(0.3)
        s.attributes["result_count"] = 5

    with tracer.span("llm_call", model="gpt-4o", step="generation") as s:
        time.sleep(0.6)
        s.attributes["tokens"] = {"prompt": 480, "completion": 200}

# 打印追踪结果
import json
print(json.dumps(tracer.export_traces(), indent=2, ensure_ascii=False))
```

Trace 的价值在于**问题定位**。当一个用户反馈"Agent 给了个错误答案"时，你可以用 session_id 找到对应的 Trace，一步步看：是哪个工具调用返回了异常数据？是 LLM 在哪一步做了错误判断？是 prompt 写得不好还是上下文信息不够？没有 Trace，这些问题只能靠猜。

## LangFuse：开源的 Agent 可观测性平台

自己写日志和追踪系统是不错的学习练习，但生产环境中你大概率会选择一个现成的平台。**LangFuse** 是目前最流行的开源 LLM 可观测性平台（GitHub 上 8000+ stars），专门为 LLM 应用和 Agent 系统设计。

LangFuse 提供四个核心能力：

- **Trace 可视化**：在 Web UI 中查看每次请求的完整链路，每个 Span 的耗时、输入输出、token 消耗一目了然
- **指标仪表盘**：延迟分布、成本趋势、成功率的实时图表，支持自定义告警
- **评分与标注**：对 Trace 打标签——标记"这是一个好的回答"或"这里出现了幻觉"，这些标注后续可以用来筛选训练数据
- **数据集管理**：从标注过的 Trace 中直接导出数据集，无缝对接微调流程

接入 LangFuse 非常简单。如果你用的是 LangChain，只需要加一个 Callback：

```python
from langchain_openai import ChatOpenAI
from langchain.agents import create_react_agent, AgentExecutor
from langchain import hub
from langchain.tools import tool
import os

# ── 设置 LangFuse 环境变量 ──────────────────────────
os.environ["LANGFUSE_PUBLIC_KEY"] = "pk-lf-xxxxxxxxxxxxx"
os.environ["LANGFUSE_SECRET_KEY"] = "sk-lf-xxxxxxxxxxxxx"
os.environ["LANGFUSE_HOST"] = "https://cloud.langfuse.com"

# ── 导入 LangFuse 的 Callback Handler ──────────────
from langfuse.callback import CallbackHandler

langfuse_handler = CallbackHandler(
    session_id="demo-session",
    user_id="user-001",
    tags=["production", "v2-agent"],
)

# ── 定义工具 ──────────────────────────────────────
@tool
def search_web(query: str) -> str:
    """搜索互联网获取信息"""
    return f"搜索结果: 关于'{query}'的最新信息..."

# ── 创建 Agent 并接入 LangFuse ────────────────────
llm = ChatOpenAI(model="gpt-4o", temperature=0)
prompt = hub.pull("hwchase17/react")
tools = [search_web]

agent = create_react_agent(llm, tools, prompt)
agent_executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

# 运行时传入 callbacks 参数即可
result = agent_executor.invoke(
    {"input": "2026年大语言模型的最新趋势是什么？"},
    config={"callbacks": [langfuse_handler]},
)

print(result["output"])
```

就这么简单。运行之后打开 LangFuse 的 Web UI，你会看到这样的界面：

左侧是按时间排列的 Trace 列表，每条 Trace 显示总耗时、token 消耗和成本。点击某条 Trace 进入详情页，右侧会展示完整的 Span 树——Root Span 下面是 LLM 调用、工具调用、再 LLM 调用的完整链路。每个 Span 可以展开看到具体的 prompt 内容、模型返回的原文、工具参数和返回结果。顶部还有耗时瀑布图，直观地展示每一步花了多长时间。

除了 LangChain 集成，LangFuse 还提供了 Python SDK 的装饰器用法，适合非 LangChain 项目：

```python
from langfuse.decorators import observe, langfuse_context

@observe()
def my_agent(query: str) -> str:
    """被 @observe 装饰的函数会自动创建 Trace"""
    plan = plan_step(query)       # 自动记录为子 Span
    result = execute_step(plan)   # 自动记录为子 Span
    return format_output(result)

@observe(as_type="generation")
def plan_step(query: str) -> str:
    """标记为 generation 类型，会自动记录 LLM 相关信息"""
    response = openai_client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": f"为以下问题制定计划: {query}"}],
    )
    # 手动更新 Span 的元数据
    langfuse_context.update_current_observation(
        usage={
            "prompt_tokens": response.usage.prompt_tokens,
            "completion_tokens": response.usage.completion_tokens,
        },
        model="gpt-4o",
    )
    return response.choices[0].message.content
```

## 数据飞轮：让 Agent 越用越好

这是整篇最让人兴奋的部分。

前面讲的日志、指标、追踪，都是"监控"——帮你发现问题、定位问题。但如果你只是监控，Agent 的能力永远不会提升。**数据飞轮**的思路完全不同：把 Agent 运行产生的日志变成训练数据，用这些数据微调一个专用模型，让 Agent 越来越强。

整个循环长这样：

```
Agent 运行 → 产生 Trace 日志 → 筛选高质量轨迹 → 构建 SFT 训练数据
    → 微调专用模型 → 替换通用 LLM → Agent 表现更好
    → 产生更多日志 → ...（循环加速）
```

这就像滚雪球——初始推动很费力，但一旦转起来，每一圈都让雪球变得更大。

### 环节 1：收集运行日志

这一步就是前面讲的可观测性。关键点是：**不仅要记录 Agent 做了什么，还要记录做得好不好**。

每个 Trace 需要附带质量信号：

- 用户有没有点"踩"（隐式反馈）
- 自动评估的分数（如答案相关性、忠实度）
- 任务是否成功完成
- 总耗时和总成本

这些质量信号在下一步筛选时就是核心过滤条件。

### 环节 2：筛选高质量轨迹

不是所有运行日志都适合做训练数据。把 Agent 犯错的过程拿去训练，只会让它"学会犯错"。

筛选策略分两层：

**自动筛选**（快速过滤掉明显低质量的数据）：
- 任务成功率 = 100%（最终输出了有效结果）
- 端到端延迟低于当日 P50（说明 Agent 没有反复试错）
- 自动幻觉检测未发现事实错误
- 用户没有负面反馈

**人工标注**（处理边界案例）：
- 对自动筛选后剩余的数据，抽样 10-20% 进行人工评审
- 重点看 Agent 的推理过程是否合理，不只是看最终结果

一个实际的筛选比例：日均 1000 次请求，运行 10 天积累 10000 条 Trace。自动筛选后剩下约 2800 条（28%），人工评审后保留 1200 条（12%）。这 1200 条就是高质量的训练数据。

### 环节 3：构建训练数据

把筛选出来的 Trace 转成 SFT（Supervised Fine-Tuning）格式。SFT 的核心思想很简单：给模型看"正确的示范"，让它学会照着做。

```python
import json
from pathlib import Path

def trace_to_sft_example(trace: dict) -> dict:
    """将一条 Trace 转换成 SFT 训练格式"""
    # 从 Trace 中提取用户输入
    user_input = trace["attributes"].get("query", "")

    # 从 Trace 的子 Span 中构建完整的推理过程
    reasoning_steps = []
    for child in trace.get("children", []):
        if child["name"] == "llm_call":
            reasoning_steps.append(
                f"[思考] {child['attributes'].get('response', '')}"
            )
        elif child["name"] == "tool_call":
            tool_name = child["attributes"].get("tool", "unknown")
            tool_result = child["attributes"].get("result", "")
            reasoning_steps.append(
                f"[调用工具 {tool_name}] 结果: {tool_result}"
            )

    # 最终输出
    final_output = ""
    for child in trace.get("children", []):
        if child["name"] == "final_output":
            final_output = child["attributes"].get("output", "")

    # 组装成完整的训练样本
    full_response = "\n".join(reasoning_steps)
    if final_output:
        full_response += f"\n[最终回答] {final_output}"

    return {
        "instruction": user_input,
        "output": full_response,
    }


def build_sft_dataset(traces: list[dict], output_path: str):
    """批量转换并保存训练数据集"""
    examples = []
    for trace in traces:
        try:
            example = trace_to_sft_example(trace)
            if example["instruction"] and example["output"]:
                examples.append(example)
        except Exception as e:
            print(f"跳过转换失败的 Trace: {e}")

    # 保存为 JSONL（Hugging Face datasets 可直接加载）
    with open(output_path, "w", encoding="utf-8") as f:
        for ex in examples:
            f.write(json.dumps(ex, ensure_ascii=False) + "\n")

    print(f"已生成 {len(examples)} 条训练数据 -> {output_path}")
    return examples


# ── 使用示例 ──────────────────────────────────────
# 假设 filtered_traces 是从环节2筛选出的高质量 Trace 列表
# build_sft_dataset(filtered_traces, "./sft_dataset.jsonl")
```

生成的训练数据长这样：

```json
{"instruction": "帮我查一下北京明天的天气", "output": "[思考] 用户需要查询天气，我需要调用天气API工具\n[调用工具 weather_api] 结果: 北京明天晴，最高温度32度\n[最终回答] 北京明天是晴天，最高温度32度，比较热，建议注意防晒。"}
```

这种格式的好处是：模型不仅学会了最终回答，还学会了**推理过程和工具调用的时机**。

### 环节 4：微调专用模型

有了训练数据，下一步是微调一个专用模型。你可以用 OpenAI 的 Fine-tuning API，也可以用 Hugging Face 的 TRL 库在本地微调开源模型（如 Llama 3 8B、Qwen 2.5 7B）。

微调后的模型在特定任务上的表现可以接近 GPT-4o，但成本低一个数量级：

| 模型 | 任务准确率 | 单次请求成本 | 延迟 P50 |
|------|-----------|-------------|---------|
| GPT-4o（通用） | 93% | $0.040 | 1.8s |
| 微调 Llama 3 8B | 91% | $0.002 | 0.4s |
| 成本节省 | -2% | **20 倍** | **4.5 倍** |

替换策略也很关键——不要一刀切地把所有请求都切到微调模型。合理的做法是**模型分级**：

- **简单任务**（意图明确、工具调用链路短）：用微调模型处理，占 60-70% 的请求量
- **复杂任务**（需要多步推理、涉及关键决策）：仍然用 GPT-4o
- **路由判断**：用一个轻量分类器（甚至用正则表达式）判断任务复杂度

这样既保证了质量，又大幅降低了成本。

### 飞轮的"复利效应"

数据飞轮最迷人的地方是它的复利效应。以一个真实场景为例——一个日均处理 1000 次请求的客服 Agent：

**第 1 个月**：积累约 1000 条高质量 Trace，微调出 v1 模型。简单任务（FAQ 回答、订单查询）的准确率从 72% 提升到 85%。月 API 成本 $3000，因为大部分请求还在用 GPT-4o。

**第 3 个月**：积累了 10000 条 Trace，微调出 v2 模型。准确率提升到 91%，可以替代 60% 的 GPT-4o 调用。月成本降到 $1500。更重要的是，v2 模型因为更熟悉业务场景，平均响应时间从 2.1s 降到了 0.8s。

**第 6 个月**：积累了 50000 条 Trace，微调出 v3 模型。准确率 95%，只有最复杂的 15% 的请求才需要 GPT-4o。月成本降到 $800——比初始降低了 73%。

这就是"数据飞轮"的威力：用户使用越多，日志越多；日志越多，训练数据越丰富；训练数据越丰富，模型越好；模型越好，用户体验越好，使用量继续增长。

和传统的"花更多钱买更强的模型"完全不同，数据飞轮是"用得越多，效果越好，成本越低"。

## 从 Demo 到生产的 Checklist

到这一篇，我们已经覆盖了 Agent 系统从开发到生产的所有关键环节。用一个表格做总结——上线前逐项检查：

| 类别 | 检查项 | 重要程度 | 说明 |
|------|--------|---------|------|
| 评估 | 基准测试集 + 自动化评估 pipeline | 必须 | 每次模型/prompt 变更后自动运行 |
| 安全 | 输入过滤 + 输出验证 + PII 脱敏 | 必须 | 至少覆盖 prompt injection 和敏感信息泄露 |
| 可观测 | 结构化日志 + 核心指标 + Trace 接入 | 必须 | 出问题能在 5 分钟内定位到具体 Span |
| 性能 | 延迟优化 + 缓存策略 + 并发控制 | 推荐 | 语义缓存可减少 30-40% 的重复调用 |
| 成本 | 预算上限 + 成本监控 + 模型分级 | 推荐 | 设置每日成本告警阈值 |
| 持续优化 | 数据飞轮 + 定期微调 | 加分项 | 上线 1 个月后开始积累，3 个月后开始首轮微调 |

"必须"级别的事项没做完，不要上线。"推荐"级别的事项没做，上线后会很痛苦。"加分项"是让你的 Agent 从"能用"变成"好用"的长期投资。

## 下一步

到这篇为止，生产化阶段的理论知识已经全部讲完——评估、安全、可观测性、数据飞轮，这四块构成了 Agent 系统从 Demo 到生产的完整闭环。

但理论总要落地。下一篇是纯实战——我们会用 LangFuse 和 Ragas 这两个工具，把前面讲的所有东西全部跑一遍：搭建 Trace 追踪、配置自动评估 pipeline、从运行日志中筛选训练数据、执行一轮微调。从安装到运行，全部是可复现的代码。

## 推荐资源

- [LangFuse 文档](https://langfuse.com/docs) -- 开源 LLM 可观测性平台，本地部署或云版均可
- [LangSmith 文档](https://docs.smith.langchain.com/) -- LangChain 官方的追踪平台，与 LangChain 生态深度集成
- [OpenAI Fine-tuning Guide](https://platform.openai.com/docs/guides/fine-tuning) -- OpenAI 官方微调入门，支持 GPT-4o-mini 等模型的微调
- [Hugging Face TRL](https://github.com/huggingface/trl) -- 用 Transformer Reinforcement Learning 微调开源模型，支持 SFT/DPO/PPO
