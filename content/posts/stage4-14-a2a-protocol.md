---
title: "14 | A2A 协议——Agent 之间怎么通信？"
date: 2026-07-08
draft: false
weight: 402
tags: ["Agent", "A2A", "Multi-Agent"]
summary: "Google 提出的 A2A（Agent-to-Agent Protocol）是 Agent 间通信的标准协议。这篇从 MCP 的局限性出发，讲解 A2A 的核心概念（Agent Card、Task、Message、Artifact）、协议流程、与 MCP 的互补关系，以及实际接入方式。"
ShowToc: true
---

上一篇讲了多 Agent 架构模式——层级式、辩论式、流水线式。当你决定把一个大任务拆给多个 Agent 来做时，一个绕不开的问题立刻出现：**它们怎么通信？**

你当然可以自己写消息传递逻辑。Agent A 调 Agent B 的 HTTP API，Agent B 调 Agent C 的 gRPC 接口，Agent C 再通过 WebSocket 把结果推回去……每对 Agent 之间都要写一套适配代码。三个 Agent 就有 6 条连接，十个 Agent 就是 90 条。

这场景是不是很熟悉？在 USB 标准出现之前，每个外设都有自己的接口——打印机用并口、鼠标用串口、键盘用 PS/2。每换一个设备就得装一套驱动。USB 的意义就是把这些混乱的接口统一成一个标准。

**A2A（Agent-to-Agent Protocol）就是 Agent 世界的"USB 标准"。** 它由 Google 在 2025 年提出，目标是定义一套标准协议，让不同框架、不同厂商构建的 Agent 能够互相发现、互相通信、互相协作。

![A2A 协议核心概念](/images/stage4-14-a2a-protocol.svg)

---

## MCP vs A2A：两种协议各管什么

在聊 A2A 之前，你可能已经听过另一个协议——**MCP（Model Context Protocol）**。这是 Anthropic 提出的，用来解决 Agent 和工具之间的连接问题。两个协议名字相近，但解决的问题完全不同。

| 维度 | MCP | A2A |
|------|-----|-----|
| 提出方 | Anthropic | Google |
| 解决的问题 | Agent 怎么用工具 | Agent 怎么找 Agent、怎么对话 |
| 连接方向 | Agent → Tool | Agent → Agent |
| 核心概念 | Tool / Resource / Prompt | Agent Card / Task / Message |
| 通信模型 | 同步调用（类函数调用） | 异步对话（类 HTTP 请求） |
| 类比 | "手"——连接工具 | "嘴"——Agent 之间对话 |

用更直观的说法：MCP 让 Agent 能**操作东西**（读文件、查数据库、调 API），A2A 让 Agent 能**找另一个 Agent 帮忙**。

一个实际场景：你的"研究 Agent"需要翻译一份日文论文。它自己不会翻译，但它可以：
- 通过 MCP 调用翻译 API（工具层面）
- 或者通过 A2A 找到"翻译 Agent"，把整个翻译任务交给它（Agent 层面）

两种方式的差别在于：MCP 调的是"工具"，Agent 自己还是决策主体；A2A 调的是"另一个 Agent"，那个 Agent 有自己的推理能力和决策权。

这两个协议是互补关系，不是竞争关系。一个成熟的多 Agent 系统通常同时需要两者——用 MCP 让每个 Agent 接入工具，用 A2A 让 Agent 之间互相协作。

---

## A2A 核心概念

A2A 协议定义了四个核心概念：**Agent Card、Task、Message、Artifact**。逐个来看。

### Agent Card：Agent 的名片

你打开一个 Agent 的网络，怎么知道它能做什么？A2A 的答案是：每个 Agent 都在一个固定路径上发布自己的"名片"——一个 JSON 文件，放在 `/.well-known/agent.json`。

```json
{
  "name": "Translation Agent",
  "description": "专业翻译 Agent，支持中/英/日/韩四种语言互译，擅长技术文档和学术论文翻译。",
  "url": "https://translation-agent.example.com",
  "version": "1.2.0",
  "capabilities": {
    "streaming": true,
    "pushNotifications": false,
    "stateTransitionHistory": true
  },
  "authentication": {
    "schemes": ["Bearer"]
  },
  "skills": [
    {
      "id": "tech-translation",
      "name": "技术文档翻译",
      "description": "翻译技术类文档，保留代码格式和术语准确性",
      "tags": ["translation", "technical", "documentation"],
      "examples": [
        "将这篇英文 API 文档翻译为中文",
        "把这份中文技术规范翻译为日文"
      ]
    },
    {
      "id": "academic-translation",
      "name": "学术论文翻译",
      "description": "翻译学术论文，保持引用格式和专业术语一致性",
      "tags": ["translation", "academic", "paper"]
    }
  ],
  "provider": {
    "organization": "Example AI Lab",
    "url": "https://example.com"
  }
}
```

几个关键字段：

- **name / description**：Agent 叫什么、能做什么。Client Agent 通过读这个描述来决定要不要找它帮忙。
- **url**：通信地址，所有 JSON-RPC 请求都发到这里。
- **capabilities**：声明自己支持哪些高级特性。`streaming: true` 表示支持 SSE 流式响应；`pushNotifications: true` 表示支持推送通知（适合长时间任务）。
- **skills**：技能列表。每个 skill 有 id、名称、描述、标签。Client Agent 通过匹配 skill 描述来决定把任务交给谁。
- **authentication**：认证方式，支持 Bearer Token、OAuth2 等标准方案。

类比：Agent Card 就像一家餐厅在大众点评上的页面——你看菜系、评分、地址，然后决定要不要去。

### Task：交互的核心单元

A2A 的交互都围绕 **Task** 展开。Client Agent 向 Remote Agent 发起一个 Task，Remote Agent 处理完后返回结果。

每个 Task 有唯一 ID，并且有明确的生命周期：

```
submitted → working → completed
                    → failed
                    → canceled
         → input-required → (补充信息后) → working → completed/failed
```

各状态含义：

- **submitted**：Task 已提交，还没开始处理
- **working**：Remote Agent 正在处理中
- **input-required**：Remote Agent 需要更多信息才能继续（比如翻译 Agent 问"目标语言是什么？"）
- **completed**：成功完成
- **failed**：处理失败
- **canceled**：被取消

这个状态机设计的关键点是 **input-required**。它允许 Remote Agent 在执行过程中向 Client 反问——这就不是简单的"函数调用"了，而是一次真正的**对话**。

举个例子：你让翻译 Agent 翻译一段文字，它发现原文混合了中英文，于是返回 `input-required` 状态，问你："这段文字的目标语言是中文还是英文？"你回答后，它再继续工作。

### Message：Task 中的通信内容

Task 内部传递的内容叫 **Message**。每个 Message 有两个核心属性：

- **role**：`user`（Client Agent 发的）或 `agent`（Remote Agent 发的）
- **parts**：消息内容，支持三种类型

```json
{
  "role": "user",
  "parts": [
    {
      "type": "TextPart",
      "text": "请把下面这段英文翻译成中文"
    },
    {
      "type": "FilePart",
      "file": {
        "name": "api-spec.txt",
        "mimeType": "text/plain",
        "bytes": "base64encoded..."
      }
    }
  ]
}
```

三种 Part 类型：

- **TextPart**：纯文本，最常见的形式
- **FilePart**：文件，可以是内联的 base64 编码，也可以是 URI 引用（`uri: "https://storage.example.com/files/doc.pdf"`）
- **DataPart**：结构化数据（JSON），用于传递机器可读的信息

```json
{
  "type": "DataPart",
  "data": {
    "sourceLanguage": "en",
    "targetLanguage": "zh",
    "preserveFormatting": true,
    "glossary": {
      "token": "token",
      "embedding": "嵌入向量"
    }
  }
}
```

DataPart 的设计很实用。很多 Agent 间的通信需要传递结构化参数，硬塞进纯文本里再让另一个 Agent 去解析，既浪费 token 又容易出错。DataPart 直接传 JSON，干净利落。

### Artifact：最终交付物

**Artifact** 是 Remote Agent 产出的最终结果。它和 Message 的区别是：

- **Message** 是沟通过程（"我收到了你的请求"、"需要补充信息"、"正在处理中"）
- **Artifact** 是最终交付物（翻译好的文档、生成的报告、分析结果）

```json
{
  "artifacts": [
    {
      "name": "translated-document",
      "description": "翻译完成的中文技术文档",
      "parts": [
        {
          "type": "TextPart",
          "text": "# API 规范文档\n\n本文档定义了 Agent 间通信的标准接口..."
        },
        {
          "type": "FilePart",
          "file": {
            "name": "translated-spec.pdf",
            "mimeType": "application/pdf",
            "uri": "https://storage.example.com/outputs/translated-spec.pdf"
          }
        }
      ]
    }
  ]
}
```

类比：你去餐厅吃饭，和服务员的对话是 Message（"请问您要几分熟？""七分熟""好的，请稍等"），端上来的牛排是 Artifact。

---

## 一次完整的 A2A 交互流程

光讲概念不够，我们用一个具体例子走一遍完整流程。

场景：**研究 Agent**（Client）需要把一份 3000 字的英文技术报告翻译成中文，它找到了**翻译 Agent**（Remote）。

### Step 1：发现 Agent Card

研究 Agent 先获取翻译 Agent 的名片：

```
GET https://translation-agent.example.com/.well-known/agent.json
```

拿到上面那个 Agent Card JSON 后，研究 Agent 确认：这个 Agent 有 `tech-translation` 技能，支持流式响应，用 Bearer Token 认证。

### Step 2：发送 Task

研究 Agent 发起 JSON-RPC 请求，方法是 `tasks/send`：

```json
{
  "jsonrpc": "2.0",
  "id": "req-001",
  "method": "tasks/send",
  "params": {
    "id": "task-2024-0715-001",
    "sessionId": "session-abc123",
    "message": {
      "role": "user",
      "parts": [
        {
          "type": "TextPart",
          "text": "请将以下英文技术报告翻译为中文，保留 Markdown 格式，术语首次出现时附英文原文。"
        },
        {
          "type": "DataPart",
          "data": {
            "sourceLanguage": "en",
            "targetLanguage": "zh",
            "preserveFormatting": true,
            "termHandling": "first-occurrence-bilingual"
          }
        },
        {
          "type": "FilePart",
          "file": {
            "name": "report.md",
            "mimeType": "text/markdown",
            "uri": "https://storage.example.com/inputs/report.md"
          }
        }
      ]
    }
  }
}
```

### Step 3：Remote Agent 处理并返回结果

翻译 Agent 收到请求后，开始处理。如果是同步模式，直接返回完成结果：

```json
{
  "jsonrpc": "2.0",
  "id": "req-001",
  "result": {
    "id": "task-2024-0715-001",
    "sessionId": "session-abc123",
    "status": {
      "state": "completed",
      "timestamp": "2026-07-15T10:32:45Z"
    },
    "artifacts": [
      {
        "name": "translated-report",
        "description": "翻译完成的中文技术报告",
        "parts": [
          {
            "type": "TextPart",
            "text": "# A2A 协议技术报告\n\n## 概述\n\nAgent-to-Agent Protocol（Agent 间通信协议，简称 A2A）是由 Google 提出的标准化通信方案..."
          },
          {
            "type": "FilePart",
            "file": {
              "name": "report-zh.md",
              "mimeType": "text/markdown",
              "uri": "https://storage.example.com/outputs/report-zh.md"
            }
          }
        ]
      }
    ]
  }
}
```

### Step 4：流式响应（SSE）

3000 字的翻译不可能瞬间完成。如果翻译 Agent 声明支持 `streaming: true`，Client 可以用 `tasks/sendSubscribe` 发起流式请求，Remote Agent 通过 **Server-Sent Events（SSE）** 逐步返回进度：

```
POST https://translation-agent.example.com
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "id": "req-002",
  "method": "tasks/sendSubscribe",
  "params": {
    "id": "task-2024-0715-001",
    "sessionId": "session-abc123",
    "message": { "..." }
  }
}
```

返回的 SSE 流：

```
event: status
data: {"jsonrpc":"2.0","id":"req-002","result":{"id":"task-2024-0715-001","status":{"state":"working","message":{"role":"agent","parts":[{"type":"TextPart","text":"开始翻译，预计 30 秒完成..."}]}}}}

event: status
data: {"jsonrpc":"2.0","id":"req-002","result":{"id":"task-2024-0715-001","status":{"state":"working","message":{"role":"agent","parts":[{"type":"TextPart","text":"已完成 50%..."}]}}}}

event: artifact
data: {"jsonrpc":"2.0","id":"req-002","result":{"id":"task-2024-0715-001","artifact":{"name":"translated-report","parts":[{"type":"TextPart","text":"# A2A 协议技术报告\n\n## 概述..."}]}}}

event: status
data: {"jsonrpc":"2.0","id":"req-002","result":{"id":"task-2024-0715-001","status":{"state":"completed","timestamp":"2026-07-15T10:33:12Z"}}}
```

SSE 的好处很明显：3000 字的翻译可能耗时 20~30 秒，如果让 Client 干等，它不知道进度如何。通过流式事件，Client 能实时看到翻译进展，用户体验好得多。

### 中间穿插：input-required 的场景

如果翻译 Agent 在处理时发现原文有一处歧义，它可以返回 `input-required` 状态：

```json
{
  "jsonrpc": "2.0",
  "id": "req-003",
  "result": {
    "id": "task-2024-0715-001",
    "status": {
      "state": "input-required",
      "message": {
        "role": "agent",
        "parts": [
          {
            "type": "TextPart",
            "text": "原文第 3 段提到 'The agent runs on the cluster'，这里的 'runs' 是指'部署运行'还是'执行任务'？"
          }
        ]
      }
    }
  }
}
```

Client Agent 收到后，把用户的回答再通过一次 `tasks/send` 发回去：

```json
{
  "jsonrpc": "2.0",
  "id": "req-004",
  "method": "tasks/send",
  "params": {
    "id": "task-2024-0715-001",
    "sessionId": "session-abc123",
    "message": {
      "role": "user",
      "parts": [
        {
          "type": "TextPart",
          "text": "指'部署运行'，翻译为'Agent 部署在集群上'"
        }
      ]
    }
  }
}
```

翻译 Agent 收到补充信息，继续工作，直到完成。

这就是 A2A 和普通 HTTP API 调用的核心区别：**它不是一次性的请求-响应，而是一个有状态的多轮对话。**

---

## A2A 的设计亮点

把上面的流程串起来看，A2A 有几个值得注意的设计决策。

### 不透明执行

Remote Agent 内部怎么工作，Client Agent 完全不知道，也不需要知道。Client 只看到输入和输出。

翻译 Agent 内部可能用了 RAG 检索术语库、用了特定的翻译模型、做了 5 轮自我修正——这些对 Client 来说都是黑箱。这就和微服务架构的理念一致：**你只关心接口，不关心实现。**

这种设计的好处是松耦合。翻译 Agent 内部升级了模型、换了框架，只要 Agent Card 和协议接口不变，Client 完全无感。

### 长时间任务支持

很多 Agent 任务不是毫秒级能完成的。生成一份报告可能要 2 分钟，训练一个模型可能要 2 小时。A2A 提供两种异步机制：

- **SSE（Server-Sent Events）**：适合秒级到分钟级的任务，Client 保持连接实时接收事件
- **Push Notification**：适合小时级的任务。Client 在 Task 创建时注册一个 webhook URL，Remote Agent 完成后主动推送通知

```json
{
  "method": "tasks/send",
  "params": {
    "id": "task-long-001",
    "pushNotification": {
      "url": "https://research-agent.example.com/webhook/task-complete",
      "authentication": {
        "schemes": ["Bearer"]
      }
    },
    "message": { "..." }
  }
}
```

这样 Client 不需要一直挂着等结果，可以去做别的事。

### 多模态原生支持

从 Message 的 parts 设计就能看出来，A2A 天然支持文本、文件、结构化数据三种形式。你不需要为每种数据类型设计不同的接口——统一用 parts 数组，想传什么类型就加什么类型的 Part。

### 安全：基于标准 HTTP

A2A 没有发明新的传输协议，完全基于 HTTP + JSON-RPC 2.0。这意味着：

- 现有的 HTTP 中间件（网关、负载均衡、限流、日志）可以直接用
- 认证走标准方案：Bearer Token、OAuth 2.0、API Key
- TLS 加密开箱即用
- 防火墙和网络策略不需要特殊配置

这对企业落地非常友好。不需要为了 A2A 专门搭一套新的基础设施，现有的 HTTP 服务架构直接兼容。

---

## A2A 当前的状态（2026 年中）

A2A 协议从 2025 年发布到现在大约一年，生态在快速建设中。几个值得关注的进展：

**框架支持方面**：
- Google ADK（Agent Development Kit）原生支持 A2A，用 Python 几十行代码就能把一个 Agent 暴露为 A2A 服务端
- LangChain / LangGraph 已经集成了 A2A 客户端和服务端适配
- CrewAI 支持把 Crew 中的 Agent 通过 A2A 暴露给外部调用
- AutoGen 社区也在推进 A2A 支持

**和 MCP 的关系**：很多项目同时支持两者。比如一个 Agent 用 MCP 连接本地工具（文件系统、数据库），同时通过 A2A 接受其他 Agent 的任务请求。两者在同一进程中并存，各管各的职责。

**现实挑战**：
- 协议还在演进，部分高级特性（Push Notification 的详细规范、错误码标准）还在讨论中
- Agent Card 的发现机制目前主要靠手动配置 URL，还没有成熟的"Agent 注册中心"
- 跨组织的 Agent 互操作，信任和认证是个大问题——你怎么确认远端 Agent 返回的结果是可信的？

总体来说，A2A 处在一个"规范已定、生态在长"的阶段。如果你在构建多 Agent 系统，现在是开始关注 A2A 的好时机——不用等到生态完全成熟，先把 Agent Card 挂上去、把基本的 Task 交互跑通，后续协议升级时迁移成本很低。

---

## 下一步

这篇讲了 Agent 间通信的标准化方案。从 MCP 到 A2A，我们已经有了让 Agent 连接工具和连接彼此的标准协议。

但"能通信"只是基础。你有了一群能协作的 Agent，它们有角色分工、有通信协议——接下来要解决的问题是：**怎么让这个多 Agent 系统稳定、高效、可控地运行？**

具体来说：怎么控制 Agent 的行为边界？怎么做资源调度和优先级管理？出错了怎么回滚和恢复？怎么监控整个系统的运行状态？

这些问题有一个统一的名字：**Harness Engineering**——Agent 运行环境的约束与编排。这是下一篇的主题。

---

## 推荐资源

- [A2A 协议规范](https://github.com/google/A2A) — GitHub 官方仓库，包含完整协议文档和示例代码
- [A2A 官方文档](https://google.github.io/A2A/) — 规范详细说明，有 Python 和 JavaScript 的 Quickstart
- [Google ADK](https://github.com/google/adk-python) — Google Agent Development Kit，原生支持 A2A
- [LangChain A2A 集成](https://python.langchain.com/docs/integrations/a2a) — 如果你已经在用 LangChain，接入 A2A 最方便的方式
