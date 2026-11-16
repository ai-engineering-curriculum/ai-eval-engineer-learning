# Expert-Review Workflows in Argilla and Label Studio

## Motivation

The three inbound queues from chapter 01 (judge disagreements, online-loop alerts, safety-eval survivors) all land as *rows waiting for a labelled verdict*. The workbench is the operator surface that turns those rows into labelled data without the eval team writing one from scratch. Two tools dominate the OSS space for this shape — **Argilla** (owned by Hugging Face since the 2024 acquisition; <!-- needs-research: re-verify Argilla's current maintainer and the current major version at https://docs.argilla.io/ --> ) and **Label Studio** (owned by HumanSignal; <!-- needs-research: re-verify the latest Label Studio Community and Enterprise capabilities at https://labelstud.io/ --> ). Both ship a web UI, a Python SDK, task templates, and multi-reviewer support. Either one, wired to your trace store with a clean schema, is sufficient for a product team.

What is *not* sufficient is a review workbench whose rows are plain `(prompt, response)` strings with no link back to the span they came from. Two grounded reasons the lineage columns are load-bearing:

- **A reviewed row is only useful if you can replay it.** A reviewer verdicts a response as "unfaithful to the retrieved context" — the mod-104 judge is going to be re-calibrated against this row. If the row does not carry the `retriever_hash`, the `model_version`, the `prompt_hash`, and the `decoding_seed`, nobody can re-run the exact same turn to check whether a code change fixed it. The verdict becomes a timestamped anecdote.
- **Drift investigations run backwards from the row.** The on-call opens the drift alert, follows the sampled traces into the workbench, reads the reviewer verdicts, and wants to know which `model_version × prompt_hash × retriever_hash` combination the verdicts cluster on. If the row only carries the final strings, the investigation falls back to join-by-prompt-text — expensive and lossy.

This chapter walks the workbench surface and what it has to do for a product team, the labelled-row schema that both Argilla and Label Studio export cleanly, the trace-store integration (the lineage columns that make every reviewed row replayable), the reviewer rota and SLA, the export back into the mod-110 eval data platform, and the trade-offs between the two tools.

## Core concepts

### What the workbench has to do, at minimum

A product-team review workbench must support five operations. Both Argilla and Label Studio cover all five; a purpose-built in-house tool would re-implement them.

- **Project per rubric, per surface.** A judge-disagreement project for the faithfulness rubric on `/v1/support-bot/chat` is a different project from the same rubric on `/v1/search-answer`. Merging them collapses the drift signal into one noisy table. Keep them separate.
- **Row schema with structured fields.** `prompt`, `response`, optional `retrieved_context`, optional `tool_calls`, plus the lineage columns, plus the rubric-specific label fields. The label fields must match the rubric vocabulary exactly — the same labels the mod-104 rubric uses, the same OWASP tag enumeration mod-108 produced.
- **Multi-reviewer labelling with disagreement visible.** Two or three reviewers label the same row; the workbench surfaces disagreements to a senior reviewer who adjudicates. Chapter 04 computes IAA on the two-reviewer labels *before* adjudication.
- **Programmatic import and export via SDK.** The eval team runs the import nightly from the drift monitor and the judge, and runs the export nightly into the eval data platform. The web UI is for reviewers; the pipeline is code.
- **Per-row audit trail.** Who labelled, when, which labels they changed, which comments they left. The audit trail is the primary evidence when a reviewer verdict is contested in a release-gate post-mortem.

### The labelled-row schema

The schema the chapter formalises has three bands of columns.

- **Content columns** — what the reviewer sees in the UI. `prompt`, `response`, `retrieved_context` (as a list of chunks with retrieval scores), `tool_calls` (as a list of `{name, args, result}` tuples), optional `metadata.user_cohort` and `metadata.surface`.
- **Lineage columns** — what makes the row replayable. `trace_id`, `span_id`, `parent_span_id`, `model_version`, `prompt_hash`, `retriever_hash`, `decoding_seed`, `policy_version`, `rubric_version`. Every one is required; no row enters the workbench without them.
- **Label columns** — what the reviewer writes. `label` (nominal or ordinal per the rubric), optional `reason_code` (a bounded enum — "retrieval missing citation", "retrieval cited wrong chunk", "model hallucinated absent citation"), free-text `comment`, `reviewer_id`, `labelled_at`, `label_time_s`.

Two disciplines make this defensible:

- **Enumerations before free text.** `reason_code` is an enum, not a text field. Reviewers who want to write a new reason file a rubric revision; the enum is only extended through the rubric change process. Free-text `comment` is for the margin, not for the signal.
- **One rubric per project, not one row.** A row cannot be labelled against two rubrics in the same project. The mod-104 rubric router decided which rubric this row is for *before* the row entered the workbench. Rubric-selection debates happen upstream.

### Trace-store integration — the lineage columns in detail

Every reviewed row must round-trip to the mod-102 trace store. Concretely, the import job that creates rows in the workbench runs this shape (pseudocode):

```python
def import_judge_disagreements(since_ts):
    for row in judge_disagreement_rows(since_ts):
        span = trace_store.get_span(row.trace_id, row.span_id)
        # lineage columns from the span, not from the judge
        workbench.create_row(
            project=row.surface_id + ":" + row.rubric_id,
            content={
                "prompt": span.input.text,
                "response": span.output.text,
                "retrieved_context": span.retrieval.chunks,
                "tool_calls": span.tool_calls,
            },
            lineage={
                "trace_id": span.trace_id,
                "span_id": span.span_id,
                "parent_span_id": span.parent_span_id,
                "model_version": span.attributes["gen_ai.response.model"],
                "prompt_hash": span.attributes["eval.prompt_hash"],
                "retriever_hash": span.attributes["eval.retriever_hash"],
                "decoding_seed": span.attributes["gen_ai.request.seed"],
                "policy_version": row.policy_version,
                "rubric_version": row.rubric_version,
            },
            label_schema=rubric_schema_for(row.rubric_id),
        )
```

Four invariants hold:

- **The span is the source of truth for lineage.** The judge does not get to invent a lineage column; it reads it from the span. If the span lacks a column, the row cannot enter the workbench — the import fails loudly, the mod-102 instrumentation gets a fix, the import re-runs.
- **The rubric version is on the row.** When the rubric revises, every row labelled against the old version is marked `rubric_version=v11`; new rows get `v12`. Re-running an old IAA number requires the old rubric; the sealed rubric sits in the mod-110 data platform alongside the row.
- **The policy version is on the row (for safety rows).** Mod-108's chapter 06 reconciliation attaches the `safety_policy_v11` tag; the workbench preserves it. When the policy rotates, the old row stays labelled against the old policy.
- **Dropping a span orphans its row.** If retention policy in mod-102 drops a span, the workbench row loses its ability to replay. Either the retention policy exempts spans with linked workbench rows or the workbench export snapshots the content at the time of review. Chapter 05's sealed-snapshot discipline covers the second mode.

### Reviewer rota, SLA, and backlog pressure

The workbench does not do reviewer management; the eval team does. Three artefacts live outside the tool and feed into it.

- **The rota.** A named reviewer per project per week. A primary reviewer and a secondary reviewer (for IAA), plus a senior adjudicator who closes disagreements. The rota rotates monthly so that no reviewer is the sole authority on a rubric. Chapter 06's vendor option replaces the primary reviewer with a vendor pool; the senior adjudicator stays in-house.
- **The SLA.** Time to first label for a row entering the queue, per source. Safety-eval survivors: 24 hours (the mod-108 runbook depends on it). Judge disagreements: 72 hours (mod-104's quick-calibration cadence tolerates the latency). Online-alert escalations: 24 hours during an active incident, 72 hours otherwise. User-feedback rows: typically lax (several days to a week). The SLA is reported as p50 / p90 / p99 on the review path and published on the mod-107 safety-reviewer dashboard.
- **The backlog cap.** A ceiling on the number of pending rows per project. When the backlog exceeds the cap, the eval team either halts imports (the judge keeps running but stops filing deferrals; the mod-104 calibration slows) or recruits a vendor pool (chapter 06). A backlog that grows unchecked is a dead queue in six months.

Two anti-patterns the module avoids:

- **One mega-queue.** Some product teams try to pool every row into a single project and rotate reviewers across it. The IAA signal is noise, the SLA cannot be enforced, and the mod-104 calibration collapses into one global rubric. Keep the queues per surface, per rubric.
- **No SLA, just a weekly review.** "Someone will look at the backlog on Fridays" is not an SLA. The queue grows between Fridays; the mod-107 disposition metric is unreadable; the mod-108 safety carve-out loses meaning. Publish p50 / p90 / p99 or the SLA does not exist.

### Export back into the eval data platform

The workbench is not the system of record; mod-110 is. Export runs nightly (continuously for an on-prem Argilla install; via SDK for Label Studio Community; via the Enterprise connector for Label Studio Enterprise).

The export delivers four tables into mod-110:

- `reviewed_rows` — one row per labelled item, with content, lineage, and label columns.
- `reviewer_actions` — one row per label change, with `reviewer_id`, `before`, `after`, `at`.
- `disagreements` — one row per two-reviewer disagreement, with `adjudicator_verdict` if adjudicated.
- `project_meta` — one row per project with the rubric version, the SLA settings, and the current backlog size.

Mod-104 reads `reviewed_rows` for its calibration step; mod-107 reads `project_meta.backlog_size` and the SLA-latency numbers for its safety-reviewer dashboard; mod-106 reads `reviewed_rows.labels` as the ground truth for the release-gate report. All reads are through mod-110, not through the workbench API directly; the workbench is swappable without breaking the downstream.

### Argilla vs Label Studio — the trade-offs

Both tools are production-ready; the choice is about fit for a product team's deployment posture and vocabulary.

| Axis | Argilla | Label Studio |
|---|---|---|
| **Owner** | Hugging Face (acquired Argilla in 2024) <!-- needs-research: re-verify the acquisition date and current corporate stewardship at https://huggingface.co/blog/argilla --> | HumanSignal |
| **OSS licence** | Apache 2.0 | Community edition Apache 2.0; Enterprise is commercial |
| **Hosted option** | Argilla on Hugging Face Spaces; also self-host | Label Studio Enterprise cloud and on-prem; Community self-host |
| **Primary audience** | LLM / NLP teams; feedback-task templates for ranking, classification, and preference | Multi-modal (text, image, audio, video, time-series); strong on computer-vision provenance |
| **SDK** | `argilla` Python SDK; strong pandas-ish ergonomics | `label-studio-sdk` Python SDK; task-oriented |
| **LLM-feedback templates** | First-class (preference pairs, chat-response rating, span annotation on LLM outputs) | Available but less idiomatic for LLM feedback shapes |
| **Multi-reviewer** | Built in (team workspaces, reviewer roles, disagreement surface) | Built in (annotators, reviewers, ground-truth flag) |
| **Export** | Python SDK → pandas / HF datasets; also direct parquet | SDK → JSON / CSV / parquet; Enterprise ships a dedicated connector |
| **On-prem posture** | Docker / Kubernetes; straightforward to deploy | Docker / Kubernetes; Enterprise ships hardened images |
| **Hosted PII posture** | Spaces respects the Space's data-handling settings | Community self-hosts; Enterprise cloud ships SOC 2 + enterprise auth <!-- needs-research: re-verify Label Studio Enterprise's current compliance certifications at https://humansignal.com/security/ --> |

Three heuristics for the pick:

- **If your product is LLM-only and your reviewers already use Hugging Face**, Argilla has the lower friction — the task templates match the shape of a chat turn, and the SDK is idiomatic for the `pandas`-shaped data you already have. The Hugging Face Spaces hosted option works for small teams; the on-prem install works for teams with data-residency requirements.
- **If you need multi-modal reviews or your pipeline already includes vision / audio / document labelling**, Label Studio is the better fit. The LLM-feedback shape is possible but not idiomatic; you pay for the extra configuration in schema plumbing.
- **If your legal / infosec posture requires SOC 2-audited hosting**, Label Studio Enterprise is the more mature answer; Argilla's hosted option is younger. Self-host either tool on your own infrastructure to short-circuit the compliance conversation entirely.

The hosted-vs-OSS choice is independent of the tool choice. A product team with strict data-residency (chapter 06 covers this at length) typically self-hosts whichever tool it picks and treats the vendor's hosted option as a contingency.

### The quick failure modes to spot early

Three failures that commonly derail a new workbench setup.

- **Reviewer turnover before IAA is computed.** A reviewer onboards, labels for two weeks, leaves. Chapter 04's IAA was run on the first week's rows only. The second-week rows no longer have a measured agreement. The rubric-stability signal lies. Mitigation: run IAA nightly on the trailing-week window, not as a one-off report.
- **Rubric version inflation.** The rubric revises twice a month; every revision splits the dataset into "before" and "after"; the mod-104 calibration never has a clean window. Mitigation: version the rubric with discipline (monthly cadence, chapter 04 kappa confirmed before shipping, a `rubric_version` column on every row).
- **Workbench-as-system-of-record.** The team starts storing feature-request issues, bug reports, and policy drafts in the workbench because the UI is convenient. The export job now ships mixed data into mod-110. Mitigation: hard rule — the workbench stores labelled rows and nothing else. Issues go to the issue tracker; policy drafts go to the policy repo.

## Summary

- The workbench has to do five things: project per rubric-per-surface, structured schema, multi-reviewer labelling, programmatic import / export, audit trail. Both Argilla and Label Studio cover all five.
- The labelled-row schema has three bands: **content columns**, **lineage columns**, **label columns**. Lineage is non-negotiable; a row without `trace_id`, `span_id`, `model_version`, `prompt_hash`, `retriever_hash`, `decoding_seed`, `policy_version`, `rubric_version` is a row nobody can replay.
- The trace-store integration reads lineage from the span, fails loudly when a column is missing, and keeps the rubric and policy version on the row so old numbers stay reproducible.
- The **rota**, the **SLA**, and the **backlog cap** live outside the tool. SLA is per source (24h safety survivors, 72h judge disagreements, laxer for user feedback). Backlog cap halts imports or triggers a vendor-pool escalation.
- Export runs nightly into mod-110 as four tables (`reviewed_rows`, `reviewer_actions`, `disagreements`, `project_meta`). Downstream reads go through mod-110, not the workbench API.
- **Argilla** fits LLM-only product teams and teams already on Hugging Face; **Label Studio** fits multi-modal pipelines and teams with Enterprise compliance requirements. Self-host either when data-residency is strict.
- Three early failure modes: reviewer turnover before IAA is computed, rubric version inflation, workbench-as-system-of-record. Mitigations exist for each.

Chapter 03 walks the other capture path — the thumbs widget, the structured form, and the free-text report that bring user feedback into the same shape.
