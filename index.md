---
layout: default
title: Home
---

# 🧠 Van | AI Architect

> 全栈 AI 架构师 · 1-2 个月从零搭建企业级 AI 平台  
> LLM / RAG / Agent / 向量数据库 / K8s 部署

---

## 最新文章

{% for post in site.posts limit:10 %}
### [{{ post.title }}]({{ post.url }})
<small>{{ post.date | date: "%Y-%m-%d" }} · {{ post.categories | join: ", " }}</small>

{{ post.excerpt }}

---
{% endfor %}

## 关于我

5 年+ AI/后端工程经验，擅长从零搭建企业级 AI 平台。

- 🔧 **技术栈**: Python / LangChain / LangGraph / vLLM / RAG / Agent / K8s
- 🏗️ **核心能力**: 端到端 AI 系统设计 · 从原型到生产
- 📧 **联系**: duongvantrang38716@gmail.com
- 💼 **LinkedIn**: [点击查看](#)
- 🐙 **GitHub**: [点击查看](#)
