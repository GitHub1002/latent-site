# Latent

> Notes from the latent space of AI

个人技术博客，记录大模型（LLM）与智能体（Agent）方向的学习笔记。

## 技术栈

- [Hugo](https://gohugo.io/) (Extended) — 静态站点生成器
- [PaperMod](https://github.com/adityatelange/hugo-PaperMod) — 主题
- [Cloudflare Pages](https://pages.cloudflare.com/) — 托管
- [Giscus](https://giscus.app/) — 评论系统
- [MathJax](https://www.mathjax.org/) — 数学公式渲染

## 本地开发

```bash
# 安装 Hugo Extended
winget install Hugo.Hugo.Extended

# 启动本地预览
hugo server --buildDrafts

# 访问 http://localhost:1313
```

## 写新文章

```bash
hugo new content posts/my-new-post.md
```

编辑 `content/posts/my-new-post.md`，写完后将 `draft: true` 改为 `draft: false`。

## 部署

推送到 GitHub 后，Cloudflare Pages 会自动构建部署。

构建命令：`hugo --gc --minify`
输出目录：`public/`
