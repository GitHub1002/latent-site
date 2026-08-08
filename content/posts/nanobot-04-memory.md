---
title: "nanobot 源码解读 · 04 | 记忆与状态：Dream 长期记忆与会话压缩"
date: 2026-08-08
draft: false
weight: 804
tags: ["nanobot", "源码解读", "记忆系统", "Agent框架"]
summary: "逐函数拆解 nanobot 的记忆与状态层：MemoryStore 如何用纯文件 I/O 管理 MEMORY.md/history.jsonl；Dream 引擎如何把会话历史沉淀进长期记忆并用 GitStore 做审计；以及 DeepWiki 完全漏掉的会话压缩层 Consolidator 与 AutoCompact——它们如何把超长会话压成 [Archived Context Summary] 注入系统提示。配真实代码、消息流向图与例子。"
ShowToc: true
---

## 一、这一篇解决什么：记忆其实分两层

前两篇我们讲完了「一个请求怎么走完」（02 核心引擎）和「一条消息怎么被拼成剧本」（03 上下文工程）。但还有一个问题没回答：**Agent 怎么"记住"东西？**

这里要分清 nanobot 里两类完全不同的"记忆"：

| 维度 | 长期记忆（跨会话） | 会话状态（单会话内） |
|---|---|---|
| 载体 | `MEMORY.md` / `SOUL.md` / `USER.md` 文件 | `Session` 对象 + `history.jsonl` |
| 生命周期 | 永久，跨重启、跨对话 | 仅当前会话，超长会被压缩 |
| 写入者 | **Dream** 整合引擎（定期跑） | 每轮 `append_history` + `Consolidator` 压缩 |
| 读法 | 拼进 system 提示（03 讲过的"长期记忆块"） | 拼进回放历史（03 讲过的"最近 50 条"） |
| 审计 | GitStore（每次变更 git commit） | 仅追加 jsonl，不进 git |

> **对照 DeepWiki 的坑**：上一轮我们核对过，DeepWiki 那篇总结只讲了"会话历史 + Dream 长期记忆压缩"，**完全漏掉了"会话压缩层"（Consolidator / AutoCompact）**。而这一层恰恰是每个长对话都不爆 context window 的关键——没有它，聊到第 200 轮就得从头截断。所以这篇要补的就是这半截。

本篇基于 `nanobot/agent/memory.py`（1221 行）、`nanobot/agent/autocompact.py`、`nanobot/session/manager.py` 与 `nanobot/command/builtin.py` 的真实代码。

---

## 二、MemoryStore：纯文件 I/O 的记忆仓库

`MemoryStore`（`memory.py:73`）只干一件事：**把记忆落盘成普通文件 + 一条追加式历史日志**。它刻意不做任何 LLM 调用——这是"纯 I/O 层"，压缩/整合是别处的职责。

### 2.1 文件布局

构造函数（`memory.py:93`）把工作区映射成一组固定文件：

```python
self.memory_dir = ensure_dir(workspace / "memory")
self.memory_file   = self.memory_dir / "MEMORY.md"      # 项目/事实记忆
self.history_file  = self.memory_dir / "history.jsonl"  # 追加式历史（Dream 原料）
self.soul_file     = workspace / "SOUL.md"              # Agent 行为规则
self.user_file     = workspace / "USER.md"              # 用户画像
self._cursor_file  = self.memory_dir / ".cursor"        # 历史自增游标
self._dream_cursor_file = self.memory_dir / ".dream_cursor"  # Dream 消费到哪了
```

四个记忆文件里，`SOUL.md`/`USER.md`/`MEMORY.md` 被注册进 `GitStore`（`memory.py:109`）——**这正是 DeepWiki 说得对的地方**：nanobot 用 git 当记忆的版本管理与审计后端，每次 Dream 改写都会 `git commit`，你能 `git diff` 出"上周它把我记成了什么"。

### 2.2 append_history：历史是"带游标的追加日志"

每条用户/Agent 交互都会写一行 jsonl。注意它的三个防御性设计（`memory.py:280`）：

```python
def append_history(self, entry, *, max_chars=None, session_key=None) -> int:
    limit = max_chars if max_chars is not None else _HISTORY_ENTRY_HARD_CAP
    ts = datetime.now().strftime("%Y-%m-%d %H:%M")
    raw = entry.rstrip()
    if len(raw) > limit:
        raw = truncate_text(raw, limit)        # 防御：单条上限 64K，防 LLM 把输入当摘要回灌
    content = strip_think(raw)                  # 剥离 <think> 泄漏 / <channel|> 标记
    with self._append_lock:                    # 线程锁：游标分配 + 追加必须原子
        cursor = self._next_cursor()
        record = {"cursor": cursor, "timestamp": ts, "content": content}
        if session_key:
            record["session_key"] = session_key
        with open(self.history_file, "a", encoding="utf-8") as f:
            f.write(json.dumps(record, ensure_ascii=False) + "\n")
        self._cursor_file.write_text(str(cursor), encoding="utf-8")
    return cursor
```

三个值得记住的点：

1. **游标自增 + 文件锁**：`strip_think` 之后若内容变空但原文非空，它**仍然持久化空串**——注释解释得很直白："否则 `strip_think` 的保证会被下游历史回放/整合给推翻"。记忆系统的信条是"宁可记空，不可记脏"。
2. **`session_key` 维度**：历史不是扁平的，按 `channel:chat_id` 分桶。DeepWiki 没提到的 `cron:`/`dream:` 前缀（`:85`）属于"内部会话"，不会被混进普通对话历史。
3. **`compact_history`**（`:442`）会丢弃最旧的已处理条目，但**永远保留 `last_dream_cursor` 之后的未消费条目**——因为那些是 Dream 还没读过的原料，丢了就再也沉淀不进长期记忆了。

### 2.3 读取：按桶过滤的"最近 N 条"

03 篇里我们看到 system 提示会塞"最近 50 条历史（限 8000 token）"，源头就在这里（`memory.py:421`）：

```python
def read_recent_history_for_prompt(self, since_cursor, *, session_key, unified_session=False):
    entries = self.read_unprocessed_history(since_cursor=since_cursor)
    if session_key is None:
        return entries
    if not unified_session:
        return [e for e in entries if e.get("session_key") == session_key]  # 只取本会话
    # unified_session：本会话 + 非内部会话（跨会话统一视图）
    return [e for e in entries
            if (s := e.get("session_key")) == session_key
            or not self._is_internal_history_session(s)]
```

---

## 三、Dream：把会话历史"沉淀"进长期记忆

`history.jsonl` 是原料堆，`Dream` 是加工厂。它**不直接参与对话**，而是在你发 `/dream` 或定时触发时，默默把原料提炼成 `SOUL.md`/`USER.md`/`MEMORY.md`。

### 3.1 触发与输入

`/dream` 命令的入口在 `command/builtin.py:424` 的 `_run_dream`。它先调 `store.build_dream_prompt()`（`memory.py:582`）拼出一条给 LLM 的"记忆整合"提示：

```python
def build_dream_prompt(self, *, max_entries: int = 20) -> tuple[str, int] | None:
    last_cursor = self.get_last_dream_cursor()           # 上次读到哪
    entries = self.read_unprocessed_history(since_cursor=last_cursor)
    if not entries:
        return None                                       # 没新东西就不跑
    batch = entries[:max_entries]                         # 每批最多 20 条
    history_text = "\n".join(
        f"[{e['timestamp']}] {truncate_text(e['content'], 1000)}" for e in batch)
    template = self._dream_template()                     # agent/dream.md
    files_section = self._render_current_memory_files()   # 当前三个记忆文件真实内容
    prompt = f"{template}\n\n{files_section}\n\n## Conversation History\n{history_text}"
    return (prompt, batch[-1]["cursor"])
```

关键设计：**当前记忆文件内容会被原样塞进提示**，模型是"对着真实文件改"，而不是对着自己脑补的旧版本改。代码注释点明这是为了消除一类"模型生成了幻觉审计记录"的 bug。

### 3.2 Dream 的"工具"是被阉割过的

Dream 不是用主 Agent 的全部工具，而是 `build_dream_tools()`（`memory.py:642`）构造一个**受限 Registry**：只能用 `ReadFile`/`EditFile`/`ApplyPatch`/`WriteFile`，且白名单只允许写三个记忆文件 + 读 skills 目录：

```python
editable_files = [self.memory_file, self.soul_file, self.user_file]
tools.register(EditFileTool(workspace=workspace, allowed_dir=skills_dir,
                            extra_write_allowed_files=editable_files, ...))
tools.register(ApplyPatchTool(workspace=workspace, allowed_dir=skills_dir,
                              extra_write_allowed_files=editable_files, ...))
```

这把"整合记忆"圈死在三个文件里——它动不了你的代码、动不了别的配置。

### 3.3 提示词：一份相当硬核的"记忆宪章"

`templates/agent/dream.md` 开头就定调，值得引述：

> You are a memory consolidation engine. Your sole task is to analyze conversation history and maintain the user's long-term memory files... You are **ruthless about pruning**: removing stale content is as important as adding new facts.

它甚至给了一张文件路由表（避免模型把"用户偏好中文"记错地方）：

| 文件 | 放什么 |
|---|---|
| `SOUL.md` | Agent 行为规则、护栏、交互模式、工具策略 |
| `USER.md` | 用户身份/偏好/习惯/沟通风格（语言、长度、语气） |
| `MEMORY.md` | 项目背景：目标、架构、战略决策、基础设施 |
| `SKILL.md` | 可复用工作流模板（具体步骤、命令、例子） |

还强制 **MECE**（互不重叠）：用户事实不准进 `SOUL.md`，技术配置不准进 `USER.md`，操作细节（命令/flag/token/URL）不准进 `MEMORY.md`——这些要去 `SKILL.md`。

### 3.4 完成判定 + 审计提交 + 游标推进

回到 `_run_dream`（`builtin.py:449`）的收尾逻辑：

```python
resp = await loop.process_direct(prompt, session_key=key, ephemeral=True,
                                 tools=store.build_dream_tools(),
                                 on_progress=progress, runtime=dream_runtime)
diff_body = store.dream_content_diff()          # git 工作区真实 diff
completed = MemoryStore.dream_run_completed(resp, had_tool_errors=progress.had_tool_errors)
if completed:
    store.set_last_dream_cursor(last_cursor)    # 只有成功才推进游标
    ...
# finally:
if store.git.is_initialized():
    commit_msg = build_dream_commit_message("dream: manual run", diff_body)
    sha = store.git.auto_commit(commit_msg)     # 用真实 diff 写审计，而非模型自述
store.compact_history()
```

注意 `dream_run_completed`（`memory.py:686`）的判定：**必须没有工具调用报错、且 `_stop_reason == "completed"`** 才算成功。失败时游标不推进，这批历史下次 Dream 会重跑——宁可重复读，不可丢原料。

> **DeepWiki 此处的可取之处**：它提到"Dream uses GitStore for long-term memory compression"——这个判断是对的，git 后端确实是 Dream 的真相来源与回滚点。但它把 Dream 描述成"压缩历史"，略窄了：Dream 其实是"**提炼 + 去重 + 路由 + 裁剪**"，git 只是它的提交后端。

---

## 四、会话压缩层：DeepWiki 完全漏掉的核心

这是本篇最该补的半截。**长期记忆（Dream）跨会话，会话状态（压缩）在会话内。** 没有后者，单会话聊太长就会爆 context window。

### 4.1 数据模型：一个游标划开"在内存 vs 已固化"

`Session`（`session/manager.py:150`）有个关键字段：

```python
@dataclass
class Session:
    key: str
    messages: list[dict] = field(default_factory=list)
    metadata: dict = field(default_factory=dict)
    last_consolidated: int = 0   # 已固化进文件（压缩掉）的消息条数
    provider_state = None
```

`messages[:last_consolidated]` 是**已经被摘要代替、不再原样回放**的部分；`messages[last_consolidated:]` 是**仍原样喂给模型**的"活跃尾部"。压缩就是把左边越推越长、右边保持在预算内的过程。

### 4.2 Consolidator.archive：把一段对话压成摘要

`archive`（`memory.py:993`）是压缩的最小单元——把一批消息交给一个**独立的"压缩模型调用"**生成摘要，再写回 `history.jsonl`：

```python
async def archive(self, messages, *, runtime, session_key=None, summary_messages=None):
    if not messages:
        return None
    messages_to_summarize = public_history_messages(summary_messages or messages)
    formatted = MemoryStore._format_messages(messages_to_summarize)
    formatted = self._truncate_to_token_budget(formatted, runtime=runtime)
    system_prompt = render_template("agent/consolidator_archive.md", strip=True)
    try:
        response = await runtime.provider.chat_with_retry(
            model=runtime.model,
            messages=[{"role": "system", "content": system_prompt},
                      {"role": "user", "content": formatted}],
            tools=None, ...)
    except Exception:
        self.store.raw_archive(messages, session_key=session_key)   # 失败降级：原样落盘
        return None
    summary = response.content or "[no summary]"
    self.store.append_history(summary, max_chars=_ARCHIVE_SUMMARY_MAX_CHARS,
                             session_key=session_key)
    return summary
```

这张模板（`templates/agent/consolidator_archive.md`）我读过，它的指令是"**提取关键事实并标注记忆属性**"，输出带 `[permanent]`/`[durable]`/`[ephemeral]`/`[skip]` 之类标签——这些标签正是下一节 Dream 读取时的路由提示（呼应 3.3 的 History attribute tags）。

**失败降级是重点**：`chat_with_retry` 抛异常或返回 `error`，不是让整个回合崩，而是 `raw_archive`（`memory.py:722`）把原始对话原样塞进历史——"宁可不压缩，也绝不丢历史"。

### 4.3 maybe_consolidate_by_tokens：按 token 预算循环压

`maybe_consolidate_by_tokens`（`memory.py:1048`）在每轮 build 前被调用，逻辑是"**压到 safe budget 一半以下为止**"，最多 5 轮：

```python
budget = self._input_token_budget(runtime)          # 上下文窗口 - 生成预算 - 1024 安全余量
target = int(budget * self.consolidation_ratio)     # 默认 0.5
# ... 先处理 replay 窗口溢出 ...
for round_num in range(self._MAX_CONSOLIDATION_ROUNDS):   # 最多 5 轮
    if estimated <= target:
        break
    boundary = self.pick_consolidation_boundary(session, max(1, estimated - target))
    if boundary is None:
        break
    chunk = session.messages[session.last_consolidated:end_idx]
    summary = await self.archive(chunk, runtime=runtime, session_key=session.key)
    session.last_consolidated = end_idx              # 推进游标
    session.provider_state = None                     # 压缩后旧 provider 状态失效
    self.sessions.save(session)
    if not summary:
        break                                         # LLM 降级，停止硬压
    estimated, source = self.estimate_session_prompt_tokens(session, runtime=runtime)
```

`pick_consolidation_boundary`（`memory.py:838`）保证**只切在 `user` 消息边界**上：

```python
for idx in range(start, len(session.messages)):
    message = session.messages[idx]
    if idx > start and message.get("role") == "user":   # 只在用户发言处切
        last_boundary = (idx, removed_tokens)
        if removed_tokens >= tokens_to_remove:
            return last_boundary
    removed_tokens += estimate_message_tokens(message)
```

为什么要切在 user 边界？因为半截对话（比如卡在 assistant 中途）压缩后会丢失上下文衔接，模型会莫名其妙。切在 user 发言前，等于"把上一个完整回合整体归档"。

### 4.4 AutoCompact：空闲会话主动压缩

`Consolidator` 是被动的（build 前按需压），`AutoCompact`（`autocompact.py:18`）是主动的——**后台定时扫描超 TTL 的空闲会话**，提前压好，下次用户回来时已经轻量：

```python
def check_expired(self, schedule_background, resolve_runtime, active_session_keys=()):
    for info in self.sessions.list_sessions():
        key = info.get("key", "")
        if not key or self._is_internal_session(key) or key in self._archiving:
            continue
        if key in active_session_keys:
            continue                                      # 正在聊的不压
        if self._is_expired(info.get("updated_at"), now) \
           and self._has_compactable_idle_tail(key):
            runtime = resolve_runtime(session)
            self._archiving.add(key)
            schedule_background(self._archive(key, runtime=runtime))   # 后台跑
```

压完的摘要存在一个很妙的地方——`session.metadata["_last_summary"]`。等用户下次回来，`prepare_session`（`autocompact.py:124`）会把这个摘要格式化后**注入系统提示**：

```python
meta = session.metadata.get("_last_summary")
if isinstance(meta, dict):
    text = meta.get("text")
    ...
    return session, self._format_summary(text, last_active)
# _format_summary: "Previous conversation summary (last active ...):\n{text}"
```

### 4.5 接线：压缩摘要怎么进系统提示

还记得 03 篇讲 `build_system_prompt` 时提到的"压缩摘要"块吗？源头全在这：

1. `loop._compact_session`（`loop.py:1888`）→ 调 `auto_compact.prepare_session` → 拿到 `pending_summary` 存进 `ctx.pending_summary`；
2. build 阶段（`loop.py:864`）把 `session_summary=ctx.pending_summary` 传给 `build_messages`；
3. `context.py:124` 把它渲染成系统提示里的一段：

```python
if session_summary:
    parts.append(f"[Archived Context Summary]\n\n{session_summary}")
```

**这就是 `[Archived Context Summary]` 标签的真正来历**——它来自 `AutoCompact`/`Consolidator` 的压缩产物，不是 Dream。DeepWiki 把"会话压缩"和"Dream 长期记忆"混为一谈，正是因为它没读过这一层。

---

## 五、完整的消息流向（串起来看）

![nanobot 记忆与状态双系统](/images/nanobot-04-memory.svg)

一句话概括流向：**每条对话实时写 `history.jsonl`；会话内超长由 `Consolidator`/`AutoCompact` 压成 `[Archived Context Summary]` 注入系统提示（保住上下文窗口）；`history.jsonl` 里没被 Dream 消费的部分，周期性由 `Dream` 提炼进 `SOUL/USER/MEMORY.md` 并 git 提交（跨会话记忆）。**

---

## 六、举三个具体例子

**例 1：聊了 300 轮，模型怎么还记得开头？**
你和一个会话连聊 300 轮。`maybe_consolidate_by_tokens` 在每轮 build 前估算 token，发现超预算就把前若干"完整回合"（切在 user 边界）交给压缩模型生成摘要、写进 `history.jsonl`，并推进 `last_consolidated`。最终模型看到的回放窗口可能只剩最近 8 轮原文 + 前面一大段 `[Archived Context Summary]`。开头那段"约定"，就活在摘要里。

**例 2：三天前你说"我偏好中文回复"，今天新会话一上来就懂？**
`/dream` 跑过之后，这条偏好被 Dream 路由进了 `USER.md`（按 3.3 的路由表，"语言偏好属于沟通风格"）。新会话 build 时，`build_system_prompt` 把 `USER.md` 内容拼进 system 提示——所以模型**还没听你说话就已知你爱中文**。这叫跨会话记忆，靠的是文件而非任何数据库。

**例 3：Dream 把文件改坏了怎么办？**
因为每次 Dream 改写都 `GitStore.auto_commit` 真实 diff，你随时可以 `cd ~/.nanobot/workspace && git log -- memory/MEMORY.md` 看到变更，`git revert` 回去即可。若那次 Dream 调工具报错（`had_tool_errors=True`），`dream_run_completed` 返回 False，**游标不推进**，这段历史下次还会重跑——不会因一次失败而永久丢失。

---

## 七、对照 DeepWiki：它说得对的和漏掉的

结合上一轮我们做的核对，把记忆这块再收个尾：

| 维度 | DeepWiki 总结 | 真实源码（本篇） | 评价 |
|---|---|---|---|
| 长期记忆后端 | "Dream uses GitStore" | `GitStore` 注册 `SOUL/USER/MEMORY`，每次 `auto_commit` | ✅ 对 |
| 记忆文件 | `MEMORY.md` + workspace | `SOUL.md`/`USER.md`/`MEMORY.md` 三件套 + `history.jsonl` | ⚠️ 偏简，漏了 SOUL/USER 与 history |
| 会话压缩层 | 未提及 | `Consolidator` + `AutoCompact`，`[Archived Context Summary]` 注入 | ❌ 完全漏掉 |
| "压缩"的含义 | 当作 Dream 的同义词 | 会话内压缩 ≠ 跨会话 Dream，两者机制、触发、落点都不同 | ❌ 概念混淆 |
| 引擎归属 | "AgentLoop 协调 LLM" | `AgentRunner` 才是模型循环（02 已纠） | ❌ 误述 |

**结论**：DeepWiki 给的是一张合格的"能力地图"——你能快速知道 nanobot"能记什么、存在哪"。但要理解"记忆怎么流动、怎么不丢、怎么跨会话"，必须读引擎源码，也就是本篇和 02/03 补上的部分。

---

## 八、小结与预告

本篇我们读透了 nanobot 的记忆与状态双系统：

- **MemoryStore**：纯文件 I/O，四个记忆文件 + 一条带游标的 `history.jsonl`，所有写入都带防御（锁、strip、丢弃策略）。
- **Dream**：只读历史的"未消费尾"，用受限工具改写三个记忆文件，git 提交做审计，游标成功才推进——跨会话记忆的真相来源。
- **Consolidator + AutoCompact**：会话内压缩层，按 token 预算循环压、切在 user 边界、失败降级原样落盘，摘要以 `[Archived Context Summary]` 注入系统提示——DeepWiki 漏掉的核心。

下一篇 **05 · 工具系统与安全边界**，我们进 `agent/tools/*` 与 `security/*`：工具是怎么被发现和注册的、shell/web/filesystem 三个内置工具怎么实现、以及 runner 在每次工具执行时如何施加 SSRF 硬边界与 workspace 软边界（呼应 02 讲过的"权限红线在 AgentLoop、安全边界在 AgentRunner"）。

---

*本篇所有行号基于你本地 `D:\github_projects\nanobot` 的 `agent/memory.py`、`agent/autocompact.py`、`session/manager.py`、`command/builtin.py`、`templates/agent/dream.md`。本系列只解读 nanobot 本身，不与其它项目联动。*
