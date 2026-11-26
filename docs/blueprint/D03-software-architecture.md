# Deliverable 3: Recommended Software Architecture

**Version:** 1.0 (draft for approval) | **Date:** 23 Nov 2026
**Builds on:** D1 (decisions DEC-01 to DEC-20, module catalog, naming), D2 (requirements, DEC-21 to DEC-28)

---

## 3.1 Architecture in One Page

**Style:** Laravel modular monolith. One codebase, one deployable application, one MySQL database, 33 modules with enforced boundaries.

**Why not microservices:** one small team, one database transaction needed across fee, accounting and audit, and no part of the system needs independent scaling at launch. Microservices would add network failure modes, distributed transactions and deployment overhead with no benefit. Module boundaries are drawn so a module (for example Notification or Attendance ingestion) can be extracted later if load demands it.

```
+-----------------------------------------------------------------------------------+
|  Browsers: staff (app), guardian, student, platform    Devices / Gateways / SMS   |
+-------------------------+------------------------------------+--------------------+
                          | HTTPS                              | HTTPS webhooks
+-------------------------v------------------------------------v--------------------+
|  HTTP LAYER   Routes per portal -> Middleware stack -> Form Requests -> Controllers|
|               (Blade views, JSON endpoints for Alpine, API Resources, webhooks)   |
+-------------------------------------------+---------------------------------------+
                                            |
+-------------------------------------------v---------------------------------------+
|  APPLICATION LAYER   Actions (one use case each, own the transaction)             |
|                      Services (domain logic reused by several actions)            |
|                      Queries (read models for reports and dashboards)             |
+-------------------------------------------+---------------------------------------+
                                            |
+-------------------------------------------v---------------------------------------+
|  DOMAIN / DATA LAYER  Eloquent models, enums, casts (Money), global scopes        |
|                       (tenant, branch), policies, domain events                   |
+------------------+----------------------+---------------------+-------------------+
                   |                      |                     |
          +--------v-------+     +--------v--------+   +--------v--------+
          |  MySQL 8.4     |     |  Redis          |   | Object storage  |
          |  (one schema)  |     | cache + queues  |   | local or S3     |
          +----------------+     +--------+--------+   +-----------------+
                                          |
+-----------------------------------------v-----------------------------------------+
|  ASYNC LAYER   Horizon workers (named queues) | Scheduler (cron) | Gotenberg (PDF)|
+-----------------------------------------------------------------------------------+
```

---

## 3.2 New Decisions in This Deliverable

| ID | Decision | Reason |
|---|---|---|
| DEC-29 | ZKTeco ADMS devices post to fixed firmware paths (`/iclock/...`) on a dedicated host (`adms.{platform-domain}`). The tenant is resolved from the device serial number, which is globally unique in `attendance_devices`. This is the one exception to the D1 webhook URL convention | ADMS firmware cannot carry an organization UUID in the path |
| DEC-30 | Alpine.js CSP build (`@alpinejs/csp`) with a strict Content Security Policy (no `unsafe-eval`, no inline scripts except nonce-tagged bootstraps). All Alpine logic lives in registered `Alpine.data()` components | Strong XSS protection for a system holding minors' data |
| DEC-31 | PDF rendering through a `PdfRenderer` interface; production implementation calls a self-hosted Gotenberg container (headless Chromium) via `gotenberg/gotenberg-php` | Correct Urdu Nastaliq shaping, Chromium isolated from the PHP servers, no Node.js on app servers |
| DEC-32 | Excel import and export through `maatwebsite/excel` 3.1 (chunked reading, queued exports, template dropdown validation) | Supports Laravel 12 and 13 and PHP 8.2+, and handles data validation lists that streaming libraries lack |
| DEC-33 | Laravel Fortify as the headless authentication backend (login pipeline, password reset, TOTP two-factor, password confirmation) with custom Blade views and custom `authenticateUsing` for tenant and multi-identifier login | First-party, well tested, avoids hand-written auth security code |
| DEC-34 | Custom RBAC and custom audit log, no third-party permission or activity-log package | Scoped assignments, visibility levels, deny overrides and append-only audit are not covered by available packages; recent versions of popular packages also require PHP 8.4+ |
| DEC-35 | Modules are plain PSR-4 folders under `modules/` with one service provider each, registered in `bootstrap/providers.php`. No module-management package | Fewer dependencies, nothing hidden from the AI agent |
| DEC-36 | Business logic lives in **Actions** (one class per use case, owns the transaction) and **Services** (reusable domain logic). Controllers and models stay thin | One predictable place for every rule |
| DEC-37 | Modules contribute to shared engines through **registries** (permissions, settings, notification events, approval types, import types, document types, reports, dashboard widgets, onboarding steps, profile tabs, clearance checks, voucher line contributors) | Lower modules never depend on higher ones |
| DEC-38 | Redis for cache, queues and locks in staging and production; Laravel Horizon manages workers. Database session driver (the `sessions` table) so active sessions can be listed and revoked (FR-IAM-007) | Reliable queues with visibility; session listing per user |
| DEC-39 | Tests run against MySQL 8.4, never SQLite | The schema uses MySQL behavior (JSON, generated columns, CTEs, window functions, locking) |
| DEC-40 | Pest 3 (PHP 8.2+) until gate UG-1; Pest 4 after UG-1 | Pest 4 requires PHP 8.3 |
| DEC-41 | Domain events and queued jobs that cause side effects run only after the database transaction commits (`ShouldDispatchAfterCommit`, `ShouldHandleEventsAfterCommit`, `afterCommit()` on jobs) | No SMS or posting for a rolled-back change |
| DEC-42 | Inbound webhooks are stored raw in `webhook_receipts` first, acknowledged fast, then processed by a queued job idempotently | Devices and gateways retry; processing must survive retries and late arrivals |

---

## 3.3 Corrections to Earlier Deliverables

| Item | Correction | Why |
|---|---|---|
| D2 dependency map: Student and Guardian | **Guardian depends on Student** (it owns `guardian_students`, which references `students`). Student no longer depends on Guardian. The Guardians tab on the student profile is contributed by Guardian through the Student profile tab registry | Removes a circular dependency |
| D2 dependency map: Reporting | Report and metric definition contracts live in **Core**; the Reporting module provides dashboards, the runner and snapshots | Reporting reads everything, and modules must register reports without depending on Reporting |
| D2: Student promotion eligibility | Student defines a `PromotionEligibilityProvider` interface; Result implements and binds it. Without Result, promotion is manual | Student is lower than Result |
| D2: Transport fees and library fines on vouchers | Fee defines a `VoucherLineContributor` registry; Transport and Library register contributors | Fee is lower than Transport and Library |
| D2: Leave updating attendance | Leave's approval handler calls Attendance's `AttendanceLeaveWriter` contract. Attendance never listens to Leave events | Leave already depends on Attendance |
| D1 webhook URL convention | ADMS exception (DEC-29) | Firmware limitation |
| Core key entities | Add `idempotency_keys`, `webhook_receipts`, `bulk_operations` | Required by FR-GEN-013, DEC-42, FR-GEN-014 |

---

## 3.4 Module Boundary Rules (enforced by architecture tests)

| # | Rule |
|---|---|
| MB-01 | Each table is owned by exactly one module. Only the owning module writes to it. |
| MB-02 | Another module may **read** an owned model through Eloquent relationships or the owner's `Contracts` and `Queries`. It never calls `create`, `update`, `delete` or `save` on it. |
| MB-03 | Cross-module **writes** go through the owner's public Action or Contract (e.g., Admission calls `Student\Contracts\StudentEnroller`). |
| MB-04 | Cross-module **side effects** (notify, update timeline, recompute snapshot) go through domain events consumed by listeners in the reacting module. |
| MB-05 | A module only references classes of modules it depends on (dependency map, 3.3). A lower module that needs behavior from a higher one defines an interface or registry that the higher one fills. |
| MB-06 | Only `Contracts`, `Events`, `Enums`, `Data` (DTOs) and `Models` (read-only use) of another module may be imported. Never its controllers, actions internals, jobs or listeners. |
| MB-07 | Controllers do not contain queries beyond simple loading, never call `DB::` directly, and never contain business rules. |
| MB-08 | No `env()` outside `config/`. No raw SQL string concatenation. Raw expressions only with bindings, in `Queries` classes. |
| MB-09 | Every tenant-owned model uses `BelongsToOrganization` (and `BelongsToBranch` where it has `branch_id`). An architecture test fails the build if a migration has `organization_id` and its model lacks the trait. |

---

## 3.5 Module Anatomy

Full folder tree in D8. Inside each module:

```
modules/Fee/
  Actions/            One class per use case: RecordPayment, GenerateVouchers
  Approval/           ApprovalHandler implementations: PaymentReversalHandler
  Contracts/          Public interfaces other modules may use: FeeClearanceChecker
  Data/               Readonly DTOs: PaymentData, VoucherLineData
  Enums/              Backed string enums: VoucherStatus, PaymentMethod
  Events/             Domain events: PaymentRecorded
  Exceptions/         Domain exceptions with translatable messages
  Exports/ Imports/   ImportDefinition / ExportDefinition classes
  Http/
    Controllers/App/        Staff portal controllers
    Controllers/Guardian/   Guardian portal controllers
    Controllers/Student/    Student portal controllers
    Controllers/Webhooks/   Inbound integration controllers
    Controllers/Api/        Versioned API (R2)
    Requests/               Form Requests
    Resources/              API Resources (JSON shape)
  Jobs/               Queued jobs
  Listeners/          Event listeners
  Models/             Eloquent models
  Notifications/      Registration of notification events for the Notification engine
  Policies/           Model policies
  Queries/            Read models for lists, reports, dashboards
  Services/           Domain services: VoucherCalculator, AllocationService
  Providers/FeeServiceProvider.php
  database/migrations | factories | seeders
  resources/views     (namespace fee::)
  lang/en | lang/ur   (namespace fee::)
  routes/app.php | guardian.php | student.php | webhooks.php | api.php
```

Tests live in the root: `tests/Feature/Fee`, `tests/Unit/Fee`, `tests/Architecture`.

**Service provider responsibilities:** load migrations, views, translations and route files (each route file wrapped in its portal middleware group), register policies, register event listeners explicitly, bind Contracts to implementations, and register contributions into registries. Schedules are declared centrally in `routes/console.php` so the whole schedule is visible in one place.

**Migration order:** module migrations use timestamps that follow the phase order in D13, so cross-module foreign keys always point to tables that already exist.

---

## 3.6 Where to Use Each Laravel Building Block

| Building block | Use it for | Do not use it for |
|---|---|---|
| **Form Requests** | All input validation for web and API writes; `authorize()` delegates to the policy; normalization in `prepareForValidation()` (phone to E.164, trim, case) | Business rules that need other records' state (those go in Actions and throw domain exceptions) |
| **Controllers** | Receiving the request, calling one Action or Query, returning a view, redirect or resource | Business logic, queries with joins, transactions |
| **Actions** | Each user-visible use case: `RecordPayment`, `MarkSectionAttendance`, `PromoteStudents`. Own the DB transaction, call services, write audit with reason, fire events | Being called by models or views |
| **Services** | Logic shared by several actions or pure calculation: `VoucherCalculator`, `ResultCalculator`, `PermissionResolver`, `NumberingService`, `PostingService` | Wrapping a single Eloquent call |
| **Queries** | Read models: paginated lists with filters, reports, dashboard metrics, exports. May use query builder with bindings | Writing data |
| **Repositories** | Only external adapters (SMS, WhatsApp, payments, devices, PDF, GPS) behind interfaces (DEC-19) | Wrapping Eloquent models |
| **Database transactions** | Admission conversion, enrollment changes, promotion chunks, voucher generation chunks, payments, reversals, refunds, cashier close, journal posting, marks submission and approval, result processing chunks, approval application, attendance finalization, payroll finalization, imports per chunk | Long-running loops across thousands of rows in one transaction |
| **Row locks** (`lockForUpdate`) | Numbering sequences, voucher during payment allocation, cashier session during close, leave balance during approval, attendance day during finalization | Read-only screens |
| **Events and listeners** | Cross-module side effects: notifications, timeline entries, snapshot refresh, audit of low-risk events | Core logic the action itself must complete (that must be inside the transaction) |
| **Queued jobs** | Bulk work (vouchers, results, promotions, PDFs, imports, exports), notifications, webhook processing, device punch resolution, snapshot computation | Anything a user must see completed in the same response (save a payment) |
| **Job batches** | Bulk operations split into chunks with progress (`Bus::batch`) linked to `bulk_operations` | Single small jobs |
| **Policies** | Every model a user can act on: checks permission key, organization, branch scope and visibility level | Hiding buttons only (Blade `@can` uses the same policy, but the server check is what counts) |
| **Gates** | Non-model abilities: `view-dashboard-owner`, `switch-branch`, `impersonate` | Model actions (use policies) |
| **Middleware** | Tenant resolution, tenant-user match, branch context, module entitlement, portal guard, locale, idle timeout, two-factor enforcement, correlation ID, security headers, webhook signature verification, rate limits | Authorization of specific records |
| **Notifications** (Laravel) | Only as transport inside the Notification module: one generic `TemplatedNotification` for database (in-app) and mail channels | Modules sending messages directly (modules fire domain events; the Notification engine decides) |
| **Scheduled commands** | Time-based work, each command fanning out one job per organization in that organization's timezone (3.11) | Heavy work inside the command process itself |
| **API Resources** | Every JSON response: Alpine endpoints, webhooks responses where structured, API v1 | Blade views |
| **Blade components** | All UI: `x-ui.*` primitives (D9) and `x-{module}.*` composites; layouts per portal | Copy-pasted markup between screens |
| **Enums** | All statuses and types (`VoucherStatus`, `AttendanceStatus`) as backed string enums with `label()` translations | Magic strings in code |
| **Casts** | `Money`, enums, `encrypted` for credentials and sensitive fields, `immutable_datetime` | Business calculations |
| **Observers / model events** | Only the `Auditable` trait and tenancy fill | Business side effects (use Actions and events) |
| **Soft deletes** | Master data only (subjects, rooms, fee heads, templates) | Payments, journals, attendance, marks, results, enrollments (PR-02) |

---

## 3.7 Request Lifecycle

```
Browser: GET https://cityschool.darsora.app/app/fee/vouchers?class=9
   |
   v
[1] AssignCorrelationId      -> X-Correlation-Id, added to log context
[2] SecurityHeaders          -> CSP (nonce), HSTS, X-Frame-Options, Referrer-Policy
[3] ResolveTenant            -> subdomain "cityschool" -> organization (cached) -> TenantContext
                                unknown or archived -> 404 page
[4] StartSession / Auth      -> session (database driver), Fortify guard "web"
[5] EnsureTenantMatchesUser  -> user.organization_id == context org, else logout + security event
[6] EnsureActiveUser         -> deactivated or custody-restricted -> logout
[7] RequireTwoFactor         -> role demands 2FA and not confirmed -> challenge
[8] IdleTimeout              -> per-role idle limit
[9] EnsurePortal:app         -> user has at least one staff role assignment
[10] SetBranchContext        -> active branch from session; validated against user's assignments
[11] EnsureModuleEnabled:fee -> entitlement check (404 if disabled)
[12] SetLocale               -> user language, dir=rtl for Urdu, Carbon locale
[13] Route model binding     -> by ULID, through global scopes (other tenant -> 404, DEC-27)
[14] FormRequest             -> authorize(): policy -> permission + scope + visibility
                                rules(): validation
[15] Controller              -> calls Query (list) or Action (write)
[16] Action                  -> DB::transaction { locks, rules, writes, audit } -> events after commit
[17] Response                -> Blade view or API Resource JSON
```

**TenantContext** (singleton, Core): organization id, active branch id, allowed branch ids, visibility level per permission, user id, impersonator id, locale, timezone, correlation id.

**Queued jobs** carry the context: every tenant job uses the `TenantAware` job middleware, which serializes organization id, branch id, user id and correlation id at dispatch and restores them before `handle()`. A job without context that touches tenant models fails loudly.

**Console commands** never run with an implicit tenant. They loop organizations explicitly and dispatch tenant-aware jobs.

---

## 3.8 Authorization Architecture (detail in D5)

```
User
 +-- user_role_assignments (role, scope_type, scope_ids, visibility, active dates)
 |       +-- role_permissions (role -> permission keys)
 +-- user_permission_overrides (grant | deny, scope, expiry)

PermissionResolver (cached per user per org, key versioned on every change)
   effective(user, branch) = (union of role permissions for assignments covering branch)
                             + grants - denies
   visibility(user, permission, branch) = widest visibility among covering assignments
```

- **Policies** call `$this->allows($user, 'fee.voucher.view', $voucher)`, a shared helper that checks permission key, record organization, record branch within the user's scope, and visibility (`own`, `assigned`, `branch`, `institution`, `organization`).
- **Lists** use the query scope `->visibleTo($user, 'student.student.view')`, which applies branch filters and, for `assigned`, joins through teaching assignments or class teacher sections.
- **Platform super admin** uses a separate domain and portal (`platform`) and has no `Gate::before` bypass into tenant data (BR-PLT-01). Support access works through time-limited impersonation of a tenant user.
- **Guardians** are authorized through `GuardianChildAccess`: the requested child must be in the user's active, portal-enabled, unrestricted links.
- **Students** see only their own records.

---

## 3.9 Data Consistency Patterns

| Pattern | Implementation |
|---|---|
| Tenant fill and scope | `BelongsToOrganization` trait: global scope plus `creating` hook that sets `organization_id` from TenantContext and ignores request input |
| Public identifiers | `uuid` column holding a ULID (`HasUlids` on a non-primary column via a Core trait `HasPublicUlid`); route keys use it |
| Idempotency | Client sends `Idempotency-Key` header (or hidden form field generated per form render). `IdempotencyGuard` stores key, user, route, request hash and response reference in `idempotency_keys` for 24 hours; a repeat returns the stored result |
| Optimistic locking | Edit forms carry the record's `updated_at`; the Form Request rejects the save if it changed (FR-GEN-010) |
| Numbering | `NumberingService::next('fee_receipt', scope)` locks the sequence row inside the caller's transaction, so a rolled-back payment does not consume a gapless receipt number |
| Money | `Money` value object (amount in integer minor units plus currency), `MoneyCast` to DECIMAL(15,2); arithmetic only through `Money` methods |
| After-commit side effects | DEC-41 |
| Webhook intake | DEC-42 |
| Versioned corrections | Attendance and marks keep a revision table; the current row points to its latest revision |
| Reversals | Payments, journals, payroll: a new reversing record links to the original; the original is marked reversed, never edited |

---

## 3.10 Domain Event Catalog (cross-module)

| Event | Emitted by | Consumed by |
|---|---|---|
| `OrganizationProvisioned` | Platform | Settings (defaults), Notification (welcome) |
| `BranchCreated` | Organization | Settings, Reporting |
| `UserCreated`, `PermissionsChanged` | Identity | Audit, Identity cache invalidation |
| `SessionActivated`, `SessionClosed` | Academic | Learning (create courses), Reporting |
| `TeachingAssignmentChanged` | Academic | Timetable (revalidate load), Learning (course teacher) |
| `ApplicationSubmitted`, `ApplicationApproved` | Admission | Notification |
| `StudentAdmitted` | Admission | Notification (guardian activation SMS), Reporting |
| `StudentEnrolled`, `StudentPromoted`, `StudentSectionChanged`, `StudentTransferred`, `StudentWithdrawn`, `StudentStruckOff`, `StudentGraduated` | Student | Notification, Reporting, Transport (end allocation on withdrawal), Library (membership) |
| `GuardianLinked`, `CustodyRestrictionFlagged` | Guardian | Identity (revoke sessions on restriction), Notification |
| `EmployeeHired`, `EmployeeStatusChanged`, `EmployeeSeparated` | HR | Identity (deactivate user), Attendance (device enrollment), Academic (end teaching assignments) |
| `SectionAttendanceMarked`, `AttendanceFinalized` | Attendance | Reporting |
| `StudentMarkedAbsent`, `StudentMarkedLate` | Attendance | Notification |
| `AttendanceCorrected` | Attendance | Notification, Reporting |
| `DeviceWentOffline`, `SuspiciousAttendanceDetected` | Attendance | Notification (IT admin, principal) |
| `LeaveApproved`, `LeaveRejected` | Leave | Notification, Timetable (substitution prompt, R2) |
| `VoucherGenerated`, `VouchersBatchGenerated` | Fee | Notification |
| `PaymentRecorded` | Fee | Notification (receipt SMS), Admission (application paid), Reporting |
| `PaymentReversed`, `ChequeBounced`, `RefundIssued` | Fee | Notification, Reporting |
| `FeeOverdue`, `StruckOffCandidatesIdentified` | Fee | Notification |
| `JournalPosted`, `FiscalYearClosed` | Accounting | Reporting |
| `DateSheetPublished` | Examination | Notification |
| `MarksSubmitted`, `MarksApproved`, `MarksReturned` | Examination | Notification, Result (mark batch stale) |
| `ResultPublished`, `ResultWithheld` | Result | Notification, Reporting |
| `TimetableActivated`, `SubstitutionAssigned` | Timetable | Notification |
| `AnnouncementPublished`, `DiaryPublished` | Communication | Notification |
| `ClosureDeclared` | Administration | Attendance (set `H`), Timetable, Notification |
| `ApprovalRequestSubmitted`, `ApprovalStepDecided`, `ApprovalRequestApproved`, `ApprovalRequestRejected`, `ApprovalRequestReturned` | Approval | Notification |
| `ImportCompleted`, `ExportReady` | ImportExport | Notification (in-app) |
| `DocumentIssued` | Document | Audit |
| `LecturePublished`, `HomeworkAssigned`, `QuizPublished` (R2) | Learning, Homework, Assessment | Notification |
| `PayrollRunFinalized` (R2) | Payroll | Notification |

---

## 3.11 Queues, Jobs and Scheduler

**Queues (Horizon supervisors)**

| Queue | Contents | Workers (start) | Timeout | Tries |
|---|---|---|---|---|
| `critical` | Payment callbacks, OTP and password reset messages, webhook receipt processing for gateways | 2 | 60 s | 5 |
| `attendance` | Punch ingestion, daily resolution, finalization | 2 | 120 s | 3 |
| `notifications` | SMS, email, WhatsApp, in-app dispatch | 3 | 60 s | 3 (backoff 1, 5, 15 min) |
| `bulk` | Voucher generation chunks, result processing chunks, promotion chunks, payroll chunks | 2 | 600 s | 1 (chunk is transactional; failed chunk reported) |
| `documents` | PDF rendering and merging | 2 | 300 s | 2 |
| `imports` | Validation and commit chunks | 1 | 900 s | 1 |
| `exports` | Large exports | 1 | 900 s | 1 |
| `reports` | Snapshot computation, scheduled reports | 1 | 600 s | 2 |
| `default` | Everything else | 2 | 120 s | 3 |

Per-organization fairness: bulk jobs use `WithoutOverlapping` keyed by organization and operation, so one tenant's month-end run cannot block another's.

**Scheduler (`routes/console.php`, all `onOneServer()->withoutOverlapping()`)**

| Command | Frequency | Effect |
|---|---|---|
| `attendance:resolve-punches` | every 5 min | Resolve new device punches into daily staff attendance |
| `attendance:finalize-windows` | every 15 min | Finalize sections whose marking window closed; trigger absence alerts |
| `attendance:not-marked-alerts` | every 15 min (school hours) | Alert class teachers and principal |
| `attendance:auto-absent` | every 15 min | Apply auto-absent after cut-off for staff |
| `attendance:device-health` | every 15 min | Detect offline devices |
| `fee:mark-overdue` | daily 00:15 org time | Set overdue status |
| `fee:apply-late-fees` | daily 00:30 org time | Calculate late fees per rules |
| `fee:struck-off-candidates` | monthly, day 1 | Build list for principal |
| `fee:reminders` | daily 10:00 org time | Due-date and overdue reminders |
| `notifications:release-quiet-hours` | daily 07:00 org time | Send held messages |
| `reporting:snapshots-today` | every 15 min | Today's attendance, collection, pending approvals |
| `reporting:snapshots-nightly` | daily 01:00 org time | Monthly and session metrics |
| `results:publish-scheduled` | every 5 min | Publish results whose time arrived |
| `hr:document-expiry-reminders` | daily 09:00 | Expiry reminders |
| `leave:accrue` | monthly, day 1 | Accrue balances |
| `core:purge-idempotency-keys` | hourly | Remove keys older than 24 h |
| `import-export:purge-expired` | hourly | Delete expired export files and temp uploads |
| `identity:prune-otps` | hourly | Remove used and expired OTPs |
| `approvals:escalate` (R2) | hourly | SLA escalation |
| `assessment:auto-submit` (R2) | every minute | Submit expired quiz attempts |
| `audit:archive` (R3) | monthly | Move old audit rows to archive storage |

Each tenant-level command dispatches one job per active organization, evaluated in that organization's timezone.

---

## 3.12 Caching

| What | Key pattern | Invalidation |
|---|---|---|
| Tenant by subdomain | `tenant:sub:{subdomain}` | On organization update or state change |
| Effective permissions | `org:{org}:perm:user:{user}:v{version}` | Version bump on role, assignment or override change |
| Resolved settings | `org:{org}:settings:{scope}:{id}:v{version}` | Version bump on settings change |
| Translation overrides | `org:{org}:i18n:{locale}:v{version}` | Version bump on override change |
| Lookup lists (levels, subjects, fee heads) | `org:{org}:lookup:{name}:v{version}` | Version bump on change in the owning module |
| Entitlements | `org:{org}:entitlements:v{version}` | On plan or module override change |

Rules: every tenant cache key starts with `org:{id}:`. Dashboards read from `daily_metric_snapshots` (database), not from cache, so numbers are consistent and auditable.

---

## 3.13 File Storage

| Disk | Driver | Contents |
|---|---|---|
| `private` | local (default) or S3-compatible | All tenant files: photos, documents, attachments, generated PDFs, imports, exports |
| `public_assets` | local or S3 with CDN | Only branding assets needed on public pages (logo on admission page, favicon) |

- Path: `org/{organization_uuid}/{module}/{yyyy}/{mm}/{ulid}.{ext}`. Original file names stored only in metadata.
- Downloads go through `core.files.show` with a short-lived signed URL; the controller re-checks the owning record's policy before streaming.
- Images are resized and recompressed on upload (Intervention Image); EXIF data stripped (privacy: GPS in phone photos).
- MIME type checked by content, not extension. Optional ClamAV scan through a `VirusScanner` adapter.
- Per-organization storage usage tracked in `usage_counters` (R2) and enforced against plan limits.

---

## 3.14 PDF and Document Rendering

```
Action (e.g., IssueLeavingCertificate)
   -> DocumentComposer: template (Blade, per type) + data DTO + branding + locale
   -> HTML with embedded fonts (Noto Nastaliq Urdu, Noto Sans / Inter)
   -> PdfRenderer interface
        GotenbergPdfRenderer (production): HTTP to gotenberg:3000 (Chromium route)
        FakePdfRenderer (tests): returns fixed bytes, records calls
   -> stored on private disk -> IssuedDocument record (certificates) -> signed download
Bulk: RenderDocumentBatch jobs (queue documents) -> Gotenberg merge route -> one PDF
```

- Gotenberg runs as a separate container on the app host or a worker host, reachable only on the private network.
- Templates use print CSS with fixed page sizes (A4, Legal, A4 landscape 3-copy challan, CR80 sheets).
- Phase 0 spike: render a voucher and a report card with Urdu names and remarks, compare against expected output, confirm timing for 1,000-voucher batches.

---

## 3.15 Integration Adapter Layer

| Interface (Contracts) | Owner module | Implementations | Release |
|---|---|---|---|
| `SmsGateway` | Notification | `HttpSmsGateway` (configurable aggregator template), named aggregator adapters added as contracts are signed, `LogSmsGateway` (local and test) | R1 |
| `WhatsAppGateway` | Notification | Meta Cloud API adapter or approved provider adapter | R2 |
| `PdfRenderer` | Document | `GotenbergPdfRenderer`, `FakePdfRenderer` | R1 |
| `AttendanceDeviceAdapter` | Attendance | `ZktecoAdmsAdapter`, `SignedWebhookAdapter`, `CsvPunchFileAdapter` | R1 |
| `PaymentGateway` | Fee | JazzCash, Easypaisa adapters (official merchant APIs) | R2 |
| `BillPaymentProvider` | Fee | 1Bill adapter (inquiry and payment notification) | R2 |
| `GpsProvider` | Transport | Provider adapters | R3 |
| `VirusScanner` | Core | `ClamAvScanner`, `NullScanner` | R1 optional |

- Credentials per organization stored with the `encrypted:array` cast; never logged; masked in UI after save.
- Inbound flow (DEC-42): verify signature or device identity, store raw payload in `webhook_receipts` (provider, organization, external id, payload, headers, received at, status), respond quickly, process in a queued job keyed by external id so repeats are ignored.

**ZKTeco ADMS specifics (DEC-29)**

| Endpoint | Purpose |
|---|---|
| `GET /iclock/cdata?SN=...` | Device handshake and options |
| `POST /iclock/cdata?SN=...&table=ATTLOG` | Punch upload (tab-separated lines) |
| `GET /iclock/getrequest?SN=...` | Device polls for commands (R2 command queue) |
| `POST /iclock/devicecmd?SN=...` | Command results (R2) |

Controls: only registered, active serial numbers are accepted; unknown serials are logged and rejected; per-device rate limit; optional IP allowlist per device; payload parsed strictly (only punch fields used); device clock drift beyond 5 minutes flagged (A-25). Many devices only speak plain HTTP, so the ADMS host accepts HTTP only for `/iclock/*` and nothing else; this is recorded as an accepted risk in D12.

---

## 3.16 HTTP Surface

| Surface | Prefix | Auth | Format |
|---|---|---|---|
| Staff portal | `/app/...` | Session (Fortify), CSRF | Blade; JSON endpoints for interactive grids under the same prefix |
| Guardian portal | `/guardian/...` | Session, CSRF | Blade (lighter JS bundle) |
| Student portal | `/student/...` | Session, CSRF | Blade |
| Platform portal | `/platform/...` on platform domain | Session, separate login, 2FA mandatory | Blade |
| Public | `/admissions/...`, `/verify/...` (R2) | None, captcha and rate limits | Blade |
| Webhooks | `/webhooks/{provider}/{organization_uuid}`, ADMS `/iclock/*` | Signature or device identity | Provider format |
| API v1 (R2) | `/api/v1/...` | Sanctum tokens with abilities | JSON via API Resources |

JSON conventions (internal and API): envelope `{ "data": ..., "meta": ... }`; errors `{ "message": ..., "errors": { field: [...] }, "code": "domain_error_code", "correlation_id": ... }`; pagination meta; `Idempotency-Key` header accepted on writes; rate limits per user and per organization. Full API architecture document is part of D8.

---

## 3.17 Frontend Architecture

- **Layouts:** one per portal (`layouts.app`, `layouts.guardian`, `layouts.student`, `layouts.platform`, `layouts.public`, `layouts.print`), all built from the same `x-ui.*` components.
- **Components:** anonymous Blade components for primitives (button, input, badge, card, modal, table shell); class-based components where data preparation is needed (data table with filters, branch switcher, notification bell).
- **Interactivity:** Alpine.js CSP build (DEC-30); components registered in `resources/js/components/*.js` via `Alpine.data`. Interactive grids (attendance marking, marks entry, timetable editor, quiz attempt) load JSON and save through JSON endpoints with idempotency keys and autosave.
- **Page model:** server-rendered pages with filters in the query string; full page navigation; no SPA router.
- **Styling:** Tailwind CSS v4, CSS-first configuration with `@theme` tokens matching D9 names; RTL through `dir="rtl"` on `<html>` and logical utilities (`ms-*`, `me-*`, `ps-*`, `pe-*`, `start-*`, `end-*`); physical left/right utilities are banned by lint rule.
- **Bundles:** `app.js` (staff), `portal.js` (guardian and student, smaller), `print.css`. Charts (Chart.js) loaded only on dashboard pages.
- **Forms:** server validation is the source of truth; client hints only (required, maxlength, inputmode for numbers and phone).

---

## 3.18 Localization Architecture

- Module translation files: `modules/{Module}/lang/{en|ur}/*.php`, used as `fee::vouchers.title`. Shared UI strings in `lang/{en|ur}/ui.php`.
- A decorating translation loader applies `translation_overrides` per organization (terminology changes) on top of file translations.
- `SetLocale` middleware sets app locale, Carbon locale and `dir`. Numbers render with Western digits; dates use per-locale formats from settings.
- Enum labels and notification templates are translated; stored data keeps both English and optional Urdu names where defined (names, subjects, fee heads, levels).
- PDF templates choose language per document setting (English, Urdu, or bilingual).

---

## 3.19 Error Handling and Logging

| Situation | Behavior |
|---|---|
| Record outside tenant or scope | 404 page or JSON (DEC-27) |
| Permission denied within tenant | 403 with translated message |
| Validation failure | 422 with field errors |
| Domain rule violation (`DomainRuleViolation` subclasses: `PeriodLocked`, `VoucherAlreadyPaid`, `StaleApprovalRequest`) | 422 with translated message and domain error code |
| Optimistic lock conflict | 409 with "record changed, reload" message |
| Entitlement missing | 404 |
| Unexpected error | 500 page with correlation ID shown to the user for support |

Logging: JSON structured logs with correlation id, organization id, branch id, user id, route. Channels: `app` (daily), `security` (tenant mismatch, lockouts, rejected webhooks, impersonation), `integrations` (provider requests and responses with secrets removed). Audit records are never written to log files. Error monitoring through Sentry-compatible SDK (optional; self-hosted GlitchTip is compatible) configured in D12.

---

## 3.20 Testing Architecture (full strategy in D12)

- **Runner:** Pest 3 on PHPUnit 11 until UG-1 (DEC-40); MySQL 8.4 test database (DEC-39) with `RefreshDatabase` and parallel testing.
- **Factories** are tenant-aware: `Organization::factory()->withBranch()->create()`, helpers `actingAsRole('principal', $branch)`, `inTenant($org, fn () => ...)`.
- **Tenant isolation suite:** a test discovers every model using `BelongsToOrganization` and asserts organization A cannot read, list, update or delete organization B's records through routes and queries.
- **Architecture tests** (Pest arch plugin): module boundary rules MB-01 to MB-09, no `env()` outside config, no `dd`/`dump`, controllers do not use `DB` facade, actions are final, enums are backed.
- **Static analysis:** Larastan at level 6 in Phase 1, raised to level 8 by Phase 6.
- **Fakes:** every adapter interface has a fake used in tests; no test calls a real SMS, payment or PDF service.

---

## 3.21 Package List

**Rule (DEC-01):** a runtime package must support PHP 8.2 through 8.4 and Laravel 12 and 13. Each Antigravity phase prompt runs `composer prohibits laravel/framework 13.0` and `composer prohibits php 8.4` after adding a package, and stops if anything is reported.

| Package | Constraint | Purpose | Status |
|---|---|---|---|
| `laravel/framework` | `^12.0` (then `^13.0` at UG-1) | Framework | Required |
| `laravel/fortify` | `^1.0` | Auth backend, 2FA (DEC-33) | First-party; verify constraint at install |
| `laravel/sanctum` | `^4.0` | API tokens | First-party; verify at install |
| `laravel/horizon` | `^5.0` | Queue workers and dashboard (DEC-38) | First-party; verify at install |
| `laravel/reverb` | `^1.0` | Realtime (optional, R2) | First-party; verify at install |
| `gotenberg/gotenberg-php` | `^2.0` | PDF via Gotenberg (DEC-31) | Verified: PHP 8.1 to 8.5 |
| `maatwebsite/excel` | `^3.1` | Imports and exports (DEC-32) | Verified: Laravel up to 13 |
| `intervention/image` | `^3.0` | Image resize and compression | Verify at install |
| `bacon/bacon-qr-code` | as required by Fortify | QR codes (ID cards, QR attendance, verification) | Installed with Fortify |
| `giggsey/libphonenumber-for-php-lite` | `^8.0` or current major | Phone normalization to E.164 | Verify at install |
| `symfony/html-sanitizer` | matching installed Symfony | Sanitize rich text (announcements, lectures) | Verify at install |
| `league/flysystem-aws-s3-v3` | `^3.0` | S3-compatible storage (optional) | Verify at install |
| `sentry/sentry-laravel` | current | Error monitoring (optional) | Verify at install |

**Dev dependencies**

| Package | Constraint | Purpose | Status |
|---|---|---|---|
| `pestphp/pest`, `pestphp/pest-plugin-laravel`, `pestphp/pest-plugin-arch` | `^3.0` until UG-1, then `^4.0` | Tests | Verified: Pest 3 needs PHP 8.2, Pest 4 needs PHP 8.3 |
| `larastan/larastan` | `^3.0` | Static analysis | Verify at install |
| `laravel/pint` | `^1.0` | Code style | First-party |
| `laravel/pail` | `^1.0` | Local log tailing | First-party |
| `laravel/telescope` | `^5.0`, local only | Local debugging; never installed in production | First-party |

**Frontend (npm)**

| Package | Purpose |
|---|---|
| `tailwindcss` v4, `@tailwindcss/vite` | Styling (DEC-06) |
| `alpinejs` CSP build (`@alpinejs/csp`), `@alpinejs/focus` | Interactivity (DEC-30) |
| `chart.js` v4 | Dashboard charts |
| `tom-select` | Searchable selects |
| `laravel-vite-plugin`, `vite` | Build |

**Infrastructure services:** MySQL 8.4 LTS, Redis 7, Gotenberg 8 container, Nginx, PHP-FPM 8.4 (8.2 locally until upgraded), Supervisor (Horizon, scheduler), optional ClamAV.

---

## 3.22 Options Considered and Rejected

| Option | Decision | Reason |
|---|---|---|
| Microservices | Rejected | See 3.1 |
| Multi-database tenancy package | Rejected for now | Shared schema with row scoping (DEC-03) is simpler to operate; dedicated database remains possible per D4 |
| Module-management package | Rejected | DEC-35 |
| Livewire, Inertia with Vue/React | Rejected | DEC-20; one interaction model, simpler for AI agent consistency |
| Third-party permission package | Rejected | DEC-34; scoped assignments and visibility levels |
| Third-party activity log package | Rejected | DEC-34; append-only, masking and in-transaction writes required |
| PHP-only PDF engines (dompdf, mPDF) | Rejected | Unreliable Urdu Nastaliq shaping (DEC-12) |
| Browsershot on app servers | Rejected | Needs Node and Chromium on every app server; Gotenberg isolates it |
| Repository per model | Rejected | DEC-19 |
| SQLite tests | Rejected | DEC-39 |
| Event sourcing | Rejected | Revision tables and audit give the needed history at much lower complexity |
| GraphQL | Rejected | REST with API Resources is enough; fewer moving parts |

---

## 3.23 Registries (DEC-37)

| Registry | Lives in | Modules contribute |
|---|---|---|
| `PermissionRegistry` | Identity | Permission keys with labels, module, screen, action |
| `SettingRegistry` | Settings | Setting definitions |
| `NotificationEventRegistry` | Notification | Event class, template key, placeholders, recipient resolvers |
| `RecipientResolverRegistry` | Notification | Resolvers like `guardians_of_student`, `class_teacher_of_section` |
| `ApprovalTypeRegistry` | Approval | Request type, handler class, payload rules, default workflow |
| `ApproverResolverRegistry` | Approval | Resolvers like `class_teacher_of_student`, `hod_of_subject` |
| `ImportRegistry`, `ExportRegistry` | ImportExport | Definitions |
| `DocumentTypeRegistry` | Document | Document types, data providers, default templates |
| `ReportRegistry`, `MetricRegistry`, `DashboardWidgetRegistry` | Core (contracts), Reporting (runtime) | Reports, metrics, widgets |
| `OnboardingStepRegistry` | Organization | Wizard steps (academic defaults, fee heads, grading) |
| `StudentProfileTabRegistry` | Student | Tabs: guardians, fees, attendance, results, discipline, learning |
| `EmployeeProfileTabRegistry` | HR | Tabs: attendance, leave, payroll, teaching load |
| `ClearanceCheckRegistry` | Student | Checks: fees, library, transport, documents |
| `SessionCloseCheckRegistry` | Academic | Checks: results published, promotions done, fee carry-forward |
| `VoucherLineContributor` registry | Fee | Transport fees, library fines |

---

## 3.24 Per-Module Technical Breakdown

Responsibilities and dependencies are in D2 (sections 2.3 to 2.36). Below: models, controllers, actions and services, policies, events emitted, listeners, jobs, and JSON or webhook endpoints. Controllers are under `Http/Controllers/App` unless a portal is named.

### Core
- **Models:** NumberingSequence, StoredFile, FileAttachment, PeriodLock, IdempotencyKey, WebhookReceipt, BulkOperation
- **Services:** TenantContext, NumberingService, PeriodLockService, FileStorageService, IdempotencyGuard, ImageProcessor; contracts `EntitlementChecker`, `VirusScanner`, report and metric definitions
- **Traits/Casts:** BelongsToOrganization, BelongsToBranch, HasPublicUlid, MoneyCast, Money
- **Middleware:** AssignCorrelationId, SecurityHeaders, ResolveTenant, EnsureTenantMatchesUser, SetBranchContext, EnsureModuleEnabled, SetLocale, VerifyWebhookSignature, TenantAware (job middleware)
- **Controllers:** FileController (signed download), BulkOperationController (progress JSON), PeriodLockController, HealthController
- **Jobs:** PurgeIdempotencyKeys, PurgeTempFiles

### Platform
- **Models:** Plan, PlanModule, OrganizationSubscription, OrganizationModule, UsageCounter, SupportAccessGrant, PlatformAnnouncement
- **Controllers (Platform):** PlanController, OrganizationController, OrganizationModuleController, SupportAccessController, PlatformAnnouncementController; **(App):** SupportAccessGrantController
- **Actions:** ProvisionOrganization, ChangeTenantState, OverrideModuleEntitlement, GrantSupportAccess, StartImpersonation, EndImpersonation
- **Services:** EntitlementService (implements Core `EntitlementChecker`), LimitEnforcer
- **Policies:** PlanPolicy, OrganizationPolicy, SupportAccessGrantPolicy
- **Events:** OrganizationProvisioned, TenantStateChanged, SupportAccessGranted, ImpersonationStarted, ImpersonationEnded
- **Jobs:** RecordDailyUsage (R2), BuildTenantExport (R2)

### Organization
- **Models:** Organization, Institution, Branch, BranchBankAccount, Department, Shift, Room
- **Controllers:** OrganizationProfileController, InstitutionController, BranchController, BranchBankAccountController, DepartmentController, ShiftController, RoomController, StructureController, OnboardingWizardController
- **Actions:** UpdateOrganizationProfile, CreateBranch, DeactivateBranch, RunOnboardingStep, CompleteOnboarding
- **Services:** OnboardingStepRegistry
- **Policies:** OrganizationPolicy, InstitutionPolicy, BranchPolicy, DepartmentPolicy, ShiftPolicy, RoomPolicy
- **Events:** BranchCreated, BranchDeactivated, OnboardingCompleted

### Identity
- **Models:** User, Role, Permission, UserRoleAssignment, UserPermissionOverride, OneTimePassword, TrustedDevice
- **Controllers:** Auth\LoginController, Auth\OtpActivationController, Auth\PasswordResetController (Fortify views), ProfileController, SessionController, TwoFactorController, RoleController, RolePermissionMatrixController, UserController, UserRoleAssignmentController, UserPermissionOverrideController, EffectivePermissionController, ContextSwitchController, TrustedDeviceController, ApiTokenController
- **Actions:** CreateStaffUser, CreateGuardianUser, CreateStudentUser, AssignRole, RevokeRole, GrantPermissionOverride, DeactivateUser, ActivateGuardianAccount, RegisterTrustedDevice
- **Services:** PermissionRegistry, PermissionResolver, VisibilityScope, OtpService, LoginIdentifierNormalizer
- **Middleware:** EnsurePortal, EnsureActiveUser, IdleTimeout, RequireTwoFactor
- **Policies:** RolePolicy, UserPolicy, UserRoleAssignmentPolicy
- **Events:** UserCreated, UserDeactivated, RoleAssigned, RoleRevoked, PermissionsChanged
- **Listens:** CustodyRestrictionFlagged, EmployeeSeparated (deactivate and revoke sessions)
- **Commands:** `identity:sync-permissions` (run on every deploy)

### Settings
- **Models:** Setting, TranslationOverride
- **Controllers:** SettingController (per category), BrandingController, LanguageController, TerminologyController
- **Services:** SettingRegistry, SettingResolver, BrandingService, OverrideTranslationLoader
- **Policies:** SettingPolicy
- **Events:** SettingChanged

### Audit
- **Models:** AuditLog, LoginEvent
- **Controllers:** AuditLogController, RecordHistoryController, LoginHistoryController
- **Services:** AuditLogger (explicit entries with reason), Auditable trait, ValueMasker
- **Policies:** AuditLogPolicy
- **Listens:** Laravel auth events (Login, Logout, Failed, PasswordReset, Lockout)
- **Jobs:** WriteAuditLog (low-risk async), ArchiveAuditLogs (R3)

### Approval
- **Models:** ApprovalWorkflow, ApprovalWorkflowStep, ApprovalRequest, ApprovalRequestStep, ApprovalAction, ApprovalDelegation
- **Contracts:** ApprovalHandler (`type()`, `rules()`, `subject()`, `describe()`, `apply()`)
- **Controllers:** WorkflowController, ApprovalInboxController, MyRequestController, ApprovalRequestController, DelegationController; **(Guardian)** ApprovalController; **(Student)** RequestController
- **Actions:** SubmitApprovalRequest, DecideApprovalStep, CancelApprovalRequest, DelegateApprovals, BulkApprove
- **Services:** ApprovalTypeRegistry, ApproverResolverRegistry, WorkflowResolver
- **Policies:** ApprovalRequestPolicy, ApprovalWorkflowPolicy
- **Events:** ApprovalRequestSubmitted, ApprovalStepDecided, ApprovalRequestApproved, ApprovalRequestRejected, ApprovalRequestReturned
- **Jobs:** EscalateOverdueApprovals (R2)

### Notification
- **Models:** NotificationTemplate, NotificationProvider, NotificationRule, NotificationDispatch, NotificationDeliveryAttempt, NotificationPreference
- **Contracts:** SmsGateway, WhatsAppGateway
- **Controllers:** TemplateController, RuleController, ProviderController, DeliveryLogController, BroadcastController, NotificationCenterController (all portals), PreferenceController; **(Webhooks)** SmsDeliveryReportController
- **Actions:** SendBroadcast, TestProvider, MarkNotificationsRead
- **Services:** NotificationEventRegistry, RecipientResolverRegistry, Dispatcher (rules, dedupe, quiet hours, quotas), TemplateRenderer, SmsSegmentCounter, TemplatedNotification
- **Listens:** every registered domain event through `DispatchNotificationsForEvent` (after commit)
- **Jobs:** SendDispatch, ReleaseQuietHoursMessages, ProcessDeliveryReport

### ImportExport
- **Models:** ImportJob, ImportJobRow, ExportJob
- **Contracts:** ImportDefinition, ExportDefinition
- **Controllers:** ImportController (upload, map, preview, commit), ImportTemplateController, ImportHistoryController, ExportController
- **Services:** ImportRegistry, ExportRegistry, TemplateBuilder, ColumnMapper, DuplicateDetector
- **Jobs:** ValidateImportFile, CommitImportChunk, FinalizeImport, BuildExport, PurgeExpiredExports
- **Events:** ImportCompleted, ExportReady

### Document
- **Models:** DocumentTemplate, IssuedDocument
- **Contracts:** PdfRenderer
- **Controllers:** DocumentTemplateController, IssuedDocumentController; **(Public, R2)** VerificationController
- **Actions:** IssueDocument, ReissueDocument, RevokeDocument, RenderDocument
- **Services:** DocumentTypeRegistry, DocumentComposer, GotenbergPdfRenderer, PdfMerger, VerificationCodeGenerator
- **Events:** DocumentIssued
- **Jobs:** RenderDocumentBatch, MergeBatchPdf

### Academic
- **Models:** AcademicSession, EducationLevel, StudyGroup, StudyGroupSubject, Subject, AcademicClass, Section, ClassSubject, TeachingAssignment
- **Controllers:** AcademicSessionController, SessionClosingController, SessionInitializationController, EducationLevelController, StudyGroupController, SubjectController, AcademicClassController, SectionController, ClassSubjectController, TeachingAssignmentController, WorkloadController
- **Actions:** ActivateSession, CloseSession, InitializeSessionFromPrevious, AssignTeacher, EndTeachingAssignment, CreateSection
- **Services:** WorkloadCalculator, SessionCloseCheckRegistry; contracts `AcademicStructureQuery`
- **Policies:** one per model
- **Events:** SessionActivated, SessionClosed, TeachingAssignmentChanged, SectionCreated
- **Listens:** EmployeeSeparated (end assignments)
- **Jobs:** CopySessionStructure

### Admission
- **Models:** AdmissionInquiry, AdmissionFormTemplate, AdmissionApplication, AdmissionAssessment
- **Controllers:** AdmissionFormTemplateController, AdmissionApplicationController, ApplicationReviewController, ApplicationConversionController, SeatAvailabilityController, GrRegisterController, InquiryController (R2), AssessmentController (R2); **(Public)** OnlineApplicationController
- **Actions:** SubmitApplication, ApproveApplication, RejectApplication, ConvertApplicationToStudent, AllocateSection
- **Services:** ApplicantDuplicateDetector, SiblingDetector, SectionAllocator
- **Policies:** AdmissionApplicationPolicy, AdmissionFormTemplatePolicy
- **Events:** ApplicationSubmitted, ApplicationApproved, ApplicationRejected, StudentAdmitted
- **Listens:** PaymentRecorded (admission voucher paid)

### Student
- **Models:** Student, StudentEnrollment, StudentSubject, StudentTimelineEvent, StudentTransfer, StudentWithdrawal, StudentCategory, Sponsor, StudentHealthProfile
- **Contracts:** StudentEnroller, EnrollmentQuery, TimelineRecorder, PromotionEligibilityProvider, ClearanceCheckRegistry, StudentProfileTabRegistry
- **Controllers:** StudentController, StudentProfileController (tabs), StudentDocumentController, StudentHealthController, EnrollmentController, PromotionController, TransferController, WithdrawalController, StruckOffController, StudentCategoryController, SponsorController, StudentCertificateController, BulkStudentActionController; **(Student)** DashboardController, ProfileController
- **Actions:** EnrollStudent, PromoteStudents, RetainStudent, ChangeSection, TransferBranch, WithdrawStudent, StrikeOffStudent, ReadmitStudent, GraduateStudents, UpdateProtectedProfile, SelectElectives
- **Approval handlers:** StudentProfileCorrectionHandler, StudentModificationHandler, BranchTransferHandler, WithdrawalHandler
- **Policies:** StudentPolicy, StudentEnrollmentPolicy, StudentHealthProfilePolicy
- **Events:** StudentEnrolled, StudentPromoted, StudentRetained, StudentSectionChanged, StudentTransferred, StudentWithdrawn, StudentStruckOff, StudentReadmitted, StudentGraduated, StudentProfileUpdated
- **Jobs:** PromoteStudentsChunk

### Guardian
- **Models:** Guardian, GuardianStudent, Family, PickupAuthorization
- **Controllers:** GuardianController, GuardianStudentLinkController, FamilyController, PickupAuthorizationController, GuardianVerificationController; **(Guardian)** DashboardController, ChildSwitchController, ChildProfileController
- **Actions:** CreateOrMatchGuardian, LinkGuardianToStudent, UnlinkGuardian, MergeFamilies, SplitFamily, FlagCustodyRestriction, VerifyGuardian
- **Services:** GuardianChildAccess, FamilyResolver; contracts `GuardianLookup`
- **Policies:** GuardianPolicy, FamilyPolicy
- **Events:** GuardianLinked, GuardianUnlinked, CustodyRestrictionFlagged, FamiliesMerged

### HR
- **Models:** Employee, Designation, EmployeeBranchAssignment, EmployeeQualification, EmployeeExperience, EmployeeContract, PerformanceReview, TrainingRecord, EmployeeDisciplinaryAction, EmployeeSeparation
- **Controllers:** EmployeeController, EmployeeProfileController, DesignationController, EmployeeBranchAssignmentController, EmployeeDocumentController, StaffDirectoryController, MyProfileController; R2: EmployeeContractController, PerformanceReviewController, TrainingRecordController, EmployeeSeparationController
- **Actions:** HireEmployee, AssignEmployeeToBranch, ChangeEmployeeStatus, CreateEmployeeUser, SeparateEmployee (R2)
- **Approval handlers:** EmployeeProfileCorrectionHandler
- **Services:** EmployeeProfileTabRegistry; contracts `EmployeeLookup`
- **Policies:** EmployeePolicy, DesignationPolicy
- **Events:** EmployeeHired, EmployeeBranchAssigned, EmployeeStatusChanged, EmployeeSeparated
- **Jobs:** SendDocumentExpiryReminders

### Attendance
- **Models:** AttendancePolicy, AttendanceDevice, DeviceEnrollment, AttendancePunch, StudentAttendance, StudentPeriodAttendance, StaffAttendance, AttendanceCorrection, QrAttendanceSession, DeviceSyncLog
- **Contracts:** AttendanceDeviceAdapter, AttendanceLeaveWriter, AttendanceSummaryQuery
- **Controllers:** StudentAttendanceMarkingController (grid and JSON save), PeriodAttendanceController, StaffCheckInController, QrKioskController, QrScanController, AttendancePolicyController, AttendanceDeviceController, DeviceEnrollmentController, PunchImportController, AttendanceCorrectionController, AdministrativeAttendanceController, AttendanceReportController, AttendanceMonthLockController; **(Webhooks)** Adms\IclockController, SignedPunchWebhookController; **(Student, Guardian)** AttendanceController
- **Actions:** MarkSectionAttendance, FinalizeAttendanceDay, CheckInStaff, CheckOutStaff, RecordAdministrativeAttendance, ApplyClosure, IngestPunches
- **Approval handlers:** AttendanceCorrectionHandler, MissingPunchHandler
- **Services:** PunchIngestor, DailyAttendanceResolver, AttendancePolicyResolver, LateCalculator, GeofenceVerifier, QrTokenService, SuspiciousActivityDetector
- **Policies:** StudentAttendancePolicy, StaffAttendancePolicy, AttendanceDevicePolicy, AttendancePolicyPolicy
- **Events:** SectionAttendanceMarked, AttendanceFinalized, StudentMarkedAbsent, StudentMarkedLate, AttendanceCorrected, DeviceWentOffline, SuspiciousAttendanceDetected
- **Listens:** ClosureDeclared, EmployeeSeparated (disable device enrollment)
- **Jobs:** ProcessWebhookReceipt, ResolveDailyAttendance, FinalizeAttendanceWindow, AutoMarkAbsent, SendNotMarkedAlerts, CheckDeviceHealth, ImportPunchFile

### Leave
- **Models:** LeaveType, LeavePolicy, LeaveBalance, LeaveApplication
- **Controllers:** LeaveTypeController, LeavePolicyController, LeaveBalanceController, LeaveApplicationController, LeaveCalendarController, MyLeaveController; **(Guardian)** ChildLeaveController
- **Actions:** ApplyLeave, CancelLeave, AdjustLeaveBalance, AccrueLeaveBalances
- **Approval handlers:** LeaveApplicationHandler (writes through `AttendanceLeaveWriter`)
- **Services:** LeaveDayCalculator (sandwich rule, half days, holidays), LeaveBalanceService
- **Policies:** LeaveApplicationPolicy, LeaveTypePolicy
- **Events:** LeaveApplied, LeaveApproved, LeaveRejected, LeaveCancelled
- **Jobs:** AccrueLeaveBalancesForOrganization

### Fee
- **Models:** FeeHead, FeeStructure, FeeStructureItem, FeeDiscountRule, StudentFeeConcession, Scholarship, InstallmentPlan, FeeVoucher, FeeVoucherItem, FeePayment, FeePaymentAllocation, FeeRefund, CashierSession, PaymentGatewayAccount, PaymentGatewayTransaction
- **Contracts:** FeeClearanceChecker, AdHocVoucherIssuer, FeeBalanceQuery, VoucherLineContributor, PaymentGateway, BillPaymentProvider
- **Controllers:** FeeHeadController, FeeStructureController, FeeDiscountRuleController, ConcessionController, ScholarshipController, VoucherController, VoucherGenerationController, VoucherPrintController, PaymentController (with JSON voucher lookup), ChequeController, BankChallanEntryController, CashierSessionController, RefundController, FeeLedgerController, DefaulterController, FeeReportController; **(Guardian)** VoucherController, PaymentHistoryController, OnlinePaymentController (R2); **(Webhooks, R2)** JazzCashCallbackController, EasypaisaCallbackController, OneBillController
- **Actions:** GenerateVouchers, GenerateStudentVoucher, CancelVoucher, IssueAdHocVoucher, RecordPayment, MarkChequeCleared, BounceCheque, OpenCashierSession, CloseCashierSession, IssueRefund, ApplyAdvanceCredit, ApplyLateFees
- **Approval handlers:** FeeConcessionHandler, ScholarshipHandler, PaymentReversalHandler, FeeRefundHandler, CashVarianceHandler
- **Services:** VoucherCalculator, DiscountRuleEngine, LateFeeCalculator, AllocationService, FeePostingRules (builds journal drafts for Accounting)
- **Policies:** FeeStructurePolicy, FeeVoucherPolicy, FeePaymentPolicy, CashierSessionPolicy, ConcessionPolicy
- **Events:** VoucherGenerated, VouchersBatchGenerated, PaymentRecorded, PaymentReversed, ChequeBounced, RefundIssued, FeeOverdue, StruckOffCandidatesIdentified
- **Jobs:** GenerateVoucherChunk, RenderVoucherPdfBatch, MarkOverdueVouchers, ApplyLateFeesForOrganization, IdentifyStruckOffCandidates, SendFeeReminders, ImportBankChallanFile, ReconcileGatewayTransactions (R2)

### Accounting
- **Models:** FiscalYear, Account, JournalEntry, JournalLine, BankAccount, BankStatementLine, BankReconciliation, Vendor, ExpenseVoucher, PurchaseBill
- **Contracts:** PostingService, AccountResolver
- **Controllers:** FiscalYearController, AccountController, JournalEntryController, CashbookController, BankBookController, LedgerController, TrialBalanceController, ReceivablesAgingController, OpeningBalanceController, BankAccountController; R2: ExpenseVoucherController, VendorController, PurchaseBillController, BankReconciliationController, FinancialStatementController
- **Actions:** OpenFiscalYear, CloseFiscalYear, CreateManualJournal, ReverseJournal, EnterOpeningBalances; R2: RecordExpense, RecordPurchaseBill, ImportBankStatement, ReconcileBankStatement
- **Approval handlers:** JournalEntryHandler; R2: ExpenseHandler, PurchaseHandler
- **Services:** DoubleEntryPostingService (balance, period and numbering checks), LedgerQuery, TrialBalanceQuery, AgingQuery
- **Policies:** AccountPolicy, JournalEntryPolicy, FiscalYearPolicy
- **Events:** JournalPosted, JournalReversed, FiscalYearClosed

### Examination
- **Models:** ExamTerm, Exam, ExamPaper, ExamPaperComponent, MarkEntry, MarkEntryRevision, MarksSubmission
- **Controllers:** ExamTermController, ExamController, ExamPaperController, DateSheetController, AdmitCardController, MarksEntryController (grid and JSON autosave), MarksSubmissionController, MarksReviewController, MarksCompletionController, MarksImportController; **(Student, Guardian)** DateSheetController
- **Actions:** CreateExam, PublishDateSheet, SaveMarks, SubmitMarks, ApproveMarksSubmission, ReturnMarksSubmission, IssueAdmitCards
- **Approval handlers:** MarksRevisionHandler
- **Services:** DateSheetClashDetector, MarksValidator
- **Policies:** ExamPolicy, MarkEntryPolicy, MarksSubmissionPolicy
- **Events:** DateSheetPublished, MarksSubmitted, MarksApproved, MarksReturned, MarksRevised
- **Jobs:** GenerateAdmitCards

### Result
- **Models:** GradingScheme, GradingSchemeBand, ResultRule, ResultBatch, StudentResult, StudentSubjectResult, ResultPublication
- **Controllers:** GradingSchemeController, ResultRuleController, ResultBatchController, ResultPublicationController, WithheldResultController, ReportCardController, ResultAnalyticsController; **(Student, Guardian)** ResultController
- **Actions:** ProcessResultBatch, PublishResults, WithholdResult, UnpublishResult
- **Approval handlers:** ResultPublicationHandler; R2: ResultRecheckHandler
- **Services:** ResultCalculator (pure functions, heavily unit tested), GradeResolver, RankingService, CumulativeResultService, PromotionEligibility (implements Student contract)
- **Policies:** GradingSchemePolicy, ResultBatchPolicy, StudentResultPolicy
- **Events:** ResultProcessed, ResultPublished, ResultWithheld, ResultUnpublished
- **Listens:** MarksRevised (flag batch stale)
- **Jobs:** ProcessResultChunk, RenderReportCardsBatch

### Timetable
- **Models:** BellSchedule, BellSchedulePeriod, Timetable, TimetableEntry, TimetableSubstitution
- **Controllers:** BellScheduleController, TimetableController, TimetableEntryController (JSON grid), TimetableViewController, SubstitutionController (R2); **(Student, Guardian)** MyTimetableController
- **Actions:** SaveTimetableEntry, ActivateTimetableVersion, CopyTimetables, AssignSubstitution (R2)
- **Services:** TimetableConflictDetector, TeachingLoadValidator, FreeTeacherFinder (R2)
- **Policies:** TimetablePolicy, BellSchedulePolicy
- **Events:** TimetableActivated, SubstitutionAssigned
- **Listens:** ClosureDeclared, TeachingAssignmentChanged

### Communication
- **Models:** Announcement, AnnouncementAudience, AnnouncementRead, DiaryEntry, MessageThread, Message, PtmEvent, PtmSlot, PtmBooking
- **Controllers:** AnnouncementController, NoticeBoardController (all portals), DiaryController, DiaryReviewController; **(Guardian)** DiaryController; R2: MessageThreadController, PtmController
- **Actions:** PublishAnnouncement, AcknowledgeAnnouncement, SaveDiaryEntry, PublishDiary; R2: SendMessage, BookPtmSlot
- **Services:** AudienceResolver, AnnouncementVisibilityScope
- **Policies:** AnnouncementPolicy, DiaryEntryPolicy, MessageThreadPolicy
- **Events:** AnnouncementPublished, DiaryPublished, MessageSent, PtmScheduled

### Learning (R2)
- **Models:** Course, CourseChapter, CourseTopic, Lecture, LectureProgress, DiscussionThread, DiscussionPost, ContentReport
- **Controllers:** CourseController, CourseChapterController, LectureController, DiscussionController, ModerationController; **(Student)** LectureController; **(Guardian)** LearningController
- **Actions:** PublishLecture, CopyLecture, RecordLectureProgress, PostDiscussion, ResolveQuestion, ReportContent
- **Services:** VideoEmbedSanitizer, RichTextSanitizer
- **Events:** LecturePublished, QuestionAsked, QuestionAnswered
- **Listens:** SessionActivated, TeachingAssignmentChanged
- **Jobs:** EnsureCoursesForSession

### Homework (R2)
- **Models:** HomeworkAssignment, HomeworkTarget, HomeworkSubmission, HomeworkSubmissionReview
- **Controllers:** HomeworkController, SubmissionReviewController; **(Student, Guardian)** HomeworkController
- **Actions:** AssignHomework, SubmitHomework, ReviewSubmission, ReturnSubmission
- **Events:** HomeworkAssigned, HomeworkSubmitted, HomeworkReviewed
- **Jobs:** SendHomeworkDeadlineReminders, MarkMissingSubmissions

### Assessment (R2)
- **Models:** QuestionBank, Question, QuestionOption, QuestionTag, Quiz, QuizQuestion, QuizTarget, QuizAttempt, QuizAttemptAnswer
- **Controllers:** QuestionBankController, QuestionController, QuizController, QuizGradingController, QuizAnalyticsController; **(Student)** QuizAttemptController (start, JSON answer save, submit)
- **Actions:** StartAttempt, SaveAnswer, SubmitAttempt, GradeSubjectiveAnswer, PublishQuizResults, TransferScoresToExam
- **Services:** QuizAssembler, AutoGrader, AttemptClock
- **Events:** QuizPublished, QuizAttemptSubmitted, QuizResultsPublished
- **Jobs:** AutoSubmitExpiredAttempts

### Payroll (R2)
- **Models:** SalaryComponent, SalaryStructure, SalaryStructureComponent, EmployeeSalary, PayrollRun, Payslip, PayslipLine, SalaryAdvance, EmployeeLoan, LoanInstallment, SalaryPayment
- **Controllers:** SalaryComponentController, SalaryStructureController, EmployeeSalaryController, PayrollRunController, PayslipController, SalaryAdvanceController, EmployeeLoanController, SalaryPaymentController, PayrollReportController, MyPayslipController
- **Actions:** CreatePayrollRun, ComputePayrollRun, AdjustPayslip, FinalizePayrollRun, RecordSalaryPayments
- **Approval handlers:** PayrollRunHandler
- **Services:** PayrollCalculator, AttendanceInputCollector, PayrollPostingRules
- **Events:** PayrollRunFinalized, SalaryPaid
- **Jobs:** ComputePayrollChunk, RenderPayslipsBatch

### Library (R3)
- **Models:** LibraryBook, LibraryBookCopy, LibraryAuthor, LibraryPublisher, LibraryCategory, LibraryMember, LibraryLoan, LibraryFine
- **Controllers:** LibraryBookController, LibraryBookCopyController, CirculationController (JSON scan), LibraryMemberController, LibraryFineController, LibraryReportController
- **Actions:** IssueBook, ReturnBook, RenewLoan, ReserveBook, ChargeFine
- **Contributions:** clearance check, voucher line contributor (fines)
- **Events:** BookIssued, BookReturned, FineCharged

### Transport (R2, R3)
- **Models:** TransportRoute, TransportStop, RouteStop, TransportAllocation, Vehicle, VehicleDocument, DriverProfile, VehicleMaintenanceLog, FuelLog, TransportAttendance, GpsDevice
- **Controllers:** TransportRouteController, TransportStopController, TransportAllocationController, RouteSheetController; R3: VehicleController, DriverProfileController, MaintenanceLogController, FuelLogController, TransportAttendanceController
- **Actions:** AllocateTransport, EndTransportAllocation, RecordBoarding (R3)
- **Contributions:** voucher line contributor, clearance check
- **Listens:** StudentWithdrawn (end allocation)
- **Events:** TransportAllocated, TransportAllocationEnded

### Inventory (R3)
- **Models:** InventoryCategory, InventoryItem, StockLocation, StockMovement, Asset, AssetAssignment, MaintenanceRequest
- **Controllers:** InventoryItemController, StockLocationController, StockMovementController, AssetController, AssetAssignmentController, MaintenanceRequestController, InventoryReportController
- **Actions:** ReceiveStock, IssueStock, TransferStock, AdjustStock, AssignAsset, ReturnAsset
- **Approval handlers:** StockAdjustmentHandler
- **Events:** StockAdjusted, AssetAssigned

### Administration
- **Models:** CalendarEvent, GatePass, VisitorLog, Complaint
- **Contracts:** CalendarQuery (`isWorkingDay(branch, date)`, used by Attendance, Leave, Timetable)
- **Controllers:** CalendarController, ClosureController, EventController, IdCardController; R2: GatePassController; R3: VisitorLogController, ComplaintController
- **Actions:** CreateCalendarEvent, DeclareClosure, PrintIdCards, IssueGatePass (R2)
- **Events:** ClosureDeclared, CalendarEventCreated

### Discipline (R2, R3)
- **Models:** DisciplineIncident, DisciplineIncidentStudent, DisciplineAction, StudentRemark, StudentAchievement, CounselingSession
- **Controllers:** DisciplineIncidentController, StudentRemarkController, StudentAchievementController, CounselingSessionController (R3)
- **Actions:** RecordIncident, TakeDisciplineAction, AddRemark, RecordAchievement
- **Policies:** CounselingSessionPolicy (separate permission, never guardian or teacher)
- **Events:** DisciplineIncidentRecorded, DisciplineActionTaken

### Reporting
- **Models:** DailyMetricSnapshot, SavedReportFilter, ScheduledReport
- **Controllers:** DashboardController (per portal), ReportCatalogController, ReportRunController (generic by report key), SavedReportFilterController, ScheduledReportController (R2)
- **Services:** ReportRunner, SnapshotComputer (uses Core `MetricRegistry`), DashboardComposer
- **Jobs:** ComputeTodaySnapshots, ComputeNightlySnapshots, SendScheduledReport (R2)

---

## 3.25 Coding Standards Summary (full text in the Phase 0 Master Context Prompt, D14)

- `declare(strict_types=1);` in every PHP file. Actions, services and DTOs are `final`. DTOs are `readonly` classes (PHP 8.2 feature, allowed).
- Allowed PHP features: everything in 8.2 (readonly classes, DNF types, enums, `true`/`false`/`null` standalone types). Forbidden until UG-1: typed class constants, `#[\Override]`, `json_validate()`, property hooks, asymmetric visibility, implicit nullable parameters.
- Backed string enums for every status and type; values are snake_case.
- Methods named after the domain (`recordPayment`, not `handleData`). No abbreviations except IDs from D2.
- Every Action: validate state, lock, write, audit, return a result DTO; events fire after commit.
- No N+1 queries: eager loading required on lists; Laravel's `preventLazyLoading()` enabled outside production.
- Every list paginates on the server. No `->get()` on unbounded tenant tables in controllers.

---

## Assumptions

| ID | Assumption |
|---|---|
| A-28 | Production can run Docker for Gotenberg (or Gotenberg is installed as a system service on a worker host). |
| A-29 | Redis is available in staging and production. Local development may use the database queue and cache drivers. |
| A-30 | One application server and one worker server are enough for R1 launch tenants; horizontal scaling is added by adding stateless app servers behind a load balancer (sessions in database, files on S3). |
| A-31 | SMS aggregators used by tenants expose an HTTP API that fits a configurable request template; aggregators that do not get a dedicated adapter. |
| A-32 | First-party Laravel packages listed as "verify at install" will publish versions compatible with both Laravel 12 and 13 on PHP 8.2; if any does not, the phase prompt stops and reports instead of downgrading or forcing. |

## Gaps Found and Added

| ID | Gap | Decision | Release |
|---|---|---|---|
| GAP-64 | ADMS devices cannot include a tenant identifier in their URL | DEC-29: tenant by device serial on a dedicated host | R1 |
| GAP-65 | No storage for idempotency keys or raw webhooks | `idempotency_keys`, `webhook_receipts` in Core (DEC-42) | R1 |
| GAP-66 | No user-visible tracking for bulk operations | `bulk_operations` linked to job batches | R1 |
| GAP-67 | XSS defense depended only on escaping | Strict CSP with Alpine CSP build (DEC-30) | R1 |
| GAP-68 | Queued jobs and scheduled commands could run without tenant context | `TenantAware` job middleware; commands fan out per organization | R1 |
| GAP-69 | Tests could pass on SQLite and fail on MySQL | DEC-39 | R1 |
| GAP-70 | Module boundaries were only a convention | Architecture tests (3.4, 3.20) | R1 |
| GAP-71 | Circular dependencies in D2 map | Corrections in 3.3 | R1 |
| GAP-72 | Phone photos carry GPS EXIF data | EXIF stripped on upload (3.13) | R1 |
| GAP-73 | No queue fairness between tenants at month end | Per-organization overlap keys on bulk jobs (3.11) | R1 |
| GAP-74 | Package compatibility not checked mechanically | `composer prohibits` checks in every phase prompt (3.21) | R1 |

---

## What Deliverable 4 Covers

Multi-tenant and multi-branch architecture in depth: why a shared database with `organization_id` and `branch_id` is the right start, the full scoping model (organization, institution, branch, shift, visibility levels), shared employees and inter-branch assignments, branch transfers, consolidated reporting, subdomain and custom domain handling, how isolation is enforced at query, policy, job, file, cache, export and API levels, the isolation test plan, and the documented path to move one large tenant to a dedicated database.
