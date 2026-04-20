# CRM DB Contract Acknowledgement

## Purpose

This document is the DB-side response to the CRM service contract documents.

It exists to do four things clearly:

1. acknowledge what CRM says it needs from the DB side
2. distinguish schema decisions from service-contract decisions
3. state what the current `healthOs-db` design can support cleanly
4. state what is not yet agreed, not yet modeled, or should not be assumed

This is not a copy of the CRM documents.
It is the DB-side interpretation and response so Team B and CRM do not proceed on silent assumptions.

---

## Source Documents Reviewed

This acknowledgement is based primarily on the following CRM documents:

- `crm-service/docs/contracts/team_b_db_handoff.md`
- `crm-service/docs/contracts/db_requirements.md`
- `crm-service/docs/contracts/db_service_contract.md`
- `crm-service/docs/contracts/db_service_sla_and_degraded_mode.md`
- `crm-service/docs/contracts/upstream_contact_policy_contract.md`
- `crm-service/docs/contracts/api_versioning_policy.md`

It is also informed by the DB-side schema work in:

- `healthOs-db/database_specification.md`
- `healthOs-db/data_layer/domain_ownership_and_entity_placement.md`
- `healthOs-db/data_layer/clinical_schema.md`
- `healthOs-db/data_layer/medication_schema.md`
- `healthOs-db/data_layer/scheduling_schema.md`
- `healthOs-db/data_layer/crm_service_schema.md`
- `healthOs-db/db_service_contract_response.md`
- `healthOs-db/db_event_contract.md`
- `healthOs-db/crm_controlled_enum_registry.md`

---

## First Principle

CRM and DB are talking about two different things that must not be mixed together.

| Layer | What it means |
|---|---|
| Data/schema layer | tables, fields, relationships, ownership boundaries |
| Service-contract layer | request metadata, idempotency rules, versioning, compare-and-set behavior, event publication, latency expectations |

This matters because some CRM requirements are valid contract requirements but are not supposed to appear as database entity columns by default.

Examples:

- `correlation_id`
- `causation_id`
- `trace_id`
- `idempotency_key`

These are primarily service-envelope and operational-control fields.
They are not automatically canonical table fields on every entity.

---

## Naming Alignment: `organization_id` (Agreed)

Status: RESOLVED — 2026-04-15

CRM has agreed to adopt `organization_id` as the canonical top-level ownership and isolation key across all service contracts, DTOs, and request metadata envelopes.

The `tenant_id` name is retired from the CRM-DB interface. Both sides will use `organization_id`.

### Agreed naming

| Scope key | Agreed name | Notes |
|---|---|---|
| top-level ownership / isolation | `organization_id` | used in all CRM DTOs, request envelopes, and DB schema |
| facility scope within the organization | `facility_id` | unchanged; both sides already aligned |

### What this means going forward

- CRM must update all DTOs, request envelope definitions, and contract documents to use `organization_id` in place of `tenant_id`
- DB schema remains as-is; no migration required
- Any future multi-tenant architecture that introduces a true distinction between tenant and organization will require an explicit versioned contract change on both sides

### What has been updated

- CRM-facing request metadata envelope now uses `organization_id`
- The "Required Clarifications CRM Should Receive Back" and "Where CRM Should Adjust Expectations" sections below reflect this as closed

---

## Executive Position

The current DB-side design can support CRM cleanly, but only if the following boundary is made explicit:

| Category | DB-side position |
|---|---|
| Canonical business entities | supported and DB-owned |
| CRM execution entities | supported and DB-backed through CRM service schema |
| Request metadata | should be treated as service-envelope metadata, not universal entity columns |
| Idempotency | required for selected operations, but must be defined at the service-contract layer |
| Record versioning | required and now written into schema docs: `version` field added to all mutable CRM-facing records |
| Compare-and-set concurrency | required for scheduling and selected execution-state updates |
| Event publication | accepted | DB-side event contract now documented in `db_event_contract.md` |

Short version:

- CRM is right to ask for these things.
- DB is right not to treat all of them as table fields.
- the missing piece is an explicit DB-side contract document, not more schema guessing.

---

## Decision Statuses Used In This Review

| Status | Meaning |
|---|---|
| `accepted` | DB side agrees with the requirement as stated in principle |
| `accepted with mapping` | DB side agrees, but CRM naming or DTO shape must map to current DB naming or structure |
| `accepted with refinement` | DB side agrees with the need, but the exact contract shape still needs tightening |
| `pending clarification` | the request is directionally valid, but important semantics are still underspecified |
| `not agreed` | CRM should not assume this is accepted in the current DB contract |

---

## Senior Review Summary

| Area | Review result | Reason |
|---|---|---|
| ownership model | `accepted` | CRM and DB align that DB owns durable truth and CRM owns execution state |
| canonical entities | `accepted with mapping` | mostly aligned, but some CRM names do not match DB naming exactly |
| request metadata envelope | `accepted with refinement` | valid contract metadata, but not default table columns |
| versioning and optimistic concurrency | `accepted` | `version` field now present on all mutable CRM-facing records; compare-and-set operations defined per entity |
| idempotency | `accepted with refinement` | required, but replay semantics still need explicit contract wording |
| scheduling atomicity | `accepted` | required and already aligned with DB scheduling design |
| custom form ownership | `accepted` | DB owns tenant-authored custom forms only |
| DB-originated events | `accepted` | CRM is right to ask, and DB-side event contract is now written in `db_event_contract.md` |
| controlled enum sets | `accepted` | the CRM-facing V1 controlled values are now ratified in `crm_controlled_enum_registry.md` |
| diagnosis outcome persistence | `accepted with refinement` | artifact-only is weak if the app needs direct querying |

---

## Entity And Naming Mapping

| CRM term | DB-side term or home | Review status | Notes |
|---|---|---|---|
| `Tenant` | `organizations` | `accepted` | naming resolved 2026-04-15: both sides use `organization_id` |
| `Facility` | `facilities` | `accepted` | names align |
| `User` | `users` and in some cases `staff` | `accepted with mapping` | user identity and workforce records should not be blurred |
| `Patient` | `patients` | `accepted` | names align |
| `Practitioner` | `practitioners` | `accepted` | names align |
| `Appointment` | `appointments` | `accepted` | canonical scheduling entity |
| `PractitionerAvailability` | `practitioner_availability` | `accepted` | canonical scheduling entity |
| `Medication` | `patient_medications` as the canonical workflow target | `accepted with mapping` | CRM should not treat raw prescriptions as the main medication target |
| `ConsentRecord` | `consent_records` plus `patient_current_consent` | `accepted with mapping` | history and current-state split is intentional |
| `Workflow` | `workflows` | `accepted` | CRM execution record |
| `Conversation` | `conversations` | `accepted` | CRM execution record |
| `Call` | `calls` | `accepted` | CRM execution record |
| `Message Thread` | `message_threads` | `accepted` | CRM execution record |
| `Operation` | `operations` | `accepted` | CRM execution record |
| `Artifact` | `artifacts` and `artifact_exports` | `accepted` | output metadata record |
| `CustomFormDefinition` | `custom_form_definitions` | `accepted` | DB-owned extension point |
| `StaffNotification` | `staff_notifications` | `accepted` | structured output |
| `PatientFeedback` | `patient_feedback` | `accepted` | structured output |
| `SymptomReport` | `symptom_reports` | `accepted with refinement` | may need stronger outcome semantics if diagnosis becomes a major app-facing surface |
| `WorkflowCampaign` | `workflow_campaigns` | `accepted` | CRM execution grouping record |

---

## What CRM Says It Needs

The CRM documents ask for the following classes of support.

| Requirement class | CRM asks for | DB-side reading |
|---|---|---|
| Canonical entity access | patient, practitioner, appointment, medication, consent, availability | valid and aligned |
| CRM durable records | workflow, conversation, call, operation, artifact, outputs | valid and aligned |
| Request metadata | tenant, facility, correlation, causation, trace, idempotency | valid as service metadata |
| Version metadata | version or ETag on mutable records | valid and should be defined explicitly |
| Safe mutation semantics | idempotent create/patch, compare-and-set where needed | valid and required for correctness |
| Scheduling correctness | atomic hold, confirm, release | valid and required |
| Custom forms | versioned published custom form lookup | valid and already aligned with current DB direction |
| Eventing | DB-originated events for visibility and cancellation/state propagation | valid and now documented on DB side |
| SLA/degraded mode | strong-read paths fail closed where correctness matters | valid and aligned in principle |

---

## CRM Method Surface Review

| CRM-facing method family | Review status | DB-side position |
|---|---|---|
| `get_tenant`, `get_facility`, `get_user` | `accepted with mapping` | valid, but `get_tenant` currently maps to organization-level ownership |
| `get_patient`, `find_patient_by_phone`, `get_current_consent_state` | `accepted` | valid hot-path reads; consent remains a dedicated model |
| `get_appointment`, `list_upcoming_patient_appointments`, `patch_appointment_reminder_state`, `append_appointment_reschedule_history` | `accepted` | reminder-state and append semantics are now locked in `db_service_contract_response.md` |
| `get_medication`, `list_active_patient_medications` | `accepted with mapping` | valid if medication means canonical patient medication, not prescription rows |
| `create_medication_adherence_log` | `accepted` | closes the medication adherence persistence gap against canonical `patient_medications` |
| `create_workflow`, `get_workflow`, `list_workflows`, `patch_workflow_status` | `accepted with refinement` | valid, but versioning and compare-and-set semantics must be explicit |
| `create_conversation`, `patch_conversation`, `append_conversation_context_switch` | `accepted` | patch and version semantics are now documented in `db_service_contract_response.md` |
| `create_call`, `get_call`, `get_call_by_provider_sid`, `upsert_call_by_provider`, `patch_call_status`, `upsert_message_thread_by_provider`, `patch_message_status` | `accepted` | provider callback correlation and provider-key semantics are now explicitly accepted |
| `create_operation`, `patch_operation`, `create_artifact`, `get_artifact`, `create_artifact_export` | `accepted with refinement` | valid, but artifact metadata vs payload boundary must stay clear |
| `list_practitioner_availability`, `hold_practitioner_slot`, `confirm_practitioner_slot`, `release_practitioner_slot` | `accepted` | valid and required; scheduling remains authoritative |
| `create_staff_notification`, `create_symptom_report`, `create_patient_feedback` | `accepted` | valid structured output writes |
| `get_custom_form_definition` | `accepted` | valid DB-owned extension point |
| `list_active_webhook_subscriptions` | `accepted` | Team B now gives active subscription lookup a durable home |

---

## What The Current DB Design Can Already Support

The schema work already completed supports the following CRM needs.

| CRM need | DB-side status | Where it lives |
|---|---|---|
| canonical patient data | supported | clinical/patient schema |
| canonical practitioner data | supported | database specification practitioner model |
| canonical appointment data | supported | scheduling schema |
| canonical slot inventory | supported | scheduling schema |
| canonical medication target for adherence | supported | medication schema |
| canonical consent history and current state | supported | database specification consent tables |
| CRM workflow/call/conversation/artifact records | supported | CRM service schema |
| structured feedback/handoff/notification outputs | supported | CRM service schema |
| diagnosis/triage output persistence | partially supported through `symptom_reports` and artifacts | CRM service schema |

Important note:

"supported" here means the ownership and data model are defined.
For the CRM-facing DB-service method surface and event surface, the remaining contract semantics are now closed by:

- `db_service_contract_response.md`
- `db_event_contract.md`
- `crm_controlled_enum_registry.md`

---

## What CRM Must Not Assume Without Explicit Agreement

CRM should not assume the following are already approved just because they appear in CRM docs.

| Assumption | Current DB-side status |
|---|---|
| every request metadata field is also a persistent column on major records | not agreed |
| every mutable record already exposes `version` in DB-side docs | now documented: `version` added to all mutable CRM-facing records |
| compare-and-set semantics are already approved for all mutable entities | approved for workflow, conversation, call, appointment, and patient_medication; must still be declared operation by operation in the DB service contract |
| event publication semantics are already finalized | now documented in `db_event_contract.md` |
| CRM enum values are automatically accepted as DB-approved controlled sets | not yet fully ratified on DB side |

That does not mean these are rejected.
It means they must be explicitly defined rather than silently inherited from CRM assumptions.

---

## Shared Request Metadata: DB-Side Interpretation

CRM uses a shared metadata envelope on DB calls.
That is a reasonable pattern.

The DB-side interpretation should be:

| Field | Meaning | Required? | DB-side position |
|---|---|---|---|
| `organization_id` | the organization whose data is being acted on; agreed canonical name as of 2026-04-15 | yes | required contract field; organization scoping is mandatory on all requests |
| `facility_id` | facility context when the action is facility-scoped | optional | valid optional contract field |
| `correlation_id` | the end-to-end request or workflow correlation key used to tie related actions together across services | yes | valid service metadata; not a default entity column |
| `causation_id` | the immediate prior event, command, or action that caused this DB request to happen | optional | valid service metadata; not a default entity column |
| `trace_id` | distributed tracing identifier used by observability tooling to follow execution across service boundaries | optional | valid service metadata; not a default entity column |
| `idempotency_key` | caller-supplied retry token used to ensure the same logical create/patch request is not applied twice | optional for some methods, required for retry-safe creates/patches | valid service metadata; may need selective persistence |

### Plain-language definitions

#### `correlation_id`

This answers:

"Which larger action or request chain does this DB call belong to?"

Example:

- a frontend user triggers an appointment reminder launch
- CRM starts a workflow
- CRM creates a conversation
- CRM creates a call
- CRM writes an artifact

All of those operations may share the same `correlation_id` so operators can trace them as one business flow.

#### `causation_id`

This answers:

"What immediately caused this specific action?"

Example:

- webhook callback arrives
- callback handling causes `patch_call_status`
- `causation_id` may be the callback event ID or command ID that directly triggered the write

`correlation_id` ties the larger story together.
`causation_id` explains the direct trigger for this one step.

#### `trace_id`

This answers:

"What distributed trace is this work part of in observability systems?"

This is mostly for tracing systems and performance debugging.
It is useful operationally but is not usually part of canonical business truth.

#### `idempotency_key`

This answers:

"If the caller retries the same create or patch, how do we know not to do it twice?"

Example:

- CRM sends `create_workflow`
- network times out after DB applied it
- CRM retries
- same `idempotency_key` lets DB return the same logical result instead of creating a duplicate

### DB-side storage position for metadata

The default position should be:

| Field | Default treatment |
|---|---|
| `organization_id` | agreed canonical contract field; persists as `organization_id` on all scoped records |
| `facility_id` | persistent where the entity itself is facility-scoped |
| `correlation_id` | request metadata by default; persist selectively if needed for audit/operations |
| `causation_id` | request metadata by default; persist selectively if needed for audit/event traceability |
| `trace_id` | request metadata only by default |
| `idempotency_key` | persist only where required to provide retry-safe semantics |

This is the key clarification:

CRM should expect these fields in the DB service contract envelope.
CRM should not assume they are first-class columns on every major business table.

### When metadata probably does need persistence

| Field | Persist when | Why |
|---|---|---|
| `correlation_id` | workflow, operation, or audit traceability requires cross-record joinability | operator support and audit |
| `causation_id` | event-driven or callback-driven writes need direct trigger traceability | reconstructing write causality |
| `trace_id` | usually do not persist in business tables | observability concern, not business truth |
| `idempotency_key` | create or patch replay safety depends on it | dedupe and safe retries |

---

## Record Versioning: DB-Side Interpretation

CRM expects mutable records to return version metadata.
That is a sound requirement.

The DB-side interpretation should be:

| Field | Meaning |
|---|---|
| `version` | monotonic record version used to detect stale updates |
| `updated_at` | last modification timestamp for the record |

### Why CRM wants this

CRM uses long-running workflows, callbacks, and retries.
Without version checks, one stale worker can overwrite a newer state.

### DB-side position

| Question | Position |
|---|---|
| should mutable CRM-facing records expose a version? | yes |
| should every table in schema docs show a `version` column today? | not necessarily |
| should the service contract expose version metadata even if internal storage differs? | yes |

This means the DB service contract can expose:

```json
{
  "version": 12,
  "updated_at": "2026-04-07T10:30:00Z"
}
```

without forcing every schema note in `healthOs-db` to pretend its final physical storage format is already settled.

### DB-side review note

CRM is right to require version metadata.
DB side should stop leaving that as an implied future concern.
It should be part of the formal contract for all mutable CRM-facing records.

---

## Compare-And-Set And Concurrency: DB-Side Interpretation

CRM asks for optimistic concurrency and compare-and-set semantics.
That is not a luxury feature.
It is required where concurrent execution can corrupt truth.

### Where DB side agrees this is required

| Area | Why it matters |
|---|---|
| practitioner slot hold/confirm/release | prevents double booking |
| appointment reminder-state patching | prevents stale workflow updates |
| workflow status patching | prevents stale workers overwriting newer state |
| conversation state patching | prevents context/state drift during concurrent updates |
| selected call status updates | prevents callback/order races |

### What compare-and-set means in plain language

It means:

"Only apply this update if the record is still in the state or version I think it is."

Example:

- worker A reads workflow version 4
- worker B updates workflow to version 5
- worker A tries to patch using expected version 4
- DB rejects it with `concurrency_conflict`

That is correct behavior.

### Minimum DB-side concurrency commitment

At minimum, DB side should explicitly commit compare-and-set semantics for:

- workflow status updates
- conversation updates
- appointment reminder-state updates
- slot hold, confirm, and release operations

Call and message-thread updates may mix idempotency and version checks depending on provider callback behavior.

---

## Idempotency: DB-Side Interpretation

CRM asks for idempotent creates and selected patches.
That is valid.

### Where idempotency should be required

| Operation type | DB-side position |
|---|---|
| create workflow | must be idempotent |
| create conversation | should be idempotent |
| create operation | must be idempotent |
| create artifact | should be idempotent where retries are expected |
| provider-correlated upserts like call/message thread | must be idempotent by provider identity and/or request key |

### What idempotency means in plain language

It means:

"If the same logical request is sent twice because of a retry, the system should not create duplicate durable records."

### Important boundary

Idempotency is a service behavior.
It may require persistence internally, but it is not just a table-description concern.

### Senior review note

CRM is correct to ask for idempotency by default on retry-prone create and upsert paths.
DB side should not leave replay behavior vague.

---

## Controlled Enums And Allowed Values

CRM uses controlled values like `workflow_type`.
Those should be explicitly defined, not left as vague examples.

### DB-side position

| Field | Position |
|---|---|
| `workflow_type` | controlled set, not free text |
| `subject_type` | controlled set, not free text |
| `call_type` | controlled set, not free text |
| `notification_type` | controlled set, not free text |

### Current practical V1 workflow set

Based on CRM docs, the likely V1 workflow set is:

- `appointment_reminder`
- `medication_adherence`
- `care_feedback`
- `general_call`
- `diagnosis`
- `custom`

Important note:

The controlled-value list is now ratified in `crm_controlled_enum_registry.md`.
The DB-side interpretation is bounded:

- `diagnosis` is the only currently ratified diagnosis-family workflow type
- `custom` is accepted only for approved custom-form-driven workflows and campaigns
- `custom` does not imply arbitrary undefined runtime workflow behavior

---

## Where CRM Is Right

| CRM concern | Review |
|---|---|
| strong consent reads before outreach | correct |
| phone lookup as hot-path read | correct |
| atomic scheduling operations | correct |
| versioning on mutable records | correct |
| idempotent create/update behavior | correct |
| stable custom-form version lookup | correct |
| structured machine-readable errors | correct |
| separation of runtime-owned vs DB-owned state | correct |

---

## Where CRM Is Over-Assuming

| CRM assumption | DB-side review |
|---|---|
| request metadata should appear as normal entity columns | too strong |
| contract DTO names should equal DB schema column names | too strong |
| entity names like `Tenant` and `Medication` already map one-to-one to DB tables | too strong |
| DB event publication details are already settled | too strong |
| enum values listed in CRM docs are automatically approved by DB without explicit bounded ratification | too strong |

---

## Custom Form Support

CRM expects the DB side to own tenant-authored custom form definitions.
This is aligned with the current DB-side direction.

### DB-side position

| Requirement | Position |
|---|---|
| latest published form lookup | accepted |
| exact version lookup | accepted |
| published form immutability | accepted |
| custom forms as declarative definitions only | accepted |
| built-in runtime definitions resolved from DB | rejected |

This boundary is important:

- DB owns custom form definitions
- CRM code owns built-in contexts, tasks, and built-in artifact schemas

---

## Event Publication

CRM expects DB-originated events.
That expectation is visible in the handoff.

### DB-side position

This requirement is accepted.
The DB-side event contract is now defined in `db_event_contract.md`.

That document closes the following points:

- which exact CRM-required events Team B will publish
- what the event envelope shape is
- what the trigger conditions are
- the required delivery guarantee

CRM should build against that event contract rather than infer eventing behavior from schema docs alone.

---

## Entity-Level Acknowledgement

This section states what CRM asked for and how DB side currently responds.

| CRM entity/need | DB-side response |
|---|---|
| Patient | accepted as canonical DB-owned entity |
| Practitioner | accepted as canonical DB-owned entity |
| Appointment | accepted as canonical DB-owned scheduling entity |
| PractitionerAvailability | accepted as canonical scheduling entity |
| Medication | accepted as canonical medication-domain entity |
| ConsentRecord / current consent | accepted as canonical policy-support entity |
| Workflow | accepted as CRM execution record stored durably |
| Conversation | accepted as CRM execution record stored durably |
| Call | accepted as CRM execution record stored durably |
| Message Thread | accepted as CRM execution record stored durably |
| Operation | accepted as CRM execution record stored durably |
| Artifact | accepted as CRM output metadata record |
| CustomFormDefinition | accepted as DB-owned extension definition |
| StaffNotification | accepted as structured CRM output |
| PatientFeedback | accepted as structured CRM output |
| SymptomReport | accepted as structured CRM output; may need strengthening if diagnosis outcome needs richer direct querying |
| WorkflowCampaign | accepted as CRM execution grouping record |

---

## Required Clarifications CRM Should Receive Back

| Topic | DB-side clarification |
|---|---|
| `organization_id` | agreed canonical name; resolved 2026-04-15 |
| `Medication` | maps to canonical patient medication records for workflow targeting |
| metadata fields | are contract-envelope fields first, not default table columns |
| `version` | will be a contract concern for mutable records even where not shown yet in schema docs |
| compare-and-set | supported on the correctness-critical mutation paths declared in `db_service_contract_response.md` |
| event publication | now explicitly defined in `db_event_contract.md` |
| diagnosis outputs | should not remain artifact-only if app/frontend needs direct querying |

---

## Where CRM Should Adjust Expectations

CRM should update its understanding in the following places.

| Topic | Clarification |
|---|---|
| request metadata | valid contract metadata, not automatically schema columns |
| version metadata | now written into DB schema docs: `version` field on all mutable CRM-facing records |
| concurrency | accepted in principle, but must be declared per operation |
| enum values | must be ratified as controlled sets, not inferred from examples |
| diagnosis output | should not be treated as artifact-only if other app surfaces need direct query access |
| consent linkage | consent is separate from `patient_contacts`, but may link to them |
| scheduling truth | CRM must treat scheduling as authoritative shared state |
| medication truth | CRM must use canonical `patient_medications`, not duplicate masters |
| naming | resolved: both sides use `organization_id` as of 2026-04-15 |

---

## Non-Assumptions

The following should be treated as explicitly not yet agreed unless later documented.

| Topic | Non-assumption |
|---|---|
| metadata persistence | `correlation_id`, `causation_id`, and `trace_id` are not assumed to be columns on every entity |
| universal record version column | now present on all mutable CRM-facing records as of 2026-04-15 |
| full event catalog | now defined in `db_event_contract.md` for the CRM-required event set |
| exact DB-service error catalog | now ratified in `db_service_contract_response.md` |
| exact enum completeness | now ratified for the CRM-facing V1 set in `crm_controlled_enum_registry.md` |
| final diagnosis-outcome model | not yet fully settled |
| one-to-one name parity | CRM names are not assumed to equal DB table names |

---

## Recommended Next DB-Side Documents

To fully close the gap with CRM, DB side should next produce:

| Document | Why |
|---|---|
| DB service contract response | completed in `db_service_contract_response.md` |
| DB-originated event contract | completed in `db_event_contract.md` |
| controlled enum registry for CRM-facing values | completed in `crm_controlled_enum_registry.md` |
| telemedicine schema | finish the canonical telemedicine side that CRM may reference |

---

## Bottom Line

CRM is asking for valid things.
The current DB schema work covers most ownership and entity placement questions.

The remaining gap is contract clarity.

The key DB-side message is:

- yes, many of CRM's requirements are acknowledged
- no, not all of them belong in table definitions
- request metadata, idempotency, versioning, compare-and-set, and eventing are now defined through the DB-side contract documents listed above
- CRM should build against those DB-side contract documents rather than infer semantics from schema docs alone
