---
title: "nanobot 源码解读 · 06 | MCP 集成：把任何外部服务变成 Agent 的工具"
date: 2026-08-09
draft: false
weight: 806
tags: ["nanobot", "源码解读", "MCP", "Agent框架"]
summary: "逐函数拆解 nanobot 的 MCP 客户端（agent/tools/mcp.py，1573 行）：从配置到 ClientSession 的启动建连、stdio/sse/streamableHttp 三种传输、mcp_{server}_{tool} 命名与 allowlist 注册、MCPToolWrapper 如何作为普通 Tool 复用 05 篇的全部机制、MCP JSON Schema 到 OpenAI 函数 schema 的翻译、调用时的超时/取消/重连韧性、连接所有权模型、以及不重启即热重载。配真实代码、消息流向图与一个 filesystem server 的完整例子。"
ShowToc: true
---

> 系列位置：本篇是 nanobot 源码解读系列的第 06 篇，承接 **05 · 工具系统与安全边界**。建议先读完 05，因为本篇最重要的一个结论就是——**MCP 工具在 nanobot 里根本不是"特殊工具"，它只是一颗碰巧会走网络的普通 `Tool`**。

## 一、MCP 是什么，nanobot 在里面扮演什么角色

**MCP（Model Context Protocol，模型上下文协议）** 是 Anthropic 推出的一个开放标准，本质是一套跑在 **JSON-RPC 2.0** 之上的"能力交换协议"。一个 MCP **Server** 可以对外暴露三类能力：

- **Tools**：可被模型调用的函数（比如"查数据库""调 GitHub API"）；
- **Resources**：可被读取的数据源（比如一个文件、一段配置）；
- **Prompts**：预置的提示词模板。

而 nanobot 在这里扮演的是 **MCP Client（客户端）**：它连上这些 Server，把远端暴露的 Tools/Resources/Prompts **翻译成自己内部的 `Tool` 对象**，注册进之前 05 篇讲过的 `ToolRegistry`。从此，模型完全不知道一个工具是"本地内置的"还是"从某个 MCP Server 拉来的"——对引擎来说它们一视同仁。

这就是读完 `agent/tools/mcp.py`（1573 行）后最该记住的一句话：**MCP 在 nanobot 里不是一个平行的新机制，而是"Tool 的一种远程来源"。** 理解了这一点，后面很多设计就顺了。

## 二、启动建连：从配置到 ClientSession

建连的起点在 `AgentLoop`。nanobot 在消息循环正式启动前，先调一次 `_connect_mcp()`：

```python
# agent/agent.py (loop.py:738)
async def _connect_mcp(self) -> None:
    """建立到所有已配置 MCP 服务器的连接。
    连接成功后，MCP 工具会被注册进 self.tools，LLM 就能在回合中调用它们。
    由网关在 loop 启动时(首次 run 前)调用一次。"""
    await agent_context.connect_mcp(self, self.tools)
```

```python
# agent/loop.py:1368 —— 在 run() 的主循环前
self._running = True
try:
    await self._connect_mcp()  # 先建好 MCP 连接，工具才可用
    logger.info("Agent loop started")
    while self._running:
        ...
```

调用链是：`connect_mcp` → `connect_mcp_servers(mcp_servers, registry)` → 对每个 server 调 `connect_single_server` → 内层 `open_single_server` 真正干活。

`open_single_server`（mcp.py:977）做了几件关键的事，我们拆开看。

### 2.1 传输类型推断与三种连接

```python
# mcp.py:984
transport_type = cfg.type
if not transport_type:
    if cfg.command:
        transport_type = "stdio"                      # 有 command → 本地子进程
    elif cfg.url:
        transport_type = (
            "sse" if cfg.url.rstrip("/").endswith("/sse") else "streamableHttp"
        )
    else:
        logger.warning("MCP server '{}': no command or url configured, skipping", name)
        await server_stack.aclose()
        return name, None
```

MCP 有三种传输，nanobot 全支持：

| 传输 | 形态 | 适用 |
|---|---|---|
| `stdio` | 启动一个本地子进程，通过 stdin/stdout 读写 JSON-RPC | 本地工具（如 filesystem、git server） |
| `sse` | 连一个 HTTP SSE 端点（`.../sse`） | 远程服务（老式） |
| `streamableHttp` | 连一个 HTTP 流式端点 | 远程服务（新式，推荐） |

对 HTTP 类（sse/streamableHttp），**第一步永远是安全校验**：

```python
# mcp.py:997
if transport_type in {"sse", "streamableHttp"}:
    ok, error = validate_url_target(cfg.url)          # 05 篇的 SSRF 黑名单
    if not ok:
        logger.warning("MCP server '{}': blocked unsafe URL {} ({})", name, _redact_url(cfg.url), error)
        await server_stack.aclose()
        return name, None
```

注意它调的就是 05 篇 `security/network.py` 里的 `validate_url_target`——**MCP 层不重新发明安全边界，直接复用**。而且服务端 URL 可能带 `?token=` 或 `user:pass@`，日志里用 `_redact_url` 脱敏，绝不把密钥打进日志。

### 2.2 一个容易被低估的细节：先 TCP 探针，再进传输

```python
# mcp.py:1023 / 1051
if not await _probe_http_url(cfg.url):
    logger.warning("MCP server '{}': {} unreachable, skipping", name, _redact_url(cfg.url))
    await server_stack.aclose()
    return name, None
```

`_probe_http_url`（mcp.py:202）先用 `asyncio.open_connection` 做一个轻量 TCP 连通性探测。**为什么多此一举？** 注释说得很直白：如果直接进 `streamable_http_client`/`sse_client`，而这些 anyio 任务组在端口不通时会抛出 `ExceptionGroup` 甚至 `RuntimeError`，**可能逃出调用方的 try/except、把整个事件循环搞崩**。所以 nanobot 先用一个可控的 TCP 探针挡在前面——这是"生产级健壮性"的一个小但关键的取舍。

### 2.3 复用 05 篇的 DNS 钉死

```python
# mcp.py:254
def _pinned_transport_kwargs() -> dict[str, Any]:
    kwargs: dict[str, Any] = {"transport": PinnedDNSAsyncTransport()}  # 05 篇的防 rebinding 传输
    mounts = httpx_env_proxy_mounts()
    if mounts:
        kwargs["mounts"] = mounts
    return kwargs
```

HTTP 类传输都挂上 `PinnedDNSAsyncTransport`（05 篇讲过的 SSRF 第二道防线：把 DNS 解析结果钉死，防 TOCTOU/DNS rebinding）。而且每个出站请求还有一层事件钩子：

```python
# mcp.py:262
async def _validate_mcp_request_url(request: httpx.Request) -> None:
    """Validate each outgoing MCP HTTP request, including redirect targets."""
    ok, error = validate_url_target(str(request.url))
    if not ok:
        raise httpx.RequestError(
            f"Blocked unsafe MCP URL {_redact_url(str(request.url))} ({error})", request=request
        )
```

**重定向目标也会被逐跳重验**——因为 `follow_redirects=True` 下，一次转发就可能把请求引到内网。这一点和 05 篇 `web.py` 的 `_stream_with_safe_redirects` 思路完全一致，只是实现位置不同（MCP 用 httpx 的 `event_hooks`，web 工具用 `follow_redirects=False` 手动控制）。

### 2.4 Windows 启动器包裹

```python
# mcp.py:277
def _normalize_windows_stdio_command(command, args, env):
    ...
    # 对 npx/npm/pnpm/yarn/bunx 以及 .cmd/.bat，包一层 cmd.exe /d /c
    return comspec, ["/d", "/c", command, *normalized_args], env
```

很多 MCP Server 是 `npx @some/package` 这种 Node 脚本。在 Windows 上直接当可执行文件拉起会失败，所以 nanobot 把这些"shell 启动器"统一包进 `cmd.exe /d /c ...`。这是**跨平台兼容性**的细节，macOS/Linux 直接原样返回。

### 2.5 真正建立会话

无论哪种传输，最后三步是统一的：

```python
# mcp.py:1073
read = _filter_malformed_mcp_progress_notifications(read, name)   # 过滤畸形 progress 通知
session = await server_stack.enter_async_context(ClientSession(read, write))
await session.initialize()                                        # 握手

tools = await session.list_tools()                               # 拉工具清单
```

`session.initialize()` 是 MCP 的握手（交换协议版本、能力）。`_filter_malformed_mcp_progress_notifications` 是个有意思的健壮性措施：有些 Server 发"不规范的 progress 通知"（缺 `progressToken`），会干扰 SDK 的流解析，nanobot 用一个异步迭代器过滤器把它们**静默丢弃**，不让脏数据影响主流程。

## 三、注册成工具：命名与 allowlist

拉到 `tools` 清单后，逐个包装注册。这里有两件事值得细看。

### 3.1 命名：`mcp_{server}_{tool}`

```python
# mcp.py:1083
available_wrapped_names = [_sanitize_mcp_tool_name(f"mcp_{name}_{tool_def.name}") for tool_def in tools.tools]
```

一个工具在远端可能叫 `read_file`，但 nanobot 给它套上 `mcp_<server>_` 前缀，变成 `mcp_filesystem_read_file`。**为什么？** 因为你可能同时挂两个 filesystem server，不加深前缀就会撞名。清洗规则在 `_sanitize_mcp_tool_name`（mcp.py:177）：

```python
# mcp.py:158
def _sanitize_name(name: str) -> str:
    return _SANITIZE_RE.sub("_", re.sub(r"[^a-zA-Z0-9_-]", "_", name))
```

```python
# mcp.py:167
def _limit_tool_name(name: str, max_length: int = _MAX_TOOL_NAME_LENGTH) -> str:
    if len(name) <= max_length:
        return name
    digest = hashlib.sha1(name.encode("utf-8")).hexdigest()[:_HASH_LENGTH]   # 超 64 字符用 sha1 截断
    prefix_length = max_length - _HASH_LENGTH - 1
    return f"{name[:prefix_length]}_{digest}"
```

模型厂商（Anthropic/OpenAI）对工具名只允许 `[a-zA-Z0-9_-]`，且长度有限。所以：非法字符换成下划线、连续下划线合并；超过 64 字符则用 `sha1` 前缀哈希，**保证名字既合法又唯一**（哈希保证不同长名不会撞成同一个短名）。

### 3.2 allowlist 与"资源/提示仅全开时注册"

```python
# mcp.py:1078
enabled_tools = set(cfg.enabled_tools)
allow_all_tools = "*" in enabled_tools
...
for tool_def in tools.tools:
    wrapped_name = _sanitize_mcp_tool_name(f"mcp_{name}_{tool_def.name}")
    if (not allow_all_tools
        and tool_def.name not in enabled_tools
        and wrapped_name not in enabled_tools):
        continue   # 不在白名单 → 跳过

wrapper = MCPToolWrapper(session, name, tool_def, tool_timeout=cfg.tool_timeout)
registry.register(wrapper)
```

配置里可以写 `enabledTools: ["*"]`（全开）或一个具体清单。更微妙的是 **resources/prompts 的注册策略**：

```python
# mcp.py:1119
# Only register resources and prompts when no tool restriction is active.
# enabledTools is a per-tool allowlist; resources and prompts have no
# equivalent name filter, so they must be skipped whenever the operator
# specified a tool subset.
register_extras = allow_all_tools
if register_extras:
    # 注册 MCPResourceWrapper / MCPPromptWrapper
else:
    logger.info("MCP server '{}': skipping resource/prompt registration ...", name)
```

意思是：只要你用 `enabledTools` 限制了工具子集（哪怕是具体几个），就**不注册该 server 的 resources/prompts**——因为 resources/prompts 没有对应的名字过滤器，注册它们会"越权"突破你"只想要这几个工具"的意图。这是一种**最小权限**的自觉：能力过滤必须一致，不能工具收紧了、资源却全开着。

## 四、三种包装器：它们都是普通 `Tool`

这是全篇最关键的架构洞见。`MCPToolWrapper` / `MCPResourceWrapper` / `MCPPromptWrapper` **全部继承 `_MCPWrapperBase`，而 `_MCPWrapperBase` 继承 `Tool`**（就是 05 篇那个抽象基类）：

```python
# mcp.py:467
class _MCPWrapperBase(Tool):
    """Common reconnect handling for wrappers bound to one MCP server session."""
    _plugin_discoverable = False
```

也就是说，MCP 工具**就是** `Tool` 的实例，拥有和 `ShellTool`/`WebFetchTool` 完全一样的接口：`name` / `description` / `parameters` / `execute` / `read_only` / `concurrency_safe` / `exclusive`。

回顾 05 篇 `runner.py` 的 `_run_tool`，它对待 MCP 工具**没有任何特殊分支**——从注册表取出来、走 `prepare_call` 校验、调 `execute`、把 `ToolResult` 塞回消息流。整套机制（并发分组、安全分类、超时、错误重试引导）**原样复用**。

`_MCPWrapperBase` 自己只多了一件事：**重连句柄**。

```python
# mcp.py:475
def _set_mcp_connection(self, session, server_name):
    self._session = session
    self._server_name = server_name
    self._reconnect = None

def set_reconnect_handler(self, reconnect):
    self._reconnect = reconnect
```

`_plugin_discoverable = False` 也很重要：MCP 工具**不参与** 05 篇讲过的 `ToolLoader` 的 `pkgutil` 自动扫描发现（它们是运行时动态注册进来的，不是模块里静态定义的类）。

### 4.1 MCPToolWrapper 的构造：把远端 schema 翻成 OpenAI schema

```python
# mcp.py:563
class MCPToolWrapper(_MCPWrapperBase):
    def __init__(self, session, server_name, tool_def, tool_timeout=30):
        self._set_mcp_connection(session, server_name)
        self._original_name = tool_def.name                          # 远端真名，调用时用
        self._name = _sanitize_mcp_tool_name(f"mcp_{server_name}_{tool_def.name}")
        self._description = tool_def.description or tool_def.name
        raw_schema = tool_def.inputSchema or {"type": "object", "properties": {}}
        self._parameters = _normalize_schema_for_openai(raw_schema)  # ← 翻译！
        self._tool_timeout = tool_timeout
```

`tool_def.inputSchema` 是 MCP 的 JSON Schema（2020-12，支持 `$ref`/`nullable`/`oneOf` 等），但 OpenAI/Anthropic 的函数 schema 更严格。所以 `execute` 之前先过一道翻译——这就是下一节。

## 五、Schema 翻译：MCP JSON Schema → OpenAI function schema

`_normalize_schema_for_openai`（mcp.py:459）做两件事：

```python
# mcp.py:459
def _normalize_schema_for_openai(schema):
    if not isinstance(schema, dict):
        return {"type": "object", "properties": {}}
    schema_mapping = cast(dict[str, Any], schema)
    return _normalize_nullable_schema(_rewrite_local_schema_refs(schema_mapping))
```

**其一，处理 `nullable`**（mcp.py:408）。MCP 常用 `["string", "null"]` 这种类型数组表示可空，但很多厂商 schema 不认数组类型。于是 `_normalize_nullable_schema` 把它压平：

```python
# mcp.py:411
raw_type = normalized.get("type")
if isinstance(raw_type, list):
    type_values = cast(list[Any], raw_type)
    non_null = [item for item in type_values if item != "null"]
    if "null" in type_values and len(non_null) == 1:
        normalized["type"] = non_null[0]
        normalized["nullable"] = True
```

并且对 `oneOf`/`anyOf` 里的可空分支做同样的合并，再**递归**处理 `properties`/`items`/`$defs`。

**其二，把本地 `$ref` 提升为 `$defs`**（mcp.py:352）。MCP 工具 schema 经常用 `#/definitions/xxx` 这种本地 JSON Pointer 引用。OpenAI 函数 schema 认的是 `$defs`。`_rewrite_local_schema_refs` 把每个本地 ref 解引用、生成一个 `ref_<sha256 12位>` 的名字塞进 `$defs`，并把原 ref 改写成 `#/$defs/<name>`。

```python
# mcp.py:385
name = f"ref_{hashlib.sha256(ref.encode()).hexdigest()[:12]}"   # 用哈希保证名字稳定唯一
generated_defs[name] = {}
generated_defs[name] = rewrite(target)                          # 先占坑再递归，处理自引用
```

**"先占坑再递归"** 这个写法是为了让递归引用（schema 里 A 引用 A）能正常终止，而不是无限递归。这又是一个"看起来平平无奇、写错就崩"的细节。

> 一句话：这套翻译存在的理由，就是 **MCP 协议宽松、模型 API 严格**。nanobot 在边界处把宽松的那侧"收口"成严格的那侧，让工具定义既能被远端 Server 理解，也能被 OpenAI/Anthropic 干净接收。

## 六、调用与韧性：超时、取消、重连、图片落盘

`MCPToolWrapper.execute`（mcp.py:590）是本篇最精彩的一段——它把"远程调用会出的所有幺蛾子"都安排得明明白白：

```python
# mcp.py:590
async def execute(self, **kwargs: Any) -> str:
    retried_transient = False
    refreshed_session = False
    while True:
        try:
            result = await asyncio.wait_for(
                self._session.call_tool(self._original_name, arguments=kwargs),
                timeout=self._tool_timeout,
            )
        except asyncio.TimeoutError:
            return ToolResult.error(f"(MCP tool call timed out after {self._tool_timeout}s)")
        except asyncio.CancelledError:
            if task_is_cancelling():           # 只有"外部真取消"（如用户 /stop）才往上抛
                raise
            return ToolResult.error("(MCP tool call was cancelled)")   # SDK 泄漏的取消则吞掉
        except Exception as exc:
            if await self._refresh_session_after_termination(exc, refreshed_session, "tool"):
                refreshed_session = True       # 会话断了 → 自动重连后重试
                continue
            if _is_transient(exc):
                if not retried_transient:
                    retried_transient = True
                    await asyncio.sleep(1)      # 瞬时错误（连接抖动）：退避 1s 重试一次
                    continue
                return ToolResult.error(f"(MCP tool call failed after retry: {type(exc).__name__})")
            return ToolResult.error(f"(MCP tool call failed: {type(exc).__name__})")
        else:
            try:
                rendered = self._render_call_result(result.content, kwargs)
                if getattr(result, "isError", False):
                    return ToolResult.error(rendered)
                return rendered
            except Exception as exc:
                return ToolResult.error(f"(MCP tool returned malformed content: {type(exc).__name__})")
```

这段 `execute` 处理了四种情况，每一类都有对应策略：

| 异常 | 处理 | 设计意图 |
|---|---|---|
| `asyncio.TimeoutError` | 返回 `ToolResult.error` | 不让单次调用卡死整个回合（超时由 `tool_timeout` 控制，默认 30s） |
| `CancelledError` | 外部取消才抛；SDK 泄漏的取消则吞掉返回 error | MCP SDK 的 anyio cancel scope 会"泄漏"取消，需甄别 |
| `_is_session_terminated` | 调 `_refresh_session_after_termination` 重连后重试 | **连接中途断了，自动重建会话再试一次**，对用户无感 |
| `_is_transient`（连接重置/断开等） | 退避 1s 重试一次，仍失败才放弃 | 网络抖动是瞬时的，重一次大概率就好 |

`_TRANSIENT_EXC_NAMES`（mcp.py:46）列得很全：`ClosedResourceError` / `BrokenPipeError` / `ConnectionResetError` / `ConnectionRefusedError` 等。

### 6.1 结果渲染：图片绝不进上下文

```python
# mcp.py:667
def _render_call_result(self, content, arguments):
    text_parts = []
    artifacts = []
    for block in content:
        if isinstance(block, types.TextContent):
            text_parts.append(block.text)
            continue
        data_url = _image_block_data_url(block, types)     # 解出 image 的 base64
        if data_url is not None:
            stored = self._store_image_block(data_url, arguments)  # 落盘成 artifact 文件
            if stored is not None:
                artifacts.append(stored)
            continue
        text_parts.append(str(block))
    if artifacts:
        return _mcp_image_tool_result(text_parts, artifacts)  # 只回传 artifact 路径 + 指引
    return "\n".join(text_parts) or "(no output)"
```

这呼应了 03 篇的"工具结果预算"和 05 篇的"图片落盘"思路：**MCP 工具返回的图片，绝不把 base64 塞进模型上下文**（那会撑爆 token）。而是解码后落到本地 artifact 文件，只回传一个路径和一句指引——"用 message 工具把这个 artifact 发给用户"。这跟内置 image_generation 工具的落盘策略一模一样。

## 七、连接生命周期与所有权

远程连接不是"开了就一直开着那么简单"。nanobot 用一个巧妙的"**连接归开启它的任务所有**"模型，避免"在错误 task/循环里关连接"的坑：

```python
# mcp.py:1198
async def connect_single_server(name, cfg):
    loop = asyncio.get_running_loop()
    ready: asyncio.Future[bool] = loop.create_future()
    close_requested = asyncio.Event()

    async def own_connection() -> None:
        stack = None
        try:
            _, stack = await open_single_server(name, cfg)
            if not ready.done():
                ready.set_result(stack is not None)
            if stack is not None:
                await close_requested.wait()          # 阻塞，直到有人要关
        finally:
            if stack is not None:
                await stack.aclose()                 # 由"拥有者任务"负责关闭

    owner = asyncio.create_task(own_connection(), name=f"mcp:{name}")
    connection = _OwnedMCPConnection(owner, close_requested)
    ...
```

```python
# mcp.py:70
class _OwnedMCPConnection:
    """Close an MCP transport from the task that originally opened it."""
    async def aclose(self):
        self._close_requested.set()                  # 通知拥有者任务"该关了"
        try:
            await asyncio.shield(self._owner)        # 等它自己跑完清理，但不被外部取消打断
        except asyncio.CancelledError:
            if not self._owner.cancelled():
                raise
```

为什么这么绕？因为 asyncio 里"谁开的连接、谁负责关"是个经典坑：如果在另一个 task 里直接 `transport.close()`，可能踩到事件循环/任务的归属问题。`_OwnedMCPConnection.aclose()` 不直接关，而是**置一个 Event 让拥有者任务自己退出循环并 `stack.aclose()`**，再用 `asyncio.shield` 等它清理完。这是把"连接生命周期"钉死在正确 owner 上的标准做法。

## 八、热重载与自动重连：不重启即生效

MCP 配置经常会改（加一个 server、换 URL）。nanobot 支持**不重启进程**就生效，靠一套运行时控制机制：

```python
# mcp.py:1289
async def reload_servers(state, registry):
    """Reconcile live MCP connections with the current config file."""
    # 重新读配置 → 对比 current vs next
    #   removed  = 当前有、新配置没有 → 注销工具 + 关连接
    #   added    = 新配置有、当前没有 → 新建连
    #   changed  = 签名变了 → 先关后建
    ...
    connected = await connect_mcp_servers(to_connect, registry)
    state._mcp_stacks.update(connected)
    _attach_reconnect_handlers(state, registry, connected)
```

触发入口是 WebUI/设置页发一条"运行时控制消息"：

```python
# mcp.py:1387
async def request_mcp_reload(bus, *, timeout=15.0):
    await bus.publish_inbound(InboundMessage(
        channel="system", sender_id="webui-settings", chat_id="runtime",
        content=RUNTIME_CONTROL_MCP_RELOAD,
        metadata={INBOUND_META_RUNTIME_CONTROL: RUNTIME_CONTROL_MCP_RELOAD, RUNTIME_CONTROL_ACK: ack},
    ))
    result = await asyncio.wait_for(ack, timeout=timeout)
```

而 `AgentLoop` 在主循环里消费消息时，会先问 `handle_runtime_control` 是不是这类控制消息（`loop.py:1394`），是就截获处理、不进回合管道。

**自动重连**（运行时断线自愈）则是靠 `_attach_reconnect_handlers` 给每个 `MCPToolWrapper` 装上回调：

```python
# mcp.py:1453
def _attach_reconnect_handlers(state, registry, server_names):
    async def reconnect(server_name, tool_name, stale_tool):
        return await _refresh_terminated_server(state, registry, server_name, tool_name, stale_tool)
    for server_name in server_names:
        for tool_name in list(registry.tool_names):
            tool = registry.get(tool_name)
            if not _tool_belongs_to_server(tool, tool_name, server_name):
                continue
            if isinstance(tool, _MCPWrapperBase):
                tool.set_reconnect_handler(reconnect)   # 把回调种进工具
```

回到 06 节 `execute` 里那句 `self._refresh_session_after_termination`：当调用时发现会话已终止，它就调这个回调 → `_refresh_terminated_server` 注销旧工具、关旧连接、**重建连接**、重挂回调、返回新工具——整个过程对模型透明，只是这一次工具调用多花了点重建时间。

还有一处"懒连接"：`connect_missing_servers`（mcp.py:1256）会在每次消息到来时检查"配置的 server 有没有还没连上"，没连就补连——所以即使启动时某个 server 没起来，后续它恢复后 nanobot 会自己接上。

## 九、安全：MCP 层如何复用 05 的边界

把安全点串一遍，你会发现 MCP 模块**几乎没有自己造轮子**：

- **SSRF**：`validate_url_target`（05 篇 `security/network.py`）对 HTTP 类 server URL 先做黑名单拦截；出站请求的事件钩子逐跳重验重定向目标。
- **DNS rebinding**：`PinnedDNSAsyncTransport`（05 篇）钉死解析 IP。
- **凭据不泄漏**：`_redact_url` 脱敏日志；URL 里的 token 永不打印。
- **最小权限**：`enabledTools` allowlist；资源/提示在工具受限时一律不注册。
- **调用层安全**：MCP 工具作为 `Tool` 进入 05 篇的 `_run_tool`，自动获得并发分组、`read_only`/`exclusive` 标记、结果回写消息流等**全部**处理——和 shell/web/filesystem 工具站在同一条安全流水线上。

> 这正是 nanobot 设计的优雅之处：**安全边界写在"Tool/网络"这一层，MCP 只是把远端能力"接进来"，而不是另开一条绕过安全的通道。**

## 十、完整的消息流向

把 02（引擎）→ 05（工具）→ 06（MCP）串起来，一次"调用 MCP 工具"的全过程如下（手绘图见文末）：

```
[启动期]
配置 mcpServers → AgentLoop._connect_mcp (loop.py:738)
  → connect_mcp_servers → connect_single_server → open_single_server
    → 建传输(stdio/sse/streamableHttp) + ClientSession.initialize()
    → list_tools → 逐个 MCPToolWrapper → registry.register
        工具名 = mcp_{server}_{tool}

[一次用户消息]
用户问"帮我读 /data/report.pdf 的第 3 页"
  → AgentLoop 分发 → ContextBuilder 装配 → AgentRunner 模型循环
    → 模型决定调用 mcp_filesystem_read_pdf
    → runner._run_tool（05 篇：并发分组/安全分类）
      → registry 取出 MCPToolWrapper → Tool.prepare_call 校验参数
      → wrapper.execute(**kwargs)
        → asyncio.wait_for(session.call_tool(原始名, 参数), timeout)
          → 走网络到远端 MCP Server 真正执行
        → 结果(文本/图片) → _render_call_result（图片落盘成 artifact）
        → ToolResult 回写消息流
    → 模型拿到结果，组织回答 → 回复用户
```

## 十一、例子：一个 filesystem MCP server 从配置到被调用

假设 `config.yaml` 里：

```yaml
tools:
  mcpServers:
    fs:
      type: stdio
      command: npx
      args: ["-y", "@modelcontextprotocol/server-filesystem", "/data"]
      enabledTools: ["*"]
```

跑起来时发生的事（对照上面代码）：

1. **启动**：`open_single_server("fs", ...)` 推断 `transport_type = "stdio"`（`cfg.command` 有值）；因为是 stdio 不需要 SSRF 校验；在 Windows 上会把 `npx ...` 包成 `cmd.exe /d /c npx ...`。
2. **建会话**：拉起子进程，建立 stdin/stdout 管道，`ClientSession.initialize()` 握手，`list_tools()` 拿到远端工具（比如 `read_file`/`write_file`/`list_directory`）。
3. **注册**：每个远端工具变成 `mcp_fs_read_file` / `mcp_fs_write_file` / `mcp_fs_list_directory`，以 `MCPToolWrapper` 身份 `registry.register`。`enabledTools: ["*"]`，所以 resources/prompts（若有）也一起注册。
4. **调用**：你问"读 /data/report.pdf"，模型选 `mcp_fs_read_file`，`runner._run_tool` 走标准 `Tool` 路径 → `execute` → `session.call_tool("read_file", {...})` 跨进程调子进程 → 结果文本回写。
5. **断线自愈**：假如这个 `npx` 进程中途崩了，下次调用抛 `BrokenResourceError`（瞬时/终止）。`execute` 检测到会话终止 → 调重连回调 → `_refresh_terminated_server` 重新 `npx` 拉起、重新 `initialize`、重新 `list_tools`、重新注册 → 这次调用重试成功。你完全感知不到中间重启过。
6. **改配置不重启**：你在 WebUI 里把 `args` 改成指向 `/data2`，点"重载 MCP"→ `request_mcp_reload` 发运行时控制消息 → `reload_servers` 发现 `fs` 签名变了 → 注销旧 `mcp_fs_*` 工具、关旧连接、按新配置重建。新消息就能用上新路径。

## 十二、小结

回顾本篇，nanobot 的 MCP 集成核心可以浓缩成三句话：

1. **MCP 是 Tool 的一种远程来源**：`MCPToolWrapper` 继承自 `Tool`，引擎（runner）对它毫无特殊待遇，05 篇的全部工具机制（注册、并发、安全、结果回写）原样复用。
2. **边界处做翻译与收口**：远端 schema（宽松的 JSON Schema 2020-12）在 `execute` 前被翻译成 OpenAI 函数 schema；远端 URL 在连接前被 SSRF 校验 + DNS 钉死；图片结果在回写前落盘。
3. **韧性写在每一层**：启动期 TCP 探针防崩、调用期超时/取消/瞬时重试/会话终止自动重连、运行期不重启即热重载、连接所有权钉在正确 task 上。

这正好印证了 05 篇结尾那句——**安全与韧性不是某个模块的事，而是贯穿"工具来源 → 注册 → 调用"整条流水线的设计纪律**。MCP 只是这条流水线接进来的"最新一种来源"而已。

下一篇 **07 · Provider 与模型路由 / 降级** 会看 `providers/*`：nanobot 如何用一个统一注册表适配多家模型后端，并在主力模型挂掉时切到备用模型。那是另一条"把外部能力收口进统一接口"的故事。

---

![nanobot 06 · MCP 集成的完整消息流向](/images/nanobot-06-mcp.svg)
