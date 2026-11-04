---
title: "Superagent 源码解读 · 04 | 执行与面板：工单在等谁"
date: 2026-11-04
slug: superagent-04-execution
draft: false
weight: 904
tags: ["Agent框架", "DeepSeek", "Harness", "Superagent", "源码解读", "执行"]
summary: "一张 Superagent 工单对应一个 Worker Session 和一个目录。阶段按「在等谁」计算，不按模型的自由文本。本文顺着派工、槽、五秒转后台、结束条件、先落库再发送，把面板的三问和一条 GitHub 评论走完。"
ShowToc: true
---

[上一篇]({{< relref "superagent-03-authorization.md" >}})在 `accept` 时留下了一张工单、一条 Mandate 和一次 pending 的派工。这篇跟着它走完。文档是 `docs/003-execution.md` 和 `docs/005-panel.md`，执行器在 `src/executor/dsh.ts`。

## 一、两个 Session

主 Session 的预设是 `dsh-superagent-router`。工具少，负责评估和派工。它不读仓库的 `AGENTS.md`，也不做技术决定。

每张工单得到一个名为 `worker-<id>` 的 Executor Session，目录在执行根下，默认是状态目录里的 `work/<短名>`。Worker 预设是 `dsh-superagent-worker`。Worker 再拉起的子代理用 Harness 的 `standard` 预设，只保留这份工作允许的工具，工单结束就收回。

Worker 以操作者的操作系统用户跑命令，不逐条审批。前台 shell 超过 5 秒就转后台，这样下一步还能收到新消息。子代理的 shell 是 60 秒后转后台。Executor 的工具集里拿掉了会向人提问、向人展示界面或修改 profile 的工具，例如 `ask_user_question`、`present`、`plugin_manager`。要问人，就在 Report 里放一个结构化问题，由主 Session 原样转发。

环境变量只描述仓库、账号、目录、工单 id、禁止的命令名，不含凭据。prepare 命令从环境变量读，Executor 在提示里读同样的值。Worker 通过代理账号的 `gh` 和机器人的 `lark-cli` 访问外部。按名字拦截的命令挡得住。文档写明这些包装和目录是默认约定：Worker 仍可以用绝对路径执行程序。只有面板能改执行环境；发给主 Session 的文字改不了配置好的 shell 命令。

和 [PocketPilot]({{< relref "pocketpilot-03-policy.md" >}}) 对照着记：那边有副作用的工具要停下来等批准；这边的产品论题是团队仓库上的少打扰，命令在操作者的用户下直接跑。你要收紧，就收白名单、接受模式、禁止命令和 prepare，而不是指望逐条审批卡片。

![主 Session 与 Worker 槽](/images/superagent-04-execution.svg)

## 二、阶段：打开时它在等谁

打开的工单在任一时刻只有一个阶段。表从上到下，第一条命中的算数。`person` 表示没有人就继续不了。

| 阶段 | 条件 | 面板 |
|------|------|------|
| 等人 / 投递 | Input 没送到 Session，按 2 秒起、加倍到一分钟重试 | Input undelivered |
| 等人 / 回答 | 有一条 Attention | Awaiting answer |
| 等人 / 评审 | 最新 Report 是 `ready` | Awaiting review |
| 等人 / 卡住 | 最新 Report 是 `blocked` 或 `failed`，之后没有人的消息 | Blocked · Failed |
| 工作 / 排队 | 没有派工，或派工还是 pending | Queued |
| 空闲 | 最新 Report 是 `waiting`，或这次派工已结束 | Waiting for a Trigger |
| 工作 | 其余 | Working |

人的新消息会把 `blocked` 或 `failed` 交回 Executor，通常回到 working。`idle` 表示这一轮做完了，没有悬着的问题；订阅线程上的新 Trigger 可以把它叫醒。

Report 的取值是：`progress`、`waiting`、`ready`、`blocked`、`failed`、`dropped`。回合结束时如果还有 shell 任务或子代理在跑，工单继续占着槽，不报 `waiting`，主 Session 也收不到一份等待报告。完成后台任务会在同一槽里把结果送回。

后台工作最多 30 分钟。到点杀掉 shell，报告 `waiting`，并写明杀掉了哪些、哪些子代理还活着。子代理可以继续，Executor 下一回合再读被杀掉的任务的结果。交给人的时候（`ready`、提问、`blocked`、`failed`）立刻释放槽。交接时子代理停掉。为了给人看而留着的服务器可以继续，不占槽，也不受 30 分钟限制；它完成时不叫醒 Executor。

## 三、槽和准备目录

工单从启动（含 prepare）或从把 idle Executor 叫醒的那次投递开始占槽，整段活跃回合都占着。配置的并发上限管这些槽。人的消息或状态查询多一个额外槽，槽满时也能叫醒已有工单，并且比新开工优先。CI 之类的其他 Input 只能等普通槽。新活挤不掉人正在跟的那张工单。启动可以并行，一条慢的 prepare 只耽误自己。

第一次派工前，如果目录还不存在，就在执行根用 `sh` 跑一次 prepare，限时 10 分钟。常见写法是 `git worktree`，所以机器上要留一份能加 worktree 的本地克隆。没有 prepare 就在空目录里开工。prepare 失败则工单失败，并记下输出；人再说话或重试会再准备一次。PR 的分支路径最后一段等于工单短名时，这张 PR 属于这张工单。

`WorkerSlots` 在启动或叫醒之前占位。短暂的模型或网络失败（HTTP 408、429、5xx，连接被拒、重置或超时）先重试三次。启动走副作用队列的退避；失败的回合会在 15、30、60 秒后收到「继续」的消息。prepare 失败不走这三次。一旦 failed，CI 和 bot 消息只记录，不投递，也不叫醒 Session。工具仍可用，人可以直接在这个 Session 里接着做。

## 四、合并与结束

自动合并不由宿主做判断，写在 Delivery requirements 这段 prompt 里。默认 Executor 不以代理账号打开 auto-merge，因为 GitHub 会把合并算在打开 auto-merge 的那个人头上。要合并，prompt 里得指定另一个账号的 CLI。默认只允许：这张工单自己开的 PR、已经 ready、所需批准覆盖当前 head、增删合计少于 200 行。再推一版之前先撤掉 auto-merge。没有评审正文的批准也会作为 Trigger 到达，可以叫醒正在等评审的 Executor。

这些句子约束的是代理，约束不了 GitHub。要强制批准，就在分支保护里要求批准，并让过期的批准失效。拿掉合并 CLI 或删掉 auto-merge 那段，就回到人工合并。

工单结束有三条路：

- 没有 PR，Report 为 `ready`：完成并结束。
- 发现没什么要人做的（复现不了、已被取代）：`dropped`。默认不发消息，除非命中的规则要求发。人之后再说话可以恢复。
- 跟了 PR：它开出的 PR 都不再处于打开状态时结束。若它一张都没开，则它跟随的 PR 都关闭时结束。结果取最后一次相关关闭。结束后仍保留订阅，人再评论可以在同一张 Mandate 下恢复。

一条关闭不会在这些情况里结束工单：还有未完成的恢复、重试或收尾；Executor 已经确认或回答了自己负责的跟进，但之后还没有一份非 `progress` 的 Report，也还没有在读完那条消息之后结束回合。聊天渠道发一条最终通知。GitHub 上 PR 自己已经显示结果，所以不再多说一遍。去掉对某张 PR 的订阅，也就不再跟随它。

因此面板上的「Awaiting review」就是：一份 `ready`，并且还有 PR。

只为了上下文而跟随的线程仍然投递消息，但不把人的跟进所有权交给这张工单，也不能恢复已结束的工单。那些消息可以单独评估。来源在有活工单订阅期间轮询这些线程，并在「最后一条 Trigger」和「订阅工单结束」里较晚的那个时刻之后再轮询两周。之后只走普通入口：GitHub 通知或飞书实时事件。

## 五、失败时先落库

`Effects` 里和执行有关的种类：

| 种类 | 谁处理 | 做什么 |
|------|--------|--------|
| `assessment` | `MasterAssessor` | 把候选卡片送进主 Session，送达后开始评估的截止时间 |
| `launch` | `DshExecutor` | prepare、建 Executor Session、派工消息 |
| `delivery` | `DshExecutor` | 按顺序把等待中的 Input 交给正在跑的派工 |
| `relay` | Sessions | 把 Worker 的报告、问题或人的跟进送进主 Session，保持顺序 |
| `notice` | Sessions | 注入通知，不叫醒主 Session，保持顺序 |
| `reply` | Outbox | 经来源把渠道消息发出去 |

某一类连续失败三次，才出现在总览的系统问题里。`delivery` 例外，失败显示在工单上。`relay` 和 `notice` 各保持自己的顺序，前面一条失败会挡住后面的。

出站消息先存储再发送。同一张工单、同一个对话里按队列顺序发，确认一定在后面的答复之前。退避从 5 秒加倍到 300 秒。其他工单的消息不跟它一起等。来源不再听这个对话，或对方永远收不到（删除的 Issue、锁住的对话、飞书已撤回或找不到的消息），才丢弃，并留下时间和原因，不再重排。回复一条已经不存在的评审评论时，改发到 PR 的主对话。

Retry 在投递失败、或卡在 `blocked` / `failed` 时可用。它结束当前派工，在同一个 Session、目录、分支、PR 上开一次新的。模型只在创建 Session 时选定，已有 Session 不换模型。人的消息已经送到之后，面板不再提供 Retry，除非这条消息本身没送到。

进程重启：正在跑的派工以「host restarted」结束，新的带着恢复说明再开始，要求先看目录、git 历史和 GitHub，不要重复外部动作，旧工单优先。`waiting`、`ready`、`blocked`、`failed` 保持原样，除非那份 Report 之后又来了 Input。Session、目录、订阅、待发送消息都留下。一次 step 在模型调用前已经认领、但被取消或错误打断、还没进历史的消息，仍算未读，会重新投递；重启前已经写进历史的，算读过。

插件在 Harness 进程里停止或重启，走同一条关闭路径。还没进 Session 历史的 Input 保留。关闭时每个 Worker 的后台工作最多等 10 秒；进程重启会直接结束这些工作。

暂停处理会停掉派工、新 Eval、评估超时、Input 投递和回复。已经在跑的回合不打断。来源继续记 Trigger。暂停期间评估不超时，截止时间往后推。暂停恢复后，人在已结束工单上的消息会先按当时的策略尝试恢复。

来源读失败时，GitHub 对每个 Issue、PR、线程单独记错误，下一轮再读。私有仓库失去权限时 GitHub 也会 404，所以 404 会重试。连续失败默认三次后，来源报错；任何一次成功就把计数清零。通知过、却一直读失败的 Issue 或 PR，在首次失败一天后丢掉，GitHub 报已删除则立刻丢掉。读位置留在原地，后面补得上漏掉的内容。列表里格式坏掉的一条会被跳过并计数。

## 六、面板只回答三问

打开面板的顺序写在 `docs/005-panel.md`：

1. 现在有没有事需要我？
2. 系统是否健康，做的是不是对的事？
3. 这一张工单从 Trigger 到 Eval 到 Ticket 的历史是什么？

页签按这个领域排：Overview、Source、Trigger、Eval、Execution、Settings。仓库和账号还没配对时，Overview 只显示缺什么，并链到对应设置。Overview 汇总过去 24 小时的活动和 Executor 负载，链到详情，自己不再变成第二份详情。

计数必须等于它链到的列表长度。健康状态不占额外空间。一次网络抖动不会升格成问题。需要人的那一行直接给出动作，破坏性操作要确认。页内说明保持一句，其余放在问号提示里。

Settings 按概念分组：来源、评估（策略版本、规则、允许名单、准入表、接受模式、超时、评估模型）、执行（Worker 模型、并发、目录根、prepare、禁止命令）、提示词、部署。每一组有自己的保存条，只在有改动、保存中或保存失败时出现。DeepSeek Harness 的插件管理器打开的是同一页。

搜索能按文本、id、飞书消息 id 或存下来的链接找 Trigger，按 `#12` 或 GitHub URL 找线程，按标题找工单。一条仍由 Executor 拥有的人的跟进，标成 Delivered to Ticket，旁边没有 Eval。

## 七、把 #12 走完

Alice 的 `accept` 已经落库。

1. 派工为 pending，阶段是 Queued。槽空出来之后 `launch` 运行。prepare 用 `git worktree` 做出目录。`worker-<id>` 开始干活，阶段变成 Working。Issue 里只有一句确认，过程中保持安静。
2. 她做完，开出 PR，Report 为 `ready`。阶段变成 Awaiting review，槽释放。给评审看的预览服务可以留着，完成时不叫醒 Worker。
3. 评审评论经订阅扇出，不再评估，不新增权限。Worker 被叫醒，仍在原来的 Mandate 里改。再推一版之前，若开过 auto-merge，先撤掉。
4. PR 合并。工单结束，订阅保留。聊天渠道有一条最终通知。GitHub 上不再重复解释合并结果。
5. Alice 之后再在这条线程里说话。来源还在听、她仍在名单上，同一张 Mandate 可以把工单叫醒。恢复说明会要求 Executor 先看目录和 git 历史。
6. 若是名单外的人、模式仍是 `observe`：第三篇的「本来会接」留在 Eval 页，Execution 页不会出现工单。

## 八、源码从哪一页翻进去

`docs/007-architecture.md` 是索引，不必从头读。按你卡住的问题跳：

| 你想看 | 文件 |
|--------|------|
| 谁组装进程 | `src/app/runtime.ts`、`src/app/passes.ts` |
| 谁写入工单 | `src/core/engine.ts`、`src/core/ports.ts` |
| 强度和再校验 | `src/policy.ts` |
| 模型研判 | `src/app/assessment.ts` |
| 槽、prepare、Session | `src/executor/dsh.ts` |
| 先落库的队列 | `src/app/effects.ts`、`src/app/outbox.ts` |
| SQLite | `src/app/store.ts` |
| 加一种来源 | `docs/008-cookbook.md`、`src/sources/kind.ts` |

## 九、小结

- 一张工单一个 Worker Session、一个目录。主 Session 只路由。
- 阶段回答「在等谁」。第一条命中的行算数。自由文本改变不了阶段。
- 人的消息有额外的槽，并且比新开工优先。前台命令 5 秒转后台，后台最多 30 分钟。
- 合并条件写在 prompt 里，分支保护写在 GitHub 上。包装目录不是安全边界。
- 发送、投递、评估都是提交之后的副作用。连续失败三次才升格成系统问题。

四篇到这里可以收束。Harness 三篇讲底座怎么换插件，这四篇讲底座上的团队包怎么把「少占注意力」写成 Gate、Mandate、工单阶段和面板。若要看另一种副作用故事，回到 [PocketPilot 的策略与审批]({{< relref "pocketpilot-03-policy.md" >}})。若要从概念课重新串，从 [学习路线图]({{< relref "agent-learning-roadmap.md" >}}) 进。
