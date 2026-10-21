# FraudShield AI — Security Threat Model

**Document ID:** FS-SEC-002
**Version:** 1.0.0
**Status:** Proposed — P0 Architecture Baseline
**Owner:** FraudShield AI Security and Architecture
**Last Updated:** 2026-10-21

---

## 1. Purpose

This document identifies security threats, attack surfaces, trust boundaries, mitigation controls, and verification requirements for FraudShield AI.

The platform provides real-time fraud detection and risk decisioning using Azure services, deterministic risk policies, machine-learning inference, and human approval workflows.

The primary security objective is to prevent unauthorized, incorrect, or unauditable financial risk decisions.

This threat model uses the STRIDE methodology:

- **S — Spoofing**
- **T — Tampering**
- **R — Repudiation**
- **I — Information Disclosure**
- **D — Denial of Service**
- **E — Elevation of Privilege**

Threats are assessed against the proposed P0 architecture. Actual control implementation must be verified during subsequent development phases.

---

## 2. System Scope

### 2.1 In-Scope Components

The following components are included:

| Component                            | Security responsibility                                    |
| ------------------------------------ | ---------------------------------------------------------- |
| Azure API Management                 | External API boundary, authentication and traffic controls |
| ASP.NET Core API                     | Request validation, authorization, tenant context          |
| Azure Event Hubs                     | Transaction event ingestion                                |
| Azure Functions / Container Apps     | Event processing and orchestration                         |
| Feature enrichment services          | Retrieve and validate fraud features                       |
| Azure Machine Learning               | Fraud risk inference                                       |
| Deterministic policy engine          | Convert risk evidence into decisions                       |
| PostgreSQL                           | Transaction, decision, and workflow persistence            |
| Azure Data Lake Storage              | Historical events, evaluation data, and evidence           |
| Human approval workflow              | Authorized review of consequential actions                 |
| Azure Key Vault                      | Secret and key management                                  |
| Microsoft Entra ID                   | Identity and access management                             |
| Azure Monitor / Application Insights | Observability and security telemetry                       |
| Terraform                            | Infrastructure provisioning and configuration              |

### 2.2 Out-of-Scope Components

The following are outside direct platform control:

- External banks' internal authorization systems.
- Third-party payment networks.
- Customer-owned endpoint security.
- External identity providers beyond configured integration boundaries.
- Downstream systems after successful authorized handoff.

Interactions with these systems remain within scope as potential attack surfaces.

---

## 3. Security Objectives

FraudShield AI must satisfy seven security invariants.

**INV-01 — Authentication**

Unauthenticated requests must not enter trusted transaction processing.

**INV-02 — Tenant Identity**

Tenant identity must be derived from verified authentication context, not trusted from client-controlled fields.

**INV-03 — Tenant Isolation**

Transactions, features, decisions, and audit records must not cross tenant boundaries without explicit authorization.

**INV-04 — Model Privilege Separation**

Machine-learning outputs must not directly execute privileged operations.

**INV-05 — Consequential Action Authorization**

Consequential operations require explicit authorization and human approval where policy requires it.

**INV-06 — Audit Evidence**

Every completed risk decision must have sufficient protected audit evidence.

**INV-07 — Dependency Failure Safety**

Required dependency failures must not silently bypass validation, authorization, policy enforcement, or audit requirements.

---

## 4. Assets Requiring Protection

| Asset ID | Asset                            | Classification | Primary concern                 |
| -------- | -------------------------------- | -------------- | ------------------------------- |
| A-01     | Financial transaction records    | C3             | Confidentiality and integrity   |
| A-02     | Customer and device identifiers  | C3             | Confidentiality                 |
| A-03     | Fraud features                   | C3             | Integrity and tenant isolation  |
| A-04     | Risk scores and reason codes     | C3             | Integrity                       |
| A-05     | Risk policy configuration        | C3             | Integrity and authorization     |
| A-06     | Decision and approval evidence   | C3             | Integrity and accountability    |
| A-07     | Authentication tokens            | C4             | Confidentiality                 |
| A-08     | API credentials and secrets      | C4             | Confidentiality                 |
| A-09     | Cryptographic keys               | C4             | Confidentiality and integrity   |
| A-10     | Model artifacts                  | C3             | Integrity and provenance        |
| A-11     | Training and evaluation datasets | C3             | Confidentiality and integrity   |
| A-12     | Deployment configuration         | C2/C4          | Integrity and secret protection |

Classifications follow `docs/security/data-classification.md`.

---

## 5. Trust Boundaries

The threat model recognizes six primary trust boundaries.

| Boundary | Description                                                       | Required controls                                                    |
| -------- | ----------------------------------------------------------------- | -------------------------------------------------------------------- |
| TB-01    | External clients to API ingress                                   | Authentication, TLS, throttling, schema validation                   |
| TB-02    | API ingress to trusted processing                                 | Verified identity, tenant context, service authorization             |
| TB-03    | Event ingestion to processing services                            | Managed identity, schema validation, provenance, idempotency         |
| TB-04    | Processing services to model inference                            | Service authentication, input validation, model version verification |
| TB-05    | Decision services to privileged execution                         | Deterministic policy, authorization, approval binding                |
| TB-06    | Application services to persistent data and external integrations | Tenant isolation, least privilege, encryption, audit controls        |

The boundary identifiers must be reconciled with the authoritative system-context diagram before P0 sign-off.

---

## 6. Threat Actors

### 6.1 External Attackers

Potential objectives include:

- Unauthorized transaction submission.
- Credential theft.
- Denial-of-service attacks.
- Exposure of sensitive transaction data.
- Fraud decision manipulation.

### 6.2 Malicious or Compromised Tenants

Potential objectives include:

- Accessing another tenant's transactions.
- Manipulating tenant identity.
- Exploiting insecure feature retrieval.
- Circumventing transaction quotas.
- Accessing another tenant's decision evidence.

### 6.3 Compromised Service Identity

Potential objectives include:

- Unauthorized access to storage.
- Modification of policy configurations.
- Retrieval of secrets.
- Forgery of trusted processing events.

### 6.4 Malicious Insider

Potential objectives include:

- Unauthorized policy changes.
- Fraudulent approval.
- Suppression of audit evidence.
- Unauthorized export of customer information.

### 6.5 Adversarial Transaction Submitter

Potential objectives include:

- Manipulating fraud features.
- Exploiting model weaknesses.
- Evading fraud thresholds.
- Replaying previously accepted transactions.

---

## 7. STRIDE Threat Register

### 7.1 Spoofing

| Threat ID | Scenario                                             | Impact                    | Mitigation                                            |
| --------- | ---------------------------------------------------- | ------------------------- | ----------------------------------------------------- |
| S-01      | Attacker submits requests using stolen credentials   | Unauthorized transactions | Token validation, short-lived credentials, monitoring |
| S-02      | Client supplies another tenant's identifier          | Cross-tenant access       | Server-derived tenant context                         |
| S-03      | Compromised service impersonates a trusted processor | Forged events             | Managed identities and service authorization          |
| S-04      | Attacker impersonates an authorized reviewer         | Fraudulent approval       | Strong authentication, role checks, approval binding  |

### 7.2 Tampering

| Threat ID | Scenario                                      | Impact                   | Mitigation                                                     |
| --------- | --------------------------------------------- | ------------------------ | -------------------------------------------------------------- |
| T-01      | Transaction fields modified during processing | Incorrect fraud decision | TLS, schema validation, immutable event identifiers            |
| T-02      | Risk score altered before policy evaluation   | Incorrect decision       | Trusted service boundary, integrity controls                   |
| T-03      | Fraud policy modified without authorization   | Unauthorized approvals   | Versioned policy, restricted changes, deployment approval      |
| T-04      | Audit records altered or deleted              | Loss of accountability   | Restricted writes, protected retention, integrity verification |
| T-05      | Model artifact replaced                       | Compromised inference    | Model registry controls, artifact provenance, release approval |

### 7.3 Repudiation

| Threat ID | Scenario                                                  | Impact                            | Mitigation                                     |
| --------- | --------------------------------------------------------- | --------------------------------- | ---------------------------------------------- |
| R-01      | Actor denies submitting a transaction                     | Investigation difficulty          | Authenticated identity and correlation IDs     |
| R-02      | Reviewer denies approving an action                       | Accountability failure            | Approval evidence bound to actor and operation |
| R-03      | Administrator denies modifying a policy                   | Untraceable configuration changes | Version control, deployment logs, audit trail  |
| R-04      | Processing service produces a decision without provenance | Unverifiable decision             | Mandatory decision evidence                    |

### 7.4 Information Disclosure

| Threat ID | Scenario                                              | Impact                 | Mitigation                                       |
| --------- | ----------------------------------------------------- | ---------------------- | ------------------------------------------------ |
| I-01      | Tenant accesses another tenant's records              | Data breach            | Tenant-scoped authorization and isolation tests  |
| I-02      | Secrets appear in logs                                | Credential compromise  | Secret redaction and log review                  |
| I-03      | Sensitive datasets are exported without authorization | Confidentiality breach | Access controls, export restrictions, monitoring |
| I-04      | Feature retrieval exposes another tenant's data       | Privacy and fraud risk | Verified tenant context and scoped queries       |
| I-05      | Diagnostic endpoints expose internal information      | Information leakage    | Restricted diagnostics and safe error responses  |

### 7.5 Denial of Service

| Threat ID | Scenario                                    | Impact                       | Mitigation                                      |
| --------- | ------------------------------------------- | ---------------------------- | ----------------------------------------------- |
| D-01      | Excessive API requests exhaust resources    | Service unavailability       | Rate limits, quotas, admission control          |
| D-02      | Event backlog overwhelms processors         | Delayed decisions            | Backpressure, autoscaling, monitoring           |
| D-03      | Model inference becomes unavailable         | Incomplete decisions         | Timeouts, circuit breakers, ABSTAIN             |
| D-04      | Database becomes unavailable                | Decision persistence failure | Retry policy, recovery procedures, safe failure |
| D-05      | Malformed events repeatedly fail processing | Queue congestion             | Bounded retries and dead-letter handling        |

### 7.6 Elevation of Privilege

| Threat ID | Scenario                                          | Impact                        | Mitigation                                   |
| --------- | ------------------------------------------------- | ----------------------------- | -------------------------------------------- |
| E-01      | Model output triggers privileged execution        | Unauthorized financial action | Model-policy-execution separation            |
| E-02      | Reviewer approves an operation outside their role | Unauthorized approval         | Role- and resource-scoped authorization      |
| E-03      | Service identity has excessive permissions        | Broad compromise              | Least-privilege RBAC                         |
| E-04      | Approval token reused for another operation       | Authorization bypass          | Single-use, operation-bound approval         |
| E-05      | Compromised CI/CD credentials alter production    | Unauthorized deployment       | Protected environments and release approvals |

---

## 8. Fraud-Specific Threat Scenarios

### 8.1 Transaction Replay

**Scenario:** An attacker resubmits a previously processed transaction.

**Required controls:**

1. Stable transaction identifiers.
2. Tenant-scoped idempotency keys.
3. Duplicate detection.
4. Defined replay behavior.
5. Auditable duplicate outcomes.

**Expected result:** A duplicate must not create a second consequential action.

### 8.2 Risk Score Manipulation

**Scenario:** An attacker attempts to influence or replace the model-generated risk score.

**Required controls:**

1. Trusted inference endpoint.
2. Model version tracking.
3. Input provenance validation.
4. Risk score range validation.
5. Deterministic policy evaluation.

**Expected result:** Untrusted risk scores must not authorize transactions.

### 8.3 Model Evasion

**Scenario:** An adversary modifies transaction behavior to avoid detection.

**Required controls:**

1. Adversarial evaluation scenarios.
2. Behavioral feature monitoring.
3. Drift detection.
4. Cohort-level performance analysis.
5. Controlled model updates.

**Expected result:** Evasion risks are measured and incorporated into model evaluation.

### 8.4 Data Poisoning

**Scenario:** Malicious or incorrect training records influence the fraud model.

**Required controls:**

1. Dataset provenance.
2. Label quality assessment.
3. Training data access controls.
4. Dataset versioning.
5. Training-serving consistency checks.

**Expected result:** Untrusted training data cannot silently enter approved model releases.

### 8.5 Human Approval Abuse

**Scenario:** An attacker attempts to approve an unauthorized or modified operation.

**Required controls:**

1. Verified reviewer identity.
2. Role-based authorization.
3. Approval bound to exact operation and arguments.
4. Approval expiration.
5. Single-use approval enforcement.
6. Protected approval audit evidence.

**Expected result:** An approval cannot be transferred to another operation.

---

## 9. Failure Handling Requirements

| Failure condition                   | Required behavior                                        |
| ----------------------------------- | -------------------------------------------------------- |
| Invalid authentication              | Reject request                                           |
| Tenant identity mismatch            | Reject request                                           |
| Invalid transaction schema          | Reject request                                           |
| Missing required provenance         | Reject or abstain according to processing stage          |
| Inference unavailable               | ABSTAIN                                                  |
| Required features unavailable       | ABSTAIN                                                  |
| Invalid model score                 | ABSTAIN                                                  |
| Policy engine unavailable           | ABSTAIN                                                  |
| Audit persistence unavailable       | No completed decision or consequential action            |
| Approval service unavailable        | Do not execute approval-gated operation                  |
| Duplicate transaction               | Return or recover the authorized idempotent outcome      |
| Persistent event processing failure | Route to controlled failure handling / dead-letter queue |

ABSTAIN is not equivalent to APPROVE.

Transport-level rejection must remain distinct from fraud decision classification.

---

## 10. Security Controls

### 10.1 Identity and Access Management

- Use Microsoft Entra ID where applicable.
- Prefer managed identities for Azure service communication.
- Validate issuer, audience, signature, and expiration of tokens.
- Apply least-privilege RBAC.
- Separate service identities by responsibility.
- Require additional authorization for administrative operations.

### 10.2 API Security

- Enforce HTTPS.
- Validate incoming schemas.
- Apply request size limits.
- Apply rate limiting.
- Avoid exposing internal exception details.
- Generate correlation identifiers.
- Enforce tenant-scoped authorization.

### 10.3 Event Security

- Authenticate event producers and consumers.
- Validate event schema versions.
- Preserve event provenance.
- Enforce idempotency.
- Apply bounded retries.
- Route poison messages to dead-letter handling.
- Prevent unauthorized event replay.

### 10.4 Data Security

- Encrypt C3 and C4 data.
- Restrict database access.
- Apply tenant-aware queries.
- Redact sensitive logs.
- Control dataset exports.
- Protect audit evidence from unauthorized modification.

### 10.5 Model Security

- Validate model artifacts.
- Record model versions.
- Monitor inference quality.
- Validate score ranges.
- Prevent model outputs from directly invoking privileged operations.
- Require evaluation evidence before model promotion.

### 10.6 Approval Security

- Authenticate reviewers.
- Authorize each requested operation.
- Bind approvals to exact operation arguments.
- Prevent approval replay.
- Record approval evidence.
- Revalidate authorization before execution.

---

## 11. Security Verification Plan

The following tests are required during implementation.

| Test ID | Test scenario                                              | Expected result                               |
| ------- | ---------------------------------------------------------- | --------------------------------------------- |
| SEC-001 | Request without authentication                             | Rejected                                      |
| SEC-002 | Client-controlled tenant ID conflicts with verified tenant | Rejected                                      |
| SEC-003 | Tenant A requests Tenant B data                            | Denied                                        |
| SEC-004 | Invalid transaction schema                                 | Rejected                                      |
| SEC-005 | Duplicate transaction event                                | No duplicate consequential action             |
| SEC-006 | Inference dependency unavailable                           | ABSTAIN                                       |
| SEC-007 | Invalid risk score                                         | ABSTAIN                                       |
| SEC-008 | Policy service unavailable                                 | ABSTAIN                                       |
| SEC-009 | Audit persistence unavailable                              | No completed decision or consequential action |
| SEC-010 | Model output requests privileged execution                 | Denied                                        |
| SEC-011 | Reviewer lacks required authorization                      | Denied                                        |
| SEC-012 | Approval reused for another operation                      | Denied                                        |
| SEC-013 | Expired approval submitted                                 | Denied                                        |
| SEC-014 | Secret included in application log                         | Test fails                                    |
| SEC-015 | Unauthorized policy modification                           | Denied                                        |
| SEC-016 | Cross-tenant feature retrieval                             | Denied                                        |
| SEC-017 | Malformed event exceeds retry limit                        | Controlled dead-letter handling               |
| SEC-018 | Unauthorized event replay                                  | Denied                                        |
| SEC-019 | Missing required provenance                                | Rejected or abstained                         |
| SEC-020 | Release lacks required security evidence                   | Release blocked                               |

These tests are specifications, not claims that the controls are already implemented.

---

## 12. Risk Assessment

Risks are prioritized using qualitative likelihood and impact assessments.

| Risk ID  | Risk                              | Likelihood | Impact   | Priority |
| -------- | --------------------------------- | ---------- | -------- | -------- |
| RISK-001 | Cross-tenant data leakage         | Medium     | Critical | Critical |
| RISK-002 | Unauthorized consequential action | Medium     | Critical | Critical |
| RISK-003 | Fraud policy tampering            | Medium     | Critical | Critical |
| RISK-004 | Model artifact compromise         | Medium     | High     | High     |
| RISK-005 | Credential compromise             | High       | High     | Critical |
| RISK-006 | Audit evidence tampering          | Medium     | High     | High     |
| RISK-007 | Transaction replay                | High       | High     | Critical |
| RISK-008 | Model inference outage            | Medium     | High     | High     |
| RISK-009 | Training data poisoning           | Medium     | High     | High     |
| RISK-010 | Event processing backlog          | High       | Medium   | High     |

These are provisional design-stage ratings. Production risk acceptance requires an approved risk assessment methodology and accountable risk owners.

---

## 13. Residual Risks

Even after implementing the proposed controls, residual risks remain.

Examples include:

- Fraud patterns not represented in training data.
- Compromise of trusted administrative identities.
- Delayed fraud labels.
- Novel adversarial evasion techniques.
- Configuration errors in Azure resources.
- Third-party service failures.
- Incorrectly calibrated risk thresholds.

Residual risks must be documented, assigned to owners, and reviewed before production deployment.

---

## 14. P0 Threat Model Acceptance Criteria

This threat model is ready for P0 sign-off when:

1. All major system components are represented.
2. Trust boundaries match the architecture documentation.
3. All STRIDE categories are addressed.
4. Critical assets have documented protections.
5. High-priority threats have mitigation plans.
6. Security invariants are explicitly covered.
7. Security tests are mapped to identified threats.
8. Failure behavior is consistent with the risk policy.
9. Residual risks are documented.
10. Security and architecture reviewers approve the baseline.

---

## 15. Related Documents

- `docs/architecture/problem-statement.md`
- `docs/architecture/system-context.md`
- `docs/architecture/container-architecture.md`
- `docs/architecture/decision-workflow.md`
- `docs/security/data-classification.md`
- `docs/security/risk-policy.md`
- `docs/architecture/adr/ADR-002-tenant-isolation.md`
- `docs/architecture/adr/ADR-003-human-approval.md`
- `contracts/risk-policy.v1.json`
- `contracts/evaluation-gates.v1.json`

---

## 16. Approval Status

**Status: Proposed**

This document defines the intended security controls and verification requirements for the P0 architecture baseline.

The controls must be implemented, tested, and reviewed before production deployment.

Threat modeling must be revisited when architecture, data flows, external integrations, model capabilities, or privileged operations change.
