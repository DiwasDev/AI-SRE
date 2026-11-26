# ADR-11: Deterministic Diagnosis Verification Over Multi-Round LLM Debate

## Status
Accepted

## Context (First Principles)
When an LLM produces a root-cause diagnosis, there is a risk of confident hallucination. One proposed approach was to introduce a second "LLM critic" that debates or cross-examines the primary diagnosis over 3 to 4 rounds.

However, from first principles:
1. LLMs running similar prompts often share the same underlying blind spots and training biases.
2. Multi-round agent debates rapidly consume token budgets and operational time. Layer 4 imposes strict constraints: a $20 per-incident cap, 500,000 token limit, and a 15-minute diagnosis ceiling (§ 4.20, § 4.21, § 4.33).
3. The ultimate source of truth in software is empirical: code either reproduces the bug or it does not; tests either pass or fail.

## Decision
1. **Defer Multi-Round LLM Debate:**
   - Multi-round LLM critic loops are deferred as a post-MVP research feature. Zero LLM-vs-LLM debate loops will run in the MVP.
2. **Deterministic Empirical Verification:**
   - Grounding and evaluation rely strictly on deterministic platform code:
     - **Evidence Gate:** Every diagnosis must link to at least 1 timestamped evidence artifact.
     - **Rule Engine:** Checks temporal alignment (`t_deploy == t_error ± 30s`) and configuration symbol intersection.
     - **Active Falsification:** Actively searches for disproving evidence and eliminates shared non-discriminating data.
     - **Empirical Sandbox Reproduction:** The candidate cause is tested in an ephemeral Docker container. If reproduced, status is marked `"Confirmed Root Cause"`; otherwise marked `"Likely Root Cause"`.

## Overrides & Traceability
- **Overrides:** Supersedes the proposed multi-round AI critic dialogue discussed in Layer 6 Q6.16.
- **Enforces:** Layer 4 § 4.5, § 4.6, § 4.7, and Layer 5 § 4.

## Consequences
- **Positive:** Predictable spend, fast execution within the 15-minute triage window, and objective verification grounded in real application behavior.
- **Negative:** Relies on deterministic rule quality and sandbox reproduction fidelity.
