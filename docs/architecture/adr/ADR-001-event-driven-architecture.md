# ADR-001: Adopt an Event-Driven Fraud Processing Architecture

**Status:** Proposed
**Date:** 2026-10-21
**Decision Makers:** FraudShield AI Architecture and Engineering
**Scope:** Transaction ingestion, processing, and fraud decisioning

## 1. Context

FraudShield AI must process financial transaction events, enrich them with fraud features, execute machine-learning inference, and apply deterministic risk policies.

The platform must support variable transaction volumes, asynchronous processing, controlled retries, failure recovery, and auditable decision outcomes.

A tightly coupled synchronous architecture would require the transaction API to remain connected to every downstream processing component.

This creates several challenges:

- Downstream failures can directly affect API availability.
- Traffic bursts can overload inference services.
- Retrying transactions becomes difficult.
- Processing components cannot scale independently.
- Event replay and forensic reconstruction become harder.

An event-driven architecture provides a mechanism to decouple transaction ingestion from fraud processing.

## 2. Decision

FraudShield AI will use an **event-driven processing architecture** built around Azure Event Hubs.

The proposed processing flow is:

1. An external client submits a transaction through Azure API Management.
2. The ASP.NET Core API authenticates and authorizes the request.
3. The API derives the verified tenant context.
4. The API validates the transaction submission.
5. The system assigns or validates a stable transaction identifier.
6. The transaction is durably accepted for asynchronous processing.
7. An event is published to Azure Event Hubs.
8. An authorized processing service consumes the event.
9. The event schema and provenance are validated.
10. Tenant-scoped fraud features are retrieved.
11. Azure Machine Learning produces a risk score.
12. A deterministic policy engine produces the risk classification.
13. Decision evidence is persisted.
14. The completed decision becomes available to authorized consumers.
15. Consequential operations pass through a separate authorization boundary.

The API must distinguish **accepted for processing** from **decision completed**.

An HTTP `202 Accepted` response may indicate successful asynchronous submission, but it must not imply that a fraud decision has already been produced.

## 3. Reliability Model

The architecture assumes that events may be delivered or processed more than once.

FraudShield AI must therefore implement:

- Tenant-scoped idempotency.
- Stable event identifiers.
- Duplicate detection.
- Bounded retries.
- Exponential backoff where appropriate.
- Poison-message handling.
- Controlled dead-letter storage or equivalent failure handling.
- Authorized replay.
- Durable decision evidence.

Azure Event Hubs must not be assumed to provide exactly-once business processing.

Exactly-once consequential effects must be enforced through application-level controls.

## 4. Ordering

Event ordering must not be assumed across all partitions.

Where ordering is required, the partitioning strategy and processing rules must be documented.

Partition keys must not expose unnecessary sensitive information.

## 5. Security Requirements

All processing stages must preserve:

- Verified tenant identity.
- Event provenance.
- Schema version.
- Correlation identifier.
- Authorization context where applicable.
- Model version.
- Policy version.

Event payload fields must not be treated as trusted merely because they originated from an internal event stream.

Consumers must validate the event envelope and applicable authorization context.

## 6. Alternatives Considered

### Alternative A: Fully Synchronous API Processing

The API directly invokes validation, enrichment, inference, and policy evaluation before responding.

**Advantages:**

- Simpler request-response semantics.
- Immediate decision delivery.
- Fewer messaging components.

**Disadvantages:**

- Tight coupling to downstream availability.
- Greater exposure to inference latency.
- More difficult burst handling.
- Reduced resilience to transient failures.

**Decision:** Not selected as the primary processing model.

### Alternative B: Queue-Based Processing

Use Azure Service Bus queues as the primary ingestion backbone.

**Advantages:**

- Native message settlement and dead-letter queues.
- Strong support for work-queue patterns.
- Useful for command-oriented workflows.

**Disadvantages:**

- Different scaling and consumption characteristics from event streaming.
- Less aligned with the proposed high-volume transaction event stream.

**Decision:** Not selected as the primary ingestion backbone. Azure Service Bus remains an option for approval commands and workflow tasks.

### Alternative C: Event-Driven Processing with Azure Event Hubs

**Advantages:**

- Decoupled producers and consumers.
- Partitioned event ingestion.
- Independent processing scale.
- Support for event retention and controlled replay.
- Suitable for transaction event streams.

**Disadvantages:**

- Additional operational complexity.
- Eventual consistency.
- Application-managed idempotency.
- Separate dead-letter handling design required.
- More complex end-to-end tracing.

**Decision:** Selected.

## 7. Consequences

### Positive

- Improved separation of ingestion and processing.
- Independent scaling of processing services.
- Better handling of traffic bursts.
- Support for replay and recovery.
- Clearer event provenance.

### Negative

- Increased infrastructure complexity.
- More complicated consistency management.
- Additional monitoring requirements.
- Potential queue waiting time.
- Explicit duplicate-processing protection required.

## 8. Operational Requirements

The platform must monitor:

- Event ingestion rate.
- Processing throughput.
- Consumer lag.
- Retry volume.
- Dead-letter volume.
- Processing latency.
- Decision completion latency.
- Event age.
- Dependency failures.

A persistent processing backlog must trigger operational alerts.

## 9. Verification

The decision will be validated through:

1. Duplicate event tests.
2. Retry tests.
3. Processing failure tests.
4. Event replay tests.
5. Backpressure tests.
6. End-to-end latency tests.
7. Tenant isolation tests.
8. Audit persistence tests.

## 10. Related Documents

- `docs/architecture/system-context.md`
- `docs/architecture/container-architecture.md`
- `docs/architecture/decision-workflow.md`
- `docs/quality/non-functional-requirements.md`
- `docs/security/threat-model.md`

## 11. Decision Status

**Proposed — P0 Architecture Baseline**

This decision becomes accepted when the architecture is reviewed and the associated reliability and security requirements are approved.
