# Policy and Claims API — Requirements Specification Document

**Author:** Luke Holder

---

## 1. Overview

### 1.1 Purpose

This document defines the requirements for the Policy & Claims API, a multi-tenant backend service that enables insurance agencies to manage their policies and process claims filed against those policies. It serves as the authoritative reference for what the system must do and the constraints it operates under, and is the basis for the subsequent design document.

### 1.2 Problem Statement

Insurance agencies need a secure, reliable system to track policies and manage the claims lifecycle. A single service that supports multiple agencies must guarantee that each agency's data remains isolated from every other agency's, enforce role-appropriate access for different staff, and provide a consistent, auditable workflow as claims move from submission to resolution. This API addresses those needs as a cloud-native service.

### 1.3 Goals

- Provide CRUD operations for policies and claims within an agency's own data boundary.
- Enforce strict tenant isolation so no agency can access another agency's data.
- Support role-based access for distinct user types (e.g., agency admin, adjuster, read-only).
- Model a claims workflow with well-defined states and permitted transitions.
- Secure all secrets and credentials using managed cloud identity, with no sensitive values stored in application configuration.
- Provide per-tenant observability and request throttling.

### 1.4 Non-Goals

- This system is not a customer-facing portal; it exposes an API, not a UI.
- It does not handle payment processing, premium billing, or financial settlement of claims.
- It does not perform underwriting, risk scoring, or premium calculation.
- It is not intended to satisfy real-world regulatory compliance (e.g., HIPAA, state insurance regulations); such concerns are noted where relevant but are out of scope for this project.

### 1.5 System Summary

The Policy & Claims API is a RESTful service built on ASP.NET Core, backed by Azure SQL, and secured through Microsoft Entra ID. Multiple insurance agencies (tenants) share the deployment while their data stays logically isolated. Authenticated users act within the scope of their agency and their assigned role. The service is designed for cloud deployment on Azure, using managed identity for secret access and Application Insights for telemetry.

---

## 2. Scope

### 2.1 In Scope

**Tenant Management**

- Representing multiple agencies as isolated tenants within a shared deployment.
- Resolving the active tenant for every request and binding all data access to that tenant.

**Authentication and Authorization**

- Authenticating users via Microsoft Entra ID–issued tokens.
- Enforcing role-based access (agency admin, adjuster, read-only) on protected operations.
- Deriving the user's tenant and role from token claims.

**Policy Management**

- Creating, reading, updating, and deactivating policies within a tenant.
- Associating policies with policyholders.

**Claim Management**

- Filing claims against existing policies.
- Reading and updating claims within a tenant.
- Advancing claims through a defined workflow of states and permitted transitions.

**Cross-Cutting Platform Capabilities**

- Tenant-level data isolation enforced at the data-access layer.
- Secret and credential management via Key Vault and managed identity.
- Per-tenant request telemetry and rate limiting.

### 2.2 Out of Scope

- Any user interface or customer-facing portal; the deliverable is an API only.
- Payment processing, premium billing, and financial settlement of claims.
- Underwriting, risk assessment, actuarial modeling, and premium calculation.
- Document generation (policy PDFs, claim letters) and file storage.
- Notifications (email, SMS) to policyholders or staff. (Note: an async notification path appears as a stretch goal in the build plan; it is out of scope for the core requirements.)
- Self-service tenant onboarding and sign-up; tenants are provisioned administratively.
- Full regulatory compliance certification (HIPAA, state insurance regulations).
- Integration with external carrier systems, rating engines, or third-party claims services.

---

## 3. Stakeholders and Actors

### 3.1 Actors

| Actor | Description | Key Interactions |
|---|---|---|
| **Agency Admin** | Manages an agency's users and its policy/claim data. Highest-privilege role within a tenant. | Full CRUD on policies and claims; manage users/roles within the tenant; view all tenant data. |
| **Adjuster** | Processes claims filed against the agency's policies. | Read policies; create and update claims; advance claims through workflow states. |
| **Read-Only User** | Views agency data without modifying it (e.g., auditor, support staff). | Read policies and claims within the tenant; no write access. |
| **System Administrator** | Operates the platform itself; provisions tenants administratively. Acts outside any single tenant's boundary. | Create/configure tenants; not a consumer of policy/claim business operations. |
| **Client Application** | Any authenticated service or front-end that consumes the API on a user's behalf. | Sends authenticated requests; carries the user's tenant and role via token claims. |

### 3.2 Role and Permission Summary

| Capability | Agency Admin | Adjuster | Read-Only |
|---|:---:|:---:|:---:|
| View policies and claims | ✓ | ✓ | ✓ |
| Create/update policies | ✓ | ✗ | ✗ |
| Deactivate policies | ✓ | ✗ | ✗ |
| File claims | ✓ | ✓ | ✗ |
| Update/advance claims | ✓ | ✓ | ✗ |
| Manage tenant users and roles | ✓ | ✗ | ✗ |

### 3.3 Non-User Stakeholders

These parties have an interest in the system but do not interact with the API directly.

- **Insurance Agencies (Tenants)** — the organizations whose data the system holds; primary beneficiaries of correct isolation and reliability.
- **Policyholders** — the customers whose policies and claims are recorded. They do not access the system directly (no customer portal is in scope) but are the ultimate subjects of the data.
- **Project Owner / Developer** — responsible for delivery; interested in the architecture serving as a demonstrable engineering artifact.

---

## 4. Domain Glossary

### 4.1 Insurance Domain Terms

- **Agency** — An insurance business that sells and services policies. In this system, each agency is a tenant.
- **Policyholder** — The customer who owns a policy. Belongs to exactly one agency (tenant).
- **Policy** — A contract of insurance coverage held by a policyholder, with a coverage type, premium, effective and expiration dates, and a status.
- **Coverage Type** — The category of insurance a policy provides (e.g., auto, home, life). Used here as a classifying attribute, not a full coverage-modeling system.
- **Premium** — The amount charged for a policy's coverage. Stored as a recorded value; the system does not calculate or bill it.
- **Effective Date** — The date a policy's coverage begins.
- **Expiration Date** — The date a policy's coverage ends.
- **Policy Status** — The lifecycle state of a policy (e.g., active, expired, cancelled).
- **Claim** — A request for payment or action filed against a policy when a covered event occurs.
- **Claim Amount** — The monetary value requested or assessed for a claim.
- **Claim Status** — The current state of a claim within its workflow.
- **Adjuster** — The staff member who reviews and processes claims.
- **Endorsement** — A formal modification to an existing policy. (Referenced for domain completeness; not implemented in the current scope.)

### 4.2 System and Architecture Terms

- **Tenant** — An isolated logical partition of the system belonging to a single agency. All business data is scoped to a tenant.
- **Multi-Tenancy** — The architectural approach where multiple tenants share one deployment while their data remains logically isolated.
- **Tenant Isolation** — The guarantee that one tenant cannot read or modify another tenant's data, enforced at the data-access layer.
- **Tenant Resolution** — The process of determining which tenant a request belongs to, derived from the authenticated user's token claims.
- **Role** — A named set of permissions assigned to a user (agency admin, adjuster, read-only).
- **Claim (as a token concept)** — A key–value assertion inside a security token (e.g., tenant ID, role). Distinct from an insurance claim; disambiguated in context.
- **Entra ID** — Microsoft's cloud identity provider (formerly Azure AD), used to authenticate users and issue tokens.
- **Managed Identity** — An Azure-managed credential that lets the service access other Azure resources (e.g., Key Vault) without stored secrets.
- **Key Vault** — The Azure service that securely stores secrets, connection strings, and keys.
- **Application Insights** — The Azure telemetry service used for tracing, metrics, and monitoring.

---

## 5. Functional Requirements

### 5.1 Tenant Management

- **FR-1.1** The system shall associate every business record (policy, claim, policyholder) with exactly one tenant.
- **FR-1.2** The system shall resolve the active tenant for each request from the authenticated user's token claims.
- **FR-1.3** The system shall reject any request that cannot be associated with a valid tenant.
- **FR-1.4** The system shall allow a System Administrator to provision new tenants administratively.

### 5.2 Authentication and Authorization

- **FR-2.1** The system shall require a valid Entra ID–issued token for all protected endpoints.
- **FR-2.2** The system shall reject requests with missing, expired, or invalid tokens.
- **FR-2.3** The system shall derive the user's role and tenant from the token's claims.
- **FR-2.4** The system shall enforce role-based authorization on each operation according to the permission chart in §3.2.
- **FR-2.5** The system shall deny operations for which the user's role lacks permission, returning an authorization error without revealing protected data.

### 5.3 Policyholder Management

- **FR-3.1** The system shall allow authorized users to create a policyholder within their tenant.
- **FR-3.2** The system shall allow authorized users to read policyholders within their tenant.
- **FR-3.3** The system shall allow authorized users to update a policyholder's details within their tenant.
- **FR-3.4** The system shall scope all policyholder operations to the requester's tenant.

### 5.4 Policy Management

- **FR-4.1** The system shall allow authorized users to create a policy associated with a policyholder in their tenant.
- **FR-4.2** The system shall allow authorized users to read policies within their tenant, individually and as a list.
- **FR-4.3** The system shall allow authorized users to update a policy's details within their tenant.
- **FR-4.4** The system shall allow authorized users to deactivate (soft-delete) a policy rather than permanently removing it.
- **FR-4.5** The system shall record a policy's coverage type, premium, effective date, expiration date, and status.
- **FR-4.6** The system shall prevent a policy from being associated with a policyholder in a different tenant.

### 5.5 Claim Management

- **FR-5.1** The system shall allow authorized users to file a claim against an existing policy in their tenant.
- **FR-5.2** The system shall reject a claim filed against a nonexistent policy or a policy in another tenant.
- **FR-5.3** The system shall allow authorized users to read claims within their tenant, individually and as a list.
- **FR-5.4** The system shall allow authorized users to update a claim's details within their tenant.
- **FR-5.5** The system shall record a claim's associated policy, amount, status, and relevant timestamps.

### 5.6 Claim Workflow

- **FR-6.1** The system shall model claim status as a defined set of states: Submitted, Under Review, Approved, Denied (and Closed, if adopted — see note).
- **FR-6.2** The system shall permit only valid transitions between claim states and reject invalid ones.
- **FR-6.3** The system shall restrict claim status transitions to authorized roles (agency admin, adjuster).
- **FR-6.4** The system shall record when a claim transitions between states.
- **FR-6.5** The system shall prevent modification of a claim once it reaches a terminal state (Approved, Denied, or Closed).

---

## 6. Non-Functional Requirements

### 6.1 Multi-Tenancy and Data Isolation

- **NFR-1.1** The system shall enforce tenant isolation at the data-access layer, not solely in application or controller logic, so that isolation holds even if a query omits an explicit tenant filter.
- **NFR-1.2** The system shall ensure that no query, under any role, returns data belonging to a tenant other than the requester's.
- **NFR-1.3** The system shall apply tenant scoping automatically to all reads and writes of tenant-owned entities.

### 6.2 Security

- **NFR-2.1** The system shall store no secrets, connection strings, or credentials in application configuration or source code.
- **NFR-2.2** The system shall retrieve secrets from Key Vault at runtime using a managed identity.
- **NFR-2.3** The system shall transmit all traffic over TLS/HTTPS.
- **NFR-2.4** The system shall validate all incoming request data and reject malformed input with an appropriate error.
- **NFR-2.5** The system shall not expose internal error details (stack traces, database errors) in API responses.

### 6.3 Observability

- **NFR-3.1** The system shall emit request telemetry (traces, durations, outcomes) to Application Insights.
- **NFR-3.2** The system shall tag telemetry with tenant context to enable per-tenant analysis, without logging sensitive data.
- **NFR-3.3** The system shall expose a health-check endpoint reporting the status of the service and its critical dependencies.
- **NFR-3.4** The system shall produce structured, queryable logs for significant operations and errors.

### 6.4 Performance and Throttling

- **NFR-4.1** The system shall apply rate limiting partitioned per tenant so that one tenant's traffic cannot exhaust capacity for others.
- **NFR-4.2** The system shall return a standard "too many requests" response when a tenant exceeds its rate limit.
- **NFR-4.3** The system shall respond to typical read operations within a defined latency target under normal load (target to be set during testing, e.g., p95 < 300 ms).

### 6.5 Reliability and Availability

- **NFR-5.1** The system shall handle dependency failures (e.g., database or Key Vault unavailability) gracefully, returning meaningful errors rather than crashing.
- **NFR-5.2** The system shall be stateless so that instances can be scaled horizontally without session affinity.

### 6.6 Maintainability and Quality

- **NFR-6.1** The system shall be organized into clear layers (API, domain/application, data) with separated responsibilities.
- **NFR-6.2** The system shall manage database schema changes through versioned migrations.
- **NFR-6.3** The system shall include automated tests covering tenant isolation, authorization rules, and claim-workflow transitions.
- **NFR-6.4** The system shall externalize environment-specific settings via configuration and managed identity rather than hard-coded values.

### 6.7 Portability and Deployment

- **NFR-7.1** The system shall be deployable to Azure using its managed services (Azure SQL, Key Vault, Application Insights, Entra ID).
- **NFR-7.2** The system shall support repeatable provisioning of its infrastructure (e.g., via infrastructure-as-code) as a stretch objective.

---

## 7. Data Requirements

### 7.1 Core Entities

**Tenant** — represents an insurance agency.

- Identifier
- Name
- Status (active/inactive)
- Created timestamp

**Policyholder** — a customer belonging to one tenant.

- Identifier
- Tenant association
- Name
- Contact information
- Created / last-updated timestamps

**Policy** — a coverage contract held by a policyholder.

- Identifier
- Tenant association
- Policyholder association
- Coverage type
- Premium (recorded value)
- Effective date
- Expiration date
- Status (e.g., active, expired, cancelled)
- Created / last-updated timestamps

**Claim** — a request filed against a policy.

- Identifier
- Tenant association
- Policy association
- Amount
- Status (Submitted, Under Review, Approved, Denied, [Closed])
- Filed timestamp
- Last status-change timestamp
- Created / last-updated timestamps

### 7.2 Supporting Data

- **DR-2.1** The system shall record claim status-transition history sufficient to show when a claim moved between states and by which role (supports FR-6.4 and auditability).
- **DR-2.2** The system shall associate users with a tenant and a role; user identity itself is owned by Entra ID, so the system stores only the references and role assignments it needs, not credentials.

### 7.3 Data Ownership and Tenancy

- **DR-3.1** Every business record (policyholder, policy, claim, claim history) shall carry a tenant association that binds it to exactly one tenant.
- **DR-3.2** The tenant association shall be non-nullable for all tenant-owned entities.
- **DR-3.3** Cross-tenant references shall be prohibited: a policy shall reference only a policyholder in the same tenant, and a claim shall reference only a policy in the same tenant.

### 7.4 Relationships

- **DR-4.1** A tenant has many policyholders; each policyholder belongs to one tenant.
- **DR-4.2** A policyholder has many policies; each policy belongs to one policyholder.
- **DR-4.3** A policy has many claims; each claim belongs to one policy.
- **DR-4.4** A claim has many status-history entries; each entry belongs to one claim.

### 7.5 Data Retention and Integrity

- **DR-5.1** The system shall not hard-delete policies or claims; policies are deactivated and claims lock at terminal states (consistent with §5).
- **DR-5.2** The system shall preserve claim status history for the life of the claim record.
- **DR-5.3** The system shall maintain referential integrity between related entities within a tenant.

---

## 8. Assumptions and Constraints

### 8.1 Assumptions

- **AC-1.1** Tenants (agencies) are provisioned administratively; no self-service sign-up flow is assumed.
- **AC-1.2** User identities are managed externally by Entra ID; the system trusts its issued tokens and does not manage credentials or authentication flows itself.
- **AC-1.3** Each user belongs to exactly one tenant and holds one role at a time.
- **AC-1.4** Client applications are responsible for acquiring tokens from Entra ID; the API only validates them.
- **AC-1.5** Premium values and claim amounts are supplied as recorded data; the system does not compute, rate, or validate them against actuarial rules.
- **AC-1.6** The volume of data and traffic is representative of a demonstration/portfolio context, not production-scale load.

### 8.2 Technical Constraints

- **AC-2.1** The system shall be implemented on ASP.NET Core (C#).
- **AC-2.2** The system shall use Entity Framework Core for data access.
- **AC-2.3** The system shall use Azure SQL Database as its data store.
- **AC-2.4** The system shall use Microsoft Entra ID for authentication.
- **AC-2.5** The system shall use Azure Key Vault for secret management and Managed Identity for accessing it.
- **AC-2.6** The system shall use Application Insights for telemetry.
- **AC-2.7** The system shall be deployable to Azure using its managed platform services.

### 8.3 Scope and Resource Constraints

- **AC-3.1** The project is developed and maintained by a single developer, which bounds the breadth of features that can be delivered.
- **AC-3.2** The insurance domain is intentionally simplified (policies and claims only); features such as endorsements, billing, and underwriting are excluded.
- **AC-3.3** The project targets Azure services that are available under free or low-cost tiers where feasible, to keep the demonstration affordable.
- **AC-3.4** Regulatory compliance (e.g., HIPAA, state insurance regulations) is out of scope; the design may be compliance-conscious but is not certified.

### 8.4 Dependencies

- **AC-4.1** The system depends on the availability of Azure managed services (SQL, Key Vault, Entra ID, Application Insights) at runtime.
- **AC-4.2** The system depends on Entra ID being correctly configured with the required app registrations, roles, and token claims.

---

## 9. Acceptance Criteria

Each criterion below is verifiable and traces to the requirements it satisfies. The system is considered complete when all criteria in a given phase pass.

### 9.1 Phase 1: API and Data Foundation

- **AC-P1.1** Policies and claims can be created, read, and updated through the API. *(FR-4, FR-5)*
- **AC-P1.2** Data persists to Azure SQL and schema changes are applied via versioned migrations. *(AC-2.3, NFR-6.2)*
- **AC-P1.3** The solution is organized into distinct API, application/domain, and data layers. *(NFR-6.1)*
- **AC-P1.4** Malformed requests are rejected with meaningful validation errors. *(NFR-2.4)*

### 9.2 Phase 2: Multi-Tenancy

- **AC-P2.1** Every business record is stored with a non-nullable tenant association. *(DR-3.1, DR-3.2)*
- **AC-P2.2** A request scoped to one tenant cannot read or modify another tenant's data, verified by an automated test. *(NFR-1.1, NFR-1.2, NFR-6.3)*
- **AC-P2.3** Tenant scoping is applied automatically at the data-access layer, not only in controllers. *(NFR-1.1, NFR-1.3)*
- **AC-P2.4** A claim cannot be filed against a policy in another tenant. *(FR-5.2, DR-3.3)*

### 9.3 Phase 3: Identity and Authorization

- **AC-P3.1** Protected endpoints reject requests lacking a valid Entra ID token. *(FR-2.1, FR-2.2)*
- **AC-P3.2** The user's tenant and role are derived from token claims. *(FR-2.3, FR-1.2)*
- **AC-P3.3** Role-based permissions match the matrix in §3.2, verified by automated tests covering each role. *(FR-2.4, FR-2.5, NFR-6.3)*
- **AC-P3.4** A read-only user is denied all write operations; an adjuster is denied policy creation. *(FR-2.4)*

### 9.4 Phase 4: Secrets and Managed Identity

- **AC-P4.1** No secrets, connection strings, or credentials appear in source or configuration files. *(NFR-2.1)*
- **AC-P4.2** The running service retrieves secrets from Key Vault via managed identity. *(NFR-2.2, AC-2.5)*
- **AC-P4.3** All traffic is served over HTTPS. *(NFR-2.3)*

### 9.5 Phase 5: Observability and Throttling

- **AC-P5.1** Request telemetry is visible in Application Insights, tagged with tenant context. *(NFR-3.1, NFR-3.2)*
- **AC-P5.2** A health-check endpoint reports service and dependency status. *(NFR-3.3)*
- **AC-P5.3** Exceeding a tenant's rate limit returns a standard "too many requests" response, verified by test. *(NFR-4.1, NFR-4.2)*
- **AC-P5.4** Internal error details are not exposed in API responses. *(NFR-2.5)*

### 9.6 Claim Workflow (spans phases)

- **AC-W.1** A claim moves through its defined states, and invalid transitions are rejected. *(FR-6.1, FR-6.2)*
- **AC-W.2** Only authorized roles can transition a claim's status. *(FR-6.3)*
- **AC-W.3** Each status change is recorded with its timestamp and the acting role. *(FR-6.4, DR-2.1)*
- **AC-W.4** A claim in a terminal state cannot be further modified. *(FR-6.5, DR-5.1)*
- **AC-W.5** Claim-workflow transitions are covered by automated tests. *(NFR-6.3)*

### 9.7 Phase 6: Async and IaC

- **AC-P6.1** A submitted claim publishes an event consumed by a background processor. *(stretch; build plan)*
- **AC-P6.2** The full Azure infrastructure can be provisioned repeatably via infrastructure-as-code. *(NFR-7.2)*