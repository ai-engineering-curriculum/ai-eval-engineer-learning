# Wiring the PR Gate: GitHub Actions, the Eval Report, and the Check

## Motivation

You now have the fixtures (chapter 02), the thresholds (chapter 03), and the eval config (chapter 04). This chapter is the plumbing that turns those three artefacts into a **check on every pull request** — the property from chapter 01 called *runnable from CI*.

The gate is the same abstract shape on any CI system, but the concrete mechanics matter because they determine whether the reviewer actually engages with the eval work. A gate that runs but posts nothing to the PR gets ignored. A gate that posts a wall of raw JSON gets skipped. A gate that takes 45 minutes to run gets bypassed. A gate that runs on every push but re-runs the full eval on every commit burns money and provokes team pushback within a week.

This chapter walks the concrete pattern for GitHub Actions — the CI system most teams building LLM apps ship on today — and covers the small handful of details (report format, permissions, caching, artefacting, cost budget, path filters) that make the gate durable. The general pattern is portable: GitLab CI, Buildkite, CircleCI, Argo Workflows all have equivalent primitives, and the exercise names them.

## Core concepts

### The gate as a required GitHub check

The gate is a **GitHub Actions workflow** that runs on the `pull_request` event and reports a **check** back to the PR. The status of that check — success or failure — is what branch protection reads to allow or block the merge.

- The workflow lives in `.github/workflows/eval-gate.yml`.
- Branch protection on `main` (Settings → Branches → Branch protection rules) requires the check named `eval-gate / gate` to succeed before merge.
- The check status is written by the runner via the default GitHub API on failure of the job step. No custom app is needed.

This is the same shape a lint or unit-test check has; the eval gate deliberately looks like every other CI check the team already respects.

### Minimum viable workflow

```yaml
# .github/workflows/eval-gate.yml
name: eval-gate

on:
  pull_request:
    paths:
      - "prompts/**"
      - "chains/**"
      - "agents/**"
      - "tools/**"
      - "eval/**"
      - ".github/workflows/eval-gate.yml"

concurrency:
  group: eval-gate-${{ github.head_ref }}
  cancel-in-progress: true

jobs:
  gate:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    permissions:
      contents: read
      pull-requests: write   # to post the report as a PR comment
      checks: write          # to write the check-run detail

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with: {node-version: "20"}

      - uses: actions/setup-python@v5
        with: {python-version: "3.11"}

      - name: Install runtime + eval deps
        run: |
          pip install -e . -r eval/requirements.txt
          npm install -g promptfoo

      - name: Restore judge-response cache
        uses: actions/cache@v4
        with:
          path: eval/.judge-cache
          key: judge-cache-${{ hashFiles('eval/fixtures/**', 'eval/rubrics/**') }}

      - name: Run eval gate
        env:
          OPENAI_API_KEY: ${{ secrets.EVAL_OPENAI_KEY }}
          PROMPTFOO_CACHE_ENABLED: "true"
        run: |
          promptfoo eval \
            --config eval/promptfooconfig.yaml \
            --output eval/out/pr-eval.json

      - name: Apply thresholds
        run: |
          python eval/apply_thresholds.py \
            --scores eval/out/pr-eval.json \
            --thresholds eval/thresholds.yaml \
            --baseline-artifact-repo ${{ github.repository }} \
            --baseline-branch main \
            --report eval/out/pr-eval-report.md

      - name: Upload artefacts
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: pr-eval-report
          path: |
            eval/out/pr-eval.json
            eval/out/pr-eval-report.md
          retention-days: 30

      - name: Post PR comment
        if: always()
        uses: marocchino/sticky-pull-request-comment@v2
        with:
          path: eval/out/pr-eval-report.md
          header: eval-gate
```

Comments on the shape:

- **`paths` filter.** The gate does not re-run on a README change. Match the paths that can move the eval score.
- **`concurrency`.** Cancels in-flight runs on a new push to the same PR — avoids paying for the eval N times when a reviewer rebases.
- **`permissions`.** Explicitly scoped down. `pull-requests: write` for the PR comment; `checks: write` to expose per-metric detail on the checks tab.
- **`timeout-minutes: 15`.** A hard budget for the gate. If it starts exceeding this you have a fixture-set size problem (or a caching problem — see below).
- **Judge-response cache.** The dominant cost of a PR gate is the judge model calls, which are (mostly) deterministic on the same inputs — cache them. Chapter 04's Promptfoo config exposes `PROMPTFOO_CACHE_ENABLED`; DeepEval has a similar `deepeval cache` primitive. Cache-key on fixture + rubric hashes to invalidate correctly.
- **Sticky comment.** Update the same comment on every push instead of posting a new one — the PR does not fill with N versions of the same report.

### The PR report: what the reviewer actually reads

The Markdown report posted to the PR is where the gate meets the reviewer. This is where most teams under-invest and then complain that the gate is ignored.

Minimum contents, in order:

1. **Header.** Verdict banner (green / red), commit SHA, timestamp, gate config version, pin fingerprint (model + snapshot + `system_fingerprint`, index hash, judge model). This is what makes the run reproducible six weeks later.
2. **Summary table.** One row per metric, columns: current, baseline (`main`), delta, threshold, severity, verdict. Sorted by severity descending (blockers at the top). Emoji or colour is fine; the words `PASS` / `FAIL` are more important.
3. **Top regressions.** For every failed metric, the top 3–5 fixtures with the largest regression, with a link to a per-fixture drill-down (either an artefact URL or the vendor UI). Include the rubric rationale from the judge if available — mod-104's rubric shape carries one.
4. **Pin diff.** If any pin changed between `main` and the PR — a new model snapshot, a new judge model, a new index build — surface it explicitly. This is what saves the reviewer twenty minutes of confusion when a metric moved for a non-code reason.
5. **Runbook links.** Every failing metric links to its runbook (chapter 03). "Faithfulness regressed — see `eval/runbooks/faithfulness_mean.md`." Do not make the reviewer hunt.
6. **Warn-severity separate section.** Advisory-only findings do not clutter the main verdict; they get their own section labelled "warnings — no merge block".

Sample report skeleton:

```markdown
## Eval gate — FAIL

Commit: `abc1234` · Gate config version: `3` · Judge: `gpt-4o-2024-11-20` (fp `fp_a1b2c3`)
Model pin: `gpt-4o-2024-11-20` · Retrieval index hash: `sha256:d34db33f…`

### Metrics
| Metric | Current | Baseline | Δ | Threshold | Severity | Verdict |
|---|---|---|---|---|---|---|
| faithfulness_mean | 0.821 | 0.876 | -0.055 | Δ ≥ -0.02 | block | **FAIL** |
| faithfulness_p05 | 0.55 | 0.62 | -0.07 | ≥ 0.60 | block | **FAIL** |
| context_precision_mean | 0.71 | 0.73 | -0.02 | Δ ≥ -0.03 | warn | PASS |
| cost_per_query_delta | +2.1% | 0 | +2.1% | ≤ +5% | block | PASS |

### Top regressions (faithfulness_mean)
1. `fix-ticketing-002` — 0.90 → 0.20 (Δ -0.70) — judge rationale: "reply names an owner not present in retrieved documents"
2. `fix-onboarding-014` — 0.85 → 0.35 …

### Pin diff vs main
- Model: unchanged (`gpt-4o-2024-11-20`)
- Judge: unchanged
- Retrieval index hash: **CHANGED** (`sha256:d34d…` → `sha256:beef…`). Likely cause: index rebuild in this PR.

### Runbooks
- faithfulness_mean → `eval/runbooks/faithfulness_mean.md`
- faithfulness_p05 → `eval/runbooks/faithfulness_p05.md`
```

Every fact on the report is machine-produced from `pr-eval.json` + `thresholds.yaml` + the pin header. Nobody types the report.

### Handling secrets, budget, and rate limits

An eval gate calls model APIs — which cost money and are rate-limited. Four practical rules:

- **Use a dedicated API key with a low cap.** Do not share the production key. A key with a $10/day cap prevents a runaway loop from bankrupting the team.
- **Cap concurrency.** Promptfoo and DeepEval both let you cap parallel model calls; keep this low enough that a burst does not hit the vendor's rate limit. A common default is 4–8 concurrent calls for a small-team eval suite.
- **Cache aggressively.** The dominant cost is the judge model; the surface model is the second-largest. Both cache well because the prompts are (mostly) deterministic on the same fixture. Chapter 04's cache pattern.
- **Watch for secret leakage in logs.** GitHub Actions masks secrets registered via `${{ secrets.NAME }}`; it does not mask secrets built from other strings. Do not concatenate secret material into a log line.

### PR-gate vs push-to-main-gate

The PR gate runs on `pull_request` and blocks the merge. There is a second, less prominent gate that runs on `push` to `main` **after** the merge. Two reasons the second exists:

- The PR gate always compared against `main`'s baseline. Once merged, the baseline is *this commit* — the push job re-writes the baseline artefact for the next PR to compare against.
- Some large / long-tail suites (e.g., safety, adversarial) are too expensive for every PR and only run post-merge on a schedule.

Minimum shape for the baseline-writer:

```yaml
# .github/workflows/eval-baseline.yml
name: eval-baseline
on:
  push:
    branches: [main]
    paths: [prompts/**, chains/**, agents/**, tools/**, eval/**]
jobs:
  baseline:
    ...
    steps:
      - ...run eval, produce eval/out/main-eval.json...
      - name: Publish baseline
        uses: actions/upload-artifact@v4
        with:
          name: eval-baseline
          path: eval/out/main-eval.json
          retention-days: 90
```

The PR gate downloads this artefact (via `actions/download-artifact` scoped to the target branch, or via a small object-store contract if the team already has one) and computes the delta locally.

### The `.eval` shard is a first-class piece of the repo

A well-organised eval directory looks like:

```
eval/
├── fixtures/              # chapter 02
│   ├── fix-ticketing-002.yaml
│   └── ...
├── rubrics/               # mod-104
│   ├── faithfulness_v3.md
│   └── ...
├── runbooks/              # chapter 03
│   ├── faithfulness_mean.md
│   └── ...
├── calibration/           # mod-104 chapter 04
│   └── faithfulness_v3-2026-03-14.md
├── thresholds.yaml        # chapter 03
├── promptfooconfig.yaml   # chapter 04
├── apply_thresholds.py    # emit-then-check bridge
├── asserts/               # runner-specific shims (promptfoo js asserts, deepeval fixtures)
└── requirements.txt
```

- CODEOWNERS (chapter 03) protects the sub-directories.
- The PR gate workflow file lives in `.github/workflows/eval-gate.yml` and is itself under CODEOWNERS of the eval team.
- The directory is code-reviewable in the same PR that changes a prompt — the reviewer sees the fixture, the threshold, the runbook, and the prompt in one diff.

### The developer inner loop: `make eval` and the pre-commit smoke suite

The PR gate is the enforcement mechanism. It is not the developer's iteration surface — waiting 10 minutes on CI to know that a prompt tweak regressed one fixture is a terrible loop. Two escape valves:

- **`make eval-fast`.** A local subset (e.g., 10 fixtures) that runs in under a minute and covers the most common regressions. Add it to the README's "how to work on this repo" section.
- **Pre-commit smoke suite** (optional). Runs `make eval-fast` on `git commit`; skipping is `git commit --no-verify`. Not all teams accept a hook this expensive; the option matters more than the default.

The PR gate is the same thing at scale — the local subset is the fast preview, the CI run is the full contract.

### Path filters, fork-PR safety, and the sample-first-N pattern

Three edge cases worth mentioning:

- **`paths` filter tuning.** Too tight (only `prompts/**`) and a retriever change misses the gate. Too loose (`**`) and a README change runs the gate. Include every code path that can move the score; exclude every path that cannot.
- **Fork-PR secrets.** The default `pull_request` trigger on a fork-PR does not have access to secrets — the gate cannot run its API key. The `pull_request_target` trigger does have secret access but runs against the base commit and can be an injection vector. The safer pattern is a two-stage workflow: `pull_request` runs a lint that checks the fixture / threshold changes; a `workflow_run` job on the base repo, gated by a manual approval, runs the full gate against the fork. GitHub documents this pattern.
- **Sample-first-N for large replay sets.** If the replay set is > ~300 fixtures, run all fixtures on `push` to main and only a stratified sample (e.g., 60 fixtures with all incident-backfill and safety fixtures always included) on the PR. Chapter 06's deploy gate then runs the full set again. Log the sub-sample identity in the report header.

### Other CI systems: the primitive mapping

The pattern generalises. The primitives:

| Need | GitHub Actions | GitLab CI | Buildkite | CircleCI |
|---|---|---|---|---|
| Run on PR | `on: pull_request` | `only: [merge_requests]` | Trigger via GH webhook | `filters.branches` |
| Post PR comment | Sticky-comment action | `gitlab-mr-note` API | Buildkite → GH via app | CircleCI orb for GH |
| Required check | Branch protection | Merge-request approval rule | Manual required step | Required workflow |
| Cache | `actions/cache` | `cache:` | `cache-plugin` | `save_cache` / `restore_cache` |
| Secrets | `secrets.*` | Masked variables | Secrets manager | Contexts |

The exercise gives credit for any of these — the shape is the same, the syntax differs.

## Summary

- The gate is a workflow (GitHub Actions example above) that runs on `pull_request`, executes the eval config against the replay set, applies thresholds, posts a Markdown report to the PR, and reports a required check.
- The **report** is what the reviewer reads: header with pin fingerprint, summary table, top regressions, pin diff vs main, runbook links, separate warnings section. Every fact machine-produced.
- Practical rules: dedicated API key with cap, capped concurrency, aggressive judge-cache, `paths` filter, sticky PR comment, `concurrency: cancel-in-progress`, 15-minute time budget.
- A **second workflow on `push` to main** writes the baseline artefact the PR gate compares against.
- Provide a **developer inner loop** — `make eval-fast` on a subset — so the CI run is not the first place a regression is discovered.
- Handle three edge cases: `paths` filter tuning, fork-PR secret safety (`pull_request_target` alternative), sample-first-N for large replay sets.
- Pattern is CI-portable: GitLab, Buildkite, CircleCI all expose the same primitives.

Chapter 06 extends the gate past the PR: the **deploy gate**, the canary + shadow-eval pattern, and the interface to the deploy pipeline the platform team owns.
