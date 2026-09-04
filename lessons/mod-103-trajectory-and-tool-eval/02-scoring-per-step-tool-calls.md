# Scoring Per-Step Tool Calls: Name, Arguments, Ordering

## Motivation

Chapter 01 established that the unit of evaluation for an agentic surface is the **trajectory**, not the final reply, and that a scorer has to answer five questions on the trajectory record alone. This chapter is the workhorse implementation of question one — *did the agent take the right steps?* — decomposed into three axes: the tool **name** at each step, the tool **arguments** at each step, and the **order** of the steps relative to their dependencies.

Per-step scoring is what makes credit assignment mechanical. Two failure modes make this concrete:

- **Right final answer via wrong tools.** The support agent from chapter 01 answered `"Where is my order 12345?"` correctly, but it did so by calling `sql_query` on the raw `orders` table instead of the authorised `docs.search` retriever. A final-answer judge scores this 5/5. A per-step name scorer flags it as `expected=docs.search, actual=sql_query, verdict=wrong_tool`. Without the per-step verdict, the golden shortcut is invisible and the ambient-permissions blast radius sits in production until a security review catches it.
- **Wrong final answer via right tools.** The agent called all the right tools with the right arguments in the right order, but the terminal `llm.reply` fabricated a shipment date. That is a fix on the reply LLM's prompt (mod-104) or on the reply model choice (mod-111), not on the plan. If the trajectory scorer conflates the two — one number for "did it work" — the on-call has to bisect by hand. If the scorer emits `name_ok=true, args_ok=true, order_ok=true, final_answer_ok=false`, the fix routes itself.

Both patterns show up in production traffic every week. Neither is visible if the scorer collapses to a scalar. The rest of this chapter defines the sub-scores that keep them visible.

## Core concepts

### The reference trajectory

A per-step scorer needs a **reference** — sometimes called *gold* or *target* — against which to compare the observed trajectory. The reference is not (usually) a single canonical sequence; it is a *specification* of the trajectories that count as acceptable for a given task. There are four common forms, in increasing order of expressiveness.

| Form | What it captures | Fixture cost | Brittleness | Best fit |
|---|---|---|---|---|
| **Strict** | Exact ordered sequence of `(tool_name, arguments)` tuples | Low (one canonical script per task) | High — any legitimate alternative fails | Tightly scripted flows; contract-test-style single-path tasks; regression tests where you *want* to catch a re-ordering |
| **Set** | Multiset of tool calls; order flexible | Low–medium | Medium — accepts spurious interleavings | Bag-of-tools tasks where dependencies are trivial (e.g., "call `weather` and `calendar`, order does not matter") |
| **DAG** | Partial order — a directed acyclic graph of prerequisites; any topological sort is valid | Medium (author the DAG per task family) | Low — accepts every legitimate execution | Multi-step agents with real data dependencies; the default for production trajectory eval |
| **Programmatic** | A rule expressed as code — e.g., "must call `open_ticket` iff no retrieved doc scored ≥ 0.7" | Medium–high (per-task predicates; needs test coverage of the predicate itself) | Low — expresses conditional behaviour no static schema can | Tasks where the correct trajectory depends on runtime observations (retrieval scores, tool return values, user preferences) |

For most production surfaces, **DAG** is the default and **programmatic** is the escape hatch for the ~10–20% of tasks whose correct trajectory branches on tool returns. Strict is a bad choice for anything an LLM authored the plan for — the model will legitimately reorder equivalent-effect steps and your scorer will thrash. Set is a bad choice as soon as there is any data dependency at all.

The reference is authored *once per task family* and lives in the fixture store (mod-110). Each production trajectory is scored against the reference for its task family; the mapping from user input to task family is upstream of the scorer (chapter 05's Inspect `Task` and `Sample` shapes; <https://inspect.aisi.org.uk/tasks.html>).

### Name matching

Exact string equality on `tool.name` is the starting point and it is not enough on its own. The four adjustments that keep name matching stable across a real production surface:

- **Aliases.** A tool renamed between model versions (`docs.search` → `docs.retrieve` in a Q3 refactor) needs an alias table so historical traces still score. Keep the alias table alongside the reference-trajectory store (mod-110); do not hard-code aliases in the scorer.
- **Namespaced prefixes.** MCP-style servers namespace their tools — `crm.lookup` from one server, `lookup` from another, potentially with the same *shape* but different semantics. The Model Context Protocol spec (<https://spec.modelcontextprotocol.io/>) is the primary reference for the naming shape. Score `namespace + name` as one unit; do not silently strip the namespace.
- **Retries collapsed.** A tool that returned an HTTP 502 and was retried by the harness produces two identical adjacent `TOOL` spans in the trace. The retry is a harness event, not an agent decision. Collapse consecutive identical `(tool_name, arguments)` calls that differ only in success status before comparing to the reference. Retention of the retry count belongs to the budget scorer (chapter 04), not the name scorer.
- **Case, whitespace, and Unicode normalisation.** Compare `tool.name` after `NFKC` normalisation and casefold. Tools authored by hand and tools authored by an OpenAPI-to-schema converter drift on capitalisation surprisingly often.

The output of the name matcher for one step is a small verdict: `{"expected": "docs.search", "actual": "sql_query", "aliased": false, "match": false, "reason": "wrong_tool"}`. If `match` is false, the arguments and ordering matchers for that step are still run — you want the full attribution, not a short-circuit.

### Argument matching

Arguments are where most of the interesting failure modes live, and where the scorer has to make a deliberate cost / precision trade-off. The levels below are ordered cheapest → most expensive; pick the cheapest level that catches the failure classes your surface cares about.

**Level 1 — Structural.** Same keys are present; each key's value has the correct JSON type (string / number / boolean / array / object / null). This catches "the model forgot to supply `order_id`" and "the model passed `order_id: "12345"` where the tool wanted an integer". It does not catch semantic mistakes.

**Level 2 — Exact-value.** JSON-equal after **canonicalisation**: keys sorted, whitespace normalised, dates coerced to ISO-8601 (RFC 3339 profile — <https://datatracker.ietf.org/doc/html/rfc3339>), numeric strings coerced to numbers where the schema is numeric. This catches every mismatch of literal content and passes only when the argument object is byte-identical to the reference under the canonical form.

**Level 3 — Semantic-equivalence.** Per-tool normalisation rules that treat semantically equivalent arguments as equal. For a search-style tool: `{"query": "order 12345 status"}` and `{"query": "status of order 12345"}` should be equal for scoring purposes. The rule set is *per tool*, authored by the tool owner, and stored with the reference trajectories. Common patterns: token-set match on natural-language fields, order-insensitive comparison on filter arrays, `strip()` on IDs.

**Level 4 — AST-based.** For tools whose arguments are themselves code — a `python.exec` tool, a `sql.execute` tool, a `shell.run` tool — parse the argument string to an AST and compare structurally rather than textually. The Berkeley Function-Calling Leaderboard's evaluator uses an AST comparison for function-call arguments (<https://gorilla.cs.berkeley.edu/leaderboard.html>), and the pattern generalises: two SQL queries that differ only in whitespace and alias names are the same query; two Python calls that differ only in the order of keyword arguments are the same call. AST comparison catches the ways a text comparator false-fails.

**Level 5 — Judge-graded.** Defer per-argument grading to an LLM judge. This is the escape hatch for free-form arguments where none of the above works — e.g., a `send_email.body` field. Judge design lives in mod-104; the trajectory scorer's job is to invoke the judge and store the returned verdict, not to author the judge prompt.

The same call, scored at each level, produces different verdicts. Consider `expected` vs `actual`:

```json
{
  "expected": {"query": "status of order 12345", "top_k": 3},
  "actual":   {"top_k": 3, "query": "order 12345 status"}
}
```

- Level 1 (structural): both objects have keys `{query, top_k}` with correct types → **match**.
- Level 2 (exact-value after canonicalisation): keys sort to `[query, top_k]` in both, but `"status of order 12345"` != `"order 12345 status"` → **no match**.
- Level 3 (semantic-equivalence, token-set on `query`): `{"status","of","order","12345"}` == `{"order","12345","status"}` (ignoring stop-word `of`) → **match**.
- Level 4 (AST-based): not applicable — arguments are not code.
- Level 5 (judge-graded): a judge returns `equivalent=true, rationale="both queries request status for order 12345"` → **match**.

The scorer emits which level was used per argument so a reviewer can see whether the pass came from a strict comparator or a lenient one. Cost per level rises by roughly an order of magnitude from Level 2 to Level 3 (custom code), and again from Level 3 to Level 5 (judge inference). Reserve Level 5 for fields that can be shown to matter by an earlier level's failure rate — do not judge every argument.

### Ordering

Given a reference trajectory (DAG or programmatic), an observed step order is either a valid execution of that reference or it is not. Three matchers cover the useful cases.

**Strict-order edit distance.** Treat the observed tool-name sequence and the reference tool-name sequence as strings; compute a Levenshtein-style edit distance where insertions, deletions, and substitutions carry per-tool weights. A missing critical tool ( `docs.search`) costs more than an extra logging call. Report the raw distance plus the aligned diff. Use this when the reference is **strict** and you care about *how far off* the observed sequence was, not just whether it matched.

**Partial-order (topological) matcher.** Given a DAG of prerequisites — "`docs.search` before `open_ticket`", "`crm.lookup` before `send_notification`" — ask: is the observed sequence a valid topological sort of the DAG? Python's standard library ships `graphlib.TopologicalSorter` (<https://docs.python.org/3/library/graphlib.html>, added in 3.9) for this; its `static_order()` yields one valid topological order and its `is_active()` / `get_ready()` / `done()` API supports incremental validation as the observed steps are consumed. The verdict is a boolean plus, on failure, the first edge violated (`crm.lookup expected before send_notification; observed send_notification at step 3 before crm.lookup at step 5`).

**Interleaving-aware matcher.** Two independent chains (retrieval sub-plan and CRM sub-plan) can legitimately interleave in any order that respects each chain's internal DAG. A pure topological check accepts these correctly, but the *reporting* has to make the interleaving legible. Represent the observed trajectory as a sequence of `(chain_id, step_id)` pairs and align each chain against its sub-DAG independently; report per-chain compliance plus a global picture. Trace-viewer swim-lanes are the natural UI for this (chapter 05).

A worked ASCII example. The reference DAG for the support agent has two sub-chains:

```
docs.search ──▶ open_ticket           (docs chain)
crm.lookup  ──▶ send_notification     (CRM chain)
```

Valid interleaving #1 (interleaved, docs chain first at end):

```
crm.lookup → docs.search → send_notification → open_ticket    OK
```

Valid interleaving #2 (docs chain fully before CRM chain):

```
docs.search → open_ticket → crm.lookup → send_notification    OK
```

Invalid interleaving (dependency violated):

```
open_ticket → docs.search → crm.lookup → send_notification    FAIL
                                                              (open_ticket before docs.search)
```

The topological matcher accepts the first two and rejects the third, and the failure verdict names the violated edge explicitly.

### Sub-scores rolled into a per-step verdict

The output of the per-step scorer for one step is a JSON record with an explicit sub-score per axis. A minimal shape:

```json
{
  "step_id": 3,
  "span_id": "a4c9e2…",
  "tool_name": {
    "expected": "docs.search",
    "actual":   "sql_query",
    "aliased":  false,
    "match":    false,
    "reason":   "wrong_tool"
  },
  "arguments": {
    "level":   "structural",
    "match":   true,
    "details": {"missing_keys": [], "type_errors": []}
  },
  "order": {
    "matcher":         "topological",
    "match":           true,
    "violated_edge":   null
  },
  "cost_ok":  true,
  "verdict":  "wrong_tool"
}
```

Four properties this shape gives up front:

1. **Attribution is explicit.** `verdict=wrong_tool` names the axis that failed; a downstream aggregator can bucket by verdict.
2. **The other axes are still evaluated.** Arguments and order are scored even though the name did not match — a reviewer can see whether the wrong tool was called with sensible arguments (probably a prompt bug) or nonsense arguments (probably a schema bug).
3. **The scoring level is recorded.** `arguments.level=structural` tells you the pass was cheap; if a Level-3 semantic-equivalence would have caught something, you have to re-score, not guess.
4. **`cost_ok` is a hand-off point.** Chapter 04's budget scorer fills in `cost_ok`; this chapter's scorer leaves it `true` unless a tool-local budget was violated (e.g., a single retrieval that exceeded a per-tool timeout).

Chapter 03 defines the roll-up from per-step verdicts to a per-trajectory score and specifies the partial-credit rubric. This chapter's contract is just to emit the per-step records reliably.

### Fixture design

**How many reference trajectories per task family?** Enough to cover the branching structure of the DAG, not enough to overfit. A good rule of thumb: at least one reference per leaf of the task family's decision tree, plus one reference per historically-observed failure class. For the support agent's `"where is my order"` family, that is roughly: (a) order found + shipped, (b) order found + not shipped + policy applies, (c) order found + not shipped + no policy match → escalate, (d) order not found. Four references, one per branch of the task family's business logic.

**Handling non-determinism.** A retriever that returns different chunk orders across runs, a CRM tool whose payload embeds a wall-clock timestamp — comparing arguments byte-for-byte fails on both. The right abstraction is **equivalence classes**: for each non-deterministic field, define a normalisation function that maps every legitimate variant to a canonical form. Timestamps → epoch-day bucket. Retrieval results → set of document IDs regardless of order. Free-text tool descriptions → their SHA-256 hash. The scorer applies the normalisation before comparison; the equivalence-class definitions live with the reference trajectories in mod-110. Chapter 07 addresses the harder case — reproducing the exact observation deterministically — via a recorded tool-response table.

**Golden-set drift.** The product ships a new tool. Every existing reference trajectory that could plausibly have used that tool is now under-specified — the scorer says "match" for a trajectory that in the new world would be suboptimal. Two counter-measures: (1) a periodic **coverage audit** that flags reference trajectories that have not been touched since a tool inventory change, and (2) a **shadow re-scoring** run whenever the tool inventory changes that re-scores a recent traffic sample against both the old and the new reference set, so drift is visible before it hits CI. Both live in the mod-110 fixture-management workflow.

**Where fixtures live.** Reference trajectories are versioned artefacts in a store that mod-110 owns; the scorer takes a `(task_family_id, fixture_version)` and materialises the reference at scoring time. Do not embed reference trajectories in the scorer's own source tree beyond a small smoke-test set — the fixture-lifecycle problem is real, and mod-110 solves it.

## Example — the support-agent trajectory scored

Take the bad trajectory from chapter 01. Recall the observed sequence (compressing spans that are not agent-decision steps):

```
step 1: llm.plan       →  tool call = sql_query({"query": "SELECT * FROM orders WHERE id=12345"})
step 2: tool.sql_query →  observation
step 3: llm.plan.2     →  tool call = docs.search({"query": "order 12345 status", "top_k": 3})
step 4: retriever.docs.search  → observation
step 5: retriever.docs.search.2 (redundant, query="order 12345 shipping", top_k=3)
step 6: llm.plan.3     →  tool call = crm.lookup({"order_id": "12345"})
step 7: tool.crm.lookup (with two retries collapsed by the name matcher)
step 8: llm.plan.4     →  tool call = open_ticket({"order_id": "12345", "reason": "no_shipping_update"})
step 9: tool.open_ticket → observation
step 10: llm.reply     →  terminal
```

The reference trajectory for the `"where is my order — not shipped, policy applies"` branch of the task family (DAG form):

```
docs.search  ──▶ open_ticket
crm.lookup   ──▶ (informs reply; no downstream tool)
```

Expected tool set: `{docs.search, crm.lookup, open_ticket}`, plus the terminal reply. Expected argument on `docs.search.query`: a token set including `{"order", "12345", "status"}`. Expected argument on `open_ticket.reason`: one of `{"no_shipping_update", "shipping_delay", "delivery_exception"}`.

Per-step verdicts:

| Step | Tool observed | Name verdict | Args verdict (level) | Order verdict | Overall |
|---|---|---|---|---|---|
| 2 | `sql_query` | **fail** (`expected=docs.search`) | n/a (wrong tool) | fail (`sql_query` not in DAG) | **wrong_tool** |
| 4 | `docs.search` | pass | pass (Level 3, token set matches) | pass | ok |
| 5 | `docs.search` (2nd call) | pass | pass (structural), but **redundant** (same task-family query, no new information) | pass | **redundant_call** |
| 7 | `crm.lookup` | pass (retries collapsed) | pass (Level 2, exact) | pass | ok |
| 9 | `open_ticket` | pass | pass (Level 3, `reason` in allowed set) | pass (`docs.search` preceded it, as DAG requires) | ok |

The verdict emitted by this chapter's scorer for the whole trajectory is a list of the five records above, plus a small header:

```json
{
  "trajectory_id": "c31f7a…",
  "task_family":   "order_status_not_shipped_policy",
  "reference_version": "v3.2",
  "per_step": [ /* the five records above */ ],
  "summary": {
    "name_pass_rate":   0.8,
    "args_pass_rate":   1.0,
    "order_pass_rate":  0.8,
    "flagged_verdicts": ["wrong_tool", "redundant_call"]
  }
}
```

Two things this verdict makes visible that the final-answer judge missed:

- **Step 2 was a golden shortcut.** `sql_query` bypassed the authorised retriever. `verdict=wrong_tool` routes this to a security-review queue independent of whether the reply was correct.
- **Step 5 was a redundant call.** The second `docs.search` cost tokens and latency for no information gain. `verdict=redundant_call` feeds the cost-attribution roll-up (chapter 04) and gives the on-call a specific span to look at when the cost budget was violated.

Chapter 03 shows how the `summary` block and the per-step verdicts fold into a scalar trajectory score with partial credit — and, critically, how the partial-credit rubric preserves the `flagged_verdicts` field so downstream aggregation (mod-107, mod-112) can still bucket by failure class.

## Summary

- A per-step tool-call scorer grades each step on three axes: **name**, **arguments**, **order**. It does not collapse to a scalar; a downstream rubric does that.
- The scorer compares against a **reference trajectory** in one of four forms — strict, set, DAG, or programmatic. DAG is the production default; programmatic is the escape hatch for trajectories that branch on runtime observations.
- Name matching is exact-string plus four adjustments: aliases, namespaced prefixes (MCP-style), retries collapsed, Unicode / case normalisation.
- Argument matching has five levels of increasing cost and precision — structural, exact-value, semantic-equivalence, AST-based, judge-graded. Record which level scored each call so a reviewer can re-score at a stricter level without guessing.
- Ordering is validated by one of three matchers — strict-order edit distance, partial-order (topological, via `graphlib.TopologicalSorter`), or interleaving-aware. Failure reports name the violated edge, not just a boolean.
- The scorer emits a per-step JSON verdict with explicit sub-scores and a hand-off field (`cost_ok`) for chapter 04's budget scorer. Chapter 03 rolls per-step verdicts into a per-trajectory score.
- Fixture design is a first-class concern: enough references to cover the DAG's branches, equivalence classes for non-deterministic observations, and a coverage-audit process for golden-set drift. Fixtures live in mod-110.

Chapter 03 defines the final-answer rubric and the partial-credit roll-up that consumes this chapter's per-step verdicts.
