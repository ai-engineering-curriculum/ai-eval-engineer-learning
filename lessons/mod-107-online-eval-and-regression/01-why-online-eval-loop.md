# Why the Online-Eval Loop: The Second Half of the Gate

## Motivation

Mod-106 closed with a change fully deployed, an online-eval loop watching, and a rollback contract still live. That final sentence is the entire point of this module — the eval program is not just at merge time, it is at every point the change is in production. What the PR gate and the deploy gate do not do — and cannot do — is watch the change *keep working* against traffic that never appeared in the replay set.

Three classes of regression are invisible to the offline gate on the artefact:

- **Regressions that only surface under distribution shift.** A support-bot prompt that scores fine on the 150-fixture replay set silently degrades when a new self-serve product surface starts routing "how do I export data" queries the fixture set never saw. Nothing about the artefact changed; the input distribution did.
- **Regressions that emerge from vendor drift.** The API model behind the alias (`gpt-4o`, `claude-3-5-sonnet-latest`) is silently rolled to a new snapshot. Rubric scores move; nobody deployed anything. The PR gate is a snapshot; the online loop is a movie.
- **Regressions that only appear under production load.** A change that passes the deploy gate on a 5 %-slice canary fails at 100 % rollout because rate-limit backoff, cross-tenant retrieval-index contention, or a cache-miss cliff only surface at scale. The mod-106 canary caught the shape; only the online loop catches the emergence.

This module is the mechanism for those three problems. The **online-eval loop** samples live traces, scores them on the same rubrics the offline gate used, tracks the distribution of the scores over time, and fires an alert (and potentially a rollback) when the distribution moves in a direction the runbook has pre-declared as bad. It is a closed loop: the alert either resolves inside the app (auto-rollback, feature-flag flip) or lands on the same on-call runbook the deploy gate points at.

This chapter fixes the vocabulary the rest of the module reuses — **online-eval loop**, **sampled-trace scoring**, **judge-tier routing at runtime**, **drift**, **canary vs shadow at deploy vs online**, **confidence sequence**, **cohort gate**, **CUPED**, **pre-registration contract** — and states the shape of the loop the chapters build. Chapters 02 – 07 each build one substrate of that loop.

## Core concepts

### The offline gate and the online loop: complementary, not redundant

The offline gate (mod-106) and the online loop (this module) share the same rubrics, the same thresholds file, the same runbook shape, and the same rollback contract. They differ in three axes:

| Axis | Offline gate (mod-106) | Online loop (this module) |
|---|---|---|
| **What it scores** | A pre-selected replay set of fixtures | A sample of live production traces |
| **When it runs** | PR event; deploy event; per-canary stage | Continuously, on a sliding window |
| **What it protects against** | Change-induced regression (someone edited the prompt) | Drift-induced regression (nobody edited anything but the distribution moved) |
| **Ground truth shape** | Fixture reference + rubric | Rubric-only (no reference); occasionally a lagged human review |
| **Alerting shape** | Pass / fail on the change | Fire on statistically-significant departure from a baseline |
| **Blocking authority** | Merge; canary promotion | Auto-rollback; feature-flag flip; page the on-call |

A team that runs only the offline gate has no visibility into vendor drift or input-distribution shift. A team that runs only the online loop has no visibility into a change-induced regression until it reaches enough traffic to trip a drift alert — by which point users have already seen it. Both matter; this track defends both.

### The online-eval loop, in one picture

```
production traffic
   │
   ▼
[app + trace instrumentation — mod-102]
   │  emits spans on every request
   ▼
[trace backend — Phoenix / Langfuse / Weave / Braintrust]
   │
   ▼
[online-eval loop — this module]
   │
   │  ┌──────────────────────────────────────────────────────────┐
   │  │  sampler (chapter 02): pick a slice of traces to score   │
   │  │      │                                                    │
   │  │      ▼                                                    │
   │  │  judge-tier router (chapter 02): route to OSS / mid /     │
   │  │      │              frontier judge inside a budget        │
   │  │      ▼                                                    │
   │  │  scored-trace store: per-trace {metric, score, cost}      │
   │  │      │                                                    │
   │  │      ▼                                                    │
   │  │  aggregator: sliding-window mean / p05 / rate per cohort  │
   │  │      │                                                    │
   │  │      ├──▶ drift monitor (chapter 03): input / output /    │
   │  │      │       judge-level distribution shift               │
   │  │      │                                                    │
   │  │      ├──▶ cohort gate (chapter 04): canary / shadow /     │
   │  │      │       rollout stages read the aggregator           │
   │  │      │                                                    │
   │  │      ├──▶ confidence sequence (chapter 05): anytime-valid │
   │  │      │       inference on the score stream                │
   │  │      │                                                    │
   │  │      ├──▶ A/B platform (chapter 06): pre-registered       │
   │  │      │       experiments consume metrics as their KPIs    │
   │  │      │                                                    │
   │  │      ▼                                                    │
   │  │  cross-persona dashboards (chapter 07): PM / on-call /    │
   │  │      safety-reviewer views over the same data             │
   │  └──────────────────────────────────────────────────────────┘
   │
   ▼
alert path (runbook — mod-106 chapter 03):
   │ page the on-call → auto-rollback (canary) or human decision (rollout)
   ▼
close the loop
```

Every arrow is a mechanised step. Every metric is versioned. Every alert links a runbook naming an owner. That is the shape this module builds.

### Vocabulary the module reuses

Six terms, each recurring across the chapters. Use these consistently in your repo — vendors call them different things (Braintrust calls the runtime scoring "online scoring", Weave calls it "production monitoring", Langfuse calls it "evaluators") and drift between labels is how teams end up with three dashboards for the same thing.

- **Sampled-trace scoring.** A subset of production traces is picked (chapter 02's sampler) and scored by the online judge. The mean-per-window is the *estimate*; the sample is what makes it affordable. Everything downstream reads sampled data, not the full stream.
- **Drift.** A statistically-significant departure of a distribution from its baseline. Chapter 03 distinguishes **input drift** (query distribution shifted — the users are asking different things), **output drift** (response distribution shifted — the model is answering differently), and **judge drift** (the judge itself is scoring differently). Different causes, different tests, different runbooks.
- **Cohort.** A slice of traffic identified by a stable key: tenant, region, model version, prompt version, feature-flag variant, retrieval-index build, product surface. Chapter 04's canary gate and chapter 06's A/B platform both cut the sampled data by cohort. Cohort keys must be recorded on the span at emit time — you cannot reconstruct them after the fact.
- **Sequential test / confidence sequence.** A statistical procedure that lets you continuously monitor a metric and declare a change *whenever* you have enough evidence, without inflating false-positive rate the way a fixed-`n` t-test does. Chapter 05 walks the two families (mSPRT and confidence sequences); this replaces the "pick a `p` and hope you didn't peek" pattern.
- **Pre-registration contract.** The A/B platform (chapter 06) requires the metric definition, the hypothesis, the sample-size or stopping rule, and the direction of a "good" change to be filed *before* the experiment runs. The eval team is not the A/B methodology owner (`model-evaluation-engineer` is); the eval team supplies the *metric definitions* and *variance-reduction covariates* the platform consumes. The contract is the interface.
- **CUPED.** Controlled-experiment Using Pre-Experiment Data — Microsoft's variance-reduction technique. Uses a covariate derived from the pre-experiment period to reduce the variance of the treatment-effect estimate, shrinking the sample size an experiment needs. Chapter 06 covers the eval-side contract: the eval program supplies the covariate; the A/B platform applies the adjustment.

### What "online" means at three time scales

The loop runs at three scales simultaneously, and the shape of the alert differs at each.

| Scale | Window | Sample size per window | What it catches | Blocks |
|---|---|---|---|---|
| **Canary / rollout stage** | Minutes to hours | Small — 20 – 500 traces | New-change regressions on the shipped slice | Rollout progression |
| **Drift monitor** | Hours to days | Medium — 1 000 – 100 000 traces | Distribution-shift regressions unrelated to a specific deploy | On-call page; conditional auto-rollback |
| **A/B experiment** | Days to weeks | Large — full-experiment sample | Product-decision-shaped effect sizes | Ship / kill / iterate decision |

The same score stream feeds all three. Chapter 02 designs the sampler to be sufficient at the *finest* window (canary) without exhausting the judge budget at the *widest* window (experiment). Chapters 04, 03, and 06 read the same aggregator with different windows and different thresholds.

### Cost and budget: why the loop is *sampled*

Judge inference is the dominant cost of the loop, and production traffic is large. Two arithmetic anchors:

- A surface at 10 requests / second is 864 000 requests / day. A mid-tier judge at $0.005 / call is $4 320 / day on 100 %-scoring. That is *not* a defensible online-eval budget; it is a second production model.
- A 1 % sample of the same traffic is 8 640 requests / day at $43 / day, or ~$1 300 / month — order-of-magnitude in a monthly eval line item, and enough for the drift monitor's medium-window aggregate.

The judge-tier routing pattern from mod-104 keeps the OSS tier at ~$0 / call for the mainline stream and reserves the paid tier for uncertain traces the drift monitor or canary gate flag. Chapter 02 walks the budget math end-to-end and derives sample rates that hit `k` scored traces per canary window without exhausting the monthly cap.

### What this module is not

- **The full A/B experimentation methodology.** Fixed-horizon vs sequential design, minimum-detectable-effect derivation, power analysis at the experiment scale, novelty-effect correction, network-interference correction — all of that is owned by `model-evaluation-engineer` / `senior-ml-engineer`. This module walks the **interface** (chapter 06) and the *variance-reduction covariate* the eval program supplies. Depth in the methodology belongs to the peer track.
- **The trace backend itself.** Mod-102 built the backend. This module *reads* the backend; it does not re-build it.
- **The rubrics and judge tiers.** Mod-104 built those. This module *runs* them at production frequency; it does not re-derive the rubric or re-calibrate the judge.
- **Safety-specific drift monitors.** Guardrail regression, injection-attempt-rate spikes, refusal-rate blowups — same mechanism, tighter thresholds, stricter runbooks. Mod-108 covers them.
- **Human review of alerts.** When a drift alert cannot be resolved by the runbook and the on-call, it lands in the mod-109 human-review queue. This module's runbook link ends at "escalate to human review"; mod-109 defines the queue.
- **The eval data platform underneath.** The scored-trace store, the lineage index, and the cost accounting are all mod-110. This module treats them as substrate.
- **Deploy pipeline mechanics.** Argo Rollouts / Flagger / Spinnaker / LaunchDarkly are owned by the platform team. Chapter 04 defines the interface at the online-loop edge; mod-106 chapter 06 defined the deploy edge; the platform team lives in the middle.

### The end-of-module posture

By the end of this module you own the online half of the eval program. Concretely, you can:

- Justify a sample rate and a monthly judge budget with real arithmetic against a specific surface's request rate.
- Ship an online-eval loop that scores a chosen sample of live traces on quality / safety / cost / latency continuously.
- Detect drift at input, output, and judge level with a defensible statistical test, not "the number looks funny."
- Wire a canary and a shadow-launch gate that reads the same online score stream.
- Monitor continuously with a confidence sequence instead of firing a fixed-`n` t-test every five minutes and burning down your alerting budget on false positives.
- File a pre-registration on the A/B platform that names the eval metric, its direction, its stopping rule, and (if applicable) the CUPED covariate.
- Ship three dashboard views over the same data — one for the PM, one for the on-call, one for the safety reviewer — that each read cleanly at a glance.

The chapters build these one by one. Chapter 02 starts with the sampler and the judge-tier router — the loop's plumbing before anything else is worth talking about.

## Summary

- The **online-eval loop** samples live traces, scores them on the same rubrics as the offline gate, aggregates over sliding windows, and fires alerts on drift or on cohort-gate failure.
- Offline gate and online loop are **complementary**, not redundant: the offline gate catches change-induced regressions; the online loop catches drift-induced ones and regressions that only surface at scale.
- Six terms recur through the module — **sampled-trace scoring**, **drift** (input / output / judge), **cohort**, **sequential test / confidence sequence**, **pre-registration contract**, **CUPED**.
- The loop runs at three time scales — **canary / rollout stage** (minutes), **drift monitor** (hours – days), **A/B experiment** (days – weeks) — over the same underlying score stream.
- **Cost is why the loop is sampled.** Judge inference is the dominant cost; 1 % sample rates plus judge-tier routing keep monthly budgets defensible.
- The **A/B methodology depth** is peer-owned (`model-evaluation-engineer` / `senior-ml-engineer`); this module walks the **interface**, not the theory.
- The module hands off to mod-108 (safety-specific loop), mod-109 (human review of alerts), mod-110 (eval data platform), mod-111 (cost / latency / quality dashboards), and mod-112 (program posture).

Chapter 02 designs the loop's plumbing: sampler, judge-tier router, aggregator, and the budget math that makes all three fit inside a monthly cap.
