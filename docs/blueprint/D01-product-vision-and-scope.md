# Deliverable 1: Product Vision and Scope

**Blueprint:** School & College Management ERP + LMS
**Version:** 1.0 (draft for approval) | **Date:** 23 Nov 2026
**Binding for:** Deliverables 2 to 15 (names, decisions and release scope defined here are reused everywhere)

---

## 1.1 Critical Finding: The Specified Runtime Is About to Lose Security Support

| Component | Version in brief | Status today | Security support ends | Days left (from 23 Nov 2026) |
|---|---|---|---|---|
| PHP | 8.2.30 | Security fixes only. Latest 8.2 patch is 8.2.35 (21 Oct 2026), so 8.2.30 is missing 5 security releases | 31 Dec 2026 | about 38 |
| Laravel | 12.71.0 | Bug fixes ended 13 Aug 2026, security fixes only | 24 Feb 2027 | about 93 |
| Laravel 13 | not in brief | Fully supported, released 17 Mar 2026, requires PHP 8.3+ | Q1 2028 | n/a |
| Laravel 14 | not in brief | In development, master branch requires PHP 8.4 | n/a | n/a |

A system holding children's records, guardian CNICs and fee money cannot go live on an unpatched runtime. Release 1 is months away, so the brief's exact stack would be end of life on launch day. The brief asks for a compatible decision when a conflict exists. Here it is.

**DEC-01: Runtime and framework policy**

1. Development starts now on Laravel 12.x as specified. Nothing waits for an upgrade.
2. All code must run unchanged on PHP 8.2 and PHP 8.4:
   - No PHP 8.3+ only features (typed class constants, `#[\Override]`, `json_validate()`, property hooks, asymmetric visibility).
   - No patterns deprecated in PHP 8.4 (implicit nullable parameters such as `function f(Carbon $d = null)`; use `?Carbon $d = null`).
3. Upgrade the local, staging and production runtime to PHP 8.4 before 31 Dec 2026. Laravel 12 officially supports PHP 8.2 to 8.5, so this needs no framework change.
4. Composer package rule: a package is accepted only if it declares support for Laravel 12 **and** 13, and PHP 8.2 through 8.4.
5. Upgrade Gate UG-1: move to Laravel 13 at the end of Phase 1 (Foundation), and in every case before 24 Feb 2027 and before the first production tenant.
6. CI runs the test suite on PHP 8.2 and 8.4 until UG-1. After UG-1, PHP 8.4 is the only target and the PHP 8.2 syntax rule is retired.
7. Standardizing on PHP 8.4 also prepares for Laravel 14.

Every later deliverable assumes DEC-01.

---

## 1.2 Product Identity

| Item | Decision |
|---|---|
| Working name | **Darsora**. Chosen on 26 Nov 2026 in place of the earlier placeholder. Confirm the trademark (IPO Pakistan), the SECP company name and the darsora.app domain before launch. |
| Code identity | The product name never appears in code. Namespaces are `App\` and `Modules\{ModuleName}\`, so a rename costs nothing. |
| Product type | Multi-tenant SaaS, Laravel modular monolith, server-rendered web app, mobile-responsive, English and Urdu. |
| White-labeling | Each organization sets logo, document headers, signatures, brand accent color (within accessible contrast limits) and subdomain. |
| Default locale | English UI, Urdu available, PKR currency, Asia/Karachi timezone, all configurable per organization. |

---

## 1.3 Vision Statement

> One trusted system where a Pakistani school, school chain or college runs its whole working day, from admission to result card, from gate attendance to fee collection, from classroom to accounts, with every parent informed and every mark, attendance change and rupee traceable to a person, a time and a reason.

## 1.4 Three-Year Outcomes

1. A school replaces paper registers, Excel sheets, standalone biometric software, separate result software and WhatsApp-only parent communication with one system.
2. An owner of 20 branches sees today's attendance, today's collection, outstanding fees and pending approvals across all branches on one screen in under 10 seconds.
3. Every change to marks, attendance and money is auditable, and protected records cannot be silently edited.
4. A parent with a budget Android phone, who prefers Urdu and has no email address, can see attendance, results, homework and fee dues without calling the school.
5. A new school is onboarded and generating its first fee vouchers within one working week.

---

## 1.5 Problem Statement (Punjab Private School and College Context)

| Area | Typical current practice | Consequence |
|---|---|---|
| Admissions | Paper forms, admission register (GR register), later typed into Excel | Duplicate records, lost documents, no history |
| Student attendance | Paper registers per section, monthly totals by hand | Parents learn about absences late, no analysis |
| Staff attendance | Biometric machine with vendor desktop software on one office PC, monthly export | Manual salary deductions, disputes, data stuck on one PC |
| Fees | Excel-printed 3-copy bank challans (bank, school, student), stamped copies reconciled by hand | Fee leakage, slow defaulter lists, reconciliation errors |
| Cash counter | Receipt books, no daily cash close | Cash variance goes unnoticed |
| Results | Excel templates per teacher, formulas copied around, positions calculated manually | Wrong results, late report cards, no approval trail |
| Parent communication | WhatsApp groups, SMS blasts | No record of what was sent, privacy leaks in groups |
| Multi-branch control | Branches email Excel to head office weekly | Owner decisions are based on stale numbers |
| HR and payroll | Excel salary sheets | Absence deductions and advances tracked inconsistently |
| Audit | None | Nobody can say who changed what |
| Session change | Last year's file copied and overwritten | History lost, transfer certificates hard to produce |

---

## 1.6 Target Market and Segments

| Code | Segment | Typical size | Characteristics | Must-haves |
|---|---|---|---|---|
| SEG-1 | Single private school | 150 to 1,500 students, 10 to 70 staff, 1 campus | Owner is often also principal, cash-heavy fees, Urdu-preferring parents | Fees and vouchers, attendance, results, parent portal, low cost |
| SEG-2 | Multi-branch school chain | 3 to 50 branches, 2,000 to 30,000 students | Central accounts, standardized curriculum and fee policy, brand control | Consolidated owner dashboard, branch isolation, central fee policy, transfers |
| SEG-3 | Educational group (schools and colleges) | Several institutions (school brand plus college brand) | Primary to intermediate, different grading per institution | Multiple institutions per organization, programs and groups, board exam focus |
| SEG-4 | Standalone intermediate college | 300 to 3,000 students | F.Sc, ICS, I.Com, FA programs, morning and evening shifts, sometimes hostel | Programs with electives, board exam support, shift support |
| SEG-5 | Platform operator (us) | n/a | Runs the SaaS, onboards and supports tenants | Tenant lifecycle, plans, entitlements, support tools |

**Not targeted in R1 to R3:** government schools (different reporting obligations), universities and degree colleges on semester and credit-hour systems, tuition academies. The academic model stays generic so these can be added later without redesign.

---

## 1.7 Deployment Scenarios

| Scenario | Example | Organizations | Institutions per org | Branches per institution | UI behavior |
|---|---|---|---|---|---|
| A: Single school | One owner, one school | 1 | 1 | 1 | Branch switcher hidden, consolidated and branch dashboards merge into one |
| B: Multi-branch | One owner, 12 branches | 1 | 1 or more | many | Owner sees consolidated views, staff see only assigned branches, branch switcher shown |
| C: Multi-org SaaS | Hundreds of independent schools | many | any | any | Complete isolation per organization, each on its own subdomain |
| D: Dedicated instance (added) | A large group wants its own server | 1 per instance | any | any | Same codebase deployed single-tenant. Enabled by the scoping design in Deliverable 4. Commercial option, not separate code. |

All four scenarios run on one data model. Scenario A is simply an organization with one institution and one branch, created automatically by the onboarding wizard.

```
                          +-----------------------------------+
                          |        PLATFORM (Darsora)         |
                          |  Super Admin, Plans, Entitlements |
                          +-----------------+-----------------+
                                            |
           +--------------------------------+-------------------------------+
           |                                |                               |
+----------v----------+          +----------v----------+         +----------v----------+
| ORGANIZATION (tenant)|          | ORGANIZATION        |         | ORGANIZATION        |
| "City School Group"  |          | "Al-Noor School"    |         | "Prime College"     |
+----------+----------+          +----------+----------+         +----------+----------+
           |                                |                               |
   +-------+--------+                 INSTITUTION                     INSTITUTION
   |                |                       |                               |
INSTITUTION     INSTITUTION               BRANCH                    BRANCH (Main)
School brand    College brand          (single campus)          Morning + Evening shift
   |                |
 +-+------+-----+   BRANCH
 |        |     |
BRANCH  BRANCH  BRANCH  ...each branch: sessions -> classes -> sections -> students, staff

External actors:  Guardians | Students | Biometric devices | Banks / 1Bill / JazzCash / Easypaisa
                  SMS aggregator | Email (SMTP) | WhatsApp provider | Object storage
```

---

## 1.8 Personas

| # | Persona | Role slug | Goals | Pain points today | Devices and context | Key needs from product |
|---|---|---|---|---|---|---|
| P1 | Platform support engineer | `platform_super_admin` | Onboard tenants fast, resolve tickets | No visibility into tenant problems | Desktop | Tenant console, entitlements, audited "login as" with consent |
| P2 | Owner of a 6-branch chain | `org_owner` | Know collection, attendance and problems across branches daily | Weekly Excel from branches, no trust in numbers | Phone (most of the time), laptop | Consolidated dashboard, branch comparison, approvals on phone |
| P3 | Owner-principal of one school | `org_owner` + `principal` | Run everything with a small team | Too many tools, too much paper | Phone, office PC | Simple setup, one login with role switching |
| P4 | Branch principal | `principal` | Smooth academic operations, discipline, results on time | Chasing teachers for marks and registers | Office PC, phone | Daily activity view, pending approvals, result approval |
| P5 | Admin officer / branch manager | `branch_manager` | Admissions, records, certificates, notices | Searching paper files | Office PC (often old hardware) | Fast search, bulk print, templates |
| P6 | Accountant | `accountant` | Accurate ledgers, month close, defaulter control | Manual challan reconciliation | Office PC | Bulk vouchers, reconciliation, period locks, reports |
| P7 | Cashier | `cashier` | Collect quickly, close cash daily | Receipt books, counting variance | Office PC, receipt printer | 3-click payment entry, cash close, reprint receipt |
| P8 | Class teacher (school) | `class_teacher` | Mark attendance, diary, marks, parent contact | Registers, Excel result sheets | Android phone, sometimes shared PC | Attendance in under 60 seconds, daily diary, marks entry grid |
| P9 | Subject lecturer (college) | `subject_teacher` | Teach multiple sections, quizzes, marks | Repeated work per section | Laptop, phone | Reusable content and question bank across sections |
| P10 | Examination controller | `exam_controller` | Date sheets, marks collection, result publishing | Collating Excel from many teachers | Office PC | Marks completion tracker, grading config, approval flow |
| P11 | HR manager | `hr_manager` | Staff records, leaves, attendance-based payroll inputs | Biometric data on one PC | Office PC | Device sync, leave balances, monthly attendance summary |
| P12 | Parent / guardian | `guardian` | Know child is safe, learning, and what fee is due | Late absence info, lost challans | Budget Android phone, Urdu, often no email, sometimes shared phone | Mobile-first Urdu portal, SMS alerts, voucher download, multiple children |
| P13 | Secondary student (Class 6 to 10) | `student` | Homework, results, timetable | Missed homework info | Parent's phone or own phone | Simple view, homework submission |
| P14 | Intermediate student | `student` | Lectures, notes, quizzes, board prep | Scattered notes | Own phone | LMS, quizzes, discussion under lectures |
| P15 | Transport manager | `transport_manager` | Routes, vans, student allocation | Paper lists | Phone | Route lists, allocation, transport fee link |
| P16 | IT administrator | `it_admin` | Devices, users, backups | Many disconnected systems | Desktop | Device registry, sync logs, user management |

---

## 1.9 Value Proposition by Stakeholder

| Stakeholder | Value |
|---|---|
| Owner | Real numbers every day across branches, fewer leakage points, approvals from the phone |
| Principal | Less chasing, clear pending-task lists, controlled result publication |
| Accountant and cashier | Bulk vouchers in minutes, automatic ledgers, daily cash close, defaulter lists on demand |
| Teacher | Fast attendance, reusable content, marks grid with automatic totals, fewer registers |
| Parent | Absence alerts, results, homework and dues on the phone in Urdu or English |
| Student | One place for timetable, homework, lectures, quizzes and results |
| Platform operator | One codebase serves single schools to large groups, module-based plans |

---

## 1.10 Product Principles (Binding)

| # | Principle | What it means in practice |
|---|---|---|
| PR-01 | Configurable, not hardcoded | Sessions, levels, groups, grading, fee heads, approval chains, attendance rules and roles are data, never code constants |
| PR-02 | History is permanent | Enrollments per session, attendance corrections as new versions, payments reversed (never deleted), marks versioned |
| PR-03 | Backend enforces everything | Hiding a button is cosmetic. Every request passes tenant scope, branch scope and a policy check |
| PR-04 | Every sensitive change leaves a trace | Audit entry with who, what, before, after, when, from where, and why (reason field on protected changes) |
| PR-05 | Mobile-first for guardians and teachers | Every guardian and teacher screen is designed for a 360px wide screen first |
| PR-06 | Urdu is a first-class language | RTL support built from day one (CSS logical properties), not retrofitted |
| PR-07 | Low bandwidth friendly | Light pages, paginated tables, no large JS bundles, images compressed on upload |
| PR-08 | Tolerant of bad connectivity | Device punches and payment callbacks are idempotent and can arrive late or twice without damage |
| PR-09 | One engine per concern | One approval engine, one notification engine, one import/export engine, one document engine, one numbering service |
| PR-10 | Simple for Scenario A, powerful for Scenario C | Multi-branch features appear only when the organization has more than one branch |
| PR-11 | No fake functionality | A screen ships only when its data, permissions and tests are real |
| PR-12 | Money is exact | No floating-point arithmetic on money, double-entry postings, locked periods |

---

## 1.11 Scope Dimensions

| Dimension | In scope |
|---|---|
| Organizational | Platform, organizations (tenants), institutions, branches, departments, shifts |
| Academic levels | Playgroup to Intermediate Part 2 (configurable list), matric groups, intermediate programs |
| Academic time | Academic sessions (per institution) and fiscal years (per organization, July to June default) as separate concepts |
| Users | 23 system role templates plus unlimited custom roles (Deliverable 5) |
| Channels | Web (desktop and mobile browser), email, SMS, WhatsApp (R2), push notifications (R3, for future apps) |
| Languages | English, Urdu (RTL), extensible |
| Currency | PKR default, configurable per organization (single currency per organization) |
| Geography | Pakistan first (Punjab defaults for boards, holidays, levels), no hard dependency on Punjab |

---

## 1.12 Release Scope

Three commercial releases. Deliverable 13 maps the development phases onto these releases.

- **R1 "Core School Operations":** the minimum product a single school or chain can run its year on.
- **R2 "Learning and Money":** LMS, online assessment, payment gateways, full accounting, HR and payroll, WhatsApp.
- **R3 "Complete ERP":** library, transport, inventory, advanced analytics, self-service SaaS billing, mobile API.

| Module | R1 | R2 | R3 |
|---|---|---|---|
| Platform (tenants, plans, entitlements) | Tenant lifecycle, plans and module flags set manually by super admin | Usage metering (students, storage, SMS) | Self-service subscription billing |
| Organization (orgs, institutions, branches, departments, shifts) | Full | | |
| Identity and Access (auth, RBAC, overrides) | Full dynamic RBAC, user overrides, optional 2FA, login history | | |
| Audit | Full | | |
| Settings and Localization | English complete; Urdu complete for guardian portal and printed documents; RTL infrastructure everywhere | Urdu complete for all portals | Additional languages |
| Academic | Full (sessions, levels, groups, programs, classes, sections, subjects, teaching assignments) | | |
| Admission | Offline admission, basic online form, bulk Excel import, ID and roll number generation | Inquiry and lead tracking, admission tests, interview scheduling | |
| Student and Guardian | Full profile, family grouping, timeline, documents | | |
| Student Lifecycle | Promotion, retention, transfers, withdrawal, leaving and character certificates | | Certificate template designer |
| HR (employees, teacher profiles) | Employee records, departments, designations, teaching assignments | Contracts, performance reviews, training, disciplinary | |
| Attendance | Teacher-marked student attendance, staff online attendance, QR attendance, CSV import, ZKTeco ADMS push adapter, signed webhook adapter, corrections, reports | On-premise sync agent for pull-only devices, RFID | Offline-capable attendance marking |
| Leave | Full (students, teachers, staff) | | |
| Approval Engine | Full configurable chains | Escalation and SLA timers | |
| Fee | Structures, heads, discounts, sibling rules, scholarships, bulk and family vouchers, manual payments (cash, bank, cheque), receipts, defaulters, cash close | Online gateways (1Bill, JazzCash, Easypaisa), automated reconciliation | Bank-specific APIs, Raast through partner bank |
| Accounting | Chart of accounts, automatic double-entry posting from fees, cashbook, journal, fee receivables, period locks | Expenses, purchases, bank reconciliation, financial statements | Advanced consolidation |
| Examination and Result | Exam setup, admit cards, marks entry, grading schemes, approval, report cards | Board exam support (registration exports, board results) | Result analytics |
| Timetable | Manual timetable with conflict detection, printable | Substitution (arrangement periods) | Assisted auto-generation |
| Communication | Announcements, notices, daily diary | Guardian-teacher messaging, parent-teacher meeting booking | Complaints |
| Notification | In-app, email, SMS | WhatsApp | Push (FCM) |
| Learning (LMS) | | Full | |
| Homework | (daily diary covers basic homework notes) | Full with submissions | |
| Assessment (question bank, quizzes, online exams) | | Full | |
| Payroll | | Full | |
| Library | | | Full |
| Transport | Transport fee head only | Routes, stops, allocation | Vehicles, maintenance, fuel, optional GPS |
| Inventory and Assets | | | Full |
| Administration | Calendar, holidays, events, ID cards | | Visitors, maintenance, complaints |
| Discipline and Wellbeing | | Incidents, remarks, warnings, rewards | Counseling records |
| Reporting and Dashboards | Role dashboards, standard reports with filter, print, PDF, Excel | Scheduled emailed reports | Branch comparison analytics |
| Import and Export | Engine plus students, guardians, staff, opening balances, attendance, marks | All remaining entities | |
| Document Generation | Vouchers, receipts, report cards, admit cards, ID cards, certificates | Salary slips | Template designer |
| API | Device and payment webhooks, internal endpoints | Versioned API v1 for mobile apps | Public partner API |

---

## 1.13 Mandatory vs Optional Integrations

| Integration | Classification | Release | Notes |
|---|---|---|---|
| SMTP email | Mandatory | R1 | Any SMTP or transactional email provider |
| SMS gateway | Mandatory (one provider adapter) | R1 | Through a licensed Pakistani aggregator with branded sender ID. Provider selected per organization. Urdu SMS uses Unicode (70 characters per segment), so cost metering is required |
| Attendance: CSV/Excel import | Mandatory | R1 | Universal fallback for any device |
| Attendance: ZKTeco ADMS push | Mandatory | R1 | ZKTeco is the most common brand in Pakistani institutions. Push support depends on model and firmware; verify per device |
| Attendance: signed webhook | Mandatory | R1 | Generic adapter for any device or middleware that can POST events |
| Attendance: on-premise sync agent | Optional | R2 | For LAN-only devices that the cloud cannot reach |
| RFID readers | Optional | R2 | Usually arrive through the same device adapters |
| QR attendance | Built in (no third party) | R1 | Rotating, expiring codes |
| Object storage (S3-compatible) | Optional | R1 | Config switch; local disk by default |
| PDF rendering (headless Chromium) | Mandatory | R1 | Required for correct Urdu Nastaliq rendering (see DEC-12) |
| 1Bill (1LINK) bill payment | Optional, high priority | R2 | Lets parents pay through any bank app; requires merchant onboarding by each organization |
| JazzCash, Easypaisa | Optional | R2 | Official merchant APIs only, each organization uses its own merchant account |
| Bank-specific APIs, Raast | Optional | R3 | Through partner bank or PSP |
| WhatsApp Business Platform | Optional | R2 | Through Meta Cloud API or an approved provider; message templates need approval |
| Laravel Reverb (realtime) | Optional | R2 | Polling fallback always present |
| Video hosting | Optional | R2 | Local upload with size caps, or external links (YouTube unlisted, Vimeo, others) |
| Live class links (Zoom, Google Meet) | Optional | R2 | Links only, no embedded conferencing |
| Leaflet with OpenStreetMap | Optional | R3 | Transport routes |
| GPS vehicle trackers | Optional | R3 | Provider adapter |
| Push notifications (FCM) | Optional | R3 | For future mobile apps |

---

## 1.14 Non-Goals (Out of Scope for R1 to R3)

1. Native Android and iOS apps (the API is prepared in R2; a PWA wrapper may be considered).
2. University and degree programs with semesters and credit hours.
3. Direct submission to government or board systems (export files only).
4. Hostel and boarding management (schema leaves room; post-R3 add-on).
5. Canteen POS, uniform shop, e-commerce.
6. Acting as a payment aggregator or holding funds. Money always goes to the organization's own merchant or bank account.
7. Webcam proctoring and AI grading of subjective answers.
8. Live video streaming infrastructure and video transcoding.
9. Statutory filing (FBR, EOBI, PESSI). Payroll produces deduction reports only.
10. Alumni management.

---

## 1.15 Module Catalog (Canonical Module Names)

These names are used as folder names (`modules/{Name}`), namespaces (`Modules\{Name}`) and permission prefixes for the rest of the blueprint. Deliverable 2 details each one.

| # | Module (folder) | Permission prefix | Tier | Covers |
|---|---|---|---|---|
| 1 | Core | `core` | Core | Base classes, Money, numbering, tenancy scopes, shared UI components |
| 2 | Platform | `platform` | Core | Plans, entitlements, tenant lifecycle, platform admin |
| 3 | Organization | `organization` | Core | Organizations, institutions, branches, departments, shifts |
| 4 | Identity | `identity` | Core | Users, authentication, roles, permissions, overrides, sessions |
| 5 | Settings | `settings` | Core | Organization and branch settings, localization, branding |
| 6 | Audit | `audit` | Core | Audit logs, login history |
| 7 | Approval | `approval` | Core | Workflow definitions, requests, steps, decisions |
| 8 | Notification | `notification` | Core | Templates, channels, delivery logs, preferences |
| 9 | ImportExport | `import_export` | Core | Templates, mapping, validation, jobs, history |
| 10 | Document | `document` | Core | PDF templates, branding, rendering, verification codes |
| 11 | Academic | `academic` | Core | Sessions, levels, groups, programs, classes, sections, subjects, teaching assignments |
| 12 | Admission | `admission` | Standard | Applications, inquiries, tests, interviews, admission decisions |
| 13 | Student | `student` | Core | Students, enrollments, lifecycle, timeline, certificates |
| 14 | Guardian | `guardian` | Core | Guardians, families, guardian-student links, verification |
| 15 | HR | `hr` | Standard | Employees, teacher profiles, contracts, reviews |
| 16 | Attendance | `attendance` | Core | Student and staff attendance, devices, corrections |
| 17 | Leave | `leave` | Core | Leave types, balances, applications |
| 18 | Fee | `fee` | Core | Fee heads, structures, discounts, vouchers, payments, receipts |
| 19 | Accounting | `accounting` | Standard | Chart of accounts, journals, ledgers, fiscal years, reconciliation |
| 20 | Examination | `examination` | Core | Exam terms, exams, date sheets, admit cards, marks entry |
| 21 | Result | `result` | Core | Grading schemes, calculation, approval, publication, report cards |
| 22 | Timetable | `timetable` | Standard | Periods, timetables, rooms, substitutions |
| 23 | Communication | `communication` | Core | Announcements, notices, daily diary, messaging, meetings |
| 24 | Learning | `learning` | Add-on | Courses, chapters, lectures, resources, discussions |
| 25 | Homework | `homework` | Add-on | Homework, assignments, submissions, grading |
| 26 | Assessment | `assessment` | Add-on | Question bank, quizzes, online exams, attempts |
| 27 | Payroll | `payroll` | Add-on | Salary structures, payroll runs, slips |
| 28 | Library | `library` | Add-on | Catalog, copies, circulation, fines |
| 29 | Transport | `transport` | Add-on | Vehicles, drivers, routes, stops, allocations |
| 30 | Inventory | `inventory` | Add-on | Items, stock, assets, maintenance |
| 31 | Administration | `administration` | Standard | Calendar, holidays, events, visitors, complaints, ID cards |
| 32 | Discipline | `discipline` | Add-on | Incidents, remarks, rewards, counseling |
| 33 | Reporting | `reporting` | Core | Dashboards, report definitions, scheduled reports |

Tier meaning: **Core** is in every plan, **Standard** is in paid plans, **Add-on** is enabled per organization through entitlements.

---

## 1.16 Canonical Naming Conventions (Binding for D2 to D15)

| Item | Convention | Examples |
|---|---|---|
| Tenant | The organization is the tenant. Tenant key column is `organization_id` on every tenant-owned table | `organization_id` |
| Branch scope | `branch_id` on every branch-owned table | `branch_id` |
| Other scope keys | `institution_id`, `academic_session_id`, `fiscal_year_id`, `shift_id` | |
| Tables | snake_case plural; domain pivots get domain names | `students`, `student_enrollments`, `teaching_assignments` |
| Primary keys | `id` BIGINT unsigned auto-increment; public-facing references use a separate `uuid` (ULID) column | `/app/students/01JD...` in URLs, never raw IDs |
| Money | DECIMAL(15,2) in the database; all arithmetic through the Core `Money` value object in integer minor units; never PHP floats | `total_amount`, `paid_amount` |
| Dates and times | Timestamps stored in UTC, displayed in organization timezone (Asia/Karachi default). School-day facts use a local DATE column | `attendance_date`, `due_date` |
| Permission keys | `{module}.{resource}.{action}` lowercase snake_case | `fee.voucher.generate`, `attendance.student_attendance.mark` |
| Standard actions | `view`, `create`, `update`, `delete`, `approve`, `reject`, `export`, `import`, `print`, `assign`, `publish`, `manage`, `restore` (brief's "Edit" maps to `update` to match Laravel policy methods) | |
| Role slugs | snake_case | `org_owner`, `class_teacher`, `guardian` |
| Portals | `platform` (super admin), `app` (all staff, incl. teachers), `student`, `guardian` | |
| Web routes | URL `/{portal}/{module}/{resource}`; name `{portal}.{module}.{resource}.{action}` | `app.fee.vouchers.index` |
| API routes | `/api/v1/{module}/{resource}`; name `api.v1.{module}.{resource}.{action}` | `api.v1.attendance.punches.store` |
| Webhooks | `/webhooks/{provider}/{organization_uuid}` | `/webhooks/zkteco-adms/...` |
| Events | Past tense | `StudentEnrolled`, `PaymentRecorded`, `ResultPublished` |
| Jobs | Imperative | `GenerateMonthlyVouchers`, `SyncDevicePunches` |
| Audit action keys | `{module}.{resource}.{past_tense_verb}` | `fee.payment.recorded`, `attendance.correction.approved` |
| Design tokens | CSS custom properties and Tailwind theme names; HEX values defined in Deliverable 9 | `--color-primary-600`, `primary`, `secondary`, `accent`, `canvas`, `surface`, `border`, `ink`, `ink-muted`, `success`, `warning`, `danger`, `info` |
| Blade components | `x-ui.*` for generic, `x-{module}.*` for module-specific | `x-ui.table`, `x-ui.modal`, `x-fee.voucher-card` |

---

## 1.17 Platform Decisions Summary

| ID | Decision | Detailed in |
|---|---|---|
| DEC-01 | Runtime policy: Laravel 12 now, PHP 8.4 runtime before 31 Dec 2026, Laravel 13 at gate UG-1 | 1.1 |
| DEC-02 | Laravel modular monolith, one deployable application, modules with explicit boundaries | D3 |
| DEC-03 | Shared MySQL database with row-level tenancy (`organization_id`, `branch_id`), enforced by global scopes plus policies plus tests; dedicated database possible later | D4 |
| DEC-04 | Hierarchy: platform, organization (tenant), institution, branch. Academic session is a time dimension owned by the institution, not a container under branch | D4, D6 |
| DEC-05 | Four portals in one design system: `platform`, `app`, `student`, `guardian`. Teachers, owners, principals and accountants share the `app` shell with role-specific dashboards and navigation, because staff often hold several roles | D10 |
| DEC-06 | Frontend: Blade components, Alpine.js, Tailwind CSS v4 via Vite. No SPA framework. Browser floor: Chrome/Edge 111+, Safari 16.4+, Firefox 128+ | D9 |
| DEC-07 | Money stored as DECIMAL(15,2), computed in integer minor units via `Money` value object | D7 |
| DEC-08 | UTC timestamps, local DATE for school-day facts | D7 |
| DEC-09 | Users belong to exactly one organization (platform staff have none). Login identifier unique per organization. A parent with children in two different organizations has two accounts | D4 |
| DEC-10 | Login with mobile number (normalized to E.164), email, or username/admission number for students. First guardian login sets password through SMS OTP | D5 |
| DEC-11 | English and Urdu from day one with RTL through CSS logical properties; Urdu font Noto Nastaliq Urdu (open license) | D9 |
| DEC-12 | PDF rendering through headless Chromium, because PHP-only PDF engines do not shape Urdu Nastaliq reliably. Exact package verified for Laravel 12/13 in D3 | D3 |
| DEC-13 | Hosting on a Linux VPS or cloud VM (Ubuntu LTS) with Supervisor, Redis, cron. Shared cPanel hosting is not supported | D12 |
| DEC-14 | Student placement stored per session in `student_enrollments`; the student record never stores "current class" as the source of truth | D6, D7 |
| DEC-15 | One generic approval engine used by every module | D11 |
| DEC-16 | Payment gateways use each organization's own merchant credentials; a payment is marked paid only by a verified server-side callback or reconciliation | D11 |
| DEC-17 | Attendance device adapter layer; R1 adapters: CSV import, ZKTeco ADMS push, signed webhook | D11 |
| DEC-18 | MySQL 8.4 LTS (minimum 8.0). MariaDB not supported, to avoid behavior differences in JSON, CTEs and window functions used by reports | D7 |
| DEC-19 | Repository pattern only for external adapters (devices, payments, SMS, WhatsApp) and complex reporting queries; Eloquent elsewhere | D3 |
| DEC-20 | Livewire not adopted. One interaction model (Blade plus Alpine plus small JSON endpoints) is simpler for AI agents to keep consistent | D3 |

---

## 1.18 Success Metrics

**Business and adoption**

| Metric | Target |
|---|---|
| Time from contract to first fee vouchers printed | 5 working days or less |
| Guardians activated within 60 days of school go-live | 60% or more |
| Daily active teachers marking attendance in system | 90% or more of sections per day |
| Fee vouchers paid through digital channels (after R2) | 30% within first year |
| Support tickets per 1,000 students per month | Fewer than 5 after month 3 |

**Operational performance (on 4 vCPU / 8 GB app server, MySQL on separate host)**

| Operation | Target |
|---|---|
| Teacher marks attendance for a 40-student section | Under 60 seconds of user time, save under 1 second |
| Monthly voucher generation for 10,000 students | Under 10 minutes, queued, progress visible |
| Result processing for 5,000 students in one exam term | Under 15 minutes, queued |
| Excel import of 5,000 students | Validation preview under 2 minutes, import queued |
| List screens (50 rows) | p95 server time under 800 ms |
| Dashboards | Under 2 seconds using cached aggregates |
| SMS burst (absence alerts for 2,000 students) | Queued within 1 minute, provider rate limits respected |

**Reliability and security**

| Metric | Target |
|---|---|
| Monthly uptime | 99.5% |
| Backup RPO / RTO | RPO 1 hour or less, RTO 4 hours or less |
| Cross-tenant access findings in penetration test | Zero |
| Tenant-owned models covered by isolation tests | 100% |
| Protected-record changes with audit entry | 100% |

---

## 1.19 Assumptions

| ID | Assumption |
|---|---|
| A-01 | The product is built by one developer (or a small team) using Google Antigravity as the coding agent, with Claude for architecture. Phase sizes are planned for that capacity. |
| A-02 | The development environment can be upgraded to PHP 8.4 before 31 Dec 2026 (DEC-01). |
| A-03 | Production runs on Linux VMs the operator controls, in a cloud region with low latency to Pakistan. Data residency requirements are handled per customer contract. |
| A-04 | Private institutions are the customers. No government reporting integrations are required. |
| A-05 | Academic sessions are defined per institution, and all branches of an institution share the session calendar. Branch-specific dates (e.g., admission window) are overrides. |
| A-06 | The fiscal year defaults to 1 July to 30 June (Pakistan), configurable per organization, and is separate from the academic session. |
| A-07 | Each organization uses one currency. PKR is default. |
| A-08 | Many guardians have no email. Mobile number is the primary guardian identifier. |
| A-09 | Students from Playgroup to Class 3 do not get logins by default (threshold configurable). Guardians act for them. |
| A-10 | Student data is visible to staff only within their assigned branches, unless their role has organization-wide scope. |
| A-11 | Board exam registration with BISE boards is done on board portals. The system exports data in a usable format but does not submit. |
| A-12 | Each organization signs its own agreements with SMS, WhatsApp and payment providers. The platform supplies adapters and stores credentials encrypted. |
| A-13 | Video content in R2 is mostly hosted externally; local uploads are capped per organization by plan. |
| A-14 | ZKTeco devices in the field are mixed. Some support ADMS push, some only LAN pull. R1 covers push, CSV and webhook; pull through an agent is R2. |
| A-15 | Office PCs at schools can run a browser meeting the floor in DEC-06. Windows 7 machines (last Chrome 109) are not supported. |
| A-16 | English is the internal working language for staff UI in R1; Urdu is complete for the guardian portal and printed documents in R1, and for all portals in R2. |
| A-17 | Grading schemes, pass marks, grace marks and ranking rules vary by institution, so the system ships editable templates rather than one fixed scheme. |
| A-18 | A user with roles in several branches chooses an active branch context; organization-wide roles can choose "All branches" for read views. |

---

## 1.20 Gaps Found and Added

Requirements missing from the brief that real Pakistani schools and colleges need, or that a production SaaS needs.

| ID | Gap | Decision | Release |
|---|---|---|---|
| GAP-01 | No SaaS commercial layer | Plans, module entitlements, limits (students, branches, storage, SMS credits), tenant states: trial, active, suspended, archived | R1 manual, R3 self-service |
| GAP-02 | No onboarding flow | Setup wizard: organization, institution, branches, session, class template, fee heads, grading template, first admin users | R1 |
| GAP-03 | No legacy migration path | Import templates for students, guardians, staff, opening fee balances and opening ledger balances | R1 |
| GAP-04 | No family concept | `families` grouping siblings; family voucher option; sibling discount rules; family ledger view | R1 |
| GAP-05 | No shifts | Morning/evening shift per branch; sections, timetable, attendance rules per shift | R1 |
| GAP-06 | No gender-wing handling | Boys/girls wings within one branch as a section attribute; optional restriction of staff visibility and photo display by wing | R1 |
| GAP-07 | Local challan format missing | Printable 3-copy bank challan (bank, school, student copies) with multiple bank accounts and payment reference | R1 |
| GAP-08 | Local fee practices missing | Vacation-month advance fees, struck-off after N unpaid months with re-admission fee, refundable security deposit, staff-child concession, orphan and similar concession categories, per-day or fixed late fine | R1 |
| GAP-09 | Sponsored students missing | Funding category for students sponsored by programs such as Punjab Education Foundation partner schemes, NGOs or trusts, with separate reporting | R1 |
| GAP-10 | Taxes on vouchers | Configurable tax lines on vouchers with no hardcoded rates | R1 |
| GAP-11 | GR number missing | General Register number per branch alongside admission number and roll number, all with configurable formats | R1 |
| GAP-12 | Urdu names | Optional Urdu name fields for student and father, used on certificates | R1 |
| GAP-13 | Daily diary missing | Teacher's daily note per section (homework, reminders) visible to guardians; R1 substitute for full homework module | R1 |
| GAP-14 | Period locks missing | Month locks for attendance, fee months and accounting periods; changes after lock need approval | R1 |
| GAP-15 | Concurrency-safe numbering | Central numbering service for vouchers, receipts, admissions, requests: sequential per branch and year, safe under parallel requests | R1 |
| GAP-16 | Fiscal year missing | Fiscal year entity separate from academic session for accounting | R1 |
| GAP-17 | Cash counter controls | Cashier session open/close, cash handover, variance recording | R1 |
| GAP-18 | Reversals | No deletion of payments or journals; reversal entries with reason and approval | R1 |
| GAP-19 | Government-notified closures | Bulk "institution closed" declaration (smog, heat, security) that updates calendar, attendance expectations and timetable | R1 |
| GAP-20 | Guardian identity without email | Mobile-first login, OTP set-password, verification code printed on admission slip or voucher for linking, handling of one mobile shared by several guardians | R1 |
| GAP-21 | Minors' privacy and consent | Photo consent, data notice, guardian consent record, retention rules, data export when a student leaves; designed against PECA 2016 and general data protection principles, with pending national data protection legislation monitored | R1 |
| GAP-22 | Age-appropriate access | No student logins below a configurable class; parent approval step optional for adult students and college programs | R1 |
| GAP-23 | Support access | Platform "login as" only with tenant consent, time-limited, visible banner, fully audited | R1 |
| GAP-24 | SMS cost control | Segment counter (Urdu Unicode SMS costs about double), per-organization quotas, usage reports | R1 |
| GAP-25 | Staff attendance rules | Shift timings, grace minutes, late-to-half-day conversion (e.g., 3 lates = 1 leave), feeding payroll | R1 rules, R2 payroll link |
| GAP-26 | Exam specifics | Admit cards, grace marks, supplementary/re-exams, subject-level pass rules, absent vs zero distinction | R1 |
| GAP-27 | Elective selection | Students in one section can take different electives; subject enrollment per student | R1 |
| GAP-28 | Idle session security | Auto-logout on idle for shared office PCs, configurable per role | R1 |
| GAP-29 | Accessibility standard | WCAG 2.2 AA as target | R1 |
| GAP-30 | Browser policy | Supported-browser banner and documented floor (DEC-06) | R1 |
| GAP-31 | Admission inquiries | Lead capture during admission season: inquiry source, follow-ups, conversion | R2 |
| GAP-32 | Board exam support | Board registration exports, board roll numbers, board result entry and analysis | R2 |
| GAP-33 | Arrangement periods | Substitute teacher assignment when a teacher is absent, linked to staff attendance and timetable | R2 |
| GAP-34 | Payroll statutory deductions | EOBI, PESSI, income tax withholding, provident fund, salary advance and loan recovery as configurable deduction types | R2 |
| GAP-35 | Document verification | QR code on certificates, report cards and receipts linking to a minimal public verification page | R2 |
| GAP-36 | Scheduled reports | Daily collection summary emailed to owner, monthly attendance summary to principal | R2 |
| GAP-37 | Messaging safeguards | Guardian-teacher messaging within school hours, moderated, no private numbers exposed, fully logged | R2 |
| GAP-38 | Help and training | In-app help, Urdu user guides, demo tenant separate from production | R2 |
| GAP-39 | Tenant offboarding | Full data export package and archival when an organization leaves | R2 |
| GAP-40 | Storage quotas | Per-organization storage limits, image compression, video size caps | R2 |
| GAP-41 | Offline marking | Offline-capable attendance marking for poor connectivity classrooms | R3 |
| GAP-42 | Single-tenant restore | Logical export/restore tooling for one organization from shared-database backups | R3 |
| GAP-43 | Hostel management | Schema room left, delivered as add-on later | Post-R3 |
| GAP-44 | Complaints with SLA | Guardian complaints and feedback with tracking | R3 |

---

## 1.21 Conflicts and Questionable Assumptions in the Brief

| ID | Issue in brief | Resolution |
|---|---|---|
| CON-01 | PHP 8.2 / Laravel 12 lose security support before go-live | DEC-01 |
| CON-02 | "Bootstrap 5 or Tailwind" left open | Tailwind CSS v4 (DEC-06): CSS-variable design tokens, built-in RTL logical utilities, default tooling in current Laravel. Trade-off: older browsers unsupported (A-15) |
| CON-03 | Hierarchy places academic session under branch | Session is a time dimension owned by the institution and shared by its branches. Classes and sections are per branch per session (DEC-04) |
| CON-04 | Brief uses `tenant_id` | Organization is the tenant, so the column is `organization_id`. One concept, one name (1.16) |
| CON-05 | Separate teacher portal vs. many staff holding multiple roles | Teachers use the `app` portal with a teacher dashboard and navigation; a teacher who is also coordinator switches role without logging out (DEC-05) |
| CON-06 | Parent approval required for student modification requests | Configurable per request type; parent step skipped for adult students or college programs where the institution chooses (GAP-22) |
| CON-07 | "Students mark attendance using devices" applied to all ages | Teacher marking is the primary student method. Device or gate punches are a second source. Deliverable 11 defines a precedence rule for the final daily status |
| CON-08 | GPA listed as a core result output | Punjab boards report marks and grades for matric and intermediate. GPA is an optional grading scheme setting |
| CON-09 | Ranking and positions | Configurable per exam, can be hidden from guardians and students |
| CON-10 | Reverb listed in core stack | Optional. Needs a long-running process; every realtime feature has a polling fallback |
| CON-11 | "Never hardcode roles" vs. needing roles on day one | System role templates are seeded per organization as editable copies. Only `platform_super_admin` and the organization owner's lockout protection are locked |
| CON-12 | Soft deletes listed generally | Soft deletes for master data only (e.g., a subject). Financial transactions use reversals, attendance uses corrections, marks use versions (PR-02) |
| CON-13 | Brief covers ~30 modules at full depth | Release slicing (1.12) and phase gates prevent a wide but shallow build |
| CON-14 | "School & College" while levels stop at Intermediate | "College" means intermediate college in this product. Degree programs are a non-goal (1.14), and the academic model stays generic |

---

## 1.22 Top Risks

| ID | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| RISK-01 | Running on unsupported PHP/Laravel | High if ignored | High | DEC-01, CI matrix, gate UG-1 |
| RISK-02 | Cross-tenant or cross-branch data leak through a missed scope (raw query, queued job, export, file URL) | Medium | Critical | Global scopes, policies, scoped jobs, signed file URLs, automated isolation test for every tenant-owned model |
| RISK-03 | AI agent produces wide but shallow features or breaks earlier work | High | High | Small phase prompts, "already built" section in every prompt, acceptance tests, verification checklist (D15) |
| RISK-04 | Biometric device diversity and LAN-only devices | High | Medium | Adapter layer, ADMS push, CSV fallback, sync agent in R2 |
| RISK-05 | Payment gateway onboarding slow per organization, documentation under NDA | High | Medium | Manual collection and bank challan in R1; gateways in R2 per organization |
| RISK-06 | Urdu Nastaliq rendering broken in PDFs or UI | Medium | High | Chromium-based PDF (DEC-12), Phase 0 technical spike with real report card and voucher |
| RISK-07 | Low guardian adoption | Medium | High | Urdu, mobile-first, SMS links with OTP onboarding, WhatsApp in R2 |
| RISK-08 | Load spikes at month start (vouchers, SMS) and result season | Medium | Medium | Queues, chunking, rate limits, cached aggregates |
| RISK-09 | Poor legacy data quality on import | High | Medium | Preview, row-level validation, failed-row download, duplicate detection |
| RISK-10 | Accounting errors from generated code | Medium | High | Single posting service as the only path to journals, invariant tests (debits equal credits), period locks |
| RISK-11 | Regulatory change (data protection, SBP payment rules, PTA SMS rules) | Medium | Medium | Adapters, configurable policies, legal review before launch |
| RISK-12 | Restoring one tenant from a shared database backup | Low | High | Logical per-organization export/restore tooling (GAP-42), daily verified backups |
| RISK-13 | Storage cost growth from videos and scans | Medium | Medium | Quotas, compression, external video links, S3-compatible storage |

---

## 1.23 Glossary

| Term | Meaning |
|---|---|
| Organization | The tenant: the legal or business owner using the platform |
| Institution | A school or college brand under an organization (e.g., "City School" and "City College") |
| Branch | A physical campus of an institution |
| Shift | Morning or evening operating schedule within a branch |
| Academic session | Teaching year of an institution, e.g., 2026-2027 |
| Fiscal year | Accounting year, default 1 July to 30 June |
| Enrollment | A student's placement in a class and section for one academic session |
| Family | A group of siblings sharing guardians, used for family vouchers and sibling discounts |
| B-Form | NADRA Child Registration Certificate number for minors |
| CNIC | Computerized National Identity Card number |
| GR number | General Register number, the entry number in the school's admission register |
| BISE | Board of Intermediate and Secondary Education (e.g., BISE Lahore), conducts matric and intermediate exams in Punjab |
| FBISE | Federal Board of Intermediate and Secondary Education |
| IBCC | Inter Board Committee of Chairmen, coordinates boards and equivalence |
| Matric (SSC) | Class 9 and 10 secondary school certificate |
| Intermediate (HSSC) | Part 1 and Part 2 after matric (F.Sc Pre-Medical, F.Sc Pre-Engineering, ICS, I.Com, FA) |
| Challan / voucher | Fee payment slip, commonly printed in three copies |
| Struck off | Student removed from rolls after unpaid fees for a set period, re-admission fee applies |
| SLC | School Leaving Certificate |
| Daily diary | Daily note from teacher to guardians about homework and reminders |
| Arrangement period | Period covered by a substitute teacher |
| PEF | Punjab Education Foundation, funds students in partner private schools |
| EOBI / PESSI | Employees' Old-Age Benefits Institution / Punjab Employees Social Security Institution |
| 1Bill | 1LINK bill payment service, lets payers pay through any bank app using a consumer number |
| Raast | State Bank of Pakistan's instant payment system |
| ADMS push | ZKTeco protocol where the device sends punches to a server over HTTP |

---

## 1.24 What Deliverable 2 Covers

Deliverable 2: Detailed Functional Requirements and Module Breakdown. For each of the 33 modules in 1.15: purpose, functional requirements with IDs (FR-{MODULE}-NNN), user stories per role, business rules, inputs and outputs, dependencies, R1/R2/R3 tagging, and acceptance criteria. Requirement IDs from D2 are referenced by the screen inventory (D10), workflows (D11), Antigravity prompts (D14) and the verification checklist (D15).
