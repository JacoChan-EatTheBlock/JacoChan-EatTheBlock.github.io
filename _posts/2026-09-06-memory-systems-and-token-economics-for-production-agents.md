---
layout: default
title: "Memory Systems and Token Economics for Production AI Agents"
date: 2026-09-06
categories: [AI Architecture, Memory, Token Economics, LLM]
---

# Memory Systems and Token Economics for Production AI Agents

*September 6, 2026 · 25 min read*

**Key Takeaways**

- Memory in production AI agents is not one system — it's three layers (short-term, long-term, cross-domain) with fundamentally different read/write economics. The read path must be cheap enough to run on every turn, including greetings. The write path can be expensive because it runs asynchronously.
- Token economics determines whether your AI agent is commercially viable. We track every LLM call's cost through a dual-write system (lineage for debugging, ledger for accounting), and we've built five token budget strategies that save ~40% of inference costs without sacrificing quality.
- Long-running agents face the "shift change" problem: each new context window starts with no memory of what came before. Our solution parallels Anthropic's initializer/coding agent pattern — preset dashboards as initialization, intent-driven incremental modifications as coding sessions, and a lineage system as the progress file.

---

## The Memory Problem in Enterprise AI

When a user opens our AI workbench on Monday morning and says "show me the same dashboard I was looking at on Friday," the system needs to know what "Friday's dashboard" was. When they say "I usually care about contract counts, not amounts," the system should remember this preference across sessions. When they switch from the conversation domain to the data dashboard domain, some context should transfer and some shouldn't.

These are three different memory problems requiring three different solutions. Most tutorials treat memory as a single vector database. In production, that approach either costs too much (vector search on every turn) or remembers too little (only storing what fits in the embedding model's context).

We built a three-layer memory system serving ~300 enterprise users across four business domains. This post describes what we built, the economics that drove each design decision, and how token costs shaped the architecture from the ground up.

![Figure 1. Three-Layer Memory Architecture](/assets/images/fig20-memory-architecture.svg)
*Figure 1. The read path (left, green) runs on every turn and costs ~2ms with no LLM involvement. The write path (right, orange) runs in the background after each turn and involves LLM distillation. The cross-domain gate (bottom, red) controls whether memories leak between business domains.*

---

## Layer 1: Short-Term Memory (Checkpoint + Rolling Summary)

### The problem with unlimited context

LangGraph's `MongoDBSaver` checkpointer stores the full conversation history per `thread_id`. In theory, this is all the short-term memory you need — the model sees every previous message on every turn. In practice, conversations with enterprise users get long. A data analyst asking iterative questions about contract billing might generate 40+ messages in a session, easily exceeding 8,000 tokens of history.

The naive solution is truncation: keep the most recent N messages, drop the rest. We shipped this in V1 and discovered why it fails:

- **Message 5**: "Show me contract count by insurance type for July"
- **Message 12**: "Now break that down by region"  
- **Message 18**: "What about the previous quarter?"
- **Message 35**: "Go back to the insurance type breakdown"

With truncation at 20 messages, message 5 is gone. The system no longer knows what "the insurance type breakdown" refers to. The user gets a confused response asking them to clarify something they said 30 messages ago.

### Rolling compression: the LibreChat pattern

We replaced truncation with rolling compression, adapted from LibreChat's dual approach: `getMessagesWithinTokenLimit` (synchronous trim) + `summarizeMessages` (LLM-powered summary, only for overflow).

The critical insight is **separating the two operations in time**:

```
User sends message
    │
    ▼
context_compress (sync, every turn)
    ├── trim(): enforce token budget
    │   Keep recent messages verbatim (configurable: last 6-10)
    │   └── Overflow messages → background task
    │
    ├── This turn: use EXISTING summary (from previous turn's background task)
    │
    └── Background: summarize overflow → write to threads.summary
        Available for NEXT turn (never blocks current response)
```

The user never waits for summarization. This turn's response uses whatever summary exists from the last time overflow occurred. The background task updates the summary for next time.

```python
async def compress(state):
    messages = state.get("messages") or []
    existing_summary = await conversation_service.get_summary(thread_id)
    
    # Sync: trim to budget (fast, deterministic)
    kept, overflow = summary_service.trim(
        messages, existing_summary=existing_summary,
        max_tokens=settings.chat_max_context_tokens,
        keep_recent=settings.chat_keep_recent_messages,
    )
    
    # Async: summarize overflow in background (slow, LLM-powered)
    if overflow:
        task = asyncio.create_task(
            _bg_summarize(thread_id, existing_summary, overflow, model))
        _bg_tasks.add(task)  # prevent GC
        task.add_done_callback(_bg_tasks.discard)
    
    return {"messages": kept, "summary": existing_summary}
```

### What the model actually sees

After compression, a typical turn's context looks like:

```
[Summary of messages 1-25]:
  "User is analyzing July contract data. They've explored 
   insurance type breakdowns, regional distribution, and 
   quarterly comparisons. Current focus: insurance type trends."

[Message 26]: assistant: "Here's the quarterly comparison..."
[Message 27]: user: "Go back to the insurance type breakdown"
[Message 28]: assistant: "Switching back to insurance type view..."
...
[Message 35]: user: "Show me the same thing for August"
```

Recent messages are verbatim. Older context is compressed into a digest. The model can resolve "the same thing" by reading both the summary and recent messages.

**The quality difference was dramatic.** Multi-turn coherence — the ability to correctly resolve references to earlier parts of the conversation — improved from ~62% to ~89% when we switched from truncation to rolling summary. This should have been the first memory feature we built, not the third.

---

## Layer 2: Long-Term Memory (Semantic + Episodic)

### Two kinds of memory in one collection

Short-term memory (Layer 1) is per-session — it disappears when the user starts a new conversation. Long-term memory persists across sessions. We store both kinds in a single MongoDB `memories` collection, distinguished by a `kind` field:

| Kind | What It Stores | Write Mode | Example |
|------|---------------|------------|---------|
| `semantic` | Stable facts about the user (identity, preferences, domain knowledge) | **Upsert** — same `(user, domain, key)` overwrites | "User is a backend engineer who prefers monthly reports" |
| `episodic` | Events that happened (decisions, notable interactions) | **Append-only** with timestamps | "On 2026-07-15, user asked about Q2 contract churn" |

This distinction matters because semantic and episodic memories have different lifecycle requirements. A user's job title should be updated (upsert) when it changes. But the fact that they asked about Q2 churn on a specific date should never be overwritten — it's an event, not a fact.

### The asymmetric read/write design

The most important architectural decision in our memory system is the **asymmetry between read and write**:

**Read (inject, every turn): cheap, no LLM, no vector search.**

```python
async def recall_context(user_id, domain=None):
    # Just a MongoDB query — load top-N semantic memories by confidence score
    rows = db["memories"].find(
        {"user_id": user_id, "kind": "semantic", "domain": domain},
        sort=[("confidence", -1)],
        limit=settings.memory_inject_max
    )
    return [f"[{category_label(r['category'])}] {r['text']}" for r in rows]
```

This runs on every turn, including "hello." It costs ~2ms (one MongoDB query) and returns formatted profile lines like:

```
[identity] User is a contract operations manager
[preference] Prefers monthly granularity over quarterly
[domain knowledge] Their company uses "启用" status for active contracts
```

No embedding, no vector search, no LLM involvement. The cost is essentially zero.

**Write (extract, after each turn): expensive, LLM-powered, async background.**

```python
async def extract_and_store(user_id, *, user_text, answer, model, domain, ...):
    # LLM distills conversation into structured memories
    extraction = await llmproxy_client.complete_typed(
        model, messages, schema_model=MemoryExtraction, temperature=0.0)
    
    for fact in extraction.facts:
        if fact.confidence < 0.6:
            continue  # Low confidence → discard
        await _upsert_semantic(user_id, domain, fact)
    
    for event in extraction.events:
        await _append_episodic(user_id, domain, event)
```

This runs in a background task after the response has already been sent to the user. The LLM uses `complete_typed` with a Pydantic schema that locks categories to an enum (identity, preference, domain_knowledge, workflow, other) and requires confidence scores. This prevents hallucinated or noisy memories from entering the store.

**Why not vector search for injection?** Because our long-term memory is small and structured. A user's semantic profile is typically 5-15 facts. Loading all of them sorted by confidence (a database index operation) is faster and more reliable than embedding the current question and doing approximate nearest neighbor search. Vector search via `recall()` exists as a reserved interface for a future `memory_search` tool — but for routine injection, a sorted query wins.

### The confidence guard

LLM distillation is inherently noisy. Without guardrails, the system would store "memories" like:

- `"User seems interested in data"` (too vague to be useful)
- `"User wants to see charts"` (trivially true of everyone)
- `"User's contract count is 1,247"` (a data point, not a user fact)

We guard against this with two mechanisms:

1. **Enum-locked categories**: The extraction schema requires each fact to be categorized (identity/preference/domain_knowledge/workflow/other). The LLM can't create a memory that doesn't fit any category.

2. **Confidence threshold**: Facts with `confidence < 0.6` are silently discarded. The threshold is conservative — we'd rather miss a real fact than store a hallucinated one.

3. **Upsert by key**: Semantic memories have a compound key `(user_id, domain, category, key)`. A new fact about the user's job title overwrites the old one, preventing duplicate proliferation.

---

## Layer 3: Cross-Domain Memory Gate

### The privacy trade-off

Our workbench has four business domains: conversation, data dashboards, skill marketplace, and MCP integration. Each domain generates different kinds of user context. Should the dashboard domain's memory ("user prefers monthly granularity") be visible to the conversation domain?

The answer depends on your privacy requirements. We implemented it as a single configuration flag:

```python
# settings.py
memory_cross_domain: bool = False  # DEFAULT: isolated per domain
```

**`MEMORY_CROSS_DOMAIN = false` (default):** Each domain only sees its own memories. Dashboard memories never leak into conversation context. This is safer — a user's data analysis preferences shouldn't influence how the conversation agent responds to general questions.

**`MEMORY_CROSS_DOMAIN = true`:** All semantic memories for a user are shared across domains. This enables richer context — the conversation agent knows the user is a contract operations manager because the dashboard domain learned this. But it also means domain-specific noise can pollute other domains.

We default to `false` because the privacy risk is real and the benefit is marginal for our use case. The flag exists so that if a deployment needs cross-domain context (e.g., a unified executive assistant), it's a configuration change, not an architecture change.

---

## Token Economics: Where the Money Goes

Memory architecture determines quality. Token economics determines whether you can afford that quality at scale. Every design decision above was shaped by cost.

![Figure 2. Token Economics — Cost Attribution & Budget Strategy](/assets/images/fig21-token-economics.svg)
*Figure 2. Left: Every LLM call dual-writes to a lineage table (for debugging) and a usage ledger (for accounting). Right: Five token budget strategies that collectively save ~40% of inference costs.*

### The dual-write cost tracking system

Every LLM call in our system is recorded twice:

1. **`llm_call` table** (lineage): Model name, provider, prompt hash, token counts, latency, cost estimate, status. Linked to `trace_id → graph_run_id → node_run_id`. Used for debugging — "why did the AI do that?"

2. **`usage_ledger` table** (accounting): Same token/cost data, but linked to `user_id`, `workspace_id`, and `prompt_id`. Used for cost reporting — "who spent how much on what?"

```python
async def record_call(*, model_name, usage, latency_ms, prompt_id, ...):
    req, resp, total = _parse_usage(usage)  # Handles both OpenAI AND LangChain formats
    cost = token_ledger.compute_cost(model_name, req, resp)
    
    # Write 1: lineage (debugging)
    llm_call_id = await lineage_writer.record_llm_call(
        model_name=model_name, request_tokens=req, response_tokens=resp,
        cost_usd=cost, latency_ms=latency_ms, prompt_id=prompt_id, ...)
    
    # Write 2: ledger (accounting)
    await token_ledger.record(
        model=model_name, request_tokens=req, response_tokens=resp,
        cost=cost, llm_call_id=llm_call_id, ...)
```

A subtle but important detail: `_parse_usage()` handles **two different field name conventions**:

```python
def _parse_usage(usage):
    # OpenAI-compatible: prompt_tokens / completion_tokens
    # LangChain usage_metadata: input_tokens / output_tokens
    req = int(usage.get("prompt_tokens") or usage.get("input_tokens") or 0)
    resp = int(usage.get("completion_tokens") or usage.get("output_tokens") or 0)
    return req, resp, req + resp
```

We discovered this the hard way: our semantic query subgraph uses LangChain's `build_chat_model().ainvoke()`, which returns `input_tokens/output_tokens`. The main graph uses our own `llmproxy_client.complete()`, which returns OpenAI-style `prompt_tokens/completion_tokens`. Without handling both, the subgraph's token costs were recorded as 0/0 (the total was correct via `total_tokens`, but the input/output split — which tells you "is the prompt too long?" — was lost).

### Per-model pricing

Our on-premise deployment uses ZhiPu GLM models with tiered pricing:

| Model | Input (per 1K) | Output (per 1K) | When We Use It |
|-------|---------------|-----------------|---------------|
| GLM-4.6 | ¥0.002 | ¥0.002 | Complex generation (dashboard layouts, data analysis) |
| GLM-4-flash | FREE | FREE | Intent routing, classification (high volume, low complexity) |
| GLM-4-air | ¥0.0005 | ¥0.0005 | Memory distillation, summarization |

The `model_router` selects the model based on the task's `ModelRole`:

- `GENERAL_FAST`: Intent routing, filter extraction → GLM-4-flash (free)
- `GENERAL`: Dashboard generation, data narration → GLM-4.6
- Memory distillation → GLM-4-air (cheap, sufficient quality)

This tiered approach means ~60% of our LLM calls (intent routing, classification) cost nothing, and only the high-value calls (generation, analysis) incur costs.

### Five token budget strategies

Beyond model selection, we've implemented five strategies that collectively reduce token consumption by ~40%:

**Strategy 1: Vector shortcircuit (saves ~37% of intent routing tokens)**

When the intent router's vector search returns a high-confidence match (cosine > 0.92, top-3 unanimous, no required slots), we skip the LLM entirely. This handles 37% of intent classifications at ~0 tokens and ~200ms, compared to ~500 tokens and ~700ms for LLM classification.

**Strategy 2: Selective few-shot injection (saves ~60% of prompt tokens vs. full injection)**

Our intent corpus has ~170 labeled utterances across 11 intents. Injecting all of them into every classification prompt would add ~2,000 tokens. Instead, we inject only the top-k vector-retrieved examples (typically 3-5), adding ~200 tokens. Same accuracy improvement, 90% fewer tokens.

This is where we diverged from RAGFlow's approach. RAGFlow's `CategorizeParam.update_prompt()` injects all examples because it typically has only 3-5 categories with ~3 examples each. We have 11 intents × ~15 examples = 170 utterances. Full injection would dilute attention; selective injection preserves it.

**Strategy 3: Rolling summary instead of full history (caps context growth)**

Without compression, a 40-message conversation consumes ~6,000 tokens of history. With rolling summary, it's capped at `max_context_tokens` (configurable, typically 4,000) regardless of conversation length. The quality trade-off is minimal — summarized older messages retain key facts even if exact phrasing is lost.

**Strategy 4: Natural language conditions (saves ~200 tokens per router call)**

When a user says "only show me bills from Shanghai Real Estate Company," the intent router needs to capture this filter. The naive approach: include a full field whitelist (~hundreds of lines: field names, types, allowed values) in every router call so the LLM can extract structured conditions. This adds ~200+ tokens to every intent routing call — even for greetings and chitchat.

Our approach: the router captures the filter as natural language (`conditions_text: "only bills from Shanghai Real Estate Company"`). The downstream `apply_filter` node — which runs only when the intent is actually `filter_change` — gets the full whitelist and translates the natural language into structured conditions. 90%+ of router calls never need the whitelist.

**Strategy 5: Semantic IR instead of raw MQL (smaller output tokens)**

The LLM writes a `SemanticQueryPlan` (structured IR: view name, metric, dimension, filters) instead of raw MongoDB aggregation pipeline JSON. A typical plan is ~150 tokens; a raw pipeline for the same query would be ~400-600 tokens. The deterministic compiler translates the plan into MQL with zero token cost.

### The compounding effect

These strategies interact multiplicatively:

```
Baseline: 100% token cost (every call hits LLM, full context, raw MQL)

After Strategy 1: 63% (37% shortcircuit at 0 tokens)
After Strategy 2: 55% (selective few-shot saves ~60% on remaining 63%)
After Strategy 3: 48% (capped context vs. unbounded growth)
After Strategy 4: 45% (no whitelist in non-filter calls)
After Strategy 5: 40% (smaller output tokens for query generation)

Net: ~40% token reduction vs. naive implementation
```

At 200 data queries/day across 300 users, the savings are not trivial — they determine whether the system is economically sustainable on domestic LLM pricing.

---

## Long-Running Agent Parallels

### The shift change problem

Anthropic's blog on effective harnesses for long-running agents describes a fundamental challenge: agents working in discrete sessions, where each new session begins with no memory of what came before. Their solution uses two components:

1. **Initializer agent**: Sets up the environment on the first run (feature list, init.sh, initial git commit)
2. **Coding agent**: Makes incremental progress each session, commits to git, updates `claude-progress.txt`

Our dashboard system faces an analogous problem and arrived at a similar two-part solution:

| Anthropic's Pattern | Our Dashboard Equivalent |
|--------------------|-----------------------|
| Initializer agent (first run) | **Preset dashboards** — curated layouts with verified queries, served as the starting state |
| Coding agent (incremental) | **Intent-driven modifications** — each user turn modifies one aspect of the dashboard (add a card, change a filter, reorder) |
| `claude-progress.txt` | **Lineage system** — 8-table trace of every request, graph run, node run, and LLM call |
| Git commits | **Board snapshots** — layout + runtime_refs persisted after each modification; data is NOT persisted (re-queried live) |
| Git revert | **restore_preset** intent — user can reset to the initialized state at any time |
| Feature list | **Component registry** — defines what chart types exist and what props they require |
| `resume_poller` | **resume_poller** — detects abandoned graph runs (server crashed mid-turn) and replays from last checkpoint |

The deepest parallel is the anti-pattern both systems guard against: **trying to do too much at once.** Anthropic's agent tried to one-shot the entire app; our V1 LLM tried to generate the entire dashboard in a single pass. Both failed because the output space was too large for reliable single-shot generation.

The fix in both cases was **incremental modification with clean state between steps**. Anthropic's coding agent works on one feature at a time and commits. Our intent system handles one intent per turn (or decomposes compound intents into a code-orchestrated DAG where each task is independently validated).

### The preset as initialization

Preset dashboards serve the same role as Anthropic's initializer agent: they establish a known-good starting state that subsequent modifications can build upon. A preset contains:

- A curated layout (which components, in what order, at what size)
- Verified query specs for each component (human-approved MQL, not AI-generated)
- Cross-object metrics that would be difficult for the AI to derive from scratch

The AI never generates a preset. It modifies one. This dramatically reduces the search space — the AI is working from a correct baseline rather than generating correctness from scratch.

---

## Contextual Retrieval: Our Approach

Anthropic's Contextual Retrieval paper describes prepending chunk-specific context to each chunk before embedding, reducing retrieval failure by 49% (67% with reranking). Our domain has a similar problem — matching user vocabulary to data vocabulary — but our corpus is structured (a semantic catalog) rather than unstructured (document chunks).

### Bigram scoring as domain-specific BM25

Traditional BM25 excels at exact term matching but fails for Chinese business terms where character insertion, reordering, and synonyms are common. Our bigram coverage scorer serves the same role as BM25 in Anthropic's hybrid retrieval — capturing lexical similarity that embedding models miss:

```python
def term_score(question, term):
    term_bigrams = {term[i:i+2] for i in range(len(term)-1)}
    query_bigrams = {question[i:i+2] for i in range(len(question)-1)}
    coverage = len(term_bigrams & query_bigrams) / len(term_bigrams)
    return int(coverage * min(90, 30 + 4 * len(term)))
```

This is our BM25 equivalent for Chinese: it handles the specific failure modes of our language without requiring a full BM25 index.

### Per-kind quota allocation as reranking

Anthropic uses a reranker to prioritize the most relevant chunks. We use **per-kind quotas**: the semantic retrieval returns up to 4 metrics, 2 derived metrics, and 3 dimensions — allocated separately, not competing in a global top-k.

Why? Without per-kind allocation, dimensions (which tend to have short, common names like "region" or "month") dominate the global ranking and crowd out metrics. Per-kind quotas ensure the LLM always sees a balanced set of candidates for both "what to measure" and "how to group it."

### The grounding block as contextual injection

Anthropic's contextual chunk includes "This chunk is from an SEC filing on ACME corp's performance in Q2 2023." Our `grounding_block()` function generates a similar contextual preamble:

```python
def grounding_block(question):
    hits = score_all_terms(question)  # Bigram scoring
    return "## Semantic Candidates\n" + format_hits(hits)
    # Output: ranked list of metrics/dimensions with descriptions
    # e.g., "启用合同数 (active contract count): contracts where is_active=true"
```

This block is injected into the LLM's context alongside the catalog summary and approved examples, providing the same grounding function as Anthropic's contextual embeddings — the LLM sees not just the term, but its meaning in our specific business domain.

---

## Designing for What You Can't Build Yet

A recurring theme in our memory system is **designing interfaces for features we haven't built yet:**

| Feature | Status | Interface Exists |
|---------|--------|-----------------|
| Vector-based memory search tool | Not built | `recall()` function reserved |
| Memory consolidation (merge duplicates) | Not built | `consolidate()` stub in design doc |
| Memory staleness (decay old facts) | Not built | `updated_at` field tracked on every memory |
| Batch extraction queue | Not built | `TASK_RUNNER` port designed for arq workers |
| Cross-domain permission per-domain whitelist | Not built | `domain` field on every memory |

This is the "migration ports" philosophy applied to memory: **know where you'll expand, don't build it yet, but make sure the interface exists.** The cost of designing the interface is near-zero. The cost of not having it when you need it is a rewrite.

---

## What We'd Change

### 1. Rolling summary from day one

The jump from truncation to rolling summary was a 3x improvement in multi-turn coherence. It should have been the first memory feature, not the third. If you're building any multi-turn AI system, start with summarization — not because it's hard (it's ~150 lines of code) but because everything you build on top of it will work better.

### 2. Token tracking from the first LLM call

We added the dual-write cost tracking in month three. The debugging pain in months one and two — "why is this graph run so slow?" with no per-node token breakdown — was severe. Every LLM call should be instrumented from the first commit.

### 3. More aggressive model tiering

We initially used GLM-4.6 for everything, including intent routing. Switching to GLM-4-flash (free) for classification dropped our per-query cost by ~35% with no measurable accuracy impact. We should have started with the cheapest model that works and only upgraded where quality demanded it.

### 4. Background extraction should batch

Currently, every turn triggers an independent LLM extraction call. At peak usage (50 concurrent conversations), that's 50 concurrent extraction calls competing for GPU time with the primary response generation. A batching queue (extract every N turns, or batch across users with a small delay) would smooth the load without meaningful quality loss.

---

## Principles

If there's a generalizable lesson from building this system, it's that **memory and token economics are not separate concerns — they are the same concern viewed from two angles.**

Every memory design decision has a token cost implication:
- Cheap injection (DB query, no embedding) keeps the per-turn floor low
- Async distillation moves the expensive LLM call off the critical path
- Rolling summary caps context growth regardless of conversation length
- Selective few-shot injection avoids the O(n) scaling of full example injection
- Semantic IR instead of raw MQL shrinks output tokens

And every token budget decision has a memory quality implication:
- Shortcircuiting intent routing means no slots are extracted — only for side-effect-free intents
- Capping context tokens means older messages are summarized — some precision is lost
- Using natural language for conditions means a second LLM call is needed downstream — but only when actually filtering

The art is finding the design where both concerns are satisfied simultaneously. The three-layer memory with asymmetric read/write economics is our best current answer. It's not the final answer — the interfaces for features we haven't built yet make that clear.

---

*The system described serves ~300 enterprise users across four business domains. All data remains on-premise in compliance with China's PIPL. Token costs are based on domestic LLM pricing (ZhiPu GLM).*
