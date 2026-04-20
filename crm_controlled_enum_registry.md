# CRM Controlled Enum Registry

## Purpose

This document ratifies the controlled CRM-facing values Team B accepts in the DB-service contract.

These values are contract values, not vague examples.

---

## `workflow_type`

Accepted values:

- `appointment_reminder`
- `medication_adherence`
- `care_feedback`
- `general_call`
- `diagnosis`
- `custom`

Contract bounds:

- `diagnosis` is the only currently ratified diagnosis-family workflow type for the current non-telemedicine CRM V1 contract
- `custom` is accepted only for approved custom-form-driven workflows, not arbitrary undefined runtime behavior

---

## `campaign_type`

Accepted values:

- `appointment_reminders`
- `medication_adherence_batch`
- `care_feedback_batch`
- `general_call_batch`
- `diagnosis_batch`
- `custom`

Contract bounds:

- `custom` is accepted only for approved custom-form-driven campaigns
- `custom` does not imply arbitrary undefined campaign logic

---

## `workflow_subject_type`

Accepted values:

- `patient`
- `appointment`
- `patient_medication`

Deferred until telemedicine schema is contracted:

- `telemedicine_session`

---

## `call_status`

Accepted values:

- `initiated`
- `queued`
- `ringing`
- `in_progress`
- `completed`
- `failed`
- `cancelled`
- `no_answer`
- `busy`

---

## `conversation_status`

Accepted values:

- `open`
- `active`
- `completed`
- `abandoned`
- `cancelled`
- `failed`

---

## `operation_status`

Accepted values:

- `accepted`
- `pending`
- `running`
- `completed`
- `failed`
- `cancelled`

---

## `campaign_status`

Accepted values:

- `draft`
- `pending`
- `running`
- `paused`
- `completed`
- `cancelled`
- `partially_failed`

---

## `notification_priority`

Accepted values:

- `low`
- `medium`
- `high`
- `urgent`

---

## `notification_status`

Accepted values:

- `open`
- `acknowledged`
- `resolved`
- `dismissed`

---

## `symptom_report_status`

Accepted values:

- `open`
- `reviewed`
- `resolved`

---

## `patient_feedback_status`

Accepted values:

- `new`
- `reviewed`
- `closed`

---

## `handoff_status`

Accepted values:

- `open`
- `accepted`
- `resolved`
- `cancelled`

---

## `practitioner_availability_status`

Accepted values:

- `available`
- `held`
- `booked`
- `blocked`
- `expired`

---

## `patient_medication_status`

Accepted values:

- `active`
- `paused`
- `completed`
- `discontinued`
- `expired`

---

## `medication_adherence_outcome`

Accepted values:

- `adherent`
- `non_adherent`
- `partially_adherent`
- `handoff_required`
- `unresolved`

---

## Bottom Line

If a value is not listed in this registry, it is not yet ratified as part of the Team B CRM-facing contract.
