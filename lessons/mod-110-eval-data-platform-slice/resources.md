# Resources for mod-110-eval-data-platform-slice (Eval-Data-Platform Slice for AI Applications)

Primary references cited across the chapters and exercises. Prefer the official docs and the original papers over blog posts — the eval-runner SDKs, data-catalogue surfaces, and A/B / cost-governance stacks move on a monthly cadence and the summaries drift. Read from the URL each time you plan to build against a specific behaviour.

## Eval runners the plane wraps

Each is a target of the chapter 04 `RunnerAdapter` contract. The exercise-03 PR wires three of these; a mature slice usually ends up with four to seven.

- **Promptfoo — documentation.** <https://www.promptfoo.dev/docs/intro/>. YAML-driven eval runner; CLI-first (`promptfoo eval`); strong on prompt-comparison matrices and `llm-rubric` assertions.
- **Promptfoo — assertions reference.** <https://www.promptfoo.dev/docs/configuration/expected-outputs/>. The native assertion vocabulary the adapter translates from.
- **DeepEval — documentation.** <https://docs.confident-ai.com/>. Python-native regression-suite runner; `LLMTestCase` + metric classes; `pytest`-shaped integration.
- **DeepEval — repo.** <https://github.com/confident-ai/deepeval>. OSS source; useful when reading the exact `BaseMetric` contract for custom metrics.
- **RAGAS — documentation.** <https://docs.ragas.io/>. RAG-specialised metric library; `faithfulness`, `answer_relevancy`, `context_precision`, `context_recall`, `answer_similarity`.
- **RAGAS — repo.** <https://github.com/explodinggradients/ragas>. OSS source.
- **Braintrust — documentation.** <https://www.braintrust.dev/docs>. Hosted eval + prompt-management platform; `Eval` + `Experiment` + `Dataset` SDK surfaces.
- **W&B Weave — documentation.** <https://weave-docs.wandb.ai/>. Weights & Biases's LLM evaluation surface; `weave.Evaluation`, `weave.Scorer`, `weave.Model`.
- **Langfuse — documentation.** <https://langfuse.com/docs>. OSS trace + evaluator platform (also hosted); SDK scores, UI-configurable evaluators, datasets.
- **Arize Phoenix — documentation.** <https://docs.arize.com/phoenix>. OSS trace backend + evaluator surface; cross-reference from mod-102 and mod-107.
- **TruLens — documentation.** <https://www.trulens.org/>. Alternate OSS eval library; `TruChain` + feedback-function pattern; viable fifth adapter target.
- **UK AISI Inspect — documentation.** <https://inspect.aisi.org.uk/>. Model-eval harness increasingly used for agent and safety evals; cross-reference from mod-103 and mod-108.
- **OpenAI Evals — repo.** <https://github.com/openai/evals>. Original OSS eval framework; chapter 04 "custom evaluator" shape is often a trimmed Evals-style harness.

<!-- needs-research: on the next research cycle, re-verify the Braintrust, Weave, and Langfuse Python SDK entry points named in chapter 04 — the hosted SDKs ship frequently and the class names (`Eval` vs `init_experiment` vs `Experiment`; `Evaluation.evaluate` vs `Evaluation.run`; `langfuse.score` vs `langfuse.create_score`) shift between minor versions. -->

## Trace standards and interchange

- **OpenTelemetry — specification.** <https://opentelemetry.io/docs/specs/otel/>. The underlying spec for the chapter 02 `traces` / `spans` table shape.
- **OpenTelemetry — GenAI semantic conventions.** <https://opentelemetry.io/docs/specs/semconv/gen-ai/>. The canonical attribute vocabulary for LLM spans; chapter 02's `spans` table joins against these attributes.
- **OpenInference — semantic conventions.** <https://github.com/Arize-ai/openinference/tree/main/spec>. Alternate GenAI span conventions widely used by vendor runtimes.
- **OpenLineage — specification.** <https://openlineage.io/docs/spec/object-model/>. The run / job / dataset model chapter 02 uses as the inter-platform interchange.
- **OpenLineage — repo.** <https://github.com/OpenLineage/OpenLineage>. OSS source; the Python client used in the exercise-01 `openlineage/emit.py` facade.
- **Marquez — documentation.** <https://marquezproject.github.io/marquez/>. Reference OpenLineage server; chapter 02's "stub endpoint" the exercise emits against.
- **DataHub — documentation.** <https://datahubproject.io/docs/>. OSS metadata platform; one of the catalogues chapter 02's OpenLineage emits are consumed by.
- **Amundsen — documentation.** <https://www.amundsen.io/amundsen/>. Alternate OSS catalogue; worth knowing when the org already runs it.

## Content-addressed storage and canonical hashing

Chapter 02's lineage key set is built on content hashes. The references below define the canonical-serialisation discipline the exercise-01 canonicalisers depend on.

- **IPFS — content-addressed storage overview.** <https://docs.ipfs.tech/concepts/content-addressing/>. The pattern of `sha256:<hex>` identifiers over canonical bytes is well-established in IPFS and OCI.
- **Git — object storage model.** <https://git-scm.com/book/en/v2/Git-Internals-Git-Objects>. The reference content-addressed store every engineer already knows; chapter 02's canonicalisation discipline mirrors Git's.
- **OCI image specification — content digests.** <https://github.com/opencontainers/image-spec/blob/main/descriptor.md#digests>. The algorithm-prefixed digest format (`sha256:...`) chapter 02 adopts.
- **JSON Canonicalization Scheme (JCS, RFC 8785).** <https://www.rfc-editor.org/rfc/rfc8785>. The IETF spec for canonical JSON; chapter 02's prompt / chain / rubric / eval-set canonicalisation follow this discipline.
- **Multihash — specification.** <https://multiformats.io/multihash/>. The algorithm-prefixed hash encoding the exercise-01 `hashing.py` facade uses so new algorithms can be added without a schema migration.
- **BLAKE3 — specification and reference.** <https://github.com/BLAKE3-team/BLAKE3>. The modern alternative to SHA-256; chapter 02's `multihash` facade is what makes a future `blake3:` prefix a one-line change.

## Database and warehouse platforms

The chapter 02 "hot row store + warm columnar tier" pattern. The references cover the concrete targets exercise-01 ships against.

- **PostgreSQL — documentation.** <https://www.postgresql.org/docs/current/>. The reference hot store for the exercise.
- **Alembic — documentation.** <https://alembic.sqlalchemy.org/>. The migration tool exercise-01's `migrations/` directory assumes if you prefer SQLAlchemy; `0001_initial_tables.sql` is equivalently expressible as an Alembic revision.
- **Flyway — documentation.** <https://documentation.red-gate.com/fd>. Alternate migration tool; same shape.
- **sqitch — documentation.** <https://sqitch.org/docs/>. Third option, Git-native migration management.
- **DuckDB — documentation.** <https://duckdb.org/docs/>. The default warm tier for the exercise; a single file, zero config.
- **ClickHouse — documentation.** <https://clickhouse.com/docs>. Alternate warm tier when the org already runs it.
- **BigQuery — documentation.** <https://cloud.google.com/bigquery/docs>. The GCP warm tier.
- **Snowflake — documentation.** <https://docs.snowflake.com/>. Alternate warm tier.
- **Apache Iceberg — documentation.** <https://iceberg.apache.org/docs/latest/>. The open table format the warm tier can be stored against for portability across engines.
- **Delta Lake — documentation.** <https://docs.delta.io/latest/index.html>. Alternate open table format.

## Site Reliability Engineering: SLOs, error budgets, burn alerts

Chapter 06's whole operability frame is Google SRE applied to the eval plane.

- **Google SRE Book — Service Level Objectives (chapter 4).** <https://sre.google/sre-book/service-level-objectives/>. The canonical definition of SLI / SLO / SLA; chapter 06's vocabulary derives from here.
- **Google SRE Workbook — Implementing SLOs.** <https://sre.google/workbook/implementing-slos/>. The chapter the exercise-04 burn-rate math reads from — SLI definition, error budget shape, multi-window burn-rate alerts.
- **Google SRE Workbook — Alerting on SLOs.** <https://sre.google/workbook/alerting-on-slos/>. The fast / medium / slow burn-rate thresholds chapter 06 names (2 % / 1 h, 5 % / 6 h, 10 % / 3 d).
- **Google SRE Book — Managing Incidents (chapter 14).** <https://sre.google/sre-book/managing-incidents/>. The incident-command and postmortem culture chapter 06's runbook discipline derives from.
- **Google SRE Workbook — Postmortem Culture.** <https://sre.google/workbook/postmortem-culture/>. The chapter 06 "every incident produces a postmortem" rule.
- **OpenSLO — specification.** <https://openslo.com/docs/overview>. OSS spec for declaring SLOs as code; the exercise-04 `slos/*.yaml` files follow this shape where practical.
- **Sloth — documentation.** <https://sloth.dev/>. OSS SLO-generator; turns an OpenSLO spec into Prometheus recording and alerting rules.
- **Pyrra — documentation.** <https://github.com/pyrra-dev/pyrra>. Alternate OSS SLO-generator for Prometheus / Kubernetes.
- **Nobl9 — documentation.** <https://docs.nobl9.com/>. Hosted SLO platform; useful if the org has adopted a dedicated SLO vendor.

## Queueing, scheduling, and rate limits

Chapter 05's weighted-fair scheduler + envelope.

- **Token-bucket algorithm — overview.** <https://en.wikipedia.org/wiki/Token_bucket>. The algorithm chapter 05's per-class quota implements.
- **Leaky-bucket algorithm — overview.** <https://en.wikipedia.org/wiki/Leaky_bucket>. The rate-limiting peer concept the envelope is a flavour of.
- **AWS Architecture Blog — "Exponential Backoff and Jitter".** <https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/>. The reference for chapter 05's exponential-backoff-with-full-jitter retry policy.
- **Google SRE Book — Addressing Cascading Failures (chapter 22).** <https://sre.google/sre-book/addressing-cascading-failures/>. The retry-storm failure mode chapter 05 defends against.
- **Weighted Fair Queueing — original paper.** Demers, A., Keshav, S., Shenker, S. "Analysis and simulation of a fair queueing algorithm." *SIGCOMM* 1989. <https://dl.acm.org/doi/10.1145/75246.75248>. The algorithm chapter 05's three-class scheduler is a per-class instance of.
- **Deficit Round Robin — original paper.** Shreedhar, M., Varghese, G. "Efficient fair queueing using deficit round robin." *IEEE/ACM Transactions on Networking* 1996. <https://dl.acm.org/doi/10.1145/231699.231709>. A simpler weighted-fair variant; often the first thing you implement before upgrading.
- **Redis — rate-limiting patterns.** <https://redis.io/docs/manual/patterns/distributed-locks/>. Reference distributed-token-bucket implementation. (Scroll for the sliding-window-counter pattern; chapter 05's envelope is a close relative.)

## Idempotency and distributed-request contracts

Chapter 05's `X-Idempotency` header contract.

- **Stripe — Idempotency keys.** <https://stripe.com/docs/api/idempotent_requests>. The industry-canonical reference for idempotency-key semantics on an HTTP API; chapter 05's TTL + replay behaviour is this pattern.
- **AWS — "Idempotency with AWS Lambda and the AWS SDKs."** <https://docs.aws.amazon.com/lambda/latest/dg/invocation-idempotency.html>. Alternative treatment, useful when the plane runs on Lambda.
- **Nguyen — "Designing an Idempotent API."** <https://brandur.org/idempotency-keys>. Practitioner reference for request-key TTLs, in-flight vs terminal states, and the race window chapter 05 covers.

## FinOps and cost governance

Chapter 05's cost-centre / quota / invoice-reconciliation + chapter 06's SLI #3.

- **FinOps Foundation — Framework.** <https://www.finops.org/framework/>. The canonical FinOps vocabulary (chargeback, showback, cost centres); chapter 05's `cost_centre` roll-up follows this discipline.
- **FinOps Foundation — AI / ML specific guidance.** <https://www.finops.org/wg/aiml/>. Working-group writeups on FinOps for generative-AI workloads; cross-reference for the pricing-snapshot problem.
- **OpenCost — specification.** <https://www.opencost.io/>. OSS cost-attribution reference; mostly Kubernetes-focused but the attribution-key patterns map to chapter 05's cost-centre.
- **Google Cloud — "FinOps for AI."** <https://cloud.google.com/architecture/finops-for-ai>. Vendor writeup of the generative-AI cost-governance pattern.
- **Anthropic — pricing.** <https://www.anthropic.com/pricing>. The upstream price list the plane's `pricing_snapshots` tracks for Claude judges.
- **OpenAI — pricing.** <https://openai.com/api/pricing/>. Same, for OpenAI judges.
- **AWS Bedrock — pricing.** <https://aws.amazon.com/bedrock/pricing/>. For Bedrock-mediated judges.
- **Google Vertex AI — pricing.** <https://cloud.google.com/vertex-ai/generative-ai/pricing>. For Vertex-mediated judges.

<!-- needs-research: on the next research cycle, confirm the vendor pricing URLs resolve to the current price pages and refresh the per-model unit costs the exercise-04 and exercise-05 `pricing_snapshots` seed data references. -->

## Software supply chain and change-management

Chapter 03's test-case-contribution flow and chapter 05's artefact-store discipline map onto software-supply-chain practices.

- **SemVer — specification.** <https://semver.org/>. The patch / minor / major rules chapter 03's eval-set versioning uses verbatim.
- **Conventional Commits — specification.** <https://www.conventionalcommits.org/en/v1.0.0/>. Commit-message convention useful when wiring the exercise-02 CI classifier to its PR check.
- **SLSA — supply-chain integrity framework.** <https://slsa.dev/>. The build-provenance discipline chapter 03's `supersedes` + `case_hash` provenance mirrors at a smaller scale.
- **in-toto — attestation framework.** <https://in-toto.io/>. The attestation shape the exercise-02 "proposal → approval → merge" audit chain follows.
- **GitHub — branch protection.** <https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches>. The two-reviewer enforcement chapter 03 relies on.

## Metrics, dashboarding, and alerting

Chapter 06's platform dashboard.

- **Prometheus — documentation.** <https://prometheus.io/docs/introduction/overview/>. The reference metrics backend for the exercise-04 SLIs.
- **Prometheus — recording and alerting rules.** <https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/>. The exercise-04 `alerting/rules.yaml` is a Prometheus-shaped rule file.
- **Grafana — documentation.** <https://grafana.com/docs/grafana/latest/>. The reference dashboarding surface for the exercise-04 four-tile dashboard.
- **Grafana — SLO plugin.** <https://grafana.com/grafana/plugins/grafana-slo-app/>. The SLO-dashboard plugin that renders error budgets natively; useful for a less hand-rolled exercise-04 dashboard.
- **PagerDuty — Response Plays.** <https://developer.pagerduty.com/docs/response-plays>. The alert-routing surface the exercise-04 `routing.yaml` targets.
- **OpenTelemetry Collector — documentation.** <https://opentelemetry.io/docs/collector/>. The dispatch target for both the plane's own traces and the metrics the SLI tiles read.

## CI integration and gating

Chapter 05's PR-gate / deploy-gate integration + exercise-05's `gate_result.json`.

- **GitHub Actions — documentation.** <https://docs.github.com/en/actions>. The CI the exercise-05 workflow targets.
- **GitHub Actions — status checks and required reviews.** <https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/collaborating-on-repositories-with-code-quality-features/about-status-checks>. The merge-blocking surface the gate reads against.
- **GitLab CI — documentation.** <https://docs.gitlab.com/ee/ci/>. Alternate CI surface.
- **Buildkite — documentation.** <https://buildkite.com/docs>. Third option; useful when the org already runs it.
- **Argo Rollouts — documentation.** <https://argoproj.github.io/argo-rollouts/>. Progressive-delivery operator; the deploy-gate stretch in exercise-05 wires against `AnalysisTemplate`.
- **Flagger — documentation.** <https://flagger.app/>. Alternate progressive-delivery operator.

## Trace backends (cross-reference, pointer-ownership for `traces`/`spans`)

Chapter 02 lets the vendor's native trace store be the authoritative home for the `traces` / `spans` tables; a pointer row per trace lets the plane query the vendor at read time.

- **Arize Phoenix — traces.** <https://docs.arize.com/phoenix/tracing>. The hosted trace backend the pointer rows reference when the org picks Phoenix.
- **Langfuse — traces.** <https://langfuse.com/docs/tracing>. Same, for Langfuse.
- **Braintrust — tracing.** <https://www.braintrust.dev/docs/guides/traces>. Same, for Braintrust.
- **W&B Weave — tracing.** <https://weave-docs.wandb.ai/guides/tracking/tracing>. Same, for Weave.
- **LangSmith — observability.** <https://docs.smith.langchain.com/observability>. LangChain-native backend; cross-reference from mod-107.

## Model-vendor snapshot discipline (cross-reference from chapter 02 lineage keys)

Chapter 02's `model_snapshot` and `judge_model_snapshot` are the single most common cause of silent score drift.

- **OpenAI — model versioning.** <https://platform.openai.com/docs/models>. Snapshot suffix (`gpt-4o-<YYYY-MM-DD>`) and alias-vs-snapshot distinction.
- **Anthropic — model overview and versioning.** <https://docs.anthropic.com/en/docs/about-claude/models>. Snapshot naming (`claude-3-5-sonnet-<YYYYMMDD>`).
- **AWS Bedrock — model versioning.** <https://docs.aws.amazon.com/bedrock/latest/userguide/model-lifecycle.html>. Snapshot pinning in the model ARN.
- **Google Vertex — model versions.** <https://cloud.google.com/vertex-ai/generative-ai/docs/learn/models>. Vertex AI model-version discipline.

## Governance and program-owner cross-references (deeper coverage in mod-112)

The platform-slice artefacts are evidence under multiple governance frames.

- **NIST AI Risk Management Framework (AI 100-1, 2023).** <https://www.nist.gov/itl/ai-risk-management-framework>. The Measure and Manage functions cover the chapter 06 SLIs (Measure) and chapter 05 cost / quota / artefact store (Manage).
- **NIST AI RMF Generative AI Profile (AI 600-1, 2024).** <https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile>. Continuous-monitoring suggested actions map directly to the mod-107 integration and the chapter 06 reproducibility check.
- **ISO/IEC 42001:2023 — AI management systems.** <https://www.iso.org/standard/81230.html>. The AI management system standard; chapter 05's audit trails (quota overrides, pricing-snapshot history, reconciliation) are 42001-relevant. Paywalled.
- **EU Artificial Intelligence Act (Regulation (EU) 2024/1689).** <https://eur-lex.europa.eu/eli/reg/2024/1689/oj>. Post-market monitoring obligations for high-risk systems (Articles 72 – 73); the chapter 05 online-loop integration is one of the artefacts a high-risk deployment would produce.
- **FINOS Open Source Readiness.** <https://community.finos.org/docs/governance/open-source-readiness>. Supply-chain governance; cross-reference for the artefact-store discipline when the eval platform spans multiple business units.

## Peer curriculum interfaces

The chapters reference these roles explicitly; chapter 07 draws the delegation boundary.

- **`model-evaluation-engineer-learning` (Model-Development family, level 30).** The peer that owns the cross-modality, multi-tenant eval-as-a-service surface. Chapter 07 is the delegation contract.
- **`ai-evaluation-engineer-learning` (Governance family, level 25).** The governance peer that signs off on release gates and continuous-monitoring architecture; cross-reference from mod-112.
- **`llm-application-developer-learning`.** The product-team peer whose prompts and chains feed the plane via the `prompt_hash` and `chain_hash` lineage keys.
- **`agentic-ai-engineer-learning`.** The agent-authoring peer whose traces feed the chapter 02 `traces` / `spans` tables; cross-reference from mod-103.
- **`senior-ml-engineer-learning` (Model-Development family).** The product-team peer that co-owns ship / iterate / kill decisions downstream of the chapter 05 gate.

<!-- needs-research: on the next research cycle, re-verify Promptfoo / DeepEval / RAGAS / Braintrust / Weave / Langfuse SDK URLs; confirm the OpenLineage spec and Marquez quickstart URLs; refresh the vendor pricing links; confirm the Google SRE Workbook burn-rate chapter is still at the cited URL (Google occasionally reorganises the workbook). -->
