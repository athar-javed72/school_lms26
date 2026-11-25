# Deliverable 5: Complete RBAC Design and Permission Matrix

**Version:** 1.0 (draft for approval) | **Date:** 23 Nov 2026
**Builds on:** D1 (role slugs, permission naming), D2 (permission keys referenced in requirements, DEC-21 visibility levels), D3 (PermissionResolver, policies, registries), D4 (scoping layers, DEC-46, DEC-47)

---

## 5.1 New Decisions in This Deliverable

| ID | Decision |
|---|---|
| DEC-53 | Permission actions use the standard vocabulary (`view`, `create`, `update`, `delete`, `approve`, `reject`, `export`, `import`, `print`, `assign`, `publish`, `manage`, `restore`) plus module verbs registered in the catalog (`generate`, `mark`, `submit`, `finalize`, `lock`, `unlock`, `convert`, `issue`, `verify`, `reverse`, `process`, `override_window`, `update_protected`, `view_full`). No verb is used unless it is in the catalog |
| DEC-54 | To act on an approval step a user must be resolved as an approver by the workflow **and** hold the request type's `approve` (or `reject`) permission in the request's branch. Misconfigured workflows cannot grant power the role does not have |
| DEC-55 | **No privilege escalation:** a user can assign a role, or grant an override, only if every permission involved is within their own effective permissions for that scope, and only inside their own scope. `org_owner` is exempt |
| DEC-56 | **Separation of duties** rules (5.11) are enforced in Actions and approval handlers, regardless of permissions |
| DEC-57 | **Baseline self-service abilities** (own profile, password, sessions, two-factor, notification preferences, own notifications, own requests, own trusted devices) belong to every authenticated user and are not permissions; they cannot be removed by role editing |
| DEC-58 | Effective visibility is capped by the assignment scope: a branch-scoped assignment never yields `institution` or `organization` visibility |
| DEC-59 | Permissions marked **org-scope only** (†) can be exercised only through an organization-scope assignment or override (configuration that affects every branch) |
| DEC-60 | Each organization's system roles are copies of versioned templates. When a release adds permissions, they are granted automatically to roles still matching their template, and offered as suggestions on customized roles |
| DEC-61 | Platform users have their own role set (`portal = platform`), stored with `organization_id = NULL`; tenant roles can never grant platform permissions and vice versa |
| DEC-62 | Sensitive identifiers (CNIC, B-Form) are masked in UI and exports unless the user holds `core.pii.view_full`; other field-level restrictions in 5.10 |
| DEC-63 | Any user holding a sensitive permission (marked S) must use two-factor login, unless the organization explicitly turns this off with an audited setting change |
| DEC-64 | `approve` and `reject` are separate keys; seeded templates grant them together |

---

## 5.2 Data Model

| Table | Key columns | Notes |
|---|---|---|
| `permissions` (global) | `id`, `key` (unique), `module`, `resource`, `action`, `label_en`, `label_ur`, `is_sensitive`, `allowed_scope_types` (JSON: organization, institution, branch), `portal` (`app`, `guardian`, `student`, `platform`), `release`, `default_template_roles` (JSON) | Synced from code by `identity:sync-permissions` |
| `roles` | `id`, `organization_id` (NULL for platform roles), `slug`, `name_en`, `name_ur`, `portal`, `template_slug` (nullable), `template_version`, `is_customized`, `is_locked`, `default_visibility`, `requires_two_factor`, `description` | Unique `(organization_id, slug)` |
| `role_permissions` | `organization_id`, `role_id`, `permission_id`, `visibility` (nullable; overrides role default for this permission) | Unique `(role_id, permission_id)` |
| `user_role_assignments` | `organization_id`, `user_id`, `role_id`, `scope_type` (`organization`, `institution`, `branch`), `scope_id` (NULL for organization), `visibility` (nullable), `starts_on`, `ends_on`, `assigned_by`, `reason` | One row per scope; a principal of two branches has two rows |
| `user_permission_overrides` | `organization_id`, `user_id`, `permission_id`, `effect` (`grant`, `deny`), `scope_type`, `scope_id`, `visibility` (grants only), `reason` (required), `expires_at` (nullable), `granted_by` | Deny wins |

---

## 5.3 Effective Permission Algorithm

```
can(user, key, branchId = null, record = null):
  1. portal(key) must equal the current portal                     -> else false
  2. module of key must be entitled for the organization           -> else false
  3. denies  = active overrides(user, key, effect=deny) whose scope covers branchId
     if denies not empty                                           -> false
  4. grants  = active assignments(user, today) whose scope covers branchId
                 and whose role has key
             + active overrides(user, key, effect=grant) whose scope covers branchId
     if key is org-scope only (†): keep only organization-scope grants
     if grants empty                                               -> false
  5. visibility = widest of:
        role_permissions.visibility ?? assignment.visibility ?? role.default_visibility
        (overrides: override.visibility)
     each capped by its own scope (DEC-58)
  6. if record is given: return recordWithinVisibility(record, visibility, user)
  7. return true

scope covers branchId:
  organization -> any branch of the organization (and branchId = null)
  institution  -> branches of that institution
  branch       -> that branch only
  branchId = null (organization-level resource) -> only organization-scope grants,
                                                    or institution-scope for institution-owned resources
```

**Visibility check for a record (`recordWithinVisibility`)**

| Visibility | Record passes when |
|---|---|
| `organization` | Record is in the user's organization |
| `institution` | Record's branch (or `institution_id`) is in an institution the grant covers |
| `branch` | Record's `branch_id` is a branch the grant covers |
| `assigned` | Record matches the user's assignment set for that resource (D4 4.10) |
| `own` | Record's subject or creator is the user (or, for guardians, a linked child; for students, themselves) |

**Caching:** compiled per user as `{branchId: {key: visibility}}`, stored at `org:{org}:perm:user:{user}:v{userVersion}.{orgRoleVersion}`. `userVersion` increments on any assignment or override change for that user; `orgRoleVersion` increments on any role permission change in the organization. TTL is the smaller of 1 hour and the time until the next expiry among the user's assignments and overrides, so expiring grants lapse on time.

---

## 5.4 Notation Used in the Matrices

**Roles (from D2)**

| Group | Members and default visibility |
|---|---|
| Management **MGT** | OW `organization`, OA `organization`, PR `branch`, VP `branch`, BM `branch` |
| Academic | HOD `assigned` (department), AC `branch`, CT `assigned`, ST `assigned`, EXC `branch` |
| Finance and HR | ACT `branch`, CSH `branch`, HRM `branch` |
| Operations | ADO `branch`, REC `branch`, LBR `branch`, TM `branch`, TS `assigned`, ITA `branch`, SUP `own` |
| Portals | STD `own`, GDN `own` (linked children) |
| Shorthand | **TCH** = CT + ST. **STAFF** = every staff role. **STAFF·o** = every staff user for their own records |

- A role listed in a cell uses its default visibility. A suffix overrides it: `·o` own, `·a` assigned, `·b` branch, `·i` institution, `·g` organization.
- **S** = sensitive permission (DEC-63). **†** = org-scope only (DEC-59). **R2/R3** = release.
- Columns combine actions with the same holders; the full key is `{module}.{resource}.{action}`.
- An empty cell means no role holds it by default (custom roles may).

---

## 5.5 System Role Templates

| Slug | Name | Portal | Default scope | Default visibility | 2FA default | Locked | Typical holder |
|---|---|---|---|---|---|---|---|
| `org_owner` | Organization Owner | app | organization | organization | Required | Yes (all permissions, cannot be edited or deleted) | Owner, chairman |
| `org_admin` | Organization Admin | app | organization | organization | Required | No | Head-office administrator |
| `principal` | Principal | app | branch | branch | Required | No | Branch principal |
| `vice_principal` | Vice Principal | app | branch | branch | Optional | No | |
| `branch_manager` | Branch Manager | app | branch | branch | Optional | No | Admin officer, campus manager |
| `hod` | Head of Department | app | branch | assigned | Optional | No | Science HOD |
| `academic_coordinator` | Academic Coordinator | app | branch | branch | Optional | No | Section or wing coordinator |
| `class_teacher` | Class Teacher | app | branch | assigned | Optional | No | |
| `subject_teacher` | Subject Teacher | app | branch | assigned | Optional | No | Lecturer |
| `accountant` | Accountant | app | branch (head office: organization) | branch | Required | No | |
| `cashier` | Cashier | app | branch | branch | Optional | No | Fee counter clerk |
| `hr_manager` | HR Manager | app | organization or branch | branch | Required | No | |
| `librarian` | Librarian | app | branch | branch | Optional | No | R3 |
| `exam_controller` | Examination Controller | app | branch (or institution) | branch | Optional | No | |
| `admission_officer` | Admission Officer | app | branch | branch | Optional | No | |
| `receptionist` | Receptionist | app | branch | branch | Optional | No | Front desk |
| `transport_manager` | Transport Manager | app | branch | branch | Optional | No | R2 |
| `transport_staff` | Transport Staff | app | branch | assigned | Optional | No | Driver, conductor (R3 boarding) |
| `it_admin` | IT Administrator | app | organization | branch | Required | No | |
| `support_staff` | Support Staff | app | branch | own | Optional | No | Peons, guards (attendance, leave only) |
| `student` | Student | student | branch | own | Optional | Yes (fixed set) | Students above login threshold |
| `guardian` | Parent / Guardian | guardian | organization | own | Optional | Yes (fixed set) | Parents, guardians |

**Platform roles (DEC-61):** `platform_super_admin` (all platform permissions), `platform_support` (tenant console read, support access requests, no plan or state changes), `platform_billing` (plans, subscriptions, usage; no support access).

**Custom roles:** created by OW or OA by cloning any template or from scratch (e.g., "Fee Clerk", "Counselor", "School Nurse", "Store Keeper"). Custom roles are never `is_locked`.

---

## 5.6 Portal Access

| Portal | Who can enter | Rule |
|---|---|---|
| `platform` | Platform users with a platform role | Platform domain only; two-factor mandatory |
| `app` | Users with at least one active staff role assignment | Tenant host |
| `guardian` | Users linked to at least one active, portal-enabled, unrestricted guardian record | Tenant host |
| `student` | Users linked to a student whose class is at or above the login threshold and whose status allows portal access (active, suspended read-only) | Tenant host |

A user with both staff assignments and guardian links sees a portal switcher (DEC-47). A permission belongs to exactly one portal (`permissions.portal`), so staff permissions never apply inside the guardian portal and the reverse.

---

## 5.7 Permission Matrices by Module

### 5.7.1 Platform (platform portal)

| Resource | View | Create / Update | Delete | Other actions |
|---|---|---|---|---|
| `platform.plan` | SA, billing | SA, billing | SA | |
| `platform.organization` | SA, support, billing | SA | | `manage` (state changes): SA |
| `platform.module_entitlement` | SA, billing | SA, billing | | |
| `platform.support_access` | SA, support | `create` (request): SA, support | | `impersonate`: SA, support (only with active grant) |
| `platform.announcement` | SA | SA | SA | `publish`: SA |
| `platform.usage` | SA, billing | | | `export`: SA, billing |
| `platform.support_access` (tenant side) | OW | | | `approve` (grant), `reject`: OW |

### 5.7.2 Organization

| Resource | View | Create / Update | Delete / Restore | Other actions |
|---|---|---|---|---|
| `organization.profile` † | MGT, ACT, HRM, ITA | `update`: OW, OA | | |
| `organization.institution` † | MGT, ITA | OW, OA | OW | |
| `organization.branch` † | MGT, ITA, HRM | OW, OA | OW | `deactivate` (registered as `manage`): OW |
| `organization.bank_account` | OW, OA, PR, ACT | OW, OA, ACT | OW, ACT | |
| `organization.department` | MGT, HRM, HOD | OW, OA, PR | OW, OA | |
| `organization.shift` | MGT, HRM, AC | OW, OA, PR, BM | OW, OA, PR | |
| `organization.room` | MGT, AC, EXC | OA, PR, BM | OA, PR | |
| `organization.onboarding` † | | | | `manage`: OW, OA |

### 5.7.3 Identity

| Resource | View | Create / Update | Delete | Other actions |
|---|---|---|---|---|
| `identity.user` | OW, OA, PR, BM, ITA, HRM | OW, OA, ITA, HRM (staff users only) | `delete` (deactivate): OW, OA, ITA | `export`, `import`: OW, OA, ITA. `manage` (unlock, reset): OA, ITA |
| `identity.role` S † | OW, OA, ITA | OW, OA | OW, OA | |
| `identity.role_assignment` S | OW, OA, PR, ITA | | | `assign` (assign and revoke): OW, OA, PR (branch roles below principal, DEC-55) |
| `identity.permission_override` S | OW, OA, ITA | OW, OA | OW, OA | |
| `identity.effective_permission` | OW, OA, ITA | | | |
| `identity.session` | | | | `manage` (other users' sessions): OA, ITA |
| `identity.api_token` S † | OW, OA, ITA | OW, OA, ITA | OW, OA, ITA | |
| `identity.trusted_device` | ITA, HRM, OA | | ITA, HRM, OA | |

### 5.7.4 Settings, Core and Audit

| Resource | View | Create / Update | Delete | Other actions |
|---|---|---|---|---|
| `settings.setting` | OW, OA, PR | `update`: OW, OA (organization and institution level), PR (branch level only) | | |
| `settings.branding` | OW, OA | OW, OA | | |
| `settings.language` † | OW, OA | OW, OA | | |
| `settings.terminology` † | OW, OA | OW, OA | | |
| `core.numbering` † | OW, OA | | | `manage`: OW, OA |
| `core.period_lock` | OW, OA, PR, ACT, HRM | | | `lock`: PR (attendance, exams), HRM (staff attendance), ACT (fee, accounting). `unlock`: only through `period_unlock` approval (5.9) |
| `core.pii` S | | | | `view_full`: OW, OA, PR, BM, ADO, HRM |
| `core.bulk_operation` | OA, ITA (all); own operations are baseline | | | |
| `audit.log` S | OW, OA, ITA, PR | | | `export`: OW, ITA |
| `audit.record_history` | MGT, ACT (financial records), HRM (employee records), EXC (marks) | | | |
| `audit.login_history` | OW, OA, ITA | | | |

### 5.7.5 Approval, Notification, ImportExport, Document

| Resource | View | Create / Update | Delete | Other actions |
|---|---|---|---|---|
| `approval.workflow` | OW, OA, PR | OW, OA | OW, OA | |
| `approval.request` | MGT (all in scope); own and assigned are baseline | | | `approve` (bulk approve): OW, PR, HRM |
| `approval.delegation` | OA | OA (for others; own delegation is baseline) | OA | |
| `notification.template` | OW, OA | OW, OA | OW, OA | |
| `notification.rule` | OW, OA | OW, OA | OW, OA | |
| `notification.provider` S † | OW, OA, ITA | OW, OA, ITA | OW, OA | `manage` (test send): OA, ITA |
| `notification.delivery_log` | OW, OA, ITA, PR | | | `export`: OW, OA, ITA |
| `notification.broadcast` | MGT | `create`: MGT | | `publish` (send): OW, OA, PR, VP, BM |
| `import_export.history` | OW, OA, ITA, PR | | | |
| `document.template` | OW, OA, PR | OW, OA | OW, OA | |
| `document.issued` | MGT, ADO | | | `issue` (reissue): PR, BM. `manage` (revoke): OW, PR |

### 5.7.6 Academic

| Resource | View | Create / Update | Delete / Restore | Other actions |
|---|---|---|---|---|
| `academic.session` | STAFF | OW, OA, PR | OW | `manage` (activate, close): OW, PR |
| `academic.education_level` | STAFF | OW, OA | OW, OA | |
| `academic.study_group` | STAFF | OW, OA | OW, OA | |
| `academic.subject` | STAFF | OW, OA, AC, HOD·a | OW, OA | `import`, `export`: OA, AC |
| `academic.class` | STAFF | OA, PR, AC | OA, PR | |
| `academic.section` | STAFF | OA, PR, VP, AC | OA, PR | |
| `academic.class_subject` | MGT, AC, HOD, EXC, TCH | PR, AC | PR, AC | |
| `academic.teaching_assignment` | MGT, AC, HOD, EXC, TCH·o | | | `assign`: PR, VP, AC, HOD·a |
| `academic.workload` | MGT, AC, HOD | | | |

### 5.7.7 Admission

| Resource | View | Create / Update | Delete | Other actions |
|---|---|---|---|---|
| `admission.form_template` | MGT, ADO | OA, ADO | OA | |
| `admission.application` | MGT, ADO, REC | ADO, REC | | `approve`, `reject`: PR, ADO. `convert`: ADO, BM. `override_window`: OW, PR. `export`, `print`: MGT, ADO |
| `admission.seat` | MGT, ADO, REC | | | |
| `admission.gr_register` | PR, BM, ADO | | | `print`, `export`: PR, BM, ADO |
| `admission.inquiry` R2 | MGT, ADO, REC | ADO, REC | ADO | `export`: MGT, ADO |
| `admission.assessment` R2 | PR, AC, ADO | PR, AC, ADO | ADO | |

### 5.7.8 Student

| Resource | View | Create / Update | Delete / Restore | Other actions |
|---|---|---|---|---|
| `student.student` | MGT, ADO, REC, ACT, CSH, EXC, AC, HOD, TCH, LBR, TM, TS·a, GDN, STD | `create`: ADO, BM. `update` (non-protected fields): ADO, BM, PR | | `export`, `print`: MGT, ADO. `import`: OA, ADO |
| `student.profile` | | | | `update_protected`: PR, BM |
| `student.document` | MGT, ADO, CT, GDN | `create`: ADO, BM, GDN (pending verification) | BM | `verify`: ADO, BM |
| `student.health` S | PR, VP, BM, CT | `update`: PR, BM | | |
| `student.photo` | | | | `view_full` (bypass wing photo restriction, 5.10): MGT, ADO |
| `student.enrollment` | MGT, ADO, AC, CT | | | `manage` (change section or class): PR, VP, BM, AC |
| `student.promotion` | MGT, AC | | | `process` (run promotion): PR, AC |
| `student.transfer` | MGT, ADO | `create` (request): PR, BM | | `approve`, `reject`: PR, OW |
| `student.withdrawal` | MGT, ADO, ACT | `create`: BM, ADO | | `approve`, `reject`: PR |
| `student.struck_off` | PR, BM, ACT | | | `manage` (strike off, readmit): PR |
| `student.category` | MGT, ACT, ADO | OA, PR, ACT | OA | |
| `student.sponsor` | MGT, ACT, ADO | OA, PR, ACT | OA | |
| `student.certificate` | MGT, ADO | | | `issue`, `print`: BM, ADO |
| `student.timeline` | MGT, ADO, CT | | | |
| `student.modification_request` | MGT, CT, GDN, STD | `create`: GDN, STD | | `approve`, `reject`: GDN (linked child), CT, PR |
| `student.bulk_action` | | | | `manage`: PR, BM, ADO |

### 5.7.9 Guardian

| Resource | View | Create / Update | Delete | Other actions |
|---|---|---|---|---|
| `guardian.guardian` | MGT, ADO, ACT, CSH | ADO, BM | | `export`, `import`: OA, ADO |
| `guardian.contact` | MGT, ADO, ACT, CSH, REC, CT | | | |
| `guardian.link` | MGT, ADO | `create`: ADO, BM | PR, BM | |
| `guardian.family` | MGT, ACT, ADO | | | `manage` (merge, split): ADO, ACT, BM |
| `guardian.custody` S | PR, OW | | | `manage`: PR, OW |
| `guardian.pickup` | REC, BM, CT, GDN | BM, ADO, GDN | BM, GDN | |
| `guardian.verification` | | | | `verify`: ADO, BM |

### 5.7.10 HR

| Resource | View | Create / Update | Delete | Other actions |
|---|---|---|---|---|
| `hr.employee` | OW, OA, PR, VP, BM, HRM, HOD·a | HRM, OA | | `export`, `import`, `print`: HRM, OA. `manage` (status change): HRM, PR |
| `hr.salary` S | OW, HRM, ACT | `update`: OW, HRM | | |
| `hr.designation` | MGT, HRM | HRM, OA | HRM, OA | |
| `hr.branch_assignment` | MGT, HRM | `create`: HRM, OA | HRM, OA | |
| `hr.document` | HRM, PR | HRM | HRM | |
| `hr.directory` | STAFF | | | |
| `hr.profile_correction` | HRM | | | `approve`, `reject`: HRM |
| `hr.contract` R2 | HRM, OW | HRM, OA | | |
| `hr.performance_review` R2 | PR, HOD·a, HRM, STAFF·o | PR, HOD·a, HRM | | `approve`: PR, OW |
| `hr.training` R2 | HRM, PR, STAFF·o | HRM | HRM | |
| `hr.disciplinary` S R2 | HRM, PR, OW | HRM, PR | | |
| `hr.separation` R2 | HRM, PR, OW | `create`: HRM | | `approve`, `reject`: PR, OW |

### 5.7.11 Attendance

| Resource | View | Create / Update | Delete | Other actions |
|---|---|---|---|---|
| `attendance.student_attendance` | MGT, AC, EXC, CT, ST, GDN, STD | | | `mark`: CT. `export`, `print`: MGT, AC, CT |
| `attendance.period_attendance` | MGT, AC, CT, ST | | | `mark`: ST |
| `attendance.staff_attendance` | OW, OA, PR, VP, BM, HRM, STAFF·o | | | `mark` (online and QR check-in for self): STAFF·o. `export`, `print`: HRM, PR |
| `attendance.administrative_entry` | PR, HRM | `create`: PR, HRM | | |
| `attendance.correction` | MGT, AC, CT, ST | `create` (request): CT, ST | | `approve`, `reject`: PR, VP |
| `attendance.missing_punch` | HRM, PR, STAFF·o | `create`: STAFF·o | | `approve`, `reject`: HRM, PR |
| `attendance.policy` | MGT, HRM | `update`: OA, PR, HRM | | |
| `attendance.device` | ITA, HRM, PR | ITA | ITA | `manage`: ITA |
| `attendance.device_enrollment` | ITA, HRM | ITA, HRM | ITA, HRM | |
| `attendance.punch_import` | ITA, HRM | | | `import`: ITA, HRM |
| `attendance.qr_kiosk` | | | | `manage` (operate kiosk screen): REC, ITA |
| `attendance.report` | MGT, AC, HRM, CT | | | `export`, `print`: MGT, AC, HRM, CT |
| `attendance.suspicious_alert` | PR, HRM, ITA | | | |

### 5.7.12 Leave

| Resource | View | Create / Update | Delete | Other actions |
|---|---|---|---|---|
| `leave.type` † | MGT, HRM | HRM, OA | HRM, OA | |
| `leave.policy` † | MGT, HRM | HRM, OA | HRM, OA | |
| `leave.balance` | HRM, PR, STAFF·o | | | `manage` (adjust): HRM |
| `leave.application` | MGT, HRM, CT (student leave), STAFF·o, GDN | `create`: STAFF·o (own), GDN (child), HRM and PR (on behalf of staff), CT and BM (on behalf of students) | | `approve`, `reject`: PR, VP, HRM, CT (student leave within workflow) |
| `leave.calendar` | MGT, HRM, AC | | | |
| `leave.report` | HRM, PR | | | `export`: HRM, PR |

### 5.7.13 Fee

| Resource | View | Create / Update | Delete / Restore | Other actions |
|---|---|---|---|---|
| `fee.head` † | MGT, ACT | OA, ACT·g | OA | |
| `fee.structure` | MGT, ACT, CSH, ADO | ACT | ACT (unused only) | `export`: ACT |
| `fee.discount_rule` | MGT, ACT | OA, ACT | OA | |
| `fee.concession` | MGT, ACT | `create` (request): ACT, PR, ADO | | `approve`, `reject`: PR, OW |
| `fee.scholarship` | MGT, ACT | `create`: ACT, PR | | `approve`, `reject`: PR, OW |
| `fee.installment_plan` | MGT, ACT | `create`: ACT | | `approve`, `reject`: PR |
| `fee.voucher` | MGT, ACT, CSH, ADO, GDN | `create` (ad hoc): ACT, ADO (admission vouchers) | | `generate`: ACT. `manage` (cancel, regenerate): ACT. `print`, `export`: ACT, CSH, BM |
| `fee.payment` | MGT, ACT, CSH, GDN | `create`: CSH, ACT | | `reverse` (request reversal): ACT, CSH. `print`, `export`: ACT, CSH |
| `fee.payment_reversal` | MGT, ACT | | | `approve`, `reject`: PR, OW |
| `fee.cheque` | ACT, CSH | `update` (clear, bounce): ACT | | |
| `fee.bank_challan` | ACT | `create`: ACT | | `import`: ACT |
| `fee.cashier_session` | ACT, PR, CSH·o | `create` (open, close own): CSH, ACT | | |
| `fee.cash_variance` | ACT, PR | | | `approve`, `reject`: ACT, PR |
| `fee.refund` | MGT, ACT | `create`: ACT | | `approve`, `reject`: PR, OW |
| `fee.ledger` | MGT, ACT, CSH, GDN | | | `print`, `export`: ACT, CSH. `import` (opening balances): ACT |
| `fee.defaulter` | ACT, PR, BM, OW, OA | | | `export`: ACT, PR. `manage` (send reminders): ACT |
| `fee.report` | OW, OA, PR, ACT, CSH·o (own daily cash) | | | `export`, `print`: OW, OA, PR, ACT |
| `fee.gateway` S † R2 | OW, OA | | | `manage`: OW, OA |

### 5.7.14 Accounting

| Resource | View | Create / Update | Delete | Other actions |
|---|---|---|---|---|
| `accounting.fiscal_year` † | OW, OA, ACT | OW, ACT·g | | `manage` (close): OW |
| `accounting.account` † | OW, OA, ACT | ACT·g | ACT·g (unused only) | |
| `accounting.journal` S | OW, OA, ACT | `create`: ACT | | `approve`, `reject`: OW, OA. `reverse`: ACT (through approval) |
| `accounting.ledger` | OW, OA, PR, ACT | | | `export`, `print`: OW, OA, ACT |
| `accounting.trial_balance` | OW, OA, ACT | | | `export`, `print`: OW, ACT |
| `accounting.receivables` | OW, OA, PR, ACT | | | `export`: OW, ACT |
| `accounting.opening_balance` † | ACT·g, OW | `create`: ACT·g | | |
| `accounting.bank_account` | OW, ACT | OW, ACT | OW | |
| `accounting.period_lock` | OW, ACT | | | `lock`: ACT. Unlock through approval |
| `accounting.expense` R2 | MGT, ACT | `create`: ACT, BM | | `approve`, `reject`: PR, OW |
| `accounting.vendor` R2 | ACT, BM | ACT | ACT | |
| `accounting.purchase` R2 | MGT, ACT | `create`: ACT, BM | | `approve`, `reject`: PR, OW |
| `accounting.reconciliation` R2 | ACT, OW | `create`: ACT | | |
| `accounting.statement` R2 | OW, OA, ACT | | | `export`, `print`: OW, OA, ACT |

### 5.7.15 Examination

| Resource | View | Create / Update | Delete | Other actions |
|---|---|---|---|---|
| `examination.exam_term` | MGT, AC, EXC, HOD, TCH | EXC, AC | EXC | |
| `examination.exam` | MGT, AC, EXC, HOD, TCH | EXC | EXC | |
| `examination.paper` | MGT, AC, EXC, HOD, TCH | EXC | EXC | |
| `examination.date_sheet` | STAFF, GDN, STD | `create`: EXC | | `publish`: EXC, PR. `print`: EXC, BM |
| `examination.admit_card` | EXC, BM, GDN, STD | | | `issue`, `print`: EXC, BM |
| `examination.marks` | PR, VP, AC, EXC, HOD, TCH | | | `mark` (enter): ST, CT. `submit`: ST, CT. `import`: EXC. `export`: EXC, PR |
| `examination.marks_review` | EXC, HOD | | | `approve`, `reject` (return): HOD, EXC |
| `examination.marks_revision` | EXC, PR, HOD, TCH·o | `create`: ST, CT, EXC | | `approve`, `reject`: PR, EXC |
| `examination.completion` | PR, AC, EXC, HOD | | | |
| `examination.invigilation` R2 | EXC, PR, TCH·o | EXC | EXC | |
| `examination.board_exam` R2 | PR, EXC | EXC | | |

### 5.7.16 Result

| Resource | View | Create / Update | Delete | Other actions |
|---|---|---|---|---|
| `result.grading_scheme` | MGT, AC, EXC | OA, EXC·i | OA | |
| `result.rule` | MGT, AC, EXC | OA, EXC·i | OA | |
| `result.batch` | PR, EXC | | | `process`: EXC |
| `result.publication` | PR, EXC, OW | `create` (request publication): EXC | | `approve`, `reject`: PR, OW. `publish` (execute after approval): EXC. `manage` (unpublish request): PR |
| `result.withhold` | PR, EXC | `create`: PR | | |
| `result.student_result` | PR, VP, AC, EXC, CT, ST (own subjects), GDN, STD (published only) | | | |
| `result.report_card` | EXC, BM, CT, GDN, STD | | | `print`: EXC, BM, CT |
| `result.analytics` | OW, OA, PR, AC, EXC, HOD | | | `export`: OW, PR, EXC |
| `result.recheck` R2 | EXC, PR, GDN, STD | `create`: GDN, STD | | `approve`, `reject`: EXC, PR |

### 5.7.17 Timetable

| Resource | View | Create / Update | Delete | Other actions |
|---|---|---|---|---|
| `timetable.bell_schedule` | STAFF | PR, AC | PR, AC | |
| `timetable.timetable` | STAFF, GDN, STD | AC | AC | `publish` (activate version): AC, PR. `print`: AC, TCH |
| `timetable.substitution` R2 | MGT, AC, TCH·o | AC, VP | AC, VP | |

### 5.7.18 Communication

| Resource | View | Create / Update | Delete | Other actions |
|---|---|---|---|---|
| `communication.announcement` | Audience members (baseline) and MGT, AC | MGT, AC | MGT | `publish`: MGT |
| `communication.diary` | MGT, AC, TCH, GDN, STD | TCH | TCH (same day only) | `publish`: TCH |
| `communication.diary_review` | AC, VP | | | `approve`, `reject`: AC, VP |
| `communication.message` R2 | TCH, GDN | `create`: TCH, GDN | | |
| `communication.message_oversight` R2 | PR, VP | | | |
| `communication.ptm` R2 | MGT, TCH, GDN | `create` (event): PR, VP; slots: TCH·o | PR, VP | `manage` (book slot): GDN |

### 5.7.19 Learning, Homework, Assessment (R2)

| Resource | View | Create / Update | Delete | Other actions |
|---|---|---|---|---|
| `learning.course` | AC, HOD, ST | | | `manage`: AC, HOD |
| `learning.lecture` | PR, AC, HOD, ST, GDN, STD | ST | ST | `publish`: ST |
| `learning.discussion` | ST, AC, STD | `create`: ST, STD | ST (moderation) | `manage` (pin, resolve, disable comments): ST, AC |
| `learning.content_report` | ST, AC | `create`: STD | | `manage` (resolve): ST, AC |
| `learning.progress` | ST, AC, GDN | | | |
| `homework.assignment` | PR, AC, HOD, TCH, GDN, STD | TCH | TCH | `publish`: TCH |
| `homework.submission` | TCH, AC, GDN, STD | `create` (submit): STD, GDN | | `approve` (review and grade), `reject` (return): TCH |
| `homework.report` | PR, AC, TCH | | | `export`: AC, TCH |
| `assessment.question_bank` | HOD, ST | HOD, ST | HOD, ST | `manage` (share): HOD, ST |
| `assessment.question` | HOD, ST | HOD, ST | HOD, ST | `import`: HOD, ST |
| `assessment.quiz` | PR, AC, HOD, EXC, ST, STD | ST, EXC | ST, EXC | `publish`: ST, EXC |
| `assessment.attempt` | ST, STD | `create`: STD | | |
| `assessment.grading` | ST, EXC | | | `manage` (grade subjective): ST |
| `assessment.analytics` | HOD, AC, ST | | | `export`: HOD, ST |

### 5.7.20 Payroll (R2, all sensitive)

| Resource | View | Create / Update | Delete | Other actions |
|---|---|---|---|---|
| `payroll.component` † | OW, HRM, ACT | OW, HRM, ACT·g | OW | |
| `payroll.structure` | OW, HRM, ACT | OW, HRM | OW | |
| `payroll.employee_salary` | OW, HRM, ACT | OW, HRM | | |
| `payroll.run` | OW, PR, HRM, ACT | `create` (create, compute, adjust): HRM, ACT | | `approve`, `reject`: OW, PR. `finalize`: ACT |
| `payroll.payslip` | OW, HRM, ACT, STAFF·o | | | `print`: HRM, ACT |
| `payroll.advance` | OW, HRM, ACT, STAFF·o | `create`: STAFF·o, HRM | | `approve`, `reject`: OW, PR |
| `payroll.loan` | OW, HRM, ACT, STAFF·o | `create`: STAFF·o, HRM | | `approve`, `reject`: OW |
| `payroll.payment` | OW, ACT | `create`: ACT | | |
| `payroll.report` | OW, HRM, ACT | | | `export`: OW, HRM, ACT |

### 5.7.21 Library, Transport, Inventory

| Resource | View | Create / Update | Delete | Other actions |
|---|---|---|---|---|
| `library.book` R3 | LBR, MGT, STAFF, STD | LBR | LBR | `import`: LBR |
| `library.copy` R3 | LBR | LBR | LBR | `print` (labels): LBR |
| `library.member` R3 | LBR | LBR | | |
| `library.circulation` R3 | LBR, STAFF·o, STD, GDN | | | `issue` (issue, return, renew): LBR |
| `library.fine` R3 | LBR, ACT, PR | | | `manage` (waive): LBR, PR |
| `library.report` R3 | LBR, PR | | | `export`: LBR |
| `transport.route` R2 | TM, MGT, TS, GDN | TM | TM | `print` (route sheets): TM |
| `transport.stop` R2 | TM, MGT | TM | TM | |
| `transport.allocation` R2 | TM, MGT, ACT, TS, GDN | TM | TM | |
| `transport.vehicle` R3 | TM, MGT | TM | TM | |
| `transport.driver` R3 | TM, MGT | TM | TM | |
| `transport.maintenance` R3 | TM, BM | TM | | |
| `transport.fuel` R3 | TM, BM, ACT | TM | | |
| `transport.boarding` R3 | TM, TS, GDN | | | `mark`: TS |
| `transport.gps` R3 | TM, GDN | | | |
| `inventory.item` R3 | BM, ACT | BM | BM | |
| `inventory.location` R3 | BM | BM | BM | |
| `inventory.stock` R3 | BM, ACT | | | `manage` (receive, issue, transfer): BM. `create` (adjustment request): BM |
| `inventory.adjustment` R3 | BM, PR | | | `approve`, `reject`: PR |
| `inventory.asset` R3 | BM, ACT, PR | BM | | `assign`: BM |
| `inventory.maintenance` R3 | BM, STAFF·o | `create`: STAFF·o; `update`: BM | | |

### 5.7.22 Administration, Discipline, Reporting

| Resource | View | Create / Update | Delete | Other actions |
|---|---|---|---|---|
| `administration.calendar` | Everyone (baseline) | OA, PR, BM | OA, PR | |
| `administration.closure` | MGT | `create`: OW, PR | | |
| `administration.event` | Everyone (baseline) | PR, BM | PR, BM | |
| `administration.id_card` | BM, ADO | | | `print`: BM, ADO |
| `administration.gate_pass` R2 | REC, BM, PR, GDN | `create`: REC, BM | | |
| `administration.visitor` R3 | REC, BM | REC | | |
| `administration.complaint` R3 | BM, PR, STAFF·o, GDN | `create`: STAFF·o, GDN | | `assign`, `manage` (resolve): BM, PR |
| `discipline.incident` R2 | PR, VP, CT | TCH, VP | | |
| `discipline.action` R2 | PR, VP, CT, GDN | `create`: VP, PR | | |
| `discipline.remark` R2 | PR, VP, CT, GDN (shared remarks only) | TCH | TCH (same day) | |
| `discipline.achievement` R2 | MGT, TCH, GDN, STD | TCH, PR | PR | |
| `discipline.counseling` S R3 | PR (view only), counselor custom role | Counselor custom role | | |
| `reporting.branch_comparison` | OW, OA | | | |
| `reporting.scheduled_report` R2 | OW, OA, PR, ACT | OW, OA, PR, ACT | OW, OA, PR, ACT | |
| `reporting.snapshot` | OA, ITA | | | `manage` (force refresh): OA, ITA |

**Dashboards:** not separate permissions. The dashboard shown depends on the active role; each widget checks the permission of the data it shows (for example the collection widget needs `fee.report.view`).

### 5.7.23 Import permissions summary

Import is always a module permission, never a generic one:

| Import type | Permission |
|---|---|
| Students | `student.student.import` |
| Guardians and families | `guardian.guardian.import` |
| Employees | `hr.employee.import` |
| Subjects | `academic.subject.import` |
| Classes and sections | `academic.class.create` plus `academic.section.create` |
| Opening fee balances | `fee.ledger.import` |
| Opening ledger balances | `accounting.opening_balance.create` |
| Attendance punches | `attendance.punch_import.import` |
| Marks | `examination.marks.import` |
| Bank challan file | `fee.bank_challan.import` |

---

## 5.8 Guardian and Student Portal Permission Sets (locked)

| Permission | Guardian (linked children) | Student (self) |
|---|---|---|
| `student.student.view` | Yes | Yes |
| `student.document.view`, `student.document.create` | Yes | No |
| `student.modification_request.create` | Yes | Yes |
| `student.modification_request.approve`, `.reject` | Yes (child's requests) | No |
| `attendance.student_attendance.view` | Yes | Yes |
| `leave.application.create`, `.view` | Yes (for child) | No |
| `fee.voucher.view`, `fee.payment.view`, `fee.ledger.view` | Yes | Configurable (default no) |
| `examination.date_sheet.view`, `examination.admit_card.view` | Yes | Yes |
| `result.student_result.view`, `result.report_card.view` | Yes (published) | Yes (published) |
| `timetable.timetable.view` | Yes | Yes |
| `communication.diary.view` | Yes | Yes |
| `guardian.pickup.view`, `.create`, `.update`, `.delete` | Yes | No |
| `administration.gate_pass.view` R2 | Yes | No |
| `learning.lecture.view`, `learning.progress.view` R2 | Yes | Lectures only |
| `learning.discussion.create`, `learning.content_report.create` R2 | No | Yes |
| `homework.assignment.view`, `homework.submission.create` R2 | Yes | Yes |
| `assessment.quiz.view`, `assessment.attempt.create` R2 | View only | Yes |
| `communication.message.create` R2 | Yes | No |
| `communication.ptm.manage` (book) R2 | Yes | No |
| `result.recheck.create` R2 | Yes | Yes (if allowed) |
| `transport.route.view`, `transport.allocation.view` R2 | Yes | No |

Organizations can switch individual student-portal features off through settings (for example hide fee status from students), but cannot add staff permissions to these roles.

---

## 5.9 Approval Permissions and Default Workflows

Each request type has a submit permission, an approve and reject permission, and a seeded default workflow (editable per organization, D11 details the flows).

| Request type | Submit permission | Default steps (approver resolver) | Approve / reject permission |
|---|---|---|---|
| `attendance_correction` | `attendance.correction.create` | 1. principal or vice principal of branch (any) | `attendance.correction.approve` / `.reject` |
| `missing_punch` | `attendance.missing_punch.create` | 1. HR manager of branch | `attendance.missing_punch.approve` / `.reject` |
| `student_leave` | `leave.application.create` | 1. class teacher of student. 2. principal if more than 3 days | `leave.application.approve` / `.reject` |
| `staff_leave` | `leave.application.create` | 1. reporting manager (or HOD). 2. principal if more than 3 days. HR manager notified | `leave.application.approve` / `.reject` |
| `student_profile_correction` | `student.modification_request.create` | 1. guardian (if raised by student; skipped if student is 18+ or in college program). 2. class teacher. 3. principal if a protected field is involved | `student.modification_request.approve` / `.reject` |
| `student_modification` | `student.modification_request.create` | Same as above | Same as above |
| `academic_correction` | `student.modification_request.create` | 1. guardian (if raised by student). 2. class teacher. 3. examination controller | `student.modification_request.approve` / `.reject` |
| `marks_revision` | `examination.marks_revision.create` | 1. HOD of subject. 2. examination controller | `examination.marks_revision.approve` / `.reject` |
| `result_publication` | `result.publication.create` | 1. principal | `result.publication.approve` / `.reject` |
| `fee_concession` | `fee.concession.create` | 1. principal. 2. owner if above threshold (default 25%) | `fee.concession.approve` / `.reject` |
| `scholarship` | `fee.scholarship.create` | 1. principal. 2. owner | `fee.scholarship.approve` / `.reject` |
| `fee_refund` | `fee.refund.create` | 1. principal. 2. owner if above amount threshold | `fee.refund.approve` / `.reject` |
| `payment_reversal` | `fee.payment.reverse` | 1. principal | `fee.payment_reversal.approve` / `.reject` |
| `cash_variance` | Raised automatically at cashier close | 1. accountant (not the cashier). 2. principal if above threshold | `fee.cash_variance.approve` / `.reject` |
| `period_unlock` | `core.period_lock.lock` holders request unlock | 1. principal (attendance, exams); owner (fee, accounting) | `core.period_lock.unlock` |
| `branch_transfer` | `student.transfer.create` | 1. principal of source branch. 2. principal of target branch | `student.transfer.approve` / `.reject` |
| `withdrawal` | `student.withdrawal.create` | 1. principal | `student.withdrawal.approve` / `.reject` |
| `document_verification` | `student.document.create` | 1. admission officer | `student.document.verify` |
| `journal_entry` | `accounting.journal.create` (above threshold) | 1. owner or organization admin | `accounting.journal.approve` / `.reject` |
| `employee_profile_correction` | Baseline (own profile) | 1. HR manager | `hr.profile_correction.approve` / `.reject` |
| `installment_plan` | `fee.installment_plan.create` | 1. principal | `fee.installment_plan.approve` / `.reject` |
| `expense` R2 | `accounting.expense.create` | 1. principal. 2. owner above threshold | `accounting.expense.approve` / `.reject` |
| `purchase` R2 | `accounting.purchase.create` | 1. principal. 2. owner above threshold | `accounting.purchase.approve` / `.reject` |
| `payroll_run` R2 | `payroll.run.create` | 1. principal. 2. owner | `payroll.run.approve` / `.reject` |
| `result_recheck` R2 | `result.recheck.create` | 1. examination controller | `result.recheck.approve` / `.reject` |
| `stock_adjustment` R3 | `inventory.stock.create` | 1. principal | `inventory.adjustment.approve` / `.reject` |

Rules: DEC-54 (workflow plus permission), BR-APR-01 (no self-approval), separation of duties (5.11).

---

## 5.10 Field-Level Visibility and Masking

| Data | Default display | Full display requires |
|---|---|---|
| CNIC and B-Form numbers | Masked (`35202-*****-1`) | `core.pii.view_full` |
| Guardian mobile numbers and address | Hidden | `guardian.contact.view` (granted to CT for assigned students; not to ST by default) |
| Student home address | Hidden from teachers | `student.student.update` or `guardian.contact.view` |
| Student health profile | Hidden | `student.health.view` S |
| Counseling records | Hidden | `discipline.counseling.view` S |
| Salary, allowances, deductions, bank account of employees | Hidden | `hr.salary.view` S or payroll permissions |
| Employee CNIC, personal phone | Masked or hidden | `core.pii.view_full` or `hr.employee.update` |
| Fee balance on student profile | Hidden from teachers | `fee.ledger.view` |
| Photos of students in a girls' wing (when organization setting `restrict_photos_by_wing` is on) | Visible only to female staff assigned to that wing and to holders of the permission | `student.photo.view_full` |
| Custody restriction details | Hidden | `guardian.custody.view` S |
| Audit log old and new values for sensitive fields | Masked in log itself (D2 FR-AUD-003) | Not recoverable from the log |

Exports apply the same masking as the screen for the exporting user.

---

## 5.11 Separation of Duties

| ID | Rule | Enforced in |
|---|---|---|
| SOD-01 | The user who recorded a payment cannot approve its reversal | `PaymentReversalHandler` |
| SOD-02 | A cashier cannot approve the variance on their own cashier session | `CashVarianceHandler` |
| SOD-03 | The creator of a manual journal cannot approve it | `JournalEntryHandler` |
| SOD-04 | A teacher who entered marks cannot approve that marks submission, even if they also hold HOD or exam controller roles | `ApproveMarksSubmission` |
| SOD-05 | The requester of a marks revision cannot approve it | `MarksRevisionHandler` |
| SOD-06 | The preparer of a payroll run cannot approve it (R2) | `PayrollRunHandler` |
| SOD-07 | A user cannot assign roles or overrides to themselves | `AssignRole`, `GrantPermissionOverride` |
| SOD-08 | A user cannot approve a concession, scholarship or refund for a student who is their own linked child | Fee handlers (check DEC-47 links) |
| SOD-09 | A user cannot approve leave for themselves | `LeaveApplicationHandler` |
| SOD-10 | The requester of a period unlock cannot approve it | `PeriodUnlockHandler` |

If SOD blocks the only resolved approver, the request escalates to the next workflow step or to the principal (BR-APR-04).

---

## 5.12 Assignment and Override Rules

| Rule | Detail |
|---|---|
| Subset rule (DEC-55) | When assigning role R at scope S, the assigner must hold every permission of R at scope S with equal or wider visibility. Same for overrides |
| Scope rule | A branch-scoped assigner can only assign branch scopes within their own branches |
| Principal limits | Principals can assign only these roles in their branch by default: `vice_principal`, `hod`, `academic_coordinator`, `class_teacher`, `subject_teacher`, `cashier`, `receptionist`, `librarian`, `transport_staff`, `support_staff`, plus custom roles passing the subset rule |
| Dates | Assignments have `starts_on` and optional `ends_on`; future-dated assignments activate automatically |
| Overrides | Reason mandatory; optional expiry; deny always wins; overrides appear on the user's effective-permission screen with who granted them |
| Owner protection | At least one active `org_owner` assignment must exist (BR-IAM-02); the `org_owner` role is locked |
| Role deletion | A role with active assignments cannot be deleted |
| Custom role from template | Cloning copies permissions and visibility; the clone is `is_customized = true` and does not receive template updates automatically |
| Audit | Every assignment, revocation, override and role permission change is audited (FR-AUD-001) |

---

## 5.13 Permission Catalog Sync and Template Updates (DEC-60)

1. Each module declares permissions in its `PermissionRegistry` contribution: key, labels, sensitivity, allowed scopes, portal, release, default template roles.
2. On deploy, `identity:sync-permissions` upserts `permissions`, never deletes keys in use (removed keys are marked deprecated and stop granting access).
3. For every organization, new keys whose `default_template_roles` include a role are granted to that role's copy if `is_customized = false`; customized roles get a "suggested permissions" notice for OW and OA.
4. `roles.template_version` records the template version applied.
5. "Reset to template" restores a customized system role to the current template after confirmation.

---

## 5.14 RBAC Test Matrix

**Fixture-driven tests:** this deliverable's matrices are encoded as `tests/Fixtures/rbac_matrix.php` (role slug, permission key, expected visibility or `false`). A generated test asserts the seeded templates match the fixture exactly, and a second generated test checks representative routes per permission.

| Test group | Examples |
|---|---|
| Template integrity | Seeded `cashier` has `fee.payment.create` and not `fee.refund.approve`; `org_owner` has every tenant permission |
| Resolution | Deny override beats role grant; expired override ignored; future-dated assignment inactive until start date |
| Scope | Principal of Branch A denied on Branch B record; org-scope-only permission denied for branch-scoped accountant |
| Visibility | Class teacher sees only assigned students; HOD sees only department subjects' marks; guardian sees only linked children |
| Visibility capping | Branch assignment with `organization` visibility behaves as `branch` |
| Portal | Staff permission not usable in guardian portal; guardian cannot reach `/app` routes |
| Entitlement | Permission held but module disabled returns 404 |
| Escalation | Principal cannot assign `accountant` (not in their set); org admin cannot grant a permission they lack; nobody can assign roles to themselves |
| Separation of duties | Each SOD rule has a failing-path test |
| Approval permission | Resolved approver without approve permission cannot decide; user with permission but not resolved by workflow cannot decide |
| Field masking | CNIC masked for teacher, full for admission officer; exports masked the same way |
| Two-factor | User holding a sensitive permission without confirmed 2FA is challenged |
| Cache | Permission change takes effect on next request; override expiry lapses without manual cache clear |
| Catalog sync | New permission auto-granted to uncustomized template copy, not to customized role |

---

## Assumptions

| ID | Assumption |
|---|---|
| A-39 | Teachers in Pakistani schools routinely phone parents, so class teachers get guardian contact visibility for assigned students by default; organizations can remove it. |
| A-40 | The head-office accountant of a chain is assigned `accountant` with organization scope; branch accountants with branch scope. |
| A-41 | School nurses, counselors and store keepers are rare enough to be custom roles rather than templates. |
| A-42 | Approval thresholds (concession 25%, refund amount, journal amount) are organization settings with these defaults. |
| A-43 | Students do not see fee status by default (setting), because many families prefer fee matters to stay with parents. |

## Gaps Found and Added

| ID | Gap | Decision | Release |
|---|---|---|---|
| GAP-89 | Only one platform role defined | Platform support and billing roles (DEC-61) | R1 |
| GAP-90 | Admins could grant themselves or others more power than they hold | Subset rule (DEC-55), SOD-07 | R1 |
| GAP-91 | No separation of duties for money, marks, payroll | SOD-01 to SOD-10 | R1 |
| GAP-92 | Self-service abilities could be removed by role editing, locking users out | Baseline abilities outside RBAC (DEC-57) | R1 |
| GAP-93 | No field-level masking rules | 5.10 and DEC-62 | R1 |
| GAP-94 | Approval steps could grant powers via workflow misconfiguration | DEC-54 | R1 |
| GAP-95 | Expiring grants could outlive expiry in cache | Cache TTL bound to next expiry (5.3) | R1 |
| GAP-96 | New permissions in later releases would not reach existing organizations | Template versioning and sync (DEC-60) | R1 |
| GAP-97 | Visibility could exceed scope | Visibility capping (DEC-58) | R1 |
| GAP-98 | Sensitive permission holders without two-factor | DEC-63 | R1 |
| GAP-99 | Wing-based photo privacy needed a permission | `student.photo.view_full` with setting | R1 |
| GAP-100 | Conflict of interest when staff approve fee matters for their own children | SOD-08 | R1 |

---

## What Deliverable 6 Covers

Complete academic structure and student lifecycle: configuration of levels, study groups and programs with Punjab defaults, session lifecycle (planning, active, closing, closed), class and section setup, subject mapping and electives, teaching assignments, the student lifecycle state machine (applicant to graduate, with every transition, its preconditions, effects and history records), promotion, retention, cross-institution progression, transfers, withdrawal, struck-off and readmission, certificates, and the student timeline event catalog.
