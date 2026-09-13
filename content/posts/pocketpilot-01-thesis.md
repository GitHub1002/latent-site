---
title: "PocketPilot 源码解读 · 01 | 它不是聊天机器人：本地优先、策略可控、可审计"
date: 2026-09-13
slug: pocketpilot-01-thesis
draft: false
weight: 910
tags: ["PocketPilot", "源码解读", "Agent Runtime", "本地优先", "策略引擎"]
summary: "PocketPilot 是一份可读的个人 Agent Runtime：Python 3.12 + Next.js，成功标准不是「答得更聪明」，而是任务质量、权限正确、过程可看、失败可收。本文先钉死产品论点、分层架构和本系列四篇怎么读，配架构图。"
ShowToc: true
---

nanobot 把「一个讲得清的分层引擎」摊开了，DeepSeek Harness 把「一切皆插件」摊开了。PocketPilot 回答的是第三问：**模型已经会调工具了，怎样才敢让它碰你自己的磁盘？**

一句话定位：

> 本地优先、策略可控、可审计的个人 Agent Runtime。用户授权一个工作区目录之后，Agent 可以列文件、读文本、抽 PDF/表格、写报告；**凡有副作用的工具调用，都必须先过策略检查与人工审批**。过程不是「跑完一轮 goal 就结束」，而是一条可回放的事件流。

它不是又一个套了 LangChain 的聊天框。成功标准写在产品论题里：任务质量、权限正确、过程可理解、审计可回放、失败可收——不是「更会说话」。

## 一、它到底在解决什么

LLM 已经能理解「把这个目录里的报价单汇总成报告」。缺的是后面三件事：

1. **安全地做**：不能 `../` 逃到工作区外，不能还没点批准就把文件写上去。
2. **可恢复地做**：写操作要能暂停、改参、拒绝；恢复之后策略不能失效。
3. **可审计地做**：模型说了什么、调了什么工具、谁批了一次高风险写，事后要能按事件重放。

Open Interpreter 一类工具把「能干活」放到了前面，权限故事偏弱。nanobot 把好用做得很满（Dream 长期记忆、默认 Shell、子 Agent、20+ 通道），安全是外围。PocketPilot 反过来：**好用对齐 nanobot 的一部分（记忆、技能、MCP、会话、流式），安全故事自己留。** 它明确不做 Dream/Heartbeat、默认 Shell/网络、子 Agent。

目标读者是两拨人：要把本地文件工作流交给 Agent 的开发者；以及想读一份**小而可检查**的 runtime、而不是一份编排框架的人。

当前产品版本在文档里称 **v0.3**，后端包版本 `0.1.0`。M0–M5 功能交付已完成，M6 对齐 nanobot 的核心能力（技能 / 记忆 / MCP / 续聊 / Chat Web / 流式）已落地。演示视频和 Telegram 通道仍在路线图上，不在本系列范围内。

## 二、分层：Agent 不碰数据库

![PocketPilot 分层运行时](/images/pocketpilot-01-architecture.svg)

依赖单向，写在架构约定里，不是靠某个 DI 框架去「保证」：

```
API / CLI / webhook / cron
        →  TaskService / Worker
                →  runtime / policy / tools / providers
                        →  storage
```

几个角色值得先记住，后面三篇都在这张图上打转：

| 角色 | 文件 | 干什么 |
|------|------|--------|
| 入口 | `web/` · `app/cli.py` · `channels/` · `scheduler/` | 只创建 Task，不自己跑循环 |
| `TaskService` | `backend/app/services/task_service.py` | 建任务、跑/恢复、审批、取消；把内存事件刷进 SQLite 和 SSE |
| `TaskWorkerManager` | `backend/app/worker/manager.py` | `asyncio.create_task`；每个任务重建 Provider 和工具表 |
| `Agent` | `backend/app/runtime/agent.py` | 有界模型—工具循环；**不访问数据库** |
| `PolicyEngine` + `WorkspaceGuard` | `backend/app/policy/` | capability 后缀分类 + 路径牢笼 |
| `ToolRegistry` | `backend/app/tools/` | 唯一注册口在 `factory.py` |
| SQLite | `backend/app/storage/` | Task、Event、Approval、Artifact、Session、Channel、ScheduledTask |

> 钥匙：**Agent 循环只在内存里累积 `RuntimeEvent`。落库、推 SSE、审批恢复，全是 TaskService 的事。** 这和 nanobot 把产品层（`AgentLoop`）与引擎层（`AgentRunner`）切开是同一类纪律，只是切法更狠——引擎连 session 都不认识。

## 三、技术选型（以及故意不选什么）

| 层 | 选型 | 为什么 |
|----|------|--------|
| 后端 | Python 3.12 · FastAPI · Typer · uv | 和工具/文档生态对齐，CLI 与 API 共用同一套服务 |
| 校验 | Pydantic v2 · `POCKETPILOT_*` | 配置进环境变量，不进代码 |
| 存储 | SQLite + SQLAlchemy 2.0 | 本地优先；仓库里没有 Postgres |
| 模型 | OpenAI 兼容 `/chat/completions`（httpx） | DeepSeek / Ollama / LM Studio / 官方 OpenAI 只换 `base_url` |
| 评测模型 | `ScriptedProvider` | 26 个用例不用花 token 就能回归安全和任务行为 |
| 文档 | PyMuPDF · python-docx · openpyxl | 发票/报价是旗舰 demo |
| 前端 | Next.js 16 · React 19 · Tailwind 4 | Chat-first 控制台；实时用 **SSE** 不是 WebSocket |
| 部署 | Docker Compose（API :8000，Web :3000，数据 `./data`） | 一键起来；首次播种 demo 工作区 |
| RAG | **没有** | 「记忆」是工作区内有界的 `MEMORY.md`，注入 system prompt |

几个取舍要提前说死，避免后面读源码时觉得「怎么这么土」：

- **单 Agent，不引入图编排。** 审批、checkpoint、SSE、策略要当一等公民；LangGraph 会把这些变成节点副作用。
- **EventBus 是进程内 `asyncio.Queue`。** 单 worker 正确；`uvicorn --workers > 1` 会丢直播。这是本地个人 runtime 的诚实边界。
- **MCP 不准 `list_tools`。** 连上就灌进 200 个工具，审批卡片会爆炸。能力必须声明，未知 capability 不注册。
- **cron / webhook 也不自动批准写盘。** 无人值守把文件写出去，产品论题就废了。

## 四、怎么跑起来

前置：Docker Desktop，或 Python 3.12 + [uv](https://docs.astral.sh/uv/) + Node 22。

```bash
# 仓库根目录
docker compose up --build
```

| 服务 | 地址 |
|------|------|
| API | http://localhost:8000 （`/health`） |
| Web | http://localhost:3000 |
| 数据 | `./data`（SQLite + 工作区）；首次会播种 demo 工作区 |

本地开发有一个真实脚枪：**必须在 `backend/` 下起 uvicorn**。`Settings` 读 `.env` / `../.env`，默认库路径相对 cwd。从仓库根目录起服务，曾经会连错库、读错 key。

```bash
cd backend
uv sync --all-groups
uv run uvicorn app.main:app --host 127.0.0.1 --port 8000 --reload

cd web
npm install
npm run dev
```

未配置 API Key 时服务可以起来；发出去的任务会很快失败，并写明去设置页配模型——不会把任务永远留在 `queued`。

质量命令（都在 `backend/`）：

```bash
uv run pytest --basetemp=.pytest-basetmp
uv run ruff check .
uv run mypy app evals
uv run pocketpilot eval          # ScriptedProvider，26/26
uv run pocketpilot eval --real   # 真模型，噪声更大
```

## 五、本系列怎么读（四篇，按论点切）

不另写 00 路线图。PocketPilot 的核心循环大约 250 行，按 nanobot 的「逐文件 9 篇」拆会出水分。四篇各自只回答一个问题：

| 篇 | 只回答 | 核心代码 |
|----|--------|----------|
| **01（本篇）** | 它不是聊天机器人 | README、架构分层、目录 |
| **02** | 一条消息怎么从输入框活到气泡 | `ChatShell` · `TaskService` · `EventBus` · `runtime/agent.py` |
| **03** | 护栏为什么是护城河 | `policy/workspace.py` · `policy/engine.py` · 审批恢复 |
| **04** | 新能力为什么都挂同一条总线 | `tools/factory.py` · 记忆/技能/MCP · cron/webhook · evals |

建议 **01 → 04 按序读**。02 是脊柱，03 是论点，04 证明「记忆、MCP、定时任务」没有另开一套权限故事。

对照阅读（可选，不作为前提）：

- 循环形态：本系列 02 vs [nanobot 02]({{< relref "nanobot-02-core-engine.md" >}}) vs [Harness 01]({{< relref "harness-01-plugin-engine.md" >}})
- 工具把关：本系列 03 vs [nanobot 05]({{< relref "nanobot-05-tools.md" >}}) vs [Harness 02]({{< relref "harness-02-tools-pipeline.md" >}})
- 总对照：[Harness vs nanobot]({{< relref "harness-vs-nanobot.md" >}})

## 六、小结

- PocketPilot 的产品论题是**可控**，不是更聪明的对话。
- 分层纪律：Agent 不碰库；TaskService 负责持久化、SSE、审批恢复。
- 策略按 capability 后缀分类，路径先过 `WorkspaceGuard`；写操作在磁盘变更前暂停。
- 本系列四篇按论点切，不按目录清单切。

下一篇 [**02 · 一条消息怎么活**]({{< relref "pocketpilot-02-message-life.md" >}}) 会沿着 `ChatShell` → `POST /api/tasks` → Worker → `Agent.run` → SSE 走一遍，并讲清为什么 `assistant_delta` 故意不落库。
