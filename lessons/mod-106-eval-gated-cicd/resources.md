# Resources for mod-106-eval-gated-cicd (Eval-Gated CI/CD for Prompts, Chains, and Agents)

Primary references cited across the chapters and exercises. Prefer these over blog posts — the CI runners and deploy tooling move fast, and this module relies on the official docs being read *from the URL* each release cycle.

## Eval-runner frameworks

- **Promptfoo — documentation.** <https://www.promptfoo.dev/docs/>. YAML config surface, assertion types, CLI, cache behaviour. The primary source for chapter 04 and exercise-03.
- **Promptfoo — GitHub Action.** <https://github.com/promptfoo/promptfoo-action>. Official action that runs Promptfoo in CI and posts results to the PR. Referenced from chapter 05 and exercise-01.
- **Promptfoo — repo.** <https://github.com/promptfoo/promptfoo>. OSS source; read the current assertion types and provider list here when the docs lag.
- **DeepEval — documentation.** <https://docs.confident-ai.com/>. Pytest hooks, metric classes, `LLMTestCase`, `assert_test`. The primary source for chapter 04 and exercise-04.
- **DeepEval — repo.** <https://github.com/confident-ai/deepeval>. OSS metric implementations; useful when writing a custom `BaseMetric` for a trajectory rubric.
- **Braintrust — documentation.** <https://www.braintrust.dev/docs>. `Eval` runner, datasets, scorers, comparison view. Chapter 04's vendor-SDK example.
- **Braintrust — CI integration.** <https://www.braintrust.dev/docs/guides/ci-cd>. PR-comment / GitHub check integration.
- **W&B Weave — documentation.** <https://weave-docs.wandb.ai/>. `weave.Evaluation`, `weave.op`, dataset refs. Chapter 04's second vendor-SDK example.
- **Langfuse — documentation.** <https://langfuse.com/docs>. Datasets, experiments, evaluators. Chapter 04's third vendor-SDK example.
- **LangSmith — documentation.** <https://docs.smith.langchain.com/>. LangChain / LangGraph-native evaluation and dataset primitives.
- **RAGAS — documentation.** <https://docs.ragas.io/>. Faithfulness, context precision / recall, answer relevancy. The reference implementations for the RAG-triad metrics you gate on in this module.

<!-- needs-research: re-verify the current PR-comment integration, self-host status, and RAG-triad scorer coverage for Braintrust / Weave / Langfuse / LangSmith on the next research cycle. The framework surface changes on a monthly cadence. -->

## GitHub Actions and CI plumbing

- **GitHub Actions — documentation.** <https://docs.github.com/en/actions>. Workflow syntax, triggers, permissions, contexts, artefacts.
- **GitHub Actions — workflow syntax reference.** <https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions>. Authoritative reference for `on:`, `concurrency:`, `permissions:`, `jobs:`.
- **GitHub Actions — security hardening (`pull_request_target`).** <https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions>. The fork-PR safety pattern from chapter 05 and exercise-01.
- **GitHub Actions — caching.** <https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows>. `actions/cache` behaviour and key scoping.
- **GitHub Actions — artefacts.** <https://docs.github.com/en/actions/using-workflows/storing-workflow-data-as-artifacts>. Upload / download / retention.
- **GitHub — branch protection rules.** <https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/defining-the-mergeability-of-pull-requests/about-protected-branches>. How to make an eval-gate check `required`.
- **GitHub — CODEOWNERS.** <https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-configuration/customizing-your-repository/about-code-owners>. Chapter 03's enforcement mechanism for threshold, rubric, runbook, and fixture reviews.
- **`marocchino/sticky-pull-request-comment`.** <https://github.com/marocchino/sticky-pull-request-comment>. The sticky-comment action referenced in chapter 05 and exercise-01.
- **`EnricoMi/publish-unit-test-result-action`.** <https://github.com/EnricoMi/publish-unit-test-result-action>. JUnit-XML rendering on PR checks; used in exercise-04.
- **GitLab CI — documentation.** <https://docs.gitlab.com/ee/ci/>. Equivalent primitives for the CI-portability discussion in chapter 05.
- **Buildkite — documentation.** <https://buildkite.com/docs>. Second CI-portability reference.
- **CircleCI — documentation.** <https://circleci.com/docs/>. Third CI-portability reference.

## Deploy pipelines, canary, and progressive rollout

- **Argo Rollouts — documentation.** <https://argoproj.github.io/argo-rollouts/>. Blue/green, canary, and progressive-rollout primitives on Kubernetes. Chapter 06's default reference.
- **Flagger — documentation.** <https://flagger.app/>. GitOps-based progressive-delivery operator; canary analysis with metrics gates.
- **Spinnaker — documentation.** <https://spinnaker.io/docs/>. Long-standing deploy tool with canary-analysis primitives.
- **Kubernetes — Deployment strategies.** <https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy>. Rolling-update and recreate strategies underlying most canary systems.
- **OpenFeature — specification.** <https://openfeature.dev/specification/>. Vendor-neutral feature-flag spec referenced in chapter 06 for the "flag as canary / rollback knob" pattern.
- **LaunchDarkly — progressive delivery.** <https://docs.launchdarkly.com/home/experimentation/progressive-delivery>. Vendor implementation of the flag-driven canary pattern.
- **Statsig — feature flags.** <https://docs.statsig.com/feature-flags/overview>. Second flag-vendor reference.
- **Google SRE Book — chapter 27, Reliable Product Launches at Scale.** <https://sre.google/sre-book/reliable-product-launches/>. Canary and staged-rollout mental model. Read alongside chapter 06.
- **Google SRE Workbook — canarying releases.** <https://sre.google/workbook/canarying-releases/>. The canonical treatment of canary analysis and rollback criteria.

## Model API pinning and reproducibility

- **OpenAI — Chat Completions API reference.** <https://platform.openai.com/docs/api-reference/chat/create>. `seed`, `system_fingerprint`, model snapshot suffixes; primary source for chapter 02's pinning discussion.
- **OpenAI — model versioning.** <https://platform.openai.com/docs/models>. Snapshot naming convention (`gpt-4o-<YYYY-MM-DD>`); note the alias-vs-snapshot distinction.
- **Anthropic — API reference.** <https://docs.anthropic.com/en/api/getting-started>. Snapshot naming (`claude-3-5-sonnet-<YYYYMMDD>`), stream shape, seed semantics.
- **Anthropic — model overview and versioning.** <https://docs.anthropic.com/en/docs/about-claude/models>. The snapshot-vs-alias convention referenced in chapter 02.
- **AWS Bedrock — model ARN / snapshot pinning.** <https://docs.aws.amazon.com/bedrock/latest/userguide/model-parameters.html>. Snapshot pinning in the ARN, region-scoped model availability.
- **vLLM — documentation.** <https://docs.vllm.ai/>. Self-hosted-inference reproducibility: seed, KV cache, tokenizer version, batching effects.
- **TensorRT-LLM — documentation.** <https://nvidia.github.io/TensorRT-LLM/>. Alternative self-hosted-inference stack; kernel-level determinism trade-offs.

## Trace backends and cross-references (deeper coverage in mod-102)

- **Arize Phoenix.** <https://docs.arize.com/phoenix>. Trace backend the fixture pipeline reads in exercise-02.
- **Langfuse (as a trace backend).** <https://langfuse.com/docs/tracing>. Second option for the fixture-snapshot source.
- **W&B Weave (as a trace backend).** <https://weave-docs.wandb.ai/quickstart>. Third option.
- **Braintrust (as a trace backend).** <https://www.braintrust.dev/docs/guides/tracing>. Fourth option.
- **LangSmith (as a trace backend).** <https://docs.smith.langchain.com/observability>. Fifth option.

## PII scrubbing in the fixture pipeline

- **Microsoft Presidio.** <https://microsoft.github.io/presidio/>. Named-entity PII detection + anonymisation. Chapter 02's default text scrubber for the fixture pipeline.
- **NIST SP 800-122 — Guide to Protecting the Confidentiality of Personally Identifiable Information.** <https://csrc.nist.gov/publications/detail/sp/800-122/final>. PII definition. Referenced from mod-102 chapter 06 and reused here.
- **GDPR — Regulation (EU) 2016/679.** <https://eur-lex.europa.eu/eli/reg/2016/679/oj>. Art. 4(1) personal data; Art. 4(5) pseudonymisation. Cited when a fixture leaks user-derived content.

## Governance and program-owner cross-references (deeper coverage in mod-112)

- **NIST AI Risk Management Framework (AI 100-1, 2023).** <https://www.nist.gov/itl/ai-risk-management-framework>. The Measure and Manage functions cover release-gate posture; chapter 03's threshold / severity / owner is a Measure artefact, and chapter 06's rollback contract is a Manage artefact.
- **NIST AI RMF Generative AI Profile (AI 600-1, 2024).** <https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile>. Suggested actions for generative AI; the runbook + rollback contract maps to the "Manage" function.
- **OWASP Top 10 for Large Language Model Applications.** <https://owasp.org/www-project-top-10-for-large-language-model-applications/>. Cross-reference for the safety-gate variant of this module's pattern; deep coverage in mod-108.
- **ISO/IEC 42001:2023 — AI management systems.** <https://www.iso.org/standard/81230.html>. Release-gate architecture and rollback rehearsals are 42001-relevant artefacts. Paywalled.
- **EU Artificial Intelligence Act (Regulation (EU) 2024/1689).** <https://eur-lex.europa.eu/eli/reg/2024/1689/oj>. Change-management and monitoring obligations for high-risk systems; the deploy-gate contract is one of the artefacts a high-risk deployment would produce.

## Software-engineering practice references

- **Google Testing Blog — snapshot testing anti-patterns.** <https://testing.googleblog.com/>. The "insta-approve" pattern chapter 02 warns about is documented across the SE community; the search index above is the entry point.
- **Semantic Versioning 2.0.0.** <https://semver.org/>. Referenced for versioning thresholds, rubrics, and gate configs; chapters 03 and 04.
- **Twelve-Factor App.** <https://12factor.net/>. Config-in-environment principles referenced in chapter 05 (secrets and env-injected knobs) and chapter 06 (deploy-gate script inputs).

## Peer curriculum interfaces

- **`agentic-ai-engineer-learning`** — <https://github.com/ai-engineering-curriculum/agentic-ai-engineer-learning> — the agent implementations whose PRs go through this module's gate.
- **`llm-application-developer-learning`** — <https://github.com/ai-engineering-curriculum/llm-application-developer-learning> — the LLM app engineering whose prompt / chain / tool changes flow through this module's gate.
- **`rag-engineer-learning`** — <https://github.com/ml-engineering-curriculum/rag-engineer-learning> — the retriever whose index-rebuild is one of the events that must flow through this module's fixture pipeline and threshold system.
- **`ai-evaluation-engineer-learning` (Governance family, level 25)** — <https://github.com/ai-governance-curriculum/ai-evaluation-engineer-learning> — the sign-off owner for the release-gate architecture; mod-112 formalises the hand-off.
- **`model-evaluation-engineer-learning` (Model-Development family, level 30)** — the peer that owns full-methodology threshold-derivation and statistical calibration; referenced in chapter 03 for threshold-numeric selection.

<!-- needs-research: on the next research cycle, re-verify current Promptfoo / DeepEval CLI flags and cache behaviour, re-check the Braintrust / Weave / Langfuse CI-integration and PR-comment surfaces, re-verify the OpenAI `system_fingerprint` and `seed` API-reference position, and refresh the Argo Rollouts / Flagger canary-analysis documentation URLs. -->
