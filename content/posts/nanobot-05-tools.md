---
title: "nanobot 源码解读 · 05 | 工具系统与安全边界：Agent 的手脚与护栏"
date: 2026-08-09
draft: false
weight: 805
tags: ["nanobot", "源码解读", "工具系统", "安全边界", "SSRF"]
summary: "逐层拆解 nanobot 的工具体系：Tool 抽象与参数校验（base.py）、注册与发现（registry.py/loader.py）、三类内置工具（shell 命令、文件系统、Web 抓取）各自的执行与安全检查点，以及 runner 如何在每一轮把工具编排起来。重点讲清两条安全边界线——SSRF（security/network.py 的 IP 黑名单 + DNS 钉死）与工作区路径边界（security/workspace_policy.py），并用真实代码与例子说明软边界与硬边界的区别。"
ShowToc: true
---

## 一、工具是 Agent 的「手脚」，安全边界是「护栏」

前面四篇我们把消息从通道走到模型、上下文怎么拼、记忆怎么沉淀都讲透了。但一个 Agent 如果只是「读 Prompt、吐文字」，那它再聪明也只是个聊天机器人。**真正让 Agent 能做事的，是工具（tool）**：读文件、跑命令、查网页、调 API。

nanobot 的工具体系分三层，本篇逐一拆开：

1. **抽象层**（`agent/tools/base.py` + `schema.py`）：一个 `Tool` 长什么样、参数怎么校验、怎么转成 OpenAI 的 function schema。
2. **注册与发现层**（`registry.py` + `loader.py`）：几十个工具怎么被自动扫描、注册进一个全局表，模型每轮从哪拿到工具清单。
3. **具体工具 + 安全边界**（`shell.py` / `filesystem.py` / `web.py` + `security/*`）：每类工具的执行路径，以及每一处「不准越界」的检查点。

最后我会把整条**调用链**在 `runner.py` 里串起来：模型说「我要调工具」之后，到底发生了什么。

> 一句总纲：**工具系统负责「能干什么」，安全边界负责「不准越界到哪」**。两者是同一个文件里紧挨着的两段代码——这是个很值得学的设计取舍。

---

## 二、工具抽象：一个 `Tool` 长什么样

所有工具都继承自 `agent/tools/base.py` 里的 `Tool` 抽象类。先看作者的设计意图——一个最小工具需要暴露四样东西：名字、描述、参数 schema、执行函数。

### 2.1 参数校验：`Schema` 与 `ToolResult`

参数校验用的是一套自研的 JSON Schema 子集（`schema.py`）。`Schema.validate_json_schema_value` 是个静态方法，递归校验一个值是否符合 schema 片段：

```python
# nanobot/agent/tools/schema.py:50
@staticmethod
def validate_json_schema_value(val: Any, schema: dict[str, Any], path: str = "") -> list[str]:
    """返回错误列表（空 = 合法）。"""
    raw_type = schema.get("type")
    nullable = (isinstance(raw_type, list) and "null" in raw_type) or schema.get("nullable", False)
    t = Schema.resolve_json_schema_type(raw_type)   # ['string','null'] -> 'string'
    label = path or "parameter"

    if nullable and val is None:
        return []
    if t == "integer" and (not isinstance(val, int) or isinstance(val, bool)):
        return [f"{label} should be integer"]
    # ... number / string(min/maxLength) / enum / object(required) / array(min/maxItems) ...
    return errors
```

注意它对 `bool` 的特殊处理：`isinstance(True, int)` 在 Python 里是 `True`，所以整数校验里要排除 `bool`，否则 `True` 会被当成整数放行。这种细节正是手写校验容易踩的坑。

工具返回值用 `ToolResult`——它本质是个 `str`，但带一个 `is_error` 标记：

```python
# nanobot/agent/tools/base.py:141
class ToolResult(str):
    """带结构化状态的字符串型工具输出。"""
    is_error: bool
    def __new__(cls, content: str, *, is_error: bool = False) -> ToolResult:
        obj = str.__new__(cls, content)
        obj.is_error = is_error
        return obj
    @classmethod
    def error(cls, content: str) -> ToolResult:
        return cls(content, is_error=True)
```

为什么用 `str` 子类而不是 `(ok, data)` 元组？因为工具结果最终要直接塞回 LLM 的消息流，做成「长得像字符串、但能区分成功/失败」最省事——下游 `is_tool_error_result()` 一眼就知道该不该当错误处理。

### 2.2 工具的四个抽象成员 + 并发标记

`Tool` 抽象类（`base.py:156`）定义了工具契约。除了 `name` / `description` / `parameters` / `execute` 这四个必须实现的成员，还有三个**并发相关标记**特别值得讲——它们在后面 runner 的调度里直接决定「能不能并行跑」：

```python
# nanobot/agent/tools/base.py:186
@property
def read_only(self) -> bool:
    """是否无副作用、可安全并行。"""
    return False

@property
def concurrency_safe(self) -> bool:
    """是否可与其他 concurrency_safe 工具一起跑。"""
    return self.read_only and not self.exclusive

@property
def exclusive(self) -> bool:
    """即使开了并发，这个工具也必须单独跑。"""
    return False
```

逻辑很清晰：**`concurrency_safe = read_only 且 非 exclusive`**。一个只读的查询工具（比如读文件）可以和别的只读工具并行；一个有副作用的写工具、或者声明了 `exclusive` 的工具（比如 `exec` 命令），就必须单独成批、串行执行。这是「性能 vs 正确性」的标准权衡，被固化进了抽象层。

还有两个预处理钩子：

```python
# nanobot/agent/tools/base.py:248
def cast_params(self, params: dict[str, Any]) -> dict[str, Any]:
    """执行前做安全的 schema 驱动类型转换。"""
    ...
def validate_params(self, params: dict[str, Any]) -> list[str]:
    """按 JSON schema 校验；空列表 = 合法。"""
    ...
```

`cast_params` 会把模型可能塞错的字符串（`"42"`）转成整数、`"true"` 转成布尔——因为 LLM 生成的 tool_call 参数经常是「看起来对但类型不对」的字符串。`validate_params` 则做正式校验，提前在「执行之前」拦掉畸形参数。

最后 `to_schema()` 把工具包装成 OpenAI 的 function schema，直接喂给模型：

```python
# nanobot/agent/tools/base.py:303
def to_schema(self) -> dict[str, Any]:
    return {
        "type": "function",
        "function": {
            "name": self.name,
            "description": self.description,
            "parameters": self.parameters,
        },
    }
```

作者还提供了一个 `@tool_parameters({...})` 类装饰器（`base.py:315`），让子类不用手写 `@property def parameters`，直接把 JSON Schema 挂上去即可。这是典型的「减少样板代码」小工具。

---

## 三、注册与发现：几十个工具怎么进系统

有了 `Tool` 抽象，下一步是「把磁盘上所有工具类自动找出来、注册进一张表」。这块代码很能体现框架的「可扩展」设计。

### 3.1 `ToolRegistry`：一张全局工具表

`registry.py` 里的 `ToolRegistry` 就是那张表。它几个关键方法：

```python
# nanobot/agent/tools/registry.py:86
def get_definitions(self) -> list[dict[str, Any]]:
    """稳定排序，方便缓存友好地拼进 Prompt。"""
    if self._cached_definitions is not None:
        return self._cached_definitions
    definitions = [tool.to_schema() for tool in self._tools.values()]
    builtins: list[dict] = []
    mcp_tools: list[dict] = []
    for schema in definitions:
        name = self._schema_name(schema)
        if name.startswith("mcp_"):
            mcp_tools.append(schema)      # MCP 工具排在后面
        else:
            builtins.append(schema)        # 内置工具排前面，作为稳定前缀
    builtins.sort(key=self._schema_name)
    mcp_tools.sort(key=self._schema_name)
    self._cached_definitions = builtins + mcp_tools
    return self._cached_definitions
```

注意它的排序哲学：**内置工具永远排在 MCP 工具前面**，且结果会缓存（`_cached_definitions`），直到下次 `register/unregister` 才失效。为什么重要？因为工具定义是拼进 system prompt 的，如果顺序每次都变，模型的注意力分布也会变——稳定排序 = 可复现。这是「工程细节影响模型表现」的一个小例子。

真正的「解析 + 校验 + 执行」在 `prepare_call` 和 `execute`：

```python
# nanobot/agent/tools/registry.py:111
def prepare_call(self, name, params) -> tuple[Tool | None, Any, str | None]:
    """解析、转换、校验一次工具调用。"""
    tool = self._tools.get(name)
    if not tool:
        suggestion = self._suggest_name(str(name))   # 名字拼错时给建议
        hint = f" Did you mean '{suggestion}'? ..." if suggestion else ""
        return None, params, ToolResult.error(f"Error: Tool '{name}' not found.{hint} ...")
    # 兼容老式 ContextAware 工具的上下文注入（内置工具直接读 ContextVar）
    if isinstance(tool, ContextAware) and (ctx := current_request_context()) is not None:
        tool.set_context(ctx)
    params = self._coerce_params(tool, params)       # 把 "{'a': 1}" 字符串解析成 dict
    if not isinstance(params, dict):
        return tool, params, ToolResult.error(f"Error: parameters must be a JSON object ...")
    cast_params = tool.cast_params(cast(dict, params))
    errors = tool.validate_params(cast_params)
    if errors:
        return tool, cast_params, ToolResult.error(f"Error: Invalid parameters for tool '{name}': " + "; ".join(errors))
    return tool, cast_params, None
```

`execute` 则在 `prepare_call` 之后真正调 `tool.execute(**params)`，并且**任何失败都追加一句提示**：`\n\n[Analyze the error above and try a different approach.]`——这是给模型的「重试引导」，让它看到工具报错后换方法，而不是死循环重试同一个错误调用。

### 3.2 `ToolLoader`：包扫描 + 插件机制

工具怎么被「发现」？`loader.py` 用 `pkgutil` 扫描 `nanobot.agent.tools` 整个包，找出所有 `Tool` 子类：

```python
# nanobot/agent/tools/loader.py:36
def discover(self) -> list[type[Tool]]:
    seen: set[int] = set()
    results: list[type[Tool]] = []
    for _importer, module_name, _ispkg in pkgutil.iter_modules(self._package.__path__):
        if module_name.startswith("_") or module_name in _SKIP_MODULES:   # 跳过 base/schema/...
            continue
        module = importlib.import_module(f".{module_name}", self._package.__name__)
        for attr_name in dir(module):
            attr = getattr(module, attr_name)
            if (isinstance(attr, type) and issubclass(attr, Tool) and attr is not Tool
                    and not attr_name.startswith("_")
                    and not getattr(attr, "__abstractmethods__", None)
                    and getattr(attr, "_plugin_discoverable", True)
                    and id(attr) not in seen):
                seen.add(id(attr))
                results.append(attr)
    results.sort(key=lambda cls: cls.__name__)
    return results
```

`_SKIP_MODULES` 是个写死的集合（`base, schema, registry, context, loader, config, file_state, sandbox, mcp, __init__, runtime_state`），这些模块是「基础设施」不是「工具」，扫出来会误报。

更妙的是**插件机制**——除了内置工具，外部包可以通过 `entry_points` 的 `nanobot.tools` 组注册自己的工具：

```python
# nanobot/agent/tools/loader.py:68
def _discover_plugins(self) -> dict[str, type[Tool]]:
    """从 entry_points('nanobot.tools') 发现外部工具插件。"""
    eps = entry_points(group="nanobot.tools")
    for ep in eps:
        cls = ep.load()
        if isinstance(cls, type) and issubclass(cls, Tool) and ...:
            plugins[ep.name] = cls
    return plugins
```

`load()` 方法把两者合并、做 `scope`/`enabled` 检查、`create()` 后注册进 registry，并对插件套一层 `_LegacyErrorPrefixTool` 兼容老式错误字符串契约。这意味着：nanobot 的工具能力是**可插拔**的——你写个新工具类、在 `pyproject.toml` 里登记一个 entry point，不用改框架一行代码就能被自动加载。

> **这一段与 02 篇的呼应**：`AgentRunner` 通过 `AgentRunSpec.tools` 拿到这张 registry 表（`runner.py:33` 导入 `ToolRegistry`）。也就是说，「工具从哪来」和「引擎怎么用工具」在架构上完全解耦——这正是 02 篇说的「小核心 + 插件式外围」。

---

## 四、shell 工具：命令执行与「最努力的安全护栏」

先看最「危险」的内置工具——执行 shell 命令的 `ExecTool`（`shell.py:167`）。它把「让 Agent 跑命令」这件事做得很周全。

执行入口 `execute`（`shell.py:298`）做参数归一化、超时控制、输出截断：

```python
# nanobot/agent/tools/shell.py:298
async def execute(self, command=None, cmd=None, working_dir=None, workdir=None,
                  timeout=None, shell=None, login=None, yield_time_ms=None,
                  max_output_chars=None, max_output_tokens=None, **kwargs) -> str:
    command = command or cmd
    working_dir = working_dir or workdir
    if not command:
        return ToolResult.error("Error: Missing command. Provide command or cmd.")
    prepared = self._prepare_command(command, working_dir, timeout, shell, login)
    if isinstance(prepared, str):
        return prepared                       # 准备阶段就报错（如护栏拦截）
    # ... 启动子进程、wait_for 超时、kill、输出截断 ...
    if len(result) > max_len:
        half = max_len // 2
        result = (result[:half] + f"\n\n... ({len(result)-max_len:,} chars truncated) ...\n\n" + result[-half:])
    return result
```

值得注意的健壮性细节：超时用 `asyncio.wait_for` 包住 `process.communicate()`，超时就 `_kill_process` 杀掉并回报错；还专门有 `_reap_pid()`「僵尸进程兜底回收」——因为容器里 child-watcher 有时会漏掉回收。这些都不是「让模型能跑命令」必需的代码，而是「让命令跑挂了也不拖垮整个进程」的工程保险。

### 4.1 `_guard_command`：最努力的安全护栏

真正的安全检查在 `_guard_command`（`shell.py:769`）。名字叫「best-effort safety guard」——**作者自己承认这是尽力而为的护栏，不是铁壁**。它做了四件事：

**(1) 黑名单 / 白名单模式匹配**

```python
# nanobot/agent/tools/shell.py:785
segments = self._split_shell_segments(lower)        # 按 && / | / ; 切分顶层命令段
explicitly_allowed = bool(self.allow_patterns) and bool(segments) and all(
    any(re.fullmatch(pattern, segment) for pattern in self.allow_patterns)
    for segment in segments
)
if not explicitly_allowed:
    for pattern in self.deny_patterns:
        if re.search(pattern, lower):
            return ToolResult.error("Error: Command blocked by deny pattern filter")
    if self.allow_patterns:
        return ToolResult.error("Error: Command blocked by allowlist filter (not in allowlist)")
```

设计很讲究：`allow_patterns`（白名单）优先级高于 `deny_patterns`（黑名单）。如果配了白名单，那么**链式命令必须每一段都命中白名单**才放行（比如用户想「在构建目录里 `rm -rf`」时可特例豁免）。没配白名单时，则看黑名单拦截危险命令（如 `rm -rf /`）。

**(2) 命令字符串里的内网 URL 扫描（SSRF 防线之一）**

```python
# nanobot/agent/tools/shell.py:798
from nanobot.security.network import contains_internal_url
if contains_internal_url(cmd, allow_loopback=current_scope_allows_loopback(...)):
    return ToolResult.error("Error: Command blocked by safety guard (internal/private URL detected)")
```

意思是：即使命令本身合法，只要里面包含了 `http://169.254.169.254/`（云元数据地址）这类内网 URL，也直接拦——防止 Agent 被人诱导去「curl 一下云上的密钥服务」。

**(3) 路径穿越检测**

```python
# nanobot/agent/tools/shell.py:810
if should_restrict:
    if "..\\" in cmd or "../" in cmd:
        return ToolResult.error("Error: Command blocked by safety guard (path traversal detected)" + _WORKSPACE_BOUNDARY_NOTE)
```

**(4) 绝对路径必须落在工作区内**

命令里出现的每个绝对路径，都会被解析后检查是否 `is_path_within` 工作区（或 media 目录、sandbox 绑定根）。一旦越界，返回硬边界错误——注意末尾拼了 `_WORKSPACE_BOUNDARY_NOTE`（见第六节），明确告诉模型「别用 shell 小技巧绕过，直接问用户」。

> **例子**：用户让 Agent「把 /etc/passwd 的内容读出来分析下」。Agent 生成 `cat /etc/passwd`。护栏检查到绝对路径 `/etc/passwd` 不在工作区 `~/.nanobot/workspace` 内，返回硬边界错误：`Error: Command blocked by safety guard (path outside working dir) (this is a hard policy boundary...)`. 模型于是改口「我无法访问该路径，需要我读工作区里的哪个文件吗？」——护栏把一次潜在的越权变成一次安全的交互。

---

## 五、文件系统工具：工作区边界的「硬守护」

`filesystem.py` 里的工具（读/写/改/列目录）共用一个基类 `_FsTool`（`filesystem.py:33`），它的核心就是**路径边界解析**。

### 5.1 边界在哪：从配置到 `resolve_allowed_path`

`create`（`filesystem.py:82`）把「是否限制在工作区」翻译出来：

```python
# nanobot/agent/tools/filesystem.py:86
agent_workspace = Path(ctx.workspace)
restrict = ctx.config.restrict_to_workspace or ctx.config.exec.sandbox
allowed_dir = agent_workspace if restrict else None
return cls(
    workspace=agent_workspace,
    allowed_dir=allowed_dir,
    extra_read_allowed_dirs=[BUILTIN_SKILLS_DIR, resolved_agent_workspace / "skills"],
    extra_read_allowed_files=[resolved_agent_workspace / "memory" / "history.jsonl"],
    restrict_to_workspace=ctx.config.restrict_to_workspace,
    sandbox_restricts_workspace=sandbox_restricts,
)
```

关键点：如果开了 `restrict_to_workspace` 或 sandbox，那么 `allowed_dir = 工作区`，任何读写都被锁死在工作区内；否则 `allowed_dir = None`，不限制。同时，作者**显式放行两个白名单**：内置技能目录、以及 `memory/history.jsonl`（03 篇讲过的会话历史日志，Agent 自己读自己的历史是允许的）。

读和写走不同的解析入口：

```python
# nanobot/agent/tools/filesystem.py:150
def _resolve_read(self, path: str) -> Path:
    return self._resolve_with_extra(path, self._extra_read_allowed_dirs,
                                     self._extra_read_allowed_files,
                                     include_media_dir=True, extra_files_require_allowed_root=True)

def _resolve_write(self, path: str) -> Path:
    return self._resolve_with_extra(path, self._extra_write_allowed_dirs,
                                     self._extra_write_allowed_files, include_media_dir=False)
```

读允许碰 media 目录、技能目录、history.jsonl；写**不允许碰 media**（避免 Agent 覆盖用户上传的媒体），且写权限更窄。最终都落到 `security/workspace_policy.py` 的 `resolve_allowed_path`：

```python
# nanobot/security/workspace_policy.py:96
def resolve_allowed_path(path, *, workspace=None, allowed_root=None,
                          extra_allowed_roots=None, extra_allowed_files=None, strict=False) -> Path:
    resolved = resolve_path(path, workspace, strict=False)
    roots = []
    if allowed_root is not None:
        roots.append(allowed_root)
    roots.extend(extra_allowed_roots or [])
    exact_allowed = bool(files) and _is_path_exactly_allowed(...)
    if not is_path_allowed(resolved, roots) and not exact_allowed:
        boundary = Path(allowed_root).expanduser() if allowed_root is not None else "allowed files"
        raise WorkspaceBoundaryError(
            f"Path {path} is outside allowed directory {boundary}" + WORKSPACE_BOUNDARY_NOTE
        )
    return resolved
```

`WorkspaceBoundaryError(PermissionError)` 一旦抛出，就会被 runner 的 `_classify_violation` 识别成**硬边界**（见第六节）。

### 5.2 设备文件黑名单

除了路径边界，`filesystem.py` 还硬拦一批危险设备文件（`filesystem.py:179`）：

```python
_BLOCKED_DEVICE_PATHS = frozenset({
    "/dev/zero", "/dev/random", "/dev/urandom", "/dev/full",
    "/dev/stdin", "/dev/stdout", "/dev/stderr",
    "/dev/tty", "/dev/console", "/dev/fd/0", "/dev/fd/1", "/dev/fd/2",
})
```

Agent 可不会「读 `/dev/random` 当普通文件」——但护栏会先挡住。shell 的 `_guard_command` 里也有对应的 `_is_benign_device_path` 放行（比如 `/dev/stderr` 在 Linux 上是 `/proc/self/fd/2` 的符号链接，意图是「写 stderr」而非「碰设备」）。

> **例子**：Agent 想「把项目说明写到 /tmp/notes.txt 让同事能拿到」。在 `restrict_to_workspace=True` 下，`/tmp` 不在工作区，`resolve_allowed_path` 抛 `WorkspaceBoundaryError`，工具返回硬边界错误。模型随即改写到 `./notes.txt`（工作区内）——护栏既挡住了越界，又引导模型用「正确姿势」完成任务。

---

## 六、Web 工具：SSRF 的三道防线

`web.py` 提供了 `WebFetchTool`（抓网页）和 `WebSearchTool`（搜索）。它们面临一个经典安全难题——**SSRF（服务端请求伪造）**：恶意构造的 URL 可能让 Agent 的服务器去访问内网资源。

### 6.1 第一道防线：scheme + DNS 解析 + IP 黑名单

`web.py` 的 `_validate_url_safe`（`web.py:111`）直接委托给 `security/network.py` 的 `validate_url_target`：

```python
# nanobot/agent/tools/web.py:111
def _validate_url_safe(url: str) -> tuple[bool, str]:
    """带 SSRF 防护的 URL 校验：scheme + 域名 + 解析后的 IP 检查。"""
    from nanobot.security.network import validate_url_target
    return validate_url_target(url)
```

而 `security/network.py` 里有一张写死的**禁止网络段黑名单**（`network.py:16`）：

```python
_BLOCKED_NETWORKS = [
    ipaddress.ip_network("0.0.0.0/8"),
    ipaddress.ip_network("10.0.0.0/8"),          # 私有 A 类
    ipaddress.ip_network("100.64.0.0/10"),       # 运营商级 NAT
    ipaddress.ip_network("127.0.0.0/8"),         # 回环
    ipaddress.ip_network("169.254.0.0/16"),      # 链路本地 / 云元数据
    ipaddress.ip_network("172.16.0.0/12"),       # 私有 B 类
    ipaddress.ip_network("192.168.0.0/16"),      # 私有 C 类
    ipaddress.ip_network("::/128"), ipaddress.ip_network("::1/128"),
    ipaddress.ip_network("fc00::/7"), ipaddress.ip_network("fe80::/10"),
]
```

`resolve_url_target`（`network.py:78`）的逻辑是：**先把 URL 解析成真实 IP，再看这个 IP 是否在黑名单里**。这比「只检查域名字符串」强太多——攻击者不能靠 `evil.com` 解析到 `127.0.0.1` 来绕过，因为最终检查的是解析后的 IP。

### 6.2 第二道防线：DNS 钉死（防 rebinding / TOCTOU）

光「先解析后检查」还不够，存在 **DNS rebinding** 攻击：第一次解析拿到合法公网 IP、检查通过，真正发请求时 DNS 又解析成内网 IP。nanobot 用 `PinnedDNSAsyncTransport`（`network.py:264`）把 DNS **钉死**：

```python
# nanobot/agent/tools/web.py:118
def _resolve_url_safe(url):
    """校验 URL 并返回要钉死的已解析 IP。"""
    return resolve_url_target(url)     # 返回 (ok, error, resolved_ips)

# network.py:264
class PinnedDNSAsyncTransport(httpx.AsyncBaseTransport):
    async def handle_async_request(self, request):
        url = str(request.url)
        ok, error, resolved_ips = resolve_url_target(url, allow_loopback=self._allow_loopback)
        if not ok:
            raise UnsafeURLRequestError(error, request=request)
        async with self._resolver_lock:
            with pin_resolved_url_dns(url, resolved_ips):   # 临时把该域名的 DNS 固定到已验证 IP
                return await self._inner.handle_async_request(request)
```

`pin_resolved_url_dns`（`network.py:214`）在请求期间**临时改写进程级 `socket.getaddrinfo`**，让该 hostname 只返回已验证的 IP。这样即使攻击者换 DNS，本次请求也只会打到当初验证过的公网地址。这是相当专业的 SSRF 防护手法。

### 6.3 第三道防线：每一跳重定向都要重验

HTTP 重定向是 SSRF 的另一个载体——初始 URL 合法，但 302 跳到内网。`web.py` 的 `_stream_with_safe_redirects`（`web.py:190`）对**每一跳重定向都重新校验**：

```python
# nanobot/agent/tools/web.py:197
for _ in range(MAX_REDIRECTS + 1):
    is_valid, error_msg, _ = _resolve_url_safe(current_url)
    if not is_valid:
        return None, None, f"Redirect blocked: {error_msg}"
    stream = client.stream("GET", current_url, follow_redirects=False)   # 手动控制重定向
    response = await stream.__aenter__()
    is_redirect = 300 <= response.status_code < 400
    if not is_redirect:
        return response, stream, None
    next_url = urljoin(str(response.url), response.headers.get("location"))
    is_valid, error_msg = _validate_url_safe(next_url)                   # 下一跳也要过 SSRF
    if not is_valid:
        await stream.__aexit__(None, None, None)
        return None, None, f"Redirect blocked: {error_msg}"
```

注意它用 `follow_redirects=False` **关掉** httpx 的自动跟随，自己手动逐跳校验——因为自动跟随会跳过对中间跳转的 SSRF 检查。

> **例子**：用户让 Agent「抓一下 http://short.url/abc （一个短链服务）」。短链 302 跳到 `http://169.254.169.254/latest/meta-data/`（云厂商实例元数据，里面有临时密钥）。第一道防线初始 URL 合法放行；但 `_stream_with_safe_redirects` 在重定向那跳发现目标 IP 是链路本地段，立刻 `Redirect blocked`。Agent 的反应是「该链接重定向到了一个被安全策略禁止的内部地址，无法抓取」——一次典型的 SSRF 攻击被拦在第三道防线。

---

## 七、调用链：runner 怎么把工具跑起来

前面都是「单个工具内部」的逻辑。现在回到引擎层，看 `AgentRunner` 在每一轮怎么编排多个工具调用（`runner.py:1556`）。

### 7.1 并发分组：能并行的并行，不能的串行

```python
# nanobot/agent/tools/runner.py:1556  (_execute_tools)
batches = self._partition_tool_batches(spec, tool_calls)
for batch in batches:
    if spec.concurrent_tools and len(batch) > 1:
        batch_results = await asyncio.gather(*(self._run_tool(spec, tc, ...) for tc in batch))
    else:
        for tool_call in batch:
            result = await self._run_tool(spec, tool_call, ...)
            tool_results.append(result)
```

`_partition_tool_batches`（`runner.py:1904`）的逻辑就是第二节说的：**`concurrency_safe` 的工具（只读查询）合并进同一批一起 `gather` 并行；有副作用的写操作各自独立成批串行**。非并发模式则每个工具一批。这样「读 3 个文件」可以并行加速，而「写文件 + 删文件」绝不会并行导致竞态。

### 7.2 单个工具的执行与边界分类

`_run_tool`（`runner.py:1618`）是每次工具调用的统一入口，流程是：**重复外查拦截 → prepare_call → execute → 错误分类**。

```python
# nanobot/agent/runner.py:1643
lookup_error = repeated_external_lookup_error(tool_call.name, tool_call.arguments, external_lookup_counts)
if lookup_error:
    # 模型反复查同一个外部资源（如反复 fetch 同一无效 URL），直接拦截防死循环
    ...

tool, params, prep_error = None, tool_call.arguments, None
if callable(prepare_call):
    prepared = prepare_call(tool_call.name, tool_call.arguments)   # registry 里解析+校验
    ...
if prep_error:
    handled = self._classify_violation(raw_text=prep_error, soft_payload=prep_error + hint,
                                       event=event, tool_call=tool_call,
                                       workspace_violation_counts=workspace_violation_counts)
    if handled is not None:
        return handled
    return prep_error + hint, event, (RuntimeError(prep_error) if spec.fail_on_tool_error else None)

await hook.before_execute_tool(context, tool_call, tool, params)
try:
    if tool is not None:
        result = await tool.execute(**params)
    else:
        result = await spec.tools.execute(tool_call.name, params)
except Exception as exc:
    handled = self._classify_violation(raw_text=str(exc), soft_payload=..., event=event, ...)
    if handled is not None:
        return handled
    ...
```

### 7.3 软边界 vs 硬边界

`_classify_violation` 是安全设计的精华。它把工具错误分成两类：

- **硬边界（hard boundary）**：SSRF 拦截、工作区越界（`WorkspaceBoundaryError`）。这类错误**不可重试**——模型再怎么换参数也绕不过去，所以返回一段明确提示（`WORKSPACE_BOUNDARY_NOTE`：「这是硬策略边界，别用 shell 小技巧绕过，需要的话直接问用户」）。
- **软边界（soft boundary）**：普通工具报错（如文件不存在、命令语法错）。这类**可重试**——把错误文本回填给模型（`payload + hint`），模型可以换方法再试。

这种分类直接呼应 03 篇 `ContextGovernor` 的「畸形 tool_call 自愈」：软错误是模型学习的机会，硬错误是系统的红线。

---

## 八、安全边界总账：两层 + 一条哲学

把全篇的安全点收拢，nanobot 的边界其实只有**两层**，都在 `security/` 下：

| 边界 | 文件 | 防线 | 越界后果 |
|---|---|---|---|
| **网络边界（SSRF）** | `security/network.py` | IP 黑名单 + DNS 解析校验 + DNS 钉死 + 重定向逐跳校验 | 请求被拒，返回硬边界错误 |
| **路径边界（workspace）** | `security/workspace_policy.py` | `is_path_within` / `require_path_within` / `resolve_allowed_path` | 抛 `WorkspaceBoundaryError`，返回硬边界错误 |

两条边界在三个具体工具里被调用：shell 的 `_guard_command`、filesystem 的 `_resolve_*`，web 的 `_validate_url_safe`。它们和「工具能干什么」是**同文件紧邻的两段代码**——作者刻意把「能力」和「护栏」放在一起，而不是把安全外包给某个独立模块。

**最重要的一条哲学**，写在 `workspace_policy.py` 开头：

```python
"""Workspace path boundary helpers.
These helpers are application-level guards.  They make path decisions
consistent across tools, but they are not a replacement for an OS sandbox.
"""
```

翻译过来：**这些都是「应用层防护」，不是「操作系统沙箱」的替代品**。也就是说，真正要安全地跑一个能执行 shell 的 Agent，你还得在 OS 层面做隔离（容器、seccomp、非 root 用户、网络策略）。nanobot 的护栏解决的是「Agent 自己别主动越界」和「把明显的攻击意图拦在模型之外」，而不是「即使 Agent 被完全攻陷也能兜住」。理解这个边界，才不会误以为「开了 restrict_to_workspace 就绝对安全」。

---

## 九、消息流向一图总览

把本篇讲的东西串成一条「模型调一次工具」的流向：

![nanobot 05 工具系统与调用链路](/images/nanobot-05-tools.svg)

核心结论：**工具系统 = 抽象（Tool）+ 注册表（Registry/Loader）+ 三类实现（shell/filesystem/web）；安全边界 = 网络层 SSRF + 路径层 workspace，由 runner 的 `_run_tool` 在每次调用时通过 `_classify_violation` 分成软/硬两类处理。** 下一轮（06 篇）我们会看 MCP——如何把「外部 MCP Server」也变成这张 registry 表里的一个 `mcp_xxx` 工具，复用本篇讲的整套机制。

---

## 十、小结与下篇预告

- **抽象层** `base.py`：`Tool` 四成员 + `read_only/concurrency_safe/exclusive` 并发标记 + `cast/validate` 预处理 + `to_schema` 转 OpenAI 格式。
- **注册发现层** `registry.py`/`loader.py`：包扫描自动发现、稳定排序缓存、`entry_points` 插件可插拔。
- **三类工具**：shell（最努力护栏：deny/allow 模式 + 内网 URL 扫描 + 路径穿越 + 工作区边界）、filesystem（路径硬守护 + 设备文件黑名单 + 读/写白名单）、web（SSRF 三道防线：IP 黑名单 / DNS 钉死 / 重定向逐跳校验）。
- **调用链** `runner.py`：`_execute_tools` 并发分组 → `_run_tool` 重复外查拦截 → `prepare_call` → `execute` → `_classify_violation` 软/硬边界。
- **安全哲学**：应用层防护，非 OS 沙箱替代品；硬边界不可重试、软边界可重试。

**下一篇（06 · MCP 集成）**：`agent/tools/mcp.py`（1573 行，全仓最大的工具文件）是怎么把一个外部 MCP Server「翻译」成 nanobot 的 `Tool`，从而让本篇所有机制（注册、并发、安全分类）都能直接复用的。我们会看 MCP 的 stdio/SSE 两种连接方式、工具/资源的双向映射，以及它如何被挂进 `registry.get_definitions()` 的 `mcp_` 前缀分组里。

---

*本篇基于你本地 `D:\github_projects\nanobot` 的真实源码（读取于 2026-08-09 版本）逐函数拆解，所有行号与代码均可在对应文件中核对。*
