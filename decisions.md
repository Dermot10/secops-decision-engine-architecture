# Architecture Decision Records

Key design decisions made during the build, with context and reasoning.
These are subject to change if remodelling of the system occurs.

---

## ADR-001: Go for infrastructure layers, Python for intelligence layers

**Decision** — Go owns services that operate at the boundary (alert ingestion, log collection, queue management). Python owns services that reason about data (enrichment, triage, detection evaluation).

**Context** — the platform requires both high-throughput event handling and rich data reasoning with LLM integration. No single language is optimal for both.

**Reasoning** — Go's concurrency model, low memory overhead, and fast startup are well-suited to services that handle concurrent HTTP requests and manage queue connections. Python's ecosystem for async HTTP clients, AI SDKs, and data validation (Pydantic) makes it the natural choice for services that call external APIs and process structured data.

**Consequence** — services communicate via RabbitMQ queues with JSON as the wire format. Neither service imports types from the other. The contract is the message shape, not a shared library.

---

## ADR-002: RabbitMQ topic exchange with dead letter and retry topology

**Decision** — a single topic exchange (`alerts.topic`) with two queues (`alerts.triage`, `alerts.results`), tiered retry queues, and a dead letter queue.

**Context** — alert processing can fail at multiple points (enrichment API timeout, LLM unavailability, transient network errors). Messages must not be lost on failure.

**Reasoning** — a topic exchange allows routing by source and severity pattern. Tiered retry queues (5s, 30s, 2m TTL) give transient failures time to resolve before a message is declared dead. The DLQ provides a recoverable store for permanently failed messages without blocking the main queue.

**Consequence** — the gateway owns and declares the full topology at startup. Adding new routing patterns requires a topology change in one place.

---

## ADR-003: Deterministic rules before LLM

**Decision** — a rules layer evaluates clear-cut cases before the LLM is called. The LLM handles only ambiguous middle cases.

**Context** — calling an LLM on every alert adds latency (~3-5s) and cost. The majority of alerts fall into clear categories that don't require language model reasoning.

**Reasoning** — high confidence critical indicators should escalate immediately without waiting for LLM response time. Low confidence low severity alerts should close immediately. Routing only ambiguous cases to the LLM keeps the fast path fast and the cost proportional to uncertainty.

**Consequence** — the `rule_triggered` field in the explainability output distinguishes deterministic from AI decisions. Decision thresholds are configurable and expected to be tuned over time.

---

## ADR-004: Intel embedded in explainability, not at the top level of TriageResult

**Decision** — the full `IntelResult` is nested inside `ExplainabilityOutput` rather than as a top-level field on `TriageResult`.

**Context** — the enrichment result is evidence that supports the explanation of a decision. It is not a separate top-level property of the decision itself.

**Reasoning** — placing intel inside the explainability output communicates the relationship correctly. When an analyst reads the result, the enrichment data is contextualised as "this is why we made this decision" rather than floating alongside it. It also keeps `TriageResult` focused — it describes the decision, not the evidence gathering process.

---

## ADR-005: No shared type library between Python services

**Decision** — `Signal`, `IntelResult`, and related types are recreated in the triage worker rather than imported from the threat intel service.

**Context** — the triage worker calls the threat intel service via HTTP. Both services need to represent the response shape.

**Reasoning** — sharing a library creates a hard dependency between services. If the threat intel service changes its model, the triage worker must update in lockstep. Services should be independently deployable. The JSON wire format is the contract — both services validate against the same JSON shape but own their own Pydantic models.

**Consequence** — the duplication is intentional and documented. If the shared types grow complex enough to justify a package, a `shared/` directory with an installable Python package is the path forward.

---

## ADR-006: MongoDB over Redis in the alert gateway

**Decision** — MongoDB is used for alert and result persistence in the gateway. Redis is not used.

**Context** — earlier designs included Redis in the gateway for result caching. Redis is used in the threat intel service.

**Reasoning** — the gateway's access pattern is simple: write once on ingestion, read by `alert_id`. MongoDB with a unique index on `alert_id` handles this with O(1) lookup — no meaningful performance difference from a Redis cache. Adding Redis would introduce an additional dependency with no rate-limit justification (unlike the threat intel service where Redis protects third-party API quotas).

---

## ADR-007: Server-side alert ID assignment

**Decision** — `alert_id` and `timestamp` are assigned by the alert gateway, not by the producer.

**Context** — alert producers (Detection-as-Code, manual submissions) submit alerts without IDs.

**Reasoning** — the gateway is the authoritative source of truth for alert identity. Client-supplied IDs cannot be trusted for uniqueness and create a surface for ID collision or spoofing. Assigning server-side ensures every alert has a globally unique, gateway-issued identifier that is consistent across MongoDB, RabbitMQ, and all downstream consumers.
