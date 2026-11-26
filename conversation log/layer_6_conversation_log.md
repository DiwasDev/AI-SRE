# Layer 6 — System Architecture & Component Design: Conversation Log

> **Context & Review Baseline:**
> This conversation log compiles and formally records all approved architectural decisions, component breakdowns, and technology evaluations for **Layer 6: System Architecture**, integrating evaluations from team transcripts, Claude, and ChatGPT against [Layer 4: Functional & Non-Functional Requirements](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/artifacts/functional_and_non_functional_requirements.md) and [Layer 5: Threat Modeling & Security](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/artifacts/threat_modeling.md).
>
> All architectural modifications, overrides, and cross-layer reconciliations are formally codified in the [`/adr`](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/README.md) directory ([ADR-08](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-08-modular-monolith-and-three-process-runtime.md) through [ADR-14](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-14-canonical-9-state-incident-lifecycle-reconciliation.md)).

---

## Master Architecture Decisions & Component Index

| Question | Topic | Approved Decision Summary | Linked ADR / Artifact Ref |
| :--- | :--- | :--- | :--- |
| **Q6.1** | 10-Minute Investigation vs HTTP Timeout | Synchronous intake handshake returning `200 OK` (p95 $\le$ 500 ms); long investigation delegated to background worker; status observed via DB polling. | [ADR-09](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-09-intake-api-handshake-and-enrichment-fallback.md) |
| **Q6.2** | Synchronous vs Asynchronous Operations | **Sync:** Intake, deduplication, DB save, status reads, approval clicks, kill switch.<br>**Async:** Telemetry rounds (bounded parallel reads capped at 4), LLM reasoning, sandbox verification, canary rollout.<br>**Approval Wait:** Worker checkpoints and yields execution while the slot remains reserved. | Layer 4 § 4.4, § 4.22, § 4.34 |
| **Q6.3** | Splitting the Do-Everything API | Three distinct roles with strict privilege separation: `api` (ingress & approval records), `worker` (triage & sandbox testing), and `executor` (production deployment only). | [ADR-08](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-08-modular-monolith-and-three-process-runtime.md), Layer 5 § 3, § 6 |
| **Q6.4** | Background Processing Responsibilities | PostgreSQL queue via `SKIP LOCKED`; 3 active slots, 20 queue cap; heartbeat supervision (60s) with 3-crash circuit breaker; budget & timeout enforcement. | [ADR-10](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-10-postgresql-backed-concurrency-and-job-queue.md) |
| **Q6.5** | Problems with Too Many Separate Services | Avoid premature microservices. Eliminates distributed transactions, multi-hop latency, tracing overhead, and complex service-to-service authentication for a small team. | [ADR-08](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-08-modular-monolith-and-three-process-runtime.md), Layer 2 § 2.2 |
| **Q6.6** | Organizing One Core Without Distributed Complexity | Modular Monolith structured with Ports & Adapters; strict public interfaces; zero cross-module direct table manipulation; CI-enforced dependency checking via `import-linter`. | [ADR-08](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-08-modular-monolith-and-three-process-runtime.md), Layer 3 § 7.2 |
| **Q6.7** | Approach for Fast MVP + Strict Boundaries | Formal adoption of **Modular Monolith with Ports & Adapters** as the primary architectural style for the 6-month MVP. | [ADR-08](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-08-modular-monolith-and-three-process-runtime.md) |
| **Q6.8** | Separate Processes Sharing One Codebase | Single codebase and single built Docker image started as 3 distinct services (`api`, `worker`, `executor`) in Docker Compose sharing domain packages and DB schemas. | [ADR-08](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-08-modular-monolith-and-three-process-runtime.md), [ADR-10](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-10-postgresql-backed-concurrency-and-job-queue.md) |
| **Q6.9** | Receive & Validate Incoming Incidents | **Incident Intake Gateway** (inside `api`): Validates 5 mandatory fields, deduplicates, atomically registers incident + job, and replies `200 OK`. Missing context marked `"needs_info"` (never dropped). | [ADR-09](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-09-intake-api-handshake-and-enrichment-fallback.md) |
| **Q6.10** | Persistent Incident Records | **Incident Repository / Record Store**: Encapsulates PostgreSQL state and insert-only append-only timeline table. In-memory buffering used only during total DB outage fallback. | Layer 4 § 4.15, § 4.18, Layer 5 § 9 |
| **Q6.11** | Durable Workflow Progress After Crash | **Workflow Orchestrator (Tracker + Supervisor)**: Checkpoints state after every discrete step; enforces heartbeats; drives canonical 9-state machine; idempotent retries. | [ADR-10](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-10-postgresql-backed-concurrency-and-job-queue.md), [ADR-14](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-14-canonical-9-state-incident-lifecycle-reconciliation.md) |
| **Q6.12** | Model Inference Requests | **Model Gateway**: Centralized proxy managing provider failover, 2-minute timeouts, schema validation, cost/token accounting ($20 / 500k tokens), and bidirectional sanitization. | Layer 4 § 4.21, § 4.23, § 4.30, § 4.31 |
| **Q6.13** | Retrieving Internal Runbooks | **Runbook Lookup**: Simple tag and keyword-indexed lookup behind a port interface returning reference markdown context; zero vector databases in MVP. | Layer 5 § 2 |
| **Q6.14** | Coordinating Evidence Collection | **Evidence Collection Coordinator**: Orchestrates read-only adapters across Round 1 ($\le 60$s) and Round 2 ($\le 3$m); enforces read-only allow-list and marks unreachable sources `"Not Checked"`. | Layer 4 § 4.4, Layer 5 § 3 |
| **Q6.15** | Generating Diagnoses | **Diagnostic Engine**: Tri-stage hybrid evaluation producing structured hypothesis objects with qualitative confidence (`High`/`Medium`/`Low`) and mandatory timestamped evidence links. | Layer 4 § 4.5, § 4.6, Layer 5 § 4 |
| **Q6.16** | Independent Review of Diagnosis | **Diagnosis Reviewer**: Deterministic rule validation and empirical sandbox reproduction. Multi-round LLM-vs-LLM debate loops deferred as post-MVP research. | [ADR-11](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-11-deterministic-diagnosis-verification-over-llm-debate.md) |
| **Q6.17** | Running AI-Generated Code Against Tests | **Sandbox Verifier**: Platform-owned runner executing 4-step protocol in an ephemeral Docker container. Existing tests mounted read-only; deterministic grading via exit codes. | Layer 4 § 4.10, Layer 5 § 4 |
| **Q6.18** | Managing Human Approval | **Approval Manager**: Renders deterministic review dossier; binds approval to composite SHA-256 fingerprint; enforces 1-hour TTL and blocks deploy-author self-approval. | [ADR-12](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-12-composite-sha256-patch-and-state-fingerprint.md) |
| **Q6.19** | Executing Approved Action | **Deployment Executor**: Isolated process holding exclusive CI/CD deploy credentials. Verifies hash and TTL before triggering 10% canary rollout with automated rollback. | [ADR-08](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-08-modular-monolith-and-three-process-runtime.md), Layer 5 § 6 |
| **Q6.20** | Independently Verify Recovery | **Recovery Verification Component**: Continuously evaluates error rates, p50/p95/p99 latency against baseline over 15–30 min; triggers automated rollback on degradation. | Layer 4 § 4.13 |
| **Q6.21** | Store Large Artifacts | **Artifact Storage Component**: Durable object storage (S3-compatible / local volume) for raw logs and diffs; PostgreSQL retains metadata, pointers, and sanitized key evidence snapshots. | Layer 4 § 4.15, Layer 5 § 9 |
| **Q6.22** | Platform Self-Monitoring | **Platform Observability Component**: Standardized on OpenTelemetry for platform health (stage latencies, worker heartbeats, token spend, queue depth) + external synthetic 60s probe. | Layer 4 § 4.16, § 4.26 |
| **Q6.23** | HTTP API Framework | **FastAPI** (with Pydantic): Thin HTTP adapter handling routing, request validation, and OpenAPI documentation; zero domain logic in handlers. | [ADR-13](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-13-fastapi-http-gateway-and-pydantic-validation.md) |
| **Q6.24** | Persistent Database Technology | **PostgreSQL**: Stores relational incident models, transactional `SKIP LOCKED` job queue, stage checkpoints, and insert-only append-only audit ledger. | [ADR-10](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-10-postgresql-backed-concurrency-and-job-queue.md) |

---

## Detailed Section Breakdown

### Part A: Architecture Decisions (Q6.1 – Q6.8)

#### Q6.1: 10-Minute Investigation vs HTTP Request Timeout
- **User View:** Acknowledge immediately; track progress in database; emit updates at key milestones (started, sandbox done, awaiting approval, PR/deploy done).
- **Consensus & Synthesis:**
  - One HTTP request cannot return multiple successive responses over time. The HTTP intake request must immediately terminate after durable registration.
  - The API does not send outbound HTTP requests to internal components; PostgreSQL is the single source of truth.
  - Outbound notifications (Slack, PagerDuty) notify human engineers at key lifecycle milestones.
- **Approved Decision:** Request-Acknowledge + Database-Backed Asynchronous Worker ([ADR-09](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-09-intake-api-handshake-and-enrichment-fallback.md)). Webhook intake commits incident and dispatch job atomically, returning `200 OK` in p95 $\le$ 500 ms. Dashboards observe status via periodic API polling (5s interval in MVP).

#### Q6.2: Synchronous vs Asynchronous Operations
- **User View:** Receiving and acknowledging is synchronous; the agent reasoning loop is "synchronous" because steps are dependent; evidence fetching is "asynchronous."
- **Consensus & Synthesis:**
  - Reconciled terminology from first principles: **Synchronous** means the caller waits for immediate return ($< 2$s); **Asynchronous** means the operation continues in the background.
  - Within background workers, operations that depend on each other execute sequentially; operations that are independent (reading config diff, logs, metrics) execute concurrently (parallel).
  - Human approval wait is a suspended checkpoint state where the worker yields execution while preserving the allocated concurrency slot.
- **Approved Decision:**
  - **Synchronous:** Webhook validation, deduplication, atomic DB persistence, status queries, human approval/cancel submissions, emergency kill switch.
  - **Asynchronous:** Evidence collection, model inference, hypothesis ranking, sandbox testing, canary deployment, recovery verification.
  - **Bounded Parallelism:** Independent read-only lookups in Round 1/2 run concurrently with a cap of 4 simultaneous requests.

#### Q6.3: Splitting the Do-Everything API
- **User View:** Split into two APIs (one from intake to sandbox proposal; second to apply patch after approval). Predefined allow-listed read-only commands only; no arbitrary shell.
- **Consensus & Synthesis:**
  - Dividing into "two APIs" recreates the same problem in the first API. The proper architectural seam is by responsibility and credential privilege.
  - The component that reads untrusted logs and communicates with external LLMs must never possess production deployment credentials.
- **Approved Decision:** Three logical roles running from one codebase ([ADR-08](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-08-modular-monolith-and-three-process-runtime.md)):
  1. `api`: Ingress, authentication, dashboard reads, recording approvals. Zero AI keys, zero deployment tokens.
  2. `worker`: Triage, telemetry, LLM reasoning, disposable sandbox testing. Holds AI keys and read-only tokens; zero deployment tokens.
  3. `executor` (`AI Deployment Executor`): Holds CI/CD deployment tokens; verifies SHA-256 patch fingerprint and TTL before triggering release.

#### Q6.4: Background Processing Responsibilities
- **User View:** API accepts incident; background processing performs deduplication and checks, places in queue, and assigns to workers; duplicates merged.
- **Consensus & Synthesis:**
  - Deduplication must execute **synchronously at API intake** within the same database transaction. Deferring deduplication to background workers creates race conditions during alert bursts.
  - Background processing owns execution governance: claiming jobs, executing checkpoints, emitting 60s heartbeats, standby crash takeover, and enforcing budget ($20) and time (15m triage) caps.
- **Approved Decision:** API handles deduplication at intake. Background workers use PostgreSQL `SKIP LOCKED` queue ([ADR-10](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-10-postgresql-backed-concurrency-and-job-queue.md)) to enforce the 3-slot concurrency limit, 20-item priority queue, 10-minute queue aging alert, and automated crash recovery.

#### Q6.5: Operational Problems of Too Many Separate Services
- **User View:** Many services introduce network communication overhead, complex monitoring, and operational burdens unjustified for an MVP.
- **Consensus & Synthesis:**
  - Microservices introduce distributed transaction failure modes across the state machine, approval records, and audit ledger.
  - Increases attack surface by requiring inter-service secrets and network authorization.
- **Approved Decision:** Reject microservices for the MVP. Maintain a single deployable Modular Monolith ([ADR-08](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-08-modular-monolith-and-three-process-runtime.md)). Any future extraction requires meeting explicit, written split triggers (such as independent scaling bottlenecks or separate team ownership).

#### Q6.6 & Q6.7: Organizing the Codebase (Modular Monolith with Ports & Adapters)
- **User View:** Modular monolith; neat separation into folders; avoid messy dependencies.
- **Consensus & Synthesis:**
  - Directory structure alone does not prevent architectural erosion. Strict dependency rules must be codified and enforced automatically.
  - Domain logic must remain independent of external tools via the Ports & Adapters pattern.
- **Approved Decision:** Adopt **Modular Monolith with Ports & Adapters** ([ADR-08](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-08-modular-monolith-and-three-process-runtime.md)). Enforce in CI using `import-linter`:
  - Modules interact solely via public interfaces.
  - Each module owns its database tables.
  - Only adapters interact with outside APIs.
  - Domain and investigation code cannot import deployment code.

#### Q6.8: Separate Processes Sharing One Codebase
- **User View:** Separate workers for different tasks (sandbox, investigation); communicate only via database.
- **Consensus & Synthesis:**
  - Dividing workers by functional stage creates unnecessary specialization and idle resources under a 3-slot cap.
  - Workers should be identical, general-purpose workers capable of executing any pipeline phase from saved checkpoints.
- **Approved Decision:** One repository, one built Docker image, executed as 3 Docker Compose services (`api`, `worker`, `executor`) ([ADR-08](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-08-modular-monolith-and-three-process-runtime.md)). Database migrations are owned centrally (Alembic) and run before application services initialize.

---

### Part B: Component Identification (Q6.9 – Q6.22)

#### Q6.9: Incident Intake Gateway
- **Component:** `Incident Intake Gateway` (within `api` process).
- **Approved Decision:** Authenticates incoming webhooks, validates the mandatory 5 fields, performs composite deduplication, and commits incident and dispatch job transactionally ([ADR-09](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-09-intake-api-handshake-and-enrichment-fallback.md)). Unresolved optional fields transition to `"needs_info"` with auto-enrichment fallback.

#### Q6.10: Incident Record Store / Repository
- **Component:** `Incident Record Store` (PostgreSQL adapter).
- **Approved Decision:** Single owner of all persistent incident state, relationships, checkpoints, and the append-only timeline table. Memory buffering is used strictly during complete database outages (§ 4.18).

#### Q6.11: Workflow Orchestrator (Tracker + Supervisor)
- **Component:** `Workflow Orchestrator`.
- **Approved Decision:** Manages stage transitions across the canonical 9-state machine ([ADR-14](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-14-canonical-9-state-incident-lifecycle-reconciliation.md)). Emits and monitors 60s heartbeats, reassigns orphaned tasks to standby workers upon crash, and trips a circuit breaker after 3 consecutive crashes ([ADR-10](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-10-postgresql-backed-concurrency-and-job-queue.md)).

#### Q6.12: Model Gateway
- **Component:** `Model Gateway`.
- **Approved Decision:** Sole component with external LLM access. Enforces 2-minute timeouts, 3 primary retries + 2 secondary failovers (§ 4.31), per-incident financial caps ($20 / 500k tokens), schema format checking, and bidirectional PII/credential sanitization.

#### Q6.13: Runbook Lookup
- **Component:** `Runbook Lookup`.
- **Approved Decision:** Simple tag- and symptom-indexed lookup service returning static Markdown runbook context. Zero vector databases or specialist embedding models in the MVP (Layer 5 § 2).

#### Q6.14: Evidence Collection Coordinator
- **Component:** `Evidence Collection Coordinator`.
- **Approved Decision:** Orchestrates read-only telemetry adapters through the Execution Harness. Enforces Round 1 ($\le 60$s) and Round 2 ($\le 3$m) sequencing, tags all findings with unique IDs, and explicitly marks unreachable systems `"Not Checked"`.

#### Q6.15: Diagnostic Engine
- **Component:** `Diagnostic Engine`.
- **Approved Decision:** Generates competing hypotheses (top 3 + mandatory `"Other/Unknown"`) with qualitative confidence ratings (`High`, `Medium`, `Low`). Requires at least one timestamped telemetry artifact per candidate cause.

#### Q6.16: Diagnosis Reviewer / Evaluator
- **Component:** `Diagnosis Reviewer`.
- **Approved Decision:** Platform-owned deterministic verification engine evaluating temporal alignment ($\pm 30$s) and symbol matching. Multi-round LLM critic debates are explicitly deferred ([ADR-11](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-11-deterministic-diagnosis-verification-over-llm-debate.md)).

#### Q6.17: Sandbox Verifier
- **Component:** `Sandbox Execution and Verification Component`.
- **Approved Decision:** Provisions disposable Docker container (30-minute TTL, no prod route, no Docker socket) and deterministically executes the 4-step verification protocol. Existing test suites are mounted read-only and immutable.

#### Q6.18: Approval Manager
- **Component:** `Approval Manager`.
- **Approved Decision:** Renders deterministic one-page review package, calculates composite SHA-256 fingerprint ([ADR-12](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-12-composite-sha256-patch-and-state-fingerprint.md)), enforces 1-hour TTL, and blocks the incident-triggering deploy author from self-approving.

#### Q6.19: Deployment Executor
- **Component:** `AI Deployment Executor`.
- **Approved Decision:** Runs as an isolated process holding exclusive CI/CD deploy credentials. Re-verifies live state fingerprint match ($H_{\text{live}} == H_0$) and TTL before triggering canary rollout via CI/CD.

#### Q6.20: Recovery Verification Component
- **Component:** `Recovery Verification Component`.
- **Approved Decision:** Evaluates error rates and latency percentiles (p50, p95, p99) against historical baselines over a sustained 15–30 minute observation window; triggers automated rollback on degradation.

#### Q6.21: Artifact Storage Component
- **Component:** `Artifact Storage Component`.
- **Approved Decision:** Backed by durable object storage for large diagnostic dumps, raw diffs, and container logs. PostgreSQL stores only metadata, hashes, and sanitized key evidence snapshots.

#### Q6.22: Platform Observability Component
- **Component:** `Platform Observability Component`.
- **Approved Decision:** Uses OpenTelemetry SDK to instrument the AI SRE platform itself (intake latency, stage durations, queue lag, model tokens/costs) alongside an external 60s blackbox ping probe.

---

### Part C: Technology Selection (Q6.23 – Q6.24)

#### Q6.23: HTTP API Framework — FastAPI with Pydantic
- **Approved Decision:** Adopt **FastAPI** with Pydantic for the `api` service ([ADR-13](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-13-fastapi-http-gateway-and-pydantic-validation.md)).
- **First Principles Justification:** Provides asynchronous non-blocking request handling, automatic schema validation and serialization via Pydantic, and automatic OpenAPI documentation while integrating natively with Python 3.11+.

#### Q6.24: Persistent Database — PostgreSQL
- **Approved Decision:** Confirm **PostgreSQL** as the core relational state store ([ADR-10](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-10-postgresql-backed-concurrency-and-job-queue.md)).
- **First Principles Justification:** Provides ACID transaction guarantees for atomic intake/queueing, row-level locking (`SKIP LOCKED`) for worker coordination without extra broker infrastructure, and strict permission controls for insert-only append-only audit ledgers.
