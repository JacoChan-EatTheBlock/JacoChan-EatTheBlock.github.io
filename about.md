---
layout: default
title: About
---

# About

I'm JacoChen, an AI Architect who builds enterprise AI platforms from the ground up.

## What I've Built

An enterprise AI workbench serving ~300 users across four business domains — conversation AI, natural language data dashboards, a skill marketplace, and external tool integration via MCP. Single-process monolith, LangGraph orchestration, MongoDB for everything, deployed on-premise in compliance with China's PIPL.

The system went through three major architecture rewrites in eight months. The hallucination rate dropped from 35% to under 7%. The blog posts on this site document the trade-offs, failures, and design decisions from that journey.

## Technical Depth

- **LangGraph**: Hierarchical graph topology with sub-graph composition. Root graph holds the only checkpointer. ReAct sub-graphs inherit and bubble up interrupts correctly.
- **Semantic Layer**: MongoDB views with `$set` (additive collapse, not destructive `$project`). Boolean labels, unified time fields, pre-computed amounts. 3 views replace 4 physical tables.
- **Intent System**: 11-intent catalog as single source of truth. Vector shortcircuit (37%, ~200ms) + LLM with few-shot fallback (63%). Slot guards, evidence-based write permissions, compound intent decomposition.
- **Anti-Hallucination**: 7-layer defense — structured output, few-shot anchoring, bigram scoring, dual-use routing, 6-gate validation, ratio/count disambiguation, nightly calibration.
- **Observability**: 8-table lineage system (trace_request → graph_run → node_run → llm_call → skill_run + trace_link + trace_event + usage). ~2ms overhead per node.

## Background

Full-stack AI engineer. NAU Online Master in Data Analytics. Experienced in Python async, MongoDB, FastAPI, Kubernetes, and LLM application architecture.

## Contact

- GitHub: [JacoChan-EatTheBlock](https://github.com/JacoChan-EatTheBlock)
- Email: duongvantrang38716@gmail.com
