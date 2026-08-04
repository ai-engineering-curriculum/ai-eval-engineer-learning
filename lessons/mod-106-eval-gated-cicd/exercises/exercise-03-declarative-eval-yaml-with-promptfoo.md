# exercise-03: Declarative Eval YAML with Promptfoo

**Estimated effort:** 3 hours

## Objective

Author a **declarative Promptfoo config** that reads the fixture set from exercise-02, drives your instrumented surface against it, invokes the mod-104 rubric as an LLM judge, asserts on quality + cost + latency + pin integrity, and emits the machine-readable JSON the exercise-01 CI job turns into the PR-comment report.

By the end you will have a real Promptfoo YAML in your repo that another engineer could read cold to answer *what does this gate actually run*. The config is the third of the three artefacts (fixtures, thresholds, config) that make the gate operational.

## Prerequisites

- Chapter 04 of this module.
- Exercises 01 and 02 completed. This exercise consumes their outputs (workflow, fixtures, thresholds).
- Promptfoo installed (`npm install -g promptfoo`; check the current install docs at <https://www.promptfoo.dev/docs/getting-started/>).
- The model API key configured as a repo secret.

## Set-up

1. Create `eval/promptfooconfig.yaml`. Point it at the fixtures from exercise-02 and the rubric from mod-104.
2. Verify locally: `promptfoo eval -c eval/promptfooconfig.yaml --output eval/out/pr-eval.json`. Fix errors before wiring into CI.
3. Confirm the exercise-01 workflow already invokes Promptfoo. If it does not (you used a placeholder in exercise-01), swap it in.

## Requirements

Produce a Promptfoo config, its supporting shims, and one comparison document. Concretely:

1. **`eval/promptfooconfig.yaml`** — the full config. Must:
   - `providers:` — one entry per model under test. Pin the snapshot suffix (chapter 04's rule) and set decoding parameters explicitly (`temperature`, `seed`, `top_p`). Do not use provider aliases like `openai:gpt-4o`; use `openai:gpt-4o-<YYYY-MM-DD>`.
   - `prompts:` — reference the prompt file(s) in the repo (`file://prompts/*.md`). The prompt change from exercise-01's demonstration PR must be visible on this path.
   - `tests:` — load fixtures via `__include__: file://eval/fixtures/*.yaml`. Every fixture becomes one row.
   - `assert:` blocks per test that cover four categories:
     - **Pin check.** A `javascript` (or `python`, or `webhook`) assertion that reads the fixture's `pin` block and compares against the runtime `provider.config` + current retrieval-index hash + current judge-model version. Fails when a pin diverges.
     - **Rubric-scored quality.** `llm-rubric` (or the `llm-rubric-with-context` variant) invoking your mod-104 rubric file. `threshold:` reads from `eval/thresholds.yaml` via the loader shim rather than hard-coding.
     - **Cost.** `cost` assertion with a per-request USD budget.
     - **Latency.** `latency` assertion with a per-request millisecond budget.
   - `sharing: false` unless your team explicitly wants to publish the run to shareable.promptfoo.dev.
   - `outputPath:` set to `eval/out/pr-eval.json`.
2. **`eval/asserts/pin-checks.js`** (or `.py` if you use the Python provider) — the pin-check assertion. Loads the fixture, compares to the runtime provider config, and returns a Promptfoo assertion result (`{ pass: bool, reason: string }`).
3. **`eval/asserts/thresholds_loader.js`** — reads `eval/thresholds.yaml` and exposes `thresholdFor(metricName)`. The `llm-rubric` `threshold:` in the config calls this instead of hard-coding a number, so a threshold change in `thresholds.yaml` propagates without touching the Promptfoo config.
4. **`eval/prompts/`** (if not already present from your surface) — the prompt files referenced in `promptfooconfig.yaml`. The Promptfoo config's `prompts:` entry is the pointer.
5. **`compare.md`** (300–500 words) — a short comparison across three axes:
   - **Rubric-scoring behaviour.** Run the same rubric via Promptfoo's `llm-rubric` and via a direct API call in a scratch script. Report whether the scores agree; describe any divergence and its cause (Promptfoo prompt-template wrapping is the usual culprit).
   - **Pin-check assertion.** Describe how you implemented the check and what it caught during authoring. Cite at least one specific mismatch it surfaced (a snapshot suffix you forgot, an index hash out of date, etc.).
   - **Ease of use vs code review.** Two paragraphs. Would a non-engineer teammate read this YAML and understand what the gate runs? If not, what would you refactor?
6. **CI integration.** The exercise-01 gate step already runs Promptfoo. Confirm:
   - The `outputPath:` matches what `apply_thresholds.py` reads.
   - Caching is on (`PROMPTFOO_CACHE_ENABLED=true` or equivalent).
   - The chapter-04 warn-on-unregistered-metric rule fires: add a metric to `promptfooconfig.yaml` that is *not* in `thresholds.yaml` and confirm the CI job goes red with a readable message. Then remove the metric.
7. **A demonstration PR** that changes the rubric prompt file, opens against your working branch, and shows the report re-scoring with the new rubric — the pin diff surface flags that the judge prompt hash changed.

## Starter guidance

- **Promptfoo's `__include__` on `vars`.** This is the mechanism for loading N fixture files as N test rows. Read the current syntax on <https://www.promptfoo.dev/docs/configuration/expected-outputs/#loading-tests-from-files> — the pattern moves occasionally.
- **`llm-rubric` uses its own judge model.** By default it uses the provider's model — you almost always want to override to a stable judge model (`gpt-4o-<snapshot>` or Claude Sonnet). Set the judge model explicitly and record it in the CI report header.
- **`javascript` assertions run inside Promptfoo's Node runtime.** Do not `require` packages that are not installed globally. Keep the pin-check assertion small (< 50 lines); complex logic belongs in a `webhook` or Python provider.
- **Do not hard-code thresholds in `promptfooconfig.yaml`.** The whole point of chapter 03's `thresholds.yaml` is a single source of truth. If your first pass hard-codes them, refactor to the loader before submitting.
- **Cost assertion needs pricing.** Promptfoo can estimate cost from token counts against a built-in price table. Verify the price table matches your vendor's current pricing (note the date in `compare.md`); if it does not, override via the `costPerToken` config field.
- **Latency measurement.** Promptfoo measures wall-clock latency including its own overhead. This is fine for a per-request budget; do not use it for TTFT / TPOT SLIs, which the trace backend measures separately.
- **Deterministic run.** For the acceptance tests to reproduce, seed everything: temperature 0 (or a low value with a fixed seed), fixed rubric prompt version, cached judge responses. A run whose report changes on every push is a broken run.
- **Read the current Promptfoo docs.** The field names and assertion types change on a monthly cadence; do not rely on this exercise's snippet syntax without verifying.

## Acceptance criteria

You are done when:

- `promptfoo eval -c eval/promptfooconfig.yaml` runs locally to completion and produces the JSON at the configured `outputPath`.
- Every fixture from exercise-02 becomes one test row in the Promptfoo output.
- Each row's assertions include the pin check, the rubric-scored quality, the cost budget, and the latency budget. All four categories are visible in the raw JSON.
- Thresholds are read from `eval/thresholds.yaml` via the loader shim; a threshold change requires no edit to `promptfooconfig.yaml`.
- Deliberately adding an unregistered metric to `promptfooconfig.yaml` makes the CI job fail with a message that names the metric. Removing it makes the CI job pass.
- The demonstration PR that changes the rubric file re-scores the run and surfaces the rubric-prompt-hash change in the report header.
- `compare.md` covers rubric-scoring divergence, one specific pin-check catch, and the YAML-readability recommendation.
- The exercise-01 CI gate still passes (the exercise's changes are additive; the pipe still works end-to-end).

## Stretch goals

- **Stratified sub-suites.** Add a `promptfoo-safety.yaml` config for the mod-108-style safety fixtures with `severity: block` on all metrics and a smaller cost budget. Confirm the CI gate runs both and reports them separately.
- **Judge-model diversity check.** Run the same rubric with two different judge models (e.g., GPT-4o and Claude Sonnet) and compare per-fixture disagreement. Cite the pattern from mod-104 chapter 05.
- **Pairwise regression check.** Add a `promptfoo-pairwise.yaml` config that runs the current prompt against the main-branch prompt and produces a pairwise winner-count. Use this only for release-gate decisions; do not gate a PR on pairwise noise.
- **Custom `webhook` assertion.** Move the pin check from a `javascript` assertion to a Python `webhook` (a small Flask endpoint the CI job starts). Prove the shape works across the assertion-execution boundary.
- **HTML report published as an artefact.** Upload `promptfoo view --format html` output as a CI artefact so the reviewer can drill into per-fixture detail beyond the Markdown PR comment.

## What this exercise does *not* cover

You are not writing the DeepEval alternative (that is exercise-04). You are not writing the deploy gate (exercise-05). You are proving that the declarative-YAML pattern from chapter 04 works end-to-end on real fixtures, real rubrics, real thresholds, and real CI plumbing.
