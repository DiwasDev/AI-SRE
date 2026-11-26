# ADR-13: FastAPI and Pydantic for HTTP Intake & Status Gateway

## Status
Accepted

## Context (First Principles)
The AI SRE platform requires a web server to receive webhook alerts from monitoring tools, serve dashboard queries, and accept human approvals.
The framework must:
1. Validate incoming JSON payloads strictly without custom parsing boilerplate.
2. Support asynchronous request handling so waiting on database I/O does not block incoming webhooks.
3. Be lightweight and native to Python 3.11+.

## Decision
1. **Adopt FastAPI with Pydantic:**
   - FastAPI is selected as the HTTP framework for the `api` process.
   - Pydantic models enforce the 5 mandatory intake fields and reject malformed schemas automatically.
2. **Strict Architectural Boundary:**
   - FastAPI resides strictly within the `api/` adapter layer.
   - It acts purely as a receptionist/controller: validating inputs, initiating database transactions, and returning responses.
   - Route handlers are strictly forbidden from executing LLM reasoning, orchestrating workflows, or running sandbox tests.

## Overrides & Traceability
- **Resolves:** Layer 6 Q6.23.

## Consequences
- **Positive:** High performance, automatic OpenAPI documentation, clean separation between web routing and core domain logic.
- **Negative:** Developers must maintain strict discipline to keep route handlers thin and avoid leaking domain logic into API endpoints.
