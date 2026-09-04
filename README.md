# JacoChen's AI Architect Blog

🧠 Building enterprise AI platforms from scratch. Architecture trade-offs, production lessons, and the engineering nobody talks about.

## Quick Start

### Method 1: GitHub Pages (Recommended)

1. Fork this repository (or create a new one)
2. Rename to `your-username.github.io`
3. Settings → Pages → Source: `main` branch
4. Wait 1-2 minutes, visit `https://your-username.github.io`

### Method 2: Local Preview

```bash
gem install bundler jekyll
bundle install
bundle exec jekyll serve
# Visit http://localhost:4000
```

## Adding New Posts

Create files in `_posts/` directory, naming format: `YYYY-MM-DD-title.md`

```markdown
---
layout: default
title: "Your Post Title"
date: 2026-09-04
categories: [Category1, Category2]
---

# Your Post Title

Content here...
```

## Directory Structure

```
blog/
├── _config.yml          # Jekyll configuration
├── _posts/              # Posts directory (add new posts here)
│   ├── 2026-09-04-building-enterprise-ai-workbench.md
│   ├── 2026-09-05-reduce-rag-hallucination.md
│   └── ... (11 posts total)
├── assets/images/       # SVG diagrams (27 figures)
├── index.md             # Homepage
├── about.md             # About page
└── README.md            # This file
```

## Author

**JacoChen** — AI Architect  
📧 jacochen1108@gmail.com
