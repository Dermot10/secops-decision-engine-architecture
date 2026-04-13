# Triage Worker

A Python asyncio service that consumes alerts from the triage queue, orchestrates enrichment and classification, and publishes structured triage decisions back to the results queue.

---

## Responsibilities

- Consume alert messages from RabbitMQ
- Call the threat intel service to enrich the alert indicator
- Apply deterministic rules for clear-cut decisions
- Call the LLM for ambiguous cases
- Produce a `TriageResult` with full explainability output
- Publish the result to the results queue

---

## Processing pipeline

```
Consume alert from alerts.triage
        ↓
Call threat intel service
  └── enrichment failed → human_review, publish, stop
        ↓
Apply deterministic rules
  ├── critical risk level              → escalate
  ├── high confidence + high severity  → escalate
  ├── high confidence + multi-source   → escalate
  ├── low confidence + low severity    → auto_close
  ├── all sources failed               → human_review
  └── ambiguous                        → LLM
        ↓
LLM classification (if needed)
  └── confidence < threshold           → human_review
        ↓
Produce TriageResult + ExplainabilityOutput
Publish to alerts.results
```

---

## Decision thresholds

| Threshold               | Default | Description                                |
| ----------------------- | ------- | ------------------------------------------ |
| `confidence_escalate`   | 0.85    | Above this, deterministic escalate         |
| `confidence_auto_close` | 0.15    | Below this, deterministic auto close       |
| `min_ai_confidence`     | 0.60    | Below this, LLM falls back to human review |

Thresholds are configurable — they are expected to be tuned over time as real alert outcomes accumulate.

---

## Triage decisions

| Decision       | Meaning                                                |
| -------------- | ------------------------------------------------------ |
| `escalate`     | Credible threat — requires immediate analyst attention |
| `auto_close`   | Likely false positive or benign activity               |
| `human_review` | Ambiguous — analyst should review before action        |

---

## Explainability output

Every decision includes full explainability:

```json
{
  "decision": "escalate",
  "confidence": 0.91,
  "reasons": ["...human readable reasons from enrichment and LLM..."],
  "signals": ["...raw signals from threat intel sources..."],
  "intel": { "...full IntelResult embedded..." },
  "llm_summary": "High confidence malicious IP confirmed across three sources.",
  "rule_triggered": "high_confidence_multi_source",
  "overrideable": true
}
```

`rule_triggered` identifies whether the decision came from a deterministic rule or the LLM. `overrideable` signals to the analyst interface that the decision can be challenged.

---

## AI reliability layer

The LLM is not trusted unconditionally. A confidence threshold check wraps every LLM response — if the model returns a confidence below `min_ai_confidence`, the decision falls back to `human_review` regardless of what the model decided. This is logged explicitly so the threshold can be tuned.

LLM failures (API errors, invalid JSON responses) also fall back to `human_review` rather than raising. The pipeline continues for all other alerts.

---

## Service layers

```
consumer_service    RabbitMQ consumption, message routing
enrichment_service  Calls threat intel HTTP endpoint
rule_service        Deterministic condition evaluation
llm_service         AI classification business logic, fallback handling
publisher_service   Serialises and publishes TriageResult
clients/llm_client  Raw LLM API call, JSON parsing
prompts/            System prompt and user prompt builder
models/             Pydantic schemas — Alert, IntelResult, TriageResult
```

The LLM client is isolated from the service layer. Swapping providers (OpenAI → Anthropic) means changing one file — the client — without touching the service, prompt, or consumer.

---

## Architecture decisions

**Python for this layer** — the worker reasons about data, calls multiple external APIs, and interacts with an LLM. Python's ecosystem for async HTTP, AI SDKs, and data validation (Pydantic) makes it the natural choice. The asyncio event loop handles concurrent enrichment calls efficiently.

**Deterministic rules before AI** — most alerts fall into clear categories. Running an LLM on every alert would add latency and cost unnecessarily. The rules layer resolves the majority of cases in milliseconds. The LLM handles only the ambiguous middle.

**Intel embedded in explainability** — the full `IntelResult` is nested inside the `ExplainabilityOutput` rather than at the top level of `TriageResult`. The enrichment evidence is part of the explanation, not a separate concern.

**No FastAPI** — the worker is a queue consumer running on an asyncio event loop. It has no HTTP endpoints to expose. FastAPI would add unnecessary overhead. Potentially reintroduced for a /health endpoint.
