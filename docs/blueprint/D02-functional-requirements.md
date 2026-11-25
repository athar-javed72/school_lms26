# Deliverable 2: Detailed Functional Requirements and Module Breakdown

**Version:** 1.0 (draft for approval) | **Date:** 23 Nov 2026
**Builds on:** Deliverable 1 (module names, naming conventions, DEC-01 to DEC-20, release scope)

---

## 2.0 How to Read This Deliverable

**ID scheme (binding for D10, D11, D14, D15)**

| Prefix | Meaning | Example |
|---|---|---|
| FR-{CODE}-NNN | Functional requirement | FR-FEE-012 |
| BR-{CODE}-NN | Business rule | BR-ATT-03 |
| AC-{CODE}-NN | Acceptance criterion | AC-RES-02 |
| US-{CODE}-NN | User story | US-GRD-01 |

**Module codes**

| Code | Module | Code | Module | Code | Module |
|---|---|---|---|---|---|
| GEN | Cross-cutting (all modules) | ACD | Academic | COMM | Communication |
| CORE | Core | ADM | Admission | LMS | Learning |
| PLT | Platform | STU | Student | HWK | Homework |
| ORG | Organization | GRD | Guardian | ASM | Assessment |
| IAM | Identity | HR | HR | PAY | Payroll |
| SET | Settings | ATT | Attendance | LIB | Library |
| AUD | Audit | LVE | Leave | TRN | Transport |
| APR | Approval | FEE | Fee | INV | Inventory |
| NTF | Notification | ACC | Accounting | ADN | Administration |
| IMX | ImportExport | EXM | Examination | DSC | Discipline |
| DOC | Document | RES | Result | RPT | Reporting |
| | | TTB | Timetable | | |

**Role abbreviations (role slugs from D1)**

| Abbr | Role slug | Abbr | Role slug | Abbr | Role slug |
|---|---|---|---|---|---|
| SA | `platform_super_admin` | CT | `class_teacher` | ADO | `admission_officer` |
| OW | `org_owner` | ST | `subject_teacher` | REC | `receptionist` |
| OA | `org_admin` | ACT | `accountant` | TM | `transport_manager` |
| PR | `principal` | CSH | `cashier` | TS | `transport_staff` |
| VP | `vice_principal` | HRM | `hr_manager` | ITA | `it_admin` |
| BM | `branch_manager` | LBR | `librarian` | STD | `student` |
| HOD | `hod` | EXC | `exam_controller` | GDN | `guardian` |
| AC | `academic_coordinator` | SUP | `support_staff` | | |

Groups: **MGT** = OW, OA, PR, VP, BM. **TCH** = CT, ST.

**Columns:** Rel = release (R1, R2, R3). Pri = M (must for that release), S (should), C (could).

**Key entities** listed per module are the canonical table names. Deliverable 7 defines their columns, keys and indexes using exactly these names.

---

## 2.1 New Decisions Introduced in This Deliverable

| ID | Decision |
|---|---|
| DEC-21 | Data visibility levels, attached to role assignments and permissions: `own`, `assigned`, `branch`, `institution`, `organization` |
| DEC-22 | One user account per login identifier per organization. Guardian records sharing one mobile number link to the same user, who sees all linked children |
| DEC-23 | Fee accounting is accrual based: voucher generation posts receivable and income; payment posts cash/bank against receivable. Posting rules detailed in D11 |
| DEC-24 | Attendance status codes: `P` present, `A` absent, `L` late, `HD` half day, `LV` leave, `H` holiday, `W` weekend, `NM` not marked |
| DEC-25 | Official daily attendance precedence: approved correction > teacher marking > device or QR punch > automatic absent |
| DEC-26 | One `active` academic session per institution at a time, plus at most one `planning` session (for next year's admissions and setup) |
| DEC-27 | Records outside the user's tenant return HTTP 404 (not 403), so record existence is never disclosed |
| DEC-28 | Exam marks use distinct codes: numeric marks, `ABS` absent, `MED` medical, `EXM` exempted. Absent is never stored as zero |

---

## 2.2 Cross-Cutting Requirements (apply to every module)

| ID | Requirement | Rel | Pri |
|---|---|---|---|
| FR-GEN-001 | Every list screen has server-side pagination (default 25, max 100 rows), search, filters, sort, and filters kept in the URL | R1 | M |
| FR-GEN-002 | Every list and report can export the current filtered result to Excel, CSV, PDF and print view, gated by `{module}.{resource}.export` / `.print`. Exports over 2,000 rows are queued | R1 | M |
| FR-GEN-003 | Every request checks, in order: authentication, tenant scope, branch scope, module entitlement, policy | R1 | M |
| FR-GEN-004 | Users with several branches get a branch context switcher. Organization-scope roles can choose "All branches" for read screens only; every write needs a specific branch | R1 | M |
| FR-GEN-005 | Every write on an audited model creates an audit entry (AUD) | R1 | M |
| FR-GEN-006 | Changes to protected records require a reason, stored with the audit entry | R1 | M |
| FR-GEN-007 | All UI text uses translation keys. English and Urdu (RTL). Western digits by default | R1 | M |
| FR-GEN-008 | Uploads: whitelisted types, size limits by plan, private storage, served only through short-lived signed URLs after a policy check | R1 | M |
| FR-GEN-009 | Master data uses soft delete and restore (`restore` permission). Transactional records are never deleted after finalization | R1 | M |
| FR-GEN-010 | Edit forms use optimistic locking: saving a record changed by someone else since it was opened shows a conflict message instead of overwriting | R1 | M |
| FR-GEN-011 | Every screen has defined empty, loading and error states (D9) | R1 | M |
| FR-GEN-012 | A module not entitled for the organization is hidden in navigation and its routes and API return 404 | R1 | M |
| FR-GEN-013 | Payments, attendance ingestion and bulk actions accept an idempotency key; a repeated submission returns the first result instead of creating a duplicate | R1 | M |
| FR-GEN-014 | Bulk operations run as queued jobs with a progress indicator, a result summary, and an in-app notification on completion | R1 | M |
| FR-GEN-015 | Writes into a locked period are rejected unless an approved unlock exists (CORE period locks) | R1 | M |
| FR-GEN-016 | WCAG 2.2 AA: keyboard navigation, labels, contrast, focus states | R1 | M |
| FR-GEN-017 | Guardian, student and teacher screens work on a 360px wide screen | R1 | M |
| FR-GEN-018 | Print views use organization and institution branding | R1 | M |
| FR-GEN-019 | Records outside the tenant return 404 (DEC-27) | R1 | M |

---

## 2.3 Module Dependency Map

| Module | Depends on |
|---|---|
| Core | none |
| Platform | Core |
| Organization | Core, Platform |
| Identity | Core, Organization |
| Settings | Core, Organization |
| Audit | Core, Identity |
| Approval | Core, Identity, Notification, Audit |
| Notification | Core, Identity, Settings |
| ImportExport | Core, Identity, Audit |
| Document | Core, Settings, Organization |
| Academic | Organization, Identity, HR (for teaching assignments) |
| Admission | Academic, Student, Guardian, Fee, Document, Notification |
| Student | Academic, Guardian, Document, Approval |
| Guardian | Identity, Notification |
| HR | Organization, Identity, Document |
| Attendance | Academic, Student, HR, Approval, Notification, Administration (calendar) |
| Leave | HR, Student, Approval, Attendance |
| Fee | Academic, Student, Guardian, Accounting, Document, Approval, Notification |
| Accounting | Organization, Approval |
| Examination | Academic, Student, HR, Fee (clearance), Document, Approval |
| Result | Examination, Attendance, Document, Approval, Notification |
| Timetable | Academic, HR, Organization (rooms), Attendance (substitution) |
| Communication | Academic, Student, Guardian, Notification |
| Learning | Academic, Student, Notification |
| Homework | Academic, Student, Notification |
| Assessment | Academic, Student, Examination (optional mark transfer) |
| Payroll | HR, Attendance, Leave, Accounting, Approval, Document |
| Library | Student, HR, Fee (fines) |
| Transport | Student, HR, Fee |
| Inventory | Organization, HR, Accounting, Approval |
| Administration | Organization, Academic, Document, Notification |
| Discipline | Student, Notification |
| Reporting | All modules (read only) |

Rule: dependencies point one way. A lower module never calls a higher one directly; it fires an event (D3).

---

## 2.4 Core (`core`) | CORE | Core tier | R1

**Purpose:** Shared foundation used by all modules. No business screens of its own.
**Key entities:** `numbering_sequences`, `stored_files`, `file_attachments`, `period_locks`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-CORE-001 | Tenancy trait applies `organization_id` (and `branch_id` where defined) automatically on every query and fills it on create from the current context, ignoring any value sent in the request | System | R1 | M |
| FR-CORE-002 | Tenant context resolved from subdomain plus authenticated user; a mismatch logs the user out and records a security event | System | R1 | M |
| FR-CORE-003 | Numbering service: configurable formats with tokens (`{BRANCH}`, `{YYYY}`, `{YY}`, `{FY}`, `{SEQ:n}`), sequence per scope (organization, branch, fiscal year), row-locked increment, gapless option for receipts | OA | R1 | M |
| FR-CORE-004 | `Money` value object and Eloquent cast; configurable rounding (e.g., vouchers rounded to whole rupees) | System | R1 | M |
| FR-CORE-005 | File storage service: private disk or S3-compatible, metadata in `stored_files`, polymorphic `file_attachments`, signed temporary URLs authorized by the owning record's policy | System | R1 | M |
| FR-CORE-006 | Period lock service: lock and unlock a module period (month) per branch; check API used by Attendance, Fee, Accounting, Examination, Payroll | ACT, PR | R1 | M |
| FR-CORE-007 | Shared Blade component library (`x-ui.*`) implementing D9 | System | R1 | M |
| FR-CORE-008 | Correlation ID generated per request, carried into logs, queued jobs and audit entries | System | R1 | M |
| FR-CORE-009 | Entitlement helper to check whether a module or feature is enabled for the organization | System | R1 | M |
| FR-CORE-010 | Health check endpoint (database, cache, queue, storage) for monitoring | ITA, SA | R1 | M |

**Business rules**
- BR-CORE-01: Tenant scope can be bypassed only through an explicitly named method allowed in Platform code and system jobs; every use is listed in a code review checklist.
- BR-CORE-02: Issued numbers are never reused, even after cancellation.
- BR-CORE-03: Money is never stored or computed as a PHP float.

**Acceptance criteria**
- AC-CORE-01: A record created in organization A's context gets A's `organization_id` even when the request posts B's id.
- AC-CORE-02: A query in A's context never returns B's rows, verified by an automated test per tenant-owned model.
- AC-CORE-03: 100 parallel number requests on one sequence produce 100 unique, consecutive numbers.
- AC-CORE-04: A signed file URL stops working after expiry and returns 404 for users of another organization.

---

## 2.5 Platform (`platform`) | PLT | Core tier | R1 to R3

**Purpose:** Run the SaaS: plans, tenants, entitlements, support access.
**Key entities:** `plans`, `plan_modules`, `organization_subscriptions`, `organization_modules`, `usage_counters`, `support_access_grants`, `platform_announcements`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-PLT-001 | Create and edit plans: name, limits (max active students, branches, storage GB, monthly SMS credits), included modules | SA | R1 | M |
| FR-PLT-002 | Create an organization: profile, subdomain, plan, trial end date, owner invitation (email or SMS) | SA | R1 | M |
| FR-PLT-003 | Tenant states `trial`, `active`, `suspended`, `archived`. Suspended: login allowed, read-only, banner. Archived: no login, data retained for the contractual period | SA | R1 | M |
| FR-PLT-004 | Per-organization module overrides (enable an add-on, extend a limit) with effective dates | SA | R1 | M |
| FR-PLT-005 | Tenant console: list organizations with state, plan, student count, branch count, storage used, SMS used, last activity | SA | R1 | M |
| FR-PLT-006 | Support access: SA requests access, organization owner grants it for a fixed number of hours; "login as" shows a visible banner; every action is audited with both identities | SA, OW | R1 | M |
| FR-PLT-007 | Platform announcements shown to organization admins (maintenance windows, releases) | SA | R1 | S |
| FR-PLT-008 | Usage metering stored daily in `usage_counters` | System | R2 | M |
| FR-PLT-009 | Tenant data export package (all data as CSV/JSON plus files) on request or on offboarding | SA, OW | R2 | M |
| FR-PLT-010 | Self-service subscription and billing for the platform's own invoices | OW | R3 | S |

**Business rules**
- BR-PLT-01: SA sees tenant metadata (counts, plan, usage) but not tenant records without an active support grant.
- BR-PLT-02: Hitting a limit blocks the specific action (e.g., new admission) with a clear message. Data is never deleted because of a limit.
- BR-PLT-03: Disabling a module hides it; its data stays intact and returns when re-enabled.

**User stories**
- US-PLT-01: As SA, I create a new organization on a trial plan so that a school can start setup the same day.
- US-PLT-02: As OW, I grant support access for 2 hours so that support can fix my voucher template without knowing my password.

**Acceptance criteria**
- AC-PLT-01: A suspended organization's users can log in and view, and every write returns a read-only message.
- AC-PLT-02: Login-as without an active grant is refused and logged.
- AC-PLT-03: When the student limit is reached, admission conversion is blocked and existing students are unaffected.

---

## 2.6 Organization (`organization`) | ORG | Core tier | R1

**Purpose:** Organizational structure and the onboarding wizard.
**Key entities:** `organizations`, `institutions`, `branches`, `departments`, `shifts`, `rooms`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-ORG-001 | Organization profile: legal name, display name, logo, address, NTN (optional), contacts, timezone, currency, fiscal year start month | OW, OA | R1 | M |
| FR-ORG-002 | Institutions: name (en/ur), type (`school`, `college`, `school_and_college`), board affiliation (e.g., BISE Lahore), registration numbers, logo and document header | OW, OA | R1 | M |
| FR-ORG-003 | Branches: code, name (en/ur), address, city, district, phones, email, opening date, map coordinates (optional), status, principal | OW, OA | R1 | M |
| FR-ORG-004 | Branch bank accounts used on challans (bank, branch, account title, number, IBAN) | OW, ACT | R1 | M |
| FR-ORG-005 | Departments at organization or branch level (e.g., Science, Accounts, Administration) with head | OA, PR | R1 | M |
| FR-ORG-006 | Shifts per branch: name, start and end time, working days | PR, BM | R1 | M |
| FR-ORG-007 | Rooms per branch: name, type (`classroom`, `lab`, `library`, `hall`, `office`), capacity | BM | R1 | M |
| FR-ORG-008 | Wing list per branch (`boys`, `girls`, `combined`) used by sections | PR | R1 | M |
| FR-ORG-009 | Structure tree view: organization, institutions, branches, departments, with counts | OW | R1 | S |
| FR-ORG-010 | Deactivate a branch (no new records, history retained); deletion only if the branch has no data | OW | R1 | M |
| FR-ORG-011 | Onboarding wizard: organization profile, institution, branches, first session, class template, fee heads, grading template, first admin users. Scenario A (single school) is created in one pass | OW, SA | R1 | M |

**Business rules**
- BR-ORG-01: Branch code unique within organization. Institution names unique within organization.
- BR-ORG-02: An organization always has at least one active institution and one active branch.
- BR-ORG-03: Branches with students, staff or transactions cannot be deleted, only deactivated.

**User stories**
- US-ORG-01: As a single-school OW, I finish the setup wizard in one sitting so that I can start admissions.
- US-ORG-02: As OW of a chain, I add a new branch mid-year so that it can start its own admissions and accounts.

**Acceptance criteria**
- AC-ORG-01: After the wizard, a single-school organization has one institution, one branch, one active session, classes, fee heads and a grading scheme, and the branch switcher is hidden.
- AC-ORG-02: Deleting a branch with one student is refused; deactivation succeeds and the branch disappears from write forms.

---

## 2.7 Identity (`identity`) | IAM | Core tier | R1

**Purpose:** Authentication, users, dynamic RBAC.
**Key entities:** `users`, `roles`, `permissions`, `role_permissions`, `user_role_assignments`, `user_permission_overrides`, `one_time_passwords`, `trusted_devices`, `personal_access_tokens`, `sessions`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-IAM-001 | Login on the organization subdomain with email, mobile number or username (students may use registration number) plus password | All | R1 | M |
| FR-IAM-002 | Password policy per organization (length, complexity, expiry optional, history of last 5) | OA, ITA | R1 | M |
| FR-IAM-003 | Account lockout after configurable failed attempts (default 5 in 15 minutes) for a configurable period; unlock by admin | ITA | R1 | M |
| FR-IAM-004 | Password reset by email link or SMS OTP; tokens single-use, expire in 15 minutes | All | R1 | M |
| FR-IAM-005 | Guardian activation: OTP sent to the registered mobile, guardian sets password; imported staff and students must change password at first login | GDN, all | R1 | M |
| FR-IAM-006 | Optional TOTP two-factor login; can be made mandatory per role (default mandatory for OW, ACT, SA) | All | R1 | S |
| FR-IAM-007 | Active sessions list per user with revoke; idle timeout per role | All, ITA | R1 | M |
| FR-IAM-008 | Permission catalog registered in code by each module and synced to `permissions` on deploy, grouped by module, screen and action | System | R1 | M |
| FR-IAM-009 | Role management: create custom roles, clone from system templates, edit permission matrix (module x screen x action grid) | OW, OA | R1 | M |
| FR-IAM-010 | Role assignment with scope: organization, institution or specific branches, plus visibility level (DEC-21); a user may hold several assignments | OW, OA, PR | R1 | M |
| FR-IAM-011 | User permission overrides: grant or deny specific permissions per user and scope, with reason and optional expiry; deny wins | OW, OA | R1 | M |
| FR-IAM-012 | Effective permission viewer: shows why a user can or cannot do an action (which role, which override) | OW, OA, ITA | R1 | M |
| FR-IAM-013 | Staff user accounts linked to employee records; guardian and student accounts created by Admission/Student/Guardian modules | OA, HRM | R1 | M |
| FR-IAM-014 | Context switcher: active branch and active role for users with several assignments | All staff | R1 | M |
| FR-IAM-015 | Deactivate a user (history kept, sessions revoked) | OA, ITA | R1 | M |
| FR-IAM-016 | Sanctum API tokens with scoped abilities for integrations and future apps | ITA | R1 | S |
| FR-IAM-017 | Trusted device registration for QR and online attendance (device fingerprint plus explicit registration, revocable) | Staff, ITA | R1 | M |

**Business rules**
- BR-IAM-01: Deny override beats any grant.
- BR-IAM-02: At least one active `org_owner` with full permissions must exist; removing the last one is refused.
- BR-IAM-03: Users cannot edit their own roles or overrides.
- BR-IAM-04: Permission changes take effect on the user's next request (cache invalidated on change).
- BR-IAM-05: Student accounts are created only for classes at or above the configured threshold (A-09).
- BR-IAM-06: Login identifier is unique per organization (DEC-09, DEC-22).

**User stories**
- US-IAM-01: As OW, I create a "Fee Clerk" role with only voucher printing and payment entry so that my clerk can't see salaries.
- US-IAM-02: As OA, I give a teacher temporary permission to enter marks for a section for one week while its teacher is on leave.

**Acceptance criteria**
- AC-IAM-01: A user with a deny override for `fee.payment.create` gets 403 on the endpoint even when a role grants it.
- AC-IAM-02: The sixth failed login within 15 minutes locks the account and creates a `login_events` record.
- AC-IAM-03: A branch-scoped principal of Branch A gets 404 when opening a Branch B student URL.
- AC-IAM-04: Removing the last owner's role assignment is refused.

---

## 2.8 Settings (`settings`) | SET | Core tier | R1

**Purpose:** Layered configuration, localization and branding.
**Key entities:** `settings`, `translation_overrides`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-SET-001 | Setting definitions registered in code with key, type, default, allowed scopes and validation | System | R1 | M |
| FR-SET-002 | Layered values: platform default, organization, institution, branch; UI shows "inherited from" and allows reset to inherited | OA, PR | R1 | M |
| FR-SET-003 | Categories: general, localization, academic defaults, attendance defaults, fee defaults, notification defaults, security, branding, working days and school timing | OA | R1 | M |
| FR-SET-004 | Branding: logos, document header and footer, signature and stamp images with signatory titles, accent color | OA | R1 | M |
| FR-SET-005 | Language: enable Urdu, organization default language, user language preference | OA, all | R1 | M |
| FR-SET-006 | Terminology overrides (e.g., "Campus" instead of "Branch") through `translation_overrides` | OA | R1 | S |
| FR-SET-007 | Every settings change audited with old and new value | System | R1 | M |

**Business rules**
- BR-SET-01: Unknown keys or values failing type validation are rejected.
- BR-SET-02: Security settings can only be tightened at branch level, never loosened below organization level.

**Acceptance criteria**
- AC-SET-01: A branch override of late threshold applies only to that branch; resetting restores the organization value.
- AC-SET-02: Switching a user to Urdu renders the layout RTL with Urdu labels on every R1 guardian screen.

---

## 2.9 Audit (`audit`) | AUD | Core tier | R1

**Purpose:** Tamper-resistant record of important actions.
**Key entities:** `audit_logs`, `login_events`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-AUD-001 | Automatic audit of: login, logout, failed login, password change, profile changes, student and employee updates, attendance changes and approvals, voucher generation, payments, refunds, reversals, marks entry and changes, result publication, document upload and deletion, role and permission changes, transfers, promotions, imports, exports, settings changes, support access | System | R1 | M |
| FR-AUD-002 | Entry fields: user, impersonator (if any), organization, branch, action key, module, record type, record id, old values, new values, reason, IP, user agent, timestamp, correlation id | System | R1 | M |
| FR-AUD-003 | Sensitive values masked in logs (passwords and tokens never stored; CNIC and B-Form masked except last 4 digits) | System | R1 | M |
| FR-AUD-004 | Audit viewer with filters: user, module, action, record, date range, branch, IP | OW, OA, ITA | R1 | M |
| FR-AUD-005 | "History" tab on major records (student, employee, voucher, payment, marks, result, attendance day) | MGT, ACT | R1 | M |
| FR-AUD-006 | Login history per user; repeated failures alert IT admin | ITA | R1 | M |
| FR-AUD-007 | Audit export (gated by `audit.log.export`, itself audited) | OW, ITA | R1 | S |
| FR-AUD-008 | Retention: financial audit 7 years minimum, others configurable; archival to cold storage | System | R3 | S |

**Business rules**
- BR-AUD-01: No application path updates or deletes audit records. The production database user for the app gets INSERT and SELECT only on audit tables (D12).
- BR-AUD-02: For Fee, Accounting, marks and attendance corrections, the audit entry is written inside the same transaction as the change.
- BR-AUD-03: Low-risk events (views, exports) are written after commit through the queue.

**Acceptance criteria**
- AC-AUD-01: Changing a student's date of birth creates one entry with old and new values and the reason.
- AC-AUD-02: No route exists that modifies `audit_logs`; a test asserts this.
- AC-AUD-03: A failed transaction leaves no audit entry for the rolled-back change.

---

## 2.10 Approval (`approval`) | APR | Core tier | R1, R2

**Purpose:** One configurable workflow engine for every request type.
**Key entities:** `approval_workflows`, `approval_workflow_steps`, `approval_requests`, `approval_request_steps`, `approval_actions`, `approval_delegations`

**Request types registered in R1:** `attendance_correction`, `missing_punch`, `student_leave`, `staff_leave`, `student_profile_correction`, `student_modification`, `academic_correction`, `marks_revision`, `result_publication`, `fee_concession`, `scholarship`, `fee_refund`, `payment_reversal`, `cash_variance`, `period_unlock`, `branch_transfer`, `withdrawal`, `document_verification`, `journal_entry`, `custom`. R2 adds `expense`, `purchase`, `payroll_run`, `result_recheck`.

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-APR-001 | Workflow definition per request type per organization, with optional branch-specific override | OW, OA | R1 | M |
| FR-APR-002 | Step approver resolvers: role in requester's branch, specific user, guardian of the student, class teacher of the student's section, HOD of the subject's department, requester's reporting manager | OW, OA | R1 | M |
| FR-APR-003 | Step modes: any one approver, or all approvers | OW, OA | R1 | M |
| FR-APR-004 | Conditions: amount thresholds (e.g., concession above 20% adds owner step), attribute rules (request category, days count) | OW, OA | R1 | M |
| FR-APR-005 | Skip rules (e.g., skip guardian step when the student is 18 or older or in a college program) | OW, OA | R1 | M |
| FR-APR-006 | Submit request: category, description, attachments, payload of proposed change; request number from numbering service | All | R1 | M |
| FR-APR-007 | Statuses: `draft`, `submitted`, `in_review`, `returned`, `approved`, `rejected`, `cancelled`, `expired` | System | R1 | M |
| FR-APR-008 | Actions: approve, reject, return for changes; comment mandatory on reject and return | Approvers | R1 | M |
| FR-APR-009 | On final approval the owning module's handler applies the change in a transaction and writes audit | System | R1 | M |
| FR-APR-010 | Inboxes in every portal: "My requests" and "Pending my approval", with counts on dashboards | All | R1 | M |
| FR-APR-011 | Approval timeline shown on the request and on the affected record | All | R1 | M |
| FR-APR-012 | Delegation: an approver on leave delegates to another eligible user for a date range | MGT | R1 | S |
| FR-APR-013 | Bulk approve for low-risk types (leave) with permission | PR, HRM | R1 | S |
| FR-APR-014 | SLA timers and escalation to the next role after N hours | OW, OA | R2 | M |

**Business rules**
- BR-APR-01: A requester can never approve their own request; the step moves to the next eligible approver.
- BR-APR-02: Handlers are idempotent. If the target record changed since submission (version mismatch), approval fails as "stale" and the request returns to the requester.
- BR-APR-03: Notifications fire at submission, each step decision and final outcome.
- BR-APR-04: If no approver can be resolved, the request goes to the branch principal and is flagged.

**User stories**
- US-APR-01: As CT, I request a correction for yesterday's attendance so that a wrongly marked student is fixed with principal approval.
- US-APR-02: As GDN, I approve my son's request to update his profile photo so that the school can apply it.

**Acceptance criteria**
- AC-APR-01: An attendance correction approved by PR updates the attendance record, keeps the original version, writes audit, and notifies the teacher.
- AC-APR-02: A principal who submits a fee concession cannot approve it.
- AC-APR-03: A concession of 30% routes to the owner step; 10% does not.

---

## 2.11 Notification (`notification`) | NTF | Core tier | R1 to R3

**Purpose:** Central engine for in-app, email, SMS, WhatsApp and push messages.
**Key entities:** `notification_templates`, `notification_providers`, `notification_rules`, `notification_dispatches`, `notification_delivery_attempts`, `notification_preferences`, `notifications`

**Event catalog (R1):** student absent, student late, attendance not marked by teacher, voucher generated, payment received, fee overdue, struck-off warning, exam date sheet published, admit cards available, result published, diary published, announcement, approval requested, approval decided, leave decided, password reset, account activation, document expiring. R2 adds homework assigned, homework deadline, quiz scheduled, lecture uploaded, PTM scheduled, message received.

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-NTF-001 | Rules: event, channels, recipient resolvers (guardians of student, class teacher, approvers, role in branch, specific users), enabled flag | OA | R1 | M |
| FR-NTF-002 | Templates per event per channel per language with placeholders, preview, and SMS segment counter | OA | R1 | M |
| FR-NTF-003 | Provider configuration per organization (SMTP, SMS aggregator; WhatsApp in R2) with encrypted credentials and test send | OA, ITA | R1 | M |
| FR-NTF-004 | Queued sending with per-provider rate limits | System | R1 | M |
| FR-NTF-005 | Retry with backoff (3 attempts) and optional fallback SMS provider | System | R1 | M |
| FR-NTF-006 | Delivery status updates from provider callbacks where supported | System | R1 | S |
| FR-NTF-007 | Delivery log searchable by recipient, event, channel, status | OA, ITA | R1 | M |
| FR-NTF-008 | User preferences per channel for non-critical events; critical events (absence, fee overdue, OTP) cannot be turned off | All | R1 | M |
| FR-NTF-009 | Quiet hours (default 21:00 to 07:00) for SMS and WhatsApp; OTP and urgent bypass | OA | R1 | M |
| FR-NTF-010 | Deduplication: same event for same recipient and record within a window is sent once | System | R1 | M |
| FR-NTF-011 | Manual broadcast with audience selection and cost estimate before sending | MGT | R1 | M |
| FR-NTF-012 | SMS credit quota per organization; warning at 80%, block at 100% except OTP | System | R1 | M |
| FR-NTF-013 | In-app notification center with unread count and mark-as-read | All | R1 | M |
| FR-NTF-014 | WhatsApp channel through approved provider with pre-approved templates | OA | R2 | M |
| FR-NTF-015 | Push notifications through FCM for future apps | System | R3 | S |

**Business rules**
- BR-NTF-01: Notifications are generated only after the business transaction commits.
- BR-NTF-02: Message content is snapshotted in the dispatch log.
- BR-NTF-03: Messages never contain passwords, full CNIC or full B-Form numbers.

**Acceptance criteria**
- AC-NTF-01: Finalizing attendance with 3 absentees sends exactly one SMS per primary guardian per absent student.
- AC-NTF-02: An SMS generated at 22:30 for a non-urgent event is held until 07:00.
- AC-NTF-03: A provider failure retries three times then marks the attempt `failed` with the provider error.

---

## 2.12 ImportExport (`import_export`) | IMX | Core tier | R1, R2

**Purpose:** Reusable bulk import and export engine.
**Key entities:** `import_jobs`, `import_job_rows`, `export_jobs`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-IMX-001 | Import types registry. R1: students, guardians, families, employees, classes and sections, subjects, opening fee balances, opening ledger balances, attendance, marks. R2: all others | System | R1 | M |
| FR-IMX-002 | Downloadable Excel templates with instructions sheet and dropdown validation | All importers | R1 | M |
| FR-IMX-003 | Upload XLSX or CSV; column mapping UI; saved mappings per organization | All importers | R1 | M |
| FR-IMX-004 | Validation pass with preview: valid rows, warnings and errors per row and column | All importers | R1 | M |
| FR-IMX-005 | Duplicate detection rules per type (B-Form, CNIC, admission number, mobile plus name); options to skip or update where allowed | All importers | R1 | M |
| FR-IMX-006 | Commit as a queued job in chunks, each chunk in a transaction | System | R1 | M |
| FR-IMX-007 | Failed rows downloadable with an error column | All importers | R1 | M |
| FR-IMX-008 | Import history: who, when, file, counts, status | MGT, ITA | R1 | M |
| FR-IMX-009 | Export jobs for lists and reports; links expire after 24 hours; export parameters audited | All exporters | R1 | M |
| FR-IMX-010 | Undo an import batch within 24 hours when no dependent records exist | MGT | R2 | C |

**Business rules**
- BR-IMX-01: Imports run with the importing user's permissions and scopes; rows outside scope are rejected.
- BR-IMX-02: Limits per plan (default 10 MB or 20,000 rows per file).

**Acceptance criteria**
- AC-IMX-01: Importing 5,000 students with 12 invalid rows imports 4,988 and offers a 12-row error file.
- AC-IMX-02: A branch-scoped user importing a row for another branch gets that row rejected.

---

## 2.13 Document (`document`) | DOC | Core tier | R1 to R3

**Purpose:** Branded PDF generation and an issued-document registry.
**Key entities:** `document_templates`, `issued_documents`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-DOC-001 | Template types: fee voucher/challan, receipt, report card, admit card, student ID card, staff ID card, leaving certificate, character certificate, migration certificate, admission form, attendance sheet, date sheet, salary slip (R2), financial reports | OA | R1 | M |
| FR-DOC-002 | Default templates shipped; per-institution branding (logo, name, address, signatures, stamp) | OA | R1 | M |
| FR-DOC-003 | Paper layouts: A4, Legal, 3-copy challan on A4 landscape, CR80 ID card sheets | OA | R1 | M |
| FR-DOC-004 | Bilingual labels (English/Urdu) per template | OA | R1 | M |
| FR-DOC-005 | Rendering through headless Chromium (DEC-12); single documents synchronous, bulk queued | System | R1 | M |
| FR-DOC-006 | Bulk merged PDF (e.g., all vouchers for a class) with progress | ACT, BM | R1 | M |
| FR-DOC-007 | Issued document registry for certificates: serial number, verification code, content snapshot | BM, PR | R1 | M |
| FR-DOC-008 | Reprints marked "Duplicate" with reprint count | System | R1 | M |
| FR-DOC-009 | Public QR verification page showing minimal data (name, document type, date, valid/revoked) | Public | R2 | S |
| FR-DOC-010 | Template designer (drag fields, sizes) | OA | R3 | C |

**Business rules**
- BR-DOC-01: Issued certificate content is immutable. Reissue creates a new version linked to the old one, and the old one shows as superseded.
- BR-DOC-02: Leaving certificate issuance requires fee clearance unless an approved waiver exists.

**Acceptance criteria**
- AC-DOC-01: A report card with Urdu name and remarks renders with correct Nastaliq shaping.
- AC-DOC-02: A bulk voucher PDF for 1,000 students completes in the queue and is downloadable from the notification.

---

## 2.14 Academic (`academic`) | ACD | Core tier | R1

**Purpose:** Configurable academic structure, the foundation for everything else.
**Key entities:** `academic_sessions`, `education_levels`, `study_groups`, `study_group_subjects`, `subjects`, `academic_classes`, `sections`, `class_subjects`, `teaching_assignments`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-ACD-001 | Academic sessions per institution: name (e.g., 2026-2027), start and end dates, admission window, exam window, promotion date, status `planning`, `active`, `closed` | OW, PR | R1 | M |
| FR-ACD-002 | Session closing checklist (results published, promotions done, fee balances carried forward) and close action that makes the session read-only | PR, OW | R1 | M |
| FR-ACD-003 | New session wizard: copy classes, sections, class subjects, teaching assignments (optional) and fee structures from the previous session | PR, AC | R1 | M |
| FR-ACD-004 | Education levels per institution: name (en/ur), order, stage (`pre_primary`, `primary`, `middle`, `secondary`, `higher_secondary`), next level for promotion, board-exam flag, active flag. Seeded with Playgroup to Intermediate Part 2 | OA, AC | R1 | M |
| FR-ACD-005 | Study groups (matric: Science, Arts/Humanities, Computer Science; intermediate: F.Sc Pre-Medical, F.Sc Pre-Engineering, ICS, I.Com, FA) with applicable levels | OA, AC | R1 | M |
| FR-ACD-006 | Study group subjects per level: compulsory or elective, minimum and maximum electives | AC | R1 | M |
| FR-ACD-007 | Subjects: name (en/ur), code, type (`theory`, `practical`, `theory_practical`), board subject flag, department | AC, HOD | R1 | M |
| FR-ACD-008 | Academic classes per branch per session per level, optionally per study group (e.g., "Class 9 Science") | PR, AC | R1 | M |
| FR-ACD-009 | Sections: name, capacity, shift, wing, class teacher, room | PR, AC | R1 | M |
| FR-ACD-010 | Class subjects: subjects offered for a class, periods per week, compulsory or elective, default marks components | AC | R1 | M |
| FR-ACD-011 | Teaching assignments: employee, section, subject, session, periods per week, role (`primary`, `co_teacher`), effective dates | AC, PR | R1 | M |
| FR-ACD-012 | Teacher workload view with maximum periods warning | AC, PR, HOD | R1 | M |
| FR-ACD-013 | Section capacity enforcement: warning or hard limit (setting) | PR | R1 | M |
| FR-ACD-014 | Inter-branch teaching assignments when the employee has an active branch assignment there | OA | R1 | S |
| FR-ACD-015 | Board exam level mapping: which levels sit board exams and with which board | OA | R2 | S |

**Business rules**
- BR-ACD-01: DEC-26: one active and at most one planning session per institution.
- BR-ACD-02: Levels, groups and subjects in use cannot be deleted, only deactivated.
- BR-ACD-03: Closed sessions are read-only for everyone except via `period_unlock` approval.
- BR-ACD-04: A teaching assignment requires an employee active in that branch on those dates.
- BR-ACD-05: Each academic class belongs to exactly one session and one branch.

**User stories**
- US-ACD-01: As AC, I copy last year's classes and subjects into the new session so that setup takes minutes.
- US-ACD-02: As PR, I assign Physics for 9-A and 9-B to one teacher so that the timetable and marks entry know who teaches what.

**Acceptance criteria**
- AC-ACD-01: Activating a second session while one is active is refused.
- AC-ACD-02: The copy wizard creates new class, section and class-subject rows for the new session without changing the old session's rows.
- AC-ACD-03: Assigning a teacher beyond the maximum periods shows a warning and records who overrode it.

---

## 2.15 Admission (`admission`) | ADM | Standard tier | R1, R2

**Purpose:** From inquiry and application to an enrolled student with linked guardians.
**Key entities:** `admission_inquiries`, `admission_form_templates`, `admission_applications`, `admission_assessments`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-ADM-001 | Admission form templates per institution: fields from student and guardian schema plus custom fields, required flags, document checklist | OA, ADO | R1 | M |
| FR-ADM-002 | Public online application page per branch with mobile OTP verification, captcha, document upload and application number | Public | R1 | M |
| FR-ADM-003 | Offline application entry at the office | ADO, REC | R1 | M |
| FR-ADM-004 | Pipeline statuses: `submitted`, `under_review`, `test_scheduled`, `interview_scheduled`, `approved`, `waitlisted`, `rejected`, `admitted`, `withdrawn` | ADO, PR | R1 | M |
| FR-ADM-005 | Review screen with document verification checklist | ADO | R1 | M |
| FR-ADM-006 | Duplicate check on B-Form, name plus date of birth plus father name; sibling detection by guardian CNIC or mobile | System | R1 | M |
| FR-ADM-007 | Approval generates admission fee voucher (Fee) | ADO, PR | R1 | M |
| FR-ADM-008 | Conversion to student after payment (or override with permission): create student, enrollment, guardian links, family, registration, GR, admission and roll numbers, section allocation, guardian portal activation SMS | ADO | R1 | M |
| FR-ADM-009 | Section allocation manual or automatic (capacity, balanced gender) | ADO, PR | R1 | S |
| FR-ADM-010 | Seat availability per class and section | ADO, PR | R1 | M |
| FR-ADM-011 | GR register printable report | BM | R1 | M |
| FR-ADM-012 | Bulk student import (via IMX) | ADO, OA | R1 | M |
| FR-ADM-013 | Admission tests and interviews: schedule, slots, result entry | ADO, PR | R2 | M |
| FR-ADM-014 | Inquiries: source, contact, class sought, follow-up dates, conversion tracking | ADO, REC | R2 | M |
| FR-ADM-015 | Rejected applications anonymized after a retention period | System | R2 | S |

**Business rules**
- BR-ADM-01: New admissions require an open admission window unless the user has `admission.application.override_window`.
- BR-ADM-02: Conversion is one transaction: either student, enrollment, guardians and numbers are all created, or nothing is.
- BR-ADM-03: An existing guardian (matching CNIC) is linked, never duplicated.

**User stories**
- US-ADM-01: As a parent, I apply online for Class 1 from my phone so that I don't have to visit twice.
- US-ADM-02: As ADO, I convert a paid application into a student in one step so that the class teacher sees the child the next morning.

**Acceptance criteria**
- AC-ADM-01: Converting an application where the father already has a child in the school links both children into one family.
- AC-ADM-02: If number generation fails mid-conversion, no student or guardian record remains.

---

## 2.16 Student (`student`) | STU | Core tier | R1

**Purpose:** Student records and full academic lifecycle with history preserved.
**Key entities:** `students`, `student_enrollments`, `student_subjects`, `student_timeline_events`, `student_transfers`, `student_withdrawals`, `student_categories`, `sponsors`, `student_health_profiles`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-STU-001 | Student profile tabs: personal, guardians, academic, health, documents, fees, attendance, results, discipline, learning activity, timeline, history | MGT, TCH (limited) | R1 | M |
| FR-STU-002 | Identifiers: registration number (organization-unique, permanent), GR number (branch), admission number, roll number per enrollment | System | R1 | M |
| FR-STU-003 | Personal fields per brief plus optional Urdu names for student and father | ADO, BM | R1 | M |
| FR-STU-004 | Photo upload with crop and compression | ADO, BM | R1 | M |
| FR-STU-005 | Category (e.g., regular, staff child, sponsored) and sponsor | ADO, ACT | R1 | M |
| FR-STU-006 | Statuses: `active`, `suspended`, `struck_off`, `withdrawn`, `transferred_out`, `graduated` | System | R1 | M |
| FR-STU-007 | Search within scope by name, father name, B-Form, mobile, any identifier | Staff | R1 | M |
| FR-STU-008 | Timeline events created automatically: admission, enrollment, promotion, transfer, status change, fee milestones, results, discipline, certificates | System | R1 | M |
| FR-STU-009 | Protected fields (name, DOB, B-Form, father name, admission date) editable only with `student.profile.update_protected` and reason | BM, PR | R1 | M |
| FR-STU-010 | Profile correction and modification requests from guardian or student through APR | GDN, STD | R1 | M |
| FR-STU-011 | Documents with verification status (pending, verified, rejected) | ADO | R1 | M |
| FR-STU-012 | Elective subject selection per enrollment within study group rules | AC, ADO | R1 | M |
| FR-STU-013 | Bulk promotion per class: eligibility from results (pass, compartment, fail), manual override with reason, target section mapping; creates next-session enrollments and closes current ones | PR, AC | R1 | M |
| FR-STU-014 | Retention (repeat class) as a new enrollment in the same level | PR | R1 | M |
| FR-STU-015 | Section and class transfer within session with effective date; enrollment history kept | PR, BM | R1 | M |
| FR-STU-016 | Branch transfer within organization through approval: closes enrollment in source, opens in target, moves fee balance per policy | PR, OW | R1 | M |
| FR-STU-017 | Withdrawal with clearance checklist (fees, library, transport, documents returned), reason, leaving certificate | BM, PR | R1 | M |
| FR-STU-018 | Struck-off processing (list from Fee rule, principal confirms) and re-admission | PR, ACT | R1 | M |
| FR-STU-019 | Graduation of final-level students at session close | PR | R1 | M |
| FR-STU-020 | Bulk actions: assign section, change category, print ID cards | BM | R1 | M |
| FR-STU-021 | Health profile (blood group, conditions, allergies, emergency instructions) visible only with `student.health.view` | BM, PR | R1 | M |
| FR-STU-022 | Character, leaving and migration certificates issued via DOC | BM, PR | R1 | M |

**Business rules**
- BR-STU-01: A student has at most one active enrollment per session, and is never enrolled in two branches at once.
- BR-STU-02: Enrollment rows are never deleted or overwritten; changes close a row and open a new one.
- BR-STU-03: B-Form unique within organization when present.
- BR-STU-04: Withdrawal is blocked while dues exist unless an approved waiver exists.
- BR-STU-05: Teachers see only students in their assigned sections (visibility `assigned`).

**User stories**
- US-STU-01: As PR, I promote all of Class 5 in one action, keeping 3 failed students back, so that next session starts with correct rolls.
- US-STU-02: As BM, I issue a leaving certificate only after the system confirms fees and library are clear.

**Acceptance criteria**
- AC-STU-01: After promotion, the student's 2026-2027 enrollment is `completed` and a 2027-2028 enrollment exists; old marks and attendance still show under the old enrollment.
- AC-STU-02: A class teacher of 6-A gets 404 for a 6-B student.
- AC-STU-03: Editing date of birth without the protected permission returns 403.

---

## 2.17 Guardian (`guardian`) | GRD | Core tier | R1, R2

**Purpose:** Guardians, families, guardian-student links and the guardian portal's data access.
**Key entities:** `guardians`, `guardian_students`, `families`, `pickup_authorizations`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-GRD-001 | Guardian profile: name (en/ur), relation, CNIC, primary and secondary mobiles, email, occupation, address | ADO, BM | R1 | M |
| FR-GRD-002 | Link to students with relation and flags: primary contact, fee payer, emergency contact, portal access, can approve requests, pickup allowed | ADO, BM | R1 | M |
| FR-GRD-003 | Custody restriction flag: no portal access and no pickup, with document and reason (restricted visibility) | PR | R1 | M |
| FR-GRD-004 | Families formed automatically by shared guardian CNIC; manual merge and split with permission | ADO, ACT | R1 | M |
| FR-GRD-005 | Family view: siblings, combined dues, family voucher setting | ACT, GDN | R1 | M |
| FR-GRD-006 | Guardian verification: by office, or by code printed on admission slip or voucher | ADO | R1 | M |
| FR-GRD-007 | Guardian portal account per DEC-22 (shared mobile links all related guardian records to one user) | System | R1 | M |
| FR-GRD-008 | Guardian portal data: children list and switcher, attendance and alerts, results and report cards, daily diary, timetable, vouchers, payment history, announcements, leave application for child, correction requests, approval of student requests, disciplinary notices | GDN | R1 | M |
| FR-GRD-009 | Pickup authorizations: names, CNIC, photo, relation | GDN, BM | R1 | S |
| FR-GRD-010 | Guardian portal R2 additions: online payment, homework and lectures, messaging, PTM booking | GDN | R2 | M |

**Business rules**
- BR-GRD-01: A guardian sees only students linked with an active link that has portal access.
- BR-GRD-02: Every student has at least one primary guardian.
- BR-GRD-03: Unlinking requires `guardian.link.delete` and a reason.

**User stories**
- US-GRD-01: As a mother with three children in two branches, I switch between them in one account so that I don't need three logins.
- US-GRD-02: As PR, I flag a father with a court order so that he cannot log in or collect the child.

**Acceptance criteria**
- AC-GRD-01: A guardian requesting another family's student URL gets 404.
- AC-GRD-02: A custody-restricted guardian's login is refused with a neutral message and the attempt is logged.

---

## 2.18 HR (`hr`) | HR | Standard tier | R1, R2

**Purpose:** Employee and teacher records.
**Key entities:** `employees`, `designations`, `employee_branch_assignments`, `employee_qualifications`, `employee_experiences`, `employee_contracts`, `performance_reviews`, `training_records`, `employee_disciplinary_actions`, `employee_separations`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-HR-001 | Employee profile per brief: personal, contact, CNIC, emergency contact, qualifications, experience, specialization, joining date, employment type, department, designation, photo | HRM, OA | R1 | M |
| FR-HR-002 | Employee ID from numbering service | System | R1 | M |
| FR-HR-003 | Employment types: `permanent`, `contract`, `visiting`, `part_time`, `daily_wage` | HRM | R1 | M |
| FR-HR-004 | Primary branch plus additional branch assignments with effective dates (shared employees) | HRM, OA | R1 | M |
| FR-HR-005 | Teaching flag and subject specializations | HRM | R1 | M |
| FR-HR-006 | Reporting manager, shift and work schedule | HRM | R1 | M |
| FR-HR-007 | Documents (CV, degrees, CNIC copy, contract) with expiry dates and reminders | HRM | R1 | M |
| FR-HR-008 | Statuses: `active`, `on_leave`, `suspended`, `resigned`, `terminated`, `retired` | HRM | R1 | M |
| FR-HR-009 | User account creation for employee with role suggestion by designation | HRM, OA | R1 | M |
| FR-HR-010 | Staff directory with contact privacy settings | Staff | R1 | S |
| FR-HR-011 | Self-service: view profile, request profile correction (APR), view attendance and leave balance | Staff | R1 | M |
| FR-HR-012 | Contracts with terms, start, end, renewal reminders | HRM | R2 | M |
| FR-HR-013 | Performance review cycles, criteria, ratings, comments, employee acknowledgment | HRM, PR | R2 | M |
| FR-HR-014 | Training records | HRM | R2 | S |
| FR-HR-015 | Disciplinary records (restricted permission) | HRM, PR | R2 | M |
| FR-HR-016 | Separation workflow with clearance (assets, library, dues) and final settlement link to Payroll | HRM | R2 | M |

**Business rules**
- BR-HR-01: CNIC unique within organization.
- BR-HR-02: Employees with any transaction cannot be deleted.
- BR-HR-03: Salary fields visible only with `hr.salary.view`.

**Acceptance criteria**
- AC-HR-01: An employee assigned to Branch A and B appears in both branches' staff lists and attendance.
- AC-HR-02: A CNIC expiring in 30 days triggers a reminder to HRM.

---

## 2.19 Attendance (`attendance`) | ATT | Core tier | R1 to R3

**Purpose:** Central attendance engine for students and staff from multiple sources.
**Key entities:** `attendance_policies`, `attendance_devices`, `device_enrollments`, `attendance_punches`, `student_attendances`, `student_period_attendances`, `staff_attendances`, `attendance_corrections`, `qr_attendance_sessions`, `device_sync_logs`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-ATT-001 | Policies per branch, shift and person type: start time, grace minutes, late threshold, half-day threshold, early departure threshold, auto-absent cut-off, lates-to-leave conversion, official sources | PR, HRM | R1 | M |
| FR-ATT-002 | Teacher marks daily student attendance per section: all present by default, tap to change status, bulk actions, remarks | CT | R1 | M |
| FR-ATT-003 | Marking window (e.g., until 11:00); after it the day is finalized and changes need a correction | System | R1 | M |
| FR-ATT-004 | Optional period-wise (subject) attendance per class, mainly for colleges | ST | R1 | S |
| FR-ATT-005 | "Not marked" alert to class teacher and principal at a set time | System | R1 | M |
| FR-ATT-006 | Staff online check-in and check-out with trusted device and optional geofence (branch coordinates and radius) | Staff | R1 | M |
| FR-ATT-007 | QR attendance: kiosk shows a code that rotates every 30 seconds; staff scan with a logged-in trusted phone; codes expire and are single use per person per session. Teachers can also show a class QR for college lectures | Staff, STD | R1 | M |
| FR-ATT-008 | Device registry: serial, vendor, model, branch, location, adapter type, secret, timezone, status, last heartbeat | ITA | R1 | M |
| FR-ATT-009 | Device enrollment mapping: device user ID to student or employee | ITA, HRM | R1 | M |
| FR-ATT-010 | ZKTeco ADMS push endpoint (handshake, punch upload, heartbeat) | System | R1 | M |
| FR-ATT-011 | Signed webhook adapter (HMAC) for other devices or middleware | System | R1 | M |
| FR-ATT-012 | CSV/Excel punch log import | ITA, HRM | R1 | M |
| FR-ATT-013 | Raw punches stored immutably with source and verification type (fingerprint, face, card, PIN, QR) | System | R1 | M |
| FR-ATT-014 | Resolver job computes daily `staff_attendances` and device-based `student_attendances` using DEC-25 precedence; calculates late minutes and early departure | System | R1 | M |
| FR-ATT-015 | Offline device alert when no heartbeat or punch for X hours on a working day | ITA | R1 | M |
| FR-ATT-016 | Guardian absence and late alerts after finalization (configurable time) | System | R1 | M |
| FR-ATT-017 | Corrections: teacher requests for finalized student attendance; staff requests for missing punch or correction; through APR | TCH, Staff | R1 | M |
| FR-ATT-018 | Approved corrections create a new version; the original stays visible in history | System | R1 | M |
| FR-ATT-019 | Authorized administrative attendance entry with mandatory reason | PR, HRM | R1 | M |
| FR-ATT-020 | Month lock through period locks | PR, HRM | R1 | M |
| FR-ATT-021 | Reports: daily, monthly register grid, student, teacher, branch, class, section, late arrivals, early departures, absentees, missing punches, correction history, percentage, device sync logs; all with filters, print, PDF, Excel | MGT, HRM, TCH | R1 | M |
| FR-ATT-022 | Suspicious pattern alerts: many identities on one device within seconds, check-in from a new device, attempts outside geofence, manual overrides above a threshold | PR, ITA | R1 | S |
| FR-ATT-023 | On-premise sync agent for LAN-only devices (pull, then push to cloud) | ITA | R2 | M |
| FR-ATT-024 | RFID reader support through adapters | ITA | R2 | S |
| FR-ATT-025 | Push user lists to devices (ADMS command queue) | ITA | R2 | S |
| FR-ATT-026 | Offline-capable attendance marking that syncs later | CT | R3 | S |

**Business rules**
- BR-ATT-01: One `student_attendances` row per student per date; one `staff_attendances` row per employee per date.
- BR-ATT-02: Raw punches unique on device, device user and punch time; duplicates within a configurable window (default 2 minutes) are collapsed.
- BR-ATT-03: Approved leave sets `LV` automatically; holidays and weekends set `H` and `W` unless an override working day exists.
- BR-ATT-04: Teachers mark only sections they are class teacher of (daily) or teach (period-wise).
- BR-ATT-05: Finalized attendance changes only through approved corrections or authorized administrative entry with reason.
- BR-ATT-06: The product never claims GPS or QR alone prevents proxy attendance. Anti-proxy controls reduce risk and flag suspicious cases.

**User stories**
- US-ATT-01: As CT, I mark 38 students on my phone in under a minute so that parents of the 2 absentees get an SMS by 10:30.
- US-ATT-02: As HRM, I see who forgot to punch out yesterday so that I can send missing-punch reminders.

**Acceptance criteria**
- AC-ATT-01: The same punch posted twice by a device creates one raw punch record.
- AC-ATT-02: A device punch says present, then the teacher marks absent: the official status is absent (teacher wins), and both are visible.
- AC-ATT-03: Editing a finalized day without a correction returns 403.
- AC-ATT-04: A punch arriving 2 days late (offline device) recalculates that day's staff attendance if the month is not locked.

---

## 2.20 Leave (`leave`) | LVE | Core tier | R1, R2

**Purpose:** Leave types, policies, balances and applications for staff and students.
**Key entities:** `leave_types`, `leave_policies`, `leave_balances`, `leave_applications`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-LVE-001 | Leave types: casual, sick, annual, maternity, hajj, unpaid, short leave, student sick, student family; applicability (staff/students), paid or unpaid, document required after N days | HRM, PR | R1 | M |
| FR-LVE-002 | Policies by employment type or designation: annual quota, accrual (monthly/yearly), carry forward cap, half-day and short leave allowed, sandwich rule (weekends between leave days counted) | HRM | R1 | M |
| FR-LVE-003 | Balances per year with opening balance entry | HRM | R1 | M |
| FR-LVE-004 | Apply: staff self-service, guardian for child, admin on behalf; date range, half day, reason, attachment | Staff, GDN, HRM | R1 | M |
| FR-LVE-005 | Checks: overlap, balance, locked month | System | R1 | M |
| FR-LVE-006 | Approval through APR; approved leave updates attendance | System | R1 | M |
| FR-LVE-007 | Cancel before start date; cancellation request after start | Staff, GDN | R1 | M |
| FR-LVE-008 | "On leave today" view and leave calendar | PR, HRM | R1 | M |
| FR-LVE-009 | Leave register and balance reports | HRM | R1 | M |
| FR-LVE-010 | Substitution prompt for teacher leave (Timetable) | AC | R2 | S |
| FR-LVE-011 | Leave encashment rules feeding Payroll | HRM | R2 | C |

**Business rules**
- BR-LVE-01: Unpaid leave days are passed to Payroll as deduction inputs (R2).
- BR-LVE-02: Balance never goes negative unless the policy allows advance leave.

**Acceptance criteria**
- AC-LVE-01: Approved 3-day leave sets `LV` for those days in staff attendance.
- AC-LVE-02: With sandwich rule on, leave on Friday and Monday counts 4 days.

---

## 2.21 Fee (`fee`) | FEE | Core tier | R1, R2, R3

**Purpose:** Fee configuration, vouchers, collection, concessions and fee ledgers.
**Key entities:** `fee_heads`, `fee_structures`, `fee_structure_items`, `fee_discount_rules`, `student_fee_concessions`, `scholarships`, `installment_plans`, `fee_vouchers`, `fee_voucher_items`, `fee_payments`, `fee_payment_allocations`, `fee_refunds`, `cashier_sessions`, `payment_gateway_accounts`, `payment_gateway_transactions`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-FEE-001 | Fee heads: name (en/ur), code, type (`monthly`, `annual`, `one_time`, `on_demand`, `refundable_deposit`, `fine`), ledger account mapping, taxable flag | ACT, OA | R1 | M |
| FR-FEE-002 | Fee structures per institution, branch, session, class, with optional section, study group and student category; items with amount and applicable months | ACT, OW | R1 | M |
| FR-FEE-003 | Effective-dated revisions within a session (mid-year fee increase) | ACT, OW | R1 | M |
| FR-FEE-004 | Vacation fee rules: collect June and July (configurable months) with an earlier voucher | ACT | R1 | M |
| FR-FEE-005 | Category rules (e.g., staff child 50% tuition, sponsored 100%) | ACT | R1 | M |
| FR-FEE-006 | Sibling discount rules (2nd child X%, 3rd child Y%, on selected heads) | ACT | R1 | M |
| FR-FEE-007 | Individual concessions: percentage or fixed, per head, date range, reason; approval above threshold | ACT, PR | R1 | M |
| FR-FEE-008 | Scholarships: type, sponsor, amount or percentage, period, approval | PR, ACT | R1 | M |
| FR-FEE-009 | Bulk monthly voucher generation by branch, class, section or all: current heads, arrears, unpaid late fee, discounts, tax lines | ACT | R1 | M |
| FR-FEE-010 | Family voucher option combining siblings in one voucher | ACT | R1 | M |
| FR-FEE-011 | Voucher fields: number, issue date, due date, validity date, fee month, guardian details, bank accounts, amounts payable within and after due date | System | R1 | M |
| FR-FEE-012 | Late fee rules: fixed after due date or per day with cap | ACT | R1 | M |
| FR-FEE-013 | Voucher statuses: `issued`, `partially_paid`, `paid`, `overdue`, `cancelled`, `superseded`; cancel or regenerate with reason | ACT | R1 | M |
| FR-FEE-014 | Ad-hoc vouchers (admission, exam fee, fines, charges) | ACT, ADO | R1 | M |
| FR-FEE-015 | Installment plans for large amounts | ACT, PR | R1 | S |
| FR-FEE-016 | Printing: 3-copy challan, bulk PDF per class; guardian download | ACT, GDN | R1 | M |
| FR-FEE-017 | Payment entry against voucher or student account: cash, bank deposit, cheque | CSH, ACT | R1 | M |
| FR-FEE-018 | Partial payments (setting) with allocation order (oldest first or head priority) | System | R1 | M |
| FR-FEE-019 | Receipt print and SMS confirmation | CSH | R1 | M |
| FR-FEE-020 | Cheque lifecycle: received, deposited, cleared, bounced (reverses payment, adds bounce charge) | ACT | R1 | M |
| FR-FEE-021 | Bank challan bulk entry: enter voucher numbers from stamped copies or import a bank file | ACT | R1 | M |
| FR-FEE-022 | Cashier sessions: open with float, close with counted cash, variance recorded and approved | CSH, ACT | R1 | M |
| FR-FEE-023 | Payment reversal through approval (reversing entries) | ACT, PR | R1 | M |
| FR-FEE-024 | Refunds (security deposit on withdrawal, excess payments) through approval | ACT, PR | R1 | M |
| FR-FEE-025 | Advance payments held as credit and applied to next vouchers | System | R1 | M |
| FR-FEE-026 | Student ledger and family ledger statements | ACT, GDN | R1 | M |
| FR-FEE-027 | Struck-off rule: N months unpaid puts the student on a list for principal action; re-admission fee | System | R1 | M |
| FR-FEE-028 | Defaulter list with SMS reminders | ACT | R1 | M |
| FR-FEE-029 | Fee clearance check used by withdrawal, certificates and admit cards (block configurable) | System | R1 | M |
| FR-FEE-030 | Reports: daily, monthly, branch-wise, class-wise, head-wise collection; outstanding; defaulters; discounts; scholarships; refunds; cashier-wise daily cash; payment history; fee ledger | ACT, OW, PR | R1 | M |
| FR-FEE-031 | Every voucher, payment, reversal and refund posts to Accounting through the posting service | System | R1 | M |
| FR-FEE-032 | Online payment from guardian portal through gateway; paid only after verified server-side callback and status inquiry | GDN | R2 | M |
| FR-FEE-033 | 1Bill consumer number on vouchers; bill inquiry and payment notification endpoints | System | R2 | M |
| FR-FEE-034 | Gateway reconciliation job and transaction log | ACT | R2 | M |
| FR-FEE-035 | Bank-specific APIs and Raast | System | R3 | S |

**Business rules**
- BR-FEE-01: One monthly voucher per student (or family) per fee month, unless the previous one is cancelled or superseded.
- BR-FEE-02: Voucher amounts are frozen at generation. Structure changes do not alter issued vouchers.
- BR-FEE-03: Payments are never edited or deleted. Corrections are reversals.
- BR-FEE-04: Allocation never exceeds outstanding; any extra becomes an advance credit.
- BR-FEE-05: A client-side success message never marks a payment paid (DEC-16).
- BR-FEE-06: Payment entry requires an open cashier session for cash payments.
- BR-FEE-07: Accrual posting per DEC-23.

**User stories**
- US-FEE-01: As ACT, I generate December vouchers for 3,000 students in one action so that they are printed the same day.
- US-FEE-02: As CSH, I record a cash payment and print a receipt in under 30 seconds.
- US-FEE-03: As OW, I see today's collection for each branch at 2 pm.

**Acceptance criteria**
- AC-FEE-01: Double-clicking "Save payment" creates exactly one payment.
- AC-FEE-02: Generating December vouchers twice for the same class creates no duplicates.
- AC-FEE-03: A reversed payment restores the voucher's outstanding and creates a balanced reversing journal.
- AC-FEE-04: Total collection on the daily report equals the sum of cash and bank postings in Accounting for that day.

---

## 2.22 Accounting (`accounting`) | ACC | Standard tier | R1 to R3

**Purpose:** Double-entry books for each organization with branch dimension.
**Key entities:** `fiscal_years`, `accounts`, `journal_entries`, `journal_lines`, `bank_accounts`, `bank_statement_lines`, `bank_reconciliations`, `vendors`, `expense_vouchers`, `purchase_bills`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-ACC-001 | Fiscal years (default July to June); open and close | ACT, OW | R1 | M |
| FR-ACC-002 | Chart of accounts tree (asset, liability, equity, income, expense; group and posting accounts), seeded school template | ACT | R1 | M |
| FR-ACC-003 | Branch and department dimensions on journal lines | System | R1 | M |
| FR-ACC-004 | Single posting service: the only way to create journals; used by Fee, Payroll, Expense and manual entries | System | R1 | M |
| FR-ACC-005 | Manual journal entries with approval above threshold | ACT | R1 | M |
| FR-ACC-006 | Entry numbering per fiscal year; statuses `draft`, `posted`, `reversed` | System | R1 | M |
| FR-ACC-007 | Cashbook, bank book, general ledger with running balance | ACT | R1 | M |
| FR-ACC-008 | Trial balance; fee receivables aging | ACT, OW | R1 | M |
| FR-ACC-009 | Opening balances entry at go-live | ACT | R1 | M |
| FR-ACC-010 | Bank accounts register | ACT | R1 | M |
| FR-ACC-011 | Monthly period locks | ACT | R1 | M |
| FR-ACC-012 | Expense vouchers with attachments and approval | ACT, PR | R2 | M |
| FR-ACC-013 | Vendors, purchase bills, vendor payments | ACT | R2 | M |
| FR-ACC-014 | Bank statement import and reconciliation matching | ACT | R2 | M |
| FR-ACC-015 | Income statement and balance sheet, per branch and consolidated | ACT, OW | R2 | M |
| FR-ACC-016 | Advanced consolidation and inter-branch transfers | ACT, OW | R3 | S |

**Business rules**
- BR-ACC-01: Every journal entry balances (sum of debits equals sum of credits), checked in the posting service inside the transaction.
- BR-ACC-02: Posted entries are immutable; corrections are reversals.
- BR-ACC-03: Posting only into open fiscal years and unlocked months.
- BR-ACC-04: System accounts (cash, bank, fee receivable, fee income heads) cannot be deleted.

**Acceptance criteria**
- AC-ACC-01: An unbalanced entry is rejected and nothing is saved.
- AC-ACC-02: Trial balance totals match for any date range after 1,000 random fee transactions in a test.
- AC-ACC-03: Posting into a locked month is refused.

---

## 2.23 Examination (`examination`) | EXM | Core tier | R1, R2

**Purpose:** Exam setup, date sheets, admit cards and marks entry.
**Key entities:** `exam_terms`, `exams`, `exam_papers`, `exam_paper_components`, `mark_entries`, `mark_entry_revisions`, `marks_submissions`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-EXM-001 | Exam terms per session: weekly test, monthly test, class test, midterm, final, send-up, mock, practical, oral; weightage for cumulative result | EXC, AC | R1 | M |
| FR-EXM-002 | Exams for selected classes and sections within a term | EXC | R1 | M |
| FR-EXM-003 | Papers per subject: date, start time, duration, room, total marks, pass marks; components (theory, practical, oral) | EXC | R1 | M |
| FR-EXM-004 | Date sheet builder with clash detection (one class two papers same slot, room capacity) and publish with notification | EXC | R1 | M |
| FR-EXM-005 | Admit cards / roll number slips with fee clearance check | EXC, BM | R1 | M |
| FR-EXM-006 | Marks entry grid per section per paper for the assigned teacher; codes per DEC-28; auto-save; validation against totals | ST, CT | R1 | M |
| FR-EXM-007 | Submit marks for approval; submission locks the grid for the teacher | ST | R1 | M |
| FR-EXM-008 | Review by HOD or EXC: approve or return with comment | HOD, EXC | R1 | M |
| FR-EXM-009 | Changes after approval only through `marks_revision` request; revisions stored | ST, EXC | R1 | M |
| FR-EXM-010 | Marks completion tracker: pending sections and subjects per exam | EXC, PR | R1 | M |
| FR-EXM-011 | Marks import via IMX | EXC | R1 | S |
| FR-EXM-012 | Invigilation duty assignment | EXC | R2 | S |
| FR-EXM-013 | Board exam support: board roll numbers, board result entry, analysis | EXC | R2 | M |

**Business rules**
- BR-EXM-01: Teachers enter marks only for their teaching assignments.
- BR-EXM-02: Approved marks change only through approved revisions.
- BR-EXM-03: Marks cannot exceed component totals; `ABS` is not zero.

**User stories**
- US-EXM-01: As ST, I enter Physics marks for 9-A on my laptop and submit them so that the HOD can approve.
- US-EXM-02: As EXC, I see that 4 of 60 papers are still pending so that I can follow up.

**Acceptance criteria**
- AC-EXM-01: A teacher not assigned to 9-B gets 403 on 9-B's marks grid.
- AC-EXM-02: After approval, editing a mark directly returns 403; an approved revision updates it and keeps the old value.

---

## 2.24 Result (`result`) | RES | Core tier | R1 to R3

**Purpose:** Configurable result calculation, approval, publication and report cards.
**Key entities:** `grading_schemes`, `grading_scheme_bands`, `result_rules`, `result_batches`, `student_results`, `student_subject_results`, `result_publications`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-RES-001 | Grading schemes per institution and level: bands (min %, grade, optional GPA point, remark en/ur) | EXC, OA | R1 | M |
| FR-RES-002 | Result rules: subject pass %, overall pass %, maximum failed subjects for "promoted with compartment", grace marks limits, rounding, term weightage, minimum attendance % for eligibility | EXC, PR | R1 | M |
| FR-RES-003 | Ranking: enabled flag, method (standard or dense), scope (section, class, branch), tie-breakers, visibility to students and guardians | EXC, PR | R1 | M |
| FR-RES-004 | Process result batch (queued): totals, percentage, grade, GPA, status (`pass`, `fail`, `compartment`, `absent`, `withheld`), positions | EXC | R1 | M |
| FR-RES-005 | Review screen with anomalies (missing marks, absent in all papers) | EXC, PR | R1 | M |
| FR-RES-006 | Approval for publication through APR | PR | R1 | M |
| FR-RES-007 | Scheduled publication; notifications to students and guardians | EXC | R1 | M |
| FR-RES-008 | Withhold individual results with reason (fee, discipline) | PR | R1 | M |
| FR-RES-009 | Re-processing after revision creates a new result version | EXC | R1 | M |
| FR-RES-010 | Report cards: single and bulk per section; teacher remarks, attendance summary, signatures | EXC, CT | R1 | M |
| FR-RES-011 | Cumulative annual result feeding promotion eligibility | System | R1 | M |
| FR-RES-012 | Basic analytics: subject averages, pass rates, top performers | PR, EXC | R1 | S |
| FR-RES-013 | Result re-check requests from guardians | GDN | R2 | S |
| FR-RES-014 | Advanced analytics (trends, branch comparison) | OW, PR | R3 | S |

**Business rules**
- BR-RES-01: Published results are immutable; unpublishing needs permission, approval and reason, and creates a new version.
- BR-RES-02: Results cannot publish while any included paper has unapproved marks.

**Acceptance criteria**
- AC-RES-01: Changing the grading band for 80% from A to A+ before processing changes all affected grades; after publication it changes nothing until reprocessed through approval.
- AC-RES-02: A withheld student's guardian sees "Result withheld, contact office" and no marks.

---

## 2.25 Timetable (`timetable`) | TTB | Standard tier | R1 to R3

**Purpose:** Bell schedules, timetables, conflict prevention and substitutions.
**Key entities:** `bell_schedules`, `bell_schedule_periods`, `timetables`, `timetable_entries`, `timetable_substitutions`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-TTB-001 | Bell schedules per branch and shift with day patterns (e.g., short Friday), periods with start, end, break flag | AC, PR | R1 | M |
| FR-TTB-002 | Timetable per section per session with versions (`draft`, `active`) and effective date | AC | R1 | M |
| FR-TTB-003 | Entries: day, period, subject, teacher, room | AC | R1 | M |
| FR-TTB-004 | Validations: teacher double booking (across branches), room double booking, section double booking, weekly load versus teaching assignment periods, maximum consecutive periods | System | R1 | M |
| FR-TTB-005 | Views: section, teacher, room, subject; "my timetable" for teacher, student, guardian | All | R1 | M |
| FR-TTB-006 | Print and PDF | AC, TCH | R1 | M |
| FR-TTB-007 | Copy from previous session | AC | R1 | S |
| FR-TTB-008 | Substitutions: absent teacher's periods listed, free teachers suggested, assignment, notification, workload record | AC, VP | R2 | M |
| FR-TTB-009 | Assisted auto-generation | AC | R3 | C |

**Business rules**
- BR-TTB-01: One active timetable version per section on any date.
- BR-TTB-02: A conflicting entry cannot be saved.

**Acceptance criteria**
- AC-TTB-01: Placing a teacher in two sections in the same period is refused with the clashing section named.

---

## 2.26 Communication (`communication`) | COMM | Core tier | R1 to R3

**Purpose:** Announcements, notices, daily diary, messaging and parent-teacher meetings.
**Key entities:** `announcements`, `announcement_audiences`, `announcement_reads`, `diary_entries`, `message_threads`, `messages`, `ptm_events`, `ptm_slots`, `ptm_bookings`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-COMM-001 | Announcements, notices and circulars with audience targeting (organization, branch, class, section, role, specific guardians or staff), attachments, schedule, expiry, pin | MGT | R1 | M |
| FR-COMM-002 | Acknowledgment required option with read receipts | MGT | R1 | M |
| FR-COMM-003 | Optional SMS or WhatsApp copy with cost estimate | MGT | R1 | M |
| FR-COMM-004 | Notice board per portal | All | R1 | M |
| FR-COMM-005 | Daily diary: per section per date, entries per subject (homework text, reminders, attachments) | TCH | R1 | M |
| FR-COMM-006 | Optional coordinator review before diary publishes | AC | R1 | S |
| FR-COMM-007 | Guardian-teacher messaging limited to assigned teachers, within set hours, moderated, logged, no deletion | GDN, TCH | R2 | M |
| FR-COMM-008 | PTM: event, teacher slots, guardian booking, attendance and notes | PR, TCH, GDN | R2 | M |
| FR-COMM-009 | Complaints handled in Administration (R3) | n/a | R3 | n/a |

**Business rules**
- BR-COMM-01: Guardians see only announcements targeting their children's branch, class or section, or them directly.
- BR-COMM-02: Published diary entries can be edited until end of day with an "edited" marker; afterwards locked.

**Acceptance criteria**
- AC-COMM-01: An announcement for Class 7 is invisible to a guardian with only a Class 5 child.

---

## 2.27 Learning (`learning`) | LMS | Add-on | R2

**Purpose:** Lecture content linked to the academic structure.
**Key entities:** `courses`, `course_chapters`, `course_topics`, `lectures`, `lecture_progress`, `discussion_threads`, `discussion_posts`, `content_reports`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-LMS-001 | Course per class subject per session, created automatically | System | R2 | M |
| FR-LMS-002 | Chapters and topics with ordering | ST | R2 | M |
| FR-LMS-003 | Lecture fields per brief: title, description, subject, class, sections, chapter, topic, teacher, attachments, video URL or upload, publish and availability dates, status, visibility, comments setting | ST | R2 | M |
| FR-LMS-004 | Resource types: uploaded video (size-capped), YouTube/Vimeo embeds, PDF, DOCX, PPTX, images, audio, links, rich text | ST | R2 | M |
| FR-LMS-005 | Copy lectures to other sections or the next session | ST | R2 | M |
| FR-LMS-006 | Student view, download (if allowed), progress tracking | STD | R2 | M |
| FR-LMS-007 | Engagement analytics per lecture and per student | ST, HOD | R2 | S |
| FR-LMS-008 | Discussions: questions, comments, replies; teacher answers, pins, marks resolved, disables comments, moderates | STD, ST | R2 | M |
| FR-LMS-009 | Report inappropriate content; moderation queue | STD, ST | R2 | M |
| FR-LMS-010 | Guardian read-only access to lectures and child's progress | GDN | R2 | S |

**Business rules**
- BR-LMS-01: Students see only lectures for sections of their current enrollment.
- BR-LMS-02: Students can edit their own post for 15 minutes; after a reply it is locked.

**Acceptance criteria**
- AC-LMS-01: A lecture scheduled for Monday 08:00 is invisible to students until then.

---

## 2.28 Homework (`homework`) | HWK | Add-on | R2

**Purpose:** Homework and assignments with online submission and grading.
**Key entities:** `homework_assignments`, `homework_targets`, `homework_submissions`, `homework_submission_reviews`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-HWK-001 | Create homework: type (daily, assignment, project), individual or group, subject, sections or selected students, instructions, attachments, assign date, due date and time, optional max marks | TCH | R2 | M |
| FR-HWK-002 | Submission modes: file, image, PDF, text, or offline (no upload) | TCH | R2 | M |
| FR-HWK-003 | Resubmission and late rules (allow with flag, penalty, or disallow) | TCH | R2 | M |
| FR-HWK-004 | Student or guardian submits; phone camera images compressed | STD, GDN | R2 | M |
| FR-HWK-005 | Review queue: grade, feedback, return for resubmission | TCH | R2 | M |
| FR-HWK-006 | Per-student status: `pending`, `submitted`, `late`, `reviewed`, `returned`, `missing` | System | R2 | M |
| FR-HWK-007 | Guardian monitoring and deadline reminders | GDN | R2 | M |
| FR-HWK-008 | Submission rate reports per section and student | TCH, AC | R2 | M |

**Acceptance criteria**
- AC-HWK-01: A submission after the deadline with "disallow" is refused server-side even if the page was open.

---

## 2.29 Assessment (`assessment`) | ASM | Add-on | R2

**Purpose:** Question bank, online quizzes and online exams.
**Key entities:** `question_banks`, `questions`, `question_options`, `question_tags`, `quizzes`, `quiz_questions`, `quiz_targets`, `quiz_attempts`, `quiz_attempt_answers`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-ASM-001 | Question bank per subject and level with chapter, topic, difficulty, tags; sharing within department or organization | ST, HOD | R2 | M |
| FR-ASM-002 | Question types: MCQ, multi-select, true/false, short, long, fill in the blanks (accepted answers), matching, image-based | ST | R2 | M |
| FR-ASM-003 | Question import from template | ST | R2 | S |
| FR-ASM-004 | Quiz builder: manual selection or random rules (count by chapter and difficulty), time limit, window, attempts, shuffling, negative marking, result and answer display policy | ST | R2 | M |
| FR-ASM-005 | Online exam mode: single attempt, full-screen prompt, tab-switch logging (recorded only, not presented as prevention) | ST, EXC | R2 | M |
| FR-ASM-006 | Attempt engine: server-side timer, autosave, auto-submit at time end, resume after disconnection within window | System | R2 | M |
| FR-ASM-007 | Automatic grading of objective types; manual grading queue for subjective | System, ST | R2 | M |
| FR-ASM-008 | Result publication | ST | R2 | M |
| FR-ASM-009 | Analytics: score distribution, item difficulty | ST, HOD | R2 | S |
| FR-ASM-010 | Transfer online exam scores into an EXM paper's marks entry | ST, EXC | R2 | S |

**Business rules**
- BR-ASM-01: Timing is enforced on the server; answers after end time are rejected.
- BR-ASM-02: Correct answers are never sent to the browser before submission.

**Acceptance criteria**
- AC-ASM-01: Closing the browser mid-quiz and reopening within the window resumes with saved answers and remaining time.

---

## 2.30 Payroll (`payroll`) | PAY | Add-on | R2

**Purpose:** Salary structures, monthly payroll and payslips.
**Key entities:** `salary_components`, `salary_structures`, `salary_structure_components`, `employee_salaries`, `payroll_runs`, `payslips`, `payslip_lines`, `salary_advances`, `employee_loans`, `loan_installments`, `salary_payments`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-PAY-001 | Components: earnings (basic, house rent, conveyance, medical, special) and deductions (absence, late, EOBI, PESSI, income tax, provident fund, loan, advance) | HRM, ACT | R2 | M |
| FR-PAY-002 | Fixed, percentage of basic, or slab-based calculation | HRM | R2 | M |
| FR-PAY-003 | Salary structures per designation or grade; employee salary with effective-dated history | HRM | R2 | M |
| FR-PAY-004 | Monthly payroll run per branch or organization: pull attendance summary, unpaid leave, late conversions; compute; adjust with reason | HRM, ACT | R2 | M |
| FR-PAY-005 | Approval, finalization (lock), journal posting | PR, OW, ACT | R2 | M |
| FR-PAY-006 | Payment recording: bank advice export, cash, cheque | ACT | R2 | M |
| FR-PAY-007 | Payslips PDF and staff portal access | Staff | R2 | M |
| FR-PAY-008 | Advances and loans with installment recovery | HRM, ACT | R2 | M |
| FR-PAY-009 | Increments, arrears, final settlement | HRM | R2 | M |
| FR-PAY-010 | Reports: payroll register, bank advice, deductions (EOBI, PESSI, tax), cost by branch and department | ACT, OW | R2 | M |

**Business rules**
- BR-PAY-01: Finalized runs are immutable; corrections go into the next month or through reversal approval.
- BR-PAY-02: Payroll data visible only with payroll permissions.

**Acceptance criteria**
- AC-PAY-01: An employee with 2 unpaid leave days gets the configured deduction on the payslip and in the journal.

---

## 2.31 Library (`library`) | LIB | Add-on | R3

**Key entities:** `library_books`, `library_book_copies`, `library_authors`, `library_publishers`, `library_categories`, `library_members`, `library_loans`, `library_fines`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-LIB-001 | Catalog: title, ISBN, authors, publisher, category, edition, language, shelf | LBR | R3 | M |
| FR-LIB-002 | Copies with accession number and barcode; label printing | LBR | R3 | M |
| FR-LIB-003 | Members created automatically for students and staff with type-based limits | System | R3 | M |
| FR-LIB-004 | Issue and return by barcode scan; due dates; renewals; reservations | LBR | R3 | M |
| FR-LIB-005 | Late fines (per day) and lost or damaged charges posted as fee items or cash | LBR, ACT | R3 | M |
| FR-LIB-006 | Stock verification | LBR | R3 | S |
| FR-LIB-007 | Reports: issued, overdue, popular, fines; clearance check for withdrawal | LBR | R3 | M |

**Acceptance criteria**
- AC-LIB-01: A student with an overdue book fails the withdrawal clearance check.

---

## 2.32 Transport (`transport`) | TRN | Add-on | R2, R3

**Key entities:** `transport_routes`, `transport_stops`, `route_stops`, `transport_allocations`, `vehicles`, `vehicle_documents`, `driver_profiles`, `vehicle_maintenance_logs`, `fuel_logs`, `transport_attendances`, `gps_devices`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-TRN-001 | Routes and ordered stops with pickup and drop times; fare per stop or zone | TM | R2 | M |
| FR-TRN-002 | Student allocation (route, stop, pickup/drop/both, dates) adds transport fee items to vouchers | TM, ACT | R2 | M |
| FR-TRN-003 | Capacity check per vehicle and route | System | R2 | M |
| FR-TRN-004 | Route sheets and emergency contact lists | TM | R2 | M |
| FR-TRN-005 | Vehicles: registration, type, capacity, ownership (own or contractor), documents with expiry (fitness, route permit, insurance, token tax) | TM | R3 | M |
| FR-TRN-006 | Drivers and conductors with license expiry alerts | TM | R3 | M |
| FR-TRN-007 | Maintenance and fuel logs | TM | R3 | M |
| FR-TRN-008 | Transport attendance (boarding) by transport staff phone | TS | R3 | S |
| FR-TRN-009 | Optional GPS provider adapter; live map (Leaflet); guardian ETA | TM, GDN | R3 | C |

**Acceptance criteria**
- AC-TRN-01: Allocating a student to a route from March adds the transport head to March and later vouchers only.

---

## 2.33 Inventory (`inventory`) | INV | Add-on | R3

**Key entities:** `inventory_categories`, `inventory_items`, `stock_locations`, `stock_movements`, `assets`, `asset_assignments`, `maintenance_requests`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-INV-001 | Items, categories, units | BM | R3 | M |
| FR-INV-002 | Stock locations per branch | BM | R3 | M |
| FR-INV-003 | Stock in, issue, transfer between branches, adjustments with approval | BM | R3 | M |
| FR-INV-004 | Reorder level alerts | BM | R3 | S |
| FR-INV-005 | Fixed asset register: tag, purchase date, cost, location, custodian; assignment and return | BM, ACT | R3 | M |
| FR-INV-006 | Maintenance requests and logs | All staff, BM | R3 | M |
| FR-INV-007 | Stock and asset reports | BM, ACT | R3 | M |

**Acceptance criteria**
- AC-INV-01: Stock on hand equals the sum of movements for every item and location.

---

## 2.34 Administration (`administration`) | ADN | Standard tier | R1 to R3

**Key entities:** `calendar_events`, `gate_passes`, `visitor_logs`, `complaints`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-ADN-001 | Academic calendar per institution or branch: holidays (public, institutional, notified closure), events, exam periods, PTMs, vacations | PR, BM | R1 | M |
| FR-ADN-002 | Bulk closure declaration (smog, heat, security) updating attendance expectations and timetable for the dates | PR, OW | R1 | M |
| FR-ADN-003 | Events with audience and optional RSVP | PR | R1 | S |
| FR-ADN-004 | Bulk ID card printing for students and staff with photo and QR | BM | R1 | M |
| FR-ADN-005 | Gate passes: early pickup verified against pickup authorizations; guardian notified | REC, BM | R2 | M |
| FR-ADN-006 | Visitor management: entry, purpose, host, badge, check-out, optional photo | REC | R3 | S |
| FR-ADN-007 | Complaints: category, assignment, SLA, resolution, satisfaction | GDN, Staff, BM | R3 | S |

**Acceptance criteria**
- AC-ADN-01: Declaring a closure for tomorrow sets all students and staff to `H` for that date and suppresses absence SMS.

---

## 2.35 Discipline (`discipline`) | DSC | Add-on | R2, R3

**Key entities:** `discipline_incidents`, `discipline_incident_students`, `discipline_actions`, `student_remarks`, `student_achievements`, `counseling_sessions`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-DSC-001 | Incidents: date, students, category, severity, description, witnesses, action | TCH, VP | R2 | M |
| FR-DSC-002 | Actions: warning, detention, suspension, guardian call; guardian notification by severity | VP, PR | R2 | M |
| FR-DSC-003 | Teacher remarks (positive and negative) | TCH | R2 | M |
| FR-DSC-004 | Rewards, achievements, co-curricular participation | TCH, PR | R2 | M |
| FR-DSC-005 | Counseling records visible only with `discipline.counseling.view`; never shown to guardians or teachers | Counselor (custom role), PR | R3 | M |
| FR-DSC-006 | Reports | PR | R2 | S |

**Acceptance criteria**
- AC-DSC-01: A class teacher cannot open counseling records even for their own student.

---

## 2.36 Reporting (`reporting`) | RPT | Core tier | R1 to R3

**Key entities:** `daily_metric_snapshots`, `saved_report_filters`, `scheduled_reports`

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-RPT-001 | Role dashboards (owner, principal, teacher, student, guardian, accountant, HR, exam controller, admin) with widgets defined in D10 | All | R1 | M |
| FR-RPT-002 | Consolidated and per-branch views | OW, OA | R1 | M |
| FR-RPT-003 | KPI snapshots updated by scheduled jobs (every 15 minutes for today's attendance and collection, nightly for others) with "last updated" time | System | R1 | M |
| FR-RPT-004 | Drill-down from any KPI to the filtered list behind it | All | R1 | M |
| FR-RPT-005 | Report catalog: each module registers its reports with permission, filters and columns | System | R1 | M |
| FR-RPT-006 | Saved filters per user | All staff | R1 | S |
| FR-RPT-007 | Basic branch comparison table (students, attendance %, collection %, outstanding) | OW | R1 | M |
| FR-RPT-008 | Scheduled reports by email | OW, PR, ACT | R2 | M |
| FR-RPT-009 | Advanced analytics and trends | OW | R3 | S |

**Business rules**
- BR-RPT-01: Dashboards and reports apply the same scopes and visibility as the underlying lists.

**Acceptance criteria**
- AC-RPT-01: The owner's attendance KPI drill-down lists exactly the absentees counted in the KPI.
- AC-RPT-02: A branch principal's dashboard never includes other branches' numbers.

---

## 2.37 Requirement Count Summary

| Release | Modules touched | Approx. FRs |
|---|---|---|
| R1 | 25 | about 330 |
| R2 | 18 | about 120 |
| R3 | 9 | about 40 |

---

## Assumptions

| ID | Assumption |
|---|---|
| A-19 | Daily student attendance is the official record for schools; period-wise attendance is optional and informational unless an institution sets it as official. |
| A-20 | Exam fee, admission fee and fines are charged through ad-hoc vouchers using the same voucher engine. |
| A-21 | A family voucher is paid as one amount and allocated across siblings by the configured allocation order. |
| A-22 | Grace marks are applied by rule during result processing, never by editing raw marks. |
| A-23 | A counselor is a custom role created by the organization; there is no seeded counselor role. |
| A-24 | Students below the login threshold have their homework submitted by guardians in R2. |
| A-25 | Device timestamps are trusted after timezone normalization; devices with clock drift beyond 5 minutes are flagged. |
| A-26 | Bank deposits recorded from stamped challans are treated as received on the bank stamp date. |
| A-27 | Report catalogs are code-defined; end users cannot build arbitrary SQL reports. |

## Gaps Found and Added

| ID | Gap | Decision | Release |
|---|---|---|---|
| GAP-45 | No custody restriction handling | Flag on guardian blocking login and pickup (FR-GRD-003) | R1 |
| GAP-46 | No subject-period attendance for colleges | Optional period-wise attendance (FR-ATT-004) | R1 |
| GAP-47 | No send-up exams or attendance eligibility | Send-up term and minimum attendance rule (FR-EXM-001, FR-RES-002) | R1 |
| GAP-48 | No early pickup control | Gate passes verified against pickup authorizations (FR-ADN-005) | R2 |
| GAP-49 | No quiet hours or deduplication | FR-NTF-009, FR-NTF-010 | R1 |
| GAP-50 | No sandwich leave rule | FR-LVE-002 | R1 |
| GAP-51 | No cheque lifecycle | FR-FEE-020 | R1 |
| GAP-52 | No mid-session fee revision | FR-FEE-003 | R1 |
| GAP-53 | No protection against lost updates | Optimistic locking (FR-GEN-010) | R1 |
| GAP-54 | No result re-check requests | FR-RES-013 | R2 |
| GAP-55 | No short-Friday schedules | Day patterns in bell schedules (FR-TTB-001) | R1 |
| GAP-56 | Shared guardian mobile numbers | DEC-22 | R1 |
| GAP-57 | Fee accounting basis undefined | Accrual (DEC-23) | R1 |
| GAP-58 | No invigilation duties | FR-EXM-012 | R2 |
| GAP-59 | No document expiry reminders | HR documents (R1), vehicle documents (R3) | R1, R3 |
| GAP-60 | Existence disclosure through 403 responses | DEC-27 | R1 |
| GAP-61 | No "attendance not marked" follow-up | FR-ATT-005 | R1 |
| GAP-62 | No device clock drift handling | A-25, flagged in sync logs | R1 |
| GAP-63 | No result withholding | FR-RES-008 | R1 |

---

## What Deliverable 3 Covers

Recommended software architecture: the modular monolith structure, module boundaries and communication rules (events versus direct calls), request lifecycle with tenancy and authorization layers, where to use Form Requests, services, actions, transactions, events, jobs, policies, middleware, notifications, scheduled commands and API resources, verified package list (Laravel 12 and 13, PHP 8.2 to 8.4), PDF engine selection, caching and queue topology, and the per-module technical breakdown (models, controllers, services, policies, events, listeners, jobs, endpoints).
