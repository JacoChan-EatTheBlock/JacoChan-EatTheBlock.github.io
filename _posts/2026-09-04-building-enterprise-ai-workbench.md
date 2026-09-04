---
layout: default
title: "Building an Enterprise AI Workbench: Six Trade-offs That Shaped Our Architecture"
date: 2026-09-04
categories: [AI Architecture, LangGraph, Enterprise]
---

# Building an Enterprise AI Workbench: Six Trade-offs That Shaped Our Architecture

*September 4, 2026 · 12 min read*

When 300 employees across sales, operations, and finance ask an AI assistant the same question — "show me last quarter's revenue by region" — you'd expect it to hit a single API and return a chart. In practice, that request touches four different backend systems, requires role-based data scoping, generates a SQL query the user never sees, renders a dashboard that persists across sessions, and stores context so the next question — "now compare it to the same quarter last year" — just works.

We built an enterprise AI workbench that handles this. One natural language entry point, four business domains (conversation, data dashboards, a skill marketplace, and external tool integration via MCP), all orchestrated by a stateful graph engine.

This post isn't a tutorial. It's a record of the six architecture decisions we wrestled with, what we gained, and what we knowingly gave up. If you're evaluating whether to build an AI platform for your organization, these trade-offs will save you months of debate.

![Figure 1. High-level architecture](/assets/images/fig1-architecture.svg)
*Figure 1. One backend, four domains, ~300 users. The layered design keeps each concern in its lane.*

---

## Trade-off 1: LangGraph Over Plain Chains

**The problem.** Our workbench isn't a chatbot. A single user turn can trigger a multi-step workflow: parse @-mentions, inject memory, compress context, execute tools in a sandbox, write back to memory, and stream partial results — all while supporting mid-flight interruption ("actually, cancel that query"). Plain LangChain chains are linear. We needed branching, looping, and the ability to pause a workflow, wait for human approval, and resume hours later.

**What we chose.** LangGraph with a hierarchical graph topology: a root graph routes by domain to sub-graphs (conversation, data, skill execution), and a reusable ReAct sub-graph handles the agent↔tool loop.

![Figure 2. Graph topology](/assets/images/fig2-graph-topology.svg)
*Figure 2. The root graph is the only place we inject a checkpointer. Sub-graphs inherit it, so `interrupt()` bubbles up correctly — a subtlety that cost us a week of debugging.*

**What we gained:**
- **Stateful interrupt and resume.** When a tool call needs human approval, the graph pauses at a checkpoint. The user approves (or not), and we resume from exactly where we stopped — even if the server restarted in between.
- **Sub-graph composition.** The ReAct loop is written once and mounted into both the conversation graph and the skill execution graph. Changes propagate everywhere.
- **Thread-level isolation.** Each conversation and each dashboard gets its own `thread_id`. Checkpoints are per-thread in MongoDB, so concurrent users never collide.

**What we accepted:**
- **Debugging is hard.** When a node deep in a sub-graph fails, the stack trace traverses graph edges, not function calls. We built an 8-table lineage system (more on that below) specifically to compensate.
- **Vendor coupling.** LangGraph is from LangChain Inc. If they pivot, we're exposed. Our mitigation: thin node functions that call domain services. The graph is glue; the logic lives in `services/`.
- **Learning curve.** New engineers need ~2 weeks to internalize graph semantics (state channels, reducers, conditional edges). We invested in internal documentation — 42 architecture docs and counting.

**The key insight.** Only the root graph holds the checkpointer. Sub-graphs compile without one and inherit the parent's at mount time. This means `interrupt()` in a deeply nested ReAct tool call correctly bubbles up to the root, pauses the entire turn, and stores the full state. We got this wrong initially — attaching checkpointers to sub-graphs caused duplicate writes and ghost threads. The single exception: a "sub-agent" mode where a spawned agent needs its own resumable thread, justified by genuinely independent state.

---

## Trade-off 2: MongoDB for Everything

**The problem.** We needed persistence for three very different workloads: LangGraph checkpoints (append-only, keyed by thread_id + checkpoint_id), long-term memory (vector-searchable, cross-session), and business data (dashboards, conversations, skills, files, billing). The textbook answer is three databases. We chose one.

**What we gained:**
- **Operational simplicity.** One connection string, one backup strategy, one team that knows the database. For a ~300-user system, this matters more than you'd think — every additional database doubles your on-call rotation complexity.
- **Transactional consistency.** When a graph run completes, the checkpoint, the conversation record, and the billing event all land in the same database. We can use MongoDB's multi-document transactions when strict consistency matters (though we rarely need to).
- **Atlas vector search.** For long-term memory retrieval, we use MongoDB Atlas's `$vectorSearch` on the same cluster where business data lives. No separate Pinecone or Qdrant to manage.

**What we accepted:**
- **No specialized performance.** A dedicated vector database would query embeddings faster. A dedicated time-series store would handle lineage more efficiently. For our scale, the difference is milliseconds, not seconds — acceptable.
- **Community edition gaps.** MongoDB Community doesn't support `$vectorSearch`. Developers on local machines must either use Atlas (cloud) or set `VECTOR_INDEX_BACKEND=fallback` (in-memory approximate search). This causes subtle "works on my machine" bugs.
- **Schema discipline is on us.** MongoDB won't stop you from writing garbage. We enforce structure through Pydantic schemas (`schemas/` directory) and treat schema violations as hard errors in tests. 759 unit tests, all green, is our schema police.

**The key insight.** We designed explicit migration ports: `StorageProvider` (MinIO → S3/OSS), `VectorIndex` (Atlas → fallback/self-hosted), `SecretProvider` (env → Vault), `model_router` (single vendor → multi-vendor). MongoDB is our default, not our destiny. The ports exist so that when we outgrow it, we can swap one layer without rewriting the application.

---

## Trade-off 3: Single-Process Monolith vs. Microservices

**The problem.** Should 300 users justify a Kubernetes-orchestrated microservice fleet? Our instinct said no.

**What we chose.** A single async Python process: FastAPI serves HTTP, LangGraph runs graphs, and services handle business logic — all in-process, all sharing the same event loop.

**What we gained:**
- **Zero inter-service latency.** A conversation turn that touches memory injection, context compression, LLM calls, and tool execution makes no network hops between services. The graph calls Python functions directly.
- **Simple deployment.** One Docker image, one `uvicorn` command. The `docker-compose.yml` has four containers: the app, MongoDB, Redis, and MinIO. That's it.
- **Easy debugging.** A stack trace shows the full call path from HTTP handler to graph node to service function to database query. No distributed tracing needed for most bugs.

**What we accepted:**
- **Vertical scaling ceiling.** If we grow from 300 to 3,000 users, one process won't be enough. We'll need to extract the graph executor into a worker pool.
- **Single point of failure.** If the process crashes, everything goes down. We mitigate this with a `resume_poller` worker that detects abandoned graph runs and replays them from the last checkpoint.
- **The ports are our escape hatch.** `TaskRunner` is designed to swap from in-process execution to `arq` + Redis workers. The interface is defined; the migration is a configuration change, not a rewrite.

**The key insight.** Premature decomposition is worse than a well-structured monolith. Our architecture document explicitly lists "migration ports" — the five seams where we'll cut when scale demands it. Knowing *where* you'll split is more valuable than splitting early.

---

## Trade-off 4: Skill Sandbox Security vs. Capability

**The problem.** Our SkillHub lets users publish and execute custom "skills" — essentially code packages that the AI agent can invoke. An agent might run a Python script that reads uploaded files, generates a report, and writes the output back. The question: how do you let arbitrary code run safely inside your infrastructure?

**What we chose.** K8s-based ephemeral sandboxes with a layered security model:

1. **YARA security scanning** before any skill is published. A dedicated scanning microservice checks for known malicious patterns.
2. **NetworkPolicy isolation.** External skills run in pods with no internet access. Internal skills (trusted, platform-authored) are allowed outbound connections.
3. **Ephemeral execution.** Each skill run gets a fresh pod. No state persists between runs. Outputs must be written to `/workspace/outputs/` and are registered as downloadable files.
4. **Async long tasks.** Skills with `mode: async` submit a K8s Job and return a `run_id` immediately, bypassing the synchronous 300-second timeout. A persistent `async_poller` monitors completion.

**What we accepted:**
- **Internal skills have internet access.** This is a deliberate privilege gap. We trust platform-authored skills more than user-authored ones. The boundary is enforced by the `origin` field in the skill manifest (internal vs. external), not by the code itself.
- **Cold-start latency.** The first invocation of a skill image takes 10-15 seconds for pod scheduling + image pull. We mitigate this with a prepuller DaemonSet that keeps popular images warm on every node.
- **Sandbox images are our responsibility.** We maintain custom Dockerfiles (`deploy/sandbox-images/`) for runtimes like "office" (LibreOffice for document conversion). These need regular security patches.

---

## Trade-off 5: Three-Layer Memory

**The problem.** Enterprise users expect the AI to remember context — not just within a conversation, but across sessions. "Last week I asked about Q2 revenue in Guangdong province" should work without repeating the filters. But cross-session memory creates privacy risks: should the AI know what a user discussed in a different business domain?

**What we chose.** A three-layer memory architecture:

| Layer | Scope | Technology | Retention |
|-------|-------|-----------|-----------|
| **Short-term** | Current thread | LangGraph checkpoint + rolling summary | Session lifetime |
| **Long-term** | Per-user, per-domain | `langmem` + MongoDB vector store | Indefinite |
| **Cross-domain** | Per-user, all domains | Same store, gated by `MEMORY_CROSS_DOMAIN` flag | Configurable |

**What we gained:**
- **Rolling context compression.** As conversations grow long, older messages are summarized into a compact digest. The LLM sees recent messages verbatim + compressed history, staying within context windows without losing important context.
- **Domain-level summaries.** After each turn, a `memory_write` node extracts key facts and stores them as vector-searchable memories. Future conversations retrieve relevant memories at the `memory_inject` step.
- **A single flag for cross-domain sharing.** `MEMORY_CROSS_DOMAIN=false` (the default) means dashboard memories never leak into conversation memories. Flipping it to `true` enables richer context — but the privacy trade-off is explicit and auditable.

**What we accepted:**
- **Privacy requires trust in the flag.** There's no fine-grained per-memory access control. It's all-or-nothing per domain. A user's dashboard data either is or isn't visible to their conversation agent.
- **Noisy recall.** Vector-based memory retrieval sometimes surfaces irrelevant context. We don't have a confidence threshold sophisticated enough to always filter noise. This occasionally wastes context window tokens.

---

## Trade-off 6: On-Premise, Domestic LLM Only

**The problem.** China's Personal Information Protection Law (PIPL) and our customers' security policies require that all data stays within domestic borders. No data can flow to OpenAI, Anthropic, or any foreign cloud.

**What we chose.** A fully on-premise deployment with ZhiPu GLM as the primary LLM, behind a `model_router` abstraction.

**What we gained:**
- **Data sovereignty.** Customer data never crosses borders. This is non-negotiable for our enterprise clients in finance and manufacturing.
- **Customer trust.** Being able to say "your data never leaves China, and we can prove it" closed deals that cloud-based competitors couldn't.

**What we accepted:**
- **No LangSmith, no Weights & Biases.** These tools assume cloud access. We built our own observability stack: an 8-table lineage system tracking every request → graph run → node run → LLM call → tool execution, with full trace linking and event capture.
- **Limited model choice.** When GPT-4o or Claude 3.5 release capabilities our domestic LLM lacks, we can't use them. The `model_router` is designed for multi-provider support (the interface exists), but regulatory reality limits us to domestic options.
- **We built what others buy.** Our `monitoring/` directory has 14 sub-modules covering audit, events, usage, metrics, graph observability, LLM observability, prompt observability, skill observability, and lineage. Each of these exists because we couldn't use a SaaS tool.

---

## The Observability Problem (and Our 8-Table Solution)

When you can't use LangSmith and your graphs are 15+ nodes deep with sub-graph nesting, you need visibility. We built an 8-table lineage system:

```
trace_request → graph_run → node_run → skill_run
                                      → llm_call
              + trace_link (cross-graph references)
              + trace_event (arbitrary structured events)
```

Every LLM call records: model, prompt hash, token count, latency, cost estimate. Every node records: input state hash, output state hash, duration, error (if any). Every skill run records: image, duration, exit code, output files.

The total overhead is ~2ms per node (async write to MongoDB). For a typical 8-node conversation turn, that's 16ms — invisible to the user, invaluable for debugging.

---

## What We'd Do Differently

No architecture survives contact with reality unchanged. Here's our honest retrospective:

1. **Start with the lineage system on day one.** We added observability in month three. The debugging pain in months one and two was severe. If you're building a graph-based system, instrument it before you write the first node.

2. **Don't under-invest in context compression.** Our initial approach was simple truncation — cut messages beyond a token limit. The quality difference when we switched to LLM-summarized rolling compression was dramatic. It should have been the first memory feature, not the third.

3. **Schema validation should be in the graph, not just in tests.** We use `llmproxy_client.complete_typed(schema_model=...)` to force structured LLM outputs. This catches hallucinated field names and wrong types at generation time, not at consumption time. We wish we'd adopted this pattern from the start instead of debugging malformed JSON in production.

4. **The monolith was the right call — for now.** At 300 users, we have zero scaling issues. The migration ports give us confidence that we can decompose when needed. We've seen teams of similar size spend 6 months building a microservice architecture that serves fewer users with more operational pain.

---

## Closing Thoughts

Building an enterprise AI workbench is less about picking the "best" technology and more about making defensible trade-offs. LangGraph gave us statefulness at the cost of complexity. MongoDB gave us simplicity at the cost of specialization. A monolith gave us speed at the cost of scale. On-premise gave us trust at the cost of tooling.

None of these choices are permanent. The migration ports ensure that. What matters is that each choice was deliberate, documented, and reversible.

If your organization is evaluating whether to build or buy an AI platform, the most important question isn't "what tech stack should we use?" It's "what trade-offs can we live with — and which ones will we need to escape from later?"

![Figure 3. Trade-off matrix](/assets/images/fig3-tradeoffs.svg)
*Figure 3. Six decisions, what we gained, what we accepted. No architecture is without cost.*

---

*The system described serves ~300 enterprise users across four business domains. All data remains on-premise in compliance with China's PIPL. The architecture has been in production since mid-2026.*
