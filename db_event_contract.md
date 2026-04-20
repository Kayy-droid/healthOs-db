# DB-Originated Event Contract For CRM

## Purpose

This document closes the DB-eventing gap for CRM.

CRM depends on Team B-originated events to:

- cancel pending outreach
- update reminder timing
- stop work against deactivated subjects
- keep long-running workflow state consistent with canonical business truth

This document defines the Team B event catalog CRM may depend on.

---

## Delivery Position

Team B event publication for CRM follows these rules:

| Topic | Team B position |
|---|---|
| delivery guarantee | at-least-once |
| duplicate handling | CRM must dedupe by `event_id` |
| ordering | per-resource ordering should be preserved where operationally feasible; cross-resource ordering is not guaranteed |
| publication safety | event publication should be tied to durable state change, preferably through an outbox-style pattern |
| source of truth | the canonical Team B write remains authoritative; the event is notification of that change |

---

## Required Event Envelope

All Team B events published for CRM must include:

| Field | Required? | Notes |
|---|---|---|
| `event_id` | yes | globally unique and stable |
| `event_type` | yes | controlled event name |
| `occurred_at` | yes | UTC timestamp |
| `organization_id` | yes | mandatory scope key |
| `facility_id` | optional | include when the source entity is facility-scoped |
| `resource.type` | yes | canonical entity type |
| `resource.id` | yes | canonical entity identifier |
| `correlation_id` | optional | include when available from the originating request |
| `payload` | yes | event-specific fields only |

Example:

```json
{
  "event_id": "evt_001",
  "event_type": "appointment.rescheduled",
  "occurred_at": "2026-04-17T12:00:00Z",
  "organization_id": "org_001",
  "facility_id": "fac_001",
  "resource": {
    "type": "appointment",
    "id": "appt_001"
  },
  "correlation_id": "corr_001",
  "payload": {
    "previous_scheduled_for": "2026-04-20T09:00:00Z",
    "new_scheduled_for": "2026-04-21T11:30:00Z"
  }
}
```

---

## Event Catalog

### 1. `consent.revoked`

Trigger:

- current consent state changes from allowing at least one automated outreach channel to disallowing that channel

Source of truth:

- `patient_current_consent`

Resource:

- `patient`

Payload fields:

- `patient_id`
- optional `phone_number`
- `voice_outbound_allowed`
- `sms_outbound_allowed`
- `effective_at`

CRM expected reaction:

- cancel pending scheduled contact affected by the revoked channel

---

### 2. `appointment.cancelled`

Trigger:

- `appointments.status` transitions to `cancelled`

Source of truth:

- `appointments`

Resource:

- `appointment`

Payload fields:

- `patient_id`
- `scheduled_for`
- optional `cancelled_at`

CRM expected reaction:

- cancel associated reminder workflows

---

### 3. `appointment.rescheduled`

Trigger:

- a new `appointment_reschedule_history` row is appended and the canonical appointment schedule changes

Source of truth:

- `appointments`
- `appointment_reschedule_history`

Resource:

- `appointment`

Payload fields:

- `patient_id`
- `previous_scheduled_for`
- `new_scheduled_for`
- optional `reason`

CRM expected reaction:

- update reminder timers and schedule-facing workflow context

---

### 4. `appointment.completed`

Trigger:

- `appointments.status` transitions to `completed`

Source of truth:

- `appointments`

Resource:

- `appointment`

Payload fields:

- `patient_id`
- `completed_at`

CRM expected reaction:

- mark appointment-reminder work as no longer actionable
- allow downstream post-visit commands to be issued by the calling system

---

### 5. `patient.deactivated`

Trigger:

- patient state transitions to a non-active state that should stop outreach

Source of truth:

- `patients`

Resource:

- `patient`

Payload fields:

- `patient_id`
- `new_status`

CRM expected reaction:

- cancel pending work for the patient

---

### 6. `medication.discontinued`

Trigger:

- `patient_medications.status` transitions to `discontinued`

Source of truth:

- `patient_medications`
- optional `patient_medication_status_history`

Resource:

- `patient_medication`

Payload fields:

- `patient_id`
- `patient_medication_id`
- `previous_status`
- `new_status`
- `effective_at`

CRM expected reaction:

- cancel or suppress medication adherence outreach tied to that medication

---

## Event Naming Rule

Team B will use:

- `<entity>.<business_change>`

Examples:

- `appointment.rescheduled`
- `patient.deactivated`
- `medication.discontinued`

This avoids CRM needing DB-internal table names.

---

## Non-Events

The following are intentionally not part of the CRM dependency contract:

| Topic | Position |
|---|---|
| `appointment.created` | not required by CRM for workflow initiation |
| `patient.created` | not required by CRM for workflow initiation |
| generic row update events | not accepted as a substitute for business events |

CRM workflow initiation remains command-driven, not generic-change-event-driven.

---

## Bottom Line

This document closes the Team B eventing gap for CRM by defining:

- the exact required event catalog
- the event envelope
- the trigger conditions
- the delivery guarantees CRM may depend on

