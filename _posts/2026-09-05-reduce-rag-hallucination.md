---
layout: default
title: "7 Production Techniques That Cut Our RAG Hallucination Rate by 80%"
date: 2026-09-05
categories: [RAG, LLM, Hallucination, Enterprise AI]
---

# 7 Production Techniques That Cut Our RAG Hallucination Rate by 80%

*September 5, 2026 · 18 min read*

Our enterprise AI workbench lets users ask questions in natural language and get data dashboards in return. "Show me active contract count by region" should produce a chart with real numbers from real data. When it hallucinates — invents a field name, picks the wrong metric, or maps data to the wrong chart axis — users see a professional-looking dashboard with wrong numbers. That's worse than no answer at all.

Over eight months in production, we reduced our hallucination rate from ~35% of data queries to under 7%. Not with a single breakthrough, but with seven interlocking techniques — each addressing a different failure mode. Here's what we built, why it works, and the failure that motivated each one.

![Figure 1. Seven-Layer Defense Pipeline](/assets/images/fig4-defense-pipeline.svg)
*Figure 1. Each layer targets a specific failure mode. Combined, they reduce hallucination by ~80%.*

---

## Hallucination Is Not One Problem — It's Seven

Before diving into techniques, we need to reframe the problem. Most teams treat "hallucination" as a single issue and reach for a single solution: better prompts, bigger models, more retrieval context. That's like treating "the server is slow" as one problem when it could be CPU, memory, disk I/O, network, or a bad query.

Our hallucination taxonomy:

| Failure Mode | Symptom | % of Total Failures |
|---|---|---|
| **Structural** | Malformed JSON, missing fields, wrong types | ~15% |
| **Semantic** | Correct structure, wrong business meaning | ~25% |
| **Retrieval** | Right metric exists but wasn't found | ~10% |
| **Classification** | Query routed to wrong handler | ~8% |
| **Mapping** | Query runs but chart shows empty/wrong data | ~12% |
| **Type confusion** | Count vs. rate, absolute vs. relative | ~5% |
| **Regression** | Previously correct query now fails after update | ~5% |

Knowing *which* 80% of failures fall into *which* categories lets you build targeted fixes instead of hoping one silver bullet works.

---

## How a Query Flows Through the Pipeline

Before the individual techniques, here's how they connect:

![Figure 2. Query Flow Decision Tree](/assets/images/fig6-query-flow.svg)
*Figure 2. A user query passes through intent routing, semantic retrieval, constrained generation, validation, and feedback — seven checkpoints before reaching the user.*

---

## Technique 1: Structured Output Enforcement

### The Failure

Early in development, the LLM generated free-form JSON for query plans. It mostly worked — until it didn't:

```json
// What we expected:
{"collection": "contracts", "pipeline": [...], "chart_type": "bar"}

// What we got on a bad day:
Here's the query plan:
```json
{"collection": "contracts", "pipeline": [...], "chart_type": "bar"}
```
The pipeline queries the contracts collection and groups by region...
```

The LLM helpfully wrapped the JSON in a markdown code fence and added an explanation. Our JSON parser choked. We added regex to strip fences. Then the LLM started using single quotes. We handled that. Then it occasionally output YAML-ish syntax. We handled that too. Each fix was a band-aid over a design flaw: **we were asking for structured data through an unstructured channel.**

### The Fix

We adopted `complete_typed(schema_model=PydanticModel)` — constrained decoding that forces the LLM to produce valid JSON conforming to a predefined Pydantic schema. Invalid field names are rejected at generation time. Missing required fields are caught before the response leaves the LLM.

```python
# Before: hope and pray
raw = await llm.complete(messages)
try:
    plan = json.loads(raw)  # might fail
except json.JSONDecodeError:
    plan = json.loads(strip_markdown_fences(raw))  # might still fail

# After: schema-constrained
plan = await llmproxy_client.complete_typed(
    model, messages, 
    schema_model=SemanticQueryPlan,
    temperature=0.0,
    parse_json=query_service.parse_llm_json  # fallback parser
)
# `plan` is always a valid SemanticQueryPlan or an explicit error
```

**How `complete_typed` works internally:** it injects the Pydantic model's JSON schema into the LLM request, constraining the decoder to only produce tokens that form valid JSON matching the schema. When validation fails (the model occasionally ignores the schema constraint), the Pydantic error is fed back to the model for up to 2 retries. This "validation-as-retry-prompt" pattern catches edge cases where the model produces valid JSON that violates semantic constraints (e.g., `{"month": "now.month"}` — syntactically valid, semantically wrong).

> 💡 **Takeaway for practitioners:** If your LLM produces data that any downstream code will parse, use constrained decoding — not regex postprocessing. The cost is near-zero (schema injection adds ~100 tokens to the prompt). The benefit is eliminating an entire class of failures.

### Impact

Eliminated ~15% of all hallucination incidents — the entire class of "syntactically invalid output" failures. The remaining 85% were semantically wrong but structurally valid, which required different techniques.

---

## Technique 2: Few-Shot Anchoring with Human-Approved Examples

### The Failure

A user asked: "How many contracts were successfully activated?" The system generated a query against the `contracts` collection with `status: "成功"`. The query ran. The chart rendered. The number was wrong — "successfully activated" means `status: "启用"` in our data model, not `status: "成功"`. The AI made a reasonable guess that happened to be incorrect.

### The Fix

We built an **example store** — a library of (question, approved_query) pairs with a two-stage approval workflow:

```
                    ┌─────────┐     human      ┌───────────┐     human      ┌───────────┐
  AI generates  ──▶ │  Draft  │ ──────────────▶ │  Pending  │ ──────────────▶ │ Published │
                    └─────────┘    review       └───────────┘   approval     └───────────┘
                         │                                                         │
                         │         ❌ NEVER auto-promotes                          │
                         └─────────────────── ✕ ──────────────────────────────────┘
                                                                        ▼
                                                                 Enters AI context
```

Only published examples enter the AI's context window. The injection strategy matters: we format examples as fake conversation history (user asked X, assistant answered Y), not as system prompt documentation. In our testing, the conversation format outperformed system prompt injection by ~20% on query accuracy — the model treats its own "previous answers" as stronger precedent than instructions.

```python
# Inject as conversation history, not system instructions
few_shot_messages = []
for example in approved_examples:
    few_shot_messages.append({"role": "user", "content": example.question})
    few_shot_messages.append({"role": "assistant", "content": json.dumps(example.plan)})

# Actual user question comes after the examples
messages = system_prompt + few_shot_messages + [{"role": "user", "content": user_query}]
```

The anti-pattern we explicitly forbid: **auto-approving examples that "work."** A query that returns data is not necessarily a query that returns *correct* data. AI-generated queries are saved as drafts, but they never auto-promote. A domain expert must verify the business semantics before an example can influence future queries.

> 💡 **Design reference:** This is adapted from [Vanna.ai](https://vanna.ai/)'s training-data closed loop (`VannaBase.add_question_sql`), with one critical difference — Vanna auto-promotes successful queries; we require human approval. In enterprise settings where wrong numbers can cause business decisions, the human gate is non-negotiable.

### Impact

Reduced semantic hallucinations (correct structure, wrong meaning) by ~25%. The improvement scales with the example library — each new approved example makes similar future queries more accurate.

---

## Technique 3: Bigram Coverage Scoring for Chinese Term Matching

### The Failure

A user asked about "启用中合同数占比" (ratio of currently-active contracts). The system failed to match it against the registered term "启用合同数" (active contract count). Why? Substring matching: `"启用合同数" not in "启用中合同数占比"` evaluates to `True` because the extra character "中" breaks the substring.

The term wasn't retrieved. The AI was told "this metric isn't registered." The user got a "sorry, I can't find that metric" response — for a metric that existed in the system.

### The Fix

We replaced substring matching with **bigram coverage scoring**: decompose both the user's question and the registered term into overlapping character pairs, then measure what fraction of the term's bigrams appear in the question.

```
Input:   question = "启用中合同数占比"
         term     = "启用合同数"

Step 1:  term bigrams  = {启用, 用合, 合同, 同数}        ← 4 pairs
Step 2:  query bigrams = {启用, 用中, 中合, 合同, 同数, 数占, 占比}  ← 7 pairs
Step 3:  overlap       = {启用, 合同, 同数} + {用合}*     ← wait, 用合 is NOT in query
         actually:  overlap = {启用, 合同, 同数}          ← 3/4 = 75% coverage

* Correction: "用合" not present because query has "用中" and "中合" but not "用合".
  Real coverage = 3/4 = 75%. Still well above threshold. ✅
```

```python
def term_score(question: str, term: str) -> int:
    q = normalize(question)  # NFKC + lowercase + strip punctuation
    t = normalize(term)
    if q == t:
        return 100  # exact match — only case that can short-circuit
    
    term_bigrams = {t[i:i+2] for i in range(len(t)-1)}
    query_bigrams = {q[i:i+2] for i in range(len(q)-1)}
    coverage = len(term_bigrams & query_bigrams) / len(term_bigrams)
    
    # Score scales with term length (longer terms = more informative)
    return int(coverage * min(90, 30 + 4 * len(t)))
```

Key design decisions:

| Decision | Reason | What We Avoided |
|---|---|---|
| **Coverage over Jaccard** | Jaccard is diluted by long queries; a 4-char term in a 20-char question → low Jaccard | Symmetric similarity that punishes short terms |
| **Score ∝ term length** | "合同" (2 chars) matches everything; "从来没启用的合同" (8 chars) is specific | Short generic terms dominating the ranking |
| **Only exact match = shortcircuit** | High score = "candidate"; exact match = "confirmed" | Premature shortcircuit on fuzzy matches |
| **Max score, not sum** | Prevents "more aliases in YAML = higher rank" | Rewarding metadata verbosity over relevance |
| **Per-kind quota** | 4 metrics + 2 derived + 3 dimensions per query (not global top-k) | Metrics crowding out dimensions (or vice versa) |

### Impact

Recall for Chinese business terms improved from ~72% to ~94%. False negatives (telling users a metric doesn't exist when it does) dropped by ~75%.

---

## Technique 4: Dual-Use Intent Routing (Vector Shortcircuit + Few-Shot Fallback)

### The Failure

Our intent classifier was an LLM call: "Given this user message, classify the intent as one of [create_dashboard, apply_filter, export, clarify, ...]." It cost ~500ms and 200 tokens per classification. With 11 intents and growing, the prompt was getting long, and accuracy was dropping as edge cases accumulated.

We added a vector-based router: embed the user message, compare against a library of labeled examples (~155 utterances), and if the top match is confident enough, skip the LLM call entirely. This worked 37% of the time (short-circuit) and saved tokens.

The problem: **the other 63% of the time, we threw away the vector results and called the LLM cold.** The retrieval had already found the most similar examples. Why weren't we using them?

### The Fix

```
                          ┌─────────────┐
                          │  User Query │
                          └──────┬──────┘
                                 ▼
                     ┌───────────────────────┐
                     │ embed(user_message)    │  ← one embedding call
                     └───────────┬───────────┘
                                 ▼
                     ┌───────────────────────┐
                     │ cosine_search(top-5)   │  ← in-memory, <1ms
                     └───────────┬───────────┘
                          ┌──────┴──────┐
                     score > 0.92       score ≤ 0.92
                     & top-3 agree      or disagree
                          │                  │
                     ┌────┴────┐       ┌─────┴──────┐
                     │ Short-  │       │ LLM + top-k│
                     │ circuit │       │  as few-shot│
                     │ (37%)   │       │   (63%)    │
                     │ ~200ms  │       │  ~700ms    │
                     │ 0 token │       │  ~150 token│
                     └─────────┘       └────────────┘
                                            │
                                    retrieval NEVER wasted
```

**An important constraint:** only intents with no required slots can short-circuit. If an intent requires extracting parameters (e.g., "filter by region=East" requires extracting "East"), the LLM must run to extract them. The intent catalog enforces this at startup — it's a compile-time invariant, not a runtime check.

**Resilience pattern:** When adding a corrected example for a misclassified intent, the fix is data-only: add the utterance to the intent corpus, re-embed, done. No code change, no prompt engineering, no redeployment.

> 💡 **Design reference:** This dual-use pattern is inspired by [RAGFlow](https://github.com/infiniflow/ragflow)'s `CategorizeParam.update_prompt()`, which injects all examples into the system prompt. Our difference: RAGFlow has ~5 categories with ~3 examples each (inject all); we have 11 intents with ~15 examples each (inject only top-k to avoid diluting attention).

### Impact

Intent classification accuracy improved from 81% to 93%. Token cost dropped ~30% (37% short-circuit rate + cheaper few-shot prompts for the rest). Latency for short-circuited cases: ~200ms vs ~700ms.

---

## Technique 5: Output Field Validation (The "Null Chart" Killer)

### The Failure

A user asked for "contract count by insurance type." The AI generated this pipeline:

```javascript
[
  { $group: { _id: "$insurance_type_cn", count: { $sum: 1 } } }
]
```

And this data mapping:

```json
{ "name": "insurance_type_cn", "value": "count" }
```

The pipeline is valid MQL. It runs. It returns results. But `$group` outputs `_id` and `count` — there is no field called `insurance_type_cn` in the output. The chart tried to read `insurance_type_cn`, got `null` for every row, and rendered a chart with empty labels.

The query "succeeded." The chart rendered. The labels were all blank. The user saw a professional-looking chart that communicated nothing.

### The Fix

After the LLM generates a query plan, a **six-gate validator** checks structural correctness before execution:

```
 LLM Output
     │
     ▼
 ┌───────────────────────┐
 │ G1: Component Whitelist│ ← Only registered chart types
 ├───────────────────────┤
 │ G2: Required Props     │ ← Each chart type's mandatory fields
 ├───────────────────────┤
 │ G3: Data Key Source    │ ← props references must trace to pipeline
 ├───────────────────────┤
 │ G4: MQL Security       │ ← Collection whitelist + forbidden ops
 ├───────────────────────┤
 │ G5: Chart Role Check   │ ← Pie needs name+value; bar needs category+value
 ├───────────────────────┤
 │ G6: Field Mapping      │ ← data_mapping fields MUST exist in pipeline output
 └───────────┬───────────┘
        PASS │        │ FAIL
             ▼        ▼
         Execute    Error → feed back to LLM → retry (max 2)
```

The G6 check — the "null chart killer" — specifically:

```python
def mapping_output_errors(pipeline, data_mapping):
    output_fields = infer_output_fields(pipeline)  # last $group or $project
    if output_fields is None:
        return []  # can't infer → skip, don't false-positive
    
    missing = [f for f in data_mapping.values() if f not in output_fields]
    if missing:
        return [f"data_mapping references {missing}, "
                f"but pipeline only outputs {output_fields}"]
    return []
```

When validation fails, the error message is fed back to the LLM for a retry. The LLM sees "you mapped `insurance_type_cn` but the pipeline only outputs `_id` and `count`" and adds a `$project` stage to rename `_id` to `insurance_type_cn`. The second attempt almost always succeeds.

### Impact

"Null chart" incidents dropped from ~18% of dashboard queries to under 2%.

---

## Technique 6: Ratio vs. Count Disambiguation

### The Failure

A user asked: "How many contracts were successfully activated?" The AI returned 87.3%. The user wanted a count, not a percentage. In our metric catalog, both "activation count" and "activation rate" contain the word "activation." The rate metric had a higher semantic similarity score, so the AI picked it.

### The Fix

An explicit signal detector scans the user's question for **ratio indicators** and **count indicators**:

```
Question: "启用了多少合同?"
                    ──────
                    count signal: "多少" → demote rate metrics, boost count metrics

Question: "启用合同的占比是多少?"
                      ────
                      ratio signal: "占比" → demote count metrics, boost rate metrics

Question: "启用合同情况"
           (no signal)  → don't intervene, let semantic scoring decide
```

```python
def detect_quantity_type(question: str) -> str | None:
    ratio_terms = {"率", "比例", "占比", "百分比", "percentage", "ratio"}
    count_terms = {"多少", "几个", "数量", "个数", "总数", "笔数"}
    
    has_ratio = any(t in question for t in ratio_terms)
    has_count = any(t in question for t in count_terms)
    
    if has_ratio and not has_count: return "ratio"
    if has_count and not has_ratio: return "count"
    return None  # ambiguous — don't intervene
```

> 💡 **Why not let the LLM figure this out?** It usually does. But "usually" means ~92% of the time. For the 8% where it fails, the user gets a convincingly wrong answer. The signal detector catches the easy cases (explicit "多少" or "率") so the LLM only needs to handle genuinely ambiguous ones.

### Impact

Ratio/count confusion dropped from ~8% of queries to under 1%. The fix is embarrassingly simple — the signal was always there in the user's question; we just weren't looking for it.

---

## Technique 7: The Evaluation Loop (Closing the Feedback Cycle)

### The Failure (Meta-Failure)

We had no systematic way to measure whether our hallucination rate was improving. Individual fixes were motivated by user complaints. We'd fix a bug, deploy, and wait for the next complaint. We had no regression tests, no accuracy benchmarks, no way to know if a model upgrade helped or hurt.

### The Fix

A three-layer evaluation system:

```
                    ┌──────────────────────────────────────────┐
                    │           Layer 1: Real-Time Gate         │
                    │  Board Validator runs on EVERY query      │
                    │  Invalid dashboards NEVER reach the user  │
                    └──────────────────────┬───────────────────┘
                                           │
                    ┌──────────────────────▼───────────────────┐
                    │         Layer 2: Nightly Calibration      │
                    │  ~50 canonical queries × real data        │
                    │  Assert structural properties, not values │
                    │  Regression → alert → investigate         │
                    └──────────────────────┬───────────────────┘
                                           │
                    ┌──────────────────────▼───────────────────┐
                    │         Layer 3: Human Feedback Loop      │
                    │  User flags "wrong" → draft created       │
                    │  Expert reviews weekly → corrects query   │
                    │  Published → enters AI context            │
                    │  AI improves → fewer flags → cycle closes │
                    └──────────────────────────────────────────┘
```

```python
# Example calibration case (Layer 2)
{
    "question": "按险种看合同数",
    "expect": {
        "collection": "contracts",
        "chart_type": "bar",
        "output_must_contain": ["insurance_type", "count"],
        "data_series_count": {"min": 2, "max": 20}
    }
}
```

**Why structural assertions, not value assertions?** The data changes daily. "Contract count = 42" is wrong tomorrow. But "the chart type is bar, the output contains insurance_type and count, and there are between 2 and 20 data series" remains valid regardless of data changes. Structural assertions are stable over time.

### Impact

The evaluation loop itself doesn't reduce hallucinations — it makes all other techniques measurable and accountable. Without it, we were guessing. With it, we can quantify: Technique X reduced failure rate by Y% on the calibration suite. Model upgrade Z regressed on 3 out of 50 cases, specifically in date-range parsing.

---

## The 80% Reduction, Visualized

![Figure 3. Waterfall: Hallucination Rate Reduction](/assets/images/fig5-waterfall.svg)
*Figure 3. Starting from a 35% baseline, each technique chips away at a different failure mode until we reach ~7%.*

![Figure 4. Quality Dimensions: Before vs After](/assets/images/fig7-radar.svg)
*Figure 4. Radar view across 7 quality dimensions. The biggest gains: regression detection (20% → 95%), semantic accuracy (55% → 80%), and field mapping (82% → 98%).*

---

## Metrics Dashboard: What to Monitor

We track hallucination cost across three layers:

### Business Layer (impact)
| Metric | Target | Why |
|---|---|---|
| `hallucination_rate` | < 7% | Core quality metric |
| `user_flag_rate` | < 2% per week | Users catching errors we missed |
| `wrong_number_incident` | 0 per month | Financial data errors — highest severity |

### System Layer (prevention)
| Metric | Target | Why |
|---|---|---|
| `validation_reject_rate` | 10-20% | Too low = validators not catching enough; too high = LLM quality degraded |
| `retry_success_rate` | > 80% | Validates that error-feedback-retry pattern works |
| `example_store_coverage` | > 60% | % of queries that have a relevant approved example |
| `intent_shortcircuit_rate` | 30-40% | Measures vector router effectiveness |

### Request Layer (per-query)
| Metric | Target | Why |
|---|---|---|
| `bigram_top_score` | > 50 | Low scores mean no relevant terms found — higher hallucination risk |
| `few_shot_count` | ≥ 2 | Queries with 0 examples hallucinate 3x more |
| `field_mapping_pass` | true | Binary gate — must pass before execution |
| `quantity_type_signal` | detected | Queries without ratio/count signal → flag for review |

> 💡 **Practical tip:** Build the metrics dashboard *before* implementing the techniques. Like the iThome LLMOps series emphasizes — "observability before optimization." You can't improve what you can't measure, and the debugging time saved in the first month pays for the dashboard setup 10x over.

---

## Implementation Roadmap

If you're starting from zero anti-hallucination infrastructure, here's the order we'd recommend:

| Phase | Timeline | What | Why This Order |
|---|---|---|---|
| **Phase 1** | Week 1 | Structured output (`complete_typed`) | Eliminates 15% of failures for ~2 hours of work |
| **Phase 2** | Week 1-2 | Output field validation (6-gate validator) | Catches the most user-visible failures (empty charts) |
| **Phase 3** | Week 2-3 | Evaluation loop (calibration suite) | You need measurement before you can claim improvement |
| **Phase 4** | Week 3-4 | Few-shot example store | Biggest single-technique improvement (~25%) but needs data |
| **Phase 5** | Week 4-5 | Bigram scoring + ratio/count detection | NLP-specific; skip if your terms are English |
| **Phase 6** | Week 5-6 | Dual-use intent routing | Optimization — reduces cost *and* improves accuracy |
| **Phase 7** | Ongoing | Feedback loop + example curation | Compound returns — each week the system gets smarter |

**Quick wins first:** Phases 1-2 take ~1 week and eliminate ~27% of hallucinations. Phase 3 gives you the ability to measure everything that follows. Phases 4-6 are the heavy lifting. Phase 7 is the flywheel that keeps improving forever.

---

## The Hidden Cost of Hallucination

Following the three-layer cost evaluation framework:

```
Hallucination cost ≠ just "wrong answer"

Business cost:   wrong financial numbers → bad decisions → trust erosion
                 user sees 87.3% when answer is 1,247 → acts on wrong data

System cost:     retry loops (validator fail → re-generate → re-validate)
                 human review of flagged dashboards (~15 min each)
                 example curation (domain expert time)

Request cost:    wasted tokens on hallucinated responses
                 wasted compute on queries that run but produce garbage
```

In our case: **each hallucination incident costs ~45 minutes** (user notices → reports → engineer investigates → fixes root cause). At a 35% hallucination rate with ~200 data queries/day, that's ~70 incidents/day × 45 min = ~52 engineer-hours/day. Not sustainable. At 7%, it's ~14 incidents × 45 min = ~10.5 hours — manageable with one dedicated person reviewing flags.

---

## What We Haven't Solved

Honesty requires listing what still fails:

1. **Novel vocabulary.** When a user invents a term we've never seen ("synergy contracts"?), no amount of bigram matching helps. The system correctly identifies that it doesn't recognize the term, but users find "I don't know that metric" frustrating.

2. **Multi-hop reasoning.** "Show me the contracts that were activated after the payment was confirmed, grouped by the original sales rep" requires joining across collections and understanding temporal ordering. We handle simple cross-references but not arbitrary multi-hop queries.

3. **Ambiguous aggregation.** "Show me contract performance" — performance by what metric? Over what time period? Grouped by what? We can ask clarifying questions, but users prefer a "just show me something reasonable" experience we can't reliably deliver.

4. **Model regression.** When our LLM provider pushes an update, our calibration suite sometimes detects regressions. We can detect them quickly now (Technique 7), but we can't prevent them. Pinning model versions helps but limits improvement.

---

## Principles, Not Just Techniques

If there's a generalizable lesson, it's this: **RAG hallucination is not one problem. It's at least seven different failure modes that happen to produce the same symptom (wrong output).** Treating it as one problem leads to one solution (better prompts, bigger models, more retrieval) that improves some cases and ignores others.

Each technique in this post addresses a specific failure mode with a specific mechanism:

| Failure Type | Fix Category | Analogy (SRE) |
|---|---|---|
| Structural failures → constrained decoding | **Input validation** | Schema enforcement at API boundary |
| Semantic failures → human-approved precedent | **Ground truth anchoring** | Golden dataset for monitoring |
| Retrieval failures → better scoring algorithms | **Search quality** | Tuning Elasticsearch relevance |
| Classification failures → dual-use retrieval | **Routing optimization** | Load balancer with health checks |
| Mapping failures → output validation | **Contract testing** | Consumer-driven contract tests |
| Type confusion → explicit signal detection | **Input preprocessing** | Request normalization middleware |
| Measurement failure → automated evaluation | **Observability** | SLI/SLO dashboards |

The 80% reduction came from identifying which 80% of failures fell into which categories, and building a targeted fix for each. The remaining 20% requires harder solutions — or, honestly, accepting that some queries are too ambiguous for any system to answer correctly without clarification.

---

## References

- [Vanna.ai](https://vanna.ai/) — Few-shot example store pattern (training-data closed loop)
- [RAGFlow](https://github.com/infiniflow/ragflow) — Categorize component with example injection
- [Economic Evaluation of LLMs](https://arxiv.org/abs/2502.00571) — Three-layer cost framework
- [Learning to Generate Structured Output with Schema Reinforcement Learning](https://arxiv.org/abs/2502.00898)
- [PROMPT CACHE: Modular Attention Reuse for Low-Latency Inference](https://arxiv.org/abs/2311.04934)

---

*The system described processes natural language queries against MongoDB in a Chinese-language enterprise environment serving ~300 users. All techniques are described generically; no internal identifiers or proprietary code are shared.*
