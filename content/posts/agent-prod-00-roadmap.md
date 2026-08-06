---
title: "系列三 · 00 | 开篇:把企业知识助手送上生产级"
date: 2026-07-28
draft: true
weight: 700
tags:
  - agent
  - 生产化
  - langgraph
  - mcp
  - 系列三
summary: "系列三总纲:把企业知识助手从 MVP 升级到生产级,聚焦招聘最高频的四块工程能力——LangGraph、MCP、Agent 评估、Docker 部署。含四阶段地图与标题表。"
ShowToc: true
---

## 为什么要开「系列三」

系列二(M0–M5)已经讲完「从 0 到 1 做出一个能用的企业知识助手」:FastAPI + LangChain + RAG + 权限红线 + 流式 + React/JWT,是一个完整闭环。读者看到 05 就知道「能跑了」。

但**能跑 ≠ 能上岗**。翻一遍 2026 年的 Agent 开发 JD(BOSS、智联、51job、牛客 JD 汇总、北京 813 个岗位大数据),生产级岗位最缺的是这四块:

- **LangGraph**(103 个岗位点名)——生产级 Agent 编排的代名词
- **MCP 协议**(多篇标「2026 必学」)——工业级工具接入标准
- **Agent 评估 / 可观测性**——40K+ 档的硬要求
- **部署(Docker)**(276 个岗位)——很多 JD 明写

系列三就干一件事:**把同一个项目从 MVP 推到生产级**,每一步都配套一篇实战文章,既涨能力又涨简历作品集。

## 四阶段地图(标题表)

| 阶段 | 文件 | 标题 | 覆盖能力 |
|---|---|---|---|
| 路线图 | `agent-prod-00-roadmap.md` | 系列三 · 00 \| 开篇:把企业知识助手送上生产级 | 总纲 + 地图 |
| 一 | `agent-prod-01-langgraph.md` | 系列三 · 01 \| 阶段一·用 LangGraph 重写编排:从 ReAct 循环到可控状态图 | StateGraph + Checkpointer + 流式 |
| 二 | `agent-prod-02-mcp.md` | 系列三 · 02 \| 阶段二·把工具封装成 MCP Server:让 Agent 说"标准语言" | FastMCP + stdio + 外部调试 |
| 三 | `agent-prod-03-evals.md` | 系列三 · 03 \| 阶段三·给 Agent 装上评估:用 LangSmith 量化准确率与越权拦截 | eval 数据集 + tracing + 指标 |
| 四 | `agent-prod-04-docker.md` | 系列三 · 04 \| 阶段四·Docker 一键部署:让作品集可在线体验 | Dockerfile + compose + 演示 |

> weight 约定:系列三统一 700+ 段(路线图 700,后续 701–704),排在系列二(600+)之后。

## 与招聘能力对照

| 系列三阶段 | 对应 JD 高频要求 | 作品集卖点 |
|---|---|---|
| LangGraph | 生产级 Agent 编排、状态/记忆、可控执行 | 从「调 chain」升级为「设计可控执行图」 |
| MCP | 2026 必学、工具标准化协议 | 懂 Agent↔工具解耦,不止 `@tool` 装饰器 |
| Evals | 40K+ 档评估体系、可观测性 | 能量化效果、防回归,面试有得讲 |
| Docker | 部署(276 岗)、Docker/K8s | `docker compose up` 即可体验的 demo |

## 贯穿始终的两条红线(不可破)

重构任何框架都**不允许**破坏已有的安全设计:

1. **权限红线**:工具零参数 + 服务端 `user_id_ctx` 注入。模型永远只能"决定调不调",不能"指定查谁"——张三查不到李四。
2. **M4 友好降级**:RAG 调过但无命中、且非本人数据时,仍降级为友好话术,不把模型可能编造的内容流出。

## 执行顺序

全局前置准备 → **① LangGraph** → **② MCP Server** → **③ Evals** → **④ Docker**。每步一篇,且每步都保持上述红线/降级不变。

## 后续可同系列扩展(第二/三档,视时间)

- `05 多智能体`(CrewAI/AutoGen/LangGraph multi-agent,市场第一需求)
- `06 GraphRAG / 混合检索 / rerank`(RAG 进阶)
- `07 文件上传个人知识库`(把 M1 的 RAG 从预置补成用户自助)
- `08 监控告警 / 成本管控`(LLMOps)
