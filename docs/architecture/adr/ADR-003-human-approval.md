# ADR-003: Require Explicit Authorization and Human Approval for Consequential Actions

**Status:** Proposed
**Date:** 2026-10-21
**Decision Makers:** FraudShield AI Architecture and Security
**Scope:** Fraud decisions, approvals, and privileged execution

## 1. Context

FraudShield AI produces fraud risk classifications using machine-learning inference and deterministic policies.

These classifications may influence consequential financial operations, including:

- Payment authorization.
- Payment decline.
- Account restriction.
- Transaction release.
- Fraud case escalation.
- Administrative overrides.

Machine-learning predictions are probabilistic and may be incorrect.

A model output must therefore not independently authorize privileged execution.

The platform requires an explicit boundary between risk classification and consequential action enforcement.

## 2. Decision

FraudShield AI will separate:

1. Machine-learning inference.
2. Deterministic risk policy evaluation.
3. Authorization.
4. Human approval.
5. Consequential action execution.

The model may produce a risk score.

The policy engine may produce a classification such as `APPROVE`, `REVIEW`, `DECLINE`, or `ABSTAIN`.

Neither the score nor the classification independently grants permission to execute a financial operation.

Every consequential operation requires authorization.

Human approval is additionally required when the applicable policy mandates it.

## 3. Decision Workflow

The proposed workflow is:

1. Receive and validate the transaction.
2. Establish verified tenant identity.
3. Perform fraud inference.
4. Evaluate the deterministic risk policy.
5. Persist the decision and required evidence.
6. Determine whether a consequential operation is requested.
7. Authorize the requested operation.
8. Determine whether human approval is required.
9. Create a pending approval request when required.
10. Present the exact operation and relevant evidence to an authorized reviewer.
11. Verify the reviewer's identity and permissions.
12. Record the approval or rejection.
13. Revalidate authorization and operation binding.
14. Execute the operation only if all controls pass.
15. Persist execution evidence.

An unavailable approval service must not cause an approval-gated operation to execute automatically.

## 4. Approval Binding

Approval evidence must be bound to:

- Verified tenant identifier.
- Requesting actor.
- Approving actor.
- Target resource.
- Operation name.
- Canonical operation arguments or their cryptographic digest.
- Applicable policy version.
- Approval identifier.
- Approval timestamp.
- Expiration timestamp.
- Approval status.

A material change to the requested operation invalidates the existing approval.

## 5. Approval Lifecycle

An approval request may have the following states:

| State     | Meaning                                        |
| --------- | ---------------------------------------------- |
| PENDING   | Awaiting authorized review                     |
| APPROVED  | Authorized reviewer approved the exact request |
| REJECTED  | Reviewer rejected the request                  |
| EXPIRED   | Approval validity period elapsed               |
| CANCELLED | Request withdrawn or invalidated               |
| CONSUMED  | Approval used for its authorized operation     |

State transitions must be controlled and auditable.

Approval evidence must not be reusable for unrelated operations.

## 6. Authorization Requirements

Before execution, the enforcement service must verify:

1. The requesting principal is authorized.
2. The reviewer was authorized to approve.
3. The operation is permitted for the tenant.
4. The target resource matches the approved resource.
5. The arguments match the approved arguments.
6. The policy version is valid for the operation.
7. The approval has not expired.
8. The approval has not been consumed.
9. The operation satisfies idempotency requirements.
10. Required audit persistence is available.

An approval is not a substitute for authorization.

## 7. Separation of Duties

Where required by policy, the requesting actor and approving actor must be different.

Administrative overrides must have explicit authorization rules.

High-impact operations may require additional approval levels.

The precise approval matrix must be established before production deployment.

## 8. Failure Handling

| Failure                         | Required behavior              |
| ------------------------------- | ------------------------------ |
| Missing reviewer authentication | Deny approval                  |
| Reviewer lacks authorization    | Deny approval                  |
| Approval expired                | Deny execution                 |
| Approval already consumed       | Deny replay                    |
| Operation arguments changed     | Deny execution                 |
| Tenant context changed          | Deny execution                 |
| Policy version mismatch         | Deny or require reapproval     |
| Approval service unavailable    | Block approval-gated execution |
| Audit persistence unavailable   | Block consequential execution  |
| Duplicate execution request     | Apply idempotent handling      |

No failure condition may silently bypass approval enforcement.

## 9. Alternatives Considered

### Alternative A: Model-Driven Automatic Execution

Model output directly triggers payment approval, decline, or restriction.

**Advantages:**

- Low operational latency.
- Minimal manual intervention.

**Disadvantages:**

- Excessive trust in probabilistic output.
- Limited human oversight.
- High impact from model errors.
- Greater risk of unauthorized consequential action.

**Decision:** Rejected.

### Alternative B: Human Approval for Every Transaction

All transactions require manual approval.

**Advantages:**

- Strong human oversight.
- Simple approval requirement.

**Disadvantages:**

- High operational cost.
- Excessive review workload.
- Poor scalability.
- Increased processing latency.

**Decision:** Rejected as the default.

### Alternative C: Policy-Gated Human Approval

Deterministic rules determine which operations require human approval.

**Advantages:**

- Risk-based oversight.
- Clear authorization boundaries.
- Scalable review workflow.
- Auditable approvals.
- Controlled privileged execution.

**Disadvantages:**

- More complex workflow management.
- Approval service availability becomes operationally important.
- Additional latency for reviewed operations.

**Decision:** Selected.

## 10. Security Requirements

The implementation must prevent:

- Approval spoofing.
- Approval replay.
- Cross-tenant approval use.
- Unauthorized reviewer access.
- Approval substitution.
- Operation argument tampering.
- Approval after expiration.
- Execution without required evidence.

Approval records must be protected against unauthorized modification.

## 11. Verification

Required tests include:

| Test                                          | Expected result                   |
| --------------------------------------------- | --------------------------------- |
| Model requests privileged action              | Denied                            |
| Unauthorized reviewer approves request        | Denied                            |
| Expired approval used                         | Denied                            |
| Approval reused for different transaction     | Denied                            |
| Operation arguments modified after approval   | Denied                            |
| Cross-tenant approval submitted               | Denied                            |
| Approval service unavailable                  | Execution blocked                 |
| Audit persistence unavailable                 | Execution blocked                 |
| Valid approval for exact authorized operation | Eligible for execution            |
| Duplicate execution attempt                   | No duplicate consequential effect |

## 12. Consequences

### Positive

- Reduced risk of unauthorized financial actions.
- Explicit separation of model output and authorization.
- Stronger auditability.
- Support for human oversight.
- Clear approval lifecycle.

### Negative

- Additional application complexity.
- Operational review workload.
- Longer latency for approval-gated operations.
- Need for approval expiration and replay protection.
- More complex recovery from partial execution failures.

## 13. Related Documents

- `docs/security/risk-policy.md`
- `docs/security/threat-model.md`
- `docs/architecture/decision-workflow.md`
- `docs/quality/non-functional-requirements.md`
- `contracts/risk-policy.v1.json`

## 14. Decision Status

**Proposed — P0 Architecture Baseline**

Acceptance requires security review of authorization enforcement, approval binding, lifecycle controls, and audit persistence.
