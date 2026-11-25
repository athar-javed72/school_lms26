# Deliverable 14: Sequential Antigravity Implementation Prompts
# Part 5 of 9: PH-06 Fee and Accounting Core

**Version:** 1.0 (draft for approval) | **Date:** 24 Nov 2026
**Builds on:** D14 Parts 1 to 4, D02 (FR-FEE, FR-ACC, BR-FEE, BR-ACC), D04 4.12, D05 5.7.13, 5.7.14, 5.9 and 5.11, D07 Part 3 (7.25 to 7.27, DEC-108 to DEC-114), D09 9.19, D10 Part 3 (10.30, 10.31, 10.47, wireframes W-09, W-10, W-14), D11 Part 1 (11.4 to 11.11, 11.13, DEC-167 to DEC-177), D12 Part 2 (DEC-215, performance budgets, mutation testing)
**How to run:** start a new Antigravity conversation for each step and paste the Phase 0 Master Context Prompt before the step prompt.

---

## 14.13 New Decisions

| ID | Decision |
|---|---|
| DEC-258 | **Posting is always on.** The Accounting module is installed for every organization because `JournalPoster` and the journal tables carry every fee posting (DEC-174). The `accounting` module entitlement only shows or hides the Accounting screens (SCR-ACC-*) and their navigation. Fee screens never depend on that entitlement. |
| DEC-259 | **Finance defaults come from an onboarding step.** Accounting adds a "Finance defaults" step to `OnboardingStepRegistry`. It creates the first fiscal year (default 1 July to 30 June), seeds the chart of accounts template with every system role (D07 7.27 plus DEC-176), and sets `accounting.head_office_branch_id` (DEC-114). Fee seeds default fee heads in the same step. No fee action can run before the step is complete; such actions fail with `fee.setup_incomplete`. |
| DEC-260 | **Late fee rules get their own table.** `fee_late_fee_rules` [T][S], alias `flr`. Columns: `branch_id`? FK->branches, `branch_key` GENERATED, `name`, `late_fee_head_id` FK->fee_heads (head type `fine`), `mode` VARCHAR(20) (`flat`, `per_day`), `amount` DECIMAL(15,2), `grace_days` TINYINT UNSIGNED, `cap_amount` DECIMAL(15,2)?, `voucher_types` JSON, `effective_from` DATE, `effective_to` DATE?, `is_active`. Key `ix_flr (organization_id, branch_key, effective_from)`. CHECK `amount >= 0`. |
| DEC-261 | **New permission `admission.application.override_payment`** (OW, PR). It lets an approved application be converted before its admission voucher is paid. The reason is mandatory and audited. It is used by the Fee binding of `AdmissionPaymentGate` (DEC-248). |
| DEC-262 | **Holds come from pending requests.** "Reversal pending", "refund pending" and similar holds are not stored as columns. `PendingApprovalQuery::hasPending(type, targetType, targetId)` (Approval public query) answers them from pending or returned requests whose snapshot targets the record. Actions that conflict with a hold call it and refuse with `approval.hold_active`. |
| DEC-263 | **A basic report runner comes before P09.3.** P06.6 builds the Reporting `ReportDefinition` contract, `ReportRegistry` and a basic SCR-RPT-003 (filters, table, totals, print, Excel and PDF export with masking). Finance needs these reports for the pilot. P09.3 adds charts, saved views, the catalog screen (SCR-RPT-002), branch comparison and snapshots on the same contract. |
| DEC-264 | **`journal_lines.party_type` gains `family`** for receivable lines of family vouchers (D11 11.11 "Party" control). The family ledger (SCR-FEE-029) reads both `family` lines and the children's `student` lines. |
| DEC-265 | **Gapless receipts.** `NumberingService::nextGapless(key, scope)` locks the counter row (`SELECT ... FOR UPDATE`) inside the payment transaction. A rolled-back payment therefore never consumes a number (A-60). Only `fee_receipt` (per branch and fiscal year) and `journal_entry` (per fiscal year) use gapless mode in R1. |

### Corrections to earlier deliverables

| Where | Change |
|---|---|
| D07 Part 3 7.26 | Adds `fee_late_fee_rules` (DEC-260) |
| D07 Part 3 7.27 | `journal_lines.party_type` gains `family` (DEC-264) |
| D05 5.7.7 | Adds `admission.application.override_payment` (OW, PR) (DEC-261) |
| D13 13.5 P06.2 | Installment plans move to P06.5 with the other corrections |
| D14 Part 4 (P04.4, P05.1, P05.2, P05.3, P05.4) | Aligned with D07 Part 3 DEC-108: `teaching_group_members` is owned by Student; `department_heads` uses alias `dhd`; there is no `departments.head_employee_id` or `branches.principal_employee_id`; attendance rows use soft `source_ref_type` and `source_ref_id` instead of `leave_application_id`. The project copy of Part 4 is updated. |

---

## 14.14 PH-06 Fee and Accounting Core

**Exit criteria:**
- MS-07 is reached.
- 10,000 vouchers generate in under 10 minutes.
- Posting and calculation classes reach a mutation score of at least 80%.
- Integrity checks report zero mismatches on the demo dataset.

### P06.1 Accounting Core

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 3 7.27 with DEC-110 and DEC-114; D11 Part 1 11.1 (DEC-173, DEC-174, DEC-176 and the entry_type corrections), 11.4 (journal_entry), 11.11 (controls); D05 5.7.14 and SOD-03; D10 Part 3 10.31 (SCR-ACC-001 to 011, 013, 014); D02 FR-ACC and BR-ACC; D13 RISK-10; D14 DEC-258, DEC-259, DEC-264, DEC-265.

PHASE GOAL
Build the double-entry core that every money action posts through: fiscal years, chart of accounts with protected system role accounts, bank accounts linked to branch bank accounts, journal entries and lines, the single JournalPoster, manual journals with approval, reversals, accounting period locks, opening balances, ledgers, cash and bank books, trial balance, day book data, fiscal year close, and the finance onboarding step.

ALREADY BUILT
PH-00 to PH-05: tenancy, identity and RBAC with SOD framework, audit, Core services (numbering, period locks, idempotency, bulk operations), approval, notification, import and export, document engines, academic structure, students, guardians, families, admission with AdmissionPaymentGate (default binding), HR, attendance, leave.

TASKS
1. Migrations (Accounting): fiscal_years, accounts, bank_accounts, journal_entries, journal_lines exactly as D07 7.27, with entry_type values extended per D11 11.1 (fee_late_fee, fee_voucher_cancellation, fee_deferred_recognition, fee_adjustment, cheque_clearance, cash_variance, deposit_forfeiture), system_role values extended per DEC-176, and party_type including family (DEC-264). CHECK constraints ck_jln_side and the header CHECK on totals.
2. Money: Core\Support\Money value object on brick/math BigDecimal (scale 2, rounding never implicit; arithmetic refuses mixed scales); Eloquent cast MoneyCast for DECIMAL(15,2) columns. Floats are forbidden in money code (Larastan rule and an architecture test).
3. JournalPoster (Modules\Accounting\Services\JournalPoster, the only writer of journal_entries and journal_lines per RISK-10): post(JournalDraft): validates lines (posting accounts only, never is_group; every line has branch_id; party on receivable, advance and deposit accounts), sums debits and credits with Money and refuses imbalance (BR-ACC-01), resolves the fiscal year by entry_date and refuses closed years and locked accounting months through PeriodLockService module accounting (BR-ACC-03), takes entry_no from NumberingService::nextGapless(journal_entry, fiscal year) (DEC-265), inserts header and lines, and is idempotent through uq_jen_source (a retry with the same source_type, source_id and entry_type returns the existing entry). It must be called inside the caller's transaction (DEC-174); calling it outside a transaction throws. reverse(entry, reason, date today): mirror lines, reversal_of_entry_id, original status reversed (DEC-173). An architecture test fails any write to the journal tables outside JournalPoster.
4. SystemAccountResolver: account for a system_role (uq_acc_role); missing role throws accounting.system_account_missing with the role name. Fee and Payroll use only this resolver or explicit ledger_account_id columns.
5. Chart of accounts template as code data (Definitions/Templates/ChartOfAccounts.php): groups for assets, liabilities, equity, income, expenses with Pakistani school accounts (cash in hand, banks, cheques in hand, fee receivable, advance fees, security deposits, exam fees payable, deferred fee income, fee tax payable, tuition income, admission income, annual charges income, lab fee income, late fee income, discount allowed, scholarship allowed, fee write-off expense, cash over and short, opening balance equity, salaries payable, salary expense), every DEC-176 role assigned; Urdu names; editable after seeding except system accounts' type and role.
6. Finance onboarding step (DEC-259) in OnboardingStepRegistry: fiscal year dates (default 1 July to 30 June, A-123), chart template applied (idempotent by codes), head-office branch chosen, fiscal year created open.
7. Actions: CreateFiscalYear (no overlap), CreateAccount, UpdateAccount (used accounts cannot change type; system accounts locked), DeactivateAccount (refused with a balance), CreateBankAccount (linked to a branch bank account and a ledger account), CreateManualJournal (draft; balanced lines with branch per line and optional party; post directly below setting accounting.journal_approval_threshold, otherwise SubmitApprovalRequest journal_entry), PostManualJournal, RequestJournalReversal (approval journal_entry with kind reversal), SaveOpeningBalances (must balance; entry_type opening; once per fiscal year per account and branch; later changes through manual journals), CloseFiscalYear (checks: all journals posted, all months locked, trial balance balanced; level 3 confirm; status closed; next year must exist), LockAccountingPeriod (PeriodLockService module accounting with lock check: no draft journals in the month).
8. Approval type journal_entry (Accounting registers): step 1 owner or org_admin (any); approve permission accounting.journal.approve; SOD-03 (creator cannot approve) registered; handler posts or reverses through JournalPoster; draft version snapshot.
9. Queries for reports: LedgerQuery (account, branch, date range: opening, movements, closing), CashBookQuery (cash_in_hand and each bank ledger per day), TrialBalanceQuery (as of date, by branch or consolidated, groups rolled up by path), DayBookQuery (entries by date). All read from the replica connection (D12 DEC-201) and use ix_jln_ledger.
10. Screens (D10 10.31 exactly): SCR-ACC-001 Fiscal years, SCR-ACC-002 Close fiscal year, SCR-ACC-003 Chart of accounts (tree with balances, system badge), SCR-ACC-004 Account form, SCR-ACC-005 Journals (filters type, date, status, source), SCR-ACC-006 Journal entry form (live debit and credit totals, branch per line, party picker; templates "Cash deposit" for F19 and "Exam fee remittance" for F17), SCR-ACC-007 Journal detail (source document link through a SourceLinkRegistry that modules fill), SCR-ACC-008 Reverse journal, SCR-ACC-009 General ledger, SCR-ACC-010 Cash and bank book, SCR-ACC-011 Trial balance, SCR-ACC-013 Opening balances, SCR-ACC-014 Bank accounts. SCR-ACC-012 Receivables arrives in P06.6.
11. Definitions: Permissions.php for accounting.* per D05 5.7.14; Settings accounting.head_office_branch_id, accounting.journal_approval_threshold (default Rs 100,000), accounting.fiscal_year_start_month (7); Navigation (Finance group: Journals, Ledgers, Trial balance, Chart of accounts, Fiscal years) shown only with the accounting entitlement (DEC-258).
12. decisions.md rows: DEC-110, DEC-114, DEC-173, DEC-174, DEC-176, DEC-258, DEC-259, DEC-264, DEC-265.

DATABASE
fiscal_years (fyr), accounts (acc), bank_accounts (bnk), journal_entries (jen), journal_lines (jln) per D07 Part 3 7.27 with the D11 11.1 and DEC-264 corrections.

BUSINESS RULES
- Every journal balances (BR-ACC-01) and is unique per source event (DEC-110).
- Entries fall in an open fiscal year and an unlocked accounting month (BR-ACC-03).
- Every line carries a branch; organization-level lines use the head-office branch (DEC-114).
- Posted journals are never edited or deleted; corrections are reversals dated today (DEC-173).
- System accounts cannot be deleted, deactivated, or have their type or role changed.
- The creator of a manual journal cannot approve it (SOD-03).

UI SPEC
- Journal form grid: account picker with search by code and name, debit and credit columns (one side per line), branch select defaulting to the active branch, party picker for sub-ledger accounts, live totals and difference in danger color until zero; Save disabled until balanced.
- Chart tree with expand and collapse, balances right-aligned in tabular figures, system accounts with a lock badge.
- Reports printable (A4) and exportable; amounts formatted with Pakistani grouping per the user's locale setting.

SECURITY AND PERMISSIONS
- Routes and permissions per D10 10.31; accounting screens need the accounting entitlement plus the permission.
- Manual journals above the threshold go through approval; every post and reversal audited.
- Database user privileges per DEC-190 (application user cannot alter journal tables' structure).

TESTS
- JournalPoster: imbalance refused; group account refused; missing branch refused; closed fiscal year and locked month refused; idempotent retry returns the same entry; called outside a transaction throws; reversal mirrors lines and links both ways.
- Gapless entry numbers: a forced rollback after numbering leaves no gap (two sequential successful posts get consecutive numbers).
- Money: mixed-scale arithmetic refused; no float casts in money classes (arch test).
- SystemAccountResolver throws a named error for a missing role.
- Manual journal: below threshold posts; above threshold creates journal_entry approval; creator cannot approve (SOD-03).
- Trial balance balances on a seeded dataset; ledger opening plus movements equals closing.
- Fiscal year close blocked with an unlocked month or draft journal.
- Architecture test: no class outside JournalPoster writes journal_entries or journal_lines.
- Screen tests, audit, isolation suite, RBAC fixture, ScreenCoverageTest; mutation testing (Infection) on JournalPoster, Money and SystemAccountResolver with MSI at least 80%.

ACCEPTANCE CRITERIA
1. On staging, the finance onboarding step creates the fiscal year and chart; a manual journal above the threshold is approved by the owner and appears in the ledger and trial balance.
2. A locked accounting month refuses a posting with core.period_locked.
3. Mutation score evidence for posting classes at least 80%.
4. CI green; staging green.

DO NOT
- Do not let any module insert journal rows directly.
- Do not use floats for money anywhere.
- Do not build expenses, purchases, vendors, bank reconciliation or financial statements (R2).

OUTPUT
Report per Master section 20 with the JournalPoster contract, the mutation report summary and trial balance screenshots.
```

### P06.2 Fee Setup

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 3 7.26 (fee_heads, fee_tax_rules, fee_structures, fee_structure_items, fee_discount_rules, student_fee_concessions, scholarships); D11 Part 1 11.4 (fee_concession, scholarship), 11.5 rules and 11.11 F1a to F1f; D05 5.7.13 and SOD-08; D10 Part 3 SCR-FEE-001 to 009; D02 FR-FEE and GAP-09, GAP-10 from D01; D06 6.6 (new session wizard step for fee structures); D14 DEC-246, DEC-260.

PHASE GOAL
Make every fee amount and rule data: fee heads with ledger accounts and allocation priority, tax rules, versioned fee structures per session, class, section and category with vacation advance months, discount rules (sibling, early payment, category), late fee rules, concessions and scholarships through approval, security deposit heads, and the calculator that turns these into voucher lines.

ALREADY BUILT
PH-00 to PH-05 and P06.1: accounting core with JournalPoster and system accounts, students with categories and sponsors, families, sessions and classes, approval engine.

TASKS
1. Migrations (Fee): fee_heads, fee_tax_rules, fee_structures, fee_structure_items, fee_discount_rules, student_fee_concessions, scholarships exactly as D07 7.26, and fee_late_fee_rules per DEC-260.
2. Default fee heads seeded by the finance onboarding step (editable): TUITION (monthly), ADMISSION (one_time), ANNUAL (annual), EXAM (on_demand), LAB (monthly), SECURITY (refundable_deposit), LATE_FEE (fine), BOARD_FEE and CAMBRIDGE_ENTRY (pass_through), each mapped to an account from the chart template; allocation priority fines and late fees first (FR-FEE-018).
3. Actions: CreateFeeHead and UpdateFeeHead (ledger account type must match head type: income for charges, liability for refundable_deposit and pass_through), CreateTaxRule and UpdateTaxRule (rates and thresholds are data, GAP-10), CreateFeeStructure (draft), UpdateFeeStructure (items with amount, applicable months, collect_with_month for vacation advance, due month for annual heads), ActivateFeeStructure (one active structure per branch, class, section key, category key and effective date per uq_fst), NewStructureVersion (a structure used by any voucher can only get a new version, never be edited), CopyStructuresToSession (percentage or fixed increase with rounding to the nearest rupee; preview per class), CreateDiscountRule and UpdateDiscountRule (sibling by position within family and sibling_scope; early payment percent if paid by a date; category discount by student category), CreateLateFeeRule, RequestConcession (permission fee.concession.create; SubmitApprovalRequest fee_concession), RequestScholarship (fee.scholarship.create; sponsor optional; SubmitApprovalRequest scholarship), RevokeConcession and EndScholarship (reason; effective from the next voucher).
4. Approval types (Fee registers): fee_concession (1 principal_of_branch; 2 owner when percent above fee.concession_owner_threshold_percent (default 25) or a fixed amount above fee.concession_owner_threshold_amount; approve permission fee.concession.approve; SOD-08 for the approver's own linked child; handler activates the concession; for issued unpaid vouchers in its months it raises an attention item for the accountant to regenerate, and for partly paid ones it offers the concession_after_issue adjustment built in P06.5) and scholarship (1 principal_of_branch; 2 owner; same handler shape, posts to scholarship_allowed through the calculator line type).
5. FeeCalculator (Modules\Fee\Services\FeeCalculator, pure and deterministic, no writes): inputs student, enrollment, fee month, voucher type, issue date; resolves the active structure (section and category specific first, then class-wide); produces lines: charges for heads applicable in that month plus vacation advance lines (collect_with_month) with their own fee_month; discount lines from discount rules (sibling position computed from family membership and active enrollments within sibling_scope; category discount); concession lines; scholarship lines; tax lines per tax rule with annual threshold checked against the student's taxable charges in the fiscal year; refundable deposit lines on admission vouchers; pass-through lines; returns a VoucherDraft with totals using Money. Every line carries source_ref_type and source_ref_id when it comes from a concession, scholarship or rule.
6. Registries for later contributors (used in P06.3): VoucherLineContributorRegistry contract defined here (contribute(student, month): lines) with no contributors yet; Transport, Library and ExternalExam register later.
7. Integrations promised earlier: SessionCopyStepRegistry step "Fee structures with increase" (DEC-246); family merge and split fee preview provider for SCR-GRD-006 (P04.1) showing the sibling discount and family voucher effect; admission conversion section on SCR-ADM-006 to choose the fee structure category and request a concession at conversion.
8. Screens (D10 10.30 exactly): SCR-FEE-001 Fee heads, SCR-FEE-002 Tax rules, SCR-FEE-003 Fee structures, SCR-FEE-004 Fee structure editor (heads as rows, months as columns, vacation advance mapping, version banner when used), SCR-FEE-005 Copy structures to a session, SCR-FEE-006 Discount rules (preview against a sample family), SCR-FEE-007 Concessions, SCR-FEE-008 Request concession (monthly effect shown; approval threshold note), SCR-FEE-009 Scholarships. Late fee rules are edited on a tab of SCR-FEE-001 (route app.fee.late_fee_rules.index, same permission fee.head.update).
9. Definitions: Permissions.php for fee.head, fee.structure, fee.discount_rule, fee.concession, fee.scholarship per D05 5.7.13; Settings fee.concession_owner_threshold_percent, fee.concession_owner_threshold_amount, fee.due_day (10), fee.prorata_admission (off, A-89), fee.sibling_scope default; Navigation (Fees group: Heads, Structures, Discounts, Concessions, Scholarships).
10. decisions.md rows: DEC-111 (preparation), DEC-170 (calculator side), DEC-260, A-88, A-89.

DATABASE
fee_heads (fhd), fee_tax_rules (ftx), fee_structures (fst), fee_structure_items (fsi), fee_discount_rules (fdr), student_fee_concessions (sfc), scholarships (sch), fee_late_fee_rules (flr) per D07 Part 3 7.26 and DEC-260.

BUSINESS RULES
- No fee amount, rate or rule lives in code; everything comes from these tables and settings.
- A used structure is never edited; a new version starts on a new effective date.
- Concessions and scholarships take effect only after approval and only from the next voucher; issued vouchers change through regeneration or adjustment.
- A user cannot approve a concession, scholarship or refund for their own linked child (SOD-08).
- Late fee waivers are concessions on the late fee head, never line edits (D11 11.5).

UI SPEC
- Structure editor grid with sticky head column, month columns from the session's months, totals per month and per year, vacation advance shown as "collected with May" chips.
- Discount rule preview: pick a family, see each child's lines before and after.
- Concession request shows the monthly effect in rupees and which approvers apply.

SECURITY AND PERMISSIONS
- Routes and permissions per D10 10.30; concession and scholarship approvals enforce SOD-08.
- Structure changes and rule changes audited with old and new values.

TESTS
- FeeCalculator unit matrix: structure precedence (section and category specific over class-wide); applicable months; vacation advance lines with future fee_month; sibling discount by position and scope; category discount; concession percent and fixed; scholarship; tax with annual threshold crossing in the month; refundable deposit on admission voucher; pass-through head; totals exact with Money; determinism (same inputs, same output).
- Structure versioning: editing a used structure refused; new version effective next month.
- Copy to session with 10% increase rounds correctly and previews per class.
- Approvals: concession above 25% adds the owner step; approver who is the student's guardian blocked (SOD-08); approval activates from the next voucher.
- Screen tests, audit, isolation suite, RBAC fixture, ScreenCoverageTest; mutation testing on FeeCalculator with MSI at least 80%.

ACCEPTANCE CRITERIA
1. On staging, a branch has structures for national and Cambridge classes with vacation advance; the calculator preview for a family of three shows correct sibling discounts.
2. A 30% concession passes principal and owner and appears in the next calculation.
3. Mutation score for FeeCalculator at least 80%.
4. CI green; staging green.

DO NOT
- Do not generate vouchers or post journals here (P06.3).
- Do not hardcode head codes, rates or thresholds in business logic.
- Do not build installment plans here (P06.5).

OUTPUT
Report per Master section 20 with the calculator test matrix, the mutation summary and structure editor screenshots.
```

### P06.3 Vouchers

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D11 Part 1 11.5 completely, DEC-170, DEC-171, and the posting rules F1 to F4 and F1g in 11.11 with the worked example; D07 Part 3 7.26 (fee_vouchers, fee_voucher_items, fee_advance_movements, student_fee_accounts) with DEC-111 and DEC-112; D09 9.19 (3-copy challan); D10 Part 3 SCR-FEE-011 to 016, 036, 037, 042 and wireframes W-10 and W-14; D12 Part 2 DEC-215; D14 DEC-248, DEC-261.

PHASE GOAL
Issue correct vouchers fast and post them correctly: monthly generation by class as a bulk job with preview, family vouchers, arrears memo lines, advance credit applied, line contributors, deferred income for future months, ad hoc and admission vouchers, cancellation and regeneration with mirror postings, overdue marking and late fees, pre-rendered 3-copy challans, guardian and student voucher views, and the Fee binding of the admission payment gate.

ALREADY BUILT
PH-00 to PH-05, P06.1 and P06.2: JournalPoster, system accounts, fee heads, structures, discount and late fee rules, concessions, scholarships, FeeCalculator, VoucherLineContributorRegistry, document engine with batch rendering.

TASKS
1. Migrations (Fee): fee_vouchers, fee_voucher_items, fee_advance_movements, student_fee_accounts exactly as D07 7.26 (ck_fvc_payer, generated outstanding_amount, uq_fvi_recurring with is_live and recurring_marker). The FK admission_applications.admission_voucher_id -> fee_vouchers is added here.
2. VoucherBuilder: turns a FeeCalculator draft plus contributor lines into voucher and items; adds the arrears memo line (sum of outstanding of the payer's earlier live vouchers; never posted, DEC-171); applies available advance credit (fee_advance_movements negative row, F1g); family mode when families.voucher_mode = family (one voucher per family, lines name each child, uq_fvi_recurring still per student); voucher_no from NumberingService key fee_voucher (per branch and month); due_date from fee.due_day; valid_until end of month.
3. Posting (FeeVoucherPoster, inside the same transaction, DEC-174): F1 per voucher: Dr fee_receivable (party student or family per DEC-264) for the net of posted lines; Dr discount_allowed for discount and concession lines; Dr scholarship_allowed for scholarship lines; Cr each charge line's head account for current-month lines and deferred_fee_income for future fee_month lines (DEC-170); Cr tax liability for tax lines; Cr security_deposit_liability for deposit lines; Cr exam_fees_payable for pass-through lines; F1g Dr advance_fees_liability Cr fee_receivable for advance applied. Voucher stores accrual_journal_entry_id.
4. Actions: GenerateVouchers (wizard input month, classes, due date; preview counts and totals per class with skipped students and reasons: already has a voucher for the month (uq_fvi_recurring), struck off, withdrawn, no structure; run as bulk operation on queue bulk in chunks of 100 students, each chunk one transaction including postings; VouchersBatchGenerated event per class), GenerateStudentVoucher (single, same rules), IssueAdHocVoucher (types admission, fine, exam_entry, readmission, ad_hoc; heads and amounts; posts F1), CancelVoucher (unpaid only; payments must be reversed first; items is_live = 0 in the same transaction; F4 mirror entry dated today, reversing deferred lines from income when already recognized; reason; level 2), RegenerateVoucher (cancel plus new voucher with superseded_by_id; used after a concession activates), RecognizeDeferredFees (command fee:recognize-deferred monthly day 1 at 00:45 organization time; F2 one entry per branch per month with a line per head), MarkOverdue (command fee:mark-overdue daily 00:15; issued and partially paid past due_date become overdue), ApplyLateFees (command fee:apply-late-fees daily 00:30; per fee_late_fee_rules: flat after grace or per day capped; adds a late_fee item, updates late_fee_amount and net, posts F3 per voucher; idempotent per voucher and day).
5. StudentFeeAccountService: recalculates student_fee_accounts (outstanding, advance, oldest unpaid month, unpaid months count) after every voucher and payment change (listener) and nightly verification job fee:verify-accounts (DEC-112) that reports differences without fixing them silently.
6. Admission (DEC-248, DEC-261): Fee binds AdmissionPaymentGate: on DecideApplication approve, an admission voucher is issued (admission heads plus the joining month per structure, pro rata only if fee.prorata_admission); conversion requires the voucher paid, or admission.application.override_payment with reason; lapse (P04.2 job) cancels the unpaid admission voucher through CancelVoucher.
7. Documents (DEC-215): document type fee_voucher (printable, Fee data provider): 3-copy challan on A4 landscape (bank, school, student copies) per D09 9.19 with voucher number barcode (Code 128), lines, arrears memo, due date, amount after due date with the late fee rule, bank account details from the branch default bank account, Urdu and English labels. Generation pre-renders PDFs per class through RenderDocumentBatch on queue documents; SCR-FEE-016 serves stored files and re-renders only on regeneration.
8. Notifications: fee.vouchers_issued (guardians of student; family vouchers to the family's primary guardian once; amount and due date; mandatory per DEC-257), fee.voucher_overdue (sent by the reminders command in P06.6).
9. Screens (D10 10.30 exactly): SCR-FEE-011 Vouchers (filters month, class, status, type, family; totals row), SCR-FEE-012 Voucher detail (lines, allocations, payments, accrual journal link, history drawer; actions by permission), SCR-FEE-013 Generate vouchers (W-10, level 3 confirm with the count), SCR-FEE-014 Issue ad hoc voucher, SCR-FEE-015 Cancel or regenerate voucher, SCR-FEE-016 Print vouchers, SCR-FEE-036 Guardian: vouchers, SCR-FEE-037 Guardian: voucher detail (W-14 without the online payment button, which is R2), SCR-FEE-042 Student portal: fees (only when fee.student_portal_enabled, A-43). guardian.home section "Fees due" per child through HomeSectionRegistry.
10. Performance: generation of 10,000 vouchers (S2 profile) in under 10 minutes: chunked eager loading of structures, rules, families and balances; one calculator call per student; batched inserts of items inside each chunk transaction; postings through JournalPoster per voucher; measured in tests/Performance/VoucherGenerationTest.
11. Definitions: Permissions.php for fee.voucher per D05 5.7.13; Settings fee.student_portal_enabled, fee.voucher_valid_days; Navigation (Fees group: Vouchers, Generate, Print).
12. decisions.md rows: DEC-111, DEC-112, DEC-170, DEC-171, DEC-215, DEC-248 (binding), DEC-261.

DATABASE
fee_vouchers (fvc), fee_voucher_items (fvi), fee_advance_movements (fav), student_fee_accounts (sfa) per D07 Part 3 7.26; FK admission_applications.admission_voucher_id.

BUSINESS RULES
- Voucher amounts are frozen at generation except late fees added before payment (BR-FEE-02).
- One live recurring charge per student, head and month across individual and family vouchers (DEC-111).
- Arrears lines are memo only (DEC-171); future-month lines credit deferred income (DEC-170).
- Vouchers with payments cannot be cancelled until the payments are reversed.
- Cancellation and regeneration post mirror entries dated today (DEC-173).
- Every voucher has exactly one fee_voucher journal; every cancelled one also has a cancellation journal.

UI SPEC
- W-10 preview with per-class rows, skipped list with reasons, family voucher count, due date and validity.
- Voucher detail shows the printed total, the posting link and a timeline (issued, late fee, payments, cancellation).
- Challan PDF prints legibly on A4 landscape with three equal copies and a cut line; Urdu labels rendered with the Nastaliq font from the spike.

SECURITY AND PERMISSIONS
- Generation needs fee.voucher.generate; cancellation fee.voucher.manage with reason; guardians see only their children's vouchers (EnsureChildContext).
- Voucher PDFs downloaded through signed URLs after the policy check.

TESTS
- Named posting tests for F1 (with the D11 worked example: Ayesha Khan December voucher posts Dr receivable 16,650, Dr discount 850, Cr tuition 8,500, Cr lab 500, Cr deferred 8,500), F1a to F1g, F2 (recognition on 1 January), F3 (late fee 200 after 10 December), F4 (cancellation mirror, including already-recognized deferred lines).
- uq_fvi_recurring blocks a second December tuition for the same student through a family voucher.
- Generation preview skips with correct reasons; idempotent rerun creates nothing new; chunk failure leaves other chunks committed and lists failed students.
- Advance applied at generation reduces net and posts F1g.
- Late fees: flat after grace, per day capped, idempotent per day.
- Admission gate: conversion blocked until paid; override with permission and reason works and is audited.
- student_fee_accounts recalculated after generation and cancellation; verify job reports a forced mismatch.
- Performance: 10,000 vouchers under 10 minutes on S2 (evidence).
- Screen tests including guardian views, audit, isolation suite, RBAC fixture, ScreenCoverageTest; mutation testing on VoucherBuilder and FeeVoucherPoster with MSI at least 80%.

ACCEPTANCE CRITERIA
1. On staging, December vouchers generate for a branch with family vouchers, print as 3-copy challans and appear in the guardian portal.
2. The D11 worked example posts exactly as documented.
3. 10,000-voucher timing evidence under 10 minutes.
4. CI green; staging green.

DO NOT
- Do not record payments here (P06.4).
- Do not post arrears lines.
- Do not build online payment or 1Bill consumer numbers (R2).

OUTPUT
Report per Master section 20 with the posting test list, generation timing and sample challan PDFs.
```

### P06.4 Counter Collection

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D11 Part 1 11.6 completely (counter flow, cashier day close, cheque lifecycle), 11.4 (cash_variance), 11.11 F5 to F8, F11 for bounces, F18 and F19; D07 Part 3 7.26 (fee_payments with ck_fpy_sign and its unique keys, fee_payment_allocations, cashier_sessions) with DEC-113; D05 5.7.13 and SOD-02; D10 Part 3 SCR-FEE-017 to 019, 021 to 026, 040, DEC-163 and wireframe W-09; D12 Part 1 money controls; D14 DEC-262, DEC-265.

PHASE GOAL
Collect fees at the counter safely and fast: cashier sessions with float and denomination close, keyboard-first payment entry with server-side allocation, gapless receipts, cash, cheque and bank deposit methods, bank challan entry and file import, cheque clearing and bouncing, cash variance through approval, receipts and SMS, and the guardian payment history.

ALREADY BUILT
PH-00 to PH-05, P06.1 to P06.3: JournalPoster, vouchers with postings, student_fee_accounts, fee_advance_movements, document engine, approval and notification engines.

TASKS
1. Migrations (Fee): fee_payments, fee_payment_allocations, cashier_sessions exactly as D07 7.26 (ck_fpy_sign, uq_fpy_receipt, uq_fpy_idem, uq_fpy_reversal, uq_fpy_gateway, uq_csn_open). payment_gateway tables are R2.
2. AllocationService: locks the payer's open live vouchers FOR UPDATE; order per setting fee.allocation_order (default: vouchers oldest first; within a voucher by head allocation_priority, fines and late fees first); never exceeds outstanding_amount (BR-FEE-04); returns allocations per voucher item and any excess.
3. Actions: OpenCashierSession (one open session per user and branch, uq_csn_open; opening float), RecordPayment (idempotency key required, FR-GEN-013, uq_fpy_idem returns the first receipt on retry; method cash needs an open session of the recorder in that branch (BR-FEE-06); cheque needs number, bank and date and sets cheque_status pending; bank_deposit needs bank account and reference; received_on defaults to today and must be in an unlocked fee period and accounting month; amount and allocations computed only on the server from vouchers, never from client totals; transaction: allocation, receipt_no via NumberingService::nextGapless(fee_receipt, branch and fiscal year) per DEC-265, payment row, allocations, advance movement for excess after explicit confirmation, voucher paid_amount and status updates, posting F5, F6 or F7, audit fee.payment.recorded; after commit PaymentRecorded), CloseCashierSession (denomination count; expected = float + cash payments - counter cash refunds; zero variance closes the session; otherwise status variance_pending and SubmitApprovalRequest cash_variance), MarkChequeCleared (F8, status cleared), BounceCheque (status bounced; calls ReversePayment internally without approval because the bank return is evidence (D11 11.6), posts F11, reopens vouchers, optionally issues a cheque bounce fine ad hoc voucher per setting fee.cheque_bounce_fine; guardian notified), RecordBankChallan (grid of voucher number, amount, bank date and reference; each row a RecordPayment with method bank_deposit), ImportBankChallanFile (import type bank_challans through the import engine: mapping per bank saved as import mappings; match by voucher number and amount; amount mismatch, unknown voucher and duplicate bank reference (ix_fpy_bankref) go to unmatched for manual review; matched rows commit through RecordPayment).
4. Approval type cash_variance (Fee registers): steps 1 accountant_of_branch (SOD-02: not the cashier of that session), 2 principal_of_branch when the absolute variance is above fee.cash_variance_principal_threshold (default Rs 1,000); handler posts F18 (shortage Dr cash_over_short Cr cash_in_hand, excess the mirror) and closes the session as closed_with_variance; return sends the cashier to recount.
5. Documents: fee_receipt (printable; thermal 80 mm and A5 layouts by setting; lists every voucher and head settled, method, cashier, received from; reprints marked "Duplicate" and audited) and cash_summary (session close print with denominations).
6. Notifications: fee.payment_received (SMS receipt to the primary guardian with amount, receipt number and remaining balance; mandatory), fee.cheque_bounced.
7. Attention items: fee.cashier_session_open_overnight (sessions open past midnight, D10 10.7), fee.cash_variance_pending.
8. Screens (D10 10.30 exactly): SCR-FEE-017 Record payment (W-09; DEC-163 keyboard-first: scanner input in the voucher field, Enter loads, amount defaults to outstanding, Ctrl+Enter saves and prints; other open vouchers listed; excess confirmation dialog), SCR-FEE-018 Payments, SCR-FEE-019 Receipt, SCR-FEE-021 Cheques, SCR-FEE-022 Bank challan entry, SCR-FEE-023 Bank challan file import, SCR-FEE-024 Cashier sessions, SCR-FEE-025 Open cashier session, SCR-FEE-026 Close cashier session, SCR-FEE-040 Guardian: payment history (receipt download).
9. Definitions: Permissions.php for fee.payment, fee.cheque, fee.bank_challan, fee.cashier_session, fee.cash_variance per D05 5.7.13; Settings fee.allocation_order, fee.cash_variance_principal_threshold, fee.cheque_bounce_fine, fee.receipt_layout; Navigation (Fees group: Collect, Payments, Cheques, Bank challans, Cashier sessions).
10. decisions.md rows: DEC-113, DEC-163, DEC-175, DEC-262 (preparation), DEC-265.

DATABASE
fee_payments (fpy), fee_payment_allocations (fpa), cashier_sessions (csn) per D07 Part 3 7.26.

BUSINESS RULES
- A payment is recorded only from the server's view of vouchers; no client-sent status or total is trusted.
- Allocations never exceed a voucher's outstanding amount (BR-FEE-04); excess becomes advance only after confirmation.
- Cash needs an open cashier session (BR-FEE-06); one open session per cashier per branch.
- Receipts are gapless per branch and fiscal year (DEC-265, A-60).
- Cheques are not cash until cleared (DEC-175); a bounce reverses the payment without approval.
- A cashier cannot approve their own variance (SOD-02); sessions are never reopened.
- Payments dated in a locked fee period or accounting month are refused.

UI SPEC
- W-09 two-panel layout: voucher and lines on the left, payment and live allocation on the right; cash in hand counter in the header; last receipts strip at the bottom.
- A routine cash payment completes in under 30 seconds with three keystrokes after the scan (DEC-163).
- Close session form with denomination rows (5000, 1000, 500, 100, 50, 20, 10, coins) and live expected versus counted.

SECURITY AND PERMISSIONS
- The save button carries the idempotency key; double submits return the same receipt.
- Impersonated sessions cannot record or reverse payments (ImpersonationGuard, money actions).
- Every payment, bounce and variance audited; receipt reprints audited.

TESTS
- Named posting tests F5 (cash), F6 (bank deposit), F7 (cheque), F8 (clearance), F11 (bounce), F18 (shortage and excess), F19 template.
- D11 worked example: Rs 20,350 on 12 December allocates 3,500 to November and 16,850 to December in priority order and both vouchers become paid.
- Concurrency: two simultaneous payments on the same voucher never over-allocate (row lock test with two connections).
- Idempotency: same key returns the first receipt; different key with same content creates a second payment only after outstanding allows it.
- Gapless receipts under a forced rollback.
- Cash without an open session refused; second open session refused (uq_csn_open).
- Variance: approver who is the cashier blocked (SOD-02); threshold adds the principal step; approval closes with variance.
- Bank challan import: matched, amount mismatch, unknown voucher and duplicate reference cases.
- Locked period refusal.
- Screen tests including the guardian history, audit, isolation suite, RBAC fixture, ScreenCoverageTest; mutation testing on AllocationService and FeePaymentPoster with MSI at least 80%.

ACCEPTANCE CRITERIA
1. On staging, a cashier opens a session, records cash, cheque and bank deposit payments with printed receipts, clears one cheque, bounces another, and closes with a variance that the accountant approves.
2. The worked example allocation and postings match D11 exactly.
3. Timing evidence for a routine cash payment under 30 seconds on W-09.
4. CI green; staging green.

DO NOT
- Do not mark anything paid from a client-side message, redirect or SMS text.
- Do not build online gateways or 1Bill (R2).
- Do not allow editing or deleting payments; corrections come in P06.5.

OUTPUT
Report per Master section 20 with the counter walkthrough, concurrency test evidence and receipt samples.
```

### P06.5 Corrections: Reversals, Refunds, Write-offs, Adjustments and Installments

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D11 Part 1 11.4 (payment_reversal, fee_refund, fee_write_off, installment_plan), 11.7, 11.8, 11.9, DEC-172, DEC-173, DEC-177 and posting rules F11 to F16 and F20; D07 Part 3 7.26 (fee_refunds, installment_plans, installment_plan_lines) with DEC-113; D04 4.12; D05 5.7.13 with SOD-01 and SOD-08; D10 Part 3 SCR-FEE-010, 020, 027; D14 DEC-262.

PHASE GOAL
Correct money without ever editing it: payment reversals through approval with holds, refunds of deposits and advance credit, deposit forfeiture, write-offs, concession-after-issue and other adjustments with reason codes, advance credit applied later, installment plans that supersede vouchers, the branch transfer balance service for P08.2, and the struck-off and withdrawal hooks.

ALREADY BUILT
PH-00 to PH-05, P06.1 to P06.4: accounting core, fee setup, vouchers with postings, payments with allocations, cashier sessions, approval engine with PendingApprovalQuery support.

TASKS
1. Migrations (Fee): fee_refunds, installment_plans, installment_plan_lines exactly as D07 7.26 (fee_refunds.student_withdrawal_id as a soft reference without FK until P08.2 creates student_withdrawals; then P08.2 adds the FK because Fee is above Student). refund_type gains forfeit.
2. PendingApprovalQuery (Approval public query, DEC-262): hasPending(type, targetType, targetId) over pending and returned requests by snapshot targets. Holds used here: payment with a pending payment_reversal cannot be reprinted, refunded or reversed again; advance with a pending fee_refund cannot be applied.
3. Actions: RequestPaymentReversal (fee.payment.reverse; reason and evidence; level 2; SubmitApprovalRequest payment_reversal), ReversePayment (negative payment row linked by reversal_of_payment_id with uq_fpy_reversal; negative allocations release vouchers back to issued, partially_paid or overdue; advance created by the original removed and the request fails with apply_failed if that advance was already used (DEC-167); original status reversed; F11 dated today (DEC-173); PaymentReversed event), CreateRefundDraft (types security_deposit, excess_payment (advance), forfeit; amount limited by DepositBalanceQuery or advance balance), RequestRefund (SubmitApprovalRequest fee_refund), MarkRefundPaid (cash from an open session or bank; F15), ForfeitDeposit (fee_refund with type forfeit; F16 to the deposit head's forfeiture account, default fee_income_default), RecordAdjustment (payment method adjustment with reason code concession_after_issue, write_off, branch_transfer_out, branch_transfer_in, deposit_offset or opening_balance_correction (DEC-172); always through the approving request's handler; allocated like a payment; posts to the account the reason names; excluded from collection reports), ApplyAdvanceCredit (F12 to an open voucher), RequestWriteOff (SubmitApprovalRequest fee_write_off for selected open vouchers), RequestInstallmentPlan (source voucher or balance split into dated lines; SubmitApprovalRequest installment_plan), IssueInstallmentVouchers (daily command fee:issue-installments creates voucher type installment for lines due within fee.installment_issue_days_ahead (default 7)).
4. Approval types (Fee registers): payment_reversal (1 principal_of_branch; SOD-01 recorder cannot approve; approve permission fee.payment_reversal.approve; handler ReversePayment; payment version snapshot), fee_refund (1 principal_of_branch; 2 owner above fee.refund_owner_threshold; SOD-08; handler approves the refund; balance re-checked at apply), fee_write_off (1 principal_of_branch; 2 owner above fee.write_off_owner_threshold (default Rs 25,000 per student, DEC-177); handler RecordAdjustment write_off posting F14 Dr fee_write_off_expense Cr fee_receivable; student flagged so readmission needs settlement or principal override), installment_plan (1 principal_of_branch; handler activates the plan, supersedes the source voucher (CancelVoucher with superseded_by reference to the plan) and schedules lines; a payment on the source voucher after submission fails the request through the version check). The fee_concession handler from P06.2 now offers the concession_after_issue adjustment (F13) for partly paid vouchers.
5. BranchTransferFeeService (public service for P08.2 TransferStudentBranch): policies settle_before_transfer (blocks while outstanding is above zero), keep_in_source (open vouchers stay in the source branch and remain payable at either counter; a payment at the target counter posts cash to the target branch line and credits receivable on the source branch line), carry_to_target (needs the accounting entitlement, A-37: source vouchers settled by adjustment branch_transfer_out, one ad hoc balance brought forward voucher in the target with branch_transfer_in; F20 Dr fee_receivable target branch, Cr fee_receivable source branch, same party). Tested now with a fake transfer.
6. Struck-off and withdrawal hooks: command fee:struck-off-candidates monthly day 1 (students with unpaid months at or above fee.struck_off_months, default 3) exposed through StruckOffCandidateQuery for SCR-STU-015 in P08.2; generation skips students not active (already in P06.3); listener for StudentWithdrawn (event arrives in P08.2) creates a security deposit refund draft when a deposit is held (D11 11.8).
7. Screens (D10 10.30 exactly): SCR-FEE-010 Installment plans, SCR-FEE-020 Request payment reversal (SOD-01 noted), SCR-FEE-027 Refunds. Write-off is started from SCR-FEE-012 voucher detail ("Write off student balance" action selecting the student's open vouchers); P06.6 adds the same action on SCR-FEE-028.
8. Definitions: Permissions.php for fee.payment_reversal, fee.refund, fee.write_off (DEC-177), fee.installment_plan per D05 5.7.13; Settings fee.refund_owner_threshold (Rs 50,000), fee.write_off_owner_threshold (Rs 25,000), fee.struck_off_months (3), fee.installment_issue_days_ahead (7), fee.deposit_forfeit_after_months; Navigation (Fees group: Refunds, Installments).
9. decisions.md rows: DEC-113, DEC-167, DEC-172, DEC-173, DEC-177, DEC-262, A-37, A-91.

DATABASE
fee_refunds (frf), installment_plans (ipl), installment_plan_lines (ipn) per D07 Part 3 7.26 (refund_type adds forfeit).

BUSINESS RULES
- Nothing posted is edited or deleted; every correction is a new row and a new journal dated today (DEC-113, DEC-173).
- The recorder of a payment cannot approve its reversal (SOD-01); no one approves money for their own linked child (SOD-08).
- Non-cash settlements use method adjustment with a reason code and go through approval (DEC-172).
- A reversal fails safely when the advance it would remove was already used (DEC-167).
- Holds are derived from pending requests (DEC-262).
- Installment plans supersede the source voucher; installment vouchers issue on schedule.

UI SPEC
- Reversal modal shows the payment, its allocations and the vouchers that will reopen; reason required; level 2 confirmation.
- Refund form shows held deposit or advance, dues offset proposal and net payable.
- Installment form shows lines with dates and amounts that must total the source amount.

SECURITY AND PERMISSIONS
- Money approvals cannot be decided in impersonated sessions.
- Every correction audited with reason, approval reference and journal link.

TESTS
- Named posting tests F11 (reversal of cash, bank, cheque payments), F12, F13, F14, F15 (deposit and advance), F16, F20 (carry_to_target with two branches).
- Reversal: SOD-01 blocks the recorder; hold blocks reprint and refund while pending; apply_failed when the created advance was used; vouchers return to the right status.
- Write-off: owner step above Rs 25,000; F14 posted; student flagged.
- Installment plan: source voucher superseded; installment vouchers issued on schedule; payment on the source voucher after submission fails the request.
- Branch transfer service: settle_before_transfer blocks; keep_in_source two-branch posting; carry_to_target adjustments balance per branch.
- Struck-off candidates list correct; withdrawal listener creates a deposit refund draft.
- Screen tests, audit, isolation suite, RBAC fixture, ScreenCoverageTest; mutation testing on ReversePayment, RecordAdjustment and BranchTransferFeeService with MSI at least 80%.

ACCEPTANCE CRITERIA
1. On staging, a payment reversal passes approval and reopens vouchers with the reversal journal dated today; a deposit refund is approved and paid; a write-off passes principal and owner.
2. Every correction leaves both original and correction rows visible in the voucher and payment history.
3. CI green; staging green.

DO NOT
- Do not delete or edit payments, vouchers or journals.
- Do not build the transfer, withdrawal or struck-off screens (P08.2); provide the services and hooks only.
- Do not build online refunds (R2).

OUTPUT
Report per Master section 20 with the correction posting tests, SOD evidence and the branch transfer policy matrix.
```

### P06.6 Fee Reports, Ledgers and Reconciliation Controls

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D11 Part 1 11.13 completely and 11.11 F21; D07 Part 3 DEC-112; D10 Part 3 SCR-FEE-028 to 033, 041, SCR-ACC-012, SCR-RPT-003 (DEC-165) and the report catalog 10.47 fee.* and accounting.day_book rows; D03 3.23 (ReportDefinition); D12 Part 2 (DEC-126 integrity, mutation testing gate); D13 PH-06 exit criteria; D14 DEC-263.

PHASE GOAL
Give finance staff the ledgers, defaulter tools and reports they use daily, load opening balances from the old system, prove the books agree with fee records every night, and close PH-06 with named posting tests and mutation evidence.

ALREADY BUILT
PH-00 to PH-05, P06.1 to P06.5: complete fee and accounting core with corrections, StudentProfileTabRegistry, HomeSectionRegistry, import engine.

TASKS
1. Reporting basics (DEC-263): Modules\Reporting\Contracts\ReportDefinition (key, permission, title keys, filters schema, query builder with visibility, columns with masking and totals, default sort, print layout, export formats), ReportRegistry, basic SCR-RPT-003 Report runner (route app.reporting.reports.show with {report_key}; filters form, paginated table, totals row, print, Excel and PDF export through the export engine with masking and formula guard). Charts, saved views and the catalog screen come in P09.3.
2. Report definitions (Fee and Accounting contribute): fee.daily_collection (by method and cashier, adjustments excluded), fee.monthly_collection (billed, collected, outstanding by class), fee.head_wise_income, fee.outstanding_aging (0 to 30, 31 to 60, 61 to 90, over 90 days), fee.concessions_register, fee.cashier_summary, accounting.day_book. Each with its D10 10.47 permission.
3. Ledgers: StudentLedgerQuery (running balance from vouchers, payments, reversals, adjustments, advance movements, in date order), FamilyLedgerQuery (children combined plus family voucher lines, DEC-264), ReceivablesQuery (student sub-ledger aging reconciled to the fee_receivable account per branch).
4. Opening balances import (F21): import type fee_opening_balances (permission fee.ledger.import; columns student registration number, outstanding amount, advance amount, as of date, optional months unpaid); commit through ImportOpeningBalance creating one ad hoc voucher type ad_hoc with head OPENING_BALANCE per student for outstanding (posted Dr fee_receivable Cr opening_balance_equity instead of income) and an advance movement for advance (Dr opening_balance_equity Cr advance_fees_liability); allowed only before the first payment of the branch or with fee.ledger.import plus a level 3 confirmation; entry_type opening.
5. Defaulters and reminders: DefaulterQuery from student_fee_accounts (filters months unpaid, amount, class, section); SendFeeReminders (template preview, recipient count, SMS segments and credits estimate, respects quiet hours and consent rules for non-mandatory messages); command fee:reminders daily 10:00 organization time (due in 3 days and overdue reminders per settings fee.reminder_days_before and fee.reminder_overdue_every_days).
6. Integrity checks (D11 11.13; DEC-126): IntegrityCheckRegistry in Core (create if absent) with command core:integrity-check nightly 01:00 (fan out per organization; replica reads). Checks registered: fee receivable control per branch, advance liability, deferred income, cheques in hand, cash in hand per branch, exam fees payable, every posted payment has exactly one fee_payment journal (and one reversal journal when reversed), every live voucher one fee_voucher journal and every cancelled voucher a cancellation journal, journals balance. Results stored as an attention item for accountants and a platform alert on mismatch; nothing auto-corrected. SCR-RPT-007 is not required yet; results appear on SCR-PLT-016 and in the attention list.
7. Screens (D10 10.30 and 10.31 exactly): SCR-FEE-028 Student fee ledger (print; write-off action from P06.5 added here), SCR-FEE-029 Family fee ledger, SCR-FEE-030 Student profile: Fees tab (registered in StudentProfileTabRegistry; hidden from teachers per D05 5.10), SCR-FEE-031 Opening balances import, SCR-FEE-032 Defaulters, SCR-FEE-033 Send fee reminders, SCR-FEE-041 Guardian: fee ledger, SCR-ACC-012 Receivables, SCR-RPT-003 Report runner (basic).
8. Named posting test suite: tests/Feature/Posting/ with one named test per R1 posting rule F1, F1a to F1g, F2 to F8, F11 to F21 (F9 and F10 are R2 and listed as skipped with reason), each asserting exact lines with assertJournalBalanced and the account roles.
9. Mutation gate: Infection on Modules/Accounting/Services, Modules/Fee/Services and Modules/Fee/Actions money classes with a CI job failing below MSI 80% (D12 Part 2 testing rules).
10. Demo dataset command demo:seed-finance (local and staging only): two branches, 600 students, three months of vouchers, payments, reversals, refunds and write-offs, used for the integrity zero-mismatch evidence.
11. Definitions: Permissions.php for fee.ledger, fee.defaulter, fee.report per D05 5.7.13 and accounting.receivables; Settings fee.reminder_days_before (3), fee.reminder_overdue_every_days (7); Navigation (Fees group: Ledgers, Defaulters, Reports; Finance group: Receivables).
12. decisions.md rows: DEC-112, DEC-126, DEC-165, DEC-263.

DATABASE
No new tables. Uses fee and accounting tables, import and export tables, and attention items. Opening balance voucher head OPENING_BALANCE seeded as a fee head of type one_time mapped to opening_balance_equity.

BUSINESS RULES
- Reports show only what the user may see, with the same masking as screens (DEC-62).
- Integrity checks never change data; mismatches raise alerts for people to investigate.
- Opening balances post against opening_balance_equity, never income.
- Reminders respect quiet hours and consent for non-mandatory messages; due and overdue reminders are mandatory events.

UI SPEC
- Ledger screens show running balance with debits and credits columns, voucher and receipt links, and a print layout with branch letterhead.
- Defaulters list with months unpaid chips, amount, last payment date, guardian mobile (masked per permission), bulk select for reminders.
- Report runner: filters panel, table with sticky header and totals, export buttons, print.

SECURITY AND PERMISSIONS
- Report permissions per catalog; exports masked and formula-safe; guardian ledger limited by EnsureChildContext.
- Opening balance import needs fee.ledger.import and level 3 confirmation after the first payment.

TESTS
- Ledger running balance equals student_fee_accounts outstanding minus advance for seeded students.
- Receivables aging reconciles to the fee_receivable account per branch.
- Opening balances import posts against opening_balance_equity; blocked after first payment without the level 3 path.
- Integrity checks: zero mismatches on the demo dataset; a deliberately broken fixture (journal line deleted through a raw test-only connection) is detected by the right check.
- Report definitions return correct totals on fixtures; masking applied in exports.
- Named posting suite complete for every R1 rule.
- Mutation gate MSI at least 80% on the listed classes.
- Screen tests, isolation suite, RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. On staging with demo:seed-finance, core:integrity-check reports zero mismatches.
2. Daily collection, outstanding aging and day book reports match ledger totals.
3. Mutation report at least 80% and the named posting suite green.
4. PH-06 exit criteria met; MS-07 and D13 13.6 gate evidence attached.
5. CI green; staging green.

DO NOT
- Do not auto-correct mismatches.
- Do not build charts, saved views, scheduled reports or the report catalog screen (P09.3 and R2).
- Do not build financial statements or bank reconciliation (R2).

OUTPUT
Report per Master section 20 with the integrity run output, mutation report summary, named posting test list and report samples. Then run the D13 13.6 gate for PH-06.
```

---

## Assumptions (Part 5)

| ID | Assumption |
|---|---|
| A-123 | Most client schools keep their books on a July to June fiscal year, matching the Pakistani tax year. Organizations can change the year during onboarding. |
| A-124 | The seeded chart of accounts template is an acceptable starting point. Each organization's accountant reviews it during onboarding. |
| A-125 | Partner banks provide daily collection files as CSV or Excel with the voucher number, amount, date and reference. Formats differ by bank and are handled through saved import mappings. |
| A-126 | Fee amounts are whole rupees in practice. Amounts are still stored as DECIMAL(15,2), so percentage discounts compute exactly. |

## Gaps Found and Added (Part 5)

| ID | Gap | Decision |
|---|---|---|
| GAP-314 | Organizations without the Accounting module would have no journals for fee postings | Posting always on; the entitlement gates screens only (DEC-258) |
| GAP-315 | Nothing guaranteed a fiscal year and system accounts before the first voucher | Finance defaults onboarding step (DEC-259) |
| GAP-316 | Late fee rules (flat, per day, capped) had no table | `fee_late_fee_rules` (DEC-260) |
| GAP-317 | Converting before the admission voucher is paid had no named permission | `admission.application.override_payment` (DEC-261) |
| GAP-318 | "Reversal pending" and similar holds had no storage | Holds derived from pending requests (DEC-262) |
| GAP-319 | Finance reports were needed before the full Reporting phase | Basic report runner in P06.6 (DEC-263) |
| GAP-320 | Family voucher receivables had no sub-ledger party type | `party_type` `family` (DEC-264) |
| GAP-321 | The mechanism for gapless receipt numbers was unspecified | Locked counters inside the payment transaction (DEC-265) |
| GAP-322 | D14 Part 4 conflicted with D07 Part 3 ownership corrections | Part 4 aligned (corrections table above) |
