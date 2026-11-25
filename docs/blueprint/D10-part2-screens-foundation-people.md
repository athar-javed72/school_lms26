# Deliverable 10: Role-Wise Dashboards and Screen Inventory
# Part 2 of 3: Screen Inventory for Platform, Foundation, Academic, People, Attendance and Leave

**Version:** 1.0 (draft for approval) | **Date:** 24 Nov 2026
**Conventions:** columns as defined in Part 1, 10.2. Route names use snake_case segments (DEC-128); URLs use the kebab-case form of the same segments. Role abbreviations follow D5 5.4.

---

## 10.10 New Decisions

| ID | Decision |
|---|---|
| DEC-160 | **Record history is one shared drawer.** Every detail page of an audited record has a "History" action that opens `SCR-AUD-003` (record history drawer), filtered to that record. It is permission-checked with `audit.record_history.view` and the source record's own view permission. Modules never build their own history screens. |
| DEC-161 | **Profile tabs belong to the module that owns their data.** The student, guardian and employee profile pages are shells; each tab is a screen of its owning module, registered through the profile tab registries (D3 3.23). A tab screen keeps its owner's ID (for example the student fee tab is `SCR-FEE-030`, listed in Part 3). |
| DEC-162 | **The attendance marking grid saves per row and per sheet.** Each status change autosaves with the sheet's idempotency key and version; "Submit" finalizes the sheet. If two people edit the same section, the second save of a changed row shows the conflict banner (DEC-136) for that row only. Offline queueing through a service worker is R3 (D1 1.12); R1 and R2 retry and show the offline banner (D9 9.16). |

### Correction to Part 1

Part 1 route names in 10.4 and 10.6 have been corrected to snake_case segments (for example `app.attendance.section_sheets.index`), matching DEC-128. The project copy of Part 1 is updated.

---

## 10.11 Platform (PLT), platform portal unless noted

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-PLT-001 | Platform home | dashboard | `platform.home` | Platform role | SA, support, billing | R1 | Widgets in Part 1, 10.6 |
| SCR-PLT-002 | Platform sign-in | public | `auth.platform.login` | none | Platform users | R1 | Separate host; TOTP always required |
| SCR-PLT-003 | Organizations | list | `platform.platform.organizations.index` | `platform.organization.view` | SA, support, billing | R1 | Filters: state, plan, created; search by name, slug, phone; never shows tenant records, only counts |
| SCR-PLT-004 | Organization detail | detail | `platform.platform.organizations.show` | `platform.organization.view` | SA, support, billing | R1 | Profile, domains, plan, modules, usage, support history, state timeline |
| SCR-PLT-005 | Provision organization | wizard | `platform.platform.organizations.create` | `platform.organization.create` | SA | R1 | Steps: organization, owner user, slug (reserved slug check), plan, modules, review; creates owner invite SMS |
| SCR-PLT-006 | Change organization state | modal | `platform.platform.organizations.state` | `platform.organization.manage` | SA | R1 | Trial, active, suspended, archived; level 3 confirm with reason; suspended tenants see a read-only notice |
| SCR-PLT-007 | Domains | detail | `platform.platform.organization_domains.index` | `platform.organization.update` | SA | R1 (custom R2) | Subdomain; custom domain verification status, TLS status |
| SCR-PLT-008 | Module entitlements | form | `platform.platform.module_entitlements.edit` | `platform.module_entitlement.update` | SA, billing | R1 | Plan modules plus overrides with expiry; shows dependency warnings from the module manifest |
| SCR-PLT-009 | Plans | list | `platform.platform.plans.index` | `platform.plan.view` | SA, billing | R1 | |
| SCR-PLT-010 | Plan form | form | `platform.platform.plans.edit` | `platform.plan.update` | SA, billing | R1 | Limits (students, branches, storage, SMS credits), modules, public flag |
| SCR-PLT-011 | Usage | report | `platform.platform.usage.index` | `platform.usage.view` | SA, billing | R1 | Per organization per month; export |
| SCR-PLT-012 | Support access requests | list | `platform.platform.support_access.index` | `platform.support_access.view` | SA, support | R1 | Requested, granted, expired, revoked |
| SCR-PLT-013 | Request support access | modal | `platform.platform.support_access.create` | `platform.support_access.create` | SA, support | R1 | Reason, target user, duration; the owner approves in the tenant (SCR-PLT-017) |
| SCR-PLT-014 | Start impersonation | modal | `platform.platform.support_access.impersonate` | `platform.support_access.impersonate` | SA, support | R1 | Only with an active grant; opens the tenant host with the impersonation banner (D9) |
| SCR-PLT-015 | Platform announcements | list, form | `platform.platform.announcements.index` | `platform.announcement.view` | SA | R1 | Audience: all, plans, organizations; schedule; shows in tenant top bar |
| SCR-PLT-016 | System health | dashboard | `platform.platform.health.show` | `platform.organization.view` | SA, support | R1 | Queues, failed jobs (retry, discard), Gotenberg, providers, webhook failures, partition status |
| SCR-PLT-017 | Support access grants (tenant side) | list | `app.platform.support_access_grants.index` | `platform.support_access.approve` | OW | R1 | Approve or reject requests, see active sessions, end a session |
| SCR-PLT-018 | Platform users and roles | list, form | `platform.identity.platform_users.index` | Platform super admin | SA | R1 | Three platform roles (DEC-61); 2FA required |
| SCR-PLT-019 | Platform audit | list | `platform.audit.logs.index` | Platform super admin | SA | R1 | Platform actions, impersonations, state changes |
| SCR-PLT-020 | Tenant export | wizard | `platform.platform.tenant_exports.create` | `platform.organization.manage` | SA | R2 | Builds a full export (D4 4.20); download link with expiry |

## 10.12 Organization (ORG)

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-ORG-001 | Onboarding wizard | wizard | `app.organization.onboarding.show` | `organization.onboarding.manage` | OW, OA | R1 | Steps from `OnboardingStepRegistry` (D7 7.52); resume later; progress shown on dashboard |
| SCR-ORG-002 | Organization profile | detail, form | `app.organization.profile.show` | `organization.profile.view` | MGT, ACT, HRM, ITA | R1 | Name (en, ur), registration numbers, address, timezone, currency, head-office branch |
| SCR-ORG-003 | Structure overview | detail | `app.organization.structure.show` | `organization.institution.view` | MGT, ITA | R1 | Tree: organization, institutions, branches, departments; counts per node |
| SCR-ORG-004 | Institutions | list, form | `app.organization.institutions.index` | `organization.institution.view` | MGT, ITA | R1 | Type (school, college), tracks offered, board affiliation, logo |
| SCR-ORG-005 | Branches | list | `app.organization.branches.index` | `organization.branch.view` | MGT, ITA, HRM | R1 | Status, institution, city, student count |
| SCR-ORG-006 | Branch form | form | `app.organization.branches.edit` | `organization.branch.update` | OW, OA | R1 | Code, address, geofence (lat, long, radius) for online check-in, phone, principal shown read-only from role assignments |
| SCR-ORG-007 | Deactivate branch | modal | `app.organization.branches.deactivate` | `organization.branch.manage` | OW | R1 | Blocked while active enrollments exist; level 3 confirm |
| SCR-ORG-008 | Branch bank accounts | list, form | `app.organization.bank_accounts.index` | `organization.bank_account.view` | OW, OA, PR, ACT | R1 | Default for vouchers; account number masked after save |
| SCR-ORG-009 | Departments | list, form | `app.organization.departments.index` | `organization.department.view` | MGT, HRM, HOD | R1 | Head comes from HR `department_heads` (SCR-HR-012) |
| SCR-ORG-010 | Shifts | list, form | `app.organization.shifts.index` | `organization.shift.view` | MGT, HRM, AC | R1 | Times, working days |
| SCR-ORG-011 | Rooms | list, form | `app.organization.rooms.index` | `organization.room.view` | MGT, AC, EXC | R1 | Type, capacity; used by timetable and exams |

## 10.13 Identity (IAM)

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-IAM-001 | Sign in | public | `auth.login` | none | Everyone | R1 | Mobile, email or username; language switch; rate limited; lockout message names the wait time |
| SCR-IAM-002 | First-time code request | public | `auth.otp.request` | none | Guardians, staff invited by mobile | R1 | Sends SMS OTP to a known number only; same response whether or not the number exists |
| SCR-IAM-003 | Verify code and set password | public | `auth.otp.verify` | none | Same | R1 | Paste allowed; resend timer; password rules shown live |
| SCR-IAM-004 | Forgot password | public | `auth.password.request` | none | Everyone | R1 | Email or SMS reset link |
| SCR-IAM-005 | Reset password | public | `auth.password.reset` | none | Everyone | R1 | Organization-aware token (DEC-96) |
| SCR-IAM-006 | Two-factor challenge | public | `auth.two_factor.challenge` | none | 2FA users | R1 | TOTP or recovery code |
| SCR-IAM-007 | Portal switcher | modal | `app.identity.portal_switch` | Baseline | Staff who are also guardians | R1 | DEC-47; also in guardian portal |
| SCR-IAM-008 | My profile | detail, form | `{portal}.identity.profile.show` | Baseline | All users | R1 | Name, photo, language, Urdu font (DEC-142), theme (R2); protected fields go through profile correction |
| SCR-IAM-009 | Change password | form | `{portal}.identity.password.edit` | Baseline | All users | R1 | Current password required; signs out other sessions on request |
| SCR-IAM-010 | Two-factor setup | wizard | `{portal}.identity.two_factor.edit` | Baseline | All users | R1 | QR, confirm code, recovery codes download; required banner for sensitive permission holders (DEC-63) |
| SCR-IAM-011 | My sessions and devices | list | `{portal}.identity.sessions.index` | Baseline | All users | R1 | Sign out a session; remove trusted device |
| SCR-IAM-012 | Users | list | `app.identity.users.index` | `identity.user.view` | OW, OA, PR, BM, ITA, HRM | R1 | Filters: status, role, branch, portal; shows last login; locked badge |
| SCR-IAM-013 | User form | form | `app.identity.users.edit` | `identity.user.create` / `.update` | OW, OA, ITA, HRM | R1 | Staff users only; links to employee record; sends invite |
| SCR-IAM-014 | User detail | detail | `app.identity.users.show` | `identity.user.view` | Same as list | R1 | Tabs: role assignments, overrides, effective permissions, sessions, login history |
| SCR-IAM-015 | Unlock or reset user | modal | `app.identity.users.manage` | `identity.user.manage` | OA, ITA | R1 | Unlock, force password change, reset 2FA (level 2 with reason) |
| SCR-IAM-016 | Deactivate user | modal | `app.identity.users.deactivate` | `identity.user.delete` | OW, OA, ITA | R1 | Ends sessions and tokens; blocked for the last owner |
| SCR-IAM-017 | Import users | wizard | `app.import_export.import_jobs.create` (type `users`) | `identity.user.import` | OW, OA, ITA | R1 | Uses SCR-IMX-002 |
| SCR-IAM-018 | Roles | list | `app.identity.roles.index` | `identity.role.view` | OW, OA, ITA | R1 | System and custom roles; "customized" badge; template version |
| SCR-IAM-019 | Role permission editor | grid | `app.identity.roles.edit` | `identity.role.update` | OW, OA | R1 | Module groups, resources as rows, actions as columns, visibility per permission; S and † markers; locked roles read-only; diff before save |
| SCR-IAM-020 | Create custom role | form | `app.identity.roles.create` | `identity.role.create` | OW, OA | R1 | Clone a template or start empty |
| SCR-IAM-021 | Reset role to template | modal | `app.identity.roles.reset` | `identity.role.update` | OW, OA | R1 | Shows permissions that will be added and removed |
| SCR-IAM-022 | Assign role | drawer | `app.identity.role_assignments.create` | `identity.role_assignment.assign` | OW, OA, PR | R1 | Role, scope (organization, institution, branches), visibility, dates, reason; subset rule (DEC-55) explains blocked roles |
| SCR-IAM-023 | Permission overrides | list, drawer | `app.identity.permission_overrides.index` | `identity.permission_override.view` | OW, OA, ITA | R1 | Grant or deny, scope, expiry, reason required |
| SCR-IAM-024 | Effective permissions viewer | detail | `app.identity.effective_permissions.show` | `identity.effective_permission.view` | OW, OA, ITA | R1 | Per branch: every permission, its visibility and its source (role, assignment, override) |
| SCR-IAM-025 | Active sessions (all users) | list | `app.identity.sessions_admin.index` | `identity.session.manage` | OA, ITA | R1 | Force sign-out |
| SCR-IAM-026 | Trusted devices (staff) | list | `app.identity.trusted_devices.index` | `identity.trusted_device.view` | ITA, HRM, OA | R1 | Remove a device used for online check-in |
| SCR-IAM-027 | API tokens | list, form | `app.identity.api_tokens.index` | `identity.api_token.view` | OW, OA, ITA | R2 | Device tokens for the sync agent (`device:punch`); token shown once |

## 10.14 Settings (SET)

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-SET-001 | Settings hub | list | `app.settings.settings.index` | `settings.setting.view` | OW, OA, PR | R1 | Categories from `SettingRegistry`; level selector (organization, institution, branch) shows inherited and overridden values |
| SCR-SET-002 | Settings category | form | `app.settings.settings.edit` | `settings.setting.update` | OW, OA; PR (branch level) | R1 | Each field shows where its value comes from; "Use inherited" resets an override |
| SCR-SET-003 | Branding | form | `app.settings.branding.edit` | `settings.branding.update` | OW, OA | R1 | Logo, brand colour with live preview and contrast gates (D9 9.20) |
| SCR-SET-004 | Language and formats | form | `app.settings.language.edit` | `settings.language.update` | OW, OA | R1 | Default language, number grouping, date format, currency label |
| SCR-SET-005 | Terminology | list, form | `app.settings.terminology.index` | `settings.terminology.update` | OW, OA | R1 | Override any translation key per language (for example "Class" to "Grade"); preview |
| SCR-SET-006 | Numbering formats | form | `app.core.numbering.edit` | `core.numbering.manage` | OW, OA | R1 | Pattern per document type (vouchers, receipts, GR numbers); next number preview; cannot reduce counters |

## 10.15 Core (CORE)

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-CORE-001 | Period locks | list | `app.core.period_locks.index` | `core.period_lock.view` | OW, OA, PR, ACT, HRM | R1 | Per module and month: locked, unlocked until, unlock request status |
| SCR-CORE-002 | Lock period | modal | `app.core.period_locks.store` | `core.period_lock.lock` | PR, HRM, ACT | R1 | Checks open items (for example unfinalized attendance) before locking |
| SCR-CORE-003 | Request unlock | modal | `app.core.period_locks.unlock_request` | `core.period_lock.lock` | PR, HRM, ACT | R1 | Reason, duration; creates `period_unlock` approval |
| SCR-CORE-004 | Bulk operation progress | detail | `app.core.bulk_operations.show` | Baseline (own) or `core.bulk_operation.view` | Initiator; OA, ITA | R1 | Live progress, failed items, result file download |
| SCR-CORE-005 | My bulk operations | list | `app.core.bulk_operations.index` | Baseline | All staff | R1 | |

## 10.16 Audit (AUD)

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-AUD-001 | Audit log | list | `app.audit.logs.index` | `audit.log.view` | OW, OA, ITA, PR | R1 | Filters: user, module, action key, record, date, IP; sensitive values masked (D5 5.10) |
| SCR-AUD-002 | Audit entry | drawer | `app.audit.logs.show` | `audit.log.view` | Same | R1 | Before and after diff, reason, request context, correlation id |
| SCR-AUD-003 | Record history | drawer | `app.audit.record_history.show` | `audit.record_history.view` + record view permission | MGT, ACT, HRM, EXC | R1 | DEC-160; used on every audited detail page |
| SCR-AUD-004 | Login history | list | `app.audit.login_history.index` | `audit.login_history.view` | OW, OA, ITA | R1 | Success, failure, lockout, 2FA, impersonation |
| SCR-AUD-005 | Export audit log | modal | `app.audit.logs.export` | `audit.log.export` | OW, ITA | R1 | Queued export with masking |

## 10.17 Approval (APR)

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-APR-001 | Approvals inbox | list | `app.approval.inbox.index` | Baseline (assigned) | All staff approvers | R1 | Tabs: waiting for me, delegated to me, done by me; filters: type, branch, age; bulk approve where allowed |
| SCR-APR-002 | Request detail | detail | `app.approval.requests.show` | Baseline (requester, assignee) or `approval.request.view` | Requester, approvers, MGT | R1 | Payload rendered by the type's handler, attachments, timeline, decision panel (approve, reject, return) with comment |
| SCR-APR-003 | My requests | list | `{portal}.approval.my_requests.index` | Baseline | All users | R1 | Status per step; withdraw while pending |
| SCR-APR-004 | All requests | list | `app.approval.requests.index` | `approval.request.view` | MGT | R1 | Oversight with escalation status |
| SCR-APR-005 | Bulk approve | modal | `app.approval.requests.bulk_approve` | `approval.request.approve` | OW, PR, HRM | R1 | Only types marked bulk-approvable; one comment for all; SOD skips listed |
| SCR-APR-006 | Workflows | list | `app.approval.workflows.index` | `approval.workflow.view` | OW, OA, PR | R1 | One per request type and scope; default badge |
| SCR-APR-007 | Workflow editor | form | `app.approval.workflows.edit` | `approval.workflow.update` | OW, OA | R1 | Ordered steps with resolver, condition (amount, days), mode (any, all), SLA (R2); validation that each step's resolver can hold the approve permission (DEC-54) |
| SCR-APR-008 | Delegations | list, form | `app.approval.delegations.index` | Baseline (own) or `approval.delegation.view` | All approvers; OA | R1 | From, to, dates, request types |
| SCR-APR-009 | Guardian requests | list | `guardian.approval.my_requests.index` | Baseline | GDN | R1 | Leave, profile changes, document uploads; also child requests awaiting the guardian's approval |

## 10.18 Notification (NTF)

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-NTF-001 | Notification centre | list | `{portal}.notification.notifications.index` | Baseline | All users | R1 | Unread first, mark all read, links to source |
| SCR-NTF-002 | Notification preferences | form | `{portal}.notification.preferences.edit` | Baseline | All users | R1 | Per event group and channel; mandatory events (fees, absence, OTP) cannot be switched off |
| SCR-NTF-003 | Templates | list | `app.notification.templates.index` | `notification.template.view` | OW, OA | R1 | Per event, channel and language; customized badge |
| SCR-NTF-004 | Template editor | form | `app.notification.templates.edit` | `notification.template.update` | OW, OA | R1 | Placeholders list, live preview, SMS segment counter (Unicode for Urdu), reset to default |
| SCR-NTF-005 | Rules | list, form | `app.notification.rules.index` | `notification.rule.view` | OW, OA | R1 | Event on or off, channels, recipients, quiet hours |
| SCR-NTF-006 | Providers | list, form | `app.notification.providers.index` | `notification.provider.view` | OW, OA, ITA | R1 (WhatsApp R2) | Credentials masked after save; priority order |
| SCR-NTF-007 | Test send | modal | `app.notification.providers.test` | `notification.provider.manage` | OA, ITA | R1 | Sends to the user's own number or email |
| SCR-NTF-008 | Delivery log | list | `app.notification.delivery_logs.index` | `notification.delivery_log.view` | OW, OA, ITA, PR | R1 | Status, attempts, provider response, cost; resend failed |
| SCR-NTF-009 | Send broadcast | form | `app.notification.broadcasts.create` | `notification.broadcast.create` | MGT | R1 | Audience builder, channel, preview, estimated SMS count and segments; publish needs `notification.broadcast.publish` |

## 10.19 ImportExport (IMX)

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-IMX-001 | Import history | list | `app.import_export.import_jobs.index` | `import_export.history.view` | OW, OA, ITA, PR | R1 | Type, status, rows, errors, user |
| SCR-IMX-002 | New import | wizard | `app.import_export.import_jobs.create` | The type's import permission (D5 5.7.23) | Per type | R1 | Choose type, download template, upload, map columns (saved mappings), validate preview, commit; see wireframe W-05 |
| SCR-IMX-003 | Import job detail | detail | `app.import_export.import_jobs.show` | Initiator or `import_export.history.view` | Same | R1 | Row results, error file download, rollback not offered (records are created through actions and audited) |
| SCR-IMX-004 | Saved mappings | list | `app.import_export.mappings.index` | Any import permission | Importers | R1 | Rename, delete |
| SCR-IMX-005 | Exports | list | `app.import_export.export_jobs.index` | Baseline (own) | All staff | R1 | Queued exports with expiry; download |

## 10.20 Document (DOC)

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-DOC-001 | Document templates | list | `app.document.templates.index` | `document.template.view` | OW, OA, PR | R1 | Per document type and institution |
| SCR-DOC-002 | Template settings | form | `app.document.templates.edit` | `document.template.update` | OW, OA | R1 | Header, footer, signatures, paper, language, fields to show; PDF preview with sample data |
| SCR-DOC-003 | Template designer | grid | `app.document.templates.design` | `document.template.update` | OW, OA | R3 | Block-based layout editor |
| SCR-DOC-004 | Issued documents register | list | `app.document.issued.index` | `document.issued.view` | MGT, ADO | R1 | Serial, type, student, issued by, status; reissue, revoke |
| SCR-DOC-005 | Reissue or revoke | modal | `app.document.issued.manage` | `document.issued.issue` / `.manage` | PR, BM / OW, PR | R1 | Reason required; revoked documents fail verification |
| SCR-DOC-006 | Verify document | public | `public.document.verify` | none | Anyone | R2 | Code or QR; shows type, holder name, issue date, valid or revoked; no other personal data |

## 10.21 Academic (ACD)

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-ACD-001 | Sessions | list | `app.academic.sessions.index` | `academic.session.view` | STAFF | R1 | Per institution and track; status planning, active, closing, closed |
| SCR-ACD-002 | Session form | form | `app.academic.sessions.edit` | `academic.session.update` | OW, OA, PR | R1 | Dates, track, working days; Cambridge defaults Aug to Jul (DEC-87) |
| SCR-ACD-003 | New session wizard | wizard | `app.academic.sessions.wizard` | `academic.session.create` | OW, OA, PR | R1 | D6: copy classes, sections, subjects, fee structures, timetable bells; review; creates the session in planning |
| SCR-ACD-004 | Activate session | modal | `app.academic.sessions.activate` | `academic.session.manage` | OW, PR | R1 | One active per institution and track (DEC-79) |
| SCR-ACD-005 | Close session checklist | detail | `app.academic.sessions.close` | `academic.session.manage` | OW, PR | R1 | Checks from `SessionCloseCheckRegistry` (results published, promotions done, fee carry-forward); close blocked until green; level 3 confirm |
| SCR-ACD-006 | Education levels and progressions | list, form | `app.academic.education_levels.index` | `academic.education_level.view` | STAFF | R1 | National and Cambridge levels, Urdu names, next level per track (DEC-80) |
| SCR-ACD-007 | Study groups and elective buckets | list, form | `app.academic.study_groups.index` | `academic.study_group.view` | STAFF | R1 | Compulsory and elective subjects, "choose exactly one" buckets, option block labels |
| SCR-ACD-008 | Subjects | list, form | `app.academic.subjects.index` | `academic.subject.view` | STAFF | R1 | Code, names, type, external syllabus codes (Cambridge), import |
| SCR-ACD-009 | Classes | list | `app.academic.classes.index` | `academic.class.view` | STAFF | R1 | Per session: level, group, sections, strength |
| SCR-ACD-010 | Class detail | detail | `app.academic.classes.show` | `academic.class.view` | STAFF | R1 | Sections, class subjects, teachers, timetable link |
| SCR-ACD-011 | Sections | list, form | `app.academic.sections.index` | `academic.section.view` | STAFF | R1 | Capacity, room, shift, class teacher (one per section, DEC-75) with history |
| SCR-ACD-012 | Class subjects | grid | `app.academic.class_subjects.edit` | `academic.class_subject.update` | PR, AC | R1 | Subjects per class with periods per week, marks scheme, elective flags |
| SCR-ACD-013 | Teaching assignments | grid | `app.academic.teaching_assignments.index` | `academic.teaching_assignment.view` | MGT, AC, HOD, EXC, TCH (own) | R1 | Matrix of section by subject with teacher; unassigned cells highlighted; assign needs `.assign` |
| SCR-ACD-014 | Teacher workload | report | `app.academic.workload.index` | `academic.workload.view` | MGT, AC, HOD | R1 | Periods per teacher against limit |
| SCR-ACD-015 | Teaching groups | list | `app.academic.teaching_groups.index` | `academic.teaching_group.view` | MGT, AC, HOD, EXC, TCH | R1 | Cambridge sets across sections (DEC-81) |
| SCR-ACD-016 | Teaching group detail | detail, grid | `app.academic.teaching_groups.show` | `academic.teaching_group.view` | Same | R1 | Members (add, move), teacher; generate from option blocks |
| SCR-ACD-017 | My classes | list | `app.academic.my_classes.index` | `academic.teaching_assignment.view` (own) | TCH | R1 | My sections and groups with quick links: attendance, marks, diary, students |

## 10.22 Admission (ADM)

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-ADM-001 | Applications | list | `app.admission.applications.index` | `admission.application.view` | MGT, ADO, REC | R1 | Stage filter, class, date, source (counter, online); age since last action |
| SCR-ADM-002 | New application | form | `app.admission.applications.create` | `admission.application.create` | ADO, REC | R1 | From the form template; duplicate check on B-Form and guardian CNIC with "possible match" panel (D7 7.46); see wireframe W-03 |
| SCR-ADM-003 | Application detail | detail | `app.admission.applications.show` | `admission.application.view` | MGT, ADO, REC | R1 | Stage timeline, documents, test result, voucher status, actions by stage |
| SCR-ADM-004 | Approve or reject | modal | `app.admission.applications.decide` | `admission.application.approve` / `.reject` | PR, ADO | R1 | Class and section offered; reason for rejection |
| SCR-ADM-005 | Admission window override | modal | `app.admission.applications.override_window` | `admission.application.override_window` | OW, PR | R1 | Late admission with reason |
| SCR-ADM-006 | Convert to student | wizard | `app.admission.applications.convert` | `admission.application.convert` | ADO, BM | R1 | Confirms student, guardians, family, section, fee structure; issues admission voucher; assigns GR and registration numbers |
| SCR-ADM-007 | Seats | report | `app.admission.seats.index` | `admission.seat.view` | MGT, ADO, REC | R1 | Capacity, enrolled, offered, available per class and section |
| SCR-ADM-008 | GR register | report, print | `app.admission.gr_register.index` | `admission.gr_register.view` | PR, BM, ADO | R1 | Bound-register layout print |
| SCR-ADM-009 | Admission form templates | list, form | `app.admission.form_templates.index` | `admission.form_template.view` | MGT, ADO | R1 | Field sets per institution and level, required documents |
| SCR-ADM-010 | Online application form | public | `public.admission.applications.create` | none | Public | R1 | Captcha alternative rules (D9 9.13); document upload; reference number by SMS |
| SCR-ADM-011 | Application status | public | `public.admission.applications.status` | none | Applicant | R1 | Reference number plus mobile OTP |
| SCR-ADM-012 | Inquiries | list, form | `app.admission.inquiries.index` | `admission.inquiry.view` | MGT, ADO, REC | R2 | Follow-up date, outcome, convert to application |
| SCR-ADM-013 | Assessments | list, grid | `app.admission.assessments.index` | `admission.assessment.view` | PR, AC, ADO | R2 | Test or interview schedule, marks entry, result |

## 10.23 Student (STU)

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-STU-001 | Students | list | `app.student.students.index` | `student.student.view` | MGT, ADO, REC, ACT, CSH, EXC, AC, HOD, TCH, LBR, TM | R1 | Visibility applied (teachers see assigned only); filters: class, section, status, category, track; search by name, reg no, GR, guardian mobile |
| SCR-STU-002 | Student profile | detail | `app.student.students.show` | `student.student.view` | Same | R1 | Header with identity and status; tabs from registry (DEC-161); see wireframe W-02 |
| SCR-STU-003 | Edit student | form | `app.student.students.edit` | `student.student.update` | ADO, BM, PR | R1 | Non-protected fields; protected fields locked with "Change protected field" action |
| SCR-STU-004 | Change protected field | modal | `app.student.students.update_protected` | `student.profile.update_protected` | PR, BM | R1 | Name, date of birth, B-Form, father name; reason and supporting document |
| SCR-STU-005 | Student documents | list | `app.student.documents.index` | `student.document.view` | MGT, ADO, CT | R1 | Profile tab; verify action |
| SCR-STU-006 | Verify document | modal | `app.student.documents.verify` | `student.document.verify` | ADO, BM | R1 | Approve or reject with reason |
| SCR-STU-007 | Health profile | form | `app.student.health.edit` | `student.health.view` / `.update` | PR, VP, BM, CT / PR, BM | R1 | Sensitive (2FA); encrypted notes |
| SCR-STU-008 | Change section or class | modal | `app.student.enrollments.change` | `student.enrollment.manage` | PR, VP, BM, AC | R1 | Effective date; keeps history; recalculates teaching group membership |
| SCR-STU-009 | Promotions | list | `app.student.promotions.index` | `student.promotion.view` | MGT, AC | R1 | Promotion runs by class with status |
| SCR-STU-010 | Promotion wizard | wizard | `app.student.promotions.create` | `student.promotion.process` | PR, AC | R1 | Source class, eligibility from Result, decisions per student (promote, conditional, detain, graduate, progress to institution), target sections, preview, run as bulk job; see wireframe W-04 |
| SCR-STU-011 | Transfers | list | `app.student.transfers.index` | `student.transfer.view` | MGT, ADO | R1 | Branch transfers with approval status |
| SCR-STU-012 | Request branch transfer | form | `app.student.transfers.create` | `student.transfer.create` | PR, BM | R1 | Target branch, section, fee policy (D4 4.12) |
| SCR-STU-013 | Withdrawals | list | `app.student.withdrawals.index` | `student.withdrawal.view` | MGT, ADO, ACT | R1 | |
| SCR-STU-014 | Request withdrawal | form | `app.student.withdrawals.create` | `student.withdrawal.create` | BM, ADO | R1 | Reason, last date; clearance checks (fee, library, transport) shown live |
| SCR-STU-015 | Struck off and readmission | list, modal | `app.student.struck_off.index` | `student.struck_off.view` / `.manage` | PR, BM, ACT / PR | R1 | Candidates from fee rules; strike off or readmit with readmission voucher |
| SCR-STU-016 | Certificates | list | `app.student.certificates.index` | `student.certificate.view` | MGT, ADO | R1 | Leaving, character, bonafide, provisional; serial numbers |
| SCR-STU-017 | Issue certificate | form | `app.student.certificates.create` | `student.certificate.issue` | BM, ADO | R1 | Preview PDF, clearance status for leaving certificate |
| SCR-STU-018 | Timeline | detail | `app.student.timeline.show` | `student.timeline.view` | MGT, ADO, CT | R1 | Profile tab; every lifecycle event (D6 catalog) |
| SCR-STU-019 | Profile change requests | list | `app.student.modification_requests.index` | `student.modification_request.view` | MGT, CT | R1 | From guardians and students; approve through approval flow |
| SCR-STU-020 | Bulk actions | wizard | `app.student.bulk_actions.create` | `student.bulk_action.manage` | PR, BM, ADO | R1 | Change category, assign section, send SMS, print ID cards; preview count before run |
| SCR-STU-021 | Student categories | list, form | `app.student.categories.index` | `student.category.view` | MGT, ACT, ADO | R1 | |
| SCR-STU-022 | Sponsors | list, form | `app.student.sponsors.index` | `student.sponsor.view` | MGT, ACT, ADO | R1 | |
| SCR-STU-023 | Student list print and export | print | `app.student.students.export` | `student.student.export` | MGT, ADO | R1 | Masked per user (D5 5.10) |
| SCR-STU-024 | Student portal: my profile | detail | `student.student.profile.show` | `student.student.view` (self) | STD | R1 | Request a change |
| SCR-STU-025 | Guardian portal: child profile | detail | `guardian.student.profile.show` | `student.student.view` (linked) | GDN | R1 | Request a change, upload document |
| SCR-STU-026 | Portal: request profile change | form | `{portal}.student.modification_requests.create` | `student.modification_request.create` | GDN, STD | R1 | Field, new value, document |

## 10.24 Guardian (GRD)

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-GRD-001 | Guardians | list | `app.guardian.guardians.index` | `guardian.guardian.view` | MGT, ADO, ACT, CSH | R1 | Search by name, CNIC (hash lookup), mobile |
| SCR-GRD-002 | Guardian profile | detail | `app.guardian.guardians.show` | `guardian.guardian.view` | Same | R1 | Linked children, family, portal access, contact (masked by permission) |
| SCR-GRD-003 | Guardian form | form | `app.guardian.guardians.edit` | `guardian.guardian.update` | ADO, BM | R1 | CNIC duplicate check; mobile normalized to E.164 |
| SCR-GRD-004 | Link or unlink child | modal | `app.guardian.links.create` | `guardian.link.create` / `.delete` | ADO, BM / PR, BM | R1 | Relation, primary contact, fee payer, portal access; one primary enforced |
| SCR-GRD-005 | Families | list | `app.guardian.families.index` | `guardian.family.view` | MGT, ACT, ADO | R1 | Family code, children, voucher mode |
| SCR-GRD-006 | Merge or split family | wizard | `app.guardian.families.manage` | `guardian.family.manage` | ADO, ACT, BM | R1 | Preview of voucher and balance effects |
| SCR-GRD-007 | Custody restriction | modal | `app.guardian.custody.edit` | `guardian.custody.manage` | PR, OW | R1 | Sensitive; document upload; removes portal access for the restricted guardian |
| SCR-GRD-008 | Pickup authorizations | list, form | `app.guardian.pickups.index` | `guardian.pickup.view` | REC, BM, CT | R1 | Photo, CNIC last 4, validity |
| SCR-GRD-009 | Verify guardian | modal | `app.guardian.verifications.create` | `guardian.verification.verify` | ADO, BM | R1 | Verification code printed on slips (FR-GRD-006) |
| SCR-GRD-010 | Guardian portal: child switcher | drawer | `guardian.guardian.children.index` | Baseline | GDN | R1 | All linked, portal-enabled children |
| SCR-GRD-011 | Guardian portal: pickup persons | list, form | `guardian.guardian.pickups.index` | `guardian.pickup.view` | GDN | R1 | Add with photo; staff review flag |

## 10.25 HR (HR)

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-HR-001 | Employees | list | `app.hr.employees.index` | `hr.employee.view` | OW, OA, PR, VP, BM, HRM, HOD (assigned) | R1 | Filters: branch, department, designation, status, teaching |
| SCR-HR-002 | Employee profile | detail | `app.hr.employees.show` | `hr.employee.view` | Same | R1 | Tabs from registry: personal, assignments, qualifications, documents, attendance, leave, teaching load, payroll (R2) |
| SCR-HR-003 | Employee form | form | `app.hr.employees.edit` | `hr.employee.create` / `.update` | HRM, OA | R1 | CNIC duplicate check; create linked user optionally |
| SCR-HR-004 | Employee status change | modal | `app.hr.employees.manage` | `hr.employee.manage` | HRM, PR | R1 | Active, on leave, suspended, separated (R2 separation flow) |
| SCR-HR-005 | Salary history | list, form | `app.hr.salary_histories.index` | `hr.salary.view` / `.update` | OW, HRM, ACT / OW, HRM | R1 | Sensitive; effective dated (DEC-103) |
| SCR-HR-006 | Branch assignments | list, drawer | `app.hr.branch_assignments.index` | `hr.branch_assignment.view` | MGT, HRM | R1 | Shared staff across branches; primary branch |
| SCR-HR-007 | Qualifications and experience | list, form | `app.hr.qualifications.index` | `hr.employee.update` | HRM, OA | R1 | Profile tab |
| SCR-HR-008 | Employee documents | list | `app.hr.documents.index` | `hr.document.view` | HRM, PR | R1 | Expiry dates and reminders |
| SCR-HR-009 | Designations | list, form | `app.hr.designations.index` | `hr.designation.view` | MGT, HRM | R1 | Suggested role per designation |
| SCR-HR-010 | Staff directory | list | `app.hr.directory.index` | `hr.directory.view` | STAFF | R1 | Name, designation, branch, work phone only |
| SCR-HR-011 | Profile corrections | list | `app.hr.profile_corrections.index` | `hr.profile_correction.view` | HRM | R1 | Requests from staff (baseline) |
| SCR-HR-012 | Department heads | list, form | `app.hr.department_heads.index` | `hr.employee.update` | HRM, OA | R1 | Effective dated (D7 Part 3 correction) |
| SCR-HR-013 | Import employees | wizard | `app.import_export.import_jobs.create` (type `employees`) | `hr.employee.import` | HRM, OA | R1 | Uses SCR-IMX-002 |
| SCR-HR-014 | Contracts | list, form | `app.hr.contracts.index` | `hr.contract.view` | HRM, OW | R2 | |
| SCR-HR-015 | Performance reviews | list, form | `app.hr.performance_reviews.index` | `hr.performance_review.view` | PR, HOD, HRM, STAFF (own) | R2 | |
| SCR-HR-016 | Training records | list, form | `app.hr.training.index` | `hr.training.view` | HRM, PR, STAFF (own) | R2 | |
| SCR-HR-017 | Disciplinary actions | list, form | `app.hr.disciplinary.index` | `hr.disciplinary.view` | HRM, PR, OW | R2 | Sensitive |
| SCR-HR-018 | Separations | list, form | `app.hr.separations.index` | `hr.separation.view` | HRM, PR, OW | R2 | Clearance, final settlement link (Payroll) |

## 10.26 Attendance (ATT)

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-ATT-001 | Section sheets | list | `app.attendance.section_sheets.index` | `attendance.student_attendance.mark` or `.view` | CT; MGT, AC | R1 | Today's sections with status: not marked, draft, submitted, finalized; marking window countdown |
| SCR-ATT-002 | Mark section attendance | grid | `app.attendance.section_sheets.show` | `attendance.student_attendance.mark` | CT | R1 | DEC-162; see wireframe W-01; codes P A L HD LV with leave and holiday pre-filled; device punches shown |
| SCR-ATT-003 | Period attendance | grid | `app.attendance.period_sheets.show` | `attendance.period_attendance.mark` | ST | R1 | Where enabled (colleges); per timetable period |
| SCR-ATT-004 | Student attendance register | report | `app.attendance.student_attendance.index` | `attendance.student_attendance.view` | MGT, AC, EXC, CT, ST | R1 | Month grid by section with totals and percentage; print register layout |
| SCR-ATT-005 | Student attendance day detail | drawer | `app.attendance.student_attendance.show` | Same | Same | R1 | Source, punches, revisions, who changed what |
| SCR-ATT-006 | Corrections | list | `app.attendance.corrections.index` | `attendance.correction.view` | MGT, AC, CT, ST | R1 | Requests after finalization |
| SCR-ATT-007 | Request correction | form | `app.attendance.corrections.create` | `attendance.correction.create` | CT, ST | R1 | Students, date, new status, reason; approval flow |
| SCR-ATT-008 | Staff attendance today | list | `app.attendance.staff_attendance.index` | `attendance.staff_attendance.view` (branch) | OW, OA, PR, VP, BM, HRM | R1 | Present, late, absent, leave, missing out-punch; filters by department |
| SCR-ATT-009 | Staff attendance register | report | `app.attendance.staff_attendance.register` | Same | Same | R1 | Month grid per employee |
| SCR-ATT-010 | Administrative entry | form | `app.attendance.administrative_entries.create` | `attendance.administrative_entry.create` | PR, HRM | R1 | Mark staff for official duty, training, etc. with reason |
| SCR-ATT-011 | My attendance (staff) | detail | `app.attendance.my_attendance.index` | `attendance.staff_attendance.view` (own) | STAFF | R1 | Today's check-in and check-out, month calendar, missing punch request button; see wireframe W-06 |
| SCR-ATT-012 | Online check-in | modal | `app.attendance.my_attendance.check_in` | `attendance.staff_attendance.mark` (own) | STAFF | R1 | Trusted device and geofence result shown; works on 360px |
| SCR-ATT-013 | QR scan check-in | detail | `app.attendance.qr_scan.show` | `attendance.staff_attendance.mark` (own) | STAFF | R1 | Camera scan of kiosk code; single use per session |
| SCR-ATT-014 | QR kiosk display | detail | `app.attendance.qr_kiosk.show` | `attendance.qr_kiosk.manage` | REC, ITA | R1 | Full-screen rotating code every 30 seconds; kiosk mode hides navigation; auto-lock after inactivity |
| SCR-ATT-015 | Missing punch requests | list, form | `app.attendance.missing_punches.index` | `attendance.missing_punch.view` | HRM, PR, STAFF (own) | R1 | Approval flow |
| SCR-ATT-016 | Attendance policies | list, form | `app.attendance.policies.index` | `attendance.policy.view` | MGT, HRM | R1 | Per branch, shift, person type; effective dated |
| SCR-ATT-017 | Devices | list | `app.attendance.devices.index` | `attendance.device.view` | ITA, HRM, PR | R1 | Online status, last punch, last heartbeat |
| SCR-ATT-018 | Device form | form | `app.attendance.devices.edit` | `attendance.device.update` | ITA | R1 | Serial (global unique), adapter, purpose, IP allowlist, secrets masked |
| SCR-ATT-019 | Device enrollments | grid | `app.attendance.device_enrollments.index` | `attendance.device_enrollment.view` | ITA, HRM | R1 | Device user id to person mapping; unmapped punches list |
| SCR-ATT-020 | Import punch file | wizard | `app.attendance.punch_imports.create` | `attendance.punch_import.import` | ITA, HRM | R1 | CSV or Excel from any device; duplicate punches skipped |
| SCR-ATT-021 | Device sync log | list | `app.attendance.device_sync_logs.index` | `attendance.device.view` | ITA | R1 | Last 3 months (D7 7.50) |
| SCR-ATT-022 | Suspicious punch alerts | list | `app.attendance.suspicious_alerts.index` | `attendance.suspicious_alert.view` | PR, HRM, ITA | R1 | Outside geofence, impossible times, clock drift |
| SCR-ATT-023 | Attendance reports | report | `app.attendance.reports.index` | `attendance.report.view` | MGT, AC, HRM, CT | R1 | Daily summary, chronic absentees, late comers, section comparison; export |
| SCR-ATT-024 | Guardian portal: attendance | detail | `guardian.attendance.student_days.index` | `attendance.student_attendance.view` | GDN | R1 | Month calendar with codes and arrival times |
| SCR-ATT-025 | Student portal: my attendance | detail | `student.attendance.my_days.index` | `attendance.student_attendance.view` | STD | R1 | |

## 10.27 Leave (LVE)

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-LVE-001 | Leave applications | list | `app.leave.applications.index` | `leave.application.view` | MGT, HRM, CT (student leave) | R1 | Staff and student tabs; status; approval link |
| SCR-LVE-002 | Apply on behalf | form | `app.leave.applications.create` | `leave.application.create` | HRM, PR (staff); CT, BM (students) | R1 | Type, dates, half day, reason, document; day count with holidays and sandwich rule shown |
| SCR-LVE-003 | My leave | list | `app.leave.my_leave.index` | Baseline | STAFF | R1 | Balance cards, history, cancel pending |
| SCR-LVE-004 | Apply for leave (staff) | form | `app.leave.my_leave.create` | `leave.application.create` (own) | STAFF | R1 | Same calculator; shows approvers in order |
| SCR-LVE-005 | Leave balances | list | `app.leave.balances.index` | `leave.balance.view` | HRM, PR | R1 | Per employee and type |
| SCR-LVE-006 | Adjust balance | modal | `app.leave.balances.adjust` | `leave.balance.manage` | HRM | R1 | Reason required; creates adjustment row |
| SCR-LVE-007 | Leave calendar | report | `app.leave.calendar.index` | `leave.calendar.view` | MGT, HRM, AC | R1 | Who is away by day |
| SCR-LVE-008 | Leave types | list, form | `app.leave.types.index` | `leave.type.view` | MGT, HRM | R1 | |
| SCR-LVE-009 | Leave policies | list, form | `app.leave.policies.index` | `leave.policy.view` | MGT, HRM | R1 | Entitlement, accrual, carry-forward, designation rules |
| SCR-LVE-010 | Leave reports | report | `app.leave.reports.index` | `leave.report.view` | HRM, PR | R1 | |
| SCR-LVE-011 | Guardian portal: apply for child's leave | form | `guardian.leave.applications.create` | `leave.application.create` (child) | GDN | R1 | Dates, reason, document; status updates by SMS and in-app |
| SCR-LVE-012 | Guardian portal: leave requests | list | `guardian.leave.applications.index` | `leave.application.view` | GDN | R1 | |

---

## 10.28 Key Screen Wireframes

### W-01 Mark section attendance (SCR-ATT-002), mobile 360px

```
+------------------------------------+
| <  9-A . Tue 24 Nov        Saved v |  autosave status
| 38 students . window closes 10:00  |
| P 34   A 2   L 1   LV 1   NM 0     |  live counts
| [ All present ]  [ Legend ]        |
+------------------------------------+
| 1  (AK) Ayesha Khan                |
|    [P] [A] [L] [HD]   arrived 7:52 |  device punch shown
| 2  (BH) Bilal Hussain              |
|    [P] [A] [L] [HD]                |
| 3  (FN) Fatima Noor        LV      |  approved leave, locked
|    Leave approved 23 to 25 Nov     |
| 4  (HR) Hamza Raza                 |
|    [P] [A] [L] [HD]   remark +     |
| ...                                |
+------------------------------------+
| [ Submit attendance              ] |  primary; disabled until no NM rows
+------------------------------------+
```

- Status buttons are 44 x 44 px; the selected one uses the 9.5 attendance rendering.
- Keyboard on desktop: arrow keys move between students; P, A, L, H set codes; Enter submits.
- After the marking window, the sheet becomes read-only with "Request correction" (SCR-ATT-007).
- Absent students trigger guardian SMS only after submit or window close (D2 rules), never on each tap.

### W-02 Student profile (SCR-STU-002), desktop

```
+--------------------------------------------------------------------------------------------------+
| Students > Ayesha Khan                                                    [Edit] [Actions v] [History]|
| (photo)  Ayesha Khan   عائشہ خان        (v) Active                                                  |
|          CS-2024-0153 . GR 1024 . 9-A (Matric, Science) . Gulberg . Session 2026-27                  |
|          Father: Imran Khan . 0300 1234567 . B-Form 35202-•••••••-3 (lock)                           |
+--------------------------------------------------------------------------------------------------+
| Overview | Guardians | Attendance | Fees | Results | Documents | Timeline | Discipline (R2) | Learning (R2) |
+------------------------------------------+-------------------------------------------------------+
| ENROLLMENT                               | THIS MONTH                                            |
| Class 9-A, roll 12, since 1 Apr 2026     | Attendance 95% (19 of 20)                             |
| Subjects: English, Urdu, Islamiyat,      | Fee: Rs 3,500 outstanding (shown if fee.ledger.view)  |
|  Pak Studies, Maths, Physics, Chemistry, | Latest result: Mid-term 82.4%, A                      |
|  Biology                                 |                                                       |
+------------------------------------------+-------------------------------------------------------+
| PERSONAL                                 | CATEGORY AND SPONSOR                                  |
| Date of birth 12 Mar 2011 (lock)         | Category: Regular                                     |
| Address: shown with permission           | Sibling discount: Family F-0042                       |
+------------------------------------------+-------------------------------------------------------+
  Actions menu: change section, request transfer, request withdrawal, issue certificate, print ID card
  (each item shown only with its permission, DEC-147)
```

### W-03 New admission application (SCR-ADM-002), desktop

```
+--------------------------------------------------------------------------------------------------+
| Admissions > New application                                   Form: Gulberg School 2026-27      |
| (1) Student  >  (2) Guardians  >  (3) Previous school  >  (4) Documents  >  (5) Review            |
+--------------------------------------------------------------------------------------------------+
| Class applied for *     [ Class 6 v ]     Seats available: 7                                      |
| Student name *          [ Ali Raza                ]   Urdu name  [ علی رضا           ]            |
| Date of birth *         [ 04/09/2014 ]               Gender *    ( ) Male ( ) Female              |
| B-Form number           [ 35202-1234567-1 ]                                                       |
|  +---------------------------------------------------------------------------------------------+ |
|  | (i) Possible match: Ali Raza, B-Form ending 567-1, withdrawn from Gulberg in 2024.          | |
|  |     [ Open record ]  [ Readmit instead ]  [ Not the same child ]                             | |
|  +---------------------------------------------------------------------------------------------+ |
| Track and group         [ Matric v ] [ General v ]                                                |
+--------------------------------------------------------------------------------------------------+
|                                                    [ Save draft ]   [ Continue ]                  |
+--------------------------------------------------------------------------------------------------+
```

### W-04 Promotion wizard, decisions step (SCR-STU-010), desktop

```
+--------------------------------------------------------------------------------------------------+
| Promotions > Class 9 to Class 10 . Session 2026-27 to 2027-28                                     |
| (1) Source  >  (2) Decisions  >  (3) Sections  >  (4) Review  >  (5) Run                           |
+--------------------------------------------------------------------------------------------------+
| Eligibility from: Annual result (published 15 Mar)    Filter: [All v]   Promote all eligible [v]  |
| +--------------------------------------------------------------------------------------------+ |
| | Student          Section  Result     Attendance  Suggested     Decision                     | |
| | Ayesha Khan      9-A      Pass 82%   95%         Promote       [ Promote v ]                | |
| | Bilal Hussain    9-A      Comp. 1    88%         Conditional   [ Conditional v ] Maths     | |
| | Hamza Raza       9-B      Fail 3     71%  (!)    Detain        [ Detain v ]  reason *       | |
| | Sara Ahmed       9-B      Withheld   92%         Hold          [ Hold v ]   fee dues        | |
| +--------------------------------------------------------------------------------------------+ |
| Promote 112 . Conditional 6 . Detain 4 . Hold 3                                                   |
|                                                          [ Back ]   [ Continue to sections ]      |
+--------------------------------------------------------------------------------------------------+
```

The run step starts a bulk operation (SCR-CORE-004) and shows progress; nothing changes until the run step is confirmed (level 3: "Type 125 to confirm").

### W-05 New import, mapping step (SCR-IMX-002), desktop

```
+--------------------------------------------------------------------------------------------------+
| Imports > Students                                                                               |
| (1) Type  >  (2) Upload  >  (3) Map columns  >  (4) Validate  >  (5) Import                       |
+--------------------------------------------------------------------------------------------------+
| File: students_class6.xlsx . 214 rows        Saved mapping: [ Last year's format v ]             |
| Your column          Sample                   Maps to                                  Status    |
| Name                 Ali Raza                 [ Student name v ]                      (v)        |
| Father               Raza Ahmed               [ Father name v ]                       (v)        |
| DOB                  04-09-2014               [ Date of birth v ]  format [DD-MM-YYYY v] (v)     |
| Form-B               3520212345671            [ B-Form number v ]                     (v)        |
| Mobile               03001234567              [ Guardian mobile v ]                   (v)        |
| Class                6                        [ Class v ]                             (v)        |
| Remarks              -                        [ Ignore this column v ]                           |
| Required not mapped: Section  (!)                                                                |
|                                             [ Save mapping ]   [ Back ]   [ Validate 214 rows ]  |
+--------------------------------------------------------------------------------------------------+
```

The validate step shows valid rows, rows with errors (downloadable file with an error column) and possible duplicates; only valid rows import, each through the same Action used by the form.

### W-06 My attendance with check-in (SCR-ATT-011 and SCR-ATT-012), mobile 360px

```
+------------------------------------+
| My attendance                      |
| Tue 24 Nov . Gulberg               |
| +--------------------------------+ |
| | Not checked in                 | |
| | Shift starts 7:45 AM           | |
| | [ Check in now               ] | |  primary
| | This device: trusted (v)       | |
| | Location: inside campus (v)    | |
| +--------------------------------+ |
| [ Scan QR at reception instead ]   |
+------------------------------------+
| NOVEMBER                           |
| Mo Tu We Th Fr                     |
|  P  P  L  P  P                     |
|  P  P  P LV  P                     |
|  P NM  P  P  .                     |
| Missing out-punch on 18 Nov        |
| [ Request missing punch ]          |
+------------------------------------+
```

### W-07 Approval request detail (SCR-APR-002), desktop

```
+--------------------------------------------------------------------------------------------------+
| Approvals > Fee concession . Bilal Hussain (9-A)                         (!) Waiting: step 1 of 2  |
+------------------------------------------+-------------------------------------------------------+
| REQUEST                                  | STEPS                                                 |
| Concession   30% on tuition              | 1 Principal (you)          waiting since 2h          |
| From         Dec 2026 to Mar 2027        | 2 Owner (above 25%)        not started                |
| Reason       Father lost job; letter     |                                                       |
|              attached                    | HISTORY                                               |
| Effect       Rs 2,550 less per month     | Submitted by Fatima Ali (Accountant), 24 Nov 9:10 AM |
| Requested by Fatima Ali                  |                                                       |
| Attachment   employer_letter.pdf  (eye)  |                                                       |
+------------------------------------------+-------------------------------------------------------+
| Comment                                                                                          |
| [                                                                                              ] |
|                                       [ Return for changes ]  [ Reject ]  [ Approve ]            |
+--------------------------------------------------------------------------------------------------+
```

When separation of duties blocks the viewer (D5 5.11), the decision panel is replaced by a notice naming the rule ("You recorded this payment, so you can't approve its reversal").

### W-08 Role permission editor (SCR-IAM-019), desktop

```
+--------------------------------------------------------------------------------------------------+
| Roles > Class Teacher (customized)                         Template v3 . [Reset to template] [Save]|
| Search permissions [            ]   Module [ All v ]   Show: [ Granted only ]                     |
+--------------------------------------------------------------------------------------------------+
| ATTENDANCE                        view   mark   export  print   approve  Visibility               |
|   Student attendance              [x]    [x]    [x]     [x]     -        [ Assigned v ]           |
|   Corrections                     [x]    -      -       -       [ ]      [ Assigned v ]           |
| STUDENT                                                                                          |
|   Students                        [x]    -      [ ]     [ ]     -        [ Assigned v ]           |
|   Health (S)                      [x]    -      -       -       -        [ Assigned v ]           |
| GUARDIAN                                                                                         |
|   Contacts                        [x]    -      -       -       -        [ Assigned v ]           |
+--------------------------------------------------------------------------------------------------+
| Changes: +1 (attendance.correction.approve), -0         Users affected: 42                        |
+--------------------------------------------------------------------------------------------------+
```

"Save" opens a level 2 confirmation listing every added and removed permission. Permissions marked S show a note that holders will need two-factor sign-in (DEC-63).

---

## Screen Count (Part 2)

| Module | Screens | Module | Screens |
|---|---|---|---|
| Platform | 20 | Academic | 17 |
| Organization | 11 | Admission | 13 |
| Identity | 27 | Student | 26 |
| Settings | 6 | Guardian | 11 |
| Core | 5 | HR | 18 |
| Audit | 5 | Attendance | 25 |
| Approval | 9 | Leave | 12 |
| Notification | 9 | | |
| ImportExport | 5 | | |
| Document | 6 | **Total, Part 2** | **225** |

---

## Assumptions (Part 2)

| ID | Assumption |
|---|---|
| A-82 | Teachers mark attendance mostly on phones, so the marking grid is designed for 360px first and enhanced for desktop keyboards. |
| A-83 | The QR kiosk runs on a tablet or spare screen at reception, signed in as a receptionist or IT admin in kiosk mode. |
| A-84 | Leaving certificates follow one format per institution, configured through document templates; board-specific formats are template variants, not separate screens. |

## Gaps Found and Added (Part 2)

| ID | Gap | Decision |
|---|---|---|
| GAP-208 | No single place to see one record's change history | Shared record history drawer (DEC-160) |
| GAP-209 | Unclear which module builds each profile tab | Tabs belong to the data owner (DEC-161) |
| GAP-210 | Two people marking one section could overwrite each other | Per-row autosave with version check (DEC-162) |
| GAP-211 | A QR code screen at reception needed a safe unattended mode | Kiosk mode with auto-lock (SCR-ATT-014) |
| GAP-212 | Admins could not see why a user has a permission | Effective permissions viewer with sources (SCR-IAM-024) |
| GAP-213 | Bulk student actions had no preview | Preview count step (SCR-STU-020) |
| GAP-214 | Duplicate children could be admitted again under a new record | "Possible match" panel with readmit option (SCR-ADM-002) |

---

Reply CONTINUE for Deliverable 10, Part 3
