# Organizational Health Management System  
## Organizational Foundation + User / Access / Workforce Schema

This document captures the tables, fields, purposes, and relationships agreed on so far for the **organizational foundation** and the **user/access/workforce domain** of the system.

---

# 1. Design Intent

This schema is designed for an **organizational health management system** that can support:

- organizations with a **single facility**
- organizations with **multiple branches/facilities**
- facility-specific departments
- organization-specific access control and staffing structures
- reusable facility-defined shifts and date-specific staff assignments

The current scope covered in this document includes:

- organizational structure
- users and access control
- sessions
- staff/workforce structure
- shifts and shift assignments
- audit logs
- notifications

---

# 2. Tables and Fields

## 2.1 `organizations`

Top-level tenant/customer entity using the platform.

| Field | Purpose |
|---|---|
| `id` | Unique identifier for the organization |
| `name` | Official name of the organization |
| `code` | Short unique code for the organization |
| `organization_type` | Type/category of organization (e.g. hospital group, clinic network, diagnostic provider) |
| `status` | Current state of the organization (e.g. active, inactive) |
| `created_at` | Timestamp when the organization record was created |
| `updated_at` | Timestamp when the organization record was last updated |

---

## 2.2 `facilities`

Represents a physical or operational branch/site under an organization.

| Field | Purpose |
|---|---|
| `id` | Unique identifier for the facility |
| `organization_id` | Organization that owns this facility |
| `parent_facility_id` | Optional reference to a parent facility for hierarchical facility structures |
| `name` | Name of the facility/branch |
| `code` | Short unique code for the facility |
| `facility_type` | Type of facility (e.g. hospital, clinic, lab, imaging center) |
| `address` | Physical address of the facility |
| `city` | City where the facility is located |
| `region` | Region/state/province of the facility |
| `country` | Country where the facility is located |
| `status` | Current operational status of the facility |
| `created_at` | Timestamp when the facility record was created |
| `updated_at` | Timestamp when the facility record was last updated |

---

## 2.3 `departments`

Operational units within a facility.

| Field | Purpose |
|---|---|
| `id` | Unique identifier for the department |
| `facility_id` | Facility this department belongs to |
| `name` | Name of the department |
| `code` | Short code for the department |
| `department_type` | Category/type of department (e.g. OPD, radiology, pharmacy, ICU) |
| `status` | Current status of the department |
| `created_at` | Timestamp when the department record was created |
| `updated_at` | Timestamp when the department record was last updated |

---

## 2.4 `users`

Authentication and identity table for accounts that can access the system.

| Field | Purpose |
|---|---|
| `id` | Unique identifier for the user |
| `organization_id` | Organization the user belongs to |
| `first_name` | User’s first name |
| `last_name` | User’s last name |
| `email` | User’s email address used for identification/login |
| `phone` | User’s phone number |
| `password_hash` | Securely hashed password |
| `account_type` | Type of account (e.g. HUMAN, SERVICE) |
| `status` | Account status (e.g. active, suspended, invited, disabled) |
| `last_login_at` | Timestamp of the user’s most recent login |
| `created_at` | Timestamp when the user record was created |
| `updated_at` | Timestamp when the user record was last updated |

> Note: We are using a **many-to-many user-role model** through `user_roles`, so there is **no `role_id` field on `users`**.

---

## 2.5 `roles`

Access control roles defined for an organization.

| Field | Purpose |
|---|---|
| `id` | Unique identifier for the role |
| `organization_id` | Organization that owns or defines the role |
| `name` | Human-readable role name |
| `code` | Short machine-friendly role code |
| `description` | Description of what the role represents |
| `is_system_role` | Indicates whether the role is a default/system-provided role |
| `created_at` | Timestamp when the role was created |
| `updated_at` | Timestamp when the role was last updated |

---

## 2.6 `permissions`

Feature-level permissions attached to roles.

| Field | Purpose |
|---|---|
| `id` | Unique identifier for the permission record |
| `role_id` | Role this permission belongs to |
| `feature` | Functional area or feature the permission applies to |
| `can_read` | Whether the role can read/view this feature |
| `can_create` | Whether the role can create records in this feature |
| `can_update` | Whether the role can update records in this feature |
| `can_delete` | Whether the role can delete records in this feature |
| `created_at` | Timestamp when the permission record was created |
| `updated_at` | Timestamp when the permission record was last updated |

---

## 2.7 `user_roles`

Join table linking users to one or more roles.

| Field | Purpose |
|---|---|
| `id` | Unique identifier for the user-role assignment |
| `user_id` | User receiving the role |
| `role_id` | Role assigned to the user |
| `assigned_at` | Timestamp when the role was assigned |
| `assigned_by` | User who assigned the role |

---

## 2.8 `user_facilities`

Defines which facilities a user can access.

| Field | Purpose |
|---|---|
| `id` | Unique identifier for the user-facility access record |
| `user_id` | User being granted facility access |
| `facility_id` | Facility the user can access |
| `access_type` | Type of access relationship (e.g. primary, secondary) |
| `assigned_at` | Timestamp when access was granted |
| `assigned_by` | User who granted the access |

---

## 2.9 `user_sessions`

Tracks active and historical login sessions for users.

| Field | Purpose |
|---|---|
| `id` | Unique identifier for the session |
| `user_id` | User this session belongs to |
| `session_token` | Session token identifier |
| `refresh_token` | Refresh token associated with the session |
| `ip_address` | IP address from which the session was created or used |
| `user_agent` | Client/device information for the session |
| `started_at` | Timestamp when the session began |
| `expires_at` | Timestamp when the session expires |
| `last_active_at` | Timestamp of the most recent session activity |
| `revoked_at` | Timestamp when the session was revoked, if applicable |
| `revoke_reason` | Reason the session was revoked |

---

## 2.10 `staff`

Employment/workforce profile table linked to a user account.

| Field | Purpose |
|---|---|
| `id` | Unique identifier for the staff record |
| `user_id` | User account associated with this staff profile |
| `organization_id` | Organization employing this staff member |
| `primary_facility_id` | Main facility where the staff member works |
| `primary_department_id` | Main department where the staff member works |
| `staff_number` | Organization-specific staff/employee number |
| `job_title` | Operational or HR title of the staff member |
| `employment_status` | Employment state (e.g. active, suspended, terminated, on_leave) |
| `hired_at` | Date/time the staff member was hired |
| `created_at` | Timestamp when the staff record was created |
| `updated_at` | Timestamp when the staff record was last updated |

---

## 2.11 `shifts`

Reusable facility-defined shift templates/definitions.

| Field | Purpose |
|---|---|
| `id` | Unique identifier for the shift definition |
| `facility_id` | Facility that owns the shift |
| `department_id` | Department the shift is associated with |
| `name` | Human-readable name of the shift |
| `start_time` | Shift start time (time-only, since this is a reusable definition) |
| `end_time` | Shift end time (time-only) |
| `shift_type` | Type of shift (e.g. day, night, on_call) |
| `created_by` | User who created the shift definition |
| `created_at` | Timestamp when the shift was created |
| `updated_at` | Timestamp when the shift was last updated |

> Note: `shifts` does **not** include a status field because it represents reusable facility-defined shifts, not one-time shift occurrences.

---

## 2.12 `shift_assignments`

Date-specific staff assignments to reusable shifts.

| Field | Purpose |
|---|---|
| `id` | Unique identifier for the shift assignment |
| `shift_id` | Shift definition being assigned |
| `staff_id` | Staff member assigned to the shift |
| `assignment_role` | Staff function within the shift (e.g. lead_doctor, support_nurse) |
| `assignment_date` | Date on which the assignment applies |
| `assigned_by` | User who made the assignment |
| `created_at` | Timestamp when the assignment was created |
| `updated_at` | Timestamp when the assignment was last updated |

> Recommended uniqueness constraint: `(shift_id, staff_id, assignment_date)`

---

## 2.13 `audit_logs`

Immutable trace records for security, compliance, and activity tracking.

| Field | Purpose |
|---|---|
| `id` | Unique identifier for the audit log record |
| `organization_id` | Organization context of the event |
| `facility_id` | Facility context of the event, if applicable |
| `event_type` | High-level category of the event (e.g. AUTH, DATA_ACCESS, DATA_CHANGE, EXPORT) |
| `action` | Specific action performed (e.g. CREATE, UPDATE, DELETE, VIEW, LOGIN) |
| `entity_type` | Type of domain entity affected (e.g. patient, invoice, user, staff) |
| `entity_id` | ID of the specific entity affected |
| `actor_type` | Type of actor responsible for the event (e.g. USER, SYSTEM, SERVICE, INTEGRATION) |
| `actor_id` | Identifier of the actor responsible for the event |
| `request_id` | Request correlation ID for tracing across services |
| `session_id` | Session associated with the event |
| `ip_address` | IP address where the event originated |
| `user_agent` | Client/device making the request |
| `source_system` | System or service that initiated the action |
| `status` | Outcome of the event (e.g. SUCCESS, FAILED, DENIED) |
| `old_value` | Previous state of the affected data, stored as JSON |
| `new_value` | New state of the affected data, stored as JSON |
| `changed_fields` | JSON representation of fields changed |
| `created_at` | Timestamp when the audit log was recorded |

---

## 2.14 `notifications`

User-targeted system notifications.

| Field | Purpose |
|---|---|
| `id` | Unique identifier for the notification |
| `organization_id` | Organization context of the notification |
| `facility_id` | Facility context of the notification, if applicable |
| `user_id` | User receiving the notification |
| `type` | Notification type/category |
| `title` | Short notification title |
| `message` | Full notification content |
| `status` | Notification state (e.g. unread, read, archived, failed) |
| `read_at` | Timestamp when the notification was read |
| `created_at` | Timestamp when the notification was created |

---

# 3. Relationship Summary

## Organizational Structure
- One `organization` can have many `facilities`
- One `facility` can have many `departments`
- One `facility` can optionally be the child of another `facility`

## Access Control
- One `organization` can define many `roles`
- One `role` can have many `permissions`
- One `user` can have many `roles` through `user_roles`
- One `user` can have access to many `facilities` through `user_facilities`

## User and Workforce
- One `organization` can have many `users`
- One `user` can have many `sessions`
- One `user` may map to one `staff` profile
- One `staff` member belongs to one organization and has a primary facility and department

## Scheduling
- One `facility` can define many `shifts`
- One `department` can have many `shifts`
- One `staff` member can have many `shift_assignments`
- One `shift` can have many `shift_assignments`

## Audit and Notifications
- `audit_logs` are scoped to organizations and optionally facilities
- `notifications` are scoped to organizations and optionally facilities, and are delivered to users

---

# 4. Mermaid ER Diagram

```mermaid
erDiagram
    ORGANIZATIONS {
        int id PK
        string name
        string code
        string organization_type
        string status
        datetime created_at
        datetime updated_at
    }

    FACILITIES {
        int id PK
        int organization_id FK
        int parent_facility_id FK
        string name
        string code
        string facility_type
        string address
        string city
        string region
        string country
        string status
        datetime created_at
        datetime updated_at
    }

    DEPARTMENTS {
        int id PK
        int facility_id FK
        string name
        string code
        string department_type
        string status
        datetime created_at
        datetime updated_at
    }

    USERS {
        int id PK
        int organization_id FK
        string first_name
        string last_name
        string email
        string phone
        string password_hash
        string account_type
        string status
        datetime last_login_at
        datetime created_at
        datetime updated_at
    }

    ROLES {
        int id PK
        int organization_id FK
        string name
        string code
        string description
        boolean is_system_role
        datetime created_at
        datetime updated_at
    }

    PERMISSIONS {
        int id PK
        int role_id FK
        string feature
        boolean can_read
        boolean can_create
        boolean can_update
        boolean can_delete
        datetime created_at
        datetime updated_at
    }

    USER_ROLES {
        int id PK
        int user_id FK
        int role_id FK
        datetime assigned_at
        int assigned_by FK
    }

    USER_FACILITIES {
        int id PK
        int user_id FK
        int facility_id FK
        string access_type
        datetime assigned_at
        int assigned_by FK
    }

    USER_SESSIONS {
        int id PK
        int user_id FK
        string session_token
        string refresh_token
        string ip_address
        string user_agent
        datetime started_at
        datetime expires_at
        datetime last_active_at
        datetime revoked_at
        string revoke_reason
    }

    STAFF {
        int id PK
        int user_id FK
        int organization_id FK
        int primary_facility_id FK
        int primary_department_id FK
        string staff_number
        string job_title
        string employment_status
        datetime hired_at
        datetime created_at
        datetime updated_at
    }

    SHIFTS {
        int id PK
        int facility_id FK
        int department_id FK
        string name
        time start_time
        time end_time
        string shift_type
        int created_by FK
        datetime created_at
        datetime updated_at
    }

    SHIFT_ASSIGNMENTS {
        int id PK
        int shift_id FK
        int staff_id FK
        string assignment_role
        date assignment_date
        int assigned_by FK
        datetime created_at
        datetime updated_at
    }

    AUDIT_LOGS {
        int id PK
        int organization_id FK
        int facility_id FK
        string event_type
        string action
        string entity_type
        int entity_id
        string actor_type
        int actor_id
        string request_id
        string session_id
        string ip_address
        string user_agent
        string source_system
        string status
        json old_value
        json new_value
        json changed_fields
        datetime created_at
    }

    NOTIFICATIONS {
        int id PK
        int organization_id FK
        int facility_id FK
        int user_id FK
        string type
        string title
        string message
        string status
        datetime read_at
        datetime created_at
    }

    ORGANIZATIONS ||--o{ FACILITIES : has
    FACILITIES ||--o{ FACILITIES : parent_of
    FACILITIES ||--o{ DEPARTMENTS : has

    ORGANIZATIONS ||--o{ USERS : has
    ORGANIZATIONS ||--o{ ROLES : defines
    ORGANIZATIONS ||--o{ STAFF : employs
    ORGANIZATIONS ||--o{ AUDIT_LOGS : owns
    ORGANIZATIONS ||--o{ NOTIFICATIONS : scopes

    ROLES ||--o{ PERMISSIONS : grants
    USERS ||--o{ USER_ROLES : assigned
    ROLES ||--o{ USER_ROLES : linked_to

    USERS ||--o{ USER_FACILITIES : accesses
    FACILITIES ||--o{ USER_FACILITIES : available_to

    USERS ||--o{ USER_SESSIONS : has
    USERS ||--|| STAFF : may_map_to

    FACILITIES ||--o{ STAFF : primary_facility
    DEPARTMENTS ||--o{ STAFF : primary_department

    FACILITIES ||--o{ SHIFTS : defines
    DEPARTMENTS ||--o{ SHIFTS : scoped_to
    STAFF ||--o{ SHIFT_ASSIGNMENTS : assigned_to
    SHIFTS ||--o{ SHIFT_ASSIGNMENTS : has

    USERS ||--o{ NOTIFICATIONS : receives
    FACILITIES ||--o{ NOTIFICATIONS : scopes

    FACILITIES ||--o{ AUDIT_LOGS : occurs_in