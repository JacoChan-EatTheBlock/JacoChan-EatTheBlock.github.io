---
layout: default
title: JacoChen — AI Architect
---

# JacoChen

**AI Architect** · Building enterprise AI platforms from scratch

I design and ship production AI systems — LangGraph orchestration, MongoDB semantic layers, anti-hallucination pipelines, and the architecture trade-offs nobody writes about. One engineer, ~300 users, four business domains, eight months in production.

---

## Writing

### Architecture & Trade-offs

**[Six Trade-offs That Shaped Our Enterprise AI Workbench]({{ site.baseurl }}/2026/09/04/building-enterprise-ai-workbench/)**
LangGraph vs plain chains, MongoDB for everything, monolith with escape hatches, K8s sandbox security, three-layer memory, and on-premise domestic LLM. What we gained, what we accepted.

**[How I Built an Enterprise AI Platform from Scratch in 2 Months]({{ site.baseurl }}/2026/09/04/build-ai-platform-from-scratch/)**
Week-by-week account of building a four-domain AI workbench solo. The shortcuts that saved time, the decisions that created debt, and the three things I'd change.

### AI Quality Engineering

**[7 Production Techniques That Cut Our RAG Hallucination Rate by 80%]({{ site.baseurl }}/2026/09/05/reduce-rag-hallucination/)**
Structured output enforcement, few-shot anchoring, bigram scoring for Chinese NLP, dual-use intent routing, output field validation, and the evaluation loop that made it all measurable. From 35% to under 7%.

### Domain Intelligence

**[From Text-to-MQL to AI-Native Dashboards: Five Architectural Shifts]({{ site.baseurl }}/2026/09/05/from-text-to-mql-to-ai-native-dashboards/)**
Semantic views with additive collapse, constrained vocabulary, multi-gate intent routing with slot guards, precedent-based generation, and the math behind it all (set theory, DAGs, normalization). Three architecture rewrites in eight months.

---

## What I Work With

| Layer | Stack |
|-------|-------|
| Orchestration | LangGraph — stateful graphs, interrupt/resume, sub-graph composition |
| Database | MongoDB — checkpoint + store + business + vector search (one DB for everything) |
| LLM Engineering | Structured output (Pydantic), semantic routing, few-shot anchoring, evaluation loops |
| Security | YARA scanning, K8s sandbox isolation, PIPL compliance, on-premise deployment |
| Observability | Custom 8-table lineage system (no LangSmith — on-premise constraints) |

---

## Contact

- GitHub: [JacoChan-EatTheBlock](https://github.com/JacoChan-EatTheBlock)
- Email: duongvantrang38716@gmail.com
