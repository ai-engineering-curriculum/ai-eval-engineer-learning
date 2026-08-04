# Resources for mod-102-trace-instrumentation (Trace-Level Instrumentation for LLM Apps and Agents)

Primary references cited across the chapters and exercises. Prefer these to blog posts — the specs and repos move faster than the write-ups about them, and this track relies on the specs being read *from the URL* each release cycle.

## Distributed-tracing fundamentals

- **W3C Trace Context.** <https://www.w3.org/TR/trace-context/>. `traceparent` / `tracestate` header specification — the propagation contract that keeps a cross-service LLM call in one trace. Referenced in chapters 01 and 05.
- **OpenTelemetry overview.** <https://opentelemetry.io/docs/>. Landing page and glossary; useful when a term (`SpanKind`, `resource`, `scope`) needs the canonical definition.
- **OpenTelemetry SDK — trace API.** <https://opentelemetry.io/docs/specs/otel/trace/api/>. `SpanKind`, span links, span events, status codes.
- **OpenTelemetry SDK — sampling.** <https://opentelemetry.io/docs/specs/otel/trace/sdk/#sampling>. `ParentBased`, `TraceIdRatioBased`, `AlwaysOn`, `AlwaysOff`. Head-sampling primitives referenced in chapter 06.
- **OpenTelemetry Protocol (OTLP).** <https://opentelemetry.io/docs/specs/otlp/>. Wire format for shipping spans over gRPC / HTTP.

## Semantic conventions

- **OpenTelemetry semantic conventions repo.** <https://github.com/open-telemetry/semantic-conventions>. The source of truth for `gen_ai.*` names and status. Read from here, not from a blog post.
- **OpenTelemetry semantic conventions — GenAI.** <https://opentelemetry.io/docs/specs/semconv/gen-ai/>. Human-readable rendering of the GenAI attributes and events. Currently in development / experimental — re-read on each release.
- **OpenInference specification.** <https://github.com/Arize-ai/openinference>. Repo root; the `spec/` directory is the authoritative attribute-name list. `openinference.span.kind`, per-kind attributes, indexed message / tool / document sub-attributes.
- **OpenInference auto-instrumentors.** <https://github.com/Arize-ai/openinference/tree/main/python/instrumentation>. The `openinference-instrumentation-*` packages for OpenAI, Anthropic, LangChain, LlamaIndex, DSPy, Bedrock, MistralAI, Groq, Vertex, and others. TS packages sit under `js/`.

## Collector

- **OpenTelemetry Collector.** <https://opentelemetry.io/docs/collector/>. Deployment, pipeline model (receivers → processors → exporters), and configuration.
- **`opentelemetry-collector-contrib`.** <https://github.com/open-telemetry/opentelemetry-collector-contrib>. The distribution that ships the tail-sampling and PII-relevant processors used in chapter 06.
- **`tailsamplingprocessor`.** <https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor>. Policies for `status_code`, `latency`, `numeric_attribute`, `string_attribute`, `rate_limiting`, `and`, `composite`, `ottl_condition`. The primary chapter-06 tool.
- **`attributesprocessor`.** <https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/attributesprocessor>. `hash`, `delete`, `update`, pattern-matched actions on attribute values. First scrubbing layer at the Collector.
- **`redactionprocessor`.** <https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/redactionprocessor>. Regex redaction on attribute values.
- **`probabilisticsamplerprocessor`.** <https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/probabilisticsamplerprocessor>. Head-style probabilistic sampling at the Collector when SDK-level head sampling is not possible.

## Backends covered in the chapter-03 bake-off

- **Arize Phoenix.** <https://docs.arize.com/phoenix>. Self-hostable OpenInference-native trace viewer + eval hooks. Repo: <https://github.com/Arize-ai/phoenix>.
- **Langfuse.** <https://langfuse.com/docs>. First-class session and user modelling; SaaS and self-host. OpenTelemetry ingest endpoint documented on the vendor docs; re-read for the current URL and auth mechanics.
- **W&B Weave.** <https://weave-docs.wandb.ai/>. `@weave.op` decorator model; traces alongside W&B experiment runs. OTel ingestion documented on the vendor docs.
- **Braintrust.** <https://www.braintrust.dev/docs>. Eval-first trace model; SDK and OTel ingest.
- **LangSmith.** <https://docs.smith.langchain.com/>. LangChain / LangGraph native; also accepts OpenTelemetry OTLP.

## Vendor-neutral trace backends and dashboards

- **Jaeger.** <https://www.jaegertracing.io/docs/>. Historical OTel-compatible trace backend; useful as a fallback when a self-host LLM-app viewer is not on the table.
- **Grafana Tempo.** <https://grafana.com/docs/tempo/latest/>. Object-storage-backed trace backend; pairs with Grafana dashboards for cost / latency roll-ups in `mod-111`.

## PII, scrubbing, and pseudonymisation

- **Microsoft Presidio.** <https://microsoft.github.io/presidio/>. Named-entity PII detection + anonymisation. Chapter-06 default text scrubber.
- **NIST SP 800-122 — Guide to Protecting the Confidentiality of Personally Identifiable Information.** <https://csrc.nist.gov/publications/detail/sp/800-122/final>. PII definition and handling guidance referenced in chapters 04 and 06.
- **GDPR — Regulation (EU) 2016/679.** <https://eur-lex.europa.eu/eli/reg/2016/679/oj>. Art. 4(1) defines personal data; Art. 4(5) defines pseudonymisation. Cited in the user-id-hashing and residual-risk sections.
- **NIST FIPS 198-1 — The Keyed-Hash Message Authentication Code (HMAC).** <https://csrc.nist.gov/publications/detail/fips/198/1/final>. HMAC specification, referenced by the salted user-id hashing recipe in chapters 04 and 06.
- **Google Cloud DLP.** <https://cloud.google.com/security/products/dlp>. Cloud alternative to Presidio for teams already in GCP.
- **Amazon Macie.** <https://aws.amazon.com/macie/>. AWS alternative to Presidio for teams already in AWS.

## Governance and safety cross-references (deeper coverage in peer modules)

- **NIST AI Risk Management Framework (AI 100-1, 2023).** <https://www.nist.gov/itl/ai-risk-management-framework>. Referenced from `mod-101` and `mod-112`; the sampling + PII policy from chapter 06 is a Measure-function artefact.
- **NIST AI RMF Generative AI Profile (AI 600-1, 2024).** <https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile>. Suggested actions for generative AI risks; the residual-risk statement in chapter 06 maps to the "Manage" and "Measure" functions.
- **OWASP Top 10 for Large Language Model Applications.** <https://owasp.org/www-project-top-10-for-large-language-model-applications/>. LLM06 (Sensitive Information Disclosure) is the driver behind the PII policy in chapter 06.
- **ISO/IEC 25059:2023 — Quality model for AI systems.** <https://www.iso.org/standard/80655.html>. Referenced from `mod-101`; the schema in chapter 04 records evidence for the quality characteristics 25059 defines. Paywalled.
- **ISO/IEC 42001:2023 — AI management systems.** <https://www.iso.org/standard/81230.html>. Referenced from `mod-112`; the retention and right-to-erasure procedures in chapter 06 are 42001-relevant artefacts.
- **EU Artificial Intelligence Act (Regulation (EU) 2024/1689).** <https://eur-lex.europa.eu/eli/reg/2024/1689/oj>. Referenced from `mod-112`; log / trace retention obligations for high-risk systems bear on the retention tiers in chapter 06.

## Application-eval frameworks that ride on this trace shape (used from mod-104 onward)

- **RAGAS.** <https://docs.ragas.io/>. Faithfulness, answer relevancy, context precision / recall — computed off the RETRIEVER + LLM.reply span content this module records. Deepened in `mod-105`.
- **DeepEval.** <https://docs.confident-ai.com/>. Application-layer eval with trace hooks. Referenced from `mod-104` / `mod-105`.
- **TruLens.** <https://www.trulens.org/>. Feedback functions attaching to traces / spans. Referenced from `mod-104` / `mod-105`.
- **Promptfoo.** <https://www.promptfoo.dev/docs/intro/>. CI-oriented prompt eval; consumes trace fixtures. Deepened in `mod-106`.
- **UK AISI Inspect agent harness.** <https://inspect.aisi.org.uk/>. Trajectory-level agent eval; consumes the AGENT / TOOL span shape. Deepened in `mod-103`.

## Peer curriculum interfaces

- **`agentic-ai-engineer-learning`** — <https://github.com/ai-engineering-curriculum/agentic-ai-engineer-learning> — the agent implementations this track instruments in chapters 04 and 05.
- **`rag-engineer-learning`** — <https://github.com/ml-engineering-curriculum/rag-engineer-learning> — the retrievers this track instruments as RETRIEVER spans.
- **`llm-application-developer-learning`** — <https://github.com/ai-engineering-curriculum/llm-application-developer-learning> — the LLM app engineering this track instruments end-to-end.
- **`ai-evaluation-engineer-learning` (Governance family, level 25)** — <https://github.com/ai-governance-curriculum/ai-evaluation-engineer-learning> — the sign-off owner for the sampling + PII policy in chapter 06.

<!-- needs-research: on the next research cycle, re-check the current OTel GenAI spec status on content-as-attribute vs content-as-event, confirm the tail-sampling processor's supported policy names, refresh vendor-hosting posture (self-host availability, EU residency options) for Weave / Braintrust / LangSmith, and refresh the OpenInference span-kind vocabulary against the `spec/` directory. -->
