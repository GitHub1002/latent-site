---
title: "nanobot 源码解读 · 08 | 多 Agent 委派：一个 Agent 如何把活派给子 Agent"
date: 2026-08-09
draft: false
weight: 808
tags: ["nanobot", "源码解读", "Agent框架", "多Agent", "子Agent"]
summary: "逐函数拆解 nanobot 的多 Agent 委派：SubagentManager 如何用 spawn(后台) 与 run_inline(同步) 两种方法派活、子 Agent 如何复用 02 篇的 AgentRunner 引擎、独立的工具注册表与权限红线如何隔离、结果又如何通过 MessageBus 注入回主 Agent 的 pending 队列。配真实代码、委派流向图与可运行例子。"
ShowToc: true
---

## 一、为什么多 Agent 不是另起炉灶

前面 02 篇我们拆解了 `AgentRunner`——它是真正跑「请求模型 → 执行工具」循环的引擎，通过 `AgentRunSpec` 与外界解耦。这个解耦现在派上大用场了：**nanobot 的「多 Agent 委派」，本质上就是「再喂一份 messages + tools + runtime 给同一个 `AgentRunner`」**。

`agent/subagent.py`（577 行）里的 `SubagentManager` 并不重新实现一遍思考循环，而是：
1. 给子任务准备一份**独立的**系统提示 + 工具集；
2. 用 `self.runner.run(AgentRunSpec(...))` 把活交给那个同一个 `AgentRunner`；
3. 子 Agent 跑完后，结果通过 `MessageBus` **注入回主 Agent** 的会话。

所以你之前读的 02/05/07 篇——引擎、工具、Provider——对子 Agent 全部复用，没有重复代码。这就是「小核心 + 复用」架构的好处。

---

## 二、两种派活方式：spawn vs run_inline

`SubagentManager` 对外暴露两个入口，区别在于**要不要等结果**：

```python
async def spawn(self, task, label=None, origin_channel="cli", session_key=None,
                workspace_scope=None, *, runtime=None) -> str:
    """后台派活，立即返回，结果稍后通过消息总线通知。"""
    task_id = str(uuid.uuid4())[:8]
    ...
    bg_task = asyncio.create_task(
        self._run_subagent(task_id, task, display_label, origin, status, runtime, ...)
    )
    self._running_tasks[task_id] = bg_task
    ...
    return f"Subagent [{display_label}] started (id: {task_id}). I'll notify you when it completes."

async def run_inline(self, task, ..., *, runtime=None) -> str:
    """同步派活，await 拿到结果再返回（直接作为工具结果回给主 Agent）。"""
    inline_task = asyncio.create_task(self._run_subagent(...))
    try:
        result = await inline_task
        if status.phase == "error" or status.stop_reason in {"error", "tool_error"}:
            return ToolResult.error(result)   # 失败 → 包装成工具错误
        return result
    finally:
        ...清理...
```

- `spawn`：后台异步，主 Agent 立刻拿到一句「已开始（id: xxx），完成后通知你」，**不阻塞**当前轮。适合「你先去查资料，我有空再看」。
- `run_inline`：`await` 等到子 Agent 跑完，结果直接作为当前工具调用的返回值。注意失败时它用 `ToolResult.error(result)` 包一下——因为对主 Agent 来说，子 Agent 就是一个「工具」。

> 这两种入口正好对应两种多 Agent 模式：**fire-and-forget（后台派活）** 与 **tool-style 调用（同步拿结果）**。nanobot 把子 Agent 当作一种「特殊工具」嵌入了 05 篇的调用链里。

---

## 三、_run_subagent：子 Agent 是怎么跑起来的

无论哪个入口，最终都进 `_run_subagent`。它的骨架很清晰：

```python
async def _run_subagent(self, task_id, task, label, origin, status, runtime,
                        origin_message_id, workspace_scope, *, announce=True) -> str:
    try:
        root = workspace_scope.project_path if workspace_scope is not None else self.workspace
        cfg = None
        if workspace_scope is not None:
            cfg = self._subagent_tools_config()
            cfg.restrict_to_workspace = workspace_scope.restrict_to_workspace
        tools = self._build_tools(tools_config=cfg)          # ① 独立工具注册表
        system_prompt = self._build_subagent_prompt(workspace=root)  # ② 子 Agent 专属系统提示
        messages = [
            {"role": "system", "content": system_prompt},
            {"role": "user",   "content": task},             # ③ 任务就是第一条用户消息
        ]
        # ④ 权限红线照旧：用 contextvars 注入（呼应 02 篇）
        request_token = bind_request_context(RequestContext(
            channel=origin["channel"], chat_id=origin["chat_id"],
            message_id=origin_message_id, session_key=..., runtime=runtime))
        token = bind_workspace_scope(workspace_scope) if workspace_scope is not None else None
        try:
            result = await self.runner.run(AgentRunSpec(    # ⑤ 复用 02 篇引擎！
                initial_messages=messages,
                tools=tools,
                runtime=runtime,
                max_iterations=self.max_iterations,
                max_tool_result_chars=self.max_tool_result_chars,
                hook=_SubagentHook(task_id, status),
                checkpoint_callback=_on_checkpoint,
                session_key=sess_key,
                workspace=root,
                llm_timeout_s=llm_timeout,
            ))
        finally:
            if token is not None: reset_workspace_scope(token)
            reset_request_context(request_token)
        # ⑥ 根据 stop_reason 决定最终回传文本 + 是否 announce
        ...
        if announce:
            await self._announce_result(task_id, label, task, final_result, origin, final_status, ...)
        return final_result
    except Exception as e:
        ...
```

把这段和 02 篇对照，你会会心一笑：**子 Agent 和主 Agent 跑的是同一套引擎，区别只在「喂进去的 messages/tools/runtime/workspace 不同」**。这正是 `AgentRunSpec` 解耦的价值——`SubagentManager` 不需要碰引擎内部，只管「准备输入、收结果」。

---

## 四、隔离：子 Agent 有自己的一套工具与红线

多 Agent 最怕「子 Agent 拿到主 Agent 的全部权限乱搞」。nanobot 用两个手段隔离：

### 4.1 独立的工具注册表（scope="subagent"）

```python
def _build_tools(self, workspace=None, tools_config=None) -> ToolRegistry:
    root = self.workspace if workspace is None else workspace
    registry = ToolRegistry()
    cfg = tools_config if tools_config is not None else self._subagent_tools_config()
    ctx = ToolContext(
        config=cfg,
        workspace=str(root.resolve()),
        exec_session_manager=self._exec_session_manager,
        file_state_store=FileStates(),                       # 独立的文件状态
        workspace_sandbox=workspace_sandbox_status(
            restrict_to_workspace=cfg.restrict_to_workspace, workspace=root),
    )
    ToolLoader().load(ctx, registry, scope="subagent")      # 仅加载 subagent 作用域的工具
    return registry

def _subagent_tools_config(self) -> ToolsConfig:
    return ToolsConfig(
        exec=self.tools_config.exec,
        web=self.tools_config.web,
        file=self.tools_config.file,
        restrict_to_workspace=self.restrict_to_workspace,    # 默认限制在工作区
    )
```

注意三点：
- `ToolLoader().load(..., scope="subagent")`——子 Agent 只加载标记为 subagent 作用域的工具，**拿不到主 Agent 的全套工具**（呼应 05 篇的加载器作用域机制）。
- `FileStates()` 是**新实例**——子 Agent 的文件读写状态与主 Agent 互不污染。
- `restrict_to_workspace=True` 默认开——子 Agent 的活动被锁在工作区里（呼应 05 篇 workspace 安全边界）。

### 4.2 权限红线沿用 contextvars

子 Agent 同样通过 `bind_request_context(RequestContext(...))` 和 `bind_workspace_scope(workspace_scope)` 注入上下文（02 篇讲过的「服务端注入、模型改不了」红线）。只不过子 Agent 的 `RequestContext.runtime` 用的是父辈传下来的 `LLMRuntime`——所以子 Agent 用的模型/Provider 和主 Agent 同源，但操作边界被 `workspace_scope` 收窄。

---

## 五、结果怎么回主 Agent：MessageBus 注入

子 Agent 跑完，结果怎么告诉主 Agent？这里有个很巧妙的设计。看 `_announce_result`：

```python
async def _announce_result(self, task_id, label, task, result, origin, status, origin_message_id):
    status_text = "completed successfully" if status == "ok" else "failed"
    announce_content = render_template(
        "agent/subagent_announce.md",
        label=label, status_text=status_text, task=task, result=result,
    )
    # 以 system 消息注入，触发主 Agent 的「中途注入」机制
    override = origin.get("session_key") or f"{origin['channel']}:{origin['chat_id']}"
    msg = InboundMessage(
        channel="system",
        sender_id="subagent",
        chat_id=f"{origin['channel']}:{origin['chat_id']}",
        content=announce_content,
        session_key_override=override,                    # 对齐主 Agent 的有效 session
        metadata={"injected_event": "subagent_result", "subagent_task_id": task_id},
    )
    await self.bus.publish_inbound(msg)                  # 丢回主 Agent 的消息总线
```

关键点：
- 子 Agent 的结果不是「函数返回值」直接塞给主 Agent，而是**作为一条 `channel="system"` 的入站消息，发回主 Agent 的 `MessageBus`**。
- `session_key_override` 故意对齐主 Agent 的 session key（考虑 unified sessions），这样这条消息会被路由到主 Agent 的 **pending 队列**，触发 02 篇讲过的「用户中途插话 / 中途注入」同一个机制——主 Agent 在下一轮循环里读到这条 system 消息，才知道「哦，子任务做完了，结果是这样」。
- `metadata["injected_event"]="subagent_result"` 是个标记，主 Agent 的循环能据此识别这是子 Agent 回汇，而不是真人新消息。

> 这个设计的优雅之处：子 Agent 回汇**复用了主 Agent 既有的消息总线 + pending 队列 + 中途注入**，没有为「多 Agent」单独发明一套通信协议。主 Agent 看待子 Agent 的结果，和看待「用户中途插一句」是同一个抽象。

---

## 六、并发管理与生命周期

`SubagentManager` 还管一堆运维细节：
- `_running_tasks: dict[str, asyncio.Task]`——所有在跑的子任务，用 `create_task` 并发；
- `max_concurrent_subagents`——并发上限（默认取自 `AgentDefaults`）；
- `_session_tasks: dict[session_key, set[task_id]]`——按会话分组，支持 `cancel_by_session` 一键取消某会话的全部子 Agent；
- `SubagentStatus`——实时状态机（`initializing → awaiting_tools → tools_completed → final_response → done/error`），配合 `_SubagentHook` 在每轮迭代后回写 `iteration/tool_events/usage`，让前端能实时展示「子 Agent 正在调哪个工具」；
- `close()`——退出时取消所有在跑的子任务并关闭共享的 `ExecSessionManager`。

---

## 七、一次多 Agent 委派的完整流向

![nanobot 08 多 Agent 委派流向](/images/nanobot-08-multiagent.svg)

串起来看：
1. 主 Agent 在 02 篇的 `_run_core` 循环里决定「调用子 Agent 工具」→ 进入 05 篇的 `_run_tool`。
2. 该工具本质是 `SubagentManager.spawn` 或 `run_inline`：前者 `create_task` 后台跑、立即返回「已开始」；后者 `await` 拿到结果。
3. 子任务进 `_run_subagent`：建**独立**工具注册表（`scope="subagent"`）→ 拼子 Agent 专属系统提示 → 注入 `RequestContext`/`WorkspaceScope` 红线 → 调**同一个 `AgentRunner.run`**（02 篇引擎）。
4. 子 Agent 内部照常「调模型 → 执行工具（受 subagent 作用域约束）→ 压缩 → 回模型」循环，直到 `stop_reason` 给出终态。
5. 若 `announce=True`：结果经 `render_template` 渲染成回汇消息，以 `system` 入站消息 `publish_inbound` 回主 Agent 总线，对齐 session key 后落入 pending 队列。
6. 主 Agent 下一轮循环读到这条 system 消息，把子任务结果纳入自己的上下文，继续推进。

---

## 八、小结

本篇要点：
- 多 Agent 委派**复用 02 篇的 `AgentRunner` 引擎**，靠 `AgentRunSpec` 喂不同输入实现，零重复代码。
- `spawn`（后台、fire-and-forget）与 `run_inline`（同步、当工具）两种派活模式。
- 子 Agent 有**独立工具注册表**（`scope="subagent"`）+ 独立 `FileStates` + 默认 `restrict_to_workspace`，权限被收窄。
- 红线沿用 contextvars 注入（`RequestContext`/`WorkspaceScope`）。
- 结果经 **MessageBus 以 system 消息注入主 Agent 的 pending 队列**，复用「中途注入」机制，无需新协议。
- `SubagentManager` 统一管理并发上限、会话级取消、实时状态。

下一篇（也是本系列最后一篇）**09 · 接入与产品化**：看 `channels/*` 如何让同一个 Agent 接进 20+ 聊天平台、`sdk/` 的 Python SDK、`api/server.py` 的 OpenAI 兼容 API、以及 `cron/` 的定时自动化——把「一个核心」变成「一个可以对外服务的产品」。
