# lab-01: Compute recall@k, MRR, and hit rate by Hand, Then by Code, Then Shift the Attribution

**Estimated duration:** 45–60 minutes
**Pairs with:** chapter 02 of this module.

## Objective

Make chapter 02's **retrieval-only metrics** physical. You compute `recall@k`, `MRR`, and `hit rate@k` on paper for a 10-item canned gold-set fixture; then you check the same numbers with a short stdlib-Python script; then you make surgical edits to the retrieved chunk-id lists — drop a relevant chunk, move a relevant chunk down the ranking, swap top-k's — and watch each metric shift the way chapter 02 says it should. The scorer is ~40 lines of pure set / list arithmetic with no LLM call and no framework; the point is that attribution on the retrieval side is cheap.

At the end, the phrase "recall@5 dropped 6 points on the multi-hop slice" produces a concrete mental picture of which three set operations ran and which per-query denominators went into the mean.

This lab deliberately does *not* duplicate the module's exercises:

- Exercise-01 scores the four triad metrics in RAGAS + TruLens on your real dataset with a real judge API.
- Exercise-02 builds the full attribution CLI (`retrieval_metrics.py` + `generator_replay.py` + `retriever_replay.py` + `attribute.py`) against your real surface's retriever and generator, with the chapter-02 decision tree walked on two seeded regressions.
- Exercise-03 grows the product-shaped gold set with three signature slices.

This lab gives you a canned 10-item fixture and asks you to compute the three retrieval-only metrics by hand first — so that when exercise-02 runs on your real retriever, you already know what every number in `results-retrieval.csv` means and why it moved.

## Prerequisites

- Chapter 02 read.
- Python 3.10+ on `$PATH` (`python3 --version`). Stdlib only; no `scipy`, no `sklearn`, no `ragas`.
- A working directory you can write files into. **No internet, no model API keys, no Docker** — the fixture is JSON, the scorer is pure set arithmetic.
- A pencil and paper. Step 2 is deliberately faster on paper than in the shell, and the point is that these metrics are hand-computable.

## Layout you will build

```
mod-105-lab-01/
  fixtures/
    gold.jsonl                  # 10 queries with relevant_chunk_ids + a slice tag
    retrieved_v1.jsonl          # baseline retriever output (k=5)
    retrieved_v2_dropped.jsonl  # surgical edit: drop one relevant chunk from top-5
    retrieved_v2_reordered.jsonl # surgical edit: push relevant chunks down the ranking
  retrieval_metrics.py           # the scorer — recall@k, MRR, hit rate@k, precision@k
  run.py                         # one-screen CLI driver
```

---

## Steps

### Step 1 — Create the working directory

```bash
mkdir -p mod-105-lab-01/fixtures && cd mod-105-lab-01
```

Confirm you are in an empty directory with `fixtures/` inside:

```bash
ls -a
```

**Expected:** `.`, `..`, `fixtures`.

### Step 2 — Author the gold set

Ten queries with hand-authored `relevant_chunk_ids` and a slice tag. Three queries are `single_chunk` (one relevant id), four are `multi_chunk` (two or three relevant ids — the chapter-02 "multi-chunk answer" case where recall@k is sharper than hit rate), and three are `out_of_scope` (empty `relevant_chunk_ids` — the chapter-03 case where retrieval metrics are undefined by design).

Create `fixtures/gold.jsonl`:

```json
{"query_id": "q01", "query": "How long is the parental leave policy?", "relevant_chunk_ids": ["c-leave-3"], "slice": "single_chunk"}
{"query_id": "q02", "query": "What is the refund window?", "relevant_chunk_ids": ["c-refund-1"], "slice": "single_chunk"}
{"query_id": "q03", "query": "How do I reset my admin password?", "relevant_chunk_ids": ["c-auth-7"], "slice": "single_chunk"}
{"query_id": "q04", "query": "How do parental and bereavement leave stack?", "relevant_chunk_ids": ["c-leave-3", "c-leave-5"], "slice": "multi_chunk"}
{"query_id": "q05", "query": "Which 2FA methods are allowed for admin roles across the two policy docs?", "relevant_chunk_ids": ["c-auth-7", "c-auth-9"], "slice": "multi_chunk"}
{"query_id": "q06", "query": "Can I combine the refund window with the warranty return window?", "relevant_chunk_ids": ["c-refund-1", "c-warranty-2"], "slice": "multi_chunk"}
{"query_id": "q07", "query": "What were the refund, warranty, and shipping policies all in one place?", "relevant_chunk_ids": ["c-refund-1", "c-warranty-2", "c-ship-4"], "slice": "multi_chunk"}
{"query_id": "q08", "query": "What was the 2019 parental leave policy?", "relevant_chunk_ids": [], "slice": "out_of_scope"}
{"query_id": "q09", "query": "How do I file a US federal tax return?", "relevant_chunk_ids": [], "slice": "out_of_scope"}
{"query_id": "q10", "query": "Who signed off on §4.2?", "relevant_chunk_ids": [], "slice": "out_of_scope"}
```

### Step 3 — Author the baseline retriever output

Each line records what the retriever surfaced for one query — the ordered top-5 chunk ids, as the mod-102 RETRIEVER span records them on `app.rag.retrieved_chunk_ids`. The baseline retriever is "pretty good but not perfect":

- `q01`–`q03` (single-chunk, in-scope): relevant chunk is **rank 1**.
- `q04`–`q05` (multi-chunk, in-scope, 2 relevant): both relevant chunks are in the top-5; one of them is at **rank 1**, the other is at **rank 3**.
- `q06`: one relevant at rank 1, the other (`c-warranty-2`) is **not retrieved** — hit rate still 1, recall@5 = 0.5.
- `q07` (3 relevant): two of three retrieved, one not — recall@5 = 2/3.
- `q08`–`q10` (out-of-scope): retriever still surfaces *some* chunks (the chapter-03 case) — the retrieval-only metrics are undefined on these items; the lab excludes them from the aggregate.

Create `fixtures/retrieved_v1.jsonl`:

```json
{"query_id": "q01", "retrieved_chunk_ids": ["c-leave-3", "c-leave-5", "c-leave-7", "c-leave-9", "c-hr-1"]}
{"query_id": "q02", "retrieved_chunk_ids": ["c-refund-1", "c-refund-3", "c-warranty-2", "c-refund-5", "c-ship-4"]}
{"query_id": "q03", "retrieved_chunk_ids": ["c-auth-7", "c-auth-9", "c-auth-3", "c-auth-5", "c-auth-1"]}
{"query_id": "q04", "retrieved_chunk_ids": ["c-leave-3", "c-leave-7", "c-leave-5", "c-leave-9", "c-hr-1"]}
{"query_id": "q05", "retrieved_chunk_ids": ["c-auth-7", "c-auth-5", "c-auth-9", "c-auth-3", "c-auth-1"]}
{"query_id": "q06", "retrieved_chunk_ids": ["c-refund-1", "c-refund-3", "c-refund-5", "c-ship-4", "c-hr-1"]}
{"query_id": "q07", "retrieved_chunk_ids": ["c-refund-1", "c-ship-4", "c-refund-3", "c-refund-5", "c-hr-1"]}
{"query_id": "q08", "retrieved_chunk_ids": ["c-leave-5", "c-hr-1", "c-leave-7", "c-leave-9", "c-leave-3"]}
{"query_id": "q09", "retrieved_chunk_ids": ["c-ship-4", "c-hr-1", "c-refund-5", "c-auth-5", "c-leave-9"]}
{"query_id": "q10", "retrieved_chunk_ids": ["c-auth-3", "c-hr-1", "c-leave-9", "c-refund-5", "c-ship-4"]}
```

### Step 4 — Hand-compute the metrics on paper

Chapter 02's three retrieval-only metrics, as formulas:

- **recall@k** = `|retrieved_top_k ∩ relevant| / |relevant|`, per query; aggregate = mean over in-scope queries.
- **MRR** = mean over queries of `1 / rank_of_first_relevant` (0 if no relevant chunk is in the top-k).
- **hit rate@k** = fraction of queries with at least one relevant chunk in the top-k.

For `k = 5`, compute each query's `recall@5` and reciprocal rank by hand. Out-of-scope queries (`q08`–`q10`) have `|relevant| = 0`; `recall@5` is undefined — exclude them from the mean. Writing the per-query intermediates saves time on step 6.

For a reality check, here is the expected shape of the answer on the baseline:

| query | slice | `|rel|` | `rel ∩ top-5` | recall@5 | rank of first rel | 1/rank |
|---|---|---|---|---|---|---|
| q01 | single_chunk | 1 | 1 | 1.00 | 1 | 1.00 |
| q02 | single_chunk | 1 | 1 | 1.00 | 1 | 1.00 |
| q03 | single_chunk | 1 | 1 | 1.00 | 1 | 1.00 |
| q04 | multi_chunk | 2 | 2 | 1.00 | 1 | 1.00 |
| q05 | multi_chunk | 2 | 2 | 1.00 | 1 | 1.00 |
| q06 | multi_chunk | 2 | 1 | 0.50 | 1 | 1.00 |
| q07 | multi_chunk | 3 | 2 | 0.67 | 1 | 1.00 |
| q08 | out_of_scope | 0 | — | n/a | — | — |
| q09 | out_of_scope | 0 | — | n/a | — | — |
| q10 | out_of_scope | 0 | — | n/a | — | — |

Aggregates over the 7 in-scope queries:

- recall@5 ≈ (1 + 1 + 1 + 1 + 1 + 0.5 + 0.67) / 7 ≈ **0.881**.
- MRR = (1 × 7) / 7 = **1.000** (every in-scope query has a relevant at rank 1 in the baseline).
- hit rate@5 = 7 / 7 = **1.000**.

Per-slice aggregates (chapter 02 insists on per-slice reporting with denominators):

- **single_chunk** (n = 3): recall@5 = 1.00, MRR = 1.00, hit rate@5 = 1.00.
- **multi_chunk** (n = 4): recall@5 = (1 + 1 + 0.5 + 0.67) / 4 ≈ **0.792**; MRR = 1.00; hit rate@5 = 1.00.

Note: the **single_chunk mean hides the multi_chunk recall drop**. If this were the only aggregate reported, the recall deficit on `q06` and `q07` would be invisible. This is chapter 02's "slice every metric" rule made concrete on a 10-item fixture.

### Step 5 — Write the scorer (`retrieval_metrics.py`)

Pure stdlib, no LLM call, no framework. Chapter 02 is explicit that this should run in well under 10 seconds on 100 items — set operations on chunk-id lists.

```python
# retrieval_metrics.py
"""Chapter-02 retrieval-only metrics on a labelled gold set.

Three metrics, one slice-aware aggregator, no LLM call:
  recall@k   = |retrieved_top_k ∩ relevant| / |relevant|   (per query)
  MRR        = mean of 1/rank_of_first_relevant  (0 if none)
  hit_rate@k = fraction of queries with >=1 relevant in top-k

Also computes precision@k as a cross-check. Out-of-scope queries
(|relevant|==0) are excluded from the recall@k mean — chapter 02 is
explicit that these have an undefined recall by construction.
"""

from __future__ import annotations

import json
import statistics
from collections import defaultdict
from pathlib import Path


def per_query_metrics(relevant: list, retrieved: list, k: int) -> dict:
    """Metrics for one (relevant_chunk_ids, retrieved_chunk_ids) pair at top-k."""
    top_k = retrieved[:k]
    top_k_set = set(top_k)
    rel_set = set(relevant)
    hits = top_k_set & rel_set

    recall = (len(hits) / len(rel_set)) if rel_set else None  # undefined on OOS
    precision = len(hits) / k if k > 0 else 0.0
    hit = 1 if hits else 0

    # Reciprocal rank — walk top_k in order, find first relevant.
    rr = 0.0
    for rank, cid in enumerate(top_k, start=1):
        if cid in rel_set:
            rr = 1.0 / rank
            break

    return {
        "recall_at_k": recall,
        "precision_at_k": precision,
        "hit_at_k": hit,
        "reciprocal_rank": rr if rel_set else None,  # undefined on OOS
        "hits": sorted(hits),
        "top_k": top_k,
    }


def score(gold: list[dict], retrieved: list[dict], k: int) -> dict:
    """Score a whole gold set at one k; return per-query rows + per-slice aggregates."""
    by_qid = {row["query_id"]: row for row in retrieved}
    per_query = []
    per_slice_in_scope = defaultdict(list)
    overall_in_scope = []

    for row in gold:
        qid = row["query_id"]
        r = by_qid.get(qid)
        if r is None:
            per_query.append({"query_id": qid, "slice": row["slice"], "error": "no_retrieval"})
            continue
        m = per_query_metrics(row["relevant_chunk_ids"], r["retrieved_chunk_ids"], k)
        rec = {"query_id": qid, "slice": row["slice"], **m}
        per_query.append(rec)
        if m["recall_at_k"] is not None:  # in-scope only
            per_slice_in_scope[row["slice"]].append(rec)
            overall_in_scope.append(rec)

    def _agg(rows: list[dict]) -> dict:
        if not rows:
            return {"n": 0, "recall_at_k": None, "mrr": None, "hit_rate_at_k": None, "precision_at_k": None}
        return {
            "n": len(rows),
            "recall_at_k": round(statistics.fmean(r["recall_at_k"] for r in rows), 4),
            "mrr": round(statistics.fmean(r["reciprocal_rank"] for r in rows), 4),
            "hit_rate_at_k": round(statistics.fmean(r["hit_at_k"] for r in rows), 4),
            "precision_at_k": round(statistics.fmean(r["precision_at_k"] for r in rows), 4),
        }

    return {
        "k": k,
        "overall_in_scope": _agg(overall_in_scope),
        "per_slice": {s: _agg(rows) for s, rows in per_slice_in_scope.items()},
        "per_query": per_query,
    }


def load_jsonl(path: Path) -> list[dict]:
    return [json.loads(line) for line in path.read_text().splitlines() if line.strip()]


if __name__ == "__main__":  # pragma: no cover
    import sys
    gold = load_jsonl(Path(sys.argv[1]))
    retrieved = load_jsonl(Path(sys.argv[2]))
    k = int(sys.argv[3]) if len(sys.argv) > 3 else 5
    print(json.dumps(score(gold, retrieved, k), indent=2))
```

### Step 6 — Write the one-screen driver (`run.py`)

```python
# run.py
"""Prints the chapter-02 retrieval-only metrics in a compact table."""

import json
import sys
from pathlib import Path

from retrieval_metrics import load_jsonl, score


def main() -> int:
    gold = load_jsonl(Path(sys.argv[1]))
    retrieved = load_jsonl(Path(sys.argv[2]))
    k = int(sys.argv[3]) if len(sys.argv) > 3 else 5
    out = score(gold, retrieved, k)

    print(f"## Retrieval-only metrics @ k={k}  (file: {sys.argv[2]})")
    print()
    a = out["overall_in_scope"]
    print(f"Overall (in-scope, n={a['n']}):")
    print(f"  recall@{k}     : {a['recall_at_k']}")
    print(f"  MRR           : {a['mrr']}")
    print(f"  hit rate@{k}   : {a['hit_rate_at_k']}")
    print(f"  precision@{k}  : {a['precision_at_k']}")
    print()
    print(f"Per-slice:")
    for sname, agg in out["per_slice"].items():
        print(f"  {sname:14s} (n={agg['n']}): "
              f"recall@{k}={agg['recall_at_k']}, "
              f"MRR={agg['mrr']}, "
              f"hit@{k}={agg['hit_rate_at_k']}")
    print()
    print(f"Per-query:")
    for r in out["per_query"]:
        if "error" in r:
            print(f"  {r['query_id']} slice={r['slice']:14s} ERROR={r['error']}")
            continue
        rec = r["recall_at_k"] if r["recall_at_k"] is not None else "n/a"
        rr = r["reciprocal_rank"] if r["reciprocal_rank"] is not None else "n/a"
        print(f"  {r['query_id']} slice={r['slice']:14s} "
              f"recall@{k}={rec} 1/rank={rr} hits={r['hits']}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

### Step 7 — Run the scorer on the baseline

```bash
python3 run.py fixtures/gold.jsonl fixtures/retrieved_v1.jsonl 5
```

**Expected (the key aggregates):**

- Overall in-scope, n=7: `recall@5 ≈ 0.881`, `MRR = 1.0`, `hit rate@5 = 1.0`.
- `single_chunk` (n=3): `recall@5 = 1.0`.
- `multi_chunk` (n=4): `recall@5 ≈ 0.792`.
- Out-of-scope queries appear in the per-query printout with `recall@5 = n/a` and `1/rank = n/a`.

If the printed numbers don't match your hand computation from step 4, the hand computation is almost certainly right — re-check the scorer's set operations on the two queries where your paper number differs.

### Step 8 — Surgical edit #1: drop a relevant chunk from top-5

Simulate a retriever regression on the multi-chunk slice: for `q05`, swap the relevant chunk `c-auth-9` (at rank 3 in the baseline) out of the top-5 entirely. This is the chapter-02 "recall@k drops without MRR moving" scenario — the first relevant chunk is still at rank 1, so MRR is unchanged, but recall@5 drops because one of two required chunks is now missing.

```bash
python3 - <<'PY'
import json
from pathlib import Path

rows = [json.loads(line) for line in Path("fixtures/retrieved_v1.jsonl").read_text().splitlines() if line.strip()]
for r in rows:
    if r["query_id"] == "q05":
        # Drop c-auth-9 from top-5 by replacing it with an unrelated distractor.
        r["retrieved_chunk_ids"] = ["c-auth-7", "c-auth-5", "c-hr-1", "c-auth-3", "c-auth-1"]

Path("fixtures/retrieved_v2_dropped.jsonl").write_text("\n".join(json.dumps(r) for r in rows) + "\n")
PY
python3 run.py fixtures/gold.jsonl fixtures/retrieved_v2_dropped.jsonl 5
```

**Expected shift (compared to the baseline):**

- `q05` per-query: `recall@5` drops from **1.00 to 0.50**; `1/rank` stays at **1.00** (relevant `c-auth-7` is still at rank 1).
- Slice `multi_chunk`: `recall@5` drops from **0.792 to 0.667** (dragged down by the half-point loss on `q05`); `MRR` stays at **1.00**.
- Slice `single_chunk`: unchanged.
- Overall in-scope: `recall@5` drops ≈ 0.07, `MRR` unchanged.

Chapter 02's reading: the retriever lost a required chunk on a multi-chunk query. MRR cannot see it because the *first* relevant chunk is still at rank 1. Hit rate cannot see it either (there is still at least one relevant chunk in the top-5). **recall@k is the only one of the three that fires.** This is why chapter 02 insists you compute all three — hit rate is "did retrieval find anything", MRR is "how high is the first thing", recall@k is "did retrieval find enough."

### Step 9 — Surgical edit #2: push a relevant chunk down the ranking

Second regression, different shape: on `q01` (single-chunk, in-scope), move the relevant chunk `c-leave-3` from rank 1 to rank 4. recall@5 does not move (the chunk is still in the top-5); hit rate does not move (the chunk is still in the top-5); **only MRR drops** — the first relevant chunk is now at rank 4, so the reciprocal rank falls from 1.0 to 0.25 on that query.

```bash
python3 - <<'PY'
import json
from pathlib import Path

rows = [json.loads(line) for line in Path("fixtures/retrieved_v1.jsonl").read_text().splitlines() if line.strip()]
for r in rows:
    if r["query_id"] == "q01":
        # Push c-leave-3 from rank 1 to rank 4.
        r["retrieved_chunk_ids"] = ["c-leave-5", "c-leave-7", "c-leave-9", "c-leave-3", "c-hr-1"]

Path("fixtures/retrieved_v2_reordered.jsonl").write_text("\n".join(json.dumps(r) for r in rows) + "\n")
PY
python3 run.py fixtures/gold.jsonl fixtures/retrieved_v2_reordered.jsonl 5
```

**Expected shift:**

- `q01` per-query: `recall@5 = 1.00` (unchanged), `1/rank = 0.25` (down from 1.00).
- Slice `single_chunk`: `recall@5 = 1.00`, `MRR = (0.25 + 1 + 1) / 3 ≈ 0.75` (down from 1.00); hit rate@5 = 1.00.
- Slice `multi_chunk`: unchanged from baseline.

Chapter 02's reading: the chunk is there, just lower-ranked. For a surface that heavily weights the top result (chat / single-answer), this is still a real quality regression. For a surface that concatenates the whole top-k into one prompt, it may not matter. MRR is the metric that distinguishes those cases — recall@k and hit rate@k cannot.

### Step 10 — Compose the two edits, read both signals

Combine the two regressions: drop `c-auth-9` from `q05`'s top-5 *and* push `c-leave-3` down to rank 4 on `q01`. Confirm that **both** MRR (via `q01`) and recall@5 (via `q05`) move, on **different slices**.

```bash
python3 - <<'PY'
import json
from pathlib import Path

rows = [json.loads(line) for line in Path("fixtures/retrieved_v1.jsonl").read_text().splitlines() if line.strip()]
for r in rows:
    if r["query_id"] == "q05":
        r["retrieved_chunk_ids"] = ["c-auth-7", "c-auth-5", "c-hr-1", "c-auth-3", "c-auth-1"]
    if r["query_id"] == "q01":
        r["retrieved_chunk_ids"] = ["c-leave-5", "c-leave-7", "c-leave-9", "c-leave-3", "c-hr-1"]

Path("fixtures/retrieved_v2_combined.jsonl").write_text("\n".join(json.dumps(r) for r in rows) + "\n")
PY
python3 run.py fixtures/gold.jsonl fixtures/retrieved_v2_combined.jsonl 5
```

**Expected:** `single_chunk` slice shows MRR down (recall@5 unchanged); `multi_chunk` slice shows recall@5 down (MRR unchanged). An aggregate-only report that averaged across all 7 in-scope queries without the per-slice split would show both numbers mildly worse but would not tell you *which* slice drove *which* metric — exactly the "slice regressions that invert" failure chapter 02 warns about.

### Step 11 — Sweep k, read the shape

Re-run the baseline at `k = 1`, `k = 3`, and `k = 10`:

```bash
for k in 1 3 10; do
  echo "=== k=$k ==="
  python3 run.py fixtures/gold.jsonl fixtures/retrieved_v1.jsonl $k
done
```

**Expected shape:**

- **`k = 1`**: recall@1 on the multi_chunk slice drops dramatically (can only capture 1 of 2–3 required chunks) — ≈ 0.42 on this fixture. Hit rate@1 drops to 1.0 (every in-scope query still has its first-rank chunk relevant). MRR is 1.0 regardless of k when the first relevant chunk is at rank 1.
- **`k = 3`**: `c-auth-9` is at rank 3 on `q05`, so still inside top-3; multi_chunk recall@3 ≈ 0.792.
- **`k = 10`**: same as k = 5 on this fixture because the top-5 already exhausts the relevant chunks that are retrievable at all.

Chapter 02's reading: k is a design parameter, not a natural fact; the metric name always carries the k. A "recall dropped" number without the k is uninterpretable. Log `k` on every retrieval-only metric.

### Step 12 — Checkpoint

You are done with this lab when every statement below is true. If any is not, re-read the step it references.

- [ ] `python3 run.py fixtures/gold.jsonl fixtures/retrieved_v1.jsonl 5` prints `recall@5 ≈ 0.881`, `MRR = 1.0`, `hit rate@5 = 1.0` on the 7 in-scope queries.
- [ ] Per-slice aggregates report different `recall@5` for `single_chunk` (1.0) and `multi_chunk` (≈ 0.792).
- [ ] Out-of-scope queries are excluded from the recall@5 and MRR means (they appear in the per-query printout with `n/a`).
- [ ] The **dropped-chunk** edit moves `recall@5` on `multi_chunk` and leaves `MRR` untouched.
- [ ] The **reordered-chunk** edit moves `MRR` on `single_chunk` and leaves `recall@5` and `hit rate@5` untouched.
- [ ] You can state, for each of the three retrieval-only metrics, one failure shape it catches that the other two miss (hit rate cannot see missing-but-partially-covered, recall@k cannot see ranking position, MRR cannot see the second-and-later relevant chunks).
- [ ] You can explain why aggregate-only reporting would hide the slice regression in step 10, and name which `app.rag.*` attribute on the mod-102 trace carries the slice tag that makes the per-slice aggregation mechanical.

## Where this hands off

- `retrieval_metrics.py` is the exact shape exercise-02's `retrieval_metrics.py` asks you to productionise — swap the canned JSONL for the mod-102 trace's `app.rag.retrieved_chunk_ids` and `app.rag.gold_relevant_ids`, and the metric arithmetic is unchanged.
- The hand-computed denominators from step 4 are the discipline chapter 05's handoff report template asks for: "recall@5 dropped X on the multi_hop slice (n=Y)" — the `(n=Y)` is what makes the report actionable for the `rag-engineer` peer.
- The surgical-edit pattern in steps 8–10 is the arithmetic underneath exercise-02's two seeded regressions; this lab's fixtures are canned, exercise-02's run on your real retriever.
- The k-sweep in step 11 is the shape exercise-02 asks you to report in `results-retrieval.csv` — recall / MRR / hit rate / precision at `k ∈ {1, 3, 5, 10}`.
