---
layout: default
title: "Building AI Observability Without LangSmith: An 8-Table Lineage System in MongoDB"
date: 2026-09-06
categories: [AI Architecture, Observability, LLMOps, MongoDB]
---

# Building AI Observability Without LangSmith: An 8-Table Lineage System in MongoDB

*September 6, 2026 · 20 min read*

**Key Takeaways**

- When regulatory constraints prohibit external SaaS observability tools (LangSmith, Weights & Biases, Langfuse), you need to build your own. Ours is 8 MongoDB collections, ~400 lines of core code, and ~2ms overhead per graph node.
- The most important design decision is **best-effort writes**: observability must never block the main conversation flow. If MongoDB is slow or unavailable, the lineage writer warns and degrades — the user's response is never delayed.
- Cost attribution at the node level ("which step in my graph costs the most?") requires binding `llm_purpose` to every LLM call at the framework level. We didn't do this until month three and lost two months of cost data.

---

## Why We Couldn't Use LangSmith

Our enterprise AI workbench serves ~300 users on-premise in mainland China. China's Personal Information Protection Law (PIPL) and our customers' security policies create a hard constraint: **no data crosses national borders.** Not metadata, not trace IDs, not token counts. Nothing.

This eliminates every SaaS observability tool in the LLM ecosystem:

| Tool | Why We Can't Use It |
|------|-------------------|
| **LangSmith** | Cloud-hosted, data leaves China |
| **Weights & Biases** | Same — servers outside China |
| **Langfuse** | Open-source but the hosted version is abroad; self-hosted option exists but we discovered it after building our own |
| **Arize Phoenix** | Same cross-border issue |
| **OpenLLMetry** | OTel-based, needs a collector endpoint we'd have to host ourselves |

We needed observability that runs entirely within our infrastructure, stores data in the same MongoDB instance we already use for business data, and adds minimal operational complexity.

The result: an 8-table lineage system that traces every request from HTTP endpoint through graph execution to individual LLM calls, with full cost attribution.

---

## The Debugging Pain That Motivated This

For the first two months, we had no tracing. When something went wrong — and in a multi-node LangGraph system, things go wrong constantly — debugging looked like this:

1. User reports: "The dashboard shows wrong numbers."
2. We check the dashboard. The chart looks fine. The numbers seem plausible.
3. We re-run the same query manually. Different result.
4. We check the graph execution... but there's no record of which nodes ran, what inputs they received, or what the LLM actually generated.
5. We add `print()` statements, reproduce the issue, read through console logs.
6. Three hours later, we discover: the `context_compress` node summarized away a critical filter condition. The LLM in the `generate_query` node never saw it.

This class of bug — "the right node ran with wrong input state" — is trivial to diagnose with a lineage trace (look at node input hashes before and after compression) and nearly impossible without one.

We built the lineage system in month three. The debugging pain in months one and two — conservatively 40+ engineering hours — would have been avoided if we'd built it on day one.

---

## The 8-Table Schema

![Figure 1. 8-Table Lineage Schema](/assets/images/fig14-lineage-schema.svg)
*Figure 1. Eight collections form a hierarchical trace: request → graph_run → node_run → llm_call/skill_run, with cross-references (trace_link), events (trace_event), and cost accounting (usage_ledger).*

### The hierarchy

```
trace_request          One per HTTP request
  └── graph_run        One per LangGraph invocation (root graph or sub-graph)
        └── node_run   One per node execution within a graph
              ├── llm_call     One per LLM API call
              └── skill_run    One per skill/tool execution
  
trace_link             Cross-graph references (parent ↔ child graph)
trace_event            Arbitrary structured events (domain-level signals)
usage_ledger           Token/cost accounting (dual-written with llm_call)
```

### What each table stores

**`trace_request`** — The entry point. Created when an HTTP request arrives, finished when the response is sent. Contains:
- `trace_id`: the correlation ID that flows through everything
- `user_id`, `workspace_id`, `tenant_id`: who made the request
- `endpoint`, `method`: what they asked for
- `status`, `latency_ms`: how it went

**`graph_run`** — One record per LangGraph graph invocation. A single request might trigger multiple graph runs (root graph → conversation sub-graph → ReAct sub-graph). Contains:
- `graph_run_id`: unique identifier
- `trace_id`: links back to the request
- `graph_name`: which graph ran (`root_graph`, `conversation_graph`, `data_graph`, `data_query_subgraph`)
- `thread_id`: LangGraph thread (for checkpoint correlation)
- `total_tokens`, `total_cost`: aggregated from child nodes (computed at `finish_graph_run`)

**`node_run`** — One record per node execution. The most queried table for debugging. Contains:
- `node_run_id`: unique identifier
- `graph_run_id`: links to parent graph
- `node_name`: which node (`detect_intent`, `generate_query`, `context_compress`, etc.)
- `input_hash`, `output_hash`: SHA-256 of the node's input and output state. When two consecutive nodes have different output hashes, you know exactly where the state changed.
- `duration_ms`: how long the node took
- `llm_purpose`: what this node's LLM calls were for (critical for cost attribution)

**`llm_call`** — One record per LLM API call. A single node might make multiple LLM calls (e.g., `generate_query` calls the LLM once for planning and once for compilation). Contains:
- `model`, `provider`: which model was called
- `request_tokens`, `response_tokens`, `total_tokens`: token counts
- `latency_ms`: API call duration
- `cost_usd`: estimated cost (computed from pricing table)
- `prompt_id`, `prompt_version`: which prompt template was used (links to prompt registry)
- `status`, `error_message`: success or failure details

**`skill_run`** — One record per skill/tool execution. Contains:
- `skill_name`, `image`: which Docker image ran
- `mode`: `sync` or `async`
- `duration_ms`, `exit_code`: execution results
- `output_files`: list of files produced

**`trace_link`** — Cross-references between entities. When a root graph spawns a sub-graph, a link connects the parent's `graph_run_id` to the child's. This enables "show me the full call tree for this request."

**`trace_event`** — Arbitrary structured events attached to any point in the trace. Examples: `dashboard.query_succeeded`, `skill.sandbox_timeout`, `memory.cross_domain_access`. The `domain` field is auto-inferred from the event name prefix.

**`usage_ledger`** — Token and cost accounting. Dual-written alongside `llm_call` for query convenience. Contains per-call cost and links to user, workspace, model, and prompt version. This table powers the "how much did each user/feature/model cost this month?" dashboard.

---

## The Writer: Best-Effort by Design

The lineage writer is the most important component to get right — not because it's complex, but because getting it wrong can destroy your application's reliability.

### The cardinal rule: never block the main flow

```python
async def _insert(collection: str, doc: dict) -> None:
    try:
        await run_in_threadpool(lambda: mongodb_repo.get_db()[collection].insert_one(doc))
    except Exception:
        logger.warning("lineage write to %s failed (degrading)", collection)
        # NEVER re-raise. The user's response must not be delayed
        # because the observability system is having a bad day.
```

Every write operation in the lineage writer follows this pattern. The `try/except` with a warning log and no re-raise is deliberate. We've seen MongoDB hiccup during high-load periods (connection pool exhaustion, slow disk I/O). Without this pattern, a MongoDB hiccup would cascade into user-facing request failures — which is exactly backwards. Observability exists to serve the application, not the other way around.

### The lifecycle

A typical conversation turn generates this sequence of lineage calls:

```python
# 1. HTTP request arrives
await lineage_writer.open_request(endpoint="/api/chat", method="POST")

# 2. Root graph starts
await lineage_writer.start_graph_run(graph_name="root_graph")

#   3. Each node executes
    async with node_tracker.node_run("detect_intent") as handle:
        # 4. LLM calls within this node auto-associate via RequestContext
        result = await llmproxy_client.complete_typed(...)
        # llm_call_tracker.record_call() is called inside complete_typed
        # It writes to BOTH llm_call AND usage_ledger

    async with node_tracker.node_run("generate_query") as handle:
        result = await llmproxy_client.complete_typed(...)

# 5. Graph finishes (aggregates token counts from child nodes)
await lineage_writer.finish_graph_run(status="success")

# 6. Request finishes
await lineage_writer.finish_request(status=200)
```

### Automatic ID propagation via RequestContext

The key engineering trick is that **callers don't pass IDs manually.** When `open_request()` is called, it generates a `trace_id` and binds it to a `RequestContext` (an async context variable). When `start_graph_run()` is called, it generates a `graph_run_id` and adds it to the same context. By the time `record_llm_call()` is called deep inside a node, it reads `trace_id`, `graph_run_id`, and `node_run_id` from the context automatically.

```python
# Inside lineage_writer.record_llm_call():
ctx = get_context()
doc = {
    "trace_id": ctx.trace_id,           # auto from open_request
    "graph_run_id": ctx.graph_run_id,   # auto from start_graph_run
    "node_run_id": ctx.node_run_id,     # auto from node_tracker
    "model": model_name,                # passed by caller
    "request_tokens": req,              # passed by caller
    ...
}
```

This means adding observability to a new node requires zero boilerplate. Wrap the node's body in `async with node_tracker.node_run("my_node"):` and all LLM calls inside it are automatically attributed.

---

## Cost Attribution: "Which Step Costs the Most?"

![Figure 2. Cost Attribution Flow](/assets/images/fig15-cost-attribution.svg)
*Figure 2. Every LLM call is dual-written to the lineage (for debugging) and the usage ledger (for billing). The `llm_purpose` field enables queries like "how much did intent routing cost this month?"*

### The dual-write pattern

Every LLM call writes to two tables simultaneously:

1. **`llm_call`** — for lineage/debugging ("what happened and why?")
2. **`usage_ledger`** — for cost/billing ("how much did it cost and who pays?")

```python
async def record_call(*, model_name, usage, latency_ms, prompt_id, ...):
    req, resp, total = _parse_usage(usage)
    cost = token_ledger.compute_cost(model_name, req, resp)
    
    # Write 1: lineage
    llm_call_id = await lineage_writer.record_llm_call(
        model_name=model_name, request_tokens=req, response_tokens=resp,
        cost_usd=cost, prompt_id=prompt_id, ...
    )
    
    # Write 2: usage ledger
    await token_ledger.record(
        model=model_name, request_tokens=req, response_tokens=resp,
        cost=cost, llm_call_id=llm_call_id, ...
    )
```

Why two tables instead of one? Different query patterns. The lineage table is queried by `trace_id` for debugging ("show me everything that happened in this request"). The usage ledger is queried by `user_id + month` for billing ("how much did this user spend?"). Different access patterns, different indexes, different retention policies.

### The pricing table

Cost estimation uses a per-model pricing table:

```python
PRICING = {
    "glm-4.6":     {"in": 0.002, "out": 0.002},    # per 1k tokens
    "glm-4-flash": {"in": 0.0,   "out": 0.0},      # free tier
    "glm-4-air":   {"in": 0.0005, "out": 0.0005},   # economy tier
}

def compute_cost(model, request_tokens, response_tokens):
    price = PRICING.get(model, {})
    return (request_tokens / 1000) * price.get("in", 0) + \
           (response_tokens / 1000) * price.get("out", 0)
```

The pricing is an estimate — our internal LLM provider's billing is more complex (volume discounts, reserved capacity). But the estimate is accurate enough for "which step is expensive?" analysis. When we eventually connect to the billing center's API, we'll replace `compute_cost` with a real lookup. The lineage schema doesn't change.

### The `llm_purpose` binding

The most useful cost attribution field is `llm_purpose` on the `node_run` table. When a node starts, the tracker binds `llm_purpose=node_name` to the RequestContext:

```python
async with node_tracker.node_run("detect_intent") as handle:
    # Every LLM call inside this block will have
    # llm_purpose="detect_intent" in usage_ledger
    result = await llmproxy_client.complete_typed(...)
```

This enables queries like:

```javascript
// "How much does intent routing cost per month?"
db.usage_ledger.aggregate([
    { $match: { "llm_purpose": "detect_intent", "month": "2026-09" } },
    { $group: { _id: null, total_cost: { $sum: "$cost_usd" }, total_tokens: { $sum: "$total_tokens" } } }
])

// "What's our most expensive graph node?"
db.usage_ledger.aggregate([
    { $match: { "month": "2026-09" } },
    { $group: { _id: "$llm_purpose", cost: { $sum: "$cost_usd" }, calls: { $sum: 1 } } },
    { $sort: { cost: -1 } }
])
```

**What we'd change:** The `llm_purpose` binding should be built into the framework from day one. We added it in month three, which means we have no cost-per-node data for months one and two. We know our total cost grew 3x during that period but can't determine which feature or which node caused the growth.

---

## The Usage Metadata Problem

A subtle but important engineering detail: different LLM libraries report token usage in different field names.

```python
def _parse_usage(usage: dict | None) -> tuple[int, int, int]:
    usage = usage or {}
    # OpenAI-compatible (via stream_chat): prompt_tokens / completion_tokens
    req = int(usage.get("prompt_tokens") or usage.get("input_tokens") or 0)
    # LangChain usage_metadata: input_tokens / output_tokens
    resp = int(usage.get("completion_tokens") or usage.get("output_tokens") or 0)
    total = int(usage.get("total_tokens") or (req + resp))
    return req, resp, total
```

We discovered this the hard way. Our data dashboard graph uses `llmproxy_client` (OpenAI-compatible, reports `prompt_tokens`). Our plan2 agent uses LangChain's `build_chat_model().ainvoke()` (reports `input_tokens`). Without parsing both field names, the plan2 path recorded `request_tokens=0, response_tokens=0, total_tokens=14652` — the total was correct, but the input/output breakdown was lost. And the input/output breakdown is exactly what you need to diagnose "is my prompt too long?"

---

## The Event System

Beyond the structured lineage tables, we have an event bus for domain-level signals:

```python
# Emitting a domain event
await emitter.emit("dashboard.query_succeeded", {
    "intent": "add_node",
    "chart_type": "pie",
    "query_duration_ms": 342,
    "result_rows": 8
})

# The domain is auto-inferred from the event name prefix
# "dashboard.query_succeeded" → domain = "dashboard"
```

Events are schemaless — the payload is arbitrary JSON. This is deliberate. The lineage tables have fixed schemas for structured queries. The event table is for ad-hoc signals that don't justify a schema change.

Common events we emit:

| Event Name | When | Payload |
|-----------|------|---------|
| `dashboard.query_succeeded` | A data dashboard query completes | intent, chart_type, duration, row count |
| `dashboard.validation_failed` | The 6-gate validator rejects a plan | gate_name, error_message, retry_count |
| `dashboard.hallucination_detected` | Adversarial probe finds a problem | field_name, expected_type, actual_value |
| `skill.sandbox_timeout` | A skill exceeds execution timeout | skill_name, timeout_seconds |
| `skill.yara_blocked` | YARA scanner blocks a skill publish | rule_name, file_path |
| `memory.cross_domain_access` | Cross-domain memory flag checked | from_domain, to_domain, allowed |

---

## Three-Layer Observability Design

Our observability architecture has three layers, though only one is fully implemented:

```
Layer 1: Metrics (Prometheus/Grafana)      → QPS, latency, error rate, token consumption
         Status: port reserved, no-op stubs implemented

Layer 2: Traces (OpenTelemetry)            → span-level distributed tracing  
         Status: port reserved, not yet connected

Layer 3: Business Lineage (MongoDB)        → semantic queryable lineage
         Status: FULLY IMPLEMENTED (this post)
```

### Why we shipped Layer 3 first

Metrics (Layer 1) tell you *what* is happening — "error rate is 12%." Traces (Layer 2) tell you *where* it's happening — "the slow span is in the `generate_query` node." But for an AI system, the most important question is *why* it's happening — "the LLM hallucinated a field name because the semantic catalog didn't include that metric."

Layer 3 answers the *why* because it stores semantic information (prompt versions, model names, input/output hashes, event payloads) that Prometheus counters and OTel spans don't capture.

### The port pattern

Layers 1 and 2 are defined as interfaces (ports) with no-op implementations:

```python
# monitoring/metrics/prometheus.py
class PrometheusMetrics:
    """No-op when metrics_enabled=False. Real implementation wraps prometheus_client."""
    
    def record_llm_latency(self, model: str, latency_ms: int): ...
    def increment_request_count(self, endpoint: str, status: int): ...
    def record_token_usage(self, model: str, tokens: int): ...
```

When we deploy Prometheus (planned for Q4), we'll implement these methods. The call sites in the application code are already in place — they just do nothing today. This is the same "migration port" pattern we use throughout the platform: define the interface now, swap the implementation later.

---

## The Sanitization Boundary

Every piece of data that enters the lineage system passes through a sanitizer:

```python
from services.shared.sanitizer import sanitize

doc = {
    "trace_id": ctx.trace_id,
    "model": model_name,
    "prompt_summary": sanitize(prompt_text, max_length=200),  # truncated + PII stripped
    "error_message": sanitize(error, max_length=500),
}
```

The sanitizer enforces two rules:

1. **Minimum data collection.** We store prompt IDs and versions, not full prompt text. We store input/output hashes, not actual inputs and outputs. The lineage tells you *which* data flowed through *which* path — not the data itself.

2. **PII removal.** User messages, LLM responses, and error messages are truncated and stripped of personal identifiers before storage. This is a PIPL requirement: even internal observability data must minimize personal information retention.

The trade-off: when debugging, we sometimes need the actual prompt text. We can retrieve it by joining the `prompt_id` with the prompt registry (which stores full templates) and the `trace_id` with the LangGraph checkpoint (which stores full conversation state). It's two extra queries, but it keeps the lineage tables clean.

---

## Cross-Border Hard Block

One more compliance control:

```python
# In the observability __init__.py:
# "Cross-border hard block: settings.allow_cross_border=False by default"

if settings.allow_cross_border:
    # Connect to external OTel collectors, Langfuse, etc.
    pass
else:
    # All observability data stays in local MongoDB
    # No external exporters are initialized
    pass
```

This is a belt-and-suspenders measure. Even if someone accidentally configures an external OpenTelemetry collector endpoint, the flag prevents the exporter from initializing. The default is `False`, and changing it requires explicit configuration — not something that happens by accident.

---

## What We Learned

### 1. Build observability before features

We said this in a previous post and it bears repeating here with concrete numbers. The lineage system is ~400 lines of core code. It took about one week to build. The debugging time it would have saved in months one and two was conservatively 40+ engineering hours. The ROI is approximately 40:1.

If you're building any graph-based AI system — LangGraph, LlamaIndex, or custom — instrument it before you write the second node.

### 2. The `llm_purpose` binding is the most valuable field

After months of production use, the single most queried field in our observability system is `llm_purpose`. It answers the question every stakeholder asks: "Why is our AI bill going up?"

The answer is always specific: "Intent routing costs 34% of total, up from 22% last month, because we added 5 new intents and the prompt grew by 400 tokens." Without `llm_purpose`, the answer would be "we made more LLM calls" — which is true but useless.

### 3. Best-effort is non-negotiable

In our first implementation, we used `await` for lineage writes without a try/except wrapper. During a MongoDB connection pool exhaustion event (caused by a spike in concurrent users), lineage writes started timing out. Because they were `await`-ed without error handling, the timeouts cascaded into the main request flow. Users experienced 30-second response times — not because the LLM was slow, but because the observability system was slow.

After adding the best-effort pattern (try/except with warning log), the same MongoDB issue produced a gap in the lineage data but zero impact on user experience. That's the correct trade-off.

### 4. Dual-write saves query complexity

We initially considered deriving cost reports by joining `llm_call` with other tables at query time. In practice, the join queries were complex (5-table joins for "per-user monthly cost by feature") and slow. The dual-write to `usage_ledger` — which denormalizes user, model, cost, and purpose into a single collection — makes billing queries trivial: one `$group` by `user_id` and `month`.

The storage cost of the duplication is negligible (~100 bytes per LLM call). The query simplification is significant.

### 5. Input/output hashes > input/output content

Storing full node inputs and outputs would be ideal for debugging but creates two problems: storage bloat (a single conversation turn generates ~50KB of state data across all nodes) and privacy concerns (the state contains user messages).

Hashes give us 80% of the debugging value at 0.1% of the storage cost. When two consecutive nodes have the same input hash but different output hashes, we know exactly which node modified the state. When a node's input hash is different from the previous node's output hash, something unexpected happened in the graph routing. The actual content, when needed, is always available in the LangGraph checkpoint store.

---

## The Query Patterns That Matter

After six months of production use, these are the five queries we run most often:

### 1. "Why did this request fail?"

```javascript
db.trace_request.findOne({ trace_id: "..." })
→ get graph_run_ids
db.node_run.find({ graph_run_id: "..." }).sort({ started_at: 1 })
→ find the node with status: "failed"
→ read error message + input_hash
```

### 2. "How much does each feature cost?"

```javascript
db.usage_ledger.aggregate([
    { $match: { month: "2026-09" } },
    { $group: { _id: "$llm_purpose", cost: { $sum: "$cost_usd" }, tokens: { $sum: "$total_tokens" } } },
    { $sort: { cost: -1 } }
])
```

### 3. "Is our hallucination rate improving?"

```javascript
db.trace_event.aggregate([
    { $match: { event_name: "dashboard.validation_failed", month: "2026-09" } },
    { $group: { _id: { $dayOfMonth: "$created_at" }, count: { $sum: 1 } } },
    { $sort: { "_id": 1 } }
])
```

### 4. "Which prompt version performs better?"

```javascript
db.llm_call.aggregate([
    { $match: { prompt_id: "dashboard.intent_route" } },
    { $group: { _id: "$prompt_version", avg_tokens: { $avg: "$total_tokens" }, calls: { $sum: 1 } } },
    { $sort: { _id: -1 } }
])
```

### 5. "Who are our heaviest users?"

```javascript
db.usage_ledger.aggregate([
    { $match: { month: "2026-09" } },
    { $group: { _id: "$actor_user_id", cost: { $sum: "$cost_usd" }, calls: { $sum: 1 } } },
    { $sort: { cost: -1 } },
    { $limit: 10 }
])
```

---

## For Teams Building AI Observability

If you're in a similar situation — on-premise, no SaaS tools, need to build your own — here's the order we'd recommend:

| Phase | What | Time | Why First |
|-------|------|------|-----------|
| **Week 1** | `trace_request` + `node_run` | 2 days | Answers "what happened and where did it fail?" |
| **Week 1** | `llm_call` + `usage_ledger` | 1 day | Answers "how much does it cost?" |
| **Week 2** | `graph_run` + `trace_link` | 1 day | Answers "how do sub-graphs relate?" |
| **Week 2** | `trace_event` | 1 day | Answers domain-specific "why?" questions |
| **Week 3** | `skill_run` | 1 day | Only if you have tool/skill execution |
| **Week 3** | Dashboard / alerting | 2 days | Make the data visible to the team |

Total: ~8 working days for a complete lineage system. The cost of NOT building it — measured in debugging hours, blind cost optimization, and undetectable regressions — is an order of magnitude higher.

---

*The system described operates entirely on-premise, stores all observability data in MongoDB, and complies with China's PIPL. No data crosses national borders. Architecture patterns are described generically; no internal identifiers are shared.*
