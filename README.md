# A Universal Standard for Predictive Signals (UPSS)

**EXPERIMENTAL — WORK IN PROGRESS**

The growing ease by which large language models can extract predictive signals from unstructured data is set to unlock vast streams of time-stamped, event-driven data across nearly every industry. This data holds immense potential for predictive modeling, enabling organizations to anticipate future events, optimize operations, and gain competitive advantages. However, realizing this potential is often hindered by significant challenges. A primary obstacle is the lack of standardization in how predictive signals – the raw metrics, indicators, scores, and *qualitative assessments* derived from this data – are represented and exchanged. Similar to the interoperability issues faced by general event data before initiatives like CloudEvents, the absence of a common format for predictive signals forces developers to build custom ingestion and processing logic for each data source and predictive model, stifling model innovation, experimentation, and reuse.

Furthermore, transforming these raw signals, especially qualitative ones, into actionable predictions typically demands considerable domain expertise and manual effort. Data scientists and analysts invest significant time in data preprocessing, feature engineering, model selection, and validation – processes that are often bespoke to specific use cases. This high barrier to entry prevents many organizations from effectively leveraging their data for predictive tasks. There exists a clear need for platforms that can automate the complex workflow from raw signal data to prediction, democratizing predictive capabilities and accelerating time-to-value, while ensuring the methods used are understandable and auditable, particularly when qualitative inputs are used.

The **Universal Predictive Signal Standard (UPSS)** aims to establish a flexible and extensible format for representing diverse predictive signals as time-ordered events. Drawing inspiration from the success of CloudEvents in standardizing general event metadata, UPSS focuses specifically on structuring data intended as input for predictive modeling. It defines a core set of metadata attributes for context and provides clear guidelines for structuring the signal payload itself—leveraging JSON's flexibility for nested qualitative data—ensuring compatibility with automated processing techniques while maintaining broad applicability. UPSS defines the *event format*, crucial for interoperability before data storage.

## A. Foundational Principles

The design of the Universal Predictive Signal Standard (UPSS) is guided by several foundational principles aimed at ensuring its broad applicability, longevity, and ease of use. These principles draw inspiration from established standards like CloudEvents and concepts from Universal Design.

1. **Industry/Use-Case Agnosticism:** The standard must be fundamentally versatile, capable of representing predictive signals from any domain, whether numerical sensor readings or qualitative assessments from sales calls. Achieving this requires avoiding domain-specific terminology within the core specification. Instead, the standard relies on a minimal, universally relevant core and robust extensibility mechanisms.
2. **Extensibility:** Recognizing that the landscape of predictive signals will inevitably evolve, the standard must be inherently extensible. It needs to gracefully accommodate new types of signals (like complex qualitative structures) or additional metadata without necessitating breaking changes to the core specification. This mirrors a key design goal of CloudEvents  and reflects best practices in flexible schema design. Flexibility in use is paramount.
3. **Time-Sensitivity:** Predictive signals are inherently temporal. The standard must rigorously preserve the time dimension, capturing the precise moment an observation or assessment occurred. This is critical for time-series analysis and ensuring point-in-time correctness during model training.
4. **Simplicity and Intuitive Use:** To encourage widespread adoption, the standard must be straightforward to understand, implement, and validate. Unnecessary complexity acts as a barrier. The structure should be logical, especially when using common formats like JSON.

A critical balance must be struck between agnosticism and utility. The design defines a minimal but meaningful core structure leveraging CloudEvents, particularly optimized for time-series data representation (entity, timestamp), while relying heavily on payload flexibility (especially JSON) and well-defined extension mechanisms to handle diversity, including nested qualitative data. UPSS defines the *event format* for data in transit, remaining agnostic to downstream storage schemas or modeling techniques.

## B. Leveraging CloudEvents for Context Metadata

UPSS leverages the CloudEvents specification for event metadata. This provides a mature, widely adopted standard for describing event context, simplifying interoperability.

UPSS utilizes the following CloudEvents context attributes:

- **`id` (REQUIRED):** Unique identifier for this signal event instance.
- **`source` (REQUIRED):** **Entity ID.** Identifies the entity generating the signal (e.g., sensor ID, sales opportunity ID).
- **`specversion` (REQUIRED):** Version of the UPSS specification (e.g., `"UPSS-1.0"`).
- **`type` (REQUIRED):** High-level category of the signal event (e.g., `com.example.sensor.reading`, `com.example.sales.qualification_analysis`).
- **`time` (REQUIRED):** Timestamp of the signal occurrence (RFC 3339).
- **`datacontenttype` (REQUIRED):** Media type of the `data` payload (e.g., `application/json`).    
- **`subject` (Optional):** Subject of the event within the `source` context (e.g., could link to a specific sales call transcript ID).
- **`dataschema` (Optional):** URI pointing to the schema definition (e.g., JSON Schema) for the `data` payload. Recommended for validation, especially with complex qualitative data.
    
Essentially, CloudEvents provides the standardized *envelope*; UPSS mandates essential attributes and focuses on defining the expected structure and semantics *within* the `data` payload, tailored for predictive signals, including qualitative ones.

## C. UPSS Schema Definition

The UPSS schema combines CloudEvents context attributes with a flexible definition for the `data` payload, designed to handle numerical, categorical, and complex qualitative signals like the sales qualification example.

**1. Core Context Attributes:** As detailed above, `id`, `source`, `specversion`, `type`, `time`, and `datacontenttype` are required.

**2. Flexible Payload Design (`data` attribute):** Contains the actual signal measurements or assessments.

- **Handling Mixed Data Types:**
    - *Numerical Metrics:* Standard JSON `number`.        
    - *Categorical Indicators:* JSON `string`, potentially with `enum` constraint via JSON Schema.        
    - *Scores:* JSON `number`, potentially with range constraints via JSON Schema.        
    - *Qualitative Assessments:* JSON `string` (e.g., "High Fit", "Medium Fit").
    - *Textual Evidence:* JSON `array` of `string`s.        
    - *Nested Structures:* JSON objects allow for hierarchical organization as seen in the sales qualification example.        
    
JSON is recommended as the primary `datacontenttype` for UPSS due to its readability, widespread support, and native handling of nested structures and arrays essential for qualitative signals. To ensure structure and validity, especially for complex payloads, JSON Schema should be used. The dataschema attribute in the CloudEvent should reference the relevant JSON Schema URI. Protobuf can be a supported alternative for performance-critical use cases, but JSON's flexibility is advantageous for evolving qualitative structures.
    
| **Feature** | **JSON** | **Protobuf** |
| --- | --- | --- |
| **Readability** | High | Low |
| **Performance (Parsing)** | Moderate | Fast |
| **Size Efficiency** | Lower | Higher |
| **Schema Enforcement** | Good (via JSON Schema) | Strong (via `.proto`) |
| **Flexibility (Nested/Qualitative)** | **Very High** | Moderate |
| **Ecosystem Support** | Very Wide | Wide |
| **Ease of Adoption** | High | Moderate |
- **Representing Multiple Signals (including Qualitative):** The **Single UPSS Event with Structured Payload (Object Approach)** should be the recommended default. The `data` payload would be a JSON object where keys represent different signals (numerical, categorical, or complex qualitative structures).
    
**3. Extensibility Mechanisms:**

- **CloudEvents Extension Attributes:** For custom *metadata* about the event.
- **Payload Extensibility:** For adding new signal types or attributes *within* the signal data, leverage JSON's flexibility by adding new key-value pairs to the `data` object. JSON Schema's `additionalProperties` can control this.

**4. Time-Series Data Modeling Considerations (Decoupling Event from Storage):** UPSS defines the *event format*. The pipeline which processes these signals to predictions chooses the *internal storage schema*. This decoupling allows the user to optimize storage (e.g., using a narrow schema with JSONB for qualitative data) independently of the standardized event format used for ingestion.

## D. Proposed UPSS Specification

The following table summarizes the core structure of a UPSS event, combining CloudEvents context attributes with UPSS-specific requirements and payload recommendations.

| **Attribute Name** | **Source** | **Required/Optional** | **Data Type** | **Description (UPSS Semantics)** |
| --- | --- | --- | --- | --- |
| `id` | CloudEvents | REQUIRED | String | Unique identifier for this signal event instance.  |
| `source` | CloudEvents | REQUIRED | URI-Reference | **Entity ID.** Identifies the entity generating the signal (e.g., sensor, sales opportunity). |
| `specversion` | UPSS Defined | REQUIRED | String | Version of the UPSS specification being used (e.g., "UPSS-1.0").  |
| `type` | CloudEvents | REQUIRED | String | High-level category of the signal event (e.g., `com.example.sales.qualification_analysis`). |
| `time` | CloudEvents | REQUIRED | Timestamp (RFC 3339) | Timestamp of the signal occurrence or assessment.  |
| `datacontenttype` | CloudEvents | REQUIRED | String | Media type of the `data` payload (e.g., `application/json`).  |
| `dataschema` | CloudEvents | OPTIONAL | URI | URI pointing to the schema definition (e.g., JSON Schema) for the `data` payload. Highly recommended.  |
| `subject` | CloudEvents | OPTIONAL | String | Subject of the event within the `source` context (e.g., specific call transcript ID).  |
| `data` | CloudEvents | OPTIONAL (but typically present for UPSS) | JSON Object/Array | The payload containing signal values/assessments. Structure defined by `dataschema`.  |
| *(Extensions)* | CloudEvents | OPTIONAL | Varies | Additional custom context attributes. |



**Example UPSS Event (Sales Qualification Use Case):**

```JSON
{
  "specversion": "UPSS-1.0",
  "type": "com.example.sales.qualification_analysis",
  "source": "/sales/opportunity/opp_789",
  "subject": "call_transcript_abc",
  "id": "evt-uuid-qual-5678",
  "time": "2025-04-16T14:30:00Z",
  "datacontenttype": "application/json",
  "dataschema": "https://example.com/schemas/sales-qualification-signals/v1.0.json",
  "data": {
    "qualification_signals": {
      "need_and_pain_points": {
        "explicit_problem_statements": {
          "assessment": "High", // Categorical signal
          "evidence": ["Customer stated 'current solution is too slow'", "Mentioned impact on quarterly targets"] // Array of strings
        },
        "quantified_pain": {
          "assessment": "Medium",
          "evidence": ["Estimated $50k loss per quarter"]
        }
        //... other need_and_pain_points signals...
      },
      "authority_and_decision_making": {
        "decision_maker_confirmation": {
          "assessment": "Low",
          "evidence": ["'I need to run this by my manager'"]
        }
        //... other authority_and_decision_making signals...
      },
      "budget_and_financial_capacity": {
         "budget_confirmation": {
           "assessment": "Medium",
           "evidence":
         }
         //... other budget_and_financial_capacity signals...
      }
      //... other top-level categories (timing, fit, etc.)...
    }
  }
}
```

## E. Schema Versioning and Evolution Strategy

UPSS uses **Semantic Versioning (SemVer)** to both the `specversion` and the `dataschema` URI. **Backward compatibility** is prioritized for MINOR/PATCH versions. Breaking changes require MAJOR version bumps. An optional **Schema Registry** can aid management.
