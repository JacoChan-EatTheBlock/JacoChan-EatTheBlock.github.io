---
layout: default
title: "How I Built an Enterprise AI Platform from Scratch in 2 Months"
date: 2026-09-04
categories: [AI Architecture, Engineering, Startup]
---

# How I Built an Enterprise AI Platform from Scratch in 2 Months

*September 4, 2026 · 14 min read*

Two months. One engineer. A backend serving ~300 enterprise users across four business domains — conversation AI, data dashboards, a skill marketplace, and external tool integration. No boilerplate to start from, no team to delegate to, and a hard deadline.

This is not a humble-brag. This is a post-mortem of what actually happened: the shortcuts that saved time, the decisions that created debt, and the three things I'd change if I did it again. If you've ever been asked to "just build an AI platform," this might be the most honest account you'll find.

---

## Week 0: Scoping Under Pressure

The ask was deceptively simple: "We need an internal AI assistant that can answer questions about our business data." Within two days of requirements gathering, "answer questions" expanded to:

- **Conversation**: Multi-turn chat with memory, @-mentions, file understanding, and tool use
- **Data Dashboards**: Natural language → MongoDB aggregation pipeline → interactive charts, persisted and shareable
- **Skill Marketplace**: Users publish and execute custom code packages (Python scripts, document processors) inside the platform
- **MCP Integration**: Connect to external tools via the Model Context Protocol

Four products, not one. The instinct to say "let's prioritize" was overruled by business reality — all four were promised to internal stakeholders. The only negotiable variable was polish.

**Decision #1: Bet on a graph engine, not a chain.** A chatbot could use LangChain's sequential chains. Four interleaved domains with shared state, interrupt/resume, and sub-workflows could not. I chose LangGraph on day one, accepting the learning curve tax, because the alternative — manually managing state machines across four domains — would have been slower by week three.

**Decision #2: One database for everything.** MongoDB for LangGraph checkpoints, business data, long-term memory, file metadata, billing events, and observability. One connection string, one backup, one set of indexes to tune. The "right" architecture (Postgres for relational, Pinecone for vectors, Redis for cache, S3 for files) would have taken a week just to set up the infrastructure.

**Decision #3: Monolith with escape hatches.** One async Python process. FastAPI serves HTTP, LangGraph runs graphs, services handle logic — all in-process. I defined five explicit "migration ports" where we'd split later: `StorageProvider`, `VectorIndex`, `SecretProvider`, `TaskRunner`, `model_router`. The ports exist as interfaces; the implementations are in-process calls. When scale demands decomposition, we change configuration, not architecture.

---

## Week 1–2: The Skeleton

**Days 1–3: Project structure.** I spent the first three days on directory layout, not features. This sounds wasteful. It wasn't.

```
apps/           → FastAPI routes (thin controllers)
orchestrator/   → LangGraph graphs, nodes, routers, state
services/       → shared/ + domain/ + infra/ (all business logic)
schemas/        → Pydantic models (the schema police)
configs/        → settings, prompts, bootstrap
monitoring/     → 8-table lineage system
```

The rule: **routes don't contain logic, nodes don't contain queries, services don't know about HTTP.** Every layer talks only to the one below it. This took discipline to enforce, but by week six, a new engineer could trace any request from endpoint to database in under five minutes.

**Days 4–7: Root graph and domain routing.** The `root_graph` is the entry point for every user request. It does three things:

1. Authenticates and loads user context
2. Routes to the correct domain graph (conversation, data, skill, MCP) based on intent classification
3. Holds the only checkpointer in the system

The last point is critical. Sub-graphs don't get their own checkpointers — they inherit the root's. This means `interrupt()` in a deeply nested tool call bubbles up correctly. I got this wrong on day 5 (gave the conversation sub-graph its own checkpointer), spent 8 hours debugging phantom state, and learned the hard way.

**Days 8–14: Conversation graph.** The conversation pipeline has six nodes:

```
prepare → memory_inject → mention_parse → context_compress → react_subgraph → memory_write
```

Each node is a pure function: takes state, returns state delta. The `react_subgraph` is a standard ReAct loop (agent ↔ tools) with human-in-the-loop interrupt. Writing it as a sub-graph meant I could reuse it in the skill execution graph later without copying code.

The `context_compress` node was initially a simple truncation — cut messages beyond 4,000 tokens. This "worked" for demos. In production, users noticed the AI forgot things they said three messages ago. Replacing truncation with LLM-summarized rolling compression (summarize older messages into a digest, keep recent ones verbatim) was a weekend rewrite that should have been in the first version.

---

## Week 3–4: The Data Engine

This is where most of the complexity lives. A user types "show me contract count by region for Q2" and the system must:

1. **Classify intent** — is this a new dashboard, a filter on an existing one, an export, or a clarification?
2. **Resolve semantics** — "contract count" maps to which MongoDB collection and which aggregation? "Region" maps to which field? "Q2" maps to which date range?
3. **Generate a query plan** — a structured plan with collection, pipeline stages, chart type, and data mappings
4. **Validate** — is the pipeline safe? Do the mapped fields actually exist in the output? Does the chart type match the data shape?
5. **Execute and persist** — run the query, render the chart, save the dashboard

### The Semantic Layer Problem

The hardest part wasn't generating MongoDB pipelines. It was **mapping user vocabulary to data vocabulary.** Users say "active contracts"; the database has `status: "启用"`. Users say "success rate"; the database has separate numerator and denominator fields that need to be divided.

Our first approach was a metric catalog — a curated dictionary of every business metric with its exact MQL definition. The AI would pick a metric ID, and we'd look up the query. This failed spectacularly:

- **Wrong pick = wrong answer.** If the AI chose `contract_count` instead of `active_contract_count`, the query was syntactically perfect but semantically wrong. Users trusted the number because the chart looked professional.
- **Catalog maintenance was a full-time job.** Every new metric, every renamed field, every schema change required a catalog update. The catalog was always stale.

We replaced it with a **few-shot example store** — a library of human-approved (question, query) pairs. Instead of telling the AI "here's the definition of 'active contract count'," we show it: "someone asked 'how many active contracts?' and the approved answer was this pipeline." The AI writes its own query, guided by precedent rather than dictated by lookup.

The example store has a two-stage approval workflow (draft → pending → published). Only published examples enter the AI's context. AI-generated queries that run successfully are saved as drafts, but they **never** auto-promote to published — a human must verify the business logic. This is the "anti-Vanna" rule: just because a query returns data doesn't mean it returns the *right* data.

### The Chinese NLP Scoring Problem

Standard substring matching fails for Chinese business terms. "启用中合同数占比" (active contract ratio) contains the characters for "启用合同数" (active contract count), but they're not a substring — there's an extra "中" in between. A naive `term in question` check misses it.

We built a bigram coverage scorer: decompose both the query and the term into character pairs, measure what fraction of the term's bigrams appear in the query. This handles character insertion, reordering, and synonym substitution without maintaining an ever-growing alias list.

```
term_score("启用中合同数占比", "启用合同数")
→ bigrams of term: {启用, 用合, 合同, 同数}
→ bigrams of query: {启用, 用中, 中合, 合同, 同数, 数占, 占比}
→ coverage: 4/4 = 1.0 → high score
```

Only exact matches (after normalization) are treated as definitive. Everything else is a scored candidate. This distinction matters: a definitive match can short-circuit retrieval; a scored candidate is one of several options the LLM must choose between.

---

## Week 5–6: Skills and Security

### The Sandbox Problem

The skill marketplace lets users upload Python packages that the AI agent can execute. This is inherently dangerous. Our security model has three layers:

1. **YARA scanning** on publish. Before a skill enters the marketplace, a scanning service checks for known malicious patterns.
2. **K8s ephemeral pods** on execution. Each skill run gets a fresh container. No persistent state between runs. Outputs go to `/workspace/outputs/`.
3. **NetworkPolicy** by trust level. User-authored skills: no internet. Platform-authored skills: internet allowed.

The cold-start problem was real — first invocation took 10–15 seconds for pod scheduling and image pull. We added a prepuller DaemonSet that keeps popular images warm. Subsequent invocations start in under 2 seconds.

### Async Long Tasks

Some skills run for minutes (document batch processing, data migration). The synchronous request timeout is 300 seconds — not enough. We added an async mode:

1. Skill declares `mode: async` in its manifest
2. Execution submits a K8s Job, returns a `run_id` immediately
3. An `async_poller` background task monitors the Job and writes results to MongoDB
4. The frontend polls by `run_id` until completion

This pattern — submit, poll, retrieve — is boring and reliable. We considered WebSocket push but decided the frontend polling complexity wasn't justified for <50 concurrent async tasks.

---

## Week 7–8: Observability and Hardening

### The 8-Table Lineage System

Without LangSmith (on-premise, remember), debugging a 15-node graph with sub-graph nesting is pain. We built our own:

| Table | What It Tracks |
|-------|---------------|
| `trace_request` | HTTP request → graph run mapping |
| `graph_run` | Graph ID, thread ID, start/end, status |
| `node_run` | Each node: input hash, output hash, duration, error |
| `llm_call` | Model, prompt hash, tokens, latency, cost estimate |
| `skill_run` | Image, duration, exit code, output files |
| `trace_link` | Cross-graph references (parent → child) |
| `trace_event` | Arbitrary structured events (business-level) |
| `usage` | Per-user, per-model token/cost aggregation |

The total overhead is ~2ms per node (async write). For a typical 8-node turn, that's 16ms — invisible to users.

**What I'd change:** Build this on day one, not week seven. The debugging pain in weeks three through six — when the data graph was producing wrong charts and I couldn't trace why — was the worst period of the project. Most of those bugs were "the right node ran but with wrong input state," which is trivial to diagnose with a lineage trace and nearly impossible without one.

### The 759 Tests

Testing an AI system is different from testing a deterministic one. You can't assert that "contract count by region" produces exactly `[{region: "East", count: 42}]` because the AI might generate a slightly different pipeline that returns the same data.

Our testing strategy:

- **Unit tests (schema validation):** Every Pydantic model has tests that verify required fields, type constraints, and serialization roundtrips. This catches 80% of "the AI generated malformed JSON" bugs.
- **Integration tests (graph execution):** Run a graph with mocked LLM responses, verify that the correct nodes execute in the correct order with the correct state transitions.
- **E2E calibration tests:** Run real user queries against real data with real LLMs. Don't assert exact answers — assert structural properties: "the chart has data," "the pipeline is valid MQL," "the mapped fields exist in the output."
- **Regression tests (example store):** Every approved example becomes a test case. If a model upgrade causes a previously-correct query to regress, we catch it.

---

## The Three Things I'd Change

### 1. Context Compression from Day One

The jump from truncation to rolling summarization was a 3x improvement in multi-turn coherence. It should have been the first memory feature, not the third. If you're building any multi-turn AI system, start with summarization — not because it's hard (it's ~100 lines of code) but because the quality difference is so dramatic that everything you build on top of it will work better.

### 2. Structured Output Everywhere

We adopted `complete_typed(schema_model=PydanticModel)` — forcing the LLM to return JSON conforming to a Pydantic schema — midway through the project. Before that, we'd generate free-form text and parse it with regex. The regex approach "worked" until it didn't: the LLM would add a markdown code fence around JSON, or use single quotes instead of double quotes, or include a helpful explanation before the JSON blob.

Structured output catches these issues at generation time. The LLM is constrained to produce valid JSON matching the schema. Invalid fields are rejected before they propagate. If I started over, every LLM call that expects structured data would use `complete_typed` from the first commit.

### 3. The Lineage System on Day One

I said this above but it bears repeating. An AI platform without request-level tracing is like a web server without access logs — you can build it, but you can't debug it. The 8-table system added ~400 lines of code. The debugging time it would have saved in the first six weeks was easily 40+ hours.

---

## The Uncomfortable Truth About "From Scratch"

Building a platform "from scratch in 2 months" sounds impressive. Here's the honest context:

- **It wasn't from zero knowledge.** I'd spent years working with Python async, MongoDB, and API design. The AI-specific components (LangGraph, embeddings, prompt engineering) were new, but the infrastructure patterns were muscle memory.
- **It wasn't production-perfect.** The first version had no rate limiting, no graceful degradation when the LLM was down, and a context compression algorithm that occasionally summarized away critical details. These were fixed in months three and four.
- **It scaled to 300 users because 300 users is small.** The monolith works because we're not at 3,000. The migration ports exist because we know we'll get there. But right now, honest engineering means admitting that a single async process is more than enough.
- **The 42 internal docs were written after the code.** Documentation happened in weeks seven and eight, not during development. The cost was paid by every engineer who joined after me and had to read code instead of docs for their first week.

The real skill wasn't building fast. It was **knowing which shortcuts would create manageable debt and which would create catastrophic debt.** MongoDB for everything? Manageable — the migration ports exist. Skipping the lineage system? Nearly catastrophic — we almost shipped a version that produced wrong financial numbers because we couldn't trace the pipeline that generated them.

---

## For the CTO Evaluating "Build vs. Buy"

If your team is considering building an internal AI platform, here's what I wish someone had told me:

1. **The AI part is 30% of the work.** The other 70% is authentication, authorization, observability, error handling, file management, deployment, testing, and documentation. If your plan allocates engineer-months only for "the AI stuff," double your estimate.

2. **Your first version will be wrong.** Not slightly wrong — structurally wrong in at least one major dimension. Design for replaceability, not perfection. Migration ports > beautiful abstractions.

3. **Structured output is non-negotiable.** Every LLM call that produces data (not prose) must be schema-constrained. The hour you spend defining Pydantic models saves the week you'd spend debugging malformed JSON in production.

4. **Observability before features.** Build the tracing system before the second feature. You will need to debug "why did the AI do that?" before the first user complaint.

5. **Start with one engineer, not a team.** A single engineer who owns the entire stack makes faster architectural decisions than a team that needs consensus. Add people after the skeleton is solid, not before.

The platform now serves ~300 users, runs on-premise in compliance with PIPL, and has been stable since launch. It started as a two-month sprint by one person. Whether that's impressive or reckless depends on which shortcuts you notice — and whether the migration ports ever get used.

---

*All data remains on-premise. No code or internal identifiers are shared in this post. Architecture patterns are described generically.*
