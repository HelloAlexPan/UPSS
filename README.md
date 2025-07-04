# Universal Predictive Signal Standard (UPSS) v0.2

A strict, machine-verifiable format for exchanging time-stamped numerical observations plus the metadata modern predictive models require.

## 1. Purpose
- Guarantee that any producer can publish a time-series record that any consumer can parse, validate and use in a model pipeline without an out-of-band contract.
- Preserve accuracy (units, precision, quality flags), consistency (canonical field names & types), scalability (partitioning rules, columnar layouts) and interoperability (JSON, CSV, Parquet, Arrow, SQL/TSDB).
- Embed full lineage, security and compliance metadata.

## 2. Model Requirements
### 2.1. Accuracy
- ISO-8601 UTC timestamps (≥ 1 s precision, optional µs).
- Explicit measurement units & value type.
- Optional uncertainty & quality flag.

### 2.2. Consistency
- Canonical field names & datatypes (strict schema).
- Reference data dictionaries for units, locations.

### 2.3. Scalability
- Support billions of rows via partition keys, compression, stream chunking.

### 2.4. Interoperability
- At minimum: CSV & JSON lines (single-row records), and columnar (Parquet/Arrow) using identical field order & names.
- JSON-Schema & Apache Avro IDs.

### 2.5. Security / Privacy
- Mandatory classification tag, optional encryption hint.
- Per-record PII flag.

## 3. Data Structure Design
### 3.1 Field Specification

| # | Field | Type | Mode | Description / Allowed values |
|---|-------|------|------|------------------------------|
| 1 | timestamp | string | REQUIRED | ISO 8601 UTC (`YYYY-MM-DDThh:mm:ss[.ffffff]`) |
| 2 | sensor_id | string | REQUIRED | Stable identifier of measurement source (UUID, URI, or integer in string) |
| 3 | location | string | REQUIRED | ISO 3166-1 alpha-2 + freeform sub-location (`US/NYC/Warehouse-3`) |
| 4 | measurement_unit | string | REQUIRED | UCUM code (`Cel`, `kg`, `%` …); "dimensionless" allowed |
| 5 | value | number | REQUIRED | 64-bit floating point |
| 6 | quality_flag | string | OPTIONAL | Enum: `OK`, `MISSING`, `SUSPECT`, `ESTIMATED` |
| 7 | uncertainty | number | OPTIONAL | ±1 σ expressed in same unit as value |
| 8 | data_source | string | REQUIRED | System-of-record or pipeline ID (`scada://plantA/line4`) |
| 9 | provenance_id | string | REQUIRED | UUID v4 tying this row to upstream event/ETL job |
| 10 | schema_version | string | REQUIRED | `UPSS-0.2` |
| 11 | pii_flag | boolean | OPTIONAL | true if record contains PII |
| 12 | classification | string | REQUIRED | Enum: `PUBLIC`, `INTERNAL`, `CONFIDENTIAL`, `RESTRICTED` |
| 13 | tags | object | OPTIONAL | Arbitrary key–value pairs (string:string) |

All additional keys MUST reside under `tags` to keep the top-level schema stable.

### 3.2 Timestamp Format
`2024-01-01T12:00:00Z` (sec) or `2024-01-01T12:00:00.123456Z` (µs). Always UTC.

## 4. Data Handling Guidelines
### 4.1 Ingestion & Cleansing
- Validate JSON-Schema (or Avro) on arrival; reject or quarantine non-conforming rows.
- Deduplicate by (sensor_id, timestamp); keep the record with highest quality_flag precedence (OK> ESTIMATED> SUSPECT> MISSING).
- Out-of-order arrivals allowed within a configurable "lateness" window (default 1 day); late but valid rows are merged.
### 4.2 Error Handling
- If value is unparseable, set quality_flag = "MISSING" and value = null.
- If timestamp invalid → entire record to dead-letter queue.
- Unknown measurement_unit → map via unit dictionary; on failure, reject.
### 4.3 Normalisation
- Units: convert to canonical unit per sensor (e.g. Fahrenheit → Celsius).
- Scaling: optional Min-Max or z-score stored in feature store; do not overwrite raw value.
- Missing gaps: fill only in model-specific feature pipelines, not in raw UPSS tables.

## 5. Exchange and Storage Formats
### 5.1 CSV (UTF-8)
Ordered columns exactly as listed in §3.1 (timestamp … tags).

Example row (line breaks added for clarity):

`2025-04-16T14:30:00Z,thermo-42,US/NYC/Warehouse-3,Cel,23.4,OK,,iot-gateway-7,6f125...,UPSS-0.2,false,INTERNAL,"{""shift"":""B""}"`

### 5.2 JSON Lines
Each line is a full record:

```json
{
  "timestamp":"2025-04-16T14:30:00Z",
  "sensor_id":"thermo-42",
  "location":"US/NYC/Warehouse-3",
  "measurement_unit":"Cel",
  "value":23.4,
  "quality_flag":"OK",
  "data_source":"iot-gateway-7",
  "provenance_id":"6f125c08-3c22-4a0e-b6b7-13e58a48d9d1",
  "schema_version":"UPSS-0.2",
  "classification":"INTERNAL",
  "tags":{"shift":"B"}
}
```
```json
{
  "$schema":"https://json-schema.org/draft/2020-12/schema",
  "$id":"https://upss.io/schema/0.2.json",
  "type":"object",
  "required":["timestamp","sensor_id","location","measurement_unit","value","data_source","provenance_id","schema_version","classification"],
  "properties":{
    "timestamp":{"type":"string","format":"date-time"},
    "sensor_id":{"type":"string","minLength":1},
    "location":{"type":"string"},
    "measurement_unit":{"type":"string","minLength":1},
    "value":{"type":["number","null"]},
    "quality_flag":{"type":"string","enum":["OK","MISSING","SUSPECT","ESTIMATED"]},
    "uncertainty":{"type":"number"},
    "data_source":{"type":"string"},
    "provenance_id":{"type":"string","format":"uuid"},
    "schema_version":{"type":"string","const":"UPSS-0.2"},
    "pii_flag":{"type":"boolean"},
    "classification":{"type":"string","enum":["PUBLIC","INTERNAL","CONFIDENTIAL","RESTRICTED"]},
    "tags":{"type":"object","additionalProperties":{"type":"string"}}
  },
  "additionalProperties":false
}
```

### 5.3 Columnar / Binary
- Parquet or Arrow using same field names; Snappy (Parquet) or ZSTD (Arrow) compression.
- Partition keys: `p_date = DATE(timestamp)`, `sensor_id_hash = hash32(sensor_id) mod 128`.

### 5.4 Time-Series Databases
- Table DDL (TimescaleDB / PostgreSQL):

```sql
CREATE TABLE UPSS (
  timestamp        TIMESTAMPTZ NOT NULL,
  sensor_id        TEXT        NOT NULL,
  location         TEXT        NOT NULL,
  measurement_unit TEXT        NOT NULL,
  value            DOUBLE PRECISION,
  quality_flag     TEXT,
  uncertainty      DOUBLE PRECISION,
  data_source      TEXT NOT NULL,
  provenance_id    UUID NOT NULL,
  schema_version   TEXT NOT NULL,
  pii_flag         BOOLEAN,
  classification   TEXT NOT NULL,
  tags             JSONB
);

SELECT create_hypertable('UPSS','timestamp',chunk_time_interval => INTERVAL '1 day');
CREATE INDEX ON UPSS (sensor_id, timestamp DESC);
```

## 6. Scalability Considerations
### 6.1 Partitioning
- Daily (or hourly for >10 M rows/day) time partitions.
- Hash sub-partition on sensor_id to avoid hotspotting.

### 6.2 Compression
- Delta + Gorilla codec in TSDBs; Snappy/ZSTD for Parquet; dictionary encoding for location, quality_flag.

### 6.3 Streaming
- Kafka topic UPSS.v1.raw with max 1 MB messages; use exactly-once semantics.
– Schema fingerprint embedded in Kafka message headers for fast validation.

### 6.4 Indexing
- Composite (sensor_id, timestamp) B-tree or skip-list index.
- Bloom filters on Parquet row-groups for sensor_id.

### 6.5 Query patterns
- "Last-N" look-ups: clustered index by (sensor_id, timestamp DESC) covers sliding-window prediction.
- Aggregation jobs should leverage predicate push-down and column pruning.

## 7. Examples
### 7.1 Normal Row

| timestamp | sensor_id | location | measurement_unit | value | quality_flag | uncertainty | data_source | provenance_id | schema_version | pii_flag | classification | tags |
|-----------|-----------|----------|------------------|-------|--------------|-------------|-------------|---------------|----------------|----------|----------------|------|
| 2025-04-16T14:30:00Z | thermo-42 | US/NYC/Warehouse-3 | Cel | 23.4 | OK | 0.1 | iot-gateway-7 | 6f125c08-3c22-4a0e-b6b7-13e58a48d9d1 | UPSS-0.2 | false | INTERNAL | {"shift":"B"} |

### 7.2 Missing Value
`2025-04-16T14:31:00Z,thermo-42,US/NYC/Warehouse-3,Cel,,MISSING,,iot-gateway-7,33e9...,UPSS-0.2,false,INTERNAL,"{}"`

### 7.3 Corrupted Timestamp (rejected)
`2025/04/16 14:32,thermo-42,US/NYC/Warehouse-3,Cel,23.6,OK,,iot-gateway-7,beef...,UPSS-0.2,false,INTERNAL,"{}"`

→ Sent to dead-letter; error code `INVALID_TIMESTAMP`.

## 8. Notes

- Time-zone discrepancies: always converted to UTC prior to publication; original zone can be stored in tags.original_tz.
- Security & privacy: if classification ≥ CONFIDENTIAL, the transport layer MUST be encrypted (TLS 1.2+).
- Extensibility: new optional fields MUST be placed under tags; a future MAJOR version may promote stable ones.
- Governance: change-data-capture of the UPSS table is registered with OpenLineage; provenance_id links back to source artefacts.
