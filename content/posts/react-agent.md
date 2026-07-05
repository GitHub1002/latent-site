---
title: "从零构建 ReAct Agent：工具调用与多步推理"
date: 2026-07-04
draft: false
tags: ["Agent", "LLM"]
summary: "实现一个 ReAct 风格的 Agent，支持搜索、计算器、代码执行等工具。覆盖 Thought-Action-Observation 循环、Prompt 工程设计和错误恢复策略。"
ShowToc: true
---

ReAct（Reasoning + Acting）是一种让 LLM 交替进行推理和执行动作的范式。这篇文章从零开始搭建一个可用的 Agent。

## 什么是 ReAct？

核心思想：不直接生成最终答案，而是生成一条 **Thought → Action → Observation** 链。每个 Thought 推理当前状态，每个 Action 调用工具，每个 Observation 提供新信息。

## Thought-Action-Observation 循环

```
问题：东京人口乘以 3 是多少？

Thought 1: 我需要先查找东京人口，然后乘以 3。让我搜索一下。
Action 1: search("东京人口 2024")
Observation 1: 东京都人口约 1400 万。

Thought 2: 现在我有人口数据（14,000,000），需要乘以 3。
Action 2: calculate("14000000 * 3")
Observation 2: 42,000,000

Thought 3: 答案已得出。东京人口 × 3 = 4200 万。
Answer: 42,000,000
```

## 工具注册

每个工具定义名称、描述和参数 schema：

```python
tools = {
    "search": {
        "description": "搜索互联网获取信息",
        "parameters": {"query": "string - 搜索关键词"},
        "fn": web_search
    },
    "calculate": {
        "description": "计算数学表达式",
        "parameters": {"expression": "string - 数学表达式"},
        "fn": safe_eval
    },
    "code_execute": {
        "description": "执行 Python 代码片段",
        "parameters": {"code": "string - Python 代码"},
        "fn": run_python
    }
}
```

## Prompt 设计

系统 prompt 是 Agent 表现的关键。它需要清晰定义工具格式并提供良好的推理链示例。一个设计良好的 prompt 在实践中可以将幻觉动作减少约 40%。

```python
SYSTEM_PROMPT = """你是一个能够使用工具解决问题的 Agent。

你可以使用以下工具：
{tool_descriptions}

请按以下格式回答：

Thought: [你的推理过程]
Action: [工具名称]
Action Input: [工具参数]
Observation: [工具返回结果]
... (重复 Thought/Action/Observation)
Thought: 我知道最终答案了
Final Answer: [最终答案]

开始解决问题。
问题：{question}"""
```

## 核心循环实现

```python
def react_loop(question: str, tools: dict, llm, max_steps: int = 6):
    """ReAct 推理循环"""
    history = []
    
    for step in range(max_steps):
        # 构建 prompt
        prompt = build_prompt(question, tools, history)
        
        # LLM 生成推理
        response = llm.generate(prompt)
        
        # 解析 Thought / Action / Final Answer
        parsed = parse_response(response)
        
        if parsed.get("final_answer"):
            return parsed["final_answer"]
        
        # 执行工具
        tool_name = parsed["action"]
        tool_input = parsed["action_input"]
        
        if tool_name in tools:
            try:
                observation = tools[tool_name]["fn"](tool_input)
            except Exception as e:
                observation = f"Error: {str(e)}"
        else:
            observation = f"Unknown tool: {tool_name}"
        
        # 记录历史
        history.append({
            "thought": parsed["thought"],
            "action": tool_name,
            "action_input": tool_input,
            "observation": observation
        })
    
    return "达到最大步数限制，未能得出答案"
```

## 错误恢复

当工具调用失败时，Agent 应将错误信息作为 Observation 接收并推理如何恢复——而不是盲目重试。这正是推理步骤的价值所在：模型可以分析**为什么**失败了，然后尝试不同的方法。

例如搜索超时：

```
Observation: Error: search timeout after 10s

Thought 2: 搜索超时了。让我换一种方式，也许可以直接用已有知识回答，
           或者用更简短的搜索词重试。
Action 2: search("东京 人口")
```
