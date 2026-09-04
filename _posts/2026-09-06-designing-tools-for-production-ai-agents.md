---
layout: default
title: "Designing Tools for Production AI Agents: From Free-Form Generation to Constrained Intelligence"
date: 2026-09-06
categories: [AI Architecture, Agent Tools, LangGraph, MCP]
---

# Designing Tools for Production AI Agents: From Free-Form Generation to Constrained Intelligence

*September 6, 2026 · 20 min read*

Anthropic's recent post on [writing effective tools for AI agents](https://www.anthropic.com/engineering/writing-tools-for-agents) makes a point that should be obvious but isn't: agents are only as effective as the tools we give them. A tool with a confusing description, overlapping functionality, or bloated response will degrade agent performance more reliably than a bad model.

We learned this the hard way. Our enterprise AI workbench serves ~300 users across four business domains — conversation AI, data dashboards, a skill marketplace, and external tool integration via MCP. The agents in this system use tools ranging from simple data lookups to executing user-authored Python code in Kubernetes sandboxes. Over eight months in production, we developed a set of principles for tool design that echo Anthropic's recommendations — and diverge in ways that matter for enterprise deployments.

This post describes what we built, what broke, and the five principles we extracted. If you're designing tools for agents that operate on real data with real consequences, the constraint spectrum matters more than the tool count.

---

## The Constraint Spectrum

The single most important concept in production tool design is the **constraint spectrum** — the degree to which a tool limits what the agent can do.

![Figure 1. The Spectrum of Tool Constraint](/assets/images/fig19-tool-design-principles.svg)
*Figure 1. From left to right: increasing constraint, decreasing hallucination. Our system operates at Levels 2-4 depending on the intent. Level 1 (raw code generation) was abandoned after two months.*

### Level 1: Raw Code Generation (abandoned)

In V1.0, our data dashboard tool was essentially: "Given this schema, write a MongoDB aggregation pipeline." The LLM had full freedom to write any valid MQL. This is Level 1 — the tool's input is an arbitrary code string.

The hallucination rate was ~35%. Not because the model was bad, but because the degrees of freedom were too high. Four tables, dozens of status codes, different month fields per table, business-specific base population filters — the search space was enormous.

### Level 2: Parameterized Templates

At Level 2, the LLM fills slots in a constrained template rather than writing arbitrary code. Our `filter_change` intent works this way: the LLM extracts a month value ("2026-07"), a completion status ("done"/"undone"/null), and an optional natural-language condition. Deterministic code applies these parameters to the existing query structure.

The key insight: the LLM is good at understanding what the user wants (semantic extraction). It's unreliable at translating that understanding into syntactically perfect database queries. Level 2 separates these concerns.

### Level 3: Semantic Intermediate Representation

At Level 3, the LLM writes a high-level plan describing *what* to retrieve, and a deterministic compiler generates the *how*. This is our `query_data` flow:

```
User: "How many active contracts are there?"
    ↓
LLM generates SemanticQueryPlan:
{
  "strategy": "catalog",
  "view": "v_contracts_base",
  "metric": "contract_count",
  "filters": [{"field": "is_active", "value": true}],
  "chart_type": null  // query_data returns numbers, not charts
}
    ↓
compiler.compile_plan() → MongoDB aggregation pipeline
    ↓
Execute → Return number
```

The compiler is ~800 lines of Python. It handles collection routing, base population injection, field validation, and pipeline construction. The LLM never writes a `$group` or `$match` stage. It declares intent; code implements it.

This is the pattern Anthropic describes as "constrained tool parameters" — the tool's input schema limits what the LLM can request. But we take it further: the schema isn't just a validation layer; it's a **compilation source**. The LLM's output is an intermediate representation (IR), not a final artifact.

### Level 4: Menu Selection

At Level 4, the LLM simply picks from a predefined catalog. Our `remove_node` and `reorder` intents work this way: the LLM identifies which chart card to act on (by matching the user's description to the component summary table) and specifies a position. Deterministic code performs the mutation. Zero LLM generation of any content — the "tool call" is pure selection.

The hallucination rate at Level 4 is under 2%, limited almost entirely to misidentification of which card the user is referring to.

### Why the spectrum matters

Different intents in the same system operate at different constraint levels. The right level depends on the **cost of error**:

| Intent | Level | Cost of Error | Why This Level |
|--------|-------|---------------|----------------|
| `query_data` | 3 (IR + compiler) | Medium — wrong number | User can ask again |
| `add_node` | 3 (IR + compiler + registry) | High — wrong chart persists | Visible to all users |
| `filter_change` | 2 (parameterized) | Medium — wrong filter | Easily reversed |
| `remove_node` | 4 (menu selection) | High — deleted card is gone | Irreversible without undo |
| `override` | 3 + interrupt | Critical — entire dashboard wiped | Requires human confirmation |

The principle: **match constraint level to error cost.** Low-stakes read operations can afford Level 2-3. High-stakes write operations should be Level 4 whenever possible, with human confirmation for destructive actions.

---

## Tool Architecture: The SkillHub

Beyond the dashboard domain, our platform includes a general-purpose tool execution system: the SkillHub. Users publish custom "skills" — Python code packages — that any agent can invoke. This is the most dangerous tool in our system, because the input is arbitrary code.

![Figure 2. SkillHub Security Layers](/assets/images/fig18-tool-architecture.svg)
*Figure 2. Three security layers between user-authored code and production infrastructure. Each layer addresses a different threat vector.*

### Layer 1: YARA Security Scanning

Before any skill enters the marketplace, a dedicated scanning microservice runs YARA rules against the code package. YARA is a pattern-matching engine originally designed for malware classification — it checks for known malicious patterns like reverse shells, data exfiltration attempts, and cryptominer signatures.

This is a publish-time gate, not a runtime gate. The cost of a false positive is a rejected skill (the user can fix and resubmit). The cost of a false negative is malicious code running inside our infrastructure.

### Layer 2: K8s Ephemeral Pods

Each skill invocation gets a fresh Kubernetes pod. No state persists between runs. The skill writes its outputs to `/workspace/outputs/`, which is the only directory mounted for writing. When the pod terminates, everything else is gone.

For long-running tasks, skills declare `mode: async` in their manifest. The system submits a K8s Job instead of a synchronous pod, returns a `run_id` immediately, and an `async_poller` background task monitors completion. This pattern — submit, poll, retrieve — handles document batch processing and data migration tasks that exceed the 300-second synchronous timeout.

### Layer 3: NetworkPolicy Isolation

The critical design decision is the **trust boundary**. We distinguish between:

- **External skills** (user-authored): NetworkPolicy denies all egress. The code cannot make any outbound network connections. It can read input files and write output files, nothing else.
- **Internal skills** (platform-authored): NetworkPolicy allows outbound connections. These are trusted code packages maintained by the platform team — document converters, report generators, API integrators.

The trust boundary is enforced by the `origin` field in the skill manifest (`internal` vs `external`), not by the code itself. This means a user cannot escalate privileges by writing clever code — the network policy is applied at the Kubernetes level based on the manifest, before any code runs.

### Cold-Start Mitigation

The first invocation of a skill image takes 10-15 seconds for pod scheduling and image pull. For frequently used skills, this is unacceptable. We run a **prepuller DaemonSet** that keeps popular images warm on every node. Subsequent invocations start in under 2 seconds.

The prepuller maintains a whitelist of images to keep warm, updated weekly based on invocation frequency. Cold images still work — they just take longer on first run.

---

## Five Principles for Production Tool Design

After eight months of building, breaking, and rebuilding tools for our agents, we distilled five principles. These overlap with Anthropic's recommendations in some places and diverge in others.

### Principle 1: One Tool Call, One Blast Radius

Anthropic recommends that tools be "self-contained, robust to error, and extremely clear with respect to their intended use." We agree, and add a stronger constraint: **one tool call should have a bounded blast radius.**

In V1.0, our dashboard generation tool produced the entire dashboard layout in a single LLM call — all charts, all queries, all data mappings. When one chart's query was wrong, the entire dashboard was wrong. The blast radius of a single tool call was the entire user interface.

V3.0 uses one MQL per component. Each chart card has its own independent query, its own data mapping, and its own error handling. If the billing chart's query fails, the contract chart and the social security chart still render correctly. The user sees partial results instead of a blank screen.

We deliberately rejected MongoDB's `$facet` operator, which would have allowed us to combine multiple chart queries into a single aggregation pipeline. The engineering rationale:

```
$facet advantages:  single network roundtrip
$facet drawbacks:   
  - One syntax error kills ALL charts (blast radius = entire dashboard)
  - $facet sub-pipelines can't use indexes efficiently  
  - 16MB BSON document limit applies to the combined output
  - Debugging is harder (which sub-pipeline failed?)

Our choice: parallel independent queries
  - Each query fails independently (blast radius = one chart)
  - Full index utilization per query
  - Easy to trace which chart caused which error
  - Async parallel execution = same wall-clock time as $facet
```

The cost is slightly higher network overhead (N queries instead of one). For ~300 users generating dashboards with 4-8 charts, this overhead is negligible. The reliability gain is not.

### Principle 2: Whitelist, Not Blacklist

Anthropic discusses "namespacing tools to define clear boundaries in functionality." Our implementation of this principle is more aggressive: we use **whitelists everywhere**, never blacklists.

The **component registry** (`component_registry.py`) is a whitelist of every chart type the AI can produce. Each entry declares:

- `dataBound`: whether the component needs a data query (a KPI card needs data; a title card doesn't)
- Required props and their types
- A one-sentence description that feeds the LLM's grounding context

The LLM cannot produce a chart type that isn't in the registry. There is no "creative mode" where it invents a new visualization. If the business needs a new chart type, an engineer adds it to the registry with proper validation rules.

The **semantic catalog** is a whitelist of every metric, dimension, and filter the AI can use in queries. The LLM cannot reference a field that isn't in the catalog. Unknown terms trigger a "this metric isn't registered" response, not a hallucinated query.

The **intent catalog** is a whitelist of every action the AI can take. The LLM's output enum is dynamically generated from the catalog at startup. Adding a new intent means adding one entry to the catalog — the enum, the prompt definition, the routing logic, and the vector corpus all derive from that single entry.

Why whitelists over blacklists? Because **blacklists fail open.** A blacklist that says "don't use these dangerous operators" will miss the next dangerous operator. A whitelist that says "only use these approved operators" blocks everything else by default, including threats you haven't imagined yet.

### Principle 3: Evidence Before Action

This is where we diverge most from Anthropic's framework. Anthropic discusses tool ergonomics — making tools easy for the agent to use correctly. We add a second dimension: **making tools hard for the agent to use incorrectly.**

Every tool that modifies the dashboard requires **evidence** — a specific piece of information that the LLM must extract from the user's input to prove the action is warranted. Evidence varies by action:

```
update_node:  targets (which chart) + change (what modification)
              → Both must be non-empty. The target ID must exist in the
                component summary. "Change something" is not evidence.

remove_node:  targets (which chart to delete)
              → Must match a real chart. Deleting a hallucinated ID 
                is a no-op, not an error.

override:     confirm_phrase (the user's EXACT word that means "redo")
              → The LLM must quote "重做" or "推倒" or "重新生成" etc.
                If it can't point to the evidence, the slot is empty,
                and code demotes to clarification.

filter_change: at least one of (month, completion, conditions_text)
               → require_any: all null = "user didn't ask to filter"
                 → code overrides to clarify.
```

The `override` design deserves special attention. Override wipes the entire dashboard and rebuilds from scratch — the most destructive action in the system. We require two layers of evidence:

1. The LLM must quote the user's exact word that indicates "redo." This forces the model to provide proof, not just confidence.
2. Even after the slot guard passes, the system triggers a LangGraph `interrupt()` — the user must click "confirm" in the UI before execution proceeds.

Three layers of protection: model classification → slot evidence → human confirmation. This is defense in depth for destructive operations.

### Principle 4: Human-Approved Examples, Never Auto-Promote

Anthropic recommends using evaluations to measure and improve tool performance. We agree, and add a constraint on the feedback loop: **AI-generated examples never automatically become training data.**

Our **example store** holds (question, approved_query) pairs that serve as few-shot context for the query generation tool. When the AI successfully generates a query that runs and returns data, the pair is saved as a **draft**. But it never automatically promotes to `published` status — a domain expert must verify the business semantics first.

```
                    ┌─────────┐     human      ┌───────────┐     human      ┌───────────┐
  AI generates  ──▶ │  Draft  │ ──────────────▶ │  Pending  │ ──────────────▶ │ Published │
                    └─────────┘    review       └───────────┘   approval     └───────────┘
                         │                                                         │
                         │         NEVER auto-promotes                             │
                         └─────────────────── ✕ ──────────────────────────────────┘
                                                                        ▼
                                                                 Enters AI context
```

Why not auto-promote? Because **a query that returns data is not necessarily a query that returns correct data.** In our domain, "contract count" could be filtered by six different status combinations. A query that returns 1,247 contracts might be off by 300 because it forgot the base population filter. The chart renders. The number looks plausible. Only a domain expert can tell it's wrong.

This is the "auto-Vanna" anti-pattern. Vanna.ai's training loop automatically promotes successful queries to the vector store. For personal analytics, this is fine. For enterprise BI where wrong numbers drive business decisions, it's dangerous.

### Principle 5: Evaluate Before You Deploy

Anthropic's post dedicates significant space to evaluation-driven development. We implement three layers of evaluation, each catching different failure types:

**Layer 1: Structural Validation (real-time, every query)**

The `board_validator` runs six checks on every dashboard before it reaches the user:

1. Component whitelist — only registered chart types
2. Required props — each chart type's mandatory properties
3. Data key source — props references must trace to pipeline output
4. MQL security — collection whitelist + forbidden operators
5. Chart role completeness — pie needs name+value, bar needs category+value
6. Field mapping — data_mapping fields must exist in pipeline output

If any check fails, the error is fed back to the LLM for a retry (max 2 attempts). This is the "validator fail → error → retry" loop that catches ~12% of hallucinations.

**Layer 2: Adversarial Probing (nightly)**

The `adversarial_probe.py` script runs a set of challenging queries against the production database every night. Each probe has structural expectations — expected chart type, expected fields, expected data series count. The probe also runs `scan_hallucination()`, which checks for:

- **Hallucinated fields**: field names in the pipeline that don't exist in the collection
- **Hallucinated enums**: filter values that aren't in the field's allowed values
- **Collection violations**: queries against collections not in the whitelist

Regressions trigger alerts. When a model upgrade degrades performance on 3 out of 50 probes, we know before any user sees it.

**Layer 3: Intent Classification Evaluation**

The `run_intent_eval.py` script tests the intent classifier in isolation. It bypasses the full HTTP graph and directly calls the classification function with test cases from `intent_eval_set`. Each case has an expected intent label. The script measures:

- Classification accuracy across all intents
- Per-intent precision and recall
- Confusion matrix (which intents are most often confused with each other)
- Impact of few-shot examples on accuracy (with vs without vector-retrieved examples)

This evaluation runs in CI and blocks merges if accuracy drops below the threshold.

---

## The ReAct Subgraph: Reusable Tool Execution

Our tool execution pattern follows the ReAct paradigm — the agent reasons about what to do, acts by calling a tool, observes the result, and decides whether to continue. But we implement this as a **reusable LangGraph sub-graph** that can be mounted in multiple contexts.

The same ReAct subgraph handles tool calls in the conversation domain (user says "look up contract #12345") and in the skill execution domain (agent decides to invoke a data cleaning skill). The sub-graph inherits the parent graph's checkpointer, so `interrupt()` calls in a deeply nested tool call bubble up correctly to the root graph.

### Human-in-the-Loop for Dangerous Tools

Not all tool calls are equal. Some tools read data (safe), others modify state (requires confirmation). We use **hooks that run before tool execution** to implement differentiated approval:

```
Tool call arrives at ReAct subgraph
    ↓
Pre-execution hook checks:
  - Is this a write operation? → interrupt() for human approval
  - Is this within the user's permission scope? → check authorization
  - Does the tool input pass validation? → reject malformed calls
    ↓
If all hooks pass → execute tool
If any hook fails → return error to agent for replanning
```

The `todo_guard` policy is one such hook: it checks whether a tool call would modify data that the user hasn't explicitly authorized. The guard runs in code, not in the prompt — we don't rely on the LLM to respect boundaries.

---

## MCP Integration: External Tools as First-Class Citizens

The Model Context Protocol (MCP) allows our agents to connect to external tool servers. In our architecture, MCP tools are treated as a fourth business domain alongside conversation, dashboards, and skills.

The key design decision: MCP tools run in the **same sandbox isolation** as external skills. An MCP server connected to our platform gets:

- Ephemeral K8s pods for execution
- NetworkPolicy isolation (no lateral movement to internal services)
- Result validation before the agent sees the response

The skill manifest defines capabilities and input/output schemas for each MCP tool. These schemas serve double duty: they constrain the agent's tool calls (preventing hallucinated parameters) and they document the tool's interface for human developers.

---

## What We'd Do Differently

### 1. Build the evaluation framework before the first tool

We built tools first and evaluations later. The debugging time in the first two months — when tools were producing wrong results and we had no systematic way to measure improvement — was the most painful period of the project. If starting over, we'd write the adversarial probes and intent evaluation before writing the first production tool.

### 2. Start at Level 3, not Level 1

We spent two months on Level 1 (raw MQL generation) before accepting it wouldn't work at enterprise accuracy levels. If starting over, we'd go directly to Level 3 (semantic IR + compiler). The compiler is ~800 lines of code. The debugging time saved by eliminating an entire class of syntax-level hallucinations would have been worth the investment from day one.

### 3. Use the intent catalog pattern from the start

The intent catalog — one dataclass per intent holding its description, target node, slot definitions, shortcircuit eligibility, and example utterances — was our V3.0 innovation. In V1.0 and V2.0, intent definitions were scattered across eight files. Every new intent required touching all eight. The catalog pattern eliminated this fragility. We should have adopted it from the first intent, not the eleventh.

### 4. Token-efficient tool responses from day one

Anthropic emphasizes "optimizing tool responses for token efficiency." We learned this lesson through cost monitoring: some tool responses included the full MongoDB document (all fields, including 50-field nested objects) when the agent only needed three fields. Trimming tool responses to include only the relevant fields reduced our token consumption by ~30%.

### 5. The `data_mapping` contract should be enforced, not suggested

Our `data_mapping` concept — the translation table between MongoDB field names and chart component role names — was initially optional. Components without explicit data_mapping would try to auto-detect the mapping. This failed silently in ~18% of cases (the "null chart" problem — charts that render but display no data). Making data_mapping mandatory and validating it against pipeline output fields eliminated this failure mode.

---

## Principles Summary

| Principle | Anthropic's Version | Our Extension |
|-----------|-------------------|---------------|
| Tools should be self-contained | ✅ Same | + One tool call = one blast radius (reject $facet) |
| Clear tool descriptions | ✅ Same | + Whitelist everything: components, metrics, intents |
| Reduce ambiguity | ✅ Same | + Evidence-based action: LLM must prove the action is warranted |
| Use evaluations | ✅ Same | + Never auto-promote AI-generated examples to training data |
| Token-efficient responses | ✅ Same | + Constraint level should match error cost |

The fundamental insight: in enterprise AI, **the tool's job is not to maximize the agent's freedom — it's to maximize the agent's accuracy within bounded freedom.** The spectrum from Level 1 (raw generation) to Level 4 (menu selection) isn't a ranking of sophistication. It's a ranking of reliability. The most sophisticated tool is the one that produces the right answer consistently, not the one that can do the most things.

---

## References

- [Writing effective tools for AI agents — with AI agents](https://www.anthropic.com/engineering/writing-tools-for-agents) — Anthropic, Sep 2025
- [Building effective AI agents](https://www.anthropic.com/engineering/building-effective-agents) — Anthropic, 2025
- [Model Context Protocol](https://modelcontextprotocol.io/) — Anthropic
- [Vanna.ai](https://vanna.ai/) — Training-data closed loop for text-to-SQL
- [RAGFlow](https://github.com/infiniflow/ragflow) — Categorize component with example injection
- [DB-GPT](https://github.com/eosphoros-ai/DB-GPT) — AWEL workflow with intent detection

---

*The system described serves ~300 enterprise users across four business domains. All data remains on-premise in compliance with China's PIPL. No internal identifiers or proprietary code are shared in this post.*
