# Deliverable 12: Security, Performance, Testing and Deployment
# Part 1 of 2: Security and Privacy

**Version:** 1.0 (draft for approval) | **Date:** 24 Nov 2026
**Builds on:** D3 (middleware, adapters, DEC-30 CSP, DEC-42 webhooks), D4 (isolation layers, DEC-43 to DEC-52), D5 (RBAC, masking, SOD, DEC-62, DEC-63), D7 (encryption DEC-95, audit tables, partitioning), D8 (middleware groups, API security), D11 (approvals, money controls)

**Two parts:**

| Part | Contents |
|---|---|
| **1** | Security objectives, data classification, threat model, controls by layer, tenant isolation summary, webhooks and payments, secrets and keys, database privileges, logging and security monitoring, secure development, privacy and legal context, incident response, OWASP mapping, accepted risks |
| 2 | Performance budgets and capacity, testing strategy and CI gates, environments, production topology, deployment and migrations, backups and disaster recovery, monitoring and alerting, runtime and framework upgrades, operations runbooks |

---

## 12.1 New Decisions

| ID | Decision |
|---|---|
| DEC-186 | **Security baseline:** OWASP ASVS Level 2 for every portal and the API, with Level 3 controls for authentication, authorization of money actions, audit integrity and key management. Each release checks the controls in 12.14. |
| DEC-187 | **Four data classes** (12.3) with handling rules for storage, display, export, logs and backups. Every column in D7 that holds Restricted data is listed and tested. |
| DEC-188 | **Passwords:** bcrypt with cost 12 (Laravel default). Minimum length 8 for guardians and students, 10 for staff, 12 for holders of sensitive permissions and platform users. Staff and platform passwords are checked against known breached passwords (k-anonymity range query); if that service is unreachable the check passes and is logged. No forced periodic rotation; rotation on suspicion. |
| DEC-189 | **Session lifetimes:** staff idle 30 minutes (15 minutes for roles holding sensitive permissions), absolute 12 hours; guardians and students idle 7 days with "Remember this device" up to 30 days; platform idle 15 minutes, absolute 8 hours. Sessions regenerate on login, privilege change and portal switch. |
| DEC-190 | **Database privilege separation:** the application user has data access only (no DDL); `audit_logs` and `login_events` allow only `SELECT` and `INSERT` to it; a separate migration user is used only by the deploy job; reports and exports read through a read-only user on the replica; partition maintenance uses a maintenance user. |
| DEC-191 | **No user-uploaded SVG anywhere.** Logos are PNG, JPG or WebP (correction to D9 9.20). SVG icons come only from the repository. |
| DEC-192 | **Uploaded images are re-encoded** on upload (Intervention Image), which strips EXIF data including GPS location, and resized to the purpose's maximum. |
| DEC-193 | **Key rotation:** `APP_KEY` rotates yearly or on suspicion using Laravel's previous-keys support plus a re-encryption job; the lookup-hash pepper rotates only on suspicion, with a dual-hash period while a job rewrites every `*_lookup_hash`. |
| DEC-194 | **Hosting is provider-neutral and single-region.** Containers, MySQL, Redis-compatible cache and S3-compatible storage only, so the platform can move to a Pakistan-based data centre if a future law or a large customer requires it. All tenant data and backups stay in the chosen region. |
| DEC-195 | **Privacy defaults:** consent for SMS, WhatsApp and photo use is captured at admission; guardians exercise access and correction through existing portal flows; personal data breaches are reported to the affected organization within 24 hours of confirmation, with the facts it needs to inform families. |
| DEC-196 | **No payment card data ever reaches the platform.** Online payments use the providers' hosted pages or redirect flows only (D11 11.10). |
| DEC-197 | **Security events alert in real time** (12.10). Alerts go to the platform security channel and, for tenant-level events, to the organization's IT admin and owner. |

### Corrections to earlier deliverables

| Where | Change |
|---|---|
| D9 9.20 | Logo upload accepts PNG, JPG or WebP, not SVG (DEC-191). |
| D4 4.7 | On-demand TLS for custom domains is provided by the edge proxy defined in Part 2, which asks the application whether a domain is a verified `organization_domains` row before issuing a certificate. |

---

## 12.2 Security Objectives

| # | Objective | Measured by |
|---|---|---|
| SO-1 | No organization can read or change another organization's data | Two-tenant tests on every route (D4 4.24), composite foreign keys, zero cross-tenant incidents |
| SO-2 | Nobody sees or changes data beyond their role, scope and visibility | RBAC fixture tests (D5 5.14), policy coverage test |
| SO-3 | Money, marks and attendance cannot change without a trace | Append-only audit, revision tables, SOD, period locks, reconciliation (D11 11.13) |
| SO-4 | Children's personal data is protected in storage, on screen, in exports and in logs | Classification rules (12.3), masking tests, log redaction tests |
| SO-5 | The platform survives the loss of any single server and recovers from data loss within agreed limits | Part 2 RPO and RTO, restore drills |
| SO-6 | Security problems are detected and acted on quickly | Alerting (12.10), incident process (12.13) |

---

## 12.3 Data Classification (DEC-187)

| Class | Examples (D7 columns) | Storage | Display | Export | Logs | Backups |
|---|---|---|---|---|---|---|
| **Restricted** | CNIC and B-Form numbers (`*_encrypted`), student health notes, counseling notes, custody restriction details, employee bank account and IBAN, salary amounts, passwords, OTPs, 2FA secrets, API tokens, provider credentials | Field encryption (Laravel `encrypted` cast) or one-way hash; never in JSON blobs or search indexes | Masked by default; full view needs a sensitive permission and 2FA (DEC-62, DEC-63) | Masked unless the exporter holds the full-view permission; counseling never exported | Never logged; redaction processor drops them | Encrypted backup; same retention |
| **Confidential** | Student and guardian identity, contacts, addresses, photos, attendance, marks, results before publication, fees and payments, staff records, audit logs | Database with tenant scoping; files in private storage | By permission and visibility (D5) | By permission; masked fields stay masked | Identifiers only (ids, ULIDs), no names or numbers in messages | Encrypted backup |
| **Internal** | Settings, templates, timetables, calendars, published results inside the portal | Database | Staff and portal users with access | Allowed | Allowed | Encrypted backup |
| **Public** | Admission form, verification result (holder name, type, status), marketing pages | Database or static | Anyone | n/a | Allowed | Normal |

Children's data is Confidential at minimum; anything identifying a child plus a sensitive attribute (health, custody, counseling, discipline) is Restricted.

---

## 12.4 Threat Model

### Actors

| Actor | Motivation | Typical access |
|---|---|---|
| External attacker | Data theft, ransom, defacement, fraud | Internet only |
| Malicious or curious tenant user | See other students' data, change marks or fees, see another school's data | Valid account with some permissions |
| Compromised guardian account | Access a child's data, payment fraud | Guardian portal session, often on a shared phone |
| Insider at a school | Change marks, steal fee cash, hide absences | Staff account with legitimate permissions |
| Platform staff | Misuse of support access | Platform portal, support grants |
| Compromised integration | Fake punches, fake payment callbacks | Device or webhook endpoints |
| Supply-chain attacker | Malicious package version | Build pipeline |

### STRIDE summary

| Threat | Example | Main controls |
|---|---|---|
| Spoofing | Credential stuffing on guardian logins; fake payment callback | Rate limits and lockout (12.5), breached-password check for staff, OTP limits, provider verification API (D11 11.10), webhook signatures (12.7) |
| Tampering | Changing marks after approval; editing a paid voucher; altering audit rows | Marks submission locks (DEC-181), revisions with approval, immutable money rows (DEC-113), period locks, audit table INSERT-only privilege (DEC-190) |
| Repudiation | "I never reversed that payment" | Audit with actor, reason, IP, correlation id; approvals with names; impersonation recorded with both identities |
| Information disclosure | Another tenant's student via guessed id; CNIC in an export; data in logs | ULIDs, scoped binding with 404 (DEC-27), composite FKs (DEC-43), masking (D5 5.10), log redaction (12.10) |
| Denial of service | Result-day traffic flood; webhook flood; huge upload | Rate limits per user and organization, per-tenant job fairness (D3), upload size limits, CDN for assets, autoscaling (Part 2) |
| Elevation of privilege | Principal assigns themselves owner rights; teacher calls an admin route | Subset rule (DEC-55), SOD-07, policies on every route, route permission coverage test |

---

## 12.5 Identity and Authentication

| Control | Rule |
|---|---|
| Identifiers | Mobile (E.164), email or username per organization (DEC-9, DEC-10); login errors never reveal whether an account exists |
| Password policy | DEC-188; Laravel `Password` rules; no composition rules beyond length; paste allowed (WCAG 3.3.8) |
| Throttling | 5 attempts per minute per identifier plus IP (D8 limiter `auth`); 10 failures in 15 minutes locks the account for 15 minutes (`users.locked_until`); lockout notifies the user by SMS or email |
| First-time and reset codes | 6-digit OTP, stored hashed, valid 5 minutes, 5 attempts, resend after 60 seconds, 5 sends per hour per number; same response whether or not the number exists |
| Password reset links | Organization-aware token (DEC-96), single use, 60 minutes, invalidated by password change |
| Two-factor | TOTP through Fortify with recovery codes; required for sensitive-permission holders (DEC-63) and all platform users; enforced by `RequireTwoFactor`; 2FA reset by an admin needs a level 2 reason and notifies the user |
| Sessions | Database driver, host-only cookies (DEC-44), `Secure`, `HttpOnly`, `SameSite=Lax`; lifetimes DEC-189; regeneration on login, role change, portal switch; "sign out everywhere" (SCR-IAM-011) |
| New device notice | First login from a new device or country sends a notice to staff and platform users |
| Trusted devices | Required for staff online check-in; a device is trusted after OTP confirmation; removable by the user, HRM or IT admin |
| Impersonation | Only through an owner-approved support grant (D5, SCR-PLT-017); time-boxed; banner on every page (DEC-150); every action records both identities; impersonated sessions cannot view Restricted fields, change passwords or 2FA, or approve money requests |
| API tokens | DEC-133; stored hashed; abilities per portal; 90-day expiry; revoked on password change, deactivation, role removal |
| Deactivation | Ends sessions and tokens immediately; device enrollments ended for separated staff |

---

## 12.6 Authorization and Tenant Isolation

### Authorization rules

| Rule | Mechanism |
|---|---|
| Every route is authorized | Form Request `authorize()` or controller `authorize()` calling a policy; `RouteConventionsTest` (D8) fails any `app.*`, `guardian.*`, `student.*`, `platform.*` or `api.v1.*` route without a permission check, except listed baseline routes |
| Lists are scoped | `->visibleTo($user, $permission)` on every Query class (D3 3.8) |
| Records are scoped | Route model binding through global scopes; policy checks record branch and visibility |
| Money and marks actions | Separation of duties (D5 5.11) enforced in Actions, not only in UI |
| Field access | Masking helpers in resources and exports; Restricted fields decrypted only when the permission check passes |
| Portal boundary | A permission belongs to one portal (D5 5.6); guardian and student portals use locked permission sets |

### Tenant isolation summary (D4 4.15 has all 22 layers)

| Layer group | Key controls |
|---|---|
| Request | Host resolves the tenant (`ResolveTenant`); user must belong to it (`EnsureTenantMatchesUser`); host-only cookies |
| Data access | `OrganizationScope` fails closed (DEC-45); `AllowedBranchesScope`; tenant-scoped validation rules (`TenantExists`, `BranchExists`) |
| Database | `organization_id` on every tenant row; composite foreign keys (DEC-43); unique keys scoped by organization |
| Async | `TenantAware` job middleware; commands fan out per organization |
| Files and cache | Paths and cache keys prefixed by organization; signed downloads through scoped `stored_files` |
| Tests | Generated two-tenant test for every route and model |

---

## 12.7 Input, Output and Browser Security

| Area | Control |
|---|---|
| Validation | Form Requests for every write; normalization in `prepareForValidation()`; unknown fields ignored; Eloquent `$fillable` explicit and `Model::shouldBeStrict()` outside production |
| SQL injection | Eloquent and query builder with bindings; raw expressions only in Query classes with bindings (MB-08); arch test forbids string-built SQL |
| Output encoding | Blade `{{ }}` everywhere; `{!! !!}` allowed only for sanitized rich text through one helper, checked by `BladeConventionsTest` |
| Rich text | Symfony HTML Sanitizer allowlist (paragraphs, lists, bold, italic, links with `rel="noopener nofollow"`, tables); no scripts, styles, iframes or event attributes |
| Content Security Policy | DEC-30: `default-src 'self'`; `script-src 'self' 'nonce-{n}'`; `style-src 'self' 'nonce-{n}'`; `img-src 'self' data: blob:`; `connect-src 'self'`; `font-src 'self'`; `frame-ancestors 'none'`; `form-action 'self'` plus payment provider hosts on the payment page only; `base-uri 'none'`; `object-src 'none'`; reports to a CSP report endpoint |
| Other headers | `Strict-Transport-Security: max-age=31536000; includeSubDomains; preload` on the platform domain; `X-Content-Type-Options: nosniff`; `Referrer-Policy: strict-origin-when-cross-origin`; `Permissions-Policy: camera=(self), geolocation=(self), microphone=()` (camera for QR scan, geolocation for check-in) |
| CSRF | Laravel CSRF on every session route; webhooks and API excluded by middleware group (D8) |
| CORS | No browser origins allowed for the API (D8 8.7) |
| Open redirects | Redirect targets limited to named routes or same-host paths |
| Uploads | Extension and detected MIME allowlist per purpose (photos: JPG, PNG, WebP; documents: PDF, JPG, PNG; imports: XLSX, CSV); size limits by plan; random storage names; private disk; optional ClamAV scan before use (`scan_status`); images re-encoded (DEC-192); no SVG (DEC-191); downloads only through signed, short-lived, permission-checked URLs with `Content-Disposition: attachment` for documents |
| Spreadsheet exports | Cells starting with `=`, `+`, `-`, `@` are prefixed to prevent formula injection |
| PDF rendering | Gotenberg runs on the internal network only, receives HTML from templates, cannot fetch external URLs (assets are inlined), and has no access to the database or storage |
| Server-side requests | The application calls only configured provider endpoints; user input never chooses a URL to fetch |

---

## 12.8 Webhooks, Devices and Payments

| Control | Rule |
|---|---|
| Store first | Every inbound call is written to `webhook_receipts` before processing (DEC-42); processing is a queued job, so a slow provider never blocks |
| Authenticity | Provider signature (HMAC or provider-specific) verified with the organization's secret; timestamp within 5 minutes where the provider sends one; IP allowlist where the provider publishes ranges |
| Replay | Dedupe on provider and external id (`ix_whr_dedupe`) plus the target table's unique keys (D7 7.46) |
| Payment truth | A payment is posted only after the provider's verification API confirms status, amount and currency (D11 11.10, DEC-166); browser returns are never trusted |
| Card data | DEC-196; the platform never sees card numbers |
| Merchant credentials | Encrypted per organization; masked after save; test transaction before going live (SCR-FEE-034) |
| ZKTeco ADMS | Only registered, active serial numbers accepted; per-device rate limit; optional IP allowlist; strict parsing of punch fields only; responses contain no personal data. Plain HTTP on the ADMS host is an accepted risk (12.15) because many devices cannot speak TLS |
| Signed punch webhook | HMAC with per-device secret and timestamp; preferred over ADMS where devices support it |
| SMS delivery receipts | Signed or token-protected per provider; used only to update delivery status |

---

## 12.9 Secrets, Keys and Encryption

| Item | Rule |
|---|---|
| Where secrets live | Server environment from the deployment secret store; never in the repository, images or logs; `.env.example` holds names only |
| Tenant credentials | Encrypted with the application key (`encrypted:array` cast); never returned to the browser after save |
| `APP_KEY` | Unique per environment; rotation per DEC-193 using the previous-keys setting so existing ciphertext stays readable while `core:reencrypt` rewrites encrypted columns in batches |
| Lookup-hash pepper | Stored like `APP_KEY` (A-53); rotation per DEC-193 |
| TLS | TLS 1.2 minimum, 1.3 preferred, at the edge; HTTP redirects to HTTPS on every host except ADMS `/iclock/*` |
| Data at rest | Encrypted disks or managed-service encryption for database, storage and backups; backup archives additionally encrypted with a key held outside the backup location |
| Field encryption | Restricted columns (12.3) use the `encrypted` cast; search uses `*_lookup_hash` (HMAC-SHA256 with pepper) and display uses `*_last4` (DEC-95) |
| Internal traffic | Application to database, Redis and Gotenberg on a private network; database and Redis require authentication and TLS where the provider supports it |

---

## 12.10 Logging and Security Monitoring (DEC-197)

### Logging rules

| Rule | Detail |
|---|---|
| Format | JSON with correlation id, organization id, user id, route, status, duration (D8 `logging.php`) |
| Redaction | Processor removes passwords, OTPs, tokens, `Authorization` and cookie headers, CNIC and B-Form values, card-like numbers, bank account numbers; request bodies are not logged |
| Error tracking | Sentry (optional) with server-side scrubbing on; user context sends only ids |
| Retention | Application logs 30 days hot, 1 year archived; audit and login events per D7 7.50 |

### Security events and alerts

| Event | Threshold | Alert to | Severity |
|---|---|---|---|
| Failed logins for one organization | 50 in 10 minutes | Platform security, organization IT admin | High |
| Account lockouts | 10 in 10 minutes in one organization | Same | High |
| OTP sends to one number | Above hourly limit | Platform security | Medium |
| Cross-tenant 404 on bound records | 20 in 10 minutes from one user or IP | Platform security | High |
| Permission denied | 30 in 10 minutes from one user | Organization IT admin | Medium |
| Role or override granted with sensitive permissions | Every event | Organization owner | Medium |
| Two-factor disabled or reset | Every event | User and organization owner | Medium |
| Support impersonation started | Every event | Organization owner | Medium |
| Bulk export containing Restricted fields | Every event | Organization owner | Medium |
| Webhook signature failures | 10 in 10 minutes per provider and organization | Platform security | High |
| Unknown ADMS serial attempts | 20 in 1 hour | Platform security | Low |
| CSP violation reports | Spike above baseline | Platform engineering | Low |
| `MissingTenantContext` thrown | Every event | Platform engineering | Critical |
| Integrity check mismatch (DEC-126, D11 11.13) | Every event | Organization accountant, platform engineering | High |

---

## 12.11 Secure Development and Supply Chain

| Practice | Rule |
|---|---|
| Code review | Every change reviewed by another person; changes to auth, RBAC, tenancy, money posting or crypto need a second reviewer |
| Static checks | Larastan (level 6 rising to 8), Pint, architecture tests (MB rules, route permission coverage, forbidden code) on every push |
| Secret scanning | gitleaks in CI on every push and on the full history weekly |
| Dependencies | `composer audit` and `npm audit` fail CI on high or critical issues; Renovate or Dependabot weekly; package rule DEC-01 (PHP 8.2 to 8.4 and Laravel 12 and 13 until UG-1); lock files committed; no packages from unofficial sources |
| Build integrity | One image built in CI per commit and promoted unchanged from staging to production; images signed and scanned for OS vulnerabilities |
| Dynamic testing | OWASP ZAP baseline scan against staging for every release; authenticated scan of staff and guardian portals monthly |
| Penetration test | Independent test before the R1 production launch, then yearly and after major auth or payment changes |
| Threat model refresh | Each release reviews 12.4 for new modules and integrations |
| Security tests in CI | See Part 2: tenant isolation, RBAC fixture, masking, log redaction, CSP header, upload rules, webhook signature and replay, SOD paths |

---

## 12.12 Privacy (DEC-195)

### Legal context (as of November 2026)

Pakistan has no comprehensive, enacted personal data protection law; the Personal Data Protection Bill remains a draft. The Prevention of Electronic Crimes Act 2016 (amended in 2025) applies to unauthorized access and data misuse, and national cybersecurity guidance on protecting personal information was issued in August 2025. The platform is designed so that the likely obligations of a future law (lawful basis and consent, purpose limitation, data subject rights, breach notification, security safeguards and possible localization) can be met without redesign. Each organization remains responsible for its own legal compliance; the platform provides the controls. Legal review is required before launch and whenever the law changes.

### Privacy controls

| Principle | How the platform supports it |
|---|---|
| Purpose and minimization | Only fields needed for school operation are mandatory; optional fields are clearly optional; health and custody fields are restricted |
| Consent | Admission form captures guardian consent for SMS and WhatsApp messages and for use of the child's photo in school publicity; stored with time and version; changeable in the guardian portal (notification preferences, D10 SCR-NTF-002) |
| Transparency | Organization privacy notice shown on admission and sign-in pages (template provided, organization edits it) |
| Access | Guardians and students see their records in the portals; a full data export for a child can be produced by the school on request (SCR-STU-023 with permission) |
| Correction | Profile change requests (D11 `student_profile_correction`) |
| Deletion | Limited by record-keeping duties (GR register, results, fees); applicants who never enrolled are anonymized after the retention period (D7 7.48); tenant deletion follows DEC-51 after export |
| Children | Photos can be restricted by wing (D5 `student.photo.view_full`); no public listing of students; verification pages show only name, document type and status |
| Processors | SMS, email, WhatsApp, payment and hosting providers are listed in the platform's data processing terms; each organization's own provider credentials are used where possible |
| Breach notification | 12.13; organization notified within 24 hours of confirmation |

### Retention schedule (defaults, organization can lengthen)

| Data | Retention |
|---|---|
| Student core record, GR register, results, certificates | Permanent (school records) |
| Attendance daily records | Permanent; raw punches 13 months hot, 5 years archive (D7 7.50) |
| Fee vouchers, payments, journals | 10 years after the fiscal year closes |
| Applications not admitted | Anonymized after 2 years (D7 7.48) |
| Guardian portal access after withdrawal | Read-only for 1 year (A-38) |
| Audit logs | 24 months hot, 10 years archive (D7 7.50) |
| Login events | 12 months hot, 3 years archive |
| Notification content | Dispatch metadata 12 months; message body 6 months |
| Uploaded documents | With the record they belong to |

---

## 12.13 Incident Response

| Severity | Definition | First response | Updates |
|---|---|---|---|
| SEV-1 | Confirmed cross-tenant exposure, data breach, payment fraud, total outage | 15 minutes, all hands | Every hour to affected organizations |
| SEV-2 | Single-tenant data exposure, major feature down (attendance, fee collection, results) | 30 minutes | Every 2 hours |
| SEV-3 | Degraded performance, provider outage with workaround | 4 hours | Daily |
| SEV-4 | Minor issue | Next working day | On resolution |

### Process

```
Detect (alert, report, customer) -> Triage (severity, incident lead, scribe)
  -> Contain (disable account or token, revoke support grant, block IP, switch provider,
              put a module into maintenance for one tenant via entitlement override)
  -> Preserve evidence (logs, audit rows, webhook receipts; no destructive cleanup)
  -> Eradicate and recover (fix, restore, re-verify integrity checks)
  -> Notify (organization within 24 hours of confirming a personal data breach, with what,
             who, when, what was done, what families should do)
  -> Review within 5 working days (cause, timeline, actions with owners; threat model updated)
```

Runbooks for SEV-1 and SEV-2 scenarios live in `docs/runbooks/` (D8 8.13): suspected cross-tenant leak, compromised staff account, compromised platform account, payment callback fraud, SMS provider abuse, ransomware or data loss, leaked secret.

---

## 12.14 OWASP Top 10 Mapping

| OWASP category | Controls in this blueprint |
|---|---|
| A01 Broken access control | RBAC with visibility (D5), policies on every route, scoped binding and 404 (DEC-27), composite FKs (DEC-43), SOD, route coverage test |
| A02 Cryptographic failures | TLS everywhere, field encryption for Restricted data, hashed tokens and OTPs, key rotation (DEC-193) |
| A03 Injection | Eloquent bindings, sanitizer for rich text, CSP, formula-injection guard in exports |
| A04 Insecure design | Threat model, double-validated approvals (DEC-167), immutable money rows, period locks |
| A05 Security misconfiguration | Config cache, debug off in production, security headers, least-privilege database users (DEC-190), hardened images |
| A06 Vulnerable components | Audits in CI, update bot, package rule DEC-01, runtime upgrade plan (Part 2) |
| A07 Identification and authentication failures | Throttling, lockout, 2FA for sensitive roles, breached-password check, session rules (DEC-189) |
| A08 Software and data integrity failures | Signed images promoted unchanged, webhook signatures and verification APIs, lock files |
| A09 Security logging and monitoring failures | Append-only audit, security events and alerts (12.10), integrity checks |
| A10 Server-side request forgery | No user-chosen URLs; Gotenberg cannot fetch external resources; provider endpoints fixed in configuration |

---

## 12.15 Accepted Risks

| Risk | Why accepted | Mitigation |
|---|---|---|
| ZKTeco ADMS over plain HTTP | Many devices in use cannot do TLS | Separate host, serial allowlist, rate limits, punch-only parsing, no personal data in responses, signed webhook adapter preferred for new devices |
| SMS OTP can be intercepted through SIM swap | Guardians rarely have email or authenticator apps | OTP only for first login and reset; rate limits; staff with sensitive permissions must use TOTP |
| Shared family phones for guardian portal | Common in Pakistan | Guardian portal shows only linked children; sensitive actions (pickup person changes) notify the primary guardian by SMS |
| Teachers using personal phones without screen lock | Cannot be enforced on personal devices | Idle timeout, no Restricted data in teacher views, remote sign-out by IT admin |
| Breached-password service outage | External dependency | Check fails open and is logged; length rules still apply |
| Single region | Cost and simplicity for R1 | Off-site encrypted backups in a second location (Part 2) |

---

## Assumptions (Part 1)

| ID | Assumption |
|---|---|
| A-96 | Each organization accepts responsibility for its own privacy notice and lawful use of student data; the platform provides controls and templates. |
| A-97 | Payment providers used (JazzCash, Easypaisa, 1Bill aggregator) offer server-side status verification for every transaction. |
| A-98 | Hosting provider offers encrypted disks, private networking and managed MySQL or the ability to run it with the same controls. |

## Gaps Found and Added (Part 1)

| ID | Gap | Decision |
|---|---|---|
| GAP-241 | No measurable security baseline | ASVS Level 2 with Level 3 areas (DEC-186) |
| GAP-242 | Handling rules differed by screen and module | Data classification (DEC-187) |
| GAP-243 | Audit rows could be altered by the application's own database user | INSERT-only privilege on audit tables (DEC-190) |
| GAP-244 | SVG logos could carry scripts | No user SVG (DEC-191) |
| GAP-245 | Photos could leak home location through EXIF | Re-encode images (DEC-192) |
| GAP-246 | No key rotation procedure | DEC-193 |
| GAP-247 | Data residency not decided | Provider-neutral single region (DEC-194) |
| GAP-248 | No consent capture or breach notification commitment | DEC-195 |
| GAP-249 | Security events had no owners or thresholds | Alert table (DEC-197, 12.10) |
| GAP-250 | Impersonated sessions could act on Restricted data or money approvals | Impersonation limits (12.5) |

---

Reply CONTINUE for Deliverable 12, Part 2
