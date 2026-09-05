# AkosMed Master Dossier

## 1. Product identity

AkosMed is a backend-first multi-tenant SaaS platform for outpatient clinics, private practices and multidisciplinary healthcare organizations.

The initial product focus is not a hospital information system. It is the operational and clinical lifecycle around:

```text
organization
→ staff
→ patients
→ scheduling
→ encounter
→ longitudinal record
→ follow-up
→ exams
→ prescriptions/documents
→ daily operation
→ audit
```

Repository:

```text
gbsalermo/AkosMed-BackEnd
```

Base package:

```text
br.com.akosmed
```

Project language:

```text
English
```

---

# 2. Architecture

Style:

```text
modular monolith by feature
```

Request flow:

```text
Client
↓ HTTP/JSON
Security
↓
Authenticated Tenant/User Context
↓
Controller
↓
Request DTO + Validation
↓
Service
↓
Repository
↓
JPA/Hibernate
↓
Database
```

Response flow:

```text
Database Entity
↓
Service/Mapper
↓
Response DTO
↓
Controller
↓
JSON
```

The backend is the source of truth for business rules, tenant isolation, unit access, authorization, scheduling concurrency and clinical-history integrity.

---

# 3. SaaS tenancy

Model:

```text
Tenant
└── Unit(s)
```

`Tenant` is the independent SaaS customer boundary.

`Unit` is an operational location inside the tenant.

The MVP uses:

```text
Shared Database
+ Shared Schema
+ tenant_id
```

Tenant identity is derived from the authenticated context, never trusted as a free internal ID from the client.

A user may have access to all units or explicit `UserUnit` links.

---

# 4. Identifiers

Canonical rule:

```text
Long id       → internal PK/FK and locks
UUID publicId → external API identifier
```

Never expose internal Long IDs through public DTOs or URLs.

UUID does not replace authorization.

---

# 5. MVP domain groups

## SaaS and access

```text
Tenant
Unit
Person
User
UserTenant
UserUnit
RefreshToken
```

## Clinical directory

```text
Specialty
Procedure
HealthcareProfessional
ProfessionalSpecialty
ProfessionalUnit
ProfessionalProcedure
Patient
```

## Scheduling

```text
AvailabilityRule
ScheduleBlock
Appointment
AppointmentEvent
```

## Clinical record

```text
MedicalRecord
Encounter
ClinicalNote
ClinicalMeasurement
```

## Longitudinal care

```text
CarePlan
CarePlanEntry
PatientInstruction
```

## Diagnostics

```text
ExamOrder
ExamOrderItem
ExamResult
```

## Prescription and documents

```text
Prescription
PrescriptionItem
ClinicalDocument
```

## Storage and operation

```text
StoredAsset
Attachment
Notification
AuditLog
```

Canonical details: `DOMAIN_MODEL.md`.

---

# 6. Core clinical principle

The core remains specialty-neutral.

Examples such as nutrition and physiotherapy are supported through shared concepts:

```text
Encounter
ClinicalNote
ClinicalMeasurement
CarePlan
CarePlanEntry
PatientInstruction
```

This gives longitudinal tracking without creating `NutritionPatient`, `PhysiotherapyPatient`, `DentalPatient`, etc.

Specialty-specific entities are added only when a real specialty workflow cannot be represented safely by the generic core.

---

# 7. Scheduling and concurrency

Availability is calculated from:

```text
AvailabilityRule
+ ScheduleBlock
+ active Appointment overlaps
= available slots
```

Calculated slots are not persisted.

Interval semantics:

```text
[start, end)
```

Overlap:

```text
existing.startAt < newEnd
AND
existing.endAt > newStart
```

Appointment creation/rescheduling serializes the disputed professional inside the transaction through pessimistic locking and revalidates the slot before insert/update.

Required race result:

```text
same professional + same period
→ 1 success
→ 1 APPOINTMENT_CONFLICT
```

Real concurrency is revalidated on PostgreSQL with Testcontainers before production readiness.

---

# 8. Clinical history

`MedicalRecord` is the longitudinal root, not a giant clinical form.

History is built from:

```text
MedicalRecord
├── Encounter
│   ├── ClinicalNote
│   ├── ClinicalMeasurement
│   ├── Prescription
│   ├── ExamOrder
│   ├── ClinicalDocument
│   └── Attachment
└── CarePlan
    ├── CarePlanEntry
    └── PatientInstruction
```

Finalized clinical history is preserved.

Corrections use explicit rectification/revision instead of silent destructive update.

---

# 9. Documents and diagnostics

Structured domain objects are kept when business rules matter:

```text
Prescription
ExamOrder
```

Generic printable documents use:

```text
ClinicalDocument
```

Initial types:

```text
MEDICAL_CERTIFICATE
ATTENDANCE_DECLARATION
REFERRAL
MEDICAL_REPORT
OTHER
```

Issued documents are snapshots.

---

# 10. Storage

Large/private files are stored outside PostgreSQL by default.

The database stores metadata through `StoredAsset`.

`StorageService` abstracts the provider.

Development may use local filesystem. Production may use object storage.

Authorization always happens before download/access.

---

# 11. API strategy

Base path:

```text
/api/v1
```

The project is backend-first and no frontend is required during MVP implementation.

Therefore:

```text
endpoint
→ Swagger/OpenAPI
→ Postman scenario
→ automated tests
```

Swagger becomes the live contract once controllers exist.

Postman grows incrementally and is consolidated for final end-to-end validation.

Canonical endpoint map: `API_CONTRACT.md`.

---

# 12. Database strategy

Early development:

```text
H2
+ JPA/Hibernate
```

Purpose:

- evolve the domain quickly;
- validate mappings;
- validate relationships;
- validate services;
- validate API behavior.

Stabilization:

```text
PostgreSQL
+ Testcontainers
+ ddl-auto=validate
```

A controlled schema migration strategy is mandatory before production PostgreSQL. Flyway is the recommended default, but it is not early-development overhead and the final tool decision occurs in the PostgreSQL stage.

---

# 13. Git and development process

`main` remains stable.

Every new change uses:

```text
branch
→ implementation/review
→ validation
→ Pull Request
→ merge into main
```

Backend code is implemented manually by the project owner. AI support is primarily for architecture, explanation, reference snippets and review unless direct implementation is explicitly requested.

---

# 14. Quality guardrails

Always review:

- English naming consistency;
- request/response DTO separation;
- tenant isolation;
- unit access;
- publicId contract;
- no Entity returned directly;
- no Controller → Repository shortcut;
- explicit link entities instead of ManyToMany;
- LAZY relations when practical;
- no convenience CascadeType.ALL;
- BigDecimal for money;
- central Clock for time-dependent rules;
- stable error codes + correlation ID;
- pagination for large collections;
- clinical-history preservation;
- concurrency where resources are disputed;
- secrets outside Git.

Canonical guardrails: `ENGINEERING_RULES.md`.

---

# 15. Current stage

The project is still in planning.

Current roadmap stage:

```text
Stage 0 — Product and API Blueprint
Substage 0.1 — MVP capability map
```

No H2/application implementation should begin until the blueprint is reviewed and accepted.

Canonical execution order: `ROADMAP.md`.
