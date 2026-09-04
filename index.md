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
Three-layer evaluation (real-time gate, nightly calibration, human feedback loop), compound intent decomposition, and code-orchestrated DAG execution.

---

## LLM Engineering

**[Prompt Engineering for Enterprise AI: The Semantic Mapper Philosophy]({{ site.baseurl }}/2026/09/06/prompt-engineering-for-enterprise-ai/)**
Prompts are not conversations — they are dynamically compiled business logic. The four pillars, the sandwich architecture, and three surgical cuts.

**[Context Engineering Lessons from Production]({{ site.baseurl }}/2026/09/06/context-engineering-lessons-from-production/)**
Three-layer memory, rolling context compression, the context budget problem, and why we include assistant history in the router (after learning the hard way not to).

**[Memory Systems and Token Economics for Production Agents]({{ site.baseurl }}/2026/09/06/memory-systems-and-token-economics-for-production-agents/)**
Semantic vs episodic memory, inject-cheap-write-expensive pattern, per-model token pricing, cost attribution, and when multi-agent burns 15x more tokens.

---

## Infrastructure & Security

**[Building AI Observability Without LangSmith]({{ site.baseurl }}/2026/09/06/building-ai-observability-without-langsmith/)**
On-premise means no SaaS observability. Our 8-table lineage system: trace_request → graph_run → node_run → llm_call, with ~2ms overhead per node.

**[Designing Tools for Production AI Agents]({{ site.baseurl }}/2026/09/06/designing-tools-for-production-ai-agents/)**
SkillHub marketplace, component registry as tool definition, semantic query plans as structured tool calls, and evaluation-driven tool improvement.

**[Sandboxing AI Skills in Kubernetes]({{ site.baseurl }}/2026/09/06/k8s-sandbox-security-for-ai-skill-execution/)**
YARA scanning, K8s ephemeral Jobs, NetworkPolicy isolation, prepuller DaemonSets, async long tasks, and zero-credential sandbox execution via presigned URLs.

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
