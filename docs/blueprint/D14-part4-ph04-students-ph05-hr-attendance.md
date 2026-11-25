# Deliverable 14: Sequential Antigravity Implementation Prompts
# Part 4 of 9: PH-04 Admission, Students and Guardians and PH-05 HR, Attendance and Leave

**Version:** 1.0 (draft for approval) | **Date:** 24 Nov 2026
**Builds on:** D14 Parts 1 to 3, D02 (FR-ADM, FR-STU, FR-GRD, FR-HR, FR-ATT, FR-LVE), D04 4.11 and 4.12, D05 5.7.7 to 5.7.12, 5.9, 5.10, D06 6.8 to 6.13, D06A A.6 and A.7, D07 Part 2 (7.18 to 7.23), D10 Part 2 (10.22 to 10.27, wireframes W-01, W-02, W-03, W-05, W-06), D11 Part 1 11.4 and Part 2 (11.15 to 11.18), D12 Part 1 12.5 and 12.12
**How to run:** start a new Antigravity conversation for each step and paste the Phase 0 Master Context Prompt before the step prompt.

---

## 14.10 New Decisions

| ID | Decision |
|---|---|
| DEC-247 | **Forward queries with empty defaults.** Academic defines two contracts in P04.1: `ClassTeacherQuery` (section to user) and `TeacherAssignmentQuery` (user to section ids and teaching group ids). Until P05.1 their bindings return empty results. Until then, `assigned` visibility shows teachers nothing (fail closed), and approval steps that resolve to a class teacher escalate under BR-APR-04. P05.1 rebinds both to real data. |
| DEC-248 | **The admission payment gate is a contract.** Admission defines `AdmissionPaymentGate`. Until P06.3 its binding returns "not required", so an approved application can be converted straight away. P06.3 binds the Fee implementation, which issues the admission voucher and requires payment or an override with permission. |
| DEC-249 | **The working-day calendar is a contract.** Core defines `WorkingDayCalendar` with `isWorkingDay(branch, date, shift)` and `reasonFor(branch, date)`. Until P09.2 its binding works from shift working days only. P09.2 rebinds it to Administration holidays and closures. Attendance, leave day counting and sheet preparation call only this contract. |
| DEC-250 | **Portal home sections come from a registry.** Reporting adds `HomeSectionRegistry` in P04.3, so modules can add sections to `guardian.home` and `student.home` before full Reporting arrives in P09.3. Reporting keeps owning the pages (DEC-236 routing through `routes/home.php`). |
| DEC-251 | **Period attendance moves to P09.1.** `student_period_attendances` and SCR-ATT-003 need `timetable_entries`, so they are built with the timetable. P05.2 builds daily section attendance only. |
| DEC-252 | **Suspicious punches get their own table.** `attendance_suspicious_alerts` [B], alias `aps`, columns: `attendance_punch_id` BIGINT UNSIGNED? and `punched_at` DATETIME? (a reference without FK, because partitioned tables can't hold foreign keys), `employee_id`? FK->employees, `alert_type` VARCHAR(30) (`outside_geofence`, `impossible_travel`, `clock_drift`), `details` JSON, `status` VARCHAR(20) (`open`, `dismissed`, `confirmed`), `reviewed_by`?, `reviewed_at`?, `review_note` VARCHAR(500)?. Key `ix_aps_status (organization_id, branch_id, status)`. |
| DEC-253 | **Devices use the platform device hosts from D12 12.28.** ADMS endpoints live on `adms.{platform_domain}` (middleware group `adms`, its own FPM pool, HTTP allowed on `/iclock/*` per D12 Part 1 12.8). Signed device webhooks live on `hooks.{platform_domain}` (middleware group `webhook`). Neither host has a tenant subdomain. The organization is resolved from the device's globally unique serial number (DEC-29) before anything else happens. Route names: `devices.adms.handshake`, `devices.adms.cdata`, `devices.adms.getrequest`, `devices.adms.devicecmd`, `devices.webhook.punches`. Both groups have no session and no CSRF, apply a per-device IP allowlist when set, and rate limit per serial. |
| DEC-254 | **Guardian accounts start without a password.** Conversion or import creates one guardian user per mobile (DEC-22) in the `pending_activation` state. The guardian activates through OTP (SCR-IAM-002 and 003). The system never sends a password by SMS. |
| DEC-255 | **Student logins.** A student gets portal access when the level's `student_login_default` is true or the branch setting `student.portal_levels` includes the level (A-09). Logins are created by the bulk action "Create student logins" (preview first). The username is the registration number. Activation is by OTP to the student's own mobile, or by a one-time setup code that a linked guardian generates in the guardian portal. |
| DEC-256 | **Roll numbers follow a setting.** `student.roll_number_mode` is `sequential_on_enrollment` (default), `alphabetical` or `manual`. "Regenerate roll numbers" is a section bulk action with a preview. `uq_sen_roll` protects uniqueness. |
| DEC-257 | **Consent is an append-only record.** New table `consent_records` [T-], alias `cns`, in Guardian. Columns: `student_id` FK->students?, `guardian_id`? FK->guardians, `admission_application_id`? FK->admission_applications, `consent_type` VARCHAR(30) (`sms`, `whatsapp`, `photo_publicity`), `granted` TINYINT(1), `notice_version` VARCHAR(20), `recorded_via` VARCHAR(20) (`online_form`, `office`, `portal`, `import`), `recorded_by`? FK->users, `recorded_at` DATETIME. Key `ix_cns (organization_id, student_id, consent_type, recorded_at)`. The current value is the latest row. SMS and WhatsApp consent covers broadcasts and non-mandatory events. Mandatory events (OTP, absence, fee due) stay on, per D10 SCR-NTF-002 (DEC-195). |

### Corrections to earlier deliverables

| Where | Change |
|---|---|
| D07 Part 2 7.22 | Adds `attendance_suspicious_alerts` (DEC-252). `attendance_punches.source` gains `admin` and `verify_type` gains `manual`, both for approved missing punches and administrative entries. |
| D07 Part 2 7.20 | Adds `consent_records` (DEC-257) |
| D10 Part 2 10.26 | SCR-ATT-003 is built in P09.1 (DEC-251) |
| D11 Part 2 11.16 | Sheet preparation reads `WorkingDayCalendar` (DEC-249) and the attendance prefill registry (P05.2) |
| D13 13.5 P05.2 | Period attendance moves to P09.1 |

---

## 14.11 PH-04 Admission, Students and Guardians

**Exit criteria:**
- MS-05 is reached.
- A 5,000-row student import previews in under 2 minutes.
- Masking tests pass on student and guardian screens and exports.

### P04.1 Students, Guardians, Families and Enrollments

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D06 6.8, 6.9, 6.12 and 6.13; D06A A.6 and A.7 (subject choices, alternative sets, option blocks); D07 Part 2 7.19 and 7.20 with DEC-99 to DEC-102 and DEC-107; D07 Part 1 encryption rule DEC-95; D05 5.7.8, 5.7.9 and 5.10 (masking); D10 Part 2 10.23 and 10.24 and wireframe W-02; D02 FR-STU, FR-GRD, BR-STU, BR-GRD; D04 4.11; D14 DEC-247, DEC-256.

PHASE GOAL
Build the permanent student record and the guardian side: students with Urdu names and encrypted B-Form, guardians with encrypted CNIC, links with flags and one primary, families formed by shared CNIC, pickup persons, documents, health profile, timeline, numbering (registration, GR, admission, roll), enrollments with subject choices that obey buckets and alternative sets, the student and guardian profile shells with tab registries, and the masked student export.

ALREADY BUILT
PH-00 to PH-03: tenancy, identity, RBAC with AssignedScopeRegistry, audit, Core services (numbering, period locks, idempotency, bulk operations, PiiMasker), approval, notification, import and export, document engines, academic structure (sessions, levels, groups, buckets, subjects, classes, class subjects, sections without class teacher, teaching groups), registries ClassDeactivationCheckRegistry and ClassSubjectRemovalCheckRegistry from P03.2.

TASKS
1. Migrations (Student): student_categories, sponsors, students, student_enrollments, student_subjects, student_branch_registrations, student_admissions (admission_application_id column without FK; P04.2 adds it), student_health_profiles, student_documents, student_timeline_events ([L], PK (id, created_at)) exactly as D07 7.19. student_transfers, student_withdrawals and student_suspensions are created in P08.2.
2. Migrations (Guardian): guardians, guardian_students, families, family_students, pickup_authorizations exactly as D07 7.20, and consent_records per DEC-257 (admission_application_id without FK until P04.2).
3. Identity number handling in Core\Support\IdentityNumber: normalize CNIC and B-Form to 13 digits (accept dashes and spaces), validate the format, compute lookup_hash = HMAC-SHA256(normalized, LOOKUP_HASH_PEPPER), keep last4, encrypt the full value with the encrypted cast. Display through PiiMasker as 35202-•••••••-3 unless the viewer holds core.pii.view_full in that branch. Health text fields use the encrypted cast (DEC-107). Search by CNIC or B-Form uses only the hash.
4. Models with BelongsToOrganization (students, guardians, families and their T- tables) or BelongsToBranch (enrollments, registrations, admissions), Auditable, HasVisibilityScope; morph aliases student, student_enrollment, guardian, family, pickup_authorization. Students never soft delete (D07 7.19).
5. Numbering keys registered with NumberingService and editable on SCR-SET-006: student_registration (organization-wide, default {INST}-{YYYY}-{SEQ5}), student_gr (per branch, default {SEQ}), student_admission (per branch and session, default {BR}-{YY}-{SEQ4}). Imports may supply legacy GR and admission numbers: NumberingService::claim(key, scope, value) refuses duplicates and moves the counter past numeric values.
6. Actions (Student): CreateStudent (registration number, B-Form uniqueness per uq_stu_bform, category default, status active, current_branch_id NULL until enrollment), UpdateStudent (non-protected fields only), UpdateProtectedStudentField (name, name_ur, father_name, father_name_ur, date_of_birth, b_form; permission student.profile.update_protected; reason and supporting document required; audit with old and new values, AC-AUD-01), EnrollStudent (session active or planning; section of that session; one active enrollment per organization per DEC-100; capacity under a row lock on the section with academic.section_capacity_mode warning or hard; first enrollment in a branch creates student_branch_registrations with a GR number; student_admissions row with type; roll number per DEC-256; student_subjects from class subjects: all compulsory, exactly one from each alternative set, electives within each bucket's min and max; sets students.current_branch_id), UpdateStudentSubjects (ends old rows with ends_on and starts new ones; bucket rules re-checked), CancelEnrollment (data-entry errors only, refused when EnrollmentDependencyCheckRegistry reports dependents; later modules register checks for attendance, fees and marks), RegenerateRollNumbers (section bulk action with preview), ChangeStudentCategory, AssignSponsor, UploadStudentDocument, VerifyStudentDocument, UpdateHealthProfile.
7. Actions (Guardian): CreateGuardian (CNIC uniqueness per uq_gdn_cnic, mobile normalized to E.164 with libphonenumber, default region PK), UpdateGuardian, LinkGuardianToStudent (relation and flags; the first link becomes primary; uq_gst_primary keeps exactly one), UnlinkGuardian (guardian.link.delete with reason, BR-GRD-03; refused if it would leave no primary, BR-GRD-02), SetPrimaryGuardian, FormOrJoinFamily (runs after each link: a guardian with a CNIC whose other linked children are in a family brings the student into that family; otherwise a new family with family_code from NumberingService key family; FR-GRD-004), MergeFamilies and SplitFamily (preview of siblings; the fee effects panel appears when Fee registers its preview provider in P06.2), SetCustodyRestriction and ClearCustodyRestriction (guardian.custody.manage, sensitive; reason encrypted; document required; turns off has_portal_access and can_pickup on that guardian's links and ends the guardian's portal sessions), CreatePickupAuthorization, DeactivatePickupAuthorization, VerifyGuardian (office verification or the code printed on slips; code checked against verification_code_hash; FR-GRD-006).
8. Registries filled: ClassDeactivationCheckRegistry (active enrollments block), ClassSubjectRemovalCheckRegistry (active student_subjects block), EnrollmentDependencyCheckRegistry (new, owned by Student), AssignedScopeRegistry resolver for student.student and guardian.* (students in sections and teaching groups returned by Academic TeacherAssignmentQuery plus sections returned for class teachers; empty until P05.1 per DEC-247). Academic adds the ClassTeacherQuery and TeacherAssignmentQuery contracts with empty bindings in this step.
9. Timeline: listener StudentTimelineRecorder writes student_timeline_events for student.created, student.enrolled, student.subjects_changed, student.protected_field_changed, student.category_changed, guardian.linked, guardian.unlinked, guardian.custody_restricted (staff visibility only), student.document_verified. Event catalog lives in Definitions/TimelineEvents.php with English and Urdu titles; later modules add theirs.
10. Profile shells (DEC-161): StudentProfileTabRegistry and GuardianProfileTabRegistry. Student contributes Overview, Documents, Health and Timeline tabs; Guardian contributes the Guardians tab on the student profile and Children, Family, Pickups tabs on the guardian profile. Every tab declares its permission and is hidden without it.
11. Screens (D10 10.23 and 10.24 exactly): SCR-STU-001 Students (filters class, section, status, category, track; search by name, registration number, GR number, guardian mobile, B-Form via hash), SCR-STU-002 Student profile (W-02), SCR-STU-003 Edit student, SCR-STU-004 Change protected field, SCR-STU-005 Student documents, SCR-STU-006 Verify document, SCR-STU-007 Health profile (sensitive, 2FA per DEC-63), SCR-STU-018 Timeline, SCR-STU-020 Bulk actions (change category, regenerate roll numbers, send SMS through a broadcast with the selected students' guardians as audience; "print ID cards" arrives in P09.2), SCR-STU-021 Student categories, SCR-STU-022 Sponsors, SCR-STU-023 Student list print and export (ExportDefinition with masking), SCR-GRD-001 Guardians, SCR-GRD-002 Guardian profile, SCR-GRD-003 Guardian form, SCR-GRD-004 Link or unlink child, SCR-GRD-005 Families, SCR-GRD-006 Merge or split family, SCR-GRD-007 Custody restriction, SCR-GRD-008 Pickup authorizations, SCR-GRD-009 Verify guardian. Student screens SCR-STU-008 to 017 belong to PH-08 and P09.2; portal screens to P04.3; SCR-STU-019 to P04.3.
12. Notification: recipient resolvers guardians_of_student (active links; respects custody restriction and consent per DEC-257; one message per child, D11 11.27) and class_teacher_of_section (through ClassTeacherQuery; empty until P05.1). Broadcast audiences "guardians of selected students" and "guardians by class and section" added to SCR-NTF-009.
13. Definitions: Permissions.php for student.* and guardian.* per D05 5.7.8 and 5.7.9 (sensitive marks on student.health and guardian.custody); Settings: student.roll_number_mode, student.portal_levels (branch), student.required_documents; Navigation (People group: Students, Guardians, Families); Seed data: default student categories (regular, staff_child, sibling, orphan, sponsored) as editable templates.
14. decisions.md rows: DEC-22, DEC-49, DEC-68, DEC-95, DEC-99, DEC-100, DEC-101, DEC-102, DEC-107, DEC-247, DEC-256, DEC-257.

DATABASE
students (stu), student_enrollments (sen), student_subjects (ssb), student_branch_registrations (sbr), student_admissions (sad), student_categories (scg), sponsors (spn), student_health_profiles (shp), student_documents (sdc), student_timeline_events (ste, log), guardians (gdn), guardian_students (gst), families (fam), family_students (fms), pickup_authorizations (pua), consent_records (cns) per D07 Part 2 and DEC-257.

BUSINESS RULES
- One active enrollment per student in the whole organization (DEC-100); a student is never active in two branches (BR-STU-01).
- Protected fields change only through UpdateProtectedStudentField with reason and document (AC-STU-03: 403 without the permission).
- Every student keeps exactly one active primary guardian (BR-GRD-02); unlinking needs a reason (BR-GRD-03).
- Families form automatically by shared guardian CNIC (FR-GRD-004); a student belongs to at most one family.
- A custody-restricted guardian has no portal access and no pickup right (FR-GRD-003).
- Subject choices obey alternative sets (exactly one) and bucket min and max (D06A A.6).
- Teachers see only students of their assigned sections and groups (BR-STU-05); other students return 404 (AC-STU-02).
- Identity numbers are never logged, never shown in full without core.pii.view_full, and never placed in notifications (BR-NTF-03).

UI SPEC
- W-02 profile header: photo (wing restriction per D05 5.10 unless student.photo.view_full), English and Urdu names, registration and GR numbers, class and section, masked B-Form with lock icon, status pill.
- Protected fields render read-only with a "Change protected field" action.
- Urdu name inputs are RTL with Nastaliq preview; mobile inputs accept 03xx formats and show the normalized number.
- Enrollment subject picker shows compulsory subjects locked, alternative sets as radio groups and buckets as cards with "choose exactly N" or "choose between N and M".
- Family merge preview lists siblings before and after.

SECURITY AND PERMISSIONS
- Routes and permissions per D10 10.23 and 10.24; health and custody are sensitive (2FA, audit with reason).
- B-Form and CNIC lookups use the hash; logs and audit metadata store only last4.
- Documents uploaded through FileStorageService (images re-encoded, DEC-192; virus scan when enabled); downloads through signed URLs after the student policy check.
- Exports mask per the exporting user (DEC-62).

TESTS
- IdentityNumber: normalization of dashed and plain input, invalid lengths refused, hash stable, masking with and without core.pii.view_full.
- EnrollStudent: second active enrollment in another branch refused at the database level (uq_sen_active); capacity warning versus hard mode under concurrent enrollment (two parallel requests, one refused in hard mode); GR number created once per branch; roll numbers per mode; subject rules (missing alternative choice refused, bucket above max refused).
- Protected field change: 403 without permission; reason required; audit has old and new values.
- Guardian links: first link is primary; unlink of the only primary refused; family auto-formed by shared CNIC; merge and split keep one family per student.
- Custody restriction: links lose portal access and pickup; active guardian sessions ended.
- Visibility: with the DEC-247 empty binding a teacher sees no students; with a fake binding returning section 6-A the teacher sees 6-A and gets 404 for a 6-B student (AC-STU-02).
- Export: masked B-Form for a user without core.pii.view_full; formula guard.
- Registry checks: class deactivation and class subject removal refused with active students.
- Screen tests (200, 403, 404 cross-tenant and cross-branch), audit entries, isolation suite, RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. On staging, a student is created, enrolled with electives, linked to two guardians (one primary) and shown on W-02 in English and Urdu with masked B-Form.
2. A second sibling sharing the father's CNIC joins the same family automatically.
3. All enrollment and guardian rules hold at the database and action levels.
4. Masking tests pass for SCR-STU-001, SCR-STU-002, SCR-GRD-002 and the student export.
5. CI green; staging green.

DO NOT
- Do not build admission, portals, imports, promotions, transfers, withdrawals or certificates here.
- Do not store CNIC or B-Form in plain text anywhere, including test snapshots and logs.
- Do not delete students or guardian links; end them with dates.

OUTPUT
Report per Master section 20 with the enrollment rule matrix, the masking evidence and the list of registered checks and resolvers.
```

### P04.2 Admission

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D11 Part 2 11.15 completely; D07 Part 2 7.18; D02 FR-ADM, BR-ADM, AC-ADM; D05 5.7.7; D10 Part 2 10.22 and wireframe W-03; D09 9.13 (public form rules); D12 Part 1 12.12 (consent, transparency, retention); D14 DEC-248, DEC-254, DEC-257.

PHASE GOAL
Run admissions from first contact to an enrolled student: form templates per institution, counter and public online applications with mobile OTP, duplicate and sibling detection, document checklist, decisions with seat checks, offer lapse, conversion to a student in one transaction, consent capture, the admission slip with the guardian verification code, the seats report and the GR register.

ALREADY BUILT
PH-00 to PH-03 and P04.1: students, guardians, families, enrollments, numbering, timeline, consent_records.

TASKS
1. Migrations (Admission): admission_form_templates, admission_applications (admission_voucher_id column without FK; Fee adds it in P06.3), admission_application_documents exactly as D07 7.18. Add the deferred FKs student_admissions.admission_application_id and consent_records.admission_application_id. admission_inquiries and admission_assessments are R2 (PH-11).
2. Form templates: AdmissionFormTemplate builder (fields JSON with key, label en and ur, type, required, options; document checklist JSON with code, label, required). A default template per institution is seeded when the module is first used. Editing creates a new version; applications keep form_version.
3. Actions: SubmitApplication (sources office, online, import; application_no from NumberingService key admission_application; B-Form hash checked against students and open applications to fill possible_duplicate_student_id; guardian CNIC hash checked against guardians to fill detected_sibling_student_id; FR-ADM-006), UpdateApplication (until decided), ReviewApplication (under_review), RecordApplicationDocument and VerifyApplicationDocument, WaitlistApplication, DecideApplication (approve with class and section offered after a seat check from SeatQuery; reject with reason; permission admission.application.approve or reject), OverrideAdmissionWindow (admission.application.override_window, reason), WithdrawApplication (applicant or staff; reason offer_lapsed for the lapse job), ConvertApplicationToStudent (permission admission.application.convert; AdmissionPaymentGate check per DEC-248; one transaction per BR-ADM-02: CreateStudent, guardians linked by CNIC hash or created (never duplicated, BR-ADM-03), FormOrJoinFamily, EnrollStudent with the offered section and chosen subjects, student_admissions type new or readmission, consent_records copied from the application, guardian verification code generated and its hash stored, guardian user per DEC-254, application admitted with converted_student_id; after commit StudentEnrolled and StudentAdmitted events).
4. Readmission path: the W-03 possible match panel offers "Readmit instead" for withdrawn or graduated students; conversion then reuses the student record and registration number and creates a new admission number (D11 11.24).
5. Public online form (SCR-ADM-010) on the tenant host, route public.admission.applications.create: privacy notice from setting privacy.notice_text with its version; guardian mobile verified by OTP before submit (OtpService with purpose admission); bot defence per D09 9.13 (honeypot field, minimum fill time, rate limit 5 per hour per IP and 3 per mobile per day); document upload limits and virus scan; consent checkboxes for SMS, WhatsApp and photo publicity (unchecked by default); reference number shown and sent by SMS. Application status page (SCR-ADM-011, public.admission.applications.status) with reference number plus mobile OTP; it shows stage and next step only.
6. Offer lapse (D11 11.15): command admission:lapse-offers daily at 09:00 organization time (fan out); reminder SMS on day 3; at admission.offer_lapse_days (default 7) the application is withdrawn with reason offer_lapsed and the seat released. Command admission:anonymize-expired monthly: applications not admitted are anonymized after admission.retention_years (default 2; FR-ADM-015) by replacing names, numbers and form data with placeholders and setting anonymized_at.
7. Document type admission_slip (printable, Admission data provider): student, class offered, registration and GR numbers, guardian names, the guardian verification code in clear text (shown only on this print; reprinting generates a new code and invalidates the old one), portal activation instructions in English and Urdu, QR to the portal sign-in.
8. Notification events: admission.application_received (applicant; contact recipient by mobile, since applicants have no user), admission.application_decided, admission.offer_reminder, admission.offer_lapsed, student.admitted (guardians of student, welcome), identity.guardian_activation (urgent and critical; OTP sign-in link). Recipient resolver application_applicant returns a contact recipient (mobile and locale).
9. Screens (D10 10.22 exactly): SCR-ADM-001 Applications (stage filter, class, date, source, age since last action), SCR-ADM-002 New application (W-03 stepper with the possible match panel), SCR-ADM-003 Application detail (stage timeline, documents, decision history, voucher status panel shown when the payment gate is active), SCR-ADM-004 Approve or reject, SCR-ADM-005 Admission window override, SCR-ADM-006 Convert to student (confirms student, guardians, family, section, subjects; fee structure section appears when Fee registers it in P06.2), SCR-ADM-007 Seats (capacity, enrolled, offered, available per class and section), SCR-ADM-008 GR register (bound-register print layout through the document engine, landscape), SCR-ADM-009 Admission form templates, SCR-ADM-010 Online application form, SCR-ADM-011 Application status.
10. Definitions: Permissions.php for admission.* per D05 5.7.7; Settings: admission.offer_lapse_days, admission.retention_years, admission.public_form_enabled (per institution), privacy.notice_text and privacy.notice_version (organization); Navigation (People group: Admissions, Seats, GR register); attention item admission.decisions_waiting (applications under review older than 3 days).
11. decisions.md rows: DEC-52, DEC-195, DEC-248, DEC-254, DEC-257.

DATABASE
admission_form_templates (aft), admission_applications (aap), admission_application_documents (aad) per D07 Part 2 7.18. Deferred FKs on student_admissions and consent_records.

BUSINESS RULES
- Conversion is one transaction; any failure leaves no student, guardian, family, enrollment or user behind (BR-ADM-02).
- Existing guardians are linked by CNIC hash, never duplicated (BR-ADM-03).
- A seat must be available at approval and again at conversion (capacity mode applies).
- Applications outside the session's admission window need an override with reason.
- Offers lapse after the configured days; lapsed offers release the seat.
- Consent defaults to not granted; it is stored with the notice version and time.
- Applications that never enrolled are anonymized after the retention period.

UI SPEC
- W-03 stepper: Student, Guardians, Previous school, Documents, Review; seats available shown next to the class.
- Possible match panel with Open record, Readmit instead and Not the same child (reason required).
- Public form is mobile-first (360px), bilingual with a language switch, large inputs, and a progress indicator; errors summarized at the top and next to fields.
- GR register print matches the bound register columns (GR number, name, father name, date of birth, admission date, class, leaving date, remarks).

SECURITY AND PERMISSIONS
- Public routes rate limited; OTP required before submit and for status checks; responses never reveal whether a B-Form or mobile exists.
- Uploaded documents stored privately; only staff with admission.application.view in the branch can open them.
- Guardian verification code stored hashed; shown in clear only on the printed slip.

TESTS
- SubmitApplication: possible duplicate set for a matching B-Form; sibling detected by guardian CNIC; online source refused without OTP verification.
- DecideApplication: seat check refuses approval in a full section in hard mode; rejection requires a reason.
- Conversion: success creates every record with one audit chain; a forced failure in EnrollStudent rolls back everything; existing guardian reused; readmission keeps the registration number and issues a new admission number.
- AdmissionPaymentGate default allows conversion; a fake gate that requires payment blocks it with a clear message.
- Offer lapse: reminder on day 3, withdrawal on day 7, seat released.
- Anonymization replaces personal data and keeps counts.
- Public form: honeypot and minimum time reject bots; rate limits; consent rows stored with notice version.
- Admission slip renders in English and Urdu with a code that verifies through VerifyGuardian; reprint invalidates the old code.
- Screen tests, audit entries, isolation suite, RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. On staging, an online application with OTP is submitted, reviewed, approved and converted; the guardian receives the activation SMS and the slip prints with a verification code.
2. A counter application for a withdrawn child offers readmission and keeps the registration number.
3. Seats and GR register screens show correct numbers after conversion.
4. CI green; staging green.

DO NOT
- Do not issue fee vouchers or require payment here (P06.3 binds the gate).
- Do not build inquiries or assessments (R2).
- Do not create guardian users with passwords or send passwords by SMS.

OUTPUT
Report per Master section 20 with the admission walkthrough, the rollback test evidence and sample slip PDFs.
```

### P04.3 Guardian and Student Portals

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D04 4.9 (portals), D05 5.4 (GDN, STD roles) and 5.9 (student request types), D10 Part 1 (guardian and student homes), D10 Part 2 SCR-IAM-002, 003, 007 to 011, SCR-APR-003, SCR-APR-009, SCR-NTF-001, 002, SCR-STU-019, 024 to 026, SCR-GRD-010, 011; D11 Part 1 11.4 (student_profile_correction, student_modification, document_verification); D12 Part 1 DEC-188, DEC-189; D02 FR-GRD-007, FR-GRD-008; D14 DEC-247, DEC-250, DEC-254, DEC-255.

PHASE GOAL
Open the guardian and student portals: OTP activation, one account per mobile across children and branches, the child switcher, portal access rules, profile and request screens, the portal versions of identity self-service, notifications and approvals, the portal switcher for staff who are also guardians, guardian and student homes through the new HomeSectionRegistry, student login creation, and the student request approval types.

ALREADY BUILT
PH-00 to PH-03, P04.1 and P04.2: students, guardians, families, admission and conversion with guardian users in pending_activation, PortalAccessResolver registry from PH-01 (guardian and student deny until now).

TASKS
1. PortalAccessResolver implementations: GuardianPortalAccess (user has at least one active guardian row with an active link where has_portal_access = 1 and the guardian is not custody restricted; read-only mode for 1 year after the last linked child withdraws or graduates, A-38) and StudentPortalAccess (user linked to a student whose status is active and whose current level allows login per DEC-255; read-only for 1 year after withdrawal or graduation).
2. Activation: OTP request (SCR-IAM-002) recognizes guardian mobiles whose user is pending_activation or active; OTP verify and set password (SCR-IAM-003) with the guardian and student minimum length 8 (DEC-188); same response whether or not the number exists. Users with several guardian rows sharing one mobile get one account (DEC-22); LinkGuardianUser runs on conversion, import and guardian edits when the mobile matches an existing guardian user.
3. Child context: middleware EnsureChildContext for guardian routes carrying {student:uuid}; the student must be in the user's portal-enabled children or the request gets 404. Child switcher drawer SCR-GRD-010 lists all linked, portal-enabled children across branches (US-GRD-01) and stores the last chosen child.
4. Visibility for portals: AssignedScopeRegistry resolvers for guardian.* routes (linked children with portal access) and student.* routes (self); BR-GRD-01.
5. Student logins (DEC-255): bulk action "Create student logins" on SCR-STU-020 (preview count by class, then bulk operation creating users with username = registration number, status pending_activation, role STD scoped to the branch); guardian portal action "Child login setup code" (one-time code valid 24 hours, single use, audited) used on the student sign-in activation page; student OTP to student_mobile_e164 when present.
6. Approval types (Student registers in Definitions/ApprovalTypes.php with handlers): student_profile_correction (protected or identity fields; steps 1 guardian_of_student when raised by the student, skipped for students aged 18 or older or in a college program; 2 class_teacher_of_student; 3 principal_of_branch when a protected field is involved, as a condition on the step; handler applies through UpdateProtectedStudentField or UpdateStudent; snapshot of the student version), student_modification (non-protected contact fields: address, city, student mobile, email, emergency contacts; same steps without step 3), document_verification (guardian uploads; step 1 named_role admission_officer of branch, any; approve permission student.document.verify; handler marks the document verified). Approval resolvers registered: guardian_of_student (linked guardians with can_approve_requests = 1, excluding custody restricted) and class_teacher_of_student (through ClassTeacherQuery; empty until P05.1, so the step escalates per BR-APR-04). academic_correction is registered in P07.2 with Examination.
7. Portal variants of shared screens (same controllers and Actions, portal-specific views): SCR-IAM-008 My profile, SCR-IAM-009 Change password, SCR-IAM-010 Two-factor setup (optional for guardians and students), SCR-IAM-011 My sessions and devices, SCR-NTF-001 Notification centre, SCR-NTF-002 Notification preferences (consent for SMS and WhatsApp shown here and written to consent_records per DEC-257), SCR-APR-003 My requests (guardian and student), SCR-APR-009 Guardian requests (own requests and child requests awaiting the guardian's approval).
8. Portal switcher SCR-IAM-007 (DEC-47): shown to users holding both staff and guardian access; switching regenerates the session and changes the portal host path; the guardian portal shows "Switch to staff" only when staff access is still valid.
9. Session lifetimes per DEC-189: guardian and student idle 7 days with "Remember this device" up to 30 days; staff unchanged.
10. HomeSectionRegistry in Reporting (DEC-250) with sections ordered by weight and permission-checked; Student and Guardian contribute: guardian.home children cards (photo, class, section, links to profile), pending requests count, "Complete your profile" prompt; student.home profile card and pending requests. Later modules add attendance, fees, results and diary sections.
11. Screens: SCR-STU-019 Profile change requests (staff list, links to approval detail), SCR-STU-024 Student portal: my profile, SCR-STU-025 Guardian portal: child profile (request a change, upload document), SCR-STU-026 Portal: request profile change (field, new value, document), SCR-GRD-010 Guardian portal: child switcher, SCR-GRD-011 Guardian portal: pickup persons (add with photo and CNIC; staff review flag; created_via guardian).
12. Broadcast audiences: guardians by branch, class or section; students by branch, class or section (SCR-NTF-009).
13. Definitions: Settings student.portal_levels (branch), guardian.portal_readonly_days (default 365); Navigation for guardian and student portals (Home, Children, Requests, Notifications, Profile); portal role templates GDN and STD baseline permissions per D05.
14. decisions.md rows: DEC-22, DEC-47, DEC-189, DEC-250, DEC-254, DEC-255.

DATABASE
No new tables. Uses users, guardians, guardian_students, students, consent_records, approval tables, notification_preferences.

BUSINESS RULES
- A guardian sees only children with an active link that has portal access (BR-GRD-01); custody-restricted guardians are denied.
- One user per mobile for guardians; all related guardian rows share it (DEC-22).
- Student requests raised by a student under 18 in a school level need guardian approval first.
- Protected field changes always end with the principal step.
- Guardians cannot change protected fields directly; every change is a request.

UI SPEC
- Portals are mobile-first (360px) with bottom navigation (Home, Children, Requests, Notifications, Profile) per D09 portal layout; Urdu default when the user's locale is ur.
- Child switcher shows photo, name, class and branch; the current child is shown in the header on every page.
- Request forms show which approvers will review in order.

SECURITY AND PERMISSIONS
- Every guardian route checks EnsureChildContext and the policy; cross-child access returns 404.
- Setup codes and OTPs are single use, rate limited, and never logged.
- Portal sessions regenerate on login and portal switch.

TESTS
- Access: guardian with portal access sees the child; link without portal access denied; custody-restricted guardian denied and sessions ended; withdrawn child read-only within 365 days and denied after.
- One mobile shared by mother and father records yields one user that sees all their children across two branches.
- EnsureChildContext: another family's child uuid returns 404.
- Student logins: bulk creation for an allowed level only; setup code works once and expires.
- Approval types: student request goes guardian, class teacher (escalates while empty), principal for protected fields; student aged 18 skips the guardian step; guardian approval requires can_approve_requests.
- Portal switcher regenerates the session; idle timeouts per DEC-189.
- Home sections render only with permission.
- Screen tests for every portal variant, audit, isolation suite, RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. On staging, a guardian activates by OTP, switches between two children in different branches and raises a profile correction that reaches the principal.
2. A student in an allowed level activates with a guardian-generated setup code.
3. A staff user who is also a guardian switches portals both ways.
4. CI green; staging green.

DO NOT
- Do not show attendance, fee, result or diary sections yet; those modules add them.
- Do not let guardians or students edit records directly.
- Do not build messaging or online payment (R2).

OUTPUT
Report per Master section 20 with portal screenshots at 360px in English and Urdu and the access rule test matrix.
```

### P04.4 Student, Guardian and Teaching Group Imports

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D11 Part 2 11.28; D10 Part 2 W-05, SCR-IMX-002, SCR-ACD-016; D06A A.7 (option blocks and teaching groups) and A.11; D07 Part 2 teaching_group_members; D05 5.7.8, 5.7.9, 5.7.23; D12 Part 2 performance budgets.

PHASE GOAL
Move existing schools in quickly and safely: student import with guardians and legacy numbers, guardian import, teaching group membership with "generate from option blocks", and close PH-04 with performance and masking evidence.

ALREADY BUILT
PH-00 to PH-03, P04.1 to P04.3: students, guardians, families, enrollments, admission, portals, import engine from P02.3.

TASKS
1. Migration (Student, which owns the table per the D07 Part 3 DEC-108 correction): teaching_group_members exactly as D07 Part 2 7.17 (uq_tgm_one_group, ix_tgm_group).
2. Actions (Student): AddTeachingGroupMember (student must have an active student_subjects row for the group's subject in the same class; capacity warning; one active group per student, session and subject), MoveTeachingGroupMember (ends the old row with left_on and starts a new one), RemoveTeachingGroupMember (left_on), GenerateTeachingGroupsFromOptionBlocks (for a class and a bucket with a timetable block label: one group per subject chosen in that bucket, students assigned by their choices, groups split when capacity is exceeded with names {subject code}-{n}; preview then bulk operation; existing memberships kept).
3. Listeners: StudentSubjectsChanged ends memberships for dropped subjects; EnrollmentClosed ends all memberships of that enrollment.
4. Import type students (permission student.student.import, D05 5.7.8): columns branch code, session name, level code, group code, section name, name, name_ur, father_name, father_name_ur, mother_name, date_of_birth (date format option), gender, b_form, category code, first_admitted_on, gr_no (optional legacy), admission_no (optional legacy), roll_no (optional), student_mobile, address, city, elective subject codes (semicolon list), guardian 1 (name, relation, cnic, mobile, is_fee_payer), guardian 2 (optional), consent flags (optional; recorded_via import). Commit through ImportStudentWithGuardians (one row per transaction inside the chunk) calling CreateStudent, NumberingService::claim for legacy numbers, EnrollStudent, CreateGuardian or reuse by CNIC hash, LinkGuardianToStudent, FormOrJoinFamily, guardian user per DEC-254 (activation SMS sent after the whole import completes, as one broadcast-style batch respecting quiet hours). Duplicate detection: B-Form hash in file and database; same name, father name and date of birth flagged as a warning.
5. Import type guardians (permission guardian.guardian.import): name, name_ur, cnic, mobile_primary, mobile_secondary, email, occupation, address, city, link to student registration number with relation and flags. Commit through CreateGuardian or update mode for contact fields by CNIC (declared update mode).
6. Import type teaching_group_members (permission academic.teaching_group.update): class name, group name, student registration number; commit through AddTeachingGroupMember.
7. Screens: SCR-ACD-016 members tab (add, move, remove; filters by section; capacity bar) and the "Generate from option blocks" action with preview; Students list import button opening SCR-IMX-002 with type students; Guardians list import button.
8. Performance work for the 5,000-row preview: chunked reads of 500 rows, hash lookups batched with whereIn on lookup hashes, class and section maps cached per job, no per-row queries for reference data; group performance test in tests/Performance/StudentImportPreviewTest with the S1 profile.
9. Masking sweep: a dedicated test file tests/Feature/Masking/PeopleMaskingTest covering SCR-STU-001, 002, 023, SCR-GRD-001, 002 and the student and guardian exports for roles without core.pii.view_full.
10. decisions.md rows: DEC-81, FR-GEN-014 alignment.

DATABASE
teaching_group_members (tgm) per D07 Part 2 7.17, owned by Student (D07 Part 3 DEC-108).

BUSINESS RULES
- Imports run every row through the same Actions as forms (D11 11.28); legacy numbers cannot duplicate existing ones.
- A student sits in at most one active teaching group per subject in a session.
- Generated groups never remove existing members; they only add students without a group.
- Guardian activation SMS go out once per guardian after the import finishes.

UI SPEC
- Members tab shows section badges because Cambridge sets mix sections.
- Generate preview lists groups to create, member counts per group and students left without a group with reasons.
- Import validation preview follows W-05; warnings for likely duplicates are reviewable before commit.

SECURITY AND PERMISSIONS
- Import permissions per type; uploaded files private; error files contain masked identity numbers (last4 only).
- Teaching group changes audited; members visible to teachers of that group after P05.1.

TESTS
- Students import: 50 rows with guardians create students, guardians (shared CNIC reused once), families, enrollments and users; legacy GR numbers claimed and the counter advanced; duplicate GR refused; B-Form duplicate in file and database flagged; error file masks identity numbers.
- Guardians import with update mode changes contact fields only.
- Teaching group membership: subject not taken refused; second active group for the same subject refused at the database level; move keeps history.
- Generate from option blocks: 3 subjects in a bucket with capacity 20 and 50 students produce correct groups; rerun adds no duplicates.
- Listeners end memberships on subject drop and enrollment close.
- Performance: 5,000-row student preview under 2 minutes on S1 (evidence in report).
- Masking sweep passes.
- Screen tests, audit, isolation suite, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. A 5,000-row student file with guardians previews in under 2 minutes and commits on staging; activation SMS batch sent after completion.
2. A Cambridge O Level class gets teaching groups generated from option blocks with correct members.
3. PH-04 exit criteria met; MS-05 evidence and D13 13.6 gate evidence attached.
4. CI green; staging green.

DO NOT
- Do not import attendance, fees, marks or staff here.
- Do not bypass Actions for speed; optimize reads, not writes.
- Do not send one SMS per row during the import.

OUTPUT
Report per Master section 20 with import timings, the generated groups example and the masking sweep results. Then run the D13 13.6 gate for PH-04.
```

---

## 14.12 PH-05 HR, Attendance and Leave

**Exit criteria:**
- MS-06 is reached.
- A 40-student section saves in under 1 second.
- Duplicate punches don't create duplicate rows.
- A locked month rejects edits unless an unlock has been approved.

### P05.1 HR Core, Teaching Assignments and Class Teachers

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 2 7.21 (HR R1 tables) and 7.17 (teaching_assignments, section_class_teacher_histories, DEC-106); D07 Part 3 correction for department_heads; D05 5.7.6 (teaching assignment, workload) and 5.7.10; D10 Part 2 10.25 and SCR-ACD-010, 011, 013, 014, 016, 017; D11 Part 1 11.4 (employee_profile_correction); D11 Part 2 11.30 (staff onboarding); D06 6.7 (workload limits); D02 FR-HR; D14 DEC-245, DEC-246, DEC-247.

PHASE GOAL
Build the R1 staff record (designations, employees, branch assignments, salary history, qualifications, experience, documents, department heads, staff user linking, staff import) and complete the academic links that needed employees: teaching assignments, class teachers with history, workload, "My classes", real assigned visibility for teachers and HODs, the session copy step and the reporting manager and HOD resolvers.

ALREADY BUILT
PH-00 to PH-04: organization structure with departments and branches (head and principal columns without FK), identity with CreateStaffUser and the subset rule, academic structure with sections (class_teacher_employee_id without FK) and teaching groups with members, students and guardians, portals, ClassTeacherQuery and TeacherAssignmentQuery with empty bindings (DEC-247), SessionCopyStepRegistry (DEC-246).

TASKS
1. Migrations (HR): designations, employees, employee_branch_assignments, employee_salary_histories, employee_qualifications, employee_experiences, employee_documents exactly as D07 7.21; department_heads [T-] alias dhd exactly as D07 Part 3 7.25 (uq_dhd). There is no departments.head_employee_id and no branches.principal_employee_id (both removed by D07 Part 3 DEC-108): the department head comes from department_heads and the principal is the holder of the principal role in the branch.
2. Migrations (Academic, DEC-245): teaching_assignments and section_class_teacher_histories exactly as D07 7.17 (ck_tas_target, uq_tas_primary, generated keys); FK sections.class_teacher_employee_id -> employees (composite).
3. Models: Employee (BelongsToOrganization; visibility by primary branch and active branch assignments), EmployeeBranchAssignment (BelongsToBranch), the T- child models, TeachingAssignment and SectionClassTeacherHistory (BelongsToBranch); Auditable; morph aliases employee, designation, teaching_assignment.
4. Actions (HR): CreateEmployee (employee_code from NumberingService key employee; CNIC hash uniqueness uq_emp_cnic; primary branch assignment created; optional linked user through Identity CreateStaffUser with the role suggested by designation.suggested_role_slug, subject to the subset rule DEC-55), UpdateEmployee, ChangeEmployeeStatus (active, on_leave, suspended, resigned, terminated, retired; separated statuses need separated_on and a reason; hr:apply-separations daily deactivates the linked user and ends branch assignments on that date; EmployeeSeparated event), AddBranchAssignment and EndBranchAssignment (one active primary per uq_eba_primary), RecordSalaryChange (effective dated, sensitive, DEC-103), Add/Update/Remove qualification and experience rows, UploadEmployeeDocument (expiry date), LinkEmployeeUser and UnlinkEmployeeUser (one user per employee, uq_emp_user), SetDepartmentHead (ends the previous head row and writes the new one; DepartmentHeadQuery is the public read path used by SCR-ORG-009 and the resolvers).
5. BranchPrincipalQuery (Identity public query): returns the users holding the principal role in a branch; HR maps them to employees for display. No column stores the principal (D07 Part 3 DEC-108).
6. Actions (Academic): AssignTeacher (session active or planning; target is a section or a teaching group, never both; subject is a class subject of that class; employee active, is_teaching, with an active assignment to the branch; role primary or co_teacher; a new primary ends the previous primary with ends_on; workload warning when periods exceed employees.max_periods_per_week, default 36), EndTeachingAssignment (sets ends_on; rows never deleted), AssignClassTeacher (DEC-75 setting academic.one_section_per_class_teacher, default on, refuses a second section in the same session; writes section_class_teacher_histories, ends the previous row, updates sections.class_teacher_employee_id).
7. Rebind DEC-247 contracts: ClassTeacherQuery reads sections.class_teacher_employee_id -> employees.user_id; TeacherAssignmentQuery reads active teaching_assignments for the user's employee plus class teacher sections. AssignedScopeRegistry resolvers: academic.section, academic.teaching_group, academic.teaching_assignment (TCH own), student.student and guardian.* now return real data; HOD·a scopes for hr.employee (employees of departments the user heads) and academic.subject and academic.teaching_assignment.assign (subjects of those departments).
8. Session copy step "Teaching assignments and class teachers" registered in SessionCopyStepRegistry: copies active primary assignments and class teachers to the target session by matching class, section and group names; unmatched rows listed in the preview.
9. Approval resolvers registered by HR: reporting_manager (employee's reporting_manager_employee_id user; falls back to the head of the employee's department) and hod_of_subject (head of the subject's department). Approval type employee_profile_correction (D11 11.4: raised from SCR-IAM-008 for employee fields; 1 hr_manager_of_branch; handler updates the employee through UpdateEmployee; employee version snapshot).
10. Commands: hr:document-expiry-reminders daily (documents and CNIC expiring within 30 days; event hr.document_expiring to HRM and the employee; FR-HR-007), hr:apply-separations daily.
11. Employee profile shell with EmployeeProfileTabRegistry: HR contributes Personal, Assignments, Qualifications, Documents, Salary (sensitive) tabs; Academic contributes Teaching load; Attendance and Leave add theirs later.
12. Screens (D10 10.25 R1 rows exactly): SCR-HR-001 to SCR-HR-013 (Employees, Employee profile, Employee form with CNIC duplicate check and optional user creation, Employee status change, Salary history, Branch assignments, Qualifications and experience, Employee documents, Designations, Staff directory, Profile corrections, Department heads, Import employees). Academic: SCR-ACD-013 Teaching assignments (section by subject matrix; unassigned cells highlighted; assign needs academic.teaching_assignment.assign), SCR-ACD-014 Teacher workload (periods against limit, over-limit highlighted), SCR-ACD-017 My classes (sections and groups with quick links; attendance link active after P05.2), class teacher column with history drawer on SCR-ACD-011, teachers column on SCR-ACD-010, teacher field on SCR-ACD-016.
13. Import types: employees (permission hr.employee.import; columns employee code optional, name, name_ur, father name, gender, CNIC, mobile, branch code, department code, designation code, joining date, employment type, is_teaching, create user flag with role slug), teaching_assignments (permission academic.teaching_assignment.assign; session, class, section or group, subject code, employee code, role, periods).
14. Seed data: default designations (principal, vice_principal, teacher, senior_teacher, coordinator, accountant, cashier, clerk, receptionist, librarian, lab_assistant, driver, guard, peon) with suggested role slugs, editable.
15. Definitions: Permissions.php for hr.* per D05 5.7.10 (hr.salary and hr.document sensitive where marked); Settings academic.one_section_per_class_teacher, hr.default_max_periods (36); Navigation (People group: Staff, Directory; Academics group: Teaching assignments, Workload; My work: My classes).
16. decisions.md rows: DEC-75, DEC-103, DEC-245 (completion), DEC-247 (rebinding).

DATABASE
designations (dsg), employees (emp), employee_branch_assignments (eba), employee_salary_histories (esh), employee_qualifications (eql), employee_experiences (eex), employee_documents (edc), department_heads (dhd), teaching_assignments (tas), section_class_teacher_histories (sct) per D07 Part 2 and D07 Part 3 7.25; FK on sections.class_teacher_employee_id.

BUSINESS RULES
- One active primary branch per employee; shared staff have extra assignments (D04 4.11).
- One active primary teacher per section or group and subject (uq_tas_primary); assignments end, never delete.
- One section per class teacher per session when the DEC-75 setting is on.
- Only active teaching employees assigned to the branch can teach there.
- Salary data is sensitive and visible only with hr.salary.view.
- Separated employees lose user access on the separation date.

UI SPEC
- Teaching assignment matrix: rows sections and groups, columns subjects, cells show teacher initials and periods; empty cells use the warning tint; clicking a cell opens the assign drawer with workload shown for each candidate.
- Employee profile tabs per registry; CNIC masked; salary tab shows a lock icon for users without permission.
- My classes cards are 360px friendly with links to attendance, students and (later) marks and diary.

SECURITY AND PERMISSIONS
- Routes and permissions per D10 10.25 and 10.21; HOD visibility limited to their departments.
- Salary and documents audited on view (low-risk view events through the queue, BR-AUD-03).
- Creating linked users respects the subset rule; roles above the creator are refused.

TESTS
- CreateEmployee: CNIC duplicate refused; employee code generated; user created with the suggested role only when the creator may grant it.
- Separation: user deactivated on separated_on by the daily command; EmployeeSeparated fired.
- Teaching assignments: both targets set refused by ck_tas_target; second primary for the same subject ends the previous one; non-teaching or other-branch employee refused; workload warning above the limit.
- Class teacher: second section in the same session refused with the setting on and allowed with it off; history rows kept.
- Visibility: a teacher now sees exactly the students of 6-A after assignment and gets 404 for 6-B (AC-STU-02 with real data); HOD sees only employees of their department.
- Approval resolvers: reporting_manager falls back to the department head; class_teacher_of_student now resolves and a student_profile_correction stops at the class teacher.
- Session copy step copies assignments by name match and lists unmatched rows.
- DepartmentHeadQuery returns the current head and the history is kept.
- Screen tests, audit, isolation suite, RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. On staging, employees are imported, teachers assigned through the matrix, class teachers set, and a teacher sees "My classes" with only their students.
2. A guardian's profile correction now routes to the real class teacher.
3. Workload report flags an over-limit teacher.
4. CI green; staging green.

DO NOT
- Do not build payroll, contracts, performance reviews, training, disciplinary actions or separations workflow (R2).
- Do not compute attendance or leave here.
- Do not delete teaching assignments or class teacher history.

OUTPUT
Report per Master section 20 with the visibility evidence (teacher and HOD), the matrix screenshot and the list of resolvers now active.
```

### P05.2 Student Attendance

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D11 Part 2 11.16 and DEC-179, DEC-180 completely, and the 11.14 correction that adds section_attendance_sheets; D07 Part 2 7.22 (attendance_policies, student_attendances, student_attendance_revisions, attendance_corrections) with DEC-24, DEC-25, DEC-73, DEC-105; D10 Part 2 10.26 (SCR-ATT-001, 002, 004 to 007, 016, 024, 025), DEC-162 and wireframe W-01; D11 Part 1 11.4 (attendance_correction); D05 5.7.11; D02 FR-ATT; D14 DEC-249, DEC-251.

PHASE GOAL
Let class teachers mark daily attendance in under a minute on a phone: sheets prepared each morning with pre-filled statuses, a per-row autosaving grid with conflict handling, submit and finalization, delayed absence alerts, not-marked alerts, corrections through approval with revision history, registers and portal views.

ALREADY BUILT
PH-00 to PH-04 and P05.1: students with enrollments, guardians with recipient resolvers, class teachers and teaching assignments, approval and notification engines, WorkingDayCalendar default binding (create it in Core in this step if P01 did not).

TASKS
1. Migrations (Attendance): attendance_policies (both person types; staff fields used in P05.3), section_attendance_sheets (D11 11.14: section_id, attendance_date, status not_marked, draft, submitted, finalized, submitted_by, submitted_at, finalized_by, finalized_at, version; uq_sas (organization_id, section_id, attendance_date)), student_attendances, student_attendance_revisions, attendance_corrections exactly as D07 7.22, with student_attendances.leave_application_id replaced by soft source_ref_type and source_ref_id (D07 Part 3 DEC-108). student_period_attendances is built in P09.1 (DEC-251).
2. Core WorkingDayCalendar per DEC-249 (default binding from shifts.working_days); AttendancePrefillProviderRegistry in Attendance: providers return statuses per student and date with a reason (Leave registers LV in P05.4; Student registers SU in P08.2; Administration registers H for closures in P09.2). Weekend days use W.
3. PolicyResolver: picks the effective student policy by branch, shift and date (branch and shift override first, then organization default); fields: start_time, grace_minutes, late_after_minutes, marking_window_ends_at, count_suspended_in_percentage, official_sources.
4. Command attendance:prepare-sheets at 06:00 organization time (fan out per branch): for each active section on a working day creates the sheet (not_marked) and rows for active enrollments with NM or a prefill status from the registry; idempotent; on a non-working day creates no sheets.
5. Actions: MarkStudentAttendance (one row per call; idempotency key per row change; sheet version check per DEC-162; returns the new sheet version; refused after finalization or in a locked period through PeriodLockService; writes a revision row for each saved change, DEC-105; source teacher or admin by role), MarkAllPresent (only NM rows), SubmitSectionSheet (no NM rows left; status submitted; SectionSheetSubmitted event), FinalizeSectionSheet (principal or the window command; draft and submitted become finalized; not_marked stays not_marked; NM rows stay NM, DEC-180; AttendanceFinalized event), RequestAttendanceCorrection (permission attendance.correction.create; payload students, date, new status, reason, evidence; SubmitApprovalRequest with type attendance_correction).
6. Approval type attendance_correction (Attendance registers): steps per D05 5.9 (1 principal or vice principal of branch, mode any); handler ApplyAttendanceCorrection writes a revision with source correction and the correction id for each student whose row version still matches; changed rows are skipped and listed; all rows changed means apply_failed; guardian SMS when the status changes to or from A.
7. Absence alerts (DEC-179): on SectionSheetSubmitted and on finalization, job SendAbsenceAlerts delayed attendance.absence_alert_delay_minutes (default 10) on queue attendance; at send time it re-reads each row and sends only for students still A; never for LV, H, SU, W. Notification event attendance.student_absent (mandatory, not urgent; recipients guardians_of_student; English and Urdu templates with child name, date, class; no identity numbers).
8. Commands: attendance:finalize-windows every 15 minutes (policy window passed -> finalize), attendance:not-marked-alerts every 15 minutes during school hours (class teacher at window minus 30 minutes, principal after the window plus 30 minutes; attention item attendance.sheets_not_marked), attendance:absence-streaks daily after finalization (three consecutive A raise an attention item for the class teacher and, when attendance.streak_sms_enabled, event attendance.consecutive_absence to guardians).
9. Device punches for students (gate devices) are shown in the grid as "arrived 7:52" when attendance_punches rows exist for the student (table built in P05.3; the grid hides the column until then); a device punch never overwrites a teacher mark (DEC-25).
10. Screens (D10 10.26): SCR-ATT-001 Section sheets (today's sections with status pills and window countdown; class teachers see their sections, principals the branch), SCR-ATT-002 Mark section attendance (W-01; Alpine grid with 44px buttons, keyboard support, live counts, per-row autosave with idempotency keys, row-level conflict banner, offline banner and retry per D09 9.16), SCR-ATT-004 Student attendance register (month grid by section with totals and percentage; print register layout through the document engine), SCR-ATT-005 Day detail drawer (source, punches, revisions with who and why), SCR-ATT-006 Corrections, SCR-ATT-007 Request correction, SCR-ATT-016 Attendance policies (both person types; effective dated), SCR-ATT-024 Guardian portal: attendance (month calendar with codes and arrival times; guardian.home section "Today" per child through HomeSectionRegistry), SCR-ATT-025 Student portal: my attendance.
11. AttendancePercentageQuery (public): per enrollment and date range, excluding H, W and NM, treating SU per policy, LV counted as present or excluded per setting attendance.leave_counts_as (default excluded); used by profile tabs, reports and later by results.
12. Profile tabs: Attendance tab on the student profile (month calendar and percentage).
13. Definitions: Permissions.php for attendance.student_attendance, attendance.correction, attendance.policy per D05 5.7.11; Settings attendance.absence_alert_delay_minutes, attendance.streak_sms_enabled, attendance.leave_counts_as; Navigation (Attendance group: Today's sheets, Register, Corrections, Policies; My work: Mark attendance).
14. decisions.md rows: DEC-24, DEC-25, DEC-73, DEC-105, DEC-162, DEC-179, DEC-180, DEC-249, DEC-251.

DATABASE
attendance_policies (apl), section_attendance_sheets (sas), student_attendances (sat), student_attendance_revisions (sar), attendance_corrections (acr) per D07 Part 2 7.22 and the D11 11.14 correction.

BUSINESS RULES
- One daily row per student per date (uq_sat); every saved change writes a revision (DEC-105).
- Source precedence: correction over teacher or admin over device or QR over automatic (DEC-25).
- After finalization every change is a correction through approval; NM never becomes A automatically (DEC-180).
- Absence alerts wait 10 minutes and are cancelled when the student is no longer A (DEC-179).
- Locked periods refuse all writes (FR-GEN-015).
- Class teachers mark their sections; principals and vice principals may mark any section of their branch as admin source.

UI SPEC
- W-01 on 360px: header with section, date and autosave status; live counts; All present; student rows with initials avatar, name, status buttons and remark; approved leave rows locked with the reason; Submit disabled until no NM rows remain.
- Desktop keyboard: arrows move, P, A, L, H set codes, Enter submits.
- Status colors per D09 9.5 attendance tokens; codes always shown as text, never color only.

SECURITY AND PERMISSIONS
- attendance.student_attendance.mark with assigned visibility for class teachers; branch visibility for PR and VP.
- Every mark and correction audited; corrections carry the approval reference.
- Guardian and student portal views limited by EnsureChildContext and self scope.

TESTS
- prepare-sheets: sheets only on working days; prefill providers applied (fake LV provider); idempotent rerun.
- Marking: idempotent repeat of the same key changes nothing; version conflict on a changed row returns the conflict for that row only; finalized sheet refuses marks; locked period refuses marks.
- Submit refused with NM rows; finalize keeps NM and not_marked.
- Alerts: submit queues delayed jobs; changing A to P within 10 minutes prevents the SMS; leave and holiday rows never alert.
- Correction: approve applies revisions; a row changed after submission is skipped and listed; all changed gives apply_failed; requester cannot approve (SOD).
- Percentage query excludes H, W, NM and follows the leave setting.
- Performance: saving all 40 rows of a section completes in under 1 s total on S1 and each row save stays within assertQueryCountAtMost(12).
- Screen tests including portal views, audit, isolation suite, RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. On staging, a class teacher marks a 40-student section on a phone, submits, and an absent child's guardian receives the Urdu SMS after the delay.
2. A correction after finalization goes through approval and shows in the day detail revisions.
3. The register prints a month grid with percentages.
4. 40-student save timing evidence under 1 s.
5. CI green; staging green.

DO NOT
- Do not build period attendance (P09.1) or staff attendance (P05.3).
- Do not turn NM into A automatically.
- Do not send an SMS on each tap.

OUTPUT
Report per Master section 20 with the marking timing evidence, the alert delay test, and W-01 screenshots in English and Urdu.
```

### P05.3 Staff Attendance and Devices

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D11 Part 2 11.17 completely; D07 Part 2 7.22 (attendance_devices, device_enrollments, attendance_punches with DEC-48 and DEC-104, staff_attendances, staff_attendance_revisions, qr_attendance_sessions, device_sync_logs) and DEC-29; D03 adapters and DEC-42 (webhooks stored first); D04 4.11 (shared staff); D10 Part 2 10.26 (SCR-ATT-008 to 015, 017 to 022) and wireframe W-06; D12 Part 1 12.7 and 12.8 (webhooks, devices); D05 5.7.11; D14 DEC-252, DEC-253.

PHASE GOAL
Collect staff punches from every realistic source (ZKTeco ADMS push, signed webhooks, CSV files, online check-in on trusted devices inside the geofence, rotating QR at reception), store them once, resolve them into daily staff attendance with late, half day and early departure rules, handle missing punches and administrative entries, monitor devices and flag suspicious punches.

ALREADY BUILT
PH-00 to PH-04, P05.1 and P05.2: employees with branch assignments and shifts, attendance_policies, approval and notification engines, identity trusted devices (P01.4), branch geofence fields (SCR-ORG-006), webhook_receipts (P02.2).

TASKS
1. Migrations (Attendance): attendance_devices, device_enrollments, attendance_punches (partition-ready: PK (id, punched_at), no partitions created yet; partitioning is switched on in P09.4), staff_attendances, staff_attendance_revisions, qr_attendance_sessions, device_sync_logs ([L]) exactly as D07 7.22 (staff_attendances uses soft source_ref_type and source_ref_id instead of leave_application_id, D07 Part 3 DEC-108) with the DEC-252 additions (attendance_punches.source gains admin; verify_type gains manual) and attendance_suspicious_alerts per DEC-252.
2. Device hosts and middleware groups adms and webhook per DEC-253; DeviceResolver loads the device by serial (global unique) and sets the tenant context; unknown or retired serials get a plain 404 and an integration log entry with the serial only; per-device IP allowlist when set.
3. ZKTeco ADMS adapter (Modules\Attendance\Adapters\Adms): handshake GET /iclock/cdata?SN={serial}&options=all returns the device options block (stamps, Delay, TransInterval, Realtime, TimeZone from the device record); POST /iclock/cdata?SN={serial}&table=ATTLOG accepts tab-separated punch lines (PIN, local datetime, status, verify type, work code), tolerant of firmware variants; GET /iclock/getrequest is the heartbeat and returns queued commands or OK; POST /iclock/devicecmd records command results. Every request is stored first in webhook_receipts (DEC-42) and processed by job ProcessAdmsReceipt on queue attendance; the HTTP reply returns OK: {count} after storage, before processing. Device local time converted to UTC with the device timezone; clock drift computed from the handshake and logged in device_sync_logs. Fixtures from at least two firmware samples in tests/Fixtures/adms.
4. Signed webhook adapter: POST devices.webhook.punches with JSON punches; header X-Signature HMAC-SHA256 over timestamp and raw body with the device secret; timestamp within 5 minutes; nonce stored to refuse replays; stored first, then processed.
5. CSV and Excel punch import: import type attendance_punches (permission attendance.punch_import.import; columns device serial or branch code, device user id or employee code, datetime, direction optional) committing through RecordPunch; SCR-ATT-020 opens SCR-IMX-002 with this type.
6. RecordPunch action (single entry point for every source): resolves person from device_enrollments (unmapped punches stored with person NULL and shown on SCR-ATT-019), computes punched_local_date in organization timezone, inserts with uq_apn so duplicates are dropped silently, marks the employee and date dirty for resolution (Redis set attendance:dirty:{org}).
7. Online check-in (SCR-ATT-012): requires a trusted device (Identity); browser geolocation; server computes haversine distance to the branch geofence; setting attendance.outside_geofence_mode reject (default: attempt refused and audited) or flag (punch stored with geofence_result outside and an outside_geofence alert); works on 360px.
8. QR attendance: StartQrSession (kiosk, purpose staff_kiosk, secret encrypted, rotation_seconds 30); kiosk screen SCR-ATT-014 shows a code computed as HMAC(secret, floor(now / rotation)) with the session id, refreshed every 30 s, kiosk mode hides navigation and auto-locks after attendance.kiosk_idle_minutes (default 10); staff scan on SCR-ATT-013 (camera via a small JS QR library bundled by Vite, CSP-safe); server accepts the current and previous step only, one punch per person per session per direction.
9. Resolution: ResolveDailyStaffAttendance (per employee and date: official sources from the policy only; punches inside attendance.collapse_seconds (default 120) marked is_duplicate; first in and last out; late if first in is after shift start plus grace; half day after half_day_after_minutes; early departure before the threshold; worked minutes; branch of the first check-in for shared staff, D04 4.11; precedence correction over admin over device, QR and online over automatic, DEC-25; never overwrites administrative entries or corrections; revision row for every change; refused for locked periods). Commands: attendance:resolve-punches every 5 minutes (dirty set only), attendance:auto-absent every 15 minutes (after auto_absent_after with no punch, no leave and a working day -> A, source system), attendance:device-health every 15 minutes (no heartbeat for 30 minutes -> event attendance.device_offline to ITA and HRM, once per outage), attendance:missing-out-punch daily at 23:30 (attention item for the employee and HRM).
10. Suspicious alerts (DEC-252): outside_geofence (flag mode), impossible_travel (punches at two branches closer in time than attendance.min_travel_minutes, default 20), clock_drift (above 5 minutes at handshake). Review actions dismiss and confirm with a note.
11. Missing punch: approval type missing_punch (Attendance registers; payload date, in or out time, reason; step 1 hr_manager_of_branch; handler adds a punch with source admin, verify_type manual and re-resolves the day; version check on the staff attendance row). Administrative entry (SCR-ATT-010): RecordAdministrativeEntry sets the day for official duty or training with status P, source admin and a remarks category, with reason.
12. Device management: CreateDevice and UpdateDevice (serial global unique; adapter; purpose; timezone; secrets encrypted, shown once, rotatable; IP allowlist), RetireDevice, EnrollDeviceUser and EndDeviceEnrollment (mapping device user id to an employee or a student; mapping re-resolves held punches), listener on EmployeeSeparated ends device enrollments.
13. Screens (D10 10.26 exactly): SCR-ATT-008 Staff attendance today (present, late, absent, leave, missing out-punch; filters by department), SCR-ATT-009 Staff attendance register (month grid per employee), SCR-ATT-010 Administrative entry, SCR-ATT-011 My attendance (W-06), SCR-ATT-012 Online check-in, SCR-ATT-013 QR scan check-in, SCR-ATT-014 QR kiosk display, SCR-ATT-015 Missing punch requests, SCR-ATT-017 Devices (online status, last punch, last heartbeat), SCR-ATT-018 Device form, SCR-ATT-019 Device enrollments (with unmapped punches list), SCR-ATT-020 Import punch file, SCR-ATT-021 Device sync log, SCR-ATT-022 Suspicious punch alerts. Employee profile Attendance tab.
14. Definitions: Permissions.php for attendance.staff_attendance, administrative_entry, missing_punch, device, device_enrollment, punch_import, qr_kiosk, suspicious_alert per D05 5.7.11; Settings attendance.outside_geofence_mode, attendance.collapse_seconds, attendance.min_travel_minutes, attendance.kiosk_idle_minutes; Navigation (Attendance group: Staff today, Staff register, Devices, Alerts; My work: My attendance).
15. decisions.md rows: DEC-25, DEC-29, DEC-42, DEC-48, DEC-104, DEC-252, DEC-253.

DATABASE
attendance_devices (atd), device_enrollments (dve), attendance_punches (apn, log, partition-ready), staff_attendances (sta), staff_attendance_revisions (sav), qr_attendance_sessions (qrs), device_sync_logs (dsl, log), attendance_suspicious_alerts (aps) per D07 Part 2 7.22 and DEC-252.

BUSINESS RULES
- A punch is stored once (uq_apn); duplicates are dropped without error.
- Device requests are stored before processing and answered quickly (DEC-42).
- Only sources listed in the policy's official_sources count toward attendance.
- The day's branch is the branch of the first check-in (D04 4.11).
- Administrative entries and approved corrections are never overwritten by later punches.
- Online check-in needs a trusted device and passes the geofence rule.

UI SPEC
- W-06 on 360px: status card with Check in now, trusted device and location results, QR alternative, month calendar, missing punch prompt.
- Kiosk screen: full-screen code with countdown ring, branch name and clock; no navigation.
- Devices list shows online dot (green under 30 minutes since heartbeat), last punch and drift.

SECURITY AND PERMISSIONS
- Device secrets and comm keys encrypted, shown once, never logged; webhook signatures verified with constant-time comparison; replay protection.
- Device hosts have no session or cookies; tenant set only from the serial.
- Geolocation stored only on the punch row; not shown to other staff except HRM, PR and ITA.

TESTS
- ADMS: handshake reply format; ATTLOG upload of 100 lines stores 100 punches; re-upload stores none (duplicates); unknown serial 404; drift logged; two firmware fixtures parse.
- Webhook: valid signature accepted; bad signature, stale timestamp and reused nonce refused.
- Resolution: late, half day, early departure; collapse window; shared staff branch; precedence (an admin entry survives a later device punch); locked period refused.
- Auto-absent only on working days without leave; device-health alert once per outage.
- Online check-in: untrusted device refused; outside geofence refused in reject mode and flagged in flag mode.
- QR: current and previous codes accepted, older refused; second scan in the same session and direction refused.
- Missing punch approval adds a manual punch and re-resolves; requester cannot approve.
- Unmapped punch resolves after mapping.
- Screen tests, audit, isolation suite (a device serial of tenant A can never write into tenant B), RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. On staging, a ZKTeco device (or the ADMS simulator script in tools/adms-sim) pushes punches that resolve into the staff today screen within 5 minutes.
2. Re-sending the same punch batch creates no duplicate rows (evidence query).
3. Online check-in and QR kiosk work on a phone at 360px.
4. CI green; staging green.

DO NOT
- Do not compute payroll or leave balances here.
- Do not trust device-reported organization or branch values; resolve from the serial.
- Do not create partitions yet (P09.4).

OUTPUT
Report per Master section 20 with the device walkthrough, duplicate evidence, and the resolution rule test matrix.
```

### P05.4 Leave, Attendance Period Locks and Reports

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D11 Part 2 11.18 (day counting, flows) and 11.26 (period locks); D11 Part 1 11.4 (student_leave, staff_leave); D07 Part 2 7.23; D05 5.7.12 and 5.9; D10 Part 2 10.27 and SCR-ATT-023; D02 FR-LVE; D14 DEC-249.

PHASE GOAL
Run leave for students and staff through the approval engine with correct day counting, balances with accrual, carry-forward and lates conversion, write leave into attendance, add the attendance and staff attendance period lock checks, and deliver attendance and leave reports to close PH-05.

ALREADY BUILT
PH-00 to PH-04, P05.1 to P05.3: employees, reporting managers, class teachers, student and staff attendance with revisions, AttendancePrefillProviderRegistry, WorkingDayCalendar, approval resolvers class_teacher_of_student and reporting_manager.

TASKS
1. Migrations (Leave): leave_types, leave_policies, leave_balances, leave_balance_adjustments, leave_applications exactly as D07 7.23.
2. Seed templates (editable data, applied per organization on first use): staff types CL casual (half day allowed), SL sick (document after 2 days), EL earned, ML maternity, UL unpaid, SHL short leave; student types SICK and GEN; default policies per employment type with annual quotas as settings-backed data, never constants in code.
3. LeaveDayCalculator: uses WorkingDayCalendar (DEC-249) to skip non-working days; sandwich rule per policy (default on for staff, off for students); half days 0.5; short leave in minutes for SHL.
4. Actions: ApplyLeave (applicant employee or student; applied_via self, guardian or on_behalf; overlap with pending or approved applications refused, FR-LVE-005; back-dated up to leave.backdate_days (default 3); document required after the type's threshold; staff balance checked at submit; application_no from NumberingService key leave_application; SubmitApprovalRequest staff_leave or student_leave), CancelLeave (not started: cancelled, balance restored, future LV rows removed; partly taken: status cancellation_requested and a staff_leave request with payload kind cancellation), AdjustLeaveBalance (leave.balance.manage, reason; writes leave_balance_adjustments).
5. Approval types (Leave registers): student_leave (1 class_teacher_of_student, skipped when the class teacher is the requester; 2 principal_of_branch when days_count is above 3; approve permission leave.application.approve) and staff_leave (1 reporting_manager; 2 principal_of_branch when days_count is above 3; notify-only HRM; balance re-checked at apply). Handlers on approve: application approved; staff balance deducted through LeaveBalanceService with an adjustment row; AttendanceLeaveWriter writes LV: future and unfinalized days directly with source system and source_ref_type leave_application plus source_ref_id (soft reference, D07 Part 3 DEC-108), finalized past days through revisions with source correction referencing the application; refused with apply_failed and a clear message when a period is locked (the requester then asks for period_unlock).
6. Prefill provider: Leave registers LV in AttendancePrefillProviderRegistry for approved applications (students and staff).
7. Balances: commands leave:accrue monthly on day 1 (per policy, yearly or monthly accrual; lates_per_leave conversion from the previous month's staff attendance with an adjustment row and event leave.lates_converted to the employee) and leave:year-end on the policy period end (carry forward up to carry_forward_max; the rest lapses with an adjustment row). LeaveBalanceService is the only writer of leave_balances.
8. Period lock checks registered with PeriodLockService: attendance (no sheets in draft or submitted for the month; pending attendance corrections listed as warnings) and staff_attendance (no dirty punches unresolved for the month, no unmapped punches for mapped devices older than 24 hours). Monthly reminder on day 1 as attention items for PR and HRM to lock the previous month. All attendance and leave writes call PeriodLockService::assertUnlocked.
9. Notification events: leave.application_decided (guardian SMS and in-app for student leave; staff in-app), leave.balance_adjusted (employee in-app), leave.lates_converted.
10. Screens (D10 10.27 exactly): SCR-LVE-001 Leave applications (staff and student tabs), SCR-LVE-002 Apply on behalf (calculator shows counted days, holidays and sandwich days), SCR-LVE-003 My leave (balance cards, history, cancel), SCR-LVE-004 Apply for leave (shows approvers in order), SCR-LVE-005 Leave balances, SCR-LVE-006 Adjust balance, SCR-LVE-007 Leave calendar, SCR-LVE-008 Leave types, SCR-LVE-009 Leave policies, SCR-LVE-010 Leave reports, SCR-LVE-011 Guardian portal: apply for child's leave, SCR-LVE-012 Guardian portal: leave requests. Attendance: SCR-ATT-023 Attendance reports (daily summary, chronic absentees below a threshold, late comers, section comparison; filters; export and print). Employee profile Leave tab; guardian.home section "Leave requests".
11. Definitions: Permissions.php for leave.* per D05 5.7.12; Settings leave.backdate_days, leave.student_sandwich_rule, attendance.chronic_absence_percent (default 75); Navigation (Leave group: Applications, Balances, Calendar, Types, Policies, Reports; My work: My leave).
12. decisions.md rows: FR-LVE alignment, DEC-249 usage, D11 11.18 defaults.

DATABASE
leave_types (lvt), leave_policies (lvp), leave_balances (lvb), leave_balance_adjustments (lba), leave_applications (lva) per D07 Part 2 7.23.

BUSINESS RULES
- Overlapping pending or approved leave for the same person is refused (FR-LVE-005).
- Staff balance checked at submit and again at apply; negative only with policy allow_negative or an HRM adjustment.
- Every balance change writes an adjustment row; leave_balances changes only through LeaveBalanceService.
- Approved leave writes LV; finalized days change through revisions with the application reference; locked periods refuse.
- Cancelling untaken leave restores balance and removes future LV rows.
- Holidays and weekends never count; sandwich rule per policy.

UI SPEC
- Apply forms show a live calculator: from, to, half day options, counted days, skipped days with reasons, sandwich days highlighted, balance after.
- Balance cards per type with used, remaining and pending.
- Guardian leave form is 360px friendly with document upload by camera.

SECURITY AND PERMISSIONS
- Staff see own applications and balances; HRM and PR per branch; CT student leave within workflow.
- Guardians apply only for linked children with portal access; can_approve_requests not required to apply.
- Adjustments audited with reason.

TESTS
- Day calculator: weekend skipped; sandwich on and off; half days; short leave minutes.
- ApplyLeave: overlap refused; back-dated limit; document threshold; insufficient balance refused at submit and at apply when balance changed meanwhile (apply_failed).
- student_leave: class teacher step, skipped when requester is the class teacher; principal step above 3 days.
- staff_leave: reporting manager step with department head fallback; HRM notified.
- Attendance effect: future days prefilled LV; finalized past days revised with source correction; locked month gives apply_failed.
- Cancellation restores balance and removes future LV.
- Accrual, lates conversion and year-end carry-forward with adjustment rows.
- Period lock checks block locking with draft sheets or dirty punches; writes into a locked month refused until period_unlock is approved (PH-05 exit criterion).
- Reports: chronic absentees and late comers computed correctly on fixtures.
- Screen tests, audit, isolation suite, RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. On staging, a guardian applies for 2 days of child leave, the class teacher approves, and the sheet shows LV rows locked with the reason.
2. A staff member applies for 4 days, passes reporting manager and principal, and the balance and attendance update.
3. A locked month refuses an attendance edit and accepts it after an approved period_unlock.
4. PH-05 exit criteria met; MS-06 and D13 13.6 gate evidence attached.
5. CI green; staging green.

DO NOT
- Do not compute payroll deductions for unpaid leave (R2 Payroll reads approved leave later).
- Do not hardcode leave quotas or type codes in business logic.
- Do not let any code other than LeaveBalanceService change balances.

OUTPUT
Report per Master section 20 with the leave flow walkthroughs, the period lock evidence and the report samples. Then run the D13 13.6 gate for PH-05.
```

---

## Assumptions (Part 4)

| ID | Assumption |
|---|---|
| A-119 | The ZKTeco devices in pilot schools run firmware with ADMS push support and can point at the device host over HTTPS, or over HTTP through the reverse proxy if the firmware can't do TLS (the risk is accepted and recorded in the D12 accepted risks). |
| A-120 | Browser geolocation on staff phones is accurate to about 100 m. The default branch geofence radius is 150 m. |
| A-121 | CNIC and B-Form numbers are 13 digits. Foreign students may have neither, so both stay optional. |
| A-122 | Most guardians activate through an SMS OTP to the primary mobile. A small share activate at the office with staff help. |

## Gaps Found and Added (Part 4)

| ID | Gap | Decision |
|---|---|---|
| GAP-302 | Teacher visibility and class-teacher approval steps were needed before HR existed | Forward queries with empty defaults (DEC-247) |
| GAP-303 | Admission conversion depended on fee vouchers that arrive later | `AdmissionPaymentGate` (DEC-248) |
| GAP-304 | Attendance and leave needed holidays before Administration is built | `WorkingDayCalendar` (DEC-249) |
| GAP-305 | Portal homes needed module sections before full Reporting | `HomeSectionRegistry` (DEC-250) |
| GAP-306 | Period attendance needed timetable entries that don't exist in PH-05 | Moved to P09.1 (DEC-251) |
| GAP-307 | Suspicious punch alerts had no table | `attendance_suspicious_alerts` (DEC-252) |
| GAP-308 | Device push endpoints had no tenant host to call | D12 device hosts with tenant resolved by serial (DEC-253) |
| GAP-309 | No safe way to give guardians their first access | Password-less pending accounts with OTP activation (DEC-254) |
| GAP-310 | Students without mobiles couldn't activate a login | Guardian-generated setup code (DEC-255) |
| GAP-311 | Roll number generation was unspecified | Roll number modes (DEC-256) |
| GAP-312 | Consent had no storage with notice version and history | `consent_records` (DEC-257) |
| GAP-313 | Approved missing punches had no punch source value | `source` `admin` and `verify_type` `manual` (D7 correction) |
