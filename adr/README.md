# Architecture Decision Records (ADRs)

This directory contains the formal Architecture Decision Records for the AI SRE Platform.

Each ADR follows a standard lightweight structure:
- **Status:** Proposed / Accepted / Superseded / Deprecated
- **Context:** The problem and background from first principles.
- **Decision:** What was chosen and why.
- **Overrides & Traceability:** Explicit links to previous requirements or layers overridden or reconciled.
- **Consequences:** Positive and negative trade-offs.

## Index of ADRs

| ADR ID | Title | Status | Date | Overrides / Reconciles |
| :--- | :--- | :--- | :--- | :--- |
| [ADR-08](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-08-modular-monolith-and-three-process-runtime.md) | Modular Monolith Codebase with 3-Process Runtime Topology | Accepted | 2026-11-26 | Layer 6 Q6.3, Q6.5, Q6.8 |
| [ADR-09](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-09-intake-api-handshake-and-enrichment-fallback.md) | Synchronous Intake Handshake (200 OK) with Graceful Missing Field Enrichment | Accepted | 2026-11-26 | Layer 4 § 4.3, § 4.28, Q6.1, Q6.9 |
| [ADR-10](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-10-postgresql-backed-concurrency-and-job-queue.md) | PostgreSQL-Backed Job Queue with SKIP LOCKED and 3-Slot Ceiling | Accepted | 2026-11-26 | Layer 4 § 4.22, § 4.34, Layer 6 Q6.4, Q6.8, Q6.11 |
| [ADR-11](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-11-deterministic-diagnosis-verification-over-llm-debate.md) | Deterministic Diagnosis Verification Over Multi-Round LLM Debate | Accepted | 2026-11-26 | Layer 4 § 4.6, § 4.7, Layer 6 Q6.16 |
| [ADR-12](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-12-composite-sha256-patch-and-state-fingerprint.md) | Composite SHA-256 Approval Fingerprinting (Target State + Canonical Diff) | Accepted | 2026-11-26 | Layer 4 § 4.32, Layer 5 § 6, Layer 6 Q6.18 |
| [ADR-13](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-13-fastapi-http-gateway-and-pydantic-validation.md) | FastAPI and Pydantic for HTTP Intake & Status Gateway | Accepted | 2026-11-26 | Layer 6 Q6.23 |
| [ADR-14](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-14-canonical-9-state-incident-lifecycle-reconciliation.md) | Canonical 9-State Incident Lifecycle Machine Reconciliation | Accepted | 2026-11-26 | Layer 3 § 5.3, Layer 4 Summary vs Table, Layer 6 Q6.11 |
