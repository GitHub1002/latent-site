---
title: "系列二 · 05 | 阶段五·交给同事：Web 界面 + JWT 登录 + Mermaid 渲染"
date: 2026-07-18
draft: false
weight: 650
tags:
  - 大模型
  - Agent
  - 工程落地
  - 鉴权
  - 前端
  - 人机协作
summary: 把前面四个阶段的 Agent 真正「交给同事用」——做一个 React 聊天界面、用 JWT 替换可被伪造的 X-User-Id 桩、并把 M4 产出的 Mermaid 文本渲染成可见流程图。
---

## 1. 目标 & 进度

走到 M4，Agent 已经能「读懂私有文件（M1）」「查本人假期（M2）」「张三查不到李四（M3）」「答不出时不编造（M4）」。但有个尴尬事实：**它现在只是个 API 端点**——靠 `X-User-Id` 请求头假装身份，靠 `curl` 测。同事根本没法用。

M5 就是把它「交付」出去，三块具体活：

| 模块 | 做什么 | 替换了什么 |
|---|---|---|
| **Web 聊天界面** | React/Vite 单页，输入框 + 消息流 | 原来只有 `/chat` 接口、无界面 |
| **JWT 登录鉴权** | 账号密码登录发令牌，后端从令牌解身份 | 替换 M0 的 `X-User-Id` 可伪造桩 |
| **Mermaid 渲染** | 前端把 ` ```mermaid ` 文本画成图 | M4 只产出文本，现在出图 |

> 里程碑：**M0–M5 全部完成**，系列二「从 0 到 1 企业知识助手」到这里收官。

关键提醒：**权限红线一行没改**。M3 锁死的是「工具零 user 参数、SQL 只查注入身份」，M5 只是把「注入的身份从哪来」从请求头换成了 JWT 令牌——工具层完全无感。

## 2. 设计决策

### 前端为什么选 React/Vite
我给了三个选项，最终选了 React/Vite 单页：
- 单文件 HTML+JS：零构建链，但聊天/登录/状态管理一复杂就难维护；
- **React/Vite（选定）**：组件化物管清晰，开发期 Vite 自带代理把 `/chat` 指回后端，生产期 `npm run build` 产物直接由后端托管；
- Gradio/Streamlit：出 UI 最快，但难精确控制权限注入和 Mermaid 渲染，不适合要掌握权限的 Agent。

代价是多了条 npm 构建链，但对「要给同事长期用」的东西，这点投入值。

### 登录为什么选 JWT（而不是 Session Cookie）
这是 M5 真正的技术分叉。用「张三登录后问年假」串一下：

**Session Cookie**：登录后服务端在「会话表」写一行 `sid=abc → zhangsan`，浏览器自动带 cookie；登出 = 删那行，能一键踢人。状态在服务端。

**JWT Bearer（选定）**：登录后服务端**不存任何东西**，直接发一串签名过的 token（里面塞了 `zhangsan` 和过期时间）；前端每次请求手动挂 `Authorization: Bearer <token>`；登出 = 前端把 token 一扔。状态在 token 自己身上（无状态）。

我原本推荐 Session（内部小工具、要可控），但你选了 JWT。合理之处：以后若接多个服务/移动端，JWT 无需共享会话存储。代价是「强制下线难」——token 过期前始终有效。

**我们的解法**：用 `jti`（JWT ID）声明 + 一份内存吊销表，`/logout` 时把 `jti` 加进黑名单，让「踢人」也能演示。生产应把吊销表换 Redis/DB。

## 3. 关键实现

### 后端：`app/auth.py` 重写为 JWT
```python
# 登录发令牌
def create_access_token(user_id, username, jti):
    payload = {"sub": user_id, "username": username,
               "exp": now + timedelta(minutes=480), "jti": jti}
    return jwt.encode(payload, SECRET_KEY, algorithm="HS256")

# /chat 从 Bearer 解身份（不再读 X-User-Id）
async def get_current_user(credentials=Depends(bearer_scheme)) -> str:
    if not credentials or not credentials.credentials:
        raise HTTPException(401, "未登录或缺少 Authorization 头。")
    payload = jwt.decode(credentials.credentials, SECRET_KEY, algorithms=["HS256"])
    if payload.get("jti") in _REVOKED:        # 支持踢人
        raise HTTPException(401, "该登录已被吊销。")
    return payload["sub"]                     # = user_id，注入 user_id_ctx
```
**契约不变**：`get_current_user` 解出的 `user_id` 仍然走 `user_id_ctx.set(...)`，下游请假工具零参数读取——M3 红线原封不动。

### 账号数据层：`app/db/users.py`
密码用标准库 `hashlib.pbkdf2_hmac` 加盐哈希（避免引入 bcrypt 这种原生编译依赖，受限环境易装失败），存储格式 `pbkdf2_sha256$<迭代>$<盐>$<哈希>`。演示账号 `zhangsan/lisi/wangwu` 密码均 `pass1234`，与假期库的 `user_id` 对齐，这样登录后查假期能对得上。

### 登录/登出端点（`app/main.py`）
```python
@app.post("/login")
def login(req: LoginRequest):
    user = user_db.authenticate(req.username, req.password)
    if not user: raise HTTPException(401, "用户名或密码错误。")
    return LoginResponse(access_token=create_access_token(...), name=user["name"])

@app.post("/logout")
def logout(credentials=Depends(bearer_scheme)):
    revoke_from_credentials(credentials)   # 吊销 jti，演示踢人
    return {"status": "ok"}
```

### 前端：登录页 + 聊天页 + Mermaid
前端调接口时**手动挂令牌**——这正是 JWT 和 Session 最大的使用差异：
```javascript
// api.js
export async function chat(token, message) {
  const res = await fetch('/chat', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${token}` },
    body: JSON.stringify({ message }),
  })
  return (await res.json()).reply
}
```
Mermaid 渲染是 M4→M5 的分工落点。M4 让模型产出 ` ```mermaid ` 文本，M5 前端用 `mermaid.js` 把它画出来：
```javascript
// Mermaid.jsx
import mermaid from 'mermaid'
mermaid.initialize({ startOnLoad: false })
useEffect(() => {
  mermaid.render(id, code).then(({ svg }) => { ref.current.innerHTML = svg })
}, [code])
```
回复先按正则拆成「文本段」和「mermaid 段」，文本走 `<pre>`，图走 `<div>` 渲染。

生产部署：前端 `npm run build` → `frontend/dist/`，后端用 `StaticFiles` 直接托管，开发期则靠 Vite 代理指回 8000，两边互不干扰。

## 4. 踩坑

1. **JWT 密钥长度告警**：开发默认密钥只有 30 字节，PyJWT 报 `InsecureKeyLengthWarning`（SHA256 建议 ≥32 字节）。已补到 32+ 字节；生产务必用环境变量 `JWT_SECRET` 覆盖，且别提交进仓库。
2. **JWT 登出是假象**：无状态 token 服务端「不知道」它失效，前端丢了 token 就等于登出，但那串 token 在过期前仍可被截获重用。我们用 `jti` + 内存吊销表缓解（重启即失效，生产换 Redis）。
3. **StaticFiles 挂载顺序**：必须**先注册 `/chat`、`/login` 等 API 路由，再把 `StaticFiles` 挂到 `/`**，否则前端 `index.html` 会吃掉 `/chat` 路径。代码里用「dist 目录存在才挂载」规避开发期冲突。
4. **Vite 代理 vs 同源**：开发期前端在 5173、后端在 8000，靠 `vite.config.js` 的 `server.proxy` 把 `/chat` 指回后端；一旦走代理，`fetch` 的 BASE 必须留空（相对路径），否则跨域报错。

## 5. 验证

沙箱无外部 LLM，但 **M5 的新增逻辑（鉴权）可以脱离 LLM 独立验证**。我用桩夹具直接测 `app.auth` + `app.db.users`，五项全过：

| 场景 | 输入 | 结果 |
|---|---|---|
| 登录成功 | `zhangsan` / `pass1234` | 返回 JWT，解出 `user_id=zhangsan` |
| 密码错误 | `zhangsan` / `wrong` | `authenticate` 返回 None，登录拒 |
| 缺令牌 | 无 `Authorization` 头 | `/chat` 返回 401 |
| 过期令牌 | `exp=0` 的 JWT | 返回 401 |
| 已吊销 | 调过 `/logout` 的令牌 | 返回 401（踢人可用） |

核心断言：登录拿到的 JWT，经 `get_current_user` 解出 **张三本人** `zhangsan` 并注入 `user_id_ctx`——权限红线在 JWT 模式下继续成立（之前 M4 已验证「lisi 查不到 zhangsan」）。

> 前端 React 应用在沙箱只做了源码编写与构建配置；完整 UI 交互（登录→聊天→看流程图）需你本地 `cd frontend && npm install && npm run dev` 后浏览器验证。

## 6. 架构图

![M5 架构：Web + JWT + Mermaid](/images/agent-build-05-m5-web-jwt-mermaid.svg)

红色虚线依然是 M3 的「身份只来自服务端」边界——只是 M5 把它从「请求头」挪到了「JWT 令牌」。模型那一侧依旧没有任何「指定查谁」的入口。

## 7. 预告

系列二到这里就**完整收官**了：一个能「读懂你公司私有文件、查本人假期、张三查不到李四、答不出不编造、带流程图、同事能在浏览器里登录使用」的企业知识助手，从 0 到 1 全部落地。

后续可选篇章（不在 M0–M5 主线）：
- **多 Agent 协作**：给当前单 Agent 加 Auditor（审答案）/ Planner（拆解复杂任务）；
- **评估与可观测**：把 M4 的「接地信号」接上 Langfuse，量化「编造率」「降级率」；
- **接公司真实数据源**：假期库换成 MySQL/HR 系统，文档换成 Confluence/企微知识库。

如果想继续，告诉我从哪篇开写。
