# Addendum to Deliverable 6: Cambridge O Level and AS & A Level Support

**Version:** 1.0 (change request, approved by product owner) | **Date:** 23 Nov 2026
**Changes:** D1 (scope, segments, module catalog), D2 (new requirements), D3 (new module), D4 (session rule), D5 (permissions), D6 (structure, grading, lifecycle). Deliverable 7 onwards includes everything in this addendum.

---

## A.1 What Changes

The product now supports two academic tracks side by side:

| Track | Code | Qualifications | Typical institutions |
|---|---|---|---|
| National | `national` | Matric (SSC), Intermediate (HSSC) through BISE or FBISE | Most private schools and colleges in Punjab |
| Cambridge / international | `cambridge` | Cambridge O Level, Cambridge IGCSE, Cambridge International AS & A Level; also Pearson Edexcel International GCSE and International A Level, and OxfordAQA, through the same model | Cambridge-track schools and A Level colleges |

An institution can run one track or both (dual-stream). Lower classes (Playgroup to Grade 8) are shared; the tracks split at Grade 9.

**Facts the design relies on (checked 23 Nov 2026)**
- IBCC treats O Level and IGCSE as SSC-equivalent when Pakistan-based students pass eight subjects including English, Urdu, Islamiyat, Pakistan Studies and Mathematics plus three electives; A Level equivalence to HSSC needs three A Level subjects with at least grade E.
- IBCC grade-to-marks conversion: A* above 90 (subject-specific values published per session), A 85, B 75, C 65, D 55, E 45; numeric grades 9 above 90, 8 = 87, 7 = 82, 6 = 77, 5 = 70, 4 = 60, 3 = 50, 2 = 40.
- Cambridge exams run in May/June and October/November in Pakistan; results come out in August and January. Pearson International A Level also has a January series.
- Cambridge O Level is graded A* to E (Ungraded below E); Cambridge International AS Level is graded a to e with no A*; A Level is graded A* to E.
- Syllabuses change over time: A Level Urdu 9686 ends with the November 2026 series and is replaced by 9866; O Level Pakistan Studies 2059 and Islamiyat 2058 are revised for 2028 exams.
- Private candidates register through the British Council; school candidates are entered by their school's exams officer. This product serves schools, not private candidates.

---

## A.2 New Decisions

| ID | Decision |
|---|---|
| DEC-77 | Academic **track** (`national`, `cambridge`, `common`) is an attribute of education levels, study groups and academic sessions. `common` levels (Playgroup to Grade 8) belong to both tracks |
| DEC-78 | Institutions get a `curriculum` attribute: `national`, `cambridge` or `dual`. It drives which seed templates are offered in onboarding |
| DEC-79 | DEC-26 is amended: an institution may have **one active session per track** (plus one planning session per track). A dual-stream institution that runs one calendar for both tracks uses a single session with track `common` |
| DEC-80 | The single `next_level_id` (D6) is replaced by **`education_level_progressions`** (from level, to level, track, is_default), so Grade 8 can lead to Class 9 or O Level I, and O Level can lead to AS Level or Intermediate Part 1 |
| DEC-81 | **Teaching groups** (subject sets) become a first-class entity: a teaching group belongs to an academic class and subject and may draw students from several sections. A teaching assignment targets either a section or a teaching group. This **replaces DEC-70** (`combined_group_code`); national-track combined elective and lab classes also use teaching groups |
| DEC-82 | Subjects keep one internal master record and gain **external syllabus codes** per awarding body and qualification, with validity by exam series (`valid_from_series`, `valid_to_series`), so syllabus replacements (e.g., 9686 to 9866) are handled without new subjects |
| DEC-83 | A new module **ExternalExam** (code `EXT`, folder `modules/ExternalExam`, permission prefix `external_exam`) manages awarding bodies, exam series, entries, entry fees, results import, enquiries about results, certificates received and IBCC equivalence. It also absorbs national board registration and board results (D2 FR-EXM-013, GAP-32) so both tracks use one mechanism |
| DEC-84 | Cambridge and other external exam fees collected from parents are **pass-through money**: they post to a liability account ("Exam fees payable to awarding body"), not income, and are cleared when the school pays the awarding body |
| DEC-85 | **Predicted grades** are stored per student, subject and target series, entered by the subject teacher and reviewable by HOD; they appear on Cambridge-style report cards and university reference exports |
| DEC-86 | For A Level, each student subject records its **target qualification** (`as_only` or `a_level`) and staging (AS sat in year 1 and carried forward, or linear in year 2) |
| DEC-87 | Cambridge-track session default: 1 August to 31 July (configurable). National-track defaults in DEC-74 are unchanged |
| DEC-88 | Release mapping: Cambridge academic structure, grading, teaching groups, internal exams, report cards, fees and promotion are **R1**. ExternalExam workflows (series, entries, fee collection from entries, results import, enquiries, certificate tracking, equivalence calculator) are **R2**, together with national board support. R1 ships the ExternalExam data foundations (awarding bodies, syllabus codes, candidate numbers) so R2 needs no restructuring |

---

## A.3 Changes to Earlier Deliverables

| Deliverable | Section | Change |
|---|---|---|
| D1 | 1.6 Segments | Add **SEG-6 Cambridge-track school** (Grades 1 to 13, O and A Level, often 3-year O Level) and **SEG-7 Dual-stream school** (Matric and O Level in the same campus) |
| D1 | 1.12 Release scope | Academic, Examination, Result and Fee rows now include Cambridge track in R1; new row ExternalExam: R1 foundations, R2 workflows |
| D1 | 1.15 Module catalog | Add module 34 **ExternalExam** (`external_exam`, Standard tier) |
| D1 | 1.23 Glossary | Add O Level, IGCSE, AS Level, A2, A Level, awarding body, exam series, entry, syllabus code, component, candidate number, UCI, EAR, predicted grade, IBCC equivalence |
| D2 | FR-ACD | Add FR-ACD-016 to FR-ACD-020 (A.10) |
| D2 | FR-EXM-013 | Moved to ExternalExam (FR-EXT) |
| D2 | FR-RES | Add FR-RES-015 and FR-RES-016 (A.10) |
| D2 | FR-FEE | Add FR-FEE-036 (A.10) |
| D2 | New section | FR-EXT-001 to FR-EXT-016 (A.9) |
| D3 | 3.24 | Add ExternalExam technical breakdown (A.9); Academic gains TeachingGroup models and actions |
| D4 | DEC-26 | Amended by DEC-79 |
| D5 | Matrices | Add `academic.teaching_group` and `external_exam.*` permissions (A.11) |
| D6 | 6.3, 6.4, 6.5 | Cambridge levels, groups, subject codes and grading added (A.4 to A.6) |
| D6 | DEC-70 | Superseded by DEC-81 |
| D6 | 6.9, 6.10 | Track selection and cross-track progression added (A.12) |

---

## A.4 Cambridge Track Structure

### A.4.1 Education levels (seed template for `cambridge` and `dual` institutions)

| Order | Code | Name (English) | Name (Urdu) | Track | Grade equivalent | External exam level | Notes |
|---|---|---|---|---|---|---|---|
| 1 to 11 | `PG` to `C8` | Playgroup to Grade 8 | (as D6 6.3) | common | | Optional Cambridge Checkpoint (Grade 6 and 8) | Display label "Grade" instead of "Class" via terminology setting |
| 12 | `O1` | O Level I | او لیول اول | cambridge | Grade 9 | No (some schools enter early subjects) | |
| 13 | `O2` | O Level II | او لیول دوم | cambridge | Grade 10 | Yes | Final year for 2-year O Level schools |
| 14 | `O3` | O Level III | او لیول سوم | cambridge | Grade 11 | Yes | Seeded inactive; enable for 3-year O Level schools |
| 15 | `AS` | AS Level | اے ایس لیول | cambridge | Grade 12 | Yes (AS series) | |
| 16 | `A2` | A2 Level | اے ٹو لیول | cambridge | Grade 13 | Yes (A Level series) | |

### A.4.2 Progression paths (`education_level_progressions`, DEC-80)

| From | To | Track | Default for |
|---|---|---|---|
| C8 | C9 | national | `national` institutions |
| C8 | O1 | cambridge | `cambridge` institutions |
| O1 | O2 | cambridge | all |
| O2 | O3 | cambridge | 3-year O Level schools |
| O2 or O3 | AS | cambridge | Default after final O Level year |
| O2 or O3 | I1 (Intermediate Part 1) | national | Cross-track progression (needs IBCC equivalence marks for merit, R2) |
| C10 | AS | cambridge | Matric students joining A Level |
| AS | A2 | cambridge | all |
| A2 | none (graduate) | | |

Dual-stream schools choose the track for each student in the Grade 8 promotion step (A.12).

---

## A.5 Study Groups, Option Blocks and Syllabus Codes

### A.5.1 O Level group

| Group | Code | Compulsory (alternative sets) | Option blocks |
|---|---|---|---|
| O Level | `OLVL` | English Language; Mathematics; Urdu First or Second Language (`urdu_choice`); Islamiyat, or exemption for non-Muslim and foreign-national students (`religion_ol`); Pakistan Studies, or exemption for foreign nationals (`pst_ol`) | Elective buckets configured as option blocks (e.g., Block A: Physics / Accounting / Sociology; Block B: Chemistry / Economics / Commerce; Block C: Biology / Computer Science / Business Studies). Each block: choose exactly 1. Minimum electives to reach 8 subjects: 3 |

Rule: the system warns when a Pakistan-based student's subject set would not meet the IBCC eight-subject rule (5 compulsory plus 3 electives); it does not block, because schools also serve dual and foreign nationals.

### A.5.2 AS & A Level groups (pathways aligned with IBCC groups)

| Group | Code | Required subjects | Notes |
|---|---|---|---|
| A Level Pre-Medical | `AL_PM` | Biology, Chemistry, Physics | IBCC pre-medical equivalence |
| A Level Pre-Engineering | `AL_PE` | Mathematics, Physics, Chemistry | IBCC pre-engineering equivalence |
| A Level Computer Science | `AL_CS` | Computer Science, Mathematics, plus Physics or Further Mathematics | |
| A Level Business | `AL_BUS` | Two or three from Business, Accounting, Economics, plus one option | |
| A Level General | `AL_GEN` | Any three or four subjects from the option blocks | |

Each AS & A Level student subject carries a target qualification (`as_only` or `a_level`, DEC-86). An optional "English General Paper" can be set as a fourth AS subject.

### A.5.3 Subject master additions with external codes (seed, DEC-82)

Codes are Cambridge syllabus numbers as currently published; each institution confirms them during onboarding (DEC-65).

| Internal code | Subject | Qualification | Cambridge code | Components (default) |
|---|---|---|---|---|
| ENGL | English Language | O Level | 1123 | Paper 1, Paper 2 |
| URD1 | Urdu First Language | O Level | 3247 | Paper 1, Paper 2 |
| URD2 | Urdu Second Language | O Level | 3248 | Paper 1, Paper 2 |
| ISL | Islamiyat | O Level | 2058 (revised syllabus for 2028) | Paper 1, Paper 2 |
| PST | Pakistan Studies | O Level | 2059 (revised syllabus for 2028) | Paper 1, Paper 2 |
| MTH | Mathematics (Syllabus D) | O Level | 4024 | Paper 1, Paper 2 |
| AMTH | Additional Mathematics | O Level | 4037 | Paper 1, Paper 2 |
| PHY | Physics | O Level | 5054 | Theory papers, Practical or Alternative to Practical |
| CHE | Chemistry | O Level | 5070 | Same |
| BIO | Biology | O Level | 5090 | Same |
| CS | Computer Science | O Level | 2210 | Paper 1, Paper 2 |
| ACC | Accounting | O Level | 7707 | Paper 1, Paper 2 |
| ECO | Economics | O Level | 2281 | Paper 1, Paper 2 |
| BST | Business Studies | O Level | 7115 | Paper 1, Paper 2 |
| SOC | Sociology | O Level | 2251 | Paper 1, Paper 2 |
| COMM | Commerce | O Level | 7100 | Paper 1, Paper 2 |
| STATO | Statistics | O Level | 4040 | Paper 1, Paper 2 |
| LIT | Literature in English | O Level | 2010 | Paper 1, Paper 2 |
| PHY | Physics | AS & A Level | 9702 | AS papers, A2 papers, practical |
| CHE | Chemistry | AS & A Level | 9701 | Same |
| BIO | Biology | AS & A Level | 9700 | Same |
| MTH | Mathematics | AS & A Level | 9709 | Pure, Mechanics or Statistics options |
| FMTH | Further Mathematics | A Level | 9231 | |
| CS | Computer Science | AS & A Level | 9618 | |
| ACC | Accounting | AS & A Level | 9706 | |
| ECO | Economics | AS & A Level | 9708 | |
| BUS | Business | AS & A Level | 9609 | |
| SOC | Sociology | AS & A Level | 9699 | |
| PSY | Psychology | AS & A Level | 9990 | |
| IT | Information Technology | AS & A Level | 9626 | |
| URDA | Urdu Language and Literature | A Level | 9866 from 2027 series (replaces 9686, last series November 2026) | |
| EGP | English General Paper | AS Level | 8021 | |
| LAW | Law | AS & A Level | 9084 | |

Pearson Edexcel and OxfordAQA codes are added the same way per institution when used.

---

## A.6 Grading Templates (Cambridge track)

| Template | Grades | Internal percentage bands (school policy defaults) | Use |
|---|---|---|---|
| `cambridge_o_level` | A*, A, B, C, D, E, U | A* 90+, A 80 to 89, B 70 to 79, C 60 to 69, D 50 to 59, E 40 to 49, U below 40 | Internal exams and mocks for O1 to O3 |
| `cambridge_as` | a, b, c, d, e, U (no A*) | a 80+, b 70 to 79, c 60 to 69, d 50 to 59, e 40 to 49, U below 40 | AS internal exams and mocks |
| `cambridge_a_level` | A*, A, B, C, D, E, U | A* 90+, A 80 to 89, B 70 to 79, C 60 to 69, D 50 to 59, E 40 to 49, U below 40 | A2 internal exams and mocks |
| `igcse_numeric` | 9 to 1, U | Set by school | IGCSE 9-1 syllabuses |

Internal bands are school policy only; real Cambridge grade thresholds change per syllabus and series and are never computed by the system. External results are stored as reported grades (A.9).

---

## A.7 Teaching Groups (DEC-81)

| Item | Rule |
|---|---|
| Entity | `teaching_groups` (organization, branch, academic class, subject, name such as "O2 Physics Set 1", capacity, room) and `teaching_group_members` (teaching group, student enrollment, joined_on, left_on) |
| Assignment target | `teaching_assignments` has `section_id` (nullable) and `teaching_group_id` (nullable); exactly one is set (check constraint) |
| Roster | Section target: students actively enrolled in the section who take the subject. Teaching group target: active members |
| Membership rules | A member must be actively enrolled in the group's academic class and take the subject in `student_subjects`; one group per student per subject per session at a time |
| Uses | Marks entry grids, period attendance, homework, lectures and quizzes (R2), timetable slots, teacher workload |
| Timetable | One slot per teaching group; students in the group are blocked from other entries in that slot (student clash check added for teaching groups) |
| National track | Combined elective and lab classes (formerly DEC-70) are teaching groups spanning sections |
| Automatic creation | When a section-level subject has students split across electives, the coordinator can generate teaching groups from option-block choices in one step |

---

## A.8 Calendar and Internal Assessment for the Cambridge Track

| Item | Default |
|---|---|
| Session | 1 August to 31 July (DEC-87) |
| Exam terms | Monthly or unit tests, Mid-year exam (December), Mock exam (January to February, before May/June series), Final exam for non-exam years (May to June) |
| External series | May/June (main), October/November (retakes and some early entries), January for Pearson International A Level units |
| Promotion | At session close after final internal results; O Level graduation once the final-year series is sat |
| Report cards | Cambridge-style template: percentage, internal grade, predicted grade, effort and attainment ratings, teacher comments per subject; ranking off by default |
| Attendance, fees, communication | Same as national track |

---

## A.9 ExternalExam Module (DEC-83)

**Purpose:** manage everything about exams set by external awarding bodies (Cambridge, Pearson Edexcel, OxfordAQA, BISE boards, FBISE) for school candidates.
**Key entities:** `awarding_bodies`, `subject_external_codes`, `external_candidate_ids`, `exam_series`, `external_exam_fee_schedules`, `external_exam_entries`, `external_exam_results`, `external_exam_result_imports`, `external_exam_enquiries`, `external_certificates`, `predicted_grades`, `ibcc_conversion_rules`, `equivalence_calculations`

### A.9.1 Requirements

| ID | Requirement | Roles | Rel | Pri |
|---|---|---|---|---|
| FR-EXT-001 | Awarding bodies: seeded catalog (Cambridge International, Pearson Edexcel, OxfordAQA, nine Punjab BISE boards, FBISE) plus organization-specific additions; centre number per branch per awarding body | OA, EXC | R1 | M |
| FR-EXT-002 | External syllabus codes per subject, awarding body and qualification with components and validity series (DEC-82) | EXC, AC | R1 | M |
| FR-EXT-003 | Candidate identifiers per student per awarding body (candidate number, UCI or board registration number) | EXC | R1 | M |
| FR-EXT-004 | Exam series: awarding body, name (e.g., "June 2027"), entry deadlines per stage (normal, late stage 1, late stage 2), retake window, results date | EXC | R2 | M |
| FR-EXT-005 | Fee schedules per series and syllabus or component option, per entry stage, in PKR (optional reference to foreign currency amount and rate) | EXC, ACT | R2 | M |
| FR-EXT-006 | Entry planning: generate draft entries from student subjects for a series; teacher or coordinator review; guardian confirmation in portal (optional) | EXC, AC, GDN | R2 | M |
| FR-EXT-007 | Entry fee vouchers generated from confirmed entries (ad-hoc vouchers through Fee); unpaid entries flagged before each deadline | EXC, ACT | R2 | M |
| FR-EXT-008 | Entry statuses: `draft`, `confirmed`, `paid`, `submitted`, `withdrawn`, `absent`, `resulted`; withdrawal after submission follows the awarding body's refund rules and an approval | EXC | R2 | M |
| FR-EXT-009 | Entry export in a format suitable for upload or manual entry into the awarding body's portal; national board registration exports | EXC | R2 | M |
| FR-EXT-010 | Results import from statement-of-results files (CSV/Excel mapping) or manual entry; syllabus grades and component grades | EXC | R2 | M |
| FR-EXT-011 | Results publication to students and guardians after review; result analytics (grade distribution per subject and teacher, value added against predicted grades) | EXC, PR | R2 | M |
| FR-EXT-012 | Enquiries about results (re-mark requests): student or guardian request, school decision, fee voucher, outcome recording | GDN, STD, EXC | R2 | S |
| FR-EXT-013 | Certificates register: received from awarding body, handed over to student with signature date | EXC, BM | R2 | S |
| FR-EXT-014 | Predicted grades per student, subject and series; HOD review; export for university references (DEC-85) | ST, HOD, EXC | R1 | S |
| FR-EXT-015 | IBCC equivalence calculator: conversion rules per session (grade to marks, subject-specific A* values), eight-subject and three-subject rules, result shown as marks out of 1100; used for cross-track progression and merit lists | EXC, ADO, GDN | R2 | S |
| FR-EXT-016 | National boards: board registration data export, board roll numbers, board results entry and analysis (replaces FR-EXM-013) | EXC | R2 | M |

### A.9.2 Entry workflow

```
Student subjects (per session)
   -> EXC creates exam series (deadlines, fees)
   -> Generate draft entries (syllabus + component option per student)
   -> Coordinator review -> optional guardian confirmation in portal
   -> Confirm -> entry fee vouchers (Fee, liability posting DEC-84)
   -> Payment before deadline -> entries "paid"
   -> Export -> exams officer submits in awarding body portal -> mark "submitted"
   -> Late entries use late-stage fee schedule
   -> Results file import -> review -> publish -> analytics
   -> Enquiries about results (optional) -> certificates received -> handed over
```

### A.9.3 Accounting for exam fees (DEC-84)

| Event | Debit | Credit |
|---|---|---|
| Entry fee voucher issued | Fee receivable | Exam fees payable to awarding body (liability) |
| Parent pays | Cash or bank | Fee receivable |
| School pays awarding body | Exam fees payable to awarding body | Bank |
| School keeps a handling charge (optional fee head) | Fee receivable | Exam handling income |

### A.9.4 Technical breakdown (D3 format)

- **Models:** AwardingBody, SubjectExternalCode, ExternalCandidateId, ExamSeries, ExternalExamFeeSchedule, ExternalExamEntry, ExternalExamResult, ExternalExamResultImport, ExternalExamEnquiry, ExternalCertificate, PredictedGrade, IbccConversionRule, EquivalenceCalculation
- **Controllers (App):** AwardingBodyController, SubjectExternalCodeController, CandidateIdController, ExamSeriesController, FeeScheduleController, EntryPlanningController, EntryController, EntryExportController, ResultImportController, ExternalResultController, EnquiryController, CertificateRegisterController, PredictedGradeController, EquivalenceController; **(Guardian, Student)** ExternalEntryController, ExternalResultController, EnquiryRequestController
- **Actions:** GenerateDraftEntries, ConfirmEntries, IssueEntryFeeVouchers, MarkEntriesSubmitted, WithdrawEntry, ImportResults, PublishExternalResults, RecordEnquiryOutcome, RecordCertificateHandover, SavePredictedGrade, CalculateEquivalence
- **Approval handlers:** EntryWithdrawalHandler
- **Services:** EntryFeeCalculator, ResultFileMapper, EquivalenceCalculator, ValueAddedAnalyzer
- **Contributions:** voucher line contributor (entry fees), clearance check (certificates collected, optional), student profile tab "External exams"
- **Events:** ExternalEntriesConfirmed, ExternalResultsPublished, ExternalEnquiryDecided
- **Jobs:** IssueEntryFeeVouchersChunk, ImportResultFile, SendEntryDeadlineReminders
- **Depends on:** Academic, Student, Guardian, Fee, Accounting (posting), Approval, Notification, ImportExport, Document

---

## A.10 Additional Requirements in Existing Modules

| ID | Requirement | Rel |
|---|---|---|
| FR-ACD-016 | Track attribute on levels, study groups and sessions (DEC-77, DEC-79) | R1 |
| FR-ACD-017 | Progression paths with defaults per track (DEC-80) | R1 |
| FR-ACD-018 | Teaching groups: create, generate from option blocks, manage members, assign teacher (DEC-81) | R1 |
| FR-ACD-019 | Option blocks as elective buckets with "choose exactly one" and timetable block label | R1 |
| FR-ACD-020 | Target qualification per A Level student subject (DEC-86) | R1 |
| FR-RES-015 | Cambridge grading templates and Cambridge-style report card template with predicted grades | R1 |
| FR-RES-016 | Grade-only results (no total or percentage) for external results and for subjects assessed by grade | R1 |
| FR-FEE-036 | Fee head type `pass_through` posting to a liability account (DEC-84) | R1 |

---

## A.11 Permission Additions (D5 format)

| Resource | View | Create / Update | Delete | Other actions |
|---|---|---|---|---|
| `academic.teaching_group` | MGT, AC, HOD, EXC, TCH | PR, AC | PR, AC | `assign` (members and teacher): AC, HOD·a |
| `external_exam.awarding_body` † | OA, EXC, PR | OA, EXC·i | OA | |
| `external_exam.syllabus_code` | AC, EXC, HOD, TCH | EXC, AC | EXC | |
| `external_exam.candidate_id` | EXC, ADO, BM | EXC | EXC | `import`: EXC |
| `external_exam.series` R2 | MGT, EXC, ACT | EXC | EXC | |
| `external_exam.fee_schedule` R2 | EXC, ACT, PR | EXC, ACT | EXC | |
| `external_exam.entry` R2 | MGT, EXC, AC, CT, ST·a, GDN, STD | `create`: EXC, AC | | `approve` (confirm): EXC. `manage` (mark submitted, withdraw request): EXC. `export`: EXC. Guardian confirmation: GDN (own) |
| `external_exam.entry_withdrawal` R2 | EXC, PR | | | `approve`, `reject`: PR |
| `external_exam.result` R2 | MGT, EXC, AC, HOD, TCH·a, GDN, STD (published) | | | `import`: EXC. `publish`: EXC, PR. `export`: EXC, PR |
| `external_exam.enquiry` R2 | EXC, PR, GDN, STD | `create`: GDN, STD, EXC | | `approve`, `reject`: EXC |
| `external_exam.certificate` R2 | EXC, BM, GDN, STD | EXC, BM | | |
| `external_exam.predicted_grade` | MGT, AC, EXC, HOD, TCH·a | ST, CT | | `approve` (HOD review): HOD, EXC |
| `external_exam.equivalence` R2 | EXC, ADO, AC, GDN, STD | `create` (calculate): EXC, ADO, GDN, STD | | |
| `external_exam.conversion_rule` † R2 | EXC, OA | OA, EXC·g | OA | |

New approval type: `external_entry_withdrawal` (submit: `external_exam.entry.manage`; step 1 principal; approve: `external_exam.entry_withdrawal.approve`).

---

## A.12 Lifecycle Changes

| Change | Detail |
|---|---|
| Track selection | In dual-stream institutions, the Grade 8 promotion wizard requires a track choice per student (national Class 9 or Cambridge O1); the default comes from the institution or a guardian preference recorded at re-registration |
| Track switch mid-stream | Allowed only at session boundaries through a class change with reason and principal approval; subject choices are re-validated |
| O Level completion | Students in the final O Level year are graduated or progressed (to AS in the same or another institution, or to Intermediate Part 1) at session close; early-sat subjects stay in their external results history |
| Cross-track progression | O Level to Intermediate and Matric to AS use progression paths (A.4.2); IBCC equivalence marks (R2) can be stored on the new enrollment for merit decisions |
| A Level staging | AS results recorded against the AS series; carried-forward AS subjects in A2 keep their AS grade link; `as_only` subjects end after AS year |
| Timeline events | Add `external_exam.entries_confirmed`, `external_exam.results_published`, `external_exam.certificate_handed_over`, `student.track_selected` |

---

## Assumptions

| ID | Assumption |
|---|---|
| A-48 | Schools enter school candidates themselves through the awarding body's portal; the product prepares and exports entries but does not connect to Cambridge, Pearson or British Council systems directly (no public API assumed). |
| A-49 | Entry fees are charged to parents in PKR; any foreign-currency reference is informational. |
| A-50 | IBCC conversion values, including subject-specific A* marks, are entered per session by the school or platform because they change yearly. |
| A-51 | Many Cambridge schools use "Grade" instead of "Class" and three-year O Level; both are configuration, not separate code. |
| A-52 | Private candidates (registered through the British Council) are out of scope. |

## Gaps Found and Added

| ID | Gap | Decision | Release |
|---|---|---|---|
| GAP-116 | No Cambridge track in the structure | Track attribute, levels, groups (DEC-77, DEC-78) | R1 |
| GAP-117 | Single next level could not express branching paths | Progression paths (DEC-80) | R1 |
| GAP-118 | Dual-stream schools with different calendars | One active session per track (DEC-79) | R1 |
| GAP-119 | Cross-section subject sets common in O and A Level | Teaching groups (DEC-81) | R1 |
| GAP-120 | Syllabus code changes over time | External codes with validity series (DEC-82) | R1 |
| GAP-121 | No external exam entries, fees or results handling | ExternalExam module (DEC-83) | R2 |
| GAP-122 | Exam fees recorded as income would overstate revenue | Pass-through liability (DEC-84) | R1 |
| GAP-123 | No predicted grades | DEC-85 | R1 |
| GAP-124 | AS-only versus full A Level subjects | Target qualification (DEC-86) | R1 |
| GAP-125 | IBCC equivalence needed for cross-track progression and university applications | Equivalence calculator | R2 |
| GAP-126 | Student timetable clashes inside teaching groups not checked | Student clash check for teaching groups | R1 |
| GAP-127 | Enquiries about results and certificate handover not tracked | FR-EXT-012, FR-EXT-013 | R2 |
