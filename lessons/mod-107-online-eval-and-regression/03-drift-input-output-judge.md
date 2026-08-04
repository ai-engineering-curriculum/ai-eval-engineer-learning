# Drift Detection at Three Levels: Input, Output, and Judge

## Motivation

The loop from chapter 02 emits a stream of scored rows. A stream by itself is not a monitor — you need a signal that says "the world moved." That signal is **drift**: a statistically-significant departure of a distribution from its baseline. Getting drift wrong in either direction is expensive.

- Miss real drift and the on-call finds out from a support ticket, a Twitter thread, or an executive escalation — the exact failure mode the online loop is supposed to prevent.
- Fire on noise and the on-call learns to ignore the alert. Cry-wolf pager fatigue is the *most common* reason online-eval programs quietly stop working; nobody deletes the loop, they just start closing the tickets without reading.

Three drift sources need separate monitors because they need separate runbooks:

- **Input drift.** The users started asking different things. Nobody deployed a change to the model; the *distribution over queries* moved. Runbook: talk to product, check for a new referral source, check a locale change, check a seasonality event.
- **Output drift.** The model started answering differently. Could be vendor drift, could be a silent retrieval-index rebuild, could be a downstream cache change. Runbook: check `model_snapshot` and `retrieval_index_hash`; escalate to the change owner named on the cohort key.
- **Judge drift.** The judge itself started scoring differently. The model behind the judge got upgraded; the judge prompt got edited without a rehash; the calibration set went stale. Runbook: pin the judge snapshot; re-run the calibration; do not act on the score movement until you have ruled the judge out.

Confusing these — attributing an input-drift shift to a model regression, or a judge-drift shift to an output-drift alarm — is how eval programs make expensive rollback decisions on evidence the on-call cannot defend. This chapter walks the three monitors and the tests each one takes.

## Core concepts

### The reference distribution: what "baseline" actually means

Every drift monitor is a two-sample test between a **reference distribution** and a **current window**. Two choices for the reference define most of the monitor's character.

- **Fixed reference.** A frozen sample from a stable historical period — e.g., "the two weeks after the last production release that ran cleanly." The reference does not drift by construction. Test statistics are directly interpretable. Downside: the reference goes stale; anything the product does in the next quarter reads as drift.
- **Rolling reference.** A trailing window (e.g., the last 14 days, shifted daily). The reference tracks the ongoing trend, so seasonality and normal evolution do not fire. Downside: a slow multi-week regression can be invisible because the reference walks with the regression.

The mature pattern is **both, side by side**. The fixed reference is the *release baseline* — anchored to the last known-good deploy; picks up slow drift. The rolling reference is the *operational baseline* — picks up abrupt shifts. Chapter 04's canary gate consults the rolling reference; the drift monitor's weekly report consults the fixed. When the two disagree, the disagreement is itself a signal.

Regardless of choice, the reference distribution is a first-class artefact. It has a hash, a source-query, a captured-at timestamp, and a documented rebase policy. A drift monitor with an undocumented baseline is a drift monitor whose alert nobody can defend.

### Input drift: monitoring the query distribution

The **input** is what the user sent (or the upstream system built): the prompt, the retrieved context, the tool inputs, the metadata that varies with user population. Input drift is *not itself a quality regression*, but it is often the *cause* of one; treat it as a precursor signal.

Three shapes of input to monitor:

- **Categorical / low-cardinality attributes on the trace.** Locale, product surface, tenant tier, session-type, referrer. Test: **Population Stability Index (PSI)** or **chi-squared** on the frequency table. PSI is the industry-standard threshold-anchored version — bin the reference and current samples the same way, compute `Σ (p_i - q_i) · ln(p_i / q_i)`, and read against the conventional bands (< 0.1 stable, 0.1 – 0.25 moderate shift, > 0.25 significant shift). The bands are heuristic and *not universally correct*; anchor them per surface, not to the folk defaults.
- **Continuous scalar attributes.** Prompt length in tokens, retrieved-context length, number of tool-call turns per session. Test: **two-sample Kolmogorov–Smirnov** or **Cramér–von Mises**. Both are cheap and non-parametric. KS is the more common default.
- **High-dimensional attributes (the prompt text itself).** Embed the prompt (a small OSS embedding — e.g., `bge-small`, `all-MiniLM-L6-v2` — is enough) and monitor the embedding distribution. Test options: **Maximum Mean Discrepancy (MMD)** with a Gaussian kernel; a **classifier-based two-sample test** (train a binary classifier to distinguish reference from current, use its AUC as the test statistic — Lopez-Paz & Oquab); a **cluster-frequency PSI** (cluster the reference embeddings into `K` clusters, assign the current sample, PSI on the cluster frequencies). All three are covered in the Alibi Detect and NannyML documentation cited in `resources.md`.

Input drift is *directional*: some directions are benign (product opened a new tenant; sessions got longer because Copilot changed its output). The runbook for an input-drift alert must include *"is there a product event that explains this?"* as step 1. If the answer is yes, close the alert and rebase the reference to include the new period.

### Output drift: monitoring the response distribution

The **output** is the model's response, including its score under the online judge. Output drift is the signal the loop most cares about because it corresponds most directly to a user-facing quality change.

Three shapes to monitor:

- **The score distribution itself.** Per rubric, per cohort, mean / p05 / p95 over the sliding window. Test: **two-sample t-test on the mean**, **KS on the full distribution**, and a **direct threshold on the p05** (a p05 dropping below the pre-registered floor is a first-order signal — the score at the low tail is what a support ticket looks like). Chapter 05's confidence sequence replaces the single-shot t-test on this stream because the monitor is continuous.
- **Response-shape scalars.** Length of the response in tokens, number of citations produced, tool-call count per response, refusal rate, "I do not know" rate. Test: KS / PSI as above. A response that suddenly gets 40 % shorter with no rubric-score movement is *still* worth investigating — it could be a judge that stopped punishing terseness, not a quality improvement.
- **Response-embedding drift.** Embed the response and monitor with MMD or a classifier-based test, same as input embeddings. A response-embedding shift with the score distribution stable is a signal that *what the model is saying* changed but the judge did not detect it — a candidate judge-blind-spot investigation.

Output-drift attribution is the hard part. The scored-row schema (chapter 02) carries `model_snapshot`, `prompt_version`, `retrieval_index_hash`, and `flag_variant` on every row. When an output-drift alert fires, the first step of the runbook is to *slice* the rows by each of those keys and see which slice moved. A `model_snapshot` change that co-occurs with the drift is a vendor snapshot roll; a `retrieval_index_hash` change is a silent index rebuild; a `flag_variant` change is an experiment leaking beyond its intended cohort. The keys are attributions; without them, the alert is uninterpretable.

### Judge drift: monitoring the judge itself

The judge drifts too. Three sub-shapes; each has its own test and its own runbook.

- **Judge-model upgrade drift.** The vendor rolled the alias behind `gpt-4o` or `claude-3-5-sonnet-latest` to a new snapshot. The judge is now a different model. Rubric scores on identical inputs can shift by 0.05 – 0.20 on a rubric-shaped criterion just from the snapshot roll — this is not hypothetical, it has happened repeatedly across vendors. Detection: run a **rubric-fixed calibration set** through the judge on a schedule (daily is fine); alert when the mean score on the fixed set moves more than the rubric-hash-locked threshold (chapter 04 of mod-104 for the shape). The calibration set is a small fixed replay — 100 – 500 items — with human-authored reference scores. Its purpose is exactly this monitor.
- **Judge-prompt drift.** The judge prompt got edited (either intentionally, or by an unversioned string change). Detection: rubric hash on the scored row. The rubric hash *is* the version. Any change in the hash without a corresponding pre-registered rubric change is the drift signal. Chapter 04 of mod-104 wired the hashing; this chapter enforces it. Runbook: locate the commit that changed the rubric string; if intentional, re-calibrate against the reference set; if not, revert.
- **Judge-tier composition drift.** The tier router's `f_OSS / f_mid / f_frontier` moved because upstream escalation triggers moved, and the aggregate score moved because of the shift in tier composition (not because of a quality change). Detection: track per-tier score means separately; a shift in *aggregate* mean with per-tier means stable is the signal. Chapter 02's scored-row schema logs `judge_tier` for exactly this decomposition.

Judge-drift alerts must not trigger a rollback. The judge is measurement infrastructure; a measurement-layer alert calls for measurement-layer remediation (pin the judge snapshot, revert the rubric, re-calibrate), not a change to the product. Runbooks for judge drift end at the eval team, not at the on-call for the product surface.

### Choosing the test: a decision matrix

| Situation | Test | Why |
|---|---|---|
| Categorical or bucketed continuous attribute; want threshold-anchored monitor | **PSI** with fixed bins | Ubiquitous in industry; PSI bands are a shared vocabulary with data-science teams |
| Continuous scalar, two-sample, no threshold prior | **Two-sample KS** | Non-parametric; scale-free; cheap; low false-positive rate under reasonable window sizes |
| High-dimensional embedding | **MMD (Gaussian kernel)** or **classifier two-sample test** | The two flagship options for distribution-free multivariate two-sample testing |
| A moving mean on a continuous stream you are continuously monitoring | **Confidence sequence** (chapter 05) | The right tool because it is anytime-valid; fixed-`n` tests over-fire on continuous monitoring |
| A step change you want to detect at the moment it happens | **CUSUM** or **Page's test** | Change-point detection; sensitive to the moment of change, not the current-vs-baseline gap |
| A frequency-per-cohort question (e.g., refusal rate) | **Chi-squared** or **two-proportion z-test** | Standard for rate metrics; use the exact test if any cohort has small counts |

You do not need a novel test. You need one you can defend on the runbook. Pick from this matrix; do not invent.

### Thresholds and severity, not just test statistics

The test statistic is a number; the alert is a rule on the number. Three layers to pre-register (mod-106 chapter 03's threshold shape applies here — the online-loop thresholds file is the same file, extended):

- **Alert threshold.** Statistic value at which the drift monitor pages. Coupled to the false-positive rate you can absorb. A drift monitor on 10 metrics × 20 cohorts × 24 windows / day at α = 0.05 will fire ~2 400 times / day. Apply **Bonferroni** or **Benjamini–Hochberg (BH)** correction across metrics × cohorts × windows, or the loop is a pager-generator. BH is the more common choice at production scale because Bonferroni is over-conservative when the tests are correlated.
- **Severity.** `warn` vs `page` vs `auto-rollback`. Judge-drift on any tier: `warn` (never auto-rollback on judge-drift). Output-drift on a safety metric: `page` immediately, `auto-rollback` if the cohort is a canary. Input-drift by itself: `warn` (unless the runbook flags a specific product signal to combine with).
- **Cool-down.** Minimum time between re-fires on the same metric-cohort pair. Without a cool-down, one real regression pages 20 times as the window slides. 30 minutes is a reasonable default; per-metric override in the thresholds file.

### Segment discovery: finding the cohort that broke

Aggregate drift with no cohort attribution is a useless alert. Two patterns are worth knowing.

- **Pre-declared cohort slicing.** The cohort keys on the scored row (chapter 02's schema) are the pre-declared slices. The drift monitor runs *per cohort* and alerts *per cohort*; the aggregate is a rollup, not the primary monitor. Pre-declared slicing is the default because the runbooks per cohort can name owners.
- **Automatic segment discovery.** When aggregate drift fires but no pre-declared cohort explains it, run a lightweight segment-discovery pass — SliceFinder, DivExplorer, or an interaction-search over the cohort keys — to surface a candidate slice. This is *hypothesis-generating*, not test-of-significance; the segment surfaced is a candidate to declare as a cohort next cycle, not a proof it caused the drift.

Segment-discovery infrastructure is *not* on the critical path. Ship pre-declared cohort slicing first; add segment discovery once the on-call is comfortable with the alerts they already receive.

### Attribution, causation, and the temptation to over-explain

A drift alert has fired. The alert is not a *cause*; it is a *coincidence-in-time*. Two rules the runbook must enforce:

- **The alert states a distribution shift; the cause is investigated separately.** The runbook's first step is `slice by every cohort key on the row`. The second is `check the change log for the same window`. The third is `check the vendor status page and the model-snapshot log`. Only after those three does the runbook allow a rollback decision.
- **Correlation with a deploy is a hypothesis, not a conclusion.** Two changes shipped in the same 24-hour window; the drift correlates with both. A/B on the flag (chapter 06) is what discriminates. The on-call does not run a p-value in the middle of the incident — the on-call runs a rollback and files the discrimination as a follow-up.

The eval program's authority ends at *observation*; product and platform own *decisions*. This is the same posture as mod-106 chapter 06's rollback contract; the online loop's alerts are inputs to the same decision path.

### Vendor coverage

The tests above are library-level, not vendor-level. Two OSS libraries are the current defaults:

- **Alibi Detect** — MMD, classifier-based drift, KS, chi-squared, embedding-based tests. Preferred for numerical and embedding drift on Python trace-processing pipelines.
- **NannyML** — CBPE-style performance monitoring, MI-based drift, DLE. Preferred when you want a batteries-included dashboard on top of the same monitors.
- **Evidently AI (open source)** — a lot of pre-built drift-report shapes; the fastest way to a dashboard from a scored-row DataFrame.

Vendor trace backends (Phoenix, Langfuse, Weave, Braintrust) ship their own drift widgets. Use them if their default statistic matches what you want to defend on the runbook; wrap your own with one of the libraries above if it does not.

## Summary

- Drift is a **two-sample test between a reference distribution and a current window**. Baselines come in two shapes — **fixed release baseline** and **rolling operational baseline** — and mature loops run both.
- **Three drift levels, three monitors, three runbooks.** Input drift (query distribution), output drift (response distribution), judge drift (measurement itself).
- **Tests by shape**: PSI or chi-squared for categorical; KS or Cramér–von Mises for scalar; MMD or classifier-two-sample for embeddings; confidence sequences (chapter 05) for continuous monitoring; CUSUM for change-point.
- **Judge drift never triggers a rollback.** Judge is measurement infrastructure; the remediation is measurement-layer (pin the snapshot, revert the rubric, re-calibrate).
- **Multiple-testing correction** (Bonferroni or BH) is not optional. A monitor at α = 0.05 across metrics × cohorts × windows is a pager generator without it.
- **Cohort attribution matters more than aggregate signal.** The scored-row schema (chapter 02) carries `model_snapshot`, `prompt_version`, `retrieval_index_hash`, `flag_variant`. Slice by each before deciding.
- The alert is **observation, not causation**. Runbook slices by cohort keys and cross-checks the change log before allowing a rollback decision.

Chapter 04 uses the same score stream and the same cohort keys to gate canary and shadow launches, where the "current window" is the newly-shipped cohort and the "reference" is the currently-serving cohort.
