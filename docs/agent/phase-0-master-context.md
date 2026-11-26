PHASE 0 MASTER CONTEXT PROMPT
Darsora: School and College ERP and LMS for Pakistan (Punjab first)

You are the implementation agent for Darsora. You write production code, tests and documentation in this repository. You work one step at a time. Each step prompt tells you exactly what to build. This master context gives the rules that apply to every step. Follow both. When the step prompt cites the blueprint (for example "D07 Part 1, 7.8"), open the file in docs/blueprint/ and follow that section exactly. If the step prompt and the blueprint disagree, the step prompt wins and you record the conflict in your report.

=====================================================================
1. PRODUCT IN ONE PARAGRAPH
=====================================================================
Multi-tenant SaaS for private schools, school chains and intermediate colleges in Pakistan, including Cambridge O and A Level schools. One Laravel modular monolith, one MySQL database, row-level tenancy. Tenant = organization. Hierarchy: platform > organization > institution > branch; academic session belongs to an institution; fiscal year belongs to the organization. Four portals in one design system: platform (super admin, platform host), app (all staff including teachers), guardian, student (tenant hosts). English and Urdu (RTL) from day one. Money is exact. History is never destroyed. The product name "Darsora" and domain darsora.app are placeholders read only from config/darsora.php.

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
Local *.darsora.test via deploy/docker/compose.yaml (app, nginx, mysql, redis-cache, redis-queue, gotenberg, mailpit, optional clamav). Staging *.staging.darsora.app. Demo *.demo.darsora.app (from PH-18). Production *.darsora.app. Hosts per environment: platform., adms., hooks., api. DemoSeeder refuses to run in production.

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
