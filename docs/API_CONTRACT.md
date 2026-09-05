# AkosMed API Contract Map

Base path:

```text
/api/v1
```

Public resource identifiers use UUID `publicId`.

The API never exposes internal numeric primary keys.

Requests that reference another domain resource use fields such as:

```text
patientPublicId
professionalPublicId
unitPublicId
procedurePublicId
encounterPublicId
```

Tenant identity comes from the authenticated context, not from a free internal tenant ID in the request.

---

# 1. Authentication

```http
POST /api/v1/auth/login
POST /api/v1/auth/refresh
POST /api/v1/auth/logout
```

Tenant-scoped login request concept:

```json
{
  "email": "user@clinic.com",
  "password": "...",
  "tenantSlug": "clinic-name"
}
```

---

# 2. Global SaaS administration

```http
POST  /api/v1/admin/tenants
GET   /api/v1/admin/tenants
GET   /api/v1/admin/tenants/{tenantPublicId}
PUT   /api/v1/admin/tenants/{tenantPublicId}
PATCH /api/v1/admin/tenants/{tenantPublicId}/activate
PATCH /api/v1/admin/tenants/{tenantPublicId}/suspend
```

Restricted to platform administration.

---

# 3. Units

```http
POST  /api/v1/units
GET   /api/v1/units
GET   /api/v1/units/{unitPublicId}
PUT   /api/v1/units/{unitPublicId}
PATCH /api/v1/units/{unitPublicId}/activate
PATCH /api/v1/units/{unitPublicId}/deactivate
```

---

# 4. Users and tenant membership

```http
POST  /api/v1/users
GET   /api/v1/users
GET   /api/v1/users/{userTenantPublicId}
PUT   /api/v1/users/{userTenantPublicId}
PATCH /api/v1/users/{userTenantPublicId}/activate
PATCH /api/v1/users/{userTenantPublicId}/deactivate
PATCH /api/v1/users/{userTenantPublicId}/role
```

Unit access:

```http
POST   /api/v1/users/{userTenantPublicId}/units/{unitPublicId}
DELETE /api/v1/users/{userTenantPublicId}/units/{unitPublicId}
GET    /api/v1/users/{userTenantPublicId}/units
```

---

# 5. Specialties

```http
POST  /api/v1/specialties
GET   /api/v1/specialties
GET   /api/v1/specialties/{specialtyPublicId}
PUT   /api/v1/specialties/{specialtyPublicId}
PATCH /api/v1/specialties/{specialtyPublicId}/activate
PATCH /api/v1/specialties/{specialtyPublicId}/deactivate
```

---

# 6. Procedures

```http
POST  /api/v1/procedures
GET   /api/v1/procedures
GET   /api/v1/procedures/{procedurePublicId}
PUT   /api/v1/procedures/{procedurePublicId}
PATCH /api/v1/procedures/{procedurePublicId}/activate
PATCH /api/v1/procedures/{procedurePublicId}/deactivate
```

---

# 7. Healthcare professionals

```http
POST  /api/v1/professionals
GET   /api/v1/professionals
GET   /api/v1/professionals/{professionalPublicId}
PUT   /api/v1/professionals/{professionalPublicId}
PATCH /api/v1/professionals/{professionalPublicId}/activate
PATCH /api/v1/professionals/{professionalPublicId}/deactivate
```

Specialty links:

```http
POST   /api/v1/professionals/{professionalPublicId}/specialties/{specialtyPublicId}
DELETE /api/v1/professionals/{professionalPublicId}/specialties/{specialtyPublicId}
GET    /api/v1/professionals/{professionalPublicId}/specialties
```

Unit links:

```http
POST   /api/v1/professionals/{professionalPublicId}/units/{unitPublicId}
DELETE /api/v1/professionals/{professionalPublicId}/units/{unitPublicId}
GET    /api/v1/professionals/{professionalPublicId}/units
```

Procedure links:

```http
POST   /api/v1/professionals/{professionalPublicId}/procedures/{procedurePublicId}
PUT    /api/v1/professionals/{professionalPublicId}/procedures/{procedurePublicId}
DELETE /api/v1/professionals/{professionalPublicId}/procedures/{procedurePublicId}
GET    /api/v1/professionals/{professionalPublicId}/procedures
```

---

# 8. Patients

```http
POST  /api/v1/patients
GET   /api/v1/patients
GET   /api/v1/patients/{patientPublicId}
PUT   /api/v1/patients/{patientPublicId}
PATCH /api/v1/patients/{patientPublicId}/activate
PATCH /api/v1/patients/{patientPublicId}/deactivate
```

Common filters:

```text
name
documentNumber
phone
recordNumber
status
```

Patient list endpoints are paginated.

---

# 9. Availability and schedule blocks

Availability rules:

```http
POST   /api/v1/professionals/{professionalPublicId}/availability
GET    /api/v1/professionals/{professionalPublicId}/availability
PUT    /api/v1/professionals/{professionalPublicId}/availability/{availabilityPublicId}
DELETE /api/v1/professionals/{professionalPublicId}/availability/{availabilityPublicId}
```

Schedule blocks:

```http
POST   /api/v1/professionals/{professionalPublicId}/schedule-blocks
GET    /api/v1/professionals/{professionalPublicId}/schedule-blocks
DELETE /api/v1/professionals/{professionalPublicId}/schedule-blocks/{blockPublicId}
```

Available slots:

```http
GET /api/v1/professionals/{professionalPublicId}/available-slots
```

Query parameters:

```text
date
unitPublicId
procedurePublicId
```

The returned slot is a snapshot, not a reservation.

---

# 10. Calendar views

Tenant/unit calendar:

```http
GET /api/v1/calendar
```

Possible filters:

```text
from
to
unitPublicId
professionalPublicId
status
```

Professional self-service calendar:

```http
GET /api/v1/me/calendar
GET /api/v1/me/calendar/today
```

Calendar endpoints are query views; no `Calendar` entity is required.

---

# 11. Appointments

```http
POST  /api/v1/appointments
GET   /api/v1/appointments
GET   /api/v1/appointments/{appointmentPublicId}
PATCH /api/v1/appointments/{appointmentPublicId}/confirm
PATCH /api/v1/appointments/{appointmentPublicId}/cancel
PATCH /api/v1/appointments/{appointmentPublicId}/check-in
PATCH /api/v1/appointments/{appointmentPublicId}/mark-no-show
POST  /api/v1/appointments/{appointmentPublicId}/reschedule
```

Creation and rescheduling revalidate the selected slot inside the transaction.

Expected race result for the same professional/period:

```text
1 request → 201 Created
1 request → 409 APPOINTMENT_CONFLICT
```

---

# 12. Medical record

```http
GET /api/v1/patients/{patientPublicId}/medical-record
GET /api/v1/patients/{patientPublicId}/medical-record/timeline
GET /api/v1/patients/{patientPublicId}/encounters
GET /api/v1/patients/{patientPublicId}/measurements
GET /api/v1/patients/{patientPublicId}/prescriptions
GET /api/v1/patients/{patientPublicId}/exam-orders
GET /api/v1/patients/{patientPublicId}/care-plans
GET /api/v1/patients/{patientPublicId}/documents
```

The timeline endpoint may aggregate summaries from multiple clinical resources without exposing JPA entities.

---

# 13. Encounters

Start from appointment:

```http
POST /api/v1/appointments/{appointmentPublicId}/start-encounter
```

Walk-in encounter:

```http
POST /api/v1/encounters/walk-in
```

Read/complete:

```http
GET   /api/v1/encounters/{encounterPublicId}
PATCH /api/v1/encounters/{encounterPublicId}/complete
PATCH /api/v1/encounters/{encounterPublicId}/cancel
```

---

# 14. Clinical notes

```http
POST /api/v1/encounters/{encounterPublicId}/notes
GET  /api/v1/encounters/{encounterPublicId}/notes
POST /api/v1/clinical-notes/{notePublicId}/rectify
```

No destructive update/delete for finalized history.

---

# 15. Clinical measurements

```http
POST /api/v1/encounters/{encounterPublicId}/measurements
GET  /api/v1/encounters/{encounterPublicId}/measurements
GET  /api/v1/patients/{patientPublicId}/measurements
```

Possible patient-history filters:

```text
code
from
to
carePlanPublicId
```

This supports graphs/history for weight, pain score, range of motion and other numeric metrics without creating specialty-specific core entities.

---

# 16. Care plans and follow-up

```http
POST  /api/v1/patients/{patientPublicId}/care-plans
GET   /api/v1/patients/{patientPublicId}/care-plans
GET   /api/v1/care-plans/{carePlanPublicId}
PUT   /api/v1/care-plans/{carePlanPublicId}
PATCH /api/v1/care-plans/{carePlanPublicId}/complete
PATCH /api/v1/care-plans/{carePlanPublicId}/cancel
```

Progress entries:

```http
POST /api/v1/care-plans/{carePlanPublicId}/entries
GET  /api/v1/care-plans/{carePlanPublicId}/entries
```

Entries are chronological history records.

---

# 17. Patient instructions/materials

Professional/admin view:

```http
POST  /api/v1/patients/{patientPublicId}/instructions
GET   /api/v1/patients/{patientPublicId}/instructions
GET   /api/v1/patient-instructions/{instructionPublicId}
DELETE /api/v1/patient-instructions/{instructionPublicId}
```

`DELETE` means logical deactivation when history/audit must be preserved.

Patient self-service:

```http
GET /api/v1/me/instructions
GET /api/v1/me/instructions/{instructionPublicId}
```

File/video access must be authorized before the backend opens or signs the stored asset.

---

# 18. Exam orders

```http
POST /api/v1/encounters/{encounterPublicId}/exam-orders
GET  /api/v1/exam-orders/{examOrderPublicId}
GET  /api/v1/patients/{patientPublicId}/exam-orders
POST /api/v1/exam-orders/{examOrderPublicId}/items
POST /api/v1/exam-orders/{examOrderPublicId}/issue
POST /api/v1/exam-orders/{examOrderPublicId}/cancel
```

Exam result:

```http
POST /api/v1/exam-order-items/{itemPublicId}/results
GET  /api/v1/exam-order-items/{itemPublicId}/results
GET  /api/v1/exam-results/{resultPublicId}
```

If a finalized result is corrected, preserve the original result and create a replacement/revision.

---

# 19. Prescriptions

```http
POST   /api/v1/encounters/{encounterPublicId}/prescriptions
GET    /api/v1/prescriptions/{prescriptionPublicId}
GET    /api/v1/patients/{patientPublicId}/prescriptions
POST   /api/v1/prescriptions/{prescriptionPublicId}/items
PUT    /api/v1/prescriptions/{prescriptionPublicId}/items/{itemPublicId}
DELETE /api/v1/prescriptions/{prescriptionPublicId}/items/{itemPublicId}
POST   /api/v1/prescriptions/{prescriptionPublicId}/issue
POST   /api/v1/prescriptions/{prescriptionPublicId}/cancel
GET    /api/v1/prescriptions/{prescriptionPublicId}/document
```

Prescription items are editable only while the prescription is in DRAFT.

---

# 20. Clinical documents

```http
POST /api/v1/encounters/{encounterPublicId}/documents
GET  /api/v1/clinical-documents/{documentPublicId}
GET  /api/v1/patients/{patientPublicId}/documents
PUT  /api/v1/clinical-documents/{documentPublicId}
POST /api/v1/clinical-documents/{documentPublicId}/issue
POST /api/v1/clinical-documents/{documentPublicId}/cancel
GET  /api/v1/clinical-documents/{documentPublicId}/download
```

Initial document types:

```text
MEDICAL_CERTIFICATE
ATTENDANCE_DECLARATION
REFERRAL
MEDICAL_REPORT
OTHER
```

Once issued, a document is a snapshot and is not silently modified.

---

# 21. Attachments and stored assets

Encounter/medical-record attachment:

```http
POST /api/v1/patients/{patientPublicId}/attachments
POST /api/v1/encounters/{encounterPublicId}/attachments
GET  /api/v1/patients/{patientPublicId}/attachments
GET  /api/v1/encounters/{encounterPublicId}/attachments
GET  /api/v1/attachments/{attachmentPublicId}/download
DELETE /api/v1/attachments/{attachmentPublicId}
```

The response never exposes the physical path or storage key.

---

# 22. Notifications and operational `/me` endpoints

```http
GET   /api/v1/me/notifications
PATCH /api/v1/me/notifications/{notificationPublicId}/read
PATCH /api/v1/me/notifications/read-all
```

Professional operation:

```http
GET /api/v1/me/calendar/today
GET /api/v1/me/waiting-patients
GET /api/v1/me/open-encounters
GET /api/v1/me/daily-summary
```

Patient self-service candidate endpoints:

```http
GET /api/v1/me/profile
GET /api/v1/me/appointments
GET /api/v1/me/prescriptions
GET /api/v1/me/exam-orders
GET /api/v1/me/documents
GET /api/v1/me/care-plans
GET /api/v1/me/instructions
```

These endpoints resolve the authenticated person/patient from server context. They do not ask the patient to provide their own patient ID.

---

# 23. Audit

```http
GET /api/v1/audit-logs
GET /api/v1/audit-logs/{auditLogPublicId}
```

Administrative/auditor-only, paginated and filtered.

---

# 24. HTTP behavior

Common responses:

```text
200 OK
201 Created
204 No Content
400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
409 Conflict
```

Cross-tenant resource access should normally return 404 to avoid disclosing resource existence.

---

# 25. Error contract

Example:

```json
{
  "timestamp": "2026-09-04T23:00:00Z",
  "status": 409,
  "code": "APPOINTMENT_CONFLICT",
  "message": "The selected time is no longer available.",
  "path": "/api/v1/appointments",
  "correlationId": "2e256f51-cc07-43a8-a9f7-7fedc6fe6258"
}
```

Stable error codes are part of the API contract. Human-readable messages may evolve.

Initial examples:

```text
VALIDATION_ERROR
RESOURCE_NOT_FOUND
ACCESS_DENIED
TENANT_SUSPENDED
UNIT_ACCESS_DENIED
APPOINTMENT_CONFLICT
APPOINTMENT_STATUS_INVALID
ENCOUNTER_STATUS_INVALID
PRESCRIPTION_NOT_EDITABLE
DOCUMENT_NOT_EDITABLE
EXAM_ORDER_STATUS_INVALID
RESOURCE_VERSION_CONFLICT
IDEMPOTENCY_KEY_REUSED
```

---

# 26. Pagination

Large collections use pagination from the start.

Typical query:

```http
GET /api/v1/patients?page=0&size=20&sort=fullName,asc
```

Recommended response:

```json
{
  "content": [],
  "page": 0,
  "size": 20,
  "totalElements": 0,
  "totalPages": 0,
  "first": true,
  "last": true
}
```

Do not expose Spring's internal `PageImpl` as the permanent public contract.

---

# 27. Swagger and Postman strategy

Unlike the older planning, API documentation and manual contract validation are incremental:

```text
first endpoint
→ OpenAPI/Swagger updated
→ Postman request added
→ automated tests added as the feature matures
```

At the final stabilization stage:

```text
Swagger/OpenAPI
→ reviewed as the canonical HTTP contract

Postman
→ consolidated into the official end-to-end collection
```

Because AkosMed is backend-first and no frontend is required for the MVP development process, the API contract must remain easy to inspect and test throughout implementation.