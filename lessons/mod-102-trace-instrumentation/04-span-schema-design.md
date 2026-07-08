# Designing the Span-Attribute Schema for LLM Apps and Agents

## Motivation

A convention (chapter 02) is a vocabulary. A backend (chapter 03) is a viewer. The **schema** is the contract *your* app agrees to emit against — which spans exist, which attributes each one carries, what content goes where, what gets hashed, what gets truncated, and what the root span always carries so downstream mod-107 sampling and mod-110 lineage can find it later.

Getting the schema right is the single most leverage-positive step in this module. A well-shaped schema means:

- The trace-viewer workflow (chapter 05) walks the tree top-down without guessing.
- The sampling policy (chapter 06) has real signals to filter on.
- The online-eval loop (mod-107) can attach judge scores back to a specific step, not a whole session.
- The RAG-triad eval (mod-105) reads retrieved documents and generation content off the trace directly.
- The cost / latency dashboards (mod-111) roll up from spans without an ETL job that has to guess semantics.

A poorly shaped schema means the on-call opens the trace, cannot find the attribute they need, and falls back to reproducing the request by hand. Six months in, that turns into a full reinstrumentation pass and a mod-110 migration. It is worth designing well the first time.

## Core concepts

### Start from the SLIs

The schema is a *derivative* of the mod-101 criteria table. For each SLI in that table, ask: *what must the trace record so an offline replay or an online judge can compute this SLI later?* If nothing in the schema supports an SLI, either the schema is incomplete or the SLI is not measurable — both are real findings.

| Family (mod-101) | SLI shape | What the trace has to carry |
|---|---|---|
| Task success | Share of surfaces where the reply / tool call is correct | Final `output.value` on the root span; full LLM `output.value` on the reply span; tool arguments; retrieved docs (for regrounding) |
| Groundedness | Faithfulness score of reply against retrieved context | LLM output content **and** retrieval `.document.content` on the RETRIEVER span |
| Safety | Refusal rate; guardrail-trip rate | GUARDRAIL span with input, output, verdict; refusal reasons; guardrail id + version |
| Cost | Tokens per session, $ per resolved conversation | `gen_ai.usage.*` per LLM span; a `llm.cost.total_usd` roll-up on the root; tool cost (if any) |
| Latency | TTFT, TPOT, p95 end-to-end | Root-span duration; per-LLM-span TTFT / TPOT attributes for streaming |

A schema that satisfies this table for your surface is the target. Everything below is how you get there.

### The span-kind inventory

Pick a span-kind vocabulary from chapter 02 — this schema uses **OpenInference** (`openinference.span.kind`) as the primary — and enumerate the kinds your surface will emit. For a nontrivial multi-tool agent, expect:

| Kind | What it wraps | Typical count per interaction |
|---|---|---|
| `AGENT` | The whole coordinator loop | 1 (root) |
| `CHAIN` | A named sub-pipeline (e.g. "rag_pipeline", "reply_pipeline") | 0–3 |
| `LLM` | Each model call (plan, reply, judge, function-selection) | 2–6 |
| `RETRIEVER` | Each retrieval (vector search, keyword, hybrid) | 0–3 |
| `EMBEDDING` | Each embed call | 0–many (batched) |
| `RERANKER` | Each rerank step | 0–1 |
| `TOOL` | Each tool call | 0–many |
| `GUARDRAIL` | Each pre / post filter check | 0–2 |
| `EVALUATOR` | Online judge scoring attached back to the trace | 0–many (mod-107) |

If a category doesn't apply to your surface, drop it — do not emit empty kinds. If a category applies but you cannot cleanly bound the child (e.g. an SDK auto-instrumentor already emits the LLM span, so you would double-count), rely on the auto-instrumentor's span and only add the outer AGENT / CHAIN wrapper.

### Per-kind minimum attribute sets

The tables below give **MUST** / **SHOULD** / **SHOULD-NOT** columns per kind. MUST is enforced by CI (mod-106); SHOULD is best-practice; SHOULD-NOT is a positive rule — putting these on the wrong span is a common bug. All attribute names below are OpenInference unless prefixed `app.*` (your extension namespace, chapter 02) or `gen_ai.*` (OTel GenAI mirror).

**AGENT (root)**

| MUST | SHOULD | SHOULD-NOT |
|---|---|---|
| `openinference.span.kind=AGENT` | `metadata` — small JSON with surface-specific context | Full LLM prompt content on the AGENT span |
| `app.surface` | `tag.tags` | Full LLM output content on the AGENT span |
| `session.id` | `experiment.arm`, `feature_flag.<name>` | Tool arguments |
| `user.id` (hashed — see below) | `app.step` — e.g., `agent.support_agent` | Retrieved chunk content |
| `service.version` | `llm.cost.total_usd` (roll-up) | Duplicating child-span data |
| `schema.version` | TTFT / TPOT roll-ups when streaming | |
| `input.value` (the user's raw message) | | |
| `output.value` (the surface's final reply) | | |

**LLM**

| MUST | SHOULD | SHOULD-NOT |
|---|---|---|
| `openinference.span.kind=LLM` | `llm.invocation_parameters` (JSON) | Storing retrieved chunk content here (put on RETRIEVER) |
| `llm.model_name` (== `gen_ai.response.model`) | `llm.prompt_template.template` OR a hash | Tool return payloads (put on TOOL) |
| `llm.provider` (== `gen_ai.system`) | `llm.prompt_template.variables` | Human-friendly labels only ("plan") — must also carry a stable `app.step` |
| `llm.input_messages.<i>.message.role/content` | `llm.tools` (JSON of tool schemas) | |
| `llm.output_messages.<i>.message.role/content` | `llm.tool_calls.<i>.tool_call.function.name/arguments` | |
| `llm.token_count.prompt`, `.completion`, `.total` | `app.ttft_ms`, `app.tpot_ms` (streaming) | |
| `gen_ai.response.finish_reasons` | `app.step` (`plan`, `reply`, ...) | |
| `app.step` | `app.cost_usd` | |

**TOOL**

| MUST | SHOULD | SHOULD-NOT |
|---|---|---|
| `openinference.span.kind=TOOL` | `tool.description`, `tool.parameters` | Full LLM prompt |
| `tool.name` | `app.step` | Retrieval scores |
| `input.value` (parsed arguments) | HTTP `url`, status (as span events or attributes per convention) | |
| `output.value` (return) | | |
| `status` (OTel status code — Error on failure) | | |

**RETRIEVER**

| MUST | SHOULD | SHOULD-NOT |
|---|---|---|
| `openinference.span.kind=RETRIEVER` | `retrieval.documents.<i>.document.metadata` | Full LLM completion |
| `input.value` (query) | Reranker score if fused | Tool return |
| `retrieval.documents.<i>.document.id` | `app.retriever_name`, `app.top_k` | |
| `retrieval.documents.<i>.document.content` (capped — see below) | | |
| `retrieval.documents.<i>.document.score` | | |

**EMBEDDING / RERANKER / GUARDRAIL / EVALUATOR** follow the same pattern: `openinference.span.kind`, `input.value` / `output.value`, and per-kind attributes. Guardrail spans should carry a `guardrail.verdict` (`allow` / `rewrite` / `block`) as a MUST — mod-108 measures on it.

### The nine cross-cutting root attributes

Every trace root carries these — they are what makes a trace joinable, filterable, and reproducible.

| # | Attribute | Value | Why |
|---|---|---|---|
| 1 | `session.id` | Stable per-conversation id | Joins sub-traces; ends up in mod-110 |
| 2 | `user.id` | HMAC-SHA256 hash of the raw user id, per-tenant salt | See "user-id-or-hash" below |
| 3 | `app.surface` / `service.name` | `support_agent`, `meeting_summariser`, ... | Cross-surface eval program (mod-112) needs this |
| 4 | `service.version` | Deploy version (e.g. `2026.07.08-a7f2`) | Attribution across deploys; canary filtering |
| 5 | `gen_ai.response.model` (or `llm.model_name`) | The served model version | Attribution across model changes |
| 6 | `llm.prompt_template.template` OR `app.prompt_template.hash` | Full template or a stable hash | Attribution across prompt changes |
| 7 | `experiment.arm`, `feature_flag.<name>` | A/B / canary arm | mod-107 needs this to bisect drift |
| 8 | `app.cost_usd` (roll-up) | Sum of per-LLM `app.cost_usd`, plus tool costs | mod-111 rolls up from here |
| 9 | `app.ttft_ms`, `app.tpot_ms`, span duration | Latency | mod-111 rolls up from here |

If the schema is silent on any of these, an SLI derived from it will not survive a deploy that changes the missing dimension.

### Prompt / response granularity

There is one common bug this section prevents: putting the full prompt and completion on every ancestor span, so that when a trace is opened the same 20 KB of text appears three times in the tree. The rule:

> **Full-fidelity content lives on the LLM span it belongs to.** The CHAIN / AGENT parent gets a *redacted repro summary* on `output.value` (a short human string, e.g. `"Reply drafted, ticket #1247 opened"`) — not the raw model text.

The benefits:

- The tree is legible top-down.
- Content-size caps (below) apply per LLM span; you do not double-store text.
- Filtering by a user's exact input is deterministic (one place to look).
- When a chain has multiple LLM children, the on-call sees which child produced which text.

Same rule for tool arguments (on the TOOL span, not the AGENT parent) and retrieved documents (on the RETRIEVER span, not the parent CHAIN).

### User-id-or-hash

Never store the raw user id or email in a span attribute. Both are PII by GDPR (Regulation (EU) 2016/679, Art. 4(1)) and NIST SP 800-122's definition. The recommendation for `user.id`:

- **HMAC-SHA256 of the raw user id, keyed with a per-tenant salt held in a KMS.**
- The salt never leaves the KMS; a small production-only service performs the HMAC before the id lands on a span.
- The hash is deterministic within the tenant, so cohort aggregations (per-user retention rate, per-user quality) still work.
- The hash is **not** anonymous under GDPR — it is *pseudonymous* (Art. 4(5)). Personal-data obligations still apply; the residual risk is enumerated in your mod-108 / governance-family plan.
- Rotate the salt periodically; keep an audit trail of rotations so a mod-109 human-reviewer can still resolve historic traces if needed.

Do NOT hash without a salt — an unsalted SHA256 of an email is a look-up-table attack.

### Content sizing

Backends will accept large string attributes (Phoenix, Langfuse, Weave, Braintrust, LangSmith all store long content), but every backend has a practical cap and every viewer degrades on multi-hundred-KB payloads. A safe rule of thumb — and it is a rule of thumb, not a spec:

> **Keep any single string attribute under ~64 KB. If the payload exceeds that, write it to blob storage and put a URL on the span.**

The exact cap is backend-specific — check the vendor's doc from chapter 03. The pattern is:

- Configure a max attribute length in your SDK wrapper.
- If a value exceeds it, upload to a blob store (S3, GCS, R2) with a content-addressable key.
- Set the attribute to a URL and a `content.sha256` fingerprint.
- Leave a `content.truncated=true` marker so mod-107 replay can hydrate on demand.

This is also the shape you want anyway for large retrieved-doc content — the RETRIEVER span records the chunk ids and scores plus a truncated preview, and the full text lives in the blob store.

### Versioning the schema

The schema will change. Bake versioning in from day one:

- Emit `schema.version` on every root span (semver — `1.0.0` at launch).
- Bump the minor version on **additive** changes (new attribute, new span kind).
- Bump the major version on **breaking** changes (renamed attribute, changed shape). Do NOT do this casually; every downstream consumer (mod-107, mod-110) has to be updated in lockstep.
- Keep a `SCHEMA.md` in the surface's repo documenting each version, plus a **deprecation window** stating how long the old shape is still emitted.

Feed the schema into mod-110's eval-data platform so the offline lake's table definitions and the online viewers agree on what a span *is*.

### Custom `app.*` extensions

Wherever the conventions are silent, use your own namespace. Common extensions:

- `app.surface`, `app.step` — pipeline step identity.
- `app.retriever_name`, `app.top_k`, `app.hybrid_alpha`.
- `app.guardrail_id`, `app.guardrail_version`.
- `app.judge_score`, `app.judge_model` (when a mod-104 judge attaches to a span).
- `app.cost_usd`, `app.ttft_ms`, `app.tpot_ms`.
- `app.trace_owner_team` — which team owns this surface; useful for cross-team eval programs (mod-112).

Do NOT overload `gen_ai.*` or `llm.*` for these — those namespaces belong to the specs.

## Example: schema for the support-agent surface

The same surface used in chapters 01, 05, 06 — one AGENT root, one LLM plan, one RETRIEVER, one TOOL, one LLM reply, one GUARDRAIL. Six spans per interaction.

**Root span — `agent.support_agent`**

```
openinference.span.kind = "AGENT"
app.surface             = "support_agent"
app.step                = "agent.support_agent"
session.id              = "sess_9e4b…"
user.id                 = "u_" + HMAC-SHA256(raw_id, tenant_salt).hex()[:16]
service.name            = "support_agent"
service.version         = "2026.07.08-a7f2"
schema.version          = "1.2.0"
experiment.arm          = "reply-model-v3"
input.value             = "Where is my order 12345?"
output.value            = "Reply drafted; ticket #1247 opened."   // redacted summary
app.cost_usd            = 0.0021          // roll-up
app.ttft_ms             = 512
metadata                = {"channel": "email", "locale": "en-US"}
```

**Child — `llm.plan`**

```
openinference.span.kind = "LLM"
app.step                = "plan"
llm.provider            = "openai"        // gen_ai.system = "openai" mirror
llm.model_name          = "gpt-4o-2024-08-06"   // gen_ai.response.model mirror
llm.invocation_parameters = {"temperature": 0.2, "max_tokens": 512}
llm.prompt_template.template  = "You are a support planner…"   // OR a hash
llm.input_messages.0.message.role/content    = "system" / "…"
llm.input_messages.1.message.role/content    = "user" / "Where is my order 12345?"
llm.output_messages.0.message.role/content   = "assistant" / "" (tool-call turn)
llm.tool_calls.0.tool_call.function.name     = "docs.search"
llm.tool_calls.0.tool_call.function.arguments = "{\"query\": \"order 12345 status\"}"
llm.token_count.prompt/completion/total       = 842 / 34 / 876
gen_ai.response.finish_reasons                = ["tool_calls"]
app.cost_usd                                  = 0.0009
app.ttft_ms                                   = 512
```

**Child — `retriever.docs.search`**

```
openinference.span.kind = "RETRIEVER"
app.step                = "docs.retrieve"
app.retriever_name      = "hybrid_bm25_dense"
app.top_k               = 3
input.value             = "order 12345 status"
retrieval.documents.0.document.id       = "doc-142"
retrieval.documents.0.document.score    = 0.81
retrieval.documents.0.document.content  = "…truncated preview…"
retrieval.documents.0.document.metadata = {"source": "kb/orders", "version": "2026-06-30"}
retrieval.documents.1.document.id       = "doc-91"
retrieval.documents.1.document.score    = 0.78
...
```

**Child — `tool.open_ticket`**

```
openinference.span.kind = "TOOL"
app.step                = "open_ticket"
tool.name               = "open_ticket"
tool.description        = "Open a support ticket in Zendesk"
input.value             = {"order_id": "12345", "reason": "no_shipping_update"}
output.value            = {"ticket_id": 1247, "status": "open"}
status                  = OK
```

**Child — `llm.reply`**

```
openinference.span.kind = "LLM"
app.step                = "reply"
llm.model_name          = "gpt-4o-2024-08-06"
llm.input_messages.<...> // includes retrieved docs as system context
llm.output_messages.0.message.content = "Ticket #1247 has been opened…"
llm.token_count.prompt/completion/total = 1128 / 83 / 1211
gen_ai.response.finish_reasons          = ["stop"]
app.cost_usd                            = 0.0012
```

**Child — `guardrail.output_policy`**

```
openinference.span.kind = "GUARDRAIL"
app.step                = "guardrail.output"
app.guardrail_id        = "policy_v14"
app.guardrail_version   = "14.2"
input.value             = "Ticket #1247 has been opened…"
output.value            = "Ticket #1247 has been opened…"     // unchanged
app.guardrail_verdict   = "allow"
```

### Mapping SLIs to attributes

For the mod-101-style eval plan on this surface, the mapping table:

| SLI | Computed from |
|---|---|
| Task success (correct ticket opened for shipping issue) | `tool.open_ticket.input.value` + expected ticket schema |
| Groundedness (reply cites retrieved doc content) | `llm.reply.llm.output_messages.0.message.content` ⋈ `retriever.docs.search.retrieval.documents.*.document.content` |
| Refusal correctness | `guardrail.output_policy.app.guardrail_verdict` per family |
| Cost | Root `app.cost_usd` (== sum of LLM `app.cost_usd`) |
| End-to-end latency | Root span duration |
| TTFT (streaming) | Root `app.ttft_ms` (== `llm.reply.app.ttft_ms`) |

If a row in this mapping is empty, the schema needs another attribute — that is the whole point of building the mapping table.

## Summary

- The schema is a contract derived from the mod-101 SLIs. Every SLI must have an attribute (or a set of attributes) that lets it be computed later — offline or on a sampled trace.
- Emit one **span kind** per structural role (AGENT / CHAIN / LLM / TOOL / RETRIEVER / GUARDRAIL / ...) using OpenInference or OTel GenAI conventions.
- Enforce a **per-kind attribute table** with MUST / SHOULD / SHOULD-NOT columns; CI (mod-106) enforces MUST.
- The **root span** carries nine cross-cutting attributes (session, user hash, surface, version, model, prompt hash, arm, cost roll-up, TTFT / TPOT).
- Prompt / response content lives on the LLM span it belongs to; the parent gets a **redacted summary**, not a copy.
- **User ids** are HMAC-SHA256 with a per-tenant salt in KMS. Pseudonymous, not anonymous.
- Cap string attributes (~64 KB rule of thumb); overflow to blob storage with a URL + sha256 on the span.
- Bake in `schema.version`; version additively; hand the schema to mod-110.
- Custom fields live under `app.*`; never overload `gen_ai.*` or `llm.*`.

Chapter 05 uses this schema to walk a bad run top-down.
