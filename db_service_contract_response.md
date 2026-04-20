# DB Service Contract Response For CRM

## Purpose

This document is Team B's authoritative response to the CRM-side DB service contract.

Its job is to close the gap between:

- what CRM asked for
- what Team B has agreed to build
- how Team B's schema maps to the CRM-facing service contract

This is the contract Team B intends to implement against the `healthOs-db` design.
It is not a copy of the CRM document.
It is the DB-side contract lock for CRM integration.

---

## Scope

This document defines:

- the CRM-facing methods Team B supports
- the Team B naming and entity mapping rules
- idempotency, versioning, and compare-and-set semantics
- the machine-readable error model
- the places where CRM terms map to existing Team B entities

This document does not define:

- transport choice
- storage engine
- indexing implementation details
- internal event bus implementation

---

## Contract Position

Team B accepts the CRM requirement that the CRM service needs a stable typed DB-service contract.

Team B will support the following principle:

- CRM owns runtime execution behavior
- Team B owns durable business truth and durable execution records exposed through the DB service contract

Short version:

- CRM should not guess Team B behavior
- Team B should not leave correctness-critical semantics implicit

For the current non-telemedicine CRM V1 scope, Team B also accepts the following bounded workflow-family refinements:

- `diagnosis` is the only currently ratified diagnosis-family workflow type
- `custom` is accepted only for approved custom-form-driven workflows and campaigns
- `custom` does not imply arbitrary undefined runtime workflow behavior

---

## Shared Request Metadata

All CRM-to-DB-service requests are scoped by the following metadata envelope.

| Field | Status | Notes |
|---|---|---|
| `organization_id` | required | canonical top-level scope key |
| `facility_id` | optional | required when the operation is facility-scoped |
| `correlation_id` | required | end-to-end trace across services |
| `causation_id` | optional | direct trigger for the request |
| `trace_id` | optional | observability/tracing field |
| `idempotency_key` | required on retry-safe create or patch methods | used only where replay-safe behavior is required |

Important rule:

- `organization_id` is a contract field and a persistent scoping field
- `correlation_id`, `causation_id`, and `trace_id` are contract-envelope metadata by default, not universal entity columns

---

## Shared Version Metadata

All mutable CRM-facing records returned by Team B will expose:

| Field | Meaning |
|---|---|
| `version` | monotonic record version |
| `updated_at` | timestamp of the latest successful write |

This version metadata is part of the contract, not optional convenience.

---

## Machine-Readable Error Codes

Team B will normalize DB-service failures into this CRM-facing set:

| Code | Meaning |
|---|---|
| `not_found` | requested resource does not exist in the requested scope |
| `validation_failed` | request shape or state transition is invalid |
| `concurrency_conflict` | compare-and-set or expected-version precondition failed |
| `permission_denied` | actor or scope is not allowed |
| `rate_limited` | request was rejected by quota or throttling policy |
| `unavailable` | upstream DB-service path is unavailable |
| `timeout` | the operation exceeded the contract timeout |
| `definition_source_unavailable` | custom-form definition source unavailable |

---

## Naming And Mapping Rules

The CRM-facing contract will use the following mappings.

| CRM term | Team B canonical home | Contract note |
|---|---|---|
| `organization` | `organizations` | aligned |
| `facility` | `facilities` | aligned |
| `user` | `users` | aligned at service-contract level |
| `patient` | `patients` + `patient_contacts` | CRM patient lookup remains a simplified projection |
| `consent state` | `patient_current_consent` | current-state read path |
| `consent history` | `consent_records` | append-only history |
| `appointment` | `appointments` | canonical scheduling entity |
| `medication` | `patient_medications` | CRM `medication_id` maps to canonical patient medication ID |
| `call provider key` | `provider_call_sid` | Team B adopts `provider_call_sid` as the CRM-facing name |
| `message provider key` | `provider_message_sid` | aligned with CRM contract |

Important rule:

- Team B will not expose a second generic medication master to satisfy CRM wording
- CRM medication-facing methods resolve against canonical `patient_medications`

---

## Supported Method Surface

The following methods are accepted as the Team B V1 CRM-facing surface.

### Organization, Facility, User

- `get_organization`
- `get_facility`
- `get_user`

### Patient And Consent

- `get_patient`
- `find_patient_by_phone`
- `get_current_consent_state`

### Appointment

- `get_appointment`
- `list_upcoming_patient_appointments`
- `patch_appointment_reminder_state`
- `append_appointment_reschedule_history`

### Medication

- `get_medication`
- `list_active_patient_medications`
- `create_medication_adherence_log`

### Workflow

- `create_workflow`
- `get_workflow`
- `list_workflows`
- `patch_workflow_status`

### Conversation, Call, Message Thread

- `create_conversation`
- `patch_conversation`
- `append_conversation_context_switch`
- `create_call`
- `get_call`
- `get_call_by_provider_sid`
- `upsert_call_by_provider`
- `patch_call_status`
- `upsert_message_thread_by_provider`
- `patch_message_status`

### Operation And Artifact

- `create_operation`
- `patch_operation`
- `create_artifact`
- `get_artifact`
- `create_artifact_export`

### Scheduling Concurrency

- `list_practitioner_availability`
- `hold_practitioner_slot`
- `confirm_practitioner_slot`
- `release_practitioner_slot`

### Structured CRM Outputs

- `create_staff_notification`
- `create_symptom_report`
- `create_patient_feedback`

### Custom Forms

- `get_custom_form_definition`

### Webhook Delivery Support

- `list_active_webhook_subscriptions`

---

## New Team B Additions Required To Close The CRM Gap

The following additions are part of Team B's contract closure work.

| Addition | Reason |
|---|---|
| `create_medication_adherence_log` | closes the current medication adherence persistence gap |
| `get_call_by_provider_sid` as an explicit accepted method | CRM already uses this as a callback hot path |
| `list_active_webhook_subscriptions` backed by a real Team B model | CRM already expects runtime subscription lookup |

---

## Method Semantics

### Strong Reads

The following methods are strong-read paths:

- `find_patient_by_phone`
- `get_current_consent_state`
- `get_call_by_provider_sid`
- `get_appointment` when used for reminder-state or reschedule correctness

### Idempotent Writes

The following methods must be retry-safe when called with the same logical request:

- `create_workflow`
- `create_conversation`
- `create_call`
- `create_operation`
- `create_artifact` where retries are expected
- `upsert_call_by_provider`
- `upsert_message_thread_by_provider`
- `release_practitioner_slot`
- `create_medication_adherence_log`

### Compare-And-Set / Optimistic Concurrency

The following methods must support explicit concurrency protection:

- `patch_workflow_status`
- `patch_conversation`
- `patch_appointment_reminder_state`
- `hold_practitioner_slot`
- `confirm_practitioner_slot`

### Append-Or-History Semantics

The following records must be append-friendly and not modeled as destructive rewrites:

- `consent_records`
- `appointment_reschedule_history`
- `call_events`
- `medication_adherence_logs`

---

## Scheduling Atomicity Rules

Team B formally accepts the CRM requirement that scheduling correctness must remain on the DB-service side.

| Operation | Required behavior |
|---|---|
| hold slot | only succeeds if the slot is currently available |
| confirm slot | only succeeds if the same conversation currently holds it |
| release slot | safe under retry |
| reschedule history append | append-only; CRM must not fetch-modify-write an array |

---

## Medication Workflow Contract Rule

Team B will not leave medication adherence persistence artifact-only.

The contract rule is:

- medication adherence workflows target canonical `patient_medications`
- structured outreach outcomes persist into `medication_adherence_logs`
- any canonical medication lifecycle update remains on `patient_medications`

This closes the current CRM-side no-op gap for medication adherence outcome persistence.

---

## Webhook Subscription Contract Rule

Team B accepts that active webhook subscription lookup must have a Team B-owned persistence home.

Contract rule:

- webhook subscription state is Team B-owned durable configuration
- CRM may list active subscriptions by event type for delivery fan-out
- webhook subscription storage is not left implicit or in-memory-only

---

## Explicitly Out Of Scope For This Contract Lock

The following are not treated as blocked CRM dependencies for the current Team B contract lock:

| Topic | Position |
|---|---|
| built-in CRM contexts and tasks | code-owned by CRM |
| transport protocol choice | Team B internal decision |
| telemedicine schema expansion | separate canonical domain; not required for the current non-telemedicine CRM contract closure |

If telemedicine-linked CRM workflows are brought into active scope, Team B must publish a separate canonical telemedicine schema contract before CRM depends on it.

---

## Bottom Line

Team B is not merely acknowledging CRM's requested contract.
Team B is locking the CRM-facing DB-service surface to the accepted methods and semantics in this document.

That means CRM can build against:

- stable method names
- stable error codes
- explicit concurrency rules
- explicit idempotency rules
- explicit mapping from CRM terms to Team B canonical entities
