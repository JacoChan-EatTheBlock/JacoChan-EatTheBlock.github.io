---
layout: default
title: "Enterprise Prompt Engineering: Why Your Prompts Should Look Like Compiled Code, Not Conversations"
date: 2026-09-06
categories: [Prompt Engineering, LLM, Enterprise AI]
---

# Enterprise Prompt Engineering: Why Your Prompts Should Look Like Compiled Code, Not Conversations

*September 6, 2026 · 20 min read*

**Key Takeaways**

- In enterprise AI systems, prompts are not conversations — they are dynamically compiled business logic. Their single job is mapping fuzzy human intent to strict system enums and variables.
- The "Sandwich Architecture" separates concerns: Schema (Pydantic) locks what the AI *can* say, the Prompt maps *how* it thinks, and the Execution Engine controls *what actually happens*. When layers 1 and 3 are strong, layer 2 becomes thin and maintainable.
- Four structural pillars — Dependency Injection, Role & Scope, Semantic Dictionary, and Few-Shot as Unit Tests — replace the common anti-patterns of defensive prompting, hardcoded business logic, and format instructions inside prompts.

---

Most prompt engineering advice is written for chatbot developers. It focuses on phrasing, tone, and the craft of getting an LLM to produce a desired response in a single turn. This works when you're building a customer support bot or a writing assistant.

It does not work when you're building an enterprise AI platform where the LLM's output is consumed by deterministic code, not humans. Where a wrong enum value silently produces a chart with incorrect financial data. Where prompts are executed thousands of times a day across 11 different intent classifications, and a single hardcoded assumption can cascade into a system-wide regression.

We built an enterprise AI workbench serving ~300 users. Over eight months, our prompt architecture evolved from a 90-line spaghetti string to a clean, modular system where adding a new intent requires changing exactly one dictionary entry. This post describes the architecture that emerged, the anti-patterns we killed, and the principles that survived.

![Figure 1. The Sandwich Architecture](/assets/images/fig12-prompt-sandwich.svg)
*Figure 1. Three layers, each with a single responsibility. When the Schema and Execution Engine are strong, the Prompt layer becomes thin.*

---

## The mental model shift: prompts as semantic mappers

The single most important insight we gained is this: **in enterprise AI, a prompt is a semantic mapper.** Its only job is translating fuzzy human language into strict system variables.

Consider what happens when a user says "show me last month's confirmed bills by region." The system needs to determine:

1. **Intent**: Is this a new chart request (`add_node`), a filter change (`filter_change`), or a data query (`query_data`)?
2. **Time reference**: "Last month" → absolute date `2026-08`
3. **Metric**: "Confirmed bills" → `is_confirmed: true` on the `v_billing_financial_base` view
4. **Dimension**: "By region" → `group_by: region`
5. **Chart type**: Unspecified → the system picks bar chart as default

The prompt's job is step 1 and 2 — mapping the fuzzy input to structured slots. Steps 3–5 are handled by deterministic code: the semantic catalog, the query compiler, the component registry. The prompt never touches MQL, never decides which database field to query, never computes a date range.

This is a radical departure from the common pattern where the prompt carries the entire burden — from understanding intent to generating the query to formatting the output. That pattern is why most LLM-based data systems hallucinate 30%+ of the time.

---

## The Sandwich Architecture

We call our prompt organization the "Sandwich Architecture" because the prompt sits between two deterministic layers that constrain its behavior from above and below.

```
┌─────────────────────────────────────────────────────────┐
│  Layer 1: SCHEMA SKELETON (Pydantic / JSON Schema)      │
│                                                         │
│  Physically locks what the AI can output.               │
│  Enum values injected from intent_catalog at startup.   │
│  Validation failure → auto-retry with error feedback.   │
│  The AI CANNOT produce invalid JSON, unknown intents,   │
│  or wrong field types.                                  │
├─────────────────────────────────────────────────────────┤
│  Layer 2: PROMPT (Semantic Mapper)                      │
│                                                         │
│  Maps fuzzy human language to schema slots.             │
│  Contains: DI context, role declaration, semantic       │
│  definitions, and few-shot boundary examples.           │
│  Does NOT contain: format instructions, business        │
│  logic, query syntax, or defensive language.            │
├─────────────────────────────────────────────────────────┤
│  Layer 3: EXECUTION ENGINE (Deterministic Python)       │
│                                                         │
│  Takes the AI's structured output and executes it.      │
│  Looks up metric definitions, applies base filters,     │
│  compiles IR → MQL, validates output fields,            │
│  enforces permissions. 100% hallucination-free.         │
└─────────────────────────────────────────────────────────┘
```

The key insight: **when layers 1 and 3 are strong, layer 2 becomes thin and maintainable.** If the schema prevents invalid outputs and the execution engine handles all business logic, the prompt only needs to do semantic mapping — and that's a much smaller, more testable task.

### Why this works: the burden distribution

| Responsibility | Traditional approach | Sandwich Architecture |
|---|---|---|
| Output format | Prompt: "Only output JSON with fields..." | Schema: Pydantic model, constrained decoding |
| Valid enum values | Prompt: hardcoded list in markdown table | Schema: enum injected from intent_catalog |
| Business logic | Prompt: "status=active AND biz_type IN..." | Engine: semantic views + compiler |
| Date computation | Prompt: "Current month is September..." | Engine: time anchor injected as {variable} |
| Error handling | Prompt: "If unsure, ask for clarification" | Engine: slot guard checks + code-driven clarify |
| Component preservation | Prompt: "Keep untouched components unchanged" | Engine: Python merges AI output with existing tree |

Every row where the burden moves from "Prompt" to "Schema" or "Engine" is a row where hallucination risk drops to zero for that specific failure mode.

---

## The Four Pillars

Inside Layer 2 (the Prompt), we organize content into exactly four sections. Anything that doesn't fit into one of these pillars doesn't belong in the prompt.

### Pillar 1: Dependency Injection

**Principle: The LLM has no memory. Never hardcode any business state in the prompt.**

Every piece of dynamic context is injected at runtime via template variables, exactly like dependency injection in a software framework:

```python
sys_prompt = template.format(
    time_anchor=render_month_anchor(now),         # "Today is 2026-09-04..."
    intent_definitions=intent_catalog.render(),    # Generated from INTENTS dict
    examples=intent_router.similar_examples(query) # Vector-retrieved top-k
)
```

What goes into DI slots:

| Variable | Source | Why not hardcode |
|---|---|---|
| `{time_anchor}` | `render_month_anchor(now)` | "Last month" changes every month |
| `{intent_definitions}` | `intent_catalog.render_intent_definitions()` | Adding an intent = adding a dict entry, not editing the prompt |
| `{examples}` | `intent_router.similar_examples(query)` | Different user queries need different examples |
| Component summary | `layout_assembler.build_component_index()` | Dashboard state changes every turn |

The anti-pattern this replaces: prompts that contain lines like "Current month is September 2026" or "Available intents: add_node, update_node, remove_node..." that must be manually updated whenever the system changes.

**Design reference:** RAGFlow's `CategorizeParam.update_prompt()` takes this approach — categories are stored as a dict, and the prompt is assembled by code. Intent names, descriptions, and examples never appear as literal strings in the prompt template. Our adaptation: RAGFlow injects all examples (it has ~5 categories); we inject only vector-retrieved top-k (we have 11 intents × ~15 examples = 170 total, which would dilute attention if injected wholesale).

### Pillar 2: Role & Scope

**Principle: One sentence of clear role declaration beats ten paragraphs of defensive instructions.**

The role declaration should be minimal, declarative, and specific:

```
You are a data dashboard intent classifier. The user's screen already has a dashboard.
Your task: select ONE intent from the definitions below and fill its declared slots.
You do not generate MQL or layouts — that is handled by downstream steps.
```

Three sentences. No hedging, no threats, no emotional appeals.

**The anti-pattern this replaces:** Defensive prompting.

We used to write things like:

> "STRICTLY FORBIDDEN to modify business data! VIOLATION of this rule IS DECEPTION of the user!"
> "ABSOLUTELY DO NOT output anything other than the specified JSON format!"
> "Do NOT be influenced by..."

This is the prompting equivalent of writing `// DO NOT DELETE THIS LINE` in code comments. It feels meaningful to the author and is ignored by the consumer. LLMs don't have emotions. They can't be threatened. They respond to patterns, not warnings.

**The evidence:** After removing all defensive language from our prompts, classification accuracy stayed the same. What actually moved the accuracy needle was adding better few-shot examples (Pillar 4).

### Pillar 3: Semantic Dictionary

**Principle: Explain what each enum means in business terms. Never explain the underlying query logic.**

The semantic dictionary tells the LLM what each intent/metric means in the user's language, without revealing how it's implemented:

```
Intent definitions (generated by intent_catalog.render_intent_definitions()):

  add_node: Add a new chart/card to the dashboard (user wants to see something new)
  update_node: Modify an EXISTING card (change chart type, title, size, remove an item within it)
  remove_node: Delete an ENTIRE card (not an item within a card — that's update_node)
  filter_change: Change global filters (time, completion status, field conditions) — layout unchanged
  query_data: Ask for a number, not a chart (if user wants a chart → add_node)
  ...
```

Notice what's absent: there's no mention of MongoDB collections, aggregation pipeline stages, field names, or status codes. The LLM doesn't need to know that "confirmed bills" maps to `bill_status == "confirmed"` — that mapping lives in the semantic view layer and the query compiler.

**Why this matters:** When we put query logic in the prompt (V1.0), the LLM would mix up status codes across tables, forget base population filters, and pick the wrong month field. Moving all query logic to the execution engine and keeping only business-level descriptions in the prompt reduced semantic hallucination by ~20%.

**Design reference:** DB-GPT's `_DEFAULT_PROMPT_ZH` takes a similar approach — the prompt describes intents in business terms; the slot definitions specify what to extract, not how to use it downstream.

### Pillar 4: Few-Shot as Unit Tests

**Principle: 5 boundary cases > 500 words of rules. Show, don't tell.**

LLMs are pattern matchers. Instead of explaining a rule in prose, show the rule as an input-output pair:

```
Before (500 words of rules):
  "When a user says 'remove' something, you must carefully determine whether
   they mean removing a data filter condition, removing an entire chart card,
   or removing a specific item within a card. If it's a data condition like
   a company name or bill number, classify as filter_change. If it's an entire
   chart referenced by title or position, classify as remove_node. If it's..."

After (3 boundary examples):
  USER: "Don't show Shanghai Company's data" → filter_change (data condition)
  USER: "Delete the billing pie chart" → remove_node (entire card)
  USER: "Remove the 'total contracts' item from the contract card" → update_node (item within card)
```

The three examples communicate the same rule in fewer tokens, with less ambiguity, and with higher classification accuracy. In our testing, replacing prose rules with equivalent few-shot examples improved intent classification accuracy by ~8%.

**Critical design decision: examples are vector-retrieved, not hardcoded.**

Our intent corpus has ~170 labeled utterances across 11 intents. Injecting all 170 into every prompt would waste tokens and dilute the model's attention. Instead, we embed the user's current query, retrieve the 5 most similar labeled utterances via cosine similarity, and inject only those as few-shot examples.

This means the examples are always relevant to the current query. A question about "billing status" gets billing-related examples, not contract-related ones. One embedding call, two uses: classification shortcircuit (37% of queries) or few-shot injection (63% of queries).

---

## Three Surgical Cuts

The evolution from our V1.0 prompt to the current architecture required three specific interventions. Each one removed a category of prompt content that was either redundant (handled by schema), misplaced (belonged in code), or counterproductive (defensive language).

### Cut 1: Remove all format instructions

**Before:**
```
Only output a single JSON object with the following fields:
route, intent, targets, confidence, clarify_question.
Do not include any explanation, markdown, or additional text.
The intent field must be one of: add_node, update_node, remove_node, ...
```

**After:** (nothing — this entire block is deleted)

**Why:** We use `complete_typed(schema_model=BoardIntent)` — constrained decoding that forces the LLM to produce valid JSON matching a Pydantic schema. The schema declares the field names, types, and valid enum values. The LLM physically cannot produce non-JSON output, unknown intent names, or missing required fields.

The format instructions in the prompt were not just redundant — they were harmful. They consumed tokens (attention budget) that could have been spent on understanding the user's intent. And they occasionally conflicted with the schema constraints, causing the LLM to hesitate between the prompt's format description and the schema's structural enforcement.

**Token savings:** ~120 tokens per prompt × thousands of classifications per day. Not trivial.

### Cut 2: Kill the "God Node"

In V1.0, we had a single `LAYOUT_GENERATE_SYSTEM` prompt that handled both creating new dashboards and modifying existing ones. It contained this instruction:

> "When the current dashboard is provided, preserve untouched components — keep their id, component type, props, MQL, metrics, and data_mapping unchanged. Only modify what the user requested."

This is asking the LLM to act as a merge function — take two data structures (existing layout + user request) and produce a merged result. LLMs are terrible at this. They lose fields, mutate IDs, and hallucinate new components in the process. In our testing, the "preserve existing" instruction failed ~25% of the time, silently corrupting untouched components.

**The fix:** We split generation into two entirely separate prompts with non-overlapping responsibilities:

| Prompt | Trigger | Scope | "Preserve existing?" |
|---|---|---|---|
| `NODE_GENERATE` | `add_node`, `update_node` | Single card only | No — Python handles merge |
| `LAYOUT_GENERATE` | `override` only | Entire dashboard from scratch | No — there's nothing to preserve |

For `add_node`, the LLM generates a single new card. Python appends it to the existing tree. For `update_node`, the LLM generates a single modified card. Python replaces the old one. The LLM never sees the full dashboard tree, so it can't corrupt it.

For `override` (the user explicitly says "redo everything"), the LLM generates an entire new dashboard from scratch. Since the user asked to wipe the slate, there's nothing to preserve.

**Result:** Layout corruption dropped from ~25% to under 2%.

### Cut 3: Replace threats with tests

**Before:**
```
★ This turn you have NOT modified the dashboard (hard constraint — violating this IS DECEIVING the user)
STRICTLY FORBIDDEN to say "I've updated the dashboard..." — the user will think the dashboard changed...
```

**After:**
```
[Boundary examples]
USER: "I want to see the ratio of successful vs failed declarations"
  ❌ Wrong: {"intent": "query_data"} — query_data returns only a number, not a chart
  ✅ Right: {"intent": "add_node"} — the user wants a visual comparison

USER: "Change this to completed status"
  ❌ Wrong: {"intent": "update_node"} — this is a filter, not a chart modification  
  ✅ Right: {"intent": "filter_change", slots: {completion: "done"}}
```

The boundary examples communicate the same constraints as the threatening language, but through demonstrated patterns rather than abstract rules. The LLM infers the constraint from seeing it applied to concrete cases.

**Evidence from production:** Our "three absolute prohibitions" (permission violation, chart demand misroute, component vs. data confusion) were originally expressed as 15 lines of threatening prose. Converting them to 6 boundary examples (2 per prohibition) maintained the same classification accuracy while removing 15 lines of prompt text.

![Figure 2. Prompt Architecture Evolution](/assets/images/fig13-prompt-evolution.svg)
*Figure 2. Left: the V1.0 "spaghetti prompt" — 90 lines, 8 places to edit per intent change. Right: the V3.0 "4-pillar prompt" — a skeleton template with dynamic injection, adding an intent requires one dict entry.*

---

## The three prohibitions that survived

Not everything was cut. Three rules survived every refactoring because they address failure modes that examples alone couldn't fix. They earned their place through production incidents:

### 1. Permission violation boundary

Users occasionally say things like "change this bill's status to paid" or "delete this record." These are requests to modify underlying business data, not dashboard visualization. The distinction matters enormously: a dashboard modification is reversible; a data modification is not.

This prohibition survived because the failure mode is catastrophic (modifying production data) and the linguistic signal is subtle (both "change the chart" and "change the data" use the word "change"). We keep it as a short, clear rule — not a threat, just a boundary:

> "Requests to modify/delete underlying business data, or capabilities outside the dashboard (file export, etc.) → always classify as chitchat, respond with a polite refusal."

### 2. Chart demand routing

"What's the activation success rate?" could be either `query_data` (return a number: 87.3%) or `add_node` (create a pie chart showing the ratio). The correct classification depends on whether the user's question contains chart-demand words (比例/趋势/对比/占比/分布).

This rule survived because misclassification has asymmetric consequences. Routing `add_node` to `query_data` means the user gets a number when they wanted a chart — disappointing but recoverable. Routing `query_data` to `add_node` means the system creates an unwanted chart card that persists on the dashboard — annoying and requires manual cleanup.

### 3. Component vs. data disambiguation

"Remove Shanghai Company" could mean three different things:

- Filter out Shanghai Company's data → `filter_change`
- Delete the chart card titled "Shanghai Company" → `remove_node`
- Remove Shanghai Company as an item within a multi-company chart → `update_node`

This disambiguation requires understanding whether the referenced entity is a data condition, a chart component, or an item within a component. We keep it as a structured decision tree in the prompt because few-shot examples alone couldn't cover the combinatorial space.

---

## The maintenance test

The ultimate measure of prompt architecture quality is maintenance cost. Here's ours:

| Operation | V1.0 (spaghetti) | V3.0 (4-pillar) |
|---|---|---|
| Add a new intent | Edit 8 files | Add 1 dict entry |
| Update intent description | Find and edit in 90-line prompt | Edit `description` field in dict |
| Add a few-shot example | Edit prompt string + redeploy | Add to `examples` tuple in dict, re-seed vector corpus |
| Change time handling | Edit hardcoded "Current month..." | Modify `render_month_anchor()` function |
| Change output format | Edit prompt + hope it matches schema | Edit Pydantic model (schema auto-updates) |
| A/B test a prompt variant | Copy 90-line string, modify, deploy | Swap `prompt_registry_service.load(key)` version |

The 4-pillar system turns prompt maintenance from a fragile text-editing task into a structured data operation. The prompt template itself rarely changes — it's the data flowing through the DI slots that evolves.

---

## Design principles summary

1. **Prompts are semantic mappers, not business logic containers.** Every line of business logic in a prompt is a line that can hallucinate. Move it to deterministic code.

2. **Schema above, engine below.** Constrained decoding (Layer 1) prevents structural hallucination. Deterministic execution (Layer 3) prevents semantic hallucination. The prompt (Layer 2) only needs to handle the mapping between natural language and structured slots.

3. **Inject, don't embed.** Every piece of dynamic state — time, available intents, relevant examples, component lists — enters the prompt through template variables, not hardcoded text.

4. **Show, don't tell.** Few-shot boundary examples communicate rules more effectively and in fewer tokens than prose explanations. Vector-retrieve the most relevant examples per query.

5. **Defensive prompting is an anti-pattern.** LLMs don't respond to threats. They respond to patterns. Replace "STRICTLY FORBIDDEN" with a concrete example showing the wrong output and the right output.

6. **The maintenance test matters.** If adding a new intent requires editing more than one file, your prompt architecture has a coupling problem.

---

## What we'd do differently

1. **Start with the 4-pillar structure from day one.** We spent two months with spaghetti prompts before the refactoring. The refactoring itself took three days. The maintenance cost of those two months was far higher than the three-day investment.

2. **Build the prompt registry earlier.** Our `prompt_registry_service` allows versioning, A/B testing, and runtime editing of prompts without redeployment. We built it in month four. Every prompt change in months one through three required a code deploy.

3. **Instrument prompt performance from the start.** We now track per-prompt classification accuracy, token consumption, and latency. This data drove every optimization described in this post. Without it, we were optimizing by intuition.

---

## References

- [RAGFlow](https://github.com/infiniflow/ragflow) — `CategorizeParam.update_prompt()`: categories as dict, prompt assembled by code
- [DB-GPT](https://github.com/eosphoros-ai/DB-GPT) — `IntentDetectionResponse` + slot discipline patterns
- [Anthropic Prompt Engineering Guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview) — Structured prompting best practices
- [OceanBase: From Distributed OLTP to AI Lakebase](https://en.oceanbase.com/blog/from-distributed-oltp-to-ai-lakebase) — Architectural shift narrative structure

---

*The system described serves ~300 enterprise users processing natural language queries against MongoDB. All prompt examples are described generically; no proprietary prompt text is shared.*
