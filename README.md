# AkosMed Backend

AkosMed is a backend-first multi-tenant SaaS platform for outpatient clinics, private practices and multidisciplinary healthcare organizations.

## Current status

```text
Implementation: not started
Current phase: planning / API blueprint
Current stage: Stage 0 — Product and API Blueprint
Current substage: 0.1 — MVP capability map
```

Do not start H2 or domain implementation before the Stage 0 blueprint is reviewed and accepted.

## Documentation order

Read:

```text
1. CONTINUITY.md
2. docs/MASTER_DOSSIER.md
3. docs/ROADMAP.md
4. docs/MVP_SCOPE.md
5. docs/DOMAIN_MODEL.md
6. docs/API_CONTRACT.md
7. docs/ENGINEERING_RULES.md
```

Documentation index: `docs/README.md`.

## Core technical direction

```text
Java 21
Spring Boot
Maven
REST
Spring Data JPA / Hibernate
modular monolith by feature
H2 during early domain development
PostgreSQL after core stabilization
Swagger/OpenAPI incrementally with endpoints
Postman incrementally with endpoints
```

## Identifiers

```text
Long id       → internal PK/FK/locks
UUID publicId → API/DTO/URL/JWT/integrations
```

Internal numeric IDs are not exposed through the public API.

## Multi-tenancy

```text
Tenant
└── Unit(s)
```

MVP isolation strategy:

```text
Shared Database
+ Shared Schema
+ tenant_id
```

Tenant is resolved from authenticated server context. Unit access is validated independently inside the tenant.

## Development workflow

`main` remains stable.

```text
branch
→ manual backend implementation
→ validation/tests
→ Swagger/Postman update
→ Pull Request
→ review
→ merge into main
```

Backend functional code is implemented manually by the project owner unless direct implementation is explicitly requested.

## MVP direction

The API is designed to cover:

```text
SaaS organization
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
follow-up
patient instructions/materials
exam orders/results
prescriptions
clinical documents
private file storage
notifications
operational APIs
audit
```

The core remains specialty-neutral and may later be extended with dedicated specialty modules when real workflows require them.