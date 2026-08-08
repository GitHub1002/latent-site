---
title: "nanobot 源码解读 · 07 | Provider 与模型路由：一个模型名如何落到正确的后端"
date: 2026-08-09
draft: false
weight: 807
tags: ["nanobot", "源码解读", "Agent框架", "Provider", "模型路由"]
summary: "逐函数拆解 nanobot 的 Provider 层：registry.py 如何用一张 ProviderSpec 表做唯一真相源、factory 如何把模型名解析到具体后端、OpenAICompatProvider 如何统一适配几十家 OpenAI 兼容服务、FallbackProvider 如何用熔断器+流式续接做模型降级，以及 base.py 里对瞬态错误的自动重试。配真实代码、路由流向图与可运行例子。"
ShowToc: true
---

## 一、为什么需要单独一层 Provider

前面 02 篇讲过 `AgentRunner` 是引擎，它只负责「调模型 → 执行工具」循环，但**「模型」到底是什么、走哪个 SDK、出错怎么办**，引擎本身不关心——这部分被抽到了 `providers/` 包。

这样做的好处：nanobot 要同时支持 OpenAI、Anthropic、Azure、AWS Bedrock、几十个 OpenAI 兼容网关（OpenRouter、硅基流动、火山引擎……）、本地模型（Ollama/vLLM），以及 OAuth 式的 Codex/Grok/Copilot。如果把这些差异都塞进引擎，引擎会变成一锅粥。Provider 层的目标就是：**让引擎看到的是一个统一的 `LLMProvider` 接口，差异全被这一层吞掉。**

本篇就拆开这一层。核心文件：
- `providers/registry.py`（约 757 行）—— 一张 `ProviderSpec` 表，全项目的「Provider 真相源」
- `providers/factory.py` —— 把「模型名 + 配置」解析成具体的 Provider 实例
- `providers/base.py`（约 1191 行）—— `LLMProvider` 抽象基类 + 重试/容错/错误分类
- `providers/openai_compat_provider.py`（约 2000 行）—— 把几十家 OpenAI 兼容服务统一适配
- `providers/fallback_provider.py` —— 熔断器 + 降级路由

---

## 二、registry.py：一张表驱动一切

先记住一个关键设计：**几乎所有 Provider 的元数据都集中在 `registry.py` 的 `PROVIDERS` 元组里**。新增一家厂商，只要在表里加一条 `ProviderSpec`，`nanobot status` 的状态展示、环境变量匹配、`Settings` 兼容分组全都自动派生出来，不用改别处代码（文件头注释原话：*Adding a new provider: 1. Add a ProviderSpec... Done.*）。

`ProviderSpec` 是个 `frozen` 的 dataclass，字段很多，但分组的逻辑很清楚：

```python
@dataclass(frozen=True)
class ProviderSpec:
    # 身份
    name: str                 # 配置字段名，如 "dashscope"
    keywords: tuple[str, ...] # 模型名关键字，用于匹配
    env_key: str              # API key 的环境变量名
    display_name: str = ""

    # 用哪个实现
    backend: str = "openai_compat"   # openai_compat | anthropic | azure_openai | ...

    # 网关 / 本地检测
    is_gateway: bool = False          # 能路由任意模型（OpenRouter 等）
    is_local: bool = False            # 本地部署（Ollama）
    detect_by_key_prefix: str = ""    # 按 api_key 前缀识别，如 "sk-or-"
    detect_by_base_keyword: str = ""  # 按 api_base URL 子串识别
    default_api_base: str = ""        # 该厂商默认的 OpenAI 兼容 base URL

    # 网关行为
    strip_model_prefix: bool = False  # 发送前剥掉 "provider/" 前缀
    supports_prompt_caching: bool = False
    # 思考/reasoning 相关的一堆开关...
    thinking_style: str = ""
    reasoning_effort_remap: tuple[tuple[str, str], ...] = ()
    implicit_reasoning_models: tuple[str, ...] = ()
    strip_history_reasoning_content: bool = False
```

注意几个很有意思的字段，它们直接决定了「同一套代码如何兼容几十家」：

- **`backend`**：只声明「用哪个实现类」。几十家 OpenAI 兼容厂商全是 `openai_compat`，只有 Anthropic 用 `anthropic`、Azure 用 `azure_openai`、Bedrock 用 `bedrock` 等寥寥几种。
- **`keywords` / `detect_by_key_prefix` / `detect_by_base_keyword`**：匹配规则。比如 OpenRouter 键以 `sk-or-` 开头、URL 含 `openrouter`；Ollama 的 `detect_by_base_keyword="11434"`（localhost:11434）。
- **`strip_model_prefix` / `strip_model_prefixes`**：网关经常不认 `anthropic/claude-3` 这种带厂商标识的模型名，发送前自动剥掉。AiHubMix 就设了 `strip_model_prefix=True`。
- **`thinking_style` / `reasoning_effort_remap` / `implicit_reasoning_models`**：思考模式的「方言翻译」。Mistral 只接受 `high`/`none`，于是 `reasoning_effort_remap=(("minimal","none"),("low","none"),("medium","high"),...)`；Magistral 永远在思考、干脆拒绝该参数，于是 `implicit_reasoning_models=("magistral",)`。

`PROVIDERS` 表里还有一个**顺序约定**：注释明确写道 *Order matters — it controls match priority and fallback. Gateways first.*。网关排在最前，因为网关能路由任意模型，在降级时优先级最高（后面会看到）。

---

## 三、factory.py：模型名如何解析到后端

`registry.py` 是「静态元数据」，真正把「用户填的模型名 + 配置」翻译成「用哪家后端」的是 `factory.py` 的 `_resolve_provider_setup`：

```python
def _resolve_provider_setup(config, *, preset, model=None):
    model = model or preset.model
    provider_name = config.get_provider_name(model, preset=preset)  # 配置里查 maps
    p = config.get_provider(model, preset=preset)
    spec = find_by_name(provider_name)                              # 在 PROVIDERS 表查 spec
    if not spec and p:
        if not p.api_base:
            raise ValueError(...)                                   # 没 spec 必须有 api_base
        spec = create_dynamic_spec(provider_name, ...)              # 动态造一个
    backend = spec.backend if spec else "openai_compat"
    # 一堆校验：Azure 必须 api_base、OAuth/本地/直连可免 key 等
    return _ProviderSetup(model=model, provider_name=provider_name,
                          provider_config=p, spec=spec, backend=backend)
```

几个要点：
1. `config.get_provider_name(model)` 先按用户配置里的 `maps`（模型名 → provider 名）找。
2. `find_by_name` 在 `PROVIDERS` 表里按 `name` 找 `ProviderSpec`；找不到但用户给了 `api_base`，就 `create_dynamic_spec` 动态生成一个「openai_compat + 直连」的 spec（支持任意自定义 OpenAI 兼容端点）。
3. 校验阶段区分 `is_oauth`/`is_local`/`is_direct` 等，决定要不要 API key。

再往后 `_make_provider_core` 根据 `backend` 分流到具体实现：

```python
if backend == "openai_codex":   # OAuth 流程
    ...
elif backend == "xai_grok":     # OAuth 流程
    ...
elif backend == "azure_openai":
    ...
elif backend == "anthropic":
    provider = AnthropicProvider(...)
elif backend == "bedrock":
    ...
else:  # openai_compat（绝大多数厂商）
    provider = OpenAICompatProvider(...)
```

也就是说，**「模型名 → 后端实现」这条路由，本质是一张配置 + 一张 `ProviderSpec` 表的联合查找**，代码不过百行，却支撑了几十家厂商。

---

## 四、OpenAICompatProvider：一个类吞掉几十家

这是全项目最重的适配逻辑。它的核心思想是：**所有 OpenAI 兼容服务都用同一个 `AsyncOpenAI` 客户端，差异全靠 `ProviderSpec` + `_build_kwargs` 时的字段裁剪来吸收。**

### 4.1 客户端懒加载

```python
def __init__(self, api_key, api_base, default_model="gpt-4o", spec=None, ...):
    ...
    # 懒加载：OpenAI client + httpx transport 创建很贵（Windows 上约 700ms）
    self._client = None
    self._client_lock = asyncio.Lock()

def _build_client(self):
    import httpx
    # 本地模型（Ollama/vLLM）容易在两次调用间让 keepalive 连接失效，
    # 于是关掉 keepalive、不走代理，每次请求开新连接（LAN 上很便宜）
    if self._is_local:
        _local_limits = httpx.Limits(keepalive_expiry=0)
        http_client = httpx.AsyncClient(limits=_local_limits, transport=httpx.AsyncHTTPTransport(proxy=None, ...))
    self._client = AsyncOpenAI(api_key=..., base_url=self._effective_base,
                               max_retries=0, timeout=timeout_s, http_client=http_client)
```

注意 `max_retries=0`：nanobot **不在 SDK 层重试**，重试由 `base.py` 自己做（见第六节），这样可以统一控制「是否瞬态、退避多久、流式如何续接」。

### 4.2 请求拼装：`_build_kwargs` 是「方言翻译器」

这是适配层最精华的函数（约 781 行起，近 180 行）。它把引擎传来的「messages / tools / model / max_tokens / temperature / reasoning_effort」翻译成某家厂商真正接受的参数：

```python
def _build_kwargs(self, messages, tools, model, max_tokens, temperature, reasoning_effort, tool_choice):
    model_name = model or self.default_model
    spec = self._spec

    # 1) prompt caching：Anthropic/Claude 打 cache_control
    if spec and spec.supports_prompt_caching:
        if model_name.lower().startswith(("anthropic/", "claude")):
            messages, tools = self._apply_cache_control(messages, tools)

    model_name = self._request_model_name(model_name)   # 网关剥前缀

    kwargs = {"model": model_name,
              "messages": self._sanitize_messages(self._sanitize_empty_content(messages))}

    # 2) GPT-5 / o 系列 reasoning 激活时拒绝 temperature，按需省略
    if self._supports_temperature(model_name, reasoning_effort):
        kwargs["temperature"] = temperature

    # 3) 有的厂商要 max_completion_tokens，有的要 max_tokens
    if spec and spec.supports_max_completion_tokens or _requires_max_completion_tokens(model_name):
        kwargs["max_completion_tokens"] = max(1, max_tokens)
    else:
        kwargs["max_tokens"] = max(1, max_tokens)

    # 4) 每模型参数覆盖（如 kimi-k2.7 强制 temperature=1.0）
    if spec:
        for pattern, overrides in spec.model_overrides:
            if pattern in model_name.lower():
                kwargs.update(overrides); break

    # 5) reasoning_effort 方言翻译（minimal→minimum、Mistral→high/none、
    #    Magistral 隐式思考直接 strip 该参数、Kimi 去掉冗余 reasoning_effort...）
    ...
    if wire_effort and semantic_effort != "none":
        kwargs["reasoning_effort"] = wire_effort

    # 6) 思考开关注入 extra_body（火山/通义/阶跃各自的 thinking 形状）
    for thinking_style in _thinking_styles_for(spec, model_name):
        extra = _thinking_extra_body(thinking_style, thinking_enabled)
        if extra: kwargs.setdefault("extra_body", {}).update(extra)

    # 7) 工具
    if tools:
        kwargs["tools"] = tools
        kwargs["tool_choice"] = tool_choice or "auto"

    # 8) DeepSeek 思考模式历史必须补 reasoning_content=""，否则 400
    if explicit_thinking or implicit_deepseek_thinking:
        for msg in kwargs["messages"]:
            if msg.get("role") == "assistant" and "reasoning_content" not in msg:
                msg["reasoning_content"] = ""
    return kwargs
```

把这段读下来，你就明白为什么 nanobot 能用一份代码支持几十家：**差异不是写在 if/else 分支里硬编码，而是声明在 `ProviderSpec` 表里，`_build_kwargs` 统一消费这些声明**。加一家新厂商，绝大多数情况只改表、不改适配函数。

---

## 五、FallbackProvider：模型降级与熔断器

`FallbackProvider` 是另一个 `LLMProvider`，它**包了一个主 Provider + 一串降级预设**。引擎不需要知道自己在用降级，照样调 `chat`/`chat_stream`，降级对引擎完全透明——和 06 篇 MCP 包装器「对引擎透明」是同一个哲学。

### 5.1 入口：`_try_with_fallback`

```python
async def _try_with_fallback(self, call, kwargs, has_streamed, on_stream_recover=None):
    primary_model = kwargs.get("model") or self._primary.get_default_model()
    if self._primary_available():
        response = await call(self._primary, kwargs)
        if response.finish_reason != "error":
            self._primary_failures = 0
            return response                      # 主模型成功，直接返回
        # 主模型报错了，判断是否「可降级」
        if not self._should_fallback(response):
            return response                      # 不可降级（如鉴权失败）→ 直接返回错误
        self._primary_failures += 1
        if self._primary_failures >= _PRIMARY_FAILURE_THRESHOLD:
            self._primary_tripped_at = time.monotonic()   # 连续失败太多 → 熔断器打开
    # 依次试每个 fallback 预设
    for fallback in self._fallback_presets:
        fb_provider = self._provider_factory(fallback)
        await self._notify_fallback_model(fallback.model)
        fb_kwargs = {**kwargs, "model": fallback.model,
                     "max_tokens": fallback.max_tokens, "temperature": fallback.temperature}
        # 切换模型时还要处理对话状态能否跨 Provider 续接、是否需要原生压缩
        ...
        fb_response = await call(fb_provider, fb_kwargs)
        if fb_response.finish_reason != "error":
            return fb_response                  # 某个降级成功即返回
    # 全失败，返回最后一个错误
```

### 5.2 熔断器 + 流式续接

两个细节很值得讲：

**① 熔断器（circuit breaker）**：`self._primary_failures` 连续累计，达到阈值 `_PRIMARY_FAILURE_THRESHOLD` 就记 `tripped_at` 时间戳，之后一段时间 `_primary_available()` 直接返回 `False`——「主模型挂了就别再浪费它了，直接去试降级」。这避免了每次请求都先锤一遍已经挂掉的主模型。

**② 流式续接（streaming continuity）**：流式输出最麻烦的是「主模型已经吐了一段字，突然超时」。这时候能不能降级？fallback_provider 用 `has_streamed[0]` 标记「是否已经吐过内容」：
- 如果**超时**且 `on_stream_recover` 存在：把 `has_streamed` 复位、调用 `on_stream_recover()`（让前端把已显示的旧片段清掉或接上新片段），然后继续降级——用户看到的是「换了个模型接着答」。
- 如果是**非超时错误**（比如内容已经输出完才报错）：直接返回原响应，不降级（因为用户已经看到答案了，没必要重来）。

`_should_fallback` 则决定「这个错误值不值得降级」——欠费（`is_arrearage_response`）、鉴权错误、速率限制等都返回 `True`；而某些明确不可重试的错误就 `False`，直接把错误透传给用户。

---

## 六、base.py：错误分类与自动重试

所有 Provider 都继承自 `LLMProvider`（`base.py`），它统一提供了重试骨架 `_run_with_retry` 和错误分类工具：

```python
async def _run_with_retry(self, call, kw, original_messages, *, retry_mode, on_retry_wait, ...):
    delays = list(self._CHAT_RETRY_DELAYS)
    persistent = retry_mode == "persistent"
    while True:
        attempt += 1
        response = await call(**kw)
        if response.finish_reason != "error":
            return response
        # 流式已吐字的情况：超时则续接重试，否则直接返回
        ...
        if not self.is_transient_response(response):
            # 非瞬态错误：若消息含图片，尝试「去掉图片重试一次」；否则直接返回
            ...
            return response
        # 瞬态错误：按 Retry-After 头或退避表退避后重试
        retry_after = self._extract_retry_after_from_response(response)
        delay = retry_after + BUFFER if retry_after else delays[min(attempt-1, ...)]
        await self._sleep_with_heartbeat(delay, ...)
    return last_response
```

关键分类方法：
- `is_transient_response`：429 限流、5xx、超时、连接错误 → 可重试。
- `_is_retryable_429_response`：429 是否值得退避重试。
- `is_arrearage_response`：欠费 → 这种错误会**触发 fallback**（见第五节）。
- 还有一个很贴心的兜底：**非瞬态错误但消息里带图片** → nanobot 会自动「去掉图片重试一次」。因为很多模型对图片输入的报错信息含糊，去掉图往往就能过。

> 呼应前面：`max_retries=0`（SDK 不重试）+ `base.py` 统一重试，让「是否重试、退避多久、流式怎么续、图片怎么剥离」全在一个地方可控，而不是散落各家 SDK 默认行为里。

---

## 七、一次请求到底走了哪些 Provider 代码

把 02 篇的 `_request_model` 和本篇串起来，一条消息从引擎到模型的路径是：

![nanobot 07 模型路由与降级流向](/images/nanobot-07-provider.svg)

文字版：
1. `AgentRunner._run_core` 调 `_request_model` → `provider.chat_stream(messages=..., model=..., tools=..., reasoning_effort=...)`。
2. 如果该 Provider 是 `FallbackProvider`（`factory` 把它包了一层）：进 `_try_with_fallback` → 先打主模型。
3. 主模型 `OpenAICompatProvider.chat_stream` → `_build_kwargs`（按 `ProviderSpec` 裁剪参数）→ `_ensure_client`（懒建 `AsyncOpenAI`）→ `client.chat.completions.create(...)`。
4. 响应经 `_parse_chunks` 转成统一的 `LLMResponse`（含 `finish_reason`/`error_kind`/`tool_calls`/`usage`）。
5. 若步骤 2 主模型返回 `error` 且 `_should_fallback` 为真 → 熔断器计数 + 依次试 `fallback_presets`，每试一个都重新 `_build_kwargs`（换模型名 + 降级专属 max_tokens/temperature）。
6. 任何一层遇到瞬态错误 → `base.py` 的 `_run_with_retry` 按退避/Retry-After 自动重试；流式已吐字则按第五节规则决定续接或放弃。

**一句话总结**：引擎只认 `LLMProvider.chat_stream` 这一个接口；模型名通过 `factory` + `ProviderSpec` 表路由到具体后端；几十家 OpenAI 兼容服务被 `OpenAICompatProvider` 用「声明式 spec + 统一 kwargs 裁剪」吞掉；主模型挂了由 `FallbackProvider` 透明降级；瞬态错误由 `base.py` 统一重试。四层各管一摊，互不越界。

---

## 八、小结

本篇要点：
- `registry.py` 的 `PROVIDERS` 表是**唯一真相源**，新增厂商大多只改表。
- `factory.py` 用「配置 maps + spec 表」把模型名路由到具体 `backend` 实现。
- `OpenAICompatProvider` 用**声明式 spec 驱动 `_build_kwargs`**，一份代码适配几十家。
- `FallbackProvider` 实现**透明降级 + 熔断器 + 流式续接**，引擎无感。
- `base.py` 统一**错误分类 + 自动重试**，并把「去掉图片重试」这种兜底收口到一处。

下一篇 **08 · 多 Agent 委派** 我们看 `agent/subagent.py`：一个 Agent 如何把子任务派给子 Agent、结果怎么回汇——这正好建立在本篇的 `LLMProvider` 和 02 篇的 `AgentRunner` 之上。
