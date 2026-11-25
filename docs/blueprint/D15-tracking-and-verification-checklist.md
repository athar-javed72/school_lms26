# Deliverable 15: Tracking and Verification Checklist

**Version:** 1.0 (draft for approval) | **Date:** 25 Nov 2026
**Builds on:** D01 1.16 and RISK-03, D02 (all FRs), D06A A.9 (FR-EXT), D10 (screen inventory, 10.50 coverage checks), D12 Part 2 12.22 to 12.24 and 12.31, D13 (13.3 milestones, 13.6 phase gate, 13.7 release checklist, 13.8 upgrade gates, DEC-216, DEC-223), D14 Parts 1 to 9 (94 steps, DEC-228 step reports, DEC-278 fix steps, DEC-293 to DEC-316)
**Where it lives:** `docs/checklists/d15-tracker.md` in the repository (DEC-226), with the machine-readable copy `docs/checklists/d15-steps.csv` that CI reads (DEC-320).

**At a glance**

| Item | R1 | R2 | R3 | Total |
|---|---|---|---|---|
| D14 steps (excluding fix steps) | 47 | 28 | 19 | 94 |
| Phases with a gate | 11 (PH-00 to PH-10) | 8 (PH-11 to PH-18) | 6 (PH-19 to PH-24) | 25 |
| Functional requirements traced | 313 | 95 | 33 | 441 |
| Screens to cover (D10 plus D14 additions) | 343 | 118 | 54 | 515 |
| Milestones | MS-01 to MS-11 | MS-12 to MS-14 | MS-15 to MS-17 | 17 |

The 54 R3 screens are the 41 D10 R3 screens plus the 13 added by D14 DEC-305 and DEC-310.

---

## 15.0 New Decisions

| ID | Decision |
|---|---|
| DEC-317 | **This checklist is the single record of progress.** A step, phase, milestone or release is done only when its row here is Done with evidence. Chat messages, agent claims and commit messages are not evidence on their own. This applies the brief's "No Unverified Completion" rule. |
| DEC-318 | **Row statuses.** `Not started`, `In progress`, `In review` (agent report submitted, reviewer checking), `Done`, `Blocked` (reason and blocker link required), `Skipped` (only the conditional steps P11.0 and P21.0, with the linked reason), `Deferred` (a screen or step moved under DEC-223 or DEC-294, with the new release or phase and the product owner decision link). No other values are allowed. |
| DEC-319 | **Who ticks a row.** The agent never sets `Done`. It sets `In review` and attaches its report. The reviewer (tech lead for steps and fix steps, product owner for phase gates, milestones and release sign-off) runs the universal verification checklist (15.2), checks every acceptance criterion against its evidence, and only then sets `Done` with their name and date. A reviewer cannot tick a step they wrote code for. |
| DEC-320 | **Checklist as code.** `docs/checklists/d15-steps.csv` holds one line per step and fix step (id, status, pr, report, ci_run, verified_by, verified_on). The CI job `ChecklistConsistencyTest` fails when: a D14 step ID has no row; a row is `Done` without an existing `docs/agent/reports/{id}.md`; a `Done` step has an open high-severity fix step; a screen in 15.11 for a released version is missing from `ScreenCoverageTest`; or an FR in 15.10 lists a screen ID that does not exist. The markdown tracker is regenerated from the CSV by `php artisan checklist:render`. |
| DEC-321 | **ScreenCoverageTest exceptions** live in `tests/Fixtures/screen_coverage_exceptions.php`. Every entry carries an exception ID (`EX-NN`), the screen or permission key, the reason, the compensating test, the owner, the release in which it must be closed or reviewed, and the 15.12 reference. An entry without all fields fails the test. New exceptions need tech lead and product owner approval recorded in 15.12. |
| DEC-322 | **Fix step rows** follow DEC-278: ID `{parent}-fix-NNN`, numbered from 001 per parent. Each row records the source (pen test, pilot, bug bash, phase gate, production incident), severity, the regression test that proves the fix, and the commit. A parent step stays `Done` while low or medium fix rows are open, but a phase gate cannot pass with an open high or critical fix row (G-11). |
| DEC-323 | **Release sign-off.** For each release, the product owner signs 15.13 only when every step row of the release is `Done`, `Skipped` or `Deferred` with links, every phase gate of the release is closed, every RC item in 15.8 has evidence, every FR of the release in 15.10 is ticked or formally deferred, and every open exception in 15.12 has a decision. The signature records name, date and the release tag. |
| DEC-324 | **Requirements without a building step get one.** Building the 15.10 matrix showed three R2 requirements with no D14 task. They are added to D14 Part 8, which is re-uploaded: FR-ADM-015 (anonymize rejected applications) as P16.4 task 7, FR-ATT-025 (push user lists to devices) as P17.3 task 5, and FR-IMX-010 (undo an import batch) as P18.3 task 4. FR-IAM-016 is tagged R1 in D02 but is delivered in P17.3 and P18.2 under DEC-283, so it is recorded as Deferred in 15.13. FR-ACD-015 is tagged R2 but is already met in R1 by P03.1 (`education_levels.is_external_exam_level`) and P07.3 (centre numbers per awarding body). |

### Corrections to earlier deliverables

| Where | Change |
|---|---|
| D14 Part 8 P16.4 | Task 7 anonymization of rejected applications and its test (FR-ADM-015, DEC-324) |
| D14 Part 8 P17.3 | Task 5 push user lists to devices and its test (FR-ATT-025, DEC-324) |
| D14 Part 8 P18.3 | Task 4 import undo and its test (FR-IMX-010, DEC-324) |
| D02 FR-IAM-016 | Release R1 delivered in R2 (P17.3, P18.2) per DEC-283; listed as Deferred in 15.13 |

---

## 15.1 How to Use This Checklist

| # | Who | Action |
|---|---|---|
| 1 | Tech lead | Set the step row to `In progress` and paste the Phase 0 Master Context Prompt plus the step prompt into a new Antigravity conversation (D14 "How to run"). |
| 2 | Agent | Build the step, open a pull request, and write `docs/agent/reports/{id}.md` in the Master section 20 format. |
| 3 | Tech lead | Set the row to `In review`. Run 15.2 against the pull request and the report. Every acceptance criterion must show PASS with evidence (test name, command output, screenshot path). |
| 4 | Tech lead | If anything fails, return the step to the agent with the failing items, or open fix steps (15.5) for findings outside the step's acceptance criteria. |
| 5 | Tech lead | Merge only when CI is green. Fill PR, report and CI run links. Set `Done` with name and date. Update `d15-steps.csv` in the same pull request. |
| 6 | Tech lead | At the end of a phase, collect gate evidence in 15.6 and run the phase demo for the product owner. |
| 7 | Product owner | Sign the phase gate row (G-10 demo and the sign-off column). Record milestones in 15.7 when their proof exists. |
| 8 | Product owner | At release time, confirm 15.8, review 15.12 exceptions and sign 15.13. |

---

## 15.2 Universal Step Verification Checklist

The reviewer checks every item for every step and fix step. An item that does not apply is marked n/a with a one-line reason in the review comment.

| # | Check | How to verify |
|---|---|---|
| V-01 | Every acceptance criterion in the step prompt is met | Report lists each criterion with PASS and evidence; reviewer reproduces at least one on staging |
| V-02 | CI green on every stage for the merge commit | CI run link; no skipped or quarantined tests added by this step |
| V-03 | `composer check` passes (Pint, Larastan at the configured level, Pest parallel); `npm run build` within bundle budgets | CI job output |
| V-04 | Tenant isolation: every new tenant model, route, export, job and file path is in the isolation suite and passes | Isolation suite report shows the new entries |
| V-05 | Branch scoping and visibility levels applied in queries, not only in views | Code review of Queries; isolation and RBAC tests |
| V-06 | Permissions: new keys in `Definitions/Permissions.php`, synced, covered by the RBAC matrix fixture, enforced in the backend (policy or action authorization) | RBAC fixture diff; test that a role without the key gets 403 |
| V-07 | Screens match D10 exactly: screen ID, route name, permission, roles, release | `ScreenCoverageTest` entries; screenshots of each new screen in English and Urdu |
| V-08 | Empty, filtered-empty, loading and error states exist on new lists and pages (D9 9.16) | Screenshots or browser test |
| V-09 | English and Urdu keys complete (`TranslationParityTest`); RTL checked at 360px for portal screens | Test output; screenshot |
| V-10 | Audit: every write on an audited model writes an audit entry; protected changes need a reason; Restricted values never appear in audit, logs or exceptions | Audit tests; `LogRedactionTest`; spot check of a log line |
| V-11 | No passwords, OTPs, tokens or provider credentials in logs, audit, exports or error reports | Redaction tests; grep of test logs for the fixture secrets |
| V-12 | Money: journals only through JournalPoster; named posting tests with `assertJournalBalanced`; no payment status from client-side data; gapless numbering where required | Posting tests; payment trust test where payments are touched |
| V-13 | No hardcoded business values (fees, grades, roles, approval chains, tax rates, thresholds, holidays); they are data or settings | Code review; arch test for literals in Actions where configured |
| V-14 | No fake functionality: no stub screens, placeholder data, "coming soon" buttons, TODO markers in shipped paths, or fake adapters outside tests and demo | `grep -rn "TODO\|FIXME\|coming soon\|lorem" Modules resources` returns nothing new; fake adapters bound only in testing and demo |
| V-15 | Historical records are never destroyed: no hard deletes of business records; corrections are reversals or new versions | Code review; immutability tests |
| V-16 | Migrations follow D7 names and aliases (`TableAliasUniquenessTest`), and after R1 GA are expand-only (D12 12.29) | Migration review; test output |
| V-17 | Performance: new flows within D12 12.17 budgets; no N+1 (lazy loading disabled outside production) | Timing or k6 report for the step's flows |
| V-18 | Security tests from D12 12.24 relevant to the step are added and green (uploads, webhooks, throttling, headers, impersonation limits) | Test list in the report |
| V-19 | `docs/decisions/decisions.md` updated for every DEC, A and GAP the step applies for the first time; runbooks updated if operations changed | Diff |
| V-20 | Report complete in the Master section 20 format, including conflicts, stopped items and next step notes | Report file |
| V-21 | Upgrade and lifecycle steps only (P01.7, P09.5, P11.0, P19.2, P21.0, P23.5): no feature changes in the same pull request; before and after screenshots; production watch recorded | Pull request diff; report |

---

## 15.3 Status Summary (update weekly)

| Release | Steps Done | Steps In progress or In review | Blocked | Deferred or Skipped | Open high fix steps | Gates closed | FRs ticked |
|---|---|---|---|---|---|---|---|
| R1 | 0 / 47 | 0 | 0 | 0 | 0 | 0 / 11 | 0 / 313 |
| R2 | 0 / 28 | 0 | 0 | 0 | 0 | 0 / 8 | 0 / 95 |
| R3 | 0 / 19 | 0 | 0 | 0 | 0 | 0 / 6 | 0 / 33 |

---

## 15.4 Step Tracker

One row per D14 step. Status values and the evidence rule are in DEC-318 and DEC-319. "Proof" is acceptance criterion 1 of the step prompt; every other criterion in the prompt must also be shown in the report. Fix steps are added under their parent step in 15.5 as they arise.

### PH-00 Bootstrap and Spikes (R1)

| Step | Title | Proof (acceptance criterion 1) | Status | PR / commit | Report | CI run | Verified by, date |
|---|---|---|---|---|---|---|---|
| P00.1 | Repository, Docker and CI | docker compose -f deploy/docker/compose.yaml up -d starts every service; php -v inside app shows 8.4.x. | Not started | | `docs/agent/reports/P00.1.md` | | |
| P00.2 | Core Skeleton, Module Manifest and Architecture Guards | php artisan about lists 2 app providers plus 34 module providers in manifest order. | Not started | | `docs/agent/reports/P00.2.md` | | |
| P00.3 | Design System Foundation | npm run build and npm run build:check pass; bundle sizes reported. | Not started | | `docs/agent/reports/P00.3.md` | | |
| P00.4 | Urdu PDF Spike | Both spike PDFs and PNG screenshots exist in docs/agent/reports/assets/P00.4/. | Not started | | `docs/agent/reports/P00.4.md` | | |
| P00.5 | Security Base, Operations Base and Staging Deploy | Merge to main builds, scans, signs and deploys to staging automatically; smoke.sh passes; ZAP baseline shows no high findings. | Not started | | `docs/agent/reports/P00.5.md` | | |
| PH-00 gate | D13 13.6 G-1 to G-11 (record in 15.6) | All rows above Done or Skipped with evidence; demo signed | Not started | | `docs/agent/reports/PH-00-gate.md` | | |

### PH-01 Foundation (R1)

| Step | Title | Proof (acceptance criterion 1) | Status | PR / commit | Report | CI run | Verified by, date |
|---|---|---|---|---|---|---|---|
| P01.1 | Tenancy Core and M01 Tables | php artisan migrate:fresh creates M00 and M01 tables; MigrationOrderTest passes with deferred FKs. | Not started | | `docs/agent/reports/P01.1.md` | | |
| P01.2 | Users, Audit Foundation and Core Service Tables | Migrations M02 (users, sessions, audit, Core) apply after M01 with MigrationOrderTest green. | Not started | | `docs/agent/reports/P01.2.md` | | |
| P01.3 | RBAC: Roles, Permissions, Assignments and Policies | RbacMatrixTest and all D05 5.14 test groups applicable to declared permissions pass. | Not started | | `docs/agent/reports/P01.3.md` | | |
| P01.4 | Authentication, Sessions and Two-Factor | Staff sign-in, OTP activation, password reset and 2FA flows work end to end on staging (log or email delivery), with screenshots in English and Urdu. | Not started | | `docs/agent/reports/P01.4.md` | | |
| P01.5 | Platform and Organization Management | MS-04 on staging: a platform admin provisions an organization; the owner receives the invitation (email or log), activates with OTP, signs in on {slug}.staging.darsgah.app, and lands in the onboarding wizard. Screenshots of each step in the report. | Not started | | `docs/agent/reports/P01.5.md` | | |
| P01.6 | Settings, Audit Screens, Localization, Navigation and Portal Homes | An owner signs in on staging and lands on app.home with the onboarding checklist, branch scope line and correct navigation; a platform admin lands on platform.home with platform widgets. Screenshots in English and Urdu. | Not started | | `docs/agent/reports/P01.6.md` | | |
| P01.7 | UG-1: Upgrade to Laravel 13 | composer show laravel/framework reports 13.x; php artisan about shows Laravel 13 on PHP 8.4 in staging. | Not started | | `docs/agent/reports/P01.7.md` | | |
| PH-01 gate | D13 13.6 G-1 to G-11 (record in 15.6) | All rows above Done or Skipped with evidence; demo signed | Not started | | `docs/agent/reports/PH-01-gate.md` | | |

### PH-02 Shared Engines (R1)

| Step | Title | Proof (acceptance criterion 1) | Status | PR / commit | Report | CI run | Verified by, date |
|---|---|---|---|---|---|---|---|
| P02.1 | Approval Engine | period_unlock works end to end on staging with screenshots of W-07 in English and Urdu. | Not started | | `docs/agent/reports/P02.1.md` | | |
| P02.2 | Notification Engine | On staging with the SMS aggregator sandbox: an OTP SMS in Urdu arrives (or the sandbox log shows it) with 2 Unicode segments counted correctly; a test send works from SCR-NTF-007. | Not started | | `docs/agent/reports/P02.2.md` | | |
| P02.3 | Import and Export Engine | Users import end to end on staging with an error file for bad rows and invitations sent for created users. | Not started | | `docs/agent/reports/P02.3.md` | | |
| P02.4 | Document Engine | A test certificate issues with serial, QR verification code and footer on staging; reissue and revoke work. | Not started | | `docs/agent/reports/P02.4.md` | | |
| PH-02 gate | D13 13.6 G-1 to G-11 (record in 15.6) | All rows above Done or Skipped with evidence; demo signed | Not started | | `docs/agent/reports/PH-02-gate.md` | | |

### PH-03 Academic Structure (R1)

| Step | Title | Proof (acceptance criterion 1) | Status | PR / commit | Report | CI run | Verified by, date |
|---|---|---|---|---|---|---|---|
| P03.1 | Sessions, Levels, Groups, Subjects and Seed Templates | On staging, a national school institution and a Cambridge institution each receive their template with correct levels, groups and subjects visible in English and Urdu. | Not started | | `docs/agent/reports/P03.1.md` | | |
| P03.2 | Classes, Sections, Teaching Groups, Session Wizard and Onboarding | Scenario A, Scenario B (two branches) and a Cambridge O and A Level institution are set up on staging from the onboarding wizard, with screenshots of classes, sections and class subjects. | Not started | | `docs/agent/reports/P03.2.md` | | |
| PH-03 gate | D13 13.6 G-1 to G-11 (record in 15.6) | All rows above Done or Skipped with evidence; demo signed | Not started | | `docs/agent/reports/PH-03-gate.md` | | |

### PH-04 Admission, Students and Guardians (R1)

| Step | Title | Proof (acceptance criterion 1) | Status | PR / commit | Report | CI run | Verified by, date |
|---|---|---|---|---|---|---|---|
| P04.1 | Students, Guardians, Families and Enrollments | On staging, a student is created, enrolled with electives, linked to two guardians (one primary) and shown on W-02 in English and Urdu with masked B-Form. | Not started | | `docs/agent/reports/P04.1.md` | | |
| P04.2 | Admission | On staging, an online application with OTP is submitted, reviewed, approved and converted; the guardian receives the activation SMS and the slip prints with a verification code. | Not started | | `docs/agent/reports/P04.2.md` | | |
| P04.3 | Guardian and Student Portals | On staging, a guardian activates by OTP, switches between two children in different branches and raises a profile correction that reaches the principal. | Not started | | `docs/agent/reports/P04.3.md` | | |
| P04.4 | Student, Guardian and Teaching Group Imports | A 5,000-row student file with guardians previews in under 2 minutes and commits on staging; activation SMS batch sent after completion. | Not started | | `docs/agent/reports/P04.4.md` | | |
| PH-04 gate | D13 13.6 G-1 to G-11 (record in 15.6) | All rows above Done or Skipped with evidence; demo signed | Not started | | `docs/agent/reports/PH-04-gate.md` | | |

### PH-05 HR, Attendance and Leave (R1)

| Step | Title | Proof (acceptance criterion 1) | Status | PR / commit | Report | CI run | Verified by, date |
|---|---|---|---|---|---|---|---|
| P05.1 | HR Core, Teaching Assignments and Class Teachers | On staging, employees are imported, teachers assigned through the matrix, class teachers set, and a teacher sees "My classes" with only their students. | Not started | | `docs/agent/reports/P05.1.md` | | |
| P05.2 | Student Attendance | On staging, a class teacher marks a 40-student section on a phone, submits, and an absent child's guardian receives the Urdu SMS after the delay. | Not started | | `docs/agent/reports/P05.2.md` | | |
| P05.3 | Staff Attendance and Devices | On staging, a ZKTeco device (or the ADMS simulator script in tools/adms-sim) pushes punches that resolve into the staff today screen within 5 minutes. | Not started | | `docs/agent/reports/P05.3.md` | | |
| P05.4 | Leave, Attendance Period Locks and Reports | On staging, a guardian applies for 2 days of child leave, the class teacher approves, and the sheet shows LV rows locked with the reason. | Not started | | `docs/agent/reports/P05.4.md` | | |
| PH-05 gate | D13 13.6 G-1 to G-11 (record in 15.6) | All rows above Done or Skipped with evidence; demo signed | Not started | | `docs/agent/reports/PH-05-gate.md` | | |

### PH-06 Fee and Accounting Core (R1)

| Step | Title | Proof (acceptance criterion 1) | Status | PR / commit | Report | CI run | Verified by, date |
|---|---|---|---|---|---|---|---|
| P06.1 | Accounting Core | On staging, the finance onboarding step creates the fiscal year and chart; a manual journal above the threshold is approved by the owner and appears in the ledger and trial balance. | Not started | | `docs/agent/reports/P06.1.md` | | |
| P06.2 | Fee Setup | On staging, a branch has structures for national and Cambridge classes with vacation advance; the calculator preview for a family of three shows correct sibling discounts. | Not started | | `docs/agent/reports/P06.2.md` | | |
| P06.3 | Vouchers | On staging, December vouchers generate for a branch with family vouchers, print as 3-copy challans and appear in the guardian portal. | Not started | | `docs/agent/reports/P06.3.md` | | |
| P06.4 | Counter Collection | On staging, a cashier opens a session, records cash, cheque and bank deposit payments with printed receipts, clears one cheque, bounces another, and closes with a variance that the accountant approves. | Not started | | `docs/agent/reports/P06.4.md` | | |
| P06.5 | Corrections: Reversals, Refunds, Write-offs, Adjustments and Installments | On staging, a payment reversal passes approval and reopens vouchers with the reversal journal dated today; a deposit refund is approved and paid; a write-off passes principal and owner. | Not started | | `docs/agent/reports/P06.5.md` | | |
| P06.6 | Fee Reports, Ledgers and Reconciliation Controls | On staging with demo:seed-finance, core:integrity-check reports zero mismatches. | Not started | | `docs/agent/reports/P06.6.md` | | |
| PH-06 gate | D13 13.6 G-1 to G-11 (record in 15.6) | All rows above Done or Skipped with evidence; demo signed | Not started | | `docs/agent/reports/PH-06-gate.md` | | |

### PH-07 Examination, Results and External Exam Foundations (R1)

| Step | Title | Proof (acceptance criterion 1) | Status | PR / commit | Report | CI run | Verified by, date |
|---|---|---|---|---|---|---|---|
| P07.1 | Exams, Date Sheets, Admit Cards and Marks Entry | On staging, a term exam is set up for national and Cambridge classes, the date sheet is published, admit cards print with fee blocks, marks are entered, submitted, returned once and approved. | Not started | | `docs/agent/reports/P07.1.md` | | |
| P07.2 | Results, Publication and Report Cards | On staging, a term result is processed for national and Cambridge classes, made stale by a revision, reprocessed, approved, published, and report cards download from the guardian portal. | Not started | | `docs/agent/reports/P07.2.md` | | |
| P07.3 | External Exam Foundations | On staging, a Cambridge branch has centre numbers, syllabus codes for O Level subjects, candidate numbers for O3 students and approved predicted grades shown on the report card. | Not started | | `docs/agent/reports/P07.3.md` | | |
| PH-07 gate | D13 13.6 G-1 to G-11 (record in 15.6) | All rows above Done or Skipped with evidence; demo signed | Not started | | `docs/agent/reports/PH-07-gate.md` | | |

### PH-08 Student Lifecycle and Session Close (R1)

| Step | Title | Proof (acceptance criterion 1) | Status | PR / commit | Report | CI run | Verified by, date |
|---|---|---|---|---|---|---|---|
| P08.1 | Promotion, Graduation, Conditional Promotion and Session Close | On staging, a branch promotes all classes into the planning session, graduates the final level, handles two conditional promotions (one cleared, one reverted) and closes the old session through the full checklist. | Not started | | `docs/agent/reports/P08.1.md` | | |
| P08.2 | Transfers, Withdrawal, Struck Off, Readmission and Suspension | On staging, one student transfers between branches with keep_in_source and another with carry_to_target; balances and postings match D11 11.9. | Not started | | `docs/agent/reports/P08.2.md` | | |
| P08.3 | Certificates | On staging, a leaving certificate is issued after withdrawal with bilingual dates in words; a reissue creates a new serial. | Not started | | `docs/agent/reports/P08.3.md` | | |
| PH-08 gate | D13 13.6 G-1 to G-11 (record in 15.6) | All rows above Done or Skipped with evidence; demo signed | Not started | | `docs/agent/reports/PH-08-gate.md` | | |

### PH-09 Timetable, Communication, Administration, Reporting and Production (R1)

| Step | Title | Proof (acceptance criterion 1) | Status | PR / commit | Report | CI run | Verified by, date |
|---|---|---|---|---|---|---|---|
| P09.1 | Timetable and Period Attendance | On staging, a branch builds a conflict-free timetable for a morning shift, activates it, and teachers, guardians and students see their timetables; a college section uses period attendance with daily derivation. | Not started | | `docs/agent/reports/P09.1.md` | | |
| P09.2 | Communication, Calendar, Closures and ID Cards | On staging, a smog closure is declared for two branches mid-morning: open sheets show H, no absence SMS go out, guardians receive the closure message, and the calendar shows the closure. | Not started | | `docs/agent/reports/P09.2.md` | | |
| P09.3 | Reporting, Dashboards and Attention Items | On staging, the owner, principal, class teacher, accountant, cashier, exam controller, HR manager, admission officer, guardian and student dashboards render per D10 10.8 with live and snapshot widgets. | Not started | | `docs/agent/reports/P09.3.md` | | |
| P09.4 | Production Build, Partitions, Backups and Monitoring | Production is live with no tenants, monitored, alerting to the on-call rota, and the status page up. | Not started | | `docs/agent/reports/P09.4.md` | | |
| P09.5 | UG-2 Slot: Laravel 14 | Either Laravel 14 runs on staging with all gates green, or ug-2-deferred.md records why it was deferred to P11.0. **Conditional: if deferred, record "Deferred to P11.0" with the ug-2-deferred.md link.** | Not started | | `docs/agent/reports/P09.5.md` | | |
| PH-09 gate | D13 13.6 G-1 to G-11 (record in 15.6) | All rows above Done or Skipped with evidence; demo signed | Not started | | `docs/agent/reports/PH-09-gate.md` | | |

### PH-10 R1 Hardening, Pilot and Launch (R1)

| Step | Title | Proof (acceptance criterion 1) | Status | PR / commit | Report | CI run | Verified by, date |
|---|---|---|---|---|---|---|---|
| P10.1 | Hardening | RC-2, RC-3 (internal part) and RC-4 evidence complete. | Not started | | `docs/agent/reports/P10.1.md` | | |
| P10.2 | External Checks: Penetration Test, Legal Review and DR Game Day | Penetration test report received; every high and critical finding closed with fix steps merged and retested; zero cross-tenant findings. | Not started | | `docs/agent/reports/P10.2.md` | | |
| P10.3 | Pilot | Three organizations live within five working days each. | Not started | | `docs/agent/reports/P10.3.md` | | |
| P10.4 | General Availability | RC-1 to RC-10 proven with links; D15 signed by the product owner; MS-11 reached on or before 24 Sep 2027 or a new date recorded with reasons. | Not started | | `docs/agent/reports/P10.4.md` | | |
| PH-10 gate | D13 13.6 G-1 to G-11 (record in 15.6) | All rows above Done or Skipped with evidence; demo signed | Not started | | `docs/agent/reports/PH-10-gate.md` | | |

### PH-11 Online Fee Payments (R2)

| Step | Title | Proof (acceptance criterion 1) | Status | PR / commit | Report | CI run | Verified by, date |
|---|---|---|---|---|---|---|---|
| P11.0 | UG-2 Slot (if not done in P09.5) | Production runs Laravel 14 with all gates green, or a dated blocker record approved by the platform owner exists. **Conditional: mark "Skipped" with the P09.5 report link if UG-2 finished in P09.5.** | Not started | | `docs/agent/reports/P11.0.md` | | |
| P11.1 | Gateway Accounts and 1Bill | In the aggregator sandbox, a voucher is inquired and paid through the test bank app flow and the receipt appears with balanced journals. | Not started | | `docs/agent/reports/P11.1.md` | | |
| P11.2 | JazzCash and Easypaisa Hosted Payments | In both providers' sandboxes a guardian pays a voucher, sees "Verifying", then success with a receipt; journals balanced. | Not started | | `docs/agent/reports/P11.2.md` | | |
| P11.3 | Reconciliation, Webhook Flood and Online Posting Rules | Webhook flood evidence with zero double postings; payment trust tests green (PH-11 exit criteria). | Not started | | `docs/agent/reports/P11.3.md` | | |
| PH-11 gate | D13 13.6 G-1 to G-11 (record in 15.6) | All rows above Done or Skipped with evidence; demo signed | Not started | | `docs/agent/reports/PH-11-gate.md` | | |

### PH-12 Accounting R2 (R2)

| Step | Title | Proof (acceptance criterion 1) | Status | PR / commit | Report | CI run | Verified by, date |
|---|---|---|---|---|---|---|---|
| P12.1 | Vendors, Expenses, Purchases and Vendor Payments | On staging an expense and a purchase pass approval and post; a vendor payment settles the bill. | Not started | | `docs/agent/reports/P12.1.md` | | |
| P12.2 | Bank Reconciliation | A month's statement for one bank reconciles to zero on staging. | Not started | | `docs/agent/reports/P12.2.md` | | |
| P12.3 | Financial Statements | Statements tie out on the demo dataset; every new posting path has assertJournalBalanced coverage (PH-12 exit criteria). | Not started | | `docs/agent/reports/P12.3.md` | | |
| PH-12 gate | D13 13.6 G-1 to G-11 (record in 15.6) | All rows above Done or Skipped with evidence; demo signed | Not started | | `docs/agent/reports/PH-12-gate.md` | | |

### PH-13 HR R2 and Payroll (R2)

| Step | Title | Proof (acceptance criterion 1) | Status | PR / commit | Report | CI run | Verified by, date |
|---|---|---|---|---|---|---|---|
| P13.1 | HR R2 | A separation passes principal and owner with clearance and deactivates the user on the last day on staging. | Not started | | `docs/agent/reports/P13.1.md` | | |
| P13.2 | Salary Components, Structures and Tax Slabs | A test organization configures structures and salaries; previews match hand-calculated fixtures. | Not started | | `docs/agent/reports/P13.2.md` | | |
| P13.3 | Payroll Runs, Advances, Loans and Postings | On staging a branch's payroll is calculated, reviewed, approved, finalized and paid with a bank upload file; P1 and P2 journals balanced. | Not started | | `docs/agent/reports/P13.3.md` | | |
| P13.4 | Payslips, Deduction Reports and Self-Service | MS-13: first payroll run finalized with payslips issued and P1 to P6 journals balanced. | Not started | | `docs/agent/reports/P13.4.md` | | |
| PH-13 gate | D13 13.6 G-1 to G-11 (record in 15.6) | All rows above Done or Skipped with evidence; demo signed | Not started | | `docs/agent/reports/PH-13-gate.md` | | |

### PH-14 Learning (LMS) (R2)

| Step | Title | Proof (acceptance criterion 1) | Status | PR / commit | Report | CI run | Verified by, date |
|---|---|---|---|---|---|---|---|
| P14.1 | Courses, Chapters, Lectures and Resources | A course with mixed lecture types is built, targeted and published on staging. | Not started | | `docs/agent/reports/P14.1.md` | | |
| P14.2 | Student Learning, Progress and Discussions | Phase exit criteria evidence; CI green; staging green. | Not started | | `docs/agent/reports/P14.2.md` | | |
| PH-14 gate | D13 13.6 G-1 to G-11 (record in 15.6) | All rows above Done or Skipped with evidence; demo signed | Not started | | `docs/agent/reports/PH-14-gate.md` | | |

### PH-15 Homework and Assessment (R2)

| Step | Title | Proof (acceptance criterion 1) | Status | PR / commit | Report | CI run | Verified by, date |
|---|---|---|---|---|---|---|---|
| P15.1 | Homework | Homework assigned, submitted twice, reviewed with resubmission on staging. | Not started | | `docs/agent/reports/P15.1.md` | | |
| P15.2 | Question Bank | A department bank with all types, approved and tagged, on staging. | Not started | | `docs/agent/reports/P15.2.md` | | |
| P15.3 | Quizzes and Online Exams | Phase exit criteria evidence; CI green; staging green. | Not started | | `docs/agent/reports/P15.3.md` | | |
| PH-15 gate | D13 13.6 G-1 to G-11 (record in 15.6) | All rows above Done or Skipped with evidence; demo signed | Not started | | `docs/agent/reports/PH-15-gate.md` | | |

### PH-16 Engagement, Admissions and Exams R2 (R2)

| Step | Title | Proof (acceptance criterion 1) | Status | PR / commit | Report | CI run | Verified by, date |
|---|---|---|---|---|---|---|---|
| P16.1 | Messaging and Parent-Teacher Meetings | Safeguard tests green; a PTM runs end to end on staging. | Not started | | `docs/agent/reports/P16.1.md` | | |
| P16.2 | WhatsApp Channel | In a test WhatsApp Business account, an absence alert template reaches a consenting test guardian and a non-consenting one receives SMS. | Not started | | `docs/agent/reports/P16.2.md` | | |
| P16.3 | Substitutions, Gate Passes, Invigilation and Scheduled Reports | Each feature works end to end on staging. | Not started | | `docs/agent/reports/P16.3.md` | | |
| P16.4 | Admission R2, Document Verification and Approval Escalation | Each feature works on staging; escalation fires within 15 minutes of SLA breach. | Not started | | `docs/agent/reports/P16.4.md` | | |
| P16.5 | External Exam Workflows, Board Support and Result Rechecks | A June series runs from entries to published results on staging for an O Level class, with fees remitted and certificates recorded. | Not started | | `docs/agent/reports/P16.5.md` | | |
| PH-16 gate | D13 13.6 G-1 to G-11 (record in 15.6) | All rows above Done or Skipped with evidence; demo signed | Not started | | `docs/agent/reports/PH-16-gate.md` | | |

### PH-17 Transport Routes, Discipline and Attendance R2 (R2)

| Step | Title | Proof (acceptance criterion 1) | Status | PR / commit | Report | CI run | Verified by, date |
|---|---|---|---|---|---|---|---|
| P17.1 | Transport Routes and Allocations | Allocated students get transport lines on the next vouchers on staging. | Not started | | `docs/agent/reports/P17.1.md` | | |
| P17.2 | Discipline | Incident to suspension flow on staging with SU attendance. | Not started | | `docs/agent/reports/P17.2.md` | | |
| P17.3 | Sync Agent and Device Token API | A pull-only device on a test LAN delivers punches through the agent with exactly-once storage; phase exit criteria evidence. | Not started | | `docs/agent/reports/P17.3.md` | | |
| PH-17 gate | D13 13.6 G-1 to G-11 (record in 15.6) | All rows above Done or Skipped with evidence; demo signed | Not started | | `docs/agent/reports/PH-17-gate.md` | | |

### PH-18 Platform R2, API v1 and R2 Launch (R2)

| Step | Title | Proof (acceptance criterion 1) | Status | PR / commit | Report | CI run | Verified by, date |
|---|---|---|---|---|---|---|---|
| P18.1 | Usage Metering, Storage Quotas and Plan Limits | Limits enforced on staging; CI green. | Not started | | `docs/agent/reports/P18.1.md` | | |
| P18.2 | API v1 | Contract test green; API isolation and masking tests green; CI green. | Not started | | `docs/agent/reports/P18.2.md` | | |
| P18.3 | Urdu Everywhere, Help, Demo Environment and Tenant Export | Export of a test organization restores into an empty database with matching counts; CI green. | Not started | | `docs/agent/reports/P18.3.md` | | |
| P18.4 | R2 Hardening and General Availability | MS-14 recorded with the checklist; CI green on the tag; production smoke tests green. | Not started | | `docs/agent/reports/P18.4.md` | | |
| PH-18 gate | D13 13.6 G-1 to G-11 (record in 15.6) | All rows above Done or Skipped with evidence; demo signed | Not started | | `docs/agent/reports/PH-18-gate.md` | | |

### PH-19 Library (R3)

| Step | Title | Proof (acceptance criterion 1) | Status | PR / commit | Report | CI run | Verified by, date |
|---|---|---|---|---|---|---|---|
| P19.1 | Catalog, Copies, Members, Circulation, Reservations and Fines | On staging a book is catalogued, copies labelled, a student issues and returns late, the fine appears on the next voucher, and a staff fine appears on the next payslip. | Not started | | `docs/agent/reports/P19.1.md` | | |
| P19.2 | UG-3: MySQL 9.7 LTS | Production runs MySQL 9.7 LTS with all gates green, the restore drill passed, and MS-15 recorded. | Not started | | `docs/agent/reports/P19.2.md` | | |
| PH-19 gate | D13 13.6 G-1 to G-11 (record in 15.6) | All rows above Done or Skipped with evidence; demo signed | Not started | | `docs/agent/reports/PH-19-gate.md` | | |

### PH-20 Transport Fleet and GPS (R3)

| Step | Title | Proof (acceptance criterion 1) | Status | PR / commit | Report | CI run | Verified by, date |
|---|---|---|---|---|---|---|---|
| P20.1 | Vehicles, Documents, Drivers, Route Assignments and Boarding | On staging a vehicle with valid documents and a licensed driver is assigned, capacity follows the vehicle, and boarding marks notify a guardian on drop. | Not started | | `docs/agent/reports/P20.1.md` | | |
| P20.2 | Maintenance, Fuel, GPS and Live Map | With the fake provider on staging, a moving vehicle shows on the live map and on the guardian page during the trip window only; a fuel log creates an expense voucher that posts after approval. | Not started | | `docs/agent/reports/P20.2.md` | | |
| PH-20 gate | D13 13.6 G-1 to G-11 (record in 15.6) | All rows above Done or Skipped with evidence; demo signed | Not started | | `docs/agent/reports/PH-20-gate.md` | | |

### PH-21 Inventory and Assets (R3)

| Step | Title | Proof (acceptance criterion 1) | Status | PR / commit | Report | CI run | Verified by, date |
|---|---|---|---|---|---|---|---|
| P21.0 | UG-5: Laravel 15 (conditional slot) | Production runs Laravel 15 with all gates green, or a dated deferral record exists. **Conditional: may run at a later phase start under the same ID (DEC-294); record where it ran.** | Not started | | `docs/agent/reports/P21.0.md` | | |
| P21.1 | Items, Stock Ledger, Receipts, Issues, Transfers, Adjustments and Assets | On staging stock is received from a purchase bill, issued, transferred to another branch and adjusted through approval; assets are registered, assigned and disposed through approval; integrity check clean. | Not started | | `docs/agent/reports/P21.1.md` | | |
| PH-21 gate | D13 13.6 G-1 to G-11 (record in 15.6) | All rows above Done or Skipped with evidence; demo signed | Not started | | `docs/agent/reports/PH-21-gate.md` | | |

### PH-22 Administration R3, Complaints and Counseling (R3)

| Step | Title | Proof (acceptance criterion 1) | Status | PR / commit | Report | CI run | Verified by, date |
|---|---|---|---|---|---|---|---|
| P22.1 | Visitor Logs, Maintenance Requests and Complaints | On staging a guardian complaint breaches SLA and escalates, then resolves and is rated; a maintenance request runs from report to resolution with an expense voucher; a visitor checks in and out. | Not started | | `docs/agent/reports/P22.1.md` | | |
| P22.2 | Counseling Sessions | The leak test suite is green and a counselor records, follows up and refers sessions on staging. | Not started | | `docs/agent/reports/P22.2.md` | | |
| PH-22 gate | D13 13.6 G-1 to G-11 (record in 15.6) | All rows above Done or Skipped with evidence; demo signed | Not started | | `docs/agent/reports/PH-22-gate.md` | | |

### PH-23 Analytics, Designers and Offline Attendance (R3)

| Step | Title | Proof (acceptance criterion 1) | Status | PR / commit | Report | CI run | Verified by, date |
|---|---|---|---|---|---|---|---|
| P23.0 | D7 Addendum R3-A | Addendum approved and recorded; migrations deployed to staging and production with no behaviour change. **Addendum approval recorded (approver, date) before P23.1 starts.** | Not started | | `docs/agent/reports/P23.0.md` | | |
| P23.1 | Result Analytics, Branch Comparison and Advanced Consolidation | On staging analytics match fixture calculations and consolidated statements eliminate a transfer. | Not started | | `docs/agent/reports/P23.1.md` | | |
| P23.2 | Document and Certificate Template Designer | On staging an admin designs and publishes a leaving certificate layout, issues one, publishes another version, and the first certificate reprints unchanged. | Not started | | `docs/agent/reports/P23.2.md` | | |
| P23.3 | Assisted Timetable Generation | On the demo school the generator produces a draft that passes the conflict detector and places at least 95% of lessons within the time limit. | Not started | | `docs/agent/reports/P23.3.md` | | |
| P23.4 | Offline-Capable Attendance Marking | Phase-level evidence: offline marks sync without overwriting newer marks or finalized sheets on staging. | Not started | | `docs/agent/reports/P23.4.md` | | |
| P23.5 | UG-4: PHP 8.5 | Production and the sync agent run PHP 8.5; MS-16 recorded. | Not started | | `docs/agent/reports/P23.5.md` | | |
| PH-23 gate | D13 13.6 G-1 to G-11 (record in 15.6) | All rows above Done or Skipped with evidence; demo signed | Not started | | `docs/agent/reports/PH-23-gate.md` | | |

### PH-24 SaaS Billing, Partner API, Push and R3 Launch (R3)

| Step | Title | Proof (acceptance criterion 1) | Status | PR / commit | Report | CI run | Verified by, date |
|---|---|---|---|---|---|---|---|
| P24.0 | D7 Addendum R3-B | Addendum approved and recorded; migrations deployed with no behaviour change. **Addendum approval and chosen locale recorded before P24.1 starts.** | Not started | | `docs/agent/reports/P24.0.md` | | |
| P24.1 | Self-Service Subscription Billing | In provider sandboxes a trial organization subscribes, upgrades, pays an invoice online, misses a payment, is suspended to read-only, pays and is reactivated. | Not started | | `docs/agent/reports/P24.1.md` | | |
| P24.2 | Partner API and Push Notifications | A test integration reads students and attendance with a scoped key on staging; a test device receives a push for a notification event. | Not started | | `docs/agent/reports/P24.2.md` | | |
| P24.3 | Bank APIs, Raast, Additional Languages and Audit Archival | In the partner sandbox a Raast payment posts after verification and a statement feed appears in reconciliation; the additional locale passes parity; an archived month restores. | Not started | | `docs/agent/reports/P24.3.md` | | |
| P24.4 | R3 Hardening and General Availability | MS-17 recorded with the checklist; CI green on the tag; production smoke tests green. | Not started | | `docs/agent/reports/P24.4.md` | | |
| PH-24 gate | D13 13.6 G-1 to G-11 (record in 15.6) | All rows above Done or Skipped with evidence; demo signed | Not started | | `docs/agent/reports/PH-24-gate.md` | | |

## 15.5 Fix Step Register (DEC-278, DEC-322)

Fix steps are opened as findings arrive and use the D14 14.21 fix prompt template with all twelve sections. The expected sources are fixed in advance so nothing is missed:

| Parent step | Expected source of fix steps | ID range |
|---|---|---|
| P10.1 | R1 bug bash and hardening findings | `P10.1-fix-001` onward |
| P10.2 | R1 penetration test, legal review, DR game day | `P10.2-fix-001` onward |
| P10.3 | Pilot triage with the three pilot organizations (DEC-222) | `P10.3-fix-001` onward |
| P18.4 | R2 penetration test delta and hardening | `P18.4-fix-001` onward |
| P24.4 | R3 penetration test and hardening | `P24.4-fix-001` onward |
| Any step | Phase gate review or production incident after release | `{step}-fix-001` onward on the step that owns the code |

Register (add one row per fix step; keep closed rows):

| Fix ID | Parent | Source and reference | Severity | Summary of the defect | Regression test | Status | PR / commit | Report | Verified by, date |
|---|---|---|---|---|---|---|---|---|---|

Severity follows D12 Part 1 (critical, high, medium, low). A critical or high fix row blocks its phase gate (G-11) and the release (RC-3 for security findings).

## 15.6 Phase Gate Record (D13 13.6)

Each cell holds the evidence link for that check, or "n/a" with a reason. A phase is closed only when all eleven cells are filled and the product owner has signed the last column.

| Phase | G-1 | G-2 | G-3 | G-4 | G-5 | G-6 | G-7 | G-8 | G-9 | G-10 | G-11 | Product owner sign-off |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| PH-00 Bootstrap and Spikes | | | | | | | | | | | | |
| PH-01 Foundation | | | | | | | | | | | | |
| PH-02 Shared Engines | | | | | | | | | | | | |
| PH-03 Academic Structure | | | | | | | | | | | | |
| PH-04 Admission, Students and Guardians | | | | | | | | | | | | |
| PH-05 HR, Attendance and Leave | | | | | | | | | | | | |
| PH-06 Fee and Accounting Core | | | | | | | | | | | | |
| PH-07 Examination, Results and External Exam Foundations | | | | | | | | | | | | |
| PH-08 Student Lifecycle and Session Close | | | | | | | | | | | | |
| PH-09 Timetable, Communication, Administration, Reporting and Production | | | | | | | | | | | | |
| PH-10 R1 Hardening, Pilot and Launch | | | | | | | | | | | | |
| PH-11 Online Fee Payments | | | | | | | | | | | | |
| PH-12 Accounting R2 | | | | | | | | | | | | |
| PH-13 HR R2 and Payroll | | | | | | | | | | | | |
| PH-14 Learning (LMS) | | | | | | | | | | | | |
| PH-15 Homework and Assessment | | | | | | | | | | | | |
| PH-16 Engagement, Admissions and Exams R2 | | | | | | | | | | | | |
| PH-17 Transport Routes, Discipline and Attendance R2 | | | | | | | | | | | | |
| PH-18 Platform R2, API v1 and R2 Launch | | | | | | | | | | | | |
| PH-19 Library | | | | | | | | | | | | |
| PH-20 Transport Fleet and GPS | | | | | | | | | | | | |
| PH-21 Inventory and Assets | | | | | | | | | | | | |
| PH-22 Administration R3, Complaints and Counseling | | | | | | | | | | | | |
| PH-23 Analytics, Designers and Offline Attendance | | | | | | | | | | | | |
| PH-24 SaaS Billing, Partner API, Push and R3 Launch | | | | | | | | | | | | |

## 15.7 Milestone Tracker (D13 13.3)

| ID | Milestone | Planned date | Delivering step | Proof required | Status | Evidence link | Recorded by, date |
|---|---|---|---|---|---|---|---|
| MS-01 | Urdu PDF spike passed | 11 Dec 2026 | P00.4 | P00.4 exit criteria met; spike report in `docs/decisions/` | Not started | | |
| MS-02 | RT-1: PHP 8.4 in local, CI and staging | 11 Dec 2026 (deadline 15 Dec) | P00.5 | CI matrix green on 8.2 and 8.4; `internal.health.ready` on staging reports 8.4.x | Not started | | |
| MS-03 | UG-1: Laravel 13 with PHP 8.4 minimum | 22 Jan 2027 (deadline 29 Jan) | P01.7 | `composer show laravel/framework` 13.x; CI on 8.4 and 8.5 | Not started | | |
| MS-04 | First organization provisioned end to end on staging | 22 Jan 2027 | P01.5 | Owner invited, logs in with OTP, sees onboarding checklist | Not started | | |
| MS-05 | First student admitted and guardian signed in | 26 Mar 2027 | P04.3 | Admission to guardian portal flow on staging | Not started | | |
| MS-06 | First attendance day finalized with device punches | 23 Apr 2027 | P05.3 | ADMS punches resolved with teacher marks | Not started | | |
| MS-07 | First voucher batch printed and paid at the counter, journals balanced | 4 Jun 2027 | P06.4 | D11 December voucher example reproduces exactly | Not started | | |
| MS-08 | First report cards published | 2 Jul 2027 | P07.2 | Pre-rendered report cards in Urdu and English | Not started | | |
| MS-09 | R1 feature complete; production environment ready | 13 Aug 2027 | P09.4 and P10.1 | `ScreenCoverageTest` R1 complete; restore drill passed | Not started | | |
| MS-10 | Pilot starts | 30 Aug 2027 | P10.3 | 3 organizations live | Not started | | |
| MS-11 | R1 general availability | 24 Sep 2027 | P10.4 | 13.7 checklist and D15 R1 sign-off (15.13) | Not started | | |
| MS-12 | First online payment posted from a verified callback | 22 Oct 2027 | P11.3 | Provider sandbox, then a pilot organization | Not started | | |
| MS-13 | First payroll run finalized | 24 Dec 2027 | P13.4 | Payslips issued; P1 to P6 journals balanced | Not started | | |
| MS-14 | R2 general availability | 26 May 2028 | P18.4 | Release checklist and D15 R2 sign-off | Not started | | |
| MS-15 | UG-3: MySQL 9.7 LTS in production | 16 Jun 2028 | P19.2 | Upgrade runbook completed; restore drill on 9.7 | Not started | | |
| MS-16 | UG-4: PHP 8.5 in production | 6 Oct 2028 (deadline 31 Dec 2028) | P23.5 | CI and production on 8.5; sync agent rebuilt | Not started | | |
| MS-17 | R3 general availability | 17 Nov 2028 | P24.4 | Release checklist and D15 R3 sign-off | Not started | | |

A milestone that slips past its planned date gets a new date and the reason in this row, and the critical path note in D13 13.9 is reviewed. Deadlines in brackets are hard limits tied to vendor support and cannot move.

---

## 15.8 Release Checklist Record (D13 13.7)

| # | Check | R1 evidence | R2 evidence | R3 evidence |
|---|---|---|---|---|
| RC-1 | All phase gates of the release passed (15.6) | PH-00 to PH-10 | PH-11 to PH-18 | PH-19 to PH-24 |
| RC-2 | Every D10 screen tagged for the release exists and passes `ScreenCoverageTest`, or was moved under DEC-223 (15.11, 15.12) | 343 screens | 118 screens | 54 screens |
| RC-3 | D12 12.24 security tests green; penetration test has no open high or critical findings (15.5) | P10.2 | P18.4 | P24.4 |
| RC-4 | Full load set within budgets on production-like staging (D12 12.22) | P10.1 | P18.4 | P24.4 |
| RC-5 | Restore drill, single-tenant restore and replica promotion passed within 30 days | P10.2 | P18.4 | P24.4 (and P19.2 drill on 9.7) |
| RC-6 | Monitoring, alerts, status page and on-call rota live | P09.4 | P18.4 | P24.4 |
| RC-7 | Runtime and framework within support with at least 3 months left (D12 12.31) | PHP 8.4, Laravel 13 or 14 | PHP 8.4, Laravel 14 | PHP 8.5, Laravel 15 (or 14 per DEC-294), MySQL 9.7 |
| RC-8 | Privacy notice template and data processing terms reviewed by a lawyer | P10.2 | Delta review for R2 data (payments, messaging, WhatsApp) | Delta review for R3 data (GPS, counseling, visitors, partner API) |
| RC-9 | Release notes in English and Urdu; user guides for new screens | P10.4 | P18.4 | P24.4 (plus the additional locale) |
| RC-10 | D15 verification checklist signed by the product owner (15.13) | | | |

Each evidence cell is replaced by the link to the proof when the release is prepared. The cells above name the step that produces the proof.

---

## 15.9 Upgrade and Lifecycle Gate Tracker (D13 13.8, D12 12.31)

| Gate | What | Step | Deadline | Status | Evidence (runbook record, CI, production watch) |
|---|---|---|---|---|---|
| RT-1 | PHP 8.4 in local, CI and staging | P00.1 and P00.5 | 15 Dec 2026 | Not started | |
| UG-1 | Laravel 13, PHP 8.4 minimum, Pest 4 | P01.7 | 29 Jan 2027 | Not started | |
| UG-2 | Laravel 14 | P09.5 or P11.0 (DEC-221) | Before 17 Mar 2028 | Not started | |
| UG-3 | MySQL 9.7 LTS | P19.2 | Before April 2029 | Not started | |
| UG-4 | PHP 8.5 | P23.5 | Before 31 Dec 2028 | Not started | |
| UG-5 | Laravel 15 | P21.0 or the first later qualifying phase start (DEC-294) | Before Laravel 14 security fixes end | Not started | |

Every gate follows `docs/runbooks/upgrade-gate.md`. A gate row is Done only with the production watch result and the post-upgrade restore or smoke evidence.

## 15.10 FR-to-Screen Traceability Matrix

Every functional requirement in D02 and D06A maps to the screens that satisfy it and the step that first delivers it. "Also in" lists the other steps that build or extend the listed screens. "*" means the requirement applies to every screen of its kind and is proven by the named arch or feature test instead of one screen. "-" means the requirement has no screen of its own and is proven by the named test. The column "Verified" is ticked when the screens' tests and the named test are green in the release CI run.

### FR-GEN

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-GEN-001 | Every list screen has server-side pagination (default 25, max 100 rows), search, filters, sort, and filters kept in the URL | R1 | * | P00.3 |  | ListScreenConventionsTest on every list screen | ☐ |
| FR-GEN-002 | Every list and report can export the current filtered result to Excel, CSV, PDF and print view, gated by `{module}.{resource}.export` / `.print`. Exports over 2,000 rows are queued | R1 | SCR-IMX-005, SCR-RPT-003, SCR-STU-023 | P02.3 | P04.1, P06.6 | ExportPermissionTest; queued export over 2,000 rows | ☐ |
| FR-GEN-003 | Every request checks, in order: authentication, tenant scope, branch scope, module entitlement, policy | R1 | * | P01.3 |  | RouteConventionsTest; middleware order test | ☐ |
| FR-GEN-004 | Users with several branches get a branch context switcher. Organization-scope roles can choose "All branches" for read screens only; every write needs a specific branch | R1 | SCR-IAM-007 | P01.6 | P04.3 | BranchContextTest (All branches read-only) | ☐ |
| FR-GEN-005 | Every write on an audited model creates an audit entry (AUD) | R1 | SCR-AUD-001, SCR-AUD-003 | P01.2 | P01.6 | AuditCoverageTest | ☐ |
| FR-GEN-006 | Changes to protected records require a reason, stored with the audit entry | R1 | SCR-STU-004, SCR-AUD-002 | P01.2 | P01.6, P04.1 | ProtectedChangeReasonTest | ☐ |
| FR-GEN-007 | All UI text uses translation keys. English and Urdu (RTL). Western digits by default | R1 | SCR-SET-004 | P00.3 | P01.6 | TranslationParityTest (G-6) | ☐ |
| FR-GEN-008 | Uploads: whitelisted types, size limits by plan, private storage, served only through short-lived signed URLs after a policy check | R1 | SCR-STU-005, SCR-HR-008 | P01.2 | P04.1, P05.1 | UploadSecurityTest; signed URL expiry | ☐ |
| FR-GEN-009 | Master data uses soft delete and restore (`restore` permission). Transactional records are never deleted after finalization | R1 | * | P00.2 |  | SoftDeleteAndImmutabilityTest | ☐ |
| FR-GEN-010 | Edit forms use optimistic locking: saving a record changed by someone else since it was opened shows a conflict message instead of overwriting | R1 | SCR-STU-003, SCR-HR-003 | P01.2 | P04.1, P05.1 | OptimisticLockConflictTest | ☐ |
| FR-GEN-011 | Every screen has defined empty, loading and error states (D9) | R1 | * | P00.3 |  | Screen state tests (empty, filtered-empty, loading, error) | ☐ |
| FR-GEN-012 | A module not entitled for the organization is hidden in navigation and its routes and API return 404 | R1 | SCR-PLT-008 | P01.5 |  | EntitlementHiddenModuleTest (404) | ☐ |
| FR-GEN-013 | Payments, attendance ingestion and bulk actions accept an idempotency key; a repeated submission returns the first result instead of creating a duplicate | R1 | SCR-FEE-017, SCR-ATT-002, SCR-CORE-004 | P01.2 | P01.6, P05.2, P06.4 | IdempotencyKeyTest | ☐ |
| FR-GEN-014 | Bulk operations run as queued jobs with a progress indicator, a result summary, and an in-app notification on completion | R1 | SCR-CORE-004, SCR-CORE-005 | P01.6 |  | BulkOperationProgressTest | ☐ |
| FR-GEN-015 | Writes into a locked period are rejected unless an approved unlock exists (CORE period locks) | R1 | SCR-CORE-001, SCR-CORE-002 | P01.6 |  | PeriodLockWriteRefusedTest | ☐ |
| FR-GEN-016 | WCAG 2.2 AA: keyboard navigation, labels, contrast, focus states | R1 | * | P10.1 |  | axe-core gate on 30 key screens (DEC-279) | ☐ |
| FR-GEN-017 | Guardian, student and teacher screens work on a 360px wide screen | R1 | SCR-RPT-008, SCR-RPT-009, SCR-ATT-002 | P10.1 | P05.2, P09.3 | Browser tests at 360px | ☐ |
| FR-GEN-018 | Print views use organization and institution branding | R1 | SCR-SET-004, SCR-DOC-002 | P02.4 | P01.6 | Print view branding snapshot test | ☐ |
| FR-GEN-019 | Records outside the tenant return 404 (DEC-27) | R1 | * | P01.1 |  | Isolation suite (404 for foreign records) | ☐ |

### FR-CORE

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-CORE-001 | Tenancy trait applies `organization_id` (and `branch_id` where defined) automatically on every query and fills it on create from the current context, ignoring any value sent in the request | R1 | * | P01.1 |  | Isolation suite; TenantScopeFailClosedTest | ☐ |
| FR-CORE-002 | Tenant context resolved from subdomain plus authenticated user; a mismatch logs the user out and records a security event | R1 | SCR-IAM-001 | P01.4 |  | TenantMismatchLogoutTest | ☐ |
| FR-CORE-003 | Numbering service: configurable formats with tokens (`{BRANCH}`, `{YYYY}`, `{YY}`, `{FY}`, `{SEQ:n}`), sequence per scope (organization, branch, fiscal year), row-locked increment, gapless option for receipts | R1 | SCR-SET-006 | P01.2 | P01.6 | NumberingServiceTest (gapless and tokens) | ☐ |
| FR-CORE-004 | `Money` value object and Eloquent cast; configurable rounding (e.g., vouchers rounded to whole rupees) | R1 | - | P01.2 |  | MoneyCastTest; rounding settings | ☐ |
| FR-CORE-005 | File storage service: private disk or S3-compatible, metadata in `stored_files`, polymorphic `file_attachments`, signed temporary URLs authorized by the owning record's policy | R1 | SCR-STU-005 | P01.2 | P04.1 | FileStorageServiceTest | ☐ |
| FR-CORE-006 | Period lock service: lock and unlock a module period (month) per branch; check API used by Attendance, Fee, Accounting, Examination, Payroll | R1 | SCR-CORE-001, SCR-CORE-002, SCR-CORE-003 | P01.6 | P02.1 | PeriodLockServiceTest | ☐ |
| FR-CORE-007 | Shared Blade component library (`x-ui.*`) implementing D9 | R1 | * | P00.3 |  | Component gallery snapshot tests | ☐ |
| FR-CORE-008 | Correlation ID generated per request, carried into logs, queued jobs and audit entries | R1 | SCR-AUD-002 | P01.2 | P01.6 | CorrelationIdPropagationTest | ☐ |
| FR-CORE-009 | Entitlement helper to check whether a module or feature is enabled for the organization | R1 | SCR-PLT-008 | P01.5 |  | EntitlementHelperTest | ☐ |
| FR-CORE-010 | Health check endpoint (database, cache, queue, storage) for monitoring | R1 | SCR-PLT-016 | P00.5 | P01.5 | Health endpoint test | ☐ |

### FR-PLT

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-PLT-001 | Create and edit plans: name, limits (max active students, branches, storage GB, monthly SMS credits), included modules | R1 | SCR-PLT-009, SCR-PLT-010 | P01.5 |  | Screen tests of listed screens | ☐ |
| FR-PLT-002 | Create an organization: profile, subdomain, plan, trial end date, owner invitation (email or SMS) | R1 | SCR-PLT-005, SCR-PLT-007 | P01.5 |  | Screen tests of listed screens | ☐ |
| FR-PLT-003 | Tenant states `trial`, `active`, `suspended`, `archived`. Suspended: login allowed, read-only, banner. Archived: no login, data retained for the contractual period | R1 | SCR-PLT-006, SCR-PLT-004 | P01.5 |  | Suspended read-only test | ☐ |
| FR-PLT-004 | Per-organization module overrides (enable an add-on, extend a limit) with effective dates | R1 | SCR-PLT-008 | P01.5 |  | Screen tests of listed screens | ☐ |
| FR-PLT-005 | Tenant console: list organizations with state, plan, student count, branch count, storage used, SMS used, last activity | R1 | SCR-PLT-003, SCR-PLT-004, SCR-PLT-001 | P01.5 | P01.6 | Screen tests of listed screens | ☐ |
| FR-PLT-006 | Support access: SA requests access, organization owner grants it for a fixed number of hours; "login as" shows a visible banner; every action is audited with both identities | R1 | SCR-PLT-012, SCR-PLT-013, SCR-PLT-014, SCR-PLT-017 | P01.5 |  | Impersonation limits test | ☐ |
| FR-PLT-007 | Platform announcements shown to organization admins (maintenance windows, releases) | R1 | SCR-PLT-015 | P01.5 |  | Screen tests of listed screens | ☐ |
| FR-PLT-008 | Usage metering stored daily in `usage_counters` | R2 | SCR-PLT-011 | P18.1 | P01.5 | UsageMeter accuracy test | ☐ |
| FR-PLT-009 | Tenant data export package (all data as CSV/JSON plus files) on request or on offboarding | R2 | SCR-PLT-020 | P18.3 |  | Tenant export completeness and isolation test | ☐ |
| FR-PLT-010 | Self-service subscription and billing for the platform's own invoices | R3 | SCR-SET-007, SCR-SET-008, SCR-PLT-021, SCR-PLT-022, SCR-PLT-023 | P24.1 |  | Screen tests of listed screens | ☐ |

### FR-ORG

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-ORG-001 | Organization profile: legal name, display name, logo, address, NTN (optional), contacts, timezone, currency, fiscal year start month | R1 | SCR-ORG-002 | P01.5 |  | Screen tests of listed screens | ☐ |
| FR-ORG-002 | Institutions: name (en/ur), type (`school`, `college`, `school_and_college`), board affiliation (e.g., BISE Lahore), registration numbers, logo and document header | R1 | SCR-ORG-004 | P01.5 |  | Screen tests of listed screens | ☐ |
| FR-ORG-003 | Branches: code, name (en/ur), address, city, district, phones, email, opening date, map coordinates (optional), status, principal | R1 | SCR-ORG-005, SCR-ORG-006 | P01.5 |  | Screen tests of listed screens | ☐ |
| FR-ORG-004 | Branch bank accounts used on challans (bank, branch, account title, number, IBAN) | R1 | SCR-ORG-008 | P01.5 |  | Screen tests of listed screens | ☐ |
| FR-ORG-005 | Departments at organization or branch level (e.g., Science, Accounts, Administration) with head | R1 | SCR-ORG-009, SCR-HR-012 | P01.5 | P05.1 | Screen tests of listed screens | ☐ |
| FR-ORG-006 | Shifts per branch: name, start and end time, working days | R1 | SCR-ORG-010 | P01.5 |  | Screen tests of listed screens | ☐ |
| FR-ORG-007 | Rooms per branch: name, type (`classroom`, `lab`, `library`, `hall`, `office`), capacity | R1 | SCR-ORG-011 | P01.5 |  | Screen tests of listed screens | ☐ |
| FR-ORG-008 | Wing list per branch (`boys`, `girls`, `combined`) used by sections | R1 | SCR-ORG-006 | P01.5 |  | Screen tests of listed screens | ☐ |
| FR-ORG-009 | Structure tree view: organization, institutions, branches, departments, with counts | R1 | SCR-ORG-003 | P01.5 |  | Screen tests of listed screens | ☐ |
| FR-ORG-010 | Deactivate a branch (no new records, history retained); deletion only if the branch has no data | R1 | SCR-ORG-007 | P01.5 |  | Screen tests of listed screens | ☐ |
| FR-ORG-011 | Onboarding wizard: organization profile, institution, branches, first session, class template, fee heads, grading template, first admin users. Scenario A (single school) is created in one pass | R1 | SCR-ORG-001 | P03.2 | P01.5 | OnboardingStepRegistry completion test | ☐ |

### FR-IAM

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-IAM-001 | Login on the organization subdomain with email, mobile number or username (students may use registration number) plus password | R1 | SCR-IAM-001, SCR-PLT-002 | P01.4 |  | Screen tests of listed screens | ☐ |
| FR-IAM-002 | Password policy per organization (length, complexity, expiry optional, history of last 5) | R1 | SCR-SET-002, SCR-IAM-009 | P01.4 | P01.6 | PasswordPolicyTest | ☐ |
| FR-IAM-003 | Account lockout after configurable failed attempts (default 5 in 15 minutes) for a configurable period; unlock by admin | R1 | SCR-IAM-001, SCR-IAM-015 | P01.4 | P01.3 | LockoutTest | ☐ |
| FR-IAM-004 | Password reset by email link or SMS OTP; tokens single-use, expire in 15 minutes | R1 | SCR-IAM-004, SCR-IAM-005 | P01.4 |  | Screen tests of listed screens | ☐ |
| FR-IAM-005 | Guardian activation: OTP sent to the registered mobile, guardian sets password; imported staff and students must change password at first login | R1 | SCR-IAM-002, SCR-IAM-003 | P04.3 | P01.4 | Guardian activation test (DEC-254) | ☐ |
| FR-IAM-006 | Optional TOTP two-factor login; can be made mandatory per role (default mandatory for OW, ACT, SA) | R1 | SCR-IAM-006, SCR-IAM-010 | P01.4 |  | Screen tests of listed screens | ☐ |
| FR-IAM-007 | Active sessions list per user with revoke; idle timeout per role | R1 | SCR-IAM-011, SCR-IAM-025 | P01.4 | P01.3 | Screen tests of listed screens | ☐ |
| FR-IAM-008 | Permission catalog registered in code by each module and synced to `permissions` on deploy, grouped by module, screen and action | R1 | SCR-IAM-019 | P01.3 |  | identity:sync-permissions test | ☐ |
| FR-IAM-009 | Role management: create custom roles, clone from system templates, edit permission matrix (module x screen x action grid) | R1 | SCR-IAM-018, SCR-IAM-019, SCR-IAM-020, SCR-IAM-021, SCR-PLT-018 | P01.3 | P01.5 | Screen tests of listed screens | ☐ |
| FR-IAM-010 | Role assignment with scope: organization, institution or specific branches, plus visibility level (DEC-21); a user may hold several assignments | R1 | SCR-IAM-022 | P01.3 |  | Screen tests of listed screens | ☐ |
| FR-IAM-011 | User permission overrides: grant or deny specific permissions per user and scope, with reason and optional expiry; deny wins | R1 | SCR-IAM-023 | P01.3 |  | Screen tests of listed screens | ☐ |
| FR-IAM-012 | Effective permission viewer: shows why a user can or cannot do an action (which role, which override) | R1 | SCR-IAM-024 | P01.3 |  | Screen tests of listed screens | ☐ |
| FR-IAM-013 | Staff user accounts linked to employee records; guardian and student accounts created by Admission/Student/Guardian modules | R1 | SCR-IAM-012, SCR-IAM-013, SCR-IAM-014, SCR-HR-002, SCR-IAM-008 | P01.3 | P01.4, P05.1 | Screen tests of listed screens | ☐ |
| FR-IAM-014 | Context switcher: active branch and active role for users with several assignments | R1 | SCR-IAM-007 | P01.6 | P04.3 | Screen tests of listed screens | ☐ |
| FR-IAM-015 | Deactivate a user (history kept, sessions revoked) | R1 | SCR-IAM-016 | P01.3 |  | Screen tests of listed screens | ☐ |
| FR-IAM-016 | Sanctum API tokens with scoped abilities for integrations and future apps | R1 | SCR-IAM-027 | P17.3 |  | Token ability tests; portal tokens in P18.2; D02 R1 tag deferred to R2 under DEC-283 (15.13) | ☐ |
| FR-IAM-017 | Trusted device registration for QR and online attendance (device fingerprint plus explicit registration, revocable) | R1 | SCR-IAM-026, SCR-ATT-012 | P01.4 | P01.3, P05.3 | Screen tests of listed screens | ☐ |

### FR-SET

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-SET-001 | Setting definitions registered in code with key, type, default, allowed scopes and validation | R1 | SCR-SET-001, SCR-SET-002 | P01.6 |  | Screen tests of listed screens | ☐ |
| FR-SET-002 | Layered values: platform default, organization, institution, branch; UI shows "inherited from" and allows reset to inherited | R1 | SCR-SET-002 | P01.6 |  | Screen tests of listed screens | ☐ |
| FR-SET-003 | Categories: general, localization, academic defaults, attendance defaults, fee defaults, notification defaults, security, branding, working days and school timing | R1 | SCR-SET-001 | P01.6 |  | Screen tests of listed screens | ☐ |
| FR-SET-004 | Branding: logos, document header and footer, signature and stamp images with signatory titles, accent color | R1 | SCR-SET-003 | P01.6 |  | Screen tests of listed screens | ☐ |
| FR-SET-005 | Language: enable Urdu, organization default language, user language preference | R1 | SCR-SET-004 | P01.6 |  | Screen tests of listed screens | ☐ |
| FR-SET-006 | Terminology overrides (e.g., "Campus" instead of "Branch") through `translation_overrides` | R1 | SCR-SET-005 | P01.6 |  | Screen tests of listed screens | ☐ |
| FR-SET-007 | Every settings change audited with old and new value | R1 | SCR-SET-002, SCR-AUD-001 | P01.6 |  | Settings audit test | ☐ |

### FR-AUD

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-AUD-001 | Automatic audit of: login, logout, failed login, password change, profile changes, student and employee updates, attendance changes and approvals, voucher generation, payments, refunds, reversals, marks entry and changes, result publication, document upload and deletion, role and permission changes, transfers, promotions, imports, exports, settings changes, support access | R1 | SCR-AUD-001 | P01.2 | P01.6 | AuditCoverageTest | ☐ |
| FR-AUD-002 | Entry fields: user, impersonator (if any), organization, branch, action key, module, record type, record id, old values, new values, reason, IP, user agent, timestamp, correlation id | R1 | SCR-AUD-002 | P01.2 | P01.6 | Screen tests of listed screens | ☐ |
| FR-AUD-003 | Sensitive values masked in logs (passwords and tokens never stored; CNIC and B-Form masked except last 4 digits) | R1 | SCR-AUD-002 | P01.2 | P01.6 | LogRedactionTest | ☐ |
| FR-AUD-004 | Audit viewer with filters: user, module, action, record, date range, branch, IP | R1 | SCR-AUD-001, SCR-PLT-019 | P01.6 |  | Screen tests of listed screens | ☐ |
| FR-AUD-005 | "History" tab on major records (student, employee, voucher, payment, marks, result, attendance day) | R1 | SCR-AUD-003 | P01.6 |  | Screen tests of listed screens | ☐ |
| FR-AUD-006 | Login history per user; repeated failures alert IT admin | R1 | SCR-AUD-004 | P01.6 |  | Screen tests of listed screens | ☐ |
| FR-AUD-007 | Audit export (gated by `audit.log.export`, itself audited) | R1 | SCR-AUD-005 | P01.6 |  | Screen tests of listed screens | ☐ |
| FR-AUD-008 | Retention: financial audit 7 years minimum, others configurable; archival to cold storage | R3 | - | P24.3 |  | Archive, verify and restore test (DEC-316) | ☐ |

### FR-APR

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-APR-001 | Workflow definition per request type per organization, with optional branch-specific override | R1 | SCR-APR-006, SCR-APR-007 | P02.1 |  | Screen tests of listed screens | ☐ |
| FR-APR-002 | Step approver resolvers: role in requester's branch, specific user, guardian of the student, class teacher of the student's section, HOD of the subject's department, requester's reporting manager | R1 | SCR-APR-007 | P02.1 |  | Resolver tests | ☐ |
| FR-APR-003 | Step modes: any one approver, or all approvers | R1 | SCR-APR-007 | P02.1 |  | Screen tests of listed screens | ☐ |
| FR-APR-004 | Conditions: amount thresholds (e.g., concession above 20% adds owner step), attribute rules (request category, days count) | R1 | SCR-APR-007 | P02.1 |  | Condition evaluation tests | ☐ |
| FR-APR-005 | Skip rules (e.g., skip guardian step when the student is 18 or older or in a college program) | R1 | SCR-APR-007 | P02.1 |  | Screen tests of listed screens | ☐ |
| FR-APR-006 | Submit request: category, description, attachments, payload of proposed change; request number from numbering service | R1 | SCR-APR-003 | P02.1 |  | Screen tests of listed screens | ☐ |
| FR-APR-007 | Statuses: `draft`, `submitted`, `in_review`, `returned`, `approved`, `rejected`, `cancelled`, `expired` | R1 | SCR-APR-002, SCR-APR-004 | P02.1 |  | Status machine test (DEC-242) | ☐ |
| FR-APR-008 | Actions: approve, reject, return for changes; comment mandatory on reject and return | R1 | SCR-APR-002 | P02.1 |  | Screen tests of listed screens | ☐ |
| FR-APR-009 | On final approval the owning module's handler applies the change in a transaction and writes audit | R1 | SCR-APR-002 | P02.1 |  | Handler contract test | ☐ |
| FR-APR-010 | Inboxes in every portal: "My requests" and "Pending my approval", with counts on dashboards | R1 | SCR-APR-001, SCR-APR-003, SCR-APR-009 | P02.1 | P04.3 | Screen tests of listed screens | ☐ |
| FR-APR-011 | Approval timeline shown on the request and on the affected record | R1 | SCR-APR-002 | P02.1 |  | Screen tests of listed screens | ☐ |
| FR-APR-012 | Delegation: an approver on leave delegates to another eligible user for a date range | R1 | SCR-APR-008 | P02.1 |  | Screen tests of listed screens | ☐ |
| FR-APR-013 | Bulk approve for low-risk types (leave) with permission | R1 | SCR-APR-005 | P02.1 |  | Screen tests of listed screens | ☐ |
| FR-APR-014 | SLA timers and escalation to the next role after N hours | R2 | SCR-APR-007 | P16.4 | P02.1 | Escalation timing test (DEC-291) | ☐ |

### FR-NTF

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-NTF-001 | Rules: event, channels, recipient resolvers (guardians of student, class teacher, approvers, role in branch, specific users), enabled flag | R1 | SCR-NTF-005 | P02.2 |  | Screen tests of listed screens | ☐ |
| FR-NTF-002 | Templates per event per channel per language with placeholders, preview, and SMS segment counter | R1 | SCR-NTF-003, SCR-NTF-004 | P02.2 |  | Screen tests of listed screens | ☐ |
| FR-NTF-003 | Provider configuration per organization (SMTP, SMS aggregator; WhatsApp in R2) with encrypted credentials and test send | R1 | SCR-NTF-006, SCR-NTF-007 | P02.2 |  | Screen tests of listed screens | ☐ |
| FR-NTF-004 | Queued sending with per-provider rate limits | R1 | SCR-NTF-008 | P02.2 |  | Rate limit test | ☐ |
| FR-NTF-005 | Retry with backoff (3 attempts) and optional fallback SMS provider | R1 | SCR-NTF-008 | P02.2 |  | Retry and fallback test | ☐ |
| FR-NTF-006 | Delivery status updates from provider callbacks where supported | R1 | SCR-NTF-008 | P02.2 |  | Delivery receipt webhook test | ☐ |
| FR-NTF-007 | Delivery log searchable by recipient, event, channel, status | R1 | SCR-NTF-008 | P02.2 |  | Screen tests of listed screens | ☐ |
| FR-NTF-008 | User preferences per channel for non-critical events; critical events (absence, fee overdue, OTP) cannot be turned off | R1 | SCR-NTF-002 | P02.2 |  | Screen tests of listed screens | ☐ |
| FR-NTF-009 | Quiet hours (default 21:00 to 07:00) for SMS and WhatsApp; OTP and urgent bypass | R1 | SCR-NTF-002, SCR-NTF-005 | P02.2 |  | Quiet hours test | ☐ |
| FR-NTF-010 | Deduplication: same event for same recipient and record within a window is sent once | R1 | SCR-NTF-008 | P02.2 |  | Deduplication test | ☐ |
| FR-NTF-011 | Manual broadcast with audience selection and cost estimate before sending | R1 | SCR-NTF-009 | P02.2 |  | Screen tests of listed screens | ☐ |
| FR-NTF-012 | SMS credit quota per organization; warning at 80%, block at 100% except OTP | R1 | SCR-PLT-011, SCR-NTF-009 | P02.2 | P01.5 | SMS quota test | ☐ |
| FR-NTF-013 | In-app notification center with unread count and mark-as-read | R1 | SCR-NTF-001 | P02.2 |  | Screen tests of listed screens | ☐ |
| FR-NTF-014 | WhatsApp channel through approved provider with pre-approved templates | R2 | SCR-NTF-006, SCR-NTF-004, SCR-NTF-009 | P16.2 | P02.2 | Screen tests of listed screens | ☐ |
| FR-NTF-015 | Push notifications through FCM for future apps | R3 | SCR-PLT-023 | P24.2 | P24.1 | Push payload content test | ☐ |

### FR-IMX

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-IMX-001 | Import types registry. R1: students, guardians, families, employees, classes and sections, subjects, opening fee balances, opening ledger balances, attendance, marks. R2: all others | R1 | SCR-IMX-002, SCR-IAM-017, SCR-HR-013, SCR-FEE-031 | P02.3 | P04.4, P05.1, P06.6 | Import type registry test | ☐ |
| FR-IMX-002 | Downloadable Excel templates with instructions sheet and dropdown validation | R1 | SCR-IMX-002 | P02.3 | P04.4 | Screen tests of listed screens | ☐ |
| FR-IMX-003 | Upload XLSX or CSV; column mapping UI; saved mappings per organization | R1 | SCR-IMX-002, SCR-IMX-004 | P02.3 | P04.4 | Screen tests of listed screens | ☐ |
| FR-IMX-004 | Validation pass with preview: valid rows, warnings and errors per row and column | R1 | SCR-IMX-003 | P02.3 |  | Screen tests of listed screens | ☐ |
| FR-IMX-005 | Duplicate detection rules per type (B-Form, CNIC, admission number, mobile plus name); options to skip or update where allowed | R1 | SCR-IMX-003 | P02.3 |  | Duplicate rule tests | ☐ |
| FR-IMX-006 | Commit as a queued job in chunks, each chunk in a transaction | R1 | SCR-IMX-003 | P02.3 |  | Screen tests of listed screens | ☐ |
| FR-IMX-007 | Failed rows downloadable with an error column | R1 | SCR-IMX-003 | P02.3 |  | Screen tests of listed screens | ☐ |
| FR-IMX-008 | Import history: who, when, file, counts, status | R1 | SCR-IMX-001 | P02.3 |  | Screen tests of listed screens | ☐ |
| FR-IMX-009 | Export jobs for lists and reports; links expire after 24 hours; export parameters audited | R1 | SCR-IMX-005 | P02.3 |  | Screen tests of listed screens | ☐ |
| FR-IMX-010 | Undo an import batch within 24 hours when no dependent records exist | R2 | SCR-IMX-003 | P18.3 | P02.3 | Import undo test (DEC-324) | ☐ |

### FR-DOC

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-DOC-001 | Template types: fee voucher/challan, receipt, report card, admit card, student ID card, staff ID card, leaving certificate, character certificate, migration certificate, admission form, attendance sheet, date sheet, salary slip (R2), financial reports | R1 | SCR-DOC-001 | P02.4 |  | Screen tests of listed screens | ☐ |
| FR-DOC-002 | Default templates shipped; per-institution branding (logo, name, address, signatures, stamp) | R1 | SCR-DOC-002 | P02.4 |  | Screen tests of listed screens | ☐ |
| FR-DOC-003 | Paper layouts: A4, Legal, 3-copy challan on A4 landscape, CR80 ID card sheets | R1 | SCR-DOC-002 | P02.4 |  | Screen tests of listed screens | ☐ |
| FR-DOC-004 | Bilingual labels (English/Urdu) per template | R1 | SCR-DOC-002 | P02.4 |  | Bilingual rendering snapshot test | ☐ |
| FR-DOC-005 | Rendering through headless Chromium (DEC-12); single documents synchronous, bulk queued | R1 | SCR-DOC-002 | P02.4 |  | Gotenberg render test | ☐ |
| FR-DOC-006 | Bulk merged PDF (e.g., all vouchers for a class) with progress | R1 | SCR-FEE-016, SCR-RES-012, SCR-CORE-004 | P02.4 | P01.6, P06.3, P07.2 | Bulk merged PDF test | ☐ |
| FR-DOC-007 | Issued document registry for certificates: serial number, verification code, content snapshot | R1 | SCR-DOC-004 | P02.4 |  | Screen tests of listed screens | ☐ |
| FR-DOC-008 | Reprints marked "Duplicate" with reprint count | R1 | SCR-DOC-005 | P02.4 |  | Duplicate watermark test | ☐ |
| FR-DOC-009 | Public QR verification page showing minimal data (name, document type, date, valid/revoked) | R2 | SCR-DOC-006, SCR-RES-021 | P16.4 |  | Screen tests of listed screens | ☐ |
| FR-DOC-010 | Template designer (drag fields, sizes) | R3 | SCR-DOC-003 | P23.2 |  | Screen tests of listed screens | ☐ |

### FR-ACD

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-ACD-001 | Academic sessions per institution: name (e.g., 2026-2027), start and end dates, admission window, exam window, promotion date, status `planning`, `active`, `closed` | R1 | SCR-ACD-001, SCR-ACD-002, SCR-ACD-004 | P03.1 |  | Screen tests of listed screens | ☐ |
| FR-ACD-002 | Session closing checklist (results published, promotions done, fee balances carried forward) and close action that makes the session read-only | R1 | SCR-ACD-005 | P08.1 | P03.2 | SessionCloseCheckRegistry test | ☐ |
| FR-ACD-003 | New session wizard: copy classes, sections, class subjects, teaching assignments (optional) and fee structures from the previous session | R1 | SCR-ACD-003, SCR-FEE-005 | P03.2 | P06.2 | SessionCopyStepRegistry test | ☐ |
| FR-ACD-004 | Education levels per institution: name (en/ur), order, stage (`pre_primary`, `primary`, `middle`, `secondary`, `higher_secondary`), next level for promotion, board-exam flag, active flag. Seeded with Playgroup to Intermediate Part 2 | R1 | SCR-ACD-006 | P03.1 |  | Screen tests of listed screens | ☐ |
| FR-ACD-005 | Study groups (matric: Science, Arts/Humanities, Computer Science; intermediate: F.Sc Pre-Medical, F.Sc Pre-Engineering, ICS, I.Com, FA) with applicable levels | R1 | SCR-ACD-007 | P03.1 |  | Screen tests of listed screens | ☐ |
| FR-ACD-006 | Study group subjects per level: compulsory or elective, minimum and maximum electives | R1 | SCR-ACD-007, SCR-ACD-015, SCR-ACD-016 | P03.1 | P03.2 | Screen tests of listed screens | ☐ |
| FR-ACD-007 | Subjects: name (en/ur), code, type (`theory`, `practical`, `theory_practical`), board subject flag, department | R1 | SCR-ACD-008 | P03.1 |  | Screen tests of listed screens | ☐ |
| FR-ACD-008 | Academic classes per branch per session per level, optionally per study group (e.g., "Class 9 Science") | R1 | SCR-ACD-009, SCR-ACD-010 | P03.2 |  | Screen tests of listed screens | ☐ |
| FR-ACD-009 | Sections: name, capacity, shift, wing, class teacher, room | R1 | SCR-ACD-011 | P03.2 |  | Screen tests of listed screens | ☐ |
| FR-ACD-010 | Class subjects: subjects offered for a class, periods per week, compulsory or elective, default marks components | R1 | SCR-ACD-012 | P03.2 |  | Screen tests of listed screens | ☐ |
| FR-ACD-011 | Teaching assignments: employee, section, subject, session, periods per week, role (`primary`, `co_teacher`), effective dates | R1 | SCR-ACD-013, SCR-ACD-017 | P05.1 |  | Screen tests of listed screens | ☐ |
| FR-ACD-012 | Teacher workload view with maximum periods warning | R1 | SCR-ACD-014 | P05.1 |  | Screen tests of listed screens | ☐ |
| FR-ACD-013 | Section capacity enforcement: warning or hard limit (setting) | R1 | SCR-ACD-011 | P03.2 |  | Capacity setting test | ☐ |
| FR-ACD-014 | Inter-branch teaching assignments when the employee has an active branch assignment there | R1 | SCR-ACD-013 | P05.1 |  | Screen tests of listed screens | ☐ |
| FR-ACD-015 | Board exam level mapping: which levels sit board exams and with which board | R2 | SCR-ACD-006, SCR-EXT-003 | P03.1 | P07.3 | is_external_exam_level flag plus centre numbers per awarding body; met in R1 ahead of the D02 R2 tag (DEC-324) | ☐ |

### FR-ADM

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-ADM-001 | Admission form templates per institution: fields from student and guardian schema plus custom fields, required flags, document checklist | R1 | SCR-ADM-009 | P04.2 |  | Screen tests of listed screens | ☐ |
| FR-ADM-002 | Public online application page per branch with mobile OTP verification, captcha, document upload and application number | R1 | SCR-ADM-010, SCR-ADM-011, SCR-ADM-005 | P04.2 |  | Screen tests of listed screens | ☐ |
| FR-ADM-003 | Offline application entry at the office | R1 | SCR-ADM-002 | P04.2 |  | Screen tests of listed screens | ☐ |
| FR-ADM-004 | Pipeline statuses: `submitted`, `under_review`, `test_scheduled`, `interview_scheduled`, `approved`, `waitlisted`, `rejected`, `admitted`, `withdrawn` | R1 | SCR-ADM-001, SCR-ADM-003 | P04.2 |  | Screen tests of listed screens | ☐ |
| FR-ADM-005 | Review screen with document verification checklist | R1 | SCR-ADM-003 | P04.2 |  | Screen tests of listed screens | ☐ |
| FR-ADM-006 | Duplicate check on B-Form, name plus date of birth plus father name; sibling detection by guardian CNIC or mobile | R1 | SCR-ADM-002 | P04.2 |  | Duplicate and sibling detection test | ☐ |
| FR-ADM-007 | Approval generates admission fee voucher (Fee) | R1 | SCR-ADM-004, SCR-FEE-014 | P06.3 | P04.2 | AdmissionPaymentGate test (DEC-248) | ☐ |
| FR-ADM-008 | Conversion to student after payment (or override with permission): create student, enrollment, guardian links, family, registration, GR, admission and roll numbers, section allocation, guardian portal activation SMS | R1 | SCR-ADM-006 | P04.2 |  | Screen tests of listed screens | ☐ |
| FR-ADM-009 | Section allocation manual or automatic (capacity, balanced gender) | R1 | SCR-ADM-006 | P04.2 |  | Screen tests of listed screens | ☐ |
| FR-ADM-010 | Seat availability per class and section | R1 | SCR-ADM-007 | P04.2 |  | Screen tests of listed screens | ☐ |
| FR-ADM-011 | GR register printable report | R1 | SCR-ADM-008 | P04.2 |  | Screen tests of listed screens | ☐ |
| FR-ADM-012 | Bulk student import (via IMX) | R1 | SCR-IMX-002 | P04.4 |  | Screen tests of listed screens | ☐ |
| FR-ADM-013 | Admission tests and interviews: schedule, slots, result entry | R2 | SCR-ADM-013 | P16.4 |  | Screen tests of listed screens | ☐ |
| FR-ADM-014 | Inquiries: source, contact, class sought, follow-up dates, conversion tracking | R2 | SCR-ADM-012 | P16.4 |  | Screen tests of listed screens | ☐ |
| FR-ADM-015 | Rejected applications anonymized after a retention period | R2 | - | P16.4 |  | Anonymization job test (DEC-324) | ☐ |

### FR-STU

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-STU-001 | Student profile tabs: personal, guardians, academic, health, documents, fees, attendance, results, discipline, learning activity, timeline, history | R1 | SCR-STU-002, SCR-STU-024, SCR-ATT-025 | P04.1 | P04.3, P05.2 | Screen tests of listed screens | ☐ |
| FR-STU-002 | Identifiers: registration number (organization-unique, permanent), GR number (branch), admission number, roll number per enrollment | R1 | SCR-STU-002, SCR-STU-003 | P04.1 |  | Screen tests of listed screens | ☐ |
| FR-STU-003 | Personal fields per brief plus optional Urdu names for student and father | R1 | SCR-STU-003 | P04.1 |  | Screen tests of listed screens | ☐ |
| FR-STU-004 | Photo upload with crop and compression | R1 | SCR-STU-003 | P04.1 |  | Screen tests of listed screens | ☐ |
| FR-STU-005 | Category (e.g., regular, staff child, sponsored) and sponsor | R1 | SCR-STU-021, SCR-STU-022 | P04.1 |  | Screen tests of listed screens | ☐ |
| FR-STU-006 | Statuses: `active`, `suspended`, `struck_off`, `withdrawn`, `transferred_out`, `graduated` | R1 | SCR-STU-001, SCR-STU-002 | P04.1 |  | Screen tests of listed screens | ☐ |
| FR-STU-007 | Search within scope by name, father name, B-Form, mobile, any identifier | R1 | SCR-STU-001 | P04.1 |  | Screen tests of listed screens | ☐ |
| FR-STU-008 | Timeline events created automatically: admission, enrollment, promotion, transfer, status change, fee milestones, results, discipline, certificates | R1 | SCR-STU-018 | P04.1 |  | Screen tests of listed screens | ☐ |
| FR-STU-009 | Protected fields (name, DOB, B-Form, father name, admission date) editable only with `student.profile.update_protected` and reason | R1 | SCR-STU-004 | P04.1 |  | Screen tests of listed screens | ☐ |
| FR-STU-010 | Profile correction and modification requests from guardian or student through APR | R1 | SCR-STU-019, SCR-STU-026 | P04.3 |  | Screen tests of listed screens | ☐ |
| FR-STU-011 | Documents with verification status (pending, verified, rejected) | R1 | SCR-STU-005, SCR-STU-006 | P04.1 |  | Screen tests of listed screens | ☐ |
| FR-STU-012 | Elective subject selection per enrollment within study group rules | R1 | SCR-STU-002 | P04.1 |  | Screen tests of listed screens | ☐ |
| FR-STU-013 | Bulk promotion per class: eligibility from results (pass, compartment, fail), manual override with reason, target section mapping; creates next-session enrollments and closes current ones | R1 | SCR-STU-009, SCR-STU-010 | P08.1 |  | Screen tests of listed screens | ☐ |
| FR-STU-014 | Retention (repeat class) as a new enrollment in the same level | R1 | SCR-STU-010 | P08.1 |  | Screen tests of listed screens | ☐ |
| FR-STU-015 | Section and class transfer within session with effective date; enrollment history kept | R1 | SCR-STU-008 | P08.2 |  | Screen tests of listed screens | ☐ |
| FR-STU-016 | Branch transfer within organization through approval: closes enrollment in source, opens in target, moves fee balance per policy | R1 | SCR-STU-011, SCR-STU-012 | P08.2 |  | Screen tests of listed screens | ☐ |
| FR-STU-017 | Withdrawal with clearance checklist (fees, library, transport, documents returned), reason, leaving certificate | R1 | SCR-STU-013, SCR-STU-014 | P08.2 |  | ClearanceCheck tests | ☐ |
| FR-STU-018 | Struck-off processing (list from Fee rule, principal confirms) and re-admission | R1 | SCR-STU-015 | P08.2 |  | Screen tests of listed screens | ☐ |
| FR-STU-019 | Graduation of final-level students at session close | R1 | SCR-STU-009 | P08.1 |  | Screen tests of listed screens | ☐ |
| FR-STU-020 | Bulk actions: assign section, change category, print ID cards | R1 | SCR-STU-020 | P04.1 |  | Screen tests of listed screens | ☐ |
| FR-STU-021 | Health profile (blood group, conditions, allergies, emergency instructions) visible only with `student.health.view` | R1 | SCR-STU-007 | P04.1 |  | Screen tests of listed screens | ☐ |
| FR-STU-022 | Character, leaving and migration certificates issued via DOC | R1 | SCR-STU-016, SCR-STU-017 | P08.3 |  | Screen tests of listed screens | ☐ |

### FR-GRD

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-GRD-001 | Guardian profile: name (en/ur), relation, CNIC, primary and secondary mobiles, email, occupation, address | R1 | SCR-GRD-001, SCR-GRD-002, SCR-GRD-003 | P04.1 |  | Screen tests of listed screens | ☐ |
| FR-GRD-002 | Link to students with relation and flags: primary contact, fee payer, emergency contact, portal access, can approve requests, pickup allowed | R1 | SCR-GRD-004 | P04.1 |  | Screen tests of listed screens | ☐ |
| FR-GRD-003 | Custody restriction flag: no portal access and no pickup, with document and reason (restricted visibility) | R1 | SCR-GRD-007 | P04.1 |  | Screen tests of listed screens | ☐ |
| FR-GRD-004 | Families formed automatically by shared guardian CNIC; manual merge and split with permission | R1 | SCR-GRD-005, SCR-GRD-006 | P04.1 |  | Screen tests of listed screens | ☐ |
| FR-GRD-005 | Family view: siblings, combined dues, family voucher setting | R1 | SCR-GRD-005, SCR-FEE-029 | P04.1 | P06.6 | Screen tests of listed screens | ☐ |
| FR-GRD-006 | Guardian verification: by office, or by code printed on admission slip or voucher | R1 | SCR-GRD-009 | P04.1 |  | Screen tests of listed screens | ☐ |
| FR-GRD-007 | Guardian portal account per DEC-22 (shared mobile links all related guardian records to one user) | R1 | SCR-GRD-010 | P04.3 |  | Screen tests of listed screens | ☐ |
| FR-GRD-008 | Guardian portal data: children list and switcher, attendance and alerts, results and report cards, daily diary, timetable, vouchers, payment history, announcements, leave application for child, correction requests, approval of student requests, disciplinary notices | R1 | SCR-GRD-010, SCR-RPT-008, SCR-STU-025, SCR-ATT-024, SCR-RES-017, SCR-FEE-036 | P04.3 | P05.2, P06.3, P07.2, P09.3 | Screen tests of listed screens | ☐ |
| FR-GRD-009 | Pickup authorizations: names, CNIC, photo, relation | R1 | SCR-GRD-008, SCR-GRD-011 | P04.1 | P04.3 | Screen tests of listed screens | ☐ |
| FR-GRD-010 | Guardian portal R2 additions: online payment, homework and lectures, messaging, PTM booking | R2 | SCR-FEE-038, SCR-HWK-008, SCR-LMS-011, SCR-COMM-015, SCR-COMM-016 | P16.1 | P11.2, P14.2, P15.1 | Portal screen tests in each delivering step | ☐ |

### FR-HR

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-HR-001 | Employee profile per brief: personal, contact, CNIC, emergency contact, qualifications, experience, specialization, joining date, employment type, department, designation, photo | R1 | SCR-HR-002, SCR-HR-003, SCR-HR-007, SCR-HR-001 | P05.1 |  | Screen tests of listed screens | ☐ |
| FR-HR-002 | Employee ID from numbering service | R1 | SCR-HR-003 | P05.1 |  | Screen tests of listed screens | ☐ |
| FR-HR-003 | Employment types: `permanent`, `contract`, `visiting`, `part_time`, `daily_wage` | R1 | SCR-HR-003 | P05.1 |  | Screen tests of listed screens | ☐ |
| FR-HR-004 | Primary branch plus additional branch assignments with effective dates (shared employees) | R1 | SCR-HR-006 | P05.1 |  | Screen tests of listed screens | ☐ |
| FR-HR-005 | Teaching flag and subject specializations | R1 | SCR-HR-003 | P05.1 |  | Screen tests of listed screens | ☐ |
| FR-HR-006 | Reporting manager, shift and work schedule | R1 | SCR-HR-003 | P05.1 |  | Screen tests of listed screens | ☐ |
| FR-HR-007 | Documents (CV, degrees, CNIC copy, contract) with expiry dates and reminders | R1 | SCR-HR-008 | P05.1 |  | Screen tests of listed screens | ☐ |
| FR-HR-008 | Statuses: `active`, `on_leave`, `suspended`, `resigned`, `terminated`, `retired` | R1 | SCR-HR-004 | P05.1 |  | Screen tests of listed screens | ☐ |
| FR-HR-009 | User account creation for employee with role suggestion by designation | R1 | SCR-HR-003, SCR-IAM-013, SCR-HR-009 | P05.1 | P01.3 | Screen tests of listed screens | ☐ |
| FR-HR-010 | Staff directory with contact privacy settings | R1 | SCR-HR-010 | P05.1 |  | Screen tests of listed screens | ☐ |
| FR-HR-011 | Self-service: view profile, request profile correction (APR), view attendance and leave balance | R1 | SCR-HR-011, SCR-ATT-011, SCR-LVE-003 | P05.1 | P05.3, P05.4 | Screen tests of listed screens | ☐ |
| FR-HR-012 | Contracts with terms, start, end, renewal reminders | R2 | SCR-HR-014 | P13.1 |  | Screen tests of listed screens | ☐ |
| FR-HR-013 | Performance review cycles, criteria, ratings, comments, employee acknowledgment | R2 | SCR-HR-015 | P13.1 |  | Screen tests of listed screens | ☐ |
| FR-HR-014 | Training records | R2 | SCR-HR-016 | P13.1 |  | Screen tests of listed screens | ☐ |
| FR-HR-015 | Disciplinary records (restricted permission) | R2 | SCR-HR-017 | P13.1 |  | Screen tests of listed screens | ☐ |
| FR-HR-016 | Separation workflow with clearance (assets, library, dues) and final settlement link to Payroll | R2 | SCR-HR-018 | P13.1 |  | Screen tests of listed screens | ☐ |

### FR-ATT

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-ATT-001 | Policies per branch, shift and person type: start time, grace minutes, late threshold, half-day threshold, early departure threshold, auto-absent cut-off, lates-to-leave conversion, official sources | R1 | SCR-ATT-016 | P05.2 |  | Screen tests of listed screens | ☐ |
| FR-ATT-002 | Teacher marks daily student attendance per section: all present by default, tap to change status, bulk actions, remarks | R1 | SCR-ATT-001, SCR-ATT-002 | P05.2 |  | Screen tests of listed screens | ☐ |
| FR-ATT-003 | Marking window (e.g., until 11:00); after it the day is finalized and changes need a correction | R1 | SCR-ATT-002 | P05.2 |  | Finalization window test (DEC-180) | ☐ |
| FR-ATT-004 | Optional period-wise (subject) attendance per class, mainly for colleges | R1 | SCR-ATT-003 | P09.1 |  | Screen tests of listed screens | ☐ |
| FR-ATT-005 | "Not marked" alert to class teacher and principal at a set time | R1 | SCR-ATT-001 | P05.2 |  | Not-marked alert test | ☐ |
| FR-ATT-006 | Staff online check-in and check-out with trusted device and optional geofence (branch coordinates and radius) | R1 | SCR-ATT-012 | P05.3 |  | Screen tests of listed screens | ☐ |
| FR-ATT-007 | QR attendance: kiosk shows a code that rotates every 30 seconds; staff scan with a logged-in trusted phone; codes expire and are single use per person per session. Teachers can also show a class QR for college lectures | R1 | SCR-ATT-013, SCR-ATT-014 | P05.3 |  | Screen tests of listed screens | ☐ |
| FR-ATT-008 | Device registry: serial, vendor, model, branch, location, adapter type, secret, timezone, status, last heartbeat | R1 | SCR-ATT-017, SCR-ATT-018 | P05.3 |  | Screen tests of listed screens | ☐ |
| FR-ATT-009 | Device enrollment mapping: device user ID to student or employee | R1 | SCR-ATT-019 | P05.3 |  | Screen tests of listed screens | ☐ |
| FR-ATT-010 | ZKTeco ADMS push endpoint (handshake, punch upload, heartbeat) | R1 | SCR-ATT-021 | P05.3 |  | ADMS protocol tests | ☐ |
| FR-ATT-011 | Signed webhook adapter (HMAC) for other devices or middleware | R1 | SCR-ATT-021 | P05.3 |  | HMAC webhook test | ☐ |
| FR-ATT-012 | CSV/Excel punch log import | R1 | SCR-ATT-020 | P05.3 |  | Screen tests of listed screens | ☐ |
| FR-ATT-013 | Raw punches stored immutably with source and verification type (fingerprint, face, card, PIN, QR) | R1 | SCR-ATT-021 | P05.3 |  | Raw punch immutability test | ☐ |
| FR-ATT-014 | Resolver job computes daily `staff_attendances` and device-based `student_attendances` using DEC-25 precedence; calculates late minutes and early departure | R1 | SCR-ATT-008, SCR-ATT-009 | P05.3 |  | Resolver tests (DEC-25) | ☐ |
| FR-ATT-015 | Offline device alert when no heartbeat or punch for X hours on a working day | R1 | SCR-ATT-017 | P05.3 |  | Offline device alert test | ☐ |
| FR-ATT-016 | Guardian absence and late alerts after finalization (configurable time) | R1 | SCR-ATT-024 | P05.2 |  | Absence alert test | ☐ |
| FR-ATT-017 | Corrections: teacher requests for finalized student attendance; staff requests for missing punch or correction; through APR | R1 | SCR-ATT-006, SCR-ATT-007, SCR-ATT-015 | P05.2 | P05.3 | Screen tests of listed screens | ☐ |
| FR-ATT-018 | Approved corrections create a new version; the original stays visible in history | R1 | SCR-ATT-005 | P05.2 |  | Revision history test (DEC-105) | ☐ |
| FR-ATT-019 | Authorized administrative attendance entry with mandatory reason | R1 | SCR-ATT-010 | P05.3 |  | Screen tests of listed screens | ☐ |
| FR-ATT-020 | Month lock through period locks | R1 | SCR-CORE-001 | P05.4 | P01.6 | Screen tests of listed screens | ☐ |
| FR-ATT-021 | Reports: daily, monthly register grid, student, teacher, branch, class, section, late arrivals, early departures, absentees, missing punches, correction history, percentage, device sync logs; all with filters, print, PDF, Excel | R1 | SCR-ATT-004, SCR-ATT-009, SCR-ATT-023 | P05.4 | P05.2, P05.3 | Screen tests of listed screens | ☐ |
| FR-ATT-022 | Suspicious pattern alerts: many identities on one device within seconds, check-in from a new device, attempts outside geofence, manual overrides above a threshold | R1 | SCR-ATT-022 | P05.3 |  | Screen tests of listed screens | ☐ |
| FR-ATT-023 | On-premise sync agent for LAN-only devices (pull, then push to cloud) | R2 | SCR-ATT-017 | P17.3 | P05.3 | Agent exactly-once test | ☐ |
| FR-ATT-024 | RFID reader support through adapters | R2 | SCR-ATT-018 | P17.3 | P05.3 | RFID adapter test | ☐ |
| FR-ATT-025 | Push user lists to devices (ADMS command queue) | R2 | SCR-ATT-019 | P17.3 | P05.3 | User list push test (DEC-324) | ☐ |
| FR-ATT-026 | Offline-capable attendance marking that syncs later | R3 | SCR-ATT-002, SCR-ATT-026 | P23.4 | P05.2 | Screen tests of listed screens | ☐ |

### FR-LVE

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-LVE-001 | Leave types: casual, sick, annual, maternity, hajj, unpaid, short leave, student sick, student family; applicability (staff/students), paid or unpaid, document required after N days | R1 | SCR-LVE-008 | P05.4 |  | Screen tests of listed screens | ☐ |
| FR-LVE-002 | Policies by employment type or designation: annual quota, accrual (monthly/yearly), carry forward cap, half-day and short leave allowed, sandwich rule (weekends between leave days counted) | R1 | SCR-LVE-009 | P05.4 |  | Screen tests of listed screens | ☐ |
| FR-LVE-003 | Balances per year with opening balance entry | R1 | SCR-LVE-005, SCR-LVE-006 | P05.4 |  | Screen tests of listed screens | ☐ |
| FR-LVE-004 | Apply: staff self-service, guardian for child, admin on behalf; date range, half day, reason, attachment | R1 | SCR-LVE-002, SCR-LVE-004, SCR-LVE-011 | P05.4 |  | Screen tests of listed screens | ☐ |
| FR-LVE-005 | Checks: overlap, balance, locked month | R1 | SCR-LVE-002, SCR-LVE-004 | P05.4 |  | Overlap, balance and lock tests | ☐ |
| FR-LVE-006 | Approval through APR; approved leave updates attendance | R1 | SCR-LVE-001, SCR-APR-002 | P05.4 | P02.1 | Screen tests of listed screens | ☐ |
| FR-LVE-007 | Cancel before start date; cancellation request after start | R1 | SCR-LVE-003, SCR-LVE-012 | P05.4 |  | Screen tests of listed screens | ☐ |
| FR-LVE-008 | "On leave today" view and leave calendar | R1 | SCR-LVE-007 | P05.4 |  | Screen tests of listed screens | ☐ |
| FR-LVE-009 | Leave register and balance reports | R1 | SCR-LVE-010 | P05.4 |  | Screen tests of listed screens | ☐ |
| FR-LVE-010 | Substitution prompt for teacher leave (Timetable) | R2 | SCR-TTB-009 | P16.3 |  | Screen tests of listed screens | ☐ |
| FR-LVE-011 | Leave encashment rules feeding Payroll | R2 | SCR-PAY-007 | P13.3 |  | Final settlement encashment test | ☐ |

### FR-FEE

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-FEE-001 | Fee heads: name (en/ur), code, type (`monthly`, `annual`, `one_time`, `on_demand`, `refundable_deposit`, `fine`), ledger account mapping, taxable flag | R1 | SCR-FEE-001 | P06.2 |  | Screen tests of listed screens | ☐ |
| FR-FEE-002 | Fee structures per institution, branch, session, class, with optional section, study group and student category; items with amount and applicable months | R1 | SCR-FEE-003, SCR-FEE-004 | P06.2 |  | Screen tests of listed screens | ☐ |
| FR-FEE-003 | Effective-dated revisions within a session (mid-year fee increase) | R1 | SCR-FEE-004 | P06.2 |  | Screen tests of listed screens | ☐ |
| FR-FEE-004 | Vacation fee rules: collect June and July (configurable months) with an earlier voucher | R1 | SCR-FEE-004 | P06.2 |  | Screen tests of listed screens | ☐ |
| FR-FEE-005 | Category rules (e.g., staff child 50% tuition, sponsored 100%) | R1 | SCR-FEE-006 | P06.2 |  | Screen tests of listed screens | ☐ |
| FR-FEE-006 | Sibling discount rules (2nd child X%, 3rd child Y%, on selected heads) | R1 | SCR-FEE-006 | P06.2 |  | Screen tests of listed screens | ☐ |
| FR-FEE-007 | Individual concessions: percentage or fixed, per head, date range, reason; approval above threshold | R1 | SCR-FEE-007, SCR-FEE-008 | P06.2 |  | Screen tests of listed screens | ☐ |
| FR-FEE-008 | Scholarships: type, sponsor, amount or percentage, period, approval | R1 | SCR-FEE-009 | P06.2 |  | Screen tests of listed screens | ☐ |
| FR-FEE-009 | Bulk monthly voucher generation by branch, class, section or all: current heads, arrears, unpaid late fee, discounts, tax lines | R1 | SCR-FEE-013 | P06.3 |  | Screen tests of listed screens | ☐ |
| FR-FEE-010 | Family voucher option combining siblings in one voucher | R1 | SCR-FEE-013, SCR-FEE-029 | P06.3 | P06.6 | Screen tests of listed screens | ☐ |
| FR-FEE-011 | Voucher fields: number, issue date, due date, validity date, fee month, guardian details, bank accounts, amounts payable within and after due date | R1 | SCR-FEE-012 | P06.3 |  | Screen tests of listed screens | ☐ |
| FR-FEE-012 | Late fee rules: fixed after due date or per day with cap | R1 | SCR-FEE-001 | P06.2 |  | Late fee rule test (DEC-260) | ☐ |
| FR-FEE-013 | Voucher statuses: `issued`, `partially_paid`, `paid`, `overdue`, `cancelled`, `superseded`; cancel or regenerate with reason | R1 | SCR-FEE-011, SCR-FEE-015 | P06.3 |  | Screen tests of listed screens | ☐ |
| FR-FEE-014 | Ad-hoc vouchers (admission, exam fee, fines, charges) | R1 | SCR-FEE-014 | P06.3 |  | Screen tests of listed screens | ☐ |
| FR-FEE-015 | Installment plans for large amounts | R1 | SCR-FEE-010 | P06.5 |  | Screen tests of listed screens | ☐ |
| FR-FEE-016 | Printing: 3-copy challan, bulk PDF per class; guardian download | R1 | SCR-FEE-016, SCR-FEE-037, SCR-FEE-042 | P06.3 |  | Screen tests of listed screens | ☐ |
| FR-FEE-017 | Payment entry against voucher or student account: cash, bank deposit, cheque | R1 | SCR-FEE-017, SCR-FEE-018 | P06.4 |  | Screen tests of listed screens | ☐ |
| FR-FEE-018 | Partial payments (setting) with allocation order (oldest first or head priority) | R1 | SCR-FEE-017 | P06.4 |  | Allocation order test | ☐ |
| FR-FEE-019 | Receipt print and SMS confirmation | R1 | SCR-FEE-019, SCR-FEE-040 | P06.4 |  | Screen tests of listed screens | ☐ |
| FR-FEE-020 | Cheque lifecycle: received, deposited, cleared, bounced (reverses payment, adds bounce charge) | R1 | SCR-FEE-021 | P06.4 |  | Screen tests of listed screens | ☐ |
| FR-FEE-021 | Bank challan bulk entry: enter voucher numbers from stamped copies or import a bank file | R1 | SCR-FEE-022, SCR-FEE-023 | P06.4 |  | Screen tests of listed screens | ☐ |
| FR-FEE-022 | Cashier sessions: open with float, close with counted cash, variance recorded and approved | R1 | SCR-FEE-024, SCR-FEE-025, SCR-FEE-026 | P06.4 |  | Screen tests of listed screens | ☐ |
| FR-FEE-023 | Payment reversal through approval (reversing entries) | R1 | SCR-FEE-020 | P06.5 |  | Screen tests of listed screens | ☐ |
| FR-FEE-024 | Refunds (security deposit on withdrawal, excess payments) through approval | R1 | SCR-FEE-027 | P06.5 |  | Screen tests of listed screens | ☐ |
| FR-FEE-025 | Advance payments held as credit and applied to next vouchers | R1 | SCR-FEE-028 | P06.4 | P06.6 | Advance credit application test | ☐ |
| FR-FEE-026 | Student ledger and family ledger statements | R1 | SCR-FEE-028, SCR-FEE-029, SCR-FEE-041 | P06.6 |  | Screen tests of listed screens | ☐ |
| FR-FEE-027 | Struck-off rule: N months unpaid puts the student on a list for principal action; re-admission fee | R1 | SCR-STU-015 | P08.2 |  | Struck-off rule test | ☐ |
| FR-FEE-028 | Defaulter list with SMS reminders | R1 | SCR-FEE-032, SCR-FEE-033 | P06.6 |  | Screen tests of listed screens | ☐ |
| FR-FEE-029 | Fee clearance check used by withdrawal, certificates and admit cards (block configurable) | R1 | SCR-FEE-030, SCR-STU-014, SCR-EXM-007, SCR-STU-017 | P06.6 | P07.1, P08.2, P08.3 | Fee clearance check test | ☐ |
| FR-FEE-030 | Reports: daily, monthly, branch-wise, class-wise, head-wise collection; outstanding; defaulters; discounts; scholarships; refunds; cashier-wise daily cash; payment history; fee ledger | R1 | SCR-RPT-003 | P06.6 |  | Fee report totals test | ☐ |
| FR-FEE-031 | Every voucher, payment, reversal and refund posts to Accounting through the posting service | R1 | SCR-ACC-007 | P06.3 | P06.1 | assertJournalBalanced posting tests | ☐ |
| FR-FEE-032 | Online payment from guardian portal through gateway; paid only after verified server-side callback and status inquiry | R2 | SCR-FEE-038, SCR-FEE-039 | P11.2 |  | Payment trust test | ☐ |
| FR-FEE-033 | 1Bill consumer number on vouchers; bill inquiry and payment notification endpoints | R2 | SCR-FEE-034, SCR-FEE-037 | P11.1 | P06.3 | Screen tests of listed screens | ☐ |
| FR-FEE-034 | Gateway reconciliation job and transaction log | R2 | SCR-FEE-035 | P11.3 | P11.1 | Webhook flood test | ☐ |
| FR-FEE-035 | Bank-specific APIs and Raast | R3 | SCR-FEE-034, SCR-ACC-023 | P24.3 | P11.1 | Screen tests of listed screens | ☐ |

### FR-ACC

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-ACC-001 | Fiscal years (default July to June); open and close | R1 | SCR-ACC-001, SCR-ACC-002 | P06.1 |  | Screen tests of listed screens | ☐ |
| FR-ACC-002 | Chart of accounts tree (asset, liability, equity, income, expense; group and posting accounts), seeded school template | R1 | SCR-ACC-003, SCR-ACC-004 | P06.1 |  | Screen tests of listed screens | ☐ |
| FR-ACC-003 | Branch and department dimensions on journal lines | R1 | SCR-ACC-006 | P06.1 |  | Screen tests of listed screens | ☐ |
| FR-ACC-004 | Single posting service: the only way to create journals; used by Fee, Payroll, Expense and manual entries | R1 | SCR-ACC-005, SCR-ACC-007 | P06.1 |  | JournalPoster only-writer arch test | ☐ |
| FR-ACC-005 | Manual journal entries with approval above threshold | R1 | SCR-ACC-006 | P06.1 |  | Screen tests of listed screens | ☐ |
| FR-ACC-006 | Entry numbering per fiscal year; statuses `draft`, `posted`, `reversed` | R1 | SCR-ACC-005, SCR-ACC-008 | P06.1 |  | Screen tests of listed screens | ☐ |
| FR-ACC-007 | Cashbook, bank book, general ledger with running balance | R1 | SCR-ACC-009, SCR-ACC-010 | P06.1 |  | Screen tests of listed screens | ☐ |
| FR-ACC-008 | Trial balance; fee receivables aging | R1 | SCR-ACC-011, SCR-ACC-012 | P06.1 | P06.6 | Screen tests of listed screens | ☐ |
| FR-ACC-009 | Opening balances entry at go-live | R1 | SCR-ACC-013, SCR-FEE-031 | P06.1 | P06.6 | Screen tests of listed screens | ☐ |
| FR-ACC-010 | Bank accounts register | R1 | SCR-ACC-014 | P06.1 |  | Screen tests of listed screens | ☐ |
| FR-ACC-011 | Monthly period locks | R1 | SCR-CORE-001 | P06.1 | P01.6 | Screen tests of listed screens | ☐ |
| FR-ACC-012 | Expense vouchers with attachments and approval | R2 | SCR-ACC-015 | P12.1 |  | Screen tests of listed screens | ☐ |
| FR-ACC-013 | Vendors, purchase bills, vendor payments | R2 | SCR-ACC-016, SCR-ACC-017, SCR-ACC-018 | P12.1 |  | Screen tests of listed screens | ☐ |
| FR-ACC-014 | Bank statement import and reconciliation matching | R2 | SCR-ACC-019, SCR-ACC-020 | P12.2 |  | Screen tests of listed screens | ☐ |
| FR-ACC-015 | Income statement and balance sheet, per branch and consolidated | R2 | SCR-ACC-021 | P12.3 |  | Screen tests of listed screens | ☐ |
| FR-ACC-016 | Advanced consolidation and inter-branch transfers | R3 | SCR-ACC-022, SCR-ACC-021 | P23.1 | P12.3 | Screen tests of listed screens | ☐ |

### FR-EXM

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-EXM-001 | Exam terms per session: weekly test, monthly test, class test, midterm, final, send-up, mock, practical, oral; weightage for cumulative result | R1 | SCR-EXM-001 | P07.1 |  | Screen tests of listed screens | ☐ |
| FR-EXM-002 | Exams for selected classes and sections within a term | R1 | SCR-EXM-002 | P07.1 |  | Screen tests of listed screens | ☐ |
| FR-EXM-003 | Papers per subject: date, start time, duration, room, total marks, pass marks; components (theory, practical, oral) | R1 | SCR-EXM-003 | P07.1 |  | Screen tests of listed screens | ☐ |
| FR-EXM-004 | Date sheet builder with clash detection (one class two papers same slot, room capacity) and publish with notification | R1 | SCR-EXM-004, SCR-EXM-005, SCR-EXM-006 | P07.1 |  | Screen tests of listed screens | ☐ |
| FR-EXM-005 | Admit cards / roll number slips with fee clearance check | R1 | SCR-EXM-007, SCR-EXM-008, SCR-EXM-020, SCR-EXM-021 | P07.1 |  | Screen tests of listed screens | ☐ |
| FR-EXM-006 | Marks entry grid per section per paper for the assigned teacher; codes per DEC-28; auto-save; validation against totals | R1 | SCR-EXM-009, SCR-EXM-010, SCR-EXM-018 | P07.1 |  | Screen tests of listed screens | ☐ |
| FR-EXM-007 | Submit marks for approval; submission locks the grid for the teacher | R1 | SCR-EXM-011 | P07.1 |  | Screen tests of listed screens | ☐ |
| FR-EXM-008 | Review by HOD or EXC: approve or return with comment | R1 | SCR-EXM-012, SCR-EXM-013 | P07.1 |  | Screen tests of listed screens | ☐ |
| FR-EXM-009 | Changes after approval only through `marks_revision` request; revisions stored | R1 | SCR-EXM-015, SCR-EXM-016 | P07.1 |  | Screen tests of listed screens | ☐ |
| FR-EXM-010 | Marks completion tracker: pending sections and subjects per exam | R1 | SCR-EXM-017 | P07.1 |  | Screen tests of listed screens | ☐ |
| FR-EXM-011 | Marks import via IMX | R1 | SCR-EXM-014 | P07.1 |  | Screen tests of listed screens | ☐ |
| FR-EXM-012 | Invigilation duty assignment | R2 | SCR-EXM-019 | P16.3 |  | Screen tests of listed screens | ☐ |
| FR-EXM-013 | Board exam support: board roll numbers, board result entry, analysis | R2 | SCR-EXT-023 | P16.5 |  | Screen tests of listed screens | ☐ |

### FR-RES

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-RES-001 | Grading schemes per institution and level: bands (min %, grade, optional GPA point, remark en/ur) | R1 | SCR-RES-001, SCR-RES-002 | P07.2 |  | Screen tests of listed screens | ☐ |
| FR-RES-002 | Result rules: subject pass %, overall pass %, maximum failed subjects for "promoted with compartment", grace marks limits, rounding, term weightage, minimum attendance % for eligibility | R1 | SCR-RES-003 | P07.2 |  | Screen tests of listed screens | ☐ |
| FR-RES-003 | Ranking: enabled flag, method (standard or dense), scope (section, class, branch), tie-breakers, visibility to students and guardians | R1 | SCR-RES-003, SCR-RES-015 | P07.2 |  | Screen tests of listed screens | ☐ |
| FR-RES-004 | Process result batch (queued): totals, percentage, grade, GPA, status (`pass`, `fail`, `compartment`, `absent`, `withheld`), positions | R1 | SCR-RES-004, SCR-RES-005 | P07.2 |  | Screen tests of listed screens | ☐ |
| FR-RES-005 | Review screen with anomalies (missing marks, absent in all papers) | R1 | SCR-RES-006 | P07.2 |  | Screen tests of listed screens | ☐ |
| FR-RES-006 | Approval for publication through APR | R1 | SCR-RES-009, SCR-RES-010 | P07.2 |  | Screen tests of listed screens | ☐ |
| FR-RES-007 | Scheduled publication; notifications to students and guardians | R1 | SCR-RES-010, SCR-RES-017, SCR-RES-019 | P07.2 |  | Screen tests of listed screens | ☐ |
| FR-RES-008 | Withhold individual results with reason (fee, discipline) | R1 | SCR-RES-008 | P07.2 |  | Screen tests of listed screens | ☐ |
| FR-RES-009 | Re-processing after revision creates a new result version | R1 | SCR-RES-006, SCR-RES-011 | P07.2 |  | Result version test | ☐ |
| FR-RES-010 | Report cards: single and bulk per section; teacher remarks, attendance summary, signatures | R1 | SCR-RES-012, SCR-RES-013, SCR-RES-018 | P07.2 |  | Screen tests of listed screens | ☐ |
| FR-RES-011 | Cumulative annual result feeding promotion eligibility | R1 | SCR-RES-007 | P07.2 |  | Screen tests of listed screens | ☐ |
| FR-RES-012 | Basic analytics: subject averages, pass rates, top performers | R1 | SCR-RES-014 | P07.2 |  | Screen tests of listed screens | ☐ |
| FR-RES-013 | Result re-check requests from guardians | R2 | SCR-RES-016, SCR-RES-020 | P16.5 |  | Screen tests of listed screens | ☐ |
| FR-RES-014 | Advanced analytics (trends, branch comparison) | R3 | SCR-RPT-010, SCR-RPT-011 | P23.1 |  | Screen tests of listed screens | ☐ |

### FR-TTB

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-TTB-001 | Bell schedules per branch and shift with day patterns (e.g., short Friday), periods with start, end, break flag | R1 | SCR-TTB-001 | P09.1 |  | Screen tests of listed screens | ☐ |
| FR-TTB-002 | Timetable per section per session with versions (`draft`, `active`) and effective date | R1 | SCR-TTB-002, SCR-TTB-004 | P09.1 |  | Screen tests of listed screens | ☐ |
| FR-TTB-003 | Entries: day, period, subject, teacher, room | R1 | SCR-TTB-003 | P09.1 |  | Screen tests of listed screens | ☐ |
| FR-TTB-004 | Validations: teacher double booking (across branches), room double booking, section double booking, weekly load versus teaching assignment periods, maximum consecutive periods | R1 | SCR-TTB-003, SCR-TTB-004 | P09.1 |  | TimetableConflictDetector tests | ☐ |
| FR-TTB-005 | Views: section, teacher, room, subject; "my timetable" for teacher, student, guardian | R1 | SCR-TTB-005, SCR-TTB-006, SCR-TTB-008, SCR-TTB-010, SCR-TTB-011 | P09.1 |  | Screen tests of listed screens | ☐ |
| FR-TTB-006 | Print and PDF | R1 | SCR-TTB-007 | P09.1 |  | Screen tests of listed screens | ☐ |
| FR-TTB-007 | Copy from previous session | R1 | SCR-TTB-002 | P09.1 |  | Screen tests of listed screens | ☐ |
| FR-TTB-008 | Substitutions: absent teacher's periods listed, free teachers suggested, assignment, notification, workload record | R2 | SCR-TTB-009 | P16.3 |  | Screen tests of listed screens | ☐ |
| FR-TTB-009 | Assisted auto-generation | R3 | SCR-TTB-012 | P23.3 |  | Screen tests of listed screens | ☐ |

### FR-COMM

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-COMM-001 | Announcements, notices and circulars with audience targeting (organization, branch, class, section, role, specific guardians or staff), attachments, schedule, expiry, pin | R1 | SCR-COMM-001, SCR-COMM-002 | P09.2 |  | Screen tests of listed screens | ☐ |
| FR-COMM-002 | Acknowledgment required option with read receipts | R1 | SCR-COMM-003 | P09.2 |  | Screen tests of listed screens | ☐ |
| FR-COMM-003 | Optional SMS or WhatsApp copy with cost estimate | R1 | SCR-COMM-002 | P09.2 |  | Screen tests of listed screens | ☐ |
| FR-COMM-004 | Notice board per portal | R1 | SCR-COMM-013, SCR-COMM-017 | P09.2 |  | Screen tests of listed screens | ☐ |
| FR-COMM-005 | Daily diary: per section per date, entries per subject (homework text, reminders, attachments) | R1 | SCR-COMM-004, SCR-COMM-005, SCR-COMM-014 | P09.2 |  | Screen tests of listed screens | ☐ |
| FR-COMM-006 | Optional coordinator review before diary publishes | R1 | SCR-COMM-006 | P09.2 |  | Screen tests of listed screens | ☐ |
| FR-COMM-007 | Guardian-teacher messaging limited to assigned teachers, within set hours, moderated, logged, no deletion | R2 | SCR-COMM-007, SCR-COMM-008, SCR-COMM-009, SCR-COMM-015 | P16.1 |  | Screen tests of listed screens | ☐ |
| FR-COMM-008 | PTM: event, teacher slots, guardian booking, attendance and notes | R2 | SCR-COMM-010, SCR-COMM-011, SCR-COMM-012, SCR-COMM-016 | P16.1 |  | Screen tests of listed screens | ☐ |
| FR-COMM-009 | Complaints handled in Administration (R3) | R3 | SCR-ADN-007 | P22.1 |  | Screen tests of listed screens | ☐ |

### FR-LMS

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-LMS-001 | Course per class subject per session, created automatically | R2 | SCR-LMS-001, SCR-LMS-004 | P14.1 |  | Screen tests of listed screens | ☐ |
| FR-LMS-002 | Chapters and topics with ordering | R2 | SCR-LMS-002 | P14.1 |  | Screen tests of listed screens | ☐ |
| FR-LMS-003 | Lecture fields per brief: title, description, subject, class, sections, chapter, topic, teacher, attachments, video URL or upload, publish and availability dates, status, visibility, comments setting | R2 | SCR-LMS-003 | P14.1 |  | Screen tests of listed screens | ☐ |
| FR-LMS-004 | Resource types: uploaded video (size-capped), YouTube/Vimeo embeds, PDF, DOCX, PPTX, images, audio, links, rich text | R2 | SCR-LMS-003 | P14.1 |  | Screen tests of listed screens | ☐ |
| FR-LMS-005 | Copy lectures to other sections or the next session | R2 | SCR-LMS-002 | P14.1 |  | Screen tests of listed screens | ☐ |
| FR-LMS-006 | Student view, download (if allowed), progress tracking | R2 | SCR-LMS-008, SCR-LMS-009 | P14.2 |  | Screen tests of listed screens | ☐ |
| FR-LMS-007 | Engagement analytics per lecture and per student | R2 | SCR-LMS-007 | P14.2 |  | Screen tests of listed screens | ☐ |
| FR-LMS-008 | Discussions: questions, comments, replies; teacher answers, pins, marks resolved, disables comments, moderates | R2 | SCR-LMS-005, SCR-LMS-010 | P14.2 |  | Screen tests of listed screens | ☐ |
| FR-LMS-009 | Report inappropriate content; moderation queue | R2 | SCR-LMS-006 | P14.2 |  | Screen tests of listed screens | ☐ |
| FR-LMS-010 | Guardian read-only access to lectures and child's progress | R2 | SCR-LMS-011 | P14.2 |  | Screen tests of listed screens | ☐ |

### FR-HWK

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-HWK-001 | Create homework: type (daily, assignment, project), individual or group, subject, sections or selected students, instructions, attachments, assign date, due date and time, optional max marks | R2 | SCR-HWK-001, SCR-HWK-002 | P15.1 |  | Screen tests of listed screens | ☐ |
| FR-HWK-002 | Submission modes: file, image, PDF, text, or offline (no upload) | R2 | SCR-HWK-002, SCR-HWK-007 | P15.1 |  | Screen tests of listed screens | ☐ |
| FR-HWK-003 | Resubmission and late rules (allow with flag, penalty, or disallow) | R2 | SCR-HWK-002 | P15.1 |  | Screen tests of listed screens | ☐ |
| FR-HWK-004 | Student or guardian submits; phone camera images compressed | R2 | SCR-HWK-007 | P15.1 |  | Screen tests of listed screens | ☐ |
| FR-HWK-005 | Review queue: grade, feedback, return for resubmission | R2 | SCR-HWK-003, SCR-HWK-004 | P15.1 |  | Screen tests of listed screens | ☐ |
| FR-HWK-006 | Per-student status: `pending`, `submitted`, `late`, `reviewed`, `returned`, `missing` | R2 | SCR-HWK-003, SCR-HWK-006 | P15.1 |  | Screen tests of listed screens | ☐ |
| FR-HWK-007 | Guardian monitoring and deadline reminders | R2 | SCR-HWK-008 | P15.1 |  | Screen tests of listed screens | ☐ |
| FR-HWK-008 | Submission rate reports per section and student | R2 | SCR-HWK-005 | P15.1 |  | Screen tests of listed screens | ☐ |

### FR-ASM

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-ASM-001 | Question bank per subject and level with chapter, topic, difficulty, tags; sharing within department or organization | R2 | SCR-ASM-001, SCR-ASM-002 | P15.2 |  | Screen tests of listed screens | ☐ |
| FR-ASM-002 | Question types: MCQ, multi-select, true/false, short, long, fill in the blanks (accepted answers), matching, image-based | R2 | SCR-ASM-003 | P15.2 |  | Screen tests of listed screens | ☐ |
| FR-ASM-003 | Question import from template | R2 | SCR-ASM-004 | P15.2 |  | Screen tests of listed screens | ☐ |
| FR-ASM-004 | Quiz builder: manual selection or random rules (count by chapter and difficulty), time limit, window, attempts, shuffling, negative marking, result and answer display policy | R2 | SCR-ASM-005, SCR-ASM-006 | P15.3 |  | Screen tests of listed screens | ☐ |
| FR-ASM-005 | Online exam mode: single attempt, full-screen prompt, tab-switch logging (recorded only, not presented as prevention) | R2 | SCR-ASM-006, SCR-ASM-007 | P15.3 |  | Screen tests of listed screens | ☐ |
| FR-ASM-006 | Attempt engine: server-side timer, autosave, auto-submit at time end, resume after disconnection within window | R2 | SCR-ASM-011, SCR-ASM-010 | P15.3 |  | Server deadline and auto-submit tests | ☐ |
| FR-ASM-007 | Automatic grading of objective types; manual grading queue for subjective | R2 | SCR-ASM-008 | P15.3 |  | Screen tests of listed screens | ☐ |
| FR-ASM-008 | Result publication | R2 | SCR-ASM-009, SCR-ASM-012, SCR-ASM-013 | P15.3 |  | Screen tests of listed screens | ☐ |
| FR-ASM-009 | Analytics: score distribution, item difficulty | R2 | SCR-ASM-009 | P15.3 |  | Screen tests of listed screens | ☐ |
| FR-ASM-010 | Transfer online exam scores into an EXM paper's marks entry | R2 | SCR-ASM-009, SCR-EXM-010 | P15.3 | P07.1 | Draft import test (DEC-288) | ☐ |

### FR-PAY

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-PAY-001 | Components: earnings (basic, house rent, conveyance, medical, special) and deductions (absence, late, EOBI, PESSI, income tax, provident fund, loan, advance) | R2 | SCR-PAY-001 | P13.2 |  | Screen tests of listed screens | ☐ |
| FR-PAY-002 | Fixed, percentage of basic, or slab-based calculation | R2 | SCR-PAY-001, SCR-PAY-002 | P13.2 |  | Calculator tests (DEC-287) | ☐ |
| FR-PAY-003 | Salary structures per designation or grade; employee salary with effective-dated history | R2 | SCR-PAY-003, SCR-PAY-004 | P13.2 |  | Screen tests of listed screens | ☐ |
| FR-PAY-004 | Monthly payroll run per branch or organization: pull attendance summary, unpaid leave, late conversions; compute; adjust with reason | R2 | SCR-PAY-005, SCR-PAY-006, SCR-PAY-007 | P13.3 |  | Determinism test | ☐ |
| FR-PAY-005 | Approval, finalization (lock), journal posting | R2 | SCR-PAY-009 | P13.3 |  | Screen tests of listed screens | ☐ |
| FR-PAY-006 | Payment recording: bank advice export, cash, cheque | R2 | SCR-PAY-010 | P13.3 |  | Screen tests of listed screens | ☐ |
| FR-PAY-007 | Payslips PDF and staff portal access | R2 | SCR-PAY-008, SCR-PAY-013 | P13.4 |  | Screen tests of listed screens | ☐ |
| FR-PAY-008 | Advances and loans with installment recovery | R2 | SCR-PAY-011, SCR-PAY-012, SCR-PAY-014 | P13.3 |  | Screen tests of listed screens | ☐ |
| FR-PAY-009 | Increments, arrears, final settlement | R2 | SCR-PAY-007, SCR-HR-005 | P13.3 | P05.1 | Screen tests of listed screens | ☐ |
| FR-PAY-010 | Reports: payroll register, bank advice, deductions (EOBI, PESSI, tax), cost by branch and department | R2 | SCR-RPT-003 | P13.4 | P06.6 | Screen tests of listed screens | ☐ |

### FR-LIB

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-LIB-001 | Catalog: title, ISBN, authors, publisher, category, edition, language, shelf | R3 | SCR-LIB-001, SCR-LIB-002 | P19.1 |  | Screen tests of listed screens | ☐ |
| FR-LIB-002 | Copies with accession number and barcode; label printing | R3 | SCR-LIB-003 | P19.1 |  | Screen tests of listed screens | ☐ |
| FR-LIB-003 | Members created automatically for students and staff with type-based limits | R3 | SCR-LIB-004 | P19.1 |  | Screen tests of listed screens | ☐ |
| FR-LIB-004 | Issue and return by barcode scan; due dates; renewals; reservations | R3 | SCR-LIB-005, SCR-LIB-006, SCR-LIB-009, SCR-LIB-010, SCR-LIB-011 | P19.1 |  | Screen tests of listed screens | ☐ |
| FR-LIB-005 | Late fines (per day) and lost or damaged charges posted as fee items or cash | R3 | SCR-LIB-007, SCR-LIB-008 | P19.1 |  | Screen tests of listed screens | ☐ |
| FR-LIB-006 | Stock verification | R3 | SCR-LIB-003 | P19.1 |  | Verification report test (DEC-297) | ☐ |
| FR-LIB-007 | Reports: issued, overdue, popular, fines; clearance check for withdrawal | R3 | SCR-RPT-003, SCR-LIB-007 | P19.1 | P06.6 | Screen tests of listed screens | ☐ |

### FR-TRN

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-TRN-001 | Routes and ordered stops with pickup and drop times; fare per stop or zone | R2 | SCR-TRN-001, SCR-TRN-002, SCR-TRN-003 | P17.1 |  | Screen tests of listed screens | ☐ |
| FR-TRN-002 | Student allocation (route, stop, pickup/drop/both, dates) adds transport fee items to vouchers | R2 | SCR-TRN-004, SCR-TRN-005 | P17.1 |  | Screen tests of listed screens | ☐ |
| FR-TRN-003 | Capacity check per vehicle and route | R2 | SCR-TRN-005 | P17.1 |  | Concurrent allocation at capacity test | ☐ |
| FR-TRN-004 | Route sheets and emergency contact lists | R2 | SCR-TRN-006 | P17.1 |  | Screen tests of listed screens | ☐ |
| FR-TRN-005 | Vehicles: registration, type, capacity, ownership (own or contractor), documents with expiry (fitness, route permit, insurance, token tax) | R3 | SCR-TRN-007, SCR-TRN-008 | P20.1 |  | Screen tests of listed screens | ☐ |
| FR-TRN-006 | Drivers and conductors with license expiry alerts | R3 | SCR-TRN-009, SCR-TRN-010 | P20.1 |  | Screen tests of listed screens | ☐ |
| FR-TRN-007 | Maintenance and fuel logs | R3 | SCR-TRN-011, SCR-TRN-012 | P20.2 |  | Screen tests of listed screens | ☐ |
| FR-TRN-008 | Transport attendance (boarding) by transport staff phone | R3 | SCR-TRN-013 | P20.1 |  | Screen tests of listed screens | ☐ |
| FR-TRN-009 | Optional GPS provider adapter; live map (Leaflet); guardian ETA | R3 | SCR-TRN-014, SCR-TRN-015, SCR-TRN-016 | P20.2 | P17.1 | Screen tests of listed screens | ☐ |

### FR-INV

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-INV-001 | Items, categories, units | R3 | SCR-INV-001, SCR-INV-002 | P21.1 |  | Screen tests of listed screens | ☐ |
| FR-INV-002 | Stock locations per branch | R3 | SCR-INV-003 | P21.1 |  | Screen tests of listed screens | ☐ |
| FR-INV-003 | Stock in, issue, transfer between branches, adjustments with approval | R3 | SCR-INV-005, SCR-INV-006, SCR-INV-007, SCR-INV-008 | P21.1 |  | Screen tests of listed screens | ☐ |
| FR-INV-004 | Reorder level alerts | R3 | SCR-INV-004 | P21.1 |  | Screen tests of listed screens | ☐ |
| FR-INV-005 | Fixed asset register: tag, purchase date, cost, location, custodian; assignment and return | R3 | SCR-INV-010, SCR-INV-011, SCR-INV-012 | P21.1 |  | Screen tests of listed screens | ☐ |
| FR-INV-006 | Maintenance requests and logs | R3 | SCR-INV-013, SCR-INV-014 | P22.1 |  | Screen tests of listed screens | ☐ |
| FR-INV-007 | Stock and asset reports | R3 | SCR-INV-009, SCR-RPT-003 | P21.1 | P06.6 | Screen tests of listed screens | ☐ |

### FR-ADN

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-ADN-001 | Academic calendar per institution or branch: holidays (public, institutional, notified closure), events, exam periods, PTMs, vacations | R1 | SCR-ADN-001, SCR-ADN-002, SCR-ADN-009, SCR-ADN-012 | P09.2 |  | Screen tests of listed screens | ☐ |
| FR-ADN-002 | Bulk closure declaration (smog, heat, security) updating attendance expectations and timetable for the dates | R1 | SCR-ADN-003 | P09.2 |  | Screen tests of listed screens | ☐ |
| FR-ADN-003 | Events with audience and optional RSVP | R1 | SCR-ADN-002 | P09.2 |  | Screen tests of listed screens | ☐ |
| FR-ADN-004 | Bulk ID card printing for students and staff with photo and QR | R1 | SCR-ADN-004 | P09.2 |  | Screen tests of listed screens | ☐ |
| FR-ADN-005 | Gate passes: early pickup verified against pickup authorizations; guardian notified | R2 | SCR-ADN-005, SCR-ADN-010 | P16.3 |  | Screen tests of listed screens | ☐ |
| FR-ADN-006 | Visitor management: entry, purpose, host, badge, check-out, optional photo | R3 | SCR-ADN-006 | P22.1 |  | Screen tests of listed screens | ☐ |
| FR-ADN-007 | Complaints: category, assignment, SLA, resolution, satisfaction | R3 | SCR-ADN-007, SCR-ADN-008, SCR-ADN-011 | P22.1 |  | Screen tests of listed screens | ☐ |

### FR-DSC

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-DSC-001 | Incidents: date, students, category, severity, description, witnesses, action | R2 | SCR-DSC-001, SCR-DSC-002, SCR-DSC-003, SCR-DSC-007 | P17.2 |  | Screen tests of listed screens | ☐ |
| FR-DSC-002 | Actions: warning, detention, suspension, guardian call; guardian notification by severity | R2 | SCR-DSC-004 | P17.2 |  | Screen tests of listed screens | ☐ |
| FR-DSC-003 | Teacher remarks (positive and negative) | R2 | SCR-DSC-005 | P17.2 |  | Screen tests of listed screens | ☐ |
| FR-DSC-004 | Rewards, achievements, co-curricular participation | R2 | SCR-DSC-006, SCR-DSC-010, SCR-DSC-011 | P17.2 |  | Screen tests of listed screens | ☐ |
| FR-DSC-005 | Counseling records visible only with `discipline.counseling.view`; never shown to guardians or teachers | R3 | SCR-DSC-008, SCR-DSC-009 | P22.2 |  | Counseling leak suite | ☐ |
| FR-DSC-006 | Reports | R2 | SCR-RPT-003 | P17.2 | P06.6 | Screen tests of listed screens | ☐ |

### FR-RPT

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-RPT-001 | Role dashboards (owner, principal, teacher, student, guardian, accountant, HR, exam controller, admin) with widgets defined in D10 | R1 | SCR-RPT-001, SCR-RPT-008, SCR-RPT-009 | P09.3 |  | Screen tests of listed screens | ☐ |
| FR-RPT-002 | Consolidated and per-branch views | R1 | SCR-RPT-001 | P09.3 |  | Screen tests of listed screens | ☐ |
| FR-RPT-003 | KPI snapshots updated by scheduled jobs (every 15 minutes for today's attendance and collection, nightly for others) with "last updated" time | R1 | SCR-RPT-007 | P09.3 |  | Snapshot schedule test | ☐ |
| FR-RPT-004 | Drill-down from any KPI to the filtered list behind it | R1 | SCR-RPT-001, SCR-RPT-003 | P09.3 | P06.6 | Screen tests of listed screens | ☐ |
| FR-RPT-005 | Report catalog: each module registers its reports with permission, filters and columns | R1 | SCR-RPT-002, SCR-RPT-003 | P09.3 | P06.6 | Screen tests of listed screens | ☐ |
| FR-RPT-006 | Saved filters per user | R1 | SCR-RPT-004 | P09.3 |  | Screen tests of listed screens | ☐ |
| FR-RPT-007 | Basic branch comparison table (students, attendance %, collection %, outstanding) | R1 | SCR-RPT-006 | P09.3 |  | Screen tests of listed screens | ☐ |
| FR-RPT-008 | Scheduled reports by email | R2 | SCR-RPT-005 | P16.3 |  | Screen tests of listed screens | ☐ |
| FR-RPT-009 | Advanced analytics and trends | R3 | SCR-RPT-011 | P23.1 |  | Screen tests of listed screens | ☐ |

### FR-EXT

| FR | Requirement | Rel | Screens | Step | Also in | Verification | Verified |
|---|---|---|---|---|---|---|---|
| FR-EXT-001 | Awarding bodies: seeded catalog (Cambridge International, Pearson Edexcel, OxfordAQA, nine Punjab BISE boards, FBISE) plus organization-specific additions; centre number per branch per awarding body | R1 | SCR-EXT-001, SCR-EXT-003 | P07.3 |  | Screen tests of listed screens | ☐ |
| FR-EXT-002 | External syllabus codes per subject, awarding body and qualification with components and validity series (DEC-82) | R1 | SCR-EXT-002 | P07.3 |  | Screen tests of listed screens | ☐ |
| FR-EXT-003 | Candidate identifiers per student per awarding body (candidate number, UCI or board registration number) | R1 | SCR-EXT-004 | P07.3 |  | Screen tests of listed screens | ☐ |
| FR-EXT-004 | Exam series: awarding body, name (e.g., "June 2027"), entry deadlines per stage (normal, late stage 1, late stage 2), retake window, results date | R2 | SCR-EXT-007, SCR-EXT-008 | P16.5 |  | Screen tests of listed screens | ☐ |
| FR-EXT-005 | Fee schedules per series and syllabus or component option, per entry stage, in PKR (optional reference to foreign currency amount and rate) | R2 | SCR-EXT-009 | P16.5 |  | Screen tests of listed screens | ☐ |
| FR-EXT-006 | Entry planning: generate draft entries from student subjects for a series; teacher or coordinator review; guardian confirmation in portal (optional) | R2 | SCR-EXT-010, SCR-EXT-011, SCR-EXT-012, SCR-EXT-024 | P16.5 |  | Screen tests of listed screens | ☐ |
| FR-EXT-007 | Entry fee vouchers generated from confirmed entries (ad-hoc vouchers through Fee); unpaid entries flagged before each deadline | R2 | SCR-EXT-013 | P16.5 |  | Pass-through posting test (DEC-84) | ☐ |
| FR-EXT-008 | Entry statuses: `draft`, `confirmed`, `paid`, `submitted`, `withdrawn`, `absent`, `resulted`; withdrawal after submission follows the awarding body's refund rules and an approval | R2 | SCR-EXT-010, SCR-EXT-015 | P16.5 |  | Screen tests of listed screens | ☐ |
| FR-EXT-009 | Entry export in a format suitable for upload or manual entry into the awarding body's portal; national board registration exports | R2 | SCR-EXT-014 | P16.5 |  | Screen tests of listed screens | ☐ |
| FR-EXT-010 | Results import from statement-of-results files (CSV/Excel mapping) or manual entry; syllabus grades and component grades | R2 | SCR-EXT-016 | P16.5 |  | Screen tests of listed screens | ☐ |
| FR-EXT-011 | Results publication to students and guardians after review; result analytics (grade distribution per subject and teacher, value added against predicted grades) | R2 | SCR-EXT-017, SCR-EXT-018, SCR-EXT-025 | P16.5 |  | Screen tests of listed screens | ☐ |
| FR-EXT-012 | Enquiries about results (re-mark requests): student or guardian request, school decision, fee voucher, outcome recording | R2 | SCR-EXT-019, SCR-EXT-025 | P16.5 |  | Screen tests of listed screens | ☐ |
| FR-EXT-013 | Certificates register: received from awarding body, handed over to student with signature date | R2 | SCR-EXT-020 | P16.5 |  | Screen tests of listed screens | ☐ |
| FR-EXT-014 | Predicted grades per student, subject and series; HOD review; export for university references (DEC-85) | R1 | SCR-EXT-005, SCR-EXT-006 | P07.3 |  | Screen tests of listed screens | ☐ |
| FR-EXT-015 | IBCC equivalence calculator: conversion rules per session (grade to marks, subject-specific A* values), eight-subject and three-subject rules, result shown as marks out of 1100; used for cross-track progression and merit lists | R2 | SCR-EXT-021, SCR-EXT-022 | P16.5 |  | Screen tests of listed screens | ☐ |
| FR-EXT-016 | National boards: board registration data export, board roll numbers, board results entry and analysis (replaces FR-EXM-013) | R2 | SCR-EXT-023 | P16.5 |  | Screen tests of listed screens | ☐ |

## 15.11 Screen Build Map (reverse check)

Every screen in the D10 inventory (502) plus the 13 screens added by D14 DEC-305 and DEC-310 is listed against the step that builds it. `ScreenCoverageTest` for a release must find every screen of that release in this map, and every step must show its screens in its report. A screen built in one step and extended later lists only the first step here; extensions appear in the later step reports.

| Step | Screens built | Count |
|---|---|---|
| P01.3 | SCR-IAM-012, SCR-IAM-013, SCR-IAM-014, SCR-IAM-015, SCR-IAM-016, SCR-IAM-018, SCR-IAM-019, SCR-IAM-020, SCR-IAM-021, SCR-IAM-022, SCR-IAM-023, SCR-IAM-024, SCR-IAM-025, SCR-IAM-026 | 14 |
| P01.4 | SCR-IAM-001, SCR-IAM-002, SCR-IAM-003, SCR-IAM-004, SCR-IAM-005, SCR-IAM-006, SCR-IAM-008, SCR-IAM-009, SCR-IAM-010, SCR-IAM-011, SCR-PLT-002 | 11 |
| P01.5 | SCR-ORG-001, SCR-ORG-002, SCR-ORG-003, SCR-ORG-004, SCR-ORG-005, SCR-ORG-006, SCR-ORG-007, SCR-ORG-008, SCR-ORG-009, SCR-ORG-010, SCR-ORG-011, SCR-PLT-003, SCR-PLT-004, SCR-PLT-005, SCR-PLT-006, SCR-PLT-007, SCR-PLT-008, SCR-PLT-009, SCR-PLT-010, SCR-PLT-011, SCR-PLT-012, SCR-PLT-013, SCR-PLT-014, SCR-PLT-015, SCR-PLT-016, SCR-PLT-017, SCR-PLT-018 | 27 |
| P01.6 | SCR-AUD-001, SCR-AUD-002, SCR-AUD-003, SCR-AUD-004, SCR-AUD-005, SCR-CORE-001, SCR-CORE-002, SCR-CORE-004, SCR-CORE-005, SCR-PLT-001, SCR-PLT-019, SCR-SET-001, SCR-SET-002, SCR-SET-003, SCR-SET-004, SCR-SET-005, SCR-SET-006 | 17 |
| P02.1 | SCR-APR-001, SCR-APR-002, SCR-APR-003, SCR-APR-004, SCR-APR-005, SCR-APR-006, SCR-APR-007, SCR-APR-008, SCR-CORE-003 | 9 |
| P02.2 | SCR-NTF-001, SCR-NTF-002, SCR-NTF-003, SCR-NTF-004, SCR-NTF-005, SCR-NTF-006, SCR-NTF-007, SCR-NTF-008, SCR-NTF-009 | 9 |
| P02.3 | SCR-IAM-017, SCR-IMX-001, SCR-IMX-003, SCR-IMX-004, SCR-IMX-005 | 5 |
| P02.4 | SCR-DOC-001, SCR-DOC-002, SCR-DOC-004, SCR-DOC-005 | 4 |
| P03.1 | SCR-ACD-001, SCR-ACD-002, SCR-ACD-004, SCR-ACD-006, SCR-ACD-007, SCR-ACD-008 | 6 |
| P03.2 | SCR-ACD-003, SCR-ACD-005, SCR-ACD-009, SCR-ACD-010, SCR-ACD-011, SCR-ACD-012, SCR-ACD-015, SCR-ACD-016 | 8 |
| P04.1 | SCR-GRD-001, SCR-GRD-002, SCR-GRD-003, SCR-GRD-004, SCR-GRD-005, SCR-GRD-006, SCR-GRD-007, SCR-GRD-008, SCR-GRD-009, SCR-STU-001, SCR-STU-002, SCR-STU-003, SCR-STU-004, SCR-STU-005, SCR-STU-006, SCR-STU-007, SCR-STU-018, SCR-STU-020, SCR-STU-021, SCR-STU-022, SCR-STU-023 | 21 |
| P04.2 | SCR-ADM-001, SCR-ADM-002, SCR-ADM-003, SCR-ADM-004, SCR-ADM-005, SCR-ADM-006, SCR-ADM-007, SCR-ADM-008, SCR-ADM-009, SCR-ADM-010, SCR-ADM-011 | 11 |
| P04.3 | SCR-APR-009, SCR-GRD-010, SCR-GRD-011, SCR-IAM-007, SCR-STU-019, SCR-STU-024, SCR-STU-025, SCR-STU-026 | 8 |
| P04.4 | SCR-IMX-002 | 1 |
| P05.1 | SCR-ACD-013, SCR-ACD-014, SCR-ACD-017, SCR-HR-001, SCR-HR-002, SCR-HR-003, SCR-HR-004, SCR-HR-005, SCR-HR-006, SCR-HR-007, SCR-HR-008, SCR-HR-009, SCR-HR-010, SCR-HR-011, SCR-HR-012, SCR-HR-013 | 16 |
| P05.2 | SCR-ATT-001, SCR-ATT-002, SCR-ATT-004, SCR-ATT-005, SCR-ATT-006, SCR-ATT-007, SCR-ATT-016, SCR-ATT-024, SCR-ATT-025 | 9 |
| P05.3 | SCR-ATT-008, SCR-ATT-009, SCR-ATT-010, SCR-ATT-011, SCR-ATT-012, SCR-ATT-013, SCR-ATT-014, SCR-ATT-015, SCR-ATT-017, SCR-ATT-018, SCR-ATT-019, SCR-ATT-020, SCR-ATT-021, SCR-ATT-022 | 14 |
| P05.4 | SCR-ATT-023, SCR-LVE-001, SCR-LVE-002, SCR-LVE-003, SCR-LVE-004, SCR-LVE-005, SCR-LVE-006, SCR-LVE-007, SCR-LVE-008, SCR-LVE-009, SCR-LVE-010, SCR-LVE-011, SCR-LVE-012 | 13 |
| P06.1 | SCR-ACC-001, SCR-ACC-002, SCR-ACC-003, SCR-ACC-004, SCR-ACC-005, SCR-ACC-006, SCR-ACC-007, SCR-ACC-008, SCR-ACC-009, SCR-ACC-010, SCR-ACC-011, SCR-ACC-013, SCR-ACC-014 | 13 |
| P06.2 | SCR-FEE-001, SCR-FEE-002, SCR-FEE-003, SCR-FEE-004, SCR-FEE-005, SCR-FEE-006, SCR-FEE-007, SCR-FEE-008, SCR-FEE-009 | 9 |
| P06.3 | SCR-FEE-011, SCR-FEE-012, SCR-FEE-013, SCR-FEE-014, SCR-FEE-015, SCR-FEE-016, SCR-FEE-036, SCR-FEE-037, SCR-FEE-042 | 9 |
| P06.4 | SCR-FEE-017, SCR-FEE-018, SCR-FEE-019, SCR-FEE-021, SCR-FEE-022, SCR-FEE-023, SCR-FEE-024, SCR-FEE-025, SCR-FEE-026, SCR-FEE-040 | 10 |
| P06.5 | SCR-FEE-010, SCR-FEE-020, SCR-FEE-027 | 3 |
| P06.6 | SCR-ACC-012, SCR-FEE-028, SCR-FEE-029, SCR-FEE-030, SCR-FEE-031, SCR-FEE-032, SCR-FEE-033, SCR-FEE-041, SCR-RPT-003 | 9 |
| P07.1 | SCR-EXM-001, SCR-EXM-002, SCR-EXM-003, SCR-EXM-004, SCR-EXM-005, SCR-EXM-006, SCR-EXM-007, SCR-EXM-008, SCR-EXM-009, SCR-EXM-010, SCR-EXM-011, SCR-EXM-012, SCR-EXM-013, SCR-EXM-014, SCR-EXM-015, SCR-EXM-016, SCR-EXM-017, SCR-EXM-018, SCR-EXM-020, SCR-EXM-021 | 20 |
| P07.2 | SCR-RES-001, SCR-RES-002, SCR-RES-003, SCR-RES-004, SCR-RES-005, SCR-RES-006, SCR-RES-007, SCR-RES-008, SCR-RES-009, SCR-RES-010, SCR-RES-011, SCR-RES-012, SCR-RES-013, SCR-RES-014, SCR-RES-015, SCR-RES-017, SCR-RES-018, SCR-RES-019 | 18 |
| P07.3 | SCR-EXT-001, SCR-EXT-002, SCR-EXT-003, SCR-EXT-004, SCR-EXT-005, SCR-EXT-006 | 6 |
| P08.1 | SCR-STU-009, SCR-STU-010 | 2 |
| P08.2 | SCR-STU-008, SCR-STU-011, SCR-STU-012, SCR-STU-013, SCR-STU-014, SCR-STU-015 | 6 |
| P08.3 | SCR-STU-016, SCR-STU-017 | 2 |
| P09.1 | SCR-ATT-003, SCR-TTB-001, SCR-TTB-002, SCR-TTB-003, SCR-TTB-004, SCR-TTB-005, SCR-TTB-006, SCR-TTB-007, SCR-TTB-008, SCR-TTB-010, SCR-TTB-011 | 11 |
| P09.2 | SCR-ADN-001, SCR-ADN-002, SCR-ADN-003, SCR-ADN-004, SCR-ADN-009, SCR-ADN-012, SCR-COMM-001, SCR-COMM-002, SCR-COMM-003, SCR-COMM-004, SCR-COMM-005, SCR-COMM-006, SCR-COMM-013, SCR-COMM-014, SCR-COMM-017 | 15 |
| P09.3 | SCR-RPT-001, SCR-RPT-002, SCR-RPT-004, SCR-RPT-006, SCR-RPT-007, SCR-RPT-008, SCR-RPT-009 | 7 |
| P11.1 | SCR-FEE-034, SCR-FEE-035 | 2 |
| P11.2 | SCR-FEE-038, SCR-FEE-039 | 2 |
| P12.1 | SCR-ACC-015, SCR-ACC-016, SCR-ACC-017, SCR-ACC-018 | 4 |
| P12.2 | SCR-ACC-019, SCR-ACC-020 | 2 |
| P12.3 | SCR-ACC-021 | 1 |
| P13.1 | SCR-HR-014, SCR-HR-015, SCR-HR-016, SCR-HR-017, SCR-HR-018 | 5 |
| P13.2 | SCR-PAY-001, SCR-PAY-002, SCR-PAY-003, SCR-PAY-004 | 4 |
| P13.3 | SCR-PAY-005, SCR-PAY-006, SCR-PAY-007, SCR-PAY-009, SCR-PAY-010, SCR-PAY-011, SCR-PAY-012, SCR-PAY-014 | 8 |
| P13.4 | SCR-PAY-008, SCR-PAY-013 | 2 |
| P14.1 | SCR-LMS-001, SCR-LMS-002, SCR-LMS-003, SCR-LMS-004 | 4 |
| P14.2 | SCR-LMS-005, SCR-LMS-006, SCR-LMS-007, SCR-LMS-008, SCR-LMS-009, SCR-LMS-010, SCR-LMS-011 | 7 |
| P15.1 | SCR-HWK-001, SCR-HWK-002, SCR-HWK-003, SCR-HWK-004, SCR-HWK-005, SCR-HWK-006, SCR-HWK-007, SCR-HWK-008 | 8 |
| P15.2 | SCR-ASM-001, SCR-ASM-002, SCR-ASM-003, SCR-ASM-004 | 4 |
| P15.3 | SCR-ASM-005, SCR-ASM-006, SCR-ASM-007, SCR-ASM-008, SCR-ASM-009, SCR-ASM-010, SCR-ASM-011, SCR-ASM-012, SCR-ASM-013 | 9 |
| P16.1 | SCR-COMM-007, SCR-COMM-008, SCR-COMM-009, SCR-COMM-010, SCR-COMM-011, SCR-COMM-012, SCR-COMM-015, SCR-COMM-016 | 8 |
| P16.3 | SCR-ADN-005, SCR-ADN-010, SCR-EXM-019, SCR-RPT-005, SCR-TTB-009 | 5 |
| P16.4 | SCR-ADM-012, SCR-ADM-013, SCR-DOC-006, SCR-RES-021 | 4 |
| P16.5 | SCR-EXT-007, SCR-EXT-008, SCR-EXT-009, SCR-EXT-010, SCR-EXT-011, SCR-EXT-012, SCR-EXT-013, SCR-EXT-014, SCR-EXT-015, SCR-EXT-016, SCR-EXT-017, SCR-EXT-018, SCR-EXT-019, SCR-EXT-020, SCR-EXT-021, SCR-EXT-022, SCR-EXT-023, SCR-EXT-024, SCR-EXT-025, SCR-RES-016, SCR-RES-020 | 21 |
| P17.1 | SCR-TRN-001, SCR-TRN-002, SCR-TRN-003, SCR-TRN-004, SCR-TRN-005, SCR-TRN-006, SCR-TRN-016 | 7 |
| P17.2 | SCR-DSC-001, SCR-DSC-002, SCR-DSC-003, SCR-DSC-004, SCR-DSC-005, SCR-DSC-006, SCR-DSC-007, SCR-DSC-010, SCR-DSC-011 | 9 |
| P17.3 | SCR-IAM-027 | 1 |
| P18.3 | SCR-PLT-020 | 1 |
| P19.1 | SCR-LIB-001, SCR-LIB-002, SCR-LIB-003, SCR-LIB-004, SCR-LIB-005, SCR-LIB-006, SCR-LIB-007, SCR-LIB-008, SCR-LIB-009, SCR-LIB-010, SCR-LIB-011 | 11 |
| P20.1 | SCR-TRN-007, SCR-TRN-008, SCR-TRN-009, SCR-TRN-010, SCR-TRN-013 | 5 |
| P20.2 | SCR-TRN-011, SCR-TRN-012, SCR-TRN-014, SCR-TRN-015 | 4 |
| P21.1 | SCR-INV-001, SCR-INV-002, SCR-INV-003, SCR-INV-004, SCR-INV-005, SCR-INV-006, SCR-INV-007, SCR-INV-008, SCR-INV-009, SCR-INV-010, SCR-INV-011, SCR-INV-012 | 12 |
| P22.1 | SCR-ADN-006, SCR-ADN-007, SCR-ADN-008, SCR-ADN-011, SCR-INV-013, SCR-INV-014 | 6 |
| P22.2 | SCR-DSC-008, SCR-DSC-009 | 2 |
| P23.1 | SCR-ACC-022, SCR-RPT-010, SCR-RPT-011 | 3 |
| P23.2 | SCR-DOC-003 | 1 |
| P23.3 | SCR-TTB-012 | 1 |
| P23.4 | SCR-ATT-026 | 1 |
| P24.1 | SCR-PLT-021, SCR-PLT-022, SCR-PLT-023, SCR-SET-007, SCR-SET-008 | 5 |
| P24.2 | SCR-IAM-028 | 1 |
| P24.3 | SCR-ACC-023, SCR-PLT-024 | 2 |
| **Total** | | **515** |

---

## 15.12 ScreenCoverageTest Exceptions Register (DEC-321)

`ScreenCoverageTest` checks that every screen of a release exists with its D10 route name and permission, and that every permission key in the catalog is used by at least one screen, widget or action. The entries below are the known cases where the plain rule does not fit. Each one has a compensating test. They are loaded from `tests/Fixtures/screen_coverage_exceptions.php`.

| ID | Screens or keys | Reason | Compensating test | Owner | Review or close |
|---|---|---|---|---|---|
| EX-01 | SCR-IAM-001 to SCR-IAM-006, SCR-PLT-002, SCR-ADM-010, SCR-ADM-011, SCR-DOC-006, SCR-RES-021 | Public or pre-login screens have no permission key | `GuestRouteTest` (throttling, CSRF, no tenant data beyond what D10 allows), verification page rate-limit tests | Tech lead | Permanent; reviewed each release |
| EX-02 | SCR-IAM-007 to SCR-IAM-011, SCR-CORE-005, SCR-APR-001, SCR-APR-003, SCR-APR-009, SCR-NTF-001, SCR-NTF-002, SCR-IMX-005, SCR-GRD-010, SCR-LVE-003, SCR-COMM-013, SCR-ADN-001, SCR-ADN-009, SCR-ADN-012, SCR-RPT-001, SCR-RPT-004, SCR-RPT-008, SCR-RPT-009 | Baseline screens available to every signed-in user of the portal (D05 baseline) | `BaselineScopeTest`: each shows only the user's own or audience-targeted records; portal guard test | Tech lead | Permanent; reviewed each release |
| EX-03 | SCR-ACD-017, SCR-ATT-011, SCR-ATT-012, SCR-ATT-013, SCR-LVE-004, SCR-TTB-008, SCR-CORE-004, SCR-APR-002, SCR-APR-008, SCR-COMM-011, SCR-EXT-024, SCR-PAY-013, SCR-PAY-014, SCR-LIB-011 | Own-scope screens: a module permission restricted to the user's own records, or baseline for own records plus a permission for others | `OwnScopeTest` per screen: own records visible, others refused (404) | Tech lead | Permanent; reviewed each release |
| EX-04 | Every SCR-PLT screen, including SCR-PLT-001, SCR-PLT-018, SCR-PLT-019 | Platform portal screens are guarded by platform roles and the platform guard, not tenant permissions | `PlatformRbacTest` for the 3 platform roles; tenant users get 404 on the platform host | Tech lead | Permanent |
| EX-05 | SCR-IMX-002, SCR-IMX-004, SCR-RPT-002, SCR-RPT-003 | Permission resolved at runtime from the import type or report definition | `ImportTypePermissionTest` and `ReportDefinitionPermissionTest` iterate every registered type and report | Tech lead | Permanent |
| EX-06 | SCR-ATT-005, SCR-ATT-009, SCR-FEE-039 | D10 permission column says "Same" as the parent screen | Test uses the parent screen's permission and route group | Tech lead | Permanent |
| EX-07 | SCR-PLT-011 Usage | Exists in R1 but shows the empty state until metering arrives in P18.1 | Empty-state test in R1; `UsageMeter` tests from P18.1 | Tech lead | Close at P18.1 |
| EX-08 | SCR-IAM-027 API tokens | Device tokens only from P17.3; portal token management from P18.2 | Device token tests in P17.3; portal token tests in P18.2 | Tech lead | Close at P18.2 |
| EX-09 | SCR-RPT-010, SCR-RPT-011, SCR-ACC-022, SCR-TTB-012, SCR-ATT-026, SCR-SET-007, SCR-SET-008, SCR-PLT-021 to SCR-PLT-024, SCR-IAM-028, SCR-ACC-023 | Screens added by D14 DEC-305 and DEC-310, not in the D10 file until the P23.0 and P24.0 addenda update it | Addendum rows reviewed in P23.0 and P24.0; then normal coverage | Product owner | Close at P23.0 and P24.0 |
| EX-10 | Route stubs created in P23.0 and P24.0 | Stubs return 404 until the building step | Stub test asserts 404 until the step's flag is on | Tech lead | Close at P23.4 and P24.3 |
| EX-11 | Screens of type modal or drawer (for example SCR-FEE-020, SCR-LIB-008, SCR-STU-004) | Rendered inside a parent view; the route name is the action endpoint | Test asserts the action route, its permission, and that the parent view includes the component | Tech lead | Permanent |
| EX-12 | SCR-IAM-008 to SCR-IAM-011 portal variants (P04.3) | One screen ID with route names in staff, guardian and student portals | Fixture maps every portal route name to the screen ID; each route tested | Tech lead | Permanent |

New exceptions are added here with the next EX number, the approval of the tech lead and the product owner, and a close or review point. An exception that reaches its close point while still open blocks the release (RC-2).

---

## 15.13 Release Sign-off

| Release | Conditions (DEC-323) | Tag | Product owner | Date | Signature or approval link |
|---|---|---|---|---|---|
| R1 | All R1 step rows Done, Skipped or Deferred with links; PH-00 to PH-10 gates closed; RC-1 to RC-9 evidence in 15.8; 313 R1 FRs ticked or deferred; EX-07 and EX-08 open with planned close; no open high or critical fix steps | `r1.0.0` | | | |
| R2 | All R2 rows as above; PH-11 to PH-18 gates closed; RC-1 to RC-9 for R2; 95 R2 FRs ticked or deferred; EX-07 and EX-08 closed | `r2.0.0` | | | |
| R3 | All R3 rows as above; PH-19 to PH-24 gates closed; RC-1 to RC-9 for R3; 33 R3 FRs ticked or deferred; EX-09 and EX-10 closed | `r3.0.0` | | | |

Deferred items carried into a later release are listed below the sign-off with the D13 DEC-223 decision link:

| Item (step, screen or FR) | From release | To release or phase | Decision link | Approved by, date |
|---|---|---|---|---|
| FR-IAM-016 Sanctum API tokens | R1 | R2 (P17.3 device tokens, P18.2 portal tokens) | D14 DEC-283, D15 DEC-324 | |

---

## Assumptions (D15)

| ID | Assumption |
|---|---|
| A-148 | A named tech lead and a named product owner are appointed before P00.1. Each can name one deputy who may sign when they are away. |
| A-149 | The repository host's issue tracker is used for findings, with labels for severity (critical, high, medium, low), the release, and the fix ID. G-11 and RC-3 read from those labels. |
| A-150 | CI run logs, step reports and evidence files are kept for the life of the release plus one year, so links in this checklist keep working for audits. |

## Gaps Found and Added (D15)

| ID | Gap | Decision |
|---|---|---|
| GAP-372 | Nothing defined who may mark work as done | DEC-319: agents propose, reviewers tick |
| GAP-373 | Conditional and moved steps had no status values | DEC-318 statuses, including Skipped and Deferred |
| GAP-374 | The checklist could drift from the code and the reports | DEC-320 `ChecklistConsistencyTest` and the CSV copy |
| GAP-375 | `ScreenCoverageTest` exceptions had no format or approval rule | DEC-321 and the 15.12 register |
| GAP-376 | The 13 screens added in D14 Part 9 were missing from the D10 counts | Counted in 15.11 and tracked by EX-09 |
| GAP-377 | D10 promised an FR-to-screen matrix that did not yet exist | 15.10 covers all 441 FRs, including FR-EXT |
| GAP-378 | RC-8 legal review had no defined scope for R2 and R3 | Delta reviews named in 15.8 |
| GAP-379 | Fix steps from the pilot and later releases had no register | 15.5 with fixed sources and ID ranges (DEC-322) |
| GAP-380 | FR-ADM-015, FR-ATT-025 and FR-IMX-010 had no building step, and FR-IAM-016 was delivered later than its D02 release | DEC-324: tasks added to D14 Part 8; FR-IAM-016 recorded as Deferred |
