# Policy and Claims API — Learning Syllabus

**Learner:** Luke Holder
**Companion docs:** `requirements.md` (RSD), `design.md` (design document)

---

## Purpose

This syllabus turns the Policy & Claims API project into a guided course. The goal is not just to produce a working API but to *understand* every concept behind it well enough to explain it in an interview. The project is built as a learning vehicle, so the process matters as much as the output.

## Teaching Approach

- **Balanced concept + code.** Each lesson teaches the concept and why it matters *in this project* before any implementation.
- **Step-by-step, built together.** The learner writes the code; the instructor coaches, reviews, and explains. Skeletons and direction are welcome; wholesale generated code is not.
- **Fundamentals re-taught.** Assume the learner is rusty on .NET web development and EF Core. Rebuild the vocabulary and foundations rather than assuming them.
- **Checkpoints map to acceptance criteria.** Every unit ends at one or more acceptance criteria from the RSD (§9), so "done" is objective and testable.

## Learner's Working Rules

These are the learner's own standing rules for building skill rather than dependence. The instructor should uphold them:

1. **Struggle first.** Attempt the problem before asking for the answer.
2. **Retype, never copy-paste.** Typing code by hand is part of the learning.
3. **AI is for review and explanation, not output.** Use guidance, review, and Socratic questioning; avoid having whole solutions generated.

## How Each Lesson Runs

1. **Concept** — the idea, taught plainly, with the *why-here* tied to the design doc.
2. **Guided implementation** — the learner writes it, with coaching.
3. **Checkpoint** — verify against the mapped acceptance criterion.

---

## Curriculum

### Unit 0 — Foundations & Setup
- **0.1** The .NET landscape: SDK, runtime, CLI, projects, solutions
- **0.2** Layered architecture: API / Application / Data, and "dependencies point inward"
- **0.3** Scaffolding the solution: create the projects, wire references, first run
- **0.4** Dependency injection & the ASP.NET Core request pipeline

### Unit 1 — Data Foundation *(Build Phase 1)*
- **1.1** EF Core fundamentals: DbContext, entities, the change tracker
- **1.2** Modeling the first entities: Tenant, Policyholder, Policy
- **1.3** Migrations & Azure SQL: from C# classes to real tables
- **1.4** Entities vs. DTOs: why entities are never exposed directly
- **1.5** First endpoints: Policy CRUD, controllers, model binding, validation
- ✅ **Checkpoint:** AC-P1.1 – AC-P1.4

### Unit 2 — Multi-Tenancy *(Build Phase 2 — the heart of the project)*
- **2.1** The isolation problem & the three models; why shared-schema
- **2.2** The tenant context: request-scoped services in DI
- **2.3** Global query filters: automatic read isolation
- **2.4** Write stamping: closing the insert-side gap
- **2.5** Cross-tenant reference prevention
- ✅ **Checkpoint:** AC-P2.1 – AC-P2.4

### Unit 3 — Identity & Authorization *(Build Phase 3)*
- **3.1** OAuth2 / OIDC / JWT fundamentals: what a token *is*
- **3.2** Wiring Entra ID token validation
- **3.3** Reading claims: deriving tenant and role
- **3.4** Role-based authorization policies
- ✅ **Checkpoint:** AC-P3.1 – AC-P3.4

### Unit 4 — Secrets & Managed Identity *(Build Phase 4)*
- **4.1** Why secrets-in-config is dangerous; the managed-identity idea
- **4.2** Key Vault + `DefaultAzureCredential` (works locally and in Azure)
- ✅ **Checkpoint:** AC-P4.1 – AC-P4.3

### Unit 5 — Observability & Throttling *(Build Phase 5)*
- **5.1** Application Insights & structured logging
- **5.2** Tenant-tagged telemetry
- **5.3** Per-tenant rate limiting
- **5.4** Health checks
- ✅ **Checkpoint:** AC-P5.1 – AC-P5.4

### Unit 6 — Claim Workflow Deep-Dive *(threads through; focused here)*
- **6.1** Modeling a state machine in C#
- **6.2** Enforcing transitions & terminal locking
- **6.3** Atomic status-plus-history writes
- ✅ **Checkpoint:** AC-W.1 – AC-W.5

### Unit 7 — Stretch: Async & IaC *(Build Phase 6)*
- **7.1** Azure Service Bus & the outbox idea
- **7.2** Infrastructure as code with Bicep
- ✅ **Checkpoint:** AC-P6.1 – AC-P6.2

---

## Prerequisites & Environment

- **.NET SDK** (current version, e.g. 8.x or 9.x) — verify with `dotnet --version`.
- **An IDE/editor** — VS Code, Visual Studio, or Rider.
- **A local SQL option** for early lessons (LocalDB, SQL Server container, or similar); **Azure SQL** and an **Azure account** are needed from Unit 1.3 onward, though early data work can target a local database first.
- **Entra ID, Key Vault, Application Insights** — introduced in their respective units; no account setup needed until then.

## Progress Tracking

Each unit's checkpoint corresponds to acceptance criteria in `requirements.md` §9. Mark a unit complete only when its checkpoint criteria pass, ideally backed by the automated tests described in `design.md` §11.
