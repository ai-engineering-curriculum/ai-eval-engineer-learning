# mod-104 — LLM-as-Judge in Product Pipelines: Rubrics, Bias Controls, and Judge-Tier Routing

**Estimated effort:** 14 hours

This is the fourth module of the AI Evaluation Engineer track. It turns the mod-101 criteria table and the mod-102 trace shape into a running, calibrated, cost-managed **judge pipeline** — the artefact that produces the per-request quality scores mod-105 (RAG-triad on the app layer), mod-106 (CI gate on the offline suite), mod-107 (online-eval loop with sampled traces), mod-108 (safety and guardrail scoring), mod-109 (human review of judge disagreements), mod-110 (judge lineage in the eval-data platform), mod-111 (judge cost inside the product cost model), and mod-112 (release-gate architecture) all consume. Without a well-formed judge, the rest of the track is scoring noise.

The module scopes to *product-side* judge work — designing rubrics that reflect real product criteria, controlling the biases documented in the LLM-as-judge literature, wiring the mainstream frameworks (DeepEval / Braintrust / Promptfoo) plus the OSS-judge fallback (Prometheus 2 / JudgeLM), calibrating quickly against a small human gold set, and routing decisions across judge tiers under a real cost budget. Full methodology depth — variance decomposition, statistical power analysis, unequal-interval ordinal analysis, non-standard resampling — is deferred to the `model-evaluation-engineer` peer (Model-Development family, level 30); chapter 04 is the escalation contract.

## Learning objectives

- Design absolute (single-answer) and pairwise rubrics for real product criteria with explicit scoring anchors.
- Apply position-bias controls (swap-and-average), length-bias controls (length-normalised scoring), and self-preference diagnostics inside the judge pipeline.
- Use G-Eval-style chain-of-thought rubrics with DeepEval / Braintrust / Promptfoo and know when to switch to a dedicated open-source judge (Prometheus 2 / JudgeLM) for cost or reproducibility.
- Quick-calibrate a judge against a small human gold set with a defensible statistic and know when to escalate to the `model-evaluation-engineer` peer for full methodology depth (Cohen's kappa / Spearman / Kendall / bootstrap CIs).
- Route judgements across judge tiers (frontier API / mid-tier / open-source) inside a cost budget and monitor for judge drift when the judge model itself upgrades.

## Lecture chapters

1. [Designing absolute and pairwise rubrics for real product surfaces](01-rubric-design.md) — the six-part absolute rubric and five-part pairwise rubric; anchor design as the largest lever on reproducibility; grounding rubrics in mod-101 criteria rather than benchmark rubrics; trajectory vs final-output rubrics; the end-to-end judge prompt template.
2. [Judge biases and pipeline controls: position, length, self-preference](02-judge-bias-controls.md) — the three documented biases; swap-and-average as the canonical position-bias control; length-normalised / length-controlled scoring; cross-family diagnostic for self-preference; wiring controls at the runtime boundary; the three diagnostic metrics.
3. [G-Eval, framework-based judges, and open-source judges](03-g-eval-frameworks-and-oss-judges.md) — chain-of-thought + form-filling + weighted decoding; DeepEval / Braintrust / Promptfoo trade-offs; when to switch to Prometheus 2 / JudgeLM for cost, reproducibility, or data residency; wiring OSS judges behind the frameworks; cross-cutting rules.
4. [Quick-calibrating a judge against a small human gold set](04-quick-calibration-against-gold.md) — scoped calibration; gold-set sizing and stratification; the right statistic per rubric shape (κ / ρ / τ / non-tie agreement); bootstrap CIs; the escalation contract with the `model-evaluation-engineer` peer; the calibration document.
5. [Judge-tier routing and drift monitoring](05-tier-routing-and-drift.md) — the three tiers and the routing rule; per-tier cost budget as an SLO; the four sources of judge drift; the drift-monitor pattern; re-calibration triggers; the frontier-upgrade playbook.

## Exercises

Each exercise builds on the last. Do them in order and keep the outputs — the artefacts (rubrics, bias-controlled runtime, framework bake-off, calibration doc, router + drift monitor) are the deliverables mod-107 and mod-112 will assume you have.

1. [Absolute and pairwise rubrics for a product surface](exercises/exercise-01-absolute-and-pairwise-rubric-for-product-surface.md) — write and hand-test two rubrics against your mod-101 criteria table.
2. [Position, length, and self-preference bias controls in the pipeline](exercises/exercise-02-position-and-length-bias-controls-in-pipeline.md) — build the swap-and-average runtime, length-adjust scoring, and cross-family diagnostic; report the residual biases honestly.
3. [G-Eval with DeepEval and Braintrust (plus a Promptfoo comparison)](exercises/exercise-03-g-eval-with-deepeval-and-braintrust.md) — implement one rubric in three frameworks; write the bake-off note; make a defensible pick.
4. [Quick human-gold calibration](exercises/exercise-04-quick-human-gold-calibration.md) — build a stratified 50–100-item gold set, compute the right statistic with a bootstrap CI, and decide whether to escalate to peer.
5. [Judge-tier routing and drift monitor](exercises/exercise-05-judge-tier-routing-and-drift-monitor.md) — wire the tier router, set the cost cap, pin a frozen replay set, write the drift monitor and the on-call runbook.

## Labs and quizzes

- Labs (see [`labs/`](labs)) build the end-to-end judge pipeline that mod-107's online-eval loop consumes and that mod-106's CI gate reads at merge time. Authored under the autonomous fill-in loop.
- Quizzes (see [`quizzes/`](quizzes)) verify the vocabulary — rubric anatomy, the three biases and their canonical mitigations, the calibration-vs-escalation decision, and the drift-triage checklist. Authored under the autonomous fill-in loop.

## Resources

External references are curated in [`resources.md`](resources.md).

## Where this module hands off

- **RAG-triad application-layer eval** consumes this module's rubrics (faithfulness / answer-relevancy / context-precision / context-recall are all judge-scored) → [`mod-105-rag-eval-app-layer`](../mod-105-rag-eval-app-layer).
- **CI-gated eval** wires the frontier-tier release-gate judge into the merge and canary gates → [`mod-106-eval-gated-cicd`](../mod-106-eval-gated-cicd).
- **Online-eval and regression** runs the OSS-tier default on sampled traces, escalates uncertain ones to the mid tier, and pages on the drift-monitor alerts from chapter 05 → [`mod-107-online-eval-and-regression`](../mod-107-online-eval-and-regression).
- **Safety and guardrail eval** applies the same rubric + bias-control shape to safety criteria (refusal, jailbreak, policy adherence) → [`mod-108-app-safety-and-guardrails-eval`](../mod-108-app-safety-and-guardrails-eval).
- **Human review workflows** consume judge disagreements and near-tie pairwise decisions as the review queue → [`mod-109-human-review-workflows`](../mod-109-human-review-workflows).
- **Eval-data platform** stores the calibration set, the replay set, and the per-judgement lineage (rubric hash + judge model + tier) as first-class tables → [`mod-110-eval-data-platform-slice`](../mod-110-eval-data-platform-slice).
- **Cost / latency / quality trade-off** rolls up per-tier judge spend against per-tier quality delta → [`mod-111-cost-latency-quality-tradeoff`](../mod-111-cost-latency-quality-tradeoff).
- **Program-owner posture** treats the routing config, the drift monitor, and the escalation contracts as first-class release-gate artefacts to defend to the Governance-family peer → [`mod-112-owning-an-ai-eval-program`](../mod-112-owning-an-ai-eval-program).
- **Escalation to the Model-Development-family `model-evaluation-engineer` peer** for full-methodology calibration and drift analysis when the product-altitude quick-calibration in chapter 04 is not enough.
