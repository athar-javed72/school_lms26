# Deliverable 8: Laravel 12 Application Folder Structure and API Architecture

**Version:** 1.0 (draft for approval) | **Date:** 23 Nov 2026
**Builds on:** D1 1.15 and 1.16 (module catalog, naming), D3 (layers, anatomy, MB rules, registries, queues, HTTP surface), D4 (hosts, tenant context), D5 (permissions), D7 (tables, migration batches M00 to M14)

---

## 8.1 New Decisions

| ID | Decision |
|---|---|
| DEC-127 | **Module manifest.** `config/modules.php` is the single source of truth for module metadata: folder, key, code, tier, release, provider and `depends_on`. Entitlements, navigation, the dependency arch test (MB-05) and the provider order test all read it. |
| DEC-128 | **URL segments are kebab-case; names and keys are snake_case.** `/app/import-export/import-jobs` has the route name `app.import_export.import_jobs.index` and the permission `import_export.import_job.view`. A Core route macro, `Route::portalResource()`, produces both from one call. |
| DEC-129 | **Class folders inside a module are StudlyCase, including `Database/Migrations`, `Database/Factories` and `Database/Seeders`.** Folders without PHP classes stay lowercase: `resources/`, `lang/`, `routes/`, `config/`. This corrects D3 3.5, whose lowercase `database/factories` would break PSR-4 autoloading on Linux. |
| DEC-130 | **Blade components.** Shared primitives live in the root `resources/views/components/ui/` and render as `x-ui.button`. Module composites live in `modules/{Name}/resources/views/components/{module-kebab}/`; the module registers that folder as an anonymous component path without a prefix, so tags render as `x-fee.voucher-card`, matching D1. An arch test checks that each module only puts components under its own subfolder. |
| DEC-131 | **Module views are grouped by portal:** `{module}::{portal}.{resource}.{view}`, for example `fee::app.vouchers.index` and `fee::guardian.vouchers.show`. PDF templates use `{module}::pdf.{document}`. |
| DEC-132 | **One base provider.** Every module provider extends `Modules\Core\Support\ModuleServiceProvider`. It loads migrations, views, translations, components and route files, registers listeners explicitly, adds morph aliases, and registers commands and registry contributions. Laravel's automatic event discovery is turned off. Policy and factory names are resolved by convention callbacks set once in Core. |
| DEC-133 | **API host and auth.** API v1 (R2) is served on `api.darsgah.app` under the path `/api/v1`, authenticated with Sanctum tokens. Each token belongs to one organization, carries one portal ability (`portal:app`, `portal:guardian`, `portal:student`) or a device ability (`device:punch`), and expires after 90 days unless rotated. Permissions are still enforced by the same policies as the web. |
| DEC-134 | **API value formats.** IDs are ULIDs. Money is `{ "amount": "8500.00", "currency": "PKR" }` with the amount as a decimal string. Instants are ISO 8601 in UTC (`2026-11-05T06:12:44Z`). School-day dates are `YYYY-MM-DD`. Enums are `{ "value": "paid", "label": "Paid" }`, with the label in the request language. |
| DEC-135 | **API contract.** The OpenAPI 3.1 specification is kept by hand in `docs/api/`. A contract test fails if any `api.v1.*` route is missing from the spec or the spec lists a path that has no route. |
| DEC-136 | **Optimistic locking (FR-GEN-010).** Web edit forms carry the record's `updated_at` as a hidden `_version`; a mismatch returns a conflict page (409). API `GET` on a single record returns an `ETag`; `PUT` and `PATCH` require `If-Match` (428 if missing, 412 if stale). |
| DEC-137 | **Navigation is a registry.** Each module contributes its menu items (portal, group, label key, route, permission, module key, sort order) to a `NavigationRegistry` in Core. Menus show an item only when the module is entitled and the user holds the permission. Nothing is hardcoded in layouts. |

### Corrections to earlier deliverables

| Where | Change |
|---|---|
| D3 3.5 module anatomy | `database/migrations`, `database/factories`, `database/seeders` become `Database/Migrations`, `Database/Factories`, `Database/Seeders` (DEC-129). New optional folders are added: `Console/Commands`, `Adapters`, `Contributions`, `Definitions`, `Rules`, `View/Components`, `resources/js`, `config` (8.6). |
| D3 3.23 registries | `NavigationRegistry` is added, owned by Core (DEC-137). |
| D7 7.50 partitioning | `notification_dispatches` is added (it's a DEC-48 log table): partitioned monthly on `created_at`, 12 months hot, archived for 3 years. |

---

## 8.2 Repository Root

One repository, one deployable application.

```
darsgah/
├── .github/
│   └── workflows/
│       ├── ci.yml                 PHP 8.2 and 8.4 matrix: pint, larastan, pest (unit, feature, arch, contract), migrations fresh and upgrade, npm build, composer audit
│       └── deploy.yml             Build, test, tag and deploy (D12)
├── app/
│   └── Providers/
│       ├── AppServiceProvider.php         Framework glue only (strict models, Vite nonce, Carbon defaults)
│       ├── HorizonServiceProvider.php     Horizon dashboard gate (platform super admin only)
│       └── TelescopeServiceProvider.php   Local environment only, never registered in production
├── bootstrap/
│   ├── app.php                    Routing, middleware groups, exception rendering (8.4)
│   ├── providers.php              App providers plus the 34 module providers in manifest order (8.5)
│   └── cache/
├── config/                        Framework and platform configuration (8.7)
├── database/
│   ├── migrations/                Laravel framework tables only: cache, cache_locks, jobs, job_batches, failed_jobs
│   └── seeders/
│       ├── DatabaseSeeder.php     Calls the platform catalog seeders of each module (D7 7.52)
│       └── DemoSeeder.php         Local and staging only; refuses to run in production
├── deploy/                        Containers, web server, process manager, scripts (8.12)
├── docs/                          Blueprint, decisions, API spec, runbooks, agent prompts (8.13)
├── lang/
│   ├── en/                        ui.php, errors.php, validation.php, auth.php, passwords.php, pagination.php
│   └── ur/                        The same files in Urdu
├── modules/                       The 34 modules (8.5, 8.6)
├── public/
│   ├── index.php
│   ├── build/                     Vite output (not committed)
│   ├── robots.txt                 Disallow all; tenant portals are never indexed
│   └── favicon.ico
├── resources/                     Shared CSS, JS, layouts, UI components, error pages (8.9)
├── routes/
│   ├── web.php                    Root redirect and the readiness check only; all portal routes come from modules
│   ├── console.php                The full schedule, in one place (D3 3.11)
│   └── channels.php               Broadcast channel authorization (R2, Reverb)
├── storage/
│   ├── app/private/orgs/{organization_ulid}/{purpose}/{yyyy}/{mm}/{file_ulid}.{ext}
│   ├── app/tmp/                   Upload staging and import scratch, purged hourly
│   ├── framework/
│   └── logs/
├── tests/                         All tests, grouped by module (8.11)
├── .editorconfig
├── .env.example                   Every variable the application reads (8.7)
├── .gitattributes
├── .gitignore
├── artisan
├── composer.json
├── package.json
├── phpstan.neon                   Larastan, level 6 at first, raised to 8 (D3 3.20), phpVersion 80200
├── phpunit.xml                    Pest; failOnDeprecation and failOnWarning on
├── pint.json                      Laravel preset plus declare_strict_types
├── vite.config.js
└── README.md
```

### `composer.json` (relevant parts)

```json
{
    "name": "darsgah/app",
    "type": "project",
    "require": {
        "php": "^8.2",
        "laravel/framework": "^12.0"
    },
    "autoload": {
        "psr-4": {
            "App\\": "app/",
            "Modules\\": "modules/",
            "Database\\Seeders\\": "database/seeders/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "Tests\\": "tests/"
        }
    },
    "scripts": {
        "lint": "pint --test",
        "analyse": "phpstan analyse --memory-limit=1G",
        "test": "pest --parallel",
        "test:arch": "pest --group=arch",
        "check": ["@lint", "@analyse", "@test"]
    },
    "config": {
        "platform-check": true,
        "sort-packages": true
    }
}
```

A single `Modules\\` root covers every module, because every class folder is StudlyCase (DEC-129). For example, `Modules\Fee\Database\Factories\FeeVoucherFactory` resolves to `modules/Fee/Database/Factories/FeeVoucherFactory.php`.

---

## 8.3 What Lives Where

| Concern | Location | Rule |
|---|---|---|
| Business logic | `modules/{Name}/Actions`, `Services` | Never in controllers, models, views or `app/` |
| Framework glue | `app/Providers`, `bootstrap/` | No business rules |
| Shared UI primitives | `resources/views/components/ui` | Generic, with no module knowledge |
| Module UI | `modules/{Name}/resources/views` | Uses `x-ui.*` and its own `x-{module}.*` |
| Layouts | `resources/views/layouts` | One per portal (D3 3.17) |
| Schedule | `routes/console.php` | Every scheduled command, in one file |
| Framework tables | `database/migrations` | Only the five Laravel tables listed in 8.2 |
| Module tables | `modules/{Name}/Database/Migrations` | Only tables that module owns (MB-01) |
| Tests | `tests/` | Never inside `modules/` |
| Technical settings | `config/` and `modules/{Name}/config` | Read `env()` only here (MB-08) |
| Business settings | Database `settings` through `SettingRegistry` | Never in config files or code |

---

## 8.4 `bootstrap/app.php`

```php
<?php

declare(strict_types=1);

use Illuminate\Auth\Middleware\Authenticate;
use Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse;
use Illuminate\Cookie\Middleware\EncryptCookies;
use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;
use Illuminate\Foundation\Http\Middleware\ValidateCsrfToken;
use Illuminate\Routing\Middleware\SubstituteBindings;
use Illuminate\Routing\Middleware\ThrottleRequests;
use Illuminate\Session\Middleware\StartSession;
use Illuminate\View\Middleware\ShareErrorsFromSession;
use Laravel\Sanctum\Http\Middleware\CheckForAnyAbility;
use Modules\Attendance\Http\Middleware as AttendanceMw;
use Modules\Core\Exceptions\ExceptionRenderer;
use Modules\Core\Http\Middleware as CoreMw;
use Modules\Identity\Http\Middleware as IdentityMw;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        channels: __DIR__.'/../routes/channels.php',
        health: '/up',
    )
    ->withEvents(discover: false)
    ->withMiddleware(function (Middleware $middleware): void {
        $middleware->alias([
            'auth' => Authenticate::class,
            'module' => CoreMw\EnsureModuleEnabled::class,
            'portal' => IdentityMw\EnsurePortal::class,
            'ability' => CheckForAnyAbility::class,
            'throttle' => ThrottleRequests::class,
        ]);

        // Tenant host, before login: session and CSRF, no auth.
        $middleware->group('tenant.web', [
            CoreMw\AssignCorrelationId::class,
            CoreMw\SecurityHeaders::class,
            CoreMw\ResolveTenant::class,
            EncryptCookies::class,
            AddQueuedCookiesToResponse::class,
            StartSession::class,
            ShareErrorsFromSession::class,
            ValidateCsrfToken::class,
        ]);

        // Login, OTP, password reset pages.
        $middleware->group('tenant.guest', [
            'tenant.web',
            CoreMw\SetLocale::class,
            'throttle:auth',
            SubstituteBindings::class,
        ]);

        // Public admission form and verification pages (captcha in the Form Request).
        $middleware->group('tenant.public', [
            'tenant.web',
            CoreMw\SetLocale::class,
            'throttle:public',
            SubstituteBindings::class,
        ]);

        foreach (['app', 'guardian', 'student'] as $portal) {
            $middleware->group("portal.{$portal}", [
                'tenant.web',
                'auth',
                CoreMw\EnsureTenantMatchesUser::class,
                IdentityMw\EnsureActiveUser::class,
                IdentityMw\RequireTwoFactor::class,
                IdentityMw\IdleTimeout::class,
                "portal:{$portal}",
                CoreMw\SetBranchContext::class,
                CoreMw\SetLocale::class,
                SubstituteBindings::class,
            ]);
        }

        // Platform host: no tenant, platform users only, 2FA always.
        $middleware->group('portal.platform', [
            CoreMw\AssignCorrelationId::class,
            CoreMw\SecurityHeaders::class,
            EncryptCookies::class,
            AddQueuedCookiesToResponse::class,
            StartSession::class,
            ShareErrorsFromSession::class,
            ValidateCsrfToken::class,
            'auth',
            IdentityMw\EnsurePlatformUser::class,
            IdentityMw\RequireTwoFactor::class,
            IdentityMw\IdleTimeout::class,
            CoreMw\SetLocale::class,
            SubstituteBindings::class,
        ]);

        // Gateway and provider callbacks: no session, no CSRF. Signature checked per provider.
        $middleware->group('webhook', [
            CoreMw\AssignCorrelationId::class,
            CoreMw\ResolveWebhookOrganization::class,
            'throttle:webhooks',
        ]);

        // ZKTeco ADMS firmware paths (DEC-29).
        $middleware->group('adms', [
            CoreMw\AssignCorrelationId::class,
            AttendanceMw\ResolveDeviceBySerial::class,
            'throttle:adms',
        ]);

        // API v1 (R2).
        $middleware->group('api.v1', [
            CoreMw\AssignCorrelationId::class,
            CoreMw\SecurityHeaders::class,
            'auth:sanctum',
            IdentityMw\ResolveTenantFromToken::class,
            IdentityMw\EnsureActiveUser::class,
            CoreMw\SetBranchContext::class,
            CoreMw\SetLocale::class,
            'throttle:api',
            SubstituteBindings::class,
        ]);

        $middleware->priority([
            CoreMw\AssignCorrelationId::class,
            CoreMw\ResolveTenant::class,
            StartSession::class,
            Authenticate::class,
            IdentityMw\ResolveTenantFromToken::class,
            CoreMw\EnsureTenantMatchesUser::class,
            CoreMw\SetBranchContext::class,
            CoreMw\EnsureModuleEnabled::class,
            SubstituteBindings::class,
        ]);
    })
    ->withExceptions(function (Exceptions $exceptions): void {
        ExceptionRenderer::register($exceptions);
    })
    ->create();
```

`SubstituteBindings` always runs after the tenant, user and branch are known, so route model binding goes through the global scopes and another tenant's ULID returns 404 (DEC-27).

### Exception mapping (`Modules\Core\Exceptions\ExceptionRenderer`)

| Exception | HTTP | `code` | Web response |
|---|---|---|---|
| `ValidationException` | 422 | `core.validation_failed` | Back with errors |
| `DomainRuleViolation` (base for module domain exceptions) | 422 | From the exception, e.g. `fee.voucher_not_payable` | Back with a flash error |
| `ModelNotFoundException`, `ModuleNotEnabled`, record outside tenant | 404 | `core.not_found` | 404 page |
| `AuthorizationException` | 403 | `core.forbidden` | 403 page |
| `EditConflict` (DEC-136) | 409 (web), 412 (API `If-Match`) | `core.edit_conflict` | Conflict page with "reload" |
| Missing `If-Match` on API update | 428 | `core.precondition_required` | Not used on web |
| `PeriodLocked` | 423 | `core.period_locked` | Back with a flash error naming the locked period |
| `IdempotencyKeyMismatch` | 422 | `core.idempotency_key_mismatch` | Back with error |
| `TooManyRequests` | 429 | `core.rate_limited` | 429 page with `Retry-After` |
| `MissingTenantContext` (DEC-45) | 500 | `core.server_error` | 500 page; reported at high severity |
| Anything else | 500 | `core.server_error` | 500 page with the correlation id |

Messages come from translation keys, so the same error reads correctly in English and Urdu. Every rendered error includes the correlation id. Passwords, OTPs, tokens, CNIC and B-Form values are never flashed or logged.

---

## 8.5 Module Manifest and Provider Order

### `config/modules.php` (excerpt; full list below)

```php
<?php

declare(strict_types=1);

return [
    'core' => [
        'folder' => 'Core', 'code' => 'CORE', 'tier' => 'core', 'release' => 'R1',
        'provider' => Modules\Core\Providers\CoreServiceProvider::class,
        'depends_on' => [],
    ],
    'fee' => [
        'folder' => 'Fee', 'code' => 'FEE', 'tier' => 'core', 'release' => 'R1',
        'provider' => Modules\Fee\Providers\FeeServiceProvider::class,
        'depends_on' => ['core', 'organization', 'identity', 'settings', 'audit', 'approval',
            'notification', 'document', 'academic', 'student', 'guardian', 'accounting'],
    ],
    'external_exam' => [
        'folder' => 'ExternalExam', 'code' => 'EXT', 'tier' => 'standard', 'release' => 'R1',
        'provider' => Modules\ExternalExam\Providers\ExternalExamServiceProvider::class,
        'depends_on' => ['core', 'academic', 'student', 'guardian', 'fee', 'examination', 'result', 'document'],
    ],
    // ... one entry per module
];
```

`depends_on` copies the dependency map from D2 2.3 as corrected in D3 3.3. The arch test `DependencyMapTest` fails if a module imports a class from a module not in its `depends_on` (MB-05).

### Module list in provider order

`bootstrap/providers.php` lists `App\Providers\AppServiceProvider`, `App\Providers\HorizonServiceProvider`, then these 34 providers in this order. `ProvidersManifestTest` checks that the file matches the manifest.

| # | Folder | Key (permission prefix) | Code | Tier | First release |
|---|---|---|---|---|---|
| 1 | Core | `core` | CORE | Core | R1 |
| 2 | Platform | `platform` | PLT | Core | R1 |
| 3 | Organization | `organization` | ORG | Core | R1 |
| 4 | Identity | `identity` | IAM | Core | R1 |
| 5 | Settings | `settings` | SET | Core | R1 |
| 6 | Audit | `audit` | AUD | Core | R1 |
| 7 | Notification | `notification` | NTF | Core | R1 |
| 8 | Approval | `approval` | APR | Core | R1 |
| 9 | ImportExport | `import_export` | IMX | Core | R1 |
| 10 | Document | `document` | DOC | Core | R1 |
| 11 | HR | `hr` | HR | Standard | R1 |
| 12 | Academic | `academic` | ACD | Core | R1 |
| 13 | Administration | `administration` | ADN | Standard | R1 |
| 14 | Student | `student` | STU | Core | R1 |
| 15 | Guardian | `guardian` | GRD | Core | R1 |
| 16 | Attendance | `attendance` | ATT | Core | R1 |
| 17 | Leave | `leave` | LVE | Core | R1 |
| 18 | Accounting | `accounting` | ACC | Standard | R1 |
| 19 | Fee | `fee` | FEE | Core | R1 |
| 20 | Admission | `admission` | ADM | Standard | R1 |
| 21 | Examination | `examination` | EXM | Core | R1 |
| 22 | Result | `result` | RES | Core | R1 |
| 23 | ExternalExam | `external_exam` | EXT | Standard | R1 |
| 24 | Timetable | `timetable` | TTB | Standard | R1 |
| 25 | Communication | `communication` | COMM | Core | R1 |
| 26 | Learning | `learning` | LMS | Add-on | R2 |
| 27 | Homework | `homework` | HWK | Add-on | R2 |
| 28 | Assessment | `assessment` | ASM | Add-on | R2 |
| 29 | Payroll | `payroll` | PAY | Add-on | R2 |
| 30 | Library | `library` | LIB | Add-on | R3 |
| 31 | Transport | `transport` | TRN | Add-on | R2 (routes, stops, allocation), R3 (vehicles, GPS) |
| 32 | Inventory | `inventory` | INV | Add-on | R3 |
| 33 | Discipline | `discipline` | DSC | Add-on | R2 (incidents, remarks), R3 (counseling) |
| 34 | Reporting | `reporting` | RPT | Core | R1 |

Reporting comes last because it consumes every module's report, metric and widget contributions at runtime.

---

## 8.6 Module Anatomy (complete)

Every module uses the same layout. Folders a module doesn't need are simply absent; nothing outside this list is allowed (`ModuleLayoutTest`).

```
modules/{Name}/
├── Actions/                       One final class per use case, owns the transaction: RecordPayment.php
├── Adapters/                      Implementations of external integration contracts (only Notification, Document, Attendance, Fee, Transport, Core)
│   └── {Provider}/                e.g. Fee/Adapters/JazzCash/JazzCashGateway.php, Fake/FakePaymentGateway.php
├── Approval/                      ApprovalHandler implementations: PaymentReversalHandler.php
├── Console/
│   └── Commands/                  Artisan commands: MarkOverdueVouchersCommand.php (fee:mark-overdue)
├── Contracts/                     Public interfaces other modules may use: FeeClearanceChecker.php
├── Contributions/                 This module's implementations of other modules' registry contracts:
│                                  ClearanceChecks/, VoucherLines/, ProfileTabs/, OnboardingSteps/, SessionCloseChecks/, Purgers/
├── Data/                          Readonly DTOs: PaymentData.php, VoucherLineData.php
├── Database/
│   ├── Factories/                 FeeVoucherFactory.php
│   ├── Migrations/                2026_12_01_070000_create_fee_heads_table.php (timestamps follow D7 7.51 batches)
│   └── Seeders/                   Platform catalog and onboarding seeders owned by this module
├── Definitions/                   Declarations handed to registries:
│   ├── Permissions.php            Permission keys, labels, screens, verbs (D5)
│   ├── Settings.php               Setting keys, types, defaults, scope (organization, institution, branch)
│   ├── Navigation.php             Menu items per portal (DEC-137)
│   ├── NotificationEvents.php     Event, template key, placeholders, recipient resolvers
│   ├── ApprovalTypes.php          Request types and default workflows
│   ├── DocumentTypes.php          Document types and data providers
│   ├── ImportsExports.php         Import and export definitions
│   ├── Reports/                   ReportDefinition classes
│   ├── Metrics/                   MetricDefinition classes
│   └── Widgets/                   DashboardWidget classes
├── Enums/                         Backed string enums with label(): VoucherStatus.php, PaymentMethod.php
├── Events/                        Past tense: PaymentRecorded.php
├── Exceptions/                    Domain exceptions extending Core DomainRuleViolation: VoucherNotPayable.php
├── Exports/                       Export classes used by ImportExport definitions
├── Imports/                       Import classes used by ImportExport definitions
├── Http/
│   ├── Controllers/
│   │   ├── App/                   Staff portal
│   │   ├── Guardian/              Guardian portal
│   │   ├── Student/               Student portal
│   │   ├── Platform/              Platform portal (Platform, Identity, Settings, Audit, Notification, Reporting only)
│   │   ├── Auth/                  Login and recovery screens (Identity only)
│   │   ├── PublicSite/            /admissions and /verify pages (Admission, Document, Result only)
│   │   ├── Webhooks/              Inbound provider callbacks
│   │   ├── Adms/                  ZKTeco ADMS endpoints (Attendance only)
│   │   └── Api/V1/                Versioned API (R2)
│   ├── Middleware/                Module-specific middleware (rare)
│   ├── Requests/                  Form Requests: StorePaymentRequest.php
│   └── Resources/
│       ├── Internal/              JSON for Alpine grids on portal pages (not versioned)
│       └── V1/                    API v1 resources: FeeVoucherResource.php
├── Jobs/                          Imperative: GenerateVoucherChunk.php
├── Listeners/                     Reacting to other modules' events: EndAllocationOnWithdrawal.php
├── Models/                        Eloquent models, singular StudlyCase of the table name: FeeVoucher.php
├── Notifications/                 Notification message classes for the Notification engine
├── Policies/                      {Model}Policy.php
├── Providers/
│   └── {Name}ServiceProvider.php
├── Queries/                       Read models: VoucherListQuery.php, DefaulterQuery.php
├── Rules/                         Custom validation rules: ValidCnic.php
├── Services/                      Reusable domain logic: VoucherCalculator.php
├── View/
│   └── Components/                Class-based Blade components (when data preparation is needed)
├── config/                        Optional technical config merged under the module key
├── lang/
│   ├── en/                        vouchers.php, payments.php, enums.php, permissions.php, notifications.php
│   └── ur/
├── resources/
│   ├── js/
│   │   ├── app/                   Alpine.data components for the staff bundle: feePaymentForm.js
│   │   └── portal/                Alpine.data components for the guardian and student bundle
│   └── views/
│       ├── app/{resource}/        index, show, create, edit, partials
│       ├── guardian/{resource}/
│       ├── student/{resource}/
│       ├── platform/{resource}/
│       ├── public/{resource}/
│       ├── pdf/                   Templates rendered by Gotenberg: voucher.blade.php
│       ├── mail/                  Email bodies
│       └── components/{module-kebab}/   Anonymous composites: voucher-card.blade.php (x-fee.voucher-card)
└── routes/
    ├── app.php
    ├── guardian.php
    ├── student.php
    ├── platform.php
    ├── auth.php
    ├── public.php
    ├── webhooks.php
    ├── adms.php
    └── api.php
```

### Base provider (Core)

```php
<?php

declare(strict_types=1);

namespace Modules\Core\Support;

use Illuminate\Database\Eloquent\Relations\Relation;
use Illuminate\Support\Facades\Blade;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Route;
use Illuminate\Support\ServiceProvider;
use Illuminate\Support\Str;

abstract class ModuleServiceProvider extends ServiceProvider
{
    /** Module key from config/modules.php, e.g. 'fee'. */
    protected string $moduleKey;

    /** @return list<string> route files to load from routes/ */
    abstract protected function routeFiles(): array;

    /** @return array<class-string, list<class-string>> event => listeners */
    protected function listeners(): array
    {
        return [];
    }

    /** @return array<string, class-string> morph alias => model (DEC-98) */
    protected function morphMap(): array
    {
        return [];
    }

    /** @return list<class-string> */
    protected function consoleCommands(): array
    {
        return [];
    }

    /** Register Definitions/* and Contributions/* into registries. */
    protected function contributions(): void
    {
    }

    public function boot(): void
    {
        $path = $this->modulePath();

        $this->loadMigrationsFrom($path.'/Database/Migrations');
        $this->loadViewsFrom($path.'/resources/views', $this->moduleKey);
        $this->loadTranslationsFrom($path.'/lang', $this->moduleKey);
        Blade::anonymousComponentPath($path.'/resources/views/components');

        foreach ($this->listeners() as $event => $listeners) {
            foreach ($listeners as $listener) {
                Event::listen($event, $listener);
            }
        }

        Relation::morphMap($this->morphMap());

        if ($this->app->runningInConsole()) {
            $this->commands($this->consoleCommands());
        }

        $this->contributions();

        if (! $this->app->routesAreCached()) {
            foreach ($this->routeFiles() as $file) {
                ModuleRouteLoader::load($this->moduleKey, $file, $path.'/routes/'.$file.'.php');
            }
        }
    }

    protected function modulePath(): string
    {
        return base_path('modules/'.config("modules.{$this->moduleKey}.folder"));
    }

    protected function urlSegment(): string
    {
        return Str::kebab(Str::camel($this->moduleKey)); // import_export -> import-export
    }
}
```

`CoreServiceProvider` additionally:
- calls `Relation::requireMorphMap()` once all modules have registered their aliases
- sets `Gate::guessPolicyNamesUsing()` (`Modules\X\Models\Y` maps to `Modules\X\Policies\YPolicy`)
- sets `Factory::guessFactoryNamesUsing()` (`Modules\X\Models\Y` maps to `Modules\X\Database\Factories\YFactory`)
- registers the Blueprint macros (DEC-89), route macros (DEC-128), rate limiters (8.15) and `Model::shouldBeStrict()` outside production

### How route files are mounted (`ModuleRouteLoader`)

| File | Host | Middleware group | URL prefix | Name prefix | Module segment added | `module:{key}` added |
|---|---|---|---|---|---|---|
| `app.php` | Tenant | `portal.app` | `/app/{module-kebab}` | `app.{module}.` | Yes | Yes |
| `guardian.php` | Tenant | `portal.guardian` | `/guardian/{module-kebab}` | `guardian.{module}.` | Yes | Yes |
| `student.php` | Tenant | `portal.student` | `/student/{module-kebab}` | `student.{module}.` | Yes | Yes |
| `platform.php` | `platform.` | `portal.platform` | `/platform/{module-kebab}` | `platform.{module}.` | Yes | No |
| `auth.php` | Tenant and `platform.` | `tenant.guest` (tenant) or platform guest group | none (`/login`, `/forgot-password`, `/otp`) | `auth.` | No | No |
| `public.php` | Tenant | `tenant.public` | `/admissions`, `/verify` (set in file) | `public.{module}.` | No | Yes |
| `webhooks.php` | `hooks.` | `webhook` | `/webhooks` (file adds `/{provider}/{organization_uuid}`) | `webhooks.{module}.` | No | Yes (after organization is resolved) |
| `adms.php` | `adms.` | `adms` | `/iclock` | `adms.` | No | Yes |
| `api.php` | `api.` | `api.v1` | `/api/v1/{module-kebab}` | `api.v1.{module}.` | Yes | Yes |

Core-tier modules (D1 1.15) pass `EnsureModuleEnabled` automatically, but the middleware is still attached so tiers can change without code changes.

### Example route file (`modules/Fee/routes/app.php`)

```php
<?php

declare(strict_types=1);

use Illuminate\Support\Facades\Route;
use Modules\Fee\Http\Controllers\App;

// Mounted at /app/fee, names app.fee.*, middleware portal.app + module:fee

Route::get('payments/voucher-lookup', [App\PaymentController::class, 'lookup'])
    ->name('payments.voucher_lookup');

Route::portalResource('payments', App\PaymentController::class)
    ->only(['index', 'create', 'store', 'show']);

Route::post('vouchers/generate', [App\VoucherGenerationController::class, 'store'])
    ->name('vouchers.generate');

Route::portalResource('vouchers', App\VoucherController::class)
    ->only(['index', 'show']);

Route::portalResource('cashier-sessions', App\CashierSessionController::class)
    ->only(['index', 'store', 'show', 'update']);
```

`Route::portalResource('cashier-sessions', ...)` creates `/app/fee/cashier-sessions/{cashier_session}` with the names `app.fee.cashier_sessions.index` and so on. Models bind by `uuid` through `HasPublicUlid::getRouteKeyName()`.

---

## 8.7 Configuration

| File | Contents |
|---|---|
| `app.php` | Name `Darsgah`, timezone `UTC` (display timezone comes from organization settings), locale `en`, fallback `en` |
| `auth.php` | Guard `web` (session), provider `users`; password broker replaced by Identity's organization-aware broker (DEC-96) |
| `cache.php` | Redis in staging and production; key prefix per environment |
| `cors.php` | Paths `api/*` only; no allowed browser origins (API clients are apps, not browsers) |
| `darsgah.php` | Root domain, hosts (`platform`, `adms`, `hooks`, `api`), reserved slugs (D4 4.7), product name, support email |
| `database.php` | MySQL 8.4, `utf8mb4`, `utf8mb4_0900_ai_ci`, strict mode, `timezone => '+00:00'`; Redis connections `default`, `cache`, `queue`, `locks` |
| `filesystems.php` | Disks `private` (default), `tmp`, `archive` (partition archives), `public` (built assets only), optional `s3` |
| `fortify.php` | Features: login, password reset, two-factor with confirmation, password confirmation; views off (custom Blade views) |
| `horizon.php` | One supervisor per queue from D3 3.11, with the same workers, timeouts and tries |
| `integrations.php` | Adapter class per provider (SMS aggregators, WhatsApp, JazzCash, Easypaisa, 1Bill, GPS); timeouts; sandbox switches |
| `logging.php` | JSON channel; processors add correlation id, organization id and user id; a redaction processor removes passwords, OTPs, tokens, CNIC and B-Form values, card data and `Authorization` headers |
| `mail.php` | SMTP; local uses Mailpit |
| `modules.php` | The module manifest (DEC-127) |
| `pdf.php` | Gotenberg URL, timeouts, paper presets (A4, Legal, A4 landscape 3-copy, CR80) |
| `queue.php` | Redis; queue names from D3 3.11; `after_commit => true` (DEC-41) |
| `sanctum.php` | Stateful domains empty (no SPA cookies); expiration handled per token (DEC-133) |
| `security.php` | CSP directives and nonce, HSTS, idle timeout defaults per role group, password rules, 2FA-required permission list source |
| `services.php` | Sentry, ClamAV, Reverb (R2) |
| `session.php` | Driver `database`, `domain => null` (host-only, DEC-44), `secure => true`, `same_site => 'lax'`, `http_only => true` |

### `.env.example` (application keys)

```
APP_NAME=Darsgah
APP_ENV=local
APP_KEY=
APP_DEBUG=true
APP_URL=http://demo.darsgah.test

DARSGAH_ROOT_DOMAIN=darsgah.test
DARSGAH_PLATFORM_HOST=platform.darsgah.test
DARSGAH_ADMS_HOST=adms.darsgah.test
DARSGAH_HOOKS_HOST=hooks.darsgah.test
DARSGAH_API_HOST=api.darsgah.test
LOOKUP_HASH_PEPPER=

DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=darsgah
DB_USERNAME=darsgah
DB_PASSWORD=

REDIS_HOST=redis
CACHE_STORE=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=database
SESSION_DOMAIN=null
SESSION_SECURE_COOKIE=true

FILESYSTEM_DISK=private
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_DEFAULT_REGION=
AWS_BUCKET=
AWS_ENDPOINT=

GOTENBERG_URL=http://gotenberg:3000
CLAMAV_ENABLED=false
CLAMAV_HOST=clamav

MAIL_MAILER=smtp
MAIL_HOST=mailpit
MAIL_PORT=1025
MAIL_FROM_ADDRESS=no-reply@darsgah.test

SENTRY_LARAVEL_DSN=
TELESCOPE_ENABLED=false
```

Provider credentials (SMS, gateways) are never in `.env`; each organization stores its own, encrypted in the database (D3 3.15).

---

## 8.8 Core Module (full tree)

```
modules/Core/
├── Adapters/
│   └── VirusScanning/              ClamAvScanner.php, NullScanner.php
├── Casts/                          MoneyCast.php, UlidCast.php
├── Concerns/                       HasPublicUlid.php, HasAuditColumns.php, HasActiveMarker.php
├── Console/Commands/
│   ├── PurgeIdempotencyKeysCommand.php     core:purge-idempotency-keys
│   ├── IntegrityCheckCommand.php           core:integrity:check (DEC-126)
│   ├── MaintainPartitionsCommand.php       core:partitions:maintain (D7 7.50)
│   └── ArchivePartitionsCommand.php        core:partitions:archive
├── Contracts/
│   ├── EntitlementChecker.php
│   ├── VirusScanner.php
│   ├── IntegrityCheck.php                  Modules contribute cache and soft-reference checks
│   ├── Navigation/NavigationItem.php, NavigationRegistry.php
│   └── Reporting/ReportDefinition.php, MetricDefinition.php, DashboardWidget.php,
│                 ReportRegistry.php, MetricRegistry.php, DashboardWidgetRegistry.php
├── Data/                           PageRequest.php, DateRange.php, BulkOperationResult.php
├── Database/
│   ├── Migrations/                 numbering_sequences, stored_files, file_attachments, period_locks,
│   │                               idempotency_keys, webhook_receipts, bulk_operations
│   ├── Factories/
│   └── Schema/
│       ├── BlueprintMacros.php     tenantColumns, branchColumns, auditColumns, softDeletesWithUser,
│       │                           publicUlid, money, tenantForeign, activeMarker, keyColumn
│       └── PartitionedTable.php    Helper that writes RANGE COLUMNS partition DDL
├── Enums/                          PeriodLockModule.php, BulkOperationStatus.php, FileScanStatus.php, FilePurpose.php
├── Events/                         BulkOperationCompleted.php, PeriodLocked.php, PeriodUnlocked.php
├── Exceptions/
│   ├── DomainRuleViolation.php     Base: errorCode(), translation key, context
│   ├── ExceptionRenderer.php       Mapping in 8.4
│   ├── MissingTenantContext.php
│   ├── ModuleNotEnabled.php
│   ├── PeriodLocked.php
│   ├── EditConflict.php
│   └── IdempotencyKeyMismatch.php
├── Http/
│   ├── Controllers/
│   │   ├── App/FileController.php, BulkOperationController.php, PeriodLockController.php
│   │   └── HealthController.php    /health readiness: database, Redis, Gotenberg, queue lag
│   ├── Middleware/
│   │   ├── AssignCorrelationId.php
│   │   ├── SecurityHeaders.php
│   │   ├── ResolveTenant.php
│   │   ├── ResolveWebhookOrganization.php
│   │   ├── EnsureTenantMatchesUser.php
│   │   ├── SetBranchContext.php    Session on web, X-Branch-Id header on API
│   │   ├── EnsureModuleEnabled.php
│   │   ├── SetLocale.php
│   │   └── VerifyWebhookSignature.php
│   ├── Requests/
│   │   └── VersionedFormRequest.php      Adds _version check (DEC-136)
│   └── Resources/
│       ├── ApiResource.php         Envelope, money and enum serializers
│       └── ApiCollection.php       Page and cursor pagination meta
├── Jobs/
│   ├── Middleware/TenantAware.php
│   ├── PurgeIdempotencyKeys.php
│   ├── PurgeTempFiles.php
│   ├── CheckOrganizationIntegrity.php
│   └── ArchivePartition.php
├── Models/                         NumberingSequence, StoredFile, FileAttachment, PeriodLock,
│                                   IdempotencyKey, WebhookReceipt, BulkOperation
├── Money/                          Money.php (integer minor units), Currency.php
├── Policies/                       PeriodLockPolicy.php, BulkOperationPolicy.php
├── Providers/CoreServiceProvider.php
├── Queries/
│   ├── ApiQuery.php                Parses filter[], sort, include, per_page against an allowlist
│   └── BulkOperationProgressQuery.php
├── Routing/RouteMacros.php         portalResource()
├── Services/
│   ├── NumberingService.php
│   ├── PeriodLockService.php
│   ├── FileStorageService.php      Paths under orgs/{organization_ulid}/..., signed URLs
│   ├── IdempotencyGuard.php
│   ├── ImageProcessor.php
│   └── PhoneNumberNormalizer.php   E.164 through libphonenumber
├── Support/
│   ├── ModuleServiceProvider.php
│   ├── ModuleRouteLoader.php
│   └── Registries/                 NavigationRegistryImpl.php, ReportRegistryImpl.php (Reporting fills them at runtime)
├── Tenancy/
│   ├── TenantContext.php           Fields from D4 4.8
│   ├── Tenancy.php                 actAs(), bypass() (audited), current()
│   ├── OrganizationScope.php
│   ├── AllowedBranchesScope.php
│   ├── BelongsToOrganization.php
│   └── BelongsToBranch.php
├── lang/en, lang/ur
├── resources/views/app/            period-locks, bulk-operations
└── routes/app.php
```

---

## 8.9 Shared Frontend (`resources/`)

```
resources/
├── css/
│   ├── tokens.css                  @theme tokens with the D9 names and HEX values
│   ├── app.css                     Staff and portal styles
│   ├── print.css                   Browser print views
│   └── pdf.css                     Inlined into Gotenberg templates
├── fonts/                          Latin UI font and Noto Nastaliq Urdu (D9), self-hosted
├── js/
│   ├── app.js                      Staff bundle
│   ├── portal.js                   Guardian and student bundle (smaller)
│   ├── bootstrap/
│   │   ├── alpine.js               @alpinejs/csp start, @alpinejs/focus
│   │   ├── http.js                 fetch wrapper: CSRF, Idempotency-Key, X-Correlation-Id, error envelope
│   │   └── autosave.js             Debounced autosave with conflict handling (DEC-136)
│   ├── components/                 Shared Alpine.data: dataTable, modal, drawer, dropdown, tabs, toast,
│   │                               confirmDialog, fileUpload, datePicker, moneyInput, branchSwitcher,
│   │                               portalSwitcher, notificationBell, localeSwitcher, searchSelect
│   └── charts/dashboardCharts.js   Chart.js, imported only by dashboard pages
└── views/
    ├── components/ui/              alert, avatar, badge, breadcrumb, button, card, checkbox, confirm-dialog,
    │                               data-table, date-picker, drawer, dropdown, empty-state, field, file-upload,
    │                               icon, input, label, loading, modal, money, page-header, pagination, radio,
    │                               search-select, select, skeleton, stat-tile, status-pill, tabs, table,
    │                               textarea, toast, toggle
    ├── layouts/                    app, guardian, student, platform, public, print, auth
    ├── partials/                   navigation (rendered from NavigationRegistry), topbar, flash, footer
    ├── errors/                     403, 404, 409, 419, 423, 429, 500, 503
    └── vendor/mail/                Branded email layout
```

### `resources/css/app.css`

```css
@import "tailwindcss";
@import "./tokens.css";

@source "../../modules/**/resources/views/**/*.blade.php";
@source "../../modules/**/resources/js/**/*.js";
```

Tailwind v4 already scans the project, but the explicit `@source` lines make the module folders visible to anyone reading the file.

### `resources/js/app.js`

```js
import './bootstrap/alpine';
import './bootstrap/http';
import './components';

// Every module's staff components, e.g. modules/Attendance/resources/js/app/attendanceMarkingGrid.js
import.meta.glob('../../modules/*/resources/js/app/**/*.js', { eager: true });
```

Each module file registers one component with `Alpine.data('{moduleCamel}{Component}', ...)`, for example `attendanceMarkingGrid` or `feePaymentForm`, so names never collide. `portal.js` does the same with `resources/js/portal/**`.

### `vite.config.js` inputs

`resources/css/app.css`, `resources/css/print.css`, `resources/css/pdf.css`, `resources/js/app.js`, `resources/js/portal.js`. Node is used only on CI and developer machines, never on servers.

---

## 8.10 Per-Module Structure Map

What each module adds beyond the standard anatomy.

| # | Module | Route files | View portals | Notable extra folders and files |
|---|---|---|---|---|
| 1 | Core | app | app | Tenancy, Money, Database/Schema, Http/Middleware, Routing (8.8) |
| 2 | Platform | platform, app | platform, app | Contributions/Purgers (TenantDataPurger registry, DEC-51), Jobs/BuildTenantExport |
| 3 | Organization | app | app | Support/OnboardingStepRegistry, Http/Controllers/App/OnboardingWizardController |
| 4 | Identity | auth, app, guardian, student, platform, api | app, guardian, student, platform, auth | Fortify/ (AuthenticateUser, ResetUserPassword, custom pipeline), Http/Middleware (EnsurePortal, EnsureActiveUser, RequireTwoFactor, IdleTimeout, EnsurePlatformUser, ResolveTenantFromToken), Services/PermissionResolver, Support/PermissionRegistry |
| 5 | Settings | app, platform | app, platform | Support/SettingRegistry, Translation/OverrideTranslationLoader |
| 6 | Audit | app, platform | app, platform | Concerns/Auditable, Services/AuditWriter, Masking/ |
| 7 | Notification | app, guardian, student, platform, webhooks | app, guardian, student, platform | Adapters/Sms, Adapters/WhatsApp, Adapters/Email, Support/NotificationEventRegistry, RecipientResolverRegistry |
| 8 | Approval | app, guardian, api | app, guardian | Support/ApprovalTypeRegistry, ApproverResolverRegistry, Services/WorkflowEngine |
| 9 | ImportExport | app | app | Support/ImportRegistry, ExportRegistry, Services/ImportValidator |
| 10 | Document | app, public | app, public, pdf | Adapters/Pdf (GotenbergPdfRenderer, FakePdfRenderer), Support/DocumentTypeRegistry, resources/views/pdf/base |
| 11 | HR | app | app | Support/EmployeeProfileTabRegistry, Contributions/ProfileTabs |
| 12 | Academic | app, api | app | Support/SessionCloseCheckRegistry, Services/NewSessionWizard |
| 13 | Administration | app, guardian, student | app, guardian, student | Contracts/CalendarQuery |
| 14 | Student | app, student, api | app, student, pdf | Support/StudentProfileTabRegistry, ClearanceCheckRegistry, Contracts/StudentEnroller, PromotionEligibilityProvider |
| 15 | Guardian | app, guardian, api | app, guardian | Contributions/ProfileTabs (guardians tab) |
| 16 | Attendance | app, guardian, student, adms, webhooks, api | app, guardian, student | Adapters/Devices (ZktecoAdmsAdapter, SignedWebhookAdapter, CsvPunchFileAdapter), Http/Controllers/Adms, Http/Middleware/ResolveDeviceBySerial, Contracts/AttendanceLeaveWriter |
| 17 | Leave | app, guardian, api | app, guardian | Services/LeaveDayCalculator |
| 18 | Accounting | app | app, pdf | Services/PostingService, Contracts/JournalPoster |
| 19 | Fee | app, guardian, student, webhooks, api | app, guardian, student, pdf | Adapters/Gateways (JazzCash, Easypaisa, OneBill, Fake), Contracts/VoucherLineContributor, Support/VoucherLineRegistry |
| 20 | Admission | app, public | app, public | Http/Controllers/PublicSite/ApplicationFormController |
| 21 | Examination | app, guardian, student | app, guardian, student, pdf | Services/MarksValidator |
| 22 | Result | app, guardian, student, public, api | app, guardian, student, public, pdf | Services/ResultCalculator, Contributions/PromotionEligibility |
| 23 | ExternalExam | app, guardian, student | app, guardian, student, pdf | Services/IbccEquivalenceCalculator, Imports/ResultFileImport |
| 24 | Timetable | app, guardian, student, api | app, guardian, student, pdf | Services/TimetableConflictDetector |
| 25 | Communication | app, guardian, student, api | app, guardian, student | Services/HtmlSanitizer usage, Support/AudienceResolver |
| 26 | Learning | app, student, guardian, api | app, student, guardian | Services/ProgressTracker |
| 27 | Homework | app, student, guardian, api | app, student, guardian | Contributions/DiaryLink |
| 28 | Assessment | app, student, guardian, api | app, student, guardian | Services/AutoGrader, Console/Commands/AutoSubmitAttemptsCommand |
| 29 | Payroll | app | app, pdf | Services/PayrollCalculator, Services/TaxSlabCalculator, Exports/BankUploadFile |
| 30 | Library | app, student, guardian | app, student, guardian | Contributions/ClearanceChecks, Contributions/VoucherLines |
| 31 | Transport | app, guardian, student, webhooks, api | app, guardian, student | Adapters/Gps, Contributions/VoucherLines, Contributions/ClearanceChecks |
| 32 | Inventory | app | app | Services/StockLedger |
| 33 | Discipline | app, guardian, student | app, guardian, student | Policies/CounselingSessionPolicy (restricted, DEC-122) |
| 34 | Reporting | app, guardian, student, platform | app, guardian, student, platform | Services/ReportRunner, SnapshotComputer, DashboardComposer |

---

## 8.11 Reference Module: Fee (expanded)

```
modules/Fee/
├── Actions/
│   ├── GenerateVouchers.php            Dispatches GenerateVoucherChunk jobs, creates the bulk operation
│   ├── GenerateStudentVoucher.php
│   ├── IssueAdHocVoucher.php
│   ├── CancelVoucher.php
│   ├── RecordPayment.php
│   ├── ReversePayment.php              Called by PaymentReversalHandler after approval (DEC-113)
│   ├── MarkChequeCleared.php
│   ├── BounceCheque.php
│   ├── ApplyAdvanceCredit.php
│   ├── ApplyLateFees.php
│   ├── OpenCashierSession.php
│   ├── CloseCashierSession.php
│   ├── IssueRefund.php
│   ├── InitiateOnlinePayment.php       R2
│   └── ConfirmGatewayPayment.php       R2, after provider verification
├── Adapters/
│   ├── JazzCash/JazzCashGateway.php
│   ├── Easypaisa/EasypaisaGateway.php
│   ├── OneBill/OneBillProvider.php
│   └── Fake/FakePaymentGateway.php, FakeBillPaymentProvider.php
├── Approval/
│   ├── FeeConcessionHandler.php
│   ├── ScholarshipHandler.php
│   ├── PaymentReversalHandler.php
│   ├── FeeRefundHandler.php
│   └── CashVarianceHandler.php
├── Console/Commands/
│   ├── MarkOverdueVouchersCommand.php      fee:mark-overdue
│   ├── ApplyLateFeesCommand.php            fee:apply-late-fees
│   ├── StruckOffCandidatesCommand.php      fee:struck-off-candidates
│   └── SendFeeRemindersCommand.php         fee:reminders
├── Contracts/
│   ├── FeeClearanceChecker.php
│   ├── AdHocVoucherIssuer.php
│   ├── FeeBalanceQuery.php
│   ├── VoucherLineContributor.php
│   ├── PaymentGateway.php
│   └── BillPaymentProvider.php
├── Contributions/
│   ├── ClearanceChecks/FeeDuesClearanceCheck.php
│   ├── ProfileTabs/StudentFeeTab.php
│   ├── SessionCloseChecks/FeeCarryForwardCheck.php
│   └── OnboardingSteps/DefaultFeeHeadsStep.php
├── Data/                               PaymentData.php, VoucherLineData.php, AllocationResult.php, GatewayInitiation.php
├── Database/
│   ├── Factories/                      FeeHeadFactory.php, FeeVoucherFactory.php, FeePaymentFactory.php, ...
│   ├── Migrations/                     One file per table in D7 7.26, in batch M07 (gateway tables in M12)
│   └── Seeders/                        FeeHeadTemplateSeeder.php
├── Definitions/
│   ├── Permissions.php                 fee.* keys from D5
│   ├── Settings.php                    fee.late_fee_rule, fee.voucher_validity_days, fee.allocation_order, ...
│   ├── Navigation.php
│   ├── NotificationEvents.php          VoucherGenerated, PaymentRecorded, fee reminders
│   ├── ApprovalTypes.php               concession, scholarship, payment reversal, refund, cash variance
│   ├── DocumentTypes.php               fee_voucher, fee_receipt
│   ├── ImportsExports.php              Opening balances import, defaulters export
│   ├── Reports/                        DailyCollectionReport.php, DefaulterReport.php, HeadWiseIncomeReport.php
│   ├── Metrics/                        TodayCollectionMetric.php, OutstandingMetric.php
│   └── Widgets/                        CollectionTodayWidget.php, DefaultersWidget.php
├── Enums/                              VoucherStatus.php, VoucherType.php, PaymentMethod.php, PaymentStatus.php,
│                                       FeeHeadType.php, VoucherLineType.php, ChequeStatus.php, CashierSessionStatus.php
├── Events/                             VoucherGenerated.php, VouchersBatchGenerated.php, PaymentRecorded.php, ...
├── Exceptions/                         VoucherNotPayable.php, OverpaymentNotAllowed.php, CashierSessionNotOpen.php
├── Exports/                            DefaultersExport.php
├── Imports/                            OpeningBalancesImport.php
├── Http/
│   ├── Controllers/
│   │   ├── App/                        FeeHeadController, FeeStructureController, FeeDiscountRuleController,
│   │   │                               ConcessionController, ScholarshipController, VoucherController,
│   │   │                               VoucherGenerationController, VoucherPrintController, PaymentController,
│   │   │                               ChequeController, BankChallanEntryController, CashierSessionController,
│   │   │                               RefundController, FeeLedgerController, DefaulterController, FeeReportController
│   │   ├── Guardian/                   VoucherController, PaymentHistoryController, OnlinePaymentController
│   │   ├── Student/                    VoucherController
│   │   ├── Webhooks/                   JazzCashCallbackController, EasypaisaCallbackController, OneBillController
│   │   └── Api/V1/                     VoucherController, PaymentController, OnlinePaymentController
│   ├── Requests/                       StoreFeeStructureRequest, GenerateVouchersRequest, StorePaymentRequest,
│   │                                   ReversePaymentRequest, CloseCashierSessionRequest, StoreRefundRequest, ...
│   └── Resources/
│       ├── Internal/                   VoucherLookupResource.php (cashier screen)
│       └── V1/                         FeeVoucherResource.php, FeeVoucherItemResource.php, FeePaymentResource.php
├── Jobs/                               GenerateVoucherChunk.php, ProcessGatewayCallback.php, RecalculateStudentFeeAccount.php
├── Listeners/                          RecalculateAccountOnPayment.php, FreezeVouchersOnWithdrawal.php
├── Models/                             FeeHead, FeeTaxRule, FeeStructure, FeeStructureItem, FeeDiscountRule,
│                                       StudentFeeConcession, Scholarship, InstallmentPlan, InstallmentPlanLine,
│                                       FeeVoucher, FeeVoucherItem, FeePayment, FeePaymentAllocation,
│                                       FeeAdvanceMovement, StudentFeeAccount, FeeRefund, CashierSession,
│                                       PaymentGatewayAccount, PaymentGatewayTransaction
├── Policies/                           FeeStructurePolicy, FeeVoucherPolicy, FeePaymentPolicy, CashierSessionPolicy, ConcessionPolicy
├── Providers/FeeServiceProvider.php
├── Queries/                            VoucherListQuery, PaymentListQuery, DefaulterQuery, DailyCollectionQuery, StudentLedgerQuery
├── Services/                           VoucherCalculator, DiscountRuleEngine, LateFeeCalculator, AllocationService, FeePostingRules
├── lang/en, lang/ur                    vouchers.php, payments.php, structures.php, enums.php, permissions.php
├── resources/
│   ├── js/app/                         feePaymentForm.js, voucherGenerationProgress.js
│   ├── js/portal/                      onlinePaymentStart.js
│   └── views/
│       ├── app/vouchers/, app/payments/, app/structures/, app/cashier-sessions/, ...
│       ├── guardian/vouchers/, guardian/payments/
│       ├── student/vouchers/
│       ├── pdf/voucher.blade.php, pdf/receipt.blade.php
│       └── components/fee/voucher-card.blade.php, amount-breakdown.blade.php
└── routes/app.php, guardian.php, student.php, webhooks.php, api.php
```

### One request through the folders: recording a cash payment

```
POST /app/fee/payments                          modules/Fee/routes/app.php
  -> middleware portal.app, module:fee          bootstrap/app.php, Core and Identity middleware
  -> StorePaymentRequest                        Http/Requests: authorize() -> FeePaymentPolicy::create
                                                rules(), prepareForValidation() (amount, idempotency key)
  -> App\PaymentController@store                Http/Controllers/App: builds PaymentData, calls the action
  -> RecordPayment::handle(PaymentData)         Actions: DB::transaction {
       IdempotencyGuard                           Core/Services
       PeriodLockService::assertOpen              Core/Services
       lock FeeVoucher rows, AllocationService    Services
       NumberingService (receipt no)              Core/Services
       FeePostingRules -> JournalPoster           Services -> Accounting/Contracts (uq_jen_source)
       AuditWriter (fee.payment.recorded)         Audit
     } then PaymentRecorded after commit          Events (DEC-41)
  -> listeners                                  Fee: RecalculateAccountOnPayment
                                                Notification: receipt SMS to guardian
                                                Reporting: today's collection snapshot refresh
  -> redirect to app.fee.payments.show          receipt print button (Document: fee_receipt)
```

---

## 8.12 Deployment and Local Environment Folders

```
deploy/
├── docker/
│   ├── php/Dockerfile              PHP-FPM 8.4 (8.2 image kept for the CI matrix), extensions: pdo_mysql, redis, intl, gd, bcmath, zip, pcntl
│   ├── php/php.ini, php/fpm.conf
│   ├── nginx/default.conf          Tenant, platform, hooks and api server blocks
│   ├── nginx/adms.conf             HTTP-only server block for /iclock/* (DEC-29)
│   └── compose.yaml                Local: app, nginx, mysql 8.4, redis 7, gotenberg 8, mailpit, clamav (optional)
├── supervisor/
│   ├── horizon.conf
│   └── scheduler.conf              php artisan schedule:work
└── scripts/
    ├── deploy.sh                   Maintenance-free deploy steps (D12)
    └── post-deploy.sh              migrate --force, iam:permissions:sync, config/route/view/event cache, horizon:terminate
```

D12 defines the production topology; these folders only fix where the files live.

---

## 8.13 Documentation Folders

```
docs/
├── blueprint/                      D01 to D15 (copies of the project documents)
├── decisions/                      decisions.md: DEC-01 onward, one row each, with the deliverable that set it
├── api/
│   ├── openapi-v1.yaml             Root document
│   └── v1/paths/{module}.yaml      One file per module, referenced from the root
├── runbooks/                       restore.md, tenant-export.md, tenant-extraction.md, partitions.md, incident.md
└── agent/
    ├── phase-0-master-context.md   The Phase 0 Master Context Prompt (D14)
    ├── phases/                     One file per phase prompt
    └── checklists/                 Verification checklists (D15)
```

---

## 8.14 Tests

```
tests/
├── Pest.php                        Uses TestCase, RefreshDatabase on MySQL (DEC-39), groups
├── TestCase.php
├── Support/
│   ├── InTenant.php                inTenant($organization, fn () => ...)
│   ├── TwoTenantFixture.php        Builds organization A and B with mirrored data
│   ├── ActsAsRole.php              actingAsRole('cashier', branch: $branch)
│   ├── FakeAdapters.php            Binds every Fake* adapter
│   └── Assertions/                 assertAudited(), assertNotVisibleToOtherTenant(), assertJournalBalanced()
├── Architecture/
│   ├── ModuleBoundariesTest.php    MB-01 to MB-09
│   ├── DependencyMapTest.php       Imports respect config/modules.php depends_on
│   ├── ModuleLayoutTest.php        Only the folders in 8.6; components under the module's own subfolder
│   ├── ProvidersManifestTest.php   bootstrap/providers.php matches the manifest
│   ├── TenancyTraitsTest.php       Models with organization_id use BelongsToOrganization
│   ├── NamingTest.php              Actions final, enums backed, events past tense, policies match models
│   ├── ForbiddenCodeTest.php       No env() outside config, no dd/dump, no DB facade in controllers, no #[\Override]
│   ├── MigrationOrderTest.php      No migration references a later table (D7 7.51)
│   └── RouteConventionsTest.php    URL kebab-case, names snake_case, every app route has a permission check
├── Feature/
│   └── {Module}/                   e.g. Feature/Fee/RecordPaymentTest.php, Feature/Fee/Isolation/FeeIsolationTest.php
├── Unit/
│   └── {Module}/                   e.g. Unit/Fee/VoucherCalculatorTest.php
├── Contract/
│   └── Api/OpenApiCoverageTest.php DEC-135
└── Browser/                        After UG-1 (Pest 4): login, attendance marking, payment, result publish
```

PHP features newer than 8.2 are caught three ways: the CI job on PHP 8.2 fails to parse them, Larastan runs with `phpVersion: 80200`, and `ForbiddenCodeTest` catches `#[\Override]`, which PHP 8.2 would otherwise ignore silently. The PHP 8.4 job fails on deprecations (for example implicit nullable parameters) because `failOnDeprecation` is on.

---

## 8.15 API Architecture

### Scope by release

| Release | Surface | Consumers |
|---|---|---|
| R1 | Internal JSON endpoints under portal prefixes; webhooks; ADMS | The application's own Alpine grids; devices; gateways |
| R2 | API v1 on `api.darsgah.app/api/v1` | Future mobile apps for guardians, students and teachers; the on-premise attendance sync agent |
| R3 | Partner read scopes on API v1 | Approved third parties, per organization |

### Layering

The API uses the same Actions, Queries and Policies as the web. Only the edges differ:

```
api.darsgah.app/api/v1/fee/vouchers/{voucher}
  -> api.v1 middleware group (token -> user -> organization, branch header, locale, rate limit)
  -> ability:portal:guardian,portal:app   (route group in modules/Fee/routes/api.php)
  -> Api\V1\VoucherController@show
  -> FeeVoucherPolicy::view (same visibility rules as the guardian portal)
  -> VoucherListQuery / model load through scopes
  -> V1\FeeVoucherResource (envelope, money, enums, ETag)
```

No business rule exists only in the API. An API controller that needs logic the web lacks gets a new Action that the web can also use.

### Authentication and tenancy

| Topic | Rule |
|---|---|
| Token issue | `POST /api/v1/identity/tokens` with identifier, password, `device_name` and portal. Guardians on first login use `POST /api/v1/identity/otp-requests` then `POST /api/v1/identity/otp-verifications` (DEC-10). |
| Token storage | Sanctum stores only a SHA-256 hash. `personal_access_tokens.organization_id` is set at issue (D7 Part 1). |
| Abilities | Exactly one of `portal:app`, `portal:guardian`, `portal:student`; device tokens carry `device:punch`. Abilities pick the portal; permissions and visibility still come from role assignments. |
| Device tokens | Issued by an IT admin to an `attendance_devices` row (tokenable morph `attendance_device`) for the R2 sync agent. They can call only the punch endpoint. |
| Lifetime | 90 days. `POST /api/v1/identity/tokens/rotations` swaps the current token for a new one. Password change, deactivation or role removal revokes all tokens. |
| Tenant | Resolved from the token owner (`ResolveTenantFromToken`). A request can never name an organization. |
| Branch | Staff with several branches send `X-Branch-Id: {branch_ulid}`; it must be in the allowed branches, otherwise 403. Without it, the user's preferred branch is used. |
| 2FA | Tokens for users whose roles require 2FA (DEC-63) can only be issued after the TOTP step in the token request. |
| Transport | HTTPS only; HSTS; no CORS origins. |

### URLs and methods

| Rule | Example |
|---|---|
| `/api/v1/{module-kebab}/{resource-kebab}` | `/api/v1/fee/vouchers` |
| Nested only one level, for ownership | `/api/v1/assessment/quiz-attempts/{attempt}/answers` |
| Path parameters are ULIDs | `/api/v1/fee/vouchers/01JD8Z3K6Q2M7V4T9X1B5C8N0R` |
| Route names `api.v1.{module}.{resource}.{action}` | `api.v1.fee.vouchers.show` |
| `GET` list and read, `POST` create or command, `PUT` full replace, `PATCH` partial update, `DELETE` only for drafts | |
| Commands are nouns created with `POST` | `POST /api/v1/approval/requests/{request}/decisions` |

### Status codes

| Code | When |
|---|---|
| 200 | Read or update succeeded |
| 201 | Created; `Location` header points at the new resource |
| 202 | Bulk work queued; body carries the `bulk_operation` id to poll |
| 204 | Delete or acknowledgement with no body |
| 400 | Malformed query (unknown filter or sort field): `core.invalid_query` |
| 401 | Missing, expired or revoked token |
| 403 | Authenticated but not allowed |
| 404 | Not found, outside the tenant, or module not enabled |
| 409 | Business conflict, for example a voucher already paid |
| 412 | `If-Match` does not match the current `ETag` |
| 422 | Validation or domain rule failed |
| 423 | Period locked |
| 428 | `If-Match` missing on `PUT` or `PATCH` |
| 429 | Rate limited; `Retry-After` set |
| 500, 503 | Server error or maintenance; body carries the correlation id |

### Response envelope

Success, single record:

```json
{
  "data": {
    "id": "01JD8Z3K6Q2M7V4T9X1B5C8N0R",
    "voucher_no": "V-2026-11-000412",
    "student": {
      "id": "01JD7Y2H5P1L6U3S8W0A4B7M9Q",
      "name": "Ayesha Khan",
      "registration_no": "CS-2024-0153"
    },
    "fee_month": "2026-11-01",
    "due_date": "2026-11-10",
    "status": { "value": "partially_paid", "label": "Partially paid" },
    "net_amount": { "amount": "8500.00", "currency": "PKR" },
    "paid_amount": { "amount": "5000.00", "currency": "PKR" },
    "outstanding_amount": { "amount": "3500.00", "currency": "PKR" },
    "links": { "pdf": "https://api.darsgah.app/api/v1/fee/vouchers/01JD8Z3K6Q2M7V4T9X1B5C8N0R/pdf?signature=..." },
    "updated_at": "2026-11-05T06:12:44Z"
  },
  "meta": { "correlation_id": "01JDA0B1C2D3E4F5G6H7J8K9M0" }
}
```

Success, page-based list:

```json
{
  "data": [ { "id": "...", "voucher_no": "..." } ],
  "meta": {
    "pagination": { "page": 1, "per_page": 25, "total": 312, "last_page": 13 },
    "correlation_id": "..."
  },
  "links": { "first": "...", "prev": null, "next": "...", "last": "..." }
}
```

Feeds (notifications, announcements, messages, punches) use cursor pagination: `meta.next_cursor` and `meta.prev_cursor`.

Error (same shape as D3 3.16):

```json
{
  "message": "This voucher has already been paid in full.",
  "errors": {},
  "code": "fee.voucher_not_payable",
  "correlation_id": "01JDA0B1C2D3E4F5G6H7J8K9M0"
}
```

Error codes follow `{module}.{reason_snake}` and never change once published. Messages follow `Accept-Language` (`en` or `ur`).

### Querying lists

| Parameter | Example | Rule |
|---|---|---|
| Filter | `filter[status]=overdue&filter[student]=01JD...` | Only fields the endpoint's Query class allows; others return 400 |
| Sort | `sort=-due_date,voucher_no` | Allowlisted fields; `-` for descending |
| Include | `include=items,student` | Allowlisted relations; nothing loads lazily in a resource |
| Page size | `per_page=50` | Default 25, maximum 100 |
| Date ranges | `filter[from]=2026-11-01&filter[to]=2026-11-30` | Local dates for school-day facts |

`Modules\Core\Queries\ApiQuery` parses these against the allowlist declared in each Query class, so web filters and API filters share one implementation.

### Writes, idempotency and concurrency

| Topic | Rule |
|---|---|
| Idempotency | `Idempotency-Key` header accepted on every `POST`; **required** on payments, online payment initiation, attendance sheets, punches and bulk commands (FR-GEN-013). Stored in `idempotency_keys` for 24 hours; same key with a different body returns 422 `core.idempotency_key_mismatch`; same key and body returns the first response. |
| Concurrency | Single-record `GET` returns `ETag: W/"{sha1(id + updated_at)}"`. `PUT` and `PATCH` require `If-Match` (DEC-136). |
| Bulk | Bulk commands return 202 with a `bulk_operation` id; progress at `GET /api/v1/core/bulk-operations/{id}`. |
| Files | `POST /api/v1/core/files` (multipart) returns a file ULID after type, size and virus checks; the ULID is then sent in the owning request. Downloads are short-lived signed URLs. |
| Money in requests | Decimal strings (`"5000.00"`); floats are rejected with 422. |

### Rate limits (defined in `CoreServiceProvider`)

| Limiter | Limit | Key |
|---|---|---|
| `api` | 60 per minute per token, and 600 per minute per organization | token id; organization id |
| `auth` | 5 per minute | identifier plus IP |
| `public` | 10 per minute | IP |
| `webhooks` | 120 per minute | provider plus organization |
| `adms` | 60 per minute | device serial |
| `exports` | 5 per minute | user |

Responses include `X-RateLimit-Limit` and `X-RateLimit-Remaining`; 429 responses include `Retry-After`. Limits are configurable per plan in platform settings.

### Versioning and deprecation

- The version is in the path (`/api/v1`). Adding fields, endpoints or optional parameters is not breaking and stays in v1.
- Removing or renaming a field, changing a type or meaning, or tightening validation is breaking and needs v2.
- A deprecated endpoint sends `Deprecation: true`, `Sunset: {date}` and a `Link` header to its successor, and stays available for at least 12 months after v2 ships.
- Resources are versioned by folder (`Http/Resources/V1`), so a v2 resource never changes v1 output.

### API v1 endpoint catalog (R2)

| Method | Path | Ability | Purpose |
|---|---|---|---|
| POST | `/api/v1/identity/tokens` | none | Log in and get a token |
| POST | `/api/v1/identity/tokens/rotations` | any | Rotate the current token |
| DELETE | `/api/v1/identity/tokens/current` | any | Log out |
| POST | `/api/v1/identity/otp-requests` | none | Guardian first-login OTP |
| POST | `/api/v1/identity/otp-verifications` | none | Verify OTP, set password, get token |
| GET | `/api/v1/identity/me` | any | Profile, portals, branches, permissions summary |
| GET | `/api/v1/guardian/children` | `portal:guardian` | Linked children with current class and section |
| GET | `/api/v1/attendance/student-days` | guardian, student | Daily attendance for a student and month |
| GET | `/api/v1/fee/vouchers` | guardian, student, app | Vouchers (filter by student, status, month) |
| GET | `/api/v1/fee/vouchers/{voucher}` | guardian, student, app | Voucher with items |
| GET | `/api/v1/fee/payments` | guardian, app | Payment history |
| POST | `/api/v1/fee/online-payments` | `portal:guardian` | Start a gateway payment (Idempotency-Key required) |
| GET | `/api/v1/result/student-results` | guardian, student | Published results |
| GET | `/api/v1/timetable/student-timetables/{student}` | guardian, student | Weekly timetable |
| GET | `/api/v1/communication/announcements` | all portals | Announcements for the user |
| POST | `/api/v1/communication/announcements/{announcement}/acknowledgements` | all portals | Acknowledge |
| GET | `/api/v1/communication/diary-entries` | guardian, student, app | Diary by date |
| POST | `/api/v1/communication/diary-entries` | `portal:app` | Teacher posts a diary entry |
| GET, POST | `/api/v1/communication/message-threads`, `.../{thread}/messages` | guardian, app | Guardian and teacher messaging |
| POST | `/api/v1/leave/applications` | guardian, app | Child leave or own leave |
| GET | `/api/v1/homework/assignments` | guardian, student, app | Homework list |
| POST | `/api/v1/homework/assignments` | `portal:app` | Teacher sets homework |
| POST | `/api/v1/homework/assignments/{assignment}/submissions` | `portal:student` | Submit homework |
| GET | `/api/v1/learning/courses`, `/api/v1/learning/lectures/{lecture}` | student, guardian | Courses and lectures |
| PUT | `/api/v1/learning/lectures/{lecture}/progress` | `portal:student` | Save progress (throttled) |
| GET | `/api/v1/assessment/quizzes` | `portal:student` | Open and upcoming quizzes |
| POST | `/api/v1/assessment/quizzes/{quiz}/attempts` | `portal:student` | Start an attempt |
| PUT | `/api/v1/assessment/quiz-attempts/{attempt}/answers/{question}` | `portal:student` | Save one answer |
| POST | `/api/v1/assessment/quiz-attempts/{attempt}/submissions` | `portal:student` | Submit the attempt |
| GET | `/api/v1/academic/my-sections` | `portal:app` | Teacher's sections and teaching groups |
| GET, PUT | `/api/v1/attendance/section-sheets/{section}/{date}` | `portal:app` | Load and mark a section's attendance (Idempotency-Key on PUT) |
| GET | `/api/v1/approval/inbox` | `portal:app` | Pending approvals for the user |
| POST | `/api/v1/approval/requests/{request}/decisions` | `portal:app` | Approve, reject or return |
| GET, PATCH | `/api/v1/notification/notifications`, `.../{notification}` | all portals | In-app notifications; mark read |
| POST | `/api/v1/attendance/punches` | `device:punch` | Sync agent punch batch (Idempotency-Key required) |
| POST | `/api/v1/core/files` | any | Upload a file |
| GET | `/api/v1/core/bulk-operations/{operation}` | any | Bulk progress |

### Internal JSON endpoints (R1)

| Rule | Detail |
|---|---|
| Location | Portal routes (`/app/...`), session auth, CSRF required |
| Controllers | Same `Http/Controllers/App` classes, separate methods (`sheet`, `lookup`, `save`) |
| Resources | `Http/Resources/Internal`, same envelope and error shape as the API |
| Contract | Not versioned and not in the OpenAPI spec; may change with the screen that uses them |
| Examples | `GET /app/attendance/section-sheets/{section}/{date}` (marking grid), `GET /app/fee/payments/voucher-lookup` (cashier), `PUT /app/examination/mark-sheets/{paper}/{section}` (marks entry autosave) |

### Inbound webhooks and ADMS

Covered in D3 3.15 and D4 4.7. Folder placement: `Http/Controllers/Webhooks` and `Http/Controllers/Adms` in the owning module, routes in `routes/webhooks.php` and `routes/adms.php`, signature checks in `VerifyWebhookSignature` with provider-specific verifiers in `Adapters/{Provider}`. Every receipt is stored in `webhook_receipts` before processing (DEC-42), and gateway payments are confirmed through the provider's verify API before posting.

### API documentation

`docs/api/openapi-v1.yaml` with one path file per module. Each endpoint lists its ability, permission, request schema, response schema, error codes and whether `Idempotency-Key` or `If-Match` is required. `OpenApiCoverageTest` keeps routes and spec in step (DEC-135).

---

## 8.16 Naming Conventions Applied to Files

| Item | Convention | Example |
|---|---|---|
| Model | Singular StudlyCase of the table | `fee_vouchers` → `FeeVoucher` |
| Migration | `{timestamp}_create_{table}_table.php` | `2026_12_01_070500_create_fee_vouchers_table.php` |
| Factory | `{Model}Factory` | `FeeVoucherFactory` |
| Action | Imperative verb plus noun | `RecordPayment`, `PublishResultBatch` |
| Service | Noun plus role | `VoucherCalculator`, `AllocationService` |
| Query | `{Subject}Query` | `DefaulterQuery` |
| Controller | `{Resource}Controller`; single-action controllers use `__invoke` | `CashierSessionController` |
| Form Request | `{Verb}{Resource}Request` | `StorePaymentRequest`, `ReversePaymentRequest` |
| Policy | `{Model}Policy` | `FeeVoucherPolicy` |
| Event | Past tense | `PaymentRecorded` |
| Listener | Verb phrase describing the reaction | `RecalculateAccountOnPayment` |
| Job | Imperative | `GenerateVoucherChunk` |
| Command class / signature | `{Verb}{Noun}Command` / `{module-kebab}:{action-kebab}` | `MarkOverdueVouchersCommand` / `fee:mark-overdue` |
| Enum | `{Concept}{Kind}` | `VoucherStatus`, `PaymentMethod` |
| DTO | `{Concept}Data` | `PaymentData` |
| Domain exception | Plain statement | `VoucherNotPayable` |
| API resource | `{Model}Resource` in `V1` | `FeeVoucherResource` |
| View | `{module}::{portal}.{resource-kebab}.{view}` | `fee::app.cashier-sessions.show` |
| Translation key | `{module}::{file}.{key}` | `fee::payments.recorded` |
| Blade component | `x-ui.{name}`, `x-{module-kebab}.{name}` | `x-ui.modal`, `x-fee.voucher-card` |
| Alpine component | `{moduleCamel}{Component}` | `attendanceMarkingGrid` |
| Setting key | `{module}.{key}` | `library.fine_per_day` |
| Permission key | `{module}.{resource}.{action}` | `fee.payment.create` |
| Audit key | `{module}.{resource}.{past_verb}` | `fee.payment.recorded` |
| Error code | `{module}.{reason}` | `fee.voucher_not_payable` |
| Cache key | `org:{id}:{area}:...:v{version}` | `org:42:lookup:fee_heads:v7` |
| Morph alias | Table name in singular | `fee_voucher` |

---

## 8.17 Adding a New Module (checklist)

1. Add the entry to `config/modules.php` with key, code, tier, release and `depends_on`.
2. Create `modules/{Name}/Providers/{Name}ServiceProvider.php` extending `ModuleServiceProvider`, and add it to `bootstrap/providers.php` at its manifest position.
3. Add `Definitions/Permissions.php`, `Navigation.php` and `Settings.php`; run `iam:permissions:sync`.
4. Write migrations under `Database/Migrations` with timestamps in the right D7 batch, using the Blueprint macros.
5. Add models with `BelongsToOrganization` (and `BelongsToBranch` where needed), policies, factories.
6. Add route files; use `Route::portalResource()`.
7. Add views under `resources/views/{portal}/` and composites under `resources/views/components/{module-kebab}/`.
8. Add `lang/en` and `lang/ur` files.
9. Add tests under `tests/Feature/{Name}` including the two-tenant isolation test; run the arch suite.
10. Add API endpoints to `docs/api/v1/paths/{module}.yaml` if the module exposes any.

---

## Assumptions

| ID | Assumption |
|---|---|
| A-69 | One repository holds the whole application; modules are not published as separate Composer packages. |
| A-70 | Native mobile apps stay out of scope (D1 non-goals); API v1 is built in R2 so they can be added later without backend changes. |
| A-71 | Local development uses the Docker Compose file in `deploy/docker`; Laravel Sail is not used. |
| A-72 | The OpenAPI specification uses version 3.1 and is written by hand. |
| A-73 | Product name and root domain (`Darsgah`, `darsgah.app`) remain placeholders and are read only from `config/darsgah.php`, so renaming touches one file and the environment. |

## Gaps Found and Added

| ID | Gap | Decision |
|---|---|---|
| GAP-173 | Lowercase `database/factories` in modules breaks PSR-4 autoloading on Linux | StudlyCase class folders (DEC-129) |
| GAP-174 | Laravel's automatic discovery of events, policies, commands and factories only looks in `app/` | Base module provider and convention callbacks (DEC-132) |
| GAP-175 | Prefixed component namespaces render as `x-fee::card`, not D1's `x-fee.card` | Unprefixed anonymous paths with module subfolders (DEC-130) |
| GAP-176 | Module views and scripts were not part of the CSS and JS build | `@source` lines and `import.meta.glob` (8.9) |
| GAP-177 | Module metadata was scattered across D1, D2 and D3 | `config/modules.php` (DEC-127) |
| GAP-178 | Multi-word module and resource names had no URL or route-name rule | DEC-128 |
| GAP-179 | Menus had no source; layouts would have hardcoded links | `NavigationRegistry` (DEC-137) |
| GAP-180 | API value formats, pagination, filtering, versioning and status codes were undefined | 8.15, DEC-134 |
| GAP-181 | Optimistic locking had no mechanism for web or API | DEC-136 |
| GAP-182 | API spec could drift from routes | Contract test (DEC-135) |
| GAP-183 | Device tokens for the sync agent had no owner | Sanctum token on `attendance_devices` with `device:punch` (8.15) |
| GAP-184 | `notification_dispatches` was missing from the D7 partition plan | Correction in 8.1 |
| GAP-185 | No home for agent prompts and checklists inside the repository | `docs/agent/` (8.13) |

---

Reply CONTINUE for Deliverable 9
