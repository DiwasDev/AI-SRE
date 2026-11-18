# Layer 3 — Architecture & Design: Conversation Log

| Questions | Agent Conversation Link / Summary | Approved Decision | Open Unknowns & Owner | Artifact Section |
| :--- | :--- | :--- | :--- | :--- |
| **Q3.1: Blank Screen — The Journey of a Request** *(Tracing request flow & where blank screens happen)* | **Team Discussion:**<br>• User view: Browser $\rightarrow$ DNS $\rightarrow$ LB $\rightarrow$ App $\rightarrow$ Response.<br>• Team listed bugs: DNS mismatch, server down, network drop.<br>• Speaker 3 noted network/DNS drops show error pages, not blank screens.<br><br>**Claude & GPT-3.5 Summary:**<br>• **Agreed:** Blank screen is a symptom (empty or crashed render), not a simple network drop. Team missed TLS, CDN, DB dependencies, and browser rendering.<br>• **Claude:** Strictly enforced Layer 2 (Python backend only; 1 tool). Flagged client-only blank screens will be missed without synthetic checks.<br>• **GPT-3.5:** Left frontend diagnostics open; used generic architecture. | **Approved Decision:**<br>• **Standard 9-Step Journey:** Browser $\rightarrow$ DNS $\rightarrow$ TLS $\rightarrow$ CDN/Edge $\rightarrow$ Gateway/LB (Nginx) $\rightarrow$ Python App $\rightarrow$ DB & Config $\rightarrow$ HTTP Response $\rightarrow$ Browser Render.<br>• **In-Scope for MVP (Steps 5–7):** Gateway, Python app, and config/DB via 1 monitoring tool.<br>• **Out-of-Scope (Steps 1–4, 9):** Browser JS crashes, DNS, CDN, TLS certs. Platform flags as out-of-scope and escalates to human.<br>• **Reference Scenario:** Bad config deploy $\rightarrow$ Python HTTP 500 $\rightarrow$ unhandled blank page. | **Unknowns:**<br>1. Target app rendering type (server-side vs. frontend SPA).<br>2. Monitoring tool visibility into gateway logs.<br><br>**Assumption:** Blank screens in MVP come from server-side 500 errors.<br>**Owner:** Tech Lead (Sprint 2 validation) | Layer 3 — Architecture & Design<br>§ 3.1 Request Journey & Diagnostic Boundaries |
| **Q3.2: Thousands of Simultaneous Requests** *(Component responsibilities under high load)* | **Team Discussion:**<br>• User: Queue requests, balance load across servers.<br>• Team: LB spreads traffic with health checks, Redis caches reads, Celery runs async jobs, DB indexes, rate limit bots.<br>• Capacity is unknown.<br><br>**Claude & GPT-3.5 Summary:**<br>• **Agreed:** LB $\ne$ app processing. Concurrency requires timeouts, worker pools, DB connection limits, and graceful HTTP 503 backpressure.<br>• **Claude:** Tied to Layer 2 read-only limits (cannot scale servers; must tag "likely traffic overload" and hand off). Group alert storms (Layer 2.17).<br>• **GPT-3.5:** Broad architectural diagram; treated auto-scaling and caching as general design choices. | **Approved Decision:**<br>• **Component Responsibilities:** LB distributes & isolates unhealthy nodes; Python app manages parallel workers with timeouts and sheds excess load with HTTP 503; DB pools connections; Queue handles async tasks; Rate limiting caps abuse.<br>• **Platform Boundary (Layer 2):** Platform is read-only; cannot scale servers. If traffic spikes before errors without a deploy, label "Likely Traffic Overload" and hand off to human.<br>• **Alert Storms:** Surging traffic alerts grouped into 1 incident. | **Unknowns:**<br>Target app baseline and peak traffic capacity (to be established by load test, not guessed).<br><br>**Assumption:** Target app runs on multiple worker instances behind an active LB.<br>**Owner:** DevOps Engineer (Sprint 2) | Layer 3 — Architecture & Design<br>§ 3.2 Concurrency & Load Responsibilities |
| **Q3.3: A Request Returns an Error** *(Inside-the-app execution flow and failure handling)* | **Team Discussion:**<br>• Request travels: Routing $\rightarrow$ Validation $\rightarrow$ Code $\rightarrow$ DB $\rightarrow$ Response; failure halts later steps.<br>• 4xx is client-side; 5xx is server-side.<br>• Corrected misconception: Speaker 2 claimed 5xx means config error (incorrect; config is only 1 cause).<br><br>**Claude & GPT-3.5 Summary:**<br>• **Agreed:** Clear 7-stage internal pipeline. Failure halts execution. Sensitive stack traces belong in internal logs, not client responses.<br>• **Claude:** 5xx is an umbrella clue, not proof of config error. MVP focuses strictly on 5xx; defer 4xx surges.<br>• **GPT-3.5:** Cataloged all status codes generically; raised trace vs. log observability level. | **Approved Decision:**<br>• **7-Stage Internal Pipeline:** Routing $\rightarrow$ Auth (401) $\rightarrow$ Authz (403) $\rightarrow$ Validation (400) $\rightarrow$ Business Logic (500) $\rightarrow$ Dependencies (500/502/503) $\rightarrow$ Error Logging & Response.<br>• **Status Code as Clue:** 500 signals server failure, not proof of config bug. Platform tests alternative hypotheses before proposing fix.<br>• **MVP Scope:** Restricted to 5xx server errors; 4xx client errors and network drops excluded from MVP triage.<br>• **Reference Scenario:** Config bug causes unhandled Python exception logged as 500. | **Unknowns:**<br>1. Target Python web framework & error handler structure.<br>2. Error log granularity.<br><br>**Assumption:** Target app logs stack trace, timestamp, and request ID for every 500.<br>**Owner:** Tech Lead (Sprint 2 validation) | Layer 3 — Architecture & Design<br>§ 3.3 Request Lifecycle & Error Handling |
| **Q3.4: External Communication Interface** *(Receiving requests from external clients)* | **Team Discussion:**<br>• Team agreed on API / HTTP API; mentioned SDKs and microservices.<br>• Diwas: "API for external clients".<br>• Gap: Named a generic concept rather than an architectural component.<br><br>**Claude & GPT-3.5 Summary:**<br>• **Agreed:** Dedicated ingress layer (API Gateway / HTTP REST API).<br>• **Claude:** Tied directly to Layer 2 (Q2.8) and security boundary (no direct DB/internal access).<br>• **GPT-3.5:** Emphasized routing, auth, and rate limiting. | **Approved Decision:**<br>• **API Gateway / HTTP REST API Layer** is the single external communication component.<br>• All external callers (monitoring webhooks, UI dashboard, CI/CD, CLI) pass through it.<br>• Direct access to database or internal modules is strictly prohibited.<br>• Centralizes authentication, validation, and rate limiting. | **Unknowns:** None (resolved in Layer 2).<br><br>**Assumption:** External systems integrate over standard HTTP/REST.<br>**Owner:** Tech Lead (Sprint 3 validation) | Layer 3 — Architecture & Design<br>§ 3.4 Ingress Architecture & API Gateway |
| **Q3.5: Persistent Customer Information** *(Managing state that survives application restarts)* | **Team Discussion:**<br>• Diwas suggested session memory and browser cookies.<br>• Corrected misconception: Cookies and sessions are temporary; they do not provide persistent customer storage.<br><br>**Claude & GPT-3.5 Summary:**<br>• **Agreed:** Database component is the only durable store across restarts. Clear 3-way split: Database (durable records), Session (temporary login state), Cookie (client identifier token).<br>• **Claude:** Tied to Layer 2.21 (PostgreSQL state DB). | **Approved Decision:**<br>• **Database Component (Relational DB / PostgreSQL)** manages persistent customer records across restarts.<br>• Clear separation: Database = durable storage on disk; Session = temporary interaction state; Cookie = client browser session ID.<br>• External clients never access the database directly.<br>• Aligns with Layer 2.21 (PostgreSQL). | **Unknowns:** Specific customer schema & privacy rules.<br><br>**Assumption:** Relational database meets schema consistency and durability needs.<br>**Owner:** Tech Lead (Sprint 2 validation) | Layer 3 — Architecture & Design<br>§ 3.5 State Persistence & Database Component |
| **Q3.6: Deploying a New Application Version** *(Multi-stage release pipeline and verification)* | **Team Discussion:**<br>• Git push $\rightarrow$ CI/CD $\rightarrow$ tests $\rightarrow$ pass deploy / fail rollback.<br>• Diwas noted gradual/canary rollout protects prod.<br>• Clarified: Canary $\neq$ A/B testing.<br><br>**Claude & GPT-3.5 Summary:**<br>• **Agreed:** Immutable build artifact (Docker), pre-deploy sandbox test, gradual canary, auto-rollback on metric degradation.<br>• **Claude:** Highlighted post-deploy monitoring window where AI SRE platform activates on metric degradation.<br>• **GPT-3.5:** Emphasized pre-deployment staging/sandbox validation. | **Approved Decision:**<br>• **4-Phase Release Pipeline:** Code (Git push) $\rightarrow$ CI (tests & Docker build) $\rightarrow$ Pre-Deploy Sandbox (smoke tests) $\rightarrow$ Canary Rollout & Verification.<br>• **Post-Deploy Monitoring Window:** System watches error rates and latency. Healthy $\rightarrow$ complete rollout; Degraded $\rightarrow$ auto-rollback and trigger alert to activate AI SRE Platform.<br>• **Layer 2 Guardrails:** 1 automated CI/CD pipeline (no manual deploys). Platform fixes must clear human gates before release. | **Unknowns:** Specific duration of post-deploy canary window.<br><br>**Assumption:** CI/CD emits deployment webhook events (commit SHA, timestamp, diffs).<br>**Owner:** DevOps Engineer (Sprint 3 validation) | Layer 3 — Architecture & Design<br>§ 3.6 Release Pipeline & Verification Lifecycle |
| **Q3.7: Configuration vs. Application Code** *(Isolating environment variable changes from code bugs)* | **Team Discussion:**<br>• Diwas: Diff shows config changes.<br>• Team: Test same code in two envs; revert env var to verify.<br>• Source code is 100% identical.<br><br>**Claude & GPT-3.5 Summary:**<br>• **Agreed:** Compare config history, test identical artifact, revert to prove causality.<br>• **Claude:** Tied directly to Reference Scenario (deploy changed config $\rightarrow$ HTTP 500s). Platform must store old value alongside new value to propose revert PR.<br>• **GPT-3.5:** Emphasized controlled comparison and masking secrets. | **Approved Decision:**<br>• **4-Step Isolation:** 1. Diff Config $\rightarrow$ 2. Correlate Timestamp (exact second errors spiked) $\rightarrow$ 3. Read Error Logs for config exceptions $\rightarrow$ 4. Sandbox Revert & Verify.<br>• **Platform Rule:** Platform ingests old and new config values from webhook; proposes automated config revert PR.<br>• Supported formats: YAML, JSON, `.env`, env vars. | **Unknowns:** Exact config storage mechanism (Git vs. Secrets Manager).<br><br>**Assumption:** Deployment pipeline captures and exposes config diffs.<br>**Owner:** DevOps Engineer (Sprint 3 validation) | Layer 3 — Architecture & Design<br>§ 3.7 Configuration Triage & Isolation |
| **Q3.8: Investigating and Isolating a Regression** *(Pinpointing breaking changes in a release)* | **Team Discussion:**<br>• Diwas: Look at Git diff to see code changes.<br>• Team: Revert to previous commit, reproduce in test env, use feature flags.<br>• Gap: Diff is only a candidate list; manual guessing is slow.<br><br>**Claude & GPT-3.5 Summary:**<br>• **Agreed:** Baseline against last known good version; reproduce in sandbox; use **Git Bisect** for multi-commit releases.<br>• **Claude:** Distinct rules: Full rollback if safe; Feature flag if other changes must stay. Enforce Layer 2.14 two-gate human approval.<br>• **GPT-3.5:** Highlighted DB migration compatibility during rollbacks. | **Approved Decision:**<br>• **5-Step Workflow:** 1. Identify baseline (last known good) $\rightarrow$ 2. Diff release $\rightarrow$ 3. Isolate commit (direct inspection for 1–2 commits; **Git Bisect** for multi-commit) $\rightarrow$ 4. Reproduce in sandbox $\rightarrow$ 5. Two-Gate Recovery.<br>• **Recovery Rules:** Full rollback via canary if clean; feature flag / revert PR if other changes must stay.<br>• **Layer 2 Guardrails:** AI proposes; humans authorize at Gate 1 and Gate 2. DB migrations must be backward-compatible. | **Unknowns:** Release commit bundle size in target team.<br><br>**Assumption:** Git commits are granular enough for automated bisection.<br>**Owner:** Tech Lead (Sprint 2 validation) | Layer 3 — Architecture & Design<br>§ 3.8 Regression Isolation & Bisection |
| **Q3.9: Declaring and Tracking an Incident** *(Incident declaration triggers & alert noise pre-filter)* | **Team Discussion:**<br>• Diwas: Geography, user count, component criticality.<br>• Team: Revenue impact, critical functions, security breaches.<br>• Gap: Lacked structured quantitative thresholds.<br><br>**Claude & GPT-3.5 Summary:**<br>• **Agreed:** Need objective thresholds to separate real incidents from noise.<br>• **Claude:** Explicitly tied to Layer 2 noise pre-filter to drop non-incidents at \$0 LLM cost.<br>• **GPT-3.5:** Emphasized business impact and customer SLAs. | **Approved Decision:**<br>• Declare incident if **any 1 of 5 conditions** is met: 1. User Impact (>5% active users), 2. Critical Function Down (auth, checkout), 3. Complete Unavailability (100% fail), 4. Security/Compliance Risk, 5. SLA Breach.<br>• **Layer 2 Pre-Filter:** Incoming alerts meeting 0 conditions are logged and dropped at \$0 LLM cost. | **Unknowns:** Target org numerical thresholds for user impact % and SLA.<br><br>**Assumption:** Target org provides numeric thresholds.<br>**Owner:** Engineering Manager (ADR-04) | Layer 3 — Architecture & Design<br>§ 3.9 Incident Triggers & Pre-Filtering |
| **Q3.10: 40% Failure Rate vs. Complete Outage** *(Distinguishing partial degradation from complete outages)* | **Team Discussion:**<br>• 40% $\ne$ 100%; test a few times; check endpoints/services.<br>• Gap: Missed standard terminology and clustering pattern analysis.<br><br>**Claude & GPT-3.5 Summary:**<br>• **Agreed:** Complete outage = 100% failure across all paths. Partial failure = degraded state where some succeed.<br>• **Claude:** Emphasized clustering by endpoint, user tier, or region. Health checks prove process is alive.<br>• **GPT-3.5:** Segmenting error rate by endpoint. | **Approved Decision:**<br>• **3 Verification Checks:** 1. Error rate metric (<100% = degraded), 2. Clustering pattern (isolated to specific endpoints/regions vs. uniform), 3. Health check (`/health` responding 200 OK proves process alive).<br>• **Platform Output:** First alert summary must report error rate, affected endpoints, and user blast radius. | **Unknowns:** None (standard diagnostic capability).<br><br>**Assumption:** Monitoring tool exposes endpoint-level error metrics.<br>**Owner:** Tech Lead (Sprint 2 validation) | Layer 3 — Architecture & Design<br>§ 3.10 Partial Degradation Diagnostics |
| **Q3.11: Latency Degradation (100 ms to 8s)** *(Characterizing severe performance degradation)* | **Team Discussion:**<br>• Performance issue, not outage; causes: DB slowness, traffic, server issues.<br>• Gap: Failed to characterize severity (80x slowdown makes app functionally dead).<br><br>**Claude & GPT-3.5 Summary:**<br>• **Agreed:** Latency incident (High Severity). App alive but unusable.<br>• **Claude:** Ordered investigation: 1. DB locks/pool, 2. External API, 3. Traffic spike, 4. Host CPU/RAM, 5. N+1 queries. Must track p95/p99.<br>• **GPT-3.5:** Separate frontend vs. backend delay. | **Approved Decision:**<br>• **Characterization:** Type = Latency Incident / Performance Degradation; Severity = High (80x baseline).<br>• **Platform Rule:** Platform investigates latency spikes even when HTTP error rate is 0%.<br>• Monitoring tool must supply p50/p95/p99 response times. | **Unknowns:** Target org SLA latency threshold.<br><br>**Assumption:** Monitoring tracks response time percentiles.<br>**Owner:** DevOps Engineer (ADR-03) | Layer 3 — Architecture & Design<br>§ 3.11 Latency & Performance Incidents |
| **Q3.12: Hypothesis-Driven Investigation** *(Scientific method for root-cause analysis)* | **Team Discussion:**<br>• Follow evidence iteratively; check past incidents and recent commits.<br>• Gap: Lacked formal scientific method and confidence scoring.<br><br>**Claude & GPT-3.5 Summary:**<br>• **Agreed:** Structured loop: form hypothesis $\rightarrow$ test against telemetry $\rightarrow$ confirm/reject.<br>• **Claude:** Attach confidence score; if all rejected, stop, declare missing data, and ask human (PRD 1.4 UC5). Never hallucinate.<br>• **GPT-3.5:** Pattern match with runbooks. | **Approved Decision:**<br>• **4-Step Scientific Loop:** 1. Pattern match against historical incidents $\rightarrow$ 2. Form hypothesis with explicit Confidence Score $\rightarrow$ 3. Test hypothesis against specific metrics $\rightarrow$ 4. Iterate or Escalate.<br>• **PRD Guardrail:** If evidence is insufficient, state missing data and ask for help; never guess. | **Unknowns:** Availability of structured runbooks.<br><br>**Assumption:** Diagnostic harness can query specific metrics per hypothesis.<br>**Owner:** Tech Lead (Sprint 2 validation) | Layer 3 — Architecture & Design<br>§ 3.12 Hypothesis-Driven Investigation |
| **Q3.13: Restoring Service (Two-Track Recovery)** *(Immediate mitigation vs. permanent fix)* | **Team Discussion:**<br>• Mukul: Knowing cause $\ne$ fix ready; rollback first, fix properly in separate env.<br>• Gap: Did not formalize parallel execution tracks.<br><br>**Claude & GPT-3.5 Summary:**<br>• **Agreed:** Two distinct parallel tracks: Track A (Stop the bleeding) & Track B (Root-cause fix).<br>• **Claude:** Both tracks require human approval gates. Track A restores uptime; Track B closes incident.<br>• **GPT-3.5:** Feature flags and canary rollbacks. | **Approved Decision:**<br>• **Track A (Immediate Mitigation):** Fast canary rollback, config revert, or feature flag toggle to restore availability immediately.<br>• **Track B (Permanent Fix):** Develop patch on branch, test in sandbox, obtain human approval, deploy.<br>• **Rule:** "Service Restored" $\ne$ "Incident Closed". Incident stays open until Track B verifies baseline. | **Unknowns:** Feature flag tooling availability.<br><br>**Assumption:** Application supports clean rollback without data loss.<br>**Owner:** DevOps Engineer (Sprint 2 validation) | Layer 3 — Architecture & Design<br>§ 3.13 Two-Track Incident Resolution |
| **Q3.14: Returning to Working Version (Rollback)** *(Safe execution of automated rollbacks)* | **Team Discussion:**<br>• "Simply rollback" / assumed capability exists.<br>• Corrected: Rollback is Layer 2 Assumption 5 (architecture-breaking assumption).<br><br>**Claude & GPT-3.5 Summary:**<br>• **Agreed:** Rollback must be automated through CI/CD, not manual SSH/copying.<br>• **Claude:** 5 steps: identify commit $\rightarrow$ trigger CI/CD $\rightarrow$ Gate 2 human approval $\rightarrow$ canary $\rightarrow$ verify baseline recovery.<br>• **GPT-3.5:** Highlighted DB migration compatibility. | **Approved Decision:**<br>• **5-Step Rollback:** 1. Retrieve last stable commit SHA $\rightarrow$ 2. Trigger rollback via CI/CD $\rightarrow$ 3. Gate 2 human approval $\rightarrow$ 4. Canary deployment $\rightarrow$ 5. Post-rollback baseline verification.<br>• Rollback capability must be formally validated in Sprint 3 (ADR-06). | **Unknowns:** Target CI/CD one-click rollback support.<br><br>**Assumption:** Target CI/CD supports automated rollback (Assumption 5).<br>**Owner:** DevOps Engineer (ADR-06) | Layer 3 — Architecture & Design<br>§ 3.14 Automated Rollback Lifecycle |
| **Q3.15: Incident Lifecycle State Machine** *(Tracking operational stages and audit logs)* | **Team Discussion:**<br>• Track in Jira: open $\rightarrow$ in progress $\rightarrow$ PR $\rightarrow$ closed.<br>• Gap: Reinvented lifecycle; PRD already defines 11-state machine.<br><br>**Claude & GPT-3.5 Summary:**<br>• **Agreed:** Full state model with explicit approval gates. Transitions must be queryable and logged.<br>• **Claude:** 11 PRD states; append-only PostgreSQL state table; Closed is only reachable from Verifying.<br>• **GPT-3.5:** Dashboard visualization of states. | **Approved Decision:**<br>• **11-State Model:** Detected $\rightarrow$ Notified $\rightarrow$ Investigating $\rightarrow$ DiagnosisProposed $\rightarrow$ FixProposed $\rightarrow$ **AwaitingFixApproval** $\rightarrow$ SandboxTesting $\rightarrow$ **AwaitingDeployApproval** $\rightarrow$ Deployed $\rightarrow$ Verifying $\rightarrow$ Closed.<br>• **Persistence:** Append-only PostgreSQL table (Layer 2.21); all transitions logged with timestamp and actor ID. | **Unknowns:** None (settled in PRD and Layer 2).<br><br>**Assumption:** PostgreSQL manages state machine transitions reliably.<br>**Owner:** Tech Lead (Sprint 2 validation) | Layer 3 — Architecture & Design<br>§ 3.15 Lifecycle State Machine |
| **Q3.16: Escalation and Handoff Mechanics** *(Transferring responsibility when first response stalls)* | **Team Discussion:**<br>• Escalate to senior with context of what was tried.<br>• Prateek: AI SRE is the junior/first responder.<br><br>**Claude & GPT-3.5 Summary:**<br>• **Agreed:** AI SRE is the first responder; handoff is a core product deliverable, not a failure.<br>• **Claude:** 3 escalation triggers: No Expertise, No Access, Time SLA Exceeded (15 min). Transfer immutable evidence ledger.<br>• **GPT-3.5:** Clear handoff documentation. | **Approved Decision:**<br>• **3 Escalation Triggers:** 1. No Expertise $\rightarrow$ service owner with hypotheses & ruled-out tests; 2. No Access $\rightarrow$ DevOps lead with specific missing IAM permission; 3. Time SLA Exceeded $\rightarrow$ on-call senior with full ledger.<br>• **Handoff Contract:** Incoming engineer receives full evidence ledger so zero work is repeated. | **Unknowns:** Escalation SLA times and on-call schedule.<br><br>**Assumption:** On-call engineer is available to receive handoffs.<br>**Owner:** Engineering Manager (ADR-04) | Layer 3 — Architecture & Design<br>§ 3.16 Escalation & Human Handoff |
| **Q3.17: Structured Event Logging** *(Essential fields for post-error reconstruction)* | **Team Discussion:**<br>• Diwas: Time travel back through execution layers.<br>• Team: Timestamp, component name, user ID, latency.<br>• Gap: Missed Request/Trace ID, stack trace, and log levels.<br><br>**Claude & GPT-3.5 Summary:**<br>• **Agreed:** Structured JSON logging. Non-negotiable fields: Request ID and Stack Trace.<br>• **Claude:** 8 mandatory fields: Timestamp (UTC ms), Log Level, Service, Trace ID, User ID, Description, Stack Trace, Context.<br>• **GPT-3.5:** Emphasized masking PII/secrets. | **Approved Decision:**<br>• **Structured Logging Standard:** Logs must emit JSON containing 8 standard fields (Timestamp, Level, Service, Trace ID, User ID, Message, Stack Trace, Context).<br>• PII/secrets redacted before logging.<br>• Platform relies on Request/Trace ID to reconstruct cross-service journeys. | **Unknowns:** Target app logging format.<br><br>**Assumption:** Target app emits structured logs with request IDs.<br>**Owner:** Tech Lead (Sprint 2 validation) | Layer 3 — Architecture & Design<br>§ 3.17 Structured Logging Specifications |
| **Q3.18: Measuring Request Failure Rate** *(Quantifying error rate over time)* | **Team Discussion:**<br>• Percentage of failed requests out of total requests.<br><br>**Claude & GPT-3.5 Summary:**<br>• **Agreed:** Standard formula: $(\text{Failed} / \text{Total}) \times 100$.<br>• **GPT-3.5:** Measure over defined time windows (e.g. 1 hour); segment by endpoint, service, and HTTP status code. Compare against normal baseline. | **Approved Decision:**<br>• **Error Rate Metric:** Calculated as $(\text{Failed Requests} / \text{Total Requests}) \times 100$ over sliding time windows.<br>• Segmented by endpoint, service, and HTTP status code.<br>• Compared continuously against historical baseline to detect anomalies. | **Unknowns:** Target org baseline error rate.<br><br>**Assumption:** Monitoring tool exposes reliable request and error counts.<br>**Owner:** DevOps Engineer (ADR-03) | Layer 3 — Architecture & Design<br>§ 3.18 Error Rate Telemetry |
| **Q3.19: Distributed Tracing & Request Journey** *(Reconstructing multi-service request flows)* | **Team Discussion:**<br>• Diwas & Mukul: Unique request ID carried across 5 services; backtrack to find failure.<br>• Gap: Did not name distributed tracing or context propagation.<br><br>**Claude & GPT-3.5 Summary:**<br>• **Agreed:** Distributed Tracing with OpenTelemetry.<br>• **Claude:** Trace ID generated at gateway, propagated via W3C HTTP headers, logging spans per service.<br>• **GPT-3.5:** Trace ID + Span ID + Parent-Child hierarchy with duration and status. | **Approved Decision:**<br>• Implement **Distributed Tracing (OpenTelemetry)**.<br>• Unique Trace ID created at gateway, propagated in HTTP headers (`traceparent`), with each service emitting a Span.<br>• Platform reconstructs full 5-service journey by querying the shared Trace ID. | **Unknowns:** Tracing backend in target org (Jaeger/Tempo).<br><br>**Assumption:** Services propagate W3C trace context headers.<br>**Owner:** Tech Lead (ADR-01) | Layer 3 — Architecture & Design<br>§ 3.19 Distributed Tracing Architecture |
| **Q3.20: Logs vs. Metrics (Frequency vs. Depth)** *(Why logs alone cannot measure failure frequency)* | **Team Discussion:**<br>• Logs tell what happened, not how many times. Counting requires scanning all logs.<br><br>**Claude & GPT-3.5 Summary:**<br>• **Agreed:** Logs = individual event records (WHAT); Metrics = pre-aggregated time-series counters (HOW OFTEN).<br>• **Claude:** Scanning raw logs for frequency is too slow/costly for real-time alerting. Need both.<br>• **GPT-3.5:** Complementary roles: metrics detect, logs diagnose. | **Approved Decision:**<br>• **Complementary Telemetry:** Platform requires both metrics and logs.<br>• Metrics provide low-cost real-time detection, rate calculation, and trend analysis.<br>• Logs provide line-by-line failure details, error messages, and stack traces. | **Unknowns:** None (architectural principle).<br><br>**Assumption:** Monitoring tool supplies both log search and time-series metrics APIs.<br>**Owner:** DevOps Engineer (ADR-03) | Layer 3 — Architecture & Design<br>§ 3.20 Telemetry Roles: Logs vs. Metrics |
| **Q3.21: Measuring Response Latency (Percentiles)** *(Tracking typical vs. outlier user experience)* | **Team Discussion:**<br>• Latency per operation/endpoint (endpoint + method + duration).<br>• Gap: Average latency is misleading; hides extreme outlier slowness.<br><br>**Claude & GPT-3.5 Summary:**<br>• **Agreed:** Track **percentiles (p50, p95, p99)** per endpoint, plus Time to First Byte (TTFB).<br>• **Claude:** Ties directly to Q3.11 (p99 explosion).<br>• **GPT-3.5:** Separate frontend render duration from backend latency. | **Approved Decision:**<br>• Track response duration per endpoint using **p50, p95, and p99 percentiles** (never a single blended average).<br>• p95/p99 latency serves as an SLA baseline for recovery verification.<br>• Collect Time to First Byte (TTFB) to isolate network from application processing. | **Unknowns:** Target endpoint latency SLAs.<br><br>**Assumption:** Application or gateway emits request duration telemetry.<br>**Owner:** DevOps Engineer (ADR-03) | Layer 3 — Architecture & Design<br>§ 3.21 Latency Percentiles Telemetry |
| **Q3.22: Establishing Incident Severity** *(5-dimensional severity assessment)* | **Team Discussion:**<br>• 30% error rate; consider revenue impact, duration, trend, and affected operations.<br><br>**Claude & GPT-3.5 Summary:**<br>• **Agreed:** Technical error rate alone does not dictate severity; business criticality is decisive.<br>• **Claude:** 5 measurements: Endpoint Criticality, Error Trend, User Impact Count, Revenue Lost (\$/min), Duration.<br>• **GPT-3.5:** Technical rate + business impact. | **Approved Decision:**<br>• **5 Severity Dimensions:** 1. Endpoint Criticality (payments = critical; health check = low), 2. Error Rate Trend (rising vs. falling), 3. User Impact Count, 4. Revenue Loss (\$/min), 5. Duration.<br>• Initial platform summary must evaluate all 5 dimensions. | **Unknowns:** Monetary cost per failed transaction.<br><br>**Assumption:** Target org can define business-critical endpoint tags.<br>**Owner:** Engineering Manager (ADR-04) | Layer 3 — Architecture & Design<br>§ 3.22 Incident Severity Framework |
| **Q3.23: Detecting CPU Saturation** *(Run queue length and load average vs. utilization)* | **Team Discussion:**<br>• CPU utilization %, latency symptom, per-process usage, traffic/DDoS spike.<br>• Gap: 85% CPU can be healthy; saturation means tasks are queueing.<br><br>**Claude & GPT-3.5 Summary:**<br>• **Agreed:** CPU % alone is insufficient.<br>• **Claude:** Saturation confirmed by **CPU Run Queue Length** and **Load Average > Core Count**. Latency is a symptom.<br>• **GPT-3.5:** Measure CPU throttling limits and per-process consumption. | **Approved Decision:**<br>• **CPU Saturation Signature:** 1. CPU Utilization % (sustained 90-100%), 2. CPU Run Queue Length (processes waiting), 3. Load Average > CPU Core Count, 4. Container CPU Throttling.<br>• Platform queries host infrastructure metrics, not just app logs. | **Unknowns:** Target VM core count and container CPU limits.<br><br>**Assumption:** Cloud monitoring exposes container CPU throttling and load average.<br>**Owner:** DevOps Engineer (Sprint 3 validation) | Layer 3 — Architecture & Design<br>§ 3.23 CPU Saturation Telemetry |
| **Q3.24: Diagnosing Memory Consumption & Leaks** *(Growth rate, OOM kills, and container memory limits)* | **Team Discussion:**<br>• Memory %, garbage collection / leaks, alert at 70%.<br>• Corrected: Docker containers **do not auto-scale memory**; limits are fixed and trigger OOM kills.<br><br>**Claude & GPT-3.5 Summary:**<br>• **Agreed:** Hard memory limits; watch for OOM kills and restart loops.<br>• **Claude:** Memory Growth Rate (steady climb = leak vs. burst), GC pause frequency, and Swap activity before crash.<br>• **GPT-3.5:** Heap usage and container OOM events. | **Approved Decision:**<br>• **Memory Diagnostics:** 1. Memory Usage % and absolute MB, 2. Memory Growth Rate slope (diagnoses leaks), 3. GC pause frequency/duration, 4. Swap disk usage, 5. Container OOM kill crash loops.<br>• Containers have fixed memory limits; OOM kills are tracked as critical alerts. | **Unknowns:** Target container memory limit allocations.<br><br>**Assumption:** Infrastructure reports OOM kill events and swap activity.<br>**Owner:** DevOps Engineer (Sprint 3 validation) | Layer 3 — Architecture & Design<br>§ 3.24 Memory & Resource Exhaustion |
| **Q3.25: Detecting Pending Background Jobs** *(Queue depth, processing lag, and oldest job age)* | **Team Discussion:**<br>• Monitor queue backlog, arrival vs. completion rate, worker health.<br><br>**Claude & GPT-3.5 Summary:**<br>• **Agreed:** Queue lag occurs when arrival rate exceeds worker processing capacity.<br>• **GPT-3.5:** Measure Queue Depth, Enqueue/Dequeue rates, **Oldest Job Age**, job duration, and retry counts. | **Approved Decision:**<br>• **Queue Lag Telemetry:** Track Queue Depth, Enqueue Rate, Dequeue Rate, **Oldest Job Age** (queue latency), and Failed/Retry counts.<br>• Alert when arrival rate persistently exceeds worker throughput or oldest job age breaches SLA. | **Unknowns:** Background queue technology (Celery/Redis/RabbitMQ).<br><br>**Assumption:** Queue exposes job count and age metrics.<br>**Owner:** Tech Lead (Sprint 2 validation) | Layer 3 — Architecture & Design<br>§ 3.25 Background Queue Monitoring |
| **Q3.26: Recording Deployment Information** *(Essential metadata for incident correlation)* | **Team Discussion:**<br>• Timestamp, commit ID, author, changed files, region.<br>• Gap: Missed configuration diffs (Reference Scenario!) and previous version ID.<br><br>**Claude & GPT-3.5 Summary:**<br>• **Agreed:** Capture code, config, author, and timing metadata.<br>• **Claude:** 6 required fields: Deploy Timestamp, **Commit SHA**, Code Diff, **Config Diff**, **Previous Version ID**, Deployer Identity.<br>• **GPT-3.5:** Include DB migrations and rollout %. | **Approved Decision:**<br>• **Deployment Record Schema:** 1. Deploy Timestamp, 2. Commit SHA, 3. Code Diff, 4. **Config/Env Diff** (critical for Reference Scenario), 5. **Previous Stable Version ID**, 6. Deployer Identity, 7. DB Migrations.<br>• Captured via automated CI/CD webhook. | **Unknowns:** CI/CD webhook payload format.<br><br>**Assumption:** CI/CD exposes config diffs and previous version IDs.<br>**Owner:** DevOps Engineer (ADR-03) | Layer 3 — Architecture & Design<br>§ 3.26 Deployment Metadata Standards |
| **Q3.27: Correlation Does Not Prove Causation** *(Proving deployment culpability through rigorous testing)* | **Team Discussion:**<br>• Correlation $\ne$ causation; third-factor events (e.g. traffic spikes) can coincide with deploys.<br><br>**Claude & GPT-3.5 Summary:**<br>• **Agreed:** Coincidence is not proof. Need multiple independent observations.<br>• **Claude:** 3 alternative explanations (third-factor, pre-existing errors, rolling restart blips). What proves causation: revert restores baseline, stack trace names changed key, exact timestamp alignment.<br>• **GPT-3.5:** Controlled rollback and testing to confirm. | **Approved Decision:**<br>• Deploy timing is a **starting hypothesis, never proof**.<br>• **Proving Causation:** 1. Error stack trace directly references changed code/config key, 2. Errors started precisely at deploy completion (not before), 3. Controlled sandbox revert restores health, 4. Third factors ruled out. | **Unknowns:** None (scientific investigation rule).<br><br>**Assumption:** Platform can safely test reverts in sandbox.<br>**Owner:** Tech Lead (Sprint 2 validation) | Layer 3 — Architecture & Design<br>§ 3.27 Causation vs. Correlation Guardrails |

---

## Detailed Evaluation & Model Comparison

### Q3.1: Blank Screen — The Journey of a Request

#### What the Team Said
* **User's View:** Browser URL $\rightarrow$ DNS address lookup $\rightarrow$ Load Balancer $\rightarrow$ Application code $\rightarrow$ Response back.
* **Team's Discussion:** Brainstormed bugs like DNS mismatch, network drops, server down, or bad deploy (using Vercel).
* **Key Insight (Speaker 3):** Network, DNS, or access drops show explicit browser error pages (*"Site can't be reached"*), not a blank screen.
* **Gap:** Debated isolated failure causes without tracing the full end-to-end journey.

#### Claude vs. GPT-3.5 Comparison

| Topic | Claude | GPT-3.5 | Consensus / Key Takeaway |
| :--- | :--- | :--- | :--- |
| **Missing Steps** | Detailed 6 missing hops: TLS, CDN, web gateway, app dependencies/config, return trip, browser rendering. | Identified missing steps: TLS, CDN/reverse proxy, database dependencies, browser rendering. | **Agreed:** Both found the team's flow incomplete; critical hops were missing. |
| **Blank Screen Cause** | Blank screen means a response *did* arrive, but was empty, broken, or unhandled. | Blank screen often means a frontend JavaScript crash rather than a backend failure. | **Agreed:** Blank screen is a symptom, not a cause. Network drops show error pages instead. |
| **MVP Boundaries** | Strictly enforces Layer 2: MVP covers Python backend only. Frontend JS, DNS, and CDN are strictly out-of-scope handoffs. | Leaves open whether MVP should diagnose frontend/browser errors or stay backend-only. | **Disagreed:** Claude adheres strictly to Layer 2 MVP limits; GPT-3.5 keeps scope wider. |
| **System Anchor** | Anchored to reference scenario: config change deploy $\rightarrow$ HTTP 500 $\rightarrow$ unhandled blank screen. | Used generic web architecture (`example.com`). | **Disagreed:** Claude ties directly to project requirements; GPT-3.5 is textbook-generic. |
| **Monitoring Gap** | Warned that server-only monitoring cannot detect client-only blank screens without synthetic checks. | Did not flag this detection gap. | **Disagreed:** Claude flagged a critical blind spot in server telemetry. |

#### Approved Decision Record
* **Standard 9-Step Journey:** `1. Browser URL` $\rightarrow$ `2. DNS` $\rightarrow$ `3. TLS/HTTPS` $\rightarrow$ `4. CDN/Edge` $\rightarrow$ `5. Load Balancer/Gateway (Nginx)` $\rightarrow$ `6. Python App` $\rightarrow$ `7. DB & Config` $\rightarrow$ `8. HTTP Response` $\rightarrow$ `9. Browser Rendering`.
* **In-Scope (Steps 5–7):** Platform investigates Gateway/Web server, Python runtime, and config/DB via the single monitoring tool.
* **Out-of-Scope (Steps 1–4, 9):** Browser JS crashes, DNS, CDN, TLS certs. Platform reports *"Out of scope"* and hands off to human SRE without guessing.
* **Working Assumption:** Target app blank screens in MVP stem from server-side HTTP 500 errors (SSR), not client-side JavaScript crashes.
  * **Owner:** Tech Lead | **Validation:** Sprint 2.
* **Open Decision:** Decide in Sprint 2 whether to add an outside-in synthetic check to catch blank screens when server logs show no errors.

---

### Q3.2: Thousands of Simultaneous Requests: Component Responsibilities

#### What the Team Said
* **User's View:** Queue requests when overloaded; balance load across servers.
* **Team's Discussion:** Discussed load balancers with health checks, Redis caching for fast reads, Celery background queues for slow jobs, database indexing, and rate-limiting abusive bots.
* **Gap:** Named physical components rather than defining duties under load (worker timeouts, pool limits, backpressure, idempotency).

#### Claude vs. GPT-3.5 Comparison

| Topic | Claude | GPT-3.5 | Consensus / Key Takeaway |
| :--- | :--- | :--- | :--- |
| **Platform Scope (Layer 2)** | Tied directly to read-only limits: platform cannot scale servers or tune load balancers. Overload requires handoff. | Treated scaling as an application/infra engineering task; ignored platform read-only boundaries. | **Disagreed:** Claude enforces Layer 2 operational limits. |
| **Reference Scenario Link** | Connected load to deployment: changing config limits (workers, pool size, timeouts) causes 500 errors *under heavy traffic*. | Discussed generic request lifecycle steps (validation, business logic, DB queries). | **Disagreed:** Claude tied traffic spikes directly to configuration bugs. |
| **Alert Storms** | Stressed grouping surging load alerts into 1 incident (Layer 2.17). | Mentioned server health monitoring, but omitted AI alert storm grouping. | **Disagreed:** Claude protected the platform from alert flooding. |
| **Backpressure & Concurrency** | Detailed worker limits, request timeouts, DB connection pools, and returning 503 instead of crashing. | Detailed concurrent execution management, connection pools, and backpressure. | **Agreed:** Both emphasized graceful load shedding over crashing. |

#### Approved Decision Record
* **Component Responsibilities Under High Load:**
  1. *Load Balancer:* Spreads traffic across healthy servers; instantly routes around nodes failing health checks.
  2. *Application Server (Python):* Runs parallel workers; enforces timeouts; sheds excess load with HTTP 503 instead of crashing; ensures POST idempotency.
  3. *Database & Cache:* Database pools connections and uses indexes; cache absorbs high-frequency read queries.
  4. *Background Queue:* Offloads slow/heavy jobs asynchronously.
  5. *Traffic Protection:* Rate-limits abusive clients and bots before they saturate workers.
* **Platform Boundary (Layer 2):** Platform is read-only; cannot add servers or tune load balancers. If errors follow a sudden traffic surge without a deploy, label **"Likely Traffic Overload"** and escalate to a human.
* **Alert Storm Handling:** All alerts from the same traffic surge are grouped into a single incident (Layer 2.17).
* **Working Assumption:** Target app runs on multiple worker instances behind a load balancer with active health checks.
  * **Owner:** DevOps Engineer | **Validation:** Sprint 2.
* **Open Unknown:** Normal vs. peak traffic capacity of target app (must be load tested, not guessed).

---

### Q3.3: A Request Returns an Error: Request-Response Lifecycle

#### What the Team Said
* **Internal Flow:** Routing $\rightarrow$ Validation $\rightarrow$ Code $\rightarrow$ Database $\rightarrow$ Response. If a step fails, subsequent steps halt and an error is returned.
* **Error Categories:** 4xx represents client-side errors; 5xx represents server-side errors.
* **Misconception Corrected:** Speaker 2 claimed 5xx means a configuration error. This was corrected: configuration is only one cause among code bugs, database crashes, timeouts, and overload.
* **Gap:** Confused system load (Q3.2) with single-request processing (Q3.3) and grouped network drops (which have no HTTP code) with HTTP errors.

#### Claude vs. GPT-3.5 Comparison

| Topic | Claude | GPT-3.5 | Consensus / Key Takeaway |
| :--- | :--- | :--- | :--- |
| **5xx Meaning** | Directly corrected the team: 5xx is an umbrella status, not proof of a config error. Status codes are clues, not diagnoses. | Grouped 5xx into standard HTTP status codes (500, 502, 503). | **Claude prevents premature diagnosis.** |
| **Reference Scenario Link** | Anchored to reference scenario: bad config deploy $\rightarrow$ unhandled Python exception $\rightarrow$ HTTP 500 $\rightarrow$ blank screen if unhandled. | Kept analysis generic with sample endpoints (`GET /api/users/123`). | **Claude grounds error in real project incident.** |
| **MVP Scope (4xx vs 5xx)** | Recommends MVP focus strictly on 5xx server crashes; defer 4xx error surges post-MVP. | Cataloged all 4xx/5xx status codes equally without narrowing MVP scope. | **Claude focuses MVP on server-side failures.** |
| **Observability Level** | Stressed assumption that Python logs contain stack trace and request ID. | Raised open decision on structured logs vs. distributed traces. | **Agreed:** Telemetry quality dictates investigation capability. |

#### Approved Decision Record
* **7-Stage Internal Pipeline:**
  1. *URL Routing:* Maps path/method to code controller. (Failure $\rightarrow$ 404)
  2. *Authentication:* Verifies identity token/session. (Failure $\rightarrow$ 401)
  3. *Authorization:* Checks user permissions. (Failure $\rightarrow$ 403)
  4. *Input Validation:* Validates payload schema and fields. (Failure $\rightarrow$ 400)
  5. *Business Logic Execution:* Executes Python code. (Unhandled exception $\rightarrow$ 500)
  6. *Dependency Interaction:* Queries DB, cache, or external APIs. (Failure $\rightarrow$ 500/502/503)
  7. *Error Translation & Logging:* Execution halts, structured error is logged with stack trace and request ID, and sanitized HTTP response is returned.
* **Diagnostic Rules for Platform (Layer 2):**
  * Status code is a diagnostic clue, not proof of a config error. Platform tests alternative hypotheses before proposing a fix.
  * MVP incident investigation is restricted to **5xx server errors**. 4xx client errors and network drops are excluded.
* **Working Assumption:** Target Python web app logs stack trace, timestamp, and request identifier for every HTTP 500 error.
  * **Owner:** Tech Lead | **Validation:** Sprint 2 (trigger config fault on test copy; verify log output).
* **Open Decision:** Defer 4xx error surge detection (e.g. mass 401/403 alerts caused by bad auth deploys) to post-MVP.

---

### Q3.4: External Communication Interface

#### What the Team Said
* **Team's View:** Everyone agreed on an HTTP API / REST API for external clients, with brief mentions of SDKs and microservices.
* **Gap:** Named a generic concept ("API") rather than an architectural component ("API Gateway / HTTP REST API Layer") and did not connect to Layer 2 (Q2.8).

#### Claude vs. GPT-3.5 Comparison

| Topic | Claude | GPT-3.5 Standard | Consensus / Key Takeaway |
| :--- | :--- | :--- | :--- |
| **Component Name** | API Gateway / HTTP REST API Layer. | API Gateway / Reverse Proxy. | **Agreed:** Dedicated ingress layer component. |
| **Layer 2 Anchor** | Directly tied to Q2.8 (single HTTP REST API, UI dashboard, CLI optional). | General client-server API entry point. | **Claude anchors directly to Layer 2 decisions.** |
| **Security Boundary** | No external client talks directly to DB or internal modules. Auth and rate limiting sit here. | Handles request routing, auth validation, and rate limiting. | **Agreed:** Single secure front door for all external callers. |

#### Approved Decision Record
* **Decision:** The **API Gateway / HTTP REST API Layer** is the component that exposes the communication interface to external clients (monitoring webhooks, UI dashboard, CI/CD, CLI).
* **Constraints (Layer 2.8):** Direct external access to database or internal modules is strictly forbidden. Centralizes auth and rate limiting.
* **Working Assumption:** All integrated external systems communicate over standard HTTP/REST webhooks and APIs.
  * **Owner:** Tech Lead | **Validation:** Sprint 3.

---

### Q3.5: Persistent Customer Information

#### What the Team Said
* **Team's View:** Discussed sessions, browser cookies, and preserving data across restarts.
* **Misconception Corrected:** Cookies and sessions do not store persistent customer data. Cookies only hold temporary browser tokens; sessions are temporary memory/cache wiped on restart.

#### Claude vs. GPT-3.5 Comparison

| Topic | Claude / Layer 2 Anchor | GPT-3.5 | Consensus / Key Takeaway |
| :--- | :--- | :--- | :--- |
| **Component** | Database Component (PostgreSQL per Layer 2.21). | Database Component (Relational DB). | **Agreed:** Database is the only component providing durable disk storage across restarts. |
| **State Separation** | Ephemeral app memory is wiped on restart; DB holds persistent state. | Detailed 3-tier split: Database (durable records), Session (temporary login), Cookie (browser token). | **Agreed:** Strict separation of durable data from temporary session tokens. |
| **Access Control** | External clients cannot touch DB; all queries go through application backend. | Application controls all CRUD operations safely. | **Agreed:** Direct external DB access strictly prohibited. |

#### Approved Decision Record
* **Decision:** The **Database Component (Relational DB / PostgreSQL)** is the component responsible for managing persistent customer information across application restarts.
* **Separation of Roles:** Database = durable customer records on disk; Session = temporary interaction state; Cookie = client browser session token.
* **Constraints (Layer 2.21):** External clients never access the database directly; PostgreSQL is the designated platform state DB.
* **Working Assumption:** Relational database meets schema consistency and durability requirements.
  * **Owner:** Tech Lead / App Lead | **Validation:** Sprint 2.

---

### Q3.6: Deploying a New Application Version

#### What the Team Said
* **Team's View:** Code push $\rightarrow$ CI/CD $\rightarrow$ automated test suites $\rightarrow$ deploy if pass, rollback if fail. Diwas noted canary deployment protects production.
* **Distinction Clarified:** Canary deployment (5-10% traffic to verify health metrics) $\neq$ A/B testing (split traffic to test user behavior/conversion).
* **Gap:** Missed the build step (packaging into a Docker image), staging/sandbox pre-validation, and post-deploy monitoring.

#### Claude vs. GPT-3.5 Comparison

| Topic | Claude | GPT-3.5 | Consensus / Key Takeaway |
| :--- | :--- | :--- | :--- |
| **Platform Trigger** | Highlights **Post-Deploy Monitoring Window**: metric degradation triggers auto-rollback and fires the alert that **activates AI SRE**. | Highlights standard release pipeline best practices. | **Claude connects the deploy directly to the AI SRE incident trigger.** |
| **Layer 2 Guardrails** | Anchors to Q2.6 (1 automated CI/CD pipeline, no manual deploys) and Q2.14 (canary + auto-rollback). | Emphasizes human approval gates and distinguishing A/B testing from canary. | **Both agree:** Automated safety gates + human oversight are mandatory. |
| **Reference Scenario Link** | Explains why reference bug happened: tests passed, but config change caused 500s on live traffic. | Focuses on pipeline architecture and pre-deployment sandbox checks. | **Claude anchors directly to the reference scenario.** |
| **CI/CD Integration** | Platform must read CI/CD deployment events (commit SHA, timestamp, diffs). | Asks open questions about specific deployment strategies. | **Claude identifies the required telemetry link.** |

#### Approved Decision Record
* **4-Phase Deployment Sequence:**
  1. *Code Phase:* Engineer pushes code $\rightarrow$ Webhook triggers CI/CD.
  2. *Continuous Integration (CI):* Static analysis, linting, and tests run $\rightarrow$ Immutable **Docker container image** is built.
  3. *Pre-Deployment Sandbox (CD):* Image deploys to isolated sandbox/staging $\rightarrow$ Smoke and config tests run.
  4. *Production Canary & Verification Phase:*
     * Gradual **Canary rollout** (e.g. 10% traffic).
     * Post-deployment monitoring window watches error rates and latency.
     * Healthy $\rightarrow$ complete rollout to 100%.
     * Degraded $\rightarrow$ automatic rollback engages $\rightarrow$ alert fires $\rightarrow$ **AI SRE Platform activates**.
* **Constraints (Layer 2.6 & 2.14):** 1 automated CI/CD tool; manual deploys strictly out of scope; canary with automated rollback is required.
* **Working Assumption:** CI/CD tool emits deployment webhook events (commit SHA, timestamp, author, diffs) readable by the platform.
  * **Owner:** DevOps Engineer | **Validation:** Sprint 3.
* **Open Decision:** Determine duration of post-deploy canary monitoring window (5 vs. 15 mins).

---

### Q3.7: Configuration vs. Application Code

#### What the Team Said
* **Team's View:** Diwas: Diff shows config changes. Team: Test same code in two envs; revert env var to verify.
* **Key Context:** Source code is 100% identical.
* **Gap:** Missed timing correlation (error spikes at exact second of deploy) and inspecting log signatures for config keys.

#### Claude vs. GPT-3.5 Comparison

| Topic | Claude | GPT-3.5 | Consensus / Key Takeaway |
| :--- | :--- | :--- | :--- |
| **Investigation Flow** | 4 steps: Diff Config $\rightarrow$ Check Timing $\rightarrow$ Read Logs $\rightarrow$ Revert & Verify. | 4 steps: Inspect Config $\rightarrow$ Verify Artifact $\rightarrow$ Controlled A/B Test $\rightarrow$ Controlled Revert. | **Agreed:** Verify artifact identity, compare values, and revert to prove causality. |
| **Reference Scenario Link** | Direct match: deploy changed config $\rightarrow$ HTTP 500s. Platform generates config revert PR. | Explains concept using general timeout examples (`PAYMENT_TIMEOUT`). | **Claude directly maps to our MVP incident lifecycle.** |
| **Platform Ingestion** | Platform must store old value alongside new value to propose revert PR. | Emphasizes masking secrets and credentials in logs. | **Both agree:** Safe, auditable configuration tracking is required. |

#### Approved Decision Record
* **4-Step Configuration Impact Isolation:**
  1. *Diff Configuration:* Compare previous vs. current values of the environment variable.
  2. *Correlate Timestamp:* Check if error rates jumped at the exact second the config applied while code hash stayed identical.
  3. *Read Error Logs:* Inspect logs for config-related failure traces (timeouts, missing keys, connection limits).
  4. *Sandbox Revert & Verification:* Apply previous config value in sandbox; if health returns, causality is proven.
* **Platform Requirements (Layer 2):** Platform ingests old and new config values from webhook; proposes automated config revert PR. Formats: YAML, JSON, `.env`, env vars. Secrets redacted.
* **Working Assumption:** Deployment pipeline captures and exposes config diffs.
  * **Owner:** DevOps Engineer | **Validation:** Sprint 3.

---

### Q3.8: Investigating and Isolating a Regression

#### What the Team Said
* **Team's View:** Look at Git diff, revert to previous commit, reproduce in test env, use feature flags.
* **Gap:** Diff is only a candidate list; manual guessing is slow. Missed Git Bisect for multi-commit releases.

#### Claude vs. GPT-3.5 Comparison

| Topic | Claude | GPT-3.5 | Consensus / Key Takeaway |
| :--- | :--- | :--- | :--- |
| **Isolation Method** | Explicitly specifies **Git Bisect** for multi-commit releases; inspect diff directly for small 1–2 commit releases. | Suggests targeted test runs, controlled reverts, and Git bisect. | **Agreed:** Binary search (bisect) is standard for multi-change releases. |
| **Response Strategy** | Clear rule: Full Rollback if whole release can be reverted; Feature Flag if other changes must stay. | Three options: Full Rollback, Targeted Revert PR, or Feature Flag. | **Agreed:** Separate instant mitigation from root-cause repair. |
| **Platform Guardrails** | Rollback requires human approval at both gates (Layer 2.14). AI never executes rollbacks unilaterally. | Highlights database migration compatibility during rollbacks. | **Both add essential Layer 2 safety guardrails.** |

#### Approved Decision Record
* **5-Step Workflow:**
  1. *Identify Baseline:* Last known good commit passing health checks.
  2. *Diff Release:* Count code, config, and DB migration changes.
  3. *Isolate Breaking Commit:* Direct diff inspection for 1–2 commits; **Git Bisect** in sandbox for multi-commit releases.
  4. *Reproduce in Sandbox:* Confirm bug on suspect commit; verify previous commit passes.
  5. *Two-Gate Recovery:* Full rollback via canary if clean; feature flag or targeted revert PR if other changes must stay.
* **Constraints (Layer 2.14):** Rollbacks require human authorization at Gate 1 and Gate 2. DB migrations must be backward-compatible.
* **Working Assumption:** Target team maintains atomic commits with clear history for bisection.
  * **Owner:** Tech Lead | **Validation:** Sprint 2.

---

### Q3.9: Declaring and Tracking an Incident

#### What the Team Said
* **Team's View:** Listed scattered criteria: affected geography, user count, component criticality, revenue impact, and compliance breaches.
* **Gap:** Did not establish structured quantitative triggers.

#### Approved Decision Record
* **Decision:** An incident is declared if **any 1 of 5 conditions** is met:
  1. *User Impact:* Meaningful percentage of active users impacted (>5%).
  2. *Critical Function Broken:* Revenue or compliance pathways down (auth, payments).
  3. *Complete Unavailability:* Application returns 0 successful responses.
  4. *Security / Compliance Breach:* Active data leak or regulatory risk.
  5. *SLA Breach:* Outage exceeds customer agreement limits.
* **Platform Pre-Filter (Layer 2):** Alerts failing all 5 criteria are dropped at \$0 LLM cost.
* **Working Assumption:** Target org provides numeric thresholds for user impact and SLAs.
  * **Owner:** Engineering Manager | **Validation:** ADR-04.

---

### Q3.10: 40% Failure Rate vs. Complete Outage

#### What the Team Said
* **Team's View:** 40% is not 100%; test a few times; check specific endpoints or DDoS patterns.
* **Gap:** Missed formal terminology (degraded state vs. complete outage) and clustering analysis.

#### Approved Decision Record
* **3 Verification Checks:**
  1. *Error Rate Metric:* 100% = complete outage; <100% = partial degradation.
  2. *Clustering Pattern:* Failures isolated to specific endpoints, user tiers, or regions vs. uniform across all paths.
  3. *Health Check:* `/health` responding 200 OK proves application process is alive.
* **Platform Summary:** Initial summary reports error rate, affected endpoints, and blast radius.

---

### Q3.11: Latency Degradation (100 ms to 8s)

#### What the Team Said
* **Team's View:** Performance issue, not outage; caused by DB slowness, traffic, or server resource issues.
* **Gap:** Failed to characterize severity (80x slowdown makes application functionally unusable).

#### Approved Decision Record
* **Characterization:**
  * *Type:* Latency Incident / Performance Degradation.
  * *Severity:* High (8 seconds causes widespread abandonment and SLA breaches).
  * *Investigation Order:* 1. DB locks/connection pool $\rightarrow$ 2. External API slowness $\rightarrow$ 3. Traffic surge $\rightarrow$ 4. Host CPU/RAM pressure $\rightarrow$ 5. N+1 queries.
* **Platform Rule:** Platform investigates latency spikes even when HTTP error rate is 0%. Must track p50/p95/p99.
* **Working Assumption:** Monitoring tool tracks response time percentiles.
  * **Owner:** DevOps Engineer | **Validation:** ADR-03.

---

### Q3.12: Hypothesis-Driven Investigation

#### What the Team Said
* **Team's View:** Follow evidence iteratively; check past incidents and recent commits.
* **Gap:** Lacked a formalized scientific investigation method and confidence scoring.

#### Approved Decision Record
* **4-Step Scientific Loop:**
  1. *Pattern Match:* Compare telemetry against past incidents and runbooks.
  2. *Form Hypothesis:* Pick most probable cause and assign explicit **Confidence Score**.
  3. *Test Hypothesis:* Query specific metrics to prove or disprove it.
  4. *Iterate or Escalate:* If confirmed, proceed to fix; if rejected, form next hypothesis. If all rejected, stop, declare missing data, and ask human (PRD 1.4 UC5). Never hallucinate.

---

### Q3.13: Restoring Service (Two-Track Recovery)

#### What the Team Said
* **Team's View:** Mukul noted knowing cause $\ne$ fix ready; rollback first to restore service, fix properly in separate env.
* **Gap:** Did not formalize parallel execution tracks.

#### Approved Decision Record
* **Two Parallel Tracks:**
  * *Track A (Immediate Mitigation):* Fast canary rollback, config revert, or feature flag toggle to restore availability immediately.
  * *Track B (Permanent Fix):* Develop patch on branch, test in sandbox, obtain human approval, deploy.
* **Rule:** "Service Restored" (Track A) and "Incident Closed" (Track B) are separate states. Incident stays open until Track B verifies baseline.

---

### Q3.14: Returning to Working Version (Rollback)

#### What the Team Said
* **Team's View:** "Simply rollback" / assumed capability exists.
* **Correction:** Rollback is Layer 2 Assumption 5 (architecture-breaking assumption).

#### Approved Decision Record
* **5-Step Rollback Sequence:**
  1. *Identify Baseline:* Retrieve commit SHA and deployment ID of last stable release.
  2. *Trigger via CI/CD:* Execute automated pipeline redeploy (never manual copying).
  3. *Gate 2 Human Approval:* Rollback deployment requires human authorization (Layer 2.14).
  4. *Canary Rollback:* Shift traffic gradually (10% $\rightarrow$ 50% $\rightarrow$ 100%).
  5. *Verify Recovery:* Confirm metrics return to pre-incident baseline before closing.
* **Working Assumption:** Target CI/CD supports automated rollback (Assumption 5).
  * **Owner:** DevOps Engineer | **Validation:** ADR-06.

---

### Q3.15: Incident Lifecycle State Machine

#### What the Team Said
* **Team's View:** Track in Jira (open $\rightarrow$ in progress $\rightarrow$ PR drafted $\rightarrow$ closed).
* **Gap:** Reinvented lifecycle; PRD already defines an 11-state machine with human approval gates.

#### Approved Decision Record
* **11-State Model:** Detected $\rightarrow$ Notified $\rightarrow$ Investigating $\rightarrow$ DiagnosisProposed $\rightarrow$ FixProposed $\rightarrow$ **AwaitingFixApproval** $\rightarrow$ SandboxTesting $\rightarrow$ **AwaitingDeployApproval** $\rightarrow$ Deployed $\rightarrow$ Verifying $\rightarrow$ Closed.
* **Persistence & Audit:** State transitions stored in append-only PostgreSQL table (Layer 2.21) with actor ID and timestamp. Closed is reachable only from Verifying.

---

### Q3.16: Escalation and Handoff Mechanics

#### What the Team Said
* **Team's View:** Escalate to senior with context of what was tried. Prateek noted AI SRE is the junior/first responder.
* **Gap:** Lacked explicit mechanical triggers and structured evidence ledger transfer.

#### Approved Decision Record
* **3 Escalation Triggers:**
  1. *No Expertise:* AI cannot confirm hypothesis $\rightarrow$ Escalate to service owner with hypotheses tested, evidence ledger, and confidence scores.
  2. *No Access:* Telemetry source requires higher IAM credentials $\rightarrow$ Escalate to DevOps lead stating missing permissions.
  3. *Time SLA Exceeded:* Diagnosis exceeds 15-minute ceiling (Layer 2.18) $\rightarrow$ Auto-escalate to on-call senior.
* **Rule:** Escalation to human is a core deliverable. Handoff ledger ensures incoming engineer never repeats work.

---

### Q3.17: Structured Event Logging

#### What the Team Said
* **Team's View:** Diwas: Trace back through execution layers like time travel. Team: Timestamp, component name, user ID, latency.
* **Gap:** Missed Request/Trace ID, stack trace, and log levels.

#### Approved Decision Record
* **Structured JSON Logging:** Logs must emit JSON with 8 standard fields: Timestamp (UTC ms), Log Level, Service Name, **Trace/Request ID**, User ID (anonymized), Description, **Stack Trace**, and Context.
* **Constraint:** Platform cannot reconstruct journeys without consistent Request/Trace IDs. PII redacted.

---

### Q3.18: Measuring Request Failure Rate

#### What the Team Said
* **Team's View:** Percentage of failed requests out of total requests.

#### Approved Decision Record
* **Error Rate Formula:** $\text{Error Rate} = (\text{Failed Requests} / \text{Total Requests}) \times 100$ over defined sliding windows (e.g. 1 hour).
* **Segmentation:** Broken down by endpoint, service, and HTTP status code; compared continuously against historical baseline.

---

### Q3.19: Distributed Tracing & Request Journey

#### What the Team Said
* **Team's View:** Diwas & Mukul: Unique request ID carried across 5 services; backtrack to find failure.
* **Gap:** Did not name distributed tracing or context propagation.

#### Approved Decision Record
* **Distributed Tracing (OpenTelemetry):**
  1. Gateway generates unique `Trace ID`.
  2. Propagated in HTTP request headers (`traceparent`) across all downstream services.
  3. Each service logs a `Span` (start time, duration, status, parent-child ID).
  4. Platform queries shared Trace ID to visualize full 5-service journey.

---

### Q3.20: Logs vs. Metrics (Frequency vs. Depth)

#### What the Team Said
* **Team's View:** Logs show what happened, not how many times. Counting requires scanning all logs.

#### Approved Decision Record
* **Complementary Telemetry:**
  * *Metrics:* Pre-aggregated time-series counters and rates; fast, low-cost detection and frequency measurement.
  * *Logs:* Discrete event records; detailed line-by-line diagnosis, error messages, and stack traces.
* **Rule:** Platform requires both metrics (detection/severity) and logs (diagnosis).

---

### Q3.21: Measuring Response Latency (Percentiles)

#### What the Team Said
* **Team's View:** Latency per operation/endpoint (endpoint + method + duration).
* **Gap:** Average latency is misleading; hides extreme outlier slowness.

#### Approved Decision Record
* **Latency Percentiles:** Track response duration per endpoint using **p50, p95, and p99 percentiles** (plus Time to First Byte). Blended averages rejected.
* **SLA Baseline:** p95/p99 latency serves as the recovery verification baseline.

---

### Q3.22: Establishing Incident Severity

#### What the Team Said
* **Team's View:** 30% error rate; consider revenue impact, duration, trend, and affected operations.

#### Approved Decision Record
* **5 Severity Dimensions:**
  1. *Endpoint Criticality:* Payment/auth = critical; health check = low.
  2. *Error Rate Trend:* Rising vs. stable vs. falling.
  3. *User Impact Count:* Absolute unique user count affected.
  4. *Revenue Impact:* Estimated \$/minute lost.
  5. *Duration:* Cumulative time degraded.

---

### Q3.23: Detecting CPU Saturation

#### What the Team Said
* **Team's View:** CPU utilization %, latency symptom, per-process usage, traffic/DDoS spike.
* **Gap:** 85% CPU can be healthy; saturation means tasks are queueing up.

#### Approved Decision Record
* **CPU Saturation Signature:** 1. CPU Utilization % (sustained 90-100%), 2. **CPU Run Queue Length** (processes waiting), 3. **Load Average > CPU Core Count**, 4. Container CPU Throttling.
* Latency is a symptom; saturation is confirmed via host/container queue metrics.

---

### Q3.24: Diagnosing Memory Consumption & Leaks

#### What the Team Said
* **Team's View:** Memory %, garbage collection / leaks, alert at 70%. Diwas suggested containers auto-scale memory.
* **Correction:** Docker containers **do not auto-scale memory**; limits are fixed and trigger OOM kills.

#### Approved Decision Record
* **Memory Diagnostics:**
  1. *Memory Growth Rate:* Steady upward slope over hours indicates a memory leak.
  2. *GC Pauses:* Frequent or long garbage collection runs indicate memory pressure.
  3. *Swap Activity:* Disk swap usage is the final warning before crash.
  4. *OOM Kill Events:* Hard container crashes caused by exceeding configured memory limits.

---

### Q3.25: Detecting Pending Background Jobs

#### What the Team Said
* **Team's View:** Monitor queue backlog, arrival vs. completion rate, worker health.

#### Approved Decision Record
* **Queue Telemetry:** Track Queue Depth, Enqueue Rate, Dequeue Rate, **Oldest Job Age** (latency), and Retry Counts.
* Alert when arrival rate persistently exceeds worker processing rate or oldest job age breaches SLA.

---

### Q3.26: Recording Deployment Information

#### What the Team Said
* **Team's View:** Timestamp, commit ID, author, changed files, region.
* **Gap:** Missed configuration diffs (Reference Scenario!) and previous stable version ID.

#### Approved Decision Record
* **Deployment Record Schema:** 1. Deploy Timestamp, 2. Commit SHA, 3. Code Diff, 4. **Config/Env Diff** (Reference Scenario), 5. **Previous Stable Version ID**, 6. Deployer Identity, 7. DB Migrations. Captured via CI/CD webhook.

---

### Q3.27: Correlation Does Not Prove Causation

#### What the Team Said
* **Team's View:** Correlation $\ne$ causation; third-factor events (e.g. traffic spikes) can coincide with deploys.

#### Approved Decision Record
* **Proving Causation:** Deployment timing is a starting hypothesis, not proof. Causation requires:
  1. Error stack trace directly references changed code/config key.
  2. Errors began precisely at deployment completion (not before).
  3. Controlled sandbox revert restores baseline health.
  4. Third factors (traffic surges, dependency outages) ruled out.
