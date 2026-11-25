# Deliverable 11: Business Workflows and Approval Processes
# Part 1 of 2: Approval Engine, Approval Types, Money Workflows and Fee Posting Rules

**Version:** 1.0 (draft for approval) | **Date:** 24 Nov 2026
**Builds on:**
- D2: functional and business rules
- D3: Actions, events after commit (DEC-41), JournalPoster, schedule (3.11)
- D5: approval types and default workflows (5.9), separation of duties (5.11)
- D7: approval tables (Part 1), fee and accounting tables (Part 3, 7.26 and 7.27), system role accounts
- D10: screen IDs used as entry points

**Two parts:**

| Part | Contents |
|---|---|
| **1** | Workflow notation, approval engine, catalog of every approval type, money workflows (fee cycle, collection, cheques, cashier day, reversals, refunds, write-offs, online payments, branch transfer balances, payroll money), fee and payroll posting rules with worked examples, reconciliation controls |
| 2 | Academic and operational workflows: admission, attendance, leave, exams and marks, results, promotion and session close, transfers, withdrawal, struck off and readmission, certificates, period locks, notifications, imports, external exams, timetable, learning, library, transport, inventory, discipline, the school-year calendar of scheduled work |

---

## 11.1 New Decisions

| ID | Decision |
|---|---|
| DEC-167 | **Approval handlers validate twice.** A handler validates the payload when the request is submitted, and again inside the transaction that applies it after the final approval. If the second validation fails (for example the voucher was paid in the meantime), the request ends in `apply_failed` with the reason, nothing is applied, and the requester is notified to resubmit. |
| DEC-168 | **Requests carry a snapshot.** The request stores the payload and the version (`updated_at`) of every record it will change. The handler compares versions at apply time; a changed record fails the request unless the handler declares the change harmless (listed per type in 11.4). |
| DEC-169 | **Reminders in R1, escalation in R2.** R1 sends a daily digest of pending approvals and marks items older than 24 hours as high in "Needs attention". R2 adds SLA escalation per workflow step (`approvals:escalate`, D3 3.11). |
| DEC-170 | **Fee lines for future months are deferred.** A voucher line whose `fee_month` is after the voucher's issue month credits `deferred_fee_income`. A monthly job, `fee:recognize-deferred` (1st of the month, organization time), moves it to the head's income account when the month arrives. |
| DEC-171 | **Arrears lines are memo only.** The arrears line on a voucher shows the earlier outstanding balance so the printed total is complete, but it never posts; that money is already in fee receivable from the earlier vouchers. |
| DEC-172 | **Non-cash settlements use payment method `adjustment`** with a required reason code: `concession_after_issue`, `write_off`, `branch_transfer_out`, `branch_transfer_in`, `deposit_offset`, `opening_balance_correction`. Adjustments go through approval, post to the account their reason code names, and are excluded from collection reports. |
| DEC-173 | **Corrections are dated today.** Reversals, cancellations and adjustments are posted on the date they happen, in an open period. Nothing is back-dated into a locked period or a closed fiscal year. Reports show both the original and the correction. |
| DEC-174 | **Posting is part of the business transaction.** The fee or payroll action calls Accounting's `JournalPoster` inside the same database transaction. If posting fails, the business action fails too. Bulk voucher generation posts per chunk, each chunk in its own transaction. Organizations without the Accounting module still get every journal entry recorded (the tables are Core-tier); only the Accounting screens are hidden. |
| DEC-175 | **Cheques are not cash until they clear.** A cheque receipt debits `cheques_in_hand`; clearing moves it to the bank account; a bounce reverses the payment. |
| DEC-176 | **New system role accounts** are seeded in every chart of accounts (correction to D7 7.27): `deferred_fee_income`, `cheques_in_hand`, `cash_over_short`, `fee_write_off_expense`, `scholarship_allowed`, `opening_balance_equity`, `fee_tax_payable` (default liability for tax rules), and in R2 `payment_gateway_clearing`, `bank_charges`, `income_tax_payable`, `eobi_payable`, `social_security_payable`, `employee_advances`, `employee_loans`. |
| DEC-177 | **Fee write-off is an approval type** (`fee_write_off`, R1). New permissions: `fee.write_off.create` (ACT), `fee.write_off.approve` and `fee.write_off.reject` (PR, OW). Default steps: principal, then owner above the organization threshold (default Rs 25,000 per student). |
| DEC-178 | **Missing approval types are added** (R2): `salary_advance` (submit `payroll.advance.create`; steps PR then OW above threshold; approve `payroll.advance.approve`), `employee_loan` (submit `payroll.loan.create`; step OW; approve `payroll.loan.approve`), `employee_separation` (submit `hr.separation.create`; steps PR then OW; approve `hr.separation.approve`). |

### Corrections to earlier deliverables

| Where | Change |
|---|---|
| D5 5.7.13 | Adds `fee.write_off` resource: `create` ACT; `approve`, `reject` PR, OW (DEC-177). |
| D5 5.9 | Adds `fee_write_off` (R1), `salary_advance`, `employee_loan`, `employee_separation` (R2) (DEC-177, DEC-178). |
| D7 7.27 `accounts.system_role` | Adds the roles in DEC-176. |
| D7 7.27 `journal_entries.entry_type` | Adds `fee_late_fee`, `fee_voucher_cancellation`, `fee_deferred_recognition`, `fee_adjustment`, `cheque_clearance`, `cash_variance`, `deposit_forfeiture`. |
| D3 3.11 scheduler | Adds `fee:recognize-deferred` (monthly, day 1, 00:45 organization time) and `approvals:daily-digest` (daily 08:00). |

---

## 11.2 Workflow Notation

Flows use plain ASCII. Each step names who acts, the screen (D10 ID) and the action class (D3).

```
[Actor] action (SCR-xxx, ActionClass)
   |  -> side effect (event, job, notification)
   v
<decision?> yes -> ...
            no  -> ...
(( end state ))
```

| Symbol | Meaning |
|---|---|
| `[Actor]` | Role abbreviation (D5) or `System` for scheduled jobs and listeners |
| `->` | Side effect after commit (DEC-41) |
| `<...?>` | Decision |
| `(( ))` | End state |
| `APR:type` | Goes through the approval engine with that request type |
| `JE:type` | Posts a journal entry of that `entry_type` |

---

## 11.3 Approval Engine

### Request lifecycle

```
              submit (validates, DEC-167)
 [Requester] ---------------------------> pending (step 1)
                                             |
           +---------------------------------+-------------------------------+
           |                 |               |               |               |
        approve            reject          return         withdraw         expire (R2 SLA
     (step complete)    (any approver)   (any approver)   (requester)       with no next
           |                 |               |               |              escalation)
     more steps?          rejected       returned          withdrawn          expired
       yes -> pending       ((end))      (requester edits    ((end))           ((end))
       (next step)                        and resubmits ->
       no  -> apply                       pending step 1)
              |
     re-validate in transaction (DEC-167, DEC-168)
       ok   -> approved ((end)), effects applied, events after commit
       fail -> apply_failed ((end)), reason stored, requester notified
```

| Status | Meaning | Editable by requester |
|---|---|---|
| `pending` | Waiting at a step | No (withdraw only) |
| `returned` | Sent back for changes | Yes, then resubmit |
| `approved` | All steps approved and effects applied | No |
| `rejected` | Refused at a step | No |
| `withdrawn` | Cancelled by the requester while pending or returned | No |
| `apply_failed` | Approved but could not be applied safely | No (submit a new request) |
| `expired` | No decision within the final SLA (R2, only for types that allow expiry) | No |

### Steps and approvers

| Rule | Detail |
|---|---|
| Resolution | Each step has a resolver (D3 `ApproverResolverRegistry`): `principal_of_branch`, `vice_principal_of_branch`, `class_teacher_of_student`, `hod_of_subject`, `hr_manager_of_branch`, `exam_controller_of_branch`, `accountant_of_branch`, `owner`, `org_admin`, `reporting_manager`, `guardian_of_student`, `principal_of_target_branch`, or a named role. Resolved users are written to `approval_step_assignees` (D7 Part 1). |
| Permission check | A resolved approver acts only if they also hold the type's `approve`/`reject` permission in the request's branch (DEC-54). |
| Modes | `any` (first decision completes the step) or `all` (every assignee must approve; any rejection rejects). Default `any`. |
| Conditions | A step can be skipped by a condition on the payload: amount above a threshold, days above a limit, protected field involved, student aged 18 or more. Conditions are data in `approval_workflow_steps`, never code. |
| Separation of duties | SOD-01 to SOD-10 (D5 5.11) plus BR-APR-01 (no self-approval) remove a blocked user from the assignee list. If a step is left with no assignee, it escalates to the next step, or to the principal when it was the last (BR-APR-04). |
| Delegation | An approver may delegate by date range and request type (SCR-APR-008). The delegate still needs the approve permission. Decisions record both names ("Approved by Ayesha Khan for Imran Ali"). |
| Snapshots | The request detail always renders the stored payload, so approvers see what was asked, not what the record became. |
| Audit | Submission, every decision (with comment), delegation use, apply result and failures are audit events `approval.request.{submitted,approved,rejected,returned,withdrawn,applied,apply_failed}`. |
| Notifications | `ApprovalRequestSubmitted` to step assignees; `ApprovalStepDecided` to the requester and next assignees; final outcome to the requester and any notify-only roles of the type. In-app always; SMS only for guardian-facing outcomes (leave, profile change). |
| Bulk approve | Allowed for types flagged `bulk_approvable` (student leave, missing punch, attendance correction, document verification). Each request is still validated and applied on its own; failures are listed. |

### Handler contract (`Approval/{Type}Handler.php`)

| Method | Runs | Responsibility |
|---|---|---|
| `rules()` | On submit | Payload validation rules |
| `validateSubmission()` | On submit | Business checks (record exists, state allows the change, amounts within limits) |
| `snapshotTargets()` | On submit | Records to version-check later (DEC-168) |
| `summary()` | On render | What approvers see: before and after, money effect, attachments |
| `apply()` | After final approval, inside a transaction | Re-validate, then call the owning module's Action; must be idempotent (a retried apply does nothing twice) |
| `onRejected()`, `onReturned()`, `onWithdrawn()` | After decision | Release holds (for example a voucher marked "reversal pending") |

---

## 11.4 Approval Type Catalog

Default workflows are seeded per organization and can be edited (SCR-APR-007). Thresholds are settings (A-42). "Hold" means a flag that blocks conflicting actions while the request is pending.

### Attendance, leave and HR

| Type | Rel | Raised from | Payload | Default steps | On approve | On reject or return | Version check |
|---|---|---|---|---|---|---|---|
| `attendance_correction` | R1 | SCR-ATT-007 | Students, date, new status, reason, evidence | 1 PR or VP (any) | Writes revision rows through Attendance (`ApplyAttendanceCorrection`); guardian SMS if status changes to or from absent | Requester notified | Attendance row version; changed rows fail only that student |
| `missing_punch` | R1 | SCR-ATT-015 | Date, in or out time, reason | 1 HRM of branch | Adds an administrative punch and re-resolves the day | Notified | Staff attendance row |
| `student_leave` | R1 | SCR-LVE-011, SCR-LVE-002 | Student, dates, half day, reason, document | 1 CT of student; 2 PR if more than 3 days | Leave application approved; future days pre-filled as `LV` through `AttendanceLeaveWriter`; past finalized days create corrections | Guardian SMS with reason | None (dates re-checked against holidays) |
| `staff_leave` | R1 | SCR-LVE-004, SCR-LVE-002 | Employee, type, dates, half day, reason | 1 reporting manager or HOD; 2 PR if more than 3 days; HRM notified | Balance deducted; days written as `LV` | Notified | Leave balance re-checked at apply |
| `employee_profile_correction` | R1 | SCR-IAM-008 | Field, new value, document | 1 HRM | Updates the employee record | Notified | Employee version |
| `employee_separation` | R2 | SCR-HR-018 | Employee, last day, reason, clearance items | 1 PR; 2 OW | Status separated; user deactivated on last day; final settlement run created (Payroll) | Notified | Employee version |
| `salary_advance` | R2 | SCR-PAY-014 | Amount, recovery months, reason | 1 PR; 2 OW above threshold | Advance approved; disbursement via SCR-PAY-011 (JE:salary_advance) | Notified | None |
| `employee_loan` | R2 | SCR-PAY-014 | Amount, installments, purpose | 1 OW | Loan approved with installment schedule | Notified | None |

### Student and academic

| Type | Rel | Raised from | Payload | Default steps | On approve | On reject or return | Version check |
|---|---|---|---|---|---|---|---|
| `student_profile_correction` | R1 | SCR-STU-026 | Field, new value, document | 1 guardian (if raised by student, skipped when 18+ or college); 2 CT; 3 PR if protected field | Updates through Student (`UpdateStudentProfile` or `UpdateProtectedField`); timeline event | Notified | Student version |
| `student_modification` | R1 | SCR-STU-026 | Same as above for non-identity fields (address, contact) | Same | Same | Same | Same |
| `academic_correction` | R1 | SCR-STU-026 | Subject choice, group, roll number | 1 guardian (if student); 2 CT; 3 EXC | Updates student subjects or enrollment detail | Notified | Enrollment version |
| `document_verification` | R1 | SCR-STU-025 (guardian upload) | Document file, type | 1 ADO | Document marked verified | Guardian asked to re-upload with reason | Document version |
| `branch_transfer` | R1 | SCR-STU-012 | Target branch, section, effective date, fee policy | 1 PR of source; 2 PR of target | Runs `TransferStudentBranch` (11.9 balances, D4 4.12) | Notified | Enrollment version |
| `withdrawal` | R1 | SCR-STU-014 | Reason, last date, clearance status | 1 PR | Runs `WithdrawStudent` (Part 2); deposit refund request created if a deposit is held | Notified | Enrollment version |
| `marks_revision` | R1 | SCR-EXM-015 | Mark entry, old and new value, reason | 1 HOD of subject; 2 EXC | Mark revision row; affected result batches marked stale | Notified | Mark entry version |
| `result_publication` | R1 | SCR-RES-009 | Result batch, audience, schedule | 1 PR | Batch approved; EXC can publish (SCR-RES-010) or the scheduled time publishes it | EXC notified with comment | Batch version (a reprocessed batch fails the request) |
| `result_recheck` | R2 | SCR-RES-020 | Subject, reason, fee voucher | 1 EXC | Recheck accepted; outcome entered through `marks_revision` | Notified; fee refunded if configured | None |
| `external_entry_withdrawal` | R2 | SCR-EXT-015 | Entry, reason | 1 PR | Entry withdrawn (`live_marker` cleared); refund rules per awarding body deadlines | Notified | Entry version |

### Money

| Type | Rel | Raised from | Payload | Default steps | On approve | On reject or return | Version check |
|---|---|---|---|---|---|---|---|
| `fee_concession` | R1 | SCR-FEE-008 | Student, head or all, percent or amount, months, reason, document | 1 PR; 2 OW above 25% | Concession active from the next voucher; for issued unpaid vouchers in its months, the accountant is prompted to regenerate; for partly paid ones, an `adjustment` (reason `concession_after_issue`) is offered | Notified | Student fee account version |
| `scholarship` | R1 | SCR-FEE-009 | Student, sponsor, heads, period, value | 1 PR; 2 OW | Same as concession, posting to `scholarship_allowed` | Notified | Same |
| `installment_plan` | R1 | SCR-FEE-010 | Source voucher or balance, installments | 1 PR | Plan active; installment vouchers issued on their dates; source voucher superseded | Notified | Voucher version (paid in between fails) |
| `payment_reversal` | R1 | SCR-FEE-020 | Payment, reason, evidence | 1 PR | `ReversePayment`: negative payment row, allocations released, JE:fee_payment_reversal; hold released | Hold released; notified | Payment version |
| `cash_variance` | R1 | Automatic at SCR-FEE-026 | Session, expected, counted, variance, explanation | 1 ACT (not the cashier, SOD-02); 2 PR above threshold | JE:cash_variance; session closed as `closed_with_variance` | Cashier must recount (session stays `closing`) | Session version |
| `fee_refund` | R1 | SCR-FEE-027 | Student, refund type (deposit, advance), amount, method | 1 PR; 2 OW above threshold | Refund approved; payment out recorded when paid (JE:fee_refund) | Notified | Advance or deposit balance re-checked |
| `fee_write_off` | R1 | SCR-FEE-028 action | Student, vouchers, amount, reason | 1 PR; 2 OW above threshold | `adjustment` payment (reason `write_off`), JE:fee_adjustment to `fee_write_off_expense` | Notified | Voucher versions |
| `period_unlock` | R1 | SCR-CORE-003 | Module, period, duration, reason | 1 PR (attendance, exams); OW (fee, accounting); HRM requests go to PR | Period unlocked until `relock_due_at`; relocks automatically | Notified | Lock version |
| `journal_entry` | R1 | SCR-ACC-006 above threshold | Journal draft | 1 OW or OA | Journal posted | Draft returned | Draft version |
| `expense` | R2 | SCR-ACC-015 | Expense voucher | 1 PR; 2 OW above threshold | Posted (JE:expense) | Notified | Voucher version |
| `purchase` | R2 | SCR-ACC-017 | Purchase bill | 1 PR; 2 OW above threshold | Posted (JE:purchase) | Notified | Bill version |
| `payroll_run` | R2 | SCR-PAY-007 | Run totals and payslip snapshot | 1 PR; 2 OW (SOD-06) | Run approved; ACT can finalize (JE:payroll) | Run returned to `under_review` | Run version (recalculated run fails) |
| `stock_adjustment` | R3 | SCR-INV-008 | Item, location, quantity, reason | 1 PR | Adjustment movement written | Notified | Stock balance re-checked |

---

## 11.5 Fee Cycle (monthly)

```
Day 20 to 25 (month M-1)
[ACT] Generate vouchers for month M (SCR-FEE-013, GenerateVouchers)
   -> bulk job per class chunk: GenerateVoucherChunk
      each voucher: lines from structure + concessions + scholarships + discount rules
                    + contributions (transport, library fines, VoucherLineContributor)
                    + arrears memo line (DEC-171) + advance applied (if credit exists)
      JE:fee_voucher per voucher (11.11)
   -> VouchersBatchGenerated -> guardians notified (SMS with amount and due date)
[ACT or CSH] Print or share vouchers (SCR-FEE-016); guardians see them in the portal
   |
   v
Day 1 to due date (default 10th)
Payments arrive: counter cash or cheque (11.6), bank challan (SCR-FEE-022/023),
online (R2, 11.10), 1Bill (R2)
   |
Due date + 1, 00:15  [System] fee:mark-overdue -> unpaid vouchers become overdue
Due date + 1, 00:30  [System] fee:apply-late-fees -> late fee per rule (flat, per day, capped)
                               JE:fee_late_fee per voucher; voucher late_fee_amount updated
Daily 10:00          [System] fee:reminders -> due-date and overdue reminders (quiet hours respected)
   |
Day 1 of M+1         [System] fee:recognize-deferred -> deferred lines for M+1 move to income
Day 1 of M+1         [System] fee:struck-off-candidates -> students unpaid for N months (setting,
                               default 3) listed for the principal (SCR-STU-015)
```

| Rule | Detail |
|---|---|
| One recurring charge per month | `uq_fvi_recurring` (D7) blocks charging the same head twice for a student and month across individual and family vouchers |
| Mid-month admission | Admission voucher includes the joining month (pro rata if the setting allows) and admission heads |
| Family vouchers | One voucher per family when `families.voucher_mode = family`; lines still name each child; discounts by sibling rule |
| Cancelled or superseded vouchers | Their lines stop counting (`is_live = 0`) in the same transaction; JE:fee_voucher_cancellation mirrors the accrual |
| Concession after issue | DEC-172 and 11.4 |
| Late fee waiver | A voucher line adjustment is not allowed; a waiver is a concession on the late fee head through `fee_concession` |

---

## 11.6 Counter Collection

```
[CSH] Open session (SCR-FEE-025, OpenCashierSession) with float
   |
[Guardian at counter] gives voucher or name
[CSH] Scan or search (SCR-FEE-017) -> voucher loaded with outstanding and other open vouchers
   |
<amount = outstanding?> yes -> allocate to this voucher by head priority
                        no, less -> partial: allocate by priority; voucher partially_paid
                        no, more -> excess: confirm "keep as advance credit"
   |
<method?>
  cash          -> requires open session
  cheque        -> cheque number, bank, date; status pending (DEC-175)
  bank deposit  -> bank account and reference
   |
[CSH] Save (RecordPayment, idempotency key)
   transaction: lock voucher rows, allocate, receipt number (gapless per branch and fiscal year),
                fee_payments + allocations + advance movement, JE:fee_payment, audit fee.payment.recorded
   -> PaymentRecorded -> receipt printed, SMS receipt to primary guardian,
      student_fee_accounts recalculated, today's collection snapshot refreshed
((receipt issued))
```

| Rule | Detail |
|---|---|
| Arrears on the current voucher | The amount first settles older open vouchers (oldest first), then the current one; the receipt lists every voucher touched |
| Allocation order | Setting `fee.allocation_order`: by head priority within a voucher (fines and late fees first by default), vouchers oldest first |
| Duplicate submit | Same idempotency key returns the first receipt (FR-GEN-013) |
| Locked period | A payment dated in a locked fee period is refused; received date defaults to today |
| Receipt reprint | Marked "Duplicate", audited |

### Cashier day close

```
[CSH] Close session (SCR-FEE-026)
   enter counted cash by denomination
   expected = float + cash received - cash refunds paid from the counter
<variance = 0?> yes -> session closed; cash summary printed; cash handed to accountant
                no  -> session "closing"; APR:cash_variance (11.4)
                        approve -> JE:cash_variance; session closed_with_variance
                        return  -> recount
[ACT] Bank deposit of collected cash: journal Dr bank, Cr cash_in_hand (SCR-ACC-006 template "Cash deposit")
```

Sessions left open past midnight appear in "Needs attention" (D10 10.7). A session can't be reopened; a new one is opened.

### Cheque lifecycle (DEC-175)

```
received (pending) --clear (SCR-FEE-021, MarkChequeCleared)--> cleared
       |                 JE:cheque_clearance: Dr bank, Cr cheques_in_hand
       |
       +--bounce (BounceCheque)--> bounced
                 ReversePayment without approval (bounce is evidence, not a choice)
                 JE:fee_payment_reversal; vouchers reopen; optional cheque bounce fine
                 as an ad hoc voucher; guardian notified
```

---

## 11.7 Payment Reversal

```
[CSH or ACT] Request reversal (SCR-FEE-020) with reason -> APR:payment_reversal
   hold: payment flagged "reversal pending" (blocks refund of resulting advance, blocks reprint)
[PR] Approve (SOD-01: the recorder cannot approve)
   apply: ReversePayment
     negative fee_payments row linked to the original (uq_fpy_reversal)
     allocations released; vouchers back to issued, partially_paid or overdue
     advance created by the original payment removed (if already used, request fails, DEC-167)
     JE:fee_payment_reversal dated today (DEC-173)
   -> PaymentReversed -> guardian notified, account recalculated
((original marked reversed; both rows kept))
```

---

## 11.8 Refunds and Write-offs

### Refund of security deposit on withdrawal

```
Withdrawal approved (Part 2) -> deposit held? yes -> FeeRefund draft created (type deposit)
[ACT] Complete refund (SCR-FEE-027): deductions for unpaid dues (offset), method
   -> APR:fee_refund
[PR] (and OW above threshold) approve
[ACT] Mark paid (cash from counter or bank)
   JE:fee_refund: Dr security_deposit_liability, Cr cash_in_hand or bank
   offset of dues: adjustment (reason deposit_offset), Dr security_deposit_liability, Cr fee_receivable
((refund paid; leaving certificate clearance updated))
```

Forfeited deposits (policy: non-refundable after N years, or left unclaimed for 12 months): `deposit_forfeiture` entry, Dr `security_deposit_liability`, Cr the deposit head's forfeiture income account (default `fee_income_default`), after principal approval through `fee_refund` with type `forfeit`.

### Refund of advance credit

Same flow with type `advance`: Dr `advance_fees_liability`, Cr cash or bank.

### Write-off

```
[ACT] From the student ledger (SCR-FEE-028) select vouchers -> APR:fee_write_off
[PR] then [OW] above threshold approve
   apply: adjustment payment (reason write_off) allocated to the vouchers
          JE:fee_adjustment: Dr fee_write_off_expense, Cr fee_receivable
((vouchers settled as written off; student flagged; readmission requires settling or principal override))
```

---

## 11.9 Balances on Branch Transfer (D4 4.12)

| Policy | Flow | Posting |
|---|---|---|
| `settle_before_transfer` | Transfer approval is blocked until the student's outstanding is zero (pay or write off) | None extra |
| `keep_in_source` | Open vouchers stay in the source branch and remain payable at either counter; new vouchers come from the target | Payments at the target counter post cash to the target branch and credit receivable on the source branch line (two-branch entry) |
| `carry_to_target` (needs Accounting, A-37) | Source vouchers settled by `adjustment` (reason `branch_transfer_out`); one ad hoc "balance brought forward" voucher in the target (reason `branch_transfer_in`) | Dr fee_receivable (target branch), Cr fee_receivable (source branch), party student |

---

## 11.10 Online Payments (R2)

### Gateway (JazzCash, Easypaisa)

```
[GDN] Pay now (SCR-FEE-038) -> InitiateOnlinePayment
   payment_gateway_transactions row (initiated, merchant_txn_ref unique), idempotency key
   -> redirect to the provider's hosted page
[Provider] guardian pays
   -> callback POST /webhooks/{provider}/{organization_uuid}
        stored in webhook_receipts first (DEC-42), signature verified, 200 returned fast
        ProcessGatewayCallback (queue critical)
   -> guardian browser returns to SCR-FEE-039 "Verifying" (DEC-166)
[System] ConfirmGatewayPayment
   calls the provider's status API (never trusts the callback alone)
   <paid and amount matches?> yes -> RecordPayment (method online, uq_fpy_gateway)
                                     JE:fee_payment: Dr payment_gateway_clearing, Cr fee_receivable
                                     guardian sees success and receipt
                              no  -> transaction failed or pending; guardian sees status; nothing posted
[System] Every 10 minutes: re-verify transactions pending for more than 15 minutes; expire after 24 hours
[ACT] Settlement (bank statement or provider report, SCR-ACC-020)
   Dr bank (net), Dr bank_charges (fee), Cr payment_gateway_clearing (gross)
```

Amount rules: the guardian pays the outstanding amount of one voucher; partial online payments are a setting (default off). A callback for an already-paid voucher creates an advance credit and an attention item for the accountant.

### 1Bill (bill payment through any bank app)

```
[Bank app] Bill inquiry with consumer number -> GET via adapter (hooks host)
   [System] returns name, amount due (within due date or after), due date, status
[Bank app] Payment notification -> stored in webhook_receipts, verified
   [System] RecordPayment (method onebill), receipt; JE:fee_payment Dr payment_gateway_clearing, Cr fee_receivable
[ACT] Settlement from the aggregator report clears payment_gateway_clearing to bank
```

---

## 11.11 Fee Posting Rules

### Controls (every entry)

| Control | Rule |
|---|---|
| Balance | Debits equal credits; enforced by `JournalPoster` and the `journal_entries` CHECK |
| Branch | Every line carries a branch (D7 DEC-114); fee lines use the voucher's or payment's branch |
| Party | Receivable, advance and deposit lines carry `party_type = student` (or `family` for family vouchers) and the party id |
| Once only | `uq_jen_source (organization_id, source_type, source_id, entry_type)` (DEC-110) makes retries harmless |
| Dates | Voucher entries on the issue date; payments on the received date; corrections today (DEC-173) |
| Periods | Entry date must be in an open fiscal year and an unlocked accounting month; otherwise the business action fails with `core.period_locked` |
| Accounts | Head income and liability accounts from `fee_heads.ledger_account_id`; others from system roles (D7 7.27, DEC-176) |
| Amounts | Exact decimal through `Money`; no rounding step (fees are whole rupees or stored paisa) |

### Posting table

| # | Event (Action) | entry_type | Debit | Credit | Notes |
|---|---|---|---|---|---|
| F1 | Voucher issued (`GenerateVouchers`, `GenerateStudentVoucher`, `IssueAdHocVoucher`) | `fee_voucher` | `fee_receivable` (net of posted lines) | Each charge line's head income account; `deferred_fee_income` for future-month lines (DEC-170) | Arrears lines not posted (DEC-171) |
| F1a | Discount, sibling, early payment line on the voucher | (same entry) | `discount_allowed` | (reduces receivable) | Shown as a separate debit so gross income stays visible |
| F1b | Concession line | (same entry) | `discount_allowed` | | |
| F1c | Scholarship line | (same entry) | `scholarship_allowed` | | A-88 |
| F1d | Tax line | (same entry) | | Tax rule's liability account (default `fee_tax_payable`) | |
| F1e | Refundable deposit line | (same entry) | | `security_deposit_liability` | |
| F1f | Pass-through line (exam entry, board fee) | (same entry) | | `exam_fees_payable` | DEC-84 |
| F1g | Advance credit applied at generation | (same entry) | `advance_fees_liability` | `fee_receivable` | Voucher shows "Advance applied" |
| F2 | Deferred month arrives (`fee:recognize-deferred`) | `fee_deferred_recognition` | `deferred_fee_income` | Head income account | One entry per branch per month, lines per head |
| F3 | Late fee applied (`ApplyLateFees`) | `fee_late_fee` | `fee_receivable` | `late_fee_income` | |
| F4 | Voucher cancelled or superseded (`CancelVoucher`) | `fee_voucher_cancellation` | Mirror of F1 credits | Mirror of F1 debits | Only for unpaid vouchers; deferred lines already recognized reverse from income |
| F5 | Cash payment (`RecordPayment`) | `fee_payment` | `cash_in_hand` | `fee_receivable` (allocated), `advance_fees_liability` (excess) | |
| F6 | Bank deposit or bank challan payment | `fee_payment` | Bank account's ledger account | Same as F5 | |
| F7 | Cheque received | `fee_payment` | `cheques_in_hand` | Same as F5 | DEC-175 |
| F8 | Cheque cleared (`MarkChequeCleared`) | `cheque_clearance` | Bank ledger account | `cheques_in_hand` | |
| F9 | Online or 1Bill payment (R2) | `fee_payment` | `payment_gateway_clearing` | Same as F5 | |
| F10 | Gateway settlement (R2) | `manual` (reconciliation template) | Bank (net), `bank_charges` (fee) | `payment_gateway_clearing` (gross) | |
| F11 | Payment reversal or cheque bounce (`ReversePayment`, `BounceCheque`) | `fee_payment_reversal` | Mirror of F5 to F9 credits | Mirror of F5 to F9 debits | Dated today |
| F12 | Advance applied later to a voucher (`ApplyAdvanceCredit`) | `fee_adjustment` | `advance_fees_liability` | `fee_receivable` | |
| F13 | Concession after issue (adjustment) | `fee_adjustment` | `discount_allowed` | `fee_receivable` | DEC-172 |
| F14 | Write-off (adjustment) | `fee_adjustment` | `fee_write_off_expense` | `fee_receivable` | DEC-177 |
| F15 | Refund paid (deposit or advance) | `fee_refund` | `security_deposit_liability` or `advance_fees_liability` | `cash_in_hand` or bank | |
| F16 | Deposit forfeited | `deposit_forfeiture` | `security_deposit_liability` | Forfeiture income account | |
| F17 | Exam fees remitted to the awarding body | `exam_fee_remittance` | `exam_fees_payable` | Bank | Recorded from SCR-EXT-014 or SCR-ACC-006 |
| F18 | Cash shortage approved | `cash_variance` | `cash_over_short` | `cash_in_hand` | Excess is the mirror |
| F19 | Cash deposited to bank | `manual` (template) | Bank | `cash_in_hand` | |
| F20 | Branch transfer carry (adjustment pair) | `fee_adjustment` | `fee_receivable` (target branch) | `fee_receivable` (source branch) | Same party |
| F21 | Opening balances import (`ImportOpeningBalances`) | `opening` | `fee_receivable` (per student) | `opening_balance_equity` | Advance balances: Dr `opening_balance_equity`, Cr `advance_fees_liability` |

### Worked example: one student, one month

Ayesha Khan, 9-A, Gulberg. December 2026 voucher issued 25 Nov 2026 with these lines:

| Line | Month | Amount |
|---|---|---|
| Tuition | Dec | Rs 8,500 |
| Lab fee | Dec | Rs 500 |
| Sibling discount (10% of tuition) | Dec | -Rs 850 |
| Tuition (winter vacation, collected in advance) | Jan | Rs 8,500 |
| Arrears (November voucher, memo) | | Rs 3,500 |
| **Printed total** | | **Rs 20,150** |

**F1, 25 Nov 2026 (issue):**

| Account | Debit | Credit |
|---|---|---|
| Fee receivable (party Ayesha Khan) | 16,650 | |
| Discount allowed | 850 | |
| Tuition income | | 8,500 |
| Lab fee income | | 500 |
| Deferred fee income | | 8,500 |
| **Total** | **17,500** | **17,500** |

The Rs 3,500 arrears is already in fee receivable from the November voucher, so it isn't posted again.

**F3, 11 Dec 2026 (late fee Rs 200 after the 10 Dec due date):** Dr Fee receivable 200, Cr Late fee income 200. The voucher total becomes Rs 20,350.

**F5, 12 Dec 2026 (cash Rs 20,350 at counter 2):**

| Account | Debit | Credit |
|---|---|---|
| Cash in hand (Gulberg) | 20,350 | |
| Fee receivable (party Ayesha Khan) | | 20,350 |

Allocation: Rs 3,500 to the November voucher (now paid), Rs 16,850 to the December voucher (late fee 200 first, then lab 500, tuition 8,500 net of discount, January tuition 8,500), so both vouchers become `paid`.

**F2, 1 Jan 2027 (recognition):** Dr Deferred fee income 8,500, Cr Tuition income 8,500.

**F11, if the payment had been reversed on 15 Dec 2026:** Dr Fee receivable 20,350, Cr Cash in hand 20,350, dated 15 Dec, with both vouchers reopened.

### Worked example: Cambridge entry fee (pass-through)

O Level Physics 5054 entry for the June 2027 series, fee Rs 9,000, voucher issued 10 Feb 2027:

| Date | Entry | Debit | Credit |
|---|---|---|---|
| 10 Feb | F1f voucher | Fee receivable 9,000 | Exam fees payable 9,000 |
| 14 Feb | F6 bank challan | Bank (HBL) 9,000 | Fee receivable 9,000 |
| 21 Feb | F17 remittance | Exam fees payable 9,000 | Bank (HBL) 9,000 |

The school's income statement is unaffected, as DEC-84 requires.

---

## 11.12 Payroll Money (R2)

### Payroll run

```
[HRM or ACT] New run (SCR-PAY-006): month, branch; staff attendance must be locked
   -> bulk calculation: components, attendance deductions, tax by slab, EOBI, loan and advance recovery
[HRM] Review (SCR-PAY-007): holds, one-time adjustments with reason
[HRM] Submit -> APR:payroll_run (SOD-06)
[PR] then [OW] approve
[ACT] Finalize (SCR-PAY-009) -> JE:payroll; payslips final; staff notified (payslip in portal)
[ACT] Pay (SCR-PAY-010): bank upload file export; mark paid -> JE:salary_payment
((run paid))
```

### Payroll posting rules

| # | Event | entry_type | Debit | Credit |
|---|---|---|---|---|
| P1 | Run finalized | `payroll` | Each earning component's expense account (default `salary_expense`), employer contribution expense | `salaries_payable` (net pay), `income_tax_payable`, `eobi_payable`, `social_security_payable`, `employee_loans` (installments recovered), `employee_advances` (advance recovered), employer contribution payables, other deduction accounts (library fines to the fine head's account) |
| P2 | Salary paid | `salary_payment` | `salaries_payable` | Bank or `cash_in_hand` |
| P3 | Advance disbursed | `salary_advance` | `employee_advances` | Bank or cash |
| P4 | Loan disbursed | `loan_disbursement` | `employee_loans` | Bank or cash |
| P5 | Tax, EOBI, social security remitted | `manual` (template) | The payable | Bank |
| P6 | Salary payment reversed | `reversal` | Bank or cash | `salaries_payable` |

Lines carry the employee's pay branch and department. Supplementary runs post the same way; approved payslips never change (DEC-116).

---

## 11.13 Reconciliation Controls

These run in the nightly integrity job (DEC-126) per organization and branch. A mismatch raises an attention item for the accountant and a platform alert; nothing is auto-corrected.

| Check | Must equal |
|---|---|
| Fee receivable control | GL balance of `fee_receivable` per branch = sum of `fee_vouchers.outstanding_amount` for live vouchers of that branch |
| Advance liability | GL balance of `advance_fees_liability` = sum of `student_fee_accounts.advance_balance` |
| Deferred income | GL balance of `deferred_fee_income` = sum of live voucher lines with a future `fee_month` not yet recognized |
| Cheques in hand | GL balance of `cheques_in_hand` = sum of pending cheque payments |
| Cash in hand | GL balance of `cash_in_hand` per branch = expected cash of open sessions + closed sessions not yet deposited |
| Exam fees payable | GL balance = entry fees collected for unsubmitted or unremitted series |
| Gateway clearing (R2) | GL balance = verified online payments not yet settled |
| Every payment posted | Each posted `fee_payments` row has exactly one `fee_payment` journal (and its reversal, if reversed) |
| Every live voucher posted | Each issued voucher has one `fee_voucher` journal; each cancelled one also has a cancellation journal |
| Journals balance | Every journal's lines balance (also a database CHECK) |

---

## Assumptions (Part 1)

| ID | Assumption |
|---|---|
| A-88 | Scholarships reduce income through `scholarship_allowed`. When a sponsor actually pays the school, that receipt is recorded as donation income by a manual journal; sponsor invoicing is out of scope for R1 to R3. |
| A-89 | Pro rata fees for mid-month admissions are off by default; schools usually charge the full month. |
| A-90 | Partial online payments are off by default; guardians pay one voucher's outstanding amount. |
| A-91 | Deposits are non-refundable after the period set by the organization (default: refundable on withdrawal only). |

## Gaps Found and Added (Part 1)

| ID | Gap | Decision |
|---|---|---|
| GAP-222 | An approved request could be applied to a record that changed after submission | Double validation and snapshots (DEC-167, DEC-168) |
| GAP-223 | Fees collected for vacation months were booked as income too early | Deferred income and monthly recognition (DEC-170) |
| GAP-224 | Arrears printed on vouchers risked being posted twice | Memo-only arrears (DEC-171) |
| GAP-225 | No way to settle a balance without cash (late concessions, write-offs, transfers) | `adjustment` with reason codes (DEC-172) |
| GAP-226 | Corrections could be back-dated into locked periods | Corrections dated today (DEC-173) |
| GAP-227 | Uncleared cheques were treated as cash | `cheques_in_hand` (DEC-175) |
| GAP-228 | Several posting accounts had no system role | DEC-176 |
| GAP-229 | No approval for writing off bad debts | `fee_write_off` (DEC-177) |
| GAP-230 | Salary advances, loans and separations had permissions but no approval types | DEC-178 |
| GAP-231 | No defined checks that the books agree with fee records | Reconciliation controls (11.13) |

---

Reply CONTINUE for Deliverable 11, Part 2
