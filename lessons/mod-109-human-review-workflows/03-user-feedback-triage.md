# Capturing, Deduplicating, and Triaging User Feedback

## Motivation

The expert-review path in chapter 02 is deliberate: the eval team *picks* which rows a reviewer sees. User feedback is the opposite shape — the user picks. A thumbs-down on a response, a "this is wrong" form submission, a "please stop doing X" free-text report. Volume is higher; signal density is lower; the owner of each row is harder to decide. And unlike the expert review, a user-feedback row is not aimed at a rubric — the user is telling you something is wrong without knowing which rubric the wrong thing belongs to.

Three grounded reasons the user-feedback path is load-bearing for the eval program.

- **User feedback is the only ground truth for whether the surface is useful.** The automated suite and the expert panel both score against written rubrics. A rubric is a model of what the product owner thinks is useful; a user telling you "this response did not solve my problem" is evidence that the model is wrong somewhere. If the eval program never ingests user feedback, the rubric never gets corrected against the real surface.
- **Volume is non-linear with signal.** 95 % of user feedback on a mature product is duplicate ("the recommended playlist is bad", filed by 400 users on different days about different playlists). Without a dedup layer, every row is a noise source; with a dedup layer, each cluster is a product bet to make. Chapter 03's dedup is the difference between a pipeline that enriches the eval program and a pipeline that buries the on-call.
- **Not all feedback is eval signal.** Users file bug reports ("the button doesn't load"), feature requests ("I wish this could do X"), and abuse ("this said something I disagree with politically"). Each needs a routing arrow, and the arrow is not "add row to eval dataset" for any of the three. The filter has to run before the dedup.

This chapter walks the capture layer (thumbs, structured forms, free-text), the dedup layer (near-duplicate detection and per-session collapse), the routing into the eval dataset with per-surface owner, severity, and SLA, and the loopback into the mod-107 online-eval dashboards.

## Core concepts

### The three capture shapes and where each belongs

Three widgets at the product surface produce three shapes of feedback. The schema must distinguish them; the downstream treatment is different for each.

- **Thumbs-up / thumbs-down.** A single-bit signal attached to a specific turn (a chat response, a search result, a tool invocation). High volume, low per-row information. The capture widget sends `{trace_id, span_id, direction, user_id_hash, surface_id, ts}`. The thumb is anchored on the span the user just saw — the widget is not a global rating.
- **Structured feedback form.** A short form with bounded fields: category (dropdown, 5 – 10 options tied to the written rubric categories), severity (`low | medium | high`), a free-text comment. Fires either voluntarily (always-visible "report this response") or on a thumbs-down (progressive disclosure: "what went wrong?"). Lower volume than thumbs, much higher per-row information.
- **Free-text report.** A support ticket, an in-app feedback box, an email. No structure beyond the message body and metadata. Lowest volume, hardest to process, often richest. Typically arrives through a separate channel (support) and is ingested into the eval pipeline on a slower cadence.

Each capture shape writes into the same `user_feedback` event stream (one topic in the mod-102 instrumentation plane) with a `shape` discriminator. Downstream consumers branch on the discriminator; the dedup, triage, and routing logic is shared.

### The instrumentation contract — thumbs ride the span

The user-feedback event is *part of* the trace, not a parallel stream. Concretely:

- The widget is wired to the mod-102 client-side instrumentation. When the user taps thumbs-down, the client emits a span (or appends an attribute to the turn's span) with `event.name = "user_feedback"`, `user_feedback.direction = "down"`, `user_feedback.shape = "thumb"`. The span inherits the trace_id of the turn being rated.
- The server-side backend receives the event through the same OTEL pipeline that carries the rest of the trace (mod-102 chapter 02) and lands it in the trace store with the turn.
- The `user_feedback_events` table in mod-110 is a projection of those spans — a query like "every span where `event.name = user_feedback`" joined to the originating turn's lineage columns.

Two invariants hold:

- **No sampling on feedback events.** Mod-102 chapter 06 sampled traces to control cost; feedback events are *always* retained. A user who taps thumbs-down cares that their signal counted. Drop the whole trace if you must (per the retention policy) but never drop the feedback event alone.
- **PII is redacted at capture.** A free-text feedback field is a PII source — users paste email addresses, phone numbers, their own content. Mod-102 chapter 06's PII policy runs on the free-text field before the event enters the eval pipeline. Rejected / redacted rows are flagged, not silently dropped; the user's signal (the thumb, the category) survives even when the comment does not.

### The dedup layer — three passes, in order

Dedup is the difference between a feedback firehose and a signal source. Three passes run in sequence; order matters.

- **Pass 1 — per-session collapse.** Multiple thumbs-down events in the same session on the same turn (users double-tap, users refresh the page, users rate the same response twice) collapse to one row. Key: `(session_id, turn_id, direction)`. Keeps the last version (users often downgrade their thumb after the next turn).
- **Pass 2 — canonical form on free-text reports.** Normalise case, strip whitespace, replace user-mentioned proper nouns with placeholders, and apply a per-surface vocabulary dictionary ("playlist", "recommendation", "song" → "item"). Hash the canonicalised form; identical hashes collapse to one row with a `user_count` column. Chapter 03's canonical form is the same canonicalisation mod-102 chapter 04 applied to prompts for the `prompt_hash` column — reuse the function, do not reimplement.
- **Pass 3 — near-duplicate detection on prompt + response.** Two feedback events on different trace_ids but the same prompt / response (modulo whitespace and user-mentioned proper nouns) are rated against the same output. Cluster them with a cheap embedding similarity threshold and a per-day blocking window. The output is a `feedback_cluster_id` on the row; the per-cluster row is the unit downstream consumers aggregate.

Three disciplines make dedup defensible:

- **Clusters are scoped by time and surface.** A cluster that spans six months on a single `/v1/search-answer` endpoint means the surface regressed to the same problem six months ago. Keep the time dimension on the cluster or you lose the temporal shape.
- **Dedup is lossy, and the loss is logged.** Every dedup pass logs a count of rows-in / rows-out; the counts land on the mod-107 dashboard as a feedback-pipeline-health metric. A pass that collapses 10 % of rows usually, and 70 % on a Tuesday afternoon, means a coordinated event (a social-media call-out, a customer-support-driven influx) — the on-call wants to know.
- **The original rows are retained.** The dedup pass produces a `feedback_cluster_id`; the raw rows stay in the `user_feedback_events` table. Downstream aggregation reads the cluster; audit reads the raw.

### The filter — what is *not* eval signal

Not every feedback row belongs in the eval dataset. The filter runs after dedup and before triage.

| Shape | Belongs in... | Why |
|---|---|---|
| "The model's answer was wrong" | Eval dataset | Rubric alignment signal; the mod-104 calibration consumes it |
| "The model refused when it should not have" | Eval dataset (chapter 06 of mod-108 reads this specifically) | Over-refusal signal against the written safety policy |
| "The button didn't load" / "the page is slow" | Product bug tracker | Not a model-output problem; routes to engineering |
| "I want feature X" | Product feature-request tracker | Not an eval signal; routes to product |
| "The model said something I disagree with politically" | Trust-and-safety review (chapter 03 of mod-108 reads this as a potential abuse signal, but not as an eval row) | Adversary-persona-relevant; routes to `ai-risk-engineer` peer, not into the eval dataset |
| "This is harmful / dangerous" | Safety-eval queue (chapter 02 of this module) | Fast-tracked into the expert-review workbench with the OWASP tag |

The filter is a classifier (a small LLM-as-judge or a rule-based filter on the structured form's category field — chapter 03 of mod-104 formalises the classifier shape). Mis-classification is survivable because the triage step (next section) names an owner for each row and the owner can re-route. Not having a filter at all is not survivable — the eval dataset drowns in bug reports.

### Routing with owner, severity, and SLA

After the filter, each row that remains has a per-cluster owner and a severity. Two artefacts drive routing.

- **The per-surface routing table.** A mapping of `{surface_id, category}` → `{owner_role, sla_hours}`. For `/v1/support-bot/chat` + `incorrect_answer`, the owner is the eval engineer for the support rubric (72 h SLA); for the same surface + `refused_valid_request`, the owner is the policy owner (24 h SLA). The table is maintained per surface by the surface's PM; the eval team supplies the template.
- **The severity rule.** A function of the `user_count` on the cluster, the `severity` on the structured form, and the surface's criticality. A single thumbs-down on a low-traffic surface is `low`; 500 thumbs-down on the same cluster across a day on the primary surface is `high`. The rule lives in a config file with a change-log; the mod-107 dashboard reads it to colour the triage queue.

Three ownership splits recur across teams:

- **On-call engineer** owns rows tagged `incorrect_answer` with severity `high` — these might be regressions the mod-107 drift monitor missed. The on-call triages within the SLA; most are dismissed as expected behaviour; some become mod-107 alerts.
- **Eval engineer** owns rows tagged `rubric_mismatch` or `judge_disagreement_candidate` — these feed back into the mod-104 calibration or trigger a rubric revision.
- **PM** owns rows tagged `feature_gap` or `cross-surface regression` — these are not eval failures but product signals routed through the eval pipeline because the capture happens to be the same widget.

The routing table is defensible when every cluster has exactly one owner at a given time; a cluster with two owners is a cluster nobody closes.

### The loopback into the online-eval dashboards

Mod-107's safety-reviewer dashboard already shows the drift signal and the canary gate. The feedback-triage pipeline adds three new tiles:

- **Backlog by surface and severity.** Pending clusters, p50 / p90 / p99 time-in-queue, per surface. A backlog exceeding the per-severity SLA page colour-shifts the tile.
- **Disposition rate.** Fraction of clusters triaged (owner assigned + first response) within the SLA. Reported per surface per day. Mod-112 reads this as a program-health metric.
- **Thumbs-down rate.** The rate of thumbs-down events per 1 000 turns, per surface, as a trailing-7-day window. A rate that trends upward is a drift signal the automated monitor may not catch (users respond to subtle regressions the metrics do not).

The loopback is the other direction too: when the mod-107 drift monitor fires a per-cohort alert, the dashboard links to the feedback clusters landing on the same cohort in the same window. The two signals triangulate — automated drift + rising thumbs-down on the same cohort is a confirmed regression; automated drift without a thumbs-down shift is often a benign distribution change.

### The bootstrap — getting from zero to a usable pipeline

Four steps to stand this up without over-building:

- **Day 1.** Add a thumbs widget to the surface; wire it to the mod-102 client instrumentation; confirm events land in the trace store with `trace_id` set. No structured form, no free-text yet.
- **Day 7.** Add the pass-1 session collapse and the pass-2 canonical-form dedup. The pass-3 near-duplicate layer waits until you have a week of data to tune the threshold.
- **Day 14.** Add the structured form on thumbs-down with 5 category options and a free-text comment. The form fields map 1-to-1 to the rubric's category enumeration; adding a category is a rubric change, not a form change.
- **Day 30.** Add the filter (bug / feature / abuse out of the eval dataset) and the routing table (owner per cluster, SLA per severity). Publish the three mod-107 dashboard tiles. Run the first retro on disposition rate.

Two common bootstrap mistakes:

- **Launching with the free-text report before the thumb.** The free-text volume is lower but the processing cost per row is higher; a team that starts there spends its first month triaging free-text and runs out of budget before the thumbs widget ships. Start with the single-bit signal; structure comes next.
- **Routing to a shared inbox.** A `feedback@` mailing list that "the team reads on Fridays" has no SLA, no severity, and no owner. Within a quarter the inbox has 4 000 unread messages. Route to a per-surface owner from the first row; a single owner for a small surface is fine as long as the owner is named.

### What this module is not for in the feedback space

- **A/B testing.** Feedback is not a treatment-controlled experiment. The mod-107 chapter 06 A/B platform runs those; feedback is an observational signal.
- **Churn prediction.** The thumbs-down rate correlates with churn but is not the churn metric. Churn modelling is a product-analytics artefact.
- **Content moderation.** User-reported harmful content routes to the mod-108 safety queue and to the `ai-risk-engineer` peer. The feedback pipeline is the capture; the harm model is the peer's artefact.

## Summary

- Three capture shapes: **thumbs** (single-bit, high volume), **structured form** (bounded fields, medium volume), **free-text report** (unbounded, low volume, high information).
- Feedback events are **part of the trace**, not a parallel stream. They ride the mod-102 instrumentation plane, inherit the turn's `trace_id`, and survive sampling. PII is redacted at capture.
- **Three dedup passes**, in order: per-session collapse, canonical form on free-text, near-duplicate detection on prompt + response. Clusters are scoped by time and surface; the raw rows are retained; the pipeline-health metric is logged.
- The **filter** keeps bug reports, feature requests, and abuse out of the eval dataset. Each gets a routing arrow — the eval dataset is not a catch-all.
- **Routing** assigns each cluster an owner and an SLA from a per-surface routing table. Three ownership splits: on-call (severity `high` regressions), eval engineer (rubric or judge-calibration signals), PM (product / feature signals).
- The **loopback** adds three tiles to the mod-107 dashboard (backlog, disposition rate, thumbs-down rate) and triangulates with the drift monitor.
- Bootstrap in four steps across the first month: thumb → session dedup → structured form → filter + routing.

Chapter 04 walks the discipline that decides whether the rubric itself is stable enough to anchor the labels this chapter and chapter 02 both produce.
