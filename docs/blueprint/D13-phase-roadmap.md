# Deliverable 13: Phase Roadmap

**Version:** 1.0 (draft for approval) | **Date:** 24 Nov 2026
**Builds on:** D1 1.12 (release scope), DEC-01 (runtime policy), D3 3.14 (Phase 0 spike), D7 7.51 and 7.52 (migration batches, seeds), D8 8.5 (module manifest and first release), D10 (screen inventory with release column), D11 (workflows), D12 (security, testing, deployment gates)

---

## 13.1 New Decisions

| ID | Decision |
|---|---|
| DEC-216 | **IDs:** phases are `PH-00` to `PH-24`. Steps are `P{phase}.{n}`, for example `P06.3`. In D14 every step is exactly one Antigravity prompt. D15 tracks progress by the same IDs. |
| DEC-217 | **Phase gates:** a phase starts only after the previous phase passes its gate (13.6). Phases run one after another, because one developer drives one agent (A-01). The only parallel work is the infrastructure track (DEC-225) and external tasks such as the penetration test and provider onboarding. |
| DEC-218 | **Migration timing:** R1 phases follow the D7 batch order M00 to M09. From R2 onward, delivery order follows business priority. Migration files always use their real creation timestamp, and the batch list is treated as dependency order. `MigrationOrderTest` enforces that no migration references a table created later. This replaces the D7 7.51 sentence "timestamps are assigned by batch". |
| DEC-219 | **Split batches:** M10 to M14 mix releases, so they split into sub-batches (13.4). Transport routes and allocations, and discipline incidents, remarks, warnings and rewards, ship in R2, matching D1 1.12 and D8 8.5. Fleet, GPS, library, inventory, counseling and Administration R3 ship in R3. |
| DEC-220 | **UG-1 is step P01.7**, the last step of PH-01, finished by 29 Jan 2027 at the latest (DEC-01 item 5, DEC-199). If PH-01 runs late, UG-1 still runs on 25 Jan 2027 and the remaining PH-01 steps continue on Laravel 13. |
| DEC-221 | **UG-2 (Laravel 14) slot:** step P09.5 if Laravel 14 has at least two point releases by 1 Jul 2027, so R1 hardening tests the final stack. Otherwise step P11.0, the first step of R2. Either way it lands within the DEC-200 limits. |
| DEC-222 | **Pilot before general availability:** 3 organizations (one Scenario A school, one Scenario B chain, and one intermediate college or Cambridge school) run on production for 4 weeks under a pilot plan. Their data is real and stays. |
| DEC-223 | **Buffers and overruns:** each phase duration includes about 20% buffer. If a phase overruns its buffer by more than one week, the product owner moves whole screens and their requirements to the next release and updates the D10 release column. Screens are never shipped half-built or faked (PR-11). |
| DEC-224 | **Schema addenda before PH-22, PH-23 and PH-24:** facility maintenance requests, the document template designer, offline attendance sync, analytics stores, self-service subscription billing and partner API keys have no D7 tables yet. A D7 addendum, approved before each of these phases starts, adds them under the same conventions. |
| DEC-225 | **Infrastructure track:** staging and the deploy pipeline exist from PH-00. Production (D12 12.28) is built in PH-09, with backups, monitoring and runbooks proven before the pilot. The demo environment arrives in PH-18 (GAP-38). |

### Corrections to earlier deliverables

| Where | Change |
|---|---|
| D7 7.51 | Timestamps are real creation times; batch order is dependency order (DEC-218). M10 to M14 split as in 13.4 (DEC-219). |
| D7 7.52 | The permission sync command is `identity:sync-permissions` (D5), not `iam:permissions:sync`. |
| D1 DEC-01 item 5 | Confirmed and given a date: UG-1 is P01.7, by 29 Jan 2027 (DEC-220). |

---

## 13.2 Roadmap at a Glance

Week 1 starts on Monday 30 Nov 2026. Dates are planned end dates and include buffers.

| Phase | Name | Release | Batches | Weeks | Start | End |
|---|---|---|---|---|---|---|
| PH-00 | Bootstrap and spikes | R1 | M00 | 2 | 30 Nov 2026 | 11 Dec 2026 |
| PH-01 | Foundation: tenancy, identity, RBAC, platform, settings, audit | R1 | M01, M02 | 6 | 14 Dec 2026 | 22 Jan 2027 |
| PH-02 | Shared engines: approval, notification, import and export, documents | R1 | M03 | 3 | 25 Jan 2027 | 12 Feb 2027 |
| PH-03 | Academic structure (with Cambridge tracks) | R1 | M04 | 2 | 15 Feb 2027 | 26 Feb 2027 |
| PH-04 | Admission, students, guardians, portals | R1 | M05 | 4 | 1 Mar 2027 | 26 Mar 2027 |
| PH-05 | HR, attendance, leave, devices | R1 | M06 | 4 | 29 Mar 2027 | 23 Apr 2027 |
| PH-06 | Fee and accounting core | R1 | M07 | 6 | 26 Apr 2027 | 4 Jun 2027 |
| PH-07 | Examination, results, external exams | R1 | M08 | 4 | 7 Jun 2027 | 2 Jul 2027 |
| PH-08 | Student lifecycle and session close | R1 | none (uses M04, M05) | 2 | 5 Jul 2027 | 16 Jul 2027 |
| PH-09 | Timetable, communication, administration, reporting, production build | R1 | M09 | 4 | 19 Jul 2027 | 13 Aug 2027 |
| PH-10 | R1 hardening, pilot and launch | R1 | none | 6 | 16 Aug 2027 | 24 Sep 2027 |
| PH-11 | Online fee payments | R2 | M12a | 4 | 27 Sep 2027 | 22 Oct 2027 |
| PH-12 | Accounting R2 | R2 | M11a | 4 | 25 Oct 2027 | 19 Nov 2027 |
| PH-13 | HR R2 and payroll | R2 | M11b, M11c | 5 | 22 Nov 2027 | 24 Dec 2027 |
| PH-14 | Learning (LMS) | R2 | M10a | 4 | 27 Dec 2027 | 21 Jan 2028 |
| PH-15 | Homework and assessment | R2 | M10b, M10c | 5 | 24 Jan 2028 | 25 Feb 2028 |
| PH-16 | Engagement, admissions and exams R2 | R2 | M12b | 5 | 28 Feb 2028 | 31 Mar 2028 |
| PH-17 | Transport routes, discipline, attendance R2 | R2 | M13a, M14a | 4 | 3 Apr 2028 | 28 Apr 2028 |
| PH-18 | Platform R2, API v1, Urdu everywhere, R2 launch | R2 | none | 4 | 1 May 2028 | 26 May 2028 |
| PH-19 | Library | R3 | M13b | 3 | 29 May 2028 | 16 Jun 2028 |
| PH-20 | Transport fleet and GPS | R3 | M13c | 4 | 19 Jun 2028 | 14 Jul 2028 |
| PH-21 | Inventory and assets | R3 | M14b | 3 | 17 Jul 2028 | 4 Aug 2028 |
| PH-22 | Administration R3, complaints, counseling | R3 | M14c | 3 | 7 Aug 2028 | 25 Aug 2028 |
| PH-23 | Analytics, designers, offline attendance | R3 | Addendum (DEC-224) | 6 | 28 Aug 2028 | 6 Oct 2028 |
| PH-24 | SaaS billing, partner API, push, R3 launch | R3 | Addendum (DEC-224) | 6 | 9 Oct 2028 | 17 Nov 2028 |

### Timeline

```
2026      Dec  |PH-00|PH-01 ...........|
2027      Jan  ...PH-01|UG-1|PH-02.....|
          Feb  PH-02|PH-03.....|
          Mar  PH-04 ......................|
          Apr  PH-05 ..............|PH-06 ...
          May  ... PH-06 ..........................
          Jun  PH-06|PH-07 .....................
          Jul  PH-07|PH-08......|PH-09 ..........   (UG-2 slot P09.5)
          Aug  PH-09|PH-10 hardening ... pilot ...
          Sep  ... pilot ...|R1 GA 24 Sep|PH-11 ...
          Oct  PH-11|PH-12 .....................
          Nov  PH-12|PH-13 .....................
          Dec  PH-13 ...........|PH-14 ......
2028      Jan  PH-14|PH-15 .....................
          Feb  PH-15|PH-16 .....................
          Mar  PH-16 ...........................
          Apr  PH-17 ...........................
          May  PH-18 ...............|R2 GA 26 May|PH-19
          Jun  PH-19 (UG-3 MySQL 9.7)|PH-20 ......
          Jul  PH-20|PH-21 .....................
          Aug  PH-21|PH-22 ..........|PH-23 ....
          Sep  PH-23 (UG-4 PHP 8.5) ................
          Oct  PH-23|PH-24 .........................
          Nov  PH-24 ............|R3 GA 17 Nov
```

---

## 13.3 Milestones

| ID | Milestone | Planned date | Proof |
|---|---|---|---|
| MS-01 | Urdu PDF spike passed | 11 Dec 2026 | P00.4 exit criteria met; spike report in `docs/decisions/` |
| MS-02 | RT-1: PHP 8.4 in local, CI and staging | 11 Dec 2026 (deadline 15 Dec) | CI matrix green on 8.2 and 8.4; staging reports PHP 8.4 |
| MS-03 | UG-1: Laravel 13 with PHP 8.4 minimum | 22 Jan 2027 (deadline 29 Jan) | `composer show laravel/framework` 13.x; CI on 8.4 and 8.5 |
| MS-04 | First organization provisioned end to end on staging | 22 Jan 2027 | Owner invited, logs in with OTP, sees onboarding checklist |
| MS-05 | First student admitted and guardian signed in | 26 Mar 2027 | Admission to guardian portal flow on staging |
| MS-06 | First attendance day finalized with device punches | 23 Apr 2027 | ADMS punches resolved with teacher marks |
| MS-07 | First fee voucher batch printed and paid at the counter, journals balanced | 4 Jun 2027 | Ayesha's December voucher example (D11) reproduces exactly |
| MS-08 | First report cards published | 2 Jul 2027 | Pre-rendered report cards in Urdu and English |
| MS-09 | R1 feature complete; production environment ready | 13 Aug 2027 | `ScreenCoverageTest` R1 complete; restore drill passed |
| MS-10 | Pilot starts | 30 Aug 2027 | 3 organizations live |
| MS-11 | **R1 general availability** | 24 Sep 2027 | 13.7 release checklist and D15 signed |
| MS-12 | First online payment posted from a verified callback | 22 Oct 2027 | Provider sandbox, then pilot organization |
| MS-13 | First payroll run finalized | 24 Dec 2027 | Payslips issued; P1 to P6 journals balanced |
| MS-14 | **R2 general availability** | 26 May 2028 | Release checklist and D15 signed |
| MS-15 | UG-3: MySQL 9.7 LTS in production | 16 Jun 2028 | Upgrade runbook completed |
| MS-16 | UG-4: PHP 8.5 in production | 6 Oct 2028 (deadline 31 Dec 2028) | CI and production on 8.5 |
| MS-17 | **R3 general availability** | 17 Nov 2028 | Release checklist and D15 signed |

---

## 13.4 Migration Batches by Phase (DEC-218, DEC-219)

| Batch | Release | Phase | Tables |
|---|---|---|---|
| M00 | R1 | PH-00 | As D7 7.51 |
| M01 | R1 | PH-01 (P01.1) | As D7 7.51 |
| M02 | R1 | PH-01 (P01.2 to P01.6) | As D7 7.51 |
| M03 | R1 | PH-02 | As D7 7.51 |
| M04 | R1 | PH-03 | As D7 7.51 |
| M05 | R1 | PH-04 | As D7 7.51 |
| M06 | R1 | PH-05 | As D7 7.51, plus `section_attendance_sheets` (D11 Part 2 correction) |
| M07 | R1 | PH-06 | As D7 7.51 |
| M08 | R1 | PH-07 | As D7 7.51 |
| M09 | R1 | PH-09 | As D7 7.51 |
| M10a | R2 | PH-14 | Learning tables (D7 7.37) |
| M10b | R2 | PH-15 | Homework tables (D7 7.38) |
| M10c | R2 | PH-15 | Assessment tables (D7 7.39) |
| M11a | R2 | PH-12 | Vendors, expense vouchers, purchase bills, vendor payments, bank reconciliation tables |
| M11b | R2 | PH-13 | HR R2 tables (contracts, reviews, training, HR disciplinary) |
| M11c | R2 | PH-13 | Payroll tables (D7 7.40), including `employee_salary_components` and the tenant use of `payroll_tax_slabs` |
| M12a | R2 | PH-11 | `payment_gateway_accounts`, `payment_gateway_transactions` |
| M12b | R2 | PH-16 | `message_threads`, `messages`, `ptm_*`, `timetable_substitutions`, `gate_passes`, `invigilation_duties`, `scheduled_reports` |
| M13a | R2 | PH-17 | `transport_routes`, `transport_stops`, `route_stops`, `transport_allocations` |
| M13b | R3 | PH-19 | All Library tables (D7 7.41), including `library_book_authors` and `library_reservations` |
| M13c | R3 | PH-20 | `vehicles`, `vehicle_documents`, `driver_profiles`, `transport_route_assignments`, `transport_attendances`, `vehicle_maintenance_logs`, `fuel_logs`, `gps_devices`, `gps_positions` |
| M14a | R2 | PH-17 | `discipline_categories`, `discipline_incidents`, `discipline_incident_students`, `discipline_actions` (warnings), `student_remarks`, `student_achievements` (rewards) |
| M14b | R3 | PH-21 | All Inventory tables (D7 7.43), including `stock_balances` |
| M14c | R3 | PH-22 | `counseling_sessions`, `visitor_logs`, `complaints`, plus the maintenance request table from the D7 addendum (DEC-224) |

Seeders (D7 7.52) arrive with the phase that owns their tables. The platform catalog seeders run on every deploy from PH-01 onward.

---

## 13.5 Phase Details

Each step size: S (1 to 2 days), M (3 to 5 days), L (6 to 8 days). Screen codes refer to D10; workflow sections refer to D11.

### PH-00 Bootstrap and Spikes (R1, M00, 2 weeks)

**Goal:** a working skeleton with CI, the design system base, security middleware, a staging deploy, and proof that Urdu PDFs render correctly.

| Step | Scope | Size |
|---|---|---|
| P00.1 | Repository, Docker Compose (PHP 8.4 app, PHP 8.2 CI image, MySQL 8.4, `redis-cache`, `redis-queue`, Gotenberg 8, Mailpit), Composer and npm setup, Pint, Larastan (`phpVersion: 80200`), Pest 3, CI push and pull request stages (D12 12.27), gitleaks, audits | M |
| P00.2 | Core skeleton: `config/modules.php` with all 34 modules, `ModuleServiceProvider`, `bootstrap/providers.php`, Blueprint macros, architecture tests (`ModuleBoundariesTest`, `DependencyMapTest`, `ModuleLayoutTest`, `ProvidersManifestTest`, `ForbiddenCodeTest`, `NamingTest`, `MigrationOrderTest`, `RouteConventionsTest` skeleton), M00 | M |
| P00.3 | Design system base: `tokens.css` (D9), Tailwind v4, Alpine CSP build, Urdu fonts, layouts (app, guardian, student, platform, public, print, auth), first `x-ui` set (button, input, field, label, card, table, modal, toast, page-header, empty-state, alert, badge), RTL switch, token contrast test, `BladeConventionsTest` | M |
| P00.4 | Urdu PDF spike (D3 3.14): `PdfRenderer`, `GotenbergPdfRenderer`, `FakePdfRenderer`, a 3-copy A4 landscape challan and a report card with Urdu names and remarks, merge route, timing for a 1,000-voucher batch | M |
| P00.5 | Security and operations base: CSP nonce and security headers, correlation ID, JSON logs with the redaction processor, `/up` and `internal.health.ready`, exception mapping (D3 3.19), image build and staging deploy (D12 12.29 steps 1 to 6) | M |

**Exit criteria:** CI green on PHP 8.2 and 8.4; staging serves the empty app over HTTPS. The spike passes: Nastaliq is correctly shaped in both PDFs (checked against reference images by the product owner), the PDF text layer contains the expected Urdu strings, a single render is under 2 s, and the 1,000-voucher batch merges in under 5 minutes on the S1 worker size. **If the spike fails, PH-01 doesn't start; D3 3.14 is revised first.**

### PH-01 Foundation (R1, M01 and M02, 6 weeks)

**Goal:** a secure multi-tenant shell where the platform provisions an organization, users sign in, roles and permissions work, and every change is audited.

| Step | Scope | Size |
|---|---|---|
| P01.1 | Tenancy core and M01: `ResolveTenant`, tenant context, fail-closed `OrganizationScope`, `AllowedBranchesScope`, `BelongsToOrganization`, `TenantAware` jobs, `TenantExists` and `BranchExists` rules, composite foreign keys, `TwoTenantFixture`, generated isolation tests | L |
| P01.2 | Core services: `Money`, `numbering_sequences`, `stored_files` with signed downloads, `file_attachments`, `period_locks`, `idempotency_keys`, `webhook_receipts`, `bulk_operations` with progress screen (SCR-CORE-004, SCR-CORE-005) | M |
| P01.3 | Authentication: users, Fortify per portal, identifiers (DEC-9, DEC-10), OTP, password rules (DEC-188), lockout, organization-aware reset tokens (DEC-96), TOTP 2FA, sessions (DEC-189), trusted devices, `login_events` | L |
| P01.4 | RBAC: roles, permissions, scoped assignments, overrides, visibility levels, `PermissionRegistry`, `identity:sync-permissions`, 22 tenant role templates and 3 platform roles, SOD framework, masking helpers, `RbacMatrixTest`, IAM screens | L |
| P01.5 | Platform and organization: provisioning wizard, plans, entitlements, organization states, subdomains, support access grants and impersonation limits (D12 12.5); organization, institution, branch, department, shift and room screens; branch switcher (PLT, ORG) | L |
| P01.6 | Settings, audit, localization and navigation: `SettingRegistry`, settings screens, branding accent generator with gates G1 to G4, translation overrides, `SetLocale`, audit writer and screens (AUD), `NavigationRegistry`, `{portal}.home` shells with the onboarding checklist widget (SET, AUD) | L |
| P01.7 | UG-1: Laravel 13, `"php": "^8.4"`, Pest 4, CI matrix 8.4 and 8.5, Larastan `phpVersion: 80400`, forbidden-feature list retired except the Eloquent rule (DEC-199) | S |

**Exit criteria:** MS-03 and MS-04 reached. The isolation suite covers every M01 and M02 model, the RBAC matrix is green, D12 12.24 tests are green for authentication, sessions, throttling, audit privilege and headers, and every PLT, ORG, IAM, SET, CORE and AUD screen tagged R1 passes `ScreenCoverageTest`.

### PH-02 Shared Engines (R1, M03, 3 weeks)

**Goal:** the one-engine-per-concern services (PR-09) every later module uses.

| Step | Scope | Size |
|---|---|---|
| P02.1 | Approval engine (D11 Part 1): type and resolver registries, workflows, request snapshots, double validation, `apply_failed`, delegations, inbox and editor screens (APR), `approvals:daily-digest` | L |
| P02.2 | Notification engine (D11 11.27): event registry, English and Urdu templates, rules, preferences, quiet hours, dedupe, one SMS aggregator adapter, email, in-app, delivery receipt webhook, credit caps (NTF) | L |
| P02.3 | Import and export engine (D11 11.28): registries, templates, column mapping, validation, preview, commit through Actions, error files, masked exports with the formula guard; first import type: users (IMX) | M |
| P02.4 | Document engine: type registry, templates, composer, branding, issued documents, verification code fields, batch rendering and merge, pre-render storage (DEC-215) (DOC) | M |

**Exit criteria:** a sample approval type runs end to end with SOD; an SMS reaches the provider sandbox in Urdu with the segment count shown; a users import commits through Actions with audit; a templated PDF renders through the queue.

### PH-03 Academic Structure (R1, M04, 2 weeks)

| Step | Scope | Size |
|---|---|---|
| P03.1 | Sessions, levels, progressions, groups and programs, Cambridge tracks and subject buckets (D6, D6A), subjects | M |
| P03.2 | Classes, sections, class subjects, class teacher history, teaching groups, teaching assignments; onboarding wizard steps for academic defaults (ACD) | M |

**Exit criteria:** Scenario A, Scenario B and a Cambridge O and A Level institution can each be set up from the wizard; session-scoped uniqueness holds.

### PH-04 Admission, Students and Guardians (R1, M05, 4 weeks)

| Step | Scope | Size |
|---|---|---|
| P04.1 | Students, families, guardians, links, documents, timeline, Urdu names, encrypted CNIC and B-Form (DEC-95), GR, admission and roll numbering, enrollments (STU, GRD) | L |
| P04.2 | Admission (D11 Part 2): offline admission, basic public online form, decisions, consent capture (DEC-195), admission slip with guardian verification code (ADM) | M |
| P04.3 | Guardian and student portals: OTP onboarding, child linking and switcher, guardian home, profile correction requests, student login threshold (A-09) | M |
| P04.4 | Imports: students, guardians, elective subject membership (`teaching_group_members`) | M |

**Exit criteria:** MS-05 reached; a 5,000-row student import previews in under 2 minutes; masking tests pass on student and guardian screens and exports.

### PH-05 HR, Attendance and Leave (R1, M06, 4 weeks)

| Step | Scope | Size |
|---|---|---|
| P05.1 | HR R1: designations, employees, teacher profiles, department heads, staff user linking, employee documents, staff import (HR) | M |
| P05.2 | Student attendance (D11 Part 2): `section_attendance_sheets`, marking grid with autosave and conflict banner, finalization, corrections through approval, absence notifications | L |
| P05.3 | Staff attendance and devices: shift timings, grace and late rules, online check-in on trusted devices, QR attendance, ZKTeco ADMS adapter, signed webhook adapter, CSV import, daily resolution and precedence, device registry (ATT) | L |
| P05.4 | Leave for students and staff through the approval engine, balances, attendance integration, attendance period locks, attendance reports (LVE) | M |

**Exit criteria:** MS-06 reached; a 40-student section saves in under 1 s; duplicate punches don't create duplicate rows; a locked month rejects edits without an approved unlock.

### PH-06 Fee and Accounting Core (R1, M07, 6 weeks)

| Step | Scope | Size |
|---|---|---|
| P06.1 | Accounting core: fiscal years, chart of accounts template, bank accounts, journals, the posting service as the only path to journals (RISK-10), system role accounts (DEC-176), accounting period locks (ACC) | M |
| P06.2 | Fee setup: heads, structures, discounts, sibling rules, scholarships, concession categories, sponsored funding (GAP-09), tax lines (GAP-10), late fine rules, security deposits | M |
| P06.3 | Vouchers: monthly, bulk and family vouchers, `VoucherLineContributor`, vacation-month advance with deferred income and `fee:recognize-deferred`, arrears memo lines, pre-rendered 3-copy challans | L |
| P06.4 | Counter collection (D11 Part 1): cashier sessions, cash, bank and cheque payments, receipts, cheque lifecycle, cash close and variance, bank challan entry | L |
| P06.5 | Corrections: reversals, refunds, write-off (`fee_write_off`, `fee.write_off`), concession after issue, `adjustment` method with its reason codes, branch transfer balances, struck-off hooks | M |
| P06.6 | Reports and controls: defaulters, collection, cashbook, ledgers, receivables, reconciliation controls (D11 11.13), opening balance imports, guardian Fees tab (SCR-FEE-030), named tests for every manual-collection posting rule | M |

**Exit criteria:** MS-07 reached; 10,000 vouchers generate in under 10 minutes; mutation score of at least 80% on posting and calculation classes; integrity checks report zero mismatches on the demo dataset.

### PH-07 Examination, Results and External Exams (R1, M08, 4 weeks)

| Step | Scope | Size |
|---|---|---|
| P07.1 | Exam terms, exams, date sheets, admit cards, marks entry grid with submission locks (DEC-181), absent versus zero, grace marks settings (EXM) | L |
| P07.2 | Grading scheme templates, queued calculation, position rules, approval, publication, pre-rendered report cards, revisions, supplementary exams (RES) | L |
| P07.3 | External exams (D6A): awarding bodies, registrations and entries, result import, IBCC conversion (EXT) | M |

**Exit criteria:** MS-08 reached; results for 5,000 students process in under 15 minutes; a published result changes only through an approved revision.

### PH-08 Student Lifecycle and Session Close (R1, no new batch, 2 weeks)

| Step | Scope | Size |
|---|---|---|
| P08.1 | Promotion and retention, session close checks (`SessionCloseCheckRegistry`), fee carry-forward | M |
| P08.2 | Branch and section transfers, withdrawal, struck off and re-admission, clearance checks (`ClearanceCheckRegistry`) | M |
| P08.3 | Leaving and character certificates and other issued documents with verification codes | S |

**Exit criteria:** a full school year can close and the next session opens with every enrollment history kept; a transferred student's balance moves through the D11 branch transfer rules.

### PH-09 Timetable, Communication, Administration, Reporting and Production (R1, M09, 4 weeks)

| Step | Scope | Size |
|---|---|---|
| P09.1 | Bell schedules, timetables, entries, conflict detection, printing (TTB) | M |
| P09.2 | Announcements with audiences and read tracking, daily diary; calendar, holidays, events, closure declarations (GAP-19) with attendance and timetable effects, ID cards (COMM, ADN) | M |
| P09.3 | Reporting: metric registry, snapshot computation, role dashboards and widgets (D10 Part 1), `AttentionItemRegistry`, R1 reports from the D10 report catalog, saved filters, print, PDF and Excel (RPT) | L |
| P09.4 | Infrastructure track: production build (D12 12.28), backups and the first restore drill, monitoring and alerts (D12 12.33), runbooks, freeze calendar | M |
| P09.5 | UG-2 slot (DEC-221): Laravel 14 if the condition holds; otherwise skipped | S |

**Exit criteria:** MS-09 reached; the owner dashboard for 20 branches loads in under 10 s; production passes a restore drill and a replica promotion drill.

### PH-10 R1 Hardening, Pilot and Launch (R1, 6 weeks)

| Step | Scope | Size |
|---|---|---|
| P10.1 | Hardening: the full D12 12.24 list, R1 `ScreenCoverageTest` complete, Urdu complete for the guardian portal and printed documents (A-16), accessibility checks, performance tuning, the full load set (D12 12.22) | L |
| P10.2 | External checks: penetration test and fixes, legal review of privacy notice templates (D12 12.12), DR game day | M (mostly waiting) |
| P10.3 | Pilot (DEC-222): 3 organizations onboarded within 5 working days each (D1 1.18), daily triage, fixes released weekly | 4 weeks |
| P10.4 | General availability: release checklist (13.7) and D15 sign-off | S |

**Exit criteria:** MS-11 reached; zero cross-tenant findings in the penetration test; pilot organizations printed vouchers, collected fees, marked attendance and published results in production.

### PH-11 Online Fee Payments (R2, M12a, 4 weeks)

| Step | Scope | Size |
|---|---|---|
| P11.0 | UG-2, if it wasn't done in P09.5 | S |
| P11.1 | Gateway accounts per organization, test transaction (SCR-FEE-034), 1Bill aggregator adapter with consumer numbers on vouchers | L |
| P11.2 | JazzCash and Easypaisa adapters through official merchant APIs, hosted or redirect flows only (DEC-196) | L |
| P11.3 | Verification-before-posting, store-first webhooks, automated reconciliation, guardian "Pay online" flow, online posting rules with named tests | M |

**Exit criteria:** MS-12 reached; the webhook flood load test shows zero double postings; a browser "success" return never changes a voucher.

### PH-12 Accounting R2 (R2, M11a, 4 weeks)

| Step | Scope | Size |
|---|---|---|
| P12.1 | Vendors, expense vouchers with approval, purchase bills, vendor payments | L |
| P12.2 | Bank reconciliation (statement import and matching) | M |
| P12.3 | Financial statements: trial balance, income statement, balance sheet, cash flow by branch and organization | M |

**Exit criteria:** statements tie out to journals on the demo dataset; every new posting path is covered by `assertJournalBalanced`.

### PH-13 HR R2 and Payroll (R2, M11b and M11c, 5 weeks)

| Step | Scope | Size |
|---|---|---|
| P13.1 | Contracts, performance reviews, training records, HR disciplinary records, onboarding and separation workflow (`employee_separation`) | M |
| P13.2 | Salary components, structures, tax slab entry by platform admin (A-62), statutory deduction types (GAP-34) | M |
| P13.3 | Payroll runs from attendance and leave, advances and loans (`salary_advance`, `employee_loan`), finalization, postings P1 to P6 | L |
| P13.4 | Payslips through the document engine, deduction reports, staff self-service payslip view (PAY) | M |

**Exit criteria:** MS-13 reached; a payroll run reruns identically until it is finalized; finalized runs are immutable.

### PH-14 Learning (R2, M10a, 4 weeks)

| Step | Scope | Size |
|---|---|---|
| P14.1 | Courses, `course_targets`, chapters, lectures, resources with storage caps and external video links | L |
| P14.2 | Student learning view, progress tracking, discussions with moderation (LMS) | M |

### PH-15 Homework and Assessment (R2, M10b and M10c, 5 weeks)

| Step | Scope | Size |
|---|---|---|
| P15.1 | Homework with submissions, grading and guardian visibility (HWK) | M |
| P15.2 | Question bank with reuse across sections (P9 persona) | M |
| P15.3 | Quizzes and online exams: attempts, timing, auto-marking of objective items, manual marking of subjective items (ASM) | L |

### PH-16 Engagement, Admissions and Exams R2 (R2, M12b, 5 weeks)

| Step | Scope | Size |
|---|---|---|
| P16.1 | Guardian-teacher messaging with safeguards (GAP-37), parent-teacher meeting booking | M |
| P16.2 | WhatsApp channel adapter with approved templates | M |
| P16.3 | Substitutions (arrangement periods), gate passes, invigilation duties, scheduled emailed reports (GAP-36) | M |
| P16.4 | Admission R2 (inquiries, follow-ups, admission tests, interviews), board exam support (GAP-32), public document verification page (GAP-35), approval SLA and escalation timers | L |

### PH-17 Transport Routes, Discipline and Attendance R2 (R2, M13a and M14a, 4 weeks)

| Step | Scope | Size |
|---|---|---|
| P17.1 | Routes, stops, route stops, allocations with seat checks, monthly transport charge through `VoucherLineContributor` (TRN) | M |
| P17.2 | Discipline incidents, remarks, warnings and rewards with guardian notifications (DSC) | M |
| P17.3 | On-premise sync agent for pull-only devices, RFID through device adapters | L |

### PH-18 Platform R2, API v1 and R2 Launch (R2, 4 weeks)

| Step | Scope | Size |
|---|---|---|
| P18.1 | Usage metering, storage quotas (GAP-40), plan limit enforcement | M |
| P18.2 | API v1 (D8): Sanctum portal abilities, OpenAPI paths, contract tests, rate limits | L |
| P18.3 | Urdu complete for all portals; in-app help and Urdu guides; demo environment (GAP-38); tenant offboarding export package (GAP-39) | M |
| P18.4 | R2 hardening, load set, penetration test delta, release checklist | M |

**Exit criteria:** MS-14 reached.

### PH-19 Library (R3, M13b, 3 weeks)

| Step | Scope | Size |
|---|---|---|
| P19.1 | Catalog, authors, copies, circulation, reservations, fines through `VoucherLineContributor`, clearance check (LIB) | L |
| P19.2 | UG-3: MySQL 9.7 LTS (DEC-201) | S |

### PH-20 Transport Fleet and GPS (R3, M13c, 4 weeks)

| Step | Scope | Size |
|---|---|---|
| P20.1 | Vehicles, vehicle documents with expiry reminders, driver profiles, route assignments, transport attendance | L |
| P20.2 | Maintenance and fuel logs linked to expense vouchers; GPS adapter, positions, Leaflet and OpenStreetMap route view | L |

### PH-21 Inventory and Assets (R3, M14b, 3 weeks)

| Step | Scope | Size |
|---|---|---|
| P21.1 | Categories, items, stock locations, movements, `stock_balances`, purchase receipts from purchase bills, issues, transfers, write-offs with approval, asset register (INV) | L |

### PH-22 Administration R3, Complaints and Counseling (R3, M14c, 3 weeks)

| Step | Scope | Size |
|---|---|---|
| P22.0 | D7 addendum for maintenance requests approved (DEC-224) | S |
| P22.1 | Visitor logs, maintenance requests, complaints with SLA (GAP-44) | M |
| P22.2 | Counseling sessions with encrypted notes and Restricted classification (D12 12.3) | M |

### PH-23 Analytics, Designers and Offline Attendance (R3, schema addendum, 6 weeks)

| Step | Scope | Size |
|---|---|---|
| P23.0 | D7 addendum approved (DEC-224) | S |
| P23.1 | Result analytics, branch comparison analytics, advanced consolidation | L |
| P23.2 | Document and certificate template designer | L |
| P23.3 | Assisted timetable generation | M |
| P23.4 | Offline-capable attendance marking (GAP-41) with conflict-safe sync | L |
| P23.5 | UG-4: PHP 8.5 (D12 12.31) | S |

### PH-24 SaaS Billing, Partner API, Push and R3 Launch (R3, schema addendum, 6 weeks)

| Step | Scope | Size |
|---|---|---|
| P24.0 | D7 addendum approved (DEC-224) | S |
| P24.1 | Self-service subscription billing (GAP-01 R3 part) | L |
| P24.2 | Public partner API with keys and scopes; push notifications (FCM) for future apps | M |
| P24.3 | Bank-specific APIs and Raast through a partner bank adapter; additional languages | M |
| P24.4 | R3 hardening, load set, penetration test, release checklist | M |

**Exit criteria:** MS-17 reached.

---

## 13.6 Phase Gate (applies to every phase)

| # | Check | Evidence |
|---|---|---|
| G-1 | Every step's acceptance criteria from its D14 prompt are met | D15 checklist rows ticked with links to commits |
| G-2 | CI green on every stage, including nightly | CI run links |
| G-3 | Isolation tests cover every new tenant model and route | Isolation suite report |
| G-4 | RBAC matrix updated for new permissions; `identity:sync-permissions` run on staging | Test report, staging log |
| G-5 | Every new screen passes `ScreenCoverageTest` for its release | Test report |
| G-6 | English and Urdu keys complete (`TranslationParityTest`) | Test report |
| G-7 | Coverage targets met (D12 12.23) | Coverage report |
| G-8 | Performance budgets met for the phase's flows on staging | k6 or timing report |
| G-9 | Decisions added to `docs/decisions/decisions.md`; runbooks updated if operations changed | Diff |
| G-10 | Deployed to staging and demonstrated to the product owner with the phase demo script | Demo notes, sign-off |
| G-11 | No open high-severity bugs from this or earlier phases | Issue tracker filter |

---

## 13.7 Release Checklist (R1, R2, R3)

| # | Check |
|---|---|
| RC-1 | All phase gates of the release passed |
| RC-2 | Every D10 screen tagged for the release exists and passes `ScreenCoverageTest` (or was moved under DEC-223 with an updated release column) |
| RC-3 | D12 12.24 security tests green; penetration test has no open high or critical findings |
| RC-4 | Full load set within budgets (D12 12.22) on the production-like staging |
| RC-5 | Restore drill, single-tenant restore and replica promotion passed within 30 days |
| RC-6 | Monitoring, alerts, status page and on-call rota live |
| RC-7 | Runtime and framework within support (D12 12.31) with at least 3 months left |
| RC-8 | Privacy notice template and data processing terms reviewed by a lawyer |
| RC-9 | Release notes in English and Urdu; user guides for new screens |
| RC-10 | D15 verification checklist signed by the product owner |

---

## 13.8 Upgrade and Lifecycle Gates on the Roadmap

| Gate | What | Slot | Deadline |
|---|---|---|---|
| RT-1 | PHP 8.4 in local, CI and staging | PH-00 | 15 Dec 2026 |
| UG-1 | Laravel 13, PHP 8.4 minimum, Pest 4 | P01.7 | 29 Jan 2027 (Laravel 12 security ends 24 Feb 2027) |
| UG-2 | Laravel 14 | P09.5 or P11.0 (DEC-221) | Before Laravel 13 security ends (17 Mar 2028) |
| UG-3 | MySQL 9.7 LTS | P19.2 | Before April 2029 |
| UG-4 | PHP 8.5 | P23.5 | Before 31 Dec 2028 |
| UG-5 | Laravel 15 | First phase that starts at least 2 months after its second point release | Under DEC-200 |

Every gate follows the `upgrade-gate.md` runbook (D12 12.35).

---

## 13.9 Dependencies and Critical Path

```
PH-00 -> PH-01 -> PH-02 -> PH-03 -> PH-04 -> PH-05 -> PH-06 -> PH-07 -> PH-08 -> PH-09 -> PH-10 (R1)
                     |                 |         |        |
                     |                 |         |        +-> PH-11 (needs vouchers, M07)
                     |                 |         |        +-> PH-12 (needs accounting core)
                     |                 |         +-> PH-13 (needs HR, attendance, leave, accounting)
                     |                 +-> PH-14, PH-15 (need academic structure and students)
                     +-> every later module uses the engines

Critical path for R1: tenancy and RBAC (PH-01) -> engines (PH-02) -> students (PH-04)
                      -> fees (PH-06) -> results (PH-07) -> hardening and pilot (PH-10)
External lead times:  SMS aggregator account (needed by PH-02), pilot recruitment (by PH-08),
                      penetration tester booking (by PH-09), payment merchant onboarding for
                      pilot organizations (start during PH-10 for PH-11)
```

---

## 13.10 Roadmap Risks

| ID | Risk | Mitigation |
|---|---|---|
| RR-01 | PH-01 overruns and squeezes the UG-1 deadline | DEC-220: UG-1 runs on 25 Jan 2027 regardless |
| RR-02 | The Urdu PDF spike fails | PH-01 is blocked until D3 3.14 is revised (P00.4 exit rule) |
| RR-03 | The agent produces shallow features | One step per prompt, gate G-1 to G-11, D15 rows per step |
| RR-04 | Provider sandbox access is slow | External lead times started early (13.9); fakes keep development moving |
| RR-05 | Pilot organizations not recruited in time | Recruit from PH-06 demos; pilot may start with 2 organizations, with GA moving 2 weeks |
| RR-06 | Laravel 14 timing unknown | DEC-221 handles both cases |
| RR-07 | Holidays and Ramadan reduce capacity | Absorbed by the 20% phase buffers (A-105) |
| RR-08 | Fee module complexity exceeds 6 weeks | DEC-223: move whole lower-priority reports to R2, never core posting |

---

## Assumptions

| ID | Assumption |
|---|---|
| A-104 | One full-time developer drives Antigravity five days a week; the product owner is available weekly for gate demos. |
| A-105 | Public holidays, Ramadan and summer vacation effects fit inside the phase buffers. |
| A-106 | Three pilot organizations agree to join by the end of PH-08. |
| A-107 | Provider sandboxes (SMS aggregator, 1Bill aggregator, JazzCash, Easypaisa) are available when PH-02 and PH-11 start. |
| A-108 | Laravel 14 and 15 keep the annual first-quarter release pattern, but the plan doesn't depend on exact dates. |

## Gaps Found and Added

| ID | Gap | Decision |
|---|---|---|
| GAP-267 | No phase or step IDs to connect prompts and tracking | DEC-216 |
| GAP-268 | No phase gate criteria | DEC-217, 13.6 |
| GAP-269 | "Timestamps by batch" blocked priority-based delivery in R2 and R3 | DEC-218 |
| GAP-270 | Batches M13 and M14 mixed R2 and R3 tables, against D1 and D8 | DEC-219, 13.4 |
| GAP-271 | No pilot stage before launch | DEC-222 |
| GAP-272 | No rule for schedule overruns | DEC-223 |
| GAP-273 | R3 features without tables (maintenance requests, designer, billing, offline sync, analytics, partner API keys) | DEC-224 |
| GAP-274 | No infrastructure timeline | DEC-225 |
| GAP-275 | UG-2 to UG-5 not placed on the plan | 13.8 |
| GAP-276 | No release-level checklist | 13.7 |

---

Reply CONTINUE for Deliverable 14
