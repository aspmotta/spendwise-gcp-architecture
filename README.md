## SpendWise — Reliable AI Processing for Financial Documents

**Production SaaS that converts credit card statements into structured financial data without trusting the AI output.**

Financial systems cannot tolerate silent errors, but LLM extraction is inherently probabilistic.

Instead of trying to make the model perfect, the system was designed so incorrect data is harder to produce than a processing failure.

**Core principle**

AI proposes → system verifies → only reconciled data is accepted

If totals don’t match:
- a constrained correction pass runs
- if still inconsistent → the system refuses the result and requires review

The platform prefers failing processing over storing wrong financial records.

---

### Why this matters

Most AI document pipelines optimize for extraction accuracy.

In accounting workflows, accuracy is not enough — correctness must be guaranteed.

This architecture demonstrates how to safely embed probabilistic models inside deterministic systems using validation, reconciliation and failure containment.

---

### What this repository shows

- Operating AI in reliability-critical workflows
- Designing validation layers around non-deterministic components
- Cost-aware multi-model orchestration
- Serverless async processing under burst workloads
- Failure containment instead of blind retries

---

### Real constraints handled

- inconsistent invoice layouts across banks
- probabilistic model output
- per-request cost pressure
- exact reconciliation requirement

AI is treated as a fallible sensor — never the source of truth.

---

## Why this is a hard problem

Credit card invoices are inconsistent across banks and layouts.  
AI extraction is inherently uncertain.  
Users, however, expect exact accounting.

This creates a constraint:

> The system must tolerate AI uncertainty without propagating financial errors.

---

## What this demonstrates

- Deterministic workflows on top of probabilistic AI outputs
- Cost-aware multi-model strategy (Pro vs Flash + caching + learning rules)
- Serverless architecture designed for burst workloads
- Async processing pipelines with reconciliation validation
- Failure containment strategies instead of blind retries

---

## System Context

SpendWise converts PDF credit card statements into structured financial insights.

Flow:
Upload → Extract → Validate → Store → Visualize

Unlike typical document AI systems, the platform refuses uncertain results rather than storing incorrect data.

---

## Production Constraints

The system operates under real-world conditions:

- unpredictable document formats (Nubank, Itaú, Bradesco, C6…)
- probabilistic AI extraction
- cost per request matters
- totals must reconcile exactly

Because of this, AI is treated as a fallible component — never as source of truth.

---

Detailed reliability, cost and scaling decisions are documented below.

## Engineering Decisions

### 1. AI is not trusted by default

LLM extraction is probabilistic and financial data cannot tolerate silent errors.

Instead of storing model output directly, the system enforces reconciliation:

- **Manifest Generation**: Extracts header data (total, due date) and card holders using the latest **Pro** model.
- **Validation Loop**: Calculates the sum of extracted transactions.
- **Correction**: If `sum(transactions) != total_amount`, the system triggers a focused correction prompt or rejects the extraction.
- **Embedded Autocorrection**: The extraction pipeline includes self-correction steps where the LLM reviews its own output against constraints.

The system prefers to fail processing rather than store incorrect financial data.

---

### 2. Serverless over persistent infrastructure

The workload is bursty (monthly invoice uploads), not constant.

Using always-on infrastructure would increase operational cost without improving reliability.

**Stack:**
- **Compute**: Google Cloud Run (scales to zero)
- **Database**: Cloud SQL (PostgreSQL)
- **Queues**: Celery with Cloud Tasks

Cloud Run was chosen to:
- scale to zero when idle
- scale horizontally during invoice processing spikes
- simplify operations (no node management)

Trade-off: cold starts were accepted and mitigated instead of eliminated.

---

### 3. Hybrid model usage for cost control

Different tasks require different levels of reasoning.

- **Complex Extraction (Manifests)**: **Gemini Pro** (High capability, higher cost)
- **Categorization & Classification**: **Gemini Flash** (Fast, 10x cheaper)

This reduced operational cost while preserving correctness where needed. The architecture optimizes for **cost per processed invoice**, not raw accuracy.

---

### 4. Context Caching Strategy

To further reduce AI costs, the system uses **Gemini Context Caching**:

1.  The PDF content is uploaded once and cached with a TTL (Time-To-Live).
2.  Subsequent prompts (Layout Analysis, Manifest Extraction, Transaction Extraction) reuse this cached token context.
3.  **Result**: ~75% reduction in input token costs for multi-step processing.

---

### 5. Detailed Processing Pipeline

Statement processing involves multiple asynchronous stages to ensure accuracy and user feedback.

1.  **Classification (Synchronous)**
    *   **Input**: First page of PDF.
    *   **Model**: **Flash**.
    *   **Action**: Determines if the document is a supported Credit Card Invoice or an unsupported Bank Statement. Rejects invalid files immediately.

2.  **Context Cache Creation**
    *   **Action**: The full PDF is uploaded to Gemini and cached. This cache ID is passed to all subsequent steps, drastically reducing token costs.

3.  **Layout Analysis**
    *   **Model**: **Pro** (via Cache).
    *   **Action**: Identifies page ranges for each cardholder and detects "miscellaneous" pages (ads, terms) to skip. This allows focused extraction.

4.  **Manifest Extraction**
    *   **Model**: **Pro** (via Cache).
    *   **Action**: Extracts the invoice header (Total, Due Date, Bank) and the list of Cardholders (Names + Subtotals). This establishes the "Ground Truth" for validation.

5.  **Transaction Extraction**
    *   **Model**: **Pro** (via Cache).
    *   **Action**: Extracts transactions for each holder.
    *   **Autocorrection**: The LLM compares its extracted sum against the Manifest subtotal *during generation* and self-corrects if they don't match.

6.  **Validation & Persistence**
    *   **Action**: Python logic verifies `sum(transactions) == manifest_total`. If valid, transactions are de-duplicated and saved to PostgreSQL.

---

### 6. Deterministic validation layer

AI is allowed to suggest structure but never define truth.

All persisted financial data passes through schema validation (Pydantic) and reconciliation logic.

The system converts probabilistic outputs into deterministic accounting data.

## Operational Reliability

Operating a financial system means failures are expected and must be contained.

The platform is designed so that incorrect data is harder to produce than a processing failure.

### AI extraction failures

LLM output may omit transactions or misinterpret document structure.

**Mitigation:**
- **Reconciliation**: Validates `sum(transactions) == total`.
- **Focused Correction**: If a specific card section is missing or malformed, a targeted prompt asks the LLM to re-examine just that part.
- **Fallback**: If structure cannot be guaranteed, the invoice is flagged for review rather than silently discarded.

---

### External dependency instability

AI providers and payment gateways are external dependencies.

**Mitigation:**
- **Retries**: Exponential backoff for transient API errors.
- **Idempotency**: Tasks can be re-run safely without creating duplicate transactions (deduplication based on unique constraints).

---

### Cost explosion protection & Categorization Hierarchy

AI costs scale with usage. To mitigate this, categorization follows a strict hierarchy (Waterfall Strategy):

1.  **Learning System (Level 1)**: The system checks if the user has previously categorized a similar transaction (e.g., "UBER" -> "Transport"). If a match is found, it is applied **without any AI call**.
2.  **Known Merchants (Level 2)**: If no user rule exists, the system checks a static database of known merchants (Spotify, Netflix, Amazon). Matches are applied automatically.
3.  **LLM Categorization (Level 3)**: Only if the description is unknown to both previous layers, the **Flash** model is called to categorize the transaction.

**Result**: This strategy reduces AI categorization calls by 60-80% for recurring users.

**Other mitigations:**
- **Model Selection**: Using `flash` for high-volume tasks.
- **Caching**: Reusing context for the same PDF.
- **Early Rejection**: The system classifies the document type (using a cheap `flash` call) immediately after upload. Non-invoices are rejected before expensive extraction begins.

---

### Scaling risks

The primary scaling risks identified:

- **AI Rate Limits**: 60 RPM limit on Gemini.
- **Database Connections**: Serverless scaling can exhaust connection pools.

**Current Solutions:**
- **Queue Throttling**: Celery workers process tasks at a controlled rate.
- **Connection Pooling**: SQLAlchemy pool settings optimized for Cloud Run.
