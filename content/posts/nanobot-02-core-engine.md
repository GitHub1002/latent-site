---
title: "nanobot 源码解读 · 02 | 核心引擎：AgentLoop 与 AgentRunner 的分工"
date: 2026-08-03
draft: true
weight: 802
tags: ["nanobot", "源码解读", "Agent框架", "核心引擎"]
summary: "逐函数拆解 nanobot 的两个核心引擎：AgentLoop（面向通道的一回合编排，负责会话、权限、持久化、流式投递）与 AgentRunner（纯模型循环，负责调 LLM、执行工具、重试与上下文治理）。本文把『一条消息从通道进来、到答案流出去』的完整链路讲透，并点明权限红线与安全边界落在哪一层。"
ShowToc: true
---

在 01 里我们建立了整体印象：nanobot 是「小核心 + 插件式外围」的结构，而**整本书的「主心骨」只有两个文件**——`agent/loop.py`（约 2264 行）和 `agent/runner.py`（约 1670 行）。读懂这两个文件，后面所有枝节（上下文、记忆、工具、Provider、Channel）都是挂在它们身上的「被调用方」。

这一篇，我们就钻进这两个文件，把**「一条用户消息从通道进来、到答案流出去」的完整链路**读透。

先记住一句话，它是理解全部代码的钥匙：

> **`AgentLoop` 管「一回合（turn）该怎么做」——会话、权限、上下文装配、持久化、流式投递、子 Agent；`AgentRunner` 只管「一次模型循环该怎么做」——把消息喂给 LLM、执行工具、重试、压缩。**

`AgentRunner` 是一个**刻意与产品层解耦**的纯引擎：你给它一组 `initial_messages` 和 `ToolRegistry`，它还你一段 `final_content` 和完整 `messages`。它不知道「用户在飞书还是 CLI」「这条消息属于哪个会话」「权限红线是什么」。这些「产品层」的事情，全在 `AgentLoop` 里。

下面我们分别拆开，再合起来看它们怎么咬合。

---

## 一、全景：一次请求的生命周期

把 01 的架构图细化到「调用栈」层面，一条消息的完整路径是这样的：

```
通道(CLI/飞书/Slack…)
   │  InboundMessage
   ▼
MessageBus.consume_inbound()            ← AgentLoop.run() 在这里阻塞等待
   │
   ▼
AgentLoop._dispatch(msg)               ← 按 session_key 加锁，保证同会话串行
   │  asyncio.create_task
   ▼
AgentLoop._process_message(msg)        ← 一个 staging 流水线
   │   restore → compact → command → build → run → save → respond
   ▼
AgentLoop._run_turn(ctx)               ← RUN 阶段
   │
   ▼
AgentLoop._run_agent_loop(...)         ← 注入 ContextVar(权限/工作区/文件状态)
   │
   ▼
AgentRunner.run(AgentRunSpec)          ← 跨过「产品层 / 引擎层」边界
   │
   ▼
AgentRunner._run_core(...)             ← 模型循环：for iteration in range(N)
   │     ├─ ContextGovernance.prepare_for_model()  (上下文治理/压缩)
   │     ├─ _request_model()                       (调 LLM，可流式)
   │     ├─ should_execute_tools? → _execute_tools()
   │     └─ 注入检查 / 重试 / length 恢复 / 错误兜底
   │
   ▼
AgentRunResult(final_content, messages, …)
   │
   ▼
流式投递 → TurnDelivery.on_stream(...) → 通道回显给用户
```

可以把它想象成**两层同心圆**：外圈 `AgentLoop` 是「公司前台 + 项目经理」，负责接待客户、分流、记录、把成果送回；内圈 `AgentRunner` 是「只干活的工程师」，你给他任务清单和工具箱，他闷头循环直到产出结果。

---

## 二、外圈引擎：AgentLoop（面向通道的一回合）

### 2.1 主循环 `run()` — `loop.py:1133`

`AgentLoop.run()` 是整个进程的「心跳」。它做的事极其克制：

```python
async def run(self) -> None:
    self._running = True
    try:
        await self._connect_mcp()          # 启动时连上配置的 MCP Server
        while self._running:
            try:
                msg = await asyncio.wait_for(self.bus.consume_inbound(), timeout=1.0)
            except asyncio.TimeoutError:
                self._check_expired_sessions_if_due()   # 顺手清理过期会话
                continue
            # ... 处理 runtime control、优先级命令、自动化延期 ...
            task = asyncio.create_task(self._dispatch(msg))   # 每条消息起一个 task
```

几个关键点：

- 它**从 `MessageBus` 上消费 `InboundMessage`**（`bus.consume_inbound()`）。所有通道（CLI、飞书、Slack…）的消息都会被统一成 `InboundMessage` 推到总线上，所以 `AgentLoop` 自己完全不关心「消息从哪来」。
- 每条消息 `asyncio.create_task(self._dispatch(msg))`——**并发在 task 级别**，而不是在 `run()` 里。这样某个会话卡住（比如模型很慢）不会阻塞其它会话的消息进入。
- `timeout=1.0` 的轮询让它能在空闲时做会话过期清理，也能干净响应 `stop()`。

### 2.2 `_dispatch` — 并发模型：`loop.py:1226`

这是 nanobot **「同会话串行、跨会话并发」**的核心实现：

```python
async def _dispatch(self, msg: InboundMessage) -> None:
    session_key = self._effective_session_key(msg)
    lock = self._get_session_lock(session_key)   # 每个 session_key 一把 asyncio.Lock
    pending: asyncio.Queue[InboundMessage] | None = None
    try:
        async with lock:                          # 同一会话必须排队
            pending = asyncio.Queue(maxsize=20)
            self._pending_queues[session_key] = pending
            delivery = self.turn_delivery_factory.create(msg, session_key, enable_stream=True)
            response = await self._process_message(msg, on_stream=..., pending_queue=pending, ...)
            await delivery.complete(response, ...)
    finally:
        # 把 pending_queue 里没消费完的消息重新 publish 回总线，绝不静默丢失
        ...
```

两个设计点值得记住：

1. **`self._get_session_lock(session_key)`**：session_key 到 `asyncio.Lock` 的弱引用映射（`loop.py:2258`）。同一会话的下一条消息会等这把锁释放——这保证了会话历史的「读-改-写」不会被并发打乱。
2. **`pending_queue` 与「中途注入」**：如果一个会话正在跑，用户又发了一条消息，`run()` 不会新起一个竞争 task，而是把消息 `put_nowait` 进这个会话的 `pending_queue`（见 `loop.py:1188`）。`AgentRunner` 在模型循环里会周期性地「抽干」这个队列，把用户的新消息当作后续输入注入当前回合。这就是「用户中途插话、Agent 立即接上」的实现基础（详见 2.5 与 3.4）。
3. **`finally` 里的兜底**：队列里没被消费的剩余消息会被 `publish_inbound` 重新放回总线，作为全新的 inbound 再处理——**绝不静默丢弃用户消息**。

### 2.3 staging 流水线 `_process_message` — `loop.py:1372`

`_process_message` 是单次消息处理的「总导演」。它先构造一个 `TurnContext`（一个贯穿整回合的**大号数据容器**，见 `loop.py:118`），然后按固定顺序跑一串 stage：

```python
await self._run_turn_stage(ctx, "restore", self._restore_turn)     # 恢复会话/检查点
await self._run_turn_stage(ctx, "compact", self._compact_session)  # 必要时压缩
if await self._run_turn_stage(ctx, "command", self._dispatch_command):
    return ctx.outbound                                               # 命中命令则直接返回
await self._run_turn_stage(ctx, "build",   self._build_turn)        # 装配上下文
await self._run_turn_stage(ctx, "run",     self._run_turn)          # 跑模型循环
await self._run_turn_stage(ctx, "save",    self._persist_turn)      # 持久化
await self._run_turn_stage(ctx, "respond", self._prepare_outbound)  # 组装出站
return ctx.outbound
```

`TurnContext`（`loop.py:118`）是这个流水线的「传球对象」：它装着 `msg`、`session`、`runtime`、`history`、`initial_messages`、`final_content`、流式回调 `on_stream/on_stream_end`、权限相关的 `request_context`，以及中途注入队列 `pending_queue`。每个 stage 读它、改它、传给下一个 stage。

> `restore` 阶段（`loop.py:1553`）干的是「把会话从磁盘恢复出来 + 还原被 `/stop` 中断时存档的 checkpoint」。注意它还会处理附件（非图片附件引用）。`compact` 阶段视情况做会话压缩（我们放到 04 专门讲）。`command` 阶段先看这是不是一条内置命令（如 `/model`、`/stop`），命中就短路返回。

### 2.4 BUILD 阶段：把「上下文」装配好 — `loop.py:1647`

`build` 阶段是最靠近「产品层智慧」的地方。`_build_turn` 做的事：

1. **确定 runtime**：`ctx.runtime = self.runtime_for_session(session)`，即从会话元数据里解析出「这一回合该用哪个模型/Provider/上下文窗口」。
2. **取历史**：`ctx.history = session.get_history(max_messages=..., max_tokens=...)`——只回放「最近 N 条 / 最近 T token」的历史，超出部分交记忆层处理（03/04 讲）。
3. **解析运行时上下文块**：`ctx.runtime_context_blocks = await self._resolve_runtime_context_for_turn(ctx)`——这是把「当前时间、工作区文件清单、可用 skill」等动态信息拼成可注入消息的块。
4. **续接 Provider 会话状态**：`ctx.provider_state`——nanobot 支持把上一回合的 Provider 内部状态（如 Anthropic 的 `conversation_id`）存进会话，下一回合 `with_pending_messages(...)` 续上，省去重复发送全量历史（这正是它「省 token」的关键之一）。
5. **装配初始消息**：`ctx.initial_messages = self._build_initial_messages(ctx)`（`loop.py:711`），最终调用 `self.context.build_messages(...)` 把历史 + 当前消息 + 记忆 + 运行时上下文拼成一个干净的 `list[dict]`。

读到这里你会发现：**`AgentRunner` 拿到的 `initial_messages` 已经是一个「成品」**——历史裁剪、记忆注入、权限作用域、运行时上下文，全在 `AgentLoop` 这层做好了。

### 2.5 RUN 阶段：跨过边界 — `_run_turn` → `_run_agent_loop`

`_run_turn`（`loop.py:1772`）很短，它只是把 `ctx.initial_messages` 和一堆回调丢给 `_run_agent_loop`。而 **`_run_agent_loop`（`loop.py:840`）是「产品层」与「引擎层」之间的真正桥梁**，它有三类关键动作：

**(a) 注入 ContextVar（权限红线的落点）**

```python
file_state_token  = bind_file_states(self._file_state_store.for_session(active_session_key))
request_token     = bind_request_context(request_ctx)   # 含 channel / sender_id / workspace
workspace_token   = bind_workspace_scope(effective_scope)
```

这里用 Python `contextvars` 把「当前请求上下文 / 工作区作用域 / 文件读写状态」**绑定到当前协程**。下游所有工具（`shell`/`filesystem`/`web`…）都通过 `contextvars` 读取这些边界——**工具函数本身不接收 user_id 之类参数，边界由 `AgentLoop` 在服务端注入**。这跟我们系列二讲过的「零参数工具 + 服务端注入」是同一个思路，是权限红线成立的根。

**(b) 构造 `AgentRunSpec`，跨过边界调用 `self.runner.run(...)`**

```python
result = await self.runner.run(AgentRunSpec(
    initial_messages=initial_messages,
    tools=effective_tools,                       # 当前会话可见的工具集
    runtime=runtime,
    max_iterations=self.max_iterations,
    concurrent_tools=True,                        # 工具可并发执行
    injection_callback=_drain_pending,           # ← 中途注入的「抽水泵」
    checkpoint_callback=_checkpoint,             # ← 把运行态存档到会话
    llm_timeout_s=...,                            # 防模型挂死
    goal_active_predicate=...,                    # 持续目标（sustained goal）
    ...
))
```

注意 `injection_callback=_drain_pending`：它就是从 2.2 提到的 `pending_queue` 里抽用户中途消息的「泵」。`checkpoint_callback` 则在每轮工具执行后把 `provider_state` 存进会话，保证 `/stop` 后能从断点续上。

**(c) 收回 ContextVar 并收尾**

`finally` 里 `reset_workspace_scope / reset_request_context / reset_file_states` 解除绑定，避免泄漏到其它协程。最后把 `result.final_content` 等写回 `ctx`。

---

## 三、内圈引擎：AgentRunner（纯模型循环）

`AgentRunner`（`runner.py:137`）的 `__init__` 只有一个成员：`self.context_governor = ContextGovernor()`——即上下文治理器（03 篇主角）。它**没有任何产品层状态**，因此可以被 `AgentLoop`、子 Agent（`SubagentManager`）甚至外部 SDK 复用。

### 3.1 外层包装 `run()` — `runner.py:371`

`run()` 是给 hook（插件）用的薄壳：包了 `before_run / after_run / on_error / on_finally`，真正逻辑在 `_run_core`。

### 3.2 心脏 `_run_core` — `runner.py:419`

这是整篇最该精读的函数。它就是一个「带工具调用的 ReAct 循环」：

```python
for iteration in range(spec.max_iterations):
    # 1) 上下文治理：在喂给模型前，按需裁剪/压缩历史
    messages_for_model = self.context_governor.prepare_for_model(
        governance_config, messages, compacted_tool_call_ids,
    )
    # 2) 调模型（可流式）
    response = await self._request_model(spec, messages_for_model, hook, context, ...)
    # 3) 抽取 reasoning / 清理内容 / 累计 token
    ...
    if response.should_execute_tools:
        # 4a) 把 assistant 消息（含 tool_calls）追加进 messages
        messages.append(assistant_message)
        # 4b) 执行工具
        results, new_events, fatal_error = await self._execute_tools(spec, response.tool_calls, ...)
        # 4c) 把每个 tool 结果作为 role="tool" 消息追加
        for tool_call, result in zip(response.tool_calls, results):
            messages.append({"role": "tool", "tool_call_id": ..., "content": ...})
        # 4d) 检查中途注入，然后 continue 进入下一轮
        await self._try_drain_injections(...)
        continue
    # 5) 不需要工具：处理空响应重试 / length 截断恢复 / error 兜底
    ...
    # 6) 没有注入则 break，回合结束
```

逐点拆解：

**① 上下文治理前置（`prepare_for_model`）**：每轮循环**不是**直接把 `messages` 喂给模型，而是先过一遍 `ContextGovernor`。它会按 `context_window_tokens`、工具结果长度上限（`max_tool_result_chars`）等做「按需压缩」——这是 nanobot 能在小上下文里跑长任务的秘密。具体算法留到 03 篇。

**② 调模型 `_request_model`（`runner.py:895`）**：
- 根据 `hook.wants_streaming()` 决定走 `chat_stream_with_retry`（流式）还是 `chat_with_retry`（整块）。流式模式里，`_stream` 把内容 delta 实时推给 hook（最终回到 `TurnDelivery.on_stream` 流给用户），`_thinking` 增量抽取推理过程，`_stream_recover` 处理「流式中断后恢复」。
- 带 **wall-clock 超时**（`NANOBOT_LLM_TIMEOUT_S`，默认 300s），防模型挂死把会话锁饿死。
- **畸形工具调用自愈**：若模型返回了参数残缺的 tool_call，`_drop_malformed_tool_calls` 会丢掉它们；若**全丢**，自动重试一次；重试后仍全丢，则降级为「不带工具」的请求，绝不让一次坏输出卡死循环。

**③ / ④ 工具执行链路**：`should_execute_tools` 为真时——
- 先把 assistant 消息（含 `tool_calls`）写进 `messages`（模型要求的「成对」结构）。
- `_execute_tools`（`runner.py:1359`）先 `_partition_tool_batches` 把工具调用分批，**同批内 `asyncio.gather` 并发执行**（`concurrent_tools=True`），不同批串行。
- 每个工具经 `_run_tool`（`runner.py:1410`）：
  - 先查「重复外部查询」节流（`repeated_external_lookup_error`），防止模型对同一 URL 死循环。
  - 通过 `spec.tools.prepare_call` 做**参数预处理与权限预校验**（如工作区路径归一化）。
  - 真正执行 `tool.execute(**params)`。
  - 出错时 `_classify_violation` 区分**硬安全边界（SSRF）**与**软边界（workspace 越界）**：SSRF 是不可绕过的红线（见 `runner.py:1527` 的 `_SSRF_MARKERS`），workspace 越界则作为「可恢复的软错误」返回给模型让它换个合法路径重试。
  - 结果经 `ContextGovernor.normalize_tool_result` 按长度上限截断，再作为 `role="tool"` 消息追加。
- 若 `fatal_error` 且 `fail_on_tool_error`，则终止回合。

**⑤ 收尾分支（无工具调用时）**：
- **空响应**：`is_blank_text(clean)` 且 finish_reason 正常 → 最多 `_MAX_EMPTY_RETRIES=2` 次重试；仍空则 `_request_finalization_retry` 做一次「总结性重试」。
- **length 截断恢复**：`finish_reason=="length"` 且未超 `_MAX_LENGTH_RECOVERIES=3` 次 → 把已产出片段拼起来，追加 `build_length_recovery_message` 再让模型续写（流式模式下 `resuming=True` 让同一张卡片继续生长，而不是开新卡片）。
- **error**：区分「欠费/余额不足」（`_ARREARAGE_ERROR_MESSAGE`）与普通错误，写占位消息并终止。

**⑥ 中途注入检查 `_try_drain_injections`（`runner.py:240`）**：在每个「可能结束」的节点（工具执行后、最终回答后、错误后、达到 max_iterations 后）都调用它。如果有用户中途插话或「持续目标」需要继续，就 `append` 进 `messages` 并 `continue`，让循环活下去。上限 `_MAX_INJECTIONS_PER_TURN=3`、`_MAX_INJECTION_CYCLES=5`，防失控。

循环正常 `break` 或跑满 `max_iterations`（`else` 分支）后，构造 `AgentRunResult`（`runner.py:121`）返回：`final_content`、`messages`、`tools_used`、`usage`、`stop_reason`、`had_injections`、`provider_state`。

### 3.3 两个引擎如何咬合：AgentRunSpec 桥

`AgentRunSpec`（`runner.py:91`）就是**产品层 → 引擎层的「参数包」**：

| 字段 | 含义 | 由谁填 |
|---|---|---|
| `initial_messages` | 已装配好的上下文 | `AgentLoop._build_turn` |
| `tools` | 当前会话可见工具集 | `AgentLoop`（含 MCP/子 Agent 注入） |
| `runtime` | 模型/Provider/上下文窗口 | `AgentLoop.runtime_for_session` |
| `injection_callback` | 中途注入泵 | `AgentLoop._run_agent_loop._drain_pending` |
| `checkpoint_callback` | 运行态存档 | `AgentLoop._run_agent_loop._checkpoint` |
| `llm_timeout_s` | 模型超时 | `AgentLoop` 依会话策略算 |
| `concurrent_tools` | 工具并发 | 固定 `True` |

`AgentRunner` 对其余一切（通道、权限、持久化）一无所知——它只认这个包。这种**强边界**让引擎可以被子 Agent、SDK、自动化任务复用，而无需复制任何产品层逻辑。

---

## 四、流式输出链路（把前面串起来）

用户看到「打字机效果」，链路是这样打通的：

1. `AgentLoop._dispatch` 创建 `TurnDelivery`（`enable_stream=True`），拿到 `delivery.on_stream / on_stream_end`。
2. 这俩回调一路透传：`_process_message` → `_run_turn` → `_run_agent_loop` → `AgentRunSpec.progress_callback / stream_progress_deltas`。
3. `AgentRunner._request_model` 在流式模式下，把每个内容 delta 调 `hook.on_stream(context, delta)` → 最终回到 `TurnDelivery.on_stream` → 通道把增量推给用户。
4. 关键细节：流式模式下 `on_stream_end(resuming=True/false)` 控制「这张卡片是继续生长还是封口」。模型要调工具时 `resuming=True`（卡片先别封），工具结果回来后下一轮内容继续往同一张卡片灌；真正答完 `resuming=False` 封口。这正是飞书这类「一张卡片持续更新」体验的来源。
5. 即便走「非流式恢复」（如 length 续写），代码也会检测「前面已经有可见片段」而把续段 `on_stream` 进同一流，避免重复前缀。

---

## 五、权限红线与安全边界落在哪一层？

读完两个引擎，可以明确回答「红线在哪」：

- **权限边界（谁能访问什么）在 `AgentLoop` 层**：通过 `contextvars` 注入 `request_context`（含 `sender_id`/`workspace`）、`workspace_scope`、`file_state`，工具本身不接收用户身份参数。跨协程靠 `contextvars` 天然隔离。
- **安全边界（SSRF / workspace 越界）在 `AgentRunner` 的工具执行层**：`_run_tool` 调 `_classify_violation` 拦截。SSRF 是「不可绕过、要模型停下来」的硬线；workspace 越界是「可恢复、让模型换合法路径」的软线。
- **防失控在两层都有**：`max_iterations` + `_MAX_INJECTION_CYCLES`（引擎层）；`pending_queue` 容量 20 + session 锁（产品层）。

一句话：**`AgentLoop` 决定「你能不能、用什么身份做」，`AgentRunner` 决定「做的时候撞到安全墙怎么处理」**。

---

## 六、小结

- `AgentLoop`（`loop.py`）是**产品层的一回合编排器**：消费总线 → 按会话加锁分发 → staging 流水线（restore/compact/command/build/run/save/respond）→ 注入 ContextVar → 调 `AgentRunner` → 流式投递。
- `AgentRunner`（`runner.py`）是**与产品层解耦的纯模型循环**：`for iteration` 里做上下文治理 → 调 LLM（含流式/超时/畸形重试）→ 执行工具（含并发/安全分类）→ 注入检查/空响应/length 恢复/错误兜底 → 返回 `AgentRunResult`。
- 两者通过 `AgentRunSpec` 这一个「参数包」解耦，`AgentRunner` 因此可被子 Agent、SDK、自动化任务复用。
- 权限红线在 `AgentLoop`（ContextVar 注入），安全边界在 `AgentRunner`（SSRF/workspace 分类）。

---

下一篇 **03 · 上下文工程** 会专门钻进 `_build_initial_messages` 调用的 `ContextBuilder`（`agent/context.py`）与 `ContextGovernor`（`agent/context_governance.py`），讲清「历史怎么裁剪、记忆怎么注入、工具结果怎么截断、超限时怎么自动压缩」——也就是 nanobot 能在小上下文里跑长任务的核心机制。

> 本文所有行号基于你本地 `D:\github_projects\nanobot` 源码：`agent/loop.py`、`agent/runner.py`（截至 2026-08-03 的版本）。建议对照阅读。
