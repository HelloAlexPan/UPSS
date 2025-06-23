# **Universal Predictive Signal Standard (UPSS) Specification**

**Version 0.1**

The Universal Predictive Signal Standard (UPSS) is a vendor-neutral specification for defining the format of predictive signals, especially those derived from unstructured or semi-structured data. It provides a common, context-rich structure that enables interoperability between signal extraction systems and downstream consumers like predictive models and business intelligence tools. By standardizing the "last-mile" of data preparation for predictive tasks, UPSS aims to accelerate the development and deployment of of predictive models.

---

### **1. Foundational Principles**

The design of UPSS is guided by several principles to ensure its broad applicability, longevity, and ease of use.

*   **Industry/Use-Case Agnosticism:** The standard is fundamentally versatile, capable of representing signals from any domain, whether numerical sensor readings or qualitative assessments from sales calls.
*   **Extensibility:** The standard is inherently extensible to accommodate new types of signals and metadata without necessitating breaking changes to the core specification.
*   **Time-Sensitivity:** Predictive signals are inherently temporal. The standard rigorously preserves the time dimension, capturing the precise moment an observation occurred.
*   **Simplicity and Intuitive Use:** To encourage widespread adoption, the standard is straightforward to understand and implement. The structure is logical and leverages common formats like JSON.

### **2. Leveraging CloudEvents for Context Metadata**

To avoid reinventing the wheel, UPSS leverages the [CloudEvents v1.0.2 specification](https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/spec.md) for the event envelope. This provides a mature, widely adopted standard for describing event context.

*   **REQUIRED CloudEvents Attributes:** `id`, `source`, `specversion`, `type`, `time`, `datacont
*   enttype`.
*   **OPTIONAL CloudEvents Attributes:** `subject`, `dataschema`.

The `specversion` for this version of the standard MUST be `"UPSS-0.1"`. The `datacontenttype` SHOULD be `application/json`.

### **3. The UPSS `data` Payload**

The `data` attribute is the core of the UPSS. It is a JSON object designed to be both human-readable and machine-consumable, providing a rich, multi-faceted view of a predictive signal.

#### **4.1. `provenance`**

*   **Type:** `String`
*   **Constraints:** REQUIRED
*   **Description:** A human-readable description of the signal's origin and how it was generated for auditability.

#### **4.2. `target`**

*   **Type:** `Object`
*   **Constraints:** REQUIRED
*   **Description:** Describes the predictive target this signal is relevant for.
    *   `id` (String): A unique identifier for the prediction target (e.g., `customer_churn_risk_90d`).
    *   `description` (String): A human-readable description.

#### **4.3. `series`**

*   **Type:** `Array` of `Objects`
*   **Constraints:** OPTIONAL
*   **Description:** The core numerical time series data associated with the signal.
    *   `timestamp` (Timestamp): The timestamp of the data point.
    *   `value` (Number): The numerical value.

#### **4.4. `context`**

*   **Type:** `Object`
*   **Constraints:** OPTIONAL
*   **Description:** The rich, natural language context that qualifies the numerical data. This field is designed to handle complex, nested, and qualitative information, ensuring human auditability and trust as well as making it ideal for consumption by context-aware models like LLMs.

#### **4.5. `features`**

*   **Type:** `Array` of `Objects`
*   **Constraints:** OPTIONAL
*   **Description:** A collection of structured, quantifiable features derived from the `context` for models like XGBoost, ARIMA, and BI tools. It is a flattened, machine-readable representation of the key insights contained within the `context` object.
*   **Object Structure:** Each object in the array is a feature and MUST contain:
    *   `name` (String): The feature name (e.g., `is_holiday`).
    *   `value` (Number/String/Boolean): The feature value.
    *   `timestamp` (Timestamp): For point-in-time features.
    *   `time_range` (Object, OPTIONAL): For features spanning a duration, containing `start` and `end` timestamps.

### **5. Example: Complex Qualitative Signal**

This example showcases the full power of UPSS to represent a complex, qualitative signal extracted from a sales call transcript. It demonstrates how both the rich, nested `context` and the simplified, tabular `features` can coexist in a single signal.

```json
{
  "specversion": "UPSS-0.1",
  "type": "com.example.sales.qualification_analysis",
  "source": "/sales/opportunity/opp_789",
  "subject": "call_transcript_abc",
  "id": "evt-uuid-qual-5678",
  "time": "2025-04-16T14:30:00Z",
  "datacontenttype": "application/json",
  "dataschema": "https://example.com/schemas/sales-qualification-signals/v0.1.json",
  "data": {
    "provenance": "Signal extracted from sales call transcript 'call_transcript_abc' using qualification model v2.1.",
    "target": {
      "id": "opportunity_close_probability_90d",
      "description": "Probability of sales opportunity closing successfully in the next 90 days."
    },
    "context": {
      "qualification_signals": {
        "need_and_pain_points": {
          "explicit_problem_statements": {
            "assessment": "High",
            "evidence": ["Customer stated 'current solution is too slow'", "Mentioned impact on quarterly targets"]
          },
          "quantified_pain": {
            "assessment": "Medium",
            "evidence": ["Estimated $50k loss per quarter"]
          }
        },
        "authority_and_decision_making": {
          "decision_maker_confirmation": {
            "assessment": "Low",
            "evidence": ["'I need to run this by my manager'"]
          }
        }
      }
    },
    "features": [
      {
        "name": "explicit_problem_assessment",
        "value": "High",
        "timestamp": "2025-04-16T14:30:00Z"
      },
      {
        "name": "quantified_pain_assessment",
        "value": "Medium",
        "timestamp": "2025-04-16T14:30:00Z"
      },
      {
        "name": "decision_maker_confirmed",
        "value": false,
        "timestamp": "2025-04-16T14:30:00Z"
      },
      {
        "name": "qualification_score",
        "value": 0.65,
        "timestamp": "2025-04-16T14:30:00Z"
      }
    ]
  }
}
```

### **6. Design Rationale & Use Cases**

UPSS is designed to solve common, practical problems in predictive modeling:

*   **Problem:** Using unstructured text in predictive models is hard and requires bespoke feature engineering for every project.
    *   **UPSS Solution:** The `context` and `features` attributes provide a standardized structure. The `context` object holds the rich, nested, qualitative text for advanced models, while the `features` object provides immediately usable, structured data for traditional models. This eliminates the need for redundant "last-mile" ETL for every new project.

*   **Problem:** My forecasts are often wrong because they miss crucial context that isn't in the historical numbers.
    *   **UPSS Solution:** The `context` object allows for the encoding of future events, scenarios, and known limitations directly into the data stream, enabling models to produce more realistic and reliable predictions.

*   **Problem:** It's difficult to combine signals from different sources (e.g., sales calls and support tickets) to get a holistic view of a customer.
    *   **UPSS Solution:** The common `subject` attribute and standardized format allow signals from any source to be merged into a single, time-ordered stream for a given entity, enabling more powerful, multi-faceted models.

*   **Problem:** I can't trust my model's predictions because I don't know where the data came from.
    *   **UPSS Solution:** The `source` and `provenance` attributes provide a clear audit trail, ensuring that every predictive signal is traceable back to its origin.

### **7. Schema Versioning and Evolution**

UPSS uses Semantic Versioning (SemVer). The `specversion` attribute refers to the version of this document. The `dataschema` URI should also be versioned. Backward compatibility is prioritized for MINOR/PATCH versions. Breaking changes to the payload structure require a MAJOR version bump in the `dataschema` URI.
