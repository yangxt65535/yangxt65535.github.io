# yangxt65535 的 Hugo 博客

基于 [Clean White](https://github.com/zhaohuabing/hugo-theme-cleanwhite) 主题，通过 Hugo Modules 引入，使用 GitHub Actions 部署到 GitHub Pages。

站点地址：https://yangxt65535.github.io/

## 目录约定

| 路径 | 用途 |
| --- | --- |
| `content/post/` | **文章存放处**（扁平放置，用 tags / categories 区分） |
| `content/about/` | 关于我 |
| `content/projects/` | 项目介绍预留 |
| `content/archive/` | 归档页 |
| `static/img/` | 头图、头像、配图 |

> Clean White 按 `Type: post` 收集首页文章，因此目录名必须是 `post`（不是 `posts`）。

## 本地预览

1. 安装 [Hugo Extended](https://gohugo.io/installation/)（建议 ≥ 0.120）
2. 在仓库根目录执行：

```bash
hugo mod tidy
hugo server
```

浏览器打开 http://localhost:1313/

## 写一篇新文章

```bash
hugo new post/my-article.md
```

或直接在 `content/post/` 新建 Markdown，front matter 示例：

```yaml
---
title: "文章标题"
date: 2026-07-22T12:00:00+08:00
draft: false
tags: ["Hugo"]
categories: ["技术笔记"]
---
```

- `categories`：建议少而稳，如 `技术笔记` / `学习笔记`
- `tags`：可随时增减，无需预先登记
- `draft: true` 时仅本地 `-D` 可见
- 建议 `date` 带时区（如 `+08:00`），避免被当成未来文章而不发布

## 部署

推送到 `master` 后，`.github/workflows/hugo.yml` 会自动构建并发布。

首次使用前请在 GitHub 仓库设置中确认：

**Settings → Pages → Build and deployment → Source** 选择 **GitHub Actions**。

## 设计文档

见 [docs/superpowers/specs/2026-07-23-hugo-blog-architecture-design.md](docs/superpowers/specs/2026-07-23-hugo-blog-architecture-design.md)。
