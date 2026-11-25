# Deliverable 6: Academic Structure and Student Lifecycle

**Version:** 1.0 (draft for approval) | **Date:** 23 Nov 2026
**Builds on:** D2 (FR-ACD, FR-STU, FR-RES, FR-ADM), D3 (Student contracts, registries), D4 (hierarchy, DEC-49, DEC-52, branch transfer), D5 (permissions)

---

## 6.1 New Decisions in This Deliverable

| ID | Decision |
|---|---|
| DEC-65 | All academic seed data (levels, groups, subjects, grading schemes) ships as **editable templates with Punjab defaults**. Each organization confirms them in the onboarding wizard against its board's current scheme of studies. Seeds never override data an organization has edited |
| DEC-66 | Every institution gets a system study group **"General"** (code `GEN`) used by levels that have no subject groups (Playgroup to Class 8). This keeps one subject-mapping mechanism for all levels |
| DEC-67 | `study_group_subjects` supports **alternative sets**: subjects in the same `alternative_set` replace each other for a student (for example Islamiyat and Tarjama-tul-Quran for Muslim students, Ethics for non-Muslim students) |
| DEC-68 | A student has one `student_enrollments` row per **placement period**. A section, class or branch change closes the current row and opens a new one. At most one `active` row per student per session, enforced by a unique index on a generated column (`active_marker = 1` when active, otherwise NULL) |
| DEC-69 | Enrollment `status` (lifecycle of the row) is separate from enrollment `outcome` (why it ended: promoted, retained, graduated, progressed, transferred, withdrawn, struck off) |
| DEC-70 | **Combined teaching groups:** teaching assignments that are taught together (elective or lab periods combining students from several sections) share a `combined_group_code`; the timetable allows them in the same slot with the same teacher and room |
| DEC-71 | **Conditional promotion** (compartment): a student may be promoted with `is_conditional = true` and a clearance deadline; a supplementary result clears the condition or triggers reversal to retention through an approval |
| DEC-72 | **Progression after graduation:** a student graduated from a school institution can be progressed into a college institution of the same organization within a configurable window (default 9 months) without a new registration number |
| DEC-73 | Attendance status codes (DEC-24) gain `SU` (suspended). Suspended days are excluded from attendance percentage by default (setting) |
| DEC-74 | Session defaults: school institutions 1 April to 31 March; college institutions 1 September to 31 August; working days Monday to Friday. All configurable |
| DEC-75 | One class teacher per section. By default a teacher can be class teacher of only one section per session (setting) |
| DEC-76 | Board-exam levels by default: Class 9, Class 10, Intermediate Part 1, Intermediate Part 2. Class 8 has an optional "board assessment" flag for provinces that run a Class 8 board-based exam |

---

## 6.2 Academic Structure Model

```
INSTITUTION
  |
  +-- education_levels (ordered, next_level_id, stage, board flags)
  |
  +-- study_groups (GEN, Science, Arts/Humanities, Computer Science, Pre-Medical, ...)
  |      +-- study_group_levels (which levels offer the group)
  |      +-- study_group_subjects (level, subject, compulsory | elective, alternative_set,
  |                                elective_bucket, min/max per bucket, default periods, components)
  |
  +-- subjects (code, names en/ur, type, components, board flag, department)
  |
  +-- academic_sessions (planning | active | closing | closed)
         |
         +-- academic_classes (branch + session + level + study group)      [per branch]
                |
                +-- class_subjects (copied from study_group_subjects, adjustable)
                +-- sections (capacity, shift, wing, class teacher, room)
                       |
                       +-- teaching_assignments (subject, employee, periods, combined_group_code)
                       +-- student_enrollments (placement periods)
                              +-- student_subjects (electives and alternatives chosen)
```

| Entity | Defined at | Purpose |
|---|---|---|
| `education_levels` | Institution | The ladder of grades and its promotion path |
| `study_groups` | Institution | Subject combinations (matric groups, intermediate programs, General) |
| `study_group_subjects` | Institution | The curriculum template per group per level |
| `subjects` | Institution | Subject master |
| `academic_sessions` | Institution | Teaching year |
| `academic_classes` | Branch + session | A level (and group) offered in a branch for a session |
| `class_subjects` | Academic class | Subjects actually offered, copied from the template and adjustable per branch |
| `sections` | Academic class | Groups of students with a class teacher |
| `teaching_assignments` | Section | Who teaches which subject |
| `student_enrollments` | Section | A student's placement for a period |
| `student_subjects` | Enrollment | Electives and alternatives a student takes |

---

## 6.3 Education Levels (seed template)

| Order | Code | Name (English) | Name (Urdu) | Stage | Next level | Board exam | Student login default | Typical age |
|---|---|---|---|---|---|---|---|---|
| 1 | `PG` | Playgroup | پلے گروپ | pre_primary | NUR | No | No | 3 |
| 2 | `NUR` | Nursery | نرسری | pre_primary | PREP | No | No | 4 |
| 3 | `PREP` | Prep / KG | پریپ | pre_primary | C1 | No | No | 5 |
| 4 | `C1` | Class 1 | جماعت اول | primary | C2 | No | No | 6 |
| 5 | `C2` | Class 2 | جماعت دوم | primary | C3 | No | No | 7 |
| 6 | `C3` | Class 3 | جماعت سوم | primary | C4 | No | No | 8 |
| 7 | `C4` | Class 4 | جماعت چہارم | primary | C5 | No | Yes | 9 |
| 8 | `C5` | Class 5 | جماعت پنجم | primary | C6 | No | Yes | 10 |
| 9 | `C6` | Class 6 | جماعت ششم | middle | C7 | No | Yes | 11 |
| 10 | `C7` | Class 7 | جماعت ہفتم | middle | C8 | No | Yes | 12 |
| 11 | `C8` | Class 8 | جماعت ہشتم | middle | C9 | Optional board assessment (DEC-76) | Yes | 13 |
| 12 | `C9` | Class 9 | جماعت نہم | secondary | C10 | Yes (SSC Part I) | Yes | 14 |
| 13 | `C10` | Class 10 | جماعت دہم | secondary | None (graduate) or college `I1` (DEC-52) | Yes (SSC Part II) | Yes | 15 |
| 14 | `I1` | Intermediate Part 1 | انٹرمیڈیٹ حصہ اول | higher_secondary | I2 | Yes (HSSC Part I) | Yes | 16 |
| 15 | `I2` | Intermediate Part 2 | انٹرمیڈیٹ حصہ دوم | higher_secondary | None (graduate) | Yes (HSSC Part II) | Yes | 17 |

Rules: institutions of type `school` get levels 1 to 13; `college` gets 14 and 15; `school_and_college` gets all. Levels can be renamed, reordered, added (e.g., "Montessori", "O Level") or deactivated (BR-ACD-02). `next_level_id` may point to a level in another institution of the same organization (DEC-52).

---

## 6.4 Study Groups and Subject Mapping (seed templates)

**Seed status:** groups marked "Active" are seeded active; "Inactive" are seeded but switched off for organizations to enable if they offer them. Subject lists follow the Punjab scheme of studies as currently understood and must be confirmed per institution (DEC-65).

### 6.4.1 General group (Playgroup to Class 8)

| Levels | Compulsory subjects (default) |
|---|---|
| Playgroup to Prep | English, Urdu, Mathematics, General Knowledge, Islamiyat (oral), Art |
| Class 1 to 3 | English, Urdu, Mathematics, General Knowledge, Islamiyat or Ethics (alternative set `religion`) |
| Class 4 to 5 | English, Urdu, Mathematics, General Science, Social Studies, Islamiyat or Ethics |
| Class 6 to 8 | English, Urdu, Mathematics, General Science, History and Geography (or Social Studies), Islamiyat or Ethics, Tarjama-tul-Quran (Muslim students), Computer Science |

### 6.4.2 Matric groups (Class 9 and 10)

| Group | Code | Seed status | Compulsory | Group subjects |
|---|---|---|---|---|
| Science | `MSCI` | Active | English, Urdu, Islamiyat or Ethics (`religion`), Tarjama-tul-Quran or Ethics (`religion_quran`), Pakistan Studies | Mathematics, Physics, Chemistry, and one of Biology or Computer Science (`elective_bucket = science_choice`, choose 1) |
| Computer Science | `MCS` | Active | Same as Science | Mathematics, Physics, Chemistry, Computer Science |
| Arts / Humanities | `MART` | Active | Same as Science | General Mathematics, General Science, plus electives from bucket `arts_electives` (choose 2; e.g., Education, Civics, Economics, Geography, Elements of Home Economics, Physiology and Hygiene, Persian, Arabic) |
| Health Sciences | `MHS` | Inactive | Same as Science | Physics, Chemistry, Biology, Health Sciences |
| Agriculture | `MAG` | Inactive | Same as Science | Physics, Chemistry, Biology, Agriculture Science |
| Fashion Design | `MFD` | Inactive | Same as Science | General Mathematics, General Science, Computer Science, Fashion Design |

Notes: Pakistan Studies is mapped to Class 10 only by default (single 100-mark paper at SSC Part II under the new scheme); institutions on the older split scheme can map it to both years.

### 6.4.3 Intermediate programs

| Program | Code | Compulsory (Part 1 and Part 2 unless noted) | Group subjects |
|---|---|---|---|
| F.Sc Pre-Medical | `FSPM` | English, Urdu, Islamiyat (Part 1), Pakistan Studies (Part 2), Tarjama-tul-Quran or Ethics | Physics, Chemistry, Biology |
| F.Sc Pre-Engineering | `FSPE` | Same | Physics, Chemistry, Mathematics |
| ICS | `ICS` | Same | Mathematics, Computer Science, and one of Physics, Statistics or Economics (bucket `ics_choice`, choose 1) |
| I.Com | `ICOM` | Same | Principles of Accounting, Principles of Economics, Principles of Commerce, Business Mathematics (Part 1), Business Statistics (Part 2), Banking or Computer Studies (bucket `icom_choice`, choose 1) |
| FA / Humanities | `FA` | Same | Three electives from bucket `fa_electives` (e.g., Education, Economics, Civics, Psychology, Sociology, Islamic Studies, Computer Science, Health and Physical Education, Fine Arts, Punjabi, Persian, Arabic) |

### 6.4.4 Mapping rules

| Rule | Detail |
|---|---|
| Alternative sets (DEC-67) | A student takes exactly one subject from each `alternative_set`. The default choice comes from the student's religion field when recorded; otherwise staff choose at enrollment |
| Elective buckets | Each bucket has `min_choices` and `max_choices`; enrollment fails validation until satisfied |
| Carry-forward | Electives chosen in Class 9 carry to Class 10, Part 1 to Part 2, unless changed with permission (and before the board registration deadline) |
| Class subjects | When an academic class is created, `class_subjects` are copied from `study_group_subjects` for that level; branches may drop electives they do not offer or adjust periods per week |
| Marks components | Each subject template carries default components (e.g., Physics: theory and practical); exams copy these into paper components (D2 FR-EXM-003) |

### 6.4.5 Subject master (seed codes)

| Code | Subject | Type | Default components | Board subject |
|---|---|---|---|---|
| ENG | English | theory | Theory | Yes |
| URD | Urdu | theory | Theory | Yes |
| ISL | Islamiyat | theory | Theory | Yes |
| ETH | Ethics (for non-Muslim students) | theory | Theory | Yes |
| TQ | Tarjama-tul-Quran | theory | Theory | Yes |
| PST | Pakistan Studies | theory | Theory | Yes |
| MTH | Mathematics | theory | Theory | Yes |
| GMTH | General Mathematics | theory | Theory | Yes |
| GSCI | General Science | theory | Theory | Yes |
| PHY | Physics | theory_practical | Theory, Practical | Yes |
| CHE | Chemistry | theory_practical | Theory, Practical | Yes |
| BIO | Biology | theory_practical | Theory, Practical | Yes |
| CS | Computer Science | theory_practical | Theory, Practical | Yes |
| STAT | Statistics | theory_practical | Theory, Practical | Yes |
| ECO | Economics | theory | Theory | Yes |
| ACC | Principles of Accounting | theory | Theory | Yes |
| COM | Principles of Commerce | theory | Theory | Yes |
| BMTH | Business Mathematics | theory | Theory | Yes |
| BSTAT | Business Statistics | theory | Theory | Yes |
| EDU, CIV, GEO, PSY, SOC, IST, HPE, FART, PER, ARB, PUN, HEC, PHYG | Arts electives | theory or theory_practical | As applicable | Yes |
| GK, SST, HG, ART | Lower-level subjects | theory | Theory | No |

---

## 6.5 Grading Scheme Templates (Result module, seeded per institution)

| Template | Use | Bands |
|---|---|---|
| `internal_default` | Internal exams, Playgroup to Class 8 | A+ 90 to 100, A 80 to 89, B 70 to 79, C 60 to 69, D 50 to 59, E 40 to 49, F below 40 (pass 40%) |
| `punjab_legacy` | Boards and schools still on the older scheme | A-1 80 to 100, A 70 to 79, B 60 to 69, C 50 to 59, D 40 to 49, E 33 to 39, F below 33 (pass 33%) |
| `ibcc_revised_2026` | Classes 9 to 12 where the institution follows the revised 10-tier scheme | A++ 96 to 100, A+ 91 to 95, A 86 to 90, B++ 81 to 85, B+ 76 to 80, B 71 to 75, C+ 61 to 70, C 51 to 60, D 40 to 50, U below 40 (pass 40%). Bands A+, A and D to be confirmed against the board notification during onboarding |

Each institution picks a default scheme per level; each exam term can override (internal tests may use `internal_default` while send-up exams mirror the board scheme). Grade remarks are stored in English and Urdu.

---

## 6.6 Academic Session Lifecycle

```
           create                activate                 start closing             close
 (none) ----------> PLANNING ----------------> ACTIVE ----------------> CLOSING ----------------> CLOSED
                       |   (only one ACTIVE per institution)      |  checklist must pass   (read-only)
                       |                                           |
                       +-- admissions for next year, setup,        +-- results, promotions,
                           fee structures, timetables                  locks, carry-forward
```

| Transition | Preconditions | Effects | Permission |
|---|---|---|---|
| Create (planning) | Institution has no other `planning` session (DEC-26) | Session exists; branches can create classes, sections, fee structures; admissions can target it | `academic.session.create` |
| Activate | Previous session is `closing` or `closed`; start date reached or override; at least one academic class exists | Session becomes the default context for attendance, exams, fees; previous `closing` session stays accessible for closing tasks | `academic.session.manage` |
| Start closing | Session `active`; a new session is `planning` or `active` | Session marked `closing`: no new admissions or enrollments into it; promotions and graduations allowed | `academic.session.manage` |
| Close | Checklist (below) passes | Session read-only for all users; enrollments remaining `active` are impossible (checklist), period locks applied to all months | `academic.session.manage` |
| Reopen | Session `closed` | Only through `period_unlock` approval by owner; reopens for a time-boxed correction window | Approval |

**Session close checklist (`SessionCloseCheckRegistry`)**

| Check | Contributed by | Passes when |
|---|---|---|
| Every enrollment has an outcome | Student | No `active` enrollment remains in the session |
| Final results published or explicitly skipped | Result | Every final-term result batch is published, or marked "not applicable" with reason |
| Marks locked | Examination | No marks submission pending or returned |
| Attendance months locked | Attendance | All months in the session locked |
| No pending approvals for the session | Approval | No open requests whose subject belongs to the session |
| Fee balances acknowledged | Fee | Accountant confirms carry-forward report (balances stay on student ledgers; nothing is deleted) |

**Session defaults (DEC-74)**

| Field | School default | College default |
|---|---|---|
| Start / end | 1 April to 31 March | 1 September to 31 August |
| Admission window | 1 February to 30 April | 1 August to 30 September |
| Working days | Monday to Friday | Monday to Friday |
| Promotion date | 31 March | 31 August |

**New session wizard (FR-ACD-003)**

| Step | Action |
|---|---|
| 1 | Choose source session and target session (target in `planning`) |
| 2 | Copy academic classes and sections per branch (names, capacities, shifts, wings, rooms); optionally copy class teachers |
| 3 | Copy class subjects and periods |
| 4 | Optionally copy teaching assignments (employees still active only) |
| 5 | Copy fee structures (via Fee module step, with optional percentage increase) |
| 6 | Copy bell schedules and timetables (optional, Timetable module) |
| 7 | Review summary and run (queued job `CopySessionStructure`) |

---

## 6.7 Classes, Sections and Teaching Assignments

| Topic | Rule |
|---|---|
| Academic class naming | Display name generated from level and group (e.g., "Class 9 Science"), editable |
| Section naming | Free text per branch (A, B, Blue, Iqbal); unique within the academic class |
| Capacity | Required; enforcement mode per branch setting: warning or hard limit (FR-ACD-013) |
| Shift and wing | Required where the branch has more than one shift or wing |
| Class teacher | One per section per session (DEC-75); changes are dated; history kept in `section_class_teacher_histories` |
| Room | Optional default room; timetable may override per period |
| Teaching assignment uniqueness | One `primary` assignment per section and subject at any date; any number of `co_teacher` assignments |
| Mid-year teacher change | End the current assignment (`ends_on`), create a new one; marks entry and period attendance use the assignment active on the relevant date |
| Elective teaching | Assignment for an elective subject covers only students in that section whose `student_subjects` include it |
| Combined teaching (DEC-70) | Assignments with the same `combined_group_code` (e.g., 9-A and 9-B Computer Science lab) may share teacher, room and slot in the timetable |
| Inter-branch teaching | Requires active `employee_branch_assignments` (D4 4.11) |
| Workload | Sum of periods per week across active assignments; warning above the teacher's configured maximum (default 36) |

---

## 6.8 Student Records: Status and Enrollment

**Student status (`students.status`)**

| Status | Meaning | Portal access | Counted in active students |
|---|---|---|---|
| `active` | Has an active enrollment | Yes | Yes |
| `suspended` | Temporarily barred (discipline); enrollment stays active | Read-only | Yes |
| `struck_off` | Removed from rolls (usually unpaid fees); may be readmitted | Read-only for guardian (fees and history) | No |
| `withdrawn` | Left the institution (any reason including transfer to another school) | Read-only for A-38 period | No |
| `graduated` | Completed the final level of the institution | Read-only for A-38 period | No |

`transferred_out` from D2 is represented as `withdrawn` with reason `transfer_to_other_school` (simpler status set, same reporting). Branch transfers within the organization keep the student `active`.

**Enrollment status and outcome (`student_enrollments`, DEC-68, DEC-69)**

| Status | Meaning |
|---|---|
| `active` | Current placement |
| `completed` | Session ended for this placement with an outcome |
| `closed` | Placement ended mid-session (section, class or branch change; withdrawal; struck off) |
| `cancelled` | Created in error and voided before any dependent records existed |

| Outcome | Set when |
|---|---|
| `promoted` | Promotion to next level |
| `promoted_conditionally` | Compartment promotion (DEC-71) |
| `retained` | Repeats the level |
| `graduated` | Final level completed |
| `progressed` | Moved to another institution of the organization (DEC-52) |
| `section_changed`, `class_changed`, `branch_transferred` | Mid-session moves |
| `withdrawn`, `struck_off` | Left the rolls |

Key enrollment columns (detail in D7): `student_id`, `academic_session_id`, `branch_id`, `academic_class_id`, `section_id`, `study_group_id`, `roll_no`, `starts_on`, `ends_on`, `status`, `outcome`, `is_conditional`, `condition_deadline`, `previous_enrollment_id`, `active_marker` (generated).

---

## 6.9 Student Lifecycle State Machine

```
                         ADMISSION (application approved + paid or override)
                                         |
                                         v
          +-------------------------->  ACTIVE  <------------------------------+
          |  reinstate                 |  |  |  ^                            |
          |                            |  |  |  | readmit                    | progress (DEC-72,
     SUSPENDED <---- suspend ----------+  |  |  |                            |  within window)
                                          |  |  +------ STRUCK_OFF <---------+-- strike off
                                          |  |                               |
                       withdraw (any reason) |                               |
                                          v  v graduate (final level)        |
                                    WITHDRAWN  GRADUATED --------------------+
                                       |
                                       +-- readmit (new admission event, same registration no)

Within ACTIVE (no status change): promote, promote conditionally, retain,
change section, change class, branch transfer, elective change.
```

**Transition catalog**

| Event | From | To (student / enrollment) | Preconditions | Effects and records | Approval | Permission |
|---|---|---|---|---|---|---|
| Admit | Application `approved` | `active` / new enrollment `active` | Seat available or override; admission window open or override; duplicate check passed; admission voucher paid or override | Student, guardians, family, registration no, GR no, admission no, roll no, enrollment, student subjects, guardian activation SMS, timeline `student.admitted` | No | `admission.application.convert` |
| Cancel admission | `active` within N days (default 15) | Student `withdrawn` (reason `admission_cancelled`) / enrollment `cancelled` if no dependent records else `closed` | No marks or attendance beyond N days | Refund per fee policy (approval), timeline | Yes (`withdrawal`) | `student.withdrawal.create` |
| Change section | `active` | `active` / old `closed` (`section_changed`), new `active` | Target section same session and branch, capacity | New roll no (setting), timeline | No | `student.enrollment.manage` |
| Change class | `active` | `active` / old `closed` (`class_changed`), new `active` | Same session; reason required (e.g., placement test) | New class subjects, timeline | No (setting can require) | `student.enrollment.manage` |
| Branch transfer | `active` | `active` / old `closed` (`branch_transferred`), new `active` in target branch | Clearance per fee policy (D4 4.12) | New GR no in target, `current_branch_id` updated, transport ended, `student_transfers` row | Yes (`branch_transfer`) | `student.transfer.create` |
| Elective change | `active` | unchanged | Before board registration deadline or override | `student_subjects` updated with history | No | `student.enrollment.manage` |
| Suspend | `active` | `suspended` / enrollment unchanged | Reason, date range | Attendance shows `SU` (DEC-73); guardian notified | Setting | `student.struck_off.manage` (suspension uses the same principal-level permission) |
| Reinstate | `suspended` | `active` | End date reached or manual | Timeline | No | Same |
| Promote | `active` at session closing | `active` / old `completed` (`promoted`), new `active` in next session | Target session `planning` or `active`; target class exists; eligibility from Result or override with reason | Roll no regenerated (setting), electives carried forward, timeline | No | `student.promotion.process` |
| Promote conditionally | Same | `active` / new enrollment `is_conditional` | Result status `compartment` | Deadline set; supplementary result later clears or reverts | Reversal needs approval | `student.promotion.process` |
| Retain | `active` at session closing | `active` / old `completed` (`retained`), new `active` same level next session | Reason | Timeline; guardian notified | No | `student.promotion.process` |
| Graduate | `active` in final level | `graduated` / `completed` (`graduated`) | Session closing | Certificates may be issued; portal read-only | No | `student.promotion.process` |
| Progress (DEC-72) | `graduated` (within window) or `active` in final level | `active` / new enrollment in other institution's session | Target institution session `planning` or `active`; study group chosen | New GR and admission no in college branch; registration no kept; optional admission voucher | No | `student.promotion.process` |
| Strike off | `active` | `struck_off` / `closed` (`struck_off`) | On struck-off candidate list (Fee rule) or manual reason | Attendance no longer expected; guardian notified; timeline | Principal action (FR-STU-018) | `student.struck_off.manage` |
| Readmit | `struck_off` (same session) or `withdrawn` | `active` / new enrollment | Dues settled or installment plan approved; seat available | Readmission fee voucher; same registration no; new admission no if after session end | No | `student.struck_off.manage` (struck off) or `admission.application.convert` (withdrawn) |
| Withdraw | `active` or `suspended` | `withdrawn` / `closed` (`withdrawn`) | Clearance checks pass or waiver approved; reason; effective date | SLC issuable, security deposit refund flow, transport and library ended, guardian access per A-38 | Yes (`withdrawal`) | `student.withdrawal.create` |

All transitions run in a single transaction, write audit entries with reason, record a timeline event, and fire the domain events listed in D3 3.10.

---

## 6.10 Promotion Process

| Step | Screen action | System behavior |
|---|---|---|
| 1. Select | Choose source session, branch, class (or all classes) | Loads active enrollments |
| 2. Eligibility | Shows each student's final result status and attendance % | Uses `PromotionEligibilityProvider` (Result module); without Result module all start as "undecided" |
| 3. Decide | Per student: Promote, Promote conditionally, Retain, Graduate, Progress, Undecided; bulk set by result status | Overrides of the suggested decision require a reason |
| 4. Target mapping | Map source sections to target sections (default same name), or "rebalance" by capacity and gender | Target classes and sections can be created inline if missing |
| 5. Checks | Shows blockers: missing target sections, fee dues (if policy blocks promotion with dues), electives incomplete | Blocking issues must be resolved or overridden with permission |
| 6. Preview | Summary counts per outcome and target section | Nothing written yet |
| 7. Run | Queued `PromoteStudentsChunk` jobs (100 students per chunk, each chunk one transaction) | Progress shown; failed rows listed with reason |
| 8. Report | Promotion report (printable): per student outcome, new class and section, roll no | Timeline events and guardian notifications sent after commit |

**Domain specifics**
- Class 9 to 10 and Part 1 to Part 2 promotions happen on internal results before board Part I results are announced; board results are recorded later (R2) and do not reverse promotion automatically.
- Class 10 and Part 2 students are graduated at session close; progression to the group's college (DEC-72) happens later when college admissions open.
- Students left "Undecided" keep their enrollment active, which blocks session close until decided.
- Re-running promotion for a student who already has a next-session enrollment is refused.

---

## 6.11 Withdrawal, Leaving and Transfer Out

| Item | Rule |
|---|---|
| Reasons (enum) | `transfer_to_other_school`, `relocation`, `financial`, `academic`, `disciplinary`, `health`, `admission_cancelled`, `deceased`, `other` |
| Clearance checks (`ClearanceCheckRegistry`) | Fees cleared (Fee), books returned (Library, R3), transport ended (Transport, R2), documents returned (Student), ID card returned (setting) |
| Fee for the withdrawal month | Setting: charge full month if withdrawal date is after day N (default 10), otherwise no charge for that month |
| Security deposit | Refund request created automatically (approval `fee_refund`) after clearance |
| Certificates | School Leaving Certificate available after approval; character certificate optional |
| Guardian access | Read-only for one year (A-38) |
| Deceased | Uses withdrawal with reason `deceased`; no notifications to guardians beyond a configurable message; fee dues written off through approval |

---

## 6.12 Struck Off and Readmission

| Item | Rule |
|---|---|
| Candidate rule | Fee setting: N unpaid months (default 2) or amount threshold; job `IdentifyStruckOffCandidates` builds the monthly list |
| Confirmation | Principal reviews list and confirms per student; nothing is struck off automatically |
| Effects | Enrollment `closed` (`struck_off`); attendance not expected; vouchers stop generating; dues remain on ledger |
| Readmission same session | New enrollment in same or different section; readmission fee voucher; dues paid or installment plan approved |
| Readmission after session end | Treated as admission into the current session with the same registration number and a new admission number |
| Reporting | Struck-off register, readmission register |

---

## 6.13 Student Identifiers

| Identifier | Scope | Assigned when | Default format (configurable via numbering service) | Changes on |
|---|---|---|---|---|
| Registration no | Organization, permanent | First admission | `{ORG}-{YY}-{SEQ:6}` | Never |
| GR no | Branch, permanent within the branch | Admission or transfer into a branch | `{BRANCH}-{SEQ:5}` | New GR in each branch the student joins |
| Admission no | Branch, per admission event | Admission, readmission after session end, progression | `{BRANCH}-ADM-{YYYY}-{SEQ:4}` | Each admission event |
| Roll no | Enrollment | Enrollment creation | Sequential within section, or by name order (setting) | Each new enrollment (setting can keep within session) |
| Board registration and roll numbers (R2) | Board | Board registration | Entered from board documents | Per board exam |

---

## 6.14 Certificates

| Certificate | Preconditions | Approval (default) | Notes |
|---|---|---|---|
| School Leaving Certificate (SLC) | Student `withdrawn` or `graduated`; clearance passed or waiver | Principal (part of withdrawal approval) | Fields: registration, GR, admission date, leaving date, class at leaving, conduct, reason, DOB in words |
| Character certificate | Any status | None (issue permission only) | Conduct remarks from discipline record (R2) or entered |
| Bonafide (studying) certificate | Student `active` | None | Very common request for banks, visas, scholarships |
| Migration certificate | Student withdrawn for transfer to another board's area | Principal | Board migration details entered manually |
| Fee clearance certificate | No dues | None | |

All certificates are issued through Document (`issued_documents`): serial number, verification code, snapshot, duplicate marking on reprint, reissue as a new version (BR-DOC-01).

---

## 6.15 Student Timeline Event Catalog

| Event key | Source module | Visible to guardian | Notes |
|---|---|---|---|
| `student.admitted` | Admission | Yes | |
| `student.enrolled` | Student | Yes | Class and section |
| `student.section_changed`, `student.class_changed` | Student | Yes | |
| `student.branch_transferred` | Student | Yes | |
| `student.promoted`, `student.promoted_conditionally`, `student.retained` | Student | Yes | |
| `student.graduated`, `student.progressed` | Student | Yes | |
| `student.suspended`, `student.reinstated` | Student | Yes | Reason visible to staff only |
| `student.struck_off`, `student.readmitted` | Student | Yes | |
| `student.withdrawn` | Student | Yes | |
| `student.profile_corrected` | Student | Yes | Field names only |
| `guardian.linked`, `guardian.unlinked` | Guardian | Staff only | |
| `document.verified` | Student | Yes | |
| `certificate.issued` | Document | Yes | |
| `fee.concession_approved`, `fee.scholarship_awarded`, `fee.refund_issued` | Fee | Yes | |
| `fee.default_warning` | Fee | Yes | Struck-off warning |
| `attendance.long_absence` | Attendance | Yes | Five or more consecutive absences (setting) |
| `attendance.correction_approved` | Attendance | Staff only | |
| `result.published`, `result.withheld` | Result | Yes | |
| `leave.approved` | Leave | Yes | |
| `discipline.incident_recorded` (R2) | Discipline | Setting per severity | |
| `ptm.attended` (R2) | Communication | Yes | |

Storage: `student_timeline_events` with `organization_id`, `branch_id`, `student_id`, `event_key`, `occurred_at`, `actor_user_id`, `title` (snapshot, both languages), `payload` (JSON), `visibility` (`staff`, `guardian`), `source_type`, `source_id`. Written through the Student `TimelineRecorder` contract (D3 3.3). Monthly voucher generation and daily attendance are not timeline events (too frequent); their history lives in the fee and attendance tabs.

---

## 6.16 Academic History View

Each student profile has an "Academic history" tab built from enrollments, grouped by session:

| Column | Source |
|---|---|
| Session, institution, branch | `student_enrollments` |
| Class, group, section, roll no, dates | `student_enrollments` |
| Subjects taken | `student_subjects` |
| Attendance % | `daily_metric_snapshots` or computed from `student_attendances` |
| Results per exam term | `student_results` |
| Outcome | `student_enrollments.outcome` |

Branch visibility follows D4 4.5: previous-branch rows are read-only for current-branch staff, and staff of a previous branch see only their own branch's rows.

---

## 6.17 Additional Business Rules

| ID | Rule |
|---|---|
| BR-ACD-06 | An academic class cannot be deleted once it has an enrollment; it can be deactivated only when it has no active enrollments |
| BR-ACD-07 | A section's capacity cannot be reduced below its current active enrollment count |
| BR-ACD-08 | A study group subject change affects only academic classes created afterwards; existing class subjects change only through the class subjects screen |
| BR-ACD-09 | Copying a session never copies enrollments; students move only through promotion |
| BR-STU-06 | Every lifecycle transition records actor, reason (where required), effective date and timeline event in the same transaction |
| BR-STU-07 | A conditional promotion past its deadline without clearance raises an approval request for reversal; nothing reverts automatically |
| BR-STU-08 | A student cannot have an active enrollment in a `closed` session |
| BR-STU-09 | Readmission always reuses the registration number |
| BR-STU-10 | Elective choices must satisfy bucket rules before an enrollment can be saved, except for levels using the `GEN` group |

---

## Assumptions

| ID | Assumption |
|---|---|
| A-44 | Most private schools in Punjab follow the provincial April start; colleges start Part 1 after matric results (August to September). |
| A-45 | Religion is an optional, sensitive student field used only to preselect the alternative subject; it is visible to staff with `student.student.update` only. |
| A-46 | Internal promotion decisions are made by schools; board results do not automatically change internal promotion outcomes. |
| A-47 | Subject lists in the seeds will be reviewed by each institution during onboarding; the product does not claim to track board scheme changes automatically. |

## Gaps Found and Added

| ID | Gap | Decision | Release |
|---|---|---|---|
| GAP-101 | No handling of Ethics in place of Islamiyat and Tarjama-tul-Quran for non-Muslim students | Alternative sets (DEC-67) | R1 |
| GAP-102 | Elective and lab periods combining sections would show as timetable clashes | Combined teaching groups (DEC-70) | R1 |
| GAP-103 | No compartment (conditional) promotion | DEC-71 | R1 |
| GAP-104 | School-to-college progression timing gap between March graduation and August admission | DEC-72 | R1 |
| GAP-105 | Bonafide (studying) certificate missing | Added to certificates | R1 |
| GAP-106 | No attendance code for suspended students | `SU` (DEC-73) | R1 |
| GAP-107 | Class 8 board assessment not modeled | Optional flag (DEC-76) | R1 |
| GAP-108 | New Punjab matric groups (Health Sciences, Agriculture, Fashion Design) not in brief | Seeded inactive | R1 |
| GAP-109 | Revised 10-tier SSC/HSSC grading not in brief | `ibcc_revised_2026` template | R1 |
| GAP-110 | No admission cancellation within a grace period | Cancel admission transition | R1 |
| GAP-111 | Long absences not surfaced | `attendance.long_absence` timeline event and alert | R1 |
| GAP-112 | Fee rule for the month of withdrawal undefined | Day-N setting (6.11) | R1 |
| GAP-113 | Promotion dependency on board Part I results unclear | Internal promotion independent of board result (A-46) | R1 |
| GAP-114 | No levels without groups in the subject-mapping model | `GEN` group (DEC-66) | R1 |
| GAP-115 | Class teacher history not kept | `section_class_teacher_histories` | R1 |

---

## What Deliverable 7 Covers

The complete database schema and entity relationship diagram: every table from D2 to D6 organized by module, with columns and types, primary keys, composite foreign keys (DEC-43), unique constraints (including the generated-column technique of DEC-68), indexes led by `organization_id`, tenant and branch columns, historical data handling, cardinalities, and explicit explanations of how the schema prevents duplicate enrollment, duplicate attendance, duplicate payments and cross-tenant access.
