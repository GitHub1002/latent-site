---
title: "AI Agent 学习路线：从零到生产，每个阶段配一个开源项目"
date: 2026-07-05
draft: false
tags: ["Agent", "LLM"]
summary: "一份系统的 AI Agent 学习路线图，分为 5 个阶段：LLM 基础 → Prompt 工程 → 单 Agent 开发 → 多 Agent 系统 → 生产部署。每个阶段推荐一个 GitHub 开源项目作为实战练手。"
ShowToc: true
---

2025-2026 年，AI Agent 已经成为大模型落地最热的方向。从简单的"问答机器人"到能自主调用工具、协作完成任务的智能体，Agent 的能力边界在快速扩展。

但对于想入门的同学来说，最大的困惑是：**我该从哪开始？学什么？用什么项目练手？**

这篇文章给你一条清晰的学习路线，分为 5 个阶段，每个阶段推荐一个 GitHub 开源项目。不贪多，一步步来。

---

## 路线图总览

![AI Agent 学习路线图](/images/agent-roadmap.svg)

```
阶段 1：LLM 基础与 Prompt 工程
    ↓
阶段 2：单 Agent 开发（ReAct、工具调用）
    ↓
阶段 3：Agent 框架实战（LangChain / LlamaIndex）
    ↓
阶段 4：多 Agent 系统（CrewAI / LangGraph）
    ↓
阶段 5：生产部署与评估（LangFuse / Ragas）
```

每个阶段建议花 2-4 周，根据你的基础灵活调整。

---

## 阶段 1：LLM 基础与 Prompt 工程

**目标**：理解大模型的工作原理，掌握 Prompt 设计的基本技巧。

### 需要学什么

- Transformer 架构的基本概念（不需要手推公式，理解 Attention 的作用即可）
- Tokenization、Context Window、Temperature、Top-p 等核心参数
- Prompt Engineering 的基本模式：Zero-shot、Few-shot、Chain-of-Thought
- 学会调用主流 LLM API（OpenAI、DeepSeek、Claude）

### 推荐项目：[Prompt-Engineering-Guide](https://github.com/dair-ai/Prompt-Engineering-Guide)

⭐ **48k+ stars** · 最全面的 Prompt 工程学习资源

这个项目是一个系统化的 Prompt 工程指南，覆盖了从基础技巧到高级策略（CoT、Self-Consistency、Tree of Thoughts 等），配有丰富的示例和论文引用。有中文翻译版本。

**怎么用它学习**：

1. 跟着指南的章节顺序学习，每个技巧都动手试一遍
2. 用 DeepSeek 或 OpenAI 的 API，把指南里的示例自己跑一遍
3. 尝试用 Few-shot + CoT 组合解决一个实际问题（比如让 LLM 做结构化信息提取）

**学完你应该能**：写出高质量的 System Prompt，理解为什么有些 prompt 效果好、有些效果差。

---

## 阶段 2：单 Agent 开发（ReAct、工具调用）

**目标**：理解 Agent 的核心循环，实现一个能调用工具的单 Agent。

### 需要学什么

- Agent 的核心循环：**感知 → 推理 → 行动 → 观察**
- ReAct（Reasoning + Acting）范式
- Function Calling / Tool Use 的实现方式
- 错误恢复和重试策略

### 推荐项目：[LangChain](https://github.com/langchain-ai/langchain)

⭐ **100k+ stars** · 最流行的 LLM 应用开发框架

虽然 LangChain 是一个完整的框架，但它的 **Agent 模块** 是学习单 Agent 开发的最佳起点。特别是 `create_react_agent` 函数，几行代码就能创建一个 ReAct Agent。

**怎么用它学习**：

1. 先不看 LangChain 的高级功能，只用 `ChatOpenAI` + `create_react_agent` 创建一个简单 Agent
2. 给 Agent 添加 2-3 个工具（搜索、计算器、Python 代码执行）
3. 观察 Agent 的 Thought-Action-Observation 循环日志，理解它的决策过程
4. 尝试自己写一个工具（比如查询天气、读取本地文件）

```python
# 一个最简 ReAct Agent 示例
from langchain_openai import ChatOpenAI
from langchain.agents import create_react_agent, AgentExecutor
from langchain.tools import tool

@tool
def multiply(a: int, b: int) -> int:
    """两个数相乘"""
    return a * b

llm = ChatOpenAI(model="gpt-4o-mini")
agent = create_react_agent(llm, tools=[multiply], prompt=...)
executor = AgentExecutor(agent=agent, tools=[multiply], verbose=True)
executor.invoke({"input": "23 乘以 17 是多少？"})
```

**学完你应该能**：实现一个能调用多个工具的 Agent，理解 Agent 是如何"思考"和"行动"的。

---

## 阶段 3：Agent 框架实战（记忆、RAG、规划）

**目标**：掌握 Agent 的高级能力——记忆管理、RAG 集成、任务规划。

### 需要学什么

- Agent 记忆：短期记忆（对话历史）、长期记忆（向量存储）、记忆压缩
- RAG（检索增强生成）：文档切分、Embedding、向量检索
- 任务规划：将复杂任务分解为子任务

### 推荐项目：[LlamaIndex](https://github.com/run-llama/llama_index)

⭐ **38k+ stars** · 专注于 RAG 和数据连接的框架

相比 LangChain 的"大而全"，LlamaIndex 在 **RAG 和数据索引** 方面做得更深入。它的 `AgentWorkflow` 模块可以创建带有 RAG 能力的 Agent。

**怎么用它学习**：

1. 用 LlamaIndex 构建一个 RAG 系统，索引一批技术文档
2. 创建一个 Agent，让它既能搜索互联网，又能查询你的本地知识库
3. 给 Agent 添加记忆功能——记住用户的偏好和历史对话
4. 尝试实现一个"研究型 Agent"：给一个论文主题，自动搜索、阅读、总结

**学完你应该能**：构建带有知识库和记忆的 Agent，处理更复杂的任务场景。

---

## 阶段 4：多 Agent 系统

**目标**：学习如何让多个 Agent 协作完成复杂任务。

### 需要学什么

- 多 Agent 架构模式：层级式（Manager-Worker）、辩论式（Debate）、流水线式（Pipeline）
- Agent 间通信：消息传递、共享状态、任务委托
- 角色设计：为不同 Agent 定义清晰的职责和能力边界

### 推荐项目：[CrewAI](https://github.com/crewAIInc/crewAI)

⭐ **28k+ stars** · 最易上手的多 Agent 框架

CrewAI 的设计理念是"像组建团队一样组建 Agent"。你定义 Agent 的角色（Role）、目标（Goal）和背景（Backstory），然后定义它们如何协作。API 非常直观。

**怎么用它学习**：

1. 创建一个简单的"写作团队"：研究员 Agent 搜集资料 → 写作 Agent 撰写文章 → 编辑 Agent 审核修改
2. 尝试不同协作模式：顺序执行 vs 层级管理（加一个 Manager Agent）
3. 构建一个"代码审查团队"：代码分析 Agent → 安全审查 Agent → 报告生成 Agent
4. 观察 Agent 之间的消息传递，理解任务如何分解和流转

```python
from crewai import Agent, Task, Crew

researcher = Agent(
    role="技术研究员",
    goal="搜集关于 AI Agent 的最新资料",
    backstory="你是一位资深技术研究员，擅长快速检索和整理信息",
    tools=[search_tool]
)

writer = Agent(
    role="技术作者",
    goal="将研究资料整理成易读的技术文章",
    backstory="你是一位经验丰富的技术博主，擅长深入浅出地解释复杂概念"
)

research_task = Task(description="研究 AI Agent 的最新趋势", agent=researcher)
write_task = Task(description="基于研究结果写一篇博客", agent=writer)

crew = Crew(agents=[researcher, writer], tasks=[research_task, write_task])
result = crew.kickoff()
```

**学完你应该能**：设计和实现多 Agent 协作系统，理解不同架构模式的适用场景。

---

## 阶段 5：生产部署与评估

**目标**：学会将 Agent 系统部署到生产环境，并持续监控和优化。

### 需要学什么

- Agent 可观测性：追踪每一步推理和工具调用
- 评估指标：任务完成率、工具调用准确率、响应延迟、成本
- 安全性：Prompt 注入防护、工具权限控制、数据隐私
- 部署模式：Serverless、容器化、API 网关

### 推荐项目：[LangFuse](https://github.com/langfuse/langfuse)

⭐ **8k+ stars** · 开源的 LLM 可观测性平台

LangFuse 是 LangSmith 的开源替代品，可以追踪 Agent 的每一步操作（LLM 调用、工具执行、RAG 检索），提供可视化的 trace 视图、成本分析、延迟统计。

**怎么用它学习**：

1. 本地部署 LangFuse（Docker 一键启动）
2. 把你之前做的 Agent 接入 LangFuse，观察 trace 日志
3. 分析 Agent 的"决策路径"：它为什么选择这个工具？在哪一步出了错？
4. 设置评估指标：自动统计任务成功率、平均耗时、token 消耗
5. 尝试 A/B 测试：换一个 Prompt 或模型，对比效果差异

**学完你应该能**：将 Agent 系统从"Demo 级别"提升到"生产级别"，具备持续优化的能力。

---

## 进阶方向

完成以上 5 个阶段后，你已经具备了 Agent 开发的核心能力。接下来可以根据兴趣深入：

| 方向 | 关键词 | 推荐关注 |
|------|--------|---------|
| MCP 协议 | Model Context Protocol | [modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers) |
| Agent 安全 | Prompt Injection、Guardrails | [NeMo Guardrails](https://github.com/NVIDIA/NeMo-Guardrails) |
| 代码 Agent | Code Interpreter、SWE | [OpenDevin](https://github.com/All-Hands-AI/OpenHands) |
| Web Agent | 浏览器自动化 | [Browser Use](https://github.com/browser-use/browser-use) |
| Agent 评估 | 基准测试 | [AgentBench](https://github.com/THUDM/AgentBench) |

---

## 总结

Agent 学习不需要一口吃成胖子。按照 **基础 → 单 Agent → 框架 → 多 Agent → 生产** 的路线，每个阶段配一个开源项目实战，3-6 个月就能建立完整的能力体系。

记住一个原则：**每个阶段都要动手写代码，不要只看文档。** Agent 开发的核心是"调"出来的——调 Prompt、调工具参数、调 Agent 角色定义——只有动手才能培养出直觉。

祝你学习顺利！
