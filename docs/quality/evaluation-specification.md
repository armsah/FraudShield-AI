# FraudShield AI — Evaluation Specification

**Document ID:** FS-QUAL-002
**Version:** 1.0.0
**Status:** Proposed — P0 Architecture Baseline
**Owner:** FraudShield AI Engineering and Machine Learning
**Last Updated:** 2026-10-21

---

## 1. Purpose

This document defines the evaluation framework for FraudShield AI.

The evaluation framework establishes the evidence required to determine whether a fraud detection model, deterministic risk policy, application release, or infrastructure change is suitable for deployment.

Evaluation covers:

- Fraud detection effectiveness.
- Model discrimination and calibration.
- Threshold selection.
- Data quality and provenance.
- Temporal validation.
- Cohort performance.
- Model drift.
- API and inference latency.
- Reliability and failure handling.
- Tenant isolation and authorization.
- Human approval enforcement.
- Audit evidence.
- Release readiness.

The machine-readable companion contract is:

`contracts/evaluation-gates.v1.json`

All numerical thresholds in the initial contract are provisional and must be calibrated before production deployment.

---

## 2. Evaluation Principles

FraudShield AI follows these principles:

1. **Reproducibility:** Evaluation results must be reproducible from versioned artifacts.
2. **Temporal integrity:** Evaluation must avoid future-data leakage.
3. **Representative data:** Evaluation datasets must reflect the intended operating environment.
4. **Class imbalance awareness:** Fraud prevalence must be considered when interpreting metrics.
5. **Cohort visibility:** Aggregate performance must not conceal important cohort regressions.
6. **Operational relevance:** Evaluation must account for review capacity and decision latency.
7. **Security enforcement:** Model quality cannot compensate for failed security controls.
8. **Auditability:** Every release decision must reference supporting evidence.
9. **Controlled promotion:** Model and policy changes require explicit release authorization.
10. **Safe failure:** Unavailable or invalid inference must not silently produce approval.

---

## 3. Evaluation Scope

The evaluation framework applies to four release categories.

| Release category       | Required evaluation                                                 |
| ---------------------- | ------------------------------------------------------------------- |
| Fraud model release    | Data quality, discrimination, calibration, cohorts, drift, security |
| Risk policy release    | Threshold behavior, authorization, auditability, regression         |
| Application release    | Functional, performance, reliability, security                      |
| Infrastructure release | Security configuration, availability, recovery, performance         |

Changes affecting multiple categories must satisfy the applicable requirements from each category.

---

## 4. Evaluation Artifacts

Each evaluation must identify:

- Evaluation run identifier.
- Evaluation timestamp.
- Dataset version.
- Dataset provenance.
- Model version.
- Policy version.
- Feature transformation version.
- Application version.
- Evaluation code version.
- Environment.
- Configuration.
- Metric results.
- Test outcomes.
- Known limitations.
- Reviewer identity.
- Approval status.

Evidence must be stored in a controlled location with appropriate access restrictions.

---

## 5. Dataset Requirements

### EVAL-DATA-001 — Dataset Provenance

Every evaluation dataset must have documented provenance.

Required information includes:

- Source system.
- Collection period.
- Extraction timestamp.
- Dataset version.
- Label source.
- Label availability date.
- Relevant transformation versions.

### EVAL-DATA-002 — Schema Validation

Evaluation records must conform to approved schemas.

Malformed or incompatible records must be reported.

### EVAL-DATA-003 — Missing Data

Missing values must be measured and documented.

The evaluation must distinguish between:

- Missing optional features.
- Missing required features.
- Unavailable enrichment data.
- Invalid or corrupted values.

### EVAL-DATA-004 — Duplicate Detection

Duplicate transaction identifiers must be identified.

Duplicate records must not distort metric calculations.

### EVAL-DATA-005 — Label Quality

Fraud labels must be assessed for:

- Source reliability.
- Completeness.
- Delayed availability.
- Potential correction.
- Ambiguity.
- Class imbalance.

### EVAL-DATA-006 — Data Leakage

Evaluation must check for:

- Future information in model features.
- Training and evaluation overlap.
- Target leakage.
- Duplicate entity leakage.
- Cross-tenant leakage.
- Post-decision information used as an input.

Any confirmed material leakage must block model promotion.

---

## 6. Dataset Partitioning

### 6.1 Temporal Splitting

FraudShield AI should use temporal validation where transaction chronology matters.

A typical structure is:

1. Training period.
2. Validation period.
3. Temporal holdout period.

The holdout period must occur after the training period.

The precise time windows must be documented for each evaluation.

### 6.2 Entity Leakage Prevention

Where appropriate, partitioning must account for related entities such as:

- Customer.
- Account.
- Device.
- Merchant.
- Transaction group.

The evaluation design must document whether related entities may appear across partitions and the implications for leakage.

### 6.3 Holdout Integrity

The final holdout dataset must not be repeatedly used for model tuning.

Repeated tuning against the holdout invalidates its independence.

---

## 7. Fraud Detection Metrics

### EVAL-ML-001 — Precision

Precision measures the proportion of predicted fraud cases that are actually fraudulent.

$$
Precision = \frac{TP}{TP + FP}
$$

**Provisional minimum:** 0.85

### EVAL-ML-002 — Recall

Recall measures the proportion of actual fraud cases detected by the model.

$$
Recall = \frac{TP}{TP + FN}
$$

**Provisional minimum:** 0.80

### EVAL-ML-003 — F1 Score

F1 balances precision and recall.

$$
F1 = 2 \times \frac{Precision \times Recall}{Precision + Recall}
$$

F1 must be reported, but no fixed minimum is established in the initial contract.

### EVAL-ML-004 — ROC-AUC

ROC-AUC measures discrimination across classification thresholds.

**Provisional minimum:** 0.85

### EVAL-ML-005 — PR-AUC

PR-AUC summarizes the precision-recall relationship.

**Provisional minimum:** 0.70

PR-AUC must be interpreted in relation to fraud prevalence.

### EVAL-ML-006 — False-Positive Rate

$$
FPR = \frac{FP}{FP + TN}
$$

**Provisional maximum:** 0.05

### EVAL-ML-007 — False-Negative Rate

$$
FNR = \frac{FN}{FN + TP}
$$

False-negative rate must be reported.

The initial contract does not establish a separate numerical maximum.

### EVAL-ML-008 — Metric Confidence Intervals

Evaluation must report confidence intervals for key model metrics.

The confidence interval method and confidence level must be documented.

### EVAL-ML-009 — Threshold Context

Threshold-dependent metrics must identify the classification threshold used.

Metrics calculated at one threshold must not be presented as applying to all policy thresholds.

---

## 8. Risk Policy Evaluation

The initial risk policy defines:

| Score range           | Classification |
| --------------------- | -------------- |
| `0.0 ≤ score < 0.30`  | APPROVE        |
| `0.30 ≤ score < 0.70` | REVIEW         |
| `0.70 ≤ score ≤ 1.00` | DECLINE        |

These thresholds are provisional.

### EVAL-POL-001 — Boundary Testing

The evaluation must test exact threshold boundaries.

Required cases include:

| Score | Expected result |
| ----- | --------------- |
| 0.00  | APPROVE         |
| 0.29  | APPROVE         |
| 0.30  | REVIEW          |
| 0.69  | REVIEW          |
| 0.70  | DECLINE         |
| 1.00  | DECLINE         |

### EVAL-POL-002 — Invalid Scores

The policy must safely handle:

- Negative scores.
- Scores greater than 1.0.
- Missing scores.
- Non-numeric scores.
- NaN.
- Positive infinity.
- Negative infinity.

Invalid inference scores must not result in APPROVE.

### EVAL-POL-003 — Threshold Calibration

Threshold calibration must consider:

- Fraud prevalence.
- Cost of missed fraud.
- Cost of false positives.
- Human review capacity.
- Tenant-specific requirements.
- Customer experience.
- Operational latency.

### EVAL-POL-004 — Policy Regression

A policy change must be evaluated against the previously approved version.

The evaluation must report:

- Decision distribution changes.
- Approval rate changes.
- Review rate changes.
- Decline rate changes.
- Estimated fraud capture changes.
- False-positive impact.
- Operational review workload.

---

## 9. Cohort Evaluation

Cohort evaluation is mandatory.

The initial contract identifies these dimensions:

- Tenant.
- Transaction amount band.
- Transaction channel.
- Geographic region.
- Customer segment.

### EVAL-COHORT-001 — Cohort Metrics

For each sufficiently populated cohort, report:

- Sample size.
- Fraud prevalence.
- Precision.
- Recall.
- False-positive rate.
- False-negative rate.
- Decision distribution.

### EVAL-COHORT-002 — Minimum Sample Size

A minimum sample size must be defined for each reported metric.

Where sample size is insufficient, the result must be marked as statistically inconclusive.

### EVAL-COHORT-003 — Cohort Regression

Material degradation in an important cohort must trigger review.

An acceptable aggregate score must not automatically override a serious cohort regression.

### EVAL-COHORT-004 — Sensitive Attributes

Cohort analysis involving sensitive or legally protected attributes requires appropriate legal, privacy, and governance review.

---

## 10. Calibration Evaluation

A fraud model's risk score must be evaluated for calibration.

Calibration assessment should include:

- Reliability diagrams.
- Calibration error.
- Score distribution.
- Observed fraud rate by score band.
- Stability over time.

A score of 0.70 must not be described as a verified 70% fraud probability unless calibration evidence supports that interpretation.

Calibration results must be documented before production threshold approval.

---

## 11. Drift Evaluation

### EVAL-DRIFT-001 — Feature Drift

Monitor changes in relevant feature distributions.

### EVAL-DRIFT-002 — Prediction Drift

Monitor changes in risk score distributions and decision rates.

### EVAL-DRIFT-003 — Performance Drift

Evaluate fraud detection effectiveness as labels become available.

### EVAL-DRIFT-004 — Delayed Labels

The monitoring system must distinguish between:

- Unlabeled transactions.
- Provisional labels.
- Confirmed labels.
- Corrected labels.

### EVAL-DRIFT-005 — Drift Response

Material drift must trigger investigation.

Drift alone must not automatically authorize an unreviewed model deployment.

---

## 12. Performance Evaluation

The following provisional latency gates are defined in the machine-readable evaluation contract.

| Metric                         | P95 target |
| ------------------------------ | ---------- |
| API acceptance latency         | ≤ 200 ms   |
| Transaction validation latency | ≤ 100 ms   |
| Model inference latency        | ≤ 300 ms   |
| End-to-end decision latency    | ≤ 1,000 ms |

### EVAL-PERF-001 — Test Conditions

Performance evidence must identify:

- Environment.
- Test duration.
- Concurrency.
- Transaction rate.
- Payload characteristics.
- Tenant distribution.
- Dependency configuration.
- Model version.
- Policy version.

### EVAL-PERF-002 — Percentile Reporting

Report P50, P95, and P99 latency.

### EVAL-PERF-003 — End-to-End Measurement

End-to-end decision latency must include asynchronous queue waiting time where applicable.

### EVAL-PERF-004 — Error Reporting

Performance results must include error rates and incomplete decisions.

Latency results must not exclude failed requests without explicitly reporting the exclusion.

---

## 13. Reliability Evaluation

Required scenarios include:

| Test ID | Scenario                         | Expected result                   |
| ------- | -------------------------------- | --------------------------------- |
| REL-001 | Duplicate transaction submission | No duplicate consequential action |
| REL-002 | Event redelivery                 | Idempotent processing             |
| REL-003 | Transient dependency failure     | Bounded retry                     |
| REL-004 | Persistent event failure         | Controlled dead-letter handling   |
| REL-005 | Authorized event replay          | Security controls remain enforced |
| REL-006 | Model inference outage           | ABSTAIN                           |
| REL-007 | Missing required features        | ABSTAIN                           |
| REL-008 | Policy engine outage             | ABSTAIN                           |
| REL-009 | Audit persistence outage         | No completed unaudited decision   |
| REL-010 | Approval service outage          | Approval-gated operation blocked  |

All required reliability tests must pass before production release unless an approved exception process explicitly applies.

---

## 14. Security Evaluation

The release must verify the following security properties.

| Test ID | Scenario                          | Expected result                               |
| ------- | --------------------------------- | --------------------------------------------- |
| SEC-001 | Unauthenticated transaction       | Rejected                                      |
| SEC-002 | Unauthorized transaction          | Rejected                                      |
| SEC-003 | Client-controlled tenant mismatch | Rejected                                      |
| SEC-004 | Cross-tenant transaction access   | Denied                                        |
| SEC-005 | Cross-tenant feature retrieval    | Denied                                        |
| SEC-006 | Model-generated privileged action | Denied                                        |
| SEC-007 | Unauthorized reviewer             | Denied                                        |
| SEC-008 | Reused approval evidence          | Denied                                        |
| SEC-009 | Expired approval evidence         | Denied                                        |
| SEC-010 | Missing audit persistence         | No completed decision or consequential action |
| SEC-011 | Unauthorized policy modification  | Denied                                        |
| SEC-012 | Missing required provenance       | Rejected or abstained                         |
| SEC-013 | Secret exposure in logs           | Test fails                                    |
| SEC-014 | Unauthorized event replay         | Denied                                        |

The initial evaluation contract permits **zero critical unresolved security findings**.

A security exception must not be used to bypass the seven foundational security invariants.

---

## 15. Audit Evaluation

Evaluation must confirm that a completed decision can be reconstructed from protected evidence.

Required checks include:

1. Transaction identifier recorded.
2. Verified tenant identifier recorded.
3. Model version recorded.
4. Policy version recorded.
5. Risk score recorded when available.
6. Decision recorded.
7. Decision timestamp recorded.
8. Reason codes recorded.
9. Input provenance available.
10. Approval evidence available when required.

Unauthorized audit modification and deletion attempts must be denied.

---

## 16. Release Evaluation Gates

The initial release requirements are:

| Gate     | Requirement                                       |
| -------- | ------------------------------------------------- |
| GATE-001 | Model version identified                          |
| GATE-002 | Policy version identified                         |
| GATE-003 | Dataset version identified                        |
| GATE-004 | Evaluation evidence available                     |
| GATE-005 | Security review completed                         |
| GATE-006 | Release manifest created                          |
| GATE-007 | Human release approval recorded                   |
| GATE-008 | Required security tests passed                    |
| GATE-009 | Required reliability tests passed                 |
| GATE-010 | Applicable model metrics meet approved thresholds |

A release must not be described as production-ready solely because the model meets aggregate accuracy thresholds.

---

## 17. Evaluation Decision Outcomes

An evaluation run must produce one of the following outcomes:

| Outcome        | Meaning                                               |
| -------------- | ----------------------------------------------------- |
| PASS           | All applicable approved gates passed                  |
| FAIL           | One or more mandatory gates failed                    |
| INCONCLUSIVE   | Evidence is insufficient for a reliable determination |
| NOT_APPLICABLE | Gate does not apply to the evaluated release category |

`INCONCLUSIVE` must not be treated as `PASS`.

A failed mandatory security gate must block release.

---

## 18. Evaluation Evidence Report

Each evaluation report must contain:

1. Executive summary.
2. Release candidate identifiers.
3. Dataset and label provenance.
4. Model and policy versions.
5. Evaluation methodology.
6. Metric results.
7. Cohort results.
8. Calibration findings.
9. Drift findings.
10. Performance results.
11. Reliability test results.
12. Security test results.
13. Audit verification results.
14. Known limitations.
15. Gate outcomes.
16. Reviewer and approval evidence.

---

## 19. P0 Acceptance Criteria

This specification is ready for P0 baseline approval when:

1. Evaluation scope is defined.
2. Required datasets and provenance are documented.
3. Model metrics are specified.
4. Temporal validation is required.
5. Data leakage checks are included.
6. Cohort evaluation is mandatory.
7. Calibration requirements are documented.
8. Drift and delayed-label handling are defined.
9. Performance verification is specified.
10. Reliability and security tests are defined.
11. Release gate outcomes are unambiguous.
12. Requirements align with the machine-readable evaluation contract.

---

## 20. Related Documents

- `docs/architecture/problem-statement.md`
- `docs/architecture/system-context.md`
- `docs/architecture/container-architecture.md`
- `docs/architecture/decision-workflow.md`
- `docs/security/data-classification.md`
- `docs/security/threat-model.md`
- `docs/security/risk-policy.md`
- `docs/quality/non-functional-requirements.md`
- `contracts/risk-policy.v1.json`
- `contracts/evaluation-gates.v1.json`
- `contracts/release-manifest.template.yaml`

---

## 21. Approval Status

**Status: Proposed — P0 Architecture Baseline**

The numerical evaluation thresholds and latency targets are provisional.

Production promotion requires representative evaluation evidence, calibrated thresholds, security review, and authorized release approval.
