# ADR-12: Composite SHA-256 Approval Fingerprinting (Target State + Canonical Diff)

## Status
Accepted

## Context (First Principles)
Human approval authorizes a production deployment. However, between the moment an engineer clicks "Approve" and the moment deployment begins, two attack vectors exist:
1. **Patch Tampering:** The diff itself could be altered (maliciously or via a software bug) to inject unauthorized code.
2. **State Drift:** A teammate could push a new commit or modify an environment configuration while approval was pending, causing the approved patch to apply against a different baseline, resulting in catastrophic production breakage.

Previous documentation had a subtle divergence:
- Layer 4 § 4.32 defined state fingerprinting: `Hash(Commit SHA + Config Files + Env Vars + Error Signature)`.
- Layer 5 § 6 defined patch fingerprinting: `Hash(target_path + base_commit_sha + raw_diff_bytes)`.

## Decision
1. **Composite SHA-256 Fingerprint:**
   - When the human engineer approves a patch, the platform calculates a unified composite SHA-256 fingerprint binding both the patch contents and the baseline target state:
     $$\text{Fingerprint } H_0 = \text{SHA256}(\text{base\_commit\_sha} + \text{canonical\_diff\_bytes} + \text{target\_path} + \text{env\_hash})$$
2. **Pre-Execution Verification:**
   - The isolated `AI Deployment Executor` recalculates $H_{\text{live}}$ immediately before initiating the CI/CD pipeline.
   - If $H_{\text{live}} \ne H_0$, or if the 1-hour TTL has expired, or if approval status is not `APPROVED`, execution immediately aborts and transitions to `INVALIDATED`.
3. **Self-Approval Guardrail:**
   - In accordance with separation of duties, the engineer whose commit or configuration change triggered the incident is blocked by default from approving the remediation deployment. A distinct peer on-call engineer must grant approval.

## Overrides & Traceability
- **Reconciles:** Layer 4 § 4.32 with Layer 5 § 6, Layer 5 § 10 (Open Decision 3), and Layer 6 Q6.18.

## Consequences
- **Positive:** Complete protection against post-approval tampering and accidental baseline drift.
- **Negative:** Fast-moving repositories might invalidate approvals if background commits land during the review window, requiring a quick re-generation of the review package.
