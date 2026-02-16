# SpendWise – Production Cloud Architecture Case Study

This repository documents the architecture and engineering decisions behind a real-world AI-powered SaaS running in production on Google Cloud.

The goal is to explain how reliability, cost control and scalability are handled when operating a system that processes financial data using probabilistic AI models.

## What this demonstrates

- Designing deterministic workflows on top of non-deterministic LLM outputs
- Operating a serverless system with real users and real billing constraints
- Cost-aware AI usage (model selection and caching strategies)
- Async processing and validation pipelines
- Failure mitigation strategies for financial data integrity

## System context

SpendWise is a financial automation platform that converts pdf credit card statements into structured data and insights.

Users upload monthly invoices → the platform extracts, validates and categorizes transactions → dashboards are generated automatically.

The architecture is designed so the system refuses uncertainty instead of storing wrong financial data.

## Production constraints

This system operates under real-world constraints:

- unpredictable document formats
- probabilistic AI extraction
- cost per request matters
- financial correctness required

Because of this, AI is treated as a fallible component — never as source of truth.

---

More details about reliability, cost strategy and scaling decisions are documented in the sections below.

## Engineering Decisions

### 1. AI is not trusted by default

LLM extraction is probabilistic and financial data cannot tolerate silent errors.

Instead of storing model output directly, the system enforces reconciliation:

- extract transactions
- calculate totals
- compare with official statement values
- reprocess if mismatch detected

The system prefers to fail processing rather than store incorrect financial data.

---

### 2. Serverless over persistent infrastructure

The workload is bursty (monthly invoice uploads), not constant.

Using always-on infrastructure would increase operational cost without improving reliability.

Cloud Run was chosen to:

- scale to zero when idle
- scale horizontally during invoice processing spikes
- simplify operations (no node management)

Trade-off: cold starts were accepted and mitigated instead of eliminated.

---

### 3. Hybrid model usage for cost control

Different tasks require different levels of reasoning.

- complex extraction → higher capability model
- categorization → lower cost model

This reduced operational cost while preserving correctness where needed.

The architecture optimizes for **cost per processed invoice**, not raw accuracy.

---

### 4. Async processing instead of synchronous APIs

Statement processing may take several seconds and involves retries.

Synchronous requests would lead to timeouts and poor UX.

Instead:

upload → queue → worker → validation → persist → notify

This isolates user experience from processing complexity.

---

### 5. Deterministic validation layer

AI is allowed to suggest structure but never define truth.

All persisted financial data passes through schema validation and reconciliation logic.

The system converts probabilistic outputs into deterministic accounting data.
