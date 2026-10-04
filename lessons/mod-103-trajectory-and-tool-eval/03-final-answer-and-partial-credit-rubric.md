# Final-Answer Scoring and the Partial-Credit Rubric

## Motivation

Chapter 02 emitted a per-step verdict for every step in the trajectory — name, arguments, ordering, each with its own sub-score. That verdict is the raw material; it is not the number a product reviewer wants to see. A reviewer wants two things: *was the final reply to the user correct?* and *how much credit does the trajectory as a whole deserve?* This chapter answers both — on top of the per-step verdicts chapter 02 produced, and underneath the budget gates chapter 04 will wrap around the result.

The gap between those two questions is the whole point of the chapter. A trajectory can be *right-step / wrong-answer* — all the authorised tools were called in the right order, but the terminal `llm.reply` fabricated a shipment date. A trajectory can be *wrong-step / right-answer* — the golden shortcut from chapter 01, where the correct reply came out of an unauthorised tool. A trajectory can be *mostly right* — one redundant retrieval, one slightly-wrong argument, otherwise clean. A single scalar collapses all three to a number and loses the attribution a reviewer needs. The rubric this chapter builds preserves the attribution: it rolls per-step verdicts and the final-answer score into a scalar *and* retains the dimensional breakdown next to it, so the downstream aggregator (mod-107) can bucket by failure class and the on-call (chapter 07's replay workflow) can route the fix.

## Core concepts

### Final-answer scoring

The terminal `llm.reply` span's `output.value` is a string (or a structured block, for tools whose action is to emit a structured object). Scoring it is a strictly smaller problem than trajectory scoring — the whole single-turn eval literature applies — but the choices mirror chapter 02's argument-matching ladder, because the same cost / precision trade-off is in play.

**Exact-match.** Normalise the reply (lower-case, whitespace, strip Markdown fences) and string-compare to the reference. Appropriate for closed-vocabulary surfaces: a `classify_intent` tool that returns one of five labels, a `date_extract` call that returns ISO-8601. Zero cost at scoring time; brittle for anything free-form.

**String-overlap metrics.** Classic NLP overlap scores — token F1 (SQuAD-style; <https://arxiv.org/abs/1606.05250>), ROUGE for summarisation (Lin, 2004 — <https://aclanthology.org/W04-1013/>), BLEU for translation-shaped outputs (Papineni et al., 2002 — <https://aclanthology.org/P02-1040/>). Useful as cheap offline signals and as regression tripwires. Not useful as a product SLI for free-form replies, because they are both lenient on fluent-nonsense and strict on legitimate paraphrases.

**Programmatic verifiers.** The reply is scored by a rule the task family's spec defines — a regex that must match, a JSON Schema the reply must validate against, a shell command whose exit status is the verdict. SWE-bench's scorer is the paradigm case: a candidate patch is applied to a repository and the project's own test suite is run; the verdict is the test-suite exit status (Jimenez et al., "SWE-bench: Can Language Models Resolve Real-World GitHub Issues?", <https://arxiv.org/abs/2310.06770>; SWE-bench Verified, OpenAI 2024, <https://openai.com/index/introducing-swe-bench-verified/>). Programmatic verifiers are the gold standard where they exist — they have no judge bias and they compose with CI — and they are the reason chapter 06 pays so much attention to borrowing *task shape* rather than task content from public benchmarks.

**Reference-based judge.** An LLM-as-judge compares the reply to a reference answer and emits a graded verdict. The reference-based variant is the one with the most defensible calibration; the free-form variant (chapter 06 of mod-101 and all of mod-104) is harder and lives there. For trajectory eval this chapter's rule is narrow: *use a judge when no cheaper method works, and record which judge, which prompt version, and which bias controls were in play*.

**Reference-free judge.** The judge is given the task and the reply and asked to score the reply on named rubric dimensions. This is where most of mod-104 lives; this chapter's rule is to call out of the trajectory scorer into the judge rather than embed the judge prompt here.

**Structured-output verifiers.** When the final action is a tool call that emits a structured object (`send_email({to, subject, body})`, `create_jira_ticket({...})`), scoring is a hybrid: structural + semantic on the fields the surface's SLIs name. Treat this as the chapter-02 argument matcher applied to the terminal action, rather than as a separate scorer. The output is one row of the per-step table where `step=terminal`.

The scorer records which method was used per trajectory so a reviewer can see whether the pass came from a strict comparator (exact-match on a label) or a lenient one (judge-graded on a free-form body). The level of inspection a reviewer applies to a 1.0 answer-credit should scale with the method; a token-F1 of 0.88 is a different claim than a judge verdict of "equivalent".

### Groundedness as a separate axis

The reply can be *content-correct* and still *ungrounded* — the information in it is right, but the trajectory never retrieved the authoritative source. Chapter 01's worked trajectory is exactly this case: the reply states a shipment ETA, the ETA happens to be right, and the agent read it from `sql_query` on the raw `orders` table rather than from the `docs.search` policy doc. A single `answer_correct` axis conflates the two; the rubric separates them.

- `answer_credit` ∈ [0, 1]: did the content of the reply match the reference by the selected scoring method?
- `groundedness_credit` ∈ [0, 1]: were the claims in the reply supported by observations the trajectory legitimately produced?

For retrieval-shaped surfaces, groundedness is computed by RAGAS-style faithfulness checks (<https://docs.ragas.io/>) over the RETRIEVER span content and the reply; mod-105 owns the depth. The trajectory rubric's job is to *carry* a `groundedness_credit` column and let mod-105's scorer fill it in — the roll-up weights it independently of `answer_credit` so the right-answer / wrong-source case does not pass unseen.

### The partial-credit rubric

The rubric is a weighted sum, with the weights pre-registered per task family and the per-axis credits coming from chapter 02 (name, args, order), from this chapter (answer, groundedness), and from chapter 04 (cost, latency). A reasonable default shape:

```
trajectory_score =
      w_name        * name_credit
    + w_args        * args_credit
    + w_order       * order_credit
    + w_answer      * answer_credit
    + w_groundedness* groundedness_credit
    + w_budget      * budget_credit        // chapter 04
subject to:
    sum(w_*) == 1.0
```

Three design rules keep this honest:

**Pre-register the weights.** The weights live in the task family's eval plan (mod-101) and are versioned alongside the reference fixtures. Do not re-tune weights after seeing results — that is p-hacking the rubric. If a team wants to change weights, open a PR on the eval plan and re-score historical trajectories under both.

**Floors, not just weighted sums.** A weighted sum alone lets a trajectory with `name_credit=0, answer_credit=1` compensate for the wrong-tool axis. For regulated surfaces, that is the wrong trade-off: the ambient-permissions blast radius does not get to be averaged away. Impose per-axis **floors**: a trajectory with `name_credit < 1.0` or a `verdict ∈ {wrong_tool, unauthorised_tool, dependency_violated}` fails the rubric regardless of the weighted sum. The floor set belongs to the surface, not the scorer; chapter 04's rollout gates read the same floors.

**Preserve the dimensional breakdown.** The output of this chapter's scorer is both the scalar `trajectory_score` and the full per-axis vector plus the `flagged_verdicts` list from chapter 02. Downstream aggregation (mod-107, mod-112) buckets by `flagged_verdicts`; a scalar alone is not sufficient to answer "which failure class is regressing this week?".

### Credit-to-step attribution

The per-step verdicts from chapter 02 come with a `step_id`, a `span_id`, and a `verdict` (`ok`, `wrong_tool`, `wrong_args`, `redundant_call`, `ordering_violation`). The rubric's roll-up function has to decide how a step-level verdict contributes to axis credits.

The simplest shape — one that holds up for most surfaces — is **worst-case over axis**: `name_credit = min(name_match ∈ {0, 1} across all required steps)`. One wrong-tool step gives `name_credit=0`; the trajectory still carries the other axes. For `args_credit` and `order_credit`, worst-case is also the default. Means and softer aggregations hide the one load-bearing failure in a cloud of ok steps.

For `answer_credit` and `groundedness_credit`, the attribution is to the terminal step; multi-reply surfaces (an agent that emits intermediate replies the user sees) carry a weighted sum over the visible replies, with weights proportional to each reply's user-facing impact.

The one surface where worst-case is wrong: tasks whose correct trajectory has **optional** or **best-of-N** steps. A planning agent that can legitimately call `calculator` *or* `python.exec` for the arithmetic step should not be penalised for picking either. The reference trajectory type for these tasks is **programmatic** (chapter 02), and the rubric reads `name_credit` from the programmatic predicate's return value, not from a strict-name match.

### Verdict taxonomy

The `flagged_verdicts` field carries the names chapter 02 emitted. The rubric extends it with the final-answer and groundedness verdicts so downstream aggregation sees everything in one list. A minimum vocabulary:

| Verdict | Meaning | Attribution |
|---|---|---|
| `wrong_tool` | Expected name ≠ actual name at a required step | per-step |
| `unauthorised_tool` | Actual tool outside the task family's allowed set | per-step |
| `wrong_args` | Argument match failed at the chosen comparator level | per-step |
| `ordering_violation` | DAG edge violated | per-step |
| `redundant_call` | Semantically-equivalent call repeated with no new information | per-step |
| `missing_required_tool` | Required tool never called before terminal step | trajectory |
| `extra_tool` | Tool called that was not in the required set and not permitted | per-step |
| `fabricated_claim` | Reply contains a claim not supported by any observation | terminal |
| `ungrounded_but_correct` | Reply content matches reference but not supported by trajectory observations | terminal |
| `wrong_final_answer` | Reply content fails the selected scoring method | terminal |
| `cost_over_budget` | Trajectory cost exceeded the SLO ceiling | trajectory (chapter 04) |
| `latency_over_budget` | Trajectory latency exceeded the SLO ceiling | trajectory (chapter 04) |
| `steps_over_budget` | Step count exceeded `max_steps` | trajectory (chapter 04) |
| `retry_budget_blown` | Retry count exceeded `max_retries_per_tool` on at least one tool | trajectory (chapter 04) |

Do not invent extra verdicts case-by-case. If a new failure mode is appearing in production, add it to this list in the eval plan and re-score historical traffic against the extended vocabulary.

### Robust aggregation across many trajectories

A per-trajectory score is not a release decision; mod-106 and mod-107 roll up across a sample. The rubric's job is to make the roll-up honest. Two patterns:

- **Pass-rate rather than mean score.** `pass_rate = fraction of trajectories where trajectory_score ≥ task_family.pass_threshold AND no floor was tripped`. Pass-rate is the shape most product reviewers reason about and the shape mod-106's merge gate consumes.
- **Per-slice reporting.** Alongside pass-rate, carry `pass_rate per task_family`, `per cohort` (locale, feature flag, traffic source — see mod-101 exercise-01), and `per verdict`. The verdict slice is what tells the on-call *why* pass-rate moved; the family slice is what tells the PM *where*.

Neither mean-score nor single-threshold pass-rate alone is sufficient. A rubric that only emits a mean lets a single catastrophic failure average out; a pass-rate without the verdict slice loses the "why".

## Example — the support-agent trajectory rolled up

Chapter 02 emitted five per-step records for the bad trajectory, with the summary:

```json
{
  "name_pass_rate":   0.8,
  "args_pass_rate":   1.0,
  "order_pass_rate":  0.8,
  "flagged_verdicts": ["wrong_tool", "redundant_call"]
}
```

The task family's weights (pre-registered in the eval plan) and floors:

```yaml
# order_status_not_shipped_policy.eval.yaml  (illustrative)
weights:
  name:          0.20
  args:          0.10
  order:         0.10
  answer:        0.20
  groundedness:  0.25
  budget:        0.15
floors:
  trip_on_verdict:
    - wrong_tool
    - unauthorised_tool
    - ordering_violation
    - fabricated_claim
pass_threshold: 0.80
```

The reply content matches the reference ("ticket opened; ETA in 2 days"), so `answer_credit = 1.0`. The groundedness check fails: the ETA came from `sql_query`, not from the authoritative `docs.search` policy doc, so `groundedness_credit = 0.3` (illustrative). The budget scorer (chapter 04) flags four violations, so `budget_credit = 0.0`.

Weighted sum (purely illustrative numbers):

```
0.20 * 0.0   +   // name:        wrong_tool at step 2 → worst-case 0
0.10 * 1.0   +   // args:        all passed
0.10 * 0.8   +   // order:       step 2 off-DAG
0.20 * 1.0   +   // answer:      content matches reference
0.25 * 0.3   +   // groundedness: ungrounded ETA
0.15 * 0.0       // budget:      multiple blown
= 0.0 + 0.1 + 0.08 + 0.2 + 0.075 + 0.0
= 0.455
```

The weighted sum is already below the 0.80 pass threshold. The floor also trips: `wrong_tool` is in `trip_on_verdict`, so the trajectory **fails** regardless of the weighted sum. The emitted verdict:

```json
{
  "trajectory_id":      "c31f7a…",
  "task_family":        "order_status_not_shipped_policy",
  "reference_version":  "v3.2",
  "rubric_version":     "v1.4",
  "trajectory_score":   0.455,
  "pass":               false,
  "floor_tripped":      "wrong_tool",
  "axes": {
    "name_credit":          0.0,
    "args_credit":          1.0,
    "order_credit":         0.8,
    "answer_credit":        1.0,
    "groundedness_credit":  0.3,
    "budget_credit":        0.0
  },
  "flagged_verdicts": [
    "wrong_tool",
    "redundant_call",
    "ungrounded_but_correct",
    "cost_over_budget",
    "latency_over_budget",
    "steps_over_budget"
  ],
  "per_step": [ /* from chapter 02 */ ],
  "method": {
    "answer_scorer":        "exact_match_normalised",
    "groundedness_scorer":  "ragas_faithfulness_v0.1"
  }
}
```

Three things to notice about this record. First, `pass=false` is dominated by the floor, not the weighted sum. A reviewer glancing at the trajectory immediately knows *why* it failed: `wrong_tool`. Second, `answer_credit=1.0` is still visible — the reply content was fine, and the fix is not on the reply LLM. Third, `flagged_verdicts` carries both the chapter-02 verdicts and the chapter-04 budget verdicts, so a weekly aggregation of production traffic can bucket by any of them.

## Common pitfalls

- **Averaging a wrong-tool step away.** A pure weighted sum lets a trajectory with one wrong-tool step and five ok steps score ~0.8. Impose floors on the verdicts that constitute real failures for your surface; do not rely on the sum alone.
- **Letting the judge grade the trajectory, not the answer.** It is tempting to hand the whole trajectory to a judge and ask "did the agent do a good job?". The problem is that the judge collapses the dimensional breakdown and introduces judge bias on a per-step basis. The rubric here invokes a judge *at specific points* (per-argument where structural comparators fail; on the terminal reply when exact-match is wrong) and treats the judge's output as one input among many.
- **One rubric for all task families.** The weights, floors, and pass thresholds are *per task family* (`order_status`, `refund_request`, `return_label`, etc.). A single rubric for a whole surface over-weights the dominant task family and under-serves the tail.
- **Tuning weights to the current sample.** The weights are pre-registered in the eval plan and versioned. If a team re-tunes weights because this week's pass-rate dropped, that is p-hacking. Changes to weights are PRs on the eval plan and trigger a historical re-score.
- **Rubric drift from spec drift.** The reference answers and the rubric both live in mod-110's fixture store, versioned together. A spec change that moves what "correct" means has to bump both at once; otherwise the scorer silently drifts.
- **Hiding partial credit from the on-call.** The `axes` block is load-bearing. A dashboard that shows only `trajectory_score` loses the attribution the rubric exists to preserve.

## Summary

- The final answer is scored by one of six methods — exact-match, overlap, programmatic verifier, reference-based judge, reference-free judge, structured-output verifier — chosen by the task family's spec. Record the method on every verdict.
- Separate `answer_credit` from `groundedness_credit`. A correct reply drawn from an unauthorised source is a failure, not a pass; mod-105's faithfulness scorer fills the groundedness column.
- The rubric is a weighted sum of per-axis credits — name, args, order, answer, groundedness, budget — with weights pre-registered per task family and floors on named verdicts for regulated surfaces.
- Credit-to-step attribution is worst-case per axis for the plan axes (name, args, order), terminal-step for the answer axes (answer, groundedness), and trajectory-level for the budget axis.
- Extend chapter 02's verdict vocabulary with final-answer and budget verdicts; carry the full `flagged_verdicts` list through the roll-up so downstream aggregation can bucket by failure class.
- Report pass-rate with per-slice (task family, cohort, verdict) breakdowns, not mean score. A mean hides the one catastrophic failure a product reviewer actually needs to see.
- Pitfalls — floor-free weighted sums, judge-grading the whole trajectory, one rubric for all families, post-hoc weight tuning, spec/rubric drift — all collapse the attribution the rubric exists to preserve.

Chapter 04 defines the budget scorer that fills `budget_credit`, the four budget-family verdicts, and the rollout gates that enforce them.
