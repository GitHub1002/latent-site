---
title: "PocketPilot 源码解读 · 03 | 护栏才是护城河：WorkspaceGuard、capability 与审批恢复"
date: 2026-09-13
slug: pocketpilot-03-policy
draft: false
weight: 912
tags: ["PocketPilot", "源码解读", "策略引擎", "WorkspaceGuard", "人工审批"]
summary: "PocketPilot 不按工具名写白名单：路径先过 WorkspaceGuard，再按 capability 后缀分成 ALLOW / REQUIRE_APPROVAL / DENY。写操作在磁盘变更前用异常打断；批准恢复时重建 Guard，下一次 write 还要再批。配策略决策图。"
ShowToc: true
---

02 把循环走通了。循环本身并不稀奇——nanobot 和 Harness 都有。PocketPilot 真正不愿意让步的是：**模型可以决定调不调工具，不能决定工具能碰到哪、以及写不写盘。**

> 一把钥匙：策略引擎不认工具名，认 capability 的最后一个冒号后缀；路径一律经 `WorkspaceGuard.resolve`。新增工具默认只改注册，不改这套判定。

![每次工具调用都走同一道门](/images/pocketpilot-03-policy.svg)

## 一、WorkspaceGuard：相对路径 + resolve 牢笼

`backend/app/policy/workspace.py` 很短，短是有意的——所有读写类工具都该经过它，而不是用模型给的字符串直接 `open()`。

```python
# backend/app/policy/workspace.py · WorkspaceGuard.resolve
def resolve(self, relative_path: str = ".") -> Path:
    candidate = Path(relative_path)
    if candidate.is_absolute():
        raise WorkspaceAccessError("Absolute paths are not allowed.")

    resolved_candidate = (self._root / candidate).resolve()
    try:
        resolved_candidate.relative_to(self._root)
    except ValueError as error:
        raise WorkspaceAccessError("Path escapes the authorized workspace.") from error
    return resolved_candidate
```

它拒绝三件事：

1. **绝对路径**。模型就算写出 `/etc/passwd` 也进不了 `open`。
2. **`..` 逃逸**。`(root / candidate).resolve()` 后再 `relative_to(root)`，穿越根目录会抛错。
3. **符号链接逃出根。** `resolve()` 会跟到真实目标；链接指到工作区外，同样过不了 `relative_to`。

初始化时根目录本身也要 `expanduser().resolve()` 且必须已是目录。工具侧的纪律是：永远 `guard.resolve(rel)`，不要拼接字符串当路径。MCP 的路径型参数、产物下载、操作员文件 API，走的是同一把 Guard——不是「Agent 有牢笼、下载接口没有」。

评测里有专门的路径逃逸用例（`evals/cases/` 下的 escape 类 JSON）。这不是文档愿望，是回归项。

## 二、capability 后缀，而不是工具名表

`backend/app/policy/engine.py` 的判定顺序就两步：有 `path` 先过 Guard；再看 `ToolSpec.capability`。

```python
# backend/app/policy/engine.py
_READ_ACTIONS = frozenset({"read", "list"})
_SIDE_EFFECT_ACTIONS = frozenset(
    {"write", "create", "update", "delete", "execute", "network"}
)

def _classify_capability(capability: str) -> tuple[PolicyDecision, str]:
    action = capability.rsplit(":", 1)[-1].lower()
    if action in _READ_ACTIONS:
        return PolicyDecision.ALLOW, action
    if action in _SIDE_EFFECT_ACTIONS:
        return PolicyDecision.REQUIRE_APPROVAL, action
    return PolicyDecision.DENY, action
```

映射到真实工具：

| 工具 | capability | 判定 |
|------|------------|------|
| `list_files` / `read_file` / `extract_document` / `analyze_table` | `file:read` | ALLOW |
| `expand_skill` | `skill:read` | ALLOW |
| `write_file` | `file:write` | REQUIRE_APPROVAL |
| `memory_write` | `memory:write` | REQUIRE_APPROVAL |
| 白名单 MCP 的执行类 | `mcp:…:execute` | REQUIRE_APPROVAL |
| 没声明 capability | `None` | **DENY**（缺省不是放行） |

未知后缀同样 DENY。这比「忘记写进黑名单」更安全：新 MCP 工具如果解析不出能力，根本注册不进表（04 会写 P0–P3）；即便漏注册进了循环，引擎也会拒。

写审批的文案还分了 create / overwrite——看目标相对路径此刻是否存在。这是给审批卡片看的，不是第二条策略。

Agent 侧接到 DENY 会抛 `PolicyViolationError`；接到 REQUIRE_APPROVAL 会抛 `ApprovalRequiredError`。ALLOW 才 `tools.execute`。02 里贴过那段循环，这里只补一句：**策略回调是注入的，Agent 自己不 import engine。** 方便单测时塞一个假 callback。

## 三、审批：打断时磁盘必须还是干净的

`ApprovalRequiredError` 带着四样东西：当前 `tool_call`（可被前端改参）、完整 `messages`、`step`、已累积的 `events`。TaskService `_create_approval_request` 把 checkpoint 塞进 `approval_required` 事件的 payload，任务进入 `waiting_approval`。

API 在 `backend/app/api/approvals.py`：`GET /api/approvals` 列待办，`POST /api/approvals/{id}/resolve` 批准或拒绝。一个容易漏掉的实现细节：**恢复跑在这发 HTTP 请求的协程里**，不再走 `worker.start_task()`。原因是 checkpoint 已经在手头，没必要再排队一次；代价是这只请求可能比较长——用户点批准之后，同一条连接要等到后续循环告一段落或再次打断。

恢复路径（`TaskService._resume_after_decision_inner`）的不变量：

1. **批准**：用**已批准**（允许被编辑过）的参数执行这一次工具，把结果追加进 messages，然后 `agent.run(messages=..., start_step=step+1)`。
2. **拒绝**：写入一条 tool-failed 消息，同样续跑——模型可能改用只读工具，或再请求另一次写。
3. **Guard 与 policy_callback 必须新建。** 曾经有过「续跑不再过策略」的 bug。修复后，下一次 `write_file` 仍然要审批。批准不是发了张全场通行证。
4. 任务级超时 `wait_for` 在恢复时仍然套着。

前端 `ApprovalCard` 目前支持批准 / 拒绝和高风险高亮。API 已经能收 `edited_arguments`，卡片还没把改参暴露出来——这是已知缺口，不是「产品决定不让改」。读代码时不要把 UI 能力当成 API 能力。

还有一条要写进安全账本，避免把「Agent 受策略约束」说成「所有写都受策略约束」：

> 控制台上传文件、`PUT` 记忆、`PUT` 技能，走的是操作员 API，**不经过审批卡片**。那是人在操作自己的工作区，不是模型在写盘。Agent 路径和操作员路径必须分开讲，混在一起就是在撒谎。

## 四、风险级别从 ToolSpec 来

`TaskService._risk_level_for_tool` 读的是工具声明，不是在服务层硬编码「write_file = high」。注册表才是风险真相源：`write_file` / `memory_write` 标 high，读类标 low。审批 inbox 的红标跟这个走。

状态机主路径在 02 写过：`running → waiting_approval → running`。不要在 `planning` 上找「先让模型计划再审批」——没有这层。计划若发生，只发生在模型自己的文字里，策略不认。

## 五、cron 和 webhook 也不特赦

04 会展开通道和调度。这里先把红线立住：`SchedulerService.fire_due` 和 `ChannelService.ingest` 都是 `create_task` + `start_task`。写文件照样 `REQUIRE_APPROVAL`。无人在场时，任务会停在 `waiting_approval`，而不是「反正是我自己设的定时，就写吧」。

这和 nanobot 的 cron、Harness 的 `ctx.approval` 都不同：nanobot 更偏向个人助理把活干完；Harness 把审批做成流水线上的一跳且不可用就 deny。PocketPilot 选择**停住等你**——本地桌面场景里，停住比静默拒绝更符合「这是我的磁盘」。

## 六、小结

- `WorkspaceGuard` 是硬边界：相对路径、`resolve()`、符号链接一并管。
- 策略认 capability 后缀：读放行，副作用审批，未知拒绝。缺省是否定。
- `ApprovalRequiredError` 保证打断时磁盘干净；恢复重建 Guard，批准不是终身授权。
- 操作员 API 与 Agent 写盘不是同一条路；cron/webhook 不自动批准。

下一篇 [**04 · 能力总线**]({{< relref "pocketpilot-04-capability-bus.md" >}}) 会看工具工厂、发票垂直、文件记忆、技能、白名单 MCP、会话与评测——证明这些能力没有绕开本篇这道门。
