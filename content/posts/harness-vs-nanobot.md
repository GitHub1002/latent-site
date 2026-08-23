---
title: "DeepSeek Harness vs nanobot：两种 Agent Runtime 设计哲学"
date: 2026-08-23
draft: false
weight: 900
tags: ["Agent框架", "DeepSeek", "nanobot", "Agent Runtime", "源码解读", "架构对比"]
summary: "刚发布的 DeepSeek Harness 与已精读的 nanobot 恰好构成一对对照样本：一个用 Cordis 插件树把 Agent 做成可无限替换的生产基座，一个用分层引擎把 Agent 讲透给你看。本文从内核组织、主循环、上下文真相源、工具并发、扩展方式、多 Agent 六个维度逐一对撞，并给出选型建议。配对照架构图。"
ShowToc: true
---

## 一、为什么把这两个摆在一起

前面我们把 nanobot 从头到尾精读过一遍——一个 **Python、分层清晰、可读性强**的 Agent Runtime，适合「把 Agent 讲透给你看」。而刚发布的 **DeepSeek Harness**（MIT，pnpm + TypeScript monorepo，由 Cordis 元框架驱动）走的是另一条路：把 Agent 做成**可无限替换的生产基座**，连「循环」本身都是个可卸载的插件。

两者恰好互补，构成一张顶配的 **Agent Runtime 知识地图**：

- 它们都是「运行框架」——owns 主循环、状态、工具执行、生命周期；
- 它们解决的是同一组问题（循环怎么转、上下文从哪来、工具怎么管、怎么扩展）；
- 但**组织哲学完全相反**：一个「分层」，一个「插件树」。

读懂这一对，你对「Agent Runtime 到底有几种长法」这件事，基本就毕业了。

![两种 Agent Runtime 对照架构](/images/harness-vs-nanobot-arch.svg)

---

## 二、内核组织：分层 vs 插件树

**nanobot —— 分层，但有清晰边界。**
它把「产品」和「引擎」切成两层，用 `AgentRunSpec` 解耦：

- `AgentLoop`（产品层）：面向「一回合」，管会话、权限、装配、持久化、流式；
- `AgentRunner`（引擎层）：面向「模型循环」，管 while、调模型、解析动作、停止判定。

好处是**心智模型简单**，你顺着调用栈往下读就能懂。代价是：要换掉某一层的行为，得进到对应模块改代码。

**DeepSeek Harness —— 没有特权内核，一切皆插件。**
它的骨架是 **Cordis 插件树**：所有能力（模型、工具、技能、UI、循环、编排、文件系统）都作为插件挂在共享 `ctx` 上。连 `agent-loop` 本身也只是个插件（`ctx.agentLoop`）。扩展产品的方式只有一种：把插件挂到别的插件旁边；卸载时副作用自动撤销。

代价是**上手门槛高**——你得先理解 Cordis 的插件生命周期与依赖注入；好处是**替换任意一层都是配置级操作**，不用碰核心代码。

> 一句话：nanobot 是「一个讲得清的引擎」；Harness 是「一个谁都能改的底座」。

---

## 三、主循环：while + 治理 vs Phase 状态机 + 事件流

两者都跑「调模型 → 读结果 → 调工具 → 再调模型」，但形状不同。

**nanobot 的引擎层（runner）是经典 while：**
```python
# agent/runner.py（引擎层，代表性伪代码）
while not done:
    messages = context_builder.assemble()   # 每轮重新装配上下文
    governor.sanitize(messages)             # 9 步治理：剥占位符/补缺失结果/裁剪
    resp = await llm.astream(messages)      # 流式调用
    action = parser.parse(resp)             # 解析 tool_call / 停止
    if action == STOP: break
    result = tool_registry.execute(action)  # 执行并把结果落盘/在飞压缩
    context_builder.append(result)
```
特征是**「每轮重建上下文 + 治理」**，循环逻辑集中、易读。

**Harness 的 `ReactLoopAgent` 是 Phase 状态机 + 事件驱动：**
```ts
// packages/core/agent-loop/src/agent.ts（节选）
private async kick(): Promise<void> {
  try { while (await this.turn()) {} }
  catch { /* 失败在驱动器边界被收容 */ }
}
```
`tun()` 发射 `turn/start` → 跑若干 `step()` → 自然停止时发射 `turn/stopping`（串行终检点）→ `turn/end`；`step()` 经 `agent/request` 瀑布流构造请求、`llm.stream` 逐 chunk 写 `assistant/chunk` 事件、再 `executeToolCalls`。

特征是**「一切皆事件，循环是状态机的一种形态」**。更妙的是它的生命周期熔断：把「调用方取消 + owner fiber 卸载 + 工厂 teardown」三路 abort 用 `AbortSignal.any` **融合（fused）成一束**，保证任何时刻卸载都不会泄漏 agent。

> 一句对照：nanobot 的循环是「一段你看得懂的 while」；Harness 的循环是「一台用事件和状态机驱动的机器」。

---

## 四、上下文真相源：每轮治理 vs 日志投影

这是两者**哲学差异最深**的地方。

**nanobot：上下文是「每轮现装」的。**
`ContextBuilder` 负责装配一次，`ContextGovernor` 每轮做 9 步治理（剥占位符、剔畸形 tool_call、补缺失结果、结果落盘、在飞压缩、历史裁剪……）。模型看到的 messages 是治理后的产物，**治理逻辑是显式的、可读的**。

**Harness：上下文是「日志投影」出来的。**
它有一条**仅追加的 `SessionEvent` 日志**，这是模型上下文的**唯一真相源**。`deriveMessages()` 从日志**投影**出模型历史。于是它立下一条硬性不变量：

> **「任何模型可见的新输入，都必须新增一个 `SessionEvent`」** —— 否则 replay 就会失真。

这跟 nanobot 的「Governor 每轮从日志重建」同源，但 Harness 把它**上升成了编译期/运行期都要守护的约束**。好处是天然支持重放、审计、断点恢复；代价是你必须时刻记得「可见即记录」。

> 一句对照：nanobot 用「治理函数」保证上下文正确；Harness 用「日志不变量」保证上下文正确。

---

## 五、工具与安全：三抽象 vs 作用域注册表 + 并发调度

**nanobot：Tool/Schema/ToolResult 三抽象 + 双边界。**
工具用 `Tool` 抽象统一注册，`ToolRegistry`/`ToolLoader` 通过 `pkgutil` + `entry_points` 做插件发现；安全上有两条线——**SSRF 三防线**（IP 黑名单 / PinnedDNS / 逐跳重定向重验）和 **workspace 路径边界**（工具只能碰授权目录）。

**Harness：把工具做成「带把关的流水线」。**
`ctx.tools`（约 5600 行，是它最大的核心包）是**作用域化（按 agent 划分）的注册表**，外加一条执行流水线。最精彩的是 `tool-calls.ts` 的**并发调度**：
- 维护 `maxParallelToolCalls` **有界滚动池**；
- **后到的 call 会重新分类**——一旦某 call 不是 parallel，就在此开一道新屏障（exclusive 独占屏障）；
- `commitReady` **只沿模型序连续提交**已就绪的结果，保证顺序与模型意图一致；
- abort 语义很讲究：已开始的调用照常提交，未开始的写合成错误 `TOOL_ABORTED_BEFORE_DISPATCH`，目的是**让 replay 仍然有效**。

> 一句对照：nanobot 的工具亮点是「安全边界清晰」；Harness 的工具亮点是「并发编排精密」。

---

## 六、扩展方式：模块内注册 vs seam + 配置替换

**nanobot：扩展靠「往模块里注册」。**
想加能力，就实现接口、注册进对应 registry。直观，但行为改动散落在代码里。

**Harness：扩展靠「能力 seam + 配置 patch」。**
每个能力由三角色构成——**Service Definition（接口）+ Service Provider（实现）+ Consumer（用它的地方，通常是模型工具）**。换一个 Provider，整个产品行为就变了；而且通过 `cordis.patch.yml` 按条目 id 做**替换/插入**，改配置就能换掉任意一层行为。更绝的是 `fs`/`subprocess`/`terminal`/`lsp` 共享同一个「执行世界」——把它们指向远程沙箱，Bash、PTY、LSP 一次性全搬过去。

> 一句对照：nanobot 的扩展是「加代码」；Harness 的扩展是「换插件 / 改配置」。

---

## 七、多 Agent 与产品化（简述）

- **nanobot**：`SubagentManager` 做委派，四个表面（Channel / API / SDK / Cron）最终**汇流到 MessageBus + AgentLoop**——你懂了核心，外围只是薄适配。
- **Harness**：多智能体是「编排插件」，Web UI 也只是插件树上的一个 `web` profile；`npx @deepseek-ai/dsh web` 一条命令起本地自托管界面，直接对标 Claude Code 的开源替代。

两者都验证了同一结论：**运行框架的护城河不在「能调模型」，而在「循环可恢复、上下文可信、工具可管、扩展可替换」**。

---

## 八、选型建议：什么时候选哪个

| 场景 | 选 nanobot 式（分层） | 选 Harness 式（插件树） |
|---|---|---|
| 想**读懂**一个 Agent 怎么跑 | ✅ 调用栈清晰、可逐行精读 | ⚠️ 需先懂 Cordis 元框架 |
| 想**快速改行为 / 换模型** | ⚠️ 进模块改代码 | ✅ 换插件 / patch 配置 |
| 要**生产级并发 + 重放 + 熔断** | ⚠️ 需自己补 | ✅ 内置（有界池 / replay 不变量 / fused abort） |
| 做**教学 / 博客 / 面试作品集** | ✅ 把原理讲透 | ✅ 展示工业级架构功底 |
| 版本稳定性 | ✅ 已成型 | ⚠️ rc 阶段，API 会变动 |

**给求职者的实话**：nanobot 适合用来「把原理讲清楚」（面试时能说清每一行）；Harness 适合用来「证明你能啃生产级 TS monorepo、理解插件化架构」。两者都吃透，你对 Agent Runtime 的理解就既「深」又「广」。

---

## 九、小结

nanobot 和 DeepSeek Harness，一个用**分层引擎**把 Agent 讲透给你看，一个用**插件树**把 Agent 做成可无限替换的基座。它们回答了同一个问题——「怎么让模型长出手脚」——却给出了两种气质迥异的答案。

对你而言，这张对照图本身就是一份资产：它把「Agent Runtime 有几种长法」这件事，从一个模糊的概念，变成了一张可以画在白板上、讲给面试官听的知识地图。

> 下一步可选：若想继续深入 Harness，它真正独到的三处（引擎状态机 / 工具并发调度 / seam 沙箱）值得单独拆解；但概念上，本文已覆盖它与 nanobot 的全部关键差异。
