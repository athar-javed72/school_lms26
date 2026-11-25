# Deliverable 14: Sequential Antigravity Implementation Prompts
# Part 9 of 9: Release 3, PH-19 to PH-24

**Version:** 1.0 (draft for approval) | **Date:** 24 Nov 2026
**Builds on:** D14 Parts 1 to 8, D01 GAP-01, GAP-41, GAP-42, GAP-44, D02 (every R3 requirement), D03 3.15 and 3.23, D04 4.20, D05 5.7.21, 5.7.22, 5.9, D07 Part 1 7.14 (documents), Part 3 7.31 and 7.33, Part 4 7.41 to 7.44, 7.50, D10 Part 1, Part 2 SCR-DOC-003 and Part 3 (every R3 screen), D11 Part 2 11.30, D12 Part 1 12.3 and Part 2 12.22 to 12.31, D13 PH-19 to PH-24, DEC-224
**How to run:** start a new Antigravity conversation for each step and paste the Phase 0 Master Context Prompt before the step prompt. R1 and R2 are live in production, so every migration is expand-only (D12 12.29). Steps P23.0 and P24.0 write and get approval for the D7 schema addenda (DEC-224) before any feature code in their phase.

---

## 14.31 New Decisions

| ID | Decision |
|---|---|
| DEC-293 | **Table alias corrections.** Four aliases collide across D7 and D14. R1 tables keep their aliases, and the later tables change: `asset_assignments` becomes `aag` (was `asa`, which `approval_step_assignees` owns), `library_book_authors` becomes `lbx` (was `lba`, which `leave_balance_adjustments` owns), `counseling_sessions` becomes `csl` (was `cns`, which `consent_records` owns under DEC-257), and `salary_advances` becomes `sva` (was `sad`, which `student_admissions` owns). Index and key names follow the new aliases, for example `uq_aag_active`, `uq_lbx`, `ix_csl`, `uq_sva_no`. D14 Part 8 P13.3 is updated. A new arch test, `TableAliasUniquenessTest`, reads every migration's table comment alias and fails on duplicates. |
| DEC-294 | **UG-5 (Laravel 15) slot is P21.0.** The step runs at the start of PH-21 if Laravel 15 meets the DEC-200 rule by then: at least two point releases, and every composer dependency supports it. Otherwise the step keeps the ID P21.0 and runs at the first later phase start where the rule is met. It always runs before Laravel 14 security fixes end, which falls after the R3 date, so RC-7 still holds at R3 GA. D15 records where it actually ran. |
| DEC-295 | **P22.0 is removed.** `maintenance_requests` already exists in D7 7.43, so PH-22 needs no schema addendum. Complaint categories and complaint SLA hours per priority are organization settings, not tables. The D13 M14c row loses "maintenance request table from the D7 addendum". |
| DEC-296 | **Library never handles money.** Student fines reach fee vouchers through `LibraryFineVoucherLineContributor` (source_ref_type `library_fine`). Employee fines reach payslips through `PayslipLineContributorRegistry` with the `library_fine` system component. When a student wants to pay at the desk, "Pay now" creates an ad hoc fee voucher holding the fine, and the cashier collects it through the normal receipt flow. Waivers: holders of `library.fine.manage` waive directly up to the branch setting `library.waiver_threshold`. A librarian's waiver above it raises approval type `library_fine_waiver` (1. principal_of_branch), decided with the new permission `library.fine.approve` (PR). A principal's own waiver needs no approval. |
| DEC-297 | **Library stock verification (FR-LIB-006).** `library_book_copies` gains expand-only columns `last_verified_at` DATETIME? and `last_verified_by` FK->users?. A verification run is a start time plus a scope (branch, or category, or rack prefix). Scanning a copy stamps it. The report lists copies in scope not stamped since the run started. Copies become `lost` only through the librarian's explicit `MarkCopiesLost` action with a reason, never automatically. |
| DEC-298 | **Transport assignment validity (R3).** A route assignment is refused when the driver's license has expired, or when the vehicle's fitness, insurance or route permit document has expired or is missing. A principal can override with a reason. The override is audited and raises an attention item every day until it is fixed. Seat capacity from R3 is the lower of `transport_routes.capacity` and the active vehicle's `seating_capacity` (DEC-282). Allocation locks the route row, then reads the active assignment. |
| DEC-299 | **GPS and maps.** Each GPS provider gets an adapter on the `GpsProvider` contract, using push to `hooks.{platform_domain}/webhooks/gps/{provider}/{organization_uuid}` or polling of the provider's official API. Positions are stored at most once every 15 seconds per device in `gps_positions` (partitioned daily, 30 days, A-68, DEC-123). Map tiles come from the tile URL in the platform setting `maps.tile_url`, under the platform's agreement with a tile provider, with OpenStreetMap attribution shown. Production never uses the public OpenStreetMap tile servers. The CSP `img-src` allows only that tile host. Guardians see the location of their own child's vehicle only during that route's trip windows (pickup time minus 30 minutes to drop time plus 30 minutes). |
| DEC-300 | **Inventory and assets do not post to the ledger in R3.** A-65 extends to assets: the purchase bill already posted the cost to the line's account, and asset depreciation, disposal gain or loss and write-offs are not journaled automatically. A depreciation schedule report (straight line from useful life and salvage value) gives accountants the figures for manual journals (A-140). Asset disposal needs approval type `asset_disposal` (1. principal_of_branch, 2. owner above `inventory.disposal_owner_threshold`), decided with the new permissions `inventory.disposal.approve` and `.reject` (PR, OW). Disposal is requested with `inventory.asset.update`. |
| DEC-301 | **Inter-branch stock transfers (FR-INV-003)** write `transfer_out` in the source branch and `transfer_in` in the target branch in one transaction, sharing `transfer_group_ulid`. The user needs `inventory.stock.manage` in both branches. Stock is received at once; R3 has no in-transit state. The target branch's custodian is notified. |
| DEC-302 | **Counseling access.** Notes in `standard` sessions can be read by the counselor who wrote them and by holders of `discipline.counseling.view` (2FA). Notes in `restricted` sessions can be read only by their author. Others with the view permission see that a restricted session exists, with its date and type, but not its notes. Every read of notes writes an audit "viewed" event with no content. Notes never appear in search, exports, imports, timelines, notifications or the API (DEC-122). A counselor is a custom role (A-41) holding `discipline.counseling.create`, `.view` and `.update`. |
| DEC-303 | **Complaints and visitors.** Complaint SLA hours per priority live in the setting `administration.complaint_sla_hours` (defaults: urgent 24, high 48, normal 72, low 120), and categories in `administration.complaint_categories`. A complaint past its SLA raises an attention item for the assignee and the principal, and notifies the principal once. The complainant can rate satisfaction (1 to 5) and can reopen within 7 days of resolution. Staff complaints about a colleague are visible only to BM and PR. Visitor photos are purged after the setting `administration.visitor_photo_retention_days` (default 90). The visitor log stores CNIC last4 only. |
| DEC-304 | **D7 Addendum R3-A (written and approved in P23.0)** adds: `analytics_snapshots` [B] alias `ans`; `document_template_versions` [T-] alias `dtv`; `timetable_generation_runs` [B] alias `tgr`; `offline_sync_batches` [B] alias `ofb`; `attendance_sync_conflicts` [B] alias `acf`. Enum additions: `journal_entries.entry_type` `inter_branch_transfer`, `accounts.system_role` `inter_branch_clearing`, and `student_attendances.source` and revision source `offline`. Expand-only column: `document_templates.current_version_id` FK->document_template_versions?. Column lists are in 14.36 P23.0. |
| DEC-305 | **R3 screens and permissions for PH-23** (D10 never listed them, so the addendum adds them): SCR-RPT-010 Result analytics, SCR-RPT-011 Trends and branch analytics, SCR-ACC-022 Inter-branch transfers, SCR-TTB-012 Timetable generator, SCR-ATT-026 Offline sync conflicts. Permissions: `reporting.analytics.view` (OW, OA, PR, VP), `accounting.transfer.view` and `.create` (ACT, OW), `timetable.generation.run` (AC), `attendance.sync_conflict.view` and `.resolve` (CT for own sections, AC, PR). |
| DEC-306 | **Offline attendance (GAP-41).** Covers only daily section attendance, for the current date, on SCR-ATT-002. When the teacher opens the sheet online, the page stores in IndexedDB the roster (roll number, display name, enrollment id; no photos, guardians or contacts) and the sheet version. Marks made offline queue locally, each with a client ULID, the base sheet version and the device time. On reconnect the queue syncs as one batch through `SyncOfflineAttendance`, which calls `MarkStudentAttendance` with source `offline` for each mark. A mark is applied when the student's row has not changed since the base version. Otherwise the server keeps the newer value and writes an `attendance_sync_conflicts` row for the teacher or coordinator to resolve. Marks for a finalized sheet, a locked period, or a date other than the device's sync day are rejected and listed. Local data is cleared after a successful sync, on logout, and at midnight. Nothing personal beyond the roster is stored offline. |
| DEC-307 | **Template designer.** Layouts are data, never markup. A layout is a JSON block tree (text with merge fields, field, image, table, QR verification, barcode, line, spacer, columns, page settings) saved as an immutable `document_template_versions` row. One renderer component turns a layout into HTML for Gotenberg. Users never enter HTML, CSS or scripts. Merge fields come only from `DocumentFieldRegistry` for the document type, which never exposes Restricted fields. Issued documents keep their `template_version` and stored PDF, so publishing a new version never changes documents already issued. |
| DEC-308 | **Assisted timetable generation produces drafts only.** `GenerateTimetable` runs as a queued job with a time limit (setting `timetable.generation_seconds`, default 120) and a stored random seed, so the same inputs and seed give the same result. Hard constraints: no teacher, room, section or teaching group clash, including the teacher's periods in other branches' active timetables; exact periods per week from teaching assignments; teacher maximum periods per week and per day; lab subjects in lab rooms when the class subject requires it; teacher unavailability entered in the run. Soft constraints, scored: spread each subject across days, avoid a subject twice in one day unless it is a double period, keep teacher gaps small, put subjects flagged "heavy" early in the day. The result is a new `draft` timetable version with the unplaced lessons and their reasons. Activation stays with the existing conflict-checked SCR-TTB-004. |
| DEC-309 | **Analytics come from nightly snapshots.** `reporting:analytics-snapshots` runs at 02:30 PKT on the read replica and writes `analytics_snapshots` per branch and scope (exam term results, attendance month, fee collection month, enrollment). Every chart shows its `computed_at` time. Drill-downs open the live report runner. Snapshots are read-only, rebuilt when the source changes (for example a republished result), and never feed back into operational data. |
| DEC-310 | **D7 Addendum R3-B (written and approved in P24.0)** adds: `subscriptions` [P] alias `sbs`; `subscription_invoices` [P] alias `sbi`; `subscription_invoice_lines` [P] alias `sil`; `subscription_payments` [P] alias `sbp`; `partner_api_keys` [T] alias `pak`; `push_device_registrations` [T] alias `pdr`; `bank_api_connections` [B] alias `bac`. [P] means a platform-owned table that carries `organization_id` but is managed only by platform billing actions. Expand-only columns: `organizations.billing_mode` (`manual`, `self_service`); `bank_statement_lines.source` (`import`, `api`). Column lists are in 14.37 P24.0. |
| DEC-311 | **Platform billing money flow.** Subscription money is the platform's revenue. It is collected through the platform's own merchant accounts with the same verification-first adapters (DEC-166, DEC-286), and recorded in platform billing tables, never in a tenant's ledger. Invoice numbers are gapless per platform fiscal year. Tax rates are platform settings with effective dates. Dunning: an unpaid invoice becomes `overdue` after its due date plus the grace days in setting `billing.grace_days`; after `billing.suspend_after_days` the organization moves to `suspended` (read-only, D4). Data is never deleted for non-payment. Upgrades are prorated by day. A downgrade is refused while usage exceeds the target plan's limits. Organizations with `billing_mode = manual` keep the R1 manual process. |
| DEC-312 | **Partner API keys.** An organization owner creates keys bound to the organization, a branch scope, and scopes from a fixed catalog: `students:read`, `guardians:read`, `staff:read`, `attendance:read`, `fees:read`, `payments:read`, `results:read`, `timetable:read`. Partner API keys are read-only in R3. A key is shown once and stored as a SHA-256 hash with an 8-character prefix. Each key has a rate limit, optional IP allowlist, expiry and revocation. A key acts as actor type `api_key` in audit. Responses apply export masking. Restricted fields are never returned, and guardian contact numbers are masked. Partner endpoints live under `api.{platform_domain}/partner/v1` with the same envelope as API v1. |
| DEC-313 | **Push notifications (FR-NTF-015)** use Firebase Cloud Messaging HTTP v1 with the platform's service account, for future apps. Devices register through API v1 (`api.v1.push.devices.store`). Tokens are encrypted, with a hash for uniqueness. Push is a channel in the notification pipeline with the same preferences and quiet hours. A push payload carries only a title and a deep link with no personal data, and the app fetches the content after sign-in. A token is revoked on logout, on FCM's "unregistered" response, or after 60 days without use. |
| DEC-314 | **Bank APIs and Raast (FR-FEE-035)** run through a partner bank or PSP adapter on the organization's own banking agreement, built only from that partner's official API documentation. Raast collection implements the `PaymentGateway` contract, with verification before posting (DEC-166). Bank statement APIs feed `bank_statement_lines` with `source = api` through the same duplicate detection as imports. Credentials live in `bank_api_connections`, encrypted. |
| DEC-315 | **Additional languages.** A locale registry (`config/locales.php` plus the platform setting `platform.enabled_locales`) lets the platform enable a locale only when `TranslationParityTest` passes at 100% for it and a native reviewer signs off. Locales are LTR or RTL. R3 ships the registry and one additional locale chosen by the product owner at P24.0 (A-145). |
| DEC-316 | **Audit archival (FR-AUD-008).** Before a partition of `audit_logs`, `login_events`, `webhook_receipts` or `device_sync_logs` is dropped (D7 7.50), it is exported to cold object storage as encrypted NDJSON with a checksum manifest, and the export is verified by reading it back. Financial audit events are kept at least 7 years. `core:archive-restore` restores one archived month into a separate read-only schema for an investigation. |

### Corrections to earlier deliverables

| Where | Change |
|---|---|
| D07 Part 4 7.41, 7.43, 7.44 and 7.40 | Aliases `lbx`, `aag`, `csl`, `sva` (DEC-293) |
| D14 Part 8 P13.3 DATABASE | `salary_advances (sva)` (DEC-293); file re-uploaded |
| D13 13.5 PH-21 | Adds P21.0 UG-5 slot (DEC-294) |
| D13 13.5 PH-22 and 13.4 M14c | P22.0 removed; no addendum (DEC-295) |
| D13 13.5 PH-24 | P24.3 also covers audit archival (DEC-316) |
| D05 5.9 | Adds approval types `library_fine_waiver` (DEC-296) and `asset_disposal` (DEC-300) |
| D05 5.7.21 and 5.7.22 | Adds `library.fine.approve`, `inventory.disposal.approve` and `.reject` |
| D10 Part 2 and Part 3 | Adds the screens in DEC-305 and DEC-310 (P23.0, P24.0) |
| D07 Part 4 7.41 | `library_book_copies.last_verified_at`, `last_verified_by` (DEC-297) |

---

## 14.32 PH-19 Library

**Exit criteria:**
- Phase gate G-1 to G-11.
- Library fines reach vouchers only through `VoucherLineContributor`, and reach payslips only through `PayslipLineContributorRegistry`.
- MS-15 is reached.

### P19.1 Catalog, Copies, Members, Circulation, Reservations and Fines

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 4 7.41 completely and DEC-121, DEC-124; D11 Part 2 11.30 (Library flow); D10 Part 3 SCR-LIB-001 to 011, widget library.overdue and the attention item for overdue books; D05 5.7.21 (library.*) and 5.9; D02 FR-LIB-001 to 007, A-63; D14 DEC-293, DEC-296, DEC-297.

PHASE GOAL
Run each branch's library: catalog with authors, publishers and categories, copies with accession numbers and barcodes, members created automatically for students and staff, barcode circulation with renewals and reservations, fines that reach vouchers and payslips through their owning modules, stock verification, reports and clearance checks.

ALREADY BUILT
R1 and R2: students, employees, fee vouchers with VoucherLineContributorRegistry and ad hoc vouchers, PayslipLineContributorRegistry with the library_fine component (P13.2, P13.3), ClearanceCheck and EmployeeClearanceCheckRegistry, document engine, import engine, purchase bills.

TASKS
1. Migrations (Library, expand-only): library_categories, library_authors, library_publishers, library_books, library_book_authors (alias lbx, DEC-293), library_book_copies (with last_verified_at and last_verified_by, DEC-297), library_members, library_loans, library_reservations, library_fines exactly as D07 7.41.
2. Catalog actions: CreateBook and UpdateBook (ISBN-10 or ISBN-13 checksum validated and normalized; authors with roles and order; category; publisher; reference-only flag), import type library_books (template with authors as a delimited column), AddCopies (accession numbers from the numbering key library_accession per branch, barcodes Code128 from accession, acquisition source, price, optional purchase_bill_line_id), UpdateCopyCondition, WithdrawCopy (reason; history kept), PrintLabels (document type library_labels: A4 label sheets with barcode, accession and call number; barcode rendering through a maintained package after the package check).
3. Members: CreateMember and listeners StudentEnrolled, EmployeeActivated (auto-create with limits from settings library.student_max_loans, library.student_loan_days, library.staff_max_loans, library.staff_loan_days), BulkCreateMembersFromClasses, SuspendMember, ExpireMembers on withdrawal or separation (listeners StudentWithdrawn, EmployeeSeparated), PrintMemberCards (document type library_card).
4. Circulation: IssueCopy (member active, under max loans, no overdue loans, unpaid fines under library.block_fine_amount, copy available and not reference-only, uq_lln_open; due date from member loan days skipping holidays through WorkingDayCalendar), ReturnCopy (condition; overdue fine from library.fine_per_day capped by library.max_fine_per_loan; damage fine entered with reason; next reservation becomes ready), RenewLoan (up to library.max_renewals; refused when a reservation waits), MarkLoanLost (lost fine from copy price or library.default_lost_fine; copy status lost).
5. Reservations: ReserveBook (member or student portal; uq_lrs_live), on return the first waiting reservation becomes ready with expires_at after library.reservation_hold_days (default 3) and the member is notified; command library:expire-reservations hourly.
6. Fines (DEC-296): LibraryFineVoucherLineContributor (pending student fines become one voucher line each with source_ref_type library_fine; fine status billed with fee_voucher_item_id; status paid when the voucher line is settled, through a FeeVoucherLineSettled listener), payslip contributor for employee fines (status billed with payslip_line_id when the payroll run is finalized), PayFineNow (creates an ad hoc voucher; no money handled in Library), WaiveFine (direct up to library.waiver_threshold or for principals; above it SubmitApprovalRequest library_fine_waiver; handler sets waived_amount and status).
7. Approval type library_fine_waiver registered by Library (1 principal_of_branch; approve permission library.fine.approve).
8. Stock verification (DEC-297): StartVerificationRun (scope), ScanForVerification, verification report, MarkCopiesLost (reason).
9. Overdue reminders: command library:overdue-reminders daily at 08:00 (students and guardians, staff); widget library.overdue; attention item overdue library books.
10. Clearance: ClearanceCheck LIB (open loans or unpaid fines block student clearance) and employee clearance check for separations.
11. Reports (ReportDefinition): library.circulation (issued, returned, overdue), library.popular_titles, library.fines, library.verification.
12. Screens (D10 exactly): SCR-LIB-001 to SCR-LIB-011.
13. Definitions: Permissions.php for library.* per D05 5.7.21 plus library.fine.approve; Settings listed above; numbering key library_accession.
14. decisions.md rows: DEC-124, DEC-293, DEC-296, DEC-297, A-63.

DATABASE
library_categories (lct), library_authors (lau), library_publishers (lpb), library_books (lbk), library_book_authors (lbx), library_book_copies (lbc), library_members (lmb), library_loans (lln), library_reservations (lrs), library_fines (lfn) per D07 Part 4 7.41 with DEC-293 and DEC-297.

BUSINESS RULES
- One open loan per copy (uq_lln_open); one live reservation per member per title.
- Fine rates, limits and hold days are settings, never code.
- Library never inserts fee or payroll rows; contributors do it (DEC-124, DEC-296).
- Each branch runs its own library; no inter-branch loans (A-63).

UI SPEC
- Circulation desk (SCR-LIB-005): scan member card, member summary with loans and fines, scan copy, one action button that switches between Issue, Return and Renew by state; keyboard and barcode scanner friendly; audible and visual confirmation; works at 1024px.
- Student and guardian pages at 360px.

SECURITY AND PERMISSIONS
- Students and guardians see only their own or their child's loans and fines; staff see own loans.
- Waivers and lost marks are audited with reasons.

TESTS
- Issue refusals (limits, overdue, fines, reference-only, unavailable); concurrent issue of one copy (one succeeds).
- Fine calculation with cap and holidays; contributor creates exactly one voucher line per fine; payslip line for staff; settled voucher marks fine paid.
- Waiver threshold routing to approval; reservation queue and expiry; renewal blocked by reservation.
- Clearance blocks; listeners on withdrawal and separation.
- ISBN validation; verification report.
- Screen tests, audit, isolation suite, RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. On staging a book is catalogued, copies labelled, a student issues and returns late, the fine appears on the next voucher, and a staff fine appears on the next payslip.
2. CI green; staging green.

DO NOT
- Do not record any payment inside Library.
- Do not support inter-branch loans.

OUTPUT
Report per Master section 20 with the circulation walkthrough and contributor tests.
```

### P19.2 UG-3: MySQL 9.7 LTS

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D12 Part 2 DEC-201 and 12.31, D12 Part 2 12.28 (production topology) and 12.30 (backups and restore), docs/runbooks/replica-promotion.md, docs/runbooks/restore.md and docs/runbooks/upgrade-gate.md, D13 13.8 UG-3, and the official MySQL 9.7 release notes and upgrade guide read at the time of the step.

PHASE GOAL
Move every environment from MySQL 8.4 LTS to MySQL 9.7 LTS with no behaviour change, no data loss and a tested rollback, and record MS-15.

ALREADY BUILT
R1 and R2 in production on MySQL 8.4 LTS with a primary and a read replica, partitioned tables (D7 7.50), nightly integrity checks, backups and restore drills.

TASKS
1. Compatibility review against the 9.7 release notes and the MySQL upgrade checker utility: removed or changed features, reserved words against every table and column name, default changes, authentication plugins (caching_sha2_password only), partitioning, generated columns, FULLTEXT, JSON functions and collation behaviour used by the codebase. Record findings in docs/decisions/ug-3-review.md.
2. CI: add MySQL 9.7 to the matrix; run the full suite, migrations from empty, isolation suite, integrity checks and the partition maintenance commands on 9.7; fix any failure without changing behaviour.
3. Staging: upgrade the replica, then switch over to it as primary with the replica-promotion runbook, then rebuild the old primary as a 9.7 replica; run the full load set and compare p95 with the 8.4 baseline (within 10%).
4. Production in the window: fresh verified backup; same replica-first sequence; smoke tests; 30-minute watch; rollback plan (keep the 8.4 primary untouched as a fallback until the watch ends, with writes frozen during switchover).
5. Restore drill on 9.7 within the same week (full and single-tenant restore).
6. Update runbooks and decisions.md; MS-15 evidence.

DATABASE
No schema changes. Any rename forced by reserved words is an expand-only change with its own migration and is reported as a blocker for review first.

BUSINESS RULES
- No behaviour changes; tenants see no difference.
- Innovation releases never run in production (DEC-201).

UI SPEC
- No UI changes.

SECURITY AND PERMISSIONS
- Database users keep least privilege; TLS between application and database stays required.

TESTS
- Full suites on 9.7 in CI; load set comparison; restore drill.

ACCEPTANCE CRITERIA
1. Production runs MySQL 9.7 LTS with all gates green, the restore drill passed, and MS-15 recorded.
2. CI green; staging and production green.

DO NOT
- Do not combine this upgrade with feature work.
- Do not skip the replica-first sequence.

OUTPUT
Report per Master section 20 with the review findings, load comparison, production watch and drill results. Then run the D13 13.6 gate for PH-19.
```

---

## 14.33 PH-20 Transport Fleet and GPS

**Exit criteria:**
- Phase gate G-1 to G-11.
- Expired driver or vehicle documents block assignments, unless a principal has overridden with a reason.
- GPS positions respect the 15-second throttle and 30-day retention.

### P20.1 Vehicles, Documents, Drivers, Route Assignments and Boarding

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 4 7.42 (vehicles, vehicle_documents, driver_profiles, transport_route_assignments, transport_attendances) with DEC-121; D11 Part 2 11.30 (Transport R3); D10 Part 1 widget transport.trips_today, attention item for vehicle documents, TS home; D10 Part 3 SCR-TRN-001 (vehicle column), SCR-TRN-007 to 010, SCR-TRN-013, SCR-TRN-016; D05 5.7.21; D02 FR-TRN-005, 006, 008; D14 DEC-282, DEC-298.

PHASE GOAL
Manage the fleet: vehicles with documents and expiry reminders, driver profiles with license and verification checks, route assignments of vehicle, driver and attendant with history, R3 seat capacity, and boarding marked by transport staff on a phone with guardian notifications.

ALREADY BUILT
P17.1 routes, stops and allocations with route capacity; employees; vendors (P12.1); notification engine; attention items.

TASKS
1. Migrations (Transport, expand-only): vehicles, vehicle_documents, driver_profiles, transport_route_assignments, transport_attendances exactly as D07 7.42.
2. Vehicles: CreateVehicle and UpdateVehicle (registration number normalized; contractor vendor for contracted vehicles), SetVehicleStatus (workshop, retired; retiring ends active assignments with a warning), AddVehicleDocument and RenewVehicleDocument (file upload; expires_on), command transport:document-reminders daily (30, 7 and 1 days before expiry; reminder_sent_at), attention item vehicle documents expiring in 30 days.
3. Drivers: CreateDriverProfile (employee required; license number, category, expiry; medical fitness; police verification date), license reminders on the same schedule.
4. Route assignments (DEC-298): AssignVehicleToRoute (vehicle, driver, optional attendant employee; uq_tra_route, uq_tra_vehicle, uq_tra_driver; refuses expired driver license or missing or expired fitness, insurance or route permit unless OverrideAssignmentBlock by a principal with reason; ends the previous assignment from the start date), EndRouteAssignment.
5. Capacity (DEC-282): AllocateStudent from P17.1 now uses the lower of route capacity and the active vehicle seating capacity; route list shows both.
6. Boarding: MarkBoarding (transport staff assigned to the route as driver or attendant, or holders of transport.boarding.mark for the branch; one row per student, date and trip type, uq_ttd; status boarded, not_boarded, dropped; source app), guardian notifications transport.boarded (optional, setting transport.notify_boarding) and transport.dropped, transport.not_boarded to guardians and the class teacher; widget transport.trips_today.
7. Guardian transport page: SCR-TRN-016 shows today's boarding status.
8. Reports: transport.boarding_log, transport.fleet_documents.
9. Screens (D10 exactly): SCR-TRN-007 Vehicles, SCR-TRN-008 Vehicle detail, SCR-TRN-009 Drivers, SCR-TRN-010 Route assignments, SCR-TRN-013 Boarding; vehicle column on SCR-TRN-001.
10. decisions.md rows: DEC-121, DEC-282, DEC-298.

DATABASE
vehicles (veh), vehicle_documents (vdc), driver_profiles (drv), transport_route_assignments (tra), transport_attendances (ttd) per D07 Part 4 7.42.

BUSINESS RULES
- One active assignment per route, per vehicle and per driver (DEC-121).
- Expired documents block assignment unless overridden with a reason (DEC-298).
- Seat capacity is the lower of route and vehicle capacity.

UI SPEC
- Boarding (SCR-TRN-013) at 360px: route header, stops in sequence with students, large tap targets for Boarded and Not boarded, Drop mode on the return trip, offline message if the connection drops (marks retry automatically).
- Vehicle detail with document cards showing days to expiry with icon and text, never colour alone.

SECURITY AND PERMISSIONS
- Transport staff see only students on their assigned route; guardian phone numbers masked; route sheets per permission.
- Overrides audited.

TESTS
- Assignment refusals and override; one active assignment rules under concurrency.
- Capacity with vehicle seating lower than route capacity.
- Boarding uniqueness, notifications, transport staff scope.
- Reminder schedule.
- Screen tests, isolation suite, RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. On staging a vehicle with valid documents and a licensed driver is assigned, capacity follows the vehicle, and boarding marks notify a guardian on drop.
2. CI green; staging green.

DO NOT
- Do not let transport staff see other routes' students.

OUTPUT
Report per Master section 20.
```

### P20.2 Maintenance, Fuel, GPS and Live Map

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 4 7.42 (vehicle_maintenance_logs, fuel_logs, gps_devices, gps_positions), 7.50 (partitioning) and DEC-123, A-68; D10 Part 3 SCR-TRN-003 (map pin), SCR-TRN-011, 012, 014, 015, 016 (live location); D12 Part 1 12.8 (webhooks) and CSP rules; D02 FR-TRN-007, 009; D14 DEC-299, P12.1 expense vouchers.

PHASE GOAL
Track running costs and vehicle location: maintenance and fuel logs that create expense vouchers through Accounting, GPS devices through provider adapters with throttled partitioned positions, a Leaflet map for managers, stop pins, and live bus location with a rough ETA for guardians during trip windows.

ALREADY BUILT
P20.1, P12.1 expense vouchers with approval and posting, partition maintenance commands (P09.4), hooks host with stored-first webhook receipts.

TASKS
1. Migrations (expand-only): vehicle_maintenance_logs, fuel_logs, gps_devices exactly as D07 7.42; gps_positions as a partitioned table (daily partitions on recorded_at, 30-day retention, no foreign keys, DEC-123) registered with the partition maintenance command.
2. Maintenance and fuel: RecordMaintenance (type, cost, vendor, odometer, next due date or km), RecordFuel (litres, amount, odometer must not go backwards for the vehicle, receipt photo), CreateExpenseVoucherFromLog (pre-fills an expense voucher from P12.1 with the log's cost, vendor and the transport expense account from setting transport.expense_account_id; the voucher follows its own approval and posting; the log stores expense_voucher_id), km per litre per fill and per month; maintenance due reminders by date or km.
3. GPS (DEC-299): GpsProvider contract (parsePush(request), verify(request), poll(device, since)), FakeGpsProvider, and one real adapter for the provider chosen by the first organization that needs it, built only from that provider's official API documentation (record the version in docs/integrations/); RegisterGpsDevice (uq_gps_device global; credentials encrypted), push endpoint on hooks host stored first, poll command transport:gps-poll every minute for polling providers; ingestion job keeps at most one position per device per 15 seconds and updates gps_devices.last_*.
4. Map: Leaflet bundled locally (no CDN), tiles from maps.tile_url with attribution, CSP img-src limited to the tile host; SCR-TRN-014 Live map (vehicles with last position and freshness, route stops, history replay for a chosen day within 30 days); stop pins on SCR-TRN-003 (drag pin to set latitude and longitude).
5. Guardian live location on SCR-TRN-016: only for the child's allocated route, only during trip windows, position hidden when older than 2 minutes; ETA to the child's stop from remaining stop sequence distance and the route's recent average speed, shown as "about N minutes" with the time it was computed.
6. Reports: transport.fuel_efficiency, transport.maintenance_costs.
7. Screens (D10 exactly): SCR-TRN-011 Maintenance logs, SCR-TRN-012 Fuel logs, SCR-TRN-014 Live map, SCR-TRN-015 GPS devices; map pin on SCR-TRN-003; live location on SCR-TRN-016.
8. decisions.md rows: DEC-123, DEC-299, A-68.

DATABASE
vehicle_maintenance_logs (vml), fuel_logs (fue), gps_devices (gps), gps_positions (gpp, log, partitioned) per D07 Part 4 7.42.

BUSINESS RULES
- Transport never posts journals; costs go through expense vouchers.
- Positions are throttled and kept 30 days; partitions are dropped, not archived (D7 7.50).
- Guardians see location only for their child's route during trip windows.

UI SPEC
- Map page with a vehicle list beside the map, freshness text ("2 min ago"), and a keyboard-accessible list alternative to the map.

SECURITY AND PERMISSIONS
- GPS webhooks verified as the provider documents and stored first; provider credentials encrypted and never logged.
- Map tiles load only from the allowlisted host.

TESTS
- Odometer validation; expense voucher creation from logs and status link.
- Throttle (positions inside 15 seconds dropped); partition creation and drop; device uniqueness across tenants.
- Guardian trip window and staleness rules; ETA with fixtures.
- Webhook verification and stored-first behaviour.
- Screen tests, isolation suite (positions of organization A never shown to B), RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. With the fake provider on staging, a moving vehicle shows on the live map and on the guardian page during the trip window only; a fuel log creates an expense voucher that posts after approval.
2. Phase exit criteria evidence; CI green; staging green.

DO NOT
- Do not load map libraries or tiles from unapproved hosts.
- Do not store positions beyond the throttle.

OUTPUT
Report per Master section 20. Then run the D13 13.6 gate for PH-20.
```

---

## 14.34 PH-21 Inventory and Assets

**Exit criteria:**
- Phase gate G-1 to G-11.
- `stock_balances` matches the movement ledger in the nightly integrity check.
- Negative stock is impossible under concurrency.

### P21.0 UG-5: Laravel 15 (conditional slot)

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D12 Part 2 DEC-200 and 12.31, D13 13.8 UG-5, D14 DEC-294, docs/runbooks/upgrade-gate.md, and the official Laravel 15 upgrade guide and release notes read at the time of the step.

PHASE GOAL
Upgrade to Laravel 15 when the DEC-200 rule is met, before any PH-21 feature work, with no behaviour change. If the rule is not met, record the check and defer the step to the next phase start under the same ID.

ALREADY BUILT
R1, R2 and PH-19 to PH-20 on Laravel 14 and PHP 8.4, MySQL 9.7.

TASKS
1. Check the rule: at least two Laravel 15 point releases, and every composer dependency (including the Laravel Zero version used by darsgah-sync-agent) supports Laravel 15 and PHP 8.4 and 8.5. If not met, write docs/decisions/ug-5-deferred.md with each blocker and stop.
2. Apply the upgrade guide checklist; keep the forbidden-feature rules (property hooks and asymmetric visibility stay banned on Eloquent models, DEC-199); no new Larastan ignores; full suite, nightly, mutation, browser, load and isolation runs.
3. Staging deploy with the full load set; production deploy in the window with a 30-minute watch.
4. Update the upgrade runbook and decisions.md.

DATABASE
No schema changes.

BUSINESS RULES
- No behaviour changes.

UI SPEC
- No UI changes; before and after screenshots of five key screens.

SECURITY AND PERMISSIONS
- D12 12.24 suite green before and after; dependency audit clean.

TESTS
- All suites and gates green; production smoke tests green.

ACCEPTANCE CRITERIA
1. Production runs Laravel 15 with all gates green, or a dated deferral record exists.
2. CI green; staging and production green.

DO NOT
- Do not combine this upgrade with feature work.

OUTPUT
Report per Master section 20 with the rule check and results.
```

### P21.1 Items, Stock Ledger, Receipts, Issues, Transfers, Adjustments and Assets

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 4 7.43 (all tables except maintenance_requests) with DEC-120, DEC-121, DEC-126, A-65; D10 Part 1 widget inventory.low_stock and its attention item; D10 Part 3 SCR-INV-001 to 012 and reports inventory.stock_valuation; D05 5.7.21 (inventory.*) and 5.9 (stock_adjustment); D02 FR-INV-001 to 005, 007; D14 DEC-293, DEC-300, DEC-301.

PHASE GOAL
Track school stock and assets: categories and items, stock locations with custodians, an append-only signed movement ledger with cached balances that can never go negative, receipts from purchase bill lines, issues, returns, transfers including between branches, adjustments and write-offs through approval, reorder alerts, and a fixed asset register with assignments, disposal approval and a depreciation schedule report.

ALREADY BUILT
P12.1 purchase bills with lines, approval engine, employees and rooms, numbering service, integrity check registry, EmployeeClearanceCheckRegistry (P13.1).

TASKS
1. Migrations (Inventory, expand-only): inventory_categories, inventory_items, stock_locations, stock_movements, stock_balances, assets, asset_assignments (alias aag, DEC-293) exactly as D07 7.43.
2. StockLedger service (the only writer of stock_movements and stock_balances): Post(movements[]) inside one transaction; locks the stock_balances rows (creating missing rows first with insert-ignore) in a fixed order of item and location to avoid deadlocks; applies signed quantities; CHECK quantity >= 0 makes negative stock fail; weighted average cost updated on receipts (A-65); movement numbers from the numbering key stock_movement.
3. Actions: CreateItem, UpdateItem, CreateLocation, RecordOpeningStock (once per item and location), ReceiveStock (against a purchase_bill_line_id of an approved bill: quantity received across receipts never exceeds the billed quantity; unit cost from the line; or direct receipt with reason), IssueStock (to employee or section; refused when stock is short), ReturnStock, TransferStock (paired movements with transfer_group_ulid; inter-branch per DEC-301), RequestStockAdjustment (adjustment or write_off with reason; SubmitApprovalRequest stock_adjustment), ApplyStockAdjustment (handler; posts the movement with approval_request_id).
4. Assets: RegisterAssetsFromBillLine (asset items: one asset per unit with tags from numbering key asset_tag, cost, vendor, warranty), RegisterAsset (direct with reason), AssignAsset and ReturnAsset (employee or room; condition out and in; uq_aag_active), SetAssetStatus (under_repair, lost with reason), RequestAssetDisposal (SubmitApprovalRequest asset_disposal per DEC-300; handler sets disposed fields), employee clearance check for assets still assigned at separation.
5. Approval types registered by Inventory: stock_adjustment (1 principal_of_branch; inventory.adjustment.approve) and asset_disposal (DEC-300).
6. Reorder alerts: nightly snapshot for widget inventory.low_stock and attention item low stock items.
7. Integrity: stock_balances recomputed from stock_movements nightly (DEC-120, DEC-126) in IntegrityCheckRegistry; mismatches alert and never auto-fix.
8. Reports (ReportDefinition): inventory.stock_valuation (weighted average), inventory.movements, inventory.asset_register, inventory.depreciation_schedule (straight line, report only, DEC-300).
9. Screens (D10 exactly): SCR-INV-001 to SCR-INV-012.
10. Definitions: Permissions.php for inventory.* per D05 5.7.21 plus inventory.disposal.approve and .reject; settings inventory.disposal_owner_threshold; numbering keys stock_movement and asset_tag.
11. decisions.md rows: DEC-120, DEC-293, DEC-300, DEC-301, A-65.

DATABASE
inventory_categories (ict), inventory_items (iit), stock_locations (stl), stock_movements (smv, append-only), stock_balances (sbl), assets (ast), asset_assignments (aag) per D07 Part 4 7.43 with DEC-293.

BUSINESS RULES
- Stock is a ledger: movements are append-only and signed; balances are a cache updated in the same transaction (DEC-120).
- Negative stock is impossible; adjustments and write-offs need approval.
- Inventory and assets never post journals in R3 (DEC-300, A-65).
- Receipts never exceed billed quantities.

UI SPEC
- Issue and transfer forms show on-hand quantity live and block over-issue before submit; the server check stays authoritative.
- Asset detail with an assignment timeline and condition history.

SECURITY AND PERMISSIONS
- Inter-branch transfers need the permission in both branches; adjustments and disposals audited with reasons.

TESTS
- Concurrency: two issues of the last unit at once; exactly one succeeds and the balance stays at zero.
- Paired transfer atomicity (failure rolls back both); inter-branch permission.
- Receipt limit against bill lines; weighted average cost.
- Adjustment and disposal approval flows; integrity check detects a tampered balance.
- Asset one-active-assignment rule; clearance check on separation.
- Screen tests, isolation suite, RBAC fixture, ScreenCoverageTest; mutation testing on StockLedger with MSI at least 80%.

ACCEPTANCE CRITERIA
1. On staging stock is received from a purchase bill, issued, transferred to another branch and adjusted through approval; assets are registered, assigned and disposed through approval; integrity check clean.
2. Phase exit criteria evidence; CI green; staging green.

DO NOT
- Do not write stock_movements or stock_balances outside StockLedger.
- Do not post inventory journals.

OUTPUT
Report per Master section 20 with the concurrency evidence. Then run the D13 13.6 gate for PH-21.
```

---

## 14.35 PH-22 Administration R3, Complaints and Counseling

**Exit criteria:**
- Phase gate G-1 to G-11.
- Counseling notes are unreadable outside DEC-302 in every route, export, search and API test.
- Complaint SLAs raise attention on time.

### P22.1 Visitor Logs, Maintenance Requests and Complaints

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 3 7.33 (visitor_logs, complaints) and D07 Part 4 7.43 (maintenance_requests); D10 Part 1 REC home quick action; D10 Part 3 SCR-ADN-006, 007, 008, 011, SCR-INV-013, 014; D05 5.7.21 (inventory.maintenance) and 5.7.22 (administration.visitor, administration.complaint); D02 FR-ADN-006, FR-ADN-007, FR-INV-006, FR-COMM-009; D01 GAP-44; D14 DEC-295, DEC-303.

PHASE GOAL
Run the front office and facilities: a visitor log with badges and host notifications, maintenance requests from any staff member with assignment and costs, and complaints from guardians and staff with SLA tracking, escalation, resolution and satisfaction.

ALREADY BUILT
R1 administration, gate passes (P16.3), assets (P21.1), expense vouchers (P12.1), notification engine, attention items, portals.

TASKS
1. Migrations (expand-only): visitor_logs, complaints exactly as D07 7.33; maintenance_requests exactly as D07 7.43.
2. Visitors: CheckInVisitor (name, mobile E.164, CNIC last4 only, purpose, host employee, badge number, optional photo captured by camera and re-encoded), host notified in-app, CheckOutVisitor, end-of-day list of visitors not checked out, command administration:purge-visitor-photos daily (DEC-303), badge print (document type visitor_badge).
3. Maintenance requests: ReportProblem (asset or room, category, priority, description, photo), AssignRequest, StartWork, ResolveRequest (resolution, cost), CloseRequest, RejectRequest (reason), CreateExpenseVoucherFromRequest (as in P20.2), linked asset set to under_repair while open and back on resolution; requester notified on each status change; numbering key maintenance_request.
4. Complaints (DEC-303): SubmitComplaint (guardian portal for own children, staff for themselves, office for others; category from setting; priority; sla_due_at from settings), AssignComplaint, UpdateComplaintStatus, ResolveComplaint (resolution text), RateComplaint (complainant within 7 days), ReopenComplaint (within 7 days), command administration:complaint-sla every 15 minutes (attention items and one principal notification on breach); numbering key complaint.
5. Reports: administration.visitors, administration.complaints_sla, inventory.maintenance_costs.
6. Screens (D10 exactly): SCR-ADN-006 Visitor log, SCR-ADN-007 Complaints, SCR-ADN-008 Complaint detail, SCR-ADN-011 Guardian: submit complaint, SCR-INV-013 Maintenance requests, SCR-INV-014 Report a problem; REC quick action Log visitor.
7. decisions.md rows: DEC-295, DEC-303, GAP-44.

DATABASE
visitor_logs (vst), complaints (cmp) per D07 Part 3 7.33; maintenance_requests (mrq) per D07 Part 4 7.43.

BUSINESS RULES
- SLA hours and categories are settings; breaches escalate through attention items and one notification.
- Guardians see only their own complaints; staff complaints about colleagues visible only to BM and PR.
- Visitor photos are purged after the retention period; CNIC last4 only.

UI SPEC
- Visitor check-in form optimized for the front desk at 1024px with camera capture; complaint tracking timeline for guardians at 360px.

SECURITY AND PERMISSIONS
- Photos stored privately with signed URLs; complaint visibility enforced in queries, not only in the UI.

TESTS
- SLA due computation and breach escalation timing; reopen and rating windows.
- Visitor photo purge; host notification.
- Maintenance status flow, asset status changes, expense voucher link.
- Complaint visibility for guardians and staff.
- Screen tests, isolation suite, RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. On staging a guardian complaint breaches SLA and escalates, then resolves and is rated; a maintenance request runs from report to resolution with an expense voucher; a visitor checks in and out.
2. CI green; staging green.

DO NOT
- Do not store full CNIC numbers for visitors.

OUTPUT
Report per Master section 20.
```

### P22.2 Counseling Sessions

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 4 7.44 (counseling_sessions) and DEC-122; D12 Part 1 12.3 (Restricted classification) and the redaction and audit rules; D10 Part 3 SCR-DSC-008, 009; D05 5.7.22 (discipline.counseling) and the masking table row for counseling records, A-41; D02 FR-DSC-005; D14 DEC-293, DEC-302.

PHASE GOAL
Let counselors keep confidential session records with encrypted notes, restricted sessions only their author can read, read-access logging, follow-up reminders and referrals from discipline incidents, with no leak through any other surface.

ALREADY BUILT
P17.2 discipline incidents and timeline, encryption casts, audit redaction processor, 2FA for sensitive permissions, global search exclusions mechanism.

TASKS
1. Migration (expand-only): counseling_sessions exactly as D07 7.44 with alias csl (DEC-293); notes_encrypted uses the encrypted cast.
2. Actions: ScheduleSession, RecordSession (notes, confidentiality standard or restricted, follow-up date), UpdateSessionNotes (author only), CancelSession, ReferFromIncident (from a discipline incident; links discipline_incident_id), ViewSessionNotes (DEC-302 rules; writes an audit viewed event with no content).
3. Timeline: counseling sessions write a timeline row "Counseling session held" with no content, visible only to holders of discipline.counseling.view.
4. Exclusions (DEC-122): model registered as excluded in global search, import and export registries, report runner and API resources; audit records changed field names only; redaction processor drops notes from logs and exceptions.
5. Follow-up reminders to the author only; widget for counselors: sessions due this week.
6. Counselor role guidance: a role template note in SCR-IAM role creation hints at the counseling permissions (custom role, A-41).
7. Screens (D10 exactly): SCR-DSC-008 Counseling sessions, SCR-DSC-009 Counseling session form.
8. decisions.md rows: DEC-122, DEC-293, DEC-302.

DATABASE
counseling_sessions (csl) per D07 Part 4 7.44 with DEC-293.

BUSINESS RULES
- Standard notes: author and view permission holders with 2FA; restricted notes: author only (DEC-302).
- Guardians, teachers and students never see counseling records.
- Notes never leave the two counseling screens.

UI SPEC
- Session list shows a lock icon with text "Restricted" for restricted sessions; notes field with a confidentiality reminder; no print or export buttons.

SECURITY AND PERMISSIONS
- Sensitive permission with 2FA; every notes read logged; impersonated sessions cannot open notes.

TESTS
- Notes encrypted at rest (raw column is not plaintext).
- Restricted notes unreadable by a principal with view permission; standard notes readable with 2FA only.
- Absence from global search, exports, imports, report runner, API, timeline content, notifications, logs and exception reports (leak test suite).
- Audit stores field names only; viewed events written.
- Impersonation blocked.
- Screen tests, isolation suite, RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. The leak test suite is green and a counselor records, follows up and refers sessions on staging.
2. Phase exit criteria evidence; CI green; staging green.

DO NOT
- Do not add counseling to any export, report or API.

OUTPUT
Report per Master section 20 with the leak test list. Then run the D13 13.6 gate for PH-22.
```

---

## 14.36 PH-23 Analytics, Designers and Offline Attendance

**Exit criteria:**
- Phase gate G-1 to G-11.
- Offline sync never overwrites a newer mark or a finalized sheet.
- The template designer cannot inject markup.
- MS-16 is reached.

### P23.0 D7 Addendum R3-A

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D13 DEC-224; D07 Part 1 (conventions, 7.14 documents), Part 2 7.22 (student attendance), Part 3 7.27 (accounts, journals) and 7.31 (timetables); D14 DEC-293, DEC-304 to DEC-309.

PHASE GOAL
Write the D7 addendum for PH-23, get product owner approval, and create the expand-only migrations, models, factories, isolation registrations and permission and screen definitions, with no feature logic.

ALREADY BUILT
Every R1, R2 and PH-19 to PH-22 table.

TASKS
1. Write docs/schema/d7-addendum-r3a.md with these tables under D7 conventions (types, keys, tenancy markers):
   - analytics_snapshots [B] alias ans: snapshot_type VARCHAR(40) (result_term, attendance_month, fee_month, enrollment), academic_session_id? FK, exam_term_id? (soft), period_month DATE?, scope_hash CHAR(64), scope JSON, metrics JSON, source_version VARCHAR(64), computed_at DATETIME. Keys uq_ans (organization_id, branch_key, snapshot_type, scope_hash) with branch_key generated, ix_ans_type (organization_id, snapshot_type, computed_at).
   - document_template_versions [T-] alias dtv: document_template_id FK, version SMALLINT UNSIGNED, layout JSON, page_settings JSON, status (draft, published, retired), published_at?, published_by?, created_by. Keys uq_dtv (organization_id, document_template_id, version).
   - timetable_generation_runs [B] alias tgr: academic_session_id FK, shift_id?, base_timetable_id? FK->timetables, result_timetable_id? FK->timetables, constraints JSON, seed INT UNSIGNED, time_limit_seconds SMALLINT UNSIGNED, status (queued, running, solved, partial, failed, cancelled), score INT?, unplaced_count SMALLINT UNSIGNED?, unplaced JSON?, requested_by, started_at?, finished_at?. Keys ix_tgr (organization_id, branch_id, created_at).
   - offline_sync_batches [B] alias ofb: client_batch_ulid CHAR(26), user_id FK, device_label VARCHAR(100)?, sync_date DATE, item_count, accepted_count, rejected_count, conflict_count SMALLINT UNSIGNED, clock_drift_seconds INT?, received_at. Keys uq_ofb (organization_id, client_batch_ulid).
   - attendance_sync_conflicts [B] alias acf: offline_sync_batch_id FK, section_attendance_sheet_id FK, student_enrollment_id FK, attendance_date DATE, offline_status VARCHAR(10), server_status VARCHAR(10), offline_marked_at DATETIME, server_changed_at DATETIME, status (open, kept_server, applied_offline), resolved_by?, resolved_at?, note?. Keys ix_acf (organization_id, branch_id, status, attendance_date).
   - Enum additions and expand-only column per DEC-304.
2. Add D10 rows for SCR-RPT-010, SCR-RPT-011, SCR-ACC-022, SCR-TTB-012, SCR-ATT-026 and the D5 permissions in DEC-305 to docs/screens and docs/rbac addenda.
3. After approval (record the approver and date in the addendum): migrations, models with tenancy traits, factories, isolation suite registrations, Permissions.php entries, route stubs returning 404 until their steps, and TableAliasUniquenessTest green.

DATABASE
The five tables, enum values and column above.

BUSINESS RULES
- Expand-only; no behaviour change in production.

UI SPEC
- None.

SECURITY AND PERMISSIONS
- New permissions synced with identity:sync-permissions on staging.

TESTS
- Migrations up from empty and on a copy of production schema; isolation suite registrations; alias uniqueness.

ACCEPTANCE CRITERIA
1. Addendum approved and recorded; migrations deployed to staging and production with no behaviour change.
2. CI green.

DO NOT
- Do not write feature logic in this step.

OUTPUT
Report per Master section 20 with the approved addendum.
```

### P23.1 Result Analytics, Branch Comparison and Advanced Consolidation

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D02 FR-RES-014, FR-RPT-009, FR-ACC-016; D10 Part 3 SCR-RPT-006 (branch comparison R1), report catalog 10.47; D09 chart rules and tokens; D07 Part 3 7.27 (journals, system roles); D14 P12.3 (StatementBuilder), DEC-304, DEC-305, DEC-309.

PHASE GOAL
Give owners and principals real analytics from nightly snapshots: result analytics per term with trends across terms, branch comparison with drill-down, attendance, fee and enrollment trends, and advanced consolidation with inter-branch transfers that eliminate in consolidated statements.

ALREADY BUILT
R1 results and branch comparison, P12.3 statements, report runner on the replica, P23.0 tables.

TASKS
1. Snapshot builders (DEC-309) registered in AnalyticsSnapshotRegistry: ResultTermSnapshot (per term and class and section and subject: mean, median, pass rate, grade distribution, top and bottom ten by student enrollment id, subject teacher), AttendanceMonthSnapshot (attendance rate by class and section, chronic absence count at 3 or more absences in the month), FeeMonthSnapshot (billed, collected, collection rate, aging buckets), EnrollmentSnapshot (admissions, withdrawals, net change by class); command reporting:analytics-snapshots at 02:30 on the replica; rebuilt on ResultPublished or republication events.
2. Analytics screens: SCR-RPT-010 Result analytics (term selector, class and subject filters, distribution chart, trend across terms for a cohort, subject comparison, teacher view limited to the viewer's visibility), SCR-RPT-011 Trends and branch analytics (organization-level tiles and line charts, branch comparison with drill-down into SCR-RPT-003 reports). Charts follow D09 chart tokens, one axis per chart, legends and a table view for each chart, computed_at shown.
3. Inter-branch transfers (FR-ACC-016): RecordInterBranchTransfer (cash or bank from branch A to branch B: lines Dr receiving branch bank, Cr inter_branch_clearing for branch B; Dr inter_branch_clearing, Cr sending branch bank for branch A; entry_type inter_branch_transfer; journal approval threshold applies), system role inter_branch_clearing backfilled into every chart; SCR-ACC-022 Inter-branch transfers.
4. Consolidation: StatementBuilder consolidated mode eliminates inter_branch_clearing balances; integrity check that inter_branch_clearing nets to zero across the organization.
5. Widgets for OW and PR homes: reporting.result_trend, reporting.collection_trend.
6. decisions.md rows: DEC-309, FR-ACC-016 rules.

DATABASE
Uses analytics_snapshots; enum and system role from P23.0.

BUSINESS RULES
- Analytics read snapshots and never change operational data.
- Visibility levels apply to snapshot queries the same as to live data (teachers see their own sections only).
- Inter-branch transfers balance per branch and net to zero for the organization.

UI SPEC
- Every chart has a table view and text summary; colour never carries meaning alone.

SECURITY AND PERMISSIONS
- reporting.analytics.view; accounting.transfer.create; student names in analytics only where the viewer can see those students.

TESTS
- Snapshot correctness against fixtures; rebuild on republication; visibility in analytics queries.
- Inter-branch posting balanced per branch; elimination in consolidated statements; integrity check.
- Performance: analytics pages within budgets on the R3 load dataset.

ACCEPTANCE CRITERIA
1. On staging analytics match fixture calculations and consolidated statements eliminate a transfer.
2. CI green; staging green.

DO NOT
- Do not compute analytics on the primary database during the day.

OUTPUT
Report per Master section 20 with chart screenshots and fixture comparisons.
```

### P23.2 Document and Certificate Template Designer

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 1 7.14 (document_templates, issued_documents) and BR-DOC-01; D10 Part 2 SCR-DOC-003 and the document screens; D14 Part 3 P02.4 (document engine, Gotenberg, fonts), DEC-307; D02 FR-DOC-010; D09 accessibility rules for drag interactions.

PHASE GOAL
Let organization admins design certificates, report card headers, ID cards and letters with a block-based layout editor whose output is data, rendered through one safe renderer, versioned, and never changing documents already issued.

ALREADY BUILT
Document engine with system Blade templates per type, issued documents with snapshots and stored PDFs, verification QR (P16.4).

TASKS
1. DocumentFieldRegistry: each document type registers its merge fields (key, label in English and Urdu, sample value, data type) and repeating collections; Restricted fields are never registrable (arch test).
2. Layout schema (JSON Schema in docs/schema/layout.schema.json): page settings (size, orientation, margins, background image), blocks text (with {{field}} merge tokens), field, image (logo, signature, photo from registry), table (collection, columns), qr_verification, barcode, line, spacer, columns (nested); per-block position, size, font family from the bundled set, size, weight, alignment, direction (ltr, rtl); validation rejects unknown keys.
3. Renderer: x-doc.layout component turning a validated layout and data into HTML for Gotenberg; text escaped; no raw HTML; images only from stored files of the organization.
4. Actions: CreateTemplateDraft (from the system template or an existing version), SaveLayout (validates schema; new draft version), PreviewTemplate (sample data from the registry, rendered PDF), PublishTemplateVersion (sets current_version_id; previous version retired; audit), RevertToVersion.
5. Document engine: types with a current designer version render from it; others keep the system Blade template; issued documents store template_version and keep their stored PDF.
6. Screen (D10 exactly): SCR-DOC-003 Template designer (canvas with blocks, properties panel, field picker, preview, version list); drag and drop plus keyboard move and resize controls (WCAG 2.5.7).
7. decisions.md rows: DEC-307, BR-DOC-01.

DATABASE
Uses document_template_versions and document_templates.current_version_id from P23.0.

BUSINESS RULES
- Layouts are data; no HTML, CSS or script from users (DEC-307).
- Publishing never changes issued documents.

UI SPEC
- Canvas at A4 or ID card size with rulers and snap grid; Urdu text blocks right-aligned by default.

SECURITY AND PERMISSIONS
- document.template.update; publishing audited.

TESTS
- Schema validation rejects unknown keys and script payloads; renderer escapes merge values containing HTML.
- Restricted fields cannot be registered.
- Issued document reprint unchanged after a new version is published.
- Bilingual rendering snapshot tests.
- Screen tests, isolation suite, RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. On staging an admin designs and publishes a leaving certificate layout, issues one, publishes another version, and the first certificate reprints unchanged.
2. CI green; staging green.

DO NOT
- Do not accept user-supplied HTML or CSS.

OUTPUT
Report per Master section 20 with designer screenshots and the security tests.
```

### P23.3 Assisted Timetable Generation

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 2 (teaching_assignments periods_per_week, employees max_periods_per_week, class_subjects), Part 1 (rooms type), Part 3 7.31 (timetables, timetable_entries, TimetableConflictDetector); D10 Part 3 SCR-TTB-002 to 004; D02 FR-TTB-009; D14 DEC-304, DEC-305, DEC-308.

PHASE GOAL
Help coordinators build a timetable faster: a generator that takes teaching assignments, bell schedules, rooms and constraints and produces a conflict-free draft version for review, with unplaced lessons explained.

ALREADY BUILT
R1 timetable editor with conflict detection and activation; P23.0 timetable_generation_runs.

TASKS
1. Generator wizard (SCR-TTB-012): session, shift, bell schedule, scope (all classes or selected), teacher unavailability grid, subject options (heavy flag, double periods, lab room required), seed (random by default, editable), time limit.
2. GenerateTimetable job (DEC-308): builds lessons from active teaching assignments; places most constrained lessons first with backtracking; then local search to improve the soft score until the time limit; reads other branches' active timetables for shared teachers; writes a new draft timetable with timetable_entries through the same validation as the editor; stores score, unplaced lessons and reasons in the run.
3. Run management: status polling, cancel, rerun with the same seed for identical output, open result in SCR-TTB-003.
4. Activation stays with SCR-TTB-004 and TimetableConflictDetector.
5. decisions.md rows: DEC-308.

DATABASE
Uses timetable_generation_runs; writes timetables and timetable_entries as drafts.

BUSINESS RULES
- Generation never changes an active timetable; output is always a draft.
- Hard constraints are never violated; unplaced lessons are reported instead.

UI SPEC
- Result summary with score breakdown, unplaced list with reasons, and a link to edit.

SECURITY AND PERMISSIONS
- timetable.generation.run; runs limited to one queued or running per branch at a time.

TESTS
- Determinism with the same seed; no hard constraint violations on fixtures of a 20-section school; unplaced reasons when infeasible; cross-branch teacher respect; time limit honored.

ACCEPTANCE CRITERIA
1. On the demo school the generator produces a draft that passes the conflict detector and places at least 95% of lessons within the time limit.
2. CI green; staging green.

DO NOT
- Do not activate generated timetables automatically.

OUTPUT
Report per Master section 20 with the run results on demo data.
```

### P23.4 Offline-Capable Attendance Marking

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D14 Part 4 P05.1 (MarkStudentAttendance, sheet versions DEC-162, finalization DEC-180), D10 Part 2 SCR-ATT-002 and wireframe W-01, D12 Part 1 CSP and session rules, D01 GAP-41, D02 FR-ATT-026, D14 DEC-304, DEC-305, DEC-306.

PHASE GOAL
Let teachers in classrooms with poor connectivity mark daily section attendance offline and sync safely later, with conflicts surfaced instead of overwritten.

ALREADY BUILT
Section attendance with sheet versions, revisions and period locks; P23.0 tables.

TASKS
1. Service worker (bundled, CSP-safe, scope limited to the attendance marking route) caching the page shell and assets; IndexedDB store for the roster snapshot and the mark queue as DEC-306 defines.
2. Page behaviour on SCR-ATT-002: an offline banner, marks stored locally with a pending count, automatic sync on reconnect with backoff, manual "Sync now", and a result summary (applied, rejected with reasons, conflicts).
3. SyncOfflineAttendance endpoint (session auth, CSRF, idempotent by client_batch_ulid; writes offline_sync_batches; calls MarkStudentAttendance per mark with source offline and the base version; conflicts into attendance_sync_conflicts; rejects finalized sheets, locked periods and dates other than the sync day).
4. Conflict resolution: SCR-ATT-026 Offline sync conflicts (keep server value or apply offline value through a normal revision with reason).
5. Local data clearing on success, logout and midnight; nothing beyond the roster stored.
6. decisions.md rows: DEC-306, GAP-41.

DATABASE
Uses offline_sync_batches and attendance_sync_conflicts; source offline on attendance and revisions.

BUSINESS RULES
- Offline marks never overwrite newer server changes or finalized sheets (DEC-306).
- Only the current day's daily section attendance works offline.

UI SPEC
- Offline banner and pending count readable at 360px; sync result shown as a list with reasons.

SECURITY AND PERMISSIONS
- Same mark permission and section scope as online; a session expired while offline requires sign-in before sync, and the queue survives sign-in.

TESTS
- Browser tests with network offline: mark, reconnect, sync applied.
- Conflict when another user changed the row; rejection after finalization; idempotent resend of a batch.
- Data cleared on logout; service worker scope limited.

ACCEPTANCE CRITERIA
1. Phase-level evidence: offline marks sync without overwriting newer marks or finalized sheets on staging.
2. CI green; staging green.

DO NOT
- Do not cache other pages or personal data offline.

OUTPUT
Report per Master section 20 with browser test evidence.
```

### P23.5 UG-4: PHP 8.5

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D12 Part 2 DEC-199 and 12.31, D13 13.8 UG-4, docs/runbooks/upgrade-gate.md, and the official PHP 8.5 migration guide read at the time of the step.

PHASE GOAL
Run production and the sync agent on PHP 8.5 before PHP 8.4 reaches end of life on 31 Dec 2028, with no behaviour change, and record MS-16.

ALREADY BUILT
CI has tested PHP 8.4 and 8.5 since UG-1; production runs PHP 8.4.

TASKS
1. Review deprecations and backward-incompatible changes in the 8.5 migration guide against the codebase and both repositories; fix deprecation notices (the test suite fails on deprecations).
2. Build runtime images on PHP 8.5 (FPM, Horizon workers, scheduler); rebuild darsgah-sync-agent binaries with static-php-cli on 8.5 and publish signed releases.
3. composer.json "php": "^8.5"; CI matrix becomes PHP 8.5 and the next supported version if released; Larastan phpVersion 80500.
4. Staging with full load set; production deploy in the window with a 30-minute watch; sync agent self-update verified on a test site.
5. Update runbooks and decisions.md; MS-16 evidence.

DATABASE
No schema changes.

BUSINESS RULES
- No behaviour changes.

UI SPEC
- No UI changes.

SECURITY AND PERMISSIONS
- Dependency audit clean on 8.5.

TESTS
- All suites and gates green on 8.5; load comparison within 10% of the baseline.

ACCEPTANCE CRITERIA
1. Production and the sync agent run PHP 8.5; MS-16 recorded.
2. CI green; staging and production green.

DO NOT
- Do not combine this upgrade with feature work.

OUTPUT
Report per Master section 20. Then run the D13 13.6 gate for PH-23.
```

---

## 14.37 PH-24 SaaS Billing, Partner API, Push and R3 Launch

**Exit criteria:**
- MS-17 is reached.

### P24.0 D7 Addendum R3-B

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D13 DEC-224; D07 Part 1 (conventions, organizations, plans, personal_access_tokens), Part 3 7.27 (bank_statement_lines); D14 DEC-310 to DEC-316.

PHASE GOAL
Write the D7 addendum for PH-24, get product owner approval, and create the expand-only migrations, models, factories, isolation registrations and permission and screen definitions, with no feature logic. Record the product owner's choice of the additional locale (DEC-315).

ALREADY BUILT
Every table through PH-23.

TASKS
1. Write docs/schema/d7-addendum-r3b.md:
   - subscriptions [P] alias sbs: organization_id FK, plan_id FK, billing_cycle (monthly, annual), status (trialing, active, past_due, suspended, cancelled), current_period_start DATE, current_period_end DATE, next_invoice_on DATE?, cancel_at_period_end TINYINT(1), started_at, cancelled_at?. Keys uq_sbs_live (organization_id, live_marker) with live_marker generated for statuses other than cancelled.
   - subscription_invoices [P] alias sbi: invoice_no VARCHAR(30) (gapless per platform fiscal year), subscription_id FK, period_start, period_end, subtotal, tax_amount, total DECIMAL(14,2), currency CHAR(3) (PKR), tax_breakdown JSON, status (draft, issued, paid, overdue, void), issued_at?, due_on?, paid_at?, stored_file_id?. Keys uq_sbi_no (invoice_no), ix_sbi (organization_id, status, due_on).
   - subscription_invoice_lines [P] alias sil: subscription_invoice_id FK, line_type (plan, proration, addon, credit), description, quantity, unit_amount, amount. Keys ix_sil (subscription_invoice_id).
   - subscription_payments [P] alias sbp: subscription_invoice_id FK, provider VARCHAR(30), provider_txn_id VARCHAR(100)?, merchant_txn_ref VARCHAR(40), amount, status (initiated, pending, verified, failed, expired), verified_at?, last_response JSON (secrets removed). Keys uq_sbp_ref (merchant_txn_ref), uq_sbp_provider (provider, provider_txn_id).
   - partner_api_keys [T] alias pak: name, key_prefix CHAR(8), key_hash CHAR(64), scopes JSON, branch_ids JSON?, ip_allowlist JSON?, rate_limit_per_minute SMALLINT UNSIGNED, expires_at?, last_used_at?, revoked_at?, created_by. Keys uq_pak_hash (key_hash), ix_pak (organization_id, revoked_at).
   - push_device_registrations [T] alias pdr: user_id FK, platform (android, ios, web), token_encrypted TEXT, token_hash CHAR(64), app_version VARCHAR(20)?, last_seen_at, revoked_at?. Keys uq_pdr_token (token_hash), ix_pdr_user (organization_id, user_id, revoked_at).
   - bank_api_connections [B] alias bac: provider VARCHAR(40), account_id FK->accounts (bank ledger account), credentials_encrypted TEXT, mode (sandbox, live), status (active, inactive, error), last_synced_at?, last_error?. Keys uq_bac (organization_id, provider, account_id).
   - organizations.billing_mode and bank_statement_lines.source per DEC-310.
2. Add D10 rows: SCR-SET-007 Subscription and plan (OW), SCR-SET-008 Billing invoices and payment (OW), SCR-PLT-021 Subscriptions (platform billing), SCR-PLT-022 Platform invoices, SCR-PLT-023 Billing settings (tax rates, platform merchant accounts, FCM credentials), SCR-PLT-024 Locales, SCR-IAM-028 Partner API keys (OW, OA), SCR-ACC-023 Bank API connections (OW, ACT). Add D5 permissions: platform.subscription.view and .manage (OW, sensitive), platform.billing.view and .manage (platform billing role), identity.partner_key.view, .create and .revoke (OW, OA, sensitive), accounting.bank_connection.view and .manage (OW, ACT, sensitive).
3. After approval (record approver, date and the chosen locale): migrations, models, factories, isolation registrations (platform tables are excluded from tenant scopes and protected by platform guards), Permissions.php entries, route stubs, TableAliasUniquenessTest green.

DATABASE
The seven tables and two columns above.

BUSINESS RULES
- Expand-only; no behaviour change.

UI SPEC
- None.

SECURITY AND PERMISSIONS
- Platform billing tables writable only by platform billing actions; tenants read their own subscription and invoices through queries scoped to their organization.

TESTS
- Migrations; isolation registrations; platform guard tests; alias uniqueness.

ACCEPTANCE CRITERIA
1. Addendum approved and recorded; migrations deployed with no behaviour change.
2. CI green.

DO NOT
- Do not write feature logic in this step.

OUTPUT
Report per Master section 20 with the approved addendum.
```

### P24.1 Self-Service Subscription Billing

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D01 GAP-01; D02 FR-PLT-010; D04 (tenant states); D07 Part 1 (plans, plan_modules, usage_counters); D10 Part 2 SCR-PLT-008 to 011; D14 P11.1 to P11.3 (gateway adapters and verification), P18.1 (limits), DEC-310, DEC-311.

PHASE GOAL
Let organizations on self-service billing choose and change plans, receive platform invoices, and pay online, with dunning that suspends to read-only and never deletes data; give the platform billing team subscription and invoice management.

ALREADY BUILT
Plans, entitlements, usage metering and limits, gateway adapters with verification-first posting, document engine, P24.0 tables.

TASKS
1. Platform merchant accounts: the platform's own JazzCash, Easypaisa and 1Bill accounts configured in SCR-PLT-023 (encrypted, never in tenant tables); adapters reused with a platform context.
2. Actions: StartSubscription (from trial; plan and cycle), ChangePlan (upgrade prorated by day with an immediate proration invoice; downgrade at period end, refused while usage exceeds target limits), CancelSubscription (at period end), GenerateInvoices (command billing:generate-invoices daily: invoices on next_invoice_on, gapless numbers, tax from platform settings with effective dates, PDF through the document engine), PayInvoiceOnline (hosted or redirect checkout; verification before marking paid, DEC-166), RecordManualPayment (platform billing, bank transfer with reference; audited), DunningRun (command billing:dunning daily: overdue after grace, reminders to the owner at set intervals, organization state suspended after billing.suspend_after_days; reactivation on payment), VoidInvoice (platform billing with reason; replaced by a new invoice).
3. Entitlements follow the active subscription's plan; module changes apply at the effective date.
4. Tenant screens: SCR-SET-007 Subscription and plan (current plan, usage against limits, compare plans, change, cancel), SCR-SET-008 Billing invoices and payment (list, PDF, pay online, payment result with the DEC-166 verifying state).
5. Platform screens: SCR-PLT-021 Subscriptions, SCR-PLT-022 Platform invoices, SCR-PLT-023 Billing settings.
6. Owner notifications: invoice issued, payment received, overdue, suspension warning, suspended, reactivated.
7. decisions.md rows: DEC-311, GAP-01.

DATABASE
subscriptions (sbs), subscription_invoices (sbi), subscription_invoice_lines (sil), subscription_payments (sbp) per P24.0.

BUSINESS RULES
- Subscription money goes to the platform's accounts and never touches tenant ledgers (DEC-311).
- Invoices are gapless and immutable once issued; corrections void and reissue.
- Non-payment suspends to read-only and never deletes data.
- Payments post only after provider verification.

UI SPEC
- Plan comparison with limits and modules; usage bars; invoice PDF with platform letterhead and tax registration.

SECURITY AND PERMISSIONS
- platform.subscription.manage (owner, 2FA); platform billing role for platform screens; every billing action audited.

TESTS
- Proration amounts; downgrade refusal over limits; gapless numbering under concurrency; tax by effective date.
- Payment trust test for subscription payments; duplicate callbacks post once.
- Dunning timeline to suspension and reactivation; manual billing_mode organizations untouched.
- Screen tests, platform guard tests, RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. In provider sandboxes a trial organization subscribes, upgrades, pays an invoice online, misses a payment, is suspended to read-only, pays and is reactivated.
2. CI green; staging green.

DO NOT
- Do not mix platform revenue into tenant accounting.
- Do not delete or hide tenant data for non-payment.

OUTPUT
Report per Master section 20 with the sandbox walkthrough.
```

### P24.2 Partner API and Push Notifications

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D08 8.15 (API envelope, versioning, ETags); D14 P18.2 (API v1), DEC-312, DEC-313; D12 Part 1 API security; D02 FR-NTF-015; D11 Part 2 11.27 (notification pipeline).

PHASE GOAL
Open a read-only partner API with organization-owned keys and scopes, and add FCM push as a notification channel for future apps.

ALREADY BUILT
API v1 with portal tokens and contract tests, export masking rules, notification pipeline with channels and preferences, P24.0 tables.

TASKS
1. Partner keys: CreatePartnerKey (name, scopes from the DEC-312 catalog, branch scope, rate limit, optional IP allowlist, expiry; shown once), RevokePartnerKey, RotatePartnerKey; SCR-IAM-028 Partner API keys with last used and request counts.
2. Partner API under api.{platform_domain}/partner/v1: AuthenticatePartnerKey middleware (prefix lookup, constant-time hash compare, IP allowlist, expiry, revocation), scope checks per route, branch scope applied to queries, rate limits per key, actor type api_key in audit; endpoints for the catalog resources using existing Queries and Resources with export masking; cursor pagination; OpenAPI 3.1 partner spec with contract tests.
3. Push: FcmPushGateway on the FCM HTTP v1 API with the platform service account (credentials in SCR-PLT-023, encrypted); api.v1.push.devices.store and destroy endpoints for portal tokens; push channel in the pipeline honoring preferences and quiet hours; payloads with title and deep link only; token cleanup on unregistered responses and after 60 days idle.
4. decisions.md rows: DEC-312, DEC-313.

DATABASE
partner_api_keys (pak), push_device_registrations (pdr) per P24.0.

BUSINESS RULES
- Partner API is read-only in R3; Restricted fields never returned; masking applies (DEC-312).
- Push payloads carry no personal data (DEC-313).

UI SPEC
- Key creation dialog shows the key once with a copy button and a warning that it cannot be shown again.

SECURITY AND PERMISSIONS
- identity.partner_key.* sensitive with 2FA; keys hashed; revocation immediate; IP allowlist enforced.

TESTS
- Scope enforcement per route; branch scope; revoked and expired keys refused; IP allowlist; rate limit.
- Masking and Restricted field absence in partner responses; isolation through partner keys.
- Partner contract test; push payload content test; token cleanup.

ACCEPTANCE CRITERIA
1. A test integration reads students and attendance with a scoped key on staging; a test device receives a push for a notification event.
2. CI green; staging green.

DO NOT
- Do not add write endpoints to the partner API.
- Do not send personal data in push payloads.

OUTPUT
Report per Master section 20 with the partner endpoint list.
```

### P24.3 Bank APIs, Raast, Additional Languages and Audit Archival

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D02 FR-FEE-035, FR-AUD-008; D07 Part 3 7.26 and 7.27 (gateway tables, bank_statement_lines), Part 4 7.50 (partitioning); D14 P11.1 to P11.3, P12.2, P18.3, DEC-314, DEC-315, DEC-316; the partner bank's official API documentation received under the organization's agreement.

PHASE GOAL
Connect banks directly where a partner bank offers an API: Raast collection with verification-first posting and statement feeds into bank reconciliation; ship the locale registry with one additional language; and archive audit partitions to cold storage before they are dropped.

ALREADY BUILT
Gateway contracts and adapters, bank reconciliation with imports, TranslationParityTest, partition maintenance, P24.0 tables.

TASKS
1. Bank connections: CreateBankApiConnection and TestBankApiConnection on SCR-ACC-023 (encrypted credentials, sandbox or live).
2. Raast collection: RaastPartnerGateway implementing PaymentGateway from the partner bank's official documentation (request to pay or dynamic QR as the partner supports); provider value raast_partner in payment_gateway_accounts; posts only through ConfirmGatewayPayment after verification; reconciliation and settlement reuse P11.3.
3. Statement feed: command accounting:fetch-bank-statements every 2 hours for active connections; lines stored with source api through the same duplicate detection; suggestions appear in SCR-ACC-020.
4. Locales (DEC-315): config/locales.php registry with direction; platform setting platform.enabled_locales on SCR-PLT-024; locale switcher lists enabled locales only; TranslationParityTest runs for every enabled locale; the chosen locale's files completed and natively reviewed; documents render in that locale where templates support it.
5. Audit archival (DEC-316): archive job before each partition drop (NDJSON, encrypted, manifest with checksums, read-back verification); drop blocked if archival fails; financial audit retained 7 years; core:archive-restore into a read-only schema; runbook audit-archive-restore.md.
6. decisions.md rows: DEC-314, DEC-315, DEC-316.

DATABASE
bank_api_connections (bac); bank_statement_lines.source.

BUSINESS RULES
- Raast payments post only after verification (DEC-166, DEC-314).
- A locale is enabled only at 100% parity with native sign-off.
- No partition is dropped without a verified archive.

UI SPEC
- Bank connection status with last sync time and error text; locale list with parity percentage.

SECURITY AND PERMISSIONS
- accounting.bank_connection.manage sensitive with 2FA; credentials never logged; archives encrypted with keys in the secret store.

TESTS
- Raast adapter signature and verification with documented test vectors; payment trust test.
- Statement feed duplicates; locale parity gating; archive, verify and restore of a month; drop blocked on archive failure.

ACCEPTANCE CRITERIA
1. In the partner sandbox a Raast payment posts after verification and a statement feed appears in reconciliation; the additional locale passes parity; an archived month restores.
2. CI green; staging green.

DO NOT
- Do not use unofficial bank interfaces or screen scraping.
- Do not drop partitions without verified archives.

OUTPUT
Report per Master section 20.
```

### P24.4 R3 Hardening and General Availability

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D13 13.6 and 13.7, D12 Part 2 12.22 to 12.24 and 12.31, D14 Part 7 P10.1, P10.2 and P10.4, Part 8 P18.4 (same procedures), DEC-278 (fix steps).

PHASE GOAL
Release R3: every R3 screen covered, security suite and penetration test on R3 surfaces clean, full load set within budgets, runtime and framework within support, release checklist proven, and MS-17 recorded.

ALREADY BUILT
PH-19 to P24.3.

TASKS
1. Repeat P10.1 for R3 scope: ScreenCoverageTest R3 (41 D10 screens plus the 13 added by DEC-305 and DEC-310), D12 12.24 including GPS webhooks, partner API, subscription payments, Raast, offline sync and counseling leak tests, accessibility on R3 key screens (circulation desk, boarding, live map list view, designer, offline banner), translation parity for every enabled locale, load set including GPS ingestion at 500 devices, analytics pages, partner API and offline sync bursts at 08:00.
2. Penetration test on R3 surfaces with fixes as P24.4-fix-NNN steps.
3. RC-7 check: PHP 8.5, Laravel 15 (or Laravel 14 with at least 3 months of security support left if P21.0 is still deferred), MySQL 9.7 LTS.
4. Release checklist RC-1 to RC-10 for R3 with evidence; release notes and guides in English, Urdu and the additional locale; D15 R3 rows signed; tag r3.0.0.
5. decisions.md rows: R3 GA decision.

DATABASE
Index additions only.

BUSINESS RULES
- GA only with every RC item proven.

UI SPEC
- None beyond fixes.

SECURITY AND PERMISSIONS
- No open high or critical findings.

TESTS
- All suites, load set and contract tests green.

ACCEPTANCE CRITERIA
1. MS-17 recorded with the checklist; CI green on the tag; production smoke tests green.

DO NOT
- Do not lower gates to meet the date.

OUTPUT
Report per Master section 20 with the R3 checklist. Then run the D13 13.6 gate for PH-24.
```

---

## Assumptions (Part 9)

| ID | Assumption |
|---|---|
| A-140 | Accountants post asset depreciation, disposal gains or losses and write-offs as manual journals from the inventory reports. Automated asset accounting comes after R3. |
| A-141 | Organizations that want GPS contract a tracker provider that offers an official push or polling API. The platform builds one adapter per provider on demand. |
| A-142 | The platform holds an agreement with a map tile provider. Tile costs are a platform expense that plan pricing covers. |
| A-143 | Offline marking covers daily section attendance only. Period attendance, staff attendance and every other screen need a connection. |
| A-144 | Subscription invoices are in PKR, and the platform's own tax registration decides which tax lines apply. Tax rates are entered as platform settings. |
| A-145 | The product owner picks one additional locale at P24.0, and a native reviewer is available for it. |
| A-146 | The partner API is read-only in R3. Outbound event webhooks and write scopes come after R3. |
| A-147 | Raast and bank statement APIs depend on a partner bank that offers a merchant API to the organization under its own agreement. Organizations without one keep imports. |

## Gaps Found and Added (Part 9)

| ID | Gap | Decision |
|---|---|---|
| GAP-351 | Four table aliases collided across D7 and D14 | DEC-293 and `TableAliasUniquenessTest` |
| GAP-352 | UG-5 (Laravel 15) had no step slot | P21.0 (DEC-294) |
| GAP-353 | P22.0 asked for an addendum for a table that already exists | Step removed (DEC-295) |
| GAP-354 | Library fines paid at the desk had no money path | Ad hoc voucher through Fee (DEC-296) |
| GAP-355 | `library_fines.waiver_approval_request_id` had no approval type | `library_fine_waiver` (DEC-296) |
| GAP-356 | Library stock verification had no storage | Copy verification columns (DEC-297) |
| GAP-357 | Route assignments could use expired licenses or vehicle papers | DEC-298 |
| GAP-358 | Map tile source and GPS throttling were undefined | DEC-299 |
| GAP-359 | `assets.disposal_approval_request_id` had no approval type, and asset accounting was undefined | DEC-300 |
| GAP-360 | Inter-branch stock transfers had no rule | DEC-301 |
| GAP-361 | Restricted counseling sessions had no access model | DEC-302 |
| GAP-362 | Complaint SLA values, reopening and visitor photo retention were undefined | DEC-303 |
| GAP-363 | PH-23 and PH-24 features had no tables or D10 screens | Addenda R3-A and R3-B (DEC-304, DEC-305, DEC-310) |
| GAP-364 | The offline conflict model was undefined | DEC-306 |
| GAP-365 | Template designer safety was undefined | DEC-307 |
| GAP-366 | Timetable generation inputs such as teacher unavailability had no home | Run constraints (DEC-308) |
| GAP-367 | FR-ACC-016 inter-branch transfers had no step | P23.1 with `inter_branch_clearing` |
| GAP-368 | Platform subscription money flow was undefined | DEC-311 |
| GAP-369 | Partner API security was undefined | DEC-312 |
| GAP-370 | Push payload privacy was undefined | DEC-313 |
| GAP-371 | FR-AUD-008 archival had no step | P24.3 (DEC-316) |

Saved to the LMS project as `blueprint/D14-part9-r3-ph19-to-ph24.md`.

Reply CONTINUE for Deliverable 15
