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

---

## ADR-008: Situational use of Waitgroup > Errgroup

**Decision** - for fan-out messaging within the log ingestion service using waitgroup to handle concurrent goroutines as opposed to errgroup

**Context** - wait group choice for log ingestion service's publisher module fan-out function.

**Reasoning** - co-ordinates completion without coupling both goroutines and coupling outcomes for both raw and triggered events. Each goroutine will log it's own error and each respective caller proceeds regardless. Errgroup would be more favourable for all-or-nothing semantics, better for multi-step data fetch where partial results are useless.

---

## ADR-009: Lifespan as Composition Root

**Decision** — Shared infrastructure (Redis, `CacheService`, `EnrichmentService`, `httpx.AsyncClient`) constructed once in the FastAPI lifespan function and stored on `app.state`. Dependencies retrieved per-request via FastAPI's dependency injection system.

**Context** — Infrastructure resources such as connection pools and service instances are expensive to construct and stateful — creating them per-request would discard connection pools, fragment cache state, and add unnecessary overhead on every call.

**Reasoning** — Lifespan is the natural composition root — the single place with legitimate knowledge of all layers simultaneously. Constructing infrastructure once ensures connection pools are reused, cache state is unified across requests, and resource lifecycle is tied explicitly to application startup and shutdown. Dependency functions (`get_cache`, `get_enrichment_service`) retrieve from `app.state` per-request, providing FastAPI's injection mechanism without rebuilding the object graph on each call. This separates construction (lifespan) from retrieval (dependencies) cleanly.

---

## ADR-010: IntelSource Protocol

**Decision** - Restructured Intel source into a Protocol class as coordinator's typed boundary. Coordinator depends on the abstraction, not the concrete implmentations.

**Context** - `EnrichmentService` was coupled directly to `BaseIntelSource`, binding the coordinator to the ABC machinery rather than the capability it needs.

**Reasoning** - Protocol class allows structural typing - any class with a matching `enrich` signature satisfies the contract without explicit inheritance. Enable runtime interchangability via dependency injection, and decouples the coordinator from implementation details of the source hierarchy.

---

## ADR-011: BaseIntelSource ABC

**Decision** - Introduced `BaseIntelSource` ABC implementing the template method pattern for shared HTTP lifecycle management.

**Context** - Each source was independently managing client creation, exception handling, and failure returns - duplicated logic with inconsistent behaviour across sources.

**Reasoning** - ABC centralises cross-cutting concerns (client creation, timeout handling, exception normalisation, safe `SourceResult` fallback) in the public `enrich` method. Subclasses implement only `_enrich` - the source-specific happy path. One change to error handling propagates to all source automatically.

---

## ADR-012: SourceResult Dataclass vs IntelResult Pydantic Model

**Decision** - `SourceResult` implemented as a dataclass, `IntelResult` as a Pydantic model.

**Context** - Both types carry enrichment data but serve different purposes - one is internal plumbing between layers, the other crosses an API boundary.

**Reasoning** - `SourceResult` never leaves the service layer, requires no validation, and is constructed from internally controlled code - dataclass overhead is sufficient. `IntelResult` is the FastAPI `response_model` , requires serialisation, field validation, and OpenAI schema generation - Pydantic is the correct toll at an API boundary. Conflating the two would add unnecessary Pydantic overhead to internal data carriers.

---

## ADR-013: VirusTotalSource Single Class with Internal Branching.

**Decision** - IP, domain, and hash enrichment handled within a single `VirusTotalSource` class via internal routing in `_enrich`.

**Context** - VirusTotal supports multiple indicator types under the same API, using the same authentication, base URL, headers, and analysis stats structure.

**Reasoning** - Shared surface (auth, base URL, headers, `_parse_analysis_stats`) outweighs the difference between indicator types, which are contained within private methods (`_enrich_ip`, `_enrich_domain`, `_enrich_hash`, `_build_ip_result` etc). Separate classes per indicator type would duplicate shared logic and fragment a cohesive source into artificial boundaries. New indicator types extend the existing class rather than requiring a new source registration.

---

## ADR-014: URLhaus as Domain Source

**Decision** - URLhaus integrated as a domain indicator source using the host lookup endpoint, not the URL lookup endpoint.

**Context** - URLhaus was initially considered for URL indicator enrichment but per-URL lookups suffer from rapid staleness - URLs are taken down and purged from the database quickly.

**Reasoning** - The host endpoint returns aggregrated domain-level intelligence - active URL count, blacklist status, malware family tags - which is stable, high-signal data for domain enrichment. Per-URL lookups return `no_results` or `invalid_url` for recently purged entries, making them unreliable for prod enrichment. Domain-level aggregation is more actionable and maps cleanly to `IndicatorType.DOMAIN`.

---

## ADR-015: Shared httpx.AsyncClient via Lifespan

**Decision** - Single `httpx.AsyncClient` constructed in lifespan, injected into `EnrichmentService`, passed to sources at call time.

**Context** - Each source was independently constructing and tearing down an `httpx.AsyncClient` per enrichment call, discarding the connection pool after a single use.

**Reasoning** - A shared client maintains a connection pool across all sources and all requests - TCP connections are reused between concurrent source calls during `gather`, Per-request client construction adds latency and resource overhead that compounds under load. Lifecycle tied to app startup and shutdown ensures clean resource management via `aclose()` on shutdown, consistent with the existing lifespan infrastructure pattern.
