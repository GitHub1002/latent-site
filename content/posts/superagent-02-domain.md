---
title: "Superagent 源码解读 · 02 | 领域模型：从一条评论到一张工单"
date: 2026-11-04
slug: superagent-02-domain
draft: false
weight: 902
tags: ["Agent框架", "DeepSeek", "Harness", "Superagent", "源码解读", "领域模型"]
summary: "Superagent 把「一条被接住的活」写成一组可审计的对象：Trigger、Thread、Eval、Mandate、Ticket、Assignment、Subscription。人不是领域对象。本文按对象、按明确排除的概念、按 Engine 的写权往下读，并用一条 GitHub 评论走到开出工单。"
ShowToc: true
---

[上一篇]({{< relref "superagent-01-attention.md" >}})把原则钉死了。原则要能执行，就得有一组边界清楚的对象。仓库把这组对象写在 `docs/001-domain.md`，实现写在 `src/core/engine.ts`。读的时候先记一句：

> 工单是已授权的工作。Session 是干活的进程。两者分开。

## 一、人不是领域对象

GitHub 登录名、飞书用户 id 只是 Trigger 上的元数据。允许名单是某个来源的策略的一部分，用那个来源自己的身份。同一个人在 GitHub 和飞书上是两个账号。团队如果需要对照，写成 prompt 里的技能（例如 `identities`），由模型在需要时加载。宿主不去建一张跨平台的「这个人」表。

这样做的直接后果是：授权永远落在「这条消息、这个来源、这份策略版本」上，可以重放。换了一个平台的账号，就走一次新的判断，而不是靠一张映射表继承权限。

## 二、一条被接住的活

```text
Source 产生 Trigger
  → 归入 Thread
  → 若是已有工单订阅的后续讨论：扇出，不再评估
  → 否则进入 Eval（先 Gate，再 Assessment）
  → accept 时发出 Mandate，创建 Ticket
  → Assignment 把 Ticket 交给一个 Executor
  → 结果经 Channel 写回原来的 Source
```

![从 Source 到 Executor](/images/superagent-02-domain.svg)

GitHub 这个来源实际听的是：通知、已订阅的 Issue 和 PR、CI 变化、PR 合并或关闭。开关打开后，还可以听仓库里所有新 Issue、新 PR、评论，以及默认分支的 CI。飞书听实时事件、历史补齐，以及已订阅话题的轮询。来源记住自己的读位置，重启不重放。它丢掉机器人自己发的消息，也丢掉监听范围外的事件。

## 三、每个对象守什么

| 概念 | 文档用词 | 它守的规则 |
|------|----------|------------|
| Source | 来源 | 事件流。有读位置和机器人身份。 |
| Trigger | 触发 | 一条不可变事件。同一个 Trigger 接两次，只产生一张工单。 |
| Thread | 线程 | 按时间排的 Trigger。同一时刻最多跑一个 Eval。一条 Thread 上可以有多张工单。 |
| Correlation | 关联 | 同一条 Trigger 属于多条 Thread 时的关系。只加上下文。 |
| Channel | 渠道 | Thread 的写侧。确认、回复、PR、解决评审，都是写。 |
| Policy | 策略 | 带版本。包含规则、接受模式、每个来源的允许名单、准入表、机器人名单。 |
| Eval | 评估 | 对一批 Trigger 做判断。无状态。两层：Gate 是确定性准入，Assessment 是模型研判。 |
| Decision | 决定 | `accept` / `stop` / `ignore` / `unavailable`。 |
| Mandate | 授权 | `accept` 发出的凭证：策略版本、范围、强度、命中的规则、发送者身份。不可变。范围要变大，就开新工单。 |
| Ticket | 工单 | 一份已授权的工作。只有 Eval 能创建。自己的生命周期，不跟着 Session 走。 |
| Assignment | 派工 | 一张工单在某一段时间绑定到一个 Executor 实例。前后可以有多次派工。 |
| Subscription | 订阅 | 工单和线程的多对多。后续消息扇出给已订阅的工单，不再过 Gate。 |
| Executor | 执行器 | 领域外面的端口。一张工单一个 Session、一个目录。 |

Decision 的四个值值得先背下来，第三篇会反复用：

- `accept`：发出 Mandate，创建工单。
- `stop`：停止已有工作。
- `ignore`：不动作。观察模式下「本来会接」也记在这里。
- `unavailable`：这次判断没有做成，例如评估超时或模型不可用。面板上可以把它标成已处理（`dismiss`），决定本身还留着。

## 四、明确不放进模型的东西

`001` 用一整节写「不建什么」。这些比对象表更值得记，因为它们是常见 Agent 框架会顺手加、这里故意拿掉的：

- **不建主体 / 成员实体**，也不跨来源合并身份。见第一节。
- **Eval 不自己定时。** 「没人认领超过 N 小时」这种条件，必须是一个会自己再检查的来源。评估器没有闹钟。
- **一条 Thread 不限制只能有一张活着的工单。** 重复由评估时看现有工单列表来避免。
- **没有「追加到已有工单」这种 Decision。** 追加就是订阅扇出。主 Session 的 `attach` 是把一条 Trigger 路由到它明确指向的工单，并结束当前这次 Eval；这是路由，不是第三种决定。
- **订阅不做准入检查。** 订阅覆盖整条 Thread。权限边界在 Mandate 上。订阅和主 Session 转发的 note 都不授予新权限。
- **工单状态不跟随 DSH Session。** 工单可以交接、恢复、换 Executor。Session 挂了，工单还在。
- **Executor 不能宣布「人让我停」。** 停不停由评估或面板命令决定，强度在第三篇。
- **策略变了，不取消正在跑的工单。** 变更记下来，作为 Input 送给受影响的工单。进行中的 Mandate 不会因此立刻作废。
- **不从报告的自由文本里推断「在等人」。** 等人必须是结构化字段。第四篇的阶段表就是按这个字段算的。

最后一条尤其要记住。很多代理把模型输出里的「我在等你确认」当成状态机。这里的状态机只认 Report 和 Attention。自由文本可以给人看，不能推动阶段。

## 五、代码里谁有写权

`Runtime`（`src/app/runtime.ts`）是组装根，不含领域规则。它锁住状态目录、打开 Store、建好各组件；构造失败会关掉库并释放锁。启动时注册管理路由、跑 Setup、装上 Session 和 Executor 的钩子、保证主 Session 存在、开始策略同步、恢复被打断的派工，然后开始周期性的 pass 和副作用队列。

`Engine`（`src/core/engine.ts`）是 Trigger、Thread、Decision、Ticket、Assignment 的唯一写入者。方法是同步的。每条命令在一个事务里改聚合。`evaluateDue` 对每个 Eval 或每张工单用单独的事务。端口在事务里把外部工作放进队列，适配器在提交之后才真正做。非法请求抛 `UserError`。

Engine 把命令分给五个部分，依赖单向：

| 部分 | 负责 |
|------|------|
| `Intake` | 来源的 Trigger：存一次，记到线程上，扇出给已订阅工单；除非是已经送到某张工单的人的消息，否则在主线程上排队等评估 |
| `Evals` | 分批、Gate 的强度、截止时间、过期重排、策略变更后重排，以及最终的四种 Decision |
| `Tickets` | 从 accept 创建工单、Mandate、订阅和第一次派工；恢复、重试、停止、结束；策略变更通知 |
| `Assignments` | Executor 报告：派工已开始、Input 送到或没送到、重启打断、Worker 的 Report |
| `Replies` | 经 Outbox 写渠道：确认、回答、结果、需要人注意的消息 |

调用方拿到的是按角色裁过的接口：来源、主 Session、人、设置、Executor 各看各的方法。来源只能 `receive`。人可以在面板上停工单、重试、要求重新评估。主 Session 可以 `decide` 和 `attach`，不能直接改允许名单。

Store（`src/app/store.ts`）是一份 SQLite。运行时先拿目录锁再打开，避免第二个进程迁移或写同一份状态。领域仓库负责 triggers、threads、tickets、decisions。读查询不写。设置和 prompt 用带编号的版本。

副作用不跟事务混在一起。`Effects`（`src/app/effects.ts`）是持久队列：命令在事务里 `enqueue`，提交之后才执行。同一种类、同一把 key 只有一行；还在等待时再次入队会换掉载荷，并立刻到期，但不改变队列顺序。失败按指数退避重试。抛 `Later` 表示推迟且不计失败，例如当前没有 Worker 槽。第四篇的「先落库再发送」就是这张表。

主循环的节拍在 `src/app/passes.ts`：一次只跑一个 pass，每两秒一次，也在有命令时跑。顺序是先处理到期的副作用，再 `Engine.evaluateDue`。来源的 Trigger 经这个 pass 进 Engine。

## 六、用一条评论走到开出工单

允许名单上的 Alice 在 Issue #12 写：`@agent 给登录加个超时`。

1. GitHub 来源产生一条 Trigger，标成 `mention`（点名），归入这条 Issue 的 Thread。同一条事件再投递一次，Engine 回答 `duplicate`，不会有第二张工单。
2. 这条 Thread 上还没有订阅它的活工单，所以它排队等 Eval。同一时刻这条 Thread 只会有一个 Eval。
3. Gate 看到：名单上的人，并且点了名。强度是 `explicit`。接受模式是什么，到这一步还无关。第三篇解释为什么。
4. 主 Session 做出 `accept`。这一次写入是原子的：Mandate、Ticket、订阅、第一次 Assignment、给 Alice 的确认，一起落库。
5. 确认经 Outbox 排队，回到 Issue #12。干活还没开始。目录、Worker Session、PR，都是第四篇的事。

如果同一条 Issue 上已经有一张活工单订着它，Alice 的下一条评论走另一条路：扇出成那张工单的 Input，不再做一次 Eval，也不签发新的 Mandate。两条路的差别是第三篇的主题。

## 七、小结

- 人是 Trigger 上的元数据。权限落在来源、策略版本和 Mandate 上。
- 工单和 Session 分开。只有 Eval 能创建工单。订阅扇出不授予新权限。
- Engine 是唯一写入者。外部动作先进副作用队列，提交之后才做。
- 状态机不读自由文本。等人必须是结构化字段。

下一篇把第 6 节的 Gate 拆开：强度、三种接受模式、Mandate 在策略变更之后还算不算数，以及 prompt 为什么改不了这扇门。
