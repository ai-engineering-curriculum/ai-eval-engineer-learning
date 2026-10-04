# exercise-05: Per-Feature Token Accounting

**Estimated effort:** 2 hours

## Objective

Wire **per-feature cost attribution** into the trace pipeline from mod-102, split tokens into the six classes chapter 06 names (**input / output / cache-read / cache-write / batch / thinking**), snapshot the vendor pricing used for each run, and publish the per-feature cost report with product-decision recommendations and the vendor-invoice reconciliation footer. Then drive **at least one real product decision** from the report — kill, downgrade, cache, batch, prompt-shorten, or cohort-scoped downgrade.

By the end of the exercise the trade-off report's cost column is a *per-feature* view that finance accepts, product acts on, and the eval team can defend against the vendor invoice to under 2 % delta. The last step — a signed product decision driven by the report — is what makes this exercise a trade-off artefact and not a billing utility.

## Prerequisites

- Chapters 01, 02, and 06 of this module.
- Mod-102 chapter 02 trace instrumentation live on at least one product surface. The `gen_ai.usage.input_tokens` / `gen_ai.usage.output_tokens` attributes must be populated on every model-call span; the request-root span must carry `feature.name` and `cost_centre`. If either side is missing, extend mod-102 first — this exercise cannot produce defensible numbers from under-tagged spans.
- **Exercise 04** (recommended but not strictly required) for the trace-pipeline validator; the same validator from exercise 04 can be extended here to check `feature.name` and `cost_centre` populate rates.
- A pricing snapshot source. The chapter 06 shape mirrors mod-110's `pricing_snapshots` table; if mod-110 is in place, read from it. If not, a YAML per vendor is enough — the per-request snapshot id must still be deterministic.
- A trace backend queryable from the exercise code (Phoenix / Langfuse / Weave / Braintrust / OTel + warehouse). At least 7 days of production spans across ≥ 3 features and ≥ 2 cost centres.
- A vendor invoice from the past month (any vendor is fine; the exercise targets whichever is primary for the surface). The reconciliation footer compares against this.
- Python 3.11+; `pyyaml`; a plotting / tabulation library for the top-features table.

## Set-up

1. Create `eval/tradeoff/cost/` in your repo:

   ```
   eval/tradeoff/cost/
   ├── config.yaml
   ├── pricing/
   │   ├── snapshots/
   │   │   ├── 2026-08-19.yaml           # one per snapshot date; sha256 is the id
   │   │   └── 2026-09-02.yaml
   │   ├── fetch.py                      # pull current price pages and diff vs. last snapshot
   │   └── review.py                     # the "accept the diff" workflow (price cut auto; price hike reviewed)
   ├── schema/
   │   ├── feature_catalog.yaml          # the valid set of feature.name values
   │   └── cost_centre.yaml              # the finance-linked bucket roster (from mod-110)
   ├── validate/
   │   └── populate_rate.py              # extends exercise-04's; adds feature.name + cost_centre
   ├── compute/
   │   ├── classify_tokens.py            # map vendor token counts to the 6 classes
   │   ├── price.py                      # tokens × snapshot → dollars
   │   ├── per_feature.py                # aggregate per feature per cohort
   │   ├── cache_savings.py              # chapter 06 cache arithmetic
   │   └── batch_vs_sync.py              # per-feature batch % + candidate list for migration
   ├── reconcile/
   │   ├── invoice_loader.py             # parse the vendor invoice (per-vendor adapter)
   │   └── attribute_delta.py            # untagged, snapshot lag, and known gaps
   ├── report/
   │   ├── build_cost_report.py
   │   ├── top_features_table.py
   │   └── recommendations.py            # chapter 06 kill/downgrade/cache/batch heuristics
   ├── decisions/
   │   └── 2026-XX-XX-<feature>.md       # one signed product decision this report drove
   ├── output/
   │   └── <report_id>/
   │       ├── cost_report.md
   │       ├── per_feature.csv
   │       ├── token_class_breakdown.csv
   │       ├── invoice_reconciliation.csv
   │       └── report.json
   ├── tests/
   │   ├── test_classify_tokens.py
   │   ├── test_price.py
   │   ├── test_cache_savings.py
   │   ├── test_batch_vs_sync.py
   │   ├── test_recommendations.py
   │   ├── test_invoice_reconcile.py
   │   └── test_report_render.py
   └── README.md
   ```

2. Fill `config.yaml`:

   ```yaml
   trace_backend:
     kind: phoenix                       # or langfuse | weave | braintrust | otel_clickhouse
     dsn:  http://localhost:6006
   window:
     start: 2026-10-01T00:00:00Z
     end:   2026-10-07T23:59:59Z
   feature_catalog: ./schema/feature_catalog.yaml
   cost_centre:     ./schema/cost_centre.yaml
   pricing:
     snapshots_dir: ./pricing/snapshots
     current:       2026-09-02.yaml
   invoice:
     vendor:  anthropic                  # extendable per adapter
     path:    ./invoices/2026-09-anthropic.csv
     target_delta_pct: 2.0
   recommendation_thresholds:
     kill:
       cost_share_min: 0.10
       value_share_max: 0.03
     downgrade:
       cohort_cost_share_min: 0.40
       cohort_value_share_max: 0.20
     cache:
       prefix_stability_min: 0.70
       current_hit_rate_max: 0.20
     batch:
       sla_minimum_hours: 24
   ```

3. Author `schema/feature_catalog.yaml` listing every valid `feature.name`. The validator rejects spans whose `feature.name` is outside the catalog — the catalog is the "no new features without an entry" discipline.

4. Author or import `schema/cost_centre.yaml` from mod-110. Each cost centre has an owner contact; the recommendations route to the right human.

5. Author or seed a pricing snapshot (chapter 06 shape; the exercise's reference snapshot can be hand-authored on day one from the current price pages):

   ```yaml
   # pricing/snapshots/2026-09-02.yaml
   id:              auto                 # = sha256 of normalised body
   effective_at:    2026-09-02T00:00:00Z
   vendor:          anthropic
   source:          https://www.anthropic.com/pricing
   source_fetched:  2026-09-02T14:12:00Z
   models:
     - model:                    claude-opus-4-7
       input_per_million:         15.00
       output_per_million:        75.00
       cache_read_per_million:     1.50
       cache_write_per_million:   18.75
       batch_input_per_million:    7.50
       batch_output_per_million:  37.50
       reasoning_per_million:     75.00
       currency:                  USD
     - model:                    claude-haiku-4-5
       ...
   ```

   Values above are illustrative placeholders — fetch the current numbers from the vendor's live pricing page before relying on them.

6. Place the past-month vendor invoice in `invoices/<yyyy-mm>-<vendor>.csv`. The exercise does not require real customer data — a representative sample is enough — but the format must match what the invoice-loader expects (usually `model, tokens_in, tokens_out, cost_usd`, sometimes with extra columns per vendor).

## Requirements

Produce a PR against your working branch that adds:

1. **`schema/feature_catalog.yaml` and `schema/cost_centre.yaml`** — the valid rosters; every `feature.name` and `cost_centre` seen in production must appear.
2. **`pricing/snapshots/*.yaml`** — at least two snapshots; each with a stable `id` (hash of normalised body); each referencing its source URL and fetch timestamp.
3. **`pricing/fetch.py`** — pulls the vendor pricing pages (one adapter per vendor) and writes a candidate snapshot. If values are unchanged, no-op. If values changed, writes `snapshots/<date>.yaml` and emits a diff.
4. **`pricing/review.py`** — the chapter 06 review workflow:
   - A price *cut* is accepted automatically with a log line.
   - A price *hike* flags the diff for human review; the diff lists which features' prior-week reports would move materially (threshold configurable; default 5 %).
5. **`validate/populate_rate.py`** — extends the exercise-04 validator. Adds `feature.name` and `cost_centre` to the required set. Refuses to proceed if either is below the configured threshold (default 99 %). Reports the top 5 untagged-request sources for operator triage.
6. **`compute/classify_tokens.py`** — maps the raw vendor usage attributes (`gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.usage.cache_read_tokens` / vendor-specific analogues, `gen_ai.usage.cache_creation_tokens`, `gen_ai.request.batch`, `gen_ai.response.thinking_tokens`) into the six-class canonical form. Documents the vendor-specific attribute names in a header comment per adapter.
7. **`compute/price.py`** — tokens × snapshot → dollars per class, per request, per feature. Reads the snapshot id from the request span attribute (chapter 06 requires every scored request to carry `pricing_snapshot_id`); if absent, the pricer either back-fills from the request timestamp against the snapshot history, or raises if the snapshot history has a gap.
8. **`compute/per_feature.py`** — aggregates:
   - Per feature: request count, mean tokens in / out, cache-hit rate, batch share, weekly dollars, dollars per request.
   - Per cohort × feature: the same aggregates sliced by the cohort roster.
   - Writes `per_feature.csv` and `token_class_breakdown.csv`.
9. **`compute/cache_savings.py`** — chapter 06 arithmetic: `savings = (cache_read_tokens × (input_price − cache_read_price)) − (cache_write_tokens × (cache_write_price − input_price))`. Reports per feature as absolute dollars and as net benefit vs. the no-cache baseline. Flags features where `cache_hit_rate < 0.2` and `cache_write_dollars > cache_read_savings`.
10. **`compute/batch_vs_sync.py`** — per feature:
    - Current batch % of tokens.
    - Request cadence inferred from the request-timestamp distribution (interactive / scheduled / nightly).
    - Candidate flag for `sync → batch` migration if `inferred_cadence ∈ {nightly, weekly}` and current `batch % < 90`.
11. **`reconcile/invoice_loader.py`** — per-vendor adapter that parses the invoice CSV / PDF into `(model, tokens_in, tokens_out, cost_usd)` rows. Rejects invoice rows that do not join cleanly against the pricing snapshot (unknown model, out-of-period, duplicate).
12. **`reconcile/attribute_delta.py`** — computes `invoice_total − sum(feature_totals)`; attributes to:
    - **Untagged requests** — the share of invoice spend that cannot be matched to any `feature.name` in the window; the attribution reads from the trace-pipeline untagged bucket.
    - **Pricing snapshot lag** — spend that was priced under a snapshot older than the effective change date.
    - **Known gaps** — any documented accounting hole (e.g., requests failing before the SDK hook; vendor-side billing for pre-request validation).
    - **Residual** — the unexplained remainder. The chapter 06 target is a total delta under 2 %; above 2 % is a report finding, not acceptance.
13. **`report/build_cost_report.py`** — the chapter 06 shape:
    - Header block.
    - **Top features by weekly spend** — the chapter 06 table (requests, avg tokens, cache hit, batch %, weekly $, $ / request).
    - **Token-class breakdown for the top mover** — six classes × tokens / dollars / Δ vs. last week.
    - **Product-decision recommendations** — rendered by `report/recommendations.py` using the thresholds from config.
    - **Reconciliation vs. vendor invoice** — the chapter 06 reconciliation footer with the delta attribution.
    - Report hash in the footer.
14. **`report/recommendations.py`** — the chapter 06 six heuristics (**kill / downgrade / cache / batch / prompt-shorten / cohort-scoped downgrade**) applied to the per-feature aggregates. Each recommendation names the feature, the evidence (specific numbers), and the proposed next step; without the evidence the recommendation is omitted. "Recommendations without evidence" is a chapter 06 anti-pattern.
15. **One signed product decision** — `decisions/<date>-<feature>.md`. The report drove the decision; the decision names the feature, the recommendation chosen, the signer (product + eval owner), and the expected delta. If the decision landed on "no action," that is still a valid decision; the markdown records the reasoning. The exercise's acceptance criteria require this file to exist — a cost report that never drives a decision is a dashboard, not a report.
16. **`tests/*`** — unit tests for:
    - `test_classify_tokens.py` — vendor-specific attribute names map to the six classes correctly; fixtures per vendor.
    - `test_price.py` — tokens × snapshot → dollars matches hand-computed; a missing snapshot id raises; the snapshot-history back-fill works.
    - `test_cache_savings.py` — a known cache-hit-rate / write-ratio returns the expected net savings; a sub-threshold hit rate is flagged.
    - `test_batch_vs_sync.py` — a feature with 5 requests / day uniformly in a 2-hour window is flagged as nightly-schedulable; a feature with continuous 60-RPS traffic is not.
    - `test_recommendations.py` — fixtures where each of the six recommendations is the correct output; the renderer omits any recommendation without evidence.
    - `test_invoice_reconcile.py` — a toy invoice × toy feature set; the attribution delta sums to the actual delta; untagged bucket is correctly sized.
    - `test_report_render.py` — stable Markdown render; the Reconciliation footer sums match within 1 % of the invoice.
17. **A demonstration run** on ≥ 7 days of real traces and the past-month invoice. The report must show:
    - At least three features in the top-features table.
    - A reconciliation delta under 2 % (or the delta attributed to a specific gap with a fix plan).
    - At least one recommendation that landed in the signed product decision.
    - A token-class breakdown that shows the top mover's growth was in a specific class (output, cache-read, etc.).
18. **`README.md`** — how to install the pricing snapshots, how to validate instrumentation, how to run the report, how to read the recommendations, how to sign a product decision; one page walking through the demonstration run.

## Starter guidance

- **The attribution key is populated at the earliest boundary, not in the model wrapper.** Chapter 06 is explicit: the HTTP handler, the queue consumer, the scheduled-job entrypoint. By the time the code reaches the model wrapper, the calling context is lost. The validator catches drift from this; the fix is upstream.
- **The feature catalog is a `MUST` list, not a `SHOULD` list.** A request that lands with `feature.name = "unknown"` is a reproducibility break — the next cost report cannot attribute it. The catalog is the schema; new features add a row before shipping.
- **Snapshot prices before you query the vendor.** Chapter 06's anti-pattern "pricing baked in" is how last month's cost report becomes wrong this month without anybody noticing. The snapshot id is per-request; the current-snapshot pointer is used only when a request is scored *now* against *current* prices.
- **A price hike is a report-shape change.** When Anthropic / OpenAI / Google change their input or cache-read price, every prior-week report's dollar numbers in the dashboard-y backward-looking view become arguable. The review workflow (chapter 06) requires a human eye; the exercise's `pricing/review.py` enforces it.
- **The reconciliation is monthly by default.** Weekly only if the vendor publishes daily usage. The exercise's demonstration run is weekly inside the window plus a monthly reconciliation footer against the past month.
- **A sub-2 % reconciliation is the target, 1 % is the goal.** Finance stops trusting the report above 2 %; above 5 % the eval team loses the conversation entirely. The delta is attributed, not just computed; "untagged" is a specific line, not a shrug.
- **Cache-hit rate is per feature and per cohort.** A global cache-hit rate averages uncacheable features (every request unique) with highly cacheable ones; the number is useless. The report always slices.
- **Batch is for offline, not interactive.** The exercise's threshold heuristic — `inferred_cadence ∈ {nightly, weekly}` — is deliberately conservative. Migrating a user-interactive feature to batch is an outage; migrating an offline eval is 50 % cost. The heuristic errs on the safe side.
- **Recommendations without evidence are a chapter 06 anti-pattern.** Every recommendation names the specific numbers (cost share, value share, cache rate, cadence) that triggered it. The renderer omits a recommendation that cannot produce evidence; the exercise's `test_recommendations.py` enforces it.
- **The signed decision is the point.** A cost report that is never signed is a dashboard. Even "no action, revisit in 30 days" is a decision; the markdown records who signed and what the revisit trigger is. The eval team's institutional memory grows from this file collection.

## Acceptance criteria

You are done when:

- The validator refuses to run the report when `feature.name` or `cost_centre` is below the threshold on any cohort.
- At least two pricing snapshots exist; each has a stable id (hash of normalised body); the `pricing/review.py` workflow has an audit trail of accept / review decisions.
- `per_feature.csv` is produced for the window; `token_class_breakdown.csv` splits each feature into the six classes with tokens and dollars.
- Cache savings are reported per feature; sub-threshold-hit-rate features are flagged.
- Batch migration candidates are flagged with the inferred cadence that justified the flag.
- The reconciliation footer sums the per-feature totals against the invoice; the delta is under 2 % or explicitly attributed with a fix plan.
- Recommendations are rendered with specific-number evidence; recommendations without evidence are omitted (not stubbed).
- At least one signed product decision markdown exists in `decisions/` referencing a specific recommendation this report produced.
- Tests pass in CI; each `tests/*.py` includes a fixture that would catch a regression in the arithmetic it is testing.
- The `report.json` is reproducible — running the generator twice on the same inputs returns identical bytes.
- The `README.md` walkthrough is followable by a reader who has not read chapter 06.

## Stretch goals

- **Per-cohort cost report.** Extend the top-features table to be top-features × cohort — surfaces cohorts whose cost per request is higher than the feature average (long inputs, tool-heavy, enterprise). The cohort-scoped downgrade recommendation gets concrete.
- **Automatic prompt-caching evaluator.** For features whose cache-hit rate is low and whose prompts have a stable prefix (static system message + static tool specs + variable user turn), auto-propose the cache-point placement that would raise the hit rate. Simulate the net savings over the window; attach the proposal to the recommendation.
- **Pricing-snapshot CI gate.** When `pricing/review.py` detects a price hike, open an auto-generated PR with the diff and a snapshot of which features would move materially. Reviewers acknowledge the diff before the snapshot promotes to current.
- **Invoice-anomaly detector.** On monthly reconciliation, flag line items whose model / cost mix differs materially from the per-feature report's expectation. A vendor billing mistake (or an app bug double-charging) shows up here before finance's monthly review.
- **FinOps roll-up.** Add a cost-centre view that rolls per-feature totals into cost-centre totals, with charge-back / showback semantics. Hook into the org's FinOps dashboard; the report becomes an input, not a parallel surface.
- **Growth attribution.** When the week-over-week delta on a feature is above a threshold, attribute the growth to one of (request count ↑, tokens-per-request ↑, class mix shift, snapshot price change). The report's commentary section names the attribution automatically.
- **Thinking-token accounting for reasoning tiers.** For features that use reasoning models (Opus thinking, o-series reasoning tokens), split the reasoning-token column by task shape (math-heavy / long-reasoning / short-reasoning) and track growth. Catches the "reasoning turned on everywhere and nobody noticed" cost blow-out.

## What this exercise does *not* cover

You are not building the multi-objective report shape (exercise 01), the routing eval (exercise 02), the replacement-regression harness (exercise 03), or the app-altitude latency report (exercise 04). You are shipping the *per-feature cost report with product-decision recommendations and invoice reconciliation* — the chapter 06 artefact that makes the trade-off report's cost column a shippable, finance-defendable, product-actionable surface.
