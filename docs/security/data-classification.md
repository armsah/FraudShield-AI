# FraudShield AI — Data Classification and Protection Standard

**Document ID:** FS-SEC-001  
**Version:** 1.0.0  
**Status:** Proposed — P0 Architecture Baseline  
**Owner:** FraudShield AI Security and Architecture  
**Last Updated:** 2026-10-21

---

## 1. Purpose

This document establishes the data classification, handling,
storage, access control, encryption, retention, and disposal
requirements for FraudShield AI.

The objective is to protect financial transaction information,
customer identifiers, fraud intelligence, model outputs, and
security evidence throughout their lifecycle.

This document applies to all FraudShield AI components,
including Azure API Management, ASP.NET Core services,
Azure Event Hubs, Azure Functions, Azure Machine Learning,
PostgreSQL, Azure Data Lake Storage, and monitoring systems.

---

## 2. Classification Levels

FraudShield AI uses four data classification levels.

| Level | Classification | Description                                                                    |
| ----- | -------------- | ------------------------------------------------------------------------------ |
| C1    | Public         | Information approved for public disclosure                                     |
| C2    | Internal       | Operational information intended for authorized personnel                      |
| C3    | Confidential   | Sensitive business, transaction, or customer information                       |
| C4    | Restricted     | Credentials, highly sensitive financial data, and privileged security material |

Classification follows the highest sensitivity of the information
contained in a dataset or record.

Derived data inherits the classification of its source unless a
documented de-identification assessment establishes otherwise.

---

## 3. Data Inventory

| Data category              | Examples                                           | Classification |
| -------------------------- | -------------------------------------------------- | -------------- |
| Public documentation       | Architecture summaries, public API descriptions    | C1             |
| Operational metadata       | Deployment identifiers, service health             | C2             |
| Transaction records        | Amount, currency, merchant, timestamp              | C3             |
| Customer identifiers       | Customer ID, account reference                     | C3             |
| Device intelligence        | Device identifiers, IP addresses                   | C3             |
| Fraud features             | Velocity, transaction history, behavioral signals  | C3             |
| Model outputs              | Risk scores, reason codes, fraud predictions       | C3             |
| Decision records           | APPROVE, REVIEW, DECLINE, ABSTAIN                  | C3             |
| Audit evidence             | Actor, decision, policy version, approval evidence | C3             |
| Authentication tokens      | Access tokens, refresh tokens                      | C4             |
| Secrets                    | API keys, connection strings, private keys         | C4             |
| Cryptographic key material | Encryption keys, signing keys                      | C4             |

Where data elements create additional risk through combination,
the combined dataset must be reclassified accordingly.

---

## 4. Data Minimization

FraudShield AI must collect and process only information
necessary for fraud detection, decisioning, security,
operational monitoring, and approved evaluation.

Requirements:

1. Do not ingest unnecessary personally identifiable information.
2. Do not store payment card security codes.
3. Do not store raw authentication credentials.
4. Prefer tokenized or pseudonymized customer identifiers.
5. Remove unnecessary personal attributes from model features.
6. Avoid including sensitive payloads in application logs.
7. Limit access to datasets based on documented business purpose.

Sensitive information must not be used for unrelated model
training or analytics without an approved legal basis and
appropriate governance.

---

## 5. Tenant Isolation

Tenant identity must be established from verified authentication
and authorization context.

A client-provided tenant identifier must not be treated as
authoritative.

Every tenant-scoped transaction, feature, decision, and audit
record must carry a validated tenant identifier.

Controls must include:

- Tenant-scoped authorization.
- Tenant-aware database queries.
- Tenant-aware feature retrieval.
- Tenant-aware event processing.
- Tenant isolation tests.
- Prevention of cross-tenant model feature leakage.
- Restricted access to tenant-specific audit evidence.

Cross-tenant access must be denied unless explicitly authorized
through a separately governed administrative workflow.

---

## 6. Encryption Requirements

### 6.1 Encryption in Transit

Sensitive data must use encrypted transport.

Requirements:

- Use TLS 1.2 or later.
- Prefer TLS 1.3 where supported.
- Reject insecure transport for production services.
- Authenticate service-to-service communication.
- Validate certificates and service identities.

### 6.2 Encryption at Rest

C3 and C4 data must be encrypted at rest.

Applicable services include:

- Azure Database for PostgreSQL.
- Azure Data Lake Storage.
- Azure Event Hubs.
- Azure Machine Learning storage.
- Backup and recovery storage.

Use Azure-managed encryption capabilities and customer-managed
keys where required by organizational policy.

### 6.3 Key Management

Secrets and cryptographic keys must be managed through
Azure Key Vault or an approved equivalent.

Requirements:

- No hardcoded production secrets.
- No secrets committed to Git.
- Prefer managed identities.
- Apply least-privilege access.
- Rotate credentials according to policy.
- Audit key and secret access.
- Separate development and production credentials.

---

## 7. Access Control

FraudShield AI must enforce least privilege.

Access decisions must consider:

- Verified identity.
- Tenant context.
- Assigned role.
- Requested resource.
- Requested operation.
- Applicable authorization policy.

Privileged operations must not be authorized solely by
machine-learning model output.

Consequential actions require explicit authorization and,
where policy requires, human approval.

Approval evidence must be bound to the exact operation,
resource, actor, arguments, and policy version.

---

## 8. Data Retention

Retention periods must be approved according to legal,
contractual, operational, and business requirements.

The following are provisional design targets,
not approved production retention periods.

| Data category               | Proposed retention         |
| --------------------------- | -------------------------- |
| Raw transaction events      | 90 days                    |
| Fraud features              | 180 days                   |
| Model evaluation datasets   | 365 days                   |
| Decision records            | 365 days                   |
| Audit evidence              | 365 days                   |
| Application diagnostic logs | 30 days                    |
| Authentication tokens       | Minimum necessary lifetime |

Retention configuration must support applicable legal holds,
regulatory requirements, and contractual obligations.

Where mandatory retention requirements exceed these targets,
the mandatory requirements take precedence.

Deletion must be controlled, auditable, and consistent with
backup retention policies.

---

## 9. Logging and Monitoring

Logs must not expose:

- Access tokens.
- Passwords.
- API keys.
- Private keys.
- Full sensitive transaction payloads.
- Unnecessary customer identifiers.

Use structured logging with appropriate redaction.

Security-relevant events must include sufficient context
to support investigation without exposing unnecessary data.

Audit records must preserve:

- Transaction identifier.
- Verified tenant identifier.
- Actor or service identity.
- Model version.
- Policy version.
- Risk decision.
- Decision timestamp.
- Reason codes.
- Approval evidence, when applicable.

Audit evidence must be protected against unauthorized
modification and deletion.

---

## 10. Data Provenance

FraudShield AI must preserve the provenance of information
used to generate fraud decisions.

Required provenance attributes include:

- Source system.
- Event identifier.
- Ingestion timestamp.
- Schema version.
- Tenant identifier.
- Feature transformation version.
- Model version.
- Policy version.

Untrusted data must not be treated as authoritative without
validation.

Missing required provenance must trigger rejection or
abstention according to the relevant processing stage.

---

## 11. Model Training and Evaluation

Training and evaluation datasets must be governed.

Requirements:

1. Record dataset versions.
2. Preserve data lineage.
3. Prevent training-serving leakage.
4. Prevent cross-tenant feature leakage.
5. Apply access controls to training datasets.
6. Evaluate delayed fraud labels.
7. Document feature transformations.
8. Remove unnecessary personal information.
9. Control dataset export and sharing.
10. Maintain reproducible evaluation evidence.

Production transaction data must not be exported to
unapproved external services.

---

## 12. Incident Response

Suspected data exposure must trigger the security
incident response process.

Required activities include:

1. Identify affected systems.
2. Contain unauthorized access.
3. Preserve relevant evidence.
4. Determine affected data classifications.
5. Assess tenant impact.
6. Rotate compromised credentials.
7. Perform required notifications.
8. Document remediation.
9. Validate corrective controls.

Incident evidence must be accessible only to authorized
responders.

---

## 13. Security Acceptance Criteria

This standard is satisfied when:

- Every data category has an assigned classification.
- Sensitive data is encrypted in transit and at rest.
- Production secrets are excluded from source control.
- Tenant isolation controls are documented and tested.
- Logging redacts sensitive information.
- Retention requirements are approved and enforced.
- Data provenance is preserved.
- Audit evidence is protected.
- Privileged operations require appropriate authorization.
- Security controls are included in automated evaluation.

---

## 14. Related Documents

- docs/architecture/system-context.md
- docs/architecture/container-architecture.md
- docs/architecture/decision-workflow.md
- docs/security/threat-model.md
- docs/security/risk-policy.md
- docs/architecture/adr/ADR-002-tenant-isolation.md
- docs/architecture/adr/ADR-003-human-approval.md
- contracts/risk-policy.v1.json

---

## 15. Approval Status

This document defines a proposed P0 security baseline.

Final data retention periods, legal requirements,
regulatory applicability, and production security controls
must be reviewed and approved before deployment.
