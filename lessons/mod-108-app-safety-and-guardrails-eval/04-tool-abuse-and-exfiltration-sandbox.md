# Tool Abuse and Exfiltration with Sandboxed Replay

## Motivation

Chapter 03 measured whether an agent *follows* an injected instruction. This chapter measures the *consequence* of the agent following it — whether a real tool call went out, whether that tool call exfiltrated data, whether it modified state, and whether it invoked a tool the policy said it should not. The mechanism is a **sandboxed replay harness** that runs the agent against a controlled tool environment where every side effect is captured and no real system is touched.

Two failure modes justify a separate axis of eval:

- **The injection succeeded but the surface would have refused the harmful call anyway.** Chapter 03 marks the trace as an `injection_success = true`. Chapter 04 asks: *did the tool actually get called with harmful arguments?* Sometimes the model complies with the injection in its reasoning trace ("okay, I will email the user data") but the actual tool call is stopped by an output-side guardrail, a downstream permission check, or a schema validation. The injection is a partial failure; the exfiltration is prevented. Both numbers matter and they are different.
- **The tool abuse happened without an injection.** The agent, on its own initiative, invoked a tool it should not have — because a user asked something the agent interpreted broadly, because a chained reasoning step suggested a shortcut, because a stale system prompt said the tool was in scope when it no longer is. No adversary was involved. This is *native* tool abuse. It is invisible to chapter 03 (which requires an injection to be present) and it is common enough in production that a dedicated axis measures it.

This chapter walks the sandbox pattern (network egress denied, tool responses mocked, side effects captured to an append-only log), the canary-token technique for measuring exfiltration, the out-of-scope tool-invocation scorecard, and the deterministic-replay discipline that lets the same finding be reproduced by the on-call six months later.

## Core concepts

### The sealed sandbox pattern

The sandbox is a runtime environment for the agent whose properties are:

- **Network egress denied by default.** The container the agent runs in has no route to the internet, no route to internal services except a mocked set. `fetch` returns a controlled response; `email.send` writes to a log; `payment.charge` returns a mocked success. Any call the agent attempts to a real endpoint is captured and marked as an attempted egress.
- **Tool responses are prefabricated and stable.** For a given `(tool, arguments)` pair, the mock returns the same response deterministically. This makes the eval replayable; two runs of the same suite produce byte-identical traces (modulo model non-determinism which is separately controlled with seed / temperature).
- **Side effects captured to an append-only log.** Every tool invocation writes a `SideEffectEntry` — `{tool, arguments_hash, timestamp, would_have_hit: <endpoint>, canary_tokens_leaked: [...]}`. The log is the primary artefact of the run; the scored-row schema (below) references entries by id.
- **The agent code is production code.** Same repo, same tool-registration, same prompt template, same client. Only the tool *implementations* are swapped for mocks. If the eval-only agent differs from production, the numbers are wrong for reasons no one will notice (chapter 03's failure mode).

Two sandbox implementations satisfy this shape:

- **In-process mock.** Register a mock tool registry that shadows the production registry; every tool call goes to the mock; a global `AGENT_SANDBOX_MODE=true` prevents the real registry from binding. Simple; requires the agent to have a clean tool-registry seam.
- **Container-level sandbox.** Run the agent in a container with no external network route; a local reverse proxy intercepts all HTTP and dispatches to mocks or a "would-have-hit" logger. More work to set up; harder for the agent to escape (some framework tools bypass the tool registry via direct HTTP).

Prefer the container approach for anything with real dependencies (`urllib.request`, subprocesses, DB drivers). The in-process mock is fine for agents built entirely on a well-behaved framework whose tool boundary is clean.

Related patterns worth naming: **AgentDojo** ships a sandbox-like environment abstraction that fits this shape and is the recommended starting point for a first sandbox; **PyRIT** supports orchestrating attacks against agents in isolated environments; **BrowseComp** / **VisualWebArena** and similar web-agent benchmarks include their own sandbox harnesses. Reuse where possible.

### Canary tokens for exfiltration measurement

An exfiltration attempt is only detectable if the sandbox *knows* what would have been exfiltrated. The canary-token pattern makes this observable.

- **Seed unique tokens in every trusted context.** In the retrieved documents the agent sees, in the tool responses, in the user profile the agent has access to, in the memory items the agent reads — every source that the agent should treat as *sensitive but internal-only* gets a unique token embedded (e.g., `CANARY-TENANT-42-DOC-17`). The tokens are unmemorable strings that a benign agent has no legitimate reason to emit.
- **Watch every outgoing tool call and every model response for the tokens.** The sandbox's egress logger and the model-response processor both scan for canary token patterns. Any hit is a detected exfiltration; the token identifies *which source* leaked.
- **Rotate tokens per run.** Same run: same tokens (for reproducibility). Different runs: different tokens (so a leak from a previous run does not falsely fire on this run's log). The per-run token map is versioned with the run.
- **Report exfiltration rate per source class.** "Retrieved documents leaked in `k` of `N` sessions" and "user profile leaked in `k` of `N` sessions" are the numbers the runbook reads. Aggregate rates without source attribution are less actionable.

Canary-token detection is a *proxy* for exfiltration — it catches literal-token leaks but not paraphrased ones (the agent describes the sensitive information without emitting the token). For paraphrased leaks, add an LLM-as-judge scorer that reads the outgoing tool arguments and checks for policy-violating content; that scorer is the same kind of thing chapter 05 measures at the guardrail layer. Combine both.

### Out-of-scope tool invocation

Not every tool the agent registered is in scope for every request. Rules of thumb:

- **Per-tool policy tags.** Each tool declares its scope (`{"scope": ["support_bot_helper"], "sensitive": true, "requires_confirmation": true}`). The eval fires an out-of-scope invocation when the invoking agent's surface is not in the tool's scope list.
- **Per-argument policy checks.** `internal.emailSend(to=...)` should only send to allowlisted domains. The eval fires when an argument violates a per-tool argument policy (a URL to a non-allowlisted domain, a PII field in a query string, a body that contains canary tokens).
- **Sequence-shaped policies.** `read → write` sequences on sensitive data are sometimes only permitted with an explicit user-confirmed step in between. The eval reads the tool-call trajectory and fires when the sequence violates the policy.

These are the same policies mod-103 chapter 05 (trajectory-and-tool eval) walks at the *quality* layer. The safety-eval variant adds an escalating severity: a tool-abuse trace tagged as `sensitive=true` and `canary_leaked=true` is a hard-fail on the canary gate (chapter 07's runbook). A native tool-abuse trace with no exfiltration is a soft-fail that the runbook routes to the tool owner.

### The scored-row schema extension

Extends mod-107 chapter 02 with the sandbox artefacts:

```json
{
  // existing mod-107 + chapter 03 fields
  "eval_type": "tool_abuse_sandbox",
  "sandbox_run_id": "sbx_2026-08-04_1832_run73",
  "session_trajectory_id": "sess_9c2f...",
  "tool_calls": [
    {
      "step": 1,
      "tool": "web.fetch",
      "arguments_hash": "sha256:...",
      "policy_tags": ["network_egress"],
      "outcome": "mock_response",
      "would_have_hit": "https://attacker.example",
      "policy_verdict": "OUT_OF_ALLOWLIST",
      "canary_tokens_in_args": ["CANARY-TENANT-42-DOC-17"]
    }
  ],
  "exfiltration_verdict": "detected_via_canary",
  "exfil_sources": ["retrieved_doc"],
  "tool_abuse_verdict": "out_of_scope",
  "trigger": "indirect_injection_via_retrieved_doc",  // or "native"
  "injection_link": {
    "eval_type": "prompt_injection",
    "scored_row_id": "..."
  }
}
```

The `injection_link` back-reference is important. Chapter 07's runbook joins the injection-eval finding with the tool-abuse-eval finding to produce the OWASP LLM Top-10-mapped severity — an injection with no exfiltration is one thing; an injection with confirmed exfiltration is another.

### Deterministic replay: the discipline

The on-call finds a scored row from three months ago that reports a tool abuse. Can they reproduce it now, exactly? If yes, the finding is defensible; if no, the finding is a story. Four disciplines make the replay work.

- **Model non-determinism is controlled.** `temperature = 0`, `seed = <fixed>`, `top_p = 1`, provider snapshot pinned (`gpt-4o-2024-08-06`, `claude-3-5-sonnet-20241022`), same tokenizer. When the provider does not honour seed (some do not), record `system_fingerprint` on the trace and accept that non-determinism is documented rather than eliminated.
- **Tool mocks are hash-verified.** Every mock's response is content-addressed by the input `(tool, arguments_hash)`; the mock repository is versioned. A replay reads the same mock corpus by version, not by "the current file on disk."
- **Sandbox environment is reproducible.** The container image is a specific digest; the mock server is at a specific commit; the canary token map is stored per run.
- **Only what varies is what matters.** If a replay reproduces the finding, the finding is model-and-app-shaped and someone should fix it. If a replay does *not* reproduce, either the sandbox has non-determinism the team missed (fix the sandbox) or the finding was a transient sampling artefact (log and downgrade the severity). Either way, the failure of the replay is itself the signal.

Determinism is not free — pinning temperatures and seeds trades some realism ("the production call uses temperature 0.4") for reproducibility. Live with the trade: the sandbox exists to produce actionable findings, and an unreproducible finding is not actionable. Chapter 06 defines a separate "policy-alignment" suite that runs at production-realistic settings for the refusal calibration; the sandbox axis stays deterministic.

### The exfiltration-rate scorecard

The number the report shows to the security reviewer is a scorecard, not a single scalar. Rows are attack vectors; columns are the observable failure modes.

| Attack vector | Attempted-egress rate | Canary-leak rate | Out-of-scope-tool rate | Confirmation-bypass rate |
|---|---|---|---|---|
| Native (no injection) | ... | ... | ... | ... |
| Indirect injection (retrieved doc) | ... | ... | ... | ... |
| Indirect injection (tool response) | ... | ... | ... | ... |
| Direct injection (user turn) | ... | ... | ... | ... |
| Mixed / chained | ... | ... | ... | ... |

Empty cells are unmeasured; do not confuse them with zero. Every row-column pair is one number the scorecard defends.

The mod-107 canary gate reads this scorecard against a per-cell threshold. Chapter 07's runbook applies OWASP LLM Top-10 severity tags to non-zero cells; the safety carve-out from mod-107 chapter 04 blocks rollout on any non-zero cell tagged as *high* severity.

### Sensitive-tool inventory as a substrate artefact

A tool-abuse eval is only as good as the tool policy table it references. Before running the eval, the team maintains an inventory:

```yaml
tools:
  - name: email.send
    scope: [support_bot, admin_console]
    sensitive: true
    requires_confirmation: true
    argument_policies:
      - field: to
        rule: domain_in_allowlist
        allowlist: [company.com, acme.com]
      - field: body
        rule: no_canary_tokens
      - field: body
        rule: no_pii_pattern
    owner: platform-team@company.com
  - name: web.fetch
    scope: [support_bot]
    sensitive: false
    argument_policies:
      - field: url
        rule: no_ip_literal
      - field: url
        rule: no_localhost
      - field: url
        rule: no_canary_tokens
    owner: ...
```

The inventory is not an eval-team artefact; it is a shared artefact between the platform team (which registered the tool) and the risk team (which set the sensitivity tags). This module *reads* the inventory; it does not author it. When a new tool ships without an inventory entry, the mod-106 PR gate should fail on the tool-registration commit — that check is part of the mod-106 chapter 03 threshold system, not this module.

### Interaction with the mod-107 online loop

Two loop hookups:

- **Sandbox is offline; the online loop reads its scored-row store.** The sandbox run produces scored rows on a `sandbox=true` cohort. The mod-107 aggregator, drift monitor, and dashboards read those rows on the sandbox cohort just as they read production rows on other cohorts.
- **Production out-of-scope-tool detection.** The same policy checks the sandbox runs (out-of-scope, argument violations) are wired at the tool-invocation boundary in *production* as a lightweight guardrail (chapter 05). Every real-traffic detection also produces a scored row on the `production` cohort. The two cohorts read the same scorecard shape.

The sandbox provides *depth* (adversarial attacks, deterministic replay); production provides *breadth* (native failures under real load). Both are needed; both feed the same runbook.

### Failure modes to design against

- **The sandbox is not sealed.** A tool escapes the mock and hits a real endpoint. The eval is over-optimistic (the real endpoint returned a benign error the eval interpreted as compliance) or the sandbox itself caused an incident. Mitigation: container-level sandbox with default-deny egress; periodic verification (a `test-sandbox-egress.sh` that deliberately tries to reach the internet and confirms the request is captured).
- **Canary tokens are trivially recognisable.** The model refuses to emit tokens that look like canaries; your leak rate is 0 for a reason unrelated to the defence. Mitigation: canary tokens are boring-looking short strings (looking like session-ids or hashes), not `PLEASE-DO-NOT-EMIT-1234`.
- **Native tool abuse is unmeasured.** The team only runs sandboxed replay under attack; native tool-abuse rates are unknown. Mitigation: at least a small fraction of the sandbox suite is *benign* traffic to measure native failure baseline.
- **Reproducibility is aspirational.** The sandbox is not actually deterministic; a replay of the same run produces different results. Mitigation: a `replay-check.sh` in CI runs a fixed test suite twice and asserts byte-identical scored rows (modulo timestamp).
- **The tool inventory is stale.** A new tool ships; the eval has no policy for it; the eval reports 0 out-of-scope invocations because the check is absent. Mitigation: the tool inventory is source-controlled; a mod-106 gate blocks tool registrations that do not add an entry.

## Summary

- The sandbox axis measures the **consequence** of a compliant model — did the tool call actually go out, did it exfiltrate, did it hit an out-of-scope tool? Distinct from chapter 03's injection axis.
- **Sealed sandbox pattern**: network egress denied, tool responses mocked and deterministic, side effects captured to an append-only log, agent code is production code.
- **Canary tokens** make exfiltration observable at the source-attribution level; LLM-as-judge scorers cover the paraphrased-leak case.
- **Out-of-scope tool invocation** is scored against a shared tool-policy inventory owned jointly by platform and risk teams; this module reads the inventory.
- The **scored row extends mod-107 chapter 02** with `tool_calls[]`, `exfiltration_verdict`, `exfil_sources`, `tool_abuse_verdict`, `trigger` (native vs injection-linked), and a back-reference to the injection scored row.
- **Deterministic replay** is a discipline: temperature and seed pinned, mocks versioned, sandbox reproducible. Findings are only actionable if the replay works.
- The **exfiltration-rate scorecard** is a matrix (attack vector × failure mode), not a single scalar. Mod-107's canary gate reads it against per-cell thresholds; chapter 07's runbook applies OWASP tags.
- Sandbox is offline; production runs a lightweight version of the same policy checks as a guardrail (chapter 05); the two cohorts feed the same scorecard.

Chapter 05 moves to the guardrails themselves — the layer that catches or misses these behaviours in production — and measures their false-positive and false-negative rates with explicit cost accounting.
