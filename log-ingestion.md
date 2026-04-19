# Log Ingestion Layer

A Go service that collects security events from multiple log sources, normalises them to a common schema, and routes them to the Detection-as-Code engine for rule evaluation. At enterprise scale a company would likely use a SIEM platform as a ingestion layer for log sources.

---

## Responsibilities

- Accept log events from multiple sources and formats
- Accept single log event or in batches
- Parse and normalise events to the common `NormalisedEvent` schema
- Route normalised events to the Detection-as-Code engine
- Persist raw events for historical querying and pattern mining

---

## Collectors

| Collector | Description                                 | Status    |
| --------- | ------------------------------------------- | --------- |
| HTTP      | Accept JSON log events via POST endpoint    | Completed |
| File      | Tail log files and parse line by line       | Planned   |
| Syslog    | Listen on UDP for syslog-formatted messages | Planned   |

The HTTP collector is the primary integration path — applications ship logs via a simple POST rather than requiring an agent or syslog configuration.

---

## Parsers

Each log source requires a parser that maps raw fields to the `NormalisedEvent` schema.

| Parser      | Handles                                   | Status    |
| ----------- | ----------------------------------------- | --------- |
| Auth        | SSH failed logins, sudo usage, PAM events | Completed |
| Web         | Nginx, Apache access and error logs       | Planned   |
| Cloud       | AWS CloudTrail, GCP Audit Logs            | Planned   |
| Application | Generic structured JSON logs              | Planned   |

Parsers are isolated and independently testable. Adding a new log source means writing one parser file.

---

## Normalised event schema

All events are converted to a common schema before being passed to the Detection-as-Code engine. This schema is aligned with OCSF (Open Cybersecurity Schema Framework) where applicable.

```json
{
  "event_id": "uuid",
  "timestamp": "2026-04-11T10:00:00Z",
  "category": "auth",
  "event_type": "login_failed",
  "source": "sshd",
  "src_ip": "185.220.101.1",
  "dest_ip": "10.0.0.5",
  "username": "admin",
  "hostname": "web-server-01",
  "raw": "Failed password for admin from 185.220.101.1 port 4823 ssh2",
  "metadata": {}
}
```

`metadata` is a flexible map for source-specific fields that don't fit the standard schema. Detection rules can reference metadata fields using the same condition syntax as top-level fields.

---

## Architecture decisions

**Go for this layer** — log ingestion is a high-throughput, low-latency concern. Parsing thousands of log lines concurrently with goroutines, minimal memory overhead, and fast startup make Go the right choice. Python's async model could handle it but Go is more natural here.

**Normalise before routing** — the Detection-as-Code engine evaluates a common schema, not raw log formats. Normalisation happens at the boundary so the rest of the system never sees source-specific formats.

**Persist raw events** — normalised events are persisted alongside the alert and triage data. This provides the historical event store required by the planned Detection Recommendation Engine for pattern mining.

**OCSF alignment** — using an established schema standard means community detection rules (Sigma) map more cleanly to the normalised events, and the system can interoperate with other OCSF-aligned tools.
