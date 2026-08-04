# mod-106 — Eval-Gated CI/CD for Prompts, Chains, and Agents

**Estimated effort:** 14 hours

This is the sixth module of the AI Evaluation Engineer track. It turns the mod-101 criteria table, the mod-102 trace shape, the mod-103 trajectory scores, the mod-104 judge, and the mod-105 RAG-triad into an **eval gate** — a mechanism that runs on every pull request and every deploy stage, blocks progression on regression, and hands the deploy pipeline a decision the platform team can act on. Without a gate, the eval work of mods 101–105 is a dashboard nobody reads; with a gate, it is a first-class quality bar the team already respects at the same altitude as type checks and unit tests.

The module scopes to the *CI/CD* side of the eval program — writing thresholds down, snapshotting production traces into regression fixtures, wiring the declarative eval runners (Promptfoo, DeepEval, and the Braintrust / Weave / Langfuse / LangSmith CI runners) into GitHub Actions, and defining the interface to the deploy pipeline the platform team owns. Full online-eval and drift monitoring lives one module downstream (mod-107); this module is where the *offline* gate becomes real.

## Learning objectives

- Wire evaluations into pull-request CI so a prompt / chain / agent change gets an eval report before merge.
- Snapshot and replay production traces as regression fixtures with pinned model version, seed, and retrieval index hash.
- Register pre-registered pass/fail thresholds and treat regressions as first-class blockers with an owner-assigned runbook.
- Build declarative eval configs with Promptfoo (YAML), DeepEval pytest hooks, and Braintrust / Weave / Langfuse / LangSmith CI runners.
- Design deployment gates (offline-eval pass → canary → progressive rollout) and the interface to the deploy pipeline the platform team owns.

## Lecture chapters

1. [Why eval-gated CI/CD](01-why-eval-gated-cicd.md) — the definition of an eval gate; three gate points (pre-commit, PR, deploy); the five properties (reproducible, pre-registered, declarative, runnable-from-CI, wired-into-deploy); the vocabulary and the end-to-end mental picture.
2. [Trace snapshots as regression fixtures](02-trace-snapshot-regression-fixtures.md) — fixture anatomy; the four pins (model + snapshot, seed / decoding params, retrieval index hash, tool stubs); selection strategies; fixture lifecycle (add / refresh / retire / quarantine); the "insta-approve" trap.
3. [Pre-registered thresholds, blockers, and runbooks](03-thresholds-blockers-runbooks.md) — the four threshold shapes; pre-registration via a code-reviewed file; `warn` vs `block` and the time-boxed promote-or-revise rule; the runbook template; CODEOWNERS as the enforcement contract.
4. [Declarative eval configs: Promptfoo, DeepEval, and framework CI runners](04-declarative-eval-configs.md) — the four runner shapes; Promptfoo YAML; DeepEval pytest; the vendor SDK runners (Braintrust / Weave / Langfuse / LangSmith); portable fixture-and-threshold loading; the bake-off criteria.
5. [Wiring the PR gate: GitHub Actions, the eval report, and the check](05-pr-gate-in-github-actions.md) — the minimum viable workflow; the PR-comment report format; secrets, budget, and caching; the baseline-writer job on `push`; the developer inner loop; CI-system portability.
6. [Deployment gates and progressive rollout](06-deployment-gates-and-rollout.md) — the three deploy-gate stages (offline pass → canary → progressive rollout); shadow vs live-slice eval; the interface to the platform team; rollback shapes; feature flags; the hand-off to the mod-107 online loop.

## Exercises

Each exercise builds on the last. Do them in order and keep the outputs — the artefacts (PR-gate workflow, fixture pipeline, Promptfoo YAML, DeepEval pytest suite, deploy-gate runbook) are the deliverables mod-107 and mod-112 will assume you have.

1. [PR eval report in GitHub Actions](exercises/exercise-01-pr-eval-report-in-github-actions.md) — stand up the PR gate: workflow, report, sticky comment, required check.
2. [Trace snapshot as regression fixture](exercises/exercise-02-trace-snapshot-as-regression-fixture.md) — snapshot three real traces from your mod-102 backend into the fixture schema; assert the four pins; add a fixture from a fake incident.
3. [Declarative eval YAML with Promptfoo](exercises/exercise-03-declarative-eval-yaml-with-promptfoo.md) — author the Promptfoo config against your fixture set; wire rubric + cost + latency assertions; produce the PR-comment report shape.
4. [DeepEval pytest hooks in CI](exercises/exercise-04-deepeval-pytest-hooks-in-ci.md) — port one metric to DeepEval as a pytest gate; run both runners in parallel; write the compare-two-runners note.
5. [Deployment gate and runbook](exercises/exercise-05-deployment-gate-and-runbook.md) — write the offline-artefact gate script, the canary-gate script that reads the trace backend, the platform-team interface JSON schema, and the on-call runbook.

## Labs and quizzes

- Labs (see [`labs/`](labs)) build the end-to-end eval-gated pipeline reference: an instrumented service (from mod-102), a replay set (from this module's chapter 02), a full PR + deploy gate (chapters 05, 06), and a smoke rollback rehearsal. Authored under the autonomous fill-in loop.
- Quizzes (see [`quizzes/`](quizzes)) verify the vocabulary — the five gate properties, the four pins, the threshold shapes, the runbook contract, the canary vs shadow distinction. Authored under the autonomous fill-in loop.

## Resources

External references are curated in [`resources.md`](resources.md).

## Where this module hands off

- **Online-eval and regression** consumes the deploy-gate interface (chapter 06) as its substrate; the sampled online judge feeds the canary-stage gate → [`mod-107-online-eval-and-regression`](../mod-107-online-eval-and-regression).
- **Safety and guardrail eval** uses the same gate framework with stricter, blocking-only severity for safety rubrics → [`mod-108-app-safety-and-guardrails-eval`](../mod-108-app-safety-and-guardrails-eval).
- **Human review workflows** consume gate-fired regressions as review-queue items when the on-call cannot resolve them from the runbook → [`mod-109-human-review-workflows`](../mod-109-human-review-workflows).
- **Eval-data platform** stores fixtures, rubric-hashed judge outputs, and gate results as first-class tables → [`mod-110-eval-data-platform-slice`](../mod-110-eval-data-platform-slice).
- **Cost / latency / quality trade-off** reads the per-fixture and per-canary cost + latency numbers that chapter 06's gate asserts against → [`mod-111-cost-latency-quality-tradeoff`](../mod-111-cost-latency-quality-tradeoff).
- **Program-owner posture** treats this module's thresholds, runbooks, and rollback contract as the load-bearing artefacts of the release-gate architecture and defends them to the Governance-family peer → [`mod-112-owning-an-ai-eval-program`](../mod-112-owning-an-ai-eval-program).
