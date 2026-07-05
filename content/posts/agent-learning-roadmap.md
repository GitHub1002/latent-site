---
title: "AI Agent 学习路线：从零到生产，每个阶段配一个开源项目"
date: 2026-07-05
draft: false
weight: 100
tags: ["Agent", "LLM"]
summary: "一份系统的 AI Agent 学习路线图，覆盖 Prompt / Context / Loop / Harness 四大工程类型，以及 MCP、A2A、推理模型、评估、安全、数据飞轮等关键概念。5 个阶段，每阶段配一个 GitHub 开源项目实战。"
ShowToc: true
---

2025-2026 年，AI Agent 已经成为大模型落地最热的方向。但 Agent 工程的知识体系也在快速膨胀——从最初的 Prompt Engineering，演化出了 Context Engineering、Loop Engineering、Harness Engineering 四大工程类型，又涌现出 MCP、A2A 等标准化协议。

对于想入门的同学，最大的困惑是：**知识地图到底有多大？我该按什么顺序学？**

这篇文章给你一份**完整的路线图**，分为 5 个阶段，显式覆盖所有关键概念，每个阶段推荐一个 GitHub 开源项目实战。

---

## 路线图总览

![AI Agent 学习路线图](/images/agent-roadmap-v2.svg)

---

## 四大工程类型

在深入各阶段之前，先理解 Agent 工程的四个递进层次。这四种工程类型**按抽象层次从低到高排列**，高一层包含低一层的所有能力：

| 工程类型 | 核心问题 | 抽象层次 | 概念起源 |
|---------|---------|---------|---------|
| **Prompt Engineering** | "这一条指令怎么写？" | 指令级 | 2022 年随 GPT-3 兴起 |
| **Context Engineering** | "给模型看到什么信息？" | 信息级 | 2025 年成为共识 |
| **Loop Engineering** | "自主循环怎么跑？" | 循环级 | 2022 年 ReAct 论文奠基 |
| **Harness Engineering** | "整个脚手架怎么搭？" | 系统级 | 2026 年随生产化需求浮现 |

注意：Loop Engineering 的核心概念（ReAct 2022、Reflexion 2023）比 Context Engineering 这一术语更早出现，但它解决的问题层次（循环控制）在抽象层级上高于 Prompt 和 Context。Harness Engineering 则是最"新"的概念，关注整个系统的环境约束与编排。

它们之间是层层嵌套的关系：Prompt 优化单次指令，Context 构建完整的信息环境，Loop 设计自主执行循环，Harness 编排整个运行架构。

---

## 阶段 1：LLM 基础与 Prompt 工程（2-3 周）

**目标**：理解大模型工作原理，掌握 Prompt 工程核心技巧，能区分普通模型与推理模型的差异。

**覆盖概念**：LLM 原理 · Prompt Engineering · Reasoning Models (o1/R1)

### 需要学什么

- Transformer 架构直觉（不需要手推公式）、Tokenization、Context Window
- Temperature、Top-p、System Prompt 等参数调优
- Zero-shot / Few-shot / Chain-of-Thought / Self-Consistency
- **推理模型**：OpenAI o1/o3、DeepSeek-R1 等模型在回答前会自主进行长链推理（test-time compute），和 CoT Prompt 有本质区别

### 推荐项目：[Prompt-Engineering-Guide](https://github.com/dair-ai/Prompt-Engineering-Guide)

⭐ **48k+ stars** · 最全面的 Prompt 工程学习资源

跟着指南的章节顺序学习，每个技巧都动手试一遍。特别关注 CoT 和 Self-Consistency 章节，以及最新的 Reasoning Models 相关内容。

**学完你应该能**：写出高质量 Prompt，理解推理模型与普通模型的区别，用 API 完成信息提取和摘要任务。

---

## 阶段 2：单 Agent 与 Loop Engineering（2-3 周）

**目标**：理解 Agent 的核心循环，实现能自主调用工具的单 Agent，掌握从 Tool 到 Skill 的封装思维。

**覆盖概念**：Loop Engineering · Function Calling · Tool → Skill 封装 · MCP 协议入门

### 需要学什么

- **Loop Engineering**：Agent 的感知→推理→行动→观察循环，ReAct 和 Reflexion 范式
- Function Calling / Tool Use 的实现方式
- **Tool → Skill 封装**：Tool 是原子操作（如搜索），Skill 是封装好的多步流程（如"搜索+对比+生成报告"）
- 停止条件设计、错误恢复、防止死循环

### 推荐项目：[LangChain](https://github.com/langchain-ai/langchain)

⭐ **100k+ stars** · 最流行的 LLM 应用开发框架

用 `create_react_agent` 几行代码创建一个 ReAct Agent，给它添加搜索、计算器、代码执行等工具，观察 Thought-Action-Observation 循环日志。然后尝试把多个工具组合成一个"技能"（Skill），理解封装的价值。

**学完你应该能**：实现带工具调用和自反思的 Agent，理解 Skill 封装的设计思想。

---

## 阶段 3：Context Engineering 与 RAG（3-4 周）

**目标**：掌握动态上下文构建，实现生产级 RAG 系统，用 MCP 标准化工具连接。

**覆盖概念**：Context Engineering · RAG · 记忆系统 · MCP 协议

### 需要学什么

- **Context Engineering**："给模型看到什么"比"怎么说"更重要——动态构建上下文窗口，管理信息优先级
- RAG 全流程：文档切分、Embedding 选型、向量检索、重排、Prompt 注入
- **高级 RAG**：Multi-hop、Self-RAG、Graph RAG
- **记忆系统**：短期对话记忆、长期知识记忆、记忆压缩策略
- **MCP（Model Context Protocol）**：Anthropic 提出的标准化工具连接协议——Agent 世界的"USB 接口"

### 推荐项目：[LlamaIndex](https://github.com/run-llama/llama_index) + [MCP Servers](https://github.com/modelcontextprotocol/servers)

⭐ LlamaIndex **38k+ stars** · MCP Servers **60k+ stars**

用 LlamaIndex 构建 RAG 系统，索引技术文档。然后写一个 MCP Server（比如连接本地文件系统 + 数据库查询），让你的 Agent 通过标准协议调用外部工具。

**学完你应该能**：构建带记忆和 RAG 的 Agent，用 MCP 标准化工具连接，理解上下文工程的设计原则。

---

## 阶段 4：多 Agent 与 Harness Engineering（3-4 周）

**目标**：设计多 Agent 协作系统，掌握 Harness Engineering 的架构思维。

**覆盖概念**：Multi-Agent · A2A 协议 · Harness Engineering · 角色设计

### 需要学什么

- 多 Agent 架构模式：层级式（Manager-Worker）、辩论式（Debate）、流水线式（Pipeline）
- **A2A（Agent-to-Agent Protocol）**：Google 提出的 Agent 间通信标准。MCP 解决"Agent 连接工具"，A2A 解决"Agent 连接 Agent"
- **Harness Engineering**：设计整个运行环境的约束、反馈循环、工具链编排——"Agents 不难，Harness 难"
- Agent 角色设计与职责边界

### 推荐项目：[CrewAI](https://github.com/crewAIInc/crewAI)

⭐ **28k+ stars** · 最易上手的多 Agent 框架

创建一个"研究团队"：研究员 Agent 搜集资料 → 分析师 Agent 整理数据 → 写作 Agent 撰写报告。尝试层级管理模式（加一个 Manager Agent 做任务分配），观察 Agent 间的消息传递。

**学完你应该能**：设计生产级多 Agent 系统，理解 Harness 的"脚手架"思维。

---

## 阶段 5：生产化——评估、安全与数据飞轮（3-4 周）

**目标**：将 Agent 系统从 Demo 提升到生产级别，建立持续优化的闭环。

**覆盖概念**：Evaluation · Safety & Guardrails · Observability · 数据飞轮 · Fine-tuning

### 需要学什么

- **评估方法论**：任务完成率、工具调用准确率、幻觉率、成本效率怎么定义和度量？Ragas 框架怎么用？
- **安全防护**：Prompt 注入攻击、越狱（Jailbreak）防护、输出验证、PII 脱敏、权限控制
- **可观测性**：追踪每一步推理和工具调用，可视化 trace，成本分析
- **数据飞轮**：Agent 运行日志 → 筛选高质量轨迹 → 构建训练数据 → 微调专用模型 → 替换通用 LLM 降本增效

### 推荐项目：[LangFuse](https://github.com/langfuse/langfuse) + [Ragas](https://github.com/explodinggradients/ragas)

⭐ LangFuse **8k+ stars** · Ragas **13k+ stars**

本地部署 LangFuse，把你之前做的 Agent 接入，观察 trace 日志。用 Ragas 定义评估指标并跑自动化测试。最后尝试从 Agent 日志中构建训练数据，微调一个小模型。

**学完你应该能**：评估 Agent 质量、防御常见攻击、建立数据飞轮实现持续优化。

---

## 进阶方向（选修）

完成以上 5 个阶段后，可根据兴趣深入：

| 方向 | 核心内容 | 推荐项目 |
|------|---------|---------|
| Code Agent | SWE-bench、代码生成与执行 | [OpenHands](https://github.com/All-Hands-AI/OpenHands) |
| Web Agent | 浏览器自动化、网页操作 | [Browser Use](https://github.com/browser-use/browser-use) |
| Multimodal Agent | 视觉+语言+音频融合 | [LLaVA-NeXT](https://github.com/LLaVA-VL/LLaVA-NeXT) |
| Reasoning Agent | MCTS、Tree of Thoughts | [gpt-researcher](https://github.com/assafelovic/gpt-researcher) |

---

## 总结

Agent 工程的知识体系比一年前膨胀了很多，但核心逻辑没变：**从单次对话到自主循环，从单 Agent 到多 Agent 协作，从 Demo 到生产**。

按照 **基础 → Loop → Context → Harness → 生产** 的路线，每个阶段配一个开源项目实战，3-4.5 个月就能建立完整的能力体系。

记住一个原则：**每个阶段都要动手写代码，不要只看文档。** Agent 开发的核心是"调"出来的——调 Prompt、调工具参数、调 Agent 角色定义、调评估指标——只有动手才能培养出直觉。

祝你学习顺利！
