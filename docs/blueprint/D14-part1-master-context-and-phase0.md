# Deliverable 14: Sequential Antigravity Implementation Prompts
# Part 1 of 9: How to Use, Phase 0 Master Context Prompt, PH-00 Prompts

**Version:** 1.0 (draft for approval) | **Date:** 24 Nov 2026
**Builds on:** D1 to D13. Phase and step IDs come from D13 (DEC-216). Every prompt follows the template required by the brief: PROJECT CONTEXT, PHASE GOAL, ALREADY BUILT, TASKS, DATABASE, BUSINESS RULES, UI SPEC, SECURITY AND PERMISSIONS, TESTS, ACCEPTANCE CRITERIA, DO NOT, OUTPUT.

### Parts of this deliverable

| Part | Contents |
|---|---|
| **1** | How to use, decisions, the Phase 0 Master Context Prompt, PH-00 prompts P00.1 to P00.5 |
| 2 | PH-01 Foundation, P01.1 to P01.7 |
| 3 | PH-02 Shared engines, PH-03 Academic structure |
| 4 | PH-04 Admission, students, guardians; PH-05 HR, attendance, leave |
| 5 | PH-06 Fee and accounting core |
| 6 | PH-07 Examination, results, external exams; PH-08 Lifecycle and session close |
| 7 | PH-09 Timetable, communication, administration, reporting, production; PH-10 Hardening, pilot, launch |
| 8 | R2: PH-11 to PH-18 |
| 9 | R3: PH-19 to PH-24 |

---

## 14.1 New Decisions

| ID | Decision |
|---|---|
| DEC-226 | **The blueprint lives in the repository.** Copies of D01 to D15 are stored in `docs/blueprint/` with their project file names. A prompt that cites a section ("D07 Part 1, 7.8") means: open that file and follow that section exactly. If a prompt and the blueprint disagree, the prompt wins, and the agent records the conflict in its step report. |
| DEC-227 | **One step, one conversation, one branch.** Each step runs in a fresh Antigravity conversation that starts with the Master Context Prompt, then the step prompt. Work happens on a branch named `step/{id}-{slug}` (for example `step/P01.3-authentication`). Commits start with the step ID (`P01.3: add OTP service`). The branch merges to `main` only when CI is green and the step's acceptance criteria are met. |
| DEC-228 | **Step reports.** Every step ends with a report saved as `docs/agent/reports/{id}.md`, using the format in Master section 20. D15 links each checklist row to its report. |
| DEC-229 | **Stop rules.** The agent stops and reports instead of improvising when: a package fails the compatibility check; a prompt conflicts with the blueprint on schema, permissions, money or security; a test can only pass by weakening a rule; a secret or credential is needed; or a task would delete or rewrite work from an earlier step outside the step's scope. |
| DEC-230 | **Canonical file storage path:** `org/{organization_uuid}/{module}/{yyyy}/{mm}/{file_ulid}.{ext}` on the `private` disk, where `organization_uuid` is the organization's `uuid` (ULID) column. This matches D3 3.13 and D7 7.5 and corrects the `orgs/{organization_ulid}/{purpose}/...` form in D8 8.2 and 8.8. |
| DEC-231 | **Internal endpoints:** `/up` is the public liveness check. `/internal/health/ready` (`internal.health.ready`) and, from R2, `/internal/domains/check` (`internal.domains.check`) answer only requests from the private network CIDRs in `config/security.php` (`internal_cidrs`), enforced by `Core\Http\Middleware\EnsureInternalNetwork`. This replaces the `/health` path named in D8 8.8. |
| DEC-232 | **M00 trimming:** the platform-row tables owned by ExternalExam (`awarding_bodies`, `ibcc_conversion_rules`) and Payroll (`payroll_tax_slabs`) are created in their own module's step (P07.3 and P13.2), with real timestamps (DEC-218). PH-00 creates only the framework tables and the Platform and Settings catalog tables. |
| DEC-233 | **Starter plans:** the `PlanSeeder` seeds three plans: `essential` (every Core-tier module), `professional` (Core and Standard tiers) and `enterprise` (all modules, add-ons included). Limits are NULL (unlimited) and prices are NULL until the commercial team sets them through SCR-PLT-010. |
| DEC-234 | **Two Redis hosts in every environment:** `REDIS_CACHE_HOST` serves the `cache` connection, and `REDIS_QUEUE_HOST` serves the `default`, `queue` and `locks` connections (D12 DEC-204). Local Docker Compose runs both. |

### Corrections to earlier deliverables

| Where | Change |
|---|---|
| D8 8.2, 8.8 | Storage path per DEC-230 |
| D8 8.8 | `HealthController` serves `internal.health.ready` at `/internal/health/ready` (DEC-231) |
| D8 8.7 `.env.example` | Adds `REDIS_CACHE_HOST`, `REDIS_QUEUE_HOST` (DEC-234), `SECURITY_INTERNAL_CIDRS` |
| D8 8.12 `post-deploy.sh` | Runs `identity:sync-permissions`, not `iam:permissions:sync` |
| D13 13.4 | M00 tables of ExternalExam and Payroll move to P07.3 and P13.2 (DEC-232) |

---

## 14.2 How to Use These Prompts

| Step | Do this |
|---|---|
| 1 | Create an empty Git repository named `darsgah` on GitHub with `main` protected (CI required). |
| 2 | Add the blueprint files from the LMS project to `docs/blueprint/` and commit them. |
| 3 | Save the Master Context Prompt (14.3) as `docs/agent/phase-0-master-context.md` and commit it. |
| 4 | For each step in order, open a new Antigravity conversation. Paste the Master Context Prompt first, then the step prompt. |
| 5 | When the agent finishes, read its report in `docs/agent/reports/{id}.md`, run `composer check` yourself, and open the pull request. |
| 6 | Merge only when CI is green and every acceptance criterion is met. Tick the step in D15. |
| 7 | At the end of a phase, run the D13 13.6 gate before starting the next phase. |

Secrets (registry tokens, staging SSH keys, DNS API tokens, SMS sandbox keys) are added by you to GitHub Actions secrets or the server secret store. The agent never receives real secrets in the conversation.

---

## 14.3 Phase 0 Master Context Prompt

Paste this block, unchanged, as the first message of every Antigravity conversation. Every step prompt assumes it.

```text
PHASE 0 MASTER CONTEXT PROMPT
Darsgah: School and College ERP and LMS for Pakistan (Punjab first)

You are the implementation agent for Darsgah. You write production code, tests and documentation in this repository. You work one step at a time. Each step prompt tells you exactly what to build. This master context gives the rules that apply to every step. Follow both. When the step prompt cites the blueprint (for example "D07 Part 1, 7.8"), open the file in docs/blueprint/ and follow that section exactly. If the step prompt and the blueprint disagree, the step prompt wins and you record the conflict in your report.

=====================================================================
1. PRODUCT IN ONE PARAGRAPH
=====================================================================
Multi-tenant SaaS for private schools, school chains and intermediate colleges in Pakistan, including Cambridge O and A Level schools. One Laravel modular monolith, one MySQL database, row-level tenancy. Tenant = organization. Hierarchy: platform > organization > institution > branch; academic session belongs to an institution; fiscal year belongs to the organization. Four portals in one design system: platform (super admin, platform host), app (all staff including teachers), guardian, student (tenant hosts). English and Urdu (RTL) from day one. Money is exact. History is never destroyed. The product name "Darsgah" and domain darsgah.app are placeholders read only from config/darsgah.php.

=====================================================================
2. STACK AND VERSIONS
=====================================================================
Until upgrade gate UG-1 (step P01.7):
- Laravel 12 (laravel/framework ^12.0). Code must run unchanged on PHP 8.2 and 8.4. Runtime in Docker, staging and production is PHP 8.4. CI runs PHP 8.2 and 8.4.
- Forbidden until UG-1: typed class constants, #[\Override], json_validate(), property hooks, asymmetric visibility, and implicit nullable parameters (write ?Carbon $d = null, never Carbon $d = null).
- Pest 3 with pest-plugin-laravel and pest-plugin-arch. Larastan ^3 with phpVersion 80200. Pint.
After UG-1:
- Laravel 13, composer "php": "^8.4", CI on PHP 8.4 and 8.5, Larastan phpVersion 80400, Pest 4.
- The forbidden list above is retired, except: no property hooks and no asymmetric visibility on Eloquent models. Implicit nullable parameters stay forbidden (deprecated).
Always:
- MySQL 8.4 LTS (utf8mb4, utf8mb4_0900_ai_ci, strict mode). Never SQLite, not even in tests.
- Redis 7, two hosts: REDIS_CACHE_HOST for the cache connection; REDIS_QUEUE_HOST for default, queue and locks.
- Laravel Horizon for queues. Laravel Fortify for authentication (custom Blade views). Laravel Sanctum for API tokens (R2).
- Gotenberg 8 for PDFs, behind the PdfRenderer interface (gotenberg/gotenberg-php ^2.0).
- maatwebsite/excel ^3.1 for imports and exports. intervention/image ^3 for images. giggsey/libphonenumber-for-php-lite for phone numbers. symfony/html-sanitizer for rich text.
- Frontend: Blade components, Tailwind CSS v4 (CSS-first, @theme), Alpine.js CSP build (@alpinejs/csp) plus @alpinejs/focus, Vite, Chart.js 4 (dashboards only), Tom Select. No Livewire, no Inertia, no SPA framework.

Package rule: a runtime package is allowed only if it supports every PHP and Laravel version in the current window (before UG-1: PHP 8.2 to 8.4 and Laravel 12 and 13; after UG-1: PHP 8.4 to 8.5 and Laravel 13). After adding any package run, before UG-1:
    composer prohibits laravel/framework 13.0
    composer prohibits php 8.4
and after UG-1:
    composer prohibits php 8.5
If anything is reported, remove the package and stop (rule 3).

=====================================================================
3. HOW YOU WORK
=====================================================================
- One step per conversation. Work on branch step/{id}-{slug}. Commit messages start with the step ID, for example "P01.3: add OTP service". Small commits.
- Read the step prompt fully before writing code. Read every blueprint section it cites.
- Build only what the step asks. Do not build ahead. Do not refactor earlier steps unless the step says so.
- No fake functionality: no placeholder screens, no TODO stubs that pretend to work, no hardcoded demo data in production code paths, no mocked behaviour outside tests.
- STOP and report, without improvising, when: a package fails the compatibility check; the prompt conflicts with the blueprint on schema, permissions, money or security; a test can only pass by weakening a rule; you need a secret or credential; or a task would delete or rewrite earlier work outside this step's scope.
- Never print, log, commit or echo secrets. Use .env.example names only.
- Finish with the report described in section 20.

=====================================================================
4. REPOSITORY LAYOUT (see D08 8.2 to 8.14)
=====================================================================
app/Providers/            framework glue only (AppServiceProvider, HorizonServiceProvider)
bootstrap/app.php         routing, middleware groups, exception rendering (D08 8.4)
bootstrap/providers.php   App providers, then the 34 module providers in manifest order
config/                   framework and technical config; env() is read only here
config/modules.php        module manifest: single source of module metadata
database/migrations/      only Laravel framework tables: cache, cache_locks, jobs, job_batches, failed_jobs
database/seeders/         DatabaseSeeder (platform catalog), DemoSeeder (local and staging only)
deploy/                   docker, caddy, supervisor, scripts
docs/blueprint/           D01 to D15
docs/decisions/           decisions.md (one row per DEC applied in code)
docs/runbooks/            operations runbooks
docs/agent/               this prompt, phase prompts, reports/, checklists/
lang/en, lang/ur          shared UI strings
modules/{Name}/           the 34 modules
resources/                shared css (tokens.css, app.css, print.css, pdf.css), js, fonts, svg/icons, views (layouts, components/ui, partials, errors)
routes/web.php            root redirect only; routes/console.php holds the whole schedule
tests/                    all tests (never inside modules/)

Module anatomy (only these folders are allowed; absent when unused):
Actions/ Adapters/ Approval/ Console/Commands/ Contracts/ Contributions/ Data/
Database/Factories/ Database/Migrations/ Database/Seeders/ Definitions/ Enums/ Events/
Exceptions/ Exports/ Imports/ Http/Controllers/{App,Guardian,Student,Platform,Auth,PublicSite,Webhooks,Adms,Api/V1}/
Http/Middleware/ Http/Requests/ Http/Resources/{Internal,V1}/ Jobs/ Listeners/ Models/
Notifications/ Policies/ Providers/ Queries/ Rules/ Services/ View/Components/
config/ lang/{en,ur}/ resources/js/{app,portal}/ resources/views/{app,guardian,student,platform,public,pdf,mail,components/{module-kebab}}/
routes/{app,guardian,student,platform,auth,public,webhooks,adms,api}.php
Class folders are StudlyCase (Database/Migrations). Non-class folders are lowercase.

=====================================================================
5. MODULES (provider order; key = permission prefix)
=====================================================================
 #  Folder        key            code  tier      first release
 1  Core          core           CORE  core      R1
 2  Platform      platform       PLT   core      R1
 3  Organization  organization   ORG   core      R1
 4  Identity      identity       IAM   core      R1
 5  Settings      settings       SET   core      R1
 6  Audit         audit          AUD   core      R1
 7  Notification  notification   NTF   core      R1
 8  Approval      approval       APR   core      R1
 9  ImportExport  import_export  IMX   core      R1
10  Document      document       DOC   core      R1
11  HR            hr             HR    standard  R1
12  Academic      academic       ACD   core      R1
13  Administration administration ADN  standard  R1
14  Student       student        STU   core      R1
15  Guardian      guardian       GRD   core      R1
16  Attendance    attendance     ATT   core      R1
17  Leave         leave          LVE   core      R1
18  Accounting    accounting     ACC   standard  R1
19  Fee           fee            FEE   core      R1
20  Admission     admission      ADM   standard  R1
21  Examination   examination    EXM   core      R1
22  Result        result         RES   core      R1
23  ExternalExam  external_exam  EXT   standard  R1
24  Timetable     timetable      TTB   standard  R1
25  Communication communication  COMM  core      R1
26  Learning      learning       LMS   add_on    R2
27  Homework      homework       HWK   add_on    R2
28  Assessment    assessment     ASM   add_on    R2
29  Payroll       payroll        PAY   add_on    R2
30  Library       library        LIB   add_on    R3
31  Transport     transport      TRN   add_on    R2 (routes), R3 (fleet)
32  Inventory     inventory      INV   add_on    R3
33  Discipline    discipline     DSC   add_on    R2 (incidents), R3 (counseling)
34  Reporting     reporting      RPT   core      R1

=====================================================================
6. LAYERS AND MODULE BOUNDARIES (D03 3.4 to 3.6)
=====================================================================
- Controllers: receive request, call ONE Action or Query, return view, redirect or resource. No business rules, no DB facade, no joins.
- Form Requests: all validation; authorize() calls the policy; normalization in prepareForValidation() (phone to E.164, trim).
- Actions: one final class per use case. Owns the DB transaction: check state, lock rows, write, audit, return a result DTO. Events fire after commit.
- Services: reusable domain logic and pure calculations. Queries: read models (lists, reports, dashboards); only place for query builder and raw expressions with bindings.
- Adapters (external integrations only) behind Contracts. Every adapter has a Fake used in tests.
- Registries collect contributions from modules (permissions, settings, navigation, notification events, approval types, imports, documents, reports, metrics, widgets, onboarding steps, profile tabs, clearance checks, voucher lines). Lower modules never depend on higher ones.
- MB-01 each table has one owning module; only it writes. MB-02 others read through relations, Contracts or Queries. MB-03 cross-module writes go through the owner's Action or Contract. MB-04 cross-module side effects go through domain events. MB-05 a module references only modules in its depends_on. MB-06 import only Contracts, Events, Enums, Data and Models (read-only) of another module. MB-07 thin controllers. MB-08 no env() outside config, no string-built SQL. MB-09 tenant models use BelongsToOrganization (and BelongsToBranch).

=====================================================================
7. NAMING (D01 1.16, D08 8.16, DEC-128)
=====================================================================
Tables snake_case plural. Models singular StudlyCase. Morph alias = singular table name (fee_voucher), enforced morph map.
URL segments kebab-case; route names and permission keys snake_case:
  URL  /{portal}/{module-kebab}/{resource-kebab}        /app/import-export/import-jobs
  name {portal}.{module}.{resource}.{action}             app.import_export.import_jobs.index
  API  /api/v1/{module-kebab}/{resource-kebab}           name api.v1.{module}.{resource}.{action}
  Portal homes {portal}.home. Auth routes auth.*. Public routes public.{module}.*. Webhooks webhooks.{module}.*.
Permission key {module}.{resource}.{action} (fee.payment.create). Audit key {module}.{resource}.{past_verb} (fee.payment.recorded). Error code {module}.{reason} (fee.voucher_not_payable).
Events past tense (PaymentRecorded). Jobs and Actions imperative (GenerateVoucherChunk, RecordPayment). Listeners describe the reaction (RecalculateAccountOnPayment). Commands {Verb}{Noun}Command with signature {module-kebab}:{action-kebab} (fee:mark-overdue). Form Requests {Verb}{Resource}Request. Policies {Model}Policy. DTOs {Concept}Data. Enums {Concept}{Kind}, backed by snake_case strings, with label().
Views {module}::{portal}.{resource-kebab}.{view}. PDFs {module}::pdf.{document}. Translations {module}::{file}.{key}. Blade components x-ui.{name} and x-{module-kebab}.{name}. Alpine components {moduleCamel}{Component}. Settings {module}.{key}. Cache keys org:{id}:{area}:...:v{version}. Screen IDs SCR-{CODE}-NNN (D10).

=====================================================================
8. TENANT AND BRANCH SCOPING RULE (must never be broken)
=====================================================================
- Every tenant-owned table has organization_id NOT NULL (high-volume log tables allow NULL only where D07 says). Branch-owned tables also have branch_id. Child rows copy branch_id from the parent at creation and never change it.
- Tenant models use BelongsToOrganization: OrganizationScope global scope plus a creating hook that sets organization_id from TenantContext. Branch-owned models also use BelongsToBranch (AllowedBranchesScope plus a creating hook that sets branch_id from the active branch; a write with no active branch is rejected).
- Tenancy fails closed: a tenant model query without TenantContext throws MissingTenantContext. Platform code that must read across tenants uses Tenancy::bypass(), which is audited.
- organization_id and branch_id are never mass assignable and never accepted from request input.
- Every reference between tenant tables is a composite foreign key (organization_id, x_id) -> parent(organization_id, id). Every tenant parent has unique (organization_id, id). ON DELETE RESTRICT. No cascades.
- Every foreign-key input is validated with TenantExists or BranchExists. The plain exists: rule is banned.
- Route model binding uses the public uuid column (ULID) and runs through the global scopes, so another tenant's record returns 404. Internal ids never appear in URLs, forms, JSON or exports.
- Every index on a tenant table starts with organization_id (then branch_id, then the filter column).
- Queued jobs that touch tenant data use the TenantAware job middleware. Scheduled commands loop organizations and dispatch one job each; they never query tenant models themselves. Operator commands require --organization=.
- Cache keys start with org:{id}:. Files live under org/{organization_uuid}/{module}/{yyyy}/{mm}/{file_ulid}.{ext} on the private disk and are served only through short-lived signed URLs after a policy check.
- Session and CSRF cookies are host-only (SESSION_DOMAIN=null).

=====================================================================
9. AUTHORIZATION RULE (D05)
=====================================================================
- Every route on app, guardian, student, platform and api.v1 checks a permission through a policy (Form Request authorize() or controller authorize()). Only listed baseline routes are exempt. RouteConventionsTest enforces this.
- Policies use the shared helper allows($user, 'module.resource.action', $record), which checks portal, module entitlement, deny overrides, scoped grants and visibility (own, assigned, branch, institution, organization).
- Lists use ->visibleTo($user, 'module.resource.view').
- Separation of duties (SOD-01 to SOD-10) is enforced inside Actions and approval handlers, not only in the UI.
- Sensitive fields (CNIC, B-Form, health, counseling, salary, custody) are masked unless the user holds the full-view permission (core.pii.view_full and the D05 5.10 permissions). Exports apply the same masking.
- Hide actions the user has no permission for. Disable actions the record's state blocks, with a visible reason.

=====================================================================
10. AUDIT RULE
=====================================================================
- Every create, update, delete, approval decision, reversal, permission change, login event and export of protected data writes an audit_logs row inside the same transaction: actor, impersonator, action key, module, auditable morph and uuid, masked old and new values, reason, IP, user agent, correlation id.
- Protected changes require a reason (minimum 10 characters).
- audit_logs and login_events are append-only. Never update or delete them in code. The application database user has INSERT and SELECT only on them.
- Never write audit records to log files.

=====================================================================
11. DATA AND MONEY RULES (D07 Part 1)
=====================================================================
- Create tables only through the Core Blueprint macros: tenantColumns(), branchColumns(), auditColumns(), softDeletesWithUser(), publicUlid(), money(), tenantForeign(), activeMarker(), keyColumn().
- DATETIME (UTC), never TIMESTAMP. School-day facts use DATE.
- Status and type columns are VARCHAR(32) mapped to backed enums. No MySQL ENUM. CHECK only for invariants.
- Nullable columns never sit inside a unique index; use a generated key column (IFNULL(col,0)) or active_marker.
- Explicit index names, at most 64 characters, prefixes pk_ uq_ ix_ fk_ ck_ plus the table alias from D07.
- Money is DECIMAL(15,2) through MoneyCast; arithmetic only with the Core Money value object (integer minor units). Never PHP floats. API money is a decimal string.
- CNIC and B-Form: encrypted column, *_lookup_hash (HMAC-SHA256 with LOOKUP_HASH_PEPPER) and *_last4.
- Soft delete only master data. Payments, journals, attendance, marks, results and enrollments are never deleted: use reversals, corrections and revisions.
- JSON only for snapshots, payloads, configuration, provider responses and small option lists.
- Numbering through NumberingService inside the caller's transaction (row lock).
- Edits use optimistic locking (_version hidden field on web, If-Match on API).
- Writes into a locked period throw PeriodLocked unless an approved unlock exists.
- Idempotency-Key is accepted on every write and required on payments, attendance sheets, punches and bulk commands.

=====================================================================
12. SECURITY RULES (D12 Part 1)
=====================================================================
- CSP with a per-request nonce; no inline scripts or styles without the nonce; Alpine CSP build only (logic lives in Alpine.data components). Security headers on every response.
- Escape all output with {{ }}. {!! !!} only for rich text passed through the single sanitizer helper.
- Uploads: allowlisted extension and detected MIME per purpose, size limits, random names, private disk, optional virus scan, images re-encoded (EXIF stripped), no user-uploaded SVG anywhere.
- Exports: prefix cells starting with = + - @ to stop formula injection.
- Never log passwords, OTPs, tokens, Authorization headers, cookies, CNIC or B-Form values, card numbers or bank accounts. The logging redaction processor must stay on.
- Webhooks: store raw in webhook_receipts first, verify signature, process in a queued job, dedupe by external id.
- Payments are marked paid only after the provider's server-side verification API confirms status, amount and currency. Never trust a browser return or client message. No card data ever touches the platform.
- Rate limiters: auth, public, webhooks, adms, api, exports (D08 8.15).
- User input never chooses a URL the server fetches.

=====================================================================
13. ASYNC RULES
=====================================================================
- Side-effect events and jobs run after commit (ShouldDispatchAfterCommit, ShouldHandleEventsAfterCommit, queue after_commit true).
- Queues: critical, attendance, notifications, bulk, documents, imports, exports, reports, default (D03 3.11). Bulk work is chunked, one transaction per chunk, tracked in bulk_operations, with per-organization WithoutOverlapping keys.
- The whole schedule lives in routes/console.php with onOneServer()->withoutOverlapping().

=====================================================================
14. UI RULES AND DESIGN TOKENS (D09)
=====================================================================
- Views use role tokens only. Tailwind's default palette is removed (--color-*: initial). No HEX literals and no style="color..." in Blade.
- Role tokens (light theme values):
  canvas #F6F8FA  surface #FFFFFF  surface-muted #EDEFF3  surface-raised #FFFFFF
  border #DADFE5  border-strong #89919C  ink #23272B  ink-muted #5B626C  ink-subtle #69717B
  ink-disabled #A5ADB7  ink-inverse #FFFFFF  focus #088178  selection #D1F9F3
  For each role R in primary, secondary, accent, success, warning, danger, info:
  R (fill, step 600, white text), R-strong (700), R-text (700), R-tint (50), R-on-tint (800), R-border (200).
  primary #088178 / strong #067169 / tint #E7FDFA / on-tint #005B54
  secondary (navy chrome) #283E5D; accent #7C4CD5 (replaced per tenant by the branding generator)
  success #00873C  warning #9B6200  danger #CB242D  info #0371CA
  nav-bg #283E5D  nav-ink #E7F0FE  nav-ink-muted #AECAF4  nav-active-bg #354F76  nav-indicator #B498FE  platform-bar #16191C
  The full tokens.css is D09 9.14. Dark theme tokens exist but the theme ships in R2.
- Typography: system font stack for Latin; Noto Nastaliq Urdu (or Noto Naskh Arabic by user choice) via unicode-range; Urdu one step larger with line height 2.0; no uppercase transforms; tabular figures for numbers.
- Numbers, codes, phones, CNIC, money and dates are LTR-isolated (<bdi dir="ltr">) through x-ui.money, x-ui.date and the @number helper, with Western digits.
- RTL: logical utilities only (ms- me- ps- pe- start- end- text-start text-end). Physical left/right utilities fail BladeConventionsTest.
- Build every screen from x-ui.* components (D09 9.15). Module composites are x-{module}.* under the module's own components folder.
- Status is never colour alone: pills carry text and an icon; attendance cells always show the code.
- One primary button per page. Confirm levels 1 to 3 for protected and bulk actions. Every screen has empty, loading and error states. Guardian, student and teacher screens work at 360px. Touch targets 44x44 below 768px.
- Menus come only from NavigationRegistry; nothing is hardcoded in layouts.

=====================================================================
15. LOCALIZATION
=====================================================================
- Every user-facing string uses a translation key in lang/en and lang/ur (module strings in the module's lang folder). TranslationParityTest fails if an en key has no ur key.
- Enum labels, error messages, notification templates and PDFs are translated. Organizations can override terms through translation_overrides.
- <html lang dir> set by SetLocale. Dates like 24 Nov 2026. Money like Rs 8,500.

=====================================================================
16. TESTING RULE
=====================================================================
- Pest on MySQL 8.4 with RefreshDatabase and parallel runs. No SQLite. No real external calls: bind Fake adapters.
- Every change ships with tests. Minimum per feature: happy path; validation failure (422); permission denied (403); other tenant gets 404; audit entry asserted.
- Money changes: journal balanced (assertJournalBalanced), idempotent on retry, reversal path. Workflow changes: every allowed transition and at least one forbidden one per state.
- Shared helpers in tests/Support: TwoTenantFixture, inTenant(), actingAsRole(), FakeAdapters, assertAudited(), assertNotVisibleToOtherTenant(), assertJournalBalanced(), assertQueryCountAtMost().
- Isolation tests are generated for every tenant model and route. Architecture tests guard every rule in sections 4 to 15.
- Coverage: 95% lines on money, results, identity, tenancy and approval code; 80% elsewhere; 85% overall; a merge that drops coverage by more than 1 point fails.
- Freeze time with Carbon::setTestNow(); organizations default to Asia/Karachi.

=====================================================================
17. CODING STANDARDS
=====================================================================
- declare(strict_types=1); in every PHP file. Pint with the project config. Larastan level 6, rising to 8 by PH-06.
- Actions, Services and DTOs are final. DTOs are readonly classes. Enums are backed strings.
- Names come from the domain (recordPayment, not handleData). No abbreviations except IDs from the blueprint.
- No N+1: eager load on lists; Model::shouldBeStrict() outside production. Every list paginates on the server. No unbounded ->get() on tenant tables in controllers.
- Domain exceptions extend Core DomainRuleViolation with a translated message and an error code.
- No dd(), dump(), ray() or var_dump() committed. No commented-out code.

=====================================================================
18. ENVIRONMENTS
=====================================================================
Local *.darsgah.test via deploy/docker/compose.yaml (app, nginx, mysql, redis-cache, redis-queue, gotenberg, mailpit, optional clamav). Staging *.staging.darsgah.app. Demo *.demo.darsgah.app (from PH-18). Production *.darsgah.app. Hosts per environment: platform., adms., hooks., api. DemoSeeder refuses to run in production.

=====================================================================
19. DEFINITION OF DONE (every step)
=====================================================================
- All acceptance criteria in the step prompt are met.
- composer check passes locally (pint --test, larastan, pest --parallel). npm run build passes with bundle budgets.
- New tenant models and routes are covered by the generated isolation tests. New permissions are in Definitions/Permissions.php, synced, and covered by the RBAC matrix fixture.
- New screens match their D10 screen ID, route name and permission, and are listed in ScreenCoverageTest (from PH-01).
- English and Urdu strings complete.
- docs/decisions/decisions.md updated for every DEC the step applies for the first time.
- Report written (section 20).

=====================================================================
20. REPORT FORMAT (docs/agent/reports/{step-id}.md)
=====================================================================
# {step id}: {title}
Summary: 3 to 6 sentences on what now works.
Files: grouped list of created and changed files.
Database: migrations added (table names, batch).
Permissions and routes: keys and route names added.
Tests: commands run and results (counts, coverage for touched areas).
Acceptance criteria: each criterion with PASS and the evidence (test name, command output, screenshot path).
Conflicts and decisions: any prompt or blueprint conflict and how it was resolved, any assumption made.
Stopped items: anything not done because of a stop rule, with the reason.
Next step notes: facts the next step needs.

=====================================================================
21. NEVER
=====================================================================
- Never weaken tenancy, authorization, audit, money or security rules to make something work.
- Never delete or rewrite historical business records.
- Never trust client-side data for payment status, prices, permissions, organization or branch.
- Never add a package that fails the package rule.
- Never hardcode business values (fees, grades, roles, approval chains, tax rates, thresholds, holidays). They are data or settings.
- Never put business logic in controllers, models, views, routes or app/.
- Never claim a step is complete without the evidence in your report.
END OF MASTER CONTEXT
```

---

## 14.4 PH-00 Bootstrap and Spikes

Phase goal (D13): a working skeleton with CI, the design system base, security middleware, a staging deploy, and proof that Urdu PDFs render correctly. Steps: P00.1 to P00.5. RT-1 (PHP 8.4 runtime in local, CI and staging) is met inside this phase.

---

### P00.1 Repository, Docker and CI

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. The repository is empty except docs/blueprint/ (D01 to D15) and docs/agent/phase-0-master-context.md. Read D08 8.2, 8.7, 8.12, 8.13, 8.14 and D12 Part 2 sections 12.26 and 12.27 before starting.

PHASE GOAL
Create the Laravel 12 application, the local Docker environment on PHP 8.4, code quality tooling, and the CI pipeline stages for push and pull request, so every later step has a green, reproducible baseline.

ALREADY BUILT
Nothing except the documentation folders.

TASKS
1. Create the Laravel 12 project in the repository root (composer create-project laravel/laravel:^12.0, keeping the existing docs/ folder). Lock laravel/framework to the latest 12.x release (12.71.0 or newer).
2. Shape composer.json as D08 8.2: name darsgah/app, "php": "^8.2", "laravel/framework": "^12.0", PSR-4 roots App\ => app/, Modules\ => modules/, Database\Seeders\ => database/seeders/, Tests\ => tests/ (dev), scripts lint, analyse, test, test:arch, check; config platform-check true, sort-packages true.
3. Remove the skeleton's users table migration, app/Models/User.php and its factory. Keep only the framework migrations for cache, cache_locks, jobs, job_batches and failed_jobs in database/migrations. (users, sessions and password reset tokens belong to Identity in P01.3.)
4. Install dev tools: pestphp/pest ^3, pestphp/pest-plugin-laravel ^3, pestphp/pest-plugin-arch ^3, larastan/larastan ^3, laravel/pint, laravel/pail. Convert the skeleton tests to Pest. Create tests/Pest.php and tests/TestCase.php using RefreshDatabase on MySQL, and the empty folders tests/Support, tests/Architecture, tests/Feature, tests/Unit, tests/Contract.
5. Add phpstan.neon (Larastan, level 6, phpVersion 80200, paths app, modules, config, database, routes), phpunit.xml (MySQL test database darsgah_test, failOnDeprecation, failOnWarning, failOnRisky), pint.json (Laravel preset plus declare_strict_types), .editorconfig, .gitattributes.
6. Create deploy/docker/compose.yaml with services: app (PHP-FPM 8.4 from deploy/docker/php/Dockerfile with extensions pdo_mysql, redis, intl, gd, bcmath, zip, pcntl, opcache, exif), nginx (deploy/docker/nginx/default.conf serving *.darsgah.test, platform.darsgah.test, hooks.darsgah.test, api.darsgah.test; adms.conf serving HTTP for adms.darsgah.test /iclock/* only), mysql 8.4 (databases darsgah and darsgah_test, utf8mb4_0900_ai_ci), redis-cache (redis 7, no persistence, maxmemory-policy allkeys-lru), redis-queue (redis 7, appendonly yes, appendfsync everysec, maxmemory-policy noeviction), gotenberg (gotenberg/gotenberg:8 for now; P00.4 replaces it with a font image), mailpit, clamav (profile "scan", optional). Limit app and gotenberg to 4 CPUs and 8 GB so local timings resemble the S1 size (D12 12.21).
7. Add deploy/docker/php/Dockerfile.ci for PHP 8.2 with the same extensions (used by the CI matrix).
8. Write .env.example exactly as D08 8.7 plus REDIS_CACHE_HOST=redis-cache, REDIS_QUEUE_HOST=redis-queue, SECURITY_INTERNAL_CIDRS=10.0.0.0/8,172.16.0.0/12,192.168.0.0/16. Local .env uses SESSION_DRIVER=file until P01.3 creates the sessions table; .env.example keeps database.
9. Configure config/database.php: MySQL charset utf8mb4, collation utf8mb4_0900_ai_ci, strict true, timezone +00:00; Redis connections default, queue, locks on REDIS_QUEUE_HOST and cache on REDIS_CACHE_HOST. config/app.php timezone UTC.
10. Create config/darsgah.php: root_domain, hosts (platform, adms, hooks, api), reserved_slugs (D04 4.7 list), product_name, support_email; all from env with safe defaults.
11. Create the documentation folders: docs/decisions/decisions.md (table header: ID | Decision | Deliverable | First applied in step), docs/runbooks/, docs/agent/phases/, docs/agent/reports/, docs/agent/checklists/. Add README.md with local setup (hosts file entries, docker compose up, composer install, npm install, php artisan key:generate, migrate, test).
12. Create .github/workflows/ci.yml with two stages from D12 12.27:
    push: composer validate; pint --test; larastan; pest --group=arch; pest tests/Unit; gitleaks; composer audit; npm ci; npm audit --audit-level=high; npm run build.
    pull_request: everything in push, plus pest --parallel on a matrix of PHP 8.2 (Dockerfile.ci image or setup-php) and PHP 8.4 with a MySQL 8.4 service and a Redis 7 service; coverage report uploaded as an artifact.
    Cache Composer and npm. Fail on any error.
13. Add .github/dependabot.yml (weekly composer, npm, github-actions, docker).

DATABASE
Only the Laravel framework tables: cache, cache_locks, jobs, job_batches, failed_jobs (unchanged from the skeleton). No business tables in this step.

BUSINESS RULES
None in this step.

UI SPEC
No screens. The default welcome route is removed; routes/web.php redirects / to /login (the login route arrives in P01.3, so until then / may return 404; do not create a placeholder page).

SECURITY AND PERMISSIONS
- .env is git-ignored; .env.example holds names only.
- gitleaks runs on every push. APP_DEBUG is false in .env.example for non-local guidance comments.
- No credentials in compose.yaml beyond local development defaults clearly marked local only.

TESTS
- tests/Feature/Core/HealthLivenessTest.php: GET /up returns 200.
- tests/Architecture/ForbiddenCodeTest.php (first version): no dd, dump, ray, var_dump in app, modules, config, routes; no env() outside config/.
- CI runs the suite on PHP 8.2 and 8.4.

ACCEPTANCE CRITERIA
1. docker compose -f deploy/docker/compose.yaml up -d starts every service; php -v inside app shows 8.4.x.
2. php artisan migrate creates only the five framework tables.
3. composer check passes locally.
4. The pull request CI run is green on PHP 8.2 and PHP 8.4.
5. composer prohibits laravel/framework 13.0 and composer prohibits php 8.4 report nothing.
6. gitleaks finds nothing.
7. README steps work on a clean machine.

DO NOT
- Do not create any module, business table, route or screen.
- Do not install Fortify, Sanctum, Horizon or Gotenberg packages yet.
- Do not use SQLite anywhere, including CI.
- Do not commit .env or any secret.

OUTPUT
Report per Master section 20 at docs/agent/reports/P00.1.md, including the CI run link and php -v output from the container.
```

---

### P00.2 Core Skeleton, Module Manifest and Architecture Guards

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D03 3.4 to 3.7, 3.21; D04 4.6; D07 Part 1 7.1 to 7.3, 7.6 (plans, plan_modules, platform_announcements) and 7.9 (platform_settings); D08 8.1, 8.3 to 8.8, 8.14 to 8.16; D02 2.3 with the D03 3.3 corrections.

PHASE GOAL
Create the module system (manifest, base provider, route loader, route macros), the 34 module provider skeletons in manifest order, the Core building blocks that need no tenant tables yet (Blueprint macros, Money, exceptions and renderer, registries contracts), the M00 platform catalog tables with seeders, and every architecture test, so later steps cannot break the structure without CI failing.

ALREADY BUILT
P00.1: Laravel 12 app, Docker on PHP 8.4, Pest 3, Larastan, Pint, CI push and PR stages, framework tables.

TASKS
1. config/modules.php (DEC-127): one entry per module in section 5 order with keys folder, code, tier, release, provider, depends_on. Fill depends_on from D02 2.3 as corrected by D03 3.3 (Guardian depends on Student, not the reverse; Student does not depend on Guardian; Reporting depends on every module; every module except Core depends on core). Use the Fee and ExternalExam entries from D08 8.5 exactly. Where D02 leaves a dependency implicit, include organization, identity, settings and audit for any module that owns tenant tables or permissions.
2. Modules\Core\Support\ModuleServiceProvider exactly as D08 8.6 (moduleKey, routeFiles(), listeners(), morphMap(), consoleCommands(), contributions(); loads Database/Migrations, views and translations under the module key, anonymous component path without prefix, explicit listeners, morph map, commands, routes via ModuleRouteLoader).
3. Modules\Core\Support\ModuleRouteLoader implementing the mounting table in D08 8.6 (file to host, middleware group, URL prefix with module kebab segment, name prefix, module:{key} middleware). Define the middleware groups from D08 8.4 in bootstrap/app.php using only middleware classes that already exist (framework classes now); each later step adds its own middleware class to the groups exactly as D08 8.4 lists them. A group must never reference a class that does not exist yet.
4. Modules\Core\Routing\RouteMacros::portalResource() (DEC-128): kebab URL from the resource segment, snake_case route name segment, parameter name in snake_case, binding by uuid.
5. Create the 34 module folders, each with Providers/{Name}ServiceProvider.php extending ModuleServiceProvider, protected string $moduleKey set, routeFiles() returning [] for now. Register App providers then the 34 providers in bootstrap/providers.php in manifest order.
6. CoreServiceProvider additionally: Relation::requireMorphMap() after modules register (use booted callback), Gate::guessPolicyNamesUsing (Modules\X\Models\Y -> Modules\X\Policies\YPolicy), Factory::guessFactoryNamesUsing (-> Modules\X\Database\Factories\YFactory), Model::shouldBeStrict() outside production, rate limiters auth, public, webhooks, adms, api, exports (D08 8.15), Blueprint macros, route macros.
7. Modules\Core\Database\Schema\BlueprintMacros (DEC-89, D07 7.2): tenantColumns() [T], tenantChildColumns() [T-], branchColumns() [B] (adds branch_id, composite FK to branches and ix_{alias}_org_branch), auditColumns() (created_by, updated_by composite FKs to users), softDeletesWithUser(), publicUlid() (uuid CHAR(26) with uq_{alias}_uuid), money($name) DECIMAL(15,2), tenantForeign($column, $table, $alias) building the composite FK with an explicit name, activeMarker(), keyColumn($source) generated IFNULL(col,0) STORED, datetimes() created_at/updated_at DATETIME. Each macro takes the table alias for index names. FKs that point at tables from a later batch (organizations, users, branches) must be created by the later batch: the macros accept a flag to skip FKs, and an addTenantForeigns helper exists for the later migration.
8. Modules\Core\Money\Money and Currency: integer minor units, PKR default, add, subtract, multiply by integer or decimal string with explicit rounding (half up), allocate by ratios without losing paisa, compare, isZero, isNegative, fromDecimalString, toDecimalString, format via the locale helper. Modules\Core\Casts\MoneyCast to DECIMAL(15,2).
9. Core exceptions (D08 8.4, 8.8): DomainRuleViolation (errorCode(), translation key, context), MissingTenantContext, ModuleNotEnabled, PeriodLocked, EditConflict, IdempotencyKeyMismatch; Modules\Core\Exceptions\ExceptionRenderer registering the full mapping table with JSON envelope { message, errors, code, correlation_id } and translated messages. Register it in bootstrap/app.php.
10. Core contracts (no implementations beyond what is listed): EntitlementChecker, VirusScanner (with NullScanner), Navigation\NavigationItem and NavigationRegistry with an in-memory implementation (NavigationRegistryImpl: add items, filter by portal, entitlement callback and permission callback), Reporting\ReportDefinition, MetricDefinition, DashboardWidget and their registry interfaces.
11. M00 migrations with real timestamps: Platform owns plans, plan_modules, platform_announcements (created_by column present, FK to users added in M02 by Identity); Settings owns platform_settings. Columns and keys exactly as D07 Part 1 7.6 and 7.9.
12. Seeders: Modules\Platform\Database\Seeders\PlanSeeder (DEC-233: essential = core tier modules; professional = core and standard; enterprise = all modules; limits NULL; display_price NULL; currency PKR; is_public 1) upserting by code; database/seeders/DatabaseSeeder calling the platform catalog seeders.
13. Architecture tests in tests/Architecture (D08 8.14): ModuleBoundariesTest (MB-01 to MB-09 as far as checkable: no cross-module imports outside Contracts, Events, Enums, Data, Models; controllers do not use DB facade; Actions final), DependencyMapTest (imports respect depends_on), ModuleLayoutTest (only allowed folders; components only under the module's own subfolder), ProvidersManifestTest (bootstrap/providers.php equals manifest order), TenancyTraitsTest (any model whose table has organization_id uses BelongsToOrganization; runs once tenant models exist, passes vacuously now), NamingTest (Actions final; enums backed with snake_case values; events past tense by suffix list; policies match models), ForbiddenCodeTest (extend: no #[\Override]; no 'exists:' validation strings in modules; no DB::table outside Queries; no env() outside config), MigrationOrderTest (parse migrations in timestamp order; fail if a migration references a table created later), RouteConventionsTest (URL segments kebab-case; names snake_case; every route named; every app., guardian., student., platform., api.v1. route has an authorization check; allowlist for baseline routes in tests/Fixtures/baseline_routes.php).
14. docs/decisions/decisions.md: add rows for DEC-01, DEC-35, DEC-89 to DEC-93, DEC-127 to DEC-132, DEC-226, DEC-232, DEC-233.

DATABASE
M00 (D07 Part 1):
- plans [P] alias pln: code VARCHAR(30) UNIQUE, name VARCHAR(100), description TEXT NULL, display_price DECIMAL(15,2) NULL, currency CHAR(3), max_active_students INT UNSIGNED NULL, max_branches SMALLINT UNSIGNED NULL, storage_gb INT UNSIGNED NULL, sms_credits_monthly INT UNSIGNED NULL, is_public TINYINT(1), is_active TINYINT(1), created_at, updated_at DATETIME.
- plan_modules [P] alias plm: plan_id FK=>plans, module_key VARCHAR(40); uq_plm (plan_id, module_key).
- platform_announcements [P] alias pan: title VARCHAR(200), body TEXT, audience VARCHAR(32) (all, plan), plan_id FK=>plans NULL, starts_at, ends_at, published_at DATETIME NULL, created_by BIGINT UNSIGNED (FK to users added in M02).
- platform_settings [P] alias pst: setting_key VARCHAR(120) UNIQUE, value JSON.
Framework tables from P00.1 stay unchanged.

BUSINESS RULES
- Module metadata comes only from config/modules.php. Nothing else may list modules.
- Money never uses floats. Allocation of 100.00 into three equal parts gives 33.34, 33.33, 33.33 (remainder to the first parts).

UI SPEC
No screens.

SECURITY AND PERMISSIONS
- Rate limiters defined with the limits in D08 8.15.
- ExceptionRenderer never includes record data in error output; 500 responses show only the correlation id.

TESTS
- Unit tests for Money (arithmetic, rounding, allocation, decimal string round trip, negative values, currency mismatch exception).
- Unit tests for portalResource route naming (cashier-sessions -> app.fee.cashier_sessions.index with parameter cashier_session) using a test module route file under tests/Fixtures.
- Feature test: each exception type maps to the HTTP status and code in D08 8.4 for web and JSON requests.
- Feature test: PlanSeeder is idempotent (running twice gives 3 plans) and plan modules match tiers from config/modules.php.
- All architecture tests above pass.

ACCEPTANCE CRITERIA
1. php artisan about lists 2 app providers plus 34 module providers in manifest order.
2. ProvidersManifestTest, DependencyMapTest, ModuleLayoutTest, MigrationOrderTest, ForbiddenCodeTest, NamingTest and RouteConventionsTest pass.
3. php artisan migrate:fresh --seed creates the framework and M00 tables and seeds 3 plans.
4. Money tests cover allocation without losing paisa.
5. CI green on PHP 8.2 and 8.4.

DO NOT
- Do not create TenantContext, scopes or any tenant table (P01.1).
- Do not add awarding_bodies, ibcc_conversion_rules or payroll_tax_slabs (DEC-232).
- Do not install a module-management package (DEC-35).
- Do not enable Laravel event auto-discovery (withEvents(discover: false)).

OUTPUT
Report per Master section 20, including the php artisan about provider list and the list of architecture tests with what each checks.
```

---

### P00.3 Design System Foundation

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D09 completely (especially 9.3, 9.4, 9.6, 9.7, 9.11, 9.12, 9.13, 9.14, 9.15, 9.17, 9.22) and D08 8.9. D03 3.17 for bundles.

PHASE GOAL
Install Tailwind v4 with the D09 tokens, the Alpine CSP build, Urdu fonts, the icon set, the seven layouts and the first set of x-ui components, with tests that stop token, contrast and RTL regressions.

ALREADY BUILT
P00.1 tooling and CI. P00.2 module system, Core building blocks, NavigationRegistry (in-memory), exception renderer, M00 tables.

TASKS
1. npm packages: tailwindcss v4, @tailwindcss/vite, alpinejs (CSP build @alpinejs/csp), @alpinejs/focus, tom-select, chart.js v4, laravel-vite-plugin, vite. Vite inputs: resources/css/app.css, resources/css/print.css, resources/css/pdf.css, resources/js/app.js, resources/js/portal.js (D08 8.9).
2. resources/css/tokens.css: copy D09 9.14 exactly (palette reset, primitives, light role tokens, typography, shape, layout, Urdu face switch, :lang(ur) sizing, dark theme block, reduced motion). resources/css/app.css imports tailwindcss and tokens.css with the @source lines from D08 8.9. print.css per D09 9.19 (hide navigation and buttons, full width tables, repeat headers). pdf.css minimal base (page margins 12mm, fonts Noto Sans and Noto Nastaliq Urdu by name).
3. Fonts: add NotoNastaliqUrdu-Regular.woff2 and NotoNaskhArabic-Regular.woff2 (OFL, from the official Noto fonts repository) to resources/fonts with OFL.txt; @font-face rules from D09 9.6 with unicode-range; Vite copies them to /build/fonts.
4. Icons: copy the Lucide subset needed by D09 9.11 (navigation icons listed there plus check-circle, x-circle, alert-triangle, clock, lock, file-pen, circle-slash, info, plus, chevron-left, chevron-right, chevron-down, search, filter, x, menu, bell, log-out, eye, eye-off, download, upload, printer, more-vertical, settings) into resources/svg/icons/{name}.svg with the ISC licence file. x-ui.icon inlines the SVG with currentColor, aria-hidden by default, size class passthrough, and mirrors directional icons in RTL (rtl:-scale-x-100) from a list in config/ui.php.
5. JS: resources/js/bootstrap/alpine.js (start Alpine CSP build with focus plugin), resources/js/bootstrap/http.js (fetch wrapper adding X-CSRF-TOKEN, X-Correlation-Id, Idempotency-Key when given, parsing the error envelope), resources/js/components/index.js registering shared Alpine.data components needed by this step: modal, dropdown, toast, confirmDialog, passwordReveal. app.js and portal.js per D08 8.9 including import.meta.glob of module js folders. All Alpine logic lives in registered components (CSP build: no inline expressions beyond property access and method calls).
6. AppServiceProvider: Vite::useCspNonce() with the nonce from the request attribute set by SecurityHeaders (P00.5 adds the middleware; until then generate the nonce in a small Core service Modules\Core\Http\CspNonce that P00.5 will use).
7. Layouts in resources/views/layouts: app (staff: topbar 56px with scope line slot, sidebar 264px from partials/navigation rendered from NavigationRegistry, content area, toasts region with aria-live), guardian and student (topbar, child switcher slot for guardian, bottom navigation 64px with max 5 items from NavigationRegistry), platform (platform-bar #16191C with "Platform" badge, DEC-150), public, print, auth (sign-in layout from D09 9.17 with accent band). Every layout sets <html lang dir data-urdu-font> from the current locale and user preference, includes Vite assets with the nonce, a skip link, and a banner slot (impersonation, all-branches, period-locked, offline).
8. Error views 403, 404, 409, 419, 423, 429, 500, 503 using the public layout, translated, showing the correlation id where available, no record data.
9. x-ui components (anonymous, resources/views/components/ui) with props, variants and states exactly as D09 9.15: button (primary, secondary, ghost, danger, link; sm, md, lg; icon, icon-end, loading, disabled with reason tooltip), input, label, field (label, help, error, required, aria wiring), card, table (static), modal, confirm-dialog (levels 1 to 3 with reason textarea and typed confirmation), toast, page-header (title, subtitle, breadcrumbs, actions slot), breadcrumb, empty-state (empty, filtered, scope variants), alert, banner, badge, status-pill (role mapping from D09 9.5 via a Core StatusTone resolver), skeleton, loading, icon, money (display, LTR isolated, tabular), date (organization timezone, D09 9.21 formats, LTR isolated).
10. Blade helpers: @number directive and a Core Formatting service for money (Rs 8,500; paisa only when non-zero; negative -Rs 500; grouping international by default) and dates (24 Nov 2026; Urdu month names with Western digits).
11. Local-only UI gallery route GET /_dev/ui named dev.ui_gallery, registered only when app()->environment('local'), rendering every component in English and Urdu, light theme, at mobile and desktop widths. Excluded from RouteConventionsTest by name prefix dev.
12. Bundle budget script scripts/check-bundle-size.mjs run by npm run build:check: fails if gzip size exceeds portal.js 60 KB, app.js 120 KB, each CSS bundle 50 KB (D12 12.17). Add it to the CI push stage.
13. lang/en/ui.php and lang/ur/ui.php with every string used by layouts and components.

DATABASE
None.

BUSINESS RULES
- Role tokens only in views. Primitive steps appear only in tokens.css.
- Status mapping per D09 9.5 is the single source for pills and attendance codes.

UI SPEC
- Follow D09 9.15 component specs and 9.17 shell wireframes exactly (sizes, spacing, radius, focus ring 2px with 2px offset, touch targets).
- Button md is 40px high on desktop and 44px below 768px.
- Urdu text uses :lang(ur) sizing; no uppercase transforms anywhere.
- Directional icons mirror in RTL; clocks and checkmarks never mirror.

SECURITY AND PERMISSIONS
- No inline scripts or styles without the CSP nonce. No external CDNs for fonts, icons or scripts.
- The UI gallery route never registers outside local.

TESTS
- tests/Unit/Core/DesignTokensTest.php: parses resources/css/tokens.css; asserts every role token exists; computes WCAG contrast for every pair listed in D09 9.4 (light and dark) and the navigation pairs, and fails below the stated thresholds.
- tests/Architecture/BladeConventionsTest.php: no physical direction utilities (ml-, mr-, pl-, pr-, left-, right-, text-left, text-right, rounded-l-, rounded-r-, border-l, border-r) in any Blade file; no {!! outside the sanitizer helper allowlist; no HEX colour literals or style="color in Blade.
- tests/Feature/Core/UiComponentsTest.php: each component renders with required props; button disabled-with-reason renders the reason; status-pill renders text and icon for every mapped status; money and date render LTR-isolated with Western digits in the ur locale; layouts set lang and dir correctly for en and ur.
- TranslationParityTest (tests/Architecture): every key in lang/en exists in lang/ur, including module lang folders.

ACCEPTANCE CRITERIA
1. npm run build and npm run build:check pass; bundle sizes reported.
2. DesignTokensTest passes for every D09 pair.
3. BladeConventionsTest and TranslationParityTest pass.
4. The local UI gallery shows every component in English and Urdu at 360px and 1280px without layout breaks; screenshots saved under docs/agent/reports/assets/P00.3/.
5. No request to any external host when loading the gallery (fonts and icons are local).
6. CI green.

DO NOT
- Do not use Tailwind's default colour names or any HEX in views.
- Do not add a Latin web font.
- Do not create module screens or business routes.
- Do not use inline Alpine expressions that need eval.

OUTPUT
Report per Master section 20 with bundle sizes, contrast test summary and gallery screenshots.
```

---

### P00.4 Urdu PDF Spike

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D03 3.14, D09 9.19, D12 12.17 (PDF budgets) and D13 13.5 PH-00 exit criteria. This is the RISK-06 spike: it decides whether the Gotenberg approach stands.

PHASE GOAL
Prove that Gotenberg renders a 3-copy fee challan and a report card with correctly shaped Urdu Nastaliq text, that the PDF text layer contains the Urdu strings, and that single renders and a 1,000-voucher merged batch meet the time budgets.

ALREADY BUILT
P00.1 Docker and CI; P00.2 module skeletons (including Document), Core building blocks; P00.3 tokens, fonts, layouts, components, pdf.css.

TASKS
1. Install gotenberg/gotenberg-php ^2.0 and run the package compatibility check.
2. deploy/docker/gotenberg/Dockerfile: FROM gotenberg/gotenberg:8; add Noto Sans, Noto Nastaliq Urdu and Noto Naskh Arabic (OFL) into the system fonts folder and run fc-cache. Point the compose gotenberg service to this image. Gotenberg has no outbound network in compose (internal network only).
3. Modules\Document\Contracts\PdfRenderer: render(string $html, PdfOptions $options): string (PDF bytes); merge(array $pdfs): string. Modules\Document\Data\PdfOptions (paper preset: a4, a4_landscape, legal, a5, cr80_sheet; margins; print background; header and footer HTML optional).
4. Modules\Document\Adapters\Pdf\GotenbergPdfRenderer using the Chromium HTML route and the merge route, with timeouts from config/pdf.php (GOTENBERG_URL, connect and request timeouts, paper presets from D08 8.7). Assets (CSS, fonts, images) are inlined into the HTML; the renderer never passes URLs for Gotenberg to fetch. Modules\Document\Adapters\Pdf\FakePdfRenderer returning fixed bytes and recording calls. Bind PdfRenderer in DocumentServiceProvider (Gotenberg outside tests, Fake in tests via FakeAdapters).
5. Spike templates (Blade, using pdf.css): document::pdf.spike.challan (A4 landscape, three copies side by side labelled Bank, School and Parent copy, dashed cut lines, header with institution name in English and Urdu, student Ayesha Khan / عائشہ خان, father Imran Khan / عمران خان, class 9-A, voucher number V-2026-11-000412, items Tuition fee Rs 8,500, Lab fee Rs 500, discount, net, due date 10 Nov 2026, amount in words in English and Urdu, consumer number and barcode placeholder rendered as text) and document::pdf.spike.report-card (A4 portrait, student identity in both languages, 8 subjects with marks and grades, total, percentage, attendance, Urdu remarks paragraph of at least 3 lines: "عائشہ نے اس سال بہت محنت کی۔ ریاضی اور سائنس میں اس کی کارکردگی نمایاں رہی۔ انگریزی تحریر میں مزید مشق کی ضرورت ہے۔"). Numbers are LTR-isolated and tabular.
6. Command document:spike-render (local and staging only; refuses in production) with options --type=challan|report-card|both, --count=N (default 1), --concurrency=C (default 4), --out=storage/app/tmp/spike. It renders N challans with varied names (Urdu and English lists in a fixture file), merges in chunks of 100 then merges the chunk files into one PDF, and prints: single render p50 and p95, total render time, merge time, total time, output size, peak memory.
7. Screenshot capture: use the Gotenberg Chromium screenshot route to save PNGs of both templates at 150 dpi equivalent into docs/agent/reports/assets/P00.4/ for the product owner's visual check of Nastaliq shaping.
8. Text layer check: add poppler-utils to the CI test image and a test that runs pdftotext on the rendered PDFs.
9. Write docs/decisions/spike-urdu-pdf.md: method, machine limits (4 CPU, 8 GB), timings table, screenshots, text extraction result, verdict against the criteria below, and the steps P00.5 must repeat on staging.

DATABASE
None.

BUSINESS RULES
- Pass criteria: (a) Nastaliq ligatures and letter joining are correct in both PDFs (product owner visual sign-off on the PNGs); (b) pdftotext output contains the exact Urdu strings for the student name, father name and the remarks sentences; (c) single render p95 under 2 seconds; (d) 1,000 challans rendered and merged in under 5 minutes at concurrency 4 within the 4 CPU / 8 GB limits.
- If any criterion fails, stop: record the failure in the spike report and do not modify D03 decisions yourself.

UI SPEC
PDF layout per D09 9.19: black text on white, header band in secondary colour, 12mm margins, tabular figures, amounts in words, no status colour carrying meaning.

SECURITY AND PERMISSIONS
- Gotenberg is reachable only on the internal Docker network; the renderer never forwards user-supplied URLs.
- The spike command refuses to run when APP_ENV=production.

TESTS
- tests/Unit/Document/FakePdfRendererTest.php: records calls and returns bytes.
- tests/Feature/Document/Rendering/SpikeRenderingTest.php in Pest group gotenberg (runs against the real Gotenberg container, excluded from the default PR run, included in the nightly job created in P00.5): renders both templates, asserts the PDF is valid, page count (challan 1, report card 1), and pdftotext contains the Urdu strings exactly.
- tests/Unit/Document/GotenbergPdfRendererTest.php: builds the multipart request correctly (HTML inlined, paper preset, no remote URLs) using an HTTP fake.

ACCEPTANCE CRITERIA
1. Both spike PDFs and PNG screenshots exist in docs/agent/reports/assets/P00.4/.
2. pdftotext test passes.
3. Timings table shows single render p95 under 2 s and 1,000 merged under 5 min, or the report states FAIL with numbers.
4. docs/decisions/spike-urdu-pdf.md complete with a verdict line: PASS or FAIL.
5. Package compatibility check clean.
6. CI green (default suite; gotenberg group runs locally with evidence).

DO NOT
- Do not use dompdf, mPDF, wkhtmltopdf or Browsershot.
- Do not let Gotenberg fetch remote assets.
- Do not build real voucher or report card features; these are spike templates kept only as rendering fixtures until P06.3 and P07.2 replace them.

OUTPUT
Report per Master section 20 with the timings table, the verdict, and the screenshot paths. Ask the product owner to sign off the visual check in the pull request.
```

---

### P00.5 Security Base, Operations Base and Staging Deploy

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D03 3.7, 3.19; D08 8.4, 8.7 (logging, security, horizon, queue), 8.12; D12 Part 1 12.7, 12.9, 12.10 and Part 2 12.26 to 12.29, 12.33; DEC-231 and DEC-234 in D14.

PHASE GOAL
Add the request security middleware, structured and redacted logging, health endpoints, Horizon with the D03 queues, and a production-style container image that CI builds, scans, signs and deploys to staging, where the PDF spike timings are repeated.

ALREADY BUILT
P00.1 to P00.4: app skeleton, CI push and PR stages, module system, design system base, PdfRenderer with Gotenberg and the spike.

TASKS
1. Modules\Core\Http\Middleware\AssignCorrelationId: accepts a valid incoming X-Correlation-Id from internal callers only, otherwise generates a ULID; sets the response header; adds it to the log context.
2. Modules\Core\Http\Middleware\SecurityHeaders using CspNonce: Content-Security-Policy default-src 'self'; script-src 'self' 'nonce-{n}'; style-src 'self' 'nonce-{n}'; img-src 'self' data: blob:; connect-src 'self'; font-src 'self'; frame-ancestors 'none'; form-action 'self'; base-uri 'none'; object-src 'none'; report-uri to a CSP report route core.csp_reports.store (rate limited, stores counts only). Plus Strict-Transport-Security max-age=31536000; includeSubDomains (preload only on the production root domain via config), X-Content-Type-Options nosniff, Referrer-Policy strict-origin-when-cross-origin, Permissions-Policy camera=(self), geolocation=(self), microphone=(). Directives come from config/security.php so later steps can add payment hosts on the payment page only.
3. Modules\Core\Http\Middleware\EnsureInternalNetwork (DEC-231) using config('security.internal_cidrs').
4. Health: keep Laravel /up. Modules\Core\Http\Controllers\HealthController for GET /internal/health/ready named internal.health.ready behind EnsureInternalNetwork: checks MySQL (select 1), redis-cache and redis-queue (ping), a queue round trip on the default queue (dispatch a tiny job that writes a cache key; wait up to 5 s), Gotenberg (health route), private disk write and delete. Returns JSON with each check, status 200 or 503. Add it to RouteConventionsTest's internal allowlist.
5. Logging (config/logging.php): JSON formatter on the app, security and integrations channels (D03 3.19); processors adding correlation_id, organization_id, branch_id, user_id when known; Modules\Core\Logging\RedactionProcessor removing keys and patterns: password, password_confirmation, current_password, otp, code, token, secret, api_key, authorization, cookie, set-cookie, cnic, b_form, card numbers (13 to 19 digit runs passing Luhn), IBAN, bank account fields; request bodies are never logged.
6. Install laravel/horizon ^5 (package check). config/horizon.php with one supervisor per queue from D03 3.11 (workers, timeout, tries, backoff). Horizon dashboard gate: platform super admin only; until platform users exist (P01.5), deny in every non-local environment. config/queue.php Redis on the queue connection with after_commit true.
7. Optional Sentry: install sentry/sentry-laravel only if the package check passes; DSN from env; send_default_pii false; before_send scrubs with the same redaction rules; traces_sample_rate 0.1.
8. Production image: deploy/docker/php/Dockerfile multi-stage: composer install --no-dev --optimize-autoloader in a builder; npm ci && npm run build in a node builder; final PHP-FPM 8.4 image with OPcache (validate_timestamps=0, memory 256M, JIT off), realpath cache 4096K, non-root user, no dev tools. Entrypoints by role: web (php-fpm with nginx sidecar), worker (php artisan horizon), scheduler (php artisan schedule:work). Startup runs php artisan optimize for the web and worker roles.
9. Staging stack: deploy/staging/compose.yaml (caddy, nginx, app web, worker, scheduler, mysql 8.4, redis-cache, redis-queue, gotenberg font image) and deploy/caddy/Caddyfile.staging serving *.staging.darsgah.app, platform.staging.darsgah.app, hooks., api. with TLS via DNS challenge (token from the server secret store; never in the repo), and adms.staging.darsgah.app with HTTP allowed only for /iclock/*. deploy/scripts/deploy.sh implementing D12 12.29 steps 1 to 6 for staging (pre-flight checks, migrate with the migration DB user, roll web, horizon:terminate, replace scheduler, smoke tests). deploy/scripts/smoke.sh: /up 200, internal.health.ready 200 from inside the network, a queue round trip, one Gotenberg render.
10. Database users on staging per DEC-190: app user (DML only; INSERT and SELECT only on audit_logs and login_events once they exist), migration user (DDL), read-only user (replica later). Provide deploy/mysql/grants.sql parameterised by environment variables; passwords come from the secret store.
11. CI: add the merge-to-main stage (build image tagged with the commit SHA, scan with Trivy and fail on fixable critical or high, sign with cosign keyless, push to the registry, deploy to staging over SSH using secrets STAGING_HOST, STAGING_SSH_KEY, REGISTRY_TOKEN, run smoke.sh, then run an OWASP ZAP baseline scan against the staging root and fail on high findings) and the nightly stage (pest --group=gotenberg against a Gotenberg service container). If any secret is missing, the job must fail with a clear message; do not invent values.
12. Repeat the P00.4 spike timings on staging with document:spike-render --type=challan --count=1000 --concurrency=4 and append the results to docs/decisions/spike-urdu-pdf.md.
13. Runbooks: docs/runbooks/deploy.md and docs/runbooks/rollback.md (redeploy previous tag) for staging.
14. decisions.md rows: DEC-30, DEC-38, DEC-186, DEC-190, DEC-197 (partial: logging), DEC-198, DEC-206, DEC-231, DEC-234.

DATABASE
None (grants file only).

BUSINESS RULES
- Production never runs PHP 8.2 (DEC-198). Staging reports PHP 8.4.x on /internal/health/ready (include php_version in the JSON).
- No deploy outside the D12 window is enforced only for production; staging deploys on every merge.

UI SPEC
Error pages from P00.3 are used by the renderer for web requests.

SECURITY AND PERMISSIONS
- CSP and headers on every response from tenant, platform, hooks and api hosts.
- /internal/* answers 404 to any address outside internal_cidrs.
- Horizon dashboard closed outside local.
- Secrets only from the CI secret store and the server secret store.

TESTS
- tests/Feature/Core/SecurityHeadersTest.php: every header present with the configured values; CSP contains a nonce that matches the nonce on the Vite script tags of a rendered layout.
- tests/Feature/Core/CorrelationIdTest.php: generated when absent; echoed in responses and error JSON.
- tests/Feature/Core/InternalHealthTest.php: 404 from an external IP; 200 with all checks passing from an internal IP (Gotenberg check faked in the default suite); 503 when Redis is unavailable (simulate via a failing connection binding).
- tests/Unit/Core/RedactionProcessorTest.php: every listed key and pattern is removed; a Luhn-valid card number is masked; a 13-digit CNIC-like value under key cnic is removed.
- tests/Feature/Core/HorizonGateTest.php: dashboard returns 403 in non-local environments.

ACCEPTANCE CRITERIA
1. Merge to main builds, scans, signs and deploys to staging automatically; smoke.sh passes; ZAP baseline shows no high findings.
2. https://{any}.staging.darsgah.app/up returns 200 over TLS 1.2 or 1.3; security headers present (verified with curl -I output in the report).
3. internal.health.ready on staging returns 200 with php_version 8.4.x (RT-1 evidence).
4. Staging spike timings recorded; 1,000 challans merged under 5 minutes, or a FAIL recorded.
5. Nightly job runs the gotenberg group successfully.
6. All tests green; CI green.

DO NOT
- Do not put any secret, token, password or private key in the repository or in the conversation.
- Do not open /internal/* to the internet.
- Do not log request bodies.
- Do not set up production in this step (P09.4).

OUTPUT
Report per Master section 20 with the CI run links for merge and nightly stages, curl -I output from staging, the health JSON, and the staging spike timings. State RT-1 as met or not met with evidence.
```

---

## Assumptions (Part 1)

| ID | Assumption |
|---|---|
| A-109 | The repository is hosted on GitHub and GitHub Actions is the CI system; a container registry (GitHub Container Registry by default) is available. |
| A-110 | Antigravity can read files in the repository, run shell commands in the Docker environment and open pull requests, but works without receiving real secrets. |
| A-111 | The product owner signs off the Urdu PDF visual check in the P00.4 pull request within two working days. |
| A-112 | A staging VM with Docker and a DNS provider that supports API-based DNS challenges is provisioned by the owner before P00.5. |

## Gaps Found and Added (Part 1)

| ID | Gap | Decision |
|---|---|---|
| GAP-277 | Prompts had no reliable way to reach exact blueprint text | Blueprint copies in the repository (DEC-226) |
| GAP-278 | No rule for how a step maps to conversations, branches and commits | DEC-227 |
| GAP-279 | No standard evidence format for completed steps | Step reports (DEC-228) |
| GAP-280 | The agent had no rule for when to stop instead of guessing | Stop rules (DEC-229) |
| GAP-281 | Two different file storage paths in D3/D7 and D8 | Canonical path (DEC-230) |
| GAP-282 | Health and internal endpoints had conflicting paths and no network restriction | DEC-231 |
| GAP-283 | M00 included tables whose columns live in later deliverables and modules | DEC-232 |
| GAP-284 | No starter plans defined for the plan seeder | DEC-233 |
| GAP-285 | `.env.example` had one Redis host, against the two-role decision | DEC-234 |
| GAP-286 | The skeleton's default users and sessions tables would clash with Identity's tables | P00.1 task 3 |

---

Reply CONTINUE for Deliverable 14, Part 2
