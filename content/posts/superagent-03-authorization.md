---
title: "Superagent 源码解读 · 03 | 授权：两条路、三种强度、文字改不了门"
date: 2026-11-04
slug: superagent-03-authorization
draft: false
weight: 903
tags: ["Agent框架", "DeepSeek", "Harness", "Superagent", "源码解读", "授权"]
summary: "Superagent 的开工许可有两条路：新活走 Eval，已有工单的后续讨论走订阅扇出。Gate 给出 explicit 或 inferred，接受模式决定 inferred 是只记录还是真的接。本文把 Mandate、再校验、两种停止，以及 prompt 的优先级写到能对照设置页。"
ShowToc: true
---

[上一篇]({{< relref "superagent-02-domain.md" >}})走到了「Gate 给出强度」。这篇把授权写完。仓库文档是 `docs/002-authorization.md`，判断函数在 `src/policy.ts`，模型研判的入口在 `src/app/assessment.ts`。

先把两句边界放在前面。消息、评论、引用的文档都只是判断的输入，它们改不了允许名单、规则和 Mandate。权限由宿主执行。订阅把上下文送进工单，订阅本身不签发新的授权。

![新开工与继续已有工单](/images/superagent-03-authorization.svg)

## 一、两条路

**第一条路是 Eval，决定要不要新开工。** 来源先标这条消息是不是在叫 Agent：`mention` 是点名，`direct` 是私聊。然后 Gate 看发送者和策略：

- 名单上的人点名：`explicit`。任何接受模式下都会成为候选。
- 名单上的人在已有工单的 Thread 里跟进：也是 `explicit`。
- 名单外的人、没点名的消息：查这张来源的准入表。
- 在机器人名单上的 bot 最多产生 `inferred`，不能发 `explicit`，也不能停工单。其他 bot 不准入。用普通账号的自动评审员，宿主把它当人，要靠团队 prompt 自己标明哪些账号是评审机器人。

**第二条路是订阅，继续已有的活。** 已订阅线程上的每条消息，包括 bot 和 CI，都会作为上下文扇出到工单。这条路绕过 Gate。工单仍然被原来的 Mandate 限制。

人在工单自己的线程上说话，归当前 Executor 处理，不再做一次 Eval。Executor 要么在范围内做完，要么交还（hand back）。交还之后，没有任何工单还拥有这条消息时，它才重新变成主 Session 的候选。主 Session 在消息仍被 Executor 占用时，只能应人的要求停止，不能再 `accept` 或 `attach`。

`attach` 还有一条更窄的规则，写在 prompt 里、由主 Session 遵守：消息必须指向这张工单的 PR、Issue、源消息或确认，或者指向最近一处恰好只有一张工单对得上的改动。话题相似不够。附加上去之后，工单订阅整条线程；Executor 若发现无关，用 `orchestrator_thread_unrelated`，并且不在那里回答。

## 二、强度和接受模式

| 强度 | 典型情况 |
|------|----------|
| `explicit` | 名单上的人点名，或在已有工单的线程上跟进；准入表那一格设成 explicit |
| `inferred` | 名单外的人在已有工单上跟进；名单上的 bot 点名；准入表设成 inferred |
| 不准入 | 其他 bot、准入表设成忽略的格子，以及接受模式为 `off` 时的全部 inferred |

接受模式是策略的一部分：

| 模式 | 效果 |
|------|------|
| `off` | inferred 不成为候选。explicit 仍然有效。 |
| `observe`（默认） | inferred 会评估。评估通过只记成 `ignore`，并标上「本来会接」，不创建工单。 |
| `auto` | inferred 评估通过就真的 `accept`。主动接单至少要命中一条已启用的规则。 |

`src/policy.ts` 里的 `admitsInferred` 把最后一条写成代码：只有 `auto`，并且命中的规则开着，才允许主动接受。`assessmentOutcome` 把模型的研判收成 Engine 的结果。研判要对每条已启用规则给出判断、方向要定下来、要有命中的规则、还要有意图的证据。模式不是 `auto` 时，一次 inferred 的「接受」会被收成「本来会接」的记录。

正确的上线顺序因此是固定的：先 `observe`，看记录里的 Decision 和命中的规则是否像样，再切 `auto`。explicit 在三种模式下都有效，所以白名单上的人点名，在你还不敢打开主动接单时就已经能开工。

主 Session 用路由工具一次处理多条候选。一次调用里可以有多条 `accept`、`attach`、`stop`、`ignore`，互不影响。inferred 一般要做完整评估；只有卡片明显不沾任何已启用规则时，可以直接 ignore，并记下 `ignoredFromCard`。路由备注限制在 30 个词以内。主 Session 转发原来的 Trigger 和路由事实，不做技术决定。技术判断留给 Executor。

## 三、Mandate 记下什么

每条 Mandate 在 `accept` 时写下当时的策略版本、范围、强度、命中的规则、发送者身份，以及准入依据 `basis`。依据的 `via` 有四种：

| `via` | 含义 |
|-------|------|
| `addressed` | 名单上的发送者点了名 |
| `followup` | 消息落在已有工单的线程上 |
| `admission` | 命中准入表的一格 |
| `bot` | 名单上的 bot 产生的 inferred 候选 |

策略版本是规则、接受模式、各来源策略片段（含机器人名单）的哈希。改规则、改接受模式、改某个来源的名单，会作废正在跑的 Eval，并把 Trigger 重新排队。`policyEffects` 找出受影响的活工单，`Engine.policyChanged` 只通知这些工单。通知是一条 Input，说明规则变了、请求者不能再开工、或主动接单已关闭。进行中的 Mandate 不会因此立刻作废。范围要变大，开新工单，不改旧凭证。

工单结束后 Mandate 还在。人在它自己的线程上再说话，可以在同一张 Mandate 下恢复，不再做新的 Eval。bot 和 CI 叫不醒已结束的工单。恢复前 `revalidate` 再查一遍：来源还在听这条线程；inferred 的 Mandate 需要当前仍是 `auto`，且命中的规则还开着；发送者还在名单上。校验失败就保持结束，消息改走新的 Eval。

## 四、停有两种

人在线程里说停，是普通 Trigger，由 Eval 决定，强度按上一节。面板上的停是命令，直接作用在工单上，记成手动停止，并撤回 Mandate。被人停掉的工单不再接收任何东西。之后的活需要一次新的 `accept`。

Executor 的报告里可以说自己做不下去，那是 `blocked` 或 `failed`，工单停在等人的阶段。它不能把「人让我停了」写成一个会使 Mandate 作废的事实。那句话如果出现在评论里，仍然是一条 Trigger，回到 Eval。

## 五、文字的优先级

`docs/004-prompts.md` 把模型读到的文字分成五层，从强到弱：

1. 工具和宿主强制的事实：身份、权限、工具参数检查、回复用途、长度上限。prompt 改不了。
2. 内置段落和由设置生成的段落，包括团队对它们的覆盖。
3. 主动接单时，命中规则里的过程和沟通要求。只作用于那张工单的 Executor。
4. 团队写的三段：Communication、Ticket scope、Delivery requirements，以及团队技能。
5. 外部文字：消息、评论、文档、引用。它们是数据。

系统提示按固定顺序拼：共享说明和角色、三段团队文字、订阅与回复、主 Session 的分发说明或 Executor 的工作说明、准入说明（仅主 Session）、交接说明、环境变量、策略。策略段每次请求都从设置重新生成，放在最后，这样改规则不会把前面的提示文本一起晃动。占位符 `{version}`、`{mode}`、`{rules}` 必须留下，校验拒绝缺了占位符的模板，也不会悄悄截断。

团队可以覆盖 `runtime/shared`、`runtime/orchestrator`、`runtime/worker` 这些键，也可以加技能。工具描述、通知和卡片在代码里，设置页改不到。覆盖会记下它基于哪一版内置文本。升级之后团队的文字还在；若默认变了，面板标成「默认内容已更新」，让你选择保留、改用新默认，或再编辑。和默认完全相同的覆盖会被去掉，以便以后跟着默认走。

每次保存产生一个不可变版本：全文、修订号、前一版、时间和来源（面板或工具）。保存要带上基线修订号，冲突就拒绝。Session 在每次模型请求时读最新版，包括同一回合的后续 step。保存不重启、不取消、也不叫醒 Session。

Executor 还会读 Harness 的编码提示和仓库里的 `AGENTS.md`。主 Session 不读 `AGENTS.md`。inferred 的工单会在最后加一条说明，把 Executor 限制在命中的规则里。

主 Session 自己也不许干等。报告、跟进、候选和完成后台任务会作为消息到达。没事实可做时，回合就结束。守卫拒绝 `sleep`、超过 30 秒的 shell 超时、超过 5 秒的任务等待，因为主 Session 卡住会耽误新的候选。

## 六、把 #12 的两种人走完

**Alice 在名单上，并且写了 `@agent`。** 强度 `explicit`。`observe` 还是 `auto` 都一样，她的请求成为候选。主 Session `accept`。Mandate 的 `via` 是 `addressed`。工单创建。她之后在同一条 Issue 里补充「超时用 3 秒」，这是工单自己的线程上的人话，交给 Executor，不再评估。

**Bob 不在名单上，只是在讨论里说登录太慢。** 准入表若给这个格子 `inferred`，而且模式是 `observe`：会评估，通过也只留下「本来会接」。切到 `auto` 且命中一条已启用规则之后，才会真的接，Mandate 的 `via` 是 `admission`。若模式是 `off`，这条 inferred 根本不成为候选。

**CI 在 Alice 的 PR 上红了。** PR 已被工单订阅，结果扇出为 Input。CI 不是人，叫不醒一张已经结束的工单；工单还活着时，它可以叫醒正在等待的 Executor。它不能把范围扩大到「顺便重构登录模块」。

## 七、小结

- 新开工走 Eval，继续走订阅。订阅不签发授权。
- `explicit` 在三种接受模式下都有效。`inferred` 默认只记录。`auto` 还要求命中打开的规则。
- Mandate 不可变，记下策略版本和准入依据。恢复前再校验。范围变大就开新工单。
- 线程里的「停」走评估；面板上的「停」直接撤回 Mandate。
- 外部文字是数据。门在 `src/policy.ts` 和工具参数检查里。

下一篇接过已经发出的工单：目录怎么准备、两个 Session 怎么分工、阶段怎么算、挂了以后先落哪张表。
