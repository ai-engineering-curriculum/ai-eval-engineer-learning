# exercise-01: Trace and Eval-Result Schema with Lineage

**Estimated effort:** 3 hours

## Objective

Design and materialise the **chapter 02 schema** in a real store, wire the **lineage-key hash computation**, back-fill from your mod-102 trace store and mod-107 scored-row store, and ship the **`reproducibility_check`** that turns the schema from "tables that exist" into "tables that guarantee reproducibility."

By the end of this exercise you will have a queryable warehouse of runs, results, judgements, rubrics, and eval-sets — every row carrying the nine lineage keys — plus a working reproducibility check the platform's own dashboard (exercise 04) will read.

This is the substrate exercises 02 – 05 build on. Exercise 02 will populate `eval_sets` and `eval_cases`; exercise 03 will drive `eval_runs` and `eval_results` through the runner-adapter plane; exercises 04 – 05 will instrument and gate against the tables you build here. Get the schema right; everything else compounds off it.

## Prerequisites

- Chapters 01 and 02 of this module.
- A trace backend from mod-102 (Phoenix / Langfuse / Weave / Braintrust, or plain OTel spans). If the mod-102 exercise is not done, a minimal service that emits OTel-GenAI-conformant spans is enough.
- The mod-107 scored-row store from exercise 01 of that module — this exercise back-fills from it. If mod-107 is not done, generate a small synthetic scored-row set (~500 rows spanning ≥ 2 cohorts and ≥ 2 rubrics) to back-fill from.
- The mod-104 rubric YAML from exercise 03 of that module — the `rubrics` table seeds from it.
- Python 3.11+; `hashlib`, a Postgres client (`psycopg` or `sqlalchemy`), an optional columnar target (DuckDB is the shortest path; ClickHouse, BigQuery, Snowflake if the org already runs one).
- A Postgres instance (local Docker is fine) and disk space for a small columnar warm tier.
- Read access to the mod-102 trace backend.
- No new API keys — this exercise is store-side; no judge inference.

## Set-up

1. Create `eval/platform/schema/` in your repo:

   ```
   eval/platform/schema/
   ├── config.yaml
   ├── migrations/
   │   ├── 0001_initial_tables.sql
   │   ├── 0002_lineage_denormalisation.sql
   │   └── 0003_pricing_snapshots_and_indexes.sql
   ├── canonicalisation/
   │   ├── prompt.py
   │   ├── chain.py
   │   ├── retriever_index.py
   │   ├── decoding_config.py
   │   ├── rubric.py
   │   └── eval_set.py
   ├── hashing.py                  # small facade around canonicalisation + sha256
   ├── models.py                   # dataclasses / pydantic for every table row
   ├── backfill/
   │   ├── from_mod102_traces.py   # traces / spans back-fill (or vendor pointer)
   │   ├── from_mod107_scored.py   # scored-row → eval_results back-fill
   │   └── from_mod104_rubrics.py  # rubric YAML → rubrics table seed
   ├── warm_tier/
   │   ├── export.py               # hot → warm periodic export
   │   └── ddl_duckdb.sql          # (or ddl_clickhouse.sql, ddl_bigquery.sql)
   ├── openlineage/
   │   └── emit.py                 # emits RunEvent per eval_run (optional wiring)
   ├── reproducibility/
   │   ├── check.py                # the reproducibility SLI check
   │   └── fixtures/               # a handful of frozen runs to check against
   ├── tests/
   │   ├── test_canonicalisation.py
   │   ├── test_hashing.py
   │   ├── test_schema.py
   │   ├── test_backfill.py
   │   └── test_reproducibility_check.py
   └── README.md
   ```

2. In `config.yaml`, name the stores, the retention windows, and the pointer to the warm tier:

   ```yaml
   hot_store:
     kind: postgres
     dsn: postgresql://eval:eval@localhost:5432/eval_platform
     retention_days: 30
   warm_tier:
     kind: duckdb                    # or clickhouse | bigquery | snowflake
     path: ./warm/eval_platform.duckdb
     export_cron: "0 * * * *"        # hourly
   traces_source:
     kind: phoenix                    # | langfuse | weave | braintrust | otel_raw
     endpoint: http://localhost:6006
   backfill:
     mod102_start: 2026-08-01
     mod107_scored_source: mod107_store.online_scored_rows
     mod104_rubric_dir: ../../mod-104-llm-as-judge-in-product/rubrics/
   openlineage:
     enabled: true
     endpoint: http://localhost:5000/api/v1/lineage
   ```

3. Apply the migrations against Postgres. Confirm the seven load-bearing tables plus `pricing_snapshots` and `eval_set_current` exist per chapter 02 with the exact column names and types.

4. Seed the `rubrics` table from the mod-104 YAML using `backfill/from_mod104_rubrics.py`. Compute the `rubric_hash` from the canonicalised spec; do not trust any `version:` field in the YAML.

5. Provision the warm tier — the exercise's default is DuckDB (a single file, zero-config); ClickHouse / BigQuery / Snowflake are all fine if the org already runs one.

## Requirements

Produce a PR against your working branch that adds:

1. **Migrations** — three SQL files that create the seven load-bearing tables (`rubrics`, `eval_sets`, `eval_cases`, `eval_runs`, `eval_results`, `judgements`, `traces`/`spans` if not in the vendor's store) plus `pricing_snapshots`, `eval_set_current`, and the indexes chapter 02 implies. Every migration is idempotent and reversible; a `down` script exists for each.
2. **`canonicalisation/*.py`** — one canonicaliser per hashable artefact (prompt, chain, retriever_index, decoding_config, rubric, eval_set). Each is a pure function: input → canonical bytes. Comment / whitespace / ordering normalisation is documented and tested. The chapter 02 canonicalisation rules apply.
3. **`hashing.py`** — thin facade: `sha256_of(canonical_bytes) -> "sha256:<hex>"`; helpers `prompt_hash(template)`, `chain_hash(graph)`, `retriever_index_hash(index_manifest)`, etc.
4. **`models.py`** — typed models (dataclasses or pydantic) for every row. Every model enforces required fields at construction — a missing `judge_model_snapshot` raises, not defaults to `null`.
5. **`backfill/from_mod102_traces.py`** — either populates the `traces`/`spans` tables from the mod-102 backend, *or* (preferred, for vendor-native stores) records a pointer row per trace that lets the plane query the vendor via `trace_id`. Handles at least the first two weeks of mod-102's trace stream.
6. **`backfill/from_mod107_scored.py`** — reads the mod-107 scored-row store; for each row, computes the lineage keys from the scored-row's recorded metadata (or the mod-107 online-loop's config), inserts one `eval_run` row per online-loop window and one `eval_results` row per scored trace, with `source='online_loop'`. If a scored row is missing a lineage key, the back-fill records the row with `null` for that key and increments a counter — never fabricate keys.
7. **`backfill/from_mod104_rubrics.py`** — reads the mod-104 rubric YAML directory; canonicalises each rubric; inserts / upserts into `rubrics`. A rubric whose canonical bytes have not changed does not re-insert.
8. **`warm_tier/export.py`** — an hourly job that copies rows aged past a threshold (default: 7 days) from hot to warm. Warm tier is append-only; deletion / update in the warm tier is a schema violation. Records the export run in a small `warm_exports` audit table.
9. **`openlineage/emit.py`** (optional but recommended) — emits an OpenLineage `RunEvent` per `eval_run` insert with `inputDatasets = [eval_set]`, `outputDatasets = [eval_results table with filter run_id=...]`, `job.name = "eval_run:<runner>:<surface>"`, and `parentRun` when the trigger is known.
10. **`reproducibility/check.py`** — the SLI. Given a historic `run_id`, the check:
    - Reads the run's full lineage-key set.
    - Reads the run's `eval_results` rows.
    - For a chosen small subset (typically 20 – 50 cases sampled from the run), re-dispatches an equivalent run against the current stored rubric + judge + target chain.
    - For deterministic runs (`seed` populated, no vendor non-determinism), asserts the re-run scores are byte-identical to the historic scores.
    - For stochastic runs, asserts the re-run scores are statistically indistinguishable (KS test on numeric scores, chi-squared on categorical verdicts, with a permissive `p > 0.05` threshold).
    - Reports pass / fail per run, with a diagnostic block naming which key changed if the check fails.
11. **`reproducibility/fixtures/`** — at least three frozen historic runs (with their `run_id`, spec, and expected results snapshot) the check exercises. One deterministic, one stochastic, one that intentionally has a `judge_model_snapshot` diff to prove the diagnostic works.
12. **`tests/*`** — unit tests for every canonicaliser (round-trip: two "equivalent" inputs hash the same; two "different" inputs do not), the hashing facade, the schema (migrations up and down cleanly on an empty DB), the back-fill (a small fixture set round-trips), and the reproducibility check (at least the three fixtures pass / fail as documented).
13. **`README.md`** in `eval/platform/schema/` — how to run the migrations, how to run the back-fill, how to run the reproducibility check, how to query the store, and where the warm tier lives.
14. **A demonstration back-fill** — run `from_mod107_scored.py` against a real (or synthetic) scored-row set of ≥ 500 rows spanning ≥ 2 cohorts, ≥ 2 rubrics, and ≥ 3 model snapshots. The resulting `eval_results` count reconciles against the source; the reproducibility check passes on at least one of the runs the back-fill created.

## Starter guidance

- **Do the canonicalisers before the tables.** The tables are boring; the canonicalisers are where the reproducibility contract actually lives. A test suite that pins each canonicaliser's equivalence class is what buys you a decade of schema stability.
- **Denormalise the lineage keys on `eval_results` from day one.** The join to `eval_runs` looks tempting; it is a query-planner risk at scale (chapter 02). Copy the keys onto the result row at insert time. Adding a column later is expensive; taking one away is trivial.
- **Every hash starts with `sha256:`.** The prefix costs nothing and makes the store forward-compatible with algorithm bumps. A bare hex column that later needs `blake3:` prefixes is a migration nobody wants to run.
- **`seed = null` is legal and load-bearing.** Do not default a null seed to `0` or `-1`. `null` means "non-deterministic on purpose"; `0` means "seed 0." The check reads the difference.
- **The vendor's own store is a legal home for `traces` / `spans`.** If Phoenix / Langfuse / Weave / Braintrust already hold the spans, do not duplicate them into Postgres. Record a pointer per trace and query the vendor at read time. The warm-tier export ignores the pointer table.
- **The reproducibility check is expensive.** Do not run it against every run. A daily sampled schedule against ≤ 1 % of runs is enough for the SLI; exercise 04 wires the sample rate.
- **Do the OpenLineage emit even if the org has no catalogue yet.** The events go to a stub endpoint; they cost nothing to emit; they are the interchange the peer will use when they absorb the slice (chapter 07). Emitting early keeps the future-hand-off cheap.
- **Back-fill is imperfect.** The mod-107 scored-row store may not have every lineage key you now require. Do not fabricate; record `null` and count. The `reproducibility_check` naturally reports these runs as "cannot verify" — that is the correct answer.

## Acceptance criteria

You are done when:

- All seven load-bearing tables plus `pricing_snapshots` and `eval_set_current` exist per chapter 02 with the exact column names, types, and indexes.
- Every canonicaliser passes a round-trip test suite — equivalent inputs hash the same, different inputs do not.
- `backfill/from_mod107_scored.py` populates `eval_runs` and `eval_results` from a scored-row source of ≥ 500 rows. Every column required by the schema is either populated or explicitly `null` (with a documented reason); no field is fabricated.
- `backfill/from_mod104_rubrics.py` seeds the `rubrics` table; each rubric's `rubric_hash` is the SHA-256 of the canonicalised spec.
- `warm_tier/export.py` runs on a schedule and copies aged rows to the warm tier; a small audit table records each export.
- `reproducibility/check.py` passes on the deterministic fixture, passes on the stochastic fixture, and produces the correct diagnostic on the intentionally-drifted fixture.
- `tests/*` pass in CI.
- `README.md` documents the migration, back-fill, and check invocations.
- The demonstration back-fill's row counts reconcile against the source (within a documented gap, if any).
- The `traces` / `spans` integration works — either the tables exist and are populated, or a pointer table with a queryable link to the vendor's store exists.

## Stretch goals

- **Migration rehearsal.** Run every migration `down` then `up` on a populated DB; assert the row counts and schema shape match before and after. This is the migration-safety pattern chapter 02 implies.
- **A schema-drift alert.** Emit a metric when the migration count differs from the deployed version's expected count; wire the alert to a `schema_drift` runbook (a stub is fine — exercise 04 fleshes it out).
- **Backfill of the full mod-102 trace history.** If the mod-102 backend has months of traces, back-fill a full quarter. Measure the wall-clock, tune the batch size, and document the throughput. The peer track will use this number.
- **Warm-tier query surface.** Ship a small `plane.query_warm(...)` helper (or DuckDB view) that exposes the seven tables to analytical queries. Confirm the mod-107 chapter 07 dashboard queries land here.
- **Multi-hash algorithm support.** Extend `hashing.py` to support `blake3:` alongside `sha256:`; add a plane-level policy that new hashes are `sha256:` while old rows are read as-is. This is the pattern the peer's platform will need.
- **PII-safe back-fill.** For back-fills that include user input, apply a Presidio-shaped scrubber before writing to `eval_cases` / `eval_results`; document the scrubber's coverage and its known false negatives. Chapter 03 covers the retirement pattern; this stretch covers the scrub.
- **OpenLineage catalogue integration.** Point the emit at a real DataHub / OpenLineage marquez instance; confirm the eval-runs appear in the catalogue with the right upstream (`eval_set`) and downstream (`eval_results`) dataset lineage.

## What this exercise does *not* cover

You are not building the contribution surface for `eval_cases` (that is exercise 02); the runner-adapter plane that will produce new `eval_runs` at scale (exercise 03); the platform SLI dashboard the reproducibility check will feed (exercise 04); or the CI + online-loop integration that will trigger runs against the schema (exercise 05). You are shipping the *substrate* — the tables, the hashes, the back-fill, and the reproducibility guarantee — that everything else in the module hangs off.
