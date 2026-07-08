# Offline, Online, and Deferred: Placing Each Criterion

## Motivation

Every criterion you extracted in chapter 02 lives in one of three places:

1. **Offline** — measured pre-merge and pre-deploy on a curated regression suite. Fast, cheap, deterministic.
2. **Online** — measured on sampled production traffic after deploy, on the surface as users actually use it.
3. **Deferred** — measured elsewhere, by someone else, and consumed as an interface. Typically the peer `model-evaluation-engineer` (statistical calibration methodology, benchmark construction depth) or the peer `ai-evaluation-engineer` (Governance, level 25 — audit / assurance shape).

Getting this triage right is the single largest driver of an eval program's cost, latency, and usefulness. Overload the offline suite and every PR takes an hour to merge. Overload the online loop and your judge bill dominates your infra bill. Fail to defer and you re-invent methodology owned by a peer. This chapter gives the decision rule and shows the trade-offs.

## Core concepts

### What offline is good at

An offline eval suite is a set of fixed **fixtures** (inputs) run against your surface, with pass/fail criteria enforced in CI. It is the right home for:

- **Regression protection.** Every prompt / chain / retriever / decoding change gets scored against known-good fixtures.
- **Attribution.** Because fixtures are fixed, a change in an SLI is *caused by* the code change under review.
- **Ship gates.** Merge-blocking and canary-blocking checks against pre-registered thresholds.
- **Reproducibility.** With pinned model version, seed, decoding, and fixture-hash you can rerun a failing case months later.

Offline suites are typically built from three sources:

- **Author-crafted fixtures** — cases you or the PM specifically want to hold the line on (a "golden set" per criterion family).
- **Snapshot-replays of production traces** ([mod-106 details this](../mod-106-eval-gated-cicd)) — real traffic scrubbed of PII, promoted into the regression suite.
- **Adversarial / adversarial-shaped fixtures** — from public research suites used as *shape templates* (e.g., AgentDojo, InjecAgent, HarmBench — see `mod-108`), not shipped verbatim in the repo.

### What offline is bad at

- **Coverage of the live distribution.** No matter how many fixtures you add, they will drift from what users actually type.
- **Detecting distribution shift.** By construction, fixtures do not shift.
- **Cost / latency measured under real load.** You must instrument a load path to get anything faithful here.
- **Novel jailbreaks and prompt-injection strings that appear post-deploy.**

### What online is good at

**Online eval** in this track means: sample a share of live production traces, run scorers over them (judges, rules, guardrail-check replays), and report the resulting SLIs against SLO thresholds. See `mod-107` for the full pattern.

Online is the right home for:

- **Live-distribution SLIs.** The share of *real* summaries that a judge scores ≥ 4 on faithfulness is the number the SLO actually cares about.
- **Drift detection.** Input drift, output drift, and judge drift are all inherently online phenomena.
- **Cost and latency under real load.** TTFT / TPOT / p95 measured on the sampling that matches user behaviour.
- **Novel failures.** The distribution-tail failures the offline suite could not enumerate.

### What online is bad at

- **Ship gates.** Online SLIs are lagging by definition; do not gate a merge on them.
- **Attribution across many changes.** If three PRs merged during the window, you need canary / shadow methodology (`mod-107`).
- **Cost.** Judge cost scales with sampling rate; sampling policy is part of the plan.

### What "deferred" means

Deferring is not skipping. It is a **named handoff** to the peer specialist who owns the depth. Two most common cases:

- **To `model-evaluation-engineer` (peer, level 30, ML Engineering family):**
  - Judge-vs-human calibration methodology (κ, Spearman, bootstrap CIs at methodology depth) beyond the quick calibration you do in `mod-104`.
  - Benchmark construction methodology when a new dataset is being built from scratch.
  - Statistical estimation for A/B experiments (CUPED and multiple-comparison correction depth).
  - MLPerf-style serving benchmarks — `mod-111` measures at product altitude; MLPerf-style methodology is deferred here.

- **To `ai-evaluation-engineer` (peer, level 25, Governance family):**
  - Audit-trail packaging.
  - Regulator-facing evidence documents (EU AI Act GPAI obligations, NIST AI RMF Manage documentation).
  - Third-party evaluator interfaces.

- **To `ai-risk-engineer` (peer, level 30):**
  - Harm-model authoring, red-team dataset generation methodology. `mod-108` in this track *measures* on the surface; the harm-model design is upstream.

A criterion you defer still shows up in your two-page eval plan (chapter 06), under a "Deferred to" section, with the peer role and the interface you rely on.

### The decision rule

For each criterion, walk this checklist in order:

```
1. Is this a criterion you should not measure yourself?
     ↳ if it needs methodology depth owned by a peer track: DEFER.

2. Does a change under review need to pass or fail on this criterion?
     ↳ if yes: OFFLINE. Build a fixture with a pre-registered threshold.

3. Is the criterion only meaningful on the live distribution?
     ↳ if yes (drift, novel failures, cost/latency-under-load): ONLINE.

4. Both? Put a coarse gate offline and a finer SLI online.
     ↳ this is the common answer.
```

### Placing the five families

A useful default, per family:

| Family | Offline (regression) | Online (sampled trace) | Deferred |
|---|---|---|---|
| Task success | Canonical golden + snapshot replays | Sampled-trace judge / rule score | Methodology-grade judge calibration (→ `model-evaluation-engineer`) |
| Groundedness | RAG-triad on fixed corpus | RAG-triad on sampled traces | RAG methodology depth (→ `rag-engineer` peer for retrieval build depth) |
| Safety | Refusal & injection fixtures with pre-registered pass rate | Guardrail hit rate, novel-attack detection | Harm-model authoring (→ `ai-risk-engineer`); regulator packaging (→ Governance peer) |
| Cost | Cost budget on the fixture set | $/trace on sampled traffic | Serving-cost benchmark methodology (→ `model-evaluation-engineer`) |
| Latency | Local timing test on fixed inputs | TTFT / TPOT / p95 on real traffic | MLPerf-style methodology (→ `model-evaluation-engineer`) |

Copy this default into your first plan. Diverge on purpose, not on accident.

## Example: triage for the meeting summariser

Continuing the summariser surface from chapter 02:

| Criterion | Offline | Online | Deferred |
|---|---|---|---|
| Summary length ≈ 200 words | Deterministic character/word check on golden set | Sampled length distribution | — |
| Action-item attribution correct | Judge score on 100-case golden set (merge gate ≥ 98%) | Judge score on ≥ 5% sampled traces (SLO ≥ 95% weekly) | Full judge-vs-human calibration → `model-evaluation-engineer` |
| No HR-content leak | Rule + judge check on HR-seeded fixture (hard gate 0 leaks) | Guardrail hit rate on live traffic + spot review | Harm model for "sensitive HR" content → `ai-risk-engineer` |
| Prompt-injection robustness via calendar-invite text | AgentDojo-shape fixtures (`mod-108`) | Novel-injection detection on sampled traces | Full injection-red-team dataset methodology → `ai-risk-engineer` |
| p95 end-to-end < 60s (well inside the 5-minute promise) | Cold-path timing test | TTFT/TPOT/p95 on live streaming | Serving-cost & MLPerf-shape methodology → `model-evaluation-engineer` |
| Cost / summary within budget | Cost calc on golden set at deploy config | $/summary on sampled traffic | — |
| Regeneration usage ≤ 2 per session | — | Sampled counter of `regenerate` events | Product analytics methodology (→ product analytics team) |

Two things worth noticing:

- The 200-word check is offline-only because it is a deterministic rule, cheap and stable.
- Prompt-injection robustness has both an offline fixture and an online watch, because novel attack strings appear post-deploy and the fixture cannot enumerate them.

## Common traps

- **Judge-in-the-offline-loop cost explosion.** Every judge call costs money; a merge-gate loop that runs 5,000 judge calls per PR will bankrupt the project. Keep offline fixtures small and pointed; grow them from *observed* regressions, not from imagined ones.
- **Sampling the online loop uniformly at random.** Random sampling misses the tail. Stratify by cohort, by feature flag, by traffic source. Sampling policy design is deep in `mod-107`; state your default here (usually stratified, per-family cap).
- **Deferring by silence.** A criterion you did not measure and did not mention is a governance hole. If a peer owns it, *say so* in the plan.

## Summary

- Every criterion is **offline**, **online**, or **deferred**. Nothing is unassigned.
- Offline owns **regression protection, attribution, and ship gates.**
- Online owns **live-distribution SLIs, drift detection, and cost/latency-under-load.**
- Deferred owns **peer-specialist methodology depth** and is *named*, not silent.
- Follow the four-step decision rule; use the per-family default table as your starting point.

The next chapter loads two external checklists — ISO/IEC 25059 and NIST AI RMF's Measure function — so you can systematically catch families of criteria the spec forgot to mention.
