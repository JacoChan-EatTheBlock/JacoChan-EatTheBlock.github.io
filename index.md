---
layout: default
title: JacoChen — AI Architect
---

# JacoChen

**AI Architect** · Building enterprise AI platforms from scratch

I design and ship production AI systems — LangGraph orchestration, MongoDB semantic layers, anti-hallucination pipelines, and the architecture trade-offs nobody writes about. One engineer, ~300 users, four business domains, eight months in production.

---

## Architecture & Trade-offs

**[Six Trade-offs That Shaped Our Enterprise AI Workbench]({{ site.baseurl }}/2026/09/04/building-enterprise-ai-workbench/)**
LangGraph vs plain chains, MongoDB for everything, monolith with escape hatches, K8s sandbox security, three-layer memory, and on-premise domestic LLM.

**[How I Built an Enterprise AI Platform from Scratch in 2 Months]({{ site.baseurl }}/2026/09/04/build-ai-platform-from-scratch/)**
Week-by-week account of building a four-domain AI workbench solo. The shortcuts that saved time, the decisions that created debt, and the three things I'd change.

**[From Text-to-MQL to AI-Native Dashboards: Five Architectural Shifts]({{ site.baseurl }}/2026/09/05/from-text-to-mql-to-ai-native-dashboards/)**
Semantic views with additive collapse, constrained vocabulary, multi-gate intent routing with slot guards, precedent-based generation, and the math behind it all.

---

## AI Quality Engineering

**[7 Production Techniques That Cut Our RAG Hallucination Rate by 80%]({{ site.baseurl }}/2026/09/05/reduce-rag-hallucination/)**
Structured output, few-shot anchoring, bigram scoring for Chinese NLP, dual-use intent routing, output field validation, and the evaluation loop. From 35% to under 7%.

**[Evals and Multi-Agent Coordination in Production]({{ site.baseurl }}/2026/09/06/evals-and-multi-agent-coordination-in-production/)**
Three-layer evaluation, compound intent decomposition, and code-orchestrated DAG execution.

---

## LLM Engineering

**[Prompt Engineering for Enterprise AI: The Semantic Mapper Philosophy]({{ site.baseurl }}/2026/09/06/prompt-engineering-for-enterprise-ai/)**
Prompts are not conversations — they are dynamically compiled business logic. The four pillars, the sandwich architecture, and three surgical cuts.

**[Context Engineering Lessons from Production]({{ site.baseurl }}/2026/09/06/context-engineering-lessons-from-production/)**
Three-layer memory, rolling context compression, the context budget problem, and why we include assistant history in the router.

**[Memory Systems and Token Economics for Production Agents]({{ site.baseurl }}/2026/09/06/memory-systems-and-token-economics-for-production-agents/)**
Semantic vs episodic memory, inject-cheap-write-expensive pattern, per-model token pricing, and cost attribution.

---

## Infrastructure & Security

**[Building AI Observability Without LangSmith]({{ site.baseurl }}/2026/09/06/building-ai-observability-without-langsmith/)**
On-premise means no SaaS observability. Our 8-table lineage system with ~2ms overhead per node.

**[Designing Tools for Production AI Agents]({{ site.baseurl }}/2026/09/06/designing-tools-for-production-ai-agents/)**
SkillHub marketplace, component registry as tool definition, semantic query plans, and evaluation-driven tool improvement.

**[Sandboxing AI Skills in Kubernetes]({{ site.baseurl }}/2026/09/06/k8s-sandbox-security-for-ai-skill-execution/)**
YARA scanning, K8s ephemeral Jobs, NetworkPolicy isolation, prepuller DaemonSets, and zero-credential sandbox execution.

---

## 📚 阅读见解

### 财富与投资

**[「纳瓦尔宝典」：一个技术人读完后的 7 个认知升级]({{ site.baseurl }}/2026/09/06/naval-almanack-reading-notes/)**
Specific Knowledge、杠杆天花板、Code and media are permissionless leverage。

**[从「巴菲特写给股东的信」中偷到的 6 个系统设计思维]({{ site.baseurl }}/2026/09/06/buffett-letters-reading-notes/)**
护城河→技术壁垒，安全边际→防御性编程，复利→知识积累，能力圈→不追热点。

**[一个老师能攒到 600 万，程序员为什么不行？]({{ site.baseurl }}/2026/09/06/millionaire-teacher-reading-notes/)**
指数基金、复利的 for 循环、理财顾问的利益冲突。

**[慢慢富的底层逻辑：4 本理财书教我的「反直觉」金钱观]({{ site.baseurl }}/2026/09/07/wealth-slow-and-steady-reading-notes/)**
先整理再投资、三个桶框架、零成本杠杆、复利的敌人。

### 思维模型与人生系统

**[「穷查理的普通常识」：反过来想，总是反过来想]({{ site.baseurl }}/2026/09/07/poor-charlie-almanack-reading-notes/)**
多元思维模型、能力圈、避免愚蠢比追求聪明更重要。

**[「原则」：把人生当系统来运维]({{ site.baseurl }}/2026/09/07/principles-ray-dalio-reading-notes/)**
痛苦+反思=进步、极度透明、算法化决策、Dalio 的 5 步流程。

**[程序员必备的 10 个思维模型]({{ site.baseurl }}/2026/09/07/great-mental-models-reading-notes/)**
地图不是疆域、第一性原理、二阶思维、奥卡姆剃刀、幸存者偏差。

**[马斯克的「算法」：五步法背后的工程暴力美学]({{ site.baseurl }}/2026/09/07/elon-musk-reading-notes/)**
Question → Delete → Simplify → Accelerate → Automate，必须按这个顺序。

### 职业独立与 AI 杠杆

**[从「找到想做的事」到「不上班也能活」]({{ site.baseurl }}/2026/09/07/find-your-path-reading-notes/)**
三环模型、读书改变行为、6 步脱离上班、MVP 思维、AI 时代的独立路线。

**[AI 杠杆时代的不对称赌注：下行可控、上行巨大]({{ site.baseurl }}/2026/09/07/ai-leverage-asymmetric-bets-and-finding-your-mission/)**
5 种 AI 杠杆 trade-off、工程化人生决策、使命不是找到的是长出来的。

### AI 行业洞察

**[Jeff Dean 的「1% 法则」与 AI 基础设施的终局思考]({{ site.baseurl }}/2026/09/07/yc-jeff-dean-1-percent-rule-ai-insights/)**
模型只是冰山 1%、TPU 的领域专用化启示、AlphaChip、AlphaEvolve、知识蒸馏。

---

## Tech Stack

| Layer | Stack |
|-------|-------|
| Orchestration | LangGraph — stateful graphs, interrupt/resume, sub-graph composition |
| Database | MongoDB — checkpoint + store + business + vector search |
| LLM | Structured output (Pydantic), semantic routing, few-shot anchoring |
| Security | YARA scanning, K8s sandbox, NetworkPolicy, PIPL compliance |
| Observability | Custom 8-table lineage (no LangSmith — on-premise) |

---

## Contact

- GitHub: [JacoChan-EatTheBlock](https://github.com/JacoChan-EatTheBlock)
- Email: jacochen1108@gmail.com
