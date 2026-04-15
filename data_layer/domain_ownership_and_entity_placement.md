# Domain Ownership And Entity Placement

## Purpose

This document fixes the domain boundary problem before more CRM-driven schema is added.

It exists to answer four questions clearly:

1. which domains are first-class in the healthOS data model
2. which records are canonical business truth
3. which records are CRM execution truth
4. where duplication is forbidden

This is not the consolidated schema specification.
It is the placement and ownership decision record used to decide where new tables should go before they are added to `database_specification.md` or the domain docs.

---

## Core Rule

CRM must not duplicate canonical healthcare or business truth.

CRM may own:

- execution records
- workflow state
- conversation and call state
- operations
- generated artifacts
- campaign state
- workflow outputs such as handoffs or adherence logs

CRM must reference canonical records for:

- patients
- practitioners
- appointments
- practitioner availability
- medications
- consent truth
- telemedicine session truth

Short version:

- canonical truth lives in the healthcare/business domain that would still exist if CRM were turned off
- CRM truth lives in records that exist because CRM executed something

---

## Domain Map

| Domain | Owns | Notes |
|---|---|---|
| Organizational / Identity | organizations, facilities, departments, users, staff, practitioners | foundational actor and tenancy truth |
| Patient | patient identity, contacts, addresses, insurance, allergies, baseline profile | long-lived patient truth |
| Scheduling | appointments, appointment reschedule history, practitioner availability | pre-encounter scheduling truth |
| Medication | patient medications, medication status lifecycle, refill state, adherence logs | longitudinal therapy truth; not just prescribing events |
| Clinical | visits, consultations, diagnoses, lab orders/results, imaging orders/results, prescriptions, EHR records | encounter care truth |
| CRM Service | workflows, conversations, calls, message threads, operations, artifacts, custom form definitions, campaign state | durable CRM execution truth |
| Engagement Outputs | staff notifications, symptom reports, patient feedback, handoff requests | may remain inside CRM Service domain at first, but are called out because they are likely to grow |
| Telemedicine | telemedicine sessions, participants, session events, room or meeting references | virtual encounter session truth |
| Inventory | inventory units, items, stock, transactions | operational stock truth |
| Billing & Finance | invoices, payments, allocations, refunds, claims | financial truth |
| System | audit logs, cross-cutting notifications | platform-wide support records |

---

## Ownership Rules

### 1. Canonical records

Canonical records represent real business or clinical truth independent of CRM execution.

Examples:

- `patients`
- `appointments`
- `practitioner_availability`
- `patient_medications`
- `telemedicine_sessions`

Rule:

- only one canonical owner
- CRM may reference these records
- CRM must not create a second authoritative master copy

### 2. Canonical events

Canonical events represent durable business changes in a canonical domain.

Examples:

- `prescriptions`
- `appointment_reschedule_history`
- `consent_records`
- medication discontinuation or status changes

Rule:

- keep event history append-friendly where appropriate
- do not replace event history with flattened CRM-side status fields

### 3. CRM execution records

CRM execution records exist because CRM accepts and runs work.

Examples:

- `workflows`
- `conversations`
- `calls`
- `message_threads`
- `operations`
- campaign launch and child-execution tracking

Rule:

- CRM may own these records durably
- they must link back to canonical subjects rather than duplicate them

### 4. CRM output records

CRM output records are the durable results of CRM execution.

Examples:

- `artifacts`
- `staff_notifications`
- `symptom_reports`
- `patient_feedback`
- `handoff_requests`
- `medication_adherence_logs`

Rule:

- outputs must reference canonical subjects and execution records
- outputs must not silently become the only source of truth for the underlying business entity

---

## Anti-Duplication Rules

| Rule | Why |
|---|---|
| Do not create a CRM-owned mirror table for a canonical entity unless it is explicitly non-authoritative | avoids drift and sync debt |
| Do not use artifacts as the only durable home for operational outcomes that need queryable structure | artifacts are outputs, not a substitute for operational records |
| Do not model longitudinal medication truth only as raw prescriptions | prescriptions are encounter events, not full therapy lifecycle truth |
| Do not store slot availability only inside workflow or call state | concurrent scheduling correctness requires shared authoritative slot truth |
| Do not treat telemedicine session state as a call-status extension | session truth and outreach truth are different domains |
| Do not reconstruct current consent by scanning history rows on the CRM side | the DB boundary must expose current-state truth directly |

---

## Entity Placement Matrix

This table is the placement rule for CRM-facing or CRM-adjacent records.

| Entity / concept | Canonical owner domain | CRM role | Duplication allowed? | Placement decision |
|---|---|---|---|---|
| Patient | Patient | reference, snapshot, outreach subject | no | keep canonical in Patient domain |
| Practitioner | Organizational / Identity | reference for scheduling and routing | no | keep canonical in practitioner model |
| Appointment | Scheduling | reference and workflow subject | no | keep canonical in Scheduling |
| Appointment reschedule history | Scheduling | append through contract when CRM drives reschedule | no duplicate master | keep canonical in Scheduling |
| Practitioner availability | Scheduling | read and request hold/confirm | no | keep canonical in Scheduling |
| Prescription | Clinical | referenceable source event | no | keep canonical in Clinical |
| Patient medication / medication plan | Medication | workflow subject for adherence/refill | no | keep canonical in Medication |
| Medication adherence log | Medication or Engagement Outputs | create as workflow output linked to canonical medication | yes, as output only | start in Medication domain and link to CRM workflow |
| Consent record history | Patient / policy-support domain | read-only consumer | no | canonical append-only history outside CRM |
| Current consent state | Patient / policy-support domain | strong-read at execution time | no | canonical current-state projection outside CRM |
| Workflow | CRM Service | durable owner | yes, CRM-owned | place in CRM Service domain |
| Conversation | CRM Service | durable owner | yes, CRM-owned | place in CRM Service domain |
| Call | CRM Service | durable owner | yes, CRM-owned | place in CRM Service domain |
| Message thread | CRM Service | durable owner | yes, CRM-owned | place in CRM Service domain |
| Operation | CRM Service | durable owner | yes, CRM-owned | place in CRM Service domain |
| Artifact metadata | CRM Service | durable owner | yes, CRM-owned | place in CRM Service domain |
| Custom form definition | CRM Service support domain | authoritative owner for extraction extension path | yes, CRM-facing canonical within this bounded domain | place in CRM Service domain |
| Staff notification | Engagement Outputs | create as CRM output | yes, as output only | place in Engagement Outputs or CRM Service first |
| Symptom report | Engagement Outputs or Clinical-support boundary | create as CRM output linked to patient/workflow | yes, as output only | place in Engagement Outputs first |
| Patient feedback | Engagement Outputs | create as CRM output linked to visit or workflow | yes, as output only | place in Engagement Outputs first |
| Handoff request | Engagement Outputs | create as CRM output for staff follow-up | yes, as output only | make first-class, not artifact-only |
| Campaign / batch | CRM Service | durable owner | yes, CRM-owned | place in CRM Service domain |
| Telemedicine session | Telemedicine | reference for outreach, reminders, follow-up | no | canonical in Telemedicine |
| Telemedicine participants | Telemedicine | reference only | no | canonical in Telemedicine |
| Telemedicine session events | Telemedicine | reference only | no | canonical in Telemedicine |

---

## Cross-Domain Linkage Rules

| From | To | Rule |
|---|---|---|
| CRM workflow | patient | store `patient_id` reference only |
| CRM workflow | appointment | use `subject.resource_type = appointment` and `resource_id` |
| CRM workflow | patient medication | use `subject.resource_type = medication` and `resource_id` |
| CRM workflow | telemedicine session | use subject linkage when CRM is operating around a virtual visit |
| Conversation / call | workflow | link by `workflow_id` where a call is part of a workflow |
| Artifact | workflow / conversation / call | store originating execution references |
| Artifact | canonical subject | store subject references; do not duplicate master subject fields |
| Adherence log | patient medication | always reference canonical medication record |
| Handoff request | workflow / patient / optional appointment or medication | link to the triggering execution and subject |

---

## Immediate Schema Work Order

This is the recommended order for DB-side work after this placement decision.

| Order | Work | Why |
|---|---|---|
| 1 | add `consent_records` and current-consent semantics | active CRM runtime depends on it today |
| 2 | create `medication_schema.md` | highest-risk anti-duplication area because prescriptions already exist |
| 3 | create `crm_service_schema.md` | completed; CRM durable execution records now have an explicit home |
| 4 | create `scheduling_schema.md` or move scheduling tables into a dedicated file | completed; appointments and availability now have a dedicated schema document |
| 5 | create `engagement_outputs_schema.md` or include those tables in `crm_service_schema.md` initially | completed for now by including outputs in `crm_service_schema.md`; split later only if the output domain grows enough to justify it |
| 6 | create `telemedicine_schema.md` | next major canonical domain; should follow the canonical-vs-CRM split already established above |

---

## Review Standard

Any new CRM-adjacent table should be rejected during review if:

- it duplicates canonical business truth already owned elsewhere
- it stores a second authoritative copy of appointment, medication, patient, practitioner, consent, or telemedicine session state
- it hides first-class workflow outputs inside unstructured artifact payloads when queryable records are needed
- it blurs scheduling truth, clinical truth, and CRM execution truth into one table family

Short rule:

first choose the owner, then add the table.
