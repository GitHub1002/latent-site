---
title: "13 | 多 Agent 架构——为什么一个 Agent 不够用？"
date: 2026-07-08
draft: false
weight: 401
tags: ["Agent", "Multi-Agent"]
summary: "单个 Agent 面对复杂任务时容易出错、上下文爆炸、缺乏专业深度。这篇文章系统讲解 3 种主流多 Agent 架构模式（层级式/辩论式/流水线式），以及 Agent 角色设计原则和职责边界划分。"
ShowToc: true
---

前三阶段我们分别学了 Prompt 工程、单 Agent 循环、RAG 检索。这些能力已经能让 Agent 独立完成不少任务了。但当你试图用它做一件真正复杂的事——比如"调研一个行业、撰写分析报告、同时生成 PPT"——你会发现：**单个 Agent 越来越力不从心。**

上下文越来越长，决策越来越混乱，某个环节出错就全盘崩溃。这就是为什么我们需要**多个 Agent 协作**。

这篇文章讲多 Agent 架构的 3 种核心模式，以及如何设计好每个 Agent 的角色和职责边界。

![多 Agent 架构模式](/images/stage4-13-multi-agent.svg)

---

## 单 Agent 的 3 个瓶颈

在拆解多 Agent 之前，先看看单 Agent 为什么在复杂任务上表现不好。

### 瓶颈 1：上下文窗口爆炸

一个 Agent 要完成"调研+分析+写报告+做 PPT"，它的循环可能要跑 20~30 轮。每轮的 Thought/Action/Observation 都在占用上下文，到了第 15 轮之后，上下文已经用了 60K+ tokens——模型留给"思考和回答"的空间越来越少。

```
128K 窗口（单 Agent 独享）
├── System Prompt:       500 tokens
├── 历史循环 (20 轮):    80,000 tokens  ← 几乎占满
├── RAG 检索结果:        30,000 tokens
├── 用户指令:            300 tokens
└── 留给模型输出:        ~17,000 tokens  ← 不够写一份完整报告
```

### 瓶颈 2：缺乏专业深度

一个通用 Agent 什么都能做一点，但什么都做不精。让它同时扮演"研究员""分析师""设计师"，每个角色的专业度都大打折扣。

类比：你让一个人同时当厨师、服务员和收银员——菜品的质量可想而知。

### 瓶颈 3：容错性差

单 Agent 系统中某个工具调用失败了——比如网络超时、API 报错，整个循环就中断了。它没有"这个 Agent 挂了换一个顶上"的容错能力。

**多 Agent 的核心优势是分工+冗余+专业化。**

---

## 3 种多 Agent 架构模式

理解了单 Agent 的瓶颈，接下来看怎么"拆"。业界主流的多 Agent 设计有 3 种模式：

### 模式 1：层级式（Hierarchical）——经理+工人

最常见的架构。一个**Manager Agent** 负责任务拆解和分配，多个 **Worker Agent** 负责具体执行。

```
[Manager Agent]
  ├── 拆解任务: "调研量子计算行业"
  ├── 分配子任务:
  │     ├── Worker A: "搜索量子计算市场规模"
  │     ├── Worker B: "分析主要玩家和竞争格局"
  │     └── Worker C: "总结技术趋势和挑战"
  ├── 收集结果 → 综合生成最终报告
```

**优点**：职责清晰，Manager 掌控全局，工人专注执行。
**缺点**：Manager 是单点瓶颈——如果它的拆解能力差，整个流程就受影响。
**适用场景**：有明确分工的任务（调研报告、软件开发、客服系统）。

### 模式 2：辩论式（Debate）——多 Agent 互相校验

让多个 Agent 对同一个问题给出不同观点，然后互相质疑、辩论，最终达成共识。

```
Agent A (正方): "Transformer 架构在未来 3 年内不会被取代"
Agent B (反方): "Mamba 等线性架构已经在长序列上展现出优势..."
Agent A: "但 Transformer 的生态优势（工具链、预训练模型）..."
Agent B: "生态可以迁移，架构的数学效率才是核心..."

[Judge Agent] 综合双方论点，给出最终结论
```

**优点**：通过对抗性讨论发现盲点，提升决策质量。
**缺点**：token 消耗大（每个 Agent 都要看完整辩论），延迟高。
**适用场景**：需要高质量决策的场景（投资分析、技术选型、策略制定）。

### 模式 3：流水线式（Pipeline）——流水线接力

把复杂任务拆成线性步骤，每个 Agent 只负责一步，上一步的输出作为下一步的输入。

```
[Agent 1: 搜集] → 原始资料
     ↓
[Agent 2: 清洗] → 结构化数据
     ↓
[Agent 3: 分析] → 洞察和结论
     ↓
[Agent 4: 撰写] → 最终报告
```

**优点**：每个 Agent 职责极简，容易调试和优化。
**缺点**：延迟叠加（总时间 = 各步之和），某一步出错就中断。
**适用场景**：流程固定的批量任务（数据处理、内容生产、代码审查）。

### 3 种模式对比

| | 层级式 | 辩论式 | 流水线式 |
|--|--------|--------|----------|
| **控制方式** | Manager 集中调度 | Agent 平等对抗 | 线性传递 |
| **适合任务** | 可并行拆解的 | 需要深度决策的 | 步骤固定的 |
| **复杂度** | 中 | 高 | 低 |
| **容错性** | Manager 是单点 | 较健壮 | 链条脆弱 |
| **token 成本** | 中 | 高 | 低 |

---

## Agent 角色设计：不是越多越好

知道了 3 种模式后，下一个问题是：**该设几个 Agent？每个 Agent 负责什么？**

常见误区是：多不等于好。 盲目个关键原则是Agent**3 个原则**：

### 原则 1：单一职责

每个 Agent 只只做一类事，不要搞"全能 Agent"：

```python
# ❌ 差的 Agent 定义
agent_research = Agent(
    role="研究员",
    goal="搜索信息、分析数据、生成报告、制作图表、发送邮件"
)

# ✅ 好的 Agent 定义
search_agent = Agent(role="搜索员", goal="搜索并返回相关资料")
analyst_agent = Agent(role="分析师", goal="分析数据并提取洞察")
writer_agent = Agent(role="写手", goal="撰写结构化报告")
```

### 原则 2：明确的输入输出

每个 Agent 要知道自己"从谁那里接收什么"、"把结果交给谁"：

```python
# Agent 2 的定义（清晰的上下游关系）
analyst_agent = Agent(
    role="数据分析师",
    goal="接收搜索员返回的原始资料，提取关键数据点并生成结构化分析",
    backstory="你是一位资深数据分析师，擅长从杂乱信息中提取核心洞察",
    inputs_from=["search_agent"],   # 从搜索员接收
    outputs_to=["writer_agent"],    # 输出给写手
)
```

### 原则 3：适度冗余

关键角色可以设"备份"——避免单点故障：

```python
# 主搜索 Agent + 备用搜索 Agent
primary_search = Agent(role="主搜索员", tools=[search_web, search_academic])
fallback_search = Agent(role="备用搜索员", tools=[search_news, search_social])

# 如果主搜索 Agent 连续失败 2 次，切换到备用
```

---

## 职责边界：谁做什么、谁不做什么

多 Agent 系统最常见的问题是**职责重叠**——两个 Agent 抢着做同一件事，或者都以为对方会做而没人做。

解决方法是用**显式的职责边界矩阵**：

| Agent | 可以做 | 不可以做 |
|-------|--------|----------|
| 搜索员 | 搜索互联网、抓取网页 | 分析数据、撰写报告 |
| 分析师 | 处理数据、生成图表 | 搜索新信息、修改报告 |
| 写手 | 撰写报告、排版 | 做新分析、搜索信息 |
| Manager | 拆解任务、分配工作 | 直接执行任何具体步骤 |

在代码中，这通常通过**工具限制**来实现——

```python
# 搜索员只有搜索工具
search_agent = Agent(
    role="搜索员",
    tools=[search_web, fetch_url]  # 没有 write_file、calculator
)

# 分析师只有分析工具
analyst_agent = Agent(
    role="分析师",
    tools=[calculator, run_python, create_chart]  # 没有 search_web
)

# 写手只有写作工具
writer_agent = Agent(
    role="写手",
    tools=[write_file, format_markdown]  # 没有搜索和分析工具
)
```

每个 Agent 只能用自己的工具，自己职责范围内的事**，无法越界**。这---

  这 这从机制上防止了职责冲突。

---

## 通信协议：Agent 之间怎么说话？

多 Agent 协作的另一个关键问题是**通信**。常见的方式有 3 种：

### 1. 共享黑板（Blackboard）

所有 Agent 都能看到同一个"黑板"（共享内存/数据库），谁都可以读、谁都可以写。

```python
# 共享状态
blackboard = {
    "raw_data": None,       # 搜索员写入
    "analysis": None,       # 分析师写入
    "report": None,         # 写手写入
    "status": "searching"   # Manager 更新
}

# 搜索员完成后
blackboard["raw_data"] = search_results
blackboard["status"] = "analyzing"

# 分析师检测到状态变化，开始工作
if blackboard["status"] == "analyzing":
    analysis = analyze(blackboard["raw_data"])
    blackboard["analysis"] = analysis
    blackboard["status"] = "writing"
```

### 2. 消息传递（Message Passing）

Agent 之间直接发消息（类似函数调用）：

```python
# 搜索员完成后，发消息给分析师
message = {
    "from": "search_agent",
    "to": "analyst_agent",
    "type": "data_ready",
    "payload": search_results
}
message_bus.send(message)

# 分析师监听消息
@analyst_agent.on_message("data_ready")
def handle_data(msg):
    analysis = analyze(msg.payload)
    ...
```

### 3. Google A2A 协议

2025 年 Google 提出的标准化协议，专门为 Agent 间通信设计。下一章会详细讲。

---

## 下一步

这篇讲了多 Agent 的 3 种架构模式和角色设计原则。但还有一个关键问题没解决：**怎么让这些 Agent 标准化地互相通信？**

下一篇讲 **A2A（Agent-to-Agent Protocol）**——Google 为解决这个问题提出的标准协议。它和 MCP（工具连接）互补，共同构成 Agent 系统的"连接层"。

---

## 推荐资源

- [CrewAI 文档](https://docs.crewai.com/) — 最易上手的多 Agent 框架
- [AutoGen 文档](https://microsoft.github.io/autogen/) — 微软的多 Agent 框架
- [Google A2A 协议规范](https://github.com/google/A2A) — Agent 间通信标准
