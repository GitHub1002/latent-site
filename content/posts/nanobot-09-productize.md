---
title: "nanobot 源码解读 · 09 | 接入与产品化：一个核心如何接进 20+ 平台与 API"
date: 2026-08-09
draft: false
weight: 809
tags: ["nanobot", "源码解读", "Agent框架", "Channel", "SDK", "OpenAI API", "定时任务"]
summary: "本系列终篇：拆开 nanobot 如何把一个核心引擎变成可对外服务的产品。Channels 插件机制（BaseChannel + _handle_message 权限/配对/入总线、ChannelManager 的发现/构建/出站合并与重试）、Python SDK（SessionClient/MemoryClient/RuntimeClient + RunStream 事件流）、OpenAI 兼容 API（/v1/chat/completions 的 JSON/多部件/流式 SSE）、以及 cron 定时任务（at/every/cron + 经 submit_cron_turn 回汇总线）。核心结论：四个表面全部汇流到 MessageBus + AgentLoop。配真实代码与汇流图。"
ShowToc: true
---

## 一、产品化的本质：一个核心，多个表面

前面 8 篇我们读完了一个「小核心」：`AgentLoop`（产品层）+ `AgentRunner`（引擎）+ 上下文/记忆/工具/MCP/Provider/子 Agent。但一个 Agent 框架要真正「可用」，还得把它**接出去**——让人能在 Telegram 里跟它聊、让别的程序用 OpenAI 兼容 API 调它、让开发者用 SDK 嵌进自己的应用、还能定时自动跑任务。

nanobot 的做法非常统一，也是本篇要讲的核心结论：

> **所有「表面」（Channel / API / SDK / Cron）都只是「生产一条 `InboundMessage` 丢进 `MessageBus`」或「直接调 `AgentLoop.process_direct`」**。它们殊途同归，最终都汇流到 02 篇讲过的 `AgentLoop`，由同一个核心处理。

这就是为什么前面 8 篇值得精读——你理解了核心，外围的「产品化」只是几层薄薄的适配。下面逐一拆。

---

## 二、Channels：把 Agent 接进聊天平台

`channels/` 下有 20+ 个平台目录（telegram/discord/feishu/slack/matrix/wecom/whatsapp/…），外加 `base.py`、`manager.py`、`registry.py`、`plugin.py` 等基础设施。

### 2.1 BaseChannel：每个平台的实现契约

`base.py` 的 `BaseChannel` 是个 ABC，定义了每个平台必须/可选实现的方法：

```python
class BaseChannel(ABC):
    name: str = "base"
    display_name: str = "Base"
    send_progress: bool = True
    show_reasoning: bool = True

    @abstractmethod
    async def start(self) -> None: ...
    @abstractmethod
    async def stop(self) -> None: ...
    @abstractmethod
    async def send(self, msg: OutboundMessage) -> None: ...

    # 流式是可选的：子类实现 send_delta 才开启
    async def send_delta(self, chat_id, delta, *, stream_id=None, ...) -> None: ...
    async def send_reasoning_delta(self, chat_id, delta, ...) -> None: ...  # 思考过程
    async def send_file_edit_events(self, chat_id, edits, ...) -> None: ...

    @property
    def supports_streaming(self) -> bool:
        cfg = ...  # 配置里 streaming 开关
        return bool(streaming) and type(self).send_delta is not BaseChannel.send_delta
```

关键设计：
- **`start`/`stop`/`send` 是抽象方法**，每个平台自己对接 SDK 实现（比如 Telegram 用 `python-telegram-bot` 轮询/Webhook，飞书用自身 SDK）。
- **流式、思考过程、文件编辑事件都是「可选覆盖」**：默认是 no-op，只有平台原生支持（Slack 的 context block、Telegram 的可折叠引用、Discord 子文本）才去覆盖 `send_delta`/`send_reasoning_delta`。这样新增平台时，最小只需实现 `start/send`，高级能力按需叠加。
- `supports_streaming` 用「配置开关 **且** 子类真覆盖了 `send_delta`」双重判断——避免「开了流式但没实现」的尴尬。

### 2.2 _handle_message：入站消息如何进总线

每个平台的 `start` 在收到平台消息后，最终调 `_handle_message`——这是**所有平台的统一入站入口**，负责权限、配对、构造 `InboundMessage` 并 `publish_inbound`：

```python
async def _handle_message(self, sender_id, chat_id, content, media=None,
                          session_key=None, is_dm=False, authorization_id=None):
    permission_id = authorization_id if authorization_id is not None else sender_id
    if not self.is_allowed(permission_id):
        if is_dm:
            code = generate_code(self.name, str(sender_id))     # 生成配对码
            await self.send(OutboundMessage(content=format_pairing_reply(code), ...))
            return
        else:
            logger.warning("Access denied for sender {}", sender_id)
            return
    meta = metadata or {}
    if self.supports_streaming:
        meta = {**meta, "_wants_stream": True}                  # 声明想要流式
    msg = InboundMessage(
        channel=self.name, sender_id=str(sender_id), chat_id=str(chat_id),
        content=content, media=media or [], metadata=meta,
        session_key_override=session_key,
    )
    await self.bus.publish_inbound(msg)                        # 丢回主总线！
```

两个亮点：
- **权限分级 `is_allowed`**：star（运维者）> allowlist（`allow_from`）> pairing store（配对过）> 默认拒绝。陌生人私聊会收到一个**配对码**（`generate_code`/`format_pairing_reply`），配对后才放行——这是「个人 Agent」该有的隐私设计。
- **入站即 `publish_inbound`**：平台差异在 `_handle_message` 之前被各平台的 `start` 消化掉（把平台消息翻译成 `sender_id/content/media`），之后就完全汇入 02 篇的主总线，和 CLI、API、子 Agent 走的是**同一条路**。

呼应 08 篇：子 Agent 回汇也是 `publish_inbound(system)`，区别在于子 Agent 用 `session_key_override` 对齐主 Agent 会话；这里用 `session_key_override` 把平台消息归到正确的会话。殊途同归。

### 2.3 ChannelManager：发现、构建、出站

`manager.py` 的 `ChannelManager` 管三件大事：

**① 发现与构建（`_init_channels`）**：`discover_plugins()` 扫描所有平台插件 → 读各自配置段 → 解析出「运行时实例规格」→ 为每个实例 `_build_channel`（实例化、注入 `bus`、应用 `send_progress` 等布尔覆盖）→ 存进 `self.channels` 字典。注意支持「一对多」：一个平台插件可实例化多个 runtime（比如同时连两个 Telegram bot）。

**② 出站分发（`_dispatch_outbound`）**：从总线 `consume_outbound()` 取 `OutboundMessage`，按 `channel` 路由到对应实例的 `send`/`send_delta`。这里有几个很讲究的优化：
- **流式合并（coalesce）**：`_coalesce_stream_deltas` 把同一 `(channel, chat_id, stream_id)` 的连续 delta 攒成一批再发，**减少平台 API 调用**、降低流式延迟（LLM 生成快于平台发送时的缓冲）。
- **去重抑制（`_should_suppress_outbound`）**：用 `(channel, chat_id, origin_message_id)` 的指纹防重复投递。
- **重试（`_send_with_retry`）**：指数退避 `(1, 2, 4)` 秒，发送失败由 manager 统一兜底，各 channel 的 `send` 只需「失败就抛」。

**③ 不停机热启停（`apply_channel_feature_action`）**：WebUI 上点「启用/禁用」某频道，不用重启网关——manager 直接 `_build_channel` 新实例、`_start_channel_task` 起任务，或 `_stop_channel` 停掉。这是「产品」该有的体验。

---

## 三、Python SDK：把 Agent 嵌进你自己的代码

`sdk/` 提供了高层 Python 接口。核心是一组「客户端」类，它们都持有 `AgentLoop` 引用，把能力包装成好用的 API：

```python
class SessionClient:       # bot.sessions
    async def ingest(session_key, messages, ...)   # 导入历史对话，不跑模型
    def get(session_key) -> SessionSnapshot        # 只读快照
    def list() -> list[SessionInfo]                # 列出所有会话
    async def restore(snapshot, ...)                # 恢复会话
    def clear / delete / flush(...)

class MemoryClient:        # bot.memory
    def read() -> str                               # 读 MEMORY.md
    def write(text) / append_history(text, ...)    # 改写/追加长期记忆

class RuntimeClient:       # bot.runtime
    @property def model / workspace
    def add_context_provider(p) -> unsubscribe     # 每轮注入上下文
    async def compact_session(session_key)          # 手动触发压缩
```

注意这些客户端**直接操作 04 篇的 `MemoryStore`、03 篇的 `SessionManager`/Consolidator**——SDK 只是给核心能力套了层友好 API，没另起炉灶。

流式则是 `RunStream` + `SDKStreamEmitter` + `SDKStreamingHook` 三件套：Hook 把 agent 生命周期事件（文本 delta、思考 delta、工具开始/完成）转成 `StreamEvent` 塞进队列，`RunStream.stream_events()` 用 `async for` 吐给调用方——**Cursor/OpenAI 风格的流式事件**。这正好对应 02 篇 `_run_core` 里 `on_content_delta` 回调那一层。

---

## 四、OpenAI 兼容 API：让任何 OpenAI 客户端都能调它

`api/server.py` 用 `aiohttp` 起了一个 HTTP 服务，**对外暴露和 OpenAI 一模一样的接口**：

```python
app.router.add_post("/v1/chat/completions", handle_chat_completions)
app.router.add_get("/v1/models", handle_models)
app.router.add_get("/health", handle_health)
```

`handle_chat_completions` 的关键流程：

```python
async def handle_chat_completions(request):
    # 1) 解析：支持 JSON 和 multipart/form-data（可带文件/图片）
    if content_type.startswith("multipart/"):
        text, media_paths, session_id, requested_model = await _parse_multipart(request)
    else:
        body = await request.json()
        stream = body.get("stream", False)
        text, media_paths = _parse_json_content(body)
        session_id = body.get("session_id")
    # 2) 只允许配置的单一模型名（保持简单）
    if requested_model and requested_model != model_name:
        return _error_json(400, f"Only configured model '{model_name}' is available")
    session_key = f"api:{session_id}" if session_id else API_SESSION_KEY
    # 3) 按会话加锁（呼应 02 篇同会话串行）
    session_lock = session_locks.setdefault(session_key, asyncio.Lock())

    if stream:
        # SSE 流式：起一个 task 跑 agent，token 通过 queue 传给 SSE writer
        resp = web.StreamResponse(); resp.content_type = "text/event-stream"
        await resp.prepare(request)
        queue: asyncio.Queue[str | None] = asyncio.Queue()
        async def _on_stream(token): await queue.put(token)
        async def _run():
            async with session_lock:
                response = await asyncio.wait_for(
                    agent_loop.process_direct(content=text, media=..., session_key=session_key,
                                               channel="api", chat_id=API_CHAT_ID,
                                               on_stream=_on_stream, on_stream_end=_on_stream_end),
                    timeout=timeout_s)
                ...
        task = asyncio.create_task(_run())
        while True:
            token = await queue.get()
            if token is None: break
            await resp.write(_sse_chunk(token, model_name, chunk_id))   # 拼 OpenAI SSE 块
        ...
    else:
        # 非流式：直接拿结果，包成 OpenAI 的 chat.completion 响应
        async with session_lock:
            response = await asyncio.wait_for(agent_loop.process_direct(...), timeout=timeout_s)
        return web.json_response(_chat_completion_response(response_text, model_name, ...))
```

最值得点出的：**API 层不自己实现任何 Agent 逻辑**，它只是把 HTTP 请求翻译成 `agent_loop.process_direct(...)`——也就是 02 篇 `AgentLoop` 的直接入口。OpenAI 的 `messages`/`model`/`stream` 字段被映射到 nanobot 的 `process_direct` 参数，`on_stream` 回调把流式 token 重新包成 SSE 的 `data: {...}` 块。于是**任何 OpenAI SDK（curl、langchain、你自己的脚本）都能零改造地调 nanobot**。

> 互通设计很克制：API 只暴露「单一已配置模型」，不支持临时换模型——因为 nanobot 的模型选择走配置/Provider 层（07 篇），HTTP 层不该越权。

---

## 五、cron：让 Agent 定时自己跑

`cron/` 提供定时任务。调度类型支持三种（`_compute_next_run`）：

```python
def _compute_next_run(schedule, now_ms):
    if schedule.kind == "at":       return schedule.at_ms if future else None   # 一次性
    if schedule.kind == "every":    return now_ms + schedule.every_ms          # 每 N 毫秒
    if schedule.kind == "cron":     # 标准 cron 表达式 + 时区
        cron = croniter(schedule.expr, base_dt); return int(cron.get_next(datetime).timestamp()*1000)
```

任务持久化用 `FileLock` 防并发损坏（`cron/service.py` 头）。真正触发时，`bound_runner.py` 的 `run_bound_cron_job` 把任务渲染成一条提醒消息，**作为正常会话轮次提交**：

```python
async def run_bound_cron_job(job, *, agent, cron):
    prompt = render_template("agent/cron_reminder.md", message=job.payload.message)
    channel, chat_id, metadata = _bound_session_delivery_context(job, ...)
    metadata[CRON_TRIGGER_META] = {"job_id": job.id, "job_name": job.name, "run_id": run_id, ...}
    metadata[CRON_DEFER_UNTIL_IDLE_META] = True     # 等会话空闲再注入
    resp = await agent.submit_cron_turn(            # ← 又是入站一条消息！
        InboundMessage(channel=channel, chat_id=chat_id, content=prompt,
                       session_key=job.payload.session_key, metadata=metadata))
```

又是那个统一结论：**cron 不调用引擎内部，而是构造一条带 `CRON_TRIGGER_META` 标记的 `InboundMessage`，经 `submit_cron_turn` 丢回总线**（和 02 篇「用户新消息」、08 篇「子 Agent 回汇」同一个入口）。`CRON_DEFER_UNTIL_IDLE_META` 还让它「会话空闲时才注入」，避免打断正在进行的对话。

---

## 六、四个表面，一条汇流

![nanobot 09 四个表面汇流到核心](/images/nanobot-09-productize.svg)

把所有表面放在一起看，它们的共同终点都是 `MessageBus` + `AgentLoop`：

| 表面 | 入口动作 | 落到核心的方式 |
|---|---|---|
| Channel（Telegram/飞书…） | 平台收到消息 → `_handle_message` | `bus.publish_inbound(InboundMessage)` |
| OpenAI API | HTTP `POST /v1/chat/completions` | `agent_loop.process_direct(...)` |
| Python SDK | `bot.sessions` / `bot.runtime` | 直接操作 `AgentLoop` / `process_direct` |
| Cron | 定时到点 → `run_bound_cron_job` | `agent.submit_cron_turn(InboundMessage(...))` |

`process_direct` 内部其实就是「手动构造一条 InboundMessage 然后走 `_dispatch` → `_process_message` 那套 staging 流水线」（02 篇讲过）。所以四种表面**本质上都等价于「喂一条消息给 AgentLoop」**——这就是 nanobot 产品化的全部秘密：核心极简，表面只是不同形状的「消息生产者」。

这也解释了为什么前面 8 篇那么重要：你读懂了 `AgentLoop` 怎么消费消息、引擎怎么跑、上下文/记忆/工具/Provider/子 Agent 怎么挂，那么 Channel/API/SDK/Cron 这 20+ 个外围模块，无非是「换着法子生产消息」而已，每个都不需要重读引擎。

---

## 七、系列结语

从 00 路线图到 09 产品化，我们完整读透了 nanobot 这个小核心框架：

- **00/01**：定位与架构总览
- **02**：双引擎 `AgentLoop`（产品层）× `AgentRunner`（引擎），靠 `AgentRunSpec` 解耦
- **03**：上下文工程 `ContextBuilder` + `ContextGovernor`（拼装一次、治理每轮、真相视图分离）
- **04**：记忆双系统（会话压缩 `Consolidator`/`AutoCompact` + 长期 `Dream`/`GitStore`）
- **05**：工具系统（base/registry/loader + shell/filesystem/web + SSRF/workspace 双边界）
- **06**：MCP（外部 Server 被翻译成 `mcp_xxx` 工具，复用 Tool 机制）
- **07**：Provider（一张 `ProviderSpec` 表驱动几十家后端 + `FallbackProvider` 透明降级）
- **08**：多 Agent（子 Agent 复用同一引擎 + MessageBus 回汇）
- **09**：产品化（Channel/API/SDK/Cron 四个表面汇流到核心）

nanobot 的设计哲学可以凝练成一句话：**「小核心 + 消息总线 + 声明式配置，让每一种能力都成为可插拔、可复用、且最终殊途同归于总线的组件」**。希望这个系列帮你建立「读一个 Agent 框架源码」的完整方法论——不只是知道它有什么，而是看清它**为什么这么分层、消息怎么流、边界在哪里**。

（系列完）
