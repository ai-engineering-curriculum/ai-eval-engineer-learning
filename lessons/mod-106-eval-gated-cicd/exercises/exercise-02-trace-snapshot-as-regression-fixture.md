# exercise-02: Trace Snapshot as Regression Fixture

**Estimated effort:** 3 hours

## Objective

Turn real production traces from your mod-102 trace backend into **regression fixtures** — the self-contained per-request records that chapter 02 defines. You will snapshot three representative traces, hand-author a fourth incident-derived fixture, verify the four pins are in place, and prove that the CI gate you built in exercise-01 fails on a synthetic regression against these fixtures.

By the end you will have a small but real replay set — the source material for the rest of the module — and the fixture pipeline that produces more of them on demand from a trace id.

## Prerequisites

- Chapter 02 of this module.
- Exercise-01 completed. This exercise's output feeds into that gate.
- A running trace backend from mod-102 (Phoenix, Langfuse, Weave, Braintrust, or LangSmith) with at least a few dozen real traces from your instrumented surface. If your backend is empty, generate traces first by running your surface against a small set of inputs.
- A retrieval index with a computable hash. If the surface has no RAG, skip the retrieval-hash pin and note it in the fixture schema doc.
- Python 3.11+.

## Set-up

1. Pick a surface with **at least one RAG call, at least one tool call, and at least one multi-turn interaction**. If your mod-102 surface is single-turn LLM-only, extend it (a small tool stub is enough).
2. Generate ~30 real traces by running the surface against a mix of inputs. Include one input you *know* would regress if the prompt were poorly changed — you will use it as the synthetic regression in the acceptance test.
3. Set up read access to your trace backend from a script (API key or SDK). Do this end-to-end once before you start building the fixture pipeline.

## Requirements

Produce a directory `eval/` (extending exercise-01) containing:

1. **`eval/fixtures/` — four fixture files.** Each must strictly follow the chapter 02 schema. At minimum:
   - Two fixtures **sampled from representative traffic** (happy-path). Selection strategy: stratify across the surface's segments (different tool paths, different session lengths, different query shapes).
   - One fixture **hand-authored as an edge case** — an empty query, an ambiguous query, an out-of-scope query, or a query with PII. Note in `added_reason` which class of edge case it exercises.
   - One fixture **modelled as an incident back-fill** — pretend you had an on-call regression, describe it in `added_reason` with a link to a fake issue id, and set the `reference` to what the correct behaviour should have been.
2. **`eval/scripts/snapshot.py`** — a script that takes a trace id and a target fixture id, queries the mod-102 backend, extracts the input, retrieval context, tool calls with args and returns, and metadata, computes / reads the four pins, scrubs PII per mod-102 chapter 06's policy, and writes a fixture YAML. Print the fixture path on completion. Support at least the backend you actually use; note in the script header which backends it covers.
3. **`eval/scripts/verify_pins.py`** — a script that reads a fixture and asserts:
   - `pin.model` includes an explicit snapshot suffix (e.g., matches a regex like `.+-\d{8}$` for OpenAI or is a Bedrock/Anthropic snapshot ARN).
   - `pin.retrieval_index_hash` matches the current index hash (or a whitelist of accepted historical hashes).
   - `pin.tool_stubs_hash` is present when `tool_calls[]` is non-empty.
   - `pin.system_fingerprint` and `pin.seed` are non-null when the vendor supports them.
   Exit non-zero on failure. This script runs in CI as one of the assertions from chapter 04.
4. **`eval/scripts/refresh_fixture.py`** — the "refresh" lifecycle event from chapter 02. Given a fixture id, re-fetch the reference material (retrieval context if the index changed, tool-return payloads) but explicitly does **not** overwrite `reference.answer` — the ground-truth field. Print a diff of what changed and require a `--yes` flag to write. The script is the mechanism that prevents the "insta-approve" trap.
5. **`eval/fixtures/schema.md`** — a short document (300–500 words) describing the fixture schema, the four pins and why each matters, the lifecycle events (add / refresh / retire / quarantine), and the CODEOWNERS rule.
6. **Two demonstration PRs (extend exercise-01):**
   - One that **adds a new fixture** via the `snapshot.py` pipeline, opens a PR, and the exercise-01 gate runs against the new fixture without changes to the surface. The gate passes.
   - One that **regresses a prompt** in a way that fails a specific fixture (e.g., strip the "cite the source documents" instruction, and confirm the incident-back-fill fixture's rubric fails). The gate goes red; the report names the specific failing fixture(s).
7. **CODEOWNERS entry** — extend the file so `/eval/fixtures/` and `/eval/scripts/` are owned by the eval team, per chapter 03.

## Starter guidance

- **PII scrub before write.** The trace backend may have already stripped PII per mod-102's policy, but do not assume. Run Presidio (or your team's scrubber) on the fixture input and any tool-argument fields before writing to disk. A fixture that leaks a real customer's email is a hard-to-recover mistake.
- **`system_fingerprint` may be null.** OpenAI documents it as best-effort; some providers do not emit it at all. Record what you have; do not fabricate a value. The `verify_pins.py` check should mark absent-when-supported as a fail and absent-when-unsupported as a pass.
- **Compute the index hash deterministically.** SHA-256 over the deterministic concatenation of `(doc_id, doc_version_hash)` for every document in the index. If your index also has chunker / embedder / ANN params that can move retrieval, include their versions in the hash input. Chapter 02's rule: rebuild the hash whenever the index rebuilds.
- **Do not overload one fixture with everything.** A fixture that tests four criteria is harder to refresh than four fixtures that each test one. Prefer many small fixtures; use `rubric_refs` to declare which rubrics apply.
- **`reference` is not `recorded_output`.** The trace's actual output goes in `recorded_output`. The correct output goes in `reference`. Chapter 02 warns about the "insta-approve" trap; the refresh script is the mechanism that prevents it.
- **Provenance is not optional.** Every fixture needs `source_trace_id`, `captured_at`, `added_reason`, and `owner`. The runbook (exercise-01) reads these when a fixture regresses.
- **Retire discipline.** If a fixture in `eval/fixtures/` no longer maps to a real surface behaviour (the feature was removed, the tool was renamed), retire it rather than leaving it. A trivially-passing fixture is worse than none.

## Acceptance criteria

You are done when:

- Four fixtures in `eval/fixtures/`, each with the full chapter 02 shape: input, retrieval context (if RAG), tool calls with stubbed responses, reference material, pin block, provenance.
- `snapshot.py` can turn a trace id from your backend into a valid fixture on the disk, in one command.
- `verify_pins.py` passes on all four fixtures. Manually break one pin (delete `retrieval_index_hash`) and confirm the script fails with a readable message.
- `refresh_fixture.py` prints the diff and requires `--yes` before overwriting non-reference fields; it refuses to touch `reference.answer`.
- The exercise-01 PR gate runs against the four fixtures and passes on `main`.
- The synthetic regression PR (prompt change) fails the gate with the specific fixture ids named in the report — not just an aggregate score drop.
- CODEOWNERS protects `/eval/fixtures/` and `/eval/scripts/`; a fixture-touching PR from a non-eval-team author requires eval-team review.
- `schema.md` explains the schema, the pins, the lifecycle, and the CODEOWNERS rule in enough detail that a new engineer can add a fixture unaided.

## Stretch goals

- **Quarantine sub-suite.** Add `eval/fixtures/quarantine/` with a lifecycle rule: fixtures move here when they flake unresolvably (e.g., vendor snapshot rolled and the previous reference no longer holds). The gate runs the quarantine set but does not block on its failures; a cron reports quarantine size weekly. Time-box every entry with `quarantined_until` and a linked issue.
- **Fixture from every incident.** Add a `POSTMORTEM.md` template with a mandatory "Fixture added?" checkbox and a link to `snapshot.py`. Wire the CODEOWNERS rule so a postmortem PR without a fixture link fails a linter.
- **Cross-language fixture stratification.** If your surface handles more than one language, add a stratification report — a script that prints the fixture distribution across language / segment / tool-path. Rebalance if any bucket is empty.
- **Manifest + object store for large binaries.** If your surface includes images / audio, extend the fixture schema with an `attachments:` block whose values are S3 / GCS URIs plus SHA-256 hashes. Write a `fetch_attachments.py` that hydrates the local cache on CI.
- **Rubric-hash pin.** Extend the pin block with `pin.rubric_hash` per fixture. Assert in `verify_pins.py` that the rubric file on disk hashes to the pinned value. When the rubric changes, the mismatch surfaces on the PR review — no silent rubric drift.

## What this exercise does *not* cover

You are not authoring the eval-config runner surface (exercise-03 / 04). You are not writing the release-gate runbook (exercise-05). You are supplying the substrate the gate scores against, and enforcing the pin discipline that makes those scores comparable across CI runs.
