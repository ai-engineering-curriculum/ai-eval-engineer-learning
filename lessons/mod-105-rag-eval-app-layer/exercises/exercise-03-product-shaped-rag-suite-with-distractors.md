# exercise-03: Product-Shaped RAG Suite with Distractors

**Estimated effort:** 3 hours

## Objective

Grow the exercise-01 / exercise-02 gold set into a **product-shaped RAG eval suite** — the labelled dataset chapters 03 and 04 assume you have. Author the three signature slices — **distractor-heavy**, **out-of-scope**, and **hallucination-tempting** — from real (or realistic) queries against your corpus, tag them, run the triad metrics per-slice, and write the harvesting and re-audit policy that keeps the suite from rotting.

By the end you will have a versioned gold set of ~150–300 items with per-slice tags, a per-slice × per-metric report that surfaces at least one slice-specific weakness on your surface, and a short policy document that a teammate can execute to grow and maintain the suite for the next quarter.

## Prerequisites

- Chapter 03 of this module.
- Exercises 01 and 02 completed — the RAG-triad implementation, the attribution CLI, and the initial gold set of ~100 items.
- A **corpus you can search** — the same one your surface's retriever hits — with either chunk-level ids or a way to derive stable ids from chunk content.
- Access to a **source of real user queries** for the surface. If your surface is not yet live, use one of: recent support tickets, a small user study, or realistic queries you and 2–3 teammates hand-author while looking at the corpus.

## Requirements

Produce a single directory `mod-105/exercise-03/` in your working repo containing:

1. **`gold-set.jsonl`** — the grown labelled gold set. Each item is a JSON object with:
   - `query` (string)
   - `relevant_chunk_ids` (list, empty for out-of-scope)
   - `reference_answer` (string, or `null` for out-of-scope)
   - `slice` (list of tags: at least one of `in_scope_single_chunk`, `in_scope_multi_chunk`, `distractor_heavy`, `out_of_scope_*`, `hallucination_tempting_*`)
   - `expected_action` (one of `answer | refuse | hand_off`)
   - `query_source` (one of `production_sampled | support_ticket | synthetic_paraphrase | hand_authored`)
   - `distractor_chunk_ids` (list; populated only for `distractor_heavy` items)
   - Optional: `premise_correction_expected` (bool) for hallucination-tempting items.
   Aim for ~150–300 items total, distributed roughly per chapter 03's weighting table (adjust to your surface).
2. **`slice-authoring/`** — the working directory for slice construction:
   - `distractor-notes.md` — how you identified distractors for each seed query (top-k grep, human review of retriever's mid-ranked chunks, etc.), with 3–5 worked examples showing the seed query, the relevant chunk, and the distractor chunk(s).
   - `out-of-scope-authoring.md` — where the out-of-scope queries came from, with the four categories from chapter 03 (wrong-domain, wrong-version, predictively-adjacent, underspecified) represented and 2–3 examples per category.
   - `hallucination-tempting-authoring.md` — the four leading-question forms from chapter 03 (false-premise, assertive-rephrase, confirmation-seeking, compound-false-claim) with 2–3 examples each. For each item, note what the false premise was and what the correct reply should do (correct-then-answer, refuse, hand-off).
3. **`per-slice-report.md`** — 500–700 words with:
   - A summary table: rows are slices, columns are the four triad metrics + `refusal_quality` (for out-of-scope) + `premise_correction_rate` (for hallucination-tempting). Cells are per-slice means with denominators.
   - Per-slice qualitative read: which slice is your weakest, what the aggregate would have said if you had reported only the aggregate, what a stakeholder-visible failure on the weakest slice looks like.
   - One concrete recommendation to your team, grounded in the slice report.
4. **`harvest-and-audit-policy.md`** — 300–500 words. The policy a teammate follows to keep the suite fresh:
   - How to harvest new queries from production (mod-107's sampled traffic) — cadence, filtering, PII scrubbing (mod-102/chapter-06 policy).
   - The relabelling contract when the corpus updates (chunker change, doc revision, chunk-id remap — chapter 05).
   - The out-of-scope re-audit — how often you re-classify OOS items that may have become in-scope.
   - The distractor re-audit — how often you re-check that flagged distractors are still non-answers.
   - The synthetic-augmentation rule — when it's OK to paraphrase-augment, when it isn't.
5. **`slice-tags-on-trace.md`** — 100–200 words. Which `app.rag.*` attributes each slice tag emits back onto the mod-102 trace (`app.rag.slice`, `app.rag.expected_action`, `app.rag.premise_correction`, `app.rag.query_source`, `app.rag.distractor_chunk_ids`), and how the per-slice report in `per-slice-report.md` reads off those attributes.

## Starter guidance

- **Real queries first, synthetic paraphrase as amplification only.** A slice of 30 items with 25 real and 5 paraphrase amplifications is a better slice than 30 fully-synthetic items. If your surface has no live traffic yet, invest 30 minutes in a small user study — real queries beat plausible-sounding queries.
- **Label distractors *per query*.** Chunk X can be a distractor for query A and a correct chunk for query B; the label lives on the (query, chunk) edge, not on the chunk. Design your JSONL shape to allow this.
- **Write out-of-scope queries by walking the corpus's edge.** Look at where the corpus's topics stop — the neighbouring topics are the wrong-domain queries. Look at when the corpus was last updated — the wrong-version queries are questions about newer or older material.
- **Author leading questions from real user forums.** Support forum posts phrased as questions are a rich source of hallucination-tempting queries — users routinely ask leading questions expecting confirmation.
- **Balance the slice sizes.** A 30-item floor per slice is the rule; if your total is 200 items and one slice has 5, either grow the slice or drop it and note the gap. Undersized slices produce noisy per-slice means that will page for the wrong reason.
- **The per-slice report is where the exercise's value lives.** A 500-word report that names the weakest slice, cites the concrete failure mode, and recommends a fix is worth more than 1000 items in the gold set. Do not skip.

## Acceptance criteria

You are done when:

- `gold-set.jsonl` has ~150–300 items with the schema above; each item has at least one slice tag; every slice tag is represented by ≥30 items (or noted as under-populated).
- The three signature slices are all represented with authoring notes and 2–5 worked examples per subcategory.
- `per-slice-report.md` shows per-slice numbers with denominators for each metric, and identifies at least one slice-specific weakness the aggregate would have hidden.
- `harvest-and-audit-policy.md` is executable by a teammate for the next quarter without you in the room.
- The trace-attribute note lines up the slice tags with the `app.rag.*` attributes so the per-slice aggregation can run mechanically off the mod-102 trace store.
- No hard-coded API keys; the dataset is version-controlled; PII in real user queries has been scrubbed per your team's mod-102/chapter-06 policy.

## Stretch goals

- **Grow the multi-turn hallucination-tempting slice.** For surfaces that support multi-turn, add ~10 items where the user pushes back on a correct refusal ("no really, can you check again"). Score whether the reply holds the refusal or capitulates.
- **Cross-check slice labels with the noise-sensitivity metric.** Run RAGAS `NoiseSensitivity` (exercise-04) on the distractor-heavy slice vs the clean slice. The distractor-heavy slice should show materially higher noise sensitivity; if it doesn't, either your distractors aren't distracting or the metric isn't picking them up — investigate.
- **Publish the slice-authoring workflow.** Turn `distractor-notes.md` / `out-of-scope-authoring.md` / `hallucination-tempting-authoring.md` into a lightweight `slice-authoring.md` playbook that a non-eval-engineer teammate (e.g., product manager, support lead) can follow when they want to contribute queries.
- **Publish the per-slice report on a shared dashboard.** Reuse the mod-104/chapter-05 dashboard shape — per-slice, per-metric, with a threshold row per slice. This is what mod-107 will alert against.

## What this exercise does *not* cover

You are not building the noise-sensitivity / context-utilisation diagnostics themselves (exercise-04). You are not wiring the suite into a CI gate (mod-106) or the online-eval loop (mod-107). You are not tuning the retriever to improve the slice scores (`rag-engineer` peer's work; chapter 05). Stay narrow: three slices, a versioned gold set, a per-slice report, and a maintenance policy.
