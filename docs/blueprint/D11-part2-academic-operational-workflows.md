# Deliverable 11: Business Workflows and Approval Processes
# Part 2 of 2: Academic and Operational Workflows

**Version:** 1.0 (draft for approval) | **Date:** 24 Nov 2026
**Conventions:** notation from Part 1, 11.2. Approval types from Part 1, 11.4. Lifecycle states and transitions from D6 6.8 to 6.10 and D6A. Screen IDs from D10.

---

## 11.14 New Decisions

| ID | Decision |
|---|---|
| DEC-179 | **Absence alerts wait 10 minutes.** When a section sheet is submitted, absence SMS to guardians are queued with a 10-minute delay (setting `attendance.absence_alert_delay_minutes`). If the teacher changes a student from `A` within that time, the queued message is cancelled. Alerts never go out for students on approved leave, holidays or suspension. |
| DEC-180 | **Attendance sheet states:** `not_marked`, `draft` (autosaved), `submitted` (teacher done, still editable until the window closes), `finalized` (window closed or principal finalized). After finalization every change is an `attendance_correction`. Unmarked students in a finalized sheet stay `NM`; they never turn into `A` automatically. |
| DEC-181 | **Marks submission states:** `draft`, `submitted`, `returned`, `approved`. Submitted and approved entries are locked; returned entries reopen. After approval, changes go only through `marks_revision`. |
| DEC-182 | **Result batches are versioned.** Processing creates a new version and supersedes the previous unpublished one. A listener marks a processed or approved batch `stale` when any of its marks change (revision approved, late submission approved). A stale batch can't be published; it must be reprocessed, which also invalidates a pending publication request (DEC-168). |
| DEC-183 | **One notification pipeline.** Every message goes event, rule, recipients, preferences, language, quiet hours, deduplication, channel, provider. Duplicates of the same event, recipient, channel and record within 10 minutes are dropped. Quiet hours (default 21:00 to 07:00) hold non-urgent messages until `notifications:release-quiet-hours`; OTPs, emergency closures and security alerts bypass them. SMS stop when the organization's monthly credit cap is reached (in-app and email continue) and admins are alerted at 80% and 100%. |
| DEC-184 | **Session close has a fixed order:** results published, promotions run, conditional promotions given deadlines, open fee items reviewed, attendance and exam periods locked, then the session closed. The close checklist (SCR-ACD-005) enforces it through `SessionCloseCheckRegistry`. |
| DEC-185 | **The school-year calendar is seeded data**, not code. Each organization gets the default yearly and monthly schedule in 11.32 (Punjab national track April to March, Cambridge August to July) as calendar events and reminders it can edit. |

### Correction to D7

| Where | Change |
|---|---|
| D7 Part 2, Attendance | Adds `section_attendance_sheets` [B]: `section_id`, `attendance_date`, `status` (`not_marked`, `draft`, `submitted`, `finalized`), `submitted_by`?, `submitted_at`?, `finalized_by`?, `finalized_at`?, `version`. Key `uq_sas (organization_id, section_id, attendance_date)`. It holds the sheet state (DEC-180) and the sheet version used by the marking grid (DEC-162); rows stay in `student_attendances`. |

---

## 11.15 Admission to Enrollment

```
[Public] Online application (SCR-ADM-010) with mobile OTP          [ADO or REC] Counter entry (SCR-ADM-002)
           \                                                              /
            +----> application `submitted` (duplicate check, FR-ADM-006) <+
                     -> applicant SMS with reference number
                     |
[ADO] Review documents (SCR-ADM-003) -> `under_review`
   <test or interview needed?> yes -> `test_scheduled` / `interview_scheduled` (R2 SCR-ADM-013)
                                        result entered -> back to review
   <seat available and window open?> no -> `waitlisted` (or override_window, SCR-ADM-005)
   |
[PR or ADO] Decide (SCR-ADM-004)
   reject  -> `rejected` ((end)); applicant SMS; anonymized after retention (FR-ADM-015)
   approve -> `approved`; class and section offered; admission voucher issued (FR-ADM-007, JE:fee_voucher)
   |
<admission voucher paid?> (Part 1, 11.6) or override with permission
   |
[ADO or BM] Convert (SCR-ADM-006, ConvertApplicationToStudent) - one transaction (BR-ADM-02):
   student (registration no), guardians (existing CNIC linked, never duplicated, BR-ADM-03), family,
   GR no, admission no, roll no, enrollment `active`, student subjects, guardian user + activation SMS,
   timeline student.admitted, audit
   -> StudentEnrolled -> transport and library prompts, ID card queue, welcome SMS
application `admitted` ((end))
```

| Rule | Detail |
|---|---|
| Duplicate found at entry | "Possible match" panel (W-03): open record, readmit instead (11.21), or confirm a different child with a reason |
| Applicant withdraws | `withdrawn`; unpaid admission voucher cancelled (F4) |
| Approved but not paid in N days (setting, default 7) | Reminder at day 3; at day N the offer lapses, voucher cancelled, seat released, application `withdrawn` with reason `offer_lapsed` |
| Cancel admission within 15 days | `withdrawal` approval with reason `admission_cancelled` (D6); refund per policy (Part 1, 11.8) |
| Cambridge applicant from national track | IBCC equivalence (R2, SCR-EXT-021) can be attached to the application for merit decisions |

---

## 11.16 Daily Student Attendance

```
06:00 [System] Sheets prepared for every section with a working day (Administration CalendarQuery)
             pre-filled: LV (approved leave), H (holiday or closure), SU (suspension), W (weekend)
             device punches for students (gate devices) shown as "arrived 7:52"
   |
[CT] Mark (SCR-ATT-002, MarkSectionAttendance) - per-row autosave (DEC-162), sheet `draft`
[CT] Submit -> `submitted` -> absence alerts queued with delay (DEC-179)
   |
Window close (policy, e.g. 10:00) [System] attendance:finalize-windows (every 15 min)
   submitted and draft sheets -> `finalized` (draft rows already saved count as marked)
   not_marked sheets stay not_marked; NM rows stay NM (DEC-180)
   -> AttendanceFinalized -> today's snapshot; queued alerts released
Every 15 min in school hours [System] attendance:not-marked-alerts -> CT, then PR after 30 min
   |
After finalization: changes only through APR:attendance_correction (SCR-ATT-007)
Month end: [PR] lock attendance period (SCR-CORE-002) -> corrections then need period_unlock
```

| Rule | Detail |
|---|---|
| Source precedence | Correction over teacher or admin over device or QR over automatic (DEC-25). A device punch never overwrites a teacher's mark. |
| Late arrival after marking | A student marked `A` who arrives later is changed to `L` by the teacher before the window closes, or by correction after |
| Half day | `HD` marked by the teacher, or derived from a gate out-punch with a gate pass (R2) |
| Period attendance (colleges) | Separate sheets per timetable period (SCR-ATT-003); the daily status is derived by the policy rule (present if present in at least N periods) |
| Emergency closure | `ClosureDeclared` (SCR-ADN-003) sets `H` for the day in every affected open sheet and cancels queued alerts |
| Teaching groups (Cambridge) | Daily attendance stays per section; period attendance can be per teaching group |
| Absence streak | Three consecutive absences raise an attention item for the class teacher and an SMS template "consecutive absence" to the guardian (setting) |

---

## 11.17 Staff Attendance

```
Device punch (ZKTeco ADMS /iclock or signed webhook) or CSV import (SCR-ATT-020)
   -> webhook_receipts stored first (DEC-42) -> ProcessWebhookReceipt (queue attendance)
   -> attendance_punches (uq_apn drops duplicates; DEC-104 partition)
Online check-in (SCR-ATT-012): trusted device + geofence result recorded with the punch
QR check-in (SCR-ATT-013): rotating kiosk code, single use per person per session
   |
Every 5 min [System] attendance:resolve-punches -> ResolveDailyAttendance per employee and day
   first in, last out (punches inside the 2-minute collapse window merged)
   late if first in > shift start + grace; half day if past half-day threshold;
   early departure if last out < threshold
   only sources in the policy's `official_sources` count
Every 15 min [System] attendance:auto-absent -> after the cut-off, no punch, no leave, no holiday -> A
   |
[Staff] Missing out-punch? -> APR:missing_punch (SCR-ATT-015)
[PR or HRM] Official duty, training -> administrative entry (SCR-ATT-010)
Suspicious punches (outside geofence, impossible times, clock drift > 5 min) -> SCR-ATT-022
   |
Month end: [HRM] lock staff attendance period -> payroll can calculate (Part 1, 11.12)
```

| Rule | Detail |
|---|---|
| Shared staff (D4) | The day's branch is the branch of the first check-in |
| Unmapped punches | Punches from device user ids with no enrollment are held on SCR-ATT-019 until mapped, then resolved |
| Device offline | `attendance:device-health` alerts IT and HR after 30 minutes without heartbeat; punches stored on the device upload when it reconnects and resolve late |
| Late conversion | Policy `lates_per_leave` (for example 3 lates equal 1 casual leave) is applied by the monthly leave accrual job with an adjustment row and a notice to the employee |

---

## 11.18 Leave

### Day counting (`LeaveDayCalculator`)

| Rule | Default |
|---|---|
| Holidays and weekends inside the range | Not counted |
| Sandwich rule | When leave is taken on both sides of a holiday or weekend, the days in between count (setting, default on for staff, off for students) |
| Half day | Counts 0.5; allowed for staff casual leave and student leave |
| Overlap | A new application overlapping an approved or pending one is refused |
| Balance | Staff leave needs enough balance at submit and again at apply (Part 1, 11.4); negative balance only with HRM override |
| Back-dated leave | Allowed up to N days (setting, default 3); finalized attendance days become corrections when approved |

### Flow summary

| Applicant | Route | Approval | Effect |
|---|---|---|---|
| Guardian for child | SCR-LVE-011 | `student_leave` | Future days `LV`; guardian SMS on decision |
| Class teacher or branch manager for a student | SCR-LVE-002 | `student_leave` (step 1 skipped when the class teacher is the requester) | Same |
| Staff for self | SCR-LVE-004 | `staff_leave` | Balance deducted; days `LV`; substitution suggested for teaching periods (R2) |
| HRM or principal for staff | SCR-LVE-002 | `staff_leave` | Same |
| Cancellation of approved future leave | SCR-LVE-003 | None if not started; `staff_leave` if partly taken | Balance restored for untaken days; `LV` rows removed for future days |

Monthly on day 1: `leave:accrue` adds entitlements per policy; carry-forward and lapse run on the policy's year end.

---

## 11.19 Examination Cycle

```
[EXC] Exam term (SCR-EXM-001) -> exams per class (SCR-EXM-002) -> papers and components (SCR-EXM-003)
[EXC] Date sheet (SCR-EXM-004): conflicts checked (teaching groups, rooms) -> publish (SCR-EXM-005)
   -> DateSheetPublished -> guardians and students notified
[EXC or BM] Admit cards (SCR-EXM-007): blocked when exam.fee_clearance_required and dues exist,
   override by permission with reason -> print (SCR-EXM-008); portal download
   |
Exam days -> [ST or CT] Marks entry (SCR-EXM-010, DEC-164) per section or teaching group
   draft -> submit (SCR-EXM-011) -> `submitted`, cells locked
[HOD or EXC] Review (SCR-EXM-013) (SOD-04: not the entering teacher)
   approve -> `approved`        return -> `returned` with comment, cells reopen -> resubmit
   |
[EXC] Completion tracker (SCR-EXM-017): reminders; "Needs attention" when 3 days remain
   |
After approval, a change -> APR:marks_revision (SCR-EXM-015) -> revision row -> batches stale (DEC-182)
Exam period lock: [PR] locks examination period after results publish
```

| Rule | Detail |
|---|---|
| Codes | `ABS` counts as zero and follows `result_rules.absent_policy`; `MED` excludes the paper from totals if the rule allows; `EXM` exempts |
| Grade-only papers | Entry by grade from the scheme's bands (FR-RES-016) |
| Teaching groups | One submission per teaching group; students from several sections appear together |
| Late submission after the deadline | Allowed; flagged late on the tracker and in the audit |
| Import (SCR-EXM-014) | Same validation as the grid; imported sheets are `draft` until the teacher or EXC submits |

---

## 11.20 Result Processing and Publication

```
[EXC] Process (SCR-RES-005): class, scope (exam, term, cumulative annual), rule, scheme
   check: all papers approved (or EXC override listing missing papers)
   -> bulk job ProcessResultChunk -> result batch version N `processed`
      per student: totals, percentage, grade, GPA, pass or fail per subject, grace marks (rule),
      compartment count, status, positions (ranking rule and tie-breakers), attendance %
[CT and PR] Report card remarks (SCR-RES-013)
[PR] Withhold where needed (SCR-RES-008): fee dues, discipline, missing documents
   |
[EXC] Request publication (SCR-RES-009) -> APR:result_publication
[PR] Approve -> batch `approved`
[EXC] Publish now or at a scheduled time (SCR-RES-010); results:publish-scheduled every 5 min
   -> batch `published`; ResultPublished -> guardians notified (SMS with grade and portal link);
      withheld students see "Contact the school"; report cards available in portals
   |
Unpublish: [PR] request (SCR-RES-011, level 3, reason) -> `unpublished`; portals hide; guardians not re-notified
Recheck (R2): APR:result_recheck -> marks_revision -> batch stale -> reprocess -> publish new version
```

| Rule | Detail |
|---|---|
| Stale batches | DEC-182; publication button disabled with reason (W-12) |
| Cumulative annual result | Uses term weightages from the rule; terms must have approved batches |
| Supplementary exam (conditional promotion) | A supplementary exam for compartment subjects produces a supplementary batch; passing clears the condition (11.21) |
| Cambridge internal results | Grade-based schemes (`cambridge_o_level`, `cambridge_as`, `cambridge_a_level`), report card shows predicted grades (FR-RES-015) |
| Positions visible to portals | Only when `result_rules.ranking_visible_to_portal` is on |

---

## 11.21 Promotion, Conditional Promotion and Session Close

### Promotion (D6 6.10)

```
[PR or AC] Promotion wizard (SCR-STU-010)
  1 select class -> 2 eligibility (Result PromotionEligibilityProvider) -> 3 decide per student
  (promote, conditional, retain, graduate, progress, hold) -> 4 target sections -> 5 checks
  -> 6 preview -> 7 run (PromoteStudentsChunk, 100 per chunk, one transaction each, level 3 confirm)
  per student: old enrollment `completed` with outcome; new enrollment `active` in the next session;
  roll numbers regenerated (setting); electives carried forward; timeline event
  -> StudentsPromoted -> guardians notified of the new class
```

### Conditional promotion

```
promoted_conditionally -> enrollment.is_conditional, condition_deadline (setting, default 60 days)
Supplementary exam result published:
   pass  -> condition cleared; timeline
   fail  -> [PR] revert to retained (approval required, D6) -> enrollment moved back to the previous level
Deadline passed with no result -> attention item for the principal
```

### Session close (DEC-184)

| Step | Check (SessionCloseCheckRegistry) | Owner |
|---|---|---|
| 1 | Annual result batches published for every class | Result |
| 2 | Every active enrollment has an outcome (promotion run complete) | Student |
| 3 | Conditional promotions have deadlines | Student |
| 4 | Open vouchers of the closing session reviewed (they stay payable; arrears carry on later vouchers) | Fee |
| 5 | Attendance and examination periods locked through the last month | Core |
| 6 | Pending approvals for the session resolved or withdrawn | Approval |
| 7 | Close: session `closed`; its records read-only (D9 9.16 banner) | Academic |

### New session (SCR-ACD-003)

Copy classes, sections, class subjects, fee structures (with increase), bell schedules and timetable as draft; session created in `planning`; activation (SCR-ACD-004) when the old session is closing or closed for that institution and track (DEC-79). Promotions can target a `planning` session.

---

## 11.22 Branch Transfer

```
[PR or BM of source] Request (SCR-STU-012): target branch, section, effective date, fee policy
   -> APR:branch_transfer (1 source PR, 2 target PR)
apply: TransferStudentBranch (one transaction)
   source enrollment `closed` (branch_transferred); target enrollment `active`; new GR no in target;
   students.current_branch_id updated (DEC-49); transport allocation ended; library returns flagged;
   teaching group memberships ended; fee balances per policy (Part 1, 11.9)
   -> StudentTransferred -> guardians notified; both principals informed
Source branch keeps read-only "transferred out" view of its own rows (D4 GAP-84)
```

---

## 11.23 Withdrawal, Leaving Certificate and Cancel Admission

```
[BM or ADO] Request withdrawal (SCR-STU-014): reason, last date
   clearance checks shown live (ClearanceCheckRegistry): fee dues, library loans, transport,
   documents, assets issued; each passes, or waiver requested with reason
   -> APR:withdrawal (PR)
apply: WithdrawStudent
   student `withdrawn`; enrollment `closed` (withdrawn); attendance no longer expected from the day after;
   future vouchers stopped; transport and library ended; guardian portal read-only for the retention
   period (A-38); deposit refund draft created (Part 1, 11.8)
   -> StudentWithdrawn -> listeners in Fee, Transport, Library, Communication
[BM or ADO] Issue leaving certificate (SCR-STU-017): clearance re-checked; serial number;
   issued_documents row; QR verification (R2)
```

Cancel admission within 15 days follows the same flow with reason `admission_cancelled`; the enrollment becomes `cancelled` if no dependent records exist (D6).

---

## 11.24 Suspension, Struck Off and Readmission

| Flow | Steps |
|---|---|
| Suspension | From Discipline (SCR-DSC-004) or Student; `student_suspensions` row with dates; status `suspended`; attendance `SU`; guardian notified; reinstated automatically on the end date (daily job) or manually |
| Struck off | Day 1 monthly: `fee:struck-off-candidates` lists students unpaid for N months; [PR] reviews (SCR-STU-015) and strikes off with reason; status `struck_off`, enrollment `closed`; vouchers stop; guardian notified |
| Readmission after struck off (same session) | Dues settled or installment plan approved; seat available; readmission voucher; new enrollment `active`; same registration no |
| Readmission after withdrawal | New application flagged as readmission (W-03 "Readmit instead"); same registration no; new admission no if after session end |

---

## 11.25 Certificates and Document Verification

```
[BM or ADO] Issue (SCR-STU-017 or module print action)
   -> Document module: template by type and institution, data provider fills fields,
      Gotenberg renders PDF (DEC-31), issued_documents row with serial and verification code
   -> DocumentIssued (audit)
Reissue (SCR-DOC-005, PR or BM): new serial, old marked superseded, reason
Revoke (OW or PR): status revoked; verification page shows "Revoked"
Verification (R2, SCR-DOC-006): code or QR -> type, holder name, issue date, valid or revoked
```

Bulk documents (vouchers, report cards, admit cards, ID cards) render as a bulk job on the `documents` queue and merge into one PDF per class.

---

## 11.26 Period Locks

```
unlocked --[holder] lock (SCR-CORE-002)--> locked
   lock checks: attendance (no draft sheets), staff attendance (no unresolved punches),
   examination (no unapproved submissions), fee and accounting (no unposted items)
locked --APR:period_unlock (SCR-CORE-003)--> unlocked_until(relock_due_at)
unlocked_until --[System] relock at relock_due_at, or [holder] relock early--> locked
```

Writes into a locked period fail with `core.period_locked` (FR-GEN-015). Fee and accounting periods lock monthly after the cashier sessions and cheques of the month are settled; fiscal year close (SCR-ACC-002) requires all its months locked.

---

## 11.27 Notification Pipeline (DEC-183)

```
Domain event after commit (for example PaymentRecorded)
  -> NotificationEventRegistry: event to template key and recipient resolvers
  -> notification_rules: on or off, channels, recipient groups (organization settings)
  -> resolve recipients (guardians_of_student, class_teacher_of_section, ...)
  -> preferences (mandatory events ignore opt-out) and language (en or ur template)
  -> quiet hours hold unless urgent
  -> dedupe (event, recipient, channel, record, 10 minutes)
  -> notification_dispatches row -> queue notifications
  -> provider in priority order (SmsGateway, email, WhatsApp R2, in-app)
       failure -> retry 1, 5, 15 minutes -> next provider -> failed (delivery log)
  -> delivery receipts (sms_dlr webhook) update status and cost
```

| Rule | Detail |
|---|---|
| Urdu SMS | Unicode, 70 characters per segment; cost estimate shown before broadcasts (SCR-NTF-009) |
| Credit cap | Plan's monthly SMS credits; alerts at 80% and 100%; urgent SMS (OTP) always allowed and billed |
| Guardian with two children | One message per event per child; fee reminders for a family voucher go once to the family's primary guardian |
| Staff and guardian in one person | Messages addressed by role reach the same user once per channel |

---

## 11.28 Imports

```
[User] New import (SCR-IMX-002): type (permission per type, D5 5.7.23)
  download template -> upload (virus scan) -> map columns (saved mappings)
  -> validate (job imports): row rules, duplicates against the database and inside the file
  -> preview: valid, errors (download error file), possible duplicates
  -> commit valid rows: each row through the same Action as the form, in chunks (one transaction per chunk)
  -> ImportCompleted -> in-app notification with counts and the result file
```

Imports never bypass business rules, never update existing records unless the type declares an update mode (for example subject list), and every created record is audited with the import job id.

---

## 11.29 External Exams (R2, D6A A.9)

```
[EXC] Series (SCR-EXT-007/008): deadlines; fee schedule (SCR-EXT-009)
[EXC or AC] Generate draft entries (SCR-EXT-011) from student subjects and target qualifications (DEC-86)
[EXC or AC] Review (SCR-EXT-012): components, retakes, private changes
[GDN] Confirm entries (SCR-EXT-024) if the organization requires guardian confirmation
[EXC] Confirm (SCR-EXT-013) -> entries `confirmed`; entry fee vouchers (pass-through, F1f)
   payment before the stage deadline -> `paid`; late stage applies the late fee schedule
[EXC] Export (SCR-EXT-014) -> submitted in the awarding body portal -> mark `submitted`
   remittance to the awarding body (F17)
Withdrawal: APR:external_entry_withdrawal (refund by awarding body rules)
Results day: [EXC] import (SCR-EXT-016) -> review -> publish (SCR-EXT-017) -> portals; analytics
Enquiries about results (SCR-EXT-019): request, fee, outcome; certificates register (SCR-EXT-020)
```

National boards (FR-EXT-016) follow the same shape: registration export, roll numbers, results entry.

---

## 11.30 Timetable, Communication, Learning and Operations

### Timetable

| Flow | Steps |
|---|---|
| Build | [AC] new version (draft) or copy active; place periods (SCR-TTB-003); conflicts must be zero |
| Activate | [AC or PR] activate with effective date (SCR-TTB-004); previous version archived; teachers notified; period attendance sheets follow the new version from that date |
| Substitution (R2) | Staff leave approved or absent by 08:00 -> SCR-TTB-009 lists affected periods and free teachers by load -> [AC or VP] assign -> substitutes notified; substitution shown in "My timetable" |

### Communication

| Flow | Steps |
|---|---|
| Announcement | [MGT or AC] compose (SCR-COMM-002), audience, schedule -> publish (permission `publish`) -> in-app to audience, SMS or WhatsApp copy if chosen -> acknowledgement tracking -> reminder to unread after 24 hours (setting) |
| Diary | [TCH] write (SCR-COMM-005) -> if diary review is on: [AC or VP] approve (SCR-COMM-006) -> published to guardians and students; same-day edits show "edited" |
| Messaging (R2) | Guardian and teacher thread per student; office hours notice; messages outside hours delivered next morning; oversight read-only for PR and VP; report or hide by moderator |
| Parent-teacher meeting (R2) | [PR or VP] event -> [TCH] slots -> [GDN] book -> reminders day before -> [TCH] mark attended |

### Homework and quizzes (R2)

| Flow | Steps |
|---|---|
| Homework | [TCH] assign (SCR-HWK-002) -> diary line -> [STD or GDN] submit (attempts kept, DEC-119) -> [TCH] review: accept with marks or ask to resubmit -> report |
| Quiz | [ST] build from bank (questions snapshotted at publish, DEC-118) -> publish to targets -> [STD] attempt with server timer, one open attempt -> auto-submit at deadline (`assessment:auto-submit`) -> auto-grade objective, [ST] grade subjective -> release results -> if exam-linked, chosen score written to marks through Examination |

### Library (R3)

```
[LBR] Issue: scan member card -> limits, overdue and fines checked -> scan copy (uq_lln_open) -> due date
[LBR] Return: scan copy -> condition -> overdue fine calculated (setting per day, cap) -> library_fines `pending`
Fines billing: students -> next fee voucher line (VoucherLineContributor, DEC-124); staff -> payslip deduction
Reservations: queue per title; copy returned -> first in queue notified, held 3 days
Clearance: open loans or unpaid fines block withdrawal clearance (ClearanceCheck)
```

### Transport (R2, R3)

```
[TM] Allocate student (SCR-TRN-005): route, stop, direction, start date; capacity check under row lock
   -> monthly fee from the stop added to vouchers (VoucherLineContributor, D7 7.42)
Change of stop: allocation ended and a new one started from the date; fee follows from the next voucher
Withdrawal or transfer: allocation ended by listener
R3 boarding: [TS] marks boarded or not by stop (SCR-TRN-013) -> guardian notified on drop; vehicle,
   driver and document expiries -> attention items 30 days ahead
```

### Inventory (R3)

```
Purchase bill approved (R2 accounting) -> [BM] receive stock (SCR-INV-005) -> stock movement + balance
[BM] Issue to a person or section -> negative stock blocked (DEC-120)
Transfer between locations -> paired movements
Count difference -> APR:stock_adjustment -> adjustment movement
Assets: tag on receipt -> assign or return (condition) -> maintenance requests -> disposal with approval
```

### Discipline (R2, R3)

```
[TCH or VP] Record incident (SCR-DSC-002) -> guardian notice if the category requires it
[VP or PR] Take action (SCR-DSC-004): warning, detention, guardian meeting,
   suspension (creates student_suspensions through Student; attendance SU)
Remarks and achievements -> timeline; shared remarks visible to guardians
R3 counseling: restricted sessions; timeline shows only "Counseling session held" (DEC-122)
```

### Staff onboarding and separation

| Flow | Steps |
|---|---|
| Onboarding | [HRM] employee (SCR-HR-003) -> branch assignments -> user with role assignment (SCR-IAM-022) -> device enrollment (SCR-ATT-019) -> salary setup (R2, SCR-PAY-004) -> welcome SMS with first-time code |
| Separation (R2) | APR:employee_separation -> clearance (assets, library, advances, loans) -> last day: user deactivated, device enrollment ended, teaching assignments reassigned prompt -> final settlement payroll run |

---

## 11.31 State Machines Summary

| Entity | States | Moves |
|---|---|---|
| Admission application | submitted, under_review, test_scheduled, interview_scheduled, approved, waitlisted, rejected, admitted, withdrawn | 11.15 |
| Student | active, suspended, withdrawn, struck_off, graduated | D6 6.9 |
| Enrollment | active, completed, closed, cancelled (with outcome) | D6 6.8 |
| Section attendance sheet | not_marked, draft, submitted, finalized | DEC-180 |
| Leave application | pending, approved, rejected, cancelled, withdrawn | 11.18 and approval statuses |
| Marks submission | draft, submitted, returned, approved | DEC-181 |
| Result batch | pending, processing, processed, stale, approved, published, unpublished, superseded | DEC-182, D7 7.29 |
| Fee voucher | issued, partially_paid, paid, overdue, cancelled, superseded | Part 1, 11.5 to 11.7 |
| Fee payment | posted, reversed | Part 1, 11.7 |
| Cheque | pending, cleared, bounced | Part 1, 11.6 |
| Cashier session | open, closing, closed, closed_with_variance | Part 1, 11.6 |
| Approval request | pending, returned, approved, rejected, withdrawn, apply_failed, expired | Part 1, 11.3 |
| Period lock | unlocked, locked, unlocked_until | 11.26 |
| Academic session | planning, active, closing, closed | D6, 11.21 |
| Timetable version | draft, active, archived | 11.30 |
| External exam entry (R2) | draft, confirmed, paid, submitted, withdrawn, absent, resulted | 11.29 |
| Payroll run (R2) | draft, calculating, calculated, under_review, approved, posted, paid, cancelled | Part 1, 11.12 |
| Library loan (R3) | on_loan, returned, lost | 11.30 |
| Transport allocation (R2) | active, suspended, ended | 11.30 |

Every state change is made by one Action, recorded in the audit log with the actor and reason, and, for student-related changes, written to the student timeline.

---

## 11.32 School-Year Calendar of Scheduled Work (DEC-185)

### National track (Punjab default, session April to March)

| When | Work | Who | Screen or job |
|---|---|---|---|
| January to February | Admissions open for April; new session planned | ADO, PR | SCR-ADM-001, SCR-ACD-003 |
| February to March | Annual exams; marks; annual results | EXC, TCH | 11.19, 11.20 |
| Late March | Results published; promotions run; old session closed | EXC, PR | SCR-RES-010, SCR-STU-010, SCR-ACD-005 |
| 1 April | New session active; April vouchers (generated around 20 to 25 March) | PR, ACT | SCR-ACD-004 |
| May to June | Summer vacation fees collected in advance (deferred, DEC-170) | ACT | 11.5 |
| June to July | Summer vacation; closure calendar events | PR | SCR-ADN-002 |
| September to October | First term or mid-term exams | EXC | 11.19 |
| Late December | Winter vacation | PR | Calendar |
| Board exam season (Class 9 to 12, spring) | Board registration exports and results (R2) | EXC | SCR-EXT-023 |

### Cambridge track (session August to July, DEC-87)

| When | Work |
|---|---|
| August | Session starts; teaching groups formed |
| September to October | November series entries (O Level retakes, AS) and fees |
| January | November series results; mocks |
| January to February | June series entries; entry fee vouchers; remittance |
| May to June | June series exams |
| August | June series results imported and published; enquiries window |

### Every month

| Day | Work |
|---|---|
| 1 | Deferred fee recognition; leave accrual; struck-off candidate list; previous month attendance lock reminder |
| 1 to 5 | Lock previous month: attendance (PR), staff attendance (HRM), fee and accounting (ACT) |
| 5 to 10 | Payroll run for previous month (R2) |
| Due date (default 10) | Fee due; next day overdue and late fees |
| 20 to 25 | Generate next month's vouchers |
| Month end | Bank reconciliation (R2); cash deposits complete |

### Every day

| Time (organization) | Work |
|---|---|
| 00:15 and 00:30 | Overdue marking and late fees |
| 01:00 | Nightly snapshots; integrity check (DEC-126) |
| 06:00 | Attendance sheets prepared |
| 07:00 | Quiet hours released |
| School hours, every 15 minutes | Window finalization, not-marked alerts, auto-absent, device health, today's snapshots |
| 08:00 | Approval digest (DEC-169) |
| 10:00 | Fee reminders |

---

## Assumptions (Part 2)

| ID | Assumption |
|---|---|
| A-92 | Most schools finalize daily attendance by mid-morning; the default window closes at 10:00 and is a branch policy. |
| A-93 | Offers lapse after 7 days unpaid by default, because seat pressure is high in admission season. |
| A-94 | Supplementary exams for conditional promotion are held by the school itself within 60 days of the new session. |
| A-95 | Guardian confirmation of external exam entries is on by default for Cambridge-track schools (fees are large and entries are hard to change after submission). |

## Gaps Found and Added (Part 2)

| ID | Gap | Decision |
|---|---|---|
| GAP-232 | A teacher's wrong tap could send an absence SMS instantly | 10-minute alert delay (DEC-179) |
| GAP-233 | Unmarked students could silently become absent | `NM` stays `NM` after finalization (DEC-180) |
| GAP-234 | Marks lock and reopen rules were not defined | Marks submission states (DEC-181) |
| GAP-235 | Results could be published from out-of-date data | Versioned batches with stale detection (DEC-182) |
| GAP-236 | Duplicate and night-time messages would annoy guardians and waste credits | Notification pipeline with dedupe, quiet hours and credit cap (DEC-183) |
| GAP-237 | Session close steps could run in the wrong order | Fixed close order (DEC-184) |
| GAP-238 | Offers approved but never paid kept seats blocked | Offer lapse rule (11.15) |
| GAP-239 | No shared yearly plan of recurring work | Seeded school-year calendar (DEC-185, 11.32) |

---

Deliverable 11 is complete. Reply CONTINUE for Deliverable 12
