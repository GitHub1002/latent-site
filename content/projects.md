---
title: "项目"
date: 2026-07-05
ShowToc: false
ShowReadingTime: false
ShowWordCount: false
ShowBreadCrumbs: false
ShowPostNavLinks: false
---

这里只放 GitHub 上真实在维护的仓库。点开就能读代码。

## 精选项目

### PocketPilot

本地优先、策略可控、可审计的个人 Agent Runtime（Python 3.12 + Next.js）。用户授权工作区后，Agent 可以读文件、抽文档、写报告；**凡有副作用的工具调用都必须先过策略检查与人工审批**，过程以结构化事件落库，并可经 SSE 观察。

`Python` `FastAPI` `Next.js` `Agent Runtime` · [系列解读 →]({{< relref "posts/pocketpilot-01-thesis.md" >}}) · [GitHub →](https://github.com/GitHub1002/PocketPilot)

---

### 墨坊

卖墨、纸、印泥的 Java 小店（Spring Boot 4 + 网关 + 三库）。浏览器只访问 8080；下单先 202 收下，库存经 RabbitMQ 后扣，余额不够再把库存加回去。两份店铺注册到 Nacos，门口限流与熔断。

`Java 21` `Spring` `RabbitMQ` `Nacos` · [系列解读 →]({{< relref "posts/ink-workshop-01-thesis.md" >}}) · [GitHub →](https://github.com/GitHub1002/ink-workshop)

---

### ResumeForge

个人求职工坊。事实库里只放真实经历，再按岗位生成多份简历，导出 PDF 或 Markdown，并记下投了哪一家。模型只能改写已经写过的内容，不会新增公司、项目或技术。

`TypeScript` `React` `Hono` `Typst` · [GitHub →](https://github.com/GitHub1002/resume-forge)

---

### 企业知识助手

博客系列二的配套代码：企业内部知识助手。私有文件做 RAG，能查库回答「还剩几天年假」，权限按人切开，答不上来时给友好提示。

`Python` `FastAPI` `RAG` `LangChain` · [系列解读 →]({{< relref "posts/agent-build-00-roadmap.md" >}}) · [GitHub →](https://github.com/GitHub1002/enterprise-knowledge-agent)

---

### ai-agent

基于 Spring Boot 与 Spring AI 的智能体：恋爱咨询问答，接 RAG 与聊天记忆，支持 DashScope / Ollama，并带工具调用（网页抓取等）。

`Java 21` `Spring AI` `RAG` · [GitHub →](https://github.com/GitHub1002/ai-agent)
