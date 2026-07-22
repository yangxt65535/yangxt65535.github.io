# Hugo 个人博客架构设计

日期：2026-07-23  
仓库：`yangxt65535.github.io`  
状态：已确认并实现（文章目录为 `content/post/`，以兼容 Clean White 的 `Type: post`）

## 1. 目标与范围

搭建以技术笔记为主的 Hugo 个人博客骨架，支持后续上传 Markdown 发布文章。

**纳入本次：**

- Hugo 源码站结构（配置、内容目录、静态资源预留）
- 主题：Clean White（`github.com/zhaohuabing/hugo-theme-cleanwhite`），通过 Hugo Modules 引入
- 文章扁平存放于 `content/post/`（主题要求 Type=`post`），用 tags / categories 区分技术文与学习笔记
- 预留 About、Projects 页面与图片目录
- GitHub Actions 构建并部署到 GitHub Pages
- 一份示例文章与 front matter 写作约定
- 删除/替换现有占位 `index.html`（由 Hugo 生成站点）

**明确不做（本次）：**

- 评论系统（utterances / giscus / Disqus 等）
- Algolia 站内搜索、打赏、统计分析账号接入
- 自定义主题深度改版
- 多语言（i18n）站点

## 2. 已确认决策

| 项 | 决定 |
| --- | --- |
| 内容重心 | 技术笔记为主，可有 About / Projects / 学习笔记 |
| 文章组织 | 扁平 `content/post/` + tags/categories |
| 主题 | Clean White |
| 主题引入 | Hugo Modules（方案 2） |
| 部署 | GitHub Actions → GitHub Pages |
| 评论 | 暂不做 |

## 3. 整体架构

```text
作者写 Markdown → 推送到 GitHub 源码仓
                 → GitHub Actions 运行 Hugo
                 → Hugo Modules 拉取 Clean White
                 → 产出静态站点
                 → 部署到 GitHub Pages
读者访问 https://yangxt65535.github.io
```

- 仓库只保留源码；不提交 `public/`
- 本地开发：`hugo server`（首次需 `hugo mod get` / `hugo mod tidy`）
- `baseURL` 设为 `https://yangxt65535.github.io/`

## 4. 仓库目录结构

```text
yangxt65535.github.io/
├── hugo.toml                 # 站点主配置
├── go.mod / go.sum           # Hugo Modules 锁定主题
├── archetypes/
│   └── default.md            # 新建文章模板（可选，从主题或本地）
├── content/
│   ├── about/                # 关于我
│   ├── archive/              # 归档页
│   ├── post/                 # 文章（扁平；主题要求 Type=post）
│   │   ├── hello-world.md
│   │   └── …
│   └── projects/             # 项目区预留
├── static/
│   └── img/                  # 头图、头像、文章配图预留
│       └── .gitkeep
├── layouts/                  # 空或最少覆盖（本次不写评论 partial）
├── assets/                   # 预留（若主题/自定义 CSS 需要）
├── .github/
│   └── workflows/
│       └── hugo.yml          # 构建 + Pages 部署
├── .gitignore                # public/、resources/_gen/、.hugo_build.lock、.superpowers/ 等
├── README.md                 # 本地预览与发文说明
└── docs/superpowers/specs/   # 本设计文档
```

**用户上传 Markdown 的位置：**

- 正文：`content/post/<slug>.md`（或 `content/post/<slug>/index.md` 若需文章专属资源）
- 配图：优先 `static/img/`，文章内引用 `/img/...`；或用 page bundle：`content/post/<slug>/` 下放 `index.md` + 图片

**现有根目录 `volcano custom plugin 编辑安装指南.md`：**  
实现阶段迁移到 `content/post/`，补全 front matter；不留在仓库根目录。

## 5. 配置要点（hugo.toml）

基于 Clean White `exampleSite/hugo.toml` 精简：

- `theme` 不通过本地 `themes/` 目录；改用：

```toml
[module]
[[module.imports]]
  path = "github.com/zhaohuabing/hugo-theme-cleanwhite"
```

- `languageCode` / `locale`：中文（`zh-cn`），`hasCJKLanguage = true`
- `title` / `params.SEOTitle` / `params.description` / `params.slogan` 使用个人占位文案（可后续改）
- 关闭或留空：Disqus、Algolia、reward、统计 ID（评论与搜索后续再开）
- 导航 `params.additional_menus`：ARCHIVE、ABOUT；Projects 可加一项指向 `/projects/`
- `params.social.github` 指向 `https://github.com/yangxt65535`
- `markup.goldmark.renderer.unsafe = true`（与主题示例一致，便于内嵌 HTML）

## 6. 内容约定

### Front matter（post）

```yaml
---
title: "文章标题"
date: 2026-07-23
draft: false
tags: ["Hugo", "GitHub Pages"]
categories: ["技术笔记"]   # 学习笔记用 categories: ["学习笔记"]
---
```

- 技术文：`categories` 建议 `技术笔记`（或更细分类如 `DevOps`）
- 学习笔记：`categories: ["学习笔记"]`，仍放在同一 `post/` 目录
- 建议 `date` 带时区（如 `+08:00`），避免 UTC 判定为未来文章而不发布
- `draft: true` 的文章本地可见，正式发布前改为 `false`

### tags / categories：不必提前锁定

Hugo 的 tags、categories 是**文章 front matter 里的自由 taxonomy**，不需要在配置里预先登记词表；新建文章时可随时新增，也可事后改已有文章的字段，重建/部署后站点列表与归档会跟着变。

- **categories**：建议少而稳（如 `技术笔记` / `学习笔记`），便于侧栏与归档浏览；需要时再拆细或改名。
- **tags**：可细、可多，按文章主题临时加即可；拼写不统一时，改几篇 front matter 就能合并。
- 本次**不**维护强制枚举清单；示例 front matter 仅作起步参考，不构成约束。

### 预留页

- `content/about.md`：关于我占位
- `content/projects/_index.md`：说明「项目介绍将放于此」，便于以后加 `projects/*.md`

## 7. 部署（GitHub Actions）

- Workflow：推送 `master`（及可选 `workflow_dispatch`）触发
- 步骤：检出 → 安装 Hugo Extended（稳定版本，实现时锁定具体 version）→ `hugo --minify` → 上传 Pages artifact → `actions/deploy-pages`
- 仓库 Settings → Pages：Source 设为 **GitHub Actions**（实现后在 README 中提醒用户检查）
- 权限：`contents: read`，`pages: write`，`id-token: write`

## 8. 本地与协作流程

1. 安装 Hugo Extended
2. `hugo mod tidy`（或首次 build 自动拉取）
3. `hugo server -D`
4. 新增文章：复制 archetype 或手写 front matter → 放入 `content/post/`
5. `git push` → Actions 自动上线

## 9. 错误与边界

- Modules 拉取失败：检查网络/代理；CI 使用官方 `peaceiris/actions-hugo` 或 `gohugoio` 推荐方式并缓存 module
- 主题更新：改 `go.mod` 版本约束后 `hugo mod get -u`，再提交 `go.mod`/`go.sum`
- 用户名 Pages 站点：`baseURL` 必须带尾斜杠的根域名，避免资源路径错误
- 不提交密钥；Algolia/评论若日后开启，密钥走 GitHub Secrets，不进仓库
- 文章目录必须为 `content/post/`（不能用 `posts`），否则首页列表为空（主题按 `Type: post` 过滤）

## 10. 成功标准

- 本地 `hugo server` 可打开首页、文章页、About、归档
- 向 `content/post/` 增加一篇 Markdown 后重建可见
- push 到 `master` 后 Actions 成功，公网可访问站点
- 仓库无 `public/` 构建产物；主题不以内嵌整仓或 submodule 形式存在
