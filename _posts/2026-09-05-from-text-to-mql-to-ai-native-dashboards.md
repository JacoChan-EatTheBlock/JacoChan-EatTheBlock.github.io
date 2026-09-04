---
layout: default
title: "From Text-to-MQL to AI-Native Dashboards: Five Architectural Shifts for Production Data Intelligence"
date: 2026-09-05
categories: [AI Architecture, Data Dashboard, MongoDB, Semantic Layer]
---

Real users don't issue one instruction at a time. "Add a pie chart for insurance types and move the billing card to the top" is two operations. V3.0 includes a Compound Intent Planner that decomposes

```json
{
  "tasks": [
    { "task_type": "board_action", "intent": "add_node", "question": "按险种看合同分布" },
    { "task_type": "board_action", "intent": "reorder", "targets": ["billing_card"], "position": "top" }
  ]
}
```

Dependencies and conflicts are resolved by **deterministic code**, not the LLM:
- `add_node` + `reorder` on the same target → `reorder` depends on `add_node` (the card must exist first)
- `remove_node` + `update_node` on the same target → conflict (can't modify what you're deleting)
- `override` + any other board action → conflict (override wipes everything)

The LLM proposes *what* to do; code decides *when* and *whether* to do it. This separation keeps the execution semantics deterministic even when the intent decomposition is probabilistic.

### The two branches: why this split matters

The intent layer's most important design decision is the hard split between **Dashboard Mutations** and **Data Intelligence**:

| Property | Dashboard Mutations (7 intents) | Data Intelligence (4 intents) |
|----------|-------------------------------|-------------------------------|
| **Side effects** | Modifies the right-side panel layout | Read-only; no layout changes |
| **LLM involvement** | `add_node`, `update_node`, `override`, `plan_board` invoke `board_composer` (LLM generates component specs) | `query_data` invokes the semantic subgraph (IR → compiler → MQL); `explain`/`analyze` call LLM for text |
| **Zero-LLM intents** | `remove_node`, `reorder`, `restore_preset` are pure code — the intent router identifies *what* to do, code executes it without any LLM generation | `filter_change` can be zero-LLM when slot values are already extracted by the router |
| **Shortcircuit eligible** | ❌ Never (write operations need evidence) | ✅ `query_data`, `analyze`, `chitchat` (read-only, recoverable) |
| **Interrupt (human confirm)** | `override`, `restore_preset` (destructive) | Never |
| **Validation gate** | 6-gate `board_validator` before any dashboard reaches the user | Semantic plan compilation (deterministic) |
| **Error recovery** | Validator fail → error fed back to LLM → retry (max 2) | Compilation fail → fallback to adhoc MQL with extra guardrails |

This split exists because **the cost of error is asymmetric.** A wrong number in a `query_data` response is bad but correctable (the user asks again). A wrong `override` that wipes a carefully curated dashboard and replaces it with garbage is catastrophic — the user loses their work.

The principle: **talk is cheap; actions are expensive.** Read operations can afford to trust the model more. Write operations must have code-verifiable evidence at every step.

layout: default
title: "From Text-to-MQL to AI-Native Dashboards: Five Architectural Shifts for Production Data Intelligence"
date: 2026-09-05
categories: [AI Architecture, Data Dashboard, MongoDB, Semantic Layer]
---

# From Text-to-MQL to AI-Native Dashboards: Five Architectural Shifts for Production Data Intelligence

*September 5, 2026 · 22 min read*

**Key Takeaways**

- Letting an LLM write raw MongoDB aggregation pipelines against physical tables gives you a 35% hallucination rate. Reducing the AI's degrees of freedom — through semantic views, constrained vocabularies, and multi-gate validation — brings that number under 7%.
- The architectural pattern that matters most is **additive collapse**: using `$set` instead of `$project` in MongoDB views so that business logic is encoded without destroying the original fields. The AI operates on boolean labels for macro questions and raw columns for micro drill-downs.
- Five shifts get us from a naive text-to-MQL prototype to a production system: physical tables to semantic views, free-form generation to constrained vocabulary, single-pass to multi-gate pipeline, metric lookup to precedent-based generation, and schema to semantics.

---

For most of its history, business intelligence required a dedicated analyst to translate a manager's question into a SQL query, validate the results, build a chart, and email the PDF. The entire cycle took hours to days. The promise of AI-powered dashboards is to compress that cycle to seconds: a user types a question in natural language, and the system produces an interactive chart backed by real data.

We built that system. It serves ~300 enterprise users, generates MongoDB aggregation pipelines from natural language, renders interactive dashboards, and persists them across sessions. Getting it to work in a demo took two weeks. Getting it to work in production — where wrong numbers create real business consequences — took three architectural rewrites over eight months.

This post describes the five shifts that took us from a 35% hallucination rate to under 7%. Each shift addresses a specific failure mode, and each one required abandoning an assumption we held in the previous generation.

![Figure 1. Three Architecture Generations](/assets/images/fig9-shift-evolution.svg)
*Figure 1. V1.0 let the AI write raw queries. V2.0 pushed business logic into database views. V3.0 added semantic routing, precedent-based generation, and continuous evaluation.*

---

## Where this starts: the naive approach

The first version did the obvious thing. A user types "show me confirmed bill count by region for July." The system sends the question to an LLM with a schema description of four MongoDB collections — `contracts`, `si_increase`, `si_decrease`, and `bills` — and asks it to produce a valid aggregation pipeline. The pipeline runs. The results render as a chart.

On good days, this worked. On bad days:

- The LLM invented field names that didn't exist in the schema. (`confirmed_count` instead of computing a count with `bill_status: "confirmed"`)
- It confused status codes across tables. (`status: "success"` when the correct field was `service_status: "declaration_success"`)
- It picked the wrong month field. (`bills` uses `create_month`, `si_increase` uses `social_month`, and the LLM guessed `query_month`.)
- It applied the wrong base population filter. Every meaningful query in our domain starts with a base population: `status=active AND business_type IN [type_a, type_b]`. The LLM forgot this ~40% of the time.

The measured hallucination rate was ~35% of data queries. Not 35% of queries returned errors — 35% returned data that looked correct but wasn't. A chart would render with confident labels and plausible numbers, and the numbers would be wrong.

We tried to fix this with a Python middleware layer (`layout_assembler.py`, 866 lines) that intercepted the LLM's output and injected the correct base population filters, status field mappings, and month bindings. This is the "correct it after it already hallucinated" pattern, and it had two fatal problems:

1. **Blast radius.** Every new metric required new correction rules. The middleware became a growing tangle of `if metric_name == "activation_rate"` branches. Adding a single new chart type touched 4–6 places in the middleware.

2. **Silent failures.** When the middleware's correction rules didn't match the LLM's output structure, the correction silently failed. The LLM would produce a pipeline shape the middleware didn't expect, the middleware would pass it through uncorrected, and the wrong numbers would reach the user.

The fundamental problem was not the LLM's capability. It was the **number of degrees of freedom** the LLM faced. Four tables, dozens of status codes with Chinese-language values, different month fields per table, and business-specific base population filters that varied by metric type. The search space was too large for reliable generation.

---

## Shift 1: From Physical Tables to Semantic Views

The first shift was the most impactful. Instead of exposing the raw physical tables to the AI, we created MongoDB views that encode business logic at the database level.

### The core insight: additive collapse

We studied how Cube.js, dbt, and MetricFlow implement semantic layers. They all share a pattern: define metrics and dimensions in a configuration layer that sits between the AI and the physical data. The AI requests metrics by name; the semantic layer generates the actual query.

For SQL databases, this is well-supported. For MongoDB aggregation pipelines, the ecosystem barely exists. We needed to build our own semantic layer, and we had a design choice: use `$project` or `$set` in our views.

**`$project` is destructive.** It selects specific fields and discards everything else. A view built with `$project` presents a clean, minimal interface — but the AI can never access the original fields for drill-down queries. If a user asks "show me confirmed bills" and then follows up with "break that down by customer name," the customer name field might have been projected away.

**`$set` is additive.** It adds new fields without removing existing ones. The view becomes a superset of the original table — every original column survives, plus new semantic labels layered on top.

We chose `$set` exclusively. The three base views:

```
Physical Tables                 Semantic Views ($set, not $project)
┌─────────────┐
│ contracts   │  ──────────▶  v_contracts_base
│ (status_cn, │                + is_active: boolean
│  biz_type,  │                + is_stopped_after_active: boolean
│  auto_sw)   │                + (all original fields preserved)
└─────────────┘

┌─────────────┐
│ si_increase │  ──$unionWith──▶  v_declaration_base
│ si_decrease │                    + is_success: boolean
└─────────────┘                    + is_fail: boolean
                                   + unified_month: string
                                   + direction: "increase" | "decrease"
                                   + (all original fields preserved)

┌─────────────┐
│ bills       │  ──────────▶  v_billing_financial_base
│ (bill_status│                + is_confirmed: boolean
│  pay_status │                + is_paid: boolean
│  inv_status │                + is_invoiced: boolean
│  fees...)   │                + expected_service_fee: number
                               + actual_service_fee: number
                               + unified_month: string
                               + (all original fields preserved)
```

![Figure 2. Semantic Collapse: Before vs After](/assets/images/fig10-semantic-collapse.svg)
*Figure 2. Left: V1.0, the AI must understand status codes, compute subtractions, and pick the right month field. Right: V2.0+, the AI writes `{$match: {is_paid: true, unified_month: "2026-07"}}`.*

### The mathematics of collapse

What happens in these views is a many-to-one mapping. The `bills` table has `bill_status` with values like `"confirmed"`, `"pending"`, `"voided"`, `"cancelled"`. The view collapses this to a boolean: `is_confirmed = (bill_status == "confirmed")`. In set theory terms, we're partitioning the universe of rows into two subsets and labeling them.

The key property is **reversibility**. Because we use `$set`, the original `bill_status` field still exists. The boolean is a macro label for common questions ("how many confirmed bills?"), but the original field remains available for micro questions ("break down the non-confirmed bills by their actual status"). This is the difference between lossy compression (discarding information) and lossless augmentation (adding a layer on top).

For amounts, we pre-compute frequently confused calculations. `expected_service_fee = service_fee_total - prepaid_period_fee` is a subtraction the LLM got wrong ~15% of the time (it would forget to subtract the prepaid component). Moving this into the view means the AI never needs to perform the subtraction — the field already contains the correct number.

### What we gained

- **Hallucination rate dropped from ~35% to ~18%.** The single biggest improvement. The AI no longer needs to remember base population filters, status code mappings, or month field names.
- **The middleware shrank.** We removed ~66 lines of semantic correction logic from `layout_assembler.py`. The remaining ~800 lines handle execution orchestration, data mapping, and provenance — things that belong in middleware.
- **New metrics became cheaper.** Adding a new business metric meant adding a `$set` field to an existing view, not writing middleware correction rules.

### What we accepted

- **View maintenance is manual.** Unlike Cube or dbt, which have CLI tooling for managing metric definitions, our MongoDB views are maintained by hand. At our scale (~300 users, 3 views), this is manageable. At 10x scale, we'd need tooling.
- **Views add query overhead.** Each view executes its `$set` pipeline for every query. For our data volumes (hundreds of thousands of rows, not millions), the overhead is negligible (~50ms). For large-scale deployments, materialized views or on-demand computation would be necessary.
- **We rejected Cube and dbt.** These are excellent tools for SQL-based semantic layers. They don't translate well to MongoDB aggregation pipelines, and deploying an external semantic layer middleware for 300 users adds infrastructure complexity that exceeds its value. MongoDB views are native, require no additional infrastructure, and cost zero operational overhead.

---

## Shift 2: From Free-Form Generation to Constrained Vocabulary

Semantic views reduced hallucination by shrinking the AI's world. But the AI could still generate arbitrary pipelines against those views — including pipelines that reference fields that don't exist, chart types we don't support, or data mappings that don't match the pipeline output.

The second shift was constraining the vocabulary: defining exactly what the AI is allowed to say.

### The component registry

Every chart type in our system is registered in a whitelist with explicit metadata:

```python
# Simplified component registry entry
{
    "GroupPie": {
        "dataBound": True,      # requires a data query
        "requiredProps": ["title"],
        "requiredRoles": {"name": "category field", "value": "numeric field"},
        "description": "Pie chart showing proportional distribution",
        "llm_generatable": True  # preset-only components are False
    }
}
```

The AI can only select from registered components. Each component declares which `data_mapping` roles it needs (`name` + `value` for a pie chart, `category` + `value` for a bar chart). This is injected into the prompt as grounding context — the AI knows exactly what shapes of output it must produce.

### Finite modeling: four question patterns

The critical insight came from analyzing six months of user queries. Despite the apparent diversity of questions, they decompose into only four patterns:

| Pattern | Example | View Operation |
|---------|---------|---------------|
| **Existence** | "How many active contracts?" | `{$match: {is_active: true}}` → `{$count}` |
| **Count** | "Contract count by region" | `{$match: {...}}` → `{$group: {_id: "$region", count: {$sum: 1}}}` |
| **Amount** | "Total expected service fee" | `{$match: {...}}` → `{$group: {_id: null, total: {$sum: "$expected_service_fee"}}}` |
| **State aggregation** | "Payment progress by month" | `{$match: {...}}` → `{$group}` → ratio computation |

New questions are new *combinations* of existing fields and patterns, not new patterns. We don't need a view per question (infinite); we need views per data model with atomic building blocks (boolean flags, counts, amounts) that compose.

### Structured output enforcement

Every LLM call that produces a query plan uses `complete_typed(schema_model=PydanticModel)` — constrained decoding that forces the output to conform to a predefined JSON schema. The schema declares exactly which intents, chart types, and field names are valid. Invalid tokens are rejected at generation time.

In V3.0, the LLM doesn't write MQL at all. It writes a **Semantic Query Plan** — a high-level intermediate representation (IR) declaring *what* to retrieve (which view, which metric, which dimension, which filters) without specifying *how* to retrieve it. A deterministic compiler (`compiler.compile_plan()`) translates the IR into a MongoDB aggregation pipeline. The compiler is pure Python — no LLM involvement, no hallucination risk. This is the core idea from the kevin.md design proposal that we fully implemented: separate intent (what) from execution (how), and let deterministic code own the execution.

This eliminates an entire class of failures: the LLM cannot produce a chart type that doesn't exist in the registry, a field name that isn't in the schema, or a pipeline structure that violates our constraints.

### One MQL per component, not `$facet`

A dashboard might contain 4–6 charts. The natural MongoDB approach is `$facet` — one aggregation pipeline with multiple sub-pipelines, one per chart. We considered this and rejected it:

1. **Blast radius.** If one sub-pipeline has a syntax error, the entire `$facet` fails. All charts go blank. With independent queries, a single failure affects only one chart — the others render normally.

2. **Index usage.** `$facet` sub-pipelines cannot efficiently use indexes the way top-level `$match` stages can. Independent queries each start with their own `$match` and benefit from index optimization.

3. **Size limits.** `$facet` output is a single BSON document, subject to MongoDB's 16MB document size limit. Independent queries have no such constraint.

We execute one MQL per component in parallel, with per-component error isolation:

```
Dashboard with 4 components:
  Component A (KPI card)  ──→ MQL_A ──→ execute ──→ ✅ render
  Component B (pie chart)  ──→ MQL_B ──→ execute ──→ ❌ error → show error state
  Component C (bar chart)  ──→ MQL_C ──→ execute ──→ ✅ render
  Component D (table)      ──→ MQL_D ──→ execute ──→ ✅ render
  
  Result: 3 of 4 charts render correctly. User sees which one failed.
```

This is more network round-trips than `$facet` but dramatically better fault tolerance. For dashboards where correctness matters more than latency, independent queries are the right trade-off.

---

## Shift 3: From Single-Pass to Multi-Gate Pipeline

V1.0 was a single pass: user question → LLM → pipeline → execute → render. V3.0 is a multi-stage pipeline with explicit checkpoints. This shift is arguably the most design-intensive — it determines not just *what* the AI does, but *what the AI is allowed to do* and *what evidence it must provide before acting*.

![Figure 3. Intent Layer Architecture](/assets/images/fig11-intent-architecture.svg)
*Figure 3. The intent layer splits into two branches: Dashboard Mutations (7 intents that modify the right-side panel) and Data Intelligence (4 intents that are read-only). Write operations require slot evidence and optional human confirmation. Read operations flow to the semantic subgraph or text response.*

### The Intent Catalog: a single source of truth

Before V3.0, the definition of a single intent was scattered across **eight places** in the codebase: the Pydantic enum, the prompt template, the route-derivation function, a 9-branch `if` chain, four separate frozensets for target requirements, and the vector corpus seed file. Adding one intent meant editing eight files. Inevitably, they'd drift.

We consolidated everything into a single `INTENTS` dictionary — one dataclass per intent holding its description, target LangGraph node, route kind, slot definitions, shortcircuit eligibility, example utterances, and whether it requires human confirmation before execution. Adding an intent is now a single dict entry. The prompt's intent definitions, the LLM's allowed enum values, the vector corpus, and the routing logic all derive from this one table at startup.

This pattern was adapted from two open-source projects:
- **RAGFlow** (`CategorizeParam.update_prompt()`): intent definitions as a dict, prompt built by code. We adapted the few-shot injection format (`USER: "原话" → 意图名`) but **don't inject all examples** — RAGFlow does because it has ~5 categories; we have 11 intents × ~15 examples = 170 utterances, which would dilute attention. We inject only the vector-retrieved top-k.
- **DB-GPT** (`IntentDetectionResponse` + `has_empty_slot()`): intent with declared slots; branch on slot completeness. We adapted the slot concept but enforce it via Pydantic `complete_typed` with auto-retry, not bare `json.loads`.

### Intent routing: Single Responsibility for LLM prompts

The router receives **only** the user's text and a lightweight component summary table (block IDs and titles — not the full MQL or dashboard JSON). It classifies into one of 11 intents and simultaneously extracts slot values.

The key insight: **intent + slot filling in one LLM call, not two.** V2.0 used a two-step flow: router classifies intent → a second LLM call extracts parameters (e.g., the month for `filter_change`). V3.0 merges them: the router's JSON schema includes both `intent` and `slots`, and the LLM fills both simultaneously. For intents like `filter_change` (which needs month/completion/conditions), this saves one LLM call per request. For intents with no required slots, it costs nothing extra.

```
Router input:   user_text + component_summary_table + context_ref (recent turns)
Router output:  { "intent": "filter_change",
                  "targets": [],
                  "slots": { "month": "2026-07", "completion": null, "conditions_text": null },
                  "thought": "用户说看7月→绝对年月",
                  "rewritten": "切换到2026年7月的数据" }
```

The router **never sees MQL or full dashboard state**. The component summary is a list of `{id, title}` pairs — just enough for the LLM to resolve references like "the pie chart on the right" to a specific block ID, without exposing the query logic that would tempt it to start modifying things.

### Dual-path classification: vector shortcircuit + LLM fallback

For efficiency, the router uses a dual-path architecture:

```
User message
     │
     ▼
 embed(message)  ← one embedding call
     │
     ▼
 cosine search against intent corpus (~155 labeled examples)
     │
     ├── score > 0.92 AND top-3 agree AND no required slots
     │   → shortcircuit: return intent directly (~200ms, 0 tokens)   [37%]
     │
     └── otherwise
         → inject top-k as few-shot examples in LLM prompt (~700ms)  [63%]
```

The retrieval is never wasted. One embedding call, two uses: classification for shortcircuit, or few-shot demonstrations for the LLM fallback.

**But shortcircuit has two iron rules** — both learned from production incidents:

1. **No required slots.** Shortcircuit skips the LLM, which means no slot values are extracted. If an intent has required slots (like `filter_change` needing a month), shortcircuit would produce an empty slot → the downstream guard forces a clarification → the user said something perfectly clear and gets asked to repeat it. We enforce this at startup: if an intent declares `shortcircuit=True` and has required slots, the process refuses to start.

2. **Only side-effect-free intents.** Shortcircuit means no LLM, no slot guard, no human verification — just cosine similarity directly triggering an action. For read-only intents (`query_data`, `analyze`, `chitchat`), a misclassification is recoverable: the user asks again. For write intents (`add_node`, `override`, `restore_preset`), a misclassification can destroy the dashboard layout with zero human checkpoint.

   Production incident: `add_node` was initially shortcircuit-eligible. "每月各险种成功率有多高?" (a question asking for a number) had high cosine similarity to `add_node` examples → the system added an empty chart card instead of answering the question. `restore_preset` is even worse — a false positive could wipe the entire dashboard.

   The principle: **read operations can trust the model; write operations must have code-verifiable evidence.**

### The `clarify` design: code decides, not the LLM

Most intent classification systems include an "Other" or "Unclear" category in the LLM's enum. We tried this and discovered a contagion effect: once the model outputs `clarify` for one ambiguous input, it starts outputting `clarify` for subsequent clear inputs too — it "learns" from its own conversation history that asking for clarification is an acceptable behavior.

Our solution: **`clarify` is not in the LLM's enum at all.** The LLM can only output one of the 11 real intents. The decision to ask for clarification is made entirely by deterministic code:

```
LLM outputs: { intent: "filter_change", slots: { month: null, completion: null } }

Code checks:  filter_change requires at least one of (month, completion, conditions_text)
              All are null → no evidence the user wants to filter
              → Code overrides to clarify, asks "你想改成哪个月?"

LLM outputs: { intent: "filter_change", slots: { month: "2026-07" } }

Code checks:  month is filled → evidence exists → proceed to apply_filter
```

This separation means the model is never rewarded for uncertainty. It must always commit to a classification. The guardrails are pure logic.

### Slot guards: evidence-based write permission

Every intent that modifies the dashboard has at least one required slot that serves as **evidence** — proof that the user actually requested this modification. The evidence varies by intent:

| Intent | Evidence (Required Slot) | What It Prevents |
|--------|-------------------------|-----------------|
| `update_node` | `targets` (which chart) + `change` (what to change) | Modifying a random chart |
| `remove_node` | `targets` (which chart to delete) | Deleting the wrong chart |
| `override` | `confirm_phrase` (the exact word from user's text that means "redo") | Accidentally wiping the entire dashboard |
| `filter_change` | At least one of `month`/`completion`/`conditions_text` | Overwriting filters with empty values |
| `reorder` | `targets` + `position` | Moving charts randomly |

For `override` specifically, the model must quote the user's exact word that indicates "redo" (重做, 推倒, 重新生成, etc.). If the model can't point to evidence in the user's text, the slot is empty, and code demotes it to clarification. This forces the model to provide proof, not just confidence.

Additionally, destructive intents like `override` and `restore_preset` trigger a LangGraph `interrupt()` — even after the model and slot guards agree, the system pauses and asks the user to click "confirm" before executing. Three layers of protection: model classification → slot evidence → human confirmation.

### The `asks_for_value()` guard: add_node's special problem

`add_node` is unique among write intents: it has no `targets` slot (you're adding something new, not modifying something existing) and its other slots (`chart_hint`, `metric_hint`) are optional. This means the model can say "add a chart" with zero verifiable evidence.

Production incident: "每月各险种成功率有多高?" should return a number (query_data). But the sentence was semantically similar to `add_node` examples about success rates. The model classified it as `add_node`, and since all slots are optional, every guard passed. The user got an empty chart card instead of an answer.

The fix is a deterministic function `asks_for_value()` that checks three conditions simultaneously:

```python
def asks_for_value(text):
    has_question_form = any(cue in text for cue in ["多少", "多高", "几个", "?", ...])
    has_placement_verb = any(cue in text for cue in ["加", "画", "做", "来个", ...])
    has_chart_demand   = any(cue in text for cue in ["比例", "趋势", "对比", ...])
    return has_question_form and not has_placement_verb and not has_chart_demand
```

All three conditions must hold: the text uses question form ("how many?") **AND** lacks any placement verb ("add", "draw") **AND** lacks chart-demand words ("trend", "comparison"). If it reads like a question and doesn't ask for a chart, it's `query_data`, not `add_node`.

Why all three? Because "我要看报增成功和报增失败的比例" has zero placement verbs but clearly wants a chart (比例 = proportion → needs a pie chart). Checking only question form would misclassify it.

### Compound intents: the Planner

Real users don't issue one instruction at a time. "Add a pie chart for insurance types and move the billing card to the top" is two operations. V3.0 includes a Compound Intent Planner that decomposes multi-instruction inputs into a task list:

```json
{
  "tasks": [
    { "task_type": "board_action", "intent": "add_node", "question": "按险种看合同分布" },
    { "task_type": "board_action", "intent": "reorder", "targets": ["billing_card"], "position": "top" }
  ]
}
```

Dependencies and conflicts are resolved by **deterministic code**, not the LLM:
- `add_node` + `reorder` on the same target → `reorder` depends on `add_node` (the card must exist first)
- `remove_node` + `update_node` on the same target → conflict (can't modify what you're deleting)
- `override` + any other board action → conflict (override wipes everything)

The LLM proposes *what* to do; code decides *when* and *whether* to do it. This separation keeps the execution semantics deterministic even when the intent decomposition is probabilistic.

### The six-gate validator

Between generation and execution, every dashboard passes through a six-gate validator:

| Gate | Check | What It Catches |
|------|-------|-----------------|
| G1 | Component whitelist | Unknown chart types |
| G2 | Required props | Missing mandatory fields per chart type |
| G3 | Data key source | Props that reference data without a pipeline source |
| G4 | MQL security | Forbidden operators (`$out`, `$merge`), collection whitelist |
| G5 | Chart role completeness | Pie without `name`+`value`, bar without `category`+`value` |
| G6 | Field mapping | `data_mapping` references fields the pipeline doesn't output |

Gate G6 — the field mapping check — addresses the single most user-visible failure in V1.0: charts that render with empty labels because the pipeline outputs `_id` and `count` but the chart expects `insurance_type` and `count`. The validator infers the pipeline's output fields from its last `$group` or `$project` stage and checks that every `data_mapping` reference exists in that output.

When validation fails, the error is fed back to the LLM for a retry (maximum 2 retries). The error message is specific: "data_mapping references `insurance_type_cn` but pipeline only outputs `_id, count`." This gives the LLM enough information to add a `$project` stage renaming `_id` to `insurance_type_cn`. The second attempt succeeds ~85% of the time.

### Impact

The multi-gate pipeline reduced dashboard hallucination from ~18% (after views) to ~10%. The remaining failures were predominantly semantic — correct structure, wrong business meaning.

---

## Shift 4: From Metric Lookup to Precedent-Based Generation

V2.0 used a **metric catalog** — a curated dictionary mapping business terms to their exact MQL definitions. When a user asked about "activation rate," the system looked up the metric ID, retrieved the pre-defined pipeline, and executed it.

This failed for a subtle reason: **wrong pick = wrong answer, silently.** If the AI selected `contract_count` instead of `active_contract_count`, the pipeline was syntactically perfect but semantically wrong. The chart rendered with confident-looking numbers. The numbers were wrong. There was no structural signal that anything had failed — the validator passed because the pipeline was well-formed.

### The example store

We replaced the metric catalog with a **precedent-based system**: a library of human-approved (question, query) pairs that the AI uses as references, not as lookup targets. The AI writes its own pipeline, guided by similar precedents rather than constrained to pre-defined answers.

```
Approval workflow:
                   ┌─────────┐    human     ┌───────────┐    human     ┌───────────┐
AI generates  ──▶  │  Draft  │  ────────▶   │  Pending  │  ────────▶  │ Published │
                   └─────────┘   review      └───────────┘  approval   └───────────┘
                        │                                                     │
                        │         ❌ NEVER auto-promotes                      │
                        └────────────── ✕ ────────────────────────────────────┘
                                                                     ▼
                                                              Enters AI context
```

The critical design rule: **queries that run successfully are never auto-promoted to published.** A query that returns data is not necessarily a query that returns correct data. `{$match: {status: "success"}}` returns results — but if the correct field value is `"declaration_success"`, the results are wrong and there's no structural way to detect this. Only a domain expert who understands the business semantics can approve an example.

This is the "anti-Vanna" rule. Vanna.ai's training loop auto-promotes successful queries. In enterprise settings where wrong numbers drive business decisions, that pattern is unacceptable.

### Injection strategy: conversation history, not system prompt

We inject approved examples as fake conversation history, not as system prompt documentation:

```python
messages = [
    {"role": "system", "content": system_prompt},
    # Approved examples as "previous conversations"
    {"role": "user", "content": "How many active contracts by region?"},
    {"role": "assistant", "content": '{"collection": "v_contracts_base", ...}'},
    {"role": "user", "content": "Show me payment progress for July"},
    {"role": "assistant", "content": '{"collection": "v_billing_financial_base", ...}'},
    # Actual user question
    {"role": "user", "content": actual_user_question}
]
```

In our testing, this format outperformed system prompt injection by ~20% on query accuracy. The model treats its own "previous answers" as stronger precedent than instructions — it imitates its own conversation patterns more reliably than it follows documentation.

### Impact

Semantic hallucination (correct structure, wrong business meaning) dropped from ~10% to ~5%. Each new approved example improves accuracy for similar future queries. The system gets smarter with use, not just with model upgrades.

---

## Shift 5: From Schema to Semantics

The final shift is the hardest to describe because it's not a single technique — it's a change in how the system thinks about data.

In V1.0, the AI's context was a **schema**: table names, field names, and types. This is how most text-to-SQL systems work. The AI knows that `bills` has a `bill_status` field of type `string`, and it generates queries accordingly.

In V3.0, the AI's context is a **semantic catalog**: not just what fields exist, but what they mean, how they relate to each other, and what questions they can answer.

### The semantic catalog

Each metric and dimension in the catalog has:

- A canonical name (the field name in the view)
- One or more aliases (how users refer to it in natural language)
- A kind (`metric`, `derived_metric`, `dimension`)
- A description (one sentence, for LLM grounding)
- Relationships to other entries (e.g., `payment_progress` = `paid_count / total_count`)

The catalog is the **vocabulary** the AI is allowed to use. When a user asks a question, the semantic retrieval layer (bigram coverage scoring for Chinese terms, cosine similarity for embeddings) identifies the most relevant catalog entries and injects them into the prompt.

### Grounding the generation

The generation pipeline receives four types of context:

1. **Catalog brief**: A compact summary of all available metrics and dimensions
2. **Semantic RAG hits**: The top-k most relevant catalog entries for this specific question
3. **Approved examples**: Similar precedent queries from the example store
4. **Component registry**: What chart types exist and what data roles they require

This is far more context than a schema dump. But it's also far more constrained — the AI can only reference metrics that exist in the catalog, chart types that exist in the registry, and query patterns that have been validated in the example store.

### Where to collapse: the boundary question

Not everything belongs in views. We learned this the hard way by initially putting customer-specific settlement rules into views, which created views that only worked for specific tenants.

The boundary rule:

| Layer | What Goes Here | Example |
|-------|---------------|---------|
| **Views** | Site-wide, verified business rules | `is_paid = (pay_status == "paid")` |
| **Catalog** | Metric definitions and aliases | `payment_progress = paid_count / total_count` |
| **Examples** | Validated query patterns | "Q: payment progress for July → A: {pipeline}" |
| **Prompts** | Dynamic context, user-specific filters | Current dashboard state, role-based data scoping |
| **NOT in views** | Customer-specific rules, experimental metrics | Per-tenant settlement calculations |

The guiding principle: views contain **universal truths** about the data model. Anything that varies by customer, tenant, or use case stays in the prompt or the application layer.

### The evaluation loop

Without measurement, all of these shifts are educated guesses. We built a three-layer evaluation system:

**Real-time validation:** The six-gate validator runs on every dashboard. Structurally invalid dashboards never reach users.

**Nightly calibration:** ~50 canonical queries run against production data every night. Each query has structural assertions (expected chart type, expected field names, expected data series count range) that don't depend on specific data values. Regressions trigger alerts.

**Human feedback loop:** Users flag incorrect dashboards. Flagged cases enter the example store as drafts. A domain expert reviews them weekly, corrects the query, and publishes the corrected example. The corrected example immediately improves similar future queries.

```
user flags "wrong" → draft created
     → expert reviews → corrects query → publishes
          → enters AI context → AI improves → fewer flags
                                                  ↺
```

### The mathematical foundation

The entire system rests on a few mathematical primitives:

- **Set theory.** The base population is a universe. `$match` selects a subset. Filters compute intersections. The funnel `contracts → declarations → bills` is a directed acyclic graph where each node is a subset of the previous one, and the direction determines which denominator to use for ratios.

- **State collapse.** Status codes are high-dimensional (dozens of possible values). Boolean labels are one-dimensional (true/false). The view performs a many-to-one mapping `f: StatusCode → Boolean` that reduces dimensionality while preserving the information needed for 90% of questions. The original field survives for the 10% that need full detail.

- **Normalization.** Division (rate = count_A / count_B) is projection onto the [0, 1] interval. It removes scale differences, which is why "what percentage of bills are confirmed?" is a fundamentally different question from "how many bills are confirmed?" — and why confusing the two is one of the most common hallucination modes.

---

## The architecture, bottom to top

![Figure 3. Full Dashboard Architecture](/assets/images/fig8-dashboard-architecture.svg)
*Figure 3. Five layers: physical storage → semantic views → domain knowledge → orchestration → execution → presentation + feedback.*

```
Layer 5: Presentation + Feedback
  ┌─────────────────────────────────────────────────────────┐
  │  SDUI Renderer          │  Feedback Loop                │
  │  (backend: lg grid      │  (user flag → draft →         │
  │   frontend: md/sm)      │   expert → published example) │
  └─────────────────────────┴───────────────────────────────┘
                              ▲
Layer 4: Execution Engine     │
  ┌───────────────────────────┴─────────────────────────────┐
  │  Parallel MQL execution (one query per component)       │
  │  data_mapping transform (BSON → component shape)        │
  │  Preset merge (complex KPIs from templates)             │
  └───────────────────────────┬─────────────────────────────┘
                              ▲
Layer 3: Orchestration        │
  ┌───────────────────────────┴─────────────────────────────┐
  │  Intent Router → Query Generator → Board Composer       │
  │  (vector shortcircuit 37%)  (complete_typed)  (SDUI)    │
  │  Board Validator (6-gate check, retry on failure)       │
  └───────────────────────────┬─────────────────────────────┘
                              ▲
Layer 2: Domain Knowledge     │
  ┌───────────────────────────┴─────────────────────────────┐
  │  Semantic Catalog  │  Example Store  │  Component Reg.  │
  │  (metrics, dims,   │  (approved Q/A  │  (chart whitelist │
  │   aliases, bigram  │   pairs, draft  │   required props, │
  │   scoring)         │   workflow)     │   data_mapping)   │
  └───────────────────────────┬─────────────────────────────┘
                              ▲
Layer 1: Semantic Views       │
  ┌───────────────────────────┴─────────────────────────────┐
  │  v_contracts_base  │  v_declaration_base │  v_billing_*  │
  │  ($set booleans,   │  ($unionWith merge, │  ($set booleans│
  │   preserve all     │   unified_month,    │   pre-computed │
  │   original fields) │   direction flag)   │   amounts)     │
  └───────────────────────────┬─────────────────────────────┘
                              ▲
Layer 0: Physical Storage     │
  ┌───────────────────────────┴─────────────────────────────┐
  │  contracts  │  si_increase  │  si_decrease  │  bills    │
  │  (MongoDB collections, on-premise, PIPL compliance)     │
  └─────────────────────────────────────────────────────────┘
```

Each layer has a clear responsibility and a clear contract with the layers above and below it. Views encode business logic. The catalog defines vocabulary. The orchestration pipeline generates, validates, and executes. The feedback loop improves the system over time.

---

## What we shipped, measured

| Metric | V1.0 | V2.0 | V3.0 |
|--------|------|------|------|
| Hallucination rate (data queries) | ~35% | ~18% | <7% |
| Median query latency | ~2.0s | ~1.5s | ~1.2s (full) / ~200ms (shortcircuit) |
| Middleware lines (semantic correction) | 866 | ~800 | ~800 (execution only, 0 semantic correction) |
| Number of validated test cases | 0 | ~20 | 50 nightly + 759 unit tests |
| Example store coverage | N/A | N/A | ~60% of common query patterns |
| Intent shortcircuit rate | N/A | N/A | 37% |
| Validator rejection rate | N/A | N/A | 12–18% (rejected → retried → 85% succeed on retry) |

The numbers tell a clear story: the hallucination rate dropped 5x, but no single technique was responsible. Views contributed ~17pp, constrained vocabulary ~8pp, multi-gate validation ~8pp, precedent-based generation ~5pp, and the evaluation loop provides ongoing compound improvement.

---

## What we'd do differently

### 1. Build the semantic views before writing any AI code

We spent two months fighting hallucination in the middleware before realizing the fix was in the database. If we'd started with views, the middleware would never have existed.

### 2. Reject the metric catalog earlier

The metric catalog felt correct — it's how dbt and Cube work. But those tools control the entire query generation pipeline. In an LLM-based system where the AI writes its own queries, a catalog is a lookup table that can be looked up wrong. We should have moved to the example store pattern in V2.0, not V3.0.

### 3. Instrument the pipeline before optimizing it

We added the nightly calibration suite in month six. Months three through five were spent fixing individual user complaints without any systematic measurement. If we'd had 50 canonical test queries from the start, every change would have been immediately measurable.

### 4. Don't compute in views what you can label in views

Our first views tried to compute derived metrics (ratios, progress percentages). This made the views complex and fragile. The better pattern: views compute boolean labels and pre-computed amounts; the LLM handles the aggregation and arithmetic, guided by examples. Views should encode **facts**, not **analysis**.

### 5. SDUI was the right call; commit to it earlier

Server-Driven UI — where the backend sends a component tree and the frontend renders it — initially felt like overengineering. It turned out to be essential: it let us iterate on dashboard layout without frontend deployments, and it gave us a clean contract between the AI (which decides what charts to show) and the frontend (which decides how to render them responsively).

---

## Further reading

The projects and papers that most influenced our architecture:

- **[Vanna.ai](https://vanna.ai/)** — Training-data closed loop for text-to-SQL. We adapted their `add_question_sql` / `get_similar_question_sql` pattern but rejected their auto-promotion of successful queries.
- **[DB-GPT](https://github.com/eosphoros-ai/DB-GPT)** — AWEL workflow architecture and metadata management. Their approach to field whitelisting informed our component registry design.
- **[RAGFlow](https://github.com/infiniflow/ragflow)** — Categorize component with example injection. Our dual-use intent routing was directly inspired by their `CategorizeParam.update_prompt()`.
- **[Cube.js](https://cube.dev/)** — Universal Semantic Layer concept. We use the same principle (AI requests metrics by name, not by query) but implement it through MongoDB views instead of external middleware.
- **[dbt + MetricFlow](https://docs.getdbt.com/docs/build/metrics-overview)** — Metric definitions as code. Our semantic catalog structure mirrors MetricFlow's metric/dimension/entity taxonomy.
- **BIRD-SQL (NeurIPS 2023)** — Demonstrated that Schema Linking + External Knowledge evidence dramatically improves text-to-SQL accuracy.
- **EntSQL (2026)** — Enterprise long-context benchmark showing only 15.9% accuracy without a semantic layer. Validated our decision to invest in views.
- **ReViSQL (2026)** — Achieved 92.96% human-level accuracy by eliminating internal inconsistency in generated SQL — the same principle behind our six-gate validator.

---

*The system described serves ~300 enterprise users generating MongoDB aggregation pipelines from natural language. All data remains on-premise in compliance with data sovereignty requirements. Architecture patterns are described generically; no company names or internal identifiers are shared.*
