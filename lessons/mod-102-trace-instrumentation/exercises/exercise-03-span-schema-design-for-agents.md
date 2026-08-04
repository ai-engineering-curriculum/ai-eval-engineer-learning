# exercise-03: Span-Schema Design for Agents

**Estimated effort:** 3 hours

## Objective

Produce the **span-attribute schema** for a real (or realistic) multi-step agent surface. The schema is the contract everything downstream in the track — `mod-106` CI enforcement, `mod-107` online eval, `mod-109` review routing, `mod-110` lineage — will read against. The output is a document another engineer could implement to; it is not code (that comes in the labs).

By the end you will have a MUST / SHOULD attribute table per span kind, the nine root-span attributes filled in for your surface, an SLI-to-attribute mapping, and a versioning + deprecation policy. This is a design exercise; the value is in the trade-offs you write down, not the number of rows.

## Prerequisites

- Chapters 02 and 04 of this module.
- Your `mod-101` criteria table for a surface. If you did not do the `mod-101` exercises, pick one of the three specs from `mod-101/exercises/exercise-01` and produce a minimal criteria table before starting here — the SLIs are the entire input to the schema.
- Exercises 01 and 02 recommended but not blocking.

## Choose a surface

Pick one — the same one you used in `mod-101/exercise-01` is ideal. If you're starting fresh, one of:

- **Support agent** with plan, RAG over docs, one `open_ticket` tool, reply, output guardrail. (The worked example in chapter 04.)
- **Meeting-summariser agent** with retrieval over transcript, one `create_action_item` tool, reply, and PII scrubbing on names.
- **Coding assistant** with plan, retrieval over the repo, one shell tool with a whitelist, a compile-and-test tool, and reply.
- **A real surface you own at work,** with employer authorisation. Redact anything sensitive before committing files.

## Requirements

Produce a single directory `mod-102/exercise-03/` in your working repo containing:

1. **`SCHEMA.md`** — the schema document itself. Structure required, but the content is yours:
   - **Section 1: Surface context.** One paragraph naming the surface, its `app.surface` value, its intended traffic shape, the framework in use (LangChain / LlamaIndex / homegrown), and the chosen primary convention (OpenInference or OTel GenAI).
   - **Section 2: Span-kind inventory.** A table (kind, count-per-interaction, wraps-what). Include every kind you will emit; explicitly note any kind you deliberately omit and why.
   - **Section 3: Per-kind attribute tables.** One MUST / SHOULD / SHOULD-NOT table per kind (AGENT, LLM, TOOL, RETRIEVER, GUARDRAIL, and any others). Every attribute name must be a real OpenInference / GenAI attribute or a namespaced `app.*` / `experiment.*` / `feature_flag.*` extension. Include example values in a column.
   - **Section 4: The nine cross-cutting root attributes.** A table with the nine rows from chapter 04, each row filled with the concrete value or value-shape your surface will emit.
   - **Section 5: Content-granularity rule.** State where full prompt / retrieval content lives, what the AGENT parent carries as a redacted `output.value` summary, and how large content overflows to blob storage. Give the exact `content.truncated` / `content.sha256` / URL fields you will use.
   - **Section 6: SLI-to-attribute mapping.** A table with one row per SLI from your `mod-101` criteria table and a column showing which span / attribute expression it is computed from. Rows with no attribute expression are unresolved; either add attributes to earlier sections or send the row back to `mod-101` as a non-measurable SLI.
   - **Section 7: User-id and PII handling.** One paragraph on `user.id` hashing (HMAC-SHA256 with per-tenant salt), one paragraph listing the attribute keys that will be scrubbed at the application vs at the Collector. Full policy is exercise-05; here you fix the *keys*, not the runtime.
   - **Section 8: Versioning and deprecation.** Semver rule for `schema.version`, the current version at launch, an example additive change vs breaking change, and the deprecation window for renamed attributes.
2. **`schema-diff-from-baseline.md`** — 200–400 words. Diff your schema against chapter 04's support-agent example. Where you agree, where you diverge, and why. If the surface is very different from the support agent (e.g., no RAG), the diff can be short — but write it down; it forces you to justify each deviation.
3. **`missing-slis.md`** — a bulleted list of SLIs from your criteria table for which the schema *cannot* produce evidence today, and the smallest schema change that would close the gap. Empty is a legitimate answer if every SLI is covered; explicitly say so.

## Starter guidance

- Sketch the trace tree on paper first (like chapter 01's worked call tree) before you touch attribute tables. If you cannot draw the tree, the schema below it will be arbitrary.
- Do MUST columns first, then SHOULD, then SHOULD-NOT. The SHOULD-NOT column is the most under-used and the most useful — chapter 04 lists common cases (LLM content on the parent, tool return on the AGENT, etc.).
- Namespace ruthlessly. Any attribute you cannot cite from OpenInference or the GenAI spec goes under `app.*`. Do NOT create a `llm.foo` or `gen_ai.foo` attribute of your own — that will collide with future spec evolution.
- Test the SLI-mapping table by reading each row aloud: "Faithfulness is computed by comparing `llm.reply.output_messages.0.message.content` against `retriever.docs.search.retrieval.documents.*.document.content`." If you cannot say the sentence, the mapping is wrong.
- Every value in Section 4 should be plausible for your surface, not for chapter 04's surface. Fictional deploy versions are fine (`2026.08.04-abc1`), but the *shape* has to match your real deploy convention.
- Do not exceed ~40 total MUST rows across all kinds. A schema that CI must enforce every field on for every trace has a real per-request cost; MUST is expensive.
- Content sizing is a rule, not a wish. State the concrete cap (chapter 04 suggests ~64 KB) and the overflow procedure.

## Acceptance criteria

You are done when:

- `SCHEMA.md` has all eight sections filled in for your chosen surface.
- Every attribute name in Section 3 is either an OpenInference / GenAI attribute you can cite from a spec URL, or a namespaced `app.*` / `experiment.*` / `feature_flag.*` extension.
- Every SLI in your `mod-101` criteria table has a row in Section 6, either mapped to an attribute expression or listed in `missing-slis.md`.
- Section 5's content rule is *specific*: which span holds full content, what the parent carries, and the exact blob-storage overflow fields.
- Section 7 names the hashing algorithm and the salted-key procedure explicitly (HMAC-SHA256, per-tenant salt in KMS).
- `schema-diff-from-baseline.md` explains each divergence from chapter 04.
- The document reads in under 15 minutes for someone who has read chapters 02 and 04.

## Stretch goals

- Add a **cardinality analysis** column to Section 3 estimating the number of distinct values each attribute will take per week. Flag any attribute whose cardinality would blow out an attribute-based sampler or a metric dimension.
- Write a `schema.json` — a machine-readable version of the MUST tables as JSON Schema (draft 2020-12) with `additionalProperties: false` per kind. The labs will consume this in the CI validator.
- Draft the mod-106 CI gate that would enforce Section 3's MUST columns on a representative sample of ~50 recorded traces. Do not implement — just the pseudocode and the assertions.
- Add a **`retriever_v2` migration example**: assume you are moving from a single-vector retriever to a hybrid retriever with a rerank step. Show the additive schema change (`RERANKER` kind, `retrieval.documents.<i>.document.rerank_score`), the `schema.version` bump, and the deprecation window.

## What this exercise does *not* cover

You are not implementing the instrumentation code — the lab does that. You are not enforcing the MUST columns in CI — `mod-106` does that. You are not writing the sampling or scrubbing policy runtime — exercise-05 does that. Focus on the contract.
