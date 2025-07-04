# Universal Predictive Signal Standard (UPSS) — v0.3 DRAFT

## Why another standard?

Predictive pipelines rarely share a common, machine-verifiable format for "signals" (time-stamped numerical observations plus the context needed to trust and govern them).

Without one, every team must re-negotiate:

- field names and units (`temp_c` vs `temperature`)
- provenance rules ("where did this value come from?")
- contract testing ("is a missing value null, NaN, -9999?")

UPSS 0.3 answers those headaches by defining one minimal, self-describing schema that travels over any transport (files, streams, APIs) and is validated the same way everywhere. Existing middleware can route it because UPSS re-uses the CloudEvents envelope; domain tools can consume it because the payload is strict, typed, and carries the metadata that models, auditors and privacy officers all need.

## 1. Scope & Non-Goals

- **In-scope**: the logical data model for a single signal record or batch of homogeneous records.
- **Out-of-scope**: prescriptive choices of broker, database, codec, or partitioning strategy. Those appear only as non-normative guidance in an appendix.

## 2. Core Data Model

A UPSS record is a CloudEvent whose data field contains either:

(a) one JSON object representing a single observation, or  
(b) a binary columnar blob (Parquet/Arrow) containing many observations.

### 2.1 Required top-level CloudEvent attributes

| Attribute | Example | Purpose |
|-----------|---------|---------|
| `id` | `f47ac10b-58cc-4372…` | Globally unique per event (dedupe) |
| `source` | `scada://plantA/line4` | Origin of the signal |
| `type` | `io.upss.signal.v0.json` | Routing & policy |
| `specversion` | `1.0` | CloudEvents version |
| `time` | `2025-04-16T14:30:00Z` | Creation time of this envelope |
| `datacontenttype` | `application/upss+json;v=0.3` | Media-type of `data` payload |
| `dataschema` | `https://upss.io/schema/0.3.json` | Machine-readable schema for validation |

*(Additional CloudEvents extension attributes are allowed but must follow CE naming rules.)*

### 2.2 Payload schema (data)

| # | Field | Type | Req.? | Description / Allowed values |
|---|-------|------|-------|------------------------------|
| 1 | `timestamp` | string | ✔ | ISO-8601 UTC (`YYYY-MM-DDThh:mm:ss[.ffffff]Z`) |
| 2 | `sensor_id` | string | ✔ | Producer-controlled identifier of the measurement source |
| 3 | `location` | string | ✔ | ISO-3166-1 α-2 + free sub-location (`US/NYC/Warehouse-3`) |
| 4 | `unit` | string | ✔ | UCUM code (`Cel`, `kg`, `%`, …); `"1"` for dimensionless |
| 5 | `value` | number\|null | ✔ | 64-bit float, null if missing |
| 6 | `data_source` | string | ✔ | System-of-record or pipeline ID |
| 7 | `provenance_id` | string | ✔ | UUID v4 identifying upstream artefact / ETL run (`6f125c08-3c22-4a0e-b6b7-13e58a48d9d1`) |
| 8 | `schema_version` | string | ✔ | Must equal `UPSS-0.3` |
| 9 | `classification` | string | ✔ | `PUBLIC`, `INTERNAL`, `CONFIDENTIAL`, `RESTRICTED` |
| 10 | `quality` | string | ✖ | `OK`, `MISSING`, `SUSPECT`, `ESTIMATED` |
| 11 | `uncertainty` | number | ✖ | ±1 σ in same unit as `value` |
| 12 | `pii` | boolean | ✖ | `true` if record contains PII |
| 13 | `tags` | object | ✖ | Additional key–value pairs (`string:string`) |

**Note**: No other top-level properties are allowed (`additionalProperties: false`). Extensions must live under `tags`.

## 3. Versioning & Extension Rules

- **Envelope stability**: UPSS leverages CloudEvents `specversion`; that changes only when CloudEvents itself does.
- **Payload stability**: Breaking changes to §2.2 fields require bumping `schema_version` → UPSS-1.0.
- New optional fields are added inside `tags`; once widely adopted they may be "promoted" in the next major version.

## 4. Minimal Binding Profiles (non-normative)

| Profile | Notes |
|---------|-------|
| **HTTP-binary** | Each record in its own request/response; CE attributes in headers, payload in body. Good for webhooks & small volumes. |
| **Kafka** | CE attributes in headers (`ce-id`, …); payload in message value. Use a compacted topic keyed by `sensor_id`. |
| **JSON Lines** | For file-based interchange; one CloudEvent per line. |
| **Parquet batch** | CE envelope wraps a Snappy-compressed Parquet file; preferable for >10 M rows. |

## 5. Illustrative Single-Record Example (JSON)
```json
{
  "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "source": "scada://plantA/line4",
  "type": "io.upss.signal.v0.json",
  "specversion": "1.0",
  "time": "2025-04-16T14:30:05Z",
  "datacontenttype": "application/upss+json;v=0.3",
  "dataschema": "https://upss.io/schema/0.3.json",
  "data": {
    "timestamp":   "2025-04-16T14:30:00Z",
    "sensor_id":   "thermo-42",
    "location":    "US/NYC/Warehouse-3",
    "unit":        "Cel",
    "value":       23.4,
    "quality":     "OK",
    "uncertainty": 0.1,
    "data_source": "iot-gateway-7",
    "provenance_id":"6f125c08-3c22-4a0e-b6b7-13e58a48d9d1",
    "schema_version": "UPSS-0.3",
    "classification":"INTERNAL",
    "tags": {
      "shift": "B",
      "calibration_due": "2025-07-01"
    }
  }
}
```

## Appendix A: Operational Guidance (optional)

- **Compression** – Use Snappy (Parquet) or ZSTD (Arrow) for columnar batches.
- **Partitioning** – Daily folders by UTC date; optional hash-sub-folders by `sensor_id`.
- **Late data** – Accept records up to 24 h late; deduplicate by (`sensor_id`, `timestamp`) using `id` precedence.
- **Security** – If `classification` ≥ CONFIDENTIAL, transmit over TLS 1.2+ and store in encrypted volumes.

*(The above are recommendations, not part of the normative standard.)*