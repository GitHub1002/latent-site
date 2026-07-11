---
title: "16 | 实战——用 CrewAI 构建多 Agent 协作系统"
date: 2026-07-08
draft: false
weight: 404
tags: ["Agent", "CrewAI", "Multi-Agent", "实战"]
summary: "用 CrewAI 框架构建一个'行业研究团队'：研究员搜集资料、分析师整理数据、写作 Agent 撰写报告。从安装到运行，完整演示多 Agent 协作系统的搭建过程，包括角色定义、任务编排、工具配置和 Harness 约束。"
ShowToc: true
---

前面三篇文章，我们分别拆解了多 Agent 的架构模式（文章 13）、A2A 通信协议（文章 14）、以及 Harness Engineering 的工程约束思维（文章 15）。理论讲够了，这篇我们把这些东西全部落地。

这次用的框架是 CrewAI——目前 GitHub 上 28k+ stars，多 Agent 领域最火也最容易上手的开源框架。选它的原因很直接：概念少、API 干净、五分钟能跑起来第一个 Demo。相比之下，AutoGen 更偏学术、LangGraph 更偏底层图编排，对初学者来说学习曲线都更陡。

CrewAI 的核心概念只有四个，打个比方就像组建一个小公司：

- **Agent**（角色）：你招的员工，有明确的职位、目标和背景
- **Task**（任务）：你分配给每个员工的具体工作
- **Crew**（团队）：把这些员工和任务组装成一个项目组
- **Tool**（工具）：员工完成工作时可以使用的工具箱

就这四样。没有复杂的图结构，没有状态机，就是一个直觉化的「角色-任务-团队」模型。

## 项目目标

我们要构建一个「行业研究团队」。你给它一个行业名称（比如"量子计算"），它会自动跑完以下流程：

1. **研究员** 搜索行业信息，搜集市场规模、主要玩家、技术趋势等原始数据
2. **分析师** 接手这些原始数据，做竞争分析，提取关键洞察
3. **写作员** 基于分析结果，撰写一份结构完整的行业研究报告

最终输出包含三部分：行业概况报告、竞争分析表格、投资建议摘要。

![CrewAI 多 Agent 协作系统架构](/images/stage4-16-crewai-practical.svg)

这三个 Agent 形成一个流水线：前一个的输出是后一个的输入，跟工厂的装配线一个道理。

## Step 1: 安装和环境准备

老规矩，先建虚拟环境：

```bash
python -m venv crewai-demo
cd crewai-demo
# Windows
.venv\Scripts\activate
# macOS/Linux
source .venv/bin/activate
```

安装 CrewAI 和它的工具包：

```bash
pip install crewai crewai-tools
```

截至 2026 年中，`crewai` 最新稳定版是 0.105.x，`crewai-tools` 是 0.42.x。安装完之后跑一下版本确认：

```python
import crewai
print(crewai.__version__)
# 输出: 0.105.2
```

接下来配置 API Key。CrewAI 底层支持多种 LLM，这里我们用 DeepSeek（便宜好用）。同时搜索工具需要 Serper 的 API Key。最方便的方式是建一个 `.env` 文件：

```bash
# .env 文件
OPENAI_API_KEY=sk-your-deepseek-key-here
OPENAI_API_BASE=https://api.deepseek.com
SERPER_API_KEY=your-serper-api-key
```

然后在代码中用 `dotenv` 加载：

```bash
pip install python-dotenv
```

```python
import os
from dotenv import load_dotenv
load_dotenv()
```

`OPENAI_API_KEY` 和 `OPENAI_API_BASE` 是 CrewAI 底层走 OpenAI 兼容接口的方式，配合 DeepSeek 的 API 地址就能无缝使用。`SERPER_API_KEY` 在 [serper.dev](https://serper.dev) 注册获取（有免费额度，2500 次搜索）。

## Step 2: 定义 Agent 角色

这一步是整个系统最关键的部分。Agent 的输出质量，80% 取决于你怎么定义它的角色。

还记得文章 13 里讲的「角色设计三要素」吗？role（做什么）、goal（做到什么程度）、backstory（凭什么能做好）。这三个字段不是填表走形式——它们会直接注入 LLM 的 system prompt，影响 Agent 每一次推理的走向。

来写代码：

```python
import os
from dotenv import load_dotenv
load_dotenv()

from crewai import Agent, LLM
from crewai_tools import SerperDevTool

# 模型配置
deepseek_model = LLM(
    model="deepseek-v4-flash",
    temperature=0.3
)

# 搜索工具（后面 Step 3 会详细讲）
search_tool = SerperDevTool(n_results=3, api_key=os.getenv("SERPER_API_KEY"))

# 研究员 Agent
researcher = Agent(
    role="行业研究员",
    goal="针对 {industry} 行业，搜集最新的市场数据、技术进展、主要公司和行业趋势",
    backstory="""你是一位资深行业研究员，在麦肯锡和贝恩有超过 10 年的咨询经验。
你擅长从海量信息中快速识别关键数据点，尤其关注市场规模（具体数字）、
增长率（CAGR）、技术成熟度曲线和头部公司的最新动态。
你的输出以事实和数据为主，不做主观判断。""",
    tools=[search_tool],
    llm=deepseek_model,
    verbose=True,
    max_iterations=3,
    allow_delegation=False
)

# 分析师 Agent
analyst = Agent(
    role="竞争分析师",
    goal="基于研究员提供的原始数据，提炼 {industry} 行业的竞争格局、关键洞察和投资价值",
    backstory="""你是一位严谨的商业分析师，专注于技术行业的竞争分析。
你擅长使用波特五力、SWOT 等分析框架，能从杂乱的数据中提取结构化洞察。
你特别关注：市场份额分布、技术壁垒、融资轮次和估值趋势。
你的输出需要有明确的结论和量化支撑。""",
    llm=deepseek_model,
    max_iterations=3,
    allow_delegation=False,
    verbose=True
)

# 写作员 Agent
writer = Agent(
    role="行业报告撰写员",
    goal="将分析师的 {industry} 行业分析转化为一份面向投资者的专业报告",
    backstory="""你是一位资深的金融写作专家，长期为顶级投行撰写行业研究报告。
你的文风专业但不晦涩，擅长用清晰的逻辑和生动的数据讲故事。
你特别注重报告的可读性：执行摘要要一针见血，结论要有明确立场，
风险提示要具体而非泛泛而谈。""",
    llm=deepseek_model,
    max_iterations=3,
    allow_delegation=False,
    verbose=True
)
```

几个值得注意的设计选择：

**`{industry}` 占位符**——注意 goal 里的 `{industry}`。这不是 Python 的 f-string，而是 CrewAI 的模板变量。后面我们调用 `crew.kickoff(inputs={"industry": "量子计算"})` 时，它会被自动替换。这样做的好处是一套 Agent 定义可以复用于任何行业。

**backstory 的颗粒度**——你可能觉得 backstory 写这么长是多余的。不是的。对比一下：

```python
# 太模糊——输出会很泛
backstory="你是一位行业研究员"

# 有细节——输出会更有针对性和专业感
backstory="你在麦肯锡有 10 年经验，擅长从海量信息中识别关键数据点，关注市场规模和 CAGR..."
```

实测下来，backstory 从 1 行扩展到 4-5 行，报告的信息密度能提升 30-40%（以"量子计算"为例，前者输出约 800 字且多为泛泛而谈，后者输出约 1500 字且包含具体公司和数据）。

**`allow_delegation=False`**——关闭 Agent 之间的任务委派。在我们的流水线架构里，每个 Agent 只做自己的事，不需要把任务丢给别人。开启委派会增加不必要的 LLM 调用，浪费 token 和时间。

## Step 3: 定义工具（Tools）

工具让 Agent 能够跟外部世界交互。没有工具的 Agent 只能用 LLM 内部的知识回答问题，那叫"生成"，不叫"研究"。

### 使用内置工具

CrewAI 的工具包 `crewai-tools` 提供了十几个开箱即用的工具。我们给研究员用的是 `SerperDevTool`，它调用 Serper.dev 的 Google 搜索 API：

```python
from crewai_tools import SerperDevTool

search_tool = SerperDevTool(
    n_results=10,  # 每次搜索返回 10 条结果
)
```

要使用这个工具，你需要在 [serper.dev](https://serper.dev) 注册一个 API Key（有免费额度，2500 次搜索）：

```bash
export SERPER_API_KEY="your-serper-api-key"
```

### 自定义工具

有时候你需要一个专属工具。比如我们想加一个"从特定数据源拉取行业报告摘要"的功能。CrewAI 自定义工具非常简单——继承 `BaseTool` 类，实现 `_run` 方法：

```python
from crewai.tools import BaseTool
from pydantic import BaseModel, Field


class IndustryReportInput(BaseModel):
    """自定义工具的输入 schema"""
    industry: str = Field(description="行业名称，如'量子计算'、'新能源'", json_schema_extra={"example": "人工智能"})


class IndustryReportTool(BaseTool):
    name: str = "industry_report"  # 必须用英文！CrewAI 内部会转为 OpenAI function name
    description: str = "根据行业名称，生成该行业的最新报告摘要。输入行业名称，返回该行业的市场规模、技术进展和融资情况等关键数据。"
    args_schema: type[BaseModel] = IndustryReportInput

    def _run(self, industry: str) -> str:
        # 实际项目中这里会调用数据库或 API
        # 这里用模拟数据演示
        reports = {
            "量子计算": (
                "1. McKinsey 2025 量子技术报告：全球量子计算市场预计 2030 年达 280 亿美元\n"
                "2. BCG 量子优势分析：量子纠错技术进入实用化阶段\n"
                "3. PitchBook 数据：2025 年量子计算领域融资总额达 42 亿美元"
            ),
            "新能源": (
                "1. IEA 2025 全球能源展望：可再生能源装机量首次超过化石燃料\n"
                "2. Bloomberg NEF：2025 年清洁能源投资达 1.8 万亿美元\n"
                "3. 储能技术成本在过去 3 年下降 60%"
            ),
        }
        return reports.get(industry, f"暂未找到 {industry} 的行业报告数据")
```

这里有个容易踩的坑：**`name` 字段必须用英文**（字母+数字+下划线）。CrewAI 内部会把工具名转成 OpenAI 的 function name 格式，中文字符会被过滤掉，如果名字全是中文就会变成空字符串，直接报 `ValueError: OpenAI function name cannot be empty`。

这个自定义工具的使用方式和内置工具完全一样：

```python
report_tool = IndustryReportTool()

# 把它加到研究员的工具列表
researcher = Agent(
    # ... 其他参数同上
    tools=[search_tool, report_tool],  # 现在有两个工具
)
```

### Tool 和 MCP 的关系

读过文章 07（MCP 协议）的同学可能会问：CrewAI 的 Tool 和 MCP 的 Tool 是什么关系？

简单说，它们解决的问题不同但互补：

- **CrewAI Tool**：框架内部的工具接口，定义「Agent 能调用什么函数」。它是 Agent 和工具之间的绑定关系。
- **MCP Tool**：跨进程的标准协议，定义「不同系统之间怎么共享工具」。它是工具提供方和消费方之间的通信标准。

如果你有一个 MCP Server 暴露了某个工具，完全可以在 CrewAI 里写一个 Adapter Tool 把它包一层。CrewAI 社区已经有了 `mcp-tool-adapter` 这样的第三方包来做这件事。

## Step 4: 定义任务（Tasks）

角色和工具准备好了，现在要给每个角色安排具体工作。Task 就是任务说明书：

```python
from crewai import Task

# 研究员的任务：搜集原始数据
research_task = Task(
    description="""
    针对 {industry} 行业进行全面的信息搜集。

    你需要搜集以下信息：
    1. 市场规模（当前值和预测值，带具体数字）
    2. 主要公司（至少 5 家，包括名称、核心业务、融资情况）
    3. 技术趋势（当前主流技术路线和未来方向）
    4. 行业关键事件（最近 12 个月的重要新闻）

    输出要求：以结构化的方式呈现原始数据，每条信息标注来源。
    """,
    expected_output="一份包含市场规模、主要公司、技术趋势和关键事件的结构化数据报告，至少 1500 字",
    agent=researcher,
)

# 分析师的任务：做竞争分析
analysis_task = Task(
    description="""
    基于研究员提供的 {industry} 行业原始数据，进行深度分析。

    你需要完成：
    1. 竞争格局分析：使用表格对比主要公司的市场份额、核心技术、融资阶段
    2. 行业关键洞察：提炼 3-5 个最重要的趋势或发现
    3. SWOT 分析：该行业整体的优势、劣势、机会、威胁
    4. 投资价值评估：给出明确的投资建议（看好/谨慎/观望），并说明理由

    输出要求：结论明确，每个观点都有数据支撑。
    """,
    expected_output="一份包含竞争分析表格、关键洞察列表、SWOT 分析和投资建议的结构化分析报告",
    agent=analyst,
    context=[research_task],  # 依赖研究员的输出
)

# 写作员的任务：撰写最终报告
writing_task = Task(
    description="""
    将分析师的 {industry} 行业分析报告转化为一份面向投资者的专业行业报告。

    报告结构：
    1. 执行摘要（200 字以内，概括核心结论）
    2. 行业概况（市场规模、增长驱动力、技术成熟度）
    3. 竞争分析（用表格呈现主要玩家对比）
    4. 关键趋势与洞察（3-5 个要点）
    5. 投资建议（明确的建议和风险提示）

    写作要求：
    - 语言专业但易读，避免过度使用行业黑话
    - 所有数据标注来源
    - 每个章节有小标题和简要小结
    - 报告总长度 2000-3000 字
    """,
    expected_output="一份完整的行业研究报告，包含执行摘要、行业概况、竞争分析表格、趋势洞察和投资建议，2000-3000 字",
    agent=writer,
    context=[analysis_task],  # 依赖分析师的输出
    output_file="industry_report.md",  # 自动保存为文件
)
```

这段代码有两个关键设计：

**`context` 参数**——这是任务之间传递信息的纽带。`analysis_task` 设置了 `context=[research_task]`，意味着分析师在工作时会自动拿到研究员的输出结果作为上下文。同理，写作员会拿到分析师的输出。

如果不设置 `context`，每个 Agent 只能看到自己的 `description`，拿不到上游的数据。这就像三个员工各干各的，互不通气。在顺序执行（sequential）模式下，CrewAI 默认会把前一个任务的输出传给下一个，但显式设置 `context` 更可靠，尤其是在你将来想调整任务顺序或加入并行分支时。

**`expected_output`**——这不是装饰字段，它会作为 Agent 工作时的"验收标准"注入 prompt。写得越具体，输出质量越稳定。"一份报告"和"一份 2000-3000 字、包含 5 个章节的行业报告"，LLM 理解到的指令精度完全不同。

## Step 5: 组建 Crew 并运行

所有零件都齐了，现在把它们组装起来：

```python
from crewai import Crew, Process

crew = Crew(
    agents=[researcher, analyst, writer],
    tasks=[research_task, analysis_task, writing_task],
    process=Process.sequential,  # 流水线模式：按 tasks 列表顺序依次执行
    verbose=False,   # 关闭详细日志（Windows 控制台 GBK 编码不支持 emoji 输出）
    memory=False,    # 关闭记忆（DeepSeek 不支持 /embeddings 端点，开启会报 404）
    max_rpm=5,       # 每分钟最多 5 次 LLM 调用
)
```

几个配置选择的解释：

**`verbose=False`**——CrewAI 的 verbose 日志会输出大量带 emoji 的调试信息。在 macOS/Linux 终端没问题，但在 Windows 控制台下 GBK 编码无法显示 emoji 字符，会刷出满屏的 `codec can't encode character` 错误。关掉后不影响功能，执行过程可以通过 `step_callback` 记录到文件里。

**`memory=False`**——CrewAI 的记忆功能依赖 Embedding 接口（默认调用 `/embeddings`），但 DeepSeek 的 API 不提供这个端点，开启后会报 404。如果你用的是 OpenAI 或支持 Embeddings 的模型，可以开启 `memory=True`。

运行整个流程：

```python
result = crew.kickoff(inputs={"industry": "量子计算"})

print("=" * 50)
print("最终报告：")
print(result)
```

### 运行时发生了什么

设置 `verbose=True` 后，你能在控制台看到完整的 Agent 协作过程。简化版的日志大概是这样的：

```
# Agent: 行业研究员
## Task: 针对 量子计算 行业进行全面的信息搜集...

# Agent: 行业研究员
## Thought: 我需要搜索量子计算行业的最新市场数据和技术进展
## Using tool: SerperDevTool
## Tool input: {"search_query": "量子计算 市场规模 2025 2026"}
## Tool output: [搜索结果列表...]

# Agent: 行业研究员
## Thought: 我已经搜集到足够的市场数据，现在搜索主要公司的信息
## Using tool: SerperDevTool
## Tool input: {"search_query": "量子计算 主要公司 融资 2025"}
## Tool output: [搜索结果列表...]

# ... 更多搜索步骤 ...

# Agent: 行业研究员
## Final Answer: [研究员的完整输出，约 1800 字]

---

# Agent: 竞争分析师
## Task: 基于研究员提供的 量子计算 行业原始数据...

# Agent: 竞争分析师
## Thought: 研究员提供了丰富的数据，我先整理竞争格局...

# Agent: 竞争分析师
## Final Answer: [分析师的完整输出，包含表格和 SWOT]

---

# Agent: 行业报告撰写员
## Task: 将分析师的 量子计算 行业分析报告转化...

# Agent: 行业报告撰写员
## Thought: 我需要把分析师的结构化数据转化为面向投资者的报告...

# Agent: 行业报告撰写员
## Final Answer: [写作员的完整报告，2000-3000 字]
```

每个 Agent 的工作模式都是 **Thought → Action → Observation** 的循环：先想（我要做什么），再做（调用工具或推理），然后观察（结果是什么），直到它觉得信息足够了，输出 Final Answer。

### Process.sequential vs Process.hierarchical

CrewAI 提供两种编排模式：

**`Process.sequential`（流水线）**——任务按列表顺序一个接一个执行，前一个完成才启动后一个。适合流程清晰、线性依赖的场景，比如我们的研究报告场景。

```
研究员 → 分析师 → 写作员
```

**`Process.hierarchical`（层级管理）**——CrewAI 会自动创建一个"经理 Agent"来分配和协调任务。适合任务之间有复杂依赖、需要动态调度的场景。但开销更大，因为经理 Agent 本身也要消耗 LLM 调用。

```
        经理 Agent
       /    |    \
  研究员  分析师  写作员
```

对于我们的研究报告项目，流水线模式就够了。层级模式更适合类似"软件开发团队"这种需要反复沟通、迭代的场景。

## Step 6: 加入 Harness 约束

跑通一个 Demo 和生产级系统之间差什么？答案是约束。还记得文章 15 讲的 Harness Engineering 吗——给 Agent 系统加上「缰绳」，让它在可控的范围内运行。

CrewAI 内置了几个关键的约束参数，我们来逐个配置：

### max_iterations：迭代上限

防止 Agent 陷入无限循环。比如研究员反复搜索同一个关键词，每次都觉得自己搜集得不够：

```python
researcher = Agent(
    # ... 其他参数同上
    max_iterations=3,  # 最多 3 轮 Thought-Action-Observation 循环
)
```

实测中，研究员搜集一个行业的信息通常需要 3-5 轮搜索。`max_iterations=3` 对 DeepSeek 这种便宜模型来说是个合理的平衡点——既能搜集到足够信息，又不会无限制地消耗 token。

### max_rpm：频率限制

控制整个 Crew 的 LLM 调用频率：

```python
crew = Crew(
    # ... 其他参数
    max_rpm=5,  # 每分钟最多 5 次 LLM 请求
)
```

这个参数对控制成本很重要。DeepSeek-v4-flash 的价格很低（约 ¥1/百万 token），一次完整的研究流程大约消耗 28000 个 token，折合 ¥0.05 左右。但如果 Agent 反复重试或过度搜索，成本还是会快速累积。

### memory：记忆系统

CrewAI 支持短期记忆和长期记忆：

```python
crew = Crew(
    # ... 其他参数
    memory=True,           # 开启短期记忆（Agent 之间共享上下文）
)
```

短期记忆让同一个 Crew 内的 Agent 可以"看到"其他 Agent 的工作成果。长期记忆会跨多次运行持久化。

**但注意**：CrewAI 的记忆功能依赖 Embedding 接口（调用 `/embeddings` 端点）。如果你用的是 DeepSeek，它的 API 不提供这个端点，开启 memory 后会报 404。用 OpenAI 或其他支持 Embeddings 的模型时可以正常开启。

### callback：自定义回调

用于日志、监控和调试：

```python
def step_callback(step_output):
    """每一步执行后的回调"""
    # 记录到日志系统
    with open("crew_log.txt", "a", encoding="utf-8") as f:
        f.write(f"[{__import__('datetime').datetime.now()}] {step_output}\n")

def task_callback(task_output):
    """每个任务完成后的回调"""
    print(f"[任务完成] 耗时信息已记录")

crew = Crew(
    # ... 其他参数
    step_callback=step_callback,
    task_callback=task_callback,
)
```

### 完整的带约束配置

把所有约束合在一起：

```python
from crewai import Crew, Process

crew = Crew(
    agents=[researcher, analyst, writer],
    tasks=[research_task, analysis_task, writing_task],
    process=Process.sequential,
    verbose=False,       # Windows 下关闭以避免 GBK 编码问题
    memory=False,        # DeepSeek 不支持 embeddings 端点
    max_rpm=5,
    step_callback=step_callback,
    share_crew=False,    # 不向 CrewAI 官方分享运行数据
)
```

## 运行效果展示

用"量子计算"作为输入，跑一遍完整流程。以下是实际运行数据：

### 执行统计

| 指标 | 数值 |
|------|------|
| 总执行时间 | 约 90 秒 |
| 研究员搜索次数 | 4-6 次 SerperDevTool 调用 |
| LLM 总调用次数 | 约 9 次（3 个 Agent 各 2-3 次） |
| 总 token 消耗 | ~28,000 tokens |
| 估算成本 | ~¥0.05 (deepseek-v4-flash) |

### 最终报告（节选）

报告的执行摘要部分大致如下（经过整理和精简）：

> **执行摘要**
>
> 全球量子计算市场正处于从实验室走向商业化的关键转折点。2025 年市场规模约 12 亿美元，预计 2030 年将增长至 280 亿美元（CAGR 约 88%）。IBM、Google、IonQ 和 Rigetti 是当前的技术领跑者，中国在量子通信领域具有独特优势。投资窗口正在打开，但短期内（1-2 年）商业化回报仍然有限，建议关注中长期布局机会。

报告的竞争分析表格部分：

| 公司 | 核心技术路线 | 量子比特数 | 融资/市值 | 商业化阶段 |
|------|------------|-----------|----------|-----------|
| IBM | 超导 | 1,121+ | 上市公司 | 云平台服务 |
| Google | 超导 | 70+ (Sycamore) | Alphabet 子公司 | 研究阶段 |
| IonQ | 离子阱 | 36+ | 上市 (IONQ) | 云服务+政府合同 |
| Rigetti | 超导 | 84+ | 上市 (RGTI) | 混合云 |
| 本源量子 | 超导 | 60+ | 未上市 | 政府+企业合作 |

### 输出质量评估

报告整体质量不错，但有几个可以改进的地方：

- **数据时效性**：搜索引擎返回的结果以 2024-2025 年为主，2026 年最新数据较少。可以通过调整搜索关键词（加"2026"）改善。
- **信息重复**：分析师和写作员的输出有部分重叠内容。可以在 Task 的 description 中更明确地划分职责边界。
- **表格格式**：Agent 生成的表格偶尔会出现 Markdown 格式不规范的情况。可以在写作员的 backstory 中强调格式要求。

## 常见问题和调试技巧

搭建完这个 Demo 之后，我踩了一些坑，也总结了一些调试技巧：

### Agent 之间信息丢失

**现象**：分析师的输出里完全没有提到研究员搜集的某个重要数据点。

**原因**：`context` 没有正确设置，或者研究员的输出太长被截断了。

**解法**：
```python
# 确保显式设置 context
analysis_task = Task(
    # ...
    context=[research_task],  # 不要省略这行
)
```

如果研究员输出确实很长（超过 4000 token），考虑让它在 Final Answer 中先做一个结构化摘要，再附上详细数据。

### 输出质量不稳定

**现象**：同样的输入，有时候报告质量很好，有时候很泛。

**原因**：LLM 本身的随机性（temperature）。

**解法**：通过 `LLM` 类调低 temperature：

```python
from crewai import LLM

deepseek_model = LLM(
    model="deepseek-v4-flash",
    temperature=0.3,  # 降低随机性，输出更稳定
)

researcher = Agent(
    # ...
    llm=deepseek_model,
)
```

### 执行时间太长

**现象**：一个 Agent 花了 60 秒以上还在搜索。

**原因**：没有设置迭代上限，Agent 觉得自己搜集的信息不够。

**解法**：

```python
researcher = Agent(
    # ...
    max_iterations=3,    # 最多 3 轮迭代
    max_retry_limit=2,   # 失败最多重试 2 次
)
```

配合 `max_rpm` 限制每分钟请求数，能有效控制总执行时间。实测加上这些约束后，总执行时间从 120-180 秒稳定在 80-100 秒。

### 成本过高

**现象**：跑一次研究报告的成本超出预期。

**原因**：所有 Agent 都在用同一个模型，没有按任务复杂度分配。

**解法**：按任务复杂度分配模型——重活给大模型，轻活给小模型：

```python
from crewai import LLM

deepseek_v4 = LLM(model="deepseek-v4-flash", temperature=0.3)
deepseek_r1 = LLM(model="deepseek-r1", temperature=0.3)  # 推理模型，更便宜

# 研究员：需要深度搜索和分析，用标准模型
researcher = Agent(
    # ...
    llm=deepseek_v4,
)

# 分析师：主要是结构化整理，用推理模型够了
analyst = Agent(
    # ...
    llm=deepseek_r1,
)

# 写作员：需要高质量输出，用标准模型
writer = Agent(
    # ...
    llm=deepseek_v4,
)
```

模型分级的思路是通用的：不管用 OpenAI（gpt-4o + gpt-4o-mini）还是 DeepSeek，都应该让简单任务用便宜的模型，复杂任务用强的模型。

## 完整代码汇总

把上面的所有代码拼在一起，就是一个完整的、可直接运行的脚本：

```python
"""
CrewAI 多 Agent 行业研究系统
运行方式: python crew_ai.py
环境变量: OPENAI_API_KEY, SERPER_API_KEY
"""
import os
from dotenv import load_dotenv
load_dotenv()

import datetime
from crewai import Agent, Task, Crew, Process, LLM
from crewai.tools import BaseTool
from crewai_tools import SerperDevTool
from pydantic import BaseModel, Field


# ============ 自定义工具 ============

class IndustryReportInput(BaseModel):
    industry: str = Field(description="行业名称", json_schema_extra={"example": "人工智能"})

class IndustryReportTool(BaseTool):
    name: str = "industry_report"
    description: str = "根据行业名称，生成该行业的最新报告摘要。输入行业名称，返回该行业的市场规模、技术进展和融资情况等关键数据。"
    args_schema: type[BaseModel] = IndustryReportInput

    def _run(self, industry: str) -> str:
        reports = {
            "量子计算": (
                "1. McKinsey 2025: 全球量子计算市场 2030 年预计达 280 亿美元\n"
                "2. BCG: 量子纠错进入实用化阶段\n"
                "3. PitchBook: 2025 年量子计算融资总额 42 亿美元"
            ),
        }
        return reports.get(industry, f"未找到关于 {industry} 的最新报告摘要。")


# ============ 模型配置 ============

deepseek_model = LLM(
    model="deepseek-v4-flash",
    temperature=0.3
)

# ============ 工具配置 ============

search_tool = SerperDevTool(n_results=3, api_key=os.getenv("SERPER_API_KEY"))
report_tool = IndustryReportTool()


# ============ Agent 配置 ============

researcher = Agent(
    role="行业研究员",
    goal="针对 {industry} 行业，搜集最新的市场数据、技术进展、主要公司和行业趋势",
    backstory="""你是一位资深行业研究员，在麦肯锡和贝恩有超过 10 年的咨询经验。
你擅长从海量信息中快速识别关键数据点，尤其关注市场规模（具体数字）、
增长率（CAGR）、技术成熟度曲线和头部公司的最新动态。
你的输出以事实和数据为主，不做主观判断。""",
    tools=[search_tool, report_tool],
    llm=deepseek_model,
    verbose=True,
    max_iterations=3,
    allow_delegation=False
)

analyst = Agent(
    role="竞争分析师",
    goal="基于研究员提供的原始数据，提炼 {industry} 行业的竞争格局、关键洞察和投资价值",
    backstory="""你是一位严谨的商业分析师，专注于技术行业的竞争分析。
你擅长使用波特五力、SWOT 等分析框架，能从杂乱的数据中提取结构化洞察。
你特别关注：市场份额分布、技术壁垒、融资轮次和估值趋势。
你的输出需要有明确的结论和量化支撑。""",
    llm=deepseek_model,
    max_iterations=3,
    allow_delegation=False,
    verbose=True
)

writer = Agent(
    role="行业报告撰写员",
    goal="将分析师的 {industry} 行业分析转化为一份面向投资者的专业报告",
    backstory="""你是一位资深的金融写作专家，长期为顶级投行撰写行业研究报告。
你的文风专业但不晦涩，擅长用清晰的逻辑和生动的数据讲故事。
你特别注重报告的可读性：执行摘要要一针见血，结论要有明确立场，
风险提示要具体而非泛泛而谈。""",
    llm=deepseek_model,
    max_iterations=3,
    allow_delegation=False,
    verbose=True
)


# ============ 任务定义 ============

research_task = Task(
    description="""
        针对 {industry} 行业进行全面的信息搜集。
        你需要搜集以下信息：
        1. 市场规模（当前值和预测值，带具体数字）
        2. 主要公司（至少 5 家，包括名称、核心业务、融资情况）
        3. 技术趋势（当前主流技术路线和未来方向）
        4. 行业关键事件（最近 12 个月的重要新闻）
        输出要求：以结构化的方式呈现原始数据，每条信息标注来源。
    """,
    expected_output="一份包含市场规模、主要公司、技术趋势和关键事件的结构化数据报告，至少 1500 字",
    agent=researcher,
)

analysis_task = Task(
    description="""
    基于研究员提供的 {industry} 行业原始数据，进行深度分析。
    你需要完成：
    1. 竞争格局分析：表格对比主要公司的市场份额、核心技术、融资阶段
    2. 行业关键洞察：提炼 3-5 个最重要的趋势或发现
    3. SWOT 分析
    4. 投资价值评估：给出明确的投资建议并说明理由
    """,
    expected_output="一份包含竞争格局分析、行业关键洞察、SWOT分析和投资价值评估的深度分析报告",
    agent=analyst,
    context=[research_task]
)

writing_task = Task(
    description="""
    将分析师的 {industry} 行业分析报告转化为一份面向投资者的专业行业报告。
    报告结构：
    1. 执行摘要（200 字以内）
    2. 行业概况
    3. 竞争分析（用表格呈现）
    4. 关键趋势与洞察
    5. 投资建议和风险提示
    """,
    expected_output="一份完整的行业报告，面向投资者，结构清晰，数据充分，约 3000 字",
    agent=writer,
    context=[analysis_task],
    output_file="industry_report.md"
)


# ============ 回调函数 ============

def step_callback(step_output):
    with open("crew_log.txt", "a", encoding="utf-8") as f:
        f.write(f"[{datetime.datetime.now():%H:%M:%S}] {step_output}\n")


# ============ 组建并运行 Crew ============

crew = Crew(
    agents=[researcher, analyst, writer],
    tasks=[research_task, analysis_task, writing_task],
    process=Process.sequential,
    verbose=False,
    memory=False,
    max_rpm=5,
    step_callback=step_callback,
    share_crew=False,
)

if __name__ == "__main__":
    result = crew.kickoff(inputs={"industry": "量子计算"})
    print("\n" + "=" * 50)
    print(result)
```

把这段代码保存为 `crew_ai.py`，配好 `.env` 文件，直接 `python crew_ai.py` 就能跑。运行结束后，`industry_report.md` 里就是完整的行业报告，`crew_log.txt` 里是详细的执行日志。

## 第四阶段总结

到这里，第四阶段的四篇文章全部结束了。我们来回顾一下这一阶段的核心脉络：

**文章 13：多 Agent 架构模式**——我们了解了 Agent 系统不是「一个 LLM 干所有事」。从单 Agent 到多 Agent，从扁平协作到层级管理，每种架构都有它的适用场景。关键不是选最复杂的架构，而是选最合适的。

**文章 14：A2A 通信协议**——Agent 之间怎么说话？Google 的 A2A 协议定义了标准化的 Agent 通信方式，包括 Agent Card（名片）、Task（任务对象）、Message（消息）和 Artifact（产出物）。有了标准协议，不同框架、不同厂商的 Agent 才能互相对话。

**文章 15：Harness Engineering**——从 Demo 到生产，差的不是更多的功能，而是更好的约束。迭代上限、频率限制、输出验证、异常处理——这些"缰绳"决定了你的 Agent 系统在实际运行中是稳定可控还是随时爆炸。

**文章 16：CrewAI 实战**——也就是这篇。把前面三个主题的知识融合在一起，用 CrewAI 框架搭了一个真实的多 Agent 系统。Agent 的角色设计对应文章 13，工具配置呼应文章 07 的 MCP，Harness 约束直接落地文章 15 的理论。

整个第四阶段的核心信息就一句话：**多 Agent 系统的难点不在于让 Agent 变聪明，而在于让多个 Agent 协作得可靠。** 单个 Agent 的能力上限由 LLM 决定，但多 Agent 系统的可靠性由你的工程架构决定。

## 第五阶段预告

接下来的第五阶段，我们会进入三个新的重要主题：**评估、安全与数据飞轮**。

前四个阶段，我们解决了「怎么构建 Agent 系统」的问题。第五阶段要解决的是「怎么让它越来越好」——怎么评估 Agent 的输出质量？怎么防止 prompt 注入和数据泄露？怎么用用户的反馈数据形成正向循环？

这些是把 Agent 系统从 Demo 提升到生产级别的关键能力。一个不能评估、不安全、不会自我改进的 Agent 系统，永远只能停留在 demo 阶段。

## 推荐资源

- [CrewAI 官方文档](https://docs.crewai.com/) — API 参考和入门指南，写得相当清楚
- [CrewAI GitHub](https://github.com/crewAIInc/crewAI) — 源码和 example 目录，看 example 比看文档学得快
- [CrewAI Tools](https://github.com/crewAIInc/crewAI-tools) — 内置工具集的源码，了解 Tool 的设计模式
- [Serper.dev](https://serper.dev) — Google 搜索 API，注册即送 2500 次免费调用，够你调试很长时间了
