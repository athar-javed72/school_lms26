# Deliverable 14: Sequential Antigravity Implementation Prompts
# Part 6 of 9: PH-07 Examination, Results and External Exam Foundations and PH-08 Student Lifecycle and Session Close

**Version:** 1.0 (draft for approval) | **Date:** 24 Nov 2026
**Builds on:** D14 Parts 1 to 5, D02 (FR-EXM, FR-RES, FR-EXT, FR-STU), D04 4.12, D05 5.7.8, 5.7.15, 5.7.16, 5.9, 5.11, D06A A.11, D06 6.5 and 6.8 to 6.12, D06A (A.6, A.9, DEC-82 to DEC-88), D07 Part 3 (7.28 to 7.30), D10 Part 2 (10.23) and Part 3 (10.32 to 10.34, wireframes W-04, W-11, W-12), D11 Part 1 11.4 and Part 2 (11.19 to 11.25), D12 Part 2 (DEC-215, performance budgets)
**How to run:** start a new Antigravity conversation for each step and paste the Phase 0 Master Context Prompt before the step prompt.

---

## 14.15 New Decisions

| ID | Decision |
|---|---|
| DEC-266 | **P07.3 builds the external exam foundations only (DEC-88).** It builds awarding bodies, centre numbers, syllabus codes, candidate identifiers, predicted grades and the `ibcc_conversion_rules` catalog table. Series, fee schedules, entries, result imports, enquiries, certificates, equivalence calculations and national board workflows are R2 and move to PH-16. Their tables are created there too. This corrects the D13 P07.3 wording and the D07 7.51 M08 batch. |
| DEC-267 | **Examination reads fee and grading data only through contracts.** Examination sits below Fee and Result in the dependency order. It defines `AdmitCardEligibilityCheckRegistry`, where Fee registers the dues check in P07.1, and `GradeOptionsProvider`, which Result binds in P07.2. Until P07.2 binds it, grade-only papers can't be entered. Result reads fee dues for its withhold suggestions through `ResultWithholdSuggestionRegistry`, which Fee fills in P07.2. `exam_terms.grading_scheme_id` becomes a soft reference (DEC-108), because `grading_schemes` belongs to Result. |
| DEC-268 | **Marks submission states match DEC-181.** `marks_submissions.status` takes the values `draft`, `submitted`, `returned` and `approved`. The D07 value `open` becomes `draft`. |
| DEC-269 | **Grace marks follow a fixed order.** For each student, grace goes first to the failing subjects closest to their pass mark. A subject gets at most `grace_marks_per_subject`, and a subject that the full grace still can't lift to a pass gets nothing. The student's total grace never exceeds `grace_marks_total`. Grace is never used to change a grade band or a position. The calculator is deterministic: the same inputs always give the same result. |
| DEC-270 | **Supplementary exams are ordinary exams.** `exam_terms.term_type` gains `supplementary`. A supplementary exam covers only the compartment subjects of conditionally promoted students. Its published `exam` batch clears or fails each condition in P08.1. |
| DEC-271 | **Report cards are frozen with the batch version.** Publishing a batch pre-renders one report card PDF per student and one merged PDF per section (DEC-215). They are stored under the batch version, and portals serve only the stored files. A later batch version renders new files and leaves the old ones in place. `ReportCardSectionRegistry` lets modules add sections, such as predicted grades in P07.3, attendance, and remarks. |
| DEC-272 | **Acknowledgement checks use one table.** New table `session_close_acknowledgements` [T-], alias `sca`, in Academic. Columns: `academic_session_id` FK->academic_sessions, `branch_id`? FK->branches, `branch_key` GENERATED, `check_key` VARCHAR(60), `acknowledged_by` FK->users, `acknowledged_at` DATETIME, `note` VARCHAR(500)?. Key `uq_sca (organization_id, academic_session_id, branch_key, check_key)`. It is used for close checks that need a person's review rather than a computed state, such as Fee's "open vouchers of the closing session reviewed" (D11 11.21 step 4). |
| DEC-273 | **New approval type `promotion_reversal`.** It reverts a failed conditional promotion to retained (D06 6.9). Submit permission: `student.promotion.process`. Step 1: `principal_of_branch`. Approve permission: new `student.promotion.approve` (PR, OW). The requester can't approve their own request (BR-APR-01). |

### Corrections to earlier deliverables

| Where | Change |
|---|---|
| D13 13.5 P07.3 | Scope per DEC-266; R2 external exam workflows go to PH-16 |
| D07 7.51 M08 | Only the external exam foundation tables are in M08; the rest move to the PH-16 batch (DEC-266) |
| D07 Part 3 7.28 | `exam_terms.term_type` gains `supplementary` (DEC-270); `exam_terms.grading_scheme_id` is a soft reference (DEC-267); `marks_submissions.status` values per DEC-268 |
| D07 Part 2 7.17 | Adds `session_close_acknowledgements` (DEC-272) |
| D05 5.7.8 and 5.9 | Adds `student.promotion.approve` (PR, OW) and the `promotion_reversal` type (DEC-273) |

---

## 14.16 PH-07 Examination, Results and External Exam Foundations

**Exit criteria:**
- MS-08 is reached.
- Results for 5,000 students process in under 15 minutes.
- A published result changes only through an approved revision and a newly approved, newly published batch version.

### P07.1 Exams, Date Sheets, Admit Cards and Marks Entry

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D11 Part 2 11.19 completely and DEC-181; D07 Part 3 7.28 (exam_terms, exams, exam_papers, exam_paper_components, exam_admit_cards, mark_entries, mark_entry_revisions, marks_submissions) with DEC-28; D10 Part 3 10.32 (SCR-EXM-001 to 018, 020, 021), DEC-164 and wireframe W-11; D11 Part 1 11.4 (marks_revision); D05 5.7.15 and SOD-04, SOD-05; D02 FR-EXM; D06A A.7 (teaching groups in exams); D14 DEC-267, DEC-268.

PHASE GOAL
Run internal exams end to end up to approved marks: exam terms and exams per class, papers with components, date sheets with conflict checks, publication, admit cards with fee clearance, the marks entry grid with codes and submission locks, review with separation of duties, marks import, completion tracking, award lists, revisions through approval, and the examination period lock check.

ALREADY BUILT
PH-00 to PH-06: academic structure with class subjects, teaching assignments and teaching groups with members, students and enrollments, attendance, fee with dues data, approval, notification, import and document engines, period locks.

TASKS
1. Migrations (Examination): exam_terms, exams, exam_papers, exam_paper_components, exam_admit_cards, mark_entries, mark_entry_revisions, marks_submissions exactly as D07 7.28 with the DEC-267 soft grading_scheme_id, DEC-268 statuses and DEC-270 term_type value. invigilation_duties is R2.
2. Actions: CreateExamTerm (session, code, type, weightage, soft grading scheme reference, dates), UpdateExamTerm, CreateExamsForClasses (one exam per term and class per uq_exm; papers copied from class subjects with components from class_subjects.components; total and pass marks from components), UpdateExamSetup (papers, components, maximum and pass marks, grade-only flag; locked once marks exist except adding components with a reason), CancelExam, ScheduleDateSheet (paper dates, start times, durations, rooms), DateSheetConflictDetector (students in two papers at overlapping times, including teaching group members across sections; room double-booked; room capacity against the number of candidates; teacher invigilation is R2), PublishDateSheet (exams.date_sheet_published_at; DateSheetPublished event), GenerateAdmitCards (per exam; roll numbers from enrollment or a setting pattern; eligibility through AdmitCardEligibilityCheckRegistry when exams.fee_clearance_required: blocked rows get blocked_reason), OverrideAdmitCardBlock (examination.admit_card.issue holders, reason required and audited), OpenMarksSubmissions (one marks_submission per paper and section or teaching group from the active teaching assignment, status draft), SaveMarkEntry (per cell; idempotency key; version check like DEC-162; MarksValidator: numeric within 0 and component max, codes ABS, MED, EXM, grade from GradeOptionsProvider for grade-only papers (DEC-267); refused when the submission is submitted or approved), SubmitMarks (blanks require confirmation, errors block; cells lock), ReviewMarksSubmission (approve or return with comment; SOD-04: the teacher who entered marks cannot approve even with HOD or EXC roles; returned reopens cells), ImportMarks (import type exam_marks per paper with the same validator; imported sheets stay draft), RequestMarksRevision (after approval; old and new value, reason; SubmitApprovalRequest marks_revision).
3. Roster rule (D06A A.7): a section submission lists students actively enrolled in the section who take the subject (student_subjects); a teaching group submission lists active members; students who joined or left are shown with dates; withdrawn students after the exam keep their marks.
4. Approval type marks_revision (Examination registers): steps 1 hod_of_subject, 2 exam_controller_of_branch; approve permission examination.marks_revision.approve; SOD-05 requester cannot approve; handler writes mark_entry_revisions and updates the mark entry with current_revision_no; MarksRevised event after commit (Result marks batches stale in P07.2, DEC-182); mark entry version snapshot.
5. Fee registers the dues check in AdmitCardEligibilityCheckRegistry: blocked when the student has overdue vouchers older than setting exam.fee_clearance_months (default 1); the reason shows the amount.
6. Period lock check registered for module examination: no submissions in draft or submitted for exams whose papers fall in the month (D11 11.26).
7. Documents: admit_card (printable, 2 per A4 per D10, photo, roll number, paper schedule, instructions in English and Urdu), date_sheet (printable per class and combined), award_list (printable per paper and section with signature lines).
8. Notifications: examination.date_sheet_published (guardians and students of the class), examination.marks_submission_returned (teacher), examination.marks_due_reminder (teachers with draft submissions 3 days before the due setting).
9. Screens (D10 10.32 exactly): SCR-EXM-001 Exam terms, SCR-EXM-002 Exams, SCR-EXM-003 Exam setup, SCR-EXM-004 Date sheet editor, SCR-EXM-005 Publish date sheet, SCR-EXM-006 Date sheet print, SCR-EXM-007 Admit cards, SCR-EXM-008 Print admit cards, SCR-EXM-009 Mark sheets, SCR-EXM-010 Marks entry (W-11), SCR-EXM-011 Submit marks, SCR-EXM-012 Submissions to review, SCR-EXM-013 Review submission (distribution chart, outliers above two standard deviations flagged, SOD-04 notice), SCR-EXM-014 Import marks, SCR-EXM-015 Request marks revision, SCR-EXM-016 Marks revisions, SCR-EXM-017 Completion tracker, SCR-EXM-018 Award list print, SCR-EXM-020 Guardian: date sheet and admit card, SCR-EXM-021 Student: date sheet and admit card. My classes (SCR-ACD-017) gains the marks link.
10. Definitions: Permissions.php for examination.* per D05 5.7.15; Settings exam.fee_clearance_months, exam.marks_due_days, exam.roll_number_pattern; Navigation (Exams group: Terms, Exams, Date sheets, Admit cards, Mark sheets, Reviews, Revisions, Completion).
11. decisions.md rows: DEC-28, DEC-164, DEC-181, DEC-267, DEC-268, DEC-270.

DATABASE
exam_terms (etm), exams (exm), exam_papers (epp), exam_paper_components (epc), exam_admit_cards (eac), mark_entries (mke), mark_entry_revisions (mkr), marks_submissions (mks) per D07 Part 3 7.28 with DEC-267, DEC-268 and DEC-270.

BUSINESS RULES
- Submitted and approved marks are locked; returned submissions reopen; after approval only marks_revision changes marks (DEC-181).
- The entering teacher never approves their own submission (SOD-04); a revision requester never approves it (SOD-05).
- ABS, MED and EXM are codes, never zero; their effect is decided by result rules later (DEC-28).
- Marks never exceed the component maximum; the database rejects negatives (ck_mke_nonneg).
- Admit cards are blocked by fee dues only when the exam requires clearance; overrides need permission and reason.
- Late submissions are allowed and flagged on the tracker and in the audit.

UI SPEC
- W-11 grid: columns per component with maximums in headers, totals and live grade once Result binds GradeOptionsProvider, codes accepted by typing (a offers ABS), Tab and Enter move down the column, per-cell autosave status, footer counts (entered, blank, errors) and Submit disabled while errors exist.
- Date sheet editor: classes as rows, dates as columns, conflict badges with explanations.
- Completion tracker: class by paper matrix with status pills and "Needs attention" when 3 days remain.

SECURITY AND PERMISSIONS
- Subject teachers enter marks only for their assigned sections and groups (assigned visibility); reviewers per D05.
- Every mark change after submission and every revision audited with reason.
- Guardian and student portals see date sheets after publication and admit cards only when issued.

TESTS
- Exam creation copies papers and components from class subjects; duplicate exam refused (uq_exm).
- Date sheet conflicts: a teaching group student with two overlapping papers detected; room capacity exceeded detected.
- Admit cards: fee dues block when clearance is required; override with reason works; not required means no block.
- Marks entry: above maximum refused; ABS stored as code; idempotent repeat; version conflict per cell; submitted cells locked; return reopens.
- SOD-04: a teacher holding HOD who entered marks cannot approve; another HOD can.
- Roster: section roster includes only students taking the subject; teaching group roster spans sections.
- Revision: approval writes a revision row and fires MarksRevised; SOD-05 enforced.
- Examination period lock blocked with draft submissions.
- Performance: saving a 40-student, 2-component sheet stays within assertQueryCountAtMost per cell and under 1 s total.
- Screen tests including portals, audit, isolation suite, RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. On staging, a term exam is set up for national and Cambridge classes, the date sheet is published, admit cards print with fee blocks, marks are entered, submitted, returned once and approved.
2. A revision after approval passes HOD and exam controller.
3. CI green; staging green.

DO NOT
- Do not compute results, grades for numeric papers, positions or report cards (P07.2).
- Do not build invigilation duties (R2).
- Do not let marks change after approval without marks_revision.

OUTPUT
Report per Master section 20 with the exam walkthrough, SOD evidence and W-11 screenshots in English and Urdu.
```

### P07.2 Results, Publication and Report Cards

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D11 Part 2 11.20 completely, DEC-182; D06 6.5 and D06A A.6 (grading templates); D07 Part 3 7.29; D10 Part 3 10.33 (SCR-RES-001 to 015, 017 to 019) and wireframe W-12; D11 Part 1 11.4 (result_publication, academic_correction); D05 5.7.16; D02 FR-RES (FR-RES-015, FR-RES-016); D12 Part 2 DEC-215 and the result processing budget; D14 DEC-267, DEC-269, DEC-270, DEC-271.

PHASE GOAL
Turn approved marks into correct, versioned results: grading schemes from templates, result rules, queued processing with grace, compartments, positions and attendance, stale detection, withholding, remarks, publication through approval with scheduling, unpublishing, frozen report cards in national and Cambridge layouts, analytics and merit lists, portals, and the promotion eligibility provider.

ALREADY BUILT
PH-00 to PH-06 and P07.1: exams with approved marks, MarksRevised events, GradeOptionsProvider contract, AttendancePercentageQuery, fee data, document engine with batch rendering, approval engine.

TASKS
1. Migrations (Result): grading_schemes, grading_scheme_bands, result_rules, result_batches, student_results, student_subject_results, result_publications exactly as D07 7.29 (uq_rbt_current, uq_rbt_version, generated scope_key and current_marker).
2. Templates: SeedGradingTemplates per institution (internal_default, punjab_legacy, ibcc_revised_2026 per D06 6.5; cambridge_o_level, cambridge_as, cambridge_a_level, igcse_numeric per D06A A.6) with Urdu remarks; contributed as onboarding step "Results defaults" and run idempotently on first use; a default result rule per institution and level from settings. Bands validated for overlap and gaps on save.
3. Bind GradeOptionsProvider (DEC-267) to the exam term's grading scheme bands so grade-only papers can be entered and the W-11 grid shows live grades.
4. ResultCalculator (pure, deterministic): inputs approved marks, rule, scheme, attendance percentages; per subject: components summed, codes applied (ABS by absent_policy exclude or zero; MED excludes the paper when the rule allows; EXM exempts), subject percentage and pass per subject_pass_percentage, grace per DEC-269; per student: totals, percentage with rounding_mode, grade and GPA from bands, failed subjects count, status pass, fail, compartment (failed subjects at or below max_failed_subjects_for_compartment), absent, incomplete, withheld; minimum attendance rule; cumulative annual uses term weightages and requires approved term batches; positions by ranking_method standard or dense within ranking_scope with tie_breakers (for example higher total in a named subject, then date of birth); Cambridge schemes grade-only per paper with no overall percentage unless the rule enables it.
5. Actions: ProcessResults (wizard input class, scope exam, term or cumulative_annual, rule, scheme; completion check lists papers without approved submissions; EXC override lists missing papers in the batch; creates version N+1 and supersedes the previous unpublished version; bulk operation ProcessResultChunk of 200 students per chunk; status processing then processed), MarkBatchesStale (listener on MarksRevised and on late approvals: processed and approved batches containing the paper become stale; a pending result_publication request fails at apply by version check, DEC-168), WithholdResult (result.withhold.create; reason; hidden from portals), SaveReportCardRemarks (teacher remarks with comment bank from setting result.comment_bank, principal remark; autosave), RequestPublication (SubmitApprovalRequest result_publication; refused for stale batches), PublishResult (only approved batches; now or scheduled; command results:publish-scheduled every 5 minutes; result_publications row; batch published; ResultPublished event after commit), UnpublishResult (result.publication.manage; level 3 with reason; portals hide; guardians not re-notified).
6. Approval types (Result and Student register): result_publication (1 principal_of_branch; batch version snapshot; handler sets approved) and academic_correction (Student registers here as promised in P04.3: steps 1 guardian_of_student when raised by a student, 2 class_teacher_of_student, 3 exam_controller_of_branch; payload subject choice, group or roll number; handler applies through UpdateStudentSubjects or enrollment detail actions; enrollment version snapshot).
7. Report cards (DEC-271): document type report_card (printable; templates national and cambridge with predicted grades and component breakdown, FR-RES-015; sections from ReportCardSectionRegistry: attendance from Attendance, remarks, positions only when the rule allows); on publish RenderReportCards pre-renders per student and merges per section on queue documents; SCR-RES-012 serves stored files.
8. Fee fills ResultWithholdSuggestionRegistry (students with overdue dues) shown as suggestions on SCR-RES-008; withholding stays a principal decision.
9. PromotionEligibilityProvider (Result implements the contract used in P08.1): suggested decision per student from the latest published annual or cumulative batch: pass -> promote, compartment -> promote conditionally, fail -> retain, final level pass -> graduate, withheld -> hold.
10. Notifications: result.published (guardians and students with grade summary and portal link; withheld students get "Contact the school"), result.publication_requested (principal), result.batch_stale (exam controller).
11. Screens (D10 10.33 exactly): SCR-RES-001 Grading schemes, SCR-RES-002 Grading scheme bands, SCR-RES-003 Result rules, SCR-RES-004 Result batches, SCR-RES-005 Process results, SCR-RES-006 Result batch detail (W-12 with stale banner and disabled publication), SCR-RES-007 Student result drawer, SCR-RES-008 Withhold result, SCR-RES-009 Request publication, SCR-RES-010 Publish result, SCR-RES-011 Request unpublish, SCR-RES-012 Report cards, SCR-RES-013 Report card remarks, SCR-RES-014 Result analytics, SCR-RES-015 Merit list, SCR-RES-017 Guardian: results, SCR-RES-018 Guardian: report card, SCR-RES-019 Student: results and report cards. Student profile Results tab; guardian.home section "Latest result".
12. Report definitions (P06.6 runner): result.class_result_sheet, result.subject_analysis, result.teacher_performance.
13. Performance: 5,000 students across 40 classes process in under 15 minutes on S2: marks loaded per class in one query per paper set, calculator in memory, chunked inserts of student and subject results; tests/Performance/ResultProcessingTest.
14. Definitions: Permissions.php for result.* per D05 5.7.16; Settings result.comment_bank, result.default_rule per level; Navigation (Exams group: Results, Report cards, Analytics).
15. decisions.md rows: DEC-168, DEC-182, DEC-215, DEC-269, DEC-271.

DATABASE
grading_schemes (grs), grading_scheme_bands (grb), result_rules (rrl), result_batches (rbt), student_results (srs), student_subject_results (ssr), result_publications (rpb) per D07 Part 3 7.29.

BUSINESS RULES
- Results come only from approved marks; missing papers need an explicit EXC override that is recorded on the batch.
- A batch with changed marks is stale and cannot be published (DEC-182); reprocessing creates a new version.
- Publication needs principal approval of that exact version; a reprocessed batch fails the pending request.
- Published report cards and results never change; a correction is a revision, a new version, a new approval and a new publication.
- Grace follows DEC-269; ranking follows the rule's method, scope and tie-breakers; positions reach portals only when ranking_visible_to_portal is on.
- Cambridge internal bands are school policy; real Cambridge thresholds are never computed (D06A A.6).

UI SPEC
- W-12: status line with version and processed time, stale banner with Reprocess, counts by status and pass rate, student rows with total, percentage, grade, status, positions; Request publication disabled with the reason when stale.
- Remarks grid with comment bank picker and character counter; Urdu remarks supported.
- Analytics: grade distribution bars and subject comparison table (charts follow the dataviz rules: categorical colors in fixed order, legend for two or more series).

SECURITY AND PERMISSIONS
- Subject teachers see their subjects' results only; class teachers their section; principals the branch.
- Withheld results never reach portals; report card downloads through signed URLs after the policy check.
- Publication, unpublishing and withholding audited with reasons.

TESTS
- Calculator matrix: ABS exclude versus zero; MED exclusion; EXM; grace allocation per DEC-269 including the case where grace cannot lift a subject; compartment limit; minimum attendance; rounding modes; standard versus dense ranking with tie-breakers; cumulative annual weightages; Cambridge grade-only paper; determinism.
- Template seeding: all seven templates with correct bands; overlap refused on edit.
- Versioning: reprocess supersedes the unpublished version; uq_rbt_current holds.
- Stale: a marks revision after processing stales the batch; a pending publication request ends apply_failed.
- Publication: scheduled publish fires; unpublish hides from portals without re-notifying.
- Report cards: pre-rendered per student and merged per section; old version files kept after a new version publishes.
- PromotionEligibilityProvider suggestions per status.
- Performance: 5,000 students under 15 minutes on S2 (evidence).
- Screen tests including portals, audit, isolation suite, RBAC fixture, ScreenCoverageTest; mutation testing on ResultCalculator with MSI at least 80%.

ACCEPTANCE CRITERIA
1. On staging, a term result is processed for national and Cambridge classes, made stale by a revision, reprocessed, approved, published, and report cards download from the guardian portal.
2. A published result changes only through revision, reprocess, approval and republication (PH-07 exit criterion evidence).
3. 5,000-student processing timing under 15 minutes.
4. CI green; staging green.

DO NOT
- Do not run promotions (P08.1).
- Do not build result rechecks or public result verification (R2).
- Do not edit published batches in place.

OUTPUT
Report per Master section 20 with the calculator matrix, version and stale walkthrough, timing evidence and sample report cards.
```

### P07.3 External Exam Foundations

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D06A A.2 (DEC-82 to DEC-88), A.9 (FR-EXT-001 to 003 and the predicted grades rule DEC-85); D07 Part 3 7.30 rows for awarding_bodies, branch_centre_numbers, subject_external_codes, external_candidate_ids, predicted_grades, ibcc_conversion_rules; D10 Part 3 10.34 (SCR-EXT-001 to 006) and SCR-ACD-008; D06A A.11 (external_exam permissions); D14 DEC-232, DEC-266, DEC-271.

PHASE GOAL
Lay the external exam foundations that R1 schools need and R2 builds on without restructuring: the awarding body catalog with organization additions, centre numbers per branch, external syllabus codes per subject with validity series, candidate identifiers, predicted grades with review, and the IBCC conversion rule catalog table; then close PH-07.

ALREADY BUILT
PH-00 to PH-06, P07.1 and P07.2: subjects, teaching groups, exams, results, report cards with ReportCardSectionRegistry, import engine.

TASKS
1. Migrations (ExternalExam, DEC-232 and DEC-266): awarding_bodies (platform rows with organization_id NULL plus organization rows; organization_key generated; uq_awb), branch_centre_numbers, subject_external_codes, external_candidate_ids, predicted_grades exactly as D07 7.30, and ibcc_conversion_rules (platform default rows plus organization overrides; uq_icr). awarding_bodies and ibcc_conversion_rules are not tenant-scoped through BelongsToOrganization alone: queries read platform rows plus the organization's own rows through a PlatformOrTenantScope; tenants never edit platform rows.
2. Platform seed data (PlatformCatalogSeeder, idempotent by code): awarding bodies CAIE, PEARSON, OXFORDAQA, FBISE and the nine Punjab BISE boards (Lahore, Gujranwala, Faisalabad, Multan, Rawalpindi, Sargodha, Bahawalpur, Dera Ghazi Khan, Sahiwal) with body_type and country; IBCC conversion rows are seeded only from a platform-maintained data file docs/data/ibcc_conversion_rules.csv reviewed by the platform team (no values are typed into code; an empty file seeds nothing).
3. Actions: CreateOrganizationAwardingBody, UpdateOrganizationAwardingBody, SetCentreNumber (per branch and awarding body with validity), CreateSyllabusCode and UpdateSyllabusCode (subject, awarding body, qualification, syllabus code, title, components JSON, valid_from_series and valid_to_series; replacing a syllabus ends the old code's validity and adds the new one, DEC-82), RecordCandidateIdentifier (candidate number, UCI, board registration number; uniqueness per uq_xci_uci and uq_xci_reg), EnterPredictedGrade (subject teacher of the student's section or teaching group; qualification grades from the scheme for that qualification; draft then submitted), ReviewPredictedGrades (HOD or EXC approve; a new prediction supersedes the previous one with superseded_at), ExportPredictedGrades (xlsx for university references; masked per permission).
4. Import types: syllabus_codes (permission external_exam.syllabus_code.create), candidate_ids (external_exam.candidate_id.create).
5. ReportCardSectionRegistry contribution: approved predicted grades appear on Cambridge report cards (DEC-85, DEC-271).
6. SCR-ACD-008 Subjects gains the external syllabus codes column (codes per awarding body with validity chips).
7. Screens (D10 10.34 R1 rows exactly): SCR-EXT-001 Awarding bodies (platform catalog read-only plus organization additions), SCR-EXT-002 Syllabus codes, SCR-EXT-003 Centre numbers, SCR-EXT-004 Candidate identifiers, SCR-EXT-005 Predicted grades entry (per teaching group and series label), SCR-EXT-006 Predicted grades review. Exam series, entries, fees, results, enquiries, certificates and the equivalence calculator are R2 (PH-16).
8. Definitions: Permissions.php for external_exam.awarding_body, syllabus_code, candidate_id, predicted_grade per D06A A.11 (R2 permissions declared with release R2 so the catalog is complete but not granted); Navigation (Exams group: External exams: Awarding bodies, Syllabus codes, Candidates, Predicted grades).
9. decisions.md rows: DEC-82, DEC-83, DEC-85, DEC-88, DEC-232, DEC-266.

DATABASE
awarding_bodies (awb), branch_centre_numbers (bcn), subject_external_codes (sxc), external_candidate_ids (xci), predicted_grades (prg), ibcc_conversion_rules (icr) per D07 Part 3 7.30.

BUSINESS RULES
- Platform catalog rows are read-only to tenants; organizations add their own bodies and overrides.
- One subject keeps one internal record; syllabus replacements are new external codes with validity series (DEC-82).
- Candidate identifiers are unique per awarding body within the organization.
- Only one current predicted grade per student, subject and series (uq_prg); history is kept.
- No external grade thresholds or IBCC values are computed from code.

UI SPEC
- Awarding bodies list shows a "Platform" badge on catalog rows with no edit action.
- Predicted grades grid per teaching group: students as rows, grade picker per student, status pills, submit for review.
- Syllabus code form shows components with codes and maximums and validity series fields.

SECURITY AND PERMISSIONS
- Predicted grades entry limited to assigned teachers; review to HOD and EXC; exports masked.
- Catalog maintenance only by platform super admins through SCR-PLT pages (seeder or a platform form added later); tenants cannot write organization_id NULL rows (test).

TESTS
- PlatformOrTenantScope returns platform plus own rows and never another organization's rows.
- Tenant write to a platform row refused.
- Syllabus replacement ends old validity and keeps history.
- Candidate identifier uniqueness per awarding body.
- Predicted grade supersession keeps one current row.
- Cambridge report card shows approved predicted grades from the registry.
- Screen tests, audit, isolation suite, RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. On staging, a Cambridge branch has centre numbers, syllabus codes for O Level subjects, candidate numbers for O3 students and approved predicted grades shown on the report card.
2. PH-07 exit criteria met; MS-08 and D13 13.6 gate evidence attached.
3. CI green; staging green.

DO NOT
- Do not build series, entries, entry fees, result imports, enquiries, certificates or the equivalence calculator (PH-16).
- Do not type IBCC or Cambridge threshold values into code.
- Do not let tenants modify platform catalog rows.

OUTPUT
Report per Master section 20 with the catalog seeding evidence and predicted grade walkthrough. Then run the D13 13.6 gate for PH-07.
```

---

## 14.17 PH-08 Student Lifecycle and Session Close

**Exit criteria:**
- A full school year can close, and the next session opens with every enrollment history kept.
- A transferred student's balance moves according to the D11 branch transfer rules.

### P08.1 Promotion, Graduation, Conditional Promotion and Session Close

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D06 6.8 to 6.10 completely; D11 Part 2 11.21 and DEC-184; D10 Part 2 SCR-STU-009, SCR-STU-010 (wireframe W-04) and SCR-ACD-005; D05 5.7.8; D02 FR-STU (promotion) and AC-STU-01; D14 DEC-270, DEC-272, DEC-273.

PHASE GOAL
Move a whole school into the next session safely: the promotion wizard with eligibility from results, decisions per student, target section mapping, checks, preview and chunked runs; retention, graduation and progression to another institution; conditional promotion with deadlines cleared or reverted by supplementary results; and the complete, ordered session close checklist.

ALREADY BUILT
PH-00 to PH-07: sessions with lifecycle and SessionCloseCheckRegistry, new session wizard with copy steps, enrollments, results with PromotionEligibilityProvider, supplementary exams, fees, attendance and examination period locks, approval engine.

TASKS
1. Migration (Academic): session_close_acknowledgements per DEC-272.
2. Promotion wizard (D06 6.10) actions: StartPromotionRun (source session, branch, classes; loads active enrollments), SuggestDecisions (PromotionEligibilityProvider; without results all start as undecided), SetPromotionDecision (promote, promote_conditionally, retain, graduate, progress, hold or undecided; overriding the suggestion needs a reason; progress needs a target institution and study group, DEC-52 and DEC-72), MapTargetSections (default same section name in the target session's class from education_level_progressions default per track; rebalance by capacity and gender as an option; missing target classes and sections can be created inline through Academic actions), CheckPromotionRun (blockers: missing targets, electives incomplete for the target level's buckets, fee dues when setting student.promotion_blocks_on_dues is on, students already holding a next-session enrollment), PreviewPromotionRun (counts per outcome and target section; nothing written), RunPromotion (level 3 confirm with the total count; bulk operation PromoteStudentsChunk, 100 students per chunk, one transaction each; per student: old enrollment completed with outcome, new enrollment active in the target session (planning or active) with previous_enrollment_id, electives carried forward through UpdateStudentSubjects, roll numbers per DEC-256 setting, conditional flag and condition_deadline (setting student.conditional_deadline_days, default 60), graduation sets students.status graduated and keeps current_branch_id, progression creates registrations and admissions in the target branch with the same registration number; timeline events; StudentsPromoted event after commit per chunk), PromotionReport (printable per class).
3. Conditional promotion: listener on ResultPublished for exams with term_type supplementary (DEC-270): pass in all compartment subjects clears is_conditional with a timeline event; failure creates an attention item for the principal and lets staff raise promotion_reversal (DEC-273) whose handler moves the student back: new-session enrollment closed with outcome retained context, new enrollment in the previous level's class of the new session, timeline and guardian notification. Command student:conditional-deadlines daily: deadlines passed with no supplementary result raise an attention item.
4. Session close checks registered in SessionCloseCheckRegistry in the fixed order of DEC-184: 1 annual result batches published for every class (Result), 2 every active enrollment of the session has an outcome (Student; undecided blocks), 3 conditional promotions have deadlines (Student), 4 open vouchers of the closing session reviewed (Fee; acknowledgement check through session_close_acknowledgements per branch with note; vouchers stay payable and later vouchers carry arrears memo lines), 5 attendance and examination periods locked through the last month (Attendance and Examination checks through PeriodLockService), 6 pending approvals for the session resolved or withdrawn (Approval; lists them with links). Each check returns passed, message and link; the checklist enforces the order so a later check cannot pass before earlier ones.
5. CloseSession (from P03.1) now succeeds only when every registered check passes; closed sessions become read-only with the D09 banner; enrollments of the closed session keep their history; the next session can be activated per DEC-79.
6. Notifications: student.promoted (guardians with the new class and section), student.graduated, student.condition_cleared, student.promotion_reverted.
7. Screens: SCR-STU-009 Promotions (runs by class with status), SCR-STU-010 Promotion wizard (W-04 steps Source, Decisions, Sections, Review, Run; decisions grid with result, attendance, suggested and chosen decision, reason field when overriding), SCR-ACD-005 Close session checklist now showing all registered checks with acknowledgement buttons for acknowledgement checks.
8. Definitions: Permissions.php adds student.promotion.approve (DEC-273); Settings student.promotion_blocks_on_dues (off), student.conditional_deadline_days (60), student.roll_regenerate_on_promotion (on); Navigation (People group: Promotions).
9. decisions.md rows: DEC-52, DEC-68, DEC-69, DEC-71, DEC-72, DEC-184, DEC-270, DEC-272, DEC-273.

DATABASE
session_close_acknowledgements (sca) per DEC-272. Uses student_enrollments, student_subjects, student_branch_registrations, student_admissions, result tables and academic sessions.

BUSINESS RULES
- A student never holds two active enrollments (DEC-100); a student who already has a next-session enrollment cannot be promoted again.
- Old enrollments are completed with an outcome, never deleted; marks and attendance stay under the old enrollment (AC-STU-01).
- Overriding a suggested decision needs a reason; undecided students block session close.
- Conditional promotions carry a deadline; reverting needs approval (DEC-273).
- The session closes only after every check passes in the DEC-184 order.

UI SPEC
- W-04 decisions grid with filters by result status, bulk "Promote all eligible", warning icons for low attendance, reason inputs revealed on override.
- Section mapping step shows source sections, target sections with capacity bars, and a rebalance preview.
- Run step shows progress from SCR-CORE-004 and failed rows with reasons.

SECURITY AND PERMISSIONS
- Promotion runs need student.promotion.process; reversal approval needs student.promotion.approve; every run audited with parameters and per-student outcomes.
- Closed sessions are read-only for every user.

TESTS
- Promotion: completed old enrollment with outcome and new active enrollment; electives carried; roll numbers regenerated; second run for the same student refused; chunk failure isolates its students.
- Graduation keeps current_branch_id and sets status graduated; progression to a college institution keeps the registration number and creates new GR and admission numbers in the target branch.
- Conditional: supplementary pass clears; failure allows promotion_reversal; requester cannot approve; deadline job raises attention.
- Session close: each check blocks in order; acknowledgement check per branch; close succeeds when all pass; closed session read-only.
- AC-STU-01: after promotion, old marks and attendance show under the old enrollment.
- Screen tests, audit, isolation suite, RBAC fixture, ScreenCoverageTest; mutation testing on PromoteStudentsChunk logic with MSI at least 80%.

ACCEPTANCE CRITERIA
1. On staging, a branch promotes all classes into the planning session, graduates the final level, handles two conditional promotions (one cleared, one reverted) and closes the old session through the full checklist.
2. CI green; staging green.

DO NOT
- Do not build transfers, withdrawals, struck off or certificates (P08.2 and P08.3).
- Do not reverse promotions automatically from board results (D06 6.10).
- Do not delete or overwrite old enrollments.

OUTPUT
Report per Master section 20 with the promotion walkthrough, session close checklist evidence and chunk failure test.
```

### P08.2 Transfers, Withdrawal, Struck Off, Readmission and Suspension

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D06 6.9 (transition catalog), 6.11 and 6.12; D11 Part 2 11.22 to 11.24; D11 Part 1 11.4 (branch_transfer, withdrawal), 11.8 and 11.9; D04 4.12; D07 Part 2 7.19 (student_transfers, student_withdrawals, student_suspensions) with the D07 Part 3 DEC-108 correction (no deposit_refund_id); D10 Part 2 SCR-STU-008, 011 to 015; D05 5.7.8; D14 DEC-262.

PHASE GOAL
Handle every mid-session move and exit correctly: section and class changes, branch transfers through two-principal approval with the fee balance policies, withdrawal with clearance checks and approval, cancel admission, struck off from the fee candidate list, readmission, suspension with automatic reinstatement, and the clearance check registry used by withdrawals and certificates.

ALREADY BUILT
PH-00 to PH-07 and P08.1: enrollments with history, BranchTransferFeeService, StruckOffCandidateQuery, deposit refund draft listener for StudentWithdrawn, AttendancePrefillProviderRegistry, teaching group membership listeners, approval engine.

TASKS
1. Migrations (Student): student_transfers, student_withdrawals (without deposit_refund_id per D07 Part 3 DEC-108; slc_issued_document_id FK -> issued_documents), student_suspensions exactly as D07 7.19. A Fee-owned migration in modules/Fee, timestamped after student_withdrawals, adds the FK fee_refunds.student_withdrawal_id -> student_withdrawals.
2. ClearanceCheckRegistry (Student) with checks: fees cleared (Fee: outstanding and pending reversals per DEC-262), documents returned (Student, from a per-student checklist), ID card returned (Student, when setting student.clearance_id_card is on); Transport (R2) and Library (R3) register later. Each check returns status and a waiver option; waivers need a reason and are stored in the withdrawal's clearance_snapshot.
3. Actions: ChangeSection (same session and branch; capacity; old enrollment closed with outcome section_changed; new active enrollment; roll number per setting; teaching group memberships recalculated (members ended, re-added where the target section's group exists)), ChangeClass (same session; reason; new class subjects; old closed with class_changed), RequestBranchTransfer (target branch, section, effective date, fee policy settle_before_transfer, keep_in_source or carry_to_target; SubmitApprovalRequest branch_transfer), TransferStudentBranch (handler apply, one transaction: source enrollment closed branch_transferred, target enrollment active, new GR number in the target (student_branch_registrations), students.current_branch_id updated (DEC-49), student_transfers row, memberships ended, BranchTransferFeeService applied, StudentTransferred event), RequestWithdrawal (reason from the D06 6.11 list, last date, clearance shown live; SubmitApprovalRequest withdrawal), WithdrawStudent (handler apply: student withdrawn, enrollment closed withdrawn, student_withdrawals row with clearance_snapshot, guardian portal read-only per A-38, future vouchers stopped, StudentWithdrawn event so Fee creates the deposit refund draft and applies the withdrawal-month fee rule: when the effective date is on or before setting fee.withdrawal_charge_after_day (default 10) the unpaid voucher of that month is cancelled through CancelVoucher, otherwise it stays), CancelAdmission (within setting student.cancel_admission_days (15); same withdrawal approval with reason admission_cancelled; enrollment cancelled when EnrollmentDependencyCheckRegistry reports no dependents, otherwise closed), StrikeOff (student.struck_off.manage; from StruckOffCandidateQuery or a manual reason; status struck_off; enrollment closed struck_off; vouchers stop; guardian notified), Readmit (struck off in the same session: dues settled or an approved installment plan, seat available, readmission voucher through Fee, new active enrollment, same registration number; withdrawn students go through admission's "Readmit instead" path from P04.2), SuspendStudent (student_suspensions with dates and reason; status suspended; guardian notified), ReinstateStudent (manual or by command student:reinstate-suspensions daily at 00:20).
4. Approval types (Student registers): branch_transfer (1 principal_of_branch of the source, 2 principal_of_target_branch; approve permission student.transfer.approve; enrollment version snapshot) and withdrawal (1 principal_of_branch; approve permission student.withdrawal.approve; clearance re-checked at apply; enrollment version snapshot). Approval resolver principal_of_target_branch registered (principal role holders of payload.target_branch_id).
5. Prefill provider: Student registers SU in AttendancePrefillProviderRegistry for active suspensions.
6. Source branch visibility after transfer (D04 GAP-84): the source branch keeps a read-only "transferred out" view of its own enrollment rows, attendance and fee history through the existing branch scopes on those rows; the student record itself follows current_branch_id.
7. Notifications: student.transferred (guardians; both principals in-app), student.withdrawn, student.struck_off, student.readmitted, student.suspended, student.reinstated.
8. Screens (D10 10.23 exactly): SCR-STU-008 Change section or class, SCR-STU-011 Transfers, SCR-STU-012 Request branch transfer (fee policy explained per D04 4.12, carry_to_target only with the accounting entitlement), SCR-STU-013 Withdrawals, SCR-STU-014 Request withdrawal (live clearance panel with waiver requests), SCR-STU-015 Struck off and readmission (candidates from the fee rule, strike off or readmit actions). Student profile actions menu shows these by permission (DEC-147).
9. Definitions: Permissions.php for student.enrollment, student.transfer, student.withdrawal, student.struck_off per D05 5.7.8; Settings student.cancel_admission_days (15), student.clearance_id_card, fee.withdrawal_charge_after_day (10); Navigation (People group: Transfers, Withdrawals, Struck off).
10. decisions.md rows: DEC-49, DEC-69, DEC-73, A-37, A-38, D04 GAP-84.

DATABASE
student_transfers (str), student_withdrawals (swd), student_suspensions (ssp) per D07 Part 2 7.19 with the D07 Part 3 correction; FK fee_refunds.student_withdrawal_id added by a Fee migration.

BUSINESS RULES
- Every move closes the old enrollment with an outcome and opens a new one; nothing is deleted.
- Branch transfers need both principals; the fee balance follows the chosen policy (D04 4.12, D11 11.9).
- Withdrawal needs clearance or recorded waivers and principal approval; guardians keep read-only access for one year (A-38).
- Struck-off students stop receiving vouchers; readmission in the same session keeps the registration number.
- Suspended students show SU in attendance and return to active on the end date.

UI SPEC
- Transfer form explains the three fee policies with the student's current balance and what each policy will do.
- Withdrawal form: reason select, last date, clearance list with pass, fail and waiver states, deposit held shown.
- Struck off screen lists candidates with months unpaid and amount; bulk select disabled (each strike-off is an individual decision with reason).

SECURITY AND PERMISSIONS
- Transfers and withdrawals through approval; strike off and readmission by principal-level permission; all audited with reasons.
- The source branch cannot see the student's data created in the target branch after transfer.

TESTS
- Section change closes and opens enrollments, regenerates roll per setting, recalculates group memberships.
- Branch transfer: two principals approve; GR number in target; current_branch_id updated; each fee policy applied through BranchTransferFeeService (keep_in_source two-branch posting, carry_to_target adjustments, settle_before_transfer block) (PH-08 exit criterion).
- Withdrawal: clearance failure blocks without waiver; waiver recorded; approval applies; Fee deposit refund draft created; withdrawal-month voucher rule before and after day 10.
- Cancel admission: cancelled without dependents, closed with dependents.
- Struck off and readmit in the same session keep the registration number; readmission blocked with unpaid dues and no installment plan.
- Suspension: SU prefilled; daily reinstatement.
- Source branch read-only visibility after transfer; target data invisible to the source branch.
- Screen tests, audit, isolation suite, RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. On staging, one student transfers between branches with keep_in_source and another with carry_to_target; balances and postings match D11 11.9.
2. A withdrawal with a held deposit produces the refund draft; a struck-off student is readmitted.
3. CI green; staging green.

DO NOT
- Do not issue leaving certificates here (P08.3).
- Do not delete enrollments or students.
- Do not build transport or library clearance checks (R2, R3).

OUTPUT
Report per Master section 20 with the transfer policy evidence, withdrawal walkthrough and the clearance registry contents.
```

### P08.3 Certificates

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D06 6.11 (certificates); D11 Part 2 11.25; D10 Part 2 SCR-STU-016, SCR-STU-017 and SCR-DOC-004, 005; D09 9.19; D02 FR-STU (certificates) and US-STU-02; D10 A-84; D14 P02.4 document engine contract.

PHASE GOAL
Issue the school's formal certificates as registered documents with serial numbers, frozen content and verification codes: school leaving certificate after an approved withdrawal and passed clearance, character certificate, bonafide certificate and provisional certificate, with reissue and revoke through the document engine; then close PH-08 by proving a full school year.

ALREADY BUILT
PH-00 to PH-07, P08.1 and P08.2: document engine with issuable types, serials, verification codes and reissue or revoke; withdrawals with clearance; results; enrollments with history.

TASKS
1. Document types (Student registers, issuable): leaving_certificate (school leaving certificate: registration and GR numbers, names in English and Urdu, father name, date of birth in figures and words in English and Urdu, admission date, class at admission, class at leaving, leaving date, reason, conduct, dues cleared statement; serial per branch), character_certificate, bonafide_certificate (currently enrolled students only; purpose field), provisional_certificate (after graduation before the board certificate; result summary from the latest published batch). Templates per institution through SCR-DOC-001 and 002; board-specific formats are template variants (A-84).
2. Data providers: DateInWordsFormatter for English and Urdu (for example "Twelfth March Two Thousand Eleven" and its Urdu equivalent) with tests; conduct and remarks from inputs on the issue form.
3. Actions: IssueStudentCertificate (type rules: leaving_certificate requires an approved withdrawal or graduation and re-runs ClearanceCheckRegistry, refusing unless every check passes or holds a recorded waiver (US-STU-02); one current leaving certificate per withdrawal, stored in student_withdrawals.slc_issued_document_id; bonafide requires an active enrollment; provisional requires graduation and a published final result), through Document IssueDocument; reissue and revoke stay in SCR-DOC-005.
4. Timeline events student.certificate_issued, student.certificate_reissued, student.certificate_revoked; audit through the document engine.
5. Screens (D10 10.23 exactly): SCR-STU-016 Certificates (leaving, character, bonafide, provisional; serials; status), SCR-STU-017 Issue certificate (type, inputs, live PDF preview, clearance status for leaving certificates). The register SCR-DOC-004 lists them with the other issued documents.
6. PH-08 year-end rehearsal command demo:rehearse-year (staging only): seeds a branch with a session, runs exams, results, promotion, withdrawals, transfers, certificates and session close, then activates the next session, and prints a verification summary (enrollment history counts, open checks, balances moved).
7. Definitions: Permissions.php for student.certificate per D05 5.7.8; Settings student.certificate_conduct_options; Navigation (People group: Certificates).
8. decisions.md rows: DEC-12, DEC-31, A-84.

DATABASE
No new tables. Uses issued_documents, document_templates, student_withdrawals.slc_issued_document_id and enrollments.

BUSINESS RULES
- A leaving certificate is issued only after approved withdrawal or graduation and passed clearance (or recorded waivers).
- Issued certificates are immutable; corrections are reissues with a reason and a new serial; revoked certificates fail verification.
- Dates of birth appear in figures and words in both languages on leaving certificates.

UI SPEC
- Issue form with type select, inputs per type, clearance panel for leaving certificates, and a PDF preview before issuing.
- Certificates list with serial, type, student, issued by, status pills.

SECURITY AND PERMISSIONS
- student.certificate.issue for BM and ADO; reissue and revoke per D10 SCR-DOC-005; downloads through signed URLs after policy checks.

TESTS
- Leaving certificate refused without approved withdrawal or with failing clearance; issued with a waiver recorded; one current per withdrawal.
- Bonafide refused for withdrawn students; provisional requires graduation and a published result.
- DateInWordsFormatter cases in English and Urdu, including 29 February and year 2000.
- Reissue supersedes; revoke changes status.
- demo:rehearse-year completes and its summary shows full enrollment histories and zero open checks.
- Screen tests, audit, isolation suite, RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. On staging, a leaving certificate is issued after withdrawal with bilingual dates in words; a reissue creates a new serial.
2. demo:rehearse-year proves a full school year closes and the next session opens with every enrollment history kept (PH-08 exit criterion).
3. PH-08 exit criteria met; D13 13.6 gate evidence attached.
4. CI green; staging green.

DO NOT
- Do not build the public verification page (R2, SCR-DOC-006).
- Do not regenerate issued certificates from current data.
- Do not issue leaving certificates without clearance.

OUTPUT
Report per Master section 20 with sample certificates, the rehearsal summary and the D13 13.6 gate for PH-08.
```

---

## Assumptions (Part 6)

| ID | Assumption |
|---|---|
| A-127 | The seeded awarding body list (CAIE, Pearson, OxfordAQA, FBISE and the nine Punjab BISE boards) covers the pilot schools. Others are added per organization. |
| A-128 | The platform team maintains the IBCC conversion data file and reviews it each session year before it is seeded. |
| A-129 | Schools hold supplementary exams for conditional promotions themselves, as normal internal exams (A-94). |
| A-130 | One leaving certificate template per institution, with board variants, meets Punjab school practice (A-84). |

## Gaps Found and Added (Part 6)

| ID | Gap | Decision |
|---|---|---|
| GAP-323 | D13 placed R2 external exam workflows in an R1 step | Foundations only in P07.3 (DEC-266) |
| GAP-324 | Examination needed fee dues and grade bands from modules above it | Registries and contracts (DEC-267) |
| GAP-325 | The D07 marks submission states didn't match DEC-181 | Aligned states (DEC-268) |
| GAP-326 | Grace mark allocation order was undefined | DEC-269 |
| GAP-327 | Supplementary exams had no place in the exam model | `supplementary` term type (DEC-270) |
| GAP-328 | Report cards could change after publication if rendered on demand | Frozen per batch version (DEC-271) |
| GAP-329 | Review-type session close checks had no storage | `session_close_acknowledgements` (DEC-272) |
| GAP-330 | Reverting a conditional promotion had no approval type | `promotion_reversal` (DEC-273) |
