---
layout: default
title: "Context Engineering Lessons from 300-User Production: Beyond Prompt Engineering"
date: 2026-09-06
categories: [AI Architecture, Context Engineering, LLM, Agents]
---

# Context Engineering Lessons from 300-User Production: Beyond Prompt Engineering

*September 6, 2026 · 20 min read*

Anthropic recently published [an excellent piece on context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) arguing that building with LLMs is becoming "less about finding the right words and phrases for your prompts, and more about answering the broader question of 'what configuration of context is most likely to generate our model's desired behavior?'" We agree — and we have eight months of production data showing exactly what that means in practice.

Our enterprise AI workbench serves ~300 users across four business domains: conversation AI, natural language data dashboards, a skill marketplace, and external tool integration. Every user turn involves assembling a context window from multiple sources — system prompts, memory, conversation history, semantic grounding, component schemas, and dynamically retrieved examples. Getting this assembly wrong doesn't just degrade quality; it actively causes hallucination, wasted tokens, and user-visible failures.

This post describes the six context engineering strategies we developed, the production incidents that motivated them, and the quantitative impact of each. Where our experience aligns with Anthropic's framework, we'll say so. Where it diverges — particularly around the constraints of on-premise deployment without LangSmith — we'll explain why.

---

## Strategy 1: Three-Layer Memory — Because Context Is Not Just the Current Turn

Anthropic describes context as "a finite resource with diminishing marginal returns." We agree, but the challenge in a multi-domain enterprise system goes further: context isn't just about the current turn. It spans sessions, domains, and user histories that accumulate over months.

### The architecture

| Layer | Scope | Technology | Retention |
|-------|-------|-----------|-----------|
| **Short-term** | Current thread | LangGraph checkpoint + rolling summary | Session lifetime |
| **Long-term** | Per-user, per-domain | `langmem` + MongoDB vector store | Indefinite |
| **Cross-domain** | Per-user, all domains | Same store, gated by `MEMORY_CROSS_DOMAIN` flag | Configurable |

**Short-term memory** is managed by the LangGraph checkpoint system. Every graph node receives the full state of the current thread, including message history. This works well for recent context but creates a growing problem: as conversations extend beyond 10–15 turns, the accumulated messages consume thousands of tokens.

**Long-term memory** is extracted at the end of each turn by a `memory_write` node. Key facts from the conversation are vectorized and stored in MongoDB. Future conversations retrieve relevant memories via the `memory_inject` node at the start of each turn. This gives the system "recall" — a user who discussed Q2 revenue last week doesn't need to re-explain the filters.

**Cross-domain memory** is the most controversial layer. Our workbench has four domains (conversation, dashboards, skills, MCP). Should the dashboard domain know what the user discussed in a conversation thread? The privacy implications are non-trivial — a user might discuss sensitive HR data in conversation and not want that context leaking into their dashboard queries.

Our solution: **a single boolean flag** (`MEMORY_CROSS_DOMAIN`, default `false`). When off, each domain's memory is isolated. When on, the system can draw from all domains for richer context. The privacy trade-off is explicit and auditable, not implicit.

### The lesson

Anthropic recommends treating context as "a finite resource." We'd add: **treat memory as three different resources with three different lifetimes and three different trust levels.** Short-term memory is high-trust (it's from the current session). Long-term memory is medium-trust (it might be stale). Cross-domain memory is low-trust (it might violate privacy expectations). Each requires different handling.

---

## Strategy 2: Rolling Context Compression — The Most Impactful Single Change

![Figure 1. Context Injection Pipeline](/assets/images/fig16-context-layers.svg)
*Figure 1. Context flows from three memory layers through an assembly pipeline that curates the minimal token set for each LLM call.*

### The failure that motivated it

Our first implementation used simple truncation: cut messages beyond a 4,000-token limit, keeping only the most recent ones. This "worked" in the sense that the system didn't crash. It failed in the sense that users noticed the AI forgot things they said three messages ago.

The pathology was specific: a user would set up a complex dashboard with filters ("show me Q2 2026, insurance type A, only confirmed bills"), then ask a follow-up question ("now break that down by region"). The truncation cut the messages containing the original filters. The follow-up query generated a dashboard with no filters at all — producing numbers that looked correct but were for the wrong data subset.

### The fix

The `context_compress` node now runs before every LLM call. Instead of truncating, it summarizes:

```
Before compression:
  Message 1 (user):   "Show me confirmed bills for Q2 2026, insurance type A"
  Message 2 (asst):   "Here's the dashboard with 3 charts..."
  Message 3 (user):   "The pie chart is hard to read, change it to a bar chart"
  Message 4 (asst):   "Done, I've updated the chart type..."
  Message 5 (user):   "Now add a filter for amounts over 10,000"
  Message 6 (asst):   "Added the amount filter..."
  [... 10 more turns ...]
  Message 17 (user):  "Break that down by region"

After compression:
  [Summary]: User is analyzing confirmed bills for Q2 2026, insurance type A,
             with an amount filter > 10,000. Dashboard has bar charts and
             KPI cards. User has been iterating on chart types and filters.
  Message 15 (user): [verbatim - recent]
  Message 16 (asst): [verbatim - recent]
  Message 17 (user): "Break that down by region"
```

Older messages are compressed into a semantic digest. Recent messages (last 3–4 turns) are preserved verbatim. The LLM sees the full context of what's been discussed without the token cost of carrying every historical message.

### The impact

Multi-turn coherence improved dramatically. Before compression, users experienced "context amnesia" after ~8 turns. After compression, conversations could extend to 30+ turns without losing track of established filters and preferences.

**The regret:** This should have been the first memory feature, not the third. Every multi-turn feature we built on top of truncation worked worse than it should have. If you're building any multi-turn system, implement rolling summarization before your second feature.

---

## Strategy 3: The Intent Router's Context Diet

Anthropic warns about "context rot" — as the context window grows, the model's ability to accurately recall information decreases. Our intent router is the most context-sensitive component in the system (it classifies every user message into one of 11 intents), and it's where we learned the most about context discipline.

### What the router receives

```
1. System prompt (skeleton with 3 dynamic slots)     ~800 tokens
2. Intent definitions (from intent_catalog)           ~350 tokens
3. Component summary ({id, title} pairs)              ~150 tokens
4. Context ref (recent conversation turns)            ~400 tokens
5. Few-shot examples (vector-retrieved top-k)         ~250 tokens
6. User message (current turn)                        ~50 tokens
                                          Total:    ~2,000 tokens
```

### What the router does NOT receive

- ❌ **Full dashboard JSON** (MQL pipelines, data mappings, props) — this is ~3,000–5,000 tokens that the router doesn't need. The router only needs to know *what charts exist* (the summary table), not *how they query data*.
- ❌ **All 170 intent example utterances** — only the vector-retrieved top-k (~5) are injected. Injecting all examples would dilute attention across irrelevant cases.
- ❌ **Field whitelist for filters** — this is hundreds of lines (field names, types, allowed values across 4 tables). The router's `filter_change` intent captures the user's filtering intent as natural language; the downstream `apply_filter` node gets the whitelist only when it actually needs to translate that intent into structured conditions.

### The decision that was reversed — and why

Our `context_ref` block (conversation history for the router) originally **filtered out assistant messages** — the router only saw user messages. The reasoning: assistant messages sometimes contained clarification questions ("I'm not sure which chart you mean"), and the model would "learn" from seeing those that clarification was an acceptable behavior, producing a "snowball clarify" effect.

We reversed this after two production incidents:

**Incident 1: Orphan reference.** The assistant's previous turn suggested: "You could display monthly success rates by insurance type." The user's next message was: "Yes, do that." The router saw only "Yes, do that" — with no assistant context, it couldn't resolve what "that" referred to. It classified as `clarify` and asked the user to repeat something they'd already clearly confirmed.

**Incident 2: Disconnected follow-up.** The assistant asked: "Which card do you want to modify?" The user replied: "The billing one." Without the assistant's question in context, the router saw "The billing one" as a standalone message and classified it as a vague search intent, not a response to a specific question.

The original concern (snowball clarify) was later resolved at the decoding layer — `clarify` was removed from the LLM's allowed enum entirely, making it impossible for the model to output regardless of what it saw in the history. With that fix in place, including assistant messages became safe and necessary.

**The lesson:** Don't solve context problems by removing context. Solve them by constraining the output space. When we removed assistant messages, we fixed one problem (snowball clarify) but created two others (orphan reference, disconnected follow-up). When we constrained the output enum instead, we fixed the original problem without side effects.

---

## Strategy 4: Semantic Grounding — Giving the Model the Right Vocabulary

Anthropic emphasizes that "tools define the contract between agents and their information/action space." In our data dashboard domain, the equivalent is the semantic layer — the vocabulary of metrics, dimensions, and business concepts the model can use.

### The grounding block

Every query-related LLM call receives a `grounding_block` assembled from three sources:

```python
messages = [
    {"role": "system", "content": catalog.catalog_brief()},       # What metrics exist
    {"role": "system", "content": board_composer._schema_summary()}, # What views are queryable
    {"role": "system", "content": rag.grounding_block(question)},  # Relevant candidates for THIS question
    *_phrasebook_block(state),                                     # Approved example queries
    {"role": "system", "content": "## Prior agent results\n" + ...},
    {"role": "user", "content": user_question},
]
```

Each source serves a different grounding function:

| Source | What It Provides | Token Cost | When Included |
|--------|-----------------|-----------|---------------|
| `catalog_brief()` | Full metric catalog summary | ~200 tok | Always (in query nodes) |
| `_schema_summary()` | Queryable view schemas | ~150 tok | Always (in query nodes) |
| `rag.grounding_block()` | Top-k semantic candidates for this question | ~100-300 tok | Always (dynamic, question-specific) |
| `_phrasebook_block()` | Approved example queries | ~200-500 tok | When examples exist |

### The key design: per-kind quotas

The semantic RAG retrieval doesn't just return a flat top-k list. It returns candidates with **per-kind quotas**: up to 4 metrics, 2 derived metrics, and 3 dimensions. This prevents a common failure mode where one category dominates the results.

In our data model, dimension names (like "region" or "insurance type") are often short and common — they match many queries with high scores. Without per-kind quotas, dimension candidates would crowd out metric candidates, and the LLM would know *how to slice the data* but not *what to measure*.

### The grounding-to-execution separation

A critical architectural decision: the semantic grounding context provides **candidates**, not **decisions**. The LLM sees "here are the metrics that might be relevant" and writes a `SemanticQueryPlan` (an intermediate representation). A deterministic compiler then translates the plan into an actual MongoDB pipeline.

This separation means hallucination in the grounding context is recoverable. If the RAG retrieval surfaces an irrelevant metric, the LLM might include it in the plan — but the compiler will reject it if the metric doesn't exist in the schema. The compiler is the final authority, not the LLM.

---

## Strategy 5: Few-Shot as Conversation History — A 20% Accuracy Gain

Anthropic recommends few-shot prompting as "pictures worth a thousand words." We agree, and we discovered that *how* you format those examples matters as much as *which* examples you choose.

### The formatting insight

We tested two approaches for injecting few-shot examples:

**Approach A: System prompt documentation**
```python
messages = [
    {"role": "system", "content": "...definitions...\n\n## Examples\n"
     "Q: '按险种看合同数' → intent: add_node\n"
     "Q: '切到7月' → intent: filter_change\n"
     "..."},
    {"role": "user", "content": actual_question}
]
```

**Approach B: Fake conversation history**
```python
messages = [
    {"role": "system", "content": "...definitions..."},
    {"role": "user", "content": "按险种看合同数"},
    {"role": "assistant", "content": '{"intent": "add_node", ...}'},
    {"role": "user", "content": "切到7月"},
    {"role": "assistant", "content": '{"intent": "filter_change", ...}'},
    {"role": "user", "content": actual_question}
]
```

Approach B outperformed Approach A by ~20% on intent classification accuracy. The model treats its own "previous answers" as stronger precedent than instructions. When it sees that it "previously" classified a similar question as `filter_change`, it's more likely to classify the current similar question the same way.

### Dynamic retrieval, not static injection

We don't inject all examples. Our intent corpus contains ~170 labeled utterances across 11 intents. Injecting all of them would consume ~2,000 tokens and dilute attention — most examples would be irrelevant to any given question.

Instead, we embed the user's question, retrieve the top-k most similar examples from the corpus (using in-memory cosine similarity, <1ms), and inject only those. This is adapted from RAGFlow's `CategorizeParam.update_prompt()` pattern, with one difference: RAGFlow injects all examples because it typically has ~5 categories with ~3 examples each. We have 11 intents × ~15 examples — too many for bulk injection.

### The dual-use pattern

The embedding call for retrieval serves two purposes:

1. **Shortcircuit** (37% of requests): If the top match has cosine similarity > 0.92 and the top-3 agree on the same intent, and that intent has no required slots, we skip the LLM entirely. ~200ms latency, zero tokens.

2. **Few-shot enhancement** (63% of requests): If we can't shortcircuit, the retrieved examples become few-shot demonstrations in the LLM prompt. The embedding is never wasted.

One embedding call, two uses. The retrieval cost is amortized whether or not we call the LLM.

---

## Strategy 6: The Context Budget Problem — What NOT to Include

![Figure 2. Context Budget by Node Type](/assets/images/fig17-context-budget.svg)
*Figure 2. Left: Token allocation for the intent router (~2,000 tokens). Right: Different nodes get different context budgets — no node sees everything.*

Anthropic frames context engineering as "finding the smallest possible set of high-signal tokens that maximize the likelihood of some desired outcome." In practice, this means deciding what to **exclude** is as important as deciding what to include.

### The field whitelist problem

Our most instructive exclusion decision involves the `filter_change` intent. When a user says "only show me bills from Shanghai," the system needs to translate "Shanghai" into a structured filter condition: `{field: "customer_city", operator: "eq", value: "Shanghai"}`. This translation requires a whitelist of filterable fields — their names, types, allowed values, and Chinese aliases.

This whitelist is ~400 lines. Including it in every intent routing call would add ~1,500 tokens to every single classification — even for chitchat, even for simple questions, even for requests that have nothing to do with filtering.

Our solution: the router captures the filtering intent as **natural language** in a slot called `conditions_text`:

```json
{
  "intent": "filter_change",
  "slots": {
    "month": null,
    "completion": null,
    "conditions_text": "only show bills from Shanghai"
  }
}
```

The downstream `apply_filter` node — which runs **only** when the intent is `filter_change` — receives the whitelist and translates the natural language into a structured condition. The whitelist is loaded lazily, only when needed, not broadcast to every call.

This is a deliberate trade-off: the router's classification is slightly less informed (it doesn't know which fields are filterable), but the token savings across all other intents are substantial. In our telemetry, `filter_change` accounts for ~15% of requests. The other 85% save ~1,500 tokens each.

### The MQL exclusion

The intent router never sees the dashboard's MQL pipelines. This was initially a bug — the router was supposed to see the full dashboard state for better classification. We discovered that including MQL caused a different problem: the router would sometimes start *rewriting* the MQL in its classification output, even though its only job was to classify intent. The presence of code-like content in the context activated the model's code generation behavior.

Removing MQL from the router's context didn't hurt classification accuracy (the component summary table with `{id, title}` pairs was sufficient for reference resolution) and eliminated an entire class of "the router tried to modify the dashboard" failures.

### The principle

**Each node in the graph gets exactly the context it needs to do its job and nothing more.** The router gets intent definitions and component summaries. The query generator gets semantic grounding and schema descriptions. The board composer gets component registries and layout constraints. The narrator gets dashboard data and user context.

No node sees the complete picture. This is not a limitation — it's a design principle. A node that sees everything is tempted to do everything. A node that sees only its inputs does only its job.

---

## The Production Metrics

| Strategy | Failure Mode Addressed | Impact |
|----------|----------------------|--------|
| Three-layer memory | Context amnesia across sessions | Long-term recall enabled |
| Rolling compression | Mid-conversation context loss | 8 turns → 30+ turns coherence |
| Router context diet | Classification confusion from irrelevant context | ~12% accuracy improvement |
| Semantic grounding | Wrong metric/dimension selection | ~25% hallucination reduction |
| Few-shot as history | Low classification accuracy | ~20% accuracy improvement |
| Lazy whitelist loading | Token waste on non-filter requests | ~1,500 tokens saved per non-filter request |

---

## What We'd Do Differently

1. **Start with compression, not truncation.** The LLM-summarized rolling compression was a weekend rewrite that should have been the first feature. Everything built on top of truncation worked worse than it should have.

2. **Measure token budgets from day one.** We didn't track per-node token consumption until month four. When we finally did, we discovered that the board composer was consuming 4,500 tokens per call — twice what it needed, because we were injecting the full component registry instead of only the components relevant to the current operation.

3. **Don't filter context — constrain outputs.** The assistant-message filtering incident taught us that removing information from context creates blind spots. Constraining the output space (removing `clarify` from the enum) solves the same problem without side effects.

4. **Design for lazy loading from the start.** The field whitelist problem could have been avoided if we'd designed the context assembly pipeline with lazy loading in mind. Instead, we initially assembled all context eagerly and only later refactored to per-node injection.

5. **The best context is no context.** The zero-LLM intents (`remove_node`, `reorder`, `restore_preset`) need no context at all beyond the classification output. Every intent we can handle without an LLM call is an intent that can't be confused by context.

---

## Principles

Anthropic's context engineering post describes the challenge of "curating what will go into the limited context window from that constantly evolving universe of possible information." After eight months of production, we'd distill our experience into five principles:

1. **Context is not one thing.** It's short-term memory, long-term memory, grounding vocabulary, examples, schema descriptions, and conversation history. Each has different trust levels, different token costs, and different retrieval strategies. Treat them as separate resources.

2. **Exclude aggressively, include lazily.** The default should be empty context, with each component justified by a measurable improvement. The field whitelist, the full MQL, the complete example corpus — these were all removed or made lazy, and quality improved each time.

3. **Compress, don't truncate.** Truncation is lossy in the worst way — it loses the oldest context, which is often the most important (initial filters, established preferences, the "setup" of a multi-turn interaction). Summarization preserves the signal while reducing the tokens.

4. **Don't solve context problems by removing context.** When something in the context causes bad behavior, the instinct is to remove it. The better approach is to constrain the output space so the bad behavior is impossible regardless of context. This preserves the information for cases where it's needed.

5. **Each node gets exactly what it needs.** A monolithic context that serves all nodes is a context that serves none of them well. Assembly should be per-node, with each node's context budget justified by its specific task.

The gap between "prompt engineering" and "context engineering" is the gap between writing a good instruction and designing a system that consistently assembles the right information for the right component at the right time. The instruction matters. The assembly matters more.

---

*The system described serves ~300 enterprise users across four business domains. All data remains on-premise in compliance with PIPL. Context strategies are described generically; no internal identifiers or proprietary content are shared.*
