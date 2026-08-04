# exercise-05: A/B Platform Interface and CUPED Contract

**Estimated effort:** 2 hours

## Objective

File a full **pre-registration bundle** for a real (or convincingly simulated) A/B experiment on top of an A/B platform of your choice — Statsig, Eppo, Optimizely, LaunchDarkly Experimentation, Split, GrowthBook, or an in-house platform. The bundle contains the metric definition, the CUPED covariate, the SRM and cohort-sanity checks, and the readout-notes template. By the end you will have shipped the interface artefacts chapter 06 defines, and you will know where the peer (`model-evaluation-engineer` / `senior-ml-engineer`) takes over.

This is a shorter exercise because the depth is deliberately delegated. The point is to *understand and file* the contract, not to re-derive the experimental methodology.

## Prerequisites

- Chapter 06 of this module.
- Exercise-01's scored-row store with ≥ 14 days of pre-experiment data on the primary rubric (for the CUPED covariate computation).
- An A/B platform account (a free tier of Statsig or GrowthBook is enough) or a documented in-house platform your team already runs.
- Familiarity with your platform's pre-registration surface. Read your platform's own documentation on pre-registered experiments before starting; the depth varies.

## Set-up

1. Pick or scaffold an experiment. If your team already runs one, use it. If not, scaffold a small proxy: `Experiment: change support-bot system prompt to add "cite the retrieved documents"; hypothesis is that faithfulness improves by ≥ 0.02 with no cost regression > $0.005 / query.`
2. Create `eval/experiments/<exp-id>/` in your repo:

   ```
   eval/experiments/exp-001-support-bot-citation-prompt/
   ├── metric-definition.yaml
   ├── cuped-covariate.yaml
   ├── preregistration.md
   ├── sanity-checks.py
   ├── readout-notes-template.md
   └── README.md
   ```

3. Confirm access to the platform's SRM alarm (native for Statsig / Eppo / Optimizely; user-implemented for LaunchDarkly / Split / in-house).

## Requirements

Produce a PR against your working branch that adds:

1. **`metric-definition.yaml`** — the primary metric and any guardrails, in the schema your A/B platform accepts. Include:
   - `metric_id` — a stable name.
   - `rubric_id` and `rubric_hash` — from chapter 02's schema.
   - `aggregation` — mean, p05, rate ≥ threshold.
   - `unit_of_assignment` — session / user / tenant.
   - `direction` — higher-is-better or lower-is-better.
   - `guardrail_tolerance` — for guardrail metrics only, the max regression allowed.
   - `runbook_url` — the mod-106 runbook for the metric.
   - `slices` — pre-declared sub-population slices (locale, tenant tier, product surface).
2. **`cuped-covariate.yaml`** — the CUPED covariate declaration. Include:
   - `covariate_id` — a stable name.
   - `source` — a SQL or DSL expression against the exercise-01 scored-row store computing the per-unit-of-assignment covariate over a pre-experiment window.
   - `pre_experiment_window` — e.g., "the 14 days ending 24h before enrolment starts".
   - `expected_correlation` — the observed `ρ` between the covariate and the primary metric on historical data (compute against the last N weeks of the store; report the number).
   - `expected_variance_reduction_pct` — `100 × (1 - (1 - ρ²))` = `100 × ρ²`.
3. **`preregistration.md`** — a human-readable pre-registration document that mirrors what you file on the platform. Sections:
   - Hypothesis (one sentence).
   - Primary metric (link to `metric-definition.yaml`).
   - Guardrail metrics.
   - Cohort assignment (unit + mechanism).
   - Sample size or stopping rule (fixed-`n` with MDE + α + power, or sequential mSPRT with `α` and mixing prior).
   - Decision rule (ship / iterate / kill on what outcomes).
   - Owner (a named human on the eval team + a named peer on the model-eval-engineer side who reviews the methodology).
   - Filed at (timestamp; must be before enrolment starts).
   - Out-of-scope questions (what will not be answered by this experiment).
4. **`sanity-checks.py`** — two functions:
   - `sample_ratio_mismatch(exp_id) -> tuple[bool, dict]` — pulls arm assignment counts from the platform (via API or exported CSV) and runs a chi-squared test against the intended split. Returns `(passed, evidence)`. Runs daily during ramp-up and on readout.
   - `cohort_key_sanity(exp_id) -> tuple[bool, dict]` — reads the exercise-01 scored-row store for rows with the experiment's `ab_experiment_arm` tag; confirms the `flag_variant` distribution matches the arm assignment; confirms cohort keys are the expected values for each arm.
5. **`readout-notes-template.md`** — the template the eval team will fill out post-experiment: what moved, what was in scope, what was out of scope, cohort deltas, follow-up questions for next cycle. Explicitly forbids post-hoc metric-shopping.
6. **A filed pre-registration on the platform.** Screenshot or link the platform's confirmation that the pre-registration is filed and locked. If the platform does not have first-class pre-registration, save the platform's config plus the `preregistration.md` in a git commit with a timestamp before enrolment starts (a git commit is a valid pre-registration mechanism).
7. **A dry-run of the sanity checks against real assignment data.** Enrol at least 100 units per arm (simulated is fine); run `sample_ratio_mismatch` and `cohort_key_sanity`; confirm both pass. If either fails, fix the assignment mechanism before continuing.
8. **A short essay in `README.md`** — the delegation boundary in your own words: which questions this exercise answers, which questions belong to the peer (`model-evaluation-engineer` / `senior-ml-engineer`), and where the ship / iterate / kill decision is owned.

## Starter guidance

- **Do the pre-registration before enrolment.** Filing it after the fact defeats the purpose. If you cannot do it before, use a git commit timestamp as your pre-registration record — it is a defensible mechanism.
- **The CUPED covariate is almost always the same rubric on the same unit in a pre-window.** Do not overthink it. Compute `ρ` on your historical store; if `ρ > 0.3` the covariate is worth it; if `ρ < 0.1` it is not.
- **Compute expected variance reduction and cite it in the pre-reg.** A pre-reg that says "we might use CUPED" is not a pre-reg; a pre-reg that says "we will use CUPED with covariate X, expected `ρ = 0.55`, expected variance reduction 30 %" is.
- **Guardrail metrics have direction.** A cost guardrail is `<= tolerance` on the delta; a latency guardrail is `<= tolerance ms`; a safety guardrail is `>= floor`. State the direction; do not leave it to interpretation.
- **Delegation matters.** The exercise ends where the peer's methodology begins. Do not derive MDE / power / novelty-effect correction; name that the peer owns them and — if there is no peer on your team — name the platform's built-in as your fallback.
- **Sanity checks are non-negotiable.** SRM and cohort-key sanity are the two things that render every downstream number wrong. Run them during ramp-up; do not treat them as post-hoc.
- **Readout-notes template is what prevents metric-shopping.** Post-hoc "but the p05 on enterprise cohort" is exactly the pattern the pre-registration exists to defeat. Follow-up questions belong in the *next* pre-registration.

## Acceptance criteria

You are done when:

- The five artefact files (`metric-definition.yaml`, `cuped-covariate.yaml`, `preregistration.md`, `sanity-checks.py`, `readout-notes-template.md`) exist and are code-reviewed.
- The pre-registration is filed on the platform (or committed to git) before enrolment begins; timestamp evidence is captured in the PR.
- CUPED covariate `expected_correlation` and `expected_variance_reduction_pct` are computed against real historical data from the exercise-01 store; the number is defensible.
- Both `sample_ratio_mismatch` and `cohort_key_sanity` run against real (or simulated ≥ 100-per-arm) assignment data and pass.
- `README.md` names the peer explicitly and the point at which the peer takes over. If no peer exists on your team, `README.md` names the platform's built-in methodology as the fallback and identifies the escalation path to add the peer later.
- The exercise does *not* re-derive experimental design; the README confirms this deliberately.

## Stretch goals

- **CUPED across multiple metrics.** Declare CUPED covariates for the primary metric and each guardrail; report the per-metric `ρ` and the total sample-size reduction.
- **Sequential-mode experiment.** File the pre-reg with `stopping_rule: mSPRT (α=0.05, mixing prior N(0, 0.02²))` instead of fixed-`n`; run against the platform's sequential mode; document the sample-size saved on a real effect.
- **Cross-experiment interaction.** If your platform runs multiple simultaneous experiments, declare the potential interaction with another live experiment in the pre-reg; consult the peer on whether a holdout design is needed.
- **Peer-review the pre-reg.** Route the pre-registration through your peer or team's `model-evaluation-engineer` (or, absent one, through a senior teammate who does not work on this surface). Capture the review comments; incorporate them; commit the version-2 pre-reg.
- **Post-experiment readout walk-through.** Once the experiment lands, fill out the readout-notes template. Present the readout to a proxy PM; confirm every question the PM asks maps back to something declared in-scope in the pre-reg (or is explicitly deferred).
- **CUPED as a scheduled recompute.** Wire the CUPED covariate as a nightly job that publishes to the platform's metric surface as its own metric; the platform reads it and applies the adjustment automatically for every new experiment on this surface.

## What this exercise does *not* cover

You are not deriving MDE / power / novelty-effect correction / network-interference correction — those are peer territory. You are not deciding ship / iterate / kill — that is product-plus-peer territory. You are shipping the pre-registration and CUPED contract that lets the eval program's AI-metrics be first-class inputs to the A/B platform.
