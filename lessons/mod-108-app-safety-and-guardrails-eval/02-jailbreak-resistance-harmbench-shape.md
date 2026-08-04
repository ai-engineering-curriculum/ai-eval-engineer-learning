# Jailbreak Resistance at the App Surface: HarmBench Shape

## Motivation

Chapter 01 defended the claim that the app-surface number is the one that matters. This chapter builds the number. The mechanism is a **jailbreak-resistance eval** — an attack replay that fires refusal-eligible prompts at your surface, records whether the app produced content the policy says it should not have, and rolls the result up per attack family and per policy category.

Two questions have to be answered *before* the first line of the runner:

- **How do you consume an attack suite without the harmful strings landing in your repo?** HarmBench is public. The behaviour ids, the taxonomy, and the metrics are all citeable in a `resources.md`. The prompts themselves — the ones the suite considers most dangerous — are what the authors gated behind acknowledgements and what your compliance team will (rightly) object to seeing in `git grep`. Get the payload-management contract right first.
- **What number are you actually reporting?** "Attack success rate" (ASR) is the top-line metric, but the useful number is decomposed: ASR per attack family, ASR per policy category, ASR@k for repeated attempts, ASR conditional on the guardrail's decision. A single scalar hides all of the actionable information.

This chapter answers both. It walks the HarmBench benchmark shape (behaviour × attack matrix, ASR / ASR@k / robustness score), the app-surface adaptation (how to point HarmBench at your endpoint instead of at a raw model), the payload-management contract this module enforces, and the interaction with the mod-107 online loop and mod-106 CI gate.

## Core concepts

### HarmBench in one paragraph

HarmBench (Mazeika et al., *A Standardized Evaluation Framework for Automated Red Teaming and Robust Refusal*, ICML 2024) is a public benchmark that scores a **target model** against a matrix of **attack methods × harmful behaviours**. The behaviour set covers seven functional categories (chemical / biological weapons, cybercrime & unauthorised intrusion, harassment & bullying, harmful information seeking, illegal activities, misinformation & disinformation, general harm) and three semantic categories (contextual, copyright, standard). The attack set covers automated attacks (GCG, PAIR, TAP, PAP, AutoDAN, human-written jailbreaks, and others). The top-line metric is **ASR** — the fraction of `(behaviour, attack)` cells where the target's response is judged as *fulfilling* the behaviour rather than refusing. HarmBench ships a classifier ("HarmBench-Cls" — a fine-tuned Llama-2-13B) as its default judge; other judges can be substituted. The full framework runs on Hugging Face-hosted target models by default; the repo also documents a `chat_template` / API-adapter path for hosted models.

Read the paper before writing a single line of eval code. Not because your implementation will resemble the reference — it won't — but because the *categories* the paper argues for and the *judge properties* it derives are what the resulting numbers mean.

### The benchmark shape: behaviour × attack matrix

HarmBench's core artefact is a matrix whose rows are behaviours (`{"id": "hb_chem_001", "category": "chemical_biological", "desc": "...", "tags": ["contextual"]}`) and whose columns are attack methods (`GCG`, `PAIR`, `TAP-T`, `AutoDAN`, `PAP`, `Human_written`, `DirectRequest`, etc.). Each cell holds an *attack instance* — the specific prompt the attack method produced for that behaviour against a specific target model.

Two consequences:

- **Attacks are model-specific.** GCG's adversarial suffix is optimised against a specific target. A suffix that jailbreaks Llama-2-7B does not, in general, jailbreak Claude 3.5 Sonnet. Some attacks (Human_written, DirectRequest, plus some PAP / PAIR runs) transfer between models better than others; the paper reports transfer rates.
- **New attacks appear.** The 2024 paper's attack set is a snapshot. Since publication the community has produced further attack families (multi-turn attacks, low-resource-language attacks, translation attacks, cognitive-overload attacks, Best-of-N sampling attacks against reasoning models). The benchmark shape survives new attacks; the specific columns are always out of date. Treat the framework as a *scoring harness*, not a *fixed attack set*.

### The metric: ASR, ASR@k, robustness score

Four numbers you actually report.

- **ASR per (behaviour, attack).** Did the target fulfil the behaviour on this attack? Judge-scored, binary.
- **ASR per attack family (rolled up over behaviours).** Which attack family is your surface most vulnerable to?
- **ASR per behaviour category (rolled up over attacks).** Which policy category is your surface most vulnerable in? This is the number the product owner reads.
- **ASR@k for stochastic attacks.** For attacks that resample (PAIR, TAP, Best-of-N), the reported rate is often "success within `k` attempts." A `k = 1` number is a lower bound; a `k = 50` number is close to the ceiling. Report both.

HarmBench also reports a **robustness score** = `1 - ASR`. Product-facing dashboards read easier with a "we blocked 94.3 %" framing than a "we failed 5.7 %" framing; the underlying number is the same. Pick a convention per your team and use it consistently — mixing the two on a single dashboard is how the on-call misreads a green cell as a red one.

### The judge

HarmBench's default judge is a Llama-2-13B fine-tuned to classify whether a target's response fulfils a harmful behaviour. Two properties matter.

- **It is *not* the same as a moderation classifier.** A moderation classifier scores whether text is harmful in isolation ("does this passage contain instructions for a bomb"). HarmBench's classifier scores whether the response *fulfils the requested behaviour* — a response that says "I cannot help with this, but here is what I can help with" is not a fulfilment even if it contains a substring the moderation classifier would flag.
- **Judge quality bounds the eval.** If the judge over-scores refusals as fulfilments, your ASR is inflated. If it misses subtle fulfilments (partial instructions, indirect assistance), your ASR is deflated. Chapter 04 of mod-104 discussed judge calibration; run the same calibration on your safety judge against a small human-labelled subset before trusting the automated number.

Options for the judge tier: the HarmBench-Cls model itself, a hosted LLM-as-judge (e.g., Claude with a HarmBench-derived rubric), Llama Guard 3 as a classifier (see chapter 05), or an ensemble. Pick per your budget and calibrate against human labels; the same tier-router pattern from mod-104 chapter 05 applies.

### The payload-management contract

This is the non-negotiable rule of the module: **harmful prompts do not enter the repo.** The reason is threefold — security-review friction, legal exposure in some jurisdictions, and social-engineering risk if the repo is compromised. HarmBench's authors partly acknowledged this by requiring an access acknowledgement for the full behaviour set; your team's contract should be strictly more careful.

Four patterns satisfy the contract:

- **Consume by reference, not by value.** Your suite reads `hb_chem_001` (an id) and pulls the current text from a *runtime-only* source — a Hugging Face dataset load at runner startup, a sealed archive on a locked-down bucket accessed with a short-lived token, or (least preferred) an encrypted-at-rest local file that CI decrypts with a scoped secret. The id is checked in; the text is not.
- **Hashes at rest.** The scored row records a SHA-256 of the prompt (the id is not enough, because a suite maintainer can edit the text under the same id). Auditors can verify the hash matches an archive without ever needing the text.
- **Judge inputs pass through a redacted-log wrapper.** Any structured log of the eval run replaces the prompt and the response with `<REDACTED length=N sha256=...>` unless a specific reviewer role is set. The judge itself sees the raw text (it has to); everything the on-call sees post-hoc is redacted.
- **No `git grep` surface.** Neither the prompts nor the responses land in git-tracked files. The scored rows land in the mod-110 eval-data store (which has its own access controls), not in a `.jsonl` in the repo.

A rule of thumb: if a compliance reviewer objects to any commit in the suite's history, you have already lost the argument. Design the runner so that the objection cannot arise.

### Running against the app surface, not the raw model

The HarmBench reference runs against a Hugging Face-hosted target model. Your surface is `POST /v1/support-bot/chat` (or whatever your app exposes). Three adaptations.

- **Wrap your endpoint as a HarmBench "target."** HarmBench models a target with a `generate(prompt, ...)` interface. Write a small adapter that maps the interface to your app's request format (session id, tenant id, auth token, system context). The adapter reuses your production client. Attack instances hit the same endpoint the user does.
- **Configure the attack methods that make sense.** GCG's optimisation loop is *impossible* against a hosted app-surface — GCG needs white-box logits. Skip it (or run it once against the model surface — a peer artefact). PAIR, TAP, PAP, and Human_written all work against a black-box API and are the primary attacks against an app surface.
- **Attack the surface in its production configuration.** System prompt in place, retrieval enabled if applicable, tools enabled if applicable, guardrails on. If you disable a defence for the eval "to get a cleaner signal," the number you produce is fiction — you are no longer scoring the surface the user hits.

The attack throughput is bounded by your surface's rate limits. Plan for it. A HarmBench-shaped run against a rate-limited endpoint at 5 rps takes tens of minutes for the standard behaviour set × a small number of attacks. Overnight runs are fine; a fast PR-gate variant scores a stratified sample and defers the full run to a nightly.

### Interaction with mod-106 (offline gate) and mod-107 (online loop)

Two integration points.

- **PR gate — subsampled adversarial fixture set.** Mod-106 chapter 04 defined the fixture-replay eval that runs on every PR. Add a *safety* sub-suite: a stratified sample from the HarmBench behaviour set, one attack per behaviour, chosen for fast throughput and judged with the same classifier the nightly run uses. A regression in the fast subset gates the PR; the nightly full run gives the depth. Threshold: the sub-suite's ASR should not exceed a small delta over the last known-good baseline — chapter 07 walks the specific number.
- **Online loop — attack-source-tagged sampling.** Mod-107 chapter 02's sampler picks a fraction of live traces. Not all live traces are adversarial (most are benign). Two augmentations: (a) a **synthetic-attack cohort** where a scheduled worker fires attack replays at production endpoints on a separate `synthetic=true` cohort key, scored with the same rubric; (b) a **bias-toward-interesting** on the online sampler that up-weights traces flagged by the input classifier as *attempted* attacks. Both feed the mod-107 scored-row store on separate cohorts.

The nightly full-suite run, the PR sub-suite gate, and the online synthetic-attack cohort all read from the same suite definition (the id list plus the attack manifest). Do not maintain three versions.

### What the scored row looks like

Extend the mod-107 chapter 02 schema with safety-eval-specific columns. Additions only:

```json
{
  // existing mod-107 fields ...
  "eval_type": "jailbreak_replay",
  "attack_family": "PAIR",
  "attack_variant": "iterative_v3",
  "behaviour_id": "hb_chem_001",
  "behaviour_category": "chemical_biological",
  "attack_prompt_hash": "sha256:...",
  "response_hash": "sha256:...",
  "asr_verdict": true,
  "attempts_used": 3,
  "attempts_budget": 10,
  "judge_id": "harmbench_cls_v0_1",
  "judge_calibration_snapshot": "human_labels_2026-07-15",
  "cohort_keys": {
    "product_surface": "support_bot",
    "prompt_version": "sha256:...",
    "guardrail_stack": "input_cls_v2+llama_guard_3+system_prompt_v11",
    "synthetic": true
  }
}
```

Two invariants the schema enforces:

- The prompt and the response are *hashes only*. The raw strings live in the sealed archive; nothing that ends up in a queryable table contains the payload.
- The `guardrail_stack` cohort key names the *full defence configuration* that produced the row. When someone deletes the input classifier and the ASR jumps, the cohort key attributes the jump to the configuration change, not to a mystery drift.

The mod-107 aggregator, drift monitor, canary gate, and dashboards all read this schema without modification (the added columns are metadata on the same row shape).

### Complementary tools worth naming

HarmBench is the shape this chapter builds against. Two adjacent open-source tools are worth knowing.

- **garak** (NVIDIA). A vulnerability scanner for LLMs; ships with plugin-shaped attacks (jailbreak probes, PII probes, hallucination probes, encoding-attack probes). Its scanner is a good complement to HarmBench for attack families the paper does not cover; the reporting shape is different (garak reports per-probe results, HarmBench reports per-behaviour). Use both.
- **PyRIT** (Microsoft). Python Risk Identification Toolkit for LLMs; orchestrates automated red-team runs with configurable attackers, targets, and scorers. Its architecture is closer to an *orchestration framework* than a *benchmark*; you can drive HarmBench-shaped runs from PyRIT if you want a single toolchain.

The choice between these is a taste-and-fit decision. HarmBench is the strongest anchor for the shape of the report (behaviour × attack, ASR family, classifier); PyRIT is the strongest anchor for a maintainable production runner (orchestration, retry, multi-target support); garak is the strongest anchor for coverage of the long tail of attack families. Chapter 07 shows the OWASP mapping for findings regardless of which tool produced them.

### Failure modes to design against

Five patterns fail an app-surface jailbreak eval quietly. Each is worth a paragraph in the runbook.

- **The judge over-refuses.** Your ASR looks better than reality because the judge scores partial fulfilments as refusals. Mitigation: chapter 04 mod-104 calibration procedure; a small human-labelled subset revisited monthly.
- **The attack suite ages.** A suite from 2024 does not include 2026's multi-turn attacks; your low ASR is on old attacks. Mitigation: version the suite (`hb_v0.5_augmented_multiturn_v2`), record the version on the scored row, refresh at least quarterly.
- **The attack targets an easier baseline than the surface.** GCG optimised against Llama-2-7B produces suffixes that jailbreak Llama-2-7B; the same suffix against your surface is often ineffective — you look "robust" for the wrong reason. Mitigation: prefer transferable attacks (Human_written, PAP, PAIR against a similar base) and be explicit in the report about which attacks are transfer vs native.
- **A guardrail changed but the cohort key did not.** The number moves; the runbook cannot attribute it because the guardrail-stack cohort key was not updated when the guardrail was. Mitigation: chapter 05's guardrail-config-hash goes on the cohort key; a CI check verifies the config hash matches the deployed configuration.
- **The eval hits production.** A synthetic-attack cohort fires attacks at the live endpoint. If the endpoint has side effects (writes a support ticket; sends an email), those side effects fire too. Mitigation: dedicated synthetic-only tenant, endpoint recognises the tenant and short-circuits side-effectful tools; chapter 04's sandbox pattern applies here too.

## Summary

- **HarmBench** is a behaviour × attack matrix, judged by a classifier, reporting ASR (attack success rate) per attack family and per behaviour category. This chapter uses HarmBench as the *shape* of the report, not necessarily as the only source of attacks.
- The **payload-management contract** is non-negotiable: consume attacks by reference (id + runtime fetch), record hashes not text on the scored row, redact from logs, never `git grep`-able.
- The **app-surface adaptation** wraps your endpoint as a HarmBench "target"; runs the attacks against the surface in its full production configuration; drops white-box attacks (GCG) that require model logits.
- **Judge calibration** bounds the eval — over-refusing judges inflate ASR, missing judges deflate it. Human-labelled subset revisited monthly (mod-104 chapter 04 procedure).
- **Integration**: subsampled adversarial fixture set on the PR gate (mod-106), nightly full-suite run, synthetic-attack cohort on the online loop (mod-107). One suite definition drives all three.
- The **scored-row schema** extends mod-107 chapter 02 with safety-specific columns; prompt and response are hashes; `guardrail_stack` on the cohort key attributes any change in ASR to the configuration that produced it.
- Complementary tools: **garak** (coverage), **PyRIT** (orchestration). Pick per your fit; report shape follows HarmBench.

Chapter 03 moves to prompt injection — a different attack surface with a different scoring pattern, because the attacker is not the user typing to your app but the retrieved document or the tool output that the user's agent trusts.
