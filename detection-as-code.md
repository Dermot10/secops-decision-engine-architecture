# Detection-as-Code Engine

A Python service that loads YAML detection rules, evaluates them against normalised security events, and fires alerts into the pipeline when rules match. Detection rules are version-controlled, independently testable, and decoupled from the pipeline infrastructure.

---

## Responsibilities

- Load and validate YAML detection rules at startup
- Evaluate rules against incoming normalised events
- Manage time-based aggregation windows (e.g. count events per IP within 60 seconds)
- Extract context fields when a rule fires
- POST alerts to the alert gateway

---

## Detection rule format

Rules are defined in YAML. Each rule specifies the event category it applies to, the conditions that must be true, an optional aggregation window, the indicator field to enrich, and the context fields to include in the alert.

```yaml
id: auth-brute-001
name: Brute force login attempt
severity: high
description: Multiple failed SSH login attempts from the same source IP.
tags:
  - brute-force
  - authentication
enabled: true
tested: false

detection:
  source: auth
  conditions:
    - field: event_type
      operator: equals
      value: login_failed
    - field: src_ip
      operator: exists
  aggregation:
    field: src_ip
    operator: greater_than
    value: 5
    window_seconds: 60

indicator:
  type: ip
  field: src_ip

context_fields:
  - src_ip
  - username
  - hostname
```

---

## Supported operators

| Operator       | Description                   |
| -------------- | ----------------------------- |
| `equals`       | Exact match                   |
| `not_equals`   | Not equal                     |
| `contains`     | String contains               |
| `greater_than` | Numeric greater than          |
| `less_than`    | Numeric less than             |
| `in`           | Value in list                 |
| `not_in`       | Value not in list             |
| `exists`       | Field is present and not null |

---

## Aggregation windows

Rules with an `aggregation` block use an in-memory sliding window to count events over time. The window is keyed by the specified field value — typically `src_ip` — so each source IP maintains its own independent count.

Events outside the window are pruned on each evaluation. This means the aggregation is approximate under high concurrency but appropriate for the detection use case.

---

## Normalised event schema

Rules evaluate against `NormalisedEvent` objects — a common schema that all log sources are converted to before rule evaluation. This means rules are source-agnostic.

Key fields:

```
event_id       unique identifier
timestamp      UTC datetime
category       auth | network | process | file | dns
event_type     login_failed | connection | port_scan | etc.
src_ip         source IP address
dest_ip        destination IP address
username       user account involved
hostname       host where event occurred
process        process name
raw            original log line
metadata       flexible dict for additional fields
```

---

## Engine components

```
app/engine/loader.py     Finds and validates YAML rule files
app/engine/runner.py     Evaluates rules against events, manages windows
app/engine/matcher.py    Evaluates individual conditions against events
app/engine/publisher.py  POSTs alerts to the alert gateway
```

Each component has a single responsibility. The runner orchestrates the others — it calls the matcher for condition evaluation and the publisher when a rule fires.

---

## Testing

Rules are tested against fixture event data before deployment. Tests validate that rules fire on matching events, produce the correct context fields, respect aggregation thresholds, and do not fire on non-matching events.

```
tests/
  fixtures/
    auth_events.json     sample auth log events
    network_events.json  sample network events
  test_rules.py          rule evaluation tests
```

The `tested` field on each rule tracks whether it has passed validation. Alerts from untested rules can be weighted lower in the triage pipeline — a rule firing for the first time in production without prior testing is a signal worth noting.

---

## Rule categories

```
detections/
  auth/
    brute_force.yaml           multiple failed logins from same IP
    privilege_escalation.yaml  sudo/su usage outside baseline
  network/
    port_scan.yaml             connection attempts to multiple ports
    suspicious_outbound.yaml   outbound to non-standard ports
```

---

## Architecture decisions

**YAML for rule definitions** — rules are human-readable, version-controllable, and reviewable by analysts without Python knowledge. The schema is validated by Pydantic at load time so malformed rules fail loudly at startup.

**Python for the engine** — condition evaluation, aggregation, and YAML parsing are all expressive in Python. The engine reads naturally and the rule schema maps cleanly to Pydantic models.

**Rules are source-agnostic** — by evaluating against a normalised event schema rather than raw log formats, rules work across any log source that can be normalised. Adding a new log source means writing a parser, not rewriting rules.

**Publisher decoupled from runner** — the runner returns `(rule, context)` tuples. The publisher handles HTTP delivery separately. This means the engine can be tested without a running alert gateway.

---

## Planned

- Rule effectiveness tracking — false positive rate per rule based on triage outcomes
- Detection recommendation engine — pattern mining on event history to suggest new rules
- Sigma rule compatibility — import and convert community Sigma rules
- AI-assisted rule generation — LLM produces candidate rules from identified patterns
