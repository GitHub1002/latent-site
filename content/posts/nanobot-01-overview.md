---
title: "nanobot 源码解读 · 01 | 开篇：定位、架构、跑起来与目录导览"
date: 2026-08-03
draft: false
weight: 801
tags: ["nanobot", "源码解读", "Agent框架", "架构"]
summary: "从定位、整体架构、三种运行入口到完整目录导览，把 nanobot 这个开源个人 AI Agent 框架的全貌摊开。重点讲清 AgentLoop 与 AgentRunner 的分工，它是理解整个项目源码的钥匙。"
ShowToc: true
---

读懂一个框架，第一步不是钻进某个函数，而是先回答三个问题：**它到底是什么、它的主干数据流长什么样、我怎么把它跑起来**。这篇就回答这三个问题，并顺手把目录结构导览一遍，给你后面逐层深入铺路。

## 一、它到底是什么

`nanobot` 是 HKUDS（香港大学数据科学实验室）开源的、用 Python 写的**个人 AI Agent 框架**。它的自我定位是 *ultra-lightweight, self-hosted personal AI agent framework*——超轻量、可自托管、面向个人。

一句话内核（来自官方 README 的架构段）：

> nanobot 把一切都围绕一个**小的 agent loop** 来组织：消息从聊天应用进来，LLM 自己决定何时需要工具，记忆和技能只在需要时作为上下文被拉进来，而不是变成一个沉重的编排层。

这句话是理解整个项目的纲。它刻意**不做**那种"把流程、状态机、编排全部中心化"的重框架，而是保持一条可读的核心路径，让通道、工具、记忆、部署都作为外围插件挂上去。

它目前能做到的事（README 原文归纳）：

- 在浏览器 WebUI 或终端里运行；
- 接入 Telegram、Discord、Slack、微信、邮件、Mattermost 等聊天应用；
- 使用文件、shell、网页搜索、网页抓取、MCP、cron、图像生成、子 Agent 等工具；
- 通过 **Dream** 保留会话历史与长期记忆；
- 跑长程目标与定时自动化；
- 暴露 Python SDK 与 OpenAI 兼容 API；
- 作为长驻的本地或服务器端 Agent 网关部署。

版本上，最新发布是 **v0.3.0（The Agency Release）**，重点是把 nanobot 从"耐久工作台"升级为"能协调助手、按会话切换模型、把授权工作执行到底"的 Agent 运行时。

## 二、整体架构：一条主干 + 外围挂载

官方 `docs/architecture.md` 给了一张核心流图，我重绘成下面这张，方便后面反复引用：

![nanobot 整体架构](/images/nanobot-01-architecture.svg)

主干流向非常清晰，从左到右只有四站：

```
Channel  →  MessageBus  →  AgentLoop  →  AgentRunner  →  { Provider, Tools }
```

然后再原路出站：`AgentRunner` 把"模型回复 / 工具结果"回填后，由 `AgentLoop` 生成 `OutboundMessage`，经 `MessageBus` 回到原通道。

而 `AgentLoop` 下面那排紫色的"工作状态"（Session、Memory、Hooks、Skills、Templates），是它在每一轮里**读写的上下文来源**——注意它们是虚线连接，表示"被引用/被读写"，而不是主干数据流的一部分。

这里有几处值得记住的设计取向：

1. **Channel 与核心解耦**：无论是 CLI、WebUI 还是某个聊天应用，进来都统一翻译成 `InboundMessage` 事件，出去都翻译成 `OutboundMessage` 事件。核心不关心消息来自哪里。
2. **总线做事件中转**：`MessageBus`（`bus/events.py`、`bus/queue.py`）负责事件的入队与派发，是通道与引擎之间的缓冲层。
3. **引擎只管两件事**：`AgentLoop` 管"一轮对话怎么组织"（选 session、定 workspace、拼 context、挂 hook、发 outbound）；`AgentRunner` 管"模型怎么对话"（调 provider、流式、执行工具、回填、迭代上限）。
4. **Provider / Tools 是叶子节点**：它们是 `AgentRunner` 向下调用的能力，结果再回到 Runner，循环直到产出终答或触达运行上限。

## 三、理解全项目的钥匙：AgentLoop vs AgentRunner

`architecture.md` 用一整节强调两者的分工，因为这是调试时定位问题的分界线。我把它展开讲：

**`AgentLoop`（面向通道的一轮）** 负责：

- 接收入站消息；
- 确定"生效的 session"和"workspace 作用域"；
- 构建上下文（context）；
- 连接 hooks、进度回调、通道元数据；
- 发布出站消息。

**`AgentRunner`（面向模型的循环）** 负责：

- 把消息发给选中的 provider；
- 处理流式增量（streaming deltas）与推理块（reasoning blocks）；
- 执行工具调用；
- 把工具结果喂回模型；
- 在产出终答或触达运行限制时停止。

官方给的调试口诀很实用：

> 如果问题出在**通道路由、session key、workspace 选择、出站投递**，从 `agent/loop.py` 入手；如果问题出在 **provider 调用、工具调用、流式、迭代上限**，从 `agent/runner.py` 入手。

这一分离也是 nanobot "小核心" 的关键：把"对话轮次的骨架"（Loop）和"与模型/工具博弈的细节"（Runner）分开，前者稳定、后者可替换可扩展。

## 四、怎么把它跑起来

nanobot 的入口不是 `python xxx.py`，而是一个安装后的命令行 `nanobot`。三种安装方式（README 摘录）：

```bash
# 1) 一行脚本安装（推荐的桌面体验，会自动起 WebUI）
curl -fsSL https://raw.githubusercontent.com/HKUDS/nanobot/main/scripts/install.sh | sh

# 2) 用 uv 安装
uv tool install nanobot-ai

# 3) 从 PyPI 用 pip 安装
python -m pip install nanobot-ai

# 4) 从源码安装（需要 bun 或 npm 来构建 WebUI）
git clone https://github.com/HKUDS/nanobot.git
cd nanobot
python -m pip install .
```

装好后，它提供**三种运行入口**，对应三种使用姿势：

| 命令 | 用途 | 特点 |
|---|---|---|
| `nanobot webui` | 浏览器工作台 | 推荐首选；自动建配置与 workspace，开启本地 WebSocket 通道，开 `http://127.0.0.1:8765`；首次运行默认只绑 localhost |
| `nanobot gateway` | 网关优先 | 跳过 WebUI 引导，直接在当前终端跑完整网关；适合把 Agent 当长驻服务 |
| `nanobot agent` | 纯终端 | 交互式终端聊天；退出后不会保留通道与自动化；`nanobot agent -m "Hello!"` 可一次性请求后退出 |

首次使用三步：打开 **Settings → Models** 选 provider / 凭证 / 模型；新建话题发 `Hello!` 验证连通；做项目前在 composer 里选好 workspace 和访问模式。

想让它"关掉终端也一直在跑"，用 `nanobot webui --background`，之后可用 `nanobot gateway status / logs / restart / stop` 管理。

> 注意：WebUI 的生产构建写在 `nanobot/web/dist/` 并打进 wheel，由 WebSocket 通道提供；健康检查端点在 `http://127.0.0.1:18790/health`，而 WebUI/WebSocket 在 `http://127.0.0.1:8765`。别把两者搞混。

## 五、目录导览：核心包与各模块职责

`nanobot/` 这个包很大，但按"主干 + 外围"来认，就不会迷路。下面这张表是读源码时的"地图"：

| 模块 | 职责 |
|---|---|
| `nanobot/nanobot.py` | 顶层 `Nanobot` 类，封装 `run / run_streamed / stream`，是读源码的最佳起点 |
| `nanobot/agent/loop.py` | `AgentLoop`：面向通道的一轮编排（session / workspace / context / hooks / outbound） |
| `nanobot/agent/runner.py` | `AgentRunner`：面向模型/工具的循环（provider 调用、流式、工具执行、回填、迭代上限） |
| `nanobot/agent/context.py` | 上下文拼装 |
| `nanobot/agent/context_governance.py` | 上下文自动压缩与预算（Context Engineering 的实体化） |
| `nanobot/agent/memory.py` | `MemoryStore` 与 **Dream** 长期记忆合并任务 |
| `nanobot/agent/tools/` | 工具实现与发现：`base.py`/`schema.py`/`registry.py` + `shell.py`/`filesystem.py`/`web.py`/`mcp.py`/`cron.py`/`image_generation.py`/`self.py` |
| `nanobot/agent/subagent.py` | 多 Agent 委派 |
| `nanobot/providers/` | Provider 元数据与实现：`registry.py` / `factory.py` / `openai_compat_provider.py` + anthropic/bedrock/fallback 等 |
| `nanobot/channels/` | 通道插件：`base.py` + `manager.py`（发现/生命周期）+ `websocket/` + telegram/discord/slack/feishu/weixin/whatsapp/qq/… 20+ 平台包 |
| `nanobot/security/` | 安全边界：workspace 作用域、SSRF、shell 沙箱、PTH 守卫 |
| `nanobot/session/manager.py` | 会话存储与压缩 |
| `nanobot/sdk/` + `nanobot/api/` | Python SDK 与 OpenAI 兼容 API |
| `nanobot/cron/` + `nanobot/triggers/` + `nanobot/gateway/` | 定时任务、触发器、长驻网关 |
| `nanobot/config/` | 配置 schema / loader / paths（默认配置在 `~/.nanobot/config.json`） |

几个**默认路径**也值得记一下（来自 `architecture.md`）：

| 数据 | 默认位置 |
|---|---|
| 配置 | `~/.nanobot/config.json` |
| Workspace | `~/.nanobot/workspace/` |
| 会话 | `<workspace>/sessions/*.jsonl` |
| 长期记忆 | `<workspace>/memory/MEMORY.md` |
| 记忆合并来源 | `<workspace>/memory/history.jsonl` |
| Cron 任务 | `<workspace>/cron/jobs.json` |

还有一个容易混淆的点：**"agent 拥有的 workspace"** 和 **"会话携带的生效 project workspace"** 经常是同一路径，但 WebUI 里可以单独选一个 project。身份文件 `SOUL.md` / `USER.md`、记忆、自定义技能归前者；项目的 `AGENTS.md`、相对工具路径、shell 工作目录归后者。后面讲上下文工程（03）和工具安全（05）时会再回来。

## 六、为什么值得读它

把 nanobot 读完，性价比在于三件事：

1. **它真的"小且可读"**。核心引擎就两个文件，没有隐藏的重编排黑箱。读完你脑子里会有一张"一个 Agent 框架到底由哪些零件组成"的完整拼图。
2. **Context Engineering 被实体化了**。上下文拼装、自动压缩、预算（03 / 04 篇）不是概念，而是 `context.py` / `context_governance.py` / `memory.py` 里的真实代码。
3. **工程范式很全**。工具发现框架、SSRF/沙箱安全边界、MCP 接入、多后端 Provider 路由、20+ 聊天平台插件、OpenAI 兼容 API——这些都是"把一个 Agent 做成产品"绕不开的题，nanobot 都给了参考答案。

## 七、本系列后续

路线图（`nanobot-00-roadmap.md`）列出了 01–09 共 9 篇。下一篇 **02 · 核心引擎** 会直接打开 `agent/loop.py` 和 `agent/runner.py`，把"一条消息从 `Channel` 进来，到 `AgentRunner` 里反复调模型/工具、最后出站"的全过程，按真实代码逐函数拆给你看。

## 八、小结

- nanobot = HKUDS 开源的、Python 写的、**小核心 + 插件式外围**的个人 AI Agent 框架。
- 主干数据流：`Channel → MessageBus → AgentLoop → AgentRunner → {Provider, Tools}`，再原路出站。
- 理解全项目的钥匙是 `AgentLoop`（面向通道的一轮）与 `AgentRunner`（面向模型的循环）的分工。
- 三种运行入口：`webui` / `gateway` / `agent`，分别对应浏览器、长驻服务、纯终端。
- 读源码从 `nanobot/nanobot.py` 起步，按"主干 → 上下文/记忆 → 工具/安全 → MCP/Provider → 多 Agent/产品化"的顺序深入。

下一篇见。
