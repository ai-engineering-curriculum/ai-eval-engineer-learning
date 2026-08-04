# Deployment Gates: Offline Pass, Canary, Progressive Rollout, and the Platform Interface

## Motivation

The PR gate from chapter 05 catches regressions before merge. That is a large fraction of them, and stopping there is a temptation: the PR check went green, `main` looks healthy, ship.

Two classes of regression the PR gate cannot catch on its own:

- **Regressions that only surface on real traffic.** A prompt tweak that scores fine on the 150-fixture replay set can degrade a specific tenant segment, a specific language, or a specific edge case that is not in the fixture set. The gate is representative, not exhaustive; the deployment is the first place the change meets real distribution.
- **Regressions that surface between merge and deploy.** A dependency upgrade in the release artefact, a vendor snapshot silently rolled between PR-gate time and canary-build time, a configuration drift in the staging cluster. The change that the PR gate scored is not exactly the change that reaches users.

This chapter closes the fifth of the five properties from chapter 01: **wired into deploy**. The eval program's job is to hand the deploy pipeline a *decision* — offline eval passed on the built artefact, canary is healthy on scored production traffic, promotion to progressive rollout is safe — and to keep that decision live at each rollout stage. The deploy pipeline itself is owned by the platform team; the interface is the contract this chapter defines.

## Core concepts

### Three deploy-gate stages

A well-shaped rollout pipeline has three points at which the eval side of the contract asserts, each with a different budget and blast radius.

| Stage | Runs against | Blocks | Time budget | Blast radius |
|---|---|---|---|---|
| **Offline pass on artefact** | Full replay set, against the exact built artefact / image / model version that will ship | Promotion to canary | 15–60 min | Nothing user-visible |
| **Canary (shadow + live-slice eval)** | A slice of real production traffic — mirrored (shadow) and/or served (live) — evaluated by the online judge | Progression past canary | 30 min – 24 h | 1–5% of traffic |
| **Progressive rollout** | Each rollout stage runs the online judge on the newly-included cohort; auto-rollback on regression | Advance to next stage | 1–72 h per stage | Increasing cohorts |

Each stage is a gate. Each gate uses the same eval config from chapter 04, the same threshold file from chapter 03, and the same replay-plus-online-fixture material. The differences are the *runner*, the *budget*, and the *blocking severity*.

### The offline gate on the built artefact

The PR gate ran against the PR branch's checkout. The deploy gate runs against the **built artefact** — the container image, the deploy bundle, the packaged model — that will ship. Two reasons this re-run matters:

- The build can introduce differences (dependency-pin drift, differing tokenizer version, differing linked library). The gate on the artefact confirms the eval-relevant behaviour survived the build.
- The vendor state can drift between PR merge and artefact build. Re-running the gate on the artefact re-observes the current `system_fingerprint`, current judge-model version, current retrieval-index hash. If any of these moved, the report says so.

Implementation shape: the CI job that produces the artefact runs the same eval config against a fresh clone plus the artefact, publishes the same report format as chapter 05, and its exit status gates the "promote to canary" step in the deploy pipeline.

Two things this gate does that the PR gate does not:

- **Runs the full replay set**, including the size-N "expensive" sub-suites that are sample-first-N on the PR (safety, adversarial, long-tail-tenant).
- **Publishes the report as a durable release artefact** (attached to the tag / GitHub Release / vendor UI). Chapter 03's runbook links the release artefact when the on-call is investigating a post-rollout incident.

### The canary: shadow eval vs live-slice eval

A canary is a rollout to a small, representative slice of production traffic. There are two eval-side patterns, and they answer different questions.

- **Shadow eval.** Mirror a fraction of real production requests through the *new* candidate in parallel with the *current* production, do not return the new candidate's reply to the user, and score both replies with the online judge (mod-104's OSS tier, typically). Answers *"if this had shipped, would the eval scores have moved?"* Blast radius: zero user-visible impact. Cost: doubled inference on the shadowed slice.
- **Live-slice eval.** Serve a small fraction of real production traffic *actually* through the new candidate (e.g., 1% of tenants, 5% of sessions), and score the served replies via the online judge. Answers *"is the shipped version's eval healthy on the shipped cohort?"* Blast radius: the slice. Cost: no doubling; the served traffic is what is scored.

Trade-off: shadow eval is safer but doesn't observe user-visible impact (latency to the user, downstream tool effects); live-slice observes those but any regression is user-visible on the slice.

A common pattern: **shadow first for one hour, then live-slice for four hours**, both scored, and both required to pass thresholds before promoting.

### The online judge's role at canary

The rubrics and thresholds are the same as chapters 03 and 04. The runner is different: instead of Promptfoo / DeepEval running against on-disk fixtures, the *online judge* (mod-104 chapter 05) runs against sampled canary traces (mod-102 chapter 06 sampling, mod-107 online-eval loop). Concretely:

- The traces from the canary slice hit the trace backend as they normally would.
- The mod-107 online-eval loop scores a sampled subset with the OSS-tier judge, escalating uncertain cases to the mid-tier.
- A canary-gate script pulls the last N minutes of scored traces, computes the mean and p05 per metric, applies the threshold file, and emits `PASS` / `FAIL`.
- The deploy pipeline consults the canary-gate script's exit status to decide whether to promote.

The mod-107 module builds the online-eval loop. This module's chapter defines the *interface*: the canary gate is a script the deploy pipeline calls; the online-eval infrastructure is the substrate.

### Progressive rollout stages

Past canary, the rollout advances in stages — 5% → 25% → 50% → 100%, or per-cell / per-region, depending on the platform's convention. The eval-side contract at each stage:

- Compute the same metrics on the cohort included in this stage.
- Auto-rollback if a blocking metric fires (chapter 03's severity applies).
- Hold each stage for a stated time window before advancing — long enough to observe enough traffic for the metric to have signal (a p05 on 20 requests is noise), short enough that the rollout completes within the SLA.
- Emit the per-stage gate result to whatever the platform team's rollout system reads (Argo Rollouts, Flagger, Spinnaker, LaunchDarkly progressive delivery, custom).

Two rollback shapes:

- **Auto-rollback on blocker.** The stage's gate fires; the deploy pipeline halts and rolls back the change. No human in the loop.
- **Auto-halt on blocker, human resume.** The stage halts and pages the on-call; the on-call reads the runbook (chapter 03) and either rolls back or resumes with a documented rationale.

Auto-rollback is safer; auto-halt-and-page is more common early in a program because auto-rollback has its own failure mode (flapping metrics roll back a healthy change). Choose per surface. Chapter 03's `severity` field on the threshold is the machine-readable knob.

### The interface: what the eval program hands the platform team

The deploy pipeline is owned by the platform team. The eval program's contract with them is a small, stable interface. Four artefacts, all owned by the eval program and consumed by the platform team:

1. **A gate script per stage.** `eval/deploy-gates/offline.sh`, `eval/deploy-gates/canary.sh`, `eval/deploy-gates/rollout-stage.sh`. Each exits `0` on pass, non-zero on fail, and writes a structured JSON result. The scripts take environment inputs (artefact ref, canary slice id, cohort id) and no other coupling to the deploy pipeline.
2. **A machine-readable result schema.** JSON with `verdict`, `metrics`, `failures[]`, `pin_diff`, `runbook_urls[]`. The platform team reads `verdict` for the rollback decision and surfaces the rest on the deploy status page.
3. **A runbook per gate.** Chapter 03's runbook shape, extended with the deploy-pipeline specifics: which pipeline stage fired, how to trigger a rollback manually, how to re-promote after fix.
4. **A rollback contract.** A single documented command the platform team runs to reverse the change (image tag pin, feature-flag flip, index restore). This must be *idempotent* and *rehearsed* — a rollback path that is only tested in prod is a rollback path that will not work in prod.

Concretely, the interface is not "the eval team owns the deploy pipeline" — it is a small set of scripts and JSON documents the platform team's existing tooling reads. This is the same contract shape as a database-migration team hands their platform: the migration script is on the eval side; the mechanism to run it, monitor it, and roll it back is on the platform side.

### Rollback in an LLM app: what actually gets reversed

"Rolling back a prompt change" is more subtle than "rolling back a code change". Three shapes:

- **Code-only rollback.** Revert the merge commit; redeploy the previous image. Works when the only thing that shipped was code (prompt file, chain wiring, tool schema). Standard blue/green or image-tag pin.
- **Model-version rollback.** The change was a model-snapshot pin bump; roll back the pin in config and redeploy. Requires the platform team to hold the previous snapshot's pin as an artefact.
- **Retrieval-index rollback.** The change rebuilt the index (new chunker, new embedding); roll back the index pointer to the previous build. The previous build must still exist — write down a retention rule (e.g., keep the last 3 indices).

The runbook for each metric (chapter 03) names which of the three the rollback is. A gate that fires without stating which shape of rollback to run leaves the on-call to figure it out — not what you want at 3am.

### Coordinating with feature flags

Prompt / chain / agent changes ship faster when they are behind a feature flag; the flag becomes the finer-grained rollout knob than the deploy pipeline. Two ways this integrates with the gate:

- **Flag as the canary knob.** Instead of shipping a new artefact and slicing traffic at the load balancer, ship the artefact with the new candidate behind a flag, and use the flag (OpenFeature, LaunchDarkly, Statsig) to control the slice. The eval-side contract is the same — the canary script scores traces from the flag-on cohort.
- **Flag as the rollback knob.** A failing gate flips the flag off; no re-deploy needed. Faster rollback (seconds), and observable — the on-call can see the flag state on a dashboard.

Feature flags do not remove the need for the gate; they change the mechanism by which the gate acts. Either shape is fine; make one of them explicit in the runbook.

### Cost and latency SLOs at the deploy gate

Chapters 04 and 05 asserted per-fixture cost and latency budgets in the PR gate. The deploy gate observes *real* cost and latency — the per-tenant, per-request numbers that flow into mod-111's cost-latency-quality dashboards.

The gate reads them from the trace backend (mod-102) and asserts:

- `cost_per_query_p95_delta <= X`
- `latency_ttft_p95_delta_ms <= Y`
- `latency_e2e_p95_delta_ms <= Z`

Cost and latency regressions are as ship-blocking as quality regressions. Chapter 03's threshold shape and severity apply. The runbook for a cost regression names the on-call playbook — sometimes the answer is "roll back", sometimes it is "accept and adjust the SLO", but the decision is documented, not silent.

### Post-deploy: the online-eval feedback loop

Once the change is 100% rolled out, the gate handoffs to the mod-107 online-eval loop. The loop scores sampled production traces continuously, monitors the same metric distribution, and alerts on drift. That alert path — sampled trace → judge → drift detection → runbook → rollback path — is the topic of mod-107. This module's chapter closes at "the change is fully deployed; the online loop is watching; the runbook is live."

The rollback contract from the deploy gate stays alive after full rollout — a drift alert from mod-107 can still fire the same rollback the canary gate would have fired. That is the loop-closure: the eval program is not just at merge time, it is at every point the change is in production.

### What this module does not build

The **online-eval loop itself** is mod-107. The **safety-specific canary rules** are mod-108. The **cost / latency roll-ups** are mod-111. The **human-in-the-loop review of alerts** is mod-109. All four consume the interface this chapter defines; none is built here.

## Summary

- Three deploy-gate stages: **offline pass on the built artefact** (blocks canary promotion), **canary** (shadow + live-slice eval; blocks progressive rollout), **progressive rollout** (per-stage gate; auto-rollback or auto-halt on blocker).
- The **online judge** at canary reads mod-107's sampled traces — this module defines the interface; mod-107 builds the online-eval loop.
- The **contract with the platform team** is a small set of scripts (per-stage gate), a JSON result schema, per-metric runbooks, and a rehearsed rollback command.
- **Rollback shape** varies by change type: code-only (revert + redeploy), model-version (pin previous snapshot), retrieval-index (roll back index pointer). Runbooks name the shape.
- **Feature flags** are a valid canary and rollback knob; use them without removing the gate.
- **Cost and latency** are gated at deploy time the same way quality is — the trace-backend numbers feed the gate, and the runbook names the response.
- After full rollout, the gate hands off to **mod-107's online-eval loop**; the same rollback contract remains active for drift alerts.

The module ends here. Exercises 01–05 build the artefacts — the PR-gate workflow, the fixture pipeline, the Promptfoo YAML, the DeepEval pytest suite, and the deploy-gate runbook — that mod-107 (online loop) and mod-112 (program-owner posture) will consume as first-class inputs.
