# Trace Warehouse and Eval-Result Store: The Schema, the Lineage Keys, the Reproducibility Contract

## Motivation

The single question the schema has to answer is: *given a score, can I re-run the exact same thing and get the same score back?* If no, the score is a curiosity — you cannot attribute a regression to it, you cannot A/B against it, you cannot defend it to a reviewer. If yes, the score is *evidence*.

"The same thing" is not "the same model plus the same prompt." A concrete example: the mod-106 PR gate scores faithfulness at 0.87. A week later, the same rubric on the same replay bundle scores 0.79. What changed?

- The vendor rolled the API alias `gpt-4o` to a new snapshot silently (`model_snapshot` diff).
- Someone edited the prompt template but committed only the whitespace change (`prompt_hash` diff).
- The retrieval index was rebuilt overnight (`retriever_index_hash` diff).
- The `temperature=0.2, top_p=1.0` in the config was moved to `temperature=0.7` for a canary and never reverted (`decoding_config_hash` diff).
- The seed was left unset (`seed=null`, non-deterministic sampling — this alone is a 2 – 5 % score wobble).
- The rubric YAML had a comment added; the schema-blind hash ignored it; the schema-strict hash caught it (`rubric_hash` diff).
- The judge model snapshot behind `claude-3-5-sonnet-latest` bumped (`judge_model_snapshot` diff).
- A junior added three cases to the eval set and did not bump the version (`eval_set_hash` diff).

Any one of these can move a rubric score by more than the regression thresholds mod-106 chapter 07 sets. All eight must be recorded on every result row, or the reproducibility question has no answer. That set of eight is the **lineage key set**, and it is the schema's core invariant.

This chapter defines the tables, the lineage keys, the storage shape, and the reproducibility check every downstream chapter reads. It is a boring chapter by design — the schema is load-bearing exactly *because* it is boring.

## Core concepts

### The lineage key set

Nine hashes and snapshots, together, identify what was scored. Every result row carries every key. Missing a key silently degrades the reproducibility check; every column is required.

| Key | Type | Source | What it identifies | Why omission breaks reproducibility |
|---|---|---|---|---|
| `model_snapshot` | string | Vendor snapshot label (`gpt-4o-2024-08-06`, `claude-3-5-sonnet-20241022`) or local model + weight-hash for OSS | The exact model weights the app called | Vendor drift is invisible; scores move with no code change |
| `prompt_hash` | `sha256:...` | SHA-256 over the canonicalised prompt template (whitespace normalised, ordering fixed) | The prompt template body | Silent template edits move scores; without the hash, blame lands on the model |
| `chain_hash` | `sha256:...` | SHA-256 over the canonicalised chain / agent graph specification (nodes, edges, tool bindings, retries) | The multi-step orchestration | Reordering a chain changes results; without the hash, the harness looks fine |
| `retriever_index_hash` | `sha256:...` | SHA-256 over the retriever index build (doc set + chunker config + embedder snapshot + index build id) | The exact corpus scored | A nightly re-index quietly shifts every RAG result |
| `decoding_config_hash` | `sha256:...` | SHA-256 over `{temperature, top_p, top_k, max_tokens, stop, ...}` normalised to a canonical JSON | The decoding parameters | A one-line canary config edit produces a "regression" that is a config edit |
| `seed` | int or `null` | The RNG seed passed to the model — `null` means non-deterministic | Whether the run is reproducible at all | `null` is legal but must be recorded — a "non-deterministic" tag is not the same as "we forgot" |
| `eval_set_hash` | `sha256:...` | SHA-256 over the canonical serialisation of the eval-set contents (case_ids + case bodies, sorted) | The test cases scored | Adding a case silently is the most common way scores "move" |
| `rubric_hash` | `sha256:...` | SHA-256 over the canonicalised rubric spec (criteria, rating scale, judge prompt template) | The judge instructions | A judge-prompt tweak drifts the judge in a direction no rubric-name change reflects |
| `judge_model_snapshot` | string | Vendor snapshot label for the judge model | The judge behind the score | A silent judge upgrade is *the* single most common cause of "our evals moved and nobody deployed anything" |

Two rules the exercise enforces:

- **Every hash is a content hash.** Not a version tag ("prompt v3"), not a commit sha (which conflates unrelated edits), not the vendor's own identifier when there is one that is not stable. The hash is computed from the canonical bytes of the *content*. Chapter 03 defines the canonicalisation for eval sets; the same discipline applies to prompts, chains, rubrics, and decoding configs.
- **Every column is populated on every result row.** A `null` is legal on `seed` (non-deterministic sampling was intended). A `null` on the other keys is a schema violation — the row is unusable for reproducibility.

### The load-bearing tables

Seven tables carry the module's weight. Two are dimensional (`rubrics`, `eval_sets`), three are event-shaped (`eval_runs`, `eval_results`, `judgements`), and two are wide (`traces`, `spans` — usually the vendor's native storage, queryable through the plane). A concrete Postgres-flavoured DDL sketch is below; the shape is portable to ClickHouse, DuckDB, BigQuery, or Snowflake.

```
rubrics
  rubric_id            text primary key      -- e.g., "faithfulness_v3"
  rubric_hash          text unique not null  -- sha256 of canonical spec
  rubric_spec          jsonb not null        -- criteria, rating scale, judge prompt template
  authored_by          text not null
  created_at           timestamptz not null
  supersedes           text                  -- rubric_id of prior version, nullable

eval_sets
  eval_set_id          text primary key      -- e.g., "support_bot_golden_v7"
  eval_set_hash        text unique not null  -- sha256 of canonical case list
  suite                text not null         -- "golden", "regression", "rot_check", "safety_v1"
  owner_team           text not null         -- cost centre routing target
  case_count           int not null
  tag_set              text[] not null       -- e.g., ["billing", "onboarding", "critical"]
  created_at           timestamptz not null
  supersedes           text                  -- eval_set_id of prior version, nullable

eval_cases
  case_id              text primary key      -- stable across suite versions
  eval_set_id          text not null references eval_sets(eval_set_id)
  case_body            jsonb not null        -- inputs, expected outputs, references
  case_hash            text not null         -- sha256 of case_body, for change detection
  tags                 text[] not null
  contributor          text not null
  contributed_at       timestamptz not null

eval_runs
  run_id               text primary key      -- ulid or uuid; stable idempotency key
  spec                 jsonb not null        -- the EvalRunSpec dispatched
  runner               text not null         -- "promptfoo" | "deepeval" | "ragas" | "braintrust" | "weave" | "langfuse" | "custom:<name>"
  cost_centre          text not null         -- "team_id/project_id"
  triggered_by         text not null         -- "pr_gate" | "deploy_gate" | "online_loop" | "scheduled" | "manual:<user>"
  triggered_at         timestamptz not null
  started_at           timestamptz
  ended_at             timestamptz
  status               text not null         -- "queued" | "running" | "succeeded" | "failed" | "cancelled" | "throttled"
  status_reason        text                  -- for failed / cancelled / throttled
  -- lineage keys (denormalised for query speed; also present on eval_results)
  model_snapshot       text not null
  prompt_hash          text not null
  chain_hash           text not null
  retriever_index_hash text not null
  decoding_config_hash text not null
  seed                 bigint                -- nullable by design
  eval_set_id          text not null references eval_sets(eval_set_id)
  eval_set_hash        text not null         -- redundant with eval_set_id lookup; kept for immutability
  rubric_id            text not null references rubrics(rubric_id)
  rubric_hash          text not null
  judge_model_snapshot text not null
  -- cost roll-up
  cost_usd             numeric(12,6)
  judge_calls          int
  cache_hits           int

eval_results
  result_id            bigserial primary key
  run_id               text not null references eval_runs(run_id)
  case_id              text not null references eval_cases(case_id)
  metric               text not null         -- "faithfulness" | "safety_refusal" | "groundedness" | ...
  score                double precision      -- numeric metrics
  verdict              text                  -- categorical metrics ("PASS" | "FAIL" | "REFUSE" | ...)
  score_confidence     double precision      -- optional; judge self-reported confidence
  trace_id             text                  -- links to traces / spans table
  session_id           text                  -- links across turns
  -- lineage keys copied from eval_runs (denormalised for query speed)
  model_snapshot       text not null
  prompt_hash          text not null
  chain_hash           text not null
  retriever_index_hash text not null
  decoding_config_hash text not null
  seed                 bigint
  eval_set_hash        text not null
  rubric_hash          text not null
  judge_model_snapshot text not null
  -- source discriminator
  source               text not null         -- "offline_gate" | "online_loop" | "shadow" | "canary" | "backfill"
  scored_at            timestamptz not null

judgements
  judgement_id         bigserial primary key
  result_id            bigint not null references eval_results(result_id)
  judge_tier           text not null         -- "oss" | "mid" | "frontier"
  judge_model_snapshot text not null
  judge_prompt_hash    text not null
  judge_input          jsonb not null        -- the exact text sent to the judge
  judge_output         jsonb not null        -- the exact text received
  cost_usd             numeric(12,6)
  latency_ms           int
  scored_at            timestamptz not null

traces / spans
  -- Usually the vendor's own store (Phoenix, Langfuse, Weave, Braintrust)
  -- Queryable through the plane; joined to eval_results via trace_id.
  -- OTel-GenAI conformant (mod-102).
```

Read every column. Each is required by at least one downstream query.

- The **denormalisation of the lineage keys onto `eval_results`** is deliberate. A canonical relational answer would say "join to `eval_runs`." In practice, the reproducibility check runs on billions of result rows per quarter; the join is a query-planner risk. The keys are cheap; store them where they are read.
- The `run_id` is a stable idempotency key. Retries do not create new run rows; they update `status` and `status_reason`. Chapter 05 covers the retry / idempotency contract.
- `case_id` is stable across suite versions. Adding a new case bumps the `eval_set_hash` but the existing cases keep their ids. Chapter 03 covers the case-id contract.
- `verdict` and `score` are both present so the same table can hold numeric-metric rows (faithfulness, groundedness) and categorical-verdict rows (safety refusal, injection outcome). A row uses one or the other; a check enforces exactly one is populated.
- `source` is the row's provenance. A drift monitor that mixes `offline_gate` and `online_loop` rows produces meaningless numbers; the discriminator is what keeps them separate downstream.

### Why hashes are content hashes, not version tags

A version tag looks convenient — "prompt v3" reads better than `sha256:beef...`. Three problems:

- **Two people cannot both bump v3 without a race.** A content hash is generated deterministically from the content; a version tag needs a monotonically-increasing register that has to be reserved. In practice, the register is a manual field that goes out of sync.
- **A comment-only edit shows as v4.** A content hash over the *canonicalised* content (comments stripped, whitespace normalised, ordering fixed) shows no diff. The reproducibility check would not fire.
- **A rollback bumps the version.** A rebased branch that reintroduces the v2 template becomes "v5." A content hash is *the same hash as v2* — the schema knows the rollback did what the rollback was supposed to do.

The version tag is a *label*, useful for humans. Chapter 03 uses `eval_set_id` as the human-readable label with a `supersedes` pointer for lineage. The hash is what the reproducibility check trusts.

### The canonicalisation contract

A content hash is only useful if two people who wrote "the same" content produce the same hash. The canonicalisation contract fixes the serialisation before hashing.

For each hashable artefact:

- **Prompt template.** Normalise line endings to `\n`; strip trailing whitespace on each line; strip a leading / trailing blank line; JSON-encode as `{"template": "...", "variables": ["..."]}` with sorted keys; UTF-8 bytes → SHA-256. Consider variable names semantic, not incidental; a `{{customer_name}}` → `{{customer}}` rename is a real change that should bump the hash.
- **Chain / agent graph.** Serialise the graph as a topologically-sorted list of nodes with edges expressed as parent references; sort node children by node id; JSON-encode; hash. LangGraph / LlamaIndex / custom orchestrators each have a way to emit this canonical form; if yours does not, the adapter's job is to add it.
- **Retriever index.** Hash the *contents* of the corpus (per-document hash aggregated into a Merkle root over sorted document ids) plus the chunker config and the embedder snapshot. A nightly re-index that "changes nothing" but rebuilds the index changes the build id but not the content hash — this is the correct answer, and it lets the reproducibility check succeed across re-indexes.
- **Decoding config.** JSON-encode the sorted `{temperature, top_p, top_k, max_tokens, stop, response_format, ...}` object; hash.
- **Rubric spec.** JSON-encode the sorted `{criteria, rating_scale, judge_prompt_template, tie_break_rule}`; hash.
- **Eval set.** JSON-encode the sorted list of `{case_id, case_hash}` pairs; hash. See chapter 03 for `case_hash` derivation.

Every canonicalisation function has an inverse test: two inputs that a human would call equivalent must hash to the same value; two that a human would call different must not. Chapter 03 exercises this on the eval-set case.

### Storage shape: hot row store, warm columnar tier

The schema has two access patterns:

- **Hot** — write-heavy, single-run reads. The PR gate writes ~1 000 result rows per run and reads them back within seconds to render the pass / fail. The online loop writes hundreds of thousands of rows per day at 1 – 100 rps sustained. A row store — Postgres, MySQL, the vendor's own store — is the right fit here. Latency matters; joins are on `run_id`; the working set is small.
- **Warm** — read-heavy, cross-run analysis. The mod-107 drift monitor queries `eval_results` filtered by `cohort_keys` and grouped by day across weeks. The mod-108 report queries by `suite`. The mod-111 cost-trade-off dashboard queries by `cost_centre`. A columnar tier — ClickHouse, DuckDB, BigQuery, Snowflake, Redshift, Iceberg-on-object-store — is the right fit here. Throughput matters; joins are wide; the working set is large.

The pattern the exercise builds: writes go to the hot store; a scheduled export (hourly or nightly, depending on volume) copies aged rows to the warm tier; the warm tier is *append-only*; the plane routes analytical queries there. The vendor's own store (Phoenix, Langfuse, Weave, Braintrust) may play either role — Braintrust and W&B Weave, in particular, expose columnar backends for cross-run analysis natively. If it does, the plane exposes the vendor's read surface through the same query interface; if it does not, the plane exports.

### OpenLineage as the interchange format

The org's data-platform team probably already runs OpenLineage — an open standard for tracking dataset and job lineage across pipelines. The eval platform slice benefits from speaking it:

- Every `eval_run` emits an OpenLineage `RunEvent` with the `inputDatasets` (the eval set), the `outputDatasets` (the eval-result rows), and a `Job` facet naming the runner.
- The `parentRun` facet chains the run to the CI event that triggered it — the mod-106 PR gate, the mod-107 online-loop window, the mod-108 report render.
- The org's data-platform catalog (DataHub, Amundsen, or the data team's homegrown catalogue) can then answer "which prompt version produced which report" without the eval team writing a bespoke lineage viewer.

The eval-data-platform slice does not *require* OpenLineage — the internal schema is complete on its own — but it is the cheapest way to publish lineage to the rest of the org. Chapter 04's runner adapters emit the event; chapter 06's platform dashboard reads back the catalog.

### The reproducibility contract

The chapter's headline artefact. In one paragraph:

> Given an `eval_run` row, the platform guarantees that a re-dispatch of `EvalRunSpec` with the same `(model_snapshot, prompt_hash, chain_hash, retriever_index_hash, decoding_config_hash, seed, eval_set_hash, rubric_hash, judge_model_snapshot)` will produce identical `eval_results` rows for the deterministic subset (`seed != null` and no vendor-side non-determinism) and statistically indistinguishable rows for the stochastic subset (`seed == null` or vendor non-determinism). The exercise ships a `reproducibility_check` that runs a chosen historic run against the current platform and asserts the equivalence.

Two failure modes the check catches:

- **A schema regression.** A migration silently dropped `judge_prompt_hash` from the `judgements` table. The re-run "reproduces" but the check no longer knows which judge prompt was used. Alert.
- **A vendor drift.** `claude-3-5-sonnet-20241022` is still the snapshot the row names, but the vendor has since retired that snapshot and silently aliases it to `20250201`. The re-run's judgements diverge. Alert; escalate; consider re-scoring the affected historic window if the drift is material.

The reproducibility check is the *load-bearing SLI* for chapter 06 — it is the platform's own health check. If it fails and no one notices, the whole slice is retroactively suspect.

### What the schema does *not* try to be

- **A queryable versioned code base.** The prompt / chain / rubric / eval-set *contents* live in Git (or a versioned object store); the schema stores their hashes and a pointer. Storing the full body is legal for small artefacts but the source of truth is the version-controlled artefact, not the row.
- **A general-purpose data warehouse.** The org's data platform is that. The slice's warm tier is either the same warehouse (as a set of eval-owned schemas) or a small dedicated store the org's data platform mirrors.
- **A UI.** Chapter 03's contribution UI is minimal — enough for a non-engineer to add a case; it is not a full eval-management console. The runners' own UIs (Braintrust's experiment view, Weave's traces UI, Langfuse's evaluators view) are the primary human surface; the plane exposes their links and glues their data together.

Confusing these three is the "vendor is the platform" and "one giant table" anti-patterns chapter 01 warned about.

## Summary

- The schema's single job is to answer *can I re-run this and get the same score back?* — the **reproducibility contract**.
- The **lineage key set** is nine keys — `model_snapshot`, `prompt_hash`, `chain_hash`, `retriever_index_hash`, `decoding_config_hash`, `seed`, `eval_set_hash`, `rubric_hash`, `judge_model_snapshot`. Every result row carries every key.
- Seven load-bearing tables: `rubrics`, `eval_sets`, `eval_cases`, `eval_runs`, `eval_results`, `judgements`, plus `traces` / `spans` (usually the vendor's native store).
- The lineage keys are **denormalised onto `eval_results`** — the read pattern is cross-run analysis at scale; the join is a query-planner risk.
- Hashes are **content hashes**, not version tags. Version tags are labels; hashes are what the check trusts.
- **Canonicalisation** is the contract that makes the hash work — normalise line endings, sort keys, strip incidental whitespace, hash canonical bytes.
- Storage is **hot row store** (Postgres or vendor-native) for writes and single-run reads, **warm columnar tier** (ClickHouse, BigQuery, DuckDB, Iceberg) for cross-run analysis.
- **OpenLineage** is the cheapest interchange to the org's data-platform catalogue; the runners emit `RunEvent`s; the catalog reads them back.
- The **`reproducibility_check`** is the schema's headline SLI — it exercises the whole store against a historic run and asserts the guarantee holds. Chapter 06 elevates it to a platform-SLI dashboard tile.
- What the schema is *not*: a code base (Git is), a general-purpose warehouse (the data-platform team's is), a UI (the runners' UIs are).

Chapter 03 opens the test-case management surface — how product teams add and version cases without touching the harness, and how the `eval_set_hash` in this schema stays a truthful summary of what they contributed.
