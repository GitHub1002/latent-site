---
title: "系列二 · 04 | 阶段四·聪明安全：友好降级 + 示意流程图"
date: 2026-07-18
draft: false
weight: 640
tags:
  - 大模型
  - Agent
  - 企业知识助手
  - 权限安全
  - RAG
  - 人机协作
summary: 给 Agent 装上「聪明且安全」的两道防线：RAG/数据库都没依据时不编造、改成友好告知；流程类问题自动附一张 Mermaid 流程图。
---
## 1. 目标 & 进度

走到 M3，Agent 已经能「读懂私有文件（M1）」「查本人假期（M2）」「且张三查不到李四（M3）」。但还有两个让人不踏实的场景：

- **答不出时，它会硬编。** 知识库和数据库都没命中，模型为了「不冷场」会凭印象编一段公司制度/数字——这是企业场景最不能接受的事。
- **能答流程时，只有一大段文字。** 「年假怎么请」这种问题，纯文字远不如一张流程图直观。

所以 **M4 = 聪明安全阶段**，目标就两条：

1. **友好降级**：公司内问题但内部无依据时，丢掉可能编造的回答，换成固定的「没找到 + 建议联系 HR/IT」话术。
2. **示意流程图**：回答涉及流程/步骤时，自动附一张 ` ```mermaid ` 代码块（渲染交给 M5 的 Web 界面）。

里程碑进度更新：

```
M0 最小闭环      ✅  FastAPI + 单轮问答 + 身份注入骨架
M1 读懂私有文件  ✅  RAG 检索工具
M2 查数据        ✅  本人假期查询
M3 权限红线      ✅  服务端注入身份，工具只能查本人
M4 聪明安全      ✅  友好降级 + Mermaid 流程图   ← 本篇
M5 交给同事      ⏳  Web 界面 / 真实登录鉴权
```

---

## 2. 设计决策

**降级不能只靠提示词。** 在 M1 的 `SYSTEM_PROMPT` 里早就有「不要编造」的软约束，但提示词是「请求」不是「开关」——模型一旦自信地以为自己知道，照样会编。要真正卡住，得有一个**结构性的安全网**：让工具在「确实返回了内部数据」时打一个标记，编排器在收尾时读这个标记；如果「用户问了公司内问题、但内部毫无依据」，就**强制**丢弃模型的回答、换成固定话术。

**「公司问题」和「闲聊」要分得清。** 关键信号是：`search_knowledge_base` 到底有没有被调用过。

- 模型真的去查了知识库（说明它判定这是公司内问题）→ `rag_called = True`；
- 但没检索到任何片段 → `rag_found = False`；
- 同时假期工具也没拿到本人数据 → `leave_found = False`。

三者同时满足，才判定为「公司内问题但无依据」，触发降级。**纯闲聊根本不会调用检索工具**，所以不会误伤。

**Mermaid 用提示词驱动生成即可。** 渲染是 M5 的事（前端引 `mermaid.js` 把 ` ```mermaid ` 块画成图）。M4 只负责「让模型在该画图时画图」，用一段示例 prompt 教会它 `flowchart TD` 语法。

**权限红线不被破坏。** 新加的「接地标记」和 M3 的 `user_id_ctx` 是同一套思路——都是请求级 `ContextVar`，**模型永远无法篡改取值**。M4 只是在红线旁边又加了一道「有没有依据」的闸。

---

## 3. 关键实现

### 3.1 新增接地追踪模块 `app/agent/grounding.py`

四个布尔 `ContextVar`，工具在返回真实数据时置 `True`：

```python
import contextvars

rag_called_ctx: contextvars.ContextVar[bool] = contextvars.ContextVar("rag_called", default=False)
rag_found_ctx:  contextvars.ContextVar[bool] = contextvars.ContextVar("rag_found",  default=False)
leave_called_ctx: contextvars.ContextVar[bool] = contextvars.ContextVar("leave_called", default=False)
leave_found_ctx:  contextvars.ContextVar[bool] = contextvars.ContextVar("leave_found",  default=False)
```

### 3.2 两个工具打标记

`rag_tool.py`：被调用即置 `rag_called`，有没有命中置 `rag_found`：

```python
rag_called_ctx.set(True)
docs = rag_search(query, k=4)
if not docs:
    rag_found_ctx.set(False)
    return EMPTY_SENTINEL          # "知识库中没有找到相关内容。"
rag_found_ctx.set(True)
# ...返回片段
```

`leave_tool.py`：同理，真正拿到本人数据才置 `leave_found`：

```python
leave_called_ctx.set(True)
user_id = user_id_ctx.get()
data = fetch_leave_balance(user_id)
if data is None:
    return f"未找到用户 {user_id} 的假期记录，请联系 HR 核实。"
leave_found_ctx.set(True)         # 真正接地
return ...
```

### 3.3 编排器加安全网 `apply_graceful`

`orchestrator.py` 里新增固定话术与判定函数：

```python
GRACEFUL_NO_ANSWER = (
    "抱歉，我在公司内部知识库和数据库中都没有找到与这个问题相关的信息，"
    "无法为你确认具体答案。为避免给出错误内容，建议直接联系对应负责人"
    "（如 HR 或 IT）核实，或查阅公司内部正式文件。"
)

def apply_graceful(final: str) -> str:
    # 全部来自请求级 ContextVar，模型无法篡改
    if rag_called_ctx.get() and not rag_found_ctx.get() and not leave_found_ctx.get():
        return GRACEFUL_NO_ANSWER
    return final
```

`SYSTEM_PROMPT` 同步补两段纪律：友好降级（RAG 空结果时不要编，如实告知 + 建议联系 HR/IT），以及示意流程图（流程类回答末尾附 ` ```mermaid ` 块，给一个 `flowchart TD` 范例）。

`run_agent` 的两个收尾返回都过一遍安全网：

```python
if not ai_msg.tool_calls:
    return apply_graceful(ai_msg.content or "")
# ...（工具循环）...
return apply_graceful("抱歉，我在限定步数内没能给出确定答案……")
```

---

## 4. 踩坑

**改三重引号字符串，开头最容易丢。** 我在 `SYSTEM_PROMPT` 里加 Mermaid 示例（` ```mermaid ` 代码块）时，后续一次编辑的 `old_string` 正好包含了 `SYSTEM_PROMPT = """你是一个……` 这一行开头，而我在 `new_string` 里没补回它——结果整段 prompt 正文变成了「游离文本」，Python 直接 `SyntaxError`。教训：改三重引号字符串的开头行时，改完务必重新读一遍文件确认开头还在。已修复。

**提示词软约束不可靠，必须上结构闸。** 这次把「不编造」从一句 prompt 升级成 `apply_graceful` 的硬判定，根因就是吃过模型「自信编造」的亏。企业场景里，这种「结构性兜底」比任何措辞都重要。

**ContextVar 按请求隔离。** 接地标记默认 `False`，且只在请求级上下文有效。真实 FastAPI 里每次请求跑在独立 asyncio 任务/上下文里，标记不会串号；工具每次调用会重新置位，天然安全。

---

## 5. 验证

沙箱里没有 langchain/向量库依赖、也连不上外部 LLM，所以我把 `langchain_core` / `fastapi` / `app.rag.retrieve` / `app.db.leave` / `app.llm` 全部用**桩模块**替换，加载**真实的** `orchestrator.py` / `grounding.py` / 两个工具，再用一个假 LLM 模拟「调工具 → 返回 → 收尾」链路。四个场景全过：

| 场景     | 输入                                | 结果                                       |
| -------- | ----------------------------------- | ------------------------------------------ |
| 友好降级 | 「公司差旅报销标准多少？」（KB 空） | 编造回答被丢弃，返回`GRACEFUL_NO_ANSWER` |
| 闲聊     | 「你好」                            | 正常回复，**未误触发**降级           |
| 流程图   | 「年假请假怎么走流程」（KB 命中）   | 回复含` ```mermaid ` 流程图              |
| 权限红线 | lisi 问「张三的年假」               | 只返回 lisi 本人数据，不出现「张三」       |

关键断言片段（节选）：

```python
# 降级：编造内容必须被丢弃
assert out1 == orch.GRACEFUL_NO_ANSWER
assert "编造" not in out1
# 流程：必须带 mermaid
assert "```mermaid" in out3
# 权限：只返回本人
assert "李四" in out4 and "张三" not in out4
```

> 注：这是用桩验证逻辑层。**真实效果**（真 LLM 的措辞、真向量库的命中）请你本地 `pip install -r requirements.txt` + 配好密钥后跑 `uvicorn` 亲自看——和 M0~M3 一样，这一步得在你的环境里完成。

---

## 6. 架构图

![M4 聪明安全：友好降级 + 示意流程图](/images/agent-build-04-m4-smart-safe.svg)

图里多了一条「接地信号 → apply_graceful 判定」的分支：工具返回时给编排器打 `rag_called / rag_found / leave_found` 标记；收尾时若「调了检索却空手而归、也没查到假期」，就走红色降级话术，否则放行最终回答（流程类会自动带 ` ```mermaid ` 块）。

---

## 7. 预告

M4 让 Agent「答不出时不乱说、该画图时会画图」。但到目前为止，它还是个 `curl` 才能调的 API，身份还靠 `X-User-Id` 请求头（M0 桩，可被伪造）。

**下一篇 M5「交给同事」** 会做三件事：

1. 做一个简单的 **Web 界面**给同事用，并把 ` ```mermaid ` 块真的渲染成图（接 `mermaid.js`）；
2. 把 M0 的鉴权桩换成**真正的登录**（JWT / Session / SSO），让 M3 那句「身份可信」彻底闭环；
3. 顺手补上「对话历史 / 多轮」的小尾巴，让同事能连续追问。

到 M5，这个企业知识助手才算真正能「交给同事」。
