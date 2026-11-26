# ADR-14: Canonical 9-State Incident Lifecycle Machine Reconciliation

## Status
Accepted

## Context (First Principles)
An incident platform must have an authoritative, predictable state machine stored in PostgreSQL.
Across earlier documentation, a minor divergence occurred:
- Layer 3 § 5.3 and Layer 5 § 8 established a 9-state machine with autonomous sandbox testing and a single human gate.
- Layer 4 § 1.3 mentioned an "11-state incident machine", but listed standard lifecycle states alongside terminal and transient tags (such as `Recurred`, `Cancelled`, `Inconclusive`).

Without one explicit, canonical list of states, database migrations, UI status badges, and worker recovery handlers will become inconsistent.

## Decision
1. **Canonical 9 Core States:**
   The primary incident lifecycle progresses through exactly 9 core states:
   1. `Detected` — Webhook received, validated, and persisted.
   2. `Notified` — Owner/worker assigned (claimed from queue).
   3. `Investigating` — Read-only evidence collection and hypothesis generation active.
   4. `DiagnosisProposed` — Root cause hypothesis formulated and rule-validated.
   5. `SandboxTesting` — Ephemeral container verifying bug reproduction, fix diff, and test suite.
   6. `AwaitingDeployApproval` — Autonomous sandbox tests passed; one-page review package presented to human.
   7. `Deployed` — Human approved; canary rollout active via CI/CD.
   8. `Verifying` — Monitoring post-deploy error rates and latency against pre-incident baseline.
   9. `Closed` — Baseline health verified and sustained for 15–30 min; audit committed.

2. **Terminal & Recovery Sub-Statuses:**
   - `Inconclusive` — Handover dossier generated due to timeout (15m), budget cap ($20), or ambiguous evidence. (Sub-state of `Investigating` / terminal handover).
   - `Cancelled` — Terminated via Tier 1 graceful stop or Tier 2 emergency kill switch.
   - `Recurred` — Reopened within 30-minute deduplication window following closure.

## Overrides & Traceability
- **Reconciles:** Layer 3 § 5.3, Layer 4 Executive Summary, Layer 5 § 8, and Layer 6 Q6.11.

## Consequences
- **Positive:** Unambiguous database schema enum; exact alignment between UI, API, and worker state machines.
- **Negative:** None.
