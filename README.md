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
