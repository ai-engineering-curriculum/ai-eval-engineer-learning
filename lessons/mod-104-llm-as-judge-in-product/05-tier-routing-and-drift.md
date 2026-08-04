# Judge-Tier Routing and Drift Monitoring

## Motivation

At this point in the module you have a defensible rubric (chapter 01), a bias-controlled pipeline (chapter 02), a judge-model choice you can back up (chapter 03), and a human-calibrated agreement number (chapter 04). The last two problems are runtime problems.

**Judge cost.** Running a frontier judge on every sampled trace of every online SLI on every surface, twice per pairwise decision (chapter 02's swap-and-average), is a real bill — often the largest line item in the eval infra. You need a rule that spends the frontier judge where its accuracy matters and drops to a mid-tier or open-source judge where accuracy is adequate but scale is what dominates. That rule is **judge-tier routing.**

**Judge drift.** The judge itself changes. Frontier vendors ship silent model updates behind stable snapshot names, deprecate the snapshot you calibrated against, upgrade the tokenizer, or shift RLHF behaviour. Open-source weights don't shift, but your prompt template does — a rubric edit, a framework upgrade. Either way, the judge you calibrated on Wednesday is not necessarily the judge running on Thursday. **Drift monitoring** is the runtime discipline that catches that.

This chapter walks both problems in the shape mod-107's online-eval loop will consume. If judge cost is 50% of your eval bill and rising, tier routing is where you fix it; if a release-gate has ever moved because the underlying judge silently changed, drift monitoring is what would have caught it.

## Core concepts

### The three tiers

The tiering from chapter 03 lands in the runtime as three concrete lanes with different cost / accuracy profiles.

| Tier | What it is | Per-call cost | Human-agreement (typical, per your calibration) |
|---|---|---|---|
| **Frontier** | The current-generation chat API (a top-of-line model of your chosen vendor) via DeepEval / Braintrust / Promptfoo | Highest | Highest — the ceiling your calibration compares everything to |
| **Mid-tier** | A cheaper API model from the same vendor, or a mid-size model from another frontier vendor | Materially lower | Usually lower than frontier; calibrate to know by how much on your rubric |
| **Open-source** | A self-hosted judge (Prometheus 2, JudgeLM, or a fine-tuned model on your own gold set) behind vLLM / TGI in your VPC | Lowest marginal (GPU cost is fixed) | Variable — depends heavily on how far your rubric is from the OSS judge's training rubrics |

Numbers change every release cycle; the ordering is stable. Verify the per-tier calibration on your rubric before you rely on the ordering — it is not axiomatic that "frontier > mid-tier > OSS" on your surface.

<!-- needs-research: on the next research cycle, refresh the specific tier boundaries with current published pricing and available snapshot names for at least Anthropic Claude, OpenAI GPT, and Google Gemini. -->

### The routing decision: what goes to which tier

A useful routing rule has three inputs: **decision stakes**, **judge confidence**, and **cost budget**. The typical pattern:

1. **Default the routine online-eval traffic to the OSS tier.** Sampled traces on the mod-102 head-sampling channel run through the OSS judge for the normal SLI dashboard. Cost is bounded by GPU capacity, not per-call price. Chapter-04 calibration for this tier decides the SLI thresholds you can defend.
2. **Escalate to the mid-tier when the OSS judge is uncertain.** If the OSS judge returns a `tie` on pairwise (chapter 02), a score at the ambiguity boundary on absolute (e.g., a 3 on a 1–5 with rationale phrases like "unclear"), or fails a self-consistency check on rerun, re-judge on the mid-tier. Uncertain judgements are a small share of traffic and get the accuracy boost where it matters.
3. **Reserve the frontier tier for release-gate and calibration-set scoring.** Release-gate A/B (candidate model vs incumbent), the pairwise A/B on the mod-101 golden set at merge time, and the periodic re-scoring of the calibration set itself (see drift monitoring below). Low volume, high stakes, budget the whole tier for these.

The routing rule is a first-class object — a small function or a config table that mod-107's online-eval reads. Version-control it; log the tier that judged each decision (`app.judge_tier`) on the EVALUATOR span from mod-102.

Sketch (framework-agnostic pseudocode):

```python
def route_judge(decision, oss_result=None):
    if decision.is_release_gate or decision.is_calibration_rescore:
        return "frontier"
    if oss_result is None:
        # First pass — cheap tier.
        return "oss"
    if oss_result.confidence == "uncertain" or oss_result.winner == "tie":
        return "mid"
    return "oss"

def judge(decision, criterion):
    tier = route_judge(decision)
    result = judges[tier](decision, criterion)
    if tier == "oss" and (result.confidence == "uncertain"
                          or result.winner == "tie"):
        tier = route_judge(decision, oss_result=result)
        result = judges[tier](decision, criterion)
    span.set_attribute("app.judge_tier", tier)
    return result
```

The tiering is per-criterion, not per-surface — a single trace might have its groundedness rubric judged on the OSS tier and its safety rubric judged on the frontier tier.

### Managing to a cost budget

Judge cost is a first-class SLO for your eval program. mod-101's cost-family SLIs treat *product* cost; the same pattern applies here to *eval* cost. Two mainstream tactics:

**Weekly / monthly cost ceiling per tier.** Frontier tier gets a hard weekly $ cap; the routing rule falls back to mid-tier when the cap is reached. mod-107's online-eval loop reads the cap and shifts its sampling accordingly. Reserve at least 20% headroom for release-gate and calibration bursts.

**Per-criterion budget.** Some criteria — safety, refusal, groundedness on high-stakes surfaces — deserve more of the budget than others. Encode this as a per-criterion tier default in the routing config: safety criteria default to frontier, monitoring criteria default to OSS.

Watch the two failure modes:

- **Underspend on the frontier tier while release-gate accuracy drops.** Usually a symptom of the routing rule not surfacing enough "uncertain" cases up-tier. Recalibrate the uncertainty threshold.
- **Frontier tier consumes the whole budget and blocks online eval.** Usually a symptom of misplaced defaults — a criterion is defaulting to frontier when it should be OSS. Audit the config quarterly.

Log the per-tier spend on the same dashboard as the judge quality metrics. When someone asks *"why did the eval bill triple last week,"* the answer is one query on the EVALUATOR-span tier attribute.

### Judge drift: what changes underneath you

Four independent sources of drift affect a judge in production:

1. **Judge model version.** The vendor ships an update behind the same snapshot name (rare on named snapshots, common on default / alias endpoints), deprecates the snapshot you pinned, or upgrades the tokenizer. Effect: the same rubric prompt returns different scores.
2. **Rubric prompt template.** A well-meaning edit — one word in an anchor, a schema tweak, a whitespace change that shifts token-level cache — invalidates your calibration. Effect: same judge, different scores.
3. **Input distribution.** Your users start asking different questions (a marketing push, a product change, a season). Effect: same judge, same rubric, but your calibration set no longer represents the input population.
4. **Generator model.** The underlying surface generator changes. Effect: judge scores drift because the joint distribution of (input, candidate) shifts; often mistaken for judge drift.

Only (1) and (2) are truly *judge* drift; (3) and (4) are population / generator drift that shows up in judge metrics. Distinguish them in your monitor.

### The drift monitor

The drift monitor is a scheduled job — daily or weekly, per criterion — that answers: *is the judge scoring today's inputs the way it scored last month's inputs, given a known-fixed sample?*

The minimum shape:

1. **A frozen replay set.** ~50 recorded (input, retrieved-context, candidate) triples, sampled once, held constant. Not the calibration gold set — a separate replay set so calibration can drift too and you notice.
2. **A frozen rubric prompt template hash.** The rubric the monitor uses is version-pinned and hashed. If the current production rubric prompt hash is different, that is drift source (2) and the monitor annotates the alert.
3. **A scheduled rerun.** Nightly or weekly, re-judge the frozen replay set with the current production judge configuration and record each score.
4. **A tracked statistic.** For absolute: the mean score on the replay set, and its distribution across levels. For pairwise: the win-rate and tie-rate on the replay set. Optional: correlation with the last-week baseline.
5. **An alert threshold.** A shift beyond a chosen bound (e.g., mean score changes by > 0.3 on a 1–5 scale, or tie rate changes by > 15%) fires an alert.

When the alert fires, the on-call runs a **triage**:

- Compare the current rubric prompt hash to the frozen hash. Different → drift source (2). Re-calibrate or roll back the rubric.
- Compare the current judge model version / snapshot / weights hash to the frozen one. Different → drift source (1). Re-run chapter-04 calibration and update the SLI thresholds if needed.
- Compare production input distribution (topic mix, length distribution) to the calibration set's. Different → drift source (3). Refresh the calibration set.
- Otherwise → drift source (4) or a genuinely new bias. Escalate to the model-evaluation-engineer peer for full-methodology re-baseline.

Attach the drift metrics to the same dashboard as the tier spend and the bias diagnostics (chapter 02). A on-call opening the judge dashboard should see: current-week per-tier spend, cross-family agreement rate, position-bias flag rate, drift-replay statistic, and the last calibration timestamp per criterion. Anything not on that dashboard is not being watched.

### Re-calibration triggers

The chapter-04 calibration document lists re-calibration triggers. The drift monitor is what enforces them. Wire them explicitly:

| Trigger | Action |
|---|---|
| Judge model version / snapshot changes (chapter-03 judge choice) | Re-run chapter-04 calibration; update SLI threshold; announce to consuming teams |
| Rubric prompt template hash changes | Same |
| Frozen replay statistic drifts beyond threshold | Triage as above; likely re-calibration |
| Calibration set older than N months (default 3–6) | Refresh calibration set; re-run calibration |
| A per-criterion pass-rate on live traffic moves outside its historical band | Investigate before re-calibrating — this may be a real regression, not judge drift |

The trigger and the action are the same three fields on every row: *what changed, what evidence, what action*. That's what a governance-family peer signs off on at mod-112.

### The frontier-upgrade playbook

Frontier vendor upgrades are the most common single cause of a bad release-gate call. When a new snapshot is announced (or the aliased default silently rolls forward):

1. **Do not upgrade the production judge yet.** Pin the current snapshot in your framework config until you have run the new one against the frozen replay set and the calibration gold set.
2. **Run the new judge on both sets side-by-side with the incumbent.** Compare per-item, not just aggregate. Any item where the two disagree by more than a level on absolute, or by more than the tie boundary on pairwise, is a flagged item.
3. **Re-calibrate against the human labels** on the calibration gold set. If the new judge's chapter-04 statistic is materially different from the incumbent's, publish the new number and update SLI thresholds — do not silently swap.
4. **Swap the production judge only after the new-judge calibration is documented** and the drift monitor's alert threshold has been reset to the new baseline.
5. **Keep the incumbent judge available for at least one drift-monitor cycle** so a rollback is one-line.

Two or three days of playbook per frontier upgrade is cheap compared to a silent quality regression caught only after a bad release. Budget the time.

### What tier routing and drift monitoring do *not* solve

These two disciplines keep a *well-calibrated* judge honest as it runs. They cannot rescue:

- A bad rubric (chapter 01) — no amount of routing or drift alerting will make a vague anchor produce useful judgements.
- Uncontrolled bias in the pipeline (chapter 02) — you will faithfully monitor a biased number and think it is stable.
- A judge that has never been calibrated (chapter 04) — a drift alert against a never-calibrated baseline is a distraction.

Do all four chapters. Then tier routing and drift monitoring are the piece that keeps the whole apparatus consumable by mod-107, mod-109, and mod-112.

## Summary

- **Three judge tiers**: frontier (release-gate + calibration), mid-tier (uncertain-case escalation), open-source (routine online monitoring). Route per-decision on stakes + judge confidence + cost budget; version-control the routing rule; log `app.judge_tier` on every EVALUATOR span.
- **Cost is an SLO for the eval program itself**. Cap per-tier spend weekly; watch the two failure modes (frontier-underspend hurting accuracy, frontier-overspend blocking online eval).
- **Judge drift has four sources**: model version, rubric prompt, input distribution, generator. Only the first two are true judge drift; the monitor must distinguish them.
- **The drift monitor** runs a frozen replay set nightly / weekly, tracks a chosen statistic, alerts on a threshold breach, and triages by comparing model / rubric / distribution / generator against the pinned baseline. Alerts wire to the chapter-04 re-calibration triggers.
- **Frontier upgrades follow a playbook**: pin, compare on frozen sets, re-calibrate, publish the new number, swap, keep rollback for a cycle. Do not silently take the vendor's new default.
- The judge dashboard is one place with per-tier spend, cross-family agreement, position-bias flag rate, drift-replay statistic, and last-calibration timestamps. Anything not on that dashboard is not being watched.

This closes the module. The next modules — mod-105 (RAG-triad off the judge pipeline this chapter defines), mod-106 (CI gates that consume the frontier-tier release-gate judge), mod-107 (the online-eval loop that runs the OSS tier), mod-108 (the safety-family rubrics this pipeline judges), mod-109 (human review of judge disagreements), mod-110 (the lineage table for `app.judge_*` spans), mod-111 (judge cost inside the surface's cost model), and mod-112 (the release-gate architecture that reads all of the above) — all consume the artefacts this module produced. Ship the rubrics, controls, calibration, and monitor as one bundle; do not skip pieces.
