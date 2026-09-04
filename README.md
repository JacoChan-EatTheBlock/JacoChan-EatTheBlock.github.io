# Van's AI Architect Blog

🧠 AI 架构实战笔记 · 从零到生产的全栈 AI 经验

## 快速开始

### 方法 1: GitHub Pages（推荐）

1. Fork 这个仓库（或创建新仓库）
2. 仓库名改为 `你的用户名.github.io`
3. Settings → Pages → Source 选 `main` 分支
4. 等 1-2 分钟，访问 `https://你的用户名.github.io`

### 方法 2: 本地预览

```bash
gem install bundler jekyll
bundle install
bundle exec jekyll serve
# 访问 http://localhost:4000
```

## 添加新文章

在 `_posts/` 目录下创建文件，命名格式：`YYYY-MM-DD-标题.md`

```markdown
---
layout: default
title: "你的文章标题"
date: 2026-09-04
categories: [分类1, 分类2]
---

# 你的文章标题

正文内容...
```

## 目录结构

```
blog/
├── _config.yml          # Jekyll 配置
├── _posts/              # 文章目录（添加新文章在这里）
│   ├── 2026-09-04-build-ai-platform-from-scratch.md
│   └── 2026-09-05-reduce-rag-hallucination.md
├── index.md             # 首页
├── about.md             # 关于页
├── projects.md          # 项目展示页
└── README.md            # 这个文件
```

## 作者

**Van Trang Duong** — AI Architect  
📧 duongvantrang38716@gmail.com
