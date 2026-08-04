# exercise-02: Input, Output, and Judge Drift Alerts

**Estimated effort:** 3 hours

## Objective

Wire three drift monitors on top of the scored-row store from exercise-01, one for each drift level from chapter 03: **input**, **output**, and **judge**. Each monitor computes a two-sample test between a pre-declared reference distribution and a rolling current window, applies a pre-registered threshold, and — after multiple-testing correction across metrics × cohorts × windows — emits an alert with a runbook link and a cohort attribution.

By the end you will have a monitor that would catch the three real regressions the runbooks are written against: a locale-population shift (input), a silent model-snapshot roll (output), and a judge-alias upgrade (judge). Exercise-03 will consume the same infrastructure to gate a canary; exercise-04 will re-implement the output-drift monitor with an anytime-valid confidence sequence.

## Prerequisites

- Chapter 03 of this module.
- The scored-row store from exercise-01, populated with ≥ 7 days of data across ≥ 2 cohorts. Simulate if real coverage does not exist yet — the exercise's shape does not require real production traffic, only a store with the right schema.
- Python 3.11+, `numpy`, `scipy`, and one of `alibi-detect`, `nannyml`, or `evidently` for the drift tests. The reference implementations below assume `alibi-detect`.
- A small OSS embedding model for input-embedding drift (`bge-small-en-v1.5`, `all-MiniLM-L6-v2`, or vendor-native embeddings). Local inference is fine.
- A **fixed calibration set** for the judge-drift monitor — 100 – 500 items with human-authored reference scores per rubric. Reuse the mod-104 exercise-04 gold set.

## Set-up

1. Create `eval/online/drift/` next to `eval/online/`:

   ```
   eval/online/drift/
   ├── references/
   │   ├── input_prompt_len.parquet
   │   ├── input_locale_freq.parquet
   │   ├── input_embedding_ref.npy
   │   ├── output_faithfulness_ref.parquet
   │   ├── output_response_len_ref.parquet
   │   └── judge_calibration_gold.jsonl
   ├── monitors/
   │   ├── input_drift.py
   │   ├── output_drift.py
   │   └── judge_drift.py
   ├── config.yaml
   ├── correction.py
   ├── alerting.py
   └── runbooks/
       ├── input_drift.md
       ├── output_drift.md
       ├── judge_upgrade_drift.md
       ├── judge_prompt_drift.md
       └── judge_tier_composition_drift.md
   ```

2. Populate `references/` from your store: the fixed release baseline is the two weeks after your last known-good deploy; snapshot to a versioned file with a hash. The rolling reference is computed at read time from the last 14 days.
3. `config.yaml` declares per-metric per-monitor thresholds and correction policy:

   ```yaml
   input:
     locale_freq_psi:
       reference: fixed
       test: psi
       threshold_warn: 0.10
       threshold_page: 0.25
     prompt_len_ks:
       reference: rolling
       window: 24h
       test: ks_two_sample
       threshold_page: 0.05
     embedding_mmd:
       reference: fixed
       test: mmd_gaussian
       threshold_page: 0.02
   output:
     faithfulness_mean_ks:
       reference: rolling
       window: 24h
       test: ks_two_sample
       threshold_page: 0.05
     faithfulness_p05_floor:
       threshold_page: 0.60
     response_len_ks:
       reference: rolling
       window: 24h
       test: ks_two_sample
       threshold_warn: 0.05
   judge:
     calibration_mean_shift:
       reference: fixed
       threshold_page: 0.05
     rubric_hash_change:
       action: page_on_any_change
     tier_composition_shift:
       reference: rolling
       window: 24h
       threshold_warn: 0.10
   correction:
     method: benjamini_hochberg
     family_wide_fdr: 0.10
   ```

4. Write `references/README.md` describing every reference file's source query, capture date, hash, and rebase policy.

## Requirements

Produce a PR against your working branch that adds:

1. **`monitors/input_drift.py`** — three monitors:
   - PSI on `cohort_keys.locale` frequency against the fixed reference.
   - Two-sample KS on `prompt_token_len` (from the trace) against the 24-hour rolling reference.
   - MMD (Gaussian kernel) on the input-prompt embedding against a fixed-reference embedding set (~500 – 2 000 rows).

   Every alert emits `{monitor, metric, cohort, statistic, p_value, verdict, runbook_url}` and the raw slicing evidence.
2. **`monitors/output_drift.py`** — three monitors:
   - Two-sample KS on the rubric score distribution per cohort against the 24-hour rolling reference.
   - A direct floor test on the 24-hour rolling `p05` of each rubric.
   - Two-sample KS on `response_token_len` against the rolling reference (warn-level).

   The output-drift monitor's runbook step 1 slices by `cohort_keys.model_snapshot`, `prompt_version`, `retrieval_index_hash`, `flag_variant`.
3. **`monitors/judge_drift.py`** — three monitors:
   - **Calibration mean shift.** Runs daily; scores the calibration gold set with the current judge; alerts when the mean shift on any rubric exceeds the threshold.
   - **Rubric-hash change.** Watches for any change in `rubric_hash` on scored rows that was not accompanied by a pre-registered rubric change (compare against `eval/rubrics/CHANGELOG.md`).
   - **Tier-composition shift.** Two-sample PSI on the `judge_tier` frequency in the scored rows against a rolling reference; warns when composition drifts.
4. **`correction.py`** — Benjamini–Hochberg correction across the current metric × cohort × window family; returns the corrected reject-list. Unit test against a known-null simulation (all metrics from the same distribution) confirms the FDR is empirically ≤ the family-wide target.
5. **`alerting.py`** — emits pages for corrected `page`-severity alerts (PagerDuty stub or Slack incoming webhook) and warns for corrected `warn`-severity alerts (Slack). Cool-down per metric × cohort (default 30 min). Every payload includes the runbook link.
6. **Five runbooks** — each names the owner, the first step, the common causes (per chapter 03), and the escalation. `judge_*` runbooks explicitly forbid a product-side rollback as remediation; the remediation is measurement-layer (pin the judge snapshot, revert the rubric, re-calibrate).
7. **Two demonstration incidents** — synthesised to prove the monitors work end-to-end:
   - Inject an input drift: filter your input stream to a single locale for 2 hours; confirm the PSI monitor fires with the right cohort attribution and a page routed to the runbook.
   - Inject a judge drift: swap the judge to a nearby snapshot (or a different tier) for a calibration run; confirm the `calibration_mean_shift` monitor fires with the right rubric named and the judge-drift runbook linked.

   Screenshot both alerts.

## Starter guidance

- **Pick the reference deliberately, not by convenience.** Chapter 03's fixed vs rolling distinction is not a preference — it is a design decision. Locale frequency is a good fixed-reference target (rare deliberate change; usually a real signal on drift). Output score distributions do better on a rolling reference (normal within-week variation would flap a fixed reference).
- **Do not skip multiple-testing correction.** The naive "run 400 tests at α = 0.05 and alert on each" is *the* single most common cause of eval-loop pager fatigue. BH is not hard; ship it in the first version.
- **Judge drift never triggers a rollback.** State this in the runbook explicitly. When the on-call reads a judge-drift alert at 3 a.m., the runbook's first line should say "This is a measurement-layer alert. Do not roll back the product. Escalate to the eval team."
- **Slice by cohort keys first.** Every drift alert must include the cohort attribution in the payload. An aggregate alert without a slice is unactionable; the on-call spends the incident time slicing manually rather than acting.
- **Segment discovery is not on the critical path.** Ship pre-declared cohort slicing first (locales, tenant tiers, product surfaces you already have keys for). Add automatic segment discovery only if the aggregate alert repeatedly fires with no pre-declared cohort explaining it.
- **Do the daily judge-calibration job even if the online loop is stable.** The judge-upgrade drift is invisible until a snapshot rolls; the calibration job is the only way you learn about it before a whole quarter's readouts become suspect.
- **Wire the cool-down.** A firing metric on a sliding window re-fires as the window advances; without a per-metric cool-down, one real drift generates 20 pages.

## Acceptance criteria

You are done when:

- All three monitors run against the exercise-01 scored-row store on a schedule (cron / scheduled job) and produce the expected JSON payload shape.
- BH correction is applied across the current test family and the FDR-controlled reject-list is what fires alerts.
- The two synthesised incidents (locale filter, judge swap) each fire the expected monitor with the correct cohort attribution and the correct runbook link.
- No warn-only alerts route to PagerDuty; no page-severity alerts go silent.
- Every runbook contains: owner, first step, common causes, escalation. The three judge runbooks explicitly forbid product-side rollback.
- On a clean 24-hour period against a known-stable stream, the loop produces < 5 alerts (near-zero false-positive posture at the pre-registered thresholds).
- The `references/README.md` documents every reference file's source query, capture date, hash, and rebase policy.

## Stretch goals

- **Add a change-point detector (CUSUM / Page's test)** on the primary output metric; compare its time-to-detection to the KS-on-rolling-window monitor's on the synthesised judge-swap incident.
- **Automatic segment discovery.** Wire `SliceFinder` or `DivExplorer` against the last-24-hour scored rows; when the aggregate drift alert fires without a pre-declared cohort explanation, produce the top-3 candidate slices and post them to the alert Slack thread.
- **Judge-blind-spot detection.** Add an embedding-drift monitor on the response text with the rubric-score-drift monitor *stable*. A significant embedding shift with stable scores is a candidate judge blind spot; page the eval team (not the product on-call).
- **Two-vendor bake-off.** Run the same monitor on Alibi Detect and Evidently or NannyML; compare fire-rates on the same synthesised incidents; document which vendor's default test statistics best match your surface.
- **Weekly drift-review meeting artefact.** Emit a weekly digest (Markdown) of every drift alert in the previous week grouped by metric, cohort, and outcome (real / noise / suppressed). Feed this into the mod-109 human-review workflow.
- **Rebase-reference policy automation.** Ship a "rebase reference" workflow that pins the current 14-day window as the new fixed reference, requires a code review, and updates the reference-file hash and `README.md`. Test the workflow end-to-end.

## What this exercise does *not* cover

You are not building the canary gate (exercise-03), the anytime-valid confidence sequence (exercise-04), the A/B interface (exercise-05), or the dashboards (exercise-06). This exercise ships the drift monitors that the other exercises will re-use or contrast with.
