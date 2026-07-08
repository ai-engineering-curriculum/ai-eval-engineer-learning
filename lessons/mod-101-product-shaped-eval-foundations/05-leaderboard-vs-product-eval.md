# When Leaderboards Collide With Real Product Surfaces

## Motivation

Sooner or later a stakeholder will land in your inbox with:

> *"Model X is 3 points ahead of our current model on MMLU / MT-Bench / Chatbot Arena / GSM8K / SWE-bench / [insert benchmark]. Ship it."*

Answering that email well is a load-bearing skill of this role. Some of those numbers deserve serious attention; others do not describe anything about your product. This chapter catalogues the failure modes that make a leaderboard number a bad proxy for a product surface, and gives you a template for accepting, adapting, or rejecting a benchmark on grounds a PM can follow.

## Core concepts

### The mismatch between leaderboard shape and product shape

A public benchmark is optimised to be a **stable, reproducible ranker of models**. That optimisation drives choices that are correct for the benchmark's purpose and often wrong for your surface:

| Benchmark property | Product-surface reality |
|---|---|
| Fixed inputs | Live-user-generated inputs |
| Single-turn (usually) | Multi-turn, streamed, tool-calling |
| Bare model (no scaffold) | Prompt + retriever + tools + guardrails + decoding config |
| Static reference answers | Answers whose correctness depends on live business state |
| Public since date X | Almost certainly in the training data of any recent model |
| Aggregate score | Users experience the tail |

Any single mismatch is enough to make the ranking uninformative for your surface. Several mismatches make it actively misleading.

### Failure modes

Six leaderboard failure modes to keep in your head. Each one is a reason to challenge a proposed benchmark.

**1. Distribution mismatch.** The benchmark's input distribution does not resemble your users' input distribution. A model that ranks 1st on academic Q&A can still write worse customer-support replies than a model that ranks 5th, because the two distributions barely overlap.

**2. Contamination / data leakage.** The benchmark's test set is in the model's training data or in retrieval corpora available at inference. Contamination inflates scores without teaching you anything about *your* surface. This is documented across many popular benchmarks; the SWE-bench maintainers themselves published [SWE-bench Verified](https://openai.com/index/introducing-swe-bench-verified/) after auditing found issues with the original set.

**3. Aggregate hides the tail.** A benchmark score of 87% averages away the 13% of cases where the model fails, but users experience the tail. Two models with the same aggregate can have wildly different tail behaviour. Product surfaces care about the tail.

**4. Goodhart's law.** "When a measure becomes a target, it ceases to be a good measure." When teams optimise against a public benchmark, the score decouples from the underlying capability. This applies equally when *you* over-optimise against a small internal fixture.

**5. Metric hackability.** Many popular benchmarks reduce to multiple choice or to string-match against a reference answer. Multiple-choice accuracy is not the same phenomenon as free-form response quality. String-match reference answers penalise legitimate paraphrase and reward reference-cribbing.

**6. Prompt sensitivity.** A model's benchmark score can move several points on prompt template alone. Unless you know the exact prompt the leaderboard used, and unless your surface uses that prompt (it won't), the number is not comparable to your surface's number.

Two supporting failure modes worth knowing:

**7. Judge-model shift.** Leaderboards that use LLM-as-judge (MT-Bench, Chatbot Arena setups) are subject to judge drift and bias. See [Zheng et al. 2023 (MT-Bench / Chatbot Arena)](https://arxiv.org/abs/2306.05685) for position bias and self-preference bias in judges — deeper coverage in `mod-104`.

**8. Static benchmark ageing.** A benchmark released years ago no longer captures the current frontier of user expectations. Static suites decay silently.

### Where leaderboards *are* useful

Not all benchmarks are worthless for a product team. Public benchmarks earn their keep in three specific roles:

- **Shape templates.** SWE-bench-Verified, τ-bench, WebArena, GAIA, BFCL, ToolBench, HarmBench, AgentDojo, InjecAgent, RAGAS, MT-Bench — you copy their *task shape* into an internal suite built against your surface's real distribution. `mod-103`, `mod-105`, and `mod-108` use them exactly this way.
- **Coarse pre-filter.** If a model is below some minimum bar on a public suite, do not bother evaluating it on your surface — but *only above that bar* does the public number carry information, and it stops carrying information about which model above the bar is best for you.
- **Sanity check.** After adopting a new model, running one or two public evals confirms your integration path is intact. It does not confirm the model is better for your surface.

### The rejection template

When a benchmark is proposed as a proxy for a product decision, walk the following checklist with the requester. Say the outcome in these terms:

1. **Distribution.** Compared to a representative sample of our production traffic, how similar are the benchmark's inputs?
2. **Contamination.** Is the benchmark's test set publicly available, and if so, what is the evidence it is *not* in the training corpus of the model we are comparing to?
3. **Scaffold.** Does the benchmark run against a bare model or against a scaffold resembling our surface's scaffold (prompt, retriever, tools, guardrails, decoding)? If bare, does the score move under our scaffold?
4. **Metric.** Does the metric penalise the failure our users will notice? (E.g., multiple-choice accuracy does not capture faithfulness in free-form summarisation.)
5. **Aggregation.** Is the score reported only in aggregate, or is per-slice / per-tail reporting available?
6. **Prompt template.** Was the benchmark's exact prompt template published? Is it the template we use?
7. **Judge.** If the benchmark uses LLM-as-judge, which judge? Are the position/length/self-preference bias controls documented?
8. **Recency.** When was the benchmark released? Has the frontier caught up with it?

Then classify:

- **Accept as-is** — rare. Only when 1–7 all pass and the surface is genuinely well-approximated by the benchmark.
- **Adapt** — most common. Use as a *shape template* for an internal suite on your surface's distribution.
- **Reject** — when the mismatch is fundamental (usually 1 or 2 fails hard, or 4 does).

Whatever the outcome, write it down in the eval plan (chapter 06). Future stakeholders will re-raise the same benchmark; the written rationale saves you re-litigating it.

### What "shape template" looks like in practice

Adopting a public benchmark as a shape template usually means:

- Keep the **task format** (chat with tool calls, multi-hop retrieval, coding-patch, etc.) so downstream reporting is legible to people familiar with the public benchmark.
- Replace the **inputs** with a curated sample from your own production distribution (scrubbed for PII), plus author-crafted edge cases.
- Replace the **reference answers** with either a small in-house gold set (see `mod-109`) or a rubric that a judge can apply consistently.
- Keep the **metric family** (e.g., trajectory-level pass rate for τ-bench-shape suites) but redefine what "pass" means for your surface.
- Version the resulting suite and pin its fixture-hash into your CI (see `mod-106`).

## Example: rejecting a benchmark for a customer-support surface

A stakeholder proposes ranking two models on MT-Bench as a proxy for a customer-support triage surface. Walking the template:

- **Distribution.** MT-Bench prompts are open-ended general chat; support triage inputs are short, jargon-heavy, and reference internal state. Fails.
- **Contamination.** MT-Bench prompts are public and old. Both candidate models are likely trained on the benchmark. Fails.
- **Scaffold.** Support triage runs behind a RAG retriever over internal docs and can call a ticket tool. MT-Bench is bare-model. Fails.
- **Metric.** MT-Bench scores freeform quality via judge. Support triage cares about *did the RAG citation match policy* and *did we route to the right queue*. Fails.
- **Aggregation, prompt, judge, recency.** Not the load-bearing issues; still concerns.

Outcome: **reject.** Replace with an internal triage-eval suite adapted from τ-bench-shape templates (`mod-103`) plus a RAG-triad suite (`mod-105`) grounded in the actual support corpus. Write the rejection into the plan; move on.

## Common traps when rejecting benchmarks

- **Rejecting without offering an alternative.** "This benchmark is bad" without "here is what would answer your question" reads as gatekeeping. Always propose the internal-suite adaptation.
- **Rejecting because the model you like scores lower.** Do not do this. Your job is to make the criteria defensible before you know the scores.
- **Rejecting sight-unseen.** Read the benchmark's paper or README before responding. Sometimes the request is right.

## Summary

- Leaderboards optimise for stable model ranking; product surfaces optimise for user experience under a specific scaffold on a live distribution.
- **Six load-bearing failure modes:** distribution mismatch, contamination, aggregate-hides-tail, Goodhart, metric hackability, prompt sensitivity — plus judge-model shift and static-benchmark ageing.
- Leaderboards are still useful as **shape templates**, **coarse pre-filters**, and **integration sanity checks**.
- Use the eight-question **rejection template** with any stakeholder pushing a benchmark; classify **accept / adapt / reject** and write the outcome into the eval plan.

The final chapter of this module pulls everything together into the two-page plan the PM, ML engineer, and governance owner all sign off on.
