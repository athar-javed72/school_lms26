# Deliverable 14: Sequential Antigravity Implementation Prompts
# Part 8 of 9: Release 2, PH-11 to PH-18

**Version:** 1.0 (draft for approval) | **Date:** 24 Nov 2026
**Builds on:** D14 Parts 1 to 7, D01 GAP-32 to GAP-40, D02 (R2 requirements), D03 3.15 (adapters), D04 4.20, D05 5.7.13 to 5.7.22, 5.9, 5.11, D06A A.9 and A.11, D07 Part 3 (7.26 gateway tables, 7.27 R2 accounting, 7.30, 7.32, 7.33) and Part 4 (7.37 to 7.40, 7.42, 7.44), D08 8.15, D10 Part 2 and Part 3 (every R2 screen), D11 Part 1 (11.4, 11.10 to 11.12) and Part 2 (11.29, 11.30), D12 Part 1 and Part 2, D13 PH-11 to PH-18
**How to run:** start a new Antigravity conversation for each step and paste the Phase 0 Master Context Prompt before the step prompt. Every R2 step assumes R1 is live in production, so migrations are expand-only (D12 12.29) and every change must be safe for tenants already using the system.

---

## 14.22 New Decisions

| ID | Decision |
|---|---|
| DEC-281 | **External exam workflows become step P16.5.** The R2 external exam workflows that DEC-266 moved out of P07.3 become step P16.5, together with result rechecks (`result_recheck`) and national board support (GAP-32). The step fits inside the PH-16 buffer. If the PH-16 gate shows slippage, screens move under DEC-223 instead of the R2 date moving. |
| DEC-282 | **Route capacity in R2.** `transport_routes` gains `capacity` SMALLINT UNSIGNED (seats). In R2 the allocation action locks the route row and counts active allocations against it. From R3, when an active vehicle assignment exists, the limit is the smaller of the route capacity and the vehicle's seating capacity. |
| DEC-283 | **Sanctum and the API arrive in two steps.** P17.3 introduces Sanctum, the `api.v1` middleware group on `api.{platform_domain}`, device tokens (`device:punch`) and the single endpoint `api.v1.attendance.punches.store`, because the sync agent needs them. P18.2 adds portal tokens and the rest of API v1. |
| DEC-284 | **The sync agent is a separate repository.** `darsora-sync-agent` is a Laravel Zero CLI distributed as one Windows and Linux binary built with static-php-cli, so school PCs need no PHP install. It pulls punches from devices on the school LAN through vendor adapters, spools them in an append-only local JSONL file, and pushes batches to `api.v1.attendance.punches.store` with its device token. It never connects to the database. |
| DEC-285 | **System roles for R2 accounting.** New roles `accounts_payable` (default payable for vendors without their own account) and `expense_default` are added to the DEC-176 list. Expense, purchase and vendor payment postings use only these roles or explicit account columns. |
| DEC-286 | **Each organization uses its own merchant accounts.** Online payment funds settle directly into the organization's bank account under its own JazzCash, Easypaisa and 1Bill merchant agreements. The platform never collects or holds tuition money. Payments use only official merchant APIs with hosted or redirect flows (DEC-196). A payment is posted only after the provider's verification API confirms status, amount and currency (DEC-166). |
| DEC-287 | **Payroll calculations are registered calculators.** Every computed component uses a `formula_key` that names a calculator class in `Modules\Payroll\Calculators`, for example `income_tax_annualized`, `eobi_employee`, `eobi_employer`, `social_security`, `absence_deduction` and `late_deduction`. Rates, slabs and wage bases are data: tax slabs in `payroll_tax_slabs` (A-62), and EOBI and social security bases and rates in organization settings with effective dates. Monthly income tax = (annual tax on projected annual taxable income minus tax already withheld this tax year) divided by the months remaining in the tax year. |
| DEC-288 | **Quiz scores reach exam marks only as drafts.** When an `exam_linked` quiz releases results, `ImportMarksFromQuiz` (an Examination public action) writes each student's chosen score into the teacher's marks submission as a draft. The teacher still reviews and submits it, so DEC-181 and SOD-04 apply as usual. |
| DEC-289 | **Messaging safeguards (GAP-37).** A thread exists only between a guardian with an active portal link and a teacher currently assigned to that student. Messages can't be edited; a moderator can hide one, and the hidden message is kept with its reason. Delivery outside the branch setting `communication.office_hours` waits until the next opening. Phone numbers and emails are never shown. Attachments are limited to images and PDFs, which are re-encoded and scanned. Principals and vice principals have read-only oversight. Message bodies follow the D12 retention schedule. |
| DEC-290 | **WhatsApp runs on each organization's own account.** Each organization connects its own WhatsApp Business account through the official WhatsApp Business Platform (Cloud API). Business-initiated messages use only templates already approved by Meta, each mapped to a notification template and language. A guardian receives WhatsApp only with `whatsapp` consent (DEC-257). Mandatory events fall back to SMS when WhatsApp fails or when no consent exists. |
| DEC-291 | **Approval SLAs and escalation.** `approval_workflow_steps.sla_hours` becomes active. Every 15 minutes, `approvals:escalate` adds the escalation resolver's users as assignees for overdue steps. The default escalation chain is principal of the branch, then owner, and each workflow can set its own. Types that allow expiry move to `expired` after their final SLA, and the requester is notified. |
| DEC-292 | **The demo environment is separate (GAP-38).** `demo.{platform_domain}` runs its own database seeded by `DemoSeeder` and resets every night at 03:00 PKT. It has no SMS, email, WhatsApp or payment providers (fake adapters only) and holds no real data. A banner on every page says it is a demo. |

### Corrections to earlier deliverables

| Where | Change |
|---|---|
| D13 13.5 PH-16 | Adds P16.5 (DEC-281) |
| D07 Part 4 7.42 | `transport_routes.capacity` (DEC-282) |
| D07 Part 3 7.27 and D11 Part 1 DEC-176 | Adds system roles `accounts_payable` and `expense_default` (DEC-285) |
| D13 13.4 M12b | Adds the external exam R2 tables (DEC-266), `admission_inquiries` and `admission_assessments` |
| D08 8.15 | Sanctum and the device endpoint start in P17.3 (DEC-283) |

---

## 14.23 PH-11 Online Fee Payments

**Exit criteria:**
- MS-12 is reached.
- The webhook flood load test shows zero double postings.
- A browser "success" return never changes a voucher.

### P11.0 UG-2 Slot (if not done in P09.5)

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read docs/decisions/ug-2-deferred.md (written in P09.5), D12 Part 2 DEC-200 and 12.31, D13 DEC-221 and 13.8, docs/runbooks/upgrade-gate.md, and the official Laravel 14 upgrade guide and release notes read at the time of the step.

PHASE GOAL
Complete the Laravel 14 upgrade deferred from P09.5 before any R2 feature work, within the DEC-200 limit (Laravel 13 security fixes end 17 Mar 2028). Skip this step entirely if P09.5 completed the upgrade.

ALREADY BUILT
R1 in production on Laravel 13 and PHP 8.4; the P09.5 report with the deferral reasons.

TASKS
1. Re-check every condition recorded in ug-2-deferred.md (point releases, package support for every composer dependency). If a package still lacks support, record the blocker with a dated plan and stop; the platform owner decides whether to replace the package.
2. Apply the upgrade exactly as P09.5 tasks 2 and 3 describe (upgrade guide checklist, forbidden-feature rules kept, no new Larastan ignores, full suite, nightly, mutation, browser, load and isolation runs).
3. Deploy to staging, run the full load set, then deploy to production in the window with the deploy pipeline and watch for 30 minutes.
4. Update the upgrade runbook and decisions.md; delete ug-2-deferred.md in the same pull request.

DATABASE
No schema changes.

BUSINESS RULES
- No behaviour changes; production tenants see no difference.

UI SPEC
- No UI changes; before and after screenshots of five key screens.

SECURITY AND PERMISSIONS
- D12 12.24 suite green before and after; dependency audit clean.

TESTS
- All suites and gates green; production smoke tests green after deploy.

ACCEPTANCE CRITERIA
1. Production runs Laravel 14 with all gates green, or a dated blocker record approved by the platform owner exists.
2. CI green; staging and production green.

DO NOT
- Do not combine this upgrade with feature work.
- Do not deploy outside the window.

OUTPUT
Report per Master section 20 with condition evidence, the checklist and production watch results.
```

### P11.1 Gateway Accounts and 1Bill

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D11 Part 1 11.10 (1Bill flow) and 11.11 (F9, F10); D07 Part 3 7.26 (payment_gateway_accounts, payment_gateway_transactions, fee_vouchers.consumer_number); D03 3.15 (adapter contracts); D10 Part 3 SCR-FEE-034, 035, 037 (consumer number); D12 Part 1 12.8 (webhooks) and DEC-196; D05 5.7.13 (fee.gateway); D14 DEC-286.

PHASE GOAL
Let guardians pay vouchers through any bank app with 1Bill: organization gateway accounts with encrypted credentials and a test transaction, consumer numbers printed on vouchers, the bill inquiry and payment notification adapter on the hooks host, and the verification-first posting path that the JazzCash and Easypaisa adapters will reuse.

ALREADY BUILT
R1 in production: vouchers with pre-rendered challans, RecordPayment with idempotency and allocations, webhook_receipts stored first, JournalPoster, integrity checks.

TASKS
1. Migrations (Fee, expand-only): payment_gateway_accounts and payment_gateway_transactions exactly as D07 7.26; DEC-176 R2 system roles payment_gateway_clearing and bank_charges seeded into every existing chart by a data backfill job (idempotent).
2. Contracts (Modules\Fee\Contracts\Gateways): PaymentGateway (initiate(voucher, transaction): RedirectInstruction; verify(transaction): VerificationResult with status, amount, currency, provider id; parseCallback(request): CallbackData; verifySignature(request): bool), BillPaymentAggregator (inquiry(consumerNumber): BillInquiryResult; notify(request): BillPaymentData; acknowledge(result)); FakeGateway and FakeAggregator for tests.
3. 1Bill adapter (OneBillAggregatorAdapter) built strictly from the aggregator's official integration document provided under the merchant agreement (store it in docs/integrations/onebill/ outside version control if confidential; record the version). Endpoints on hooks host: hooks.fee.onebill.inquiry and hooks.fee.onebill.payment under /webhooks/onebill/{organization_uuid}, authenticated as the document requires (credentials and IP allowlist from the gateway account); requests stored in webhook_receipts first (DEC-42); inquiry answers from the voucher (name, amount within due date, amount after due date with the late fee rule, due date, status) without exposing any other data; payment notifications processed by job ProcessOneBillPayment on queue critical.
4. Consumer numbers: AssignConsumerNumber on voucher generation when the organization has an active 1Bill account (format from the aggregator, unique per uq_fvc_consumer); printed on the challan and shown on SCR-FEE-037.
5. Verification-first posting (shared by all providers): ConfirmGatewayPayment (looks up the transaction, calls the provider's verification or the aggregator's settled data, checks status paid, amount equals the expected amount and currency PKR, then RecordPayment with method onebill or online and payment_gateway_transaction_id (uq_fpy_gateway), posting F9 Dr payment_gateway_clearing Cr fee_receivable; a duplicate notification returns the same receipt; a payment for an already-paid voucher becomes advance credit plus an attention item for the accountant (D11 11.10)).
6. Gateway accounts: CreateGatewayAccount and UpdateGatewayAccount (provider, branch optional, merchant id, credentials encrypted and never shown again, sandbox or live, settlement bank account), RunTestTransaction (sandbox only; end-to-end through the fake voucher), DeactivateGatewayAccount.
7. Settlement (F10): SettleGatewayPayments (from the aggregator's settlement report import or a manual entry: Dr bank (net), Dr bank_charges (fee), Cr payment_gateway_clearing (gross)), import type gateway_settlements.
8. Integrity checks: gateway clearing equals verified payments not yet settled (D11 11.13), registered in IntegrityCheckRegistry.
9. Screens (D10 exactly): SCR-FEE-034 Payment gateway accounts, SCR-FEE-035 Gateway transactions (initiated, verified, failed, expired; re-verify action), consumer number on SCR-FEE-037 and on the challan PDF.
10. Definitions: Permissions.php for fee.gateway (sensitive) per D05 5.7.13; Settings fee.online_partial_payments (off, A-90); attention items gateway payments awaiting verification.
11. decisions.md rows: DEC-42, DEC-166, DEC-196, DEC-286.

DATABASE
payment_gateway_accounts (pga), payment_gateway_transactions (pgt) per D07 Part 3 7.26; data backfill of system role accounts.

BUSINESS RULES
- A payment posts only after verification of status, amount and currency (DEC-166, DEC-286); notifications alone never post.
- Each organization's own merchant credentials are used; funds settle to its bank.
- One posted payment per provider transaction (uq_fpy_gateway); duplicates return the same receipt.
- Inquiry responses reveal only what is printed on the voucher.

UI SPEC
- Gateway form masks credentials after save, shows mode badge (sandbox or live) and last test result.
- Transactions list with status pills and a re-verify button.

SECURITY AND PERMISSIONS
- Credentials encrypted and never logged; webhook endpoints verify authentication and IP allowlist; stored first.
- fee.gateway is sensitive (2FA); every account change audited.

TESTS
- Inquiry returns correct amounts before and after due date; unknown consumer number answer per the document without data leakage.
- Payment notification: verified once, posted once; duplicate returns same receipt; amount mismatch not posted and flagged.
- Already-paid voucher payment becomes advance plus attention item.
- F9 and F10 named posting tests; integrity check for gateway clearing.
- Payment trust test: no client message or redirect changes a voucher.
- Screen tests, audit, isolation suite (a notification for organization A can never touch organization B), RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. In the aggregator sandbox, a voucher is inquired and paid through the test bank app flow and the receipt appears with balanced journals.
2. CI green; staging green.

DO NOT
- Do not post from unverified notifications.
- Do not store card or bank login data; none is ever received.
- Do not invent API fields; follow the official document only.

OUTPUT
Report per Master section 20 with the sandbox walkthrough, the integration document version and posting test list.
```

### P11.2 JazzCash and Easypaisa Hosted Payments

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D11 Part 1 11.10 (gateway flow); D10 Part 3 SCR-FEE-038, SCR-FEE-039 and wireframe W-14, DEC-166; D12 Part 1 DEC-196; D14 P11.1 contracts and DEC-286.

PHASE GOAL
Let guardians pay a voucher online from the portal through JazzCash and Easypaisa hosted or redirect checkout, with server-side verification before any posting, a "Verifying" return page, re-verification of pending transactions and expiry.

ALREADY BUILT
P11.1: gateway accounts, transactions, PaymentGateway contract, ConfirmGatewayPayment, F9 and F10 postings, guardian voucher screens.

TASKS
1. Adapters JazzCashGateway and EasypaisaGateway implementing PaymentGateway strictly from each provider's official merchant integration guide received under the organization's merchant agreement (record guide versions in docs/integrations/; never copy code from unofficial sources): hosted or redirect checkout only (DEC-196); request signing and response or callback signature verification as each guide specifies; status inquiry API used by verify().
2. Actions: InitiateOnlinePayment (guardian with portal access to the child; one open voucher; amount equals outstanding (partial only when fee.online_partial_payments is on); creates payment_gateway_transactions with a unique merchant_txn_ref and an idempotency key; returns the redirect instruction; refuses when the voucher has another transaction in initiated or pending within 15 minutes), HandleGatewayCallback (POST /webhooks/{provider}/{organization_uuid} on hooks host: stored first, signature verified, 200 returned fast, job ProcessGatewayCallback on queue critical which calls ConfirmGatewayPayment), HandleGatewayReturn (guardian browser return: shows SCR-FEE-039 "Verifying your payment" and polls the server; never trusts query parameters), ReverifyPendingTransactions (command fee:reverify-gateway every 10 minutes for transactions pending more than 15 minutes; expire after 24 hours, D11 11.10).
3. Guardian notifications: fee.online_payment_received (receipt), fee.online_payment_failed.
4. Screens (D10 exactly): SCR-FEE-038 Guardian: pay online (method choice, amount, redirect), SCR-FEE-039 Guardian: payment result (DEC-166 states: verifying, success with receipt, failed with retry); "Pay now" on SCR-FEE-037 (W-14).
5. Reconciliation hooks for P11.3: each transaction records verification_method (callback, inquiry, reconciliation) and last_response with secrets removed.
6. decisions.md rows: DEC-166, DEC-196, DEC-286.

DATABASE
No new tables; uses payment_gateway_transactions and fee_payments.

BUSINESS RULES
- The browser return never changes a voucher; only server verification does (DEC-166).
- One open online transaction per voucher at a time; amounts are fixed at initiation.
- Expired transactions never post, even if a late callback arrives; a late callback is verified and, if paid, posts normally (the provider's confirmed money must be recorded) with an attention item.

UI SPEC
- W-14 at 360px: voucher summary, Pay button, provider choice, verifying screen with spinner text in English and Urdu, success with receipt download, failure with retry.

SECURITY AND PERMISSIONS
- Only guardians with portal access to the child can initiate; EnsureChildContext applies.
- Signatures verified with constant-time comparison; secrets never logged; stored-first receipts.

TESTS
- Signature verification good and bad for each adapter using the guides' documented test vectors.
- Return with forged success parameters: voucher unchanged (payment trust test).
- Callback flood with duplicates posts once; late callback after expiry handled per rule; pending re-verification posts or expires.
- Amount or currency mismatch never posts.
- Screen tests for portal screens, isolation suite, RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. In both providers' sandboxes a guardian pays a voucher, sees "Verifying", then success with a receipt; journals balanced.
2. CI green; staging green.

DO NOT
- Do not collect card numbers or wallet PINs in any form.
- Do not post on redirect or on unverified callbacks.

OUTPUT
Report per Master section 20 with sandbox evidence for both providers and the trust test results.
```

### P11.3 Reconciliation, Webhook Flood and Online Posting Rules

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D11 Part 1 11.10, 11.11 (F9, F10) and 11.13; D12 Part 2 12.22 (webhook flood scenario) and 12.24 (webhooks, payment trust); D13 PH-11 exit criteria; D14 DEC-286.

PHASE GOAL
Prove online money is safe at scale: automated reconciliation of provider reports against transactions and payments, settlement posting, the webhook flood load test with zero double postings, named tests for every online posting rule, and close PH-11.

ALREADY BUILT
P11.1 and P11.2: 1Bill, JazzCash and Easypaisa adapters, verification-first posting, settlement action.

TASKS
1. ReconcileGatewayReport (import type gateway_reports per provider mapping): matches provider rows to payment_gateway_transactions by provider id or merchant ref; outcomes matched, missing_in_platform (verified paid at provider but no payment: run ConfirmGatewayPayment), missing_at_provider (payment without provider row: attention item, never auto-reversed), amount_mismatch (attention item); marks matched transactions reconciled.
2. Daily command fee:reconcile-gateways at 06:00 for providers that offer a reconciliation or transaction listing API per their official guides; otherwise the accountant imports the report.
3. Settlement posting via SettleGatewayPayments in bulk from the reconciled report (F10).
4. Webhook flood (D12 12.22): k6 script webhook-flood.js with 100 callbacks per second and 20% duplicates against staging fakes; acknowledgement under 500 ms; zero double postings verified by uq_fpy_gateway and a count query; results in docs/evidence/load/R2/.
5. Named posting tests F9 and F10 (moving them from skipped to active in tests/Feature/Posting), plus reconciliation outcome tests.
6. Runbook payment-provider-outage.md updated with reconciliation after recovery.
7. decisions.md rows: DEC-286, D11 11.13 gateway control.

DATABASE
No new tables.

BUSINESS RULES
- Reconciliation never reverses or deletes payments automatically; mismatches go to people.
- Settlement entries balance gross, net and charges exactly.

UI SPEC
- Reconciliation result screen inside SCR-FEE-035 with tabs per outcome and actions per row.

SECURITY AND PERMISSIONS
- Report imports need fee.gateway.manage; every reconciliation action audited.

TESTS
- Reconciliation outcomes; settlement postings; flood test zero double postings; payment trust test across all three providers.

ACCEPTANCE CRITERIA
1. Webhook flood evidence with zero double postings; payment trust tests green (PH-11 exit criteria).
2. A pilot organization's first live online payment posts from a verified callback (MS-12).
3. CI green; staging and production green.

DO NOT
- Do not auto-reverse payments.
- Do not relax verification to speed up posting.

OUTPUT
Report per Master section 20 with reconciliation examples, flood results and MS-12 evidence. Then run the D13 13.6 gate for PH-11.
```

---

## 14.24 PH-12 Accounting R2

**Exit criteria:**
- Financial statements tie out to journals on the demo dataset.
- Every new posting path is covered by `assertJournalBalanced`.

### P12.1 Vendors, Expenses, Purchases and Vendor Payments

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 3 7.27 R2 tables (vendors, expense_vouchers, purchase_bills, purchase_bill_lines, vendor_payments); D11 Part 1 11.4 (expense, purchase); D10 Part 3 SCR-ACC-015 to 018; D05 5.7.14; D14 DEC-285.

PHASE GOAL
Record what the school spends: vendors, expense vouchers with receipts and approval, purchase bills with lines and approval, vendor payments against bills, all posted through JournalPoster.

ALREADY BUILT
R1 accounting core with JournalPoster, system accounts, period locks, approval engine; P11 online payments.

TASKS
1. Migrations (Accounting, expand-only): vendors, expense_vouchers, purchase_bills, purchase_bill_lines, vendor_payments exactly as D07 7.27; backfill system roles accounts_payable and expense_default into every chart (DEC-285).
2. Actions: CreateVendor and UpdateVendor (NTN optional, payable account default accounts_payable), CreateExpenseVoucher (expense account, amount, payment method, paid from cash or bank, vendor optional, receipt upload; SubmitApprovalRequest expense), PostExpense (handler: E1 Dr expense account (branch line) Cr paid-from account), CreatePurchaseBill (lines with account, quantity, unit price; total equals lines; SubmitApprovalRequest purchase), PostPurchaseBill (handler: E2 Dr each line's account Cr vendor payable with party vendor), RecordVendorPayment (against an approved bill or on account; E3 Dr vendor payable Cr bank or cash; bill status partially_paid or paid), CancelPurchaseBill (unpaid only; reversal entry dated today).
3. Approval types (Accounting registers): expense (1 principal_of_branch; 2 owner above accounting.expense_owner_threshold) and purchase (same shape); approve permissions accounting.expense.approve and accounting.purchase.approve.
4. Screens (D10 exactly): SCR-ACC-015 Expense vouchers, SCR-ACC-016 Vendors, SCR-ACC-017 Purchase bills, SCR-ACC-018 Vendor payments.
5. Settings accounting.expense_owner_threshold, accounting.purchase_owner_threshold; numbering keys expense_voucher, vendor_payment.
6. decisions.md rows: DEC-285.

DATABASE
vendors (vnd), expense_vouchers (exv), purchase_bills (pbl), purchase_bill_lines (pbn), vendor_payments (vpy) per D07 Part 3 7.27.

BUSINESS RULES
- Expenses and purchases post only after approval; corrections are reversals dated today.
- Vendor payments never exceed the bill's outstanding amount unless recorded on account.

UI SPEC
- Purchase bill grid with live totals; receipt viewer on expense detail.

SECURITY AND PERMISSIONS
- Approval rules and audit as for every money action; impersonated sessions cannot approve.

TESTS
- Named posting tests E1, E2, E3 and the cancellation reversal with assertJournalBalanced; approval thresholds; overpayment refused.
- Screen tests, isolation suite, RBAC fixture, ScreenCoverageTest.

ACCEPTANCE CRITERIA
1. On staging an expense and a purchase pass approval and post; a vendor payment settles the bill.
2. CI green; staging green.

DO NOT
- Do not write journals outside JournalPoster.
- Do not build inventory receipts (R3).

OUTPUT
Report per Master section 20 with posting tests and screenshots.
```

### P12.2 Bank Reconciliation

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 3 7.27 (bank_statement_lines, bank_reconciliations); D10 Part 3 SCR-ACC-019, 020; D11 Part 1 11.13; D05 5.7.14.

PHASE GOAL
Let accountants reconcile each bank account monthly: statement import with per-bank mappings, suggested matches against ledger lines, manual matching, reconciliation reports with the difference brought to zero.

ALREADY BUILT
Accounting core with bank accounts and ledgers, P12.1 vendor payments, P11 gateway settlements, import engine.

TASKS
1. Migrations (expand-only): bank_statement_lines, bank_reconciliations exactly as D07 7.27.
2. Import type bank_statements (mapping per bank saved as import mappings; date formats; debit and credit or signed amount; duplicate lines detected by date, amount and reference).
3. MatchSuggester: candidates from journal lines on the bank's ledger account within plus or minus 3 days and the same amount, scored by reference similarity; one-to-one, one-to-many (deposit of many receipts) and many-to-one support.
4. Actions: MatchStatementLine, UnmatchStatementLine, IgnoreStatementLine (reason), CreateAdjustingJournal from an unmatched line (bank charges, interest) through JournalPoster, CompleteReconciliation (statement balance, book balance, outstanding items listed; difference must be zero; status reconciled; locks the matched lines).
5. Screens (D10 exactly): SCR-ACC-019 Bank statement import, SCR-ACC-020 Bank reconciliation (two-pane grid with suggestions, difference counter).
6. decisions.md rows: reconciliation rules.

DATABASE
bank_statement_lines (bsl), bank_reconciliations (brc) per D07 Part 3 7.27.

BUSINESS RULES
- A reconciliation completes only with zero difference; completed periods cannot be re-matched without reopening by an owner with reason.

UI SPEC
- Left pane statement lines, right pane ledger lines, suggestions highlighted, keyboard match.

SECURITY AND PERMISSIONS
- accounting.reconciliation.create; audit on every match and completion.

TESTS
- Import duplicates; suggestion scoring; one-to-many matching; completion blocked with difference; adjusting journal balanced.

ACCEPTANCE CRITERIA
1. A month's statement for one bank reconciles to zero on staging.
2. CI green; staging green.

DO NOT
- Do not change posted journals during matching.

OUTPUT
Report per Master section 20 with the reconciliation walkthrough.
```

### P12.3 Financial Statements

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D10 Part 3 SCR-ACC-011 and SCR-ACC-021; D07 Part 3 7.27 (accounts paths and types); D13 PH-12 exit criteria; D05 5.7.14; D14 DEC-263 (report runner).

PHASE GOAL
Produce trustworthy statements by branch and for the organization: trial balance, income and expenditure (income statement), balance sheet and cash flow, all tying out to journals; close PH-12.

ALREADY BUILT
Accounting core and R2 accounting, report runner, demo finance seeder.

TASKS
1. StatementBuilder queries on the replica: income and expenditure for a period (income and expense groups by path), balance sheet as of a date (assets, liabilities, equity including current year surplus), cash flow for a period (indirect method from surplus with working capital movements in receivables, deferred income, advance fees, payables, and investing and financing sections from tagged accounts); by branch or consolidated; comparative previous period column.
2. Account tagging for cash flow: accounts gain a cash_flow_class setting stored in account metadata through UpdateAccount (operating, investing, financing, cash), seeded for template accounts.
3. Screens (D10 exactly): SCR-ACC-021 Financial statements (statement selector, period, branch, export and print), trial balance enhancements on SCR-ACC-011 (comparative columns).
4. Tie-out tests and a statements consistency check registered in IntegrityCheckRegistry (balance sheet balances; cash flow closing equals cash and bank balances).
5. demo:seed-finance extended with R2 expenses, purchases, payments and reconciliations.
6. decisions.md rows: cash flow method.

DATABASE
No new tables; account metadata column if absent (expand-only JSON column metadata on accounts).

BUSINESS RULES
- Statements are read-only views of posted journals; no figure is typed in.

UI SPEC
- Statements in A4 print layout with organization letterhead, Urdu headings available.

SECURITY AND PERMISSIONS
- accounting.statement.view; exports masked of party details.

TESTS
- Balance sheet balances; income statement equals change in equity; cash flow ties to cash accounts; branch sums equal consolidated.

ACCEPTANCE CRITERIA
1. Statements tie out on the demo dataset; every new posting path has assertJournalBalanced coverage (PH-12 exit criteria).
2. CI green; staging green.

DO NOT
- Do not allow manual statement figures.

OUTPUT
Report per Master section 20 with statement samples and tie-out evidence. Then run the D13 13.6 gate for PH-12.
```

---

## 14.25 PH-13 HR R2 and Payroll

**Exit criteria:**
- MS-13 is reached.
- A payroll run reruns identically until it is finalized.
- Finalized runs are immutable.

### P13.1 HR R2

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 2 7.21 R2 HR tables (employee_contracts, performance_reviews, training_records, employee_disciplinary_actions, employee_separations); D10 Part 2 SCR-HR-014 to 018; D11 Part 1 11.4 (employee_separation) and 11.30 (separation flow); D05 5.7.10; D11 DEC-178.

PHASE GOAL
Complete HR records for R2: contracts with renewal reminders, performance reviews with acknowledgement, training records, HR disciplinary records, and the separation workflow with clearance through approval.

ALREADY BUILT
R1 HR, attendance, leave, approvals, ClearanceCheck patterns.

TASKS
1. Migrations (expand-only): employee_contracts, performance_reviews, training_records, employee_disciplinary_actions, employee_separations exactly as D07 7.21.
2. Actions: CreateContract and RenewContract (reminder date; command hr:contract-reminders daily), CreatePerformanceReview (criteria JSON from a review template setting), SubmitReview, AcknowledgeReview (employee), ApproveReview (principal), RecordTraining, RecordHrDisciplinaryAction (sensitive), RequestSeparation (SubmitApprovalRequest employee_separation with clearance items: assets, library, advances, loans as registered checks in EmployeeClearanceCheckRegistry), ApplySeparation (handler: status separated on last day, user deactivated on the last day by hr:apply-separations, device enrollments ended, teaching assignments flagged for reassignment, final settlement run requested from Payroll in P13.3).
3. Approval type employee_separation (1 principal_of_branch, 2 owner; approve permission hr.separation.approve; employee version snapshot).
4. Screens (D10 exactly): SCR-HR-014 Contracts, SCR-HR-015 Performance reviews, SCR-HR-016 Training records, SCR-HR-017 Disciplinary actions, SCR-HR-018 Separations. Employee profile tabs added.
5. decisions.md rows: DEC-178.

DATABASE
employee_contracts (ect), performance_reviews (prv), training_records (trn), employee_disciplinary_actions (eda), employee_separations (esp) per D07 Part 2 7.21.

BUSINESS RULES
- Separation applies only after approval and clearance or recorded waivers; records are never deleted.
- Disciplinary records are sensitive and visible only with hr.disciplinary.view.

UI SPEC
- Review form with criteria ratings and comments; acknowledgement banner for the employee.

SECURITY AND PERMISSIONS
- Sensitive resources need 2FA; staff see their own reviews and training only.

TESTS
- Contract reminders; review acknowledgement flow; separation approval with clearance and user deactivation on the last day.

ACCEPTANCE CRITERIA
1. A separation passes principal and owner with clearance and deactivates the user on the last day on staging.
2. CI green; staging green.

DO NOT
- Do not calculate final settlement pay here (P13.3).

OUTPUT
Report per Master section 20 with the separation walkthrough.
```

### P13.2 Salary Components, Structures and Tax Slabs

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 4 7.40 Setup tables and DEC-115, DEC-117; D07 Part 4 7.52 PayrollTaxSlabSeeder and A-62; D14 Part 1 DEC-232 (payroll_tax_slabs created here); D10 Part 3 SCR-PAY-001 to 004; D05 5.7.20; D01 GAP-34; D14 DEC-287.

PHASE GOAL
Model pay as data: salary components with system roles and registered calculators, platform tax slabs per tax year with organization override, salary structures, employee salary setup with pay branch, encrypted bank details and statutory numbers.

ALREADY BUILT
R1 HR with salary histories (DEC-103), P13.1 HR R2, accounting with system roles.

TASKS
1. Migrations (Payroll, expand-only): salary_components, payroll_tax_slabs (platform rows with organization_id NULL plus organization overrides; organization_key generated; uq_ptx), salary_structures, salary_structure_components, employee_salaries (pay_branch_id, encrypted bank account and IBAN with last4 per DEC-117), employee_salary_components exactly as D07 7.40.
2. Seed system components per organization (editable names, locked roles): basic, house rent, medical, conveyance (earnings); income_tax, eobi_employee, social_security, absence_deduction, late_deduction, loan_recovery, advance_recovery, library_fine (deductions); eobi_employer (employer contribution); each mapped to a ledger account (salary_expense, payables per DEC-176 R2 roles: income_tax_payable, eobi_payable, social_security_payable, employee_loans, employee_advances).
3. Calculator registry (DEC-287) in Modules\Payroll\Calculators with formula keys income_tax_annualized, eobi_employee, eobi_employer, social_security, absence_deduction, late_deduction, percent_of_basic, percent_of_gross; each reads rates and bases from data (payroll_tax_slabs, settings with effective dates payroll.eobi_wage_base, payroll.eobi_employee_percent, payroll.eobi_employer_percent, payroll.social_security_*); unit tests per calculator with fixtures, never with values typed into production code.
4. Platform tax slab entry: platform screen inside SCR-PLT pages (platform super admin) to enter slabs per tax year; organization override rows through SCR-PAY-002 with permission; a missing slab set for the current tax year blocks payroll calculation with a clear message.
5. Actions: CreateSalaryComponent, UpdateSalaryComponent (system role locked), CreateSalaryStructure, SetEmployeeSalary (structure, basic from HR history, pay branch (DEC-115), payment method, bank details encrypted, EOBI and social security numbers, NTN, tax exempt flag; one active per uq_esl_active; effective dated), SetEmployeeComponentOverride.
6. Screens (D10 exactly): SCR-PAY-001 Salary components, SCR-PAY-002 Income tax slabs, SCR-PAY-003 Salary structures, SCR-PAY-004 Employee salary setup.
7. decisions.md rows: DEC-115, DEC-117, DEC-232, DEC-287, A-62, GAP-34.

DATABASE
salary_components (slc), payroll_tax_slabs (ptx), salary_structures (sst), salary_structure_components (ssc), employee_salaries (esl), employee_salary_components (esc) per D07 Part 4 7.40.

BUSINESS RULES
- No tax rate, slab, wage base or percentage lives in code (A-62, DEC-287).
- One active salary setup and one pay branch per employee (DEC-115).
- Bank details are encrypted and shown only as last4.

UI SPEC
- Salary setup shows a monthly preview computed by the calculators with the data in force.

SECURITY AND PERMISSIONS
- All payroll screens are sensitive (2FA, D05 5.7.20); audit with masked values.

TESTS
- Calculators with slab fixtures across slab boundaries; annualized tax with mid-year joiner and YTD withholding; EOBI with wage base changes by effective date; tenant cannot edit platform slabs.

ACCEPTANCE CRITERIA
1. A test organization configures structures and salaries; previews match hand-calculated fixtures.
2. CI green; staging green.

DO NOT
- Do not hardcode statutory values.
- Do not run payroll (P13.3).

OUTPUT
Report per Master section 20 with calculator test tables and salary setup screenshots.
```

### P13.3 Payroll Runs, Advances, Loans and Postings

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D11 Part 1 11.12 completely (payroll run flow, posting rules P1 to P6), 11.4 (payroll_run, salary_advance, employee_loan); D07 Part 4 7.40 Runs, Advances and loans tables with DEC-115, DEC-116, DEC-125; D10 Part 3 SCR-PAY-005 to 012, 014 and wireframe W-16; D05 5.7.20 and SOD-06; D13 PH-13 exit criteria; D14 DEC-287.

PHASE GOAL
Pay staff correctly once: payroll runs per pay branch and month from locked staff attendance and approved leave, review with holds and one-time lines, approval with separation of duties, finalization with postings, salary payments with bank upload files, advances and loans with recovery, supplementary and final settlement runs.

ALREADY BUILT
P13.1 and P13.2; R1 staff attendance with period locks, leave with approved unpaid days, library fines contributor pattern (DEC-124), JournalPoster, approval engine, export engine.

TASKS
1. Migrations (expand-only): payroll_runs, payslips, payslip_lines, salary_advances, employee_loans, loan_installments, salary_payments exactly as D07 7.40.
2. Actions: CreatePayrollRun (branch, month, run type regular, supplementary or final_settlement; refused while the staff_attendance period for the month is unlocked (D07 7.40); uq_prr_regular), CalculatePayrollRun (bulk operation in chunks of 50 employees: for each employee whose pay branch is the run's branch: components from structure and overrides, attendance figures from staff_attendances in the window (present, paid leave, unpaid leave, absent, late count), calculators per DEC-287, loan installments due and advance recoveries, payslip deduction contributors (PayslipLineContributorRegistry: Library fines in R3), arrears lines; deterministic so a rerun with the same inputs produces identical payslips; status calculated), RecalculatePayrollRun (replaces draft payslips; allowed until approval), HoldPayslip and ReleasePayslip (reason), AddOneTimeLine (reason required), SubmitPayrollRun (SubmitApprovalRequest payroll_run with run totals and payslip snapshot), FinalizePayrollRun (after approval; payslips final; P1 posting per D11 11.12; status posted; payslips immutable (DEC-116)), ExportBankUploadFile (export engine; bank formats as mappings; masked except for the export permission), RecordSalaryPayments (per payslip or batch; P2; uq_spy_payslip), ReverseSalaryPayment (P6, reason), CancelPayrollRun (before finalization only; clears regular markers), RequestSalaryAdvance and RequestEmployeeLoan (staff self-service; approval types salary_advance and employee_loan per DEC-178), DisburseAdvance (P3) and DisburseLoan (P4) with installment schedules, DeferLoanInstallment (reason, approval by owner).
3. Approval types (Payroll registers): payroll_run (1 principal_of_branch, 2 owner; SOD-06 preparer cannot approve; run version snapshot so a recalculated run fails apply), salary_advance (1 principal_of_branch, 2 owner above payroll.advance_owner_threshold), employee_loan (1 owner).
4. Final settlement: run type final_settlement for an employee with an approved separation: prorated pay to the last day, leave encashment per policy setting, recoveries of outstanding advances and loans, payable or receivable shown.
5. Integrity: employee_loans.outstanding_amount cache verified nightly (DEC-126); payslip net equals gross minus deductions (CHECK).
6. Screens (D10 exactly): SCR-PAY-005 Payroll runs, SCR-PAY-006 New payroll run, SCR-PAY-007 Payroll run review (W-16), SCR-PAY-009 Finalize payroll, SCR-PAY-010 Salary payments, SCR-PAY-011 Salary advances, SCR-PAY-012 Employee loans, SCR-PAY-014 Request advance or loan. Widget payroll.run_status and attention item for runs awaiting approval or payment.
7. decisions.md rows: DEC-115, DEC-116, DEC-125, DEC-178, DEC-287.

DATABASE
payroll_runs (prr), payslips (psl), payslip_lines (pll), salary_advances (sva, DEC-293), employee_loans (eln), loan_installments (lin), salary_payments (spy) per D07 Part 4 7.40.

BUSINESS RULES
- One regular payslip per employee per month across the organization (DEC-115).
- A run cannot calculate while the month's staff attendance is unlocked.
- Approved and finalized payslips never change; corrections go to supplementary runs or next-month arrears or recovery lines (DEC-116).
- The preparer of a run cannot approve it (SOD-06).

UI SPEC
- W-16 grid with totals, holds, row actions and the attendance lock indicator.

SECURITY AND PERMISSIONS
- Sensitive throughout; bank upload files downloadable only by holders of payroll.payment export permission and expire in 24 hours.

TESTS
- Determinism: two calculations with the same inputs produce byte-identical payslip snapshots (PH-13 exit criterion).
- Immutability: edits to finalized payslips refused at the Action level.
- Shared teacher paid only once through pay branch (uq_psl_regular).
- Named posting tests P1 to P6 with assertJournalBalanced.
- Loan recovery and deferral; advance recovery; final settlement calculation.
- SOD-06 enforcement; recalculated run fails a pending approval at apply.
- Screen tests, isolation suite, RBAC fixture, ScreenCoverageTest; mutation testing on calculators and CalculatePayrollRun with MSI at least 80%.

ACCEPTANCE CRITERIA
1. On staging a branch's payroll is calculated, reviewed, approved, finalized and paid with a bank upload file; P1 and P2 journals balanced.
2. CI green; staging green.

DO NOT
- Do not change finalized runs.
- Do not calculate from unlocked attendance.

OUTPUT
Report per Master section 20 with determinism evidence, posting tests and W-16 screenshots.
```

### P13.4 Payslips, Deduction Reports and Self-Service

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D10 Part 3 SCR-PAY-008 and SCR-PAY-013; D10 Part 3 10.47 (payroll.salary_sheet, payroll.deductions); D14 P02.4 (document engine); D13 PH-13 exit criteria.

PHASE GOAL
Deliver payslips and payroll reports: payslip PDFs through the document engine, staff self-service payslips, the salary sheet and statutory deduction reports, and close PH-13.

ALREADY BUILT
P13.1 to P13.3.

TASKS
1. Document type payslip (printable, pre-rendered at finalization per DEC-215; English and Urdu; masked bank last4; employer details; component breakdown; year-to-date tax).
2. Reports (ReportDefinition): payroll.salary_sheet, payroll.deductions (tax, EOBI, social security per month with employee numbers for filing), payroll.bank_transfer_summary.
3. Staff self-service: SCR-PAY-013 My payslips (download; own only); notification payroll.payslip_ready.
4. Screens (D10 exactly): SCR-PAY-008 Payslip, SCR-PAY-013 My payslips.
5. decisions.md rows: DEC-215 usage.

DATABASE
No new tables.

BUSINESS RULES
- Payslips are frozen snapshots; reprints reproduce the stored PDF.

UI SPEC
- Payslip A4 and A5 layouts by setting.

SECURITY AND PERMISSIONS
- Staff see only their own payslips; downloads through signed URLs.

TESTS
- Payslip rendering in both languages; reports totals equal payslip sums; own-only access.

ACCEPTANCE CRITERIA
1. MS-13: first payroll run finalized with payslips issued and P1 to P6 journals balanced.
2. CI green; staging green.

DO NOT
- Do not render payslips from current data after finalization.

OUTPUT
Report per Master section 20 with sample payslips and reports. Then run the D13 13.6 gate for PH-13.
```

---

## 14.26 PH-14 Learning (LMS)

**Exit criteria:**
- Phase gate G-1 to G-11.
- A student sees only courses targeted to their current placement, and progress saves at most once every 30 seconds per lecture.

### P14.1 Courses, Chapters, Lectures and Resources

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 4 7.37 (courses, course_targets, course_chapters, course_topics, lectures); D10 Part 3 SCR-LMS-001 to 004; D05 5.7.19; D12 Part 1 (uploads, CSP); D01 GAP-40.

PHASE GOAL
Let coordinators and subject teachers build courses per class and subject with chapters, topics and lectures (video links, uploaded videos within caps, documents, rich text, links), targeted to sections or teaching groups, copied from earlier sessions, and published.

ALREADY BUILT
R1 academic structure with teaching assignments and groups, FileStorageService, document scanning, CSP with nonces.

TASKS
1. Migrations (Learning, expand-only): courses, course_targets, course_chapters, course_topics, lectures exactly as D07 7.37.
2. Actions: CreateCourse, CopyCourseFromSession, UpdateCourse, PublishCourse, ArchiveCourse, SetCourseTargets (sections or teaching groups of the class; exactly one per target row), CreateChapter, CreateTopic, CreateLecture (types video_url (YouTube or Vimeo only; embedded through privacy-enhanced embed hosts; CSP frame-src allowlist limited to those hosts), video_file (size cap from setting learning.max_video_mb and plan storage quota from P18.1 when present), document (PDF), rich_text (sanitized), link), ReorderItems (move up and down buttons plus drag, WCAG 2.5.7), PublishLecture with available_from.
3. Screens (D10 exactly): SCR-LMS-001 Courses, SCR-LMS-002 Course builder, SCR-LMS-003 Lecture form, SCR-LMS-004 Course targets.
4. Settings learning.max_video_mb (default 200), learning.allowed_video_hosts (youtube, vimeo).
5. decisions.md rows: video host allowlist, GAP-40 interim caps.

DATABASE
courses (crs), course_targets (crt), course_chapters (cch), course_topics (ctp), lectures (lec) per D07 Part 4 7.37.

BUSINESS RULES
- Rich text is sanitized; only allowlisted video hosts embed; uploads respect caps.
- A course's targets must belong to its class.

UI SPEC
- Course builder tree with chapters, topics and lectures; preview as student.

SECURITY AND PERMISSIONS
- learning.course.manage for AC and HOD; learning.lecture.create for subject teachers of the class subject.

TESTS
- Sanitization; video URL host validation; upload cap; targets validation; copy from session.

ACCEPTANCE CRITERIA
1. A course with mixed lecture types is built, targeted and published on staging.
2. CI green; staging green.

DO NOT
- Do not allow arbitrary iframes or scripts.

OUTPUT
Report per Master section 20 with course builder screenshots.
```

### P14.2 Student Learning, Progress and Discussions

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 4 7.37 (lecture_progress, discussion_threads, discussion_posts, content_reports) and its rules; D10 Part 3 SCR-LMS-005 to 011; D05 5.7.19; D10 Part 1 widgets learning.course_progress and student.continue_learning.

PHASE GOAL
Give students a clean learning view with progress tracking, moderated discussions with reporting, teacher progress reports and guardian visibility; close PH-14.

ALREADY BUILT
P14.1 courses and lectures; portals; widgets registry.

TASKS
1. Migrations (expand-only): lecture_progress, discussion_threads, discussion_posts, content_reports exactly as D07 7.37.
2. Visibility: CourseVisibilityQuery (student sees a course only when a target matches their current section or teaching group membership); guardians see progress of linked children.
3. Actions: RecordLectureProgress (throttled endpoint at most once every 30 seconds per lecture per student; watched seconds and percent; completion at 90% for video, open for documents), CreateDiscussionThread, PostReply (one level), HidePost and PinThread and LockThread (moderators), ReportContent (uq_crp), ActionContentReport.
4. Screens (D10 exactly): SCR-LMS-005 Discussion moderation, SCR-LMS-006 Content reports, SCR-LMS-007 Course progress, SCR-LMS-008 Student: courses, SCR-LMS-009 Student: lecture, SCR-LMS-010 Student: discussion thread, SCR-LMS-011 Guardian: learning progress. Widgets learning.course_progress and student.continue_learning.
5. decisions.md rows: progress rules.

DATABASE
lecture_progress (lpr), discussion_threads (dth), discussion_posts (dps), content_reports (crp) per D07 Part 4 7.37.

BUSINESS RULES
- Students see only targeted courses; progress throttled; posts sanitized and limited to 5,000 characters.

UI SPEC
- Lecture player page with next lecture, attachments and discussion; works at 360px.

SECURITY AND PERMISSIONS
- Moderation by subject teachers and coordinators; hidden posts kept with reason.

TESTS
- Visibility by placement change; throttle; moderation and reporting flows.

ACCEPTANCE CRITERIA
1. Phase exit criteria evidence; CI green; staging green.

DO NOT
- Do not build live classes or video conferencing.

OUTPUT
Report per Master section 20. Then run the D13 13.6 gate for PH-14.
```

---

## 14.27 PH-15 Homework and Assessment

**Exit criteria:**
- Phase gate G-1 to G-11.
- Quiz attempts honor the server deadline, auto-submit at timeout, and keep question snapshots (DEC-118).

### P15.1 Homework

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 4 7.38 and DEC-119; D11 Part 2 11.30 (homework flow); D10 Part 3 SCR-HWK-001 to 008; D05 5.7.19.

PHASE GOAL
Let teachers assign homework with online, offline or no submission, collect attempts with full history, review with marks and feedback or request resubmission, and show it to students and guardians, with a diary line created automatically.

ALREADY BUILT
R1 diary, teaching assignments and groups, FileStorageService, portals.

TASKS
1. Migrations (expand-only): homework_assignments, homework_targets, homework_submissions, homework_submission_reviews exactly as D07 7.38.
2. Actions: CreateHomework (targets section, group or individual student; due, late rules; creates a diary line through Communication), PublishHomework, SubmitHomework (new attempt row; previous attempt superseded; is_late computed; files via file_attachments), ReviewSubmission (marks within max, outcome accepted or resubmit), ExcuseStudent, CloseHomework.
3. Not-submitted list computed as roster minus current submissions (never stored).
4. Screens (D10 exactly): SCR-HWK-001 to 008. Widgets homework.to_review, guardian.child_homework_due, student.due_soon homework part.
5. decisions.md rows: DEC-119.

DATABASE
homework_assignments (hwa), homework_targets (hwt), homework_submissions (hws), homework_submission_reviews (hwr) per D07 Part 4 7.38.

BUSINESS RULES
- Attempts are kept; only the current attempt counts; late submissions allowed only until late_until.

UI SPEC
- Camera upload on mobile; attempt history visible to student and teacher.

SECURITY AND PERMISSIONS
- Teachers see their assignments; guardians submit for young children only where enabled.

TESTS
- Attempt history and current marker; late rules; review marks bound; not-submitted computation.

ACCEPTANCE CRITERIA
1. Homework assigned, submitted twice, reviewed with resubmission on staging.
2. CI green; staging green.

DO NOT
- Do not delete attempts.

OUTPUT
Report per Master section 20.
```

### P15.2 Question Bank

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 4 7.39 (question_banks, questions, question_options, question_tags); D10 Part 3 SCR-ASM-001 to 004; D05 5.7.19.

PHASE GOAL
Build reusable question banks with sharing levels, seven question types, Urdu and math input, approval and versioning, and imports.

ALREADY BUILT
P15.1, R1 subjects, departments and import engine.

TASKS
1. Migrations (expand-only): question_banks, questions, question_options, question_tags exactly as D07 7.39.
2. Actions: CreateQuestionBank (sharing private, department, branch, organization), CreateQuestion and UpdateQuestion (editing an approved question creates a new version), ApproveQuestion (HOD), RetireQuestion, TagQuestion.
3. Math input stored as LaTeX in the stem and rendered client-side with a bundled math renderer (KaTeX, CSP-safe build); Urdu stems RTL.
4. Import type questions (template per type).
5. Screens (D10 exactly): SCR-ASM-001 Question banks, SCR-ASM-002 Questions, SCR-ASM-003 Question editor, SCR-ASM-004 Import questions.
6. decisions.md rows: math rendering choice.

DATABASE
question_banks (qbk), questions (qst), question_options (qop), question_tags (qtg) per D07 Part 4 7.39.

BUSINESS RULES
- Sharing level controls visibility; approved questions are versioned, not edited in place.

UI SPEC
- Editor preview per type in both languages.

SECURITY AND PERMISSIONS
- Bank owners and HODs manage; sharing respected in queries.

TESTS
- Sharing visibility; versioning; import validation per type.

ACCEPTANCE CRITERIA
1. A department bank with all types, approved and tagged, on staging.
2. CI green; staging green.

DO NOT
- Do not allow script in stems.

OUTPUT
Report per Master section 20.
```

### P15.3 Quizzes and Online Exams

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 4 7.39 (quizzes, quiz_questions, quiz_targets, quiz_attempts, quiz_attempt_answers) and DEC-118, DEC-121; D11 Part 2 11.30 (quiz flow); D10 Part 3 SCR-ASM-005 to 013 and wireframe W-15; D05 5.7.19; D14 DEC-288.

PHASE GOAL
Run quizzes and online exams safely: building from banks with snapshots at publish, server-timed attempts with autosave, auto-submission at the deadline, auto-grading of objective items, manual grading of subjective items, result release, analytics, and exam-linked scores flowing into marks as drafts; close PH-15.

ALREADY BUILT
P15.1, P15.2, R1 examination with marks submissions.

TASKS
1. Migrations (expand-only): quizzes, quiz_questions, quiz_targets, quiz_attempts, quiz_attempt_answers exactly as D07 7.39.
2. Actions: CreateQuiz, AddQuestions (pick or randomize by tags and difficulty), PublishQuiz (snapshots questions into quiz_questions.question_snapshot (DEC-118)), StartAttempt (one open attempt per uq_qat_open; server deadline from time limit and closes_at; question order stored), SaveAnswer (rejected after deadline plus 30 seconds grace), SubmitAttempt, AutoSubmitExpired (command assessment:auto-submit every minute), GradeObjective (automatic on submit: single and multi choice, true false, numeric with tolerance, short answer and fill blank against accepted answers normalized), GradeSubjective (grading queue), VoidAttempt (reason), ForceClose, ReleaseResults, ImportMarksFromQuiz (DEC-288, Examination public action writing drafts).
3. Screens (D10 exactly): SCR-ASM-005 to SCR-ASM-013 including W-15.
4. Analytics: item difficulty, discrimination index and distractor analysis on SCR-ASM-009.
5. decisions.md rows: DEC-118, DEC-121, DEC-288.

DATABASE
quizzes (quz), quiz_questions (qqn), quiz_targets (qtt), quiz_attempts (qat), quiz_attempt_answers (qaa) per D07 Part 4 7.39.

BUSINESS RULES
- Question snapshots freeze at publish; later bank edits never change attempts (DEC-118).
- The server timer rules; answers after the deadline plus grace are rejected; one open attempt per student per quiz.
- Exam-linked scores enter marks only as drafts for the teacher to submit (DEC-288).

UI SPEC
- W-15 at 360px: server timer, autosave per answer, navigator, warning at 1 minute, offline message.

SECURITY AND PERMISSIONS
- Students see only targeted quizzes; results shown per show_results setting.

TESTS
- Deadline enforcement with clock skew; auto-submit; snapshot immutability; grading correctness per type; exam-linked draft import respects submission states.
- Load: 2,000 concurrent attempts autosaving on staging within budgets.

ACCEPTANCE CRITERIA
1. Phase exit criteria evidence; CI green; staging green.

DO NOT
- Do not trust client timers.

OUTPUT
Report per Master section 20 with grading tests and load evidence. Then run the D13 13.6 gate for PH-15.
```

---

## 14.28 PH-16 Engagement, Admissions and Exams R2

**Exit criteria:**
- Phase gate G-1 to G-11.
- Messaging safeguards (DEC-289) are proven by tests.
- WhatsApp sends only approved templates to consenting guardians.
- Approval escalation fires on time.

### P16.1 Messaging and Parent-Teacher Meetings

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 3 7.32 (message_threads, messages, ptm_events, ptm_slots, ptm_bookings); D11 Part 2 11.30 (messaging, PTM); D10 Part 3 SCR-COMM-007 to 012, 015, 016; D05 5.7.18; D01 GAP-37; D14 DEC-289.

PHASE GOAL
Let guardians and teachers talk safely about a child within school rules, and let schools run parent-teacher meetings with slot booking, reminders and attendance.

ALREADY BUILT
R1 communication, portals with child context, teaching assignments, notification engine, content_reports from P14.2.

TASKS
1. Migrations (expand-only): message_threads, messages, ptm_events, ptm_slots, ptm_bookings exactly as D07 7.32.
2. Actions: OpenThread (DEC-289 eligibility), SendMessage (immutable; attachments images and PDF re-encoded and scanned; outside office hours queued until opening), HideMessage (moderator with reason), CloseThread and LockThread, ReportMessage (content_reports), CreatePtmEvent, PublishSlots (teacher), BookSlot (guardian; uq_ptb), CancelBooking, MarkPtmAttendance; reminders the day before.
3. Screens (D10 exactly): SCR-COMM-007, 008, 009, 010, 011, 012, 015, 016.
4. Notifications: communication.message_received (in-app; SMS digest optional), communication.ptm_reminder.
5. decisions.md rows: DEC-289, GAP-37.

DATABASE
message_threads (mth), messages (msg), ptm_events (pte), ptm_slots (pts), ptm_bookings (ptb) per D07 Part 3 7.32.

BUSINESS RULES
- DEC-289 safeguards; oversight read-only; no contact details exposed.

UI SPEC
- Thread view with office hours notice; booking grid of slots at 360px.

SECURITY AND PERMISSIONS
- Only eligible pairs can message; teachers lose access when the assignment ends; oversight by PR and VP only.

TESTS
- Eligibility, office hours queueing, immutability, hide with reason, oversight read-only, booking capacity.

ACCEPTANCE CRITERIA
1. Safeguard tests green; a PTM runs end to end on staging.
2. CI green; staging green.

DO NOT
- Do not expose phone numbers or emails in messages.

OUTPUT
Report per Master section 20 with safeguard test evidence.
```

### P16.2 WhatsApp Channel

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D11 Part 2 11.27 (pipeline), D10 Part 2 SCR-NTF-006 (WhatsApp providers R2), D14 Part 3 P02.2 (notification engine), DEC-257 (consent), DEC-290; the official WhatsApp Business Platform documentation read at the time of the step.

PHASE GOAL
Add WhatsApp as a notification channel on each organization's own WhatsApp Business account using only approved templates, consent, delivery status webhooks and SMS fallback.

ALREADY BUILT
Notification engine with SMS, email and in-app, consent_records, broadcasts.

TASKS
1. WhatsAppGateway contract and CloudApiWhatsAppGateway adapter built from the official documentation (template messages with language and parameters; credentials per organization encrypted; phone number id; rate limits respected).
2. Template mapping: notification templates gain a WhatsApp variant mapped to an approved template name, language and ordered parameters; a sync action imports approved template names and statuses from the account; unapproved or paused templates are never sent.
3. Pipeline changes: channel whatsapp allowed only with current whatsapp consent (DEC-257, DEC-290); fallback to SMS for mandatory events on failure or no consent; quiet hours apply.
4. Status webhook on hooks host (/webhooks/whatsapp/{organization_uuid}) with signature verification, stored first, updating dispatch status.
5. Screens: SCR-NTF-006 WhatsApp provider section, SCR-NTF-004 WhatsApp template mapping, SCR-NTF-009 broadcast channel option with template choice; SCR-COMM-002 WhatsApp copy enabled.
6. decisions.md rows: DEC-290.

DATABASE
Expand-only columns for template mapping on notification_templates (whatsapp_template_name, whatsapp_language, whatsapp_parameters JSON) and provider type whatsapp in notification_providers.

BUSINESS RULES
- Only approved templates; only consenting recipients; mandatory events always reach the guardian by SMS if WhatsApp fails.

UI SPEC
- Template mapping shows approval status from the account and a preview with sample parameters.

SECURITY AND PERMISSIONS
- Credentials encrypted; webhook verified; no message content in logs.

TESTS
- Consent gating; unapproved template blocked; fallback; webhook verification; parameter count validation.

ACCEPTANCE CRITERIA
1. In a test WhatsApp Business account, an absence alert template reaches a consenting test guardian and a non-consenting one receives SMS.
2. CI green; staging green.

DO NOT
- Do not send free-form business-initiated messages.
- Do not use unofficial WhatsApp libraries or web automation.

OUTPUT
Report per Master section 20 with sandbox evidence.
```

### P16.3 Substitutions, Gate Passes, Invigilation and Scheduled Reports

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 3 7.31 (timetable_substitutions), 7.33 (gate_passes), 7.28 (invigilation_duties), 7.34 (scheduled_reports); D11 Part 2 11.30 (substitution); D10 Part 2 and Part 3 SCR-TTB-009, SCR-ADN-005, SCR-ADN-010, SCR-EXM-019, SCR-RPT-005; D01 GAP-33, GAP-36.

PHASE GOAL
Cover absent teachers' periods, control early pick-ups with gate passes, schedule exam invigilation without clashes, and email scheduled reports to owners and principals.

ALREADY BUILT
R1 timetable, staff attendance and leave, pickup authorizations, exams, report runner.

TASKS
1. Migrations (expand-only): timetable_substitutions, gate_passes, invigilation_duties, scheduled_reports exactly as D07.
2. Substitutions: SubstitutionPlanner (by 08:00 lists periods of teachers on approved leave or absent, suggests free teachers ranked by load and subject), AssignSubstitute (uq_tsb), notifications to substitutes, My timetable shows substitutions.
3. Gate passes: IssueGatePass (picker must be an active pickup authorization or guardian; photo shown for matching; CNIC last4 only; guardian notified), attendance note for half day where policy sets.
4. Invigilation: AssignInvigilation (teacher clash check against timetable and other duties; uq_ivd), duty slips print.
5. Scheduled reports: CreateScheduledReport (report key, filters, format, frequency, send time, recipients among organization users), command reporting:send-scheduled every 15 minutes, email with masked attachment generated through the export engine; unsubscribe link for recipients.
6. Screens (D10 exactly): SCR-TTB-009 Substitutions, SCR-ADN-005 Gate passes, SCR-ADN-010 Guardian: gate passes, SCR-EXM-019 Invigilation duties, SCR-RPT-005 Scheduled reports. Widget timetable.substitutions_today.
7. decisions.md rows: GAP-33, GAP-36.

DATABASE
timetable_substitutions (tsb), gate_passes (gtp), invigilation_duties (ivd), scheduled_reports (scr) per D07 Part 3.

BUSINESS RULES
- One substitute per entry and date; gate passes only to authorized pickers; scheduled reports go only to organization users with the report's permission at send time.

UI SPEC
- Substitution grid with free-teacher suggestions; gate pass form with photo match.

SECURITY AND PERMISSIONS
- Reports re-check recipient permission each send; attachments masked.

TESTS
- Substitution suggestions; gate pass authorization; invigilation clashes; scheduled report permission re-check.

ACCEPTANCE CRITERIA
1. Each feature works end to end on staging.
2. CI green; staging green.

DO NOT
- Do not email reports to addresses outside the organization's users.

OUTPUT
Report per Master section 20.
```

### P16.4 Admission R2, Document Verification and Approval Escalation

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 2 7.18 (admission_inquiries, admission_assessments); D10 Part 2 SCR-ADM-012, 013, SCR-DOC-006; D10 Part 3 SCR-RES-021; D11 Part 1 11.3 (expiry) and DEC-169; D01 GAP-35; D14 DEC-291.

PHASE GOAL
Add inquiry follow-ups and admission tests and interviews, the public verification page for certificates, report cards and receipts, and approval SLA escalation with expiry.

ALREADY BUILT
R1 admission, document engine with verification codes, report cards, approval engine with sla_hours column.

TASKS
1. Migrations (expand-only): admission_inquiries, admission_assessments exactly as D07 7.18.
2. Inquiries: CreateInquiry, LogFollowUp, ConvertInquiryToApplication, MarkLost; follow-up reminders and an attention item.
3. Assessments: ScheduleAssessment (test or interview), RecordAssessmentResult, decision panel shows results.
4. Public verification (GAP-35): SCR-DOC-006 public.document.verify and SCR-RES-021 public.result.verify on the tenant host: code entry or QR; rate limited per IP; shows document type, holder name, issue date, status (valid, superseded, revoked) only; QR added to issued document footers and report cards.
5. Escalation (DEC-291): workflow editor enables SLA hours per step and escalation resolver; approvals:escalate every 15 minutes; expiry for types that allow it; notifications.
6. Screens (D10 exactly): SCR-ADM-012 Inquiries, SCR-ADM-013 Assessments, SCR-DOC-006 Verify document, SCR-RES-021 Verify result card; SCR-APR-007 SLA fields enabled.
7. Retention of rejected applications (FR-ADM-015, added by D15 DEC-324): command admission:anonymize-rejected runs monthly; applications in rejected or lapsed status older than the setting admission.rejected_retention_months (default 12) have personal fields replaced with fixed placeholders, their documents deleted from storage, and the application number, class, dates and outcome kept for statistics; audit records the run and counts, never the values.
8. decisions.md rows: DEC-169, DEC-291, GAP-35, DEC-324.

DATABASE
admission_inquiries (ainq), admission_assessments (aas) per D07 Part 2 7.18.

BUSINESS RULES
- Verification pages show minimal data; escalation never removes original assignees.

UI SPEC
- Verification page mobile-first with clear valid or revoked states in English and Urdu.

SECURITY AND PERMISSIONS
- Public pages rate limited and free of personal data beyond the holder name.

TESTS
- Inquiry conversion; assessment flow; verification states and rate limit; escalation timing and expiry.
- Anonymization: only rejected or lapsed applications past the retention period change; converted applications never change; documents removed from storage.

ACCEPTANCE CRITERIA
1. Each feature works on staging; escalation fires within 15 minutes of SLA breach.
2. CI green; staging green.

DO NOT
- Do not show other personal data on verification pages.

OUTPUT
Report per Master section 20.
```

### P16.5 External Exam Workflows, Board Support and Result Rechecks

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D06A A.9 completely (FR-EXT-004 to 016) and DEC-84, DEC-86; D07 Part 3 7.30 R2 tables (exam_series, external_exam_fee_schedules, external_exam_entries, external_exam_result_imports, external_exam_results, external_exam_enquiries, external_certificates, equivalence_calculations); D11 Part 2 11.29; D11 Part 1 11.4 (external_entry_withdrawal, result_recheck) and 11.11 (F1f, F17 worked example); D10 Part 3 SCR-EXT-007 to 025, SCR-RES-016, SCR-RES-020; D06A A.11; D14 DEC-266, DEC-281.

PHASE GOAL
Run Cambridge and national board external exams end to end: series with deadlines, fee schedules, draft entries from student subjects, review and guardian confirmation, entry fee vouchers as pass-through money, export and submission, withdrawals, result import and publication, enquiries, certificates register, IBCC equivalence calculator, national board registration and results, and internal result rechecks.

ALREADY BUILT
P07.3 external exam foundations, fee vouchers with pass-through heads and F1f, F17 journal template, approval engine, import engine.

TASKS
1. Migrations (expand-only): exam_series, external_exam_fee_schedules, external_exam_entries, external_exam_result_imports, external_exam_results, external_exam_enquiries, external_certificates, equivalence_calculations exactly as D07 7.30.
2. Actions: CreateSeries (deadlines normal, late 1, late 2, retake, results date), SetFeeSchedules, GenerateDraftEntries (from student_subjects and target qualifications per DEC-86; syllabus codes valid for the series), ReviewEntries, ConfirmEntriesByGuardian (portal, when setting external_exam.guardian_confirmation is on, A-95), ConfirmEntries (issues exam_entry ad hoc vouchers through Fee with pass-through lines, F1f; stage fee by date), MarkEntriesSubmitted, ExportEntries (awarding body file format as mapping), RequestEntryWithdrawal (approval external_entry_withdrawal: 1 principal_of_branch), ImportResults (statement of results file mapping; matched and unmatched candidates), PublishExternalResults, RecordEnquiry (fee voucher if configured; outcome), RecordCertificateReceived and HandOverCertificate (signature), CalculateIbccEquivalence (from ibcc_conversion_rules for the session year; saved calculations; guardians and students may run it on their published results), RemitEntryFees (F17 through the template), BoardRegistrationExport and BoardResultEntry (FR-EXT-016, SCR-EXT-023).
3. Result rechecks: RequestResultRecheck (portal; fee voucher if configured), approval result_recheck (1 exam_controller_of_branch), outcome through marks_revision; screens SCR-RES-016, SCR-RES-020.
4. Screens (D10 exactly): SCR-EXT-007 to SCR-EXT-025; widget external_exam.deadlines; attention item for entry deadlines within 7 days.
5. decisions.md rows: DEC-84, DEC-86, DEC-281.

DATABASE
External exam R2 tables per D07 Part 3 7.30.

BUSINESS RULES
- Entry fees are pass-through liabilities, never income (DEC-84); remittance clears the liability.
- Real Cambridge grade thresholds are never computed; results are stored as reported.
- Withdrawals after submission follow approval and awarding body refund rules.

UI SPEC
- Entry review grid by syllabus with component options and guardian confirmation status.

SECURITY AND PERMISSIONS
- Permissions per D06A A.11; guardians see only their child's entries and published results.

TESTS
- Draft generation with validity series; voucher pass-through postings and remittance (D11 worked example 5054); result import matching; IBCC calculation from data fixtures; recheck flow.

ACCEPTANCE CRITERIA
1. A June series runs from entries to published results on staging for an O Level class, with fees remitted and certificates recorded.
2. Phase exit criteria evidence; CI green; staging green.

DO NOT
- Do not compute awarding body grades.
- Do not post entry fees to income.

OUTPUT
Report per Master section 20 with the series walkthrough. Then run the D13 13.6 gate for PH-16.
```

---

## 14.29 PH-17 Transport Routes, Discipline and Attendance R2

**Exit criteria:**
- Phase gate G-1 to G-11.
- Transport fees reach vouchers only through `VoucherLineContributor`.
- Sync agent punches arrive exactly once.

### P17.1 Transport Routes and Allocations

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 4 7.42 (transport_routes, transport_stops, route_stops, transport_allocations) with DEC-121; D11 Part 2 11.30 (transport); D10 Part 3 SCR-TRN-001 to 006, 016; D05 5.7.21; D14 DEC-124, DEC-282.

PHASE GOAL
Run school transport for R2: routes with stops, times and monthly fees, student allocations with seat checks, transport charges on vouchers through the contributor registry, route sheets, guardian view, and lifecycle hooks.

ALREADY BUILT
R1 fee vouchers with VoucherLineContributorRegistry, students and enrollments, clearance registry, withdrawal and transfer events.

TASKS
1. Migrations (Transport, expand-only): transport_routes (with capacity per DEC-282), transport_stops, route_stops, transport_allocations exactly as D07 7.42.
2. Actions: CreateRoute, AddRouteStop (sequence, times, monthly fee), AllocateStudent (locks the route row; counts active allocations against capacity; uq_tal_active; monthly fee snapshot and fee head), ChangeStop (ends allocation and starts a new one from the date), SuspendAllocation, EndAllocation.
3. Fee contributor: TransportVoucherLineContributor adds one line per active allocation for the fee month with source_ref_type transport_allocation (DEC-124); pro rata per setting transport.prorata.
4. Listeners: StudentWithdrawn and StudentTransferred end allocations; ClearanceCheck transport registered.
5. Screens (D10 exactly): SCR-TRN-001 to SCR-TRN-006 and SCR-TRN-016. Widget transport.capacity.
6. decisions.md rows: DEC-124, DEC-282.

DATABASE
transport_routes (trt), transport_stops (tsp), route_stops (rst), transport_allocations (tal) per D07 Part 4 7.42 with DEC-282.

BUSINESS RULES
- One active allocation per student; capacity enforced under a row lock; fees only through the contributor.

UI SPEC
- Route form with ordered stops and fees; allocation modal with capacity and fee preview.

SECURITY AND PERMISSIONS
- Route sheets mask guardian phones per permission.

TESTS
- Concurrent allocation at capacity; contributor lines in voucher generation; listeners end allocations.

ACCEPTANCE CRITERIA
1. Allocated students get transport lines on the next vouchers on staging.
2. CI green; staging green.

DO NOT
- Do not insert fee lines directly from Transport.

OUTPUT
Report per Master section 20.
```

### P17.2 Discipline

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 4 7.44 (discipline_categories, discipline_incidents, discipline_incident_students, discipline_actions, student_remarks, student_achievements); D11 Part 2 11.30 (discipline); D10 Part 3 SCR-DSC-001 to 007, 010, 011; D05 5.7.22; D02 FR-DSC.

PHASE GOAL
Record incidents with students and involvement, decide actions including suspensions through Student, keep remarks and achievements, notify guardians by category, and show shared items in portals.

ALREADY BUILT
R1 students with suspensions (P08.2), timeline, notification engine, portals.

TASKS
1. Migrations (expand-only): discipline_categories, discipline_incidents, discipline_incident_students, discipline_actions, student_remarks, student_achievements exactly as D07 7.44.
2. Actions: RecordIncident, AddStudentToIncident (involvement), DecideAction (suspension calls Student SuspendStudent and stores student_suspension_id), RevokeAction, CloseIncident, AddRemark (visibility internal or guardian), RecordAchievement (show_on_portal); timeline events for each.
3. Guardian notifications per category requires_guardian_notice.
4. Screens (D10 exactly): SCR-DSC-001 to 007, 010, 011. Widget discipline.recent_incidents.
5. decisions.md rows: discipline rules.

DATABASE
Discipline tables (except counseling, R3) per D07 Part 4 7.44.

BUSINESS RULES
- Guardians see only shared remarks and actions; suspensions only through Student.

UI SPEC
- Incident form with multiple students and involvement chips.

SECURITY AND PERMISSIONS
- Incident visibility per D05; victims and witnesses never shown to other guardians.

TESTS
- Suspension creation via Student; guardian visibility; notifications by category.

ACCEPTANCE CRITERIA
1. Incident to suspension flow on staging with SU attendance.
2. CI green; staging green.

DO NOT
- Do not build counseling (R3).

OUTPUT
Report per Master section 20.
```

### P17.3 Sync Agent and Device Token API

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D08 8.15 (API architecture, device tokens), D10 Part 2 SCR-IAM-027 (API tokens), D14 Part 4 P05.3 (punch intake, RecordPunch), D12 Part 1 12.8, D14 DEC-283, DEC-284.

PHASE GOAL
Collect punches from pull-only and RFID devices that cannot push: Sanctum device tokens and the first API v1 endpoint in the main application, and the on-premise sync agent in its own repository.

ALREADY BUILT
R1 attendance with RecordPunch, uq_apn duplicates, device registry; adms and hooks hosts.

TASKS
1. Main repository: install laravel/sanctum (package check), personal_access_tokens with organization_id (D7 Part 1), api.v1 middleware group on api host, ResolveTenantFromToken, device tokens issued to attendance_devices with ability device:punch (90-day expiry, rotation endpoint), endpoint POST /api/v1/attendance/punches (api.v1.attendance.punches.store; batch up to 500; idempotent via uq_apn; returns accepted and duplicate counts), rate limit per token, OpenAPI entry in docs/api.
2. Screen SCR-IAM-027 API tokens (device tokens only in this step; token shown once).
3. Sync agent repository darsora-sync-agent (DEC-284): Laravel Zero CLI; configuration file with API base URL, device token and device list; adapter interface DeviceReader with implementations for ZKTeco pull mode and a CSV folder watcher, plus an RFID reader adapter through the same interface; append-only JSONL spool with rotation; push loop with backoff; Windows service and Linux systemd install scripts; single binaries built with static-php-cli in CI; signed releases; self-update check against signed release metadata.
4. Device registry: adapter value agent; agent heartbeat endpoint POST /api/v1/attendance/agent-heartbeats.
5. Push user lists to devices (FR-ATT-025, added by D15 DEC-324): QueueDeviceUserSync builds ADMS user commands from active device_enrollments for one device or a branch; commands are held in the Redis list attendance:device_commands:{device_id}, returned by the existing ADMS getrequest handler (P05.3), and their results from devicecmd are written to device_sync_logs; agent-managed devices receive the same list through GET /api/v1/attendance/agent-users; a "Push to device" action on SCR-ATT-019 (permission attendance.device.manage).
6. decisions.md rows: DEC-283, DEC-284, DEC-324.

DATABASE
personal_access_tokens with organization_id (expand-only if not present).

BUSINESS RULES
- The agent never connects to the database; punches arrive only through the API and are stored once.

UI SPEC
- API tokens list with device name, last used, expiry, revoke.

SECURITY AND PERMISSIONS
- Device tokens can call only punch and heartbeat endpoints; tokens hashed; revocation immediate.

TESTS
- Token abilities; idempotent batch; tenant from token only; agent spool and retry (agent repository tests).
- User list push: ADMS command format from fixtures, result logging, only active enrollments sent, device of another organization never receives users.

ACCEPTANCE CRITERIA
1. A pull-only device on a test LAN delivers punches through the agent with exactly-once storage; phase exit criteria evidence.
2. CI green in both repositories; staging green.

DO NOT
- Do not open database ports to schools.

OUTPUT
Report per Master section 20 with agent install guide and evidence. Then run the D13 13.6 gate for PH-17.
```

---

## 14.30 PH-18 Platform R2, API v1 and R2 Launch

**Exit criteria:**
- MS-14 is reached.

### P18.1 Usage Metering, Storage Quotas and Plan Limits

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D07 Part 1 (usage_counters, plans, plan_modules), D10 Part 2 SCR-PLT-010, 011; D01 GAP-40; D14 Part 3 P02.2 (LimitEnforcer for SMS).

PHASE GOAL
Measure and enforce plan limits fairly: students, branches, storage and SMS credits, with image compression and video caps, warnings and blocks that never lose data.

ALREADY BUILT
Plans with limits, usage_counters, LimitEnforcer for SMS, FileStorageService.

TASKS
1. UsageMeter: counters per organization and month for active students, branches, storage bytes (sum of stored_files), SMS segments, WhatsApp messages; recomputed nightly and incremented on events.
2. LimitEnforcer generalized: student and branch creation refused above plan limits with a clear message; storage uploads refused above quota (existing files untouched); warnings at 80% and 100% to owners.
3. Image compression on upload already re-encodes; add max dimensions per purpose and video caps from plan.
4. Screens: SCR-PLT-011 Usage enhancements; tenant usage page under settings for owners.
5. decisions.md rows: GAP-40.

DATABASE
No new tables (usage_counters columns expand-only if needed).

BUSINESS RULES
- Limits block new creation only; nothing existing is deleted or hidden.

UI SPEC
- Usage bars with thresholds.

SECURITY AND PERMISSIONS
- Usage visible to owners and platform billing.

TESTS
- Limit enforcement and warnings; counters accuracy.

ACCEPTANCE CRITERIA
1. Limits enforced on staging; CI green.

DO NOT
- Do not delete tenant data on limit breach.

OUTPUT
Report per Master section 20.
```

### P18.2 API v1

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D08 8.15 completely (layering, auth, URLs, status codes, envelope, ETags) and DEC-133, DEC-135; D12 Part 1 API security; D14 DEC-283.

PHASE GOAL
Publish API v1 for future guardian, student and teacher apps using the same Actions, Queries and Policies as the web, with portal tokens, OpenAPI 3.1 contract tests and rate limits.

ALREADY BUILT
P17.3 Sanctum, api.v1 group and device endpoint; every R1 and R2 module.

TASKS
1. Identity endpoints: tokens (password, OTP request and verification for guardians, 2FA step), rotations, revocation, me.
2. Guardian and student endpoints: children, attendance days, vouchers (with signed PDF links), payments, online payment initiation and status, results and report cards, diary, announcements, timetable, homework and submissions, quizzes listing (attempts stay web-only in R2), leave applications for children, notifications.
3. Teacher endpoints: my classes, section sheet read and mark (same DEC-162 rules), diary write, homework review list.
4. Resources with the D08 envelope, money objects, enums with labels, ETag and If-Match on updates, pagination, filters and sorts whitelisted.
5. OpenAPI 3.1 in docs/api with contract test (DEC-135); rate limits per token and per route group; versioning headers.
6. SCR-IAM-027 portal token management for admins (revoke).
7. decisions.md rows: DEC-133, DEC-135.

DATABASE
No new tables.

BUSINESS RULES
- No business rule exists only in the API; API controllers call existing Actions.

UI SPEC
- None beyond token management.

SECURITY AND PERMISSIONS
- Abilities pick the portal; permissions and visibility from role assignments; tenant from token; branch header validated.

TESTS
- Contract test; isolation through API; masking in API responses; ETag conflicts; rate limits.

ACCEPTANCE CRITERIA
1. Contract test green; API isolation and masking tests green; CI green.

DO NOT
- Do not duplicate business logic in controllers.

OUTPUT
Report per Master section 20 with the endpoint list.
```

### P18.3 Urdu Everywhere, Help, Demo Environment and Tenant Export

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D01 GAP-38, GAP-39; D04 4.20 (tenant export and extraction); D10 Part 2 SCR-PLT-020; D14 DEC-280, DEC-292.

PHASE GOAL
Finish R2 polish: Urdu complete for every portal, in-app help with Urdu guides, the demo environment, and the tenant offboarding export package.

ALREADY BUILT
All R2 modules; TranslationParityTest extended in P10.1.

TASKS
1. TranslationParityTest extended to all staff portal views; native review for staff portal and R2 documents.
2. In-app help: help links per screen to docs in English and Urdu; contextual help drawers on complex screens (fee generation, promotion, payroll run, timetable editor).
3. Demo environment (DEC-292): separate deployment, DemoSeeder, nightly reset, fake adapters, demo banner.
4. Import undo (FR-IMX-010, added by D15 DEC-324): UndoImportBatch on SCR-IMX-003 within 24 hours of completion; each import type declares canUndo (dependency check per created record from import_job_rows.created_record_type and created_record_id) and an undo action (soft delete for master data); financial imports such as opening fee balances are never undone and are corrected through their own reversal flows; the job status becomes undone; audit records the undo with counts.
5. Tenant export (GAP-39, D4 4.20): ExportTenant builds a package per organization (one NDJSON file per table, files from storage, archived partitions included, manifest with checksums, encrypted with a password delivered separately); SCR-PLT-020 Tenant export wizard; download link with expiry.
6. decisions.md rows: DEC-292, GAP-38, GAP-39, DEC-324.

DATABASE
No new tables.

BUSINESS RULES
- Tenant exports contain only that organization's rows and files; offboarding follows DEC-51 after export.

UI SPEC
- Help drawer pattern per D09.

SECURITY AND PERMISSIONS
- Tenant export needs platform.organization.manage and owner confirmation; package encrypted.

TESTS
- Translation parity; export completeness and isolation (no other tenant rows); demo reset.
- Import undo: refused after 24 hours or when a dependent record exists; financial import types refuse undo; undone rows soft-deleted and audited.

ACCEPTANCE CRITERIA
1. Export of a test organization restores into an empty database with matching counts; CI green.

DO NOT
- Do not put real data in demo.

OUTPUT
Report per Master section 20.
```

### P18.4 R2 Hardening and General Availability

```text
PROJECT CONTEXT
The Phase 0 Master Context Prompt applies. Read D13 13.6 and 13.7, D12 Part 2 12.22 to 12.24, D14 Part 7 P10.1, P10.2 and P10.4 (same procedures), DEC-278 (fix steps).

PHASE GOAL
Release R2: every R2 screen covered, security suite and penetration test delta clean, full load set including online payments, quizzes and API within budgets, release checklist proven, and MS-14 recorded.

ALREADY BUILT
PH-11 to P18.3.

TASKS
1. Repeat P10.1 for R2 scope (ScreenCoverageTest R2, D12 12.24 including payment webhooks and API, accessibility on R2 key screens, Urdu, load set with webhook flood on real adapters' fakes and quiz concurrency).
2. Penetration test delta on R2 surfaces (payments, API, messaging, WhatsApp webhooks, sync agent endpoints) with fixes as P18.4-fix-NNN steps.
3. Release checklist RC-1 to RC-10 for R2 with evidence; release notes and guides in English and Urdu; D15 R2 rows signed; tag r2.0.0.
4. decisions.md rows: R2 GA decision.

DATABASE
Index additions only.

BUSINESS RULES
- GA only with every RC item proven.

UI SPEC
- None beyond fixes.

SECURITY AND PERMISSIONS
- No open high or critical findings.

TESTS
- All suites, load set and contract tests green.

ACCEPTANCE CRITERIA
1. MS-14 recorded with the checklist; CI green on the tag; production smoke tests green.

DO NOT
- Do not lower gates to meet the date.

OUTPUT
Report per Master section 20 with the R2 checklist. Then run the D13 13.6 gate for PH-18.
```

---

## Assumptions (Part 8)

| ID | Assumption |
|---|---|
| A-135 | Organizations that want online payments sign their own merchant agreements with JazzCash, Easypaisa and a 1Bill aggregator. Each provider hands over its official integration guide under that agreement. |
| A-136 | WhatsApp Business account verification and template approval take several weeks, so organizations start that process before P16.2 reaches them. |
| A-137 | A platform admin enters income tax slabs from each Finance Act before the tax year starts on 1 July, and EOBI and social security rates as organization settings with effective dates. |
| A-138 | School PCs that run the sync agent use Windows 10 or later, or a current Linux, with outbound HTTPS to the API host. |
| A-139 | Mobile apps are outside R2. API v1 is published for them, but the apps themselves are a separate project. |

## Gaps Found and Added (Part 8)

| ID | Gap | Decision |
|---|---|---|
| GAP-339 | No R2 step held the external exam workflows moved out of R1 | P16.5 (DEC-281) |
| GAP-340 | Transport seat checks depended on R3 vehicle tables | Route capacity (DEC-282) |
| GAP-341 | The sync agent needed an API before API v1 existed | Staged API (DEC-283) |
| GAP-342 | The sync agent's form and runtime were undefined | Separate repository and single binary (DEC-284) |
| GAP-343 | Payables and default expense postings had no system roles | DEC-285 |
| GAP-344 | Whose merchant account receives online payments was undefined | Organization's own accounts (DEC-286) |
| GAP-345 | Payroll statutory calculations risked hardcoded rates | Registered calculators with data rates (DEC-287) |
| GAP-346 | Quiz scores could bypass the marks review | Draft import only (DEC-288) |
| GAP-347 | Messaging safeguards were listed but not specified | DEC-289 |
| GAP-348 | The WhatsApp account model and fallback were undefined | DEC-290 |
| GAP-349 | The R2 approval SLA mechanism was undefined | DEC-291 |
| GAP-350 | The demo environment was not specified | DEC-292 |
