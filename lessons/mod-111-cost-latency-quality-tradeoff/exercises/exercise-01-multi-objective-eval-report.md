# exercise-01: Multi-Objective Eval Report

**Estimated effort:** 2 hours

## Objective

Design and ship the **chapter 02 multi-objective eval report** against a real (or realistic) shipping decision — pick between at least three candidate configurations for one product surface — and get product to sign off on the pre-registered rule before you see the numbers.

By the end of the exercise you will have a repo-committed report shape, a signed pre-registration file, a generated report page (Markdown or HTML), and a rollback trigger wired to the mod-107 online loop (stubbed if mod-107 is not in place). Every subsequent exercise's numbers land in this shape.

## Prerequisites

- Chapters 01 and 02 of this module.
- A trace store from mod-102 (Phoenix / Langfuse / Weave / Braintrust or plain OTel spans) with `feature.name`, `cohort.*`, and `gen_ai.*` attributes populated on at least one product surface's traffic.
- A rubric from mod-104 (any product-quality rubric — faithfulness, helpfulness, task-completion) with a stable hash.
- An eval set from mod-106 (a replay bundle) or a hand-authored fixture set of ≥ 100 cases with pre-declared cohort tags.
- The mod-110 platform slice tables (`eval_runs`, `eval_results`, `rubrics`, `eval_sets`) — or a scaled-down Postgres substitute with the same shapes. If mod-110 is not done, `eval_results` as a CSV + `pricing_snapshots` as a YAML is enough.
- API access to at least three model candidates you can score the eval set on (a frontier + a mid-tier + an OSS via a router, or three vendor models, or three prompts on the same model — any three configurations that produce meaningfully different quality / cost / latency numbers).
- Python 3.11+; a plotting library (`matplotlib` or `plotly`) for the Pareto small-multiples.

## Set-up

1. Create `eval/tradeoff/report/` in your repo:

   ```
   eval/tradeoff/report/
   ├── config.yaml
   ├── candidates/
   │   ├── inc.yaml                     # incumbent candidate definition
   │   ├── cand-a.yaml
   │   ├── cand-b.yaml
   │   └── cand-c.yaml
   ├── cohorts.yaml                     # pre-declared cohort roster
   ├── pre_registration/
   │   └── 2026-XX-XX-<surface>.md      # the signed rule
   ├── generate/
   │   ├── run_candidates.py            # score each candidate against the eval set
   │   ├── build_report.py              # assemble the report page
   │   ├── pareto.py                    # frontier + small-multiples plot
   │   └── decision.py                  # evaluate pre-registered rule against report
   ├── rollback/
   │   └── trigger.yaml                 # mod-107 online-loop trigger definition
   ├── output/
   │   └── <decision_id>/               # one directory per generated report
   │       ├── report.md
   │       ├── pareto.svg
   │       ├── cohort_matrix.csv
   │       └── report.json              # machine-readable form
   ├── tests/
   │   ├── test_dominance.py
   │   ├── test_pareto.py
   │   ├── test_decision_rule.py
   │   └── test_report_render.py
   └── README.md
   ```

2. In `config.yaml`, name the eval store, the surface, and the pricing source:

   ```yaml
   surface: support_bot
   eval_set: support-eval-v14        # hash pulled from mod-110 eval_set_current
   rubric:   support-faithfulness-v9 # hash pulled from mod-110 rubrics
   store:
     kind: postgres                  # or csv | duckdb
     dsn:  postgresql://eval:eval@localhost:5432/eval_platform
   pricing:
     snapshot: 2026-08-19
     path:     ./pricing/2026-08-19.yaml
   rollback:
     mod107_metric: online.faithfulness.mean
     cohort:        regulated
     window:        24h
     threshold:     -0.5
   ```

3. Pre-declare the cohorts in `cohorts.yaml`. The rule from chapter 02: every cohort a stakeholder would ask about after a regression. Between 4 and 10 cohorts. Example:

   ```yaml
   cohorts:
     - name: regulated
       predicate: tenant.compliance_tier == "regulated"
     - name: enterprise
       predicate: tenant.tier == "enterprise"
     - name: en-US
       predicate: request.locale == "en-US"
     - name: es
       predicate: request.locale == "es"
     - name: long-input
       predicate: input.token_count > 4000
     - name: short-input
       predicate: input.token_count <= 512
   ```

4. Define each candidate in `candidates/<id>.yaml`. Every candidate has a stable `candidate_id`, a full lineage-key set (model snapshot, prompt hash, chain hash, retriever index hash, decoding config, optional router policy, optional cache policy). The incumbent is one; the challengers are the rest.

5. **Write and sign the pre-registration file *before* running the candidates.** This step is the anti-motivated-reasoning contract. The file's shape (from chapter 02):

   ```markdown
   # Pre-registered decision rule — <surface> <decision_id>
   Written: 2026-XX-XX
   Signed:  PM=<name>, ENG=<name>

   Ship a candidate C if all of:
     1. quality on <critical_cohort> Δ vs. incumbent ≥ -0.5 rubric points
     2. no cohort's quality Δ vs. incumbent < -1.0 rubric points
     3. cost per request (mean) Δ vs. incumbent ≤ -30% (i.e., ≥ 30% drop)
     4. streaming p95 vs. incumbent Δ ≤ +200 ms
     5. zero OWASP LLM Top-10 category regression vs. incumbent

   Hold if any 1-5 is unmet but gap < 25% of the threshold. Reject otherwise.

   Tie-breakers (in order): cohort-preservation margin, operational simplicity,
   safety headroom, cost trajectory, latency headroom for roadmap growth.

   Rollback trigger:
     If mod-107 online-loop <metric> on <cohort> Δ vs. pre-ship baseline < -0.5
     over rolling 24h, auto-rollback via flag <name>.
   ```

   The file is committed and signed (Git commit-signed or a signed `SIGNATURES.md` sibling) before `run_candidates.py` runs. The report-generator refuses to build a report whose `decision_id` does not have a signed pre-registration file.

## Requirements

Produce a PR against your working branch that adds:

1. **`config.yaml`, `candidates/*.yaml`, `cohorts.yaml`** — the run's configuration; candidate lineage-key sets fully populated; cohort predicates parseable.
2. **`pre_registration/<decision_id>.md`** — the pre-registered decision rule, signed by product and engineering. The report generator asserts the file exists and is signed before running.
3. **`generate/run_candidates.py`** — scores every candidate against the eval set; writes rows to `eval_results` (or the CSV substitute); computes per-cohort quality means and worst-cohort values; records the pricing snapshot, model snapshot, and lineage keys per row. Idempotent (does not re-score a candidate whose lineage keys are unchanged).
4. **`generate/pareto.py`** — computes the Pareto frontier across the four axes; produces three 2D small-multiples plots (quality × cost, quality × latency, cost × latency) with each candidate labelled and the frontier highlighted. Safety-failing candidates are marked distinctly (red X) rather than dropped.
5. **`generate/build_report.py`** — assembles the report page (Markdown by default; optional HTML) with:
   - header block (decision id, timestamp, eval-set + hash, rubric + hash, candidate list, cohort roster, pre-registration link, signer line).
   - main table (per candidate: quality mean + worst-cohort, cost mean, TTFT p95 or streaming p95, safety pass/fail; Pareto members marked).
   - the Pareto small-multiples plot (embedded / linked).
   - the cohort quality matrix (per cohort × candidate, Δ vs. incumbent, δ_c pass/fail).
   - the decision block (pre-registered rule quoted verbatim, evaluated against numbers, result per candidate).
   - the rollback trigger stanza.
   - the signature line for product to countersign.
6. **`generate/decision.py`** — parses the pre-registered rule into a machine-executable form (five clauses × pass / fail per candidate); ships the ship / hold / reject decision per candidate; produces the same result whether re-run today or a month from now against the same inputs.
7. **`rollback/trigger.yaml`** — a mod-107-compatible trigger definition (metric, cohort, window, threshold, action). If mod-107 is not yet integrated, a stub that logs the decision and rings a webhook is acceptable. The report references the trigger by its file hash.
8. **`output/<decision_id>/`** — the generated report (`report.md`), plots (`pareto.svg`), cohort matrix (`cohort_matrix.csv`), machine-readable form (`report.json`). All files hashed; the report page includes a `report_hash` in the footer.
9. **`tests/*`** — unit tests for:
   - dominance / Pareto frontier (a synthetic 4-axis candidate set with a known frontier; the code returns the same set).
   - decision-rule parsing and evaluation (edge cases: a candidate with all-numbers-below-threshold; a candidate that gains on some axes and loses on others; a tied candidate).
   - report renderer (renders a fixture set to a stable Markdown string; renders the plot to a non-empty SVG).
10. **A demonstration run** on a real (or realistic) surface with ≥ 100 eval cases and 3 candidates. The generated report has:
    - at least one Pareto frontier that is not "the incumbent alone."
    - at least one non-critical cohort where a candidate wins and one where it loses.
    - a decision that is *not* "ship the highest-quality candidate" (i.e., the report earns its keep).
11. **`README.md`** in `eval/tradeoff/report/` — how to declare a decision, how to write a pre-registration, how to run the candidates, how to generate the report, how to sign it. A page-long walkthrough of the demonstration run.

## Starter guidance

- **Write the pre-registration first.** This is the exercise's central discipline. If the rule is written after the numbers are seen, the exercise's whole shape is compromised. Get product and engineering signatures on the rule file (or on the commit) before you run `run_candidates.py`.
- **Keep the report one page.** Everything the report readers need to make the decision is on the front page. The plot goes on the page; the raw per-case scores go into `output/<decision_id>/` as a supplement but not on the front page.
- **The Pareto view is three small plots, not one 3D plot.** 3D plots are unreadable. Use quality × cost, quality × latency, cost × latency in a row.
- **Ban `mean` alone in the report.** Every `mean` displays alongside a `p95` (or the worst-cohort value). This is the anti-average-quality discipline; enforce it in the report renderer.
- **The `report.json` matters more than the `report.md` for reproducibility.** The JSON is what future automation reads; the Markdown is what humans sign. Both are outputs.
- **Every candidate needs a `candidate_id`.** Not "the mid-tier model" — that changes when the vendor bumps the snapshot. `candidate_id = sha256(candidate.yaml)` is fine.
- **If mod-107 is not integrated yet, stub the rollback trigger.** A YAML that names the metric and threshold is enough for this exercise; the mod-107 exercise-04 wiring hooks it up for real.
- **Cohort predicates are simple boolean expressions.** Do not build a DSL; a small parser over `attr.op.value` is enough. The exercise is about the report shape, not a query engine.
- **Test the "safety-failing candidate is marked, not dropped" case.** A frequent bug is to drop safety-failing candidates entirely; the report has to show them and mark them excluded so the decision-reader knows they were considered.

## Acceptance criteria

You are done when:

- `pre_registration/<decision_id>.md` exists, is signed by both product and engineering, and is committed before `run_candidates.py` runs (verifiable via Git commit timestamps).
- Every candidate in `candidates/` has a full lineage-key set; two candidates with different lineage keys have different `candidate_id`s; two with identical lineage keys have identical ids.
- `run_candidates.py` produces `eval_results` rows for every case × candidate; every row carries the pricing snapshot hash, the rubric hash, and the model snapshot; the run is idempotent under re-execution.
- `pareto.py` returns the correct Pareto frontier on the synthetic test fixture; produces the three small-multiples plot with safety-failing candidates marked.
- `build_report.py` produces a report page whose one front page has the header, main table, Pareto plot, cohort matrix, decision block, and rollback stanza — and no `mean` displayed without a paired `p95` / `worst-cohort`.
- `decision.py` parses the pre-registered rule and produces the same ship / hold / reject verdict when re-run against the same inputs.
- `report.json` is produced and hashed; `report.md`'s footer contains the report hash.
- The demonstration run's report shows a non-trivial trade-off — at least one candidate is on the Pareto frontier that would not be chosen by any single-axis argmax.
- The tests in `tests/*` pass in CI.
- The `README.md` walkthrough is followable end-to-end by a reader who has not seen the exercise before.

## Stretch goals

- **HTML report.** In addition to the Markdown, render the report as HTML with an interactive Pareto plot (Plotly). The HTML is a supplement, not a replacement — the Markdown is what gets signed.
- **Report diff.** Given two report `report.json`s, produce a diff report that highlights which candidates moved on which axes and how the decision would have changed. Useful for re-running with a new candidate.
- **Sign-off notification.** Once the report is generated, post a summary (with the report link) to a Slack channel or email list; the signer replies with a countersign that closes the decision.
- **Historic report browser.** A small directory-listing over `output/*` that shows every past decision with its verdict and timestamp. The eval team's institutional memory over shipping decisions.
- **Sensitivity analysis.** For each pre-registered rule threshold, show how much the threshold would need to move for the decision to change. If a threshold's sensitivity is < 5 %, the report notes it — a tighter or looser rule would have flipped the decision.
- **Cohort-slice discovery.** Run a mod-107-style slice-finder over the per-case results to check whether any un-declared cohort has a materially different outcome; report as a supplementary section (not as a decision input). This catches cohorts the pre-declared roster missed.
- **Report signing hooks.** Wire the pre-registration and the final sign-off to Git commit-signing or a cryptographic signing service (Sigstore / DSSE). Every report artefact is verifiably tied to the two signers.

## What this exercise does *not* cover

You are not building the routing eval (that is exercise 02); the replacement-regression paired-comparison harness (exercise 03); the app-altitude latency measurement (exercise 04); or the per-feature cost accounting (exercise 05). You are shipping the *report shape and the decision discipline* — the shape every downstream exercise's numbers land in.
