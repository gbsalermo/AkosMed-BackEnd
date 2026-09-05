# AkosMed MVP Scope

## Purpose

AkosMed is a multi-tenant SaaS backend for outpatient clinics, private practices and multidisciplinary healthcare organizations.

The MVP must be generic enough to support different clinical realities without creating a different core model for each specialty.

The API is the product boundary. A web frontend, mobile application or external integration may consume it later, but business rules remain in the backend.

---

# 1. MVP capability map

## 1.1 SaaS organization

The platform must support:

- multiple independent tenants;
- one or many units per tenant;
- tenant suspension/activation;
- unit activation/deactivation;
- tenant timezone;
- users linked to one or more tenants;
- users restricted to specific units when required.

A tenant represents the customer organization. A unit represents an operational location inside that organization.

Examples:

```text
Private practice
Tenant
└── Main Unit

Clinic
Tenant
├── Downtown Unit
└── North Unit

Multidisciplinary clinic
Tenant
└── Main Unit
    ├── Medicine
    ├── Nutrition
    ├── Physiotherapy
    └── Psychology
```

---

## 1.2 Identity and access

The MVP must support:

- login;
- access token;
- refresh token;
- logout/revocation;
- tenant-scoped sessions;
- role-based authorization;
- unit access validation;
- global platform administration.

Initial tenant roles:

```text
TENANT_ADMIN
RECEPTIONIST
HEALTHCARE_PROFESSIONAL
PATIENT
AUDITOR
```

Global capability:

```text
SUPER_ADMIN
```

The tenant is resolved from the authenticated context. The API never trusts a free internal `tenantId` received from the client.

---

## 1.3 People, patients and professionals

The MVP must support:

- person registration;
- patient registration;
- healthcare professional registration;
- professional specialties;
- professional units;
- professional procedures/services;
- professional registry data;
- patient administrative status;
- patient record number;
- tenant-scoped search and pagination.

A person may participate in the tenant as a patient, a healthcare professional or both.

---

## 1.4 Specialty and procedure catalog

The MVP must support a tenant-owned catalog of:

- specialties;
- procedures/services;
- default duration;
- reference price when useful;
- professional-specific duration override;
- professional-specific price override;
- activation/deactivation.

The catalog is generic. It must not hardcode medicine, nutrition or physiotherapy into the core.

---

## 1.5 Calendar and scheduling

The MVP must support:

- recurring professional availability;
- availability per unit;
- schedule blocks;
- vacations/absence/meeting/manual blocks;
- available-slot calculation;
- appointments;
- appointment confirmation;
- cancellation;
- check-in;
- no-show;
- rescheduling;
- professional calendar views;
- unit calendar views;
- patient appointment history;
- concurrency protection against double booking.

Available slots are calculated, not persisted.

Appointment intervals use `[start, end)` semantics.

---

## 1.6 Clinical encounter

The MVP must support both scheduled and walk-in encounters.

A clinical encounter must provide:

- patient;
- healthcare professional;
- unit;
- optional appointment;
- start/end timestamps;
- encounter status;
- clinical history connection;
- clinical notes;
- measurements;
- prescriptions;
- exam orders/results;
- generated clinical documents;
- attachments;
- longitudinal follow-up links.

---

## 1.7 Medical record and clinical history

Each patient has one medical record per tenant.

The medical record acts as the longitudinal clinical root and must expose history through encounters rather than becoming one oversized table.

The MVP must support:

- chronological encounter history;
- clinical notes;
- corrections/rectifications without destroying the original entry;
- measurements over time;
- prescriptions history;
- exam history;
- clinical documents;
- attachments;
- care plan history.

Clinical history is preserved. Destructive hard delete is not allowed for finalized clinical records.

---

## 1.8 Clinical measurements

To support generic longitudinal tracking across specialties, the MVP should allow structured numeric measurements such as:

- weight;
- height;
- BMI;
- pain score;
- range of motion;
- heart rate;
- blood pressure values;
- body composition measurements;
- other tenant/professional-defined measurements.

The core entity remains generic through a measurement code/name, numeric value and unit.

This avoids creating specialty-specific tables before a real specialty module requires them.

---

## 1.9 Care plans and follow-up

The MVP must support longitudinal follow-up beyond a single appointment.

Examples:

```text
Nutrition
patient
→ care plan
→ periodic measurements
→ follow-up entries
→ new goals/instructions

Physiotherapy
patient
→ care plan
→ sessions
→ pain/range-of-motion measurements
→ exercise instructions
→ progress history
```

Capabilities:

- create care plan;
- assign responsible professional;
- optional specialty;
- define title/goal;
- activate/complete/cancel;
- add follow-up entries;
- link entries to encounters when applicable;
- store next-review date;
- expose chronological progress;
- send patient instructions/materials.

---

## 1.10 Patient instructions and materials

The backend should support professional-to-patient guidance such as:

- text instructions;
- links;
- documents;
- videos;
- exercise guidance;
- nutrition guidance;
- post-procedure instructions.

Generic type:

```text
TEXT
LINK
FILE
VIDEO
```

Media is stored outside PostgreSQL through a storage abstraction. The API never exposes the internal storage key.

---

## 1.11 Exam orders and results

The MVP must support diagnostic workflows without becoming a laboratory information system.

Capabilities:

- create an exam order from an encounter;
- add one or more requested exams;
- free-text instructions;
- requested date/status;
- register result metadata;
- result summary;
- result date;
- optional result file;
- chronological patient exam history.

The core must work for laboratory exams, imaging and other diagnostic requests.

---

## 1.12 Prescriptions

The MVP must support structured prescriptions:

- draft;
- prescription items;
- medication name;
- concentration/presentation;
- dose;
- administration route;
- frequency;
- duration;
- instructions;
- issue;
- cancel;
- immutable normal flow after issue;
- patient prescription history;
- printable/renderable representation.

No pharmaceutical catalog is required for the first MVP.

---

## 1.13 Clinical documents

The API must support generic clinical document generation for cases such as:

```text
MEDICAL_CERTIFICATE
ATTENDANCE_DECLARATION
REFERRAL
MEDICAL_REPORT
OTHER
```

Prescriptions and exam orders remain structured domain objects and may also generate printable documents.

A finalized document is treated as a snapshot. Editing the underlying encounter must not silently rewrite an already finalized document.

Digital signature is outside the first MVP.

---

## 1.14 Files and storage

The MVP must support private clinical files through a `StorageService` abstraction.

Use cases:

- encounter attachments;
- exam result files;
- generated documents;
- patient instruction files/videos.

Rules:

- no large BLOBs in PostgreSQL by default;
- authorization before file access;
- storage key remains internal;
- MIME/type and size validation;
- hash/metadata recorded when useful;
- local filesystem may be used in development;
- object storage may be used later in production.

---

## 1.15 Notifications and daily operation

The first MVP should support internal notifications and operational views:

- professional daily agenda;
- checked-in patients;
- upcoming appointments;
- open encounters;
- draft prescriptions;
- unread notifications;
- daily summary endpoints.

External WhatsApp/SMS/push integrations are deferred.

---

## 1.16 Audit and traceability

The MVP must support explicit audit records for critical operations such as:

- access to sensitive clinical records when required;
- appointment changes;
- clinical document issue/cancel;
- prescription issue/cancel;
- permission/profile changes;
- clinical record corrections;
- file access when classified as critical.

Every request must support a correlation ID.

Sensitive clinical content, passwords and full tokens must not be written to application logs.

---

# 2. API-facing use cases

The backend MVP should be capable of supporting the following end-to-end scenario:

```text
Create tenant
→ create unit
→ create tenant administrator
→ create healthcare professional
→ configure specialty and procedures
→ create patient
→ configure professional availability
→ calculate available slots
→ schedule appointment
→ confirm appointment
→ check in patient
→ start encounter
→ record clinical note
→ record measurements
→ optionally create/update care plan
→ optionally request exams
→ optionally register exam result later
→ optionally create prescription
→ optionally create certificate/referral/report
→ conclude encounter
→ query complete patient history
→ query professional calendar
```

---

# 3. Out of scope for the first MVP

Do not implement these until there is a concrete requirement:

- full billing/financial module;
- insurance/TISS;
- inventory/stock;
- laboratory management;
- telemedicine/video consultation;
- real-time chat;
- AI clinical decision making;
- dynamic permission engine;
- SaaS subscription billing;
- advanced digital signature;
- RNDS;
- external messaging gateways;
- pharmacy catalog;
- specialty-specific complex forms;
- hospital admission/bed management.

---

# 4. MVP design principle

```text
Generic core
+ explicit domain relationships
+ specialty-neutral longitudinal history
+ optional specialty modules later
= broad outpatient API without premature specialization
```

The MVP must be broad, but every entity still needs a clear lifecycle and real use case. Generic does not mean creating unbounded JSON/EAV structures for everything.