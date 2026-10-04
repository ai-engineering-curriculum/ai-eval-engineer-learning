# exercise-03: Inspect Agent Harness — Solver + Scorer

**Estimated effort:** 3 hours

## Objective

Wrap the scorers from exercises 1 and 2 into an Inspect `Task`: a real `Solver` that drives a small agent against a model, three composed `Scorer`s (per-step, budget, rubric), a `Tool` catalog that your Solver uses, and a tiny `Sample` dataset with reference trajectories in `metadata`. Run `inspect eval`, open the resulting log in the Inspect viewer, and confirm that your scorer stack produces the same verdict shape in-Inspect as it did out-of-Inspect in exercises 1 and 2.

By the end you will have a reproducible Inspect project the labs extend into the full regression suite and that `mod-106` wires into CI.

## Prerequisites

- Exercises 1 and 2 complete.
- Chapter 05 of this module. Read the current Inspect docs at <https://inspect.aisi.org.uk/> *from the URL* — the API evolves across releases and the chapter marks the moving parts explicitly.
- `pip install inspect-ai` (or the current package name on the Inspect docs). Pin the version in a `requirements.txt` or `pyproject.toml`.
- A model API key with a small budget cap. You will run ~5 Samples × ~2 iterations = roughly 10–20 model calls at low cost.
- A local trace viewer compatible with OpenInference (Phoenix in Docker is simplest; see mod-102 chapter 03 if you have not stood one up). Not strictly required but strongly recommended; your Inspect run will dual-emit traces that mod-107 reads later.

## Set up

1. Create a project directory `mod-103/exercise-03/` with a minimal Python layout: `pyproject.toml`, `support_agent_eval/`, `tests/`.
2. Install Inspect, OpenInference auto-instrumentors, and your preferred model SDK.
3. Reuse exercise-01 and exercise-02 code either by `pip install -e ../` or by vendoring the two scorer modules into this project. Document the choice.
4. Enable OpenInference auto-instrumentation so the Inspect run writes traces to your local collector / Phoenix instance.

## Author the Task

Produce `support_agent_eval/task.py` with the following shape (the import paths and decorator names reflect the Inspect docs; **read them from the URL** and adjust if the API has moved — the chapter marks the volatile bits with `<!-- needs-research -->`):

- **Three `@tool`-decorated functions**: `docs_search`, `crm_lookup`, `open_ticket`. The first two return stubs from a small in-memory dict keyed by `order_id`; `open_ticket` returns a synthetic ticket id (and does **not** emit email, network, or side effect — route side-effectful tools through a sandbox or stub).
- **A `Solver`** that drives the agent through the tool loop. Prefer Inspect's built-in agent solver (per the current docs) rather than hand-rolling the loop; chapter 05 explains why.
- **Three `@scorer`-decorated scorers**:
  - `per_step_scorer()` — wraps exercise-01's `score_per_step`. Reads `state.messages` and `state.metadata["reference_trajectory"]`.
  - `budget_scorer()` — wraps exercise-01's `score_budget`.
  - `rubric_scorer()` — wraps exercise-02's `roll_up`. Reads the other two scorers' metadata off `state`.
- **A dataset of 5 Samples** across the `order_status_not_shipped_policy` task family's branches:
  - `S1` found + not shipped + policy applies (the happy path).
  - `S2` found + shipped (should not open a ticket).
  - `S3` found + not shipped + no policy match → escalate.
  - `S4` not found.
  - `S5` the chapter-01 bad-trajectory input (`"Where is my order 12345?"`) with a planner that is intentionally weak — a *regression fixture*; it should fail the rubric.
- **Task-level pins**: model identifier (served, not aliased), `message_limit`, `token_limit`, `time_limit`, `working_limit` set comfortably above the SLO so hard-stops do not confound scoring (chapter 04). Record every pin in `metadata`.

Then:

- **`inspect eval support_agent_eval.task:support_agent_order_status --log-dir ./logs/`**
- **`inspect view ./logs/`** and inspect each Sample's trajectory next to the per-axis scorer output.
- Confirm that `S5` fails with `floor_tripped` naming at least one of `{wrong_tool, ordering_violation}` and that `S1–S4` pass.
- Confirm the OpenInference auto-instrumentor wrote the same five trajectories as AGENT sub-trees to your trace backend.

## Requirements

The deliverable directory contains:

1. **`support_agent_eval/task.py`** as described above.
2. **`support_agent_eval/dataset.py`** that produces the 5 Samples with full `metadata` (including `reference_trajectory`, `task_family`, `reference_version`, `rubric_version`, `slo`).
3. **`support_agent_eval/scorers.py`** wrapping exercise-01 and exercise-02 as Inspect scorers. Scorers are **pure** — no state mutation.
4. **`support_agent_eval/tools.py`** — the three tool stubs with tight typed signatures and docstrings the harness surfaces as tool descriptions.
5. **`support_agent_eval/configs/`** — the rubric YAML from exercise-02, the reference trajectories from exercise-01, and a model price table.
6. **`logs/`** — the artefact of at least one `inspect eval` run, committed (or redacted and committed if your run contains sensitive data).
7. **`memo.md`** (300–500 words) covering:
   - Which Inspect Solver you used and why.
   - Which Inspect version you pinned.
   - Which scorer did the heaviest attribution work on the `S5` regression fixture.
   - Where you had to deviate from the chapter-05 shape and why.
   - What `inspect view` showed you that the raw JSON did not.

## Starter guidance

- **Read the current Inspect docs from the URL** before writing code. The decorator names (`@task`, `@solver`, `@scorer`, `@tool`), the state object (`TaskState`), and the scoring return (`Score`) are stable; the specific built-in agent solver names move. Chapter 05 marks the volatile parts.
- Keep the scorer wrappers thin. The scoring logic lives in exercise-01 and exercise-02; the Inspect layer only adapts I/O.
- `Score.value` is a scalar; store the full per-axis vector in `Score.metadata`. The Inspect viewer renders metadata inline.
- Pin the model via the Task argument, not an environment variable. The model identifier should round-trip into the log.
- Confirm your tools' return values are JSON-shaped (dicts) and not strings. The next model turn and the chapter-02 argument matcher both score more cleanly against structured returns.
- For the regression Sample `S5`, use a *weakened system prompt* that is likely to pick `sql_query` wrongly — but do not add `sql_query` to the authorised tool list. The chapter-02 `unauthorised_tool` verdict should fire. If you add `sql_query` to the tools, the exercise degenerates into testing final-answer correctness.
- Keep the dataset literal small and in-tree for this exercise. Exercise-05 is where you load from mod-110's store.

## Acceptance criteria

You are done when:

- `inspect eval` runs end-to-end and emits a log for all 5 Samples.
- `inspect view` renders each trajectory with the per-step verdict, the budget verdict, and the rubric roll-up visible.
- `S1`–`S4` pass the rubric; `S5` fails with a floor tripped.
- The scorers are the exact logic from exercises 1 and 2 — rerunning the standalone CLI on the Inspect-captured trajectories should produce the same verdicts (within token-counting rounding).
- Trace spans appear in your trace backend for all 5 Samples with `AGENT`, `LLM`, `TOOL` kinds.
- The memo answers all five bullets above.
- The pinned Inspect version is recorded in `pyproject.toml` and in the eval log header.
- Nothing in the project embeds a model API key; it is read from the environment.

## Stretch goals

- **Dual-run.** Run the Task once at `temperature=0.0` and once at `temperature=0.3`; carry a `pass^2` reliability metric (chapter 06's τ-bench shape). Report whether the `S3` (escalate) branch is unstable.
- **Judge stub.** Replace exercise-02's `exact_match_normalised` final-answer scorer with a reference-based judge using Inspect's `grader` facilities (per the docs). Note the per-run cost delta and the attribution the judge adds.
- **Custom Solver.** Author a small `@solver` that composes the Inspect built-in agent with a *pre-check* step that refuses out-of-scope inputs. Verify the pre-check does not count as an agent step for `max_steps`.
- **Collector integration.** Configure the OpenInference auto-instrumentor to attach the rubric verdict back to the trace as an `EVALUATOR` span (mod-102 schema). Confirm it round-trips to Phoenix.

## What this exercise does *not* cover

You are not auditing public benchmarks (exercise-04), you are not replaying anything (exercise-05), you are not wiring Inspect into CI (mod-106), and you are not running on live traffic (mod-107). The dataset is a 5-Sample literal; mod-110 is where the dataset lifecycle lives.
