---
layout: default
title: "我如何在 2 个月内从零搭建企业级 AI 平台"
date: 2026-09-04
categories: [AI架构, 实战]
---

# 我如何在 2 个月内从零搭建企业级 AI 平台

> 这不是吹牛，而是方法论。把正确的事情按正确的顺序做，速度自然就上来了。

## 为什么能这么快？

大多数人搭 AI 平台慢，不是因为技术不够，而是因为：

1. **没有删除不需要的东西**（马斯克算法第二步）
2. **先优化了不该存在的流程**（马斯克算法第三步）
3. **一开始就想做得"完美"**

我的方法论：**先做最小可用版本 → 上线 → 根据真实反馈迭代**。

## 技术选型决策树

| 层 | 选型 | 为什么 |
|---|------|--------|
| LLM | GPT-4o（主） + Claude 3.5（备） | 前者综合最强，后者长文本+安全场景 |
| RAG | LangChain + Qdrant | LangChain 生态最全，Qdrant 开源可控 |
| Agent | LangGraph | 比纯 LangChain 更灵活的状态管理 |
| API | FastAPI | 异步 + SSE Streaming + 性能好 |
| 部署 | Docker + K8s | 标准化，任何云都能跑 |
| 监控 | OpenTelemetry + Grafana | 开源全栈可观测 |

## 第一周做什么

```
Day 1-2: 搭建基础 API 骨架（FastAPI + Docker）
Day 3-4: 接入 LLM + 实现基础对话
Day 5:   搭建 RAG Pipeline（文档解析 → Chunking → Embedding → Qdrant）
```

## 第二周做什么

```
Day 6-7: Agent 编排（LangGraph）
Day 8-9: 工具调用（Function Calling）
Day 10:  部署到 K8s + 基础监控
```

## 最容易踩的 5 个坑

### 坑 1: Chunking 策略选错
...（待补充你的真实经验）

### 坑 2: 向量检索的"假阳性"
...

### 坑 3: Agent 死循环
...

### 坑 4: SSE Streaming 的坑
...

### 坑 5: Token 成本失控
...

## 最终架构图

（待补充）

## Takeaway

1. 先删除不需要的，再优化剩下的
2. 第一版不需要完美，需要能跑
3. 选型不要追热点，选你最熟的
4. 先部署再优化，不要在本地调到完美
5. 监控从第一天就要有

---

*有问题欢迎在 GitHub Issues 讨论。*
