# ADR-10: PostgreSQL-Backed Job Queue with SKIP LOCKED and 3-Slot Ceiling

## Status
Accepted

## Context (First Principles)
When multiple investigations are initiated, background workers need a coordinated way to pick up jobs without multiple workers accidentally working on the exact same incident.
In an MVP with a strict concurrency ceiling (exactly 3 active investigations and a maximum of 20 queued incidents per Layer 4 § 4.22), adding a dedicated external message broker (such as RabbitMQ, Kafka, or Redis) adds another stateful service to configure, monitor, secure, and back up on a single cloud VM.

## Decision
1. **PostgreSQL as the Queue Mechanism:**
   - Use PostgreSQL tables with `SELECT ... FOR UPDATE SKIP LOCKED` for worker dispatch and queue coordination.
   - Guarantees transactional consistency: saving the incident and queueing the job happen in a single, unbreakable database transaction.
2. **Concurrency Ceiling Enforcement:**
   - Exactly 3 active concurrency slots are permitted.
   - When 3 slots are occupied, incoming incidents enter the priority queue (scored by severity, production status, and wait time).
   - Starvation prevention: an alert fires if an incident waits longer than 10 minutes.
   - If the queue exceeds 20 incidents, the platform issues an emergency capacity warning: *"Platform busy (20+ queued) — handle manually"*, while continuing to persist raw alerts durably to prevent data loss.
3. **Resilience & Sweeper:**
   - Active workers emit a heartbeat to the database every 60 seconds.
   - A lightweight sweeper daemon runs every 30 seconds to reclaim orphaned jobs from dead workers (missing heartbeat > 60s) and resume from the last saved checkpoint.

## Overrides & Traceability
- **Reconciles:** Layer 4 § 4.17, § 4.22, § 4.29, § 4.34 with Layer 6 Q6.4, Q6.8, Q6.10, Q6.11. Eliminates the need for an external message broker in the MVP.

## Consequences
- **Positive:** Zero extra operational infrastructure; transactionally sound; crash recovery is baked directly into the primary database.
- **Negative:** Not designed for tens of thousands of jobs per second (which is completely acceptable for our 3-slot single-team MVP).
