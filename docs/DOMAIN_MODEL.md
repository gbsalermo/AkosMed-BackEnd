# AkosMed Domain Model

This document defines the initial canonical entity map for the backend MVP.

All persisted domain entities use:

```text
Long id       → internal PK/FK only
UUID publicId → external API identifier
```

Tenant-scoped entities also carry a `Tenant` relationship when isolation/querying requires it.

---

# 1. Organization and access

## Tenant

Represents one SaaS customer organization.

Core fields:

```text
id
publicId
name
tradeName nullable
slug
documentNumber nullable
status
timezone
createdAt
updatedAt
```

Relations:

```text
Tenant 1:N Unit
Tenant 1:N Person
Tenant 1:N UserTenant
```

---

## Unit

Represents an operational clinic/office/location inside a tenant.

Core fields:

```text
id
publicId
tenant
name
code
phone nullable
email nullable
address fields nullable
active
createdAt
updatedAt
```

Constraint:

```text
unique(tenant_id, code)
```

---

## Person

Stores personal data inside one tenant.

Core fields:

```text
id
publicId
tenant
fullName
socialName nullable
documentNumber nullable
birthDate nullable
phone nullable
contactEmail nullable
createdAt
updatedAt
```

A Person may become a Patient, HealthcareProfessional, or both.

---

## User

Global authentication identity.

```text
id
publicId
loginEmail
passwordHash
status
superAdmin
lastLoginAt nullable
createdAt
updatedAt
```

`loginEmail` is globally unique.

---

## UserTenant

Links a global User to a Tenant and Person.

```text
id
publicId
user
tenant
person
role
allUnitsAccess
active
createdAt
```

Constraints:

```text
unique(user_id, tenant_id)
unique(tenant_id, person_id)
```

---

## UserUnit

Explicit unit access when `UserTenant.allUnitsAccess = false`.

```text
id
publicId
userTenant
unit
```

Constraint:

```text
unique(user_tenant_id, unit_id)
```

---

## RefreshToken

```text
id
publicId
user
userTenant nullable
tokenHash
expiresAt
revokedAt nullable
createdAt
```

---

# 2. Clinical directory

## Specialty

Tenant-owned specialty catalog.

```text
id
publicId
tenant
name
code
description nullable
active
createdAt
updatedAt
```

Constraint:

```text
unique(tenant_id, code)
```

---

## Procedure

Tenant-owned service/procedure catalog.

```text
id
publicId
tenant
name
code
description nullable
defaultDurationMinutes
referencePrice nullable
active
createdAt
updatedAt
```

Use `BigDecimal` for prices.

---

## HealthcareProfessional

```text
id
publicId
tenant
person
professionalType
council nullable
registrationNumber nullable
registrationState nullable
active
createdAt
updatedAt
```

Constraint:

```text
unique(tenant_id, person_id)
```

---

## ProfessionalSpecialty

Explicit N:N link entity.

```text
id
publicId
professional
specialty
primary
```

Constraint:

```text
unique(professional_id, specialty_id)
```

---

## ProfessionalUnit

```text
id
publicId
professional
unit
active
```

Constraint:

```text
unique(professional_id, unit_id)
```

---

## ProfessionalProcedure

```text
id
publicId
professional
procedure
durationMinutesOverride nullable
priceOverride nullable
active
```

Constraint:

```text
unique(professional_id, procedure_id)
```

---

## Patient

```text
id
publicId
tenant
person
recordNumber
status
administrativeNotes nullable
createdAt
updatedAt
```

Constraints:

```text
unique(tenant_id, person_id)
unique(tenant_id, record_number)
```

The patient is tenant-wide. Unit history is derived from appointments and encounters.

---

# 3. Scheduling

## AvailabilityRule

Recurring professional availability.

```text
id
publicId
tenant
professional
unit
dayOfWeek
startTime
endTime
validFrom
validUntil nullable
active
```

No calculated slots are persisted.

---

## ScheduleBlock

Temporary period in which a professional cannot be scheduled.

```text
id
publicId
tenant
professional
unit
startAt
endAt
type
reason nullable
active
createdByUser
createdAt
```

Initial types:

```text
VACATION
ABSENCE
MEETING
HOLIDAY
MANUAL
OTHER
```

---

## Appointment

```text
id
publicId
tenant
unit
patient
professional
procedure
startAt
endAt
status
source
administrativeNotes nullable
createdByUser
createdAt
updatedAt
```

Initial status flow:

```text
REQUESTED
→ CONFIRMED
→ CHECKED_IN
→ IN_PROGRESS
→ COMPLETED
```

Alternative terminal states:

```text
NO_SHOW
CANCELLED_BY_PATIENT
CANCELLED_BY_CLINIC
```

---

## AppointmentEvent

Append-only scheduling history.

```text
id
publicId
tenant
appointment
type
previousStatus nullable
newStatus nullable
previousStartAt nullable
previousEndAt nullable
reason nullable
user nullable
createdAt
```

---

# 4. Clinical record

## MedicalRecord

One longitudinal record per patient per tenant.

```text
id
publicId
tenant
patient
createdAt
```

Constraint:

```text
unique(tenant_id, patient_id)
```

---

## Encounter

Represents one clinical attendance/session.

```text
id
publicId
tenant
medicalRecord
unit
professional
appointment nullable
startAt
endAt nullable
status
encounterType
createdAt
updatedAt
```

A walk-in encounter has `appointment = null`.

The patient is derived through:

```text
Encounter
→ MedicalRecord
→ Patient
```

---

## ClinicalNote

Append-only professional note.

```text
id
publicId
tenant
encounter
professional
content
rectifiesNote nullable
rectificationReason nullable
createdAt
```

A correction creates a new note and preserves the original.

---

## ClinicalMeasurement

Generic structured numeric measurement.

```text
id
publicId
tenant
medicalRecord
encounter nullable
carePlan nullable
code
name
numericValue
unit nullable
recordedAt
recordedByProfessional
notes nullable
```

Examples:

```text
WEIGHT              72.4 kg
PAIN_SCORE           4
KNEE_FLEXION_ROM   120 deg
SYSTOLIC_BP         120 mmHg
DIASTOLIC_BP         80 mmHg
```

This entity is intentionally simple. Qualitative clinical content remains in ClinicalNote/CarePlanEntry rather than turning the database into a generic EAV system.

---

# 5. Longitudinal care

## CarePlan

Represents a multi-session follow-up plan.

```text
id
publicId
tenant
patient
responsibleProfessional
specialty nullable
title
goal nullable
status
startDate
endDate nullable
createdAt
updatedAt
```

Initial statuses:

```text
ACTIVE
COMPLETED
CANCELLED
```

---

## CarePlanEntry

Chronological progress record inside a CarePlan.

```text
id
publicId
tenant
carePlan
professional
encounter nullable
progressNote
nextReviewAt nullable
createdAt
```

Append-only by default. Corrections should preserve history.

---

## PatientInstruction

Professional guidance/material delivered to a patient.

```text
id
publicId
tenant
patient
professional
carePlan nullable
encounter nullable
title
description nullable
type
externalUrl nullable
storedAsset nullable
visibleFrom nullable
expiresAt nullable
active
createdAt
```

Types:

```text
TEXT
LINK
FILE
VIDEO
```

---

# 6. Exams and diagnostics

## ExamOrder

```text
id
publicId
tenant
patient
professional
encounter
status
instructions nullable
orderedAt
cancelledAt nullable
createdAt
```

Initial statuses:

```text
DRAFT
ORDERED
PARTIALLY_RESULTED
COMPLETED
CANCELLED
```

---

## ExamOrderItem

One requested exam inside an order.

```text
id
publicId
examOrder
code nullable
name
instructions nullable
status
orderIndex
```

---

## ExamResult

Result/history for one exam item.

```text
id
publicId
tenant
examOrderItem
status
resultSummary nullable
performedAt nullable
reportedAt nullable
providerName nullable
storedAsset nullable
supersedesResult nullable
createdAt
```

Results should preserve revisions rather than silently overwriting finalized history.

---

# 7. Prescriptions and documents

## Prescription

```text
id
publicId
tenant
encounter
professional
status
notes nullable
issuedAt nullable
validUntil nullable
createdAt
updatedAt
```

Statuses:

```text
DRAFT
ISSUED
CANCELLED
```

---

## PrescriptionItem

```text
id
publicId
prescription
medicationName
concentration nullable
pharmaceuticalForm nullable
dose
doseUnit nullable
administrationRoute
frequencyText nullable
timesPerDay nullable
intervalHours nullable
durationDays nullable
startDate nullable
endDate nullable
continuousUse
asNeeded
instructions nullable
orderIndex
```

Items are editable only while the prescription is DRAFT.

---

## ClinicalDocument

Generic finalized or draft document generated for clinical/administrative use.

```text
id
publicId
tenant
patient
professional
encounter nullable
type
status
title
contentSnapshot
issuedAt nullable
validUntil nullable
storedAsset nullable
createdAt
updatedAt
```

Types:

```text
MEDICAL_CERTIFICATE
ATTENDANCE_DECLARATION
REFERRAL
MEDICAL_REPORT
OTHER
```

Statuses:

```text
DRAFT
ISSUED
CANCELLED
```

Prescriptions and exam orders remain specialized entities because they have structured rules of their own.

---

# 8. Storage

## StoredAsset

Infrastructure-level metadata for a private file.

```text
id
publicId
tenant
storageKey
originalName
mimeType
sizeBytes
hash nullable
createdAt
```

The API never exposes `storageKey`.

---

## Attachment

Generic medical-record/encounter attachment.

```text
id
publicId
tenant
medicalRecord
encounter nullable
storedAsset
category
uploadedByUser
active
createdAt
```

---

# 9. Operation and audit

## Notification

```text
id
publicId
tenant
userTenant
category
title
message
read
createdAt
readAt nullable
```

---

## AuditLog

Prefer primitive/internal references instead of a large JPA graph.

```text
id
publicId
tenantId nullable
unitId nullable
userId nullable
userTenantId nullable
action
resourceType
resourceId nullable
correlationId nullable
ipAddress nullable
userAgent nullable
metadataJson nullable
createdAt
```

No sensitive clinical body should be copied into audit metadata by default.

---

# 10. Relationship overview

```text
Tenant
├── Unit
├── Person
│   ├── Patient
│   │   └── MedicalRecord
│   │       ├── Encounter
│   │       │   ├── ClinicalNote
│   │       │   ├── Prescription
│   │       │   │   └── PrescriptionItem
│   │       │   ├── ExamOrder
│   │       │   │   └── ExamOrderItem
│   │       │   │       └── ExamResult
│   │       │   ├── ClinicalDocument
│   │       │   └── Attachment
│   │       ├── ClinicalMeasurement
│   │       └── CarePlan
│   │           ├── CarePlanEntry
│   │           └── PatientInstruction
│   │
│   └── HealthcareProfessional
│       ├── ProfessionalSpecialty
│       ├── ProfessionalUnit
│       ├── ProfessionalProcedure
│       └── AvailabilityRule
│
├── UserTenant
│   └── UserUnit
│
├── Specialty
├── Procedure
├── ScheduleBlock
└── Appointment
    └── AppointmentEvent

User
├── UserTenant
└── RefreshToken

StoredAsset
├── Attachment
├── PatientInstruction
├── ExamResult
└── ClinicalDocument
```

---

# 11. JPA relationship rules

- Prefer unidirectional relationships unless reverse navigation is actually needed.
- To-one relationships are LAZY when practical.
- Do not use `@ManyToMany` for domain links; use explicit link entities.
- Do not use `CascadeType.ALL` by convenience.
- Do not use destructive cascade/orphan removal on clinical history.
- Do not serialize entities directly through controllers.
- External relationships are resolved by `...PublicId` + authenticated tenant context.
- Internal FKs use numeric IDs.
- `@Version` is evaluated selectively for mutable administrative records.
- Append-only historical entities do not receive optimistic locking by reflex.

---

# 12. Domain model review gate

Before implementation begins, verify each entity against these questions:

1. Does it represent a real lifecycle/use case?
2. Is it tenant-scoped or global?
3. Which entity owns the FK?
4. What is externally addressable by publicId?
5. Which states are mutable and which are historical?
6. Does the relation require a link entity?
7. Is hard delete allowed?
8. Is optimistic locking useful?
9. Which endpoints need pagination?
10. Which operations require audit or concurrency protection?
