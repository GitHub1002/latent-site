---
title: "DeepSeek Harness 源码解读 · 03 | 能力 seam 与统一沙箱世界：换一个 Provider，Bash/PTY/LSP 全搬走"
date: 2026-08-23
draft: false
weight: 893
tags: ["Agent框架", "DeepSeek", "Harness", "源码解读", "沙箱", "扩展机制"]
summary: "Harness 的扩展是配置级的，秘密在能力 seam（接口/实现/消费方 三角色）。最精彩的一处：fs、subprocess、shell、terminals、lsp、sandbox 共享同一个沙箱模式与策略（ctx.sandboxPolicy），因此把文件系统与子进程指向远程沙箱，Bash、PTY、LSP 会一次性全部搬走，无需各自 fork。ctx.e2b 用一个共享 SDK 句柄把 fs-e2b 与 subprocess-e2b 收进同一 Linux 运行时。配共享沙箱世界图。"
ShowToc: true
---

## 一、扩展为什么是「配置级」的

nanobot 加能力，是「往模块里注册」；Harness 加能力，是「换插件 / 改配置」。这背后的机制叫 **能力 seam（capability seam）**。

每个能力由三角色构成：

- **Service Definition（SD）**：接口，定义这个能力「能做什么」；
- **Service Provider**：实现，真正干活的那个；
- **Consumer**：用它的地方，通常是面向模型的工具。

换一个 Provider，整个产品行为就变了——而消费方一行不动。再叠加 `cordis.patch.yml` 按条目 id 做替换 / 插入，改配置就能换掉任意一层行为。

![能力 seam 与统一沙箱世界](/images/harness-03-seam-sandbox.svg)

## 二、最精彩的一处：统一沙箱世界

光说概念太虚，看一个 Harness 真正独到的设计。在 `capability-seams` 文档里，有一组「执行世界」的 seam：

- `ctx.fs`（文件系统）：`fs-local` / `fs-sandbox` / `fs-e2b`
- `ctx.subprocess`（子进程 spawn）：`subprocess-local` / `subprocess-e2b`
- `ctx.shell`（面向模型的 shell 工具）：`bash-local` / `bash-sandbox` / `pwsh-local`
- `ctx.terminals`（PTY 会话）：`terminal-bash`
- `ctx.lsp`（语言服务器）：`lsp-local`
- `ctx.sandbox` / `ctx.sandboxPolicy`（沙箱策略）

关键点在于 `ctx.sandboxPolicy` 的文档说明：

> 统一保存部署默认模式和工作区根目录；只有沙箱执行器和提供方读取该服务……两类强制执行组件都读取该服务，**因此 bash 与 fs 不会限制到不同的根目录**。

这意味着什么？**把文件系统（`ctx.fs`）和子进程（`ctx.subprocess`）同时指向远程沙箱，Bash、PTY、LSP 会一次性全部搬过去，无需各自 fork 一份远程逻辑。** 它们共享同一个「沙箱模式 + 工作区根」。

对比一下：在很多 Agent 框架里，你想让工具跑在远程沙箱，得分别为 shell 工具、文件工具、LSP 各接一套远程实现。Harness 把它们收口到 seam，切换点只有一个。

## 三、e2b：把「同一运行时」做实

文档里 `ctx.e2b` 这条最能说明「收口」不是嘴上说说：

> 拥有一个共享的 E2B SDK 句柄、远程工作目录和最终沙箱处置，使两个基础 E2B 提供方（`fs-e2b` 与 `subprocess-e2b`）处于**同一个 Linux 运行时**中。

也就是说，文件操作和命令执行不是两个各跑各的远程进程，而是**共用一个沙箱生命周期**——SDK 句柄共享、工作目录一致、沙箱销毁时一起回收。这正是「统一沙箱世界」的工程落地。

## 四、与 nanobot 的对照

| 维度 | nanobot | DeepSeek Harness |
|---|---|---|
| 加能力 | 实现接口、注册进 registry | 换 Provider / patch 配置 |
| 改沙箱 | 各自接远程逻辑 | 指 seams 到同一沙箱，批量搬 |
| 隔离粒度 | workspace 路径边界 | seam + sandboxPolicy 统一根 |

nanobot 的方式直观、好懂；Harness 的方式在「要频繁换底座 / 要远程沙箱」时优势明显——你不必改几十个工具，只要动几个 seam 的实现。

## 五、小结

Harness 的扩展哲学可以浓缩成一句：**能力的「接口 / 实现 / 消费」三者解耦，切换实现是配置级操作，而统一沙箱世界把这种切换的威力放大到了整个执行面。**

到这三篇，Harness 最独到的三处（插件模型与引擎、工具流水线并发、seam 沙箱）已经讲透。若想收个尾，可以把它们与 nanobot 摆在一起对照——那篇《nanobot vs DeepSeek Harness》已在站内，正好作收束。
