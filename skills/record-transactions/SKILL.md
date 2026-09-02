---
name: record-transactions
description: The go-to guide for recording any transaction — pick the correct tool (Expense vs Bill vs BillPayment, Invoice vs SalesReceipt vs ReceivePayment vs Deposit), follow the universal safety protocol, and handle edge cases. Use when the user says record, enter, book, or log a transaction, payment, purchase, sale, or deposit and no more specific skill applies.
---

# Record Transactions Skill

The master guide for recording transactions in QuickBooks. Answers the two questions every recording starts with: **which tool** and **in what order**. For deep workflows, defer to the sister skills: accounts-payable, accounts-receivable, journal-entries, bank-feed-processing.

## Which company? (do this first)

One DeepLedger connection reaches every company the user can access. Before
anything else in this skill:

1. Call `qbCompanyProfile` (operation `profile`, the default). Its `company`
   field names the company you are in. Say the company name to the user in your
   first line.
2. If it answers `NO_ACTIVE_COMPANY`, call `qbCompanyProfile` with
   `operation: "list"`, then `operation: "switch"` with the `organizationId` the
   user means. Never guess between similar names; ask.
3. If the user names a different company than the active one, switch first.
   The switch is proven by a CompanyInfo read; a `qbConnected: false` answer
   means QuickBooks is not connected for that company and only tasks, documents,
   memory and the bank feed will work.
4. Every tool result carries `company`. If one carries `warning` (the active
   company was moved from the portal or another client), stop and confirm the
   company with the user before writing.

## Trigger

Activate when the user wants to:
- Record, enter, book, or log any transaction
- Record a payment, purchase, sale, refund, or deposit
- Is unsure how a transaction should be recorded

## Universal Recording Protocol

Every recording follows the same six steps — hooks enforce 1, 3, and 5:

1. **Lookup** — `qbMasterData` for vendor/customer ID, source account, and category account IDs.
2. **Check memory** — `agentMemory(operation="read")` for client `policies` (CPA rules like "all Uber rides → Travel") and confirmed recurring `patterns`. Vendor→account mappings are never stored in memory — infer them from QB history in the decide step.
3. **Duplicate check** — `qbFetchTransactions` with entity + date (±15 days) + amount. If matches return, show them and get explicit confirmation before proceeding.
4. **Decide** — Proceed only if ONE of these is true:
   - The user explicitly requested or confirmed this exact transaction (the request IS the confirmation — don't re-ask), OR
   - A CPA approved it via task — use `effectiveCategory` verbatim, OR
   - QB history clearly supports the categorization (consistency rule below).
   Otherwise → `tasks(operation="create")` with specific `aiReasoning`. Never guess an account.
5. **Record** — One tool call per transaction (there is no batch tool). Show what was recorded: account name, code, ID, amounts.
6. **Attach & learn** — `qbAttachFile` with the source document (audit-ready books). If the charge is now a confirmed recurring stream (seen 2+ times), write an `agentMemory` `patterns` entry: vendor/source, amount range, frequency, QB account.

**Consistency rule** (`qbFetchTransactions` — entity's 6-month history, no transactionType): use the dominant account ONLY IF ≥3 transactions, dominant share ≥70%, no runner-up ≥20%, and amount within 5× the median.

**When to pause for the user** — confirmation is reserved for interrupts, not routine writes:
- Duplicate check returned potential matches → show them, get explicit "not a duplicate"
- Amount anomaly (>5× the historical median — charter §6 — or under 1/3 the minimum for this entity)
- A wrong-type guard fires (e.g., outstanding bill exists but an Expense was requested)
- Voids — `qbVoidTransaction` asks the human directly via elicitation; irreversible master-data choices (`accountType`)

Reads and workflow tools (reports, fetches, tasks, agentMemory, closeRun) never need approval.

## Tool Selection: Money OUT

| Situation | Tool |
|-----------|------|
| Paid now — card swipe, ACH, check, cash | `qbExpense` |
| Vendor invoice to pay later ("net 30", "bill from", "on account") | `qbBill` |
| Paying a bill already entered in QB | `qbBillPayment` |
| Refunding a customer | `qbRefundReceipt` |
| Vendor issued a credit (return, adjustment) | `qbCredit(creditType="vendor")` |
| Cost incurred on behalf of a client (reimbursable) | `qbExpense`/`qbBill` line with `customerId` + `billableStatus: "Billable"` |

**Before any expense**: `qbFetchTransactions(transactionType="Bill", outstandingOnly=true, entityId=vendorId)`. If an outstanding bill exists → `qbBillPayment`, never a second `qbExpense` (double-counts when the bill is paid).

## Tool Selection: Money IN

| Situation | Tool |
|-----------|------|
| Customer paying an existing invoice | `qbReceivePayment` |
| Sale + payment in one step (no invoice) | `qbSalesReceipt` |
| Billing a customer to pay later | `qbInvoice` |
| Non-customer income — interest, rebates, insurance proceeds, vendor refunds received as cash | `qbDeposit` (direct lines) |
| Moving Undeposited Funds into the bank | `qbDeposit` (LinkedTxn lines) |
| Customer credit (return, adjustment, discount) | `qbCredit(creditType="customer")` |

**Before any deposit or sales receipt**: `qbFetchTransactions(transactionType="Invoice", outstandingOnly=true, entityId=customerId)`. If an outstanding invoice exists → `qbReceivePayment`. A Deposit never touches AR; a SalesReceipt double-counts income.

**The AR flow is two steps**: `qbReceivePayment`/`qbSalesReceipt` → Undeposited Funds → `qbDeposit` → bank. ReceivePayment alone does NOT put money in the bank.

## Tool Selection: Neither In Nor Out

| Situation | Tool |
|-----------|------|
| Moving money between own accounts (checking → savings) | `qbTransfer` |
| Paying off the company credit card from the bank | `qbTransfer` (bank → CC account) |
| Accruals, depreciation, reclassifications | `qbJournalEntry` (debits must equal credits) |
| Quote or purchase order | No posting tool exists (non-posting documents) — create one-offs in QB Online directly or escalate via `tasks`; recurring templates via `qbRecurringTransaction` (supports Estimate and PurchaseOrder types) |
| Recurring template | `qbRecurringTransaction` |

## Edge Cases

**Credit card purchase** — `qbExpense` with the credit card account as the source (`paymentType: "CreditCard"`). The later card payment is a `qbTransfer`, not another expense — expensing both double-counts.

**Vendor refund** — Two forms: cash hit the bank → `qbDeposit` crediting the original expense account; credit against future bills → `qbCredit(creditType="vendor")`, applied in the next `qbBillPayment` with `txnType: "VendorCredit"`.

**Partial payment** — Pay/apply less than the full amount; the bill or invoice stays partially outstanding. Never edit the original to match the payment.

**Overpayment** — `qbReceivePayment` creates an unapplied credit (`unappliedAmount` in the response). Tell the user; apply it to the next invoice.

**Personal expense on business card** — Record `qbExpense` with the line pointed at Owner's Draw (equity), never a business expense category.

**Owner contribution / draw** — Equity accounts only: contribution = `qbDeposit` to Owner's Contribution; draw = `qbExpense` to Owner's Draw. Never income or expense categories.

**Loan payment** — Split lines: principal to the loan liability account, interest to Interest Expense. One expense with two lines.

**Billable client cost (reimbursable)** — To re-bill a cost to a client, set BOTH `customerId` AND `billableStatus: "Billable"` on the same line of `qbExpense`/`qbBill` — line-level, not header; Billable without a line `customerId` is invalid. `customerId` alone (or `NotBillable`) is job-costing only: the cost shows in project profitability but is never offered for re-billing. Requires "track billable expenses" enabled in QB company settings. The charge surfaces on that customer's next invoice — `qbInvoice` cannot auto-link it, so if you add it as a manual invoice line, update the original line to `HasBeenBilled` to prevent double-billing. `qbJournalEntry` lines take `customerId` for job costing but have no `billableStatus`.

**Nonprofit restricted funds** — Nonprofits use the same line fields to track restricted-fund and grant spending: each grant/fund is set up as a customer (or sub-customer/project), and expense lines are tagged with that `customerId` so spending reports against the fund. Use `billableStatus: "NotBillable"` — unless the grantor actually reimburses expenses, then mark `"Billable"` and invoice the grantor.

**Payroll** — Recorded from payroll provider reports via `qbJournalEntry` (gross wages, taxes, liabilities). If no report is available, flag for CPA review rather than guessing.

**Source = category collision** — The source account (where money comes from) must differ from every line account (what it was for). Same account on both sides is a zero-net entry that breaks reconciliation.

**Amount anomaly** — If the amount is more than 5× the historical median (charter §6; or below 1/3 the minimum) for this entity — from the 6-month scan or a `patterns` entry — confirm with the user even if the decide gate passes.

**Closed-period error** — Never modify or void in a closed period. Record a reversing journal entry in the current period (see journal-entries skill).

**Wrong transaction already recorded (open period)** — `qbFetchTransactions` to verify, `qbVoidTransaction` (preserves audit trail — never delete), then re-record correctly.

**New vendor/customer** — Not in `qbMasterData` results? Check for name variants first (ACME vs ACME LLC), then create via `qbMasterData(operation="create")` with user confirmation, then record.

**Can't determine category** — Don't guess. `tasks(operation="create")` with specific `aiReasoning` ("New vendor X, no mapping"; "Ambiguous description: CHECKCARD 0315") and a `suggestedCategory` if you have one.

## Safety Checklist

- [ ] `qbMasterData` lookup done — all IDs from this conversation's results
- [ ] Outstanding bills/invoices checked before choosing the tool
- [ ] Duplicate check run (entity + date ±15 days + amount)
- [ ] Source account differs from all line accounts
- [ ] Amount within the historical range, or confirmed by user
- [ ] Decide gate passed — user-requested, CPA-approved, or history-consistent; otherwise a CPA task was created instead
- [ ] Document attached; newly confirmed recurring charges written as `patterns` entries

## Common Mistakes to Avoid

- `qbExpense` when an outstanding Bill exists → use `qbBillPayment`
- `qbDeposit` or `qbSalesReceipt` when an outstanding Invoice exists → use `qbReceivePayment`
- Recording a customer sale as a direct Deposit → no customer linkage, breaks sales reports
- Expensing the credit card payment after expensing the purchases → double-count; use `qbTransfer`
- Journal entry for a simple bank-to-bank move → use `qbTransfer`
- Skipping `vendorId`/`customerId` on transactions → breaks aging and spending reports
- Guessing a category instead of flagging → CPA loses trust in the ledger
