# Deliverable 12: Security, Performance, Testing and Deployment
# Part 2 of 2: Performance, Testing, Deployment and Operations

**Version:** 1.0 (draft for approval) | **Date:** 24 Nov 2026
**Builds on:** D1 1.18 (success metrics), D3 3.11 (queues), 3.19 (logging), 3.20 (testing architecture), 3.21 (packages, DEC-01), D4 (hosts, tenant export and extraction), D7 (indexes, partitions, snapshots), D8 8.13 and 8.14 (docs and tests trees), D10 (dashboards, `ScreenCoverageTest`), D12 Part 1 (security)

---

## 12.16 New Decisions

| ID | Decision |
|---|---|
| DEC-198 | **Runtime switch (RT-1):** local, CI and staging run PHP 8.4 by 15 Dec 2026. Production never runs PHP 8.2. Until UG-1, CI still runs the PHP 8.2 job so the code stays inside the DEC-01 feature limits. |
| DEC-199 | **UG-1 deadline:** the move to Laravel 13 is finished by 31 Jan 2027. No production traffic runs on Laravel 12 after 24 Feb 2027. At UG-1 the platform minimum becomes PHP 8.4 (`"php": "^8.4"`), the CI matrix becomes PHP 8.4 and 8.5, Larastan uses `phpVersion: 80400`, and Pest moves to 4. The DEC-01 forbidden-feature list is retired at the same time, with one exception: property hooks and asymmetric visibility stay banned on Eloquent models, because models depend on magic attributes. |
| DEC-200 | **UG-2 (Laravel 14):** adopted after its first point releases and within 6 months of its release date. The hard limit is the end of Laravel 13 security fixes. |
| DEC-201 | **Database line:** production stays on MySQL 8.4 LTS. UG-3 moves to MySQL 9.7 LTS during 2028, before 8.4 premier support ends. Innovation releases never run in production. |
| DEC-202 | **Performance budgets (12.17) are release gates.** They are measured by the staging load test before each release and by production p95 every week. |
| DEC-203 | **Replica reads:** only reports, exports, snapshot computation and scheduled report jobs read from the replica (`mysql_replica` connection). Every user-facing transactional screen reads the primary. If replica lag is over 30 seconds, report jobs wait and report screens show the "as of" time from the replica. |
| DEC-204 | **Two Redis roles:** `redis-cache` (eviction `allkeys-lru`, no persistence) for cache, and `redis-queue` (eviction `noeviction`, AOF every second) for queues, locks and rate limiters. A full cache can never evict a queued job. |
| DEC-205 | **Edge and TLS:** Caddy at the edge. Platform hosts use a wildcard certificate for `*.darsgah.app`, issued through a DNS challenge. Custom domains use on-demand TLS. Before issuing, Caddy calls the internal endpoint `internal.domains.check`, which returns 200 only for a verified `organization_domains` row. |
| DEC-206 | **One container image** per commit serves the web, worker and scheduler roles with different start commands. The image is built once, scanned, signed and promoted unchanged. |
| DEC-207 | **Zero-downtime deploys with expand-and-contract migrations.** A release only adds schema. Removals ship in a later release, after no running code uses the old schema. Production never runs `down` migrations; rollback means redeploying the previous image. |
| DEC-208 | **Backups:** binary logs stream continuously to object storage, and a full physical backup runs every night. Every backup is encrypted, with an immutable off-site copy under a 35-day object lock. The RPO target is 15 minutes against a commitment of 1 hour (D1). RTO is 4 hours for component loss and data corruption, and 24 hours for loss of the whole region (accepted in Part 1 12.15). |
| DEC-209 | **A single-tenant restore never overwrites other tenants.** We restore to a point in time on a scratch instance, then repair or re-import only that organization's rows with tenant-scoped tooling. |
| DEC-210 | **Testing rules:** all tests run on MySQL 8.4 (DEC-39). Every change meets the definition of done in 12.25. CI gates in 12.27 block merging. |
| DEC-211 | **Load tests** use k6 scripts in `tests/Load/` (correction to D8 8.14). They run against staging before every release, and the full scenario set runs weekly. |
| DEC-212 | **Deploy window:** 22:00 to 02:00 PKT, Sunday to Thursday. No planned deploys during school hours (07:00 to 15:00 PKT), on Friday after 18:00, or on Saturday. The platform freeze calendar blocks deploys around declared peak events. Hotfixes can go out at any time with the incident lead's approval. |
| DEC-213 | **Service level and monitoring:** monthly availability SLO of 99.5% (D1), measured on the tenant web host and the guardian portal. Every scheduled command pings a dead-man's switch monitor on success. Alerts follow 12.33. |
| DEC-214 | **Production data stays in production.** A lower environment gets only an anonymized copy made by `core:anonymize-copy`, run by platform engineering with the platform owner's approval, for performance testing. |
| DEC-215 | **High-volume PDFs are pre-rendered.** Report cards are rendered when results are published, and voucher batches when vouchers are issued. Both are stored as `stored_files` and served from object storage. Live Gotenberg rendering is only for single documents on demand. Pre-rendered voucher PDFs are deleted after 13 months; the data stays and the PDF can be re-rendered. |

### Corrections to earlier deliverables

| Where | Change |
|---|---|
| D8 8.14 | Adds `tests/Load/` (k6 scripts, DEC-211) and `tests/Feature/Ops/` (health, backup and scheduler checks). |
| D8 8.2 and D3 3.21 | `composer.json` keeps `"php": "^8.2"` until UG-1, then `"php": "^8.4"` (DEC-199). After UG-1, the package rule DEC-01 reads "supports PHP 8.4 and 8.5 and Laravel 13 and 14". |
| D8 8.17 step 3 | The command is `identity:sync-permissions` (as already corrected in D10 10.1). |
| D4 hosts | Adds `status.darsgah.app`, an externally hosted status page that stays up during a platform outage. No tenant data. |
| D3 3.11 | Queue workers come from DEC-204 `redis-queue`. Wait-time targets and alerts are in 12.17 and 12.33. |

---

## 12.17 Performance Budgets (DEC-202)

Server time is measured at the application (request start to response end). Device timings are measured on a mid-range Android phone over 4G.

| Area | Budget (p95 unless stated) | Source |
|---|---|---|
| List screens, 50 rows | Under 800 ms server time; target 500 ms | D1 |
| Detail and form pages | Under 400 ms | New |
| Save attendance for a 40-student section | Under 1 s | D1 |
| Student quick search | Under 300 ms | New |
| Dashboard, full page | Under 2 s; each widget under 300 ms | D1, D10 |
| Owner dashboard, 20 branches | Under 10 s, end to end | D10 10.5 |
| API read endpoints | Under 300 ms | New |
| API write endpoints | Under 600 ms | New |
| Webhook acknowledgement (store first) | Under 500 ms | New |
| ADMS punch acknowledgement | Under 300 ms | New |
| Single PDF on demand (one voucher, one certificate) | Under 2 s | New |
| Voucher generation for 10,000 students | Under 10 minutes, queued | D1 |
| Result processing for 5,000 students | Under 15 minutes, queued | D1 |
| Import validation, 5,000 rows | Preview under 2 minutes | D1 |
| SMS burst for 2,000 absences | Queued within 1 minute | D1 |
| Guardian portal home | LCP under 2.5 s; INP under 200 ms; CLS under 0.1 | New |
| Staff portal list page | LCP under 2.5 s on a 4-year-old laptop over a 10 Mbps link | New |
| JavaScript, gzip | `portal.js` up to 60 KB; `app.js` up to 120 KB; Chart.js loaded only on dashboards | D3 3.17 |
| CSS, gzip | Up to 50 KB per bundle | New |
| Queries per request | Up to 30 (asserted in tests); no lazy loading outside production | New |

### Queue wait-time targets (time from dispatch to start)

| Queue (D3 3.11) | Target p95 wait | Alert when |
|---|---|---|
| `critical` | 5 s | Over 30 s for 5 minutes |
| `attendance` | 30 s | Over 2 minutes |
| `notifications` | 60 s | Over 5 minutes |
| `documents` | 2 minutes | Over 10 minutes |
| `bulk` | 2 minutes | Over 15 minutes |
| `imports`, `exports` | 5 minutes | Over 20 minutes |
| `reports` | 10 minutes | Over 30 minutes |
| `default` | 60 s | Over 5 minutes |

---

## 12.18 Database Performance Rules

| Rule | Detail |
|---|---|
| Index shape | Every tenant list query uses an index that starts with `organization_id`, then the filter columns, then the sort column (D7 7.3 index naming). |
| Query review | Every new Query class has a test that runs `EXPLAIN` on a seeded dataset and fails on a full table scan of any table marked `large` in D7 (attendance, punches, audit, journals, notifications, payments). |
| Pagination | Offset pagination for ordinary lists up to 10,000 rows. Cursor pagination for large tables (audit log, punches, delivery logs, journal lines). |
| Counts | No exact `COUNT(*)` on large tables in lists; use "simple" pagination with "next" and "previous". Totals come from snapshots. |
| Eager loading | `Model::preventLazyLoading()` outside production; resources declare their relations. |
| Writes in bulk | Attendance sheets, voucher batches and result rows use chunked `insert` or `upsert` inside one transaction per chunk (D3). |
| Locks | Money and counters use `SELECT ... FOR UPDATE` on the smallest row set (D7 7.46 sequences); no table locks; lock wait timeout 10 s for web requests. |
| Snapshots | Dashboards read `daily_metric_snapshots` (D7 7.34). Live widgets cache for 60 seconds (D10). |
| Partitions | Large time-series tables are partitioned by month or year (D7 7.50). `core:partitions:maintain` creates future partitions monthly, and `core:partitions:archive` moves old ones out. |
| Slow queries | `long_query_time = 0.5` in production. The weekly review fixes or explains every query above 1 s. |
| Replica | DEC-203. The Laravel connection uses `sticky => true` so a request that wrote reads its own writes. |

### MySQL 8.4 configuration baseline

| Setting | Value | Why |
|---|---|---|
| `innodb_buffer_pool_size` | About 70% of RAM | Working set in memory |
| `innodb_flush_log_at_trx_commit` | 1 | No lost commits on crash (money) |
| `sync_binlog` | 1 | Binary log durable for point-in-time recovery |
| `binlog_format` | ROW | Safe replication |
| `gtid_mode`, `enforce_gtid_consistency` | ON | Simple replica promotion |
| `binlog_expire_logs_seconds` | 7 days on the server (backups keep longer) | Disk control |
| `max_connections` | Sized to FPM children plus workers plus 20% | Avoid connection storms |
| `character_set_server`, `collation_server` | `utf8mb4`, `utf8mb4_0900_ai_ci` | Urdu and English text (D7) |
| `sql_mode` | MySQL 8.4 default (strict) | Matches tests |
| `transaction_isolation` | REPEATABLE-READ (default) | Matches tests; money uses explicit row locks |

---

## 12.19 Caching

| Cache | Store | Key pattern (D8 8.16) | Invalidation |
|---|---|---|---|
| Config, routes, views, events | Files in the image | n/a | Built at container start (`optimize`) |
| Tenant resolution by host | `redis-cache` | `host:{host}:v{n}` | On domain or status change |
| Permissions per user and branch | `redis-cache` | `org:{id}:perm:{user}:v{version}` | Version bump on role, override or assignment change (D5) |
| Settings | `redis-cache` | `org:{id}:settings:v{version}` | Version bump on save |
| Lookups (fee heads, classes, subjects) | `redis-cache` | `org:{id}:lookup:{name}:v{version}` | Version bump on change |
| Live widgets | `redis-cache` | `org:{id}:widget:{key}:{scope-hash}` | 60-second TTL |
| Translations with overrides | `redis-cache` | `org:{id}:lang:{locale}:v{version}` | Version bump on override change |
| Static assets | CDN and browser | Hashed file names | New build |
| Pre-rendered PDFs | Object storage | `stored_files` | DEC-215 retention |

Rules: personal data is never cached in shared keys, and keys always carry the organization. Business data in a page is never full-page cached. A cache miss always recomputes safely.

---

## 12.20 Frontend Performance

| Rule | Detail |
|---|---|
| Server-rendered first | Blade renders the content; Alpine adds behaviour (D3). |
| Bundle split | `app.js` for staff, `portal.js` for guardians and students, Chart.js loaded only on dashboard routes (D3 3.17). The CI build fails if a bundle goes over its budget. |
| Images | Re-encoded at upload (DEC-192), served at the needed size (thumbnails 96 px and 240 px), `loading="lazy"` below the fold, width and height set to stop layout shift. |
| Fonts | System fonts for Latin text. Noto Nastaliq Urdu is loaded only when Urdu text is present, through `unicode-range`, with `font-display: swap` (D9). |
| Caching | Hashed assets with `Cache-Control: public, max-age=31536000, immutable`. HTML is `no-store` for authenticated pages. |
| Compression | Brotli or gzip at the edge. |
| Tables | Server pagination; no rendering of thousands of rows in the browser. |
| Widgets | Each widget loads on its own (D10 DEC-159), so a slow one doesn't block the page. |

---

## 12.21 Capacity Model

### Planning estimates (A-99: average organization of 500 students)

| Item | At 2,000 organizations (A-35) |
|---|---|
| Students | About 1,000,000 |
| Staff users | About 67,000 |
| Active guardian users | About 800,000 |
| Sections marking attendance each morning | About 28,600, mostly between 08:00 and 09:30 |
| Attendance saves at peak | About 16 per second (3 times the 90-minute average), about 560 rows per second |
| Absence SMS after finalization | About 50,000 in 2 hours, provider limits permitting |
| Result day peak (A-100) | About 670 page requests per second for the first hour if 30% of guardians sign in |
| Attendance rows per year | About 220 million (partitioned) |
| Audit rows per year | About 100 million (partitioned, archived per D7 7.50) |
| Object storage | About 2.5 TB of photos and documents, plus pre-rendered PDFs within DEC-215 retention |

### Scaling stages

| Stage | Size | Topology | Move to the next stage when |
|---|---|---|---|
| S1 (launch) | Up to about 300 organizations | 2 app nodes (4 vCPU, 8 GB), 1 worker node (4 vCPU, 8 GB), MySQL primary and replica (8 vCPU, 32 GB, NVMe), 2 small Redis, 2 Gotenberg (2 vCPU, 4 GB), 2 edge nodes | CPU over 60% at peak for 2 weeks, or p95 over budget |
| S2 | Up to about 2,000 organizations | 4 to 6 app nodes, 3 worker nodes, MySQL 16 vCPU with 64 to 128 GB, 4 Gotenberg | Primary over 60% CPU at peak, buffer pool hit rate under 99%, or storage growth beyond the node |
| S3 | Beyond 2,000 organizations, or one very large tenant | A second database cluster; organizations moved with the D4 tenant extraction runbook; tenant-to-cluster routing in the platform registry | n/a |

### App node tuning

| Setting | Value |
|---|---|
| PHP-FPM | `pm = static`, `pm.max_children` = 6 per vCPU (24 on 4 vCPU), `pm.max_requests = 1000` |
| OPcache | Enabled, `validate_timestamps = 0` (immutable image), memory 256 MB, JIT off |
| Realpath cache | 4096K, TTL 600 |
| Request limits | Body up to 25 MB (imports), timeout 60 s for web; long work always queued |

---

## 12.22 Load Testing (DEC-211)

k6 scripts in `tests/Load/` run against staging, seeded by `core:anonymize-copy` or the `LoadTestSeeder` (2,000 synthetic organizations).

| Scenario | Script | Load | Pass criteria |
|---|---|---|---|
| Morning attendance | `attendance-morning.js` | 20 saves per second for 30 minutes, plus list browsing | Save p95 under 1 s; errors under 0.1%; `attendance` queue wait under 30 s |
| Result day | `result-day.js` | Ramp to 700 requests per second over 10 minutes, hold 30 minutes; report card downloads from storage | p95 under 800 ms; errors under 0.5%; no Gotenberg calls for report cards (DEC-215) |
| Fee counter rush | `fee-counter.js` | 200 cashiers recording payments together | Record payment p95 under 600 ms; zero duplicate receipts; journals balanced |
| Month-start vouchers | `voucher-batch.js` | 20 organizations generating 10,000 vouchers each at once | Each under 10 minutes; other tenants' list p95 still within budget (fairness) |
| Import | `import-5k.js` | 10 parallel 5,000-row student imports | Preview under 2 minutes each |
| Webhook flood | `webhook-flood.js` | 100 payment callbacks per second with 20% duplicates | Acknowledge under 500 ms; zero double postings |
| ADMS burst | `adms-burst.js` | 2,000 devices sending punches over 10 minutes | Acknowledge under 300 ms; no lost punches |
| Soak | `soak.js` | Mixed traffic at 50% of peak for 8 hours | No memory growth in workers; no rise in p95 |

---

## 12.23 Testing Strategy (DEC-210)

### Test types

| Type | Tool | Location (D8 8.14) | Runs | Share of suite |
|---|---|---|---|---|
| Unit | Pest | `tests/Unit/{Module}` | Every push | About 30% |
| Feature (HTTP, Actions, jobs) | Pest, MySQL 8.4 | `tests/Feature/{Module}` | Every push | About 60% |
| Architecture | Pest arch plugin | `tests/Architecture` | Every push | Small, strict |
| Tenant isolation | Generated from models and routes | `tests/Feature/{Module}/Isolation` | Every push | 100% of tenant models and routes |
| RBAC matrix | Data provider built from module `Definitions/Permissions.php` and D5 role defaults | `tests/Feature/Identity/RbacMatrixTest.php` | Every push | Every role and permission |
| API contract | OpenAPI coverage (DEC-135) | `tests/Contract` | Every push | Every API route |
| Screen coverage | `ScreenCoverageTest` (D10) | `tests/Feature/Reporting` | Every push | Every screen ID |
| Translation parity | `TranslationParityTest` | `tests/Architecture` | Every push | Every `en` key has an `ur` key |
| Browser | Pest 4 browser testing (after UG-1) | `tests/Browser` | On merge to main and nightly | Key journeys |
| Accessibility | axe checks inside browser tests | `tests/Browser/Accessibility` | Nightly | Wireframed screens W-01 to W-16 and every portal home |
| PDF rendering | Real Gotenberg with Urdu text, page count and text checks | `tests/Feature/Document/Rendering` (group `gotenberg`) | Nightly | Every PDF template |
| Mutation | Pest mutation testing | Fee, Accounting, Payroll, Result posting and calculation classes | Nightly | Mutation score at least 80% on those classes |
| Load | k6 | `tests/Load` | Before release and weekly | 12.22 |
| Security scan | OWASP ZAP | CI job | Per release, monthly authenticated | Part 1 12.11 |

### Shared fixtures and helpers (D8 8.14)

| Helper | Use |
|---|---|
| `TwoTenantFixture` | Organization A and B with mirrored data for isolation tests |
| `actingAsRole('cashier', branch: $branch)` | Logs in a user with exactly that role's defaults |
| `inTenant($org, fn () => ...)` | Runs code inside a tenant context |
| `FakeAdapters` | Fakes for SMS, email, WhatsApp, payments, devices and PDF; no test calls a real provider |
| `assertAudited()`, `assertNotVisibleToOtherTenant()`, `assertJournalBalanced()` | Common assertions |
| `assertQueryCountAtMost(30)` | Query budget (12.17) |
| `Carbon::setTestNow()` with `Asia/Karachi` organization time zone | Date rules (late marks, due dates, period locks) |
| `DemoSeeder` | Deterministic sample data for staging and demo (sample organizations "Al-Noor" and "City School") |

### Coverage targets

| Area | Line coverage | Extra rule |
|---|---|---|
| Actions and Services in Fee, Accounting, Payroll, Result, Examination | 95% | Every posting rule F1 to F21 and P1 to P6 (D11) has a named test |
| Identity, Core tenancy, Approval | 95% | Every SOD rule SOD-01 to SOD-10 has a positive and a negative test |
| Other modules | 80% | |
| Whole application | 85% | A merge that lowers coverage by more than 1 point fails |

---

## 12.24 Required Security and Integrity Tests

| Test | What it proves |
|---|---|
| Isolation suite | Organization A cannot read, list, update or delete B's records through routes, queries, exports, jobs or files |
| Route permission coverage | Every protected route checks a permission (`RouteConventionsTest`) |
| RBAC matrix | Each default role gets exactly its D5 permissions, and visibility levels behave as defined |
| Masking | Restricted fields are masked in screens, exports and API responses without `core.pii.view_full` |
| Log redaction | Passwords, OTPs, tokens, CNIC values and card-like numbers never reach log output |
| Headers | CSP with nonce and the other Part 1 12.7 headers on every portal response |
| Uploads | Disallowed types and SVG rejected; EXIF stripped; downloads need a valid signature and permission |
| Webhooks | Bad signatures rejected; duplicates do not post twice; payments post only after the verification fake confirms |
| Payment trust | A browser return URL with "success" never changes a voucher |
| SOD | Every SOD rule blocks the same user from both sides |
| Immutability | Paid vouchers, posted journals and locked periods reject edits at the Action level |
| Impersonation limits | Impersonated sessions cannot see Restricted fields, change credentials or approve money requests |
| Audit privilege | The app database user cannot `UPDATE` or `DELETE` audit rows (run against the CI database with production grants) |
| Session rules | Idle and absolute timeouts per DEC-189; regeneration on login and portal switch |
| Throttling | Login, OTP and API limiters return 429 at the stated limits |

---

## 12.25 Definition of Done for Every Change

| Check | Required |
|---|---|
| Feature tests | Happy path, validation failure (422), permission denied (403), other tenant (404), audit entry asserted |
| Money changes | Journal balanced, idempotent on retry, reversal path tested |
| Workflow changes | Every allowed transition and at least one forbidden transition per state |
| New screen | Entry in the D10 screen inventory and `ScreenCoverageTest`; empty, loading and error states; works at 360 px width; keyboard reachable |
| New notification | Template exists in `en` and `ur`; preference rule respected |
| New import or export | Template, error file, masking, permission per type |
| New API endpoint | OpenAPI path and contract test |
| New query on a large table | `EXPLAIN` test (12.18) |
| New package | DEC-01 rule check, `composer audit` clean, licence acceptable |
| Docs | Decision added to `docs/decisions/decisions.md` if a rule changed |
| Checks | Pint, Larastan, all test suites green in CI |

---

## 12.26 Environments

| Environment | Purpose | Hosts | Data | Integrations |
|---|---|---|---|---|
| Local | Development | `*.darsgah.test` (Docker Compose, A-71) | `DemoSeeder` | Fakes; Mailpit for email; local Gotenberg |
| CI | Automated tests | n/a | Factories | Fakes; real Gotenberg in the nightly group |
| Staging | Release verification, load tests | `*.staging.darsgah.app` | `DemoSeeder`, `LoadTestSeeder`, or an anonymized copy (DEC-214) | Provider sandboxes (payment test merchants, SMS test sender) |
| Demo | Sales sandbox | `*.demo.darsgah.app` | `DemoSeeder`, reset every night | Fakes; messages shown in an in-app outbox only |
| Production | Live | `*.darsgah.app` and verified custom domains | Real | Live providers with each organization's credentials |

Rules: each environment has its own `APP_KEY`, pepper, database, Redis, storage bucket and provider credentials. Staging and demo send no SMS or email to real numbers outside an allowlist. Access to production servers is limited to named platform engineers through SSO and a bastion, with session recording.

---

## 12.27 CI Pipeline and Gates

```
Every push (target under 10 minutes)
  composer validate -> pint --test -> larastan -> arch tests -> unit tests
  gitleaks -> composer audit -> npm audit -> vite build with bundle budget check

Every pull request (adds, target under 25 minutes)
  feature tests in parallel on MySQL 8.4
    PHP matrix: 8.2 and 8.4 until UG-1, then 8.4 and 8.5 (DEC-199)
  isolation suite -> RBAC matrix -> contract tests -> screen coverage -> translation parity
  migrate:fresh --seed on an empty database (migration order check, D7 7.51)
  coverage report and coverage gate

Merge to main
  build image (tag = commit SHA) -> OS vulnerability scan -> sign -> push to registry
  deploy to staging -> smoke tests -> browser tests -> ZAP baseline scan

Nightly
  full browser and accessibility suite -> Gotenberg rendering group -> mutation tests
  reduced load test (attendance-morning, webhook-flood)

Release (manual approval by the release owner)
  full load test set passed within 7 days -> production deploy in the DEC-212 window
  -> post-deploy smoke tests -> 30-minute watch (12.28)
```

| Gate | Blocks |
|---|---|
| Any failing test, lint or static analysis | Merge |
| High or critical `composer audit` or `npm audit` finding | Merge |
| Secret found by gitleaks | Merge |
| Bundle over budget | Merge |
| Coverage drop over 1 point, or below an area target | Merge |
| Critical or high image vulnerability with a fix available | Promotion to production |
| Load test outside budget | Release |
| ZAP high finding | Release |

---

## 12.28 Production Topology

```
                         Internet
                            |
              DNS (*.darsgah.app, custom domains CNAME to edge)
                            |
        +-------------------+--------------------+
        |  Edge: 2 x Caddy (DEC-205), floating IP or cloud load balancer
        |  TLS 1.2+/1.3, wildcard cert, on-demand TLS -> internal.domains.check
        |  Brotli, request size limits, rate limits per IP, CDN for /build assets
        +-------------------+--------------------+
                            |  private network
     +----------------------+------------------------------+
     |                      |                              |
 App nodes (web role)   Worker nodes (worker role)    Scheduler (1 node, scheduler role)
 Nginx + PHP-FPM 8.4    Horizon supervisors per        schedule:work, onOneServer locks
 /up health check       D3 3.11 queues                 dead-man's switch pings (DEC-213)
     |                      |                              |
     +-----------+----------+-------------+----------------+
                 |                        |
        MySQL 8.4 LTS                 redis-cache  (allkeys-lru, no persistence)
        primary + replica (GTID)      redis-queue  (noeviction, AOF everysec)  DEC-204
        binlog stream -> object storage
                 |
        Gotenberg 8 (2+ containers, internal only, no outbound network)
        ClamAV (optional, internal only)
        Object storage (S3-compatible, private buckets, versioning)
        Off-site backup bucket (second location, object lock 35 days)

 Separate: ADMS host adms.darsgah.app (HTTP allowed on /iclock/*, Part 1 12.8)
           hooks.darsgah.app (webhooks), api.darsgah.app (API v1), platform.darsgah.app
           status.darsgah.app (external status page)
 Observability: logs, metrics, traces and alerts (12.33)
```

### Host routing

| Host | Served by | Middleware group (D8) |
|---|---|---|
| `{slug}.darsgah.app`, verified custom domains | App nodes | `tenant.web`, `tenant.guest`, `tenant.public`, `portal.*` |
| `platform.darsgah.app` | App nodes | `portal.platform` (IP allowlist optional) |
| `hooks.darsgah.app` | App nodes, separate FPM pool | `webhook` |
| `adms.darsgah.app` | App nodes, separate FPM pool | `adms` |
| `api.darsgah.app` | App nodes | `api.v1` |

Webhooks and ADMS get their own FPM pools so a device flood can't starve the web pool.

---

## 12.29 Deployment Process (DEC-206, DEC-207)

```
1  Pre-flight (automated)
     latest binlog in backup under 15 minutes old; last full backup under 26 hours old
     replica healthy and lag under 30 s; no SEV-1 or SEV-2 open; inside the DEC-212 window
2  Migrations (migration database user, DEC-190)
     expand-only changes; each migration under the time limit below, or it has a runbook
3  Web role: roll app nodes one at a time
     new container starts -> php artisan optimize -> /up healthy -> joins the pool
     old container drains for 30 s, then stops
4  Worker role: php artisan horizon:terminate on old workers
     running jobs finish their current chunk (bulk jobs are chunked and transactional)
     new worker containers start
5  Scheduler role: replace the container (locks prevent double runs)
6  Smoke tests: login page per portal, health ready endpoint, one tenant read,
     a queue round trip on each queue, one PDF render, a webhook to the test receiver
7  Watch for 30 minutes: error rate, p95 per route group, queue waits, failed jobs
8  Rollback when needed: redeploy the previous image tag (schema is backward compatible)
```

### Migration rules

| Rule | Detail |
|---|---|
| Expand first | Add new tables, nullable columns or columns with defaults, and new indexes. Code writes both the old and new shape if needed. |
| Backfill | Queued job in chunks of 1,000 rows, idempotent and resumable, with progress in `bulk_operations` |
| Contract later | Drop or rename columns only in a later release, after the code no longer reads them |
| Online DDL | Prefer `ALGORITHM=INSTANT` or `INPLACE`. On large tables, a change that needs a table copy runs with an online schema change tool during the window, with a runbook. |
| Time limit | Any migration estimated over 30 seconds on production-size data (tested on the anonymized staging copy) needs a runbook and its own window |
| No data destruction | Migrations never delete business rows (brief rule 6). Data fixes are Actions with audit, not migrations. |
| `down()` methods | Written for local development only; never run in production |

---

## 12.30 Backups and Disaster Recovery (DEC-208, DEC-209)

| Item | Method | Frequency | Retention |
|---|---|---|---|
| MySQL binary logs | Streamed to object storage | Continuous | 35 days |
| MySQL full physical backup | A physical backup tool that supports MySQL 8.4, or the managed service's snapshots | Nightly at 02:30 PKT | Daily 35 days, monthly 12 months, yearly 7 years |
| Object storage | Versioning plus replication to the off-site bucket | Continuous | Deleted versions kept 35 days |
| Off-site copy | Encrypted, object lock 35 days | Nightly | As above |
| Redis | Not backed up | n/a | Queues rebuilt (below) |
| Configuration and secrets | Secret store with its own versioning | On change | 12 months |

Redis loss: work that matters is always recorded in MySQL first (`webhook_receipts`, `bulk_operations`, `notification_dispatches`, import jobs). `core:requeue-stuck` finds rows that are pending with no running job and dispatches them again. It is safe to repeat because handlers are idempotent.

### Recovery scenarios

| Scenario | Action | RTO | RPO |
|---|---|---|---|
| App or worker node lost | Load balancer removes it; replacement starts automatically | Minutes | 0 |
| Primary database lost | Promote the replica (GTID); point the app to it; build a new replica | 30 minutes | Seconds (asynchronous replication); webhooks and provider reconciliation recover any lost payment updates |
| Data corruption across tenants (bad release) | Stop writes in the affected module; restore to a point in time before the event on a scratch instance; repair through audited Actions or a reviewed script | 4 hours | 15 minutes target |
| One tenant's data damaged | DEC-209: point-in-time restore to scratch; tenant-scoped comparison using audit diffs; repair only that organization | 4 hours | 15 minutes target |
| Ransomware or account takeover of hosting | Rebuild from code and images; restore from the immutable off-site copy | 24 hours | Up to 24 hours (last off-site copy) plus binlogs if intact |
| Whole region lost | Rebuild in another region from off-site backups; update DNS | 24 hours | 1 hour |

### Drills

| Drill | Frequency | Pass criteria |
|---|---|---|
| Automated restore of last night's backup to a scratch instance, with row counts and checksums | Weekly | Completes; counts match the backup report |
| Single-tenant restore of a random organization | Monthly | Within 4 hours; no other tenant touched |
| Replica promotion in staging | Quarterly | Within 30 minutes |
| Full DR game day (staging rebuilt from off-site copy) | Twice a year | Within 24 hours; integrity checks (D11 11.13) pass |

---

## 12.31 Version Lifecycle and Upgrade Gates

| Component | Production version | Support facts | Gate |
|---|---|---|---|
| PHP | 8.4 | PHP 8.2 reaches end of life on 31 Dec 2026; 8.4 moves to security fixes only that day and reaches end of life on 31 Dec 2028 | RT-1 by 15 Dec 2026 (DEC-198); UG-4 to PHP 8.5 or later before 31 Dec 2028 |
| Laravel | 12, then 13 at UG-1 | Laravel 12 security fixes end on 24 Feb 2027 (bug fixes ended 13 Aug 2026). Laravel 13 was released on 17 Mar 2026, needs PHP 8.3 or later, and gets security fixes until 17 Mar 2028 | UG-1 by 31 Jan 2027 (DEC-199); UG-2 to Laravel 14 per DEC-200 |
| MySQL | 8.4 LTS | Premier support to April 2029, extended to April 2032. LTS upgrades go one line at a time (8.4 to 9.7, no skipping) | UG-3 to 9.7 LTS during 2028 (DEC-201) |
| Pest | 3, then 4 at UG-1 | Pest 4 needs PHP 8.3 (D3 3.21) | With UG-1 |
| Redis | 7 | Track vendor support | Yearly review |
| Gotenberg | 8 | Track releases; the Urdu rendering test must pass | Yearly review |
| OS base image | Current LTS distribution | Rebuilt weekly for patches | Weekly |

Every upgrade gate follows the same steps: branch, update dependencies, run the full CI and nightly suites, run the load set on staging, run a production deploy in the window, and watch for 24 hours.

---

## 12.32 Release and Change Management

| Item | Rule |
|---|---|
| Release cadence | Weekly release train; hotfixes as needed |
| Version naming | `YYYY.MM.N` (for example `2027.02.3`); shown in the platform portal and the footer of staff pages |
| Release notes | One page per release in the platform portal (SCR-PLT announcements), written for school staff, in English and Urdu |
| Module rollout | New modules and features reach tenants through module entitlements (D4); no separate feature-flag system |
| Technical toggles | Environment configuration only (for example a provider switch); every toggle has an owner and a removal date |
| Freeze calendar | Kept in the platform portal: Punjab board result days, annual result season, the first 10 days of each month (fee peak, no risky migrations), and admission peak weeks |
| Change approval | Any change to auth, tenancy, money posting, backups or the edge needs the release owner and a second engineer |

---

## 12.33 Monitoring and Alerting (DEC-213)

### What is measured

| Signal | Source |
|---|---|
| Availability | External checks every minute on the tenant host, guardian portal, API, webhooks and ADMS |
| Health | `/up` (Laravel liveness) and `internal.health.ready` (database, both Redis roles, queue round trip, Gotenberg, storage write) |
| Request metrics | p50, p95 and p99 per route group, error rate, by organization for the top 20 |
| Errors and traces | Sentry with 10% performance sampling (Part 1 scrubbing rules) |
| Queues | Horizon wait time, throughput, failed jobs per queue |
| Database | CPU, connections, buffer pool hit rate, replication lag, slow queries, disk, deadlocks |
| Redis | Memory, evictions (must be 0 on `redis-queue`), latency |
| Scheduler | Dead-man's switch per scheduled command |
| Backups | Success, age of the latest binlog and full backup, weekly restore result |
| Certificates | Days to expiry for wildcard and custom domains |
| Providers | SMS delivery rate per provider, payment verification errors, WhatsApp (R2) |
| Business health | Vouchers issued, payments posted, attendance finalized per day; sudden drops alert |

### Alerts

| Alert | Threshold | Severity | To |
|---|---|---|---|
| Tenant host or guardian portal down | 2 failed external checks in a row | SEV-1 | On-call (phone) |
| Error rate | Over 2% of requests for 5 minutes | SEV-2 | On-call |
| p95 over budget | Any route group over 150% of its budget for 15 minutes | SEV-3 | On-call |
| `critical` queue wait | Over 30 s for 5 minutes | SEV-2 | On-call |
| Failed jobs | Over 20 in 15 minutes on one queue | SEV-3 | On-call |
| Replication lag | Over 60 s for 5 minutes | SEV-3 | On-call |
| Database CPU | Over 80% for 15 minutes | SEV-3 | On-call |
| Disk | Over 80% used, or projected full within 7 days | SEV-3 | On-call |
| `redis-queue` evictions | Any | SEV-2 | On-call |
| Scheduled command missed | No ping within its grace time | SEV-3 (SEV-2 for `fee:recognize-deferred`, backups and partitions) | On-call |
| Backup too old | Binlog over 30 minutes, full over 26 hours | SEV-2 | On-call |
| Weekly restore failed | Any failure | SEV-2 | On-call |
| Certificate expiry | Under 14 days | SEV-3 | Platform engineering |
| SMS delivery rate | Under 85% for one provider over 30 minutes | SEV-3 | On-call; provider priority switched (D11 11.27) |
| Payment verification errors | Over 5% for one provider over 15 minutes | SEV-2 | On-call |
| Business signal drop | Attendance finalized under 50% of the 4-week average by 11:00 on a school day | SEV-3 | On-call |
| Security events | Part 1 12.10 | As listed there | As listed there |

On-call: one primary and one backup engineer (A-102), a weekly rotation, SEV-1 acknowledged within 15 minutes (Part 1 12.13).

---

## 12.34 Operations Calendar

| Frequency | Task | Owner | Automated |
|---|---|---|---|
| Daily | Backup report check; failed jobs review; security alerts review | On-call | Report automated, review manual |
| Daily | Snapshot computation, deferred fee recognition, approval digests (D3 scheduler) | System | Yes |
| Weekly | Restore test; slow query review; dependency update merge; OS image rebuild; production p95 report against budgets | Platform engineering | Mostly |
| Monthly | Single-tenant restore drill; `core:partitions:maintain`; platform user and support grant access review; authenticated ZAP scan; cost review | Platform engineering, security lead | Partly |
| Quarterly | Replica promotion drill; capacity review against 12.21; threat model review; key rotation readiness check | Platform engineering | No |
| Twice a year | Full DR game day | Platform engineering | No |
| Yearly | Penetration test; `APP_KEY` rotation (DEC-193); review of retention schedule (Part 1 12.12); session rollover readiness before each session start in the D11 school-year calendar | Security lead, product owner | No |

---

## 12.35 Runbooks

Stored in `docs/runbooks/` (D8 8.13). Each runbook has: when to use it, who may run it, steps with exact commands, how to verify, and how to roll back.

| Runbook | Purpose |
|---|---|
| `deploy.md` | Normal release (12.29) |
| `rollback.md` | Redeploy the previous image; handling of expand-only schema |
| `restore.md` | Full restore, point-in-time restore, single-tenant restore (DEC-209) |
| `replica-promotion.md` | Promote the replica and rebuild a new one |
| `redis-loss.md` | Restart Redis roles; run `core:requeue-stuck` |
| `gotenberg-down.md` | Queue holds `documents` jobs; serve pre-rendered PDFs; restart containers |
| `sms-provider-outage.md` | Switch provider priority; hold non-urgent broadcasts |
| `payment-provider-outage.md` | Hide the online payment option for affected organizations through settings; run reconciliation after recovery |
| `custom-domain-tls.md` | Certificate issuance failures and domain verification |
| `disk-full.md` | Database, logs and storage cleanup without deleting business data |
| `partitions.md` | Maintenance and archive failures |
| `online-schema-change.md` | Large table changes during a window |
| `upgrade-gate.md` | RT-1, UG-1, UG-2, UG-3 and UG-4 steps |
| `key-rotation.md` | `APP_KEY` and pepper rotation (DEC-193) |
| `tenant-export.md`, `tenant-extraction.md` | D4 procedures |
| `incident.md` | Part 1 12.13 process, with the SEV-1 and SEV-2 scenario pages |

---

## Assumptions (Part 2)

| ID | Assumption |
|---|---|
| A-99 | Capacity estimates use an average organization of 500 students, one staff member per 15 students and 0.8 active guardians per student. |
| A-100 | On the busiest result day, up to 30% of guardians sign in within the first hour. |
| A-101 | The R1 production launch happens after UG-1, so production starts on Laravel 13 with PHP 8.4. |
| A-102 | Platform operations have one primary and one backup on-call engineer reachable by phone. |
| A-103 | Open-source tools (k6, Caddy, Grafana-compatible monitoring) are acceptable, and the hosting provider supports object storage with object lock. |

## Gaps Found and Added (Part 2)

| ID | Gap | Decision |
|---|---|---|
| GAP-251 | Laravel 12 security support ends before the likely launch date | UG-1 deadline (DEC-199), A-101 |
| GAP-252 | The DEC-01 forbidden-feature list had no end date | Retired at UG-1, with the Eloquent exception (DEC-199) |
| GAP-253 | No rule on which reads may use the replica | DEC-203 |
| GAP-254 | A full cache could evict queued jobs | Two Redis roles (DEC-204) |
| GAP-255 | Custom domain certificate issuance was undefined | DEC-205 |
| GAP-256 | No schema rollback rule | Expand and contract, image rollback (DEC-207) |
| GAP-257 | A single-tenant restore could overwrite other tenants | DEC-209 |
| GAP-258 | Region-loss recovery time was not stated | DEC-208 |
| GAP-259 | No definition of done | 12.25 |
| GAP-260 | No load test tooling or scenarios | DEC-211, 12.22 |
| GAP-261 | Scheduled commands could stop silently | Dead-man's switch (DEC-213) |
| GAP-262 | Production data could reach lower environments | DEC-214 |
| GAP-263 | Losing Redis would lose queued work | `core:requeue-stuck`, records first in MySQL (12.30) |
| GAP-264 | Deploys could clash with school hours and peak days | DEC-212, freeze calendar (12.32) |
| GAP-265 | Live PDF rendering would collapse on result day | Pre-rendered PDFs (DEC-215) |
| GAP-266 | No status page during platform outages | `status.darsgah.app` (correction to D4) |

---

Reply CONTINUE for Deliverable 13
