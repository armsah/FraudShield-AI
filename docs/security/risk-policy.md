# FraudShield AI — Risk Decision and Enforcement Policy

**Document ID:** FS-SEC-003
**Version:** 1.0.0
**Status:** Proposed — P0 Architecture Baseline
**Owner:** FraudShield AI Security and Architecture
**Last Updated:** 2026-10-21

---

## 1. Purpose

This document defines the deterministic fraud risk decision policy for FraudShield AI.

It specifies how validated transaction events, fraud features, machine-learning inference, authorization controls, and audit evidence are combined to produce trustworthy risk decisions.

The policy separates **fraud risk classification** from **authorization to execute consequential actions**.

A model prediction must never independently authorize a payment, account restriction, transaction decline, or other privileged operation.

The machine-readable companion contract is:

`contracts/risk-policy.v1.json`

---

## 2. Scope

This policy applies to:

- Transaction ingestion and validation.
- Fraud feature enrichment.
- Machine-learning risk inference.
- Deterministic policy evaluation.
- Risk decision persistence.
- Human review and approval.
- Consequential action authorization.
- Decision audit evidence.
- Failure handling and recovery.

It applies to all tenants and processing environments.

Production use requires approval of the policy, thresholds, and operational controls.

---

## 3. Policy Principles

FraudShield AI enforces the following principles:

1. **Verified identity:** Every trusted transaction must originate from an authenticated and authorized principal.
2. **Tenant isolation:** Tenant identity must be derived from verified security context.
3. **Deterministic decisions:** Risk classifications must be produced by a versioned policy engine.
4. **Model separation:** Machine-learning inference supplies evidence, not privileged authorization.
5. **Fail-closed execution:** Missing mandatory security controls must not result in an authorized consequential action.
6. **Idempotency:** Reprocessing an event must not cause duplicate consequential operations.
7. **Auditability:** Every completed risk decision must have protected supporting evidence.
8. **Human oversight:** Operations requiring human approval must not execute before valid approval is recorded.
9. **Version traceability:** Decisions must reference the model and policy versions used.
10. **Controlled change:** Policy changes require review, evaluation, and release authorization.

---

## 4. Decision Types

FraudShield AI defines four risk decision classifications.

| Decision  | Meaning                                       | Operational interpretation                                         |
| --------- | --------------------------------------------- | ------------------------------------------------------------------ |
| `APPROVE` | Transaction has low estimated fraud risk      | Eligible for downstream processing, subject to authorization       |
| `REVIEW`  | Transaction requires additional investigation | Hold or route to an authorized review workflow                     |
| `DECLINE` | Transaction has high estimated fraud risk     | Recommend denial or restriction, subject to authorized enforcement |
| `ABSTAIN` | Reliable classification cannot be produced    | Do not treat as approval; route to controlled failure handling     |

A risk classification is not equivalent to permission to execute a financial operation.

The API may also return transport-level errors, including unauthorized requests and invalid schemas. These are not fraud classifications.

---

## 5. Risk Score Requirements

### 5.1 Score Definition

The fraud model produces a normalized risk score:

$$
0.0 \leq s \leq 1.0
$$

Where:

- `s = 0.0` indicates the lowest modeled fraud risk.
- `s = 1.0` indicates the highest modeled fraud risk.
- Higher scores indicate higher estimated fraud risk.

The score must be:

- Finite.
- Numeric.
- Within the configured range.
- Associated with a verified model version.
- Derived from validated inputs.
- Accompanied by required provenance.

Invalid, missing, non-finite, or out-of-range scores must not be classified as low risk.

### 5.2 Provisional Thresholds

| Score range     | Decision  |
| --------------- | --------- |
| `0.0 ≤ s < 0.3` | `APPROVE` |
| `0.3 ≤ s < 0.7` | `REVIEW`  |
| `0.7 ≤ s ≤ 1.0` | `DECLINE` |

Boundary examples:

| Risk score | Expected decision |
| ---------- | ----------------- |
| `0.00`     | APPROVE           |
| `0.29`     | APPROVE           |
| `0.30`     | REVIEW            |
| `0.69`     | REVIEW            |
| `0.70`     | DECLINE           |
| `1.00`     | DECLINE           |

These thresholds are **provisional engineering defaults**.

They must be calibrated using representative evaluation data, false-positive costs, false-negative costs, tenant requirements, and operational review capacity.

---

## 6. Decision Processing Workflow

A transaction follows this sequence:

1. Authenticate the caller.
2. Verify authorization to submit transactions.
3. Derive the tenant identity from verified security context.
4. Validate the transaction schema.
5. Validate transaction provenance.
6. Apply tenant-scoped idempotency checks.
7. Retrieve permitted fraud features.
8. Execute approved model inference.
9. Validate the risk score and model version.
10. Evaluate the deterministic risk policy.
11. Persist decision evidence.
12. Return or publish the risk classification.
13. Request additional authorization before any consequential operation.
14. Obtain human approval when required.
15. Execute only the authorized operation.
16. Record execution and approval outcomes.

A failure in a mandatory control must not be silently bypassed.

---

## 7. Policy Evaluation Rules

### RULE-001 — Authentication

Unauthenticated transaction submissions must be rejected.

### RULE-002 — Tenant Identity

Tenant identity must be derived from verified authentication context.

Client-controlled tenant identifiers must not override the verified tenant identity.

### RULE-003 — Schema Validation

Transactions that fail required schema validation must be rejected.

### RULE-004 — Idempotency

Duplicate transaction submissions must not produce duplicate consequential actions.

The idempotency scope must include the verified tenant and a stable transaction identifier.

### RULE-005 — Risk Score Validation

Risk scores must be finite numbers within the configured range.

Invalid scores result in `ABSTAIN` when encountered during trusted inference processing.

### RULE-006 — Deterministic Classification

Valid risk scores must be classified using the active versioned risk policy.

### RULE-007 — Model Privilege Separation

Model inference must not directly execute privileged operations.

### RULE-008 — Consequential Action Authorization

Every consequential operation requires authorization for the exact actor, tenant, resource, operation, and arguments.

### RULE-009 — Human Approval

When the policy requires human approval, execution must be blocked until valid approval evidence is available.

### RULE-010 — Audit Evidence

A completed decision must not be issued without the required audit evidence.

### RULE-011 — Required Dependency Failure

Failures of mandatory dependencies must not silently produce an `APPROVE` decision or bypass security controls.

### RULE-012 — Version Traceability

Every completed decision must identify the active policy version and model version.

---

## 8. Failure Handling

The following behavior is required.

| Condition                              | Required outcome                                             |
| -------------------------------------- | ------------------------------------------------------------ |
| Missing authentication                 | Reject request                                               |
| Unauthorized caller                    | Reject request                                               |
| Tenant identity mismatch               | Reject request                                               |
| Invalid transaction schema             | Reject request                                               |
| Missing transaction identifier         | Reject request                                               |
| Invalid client-provided tenant context | Reject request                                               |
| Missing required model features        | ABSTAIN                                                      |
| Model inference unavailable            | ABSTAIN                                                      |
| Invalid model score                    | ABSTAIN                                                      |
| Policy engine unavailable              | ABSTAIN                                                      |
| Audit persistence unavailable          | No completed decision or consequential action                |
| Human approval unavailable             | Block approval-gated execution                               |
| Duplicate transaction                  | Return or recover the existing authorized idempotent outcome |

Where the machine-readable contract specifies `ABSTAIN` for audit persistence failure, this represents the internal fail-safe classification. It must not be interpreted as permission to emit an unaudited completed decision.

An internal failure classification and a successfully persisted decision are distinct states.

---

## 9. Human Review and Approval

### 9.1 Review Workflow

A `REVIEW` classification requires routing to an authorized review workflow.

The review workflow must preserve:

- Transaction identifier.
- Verified tenant identifier.
- Risk score.
- Model version.
- Policy version.
- Reason codes.
- Review status.
- Assigned reviewer.
- Review timestamps.

### 9.2 Approval Requirements

An approval must be:

1. Issued by an authenticated reviewer.
2. Authorized for the relevant tenant and operation.
3. Bound to the exact requested operation.
4. Bound to the target resource.
5. Bound to the operation arguments.
6. Bound to the policy version.
7. Time-limited.
8. Single-use where applicable.
9. Recorded in protected audit evidence.

Approval for one transaction must not authorize another transaction.

Approval must not remain valid after a material change to the requested operation.

### 9.3 Separation of Duties

Where required by risk level or organizational policy, the person requesting an operation must not be the person approving it.

Privileged overrides require separately defined authorization rules and audit evidence.

---

## 10. Consequential Action Enforcement

The policy engine produces a risk classification.

A separate enforcement boundary determines whether an operation may execute.

The enforcement boundary must verify:

- Actor identity.
- Tenant authorization.
- Target resource.
- Requested operation.
- Exact operation arguments.
- Applicable policy version.
- Required approval evidence.
- Idempotency constraints.
- Audit persistence availability.

Consequential actions must not execute solely because the risk decision is `APPROVE` or `DECLINE`.

Examples of consequential actions include:

- Authorizing a payment.
- Declining a payment.
- Freezing an account.
- Restricting a customer.
- Releasing a held transaction.
- Applying a financial override.

---

## 11. Audit Evidence

Every completed decision must include sufficient evidence for reconstruction and investigation.

### 11.1 Required Decision Evidence

- Transaction identifier.
- Verified tenant identifier.
- Correlation identifier.
- Event or request timestamp.
- Decision timestamp.
- Model version.
- Policy version.
- Risk score, when available.
- Risk classification.
- Reason codes.
- Input provenance reference.
- Decision service identity.

### 11.2 Required Approval Evidence

Where approval is required, record:

- Approval identifier.
- Reviewer identity.
- Reviewer authorization context.
- Tenant identifier.
- Target resource.
- Operation name.
- Operation arguments or canonical argument digest.
- Policy version.
- Approval timestamp.
- Expiration timestamp.
- Approval outcome.
- Execution reference.

Audit evidence must be protected against unauthorized modification and deletion.

Sensitive fields must be minimized or redacted according to the data classification standard.

---

## 12. Idempotency and Replay Protection

FraudShield AI must prevent duplicate processing from creating duplicate consequential operations.

The idempotency design must:

1. Scope keys to verified tenant identity.
2. Use stable transaction identifiers.
3. Detect conflicting payloads using a canonical payload digest.
4. Preserve the outcome of completed processing.
5. Support safe recovery after partial failures.
6. Prevent replayed approval evidence from authorizing new operations.
7. Maintain auditable duplicate-processing outcomes.

A repeated request with the same identifier but materially different content must be treated as a conflict rather than silently reused.

---

## 13. Policy Versioning and Change Control

Every policy release must have a unique version.

Policy changes must include:

- Change rationale.
- Version increment.
- Evaluation evidence.
- Security impact review.
- Threshold calibration evidence.
- Compatibility assessment.
- Rollback procedure.
- Authorized release approval.

A policy version must not be silently replaced while retaining the same version identifier.

Decisions must preserve the exact policy version used during evaluation.

---

## 14. Model Evaluation Dependencies

Production promotion requires evaluation of:

- Precision.
- Recall.
- False-positive rate.
- False-negative rate.
- PR-AUC.
- ROC-AUC.
- Threshold calibration.
- Cohort performance.
- Temporal stability.
- Data leakage.
- Delayed labels.
- Operational review capacity.

The release must satisfy the approved evaluation gates.

The initial machine-readable evaluation contract is:

`contracts/evaluation-gates.v1.json`

Provisional numerical gates must not be represented as validated production acceptance criteria.

---

## 15. Monitoring and Operational Controls

Monitoring must cover:

- Decision volume by classification.
- Decision latency.
- Model inference failures.
- ABSTAIN rates.
- Authentication failures.
- Authorization denials.
- Tenant isolation violations.
- Policy version changes.
- Approval failures.
- Audit persistence failures.
- Duplicate transaction rates.
- Model drift.
- Fraud detection performance.
- Human review backlog.

Alerts must distinguish security failures, dependency failures, and model quality degradation.

---

## 16. Security Test Requirements

| Test ID | Scenario                              | Expected result                               |
| ------- | ------------------------------------- | --------------------------------------------- |
| POL-001 | Score `0.00`                          | APPROVE                                       |
| POL-002 | Score `0.29`                          | APPROVE                                       |
| POL-003 | Score `0.30`                          | REVIEW                                        |
| POL-004 | Score `0.69`                          | REVIEW                                        |
| POL-005 | Score `0.70`                          | DECLINE                                       |
| POL-006 | Score `1.00`                          | DECLINE                                       |
| POL-007 | Score below `0.00`                    | ABSTAIN                                       |
| POL-008 | Score above `1.00`                    | ABSTAIN                                       |
| POL-009 | Non-finite score                      | ABSTAIN                                       |
| POL-010 | Unauthenticated request               | Rejected                                      |
| POL-011 | Tenant identity mismatch              | Rejected                                      |
| POL-012 | Missing required features             | ABSTAIN                                       |
| POL-013 | Model inference unavailable           | ABSTAIN                                       |
| POL-014 | Policy engine unavailable             | ABSTAIN                                       |
| POL-015 | Audit persistence unavailable         | No completed decision or consequential action |
| POL-016 | Unauthorized consequential operation  | Denied                                        |
| POL-017 | Approval reused for another operation | Denied                                        |
| POL-018 | Duplicate transaction                 | No duplicate consequential action             |
| POL-019 | Conflicting idempotency payload       | Conflict                                      |
| POL-020 | Missing policy version                | Decision completion blocked                   |

These are acceptance test specifications. They must be implemented in subsequent development phases.

---

## 17. P0 Acceptance Criteria

This policy is ready for P0 sign-off when:

1. All decision types are defined.
2. Risk score boundaries are unambiguous.
3. Failure behavior is documented.
4. Tenant identity rules are explicit.
5. Consequential action authorization is separated from model inference.
6. Human approval requirements are documented.
7. Idempotency requirements are defined.
8. Required audit evidence is specified.
9. Policy versioning is established.
10. The document is consistent with the machine-readable risk policy contract.
11. Security reviewers approve the proposed baseline.

---

## 18. Related Documents

- `docs/architecture/problem-statement.md`
- `docs/architecture/system-context.md`
- `docs/architecture/container-architecture.md`
- `docs/architecture/decision-workflow.md`
- `docs/security/data-classification.md`
- `docs/security/threat-model.md`
- `docs/quality/evaluation-specification.md`
- `docs/architecture/adr/ADR-003-human-approval.md`
- `contracts/risk-policy.v1.json`
- `contracts/evaluation-gates.v1.json`

---

## 19. Approval Status

**Status: Proposed — P0 Architecture Baseline**

This document defines the intended risk decision and enforcement behavior.

The policy thresholds, authorization controls, evaluation gates, and operational procedures must be reviewed and approved before production deployment.
