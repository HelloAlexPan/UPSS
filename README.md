# Universal Predictive Signal Standard (UPSS) Specification

**Version 0.1**

The Universal Predictive Signal Standard (UPSS) is a vendor-neutral specification for defining the format of predictive signals, especially those derived from unstructured or semi-structured data. It provides a common, context-rich structure that enables interoperability between signal extraction systems and downstream consumers like predictive models and business intelligence tools. By standardizing the "last-mile" of data preparation for predictive tasks, UPSS aims to accelerate the development and deployment of predictive models.

---

### 1. Foundational Principles

The design of UPSS is guided by several principles to ensure its broad applicability, longevity, and ease of use.

*   **Industry/Use-Case Agnosticism:** The standard is fundamentally versatile, capable of representing signals from any domain, whether numerical sensor readings or qualitative assessments from sales calls. It relies on a minimal, universally relevant core and robust extensibility.
*   **Extensibility:** The standard is inherently extensible to accommodate new types of signals and metadata without necessitating breaking changes to the core specification.
*   **Time-Sensitivity:** Predictive signals are inherently temporal. The standard rigorously preserves the time dimension, capturing the precise moment an observation occurred, which is critical for ensuring point-in-time correctness.
*   **Simplicity and Intuitive Use:** To encourage widespread adoption, the standard is straightforward to understand and implement. The structure is logical and leverages common formats like JSON.

### 2. Leveraging CloudEvents for Context Metadata

To avoid reinventing the wheel, UPSS leverages the [CloudEvents v1.0.2 specification](https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/spec.md) for the event envelope. This provides a mature, widely adopted standard for describing event context, simplifying interoperability. UPSS mandates the use of specific CloudEvents attributes and defines the structure of the `data` payload.

- **REQUIRED CloudEvents Attributes:** `id`, `source`, `specversion`, `type`, `time`, `datacontenttype`.
- **OPTIONAL CloudEvents Attributes:** `subject`, `dataschema`.

The `specversion` for this version of the standard MUST be `"UPSS-0.1"`. The `datacontenttype` SHOULD be `application/json`.

### 3. The UPSS `data` Payload

The `data` attribute is the core of the UPSS. It is a JSON object designed to be both human-readable and machine-consumable.

#### 3.1. `provenance`

*   **Type:** `String`
*   **Constraints:** REQUIRED
*   **Description:** A human-readable description of the signal's origin and how it was generated.

#### 3.2. `target`

*   **Type:** `Object`
*   **Constraints:** REQUIRED
*   **Description:** Describes the predictive target this signal is relevant for.
    *   `id` (String): A unique identifier for the prediction target (e.g., `customer_churn_risk_90d`).
    *   `description` (String): A human-readable description.

#### 3.3. `series`

*   **Type:** `Array` of `Objects`
*   **Constraints:** OPTIONAL
*   **Description:** The core numerical time series data. Each object in the array represents a data point.
    *   `timestamp` (Timestamp): The timestamp of the data point.
    *   `value` (Number): The numerical value.

#### 3.4. `context`

*   **Type:** `Object`
*   **Constraints:** OPTIONAL
*   **Description:** The rich, natural language context that qualifies the numerical data. This is intended for human analysis and context-aware models.
    *   `descriptive` (String): Text describing historical patterns.
    *   `predictive` (String): Text describing future events or scenarios.
    *   `constraints` (String): Text describing hard constraints on future values.

#### 3.5. `features`

*   **Type:** `Array` of `Objects`
*   **Constraints:** OPTIONAL
*   **Description:** A collection of structured, quantifiable features derived from the `context`. This provides a **compatibility layer for traditional models** (e.g., XGBoost, ARIMA) and BI tools.
*   **Object Structure:** Each object in the array is a feature and MUST contain:
    *   `name` (String): The feature name (e.g., `is_holiday`).
    *   `value` (Number/String/Boolean): The feature value.
    *   `timestamp` (Timestamp): For point-in-time features.
    *   `time_range` (Object, OPTIONAL): For features spanning a duration, containing `start` and `end` timestamps.

#### 3.6. `decomposition`

*   **Type:** `Object`
*   **Constraints:** OPTIONAL
*   **Description:** Decomposed components of the time series.
    *   `trend`, `seasonality`, `events`: Each can contain a `series` (Array) and/or a `description` (String).

### 4. Example

A signal extracted from a weather alert, relevant for forecasting electricity demand.

```json
{
  "specversion": "UPSS-0.1",
  "type": "com.example.signal.weather_alert",
  "source": "https://weather.com/alerts/city-a",
  "subject": "city-a-electricity-grid",
  "id": "d234-5678-9012-e456",
  "time": "2025-07-01T10:00:00Z",
  "datacontenttype": "application/json",
  "dataschema": "https://example.com/schemas/upss/v1.0.json",
  "data": {
    "provenance": "Signal extracted from National Weather Service alert #5678.",
    "target": {
      "id": "electricity_demand_kw",
      "description": "Forecasted electricity consumption in Kilowatts for City A."
    },
    "context": {
      "predictive": "A severe heatwave is forecast for the next 3 days, with temperatures expected to exceed 40°C. Demand is projected to be 2.5x the seasonal average."
    },
    "features": [
      {
        "name": "heatwave_alert",
        "value": true,
        "time_range": {
          "start": "2025-07-02T00:00:00Z",
          "end": "2025-07-05T00:00:00Z"
        }
      },
      {
        "name": "expected_demand_multiplier",
        "value": 2.5,
        "time_range": {
          "start": "2025-07-02T00:00:00Z",
          "end": "2025-07-05T00:00:00Z"
        }
      }
    ]
  }
}
```
