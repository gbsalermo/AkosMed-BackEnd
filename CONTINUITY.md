# AkosMed Backend Continuity

**Repository:** `gbsalermo/AkosMed-BackEnd`  
**Stable branch:** `main`  
**Documentation branch:** `docs/mvp-blueprint`  
**Project language:** English  
**Current phase:** planning / API blueprint  
**Current stage:** Stage 0 — Product and API Blueprint  
**Current substage:** 0.1 — MVP capability map  

This file is the first checkpoint to read when resuming the project.

---

# 1. Current reality

The backend implementation has not started yet.

Current work is intentionally focused on defining the real API/product scope before H2, Spring entities or CRUD implementation.

The blueprint currently covers:

```text
SaaS tenant/unit organization
identity/access
patients/professionals
specialties/procedures
calendar/scheduling
appointments
medical record
encounters
clinical notes
clinical measurements
longitudinal care plans
follow-up history
patient instructions/materials
exam orders/results
prescriptions
clinical documents
private file storage
notifications
operational /me APIs
audit
```

---

# 2. Current documents

Canonical documentation is being reduced to:

```text
CONTINUITY.md
docs/README.md
docs/MASTER_DOSSIER.md
docs/ROADMAP.md
docs/MVP_SCOPE.md
docs/DOMAIN_MODEL.md
docs/API_CONTRACT.md
docs/ENGINEERING_RULES.md
```

Old duplicated planning files are removed from the active documentation set; Git history remains the archive.

---

# 3. Decisions currently accepted

## Language

All code/domain/API naming is English.

## Architecture

```text
Java 21
Spring Boot
Maven
REST
Spring Data JPA / Hibernate
modular monolith by feature
```

## API identifiers

```text
Long id       → internal PK/FK/locks
UUID publicId → public API/DTO/URL/JWT/integrations
```

## Multi-tenancy

```text
Shared Database
+ Shared Schema
+ tenant_id
```

Tenant is resolved from authenticated server context.

Unit is an operational sub-boundary inside a Tenant and must be explicitly authorized.

## DTOs

Separate request/response DTOs from the first CRUD.

## Git workflow

```text
small branch
→ implementation/review
→ tests/validation
→ Pull Request
→ main
```

No new backend functionality is developed directly on `main`.

## Development ownership

Backend functional code is implemented manually by the project owner unless direct implementation is explicitly requested.

AI support is used for:

- planning;
- architecture;
- explanations;
- reference snippets;
- review;
- debugging guidance.

## Database

Early implementation uses H2/JPA for fast domain evolution.

PostgreSQL enters only after the core is stable.

A controlled migration strategy is mandatory before production PostgreSQL. Flyway is the recommended default, but it is not required during early H2 development.

## Swagger/Postman

Because the project is API-first and no frontend is required during backend development:

```text
Swagger/OpenAPI grows with endpoints
Postman grows with endpoints
```

Final stages consolidate them as the official HTTP contract and end-to-end validation suite.

---

# 4. Current MVP model direction

Main domain groups:

```text
Organization
Identity
Clinical Directory
Scheduling
Clinical Record
Longitudinal Care
Diagnostics
Prescription/Documents
Storage
Notifications/Audit
```

The core remains specialty-neutral.

Nutrition, physiotherapy and similar realities are supported through generic concepts such as:

```text
Encounter
ClinicalMeasurement
CarePlan
CarePlanEntry
PatientInstruction
```

Specialty-specific entities are deferred until the generic core is insufficient for a concrete workflow.

---

# 5. Current next task

## Stage 0.1 — MVP capability map review

Before any code or H2 configuration:

1. review `docs/MVP_SCOPE.md`;
2. confirm missing/extra clinic use cases;
3. confirm which capabilities belong to the MVP;
4. confirm out-of-scope items;
5. confirm tenant vs unit semantics;
6. confirm follow-up/care-plan expectations;
7. confirm exam/result scope;
8. confirm document types;
9. only then mark 0.1 complete.

After 0.1:

```text
0.2 → workflows/business rules
0.3 → domain model review
0.4 → API contract review
0.5 → blueprint freeze
1.1 → Spring Initializr project
1.2 → H2/profiles
```

Do not start H2 before Stage 0 is accepted.

---

# 6. Resume protocol

When resuming:

```text
1. read CONTINUITY.md
2. inspect main/current branch
3. read docs/MASTER_DOSSIER.md
4. open current substage in docs/ROADMAP.md
5. read the relevant canonical technical document
6. work only on that substage
7. validate
8. update CONTINUITY.md
9. open/review PR
10. merge before advancing
```

Do not infer progress from planning documents alone.