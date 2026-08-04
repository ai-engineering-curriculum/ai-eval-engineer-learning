# exercise-01: PR Eval Report in GitHub Actions

**Estimated effort:** 3 hours

## Objective

Stand up the **PR eval gate** for the instrumented surface you built in mod-102 — a GitHub Actions workflow that runs on every pull request, executes an eval config against a small replay set, applies pre-registered thresholds, posts a Markdown report as a sticky PR comment, and reports a required check that blocks merge on a regression.

By the end you will have the load-bearing pipe of the whole module — the CI plumbing the next four exercises hang off of. Exercise-02 supplies the fixtures, exercise-03 supplies the Promptfoo config, exercise-04 adds a DeepEval variant, and exercise-05 extends the pattern to the deploy gate. You are wiring the pipe first because it is the piece a reviewer first meets; everything else refines what flows through it.

## Prerequisites

- Chapters 01, 03, and 05 of this module. Chapters 02 and 04 are helpful; you can finish this exercise with a placeholder fixture set and refine in exercise-02 / 03.
- A repo with an instrumented LLM surface (the artefact from mod-102). If you finished mod-102 exercises 01–02 you have this; otherwise, a small `answer(question)` service is enough.
- A GitHub repository you can push to and configure branch-protection rules on. GitHub Actions is the primary target; the exercise's stretch goal covers GitLab / Buildkite / CircleCI equivalents.
- A model API key with a low daily cap (a `EVAL_OPENAI_KEY` or equivalent scoped separately from the production key). ≤ $10 / day cap is sufficient for this exercise.
- Node 20, Python 3.11+.

## Set-up

1. Create an `eval/` directory in your repo with the layout from chapter 05:

   ```
   eval/
   ├── fixtures/
   ├── rubrics/
   ├── runbooks/
   ├── thresholds.yaml
   ├── promptfooconfig.yaml
   ├── apply_thresholds.py
   └── requirements.txt
   ```

2. Author 6–10 minimal fixtures — YAML files matching the chapter 02 schema. If you have not done exercise-02 yet, hand-author against a real trace; refine them in exercise-02. Each fixture must at minimum have `id`, `input`, `pin`, `reference`, and `rubric_refs`.
3. Author `eval/rubrics/faithfulness.md` — a rubric text file matching mod-104's shape. If you have not done mod-104 yet, use a short 1–5 anchored rubric for reply groundedness.
4. Write `eval/thresholds.yaml` with two blocking metrics (`faithfulness_mean >= 0.80` absolute, `faithfulness_mean delta_vs_main >= -0.02`) and one warn metric (`cost_per_query_delta <= +5%`, warn). Add an owner alias per metric.
5. Register the model API key as a repo secret named `EVAL_OPENAI_KEY` (Settings → Secrets and variables → Actions).

## Requirements

Produce a PR (against your fork or working branch) that adds:

1. **`.github/workflows/eval-gate.yml`** — the PR-gate workflow. Must:
   - Trigger on `pull_request` with a tight `paths` filter (chapter 05's list — prompts, chains, agents, tools, eval directory, the workflow file itself).
   - Set `concurrency` with `cancel-in-progress: true`, keyed on `github.head_ref`.
   - Set explicit `permissions` — `contents: read`, `pull-requests: write`, `checks: write` — no wildcards.
   - Set a hard `timeout-minutes` budget (15 minutes is a reasonable ceiling; smaller is better).
   - Cache the judge responses across runs, keyed on fixture + rubric hashes.
   - Run the eval config against the fixture set, apply thresholds, produce `eval/out/pr-eval-report.md`.
   - Upload `pr-eval-report.md` and the raw JSON as an artefact with 30-day retention.
   - Post the report as a **sticky** PR comment (updates in place on repushes; does not spawn a comment per push).
2. **`.github/workflows/eval-baseline.yml`** — the baseline-writer that runs on `push` to `main`, produces `main-eval.json`, and uploads it as a 90-day-retention artefact the PR gate downloads.
3. **`eval/apply_thresholds.py`** — the emit-then-check bridge from chapter 04. Reads the runner's raw scores JSON, reads `eval/thresholds.yaml`, reads the baseline artefact when a delta threshold applies, produces the Markdown report in chapter 05's format (header + summary table + top regressions + pin diff + runbook links + warnings section), and exits nonzero on any blocking failure.
4. **`eval/runbooks/faithfulness_mean.md`** — at minimum the metric owner, threshold justification, first step, common causes, and escalation from chapter 03. The runbook must be executable by an engineer who did not write the PR.
5. **`.github/CODEOWNERS`** — protects `/eval/thresholds.yaml`, `/eval/rubrics/`, and `/eval/runbooks/` behind a named team.
6. **Branch protection rule** on `main` making `eval-gate / gate` a required check. Screenshot or paste of the Settings page is acceptable evidence.
7. **Two demonstration PRs** against your working branch:
   - One that changes a prompt in a way you *expect to pass* — the gate posts a green report with all metrics passing and no blockers. Screenshot the PR comment.
   - One that changes a prompt in a way you *expect to fail* (e.g., strip the "cite the source documents" instruction). The gate posts a red report, the required check turns red, merge is blocked. Screenshot the PR comment and the check-blocked merge button.
8. **`README.md` addition** — a short "eval gate" section explaining how a developer runs the eval locally (`make eval-fast`), how to interpret the PR comment, how to add or update a fixture (points at exercise-02), and the override path (chapter 03's documented-rationale + auto-issue-follow-up).

## Starter guidance

- **Do not skip the sticky-comment.** The PR filling with N versions of the report is the single most common reason reviewers stop reading. Use `marocchino/sticky-pull-request-comment@v2` or an equivalent action; keep a stable `header:` value.
- **Do the paths filter carefully.** Include the workflow file itself in the paths filter — otherwise your own workflow edits do not trigger a run. Exclude `README.md` and other paths that cannot move a score.
- **Verify the required-check name matches.** GitHub matches required checks by exact string; typos here are why the merge button unblocks when it should not.
- **Do not paste the API key into logs.** Use `${{ secrets.EVAL_OPENAI_KEY }}` only via `env:`. Never build a curl call by string-concatenating a secret into a shell command.
- **Cache judge responses.** The dominant CI cost is judge model calls. Promptfoo's cache (`PROMPTFOO_CACHE_ENABLED`) or a similar layer for whatever runner you use is what keeps a re-run of the gate cheap.
- **Timeout is a smell detector.** If the gate is running longer than 15 minutes, either the fixture set is too big for the PR gate (move some fixtures to the deploy gate in exercise-05) or caching is misconfigured. Fix the smell; do not raise the timeout.
- **Report format.** The chapter-05 report has six named sections. Include all six — even a "no warnings" section if empty. Reviewers form muscle memory around section order.
- **Runbook is executable, not aspirational.** The runbook must reference specific filenames and commands. Test by handing it to a teammate cold and confirming they can execute step 1 without asking questions.

## Acceptance criteria

You are done when:

- The `pull_request` workflow runs on every PR against the paths in the filter, and does not run on paths outside it.
- The PR gets a **sticky** comment with the chapter-05 report format — header (pin fingerprint), summary table, top regressions, pin diff vs main, runbook links, warnings section.
- The check `eval-gate / gate` shows on the PR check list, is required by branch protection, and blocks merge on a blocking failure.
- The passing-PR demonstration shows a green report; the failing-PR demonstration shows a red report and a blocked merge.
- `apply_thresholds.py` produces the exact same report on a re-run against the same JSON inputs (deterministic). No timestamps or run IDs in the body other than the header.
- The judge-response cache reuse is observable — a second run of the gate on the same PR completes noticeably faster than the first.
- The runbook file for the demonstrated failing metric names the metric owner, the threshold justification, first step, common causes, and escalation. A teammate can execute the first step cold.
- No secret material appears in logs. `contents: read` is the workflow's write permission maximum (except `pull-requests: write` and `checks: write` where required).

## Stretch goals

- **Baseline diff via delta threshold.** In addition to the absolute threshold, add a delta-vs-main threshold on faithfulness (`>= -0.02`). Confirm the PR gate downloads the `eval-baseline` artefact, computes the delta correctly, and turns red on a delta regression even when the absolute score is still above the floor.
- **Sample-first-N for PR speed.** Split the replay set into `pr_sample.yaml` (a stratified 30-fixture subset) and `full_set.yaml`. The PR gate runs the sample; a separate `eval-nightly.yml` cron runs the full set against `main`. Confirm the PR gate report header logs which sub-sample it ran.
- **Fork-PR safety.** Implement the chapter-05 two-stage pattern for fork PRs: a `pull_request` job lint-only (no secrets), and a `workflow_run` job that runs the full gate against forks only after a maintainer approval. Confirm no secret leaks on a fork PR.
- **Override contract.** Add an `override-followup.yml` workflow that fires on merge with an `eval-override` label, auto-files a GitHub issue naming the eval-team CODEOWNER, and posts the override rationale to a dashboard channel. Wire the label add + rationale as a required PR field for the override path.
- **Port to GitLab CI or Buildkite.** Reproduce the same pattern on a second CI system your team also runs. The exercise's acceptance still applies — a required check that blocks merge, a sticky comment, the same report shape.
- **Pre-commit smoke.** Add a `pre-commit-config.yaml` hook running `make eval-fast` (10-fixture subset) locally on `git commit`. Document the `--no-verify` escape valve in the README.

## What this exercise does *not* cover

You are not authoring the full fixture pipeline (exercise-02), the Promptfoo config surface (exercise-03), the DeepEval alternative (exercise-04), or the deploy-side gate (exercise-05). You are wiring the pipe: workflow, report, check, override. Refinement on what flows through the pipe belongs in the next four exercises.
