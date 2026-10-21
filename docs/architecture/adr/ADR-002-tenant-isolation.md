# ADR-002: Enforce Tenant Isolation Across All Processing Layers

**Status:** Proposed
**Date:** 2026-10-21
**Decision Makers:** FraudShield AI Architecture and Security
**Scope:** Identity, APIs, events, features, persistence, and audit

## 1. Context

FraudShield AI is designed to support multiple organizations or tenants.

Each tenant may submit transactions, maintain fraud features, review decisions, and access audit evidence.

A failure to isolate tenant data could expose sensitive financial information or permit unauthorized actions.

Tenant isolation must therefore be enforced consistently across the platform.

## 2. Decision

FraudShield AI will implement **verified tenant-context isolation at every relevant trust boundary**.

Tenant identity must originate from authenticated and authorized security context.

Client-provided tenant identifiers must never independently establish authorization.

Every tenant-scoped operation must verify that the requesting principal is authorized for the target tenant and resource.

## 3. Identity Model

The platform must:

1. Authenticate the requesting principal.
2. Validate the token or credential.
3. Derive tenant identity from verified identity claims or an authorized tenant mapping.
4. Establish an immutable request-scoped tenant context.
5. Authorize access to the requested resource.
6. Propagate trusted tenant context to downstream services.

A request body containing a tenant identifier must not override the verified tenant context.

If a client-provided tenant identifier conflicts with verified identity, the request must be rejected.

## 4. API Isolation

API endpoints must enforce tenant-scoped authorization.

Requirements include:

- No unauthenticated trusted processing.
- No direct trust in client-controlled tenant fields.
- Tenant-scoped resource access.
- Safe error responses.
- Authorization checks for administrative operations.
- Tenant-aware rate limiting where appropriate.

## 5. Event Isolation

Transaction events must include tenant provenance.

Consumers must verify that the event was produced through an authorized ingestion path.

An event's tenant field must not be considered sufficient proof of tenant authorization.

Required controls include:

- Authorized event producers.
- Versioned event schemas.
- Trusted tenant provenance.
- Consumer-side validation.
- Tenant-scoped idempotency.
- Rejection of inconsistent tenant context.

## 6. Database Isolation

All tenant-scoped database records must include a validated tenant identifier.

The proposed initial implementation uses a shared PostgreSQL database with logical tenant isolation.

Controls must include:

- Mandatory tenant predicates.
- Tenant-aware repositories.
- Least-privilege database access.
- Row-level security where feasible.
- Automated cross-tenant access tests.
- Restricted administrative access.

Application-layer filtering alone must not be the sole long-term isolation control for sensitive production records.

Database row-level security and role configuration must be evaluated during implementation.

## 7. Feature Isolation

Fraud feature enrichment must not retrieve data from another tenant without explicit authorization.

Feature storage and retrieval must:

1. Use verified tenant context.
2. Apply tenant-scoped keys.
3. Prevent cross-tenant cache collisions.
4. Validate feature provenance.
5. Restrict shared aggregate features.
6. Record feature transformation versions.

Any cross-tenant intelligence sharing requires a separately approved governance model.

## 8. Model Isolation

Model inference must not introduce unauthorized cross-tenant information disclosure.

Requirements include:

- Tenant-aware feature retrieval.
- Controlled training data access.
- Dataset provenance.
- Evaluation of cross-tenant leakage.
- Restricted model artifact access.
- Tenant-aware monitoring.

Use of shared models must be documented and reviewed for privacy, confidentiality, and contractual requirements.

## 9. Audit Isolation

Audit records must be associated with the verified tenant.

Tenant users must not access another tenant's audit evidence.

Administrative cross-tenant investigation must require explicit authorization and be auditable.

## 10. Alternatives Considered

### Alternative A: Separate Infrastructure Per Tenant

Each tenant receives dedicated application, messaging, and storage resources.

**Advantages:**

- Strong physical or resource-level separation.
- Independent scaling.
- Reduced shared-resource exposure.

**Disadvantages:**

- Higher operational cost.
- Greater deployment complexity.
- More difficult management at scale.

**Decision:** Not selected as the initial default.

### Alternative B: Shared Infrastructure with Tenant Isolation

Tenants share infrastructure while identity, authorization, storage, and processing controls enforce isolation.

**Advantages:**

- Lower initial operating cost.
- Simplified centralized operations.
- More efficient resource utilization.

**Disadvantages:**

- Greater dependence on correct authorization.
- Higher risk from tenant-filtering defects.
- Need for rigorous automated isolation testing.

**Decision:** Selected as the initial architecture.

### Alternative C: Shared Application with Dedicated Tenant Databases

The application layer is shared, but tenant persistence is separated.

**Advantages:**

- Stronger data separation.
- Tenant-specific backup and retention options.

**Disadvantages:**

- More database management overhead.
- More complex tenant provisioning.
- Higher operating cost.

**Decision:** Retained as a future option for high-assurance tenants.

## 11. Consequences

### Positive

- Consistent tenant security model.
- Shared infrastructure efficiency.
- Explicit tenant authorization requirements.
- Clear testability of isolation controls.

### Negative

- Additional authorization complexity.
- Greater need for secure query design.
- Tenant-aware caching and event processing requirements.
- More extensive security regression testing.

## 12. Verification

Required tests include:

| Test                                            | Expected result              |
| ----------------------------------------------- | ---------------------------- |
| Unauthenticated transaction submission          | Rejected                     |
| Client tenant ID conflicts with verified tenant | Rejected                     |
| Tenant A reads Tenant B transaction             | Denied                       |
| Tenant A retrieves Tenant B features            | Denied                       |
| Tenant A reads Tenant B audit evidence          | Denied                       |
| Tenant-scoped idempotency collision             | No cross-tenant interference |
| Unauthorized administrative access              | Denied                       |
| Event with inconsistent tenant provenance       | Rejected                     |

## 13. Related Documents

- `docs/security/data-classification.md`
- `docs/security/threat-model.md`
- `docs/security/risk-policy.md`
- `docs/architecture/container-architecture.md`
- `docs/quality/evaluation-specification.md`

## 14. Decision Status

**Proposed — P0 Architecture Baseline**

Acceptance requires security review of the identity model, database isolation design, and cross-tenant verification strategy.
