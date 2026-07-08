---
title: "08 | 实战——用 LangChain 构建一个 ReAct Agent"
date: 2026-07-06
draft: false
weight: 204
tags: ["Agent", "LangChain", "实战", "ReAct"]
summary: "用 LangChain 从零构建一个带工具调用的 ReAct Agent。包括创建 Agent、添加工具、观察循环日志、把多个工具组合成 Skill。完整可运行代码，跟着做就能跑起来。"
ShowToc: true
---
前三篇我们学了 Agent 循环的理论（ReAct/Reflexion）、Function Calling 的机制、以及 Tool→Skill→MCP 的封装思维。这篇文章把它们全部落地——用 LangChain 框架构建一个真正能运行的 ReAct Agent。

我们的目标是做一个**"智能研究助手" Agent**：给它一个研究主题，它能自主搜索信息、分析数据、计算结果，最终生成一份研究报告。

![实战项目架构](/images/stage2-08-langchain-agent.svg)

---

## 环境准备

### 安装依赖

```bash
pip install langchain langchain-classic langchain-openai langchain-community langsmith ddgs python-dotenv
```

> 注意：LangChain 1.3.x 将经典 Agent 框架（`create_react_agent`、`AgentExecutor`）迁移到了独立的 `langchain-classic` 包，拉取 Prompt 模板需要通过 `langsmith` 客户端。

### 配置 API Key

```bash
# .env 文件
OPENAI_API_KEY=sk-your-key-here
# 或者用 DeepSeek：
# OPENAI_BASE_URL=https://api.deepseek.com
# OPENAI_API_KEY=sk-your-deepseek-key
# 如果用 LangSmith Hub 拉取 Prompt（可选）：
# LANGSMITH_API_KEY=your-langsmith-key
```

```python
import os
from dotenv import load_dotenv
load_dotenv()
```

---

## Step 1：创建你的第一个 Agent

LangChain 提供了 `create_react_agent` 函数，几行代码就能创建一个标准的 ReAct Agent：

```python
from langchain_openai import ChatOpenAI
from langchain_classic.agents import create_react_agent, AgentExecutor
from langsmith import Client

# 1. 选择 LLM
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# 2. 拉取 ReAct 提示模板
client = Client()
prompt = client.pull_prompt("hwchase17/react", dangerously_pull_public_prompt=True)

# 3. 暂时不加工具，先看看 Agent 的结构
agent = create_react_agent(llm, tools=[], prompt=prompt)
agent_executor = AgentExecutor(agent=agent, tools=[], verbose=True)
```

> 从 LangChain 1.3.x 开始，`create_react_agent` 和 `AgentExecutor` 从 `langchain_classic.agents` 导入。Prompt 模板通过 LangSmith 客户端从 Hub 拉取，`dangerously_pull_public_prompt=True` 表示显式允许拉取公共模板。

`verbose=True` 是关键——它会打印 Agent 的每一轮思考、行动和观察，让你能"看到"循环在怎么跑。

---

## Step 2：添加工具

给 Agent 加三个工具：网页搜索、数学计算、Python 执行。

```python
from langchain.tools import tool
from ddgs import DDGS

# 工具 1: 网页搜索
@tool
def search_web(query: str) -> str:
    """搜索互联网获取实时信息。输入搜索关键词。"""
    with DDGS() as ddgs:
        results = list(ddgs.text(query, max_results=3))
    return "\n".join([f"{r['title']}: {r['body']}" for r in results])

# 工具 2: 数学计算
@tool
def calculator(expression: str) -> str:
    """计算数学表达式。输入如 '2 + 3 * 4' 或 '(100 - 20) / 4'。"""
    try:
        # 安全计算（仅允许数学运算）
        allowed = set("0123456789+-*/()._ ")
        if not all(c in allowed for c in expression):
            return "错误：包含不允许的字符"
        result = eval(expression)
        return f"计算结果: {result}"
    except Exception as e:
        return f"计算错误: {e}"

# 工具 3: Python 代码执行
@tool
def run_python(code: str) -> str:
    """执行 Python 代码并返回输出。适合数据处理和分析任务。"""
    import io
    import contextlib
    output = io.StringIO()
    try:
        with contextlib.redirect_stdout(output):
            exec(code)
        return output.getvalue() or "代码执行成功（无输出）"
    except Exception as e:
        return f"执行错误: {e}"

# 组装工具列表
tools = [search_web, calculator, run_python]
```

注意每个工具函数的 **docstring**——LangChain 会把它当作工具的 description，传给 LLM 做 Function Calling。写得越清楚，Agent 选工具越准。

---

## Step 3：运行 Agent 并观察循环

现在让 Agent 做一个需要多个工具协作的任务：

```python
# 重建 Agent（带工具）
agent = create_react_agent(llm, tools=tools, prompt=prompt)
agent_executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

# 运行
result = agent_executor.invoke({
    "input": "帮我查一下 2025 年全球 AI 市场规模（亿美元），"
             "然后计算如果年增长率是 35%，2027 年预计达到多少？"
})

print("\n最终答案:", result["output"])
```

`verbose=True` 会打印出 Agent 的完整循环日志，类似这样：

```
> Entering new AgentExecutor chain...

Thought: 我需要先搜索 2025 年全球 AI 市场规模的数据
Action: search_web
Action Input: "2025 global AI market size billion dollars"
Observation: Global AI market size in 2025 is estimated at $196 billion...

Thought: 找到了，2025 年市场规模约 1960 亿美元。现在计算 2027 年的预测值
Action: calculator
Action Input: "1960 * (1 + 0.35) ** 2"
Observation: 计算结果: 3571.1

Thought: 计算完成。2025 年 1960 亿美元，按 35% 年增长率，2027 年约 3571 亿美元
Action: Final Answer
Final Answer: 2025 年全球 AI 市场规模约 1960 亿美元。
按 35% 的年增长率计算，预计 2027 年将达约 3571 亿美元。

> Finished chain.
```

**仔细观察这个循环**：

- 第 1 轮：Thought 决定搜索 → Action 调用 search_web → Observation 获得数据
- 第 2 轮：Thought 决定计算 → Action 调用 calculator → Observation 获得结果
- 第 3 轮：Thought 决定信息足够 → Action 输出 Final Answer

这就是 ReAct 循环在实际运行中的样子。Agent 自己决定什么时候用什么工具、什么时候结束。

---

## Step 4：把工具封装成 Skill

现在我们有了三个原子工具。按照上一篇讲的封装思维，把"研究并生成报告"封装成一个 Skill：

```python
@tool
def research_and_report(topic: str) -> str:
    """对指定主题进行深度研究：搜索多个来源，分析数据，生成结构化报告。
    适合需要综合多个信息源的研究任务。"""

    queries = [
        f"{topic} market size 2025",
        f"{topic} trends and predictions",
        f"{topic} key players"
    ]

    all_results = []
    for query in queries:
        with DDGS() as ddgs:
            results = list(ddgs.text(query, max_results=3))
        all_results.append("\n".join([f"{r['title']}: {r['body']}" for r in results]))

    combined = "\n".join([f"- {item}" for item in all_results])

    from langchain_openai import ChatOpenAI
    llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

    report = llm.invoke(
        f"基于以下搜索结果，为{topic}撰写一份简明报告：\n{combined}"
    )

    return report.content
```

现在 Agent 可以把 `research_and_report` 当作一个高级工具使用，而不需要自己编排搜索+分析的步骤：

```python
# Agent 现在有 4 个工具：3 个原子工具 + 1 个 Skill
all_tools = [search_web, calculator, run_python, research_and_report]

agent = create_react_agent(llm, tools=all_tools, prompt=prompt)
agent_executor = AgentExecutor(agent=agent, tools=all_tools, verbose=True)

# Agent 会自动选择合适的工具
result = agent_executor.invoke({
    "input": "帮我做一个关于量子计算行业的研究"
})
```

---

## Step 5：添加防护机制

生产级 Agent 必须加防护。把第五篇讲的安全措施落地：

```python
agent_executor = AgentExecutor(
    agent=agent,
    tools=all_tools,
    verbose=True,

    # 防护 1: 最大循环次数
    max_iterations=10,

    # 防护 2: 超时
    max_execution_time=120,  # 秒

    # 防护 3: 错误处理
    handle_parsing_errors=True,   # 自动处理输出解析错误
    handle_tool_errors=True,      # 自动处理工具调用错误

    # 防护 4: 提前停止回调
    early_stopping_method="generate",    # 超时时让 LLM 生成当前最佳回答
    early_stopping_threshold=0.5,        # 提前停止的置信度阈值
)
```

---

## Step 6：循环日志分析

加一个简单的日志系统，记录每轮循环的详细信息：

```python
from langchain_core.callbacks.base import BaseCallbackHandler
import time

class LoopLogger(BaseCallbackHandler):
    def __init__(self):
        self.loop_count = 0
        self.start_time = None
        self.logs = []

    def on_agent_action(self, action, **kwargs):
        if self.start_time is None:
            self.start_time = time.time()

        self.loop_count += 1
        elapsed = time.time() - self.start_time

        log_entry = {
            "loop": self.loop_count,
            "thought": action.log,
            "tool": action.tool,
            "tool_input": action.tool_input,
            "elapsed_seconds": round(elapsed, 2)
        }
        self.logs.append(log_entry)
        print(f"\n=== Loop #{self.loop_count} ({log_entry['elapsed_seconds']}s) ===")
        print(f"Tool: {action.tool}")
        print(f"Input: {action.tool_input}")

    def on_agent_finish(self, finish, **kwargs):
        total = time.time() - self.start_time if self.start_time else 0
        print(f"\n=== Summary ===")
        print(f"Total loops: {self.loop_count}")
        print(f"Total time: {total:.1f}s")

# 使用
logger = LoopLogger()
agent_executor = AgentExecutor(
    agent=agent, tools=all_tools,
    callbacks=[logger], verbose=False
)

result = agent_executor.invoke({"input": "研究一下 RAG 技术的最新进展"})

# 事后分析日志
for log in logger.logs:
    print(f"Loop {log['loop']}: {log['tool']} | {log['elapsed_seconds']}s")
```

这些日志数据在第五阶段会接入 LangFuse 做系统化分析。

---

## 关键收获

通过这个实战项目，你应该理解了：

**ReAct 循环的实际运行**：Thought→Action→Observation 不只是理论，它是 Agent 每次决策的真实过程。通过 verbose 日志和自定义 Callback，你能"看到"循环在跑。

**工具设计的直接影响**：docstring 写得好不好，直接决定 Agent 能不能正确选择工具。模糊的描述 = 错误的行为。

**Skill 封装的价值**：`research_and_report` 把 3 步操作压缩成 1 次调用，减少了 Agent 的循环次数和出错概率。

**防护不是可选的**：max_iterations、timeout、error handling 缺一不可。没有防护的 Agent 就像一个没有刹车的汽车。

---

## 第二阶段总结

恭喜完成第二阶段！回顾你学到的东西：

| 篇目 | 核心知识                                                                       |
| ---- | ------------------------------------------------------------------------------ |
| 05   | Agent 的核心是 ReAct 循环：Thought→Action→Observation，以及 Reflexion 自反思 |
| 06   | Function Calling 的机制：LLM 输出调用意图，外部代码执行，结果回传              |
| 07   | Tool 是原子操作，Skill 是组合流程，MCP 是标准化连接协议                        |
| 08   | 用 LangChain 构建 ReAct Agent 的完整实战                                       |

下一阶段我们会进入 **Context Engineering 与 RAG**——学习怎么给 Agent 提供高质量的上下文信息，让它做出更好的决策。你会学到 RAG 全流程、记忆系统、以及 MCP 的深入应用。

---

## 扩展练习

想继续深入？试试这些：

1. **添加 Reflexion**：当 Agent 的工具调用失败时，触发一个专门的反思步骤
2. **接入 MCP**：把 search_web 工具替换为 MCP Server 调用，体验标准化协议
3. **多轮对话 Agent**：给 Agent 加记忆，让它能进行多轮对话而不是单次任务
4. **对比框架**：用 CrewAI 或 AutoGen 实现同样的 Agent，对比 LangChain 的体验
