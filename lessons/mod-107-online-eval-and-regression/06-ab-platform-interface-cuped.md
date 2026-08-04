# The A/B Platform Interface: Pre-Registration, CUPED, and the Delegation Boundary

## Motivation

The canary gate (chapter 04) blocks a bad rollout. The drift monitor (chapter 03) catches a distribution shift after the change is live. Neither answers the question the product team actually asks after a change ships: **"did the change move the number we care about, by an amount worth defending, on the users we intended?"**

That question is the domain of the **A/B testing platform** — Statsig, Eppo, Optimizely, LaunchDarkly Experimentation, Split, or an in-house platform. Product organisations at any nontrivial size have one; the eval program *interfaces with it*, does not own it. This chapter defines that interface.

The interface has two shapes:

- **Metric definitions and their contract.** The eval program supplies the rubric-scored metrics (chapter 02's stream) that the A/B platform treats as candidate KPIs. The contract states how the metric is computed, what its acceptable direction is, what cohorts it splits on, and how it composes with the platform's existing metric surface.
- **Variance-reduction covariates.** The eval program can supply a **CUPED covariate** — a pre-experiment scalar derived from the same rubric — that the A/B platform uses to shrink variance in the treatment-effect estimate, and therefore shrink required sample size. This is a small, load-bearing contribution.

What the eval program does *not* own: fixed-horizon experiment design, minimum-detectable-effect derivation, novelty-effect / carry-over correction, network-interference correction, spillover analysis, cross-metric decision-making, or the guardrail-metric framework at the platform level. All of those belong to `model-evaluation-engineer` (peer, ML Engineering family, level 30) and `senior-ml-engineer` on the product's own team. This chapter walks the interface and names the boundary.

## Core concepts

### The eval program's contribution to the A/B stack

A well-shaped A/B stack has four layers.

```
[experiment design]        ← model-evaluation-engineer / senior-ml-engineer
[experiment platform]      ← platform / data-platform team
[metric definitions]       ← eval program (this module) + product analytics
[variance reduction]       ← eval program (this module) + platform analytics
```

The eval program (that is you) owns the bottom two rows for the AI-specific metrics: the rubric-scored quality metrics, the safety metrics, the cost / latency SLO metrics. The top two rows are peer-owned. Interfacing means:

- Publishing metrics that the platform can consume as first-class KPIs — same aggregation, same denominator, same slicing keys as the platform's other metrics.
- Publishing pre-experiment CUPED covariates that shrink the variance of those metrics inside the platform's engine.
- Filing the pre-registration document (below) *before* an experiment starts.
- Reading the platform's experiment result and updating the runbook when a shipped variant becomes the new baseline.

Not owning the top two layers is *the correct posture*. Fully-owned in-house A/B design at this scale is a career for a specialist; the eval program's leverage comes from bringing the AI-metric contribution into an existing platform, not from replacing the platform.

### The pre-registration contract

A pre-registered experiment specifies, before the experiment begins:

- **Primary metric.** One name; one definition; one direction. Ambiguity here is the single largest source of post-hoc metric-shopping in AI experiments.
- **Guardrail metrics.** Metrics that must not regress by more than a stated tolerance regardless of the primary outcome. Safety, latency p95, and cost are the standard AI-side guardrails.
- **Cohort assignment.** The randomisation unit (session / user / tenant) and the assignment mechanism.
- **Sample size or stopping rule.** Either a fixed-`n` power calculation (with MDE, power, α) or a sequential-testing configuration (mSPRT / CS with per-look α).
- **Sub-population slices to report.** Locale, tenant tier, device, product surface — the slices the eval program has cohort keys for on the scored row.
- **Decision rule.** What outcome ships / kills / iterates. Pre-declared, not decided on the review call.
- **Owner.** Named human, not team.
- **Filed at.** Timestamp; before the experiment enrolment starts, not the day the readout lands.

The pre-registration lives in the A/B platform (Statsig, Eppo, and Optimizely all support pre-registered experiments as a first-class object). The eval program's contribution is the **metric-definition side** of the doc — the exact rubric hash, the aggregation function, the tolerance direction, and the runbook link.

A pre-registration is *not* a formality; it is the mechanism that prevents metric-shopping after the readout. When the primary metric moves the wrong direction and someone asks "but what about the p05 on the enterprise cohort," the pre-registered doc answers: *not in scope*. The runbook can add it as a next-cycle question.

### The metric definition on the eval side

The A/B platform consumes metrics as one-row-per-unit-of-assignment. The eval program's scored rows (chapter 02) are one-row-per-scored-trace. Two shapes of aggregation are worth naming.

- **Per-user (or per-session, per-tenant) mean rubric score.** For each unit of assignment, compute the mean of the rubric scores across the traces this unit generated in the experiment window. The A/B platform runs a two-sample test on the resulting per-unit means. This is the canonical shape and matches most existing platform tooling.
- **Trace-level rubric score with unit clustering.** Feed the raw trace-level scored rows to the platform; declare the unit-of-assignment cluster; use a cluster-robust variance estimator. Statistically preferable when per-unit sample sizes vary widely; requires the platform to support cluster-robust estimation.

Either is valid; pick per platform capability. Whichever you pick, the metric definition document names:

- The `rubric_id` and `rubric_hash` (chapter 02's schema).
- The aggregation function (`mean`, `p05`, `rate ≥ 0.8`).
- The unit-of-assignment / clustering rule.
- The window (typically the experiment window, but sometimes a lagging window if the metric needs multi-day observation).

### CUPED: variance reduction from pre-experiment data

**Controlled-experiment Using Pre-Experiment Data** (Deng, Xu, Kohavi, Walker; 2013) is the standard variance-reduction technique in industry A/B. The intuition: if you have a covariate that is correlated with the metric of interest and that is measured *before* the experiment starts, you can adjust the metric for that covariate and shrink its variance.

Mechanically, if `Y` is your metric and `X` is the pre-experiment covariate:

- Compute `θ = Cov(Y, X) / Var(X)` on pre-experiment data.
- Define the CUPED-adjusted metric `Y_cuped = Y - θ · (X - E[X])`.
- Run the A/B test on `Y_cuped` instead of `Y`.

The unbiasedness of the treatment-effect estimator is preserved (`X` is pre-treatment and independent of assignment); the variance shrinks by a factor of `(1 - ρ²)` where `ρ = corr(Y, X)`. A covariate with `ρ = 0.6` cuts required sample size by 64 %; `ρ = 0.8` cuts by 82 %.

The eval program's role in CUPED: **supply the covariate**.

The best CUPED covariate for an AI experiment is usually the *same rubric score, measured on the same unit-of-assignment in a pre-experiment window*. For example:

- Experiment metric: mean faithfulness per user over the experiment window.
- CUPED covariate: mean faithfulness per user over the 14 days *before* the experiment.
- `ρ` is typically 0.4 – 0.7 on rubric scores; expected variance reduction 15 – 50 %.

The eval program computes the covariate from the scored-row store (chapter 02) and publishes it to the A/B platform as a first-class metric alongside the experiment metric. The platform's CUPED implementation handles the adjustment; the eval program does not implement the estimator.

### Sanity checks the eval program owes the platform

Two checks the eval side should run on every experiment before it consumes the platform's readout.

- **Sample-ratio mismatch (SRM).** The observed traffic split (e.g., 50/50) should match the intended split within a chi-squared test at reasonable significance. An SRM failure means the assignment mechanism is broken — traffic that was supposed to be 50/50 came in at 47/53 — and *no downstream metric from the experiment is trustworthy*. Platforms flag this automatically; if yours does not, add a check. Eppo, Statsig, and Optimizely all ship an SRM alarm.
- **Cohort-key sanity.** The scored rows for the experiment cohorts have the expected `flag_variant` values and the expected `ab_experiment_arm` distribution. A silent flag misassignment (a stale cache, a bug in the assignment SDK, a partial rollout of the assignment library) is a common failure mode; the eval program's cohort-key check catches it.

Both checks belong in the pre-experiment ramp-up ("dry-run") window, before the experiment is treated as decision-supporting.

### The delegation boundary: what belongs to the peer

Five topics that are *not* eval-program depth in this track. Each is model-evaluation-engineer or senior-ml-engineer territory; the interface here is "know the vocabulary and know where the peer takes over."

- **Sample-size / MDE / power calculation for the experiment.** The eval program supplies the metric definition and its historical variance; the peer derives the sample size. If the peer is not available, the platform's built-in power calculator is a reasonable stand-in.
- **Novelty-effect and primacy-effect correction.** A new feature's first-week metric is often systematically different from its steady-state. Correcting for this is peer-territory; the eval program flags "the metric moved but the first-week samples dominate" as a note on the readout.
- **Network interference / spillover.** When users interact (shared knowledge base, shared tools, shared cache), assignment-level randomisation can leak. Cluster-randomised designs and interference-robust estimators are peer-territory.
- **Multi-experiment interaction / holdout design.** When multiple experiments run simultaneously on overlapping cohorts, the interaction between them requires joint analysis. Peer-territory.
- **Ship / iterate / kill decision on a mixed result.** The primary moved, one guardrail flapped, another moved the wrong way. The eval program does not decide; the eval program supplies the readout and the runbook. Product-plus-peer decides.

The peer's job is to defend the *methodology* of the readout. The eval program's job is to defend the *metric* the readout is computed on. Both defences are load-bearing.

### The interaction with sequential testing

Chapter 05 named a distinction that matters here: the online-loop monitor uses **anytime-valid inference** (confidence sequences, mSPRT) because it looks at the stream repeatedly with no fixed horizon; the A/B experiment uses **pre-registered stopping** (fixed-`n` or alpha-spending) because it has a horizon and a pre-registered look schedule.

Modern platforms (Statsig, Eppo, Optimizely) offer **sequential mode** as an option on top of pre-registration — an mSPRT-based stopping rule that lets the experiment end early *and* stays pre-registered. When the platform supports it, prefer sequential mode: the experiment can conclude in half the time on a real effect. The pre-registration contract still applies; the stopping rule is documented as "sequential mSPRT with α = 0.05, effect-size prior N(0, σ²)".

### Cross-referencing the mod-106 gate

The A/B experiment does not replace the mod-106 offline gate or this module's canary gate. The three layer:

- **Offline gate (mod-106 ch. 06)** — replay-set floor compliance on the built artefact. Blocks canary promotion.
- **Canary / rollout gate (chapter 04)** — pre-registered floor compliance on the live-served candidate cohort. Blocks progressive rollout advancement.
- **A/B experiment (this chapter's interface)** — pre-registered treatment-effect estimation, on a longer horizon, using the platform's engine. Decides *ship / iterate / kill* on the product-effect question.

A change ships through all three; each answers a different question; each has its own owner. The eval program participates in all three.

### The eval-side deliverable per experiment

For each experiment the eval program supports, produce a small artefact set (versioned in the repo):

- `experiments/<exp-id>/metric-definition.yaml` — rubric id, rubric hash, aggregation, unit-of-assignment, cohort slices, direction, runbook link.
- `experiments/<exp-id>/cuped-covariate.yaml` — covariate definition, pre-experiment window, expected `ρ` from historical data.
- `experiments/<exp-id>/preregistration.md` — the mirror of the platform's pre-registration for the eval-metric side, code-reviewed by the eval team and by the peer.
- `experiments/<exp-id>/readout-notes.md` — post-experiment: what moved, what was in scope, what was out of scope, what the next-cycle questions are.

These artefacts live next to the mod-106 `thresholds.yaml` and `runbooks/`. The peer reviews the pre-registration; the eval team owns the metric-definition and the covariate; the product team owns the outcome.

## Summary

- The A/B platform is **peer-owned**. The eval program owns two rows of the stack: **metric definitions** and **variance-reduction covariates**.
- **Pre-registration contract** names primary metric, guardrails, cohort assignment, sample size or stopping rule, slices, decision rule, owner, and filing timestamp. Filed *before* enrolment, not before readout.
- The metric-definition side lives on the eval program: `rubric_id`, `rubric_hash`, aggregation, unit-of-assignment, direction, runbook link.
- **CUPED** shrinks variance by `(1 - ρ²)` using a pre-experiment covariate. The eval program supplies the covariate — usually the same rubric on the same unit in a pre-experiment window — and the platform applies the adjustment.
- **Sanity checks** the eval program owes: **sample-ratio-mismatch** (via the platform's built-in) and **cohort-key sanity** (verify `flag_variant` distributions).
- **Delegation boundary**: sample-size / MDE / power, novelty-effect, network interference, multi-experiment interaction, and ship / iterate / kill decision are `model-evaluation-engineer` / `senior-ml-engineer` territory. The eval program supports; the peer decides.
- Prefer the platform's **sequential mode** when supported — pre-registered *and* early-stoppable. Still requires the pre-registration document.
- The **three layers** (offline gate, canary gate, A/B experiment) answer different questions and do not replace each other.

Chapter 07 covers the last piece of the loop — dashboards that a product manager, an on-call engineer, and a safety reviewer can each read at a glance.
