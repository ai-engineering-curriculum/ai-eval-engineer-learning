# G-Eval, Framework-Based Judges, and Open-Source Judges

## Motivation

Chapters 01 and 02 defined the rubric and the pipeline controls. This chapter is about **who runs them**: the judge model and the framework that wraps it. The three real choices in a product pipeline today are:

1. **A frontier chat API used as a G-Eval-style chain-of-thought judge**, wired via DeepEval / Braintrust / Promptfoo (or your own scaffolding). Highest agreement with human raters on most tasks; highest per-call cost; changes underneath you when the vendor upgrades.
2. **A dedicated open-source judge model** — Prometheus 2 (Kim et al. 2024, <https://arxiv.org/abs/2405.01535>) or JudgeLM (Zhu et al. 2023, <https://arxiv.org/abs/2310.17631>) — self-hosted for cost predictability, reproducibility across model versions, and data residency.
3. **A hybrid**: framework-based frontier judge for the low-volume high-stakes decisions (release-gate A/B, calibration set scoring), OSS judge for the high-volume monitoring pass on live traffic. Chapter 05 is where routing across these tiers lands.

Each choice has real trade-offs, and the correct choice for your surface is a function of volume, budget, reproducibility requirements, and how far your rubric drifts from the ones the OSS judges were trained on. This chapter walks the G-Eval method the mainstream frameworks implement, sketches the three frameworks' judge primitives, and lays out the decision criteria for switching to a dedicated OSS judge.

## Core concepts

### G-Eval: chain-of-thought + form-filling + weighted decoding

**G-Eval** (Liu et al. 2023, "G-Eval: NLG Evaluation using GPT-4 with Better Human Alignment", <https://arxiv.org/abs/2303.16634>) is the paper the mainstream frameworks operationalise for absolute-rubric scoring. It is worth reading; the summary of what it prescribes:

1. **Chain-of-thought before the score.** The judge prompt asks for step-by-step evaluation reasoning against the criterion before it emits the score, on the theory that reasoning tokens produce more consistent scores than direct emission.
2. **Form-filling output.** The judge fills a structured output form (score + rationale) rather than free-text. This is a straight superset of the "output schema" requirement from chapter 01.
3. **Probability-weighted score aggregation.** For a discrete numeric scale, weight the score by the model's output logits: instead of taking the single highest-probability integer, compute `sum(i * P(i))` over the possible score values. The paper argues this reduces the "flat" behaviour of judges that always emit 4/5.

In practice, most product-side implementations follow (1) and (2) and skip (3), because logit access is not always available on the current-generation frontier judge APIs, and because framework libraries default to token-level scoring rather than logit-weighted. That is the pragmatic default; know that you are trading a small amount of measured agreement improvement for portability and simplicity. If your framework does expose logit-weighted G-Eval (DeepEval's `GEval` metric is the canonical example — see the confident-ai docs — and it degrades gracefully to sampled scoring when logits are unavailable), turn it on.

<!-- needs-research: re-check on the next research cycle whether the current top-tier frontier APIs expose the top-token log-probabilities G-Eval's weighted-score variant relies on, and whether DeepEval's GEval scorer still degrades gracefully when they do not. -->

### The three mainstream frameworks

Each of DeepEval, Braintrust, and Promptfoo ships a G-Eval-style scorer plus the pipeline plumbing to wire it into a test suite or an online eval loop. The differences that matter for a product pipeline:

**DeepEval** — <https://docs.confident-ai.com/>. Python-first. Its `GEval` metric implements the chain-of-thought form-filling pattern with an explicit `criteria` string and `evaluation_steps` list, and returns a score with rationale. Integrates with pytest-style tests, which is where mod-106 CI gates land. Ships a family of pre-built metrics (faithfulness, answer relevancy, contextual precision, hallucination detection) alongside the general-purpose `GEval`. Judge model is pluggable — swap frontier / mid-tier / OSS at the metric constructor.

**Braintrust** — <https://www.braintrust.dev/docs>. Eval-first data model: the primary artefact is a scored *experiment*. The `autoevals` library (open-source, <https://github.com/braintrustdata/autoevals>) ships graders including `LLMClassifier` (categorical), `ClosedQA` (rubric-based scoring against a criterion), and `Battle` (pairwise). Uses handlebars-style prompt templates so the rubric prompt is a first-class versioned file rather than a Python string. Pairs naturally with mod-102 tracing when you already run Braintrust as your trace backend.

**Promptfoo** — <https://www.promptfoo.dev/docs/intro/>. CLI-and-YAML first, oriented toward eval-in-CI. Its `llm-rubric` assertion (<https://www.promptfoo.dev/docs/configuration/expected-outputs/model-graded/>) is the equivalent primitive: a rubric string + a judge model returns a pass/fail with rationale. Also ships `factuality`, `model-graded-closedqa`, and pairwise comparison assertions. Strong fit when your rubric lives alongside a prompt-versioned regression suite and you want CI to fail on a numeric threshold with no additional plumbing.

The choice among the three is mostly about the shape of the surrounding surface:

- **Pytest-native, Python surface, judge-scored SLIs feed a Python dashboard** → DeepEval.
- **Eval loop is the primary artefact and Braintrust is (or would be) the trace backend** → Braintrust `autoevals`.
- **Regression suite is prompt-versioned YAML, gated in CI, no Python glue needed** → Promptfoo `llm-rubric`.

None of them lock you in. All three let you swap the judge model at the scorer constructor, all three emit structured output you can attach to a trace, all three accept a rubric that reads verbatim from the chapter-01 spec.

### When to switch to a dedicated open-source judge

Frontier-judge frameworks are the default for a reason: they land the highest human-agreement numbers on most rubrics out of the box. There are three real reasons to switch (or add) a dedicated open-source judge:

**Cost.** For online monitoring at scale — every sampled trace judged on 3–5 criteria — frontier per-call costs add up quickly. Self-hosted Prometheus 2 or JudgeLM behind vLLM / TGI can drop the per-call cost by an order of magnitude, and once you have the GPU it is a fixed cost regardless of volume. The break-even depends on volume, GPU price, and current frontier prices; run the math for your surface.

**Reproducibility.** A frontier API can change underneath you — a new snapshot, a silent quality shift. For research artefacts, incident post-mortems, or regulatory evidence where you must be able to say "this judge score is the score this model would return today and next quarter," a pinned open-source judge on your own weights is the only defensible answer. Chapter 05's drift monitor plays with this — frontier judges have to be re-baselined more often.

**Data residency and privacy.** If your traces cannot leave your VPC (healthcare, government, EU-residency contracts), a self-hostable judge is the only option. Frontier-vendor terms occasionally accommodate this with private deployment tiers; the OSS-judge path avoids the negotiation.

### Prometheus 2 and JudgeLM at a glance

**Prometheus 2** (Kim et al. 2024, <https://arxiv.org/abs/2405.01535>) — a family of open-source evaluator LLMs released in 7B and 8×7B sizes, produced by weight-merging fine-tuned single-family evaluators. Trained on the *Feedback Collection* and *Preference Collection* datasets, which include per-rubric scoring rubrics and A/B preferences. The model is designed to consume a custom rubric at inference time (not just its training-time rubrics), so it slots into a G-Eval-style prompt. Weights are on Hugging Face; the paper reports competitive human-agreement with frontier judges on the covered task families.

**JudgeLM** (Zhu et al. 2023, <https://arxiv.org/abs/2310.17631>) — a family of fine-tuned open judges (7B, 13B, 33B in the paper) trained on GPT-4-generated judgements over a large multi-task dataset. Ships an evaluation harness and weights (<https://github.com/baaivision/JudgeLM>). Emphasises pairwise judgement quality and has mitigations for position and length biases baked in during fine-tuning.

Both are pragmatic starting points. Both are also research artefacts — treat their reported numbers as "how they did on the datasets they were evaluated on," not as "what they will do on your surface." Chapter 04's quick-calibration is how you turn "the paper says the OSS judge tracks GPT-4 with κ ≈ X on their benchmarks" into "this OSS judge tracks our human raters with κ ≈ Y on our rubric." Only Y decides whether the switch is safe for you.

<!-- needs-research: on the next cycle, check for newer open-source judge families (e.g., successors to Prometheus 2 and JudgeLM) that publish weights and inference recipes, and update this chapter accordingly. Also re-verify the specific dataset names, sizes, and reported metrics for Prometheus 2 and JudgeLM against the current arXiv versions and repo READMEs. -->

### A decision framework

Use this to pick where each rubric on your surface lands. The rows are chosen so each row's "yes" answer selects a tier — pick the topmost tier every "yes" points to.

| Question | If yes, use |
|---|---|
| Is this rubric a release-gate A/B where a wrong call ships a regression? | Frontier judge via DeepEval / Braintrust / Promptfoo |
| Is the calibration set for this rubric being *built*, not scored? | Frontier judge (best per-item agreement while you build the gold set) |
| Does this rubric run on ≥ N traces per day where the frontier bill exceeds the OSS-GPU cost? | OSS judge (Prometheus 2 / JudgeLM) via vLLM |
| Must this judgement be reproducible next quarter and beyond? | OSS judge pinned to a version |
| Must this judgement not leave your VPC? | OSS judge in your VPC |
| None of the above — routine online-eval monitoring? | Whichever your framework already ships and your calibration says agrees with humans well enough (chapter 04) |

For N (the volume threshold): rough rule of thumb is compare the *marginal per-call cost* of the frontier judge to the *fully loaded* cost of the OSS-judge GPU including engineering time to maintain it. Below the threshold the frontier judge wins because the OSS-judge maintenance cost dominates; above it, cost scales linearly with volume for frontier but stays flat for OSS. Compute the number quarterly against your traffic; do not hard-code it.

### Wiring an OSS judge behind a framework

The frameworks above are model-agnostic. To route to a self-hosted OSS judge, you have two mainstream paths:

1. **OpenAI-compatible endpoint.** vLLM (<https://docs.vllm.ai/>) and TGI expose OpenAI-compatible `/v1/chat/completions` endpoints. Point DeepEval / Braintrust / Promptfoo's judge model at that endpoint via a custom base URL and the framework's judge machinery is agnostic to what is behind it.
2. **Custom judge callback.** All three frameworks accept a callable in place of a model name, so you can call the OSS judge directly (via `transformers`, `vllm`, or an HTTP client) and return the structured judgement.

In either path, keep the OSS judge's model version, weights hash, and vLLM version on the judgement (`app.judge_model_hash`, `app.judge_runtime_version`). Reproducibility depends on it — chapter 05's drift monitor is the consumer.

### Cross-cutting rules

- **Version the rubric prompt template and hash it on every judgement.** The frameworks all support this out of the box; do not skip it.
- **Emit an `EVALUATOR` span for every judgement** with the `app.judge_*` attributes from chapter 02. The mod-102 schema is the contract.
- **Do not judge on the same model your surface generates with**, especially not for pairwise A/B where the "winner" is credited to that model. Self-preference bias (chapter 02) contaminates the outcome.
- **Do not stack multiple criteria into one prompt** unless your framework's scorer is designed for it. One prompt per criterion — even if it means running N prompts — keeps the rubric clean and the calibration tractable.
- **Cache aggressively but by content, not by trace id.** A cached judgement for a rewritten rubric is silently wrong; hashing the (rubric prompt template + candidate + evidence) is the safe cache key.

## Summary

- **G-Eval** = chain-of-thought reasoning + form-filling output + (when the API allows) probability-weighted score aggregation. It is what the mainstream frameworks implement for absolute rubrics.
- **DeepEval** (Python / pytest), **Braintrust** (eval-first, `autoevals` library), and **Promptfoo** (YAML / CLI in CI) are the three real framework choices; pick by the shape of the surrounding surface, not by feature checkbox.
- **Switch to a dedicated OSS judge** (Prometheus 2, JudgeLM) when cost at scale, cross-quarter reproducibility, or data residency forces it. Continue to use a frontier judge for calibration and release-gate A/B where agreement quality matters more than cost.
- OSS judges wire behind DeepEval / Braintrust / Promptfoo trivially via an OpenAI-compatible vLLM endpoint or a custom callback. Version-pin the weights and record the pin on every judgement.
- Never judge with the same model family the generator lives in for competitive A/B; the self-preference bias from chapter 02 will contaminate the result.

Chapter 04 turns any of these judges — frontier, framework, or OSS — into a *calibrated* judge against a small human gold set, and marks the line where you escalate to the model-evaluation-engineer peer for full statistical depth.
