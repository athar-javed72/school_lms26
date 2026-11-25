# Deliverable 10: Role-Wise Dashboards and Screen Inventory
# Part 3 of 3: Screen Inventory for Finance, Exams, Results, Timetable, Communication, Operations and R2/R3 Modules

**Version:** 1.0 (draft for approval) | **Date:** 24 Nov 2026
**Conventions:** columns as defined in Part 1, 10.2. Route names use snake_case segments (DEC-128). Role abbreviations follow D5 5.4. Wireframes continue from Part 2 (W-09 onward).

---

## 10.29 New Decisions

| ID | Decision |
|---|---|
| DEC-163 | **The cashier screen is keyboard-first.** The voucher number field takes barcode scanner input; Enter loads the voucher, the amount defaults to the outstanding balance, allocation follows the configured order, and Ctrl+Enter saves and prints the receipt. A routine cash payment takes under 30 seconds and three keystrokes after the scan. |
| DEC-164 | **Marks entry uses the attendance grid model** (DEC-162): per-cell autosave with version checks, codes `ABS`, `MED`, `EXM` alongside numbers, live validation against component maximums, and "Submit" per section or teaching group, which locks the cells until the submission is returned. |
| DEC-165 | **One report runner.** Standard reports are `ReportDefinition` classes (D3 3.23) rendered by a single screen, SCR-RPT-003: filters, table, optional chart, print and export. Each report has a key and a permission, listed in the report catalog (10.47); reports are not separate screens. |
| DEC-166 | **Online payment return pages never trust the browser.** After the gateway redirects back, the guardian sees "Verifying your payment" until the server has confirmed the transaction with the provider's verification API (D3 3.15, DEC-42). Only then does the page show success and a receipt. A payment the provider confirms later still posts and notifies the guardian. |

---

## 10.30 Fee (FEE)

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-FEE-001 | Fee heads | list, form | `app.fee.heads.index` | `fee.head.view` | MGT, ACT | R1 | Type (monthly, annual, one time, deposit, fine, pass-through), ledger account, allocation priority |
| SCR-FEE-002 | Tax rules | list, form | `app.fee.tax_rules.index` | `fee.head.update` | OA, ACT | R1 | Rate, heads, annual threshold, effective dates |
| SCR-FEE-003 | Fee structures | list | `app.fee.structures.index` | `fee.structure.view` | MGT, ACT, CSH, ADO | R1 | Per session, class, section and category; version and effective date |
| SCR-FEE-004 | Fee structure editor | grid | `app.fee.structures.edit` | `fee.structure.update` | ACT | R1 | Heads as rows, months as columns; vacation advance months; used structures can only get a new version |
| SCR-FEE-005 | Copy structures to a session | wizard | `app.fee.structures.copy` | `fee.structure.create` | ACT | R1 | Percentage or fixed increase; preview per class |
| SCR-FEE-006 | Discount rules | list, form | `app.fee.discount_rules.index` | `fee.discount_rule.view` | MGT, ACT | R1 | Sibling, early payment, category; preview against a sample family |
| SCR-FEE-007 | Concessions | list | `app.fee.concessions.index` | `fee.concession.view` | MGT, ACT | R1 | Status, approval link, effective months |
| SCR-FEE-008 | Request concession | form | `app.fee.concessions.create` | `fee.concession.create` | ACT, PR, ADO | R1 | Head or all, percent or amount, dates, reason, document; shows monthly effect; approval threshold note |
| SCR-FEE-009 | Scholarships | list, form | `app.fee.scholarships.index` | `fee.scholarship.view` | MGT, ACT | R1 | Sponsor, heads, period; approval flow |
| SCR-FEE-010 | Installment plans | list, form | `app.fee.installment_plans.index` | `fee.installment_plan.view` | MGT, ACT | R1 | Split a voucher or balance into dated installments; approval flow |
| SCR-FEE-011 | Vouchers | list | `app.fee.vouchers.index` | `fee.voucher.view` | MGT, ACT, CSH, ADO | R1 | Filters: month, class, status, type, family; totals row |
| SCR-FEE-012 | Voucher detail | detail | `app.fee.vouchers.show` | `fee.voucher.view` | Same | R1 | Lines, allocations, payments, accrual journal link, history; print, cancel, regenerate actions by permission |
| SCR-FEE-013 | Generate vouchers | wizard | `app.fee.vouchers.generate` | `fee.voucher.generate` | ACT | R1 | Month, classes, due date, preview counts and totals, skipped students with reasons, run as bulk job; see W-10 |
| SCR-FEE-014 | Issue ad hoc voucher | form | `app.fee.vouchers.create` | `fee.voucher.create` | ACT, ADO | R1 | Admission, fine, exam entry, other; heads and amounts |
| SCR-FEE-015 | Cancel or regenerate voucher | modal | `app.fee.vouchers.manage` | `fee.voucher.manage` | ACT | R1 | Blocked when payments exist (reversal first); level 2 with reason; regenerate supersedes |
| SCR-FEE-016 | Print vouchers | print | `app.fee.vouchers.print` | `fee.voucher.print` | ACT, CSH, BM | R1 | Batch PDF by class or month (3-copy layout, D9 9.19); bulk job for large batches |
| SCR-FEE-017 | Record payment | form | `app.fee.payments.create` | `fee.payment.create` | CSH, ACT | R1 | DEC-163; see W-09; needs an open cashier session for cash |
| SCR-FEE-018 | Payments | list | `app.fee.payments.index` | `fee.payment.view` | MGT, ACT, CSH | R1 | Method, cashier, date; reversed shown struck through with link |
| SCR-FEE-019 | Receipt | detail, print | `app.fee.payments.show` | `fee.payment.view` | Same | R1 | Allocations, reprint (marked "Duplicate"), reversal status |
| SCR-FEE-020 | Request payment reversal | modal | `app.fee.payments.reverse` | `fee.payment.reverse` | ACT, CSH | R1 | Level 2 with reason; creates `payment_reversal` approval; SOD-01 noted |
| SCR-FEE-021 | Cheques | list | `app.fee.cheques.index` | `fee.cheque.view` | ACT, CSH | R1 | Pending, cleared, bounced; clear or bounce (bounce reverses and can add a fine) |
| SCR-FEE-022 | Bank challan entry | grid | `app.fee.bank_challans.create` | `fee.bank_challan.create` | ACT | R1 | Enter vouchers paid at the bank by voucher number, date, bank reference |
| SCR-FEE-023 | Bank challan file import | wizard | `app.fee.bank_challans.import` | `fee.bank_challan.import` | ACT | R1 | Bank collection file mapping; matched, unmatched, duplicates |
| SCR-FEE-024 | Cashier sessions | list | `app.fee.cashier_sessions.index` | `fee.cashier_session.view` | ACT, PR, CSH (own) | R1 | Open sessions from earlier days highlighted |
| SCR-FEE-025 | Open cashier session | modal | `app.fee.cashier_sessions.store` | `fee.cashier_session.create` | CSH, ACT | R1 | Opening float; one open session per user (D7 `uq_csn_open`) |
| SCR-FEE-026 | Close cashier session | form | `app.fee.cashier_sessions.update` | `fee.cashier_session.create` | CSH, ACT | R1 | Denomination count, expected versus counted, variance creates `cash_variance` approval; prints cash summary |
| SCR-FEE-027 | Refunds | list, form | `app.fee.refunds.index` | `fee.refund.view` | MGT, ACT | R1 | Deposit refunds on withdrawal, excess payments; approval flow |
| SCR-FEE-028 | Student fee ledger | report | `app.fee.ledgers.student` | `fee.ledger.view` | MGT, ACT, CSH | R1 | Running balance, vouchers, payments, advance credit; print |
| SCR-FEE-029 | Family fee ledger | report | `app.fee.ledgers.family` | `fee.ledger.view` | MGT, ACT, CSH | R1 | All children combined |
| SCR-FEE-030 | Student profile: Fees tab | detail | `app.fee.student_tab.show` | `fee.ledger.view` | MGT, ACT, CSH | R1 | DEC-161; summary, open vouchers, last payments; hidden from teachers (D5 5.10) |
| SCR-FEE-031 | Opening balances import | wizard | `app.fee.ledgers.import` | `fee.ledger.import` | ACT | R1 | Uses SCR-IMX-002 with fee validation |
| SCR-FEE-032 | Defaulters | list | `app.fee.defaulters.index` | `fee.defaulter.view` | ACT, PR, BM, OW, OA | R1 | From `student_fee_accounts`; filters: months unpaid, amount, class; export |
| SCR-FEE-033 | Send fee reminders | modal | `app.fee.defaulters.remind` | `fee.defaulter.manage` | ACT | R1 | Template preview, SMS count, quiet hours respected |
| SCR-FEE-034 | Payment gateway accounts | list, form | `app.fee.gateway_accounts.index` | `fee.gateway.manage` | OW, OA | R2 | Merchant credentials masked; sandbox or live; test transaction |
| SCR-FEE-035 | Gateway transactions | list | `app.fee.gateway_transactions.index` | `fee.payment.view` | MGT, ACT | R2 | Initiated, verified, failed, expired; re-verify action |
| SCR-FEE-036 | Guardian: vouchers | list | `guardian.fee.vouchers.index` | `fee.voucher.view` | GDN | R1 | Open first; status pills |
| SCR-FEE-037 | Guardian: voucher detail | detail | `guardian.fee.vouchers.show` | `fee.voucher.view` | GDN | R1 | Lines, due date, download PDF, consumer number (1Bill, R2); see W-14 |
| SCR-FEE-038 | Guardian: pay online | form | `guardian.fee.online_payments.create` | `fee.voucher.view` + gateway enabled | GDN | R2 | Choose method, redirect to provider; idempotency key |
| SCR-FEE-039 | Guardian: payment result | detail | `guardian.fee.online_payments.show` | Same | GDN | R2 | DEC-166: verifying, success with receipt, failed with retry |
| SCR-FEE-040 | Guardian: payment history | list | `guardian.fee.payments.index` | `fee.payment.view` | GDN | R1 | Receipts download |
| SCR-FEE-041 | Guardian: fee ledger | report | `guardian.fee.ledger.show` | `fee.ledger.view` | GDN | R1 | |
| SCR-FEE-042 | Student portal: fees | list | `student.fee.vouchers.index` | `fee.voucher.view` (if enabled) | STD | R1 | Hidden unless the organization enables it (A-43) |

## 10.31 Accounting (ACC)

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-ACC-001 | Fiscal years | list, form | `app.accounting.fiscal_years.index` | `accounting.fiscal_year.view` | OW, OA, ACT | R1 | July to June default |
| SCR-ACC-002 | Close fiscal year | wizard | `app.accounting.fiscal_years.close` | `accounting.fiscal_year.manage` | OW | R1 | Checks: all journals posted, periods locked, trial balance balanced; level 3 confirm |
| SCR-ACC-003 | Chart of accounts | list | `app.accounting.accounts.index` | `accounting.account.view` | OW, OA, ACT | R1 | Tree with balances; system accounts badged and protected |
| SCR-ACC-004 | Account form | form | `app.accounting.accounts.edit` | `accounting.account.update` | ACT (org scope) | R1 | Parent, type, code; used accounts cannot change type |
| SCR-ACC-005 | Journals | list | `app.accounting.journals.index` | `accounting.journal.view` | OW, OA, ACT | R1 | Filters: type (manual, fee voucher, payment), date, status, source |
| SCR-ACC-006 | Journal entry form | grid | `app.accounting.journals.create` | `accounting.journal.create` | ACT | R1 | Balanced lines (debit equals credit shown live), branch per line, party; above threshold goes to approval |
| SCR-ACC-007 | Journal detail | detail | `app.accounting.journals.show` | `accounting.journal.view` | Same | R1 | Lines, source document link, approval, reversal link |
| SCR-ACC-008 | Reverse journal | modal | `app.accounting.journals.reverse` | `accounting.journal.reverse` | ACT | R1 | Creates a reversing entry through approval |
| SCR-ACC-009 | General ledger | report | `app.accounting.ledgers.show` | `accounting.ledger.view` | OW, OA, PR, ACT | R1 | Account, date range, branch; opening, movements, closing |
| SCR-ACC-010 | Cash and bank book | report | `app.accounting.cashbook.show` | `accounting.ledger.view` | Same | R1 | Daily cash book and bank books |
| SCR-ACC-011 | Trial balance | report | `app.accounting.trial_balance.show` | `accounting.trial_balance.view` | OW, OA, ACT | R1 | As of date, by branch or consolidated |
| SCR-ACC-012 | Receivables | report | `app.accounting.receivables.show` | `accounting.receivables.view` | OW, OA, PR, ACT | R1 | Student sub-ledger aging, reconciled to fee receivable account |
| SCR-ACC-013 | Opening balances | grid | `app.accounting.opening_balances.edit` | `accounting.opening_balance.create` | ACT (org scope), OW | R1 | Must balance before saving |
| SCR-ACC-014 | Bank accounts | list, form | `app.accounting.bank_accounts.index` | `accounting.bank_account.view` | OW, ACT | R1 | Linked ledger account and branch bank account |
| SCR-ACC-015 | Expense vouchers | list, form | `app.accounting.expenses.index` | `accounting.expense.view` | MGT, ACT | R2 | Approval flow; receipt upload |
| SCR-ACC-016 | Vendors | list, form | `app.accounting.vendors.index` | `accounting.vendor.view` | ACT, BM | R2 | |
| SCR-ACC-017 | Purchase bills | list, form | `app.accounting.purchases.index` | `accounting.purchase.view` | MGT, ACT | R2 | Lines, approval, payable |
| SCR-ACC-018 | Vendor payments | list, form | `app.accounting.vendor_payments.index` | `accounting.purchase.view` | ACT | R2 | |
| SCR-ACC-019 | Bank statement import | wizard | `app.accounting.bank_statements.import` | `accounting.reconciliation.create` | ACT | R2 | CSV mapping per bank |
| SCR-ACC-020 | Bank reconciliation | grid | `app.accounting.reconciliations.edit` | `accounting.reconciliation.create` | ACT | R2 | Match statement lines to ledger lines, suggestions, difference to zero |
| SCR-ACC-021 | Financial statements | report | `app.accounting.statements.show` | `accounting.statement.view` | OW, OA, ACT | R2 | Income and expenditure, balance sheet, by branch or consolidated |

## 10.32 Examination (EXM)

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-EXM-001 | Exam terms | list, form | `app.examination.exam_terms.index` | `examination.exam_term.view` | MGT, AC, EXC, HOD, TCH | R1 | Type, weightage, grading scheme, dates |
| SCR-EXM-002 | Exams | list | `app.examination.exams.index` | `examination.exam.view` | Same | R1 | One per term and class; status |
| SCR-EXM-003 | Exam setup | grid | `app.examination.exams.edit` | `examination.exam.update` | EXC | R1 | Papers per subject with components (theory, practical, paper 1, 2), maximum and pass marks, grade-only flag |
| SCR-EXM-004 | Date sheet editor | grid | `app.examination.date_sheets.edit` | `examination.date_sheet.create` | EXC | R1 | Dates by class, times, rooms; conflict detection for students in teaching groups and room capacity |
| SCR-EXM-005 | Publish date sheet | modal | `app.examination.date_sheets.publish` | `examination.date_sheet.publish` | EXC, PR | R1 | Notifies guardians and students |
| SCR-EXM-006 | Date sheet print | print | `app.examination.date_sheets.print` | `examination.date_sheet.print` | EXC, BM | R1 | Per class and combined |
| SCR-EXM-007 | Admit cards | list | `app.examination.admit_cards.index` | `examination.admit_card.view` | EXC, BM | R1 | Issued, blocked (fee clearance) with reason, override by permission |
| SCR-EXM-008 | Print admit cards | print | `app.examination.admit_cards.print` | `examination.admit_card.print` | EXC, BM | R1 | Batch PDF, 2 per A4 |
| SCR-EXM-009 | Mark sheets | list | `app.examination.mark_sheets.index` | `examination.marks.view` | PR, VP, AC, EXC, HOD, TCH | R1 | My papers for teachers; all papers for EXC; status per section |
| SCR-EXM-010 | Marks entry | grid | `app.examination.mark_sheets.edit` | `examination.marks.mark` | ST, CT | R1 | DEC-164; see W-11 |
| SCR-EXM-011 | Submit marks | modal | `app.examination.mark_sheets.submit` | `examination.marks.submit` | ST, CT | R1 | Summary of blanks and codes; after submit cells lock |
| SCR-EXM-012 | Submissions to review | list | `app.examination.submissions.index` | `examination.marks_review.view` | EXC, HOD | R1 | |
| SCR-EXM-013 | Review submission | detail | `app.examination.submissions.show` | `examination.marks_review.approve` | HOD, EXC | R1 | Distribution chart, outliers, approve or return with comment; SOD-04 enforced |
| SCR-EXM-014 | Import marks | wizard | `app.examination.marks.import` | `examination.marks.import` | EXC | R1 | Template per paper; validation against maximums |
| SCR-EXM-015 | Request marks revision | form | `app.examination.marks_revisions.create` | `examination.marks_revision.create` | ST, CT, EXC | R1 | After approval; old and new marks, reason; approval flow |
| SCR-EXM-016 | Marks revisions | list | `app.examination.marks_revisions.index` | `examination.marks_revision.view` | EXC, PR, HOD, TCH (own) | R1 | |
| SCR-EXM-017 | Completion tracker | report | `app.examination.completion.show` | `examination.completion.view` | PR, AC, EXC, HOD | R1 | Matrix of class by paper with status; reminders to teachers |
| SCR-EXM-018 | Award list print | print | `app.examination.mark_sheets.print` | `examination.marks.view` | EXC, TCH | R1 | Signed paper list per section for records |
| SCR-EXM-019 | Invigilation duties | grid | `app.examination.invigilation.edit` | `examination.invigilation.view` | EXC, PR, TCH (own) | R2 | Room and time slots; teacher clash check; duty slips |
| SCR-EXM-020 | Guardian: date sheet and admit card | detail | `guardian.examination.date_sheet.show` | `examination.date_sheet.view` | GDN | R1 | Admit card download when issued |
| SCR-EXM-021 | Student: date sheet and admit card | detail | `student.examination.date_sheet.show` | `examination.date_sheet.view` | STD | R1 | |

## 10.33 Result (RES)

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-RES-001 | Grading schemes | list | `app.result.grading_schemes.index` | `result.grading_scheme.view` | MGT, AC, EXC | R1 | Seeded templates (D7 7.52) and custom |
| SCR-RES-002 | Grading scheme bands | grid | `app.result.grading_schemes.edit` | `result.grading_scheme.update` | OA, EXC | R1 | Bands with overlap check; GPA points; Urdu remarks |
| SCR-RES-003 | Result rules | form | `app.result.rules.edit` | `result.rule.update` | OA, EXC | R1 | Pass percentages, compartment limit, grace marks, rounding, term weightages, attendance minimum, ranking |
| SCR-RES-004 | Result batches | list | `app.result.batches.index` | `result.batch.view` | PR, EXC | R1 | Per class and scope: version, status (processed, stale, approved, published) |
| SCR-RES-005 | Process results | wizard | `app.result.batches.create` | `result.batch.process` | EXC | R1 | Class, scope (exam, term, annual), rule, scheme; completion check; preview; runs as bulk job |
| SCR-RES-006 | Result batch detail | detail | `app.result.batches.show` | `result.batch.view` | PR, EXC | R1 | Student results, status counts, positions; stale warning when marks changed after processing; see W-12 |
| SCR-RES-007 | Student result | drawer | `app.result.student_results.show` | `result.student_result.view` | PR, VP, AC, EXC, CT, ST (own subjects) | R1 | Subject breakdown, grace applied, component marks |
| SCR-RES-008 | Withhold result | modal | `app.result.withholds.create` | `result.withhold.create` | PR | R1 | Reason (fee dues, discipline); hidden from portals |
| SCR-RES-009 | Request publication | modal | `app.result.publications.create` | `result.publication.create` | EXC | R1 | Creates `result_publication` approval |
| SCR-RES-010 | Publish result | modal | `app.result.publications.publish` | `result.publication.publish` | EXC | R1 | Only after approval; now or scheduled; notifies guardians |
| SCR-RES-011 | Request unpublish | modal | `app.result.publications.unpublish` | `result.publication.manage` | PR | R1 | Level 3 with reason |
| SCR-RES-012 | Report cards | print | `app.result.report_cards.index` | `result.report_card.view` | EXC, BM, CT | R1 | Batch PDF by section; national and Cambridge templates (FR-RES-015) |
| SCR-RES-013 | Report card remarks | grid | `app.result.remarks.edit` | `result.report_card.print` | CT, PR | R1 | Teacher remarks per student (comment bank), principal remark; autosave |
| SCR-RES-014 | Result analytics | report | `app.result.analytics.show` | `result.analytics.view` | OW, OA, PR, AC, EXC, HOD | R1 | Pass rates, grade distribution, subject and teacher comparison |
| SCR-RES-015 | Merit list | report | `app.result.merit_lists.show` | `result.analytics.view` | Same | R1 | Positions by section, class, branch per ranking rule |
| SCR-RES-016 | Recheck requests | list | `app.result.rechecks.index` | `result.recheck.view` | EXC, PR | R2 | Approval flow; outcome updates marks through revision |
| SCR-RES-017 | Guardian: results | list | `guardian.result.student_results.index` | `result.student_result.view` | GDN | R1 | Published only; withheld shows "Contact the school" |
| SCR-RES-018 | Guardian: report card | detail | `guardian.result.report_cards.show` | `result.report_card.view` | GDN | R1 | Download PDF |
| SCR-RES-019 | Student: results and report cards | list | `student.result.student_results.index` | `result.student_result.view` | STD | R1 | |
| SCR-RES-020 | Portal: request recheck | form | `{portal}.result.rechecks.create` | `result.recheck.create` | GDN, STD | R2 | Subject, reason; fee voucher if configured |
| SCR-RES-021 | Verify result card | public | `public.result.verify` | none | Anyone | R2 | Code or QR on result card; shows name, class, result status only |

## 10.34 ExternalExam (EXT)

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-EXT-001 | Awarding bodies | list, form | `app.external_exam.awarding_bodies.index` | `external_exam.awarding_body.view` | OA, EXC, PR | R1 | Platform catalog plus organization additions |
| SCR-EXT-002 | Syllabus codes | list, form | `app.external_exam.syllabus_codes.index` | `external_exam.syllabus_code.view` | AC, EXC, HOD, TCH | R1 | Subject to syllabus code, qualification, components, valid series (DEC-82) |
| SCR-EXT-003 | Centre numbers | list, form | `app.external_exam.centre_numbers.index` | `external_exam.awarding_body.update` | OA, EXC | R1 | Per branch and awarding body |
| SCR-EXT-004 | Candidate identifiers | grid | `app.external_exam.candidate_ids.index` | `external_exam.candidate_id.view` | EXC, ADO, BM | R1 | Candidate number, UCI, board registration; import |
| SCR-EXT-005 | Predicted grades entry | grid | `app.external_exam.predicted_grades.edit` | `external_exam.predicted_grade.create` | ST, CT | R1 | Per teaching group and series (DEC-85) |
| SCR-EXT-006 | Predicted grades review | list | `app.external_exam.predicted_grades.index` | `external_exam.predicted_grade.approve` | HOD, EXC | R1 | Approve; export for university references |
| SCR-EXT-007 | Exam series | list | `app.external_exam.series.index` | `external_exam.series.view` | MGT, EXC, ACT | R2 | June and November series; deadlines countdown |
| SCR-EXT-008 | Series form | form | `app.external_exam.series.edit` | `external_exam.series.update` | EXC | R2 | Normal, late 1, late 2 and retake deadlines; results date |
| SCR-EXT-009 | Entry fee schedules | grid | `app.external_exam.fee_schedules.edit` | `external_exam.fee_schedule.update` | EXC, ACT | R2 | Fee per syllabus, component option and stage; reference currency and rate |
| SCR-EXT-010 | Entries | list | `app.external_exam.entries.index` | `external_exam.entry.view` | MGT, EXC, AC, CT, ST (assigned) | R2 | By series, student, syllabus, status |
| SCR-EXT-011 | Generate draft entries | wizard | `app.external_exam.entries.generate` | `external_exam.entry.create` | EXC, AC | R2 | From student subjects and target qualifications; preview |
| SCR-EXT-012 | Entry review | grid | `app.external_exam.entries.review` | `external_exam.entry.create` | EXC, AC | R2 | Students by syllabus with component options; guardian confirmation status |
| SCR-EXT-013 | Confirm entries and issue vouchers | modal | `app.external_exam.entries.confirm` | `external_exam.entry.approve` | EXC | R2 | Issues entry fee vouchers (pass-through, DEC-84) |
| SCR-EXT-014 | Entry export | print | `app.external_exam.entries.export` | `external_exam.entry.export` | EXC | R2 | File for the awarding body portal; mark as submitted |
| SCR-EXT-015 | Entry withdrawals | list | `app.external_exam.withdrawals.index` | `external_exam.entry_withdrawal.view` | EXC, PR | R2 | Approval flow; refund handling note |
| SCR-EXT-016 | Import results | wizard | `app.external_exam.result_imports.create` | `external_exam.result.import` | EXC | R2 | Statement of results file mapping; matched and unmatched candidates |
| SCR-EXT-017 | Review and publish results | detail | `app.external_exam.results.index` | `external_exam.result.view` | MGT, EXC, AC, HOD | R2 | Publish to portals (`external_exam.result.publish`) |
| SCR-EXT-018 | External result analytics | report | `app.external_exam.analytics.show` | `external_exam.result.view` | EXC, PR, HOD | R2 | Grade distribution, value added against predicted grades |
| SCR-EXT-019 | Enquiries about results | list, form | `app.external_exam.enquiries.index` | `external_exam.enquiry.view` | EXC, PR | R2 | Service type, fee, outcome |
| SCR-EXT-020 | Certificates register | list, form | `app.external_exam.certificates.index` | `external_exam.certificate.view` | EXC, BM | R2 | Received, handed over with signature |
| SCR-EXT-021 | IBCC equivalence calculator | form | `app.external_exam.equivalence.create` | `external_exam.equivalence.create` | EXC, ADO | R2 | Grades in, marks out of 1100; saved calculations |
| SCR-EXT-022 | Conversion rules | grid | `app.external_exam.conversion_rules.index` | `external_exam.conversion_rule.view` | EXC, OA | R2 | Per session year; subject-specific A* values |
| SCR-EXT-023 | National board registrations and results | grid | `app.external_exam.board.index` | `external_exam.entry.view` | EXC | R2 | Registration export, board roll numbers, results entry (FR-EXT-016) |
| SCR-EXT-024 | Guardian: confirm entries | form | `guardian.external_exam.entries.confirm` | `external_exam.entry.view` (own) | GDN | R2 | Subjects, components, fees; confirm or ask for change |
| SCR-EXT-025 | Portal: external results and enquiry | detail | `{portal}.external_exam.results.index` | `external_exam.result.view` (published) | GDN, STD | R2 | Request enquiry; equivalence calculator for guardians and students |

## 10.35 Timetable (TTB)

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-TTB-001 | Bell schedules | list, form | `app.timetable.bell_schedules.index` | `timetable.bell_schedule.view` | STAFF | R1 | Periods and breaks per shift and day pattern |
| SCR-TTB-002 | Timetable versions | list | `app.timetable.timetables.index` | `timetable.timetable.view` | STAFF | R1 | Draft, active, archived per branch, session and shift (DEC-109) |
| SCR-TTB-003 | Timetable editor | grid | `app.timetable.timetables.edit` | `timetable.timetable.update` | AC | R1 | Views by section, teacher, room; drag or buttons (WCAG 2.5.7); live conflicts; workload counters; see W-13 |
| SCR-TTB-004 | Activate version | modal | `app.timetable.timetables.publish` | `timetable.timetable.publish` | AC, PR | R1 | Blocked while conflicts exist; effective date; notifies teachers |
| SCR-TTB-005 | Teacher timetable | report, print | `app.timetable.teacher_view.show` | `timetable.timetable.view` | STAFF | R1 | Per teacher, printable |
| SCR-TTB-006 | Room timetable | report | `app.timetable.room_view.show` | `timetable.timetable.view` | STAFF | R1 | Free rooms by period |
| SCR-TTB-007 | Section timetable print | print | `app.timetable.timetables.print` | `timetable.timetable.print` | AC, TCH | R1 | Classroom display layout |
| SCR-TTB-008 | My timetable | detail | `app.timetable.my_timetable.show` | `timetable.timetable.view` (own) | TCH | R1 | Today highlighted; substitutions added (R2) |
| SCR-TTB-009 | Substitutions | grid | `app.timetable.substitutions.index` | `timetable.substitution.view` | MGT, AC, TCH (own) | R2 | Absent teachers' periods today, suggested free teachers by load; notify substitutes |
| SCR-TTB-010 | Guardian: timetable | detail | `guardian.timetable.timetable.show` | `timetable.timetable.view` | GDN | R1 | |
| SCR-TTB-011 | Student: timetable | detail | `student.timetable.my_timetable.show` | `timetable.timetable.view` | STD | R1 | |

## 10.36 Communication (COMM)

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-COMM-001 | Announcements | list | `app.communication.announcements.index` | `communication.announcement.view` | Audience, MGT, AC | R1 | Pinned first; read and acknowledged counts for authors |
| SCR-COMM-002 | Announcement form | form | `app.communication.announcements.create` | `communication.announcement.create` | MGT, AC | R1 | Audience builder (organization, branch, class, section, role, user, portal), schedule, expiry, acknowledgement, SMS or WhatsApp copy with cost estimate |
| SCR-COMM-003 | Announcement detail | detail | `app.communication.announcements.show` | `communication.announcement.view` | Same | R1 | Who has read and acknowledged; remind unread |
| SCR-COMM-004 | Diary entries | list | `app.communication.diary_entries.index` | `communication.diary.view` | MGT, AC, TCH | R1 | By date and section; status |
| SCR-COMM-005 | Write diary entry | form | `app.communication.diary_entries.create` | `communication.diary.create` | TCH | R1 | Section or teaching group, subject, type (homework, reminder, note); same-day edit only; review if enabled |
| SCR-COMM-006 | Diary review | list | `app.communication.diary_entries.review` | `communication.diary_review.approve` | AC, VP | R1 | Approve or return |
| SCR-COMM-007 | Message threads (staff) | list | `app.communication.message_threads.index` | `communication.message.view` | TCH | R2 | One thread per student, guardian and teacher |
| SCR-COMM-008 | Message thread | detail | `app.communication.message_threads.show` | `communication.message.view` | TCH | R2 | Office hours notice; report or hide message |
| SCR-COMM-009 | Message oversight | list | `app.communication.message_oversight.index` | `communication.message_oversight.view` | PR, VP | R2 | Read-only; flagged messages |
| SCR-COMM-010 | Parent-teacher meetings | list, form | `app.communication.ptm_events.index` | `communication.ptm.view` | MGT, TCH | R2 | Event, classes, date |
| SCR-COMM-011 | My meeting slots | grid | `app.communication.ptm_slots.edit` | `communication.ptm.create` (own slots) | TCH | R2 | Slot length, capacity; booked list |
| SCR-COMM-012 | Meeting bookings | report | `app.communication.ptm_bookings.index` | `communication.ptm.view` | MGT, TCH | R2 | Attendance marking at the meeting |
| SCR-COMM-013 | Guardian: announcements | list | `guardian.communication.announcements.index` | Baseline (audience) | GDN | R1 | Acknowledge button where required |
| SCR-COMM-014 | Guardian: diary | list | `guardian.communication.diary.index` | `communication.diary.view` | GDN | R1 | By date for the selected child |
| SCR-COMM-015 | Guardian: messages | list, detail | `guardian.communication.message_threads.index` | `communication.message.create` | GDN | R2 | |
| SCR-COMM-016 | Guardian: book a meeting slot | form | `guardian.communication.ptm_bookings.create` | `communication.ptm.manage` | GDN | R2 | |
| SCR-COMM-017 | Student: announcements and diary | list | `student.communication.feed.index` | `communication.diary.view` | STD | R1 | |

## 10.37 Administration (ADN)

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-ADN-001 | Calendar | detail | `app.administration.calendar.index` | Baseline | Everyone | R1 | Month and list views; filters by type and branch |
| SCR-ADN-002 | Calendar event form | form | `app.administration.calendar.create` | `administration.calendar.create` / `administration.event.create` | OA, PR, BM | R1 | Holiday, vacation, event, exam period, working day override; affects attendance flag |
| SCR-ADN-003 | Declare closure | modal | `app.administration.closures.create` | `administration.closure.create` | OW, PR | R1 | Emergency closure: dates, branches, reason; marks attendance `H`, notifies guardians and staff |
| SCR-ADN-004 | ID cards | wizard | `app.administration.id_cards.index` | `administration.id_card.print` | BM, ADO | R1 | Select students or staff, template, missing photo report, batch PDF |
| SCR-ADN-005 | Gate passes | list, form | `app.administration.gate_passes.index` | `administration.gate_pass.view` | REC, BM, PR | R2 | Pick-up by authorized person with photo match; guardian notified |
| SCR-ADN-006 | Visitor log | list, form | `app.administration.visitors.index` | `administration.visitor.view` | REC, BM | R3 | Check in, badge, host, check out |
| SCR-ADN-007 | Complaints | list | `app.administration.complaints.index` | `administration.complaint.view` | BM, PR, STAFF (own) | R3 | SLA due, priority, status |
| SCR-ADN-008 | Complaint detail | detail | `app.administration.complaints.show` | `administration.complaint.manage` | BM, PR | R3 | Assign, resolve, satisfaction |
| SCR-ADN-009 | Guardian: calendar | detail | `guardian.administration.calendar.index` | Baseline | GDN | R1 | |
| SCR-ADN-010 | Guardian: gate passes | list | `guardian.administration.gate_passes.index` | `administration.gate_pass.view` | GDN | R2 | |
| SCR-ADN-011 | Guardian: submit complaint | form | `guardian.administration.complaints.create` | `administration.complaint.create` | GDN | R3 | Status tracking |
| SCR-ADN-012 | Student: calendar | detail | `student.administration.calendar.index` | Baseline | STD | R1 | |

## 10.38 Reporting (RPT)

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-RPT-001 | Staff dashboard | dashboard | `app.home` | Baseline | All staff | R1 | Part 1, 10.8 and 10.9 |
| SCR-RPT-002 | Report catalog | list | `app.reporting.reports.index` | Any report permission | Staff with report access | R1 | Grouped by module; only reports the user may run; saved views shown |
| SCR-RPT-003 | Report runner | report | `app.reporting.reports.show` | The report's permission | Per report | R1 | DEC-165: filters, results, chart, print, Excel and PDF export (masking applied), save view |
| SCR-RPT-004 | Saved views | list | `app.reporting.saved_filters.index` | Baseline | Staff | R1 | Rename, delete, share (R2) |
| SCR-RPT-005 | Scheduled reports | list, form | `app.reporting.scheduled_reports.index` | `reporting.scheduled_report.view` | OW, OA, PR, ACT | R2 | Report, filters, format, frequency, recipients |
| SCR-RPT-006 | Branch comparison | report | `app.reporting.branch_comparison.show` | `reporting.branch_comparison.view` | OW, OA | R1 | Branches side by side over a period |
| SCR-RPT-007 | Snapshot status | detail | `app.reporting.snapshots.index` | `reporting.snapshot.view` | OA, ITA | R1 | Last computed per metric; force refresh |
| SCR-RPT-008 | Guardian home | dashboard | `guardian.home` | Baseline | GDN | R1 | Part 1, 10.9 |
| SCR-RPT-009 | Student home | dashboard | `student.home` | Baseline | STD | R1 | Part 1, 10.9 |

## 10.39 Learning (LMS), R2

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-LMS-001 | Courses | list | `app.learning.courses.index` | `learning.course.view` | AC, HOD, ST | R2 | Per session, class, subject; status |
| SCR-LMS-002 | Course builder | grid | `app.learning.courses.edit` | `learning.course.manage` / `learning.lecture.create` | AC, HOD / ST | R2 | Chapters, topics, lectures tree; reorder by buttons or drag |
| SCR-LMS-003 | Lecture form | form | `app.learning.lectures.edit` | `learning.lecture.create` | ST | R2 | Video link or upload, document, rich text; availability date; publish |
| SCR-LMS-004 | Course targets | drawer | `app.learning.course_targets.edit` | `learning.course.manage` | AC, HOD | R2 | Sections or teaching groups |
| SCR-LMS-005 | Discussion moderation | list | `app.learning.discussions.index` | `learning.discussion.manage` | ST, AC | R2 | Pin, lock, hide posts |
| SCR-LMS-006 | Content reports | list | `app.learning.content_reports.index` | `learning.content_report.manage` | ST, AC | R2 | Action or dismiss |
| SCR-LMS-007 | Course progress | report | `app.learning.progress.show` | `learning.progress.view` | ST, AC | R2 | Per student and lecture |
| SCR-LMS-008 | Student: courses | list | `student.learning.courses.index` | `learning.lecture.view` | STD | R2 | Progress bars |
| SCR-LMS-009 | Student: lecture | detail | `student.learning.lectures.show` | `learning.lecture.view` | STD | R2 | Player, attachments, next lecture, discussion; progress saved every 30 seconds |
| SCR-LMS-010 | Student: discussion thread | detail | `student.learning.discussions.show` | `learning.discussion.create` | STD | R2 | Report a post |
| SCR-LMS-011 | Guardian: learning progress | detail | `guardian.learning.progress.show` | `learning.progress.view` | GDN | R2 | |

## 10.40 Homework (HWK), R2

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-HWK-001 | Homework assignments | list | `app.homework.assignments.index` | `homework.assignment.view` | PR, AC, HOD, TCH | R2 | Due dates, submissions count |
| SCR-HWK-002 | Assignment form | form | `app.homework.assignments.create` | `homework.assignment.create` | TCH | R2 | Targets, due, mode (online, offline, none), marks, late rules, attachments; creates diary line |
| SCR-HWK-003 | Submissions | grid | `app.homework.submissions.index` | `homework.submission.view` | TCH, AC | R2 | Submitted, late, not submitted (computed), reviewed |
| SCR-HWK-004 | Review submission | detail | `app.homework.submissions.show` | `homework.submission.approve` | TCH | R2 | Files, marks or grade, feedback, request resubmission |
| SCR-HWK-005 | Homework report | report | `app.homework.reports.show` | `homework.report.view` | PR, AC, TCH | R2 | Compliance by section and student |
| SCR-HWK-006 | Student: homework | list | `student.homework.assignments.index` | `homework.assignment.view` | STD | R2 | Due soon first |
| SCR-HWK-007 | Student: submit homework | form | `student.homework.submissions.create` | `homework.submission.create` | STD | R2 | Text and files (camera upload on mobile), attempt history |
| SCR-HWK-008 | Guardian: homework | list, form | `guardian.homework.assignments.index` | `homework.assignment.view` / `homework.submission.create` | GDN | R2 | View and submit for young children |

## 10.41 Assessment (ASM), R2

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-ASM-001 | Question banks | list, form | `app.assessment.question_banks.index` | `assessment.question_bank.view` | HOD, ST | R2 | Sharing level |
| SCR-ASM-002 | Questions | list | `app.assessment.questions.index` | `assessment.question.view` | HOD, ST | R2 | Filters: type, difficulty, tag, chapter, status |
| SCR-ASM-003 | Question editor | form | `app.assessment.questions.edit` | `assessment.question.update` | HOD, ST | R2 | Types (D7 7.39), Urdu and math input, options, accepted answers, explanation, preview |
| SCR-ASM-004 | Import questions | wizard | `app.assessment.questions.import` | `assessment.question.import` | HOD, ST | R2 | |
| SCR-ASM-005 | Quizzes | list | `app.assessment.quizzes.index` | `assessment.quiz.view` | PR, AC, HOD, EXC, ST | R2 | Status, window, attempts |
| SCR-ASM-006 | Quiz builder | form | `app.assessment.quizzes.edit` | `assessment.quiz.update` | ST, EXC | R2 | Pick or randomize questions, marks, timing, attempts, shuffle, result release, targets, exam link |
| SCR-ASM-007 | Quiz monitor | detail | `app.assessment.quizzes.monitor` | `assessment.quiz.view` | ST, EXC | R2 | Live attempts, time left, force close, void with reason |
| SCR-ASM-008 | Grading queue | grid | `app.assessment.grading.index` | `assessment.grading.manage` | ST | R2 | Subjective answers one by one or by question |
| SCR-ASM-009 | Quiz results and analysis | report | `app.assessment.analytics.show` | `assessment.analytics.view` | HOD, AC, ST | R2 | Scores, item difficulty, distractor analysis; release results |
| SCR-ASM-010 | Student: quizzes | list | `student.assessment.quizzes.index` | `assessment.quiz.view` | STD | R2 | Open, upcoming, completed |
| SCR-ASM-011 | Student: quiz attempt | detail | `student.assessment.attempts.show` | `assessment.attempt.create` | STD | R2 | Server timer, autosave per answer, question navigator; see W-15 |
| SCR-ASM-012 | Student: quiz review | detail | `student.assessment.attempts.review` | `assessment.attempt.create` | STD | R2 | After result release: answers, correct answers, explanations |
| SCR-ASM-013 | Guardian: quiz results | list | `guardian.assessment.quizzes.index` | `assessment.quiz.view` | GDN | R2 | |

## 10.42 Payroll (PAY), R2, all sensitive

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-PAY-001 | Salary components | list, form | `app.payroll.components.index` | `payroll.component.view` | OW, HRM, ACT | R2 | Earning, deduction, employer contribution; calculation type; ledger account |
| SCR-PAY-002 | Income tax slabs | list | `app.payroll.tax_slabs.index` | `payroll.component.view` | OW, HRM, ACT | R2 | Platform slabs per tax year with organization override (A-62) |
| SCR-PAY-003 | Salary structures | list, form | `app.payroll.structures.index` | `payroll.structure.view` | OW, HRM, ACT | R2 | |
| SCR-PAY-004 | Employee salary setup | form | `app.payroll.employee_salaries.edit` | `payroll.employee_salary.update` | OW, HRM | R2 | Structure, basic, pay branch, bank (masked), EOBI and tax fields, component overrides |
| SCR-PAY-005 | Payroll runs | list | `app.payroll.runs.index` | `payroll.run.view` | OW, PR, HRM, ACT | R2 | Per branch and month; status |
| SCR-PAY-006 | New payroll run | wizard | `app.payroll.runs.create` | `payroll.run.create` | HRM, ACT | R2 | Month, branch, run type; checks staff attendance lock; calculate as bulk job |
| SCR-PAY-007 | Payroll run review | grid | `app.payroll.runs.show` | `payroll.run.view` | Same | R2 | Payslips with gross, deductions, net; hold, adjust, recalculate; totals; submit for approval; see W-16 |
| SCR-PAY-008 | Payslip | detail, print | `app.payroll.payslips.show` | `payroll.payslip.view` | OW, HRM, ACT | R2 | |
| SCR-PAY-009 | Finalize payroll | modal | `app.payroll.runs.finalize` | `payroll.run.finalize` | ACT | R2 | After approval; posts journal |
| SCR-PAY-010 | Salary payments | grid | `app.payroll.payments.index` | `payroll.payment.view` | OW, ACT | R2 | Bank upload file export, mark paid, cash payments |
| SCR-PAY-011 | Salary advances | list, form | `app.payroll.advances.index` | `payroll.advance.view` | OW, HRM, ACT | R2 | Approval flow; recovery schedule |
| SCR-PAY-012 | Employee loans | list, detail | `app.payroll.loans.index` | `payroll.loan.view` | OW, HRM, ACT | R2 | Installments, deferral, outstanding |
| SCR-PAY-013 | My payslips | list | `app.payroll.my_payslips.index` | `payroll.payslip.view` (own) | STAFF | R2 | Download PDF |
| SCR-PAY-014 | Request advance or loan | form | `app.payroll.my_requests.create` | `payroll.advance.create` / `payroll.loan.create` (own) | STAFF | R2 | Amount, months, reason |

## 10.43 Library (LIB), R3

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-LIB-001 | Catalog | list | `app.library.books.index` | `library.book.view` | LBR, MGT, STAFF | R3 | Search by title, author, ISBN, category; availability |
| SCR-LIB-002 | Book form | form | `app.library.books.edit` | `library.book.update` | LBR | R3 | Authors, publisher, category, ISBN; import |
| SCR-LIB-003 | Copies | list | `app.library.copies.index` | `library.copy.view` | LBR | R3 | Accession, barcode, condition; print labels |
| SCR-LIB-004 | Members | list, form | `app.library.members.index` | `library.member.view` | LBR | R3 | Card number, limits, validity; bulk create from classes |
| SCR-LIB-005 | Circulation desk | form | `app.library.circulation.index` | `library.circulation.issue` | LBR | R3 | Scan member card, scan copy: issue, return, renew; fines shown |
| SCR-LIB-006 | Reservations | list | `app.library.reservations.index` | `library.circulation.view` | LBR | R3 | Queue per title; notify when ready |
| SCR-LIB-007 | Overdue and fines | list | `app.library.fines.index` | `library.fine.view` | LBR, ACT, PR | R3 | Send reminders; bill to voucher or payroll (DEC-124) |
| SCR-LIB-008 | Waive fine | modal | `app.library.fines.waive` | `library.fine.manage` | LBR, PR | R3 | Reason |
| SCR-LIB-009 | Student: library | list | `student.library.loans.index` | `library.circulation.view` | STD | R3 | My loans, due dates, catalog search, reserve |
| SCR-LIB-010 | Guardian: library | list | `guardian.library.loans.index` | `library.circulation.view` | GDN | R3 | Child's loans and fines |
| SCR-LIB-011 | Staff: my loans | list | `app.library.my_loans.index` | `library.circulation.view` (own) | STAFF | R3 | |

## 10.44 Transport (TRN)

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-TRN-001 | Routes | list | `app.transport.routes.index` | `transport.route.view` | TM, MGT, TS | R2 | Capacity, allocated, vehicle (R3) |
| SCR-TRN-002 | Route form | form | `app.transport.routes.edit` | `transport.route.update` | TM | R2 | Stops in sequence with times and monthly fee |
| SCR-TRN-003 | Stops | list, form | `app.transport.stops.index` | `transport.stop.view` | TM, MGT | R2 | Map pin (Leaflet, R3) |
| SCR-TRN-004 | Allocations | list | `app.transport.allocations.index` | `transport.allocation.view` | TM, MGT, ACT, TS | R2 | |
| SCR-TRN-005 | Allocate student | modal | `app.transport.allocations.create` | `transport.allocation.create` | TM | R2 | Route, stop, direction, start date; capacity check; fee preview |
| SCR-TRN-006 | Route sheet | print | `app.transport.routes.print` | `transport.route.print` | TM | R2 | Students by stop with guardian phones (permission-masked) |
| SCR-TRN-007 | Vehicles | list | `app.transport.vehicles.index` | `transport.vehicle.view` | TM, MGT | R3 | Documents expiring badge |
| SCR-TRN-008 | Vehicle detail | detail | `app.transport.vehicles.show` | `transport.vehicle.view` | TM, MGT | R3 | Documents, assignments, maintenance, fuel |
| SCR-TRN-009 | Drivers | list, form | `app.transport.drivers.index` | `transport.driver.view` | TM, MGT | R3 | License expiry, verification |
| SCR-TRN-010 | Route assignments | list, form | `app.transport.route_assignments.index` | `transport.route.update` | TM | R3 | Vehicle, driver, attendant per route with history |
| SCR-TRN-011 | Maintenance logs | list, form | `app.transport.maintenance.index` | `transport.maintenance.view` | TM, BM | R3 | |
| SCR-TRN-012 | Fuel logs | list, form | `app.transport.fuel.index` | `transport.fuel.view` | TM, BM, ACT | R3 | Litres, cost, km per litre |
| SCR-TRN-013 | Boarding | grid | `app.transport.boarding.edit` | `transport.boarding.mark` | TS | R3 | Mobile list by stop: boarded or not; guardian notified on drop |
| SCR-TRN-014 | Live map | detail | `app.transport.gps.show` | `transport.gps.view` | TM | R3 | Vehicle positions (30-day history) |
| SCR-TRN-015 | GPS devices | list, form | `app.transport.gps_devices.index` | `transport.vehicle.update` | TM | R3 | Provider, device id (global unique) |
| SCR-TRN-016 | Guardian: transport | detail | `guardian.transport.allocation.show` | `transport.allocation.view` | GDN | R2 | Route, stop, pickup time; live bus location (R3) |

## 10.45 Inventory (INV), R3

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-INV-001 | Items | list | `app.inventory.items.index` | `inventory.item.view` | BM, ACT | R3 | Stock on hand, reorder level |
| SCR-INV-002 | Item form | form | `app.inventory.items.edit` | `inventory.item.update` | BM | R3 | Category, unit, kind |
| SCR-INV-003 | Stock locations | list, form | `app.inventory.locations.index` | `inventory.location.view` | BM | R3 | Custodian |
| SCR-INV-004 | Stock balances | report | `app.inventory.stock.index` | `inventory.stock.view` | BM, ACT | R3 | By item and location; valuation |
| SCR-INV-005 | Receive stock | form | `app.inventory.stock.receive` | `inventory.stock.manage` | BM | R3 | Against purchase bill line or direct |
| SCR-INV-006 | Issue stock | form | `app.inventory.stock.issue` | `inventory.stock.manage` | BM | R3 | To employee or section; negative stock blocked |
| SCR-INV-007 | Transfer stock | form | `app.inventory.stock.transfer` | `inventory.stock.manage` | BM | R3 | Paired movements |
| SCR-INV-008 | Stock adjustment request | form | `app.inventory.adjustments.create` | `inventory.stock.create` | BM | R3 | Approval flow |
| SCR-INV-009 | Stock movements | list | `app.inventory.movements.index` | `inventory.stock.view` | BM, ACT | R3 | Append-only ledger |
| SCR-INV-010 | Assets register | list | `app.inventory.assets.index` | `inventory.asset.view` | BM, ACT, PR | R3 | Tag, location, holder, status |
| SCR-INV-011 | Asset detail | detail | `app.inventory.assets.show` | `inventory.asset.view` | Same | R3 | Assignment history, maintenance, disposal |
| SCR-INV-012 | Assign or return asset | modal | `app.inventory.assets.assign` | `inventory.asset.assign` | BM | R3 | Condition out and in |
| SCR-INV-013 | Maintenance requests | list, detail | `app.inventory.maintenance.index` | `inventory.maintenance.view` | BM, STAFF (own) | R3 | Assign, resolve, cost |
| SCR-INV-014 | Report a problem | form | `app.inventory.maintenance.create` | `inventory.maintenance.create` | STAFF | R3 | Asset or room, category, photo |

## 10.46 Discipline (DSC)

| ID | Screen | Type | Route name | Permission | Default roles | Rel | Key behaviour and states |
|---|---|---|---|---|---|---|---|
| SCR-DSC-001 | Incidents | list | `app.discipline.incidents.index` | `discipline.incident.view` | PR, VP, CT | R2 | Category, severity, status |
| SCR-DSC-002 | Record incident | form | `app.discipline.incidents.create` | `discipline.incident.create` | TCH, VP | R2 | Students and involvement, description, guardian notice flag |
| SCR-DSC-003 | Incident detail | detail | `app.discipline.incidents.show` | `discipline.incident.view` | Same | R2 | Statements, actions, timeline |
| SCR-DSC-004 | Take action | modal | `app.discipline.actions.create` | `discipline.action.create` | VP, PR | R2 | Warning, detention, suspension (creates suspension through Student), guardian meeting |
| SCR-DSC-005 | Student remarks | list, form | `app.discipline.remarks.index` | `discipline.remark.view` | PR, VP, CT | R2 | Positive, concern, general; share with guardian flag |
| SCR-DSC-006 | Achievements | list, form | `app.discipline.achievements.index` | `discipline.achievement.view` | MGT, TCH | R2 | Level, position, certificate; show on portal |
| SCR-DSC-007 | Discipline categories | list, form | `app.discipline.categories.index` | `discipline.incident.view` | PR, VP | R2 | Severity, guardian notice default |
| SCR-DSC-008 | Counseling sessions | list | `app.discipline.counseling.index` | `discipline.counseling.view` | PR (view), counselor custom role | R3 | Restricted (DEC-122); 2FA |
| SCR-DSC-009 | Counseling session form | form | `app.discipline.counseling.edit` | `discipline.counseling.create` | Counselor custom role | R3 | Encrypted notes; no export |
| SCR-DSC-010 | Guardian: remarks and achievements | list | `guardian.discipline.feed.index` | `discipline.remark.view` (shared), `discipline.achievement.view` | GDN | R2 | Shared remarks and actions only |
| SCR-DSC-011 | Student: achievements | list | `student.discipline.achievements.index` | `discipline.achievement.view` | STD | R2 | |

---

## 10.47 Report Catalog (rendered by SCR-RPT-003)

| Report key | Title | Permission | Rel |
|---|---|---|---|
| `fee.daily_collection` | Daily collection by method and cashier | `fee.report.view` | R1 |
| `fee.monthly_collection` | Monthly billed, collected and outstanding by class | `fee.report.view` | R1 |
| `fee.head_wise_income` | Income by fee head | `fee.report.view` | R1 |
| `fee.outstanding_aging` | Outstanding by age bucket | `fee.defaulter.view` | R1 |
| `fee.concessions_register` | Concessions and scholarships register | `fee.concession.view` | R1 |
| `fee.cashier_summary` | Cashier session summary and variances | `fee.cashier_session.view` | R1 |
| `accounting.day_book` | Day book | `accounting.ledger.view` | R1 |
| `attendance.daily_summary` | Daily attendance summary by section | `attendance.report.view` | R1 |
| `attendance.monthly_register` | Monthly register | `attendance.report.view` | R1 |
| `attendance.chronic_absentees` | Students below attendance threshold | `attendance.report.view` | R1 |
| `attendance.late_comers` | Late arrivals | `attendance.report.view` | R1 |
| `attendance.staff_monthly` | Staff monthly attendance | `attendance.staff_attendance.view` | R1 |
| `student.strength` | Strength by class, section, gender and category | `student.student.view` | R1 |
| `student.admissions_withdrawals` | Admissions and withdrawals | `student.student.view` | R1 |
| `student.guardian_contacts` | Guardian contact list | `guardian.contact.view` | R1 |
| `result.class_result_sheet` | Class result sheet | `result.analytics.view` | R1 |
| `result.subject_analysis` | Subject analysis | `result.analytics.view` | R1 |
| `result.teacher_performance` | Results by teacher | `result.analytics.view` | R1 |
| `hr.staff_list` | Staff list | `hr.employee.view` | R1 |
| `hr.document_expiry` | Employee document expiry | `hr.document.view` | R1 |
| `leave.summary` | Leave taken and balances | `leave.report.view` | R1 |
| `notification.sms_usage` | SMS usage and cost | `notification.delivery_log.view` | R1 |
| `audit.sensitive_changes` | Changes to marks, attendance and money | `audit.log.view` | R1 |
| `payroll.salary_sheet` | Salary sheet | `payroll.report.view` | R2 |
| `payroll.deductions` | Tax, EOBI and social security deductions | `payroll.report.view` | R2 |
| `homework.compliance` | Homework compliance | `homework.report.view` | R2 |
| `assessment.quiz_summary` | Quiz summary | `assessment.analytics.view` | R2 |
| `transport.route_strength` | Students per route and stop | `transport.route.view` | R2 |
| `library.circulation` | Circulation and overdue | `library.report.view` | R3 |
| `inventory.stock_valuation` | Stock valuation | `inventory.stock.view` | R3 |

---

## 10.48 Key Screen Wireframes

### W-09 Record payment (SCR-FEE-017), desktop

```
+--------------------------------------------------------------------------------------------------+
| Fee collection . Counter 2 . Session open since 8:02 AM                       Cash in hand Rs 131,500|
+--------------------------------------------------------------------------------------------------+
| Voucher no, consumer no or student  [ V-2026-11-000412            ] (scanner ready)  [Find]       |
+------------------------------------------+-------------------------------------------------------+
| (AK) Ayesha Khan . 9-A . CS-2024-0153    | PAYMENT                                               |
| Family F-0042 (3 children)               | Amount *        [ 3,500.00 ]  (outstanding 3,500)     |
| Voucher V-2026-11-000412 . Nov 2026      | Method *        (o) Cash ( ) Cheque ( ) Bank deposit  |
| Due 10 Nov . (!) Overdue . Late fee 200  | Received from   [ Imran Khan (father)        v ]     |
|                                          |                                                       |
| Tuition            Rs 8,500              | ALLOCATION (auto, by priority)                        |
| Sibling discount  -Rs 850                | Late fee          Rs 200                              |
| Late fee           Rs 200                | Tuition           Rs 3,300                            |
| Paid earlier      -Rs 4,350              | Remaining after   Rs 0                                |
| Outstanding        Rs 3,500              |                                                       |
| Other open vouchers: none                | [ Save and print receipt  (Ctrl+Enter) ]              |
+------------------------------------------+-------------------------------------------------------+
| Last receipts: R-2026-004588 Rs 8,500 (Bilal Hussain) . R-2026-004587 Rs 24,000 (F-0042)          |
+--------------------------------------------------------------------------------------------------+
```

- Amounts above the outstanding balance ask whether to keep the excess as advance credit (D7 `fee_advance_movements`).
- Family vouchers show each child's lines; allocation follows the configured order.
- The save button carries the form's idempotency key and shows loading until the receipt is ready (D9 9.15).

### W-10 Generate vouchers, preview step (SCR-FEE-013), desktop

```
+--------------------------------------------------------------------------------------------------+
| Vouchers > Generate . December 2026                                                              |
| (1) Month and classes  >  (2) Preview  >  (3) Generate                                            |
+--------------------------------------------------------------------------------------------------+
| Class        Students  Vouchers  Gross          Discounts     Arrears       Net                  |
| Class 6          118       118   Rs 1,003,000   -Rs 61,200    Rs 84,500     Rs 1,026,300         |
| Class 9          132       129   Rs 1,254,000   -Rs 88,700    Rs 121,000    Rs 1,286,300         |
| O1                41        41   Rs 655,000     -Rs 32,000    Rs 18,000     Rs 641,000           |
| Total            291       288   Rs 2,912,000   -Rs 181,900   Rs 223,500    Rs 2,953,600         |
+--------------------------------------------------------------------------------------------------+
| Skipped (3): 2 already have a December voucher . 1 struck off              [ View skipped ]       |
| Family vouchers: 41 families combined                                                            |
| Due date 10 Dec 2026 . Valid until 31 Dec 2026                                                   |
|                                           [ Back ]   [ Generate 288 vouchers ]   (level 3 confirm)|
+--------------------------------------------------------------------------------------------------+
```

### W-11 Marks entry (SCR-EXM-010), desktop

```
+--------------------------------------------------------------------------------------------------+
| Marks > Term 1 . Physics . 10-C                       Saved 10:42 AM . Due 26 Nov . Status: Draft |
| Max: Theory 60 . Practical 15 . Total 75           Codes: ABS absent . MED medical . EXM exempt  |
+--------------------------------------------------------------------------------------------------+
| #  Student            Theory (60)   Practical (15)   Total   Grade                                |
| 1  Ayesha Khan        [ 52   ]      [ 14   ]          66     A+                                   |
| 2  Bilal Hussain      [ 38   ]      [ 12   ]          50     B                                    |
| 3  Fatima Noor        [ ABS  ]      [ 13   ]          13     F    (!) absent in theory            |
| 4  Hamza Raza         [ 63   ] (!)  [      ]                     63 is above the maximum 60       |
| ...                                                                                              |
+--------------------------------------------------------------------------------------------------+
| Entered 36 of 38 . Blank 2 . Errors 1          [ Download award list ]   [ Submit for review ]    |
+--------------------------------------------------------------------------------------------------+
```

- Tab and Enter move down the column; typing `a` in a cell offers `ABS`.
- Submit is disabled while errors exist; blanks require confirmation.
- A returned submission reopens the cells and shows the reviewer's comment at the top.

### W-12 Result batch detail (SCR-RES-006), desktop

```
+--------------------------------------------------------------------------------------------------+
| Results > Class 9 . Term 1 . Version 2                     Status: Processed . 18 Nov 2:10 PM     |
| (!) Marks changed after processing (2 papers). This batch is stale.          [ Reprocess ]        |
+--------------------------------------------------------------------------------------------------+
| Pass 112 . Compartment 6 . Fail 4 . Absent 1 . Withheld 2      Pass rate 90.3%                     |
| Student          Section  Total     %      Grade  Status        Position                         |
| Ayesha Khan      9-A      612/700   87.4   A+     Pass          1 (section) . 3 (class)          |
| Bilal Hussain    9-A      455/700   65.0   B      Compartment   Maths                             |
| Sara Ahmed       9-B      .         .      .      Withheld      fee dues (lock)                   |
| ...                                                                                              |
+--------------------------------------------------------------------------------------------------+
| [ Print report cards ]  [ Remarks ]       [ Request publication ]  (disabled: batch is stale)     |
+--------------------------------------------------------------------------------------------------+
```

### W-13 Timetable editor, section view (SCR-TTB-003), desktop

```
+--------------------------------------------------------------------------------------------------+
| Timetable > Gulberg . Morning . 2026-27 . Version 3 (draft)     View: [Section v] [9-A v]          |
| Conflicts: 1 (!)                                              [ Check ]  [ Activate version ]     |
+--------+------------+------------+------------+------------+------------+-------------------------+
| Period | Mon        | Tue        | Wed        | Thu        | Fri        | UNPLACED (9-A)          |
| 1 8:00 | English    | English    | Maths      | English    | Maths      | Physics  1 of 6 left    |
|        | Hina R.    | Hina R.    | Asad M.    | Hina R.    | Asad M.    | Urdu     0 of 5 left    |
| 2 8:40 | Maths      | Physics    | Physics(!) | Urdu       | Chemistry  |                         |
|        | Asad M.    | Kamran A.  | Kamran A.  | Saima K.   | Nadia I.   | TEACHER LOAD            |
| Break  | . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . .| Kamran A. 28 of 30      |
| 3 9:40 | ...        |            |            |            |            |                         |
+--------+------------+------------+------------+------------+------------+-------------------------+
  (!) Wed period 2: Kamran A. is also placed in 10-C. [ Move ] [ Swap ] [ Show free teachers ]
```

### W-14 Guardian voucher and online payment (SCR-FEE-037, SCR-FEE-038, SCR-FEE-039), mobile 360px

```
+------------------------------------+     +------------------------------------+
| < Fee . November 2026              |     | Verifying your payment             |
| ( Ayesha Khan, 9-A        v )      |     |                                    |
| (!) Partly paid . due 10 Nov       |     |      (spinner)                     |
| Outstanding        Rs 3,500        |     | Please don't close this page.      |
| Tuition            Rs 8,500        |     | We're confirming with JazzCash.    |
| Sibling discount  -Rs 850          |     |                                    |
| Late fee           Rs 200          |     +------------------------------------+
| Paid              -Rs 4,350        |                    |
| Consumer no 1234 5678 9012 (R2)    |                    v
| [ Pay Rs 3,500 now               ] |     +------------------------------------+
| [ Download voucher (PDF) ]         |     | (v) Payment received               |
+------------------------------------+     | Rs 3,500 . receipt R-2026-004601   |
                                           | [ Download receipt ]               |
                                           +------------------------------------+
```

### W-15 Quiz attempt (SCR-ASM-011), student mobile 360px

```
+------------------------------------+
| Maths quiz 3        (clock) 12:41  |  server timer
| Question 4 of 10          Saved v  |
+------------------------------------+
| Solve for x:  2x + 5 = 17          |
|                                    |
| ( ) x = 5                          |
| (o) x = 6                          |
| ( ) x = 11                         |
| ( ) x = 8.5                        |
+------------------------------------+
| [ < Previous ]        [ Next > ]   |
| 1 2 3 [4] 5 6 7 8 9 10             |  answered shown filled
| [ Submit quiz ]                    |
+------------------------------------+
```

- At 1 minute left the timer turns `warning`; at zero the server auto-submits (D3 `assessment:auto-submit`).
- Losing connection shows "Answers will save when you're back online"; the deadline does not pause.

### W-16 Payroll run review (SCR-PAY-007), desktop

```
+--------------------------------------------------------------------------------------------------+
| Payroll > Gulberg . November 2026 . Regular                Status: Calculated . 23 Nov 4:05 PM    |
| Staff attendance for November: locked (v)                    [ Recalculate ] [ Submit for approval ]|
+--------------------------------------------------------------------------------------------------+
| Employees 61 . Gross Rs 4,812,000 . Deductions Rs 391,400 . Net Rs 4,420,600                      |
| Employee         Designation  Days  Absent  Gross       Deductions  Net         Status            |
| Hina Raza        Teacher      22    0       Rs 85,000   Rs 6,900    Rs 78,100   Final             |
| Kamran Ali       Teacher      22    1       Rs 92,000   Rs 11,300   Rs 80,700   Final             |
| Usman Ali        Clerk        22    3 (!)   Rs 41,000   Rs 6,100    Rs 34,900   Held: query      |
| ...                                                                                              |
+--------------------------------------------------------------------------------------------------+
  Row actions: view payslip, hold or release, add one-time earning or deduction (reason required)
```

---

## 10.49 Guardian and Student Portal Summary

### Guardian portal screens

| Area | Screens |
|---|---|
| Home and account | SCR-RPT-008, SCR-GRD-010, SCR-IAM-008 to SCR-IAM-011, SCR-NTF-001, SCR-NTF-002, SCR-IAM-007 |
| Child | SCR-STU-025, SCR-STU-026, SCR-GRD-011, SCR-APR-009 |
| Attendance and leave | SCR-ATT-024, SCR-LVE-011, SCR-LVE-012 |
| Fees | SCR-FEE-036 to SCR-FEE-041 |
| Exams and results | SCR-EXM-020, SCR-RES-017, SCR-RES-018, SCR-RES-020, SCR-EXT-024, SCR-EXT-025 |
| Daily school life | SCR-TTB-010, SCR-COMM-013 to SCR-COMM-016, SCR-ADN-009 to SCR-ADN-011 |
| Learning (R2) | SCR-LMS-011, SCR-HWK-008, SCR-ASM-013 |
| Other | SCR-TRN-016, SCR-LIB-010, SCR-DSC-010 |

### Student portal screens

| Area | Screens |
|---|---|
| Home and account | SCR-RPT-009, SCR-IAM-008 to SCR-IAM-011, SCR-NTF-001, SCR-NTF-002 |
| Me | SCR-STU-024, SCR-STU-026, SCR-ATT-025, SCR-FEE-042 (if enabled) |
| Exams and results | SCR-EXM-021, SCR-RES-019, SCR-RES-020, SCR-EXT-025 |
| Daily school life | SCR-TTB-011, SCR-COMM-017, SCR-ADN-012 |
| Learning (R2) | SCR-LMS-008 to SCR-LMS-010, SCR-HWK-006, SCR-HWK-007, SCR-ASM-010 to SCR-ASM-012 |
| Other | SCR-LIB-009, SCR-DSC-011 |

Every guardian and student screen is designed at 360px first (FR-GEN-017) and uses only the permissions in D5 5.8.

---

## 10.50 Screen Counts

### Part 3 by module

| Module | Screens | Module | Screens |
|---|---|---|---|
| Fee | 42 | Learning | 11 |
| Accounting | 21 | Homework | 8 |
| Examination | 21 | Assessment | 13 |
| Result | 21 | Payroll | 14 |
| ExternalExam | 25 | Library | 11 |
| Timetable | 11 | Transport | 16 |
| Communication | 17 | Inventory | 14 |
| Administration | 12 | Discipline | 11 |
| Reporting | 9 | **Total, Part 3** | **277** |

### Whole inventory

| | R1 | R2 | R3 | Total |
|---|---|---|---|---|
| Part 2 | 214 | 10 | 1 | 225 |
| Part 3 | 129 | 108 | 40 | 277 |
| **All screens** | **343** | **118** | **41** | **502** |

Counted by each screen's first release. Reports in the catalog (30) are rendered by SCR-RPT-003 and are not counted as screens.

### Coverage checks

| Check | Result |
|---|---|
| Every module in the manifest (D8 8.5) has screens | Yes: 34 of 34 (Core, Platform and Reporting included) |
| Every portal has a home, account and notification screens | Yes |
| Every D5 approval type (5.9) has a submit screen and is decided in SCR-APR-002 | Yes |
| Every list has an empty, filtered-empty, loading and error state | By rule (10.2, D9 9.16) |
| Every permission key in the catalog is used by at least one screen, widget or action | Enforced by a new arch test, `ScreenCoverageTest`, which compares the permission catalog with route middleware, widget definitions and action authorization (D15 tracks exceptions) |
| Every functional requirement maps to screens | The full FR-to-screen traceability matrix is part of D15, built from these screen IDs |

---

## Assumptions (Part 3)

| ID | Assumption |
|---|---|
| A-85 | Fee counters use USB barcode scanners that act as keyboards; no scanner driver integration is needed. |
| A-86 | Most report needs are met by the 30 catalog reports plus saved views; custom report building is out of scope for R1 to R3. |
| A-87 | Live bus location is shown only to the guardian of an allocated student and only during trip hours. |

## Gaps Found and Added (Part 3)

| ID | Gap | Decision |
|---|---|---|
| GAP-215 | Fee counters need speed under queue pressure at month start | Keyboard-first cashier screen (DEC-163) |
| GAP-216 | Marks entry needed the same safety as attendance marking | Grid model with codes and submit lock (DEC-164) |
| GAP-217 | Dozens of reports would each need a screen | One report runner and a catalog (DEC-165) |
| GAP-218 | Guardians could see "success" from a gateway redirect before payment was confirmed | Server-verified result page (DEC-166) |
| GAP-219 | Stale results could be published after marks changed | Stale banner and disabled publication (W-12) |
| GAP-220 | Teachers had no screen for report card remarks | SCR-RES-013 with comment bank |
| GAP-221 | Permission keys could exist with no screen using them | `ScreenCoverageTest` (10.50) |

---

Deliverable 10 is complete. Reply CONTINUE for Deliverable 11
