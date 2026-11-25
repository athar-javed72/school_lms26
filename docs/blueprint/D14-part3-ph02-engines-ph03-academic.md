# Deliverable 14: Sequential Antigravity Implementation Prompts
# Part 3 of 9: PH-02 Shared Engines and PH-03 Academic Structure

**Version:** 1.0 (draft for approval) | **Date:** 24 Nov 2026
**Builds on:** D14 Parts 1 and 2, D2 (FR-APR, FR-NTF, FR-IMX, FR-DOC, FR-ACD), D3 3.14, 3.15, 3.23, 3.24, D5 5.7.5, 5.7.6, 5.9, 5.11, D6, D6A, D7 Part 1 (7.11 to 7.14) and Part 2 (7.17), D10 Part 2 (10.17 to 10.21), D11 Part 1 (11.2 to 11.4) and Part 2 (11.25 to 11.28)
**How to run:** each step goes into a new Antigravity conversation, after the Phase 0 Master Context Prompt.

---

## 14.7 New Decisions

| ID | Decision |
|---|---|
| DEC-242 | **Approval statuses follow D11 11.3.** `approval_requests.status` takes the values `pending`, `returned`, `approved`, `rejected`, `withdrawn`, `apply_failed`, `expired`. The column `cancelled_at` is named `withdrawn_at`. `approval_actions.action` takes `submitted`, `resubmitted`, `approved`, `rejected`, `returned`, `withdrawn`, `escalated`, `commented`, `applied`, `apply_failed`. `approval_workflow_steps.resolver_type` takes the D11 resolver keys plus `named_role` and `user`. This corrects D7 Part 1 7.11, which predates DEC-167. |
| DEC-243 | **Platform fallback SMS provider.** The platform super admin configures a provider in platform settings (credentials encrypted). It is used only for identity-critical messages (OTP, password reset, owner invitation, lockout notice) when an organization has no active SMS provider of its own. Every other message uses the organization's own provider (A-12). |
| DEC-244 | **Notification templates and rules default in code.** The `NotificationEventRegistry` holds each event's default rule and its English and Urdu templates. `notification_templates` and `notification_rules` rows exist only where an organization has customized something. Resetting a customization deletes that organization's row, so the code default applies again. |
| DEC-245 | **Teaching assignments and class teachers move to P05.1.** These tables reference `employees`, which HR creates in PH-05: `teaching_assignments`, `section_class_teacher_histories`, and the FK on `sections.class_teacher_employee_id`. P05.1 creates them with real timestamps (DEC-218). It also builds SCR-ACD-013, 014 and 017, the class teacher field on SCR-ACD-011, and the assigned-scope resolvers. PH-03 builds sections without a class teacher. This corrects the M04 contents in D7 7.51 and the P03.2 scope in D13 13.5. |
| DEC-246 | **Session copy steps are a registry.** Academic owns `SessionCopyStepRegistry`, which the new session wizard (SCR-ACD-003) uses. Academic contributes classes, sections and class subjects in PH-03. Later contributions: teaching assignments and class teachers (P05.1), fee structures (P06.2), bell schedules and timetables (P09.1). |

### Corrections to earlier deliverables

| Where | Change |
|---|---|
| D7 Part 1 7.11 | Approval statuses, actions and resolver types follow DEC-242 |
| D7 7.51 M04 | `teaching_assignments` and `section_class_teacher_histories` move to P05.1 (DEC-245) |
| D13 13.5 P03.2 | Teaching assignments move to P05.1 (DEC-245) |
| D3 3.15 | `SmsGateway` has a platform fallback binding for identity-critical messages (DEC-243) |

---

## 14.8 PH-02 Shared Engines

**Phase goal (D13):** build the engines every later module uses, one per concern (PR-09). **Exit:** a sample approval type runs end to end with SOD; an SMS reaches the provider sandbox in Urdu, with its segment count shown; a users import commits through Actions with audit; and a templated PDF renders through the queue.

| Step | Scope |
|---|---|
| P02.1 | Approval engine |
| P02.2 | Notification engine |
| P02.3 | Import and export engine |
| P02.4 | Document engine |

---

### P02.1 Approval Engine

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D11 Part 1 11.2, 11.3 and 11.4 completely; D05 5.7.5 (approval permissions), 5.9 (types and default workflows), 5.11 (SOD); D07 Part 1 7.11 with the DEC-242 corrections; D10 Part 2 10.17 (SCR-APR-001 to 009), wireframe W-07, and SCR-CORE-003; D11 Part 2 11.26 (period locks); D03 3.24 (Approval).

PHASE GOAL
Build the single configurable approval engine: workflows with ordered steps and resolvers, requests with payload snapshots, double validation (DEC-167, DEC-168), delegation, separation of duties, the inbox and decision screens, bulk approve and daily digest, and deliver the first real request type, period_unlock, end to end.

ALREADY BUILT
PH-00 and PH-01: tenancy, users, audit, Core services (numbering, period locks, idempotency, bulk operations), RBAC with BasePolicy, SOD framework and PermissionResolver, authentication and 2FA, platform and organization management with impersonation guard, settings, navigation, portal homes with AttentionItemRegistry, Laravel 13.

TASKS
1. Migrations (Approval): approval_workflows, approval_workflow_steps, approval_requests, approval_request_steps, approval_step_assignees, approval_actions, approval_delegations exactly as D07 7.11 with DEC-242 changes. Add the deferred FK period_locks.unlock_request_id -> approval_requests (composite).
2. Contracts in Modules\Approval\Contracts: ApprovalHandler with the D11 11.3 methods (type(), rules(), validateSubmission(), snapshotTargets(), summary(), apply(), onRejected(), onReturned(), onWithdrawn()) plus metadata methods bulkApprovable(), allowsExpiry(), harmlessChanges(); ApproverResolver (resolve(request, step): list of users). Modules register types in Definitions/ApprovalTypes.php: type key, handler class, submit permission, approve and reject permission keys, default workflow (ordered steps with resolver key, config, mode, conditions), notify-only roles, release.
3. Registries: ApprovalTypeRegistry and ApproverResolverRegistry. Resolvers in this step: principal_of_branch, vice_principal_of_branch, hr_manager_of_branch, accountant_of_branch, exam_controller_of_branch, owner, org_admin, named_role (role slug in config, scoped to the request branch), user (fixed user). Resolvers that need other modules (class_teacher_of_student, hod_of_subject, reporting_manager, guardian_of_student, principal_of_target_branch) are registered by those modules later; a workflow step using an unregistered resolver fails validation in the editor.
4. WorkflowResolver: picks the active workflow for the request type and branch (branch override first, then organization default per uq_awf_active); seeds default workflows per organization lazily on first use from the type definition (version 1); editing creates a new version and in-flight requests keep their version.
5. Actions: SubmitApprovalRequest (permission check, handler rules and validateSubmission, snapshotTargets stores version fingerprints in payload metadata, request_no from NumberingService key approval_request, steps created, step 1 assignees resolved and written to approval_step_assignees after removing users blocked by SOD and BR-APR-01; if a step is left with no assignee it escalates to the next step or to the principal of the branch per BR-APR-04), DecideApprovalStep (approve, reject, return with comment; actor must be an assignee or active delegate AND hold the approve or reject permission in the request branch, DEC-54; impersonated sessions cannot decide money types via ImpersonationGuard; mode any or all; on final approval run apply), ApplyApprovalRequest (inside a transaction: re-run validateSubmission, compare snapshot versions unless the change is declared harmless, call handler apply(); success sets approved and applied_at; failure sets apply_failed with apply_error and notifies the requester; apply must be idempotent), ResubmitApprovalRequest (after returned; back to step 1), WithdrawApprovalRequest (requester, while pending or returned), DelegateApprovals and RevokeDelegation, BulkApprove (only bulk_approvable types; each request validated and applied on its own; failures listed).
6. Events after commit: ApprovalRequestSubmitted, ApprovalStepDecided, ApprovalRequestApproved, ApprovalRequestRejected, ApprovalRequestReturned, ApprovalRequestWithdrawn, ApprovalRequestApplyFailed. Audit keys approval.request.submitted, approved, rejected, returned, withdrawn, applied, apply_failed (with comments and delegation names).
7. First type: period_unlock (Core resource, handler Modules\Approval\Approval\PeriodUnlockHandler calling Core PeriodLockService through its public service; payload module, period, duration in days (max 14), reason; default steps per D05 5.9: principal for attendance and examination, owner for fee and accounting, HRM requests go to principal; approve permission core.period_lock.unlock; register SOD-10 (requester cannot approve)). SCR-CORE-003 Request unlock modal on the period locks screen. Automatic relock: command core:relock-periods every 15 minutes locks periods whose relock_due_at has passed.
8. Attention item: approval.pending_for_me (count of pending assignments for the user; items older than 24 hours flagged high, DEC-169). Command approvals:daily-digest at 08:00 organization time (fan out per organization) fires one ApprovalDigestDue domain event per approver with pending items. Delivery is added in P02.2, which registers the approval.daily_digest notification event for it; the command does not change.
9. Screens (D10 10.17 exactly): SCR-APR-001 Approvals inbox (tabs waiting for me, delegated to me, done by me; filters type, branch, age; bulk approve where allowed), SCR-APR-002 Request detail (W-07: payload rendered by the handler summary, attachments, steps, history timeline as x-approval.timeline, decision panel; SOD notice replaces the panel when blocked, naming the rule), SCR-APR-003 My requests (app portal now; guardian and student variants in PH-04), SCR-APR-004 All requests (oversight), SCR-APR-005 Bulk approve, SCR-APR-006 Workflows, SCR-APR-007 Workflow editor (ordered steps, resolver, condition builder for amount, days, protected field, student age; mode any or all; validation that each step's resolver can hold the approve permission per DEC-54; SLA field visible but disabled with "Available in R2"), SCR-APR-008 Delegations. SCR-APR-009 (guardian) arrives in PH-04.
10. Definitions: Permissions.php for approval.* (D05 5.7.5); Navigation (Home group: Approvals with pending badge); Settings: approval.reminder_hours (default 24).
11. Tests support: a TestApprovalType fixture registered only in tests with a fake handler to exercise every engine path.
12. decisions.md rows: DEC-15, DEC-54, DEC-56, DEC-167, DEC-168, DEC-169, DEC-242.

DATABASE
approval_workflows (awf), approval_workflow_steps (aws), approval_requests (arq), approval_request_steps (ars), approval_step_assignees (asa), approval_actions (aac), approval_delegations (adl) per D07 Part 1 7.11 with DEC-242: arq.status values pending, returned, approved, rejected, withdrawn, apply_failed, expired; arq.withdrawn_at instead of cancelled_at; aac.action values per DEC-242; aws.resolver_type values per DEC-242. FK period_locks.unlock_request_id added.

BUSINESS RULES
- No self-approval (BR-APR-01). SOD rules remove blocked users from assignees; an empty step escalates (BR-APR-04).
- Acting needs both: resolved as assignee (or active delegate) and the type's approve or reject permission in the request's branch (DEC-54).
- Payload and snapshot are immutable after submission; approvers always see the stored payload.
- Apply re-validates; a changed target fails as apply_failed unless declared harmless (DEC-167, DEC-168).
- Conditions are data in approval_workflow_steps, never code.
- Withdraw allowed only while pending or returned.

UI SPEC
- W-07 layout; decision buttons Return for changes, Reject, Approve with a comment field (comment required for reject and return).
- Inbox rows show type, subject, requester, branch, age; items older than 24 hours show a warning pill.
- Workflow editor uses a vertical stepper with move up and down buttons (no drag-only interaction, WCAG 2.5.7).

SECURITY AND PERMISSIONS
- Routes and permissions per D10 10.17; detail visible to requester, assignees, delegates and holders of approval.request.view.
- Decisions audited with comment, delegation and IP. Impersonated sessions cannot approve money types.
- Payload attachments stored through FileStorageService and downloaded through signed URLs with the request policy.

TESTS
- Engine tests with TestApprovalType: submit validation failure; single-step approve applies; two-step flow; mode all requires every assignee; reject ends; return then resubmit restarts at step 1; withdraw; apply_failed when the target changed after submission; harmless change passes; apply idempotent on retry.
- DEC-54: resolved approver without permission cannot decide (403); user with permission but not resolved cannot decide.
- SOD and BR-APR-01: requester removed from assignees; empty step escalates to the next step or the principal.
- Delegation: delegate decides, decision records both names; expired delegation ignored.
- Conditions: amount threshold adds the owner step; below threshold skips it.
- Workflow versioning: editing creates version 2; an in-flight request continues on version 1.
- period_unlock end to end: request from SCR-CORE-003, principal approves, period unlocked until relock_due_at, write allowed, core:relock-periods relocks, write refused again; requester cannot approve (SOD-10).
- Screen tests (200, 403, 404 cross-tenant), audit entries, isolation suite, RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. period_unlock works end to end on staging with screenshots of W-07 in English and Urdu.
2. All engine path tests pass, including apply_failed and idempotent apply.
3. Workflow editor saves a branch override and the resolver for the request branch changes accordingly.
4. Daily digest command runs per organization without errors.
5. CI green; staging green.

DO NOT
- Do not hardcode approval chains in handlers; defaults live in Definitions and are editable data.
- Do not let any module write approval tables directly; modules use SubmitApprovalRequest.
- Do not build SLA escalation (R2).

OUTPUT
Report per Master section 20 with the engine test matrix, the period_unlock walkthrough, and the list of resolvers registered.
```

---

### P02.2 Notification Engine

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D11 Part 2 11.27; D02 FR-NTF-001 to 013 and BR-NTF-01 to 03, AC-NTF-01 to 03; D03 3.15 (SmsGateway), 3.24 (Notification); D07 Part 1 7.12; D05 5.7.5 (notification permissions); D10 Part 2 10.18 (SCR-NTF-001 to 009); D12 Part 1 12.8 (delivery receipts); D14 DEC-239, DEC-243, DEC-244.

PHASE GOAL
Deliver every message through one engine: event registry with English and Urdu templates, organization rules and preferences, quiet hours, dedupe, credit caps, SMS, email and in-app channels with retry and fallback providers, delivery receipts, broadcasts and the notification centre, and move OTP and invitation delivery onto it.

ALREADY BUILT
PH-00, PH-01 and P02.1: foundation shell, OtpDelivery (email and log), approval engine with events and the digest command firing ApprovalDigestDue.

TASKS
1. Migrations (Notification): notification_templates, notification_providers, notification_rules, notification_dispatches ([L]), notification_delivery_attempts ([L]), notification_preferences, and Laravel's notifications table with organization_id, exactly as D07 Part 1 7.12.
2. Modules\Notification\Support\NotificationEventRegistry (DEC-244): modules contribute Definitions/NotificationEvents.php entries: event_key {module}.{event_snake}, the domain event class, placeholders with sample values, recipient resolver keys, default channels, is_critical (cannot be opted out), is_urgent (bypasses quiet hours and credit block), default English and Urdu templates per channel (SMS body, email subject and body, in-app title and body). RecipientResolverRegistry with resolvers in this step: user (the subject user), users_with_role_in_branch, approval_step_assignees, approval_requester, organization_owners. Guardian and class-teacher resolvers arrive with their modules.
3. Register events for modules built so far: identity.otp_code (urgent, critical), identity.password_reset (urgent, critical), identity.account_locked (critical), identity.new_device (critical), identity.two_factor_reset (critical), platform.owner_invitation (urgent, critical), platform.support_access_requested, approval.request_submitted, approval.step_decided, approval.request_decided, approval.request_apply_failed, approval.daily_digest, core.bulk_operation_completed. Templates never contain passwords, full CNIC or full B-Form (BR-NTF-03); OTP templates show the code only.
4. Dispatcher pipeline (D11 11.27): listener DispatchNotificationsForEvent (after commit) -> rule (organization row or registry default) -> resolve recipients -> preferences (critical events ignore opt-out) -> language (user locale; Urdu template when locale ur) -> quiet hours (setting notification.quiet_hours, default 21:00 to 07:00 organization time; SMS and WhatsApp only; urgent bypass) -> dedupe key hash(event, recipient, channel, source record) within 10 minutes -> credit check for SMS (plan sms_credits_monthly via LimitEnforcer; warnings at 80% and 100% to owners; block at 100% except urgent) -> notification_dispatches row with rendered snapshot (BR-NTF-02) and segments -> job SendDispatch on queue notifications (urgent identity messages on critical).
5. Channels and adapters: Modules\Notification\Contracts\SmsGateway (send(to, body, unicode): ProviderResult); HttpSmsGateway (configurable request template per aggregator: method, URL, auth, field mapping, success parsing; credentials from notification_providers encrypted), LogSmsGateway (local and testing), FakeSmsGateway (tests). Email channel through Laravel Mail with the branded layout. In-app channel through the notifications table via TemplatedNotification. Provider priority and fallback: retry 1, 5, 15 minutes on the same provider, then the next active provider of the channel, then failed with the provider error (AC-NTF-03). Per-provider rate limit (rate_limit_per_minute) through Redis limiter.
6. Platform fallback provider (DEC-243): platform setting notification.platform_sms_provider (encrypted), configured in a section of the SCR-PLT-016 system health page (platform super admin only); used only for events flagged identity_critical when the organization has no active SMS provider.
7. SmsSegmentCounter: GSM-7 160 and 153 per part; Unicode (any Urdu or non-GSM character) 70 and 67 per part; used in template editor, broadcast estimate and dispatch rows.
8. Delivery receipts: webhook route webhooks.notification.sms_dlr (POST /webhooks/sms-dlr/{organization_uuid}) storing webhook_receipts first, verifying a per-provider token, then job ProcessDeliveryReport updating dispatch status delivered or failed and cost_units.
9. Rebind Identity OtpDelivery to NotificationOtpDelivery that dispatches identity.otp_code through the engine (SMS when the destination is a mobile, email when an email), keeping the existing OtpService unchanged. Replace the P02.1 digest wrapper with the engine. Owner invitations from P01.5 use platform.owner_invitation. ApprovalDigestDue maps to approval.daily_digest (in-app).
10. Screens (D10 10.18 exactly): SCR-NTF-001 Notification centre (app and platform portals; unread first, mark all read, links to source; topbar bell with unread count via x-notification.bell), SCR-NTF-002 Notification preferences (event groups by channel; critical events locked on), SCR-NTF-003 Templates (customized badge), SCR-NTF-004 Template editor (placeholders, live preview with sample values in English and Urdu, SMS segment counter, reset to default), SCR-NTF-005 Rules (on or off, channels, recipients, quiet hours exempt for non-critical only by OA), SCR-NTF-006 Providers (credentials masked after save, priority order, active and fallback flags), SCR-NTF-007 Test send (to own number or email), SCR-NTF-008 Delivery log (filters recipient, event, channel, status; attempts; resend failed), SCR-NTF-009 Send broadcast (audience in this step: staff by role and branch; guardian and student audiences are added in PH-04; channel; preview; estimated recipients, SMS segments and credits; publish needs notification.broadcast.publish; sent as a bulk operation).
11. Commands: notifications:release-quiet-hours daily 07:00 organization time (fan out) releasing held dispatches; schedule in routes/console.php.
12. Definitions: Permissions.php for notification.* (D05 5.7.5); Settings: notification.quiet_hours_start, notification.quiet_hours_end, notification.dedupe_minutes (default 10), notification.sms_credit_warning_percent (80); Navigation entries (Settings group: Notifications; Home group: Notifications centre).
13. decisions.md rows: DEC-183, DEC-243, DEC-244.

DATABASE
notification_templates (ntp), notification_providers (npv), notification_rules (nrl), notification_dispatches (ndp, log), notification_delivery_attempts (nda, log), notification_preferences (npf), notifications (Laravel, with organization_id) per D07 Part 1 7.12.

BUSINESS RULES
- Notifications only after the business transaction commits (BR-NTF-01). Content snapshotted (BR-NTF-02). No secrets or full identity numbers in messages (BR-NTF-03).
- Critical events cannot be switched off by users; organizations can switch off non-critical events only.
- Quiet hours hold non-urgent SMS and WhatsApp (AC-NTF-02: a 22:30 non-urgent SMS waits until 07:00).
- Same event, recipient, channel and record within 10 minutes is sent once (FR-NTF-010).
- SMS credit warning at 80%, block at 100% except urgent (FR-NTF-012).
- Staff who are also guardians receive one message per channel per event addressed to them.

UI SPEC
- Template editor shows English and Urdu side by side, Urdu field RTL with Nastaliq preview, segment counter updating live (Unicode note for Urdu).
- Broadcast form shows the estimate panel (recipients, segments per message, total segments, remaining credits) before publish; publish uses level 2 confirmation.
- Delivery log uses the data table pattern; status pills per D09 9.5 (queued info, sent and delivered success, failed danger, held warning).

SECURITY AND PERMISSIONS
- Provider credentials encrypted, never returned after save, never logged (integration log channel strips them).
- Delivery receipt webhook verified per provider token; stored first.
- Routes and permissions per D10 10.18; preferences and centre are baseline.

TESTS
- Pipeline unit tests: rule default versus organization override; recipient resolution; critical events ignore preference opt-out; Urdu template chosen for ur locale; quiet hours hold and release (AC-NTF-02); urgent bypass; dedupe within 10 minutes; credit warning and block with urgent allowed.
- Adapter tests: HttpSmsGateway builds the configured request (HTTP fake) and parses success and failure; retry 1, 5, 15 minutes then fallback provider then failed with error (AC-NTF-03).
- SmsSegmentCounter: GSM and Unicode boundaries (160, 161, 70, 71 characters).
- Delivery receipt webhook: stored first, invalid token rejected, status updated.
- OTP through the engine: SMS dispatched on the critical queue; platform fallback used when the organization has no provider; never held by quiet hours.
- Screen tests, audit entries for template, rule and provider changes, isolation suite, RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. On staging with the SMS aggregator sandbox: an OTP SMS in Urdu arrives (or the sandbox log shows it) with 2 Unicode segments counted correctly; a test send works from SCR-NTF-007.
2. A broadcast to staff in one branch shows the estimate and delivers in-app and SMS.
3. Quiet hours, dedupe and credit tests pass.
4. Delivery log shows attempts and provider responses without secrets.
5. CI green; staging green.

DO NOT
- Do not let modules send SMS, email or in-app messages directly; they fire domain events.
- Do not store provider credentials in .env or config.
- Do not build WhatsApp (R2) or push (R3).

OUTPUT
Report per Master section 20 with the sandbox evidence, segment counter cases and the list of registered events.
```

---

### P02.3 Import and Export Engine

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D11 Part 2 11.28; D02 FR-IMX; D03 3.24 (ImportExport); D07 Part 1 7.13; D05 5.7.5 and 5.7.23; D10 Part 2 10.19 (SCR-IMX-001 to 005), wireframe W-05, SCR-IAM-017 and SCR-AUD-005; D12 Part 1 12.7 (formula injection, uploads).

PHASE GOAL
Provide one import engine (template, upload, column mapping with saved mappings, validation preview, commit through the same Actions as forms, error files) and one export engine (queued, masked, formula-safe, expiring downloads), proven with the users import and the audit log export.

ALREADY BUILT
PH-00, PH-01, P02.1 approval engine, P02.2 notification engine (core.bulk_operation_completed and import_export events can now be delivered).

TASKS
1. Install maatwebsite/excel ^3.1 (package check).
2. Migrations (ImportExport): import_jobs, import_job_rows, import_mappings, export_jobs exactly as D07 7.13.
3. Contracts: ImportDefinition (key, permission, label keys, columns with key, label en and ur, required, type, allowed values or lookup, sample, help; uniqueness keys for duplicate detection; validateRow(row, context): RowResult with errors and warnings; commitRow(row, context): created record reference through the module's Action; optional update mode flag, default false) and ExportDefinition (key, permission, columns with masking rules, query builder from a Query class with the user's visibility, formats xlsx and csv, optional pdf later). Registries ImportRegistry and ExportRegistry; modules contribute in Definitions/ImportsExports.php.
4. Services: TemplateBuilder (xlsx template with header row in the chosen language, a hidden key row, data validation dropdowns for enum columns, an instructions sheet in English and Urdu), ColumnMapper (auto-map by header text in English or Urdu, saved mappings per type), ImportValidator (chunked reading, row rules, in-file duplicates and database duplicates via uniqueness keys, warnings versus errors), DuplicateDetector, ErrorFileBuilder (original rows plus an errors column).
5. Jobs: ValidateImportFile (queue imports), CommitImportChunk (one transaction per chunk of 200 rows; only valid rows; each row calls commitRow which uses the module Action so audit, numbering and business rules apply; audit entries carry import_job_id in metadata), FinalizeImport (counts, error file, status completed or completed_with_errors, event ImportCompleted), BuildExport (queue exports; applies PiiMasker per column for the exporting user; prefixes cells starting with = + - @ with an apostrophe; stores file with expiry 24 hours; event ExportReady), PurgeExpiredExports. Command import-export:purge-expired hourly. Import rows purged 90 days after completion (scheduled job).
6. Import flow screens (D10 10.19): SCR-IMX-001 Import history, SCR-IMX-002 New import wizard (W-05 steps: type, download template, upload with virus scan status, map columns with saved mapping selector and date format option, validate preview with valid, warnings, errors and possible duplicates, commit; progress through SCR-CORE-004), SCR-IMX-003 Import job detail (row results, error file download; no rollback), SCR-IMX-004 Saved mappings, SCR-IMX-005 Exports (own queued exports, download until expiry).
7. First import type: users (Identity contributes; permission identity.user.import; columns name, name_ur, mobile, email, username, initial role slug, branch code; commit through CreateStaffUser with the subset rule; sends invitations through the notification engine after commit). Wire SCR-IAM-017 to SCR-IMX-002 with type users.
8. First export type: audit log (Audit contributes; permission audit.log.export; uses the AUD-001 filters; masked values). Build SCR-AUD-005 Export audit log modal (queued, masked).
9. A generic export button component x-import-export.export-button used by list screens: posts the current filters to the export definition and shows the exports drawer.
10. Definitions: Permissions.php for import_export.history.view; Navigation (Settings group: Imports; My work: My exports).
11. decisions.md rows: DEC-32, FR-GEN-014 alignment, DEC-62 (export masking).

DATABASE
import_jobs (imj), import_job_rows (imr), import_mappings (imm), export_jobs (exj) per D07 Part 1 7.13. Files in stored_files with purposes import and export.

BUSINESS RULES
- Imports never bypass business rules: every row goes through the same Action as the form (D11 11.28).
- Imports never update existing records unless the definition declares update mode.
- Only valid rows commit; errors are downloadable; rollback is not offered because records are audited creations.
- Exports show exactly what the exporting user may see on screen, with the same masking.
- Export files expire after 24 hours and can be downloaded only by the user who requested them.

UI SPEC
- W-05 mapping table with sample values, status per row, "Required not mapped" warning, date format selector.
- Validation preview shows counts and the first 50 rows of each category with reasons; downloads for errors.
- Large imports show progress and are safe to leave; an in-app notification arrives on completion.

SECURITY AND PERMISSIONS
- Import permission per type (D05 5.7.23); upload limited to xlsx and csv, size limit by plan, virus scan when enabled.
- Formula injection guard on every export cell.
- Export download through signed URL bound to the requesting user.

TESTS
- Template builder produces headers in English and Urdu and dropdowns for enum columns.
- Mapper auto-maps English and Urdu headers; saved mapping reused.
- Users import: 50 valid rows create 50 users with audit entries carrying import_job_id; duplicate mobile inside the file and against the database flagged; invalid role slug error; subset rule violation error; error file contains the errors column.
- Chunk failure isolation: an exception in one chunk marks that chunk's rows failed and others still commit.
- Export: masked CNIC-like column for a user without core.pii.view_full; formula cells prefixed; expiry enforced; another user cannot download.
- 5,000-row validation preview under 2 minutes on the S1 profile (performance test, group performance).
- Screen tests, isolation suite, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. Users import end to end on staging with an error file for bad rows and invitations sent for created users.
2. Audit log export downloads a masked, formula-safe xlsx.
3. 5,000-row preview within budget (evidence in report).
4. CI green; staging green.

DO NOT
- Do not insert imported rows with raw queries or bypass Actions.
- Do not allow exports to include fields the user cannot see on screen.
- Do not build import types for modules not yet built.

OUTPUT
Report per Master section 20 with the import walkthrough, preview timing, and the export masking evidence.
```

---

### P02.4 Document Engine

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D03 3.14, D07 Part 1 7.14, D09 9.19, D10 Part 2 10.20 (SCR-DOC-001, 002, 004, 005), D11 Part 2 11.25, D12 Part 2 DEC-215 (pre-rendered PDFs), D05 5.7.5 (document permissions), and the P00.4 spike report in docs/decisions/spike-urdu-pdf.md.

PHASE GOAL
Turn the spike renderer into the document engine: document types registered by modules, institution templates with branding and language modes, issued documents with immutable snapshots, serial numbers and verification codes, reissue and revoke, batch rendering and merging on the documents queue, and pre-rendered storage for high-volume PDFs.

ALREADY BUILT
PH-00 (PdfRenderer with Gotenberg and Fake, spike templates), PH-01, P02.1 to P02.3 (approval, notification, import and export engines).

TASKS
1. Migrations (Document): document_templates and issued_documents exactly as D07 7.14.
2. Contracts and registry: DocumentType (key, label keys, owning module, data provider class, default view key {module}::pdf.{document}, paper preset, orientation, supported locale modes en, ur, bilingual, issuable flag for registered documents such as certificates versus printable-only documents such as vouchers, numbering sequence key for serials); DocumentDataProvider (provide(subject, options): DocumentData DTO with fields and sample()). DocumentTypeRegistry; modules contribute in Definitions/DocumentTypes.php.
3. DocumentComposer: resolves the institution template (or the type default), merges branding (institution logo, name in English and Urdu, header band in secondary or tenant accent, signatures as stored images), locale mode, data DTO; renders Blade to HTML with pdf.css and fonts inlined; calls PdfRenderer. Template options JSON per D09 9.19 (header, footer, signature blocks, fields shown).
4. Actions: IssueDocument (issuable types only: serial from NumberingService per type and branch, verification_code 12 random characters from an unambiguous alphabet unique globally, content_snapshot JSON frozen, rendered PDF stored as stored_files purpose generated_pdf, issued_documents row, event DocumentIssued, audit), ReissueDocument (new serial, old row superseded with superseded_by_id, reason required, reprint_count on the old row), RevokeDocument (status revoked, reason, verification will show revoked), RenderDocument (printable types: render without issuing; optional store for pre-rendered batches), RenderDocumentBatch (bulk operation on queue documents: chunks of 50 subjects, each chunk rendered and stored, then MergeBatchPdf merges chunk files into one PDF per batch scope, result file on the bulk operation, DEC-215 pre-render storage with retention metadata).
5. Footer standard (D09 9.19): printed by, printed at (organization timezone), verification code with QR for issued types (bacon/bacon-qr-code), page x of y.
6. Replace the spike templates' use in tests: keep them only as rendering fixtures under tests/Fixtures/pdf; register a test-only document type (test_certificate) for engine tests. Real types arrive with their modules (fee_voucher and fee_receipt P06.3 and P06.4, report_card P07.2, admit_card P07.1, leaving_certificate and character_certificate P08.3, id_card P09.2).
7. Screens (D10 10.20): SCR-DOC-001 Document templates (per type and institution), SCR-DOC-002 Template settings (header, footer, signatures upload as PNG, paper, language mode, fields to show; PDF preview with sample data through the type's sample()), SCR-DOC-004 Issued documents register (serial, type, subject, issued by, status; filters), SCR-DOC-005 Reissue or revoke (reason required; PR, BM reissue; OW, PR revoke). SCR-DOC-003 designer is R3 and SCR-DOC-006 public verification is R2.
8. Definitions: Permissions.php for document.* (D05 5.7.5); Navigation (Settings group: Documents); Settings: document.default_locale_mode.
9. Nightly rendering test group gotenberg extended to render test_certificate in en, ur and bilingual modes and assert text extraction.
10. decisions.md rows: DEC-12, DEC-31, DEC-215.

DATABASE
document_templates (dtp), issued_documents (isd) per D07 Part 1 7.14. Uses stored_files and numbering_sequences.

BUSINESS RULES
- Issued document content is immutable (BR-DOC-01): reprints reproduce the snapshot, not current data.
- Verification codes are unique globally and never reused; revoked documents fail verification.
- Reissue requires a reason and creates a new serial; the old document is superseded, not deleted.
- High-volume printable documents are pre-rendered by batch jobs and served from storage (DEC-215).

UI SPEC
- Template settings preview renders a real PDF from sample data in an embedded viewer with download.
- Register shows status pills (issued info, superseded neutral, revoked danger) and the verification code (masked partially in lists, full on detail for permitted users).

SECURITY AND PERMISSIONS
- PDF downloads only through signed URLs after the subject record's policy check.
- Signature images are PNG only, stored privately, never public.
- Gotenberg receives inlined HTML only (no remote fetch).

TESTS
- Composer merges branding and locale modes; bilingual output contains both languages.
- IssueDocument: serial sequence per branch, unique verification code, snapshot frozen (changing the subject later does not change a reprint), audit entry.
- Reissue supersedes; revoke changes status; both require reasons.
- RenderDocumentBatch: 120 subjects produce 3 chunks and one merged PDF; failure in one chunk reported as completed_with_errors with the failed subjects listed.
- Screen tests, RBAC fixture, isolation suite, ScreenCoverageTest; nightly gotenberg group.

ACCEPTANCE CRITERIA
1. A test certificate issues with serial, QR verification code and footer on staging; reissue and revoke work.
2. Batch render of 1,000 test documents merges into one PDF within 5 minutes on staging (evidence).
3. Template preview works in English, Urdu and bilingual modes.
4. CI green; nightly green; staging green.

DO NOT
- Do not let modules call PdfRenderer directly; they use DocumentComposer and the Actions.
- Do not regenerate issued documents from current data.
- Do not build the public verification page (R2) or the designer (R3).

OUTPUT
Report per Master section 20 with sample PDFs (paths), batch timing and the registry contract summary. Then run the D13 13.6 gate for PH-02.
```

---

## 14.9 PH-03 Academic Structure

**Phase goal (D13):** the academic ladder and offering per institution and session, for both national and Cambridge tracks. **Exit:** Scenario A, Scenario B and a Cambridge O and A Level institution can each be set up from the wizard, and session-scoped uniqueness holds. Teaching assignments are delivered in P05.1 (DEC-245).

| Step | Scope |
|---|---|
| P03.1 | Sessions, levels, progressions, study groups, buckets, subjects, seed templates |
| P03.2 | Classes, class subjects, sections, teaching groups, session wizard and lifecycle, onboarding step |

---

### P03.1 Sessions, Levels, Groups, Subjects and Seed Templates

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D06 6.1 to 6.6 completely, D06A A.2 to A.5 and A.12, D07 Part 2 7.17 (academic_sessions, education_levels, education_level_progressions, study_groups, study_group_levels, study_group_elective_buckets, study_group_subjects, subjects), D05 5.7.6, D10 Part 2 10.21 (SCR-ACD-001, 002, 004, 006, 007, 008), D02 FR-ACD and BR-ACD.

PHASE GOAL
Model the academic ladder per institution with tracks (national, cambridge, common), progression paths, study groups with compulsory subjects, alternative sets and elective buckets (option blocks), the subject master, and academic sessions with their lifecycle states, seeded from the D06 and D06A templates.

ALREADY BUILT
PH-00, PH-01 and PH-02: foundation shell, approval, notification, import and export, and document engines.

TASKS
1. Migrations (Academic, real timestamps): academic_sessions, education_levels, education_level_progressions, study_groups, study_group_levels, study_group_elective_buckets, subjects, study_group_subjects exactly as D07 Part 2 7.17 (including generated markers, uq_acs_active, uq_acs_planning, ck constraints). subjects.department_id composite FK to departments.
2. Models with BelongsToOrganization (all institution-owned; no branch scope), Auditable, HasVisibilityScope, morph aliases academic_session, education_level, study_group, subject; enums SessionStatus (planning, active, closing, closed), AcademicTrack (national, cambridge, common), EducationStage, SubjectType, SubjectRequirement (compulsory, elective), TargetQualification (as_only, a_level).
3. Seed templates as code data in Modules\Academic\Definitions\Templates: NationalLevels (D06 6.3, 15 levels with Urdu names, stage, next level, board flags, login default), CambridgeLevels (D06A A.4.1: PG to C8 as common, O1, O2, O3 inactive, AS, A2), Progressions (D06A A.4.2 defaults per track), StudyGroups and StudyGroupSubjects (D06 6.4 General, matric groups MSCI, MCS, MART active and MHS, MAG, MFD inactive; intermediate programs as listed in D06 6.4; D06A A.5 OLVL with alternative sets urdu_choice, religion_ol, pst_ol and option blocks as elective buckets with choose exactly 1, and the A Level group), Subjects (D06 and D06A subject lists with codes, Urdu names, types and default components).
4. Action ApplyAcademicTemplate(institution, curriculum): national -> national levels per institution type (school levels 1 to 13, college 14 and 15, school_and_college all, per D06 6.3 rule); cambridge -> common PG to C8 plus Cambridge levels; dual -> both; creates progressions, study groups, buckets, subjects and group subject mappings; idempotent by codes; audited. For cambridge and dual institutions also seeds translation overrides that relabel "Class" as "Grade" for the institution's organization only when the organization has no national institution (otherwise leaves terminology unchanged and records a note).
5. Actions for maintenance: Create, Update, Deactivate for education levels (reorder, rename, Urdu name, stage, track, flags; deactivation refused when used by an active session class, BR-ACD-02), progressions (one default per from level and track), study groups (system group GEN cannot be deleted), study group levels, elective buckets (min and max choices, timetable block label), study group subjects (requirement, alternative set, bucket, periods, components, qualification), subjects (code unique per institution, components JSON validated: codes unique, max marks positive), all audited.
6. Sessions: CreateSession (planning; one planning per institution and track, DEC-79; defaults DEC-74 school 1 Apr to 31 Mar, college 1 Sep to 31 Aug, DEC-87 Cambridge 1 Aug to 31 Jul; admission window and promotion date defaults), UpdateSession (dates editable only in planning; name immutable after active), ActivateSession (preconditions D06 6.6: previous session closing or closed, start date reached or override with reason, at least one academic class exists, and the common versus track-specific rule from D07 7.17; one active per institution and track), StartClosingSession, CloseSession (runs SessionCloseCheckRegistry; Academic contributes no checks; later modules add theirs; level 3 confirm; applies period locks to all months of the session through PeriodLockService), ReopenSession is only through period_unlock (no direct action). Events SessionActivated, SessionClosed (after commit).
7. SessionCloseCheckRegistry contract (Academic) with result objects (passed, message, link) for the checklist screen.
8. Screens (D10 10.21): SCR-ACD-001 Sessions (per institution and track), SCR-ACD-002 Session form, SCR-ACD-004 Activate session modal, SCR-ACD-006 Education levels and progressions, SCR-ACD-007 Study groups and elective buckets (compulsory, elective, alternative sets, choose exactly one buckets with block labels), SCR-ACD-008 Subjects (code, names, type, components; external syllabus codes column appears in P07.3; import button). SCR-ACD-003 wizard and SCR-ACD-005 close checklist arrive in P03.2.
9. Import types: subjects (permission academic.subject.import; update mode allowed for names and components by code).
10. Definitions: Permissions.php for academic.session, education_level, study_group, subject (D05 5.7.6); Settings: academic.session_defaults per institution type (editable); Navigation (Academics group: Sessions and classes, Subjects, Levels and groups).
11. decisions.md rows: DEC-26, DEC-65 to DEC-67, DEC-74, DEC-76, DEC-77, DEC-78, DEC-79, DEC-80, DEC-82 (preparation), DEC-86, DEC-87.

DATABASE
academic_sessions (acs), education_levels (edl), education_level_progressions (elp), study_groups (stg), study_group_levels (sgl), study_group_elective_buckets (sgb), subjects (sub), study_group_subjects (sgs) per D07 Part 2 7.17.

BUSINESS RULES
- One active and one planning session per institution and track (DEC-79); a common session excludes track-specific active sessions in the same institution and the reverse.
- Levels used by active-session classes cannot be deactivated (BR-ACD-02).
- Progression to a level in another institution of the same organization is allowed (DEC-52).
- Seed subject lists are templates to be confirmed per institution (DEC-65); nothing is hardcoded in business logic.
- Session dates are editable only while planning.

UI SPEC
- Level list shows order with move up and move down, track badge, stage, board flag, Urdu name.
- Study group editor: per level, compulsory list, alternative sets grouped, buckets as cards with "choose exactly N" text and block label.
- Subjects list with components chips (TH 75, PR 25).

SECURITY AND PERMISSIONS
- Institution-owned data: organization-scope roles and principals per D05 5.7.6; teachers view only.
- All changes audited.

TESTS
- ApplyAcademicTemplate for national school, national college, school_and_college, cambridge and dual institutions: correct level counts, progressions defaults, groups active and inactive, buckets with min and max, idempotent second run.
- Session rules: second planning per track refused; activation preconditions (no class -> refused); common versus track rule; dates locked after activation; close applies period locks and runs registered checks (fake failing check blocks close).
- Level deactivation refused when in use (after P03.2 creates classes; here use a factory class).
- Subjects import with update mode by code.
- Screen tests, audit, isolation suite, RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. On staging, a national school institution and a Cambridge institution each receive their template with correct levels, groups and subjects visible in English and Urdu.
2. Session lifecycle tests pass including DEC-79 constraints at the database level.
3. Subjects import works with update mode.
4. CI green; staging green.

DO NOT
- Do not create classes, sections or teaching groups (P03.2).
- Do not create teaching assignments or class teachers (P05.1, DEC-245).
- Do not hardcode level codes or subject names in business rules; read them from data.

OUTPUT
Report per Master section 20 with template application counts per curriculum and session lifecycle evidence.
```

---

### P03.2 Classes, Sections, Teaching Groups, Session Wizard and Onboarding

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D06 6.6 (new session wizard, close checklist) and 6.7, D06A A.7, D07 Part 2 7.17 (academic_classes, class_subjects, sections, teaching_groups), D04 4.14, D05 5.7.6 and D06A A.11 (academic.teaching_group), D10 Part 2 10.21 (SCR-ACD-003, 005, 009, 010, 011, 012, 015, 016), D14 DEC-245 and DEC-246.

PHASE GOAL
Let each branch offer classes per session (level and study group), with class subjects copied from the group template and adjustable, sections with capacity, shift, wing and room, and teaching groups for subject sets; provide the new session wizard with a copy-step registry, the close checklist screen, and the onboarding step that sets up academic defaults.

ALREADY BUILT
PH-00 to PH-02; P03.1 sessions, levels, progressions, study groups, buckets, subjects, templates and session lifecycle.

TASKS
1. Migrations (Academic): academic_classes, class_subjects, sections, teaching_groups exactly as D07 Part 2 7.17, with these exceptions per DEC-245: sections.class_teacher_employee_id column created without FK (P05.1 adds the FK); section_class_teacher_histories and teaching_assignments are not created here; teaching_group_members is created in P04.4.
2. Models: AcademicClass, ClassSubject, Section, TeachingGroup with BelongsToOrganization and BelongsToBranch, Auditable, HasVisibilityScope; morph aliases academic_class, class_subject, section, teaching_group.
3. Actions: CreateAcademicClass (branch, session in planning or active, level, study group; level must be offered by the group via study_group_levels; display name generated from level and group, editable; copies class_subjects from study_group_subjects for that level with requirement, alternative set, bucket, periods, components, qualification), UpdateAcademicClass, DeactivateAcademicClass (refused with active enrollments once Student exists; registry check), UpdateClassSubjects (add, remove, change periods, components and elective flags; removal refused when students take the subject, via a registered check filled by Student in PH-04), CreateSection (name unique within class; capacity required; shift from the branch's shifts; wing from the branch's available_wings; room), UpdateSection, DeactivateSection, CreateTeachingGroup (class, subject must be a class subject, name, capacity, room), UpdateTeachingGroup, DeactivateTeachingGroup. Capacity enforcement mode setting academic.section_capacity_mode (warning or hard) is applied later at enrollment; store it now.
4. SessionCopyStepRegistry (DEC-246) and the new session wizard: steps from the registry in order; Academic contributes "classes and sections" (copy classes per branch with names, sections with capacity, shift, wing, room) and "class subjects and periods". Wizard flow per D06 6.6 steps 1 to 3 and 7 now (source and target session; choose branches; preview counts per branch; run as bulk job CopySessionStructure with progress on SCR-CORE-004). Steps 4 to 6 appear automatically when P05.1, P06.2 and P09.1 register their contributions.
5. Close checklist screen SCR-ACD-005 rendering SessionCloseCheckRegistry results with links, close button enabled only when all pass, level 3 confirmation.
6. Onboarding step "academic defaults" contributed to OnboardingStepRegistry: per institution choose curriculum confirmation and apply the P03.1 template; create the first session in planning with defaults; for Scenario A create classes for the selected levels with one section A each in the single branch; mark the step complete.
7. Screens (D10 10.21): SCR-ACD-003 New session wizard, SCR-ACD-005 Close session checklist, SCR-ACD-009 Classes (per session: level, group, sections, strength column shows 0 until PH-04), SCR-ACD-010 Class detail (sections, class subjects, teaching groups; teachers column appears in P05.1; timetable link in P09.1), SCR-ACD-011 Sections (capacity, room, shift, wing; class teacher column added in P05.1), SCR-ACD-012 Class subjects grid (periods per week, components, elective flags, buckets), SCR-ACD-015 Teaching groups list, SCR-ACD-016 Teaching group detail (name, subject, capacity, room; members tab arrives in P04.4; teacher in P05.1; "generate from option blocks" arrives in P04.4).
8. Import types: classes and sections (permissions academic.class.create plus academic.section.create; columns branch code, session name, level code, group code, class name, section name, capacity, shift name, wing, room code).
9. Session context in the topbar: x-academic.session-switcher showing the active session per institution and allowing staff to view a planning or closed session in read mode with the D09 banner ("You are viewing session 2025-26 (closed). Records are read-only.").
10. Definitions: Permissions.php for academic.class, academic.section, academic.class_subject, academic.teaching_group (D05 5.7.6, D06A A.11); Settings: academic.section_capacity_mode, academic.class_name_pattern; Navigation (Academics group: Classes, Sections, Teaching groups).
11. decisions.md rows: DEC-68 (preparation), DEC-75 (preparation), DEC-81, DEC-245, DEC-246.

DATABASE
academic_classes (acl), class_subjects (csb), sections (sec, class_teacher_employee_id without FK), teaching_groups (tgp) per D07 Part 2 7.17.

BUSINESS RULES
- One academic class per branch, session, level and group (uq_acl).
- Section names unique within the class; capacity at least 1; wing must be one of the branch's available wings.
- Class subjects start as a copy of the group template and can diverge per branch without changing the template.
- Closed sessions are read-only for every user; classes and sections cannot be created in closing or closed sessions.
- Teaching group subject must be one of the class subjects.

UI SPEC
- Class detail uses tabs (Sections, Subjects, Teaching groups) with counts.
- Wizard follows D09 stepper; preview shows counts per branch; run uses level 3 confirmation with the total count.
- Closed session banner and read-only rendering everywhere in academic screens.

SECURITY AND PERMISSIONS
- Branch-owned rows scoped by AllowedBranchesScope; principals manage their branch; academic coordinators per D05.
- All changes audited; wizard run audited with parameters.

TESTS
- CreateAcademicClass copies class subjects from the template; level not offered by the group refused; duplicate class refused (uq_acl).
- Sections: wing validation, unique names, capacity validation.
- Teaching groups: subject must be a class subject.
- Session wizard copies classes and sections across two branches correctly; idempotent rerun skips existing classes; progress tracked.
- Close checklist blocks with a failing registered check and allows close when green.
- Onboarding Scenario A path creates template, session, classes and sections; Scenario B leaves class creation to staff; Cambridge institution path creates Cambridge classes.
- Read-only enforcement for closed sessions (403 or disabled with reason).
- Classes and sections import.
- Screen tests, audit, isolation suite (branch isolation: Branch A principal cannot see Branch B classes), RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. Scenario A, Scenario B (two branches) and a Cambridge O and A Level institution are set up on staging from the onboarding wizard, with screenshots of classes, sections and class subjects.
2. New session wizard copies a session's structure into a planning session for two branches.
3. Branch isolation tests pass.
4. PH-03 exit criteria met; D13 13.6 gate evidence attached.
5. CI green; staging green.

DO NOT
- Do not create teaching assignments, class teacher histories or employee references (P05.1, DEC-245).
- Do not create teaching group members or enrollments (PH-04).
- Do not hardcode section names or class naming beyond the configurable pattern.

OUTPUT
Report per Master section 20 with the three scenario walkthroughs and the wizard evidence. Then run the D13 13.6 gate for PH-03.
```

---

## Assumptions (Part 3)

| ID | Assumption |
|---|---|
| A-116 | The chosen SMS aggregator offers a sandbox or test sender ID before P02.2, and supports Unicode (Urdu) SMS. |
| A-117 | The platform operator signs one fallback SMS agreement for identity-critical messages (DEC-243). Its cost is covered by the platform. |
| A-118 | The D06 seed subject lists for Punjab and the D06A Cambridge lists are acceptable starting templates. Schools confirm them during onboarding (DEC-65). |

## Gaps Found and Added (Part 3)

| ID | Gap | Decision |
|---|---|---|
| GAP-296 | The approval statuses in D7 disagreed with the D11 lifecycle | DEC-242 |
| GAP-297 | New organizations without their own SMS provider couldn't receive OTPs or invitations | DEC-243 |
| GAP-298 | Seeding every template and rule per organization would create thousands of identical rows | Code defaults with override rows (DEC-244) |
| GAP-299 | Academic tables referenced `employees` before HR created that table | DEC-245 |
| GAP-300 | The new session wizard needed contributions from modules built later | `SessionCopyStepRegistry` (DEC-246) |
| GAP-301 | Approval handlers for Core-owned resources had no home that avoided a Core-to-Approval dependency | `PeriodUnlockHandler` lives in Approval and calls Core's public service (P02.1) |

---

Reply CONTINUE for Deliverable 14, Part 4
