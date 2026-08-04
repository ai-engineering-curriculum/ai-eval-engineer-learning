# Trace Snapshots as Regression Fixtures

## Motivation

The reproducibility property from chapter 01 rests on one specific artefact: the **regression fixture**. A fixture is the unit an eval gate scores against — one input, its ground truth, and everything else the run depends on. Because an LLM chain is stochastic, "the same run" only makes sense if every non-deterministic knob is pinned; without pinning, two CI invocations on the same commit produce different scores, the gate flakes, and engineers learn to ignore its verdicts.

The material for these fixtures is already sitting in your trace backend. Mod-102 taught you to emit a trace for every user interaction; those traces are, essentially, per-request fixtures with the retrieval context, tool arguments, model outputs, and cost/latency numbers attached. What is missing is the **lifecycle** — the process of selecting a trace, freezing the pieces that must stay identical for the replay to make sense, storing the fixture where CI can read it, and updating or retiring it when the surface changes.

Doing this well is what separates an eval suite that catches real regressions from one that catches OpenAI-nightly noise. This chapter walks the shape of a fixture, the four things you always pin, the selection procedure, and the maintenance lifecycle.

## Core concepts

### What is inside a fixture

A fixture is a self-contained record. Anyone should be able to check it out of the repo, run the surface against it locally, and see a comparable score without any additional setup beyond the pinned dependencies. Minimum contents:

- **Input.** The user message, the session state, and any prior turns needed to reconstruct the request. Copy verbatim from the source trace; do not paraphrase.
- **Expected retrieval context** (for RAG surfaces). The ranked chunks the surface pulled at the time the trace was captured, keyed by document id. Chapter 05 explains how a chunk-id list combined with an index-hash pins retrieval.
- **Reference outputs.** Whatever ground truth your rubric compares against — a reference answer, an acceptable-answer regex, a set of required citations, a tool-call trajectory the agent should have produced.
- **Rubric pointer.** The mod-104 rubric this fixture is scored against, by name and version. One fixture can be scored by multiple rubrics; the pointer is the join key.
- **Pinned run environment.** The four values from the next section: model id + snapshot, sampling seed, retrieval index hash, tool-stub set. Without these, the fixture is not reproducible.
- **Provenance metadata.** The trace id it was derived from, the date captured, the reason it was added (bug link, canary regression, edge case), the owner. Chapter 03 uses the owner field for the runbook contract.

A fixture is a plain file — YAML or JSON on disk, one file per fixture, checked into the repo alongside the eval config. Storing fixtures in a vendor DB works too, but the on-disk pattern is the one that lets a code reviewer diff a fixture change in the same PR that changes a prompt.

### The four things you always pin

The stochasticity of an LLM run has four independent sources. Pin all four, or the replay is unrepeatable.

1. **Model version + snapshot.** Not `gpt-4o` — `gpt-4o-2024-11-20`. Not `claude-3-5-sonnet` — `claude-3-5-sonnet-20241022`. Vendors ship silent updates behind the alias; the alias is a moving target and cannot be the pin. For self-hosted models, pin the weights hash (SHA-256 of the checkpoint file), the runtime (vLLM version, TensorRT-LLM version), and the quantisation. `mod-102`'s `gen_ai.request.model` / `gen_ai.response.model` attributes are where these come from — use them.
2. **Sampling seed and decoding parameters.** Temperature > 0 is not deterministic; even temperature = 0 can be non-deterministic across vendors due to batching effects (this is documented in the OpenAI API docs — see <https://platform.openai.com/docs/api-reference/chat/create> for the `seed` parameter and the "system_fingerprint" field, which the docs describe as best-effort). Record the seed, the temperature, the top-p, and (crucially) the `system_fingerprint` returned by the vendor if it is available. If the fingerprint changes on the vendor side, the pin has moved even if you did not touch the code — chapter 03's threshold policy has to account for this.
3. **Retrieval index hash.** For a RAG surface, the same query against a different index returns different chunks. Record a hash over the corpus contents *and* the index build parameters (chunker version, chunk size, embedding model + version, ANN algorithm + parameters). The simplest is a SHA-256 over the deterministic concatenation of `(doc_id, doc_version_hash)` for every document in the index. Rebuild the hash whenever the index rebuilds; store it in the fixture and check it at replay time.
4. **Tool stubs / mocks.** If the surface calls external tools (a CRM, a booking API, a database), replays must stub those calls with recorded responses — you cannot re-hit prod every CI run for cost, safety, and rate-limit reasons. Record the exact tool arguments and the exact return payload alongside the fixture, keyed by tool name and argument hash. Chapter 04 shows the Promptfoo `providers` and DeepEval `mock_tool` pattern.

A fixture missing any one of these four is a fixture whose scores you cannot trust across CI runs.

### Sourcing fixtures from production traces

The pipeline that turns a production trace into a fixture has three steps.

**Step 1 — Select.** You do not want every trace; you want the ones a regression would move on. Four selection strategies in decreasing order of importance:

- **Regression back-fills.** Every time a bug or regression is caught in production (mod-107 alerts, on-call incidents, customer reports), the fix PR must include a fixture derived from the trace that reproduced the bug. This is the single largest source of long-term suite value — every incident adds a fixture that would have caught it. Chapter 03's runbook enforces the discipline.
- **Sampled representative traffic.** A stratified sample across surface segments (tenant tier, use case, language, session length). The sampling policy from mod-102 chapter 06 is the source; the replay set is the destination.
- **Edge cases and adversarial inputs.** Product-known edge cases (empty query, extremely long query, ambiguous query, PII in query, tool-call refusal, out-of-scope question). These are usually hand-authored on top of a real trace template.
- **Frontier examples flagged by the judge.** Traces where the mod-104 judge disagreed with itself, was uncertain, or landed near a threshold. These are the drift-detection material.

Size the replay set so a full CI run fits inside the PR-gate time budget (chapter 05 discusses this — a common budget is 5–15 minutes wall-clock for the PR gate, longer for the deploy gate). A 50–200-fixture replay set is normal for a mid-sized surface; specialised sub-suites (e.g., safety) are held separately.

**Step 2 — Freeze.** Query the trace backend for the trace id. Extract input, retrieval context, tool calls with args and returns, model outputs, and metadata. Snapshot the retrieval index hash *at the time the trace was captured*, not the current one. Copy tool responses verbatim. Scrub PII before writing to disk — the mod-102 chapter 06 policy is the source; the fixture must not leak what production sanitised.

**Step 3 — Attach reference material.** The trace tells you what the surface *did*; the fixture also needs what the surface *should have done*. For most rubrics the reference is one of:

- A **reference answer** (used by exact-match / semantic-similarity / faithfulness rubrics).
- A **reference retrieval set** (used by RAG context-precision/recall).
- A **reference trajectory** — the sequence of tool calls a correct run should produce (used by mod-103's trajectory rubrics).
- **No reference at all** — the rubric is reference-free (faithfulness-to-context, style, refusal). Fine; leave the field null.

For traces sampled from happy-path production, the recorded output *is* the reference. For regression fixtures, the reference is what the on-call determined the correct output should have been — this is a judgement call, and it goes in the fixture's provenance field.

### Fixture file shape

A canonical fixture file, one per YAML/JSON:

```yaml
id: fix-ticketing-002
source_trace_id: 4b8a5c7d9e1f2a3b
captured_at: 2026-04-11T09:42:17Z
added_reason: "regression bug INC-4231 — agent hallucinated ticket owner"
owner: eval-team
rubric_refs:
  - groundedness_v3
  - trajectory_v2

pin:
  model: gpt-4o-2024-11-20
  system_fingerprint: fp_a1b2c3d4e5
  temperature: 0.2
  top_p: 1.0
  seed: 42
  retrieval_index_hash: sha256:d34db33f...
  tool_stubs_hash: sha256:c0ffee01...

input:
  session_id_hash: sha256:...
  messages:
    - role: user
      content: "Who owns ticket 4231?"

retrieval:
  index_hash: sha256:d34db33f...
  documents:
    - id: doc-142
      content: "Ticket 4231 is assigned to team platform-oncall..."
      score: 0.81
    - id: doc-91
      ...

tool_calls:
  - name: crm.lookup
    args: {"ticket_id": "4231"}
    stub_response: {"owner_id": "u-97", "team": "platform-oncall"}

reference:
  answer: "Ticket 4231 is owned by platform-oncall (assignee u-97)."
  required_citations: [doc-142]
  trajectory:
    - {kind: LLM, name: plan}
    - {kind: TOOL, name: crm.lookup}
    - {kind: LLM, name: reply}

recorded_output: "Ticket 4231 is assigned to Jane Doe on the analytics team."
```

The fixture is checked into the repo. A prompt change PR that would previously have hallucinated the assignee now runs against this fixture in CI and fails the groundedness and trajectory rubrics before merge. That is the shape the whole module builds toward.

### The fixture lifecycle

Fixtures do not live forever. Four lifecycle events:

- **Add.** New regression fixture on every incident-close PR (this is a hard contract, not a soft one — see chapter 03's runbook). New sampled fixture on the monthly refresh of the sampled sub-suite.
- **Refresh.** When the source of ground truth changes — the retrieval corpus rebuilds, a policy document is updated, a tool schema changes — the fixture's reference material and pinned index hash must be refreshed. The refresh is a code-review-able diff.
- **Retire.** When the surface removes a feature, the fixtures exercising that feature retire with it. Do not leave dead fixtures around — they either fail forever (annoying) or pass trivially (misleading).
- **Quarantine.** When a fixture flakes unresolvably — a vendor snapshot silently rolled and the new snapshot cannot reproduce the pinned output within the threshold — quarantine it into a separate sub-suite with a `known_broken` flag, file an issue naming the owner, and re-admit it when the resolution lands. Chapter 03's runbook covers the mechanics; the important rule is that **quarantine is time-boxed** and shows up on the on-call review, so drift does not accumulate silently.

### Golden files, snapshot testing, and the "insta-approve" trap

Snapshot testing (Jest's `toMatchSnapshot`, `insta` in Rust) has a well-known failure mode: when a snapshot fails, developers hit "update" without reading the diff. Trace fixtures have the same risk in more dangerous form — updating a "recorded_output" field to whatever the current model produces makes the fixture pass again, and permanently lowers the bar.

Two rules to prevent this:

- **Never store the current model output as ground truth without human review.** The `reference` field is what the surface *should* produce; the `recorded_output` field is what it *did* produce at capture time. They are separate on purpose. Bulk-refreshing `reference` from `recorded_output` in a PR is the tell that the pattern has broken.
- **A fixture update is a semantic change that requires a rubric-aware reviewer sign-off.** The PR review checklist (chapter 03) treats fixture-diff PRs the same as rubric-diff PRs — a member of the eval-owner list on the CODEOWNERS file has to approve.

### Fixture storage: file, table, or vendor?

Three storage choices, each with a trade-off:

| Storage | Pros | Cons |
|---|---|---|
| **YAML/JSON in repo** | Diffable, code-reviewable, portable across CI runners; no auth to configure | Grows the repo; large binaries (audio, images) are awkward |
| **Vendor dataset** (Braintrust datasets, Langfuse datasets, LangSmith datasets) | Rich UI, versioning, integration with the vendor's eval runner | Diff is opaque unless the vendor exposes it in the PR; portability locked to the vendor |
| **Object storage** (S3, GCS) with a manifest file in repo | Handles large binaries, still diffable at the manifest level | Auth setup on every runner; the fixture change is not visible in the PR diff |

The default recommendation for a text-only surface is **YAML/JSON in the repo**, with a manifest-plus-object-store option for surfaces with large binary payloads. Chapter 04's Promptfoo config assumes the on-disk pattern; the Braintrust and Langfuse configs demonstrate the vendor-dataset pattern.

### Determinism is a spectrum, not a guarantee

Even with all four pins in place, exact determinism is not guaranteed — vendors document this. OpenAI's own guidance around `seed` says it is best-effort; on-prem inference is more deterministic than API but is not free of it (batch composition, kernel non-associativity in floating-point). The correct posture:

- **Assert on distribution, not on exact string.** Rubrics score against a criterion; the criterion tolerates textual variation. An exact-match assertion in an LLM eval gate is nearly always the wrong assertion.
- **Score against a threshold with a stated tolerance.** Chapter 03 walks the threshold shapes. Even a well-pinned run will vary within a small window — the tolerance is what makes the gate stable.
- **Log the pin values on every run and diff them.** If the vendor's `system_fingerprint` changes between two runs on the same commit, the gate should surface that in the report — it explains a score shift the reviewer would otherwise have blamed on the code.

## Summary

- A **fixture** is a per-request unit an eval gate scores: input, retrieval context, tool stubs, reference outputs, rubric pointer, pinned run environment, provenance.
- The **four pins** are model + snapshot, sampling seed + decoding parameters (record `system_fingerprint` too), retrieval index hash, and tool-stub set. Miss any one and the gate flakes.
- Source fixtures from real production traces via four selection strategies: regression back-fills (the single largest source of value), sampled representative traffic, hand-authored edge cases, and judge-uncertainty flagged frontier examples.
- The fixture lifecycle has four events: add, refresh, retire, quarantine — and quarantine is time-boxed and audited.
- Do not conflate `recorded_output` with `reference`. Fixture updates are semantic changes and require rubric-aware review.
- Even with pins, LLM output is not exactly deterministic. Score against thresholds with stated tolerances (chapter 03), and log pin values on every run so drift is visible.

Chapter 03 turns this replay set into a gate: the pre-registered pass/fail thresholds, the regression-as-blocker policy, and the owner-assigned runbook.
