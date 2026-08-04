# exercise-05: Deployment Gate and Runbook

**Estimated effort:** 2 hours

## Objective

Extend the PR gate you built in exercises 01–04 into a **deployment gate** — the offline-artefact re-run, the canary-stage gate that reads the mod-102 trace backend, the JSON interface handed to the platform team, and the on-call runbook for a gate-fired rollback. This closes the last of the five gate properties from chapter 01: **wired into deploy**.

By the end you will have the machinery that turns a green PR into a *conditional* deploy — one that continues only while the offline gate, the canary gate, and each progressive-rollout stage are passing, with a documented rollback path.

## Prerequisites

- Chapters 03 and 06 of this module.
- Exercises 01–04 completed. This exercise's scripts consume their outputs.
- A trace backend from mod-102 with production-shaped traffic hitting the surface (real or simulated). Simulated traffic is fine — a small script producing 200 traces / minute against your surface will do.
- A "deploy pipeline" you can drive. Anything works: a shell script, a GitHub Actions workflow with a `workflow_dispatch` trigger, a Kubernetes manifest via `kubectl` locally, an Argo Workflow, an Argo Rollouts config, a LaunchDarkly / OpenFeature project. The point is that *something* consumes the JSON contract.
- Python 3.11+.

## Set-up

1. Decide the shape of your "canary" for the exercise. Two acceptable options:
   - **Traffic split** — 5–10% of traffic routed to the new candidate via your load balancer / service mesh / feature flag; both cohorts write traces to the same backend, tagged with a cohort attribute.
   - **Shadow** — 100% of traffic served by production; a fraction mirrored asynchronously through the candidate, results not returned to the user; the candidate's traces are tagged separately.
   Either satisfies the exercise; state which one you chose in `deploy-interface.md`.
2. Instrument the surface (extending mod-102) to tag traces with the cohort id and the release id (`app.release_id`, `app.cohort`). Both fields must be queryable from the trace backend.

## Requirements

Produce a directory `eval/deploy-gates/` containing the scripts, the interface, and the runbook:

1. **`eval/deploy-gates/offline.sh`** — the offline-artefact gate:
   - Inputs (env): `ARTEFACT_REF` (image tag / model version / commit SHA), `CONFIG_ROOT` (path to `eval/`).
   - Runs the *full* replay set (not the PR-gate sample) against the artefact.
   - Emits `eval/out/offline-gate.json` in the schema below.
   - Exit 0 on pass, non-zero on any blocking failure.
2. **`eval/deploy-gates/canary.sh`** — the canary-stage gate:
   - Inputs (env): `RELEASE_ID`, `COHORT`, `WINDOW_MIN` (default 30), `TRACE_BACKEND_URL`, `TRACE_BACKEND_KEY`.
   - Queries the trace backend for traces from the cohort in the last `WINDOW_MIN` minutes.
   - Reads the mod-107-style online judge scores from the traces (or runs the OSS-tier judge inline if mod-107 is not yet built). Chapter 06's note on the mod-107 interface applies.
   - Applies `eval/thresholds.yaml` (chapter 03) with a *stricter* delta budget than the PR gate — canary regressions are more actionable because they represent real distribution.
   - Emits `eval/out/canary-gate.json`.
   - Exit 0 on pass, non-zero on any blocking failure.
3. **`eval/deploy-gates/rollout-stage.sh`** — the per-stage rollout gate:
   - Inputs (env): `RELEASE_ID`, `STAGE_PERCENT`, `WINDOW_MIN`.
   - Same logic as canary but scoped to the cohort included in this stage.
   - Emits `eval/out/rollout-stage-<PERCENT>.json`.
   - Exit 0 on pass, non-zero on blocker.
4. **JSON schema `eval/deploy-gates/schema.json`** — the machine-readable result contract every gate script emits. At minimum:
   - `verdict`: `pass` | `fail` | `warn`
   - `stage`: `offline` | `canary` | `rollout`
   - `release_id`: string
   - `pin_fingerprint`: object (model, snapshot, judge model, retrieval index hash)
   - `metrics[]`: each with `name`, `value`, `baseline`, `delta`, `threshold`, `severity`, `pass`
   - `failures[]`: list of failed metric names + top-3 offending fixture / trace ids
   - `runbook_urls[]`: URLs to the per-metric runbooks
   - `timestamp`, `commit`, `gate_config_version`
5. **`eval/deploy-gates/rollback.sh`** — the rollback command handed to the platform team. Should be idempotent, safe to re-run, and cover at least one of the three rollback shapes from chapter 06 (code-only revert, model-version pin, retrieval-index restore). State clearly which shape it covers.
6. **`eval/runbooks/deploy-gate.md`** — the on-call runbook for a canary-stage or rollout-stage gate firing. At minimum:
   - Named on-call rotation.
   - The commands to (a) read the JSON verdict, (b) inspect the top failing traces in the backend, (c) trigger the rollback, (d) resume the rollout after fix.
   - The three rollback shapes and how to decide which one applies.
   - Escalation contact if unresolved within the SLA window.
7. **`eval/deploy-gates/deploy-interface.md`** — the platform-team-facing document. 400–600 words. Names the scripts, the JSON contract, the runbook, and the rollback command. Reads like a hand-off doc to a team that did not build this. Includes:
   - Which "canary" shape you chose (traffic split or shadow) and why.
   - The environment variables each script reads.
   - The exit-code contract (0 = pass, non-zero = fail; specific non-zero codes if you use them).
   - The severity semantics (blocker fails, warn passes).
   - The rehearsal procedure (see below).
8. **A rollback rehearsal.** Actually run through the rollback once. Trigger a synthetic failure — either a prompt regression on the canary cohort or a hard-fail an assertion in the offline gate — walk the runbook step by step, execute the rollback, and confirm the cohort's traces return to baseline. Record a short (< 300 words) walkthrough in `deploy-interface.md`'s "Rehearsal" section.

## Starter guidance

- **Chapter 06 is the source.** Re-read it before starting. Every acceptance criterion in this exercise maps to a section in that chapter.
- **The gate is a script, not a service.** The platform team's deploy pipeline invokes the script; the eval team owns what the script does. Do not build a long-running service — the gate is a black-box command with a clear exit code.
- **Stricter delta budgets at canary.** The PR gate scored the replay set; the canary scores real traffic and is the last chance before user impact. Chapter 06's rule: `severity: block` should be the default for canary; `warn` is rare.
- **The stage cohort is finite.** A p05 on 20 requests is noise. Set `WINDOW_MIN` such that the cohort has ≥ 100 traces before the gate emits a verdict; return `warn` (not `fail`) if the traffic is insufficient.
- **Sample-and-judge, do not judge-all.** For canary and rollout stages, sample traces per mod-102 chapter 06's policy and score the sample with the OSS-tier judge (mod-104 chapter 05). Do not score every single trace inline — cost and latency will spiral.
- **Rollback must be rehearsed.** A rollback path that is only tested in prod is one that will not work in prod. The exercise's rehearsal step is not optional.
- **The runbook is executable cold.** Same rule as exercise-01's PR runbook. Hand it to a teammate; they either execute step 1 without questions or you rewrite step 1.
- **State what you do not build.** The exercise sits on top of mod-107's online-eval loop, which is one module downstream. If your mod-107 loop is not built yet, the canary script can call the mod-104 OSS judge inline as a bootstrap; note this in `deploy-interface.md` and cite the mod-107 interface.

## Acceptance criteria

You are done when:

- Three gate scripts (`offline.sh`, `canary.sh`, `rollout-stage.sh`) each run to completion, emit JSON matching `schema.json`, and exit 0 or non-zero per severity.
- The JSON contract is documented in `schema.json` and the schema is enforced (a script that emits invalid JSON fails a schema-lint step).
- `rollback.sh` is idempotent (running it twice does not double-rollback) and covers at least one of the three rollback shapes.
- `deploy-gate.md` runbook names an on-call rotation, the specific commands, the three rollback shapes and their decision criteria, and the escalation contact.
- `deploy-interface.md` reads as a hand-off doc — names the scripts, the contract, the runbook, the rollback, the rehearsal procedure.
- The rollback rehearsal is documented with the actual sequence of events, the commands executed, and the observed trace-backend recovery. Not aspirational — actually executed.
- The canary script correctly falls back to `warn` when the cohort has fewer than 100 traces in the window (do not fail the gate on insufficient signal).
- The pin-fingerprint from the offline-gate JSON matches the artefact under test (model snapshot, judge model, retrieval-index hash). A mismatch fires the gate with a readable message.
- No script hard-codes a secret. All API keys and backend URLs are env-injected.

## Stretch goals

- **Auto-halt-with-page.** Wire the canary gate's non-zero exit to a page (PagerDuty test alert, a Slack webhook to a `#eval-oncall` channel) that includes the runbook URL and the failing metric names. Confirm the page reaches the on-call within 60 seconds.
- **Auto-rollback for the safest failure mode.** Choose one specific failure mode (e.g., "safety-refusal rate spiked >10%") and wire the deploy pipeline to auto-execute `rollback.sh` on that specific failure, no human in the loop. Log the auto-rollback event to the eval-program dashboard.
- **Feature-flag knob.** Replace the traffic-split canary with a feature-flag canary (OpenFeature / LaunchDarkly / Statsig). The rollback becomes a flag flip. Document the trade-off in `deploy-interface.md`: faster rollback, but the deploy artefact does not roll back — so the *next* deploy still contains the change.
- **Progressive-rollout demo.** Simulate a 5% → 25% → 50% → 100% rollout, run the stage gate at each stage, and cause an artificial regression at 50%. Show the auto-halt with the correct verdict, cohort id, and runbook link.
- **Cost / latency canary.** Extend `canary.sh` to also assert cost-per-query and p95-latency deltas against the baseline cohort. Chapter 06's rule: cost/latency regressions are as ship-blocking as quality regressions.

## What this exercise does *not* cover

You are not building the mod-107 online-eval loop from scratch (that is mod-107). You are not implementing the platform team's deploy pipeline (that is theirs). You are producing the eval-side interface — the scripts, the schema, the runbook, the rehearsed rollback — that the platform team's pipeline consumes.
