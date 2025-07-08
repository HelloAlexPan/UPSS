# Universal Predictive Signal Standard (UPSS) — v0.4 DRAFT

## Why another standard?

Predictive pipelines rarely share a common, machine-verifiable format for "predictive signals" (time-stamped numerical observations plus the context needed to trust and govern them).

Without one, every team must re-negotiate:

- field names and units (`temp_c` vs `temperature`)
- provenance rules ("where did this value come from?")
- contract testing ("is a missing value null, NaN, -9999?")

UPSS solves these headaches by providing a minimal, self-describing schema that travels over any transport and validates the same way everywhere. Reusing CloudEvents, existing middleware can route it whilst carrying strict, typed metadata for models, audits, and governance for consumption by predictive models, auditors, and privacy officers.

## 1. Core Data Model

A UPSS record is a CloudEvent whose data field contains either:

(a) one JSON object representing a single observation, or  
(b) a binary columnar blob (Parquet/Arrow) containing many observations.

### 1.1 Required top-level CloudEvent attributes

| Attribute | Example | Purpose |
|-----------|---------|---------|
| `id` | `f47ac10b-58cc-4372…` | Globally unique per event (dedupe) |
| `source` | `com.acme:extractor-v1` | Identifies the context where the signal originated (e.g., the application, service, or domain). MUST be a namespaced URI. |
| `type` | `io.upss.signal.v0.4` | Routing & policy |
| `specversion` | `1.0` | CloudEvents version |
| `time` | `2025-06-15T10:05:05Z` | Creation time of this envelope |
| `datacontenttype` | `application/upss+json;v=0.4` | Media-type of `data` payload |
| `dataschema` | `https://upss.io/schema/0.4.json` | Machine-readable schema for validation |

*(Additional CloudEvents extension attributes are allowed but must follow CE naming rules.)*

### 1.2 Payload schema (data)

| # | Field | Type | Req.? | Description / Allowed values |
|---|-------|------|-------|------------------------------|
| 1 | `timestamp` | string | ✔ | ISO-8601 UTC (`YYYY-MM-DDThh:mm:ss[.ffffff]Z`) |
| 2 | `stream_id` | string | ✔ | Producer-controlled identifier for the logical signal stream. This groups related time series data. MUST be namespaced (see §2.3). |
| 3 | `unit` | string | ✔ | UCUM code (`Cel`, `kg`, `%`, …); `"1"` for dimensionless |
| 4 | `value` | number\|null | ✔ | 64-bit float, null if missing |
| 5 | `provenance_id` | string | ✔ | UUID v4 identifying upstream artefact / ETL run (`6f125c08-3c22-4a0e-b6b7-13e58a48d9d1`) |
| 6 | `schema_version` | string | ✔ | Must equal `UPSS-0.4` |
| 7 | `classification` | string | ✔ | `PUBLIC`, `INTERNAL`, `CONFIDENTIAL`, `RESTRICTED` |
| 8 | `subject` | string | ✖ | An identifier for the primary entity that this signal's value describes. This provides a more granular context than the top-level `source`. SHOULD be a unique, namespaced identifier (see §2.3), e.g., `com.salesforce.acme-crm:001` |
| 9 | `quality` | string | ✖ | `OK`, `MISSING`, `SUSPECT`, `ESTIMATED` |
| 10 | `uncertainty` | number | ✖ | ±1 σ in same unit as `value` |
| 11 | `pii` | boolean | ✖ | `true` if record contains PII |
| 12 | `context_text` | string | ✖ | Raw text that generated the signal (e.g., call transcript snippet). |
| 13 | `event_type` | string | ✖ | Semantic descriptor for the signal (e.g., `churn_risk_detected`). |
| 14 | `entity_references` | array[object] | ✖ | An array of objects, each identifying a linked entity. Each object MUST contain a `type` (string) and an `id` (string). The `id` value SHOULD be namespaced (see §2.3). |
| 15 | `tags` | object | ✖ | Additional key–value pairs for application-specific metadata. Note: For interoperability with generic CloudEvents tooling, standardized extensions SHOULD be placed at the top level. This `tags` field is for payload-specific context. |

**Note**: No other top-level properties are allowed (`additionalProperties: false`). Extensions must live in `tags`.

****

### 1.3 Guidance on Namespacing

To ensure signals are globally unique and to prevent collisions in federated environments, key identifiers like `stream_id` and `entity_references` SHOULD be namespaced.

The recommended best practice is to use Reverse Domain Name Notation as the namespace prefix.

- **Tier 1 (Recommended): Reverse Domain Name**. Use your owned domain in reverse (e.g., `com.example` for `example.com`). Example: `com.example:thermo-42`.
- **Tier 2 (Acceptable): Public Repository**. For OSS projects or individuals, a unique public repository URL format like `com.github.user.project` can be used.
- **Tier 3 (Local Use Only): Simple Name**. For local development, a simple name like `my-project` is acceptable but must be updated before integrating with a hosted or third-party system.


## 2. Versioning & Extension Rules

- **Envelope stability**: UPSS uses CloudEvents `specversion` and changes only when CloudEvents does.
- **Payload stability**: Breaking changes to §2.2 fields require bumping `schema_version` → UPSS-1.0.
- New optional fields go in `tags`; widely adopted ones may be promoted in future versions of UPSS.

## 3. Minimal Binding Profiles (non-normative)

| Profile | Notes |
|---------|-------|
| **HTTP-binary** | Each record in its own request/response; CE attributes in headers, payload in body. Good for webhooks & small volumes. |
| **Kafka** | CE attributes in headers (`ce-id`, …); payload in message value. Use a compacted topic keyed by `stream_id`. |
| **JSON Lines** | For file-based interchange; one CloudEvent per line. |
| **Parquet batch** | CE envelope wraps a Snappy-compressed Parquet file; preferable for >10 M rows. |

## 4. Illustrative Single-Record Example (JSON)
```json
{
  "id": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
  "source": "com.example.supportdesk",
  "type": "io.upss.signal.v0.4",
  "specversion": "1.0.2",
  "time": "2025-06-15T10:05:05Z",
  "datacontenttype": "application/upss+json;v=0.4",
  "dataschema": "https://upss.io/schema/0.4.json",
  "data": {
    "timestamp": "2025-06-15T10:05:00Z",
    "stream_id": "com.example.supportdesk:sentiment-scores",
    "subject": "com.example.crm:customer-12345",
    "unit": "1",
    "value": -0.85,
    "provenance_id": "7a8b9c0d-1e2f-3a4b-5c6d-7e8f9a0b1c2d",
    "schema_version": "UPSS-0.4",
    "classification": "CONFIDENTIAL",
    "pii": true,
    "context_text": "I am very frustrated with the latest update and am considering canceling.",
    "event_type": "churn_risk_signal",
    "entity_references": [
      { "type": "support_ticket", "id": "com.example.supportdesk:TICK-9876" }
    ],
    "tags": {
      "model_version": "2.1.3"
    }
  }
}
```
