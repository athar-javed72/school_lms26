# Deliverable 14: Sequential Antigravity Implementation Prompts
# Part 2 of 9: PH-01 Foundation (P01.1 to P01.7)

**Version:** 1.0 (draft for approval) | **Date:** 24 Nov 2026
**Builds on:** D14 Part 1 (Master Context Prompt, PH-00), D3, D4, D5, D7 Part 1, D8, D9, D10 Parts 1 and 2, D12 Part 1, D13 13.5
**How to run:** each step below is pasted after the Phase 0 Master Context Prompt in a new Antigravity conversation (DEC-227).

---

## 14.5 New Decisions

| ID | Decision |
|---|---|
| DEC-235 | **Platform role assignments live in `platform_user_roles`** (Identity, [P] table: `id`, `user_id` FK=>users, `role_id` FK=>roles, `assigned_by` FK=>users, `reason`, `created_at`, `revoked_at`; `uq_pur (user_id, role_id)`). `user_role_assignments` stays tenant-only with `organization_id NOT NULL`. |
| DEC-236 | **Cross-cutting contracts live in Core and are implemented higher up:** `AuditRecorder` (implemented by Audit), `PermissionChecker` (Identity), `TenantResolver` (Platform), `EntitlementChecker` (Platform). Core also holds the `Auditable` trait, the `BasePolicy` class with `allows()`, the `HasVisibilityScope` model trait (`visibleTo()`), and the `TenantExists` and `BranchExists` rules. Any module can then audit, authorize and scope through Core without depending on a higher module. This corrects D8 8.10, which placed `Auditable` in Audit. |
| DEC-237 | **Dependency correction:** Organization depends on Core only. Platform depends on Core, Organization, Identity and Settings, because provisioning creates organizations, owner users and default settings through those modules' contracts. Identity depends on Core and Organization. This corrects D2 2.3, which made Organization depend on Platform. |
| DEC-238 | **PH-01 step order adjusted for table dependencies:** P01.2 creates `users`, `sessions`, `audit_logs` and `login_events` together with the Core service tables. P01.3 builds RBAC. P01.4 builds authentication. This corrects the P01.2 to P01.4 split in D13 13.5. Scope, phase duration and exit criteria are unchanged. |
| DEC-239 | **OTP delivery contract:** Identity defines `OtpDelivery`. In PH-01 it is bound to email (SMTP, Mailpit locally) plus a log channel in local and staging. P02.2 rebinds it to SMS and email through the Notification engine on the `critical` queue. |
| DEC-240 | **First platform super admin:** the command `identity:create-platform-admin` asks interactively for name, email and mobile. It refuses if a super admin already exists, unless `--additional` is passed. It sets `must_change_password` and forces 2FA enrolment at the first sign-in. |
| DEC-241 | **Screen coverage fixture:** `tests/Fixtures/screens.php` lists every built screen: ID, route name, permission, portal and release. `ScreenCoverageTest` asserts that each listed route exists, carries the listed permission check, and renders for a user holding that permission. Each step appends its screens. P10.1 compares the fixture against the full D10 R1 list. |

### Corrections to earlier deliverables

| Where | Change |
|---|---|
| D2 2.3, `config/modules.php` | Dependencies follow DEC-237 |
| D8 8.10 | `Auditable` and `AuditRecorder` live in Core (DEC-236) |
| D7 Part 1 7.8 | Adds `platform_user_roles` (DEC-235) |
| D13 13.5 PH-01 | Step contents follow DEC-238 |

---

## 14.6 PH-01 Foundation

**Phase goal (D13):** a secure multi-tenant shell. The platform provisions an organization, users sign in, roles and permissions work, and every change is audited. **Exit:** MS-03 (UG-1) and MS-04 (first organization provisioned end to end on staging).

| Step | Scope |
|---|---|
| P01.1 | Tenancy core and M01 tables |
| P01.2 | Users, audit foundation and Core service tables |
| P01.3 | RBAC |
| P01.4 | Authentication and sessions |
| P01.5 | Platform and organization management |
| P01.6 | Settings, audit screens, localization, navigation and portal homes |
| P01.7 | UG-1: Laravel 13 |

---

### P01.1 Tenancy Core and M01 Tables

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D04 completely (4.3 to 4.9, 4.15 to 4.17, 4.24), D03 3.7 to 3.9, D07 Part 1 7.2, 7.6 (organization_subscriptions, organization_modules, organization_domains, usage_counters) and 7.7 (all Organization tables), D08 8.8 (Tenancy folder), and D14 Part 2 decisions DEC-236 and DEC-237.

PHASE GOAL
Make tenancy automatic and fail-closed: resolve the organization from the host, carry it through requests, jobs and tests, scope every tenant model, and create the organization structure tables with composite foreign keys, so no later code can read or write across tenants by accident.

ALREADY BUILT
PH-00: Laravel 12 app on PHP 8.4 in Docker and staging; CI push, PR, merge and nightly stages; module system with 34 provider skeletons and config/modules.php; Blueprint macros, Money, exceptions and renderer; registries contracts; design system base and layouts; PdfRenderer with Gotenberg; security headers, correlation id, redacted logs, health endpoints, Horizon.

TASKS
1. Update config/modules.php depends_on per DEC-237 (organization: core; platform: core, organization, identity, settings; identity: core, organization). Keep DependencyMapTest green.
2. Modules\Core\Tenancy\TenantContext (singleton) with the fields in D04 4.8: organizationId, organizationUuid, institutionIds, allowedBranchIds, activeBranchId (null = all branches read mode), userId, impersonatorId, portal, locale, timezone, correlationId. Methods: hasOrganization(), requireOrganization() (throws MissingTenantContext), isAllBranchesMode().
3. Modules\Core\Tenancy\Tenancy: actAs(Organization-like id or object, ?branchId), bypass(callable, string $reason) which sets a bypass flag for the callback only and records an audit entry through the AuditRecorder contract when bound (P01.2 binds it; until then bypass records to the security log), current(), forget().
4. Modules\Core\Tenancy\OrganizationScope (fail closed: throws MissingTenantContext when no context and no bypass), AllowedBranchesScope (limits to allowedBranchIds; organization-wide roles get all branches), BelongsToOrganization trait (adds scope; creating hook sets organization_id from context; organization_id guarded; updating hook forbids changing organization_id), BelongsToBranch trait (adds AllowedBranchesScope; creating hook sets branch_id from activeBranchId, rejects when null with DomainRuleViolation core.select_branch_first; explicit branch_id must be in allowedBranchIds; branch_id immutable after create), HasPublicUlid (uuid generated on creating, route key uuid).
5. Modules\Core\Contracts\Tenancy\TenantResolver (resolveByHost(string $host): ?ResolvedTenant with organization id, uuid, slug, status, timezone, locale). Platform implements it in Modules\Platform\Services\DomainTenantResolver using organization_domains and organizations, cached at host:{host}:v{n} in the cache store, ignoring archived organizations and unverified custom domains, honouring redirect_until for old slugs (301 to the primary domain).
6. Modules\Core\Http\Middleware\ResolveTenant: tenant hosts only (not platform, hooks, adms, api); reserved slugs and unknown hosts return the 404 page; suspended organizations get context with a read-only flag (writes later blocked by a Core ReadOnlyTenant guard returning 423 with core.tenant_suspended). Add it to the tenant.web group in bootstrap/app.php as D08 8.4 lists.
7. Modules\Core\Jobs\Middleware\TenantAware: captures organization id, branch id, user id, correlation id at dispatch; restores them before handle(); fails the job with MissingTenantContext if absent. A Core trait DispatchesWithTenant makes jobs capture context automatically.
8. Modules\Core\Rules\TenantExists and BranchExists (DEC-236): validate a ULID refers to a row of the given model visible in the current context (BranchExists also checks allowedBranchIds). Add ForbiddenCodeTest enforcement that the string 'exists:' never appears in modules (already present) and that Form Requests with *_id or *_uuid inputs use these rules (heuristic check by field name suffix).
9. Modules\Core\Support\TenantCache wrapper that prefixes org:{id}: and refuses to run without context.
10. M01 migrations (real timestamps, Blueprint macros, explicit names from D07): organizations, institutions, branches, branch_bank_accounts, departments, shifts, rooms (owned by Organization); organization_domains, organization_subscriptions, organization_modules, usage_counters (owned by Platform). Deferred foreign keys: created_by and updated_by on M01 tables point at users, which P01.2 creates; create the columns now without FKs and let P01.2 add them through a migration named add_user_foreigns_to_m01_tables. Also deferred: organizations.logo_file_id and institutions.logo_file_id (to stored_files, P01.2). Apply the D07 Part 3 corrections (DEC-108) now: do not create branch_bank_accounts.account_id, departments.head_employee_id or branches.principal_employee_id (the ledger mapping lives in Accounting, department heads in HR department_heads, and the principal is the holder of the principal role). organization_modules.granted_by and organization_subscriptions have no user FK until P01.2.
11. Models: Modules\Organization\Models\Organization (not tenant-scoped; it is the tenant), Institution, Branch, BranchBankAccount, Department, Shift, Room (BelongsToOrganization; Branch-owned models also BelongsToBranch per D07 tags [B]); Modules\Platform\Models\OrganizationDomain, OrganizationSubscription, OrganizationModule, UsageCounter (Platform per-tenant tables: no OrganizationScope; only Platform code reads them, D04 4.4). Morph aliases: organization, institution, branch, department, shift, room. Factories for each, with Organization::factory()->withInstitution()->withBranch().
12. Test support in tests/Support: InTenant (inTenant($org, fn) and inBranch($org, $branch, fn)), TwoTenantFixture (organizations A and B, each with one institution, two branches, departments, shifts, rooms, mirrored), Assertions\assertNotVisibleToOtherTenant().
13. Generated isolation suite tests/Feature/Core/Isolation/TenantIsolationTest.php: discovers every model using BelongsToOrganization through the module manifest and reflection; for each, creates rows in A and B and asserts: query in A never returns B rows; count, update and delete of B rows from A context affect nothing; query without context throws MissingTenantContext; organization_id in mass assignment is ignored. Also a database test: a raw insert of a branch-owned child with B's parent id inside A's organization fails on the composite FK.
14. decisions.md rows: DEC-03, DEC-27, DEC-43 to DEC-46, DEC-48, DEC-50, DEC-236, DEC-237.

DATABASE
All columns, types, keys and aliases exactly as D07 Part 1:
- organizations (alias org): uuid, slug UNIQUE, legal_name, display_name, display_name_ur, status (trial, active, suspended, archived), timezone default Asia/Karachi, currency default PKR, default_locale default en, fiscal_year_start_month default 7, ntn, address, city, phone, email, logo_file_id (FK later), trial_ends_at, suspended_at, archived_at, created_at, updated_at. Also unique (id) is the PK; tenant children reference organizations(id) through fk_{alias}_org.
- institutions [T][S] (ins): name, name_ur, short_name, type (school, college, school_and_college), curriculum (national, cambridge, dual), registration_no, logo_file_id, address, phone, email, is_active; uq_ins_name (organization_id, name).
- branches [T][S] (brn): institution_id FK->institutions, code, name, name_ur, address, city, district, phone, email, latitude, longitude DECIMAL(10,7), geofence_radius_m, available_wings JSON, opened_on, status (active, inactive), deactivated_at (no principal_employee_id, per the D07 Part 3 correction); uq_brn_code (organization_id, code), ix_brn_ins (organization_id, institution_id); uq_brn_org_id (organization_id, id).
- branch_bank_accounts [B][S] (bba) without account_id, departments [T][S] with branch_key generated and without head_employee_id (dpt), shifts [B][S] (shf), rooms [B][S] (rom): D07 7.7 with the D07 Part 3 corrections.
- organization_domains (odm), organization_subscriptions (osb), organization_modules (omd), usage_counters (usc): exactly as D07 7.6.
Every tenant parent table has uq_{alias}_org_id (organization_id, id). All FKs ON DELETE RESTRICT.

BUSINESS RULES
- A branch belongs to one institution for its lifetime (A-34).
- Reserved slugs from config/darsgah.php can never resolve to a tenant.
- Archived organizations resolve to 404; suspended ones resolve read-only.
- Writes to branch-owned models in all-branches mode are rejected with core.select_branch_first.

UI SPEC
No screens in this step (Organization screens arrive in P01.5). Unknown hosts render the 404 page from P00.3.

SECURITY AND PERMISSIONS
- Isolation layers 1, 4, 5, 6, 7, 8, 10, 12, 13 from D04 4.15 are implemented here and tested.
- Tenancy::bypass() requires a reason string and is always recorded.
- TenantResolver cache is invalidated when a domain or organization status changes (event hooks on the Platform and Organization models).

TESTS
- Unit: TenantContext requireOrganization throws without context; Tenancy::bypass restores state after exceptions.
- Feature: ResolveTenant resolves {slug}.darsgah.test to the organization; unknown host 404; reserved slug 404; archived 404; suspended sets read-only; old slug within redirect_until returns 301 to the primary domain.
- Feature: TenantAware job dispatched in A processes only A rows; job without context fails with MissingTenantContext.
- Feature: BelongsToBranch create without active branch throws core.select_branch_first; with a branch outside allowedBranchIds throws; branch_id change after create throws.
- Feature: TenantExists rejects B's ULID inside A; BranchExists rejects a branch outside allowed branches.
- Generated isolation suite (task 13) green for every M01 tenant model.
- Unit: TenantCache keys are prefixed and refuse without context.

ACCEPTANCE CRITERIA
1. php artisan migrate:fresh creates M00 and M01 tables; MigrationOrderTest passes with deferred FKs.
2. TenantIsolationTest covers Institution, Branch, BranchBankAccount, Department, Shift, Room and passes.
3. Composite FK test proves the database rejects a cross-tenant parent.
4. Any tenant model query without context throws MissingTenantContext (test evidence).
5. DependencyMapTest passes with the DEC-237 dependencies.
6. CI green on PHP 8.2 and 8.4; staging deploy green.

DO NOT
- Do not add users, roles or any authentication (P01.2 to P01.4).
- Do not build Organization or Platform screens (P01.5).
- Do not use Laravel's plain exists rule or unscoped queries in modules.
- Do not cascade deletes.

OUTPUT
Report per Master section 20, with the list of models covered by the generated isolation suite and the deferred foreign keys with the step that adds each.
```

---

### P01.2 Users, Audit Foundation and Core Service Tables

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 1 7.5 (Core tables and framework table changes), 7.8 (users only), 7.10 (Audit), D03 3.9, 3.13, D05 5.10, D12 Part 1 12.3, 12.7 (uploads), 12.9, and D14 decisions DEC-190, DEC-191, DEC-192, DEC-230, DEC-236, DEC-238.

PHASE GOAL
Create the users table, the append-only audit trail with masking, and the Core services every module needs (numbering, files, attachments, period locks, idempotency, webhook receipts, bulk operations, phone normalization), so later steps can audit, number, store files and accept retries safely.

ALREADY BUILT
PH-00 plus P01.1: TenantContext, Tenancy, scopes, BelongsToOrganization and BelongsToBranch, TenantResolver (Platform), ResolveTenant, TenantAware, TenantExists and BranchExists, TenantCache, M01 tables with deferred user FKs, generated isolation suite.

TASKS
1. Identity migration users (D07 7.8 users table, all columns and keys including organization_key generated column, uq_usr_org_id, uq_usr_email, uq_usr_mobile, uq_usr_username). Model Modules\Identity\Models\User (Authenticatable; NOT globally scoped by OrganizationScope because platform users have NULL organization; instead a UserScope applies organization filtering when TenantContext has an organization and excludes platform users on tenant hosts; tenant code always reaches users through this scope). Password cast hashed (bcrypt, BCRYPT_ROUNDS=12). Morph alias user. Factory with states platformUser(), inOrganization($org), locked(), inactive().
2. Identity migration sessions: Laravel session columns plus organization_id BIGINT UNSIGNED NULL and index (organization_id, user_id). Switch .env to SESSION_DRIVER=database.
3. Migration add_user_foreigns_to_m01_tables: add the deferred created_by and updated_by composite FKs on M01 tenant tables, organization_modules.granted_by and organization_subscriptions FKs where D07 defines them, and platform_announcements.created_by FK=>users.
4. Audit tables audit_logs and login_events exactly as D07 7.10 ([L] log tables: no foreign keys, PK (id, created_at), DATETIME(3), indexes starting with organization_id). Create them partition-ready but not partitioned yet (partitioning switches on in PH-09 per D07 7.50).
5. Modules\Core\Contracts\Audit\AuditRecorder (record(AuditEntry $entry): void) and Modules\Core\Data\AuditEntry (action, module, auditable type, id, uuid, old values, new values, reason, metadata). Modules\Core\Concerns\Auditable trait for models: on created, updated and deleted it records {module}.{resource}.created|updated|deleted with changed attributes only; models declare $auditModule, $auditResource, $auditMasked (fields masked as ***), $auditExcluded (never stored, for example password, remember_token, two_factor_secret). Explicit audit entries with reasons are written by Actions through AuditRecorder.
6. Modules\Audit\Services\AuditWriter implements AuditRecorder: writes inside the current transaction; fills organization_id, branch_id, user_id, impersonator_user_id, ip, user agent (500 chars), correlation id from TenantContext and the request; masks values through Modules\Audit\Masking\ValueMasker (masks *_encrypted, cnic, b_form, bank account, salary fields, health and counseling text per D05 5.10 and the model's $auditMasked). Never writes to log files. Bind it in AuditServiceProvider. Rebind Tenancy::bypass() auditing to it (core.tenancy.bypassed).
7. Modules\Audit\Listeners\RecordAuthenticationEvent for Laravel auth events (Login, Logout, Failed, PasswordReset, Lockout) writing login_events with identifier_masked (mobile shows last 3 digits, email shows first character and domain), event, portal, ip, user agent. Tenant mismatch and impersonation events are written by the middleware and actions that detect them (P01.4, P01.5) through a LoginEventRecorder service in Audit exposed via a Core contract LoginEventRecorder.
8. Core tables and models (D07 7.5): numbering_sequences, stored_files, file_attachments, period_locks (unlock_request_id column present, FK to approval_requests added in P02.1), idempotency_keys, webhook_receipts ([L]), bulk_operations. Models NumberingSequence, StoredFile, FileAttachment, PeriodLock, IdempotencyKey, WebhookReceipt, BulkOperation with BelongsToOrganization where they are tenant tables (WebhookReceipt is a log table without scope; only webhook processing jobs read it with explicit organization context).
9. Core services:
   - NumberingService::next(string $sequenceKey, ScopeRef $scope, ?string $periodKey): string. Locks the sequence row with SELECT ... FOR UPDATE inside the caller's transaction (throws if called outside a transaction), creates the row on first use from a registered default format, applies format tokens {BRANCH} {INST} {YYYY} {YY} {MM} {FY} {SEQ:n}, respects reset_policy and is_gapless. A numbering format registry lets modules register sequence keys with default formats.
   - FileStorageService: store(UploadedFile, FilePurpose, module, attachable?) with extension and detected-MIME allowlists per purpose (photo: jpg, png, webp; document: pdf, jpg, png; import: xlsx, csv; attachment: pdf, jpg, png, docx, xlsx), size limits from plan or config, SVG always rejected (DEC-191), random ULID names, path org/{organization_uuid}/{module}/{yyyy}/{mm}/{file_ulid}.{ext} on the private disk (DEC-230), checksum sha256, scan_status via the VirusScanner contract (NullScanner marks skipped; ClamAvScanner optional), images re-encoded and resized through Core ImageProcessor (intervention/image; EXIF stripped, DEC-192). Signed temporary download URLs (default 5 minutes) bound to the user id.
   - Modules\Core\Http\Controllers\App\FileController (route core.files.show, signed): re-checks the owning record's policy through the attachable morph before streaming; Content-Disposition attachment for documents; inline only for images.
   - PeriodLockService: assertOpen(module, branchId?, date) throws PeriodLocked unless the period is unlocked (relock_due_at in the future); lock(...) and unlock(...) (unlock is called only by the period_unlock approval handler in P02.1).
   - IdempotencyGuard: middleware alias idempotent for routes; reads Idempotency-Key header or _idempotency form field; stores key, user, route, request hash, response ref for 24 hours; same key and same hash returns the stored redirect or JSON; same key with a different hash throws IdempotencyKeyMismatch (422). Blade helper @idempotencyField renders a per-render key.
   - PhoneNumberNormalizer (giggsey/libphonenumber-for-php-lite, region PK): toE164, display format 0300 1234567.
   - Bulk operation support: Modules\Core\Services\BulkOperationRunner creating bulk_operations rows linked to Laravel job batches with progress updates, completed_with_errors handling and result file.
10. Console commands core:purge-idempotency-keys (hourly) and core:purge-temp-files (hourly) registered in routes/console.php with onOneServer()->withoutOverlapping().
11. Database grants: extend deploy/mysql/grants.sql so the application user has INSERT and SELECT only on audit_logs and login_events (DEC-190). Add CI setup that creates the restricted app user and a test group audit_privilege that connects as that user.
12. Add every new tenant model to the generated isolation suite (automatic through the trait) and add Assertions\assertAudited($action, $model) to tests/Support.
13. decisions.md rows: DEC-41 (preparation), DEC-90, DEC-95 (preparation), DEC-97, DEC-98, DEC-190, DEC-191, DEC-192, DEC-230, DEC-238.

DATABASE
- users, sessions (Identity): D07 7.8 and 7.5 framework changes.
- audit_logs, login_events (Audit): D07 7.10.
- numbering_sequences (nsq), stored_files (sfl), file_attachments (fat), period_locks (plk), idempotency_keys (idk), webhook_receipts (whr), bulk_operations (bop) (Core): D07 7.5, every column and key as written there.
- Deferred FK migration for M01 and M00 user references.

BUSINESS RULES
- Audit rows are append-only; no update or delete code path exists. Masked values cannot be recovered from the log.
- A rolled-back transaction must not consume a gapless number (lock within the caller's transaction).
- Same idempotency key with a different body is rejected; with the same body returns the first result.
- Uploads: SVG always rejected; images always re-encoded; files never public.

UI SPEC
No screens (audit and bulk operation screens arrive in P01.6).

SECURITY AND PERMISSIONS
- Restricted grants on audit tables (test group audit_privilege proves UPDATE and DELETE fail).
- File downloads require a valid signature, the same user, and a passing policy check on the owning record.
- ValueMasker covers every Restricted field class in D12 12.3.

TESTS
- Feature: Auditable writes created, updated, deleted entries with changed fields only; excluded fields never stored; masked fields stored as ***; entry has correlation id and actor.
- Feature (group audit_privilege): connecting as the restricted app user, UPDATE and DELETE on audit_logs fail.
- Feature: login events recorded with masked identifiers for success, failure and logout (using Laravel auth events fired manually until P01.4).
- Feature: NumberingService concurrency: 20 parallel transactions (pcntl or separate processes) produce 20 unique sequential numbers; a rolled-back transaction leaves no gap for a gapless sequence; format tokens render correctly; yearly reset.
- Feature: FileStorageService rejects SVG, an exe renamed to .pdf (MIME mismatch), oversize files; strips EXIF GPS from a fixture JPEG; stores under the canonical path; signed URL works for the same user and fails for another user and another tenant.
- Feature: IdempotencyGuard replays the first response for the same key and hash and returns 422 for a different hash.
- Feature: PeriodLockService throws PeriodLocked inside a locked month and passes after an unlock with relock_due_at in the future.
- Unit: PhoneNumberNormalizer handles 03001234567, +923001234567, 0300-1234567 and rejects invalid input.
- Isolation suite includes the new tenant models.

ACCEPTANCE CRITERIA
1. Migrations M02 (users, sessions, audit, Core) apply after M01 with MigrationOrderTest green.
2. Audit privilege test proves INSERT-only behaviour for the restricted user.
3. Numbering concurrency test passes with no duplicates and no gaps for gapless sequences.
4. Upload tests prove SVG rejection and EXIF stripping.
5. Every acceptance item above has a passing test named in the report.
6. CI green; staging deploy green with SESSION_DRIVER=database.

DO NOT
- Do not build login, Fortify or any screen.
- Do not write audit entries to log files or make them updatable.
- Do not expose stored files through public URLs.

OUTPUT
Report per Master section 20, including the numbering concurrency evidence and the masked audit sample row (values shown masked).
```

---

### P01.3 RBAC: Roles, Permissions, Assignments and Policies

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D05 completely (5.1 to 5.14), D04 4.9 and 4.10, D03 3.8, D07 Part 1 7.8 (roles, permissions, role_permissions, user_role_assignments, user_permission_overrides), D10 Part 2 10.13 (SCR-IAM-012 to 016, 018 to 026) and wireframe W-08, D14 decisions DEC-235, DEC-236, DEC-241.

PHASE GOAL
Build dynamic RBAC with scoped role assignments, deny-wins overrides, visibility levels, the no-escalation rule, separation-of-duties framework, field masking, the permission catalog sync, role templates, and the identity management screens, so every later module authorizes through one tested mechanism.

ALREADY BUILT
PH-00; P01.1 tenancy and M01; P01.2 users, sessions, audit trail with masking, Core services (numbering, files, period locks, idempotency, bulk operations).

TASKS
1. Migrations (Identity): permissions [P], roles, role_permissions, user_role_assignments [T-], user_permission_overrides [T-] exactly as D07 7.8; platform_user_roles per DEC-235.
2. Modules\Identity\Support\PermissionRegistry: modules contribute through Definitions/Permissions.php returning entries with key, module, resource, action, label_en, label_ur, is_sensitive, allowed_scope_types, portal, release, org_scope_only, default_template_roles (map of role slug to visibility override or null). Validate keys against {module}.{resource}.{action} and the D05 5.1 action vocabulary (DEC-53); unknown verbs fail registration.
3. Definitions/Permissions.php for the foundation modules built so far, copying D05 exactly: Platform (5.7.1, including platform.support_access.approve and .reject on the tenant side), Organization (5.7.2), Identity (5.7.3), Settings, Core and Audit (5.7.4). Mark sensitive (S) and org-scope-only (†) as D05 marks them.
4. Modules\Identity\Definitions\RoleTemplates.php: the 22 tenant templates from D05 5.5 (slug, name_en, name_ur, portal, default scope type, default_visibility, requires_two_factor, is_locked, template_version 1) and the 3 platform roles (platform_super_admin, platform_support, platform_billing). org_owner holds every tenant permission automatically and is locked. student and guardian are locked fixed sets (their permissions arrive with their modules).
5. Command identity:sync-permissions (D05 5.13): upserts permissions; marks removed keys is_deprecated (deprecated keys never grant); for every organization, grants new keys to uncustomized template roles per default_template_roles, records suggestions for customized roles (store in a small identity_permission_suggestions table [T-]: role_id, permission_id, created_at, dismissed_at); keeps org_owner complete; syncs platform roles. Idempotent. Add it to deploy/scripts/post-deploy step list and to CI after migrate.
6. Modules\Identity\Actions\SeedRoleTemplatesForOrganization (used by provisioning in P01.5): creates the organization's role copies and their role_permissions with visibility overrides.
7. Modules\Identity\Services\PermissionResolver implementing the algorithm in D05 5.3 exactly: portal check, module entitlement through Core EntitlementChecker (P01.5 binds Platform's implementation; until then a Core default returns true for core-tier modules and false otherwise), deny overrides first, scoped grants from active assignments (starts_on <= today, ends_on null or >= today) and grant overrides, org-scope-only filter, visibility as the widest allowed capped by scope (DEC-58), cache compiled per user at org:{org}:perm:user:{user}:v{userVersion}.{orgRoleVersion} with TTL min(1 hour, next expiry). Platform users resolve through platform_user_roles at platform:perm:user:{user}:v{n}.
8. Implement the Core contracts (DEC-236): Modules\Core\Contracts\Authorization\PermissionChecker (can(user, key, ?branchId, ?record), visibility(user, key, ?branchId)); Modules\Core\Support\Authorization\BasePolicy with allows($user, $key, $record = null) which checks permission, record organization, record branch within scope and record visibility (own, assigned, branch, institution, organization); assigned-set resolution delegates to a Core AssignedScopeRegistry where modules register resolvers per resource (Academic registers teaching-assignment resolvers in PH-03; until then assigned grants match nothing). Modules\Core\Concerns\HasVisibilityScope adds ->visibleTo($user, $key) to tenant models applying branch filters and, for assigned, the registered resolver's subquery.
9. Modules\Core\Http\Middleware\SetBranchContext (add to portal groups per D08 8.4): computes allowedBranchIds from active assignments (organization scope = all branches; institution scope = that institution's branches; branch scope = listed branches), sets activeBranchId from the session (validated against allowed) or the user's preferred_branch_id; supports "all branches" read mode only for users with organization or institution scope. Gate switch-branch and ContextSwitchController (route app.identity.context.switch) to change the active branch (baseline ability).
10. Actions (all audited with reasons where D05 requires): CreateStaffUser, UpdateUser, DeactivateUser (ends sessions; blocked for the last active org_owner, BR-IAM-02), UnlockUser, ForcePasswordChange, ResetTwoFactor (level 2 reason; notifies the user through OtpDelivery email channel in PH-01), AssignRole (subset rule DEC-55 with scope rule and principal limits from D05 5.12; SOD-07 no self-assignment; overlapping same role and scope rejected; future-dated allowed), RevokeRole (ends the row, never deletes), GrantPermissionOverride and RevokePermissionOverride (reason required, deny wins, subset rule, SOD-07), CreateCustomRole (clone template or empty; is_customized true), UpdateRolePermissions (diff, level 2 confirm, locked roles refused, bumps orgRoleVersion), ResetRoleToTemplate, DeleteRole (refused with active assignments). Each change bumps the right cache version and fires PermissionsChanged.
11. Separation-of-duties framework: Modules\Core\Support\Authorization\SeparationOfDuties::assert(string $ruleId, User $actor, array $context) with a registry of rule checkers (SOD-01 to SOD-10 registered by their owning modules when built; SOD-07 registered now by Identity). Violations throw DomainRuleViolation core.separation_of_duties with the rule's translated explanation.
12. Masking: Modules\Core\Services\PiiMasker (CNIC 35202-•••••••-3, phone 0300 •••••67, generic) and Blade component x-ui.masked (value, permission key; shows lock icon with tooltip when masked); exports will reuse PiiMasker.
13. Screens (app portal, routes and permissions exactly as D10 10.13): SCR-IAM-012 Users list, SCR-IAM-013 User form (staff users; employee link arrives in P05.1), SCR-IAM-014 User detail (tabs: role assignments, overrides, effective permissions, sessions placeholder hidden until P01.4 fills it, login history), SCR-IAM-015 Unlock or reset, SCR-IAM-016 Deactivate, SCR-IAM-018 Roles, SCR-IAM-019 Role permission editor (W-08 grid: module groups, resources as rows, actions as columns, visibility per permission, S and † markers, locked roles read-only, diff and level 2 confirm with users affected count), SCR-IAM-020 Create custom role, SCR-IAM-021 Reset to template, SCR-IAM-022 Assign role drawer (explains blocked roles by subset rule), SCR-IAM-023 Permission overrides, SCR-IAM-024 Effective permissions viewer (per branch: key, visibility, source), SCR-IAM-025 Active sessions (lists sessions table rows; force sign-out), SCR-IAM-026 Trusted devices list (empty until P01.4 registers devices). Navigation items in Identity Definitions/Navigation.php (group Settings: Users, Roles).
14. tests/Fixtures/rbac_matrix.php encoding D05 matrices for the permissions declared so far (role slug, permission key, expected visibility or false); RbacMatrixTest asserts seeded templates match exactly. tests/Fixtures/screens.php and ScreenCoverageTest (DEC-241) with the screens of this step.
15. tests/Support/ActsAsRole: actingAsRole('principal', branch: $branch) creates a user with exactly that template role assignment in the given scope and sets TenantContext.
16. decisions.md rows: DEC-21, DEC-34, DEC-53 to DEC-62, DEC-64, DEC-235, DEC-241.

DATABASE
- permissions (prm), roles (rol), role_permissions (rpm), user_role_assignments (ura), user_permission_overrides (upo): D07 7.8, every column and key.
- platform_user_roles (pur): DEC-235.
- identity_permission_suggestions (ips) [T-]: role_id FK->roles (composite), permission_id FK=>permissions, created_at, dismissed_at NULL; uq_ips (organization_id, role_id, permission_id).

BUSINESS RULES
- Deny overrides always win. Expired overrides and ended assignments never grant.
- Visibility never exceeds the assignment scope (DEC-58).
- Subset rule: an assigner may assign a role or grant an override only if every permission involved is within their own effective permissions for that scope with equal or wider visibility, and only inside their own scope; org_owner exempt (DEC-55).
- Principals may assign only the roles listed in D05 5.12 plus custom roles passing the subset rule.
- At least one active org_owner must always exist.
- A role with active assignments cannot be deleted. Locked roles cannot be edited.

UI SPEC
- W-08 layout for the role editor; level 2 confirmation listing every added and removed permission; S permissions show the 2FA note.
- Users list filters: status, role, branch, portal; locked badge; last login.
- Effective permissions viewer groups by module and shows the source of each grant.
- All screens use x-ui components, empty and error states, English and Urdu.

SECURITY AND PERMISSIONS
- Every route authorizes through policies extending BasePolicy with the D10 permission key.
- Self-assignment blocked (SOD-07). Escalation blocked (DEC-55).
- Changing roles, assignments or overrides is audited with before and after and reason.
- Sensitive permission keys are flagged for DEC-63 enforcement (P01.4 enforces 2FA).

TESTS
- RbacMatrixTest (fixture-driven) for every declared permission.
- Resolver tests from D05 5.14: deny beats grant; expired override ignored; future-dated assignment inactive until start; branch-scoped accountant denied an org-scope-only permission; visibility capping; cache change takes effect on the next request; override expiry lapses without manual cache clear (time travel).
- Escalation tests: principal cannot assign accountant; org admin cannot grant a permission they lack; nobody can assign roles to themselves.
- BasePolicy tests: record in another branch denied for a branch-scoped user; own visibility passes only for own records; assigned visibility with a test resolver.
- SetBranchContext tests: allowed branches per scope type; all-branches mode only for organization or institution scope; invalid session branch falls back to preferred or first allowed.
- Feature tests for each screen: 200 with permission, 403 without, 404 for another tenant's user ULID, audit entry on every change.
- Command test: identity:sync-permissions idempotent; new key granted to uncustomized template copy, suggested for customized role; deprecated key stops granting.
- ScreenCoverageTest passes for the listed screens.

ACCEPTANCE CRITERIA
1. RbacMatrixTest and all D05 5.14 test groups applicable to declared permissions pass.
2. The role editor saves a change with a level 2 confirmation, bumps the cache version and the change applies on the next request (test evidence).
3. Escalation and SOD-07 attempts fail with translated messages.
4. identity:sync-permissions runs in CI and on staging deploy.
5. Screens SCR-IAM-012 to 016 and 018 to 026 exist with the D10 route names and permissions and pass ScreenCoverageTest.
6. CI green; staging green.

DO NOT
- Do not use any third-party permission package (DEC-34).
- Do not add a Gate::before bypass for any role, including platform super admin (BR-PLT-01).
- Do not hardcode role checks (no "if role is principal"); always check permission keys.
- Do not build login or 2FA (P01.4).

OUTPUT
Report per Master section 20, with the RBAC matrix coverage count, screenshots of W-08 in English and Urdu, and the escalation test names.
```

---

### P01.4 Authentication, Sessions and Two-Factor

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D05 5.6, D03 3.7 (middleware 4 to 12), D04 4.7 (cookies) and 4.8, D07 Part 1 7.8 (password_reset_tokens, one_time_passwords, trusted_devices), D08 8.4 (groups) and 8.7 (fortify, session, security), D09 9.17 (sign-in wireframe) and 9.13 (3.3.8 accessible authentication), D10 10.13 (SCR-IAM-001 to 006, 008 to 011) and 10.11 (SCR-PLT-002), D12 Part 1 12.5 completely, D14 decisions DEC-239 and DEC-240.

PHASE GOAL
Let staff and platform users sign in safely on their own hosts with mobile, email or username; give first-time users an OTP path to set a password; support organization-aware password reset, TOTP two-factor, idle and absolute session limits, trusted devices and self-service profile screens, with every event logged.

ALREADY BUILT
PH-00; P01.1 tenancy; P01.2 users, sessions, audit trail, Core services; P01.3 RBAC with PermissionResolver, BasePolicy, SetBranchContext, role templates, identity:sync-permissions, IAM management screens, platform_user_roles.

TASKS
1. Install laravel/fortify (package check). config/fortify.php: features reset passwords, two-factor authentication with confirm, update passwords, confirm password; registration and email verification off; views off. Custom pipeline: Modules\Identity\Fortify\AuthenticateUser via Fortify::authenticateUsing (tenant host: identifier normalized by LoginIdentifierNormalizer to mobile E.164, email lowercase or username, looked up inside the resolved organization through organization_key; platform host: users with organization_id NULL only), status checks (inactive or locked refuse with a generic message), lockout (failed_login_count, locked_until per DEC rules below), password rehash when needed, last_login_at and last_login_ip.
2. Migrations: password_reset_tokens custom (DEC-96), one_time_passwords, trusted_devices exactly as D07 7.8. Organization-aware password broker Modules\Identity\Services\OrganizationPasswordBroker (token hashed SHA-256, single use, 60 minutes, invalidated by password change; channel email or sms) replacing Laravel's broker in config/auth.php.
3. Modules\Identity\Services\OtpService: issue(destination, purpose, ?user) generates a 6-digit code, stores code_hash, expires in 5 minutes, max 5 attempts, resend allowed after 60 seconds, max 5 sends per hour per destination; verify(destination, purpose, code). Responses identical whether or not the destination exists. Modules\Identity\Contracts\OtpDelivery (DEC-239) with EmailOtpDelivery (Laravel Mail, translated template) and LogOtpDelivery (local and staging only, writes the code to the log with a [DEV-ONLY] marker; refuses in production). Command identity:prune-otps hourly.
4. Password rules (DEC-188) in Modules\Identity\Rules\PasswordPolicy: minimum 8 for guardians and students, 10 for staff, 12 for platform users and holders of sensitive permissions; staff and platform passwords also checked with Laravel's uncompromised() rule; if the breach service is unreachable the check passes and an entry goes to the security log channel. No composition rules. Paste allowed.
5. Middleware (add to groups per D08 8.4): Modules\Core\Http\Middleware\EnsureTenantMatchesUser (logs out a user whose organization differs from the host's and records login_events tenant_mismatch plus a security log entry), Modules\Identity\Http\Middleware\EnsureActiveUser, EnsurePlatformUser, RequireTwoFactor (platform users always; tenant users when any effective permission is sensitive and the organization setting identity.require_two_factor_for_sensitive is true, default true per DEC-63; redirects to SCR-IAM-010 setup or SCR-IAM-006 challenge), IdleTimeout (DEC-189: staff idle 30 minutes, 15 minutes for users with sensitive permissions, absolute 12 hours; platform idle 15 minutes, absolute 8 hours; guardian and student values are applied when those portals open in PH-04; values from config/security.php, overridable only to stricter values by organization settings), EnsurePortal (portal:app requires an active staff role assignment in the organization; portal:platform requires a platform role; guardian and student resolve through a PortalAccessResolver registry that denies until Guardian and Student register resolvers in PH-04). Session regenerated on login, on role change for the current user, on branch switch and on portal switch.
6. Auth routes in Modules\Identity routes/auth.php (tenant hosts with tenant.guest; platform host with a platform guest group): auth.login (GET, POST), auth.logout (POST), auth.otp.request, auth.otp.verify (first-time activation or password set; works for any existing user with a mobile or email; sets mobile_verified_at or email_verified_at), auth.password.request, auth.password.reset, auth.two_factor.challenge, auth.platform.login (platform host). Rate limiter auth (5 per minute per identifier plus IP); 10 failures in 15 minutes sets locked_until 15 minutes ahead and notifies the user through OtpDelivery email channel; lockout message names the wait time.
7. Self-service screens (baseline abilities, DEC-57) for the app and platform portals: SCR-IAM-008 My profile ({portal}.identity.profile.show: name, Urdu name, locale, Urdu font Nastaliq or Naskh, photo upload through FileStorageService), SCR-IAM-009 Change password ({portal}.identity.password.edit; current password required; option to sign out other sessions), SCR-IAM-010 Two-factor setup ({portal}.identity.two_factor.edit: QR, confirm code, recovery codes download, required banner), SCR-IAM-011 My sessions and devices ({portal}.identity.sessions.index: sign out a session; remove a trusted device). Fill the sessions tab on SCR-IAM-014 and make SCR-IAM-025 force sign-out work. Guardian and student variants of these routes are added in PH-04.
8. Trusted devices: RegisterTrustedDevice action (OTP confirmation; fingerprint hash from a device token cookie plus user agent family; revoked on removal); listed in SCR-IAM-011 and SCR-IAM-026 (removal by ITA, HRM, OA with audit). Used by staff online check-in in PH-05.
9. New device notice: first sign-in from a new device token for staff and platform users sends a notice via OtpDelivery email channel.
10. Command identity:create-platform-admin (DEC-240).
11. Landing after sign-in: app users to app.identity.profile.show and platform users to platform.identity.profile.show until P01.6 adds {portal}.home; implement through a single Modules\Identity\Services\HomeRedirector so P01.6 changes one place.
12. Sign-in page per D09 9.17 (auth layout, organization logo and name when on a tenant host, language switch English and Urdu, identifier field with inputmode, password reveal, "Forgot password?" and "First time? Get code" links, rate-limit and lockout messages). OTP fields accept a full pasted code. No puzzle captcha.
13. Append the new screens to tests/Fixtures/screens.php; add baseline routes to tests/Fixtures/baseline_routes.php.
14. decisions.md rows: DEC-09, DEC-10, DEC-33, DEC-44, DEC-47 (preparation), DEC-57, DEC-63, DEC-96, DEC-188, DEC-189, DEC-239, DEC-240.

DATABASE
password_reset_tokens (prt), one_time_passwords (otp), trusted_devices (tdv) exactly as D07 Part 1 7.8. Uses users, sessions and login_events from P01.2.

BUSINESS RULES
- Login errors never reveal whether an account exists.
- A user of organization A on organization B's host is logged out (tenant mismatch) and the event is recorded.
- Platform users can sign in only on the platform host; tenant users never on the platform host.
- Deactivation ends all sessions immediately (database session rows removed for that user).
- Password change invalidates reset tokens and, on request, other sessions.
- OTP: 6 digits, 5 minutes, 5 attempts, resend after 60 seconds, 5 per hour per destination.

UI SPEC
- D09 9.17 sign-in wireframe; D09 9.15 field and input specs; errors with icon and fix hint; focus on the error summary.
- Two-factor setup is a 3-step wizard (QR, confirm, recovery codes) using x-ui.stepper (add the component if missing, per D09 9.15).
- Profile screen allows choosing Urdu font (data-urdu-font) and language; changes apply immediately.

SECURITY AND PERMISSIONS
- Host-only cookies, Secure, HttpOnly, SameSite=Lax (DEC-44).
- Passwords bcrypt cost 12; OTP and reset tokens stored hashed only.
- No password, OTP or token appears in logs (RedactionProcessor tests extended with the new field names).
- Two-factor enforced for platform users and sensitive-permission holders.
- Login, logout, failure, lockout, password reset, two_factor_failed and tenant_mismatch recorded in login_events.

TESTS
- Feature: sign in with mobile (several input formats), email and username on a tenant host; wrong password generic message; inactive and locked users refused; lockout after 10 failures in 15 minutes with locked_until set; auth limiter returns 429.
- Feature: tenant user on another tenant's host is logged out and tenant_mismatch recorded; session cookie from host A not accepted on host B (host-only cookie test).
- Feature: platform user on a tenant host refused; tenant user on the platform host refused.
- Feature: OTP request returns the same response for known and unknown numbers; verify succeeds once; expired and over-attempt codes fail; send limits enforced.
- Feature: password reset token is organization-aware, single use, expires at 60 minutes, invalid after password change.
- Feature: RequireTwoFactor challenges a user who gains a sensitive permission without confirmed 2FA; platform user always challenged; recovery code works once.
- Feature: IdleTimeout logs out after the configured idle period and after the absolute lifetime (time travel); session regenerated on login and branch switch.
- Feature: EnsurePortal denies app portal to a user without staff assignments; guardian and student portals deny until PH-04.
- Feature: deactivated user's existing session is rejected on the next request.
- Unit: PasswordPolicy lengths by user class; uncompromised check fails open with a security log entry when the service is unreachable (HTTP fake).
- Redaction tests extended; ScreenCoverageTest extended.

ACCEPTANCE CRITERIA
1. Staff sign-in, OTP activation, password reset and 2FA flows work end to end on staging (log or email delivery), with screenshots in English and Urdu.
2. identity:create-platform-admin creates the first super admin; first sign-in forces password change and 2FA enrolment.
3. All listed security tests pass; login_events rows present for each event type.
4. D12 12.24 tests for authentication, sessions and throttling pass.
5. CI green; staging green.

DO NOT
- Do not send SMS directly; use OtpDelivery (SMS arrives in P02.2).
- Do not enable Fortify registration.
- Do not store OTPs or reset tokens in plain text or log them, including the LogOtpDelivery channel outside local and staging.
- Do not open guardian or student portals (PH-04).

OUTPUT
Report per Master section 20, including the flow screenshots, the login_events sample (masked), and the staging verification of the first platform admin.
```

---

### P01.5 Platform and Organization Management

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D04 4.7, 4.19, 4.21; D05 5.7.1, 5.7.2, 5.12; D03 3.24 (Platform and Organization); D07 Part 1 7.6 and 7.7 (support_access_grants included); D10 Part 2 10.11 (SCR-PLT-003 to 018) and 10.12 (SCR-ORG-001 to 011); D12 Part 1 12.5 (impersonation limits); D09 9.17 (platform portal, impersonation banner); D14 DEC-233, DEC-235, DEC-237.

PHASE GOAL
Let platform staff provision and manage organizations (domains, plans, entitlements, states, support access with owner-approved impersonation), and let organization owners and admins manage their structure (profile, institutions, branches, bank accounts, departments, shifts, rooms) through an onboarding wizard, with entitlement checks enforced on every module route.

ALREADY BUILT
PH-00; P01.1 tenancy and M01 tables; P01.2 users, audit, Core services; P01.3 RBAC, role templates, IAM screens; P01.4 authentication, OTP, 2FA, sessions, platform admin command.

TASKS
1. support_access_grants migration (D07 7.6) owned by Platform.
2. Modules\Platform\Services\EntitlementService implements Core EntitlementChecker: module enabled when the organization's active subscription plan includes it or an organization_modules override enables it (within starts_on and ends_on), and all its depends_on modules are enabled; cached at org:{id}:entitlements:v{n}; core-tier modules always enabled. Modules\Core\Http\Middleware\EnsureModuleEnabled (alias module) returns 404 (ModuleNotEnabled) when not entitled; added by ModuleRouteLoader as D08 8.6 states. LimitEnforcer reads plan limits (students, branches, storage, SMS) with NULL meaning unlimited; branch creation checks max_branches now.
3. Actions (Platform): ProvisionOrganization (one transaction: organizations row with status trial or active, primary organization_domains row {slug}.{root_domain} type platform_subdomain, organization_subscriptions row for the chosen plan, optional module overrides, Identity contract to seed role templates, Settings contract to apply defaults (no-op until P01.6 registers settings), Identity contract to create the owner user with org_owner organization-scope assignment, then after commit send the owner invitation through OtpDelivery with an activation link to auth.otp.request; event OrganizationProvisioned), ChangeTenantState (trial, active, suspended, archived; level 3 with reason; suspended tenants read-only; archived tenants cannot sign in), ChangeOrganizationSlug (SA only; old domain kept with redirect_until now + 90 days), OverrideModuleEntitlement (reason, expiry; warns about depends_on), UpdatePlan, GrantSupportAccess flow: RequestSupportAccess (platform support or SA: reason, target tenant user, duration up to 8 hours), ApproveSupportAccess and RejectSupportAccess (tenant side, org_owner, platform.support_access.approve and .reject), RevokeSupportAccess, StartImpersonation (active grant required; opens the tenant host via a signed one-time token; impersonated session stores impersonator_id in TenantContext and session), EndImpersonation. Every action audited; impersonation start and end also recorded in login_events.
4. Impersonation limits (D12 12.5): while impersonating, Restricted fields stay masked even if the impersonated user holds full-view permissions, password, 2FA and profile security changes are refused, and money approvals are refused (a Core ImpersonationGuard used by PiiMasker, Identity actions and, later, approval decisions). The D09 impersonation banner shows on every page with "End session".
5. Platform users (SCR-PLT-018): list and form for platform users with the three platform roles through platform_user_roles; 2FA required; only platform_super_admin manages them. Horizon gate now allows platform_super_admin.
6. Actions (Organization): UpdateOrganizationProfile, CreateInstitution, UpdateInstitution, CreateBranch (LimitEnforcer), UpdateBranch (geofence lat, long, radius), DeactivateBranch (blocked while active enrollments exist; until Student exists this check is a registered BranchDeactivationCheck registry with no contributors; level 3 confirm), bank account CRUD (account number masked after save), departments, shifts (times, working days), rooms. Events BranchCreated, BranchDeactivated.
7. Onboarding: Modules\Organization\Support\OnboardingStepRegistry and SCR-ORG-001 wizard (app.organization.onboarding.show) with Organization's own steps now: organization profile, institutions (type and curriculum: national, cambridge, dual), branches (Scenario A creates one institution and one branch automatically when the owner chooses "single school"), shifts. Progress is stored in the organization_onboarding_steps table [T-] (step_key, completed_at, completed_by; uq_oos (organization_id, step_key)). Later modules add steps (academic defaults PH-03, fee heads PH-06, grading PH-07).
8. Screens (routes and permissions exactly as D10): Platform portal SCR-PLT-003 Organizations, 004 Organization detail, 005 Provision organization wizard, 006 Change state, 007 Domains (subdomain only; custom domains are R2), 008 Module entitlements, 009 Plans, 010 Plan form, 011 Usage (shows usage_counters; values fill from R2 metering, the screen shows "No usage recorded yet" until then), 012 Support access requests, 013 Request support access, 014 Start impersonation, 015 Platform announcements (list, form, schedule; tenant top bar shows active announcements), 016 System health (Horizon queue sizes and waits, failed jobs with retry and discard, Gotenberg health, webhook failures count), 018 Platform users. App portal SCR-PLT-017 Support access grants (tenant side), SCR-ORG-001 to 011.
9. Navigation contributions: Platform portal items per D10 10.4 (Organizations, Plans, Module entitlements, Usage, Support access, Announcements, Platform users, Queue and system health); staff portal Settings group items for Organization profile, Structure, Institutions, Branches, Bank accounts, Departments, Shifts, Rooms, Support access.
10. Branch switcher component x-organization.branch-switcher in the app topbar (hidden for single-branch organizations, PR-10) wired to app.identity.context.switch; scope line shows active branch or "All branches" with the D09 all-branches banner.
11. Append screens to the fixture; extend RBAC fixture for Platform and Organization permissions; isolation suite covers new tenant models.
12. decisions.md rows: DEC-04, DEC-49 (preparation), DEC-50, DEC-51 (preparation), DEC-61, DEC-150, DEC-233, DEC-237.

DATABASE
- support_access_grants (sag) per D07 7.6.
- organization_onboarding_steps (oos) [T-]: step_key VARCHAR(60), completed_at DATETIME, completed_by FK->users; uq_oos (organization_id, step_key).
- Uses M01 tables from P01.1.

BUSINESS RULES
- Slugs: lowercase letters, digits and hyphens, 3 to 40 characters, not reserved, unique; only SA changes them.
- Provisioning is atomic; the invitation is sent only after commit.
- Platform staff never see tenant records, only counts and metadata; tenant data access requires an approved, unexpired support grant (BR-PLT-01).
- An organization with one branch never sees branch switching or branch columns.
- Module entitlement disabled means hidden navigation and 404 routes (FR-GEN-012).

UI SPEC
- Platform portal shell with platform bar (D09 9.17, DEC-150).
- Provision wizard steps: organization, owner user, slug with live availability check, plan, modules, review (D10 SCR-PLT-005).
- Change state uses level 3 confirmation with reason.
- Impersonation banner per D09 9.17 on every tenant page during support sessions; cannot be dismissed.
- Organization screens use list and form patterns from D09 9.16; bank account numbers masked after save.

SECURITY AND PERMISSIONS
- Platform routes on the platform host only, portal.platform group, permissions from D05 5.7.1.
- Tenant-side grant approval only by org_owner (platform.support_access.approve).
- Impersonation limits enforced server-side (ImpersonationGuard) and tested.
- Every platform action audited with organization_id NULL and the platform user as actor; impersonated actions carry both identities.

TESTS
- Feature: provisioning creates organization, domain, subscription, role copies (22), owner user with org_owner assignment; invitation sent after commit only (rollback test sends nothing); OrganizationProvisioned fired.
- Feature: state changes (suspended tenant read-only returns 423 on writes; archived tenant sign-in refused); slug change creates a redirect domain for 90 days.
- Feature: entitlement: disabled module routes return 404 and navigation hides items; override with expiry lapses; depends_on enforced.
- Feature: support access: request, owner approval, impersonation start with signed token, banner present, Restricted fields masked during impersonation, password change refused during impersonation, expired grant refused, end session recorded in audit and login_events.
- Feature: branch limit enforced from plan; deactivate branch blocked by registered checks (fake contributor in test).
- Feature: onboarding wizard single-school path creates one institution and one branch.
- Feature tests per screen: 200 with permission, 403 without, 404 cross-tenant; audit entries.
- RBAC fixture updated and green; isolation suite green; ScreenCoverageTest green.

ACCEPTANCE CRITERIA
1. MS-04 on staging: a platform admin provisions an organization; the owner receives the invitation (email or log), activates with OTP, signs in on {slug}.staging.darsgah.app, and lands in the onboarding wizard. Screenshots of each step in the report.
2. Support access with impersonation works end to end with the banner and limits.
3. Entitlement 404 behaviour proven for a disabled standard-tier module route (use a test route registered in the test suite).
4. All screens listed exist with D10 route names and permissions.
5. CI green; staging green.

DO NOT
- Do not give platform users any path to tenant data without an approved support grant.
- Do not implement custom domains, usage metering or tenant export (R2).
- Do not hardcode plan contents in code paths other than the seeder.

OUTPUT
Report per Master section 20 with the MS-04 staging walkthrough screenshots and the impersonation test names.
```

---

### P01.6 Settings, Audit Screens, Localization, Navigation and Portal Homes

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 1 7.9 (settings, translation_overrides), D03 3.18, 3.23, D04 4.14 (configuration levels), D09 9.20 (branding generator and gates G1 to G4), 9.21 (formats), D10 Part 1 10.1 (DEC-152 to DEC-159), 10.4, 10.5, 10.6 (platform widgets, onboarding checklist), D10 Part 2 10.14 to 10.16 and SCR-PLT-001, SCR-PLT-019, D14 DEC-241.

PHASE GOAL
Provide layered settings with inheritance, tenant branding with contrast gates, terminology overrides, numbering formats, period lock screens, bulk operation screens, audit log and record history screens, the navigation registry wired into every portal, and the portal home dashboards with the onboarding checklist, so PH-01 ends with a usable, auditable shell.

ALREADY BUILT
PH-00; P01.1 tenancy; P01.2 users, audit trail, Core services; P01.3 RBAC and IAM screens; P01.4 authentication, OTP, 2FA, sessions, self-service screens; P01.5 platform and organization management, entitlements, support access and impersonation, onboarding wizard with Organization steps.

TASKS
1. Migrations (Settings): settings [T-] and translation_overrides [T-] exactly as D07 7.9. Onboarding progress stays in organization_onboarding_steps (P01.5); do not copy it into settings.
2. Modules\Settings\Support\SettingRegistry: modules contribute Definitions/Settings.php entries (key {module}.{key}, type: bool, int, decimal, string, enum, json, color, file; default; allowed scopes organization, institution, branch; tighten_only flag for security settings; category; label keys; validation rules). SettingResolver: value = branch override, else institution override, else organization value, else platform_settings, else code default; cached at org:{org}:settings:{scope}:{id}:v{version}; SettingChanged event; every change audited with old and new values.
3. Register settings for the foundation modules: identity.require_two_factor_for_sensitive (bool, tighten-only off->on allowed freely, on->off requires level 2 reason and is audited, DEC-63), security.idle_minutes_staff and security.idle_minutes_sensitive (int, tighten-only), localization.default_locale, localization.number_grouping (international, south_asian), localization.date_format, branding.brand_color (color), branding.logo_file_id (file), organization display preferences. Settings defaults applied at provisioning through the Settings contract used by P01.5.
4. Branding (SCR-SET-003): logo upload (PNG, JPG or WebP only, DEC-191; resized copies 32px, 64px, PDF 18mm, ID card), brand colour input; Modules\Settings\Services\BrandingService generating the 11-step accent scale in OKLCH at the D09 9.20 lightness targets, applying gates G1 to G4 with the 0.01 adjustment loop, storing the 11 values; layouts emit a nonce-tagged <style> block with --color-accent-* and the accent role tokens for the organization; live preview of top bar, sign-in, voucher header and ID card; warning (not block) when the hue is within 20 degrees of the danger hue.
5. Translation: Modules\Settings\Translation\OverrideTranslationLoader decorating the translator so translation_overrides apply per organization and locale; cached at org:{org}:i18n:{locale}:v{version}. Modules\Core\Http\Middleware\SetLocale sets app locale, Carbon locale, html lang and dir, and the Urdu font preference.
6. Screens (routes and permissions exactly as D10): SCR-SET-001 Settings hub (categories, level selector, inherited versus overridden values), SCR-SET-002 Settings category form (source of each value, "Use inherited"), SCR-SET-003 Branding, SCR-SET-004 Language and formats, SCR-SET-005 Terminology (override any translation key per language with preview), SCR-SET-006 Numbering formats (app.core.numbering.edit; pattern per registered sequence key, next number preview, counters can never be reduced), SCR-CORE-001 Period locks list, SCR-CORE-002 Lock period (open-item checks registry, empty until modules contribute), SCR-CORE-004 Bulk operation progress (live progress JSON, failed items, result file), SCR-CORE-005 My bulk operations, SCR-AUD-001 Audit log (filters user, module, action key, record, date, IP; masked values), SCR-AUD-002 Audit entry drawer (diff, reason, request context, correlation id), SCR-AUD-003 Record history drawer (DEC-160; checks audit.record_history.view plus the record's view permission; a reusable x-audit.history-button for detail pages), SCR-AUD-004 Login history, SCR-PLT-019 Platform audit (platform host; platform actions, impersonations, state changes). SCR-CORE-003 Request unlock arrives in P02.1 and SCR-AUD-005 Export audit log in P02.3.
7. Navigation: Modules\Core NavigationRegistry runtime wired to the app, guardian, student and platform layouts; Modules contribute Definitions/Navigation.php items (portal, group, label key, route, permission, module key, sort order, icon). Staff groups fixed per D10 10.4 (Home, My work, Students, Academics, Finance, People, Operations, Reports, Settings); items show only when the module is entitled and the user holds the view permission for the active branch (DEC-158). "My work" baseline items always present.
8. Reporting module minimum for homes (DEC-152): Modules\Reporting routes/home.php mounted at /{portal} with names app.home, guardian.home, student.home, platform.home using each portal's group (add home.php support to ModuleRouteLoader per the D10 10.1 correction). Modules\Reporting\Services\DashboardComposer rendering the frame with skeletons and per-widget loading through GET /app/reporting/widgets/{key} (internal JSON, DEC-159) and the platform equivalent; Core contracts DashboardWidget and AttentionItemRegistry (DEC-155) with Reporting runtime implementations; "View as" switcher per DEC-153 precedence (remembered in users.preferred_portal or a user preference setting). Widgets in this step: organization.onboarding_checklist (Organization; shown to OW and OA until onboarding completes), the "Needs attention" widget frame (empty "All clear" state until modules contribute items), and the platform widgets platform.tenants_by_state, platform.new_organizations, platform.support_sessions, platform.system_health, platform.webhook_failures (platform.usage_limits shows its empty state until R2 metering). Switch Identity HomeRedirector to {portal}.home.
9. Platform announcements display in the tenant top bar (active, audience all or plan) with dismiss per user per announcement (stored in the user preference setting).
10. Append screens to the fixture; RBAC fixture for settings, core and audit permissions; TranslationParityTest green.
11. decisions.md rows: DEC-11, DEC-137, DEC-138 to DEC-143, DEC-147, DEC-151 to DEC-159, DEC-160.

DATABASE
settings (set), translation_overrides (tro) per D07 Part 1 7.9 (platform_settings exists from M00).

BUSINESS RULES
- Settings inheritance: branch over institution over organization over platform over code default. Security settings can only be tightened at lower levels (D04 4.14).
- Branding changes the accent scale only; primary actions and status colours never change (DEC-141). Gates G1 to G4 always pass for stored scales.
- Numbering counters never decrease; format changes apply to the next number only.
- Record history shows only to users who can view the record itself.

UI SPEC
- Settings hub shows inherited values with their source and an override indicator.
- Branding preview per D09 9.20 with the computed gate values shown.
- Audit log uses the data table pattern with filters in the URL; entry drawer shows before and after diff with masked values.
- Dashboards follow D10 10.5: scope line first, needs attention, then numbers; per-widget skeletons; "As of" stamps on snapshot widgets.
- Navigation renders from the registry in all portals; RTL mirrored correctly.

SECURITY AND PERMISSIONS
- Routes and permissions exactly as D10 10.14 to 10.16 and SCR-PLT-019.
- Setting changes audited with old and new values; turning 2FA enforcement off requires level 2 reason.
- Widget endpoints check each widget's data permission for the active branch before returning data (DEC-154).
- Platform audit visible to platform super admin only.

TESTS
- Unit: SettingResolver inheritance across all levels; tighten-only refusal at a lower level; cache version bump on change.
- Unit: BrandingService generates passing scales for maroon #7A1F2B, green #1B5E20, gold #F4B400 and edge colours (pure yellow #FFFF00, near-black #111111, grey #808080, neon green #39FF14), each passing G1 to G4.
- Feature: translation override changes a rendered label for that organization only (isolation).
- Feature: navigation shows an item only with entitlement and permission; disabled module item hidden; baseline items always present.
- Feature: widget endpoint returns 403 without data permission and 200 with it; onboarding checklist disappears when onboarding completes.
- Feature: audit log masking; record history drawer refuses users without the record's view permission.
- Feature: period lock screen locks a month; PeriodLocked thrown afterwards for a registered test module.
- Feature tests per screen (200, 403, 404 cross-tenant) and audit entries; ScreenCoverageTest; TranslationParityTest.

ACCEPTANCE CRITERIA
1. An owner signs in on staging and lands on app.home with the onboarding checklist, branch scope line and correct navigation; a platform admin lands on platform.home with platform widgets. Screenshots in English and Urdu.
2. Branding with a tenant colour updates the accent across the shell and sign-in page with gates passing.
3. Terminology override ("Class" to "Grade") shows only in that organization.
4. Audit log and record history show masked values and correct filtering.
5. All screens listed pass ScreenCoverageTest.
6. CI green; staging green.

DO NOT
- Do not put business settings in config files or code constants.
- Do not let tenant branding change primary or status colours.
- Do not build dashboards for modules that do not exist yet; only the widgets listed.

OUTPUT
Report per Master section 20 with home screenshots for owner and platform admin, branding samples with gate values, and the list of settings registered.
```

---

### P01.7 UG-1: Upgrade to Laravel 13

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies (this step changes section 2 from "Until UG-1" to "After UG-1"). Read D01 1.1 (DEC-01), D03 3.21, D08 8.14, D12 Part 2 12.31 and 12.35 (upgrade-gate runbook), D13 DEC-220 and 13.8. The official Laravel 13 upgrade guide applies. Deadline: done by 29 Jan 2027; Laravel 12 receives no security fixes after 24 Feb 2027.

PHASE GOAL
Move the application to Laravel 13 with a PHP 8.4 minimum, Pest 4 and a PHP 8.4 and 8.5 CI matrix, with every test green and staging deployed, and retire the PHP 8.2 compatibility rules in a controlled way.

ALREADY BUILT
PH-00 and P01.1 to P01.6: the complete foundation shell on Laravel 12 with PHP 8.4 runtime and a PHP 8.2 and 8.4 CI matrix.

TASKS
1. Create docs/runbooks/upgrade-gate.md (branch, update, full CI and nightly, staging load check, deploy, 24-hour watch) and follow it for this step.
2. Before changing anything, check every runtime package for Laravel 13 and PHP 8.4 and 8.5 support (laravel/fortify, laravel/horizon, gotenberg/gotenberg-php, maatwebsite/excel, intervention/image, giggsey/libphonenumber-for-php-lite, symfony/html-sanitizer, sentry/sentry-laravel if installed). If any package lacks support, stop and report (rule 3).
3. composer.json: "php": "^8.4", "laravel/framework": "^13.0"; update first-party packages to their Laravel 13 compatible majors; pestphp/pest, pest-plugin-laravel, pest-plugin-arch to ^4; larastan to its Laravel 13 compatible version. Apply every change required by the Laravel 13 upgrade guide that affects this codebase, with one commit per guide item.
4. Tooling: phpstan.neon phpVersion 80400; CI matrix PHP 8.4 and 8.5 (replace Dockerfile.ci 8.2 with an 8.5 image); production and staging images stay on PHP 8.4.
5. Architecture tests: remove the bans on typed class constants, #[\Override] and json_validate() from ForbiddenCodeTest; add a ban on property hooks and asymmetric visibility in classes extending Illuminate\Database\Eloquent\Model; keep the implicit nullable parameter ban (failOnDeprecation stays on).
6. Package rule: update the Master Context Prompt copy in docs/agent/phase-0-master-context.md only by recording "UG-1 completed on {date}" at the top; the rules in section 2 already describe the post-UG-1 state. Update D03 3.21 rule notes in docs/decisions/decisions.md (DEC-01 status: UG-1 done) and DEC-199.
7. Prepare browser testing (Pest 4): add tests/Browser with one smoke test (sign-in page renders in English and Urdu with no console errors) running in the nightly stage; full browser and accessibility suites arrive in later phases (D12 12.23).
8. Run the full suite, the nightly gotenberg group, the audit_privilege group, and a smoke run on staging; deploy to staging through the normal merge pipeline.
9. Write the upgrade notes in docs/agent/reports/P01.7.md including every guide item applied.

DATABASE
No schema changes. Migrations must run unchanged on a fresh database and on a copy of the staging database.

BUSINESS RULES
- No behaviour change is allowed. Any test change must be a mechanical update for framework APIs, never a weakened assertion.

UI SPEC
No visual change. Compare the P00.3 UI gallery and the P01.6 home screenshots before and after; differences must be zero or explained.

SECURITY AND PERMISSIONS
- Security headers, CSP, cookies, rate limiters, redaction and tenancy tests must remain green without modification of their assertions.
- composer audit and npm audit clean.

TESTS
- Full suite on PHP 8.4 and 8.5; architecture tests updated per task 5; browser smoke test in nightly.
- Migration fresh and upgrade checks.

ACCEPTANCE CRITERIA
1. composer show laravel/framework reports 13.x; php artisan about shows Laravel 13 on PHP 8.4 in staging.
2. CI green on PHP 8.4 and 8.5; nightly green including the browser smoke test.
3. composer prohibits php 8.5 reports nothing.
4. Staging deployed from the merge pipeline; smoke tests pass; screenshots unchanged.
5. MS-03 recorded with date in docs/decisions/decisions.md.

DO NOT
- Do not combine this upgrade with feature work.
- Do not downgrade or fork a package to force compatibility; stop instead.
- Do not weaken any test assertion to make the upgrade pass.

OUTPUT
Report per Master section 20 with the package compatibility table, upgrade guide items applied, CI links for PHP 8.4 and 8.5, and staging about output. Then run the D13 13.6 phase gate for PH-01 and attach its evidence list.
```

---

## PH-01 Gate Reminder

Before PH-02 starts, run D13 13.6 (G-1 to G-11) and confirm that MS-03 and MS-04 are recorded. These must all be green:
- the isolation suite for every model in M01 and M02
- the RBAC matrix for every declared permission
- the D12 12.24 tests for authentication, sessions, throttling, audit privilege and headers
- `ScreenCoverageTest` for every PLT, ORG, IAM, SET, CORE and AUD screen built in this phase

---

## Assumptions (Part 2)

| ID | Assumption |
|---|---|
| A-113 | Email (SMTP) works on staging before P01.4, so invitations and OTPs can be received before SMS arrives in P02.2. |
| A-114 | Laravel's breached-password check (k-anonymity range query) is reachable from staging and production. When it isn't, the check fails open and the event is logged (DEC-188). |
| A-115 | Support sessions last at most 8 hours per grant. Owners can end one early from SCR-PLT-017. |

## Gaps Found and Added (Part 2)

| ID | Gap | Decision |
|---|---|---|
| GAP-287 | Platform users had no place to hold their roles (`user_role_assignments` requires an organization) | `platform_user_roles` (DEC-235) |
| GAP-288 | Lower modules needed audit, authorization and tenant resolution from higher modules, which created circular dependencies | Core contracts implemented higher up (DEC-236) |
| GAP-289 | Organization and Platform depended on each other | DEC-237 |
| GAP-290 | M01 and Core tables referenced `users` before it existed | Users and audit tables moved to P01.2 (DEC-238) |
| GAP-291 | OTP delivery needed SMS before the Notification engine existed | `OtpDelivery` contract (DEC-239) |
| GAP-292 | No defined way to create the first platform super admin | DEC-240 |
| GAP-293 | Screen coverage had no shared source to grow step by step | `tests/Fixtures/screens.php` (DEC-241) |
| GAP-294 | Onboarding progress had no store before Settings existed | `organization_onboarding_steps` table (P01.5) |
| GAP-295 | Customized roles had nowhere to keep suggested new permissions (D5 5.13) | `identity_permission_suggestions` table (P01.3) |

---

Reply CONTINUE for Deliverable 14, Part 3
