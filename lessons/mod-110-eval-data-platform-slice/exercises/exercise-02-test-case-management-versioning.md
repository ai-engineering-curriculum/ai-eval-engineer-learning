# exercise-02: Test-Case Management and Versioning

**Estimated effort:** 3 hours

## Objective

Ship the **test-case management surface** chapter 03 describes: the contribution flow that lets a product-team member — including a non-engineer — add, tag, and version test cases without touching the harness code, and the versioning machinery that keeps the chapter 02 `eval_set_hash` a truthful summary of every contribution.

By the end of this exercise, a non-engineer on a product team can either (a) open a PR in a dedicated `eval-sets/` repo folder with a small YAML file, or (b) click "add case" in an internal UI / submit through an API — and in either case the result is one new row in `eval_cases`, one bumped `eval_set` version with the new `eval_set_hash`, an audit row naming the contributor and the reviewer, and a cost-preview comment on the change.

This is the surface the **exercises 03 – 05** read from. Exercise 03's `RunnerAdapter.plan` will resolve an `eval_set_id` to a specific `eval_set_hash` from your `eval_set_current` pointer; exercise 04's platform dashboard reads the test-case-age tile from the rows you insert here; exercise 05's `gate_result.json` cites the `eval_set_hash` your versioning emits. Get the contribution flow boring and auditable; the plane downstream reuses it unchanged.

## Prerequisites

- Chapters 01 and 03 of this module. Chapter 02 if the schema layout for `eval_sets` / `eval_cases` / `eval_set_current` is not yet in muscle memory.
- The schema from **exercise-01** of this module — the `eval_sets`, `eval_cases`, and `eval_set_current` tables, the canonicalisation facade (`canonicalisation/eval_set.py`), and the `hashing.py` module. If exercise-01 is not done, stub them locally; this exercise's acceptance criteria read against them.
- A Git host that supports PR workflows (GitHub, GitLab, Gitea). GitHub Actions / GitLab CI is the shortest path for the CI validation step.
- Python 3.11+, `pydantic` v2 (schema validation), `pyyaml`, `click` or `typer` (CLI), one of `fastapi` / `flask` / `litestar` (contribution API), a tiny web template or a `streamlit`-shaped UI for the eval-set-as-database surface. The reference implementation below assumes FastAPI + a minimal HTML template.
- An auth source for the UI — any OIDC provider the org already uses is fine; a local dev shim (`X-Debug-User`) is legal for the exercise but must be a config flag that is **off** in staging.
- An existing suite seed: pull 20 – 50 cases from any one of your previous modules' exercises (mod-104's rubric gold set, mod-105's RAG eval set, mod-106's replay bundle, mod-108's jailbreak corpus). Any suite is fine — the shape matters, not the content.

## Set-up

1. Create `eval/platform/testcases/` in your repo:

   ```
   eval/platform/testcases/
   ├── config.yaml
   ├── repo/                              # eval-set-as-code surface
   │   ├── suites/
   │   │   ├── support_bot_golden/
   │   │   │   ├── MANIFEST.yaml          # suite-level metadata
   │   │   │   ├── cases/
   │   │   │   │   ├── CASE-support-billing-refund-2026-03-12.yaml
   │   │   │   │   └── ...
   │   │   │   └── CHANGELOG.md
   │   │   ├── support_bot_regression/
   │   │   └── support_bot_rot_check_2026_08/
   │   └── tag_catalog.yaml
   ├── schema/
   │   ├── case.py                        # pydantic model for a case
   │   ├── manifest.py                    # pydantic model for suite manifest
   │   └── tag_catalog.py                 # pydantic model + validator
   ├── versioning/
   │   ├── compute_hash.py                # thin wrapper over exercise-01's canonicaliser
   │   ├── semver.py                      # patch/minor/major classifier
   │   └── bump.py                        # applies the bump on merge
   ├── api/
   │   ├── app.py                         # FastAPI app
   │   ├── routes.py                      # submission, review, approval endpoints
   │   ├── review.py                      # four-eyes rule
   │   └── templates/                     # minimal HTML — contribution form, review queue
   ├── ci/
   │   ├── validate.py                    # schema, tag, case_id, case_hash, dry-run
   │   ├── cost_preview.py                # Δ cases × tier cost × rubrics
   │   └── github_action.yaml             # or gitlab-ci.yml
   ├── cli/
   │   └── eval_cases.py                  # `eval-cases` CLI — add / review / promote / retire
   ├── migrations/
   │   └── 0004_eval_cases_review_columns.sql  # audit columns on top of exercise-01 schema
   ├── tests/
   │   ├── test_schema.py
   │   ├── test_hash.py
   │   ├── test_semver.py
   │   ├── test_ci_validate.py
   │   ├── test_api.py
   │   └── test_roundtrip.py              # repo YAML ↔ DB row round-trip
   └── README.md
   ```

2. Fill in `config.yaml`:

   ```yaml
   repo:
     path: ./repo
     default_branch: main
     require_two_reviewers: true
   db:
     dsn: postgresql://eval:eval@localhost:5432/eval_platform
   api:
     bind: 0.0.0.0:8080
     auth: oidc                           # or "local_dev" (dev only; blocked in staging)
     oidc_issuer: https://auth.example.com
   cost_preview:
     pricing_snapshot: current            # falls back to exercise-01's pricing_snapshots
     mean_judge_cost_usd:
       oss: 0.0004
       mid: 0.004
       frontier: 0.04
   tag_catalog:
     path: ./repo/tag_catalog.yaml
   runners_for_dry_run:
     - custom                             # minimal; exercise-03 wires more
   semver:
     major_shape_fields: [case_body_schema_version]
   retention:
     retired_at_hard_delete_days: never   # retirement never hard-deletes
   ```

3. Author the suite `MANIFEST.yaml`:

   ```yaml
   suite_id: support_bot_golden
   owner_team: team-support
   subset_role: golden                    # golden | regression | rot_check
   semver: 1.0.0
   case_body_schema_version: 1
   description: >
     The golden set for the support bot. Load-bearing for the mod-106 PR gate.
     Changes require two reviewers plus a cost preview.
   tag_axes_required: [feature, capability, criticality]
   tag_axes_optional: [tenant_visible]
   runbook: ../../../runbooks/support_bot_golden.md
   ```

4. Author `tag_catalog.yaml`:

   ```yaml
   version: 1.0.0
   axes:
     feature:
       values: [billing, onboarding, cancellation, refund, account, other]
       owner: team-support
     capability:
       values: [extraction, retrieval, reasoning, generation, tool_call, safety_refusal]
       owner: team-eval
     criticality:
       values: [must_never_regress, target, aspiration]
       owner: team-eval
     tenant_visible:
       values: [yes, no]
       owner: team-support
   ```

5. Author one example case so the test suite has something concrete to validate:

   ```yaml
   # repo/suites/support_bot_golden/cases/CASE-support-billing-refund-2026-03-12.yaml
   case_id: CASE-support-billing-refund-2026-03-12
   case_body:
     schema_version: 1
     inputs:
       user_turn: "I was charged twice for May. Can I get one refunded?"
       channel: web_chat
     expected:
       answer_shape: refund_eligibility
       must_include_sources: true
   tags:
     feature: billing
     capability: [retrieval, reasoning]
     criticality: must_never_regress
     tenant_visible: yes
   contributor: alice@example.com
   contributed_at: 2026-03-12T10:14:00Z
   why: >
     Incident INC-2026-03-11 — bot answered without citing a source. This case
     anchors "must cite" for the billing feature.
   ```

6. Apply migration `0004_eval_cases_review_columns.sql`. It adds `proposed`, `submitted_by`, `submitted_at`, `reviewed_by`, `reviewed_at`, `approved_by`, `approval_reason`, `superseded_by`, `retired_at` to the exercise-01 `eval_cases` table. Keep the migration idempotent and reversible per chapter 02's rules.

## Requirements

Produce a PR against your working branch that adds:

1. **Case / suite / tag-catalog schema (`schema/*.py`)** — pydantic models that reject malformed cases at parse time. A missing required tag axis, an unknown tag value, a `case_id` that does not match the suite-prefix pattern, a `tags` block that is a string instead of a dict, a `case_body` whose `schema_version` is not accepted by the suite's `case_body_schema_version` → each is a specific validation error with a human-readable message.
2. **Hash + semver (`versioning/*.py`)** —
   - `compute_hash.py` wraps exercise-01's canonicaliser; given a suite directory, it produces the canonical `[{case_id, case_hash}, ...]` list, sorts by `case_id`, hashes it, and returns `"sha256:<hex>"`. Exercises the ordering, whitespace, and comment-insensitivity properties.
   - `semver.py` classifies a proposed change against the current suite state:
     - **patch** — added cases, added tags on existing cases, typo fix in `description` fields, added metadata on the suite.
     - **minor** — renamed tag, corrected `case_body.expected` on an existing case (case keeps its id, body changes, hash changes), retired a case with `superseded_by`.
     - **major** — bumped `case_body_schema_version` on the suite, removed a tag axis, metric family change recorded in the suite manifest.
   - `bump.py` applies the bump — writes the new `semver` into `MANIFEST.yaml`, computes the new `eval_set_hash`, inserts a new `eval_sets` row with the new hash and a `supersedes` pointer to the previous row, updates `eval_set_current(suite -> eval_set_id -> eval_set_hash)` in one transaction.
3. **CLI (`cli/eval_cases.py`)** — `eval-cases` with subcommands:
   - `eval-cases new --suite <id> --from-template <role>` — writes a new case YAML with required-field stubs.
   - `eval-cases validate --suite <id>` — runs the full validation chain locally (schema, tags, case-id uniqueness, case-hash uniqueness, suite-dry-run).
   - `eval-cases diff --base <ref>` — against a Git base ref, prints the classified bump (patch / minor / major) and the computed new `eval_set_hash`.
   - `eval-cases cost-preview --suite <id>` — the chapter 03 cost-preview arithmetic: `Δ cases × mean_judge_cost_usd[tier] × rubrics_count`.
   - `eval-cases retire --case <id> --reason <text> [--superseded-by <id>]` — soft-delete; mutates the YAML with `retired_at` and `superseded_by`.
   - `eval-cases promote-from-rot-check --from <rot_check_suite> --cases <list> --to <golden_suite>` — the chapter 03 quarterly promotion flow.
4. **CI validation (`ci/*`)** — a GitHub Actions (or GitLab CI) workflow triggered on PR that:
   - Runs `eval-cases validate` on every changed suite.
   - Rejects any case whose tags are not in the current `tag_catalog.yaml`.
   - Rejects duplicate `case_id` or `case_hash` across the suite.
   - Runs a `plan --dry-run` through exercise-01's `custom` runner adapter stub (no judge calls) to confirm the cases would at least parse.
   - Runs `eval-cases cost-preview` and leaves the delta on the PR as a comment.
   - Classifies the bump via `eval-cases diff` and asserts the PR's `MANIFEST.yaml` semver matches — a PR that renames a tag but still claims a patch bump is rejected.
   - Enforces the two-reviewer rule via branch protection (document the GitHub branch-protection config in `README.md`).
5. **Contribution API + UI (`api/*`)** — eval-set-as-database surface:
   - `POST /v1/suites/{suite}/cases` — submit a proposed case. Body is the same schema as the YAML. Returns `case_id` and `proposal_id`. Writes a row with `proposed=true`, `submitted_by`, `submitted_at`.
   - `GET /v1/suites/{suite}/proposals` — list open proposals.
   - `POST /v1/proposals/{id}/approve` — four-eyes rule: `approved_by` must differ from `submitted_by` (and must have the owner-team or eval-team role). On approval: the case joins the suite, the suite's `eval_set_hash` bumps per the same `versioning/bump.py` as the repo path, the UI shows the new version.
   - `POST /v1/proposals/{id}/reject` — `rejected_by`, `rejection_reason`.
   - A minimal HTML UI (one page: list proposals; one page: add-case form) that a non-engineer can use. The form includes a mandatory "why this case" field and renders the cost-preview arithmetic live as tags change.
   - The API refuses to run if `auth: local_dev` is set and the environment is not `dev`.
6. **Round-trip test (`tests/test_roundtrip.py`)** — the single most load-bearing test. For a case authored through the API, the eventual `eval_cases` row round-trips to the equivalent YAML on disk (via a nightly export job you also ship, `cli: eval-cases export-to-repo`). For a case authored through the YAML, the merge pipeline writes an identical `eval_cases` row. Both paths write to the same schema; a drift between surfaces fails this test.
7. **Suite subset roles** — the three subset roles from chapter 03 (`golden`, `regression`, `rot_check`) are first-class fields in `MANIFEST.yaml`. The CI validation enforces different rules per role:
   - **Golden** requires two reviewers, a `why` on every case, and a cost-preview on every PR.
   - **Regression** requires one reviewer, a `linked_incident` id on every case, and *monotonic* growth (CI rejects a PR that removes a case from the regression suite without retirement).
   - **Rot-check** requires one reviewer and allows bulk additions; is rebuilt monthly (a stretch goal below automates the monthly refresh).
8. **Retirement, not deletion** — the CLI's `retire` subcommand soft-deletes. A hard `DELETE FROM eval_cases` is rejected at the DB level (migration adds a trigger that raises). A compliance-mandated scrub replaces the case's `case_body.inputs` with a redaction marker while preserving the `case_id`, `case_hash` (recomputed on redacted body), and all audit columns.
9. **Vendor-UI drift audit** — a scheduled job (daily) that lists every runner's native dataset tagged with the `eval_set_hash` and asserts the vendor's row count matches `eval_cases` count. A mismatch emits an alert pointing at the sync code in exercise-03's adapter — the single most common cause is a case added in the vendor UI that never round-tripped back. Shippable even before exercise-03 wires real adapters — stub the adapters to always report "matched" for the exercise's purpose.
10. **Documentation (`README.md`)** — three audiences:
    - **Product-team contributor (non-engineer).** How to open the UI, how to add a case, what each form field means, how to read the cost preview, who the reviewer is for your suite.
    - **Product-team contributor (engineer).** How to open the PR, how to run `eval-cases` locally, what CI will check, what to do if the semver classifier flags your change.
    - **Eval-team operator.** How to add a new suite, how to run the quarterly rot-check-to-golden promotion, how to scrub a case for compliance, how to interpret the vendor-UI drift alert.
11. **Tests (`tests/*`)** — unit tests for the schema (round-trip valid cases, reject invalid ones), the hash (equivalent orderings produce the same hash; a comment-only edit does not), the semver classifier (hand-authored before/after pairs for each class), the CI validator (fixture suites that pass and fail per expectation), the API (submit → review → approve round-trip; four-eyes rule enforced; auth gated).

## Starter guidance

- **Author `case_id`s like filenames, not database keys.** `CASE-support-billing-refund-2026-03-12` is right; `CASE-000042` is wrong. The id lands in Git blame, in dashboards, in incident runbooks — meaningful ids pay back the shoe-leather for years. Chapter 03 makes this a rule, not a style preference.
- **Make the vendor-UI drift alert pessimistic.** If the runner adapter does not yet report its dataset state, assume drift and page. A false-negative here (a case added in Braintrust that never lands in `eval_cases`) is a real reproducibility break; a false-positive is a minute of eval-team time.
- **Do the round-trip test first.** It is the test that makes the "two surfaces, one schema" promise real. If you do not have the test, the two surfaces will diverge in month three and the schema will be the sad arbiter.
- **The four-eyes rule applies to the eval team too.** The eval-team lead cannot be submitter and reviewer of the same case. The CLI and the API both enforce this at the user level.
- **Semver is a label; the hash is trust.** The CI must reject a PR that labels a backward-incompatible change as a patch, because the humans downstream read the semver and plan against it. But the reproducibility check (exercise-01, exercise-04) joins on the hash, not the semver.
- **Promote from rot-check deliberately, quarterly.** The CLI promotes but does not schedule. The quarterly review is a human artefact (two leads meeting over the rot-check misses). Automate the promotion only after two or three quarterly cycles when the heuristic is clear.
- **Non-engineer "add a case" is a UX exercise.** The form's labels, the help text, the "why this case" prompt — these are what make the surface actually used in month six. Treat the UI as production-ready work; empty inputs with no help text produce zero contributions.
- **Retire by default on PII.** A customer-name-containing case is retired within one business day of discovery; the row stays. The compliance-scrub path exists for extraction orders; it is slower, loud, and audited.

## Acceptance criteria

You are done when:

- The three CLI invocations (`validate`, `diff`, `cost-preview`) run cleanly against a locally-cloned repo and the exercise-01 Postgres instance.
- A fresh PR that adds a well-formed case passes CI; the CI comment includes the correct `eval_set_hash`, the correct semver classification, and the correct cost-preview delta.
- A PR that violates any CI rule (unknown tag, duplicate id, mis-classified bump) is rejected with a specific error pointing at the offending file or field.
- A proposal submitted through the API by `alice@example.com` and approved by `bob@example.com` lands as a new `eval_cases` row with `approved_by=bob@example.com`; a self-approval by `alice` is rejected with a 403.
- The round-trip test passes — a case authored through the API round-trips to the repo YAML; a case authored through the YAML round-trips to the DB row; the two surfaces are symmetric.
- The retention test passes — `eval-cases retire` soft-deletes; a direct `DELETE FROM eval_cases` is refused by the DB trigger; a scrub replaces the body while preserving the id and the audit chain.
- The three subset-role rules are enforced — a golden PR without `why` is rejected; a regression PR that removes a case without retirement is rejected; a rot-check PR with 50 bulk-added cases passes.
- The vendor-UI drift audit runs on a schedule and alerts (or produces a green row) for every suite.
- The three audiences' sections in `README.md` are legible to a reader who has not read chapter 03 (test by handing it to a colleague who hasn't).
- Tests pass in CI.

## Stretch goals

- **Automatic monthly rot-check rebuild.** A scheduled job pulls the current month's mod-107 online-loop failures (filtered by cohort + rubric), proposes them as cases in `support_bot_rot_check_YYYY_MM`, auto-tags from the cohort keys, requires one reviewer. The suite's `eval_set_hash` bumps monthly without manual effort.
- **Tag-rename migration.** Script the mechanics — rename `feature:refund` to `feature:refunds` across the catalog and every case that references it, bump the catalog version, open a single PR with the full re-tag. Add a test fixture.
- **Bidirectional vendor sync.** Pick one runner from exercise-03 (Braintrust or Weave) and wire the "case added in the vendor UI → proposed contribution" path. The vendor-UI drift audit now auto-closes a drift by surfacing the new case as a proposal. Document the sync's known limits.
- **Contribution-rate dashboard.** A small Grafana (or `streamlit`) dashboard: cases added per team per week, median review latency, number of PRs rejected at CI vs merged, number of rot-check promotions this quarter. Chapter 03's "signals it is working" list becomes a dashboard.
- **Four-eyes bypass for emergencies.** A `--emergency` flag on `eval-cases` or an API override token (same shape as chapter 05's quota-override) that lets a sole reviewer merge a case under a documented incident number. Audited; the eval-team lead reviews bypasses weekly.
- **Semver-aware runner refusal.** Wire the exercise-03 adapters to refuse a run against an `eval_set_hash` whose suite's `semver.major` exceeds the adapter's declared `supported_major`. Confirms the major-bump contract from chapter 03.
- **Mod-108 safety-case payload split.** For safety-sensitive cases (jailbreak, injection), split the case body — the shape of the input + the metadata stay in `eval_cases`; the payload text lives in a sealed object store with the row carrying only the hash. Confirm the exercise-01 reproducibility check still passes.

## What this exercise does *not* cover

You are not building the runner-adapter plane (that is exercise-03), the platform SLIs that watch the test-case-age tile (exercise-04), or the CI gate the `eval_set_hash` is read by (exercise-05). You are shipping the contribution surface and the versioning machinery — the surface product teams touch, and the surface every downstream run joins back to.
