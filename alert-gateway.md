# Alert Gateway

A Go service that acts as the entry point for all security alerts. It accepts alert submissions via HTTP, persists them to MongoDB, publishes them to the triage queue, and consumes triage results back from the worker.

---

## Responsibilities

- Accept alert DTOs via HTTP POST
- Validate and assign server-side fields (alert_id, timestamp)
- Persist alerts to MongoDB
- Publish alerts to RabbitMQ for async triage processing
- Consume triage results from the results queue
- Persist triage results to MongoDB
- Expose HTTP endpoints for alert and result retrieval

---

## Endpoints

```
POST /v1/alerts                      submit a new alert
GET  /v1/alerts/{alert_id}           retrieve a submitted alert
GET  /v1/results/{alert_id}          retrieve the triage result for an alert

GET  /health                         service health + dependency status
```

### Alert submission

Returns `202 Accepted` immediately — the alert has been queued, not yet triaged. The caller receives an `alert_id` to poll for results.

```json
{
  "alert_id": "uuid",
  "status": "queued"
}
```

### Alert DTO

```json
{
  "source": "detection-as-code",
  "rule_id": "auth-brute-001",
  "rule_name": "Brute force login attempt",
  "severity": "high",
  "indicator": {
    "type": "ip",
    "value": "185.220.101.1"
  },
  "context": {
    "src_ip": "185.220.101.1",
    "username": "admin",
    "failed_attempts": 12,
    "service": "ssh"
  },
  "raw_log": "Failed password for admin from 185.220.101.1 port 4823 ssh2"
}
```

`context` is a flexible `map[string]any` — different detection rules produce different context shapes. The triage worker passes it through to the LLM as evidence for classification.

`alert_id` and `timestamp` are assigned server-side. Client-supplied values are ignored.

---

## Queue management

The gateway owns the full RabbitMQ topology and declares it at startup:

```
Exchange:  alerts.topic (topic, durable)

Queues:
  alerts.triage    bound to alerts.*.*    (gateway → worker)
  alerts.results   bound to results.*.*   (worker → gateway)
  alerts.dlq       dead letter queue
  alerts.retry.5s  retry with 5s TTL
  alerts.retry.30s retry with 30s TTL
  alerts.retry.2m  retry with 120s TTL
```

Routing keys follow the pattern `alerts.{source}.{severity}` for triage and `results.triage.{decision}` for results.

Failed messages follow the retry chain before landing in the DLQ. Manual acknowledgement is used throughout — messages are only acked after successful processing.

---

## Storage

Two MongoDB collections:

| Collection       | Content                                          |
| ---------------- | ------------------------------------------------ |
| `alerts`         | Raw alert DTOs as submitted                      |
| `triage_results` | Triage decisions with full explainability output |

Both collections are indexed on `alert_id` with a unique constraint.

Redis is not used in the gateway — MongoDB direct lookup by indexed `alert_id` is performant enough for this access pattern. Redis is reserved for the threat intel service where API rate limit protection justifies a cache layer.

---

## Service layers

```
handler     HTTP concerns — decode, validate, encode response
service     Business logic — alert creation, result retrieval
store       MongoDB operations — insert, find by alert_id
queue       RabbitMQ — publish, consume, topology setup
dto         Shared data types — Alert, TriageResult, ExplainabilityOutput
```

The handler orchestrates service calls but has no direct knowledge of MongoDB or RabbitMQ. The service layer has no HTTP concerns. Each layer has one reason to change.

---

## Architecture decisions

**202 Accepted over 200 OK** — triage is async. Returning 200 would imply the decision is in the response body. 202 communicates that work has been queued and the result will be available separately.

**Server-side ID assignment** — alert producers are not trusted to supply their own IDs. The gateway assigns a UUID at ingestion time, which becomes the stable reference across all downstream systems.

**Go for this layer** — the gateway is the infrastructure boundary. It handles concurrent HTTP requests, manages queue connections, and coordinates between MongoDB and RabbitMQ. Go's concurrency model, low memory overhead, and fast startup make it the right choice here.

**Topology ownership** — the gateway declares the full queue topology at startup. If topology changes are needed (new queues, binding updates), they are made in one place and applied at next startup.
