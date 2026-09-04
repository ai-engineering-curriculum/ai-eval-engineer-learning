# Model Routing Evaluation with Cohort Preservation

## Motivation

Sooner or later every production LLM system stops being *one model on every request* and starts being *the cheap model on the easy cases, the frontier model on the hard cases*. The economics push there: frontier models are 10 – 50× more expensive per token than mid-tier or open-source alternatives, and 60 – 80 % of production traffic is easy enough that a mid-tier model handles it at indistinguishable quality. Routing captures that gap.

The problem is that routing is where **average quality lies loudest**. A router that scores 99 % on the easy cases and 60 % on the hard cases has the same average as a router that scores 90 % across the board — and product ships the average without noticing that the enterprise-support cohort (which happens to concentrate in the hard cases) dropped from 4.3 to 3.1 rubric points. Two weeks later customer success escalates; nobody can attribute the regression because the routing was silently gated on classifier confidence and the eval never held the router accountable per cohort.

This chapter builds the routing-eval discipline the multi-objective report (chapter 02) needs. Three pieces:

- **The cohort-preservation contract** — no cohort's quality drops more than δ vs. the frontier baseline. Pre-registered, per-cohort, published on the report.
- **The router-error separation** — a routing failure (sent an easy case to the expensive model, or a hard case to the cheap model) is a *different* number from a model failure. The report separates them; the fix is different.
- **The "always-frontier-when-unsure" fallback** — every router has a fallback tier the classifier can back off into. Evaluating the fallback rate is part of evaluating the router.

Chapter 04 will apply a related but distinct discipline (paired replacement regression) to the case where the whole model is being swapped rather than routed to. Both chapters use the multi-objective report shape from chapter 02.

## Core concepts

### The four routing patterns

Not every "routing" system is the same shape. The eval methodology depends on which pattern is in play; the report shape is identical.

| Pattern | How it decides | Where it fails |
|---|---|---|
| **Tier-based (rules)** | Explicit rules — enterprise tenants → frontier; free tier → mid-tier; long-context requests → the long-context model | Easy to explain, hard to tune; misses edge cases the rules do not anticipate |
| **Classifier-gated** | A small model / classifier scores every request; above threshold → frontier, below → mid-tier | Classifier drift; hard-case cohort concentration in the below-threshold bucket |
| **Difficulty-scored** | An estimate of task difficulty (perplexity of a small probe, retrieval-uncertainty, chain-length prediction) chooses tier | Requires the difficulty proxy to correlate with actual quality gap between tiers |
| **Cache-first** | A semantic cache hit returns without inference; a miss falls through to the primary model | Cache-hit false-positives; quality drift on cached answers when upstream data changes |

Real systems chain these. A typical stack is: **cache-first** (answer from cache if hit) → **tier-based rules** (regulated tenants → frontier regardless) → **classifier-gated** (classify remaining traffic and split) → **always-frontier fallback** on classifier low-confidence. The eval methodology handles the composition — each layer is a "routing decision" the eval attributes independently.

The report never says just "the router." It names the layer: `cache_hit`, `tier_rule[regulated]`, `classifier_route[mid-tier]`, `fallback_frontier`. Every request's trace carries which layer routed it (a `routing_decision` span attribute); the report groups by that attribute.

### The cohort-preservation contract

The heart of routing eval.

Formalise: given a **baseline candidate** `B` (usually "frontier model, no routing") and a **routed candidate** `R`, for every named cohort `c`:

```
quality(R, c) >= quality(B, c) - delta_c
```

Where `delta_c` is a **pre-registered, per-cohort tolerance** — a small number chosen before the routing eval is run. Typical values:

- `delta_c = 0.0` — critical cohorts (e.g., `regulated`, `enterprise-paying`, `safety-sensitive`). Zero regression tolerated.
- `delta_c = 0.3` – `0.5` — important cohorts (`en-US`, `pro-tier`, `paid`).
- `delta_c = 1.0` – `1.5` — best-effort cohorts (`free-tier`, `experimental`) — larger tolerance because the cost saving justifies more quality drop.

The scale is rubric-dependent. On a 0 – 5 rubric, `0.5` is a substantive quality change; on a 0 – 100 pass-rate, the analogue is 2 – 3 percentage points. Chapter 04 of mod-104 walked how to calibrate `delta_c` against inter-rater agreement.

The contract is a **per-cohort** guarantee, not an average. A router that respects the contract on 9 of 10 cohorts and fails on the 10th fails the contract — the failing cell is red, the ship decision is `hold` or `reject`, and the routing policy is tuned. Averaging the cells across cohorts destroys the guarantee.

Three properties matter:

- **Pre-registered.** Written before the routing candidate exists. Tolerance chosen against business priorities, not against the numbers the routing happens to produce.
- **Published on the report.** Every routing report shows the contract, the per-cohort deltas, and the pass / fail per cell. No hidden thresholds.
- **Signed by product.** The tolerance is a product decision (how much quality is the cost saving worth on each cohort). Engineering does not set δ alone.

### Baseline candidate: what to compare against

The most common baseline mistake is comparing the routed candidate to the *previous production system* — which may itself be a routed system with its own regressions the previous eval missed. The comparison silently drifts toward whichever earlier baseline was weakest.

Fix: for a routing eval, the baseline is **the frontier-tier model on every request** (i.e., "no routing"). The comparison is the pure quality gap the routing introduces, without cumulative drift.

Two exceptions worth naming:

- **Frontier-only is prohibitively expensive to run at eval-set scale.** The frontier candidate is scored on a *representative* sample; the routed candidate is scored on the full set; the cohort deltas are computed on the intersection. Chapter 04 of mod-106 covered the replay-bundle discipline.
- **The frontier model has known cohort weaknesses of its own** (some frontier models score worse on some languages than a mid-tier competitor). The baseline in that case is not "frontier-only" but "best-of-tier-per-cohort" — the strongest model available for that cohort, regardless of tier. The routing eval measures against the true best, not the assumed-best.

The report always names the baseline explicitly (`baseline: frontier-only` or `baseline: best-of-tier-per-cohort`). Comparing routed candidates against each other without a common baseline is how a series of gradual regressions looks like "improvement."

### The router-error separation

A misrouted request looks like a model failure on the report unless the eval methodology separates them.

Consider a request from the `regulated` cohort that the classifier scored as easy (mistakenly) and sent to the mid-tier model, which produced a mediocre answer. Two errors happened:

1. **Router error.** The classifier should have sent this to the frontier tier; it did not.
2. **Model error (conditional).** *Given* it was routed to mid-tier, the mid-tier's answer was mediocre.

If the report treats this as a single "mid-tier failed on the regulated cohort" data point, the wrong fix follows — you fine-tune the mid-tier when the correct fix is to tune the router. Conversely, if the router is perfect and the mid-tier still fails on hard cases within its tier, the fix is on the model, not the router.

The separation:

- **Route the eval set through the routed system.** Record which tier each case was routed to.
- **Re-route every case through the baseline (frontier-only) and every non-frontier tier.** Score all versions.
- **For each case, compute:**
  - `route_correct[c]` = would the *ideal* router (an oracle post-hoc that picked whichever tier produced the best score) have chosen the same tier? If yes, router-correct; if no, router-error.
  - `quality_gap[c]` = baseline quality − routed quality.
  - `router_attributable_gap[c]` = for router-error cases, `quality(oracle_tier) - quality(routed_tier)`. This is the gap the router is responsible for.
  - `model_attributable_gap[c]` = for router-correct cases, `quality(baseline) - quality(routed)`. This is the gap the routed tier is responsible for, given it was correctly chosen.
- **Report both.** The routing report has a column for `router_attributable_gap` and a column for `model_attributable_gap`. The sum is the total gap; each column tells you where to fix.

Two consequences:

- **A router with 100 % route-correctness but a big model gap** → the mid-tier is under-powered for the cases the router (correctly) sent to it. The fix is either "raise the router threshold so more cases go to frontier" or "improve the mid-tier."
- **A router with a big router-attributable gap but 0 model gap** → the classifier is wrong. The fix is on the classifier, not on either model.

The oracle-post-hoc router is a *measurement* device, not a shipping candidate — it uses information (the scores) unavailable at inference time. It exists to attribute the router-error separately from the model-error.

### The always-frontier-when-unsure fallback

Every classifier-gated router has a fallback: when the classifier's confidence is below some threshold, fall through to the frontier tier. The fallback is a safety valve against router errors — a low-confidence classification is more likely to be wrong; sending it to frontier is cheap insurance.

Two evaluation questions the report answers:

- **What is the fallback rate?** Percentage of requests the classifier hands to the fallback. Too low → the fallback is not doing its job; too high → the routing is not saving cost. The rate is a report metric, tracked per cohort.
- **What is the quality of the fallback tier's own performance?** The fallback catches the router's low-confidence cases; those cases are also the *hardest* cases. The fallback tier (usually frontier) has to perform on them at least as well as it performs on the full set — otherwise the fallback is not actually protecting quality.

The rule of thumb the module recommends: **the classifier's confidence threshold should be tuned so the fallback rate on each critical cohort is at least 20 – 30 %**. Lower and you are leaking hard cases into the mid-tier; higher and the router is not saving enough cost to justify the machinery. Every deployment tunes its own threshold — the number is starting-point folklore, not law.

### The routing report shape

An extension of the multi-objective report from chapter 02. Same four columns, same cohort matrix, same pre-registered rule — plus routing-specific rows.

```
# Routing report — support-bot cache→rules→classifier→fallback

decision_id:            RR-2026-08-24-support-bot
routing_policy_id:      support-router-v11 (hash sha256:8b1...4a1)
baseline:               frontier-only (support-bot-v3 frontier tier on every request)
eval_set:               support-eval-v14  (hash sha256:f2c...c13)
pre_registered:         rules/support-router-2026-08-20.md (sig: PM=alex, ENG=jamie)
delta_c (contract):
  regulated=0.0, enterprise=0.3, en-US=0.5, en-GB=0.5, es=0.5, de=0.5, ja=0.5,
  short=0.5, long=0.5, free-tier=1.0

## Routing outcome — traffic distribution

| layer              | share | avg cost / req | quality mean |
|--------------------|-------|----------------|--------------|
| cache_hit          | 22%   | $0.0002        | 4.31         |
| tier_rule[regulated]| 8%   | $0.0184        | 4.24         |
| classifier[mid]    | 47%   | $0.0059        | 4.02         |
| classifier[frontier]| 15%  | $0.0184        | 4.21         |
| fallback_frontier  | 8%    | $0.0184        | 4.14         |

Weighted mean cost / req:   $0.0072  (baseline: $0.0184, -61%)
Weighted mean quality:      4.14     (baseline: 4.22, -0.08 avg)

## Cohort quality Δ vs. baseline (rubric points; ⚠ = fails delta_c)

|              | regulated | enterprise | en-US | en-GB | es    | de    | ja    | short | long  | free  |
|--------------|-----------|------------|-------|-------|-------|-------|-------|-------|-------|-------|
| Δ            | -0.02     | -0.11      | -0.06 | -0.09 | -0.14 | -0.19 | -0.31 | -0.03 | -0.24 | -0.51 |
| δ_c          | 0.0       | 0.3        | 0.5   | 0.5   | 0.5   | 0.5   | 0.5   | 0.5   | 0.5   | 1.0   |
| pass?        | ⚠ over    | ✓          | ✓     | ✓     | ✓     | ✓     | ✓     | ✓     | ✓     | ✓     |

## Router-error separation (rubric points, absolute)

|              | regulated | enterprise | en-US | en-GB | es    | de    | ja    | free  |
|--------------|-----------|------------|-------|-------|-------|-------|-------|-------|
| router-gap   | 0.02      | 0.03       | 0.01  | 0.01  | 0.04  | 0.05  | 0.12  | 0.19  |
| model-gap    | 0.00      | 0.08       | 0.05  | 0.08  | 0.10  | 0.14  | 0.19  | 0.32  |

Interpretation: on `ja` and `free`, the mid-tier's quality gap is larger than the router's;
on `regulated`, the router is the whole gap (small, but on the tightest δ). Fix: raise
classifier threshold so more `regulated` traffic falls through to fallback.

## Fallback rate per cohort

|              | regulated | enterprise | en-US | en-GB | es    | de    | ja    | free  |
|--------------|-----------|------------|-------|-------|-------|-------|-------|-------|
| fallback     | 71%       | 34%        | 6%    | 9%    | 11%   | 12%   | 17%   | 2%    |

## Decision

Pre-registered rule:
  Ship the routing policy if all of:
    (1) cohort-preservation contract passes on every cohort           [FAIL: regulated -0.02 > 0.0]
    (2) weighted mean cost / req drops ≥ 50%                          [PASS: -61%]
    (3) streaming p95 does not increase vs. baseline                  [PASS: -0.4 s]
    (4) safety column: zero OWASP category regressions                [PASS]

Result: HOLD.
  Rule (1) fails on regulated cohort by 0.02 rubric points against a δ=0.0 tolerance.
  Recommended fix: raise classifier confidence threshold for regulated tenants so
  fallback rate on that cohort rises from 71% to ≥ 90%.
```

The report reads like the chapter-02 shape with two additions — the routing-layer traffic distribution and the router-vs-model error separation. Product signs the same way; the pre-registered rule is the same shape; the rollback trigger references mod-107 the same way.

### Evaluating routing under drift

The routing eval is not one-and-done. Three drift sources chapter 03 of mod-107 already flagged apply here:

- **Traffic drift.** The distribution of incoming requests shifts. A router tuned on last quarter's traffic may over- or under-serve fallback on this quarter's. The routing eval re-runs on the mod-107 online-loop scored-row set at least quarterly.
- **Model drift.** Vendors update snapshots. A mid-tier model that was 0.4 points behind frontier last month may be 0.2 or 0.6 behind this month. The routing report's baseline uses the *current* frontier snapshot; the routed candidate uses the *current* mid-tier snapshot. Every snapshot bump is a new candidate.
- **Classifier drift.** The router's classifier is itself a model. Its calibration decays. The router's classifier accuracy is a metric on the routing report — dropping below a pre-registered threshold triggers re-training.

Every routing report is stamped with the mod-110 lineage keys. The `baseline` candidate id, the `routing_policy_id`, the classifier's own hash, and the model snapshots are all part of the report's identity.

### Anti-patterns to avoid

**"Average preservation."** The router "preserves" quality on average, ignoring the cohort where it does not. This is the failure the cohort-preservation contract exists to prevent.

**"Post-hoc cohort discovery."** Ship the router, then slice the online-loop metrics to find the regressed cohort. This is p-hacking on cohorts. Cohorts are declared *before* the report is generated.

**"Router-and-model conflation."** Report a single quality-gap number and let the reader guess whether to fix the router or the model. The router-error separation is the tool that keeps the fix targeted.

**"Classifier as an accuracy score."** Score the classifier's own accuracy in isolation (against a labelled easy / hard set) and report a percentage. The report shape here says: the classifier's quality is measured by the *quality of the downstream routed request*, not by its own confusion matrix. A classifier that is 95 % accurate but mis-classifies exactly the enterprise cohort is worse than an 85 %-accurate classifier that mis-classifies uniformly.

**"Silent fallback."** The fallback tier catches errors invisibly — the report shows only the classifier's chosen tier. The report shape here requires the fallback rate as a first-class row; a rising fallback rate is the earliest signal a classifier is drifting.

**"Routing tuned on the eval set."** The classifier's threshold is chosen to maximise the eval-set metric. The threshold overfits; production quality regresses because the eval set is not the production distribution. Fix: threshold is tuned on a hold-out set that overlaps with production traffic distribution, not on the pre-registered eval set. Mod-107's online-loop scored rows are the source.

### Where this chapter plugs into the trade-off report

- **The `quality` column** on the multi-objective report gets a routing-specific sub-column: `router-attributable-gap` and `model-attributable-gap`.
- **The `cohort matrix`** gains the routing report's cohort-preservation pass/fail row against δ_c.
- **The `pre-registered decision rule`** adds the cohort-preservation clause explicitly as a rule.
- **The `rollback trigger`** references the mod-107 online loop's per-cohort quality and the fallback rate — a fallback-rate spike is often the first signal.

## Summary

- Routing patterns compose — **cache-first → tier-based → classifier-gated → always-frontier fallback**. The report attributes each layer's routing decision separately via a `routing_decision` span attribute.
- The **cohort-preservation contract** is the routing eval's central discipline: for every named cohort `c`, `quality(routed) ≥ quality(baseline) − δ_c`, with `δ_c` pre-registered per cohort (tighter for critical cohorts, looser for best-effort).
- The **baseline is frontier-only** (or best-of-tier-per-cohort if frontier has known cohort weaknesses), not "the previous production system" — comparing against a previous routed system drifts silently.
- The **router-error separation** splits the total quality gap into `router-attributable-gap` (mis-classification) and `model-attributable-gap` (tier is under-powered on cases correctly routed to it). The fix depends on which is larger.
- The **always-frontier-when-unsure fallback** is a safety valve; the report tracks fallback rate per cohort as a first-class metric. The classifier threshold is tuned so the fallback rate on critical cohorts is high (typically ≥ 20 – 30 %).
- The **routing report** is the chapter-02 multi-objective shape plus two additions — the routing-layer traffic distribution and the router-vs-model attribution. Product signs the same way.
- **Drift** is continuous — traffic, model snapshots, classifier calibration all decay. The routing report re-runs at least quarterly, stamped with mod-110 lineage keys.
- **Anti-patterns**: average preservation, post-hoc cohort discovery, router-and-model conflation, classifier-as-accuracy-score, silent fallback, routing tuned on the eval set.

Chapter 04 opens the sibling discipline — replacement regression when the whole model is being swapped, not routed to.
