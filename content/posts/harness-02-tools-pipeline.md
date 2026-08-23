---
title: "DeepSeek Harness 源码解读 · 02 | 工具执行流水线：循环不变，把关全在流水线里"
date: 2026-08-23
draft: false
weight: 892
tags: ["Agent框架", "DeepSeek", "Harness", "源码解读", "工具系统", "并发调度"]
summary: "工具不是直接执行，而是过一条带把关的流水线：tool/call 事件先于执行被记录，再依次经过 pre-execute 瀑布（钩子·权限·沙箱）、单调守卫、execute 瀑布（超时·重试·指标）、工具本体、post-execute 瀑布、finalizeContent，最终冻结为 tool/result 事件。并发层用独占屏障 + 有界并行池 + 模型序提交编排多工具，abort 时合成 TOOL_ABORTED_BEFORE_DISPATCH 保证 replay 有效。配流水线图。"
ShowToc: true
---

## 一、为什么工具要「过流水线」

在 nanobot 里，工具是 `Tool` 抽象 + 注册表，执行相对直接。Harness 走得更重：它在 `ctx.tools`（约 5600 行，最大的核心包）里把**执行做成了带把关的流水线**。官方对 `ctx.tools` 的定义一句话点题：

> 注册能力，负责 Code Mode 传输，并让调用依次经过**策略前处理、单调守卫、环绕分派、策略后处理和最终结果观测**。

注意主语是「调用依次经过」——循环本身不知道钩子/权限/沙箱的存在，这些都挂在流水线上。这正是 Cordis「拦截和策略优先用事件」的实践规则。

## 二、一条调用的完整旅程

官方文档给了精确顺序（见下图）。从模型吐出 tool-call 到模型看到结果，中间有十步：

![工具执行流水线](/images/harness-02-tools-pipeline.svg)

1. 模型产出 `tool-call` 块；
2. **记录 `tool/call` 事件（执行「前」就落盘）**——这是「模型可见即已记录」不变量的体现；
3. `tools/pre-execute` 瀑布：钩子、权限、**沙箱**；
4. 单调守卫（monotonic guards）：deny / abstain，身份受保护；
5. `ctx.approval` 一次性询问（无人应答 → deny）；
6. `tools/execute` 瀑布：超时、重试、指标（环绕分派）；
7. 工具本体 `execute()`；
8. `tools/post-execute` 瀑布：接受 / 拦截 / 替换 / 补充上下文；
9. `finalizeContent` → `tools/result`（**冻结的权威结果**）；
10. 记录 `tool/result` 事件（**唯一面向模型的产出**）。

几个值得记住的点：

- **`tool/call` 先于执行、`tool/result` 是终态**——两者都是 `SessionEvent`，直接服务「日志即真相源」的设计。
- **守卫与审批分离**：单调守卫是「不得重排序的所有者策略」，而 `ctx.approval` 在守卫之前处理「询问」；审批不可用就 deny，不会卡住循环。
- **`post-execute` 能补上下文**：工具可以在结果外额外注入 user/message，且通过 `additionalContexts` FIFO 保证「调用与结果相邻」。

## 三、并发调度：一个批次里多个工具怎么跑

`tool-calls.ts` 的 `executeToolCalls` 是编排精华，三种语义：

- **独占屏障（exclusive）**：某些调用必须串行独占；
- **有界并行池**：`fillPool` 维护 `maxParallelToolCalls`，后到的 call 会**重新分类**——一旦某 call 不是 parallel，就在此开一道新屏障；
- **模型序提交（commitReady）**：只沿**模型序**连续提交已就绪的 slot（`tools.finalize` / `finish`），保证结果顺序与模型意图一致。

最讲究的是 **abort 语义**：

> 已开始的调用照常提交；未开始的写合成错误结果 `TOOL_ABORTED_BEFORE_DISPATCH`。

为什么要「合成一个错误」而不是静默丢弃？因为**会话日志必须保持可重放（replay）**——如果中途消失一个调用，日志就对不上了。合成错误让 replay 仍然有效。这又把我们拉回那条核心不变量：**可见即记录**。

## 四、小结

Harness 的工具观：**

- 执行不是「调一下函数」，而是一条**不变循环、可变策略**的流水线；
- 并发不是「全并行」，而是「屏障 + 有界池 + 模型序提交」的精密编排；
- 连中止都要为「日志可重放」让路。

下一篇我们看最精彩的一处：**能力 seam 与统一沙箱世界**——为什么换一个 Provider，Bash / PTY / LSP 会一次性全搬走。
