---
title: "DeepSeek Harness 源码解读 · 01 | 插件模型与引擎主循环：一切皆插件，连循环也是"
date: 2026-08-23
draft: false
weight: 891
tags: ["Agent框架", "DeepSeek", "Harness", "Cordis", "源码解读", "Agent Runtime"]
summary: "DeepSeek Harness 由 Cordis 元框架驱动，核心是「一切皆插件，没有特权内核」。本文先讲 Cordis 的五个概念与服务容器 ctx，再拆开 agent-loop 这个「只是个插件」的默认驱动器：ReactLoopAgent 的 Phase 状态机、kick/turn/step 主循环、以及三路 abort 融合的生命周期熔断。配插件树与状态机图。"
ShowToc: true
---

## 一、一句话定位

如果 nanobot 是「一个讲得清的分层引擎」，那 DeepSeek Harness 就是「一个谁都能改的插件底座」。它的全部秘密写在两个字里：**Cordis**——一个 vendor 进来的插件元框架。

官方给过一个极端的判断：**连「循环（loop）」本身都是个可卸载的插件**。在 Harness 里没有特权内核，模型、工具、UI、编排、文件系统，包括 `agent-loop`，全都是挂在共享 `ctx` 上的插件。

## 二、Cordis 的五个核心概念

读懂 Harness，先读懂 Cordis 的五件事：

1. **插件是实现 `Service` 的对象**。可以是一个带 `inject` 和 `apply(ctx)` 的函数，也可以是 `Service` 子类；生命周期由 Cordis 挂载到当前上下文。
2. **上下文是服务的容器**。一个服务占据稳定的 `ctx.<key>`（如 `ctx.tools`、`ctx.llm`、`ctx.sessions`）；其他插件通过 key 查找，而非导入具体实现。
3. **通过 `inject` 声明依赖**。插件声明所需服务后，会等它们就绪才启动；加载顺序由依赖表达，而非手动编排。
4. **类型化事件用于通信**。服务通过 `emit` / `waterfall` / `parallel` / `serial` 分发——分别应对观察、环绕包装、并行扇出、按序执行。
5. **注册是可逆的副作用**。`ctx.effect()` 或 `ctx.on()` 安装的提示词片段、工具 schema、监听器，在 reload / teardown 时按预期撤销。

> 关键心智：**你从不直接 `import` 一个具体实现，只向 `ctx` 要一个能力**。换实现 = 换插件，消费方一行不动。

![Cordis 插件树与 agent-loop](/images/harness-01-plugin-engine.svg)

## 三、agent-loop：一个「只是插件」的驱动器

`capability-seams` 文档里把 `ctx.agentLoop` 的角色标成 **`bundle`**，附注写得直白：

> 唯一的具体循环插件；扩展包依赖 dsh-agent 的事件和服务，而不依赖此包。

也就是说，Harness 默认只提供**一个**循环实现（`ReactLoopAgent`），但架构上它和「工具」「LLM」平级，随时可被替换。

### 3.1 主循环：kick → turn → step

`agent.ts` 里的驱动器入口极干净：

```ts
// packages/core/agent-loop/src/agent.ts
private async kick(): Promise<void> {
  try { while (await this.turn()) {} }
  catch { /* 失败在驱动器边界被收容 */ }
}
```

- `turn()` 发射 `turn/start` → 跑若干 `step()` → 自然停止时发射 `turn/stopping`（串行终检点）→ `turn/end`；
- `step()` 经 `agent/request` 瀑布流构造请求、`llm.stream` 逐 chunk 写 `assistant/chunk` 事件，再 `executeToolCalls`；
- 整个过程被一个 **Phase 状态机**（idle / maintenance / running）管着，输入分 `followup` / `steer` / `inject` / `cancel` 四类。

### 3.2 生命周期熔断：三路 abort 融合

最见功力的是「卸载不能泄漏 agent」。`index.ts` 把三路 abort 用 `AbortSignal.any` **融合（fused）成一束**：

- 调用方主动取消；
- owner fiber 卸载（插件被卸）；
- 工厂 teardown。

任意一路触发，其余两路也被一并点燃，保证正在跑的 agent 一定会被干净回收。这是生产级运行时才在意、教学型引擎常常忽略的细节。

## 四、小结

Harness 的第一课：**它不是一个「框架里有循环」，而是「循环也只是框架里的一个插件」**。理解了 Cordis 的服务容器与依赖注入，你就拿到了打开整个代码库的钥匙——后面读工具、读会话、读沙箱，都是在 `ctx.<key>` 之间穿行。

下一篇我们顺着 `ctx.tools` 往下，看它的工具执行流水线如何在「不改循环」的前提下，把钩子、权限、沙箱、并发编排全塞进去。
