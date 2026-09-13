---
title: "PocketPilot 源码解读 · 04 | 能力全挂同一条总线：工具、记忆、MCP、入口与评测"
date: 2026-09-13
slug: pocketpilot-04-capability-bus
draft: false
weight: 913
tags: ["PocketPilot", "源码解读", "MCP", "技能", "评测"]
summary: "记忆、技能、MCP、cron、webhook 在 PocketPilot 里都不是特权子系统：它们经同一张工具表注册，过同一道策略门，最终仍是一条可审计 Task。评测用 ScriptedProvider 把路径逃逸和拒批锁成 26 个无 LLM 用例。配能力总线图。"
ShowToc: true
---

03 把那道门讲清楚了。这篇证明一件更乏味、也更重要的事：**后来加上的能力，没有为了「好用」再开一扇门。**

> 一句话：`build_tool_registry()` 是唯一注册口；Chat / webhook / cron / CLI 是同一类入口——创建普通 Task。新能力改注册与声明 capability，不改 Agent 循环。

![能力全挂同一条策略总线](/images/pocketpilot-04-capability-bus.svg)

## 一、工具工厂：文件垂直，统计不靠模型目测

`backend/app/tools/factory.py` 按工作区装配工具表。内置能力是一份小清单，不是插件市场：

| 工具 | 能力 | 要点 |
|------|------|------|
| `list_files` | `file:read` | `max_depth` 2，`max_entries` 100 |
| `read_file` | `file:read` | txt/csv/json/md；utf-8 / gb18030；默认 100KB |
| `extract_document` | `file:read` | PDF / DOCX / XLSX，真正解析在 `documents/extractors.py` |
| `analyze_table` | `file:read` | 供应商 / 金额 / 日期；中位数 × `outlier_multiplier`（默认 3） |
| `write_file` | `file:write` | md/csv/xlsx；5MiB；**禁止写入 `memory/`** |
| `memory_write` | `memory:write` | 路径必须落在 `memory/`；1MiB；replace/append |
| `expand_skill` | `skill:read` | 按需展开 `SKILL.md` 正文 |

旗舰 demo 是发票/报价汇总（`demo-workspace/` + 内置 `invoice-summary` 技能）。`analyze_table` 故意做成**确定性统计**：异常金额用中位数和倍率抓，不把整张表丢给模型「你看看哪行不对」。模型负责决定何时调用、以及怎么把结果写成报告；算术和格式解析留在 Python。这是 01 里「任务质量」的具体化——能算的不要生成。

`write_file` 成功后，`TaskService._maybe_record_artifact` 记一条产物元数据。下载接口会再跑一遍工作区根检查（`api/artifacts.py`），避免「事件里写过路径就可以任意取文件」。

操作员文件控制台（`web/app/workspace/page.tsx`）可以浏览、上传、预览。上传**不走** Agent 审批——03 已经把这条账分开了。删工作区行也不会删磁盘文件：元数据走了，目录还在。这是保守，也是脚枪，用的人要知道。

## 二、有界记忆：一块会过期的 Markdown，不是 RAG

PocketPilot 没有向量库。长期记忆是工作区里的文件：

- `memory/MEMORY.md` 主记忆
- 可选 `memory/YYYY-MM-DD.md` 日记
- 注入：`memory/loader.py` 读入，默认上限 **8192 字节**，超了从末尾截（新的留下）
- 注入前加不可信数据前缀（`memory/prompt.py`），避免记忆里的句子被当成系统指令
- 模型要改记忆，只能调 `memory_write`，capability 是 `memory:write`，**走写审批**
- 操作员可以直接 `GET/PUT /api/workspaces/{id}/memory`，同样不经卡片

对照 nanobot 的 Dream：Dream 会合并、沉淀、在心跳里做记忆维护。PocketPilot 选择透明和可审计——你打开文件就能看到 Agent「记住」了什么，WorkspaceGuard 天然罩着这块目录，`write_file` 还被禁止往 `memory/` 偷写。代价是没有自动整理。需要整理时，那是另一次带审批的 `memory_write`，或者你自己改文件。

没有 `history.jsonl`，没有记忆表。会话历史若要进下一轮，走 04 后面的 follow-up 拼接，仍然标成不可信。

## 三、技能：渐进披露，不当成隐藏系统提示

技能是 `SKILL.md`，加载顺序：内置 → `POCKETPILOT_SKILLS_DIR` → 工作区 `skills/`。内置目前就是发票汇总。

system prompt 里只放**目录**（`skills/prompt.py`，默认最多 4096 字节）。模型觉得某条技能相关，再调 `expand_skill` 取正文。这是 progressive disclosure：别把所有手册一次性塞进上下文。`expand_skill` 是 `skill:read`，自动放行——读手册不该弹审批；手册若教它去写文件，真正写的那一下仍会停住。

操作员 CRUD 在 `/api/workspaces/{id}/skills`。远程安装 Agent Plugin、从网上下载技能，明确不做。

## 四、MCP：白名单，而且不准 list_tools

MCP 是最容易把策略故事冲垮的能力。PocketPilot 的约束写得很硬：

1. **工作区 policy 的 `allowed_mcp_servers` 为空，就不注册 MCP。** 全局 `POCKETPILOT_MCP_SERVERS` 不够，还要工作区点头。
2. 白名单含 `workspace-fs` 且设置里没有同名 server 时，注入进程内文件系统（`read_text` / `write_text`），仍过 Guard。
3. stdio server 必须声明 `command`、**每个工具的 capability** 和 `input_fields`。**从不调用 `list_tools`。**
4. 每次调用 **spawn 一个进程**（`cwd=workspace_root`，30 秒超时），不做粘性会话。保守、慢、可预测。
5. 工具名 `mcp_{server}_{tool}`。路径型参数先 `WorkspaceGuard` 再进 MCP。
6. capability 解析：P0 配置声明 → P1 annotations / `readOnlyHint` → P2 启发式 → **P3 不注册**。

未知能力不进表，比进表再 DENY 更干净：模型根本看不到那个函数。评测用例 `mcp_path_escape.json` 锁的就是「MCP 也逃不出工作区」。

对照 nanobot 的 MCP：那边更接近「接上就当本地工具用」，重点在 wrapper、重连、热重载。PocketPilot 的重点是**少接、声明能力、路径仍过牢笼**。两边都对，场景不同。

## 五、会话、通道、cron：入口再多，也是 Task

`ChatSession` 是独立表，不是 messages 表。改标题、归档、空会话都在 session 上；真正的一轮问答仍是 Task。续聊用 `Task.follow_up_of`：`build_followup_goal()` 把有限历史拼进新 goal（约 8KB，旧的丢掉），并标成不可信。cron / webhook 自动创建的会话里，`session_id == task.id`——没有人在聊，也要留一条可点开的审计线索。

通道目前只有 `webhook`。token 展示一次，校验用 `X-PocketPilot-Token` 或 `Authorization: Bearer`，`secrets.compare_digest`。请求体 `{"text": "...", "follow_up_of": optional}`，text ≤ 4000。`POST /api/channels/{id}/webhook` → `create_task` + `start_task`。没有 Telegram / 飞书长轮询。

cron 是进程内轮询：五字段 UTC，外加 `@hourly/@daily/@weekly`，默认每 15 秒看一次。到期就创建 Task。上次运行还没到终态则跳过，不补跑错过的 tick。**写操作照样审批。** 你不在场，任务会停在 `waiting_approval`。

把这三件事放在同一篇，是为了打掉一种常见设计幻觉：「对话走策略，定时任务走快捷通道」。PocketPilot 没有快捷通道。

## 六、质量怎么锁：26 个不用 LLM 的用例

`backend/evals/` 用 `ScriptedProvider` 演戏：该调什么工具、返回什么、何时申请写盘，都写在脚本里。scorer 看关键词、产物路径、禁止路径、任务状态。最近一次提交的报告是 **26/26，约 1.58 秒，安全覆盖 5/5**。

隔离纪律：每个用例独立内存 SQLite，**不碰** `data/pocketpilot.sqlite3`。跑评测不会把你的真实工作区写脏。`--real` 模式存在，但同义词和 `status_also_ok` 会让分数发飘——安全回归以 fake 为准。

pytest 覆盖 health、agent、policy、documents、write、memory、skills、MCP、streaming、sessions、scheduler、channels、API、E2E。前端没有测试框架，这是缺口。`planning` 状态、`token_usage` 列（不是真 tokenizer 预算）、审批卡片没有改参 UI，也都还挂着。

Docker Compose 把 API 和 Web 一起拉起，volume `./data`。`evals` 包不进 wheel（`packages = ["app"]`）——评测是开发者工具，不是安装产物。这些都是诚实的成熟度，不必在介绍文里涂成「生产完备」。

## 七、故意不做（以及为什么）

| 不做 | 原因 |
|------|------|
| Dream / Heartbeat | 自动沉淀会让「磁盘上那份记忆」不再是真相源 |
| 子 Agent | 审批与审计会在树状委派里分叉，第一版不碰 |
| 默认 Shell / 网络 | 个人磁盘场景里，这两项是高风险且默认关；开了也必须走 execute/network 审批 |
| MCP `list_tools` / 远程 MCP | 审批爆炸；远程信任链超出本地优先 |
| 定时任务自动批准写盘 | 无人值守写盘 = 产品论题作废 |
| messages 表 | 会把 Task 级审计藏进聊天记录 |
| 多 worker EventBus | 没做分布式总线之前，复制进程只会造成「有的人看不见直播」 |

读到这里，三份源码样本可以收成一张对照：

| | nanobot | DeepSeek Harness | PocketPilot |
|--|---------|------------------|-------------|
| 内核组织 | 分层 Loop / Runner | Cordis 插件树 | 分层；Agent 不碰库 |
| 循环 | while + 治理 | Phase 状态机 + 事件 | 有界 for-loop |
| 工具把关 | 注册表 + SSRF/workspace | 流水线钩子 + 审批 | capability 后缀 + Guard + 停住等你 |
| 记忆 | Dream | 会话日志投影 | 有界 `MEMORY.md` |
| 适合讲什么 | 把原理讲透 | 生产级可替换底座 | 敢让 Agent 碰本地磁盘的最小闭环 |

## 八、小结

- 工具、记忆、技能、MCP 都从 `build_tool_registry` 进场，过 03 那道门。
- `analyze_table` 把能算的留下 Python；模型负责调用时机和报告。
- 入口无论 Chat、webhook 还是 cron，只创建普通 Task，不特赦写盘。
- 26 个 ScriptedProvider 用例把路径逃逸和拒批锁成回归，不依赖模型发挥。

本系列四篇到此结束。若要对照另外两种 Runtime，从 [Harness vs nanobot]({{< relref "harness-vs-nanobot.md" >}}) 进；若要从概念课重新串一遍，从 [学习路线图]({{< relref "agent-learning-roadmap.md" >}}) 进。
