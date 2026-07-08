---
title: "07 | 从 Tool 到 Skill——封装思维与 MCP 协议"
date: 2026-07-06
draft: false
weight: 203
tags: ["Agent", "MCP", "Tool Use", "Skill"]
summary: "Tool 是原子操作，Skill 是封装好的多步流程。这篇文章讲清楚从 Tool 到 Skill 的设计思维，以及 Anthropic 提出的 MCP 协议如何让工具连接变得标准化和即插即用。"
ShowToc: true
---
上一篇讲了 Function Calling——LLM 怎么调用单个工具。但实际场景中，完成一个任务往往需要**多个工具协作**：先搜索、再分析、再生成报告。如果每次都要 LLM 自己编排这些步骤，既浪费 token 又容易出错。

这就引出了两个问题：

1. 能不能把多个工具**封装成一个更高层的"技能"**，让 LLM 一次调用就完成多步操作？
2. 当工具越来越多时，怎么**标准化地管理**这些工具连接？

前者是 **Skill 封装思维**，后者是 **MCP 协议**要解决的问题。

![Tool → Skill → MCP 三层架构](/images/stage2-07-tool-skill-mcp.svg)

---

## Tool vs Skill：一个类比

先看一个生活中的例子。想象你是一个餐厅经理：

- **Tool（工具）**：切菜刀、炒锅、烤箱、搅拌机——每个都是独立的、原子化的操作
- **Skill（技能）**：做一道"红烧肉"——涉及切肉、焯水、上色、炖煮等多个步骤的有序组合

一个新手厨师需要逐步执行每个步骤（相当于 LLM 每轮循环选一个 Tool）。一个老手厨师直接说"做红烧肉"就够了（相当于调用一个 Skill）。

在 Agent 开发中：

```
Tool 层级（原子操作）:
  - search_web(query)        → 搜索网页
  - fetch_url(url)           → 抓取网页内容
  - summarize_text(text)     → 生成摘要
  - write_file(path, content) → 写入文件

Skill 层级（组合流程）:
  - research_report(topic)   → 搜索 + 抓取 + 分析 + 生成报告
    内部调用了: search_web → fetch_url (x N) → summarize_text → write_file
```

### 为什么要封装 Skill？

|                          | 只用 Tool                 | 封装 Skill               |
| ------------------------ | ------------------------- | ------------------------ |
| **Agent 循环次数** | 每步一个循环，5 步 = 5 轮 | 1 次调用完成             |
| **Token 消耗**     | 每轮都要推理，总消耗大    | 只在 Skill 内部消耗      |
| **出错概率**       | LLM 每步都可能选错        | 流程固定，中间步骤不出错 |
| **可复用性**       | 每个任务重新编排          | 同类任务复用同一个 Skill |

核心思想：**把确定性的多步流程固定下来，只把需要"判断"的环节留给 LLM。**

---

## 如何设计一个好的 Skill？

### 原则 1：Skill 内部是确定性流程

```python
class ResearchSkill:
    """搜索一个主题并生成研究报告"""

    def execute(self, topic: str) -> dict:
        # Step 1: 搜索（确定性调用）
        urls = search_web(topic, num_results=5)

        # Step 2: 抓取内容（确定性调用）
        contents = [fetch_url(url) for url in urls]

        # Step 3: 分析和总结（需要 LLM 判断）
        summary = llm_summarize(contents)

        # Step 4: 生成报告（需要 LLM 判断）
        report = llm_generate_report(topic, summary)

        return {"urls": urls, "summary": summary, "report": report}
```

注意 Step 1-2 是确定性的——搜索 5 个结果、逐个抓取，不需要 LLM 来决定。Step 3-4 才需要 LLM 的智能。

### 原则 2：Skill 对外暴露简单接口

Agent 只需要知道"给我一个主题，我还你一份报告"，不需要关心内部用了几步、用了哪些工具。

```python
# Agent 视角：只看到一个工具
skill_tool = {
    "name": "research_report",
    "description": "对指定主题进行深度研究，搜索多个来源并生成结构化报告",
    "parameters": {
        "type": "object",
        "properties": {
            "topic": {"type": "string", "description": "研究主题"},
            "depth": {"type": "string", "enum": ["quick", "standard", "deep"],
                      "description": "研究深度，默认 standard"}
        },
        "required": ["topic"]
    }
}

# 执行时自动走完整流程
def execute_research_report(topic, depth="standard"):
    num_sources = {"quick": 3, "standard": 5, "deep": 10}[depth]
    return ResearchSkill().execute(topic, num_sources=num_sources)
```

### 原则 3：Skill 可以嵌套

```python
# 一个更高层的 Skill：竞品分析
class CompetitorAnalysisSkill:
    def execute(self, product_name: str):
        # 调用已有的 ResearchSkill
        market = ResearchSkill().execute(f"{product_name} market size")
        competitors = ResearchSkill().execute(f"{product_name} competitors")
        pricing = ResearchSkill().execute(f"{product_name} pricing models")

        # 综合分析
        return llm_competitive_analysis(market, competitors, pricing)
```

这就是**封装的力量**：高层 Skill 复用低层 Skill，像搭积木一样构建复杂能力。

---

## MCP：Agent 世界的 USB 接口

理解了 Tool 和 Skill 的设计之后，还有一个工程问题：**当你的 Agent 需要连接几十个外部服务时，怎么管理？**

每个服务都有自己的认证方式、API 格式、错误处理。如果每个都手写适配代码，Agent 项目会迅速变得不可维护。

**MCP（Model Context Protocol）** 是 Anthropic 在 2024 年底提出的标准化协议，目标是让 Agent 连接外部工具变得像 USB 一样即插即用。

### MCP 解决了什么问题？

在 MCP 之前，每个 Agent 框架都有自己的工具连接方式：

```
M 个 Agent 框架 × N 个外部服务 = M × N 个适配器

LangChain + GitHub API  → 一个适配器
LangChain + Slack API   → 一个适配器
CrewAI + GitHub API     → 又一个适配器
CrewAI + Slack API      → 又一个适配器
...
```

MCP 通过定义统一协议，把 M×N 简化为 M+N：

```
每个 Agent 框架实现一个 MCP Client（M 个）
每个外部服务实现一个 MCP Server（N 个）
总共只需要 M + N 个实现
```

### MCP 的架构

```
┌──────────────────────────────────────────────┐
│                MCP Host (你的 Agent)           │
│                                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │MCP Client│  │MCP Client│  │MCP Client│    │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘    │
│       │              │              │          │
└───────┼──────────────┼──────────────┼──────────┘
        │              │              │
   ┌────┴────┐   ┌────┴────┐   ┌────┴────┐
   │MCP Server│   │MCP Server│   │MCP Server│
   │(GitHub)  │   │(Slack)   │   │(数据库)  │
   └──────────┘   └──────────┘   └──────────┘
```

三个核心角色：

- **Host**：你的 Agent 应用，管理多个 MCP 连接
- **Client**：Host 中的一个连接实例，对应一个 Server
- **Server**：提供具体能力的服务端（如 GitHub、Slack、数据库）

### MCP Server 提供什么？

一个 MCP Server 可以暴露三种能力：

| 能力                | 说明             | 示例                       |
| ------------------- | ---------------- | -------------------------- |
| **Tools**     | 可调用的函数     | 创建 Issue、发送消息       |
| **Resources** | 可读取的数据     | 文件内容、数据库记录       |
| **Prompts**   | 预定义的提示模板 | 代码审查模板、Bug 报告模板 |

### 写一个简单的 MCP Server

用 Python SDK 写一个文件系统 MCP Server：

```python
from mcp.server import Server
from mcp.types import Tool, TextContent
import json

server = Server("filesystem")

# 定义工具
@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="read_file",
            description="读取指定路径的文件内容",
            inputSchema={
                "type": "object",
                "properties": {
                    "path": {"type": "string", "description": "文件路径"}
                },
                "required": ["path"]
            }
        ),
        Tool(
            name="list_directory",
            description="列出指定目录下的文件和子目录",
            inputSchema={
                "type": "object",
                "properties": {
                    "path": {"type": "string", "description": "目录路径"}
                },
                "required": ["path"]
            }
        )
    ]

# 实现工具
@server.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "read_file":
        with open(arguments["path"], "r") as f:
            content = f.read()
        return [TextContent(type="text", text=content)]

    elif name == "list_directory":
        import os
        entries = os.listdir(arguments["path"])
        return [TextContent(type="text", text=json.dumps(entries))]

# 启动
if __name__ == "__main__":
    import asyncio
    from mcp.server.stdio import stdio_server

    async def main():
        async with stdio_server() as (read, write):
            await server.run(read, write)

    asyncio.run(main())
```

这个 Server 启动后，任何支持 MCP 的 Agent 都能通过标准协议来读写文件——不需要为每个 Agent 框架单独写适配代码。

### 现有的 MCP Server 生态

MCP 协议发布后，社区快速构建了丰富的 Server 生态：

- **官方 Server**：[GitHub](https://github.com/modelcontextprotocol/servers)、Slack、PostgreSQL、Google Drive、Brave Search
- **社区 Server**：Notion、Linear、Jira、Discord、Docker、Kubernetes
- **本地工具**：文件系统、终端命令、浏览器自动化

你可以在 [MCP Servers 仓库](https://github.com/modelcontextprotocol/servers)（GitHub 60k+ stars）找到大部分已有的实现。

---

## Tool、Skill、MCP 的关系总结

把这三层概念放在一起看：

```
层次 1: Tool（工具）
  → 原子操作：search、fetch、calculate
  → 由 Function Calling 驱动

层次 2: Skill（技能）
  → 组合流程：多个 Tool 的有序编排
  → 对外暴露简单接口，内部封装复杂逻辑

层次 3: MCP（协议）
  → 标准化连接：让 Agent 即插即用地访问外部服务
  → 一个 MCP Server 可以同时提供 Tool、Resource、Prompt
```

用一个比喻：Tool 是螺丝刀和扳手，Skill 是"组装一张桌子"的操作手册，MCP 是标准化的工具箱接口——不管谁家的工具箱，打开就能用里面的工具。

---

## 下一步

到现在为止，你已经理解了 Agent 的完整理论体系：循环（05）、工具调用（06）、封装与标准化（07）。下一篇是**实战篇**——我们用 LangChain 框架把这些概念变成真正可运行的 Agent。

---

## 推荐资源

- [MCP 官方规范](https://modelcontextprotocol.io/) — 协议文档和规范说明
- [MCP Servers 仓库](https://github.com/modelcontextprotocol/servers) — 官方和社区 MCP Server 集合
- [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk) — Python 版 MCP 开发工具
- [Anthropic MCP 博客](https://www.anthropic.com/news/model-context-protocol) — MCP 的设计理念介绍
