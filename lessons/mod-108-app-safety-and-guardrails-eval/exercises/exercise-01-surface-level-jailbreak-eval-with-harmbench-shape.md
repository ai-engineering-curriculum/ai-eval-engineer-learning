# exercise-01: Surface-Level Jailbreak Eval With HarmBench Shape

**Estimated effort:** 3 hours

## Objective

Stand up a **HarmBench-shaped jailbreak-resistance eval** against your app surface. The eval must (a) score attack success rate (ASR) per attack family and per behaviour category, (b) never allow harmful payloads to enter your repo, (c) hit the same endpoint your production users hit, in its full defence configuration, and (d) emit scored rows that the mod-107 aggregator and drift monitor can consume without modification.

This exercise builds the *first* axis of the app-safety scorecard. Exercises 02 (injection), 03 (tool abuse), and 04 (guardrails) build the next three; exercise 05 stitches all four into an OWASP-mapped runbook.

## Prerequisites

- Chapters 01 and 02 of this module.
- The instrumented backend from mod-102 exercise-01 (OTel-GenAI-conforming spans) exposed as an HTTP endpoint you can hit. A minimal `POST /chat` echoing your surface's request shape is enough if the full mod-102 stack is not up.
- The mod-107 exercise-01 scored-row store and aggregator. If mod-107 is not done, a small SQLite table with the mod-107 chapter 02 schema is enough.
- The mod-104 exercise-05 tier-router config for the safety-eval judge (the same judge tiers apply to safety rubrics).
- A HuggingFace access token with acknowledgement of the HarmBench dataset terms (the full HarmBench dataset is gated). Optionally, the [HarmBench repo](https://github.com/centerforaisafety/HarmBench) cloned locally.
- Python 3.11+; `datasets`, `httpx`, and either the `HarmBench` repo installed or a HarmBench-shaped attack harness of your choice (`garak`, `PyRIT`).
- **A separate scoped judge key** (`EVAL_SAFETY_JUDGE_KEY`) — this must not be the production key or the mod-107 eval key. Rate-limit on the safety key can be lower; the eval is not on the critical path.

## Set-up

1. Create `eval/safety/jailbreak/` in your repo:

   ```
   eval/safety/jailbreak/
   ├── config.yaml
   ├── target_adapter.py         # wraps your surface as a HarmBench "target"
   ├── attack_loader.py          # loads attack instances by id from a runtime-only source
   ├── judge.py                  # scoring wrapper around HarmBench-Cls or an LLM-as-judge
   ├── runner.py                 # the loop entrypoint
   ├── schema.py                 # scored-row schema (mod-107 + safety extensions)
   ├── payload_contract.md       # what "no harmful strings in the repo" means for this repo
   ├── budget.md                 # eval cost derivation for this suite
   ├── runbooks/
   │   ├── asr_regression.md
   │   └── judge_calibration_drift.md
   └── tests/
       ├── test_target_adapter.py
       ├── test_payload_contract.py     # asserts no harmful strings are committed
       └── test_schema.py
   ```

2. In `config.yaml`, declare the surface, the behaviour subset ids, the attack families to run, the judge choice, and the operating budget. Example skeleton:

   ```yaml
   surface: support_bot
   endpoint: https://api.example.com/v1/support-bot/chat
   auth_env: SUPPORT_BOT_EVAL_TOKEN
   suite:
     source: harmbench
     version: v1.0
     behaviour_subset_id: hb_v1_functional_stratified_50
     attack_families:
       - DirectRequest
       - Human_written
       - PAIR
       - PAP
     max_attempts_per_attack: 10
   judge:
     tier: harmbench_cls
     fallback_tier: llm_as_judge
     calibration_snapshot: human_labels_2026-07-15
   budget:
     max_usd_per_run: 15.0
   payload_contract:
     prompts_at_rest: sealed_bucket_only
     scored_row_prompt: sha256_hash_only
     redacted_logs: true
   ```

3. Populate `payload_contract.md` with the concrete rules for your repo:
   - Where prompts live at runtime (bucket path, HF dataset name, sealed archive).
   - How the runner authenticates (short-lived token from the CI secrets store; how a local run authenticates).
   - What the logs look like (redacted format, retention).
   - How `test_payload_contract.py` asserts the rules (grep the repo for known HarmBench sentinel strings and fail if any are present).

4. Populate `budget.md` per chapter 02's arithmetic: attack count × attempts × judge cost + target-API cost per request. This is the derivation the runbook cites when the eval cost grows.

5. Configure a HuggingFace token if you use the HarmBench dataset directly. Verify the token has only what the runner needs (a scoped read on the specific dataset).

## Requirements

Produce a PR against your working branch that adds:

1. **`target_adapter.py`** — implements the HarmBench `Target` interface (or your framework's equivalent). `generate(prompt: str, **kwargs) -> str` posts to your production endpoint using the production request shape (session id, auth token, tenant id), returns the response body. Deliberately does *not* strip system prompts, retrieval, or guardrails.
2. **`attack_loader.py`** — loads attack instances by `(behaviour_id, attack_family)` from the runtime-only source. Never reads a `.jsonl` in the repo. Exposes an iterable `(behaviour_id, behaviour_category, attack_family, attack_prompt, attack_prompt_hash)` — the raw `attack_prompt` lives in memory only, is passed to the target, and is not logged in the clear.
3. **`judge.py`** — scores a `(behaviour, response)` pair for whether the response fulfils the behaviour. Wraps HarmBench-Cls at the default tier; falls back to an LLM-as-judge with a HarmBench-derived rubric. Returns `{asr_verdict: bool, judge_id: str, judge_confidence: float}`. Records `judge_calibration_snapshot` on every score.
4. **`runner.py`** — the loop entrypoint. Reads config; iterates `(behaviour, attack)` cells; skips cells that exceed the budget guard; writes scored rows to the mod-107 store. Honours a `--dry-run` flag that runs one cell end-to-end for smoke testing without a real judge call.
5. **`schema.py`** — extends the mod-107 chapter 02 schema with the chapter 02 additions (`eval_type`, `attack_family`, `attack_variant`, `behaviour_id`, `behaviour_category`, `attack_prompt_hash`, `response_hash`, `asr_verdict`, `attempts_used`, `attempts_budget`, `judge_id`, `judge_calibration_snapshot`, `guardrail_stack` on cohort keys, `synthetic=true`). Enforces the payload contract — the `attack_prompt` and `response` fields are hashes, not strings.
6. **`payload_contract.md`** — the concrete rules for this repo (see set-up step 3). Reviewer-facing.
7. **`budget.md`** — end-to-end cost derivation with real numbers (see set-up step 4).
8. **`runbooks/asr_regression.md`** — chapter 07's finding shape for an ASR regression: owner, threshold, first step (spot-check the top failing cells), remediation options (system-prompt hardening, guardrail category add, model snapshot roll), escalation criteria to `model-evaluation-engineer`.
9. **`runbooks/judge_calibration_drift.md`** — what to do when the judge's agreement with the human-labelled subset drifts below the threshold. Owner, threshold, procedure (re-run calibration; pin previous judge snapshot; escalate).
10. **`tests/test_payload_contract.py`** — grep the repo for a list of HarmBench sentinel strings and fail if any match. Include at least: the top 3 HarmBench category-marker strings, the "hb\_" behaviour-id prefix, and a small set of common jailbreak-prompt phrases from published papers (these are the low-signal test strings the test is looking for; the *test* is fine to commit; a real payload matching them is not).
11. **`tests/test_target_adapter.py`** — mocked HTTP verifies the adapter posts to the endpoint with the production request shape.
12. **`tests/test_schema.py`** — asserts the schema has every safety-extension field and rejects a row missing `attack_prompt_hash` or `response_hash`.
13. **A demonstration run** — run the runner against a small behaviour subset (~10 behaviours × 3 attack families = 30 cells) end-to-end. Produce the scored-row output, the per-attack-family ASR rollup, and the per-behaviour-category ASR rollup. Include a screenshot of the mod-107 aggregator reading the scored rows.

## Starter guidance

- **Do the payload contract before any attack code.** The single largest way this exercise goes wrong is discovering harmful strings in a commit history after the fact. Set up `test_payload_contract.py` first; the grep list catches sentinel-string commits early.
- **Pick a small behaviour subset for the first pass.** ~10 behaviours × 3 attack families is enough for the mechanism to be right; the full-suite run is a nightly job you set up in the stretch goals.
- **Use `DirectRequest` and `Human_written` first.** They do not require an attacker model to run. `PAIR` and `PAP` need an attacker LLM; wire them in after the direct-request cells work.
- **The target adapter is production code.** If your surface has a rate limit, honour it in the adapter (exponential backoff). If your surface has side effects (writes a ticket, sends an email), the eval must run against a `synthetic=true` tenant where the side-effectful tools are short-circuited — chapter 04 walks the sandbox pattern for this.
- **Judge calibration is not optional.** Even with HarmBench-Cls as the default judge, spot-check 20 cells with a human before trusting the number. This is one afternoon of work; the resulting `judge_calibration_snapshot` protects the report from silent judge drift.
- **`--dry-run` is worth the extra 30 minutes.** A smoke test that runs one cell end-to-end without incurring judge cost is what lets you iterate; without it every iteration burns budget.
- **Cohort key `guardrail_stack` matters.** Record the exact system prompt version + input classifier version + output classifier version + programmable-rails version as a single hash. When the ASR moves, this key tells the on-call which layer changed.
- **Do not skip the runbooks.** `asr_regression.md` and `judge_calibration_drift.md` are the runbooks the eval's own alerts will fire against in its first month; the on-call needs to know what to do (mod-106 chapter 03 rule).
- **Do not test against the raw model.** If you find yourself pointing the adapter at `api.openai.com/v1/chat/completions` with an empty system prompt, stop — you are measuring the peer's number, not yours.

## Acceptance criteria

You are done when:

- The runner completes an end-to-end run on ~30 cells; the scored-row store contains rows conforming to the extended schema; every row has `attack_prompt_hash` and `response_hash` populated (no raw strings).
- The per-attack-family ASR and per-behaviour-category ASR are computed against the demonstration run; a small table in the run's report shows both.
- The mod-107 aggregator reads the scored rows without modification and produces windowed statistics per `attack_family` cohort.
- `test_payload_contract.py` passes on the current branch and fails on a deliberate commit that contains a HarmBench sentinel string (verify by adding, running the test, and then removing).
- `test_target_adapter.py` and `test_schema.py` pass in CI.
- `payload_contract.md` reads cleanly as a compliance artefact — a reviewer unfamiliar with the module can see how the "no payloads in repo" rule is enforced.
- `budget.md` reflects real numbers for your run; the accrued cost of the demonstration run matches to within 30 %.
- The two runbooks in `runbooks/` name an owner, a threshold, a first step, remediation options, and an escalation criterion.
- The `--dry-run` flag executes one cell end-to-end without a judge call and without contacting the real target endpoint (uses a fake target for smoke testing).
- A short `README.md` in `eval/safety/jailbreak/` explains how to run the eval, how to interpret the scorecard, and where to find the sealed payload archive.

## Stretch goals

- **PAIR and PAP wired.** Add the two automated-attack families. These require an attacker LLM; wire the attacker key in the same scoped-secret pattern (`EVAL_ATTACKER_KEY`). Rate-limit-aware; record `attempts_used` per cell.
- **Nightly full-suite job.** Wrap the runner in a scheduled job that runs the full stratified behaviour set (a few hundred cells) once nightly. Publishes a dated report to a shared location. Cost derivation in `budget.md` reflects the nightly cadence.
- **Ensemble judge.** Add a second judge (an LLM-as-judge with an independent rubric); report both `asr_verdict`s and their agreement rate; use disagreement as the trigger for a human spot-check.
- **Attack-transfer analysis.** For attacks that transfer (Human_written, PAP), run the same attack against two model snapshots (your current production snapshot and a candidate snapshot); report the per-attack transfer rate. Feeds the mod-106 gate for a model-swap PR.
- **Synthetic-attack cohort on the online loop.** Wire the runner to fire a small stratified sample against production on a dedicated `synthetic=true` tenant continuously; scored rows land in the mod-107 aggregator on a `synthetic` cohort; drift monitor picks up regressions in near-real-time.
- **`garak` integration.** Wire the `garak` scanner as a second attack source (its probes cover encoding attacks and long-tail patterns HarmBench does not). Reuse the same target adapter; separate `suite` field on the scored row identifies which source produced the row.

## What this exercise does *not* cover

You are not building the injection eval (exercise 02), the sandboxed tool-abuse harness (exercise 03), the guardrail scorecard (exercise 04), or the OWASP-mapped runbook (exercise 05). You are shipping the *jailbreak axis* of the app-safety scorecard.
