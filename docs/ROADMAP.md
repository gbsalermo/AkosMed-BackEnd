# AkosMed Development Roadmap

This roadmap is the canonical implementation sequence.

Work one substage at a time.

`[x]` means implemented/tested/reviewed when code exists, not merely planned.

---

# Stage 0 — Product and API Blueprint

No application code or database setup is required in this stage.

The goal is to define what the backend must do before choosing tables or writing CRUDs.

## 0.1 MVP capability map

Status: **current**.

Tasks:

- [ ] confirm the target outpatient-clinic realities;
- [ ] confirm the MVP capability list;
- [ ] confirm what is explicitly out of scope;
- [ ] confirm tenant vs unit semantics;
- [ ] confirm longitudinal care/follow-up requirements;
- [ ] confirm exams/results requirements;
- [ ] confirm document-generation requirements;
- [ ] confirm API-first/no-frontend development strategy;
- [ ] review `docs/MVP_SCOPE.md`;
- [ ] update `CONTINUITY.md` after approval.

Acceptance:

```text
The MVP capability list is broad enough for real clinic/private-practice use,
but each capability has a concrete use case and lifecycle.
```

## 0.2 Core workflows and business rules

Map and validate the main flows:

```text
Tenant/Unit onboarding
User/role/unit access
Patient registration
Professional registration
Availability
Appointment booking
Appointment conflict
Check-in
Encounter start/complete
Clinical note/rectification
Measurement history
Care plan/follow-up
Exam order/result
Prescription issue/cancel
Clinical document issue/cancel
Attachment/file access
Notifications
Audit
```

Acceptance:

- [ ] state transitions defined;
- [ ] critical forbidden transitions defined;
- [ ] tenant/unit rules defined;
- [ ] clinical history immutability defined;
- [ ] concurrency rules defined.

## 0.3 Domain model

Review `docs/DOMAIN_MODEL.md` entity by entity.

Acceptance:

- [ ] entity list approved;
- [ ] relationship directions approved;
- [ ] link entities approved;
- [ ] tenant-scoped/global classification approved;
- [ ] unique constraints documented;
- [ ] historical vs mutable entities identified;
- [ ] no specialty-specific entity introduced without necessity.

## 0.4 API contract map

Review `docs/API_CONTRACT.md`.

Acceptance:

- [ ] main endpoints mapped;
- [ ] request relationships use publicId;
- [ ] `/me` semantics approved;
- [ ] pagination candidates identified;
- [ ] error codes/actions mapped;
- [ ] Swagger/Postman incremental strategy approved.

## 0.5 Blueprint freeze

Final pre-code review.

- [ ] `MVP_SCOPE.md` reviewed;
- [ ] `DOMAIN_MODEL.md` reviewed;
- [ ] `API_CONTRACT.md` reviewed;
- [ ] `ENGINEERING_RULES.md` reviewed;
- [ ] open decisions resolved or explicitly deferred;
- [ ] `CONTINUITY.md` points to Stage 1.1.

Only after Stage 0 is accepted should project implementation begin.

---

# Stage 1 — Project Foundation

## 1.1 Spring Initializr project

Create the backend manually with:

```text
Java 21
Maven
Spring Boot
Group: br.com.akosmed
Artifact: akosmed-backend
Package: br.com.akosmed
```

Initial dependencies:

```text
Spring Web
Spring Data JPA
Validation
H2 Database
Spring Boot Test
```

Do not add yet unless needed by the specific substage:

```text
PostgreSQL Driver
Spring Security/JWT
object-storage SDK
messaging
complex mapping libraries
```

Acceptance:

```text
mvn test → BUILD SUCCESS
application starts
base package is correct
no domain entities created yet
```

Git:

```text
branch: feat/project-foundation
Pull Request required
```

## 1.2 H2 and profiles

Create environment configuration:

```text
application.yml
application-dev.yml
application-test.yml
```

Development:

```text
H2
Hibernate-managed development schema
optional H2 console only in dev
```

Tests:

```text
in-memory H2
isolated schema lifecycle
```

No real secrets committed.

## 1.3 Shared HTTP foundation

Implement:

- `ApiErrorResponse`;
- field validation errors;
- `GlobalExceptionHandler`;
- stable error code pattern;
- `CorrelationIdFilter`;
- central `Clock` bean;
- pagination response abstraction when first required.

## 1.4 OpenAPI and manual API validation foundation

Because the project is API-first, add Swagger/OpenAPI before the first real CRUD is considered complete.

Create the initial Postman environment/collection structure when the first endpoint is implemented.

## 1.5 Package foundation and smoke tests

Create only packages currently required.

Run:

```text
contextLoads
basic error contract test
correlation ID test
Clock.fixed test
mvn test
```

---

# Stage 2 — SaaS Organization

## 2.1 Tenant

Implement full vertical slice:

```text
Entity
Enum
Request DTOs
Response DTOs
Repository
Service
Controller
Tests
Swagger
Postman
```

Main rules:

- publicId immutable;
- slug globally unique;
- timezone required;
- no hard delete;
- activation/suspension actions;
- optimistic locking evaluated.

## 2.2 Unit

Main rules:

- belongs to Tenant;
- code unique inside Tenant;
- suspended Tenant cannot receive new active Unit;
- publicId in API;
- same code allowed in different Tenants;
- tenant isolation tests required.

---

# Stage 3 — Identity, Authentication and Access

Add Spring Security/JWT now.

## 3.1 Person
## 3.2 User
## 3.3 UserTenant
## 3.4 UserUnit
## 3.5 Tenant-scoped login
## 3.6 Refresh token/logout
## 3.7 Authenticated context
## 3.8 Unit access service
## 3.9 Cross-tenant and role tests

Stage gate:

```text
Tenant is server-resolved from authenticated identity.
Unit access is validated.
No internal database ID is required by the public API.
```

---

# Stage 4 — Clinical Directory

## 4.1 Specialty
## 4.2 Procedure
## 4.3 HealthcareProfessional
## 4.4 ProfessionalSpecialty
## 4.5 ProfessionalUnit
## 4.6 ProfessionalProcedure
## 4.7 Patient

Required qualities:

- explicit link entities;
- tenant-safe relationships;
- pagination for professionals/patients;
- BigDecimal for prices;
- no specialty-specific subclass explosion.

---

# Stage 5 — Scheduling Foundation

## 5.1 AvailabilityRule
## 5.2 ScheduleBlock
## 5.3 Available-slot calculation
## 5.4 Calendar query views

Rules:

- no persisted calculated slots;
- professional must belong to Unit;
- effective procedure duration resolves professional override first;
- tenant timezone respected;
- schedule block removes availability.

---

# Stage 6 — Appointment Lifecycle and Concurrency

## 6.1 Appointment
## 6.2 AppointmentEvent
## 6.3 confirm/cancel/check-in/no-show
## 6.4 rescheduling
## 6.5 pessimistic concurrency protection
## 6.6 optional idempotency evaluation

Critical rule:

```text
same professional + overlapping active period
→ never two successful active appointments
```

Expected race result:

```text
1 success
1 409 APPOINTMENT_CONFLICT
```

---

# Stage 7 — Medical Record and Encounters

## 7.1 MedicalRecord
## 7.2 Encounter from appointment
## 7.3 Walk-in Encounter
## 7.4 Encounter lifecycle
## 7.5 ClinicalNote
## 7.6 note rectification
## 7.7 ClinicalMeasurement
## 7.8 patient timeline query

Clinical history is preserved and cross-tenant access is denied.

---

# Stage 8 — Longitudinal Care and Follow-up

## 8.1 CarePlan
## 8.2 CarePlanEntry
## 8.3 measurement history by CarePlan
## 8.4 PatientInstruction
## 8.5 text/link material
## 8.6 private file/video material through StorageService

Goal:

Support generic nutrition, physiotherapy and other multi-session follow-up workflows without building specialty-specific core modules.

---

# Stage 9 — Diagnostics

## 9.1 ExamOrder
## 9.2 ExamOrderItem
## 9.3 issue/cancel order
## 9.4 ExamResult
## 9.5 result revision/history
## 9.6 optional result file
## 9.7 patient diagnostic history

The module manages orders/results, not full laboratory operations.

---

# Stage 10 — Prescription and Clinical Documents

## 10.1 Prescription
## 10.2 PrescriptionItem
## 10.3 issue/cancel
## 10.4 printable prescription representation
## 10.5 ClinicalDocument
## 10.6 certificate/declaration/referral/report
## 10.7 issue/cancel document
## 10.8 printable/downloadable snapshot

Issued records are not silently edited.

---

# Stage 11 — Storage and Attachments

Storage infrastructure may start earlier when Stage 8/9 requires it, but this stage consolidates it.

## 11.1 StoredAsset
## 11.2 StorageService
## 11.3 local storage implementation
## 11.4 Attachment
## 11.5 MIME/size/hash rules
## 11.6 authorization before download
## 11.7 failure consistency tests

---

# Stage 12 — Notifications and Operational API

## 12.1 Notification
## 12.2 professional daily agenda
## 12.3 waiting patients
## 12.4 open encounters
## 12.5 daily summary
## 12.6 patient `/me` read APIs

No external messaging integration yet.

---

# Stage 13 — Audit and Security Hardening

## 13.1 AuditLog
## 13.2 resource × role × action matrix
## 13.3 sensitive-log review
## 13.4 refresh-token hardening
## 13.5 CORS/security headers
## 13.6 authentication rate limiting strategy
## 13.7 optimistic locking review
## 13.8 idempotency review
## 13.9 storage-access review

---

# Stage 14 — Core Review in H2

Before PostgreSQL:

- all automated tests green;
- no H2-specific domain dependency;
- tenant isolation reviewed everywhere;
- publicId reviewed everywhere;
- no Long ID exposed;
- DTO sizes/contracts reviewed;
- JPA relationships reviewed;
- cascades reviewed;
- pagination reviewed;
- appointment concurrency behavior reviewed;
- clinical history immutability reviewed;
- Swagger matches implementation;
- Postman collection matches implementation;
- full MVP flow executes locally.

---

# Stage 15 — PostgreSQL and Real Database Validation

Add:

```text
PostgreSQL Driver
Testcontainers PostgreSQL
```

Before production database use, choose/implement the schema migration strategy.

Recommended default:

```text
Flyway
+
ddl-auto=validate
```

But migration tooling is a Stage 15 decision, not early-development overhead.

Validate:

- constraints;
- indexes;
- UUID type mapping;
- numeric precision;
- timezone;
- pagination;
- locks;
- optimistic locking;
- real concurrent appointment transactions;
- clean database bootstrap/migrations.

Evaluate PostgreSQL exclusion constraint as a second appointment-overlap barrier.

---

# Stage 16 — API End-to-End Freeze

Swagger/OpenAPI:

- review all paths;
- schemas;
- examples;
- error codes;
- JWT;
- pagination;
- idempotency headers where adopted.

Postman:

- happy paths;
- validation;
- authentication;
- tenant isolation;
- unit access;
- conflicts;
- complete clinical flow.

---

# Stage 17 — Backend MVP Production Readiness

- Actuator/health;
- production secrets strategy;
- upload limits;
- private storage configuration;
- backup strategy;
- restore procedure and real restore test;
- observability baseline;
- logs/correlation review;
- production HTTPS/deployment documentation;
- README execution guide;
- final continuity update;
- release/tag `Backend MVP 1.0`.

---

# Post-MVP

Only after MVP stabilization:

- specialty-specific modules;
- insurance/TISS;
- billing;
- inventory;
- external messaging;
- digital signatures;
- telemedicine;
- patient mobile app;
- professional assistant;
- integrations;
- SaaS billing;
- AI-assisted non-autonomous workflows.

---

# Working rule

For every implementation substage:

```text
read CONTINUITY.md
→ read this roadmap
→ read related canonical docs
→ create branch
→ implement manually
→ run tests
→ validate Swagger/Postman
→ review
→ open PR
→ merge
→ update CONTINUITY.md
→ next substage
```
