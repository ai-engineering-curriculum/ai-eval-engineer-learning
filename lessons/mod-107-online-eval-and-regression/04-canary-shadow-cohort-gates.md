# Canary and Shadow Launches with Per-Cohort Quality and Safety Gates

## Motivation

Mod-106 chapter 06 defined the deploy gate — offline pass on the artefact, canary against a slice, progressive rollout — and named the interface to the platform team's rollout system. That chapter deliberately did not build the *evaluation* side of the canary; it named the online judge as the substrate and pointed here.

This chapter builds it. Concretely: how the sampled score stream from chapter 02 becomes a **per-cohort gate** that the deploy pipeline consults at canary time and at each rollout stage; when to run **shadow** versus **live-slice**; how to slice a gate per **cohort**; how to sequence a rollout so a failing gate can auto-rollback the smallest surface area possible; and where the **safety-specific** severity rules diverge from the quality rules.

The gate at deploy time and the drift monitor at run time share the same score stream (chapter 02), the same statistical tests (chapter 03), the same continuous-monitoring machinery (chapter 05), and the same threshold file (mod-106 chapter 03). The distinguishing question at deploy time is: *is the new cohort statistically indistinguishable from — or better than — the reference cohort, on the metrics we pre-declared as ship-critical?*

## Core concepts

### Shadow, live-slice, and the two questions each answers

Two evaluation shapes at canary time (introduced briefly in mod-106 chapter 06; expanded here). They answer different questions and both are usually run in sequence.

- **Shadow eval.** Mirror a fraction of production requests through the *candidate* in parallel with production; discard the candidate's reply from the user path; score both replies. Answers: *"if this candidate had shipped, would the score have moved?"* Zero user-visible blast radius; roughly doubles inference cost on the shadowed slice; blind to user-visible latency and downstream tool effects.
- **Live-slice eval.** Route a small fraction of real production traffic *actually* through the candidate; score the served replies. Answers: *"is the shipped version healthy on the shipped cohort?"* Blast radius = the slice; no cost doubling; observes user-visible latency and any downstream cascading effects.

The default sequence: **shadow for one to four hours**, promote iff shadow gate passes, **live-slice at 1 – 5 % for four to twenty-four hours**, promote iff live-slice gate passes, then progressive rollout.

Two counter-patterns are worth naming:

- **Shadow-only rollouts** cannot observe latency-to-user, tool-side-effect impact, cache-warmup effects, or rate-limit changes. Fine for a pure-code prompt tweak; insufficient for a chain re-wiring, a model-snapshot bump, or a tool-schema change.
- **Live-slice without shadow** is faster to deploy but riskier: a surprise regression is user-visible. For non-safety changes on surfaces with a small cohort of tolerant early-tenant customers, this is a defensible trade; for anything safety-touched or high-severity, the shadow stage is the cheap insurance the rollback contract expects.

### The gate script: same interface as mod-106

Mod-106 chapter 06 defined the platform-team interface as a small set of scripts plus a JSON result schema. The online-loop canary gate satisfies the same interface — this is not a new contract, it is the same one, now reading production traces instead of on-disk fixtures.

```
$ eval/deploy-gates/canary.sh \
    --candidate-cohort '{"flag_variant":"canary_v3"}' \
    --reference-cohort '{"flag_variant":"stable"}' \
    --window-seconds 3600 \
    --threshold-file eval/thresholds.yaml \
    --out /tmp/canary-result.json
```

Result shape (JSON, machine-readable):

```json
{
  "verdict": "PASS" | "FAIL" | "INSUFFICIENT_DATA",
  "generated_at": "2026-08-04T15:00:00Z",
  "window": {"start": "...", "end": "...", "seconds": 3600},
  "candidate": {"cohort_keys": {...}, "n": 812, "n_effective_weighted": 720},
  "reference": {"cohort_keys": {...}, "n": 21430, "n_effective_weighted": 21200},
  "metrics": {
    "faithfulness": {
      "candidate": {"mean": 0.83, "p05": 0.61},
      "reference": {"mean": 0.86, "p05": 0.68},
      "delta_mean": -0.03,
      "confidence_interval_95": [-0.05, -0.01],
      "verdict": "FAIL",
      "reason": "delta_mean below floor -0.02"
    },
    "safety_refusal_correctness": {"...": "..."},
    "latency_e2e_ms_p95": {"...": "..."},
    "cost_per_query": {"...": "..."}
  },
  "failures": ["faithfulness"],
  "runbook_urls": ["https://.../runbooks/faithfulness.md"]
}
```

The platform team's rollout system consults `verdict`; every other field surfaces on the deploy-status page. This shape matches the deploy-gate contract from mod-106 chapter 06 — the same schema is what the offline-artefact gate emits.

### Cohort-key discipline: the gate is only as good as the slice

The gate slices the scored rows by cohort key. Chapter 02's scored-row schema carries the keys; chapter 03 already stressed cohort attribution. Two rules specific to the gate.

- **The candidate and reference cohort keys must differ on exactly one axis** — the one the deploy is changing. If the canary is `flag_variant=canary_v3` vs `flag_variant=stable`, other keys (region, tenant, product surface) must be *drawn from the same distribution* in both slices. Otherwise the gate is measuring the cohort shift, not the change. If cohorts are not comparable, the gate returns `INSUFFICIENT_DATA` with a `reason` and does not promote.
- **The gate runs *per pre-declared cohort*, not only in aggregate.** A candidate whose aggregate faithfulness is unchanged but whose Japanese-locale faithfulness dropped 15 points is not a passing gate. The pre-declared cohorts are locales, tenant tiers, product surfaces, and any experiment-arm assignments that pre-date this change. Any one cohort failure fails the gate; the runbook decides whether to widen, roll back, or accept-with-a-scoped-suppression.

### Per-cohort thresholds: absolute floor and delta

The threshold file (mod-106 chapter 03) carries per-metric absolute floors and per-metric deltas. At the canary gate, add a third clause: **per-metric per-cohort delta**. Example:

```yaml
metrics:
  faithfulness:
    absolute_floor: 0.80
    delta_vs_reference: -0.02
    per_cohort_delta_vs_reference: -0.05
    severity: block
    owner: eval-team
    runbook: runbooks/faithfulness.md
  safety_refusal_correctness:
    absolute_floor: 0.98
    delta_vs_reference: -0.01
    per_cohort_delta_vs_reference: -0.02
    severity: block
    owner: safety-team
    runbook: runbooks/safety_refusal.md
    cool_down_seconds: 0
```

Two moves in the per-cohort delta:

- The per-cohort tolerance is *looser* than the aggregate delta, because per-cohort sample sizes are smaller and the CI is wider. Setting the same tolerance per cohort as in aggregate is the recipe for a gate that fails on noise every time it evaluates a small cohort.
- The floor is *not* relaxed per cohort. A cohort whose absolute score falls below the floor is failing regardless of aggregate. Safety metrics keep the strictest per-cohort clause; do not degrade the safety floor to reduce false positives — the false-positive cost is a rolled-back deploy, the false-negative cost is a shipped safety regression.

### Statistical rigour at the canary gate

The canary window is small — hundreds to low thousands of scored traces. The gate is repeatedly re-evaluating the same score stream as the window slides (chapter 05 is the general treatment of this problem). Two mistakes to avoid at this scale:

- **Do not use a fixed-`n` t-test that is re-evaluated every minute.** The false-positive rate compounds; the gate flaps. Use a **confidence sequence** (chapter 05) instead — anytime-valid; the interval only narrows as more data arrives, and the decision "candidate is worse than reference by more than the delta" is stable once made.
- **Do compute effective sample size for weighted samples.** Chapter 02's sampler produces `sample_weight` when biased sampling is used; `n_effective = (Σ w)² / Σ w²` is what the CI is derived from, not raw row count. Skipping this over-shrinks the CI and the gate under-fires on regressions.

The gate reports the confidence interval on the delta, not just a point estimate. A delta of `-0.03` with CI `[-0.05, -0.01]` is a clear regression; a delta of `-0.03` with CI `[-0.09, +0.03]` is `INSUFFICIENT_DATA`. Treat the interval as first-class output.

### The safety carve-out: stricter gates, no soft-fail

Safety metrics — jailbreak resistance, PII-leakage rate, harmful-content rate, refusal-correctness — obey the same gate mechanism with three carve-outs.

- **No `warn`-only severity.** All safety metrics are `severity: block`. There is no soft-fail state for a safety regression.
- **Shorter cool-downs; no auto-suppression.** A safety alert re-fires on every window until the cohort clears the floor. There is no `snooze` button.
- **Zero-tolerance floors on some metrics.** Rate of returning content the guardrail marked "block" must be exactly zero in the canary cohort. Any nonzero rate is an immediate `FAIL` regardless of sample size.

Mod-108 owns the safety rubrics themselves and their calibration; this chapter's gate machinery is what enforces them at deploy time.

### Progressive rollout: gate at every stage, hold for signal

Past canary, the deploy pipeline advances in stages — 5 % → 25 % → 50 % → 100 %, or per-cell / per-region. At each stage the gate script re-runs, now with the newly-included cohort as the candidate.

Two rules make the per-stage gate operational:

- **Hold each stage long enough to collect the metric's effective sample size.** The absolute-floor test on a `p05` needs enough tail samples to be estimable — a five-minute hold on a low-traffic stage is almost always insufficient. The runbook per metric names the minimum sample size; the platform team's rollout system reads it. Chapter 05's confidence sequence answers "have we seen enough" without a hard-coded per-metric `n`.
- **Do not skip stages on prior success.** A candidate that passed the 5 % stage is not automatically going to pass the 25 % stage: a wider cohort activates new tenants, new locales, new tool-call paths that the small stage never exercised. The gate re-runs at each stage against the newly-included cohort — not the cumulative served cohort — for the same reason.

### Rollback shapes at the online loop

Mod-106 chapter 06 named three rollback shapes: code-only, model-version, retrieval-index. The online loop adds a fourth.

- **Feature-flag flip.** When the canary is behind a flag, flipping the flag off is the rollback. Sub-second effect; observable on the flag-provider's dashboard; no re-deploy needed. The runbook per metric names whether the metric's rollback is a flag flip or a redeploy; the on-call reads that answer at 3 a.m., they do not derive it.

The rollback contract is *rehearsed*. A gate that fires on Tuesday for the first time and the on-call cannot execute the rollback in five minutes is a broken contract regardless of whether the eval side was correct. Mod-106 chapter 06 named the rehearsal cadence; this chapter reinforces it — an online-loop alert that no one can act on is a dashboard, not a gate.

### Auto-rollback versus auto-halt-and-page

The platform team's rollout system supports two automated responses on a failing gate:

- **Auto-rollback.** Gate fires → pipeline halts the rollout and reverses the change (flag off, revert deploy, restore index pointer). No human in the loop.
- **Auto-halt and page.** Gate fires → pipeline halts at the current stage and pages the on-call. The on-call reads the runbook and either rolls back or resumes with a documented rationale.

Choose per surface. Auto-rollback is safer for narrow, well-shaped changes on cohorts where a flap is inexpensive (a two-minute mistake is fine). Auto-halt-and-page is the default when the surface is broad, the change is intentional, and a false-positive rollback would itself be user-visible (flapping the model snapshot every ten minutes is worse than one minute of a real regression).

The choice per metric goes into the thresholds file — mod-106 chapter 03's `severity` field extends to `severity: block-and-auto-rollback` vs `severity: block-and-page`.

### Interfacing with mod-106's deploy-gate script

The `eval/deploy-gates/` directory the mod-106 chapter 06 exercise built already contains `offline.sh`. This chapter adds:

- `eval/deploy-gates/shadow.sh` — reads the shadow-scored rows for the candidate; returns the same JSON schema.
- `eval/deploy-gates/canary.sh` — reads the live-slice-scored rows for the candidate; returns the same JSON schema.
- `eval/deploy-gates/rollout-stage.sh` — parameterised by stage cohort; returns the same JSON schema.

The platform team's rollout controller (Argo Rollouts `AnalysisTemplate`, Flagger `MetricTemplate`, Spinnaker canary analysis, LaunchDarkly experiment) consumes the JSON `verdict` and progresses or halts accordingly. The eval program does not own the controller; the online loop hands the controller a decision.

### What the online-loop gate does *not* replace

- **The offline gate on the artefact** stays. Chapter 02 of this module cannot detect a build-time dependency drift; the artefact gate does.
- **The A/B experiment on the rolled-out change** is different from the canary gate — the A/B is measuring product effect (feature adoption, retention, downstream conversion) with an experimental design; the canary gate is measuring eval-metric floor compliance. Chapter 06 walks the interface.
- **The safety-specific canary rules** are mod-108. This chapter's gate machinery is generic; mod-108 supplies the safety rubrics, the harmed-cohort definitions, and the zero-tolerance floors.

## Summary

- **Shadow eval** answers "would the score have moved?" (zero blast radius, doubled inference cost). **Live-slice eval** answers "is the served cohort healthy?" (real blast radius, real latency and side-effect coverage). Run shadow → live-slice → progressive rollout.
- The **gate script** matches mod-106 chapter 06's interface: `PASS / FAIL / INSUFFICIENT_DATA` JSON with per-metric CIs, cohort attribution, and runbook links.
- **Cohort-key discipline**: candidate and reference must differ on exactly one axis. Per-cohort gates fail-fast on any one cohort; do not aggregate away a locale regression.
- **Per-cohort thresholds** relax the *delta* but not the *floor*. Safety-metric floors are never relaxed per cohort.
- **Statistical rigour at small windows**: use a **confidence sequence** (chapter 05), not a re-evaluated fixed-`n` t-test; compute **effective sample size** for weighted samples; treat the **confidence interval** as first-class output.
- **Safety carve-out**: no soft-fail, no snooze, zero-tolerance floors where mod-108 declares them.
- **Rollout gates hold for signal**; do not skip stages; re-evaluate the newly-included cohort at each stage.
- **Rollback shapes** now include **feature-flag flip** (sub-second, no redeploy). The rollback contract is **rehearsed**.
- **Auto-rollback vs auto-halt-and-page** is a per-metric choice recorded in the thresholds file.

Chapter 05 walks the confidence-sequence machinery this chapter's gate relies on — the anytime-valid inference that lets the gate re-evaluate the score stream without inflating the false-positive rate.
