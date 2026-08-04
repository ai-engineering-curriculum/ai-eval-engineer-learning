# Guardrail Effectiveness with an Explicit FP / FN Report

## Motivation

Chapters 02 – 04 measured the app under attack. This chapter measures the *layer that catches the attack when it fires* — the guardrail. The report shape is a confusion matrix, not a single scalar, because the two failure modes have opposite costs:

- **False positive.** The guardrail blocked a request that was fine. The user hits a "your request violated our safety policy" message on a query the policy actually allows. Repeated FPs are a first-order product regression: users stop trusting the surface. Product will (correctly) push back if the FP rate is unreported.
- **False negative.** The guardrail let through content the policy prohibits. A single FN can be a public incident; the FN rate is a first-order safety regression. The safety team will (correctly) push back if the FN rate is unreported.

A guardrail vendor's marketing page will show either "we caught 99 % of attacks" (recall-only) or "we have only 1 % false positives" (precision-only). Both are half-truths; the on-call needs both numbers, at *your* operating point, on *your* traffic, priced in *your* dollars and milliseconds. This chapter walks the confusion matrix, the operating-point choice, the guardrail-stack survey, and the cost / latency accounting that turns "we deployed a guardrail" into a defensible line item.

## Core concepts

### The guardrail-stack survey

A production LLM app usually has three or four guardrails at different layers. Naming the stack is the first step of any eval.

| Layer | What it sees | Common tools |
|---|---|---|
| **Input classifier** | The user turn before the model sees it | Llama Guard, OpenAI Moderation, Azure AI Content Safety, Google Model Armor, Perspective API, in-house classifier |
| **Programmable rails** | The full conversation, retrieved context, tool-call plans; can rewrite or redirect | NeMo Guardrails, Guardrails AI, LangChain's constitutional AI, custom |
| **Output classifier** | The model's response before it goes back to the user | Llama Guard, OpenAI Moderation, Azure AI Content Safety, Google Model Armor, Perspective API, custom |
| **Tool-argument checker** | Tool invocation arguments before execution | Custom policy engine, Guardrails AI validators, chapter 04's inventory policies |

Two properties define a guardrail for eval purposes:

- **Decision.** Binary block / allow, categorical (`safe | S1_violence | S2_sexual | S3_criminal | ...`), or scored (probability). The confusion matrix eval below treats every guardrail as `block | allow` at an operating point (a threshold on the score, or a set-membership on the category).
- **Wrapping.** Rule-based (regex, keyword) vs classifier-based (a model returning a score) vs programmable (a framework running a policy). Each has different failure modes; the eval scorecard is the same shape regardless.

Read the docs for the ones you deploy — the semantics differ. Llama Guard 3 categorises across MLCommons's hazard taxonomy; OpenAI Moderation returns fine-grained categories and scores; Azure AI Content Safety splits into severity levels per category; Google Model Armor is a policy-shaped guardrail with detectors; NeMo Guardrails is a dialog manager whose rails are configured via Colang; Guardrails AI is a validator library ("this output must match this XML schema, no PII, etc.").

**A single-vendor stack is a bad default.** Guardrail vendors' training data biases their FN pattern; combining two vendors' classifiers reduces the residual FN rate on categories one covers better than the other. The cost trade-off (chapter 05's operating-point section) determines when the second vendor is worth it.

### The confusion matrix, at *your* operating point

Every guardrail eval produces four numbers per policy category:

|  | **Policy says HARMFUL** | **Policy says ALLOWED** |
|---|---|---|
| **Guardrail says BLOCK** | TP (true positive) | FP (false positive) |
| **Guardrail says ALLOW** | FN (false negative) | TN (true negative) |

Derived rates:

- `precision = TP / (TP + FP)` — of the requests the guardrail blocked, how many were actually harmful.
- `recall = TP / (TP + FN)` — of the harmful requests, how many the guardrail blocked.
- `FPR = FP / (FP + TN)` — of the benign requests, how many the guardrail (wrongly) blocked.
- `F1 = 2 · precision · recall / (precision + recall)` — the balanced harmonic mean.

Product cares about **FPR** on the benign stream (how often a normal user gets wrongly blocked). Safety cares about **recall** on the harmful stream (how many attacks got through). The runbook shows both.

Two disciplines make this defensible:

- **Ground truth is human-reviewed and policy-aligned.** The label for each row is "does this request violate *your written safety policy*?", not "does this vendor's classifier fire?" A vendor-labelled dataset trains vendors' own numbers upward; you want your own labels because your policy is not the vendor's policy. Chapter 06's policy reconciliation is the shared labelling substrate.
- **The labelled set is stratified.** Half harmful, half benign, per policy category — otherwise the class imbalance in production traffic (usually ~99.9 % benign) hides the FN rate. Report per-category numbers; the "average" over categories hides the categories the guardrail is weak on.

### The ROC / PR curve and operating point selection

Classifier-based guardrails return a *score*, and the operating point is a *threshold*. The **ROC curve** (TPR vs FPR as the threshold varies) and the **precision-recall curve** (precision vs recall as the threshold varies) both parameterise the trade-off.

- **AUC-ROC / AUC-PR** are single-number quality metrics for the classifier. Useful for cross-vendor comparison. Not sufficient for a deployment decision.
- **The operating point** is a specific threshold you shipped with. Every deployment metric (FP, FN, cost, latency) is *at* that operating point. When the report says "recall = 91 % at FPR = 0.5 %", the "at" is the load-bearing word.
- **Product-driven point.** The operating point is chosen to satisfy a product constraint, not to maximise F1. Typical constraint: "FPR ≤ 0.5 % on the benign eval slice at 90 %+ recall on the harmful eval slice for the category `<x>`." If no point on the ROC satisfies both, the guardrail is not good enough for the surface; escalate.

Two operating-point patterns worth naming:

- **Chained low-threshold + high-threshold cascade.** A cheap low-threshold classifier catches 80 % of harmful traffic (high FPR is tolerable at this stage because the next stage disambiguates); a mid-tier classifier resolves the 20 % on the boundary. Reduces average cost.
- **Category-specific thresholds.** The same guardrail runs at different operating points per policy category. `self-harm` recall must be near 100 % (low threshold); `mild profanity` FPR must be low (higher threshold). Vendors like Azure AI Content Safety and Llama Guard 3 support per-category thresholds natively; NeMo Guardrails supports it via Colang.

### Cost accounting per guarded request

Every guardrail call is a cost. On a production surface at 10 rps, guardrail cost is often *the dominant safety-eval line item*, and if it is not measured it is the item that will grow silently.

Derive the cost end-to-end:

- `R` — request rate (e.g., 10 rps → 864 000 requests / day)
- `n_g` — guardrails on the critical path per request (e.g., 3: input classifier + programmable rails + output classifier)
- `c_g` — cost per guardrail call (e.g., input classifier at $0.0001, output classifier at $0.001, programmable rails at $0.0002 for embedding-based checks)
- Daily cost = `R × 86400 × Σ c_g` = `864 000 × ($0.0001 + $0.001 + $0.0002)` = `~$1 123 / day`, or ~$34 000 / month.

That number is worth defending against the ASR / exfil-rate improvement it buys. Compare against:

- The mod-107 online-eval judge budget (typically $50 – $2 500 / month for the same surface).
- The model-inference cost for the same request rate (typically 10 – 100× larger than either).

If the guardrail cost is a substantial fraction of the model-inference cost, the operating point is possibly wrong (too much guardrail traffic; move to a cascade), the guardrail choice is possibly wrong (a $0.001 / call classifier where a $0.00005 / call one would work), or the guardrail is possibly not on the critical path (some categories can be sampled rather than run per-request).

### Latency accounting

Guardrails on the critical path add latency; users notice. Two disciplines:

- **Per-guardrail latency budget.** An input classifier at the front adds `median = 40ms, p95 = 200ms`. An output classifier adds `median = 60ms, p95 = 350ms`. Programmable rails at 100+ms per rail can add hundreds of ms. Report a **latency budget** for the safety layer as a fraction of the surface's total latency budget (typical: 10 – 20 % of the SLO).
- **Parallelisation and cascading.** The input classifier and the retrieval hop are independent — fire in parallel. The output classifier and the response-streaming decision are not (you cannot stream a response you have not yet cleared). Cascading a cheap classifier before an expensive one reduces mean latency without sacrificing recall.

The cost and latency numbers land in the same report. Mod-111 (cost / latency / quality trade-off) reads them; the guardrail scorecard is one input.

### The scored-row schema for guardrail eval

Additions to the mod-107 chapter 02 schema:

```json
{
  // existing mod-107 fields
  "eval_type": "guardrail_effectiveness",
  "guardrail_id": "llama_guard_3_at_input",
  "guardrail_config_hash": "sha256:...",
  "guardrail_position": "input",  // input | rails | output | tool_args
  "operating_point": {
    "threshold": 0.35,
    "category_thresholds": {"S1_violence": 0.20, "S8_self_harm": 0.10}
  },
  "policy_label": "HARMFUL",  // ground truth from the human-labelled set
  "policy_category": "S8_self_harm",
  "guardrail_decision": "BLOCK",
  "guardrail_score": 0.72,
  "guardrail_categories_fired": ["S8_self_harm"],
  "verdict": "TP",  // TP | FP | FN | TN
  "guardrail_cost_usd": 0.001,
  "guardrail_latency_ms": 63,
  "human_reviewer_id": "reviewer_7f2a...",
  "policy_snapshot": "safety_policy_v11"
}
```

Two invariants:

- Every row has both `policy_label` (ground truth) and `guardrail_decision` (system output) so the confusion-matrix rollup is a direct group-by.
- `policy_snapshot` is on every row so a mid-run policy change is not a silent invalidation. When the policy is bumped (chapter 06's reconciliation), a re-labelling pass is a documented event, not a mystery drift.

### The scorecard shape the report produces

For each guardrail, per policy category:

```
policy_category: S8_self_harm
operating_point: threshold=0.10 (category-specific)

               harmful  benign
    BLOCK        142       3        precision = 0.979
    ALLOW          4     998        recall    = 0.973
                                    FPR       = 0.003
                                    F1        = 0.976

cost (per guarded request):  $0.001
latency (per guarded request, p95): 58ms

change vs previous config (safety_policy_v10 → safety_policy_v11):
    recall  0.968 → 0.973  (+0.005)
    FPR     0.002 → 0.003  (+0.001)

decision: KEEP  (recall improvement dominates FPR regression per acceptance criteria)
```

The trailing "decision" line is the eval team's recommendation to product / safety; the actual choice is theirs. The report format is the same across guardrails so head-to-head comparison is straightforward.

### Comparing guardrails: the head-to-head

When picking between two guardrails (or two operating points on one guardrail), the head-to-head is a table:

|   | Guardrail A | Guardrail B | Ensemble (A ∨ B) |
|---|---|---|---|
| Recall on `S1_violence` | 0.94 | 0.89 | 0.97 |
| Recall on `S8_self_harm` | 0.97 | 0.99 | 0.995 |
| FPR on benign | 0.005 | 0.008 | 0.012 |
| Cost per request | $0.0001 | $0.0010 | $0.0011 |
| p95 latency | 40ms | 60ms | 60ms (parallel) |

The ensemble row is often the deployment choice for high-stakes surfaces — `A ∨ B` (block if either fires) increases recall at the cost of FPR, cost, and latency. Chapter 07's runbook records which combination is deployed and why.

### Interaction with mod-107 and mod-106

- **PR gate.** Guardrail configuration changes (threshold changes, category-list changes, guardrail-swap) go through the mod-106 CI gate. The gate replays a labelled eval slice against the new configuration; a regression on recall (or a significant regression on FPR) blocks the merge. Chapter 07 walks the thresholds.
- **Online loop.** The mod-107 sampler bias-toward-interesting includes guardrail-fired traces. A rising `guardrail_fired_rate` in production is a signal (either a real attack wave or a benign traffic pattern the guardrail is FP-ing on); the drift monitor picks it up.
- **Canary gate.** A guardrail-config change canary reads the same scorecard. The safety carve-out from mod-107 chapter 04 applies: a canary that raises the FN rate on any category with a zero-tolerance floor is rejected.

### Failure modes to design against

- **The vendor's benchmark is not your operating point.** The vendor reports `AUC = 0.98` on their own eval; on your traffic at your threshold it is `AUC = 0.89`. Mitigation: never quote the vendor's number in a production runbook; always measure at your operating point on your labels.
- **The policy label and the guardrail categories are misaligned.** You label rows against your policy's `sensitive_content`; the vendor's categories are `S3_sexual_content` and `S4_hate_speech`; the mapping is one-to-many. Mitigation: chapter 06's reconciliation defines the mapping; the labelled set uses your policy categories; the vendor's output categories are joined via a per-vendor mapping table.
- **The labelled set is stale.** A category that was uncommon a year ago is common now; the labels do not cover it. Mitigation: quarterly refresh of the labelled set from real production traffic (with the redaction contract from chapter 02).
- **Cost accounting is missing.** The team compares guardrails on F1 alone; the deployed guardrail is 10× more expensive than the alternative for a 0.5 % F1 improvement. Mitigation: cost per guarded request is on the scorecard; the recommendation line reads both quality and cost.
- **The programmable rails are unmeasured.** NeMo Guardrails / Guardrails AI shipped; the team assumed they add value; the FP / FN measurement was skipped because the rails are "not classifiers." Mitigation: every guardrail on the critical path is scored — the rail's block / allow / rewrite decision is measurable regardless of the mechanism.

## Summary

- The **confusion matrix** at *your* operating point on *your* labels is the load-bearing eval artefact. `precision`, `recall`, `FPR`, `F1` per category — not a single scalar.
- The **guardrail stack survey**: input classifier, programmable rails, output classifier, tool-argument checker. Different layers, different vendors, same scorecard shape.
- **AUC-ROC / AUC-PR** compare classifiers. The **operating point** is what ships. Per-category thresholds are standard on Llama Guard 3, Azure AI Content Safety, and NeMo Guardrails.
- **Cost accounting** is derived end-to-end: `R × 86400 × Σ c_g × 30`. Latency accounting: per-guardrail budget as a fraction of the SLO. Both live on the scorecard.
- The **scored row extends mod-107 chapter 02** with `guardrail_id`, `operating_point`, `policy_label`, `guardrail_decision`, `verdict`, cost, latency, and `policy_snapshot`.
- The **head-to-head report** compares guardrails and ensembles on recall, FPR, cost, and latency together — the ensemble row is often the deployment choice for high-stakes surfaces.
- **Integration** with the mod-106 PR gate, the mod-107 online loop, and the mod-107 canary gate uses the same scorecard.
- Failure modes: vendor number ≠ your number, policy / category misalignment, stale labelled set, missing cost accounting, unmeasured programmable rails.

Chapter 06 moves to the calibration axis — refusal and over-refusal — where the ground truth is not "harmful vs allowed" but "does the model's decision match the written safety policy?"
