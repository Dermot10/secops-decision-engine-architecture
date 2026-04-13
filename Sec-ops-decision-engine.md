# SecOps Decision Engine — Architecture

A modular, event-driven security operations platform built with AI-assisted triage, detection-as-code, and a threat intelligence enrichment pipeline.

This repository documents the architecture, design decisions, and component breakdown of the platform. It is intended as a public reference for the system design — not the implementation.

---

## What it does

The platform ingests security events, enriches indicators of compromise against multiple threat intelligence sources, applies deterministic detection rules and AI-assisted classification, and produces triage decisions with full explainability output. Human analysts can review, override, and provide feedback that improves detection quality over time.

---

## High-level architecture

```
Log sources (auth, network, application)
            ↓
    Log Ingestion Layer          normalises events to common schema
            ↓
  Detection-as-Code Engine       YAML rules run against normalised events
            ↓
      Alert Gateway (Go)         accepts alerts, persists, queues
            ↓
      RabbitMQ (topic exchange)  event-driven pipeline
            ↓
      Triage Worker (Python)     enrichment + rules + AI classification
            ↓
  Threat Intel Service (Python)  AbuseIPDB, VirusTotal, GreyNoise
            ↓
      RabbitMQ results queue     triage decisions published back
            ↓
      Alert Gateway (Go)         consumes results, persists to MongoDB
            ↓
         Dashboard               analyst review, override, feedback
```

---

## Services

| Service                                                | Language             | Responsibility                                      |
| ------------------------------------------------------ | -------------------- | --------------------------------------------------- |
| [threat-intel-service](./docs/threat-intel-service.md) | Python               | Enriches indicators against threat intel APIs       |
| [alert-gateway](./docs/alert-gateway.md)               | Go                   | HTTP ingestion, queue management, result storage    |
| [triage-worker](./docs/triage-worker.md)               | Python               | Enrichment orchestration, rules engine, LLM triage  |
| [detection-as-code](./docs/detection-as-code.md)       | Python               | YAML detection rules, event matching, alert firing  |
| [log-ingestion](./docs/log-ingestion.md)               | Go                   | Log collection, normalisation, event routing        |
| [dashboard](./docs/dashboard.md)                       | TypeScript / Next.js | Analyst interface, triage review, override controls |

---

## Language decisions

**Go** owns the infrastructure boundary layers — services that operate at volume, need low latency, or manage queue topology. Alert ingestion, log collection, and result routing are all Go.

**Python** owns the intelligence layers — services that reason about data, call external APIs, or interact with LLMs. Enrichment, triage classification, and detection rule evaluation are all Python.

These two languages communicate via RabbitMQ queues. The wire format (JSON) is the contract between them — neither service imports types from the other.

---

## Design principles

**Explainability first** — every triage decision includes the signals that contributed to it, the confidence score, the rule or LLM that made the decision, and a human-readable summary. Nothing is a black box.

**Deterministic before AI** — a rules layer evaluates clear cases before the LLM is called. High confidence critical indicators escalate immediately. Low confidence low severity alerts close automatically. The LLM handles the ambiguous middle. This keeps AI costs low and decision latency fast for the majority of alerts.

**Graceful degradation** — if an enrichment source fails, the pipeline continues with the remaining sources. If the LLM is unavailable, the decision falls back to human review. No single dependency failure crashes the pipeline.

**Detection as code** — detection rules are YAML files, version controlled, testable against fixture data, and independent of the pipeline infrastructure. Rules can be added, modified, and tested without touching application code.

**Feedback loop** — human override decisions are recorded alongside the original triage decision. Over time this data informs detection rule effectiveness, false positive rates, and LLM prompt tuning.

---

## Data flow

### Alert ingestion

```
POST /v1/alerts
      ↓
Validate DTO
Assign alert_id + timestamp server-side
Persist to MongoDB (alerts collection)
Publish to RabbitMQ alerts.triage queue
Return 202 Accepted + alert_id
```

### Triage pipeline

```
Consume from alerts.triage
      ↓
Call threat intel service → IntelResult
      ↓
Apply deterministic rules
  ├── high confidence + high severity → escalate
  ├── low confidence + low severity  → auto_close
  └── ambiguous                      → LLM
      ↓
LLM classification (if needed)
  └── confidence below threshold     → human_review
      ↓
Produce TriageResult + ExplainabilityOutput
Publish to alerts.results queue
```

### Result storage

```
Consume from alerts.results
      ↓
Persist to MongoDB (triage_results collection)
      ↓
Available via GET /v1/results/{alert_id}
```

---

## Queue topology

```
Exchange: alerts.topic (topic, durable)

Bindings:
  alerts.*.*     → alerts.triage      (Go → Python worker)
  results.*.*    → alerts.results     (Python worker → Go)

Dead letter:
  alerts.dlx (direct exchange)
  alerts.dlq (dead letter queue)

Retry queues (TTL → back to main exchange):
  alerts.retry.5s   (5s TTL)
  alerts.retry.30s  (30s TTL)
  alerts.retry.2m   (120s TTL)
```

---

## Triage decision model

Every alert produces a `TriageResult` with an embedded `ExplainabilityOutput`:

```json
{
  "alert_id": "uuid",
  "indicator": "185.220.101.1",
  "decision": "escalate",
  "confidence": 0.91,
  "explainability": {
    "decision": "escalate",
    "confidence": 0.91,
    "reasons": [
      "IP has high abuse confidence score (100%)",
      "Classified as malicious by GreyNoise",
      "Flagged by 14 engines on VirusTotal"
    ],
    "signals": [
      { "source": "abuseipdb", "field": "abuse_confidence_score", "value": 100, "weight": 0.6 },
      { "source": "greynoise", "field": "classification", "value": 100, "weight": 0.4 },
      { "source": "virustotal", "field": "malicious_engines", "value": 15, "weight": 0.5 }
    ],
    "intel": { "...full IntelResult..." },
    "llm_summary": "High confidence malicious IP confirmed across three independent sources. Known Tor exit node with active scanning behaviour.",
    "rule_triggered": "high_confidence_multi_source",
    "overrideable": true
  },
  "processed_at": "2026-04-11T10:00:00Z"
}
```

Decisions: `escalate` | `auto_close` | `human_review`

---

## Technology stack

| Concern                | Technology                       |
| ---------------------- | -------------------------------- |
| API framework (Python) | FastAPI                          |
| API framework (Go)     | net/http                         |
| Queue                  | RabbitMQ (aio-pika / amqp091-go) |
| Cache                  | Redis                            |
| Database               | MongoDB                          |
| HTTP client            | httpx (Python) / net/http (Go)   |
| Data validation        | Pydantic v2 (Python)             |
| LLM                    | OpenAI / Anthropic Claude        |
| Threat intel           | AbuseIPDB, VirusTotal, GreyNoise |
| Dashboard              | Next.js + TypeScript + Tailwind  |

---

## Roadmap

- [ ] Detection recommendation engine — pattern mining, gap analysis, LLM-assisted rule generation
- [ ] Log ingestion layer — HTTP collector, syslog listener, auth log parser
- [ ] OCSF schema alignment
- [ ] Additional indicator types — domain, hash, URL enrichment
- [ ] Sigma rule compatibility
- [ ] Dashboard — alert feed, triage review, override controls
