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
| **Q6.4** | Background Processing Responsibilities | Temporal task queues for worker dispatch; 3 active slots enforced via Temporal worker concurrency settings; 20-queue cap enforced at API intake; budget & timeout enforcement via Temporal timers. | [ADR-10](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-10-temporal-workflow-engine.md) |
| **Q6.5** | Problems with Too Many Separate Services | Avoid premature microservices. Eliminates distributed transactions, multi-hop latency, tracing overhead, and complex service-to-service authentication for a small team. | [ADR-08](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-08-modular-monolith-and-three-process-runtime.md), Layer 2 § 2.2 |
| **Q6.6** | Organizing One Core Without Distributed Complexity | Modular Monolith structured with Ports & Adapters; strict public interfaces; zero cross-module direct table manipulation; CI-enforced dependency checking via `import-linter`. | [ADR-08](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-08-modular-monolith-and-three-process-runtime.md), Layer 3 § 7.2 |
| **Q6.7** | Approach for Fast MVP + Strict Boundaries | Formal adoption of **Modular Monolith with Ports & Adapters** as the primary architectural style for the 6-month MVP. | [ADR-08](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-08-modular-monolith-and-three-process-runtime.md) |
| **Q6.8** | Separate Processes Sharing One Codebase | Single codebase and single built Docker image started as 3 distinct services (`api`, `worker`, `executor`) in Docker Compose sharing domain packages and DB schemas. Alembic manages DB migrations centrally, run before services initialize. | [ADR-08](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-08-modular-monolith-and-three-process-runtime.md), [ADR-10](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-10-temporal-workflow-engine.md) |
| **Q6.9** | Receive & Validate Incoming Incidents | **Incident Intake Gateway** (inside `api`): Validates 5 mandatory fields, deduplicates, atomically registers incident + job, and replies `200 OK`. Missing context marked `"needs_info"` (never dropped). | [ADR-09](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-09-intake-api-handshake-and-enrichment-fallback.md) |
| **Q6.10** | Persistent Incident Records | **Incident Repository / Record Store**: Encapsulates PostgreSQL state and insert-only append-only timeline table. In-memory buffering used only during total DB outage fallback. | Layer 4 § 4.15, § 4.18, Layer 5 § 9 |
| **Q6.11** | Durable Workflow Progress After Crash | **Temporal Workflow Engine**: Durable execution history survives worker crashes; activities resume from last completed step without manual checkpoint code; idempotent retries and TTL timers built-in; drives canonical 9-state machine. | [ADR-10](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-10-temporal-workflow-engine.md), [ADR-14](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-14-canonical-9-state-incident-lifecycle-reconciliation.md) |
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
| **Q6.24** | Persistent Database Technology | **PostgreSQL**: Stores relational incident models, structured relationships, JSONB evidence payloads, and insert-only append-only audit ledger. Business-domain state owner (Temporal owns workflow execution state). | [ADR-10](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-10-temporal-workflow-engine.md) |
| **Q6.25** | Schema Migration Management | **Alembic**: Version-controlled numbered migration files (`upgrade()` / `downgrade()`). Run as a one-shot step before services start. CI gate enforces correctness. | ADR-08, Established in Q6.8 |
| **Q6.26** | Durable Workflow Engine | **Temporal**: Durable execution history survives crashes; activities resume from last completed step; built-in retry policies, TTL timers, and Signals for approval wait. Overrides previous PostgreSQL orchestrator decision. | [ADR-10](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-10-temporal-workflow-engine.md) |
| **Q6.27** | Initial Model Providers | **Anthropic Claude** (primary) + **OpenAI GPT-4o** (secondary failover). Different companies ensure a full provider outage doesn't take both down. API-based only; self-hosted deferred. | Layer 4 § 4.21, § 4.31 |
| **Q6.28** | Provider Abstraction Layer | **Provider Adapter pattern** inside Model Gateway: `ModelRequest → provider-specific call`; `provider response → ModelResponse`. Investigation code never imports provider SDKs directly. | Layer 6 Q6.12, Ports & Adapters (ADR-08) |
| **Q6.29** | Runbook Retrieval | **BM25** keyword ranking. In-process, no separate server, no embedding model. Post-MVP trigger: corpus >200 docs or recall <80%. | Layer 6 Q6.13 |
| **Q6.30** | Execution Isolation | **Ephemeral Docker containers**: fresh per run, 4-step protocol, existing tests read-only, no prod network, no Docker socket, 30-min TTL. | Layer 4 § 4.10, Layer 6 Q6.17 |
| **Q6.31** | Observability Stack | **OpenTelemetry** (instrument) + **Prometheus** (store/query) + **Grafana** (dashboard). Three separate responsibilities. | Layer 6 Q6.22 |
| **Q6.32** | Local Dev Environment | **Docker Compose**: all services (api, worker, executor, postgres, temporal) in one file; `docker compose up` starts everything. Postgres first → Alembic migrations → app services. | ADR-08 |
| **Q6.33** | Technology Avoidance List | Kafka, Redis, Kubernetes, vector DB, self-hosted LLM all deferred with explicit trigger conditions. Full candidate stack signed off. | Layer 2 § 2.2, Layer 6 decisions above |
| **Q6.34** | Visual Representation of Complete Architecture | **C4 Model (Context → Container/Process → Component)** paired with end-to-end sequence flows and Trust Boundary overlays. Replaces generic/AWS diagrams to accurately reflect the 3-process Modular Monolith on Docker Compose. | [ADR-08](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-08-modular-monolith-and-three-process-runtime.md), Layer 6 Deliverable |
| **Q6.35** | API, PostgreSQL & Temporal Diagram | **Dependency & Sync/Async Boundary Diagram**: Solid sync arrows (`API → PostgreSQL`, `API → Temporal` start/signal); dashed async pull arrows (`Worker/Executor` polling `Temporal`); purely passive `PostgreSQL` owning business state while `Temporal` owns workflow state. | [ADR-09](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-09-intake-api-handshake-and-enrichment-fallback.md), [ADR-10](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-10-temporal-workflow-engine.md) |
| **Q6.36** | Agent ↔ Model Gateway ↔ Providers Diagram | **Ports & Adapters Fan-Out Diagram**: `Diagnostic Engine` (in `worker`, NOT `api`) calls internal `ModelGateway` port (`ModelRequest`/`ModelResponse`); Gateway enforces sanitization/timeout/budget, then routes via `AnthropicAdapter` (**Claude Primary**) and `OpenAIAdapter` (**GPT-4o Failover**). | Layer 4 § 4.21, § 4.31, Layer 6 Q6.12, Q6.27, Q6.28 |
| **Q6.37** | Sandbox, Approval Service & Trusted Dispatcher Diagram | **Trust Boundary + Blocked-Path Diagram**: Explicit Untrusted vs. Trusted zones with blocked direct path (`Sandbox -.-x Executor`); `Sandbox Verifier` mediates untrusted test results; `Approval Service` (including Human Approver gate) persists `APPROVED + H_0 SHA-256 + 1h TTL`; `AI Deployment Executor` pulls from PostgreSQL and re-verifies `H_live == H_0` before canary release. | [ADR-12](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-12-composite-sha256-patch-and-state-fingerprint.md), Layer 5 § 4, § 6 |
| **Q6.38** | Logical Modules vs. Independently Deployed Processes | **Nested Process + Module View**: Outer boxes show runtime container boundaries (`api`, `worker`, `executor`, `Ephemeral Sandbox`) launched from 1 Docker image; inner boxes show logical modules (`import-linter` enforced). Cross-process coordination happens strictly via `PostgreSQL` and `Temporal` (never direct bridge RPC between containers). | [ADR-08](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-08-modular-monolith-and-three-process-runtime.md), [ADR-10](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-10-temporal-workflow-engine.md) |
| **Q6.39** | Identifying Security Boundaries | **4-Axis Boundary Rule**: Separate process/container isolation whenever components differ in (1) Untrusted input/code execution, (2) Secret scope, (3) Production write authority, or (4) Network exposure. Secrets live in an External Secrets Store and are injected out-of-band per process (`api`, `worker`, `Ephemeral Sandbox`, `executor`). | [ADR-08](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-08-modular-monolith-and-three-process-runtime.md), Layer 5 § 2, § 3, § 7 |

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
- **Approved Decision:** API handles deduplication at intake. Background workers use **Temporal** task queues ([ADR-10](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-10-temporal-workflow-engine.md)) to enforce the 3-slot concurrency limit (`max_concurrent_workflow_task_executors = 3`), 20-incident queue cap at intake, 10-minute starvation alert, and durable crash recovery *(overriding the earlier PostgreSQL `SKIP LOCKED` queue proposal)*.

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

#### Q6.11: Workflow Orchestrator (Temporal)
- **Component:** `Workflow Orchestrator` (**Temporal**).
- **Approved Decision:** Manages stage transitions across the canonical 9-state machine ([ADR-14](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-14-canonical-9-state-incident-lifecycle-reconciliation.md)) via Temporal durable execution history, task queues, activity retries, TTL timers, and Signals ([ADR-10](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-10-temporal-workflow-engine.md)). *(Updated when Temporal overrode the earlier PostgreSQL orchestrator proposal.)*

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
- **Approved Decision:** Confirm **PostgreSQL** as the core relational state store ([ADR-10](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-10-temporal-workflow-engine.md)).
- **First Principles Justification:** Provides ACID transaction guarantees for atomic incident registration, structured relational model for incident ↔ evidence ↔ hypothesis relationships, JSONB for semi-structured evidence payloads, and strict permission controls for insert-only append-only audit ledgers. PostgreSQL continues to own all business-domain state; Temporal owns workflow execution state.

---

### Part C (continued): Technology Selection (Q6.25 – Q6.33)

#### Q6.25: Schema Migrations — Alembic
- **User View:** Keep a change log tracking who changed what and when; a senior person reviews whether each change is safe.
- **What was missing:** The formal name (database migration), the tool (Alembic), and that safety comes from automated versioned sequencing — not manual admin review.
- **Consensus & Synthesis:**
  - Schema changes are version-controlled migration files, each with an `upgrade()` and `downgrade()` function. Applied in fixed numeric order. The tool tracks which have been applied in an `alembic_version` table.
  - Safety guarantees: ordering (version numbers prevent apply-out-of-order), idempotency (already-applied migrations skipped), reversibility (`downgrade()` rollback), and CI gate (migrations run before services start, failing the deploy on error).
  - Admin review is replaced by: PR review in version control + automated CI enforcement.
- **Approved Decision:** **Alembic** as the migration tool. Migrations run as a one-shot step before Docker Compose services initialize (established in Q6.8 / ADR-08).

#### Q6.26: Workflow Engine — Temporal (Overrides PostgreSQL Orchestrator)
- **User View:** "Process crash = worker/thread dies midway. Kafka or Temporal supports this. Temporal accepted after watching intro video."
- **What was missing:** Kafka is a message bus (not a state store); Temporal is a durable execution engine — fundamentally different. Temporal's value is resuming from the last completed activity, not replaying messages.
- **Conflict resolved:** Previous Q6.11 / ADR-10 decision (PostgreSQL-backed custom orchestrator) is **overridden**. ADR-10 (PostgreSQL version) deleted. New [ADR-10](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-10-temporal-workflow-engine.md) adopts Temporal.
- **Approved Decision:** **Temporal** as the durable workflow engine. Durable execution history survives worker crashes; activities resume from last completed step; built-in retry policies, TTL timers, and Signals for human approval wait. PostgreSQL continues to own business-domain state (incident records, audit ledger); Temporal owns workflow execution state.

#### Q6.27: Initial Model Providers — Anthropic Claude + OpenAI GPT-4o
- **User View:** "Claude or GPT — use API key, don't host own GPU. Can add self-hosted open-weights for enterprise."
- **What was missing:** Why two providers (failover, not just choice); why these two specifically (function/tool calling, long context, \$20 cap compatibility); self-hosted deferred explicitly by Layer 2.
- **Approved Decision:** **Anthropic Claude** (primary) + **OpenAI GPT-4o** (secondary failover). Both are API-based, support tool/function calling, support long context (≥100k tokens). Different companies ensures a full provider outage doesn't take both down. Model Gateway (Q6.12) manages switching transparently — investigation code never talks to providers directly. Self-hosted open-weights deferred to post-MVP.

#### Q6.28: Provider Abstraction — Provider Adapter Pattern (inside Model Gateway)
- **User View:** "We can use an adapter."
- **What was missing:** What the adapter hides (both request format AND response format differ between providers); where it lives (inside Model Gateway component from Q6.12); what the rest of the system sees (single internal `ModelRequest` / `ModelResponse` schema).
- **Approved Decision:** Provider-specific adapters inside the **Model Gateway** port. Each adapter maps internal `ModelRequest → provider-specific API call` and `provider response → ModelResponse`. The Diagnostic Engine imports only the Model Gateway port — never the Anthropic or OpenAI SDK directly. Consistent with the Ports & Adapters pattern (Q6.6/Q6.7).

#### Q6.29: Runbook Retrieval — BM25 Keyword Search
- **User View:** "BM25 — we need exact things. Vector database is expensive initially, introduces complexity we don't want."
- **What was missing:** Why BM25 specifically (runs in-process, no separate server, no embedding model); post-MVP trigger for vector search.
- **Approved Decision:** **BM25** keyword ranking as the MVP retrieval strategy. Runbooks indexed by service name and symptom keywords. No vector database, no embedding model. Aligns with Layer 6 log Q6.13 (simple tag/keyword lookup, zero vector DBs in MVP). **Post-MVP trigger:** introduce vector search when runbook corpus exceeds ~200 documents or keyword recall drops below 80%.

#### Q6.30: Execution Isolation — Ephemeral Docker Container
- **User View:** "Network isolation + process isolation. cgroups for filesystem/process isolation. Use Docker — reconstruct error in Docker container."
- **What was missing:** Container must be ephemeral (destroyed after each run); existing tests mounted read-only; no production network route; no Docker socket mounted; 30-minute TTL; isolation is the mechanism — the 4-step verification protocol is what makes it safe.
- **Approved Decision:** **Ephemeral Docker containers** for sandbox isolation. Each verification run: fresh container → 4-step protocol (reproduce → patch → regression test → blast-radius suite) → container destroyed. Existing test suites mounted read-only; no production network route; no Docker socket; 30-minute force-kill TTL. Aligns with Layer 6 log Q6.17 / Layer 4 § 4.10.

#### Q6.31: Observability Stack — OpenTelemetry + Prometheus + Grafana
- **User View:** "OpenTelemetry — provides instrumentation, plugs into codebase, extracts metrics. I don't know about it, don't have the full answer. Prometheus yes. Grafana — don't know what that means."
- **What was missing:** The three-layer stack: OTel (instrument + export) → Prometheus (store + query) → Grafana (visualize/dashboard). They are three separate responsibilities; none can replace the others.
- **Approved Decision:** **OpenTelemetry SDK** for instrumentation (stage latencies, queue depth, token spend, worker heartbeat metrics), **Prometheus** for time-series metric storage and querying, **Grafana** for dashboard visualization. Aligns with Layer 6 log Q6.22.

#### Q6.32: Local Development Environment — Docker Compose
- **User View:** "Docker Compose. I haven't worked with it. I just have the name."
- **What was missing:** Why it solves the problem — single `docker-compose.yml` defines all services (api, worker, executor, postgres, temporal), their startup order, environment variables, and network — `docker compose up` starts everything.
- **Approved Decision:** **Docker Compose** for local development. All 3 processes + PostgreSQL + Temporal server defined in one file. Startup order: PostgreSQL first → Alembic migrations → then app services. No Kubernetes needed at MVP scale.

#### Q6.33: Technologies to Avoid Initially + Candidate Stack Sign-Off
- **User View:** "Introduce technology only with solid reason: cost, better performance. SQLAlchemy for ORM (noted SQLModel as alternative). Alembic for migrations. Temporal accepted. Prometheus, OTel, Docker Compose, UV all accepted."
- **Clarification — SQLAlchemy vs SQLModel:** SQLModel is a convenience wrapper *on top of* SQLAlchemy — not an alternative at the same layer. SQLAlchemy is the foundation. Either is acceptable; SQLAlchemy is the safer choice for MVP (more battle-tested). Worksheet candidate (SQLAlchemy) accepted.
- **Technologies to avoid and their trigger conditions:**

| Technology | Avoid because | Introduce when |
| :--- | :--- | :--- |
| Kafka / RabbitMQ | Message broker overhead — PostgreSQL outbox handles MVP event volume | >1000 events/min OR fan-out to 5+ independent consumers |
| Redis | Extra cache layer — PostgreSQL handles MVP query load | Cache hit rate analysis shows DB query bottleneck |
| Kubernetes | 3-process monolith on 1 host needs no orchestration | Multi-host deployment or auto-scaling requirement |
| Vector DB | Embedding model + server overhead — BM25 handles small corpus | Runbook corpus >200 docs OR keyword recall <80% |
| Self-hosted LLM | GPU ops expertise and cost — API providers cover MVP | Enterprise data-residency requirement OR API per-token cost exceeds self-host at scale |

- **UV:** Python package manager (replaces pip + venv) — dramatically faster dependency resolution and lockfile management. Accepted.
- **Approved candidate stack (full sign-off):**

| Responsibility | Technology | Decision |
| :--- | :--- | :--- |
| Backend API | FastAPI | ✅ Accept |
| Validation | Pydantic | ✅ Accept |
| Database | PostgreSQL | ✅ Accept |
| ORM | SQLAlchemy | ✅ Accept |
| Migrations | Alembic | ✅ Accept |
| Workflow engine | **Temporal** | ✅ Accept (overrides PostgreSQL orchestrator — ADR-10 updated) |
| Model providers | Claude + GPT-4o | ✅ Accept |
| Retrieval | BM25 | ✅ Accept |
| Execution | Ephemeral Docker sandbox | ✅ Accept |
| Metrics | Prometheus | ✅ Accept |
| Dashboard | Grafana | ✅ Accept |
| Tracing | OpenTelemetry | ✅ Accept |
| Dev environment | Docker Compose | ✅ Accept |
| Dependency mgmt | UV | ✅ Accept |

---

### Part D: Architecture Views (Q6.34 – Q6.37)

#### Q6.34: Complete Architecture Visual Representation — C4 Model + Sequence & Trust Overlays
- **User View:** Represent architecture visually using flowcharts and "AWS models."
- **What was missing & conflict resolved:** A single flowchart cannot convey system context, runtime processes, and internal code boundaries at once; AWS vendor diagrams conflict with our Modular Monolith running as 3 Docker Compose processes (`api`, `worker`, `executor`) per [ADR-08](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-08-modular-monolith-and-three-process-runtime.md).
- **Approved Decision:** Use **C4 Model zoom levels** paired with behavioral flows and security overlays:
  1. **Context Diagram (Level 1):** Platform boundary, On-call Engineer, and external integrations (Monitoring/Webhooks, Git/CI-CD, Telemetry, LLM Providers).
  2. **Container / Process Diagram (Level 2):** The 3 runtime processes (`api`, `worker`, `executor`), `PostgreSQL`, `Temporal`, `Artifact Storage`, and connection protocols.
  3. **Component Diagram (Level 3):** Internal modules (`Incident Intake Gateway`, `Diagnostic Engine`, `Model Gateway`, `Sandbox Verifier`, `Approval Manager`, `Deployment Executor`) communicating strictly via Ports & Adapters.
  4. **Supplemental Sequence & Trust Boundary Views:** End-to-end incident flow showing temporal order plus explicit Untrusted Perimeter vs. Trusted Core boundaries.

#### Q6.35: API, PostgreSQL, and Temporal Communication & Dependencies
- **User View:** Suggested using a flowchart (then moved directly into Q6.36 before detailing the API/PostgreSQL/Temporal dependencies).
- **What was missing:** Explicit ownership boundaries, directional dependency arrows, and sync vs. async separation.
- **Approved Decision:** Represent as a **Sync/Async Dependency Diagram** making four rules explicit:
  1. **Sync calls (solid arrows):** `API` synchronously writes/reads business state in `PostgreSQL` and synchronously calls `Temporal` (`StartWorkflow` / `Signal` / queue-depth check).
  2. **Async pull (dashed arrows):** `Worker` and `Executor` processes asynchronously **poll** `Temporal` task queues — `API` never sends direct requests to `Worker`/`Executor`.
  3. **Passive database:** All SQL arrows point **into** `PostgreSQL`; it never initiates outbound calls.
  4. **Clear state split ([ADR-10](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-10-temporal-workflow-engine.md)):** `PostgreSQL` owns business-domain state (incidents, approvals, append-only `incident_timeline`), while `Temporal` owns workflow execution state (activity history, retries, timers, signals), using a separate schema in PostgreSQL as its persistence backend in MVP.

#### Q6.36: Agent ↔ Model Gateway ↔ Multiple Model Providers
- **User View:** API calls Model Gateway, with 3 branches fanning out to ChatGPT, Claude, and Tinker.
- **Conflicts resolved:**
  1. Caller corrected from `api` to **`Diagnostic Engine` (inside `worker`)** — per Q6.3 and Q6.12, `api` holds zero AI keys and never invokes LLMs.
  2. Providers corrected from 3 (`ChatGPT, Claude, Tinker`) to **2 API providers (`Anthropic Claude` primary + `OpenAI GPT-4o` secondary failover)** per Q6.27 (`Tinker` rejected as a worksheet placeholder; self-hosted open-weights deferred post-MVP).
- **Approved Decision:** Represent as a **Ports & Adapters Fan-Out Diagram**:
  - `Diagnostic Engine` (in `worker`) → internal `ModelGateway` port (`ModelRequest` / `ModelResponse`) — zero provider SDK imports outside the gateway.
  - Inside `Model Gateway`: bidirectional sanitization, 2-minute timeout, JSON schema validation, and \$20 / 500k token budget enforcement sit before provider routing.
  - Fan-out uses `AnthropicAdapter` (**Primary**, 3 retries) with dashed failover edge to `OpenAIAdapter` (**Secondary Failover**, 2 attempts).

#### Q6.37: Sandbox, Approval Service, and Trusted Dispatcher
- **User View:** Sandbox creates a proposal after verification; Approval Service (with human as part of the approval service) accepts, rejects, or triggers retries; Trusted Dispatcher depends on Approval Service to apply changes.
- **Clarification & conflicts resolved:**
  - **Proposal & verification ownership:** The Ephemeral Docker Sandbox is an untrusted, passive container — it executes tests and emits untrusted stdout/exit codes. The **`Diagnostic Engine`** drafts the candidate diff, the platform-owned **`Sandbox Verifier`** runs/grades the 4-step verification, and the **`Approval Service` (`Approval Manager`)** assembles the dossier.
  - **Human inside Approval Service:** Confirmed — the Authorized Human Approver (via Email/Password + RBAC, barred from self-approving their own triggering deploy) is the decision-maker inside the Approval Service gate; the service persists the decision rather than deciding autonomously.
- **Approved Decision:** Represent as a **Trust Boundary & Fail-Closed Verification Diagram**:
  1. **Two security zones + blocked path:** Explicitly draw a crossed-out forbidden path (`Sandbox -.-x AI Deployment Executor / Prod CI/CD`) proving zero network route and zero production credentials inside the sandbox.
  2. **Mediated gate:** `Ephemeral Sandbox` → (untrusted exit code/stdout) → `Sandbox Verifier` → `Approval Service` (dossier + Human Approver click) → persists `APPROVED`, composite SHA-256 fingerprint ($H_0$), and 1-hour TTL in `PostgreSQL`.
  3. **Pull & re-verify at Trusted Dispatcher:** `AI Deployment Executor` (exclusive holder of CI/CD deploy tokens) fetches the approval record directly from `PostgreSQL`, recomputes $H_{\text{live}}$, verifies $H_{\text{live}} == H_0$ and `TTL <= 1h`, and fails closed if tampered or expired before triggering 10% canary rollout.

#### Q6.38: Distinguishing Logical Modules vs. Independently Deployed Processes
- **User View:** Logical modules are folders in the codebase; independently deployed processes are 3 separate containers (`api`, `worker`, `executor`) communicating via a virtual network bridge.
- **Conflict resolved:** Corrected the "virtual bridge communication" misconception — containers share a Docker Compose network only to reach `PostgreSQL` and `Temporal`, never to make direct `api ↔ worker ↔ executor` RPC calls (per Q6.1, Q6.8, Q6.35). Clarified that all 3 runtime processes run from **one codebase and one built Docker image** with separate entrypoints and injected secrets ([ADR-08](file:///c:/Users/diwas/OneDrive/Desktop/Projects/ai-sre/adr/ADR-08-modular-monolith-and-three-process-runtime.md)).
- **Approved Decision:** Distinguish them visually using **nested container/module diagrams and arrow semantics**:
  - **Outer process boxes:** Runtime containers (`api`, `worker`, `executor`, `Ephemeral Sandbox`) labeled with entrypoint and secret profile.
  - **Inner nested boxes:** Logical Python modules executing inside each process (`import-linter` enforced in CI).
  - **Arrow semantics:** Thin internal arrows represent in-process Python calls; cross-box arrows represent network/IPC boundaries through `PostgreSQL`, `Temporal`, or the `Docker` engine.

#### Q6.39: Identifying Which Components Need Separate Security Boundaries
- **User View:** Separate components into two categories: non-deterministic components and components that store/reveal credentials.
- **Refinement & conflict resolved:** Per Layer 5 § 7, components must **not** store raw credentials internally — secrets live in an External Secrets Store and are injected out-of-band per runtime process. Expanded the two-bucket intuition into a **4-axis classification rule**.
- **Approved Decision:** Apply the **4-Axis Security Boundary Rule** — isolate components into separate OS/container boundaries whenever they differ in:
  1. **Untrusted input or code execution** (external webhooks/logs, LLM responses, `Ephemeral Sandbox` test code).
  2. **Credential & secret scope** (`api` auth-only; `worker` LLM + read-only telemetry; `Sandbox` zero secrets; `executor` exclusive prod CI/CD tokens).
  3. **Production write authority** (read-only vs. non-prod write vs. production canary deploy).
  4. **External network exposure** (public webhook ingress vs. internal-only execution).
