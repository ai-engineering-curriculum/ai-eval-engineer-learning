# exercise-04: DeepEval Pytest Hooks in CI

**Estimated effort:** 2 hours

## Objective

Port at least one of the metrics from exercise-03 to a **DeepEval pytest gate** and run it in parallel with the Promptfoo gate in the same CI job. Produce a short note comparing the two runners on the same fixture set and same rubric — where they agree, where they diverge, and which one fits which team.

By the end you will have first-hand evidence for chapter 04's bake-off table on the two most-adopted OSS runners, and you will know from experience whether your team wants Promptfoo, DeepEval, or both.

## Prerequisites

- Chapter 04 of this module.
- Exercises 01, 02, 03 completed. This exercise reuses the fixtures, rubric, thresholds, and workflow from those.
- DeepEval installed (`pip install deepeval`; check the current docs at <https://docs.confident-ai.com/>).
- Python 3.11+ with pytest.

## Set-up

1. Create `tests/eval/` in your repo (separate from any non-eval test suite so path filters and CI parallelism are cleaner).
2. Copy the fixture loader from exercise-02 into a pytest-usable form (`eval/fixtures/loader.py` returning a list of `Fixture` dataclasses).
3. Verify locally: `pytest tests/eval/ -m llm_eval -v`. Fix errors before wiring into CI.

## Requirements

Produce a DeepEval variant of the exercise-03 gate. Concretely:

1. **`tests/eval/conftest.py`** — pytest fixtures and hooks:
   - A `@pytest.fixture(params=fixtures, ids=...)` that yields one `Fixture` per parameter.
   - A `@pytest.fixture(scope="session")` that loads `eval/thresholds.yaml` and exposes `threshold_for(metric_name)`.
   - A pytest hook (`pytest_collection_modifyitems`) that applies the `llm_eval` marker to every collected test.
2. **`tests/eval/test_reply_faithfulness.py`** — the pytest test that:
   - Loads a fixture and asserts its pin block against the runtime provider config (mirror of the exercise-03 pin check, in Python).
   - Runs the surface against the fixture input with the pinned model.
   - Constructs a DeepEval `LLMTestCase` with `input`, `actual_output`, `retrieval_context`, `expected_output`.
   - Invokes DeepEval's `FaithfulnessMetric` (or equivalent for your rubric) with the threshold read from `thresholds.yaml`.
   - Uses `assert_test(tc, [metric])` to raise a pytest failure on threshold miss.
3. **`tests/eval/test_cost_and_latency.py`** — a second test that asserts on the same per-request cost and latency budgets from exercise-03. Reads token counts from the surface (or the mod-102 trace) and computes cost against your pricing table.
4. **CI wiring — add to `.github/workflows/eval-gate.yml`:**
   - A new step `Run DeepEval gate` that invokes `pytest tests/eval/ -m llm_eval --junitxml=eval/out/pytest.xml`.
   - The step is **parallel to** the Promptfoo step (both must pass for the job to pass), not in place of.
   - Publish the JUnit XML via an action that surfaces per-test failure names on the PR check UI (`EnricoMi/publish-unit-test-result-action@v2` or equivalent).
5. **`compare-runners.md`** (400–600 words) covering:
   - **Score agreement.** Run both Promptfoo and DeepEval on the same 6–10 fixtures with the same rubric and the same judge model. Table: fixture id, Promptfoo score, DeepEval score, delta, whether the pass/fail decision agrees. If the two disagree by more than ~5% on a fixture, note the likely cause (prompt-template wrapping, tokenization differences, judge-invocation shape).
   - **Author-experience diff.** Two paragraphs. Which one was faster to author? Which one produces a more readable failure message when a test fails? Which one is easier for a non-engineer to read?
   - **Team-fit recommendation.** One paragraph. Which runner would your team keep, why, and where would you use both (e.g., Promptfoo for PR, DeepEval for a specialised sub-suite the eng-heavy team owns)? Cite the chapter-04 bake-off table.
6. **A demonstration PR** where the DeepEval gate turns red on a rubric-regressing prompt change, and the failing test names the specific fixture and metric in the pytest output.

## Starter guidance

- **Do not duplicate the fixture set.** Both runners read the same YAML files from exercise-02. Duplicated fixture sets diverge silently within a week.
- **Do not duplicate the thresholds.** Both runners read `thresholds.yaml` via a loader; a threshold change propagates to both. If you find yourself editing two files to change one number, refactor.
- **Judge model pin.** DeepEval's `model=` argument on `FaithfulnessMetric` (and its cousins) determines the judge. Set it explicitly to the same snapshot as the Promptfoo config. Do not leave it defaulting.
- **Deterministic pytest runs.** DeepEval's metrics call a judge model; without seed / temperature control they can flake. Set `temperature=0` on the metric's model init. Skip `pytest-xdist` for the eval tests — the metric internals may not be parallel-safe in every DeepEval release.
- **Read the current DeepEval docs.** The metric class names, `LLMTestCase` fields, and `assert_test` signature evolve — verify the current signatures before pasting anything from this exercise's snippets.
- **JUnit XML matters.** Uploading a well-formed JUnit XML is what makes the DeepEval failures render nicely on the PR check UI. Do not skip this.
- **Do not remove Promptfoo.** The exercise runs both runners in parallel — the whole point of the comparison. Removing Promptfoo removes half the evidence.
- **Score-agreement expectation.** Do not expect exact equality across the two runners. Both use LLM judges; both wrap the rubric prompt slightly differently. A 3-5% divergence per fixture is normal. Larger divergence is the interesting signal — look at the actual judge prompts each runner sent.

## Acceptance criteria

You are done when:

- `pytest tests/eval/ -m llm_eval` runs locally to completion; every fixture produces one test result; failing thresholds fail the test.
- The CI job runs Promptfoo and DeepEval in parallel; both must pass for the job to succeed.
- JUnit XML from the DeepEval run is uploaded and rendered on the PR check UI with per-test failure names.
- `thresholds.yaml` is the single source of truth — both runners read from it via loaders, and a threshold change is visible in both without duplication.
- Pin-check asserts fire in DeepEval the same way they fire in Promptfoo — a fixture with the wrong `pin.model` fails the pytest test with a readable message before invoking the surface.
- `compare-runners.md` includes the per-fixture score table, the author-experience paragraphs, and a defensible team-fit recommendation citing the chapter-04 table.
- The demonstration PR turns the DeepEval gate red on a rubric-regressing prompt change and names the failing fixture in pytest output.

## Stretch goals

- **Custom `BaseMetric` for a trajectory rubric.** Write a DeepEval `BaseMetric` subclass that scores the tool-call trajectory (chapter 04's mention of the pattern; mod-103 covers the rubric shape). Register it alongside `FaithfulnessMetric` and confirm the pytest failure names the trajectory step that regressed.
- **Confident-AI hosted dataset.** If your team uses DeepEval's SaaS (Confident-AI), push the fixtures as a dataset and demonstrate the hosted diff view alongside the CI runs. Note the data-residency posture.
- **Sample-first-N parity.** Mirror the exercise-01 stretch goal: the PR runs a stratified subset of `tests/eval/`; a nightly runs the full set. Use `pytest --collect-only` and a small selector script to drive.
- **Skip-on-pin-mismatch.** Add a `@pytest.mark.skipif(pin_mismatch, reason=...)` variant that skips the surface-invocation part of a test when the pin mismatch is detected (still failing the pin-check test itself). Reduces wasted API calls when a bulk pin update is in flight.
- **Report-shape parity.** Extend `apply_thresholds.py` to consume the DeepEval JUnit XML as well as the Promptfoo JSON, so the PR-comment report is a single unified surface across both runners.

## What this exercise does *not* cover

You are not choosing "Promptfoo or DeepEval forever". You are producing the evidence to make that choice defensibly. Some teams end up running both; some pick one and remove the other. Either is fine — the exercise's deliverable is the informed decision, not a specific runner.
