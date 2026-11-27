# quiz-01: LLM-as-Judge in Product Pipelines

**Estimated time:** 15–20 minutes
**Covers:** chapters 01–05 of this module.

Answer each question, then check the key at the bottom. Explanations cite the chapter the answer comes from so you can re-read the relevant section if an answer surprised you.

---

## Questions

### 1. Chapter 01 draws a sharp line between the two rubric shapes. A team is deciding whether to flip from prompt template `v12` to `v13`, where internal spot-checks suggest `v13` is only *slightly* better. Which rubric shape does the chapter prescribe, and why?

- A. Absolute, because `v13` needs a number on the dashboard.
- B. Pairwise, because when the quality difference between candidates is small compared to the scale noise, pairwise sidesteps the "what does 3 vs 4 actually mean" argument.
- C. A weighted mean of absolute and pairwise.
- D. Either works; pick whichever the framework defaults to.

### 2. Chapter 01's absolute rubric has **six** parts, each removing a specific class of ambiguity. Which of the following is *not* one of them (and would be a chapter-02 concern instead)?

- A. Criterion name and one-line definition.
- B. Scale and levels with written anchors per level.
- C. Evidence pointer naming the exact mod-102 spans and attributes the judge reads.
- D. Position-bias swap-and-average aggregation rule.

### 3. Chapter 01 calls anchor design the "single largest lever on judge reproducibility" and lists three anchor failure modes. Which triple matches the chapter?

- A. Too many levels; too few levels; integer-only scales.
- B. Vague level descriptions ("good", "excellent"); **non-monotone** levels (level 4 introduces a criterion level 3 did not mention); anchors that reference evidence the judge prompt does not materialise.
- C. Anchors in Markdown; anchors in YAML; anchors in JSON.
- D. Anchors longer than 20 words; anchors shorter than 5 words; anchors that cite external URLs.

### 4. Chapter 02 prescribes **swap-and-average** for position bias. For a pairwise comparison, the judge is run twice with candidates in swapped positions. Which aggregation rule matches the chapter?

- A. Report whichever winner the first run returned; use the second run only as a tie-breaker.
- B. Average the two outputs numerically and round.
- C. Both runs pick the same non-tie candidate → report that winner; the two runs disagree → report `tie`; either run returned a tie → report `tie` (or the winner of the non-tie run, per a stated policy).
- D. Majority of three re-runs; two of three must agree.

### 5. Chapter 02 names two mainstream length-bias mitigations ("often use both"). Which pair matches the chapter, and which approach does the chapter **explicitly reject**?

- A. Pair: length-normalised post-hoc scoring (logistic regression on length difference, à la AlpacaEval length-controlled) + a length constraint anchored in the rubric itself. Rejected: truncating candidates to a fixed length before judging (loses signal, hides real differences in the last paragraph).
- B. Pair: majority-vote + temperature-0 sampling. Rejected: logit-weighted scoring.
- C. Pair: fixed-length truncation + a verbosity-penalty prompt line. Rejected: length-adjusted post-hoc regression.
- D. Pair: swap-and-average + cross-family diagnostics. Rejected: recording `app.judge_candidate_len`.

### 6. Chapter 02 is explicit that **self-preference / self-enhancement bias** does not have a cheap unbiased mitigation. Which is the chapter's recommended *diagnostic* (at least once per quarter)?

- A. Up-weight self-generated candidates to counteract the bias numerically.
- B. Re-judge a stratified sample of ~100 recorded judgements with a **different-family** judge, compute the cross-judge agreement rate, and alarm when it drops below the calibration-set inter-rater threshold.
- C. Fine-tune the primary judge on a sample of its own disagreements.
- D. Instruct the judge in prose to "not prefer models that write like you".

### 7. Chapter 03 defines **G-Eval** as a three-part method the mainstream frameworks operationalise. Which triple matches?

- A. Retrieval-augmented reasoning, agent-style tool use, pairwise ranking.
- B. (1) Chain-of-thought evaluation reasoning before the score; (2) form-filling structured output (score + rationale); (3) probability-weighted score aggregation over the discrete scale when logits are available.
- C. (1) Prompt hardening; (2) swap-and-average; (3) length normalisation.
- D. (1) Fine-tune on preference data; (2) RLHF against human votes; (3) DPO on pairwise wins.

### 8. A team's regression suite is prompt-versioned YAML, gated in CI with no Python glue, and wants a rubric to fail the build on a numeric threshold. Chapter 03 points them at which framework?

- A. DeepEval's `GEval` metric (pytest-native).
- B. Braintrust's `autoevals` library (eval-first data model, Experiments).
- C. Promptfoo's `llm-rubric` assertion (CLI + YAML, pass/fail with rationale).
- D. A hand-rolled Python script because none of the three support CI.

### 9. Chapter 03 names **three** real reasons to switch from a frontier-API judge to a dedicated open-source judge (Prometheus 2 / JudgeLM). Which set matches?

- A. Cost at scale, reproducibility across model versions, data residency / privacy.
- B. Better human agreement out of the box than frontier judges on every rubric.
- C. Shorter rubric prompts, lower prompt-token count, higher max-output-tokens.
- D. Open-source alignment research, academic publishing, benchmark leaderboards.

### 10. Chapter 04 gives a product-altitude target for calibration gold-set size. A new rubric on a new surface has no labelled data yet. Which plan is consistent with the chapter?

- A. Label 10 items; call it done.
- B. Label ~50–100 items, stratified by rubric outcome level, surface sub-class, and known failure modes; report the statistic with a bootstrap 95% CI and a per-stratum breakdown.
- C. Label 500 items unstratified, since more is always better.
- D. Skip calibration; let the online-eval loop produce the labels passively.

### 11. Chapter 04 gives a statistic-per-rubric-shape table. Which pairing matches the chapter?

- A. Nominal → Pearson r; ordinal 1–5 → Cohen's κ; pairwise A/B/tie → Kendall τ.
- B. Nominal → **Cohen's κ** (chance-corrected categorical agreement); ordinal 1–5 → **Spearman ρ** or **Kendall τ** on paired scores (treat 4-vs-3 as closer than 4-vs-1); pairwise A/B/tie → **three-class κ** over winner labels, with **non-tie agreement** as the secondary number the release gate actually cares about.
- C. All rubric shapes → accuracy (count of exact matches ÷ N).
- D. All rubric shapes → F1 over the "winner" label.

### 12. Chapter 04 names five conditions that should push the calibration to the **`model-evaluation-engineer` peer** rather than keep it at product altitude. Which of the following is *not* in the chapter's escalation list?

- A. High-stakes rubric that needs a defensible **variance decomposition** (judge noise vs rater disagreement vs item ambiguity).
- B. **Statistical power analysis** — "how big does the gold set need to be to detect a 3-point regression at α=0.05, β=0.2".
- C. The judge disagreed with a human labeller on a single item and the team wants the item re-labelled.
- D. **Combining multiple raters** (LLM + human panel + a second LLM) with a defensible aggregation rule (weighted κ, Fleiss's κ, Krippendorff's α).

### 13. Chapter 05 describes a **three-tier** router with per-tier defaults. Which routing of decisions matches the chapter's rule?

- A. Default *every* judgement to the frontier tier; downgrade only when the per-month bill is exceeded.
- B. Default routine online-eval traffic to the **OSS** tier; **escalate to the mid-tier** when the OSS judge is uncertain (a pairwise `tie`, a 3-on-a-1–5 scale, or a failed self-consistency check); **reserve the frontier** tier for release-gate A/B and periodic re-scoring of the calibration set.
- C. Alternate tiers round-robin regardless of the decision.
- D. Route by trace id modulo 3; let the sampler decide.

### 14. Chapter 05 names **four** independent sources of drift that affect a judge in production, but only two of them are "true" judge drift; the other two are population / generator drift that merely *shows up* in judge metrics. Which pairing is correct?

- A. True judge drift: **judge model version**, **rubric prompt template**. Population / generator drift: **input distribution**, **generator model**.
- B. True judge drift: input distribution, generator model. Population: judge model version, rubric prompt.
- C. All four are judge drift; the distinction is irrelevant.
- D. Only the judge model version counts; the other three are noise.

### 15. Chapter 05 gives a five-step **frontier-upgrade playbook** for when a vendor ships a new snapshot. Which ordered sequence matches?

- A. Swap immediately in production; re-calibrate only if the online-eval dashboard moves by > 10%.
- B. Pin the current snapshot → run the new judge on both the frozen replay set and the calibration gold set side-by-side with the incumbent → re-calibrate against the human labels and publish the new statistic → swap production judge only after the new-judge calibration is documented and the drift-monitor alert threshold is reset → keep the incumbent available for at least one drift-monitor cycle for one-line rollback.
- C. Delete the incumbent snapshot from your framework config and migrate the drift monitor before touching the production pipeline.
- D. Freeze all releases for a quarter until the vendor publishes an official migration note.

---

## Answer key

1. **B.** Chapter 01's "rules of thumb" prefer pairwise for release-gate A/B precisely when the difference between candidates is small compared to the absolute-scale noise — it sidesteps "what does 3 vs 4 actually mean." Absolute is preferred for continuous online monitoring (option A conflates the two cases).
2. **D.** Chapter 01's six-part absolute rubric is: criterion + definition, scale + levels, anchors per level, evidence pointer, tie/abstain policy, output schema. Position-bias mitigation is a *pipeline* concern wrapped around the rubric, not part of the rubric itself — chapter 02 handles it.
3. **B.** Chapter 01's "Anchor design" section names exactly these three: vague level descriptions, non-monotone levels, and anchors that reference evidence the judge prompt does not materialise. The chapter's fix for non-monotonicity is to write levels as a *chain of relaxations* from 5 downward.
4. **C.** Chapter 02's swap-and-average aggregation rule is exactly this: agreement on a non-tie winner → report it; disagreement collapses to `tie` (the disagreement is the position-bias signal); either run tying → tie (or the non-tie winner per a stated policy — the chapter says "pick one and stick with it").
5. **A.** Chapter 02 names post-hoc **length-normalised scoring** (logistic regression on the length difference, as in Dubois et al.'s length-controlled AlpacaEval) and a **rubric-level length constraint** as the two mainstream mitigations, and explicitly rejects **truncating candidates to a fixed length** ("truncation loses signal, and it will hide real quality differences that live in the last paragraph").
6. **B.** Chapter 02's cross-family diagnostic is the chapter's canonical recommendation: periodically re-judge a stratified ~100-item sample with a different-family judge and track the agreement rate as a first-class metric. Option D is explicitly called out as a failing approach (Zheng et al. showed position-style instructions in prose do not remove the bias; the same logic applies to self-preference).
7. **B.** Chapter 03's G-Eval section describes exactly these three components — chain-of-thought first, form-filling output, then probability-weighted score aggregation (`sum(i * P(i))`) when logits are available. The chapter notes most product-side implementations ship (1) and (2) and skip (3) because frontier APIs do not always expose logits.
8. **C.** Chapter 03's framework-selection section maps "regression suite is prompt-versioned YAML, gated in CI, no Python glue needed" → Promptfoo's `llm-rubric`. DeepEval is the pytest-native Python pick; Braintrust is the eval-first `autoevals` pick.
9. **A.** Chapter 03's "When to switch to a dedicated open-source judge" section names exactly three reasons: **cost** at scale (marginal per-call cost of frontier vs fixed GPU cost of self-hosted OSS), **reproducibility** across quarters (a pinned OSS judge is the only defensible answer when frontier snapshots drift), and **data residency / privacy** (traces that cannot leave a VPC).
10. **B.** Chapter 04's "How big does the gold set have to be" section targets ~50–100 items stratified by rubric outcome level, surface sub-class, and failure-mode categories. Below ~30 is spot-check territory; above ~200–300 is peer-role territory. Report with a bootstrap 95% CI and per-stratum breakdown.
11. **B.** Chapter 04's statistic table pairs nominal → Cohen's κ (or Fleiss for >2 raters), ordinal → Spearman ρ / Kendall τ (treat ranks as ordinal), and pairwise → three-class κ on winner labels with non-tie agreement as a secondary number the release gate ultimately cares about. Pearson/accuracy/F1 are not the chapter's picks.
12. **C.** Chapter 04's escalation list names: variance decomposition, statistical power analysis, unequal-interval ordinal analysis, multi-rater aggregation rules (weighted κ, Fleiss, Krippendorff), and non-standard resampling schemes (clustered / hierarchical bootstrap). A single-item disagreement is normal rubric iteration, not peer-role work — the chapter is explicit that the response to a bad calibration is not "label more items" but fix the rubric, pipeline, or judge tier (then escalate if that still is not enough).
13. **B.** Chapter 05's routing rule is exactly this: OSS by default for routine traffic, mid-tier on OSS uncertainty, frontier reserved for release-gate and calibration re-scoring. The routing is per-criterion, not per-surface — one trace can have groundedness judged by OSS and safety judged by frontier.
14. **A.** Chapter 05's "Judge drift: what changes underneath you" section names four sources. Only (1) **judge model version** and (2) **rubric prompt template** are *judge* drift; (3) input distribution and (4) generator model are population / generator drift that merely surfaces in judge metrics. The monitor must distinguish them so the on-call does not re-calibrate when the right fix is "refresh the calibration set".
15. **B.** Chapter 05's "frontier-upgrade playbook" lists this exact five-step sequence: pin → side-by-side on frozen sets → re-calibrate and publish → swap after documenting → keep the incumbent available for one drift-monitor cycle (for one-line rollback). The chapter's framing: "two or three days of playbook per frontier upgrade is cheap compared to a silent quality regression caught only after a bad release."
