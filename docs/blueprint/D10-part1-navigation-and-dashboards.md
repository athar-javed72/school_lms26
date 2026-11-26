# Deliverable 10: Role-Wise Dashboards and Screen Inventory
# Part 1 of 3: Conventions, Navigation and Role Dashboards

**Version:** 1.0 (draft for approval) | **Date:** 24 Nov 2026
**Builds on:**
- D2: functional requirements per module
- D5: role templates (5.5), portal access (5.6), permission matrices (5.7), locked portal sets (5.8), approval types (5.9), masking (5.10)
- D8: route naming and route files (8.6, DEC-128)
- D9: shells, components, patterns and states (9.15 to 9.17)

**Why three parts:** the inventory covers about 500 screens (502 in the final count) across 34 modules and 4 portals. Each part is complete for what it covers.

| Part | Contents |
|---|---|
| **1** | Screen ID scheme, inventory columns, navigation per portal, dashboard rules, widget catalog, attention items, the dashboard for every role, dashboard wireframes |
| 2 | Screen inventory with key screen wireframes: Platform, Organization, Identity, Settings, Core, Audit, Approval, Notification, ImportExport, Document, Academic, Admission, Student, Guardian, HR, Attendance, Leave |
| 3 | Screen inventory with key screen wireframes: Fee, Accounting, Examination, Result, ExternalExam, Timetable, Communication, Administration, Reporting, Learning, Homework, Assessment, Payroll, Library, Transport, Inventory, Discipline; guardian and student portal summary; screen counts; coverage check against D2 |

---

## 10.1 New Decisions

| ID | Decision |
|---|---|
| DEC-151 | **Screen IDs** use `SCR-{CODE}-NNN` with the D2 module codes (for example `SCR-FEE-012`). They are binding for D11 (workflows), D14 (phase prompts) and D15 (verification checklist). Modals and drawers that have their own permission or business action get their own ID; plain confirmations do not. |
| DEC-152 | **Portal home routes** are `/app`, `/guardian`, `/student` and `/platform`, named `app.home`, `guardian.home`, `student.home` and `platform.home`. Reporting owns them through a new route file `routes/home.php`, mounted without a module segment (correction to D8 8.6). |
| DEC-153 | **One dashboard per active role.** A user with several staff roles sees a "View as" switcher on the dashboard. The default is the highest role by this precedence: OW, OA, PR, VP, BM, ACT, HRM, EXC, AC, HOD, ADO, CT, ST, CSH, REC, LBR, TM, TS, ITA, SUP. The choice is remembered per user. Navigation is not role-switched; it always shows everything the user may open (DEC-158). |
| DEC-154 | **Widget contract.** Every widget declares: key, title, owning module, data permissions, default roles, width (1 to 4 columns), source (`snapshot` or `live`), refresh interval, empty state, drill-down route and release. A widget renders only if the user holds its data permission for the active branch, so dashboards never leak data the user can't open elsewhere (D5 5.7.22). |
| DEC-155 | **Attention items.** A new registry, `AttentionItemRegistry` (contract in Core, runtime in Reporting), lets each module contribute actionable counts (sections not marked, marks not submitted, cheques to clear, devices offline). The "Needs attention" widget shows them in severity order, each linking to the screen that resolves it. Pending approvals come from the Approval inbox through the same registry. |
| DEC-156 | **Numbers say how fresh they are.** Snapshot widgets show "As of 10:45 AM" and refresh on the D3 schedule (every 15 minutes for today's figures, nightly for trends). Live widgets show no stamp. |
| DEC-157 | **Layouts are fixed per role in R1.** In R2 users can hide and reorder widgets; the choice is stored in user preferences. Roles never gain widgets their permissions don't allow. |
| DEC-158 | **Navigation is the union.** The sidebar shows every item whose module is entitled and whose view permission the user holds in the active branch, grouped by the fixed groups in 10.4. Self-service items (baseline, DEC-57) are always present under "My work". |
| DEC-159 | **Widget data loads separately.** The dashboard page renders the frame and skeletons; each widget fetches `GET /app/reporting/widgets/{key}` (internal JSON, D8 8.15). A slow or failing widget never blocks the others. Targets: dashboard frame under 1 second, each snapshot widget under 300 ms at the 95th percentile. |

### Corrections to earlier deliverables

| Where | Change |
|---|---|
| D7 7.52, D8 8.12 and 8.17 | The permission sync command is `identity:sync-permissions` (D5 5.13), not `iam:permissions:sync`. |
| D8 8.6 route loader | Adds `home.php` (Reporting only), mounted at `/{portal}` with names `{portal}.home`, using that portal's middleware group. |
| D3 3.23 registries | Adds `AttentionItemRegistry` (contract in Core, runtime in Reporting, DEC-155). |

---

## 10.2 Screen Inventory Columns (Parts 2 and 3)

| Column | Meaning |
|---|---|
| ID | `SCR-{CODE}-NNN` (DEC-151) |
| Screen | Name shown in the page title |
| Type | `list`, `detail`, `form`, `wizard`, `grid` (interactive editing grid), `dashboard`, `report`, `print` (PDF or print view), `modal`, `drawer`, `public` |
| Route name | D8 naming: `{portal}.{module}.{resource}.{action}` |
| Permission | The D5 key checked to open the screen. "Baseline" means a self-service ability (DEC-57). |
| Default roles | Roles holding that permission by default (D5 abbreviations) |
| Rel | First release |
| Key behaviour and states | What matters for building and testing it, beyond the D9 standard states |

Every list screen has the D9 empty, filtered-empty, loading and error states unless the row says otherwise. Every form has validation, error summary, optimistic locking (DEC-136) and audit.

---

## 10.3 Portals at a Glance

| Portal | Host and prefix | Users | Shell (D9 9.17) | Home |
|---|---|---|---|---|
| Platform | `platform.darsora.app/platform` | SA, platform support, platform billing | Platform bar, left navigation | `platform.home` |
| Staff | `{slug}.darsora.app/app` | All 20 staff role templates and custom roles | Navy sidebar, top bar with branch and session | `app.home` |
| Guardian | `{slug}.darsora.app/guardian` | Linked, portal-enabled guardians | Child switcher, bottom navigation | `guardian.home` |
| Student | `{slug}.darsora.app/student` | Students at or above the login threshold | Bottom navigation | `student.home` |
| Public | `{slug}.darsora.app/admissions`, `/verify` | Anyone | Public layout | none |
| Sign-in | `{slug}.darsora.app/login`, `platform.../login` | Everyone | Auth layout | none |

---

## 10.4 Navigation

### Staff portal sidebar (groups and items)

Items appear only when the module is entitled and the user holds the view permission (DEC-158). "Rel" is the release in which the item first appears.

| Group | Item | Route | View permission | Rel |
|---|---|---|---|---|
| **Home** | Dashboard | `app.home` | Baseline | R1 |
| | Approvals | `app.approval.inbox.index` | Baseline (own and assigned) | R1 |
| | Notifications | `app.notification.notifications.index` | Baseline | R1 |
| **My work** | My profile | `app.identity.profile.show` | Baseline | R1 |
| | My attendance | `app.attendance.my_attendance.index` | `attendance.staff_attendance.view` (own) | R1 |
| | My leave | `app.leave.my_leave.index` | Baseline (own requests) | R1 |
| | My classes | `app.academic.my_classes.index` | `academic.teaching_assignment.view` (own) | R1 |
| | My payslips | `app.payroll.my_payslips.index` | `payroll.payslip.view` (own) | R2 |
| **Students** | Students | `app.student.students.index` | `student.student.view` | R1 |
| | Admissions | `app.admission.applications.index` | `admission.application.view` | R1 |
| | Guardians and families | `app.guardian.guardians.index` | `guardian.guardian.view` | R1 |
| | Promotions | `app.student.promotions.index` | `student.promotion.view` | R1 |
| | Transfers and withdrawals | `app.student.transfers.index` | `student.transfer.view` or `student.withdrawal.view` | R1 |
| | Certificates | `app.student.certificates.index` | `student.certificate.view` | R1 |
| | Discipline | `app.discipline.incidents.index` | `discipline.incident.view` | R2 |
| **Academics** | Sessions and classes | `app.academic.sessions.index` | `academic.session.view` | R1 |
| | Subjects | `app.academic.subjects.index` | `academic.subject.view` | R1 |
| | Teaching assignments | `app.academic.teaching_assignments.index` | `academic.teaching_assignment.view` | R1 |
| | Teaching groups | `app.academic.teaching_groups.index` | `academic.teaching_group.view` | R1 |
| | Timetable | `app.timetable.timetables.index` | `timetable.timetable.view` | R1 |
| **Attendance** | Mark attendance | `app.attendance.section_sheets.index` | `attendance.student_attendance.mark` | R1 |
| | Student attendance | `app.attendance.student_attendance.index` | `attendance.student_attendance.view` | R1 |
| | Staff attendance | `app.attendance.staff_attendance.index` | `attendance.staff_attendance.view` (branch) | R1 |
| | Corrections | `app.attendance.corrections.index` | `attendance.correction.view` | R1 |
| | Leave management | `app.leave.applications.index` | `leave.application.view` (branch) | R1 |
| | Devices | `app.attendance.devices.index` | `attendance.device.view` | R1 |
| **Exams and results** | Exams | `app.examination.exam_terms.index` | `examination.exam_term.view` | R1 |
| | Marks entry | `app.examination.mark_sheets.index` | `examination.marks.view` | R1 |
| | Results | `app.result.batches.index` | `result.batch.view` | R1 |
| | Report cards | `app.result.report_cards.index` | `result.report_card.view` | R1 |
| | External exams | `app.external_exam.series.index` | `external_exam.series.view` | R2 |
| | Predicted grades | `app.external_exam.predicted_grades.index` | `external_exam.predicted_grade.view` | R1 |
| **Learning** | Courses | `app.learning.courses.index` | `learning.course.view` | R2 |
| | Homework | `app.homework.assignments.index` | `homework.assignment.view` | R2 |
| | Question bank and quizzes | `app.assessment.quizzes.index` | `assessment.quiz.view` | R2 |
| **Finance** | Fee collection | `app.fee.payments.create` | `fee.payment.create` | R1 |
| | Vouchers | `app.fee.vouchers.index` | `fee.voucher.view` | R1 |
| | Defaulters | `app.fee.defaulters.index` | `fee.defaulter.view` | R1 |
| | Fee setup | `app.fee.structures.index` | `fee.structure.view` | R1 |
| | Concessions and scholarships | `app.fee.concessions.index` | `fee.concession.view` | R1 |
| | Accounts | `app.accounting.journals.index` | `accounting.journal.view` | R1 |
| | Payroll | `app.payroll.runs.index` | `payroll.run.view` | R2 |
| **People** | Employees | `app.hr.employees.index` | `hr.employee.view` | R1 |
| | Staff directory | `app.hr.directory.index` | `hr.directory.view` | R1 |
| **Communication** | Announcements | `app.communication.announcements.index` | `communication.announcement.view` | R1 |
| | Diary | `app.communication.diary_entries.index` | `communication.diary.view` | R1 |
| | Messages | `app.communication.message_threads.index` | `communication.message.view` | R2 |
| | Parent-teacher meetings | `app.communication.ptm_events.index` | `communication.ptm.view` | R2 |
| **Operations** | Calendar | `app.administration.calendar.index` | Baseline | R1 |
| | ID cards | `app.administration.id_cards.index` | `administration.id_card.view` | R1 |
| | Gate passes | `app.administration.gate_passes.index` | `administration.gate_pass.view` | R2 |
| | Visitors | `app.administration.visitors.index` | `administration.visitor.view` | R3 |
| | Complaints | `app.administration.complaints.index` | `administration.complaint.view` | R3 |
| | Library | `app.library.circulation.index` | `library.circulation.view` | R3 |
| | Transport | `app.transport.routes.index` | `transport.route.view` | R2 |
| | Inventory and assets | `app.inventory.items.index` | `inventory.item.view` | R3 |
| **Reports** | Report catalog | `app.reporting.reports.index` | Any report's permission | R1 |
| | Scheduled reports | `app.reporting.scheduled_reports.index` | `reporting.scheduled_report.view` | R2 |
| **Settings** | Organization | `app.organization.profile.show` | `organization.profile.view` | R1 |
| | Users and roles | `app.identity.users.index` | `identity.user.view` | R1 |
| | Settings | `app.settings.settings.index` | `settings.setting.view` | R1 |
| | Notification templates and rules | `app.notification.templates.index` | `notification.template.view` | R1 |
| | Approval workflows | `app.approval.workflows.index` | `approval.workflow.view` | R1 |
| | Document templates | `app.document.templates.index` | `document.template.view` | R1 |
| | Imports and exports | `app.import_export.import_jobs.index` | `import_export.history.view` | R1 |
| | Period locks | `app.core.period_locks.index` | `core.period_lock.view` | R1 |
| | Audit log | `app.audit.logs.index` | `audit.log.view` | R1 |

### Guardian portal

| Bottom nav (below 1024px) | Route | Permission (D5 5.8) |
|---|---|---|
| Home | `guardian.home` | Baseline |
| Attendance | `guardian.attendance.student_days.index` | `attendance.student_attendance.view` |
| Fees | `guardian.fee.vouchers.index` | `fee.voucher.view` |
| Results | `guardian.result.student_results.index` | `result.student_result.view` |
| More | menu | |

"More" menu: Diary, Homework (R2), Timetable, Date sheet and admit card, Announcements, Leave requests, Pickup persons, Messages (R2), Parent-teacher meetings (R2), Transport (R2), Documents, Profile change requests, External exams (R2), Settings and language, Help.

### Student portal

| Bottom nav | Route | Permission |
|---|---|---|
| Home | `student.home` | Baseline |
| Timetable | `student.timetable.my_timetable.show` | `timetable.timetable.view` |
| Homework | `student.homework.assignments.index` | `homework.assignment.view` (R2; R1 shows Diary) |
| Learn | `student.learning.courses.index` | `learning.lecture.view` (R2; R1 shows Results) |
| More | menu | |

"More" menu: Attendance, Results and report cards, Date sheet and admit card, Quizzes (R2), Announcements, Fees (if enabled, A-43), External exams (R2), Profile change requests, Settings and language, Help.

### Platform portal

Organizations, Plans, Module entitlements, Usage, Support access, Announcements, Platform users, Queue and system health, Audit.

---

## 10.5 Dashboard Rules

| Rule | Detail |
|---|---|
| Scope line | Every staff dashboard starts with the active branch (or "All branches"), the session and today's date, with a refresh time for snapshot data |
| Order | Needs attention, then today's numbers, then lists of work, then trends; on mobile widgets stack in that order |
| Width | 12-column grid on desktop: widgets take 3, 6, 9 or 12 columns (widths 1 to 4); 4 columns on mobile where every widget is full width |
| Drill-down | Every number links to the list that produced it, with the same filters applied |
| Empty organization | New organizations see an onboarding checklist widget (`organization.onboarding_checklist`) instead of empty charts until the checklist is complete |
| All branches mode | Owners and organization admins see branch comparison rows; branch users never see other branches' numbers |
| Single branch | Organizations with one branch never see branch columns or the branch comparison widget (PR-10) |
| Privacy | Widgets apply masking and visibility exactly like their source screens (DEC-154) |
| Performance | D1 outcome 2: an owner of 20 branches sees today's attendance, collection, outstanding and pending approvals on one screen in under 10 seconds. Snapshots make this a single indexed read per widget. |

---

## 10.6 Widget Catalog

Source: `snapshot` reads `daily_metric_snapshots` (D7 7.34); `live` runs a Query class with a short cache (60 seconds unless noted). Width in grid units (1 = 3 columns).

### Cross-module and staff self-service

| Key | Title | Data permission | Source, refresh | Width | Drill-down | Rel |
|---|---|---|---|---|---|---|
| `reporting.attention` | Needs attention | Per item (10.7) | live, 60 s | 2 | Item links | R1 |
| `approval.my_pending` | Waiting for my approval | Baseline (assigned requests) | live | 2 | `app.approval.inbox.index` | R1 |
| `organization.onboarding_checklist` | Finish setting up | `organization.onboarding.manage` | live | 4 | Onboarding wizard | R1 |
| `attendance.my_checkin` | My attendance today | `attendance.staff_attendance.mark` (own) | live | 1 | `app.attendance.my_attendance.index` | R1 |
| `timetable.my_today` | My classes today | `timetable.timetable.view` (own) | live | 2 | `app.timetable.my_timetable.show` | R1 |
| `leave.my_balance` | My leave balance | `leave.balance.view` (own) | live | 1 | `app.leave.my_leave.index` | R1 |
| `administration.upcoming` | Holidays and events | Baseline | live, 1 h | 1 | `app.administration.calendar.index` | R1 |
| `communication.latest_announcements` | Announcements | Baseline (audience) | live | 2 | `app.communication.announcements.index` | R1 |

### Students, admission and attendance

| Key | Title | Data permission | Source, refresh | Width | Drill-down | Rel |
|---|---|---|---|---|---|---|
| `student.strength` | Students | `student.student.view` | snapshot, nightly + on enrollment events | 1 | `app.student.students.index` | R1 |
| `student.strength_by_class` | Strength by class | `student.student.view` | snapshot, nightly | 2 | Students filtered by class | R1 |
| `admission.pipeline` | Admissions this session | `admission.application.view` | live | 2 | `app.admission.applications.index` | R1 |
| `admission.seats` | Seats available | `admission.seat.view` | live, 5 min | 2 | `app.admission.seats.index` | R1 |
| `student.pending_documents` | Documents to verify | `student.document.verify` | live | 1 | Documents filtered `pending` | R1 |
| `attendance.today_students` | Student attendance today | `attendance.report.view` | snapshot, 15 min | 1 | `app.attendance.student_attendance.index` | R1 |
| `attendance.unmarked_sections` | Sections not marked | `attendance.student_attendance.view` (branch) | live | 2 | Section sheets filtered `not_marked` | R1 |
| `attendance.my_sections_to_mark` | Mark attendance | `attendance.student_attendance.mark` | live | 2 | `app.attendance.section_sheets.show` | R1 |
| `attendance.today_staff` | Staff attendance today | `attendance.staff_attendance.view` (branch) | snapshot, 15 min | 1 | `app.attendance.staff_attendance.index` | R1 |
| `attendance.absent_students_today` | Absent today | `attendance.student_attendance.view` | live | 2 | Student attendance filtered `A` | R1 |
| `attendance.chronic_absentees` | Below attendance threshold | `attendance.report.view` | snapshot, nightly | 2 | Attendance report | R1 |
| `attendance.trend_30d` | Attendance, last 30 days | `attendance.report.view` | snapshot, nightly | 2 | Attendance report | R1 |
| `attendance.device_health` | Attendance devices | `attendance.device.view` | live | 1 | `app.attendance.devices.index` | R1 |
| `leave.staff_on_leave_today` | Staff on leave today | `leave.calendar.view` | live, 5 min | 1 | `app.leave.calendar.index` | R1 |

### Finance

| Key | Title | Data permission | Source, refresh | Width | Drill-down | Rel |
|---|---|---|---|---|---|---|
| `fee.today_collection` | Collection today | `fee.report.view` | snapshot, 15 min | 1 | Daily collection report | R1 |
| `fee.month_billed_collected` | This month: billed and collected | `fee.report.view` | snapshot, 15 min | 2 | Monthly collection report | R1 |
| `fee.outstanding_aging` | Outstanding by age | `fee.defaulter.view` | snapshot, nightly | 2 | `app.fee.defaulters.index` | R1 |
| `fee.top_defaulters` | Largest outstanding | `fee.defaulter.view` | snapshot, nightly | 2 | Defaulters sorted by amount | R1 |
| `fee.collection_trend` | Collection, last 12 months | `fee.report.view` | snapshot, nightly | 2 | Collection report | R1 |
| `fee.my_cashier_session` | My cash counter | `fee.cashier_session.view` (own) | live | 2 | `app.fee.cashier_sessions.show` | R1 |
| `fee.my_receipts_today` | My receipts today | `fee.report.view` (own daily cash) | live | 2 | Payments filtered to me, today | R1 |
| `fee.cheques_to_clear` | Cheques to clear | `fee.cheque.view` | live | 1 | `app.fee.cheques.index` | R1 |
| `fee.vouchers_due_week` | Vouchers due this week | `fee.voucher.view` | snapshot, nightly | 1 | Vouchers filtered by due date | R1 |
| `accounting.cash_bank` | Cash and bank balances | `accounting.ledger.view` | live, 5 min | 2 | `app.accounting.ledgers.show` | R1 |
| `accounting.journals_to_approve` | Journals to approve | `accounting.journal.approve` | live | 1 | Journals filtered `draft` | R1 |
| `payroll.run_status` | Payroll this month | `payroll.run.view` | live | 2 | `app.payroll.runs.show` | R2 |

### Exams and results

| Key | Title | Data permission | Source, refresh | Width | Drill-down | Rel |
|---|---|---|---|---|---|---|
| `examination.marks_progress` | Marks submission | `examination.completion.view` | live, 5 min | 2 | `app.examination.completion.show` | R1 |
| `examination.my_marks_due` | My marks to enter | `examination.marks.mark` | live | 2 | `app.examination.mark_sheets.edit` | R1 |
| `examination.reviews_pending` | Submissions to review | `examination.marks_review.approve` | live | 1 | Submissions filtered `submitted` | R1 |
| `examination.upcoming` | Upcoming exams | `examination.date_sheet.view` | live, 1 h | 2 | `app.examination.date_sheets.show` | R1 |
| `result.latest_summary` | Latest results | `result.analytics.view` | snapshot, on publish | 2 | `app.result.analytics.show` | R1 |
| `external_exam.deadlines` | Exam entry deadlines | `external_exam.series.view` | live, 1 h | 2 | `app.external_exam.series.index` | R2 |

### People and communication

| Key | Title | Data permission | Source, refresh | Width | Drill-down | Rel |
|---|---|---|---|---|---|---|
| `hr.headcount` | Staff | `hr.employee.view` | snapshot, nightly | 1 | `app.hr.employees.index` | R1 |
| `hr.expiring_documents` | Documents expiring in 30 days | `hr.document.view` | snapshot, nightly | 2 | Employee documents filtered | R1 |
| `communication.diary_to_review` | Diary entries to review | `communication.diary_review.approve` | live | 1 | Diary filtered `pending_review` | R1 |
| `communication.my_diary_today` | Today's diary | `communication.diary.create` | live | 2 | `app.communication.diary_entries.create` | R1 |
| `homework.to_review` | Homework to review | `homework.submission.approve` | live | 2 | Submissions filtered `submitted` | R2 |
| `assessment.to_grade` | Quiz answers to grade | `assessment.grading.manage` | live | 1 | Grading queue | R2 |
| `learning.course_progress` | Course progress | `learning.progress.view` | snapshot, nightly | 2 | Course progress report | R2 |
| `timetable.substitutions_today` | Substitutions today | `timetable.substitution.view` | live | 2 | `app.timetable.substitutions.index` | R2 |
| `discipline.recent_incidents` | Recent incidents | `discipline.incident.view` | live | 2 | `app.discipline.incidents.index` | R2 |

### Operations and IT

| Key | Title | Data permission | Source, refresh | Width | Drill-down | Rel |
|---|---|---|---|---|---|---|
| `notification.delivery_health` | Message delivery | `notification.delivery_log.view` | snapshot, 15 min | 2 | `app.notification.delivery_logs.index` | R1 |
| `identity.account_alerts` | Locked accounts and failed logins | `identity.user.view` + `audit.login_history.view` | live | 1 | Users filtered `locked` | R1 |
| `import_export.recent_jobs` | Recent imports and exports | `import_export.history.view` | live | 2 | `app.import_export.import_jobs.index` | R1 |
| `transport.capacity` | Route capacity | `transport.route.view` | live, 5 min | 2 | `app.transport.routes.index` | R2 |
| `transport.trips_today` | Trips today | `transport.boarding.view` | live | 2 | Boarding log | R3 |
| `library.overdue` | Overdue books | `library.circulation.view` | snapshot, nightly | 1 | Loans filtered `overdue` | R3 |
| `inventory.low_stock` | Low stock | `inventory.stock.view` | snapshot, nightly | 1 | Stock filtered below reorder | R3 |

### Multi-branch

| Key | Title | Data permission | Source, refresh | Width | Drill-down | Rel |
|---|---|---|---|---|---|---|
| `reporting.branch_comparison` | Branches today | `reporting.branch_comparison.view` | snapshot, 15 min | 4 | Switch to that branch's dashboard | R1 |

### Guardian portal

| Key | Title | Permission (D5 5.8) | Source | Rel |
|---|---|---|---|---|
| `guardian.child_today` | Today: attendance and arrival time | `attendance.student_attendance.view` | live | R1 |
| `guardian.child_fee_due` | Fee due | `fee.voucher.view` | live | R1 |
| `guardian.child_diary` | Diary for today | `communication.diary.view` | live | R1 |
| `guardian.child_upcoming_exams` | Upcoming exams | `examination.date_sheet.view` | live | R1 |
| `guardian.child_latest_result` | Latest result | `result.student_result.view` | live | R1 |
| `guardian.announcements` | Announcements | Baseline (audience) | live | R1 |
| `guardian.child_homework_due` | Homework due | `homework.assignment.view` | live | R2 |
| `guardian.child_transport` | Bus and stop | `transport.allocation.view` | live | R2 |
| `guardian.pending_confirmations` | Needs your response (leave status, entry confirmation, profile change approvals) | Per item | live | R1 |

### Student portal

| Key | Title | Permission | Source | Rel |
|---|---|---|---|---|
| `student.next_period` | Next class | `timetable.timetable.view` | live | R1 |
| `student.due_soon` | Due soon | `homework.assignment.view`, `assessment.quiz.view` (R2); diary (R1) | live | R1 |
| `student.continue_learning` | Continue learning | `learning.lecture.view` | live | R2 |
| `student.my_attendance_month` | My attendance this month | `attendance.student_attendance.view` | live | R1 |
| `student.latest_result` | Latest result | `result.student_result.view` | live | R1 |
| `student.announcements` | Announcements | Baseline | live | R1 |

### Platform portal

| Key | Title | Permission | Source | Rel |
|---|---|---|---|---|
| `platform.tenants_by_state` | Organizations by state | `platform.organization.view` | live, 5 min | R1 |
| `platform.new_organizations` | New organizations (30 days) | `platform.organization.view` | live, 1 h | R1 |
| `platform.usage_limits` | Near plan limits | `platform.usage.view` | snapshot, nightly | R1 |
| `platform.support_sessions` | Active support sessions | `platform.support_access.view` | live | R1 |
| `platform.system_health` | Queues, failed jobs, Gotenberg, providers | `platform.organization.view` (SA, support) | live, 60 s | R1 |
| `platform.webhook_failures` | Failed webhook processing | `platform.organization.view` | live | R1 |

---

## 10.7 Attention Items (DEC-155)

Each item is a count with a severity and a link. The widget hides items with a zero count and shows "All clear" when nothing remains.

| Item | Module | Shown to (permission) | Severity | Link |
|---|---|---|---|---|
| Approvals waiting for you | Approval | Baseline (assigned) | High after 24 hours, otherwise normal | Approvals inbox |
| Sections not marked after the marking window | Attendance | `attendance.student_attendance.view` (branch) | High | Section sheets, not marked |
| Your sections not marked today | Attendance | `attendance.student_attendance.mark` | High | Mark attendance |
| Attendance devices offline | Attendance | `attendance.device.view` | High | Devices |
| Suspicious punches | Attendance | `attendance.suspicious_alert.view` | Normal | Suspicious alerts |
| Missing punch requests | Attendance | `attendance.missing_punch.approve` | Normal | Approvals inbox, filtered |
| Marks not submitted, exam closes in 3 days or less | Examination | `examination.completion.view` | High | Completion |
| Your marks to enter | Examination | `examination.marks.mark` | Normal (high in last 2 days) | Mark sheets |
| Results ready to publish | Result | `result.publication.create` | Normal | Result batches |
| Cheques due to clear | Fee | `fee.cheque.view` | Normal | Cheques |
| Cashier sessions open from earlier days | Fee | `fee.cashier_session.view` (branch) | High | Cashier sessions |
| Voucher generation not run for next month (after the 20th) | Fee | `fee.voucher.generate` | Normal | Voucher generation |
| Gateway payments awaiting verification (R2) | Fee | `fee.payment.view` | High | Gateway transactions |
| Unbalanced or unposted journals | Accounting | `accounting.journal.view` | High | Journals |
| Admission applications older than 3 days without action | Admission | `admission.application.update` | Normal | Applications |
| Student documents to verify | Student | `student.document.verify` | Normal | Documents |
| Employee documents expiring in 14 days | HR | `hr.document.view` | Normal | Employee documents |
| Diary entries waiting for review | Communication | `communication.diary_review.approve` | Normal | Diary |
| Notification failures in last 24 hours above 5% | Notification | `notification.delivery_log.view` | High | Delivery log |
| Import jobs with errors | ImportExport | `import_export.history.view` | Normal | Import history |
| Period unlocks due to relock today | Core | `core.period_lock.view` | Normal | Period locks |
| Exam entry deadline within 7 days with unconfirmed entries (R2) | ExternalExam | `external_exam.entry.view` | High | Entries |
| Payroll run awaiting approval or payment (R2) | Payroll | `payroll.run.view` | Normal | Payroll runs |
| Overdue library books (R3) | Library | `library.circulation.view` | Normal | Loans |
| Low stock items (R3) | Inventory | `inventory.stock.view` | Normal | Stock |
| Vehicle documents expiring in 30 days (R3) | Transport | `transport.vehicle.view` | Normal | Vehicles |

---

## 10.8 Dashboards by Role

Widgets listed in display order. Quick actions are buttons in the dashboard header (at most 3; the first is the page's primary action, UX-02). A widget listed for a role still renders only if that user holds its data permission (DEC-154).

| Role | Focus | Widgets | Quick actions |
|---|---|---|---|
| **OW** Organization Owner | Whole organization at a glance | `reporting.attention`, `approval.my_pending`, `reporting.branch_comparison` (multi-branch), `fee.today_collection`, `attendance.today_students`, `attendance.today_staff`, `student.strength`, `fee.month_billed_collected`, `fee.outstanding_aging`, `fee.collection_trend`, `attendance.trend_30d`, `result.latest_summary`, `organization.onboarding_checklist` (new organizations) | Approvals, Reports, Send announcement |
| **OA** Organization Admin | Setup health and operations across branches | `organization.onboarding_checklist`, `reporting.attention`, `approval.my_pending`, `reporting.branch_comparison`, `student.strength`, `attendance.today_students`, `fee.today_collection`, `notification.delivery_health`, `import_export.recent_jobs`, `identity.account_alerts` | Add user, Import data, Settings |
| **PR** Principal | Today in my branch | `reporting.attention`, `approval.my_pending`, `attendance.today_students`, `attendance.today_staff`, `fee.today_collection`, `student.strength`, `attendance.unmarked_sections`, `examination.marks_progress`, `leave.staff_on_leave_today`, `attendance.absent_students_today`, `fee.outstanding_aging`, `attendance.trend_30d`, `result.latest_summary`, `administration.upcoming` | Approvals, Send announcement, Reports |
| **VP** Vice Principal | Discipline of the day: attendance, staff, timetable | `reporting.attention`, `approval.my_pending`, `attendance.unmarked_sections`, `attendance.today_students`, `attendance.today_staff`, `leave.staff_on_leave_today`, `timetable.substitutions_today` (R2), `communication.diary_to_review`, `discipline.recent_incidents` (R2), `administration.upcoming` | Approvals, Mark substitution (R2), Announcement |
| **BM** Branch Manager | Front office, admissions, fees, operations | `reporting.attention`, `approval.my_pending`, `admission.pipeline`, `admission.seats`, `fee.today_collection`, `fee.vouchers_due_week`, `student.pending_documents`, `attendance.today_staff`, `transport.capacity` (R2), `inventory.low_stock` (R3) | New admission, Print ID cards, Reports |
| **HOD** Head of Department | Department teaching and marks | `reporting.attention`, `approval.my_pending`, `timetable.my_today`, `examination.reviews_pending`, `examination.marks_progress` (department), `result.latest_summary` (department), `assessment.to_grade` (R2), `attendance.my_checkin` | Review marks, My classes, Question bank (R2) |
| **AC** Academic Coordinator | Classes, timetable, diary and marks across the branch | `reporting.attention`, `approval.my_pending`, `attendance.unmarked_sections`, `examination.marks_progress`, `communication.diary_to_review`, `timetable.substitutions_today` (R2), `learning.course_progress` (R2), `attendance.today_students` | Timetable, Diary review, Teaching assignments |
| **CT** Class Teacher | My class today | `attendance.my_sections_to_mark`, `reporting.attention`, `approval.my_pending`, `timetable.my_today`, `attendance.absent_students_today` (my section), `communication.my_diary_today`, `examination.my_marks_due`, `homework.to_review` (R2), `attendance.my_checkin`, `leave.my_balance`, `communication.latest_announcements` | Mark attendance, Write diary, Enter marks |
| **ST** Subject Teacher | My periods and marks | `timetable.my_today`, `reporting.attention`, `examination.my_marks_due`, `communication.my_diary_today`, `homework.to_review` (R2), `assessment.to_grade` (R2), `attendance.my_checkin`, `leave.my_balance`, `communication.latest_announcements` | Enter marks, Write diary, Take period attendance (where enabled) |
| **ACT** Accountant | Money in, money owed, books in order | `reporting.attention`, `approval.my_pending`, `fee.today_collection`, `fee.month_billed_collected`, `fee.cheques_to_clear`, `accounting.cash_bank`, `fee.outstanding_aging`, `fee.top_defaulters`, `fee.collection_trend`, `accounting.journals_to_approve`, `payroll.run_status` (R2) | Generate vouchers, Record payment, Defaulters |
| **CSH** Cashier | My counter | `fee.my_cashier_session`, `fee.my_receipts_today`, `reporting.attention` (own items), `attendance.my_checkin` | Record payment, Open or close session, Print voucher |
| **HRM** HR Manager | Staff presence and records | `reporting.attention`, `approval.my_pending`, `attendance.today_staff`, `leave.staff_on_leave_today`, `hr.headcount`, `hr.expiring_documents`, `attendance.device_health`, `payroll.run_status` (R2) | Add employee, Staff attendance, Leave calendar |
| **LBR** Librarian (R3) | Circulation | `library.overdue`, `reporting.attention`, `attendance.my_checkin` | Issue or return, Add book, Overdue list |
| **EXC** Examination Controller | Exam cycle | `reporting.attention`, `approval.my_pending`, `examination.upcoming`, `examination.marks_progress`, `examination.reviews_pending`, `result.latest_summary`, `external_exam.deadlines` (R2) | Date sheet, Process results, Print admit cards |
| **ADO** Admission Officer | Applications and new students | `admission.pipeline`, `admission.seats`, `reporting.attention`, `student.pending_documents`, `approval.my_pending`, `student.strength_by_class` | New application, Applications, Verify documents |
| **REC** Receptionist | Front desk | `reporting.attention` (front desk items), `admission.pipeline` (inquiries), `attendance.absent_students_today` (contact view only), `administration.upcoming`, `attendance.my_checkin` | New inquiry, Issue gate pass (R2), Log visitor (R3) |
| **TM** Transport Manager (R2) | Routes and capacity | `transport.capacity`, `transport.trips_today` (R3), `reporting.attention`, `attendance.my_checkin` | Allocate student, Route sheets, Vehicles (R3) |
| **TS** Transport Staff (R3) | My route today | `transport.trips_today` (own route), `attendance.my_checkin` | Mark boarding |
| **ITA** IT Admin | System health in the organization | `reporting.attention`, `attendance.device_health`, `notification.delivery_health`, `identity.account_alerts`, `import_export.recent_jobs` | Users, Devices, Imports |
| **SUP** Support Staff | Self-service only | `attendance.my_checkin`, `leave.my_balance`, `communication.latest_announcements`, `administration.upcoming` | Check in, Apply for leave |
| **GDN** Guardian | One child at a time | `guardian.pending_confirmations`, `guardian.child_today`, `guardian.child_fee_due`, `guardian.child_diary`, `guardian.child_homework_due` (R2), `guardian.child_upcoming_exams`, `guardian.child_latest_result`, `guardian.child_transport` (R2), `guardian.announcements` | Pay fee (R2), Apply for leave, Message teacher (R2) |
| **STD** Student | My day | `student.next_period`, `student.due_soon`, `student.continue_learning` (R2), `student.my_attendance_month`, `student.latest_result`, `student.announcements` | Timetable, Homework (R2), Quizzes (R2) |
| **SA** Platform Super Admin | Tenants and system health | `platform.system_health`, `platform.tenants_by_state`, `platform.new_organizations`, `platform.usage_limits`, `platform.support_sessions`, `platform.webhook_failures` | Provision organization, Organizations, Announcements |
| Platform support | Tenant help | `platform.support_sessions`, `platform.tenants_by_state`, `platform.system_health`, `platform.webhook_failures` | Request support access, Organizations |
| Platform billing | Plans and usage | `platform.usage_limits`, `platform.tenants_by_state`, `platform.new_organizations` | Plans, Usage export |
| Custom roles | Built from permissions | Every widget whose data permission the role holds, ordered by the precedence of the nearest template the role was cloned from; roles built from scratch use the generic order in 10.5 | First three permitted quick actions from the template |

---

## 10.9 Dashboard Wireframes

### Organization Owner, multi-branch (desktop)

```
+--------------------------------------------------------------------------------------------------+
| Good morning, Asif                 All branches v  .  Session 2026-27  .  Tue 24 Nov 2026        |
| As of 10:45 AM                                      [ Approvals (4) ] [ Reports ] [ Announcement ]|
+------------------------------------------+-------------------------------------------------------+
| NEEDS ATTENTION                          | WAITING FOR MY APPROVAL                               |
| (!) 3 sections not marked (2 branches) > | Fee concession, Bilal Hussain, 30%     Gulberg   2h > |
| (!) 1 attendance device offline        > | Refund Rs 12,000, Sara Ahmed           DHA       5h > |
|     2 journals awaiting approval       > | Result publication, Class 9 Term 1     Gulberg   1d > |
|     5 cheques to clear                 > | Payment reversal R-2026-004512         Johar     1d > |
+------------------------------------------+-------------------------------------------------------+
| BRANCHES TODAY                                                                                    |
| Branch       Students  Attendance   Staff present  Collection today  Outstanding   Approvals    |
| Gulberg         1,240      93.1%        58 / 61       Rs 412,500      Rs 2.84M          2      > |
| DHA               980      91.4%        44 / 47       Rs 288,000      Rs 1.92M          1      > |
| Johar Town        760      88.7%  (!)   33 / 38       Rs 140,250      Rs 1.31M          1      > |
| Total           2,980      91.5%       135 / 146      Rs 840,750      Rs 6.07M          4        |
+----------------------+----------------------+---------------------------+------------------------+
| Collection today     | Student attendance   | Staff attendance          | Students               |
| Rs 840,750           | 91.5%                | 135 of 146 present        | 2,980 active           |
| 312 receipts         | 254 absent           | 7 late . 4 on leave       | +46 admissions . -9    |
+----------------------+----------------------+---------------------------+------------------------+
| NOVEMBER: BILLED AND COLLECTED            | OUTSTANDING BY AGE                                    |
| Billed     Rs 9.62M  ################     | 0-30 days   Rs 3.10M  ##########                       |
| Collected  Rs 6.71M  ###########  70%     | 31-60 days  Rs 1.64M  #####                            |
|                                           | 61-90 days  Rs 0.79M  ##                               |
|                                           | 90+ days    Rs 0.54M  ##               [View table]    |
+-------------------------------------------+-------------------------------------------------------+
| COLLECTION, LAST 12 MONTHS  (line)        | ATTENDANCE, LAST 30 DAYS  (line, one per branch)      |
+-------------------------------------------+-------------------------------------------------------+
```

### Principal (desktop)

```
+--------------------------------------------------------------------------------------------------+
| Gulberg Branch  .  Session 2026-27  .  Tue 24 Nov 2026          [ Approvals (2) ] [ Announcement ]|
+------------------------------------------+-------------------------------------------------------+
| NEEDS ATTENTION                          | WAITING FOR MY APPROVAL                               |
| (!) 2 sections not marked: 7-B, 10-C   > | Attendance correction, 8-A, 21 Nov          2h  >     |
| (!) Marks: Term 1 closes in 2 days,    > | Leave, Nadia Iqbal (teacher), 3 days        4h  >     |
|     6 of 48 papers not submitted         |                                                       |
+----------------------+-------------------+---------------+----------------------------------------+
| Student attendance   | Staff attendance  | Collection    | Students                               |
| 93.1%  (84 absent)   | 58 / 61 present   | Rs 412,500    | 1,240 active                           |
+----------------------+-------------------+---------------+----------------------------------------+
| SECTIONS NOT MARKED            | MARKS SUBMISSION, TERM 1            | STAFF ON LEAVE TODAY            |
| 7-B  Ms Hina     window closed | Submitted   38  ##############      | Nadia Iqbal   Casual            |
| 10-C Mr Kamran   window closed | Approved    30  ###########         | Usman Ali     Sick              |
|                                | Pending      6  ##                  |                                 |
|                                | Returned     4  #                   |                                 |
+--------------------------------+-------------------------------------+---------------------------------+
| ABSENT TODAY (84)                         | OUTSTANDING BY AGE           | HOLIDAYS AND EVENTS      |
| 9-A  Ayesha Khan   3rd day in a row  (!)  | ...                          | 25 Dec  Quaid-e-Azam Day |
| ...                              View all |                              | 28 Nov  PTM (Classes 6-8)|
+-------------------------------------------+------------------------------+--------------------------+
```

### Class Teacher (mobile, 360px)

```
+------------------------------------+
| Good morning, Hina        [bell 2] |
| Gulberg . Tue 24 Nov               |
+------------------------------------+
| MARK ATTENDANCE                    |
| 9-A  .  38 students                |
| Window closes 10:00 AM             |
| [ Mark 9-A now                   ] |  primary
+------------------------------------+
| NEEDS ATTENTION                    |
| Term 1 marks: 2 papers to enter  > |
| 1 leave request from a guardian  > |
+------------------------------------+
| MY CLASSES TODAY                   |
| 8:00   9-A  English   Room 12      |
| 9:40   10-B English   Room 14      |
| 11:20  9-A  Library                |
+------------------------------------+
| TODAY'S DIARY                      |
| 9-A: not written yet               |
| [ Write diary ]                    |
+------------------------------------+
| My attendance: In at 7:41 AM (v)   |
| Leave: 6 casual, 8 sick left       |
+------------------------------------+
| [Home] [Attend.] [Marks] [Diary] [More] |  staff mobile bottom shortcuts for teacher roles
+------------------------------------+
```

Teacher roles (CT, ST) get a bottom shortcut bar on screens below 768px in the staff portal, built from their top four quick actions; other staff roles use the drawer only.

### Subject Teacher (desktop)

```
+--------------------------------------------------------------------------------------------------+
| Good morning, Kamran      Gulberg . Tue 24 Nov 2026       [ Enter marks ] [ Write diary ]        |
+------------------------------------------+-------------------------------------------------------+
| MY CLASSES TODAY                         | MY MARKS TO ENTER                                     |
| 8:00  10-C Physics   Room 21    now      | Term 1, Physics 10-C   Theory    due 26 Nov   (!)  >  |
| 9:40  9-B  Physics   Lab 2               | Term 1, Physics 9-B    Practical due 26 Nov        >  |
| 12:10 O2  Physics 5054 (group P1)        | Mock, Physics 5054 O2  Paper 2   due 2 Dec         >  |
+------------------------------------------+-------------------------------------------------------+
| NEEDS ATTENTION             | TODAY'S DIARY            | My attendance  | Leave balance           |
| 1 marks paper returned  >   | 10-C written (v)         | In 7:52 AM (v) | 6 casual . 8 sick       |
|   by HOD                    | 9-B  not written         |                |                         |
+-----------------------------+--------------------------+----------------+-------------------------+
| ANNOUNCEMENTS                                                                                     |
+--------------------------------------------------------------------------------------------------+
```

### Accountant (desktop)

```
+--------------------------------------------------------------------------------------------------+
| Gulberg Branch . Nov 2026 . As of 10:45 AM   [ Generate vouchers ] [ Record payment ] [Defaulters]|
+------------------------------------------+-------------------------------------------------------+
| NEEDS ATTENTION                          | WAITING FOR MY APPROVAL                               |
| 5 cheques due to clear                 > | Cash variance, counter 2, -Rs 500, 23 Nov        >   |
| 1 cashier session open since 23 Nov (!)> |                                                       |
| December vouchers not generated        > |                                                       |
+----------------------+-------------------+-----------------------+-------------------------------+
| Collection today     | November          | Cash and bank         | Journals to approve           |
| Rs 412,500           | Billed Rs 3.98M   | Cash  Rs 186,400      | 2                             |
| 128 receipts         | Coll.  Rs 2.79M   | HBL   Rs 4,120,000    |                               |
|                      | 70.1%             | MCB   Rs 1,045,300    |                               |
+----------------------+-------------------+-----------------------+-------------------------------+
| OUTSTANDING BY AGE                        | LARGEST OUTSTANDING                                   |
| ...                                       | Family F-0042 (3 children)   Rs 96,000   4 months  > |
|                                           | Bilal Hussain 9-A            Rs 38,500   3 months  > |
+-------------------------------------------+-------------------------------------------------------+
| COLLECTION, LAST 12 MONTHS                                                                        |
+--------------------------------------------------------------------------------------------------+
```

### Cashier (desktop)

```
+--------------------------------------------------------------------------------------------------+
| Counter 2 . Gulberg . Tue 24 Nov 2026                  [ Record payment ] [ Close session ]       |
+------------------------------------------+-------------------------------------------------------+
| MY CASH COUNTER                          | MY RECEIPTS TODAY (41)                                |
| Session opened 8:02 AM                   | R-2026-004588  Ayesha Khan 9-A   Cash    Rs 3,500   |
| Opening float      Rs 5,000              | R-2026-004587  Family F-0042     Cheque  Rs 24,000  |
| Cash received      Rs 131,500            | R-2026-004586  Bilal Hussain 9-A Cash    Rs 8,500   |
| Cheques (3)        Rs 41,000             | ...                                        View all |
| Expected cash      Rs 136,500            |                                                       |
+------------------------------------------+-------------------------------------------------------+
| NEEDS ATTENTION: none. All clear.        | My attendance: In 7:55 AM (v)                         |
+------------------------------------------+-------------------------------------------------------+
```

When no session is open, the counter widget shows "No session open" with an "Open session" primary button, and "Record payment" is disabled with the reason "Open a cash session first" (DEC-147).

### Examination Controller (desktop)

```
+--------------------------------------------------------------------------------------------------+
| Gulberg . Term 1 2026-27                        [ Date sheet ] [ Process results ] [ Admit cards ]|
+------------------------------------------+-------------------------------------------------------+
| NEEDS ATTENTION                          | UPCOMING EXAMS                                        |
| (!) 6 papers not submitted, closes 26  > | Term 1 written: 1 Dec to 12 Dec (date sheet draft)   |
| 4 submissions to review                > | Mock O Level: 8 Jan (published)                       |
| Results for Class 9 ready to publish   > | June 2027 entries close 21 Feb (R2)                   |
+------------------------------------------+-------------------------------------------------------+
| MARKS SUBMISSION BY CLASS                 | LATEST RESULTS                                       |
| Class 6   ########## 100%                 | Class 10 Mid-term: pass 91.2%, A+ 18%                |
| Class 7   ########   82%                  | Class 9  Mid-term: pass 87.5%, A+ 12%                |
| Class 9   ######     64%  (!)            |                                                       |
+-------------------------------------------+-------------------------------------------------------+
```

### HR Manager (desktop)

```
+--------------------------------------------------------------------------------------------------+
| All branches v . Tue 24 Nov 2026                 [ Add employee ] [ Staff attendance ] [ Leave ]  |
+------------------------------------------+-------------------------------------------------------+
| NEEDS ATTENTION                          | WAITING FOR MY APPROVAL                               |
| 3 missing punch requests               > | Missing punch, Usman Ali, 22 Nov            >         |
| 2 CNICs expire within 14 days          > | Profile correction, Hina Raza               >         |
| (!) Device "Gate 2" offline since 9:10 > |                                                       |
+----------------------+-------------------+-----------------------+-------------------------------+
| Staff today          | On leave today    | Staff                 | Devices                       |
| 135 / 146 present    | 4                 | 146 active            | 5 online . 1 offline (!)      |
| 7 late               |                   | 98 teaching, 48 other |                               |
+----------------------+-------------------+-----------------------+-------------------------------+
| DOCUMENTS EXPIRING IN 30 DAYS                                                                     |
+--------------------------------------------------------------------------------------------------+
```

### Admission Officer (desktop)

```
+--------------------------------------------------------------------------------------------------+
| Gulberg . Admissions 2026-27                 [ New application ] [ Applications ] [ Documents ]  |
+------------------------------------------+-------------------------------------------------------+
| ADMISSIONS THIS SESSION                  | SEATS AVAILABLE                                       |
| Submitted       42                       | Playgroup   4 of 30                                   |
| Test scheduled  11                       | Class 1     0 of 40  (full)                           |
| Approved        18                       | Class 6     7 of 40                                   |
| Fee pending      9                       | Class 9     2 of 45                                   |
| Admitted        46                       | O1          5 of 25                                   |
+------------------------------------------+-------------------------------------------------------+
| NEEDS ATTENTION                          | DOCUMENTS TO VERIFY (12)                              |
| 7 applications with no action for 3d   > |                                                       |
+------------------------------------------+-------------------------------------------------------+
```

### Receptionist (mobile or desktop)

```
+------------------------------------------------+
| Front desk . Gulberg . Tue 24 Nov              |
| [ New inquiry ] [ Gate pass (R2) ] [ Visitor (R3) ] |
+------------------------------------------------+
| NEEDS ATTENTION                                |
| 3 inquiries to call back today               > |
+------------------------------------------------+
| ABSENT TODAY: call list (84)                   |
| 9-A Ayesha Khan     Imran Khan   0300 1234567  |
| ...                                            |
+------------------------------------------------+
| HOLIDAYS AND EVENTS                            |
+------------------------------------------------+
```

The absent call list shows guardian phone numbers only when the receptionist holds `guardian.contact.view` (D5 5.10); otherwise it shows names and classes only.

### Guardian (mobile, 360px)

```
+------------------------------------+
| [Logo] City School       [bell 2]  |
| ( Ayesha Khan, 9-A        v )      |
+------------------------------------+
| NEEDS YOUR RESPONSE                |
| Confirm June 2027 exam entries   > |  (R2, Cambridge track)
+------------------------------------+
| TODAY                              |
| (v) Present . arrived 7:52 AM      |
| This month: 19 of 20 days present  |
+------------------------------------+
| FEE                                |
| November . due 10 Nov              |
| Rs 3,500 outstanding (!) Partly paid|
| [ Pay now (R2) ]  [ View voucher ] |
+------------------------------------+
| DIARY . TODAY                      |
| Maths: Exercise 4.2, Q1 to 10      |
| English: Read chapter 6            |
+------------------------------------+
| UPCOMING EXAMS                     |
| Term 1 starts Mon 1 Dec            |
| [ Date sheet ]                     |
+------------------------------------+
| LATEST RESULT                      |
| Mid-term: 82.4%, Grade A           |
| [ Report card ]                    |
+------------------------------------+
| ANNOUNCEMENTS (1 new)            > |
+------------------------------------+
| [Home] [Attend.] [Fees] [Results] [More] |
+------------------------------------+
```

### Student (mobile, 360px)

```
+------------------------------------+
| [Logo] City School   [bell] (AK)   |
+------------------------------------+
| Good morning, Ayesha               |
| NEXT CLASS                         |
| Physics . 9:40 AM . Lab 2          |
+------------------------------------+
| DUE SOON                           |
| Chemistry worksheet . tomorrow     |
| Maths quiz 3 . opens 2:00 PM (R2)  |
+------------------------------------+
| CONTINUE LEARNING (R2)             |
| Physics: Motion, lecture 4  60%  > |
+------------------------------------+
| MY ATTENDANCE . NOVEMBER           |
| 19 of 20 days . 95%                |
+------------------------------------+
| LATEST RESULT                      |
| Mid-term: 82.4% . A                |
+------------------------------------+
| [Home] [Timetable] [Homework] [Learn] [More] |
+------------------------------------+
```

### Platform Super Admin (desktop)

```
+--------------------------------------------------------------------------------------------------+
| (PLATFORM) System and tenants                      [ Provision organization ] [ Organizations ]   |
+------------------------------------------+-------------------------------------------------------+
| SYSTEM HEALTH                            | ORGANIZATIONS BY STATE                                |
| Queues   critical 0 . bulk 12 . docs 3   | Active 214 . Trial 31 . Suspended 4 . Archived 9      |
| Failed jobs (24h)  2                   > |                                                       |
| Gotenberg  ok . p95 1.8 s                | NEW ORGANIZATIONS, 30 DAYS: 17                        |
| SMS providers  2 ok . 1 degraded (!)   > |                                                       |
+------------------------------------------+-------------------------------------------------------+
| NEAR PLAN LIMITS                          | ACTIVE SUPPORT SESSIONS                               |
| City School  students 1,940 / 2,000  (!)> | Ali (support) in Al-Noor, until 3:00 PM            >  |
| Al-Noor      SMS 92% of monthly credits > |                                                       |
+-------------------------------------------+-------------------------------------------------------+
| FAILED WEBHOOK PROCESSING (24h): 1 (jazzcash, Al-Noor)                                       >    |
+--------------------------------------------------------------------------------------------------+
```

---

## Assumptions (Part 1)

| ID | Assumption |
|---|---|
| A-79 | Teachers use phones in the staff portal often enough to justify a bottom shortcut bar for teacher roles below 768px. |
| A-80 | Branch comparison is useful mainly to owners and organization admins; principals of one branch never see other branches' figures. |
| A-81 | Attention thresholds (24 hours for approvals, 3 days for admissions, 14 days for documents, 5% message failures) are organization settings with these defaults. |

## Gaps Found and Added (Part 1)

| ID | Gap | Decision |
|---|---|---|
| GAP-199 | Screens had no stable IDs for workflows, prompts and checklists | `SCR-{CODE}-NNN` (DEC-151) |
| GAP-200 | Portal home routes had no owner or name | `{portal}.home` in Reporting (DEC-152) |
| GAP-201 | Users holding several roles had no rule for which dashboard to see | "View as" switcher with precedence (DEC-153) |
| GAP-202 | Dashboard widgets could reveal data the user can't open | Widget data permissions (DEC-154) |
| GAP-203 | Actionable items were scattered across modules | `AttentionItemRegistry` (DEC-155) |
| GAP-204 | Snapshot numbers looked live | "As of" stamp (DEC-156) |
| GAP-205 | A slow widget could block the dashboard | Per-widget loading (DEC-159) |
| GAP-206 | Two names existed for the permission sync command | `identity:sync-permissions` (correction in 10.1) |
| GAP-207 | New organizations would see empty charts | Onboarding checklist widget (10.5) |

---

Reply CONTINUE for Deliverable 10, Part 2
