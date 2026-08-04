# exercise-05: Judge-Tier Routing and Drift Monitor

**Estimated effort:** 3 hours

## Objective

Wire the judge pipeline you built through exercises 01–04 into a three-tier router (frontier / mid-tier / open-source) that respects a stated cost budget, and stand up a small **drift monitor** that reruns the judge on a frozen replay set on a schedule and alerts on statistic shift. This is the closing artefact of the module — everything above becomes durable when it is routed and monitored.

By the end you will have a routing configuration, a runtime that reads it, a frozen replay set held separately from your calibration gold set, a monitor script that runs the replay and computes the drift statistic, a first (baseline) run of the monitor, and a written runbook describing the on-call triage per drift alert.

## Prerequisites

- Chapter 05 of this module.
- Exercises 01 (rubrics), 02 (bias-controlled pipeline), 03 (framework wiring), 04 (calibration) completed. The routing rule reads exercise-04's calibration; the drift monitor reads exercise-01's rubric hash.
- Access to at least two judge tiers — a frontier API and either a mid-tier API or a self-hosted OSS judge (Prometheus 2 or JudgeLM via vLLM). If you can only wire two tiers, note the missing tier as a gap in the routing config; the exercise still works.
- Python 3.11+; a scheduler you can point at a script (cron, systemd timer, or your team's job scheduler). GitHub Actions on a cron schedule works if you have nothing else.

## Set-up

1. Extract the judge-tier list into a small config file — `judge-tiers.yaml` — with one entry per tier: name, model, framework wrapper (from exercise-03), per-call cost estimate, chapter-04 calibration statistic (from exercise-04's `calibration.md`).
2. Prepare a **frozen replay set** — 50 (input, retrieved-context, candidate) triples, held completely separate from the exercise-04 calibration gold set. The replay set does *not* need human labels; it exists to detect judge drift by comparing today's judge output to a pinned baseline. Sample stratified as in exercise-04.
3. Run the current-production judge (whichever tier is your default) on the frozen replay set once, and pin the results as the baseline (`replay-baseline.csv`).

## Requirements

Produce a single directory `mod-104/exercise-05/` in your working repo containing:

1. **`judge-tiers.yaml`** — the tier config. One entry per tier, with:
   - `name`: `frontier` / `mid` / `oss`
   - `model`: model id + snapshot / weights hash
   - `framework`: `deepeval` / `braintrust` / `promptfoo` / custom
   - `per_call_cost_usd`: your best estimate (compute from current pricing; note the date)
   - `calibration_ref`: pointer to the exercise-04 calibration doc for this tier
   - `default_for_criteria`: list of criterion names that route to this tier by default (e.g., release-gate rubrics default to `frontier`).
2. **`router.py`** — implements chapter 05's `route_judge(decision, oss_result=None)` logic. Read `judge-tiers.yaml`; return a tier name. Include the escalation path: OSS judge returns `tie` or `uncertain` → route the same decision to the mid tier. Emit `app.judge_tier` on the EVALUATOR span (real trace hook if you have one; a print statement if not).
3. **`budget.py`** — a small budget tracker. Reads a monthly per-tier cost cap; increments spend after each judgement; when a tier's cap is hit, the router falls back to the next-cheaper tier. Print (or emit as a metric) the per-tier spend so far.
4. **`replay-baseline.csv`** — the pinned baseline of judge outputs on the frozen replay set. Columns: `item_id`, `judge_model`, `judge_prompt_hash`, `judge_score` (or `winner`), `judge_run_at`. This is the ground-truth "how did the judge score last month" file. Committed alongside the code.
5. **`drift-monitor.py`** — a script that:
   - Loads `replay-baseline.csv` and the frozen replay set.
   - Runs the current-production judge on the same items.
   - Computes a drift statistic. For absolute rubrics: the shift in mean score, plus the distribution of per-item score deltas. For pairwise: shift in win-rate and tie-rate, plus per-item winner-agreement. Also: whether the current judge prompt hash matches the baseline hash, and whether the current judge model version matches the baseline model version.
   - Emits a structured result to `drift-monitor-YYYY-MM-DD.json`, and optionally to whatever telemetry sink your team uses.
   - Exits nonzero (or fires an alert) if the drift exceeds a chosen threshold (e.g., mean score shift > 0.3 on a 1–5 scale, or tie-rate shift > 15%). Threshold is configurable.
6. **`drift-runbook.md`** — the on-call runbook for a drift alert. Walk chapter 05's triage checklist:
   - Compare rubric prompt hash → if changed, action X.
   - Compare judge model / snapshot → if changed, action Y.
   - Compare input distribution vs calibration set → if diverged, action Z.
   - Otherwise → escalate to the model-evaluation-engineer peer per exercise-04.
   Include the concrete commands the on-call runs (which script, which config file, which telemetry query).
7. **`bakeoff-scenario.md`** — 300–500 words. Walk one plausible scenario end-to-end: "the frontier vendor announces a new snapshot next Tuesday; here is what happens in this pipeline." Cite chapter 05's frontier-upgrade playbook. What gets pinned, what gets re-calibrated, what gets rolled back if the new baseline is worse.

## Starter guidance

- **Do not run the drift monitor on the calibration gold set.** The gold set is what defends the SLI threshold; the replay set is what detects change. If you use the same set for both, drift-monitor triggers force calibration re-runs and you never accumulate signal.
- **Do not implement OSS routing without exercise-03's OSS-judge variant** — you cannot honestly claim OSS calibration without having built one. If exercise-03's OSS variant is a stretch goal you did not do, note it as a gap and route only between frontier and mid.
- **Budget-cap the frontier tier low** in the first run — you want to observe the fallback path fire. If the fallback never fires, you are not stress-testing the router.
- The **first drift-monitor run's job is to become the baseline**, not to trigger. Confirm this in `bakeoff-scenario.md`.
- For the drift statistic, keep it **honest and simple**. Mean score shift + per-item delta distribution is more useful than a fancy composite that hides what changed.
- The runbook is a **document that another engineer can execute cold**. Trim jargon. Include the exact filenames and commands.

## Acceptance criteria

You are done when:

- `judge-tiers.yaml` lists at least two tiers with all fields populated, and `default_for_criteria` names the criteria from exercise-01 explicitly.
- `router.py` correctly routes decisions per chapter 05's rule, and OSS uncertain cases escalate to mid.
- `budget.py` tracks per-tier spend and forces fallback when a tier hits its cap.
- `replay-baseline.csv` is committed and holds a first pinned baseline from a specific judge model + prompt hash + date.
- `drift-monitor.py` produces a structured result and fires (nonzero exit or alert) when the current judge output shifts beyond the configured threshold.
- `drift-runbook.md` is a document another engineer could execute cold; it names the specific triage actions per chapter 05.
- `bakeoff-scenario.md` walks a plausible frontier-upgrade scenario end-to-end and cites the chapter-05 playbook.

## Stretch goals

- **Schedule the drift monitor** to run nightly against your production judge configuration. Confirm at least three runs produce baseline-agreeing output; then intentionally rewrite one word of your rubric prompt template and confirm the monitor fires.
- **Add a per-criterion cost dashboard.** Break down the per-tier spend by criterion so you can see which criterion is driving frontier spend. Plot as a small streamlit / grafana / whatever.
- **Add the mod-108 safety rubric** to the routing config with a hard override to the frontier tier regardless of cost cap. Note in `drift-runbook.md` how that changes the fallback behaviour.
- **Wire the drift alert to a mod-109 human-review queue** — on a fire, promote the drifted replay items to human review. Confirms the mod-102 → mod-104 → mod-109 loop is real.
- **Write a migration playbook** for swapping the frontier judge model when the current snapshot is deprecated. Chapter 05's frontier-upgrade playbook is the source; make it concrete for your team's chosen vendor.

## What this exercise does *not* cover

You are not building the full mod-107 online-eval loop that reads this router — that is mod-107's exercise. You are not doing the full-methodology drift analysis (that is model-evaluation-engineer peer territory). You are not writing the mod-112 release-gate architecture — that reads this router as one input among many. Stay narrow: tiers, budget, replay, monitor, runbook.
