# Deliverable 4: Multi-Tenant and Multi-Branch Architecture

**Version:** 1.0 (draft for approval) | **Date:** 23 Nov 2026
**Builds on:** D1 (DEC-03, DEC-04, DEC-09), D2 (DEC-21, DEC-22, DEC-27), D3 (TenantContext, middleware stack, registries, DEC-29, DEC-41, DEC-42)

---

## 4.1 The Brief's Question: Is a Shared Database with Tenant and Branch Columns the Right Start?

**Answer: yes, with strict conditions.** A shared MySQL database with one shared schema, where every tenant-owned row carries `organization_id` (and `branch_id` where the data belongs to a branch), is the right starting point for this product. The conditions in 4.4 to 4.16 make it safe, and 4.20 keeps a dedicated database open for any tenant that needs one.

**Options compared**

| Criterion | A: Database per tenant | B: Schema per tenant (MySQL treats this the same as A) | C: Shared schema, row-level scoping (chosen) |
|---|---|---|---|
| Isolation strength | Strongest (separate database) | Strong | Good, only if enforced in several layers (4.15) |
| Operations for 500+ small schools | 500+ databases to migrate, back up, monitor | Same as A | One database to run |
| Migrations | Run N times, partial failures likely | Same as A | Run once |
| Platform-wide metrics (super admin console, usage) | Fan out across N databases | Same as A | One query |
| Cost per small tenant | High (connections, memory, backups) | High | Lowest |
| Restore one tenant | Easy | Easy | Needs per-tenant export tooling (4.19) |
| Noisy neighbor risk | Low | Low | Medium, handled by rate limits, queue fairness and query timeouts (4.18) |
| Fit for SEG-1 single schools paying a small fee | Poor | Poor | Good |
| Fit for a large group demanding isolation (Scenario D) | Good | Good | Handled by extracting that tenant to a dedicated instance (4.20) |
| AI-agent implementation risk | Lower leak risk, higher ops complexity | Same as A | Higher leak risk unless scoping is automatic and tested; addressed by fail-closed global scopes and generated isolation tests |

**Why C wins here:** most customers are small private schools that need low cost and zero ops overhead per tenant. Isolation risk is real but controllable through automatic scoping that fails closed, database-level composite foreign keys, and tests generated for every tenant model. Global auto-increment IDs plus `organization_id` on every row make lifting one tenant into its own database a mechanical export, not a redesign.

---

## 4.2 New Decisions in This Deliverable

| ID | Decision |
|---|---|
| DEC-43 | Every reference from one tenant-owned table to another uses a **composite foreign key** `(organization_id, {parent}_id)` referencing the parent's unique key `(organization_id, id)`. The database itself rejects a row pointing to another tenant's record |
| DEC-44 | Session and CSRF cookies are **host-only** (no `Domain=.platform-domain`). A session from `cityschool.darsora.app` is never sent to `alnoor.darsora.app` |
| DEC-45 | Tenancy **fails closed**: querying a tenant-owned model without a tenant context throws `MissingTenantContext`. Platform code that needs cross-tenant access uses an explicit, audited `Tenancy::bypass()` block |
| DEC-46 | Two global scopes: `OrganizationScope` (always on tenant models) and `AllowedBranchesScope` (on branch-owned models; limits rows to the union of branches the current user may access). Active-branch filtering and visibility levels are applied by Queries and policies on top |
| DEC-47 | One user may hold staff role assignments and guardian links in the same organization (a teacher whose child studies there). A portal switcher moves between `app` and `guardian`. Student accounts are always separate users |
| DEC-48 | High-volume append-only log tables (`attendance_punches`, `audit_logs`, `login_events`, `notification_dispatches`, `notification_delivery_attempts`, `webhook_receipts`, `device_sync_logs`) have **no database foreign keys** (integrity enforced in code and tests) so they can be range-partitioned by date later. Every index on them starts with `organization_id` |
| DEC-49 | `students.current_branch_id` is a denormalized pointer to the branch of the active enrollment, written only by Student enrollment actions. Historical truth is `student_enrollments.branch_id` |
| DEC-50 | Tenant hostnames live in `organization_domains` (Platform module): one platform subdomain per organization in R1, verified custom domains in R2 |
| DEC-51 | Tenant data deletion runs through a `TenantDataPurger` registry, module by module in reverse dependency order, only after export and the contractual retention period |
| DEC-52 | Cross-institution progression inside one organization (Class 10 at the school brand to Intermediate Part 1 at the college brand) is a promotion target, not a new admission. Registration number stays the same |

---

## 4.3 Hierarchy and Scope Keys

```
PLATFORM                       (no organization_id: plans, permission catalog, platform users)
 |
 +-- ORGANIZATION  = TENANT    organization_id
      |
      +-- INSTITUTION          institution_id   (school brand, college brand)
      |     |
      |     +-- ACADEMIC SESSION   academic_session_id   (time dimension, shared by the institution's branches)
      |     |
      |     +-- BRANCH         branch_id        (campus)
      |           |
      |           +-- SHIFT    shift_id         (morning, evening)
      |           +-- ACADEMIC CLASS (branch + session + level [+ study group])
      |                 +-- SECTION (capacity, shift, wing, class teacher, room)
      |                       +-- STUDENT ENROLLMENT (student + section + session)
      |
      +-- FISCAL YEAR          fiscal_year_id   (accounting time dimension, organization-wide)
```

**Reading the tree:** an academic session belongs to an institution, not a branch, so all branches of "City School" run the 2026-2027 session together. Classes and sections are created per branch inside that session. Fiscal years belong to the organization because the books are organization-wide with a branch dimension.

---

## 4.4 Table Categories and Their Scope Columns

| Category | Scope columns | Global scopes | Examples |
|---|---|---|---|
| **Platform (global)** | none | none | `plans`, `plan_modules`, `permissions`, `platform_announcements` |
| **Platform per-tenant** | `organization_id` | none (Platform module only, read by SA) | `organization_subscriptions`, `organization_modules`, `organization_domains`, `usage_counters`, `support_access_grants` |
| **Organization-owned** | `organization_id` | OrganizationScope | `institutions`, `branches`, `users`, `roles`, `guardians`, `families`, `students`, `employees`, `accounts`, `fiscal_years`, `leave_types`, `notification_templates`, `fee_heads` (with optional `institution_id`) |
| **Institution-owned** | `organization_id`, `institution_id` | OrganizationScope | `academic_sessions`, `education_levels`, `study_groups`, `subjects`, `grading_schemes`, `result_rules`, `admission_form_templates` |
| **Branch-owned** | `organization_id`, `branch_id` | OrganizationScope + AllowedBranchesScope | `academic_classes`, `sections`, `student_enrollments`, `student_attendances`, `staff_attendances`, `fee_vouchers`, `fee_payments`, `cashier_sessions`, `exams`, `timetables`, `diary_entries`, `attendance_devices`, `rooms`, `shifts` |
| **Branch-owned children** | `organization_id`, `branch_id` (copied from parent at creation, immutable) | Same as parent | `fee_voucher_items`, `fee_payment_allocations`, `journal_lines`, `mark_entries`, `timetable_entries` |
| **Organization-owned children** | `organization_id` | OrganizationScope | `guardian_students`, `role_permissions`, `user_role_assignments`, `approval_request_steps` |
| **Optional branch** | `organization_id`, nullable `branch_id` | OrganizationScope; branch filter applied only when `branch_id` is set | `departments`, `announcements`, `calendar_events`, `settings` (polymorphic scope), `approval_workflows` (branch override) |
| **High-volume logs** (DEC-48) | `organization_id`, nullable `branch_id` | OrganizationScope | `attendance_punches`, `audit_logs`, `login_events`, `notification_dispatches`, `webhook_receipts` |

**Rules**
1. Every tenant-owned table carries `organization_id`, including child tables whose parent already has it. Reasons: one global scope works on every table, composite foreign keys need it, and tenant extraction becomes a simple `WHERE organization_id = ?` per table.
2. A child carries `branch_id` whenever its parent does and the child is queried in branch reports. The value is copied at creation and never changes.
3. Platform users have `users.organization_id = NULL` and log in only on the platform domain.

---

## 4.5 Students, Employees and Guardians Across Branches

| Entity | Owned at | Branch link | Who sees it |
|---|---|---|---|
| `students` | Organization (registration number unique per organization) | `current_branch_id` (DEC-49); each enrollment has its own `branch_id` | Staff whose allowed branches include `current_branch_id`. Staff of a previous branch see a read-only "transferred out" profile with only their branch's records (fees, attendance, marks tagged with their `branch_id`) |
| `student_enrollments` | Branch | `branch_id` per session placement | Branch staff by scope |
| `employees` | Organization (CNIC unique per organization) | `primary_branch_id` plus `employee_branch_assignments` with dates | Staff with HR scope over any branch the employee is assigned to |
| `guardians` | Organization | None directly | Staff who can see at least one linked student; the guardian themselves |
| `families` | Organization | None directly | Same as guardians; siblings in different branches stay in one family |
| `users` | Organization | Role assignments carry branch scopes | Per D5 |

---

## 4.6 Composite Foreign Keys (DEC-43)

Every tenant parent table has, besides its primary key, a unique index on `(organization_id, id)`. Children reference it with a composite key.

```
sections:            PRIMARY KEY (id)
                     UNIQUE uq_sections_org_id (organization_id, id)

student_enrollments: FOREIGN KEY (organization_id, section_id)
                       REFERENCES sections (organization_id, id)
                     FOREIGN KEY (organization_id, student_id)
                       REFERENCES students (organization_id, id)
```

Laravel migration form:

```php
$table->foreign(['organization_id', 'section_id'])
      ->references(['organization_id', 'id'])->on('sections')
      ->restrictOnDelete();
```

**What the database guarantees:** an enrollment cannot point to a section or student of another organization, even if application code has a bug.

**What the application guarantees (not the database):** branch consistency, for example that an enrollment's section is in the enrollment's branch. Branch rules have legitimate exceptions (inter-branch teaching assignments), so they are enforced in Actions, validated in Form Requests, and covered by tests.

**Delete behavior:** `RESTRICT` by default. Tenant data is never cascade-deleted by the database; purging goes through DEC-51.

**Exceptions:** high-volume log tables (DEC-48) carry the same columns without foreign keys.

---

## 4.7 Tenant Resolution and Hostnames

| Host | Purpose | Tenant resolved by |
|---|---|---|
| `{slug}.darsora.app` | Tenant portals (`/app`, `/guardian`, `/student`, `/admissions`) | `organization_domains.domain` |
| Custom domain, e.g. `portal.cityschool.edu.pk` (R2) | Same as above | `organization_domains.domain` after DNS TXT verification |
| `platform.darsora.app` | Super admin portal | No tenant; platform users only |
| `adms.darsora.app` | ZKTeco ADMS (`/iclock/*`) | Device serial (DEC-29) |
| `hooks.darsora.app` | Gateway and provider webhooks `/webhooks/{provider}/{organization_uuid}` | UUID in path, then signature verified with that organization's credentials |
| `api.darsora.app` (R2) | API v1 | Token owner's organization |
| `darsora.app` | Marketing and "find your school" page | None; user types the school code, the page never lists schools |

**`organization_domains` columns:** `organization_id`, `domain` (unique, lowercase), `type` (`platform_subdomain`, `custom`), `is_primary`, `verification_token`, `verified_at`, `tls_status`, timestamps.

**Reserved slugs:** `www`, `app`, `api`, `platform`, `admin`, `adms`, `hooks`, `mail`, `smtp`, `static`, `assets`, `cdn`, `status`, `help`, `support`, `docs`, `billing`, `test`, `staging`, `demo`.

**Slug changes:** only SA can change a slug; the old domain row stays for 90 days with a redirect, then is removed.

**Cookies (DEC-44):** `SESSION_DOMAIN` is null so cookies are host-only. `SameSite=Lax`, `Secure`, `HttpOnly`. Each tenant host has its own session and CSRF token.

**Custom domains (R2):** the organization adds a CNAME to its platform subdomain plus a TXT record with the verification token; a scheduled job verifies DNS, then TLS is issued automatically (on-demand TLS at the edge proxy, detailed in D12). Unverified domains return 404.

---

## 4.8 Tenant Context in Every Execution Path

| Path | How context is set | If context is missing |
|---|---|---|
| Web request on tenant host | `ResolveTenant` from host, then `EnsureTenantMatchesUser` after auth | Unknown host: 404 page. Mismatch: logout plus security event |
| Web request on platform host | No tenant. Platform controllers use Platform models and explicit `Tenancy::bypass()` blocks for aggregate counts | Any tenant model query outside a bypass block throws (DEC-45) |
| Queued job | `TenantAware` job middleware restores organization, branch, user, correlation id captured at dispatch | Job fails with `MissingTenantContext`; no partial work |
| Scheduled command | Loops active organizations, dispatches one job per organization | Command itself never queries tenant models |
| Artisan command run by an operator | Requires `--organization=` option; sets context explicitly | Refuses to run |
| Webhook | Resolves organization from path UUID or device serial before processing; raw receipt stored with organization | Unknown UUID or serial: logged, 404, no processing |
| Tests | `inTenant($organization, fn () => ...)` helper; factories set context | Test fails loudly, which is the point |
| Seeders | Platform seeders (plans, permissions) run without context; tenant demo seeders require an organization | Refuse to run |
| Tinker / debugging | Must call `Tenancy::actAs($organization)` first | Queries throw |

**TenantContext fields:** `organizationId`, `institutionIds` (allowed), `allowedBranchIds`, `activeBranchId` (nullable means "All branches" read mode), `userId`, `impersonatorId`, `portal`, `locale`, `timezone`, `correlationId`.

---

## 4.9 Branch Scoping Model (DEC-46)

**Three layers, applied in this order:**

| Layer | Mechanism | Purpose |
|---|---|---|
| 1. Allowed branches | `AllowedBranchesScope` (global scope on branch-owned models) restricts to `TenantContext.allowedBranchIds` | Coarse guard: a user never sees rows from a branch where they hold no assignment. Organization-scope roles get all branches |
| 2. Active branch | Queries apply `->forActiveBranch()` unless the screen is in "All branches" mode | What the user is working on right now |
| 3. Visibility level | Policies and `->visibleTo($user, $permission)` apply `own`, `assigned`, `branch`, `institution`, `organization` per permission | Fine-grained: a teacher sees only assigned sections even inside their branch |

**Allowed branches** = union of branches covered by the user's active role assignments (organization scope covers all branches; institution scope covers that institution's branches; branch scope covers listed branches). For a guardian, allowed branches are the branches of linked, portal-enabled children. For a student, their current branch.

**Writes:** the `creating` hook on branch-owned models sets `branch_id` from `activeBranchId`. If the active branch is null ("All branches"), the write is rejected with "Select a branch first". If code sets a different `branch_id` explicitly, it must be in `allowedBranchIds` and the policy must allow it, otherwise the save throws.

**System jobs** run with `allowedBranchIds` = all branches of the organization and `activeBranchId` = the branch the job targets (or null for organization-wide jobs that write nothing branch-owned).

---

## 4.10 Visibility Levels in Practice (DEC-21)

| Level | Meaning | Typical holders |
|---|---|---|
| `own` | Records where the user is the subject or creator | Staff viewing own leave, attendance, payslips; student viewing own data |
| `assigned` | Records tied to the user's assignments (see table below) | Class teacher, subject teacher, HOD, transport manager |
| `branch` | All records in branches within scope | Principal, branch manager, accountant, cashier |
| `institution` | All branches of institutions within scope | Academic director of a brand (custom role) |
| `organization` | Every branch | Owner, organization admin, head-office accountant |

**What "assigned" means per resource**

| Resource | Assigned set |
|---|---|
| Students, guardians (contact view) | Students enrolled in sections where the user is class teacher or has a teaching assignment in the current session |
| Student attendance (daily) | Sections where the user is class teacher |
| Student attendance (period) | Section-subject pairs from teaching assignments |
| Marks entry | Exam papers whose subject and section match a teaching assignment |
| Homework, lectures, quizzes (R2) | Sections and subjects from teaching assignments |
| Diary entries | Sections with a teaching assignment or class teacher role |
| Discipline incidents | Incidents involving assigned students, or created by the user |
| HOD scope | Teaching assignments and marks for subjects in the HOD's department, within their branch scope |
| Transport (R2) | Routes assigned to the transport manager or staff |
| Approval requests | Requests where the user is a resolved approver, plus own requests |

---

## 4.11 Shared and Inter-Branch Employees

```
employees (organization_id, primary_branch_id, ...)
   |
   +-- employee_branch_assignments (organization_id, employee_id, branch_id,
   |                                designation_id, starts_on, ends_on, is_primary)
   +-- user_role_assignments (user of the employee, role, scope branch ids)
   +-- teaching_assignments (organization_id, branch_id, section_id, subject_id, employee_id, ...)
```

| Concern | Rule |
|---|---|
| Visibility | HR staff of any assigned branch see the employee; salary fields need payroll permission in the primary branch or organization scope |
| Roles per branch | A user can be `subject_teacher` in Branch A and `academic_coordinator` in Branch B; permissions are evaluated per branch |
| Teaching in another branch | Allowed only with an active `employee_branch_assignments` row for that branch and date range (BR-ACD-04) |
| Timetable | Teacher double-booking is checked across all branches of the organization, including travel-free assumption (no minimum gap; can be added as a rule later) |
| Attendance | The employee may punch at any assigned branch's device. The daily resolver merges punches across branches into one `staff_attendances` row per employee per date; `branch_id` on that row is the branch of the first check-in; each punch keeps its own device branch |
| Leave | One balance per employee (organization-level), approved by the primary branch chain unless the workflow says otherwise |
| Payroll (R2) | Paid by the primary branch; optional cost split percentages per branch assignment for branch-wise expense reports |
| Separation | Ends all branch assignments, role assignments and teaching assignments in one action (event `EmployeeSeparated`) |

---

## 4.12 Student Branch Transfer

```
CT/BM (Branch A)       Approval engine        PR (A)        PR (B)        System
      |  submit branch_transfer request (target branch B, target class/section, effective date)
      |-------------------->|
      |                     |-- step 1 --------->| approve
      |                     |-- step 2 ---------------------->| approve (choose section in B)
      |                     |-- apply (BranchTransferHandler, one transaction) --------------->|
      |                                                                   1. clearance checks in A
      |                                                                      (fees per policy, library, transport)
      |                                                                   2. close A enrollment: status
      |                                                                      transferred_out, ends_on
      |                                                                   3. create B enrollment, same session,
      |                                                                      new roll number
      |                                                                   4. new GR number in B (A's GR kept)
      |                                                                   5. students.current_branch_id = B
      |                                                                   6. fee balance per policy (below)
      |                                                                   7. end transport allocation (event)
      |                                                                   8. student_transfers row, timeline,
      |                                                                      audit
      |<---------------------------------- notifications (guardian, both principals, class teachers)
```

**Fee balance policy (organization setting)**

| Option | Effect |
|---|---|
| `settle_before_transfer` (default) | Transfer blocked until the student has no dues in Branch A (or an approved waiver exists) |
| `carry_to_target` | Outstanding moves as an opening line on the student's ledger in Branch B; Accounting posts an inter-branch entry (receivable out of A, into B) |
| `keep_in_source` | Dues stay as Branch A receivable; Branch A can still collect; Branch B starts clean |

**What does not move:** attendance, marks, results, discipline records and payments stay tagged to Branch A. They appear on the student profile in both branches, but Branch B staff see Branch A rows read-only and Branch A staff see only Branch A rows.

**Teacher transfer** follows the same pattern through HR: end the old branch assignment, create the new one, end or reassign teaching assignments, and move role assignment scopes.

---

## 4.13 Cross-Institution Progression (DEC-52)

Common in SEG-3: a student finishing Class 10 at the group's school continues to Intermediate Part 1 at the group's college.

- Education level "Class 10" (school institution) can be configured with `next_level_id` pointing to "Intermediate Part 1" in the college institution.
- The promotion wizard offers "Progress to: City College, Intermediate Part 1, session 2027-2028" and requires choosing study group and branch.
- Result: the school enrollment is completed, a new enrollment is created in the college institution's planning or active session. Registration number, guardians, family and timeline stay; a new GR number and admission number are issued in the college branch, and an optional college admission fee voucher is generated.
- Students who do not progress are graduated normally.

---

## 4.14 Shared Versus Branch-Specific Configuration

| Configuration | Default level | Override possible at | Notes |
|---|---|---|---|
| Roles and permissions | Organization | Assignment scope per branch | One role catalog per organization |
| Settings | Organization | Institution, branch (D2 FR-SET-002) | Security settings can only be tightened lower down |
| Education levels, study groups, subjects, grading schemes, result rules | Institution | None (shared by the institution's branches) | Keeps results comparable across branches |
| Academic sessions | Institution | Branch-specific dates (admission window) | |
| Fee heads | Organization (optional institution) | None | Same head names everywhere for consolidated reports |
| Fee structures | Branch + session + class | n/a | Branches often charge different fees |
| Discount rules (sibling) | Organization | Institution | Siblings in different branches count together if `sibling_scope = organization` |
| Chart of accounts | Organization | None | Branch is a dimension on journal lines |
| Leave types and policies | Organization | Designation or employment type | |
| Attendance policies | Branch + shift | Person type | Different gate timings per campus |
| Bell schedules | Branch + shift | Day pattern | |
| Notification templates | Organization | Institution | Branding per institution |
| Document templates | Institution | None | Each brand has its own report card and certificate look |
| Approval workflows | Organization | Branch override (FR-APR-001) | |
| Question banks (R2) | Department or organization (sharing flag) | n/a | |
| Library catalog (R3) | Organization | Copies are per branch | |

---

## 4.15 Isolation Enforcement Layers

| # | Layer | Mechanism | Failure mode it prevents | Test |
|---|---|---|---|---|
| 1 | Host | Tenant from host via `organization_domains`; reserved slugs | Wrong tenant served | Unknown host returns 404 |
| 2 | Cookies | Host-only session and CSRF cookies (DEC-44) | Session reuse across tenant subdomains | Cookie from host A rejected on host B |
| 3 | Middleware | `EnsureTenantMatchesUser` | Logged-in user of A browsing B's host | User of A on host B is logged out, security event logged |
| 4 | Global scope | `OrganizationScope`, fail closed (DEC-45) | Forgotten `where organization_id` | Query without context throws; query in A never returns B rows |
| 5 | Global scope | `AllowedBranchesScope` | Branch staff reading other branches | Branch A principal cannot list Branch B vouchers |
| 6 | Model hooks | `creating` sets `organization_id` and `branch_id` from context; mass assignment of these columns blocked (`$guarded`) | Request body injecting another tenant id | Posting `organization_id` of B is ignored |
| 7 | Route binding | ULID lookup through scopes | ID guessing (IDOR) | B's ULID on A's host returns 404 |
| 8 | Validation | `TenantExists` and `BranchExists` rules for every foreign-key input; plain `exists:` rule banned by architecture test | Referencing B's section in a form or JSON body | Submitting B's section ULID fails validation |
| 9 | Policies | Organization, branch scope and visibility checks | Over-broad access inside the tenant | Teacher cannot open non-assigned section's marks |
| 10 | Database | Composite foreign keys (DEC-43) | Bugs bypassing layers 4 to 9 | Direct insert with cross-tenant parent fails at DB |
| 11 | Raw SQL | Allowed only in `Queries` classes extending `TenantQuery`, which starts every builder with the organization filter; `DB::table` banned elsewhere | Report queries leaking | Architecture test plus report isolation tests |
| 12 | Jobs | `TenantAware` middleware | Job running for the wrong tenant or none | Job dispatched in A processes only A's rows |
| 13 | Cache | `org:{id}:` key prefix enforced by a `TenantCache` wrapper | Cached data shown to another tenant | Same key name in A and B returns different values |
| 14 | Files | Path prefix `org/{uuid}/`, signed URLs bound to user id and expiry, policy re-check on download | Guessed or shared file links | B's user with A's signed URL gets 404 |
| 15 | Exports and imports | Run under the user's scopes; files under the organization's path; download requires same user | Export containing other tenants' or branches' rows | Export by branch user contains only that branch |
| 16 | Notifications | Recipient resolvers run in tenant context; templates organization-scoped | Message sent to another tenant's guardian | Absence alert recipients all belong to the student's organization |
| 17 | Webhooks | Organization from UUID or device serial; signature checked with that organization's secret; payload references outside the organization rejected | Forged or misrouted callbacks | Callback signed with A's secret to B's URL rejected |
| 18 | API tokens (R2) | Token belongs to a user, user to an organization; abilities checked | Token reuse across tenants | A's token on B's host returns 401 |
| 19 | Platform | No tenant data without support grant; aggregate queries in `Tenancy::bypass()` are audited | Super admin browsing tenant data | SA without grant gets 403 on login-as and 404 on tenant routes |
| 20 | Realtime (R2) | Private channel names include organization UUID; channel authorization checks scope | Event leakage via websockets | Subscribing to B's channel from A fails |
| 21 | Error pages and logs | No record data in error pages; logs include ids, not personal data | Leaks through errors | Error page snapshot test |
| 22 | Backups and restores | Per-tenant export and restore tooling (4.19) | Restoring one tenant overwriting others | Restore drill on staging |

---

## 4.16 Tenant-Aware Unique Constraints

| Table | Unique key |
|---|---|
| `organization_domains` | `(domain)` globally |
| `branches` | `(organization_id, code)` |
| `users` | `(organization_id, mobile_e164)`, `(organization_id, email)`, `(organization_id, username)`; each nullable (MySQL allows multiple NULLs) |
| `students` | `(organization_id, registration_no)`, `(organization_id, b_form_no)` |
| `student_enrollments` | `(organization_id, student_id, academic_session_id)` for active-or-completed rows (see D7 for the partial-uniqueness technique), plus `(organization_id, branch_id, academic_session_id, gr_no)` if GR is per enrollment |
| `employees` | `(organization_id, employee_code)`, `(organization_id, cnic)` |
| `guardians` | `(organization_id, cnic)` |
| `fee_vouchers` | `(organization_id, voucher_no)`, `(organization_id, student_id, fee_month, voucher_type)` for monthly vouchers not cancelled or superseded |
| `fee_payments` | `(organization_id, receipt_no)`, `(organization_id, idempotency_key)` |
| `student_attendances` | `(organization_id, student_id, attendance_date)` |
| `staff_attendances` | `(organization_id, employee_id, attendance_date)` |
| `attendance_punches` | `(organization_id, attendance_device_id, device_user_id, punched_at)` |
| `attendance_devices` | `(serial_number)` globally (DEC-29) |
| `journal_entries` | `(organization_id, fiscal_year_id, entry_no)` |
| `mark_entries` | `(organization_id, exam_paper_id, exam_paper_component_id, student_id)` |
| `roles` | `(organization_id, slug)` |
| `numbering_sequences` | `(organization_id, sequence_key, scope_type, scope_id, period_key)` |

D7 lists every unique constraint; this table shows the pattern: business keys are always unique **within** an organization, never globally, except hostnames and device serials.

---

## 4.17 Identifiers and Enumeration

- Internal `id` (BIGINT, global auto-increment) never appears in URLs, forms, JSON or exports.
- Public `uuid` (ULID) is used in URLs and JSON. ULIDs are not guessable in practice and still sort by time.
- Human numbers (voucher number, receipt number, GR number, registration number) are displayed and searchable, but every lookup by them runs inside tenant and branch scopes.
- Because `id` is globally unique across tenants, a tenant's rows can be moved into an empty dedicated database without renumbering (4.20).

---

## 4.18 Noisy Neighbors and Fair Use

| Risk | Control |
|---|---|
| One tenant floods HTTP | Rate limits per user and per organization (e.g., 600 requests/minute per organization for staff routes, tighter for exports and logins) |
| Month-end bulk jobs | Per-organization `WithoutOverlapping` keys; bulk queue shared fairly by chunk size (D3 3.11) |
| Heavy reports | Reports read from snapshots where possible; long queries get `MAX_EXECUTION_TIME` hints (statement timeout) in `TenantQuery` for report endpoints; very large exports always queued |
| SMS bursts | Per-organization quotas and per-provider rate limits |
| Storage growth | Per-plan storage limits, image compression, video caps |
| Growth beyond the shared database | Move a tenant to a dedicated instance when it exceeds roughly 25,000 active students or consistently uses more than 15% of database load (4.20) |

---

## 4.19 Tenant Lifecycle

| Stage | Steps | Owner |
|---|---|---|
| Provision | Create `organizations` row, primary `organization_domains` row, subscription and module entitlements, copy system role templates into `roles`, apply setting defaults, create owner user, send invitation, open onboarding wizard | SA (`ProvisionOrganization`) |
| Onboard | Wizard steps from `OnboardingStepRegistry` (D3) | Owner |
| Active | Normal use | All |
| Suspend | Read-only mode with banner; scheduled jobs that send messages or generate vouchers skip suspended tenants | SA |
| Archive | No logins; data retained per contract | SA |
| Export | Full package: CSV/JSON per table plus files (`BuildTenantExport`, R2) | SA or Owner |
| Purge | After export and retention period: `TenantDataPurger` runs each module's purger in reverse dependency order, chunked deletes by `organization_id`, then files under `org/{uuid}/`, then platform rows except billing history | SA with second-person confirmation |

**Per-tenant export and restore tooling (needed for 4.20 and for single-tenant recovery):**
- `tenant:export --organization={uuid}` writes every tenant table filtered by `organization_id`, in foreign-key order, plus global catalog tables in full, plus files.
- `tenant:import --into={connection}` loads an export into an empty database with the same schema version.
- Single-tenant point-in-time recovery: restore the nightly backup plus binary logs into a scratch database, then `tenant:export` from the scratch database and replace that tenant's rows in production inside a maintenance window (runbook in D12).

---

## 4.20 Path to a Dedicated Database (Scenario D or a Large Tenant)

**Design properties that make this possible (already required):**
1. Every tenant table has `organization_id`; global tables are small catalogs.
2. No foreign key crosses tenants (DEC-43).
3. IDs are globally unique, so an extracted tenant keeps its IDs.
4. Files are stored under `org/{uuid}/`.
5. Numbering sequences, settings and roles are per organization.
6. No code hardcodes a database connection name; models use the default connection.

**Extraction runbook (summary; full runbook in D12)**

| Step | Action |
|---|---|
| 1 | Provision a dedicated instance (same release version) with its own MySQL, Redis, storage and Gotenberg |
| 2 | Announce a maintenance window; set the tenant to read-only |
| 3 | Run `tenant:export` on the shared database; copy files under `org/{uuid}/` |
| 4 | Run `tenant:import` on the dedicated database; verify row counts and checksums per table |
| 5 | Point the tenant's domains to the dedicated instance; issue TLS |
| 6 | Smoke test with the tenant owner; lift read-only on the dedicated instance |
| 7 | Keep the shared copy read-only for 30 days, then purge it (DEC-51) |

**Not supported:** merging a dedicated tenant back into the shared database (ID collisions), merging two organizations, or splitting a branch out into a new organization. These are handled as one-off data migration projects if ever required.

**Future option (not built now):** routing tenants to different database connections inside one application instance. The rules above keep it possible; it would need a connection resolver in `ResolveTenant` and per-connection queues.

---

## 4.21 Scenario Walkthroughs

| Aspect | A: Single school | B: Multi-branch chain | C: Many organizations | D: Dedicated instance |
|---|---|---|---|---|
| Rows | 1 organization, 1 institution, 1 branch | 1 organization, 1+ institutions, N branches | Many organizations, each as A or B | Same as A or B, alone in its database |
| Owner experience | One dashboard; branch switcher hidden; owner may also hold principal role (role switcher) | Consolidated dashboard plus per-branch drill-down; branch comparison | Each owner sees only their organization | Same as B |
| Principal | Sees everything in the one branch | Sees own branch only | Same | Same |
| Accountant | Branch scope = whole school | Branch or organization scope (head office accountant) | Same | Same |
| Guardians | One account per mobile; all children | Children in any branch under one account | One account per organization | Same |
| Platform super admin | Sees tenant metadata only | Same | Same | Platform admin of that instance |
| Data separation | Row scoping | Row scoping plus branch scoping | Row scoping between organizations | Physical database separation |

---

## 4.22 Edge Cases

| Case | Handling |
|---|---|
| Teacher whose child studies in the same school | One user: staff role assignments plus guardian links; portal switcher (DEC-47) |
| Guardian with children in two different organizations | Two accounts, one per organization (DEC-09) |
| Employee working for two different organizations | Two accounts |
| Siblings in two branches | One family; sibling discount counts them together when `sibling_scope = organization` |
| Staff member who is also a guardian and gets custody-restricted as guardian | Guardian links blocked; staff access unaffected |
| Branch moves from one institution to another | Not supported; create a new branch in the target institution and transfer students |
| Branch closure with active students | Deactivation blocked until students are transferred or withdrawn |
| Owner sells one branch to another owner | Not supported in R1 to R3; one-off migration project |
| Two organizations merge | Not supported in R1 to R3; one-off migration project |
| User of organization A opens a link from organization B (e.g., forwarded voucher link) | Signed links are user-bound; on B's host the user is not logged in; after logging in as an A user they are logged out by layer 3 |
| Device serial registered by mistake to the wrong organization | SA reassigns; past punches stay with the original organization and are reviewed manually |
| Guardian's only linked child withdraws | Account stays active with read-only access to that child's history for a configurable period (default 1 year), then portal access ends |
| Student transfers branch mid-month after voucher generated in A | Voucher stays in A; fee policy (4.12) decides collection |

---

## 4.23 Data Volume and Index Strategy

**Rough yearly volume for one organization with 1,000 students and 80 staff**

| Table | Rows per year | Notes |
|---|---|---|
| `student_attendances` | about 220,000 | 1,000 students x about 220 school days |
| `staff_attendances` | about 20,000 | |
| `attendance_punches` | about 400,000 | Gate and staff punches, duplicates collapsed |
| `fee_vouchers` | about 12,000 | Monthly plus ad hoc |
| `fee_voucher_items` | about 60,000 | |
| `fee_payments` | about 12,000 | |
| `journal_lines` | about 80,000 | Accrual postings plus payments |
| `mark_entries` | about 60,000 | 1,000 students x 8 subjects x several exams |
| `notification_dispatches` | about 150,000 | Absence alerts, receipts, announcements |
| `audit_logs` | about 300,000 | |

At 500 organizations of this size, `student_attendances` grows by about 110 million rows a year and `attendance_punches` by about 200 million. That is manageable on a well-indexed MySQL 8.4 server, but it drives these rules:

1. Every index on a tenant table starts with `organization_id`, then `branch_id` when present, then the main filter column (`attendance_date`, `fee_month`, `created_at`).
2. Log tables (DEC-48) are partition-ready: no foreign keys, a date column included in the primary key when partitioning is switched on (D7 defines the exact keys).
3. Dashboards and reports read `daily_metric_snapshots` instead of scanning attendance and fee tables.
4. Closed sessions older than a configurable number of years can be moved to archive tables for attendance (R3), with reports reading both through a view.

---

## 4.24 Isolation Test Plan

| Suite | What it does | Example tests |
|---|---|---|
| Tenant isolation (generated) | Finds every model using `BelongsToOrganization`; creates rows in organizations A and B; asserts A context never reads, counts, updates or deletes B rows | `tenant_isolation_for_{model}` |
| Route IDOR (generated) | For every route with a model binding, requests B's ULID while logged in as A | `route_{name}_returns_404_for_foreign_tenant` |
| Foreign key input | For every Form Request with a foreign-key field, submits B's ULID | `request_{name}_rejects_foreign_tenant_reference` |
| Branch isolation | Branch A principal against Branch B lists, records and reports | `branch_principal_cannot_list_other_branch_vouchers` |
| Visibility levels | Teacher, HOD, accountant, owner across assigned and non-assigned data | `teacher_cannot_view_unassigned_section_marks` |
| Guardian access | Guardian against non-linked students, restricted links, withdrawn children | `guardian_cannot_view_unlinked_student` |
| Fail-closed | Tenant model query with no context | `query_without_context_throws` |
| Jobs | Dispatch in A, run, assert only A affected; dispatch without context fails | `voucher_job_only_touches_dispatching_tenant` |
| Cache | Same logical key in A and B | `tenant_cache_keys_are_isolated` |
| Files | Signed URL used by another user and another tenant | `signed_url_rejected_for_other_tenant_user` |
| Exports | Export as branch user and as organization user | `branch_export_contains_only_branch_rows` |
| Database | Raw insert with cross-tenant parent | `composite_fk_rejects_cross_tenant_parent` |
| Webhooks | Wrong UUID, wrong signature, unknown device serial, payload referencing another tenant | `adms_rejects_unregistered_serial` |
| Cookies and hosts | Session cookie from host A presented to host B; unknown host | `session_not_shared_across_tenant_hosts` |
| Platform | SA without grant, SA with expired grant | `super_admin_without_grant_cannot_impersonate` |
| Mass assignment | Posting `organization_id` and `branch_id` in request bodies | `organization_id_in_request_is_ignored` |
| "All branches" mode | Attempted write with no active branch | `write_rejected_without_active_branch` |

These suites run on every commit in CI. A new tenant model or route is automatically covered by the generated suites; a failing generated test blocks the merge.

---

## Assumptions

| ID | Assumption |
|---|---|
| A-33 | All branches of an organization share one timezone (Pakistan uses one timezone); timezone is set per organization. |
| A-34 | A branch belongs to one institution for its lifetime. |
| A-35 | One shared MySQL primary (with a replica for reports and backups) serves up to about 2,000 small organizations before further scaling decisions are needed. |
| A-36 | The organization slug is chosen at provisioning and changed only by SA. |
| A-37 | Inter-branch fee balance transfer (`carry_to_target`) requires the Accounting module; without it only `settle_before_transfer` and `keep_in_source` are offered. |
| A-38 | Guardians keep read-only access to a withdrawn child's history for one year by default. |

## Gaps Found and Added

| ID | Gap | Decision | Release |
|---|---|---|---|
| GAP-75 | Staff who are also guardians in the same school | DEC-47 portal switcher | R1 |
| GAP-76 | School to college progression inside one group | DEC-52 | R1 |
| GAP-77 | Session cookies could leak across tenant subdomains | DEC-44 | R1 |
| GAP-78 | No custom domain support or hostname registry | DEC-50, `organization_domains` | R1 subdomain, R2 custom |
| GAP-79 | Attendance and log volume at platform scale | DEC-48, index rules, snapshots, archive plan | R1 rules, R3 archive |
| GAP-80 | Queries without tenant context could return all rows | DEC-45 fail closed | R1 |
| GAP-81 | Foreign-key inputs validated with unscoped `exists` rules | `TenantExists` and `BranchExists` rules, architecture test | R1 |
| GAP-82 | No defined tenant deletion process | DEC-51 purger registry | R2 |
| GAP-83 | Subdomain collisions with system hosts | Reserved slug list | R1 |
| GAP-84 | Previous-branch visibility after transfer undefined | Read-only "transferred out" view limited to own-branch rows | R1 |
| GAP-85 | Shared staff costs per branch | Optional cost split per branch assignment | R2 |
| GAP-86 | Fee balance on branch transfer undefined | Three policy options (4.12) | R1 |
| GAP-87 | Guardian access after a child leaves undefined | A-38 | R1 |
| GAP-88 | No "find your school" entry point that avoids listing tenants | School code entry on root domain | R1 |

---

## What Deliverable 5 Covers

The complete RBAC design and permission matrix: the full permission catalog per module (`{module}.{resource}.{action}` keys), the 23 system role templates with their default permissions and visibility levels in matrix tables, scope and override rules, approval permissions, sensitive-data permissions (health, counseling, salary, CNIC), portal access rules, the effective-permission algorithm, caching, and the RBAC test matrix.
