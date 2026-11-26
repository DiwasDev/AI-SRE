# ADR-09: Synchronous Intake Handshake (200 OK) with Graceful Missing Field Enrichment

## Status
Accepted

## Context (First Principles)
External monitoring systems (Datadog, Prometheus, Alertmanager) send webhooks when production systems fail. These monitoring tools expect a rapid response. If the intake endpoint takes several seconds or minutes (because it started diagnosing), the monitoring system will timeout and repeatedly retry, flooding the platform with duplicate alerts.

Furthermore, incoming alert payloads frequently miss non-essential context (e.g., exact commit hash or endpoint path). Dropping an alert because one field is missing leaves on-call engineers blind to real production outages.

## Decision
1. **Synchronous Intake Handshake (`200 OK`):**
   - The HTTP intake route performs only four fast operations:
     1. Authenticate sender token/signature.
     2. Validate the mandatory 5 fields schema (`service_id`, `environment`, `timestamp`, `symptom`, `source_endpoint`).
     3. Perform composite deduplication lookup (`org_id` + `service_id` + `error_signature` + `environment` over a 30-minute sliding window).
     4. Persist the incident and enqueue a dispatch job atomically in PostgreSQL.
   - Immediately returns `200 OK` with the `incident_id` within the SLA of `p95 <= 500 ms` (max 2 seconds).
2. **Missing Field Handling:**
   - If optional or enrichable metadata is missing, the alert is **never dropped**. The incident is saved and flagged with `"needs_info"`.
   - The background worker attempts automated enrichment via read-only CI/CD integrations.
   - If still unresolved, a one-shot query is dispatched to the on-call engineer (with a 5-minute timeout) before falling back to `"Inconclusive"`.

## Overrides & Traceability
- **Standardizes:** Adopts `200 OK` (from Layer 4 § 4.28) over `202 Accepted` to ensure consistency with existing Layer 4 SLAs.
- **Reconciles:** Layer 4 § 4.3 & § 4.28 with Layer 6 Q6.1 & Q6.9.

## Consequences
- **Positive:** Zero risk of HTTP timeout; predictable webhook acknowledgement; alerts are never silently discarded.
- **Negative:** The client must poll or read the API to observe subsequent investigation progress.
