# Per-Feature Token Accounting: The Cost Report That Drives Product Decisions

## Motivation

The vendor invoice arrives on the 5th of the month. It is one number, roll-up per vendor, split at best by API key. Finance opens a ticket asking "which features drove the 3× growth"; the eval team says "we know per-model token counts"; finance says "that is not what I asked for." The month closes; no product decision is made; the growth compounds into the next invoice.

The cost column of the trade-off report only becomes useful when it can answer the questions product actually asks:

- *Which feature is 40 % of our monthly spend for 3 % of the value? Kill it, downgrade it, or route it.*
- *Which cohort is 15 % of requests but 60 % of tokens? Is the prompt shape wrong for that cohort?*
- *We just enabled Anthropic prompt caching on the support-bot preamble. How much did the cache save this week, and what is the cache-hit rate per feature?*
- *We are running the RAG re-index eval offline. Should it move to the Batch API for the 50 % discount, or does the eval loop need same-day results?*

None of those questions is answerable from a vendor invoice. All of them are answerable from a per-feature cost report driven by per-request attribution keys emitted at the trace boundary.

This chapter builds the accounting the trade-off report's cost column depends on. Five pieces:

- **Per-feature attribution** as a span attribute populated at the app boundary.
- **Token classes** — input / output / cache-read / cache-write / batch / thinking (where applicable) — priced separately.
- **The pricing snapshot** — vendor prices at the time of the request, versioned like a rubric.
- **Cache and batch accounting** — how prompt caching and Batch API discounts show up in the report.
- **Product decisions** — the report shape that drives kill / downgrade / cache / batch actions.

Chapter 05 built the latency column. This chapter builds the cost column. Chapter 07 walks the delegation shape once the report is complete.

## Core concepts

### The per-feature attribution key

Every request the app makes has a **feature** it belongs to. The feature is a product-level identifier — `support_bot.answer`, `research.summarise`, `code.completion`, `onboarding.welcome_email`, `finance.doc_extract` — not a technical identifier (never the model name, never the endpoint URL). One feature can call multiple models; one model can serve multiple features. The report attributes cost by feature.

The attribution is a span attribute on the request-root span:

```
feature.name         = "support_bot.answer"       # required
feature.version      = "v14"                       # optional but recommended
feature.surface      = "web"                       # optional (web / mobile / api / batch)
cost_centre          = "customer_success"          # required — the finance-linked bucket
cost_centre.project  = "support_bot"               # optional sub-bucket
```

The cost-centre roster is chapter 05 of mod-110's `cost_centre` field carried forward. The feature is a finer-grain bucket; usually a cost centre owns several features, and the per-feature report rolls up to the cost centre for finance reconciliation.

The attribute is populated at the *earliest* possible boundary in the app — the HTTP handler, the queue consumer, the scheduled job entrypoint. Never in the model wrapper: by the time the code reaches the model wrapper, the calling context is lost. The instrumentation validator (exercise 05) walks a day of production traces and reports the population rate of `feature.name`; below 99 % is unfinished instrumentation.

For features that fan out into multiple model calls (a chain calls the router model, then the retriever, then the generator, then a judge), every sub-call inherits the request-root's `feature.name` via context propagation. The mod-102 chapter-04 instrumentation walked the propagation shape; this chapter's report reads the propagated value on every model-call span.

### Token classes and their prices

Not all tokens cost the same. The report's cost column splits them.

| Class | What it is | Typical price shape |
|---|---|---|
| **input** | Tokens sent to the model in the prompt / messages | Vendor's "input token" rate |
| **output** | Tokens the model generated | Higher than input — typically 3 – 5× the input rate |
| **cache-read** | Input tokens served from a prompt cache hit | Substantially discounted — Anthropic and OpenAI publish current rates in their pricing pages |
| **cache-write** | Input tokens written into a prompt cache on first use | Typically at a small premium to input (Anthropic) or same as input (some vendors); vendor-specific |
| **batch** | Tokens submitted via a Batch API endpoint | Discounted from the synchronous rate — commonly around 50 % of the corresponding sync rate; check the vendor's pricing page |
| **thinking** / **reasoning** | For reasoning-tier models — tokens the model spent internally before output | Priced separately (typically at output-token rates); some vendors bill only the visible output tokens, others bill the reasoning tokens too |

The report shows each class as its own column, with token count and dollar total. Aggregation to a single "cost per request" always names the class breakdown as a drill-down.

Two consequences worth naming:

- **A "cheaper per-token" model can be more expensive per request.** A mid-tier model priced at 20 % of the frontier's per-token rate but generating 5× the output tokens costs the same. The report shows tokens *and* dollars, per class.
- **Cache savings are a first-class metric.** The dollar total in the `cache-read` class × the (input − cache-read) price gap is the savings from prompt caching. When product asks "did the cache pay off," this arithmetic answers.

### The pricing snapshot

Prices change. A vendor drops the frontier model's input price by 20 %; a new snapshot ships at a different price than the alias implied; the org negotiates an enterprise discount. The trade-off report has to be reproducible against past pricing, which means the pricing is a **snapshot** stored with the run.

Shape (mirrors mod-110's `pricing_snapshots` table):

```
pricing_snapshot:
  id:              sha256:2f1...b8c
  vendor:          anthropic
  effective_at:    2026-08-19T00:00:00Z
  source:          https://www.anthropic.com/pricing (fetched 2026-08-19)
  models:
    - model:                 claude-3-5-sonnet-20250929
      input_per_million:      3.00
      output_per_million:     15.00
      cache_read_per_million: 0.30
      cache_write_per_million:3.75
      batch_input_per_million: 1.50
      batch_output_per_million: 7.50
      currency:              USD
    - model:                 claude-3-5-haiku-20250929
      input_per_million:      0.80
      output_per_million:     4.00
      ...
```

Every request's cost calculation attaches the `pricing_snapshot.id` used. Re-running the cost report against a past run's timestamp uses that run's snapshot; the run's dollar total does not change when today's prices do.

The snapshot is refreshed on a cadence (usually weekly, or whenever a vendor announces a change). The refresh is a diff, reviewed like a rubric — a vendor price cut is fine to accept automatically; a vendor price hike triggers a review because it affects the trade-off decisions the previous week's reports drove.

Two edge cases the snapshot format handles:

- **Regional pricing.** Some vendors price differently per region. The snapshot has a `region` field; the request tags its region; the cost calc uses the regional row.
- **Enterprise-negotiated rates.** An enterprise contract overrides the public list price. The snapshot carries an `override_source` field naming the contract; a config file (out-of-repo — the contract terms are confidential) provides the actual numbers. The report does not display the override rates to non-authorised readers.

### Prompt caching accounting

Prompt caching is the single largest cost lever most production LLM apps have. All three major vendors ship it (Anthropic prompt caching, OpenAI Prompts / prompt caching, Google Gemini context caching); the accounting shape is the same.

The mechanics:

- **First request with a cacheable prefix**: the prefix's tokens are billed as `cache-write` (usually a small premium over input); the prefix is stored in the vendor's cache for a TTL (typically 5 minutes; longer with beta / enterprise features).
- **Subsequent request with the same prefix within TTL**: the prefix's tokens are billed as `cache-read` (heavily discounted); the incremental tokens beyond the prefix are billed as normal input.
- **The savings** = `(cache_read_tokens × (input_price − cache_read_price)) − (cache_write_tokens × (cache_write_price − input_price))`. Usually a large positive number for cacheable workloads; can be negative if the cache-hit rate is too low.

The report shows, per feature:

- Cache-hit rate = `cache_read_tokens / (cache_read_tokens + eligible_input_tokens)`.
- Cache-write / cache-read ratio.
- Net savings vs. no-cache baseline (dollars per day).
- The p95 of cache-read tokens per request (some features have huge stable prefixes; some have small variable ones).

Two prompt-caching failure modes worth naming:

- **Cache-hit rate too low for the cache-write premium to pay off.** Some workloads are inherently uncacheable (every request is unique); the cache-write premium eats the cost budget without any read savings. The report flags features with a `cache_hit_rate < 0.2` and `cache_write_dollars > cache_read_savings`.
- **Cache-hit rate reported by the vendor differs from the app's rate.** Vendors report cache-hit metrics in their own console; those metrics are per-account, not per-feature. The report's authoritative source is the app's own spans — the vendor console is a cross-check, not the source of truth.

### Batch API accounting

Batch APIs give ~50 % discount off the synchronous rate in exchange for asynchronous execution (results within 24 h on most vendors). The trade is worth it for:

- Offline evals (mod-106 replay bundle; mod-104 rubric re-runs; mod-108 red-team suites).
- Bulk data-generation runs (synthetic-eval-set expansion; RAG index seeding).
- Anything the product does not need same-day (nightly digest emails; weekly report generation).

The trade is *not* worth it for:

- Interactive product features (user is waiting).
- The online-loop scored-row runner (mod-107; 24 h latency defeats the loop's purpose).
- Anything with a same-day SLA.

The report attributes tokens to `batch` vs. `sync`, per feature. Two things the report enables product to see:

- **Features on `sync` that could be `batch`.** A feature whose request cadence is scheduled (nightly, weekly) but that hits the sync endpoint is leaving 50 % of its cost on the table. The report flags candidates for migration.
- **Features that migrated to `batch` and are still slow.** The Batch API has its own SLA (typically 24 h; sometimes longer under load). The report tracks batch-job wall-clock; a feature migrated for the discount but whose users need the results in 2 h is a candidate for rollback.

### The cost report shape

An extension of the multi-objective report from chapter 02.

```
# Per-feature cost report — 2026-08-19 to 2026-08-25

pricing_snapshot: sha256:2f1...b8c (fetched 2026-08-19)
scope:            all features under cost_centre in {customer_success, research, code_platform}

## Top features by weekly spend

| feature                       | requests | avg tokens/req (in/out) | cache hit | batch % | weekly $ | $ / request |
|-------------------------------|----------|-------------------------|-----------|---------|----------|-------------|
| support_bot.answer            | 1.2M     | 2400 / 320              | 78%       | 0%      | $8,120   | $0.0068     |
| research.summarise            | 84K      | 12800 / 890             | 22%       | 0%      | $5,940   | $0.0707     |
| code.completion               | 3.1M     | 1900 / 40               | 91%       | 0%      | $3,410   | $0.0011     |
| eval.rag_faithfulness_nightly | 51K      | 3200 / 180              | 0%        | 100%    | $290     | $0.0057     |
| onboarding.welcome_email      | 4.1K     | 800 / 340               | 0%        | 100%    | $32      | $0.0078     |

Weekly total (in scope):  $17,792
Vs. last 4-week average:  +18%  (growth attributable primarily to `support_bot.answer` +26%)

## Token class breakdown — support_bot.answer

| class        | tokens (M) | $ this week | $ last week | Δ     |
|--------------|-----------|------------|------------|-------|
| input        | 780       | $2,340     | $2,180     | +7%   |
| cache-read   | 2,050     | $615       | $410       | +50%  |
| cache-write  | 60        | $225       | $220       | +2%   |
| output       | 384       | $5,760     | $4,590     | +25%  |
| batch        | 0         | $0         | $0         | 0     |
| thinking     | 0         | $0         | $0         | 0     |
| ---          |           |            |            |       |
| feature $    |           | $8,940     | $7,400     | +21%  |
| cache savings|           | $2,050     | $1,640     | (adds to feature-value)|

Note: output tokens grew 25% week-over-week. Investigate prompt-tuning cause;
check whether recent prompt update raised `max_tokens` or changed a system
prompt that removed a "be concise" directive.

## Product-decision recommendations

- support_bot.answer: prompt caching healthy (78% hit rate, net +$2K/week). Growth
  is on the output side — recommend a prompt review (are we generating longer
  responses than necessary?)
- research.summarise: cache hit only 22% and per-request cost is 10× the app avg.
  Feature has 84K weekly requests for ~2K active weekly users. Recommend: (a) audit
  whether the pre-summary retrieval preamble can be cached (currently varies per
  request); (b) evaluate routing eval per chapter 03 — is the frontier tier justified
  for all 84K requests?
- eval.rag_faithfulness_nightly: on Batch API; 24h SLA is fine (nightly run).
  Consider migrating the mod-108 red-team suite to Batch too — currently sync;
  ~$180/week could halve.

## Reconciliation vs. vendor invoice (last month)

Vendor invoice total:                     $71,400
Sum of feature totals (from report):      $70,850
Delta:                                    $550 (0.8%)
Attribution of delta:
  - untagged requests (feature.name unpopulated): $340
  - pricing snapshot lag (2 days after price change): $210
Action: raise instrumentation validator's failure threshold from 99% to 99.5%
        for feature.name; refresh pricing snapshots on the price-announce date, not weekly.
```

The report shows the top features, the token-class breakdown for movers, the product-decision recommendations the accounting drives, and the reconciliation against the vendor invoice. The reconciliation is what makes finance trust the report; the recommendations are what make product act on it.

### The invoice reconciliation contract

Every cost report has a reconciliation against the vendor invoice. Two rules:

- **The delta is under 2 % — ideally under 1 %.** A delta above 2 % is an instrumentation problem (untagged requests) or a pricing snapshot problem (stale prices). Above 5 % and finance stops trusting the report.
- **The delta is attributed.** The attribution names untagged requests, stale pricing snapshots, and any known accounting gap (e.g., the vendor bills a request the app never traced because the request failed at the vendor's WAF before hitting the SDK's instrumentation hook).

The reconciliation is monthly by default (aligned with the invoice cycle). A weekly reconciliation is possible if the vendor publishes daily usage — the shorter cycle catches drift earlier.

### Cost as a driver of product decisions

The report shape exists to answer product questions. Six recurring ones:

- **Kill.** A feature is 5 % of value and 30 % of cost. Recommendation: kill or downgrade to an OSS model. The report provides the evidence.
- **Downgrade.** A feature is on the frontier; the report shows the per-feature quality metrics side-by-side with cost. Product ships the routing report (chapter 03) that routes this feature to mid-tier for the cost saving.
- **Cache.** A feature has a low cache-hit rate and a stable prefix that is not currently cached. Enabling prompt caching is a code change; the report predicts the savings.
- **Batch.** An offline feature is on sync. Migrating to batch drops the cost 50 %. The report identifies candidates.
- **Prompt-shorten.** A feature's cost growth is on the output side; the prompt is generating longer answers than needed. Product ships a `max_tokens` cap or a "be concise" system message; the report tracks the delta.
- **Cohort-scoped downgrade.** The `free-tier` cohort is 60 % of requests and 20 % of revenue; downgrading `free-tier` alone to mid-tier saves substantial cost without touching paying users. Chapter 03's routing eval implements it.

Every one of these is a product decision that a well-shaped cost report enables. Without the report, the decisions happen in the dark (or not at all).

### Anti-patterns to avoid

**"Vendor invoice as the report."** Cost is one number per vendor per month; product cannot act on it. The per-feature attribution is the entire point.

**"Per-model report, no per-feature."** A per-model cost is a fleet-altitude view. Per-feature is the app-altitude view product needs. Both belong on the trade-off report; the per-feature is the primary.

**"Cache hit rate reported once."** The cache-hit rate matters per feature and per cohort. A global cache-hit rate that includes the un-cacheable feature drags the overall rate down and hides the cacheable feature's success.

**"Batch and sync conflated."** Batch tokens billed at ~50 % of sync tokens; a report that averages them across a feature drops the visibility into which parts are on the discount and which are not.

**"Pricing baked in."** The report code has vendor prices as constants. When prices change, the report is wrong until someone updates the constant. Snapshot the price with the run.

**"No reconciliation."** The report claims a weekly total; the invoice says something different; nobody can explain the delta; finance stops trusting the report. Reconciliation is not optional.

**"Recommendations without evidence."** The report says "kill this feature"; product asks "why"; the report has no answer. Every recommendation is backed by the specific numbers on the report (cost share, value share, cache rate, cohort concentration).

**"Cost isolated from quality."** A cost report that omits the quality context (chapter 02's cohort-quality matrix) leads to "let's downgrade the support-bot" decisions that regress the enterprise cohort. The trade-off report is multi-axis for a reason.

## Summary

- Every request carries a **`feature.name`** span attribute at the request-root; sub-calls inherit via context propagation. The report attributes cost by feature; features roll up to **cost centres** (from mod-110 chapter 05) for finance reconciliation.
- **Token classes** are priced separately: **input**, **output**, **cache-read**, **cache-write**, **batch**, **thinking / reasoning** (where applicable). A single "cost per request" always drills into the class breakdown.
- The **pricing snapshot** is a versioned artefact — vendor prices at the time of the request, stored with the run so the cost calculation is reproducible. Refreshed on the price-announce date, reviewed like a rubric.
- **Prompt caching** (Anthropic prompt caching, OpenAI Prompts / prompt caching, Gemini context caching) is the single largest cost lever most apps have. The report tracks cache-hit rate, cache-write / cache-read ratio, net savings, and flags features where the cache-write premium exceeds the read savings.
- **Batch API** halves the cost for offline workloads with a 24 h SLA — evals, nightly digests, bulk generation. The report flags candidates for `sync → batch` migration.
- The **cost report shape** shows top features by spend, token-class breakdown for movers, product-decision recommendations backed by evidence, and reconciliation vs. vendor invoice (delta target < 2 %; attributed).
- **Cost as a product-decision driver**: kill, downgrade, cache, batch, prompt-shorten, cohort-scoped downgrade — six recurring shapes the report enables.
- **Anti-patterns**: vendor invoice as the report; per-model without per-feature; global cache-hit rate; batch and sync conflated; prices baked in; no reconciliation; recommendations without evidence; cost isolated from quality.

Chapter 07 closes the module — the delegation boundary to `model-evaluation-engineer` (model-altitude quality and MLPerf-shaped serving) and `ai-infra-performance` (fleet-altitude kernels and capacity), and the escalation shape for gaps the app-altitude report surfaces but cannot fix.
