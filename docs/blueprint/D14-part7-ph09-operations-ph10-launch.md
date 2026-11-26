# Deliverable 14: Sequential Antigravity Implementation Prompts
# Part 7 of 9: PH-09 Timetable, Communication, Administration, Reporting and Production and PH-10 R1 Hardening, Pilot and Launch

**Version:** 1.0 (draft for approval) | **Date:** 24 Nov 2026
**Builds on:** D14 Parts 1 to 6, D02 (FR-TTB, FR-COMM, FR-ADN, FR-RPT), D05 5.7.17, 5.7.18 and 5.7.22, D06 6.6, D07 Part 3 (7.31 to 7.34) and Part 4 (7.50), D09, D10 Part 1 (10.5 to 10.9) and Part 3 (10.35 to 10.38, 10.47, wireframe W-13), D11 Part 2 (11.16, 11.30, 11.32), D12 Part 1 and Part 2 (12.17, 12.22, 12.24, 12.28 to 12.35), D13 (13.6, 13.7, DEC-221, DEC-222)
**How to run:** start a new Antigravity conversation for each step and paste the Phase 0 Master Context Prompt before the step prompt.

---

## 14.18 New Decisions

| ID | Decision |
|---|---|
| DEC-274 | **Widgets replace home sections.** P09.3 builds `WidgetRegistry` with the DEC-154 contract. Every section registered earlier through `HomeSectionRegistry` (DEC-250) is re-registered as a widget under its D10 10.6 key: for example "Today" becomes `guardian.child_today`, and "Fees due" becomes `guardian.child_fee_due`. P09.3 then removes `HomeSectionRegistry`, and an architecture test makes sure no module still references it. |
| DEC-275 | **Period attendance can be official.** The branch setting `attendance.period_official` is off by default (A-19). When it is on, the daily status of a section is derived from its period rows each day after the last period: `P` when the student is present in at least `attendance.period_min_present` periods (default half, rounded up), `L` when late in the first period and present enough, `A` otherwise. Derived rows use source `system`. A daily mark already made by a teacher or admin is never overwritten (DEC-25). |
| DEC-276 | **Closure effects.** `ClosureDeclared` writes `H` through the attendance prefill registry for future days. For today, it updates open sheets that aren't finalized (source `system`) and cancels queued absence alerts. Finalized sheets stay as they are, and an attention item asks the principal to request corrections. Timetables show the day as closed, and no period sheets open for it. |
| DEC-277 | **Partition switch-on happens in P09.4.** The production database is created fresh in P09.4. Its migration set includes `core_switch_on_partitions`, which converts every [L] table in D07 7.50 to RANGE COLUMNS partitions. Each table gets one partition per month from 2027-01 up to three months ahead, plus `p_future` (`student_timeline_events` is yearly, and `gps_positions` arrives with R3). The same migration runs on staging after an anonymized copy is taken. This completes the plan in D14 Part 2 (P01.2) and D07 7.50. |
| DEC-278 | **Fix steps have their own IDs and use the full template.** Fixes from the penetration test and the pilot are numbered `P10.2-fix-NNN` and `P10.3-fix-NNN`. Each has a branch `step/P10.3-fix-NNN-{slug}` and a report at `docs/agent/reports/P10.3-fix-NNN.md`. Each one runs in its own conversation, using the fix prompt template in 14.21 with all twelve sections. |
| DEC-279 | **Accessibility gate.** Pest 4 browser tests run axe-core on the 30 key screens listed in P10.1. They run in both languages and at 360 px and 1280 px widths. Any WCAG 2.2 AA violation of serious or critical impact fails CI. Contrast and focus checks use the D09 tokens. |
| DEC-280 | **Translation completeness covers printed documents.** `TranslationParityTest` (G-6) is extended to the Blade views of every document type and every guardian and student portal view. A key used there without an Urdu value fails the test (A-16). |

### Corrections to earlier deliverables

| Where | Change |
|---|---|
| D14 Part 4 DEC-253 and P05.3 | Device endpoints use the D12 12.28 hosts: ADMS on `adms.{platform_domain}` (middleware group `adms`), signed device webhooks on `hooks.{platform_domain}` (group `webhook`). The project copy of Part 4 is updated. |
| D14 Part 6 P07.3 | External exam permissions come from D06A A.11 (D05 5.7.17 is Timetable). The project copy of Part 6 is updated. |
| D14 Part 4 DEC-250 | Superseded by DEC-274 in P09.3 |
| D10 Part 2 SCR-ATT-003 | Built in P09.1 with DEC-275 |

---

## 14.19 PH-09 Timetable, Communication, Administration, Reporting and Production

**Exit criteria:**
- MS-09 is reached.
- The owner dashboard for 20 branches loads in under 10 seconds.
- Production passes a restore drill and a replica promotion drill.

### P09.1 Timetable and Period Attendance

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 3 7.31 (bell_schedules, bell_schedule_periods, timetables, timetable_entries) with DEC-109; D07 Part 2 7.22 student_period_attendances (soft timetable_entry_id per D07 Part 3 DEC-108); D10 Part 3 10.35 (SCR-TTB-001 to 008, 010, 011) and wireframe W-13; D10 Part 2 SCR-ATT-003; D11 Part 2 11.16 (period attendance) and 11.30 (timetable); D05 5.7.17; D06 6.6 (new session wizard copy steps); D02 FR-TTB; D14 DEC-246, DEC-251, DEC-275.

PHASE GOAL
Build school timetables that never clash: bell schedules per shift and day pattern, branch-wide timetable versions with draft, active and archived states, an editor with section, teacher and room views and live conflicts, activation with an effective date, printouts, personal timetables for staff, guardians and students, period attendance sheets for colleges, and the session copy step for bells and timetables.

ALREADY BUILT
PH-00 to PH-08: academic structure with sections, teaching groups and teaching assignments, rooms and shifts, attendance with daily sheets and the prefill registry, WorkingDayCalendar default binding, approval, notification and document engines, SessionCopyStepRegistry.

TASKS
1. Migrations (Timetable): bell_schedules, bell_schedule_periods, timetables, timetable_entries exactly as D07 7.31 (uq_ttb_active, uq_tte_teacher, uq_tte_room, uq_tte_section, uq_tte_group). Migration (Attendance): student_period_attendances exactly as D07 7.22 with timetable_entry_id as a soft reference (D07 Part 3 DEC-108).
2. Actions: CreateBellSchedule and UpdateBellSchedule (shift, day pattern, periods with times and breaks; ends after starts; no overlaps), CreateTimetableVersion (draft for branch, session and shift; or CopyTimetableVersion from the active one), PlaceEntry (day, period, teaching assignment; copies section or group, subject, employee; room optional), MoveEntry, SwapEntries, RemoveEntry, TimetableConflictDetector (database uniques per slot plus checks the database cannot express: the same teacher in another branch's active timetable at an overlapping time using bell times, students of a teaching group placed against their section's other periods, room capacity against section or group size, teacher weekly load against max_periods_per_week, unplaced periods per class subject), ActivateTimetable (refused while conflicts exist; effective date; previous active version archived; TimetableActivated event notifies affected teachers; period sheets follow the new version from the effective date), ArchiveTimetable.
3. Queries: MyTimetableQuery (teacher by employee, guardian child by section and teaching groups, student self), FreeRoomsQuery, TeacherLoadQuery (used by SCR-ACD-014 workload from now on).
4. Period attendance (DEC-251, DEC-275): period sheets open for a teaching assignment's timetable entries on working days (WorkingDayCalendar); MarkPeriodAttendance (subject teacher of the entry; statuses P, A, L, LV; LV prefilled from approved leave; idempotency and version per DEC-162; same lock and finalization rules as daily sheets with the policy window per period); DeriveDailyFromPeriods (command attendance:derive-daily-from-periods after the last period of each shift, only where attendance.period_official is on; DEC-275 rule; never overwrites teacher or admin daily marks).
5. Session copy step "Bell schedules and timetable as draft" registered in SessionCopyStepRegistry (DEC-246): copies bell schedules and the active timetable as a draft in the target session, mapping teaching assignments by section, group and subject; unmapped entries listed in the preview.
6. Documents: timetable_section (classroom display layout), timetable_teacher, both printable in English and Urdu.
7. Screens (D10 10.35 R1 rows exactly): SCR-TTB-001 Bell schedules, SCR-TTB-002 Timetable versions, SCR-TTB-003 Timetable editor (W-13: views by section, teacher and room; place by buttons and drag, WCAG 2.5.7 alternative with Move and Swap; live conflicts panel; unplaced list; teacher load counters), SCR-TTB-004 Activate version, SCR-TTB-005 Teacher timetable, SCR-TTB-006 Room timetable, SCR-TTB-007 Section timetable print, SCR-TTB-008 My timetable, SCR-TTB-010 Guardian: timetable, SCR-TTB-011 Student: timetable, SCR-ATT-003 Period attendance. SCR-ACD-010 Class detail gets its timetable link; SCR-ACD-017 My classes gets period attendance links where enabled.
8. Definitions: Permissions.php for timetable.* per D05 5.7.17 and attendance.period_attendance per D05 5.7.11; Settings attendance.period_official (branch, off), attendance.period_min_present; Navigation (Academics group: Bell schedules, Timetables; My work: My timetable).
9. decisions.md rows: DEC-109, DEC-246, DEC-251, DEC-275.

DATABASE
bell_schedules (bsc), bell_schedule_periods (bsp), timetables (ttb), timetable_entries (tte) per D07 Part 3 7.31; student_period_attendances (spa) per D07 Part 2 7.22 with the soft reference.

BUSINESS RULES
- One active timetable per branch, session and shift (uq_ttb_active); a version activates only with zero conflicts.
- A teacher, room, section or teaching group holds at most one entry per slot; cross-branch teacher clashes are checked by bell time.
- Entries come only from active teaching assignments; ending an assignment flags its entries for replacement.
- Period attendance is informational unless the branch makes it official (A-19, DEC-275); derived daily rows never overwrite manual daily marks.

UI SPEC
- W-13 grid: periods as rows, days as columns, entry cards with subject and teacher, conflict badge with Move, Swap and "Show free teachers" actions, unplaced counts per subject, teacher load panel.
- Keyboard: arrow keys move focus between cells; Enter opens the place dialog.
- Personal timetables highlight today and the current period.

SECURITY AND PERMISSIONS
- Editing needs timetable.timetable.update; activation timetable.timetable.publish; teachers view own; guardians and students view their own timetables only.
- Activation audited with the version and effective date.

TESTS
- Bell schedule overlap refused.
- Conflicts: teacher double slot (database), room double slot, teaching group student clash, cross-branch teacher overlap by time, room capacity, load over the limit; activation refused with any conflict.
- Activation archives the previous version and notifies teachers; period sheets use the new version from the effective date.
- Period attendance marking with LV prefill; DeriveDailyFromPeriods only when official and never over manual marks.
- Session copy step maps entries and lists unmapped ones.
- Screen tests including portals, audit, isolation suite, RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. On staging, a branch builds a conflict-free timetable for a morning shift, activates it, and teachers, guardians and students see their timetables; a college section uses period attendance with daily derivation.
2. CI green; staging green.

DO NOT
- Do not build substitutions (R2) or automatic timetable generation.
- Do not overwrite daily marks from period data.

OUTPUT
Report per Master section 20 with the conflict test matrix, W-13 screenshots and the period attendance evidence.
```

### P09.2 Communication, Calendar, Closures and ID Cards

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 3 7.32 (announcements, announcement_audiences, announcement_reads, diary_entries) and 7.33 (calendar_events); D10 Part 3 10.36 (SCR-COMM-001 to 006, 013, 014, 017) and 10.37 (SCR-ADN-001 to 004, 009, 012); D11 Part 2 11.30 (communication) and 11.32 with DEC-185; D05 5.7.18 and 5.7.22; D01 GAP-19 (closures); D02 FR-COMM and FR-ADN; D14 DEC-249, DEC-276.

PHASE GOAL
Give schools their daily communication and calendar: announcements with audiences, schedules, acknowledgements and optional SMS copies, the teacher's daily diary with optional review, the school calendar with holidays, vacations and events seeded from the school-year template, emergency closure declarations with their attendance and timetable effects, the real WorkingDayCalendar, and ID cards for students and staff.

ALREADY BUILT
PH-00 to PH-08 and P09.1: notification engine with broadcasts, attendance with prefill registry and alerts, timetables, document engine, WorkingDayCalendar default binding, students and employees with photos.

TASKS
1. Migrations (Communication): announcements, announcement_audiences, announcement_reads, diary_entries exactly as D07 7.32 (message threads and PTM tables are R2). Migration (Administration): calendar_events exactly as D07 7.33 (gate passes, visitors and complaints are R2 and R3).
2. Announcements: CreateAnnouncement (draft; title and body in English and optionally Urdu; body sanitized with an allowlist (HTMLPurifier configuration: paragraphs, lists, bold, italic, links with rel noopener)), SetAudience (organization, institution, branch, academic class, section, role, user; portal app, guardian or student), PublishAnnouncement (communication.announcement.publish; now or scheduled by command communication:publish-scheduled every 5 minutes; in-app to the resolved audience; optional SMS or WhatsApp copy (WhatsApp R2) as a notification broadcast with the estimate panel), AcknowledgeAnnouncement, RemindUnread (manual and automatic after communication.ack_reminder_hours, default 24), WithdrawAnnouncement, expiry by communication:expire.
3. Diary: WriteDiaryEntry (section or teaching group the teacher is assigned to; type homework, reminder or note; same-day edits only, marked edited after publish), ReviewDiaryEntry (when setting communication.diary_review is on: pending_review then approve or return), publication to guardians and students of the section or group.
4. Calendar (DEC-185): CreateCalendarEvent and UpdateCalendarEvent (types public_holiday, institution_holiday, closure, vacation, event, exam_period, ptm, working_day_override; affects_attendance flag; branch or institution scope); SeedSchoolYearCalendar per organization and session from the D11 11.32 template (national April to March, Cambridge August to July) as editable events and reminders, run by the onboarding finance or academic step on first session creation; public holidays come from a platform data file docs/data/pk_public_holidays_{year}.csv reviewed yearly (no dates typed into code).
5. WorkingDayCalendar rebind (DEC-249): Administration implementation combining shift working days, holidays, vacations, closures and working day overrides by branch; cached per branch and month and invalidated on calendar changes; Attendance prefill provider registered for H with the event title as the reason.
6. Closures (D01 GAP-19, DEC-276): DeclareClosure (administration.closure.create; dates, branches, reason such as smog, heat or security; level 2 confirm) creating closure events and firing ClosureDeclared; listeners: Attendance applies DEC-276 to today's open sheets and cancels queued absence alerts; Timetable marks the day closed; notification event administration.closure_declared to guardians and staff of the branches (urgent, bypasses quiet hours).
7. ID cards: document type id_card (printable; student and staff layouts, CR80 size on A4 sheets 10 per page, photo, name in English and Urdu, class or designation, registration or employee code, barcode, validity, emergency phone, branch logo); IdCardBatch (select students by class or section, or staff by department; missing photo report before rendering; RenderDocumentBatch). SCR-STU-020 "print ID cards" bulk action now works.
8. Notifications: communication.announcement_published (audience), communication.diary_published (guardians and students; digest per day per child at 16:00 setting communication.diary_digest_time instead of one message per entry), administration.closure_declared.
9. Screens (D10 10.36 and 10.37 R1 rows exactly): SCR-COMM-001 Announcements, SCR-COMM-002 Announcement form (audience builder, schedule, expiry, acknowledgement, SMS copy with cost estimate), SCR-COMM-003 Announcement detail (read and acknowledged lists, remind unread), SCR-COMM-004 Diary entries, SCR-COMM-005 Write diary entry, SCR-COMM-006 Diary review, SCR-COMM-013 Guardian: announcements, SCR-COMM-014 Guardian: diary, SCR-COMM-017 Student: announcements and diary, SCR-ADN-001 Calendar, SCR-ADN-002 Calendar event form, SCR-ADN-003 Declare closure, SCR-ADN-004 ID cards, SCR-ADN-009 Guardian: calendar, SCR-ADN-012 Student: calendar.
10. Definitions: Permissions.php for communication.* per D05 5.7.18 and administration.calendar, closure, event, id_card per D05 5.7.22; Settings communication.diary_review (off), communication.ack_reminder_hours (24), communication.diary_digest_time (16:00); Navigation (Communication group: Announcements, Diary; Administration group: Calendar, Closures, ID cards).
11. decisions.md rows: DEC-185, DEC-249 (rebind), DEC-276, D01 GAP-19.

DATABASE
announcements (ann), announcement_audiences (aau), announcement_reads (ard), diary_entries (dry) per D07 Part 3 7.32; calendar_events (cal) per D07 Part 3 7.33.

BUSINESS RULES
- Announcement bodies are sanitized; scripts and inline event handlers never survive.
- Audiences resolve at publish time and are stored; later joiners do not see old announcements unless the audience is organization-wide.
- Diary entries can be edited only on the same day; edits after publishing are marked.
- Closures set H only for days not yet finalized; finalized days need corrections (DEC-276).
- Holidays and vacations change working days for attendance, leave counting and timetables through one calendar contract (DEC-249).

UI SPEC
- Audience builder with chips per audience row and a live recipient count; SMS copy toggle shows segments and credits.
- Calendar month and list views with type colors from D09 status tokens and text labels; closure banner on dashboards of affected branches.
- ID card preview before batch rendering with the missing photo list.

SECURITY AND PERMISSIONS
- Publishing needs communication.announcement.publish; diary writing only for assigned sections and groups; closures for OW and PR.
- Announcement reads and acknowledgements are visible to authors and MGT only.

TESTS
- Sanitizer strips scripts and event handlers; allowed formatting survives.
- Audience resolution for branch, class, section, role and portal; scheduled publish fires; acknowledgement and reminders.
- Diary same-day edit rule; review flow when enabled; daily digest per child.
- WorkingDayCalendar: holiday, vacation, closure, override and weekend cases; cache invalidation on change; leave day counting and sheet preparation change accordingly.
- Closure: today's open sheets set H, finalized sheets untouched with an attention item, queued absence alerts cancelled, urgent notification sent.
- ID card batch renders with missing photo report.
- Screen tests including portals, audit, isolation suite, RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. On staging, a smog closure is declared for two branches mid-morning: open sheets show H, no absence SMS go out, guardians receive the closure message, and the calendar shows the closure.
2. An announcement with acknowledgement reaches guardians of one class with an Urdu SMS copy; ID cards print for a section.
3. CI green; staging green.

DO NOT
- Do not build messaging, parent-teacher meetings or gate passes (R2), visitors or complaints (R3).
- Do not type holiday dates into code.

OUTPUT
Report per Master section 20 with the closure walkthrough, calendar cases and sample ID cards.
```

### P09.3 Reporting, Dashboards and Attention Items

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D10 Part 1 completely (10.5 dashboard rules, 10.6 widget catalog, 10.7 attention items, 10.8 dashboards by role, 10.9 wireframes, DEC-153 to DEC-159); D10 Part 3 10.38 (SCR-RPT-001 to 004, 006 to 009) and 10.47 (report catalog); D07 Part 3 7.34 (daily_metric_snapshots, saved_report_filters); D02 FR-RPT and AC-RPT-01, AC-RPT-02; D05 5.7.22; D12 Part 2 12.17 (dashboard budgets) and DEC-203 (replica reads); D14 DEC-263, DEC-274.

PHASE GOAL
Show every role what matters today in one place, fast and without leaking data: the metric registry and snapshot computation, the widget registry with separate widget loading, role dashboards with the "View as" switcher, attention items, branch comparison, the complete R1 report catalog on the runner with charts and saved views, guardian and student homes built from widgets, and the 20-branch owner dashboard under 10 seconds.

ALREADY BUILT
PH-00 to PH-08, P09.1 and P09.2: every R1 module with its queries, AttentionItemRegistry contributions from earlier steps, HomeSectionRegistry (DEC-250), basic report runner and ReportRegistry (DEC-263), replica connection.

TASKS
1. Migrations (Reporting): daily_metric_snapshots and saved_report_filters exactly as D07 7.34 (scheduled_reports is R2).
2. MetricRegistry: modules contribute metrics (key, owning module, dimension keys such as section, class, method, compute(organization, branch, date) on the replica, schedule today or nightly or event). Metrics for R1 widgets: student strength and by class, attendance today students and staff (present, absent, late, leave, not marked), attendance trend, chronic absentees, fee collection today and month, billed versus collected, outstanding aging, top defaulters, collection trend, vouchers due this week, staff headcount, expiring documents, notification delivery health, result latest summary.
3. Snapshot computation: commands reporting:snapshots-today every 15 minutes from 07:00 to 16:00 organization time (fan out per organization on queue reports; replica reads per DEC-203 with lag guard), reporting:snapshots-nightly at 01:30 (after core:integrity-check), event refresh for enrollment and publication events (debounced 2 minutes); upsert by uq_dms; SCR-RPT-007 shows last computed time per metric and a force refresh (reporting.snapshot.manage).
4. WidgetRegistry (DEC-154, DEC-274) with every R1 widget in D10 10.6: key, title keys, owning module, data permissions, default roles, width, source snapshot or live, refresh, empty state, drill-down route, release. Endpoint GET /app/reporting/widgets/{key} (and the guardian and student equivalents) returns JSON for one widget after the permission check for the active branch (DEC-159); live widgets cache 60 seconds per user scope; snapshot widgets show "As of" (DEC-156). Re-register HomeSectionRegistry sections as guardian and student widgets with D10 keys and remove HomeSectionRegistry (DEC-274) with an architecture test.
5. Dashboards (D10 10.8): DashboardComposer picks the layout for the active role by DEC-153 precedence, with the "View as" switcher remembered per user; layouts fixed per role in R1 (DEC-157); custom roles use the nearest template order; empty organizations see the onboarding checklist widget; single-branch organizations never see branch columns (PR-10). Quick actions per role in the header (at most 3).
6. Attention items (DEC-155): AttentionItemRegistry runtime with every R1 item in D10 10.7 (approvals waiting, sections not marked, your sections, devices offline, suspicious punches, missing punch requests, marks not submitted, your marks, results ready, cheques, cashier sessions from earlier days, voucher generation not run after the 20th, unbalanced journals, admissions older than 3 days, documents to verify, employee documents expiring, diary to review, notification failures, import errors, period unlocks due to relock); the reporting.attention widget hides zero counts and shows "All clear".
7. Branch comparison: SCR-RPT-006 and the reporting.branch_comparison widget (students, attendance percentage, collection percentage, outstanding) for organization and institution scope users only (FR-RPT-007).
8. Reports: complete SCR-RPT-003 (charts with Chart.js loaded only on dashboards and report pages, D12 12.17; save view), SCR-RPT-002 Report catalog (grouped by module, only permitted reports), SCR-RPT-004 Saved views; register the remaining R1 catalog keys from D10 10.47: attendance.daily_summary, attendance.monthly_register, attendance.chronic_absentees, attendance.late_comers, attendance.staff_monthly, student.strength, student.admissions_withdrawals, student.guardian_contacts, hr.staff_list, hr.document_expiry, leave.summary, notification.sms_usage, audit.sensitive_changes (plus the fee, accounting and result keys already registered). SCR-ATT-023 and SCR-LVE-010 link into the runner for their keys.
9. Charts follow the dataviz rules: one axis per chart, categorical colors in fixed order from the D09 chart palette validated for color vision deficiency, legend for two or more series, values in text tokens, a table view toggle, light and dark variants.
10. Performance: dashboard frame under 1 s; each snapshot widget under 300 ms p95; owner dashboard for 20 branches under 10 s end to end on S2 (D10 10.5, PH-09 exit criterion); k6 scenario tests/Load/owner-dashboard.js.
11. Screens (D10 10.38 R1 rows exactly): SCR-RPT-001 Staff dashboard (app.home), SCR-RPT-002 Report catalog, SCR-RPT-003 Report runner, SCR-RPT-004 Saved views, SCR-RPT-006 Branch comparison, SCR-RPT-007 Snapshot status, SCR-RPT-008 Guardian home (guardian.home), SCR-RPT-009 Student home (student.home). The dashboard wireframes in D10 10.9 are the layout reference.
12. Definitions: Permissions.php for reporting.* per D05 5.7.22; Navigation (Home; Reports group: Catalog, Saved views, Branch comparison).
13. decisions.md rows: DEC-153 to DEC-159, DEC-165, DEC-203, DEC-274.

DATABASE
daily_metric_snapshots (dms), saved_report_filters (srf) per D07 Part 3 7.34.

BUSINESS RULES
- A widget renders only when the user holds its data permission in the active branch; numbers follow the same scopes and masking as their source lists (BR-RPT-01, DEC-154).
- Every number drills down to the list that produced it with the same filters (AC-RPT-01).
- Branch users never see other branches' numbers (AC-RPT-02); single-branch organizations never see branch comparison.
- Snapshots are computed from the replica and stamped with their time; live widgets carry no stamp.

UI SPEC
- Dashboard order: needs attention, today's numbers, work lists, trends; 12-column grid with widths 3, 6, 9 or 12; mobile stacks in the same order.
- Each widget has skeleton, empty, error (retry) and loaded states; a failing widget never blocks others.
- Guardian home at 360px: pending confirmations first, then child today, fee due, diary, upcoming exams, latest result, announcements; child switcher in the header.

SECURITY AND PERMISSIONS
- Widget endpoints check permission and branch per request; cache keys include organization, branch, user scope and role.
- Exports from the runner are masked and formula-safe; reports read from the replica.

TESTS
- AC-RPT-01: the owner's attendance KPI drill-down lists exactly the absentees counted.
- AC-RPT-02: a branch principal's dashboard and widget endpoints never include other branches' numbers.
- Widget permission matrix: each widget hidden without its data permission; masking in widgets.
- Snapshot jobs idempotent; lag guard skips when replica lag is over 30 s and retries.
- Attention items appear and clear with their underlying state.
- DEC-274 architecture test: no reference to HomeSectionRegistry remains.
- Performance: owner dashboard for 20 branches under 10 s; widget p95 under 300 ms (k6 evidence).
- Screen tests including portals, isolation suite, RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. On staging, the owner, principal, class teacher, accountant, cashier, exam controller, HR manager, admission officer, guardian and student dashboards render per D10 10.8 with live and snapshot widgets.
2. The 20-branch owner dashboard loads in under 10 s (evidence).
3. The full R1 report catalog runs with print and export.
4. CI green; staging green.

DO NOT
- Do not build scheduled reports or user-arranged dashboards (R2) or advanced analytics (R3).
- Do not compute dashboard numbers by scanning transaction tables on page load; use snapshots or short-cached queries.

OUTPUT
Report per Master section 20 with dashboard screenshots per role, timing evidence and the report catalog list.
```

### P09.4 Production Build, Partitions, Backups and Monitoring

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D12 Part 2 12.26 to 12.35 completely (environments, CI, production topology and host routing, deployment process, migration rules, backups and DR, version gates, release management, monitoring and alerting, operations calendar, runbooks) and DEC-203 to DEC-214; D12 Part 1 12.7 to 12.10 (headers, secrets, database privileges, logging); D07 Part 4 7.50 (partitioning plan); D14 Part 1 DEC-231 and DEC-234; D14 DEC-277.

PHASE GOAL
Stand up production exactly as D12 describes and prove it can be operated: hosts and roles from one image, database primary and replica with privilege separation, two Redis roles, Gotenberg and storage, partitions switched on with maintenance and archive commands, continuous and nightly backups with an off-site immutable copy, restore and replica promotion drills, monitoring, alerts, dead-man's switches, status page, runbooks, the deploy pipeline with smoke tests, and the freeze calendar.

ALREADY BUILT
PH-00 to PH-08, P09.1 to P09.3: the complete R1 application on staging; CI with all gates; Docker image build; health endpoints (/up and internal.health.ready); integrity checks; core:anonymize-copy and core:requeue-stuck if built earlier (build them here if not).

TASKS
1. Infrastructure as code in infra/ (Terraform for cloud resources and Ansible or the provider's container platform for hosts; choose one per provider and record it in docs/decisions): edge (2 Caddy with wildcard certificate by DNS challenge for *.darsora.app; on-demand TLS calling internal.domains.check from R2 only), app nodes (web role), worker nodes (Horizon supervisors per queue list), one scheduler node, MySQL 8.4 primary and replica with GTID, redis-cache (allkeys-lru, no persistence) and redis-queue (noeviction, AOF every second) per DEC-204, Gotenberg 8 (2 or more containers, no outbound network), optional ClamAV, S3-compatible private buckets with versioning, off-site bucket with 35-day object lock, secret store. Host routing per D12 12.28 including separate FPM pools for hooks and adms hosts.
2. Database users (DEC-190): app (data only; SELECT and INSERT only on audit_logs and login_events), migration (deploy job only), read-only replica user for reports and exports, maintenance user for partitions; grants scripted and tested in CI with the production grant set (Audit privilege test in D12 12.24).
3. Partitions (DEC-277): migration core_switch_on_partitions for every [L] table in D07 7.50 with monthly RANGE COLUMNS partitions from 2027-01 to three months ahead plus p_future (student_timeline_events yearly); commands core:partitions:maintain (1st of each month, splits p_future so three future periods exist) and core:partitions:archive (per partition past retention: one compressed NDJSON file per organization to object storage, row count and checksum verification, archive metadata recorded, then drop); retention values are platform settings; run on staging first with an anonymized copy and time it.
4. Backups and DR (DEC-208, DEC-209): binlog streaming to object storage (continuous, 35 days), nightly physical backup at 02:30 PKT (daily 35 days, monthly 12 months, yearly 7 years), encrypted, off-site copy with object lock; scripts ops/backup/verify that report latest binlog age and full backup age; weekly automated restore to a scratch instance with row counts and checksums; single-tenant restore tooling (scratch point-in-time restore, tenant-scoped diff from audit, repair through Actions or a reviewed script); replica promotion procedure. Run the first full restore drill, one single-tenant restore drill and one replica promotion drill on production-like staging and on production before tenants exist; record times against RTO and RPO.
5. Deploy pipeline (D12 12.29, DEC-206, DEC-207, DEC-212): one signed image per commit promoted unchanged; pre-flight checks (backup ages, replica lag, no open SEV-1 or SEV-2, inside the deploy window or a hotfix flag with approval); migrations with the migration user and the expand-only rule; rolling web nodes with /up; horizon:terminate for workers; scheduler swap; smoke tests (login page per portal, internal.health.ready, one tenant read, a queue round trip per queue, one PDF render, a webhook to the test receiver); 30-minute watch; rollback by previous image tag. Freeze calendar file ops/freeze-calendar.yaml read by the pipeline.
6. Monitoring and alerting (D12 12.33, DEC-213): external checks every minute on tenant host, guardian portal, hooks and adms; metrics for request p50, p95, p99 per route group and top 20 organizations; Sentry with scrubbing rules; Horizon queue waits and failed jobs; database, Redis, disk, certificates; dead-man's switch ping for every scheduled command (a scheduler macro adds the ping so no command is missed, with a test listing all scheduled commands); business health signals; every alert in the 12.33 table with its threshold, severity and route; on-call rota with primary and backup; status page status.darsora.app.
7. Operations: runbooks in docs/runbooks/ for every row of D12 12.35 with exact commands, verification and rollback; operations calendar tasks scheduled or ticketed per 12.34; core:requeue-stuck and core:anonymize-copy commands with tests (anonymize replaces names, numbers, CNIC and B-Form data, mobiles and emails with consistent fakes and keeps referential integrity).
8. Security baseline in production: headers and CSP nonce per D12 Part 1 12.7 verified by an automated check; TLS 1.2 minimum; secrets only from the secret store (never in images); log redaction verified on production log samples.
9. decisions.md rows: DEC-190, DEC-203 to DEC-214, DEC-277, and the infrastructure tool choice.

DATABASE
No new business tables. Converts the [L] tables to partitioned tables per DEC-277; archive metadata is stored in platform settings tables.

BUSINESS RULES
- Production data never leaves production except as anonymized copies made with approval (DEC-214).
- Deploys happen only in the window unless a hotfix is approved (DEC-212); migrations are expand-only and never run down() (DEC-207).
- Archives are verified before any partition is dropped; business tables are never dropped or truncated.
- Every scheduled command reports success to a dead-man's switch.

UI SPEC
- SCR-PLT-016 System health shows queues, failed jobs with retry and discard, Gotenberg, providers, webhook failures, partition status, backup ages and the last restore drill result.
- Status page lists tenant web, guardian portal, payments (R2 placeholder), SMS and device intake components.

SECURITY AND PERMISSIONS
- Only platform engineering can run deploys, restores and anonymized copies; every run logged with the operator.
- The app database user cannot alter schema or modify audit rows (CI test with production grants).
- Internal endpoints answer only private network CIDRs (DEC-231).

TESTS
- Partition migration on a seeded staging copy: row counts unchanged, queries use partition pruning (EXPLAIN shows partitions), maintain creates three future partitions, archive writes verified files and drops only verified partitions.
- Grants test with production grants: UPDATE and DELETE on audit_logs refused for the app user.
- Scheduler test: every scheduled command has a dead-man's switch ping.
- anonymize-copy test: no original personal values remain; foreign keys intact.
- Restore drill, single-tenant restore and replica promotion results within RTO and RPO (evidence documents).
- Smoke test script passes against staging and production.

ACCEPTANCE CRITERIA
1. Production is live with no tenants, monitored, alerting to the on-call rota, and the status page up.
2. A full restore drill, a single-tenant restore and a replica promotion pass with times recorded (PH-09 exit criterion).
3. A full deploy and a rollback run through the pipeline in the window with smoke tests green.
4. CI green; staging and production green.

DO NOT
- Do not copy production data to lower environments except through core:anonymize-copy with approval.
- Do not store secrets in images, repositories or environment files committed to Git.
- Do not drop partitions without verified archives.

OUTPUT
Report per Master section 20 with the topology as built, drill timings, alert list and the runbook index.
```

### P09.5 UG-2 Slot: Laravel 14

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D12 Part 2 DEC-200 and 12.31 (version gates), D13 DEC-221 and 13.8, docs/runbooks/upgrade-gate.md, and the official Laravel 14 upgrade guide and release notes (read them from the official Laravel documentation at the time of the step; do not rely on memory).

PHASE GOAL
Move the application to Laravel 14 before R1 hardening if, and only if, the DEC-221 condition holds, so the pilot and launch run on the final R1 stack; otherwise record the skip and leave the upgrade to P11.0.

ALREADY BUILT
The complete R1 application on Laravel 13 with PHP 8.4 (UG-1), CI on PHP 8.4 and 8.5, production built in P09.4.

TASKS
1. Check the condition (DEC-221): Laravel 14 is released and has at least two point releases by 1 Jul 2027; every first-party and third-party package in composer.json declares Laravel 14 support (Fortify, Sanctum, Horizon, Pest and its plugins, Larastan, maatwebsite/excel, bacon/bacon-qr-code, intervention/image, libphonenumber, brick/math, HTMLPurifier wrapper, Sentry SDK). Record the evidence (versions and links) in the report. If any check fails, stop: write docs/decisions/ug-2-deferred.md with the reasons and the P11.0 date, and end the step with the report.
2. If the condition holds: create branch step/P09.5-ug-2-laravel-14; update composer constraints; follow the upgrade guide item by item, listing each item as applied or not applicable in the report; keep the forbidden-feature rules from the Master Context (no property hooks or asymmetric visibility on Eloquent models; no implicit nullable parameters); update Larastan level and baseline only by fixing code, never by adding ignores for new errors.
3. Run the full test suite, the nightly group (gotenberg), mutation gates, browser tests, the full k6 load set on staging, and the isolation suite; compare performance against the P09.3 and P09.4 evidence.
4. Update docs/blueprint copies only where D12 12.31 version facts change (record a decision row), the upgrade runbook, and the Master Context stack section through a decisions.md row (the Master Context file itself changes only by product owner approval).

DATABASE
No schema changes.

BUSINESS RULES
- The upgrade changes no behaviour; every existing test passes without modification except framework API updates listed in the report.

UI SPEC
- No UI changes; screenshots of five key screens before and after for visual comparison.

SECURITY AND PERMISSIONS
- D12 12.24 security suite green before and after; dependency audit clean.

TESTS
- Full suite, nightly, mutation, browser, load and isolation runs all green on the upgraded branch.

ACCEPTANCE CRITERIA
1. Either Laravel 14 runs on staging with all gates green, or ug-2-deferred.md records why it was deferred to P11.0.
2. CI green; staging green.

DO NOT
- Do not upgrade if any condition fails.
- Do not add Larastan ignores or skip tests to get green.
- Do not change behaviour or schema in this step.

OUTPUT
Report per Master section 20 with the condition evidence, the upgrade guide checklist and before and after performance numbers. Then run the D13 13.6 gate for PH-09.
```

---

## 14.20 PH-10 R1 Hardening, Pilot and Launch

**Exit criteria:**
- MS-11 is reached.
- The penetration test has zero cross-tenant findings.
- Pilot organizations have printed vouchers, collected fees, marked attendance and published results in production.

### P10.1 Hardening

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D12 Part 2 12.17 (budgets), 12.22 (load scenarios), 12.23 (testing strategy and coverage), 12.24 (required security and integrity tests) and 12.25 (definition of done); D10 Part 3 10.50 (coverage checks); D09 (accessibility rules and tokens); D13 13.6 and 13.7; D01 A-16; D14 DEC-279, DEC-280.

PHASE GOAL
Make R1 complete and solid before real schools use it: every R1 screen present and covered, every D12 12.24 test green, Urdu complete for the guardian portal and printed documents, accessibility verified, performance tuned to the budgets, and the full load set passing on production-like staging.

ALREADY BUILT
The complete R1 application, production environment (P09.4), dashboards and reports (P09.3), and the stack decided in P09.5.

TASKS
1. ScreenCoverageTest for R1: every D10 screen tagged R1 exists with its route name and permission and passes; every permission key is used by at least one screen, widget or action; exceptions listed in tests/Fixtures/screen_coverage_exceptions.php with a reason and a D15 reference (D10 10.50).
2. D12 12.24 full list green and visible as a CI job summary: isolation suite across every tenant model, route, export, job and file path; RouteConventionsTest; RBAC matrix for all 22 tenant role templates and 3 platform roles; masking in screens, exports and API; log redaction; headers; uploads; webhooks (SMS delivery receipts and device webhooks now, payments in R2); payment trust test for client-side success messages on fee screens; SOD-01 to SOD-10 except SOD-06 (R2); immutability of paid vouchers, posted journals and locked periods; impersonation limits; audit privilege; session rules; throttling.
3. Urdu completeness (DEC-280, A-16): TranslationParityTest extended to document views and portal views; native-speaker review of the guardian portal and every printed document (vouchers, receipts, admit cards, report cards, certificates, ID cards, slips) with fixes; Nastaliq rendering checked in PDFs for joins and numerals; RTL layout checks at 360px.
4. Accessibility (DEC-279): Pest 4 browser tests with axe-core on these 30 screens in English and Urdu at 360 px and 1280 px: SCR-IAM-001, 002, 003; SCR-RPT-001, 008, 009; SCR-ATT-002, 011, 012; SCR-STU-001, 002; SCR-ADM-002, 010; SCR-FEE-013, 017, 037; SCR-EXM-010; SCR-RES-006, 017; SCR-LVE-004, 011; SCR-APR-001, 002; SCR-NTF-001; SCR-IMX-002; SCR-TTB-003, 008; SCR-COMM-002; SCR-ADN-001; SCR-GRD-010. Fix every serious or critical violation; keyboard-only walkthroughs for attendance marking, fee collection and marks entry recorded.
5. Performance tuning against D12 12.17: slow query log review on staging under load, missing indexes added (expand-only), N+1 removed (lazy loading disabled outside production already), cache hit rates checked, bundle sizes within budget (portal.js 60 KB, app.js 120 KB gzip), Lighthouse or WebPageTest numbers for the guardian home on a mid-range Android profile (LCP, INP, CLS).
6. Full load set (D12 12.22) on production-like staging with the LoadTestSeeder: attendance-morning, result-day, fee-counter, voucher-batch, import-5k, webhook-flood (SMS delivery receipts and device webhooks in R1), adms-burst, soak 8 hours, plus owner-dashboard from P09.3; every pass criterion met; results stored in docs/evidence/load/R1/.
7. Coverage targets (D12 12.23) met; mutation gates green; nightly group green for 7 consecutive nights.
8. Bug bash: a scripted end-to-end walkthrough per role (D10 10.8 roles) on staging in both languages; every high-severity issue fixed in this step or raised as a P10.1-fix item before P10.3.
9. decisions.md rows: DEC-279, DEC-280, A-16.

DATABASE
Index additions only, expand-only per D12 12.29 migration rules.

BUSINESS RULES
- No R1 feature is added in this step; only completion of tagged R1 screens, fixes and tuning.
- A screen that cannot be completed moves release only under DEC-223 with an updated D10 release column and product owner approval.

UI SPEC
- Every list has empty, filtered-empty, loading and error states (D10 10.2, D09 9.16); verified by the browser tests on the 30 screens.

SECURITY AND PERMISSIONS
- Security suite results attached; dependency and container image scans clean of high and critical issues.

TESTS
- ScreenCoverageTest R1 complete; D12 12.24 suite green; TranslationParityTest extended; axe browser tests green; full load set within budgets; coverage and mutation gates.

ACCEPTANCE CRITERIA
1. RC-2, RC-3 (internal part) and RC-4 evidence complete.
2. Urdu review sign-off from a native speaker for portal and documents.
3. CI green for 7 consecutive nights; staging green.

DO NOT
- Do not add features or change scope.
- Do not lower budgets, coverage targets or gates to pass.

OUTPUT
Report per Master section 20 with links to the evidence folders for coverage, security, accessibility, translations and load.
```

### P10.2 External Checks: Penetration Test, Legal Review and DR Game Day

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D12 Part 1 12.2 to 12.14 (threat model, controls, OWASP mapping, accepted risks, privacy and legal context), D12 Part 2 12.30 (DR drills) and 12.35 (runbooks), D13 13.7 (RC-3, RC-5, RC-8); D14 DEC-278.

PHASE GOAL
Prepare everything external reviewers need, support their work, and close their findings: a penetration test pack and environment, legal review of the privacy notice and data processing terms, and a full DR game day; every finding becomes a P10.2-fix item with its own conversation.

ALREADY BUILT
Hardened R1 on production-like staging and an empty production environment (P10.1).

TASKS
1. Penetration test pack in docs/security/pentest-R1/: scope (hosts, portals, roles, API not in R1, device and SMS webhooks), rules of engagement, test window, two test organizations with seeded data and accounts for every role template including guardian and student, platform support account with an approved support grant, ADMS test device serials and secrets, threat model summary from D12 Part 1, OWASP ASVS L2 and L3 control list (DEC-186) with where each control lives, known accepted risks.
2. A dedicated pentest environment cloned from staging with anonymized data (DEC-214), monitoring on, and alerts routed to the security lead; reset script.
3. Findings intake: docs/security/pentest-R1/findings.md table (ID, severity, title, affected screens or endpoints, status, fix step ID); each high or critical finding gets a P10.2-fix-NNN prompt written from the 14.21 template; medium findings fixed or accepted with a reason by the security lead; every cross-tenant finding is critical by definition.
4. Legal review pack: privacy notice template (organization editable, English and Urdu), consent texts (DEC-257), data processing terms listing processors (SMS, email, hosting, payment providers for R2), retention schedule (D12 12.12), breach notification commitment; lawyer comments tracked to closure (RC-8).
5. DR game day (twice-yearly drill, first run): rebuild staging from the off-site immutable copy within 24 hours, run core:integrity-check, verify a sample of tenants, record timings and lessons; update runbooks.
6. decisions.md rows: DEC-186, DEC-278, accepted risks changes if any.

DATABASE
No schema changes in this step; fixes happen in fix steps.

BUSINESS RULES
- No high or critical finding remains open at P10.4 (RC-3); zero cross-tenant findings is a PH-10 exit criterion.
- Accepted risks need the security lead's written reason and appear in D12 accepted risks.

UI SPEC
- No UI changes in this step.

SECURITY AND PERMISSIONS
- Test accounts and secrets are destroyed after the test; the pentest environment is torn down.

TESTS
- Every fix step adds a regression test reproducing its finding.

ACCEPTANCE CRITERIA
1. Penetration test report received; every high and critical finding closed with fix steps merged and retested; zero cross-tenant findings.
2. Lawyer sign-off on privacy notice template and data processing terms.
3. DR game day completed within 24 hours with integrity checks passing.

DO NOT
- Do not test against production.
- Do not fix findings inside this step's branch; use P10.2-fix-NNN steps.

OUTPUT
Report per Master section 20 with the findings table, legal sign-off record and game day timings.
```

### P10.3 Pilot

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D13 DEC-222 and PH-10; D01 1.18 (success metrics, onboarding within 5 working days); D10 Part 1 (onboarding checklist); D11 Part 2 11.32 (school-year calendar); D12 Part 2 12.29 (deploys) and 12.32 (release and change management); D14 DEC-278.

PHASE GOAL
Run three real organizations on production for four weeks: one Scenario A school, one Scenario B chain and one intermediate college or Cambridge school (DEC-222), each onboarded within five working days, with daily triage and weekly fix releases, until they have printed vouchers, collected fees, marked attendance and published results.

ALREADY BUILT
Hardened R1 in production with monitoring, backups and runbooks; external checks under way or complete.

TASKS
1. Pilot tooling: docs/pilot/R1/ with the pilot plan (organizations, contacts, dates, success measures from D01 1.18), an onboarding tracker per organization (provisioning, structure, academic defaults, finance defaults, imports for staff, students, guardians and opening balances, devices, SMS provider, first vouchers, first attendance day), a data preparation guide for school staff in English and Urdu with the import templates, and a daily triage log.
2. Feedback intake inside the product: a "Report a problem" link in the staff and guardian portals (platform-level form storing the page URL, user role, organization and a description, with optional screenshot upload through FileStorageService) visible only to pilot organizations through a platform feature flag; triage into the issue tracker with labels severity, module and organization.
3. Daily triage (process in the pilot plan): each morning review overnight alerts, failed jobs, triage log and feedback; classify SEV-1 to SEV-4 (D12 Part 1 12.13); SEV-1 and SEV-2 fixed as hotfixes under DEC-212 hotfix rules; others batched into the weekly release in the deploy window.
4. Every fix is a P10.3-fix-NNN step written from the 14.21 template (DEC-278), with a regression test and its own report; the weekly release notes list the fix IDs in English and Urdu.
5. Pilot evidence: per organization, screenshots and counts from production for vouchers printed, payments collected with balanced journals, attendance days finalized with device punches where installed, results published with report cards downloaded by guardians; time to onboard measured against five working days.
6. Pilot exit review with the product owner: open issues by severity, decisions taken, D15 rows affected.

DATABASE
No schema changes outside fix steps.

BUSINESS RULES
- Pilot data is real and stays after GA (DEC-222).
- No feature work during the pilot; only fixes and configuration.
- Hotfixes outside the deploy window need the platform owner's approval and follow the runbook.

UI SPEC
- "Report a problem" is a small link in the user menu, not a floating button; it opens a short form with the current page captured automatically.

SECURITY AND PERMISSIONS
- Feedback screenshots are stored privately per organization and visible only to platform support with an approved support grant.
- Support access to pilot tenants only through owner-approved grants (SCR-PLT-017).

TESTS
- Feedback form stores context without personal data in the URL; flag hides it for non-pilot organizations.
- Each fix step's regression test.

ACCEPTANCE CRITERIA
1. Three organizations live within five working days each.
2. Each printed vouchers, collected fees, marked attendance and published results in production (PH-10 exit criterion evidence).
3. No open SEV-1 or SEV-2 at the end of week 4.

DO NOT
- Do not add features during the pilot.
- Do not access pilot tenant data without an approved support grant.
- Do not deploy outside the window except approved hotfixes.

OUTPUT
Report per Master section 20 with the pilot evidence per organization, the fix list and the exit review notes.
```

### P10.4 General Availability

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D13 13.6 (phase gate) and 13.7 (release checklist RC-1 to RC-10), D12 Part 2 12.31 (version support), 12.32 (release management), and Deliverable 15 (tracking and verification checklist).

PHASE GOAL
Declare R1 generally available only when every release check is proven: phase gates, screens, security, load, drills, monitoring, version support, legal review, release notes and user guides, and the D15 sign-off.

ALREADY BUILT
Everything from PH-00 to P10.3 with evidence folders.

TASKS
1. Assemble docs/release/R1/checklist.md with one section per RC-1 to RC-10, each linking to its evidence (gate reports per phase, ScreenCoverageTest run, security suite and pentest closure, load results within 30 days, drills within 30 days, monitoring and on-call proof, version support dates with at least 3 months left, lawyer sign-off, release notes).
2. Release notes in English and Urdu and user guides for each role's main screens (short task-based guides with screenshots) in docs/user-guides/R1/, also published as in-app help links on the relevant screens.
3. D15 verification checklist completed: every R1 row ticked with evidence links; open exceptions listed with product owner decisions.
4. Plan and pricing readiness on the platform: plans from DEC-233 configured with prices and limits by the commercial team through SCR-PLT-010; the pilot feature flag removed or kept per decision.
5. GA tag: git tag r1.0.0 on the production image; production change record; announcement to pilot organizations; MS-11 recorded.
6. decisions.md rows: GA decision with date and signatories.

DATABASE
No schema changes.

BUSINESS RULES
- GA happens only with every RC item proven; a missing item blocks GA, with no exceptions other than items moved under DEC-223 with product owner approval.

UI SPEC
- Help links on screens open the matching user guide section in the user's language.

SECURITY AND PERMISSIONS
- Release tagging and production change records are made by platform engineering with the product owner's sign-off.

TESTS
- Final full CI run on the release tag; smoke tests on production after the tag deploy.

ACCEPTANCE CRITERIA
1. RC-1 to RC-10 proven with links; D15 signed by the product owner; MS-11 reached on or before 24 Sep 2027 or a new date recorded with reasons.
2. CI green on the release tag; production smoke tests green.

DO NOT
- Do not tag GA with any open high or critical security finding or any failing RC item.

OUTPUT
Report per Master section 20 with the completed release checklist and the D15 sign-off record. Then run the D13 13.6 gate for PH-10.
```

---

## 14.21 Fix Prompt Template (DEC-278)

Use this template for every `P10.2-fix-NNN` and `P10.3-fix-NNN` step, and for later fix steps in R2 and R3 (`P{phase}.{step}-fix-NNN`). Fill in every section. Start a new conversation and paste the Master Context Prompt first.

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Fix ID {P10.3-fix-NNN}. Source: {pentest finding ID or pilot triage entry with date and organization (no personal data)}. Read the step report of the module's original step ({step ID}) and the blueprint sections it cites: {list}.

PHASE GOAL
{One sentence: what is wrong today and what correct behaviour looks like after the fix.}

ALREADY BUILT
The R1 application as of release {tag}. Related code: {module, Actions, screens}.

TASKS
1. Reproduce the problem with a failing test named after the fix ID (tests/Regression/{FixId}Test.php).
2. Fix the cause in the owning module's Action, Service, Query, policy or view, following the Master Context layers and rules.
3. {Further specific tasks.}
4. Update translations, decisions.md or runbooks if the fix changes behaviour, a rule or an operation.

DATABASE
{None, or the expand-only migration with its name and reason.}

BUSINESS RULES
{The rules from the blueprint that the fix restores or clarifies.}

UI SPEC
{None, or the exact screen and state that changes, in English and Urdu.}

SECURITY AND PERMISSIONS
{Permission, scope, masking or audit implications; for security findings, the control and ASVS item restored.}

TESTS
- The regression test fails before and passes after.
- Affected screen tests, isolation suite and RBAC fixture still pass.
- {Specific extra tests.}

ACCEPTANCE CRITERIA
1. The reporter's scenario works on staging (and production after the release for pilot fixes).
2. CI green; staging green.

DO NOT
- Do not change unrelated code or add features.
- Do not weaken tests, gates or budgets.

OUTPUT
Report per Master section 20 at docs/agent/reports/{FixId}.md with cause, fix, test names and release in which it ships.
```

---

## Assumptions (Part 7)

| ID | Assumption |
|---|---|
| A-131 | A penetration testing vendor is booked for August 2027, with a two-week test window and a retest within one week of fixes. |
| A-132 | The three pilot organizations supply their staff, student, guardian and opening balance data in Excel using the import templates. |
| A-133 | The chosen hosting provider offers S3-compatible storage with object lock, managed or self-hosted MySQL 8.4 with GTID replication, and a second location for off-site copies. |
| A-134 | Public holiday dates for Pakistan are reviewed each year by the platform team and supplied as a data file. Organizations add provincial and local holidays themselves. |

## Gaps Found and Added (Part 7)

| ID | Gap | Decision |
|---|---|---|
| GAP-331 | Home sections (DEC-250) and D10 widgets were two mechanisms for the same thing | Widgets replace home sections (DEC-274) |
| GAP-332 | The rule for official period attendance was undefined | DEC-275 |
| GAP-333 | Closure effects on finalized sheets and queued alerts were undefined | DEC-276 |
| GAP-334 | The timing and mechanism for switching on partitions was unspecified | DEC-277 |
| GAP-335 | Pilot and pentest fixes had no step format | Fix prompt template and IDs (DEC-278) |
| GAP-336 | Accessibility had no automated gate | axe-core browser tests (DEC-279) |
| GAP-337 | Urdu completeness was checked only for keys, not for printed documents | DEC-280 |
| GAP-338 | Part 4 device hosts and Part 6 permission references were inconsistent with D12 and D06A | Corrected (corrections table above) |
