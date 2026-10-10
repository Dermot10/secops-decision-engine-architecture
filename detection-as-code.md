# Detection-as-Code Engine

**Architecture Overview, Design Decisions and Operational Behaviour**

**Status:** Existing implementation with documented design intentions and planned enhancements
**Language:** Python
**Framework:** FastAPI
**Rule format:** YAML
**Version control:** GitHub, abstracted behind a VCS client
**Purpose:** Security detection engineering and automated rule deployment

---

## 1. Executive Summary

The Detection-as-Code service is a Python application that loads declarative YAML detection rules, evaluates them against normalised security events, and publishes alerts when detection conditions are satisfied.

Detection logic is maintained independently of application code and version-controlled through GitHub. A rule-management API and GitHub webhook integration provide mechanisms for authoring, reviewing, synchronising, and reloading detection definitions.

The architecture separates detection content from the engine responsible for executing it. This allows detection rules to evolve through a version-controlled workflow without requiring changes to the core evaluation logic for every new detection.

The system is organised around five principal concerns:

- **Rule management:** Load, validate, and maintain detection definitions.
- **Event evaluation:** Match normalised events against rule conditions.
- **Stateful detection:** Track event counts within time-based aggregation windows.
- **Alert publication:** Extract relevant context and send alerts to the alert gateway.
- **Rule lifecycle management:** Synchronise rule changes from version control and reload the active detection set.

The architectural objective is to establish a modular, testable, and extensible detection engine in which security analysts can develop detection content independently of the underlying execution infrastructure.

## 2. Scope and Responsibilities

### 2.1 In scope

The service is responsible for:

1. Loading and validating detection rules at application startup.
2. Exposing API endpoints for event processing and rule management.
3. Evaluating detection conditions against normalised events.
4. Maintaining in-memory aggregation state for threshold-based detections.
5. Extracting configured context fields when detections fire.
6. Publishing alerts to a downstream alert gateway.
7. Integrating with GitHub through a version-control abstraction.
8. Synchronising relevant rule changes and triggering rule reloads.
9. Enforcing authentication and webhook verification at the appropriate boundaries.

### 2.2 Out of scope

The engine is not intended to own every stage of the security pipeline. Responsibilities such as raw log collection, source-specific parsing, long-term event storage, analyst case management, and downstream alert triage can be delegated to other services.

The exact ownership of event normalisation and alert triage depends on the wider application architecture and should be documented alongside the corresponding services.

## 3. High-Level Architecture

The service can be understood as two related flows: the event detection path and the rule management path.

### 3.1 Event detection path

```text
Log Sources
    |
    v
Source-specific Parsers
    |
    v
Normalised Events
    |
    v
Event API / Processing Endpoint
    |
    v
RuleRunner
    |
    +----> RuleMatcher
    |
    +----> Aggregation State
    |
    v
Matching Rule + Context
    |
    v
Alert Publisher
    |
    v
Alert Gateway
    |
    v
Downstream Triage / Response
```

The normalisation layer converts source-specific events into a common representation. The engine evaluates this representation against the configured rules, applies aggregation where required, and prepares alerts for downstream processing.

### 3.2 Rule management path

```text
Analyst / Rule Author
    |
    v
Rule Management API
    |
    v
VCS Client
    |
    v
GitHub Repository
    |
    +----> Review / Branch / Commit Workflow
    |
    v
GitHub Push Webhook
    |
    v
Signature Verification
    |
    v
Event and Branch Filtering
    |
    v
Detection File Change Identification
    |
    v
Local File Synchronisation
    |
    v
RuleLoader
    |
    v
RuleRunner Reload
```

GitHub provides the version-controlled source of detection content. Webhooks notify the service of relevant repository changes, while the VCS client encapsulates the operations required to retrieve and write files.

**Important distinction:** GitHub is the source of truth for version-controlled detection definitions. The in-memory rules and local detection directory are runtime representations that must be kept consistent with that source.

## 4. Component Architecture

The application uses a modular structure in which each component owns a specific responsibility.

### 4.1 Engine components

| Component                 | Responsibility                                                                  |
| ------------------------- | ------------------------------------------------------------------------------- |
| `app/engine/loader.py`    | Discover, parse, and validate YAML detection definitions.                       |
| `app/engine/runner.py`    | Orchestrate rule evaluation, aggregation, and reload behaviour.                 |
| `app/engine/matcher.py`   | Evaluate individual detection conditions against events.                        |
| `app/engine/publisher.py` | Deliver generated alerts to the alert gateway.                                  |
| Detection schemas         | Define the structure and validation rules for detection definitions and events. |

The runner coordinates the evaluation process. It invokes the matcher to determine whether conditions are satisfied and uses the publisher to deliver the resulting alerts.

The intended separation between matching and publication allows rule evaluation to be tested independently of HTTP delivery.

### 4.2 API and application components

| Component                                 | Responsibility                                                                                            |
| ----------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| FastAPI routers                           | Expose webhook, event-processing, and rule-management endpoints.                                          |
| Application lifespan                      | Initialise application resources and load rules during startup.                                           |
| Dependencies                              | Construct or provide reusable services, including the VCS client and rule runner.                         |
| Configuration                             | Supply branch names, repository settings, filesystem paths, credentials, and other runtime configuration. |
| Authentication middleware or dependencies | Validate bearer tokens for protected application endpoints.                                               |

The lifespan mechanism provides a central place for startup initialisation. Detection rules are loaded into application state during startup so that request handlers can use the active engine.

### 4.3 Version-control components

The VCS package separates the application's version-control contract from the GitHub-specific implementation.

```text
app/vcs/
    __init__.py
    base.py
    github.py
```

**`base.py` — abstraction**

Defines the common VCS interface, including file reading, listing, writing, and webhook validation operations. Shared result and file models, such as `VCSCommitResult` and `VCSFile`, provide consistent contracts.

**`github.py` — provider implementation**

Implements the VCS operations using GitHub's API. It handles repository paths, content encoding, commits, branch creation, and pull requests.

**`__init__.py` — public package interface**

Re-exports the intended public types and implementations so consumers can import them from the package rather than depending on individual internal modules.

For example:

```python
from app.vcs import GitHubClient, VCSClient
```

This reduces import coupling and clarifies which names form the package's public interface.

### 4.4 Dependency injection

FastAPI dependency injection provides the webhook and other endpoints with their required services.

For example:

```python
vcs: VCSClient = Depends(get_vsc_client)
rule_runner: RuleRunner = Depends(get_rule_runner)
```

The type annotations communicate the expected interfaces, while `Depends(...)` instructs FastAPI to resolve the dependencies.

This design avoids constructing concrete infrastructure clients directly inside route handlers. It improves testability and allows dependencies to be replaced with mocks or alternative implementations.

The VCS client is also cached using an LRU-based mechanism, according to the current application notes, to avoid repeatedly constructing equivalent clients. The cache's lifetime, configuration invalidation, and concurrency characteristics should be verified against its implementation.

---

## 5. Detection Rule Model

Detection rules are expressed as YAML documents. The declarative format makes rules readable, reviewable, and version-controllable without requiring authors to implement detection logic in Python.

### 5.1 Example detection rule

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

This example describes a rule that matches failed login events, requires a source IP to be present, and applies a threshold over a time window.

The exact semantics of the aggregation threshold — including when the sixth event fires, whether matching events are counted before evaluation, and whether subsequent events generate repeated alerts — depend on the runner implementation and should be specified in the rule-engine contract.

### 5.2 Rule fields

| Field                 | Purpose                                                                   |
| --------------------- | ------------------------------------------------------------------------- |
| `id`                  | Stable identifier for the detection rule.                                 |
| `name`                | Human-readable detection name.                                            |
| `category`            | Classification used for organisation and potentially directory placement. |
| `severity`            | Intended urgency or impact classification.                                |
| `description`         | Explains the behaviour the rule detects.                                  |
| `enabled`             | Indicates whether the rule should participate in evaluation.              |
| `tested`              | Records whether the rule has passed the relevant validation process.      |
| `tags`                | Supports classification, discovery, and filtering.                        |
| `detection`           | Defines applicable event criteria and optional aggregation.               |
| `indicator`           | Identifies the principal entity extracted from a matching event.          |
| `context_fields`      | Names the additional event fields included in an alert.                   |
| `sigma_rule_id`       | Optional reference to a corresponding Sigma rule.                         |
| `false_positive_rate` | Optional field intended to record rule effectiveness information.         |

The `tested` and `false_positive_rate` fields support a future workflow in which rule quality and observed effectiveness can inform alert triage. Their presence in the schema does not by itself establish that automated effectiveness tracking is implemented.

### 5.3 Rule schema validation

Pydantic models provide a typed contract for rule definitions.

The model includes an operator enumeration, condition objects, an optional aggregation window, indicator mappings, and the top-level detection rule.

Validation should occur during rule loading so malformed definitions are detected before they are relied upon in event processing.

The loader's exact failure policy — whether a single invalid rule prevents all rules from loading or invalid rules are isolated — should be documented according to its implementation.

### 5.4 Supported operators

| Operator       | Intended semantics                                  |
| -------------- | --------------------------------------------------- |
| `equals`       | Exact equality.                                     |
| `not_equals`   | Values are not equal.                               |
| `contains`     | A value contains the specified substring or member. |
| `greater_than` | Numeric value is greater than the threshold.        |
| `less_than`    | Numeric value is less than the threshold.           |
| `in`           | Value is a member of a specified collection.        |
| `not_in`       | Value is not a member of a specified collection.    |
| `exists`       | Field is present and not null.                      |

The matcher should define consistent behaviour for missing fields, null values, incompatible types, and invalid operator inputs. In particular, the distinction between a missing field and a field explicitly set to `null` should be intentional rather than accidental.

### 5.5 Rule organisation

An example repository layout is:

```text
detections/
    auth/
        brute_force.yaml
        privilege_escalation.yaml
    network/
        port_scan.yaml
        suspicious_outbound.yaml
    web/
        web_bruteforce.yaml
        web_sqli.yaml

```

Organising rules by category makes them easier to discover, review, and maintain.

The repository layout and the rule schema should remain consistent. The provisional schema includes a top-level `category` field, while the original example places the applicable category in `detection.source`. These fields should be reconciled in the canonical schema before the format is treated as a stable public contract.

---

## 6. Normalised Event Model

### 6.1 Design rationale

Different security log sources represent similar activities using different field names, formats, and structures.

The engine therefore evaluates a common `NormalisedEvent` model instead of implementing source-specific logic in every detection rule.

Source-specific parsers are responsible for converting incoming data into this representation. This allows the detection engine to remain largely independent of the underlying log source.

Adding a new source should primarily require a parser and appropriate mappings, rather than rewriting every rule.

### 6.2 Event schema

The current model notes describe the following fields:

| Field                                    | Purpose                                                      |
| ---------------------------------------- | ------------------------------------------------------------ |
| `event_id`                               | Unique event identifier.                                     |
| `timestamp`                              | Event timestamp, intended to use UTC.                        |
| `category`                               | Broad activity category, such as authentication or network.  |
| `event_type`                             | Specific activity, such as `login_failed` or `port_scan`.    |
| `source`                                 | Identifier for the originating log source.                   |
| `src_ip`, `dest_ip`                      | Source and destination IP addresses.                         |
| `src_port`, `dest_port`                  | Source and destination ports.                                |
| `username`                               | Associated user account.                                     |
| `hostname`                               | Relevant host.                                               |
| `process`                                | Process name or identifier, depending on the source mapping. |
| `file_path`                              | Relevant file path.                                          |
| `http_method`, `http_uri`, `http_status` | HTTP request context.                                        |
| `bytes_sent`                             | Network or HTTP transfer context.                            |
| `user_agent`                             | HTTP client identification string.                           |
| `raw`                                    | Original log line, where retained.                           |
| `metadata`                               | Additional source-specific fields.                           |

Optional fields allow the same event model to represent different kinds of activity without requiring every source to populate every field.

The model includes event categories for authentication, network, process, file, DNS, and web activity. Its event-type enumeration includes examples such as successful and failed logins, port scans, process starts, file modifications, and HTTP errors.

### 6.3 Schema consistency

The canonical event contract should explicitly define:

- Whether timestamps must be timezone-aware and normalised to UTC.
- How source-specific fields are mapped to canonical fields.
- How missing values are represented.
- Whether event types must be valid for their corresponding categories.
- How additional fields in `metadata` are named and accessed by detection conditions.
- Whether the original event payload is retained and, if so, how sensitive information is handled.

The rule schema's `detection.source` and the event model's `source` have different potential meanings: the former appears to identify an applicable event category, whereas the latter identifies a log source. A single, unambiguous naming convention should be adopted to avoid confusing source filtering with category filtering.

---

## 7. Rule Evaluation Lifecycle

The engine's evaluation process can be described as the following logical sequence:

1. Receive a normalised event.
2. Identify the enabled rules applicable to that event.
3. Evaluate each rule's configured conditions through the matcher.
4. Update aggregation state for rules requiring a time-based threshold.
5. Determine whether the detection threshold has been met.
6. Extract the configured indicator and context fields.
7. Produce a rule-match result.
8. Pass the result to the alert publication layer.

The runner coordinates this sequence, while the matcher evaluates individual conditions and the publisher handles downstream delivery.

The precise order of category filtering, condition matching, and aggregation updates should be confirmed against the runner implementation and documented as part of the engine's behavioural contract.

### 7.1 Time-based aggregation

Some detections require repeated events within a specified interval rather than a single matching event.

For example, a brute-force detection may look for more than five failed login attempts associated with the same source IP within 60 seconds.

The current design uses an in-memory sliding window keyed by the configured aggregation field. Events associated with different keys, such as different source IPs, maintain independent aggregation state.

Events outside the window are pruned during evaluation.

This approach is simple and avoids introducing an external state store, but it has important consequences:

- Aggregation state is local to the running service.
- State is lost when the process restarts.
- Separate application workers do not automatically share counts.
- Concurrent event processing can affect counting and pruning behaviour unless explicitly synchronised.
- High event volumes may increase memory usage and evaluation overhead.

The original design notes characterise the window as approximate under high concurrency. That limitation should remain explicit unless the implementation is strengthened with appropriate synchronisation or an external state-management strategy.

### 7.2 Alert context

When a rule fires, configured `context_fields` determine which event attributes should be included in the resulting alert.

For the example brute-force rule, these fields are `src_ip`, `username`, and `hostname`.

The indicator mapping identifies the primary entity associated with the detection, such as an IP address, domain, hash, or URL.

Context extraction should handle missing optional fields and should avoid copying sensitive or unnecessarily large payloads into downstream alerts.

### 7.3 Alert publication

The publisher is responsible for communicating with the alert gateway.

The intended design separates the rule-matching result from the delivery mechanism. This allows the engine's detection logic to be tested without requiring a live alert gateway.

The alert publication contract should specify the alert payload schema, timeout behaviour, retry policy, duplicate handling, and treatment of delivery failures. These operational guarantees should not be assumed solely from the presence of a publisher component.

---

## 8. Application Startup and Rule State

Detection rules are loaded into application state during the FastAPI lifespan.

This provides a defined initialisation point before the service begins normal request processing.

The startup sequence is conceptually:

1. Initialise application dependencies.
2. Load detection definitions from the configured directory.
3. Parse and validate the rule documents.
4. Initialise the runner with the active rule set.
5. Make the application available to handle requests.

The exact order and failure policy should follow the lifespan and loader implementations.

### 8.1 Runtime reload behaviour

The current implementation notes indicate that reloads modify the existing rules in place.

This is an important architectural characteristic because in-place mutation does not inherently guarantee an atomic transition between rule versions.

If event processing and rule reloads occur concurrently, requests may potentially observe partially updated state unless the runner provides suitable locking, snapshotting, or other synchronisation.

A future improvement would be to load and validate a complete replacement rule set independently, then switch the active reference only after successful validation. This can provide a cleaner transition and allow the last known-good rule set to remain active if the replacement fails.

That improvement is a recommendation, not a claim about the current implementation.

---

## 9. GitHub Integration and Detection-as-Code Workflow

### 9.1 Source of truth

GitHub is the authoritative version-controlled repository for detection definitions.

The service maintains a local representation of these definitions so they can be loaded by the engine. GitHub commits, branches, and pull requests provide the change history and review workflow.

The service's local filesystem and in-memory rules are derived state and must be reconciled with the repository when changes are applied.

### 9.2 VCS abstraction

The VCS interface defines the operations needed by the application, including:

- Reading a file.
- Listing repository content.
- Writing a file.
- Validating webhook signatures.

The GitHub implementation provides these operations using the GitHub API.

This separation allows the API and engine layers to depend on an interface rather than directly on GitHub-specific HTTP operations.

It also makes it easier to mock repository operations during tests and potentially introduce another VCS provider in the future.

### 9.3 Writing detection rules

The rule-writing workflow supports two commit strategies.

**Direct commit**

The client writes the file to the configured branch through GitHub's contents API.

**Pull request**

The client creates or reuses a dedicated update branch, writes the file to that branch, and attempts to open a pull request against the configured base branch.

The pull-request strategy supports review before merging. The direct strategy is simpler but bypasses the review boundary provided by a pull request.

The strategy is selected by the configured commit strategy in the GitHub client.

### 9.4 Git SHA semantics

When updating an existing file through the GitHub contents API, the current file's SHA identifies the version being updated. GitHub uses this value to ensure the update applies to the expected file version.

The returned commit SHA identifies the resulting commit.

The two values serve different purposes:

- **Current file SHA:** Identifies the existing file version for an update.
- **Commit SHA:** Identifies the newly created commit.

For file content, the client converts the Python string to bytes, Base64-encodes those bytes, and converts the encoded result back to a string for the JSON request body.

Conceptually:

```text
Python string
    -> UTF-8 bytes
    -> Base64-encoded bytes
    -> Base64 string in JSON
    -> GitHub decodes the content
```

Base64 is an encoding mechanism, not encryption.

### 9.5 Pull-request considerations

The pull-request workflow creates a branch based on the configured base branch, commits the requested file, and attempts to open a pull request.

The implementation notes describe handling an existing branch by updating its reference to the current base SHA. This should be reviewed carefully because force-updating a branch may replace earlier unmerged changes on that branch.

The API client should also distinguish an already-existing pull request from other unsuccessful pull-request responses, preserve the relevant pull-request URL where possible, and avoid treating unrelated failures as successful completion.

---

## 10. GitHub Webhook Processing

### 10.1 Webhook semantics

A webhook is an HTTP endpoint exposed by the service and registered with an external system.

Rather than polling GitHub for changes, the service receives an HTTP POST when GitHub emits a configured event.

In this architecture, the webhook is the notification mechanism. GitHub remains the source of truth for repository content.

### 10.2 Processing flow

The provided webhook implementation exposes:

`POST /webhooks/github`

The documented processing sequence is:

1. Receive the HTTP request and read the raw request body.
2. Validate the GitHub webhook signature through the VCS client.
3. Reject an invalid signature with HTTP 401.
4. Ignore events other than `push`.
5. Extract the pushed branch and compare it with the configured branch.
6. Identify added and modified YAML/YML files under the configured detection path.
7. Ignore pushes that contain no relevant detection changes.
8. Fetch the changed files through the VCS client.
9. Write the files into the local detection directory.
10. Reload the detection rules.
11. Return a success response containing the changed files and active rule count.

Unexpected exceptions during synchronisation or reload are logged and surfaced as HTTP 500 responses.

### 10.3 Event filtering

The webhook filters on three primary criteria:

- Event type must be `push`.
- The pushed branch must match the configured branch.
- At least one relevant detection YAML file must have been added or modified.

Other GitHub events are acknowledged but ignored.

Changes to unrelated repository content do not trigger a rule reload.

### 10.4 Signature validation

The webhook uses GitHub's HMAC SHA-256 signature mechanism.

The signature is validated against the raw request body before the payload is processed. This establishes that the request body matches a message authenticated using the configured webhook secret.

The implementation should use a trusted validation routine that compares signatures safely and should never log the webhook secret or other sensitive authentication material.

A valid signature authenticates the message's origin and integrity under the shared-secret model; it does not by itself guarantee that a push is safe, authorised for every business purpose, or unique.

Duplicate deliveries and replay-related behaviour should be considered separately.

### 10.5 File synchronisation and deletion reconciliation

The webhook extracts added and modified detection YAML paths from the commits included in the push payload.

The synchronisation function translates each repository path into a relative path beneath the configured local detection directory. It creates missing parent directories, fetches the content through the VCS client, and writes the file locally.

After synchronisation, the service reloads the complete detection set rather than only loading the changed files.

This simplifies rule loading because the loader remains responsible for interpreting the complete directory state.

However, **the provided implementation does not explicitly remove local files deleted from GitHub**. Removed paths are excluded from the change list, but a subsequent full reload still sees any deleted files that remain on disk.

Therefore, complete deletion reconciliation is an intended design requirement that still needs explicit implementation or verification.

A robust reconciliation strategy should ensure that deleted repository files are removed from local state before the next rule set is loaded. It should also account for failed synchronisation, path validation, and concurrent updates.

### 10.6 Webhook dependency boundaries

The webhook depends on the VCS interface and the rule runner, supplied through FastAPI dependency injection.

This keeps GitHub API behaviour separate from request-handling logic and makes the webhook easier to test.

The route is responsible for deciding whether a repository event is relevant, while the VCS client handles repository operations and signature validation.

---

## 11. API Surface

The provisional API inventory includes the following endpoints.

| Method | Route              | Responsibility                          |
| ------ | ------------------ | --------------------------------------- |
| `POST` | `/webhooks/github` | Receive GitHub webhook events.          |
| `POST` | `/events`          | Accept events for detection processing. |
| `GET`  | `/rules`           | List available detection rules.         |
| `POST` | `/rules`           | Write or submit a detection rule.       |

The GitHub webhook route is confirmed by the supplied implementation. The other routes are taken from the provisional notes and should be verified against the current router definitions before being treated as an authoritative API contract.

Application endpoints are intended to require JWT bearer-token authentication, with explicitly public endpoints such as the GitHub webhook excluded from that requirement.

Public access to the webhook must not mean unauthenticated trust: the webhook has its own signature-validation boundary.

Request and response models, authentication requirements, expected status codes, and error payloads should be documented for each endpoint.

---

## 12. Security Architecture

Security controls operate at different boundaries and serve different purposes.

### 12.1 JWT bearer-token authentication

The provisional design uses a JWT bearer token to protect application endpoints.

The token is decoded and validated using a configured secret and the HS256 algorithm.

The implementation should explicitly enforce the expected signing algorithm and validate relevant claims, including expiration and any required issuer, audience, or subject constraints.

JWT validation authenticates a request under the application's token model. Authorisation — deciding what an authenticated user may do — is a separate concern.

The shared signing secret must be supplied through secure runtime configuration and must never be committed to the repository.

### 12.2 GitHub webhook authentication

Webhook authentication uses HMAC SHA-256 signature verification against the raw request body.

This is independent of JWT authentication. GitHub webhook delivery should be validated using the configured webhook secret rather than requiring a user bearer token.

### 12.3 GitHub API credentials

The VCS client requires appropriate repository permissions to read and write detection content.

Credentials should follow the principle of least privilege, be managed outside source control, and be scoped to the operations the service actually needs.

The required permissions differ depending on whether the service reads content only, commits directly, or creates pull requests.

### 12.4 Input validation and path safety

Repository paths and API-provided rule content are inputs and must be validated.

For filesystem synchronisation, path resolution should ensure that the resulting local path remains within the configured detection directory. This is important even when paths originate from authenticated webhook payloads.

Rule validation should reject malformed schemas and unsupported values before a rule becomes active.

### 12.5 Error handling and observability

The webhook logs relevant processing activity and returns HTTP 500 when synchronisation or reload fails.

Logs should provide enough context to troubleshoot failures without exposing secrets, bearer tokens, webhook signatures, or unnecessarily sensitive event data.

Where useful, structured logs and metrics should distinguish signature failures, ignored events, repository API errors, rule validation failures, reload failures, and alert publication failures.

### 12.6 Public documentation boundary

The public architecture document describes security mechanisms and trust boundaries without publishing actual credentials or private deployment details.

Configuration variable names and example schemas may be documented, but real secrets, private endpoints, repository credentials, and sensitive operational values must remain outside the public repository.

---

## 13. Testing Strategy

The engine is designed to support independent testing of detection rules and application components.

### 13.1 Rule evaluation tests

Representative fixtures provide normalised events for different categories.

Example test structure:

```text
tests/
    fixtures/
        auth_events.json
        network_events.json
    test_rules.py
```

Rule tests should verify:

- Matching events trigger the expected rule.
- Non-matching events do not trigger the rule.
- Individual operators behave correctly.
- Context fields are extracted correctly.
- Aggregation thresholds are applied over the expected time window.
- Disabled rules are not evaluated.
- Invalid rules are rejected according to the loader's failure policy.

The `tested` field should have a defined meaning. A reliable approach is to update it through an explicit validation workflow rather than assuming that the existence of a test or a successful production run proves the rule has passed its intended checks.

### 13.2 Component isolation

Dependency injection supports unit testing without requiring every external service to be available.

Examples include:

- Mocking the VCS client when testing webhook event filtering.
- Mocking the alert gateway when testing matching and context extraction.
- Testing rule loading against temporary directories.
- Testing GitHub API response handling independently of the engine.
- Testing JWT validation separately from route business logic.

### 13.3 Webhook tests

Webhook tests should cover:

- Invalid signatures.
- Valid signatures and unsupported event types.
- Pushes to the wrong branch.
- Pushes without relevant detection changes.
- Added and modified rule files.
- Deleted rule files.
- Repository API failures.
- Partial file synchronisation failures.
- Rule-loader failures.
- Duplicate deliveries and repeated updates.

These tests are especially important because repository synchronisation changes the active detection behaviour of the running service.

### 13.4 Concurrency and reliability tests

Additional tests should cover event processing during reload, simultaneous webhook deliveries, repeated aggregation events, and failure recovery.

The objective is to establish whether the service preserves a consistent active rule set and predictable aggregation behaviour under concurrent activity.

---

## 14. Architectural Decisions

### ADR-001: YAML for detection definitions

**Decision:** Represent detection rules using YAML.

**Rationale:** YAML is human-readable, supports declarative configuration, integrates with version control, and allows analysts to review rule changes without implementing the evaluation logic in Python.

**Consequences:** The rule schema must be validated, versioned, and maintained as a stable contract. Schema evolution requires compatibility planning.

### ADR-002: Python for the detection engine

**Decision:** Implement the rule loader, matcher, runner, and publisher in Python.

**Rationale:** Python provides suitable parsing, validation, and asynchronous HTTP capabilities, while Pydantic models make the rule and event contracts explicit.

**Consequences:** Runtime performance, concurrency behaviour, and resource usage must be evaluated as event volume and rule complexity increase.

### ADR-003: Normalised events

**Decision:** Evaluate detections against a common event model rather than raw source-specific log formats.

**Rationale:** This reduces coupling between detections and individual log sources.

**Consequences:** Parsers and schema mappings become critical dependencies. Changes to the canonical event model require careful compatibility management.

### ADR-004: Separate matching and publishing

**Decision:** Keep rule evaluation separate from alert delivery.

**Rationale:** Detection logic can be tested independently of the downstream alert gateway, and publication can evolve without rewriting the matcher.

**Consequences:** The boundary between matching results and published alerts needs a stable contract, including error handling and delivery semantics.

### ADR-005: VCS abstraction

**Decision:** Define a common VCS interface and provide a GitHub implementation.

**Rationale:** Repository operations remain independent of the HTTP route layer, and tests can substitute mock implementations.

**Consequences:** The interface must remain provider-neutral where practical, and provider-specific behaviours must be handled within the implementation.

### ADR-006: GitHub webhooks instead of polling

**Decision:** Use webhook notifications to trigger detection-content synchronisation.

**Rationale:** Push-based notifications avoid continuous polling and provide a direct link between repository changes and runtime rule updates.

**Consequences:** The service must handle duplicate delivery, transient failures, retries, repository unavailability, and state reconciliation.

### ADR-007: Full rule reload after relevant changes

**Decision:** Reload the complete detection directory after synchronising relevant changed files.

**Rationale:** Full loading keeps rule interpretation centralised in the loader and avoids maintaining a separate incremental-loading implementation.

**Consequences:** Reload cost grows with the rule set. File deletion reconciliation and reload consistency must be handled explicitly.

### ADR-008: In-memory aggregation

**Decision:** Keep time-window aggregation state in memory within the running service.

**Rationale:** This avoids an external state dependency and provides a straightforward implementation for time-based detection.

**Consequences:** Aggregation state is lost on restart and is not automatically shared across multiple workers or instances. Concurrency and memory limits require consideration.

### ADR-009: Direct commit and pull-request strategies

**Decision:** Support direct commits and pull-request-based rule updates through the VCS layer.

**Rationale:** Different environments may require different governance levels, from simple automated updates to mandatory human review.

**Consequences:** The selected strategy must align with repository permissions and the desired review boundary. Pull-request workflows require careful handling of existing branches and repeated submissions.

### ADR-010: JWT authentication for application endpoints

**Decision:** Protect application endpoints with JWT bearer-token authentication, except explicitly public endpoints such as the GitHub webhook.

**Rationale:** Authentication restricts access to application functionality, while webhook signatures provide a separate trust mechanism for GitHub-originated notifications.

**Consequences:** Token validation, secret management, endpoint authorisation, and public-route exceptions must be maintained consistently.

---

## 15. Current Limitations and Reliability Considerations

The following areas are either established limitations of the described design or behaviours that require confirmation against the implementation.

| Area                    | Current understanding                                                   | Recommended direction                                                                              |
| ----------------------- | ----------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| Rule reload             | Existing rules are modified in place.                                   | Consider loading and validating a replacement set before switching active rules.                   |
| Aggregation state       | Stored in process memory.                                               | Define restart behaviour and assess whether shared storage is needed for multi-worker deployments. |
| Concurrent reloads      | Safe transition is not established by the supplied notes.               | Add explicit synchronisation and test concurrent event processing.                                 |
| File deletions          | The shown synchronisation function does not remove deleted local files. | Implement repository-to-local reconciliation.                                                      |
| Partial synchronisation | Files are written before the full reload.                               | Stage changes, validate the complete set, and define rollback or recovery behaviour.               |
| Webhook retries         | Failures return HTTP 500.                                               | Establish idempotent processing and explicit retry/recovery expectations.                          |
| Duplicate deliveries    | No deduplication mechanism is established by the supplied code.         | Evaluate delivery identifiers, repeated commits, and concurrent requests.                          |
| Alert delivery          | Publisher exists as a separate component.                               | Define timeouts, retries, delivery guarantees, and duplicate handling.                             |
| Rule validation         | Pydantic validation is part of the design.                              | Define whether invalid rules prevent all loading or are isolated.                                  |
| Rule effectiveness      | Effectiveness fields are described in the schema.                       | Implement and validate metrics based on triage outcomes before claiming effectiveness tracking.    |

These considerations do not invalidate the current architecture. They identify the operational guarantees that should be established as the service evolves.

---

## 16. Planned Enhancements

The following capabilities are part of the roadmap rather than established features of the current implementation.

### 16.1 Rule effectiveness tracking

Track detection performance using triage outcomes and false-positive measurements.

This could help identify noisy rules and prioritise tuning. Metrics should use a documented methodology, sufficient sample sizes, and clearly defined outcome categories.

### 16.2 Detection recommendation engine

Analyse historical event patterns to suggest candidate detections.

Recommendations should be treated as hypotheses that require validation rather than automatically promoted to production rules.

### 16.3 Sigma compatibility

Import and convert compatible Sigma rules into the service's internal schema.

This requires mapping Sigma log-source definitions, selections, condition expressions, modifiers, and supported features to the engine's capabilities. Unsupported semantics should be reported rather than silently discarded.

### 16.4 AI-assisted rule generation

Use a language model to propose candidate detection rules from observed patterns or analyst-provided requirements.

Generated rules should pass schema validation, test fixtures, security review, and the established change-control process before deployment.

### 16.5 Safer rule deployment

Strengthen the reload process with staged synchronisation, complete validation, and controlled activation of the replacement rule set.

This would reduce the risk of a failed or incomplete repository update affecting active detection behaviour.

---

## 17. Glossary

| Term                | Definition                                                                                                    |
| ------------------- | ------------------------------------------------------------------------------------------------------------- |
| Detection-as-Code   | Managing security detections as version-controlled, testable definitions.                                     |
| Detection rule      | A declarative definition describing which events should trigger a detection.                                  |
| Normalised event    | A common event representation independent of the original log format.                                         |
| Matcher             | Component that evaluates rule conditions against event data.                                                  |
| Runner              | Component that orchestrates rule evaluation and associated state.                                             |
| Aggregation window  | Time interval over which related events are counted or grouped.                                               |
| Indicator           | An entity extracted from a detection, such as an IP address or domain.                                        |
| Context             | Additional event information attached to an alert.                                                            |
| VCS                 | Version-control system used to store and manage detection definitions.                                        |
| Webhook             | An HTTP notification delivered when a configured event occurs.                                                |
| HMAC                | A message authentication mechanism using a cryptographic hash and shared secret.                              |
| JWT                 | A token format used to carry claims between parties.                                                          |
| Reconciliation      | Bringing local state into agreement with the authoritative repository state.                                  |
| Atomic-style reload | Preparing a replacement rule set before switching active rules, rather than exposing partially updated state. |

---

## 18. Conclusion

The Detection-as-Code service establishes a modular approach to security detection engineering.

Its architecture separates declarative rule definitions from the engine that evaluates them, normalises event data to reduce source-specific coupling, and isolates version-control operations behind a dedicated abstraction.

GitHub integration introduces a version-controlled workflow for detection changes, while webhook-driven synchronisation connects repository activity to runtime rule management. Authentication, schema validation, component isolation, and testable interfaces provide foundations for a maintainable security service.

The most important next steps are to make repository reconciliation complete, strengthen reload consistency, define aggregation behaviour under concurrency, and establish explicit operational guarantees for webhook processing and alert publication.

These improvements would build on the existing architecture while making its consistency, security, and reliability properties more explicit and defensible.
