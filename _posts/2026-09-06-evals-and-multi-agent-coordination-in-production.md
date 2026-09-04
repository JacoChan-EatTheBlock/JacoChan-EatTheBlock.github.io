---
layout: default
title: "Evals and Multi-Agent Coordination: What Anthropic's Playbook Looks Like in a Production Dashboard System"
date: 2026-09-06
categories: [AI Evaluation, Multi-Agent, LangGraph, Production]
---

# Evals and Multi-Agent Coordination: What Anthropic's Playbook Looks Like in a Production Dashboard System

*September 6, 2026 · 25 min read*

**Key Takeaways**

- Anthropic's evaluation taxonomy (task/trial/grader/transcript/outcome) maps cleanly to production AI systems — but enterprise dashboards require a three-layer evaluation architecture that combines real-time gates, nightly calibration, and human feedback loops.
- Multi-agent coordination in enterprise settings looks different from research-style orchestrator-worker patterns. When tasks share state (a dashboard layout), code-orchestrated DAG execution outperforms model-orchestrated delegation — at 2-3x the token cost of chat, not 15x.
- The evaluation system should exist before the second feature. We built ours in month three. The debugging pain in months one and two was the worst period of the project.

---

Anthropic recently published two engineering posts that resonated deeply with our production experience: "[Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)" and "[How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)." Both describe patterns we've independently discovered while building an enterprise AI dashboard system serving ~300 users.

This post maps Anthropic's frameworks to our production codebase, showing where the theory holds, where it needs adaptation, and what we learned that their posts don't cover.

---

## Part 1: Evaluation — From Anthropic's Taxonomy to Production Reality

### Anthropic's evaluation vocabulary

Anthropic defines a clean taxonomy: a **task** is a single test with defined inputs and success criteria. Each attempt is a **trial**. A **grader** scores some aspect of performance. The **transcript** is the complete record. The **outcome** is the final state in the environment.

This vocabulary is useful. But in a production dashboard system, the challenge isn't defining what an eval is — it's deciding *where* in the pipeline to evaluate, *how often*, and *what to do when evaluation fails*.

### Our three-layer architecture

We learned through painful experience that evaluation must happen at three different timescales, each catching different failure modes.

![Figure 1. Three-Layer Evaluation Architecture](/assets/images/fig22-eval-three-layers.svg)
*Figure 1. Layer 1 runs on every query (milliseconds). Layer 2 runs nightly (minutes). Layer 3 runs continuously through human feedback (days to weeks). Each layer catches failures the others miss.*

---

### Layer 1: Real-Time Gates (every query, ~2ms overhead)

Before any AI-generated dashboard reaches a user, it passes through a six-gate validator. This is our equivalent of Anthropic's code-based graders, but running in the hot path — not as a post-hoc evaluation, but as a production quality gate.

```
LLM Output (component spec + MQL pipeline + data mapping)
    │
    ▼
┌──────────────────────────┐
│ G1: Component Whitelist   │  Only registered chart types allowed
├──────────────────────────┤
│ G2: Required Props        │  Each chart type declares mandatory fields
├──────────────────────────┤
│ G3: Data Key Source       │  props references must trace to pipeline output
├──────────────────────────┤
│ G4: MQL Security          │  Collection whitelist + forbidden operators ($out, $merge)
├──────────────────────────┤
│ G5: Chart Role Check      │  Pie needs name+value; bar needs category+value
├──────────────────────────┤
│ G6: Field Mapping         │  data_mapping fields MUST exist in pipeline output
└──────────────┬───────────┘
          PASS │        │ FAIL
               ▼        ▼
           Execute    Error message → fed back to LLM → retry (max 2)
```

The critical design decision: **validation failure is not terminal — it's a retry signal.** When G6 detects that the LLM mapped `insurance_type_cn` but the pipeline only outputs `_id` and `count`, the specific error message ("you mapped X but only Y exists in the output") is fed back to the LLM. The LLM sees exactly what went wrong and can fix it. In our measurements, the first retry succeeds ~85% of the time.

This is fundamentally different from Anthropic's eval paradigm, where grading happens *after* the agent finishes. Our grading happens *during* generation, and failure triggers corrective action, not just a score.

**Overhead:** ~2ms per validation pass (pure Python, no LLM calls). For a typical dashboard with 4-6 components, that's 8-12ms of validation — invisible to users.

### Layer 2: Nightly Calibration (~50 canonical queries)

Anthropic describes **regression evals** as frozen test cases that detect when changes make things worse. Our nightly calibration suite is exactly this: ~50 canonical queries that run against the production database every night.

The key insight: **assert structure, not values.** The data changes daily. "Contract count = 42" is wrong tomorrow. But structural properties remain stable:

```python
# Example calibration case
{
    "question": "Show contract count by insurance type",
    "expect": {
        "collection": "v_contracts_base",       # Must use semantic view, not raw table
        "chart_type": "bar",                      # Bar chart, not pie
        "output_must_contain": ["insurance_type", "count"],  # These fields must exist
        "data_series_count": {"min": 2, "max": 20},          # Reasonable range
        "pipeline_must_not_contain": ["$out", "$merge"]       # Security check
    }
}
```

When a model upgrade or prompt change causes a previously-correct query to regress, the calibration suite catches it *before* it reaches production. We've blocked three deployments this way — each time, the regression was in date-range parsing (the LLM interpreted "last quarter" differently after an update).

**Evaluation tools from our codebase:**

| Tool | What It Tests | Anthropic Category |
|------|--------------|-------------------|
| `board_validator.py` | Structural validity of every dashboard | Code-based grader (hot path) |
| `adversarial_probe.py` | Hallucinated fields and enum values in MQL | Code-based grader (proactive) |
| `run_intent_eval.py` | Intent classification accuracy (~155 utterances) | Code-based grader (regression) |
| `run_dashboard_cases.py` | End-to-end dashboard generation | Code-based + model-based |
| `board_caliber_gate` | Deployment quality gate | Code-based (blocks deployment) |
| `narrate` node | LLM judges if dashboard "makes sense" | Model-based grader |
| User flags | Users mark dashboards as "wrong" | Human grader |

The `adversarial_probe.py` deserves special mention. It doesn't wait for a user to encounter a hallucination — it proactively generates adversarial queries designed to trigger known failure modes, then checks the output for hallucinated fields:

```python
def scan_hallucination(collection: str, pipeline: list) -> dict:
    """Run the pipeline through a validation gauntlet.
    Returns {"fields": [...hallucinated_fields], "enums": [(field, value)...]}
    """
    # Check every field reference in the pipeline against the actual schema
    # Check every enum value against the allowed values whitelist
    # Flag any field or value that doesn't exist in the database
```

### Layer 3: Human Feedback Loop (continuous)

Anthropic mentions human graders as the most expensive but most accurate evaluation method. In our system, human feedback is not just a grading mechanism — it's a *training data pipeline*.

```
User flags "wrong metric" on a dashboard
        │
        ▼
Draft created in example_store
(question + AI-generated query + user's correction note)
        │
        ▼
Domain expert reviews weekly
(corrects the query if needed, verifies business logic)
        │
        ▼
Published example enters AI context
(available as few-shot for future similar queries)
        │
        ▼
Future queries improve → fewer flags → cycle closes
```

The compounding effect is powerful: each corrected example makes similar future queries more accurate. After six months, our example store contains ~200 published examples, and the hallucination rate for queries similar to existing examples is under 2% — compared to ~12% for novel queries with no precedent.

### Capability evals vs. regression evals in practice

Anthropic distinguishes between **capability evals** (testing new abilities) and **regression evals** (ensuring existing abilities don't degrade). In practice, we found a third category: **graduation evals** that move cases between the two.

```
Capability eval: new intent type "plan_board" introduced
  → Start with ~60% accuracy on test cases
  → Iterate on prompt, few-shot examples, slot definitions
  → Accuracy climbs to 85%, then 92%, then 96%

Graduation: when accuracy hits >95% for 2 consecutive weeks
  → Those test cases move to the regression suite
  → They become the baseline for detecting future regressions
  → New capability eval cases replace them (next intent type)
```

This creates a ratchet effect: the regression suite grows monotonically, and each new capability eventually becomes a guaranteed baseline.

---

## Part 2: Multi-Agent Coordination — A Different Pattern

### Anthropic's research pattern: orchestrator-worker

Anthropic's multi-agent research system uses an orchestrator-worker pattern: a lead agent analyzes the query, spawns subagents with specific research tasks, subagents search independently with their own context windows, and the lead agent synthesizes results.

Key characteristics:
- Subagents have **independent context windows** (separation of concerns)
- The lead agent delegates via **natural language instructions** to subagents
- Token usage is ~**15x** chat (heavy parallelization)
- The system excels at **breadth-first search** where subtasks are independent

Their insight: "token usage by itself explains 80% of the performance variance." Multi-agent systems work primarily by spending enough tokens to solve the problem.

### Our pattern: planner-executor with shared state

Our enterprise dashboard system faces a fundamentally different coordination problem. When a user says "Add a pie chart for insurance types, move billing to the top, and switch to July," this is three operations — but they all share the same dashboard layout. The operations have dependencies and conflicts that must be resolved.

We use a **Compound Intent Planner** that looks superficially similar to Anthropic's orchestrator, but with a critical difference: **coordination is code-enforced, not model-delegated**.

![Figure 2. Compound Intent DAG Execution](/assets/images/fig23-compound-intent-dag.svg)
*Figure 2. The LLM decomposes the request into tasks. Code analyzes dependencies and conflicts. Parallel execution where safe, sequential where required. All tasks share one working tree.*

### How it works

**Step 1: LLM Decomposition.** The Compound Intent Planner (a single LLM call with `complete_typed`) decomposes the user's message into a task list:

```json
{
  "tasks": [
    {
      "task_id": "t1",
      "task_type": "board_action",
      "intent": "add_node",
      "question": "pie chart for insurance type distribution"
    },
    {
      "task_id": "t2",
      "task_type": "board_action",
      "intent": "reorder",
      "targets": ["billing_card"],
      "slots": { "position": "top" }
    },
    {
      "task_id": "t3",
      "task_type": "board_action",
      "intent": "filter_change",
      "slots": { "month": "2026-07" }
    }
  ]
}
```

**Step 2: Code-enforced DAG analysis.** Pure Python, zero LLM involvement:

```python
def _derive_compound_dependencies(tasks):
    dependencies = {task.task_id: set() for task in tasks}
    conflicts = {}
    
    # Rule 1: add_node + reorder on same target → dependency
    # (must add the card before you can move it)
    
    # Rule 2: remove_node + update_node on same target → conflict
    # (can't modify what you're deleting)
    
    # Rule 3: override + any other board action → conflict
    # (override wipes everything, nothing else makes sense)
    
    return dependencies, conflicts
```

**Step 3: Execution with shared working tree.** Tasks without dependencies execute in parallel. Tasks with dependencies execute in topological order. All tasks operate on the same `working_tree` — mutations accumulate. If T1 adds a card, T2 can immediately reference it for reordering.

### Why code-orchestrated, not model-orchestrated?

Anthropic's research system delegates coordination to the lead agent because research subtasks are genuinely independent — searching for "semiconductor supply chain" doesn't affect searching for "AI chip companies." Each subagent can work in isolation.

Dashboard operations are **not independent.** Moving a card requires knowing where it is. Deleting a card affects the layout of everything around it. Overriding the dashboard invalidates every other pending operation. These dependencies are **structural, not semantic** — they can be computed deterministically from the operation types and targets, without any LLM reasoning.

Letting the LLM handle coordination would mean:
1. Spending tokens on something code does for free
2. Introducing hallucination risk in the orchestration layer (the model might say "T2 doesn't depend on T1" when it does)
3. Making the execution order non-deterministic and harder to debug

Our principle: **the LLM proposes what to do; code decides when and whether to do it.**

### The cost comparison

| Metric | Anthropic Research | Our Dashboard |
|--------|-------------------|---------------|
| Token multiplier vs chat | ~15x | ~2-3x |
| Subagent context windows | Independent (separate) | Shared (one working tree) |
| Coordination mechanism | LLM natural language delegation | Deterministic code (DAG) |
| Ideal for | Breadth-first search, independent subtasks | Multi-operation edits with dependencies |
| Failure mode | Subagents duplicate work | Task conflicts block execution |
| Recovery | Lead agent re-delegates | Code detects conflict, falls back to single-intent |

The 15x vs 2-3x token difference is significant. Anthropic acknowledges this cost: "for economic viability, multi-agent systems require tasks where the value of the task is high enough to pay for the increased performance." In enterprise dashboard editing, the value of each operation is modest — we can't justify 15x token cost. Code-orchestrated execution gives us the coordination benefit at a fraction of the cost.

---

## The ReAct Subgraph: Reusable Agent Primitive

Both our conversation domain and skill execution domain need an agent↔tool loop with human-in-the-loop interrupt capability. Rather than implementing this twice, we built a **ReAct subgraph** that's written once and mounted everywhere.

```
┌─────────────────────────────────────────┐
│            ReAct Subgraph               │
│                                         │
│  ┌─────────┐     ┌─────────────────┐    │
│  │  Agent   │ ──▶ │  Tool Execution │    │
│  │ (reason  │ ◀── │  + HITL hooks   │    │
│  │  + plan) │     │  + todo_guard   │    │
│  └─────────┘     └─────────────────┘    │
│        │                                │
│        ▼                                │
│  ┌─────────┐                            │
│  │ Interrupt│ ← dangerous tool detected │
│  │ (human   │                           │
│  │ confirm) │                           │
│  └─────────┘                            │
└─────────────────────────────────────────┘

Mounted in:
  - conversation_graph (general tool use)
  - skill execution graph (SkillHub runtime)
  - data_query_subgraph (semantic query agent)
```

The subgraph inherits the root graph's checkpointer, so `interrupt()` in a deeply nested tool call bubbles up correctly. This is the LangGraph equivalent of function composition in traditional programming — write the primitive once, compose it everywhere.

---

## The Semantic Query Subgraph: A Mini Multi-Agent System

For the `query_data` intent (asking a question that requires generating and executing a MongoDB query), we run a mini pipeline that resembles Anthropic's orchestrator pattern:

```
User question: "What's the total service fee for confirmed bills in July?"
        │
        ▼
┌───────────────────────┐
│ Semantic RAG           │  bigram scoring + catalog lookup
│ (grounding candidates) │  → top candidates injected as context
└───────────┬───────────┘
            ▼
┌───────────────────────┐
│ Query Planner (LLM)    │  Writes SemanticQueryPlan (IR, not raw MQL)
│ + few-shot examples    │  Constrained by json_schema
│ + catalog brief        │  
│ + schema summary       │  
└───────────┬───────────┘
            ▼
┌───────────────────────┐
│ Compiler (Python)      │  IR → MongoDB aggregation pipeline
│ (deterministic, 0 LLM) │  Validates view names, field names, operators
└───────────┬───────────┘
            ▼
┌───────────────────────┐
│ Executor               │  Runs pipeline against MongoDB
│ + result validation    │  Checks non-empty, reasonable size
└───────────┬───────────┘
            ▼
┌───────────────────────┐
│ Narrator (LLM)         │  Summarizes result in natural language
│ + follow-up suggestions │  Recommends next questions
└───────────────────────┘
```

This is similar to Anthropic's research pipeline (plan → search → synthesize), but with a critical intermediate step: the **compiler**. The LLM never writes raw MQL. It writes a structured intermediate representation (`SemanticQueryPlan`), and deterministic Python code translates that to a valid aggregation pipeline. This eliminates an entire class of MQL syntax errors and security violations.

The `rag.grounding_block(question)` function is our version of Anthropic's Contextual Retrieval — it injects semantically relevant candidates (metric names, dimension names, their descriptions) so the LLM has grounding context for its query plan. The difference: Anthropic uses contextual embeddings + BM25 + reranking. We use bigram coverage scoring (better for Chinese business terms) + per-kind quota allocation (separate slots for metrics, derived metrics, and dimensions to prevent one category from crowding out others).

---

## Lessons for Production Eval Systems

### 1. Build evals before the second feature

We built our evaluation system in month three. The debugging pain in months one and two — when dashboards produced wrong numbers and we couldn't systematically determine why — was the worst period of the project. If we started over, the nightly calibration suite would exist before the first chart type.

Anthropic's observation matches: "Writing evals is useful at any stage in the agent lifecycle. Early on, evals force product teams to specify what success means."

### 2. The human feedback loop is your most valuable eval — and your slowest

Code-based graders catch structural failures in milliseconds. Model-based graders catch semantic oddities in seconds. But the human feedback loop — where a domain expert says "this number is wrong because 'confirmed' in our business means X, not Y" — is the only eval that catches business logic errors. It takes days to weeks, but each correction permanently improves the system.

### 3. Adversarial probes > random testing

Rather than running random queries and hoping to find hallucinations, we run targeted adversarial probes designed to trigger known failure modes. `scan_hallucination()` specifically checks for:
- Fields referenced in the pipeline that don't exist in the collection schema
- Enum values that aren't in the allowed list
- Collections that aren't in the whitelist
- Aggregation operators that are forbidden ($out, $merge, $currentOp)

This is cheaper and more effective than random sampling, because the failure modes are known (we've cataloged them from production incidents), and the probes can be exhaustive within each mode.

### 4. Multi-agent is not always the answer

Anthropic's finding — "some domains that require all agents to share the same context or involve many dependencies between agents are not a good fit for multi-agent systems" — exactly describes our dashboard domain. We tried model-orchestrated multi-agent coordination early on. The model would sometimes:
- Execute operations in the wrong order (moving a card before it was added)
- Miss dependencies (deleting a card another operation was about to modify)
- Duplicate work (two tasks both trying to modify the same chart)

Code-orchestrated DAG execution eliminated all three failure modes. The LLM's job is semantic decomposition (understanding what the user wants); the code's job is structural orchestration (deciding how and when to execute).

### 5. Graduation creates a ratchet

The pattern of moving test cases from capability evals to regression evals as they mature creates a one-way ratchet: the quality bar can only go up. Every production incident becomes a regression test case. Every new capability eventually becomes a baseline guarantee. Over eight months, our regression suite grew from 0 to ~50 cases, and none have been removed.

---

## The Evaluation-First Culture

The hardest part of evaluation isn't the infrastructure — it's the cultural shift from "we'll know it's good when users stop complaining" to "we'll know it's good when the calibration suite passes." The first approach is reactive and slow. The second is proactive and fast.

Anthropic says: "evals make problems and behavioral changes visible before they affect users, and their value compounds over the lifecycle of an agent." We'd add: the compounding happens in three ways simultaneously:

1. **The regression suite grows** — more test cases, wider coverage, fewer blind spots
2. **The example store grows** — more precedent, higher accuracy for similar queries, fewer hallucinations
3. **The team's mental model grows** — each eval failure teaches the team about a new failure mode, which informs better prompts, better tools, and better validation

This triple compounding means the system improves faster the longer it's been in production. Month eight is dramatically easier than month one — not because the problems are simpler, but because the evaluation infrastructure makes them visible, measurable, and fixable.

---

*The system described serves ~300 enterprise users generating natural language dashboards against MongoDB. Evaluation and coordination patterns are described generically; no internal identifiers are shared.*
