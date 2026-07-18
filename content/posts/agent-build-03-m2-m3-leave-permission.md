---
title: "系列二 · 03 | 阶段三·查数据 + 权限红线：让 Agent 安全查询年假"
date: 2026-07-18
draft: false
weight: 630
tags:
  - Agent
  - 企业知识助手
  - 权限
  - 数据库
  - LangChain
summary: "系列二第三阶段：给 Agent 接入数据库查询（年假），并用「服务端注入身份 + 工具无 user 参数」把权限红线钉死，让张三永远查不到李四。"
ShowToc: true
---

> 系列二主线：边写一个企业知识助手 Agent，边把它拆成可落地的阶段文章。
> 上一篇（[阶段二·读懂私有文件 RAG](/posts/agent-build-02-m1-rag/)）让 Agent 能查文档；这一篇让它**能查数据库**，并且**安全地查**。

## 一、目标 & 进度

回顾一下我们已经走过的路：

- **M0 最小闭环**：FastAPI + 单轮问答，身份从 `X-User-Id` 注入（搭好骨架）。
- **M1 读懂文件**：接入 RAG，Agent 自己决定何时检索私有文档。
- **本篇 M2+M3**：让 Agent **能查数据库（年假）**，并给查询加上**权限红线**。

| 里程碑 | 解决什么 | 关键产出 |
|--------|----------|----------|
| M2 查数据 | 员工想知道「我的年假还有几天」 | `get_leave_balance` 工具 + SQLite 数据层 |
| M3 权限红线 | 张三不能查到李四的年假 | `user_id` 服务端注入上下文，工具无 user 参数 |

进度条：`M0 ✅ → M1 ✅ → M2+M3 ✅ → M4(流程图+降级) → M5(交给同事)`

## 二、设计决策

### 1. 为什么先做「年假」这个查询

年假是员工最高频、又**天然带权限属性**的问题：每个人只能看自己的。它比「查全公司工资表」风险小、比「查公开制度」更有「查数据库」的代表性。所以拿它当 M2 的第一个数据库工具最合适。

### 2. 数据库选型：本地用 SQLite，生产换 MySQL/PostgreSQL

设计稿当初写的是 MySQL/PostgreSQL（年假字段现成、直接查）。但本地开发如果还要先起一个数据库服务，体验就很差。我的取舍是：

- **本地开发用 SQLite**（`data/leave.db`，Python 标准库自带，零依赖、开箱即用）；
- 把所有数据库访问收敛到一个函数 `get_connection()`，生产环境只要把这里换成 `pymysql` / `psycopg2` 的连接，**查询代码一行不用改**。

这正是「效果优先」——先用最小成本把链路跑通，换库是配置级改动。

### 3. 权限红线怎么落（本篇重点）

这是 M3 的核心，也是一个很容易做错的地方。常见的错误是：让模型在工具调用里传 `user_id`，比如 `get_leave_balance(user_id="lisi")`。**一旦这样，张三就能骗模型去查李四**——因为模型完全可控。

正确做法是**结构性隔离**：

> 模型只决定「调不调」(whether)，服务端决定「查谁」(who)。

具体落三点：

1. `user_id` 在 `/chat` 入口由服务端从 `X-User-Id` 注入到一个**请求级上下文**（`contextvars`）；
2. 工具函数**干脆没有 user 参数**，它只能从上下文读出「当前登录用户」；
3. DB 查询永远按这个注入的 `user_id` 做参数化查询。

模型在工具调用里根本没有「指定查谁」的入口，红线就被钉死了。

## 三、关键实现

### 3.1 数据层 `app/db/leave.py`

所有 DB 访问集中在这里，查询全部参数化：

```python
def get_connection(db_path: str | None = None) -> sqlite3.Connection:
    """返回 SQLite 连接。生产换 MySQL/PostgreSQL 只改这里。"""
    db_path = db_path or os.getenv("LEAVE_DB_PATH") or str(DEFAULT_DB_PATH)
    Path(db_path).parent.mkdir(parents=True, exist_ok=True)
    conn = sqlite3.connect(db_path)
    conn.row_factory = sqlite3.Row
    return conn

def get_leave_balance(user_id: str) -> dict | None:
    """查询单个员工的假期余额。参数化查询，安全。"""
    with get_connection() as conn:
        cur = conn.execute(
            "SELECT user_id, name, annual_total, annual_used, sick_total, sick_used "
            "FROM leave_balances WHERE user_id = ?",   # ? 占位，杜绝 SQL 注入
            (user_id,),
        )
        row = cur.fetchone()
    if row is None:
        return None
    return {
        "user_id": row["user_id"], "name": row["name"],
        "annual_total": row["annual_total"], "annual_used": row["annual_used"],
        "annual_remaining": row["annual_total"] - row["annual_used"],
        "sick_total": row["sick_total"], "sick_used": row["sick_used"],
        "sick_remaining": row["sick_total"] - row["sick_used"],
    }
```

种子数据用 `scripts/seed_db.py` 灌（张三/李四/王五），方便联调。

### 3.2 权限上下文 `app/auth.py`

加一个请求级的 `ContextVar`，身份在这里落地：

```python
import contextvars
from fastapi import Header, HTTPException

# 关键点：该值由服务端注入，模型永远无法通过工具参数覆盖它——权限红线的根基。
user_id_ctx: contextvars.ContextVar[str] = contextvars.ContextVar("user_id", default="")
```

### 3.3 入口注入 `app/main.py`

`/chat` 在调用 Agent 前把身份写进上下文，结束后 `reset`（避免异步并发串号）：

```python
from app.auth import get_current_user, user_id_ctx

@app.post("/chat", response_model=ChatResponse)
async def chat(req: ChatRequest, user_id: str = Depends(get_current_user)):
    token = user_id_ctx.set(user_id)      # 服务端注入
    try:
        reply = await run_agent(req.message, user_id)
    finally:
        user_id_ctx.reset(token)          # 必须 reset
    return ChatResponse(reply=reply, user_id=user_id)
```

### 3.4 工具 `app/agent/tools/leave_tool.py`（无 user 参数）

注意工具函数**没有任何 user 形参**，它读的是上下文里的当前用户：

```python
from langchain_core.tools import tool
from app.auth import user_id_ctx
from app.db.leave import get_leave_balance as fetch_leave_balance

@tool
def get_leave_balance() -> str:
    """查询**当前登录用户本人**的年假和病假余额。只能查本人，无法查他人。"""
    user_id = user_id_ctx.get()           # 来自服务端，不是模型传的
    if not user_id:
        return "未识别到登录用户，无法查询假期余额。"
    data = fetch_leave_balance(user_id)
    if data is None:
        return f"未找到用户 {user_id} 的假期记录，请联系 HR 核实。"
    return (f"【{data['name']}（{data['user_id']}）的假期余额】\n"
            f"年假：总共 {data['annual_total']} 天，已用 {data['annual_used']} 天，"
            f"剩余 {data['annual_remaining']} 天\n"
            f"病假：总共 {data['sick_total']} 天，已用 {data['sick_used']} 天，"
            f"剩余 {data['sick_remaining']} 天")
```

> 注意 import 时用 `as fetch_leave_balance` 给 DB 函数改名，避免和工具同名遮蔽（见踩坑 4.1）。

### 3.5 注册到编排器 `app/agent/orchestrator.py`

把工具加进列表，并在系统提示里明确权限红线：

```python
from app.agent.tools.leave_tool import get_leave_balance as get_leave_balance_tool

SYSTEM_PROMPT = """...（略）...
- 当用户询问"我的年假/病假还有多少"等**本人假期**问题时，调用 get_leave_balance 工具；
权限红线（务必遵守）：
- get_leave_balance 只能查**当前登录用户本人**，你无法也不得尝试查询他人假期；
  若用户想查别人的假期，请礼貌告知本助手仅支持查询本人。
- 你永远不要假设或编造某个具体员工（如"李四"）的假期数字。"""

tools: list[BaseTool] = [search_knowledge_base, get_leave_balance_tool]
```

## 四、踩坑

### 4.1 命名遮蔽：DB 函数和工具同名

一开始工具里写 `from app.db.leave import get_leave_balance`，又定义 `@tool def get_leave_balance()`。Python 里后定义的直接把前面的覆盖了——工具体内再调用 `get_leave_balance(user_id)` 就变成**调自己**（递归/报错）。
**修法**：DB 函数 import 时改名 `as fetch_leave_balance`。

### 4.2 contextvar 必须 reset

`contextvars` 是**请求级**的，但 FastAPI + asyncio 下同一个线程可能服务多个请求。如果只 `set` 不 `reset`，下一个请求的上下文可能读到上一个用户的 `user_id`——**串号是严重权限事故**。
**修法**：用 `try/finally` 包住，结束必 `reset(token)`。

### 4.3 SQLite 返回行不是 dict

默认 `sqlite3` 返回的是 tuple，按索引取易错。用 `conn.row_factory = sqlite3.Row` 后可用 `row["user_id"]` 按列名取，清晰且不易错位。

### 4.4 工具无参数时 LangChain 的调用

因为 `get_leave_balance` 没有参数，模型发起工具调用时 `args` 是空对象 `{}`。验证脚本里我用 `leave_tool.invoke({})` 调用；外部即使有人试图传 `{"user_id":"lisi"}` 也会被忽略（工具根本不读这个参数）。

## 五、验证

我写了一个临时校验脚本，跑通了四条关键断言（已在沙箱用独立 venv 验证，`import app.main` 无导入错误）：

```
[OK] DB 层：zhangsan 剩余 7 天；lisi 已用 0；nobody 返回 None
[OK] 权限红线：工具参数={}（无 user 字段，模型无法指定查谁）
[OK] 以 zhangsan 身份调用 -> 【张三（zhangsan）的假期余额】 ...（仅本人）
[OK] 尝试传入 user_id=lisi 被忽略，仍返回：【张三（zhangsan）的假期余额】
[OK] 未知用户 -> 未找到用户 ghost 的假期记录，请联系 HR 核实。
[OK] import app.main 成功（含 M2+M3 改动）
```

**你本机怎么跑**：

```bash
# 1) 灌样本数据
python scripts/seed_db.py

# 2) 起服务（另一个终端）
uvicorn app.main:app --reload

# 3) 用 Swagger 最省事：浏览器开 http://127.0.0.1:8000/docs
#    POST /chat → body: {"message":"我的年假还有几天？"} → X-User-Id 填 zhangsan → Execute
```

想确认「张三查不到李四」？把 `X-User-Id` 换成 `zhangsan`、问题写成「李四的年假还有几天」，Agent 会礼貌回绝——因为工具根本不接收「查谁」的参数。

## 六、架构图

![M2+M3 架构：查数据 + 权限红线](/images/agent-build-03-m2-m3-leave-permission.svg)

这张图的核心就一句话：**红色虚线标出的身份，只来自服务端注入，模型那一侧根本没有「指定查谁」的入口**。所以张三查李四在结构上不可能发生。

## 七、预告

下一篇 **M4（聪明 & 安全）**：

- 当问题适合用流程图表达时（如「请假审批怎么走」），让 Agent **输出 Mermaid 流程图**而不是大段文字；
- 当模型真的答不上来时，给一个**友好降级**提示，而不是硬编或报错。

到 M4，这个 Agent 就已经具备「能查文档、能查本人数据、答得出流程图、答不出会认怂」的雏形了。我们下篇见。

---

*系列二文章索引：[00 路线图](/posts/agent-build-00-roadmap/) · [01 M0 搭地基](/posts/agent-build-01-m0-foundation/) · [02 M1 RAG](/posts/agent-build-02-m1-rag/) · 03 M2+M3（本篇）*
