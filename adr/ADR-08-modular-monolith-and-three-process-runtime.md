# ADR-08: Modular Monolith Codebase with 3-Process Runtime Topology

## Status
Accepted

## Context (First Principles)
An incident platform handles three radically different kinds of tasks:
1. Fast, high-availability web requests (receiving webhooks, serving the dashboard).
2. Long-running, un-trusted reasoning (reading logs, querying LLMs, running test containers).
3. Highly privileged production actions (canary deployments, emergency rollbacks).

If all of these live in one single process, a 10-minute LLM diagnosis freezes incoming webhooks, and an untrusted prompt injection could compromise production deployment keys.
Conversely, splitting these into separate microservices (independent git repositories and network services) creates distributed consistency problems, requires distributed tracing, increases operational overhead for a small team, and multiplies network failure points.

## Decision
1. **Single Codebase (Modular Monolith):** The platform is developed and versioned as a single Python codebase structured with strict internal module boundaries using the Ports & Adapters pattern.
2. **Single Built Image, Three Process Entrypoints:** A single Docker container image is built, but started in 3 distinct roles via Docker Compose:
   - `api`: Handles HTTP ingress, authentication, UI queries, and recording human approvals. Holds **zero AI keys and zero deployment credentials**.
   - `worker`: Background investigation pool (identical workers handling up to 3 concurrent active slots). Holds AI API keys and read-only telemetry tokens, but **zero production deployment credentials**.
   - `executor` (`AI Deployment Executor`): Runs as an isolated worker dedicated solely to production release operations. **Exclusively holds CI/CD deployment credentials** and executes actions only after cryptographically verifying approved patch hashes and TTL validity.
3. **Internal Module Boundary Enforcement:** In-process Python calls connect modules. An automated import linter (`import-linter`) runs in CI to forbid cross-boundary violations (e.g., `investigation` can never import `execution`).

## Overrides & Traceability
- **Overrides/Clarifies:** Reconciles the debate in Q6.3 (two APIs vs. roles) and Q6.8 (separate folders vs. shared core).
- **Enforces:** Layer 2 § 2.2 (Modular Monolith), Layer 5 § 3 (Zone 1/2/3 separation), and Layer 5 § 6 (Deployment Executor isolation).

## Consequences
- **Positive:** No duplicated business logic; simple local setup; clear separation of security privileges; easy deployment via Docker Compose.
- **Negative:** Requires strict CI linting to keep code boundaries clean over time.
