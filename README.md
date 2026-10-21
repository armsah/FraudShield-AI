# FraudShield AI

### Cloud-Native Financial Fraud Detection and Risk Decisioning Platform

**FraudShield AI** is an engineering portfolio project demonstrating the architecture and incremental implementation of a secure, event-driven financial fraud detection platform on Microsoft Azure.

The project combines:

- ASP.NET Core and .NET 10.
- Azure-native event processing.
- Machine-learning fraud risk inference.
- Deterministic risk decision policies.
- Multi-tenant security.
- Human approval workflows.
- Protected audit evidence.
- Infrastructure as Code.
- Automated testing and evaluation.

> **Project status:** P0 architecture and design baseline in progress. Application services, cloud infrastructure, machine-learning pipelines, and production capabilities are planned for subsequent implementation phases. Documentation describes intended controls; it does not establish that those controls are already deployed.

---

## 1. Project Objectives

FraudShield AI is designed to demonstrate how a financial fraud detection platform can:

1. Accept authenticated transaction submissions.
2. Validate transaction schemas and provenance.
3. Enforce tenant isolation.
4. Process transaction events asynchronously.
5. Enrich transactions with fraud-related features.
6. Generate machine-learning risk scores.
7. Apply deterministic risk policies.
8. Route uncertain cases for human review.
9. Enforce authorization before consequential actions.
10. Preserve auditable decision evidence.
11. Monitor latency, reliability, drift, and security.
12. Support controlled model and application releases.

The project emphasizes **secure decisioning**, not merely model accuracy.

---

## 2. High-Level Architecture

The proposed processing architecture is:

```text
                    External Transaction Client
                               |
                               v
                     Azure API Management
                               |
                               v
                       ASP.NET Core API
                               |
                  Authentication / Authorization
                               |
                  Tenant Context Verification
                               |
                     Transaction Validation
                               |
                               v
                        Azure Event Hubs
                               |
                               v
                 Event Processing Service
                  (Functions / Container Apps)
                               |
                    Schema and Provenance
                         Validation
                               |
                               v
                      Feature Enrichment
                               |
                               v
                     Azure ML Inference
                               |
                         Risk Score
                               |
                               v
                  Deterministic Policy Engine
                               |
              +----------------+----------------+
              |                |                |
              v                v                v
           APPROVE           REVIEW           DECLINE
              |                |                |
              +----------------+----------------+
                               |
                               v
                    Protected Decision Audit
                               |
                               v
                   Authorization Boundary
                               |
                     Human Approval Where
                            Required
                               |
                               v
                  Consequential Action
                       Enforcement

                  ABSTAIN: Controlled failure
                  or insufficient evidence
```

**Important:** A risk classification is not authorization to execute a financial operation.

A completed decision requires the relevant audit evidence. An asynchronous API acceptance response is not equivalent to a completed fraud decision.

---

## 3. Risk Decision Policy

The initial deterministic policy uses a normalized fraud risk score between `0.0` and `1.0`.

| Risk score                       | Classification | Meaning                             |
| -------------------------------- | -------------- | ----------------------------------- |
| `0.0 ≤ score < 0.3`              | `APPROVE`      | Low estimated fraud risk            |
| `0.3 ≤ score < 0.7`              | `REVIEW`       | Requires further investigation      |
| `0.7 ≤ score ≤ 1.0`              | `DECLINE`      | High estimated fraud risk           |
| Invalid or unavailable inference | `ABSTAIN`      | Reliable classification unavailable |

These thresholds are provisional and require calibration before production use.

Authentication failures, unauthorized requests, invalid schemas, and tenant identity mismatches must be rejected rather than converted into ordinary fraud classifications.

### Fail-Safe Behavior

Critical failures must not silently produce approval.

Examples:

- Model inference unavailable → `ABSTAIN`.
- Required fraud features unavailable → `ABSTAIN`.
- Policy evaluation unavailable → `ABSTAIN`.
- Audit persistence unavailable → no completed unaudited decision.
- Human approval unavailable → block approval-gated execution.

See:

`docs/security/risk-policy.md`

---

## 4. Technology Stack

The planned technology stack includes:

| Layer                       | Technology                              |
| --------------------------- | --------------------------------------- |
| Application framework       | .NET 10 / ASP.NET Core                  |
| API gateway                 | Azure API Management                    |
| Event streaming             | Azure Event Hubs                        |
| Event processing            | Azure Functions / Azure Container Apps  |
| Machine learning            | Azure Machine Learning                  |
| Relational persistence      | PostgreSQL                              |
| Data lake                   | Azure Data Lake Storage                 |
| Authentication and identity | Microsoft Entra ID                      |
| Secrets management          | Azure Key Vault                         |
| Monitoring                  | Azure Monitor / Application Insights    |
| Infrastructure as Code      | Terraform                               |
| Version control             | Git / GitHub                            |
| Development environment     | Windows PowerShell / Visual Studio Code |

These are planned architecture components. Their presence in this table does not imply that they have already been provisioned or implemented.

---

## 5. Security Architecture

FraudShield AI is designed around seven foundational security invariants:

1. **Authenticated processing:** Trusted transactions require authenticated principals.
2. **Verified tenant identity:** Tenant context comes from verified credentials.
3. **Tenant isolation:** Tenant-scoped data and operations must remain isolated.
4. **Model privilege separation:** Model output cannot independently authorize privileged actions.
5. **Consequential action authorization:** Financially consequential operations require explicit authorization.
6. **Protected audit evidence:** Completed decisions require durable, protected evidence.
7. **No silent security bypass:** Mandatory dependency failures must not silently weaken controls.

### Human Approval

When required by policy, approval evidence must be:

- Bound to the exact operation.
- Bound to the actor and tenant.
- Bound to the target resource and arguments.
- Bound to the policy version.
- Time-limited.
- Protected against replay.
- Auditable.

The detailed decisions are documented in:

- `docs/security/threat-model.md`
- `docs/security/data-classification.md`
- `docs/security/risk-policy.md`
- `docs/architecture/adr/ADR-002-tenant-isolation.md`
- `docs/architecture/adr/ADR-003-human-approval.md`

---

## 6. Non-Functional Requirements

The P0 baseline defines the following provisional performance objectives.

| Measurement                    | Provisional P95 target |
| ------------------------------ | ---------------------- |
| API acceptance latency         | ≤ 200 ms               |
| Transaction validation latency | ≤ 100 ms               |
| Model inference latency        | ≤ 300 ms               |
| End-to-end decision latency    | ≤ 1,000 ms             |

The end-to-end measurement includes asynchronous waiting time and completion of required decision persistence.

Additional requirements cover:

- Availability.
- Idempotent processing.
- Retry and replay controls.
- Dead-letter handling.
- Horizontal scalability.
- Tenant-aware resource controls.
- Structured logging and distributed tracing.
- Disaster recovery.
- Model monitoring and drift detection.
- Audit integrity.

These are design targets, not measured performance results.

See:

`docs/quality/non-functional-requirements.md`

---

## 7. Machine-Learning Evaluation

The evaluation framework considers:

- Precision.
- Recall.
- F1 score.
- ROC-AUC.
- PR-AUC.
- False-positive and false-negative rates.
- Calibration.
- Temporal validation.
- Data leakage.
- Cohort performance.
- Drift.
- Delayed fraud labels.

Model quality alone is insufficient for release approval.

A release must also satisfy applicable security, reliability, performance, auditability, and authorization requirements.

The initial evaluation requirements are documented in:

- `docs/quality/evaluation-specification.md`
- `contracts/evaluation-gates.v1.json`

---

## 8. Repository Structure

The P0 repository contains the following architecture and contract artifacts:

```text
FraudShield-AI/
|
+-- contracts/
|   +-- risk-policy.v1.json
|   +-- evaluation-gates.v1.json
|   +-- release-manifest.template.yaml
|
+-- docs/
|   +-- architecture/
|   |   +-- problem-statement.md
|   |   +-- system-context.md
|   |   +-- container-architecture.md
|   |   +-- decision-workflow.md
|   |   |
|   |   +-- adr/
|   |       +-- ADR-001-event-driven-architecture.md
|   |       +-- ADR-002-tenant-isolation.md
|   |       +-- ADR-003-human-approval.md
|   |
|   +-- security/
|   |   +-- data-classification.md
|   |   +-- threat-model.md
|   |   +-- risk-policy.md
|   |
|   +-- quality/
|       +-- non-functional-requirements.md
|       +-- evaluation-specification.md
|
+-- .gitignore
+-- README.md
```

Additional source code, tests, infrastructure definitions, and deployment workflows will be introduced in later phases.

---

## 9. Architecture Decision Records

The project documents three foundational architecture decisions.

| ADR     | Decision                                                                                 |
| ------- | ---------------------------------------------------------------------------------------- |
| ADR-001 | Adopt an event-driven fraud processing architecture                                      |
| ADR-002 | Enforce tenant isolation across processing layers                                        |
| ADR-003 | Require explicit authorization and policy-gated human approval for consequential actions |

These decisions establish the intended implementation direction.

They remain subject to architectural and security review before being marked accepted.

---

## 10. Release Management

FraudShield AI uses versioned contracts and a proposed release manifest.

The release manifest records:

- Source commit.
- Application version.
- Model version.
- Policy version.
- Infrastructure version.
- Evaluation evidence.
- Security results.
- Performance results.
- Reliability results.
- Required approvals.
- Rollback information.

The template is located at:

`contracts/release-manifest.template.yaml`

The template intentionally begins in a draft state with unapproved release gates.

A populated manifest does not itself authorize deployment.

---

## 11. Development Roadmap

FraudShield AI is being developed incrementally.

| Phase     | Focus                                                                                          | Status      |
| --------- | ---------------------------------------------------------------------------------------------- | ----------- |
| P0        | Architecture, security, quality, and contracts                                                 | In progress |
| P1        | .NET solution and application foundations                                                      | Planned     |
| P2 onward | Domain services, event processing, data, ML, security, infrastructure, testing, and deployment | Planned     |

Subsequent phases will implement and verify the capabilities specified in P0.

The roadmap may evolve as architectural decisions and evaluation results are reviewed.

---

## 12. Local Development

### Prerequisites

The initial application development environment is expected to use:

- .NET 10 SDK.
- Git.
- Visual Studio Code or a compatible IDE.
- PowerShell.

Azure and Terraform tooling will be introduced when the corresponding implementation phases begin.

### Verify the .NET SDK

```powershell
dotnet --list-sdks
```

### Verify Git

```powershell
git --version
```

### Check Repository Status

```powershell
git status
```

### Application Execution

Application build and run instructions will be added when the .NET solution is implemented in P1.

At the P0 stage, the repository is primarily a documentation and contract baseline.

---

## 13. Engineering Quality

The project aims to demonstrate:

- Clean architectural boundaries.
- Secure-by-design development.
- Explicit failure handling.
- Deterministic policy behavior.
- Versioned contracts.
- Automated regression testing.
- Reproducible model evaluation.
- Infrastructure as Code.
- Operational observability.
- Controlled release processes.

Security and reliability requirements are treated as system design constraints rather than optional enhancements.

---

## 14. Project Limitations

FraudShield AI is an engineering demonstration project.

At the current P0 stage:

- Fraud model performance has not been experimentally validated.
- Latency targets have not been load-tested.
- Production availability has not been demonstrated.
- Azure resources have not been confirmed as deployed.
- The risk thresholds are provisional.
- Architecture documents are proposed baselines.
- Regulatory compliance certifications are not claimed.

The project must not be represented as a production-certified financial fraud prevention system.

---

## 15. Documentation Index

### Architecture

- [Problem Statement](docs/architecture/problem-statement.md)
- [System Context](docs/architecture/system-context.md)
- [Container Architecture](docs/architecture/container-architecture.md)
- [Decision Workflow](docs/architecture/decision-workflow.md)

### Architecture Decisions

- [ADR-001: Event-Driven Architecture](docs/architecture/adr/ADR-001-event-driven-architecture.md)
- [ADR-002: Tenant Isolation](docs/architecture/adr/ADR-002-tenant-isolation.md)
- [ADR-003: Human Approval](docs/architecture/adr/ADR-003-human-approval.md)

### Security

- [Data Classification](docs/security/data-classification.md)
- [Threat Model](docs/security/threat-model.md)
- [Risk Policy](docs/security/risk-policy.md)

### Quality

- [Non-Functional Requirements](docs/quality/non-functional-requirements.md)
- [Evaluation Specification](docs/quality/evaluation-specification.md)

### Contracts

- [Risk Policy Contract](contracts/risk-policy.v1.json)
- [Evaluation Gates Contract](contracts/evaluation-gates.v1.json)
- [Release Manifest Template](contracts/release-manifest.template.yaml)

---

## 16. Project Status

**Current stage:** P0 — Architecture and Design Baseline.

The repository documents the intended platform architecture and its foundational security, reliability, and evaluation requirements.

Implementation and verification will proceed incrementally in subsequent phases.

---

## License

No license has been declared yet. All rights are reserved unless a license is added to the repository.
