---
title: "nanobot 源码解读 · 00 | 路线图：我们要读完一个怎样的框架"
date: 2026-08-03
draft: true
weight: 800
tags: ["nanobot", "源码解读", "Agent框架"]
summary: "本文是 nanobot 源码解读系列的路线图。先交代这个开源个人 AI Agent 框架的定位、整体架构与目录结构，并列出后续 9 篇的选题与对应核心源码文件，方便按顺序阅读。"
ShowToc: true
---

## 一、这个系列要做什么

`nanobot` 是 **HKUDS（香港大学数据科学实验室）开源的个人 AI Agent 框架**，口号是 *ultra-lightweight, self-hosted personal AI agent framework*。它用 Python 写成，把"工具、长期记忆、MCP 集成、模型路由、多 Agent 委派、定时自动化、OpenAI 兼容 API"都塞进一个**可读的小核心**里。

市面上讲 Agent 的教程大多停留在"调一个 chain / 用一个框架"，很少带你**真正读完一个框架的源码**。而这个项目恰好满足"可读"这个前提：核心引擎只有两个文件（`agent/loop.py` 约 2264 行、`agent/runner.py` 约 1670 行），其余能力（工具、Provider、Channel、记忆、安全）都以清晰的分层挂在核心之外。

这个系列的目标，就是**逐层把 nanobot 读透**——从"一个请求怎么走到答案"，到上下文工程、长期记忆、工具与安全边界、MCP、模型路由、多 Agent、以及一个框架如何接入 20+ 聊天平台并对外提供 API。全部基于你本地 `D:\github_projects\nanobot` 的真实代码，逐文件、逐函数讲解。

> 说明：本系列**只解读 nanobot 这个项目本身**，不与其它项目做任何对照，独立成篇。

## 二、后续篇章规划（9 篇 + 本路线图）

| 篇 | 标题 | 核心源码 | 看点 |
|---|---|---|---|
| 00 | 路线图（本篇） | — | 定位、架构总览、目录导览、篇目表 |
| 01 | 开篇：定位 · 架构 · 跑起来 · 目录导览 | `nanobot.py` / `docs/architecture.md` | 项目是什么、核心数据流、三种运行入口 |
| 02 | 核心引擎：`AgentLoop` 与 `AgentRunner` | `agent/loop.py` / `agent/runner.py` | 一次请求如何从通道走到答案；两个角色的分工 |
| 03 | 上下文工程：ContextGovernance 与自动压缩 | `agent/context.py` / `agent/context_governance.py` | 上下文怎么拼、怎么在超限时自动压缩 |
| 04 | 记忆与状态：Dream 长期记忆 + 会话压缩 | `agent/memory.py` / `session/manager.py` | 长期记忆如何合并沉淀；会话历史如何压缩 |
| 05 | 工具系统与安全边界 | `agent/tools/*` / `security/*` | 工具发现框架、shell/web/filesystem 实现、SSRF 与沙箱 |
| 06 | MCP 集成 | `agent/tools/mcp.py` | 如何把外部 MCP Server 接进来成为工具 |
| 07 | Provider 与模型路由 / 降级 | `providers/*` | 多后端适配、模型预设、fallback 路由 |
| 08 | 多 Agent 委派 | `agent/subagent.py` | 一个 Agent 如何把子任务派给子 Agent |
| 09 | 接入与产品化：Channel + SDK/OpenAI API + 自动化 | `channels/*` / `sdk/` / `api/` / `cron/` | 20+ 平台接入插件、Python SDK、OpenAI 兼容 API、定时任务 |

标注「核心」的是最值得精读的几篇：01 / 02 / 03 / 04 / 05 / 06。

## 三、阅读顺序建议

建议**严格按 00 → 09 顺序读**。因为 nanobot 是"小核心 + 插件式外围"的结构：

- 先懂核心数据流（01）和两个引擎角色（02），后面的每一层（上下文、记忆、工具、Provider、Channel）都是挂在 `AgentRunner` / `AgentLoop` 上的"被调用方"，理解了主干再看枝节会非常顺。
- 03 / 04 属于"上下文与状态"主题，紧接引擎之后最自然。
- 05 / 06 / 07 是三类"能力扩展"（工具、MCP、模型）。
- 08 / 09 是"协作与产品化"，放在最后。

## 四、本系列的写作约定

- 每篇都给出**精确的文件路径与行数**，方便你本地对照源码。
- 关键函数会**逐段注释**，而不是只贴结论。
- 遇到"为什么这么设计"，会单独点出取舍，而不是只描述"它做了什么"。
- 配图放在 `static/images/nanobot-NN-*.svg`。

---

下一篇 **01 · 开篇** 会从"这个项目到底是什么"讲起，并把它完整的运行时架构和目录结构摊开给你看。
