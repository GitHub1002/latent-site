---
title: "05 | Loop Engineering——Agent 的核心循环"
date: 2026-07-06
draft: false
weight: 201
tags: ["Agent", "Loop Engineering"]
summary: "Agent 和普通 LLM 对话的本质区别是什么？答案是循环。这篇文章详解 Agent 的核心执行循环：ReAct 范式、Reflexion 自反思机制，以及停止条件、错误恢复、防止死循环等工程细节。"
ShowToc: true
---

第一阶段的 4 篇文章，我们学了怎么跟 LLM "对话"——写好 Prompt、选好模型、拿到好答案。但对话有一个本质限制：**一问一答，模型不会主动做事**。

Agent 的不同之处在于：它能**自主循环执行任务**。你给它一个目标，它自己规划步骤、调用工具、检查结果、发现错误后重试——直到任务完成或者明确失败。

这就是 **Loop Engineering** 要解决的问题：怎么设计这个自主循环？

![Agent 核心循环](/images/stage2-05-loop-engineering.svg)

---

## Agent 的本质：一个带工具的循环

先看一个最简单的 Agent 执行过程。假设你让它"帮我查一下北京明天的天气，如果气温低于 15 度就提醒我带外套"：

```
[循环开始]

第 1 轮：
  思考：用户想知道北京明天的天气，我需要用天气工具查询
  行动：调用 weather_api("北京", "明天")
  观察：返回结果 {"temp_high": 12, "temp_low": 3, "weather": "多云"}

第 2 轮：
  思考：最高温 12°C，低于 15°C，需要提醒用户带外套
  行动：生成回复
  观察：回复已生成，任务完成

[循环结束]
```

这个"思考→行动→观察"的循环，就是 Agent 的核心架构。每一轮循环，Agent 都在做三件事：

1. **推理（Reasoning）**：根据当前状态决定下一步做什么
2. **行动（Action）**：执行具体操作（调用工具、生成回复等）
3. **观察（Observation）**：获取行动的反馈结果

然后把观察结果带回下一轮推理，如此往复。

> 关键认知：Agent 不是一个"更聪明的聊天机器人"，它是一个**带工具的 while 循环**。这个认知会帮你理解后面所有的 Agent 设计决策。

---

## ReAct 范式：推理与行动的交替

**ReAct**（Reasoning + Acting）是 Agent 领域最重要的范式之一，由 Yao et al. 在 2022 年的论文 *"ReAct: Synergizing Reasoning and Acting in Language Models"* 中提出。

ReAct 的核心思想很简单：让模型在采取行动之前先"说出"自己的推理过程，然后根据推理结果选择行动。

### ReAct 的标准格式

```
Thought: 我需要搜索 Python 3.12 的新特性
Action: search
Action Input: "Python 3.12 new features"
Observation: [搜索结果返回了 5 条相关信息...]

Thought: 搜索结果显示 Python 3.12 主要有 3 个重要变化...
         但我还需要确认性能提升的具体数据
Action: search
Action Input: "Python 3.12 performance benchmarks"
Observation: [性能测试结果...]

Thought: 现在我有了足够的信息来回答用户的问题
Action: final_answer
Action Input: "Python 3.12 的主要新特性包括..."
```

每一轮都包含三个元素：

| 元素 | 作用 | 示例 |
|------|------|------|
| **Thought** | 模型的推理——为什么这么做 | "我需要搜索..." |
| **Action** | 选择什么工具/操作 | search, calculator, code_exec |
| **Observation** | 工具返回的结果 | 搜索结果、计算值、执行输出 |

### ReAct 为什么有效？

对比两种做法：

**不用 ReAct（直接让模型调用工具）**：模型可能在没想清楚的情况下就调用工具，得到结果后又不知道该怎么处理。就像一个新手拿到工具箱却不知道先用什么、后用什么。

**用 ReAct（先推理再行动）**：Thought 步骤强制模型"先想清楚再动手"。模型必须解释为什么选择这个工具、期望得到什么结果、拿到结果后怎么处理。推理链条提升了决策质量。

这和第一篇讲的 CoT（Chain-of-Thought）是一脉相承的——ReAct 本质上就是**把 CoT 嵌入了一个循环结构**。

---

## Reflexion：让 Agent 学会自我反思

ReAct 解决了"推理→行动"的问题，但还有一个场景没覆盖：**Agent 做错了怎么办？**

最简单的做法是：如果行动失败，直接把错误信息当作 Observation 传入下一轮，让模型自己想办法。这在很多情况下够用，但对于复杂任务，模型可能反复犯同一个错误。

**Reflexion**（Shinn et al., 2023）提出了一种更优雅的方案：让 Agent 在行动失败后进行**结构化反思**，把教训记录下来，在后续轮次中避免重复犯错。

### Reflexion 的执行流程

```
[任务] 编写一个 Python 函数，计算列表中所有素数的和

第 1 轮：
  Thought: 我需要写一个判断素数的函数，然后对列表中的素数求和
  Action: code_exec
  Code:
    def sum_primes(lst):
        return sum(x for x in lst if x > 1 and all(x % i for i in range(2, x)))
  Observation: 测试失败——输入 [2,3,4,1,0,-1] 时返回了错误结果（0 和负数应该被排除）

  [Reflexion 步骤]
  反思: 我的素数判断函数对 0、1、负数处理不正确。
       x > 1 这个条件应该已经排除了 0 和 1，
       但 range(2, x) 在 x=2 时返回空列表，all() 对空列表返回 True，
       所以 2 会被正确判断为素数...
       问题可能在 -1：range(2, -1) 也是空列表，all() 返回 True，-1 被误判为素数！
  教训: 需要明确排除负数和 0、1

第 2 轮：
  Thought: 根据上次的反思，我需要先检查 x >= 2 再判断素数
  Action: code_exec
  Code:
    def sum_primes(lst):
        def is_prime(n):
            if n < 2: return False
            return all(n % i for i in range(2, int(n**0.5) + 1))
        return sum(x for x in lst if is_prime(x))
  Observation: 所有测试通过 ✓
```

注意第 1 轮结束时的 **Reflexion 步骤**——这不是简单的"把错误丢回循环"，而是一个**专门的反思环节**：分析失败原因、提取可操作的教训、指导下次行动。

### ReAct vs Reflexion

| | ReAct | Reflexion |
|--|-------|----------|
| **核心循环** | Thought → Action → Observation | Thought → Action → Observation → Reflection |
| **错误处理** | 把错误当 Observation 传回 | 专门的反思步骤，提取教训 |
| **适合场景** | 工具调用为主的任务 | 需要迭代改进的任务（编码、写作） |
| **代价** | 较轻 | 较重（多一次 LLM 调用做反思） |

实际中两者经常组合使用：简单任务用 ReAct，遇到失败时切换到 Reflexion 进行深度反思。

---

## 工程细节：让循环可靠运行

理解了 ReAct 和 Reflexion 的概念后，真正区分"Demo 级 Agent"和"生产级 Agent"的是一系列工程细节。

### 停止条件设计

循环不能无限跑下去。你需要明确的停止条件：

```python
# 停止条件 1: Agent 明确给出了最终答案
if action == "final_answer":
    return result

# 停止条件 2: 达到最大循环次数
if loop_count >= max_iterations:
    return "任务未完成：超过最大循环次数限制"

# 停止条件 3: 超时
if elapsed_time >= max_time:
    return "任务未完成：执行超时"

# 停止条件 4: 重复检测（连续 N 轮没有进展）
if is_stuck_in_loop(recent_observations, threshold=3):
    return "任务未完成：Agent 陷入循环"
```

**实际建议**：max_iterations 一般设为 10～20 次。太少可能完不成复杂任务，太多则可能浪费 token 且增加死循环风险。

### 错误恢复

工具调用失败是常态，Agent 需要能优雅地处理：

```python
try:
    result = execute_tool(action, action_input)
except ToolTimeoutError:
    result = "工具执行超时，请尝试其他方法或简化输入"
except ToolPermissionError:
    result = "没有权限执行此操作，请换一种方式"
except Exception as e:
    result = f"工具执行失败: {str(e)}。请分析原因并重试"
```

关键原则：**永远不要把原始异常直接传给 Agent 的推理环节**。把异常翻译成 Agent 能理解的自然语言提示，让它知道发生了什么以及可以尝试什么。

### 防止死循环

最常见的死循环模式：Agent 反复调用同一个工具、传同样的参数、得到同样的错误。

```python
# 方法 1: 记录历史行动，检测重复
action_history = []

def detect_repetition(new_action):
    if new_action in action_history[-3:]:
        return "你已经在重复同一个操作。请换一种方法，或者承认当前无法完成。"
    return None

# 方法 2: 限制同一工具的使用次数
tool_use_count = defaultdict(int)
MAX_TOOL_USE = 5

def check_tool_limit(tool_name):
    if tool_use_count[tool_name] >= MAX_TOOL_USE:
        return f"{tool_name} 已使用 {MAX_TOOL_USE} 次，请换其他工具"
    return None
```

### 循环日志：Agent 开发最重要的调试工具

每一轮循环都应该留下详细日志。当 Agent 行为不符合预期时，日志是你唯一的调试手段：

```
=== Loop #1 ===
[Thought] 用户想要查天气，我需要调用天气 API
[Action] weather_api(city="北京", date="tomorrow")
[Observation] {"temp": 12, "condition": "多云"}
[Latency] 1.2s | [Tokens] 150 in / 80 out

=== Loop #2 ===
[Thought] 温度 12°C < 15°C，需要提醒用户
[Action] final_answer("明天北京多云，最高温 12°C，建议带外套")
[Observation] 任务完成
[Latency] 0.8s | [Tokens] 200 in / 50 out

=== Summary ===
Total loops: 2 | Total time: 2.0s | Total tokens: 480
```

后面第五阶段的可观测性（Observability）会专门讲怎么系统地追踪和分析这些日志。

---

## 一个完整的 Agent 循环伪代码

把上面的所有内容整合起来：

```python
def run_agent(task, tools, max_iterations=15, timeout=120):
    """一个生产级的 Agent 执行循环"""
    history = []
    start_time = time.time()

    for i in range(max_iterations):
        # 超时检查
        if time.time() - start_time > timeout:
            return {"status": "timeout", "history": history}

        # 1. 推理：让 LLM 根据历史和任务决定下一步
        thought, action, action_input = llm_reason(task, history)

        # 2. 检查是否完成
        if action == "final_answer":
            return {"status": "success", "answer": action_input, "history": history}

        # 3. 检查重复
        if detect_repetition(action, action_input, history):
            history.append({"role": "system", "content": "检测到重复操作，请换方法"})
            continue

        # 4. 执行行动
        try:
            observation = execute_tool(action, action_input, tools)
        except Exception as e:
            observation = f"执行失败: {e}，请分析原因"

        # 5. 记录到历史
        history.append({
            "thought": thought,
            "action": action,
            "action_input": action_input,
            "observation": observation
        })

    return {"status": "max_iterations", "history": history}
```

这段伪代码覆盖了 Agent 循环的所有关键要素：推理、行动、观察、停止条件、重复检测、错误恢复。下一章讲 Function Calling 时，我们会把这个框架变成真正可运行的代码。

---

## 下一步

这篇讲了 Agent 的核心循环——从 ReAct 的基础范式到 Reflexion 的自反思，再到停止条件、错误恢复等工程细节。

下一篇 **06 | Function Calling** 会深入讲"行动"这个环节——LLM 怎么调用外部工具？工具怎么定义？参数怎么传递？这是让 Agent 从"只能推理"进化为"能真正做事"的关键一步。

---

## 推荐资源

- [ReAct 论文](https://arxiv.org/abs/2210.03629) — Yao et al., "ReAct: Synergizing Reasoning and Acting in Language Models" (2022)
- [Reflexion 论文](https://arxiv.org/abs/2303.11366) — Shinn et al., "Reflexion: Language Agents with Verbal Reinforcement Learning" (2023)
- [LangGraph Agent 教程](https://langchain-ai.github.io/langgraph/) — LangChain 团队的 Agent 循环框架
