# Threat Intel Enrichment Service

A FastAPI service that enriches indicators of compromise against multiple threat intelligence sources concurrently, returning a structured result with confidence scoring and explainability signals.

---

## Responsibilities

- Accept an indicator (IP address, domain, hash, URL) via HTTP
- Fan out enrichment requests to multiple sources concurrently
- Normalise responses into a common signal schema
- Calculate a weighted confidence score across all sources
- Cache results in Redis to preserve API rate limits
- Return a structured `IntelResult` with signals, reasons, and risk level

---

## Endpoints

```
GET /v1/intel/ip/{indicator}
GET /v1/intel/domain/{indicator}    (planned)
GET /v1/intel/hash/{indicator}      (planned)
GET /v1/intel/url/{indicator}       (planned)

GET /health
```

---

## Enrichment sources

| Source     | Indicator types       | Key signals                                                  |
| ---------- | --------------------- | ------------------------------------------------------------ |
| AbuseIPDB  | IP                    | Abuse confidence score, Tor exit node, report count          |
| VirusTotal | IP, domain, hash, URL | Malicious engine count, suspicious engines, tags, reputation |
| GreyNoise  | IP                    | Internet scanning activity, classification, RIOT membership  |

Sources are isolated — each lives in its own module and returns a typed tuple of `(parsed_response, signals, reasons)`. A failed source is recorded in `sources_failed` and the pipeline continues with remaining sources.

---

## Confidence scoring

Each signal carries a weight between 0 and 1. Signal values are normalised to a 0–1 scale before weighting:

- Boolean signals (e.g. `is_tor`) contribute their full weight when true
- Score signals (e.g. `abuse_confidence_score: 100`) are divided by 100
- Reputation signals with negative scales are inverted before normalisation

The final confidence is the weighted average across all signals from all sources.

```
confidence = sum(normalised_value × weight) / sum(weights)
```

Risk level is derived from confidence:

| Confidence | Risk level |
| ---------- | ---------- |
| >= 0.8     | critical   |
| >= 0.6     | high       |
| >= 0.35    | medium     |
| > 0        | low        |
| 0          | unknown    |

---

## Caching

Results are cached in Redis with a 1-hour TTL keyed by `intel:ip:{indicator}`. Cache hits are flagged in the response with `"cached": true`. Cache failures degrade gracefully — the request falls through to live API calls.

The caching layer exists specifically to protect third-party API rate limits, not for performance. AbuseIPDB, VirusTotal, and GreyNoise all have request quotas on free and paid tiers.

---

## Response shape

```json
{
  "indicator": "185.220.101.1",
  "indicator_type": "ip",
  "risk_level": "critical",
  "confidence": 1.0,
  "signals": [
    {
      "source": "abuseipdb",
      "field": "abuse_confidence_score",
      "value": 100,
      "weight": 0.6,
      "raw": { "...full source response..." }
    }
  ],
  "reasons": [
    "IP has high abuse confidence score (100%)",
    "IP is a known Tor exit node",
    "Classified as malicious by GreyNoise"
  ],
  "sources_queried": ["abuseipdb", "virustotal", "greynoise"],
  "sources_failed": [],
  "cached": false,
  "queried_at": "2026-04-11T10:00:00Z",
  "cache_ttl_seconds": 3600
}
```

---

## Architecture decisions

**Concurrent source fan-out** — all three sources are called simultaneously using `asyncio.gather`. On real network conditions this reduces enrichment latency from ~1.5s sequential to ~0.5s concurrent.

**Source isolation** — each source module is independently testable and replaceable. Adding a new source means adding one file and registering it in the orchestrator. Nothing else changes.

**Typed response models** — each source has a Pydantic response model with field aliases matching the API's camelCase output. Validation failures are caught at parse time, not silently downstream.

**Raw signal preservation** — the first signal from each source includes the full raw API response. This provides a complete audit trail without requiring a separate lookup.
