# Declarative Eval Configs: Promptfoo, DeepEval, and the Framework CI Runners

## Motivation

Chapters 02 and 03 gave you the two artefacts a gate needs: a **replay set** whose scores can be trusted and a **threshold file** that turns those scores into a decision. This chapter is about the third artefact — the **eval config** — that reads both, drives the surface against the replay set, computes the scores, applies the thresholds, and emits a machine-readable report.

The property this chapter builds is **declarative**. The config lives in a file, not in a vendor UI. It is code-reviewed like source. Any team member can look at it and answer *what does this gate actually run*. A gate whose behaviour lives only in "click this in Braintrust" or "log in to Langfuse and edit the experiment" is not code — it is a step in an untracked handbook — and the moment the person who set it up leaves the team, the gate rots.

The good news is that the field has converged on a small set of usable declarative shapes. Promptfoo ships a YAML config. DeepEval ships a pytest hook. Braintrust, Weave, and Langfuse ship SDK-first eval runners that CI can call as a script. None of them is a bad choice on its own — the choice comes down to what your team already runs. This chapter walks the four shapes side by side, calls out where they overlap, and describes when to combine them.

## Core concepts

### The four shapes of an eval-config runner

The runners cluster into four shapes by how the config is expressed and by how it runs in CI.

| Runner | Config surface | CI shape | Sweet spot |
|---|---|---|---|
| **Promptfoo** | `promptfooconfig.yaml` — providers, prompts, tests, assertions | `promptfoo eval` CLI + GitHub Action | Prompt / chain-level A/B and pairwise; PR comment natively |
| **DeepEval** | Python test file — `@pytest.mark.llm_eval`, `assert_test(...)` | `pytest` in CI | Team already fluent in pytest; per-fixture Python logic |
| **Vendor SDK runner** (Braintrust, Weave, Langfuse, LangSmith) | Python / TS script — `braintrust.Eval(...)`, `weave.Evaluation(...)`, `langfuse.evaluate(...)` | Script executed by CI, results posted to vendor UI | You already store fixtures / traces / rubrics in the vendor and want the diff view on the vendor UI |
| **Home-grown** | Custom Python + Great-Expectations / pytest / bespoke | Anything | The gate does something none of the above supports (rare; usually a sign that you should re-check the above) |

A mid-sized team almost always ends up running **two** of these in parallel. A common pattern is: Promptfoo for the mainline PR gate (fast to author, cheap CI, PR comment), plus a vendor runner (Braintrust / Weave / Langfuse) for the release-gate suite that ships the artefacts to a UI product managers actually open. Chapter 05 wires the PR side; chapter 06 wires the deploy side.

### Promptfoo: the YAML config

Promptfoo (<https://www.promptfoo.dev/docs/>) is the simplest place to start. The config is one YAML file. A minimal `promptfooconfig.yaml` for a RAG surface:

```yaml
description: RAG surface offline eval — PR gate

providers:
  - id: openai:gpt-4o-2024-11-20
    label: pr-candidate
    config:
      temperature: 0.2
      seed: 42

prompts:
  - file://prompts/reply.md

tests:
  - description: fixture-loaded suite
    vars:
      # each fixture file becomes one row
      __include__: file://eval/fixtures/*.yaml
    assert:
      - type: javascript
        value: file://eval/asserts/pin-checks.js  # verifies model + index hash match fixture
      - type: llm-rubric
        rubric: file://eval/rubrics/faithfulness_v3.md
        threshold: 0.85
      - type: cost
        threshold: 0.002  # per-request $ budget
      - type: latency
        threshold: 4000   # ms

sharing: false
outputPath: pr-eval-report.json
```

Notes on the shape:

- `providers` pin the model + snapshot + decoding parameters (chapter 02's first pin).
- `prompts` reference the prompt files in the repo — a prompt change PR touches these files, and Promptfoo picks up the change automatically.
- `tests` load fixtures via `__include__` on the vars — one YAML fixture, one test row. This is where the chapter-02 fixture files feed in.
- `assert` blocks are declarative. `llm-rubric` invokes an LLM judge against the rubric file (mod-104), `cost` and `latency` are built-in numeric assertions, `javascript` runs custom logic (used here to check that fixture pins match the runtime).
- `outputPath` is the machine-readable report chapter 05 posts on the PR.

The CLI is `promptfoo eval -c promptfooconfig.yaml`. It exits non-zero on any assertion failure, which is what makes the CI gate a gate.

Two Promptfoo-specific niceties that matter for a PR gate:

- **`share` / PR comment.** Promptfoo ships a GitHub Action (<https://github.com/promptfoo/promptfoo-action>) that posts the eval results as a PR comment. Chapter 05 walks the wiring.
- **Baseline diff.** `promptfoo eval --repeat 3 --output pr.json` and a small script comparing against a stored baseline is the canonical shape for a delta-threshold gate.

### DeepEval: pytest-native eval

DeepEval (<https://docs.confident-ai.com/>) is a Python library that exposes eval work as pytest tests. If your team already treats pytest as the CI contract, DeepEval is the least friction to adopt.

A minimal test file `tests/test_reply_eval.py`:

```python
import pytest
from deepeval import assert_test
from deepeval.metrics import FaithfulnessMetric, ContextualPrecisionMetric
from deepeval.test_case import LLMTestCase

from eval.fixtures import load_fixture
from myapp.surface import answer

FIXTURES = load_fixture.iter_all("eval/fixtures")


@pytest.mark.parametrize("fixture", FIXTURES, ids=lambda f: f.id)
def test_reply_faithfulness(fixture):
    # runtime pin-check
    assert fixture.pin.model == "gpt-4o-2024-11-20"

    reply = answer(fixture.input, retrieval=fixture.retrieval, tool_stubs=fixture.tool_calls)

    tc = LLMTestCase(
        input=fixture.input.messages[-1].content,
        actual_output=reply,
        retrieval_context=[d.content for d in fixture.retrieval.documents],
        expected_output=fixture.reference.answer,
    )

    faithfulness = FaithfulnessMetric(threshold=0.85, model="gpt-4o-2024-11-20")
    precision = ContextualPrecisionMetric(threshold=0.80, model="gpt-4o-2024-11-20")

    assert_test(tc, [faithfulness, precision])
```

The CI job is `pytest tests/ -m llm_eval --junitxml=pr-eval.xml`. Two properties this shape gives you for free:

- **JUnit XML.** CI runners already understand JUnit XML for per-test reporting; the PR check surface shows the failing test names and messages without extra plumbing.
- **Per-fixture Python.** When a fixture needs custom loading, tool stubbing, or setup, you have full Python at hand — you are not fighting a YAML DSL.

The trade-off vs Promptfoo: the config is Python code, not YAML, so a non-engineer product manager cannot as easily read it or tweak a threshold. For engineering-team-facing gates this is fine; for cross-functional gates that a PM should be able to inspect, Promptfoo's YAML is friendlier.

DeepEval's `assert_test` model is what wires the gate into pytest — a metric that misses the threshold raises a test failure, and pytest exits non-zero. That is the property CI cares about.

### Braintrust, Weave, and Langfuse: the vendor SDK runners

The three eval-first vendor platforms all ship an SDK-driven eval runner. The shape is:

```python
# braintrust
import braintrust

def run(fixture):
    return {"output": answer(fixture["input"], ...), "expected": fixture["reference"]}

braintrust.Eval(
    "rag-surface-pr",
    data=lambda: load_fixtures("eval/fixtures"),
    task=run,
    scores=[Faithfulness, ContextPrecision],
    metadata={"commit": os.environ["GITHUB_SHA"], "branch": os.environ["GITHUB_HEAD_REF"]},
)
```

```python
# weave
import weave

weave.init("rag-surface")

@weave.op
def surface(fixture):
    return answer(fixture["input"], ...)

evaluation = weave.Evaluation(
    dataset=weave.ref("rag-fixtures:v14"),
    scorers=[FaithfulnessScorer(), ContextPrecisionScorer()],
)

await evaluation.evaluate(surface)
```

```python
# langfuse
from langfuse import Langfuse
from langfuse.decorators import observe

langfuse = Langfuse()

for item in langfuse.get_dataset("rag-fixtures").items:
    with item.observe(run_name=os.environ["GITHUB_SHA"]) as span:
        span.trace.update(output=answer(item.input, ...))
        # scorers registered separately, evaluated async or via CI Ingestion
```

The shape is the same in all three: a dataset (the fixtures), a task (the surface under test), a set of scorers (the rubrics), and a run object that ties the results to a commit / branch. The vendor UI then shows the diff between two runs (usually PR vs main), and the CI job either:

- **Reads the vendor API for the pass/fail** and exits accordingly. This is the pattern that turns the vendor into an eval-gate runner. Each vendor exposes it slightly differently; check the current docs.
- **Just posts the results and links the vendor UI on the PR** without gating. This is the pattern for a rich diff view without merge-blocking; usually combined with Promptfoo or DeepEval running as the actual gate.

Advantages of the vendor runner:

- **Rich diff UI.** Row-by-row diff between two runs, with the rubric rationales rendered, cost breakdown, per-fixture drill-down. Way better than reading raw JSON.
- **Sample-size and statistical sugar.** Most of the vendors ship built-in bootstrapping, per-scorer aggregate views, and stratification breakdowns.
- **Historical rollups.** Runs are searchable across time; you can answer "how did this metric move over the last quarter" without building a warehouse yourself.

Trade-offs:

- **Config lives in a mix of Python + vendor UI.** The dataset version, the scorer definitions, and the run naming are in code; the diff-view configuration and the alert configuration usually live in the vendor UI. Portability is lower than Promptfoo/DeepEval.
- **Auth setup on every CI runner.** Vendor API key + workspace + project need to be scoped as secrets; there is a small friction cost for self-hosted / air-gapped teams.
- **Data residency.** For teams with strict data-residency requirements, sending eval inputs to a SaaS vendor is a compliance question. Braintrust, Langfuse, and Weave all offer self-hosting; check the current posture.

### Framework bake-off: the criteria

When you have to pick, the criteria that matter for a PR gate are:

| Criterion | Promptfoo | DeepEval | Braintrust | Weave | Langfuse |
|---|---|---|---|---|---|
| **Config declarativeness** | YAML native | Python (pytest) | Python | Python | Python |
| **PR comment support** | First-class Action | Via JUnit / custom | Via API | Via API | Via API |
| **Judge model choice** | Any provider | Any provider | Any provider | Any provider | Any provider |
| **RAG-triad metrics** | Built-in + custom | Built-in (contextual*) | Custom / library | Custom / library | Custom / library |
| **Diff UI** | HTML + JSON | Terminal + JUnit | Rich vendor UI | Rich vendor UI | Rich vendor UI |
| **Self-host** | Yes (OSS) | Yes (OSS, with SaaS Confident-AI option) | SaaS + self-host | SaaS + self-host | Yes (OSS) + SaaS |
| **CI runner story** | Native GH Action | pytest-native | SDK script | SDK script | SDK script |
| **Best fit** | PR gate, prompt/chain A/B | Engineering pytest culture | Rich diff review | W&B users already | Trace-native culture |

<!-- needs-research: re-verify each vendor's current PR comment integration, self-host status, and built-in RAG-triad scorer coverage on the next research cycle. The framework surface changes on a monthly cadence. -->

**Default recommendation.** If you are picking one to start, pick **Promptfoo for the PR gate** — it has the shortest path from zero to a PR-comment-posting gate, and its YAML is the most immediately code-reviewable shape. If your team lives in pytest, **DeepEval** is a co-equal choice. If you already run one of the vendor platforms for trace-instrumentation (Braintrust from mod-102, or Langfuse / Weave / LangSmith), **use the same vendor for the release-gate suite** so the report artefacts live where the team already looks.

### Reading the same fixtures from every runner

The three runners disagree on shape but agree on need. The design that keeps you portable: **fixtures on disk in the canonical schema from chapter 02, and one loader per runner** that adapts them.

- Promptfoo reads them via `__include__: file://eval/fixtures/*.yaml` and a small `assert-javascript` shim.
- DeepEval loads them via `load_fixture.iter_all("eval/fixtures")` in the pytest fixture.
- Vendor runners load them via `def data(): return load_fixture.iter_all("eval/fixtures")`.

The loader normalises to a common Python dataclass so the rubric wiring is identical across runners. This is the shape that lets you run **Promptfoo as the PR gate and Braintrust as the release-gate view simultaneously** without maintaining two fixture sets.

### Reading the threshold file from every runner

Chapter 03's `thresholds.yaml` file must not be duplicated per runner. Two patterns:

- **Loader macro.** A small `thresholds.py` module reads the YAML and exposes `threshold_for("faithfulness_mean")`. Promptfoo and vendor runners call it via a JavaScript / Python shim in the assertion block; DeepEval calls it directly.
- **Emit-then-check.** Each runner emits raw per-fixture scores; a downstream `apply-thresholds.py` script reads the emitted JSON and the threshold file, applies the pass/fail, and exits with the CI status. Chapter 05 uses this pattern for the PR gate because it keeps runner-specific complexity out of the shared thresholds config.

The emit-then-check pattern is the more portable of the two — the runner's job is to score, the threshold script's job is to gate, and the threshold script is a single point of truth across every runner.

### Cost, latency, and pin assertions belong in the config

The rubric asserts on quality. The config asserts on the rest of the SLIs (cost per query, p95 latency) and the pins (model snapshot, retrieval index hash, judge model). Two rules:

- **Every fixture asserts its pins match runtime.** If the runtime is running against `gpt-4o-2024-11-20` but the fixture was captured on `gpt-4o-2024-08-06`, the assertion fires and the gate turns red with a clear "pin mismatch" message. This is what surfaces silent vendor upgrades.
- **Every rubric has a paired cost / latency budget.** Faithfulness at 0.99 is not a win if per-query cost tripled. Chapter 06 covers the cost SLO wiring against the deploy gate; the PR gate at least enforces a per-fixture budget.

### Reproducibility again: the CI job header

Every CI run of the gate emits a **header** at the top of the report that names the four pins in force (model, seed / fingerprint, index hash, judge model), the version of the threshold file, and the commit SHA of the eval-config directory. This is what lets a reviewer six weeks later reproduce the exact run — the header is the fingerprint of the gate itself. Chapter 05's PR comment format includes it.

### When the framework does not fit

The framework surface is not universal. Two shapes where you fall back to home-grown Python:

- **Multi-turn dialog eval.** Most single-shot eval frameworks handle multi-turn awkwardly; you often end up writing a small pytest harness that drives the surface turn-by-turn and calls the framework's scorer per turn.
- **Trajectory eval (mod-103).** Rubrics that judge the sequence of tool calls rather than the final output do not fit cleanly into any of the four runners today. Pattern: write the trajectory rubric as a Python scorer and register it as a DeepEval `BaseMetric` or a Braintrust custom scorer.

In both cases the pattern is *"use the framework for the pass/fail wiring and CI plumbing; write the scorer yourself"*. Do not build a home-grown CI runner — the CI wiring is where you want the framework's work.

## Summary

- The eval config is the third artefact of a serviceable gate: **declarative**, in the repo, code-reviewed, portable across runners.
- Four runner shapes: **Promptfoo** (YAML + GH Action), **DeepEval** (pytest-native), **vendor SDK runners** (Braintrust / Weave / Langfuse / LangSmith), and **home-grown**.
- Default pick: Promptfoo for the PR gate; add a vendor runner for the release-gate diff view if your team already lives in one.
- Every runner reads the same fixture files (chapter 02 schema) via a small loader per runner; every runner reads the same `thresholds.yaml` (chapter 03) via a loader macro or emit-then-check pattern.
- Every fixture asserts its pins match the runtime, and every rubric is paired with a cost + latency assertion.
- The CI job emits a **header** naming every pin, so a run is reproducible six weeks later.
- Framework, not home-grown: use the framework for CI wiring; write bespoke scorers only where the surface is not covered (multi-turn, trajectory).

Chapter 05 wires this config into the PR CI — a GitHub Actions job that runs the gate, posts the report, and turns the check red on regression.
